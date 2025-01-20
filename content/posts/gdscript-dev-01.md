+++
date = '2025-01-16T16:32:43+03:00'
title = 'GDScript codegen research: 01'
tags = ['gdscript','asm','c++','jit']
+++

# Foreword

About a year ago, I completed my hobby project, an [8086 assembler compiler](https://github.com/bagggage/asm-compiler), and started seeking a more ambitious and practical project. Naturally, the idea of creating yet another programming language crossed my mind, but I quickly abandoned it.

Around the same time, I became interested in the performance of GDScript, suspecting that, as a scripting language, it lacked a sophisticated backend (e.g., machine code compilation). This turned out to be correct: Godot uses a stack-based virtual machine to interpret GDScript, compiling source code into bytecode beforehand. I saw this as an excellent opportunity to develop a more efficient backend for GDScript. However, only now have I found the time and energy to dive into this.

# Concept

GDScript is currently a fairly feature-rich scripting language with various capabilities. Developing a machine code compiler for something like GDScript is no trivial task. It requires time, a methodical approach, and collaboration with other Godot developers to avoid misunderstandings and deliver genuine value to the project—rather than merely pursuing compiler development as a personal interest.

Thus, I decided to start a series of posts where I can share my ideas, research, and progress.

# Let’s Get Started!

## Code Generation Backend

Developing a custom machine code generator from scratch feels unnecessary. It would demand significant time and effort, as bugs in code generation would inevitably lead to crashes, and the x86-64 instruction set alone is quite extensive. Moreover, full-fledged support would also require arm64 and potentially other instruction sets for various platforms.

For this reason, integrating an existing, maintained backend is undoubtedly the more optimal choice for Godot. My first priority was to avoid large, heavyweight backends. While LLVM is the most popular and well-supported solution, it’s too heavy; its code generation backend alone outweighs the entire Godot project. GDScript could be faster, but at this stage, code optimization on the level of C++'s `-O2/-O3` might be overkill.

Therefore, I explored lightweight, open-source, fast backends. Open-source backends also allow for contributions to improve optimization passes, instruction selection, and other aspects if needed.

Here are the options I considered:

- [**MIR**](https://github.com/vnmakarov/mir): A lightweight alternative to LLVM, stable, and written entirely in C. (however, it might still be too large for Godot.)
- [**asmjit**](https://github.com/asmjit/asmjit): The main drawback is that asmjit is an assembler for C++ without any intermediate representation (IR) or platform-independent API. This is a significant limitation for writing a cross-platform compiler.
- [**sljit**](https://github.com/zherczeg/sljit): A platform-independent JIT compiler written in C, supporting SIMD, self-modifying code, and more. Definitely worth considering.
- [**bunny-jit**](https://github.com/signaldust/bunny-jit): The lightest of all the options, though the youngest and perhaps the least feature-rich. Nonetheless, it’s very convenient, fully written in C++, and supports x86-64 and Aarch64.

---

I chose **bunny-jit** for my research and experiments. While **sljit** has more features, bunny-jit is easy to integrate, has a simple and intuitive API, and performs its tasks efficiently.

## GDScript Execution

To better understand the task, here’s a brief overview of the key aspects I’ll be working with.

Currently, GDScript uses a stack-based virtual machine. The implementation essentially revolves around the method [`GDScriptFunction::call(...)`](https://github.com/godotengine/godot/blob/4ce466d7fa79e351d4295d5bb47e3266089c3a59/modules/gdscript/gdscript_vm.cpp#L473), which contains numerous macros. This is because the virtual machine is not a separate class, and for simplicity, repetitive code has been abstracted into macros that act like utility functions.

The most notable characteristic is the large number of opcodes. The bytecode itself seems designed with the virtual machine in mind. The basic data unit in the VM is the `Variant` class, which enables dynamic typing in the language. However, this dynamic typing introduces overhead. While the bytecode generator can sometimes emit type-specific opcodes for known types, the VM itself does not implement significant optimizations for them.

The bytecode is high-level. By "high-level," I mean it lacks low-level opcodes like `ADD`, `MUL`, `DIV`, `STORE`, or `LOAD`. Instead, it mostly consists of external function or method calls for `Variant`, such as `OPCODE_CALL_UTILITY`, `OPCODE_CALL_METHOD_BIND`, and `OPCODE_OPERATOR`. This tight coupling of bytecode and its runtime environment makes it challenging to use the bytecode as a data source for machine code generation. Each high-level operation requires ABI compliance and other intricate details.

I experimented with generating machine code for some opcodes. Unsurprisingly, the generated machine code closely resembled the original C++ code, with some performance gains from skipping runtime opcode parsing and checks. However, this is not particularly significant.

Fortunately, the GDScript module already provides a code generation interface via the [`GDScriptCodeGenerator`](https://github.com/godotengine/godot/blob/4ce466d7fa79e351d4295d5bb47e3266089c3a59/modules/gdscript/gdscript_codegen.h#L40) class, which allows integrating new code generator implementations without rewriting much existing code. The compiler [`GDScriptCompiler`](https://github.com/godotengine/godot/blob/4ce466d7fa79e351d4295d5bb47e3266089c3a59/modules/gdscript/gdscript_compiler.h#L41) uses this interface, enabling temporary variable allocation, function parameter handling, and type computations.

## VM Inefficiencies

The main bottlenecks are the thick layers of abstraction wrapping native operators, methods, and functions. While this abstraction enables dynamic typing, most variables have known types at compile time or are easy to infer. This provides room for optimization. However, retaining dynamic typing capabilities while executing code safely imposes constraints.

For instance, basic types like `Variant::INT`, `Variant::FLOAT`, and `Variant::BOOL` are manipulated through a unified, dynamic API. Nativeizing and generating machine code for these types would be a significant improvement. It would also be beneficial to have fully native implementations for type structures like `Vector2`, `Vector3`, `Rect2D`, `Quaternion`, etc. Although more complex, this is achievable without excessive effort.

Let’s analyze a simpler example:

```gd
var hf := 2.0
var sf := 2.0
var i := 2
i *= hf
i *= sf
```

The generated bytecode looks like this:

```
 0: line 4: var hf := 2.0
 2: assign stack(4) = const(2.0)
 5: line 5: var sf := 2.0
 7: assign stack(5) = const(2.0)
10: line 7: var i := 2
12: assign stack(6) = const(2)
15: line 8: i *= hf
17: validated operator stack(11) = stack(6) * stack(4)
22: assign typed builtin (int) stack(6) = stack(11)
26: line 9: i *= sf
28: operator stack(12) = stack(6) * stack(5)
37: assign typed builtin (int) stack(6) = stack(12)
41: assign stack(12) = null
```

The most apparent inefficiency here is that the final value of `i` can be computed at compile time since all values and types are known beforehand. Yet, the compiler does not perform this optimization.

Another issue is the excessive use of the stack. For instance, starting at byte `15`, the result of multiplication is stored in a temporary slot (`stack(11)`) and then copied to `stack(6)`. This redundant assignment could be eliminated by storing the result directly in `stack(6)`.

Similar issues arise in more complex cases, such as:

```gd
var pos = position
position.y += 1
```

The bytecode:
```
 0: line 4: var pos = position
 2: get_member stack(5) = ["position"]
 5: assign stack(4) = stack(5)
 8: line 6: position.y += 1
10: get_member stack(5) = ["position"]
13: get_named validated stack(7) = stack(5)["y"]
17: validated operator stack(6) = stack(7) + const(1)
22: set_named validated stack(5)["y"] = stack(6)
26: set_member ["position"] = stack(5)
```

Here, accessing `position` and modifying `y` involves redundant copying. The VM cannot cache references to `position` and instead always operates on a copy.

## Can This Be Fixed?

Absolutely **yes**. Addressing these inefficiencies is a key goal of developing a JIT compiler.

For now, I envision two main stages for designing and implementing the JIT compiler:

- **Stage 1**: Define basic goals, establish code generation criteria, and apply optimizations for common cases.
- **Stage 2**: Introduce more complex optimizations, including nativeizing trivial types, optimizing method calls, and accessing structure fields.
- **Stage 3 (?):** Explore deeper and more advanced optimizations.

### Stage 1

The primary task is compatibility with `Variant` to maintain the ability to work with a *thick* dynamic API and to implement the native handling of both utility code and native types: `Variant::INT`, `Variant::FLOAT`, `Variant::BOOL`.

By utility code, I mean various instructions for working natively with the `Variant` class, as well as code for preparing arguments before API function calls, runtime safety, etc.

To more accurately and safely define the mechanism of lazy loading and other potential constraints, I also identify three main groups of types:

*Native* :
- `int`/`int64_t`
- `float`/`double`
- `bool`

*Trivial/Builtin* :
- `Vector2`
- `Vector2i`
- `Vector3`
- `Vector3i`
- `Vector4`
- `Vector4i`
- ...

*Other* :
- `Object`
- `Transform`
- ...

Other types include all types stored in `Variant` as pointers and requiring more complex logic, which I will leave for later and not address at this stage.

> Although `Variant` represents the `FLOAT` and `INT` types as `double` and `int64_t` in C++, support for and work with the `float` and `int` types will be necessary in the future for structures like `VectorXX`, etc., which contain fields of these types.

I also include lazy loading/usage of data in `stage 1`. At the code generation level, we cannot control the invocation of operations themselves, such as executing an addition whose result is not subsequently used (dead code)—this is the compiler's responsibility. GDScript seems to analyze and identify unused variables, but the compiler nonetheless performs full compilation without eliminating dead code. Therefore, our task focuses on lazy code generation for accessing `Variant` values and objects.

I intentionally highlighted lazy loading for the following reason: for storing intermediate/local values, it will be preferable to use the stack, as implemented in the current VM, and to store full-fledged `Variant` objects on the stack, which will inevitably be needed for various external function calls.

To optimally work with `Variant::INT`, `Variant::FLOAT`, and `Variant::BOOL` values, processor registers and corresponding instructions should be used—this will significantly speed up script execution. However, each value must also be accessible as a `Variant` object when passed to an external function. Therefore, whenever possible, all calculations are performed directly with the values, and when a `Variant` object is required, it is constructed on the stack and passed by pointer. Similarly, *unpacking* a value from a `Variant` occurs as follows: for example, if a parameter `delta: float` (a full-fledged object) is passed to a function, retrieving the `float` value (which is actually a `double`/`64-bit float`) requires loading the `double` from memory at `variant_ptr + offsetof(data)`, where `variant_ptr` is a pointer to a `Variant` object, and `offset(data)` is the offset to the field containing the value.

The `Variant` class looks roughly like this:
```cpp
class Variant {
private:
    enum Type {
        ...
    };

    Type type;

    ...
    union {
        ...
    } alignas(8) _data;
    ...
};
```

Thus, constructing `Variant` objects for native and trivial types boils down to simple memory operations. This applies to many mathematical data types like `Vector2`, `Vector3`, etc., but I am leaving them for `stage 2` since they also require handling field access (`vec.x/y/z`), more complex operations like multiplying a vector by a scalar or adding two vectors, as well as native method calls like `max(...)`, `length()`, etc.

Generally, values operated on by the compiled function can reside on the stack, in registers (only for native types), or in external memory. In GDScript's implementation, when calling `GDScriptFunction::call(...)`, arguments are passed as an array of `Variant` objects. Constants are stored in the `GDScriptFunction::constants` array, and temporary values and local variables are allocated on the stack.

Similarly, various use cases can be classified depending on the data source, its type, and the access method (by value or pointer). Access can be by value (for native types, meaning data is stored in registers), by pointer to data (only the `Variant::_data` field is initialized), or by pointer to the entire `Variant` object (a full object is constructed with the `type` and `_data` fields initialized).

The main distinction between temporary and local (native) values is that they do not necessarily need to exist on the stack—this is only required when accessed by pointer. Depending on whether a pointer to the entire object or just the data is required, the `Variant::type` field may remain uninitialized for optimization purposes. Native constants also do not need to be added to the `GDScriptFunction::constants` vector unless the constant is passed as an argument, as they can be encoded directly into instructions as immediate values.

Based on these criteria, I compiled tables characterizing certain patterns between values and types:

| access\type | *Native* | *Builtin* | *Other* |
|---|---|---|---|
| by value (*immediate*/*registers*) | [x] | [-] | [-] |
| by pointer to a value | [x] | [x] | [-] |
| by pointer to an object | [x] | [x] | [x] |

Here, a pointer to a value refers to a pointer to the `Variant::_data` field. This is relevant for structures embedded in `Variant`.

||changeable|allocatable|source|
|---|---|---|---|
|constant| **NO** | **YES** (in `constants` vector) | *immediate*/*memory* |
|function argument| **VALUE** | **NO** | *memory* |
|local| **VALUE** | **YES** (on a stack) | *registers*/*memory* |
|temporary| **TYPE** and **VALUE** | **YES** | *registers*/*memory* |
|external| **VALUE** (?) | **NO** | *memory* |

The primary idea of lazy loading is as follows:

1. Value loading is deferred (cached) by the code generator.
2. When the value is accessed, depending on the type of access, memory may need to be allocated on the stack to construct a `Variant` object, or the cached/used value may be returned directly (*immediate*/*registers*).

Temporary values are requested by the compiler and can be released by calling `GDScriptCodeGenerator::pop_temporary()`. To save stack space, I implemented a simple mechanism that reuses previously allocated temporary stack values (likely already containing constructed objects). However, the type of the temporary object may also change, which needs to be tracked during compilation to load the correct value into memory in the `Variant::type` field when accessed via a pointer to the object.

**External C++ API**

The **bunny-jit** backend natively supports a C/C++ ABI out of the box and uses the appropriate calling convention depending on the platform (Windows/Unix), so there is no need to worry about this. The main task here is obtaining the address of the required function and preparing the arguments.

Preparing arguments is similar to the current VM implementation, but now done natively. Due to high-level abstraction, method or function arguments are often passed as an array of `Variant` object pointers (i.e., `const **Variant`). This array is also allocated on the stack and filled with the necessary values just before the function call, after which the stack space can be reused for storing temporary and local values.

However, the main issue is that C++ does not provide a safe way to obtain the address of a method. GCC has an extension for this, but it is not implemented in clang. It is also not possible to extract a virtual method pointer from the `vtable` because C++ does not provide safe and reliable mechanisms to determine the offset of the `vtable` pointer inside the object or the index of the method in the table. Thus, for methods, some wrappers must be implemented as regular static functions, and pointers to them are used to ensure ABI compatibility.

Other functions can be called directly. Additionally, for many GDScript calls, it is possible to obtain a pointer to a utility/native function or operator function, embedding the address in the code as an immediate value to avoid retrieving the pointer via a hash table at runtime (as the current VM mostly does). Although the VM has some optimization that stores validated pointers in `GDScriptFunction` arrays, embedding addresses in the code will be a significant optimization.

### Stage 2

For `stage 2`, the main task is the native handling of trivial data structures and mathematical types, primarily vectors. At this stage, loop native handling can also be included.

Optimizing structures stored in `Variant` as pointers is also possible, but when creating/deleting an object, dynamic memory allocation and deallocation are required. This applies to types like `Variant::TRANSFORM`, `Variant::OBJECT`, `Variant::AABB`, and others. However, I have not conducted sufficient analysis of possible optimizations for them and left this for the future (perhaps this will be `stage 3`?).

All structure optimizations rely on directly accessing and working with fields instead of calling `Variant` object methods. For example, replacing calls to `get_named("y")` and `set_named("x")` with simple `mov` instructions that load and write values from/to memory. This also includes operator calls, which can be fully implemented natively. While the backend `bunny-jit` does not yet fully support SIMD, this may change in the future.

During the development of the code generator, I have not yet completed the native handling of trivial data structures, so I will describe this stage in more detail in the next post.

## What about the implementation?

At the moment, I have some code, but I often experiment and change approaches, so it is more of a draft for now. I am currently refactoring it and writing a cleaner version. If you are interested, [the repository](https://github.com/bagggage/godot/tree/gdscript_jit) is available on GitHub. I will try to update it as often as possible and keep it up to date. It already includes `bunny-jit`, and you just need to perform a standard Godot build. All jit specific code is located in the `modules/gdscript/gdscript_jit_*.*` files. Enjoy studying it! :)
