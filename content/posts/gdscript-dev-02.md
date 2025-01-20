+++
date = '2025-01-21T01:48:02+03:00'
title = 'GDScript codegen research: 02'
tags = ['gdscript','asm','c++','jit']
+++

# Again About `stage 2`

In the previous post, I briefly described `stage 2` but left out many important aspects, such as loops, arrays, and branching. Why? Because each of these subtopics represents a significant amount of work. I also consider all these subtopics part of `stage 2`, and as development progresses, I will delve deeper into this topic and enhance the code generation to support all these features in the JIT compiler.

> While branching with `if`/`else` is relatively straightforward to implement, we’ll discuss this in more detail next time.

In this post, I will continue the discussion of trivial (or not-so-trivial) structures, including field access, method calls, and specialized operators.

# Overview of JIT API/ABI

Let’s start with a quick breakdown of the current code generator implementation.

## JIT-Compiled Function

The current implementation assumes the compilation of a function that uses a C++ ABI-compatible calling convention for argument passing and invocation. The function prototype in C++ might look something like this:

```cpp
void jit_func(Object* p_script_instance, Variant* p_members, const Variant* p_constants, const Variant** p_args);
```

- `p_script_instance`: A pointer to the `Object` retrieved as `p_instance->owner` from `gdscript_vm.cpp`. It’s used for calling instance methods, accessing fields, and more.
- `p_members`: A pointer to the array of all members of the current `GDScriptInstance`. This speeds up addressing the required member in the script since the position (index) of the desired field is known in advance. This includes only fields declared within the script (if I’m not mistaken).
- `p_constants`: A pointer to an array of constants that, one way or another, require their address to be taken within the function.
- `p_args`: An array of pointers to the function arguments. This is a common approach in Godot’s codebase for making generalized function calls, where the types and arguments are only known at runtime.

## Code Generator

All its code is located in [`modules/gdscript/gdscript_jit_codegen.*`](https://github.com/bagggage/godot/blob/gdscript_jit/modules/gdscript/gdscript_jit_codegen.cpp), while helper functions and other supporting code can be found in [`modules/gdscript/gdscript_jit.*`](https://github.com/bagggage/godot/blob/gdscript_jit/modules/gdscript/gdscript_jit.h).

As mentioned in the previous post, I implemented the `ValueRef` structure, which stores all the necessary information about a value being used during code generation. Based on `ValueRef`, lazy loading and memory storage for native types have been implemented. Below are some of the main functions in the code generator tied to `ValueRef`:

- `get_value_ref(const Address& p_address)`: Returns a reference to a `ValueRef`.
- `try_alloc_variant_object(ValueRef& p_value)`: Allocates a `Variant` object for `ValueRef` if it hasn’t been allocated yet. For local values, allocation happens on the stack; for constants, the object is allocated in the `GDScriptFunction::constants` vector.
- `emit_ptr_to_object(ValueRef& p_value)`: Generates code to obtain a pointer to a `Variant` object, which is allocated and constructed if it hasn’t been already.
- `emit_ptr_to_data(ValueRef& p_value)`: Similar to `emit_ptr_to_object(...)`, but generates code to obtain a pointer to the `Variant::_data` field.
- `emit_cast_native(ValueRef& p_value, const Variant::Type p_target_type)`: Generates code for converting a value from one native type to another.
- `emit_get_native(ValueRef& p_value)`: Generates code to retrieve a value for a native type `ValueRef`.
- `emit_set_native(ValueRef& p_value, const bjit::Value p_value)`: Lazily loads a value into the data field for a native type.
- `emit_load_ptr_args(const Vector<Address>& p_args)`: Generates code to create an array of pointers to the `Variant` objects listed in `p_args` on the stack, and returns the address of this array. This is used for passing arguments to external functions that accept `const Variant** p_args`.

# Small Example of the Result

It’s still too early to compare or discuss performance, but here’s a small example of compiling simple GDScript code:

```gd
func _init():
    var hf := 2.0
    var sf := 2.5
    var i := 2
    i *= hf
    i *= sf
    
    print(i)
```

**VM Opcodes:**

```
 0: line 4: 	var hf := 2.0
 2: assign stack(3) = const(2.0)
 5: line 5: 	var sf := 2.5
 7: assign stack(4) = const(2.5)
10: line 6: 	var i := 2
12: assign stack(5) = const(2)
15: line 7: 	i *= hf
17: validated operator stack(6) = stack(5) * stack(3)
22: assign typed builtin (int) stack(5) = stack(6)
26: line 8: 	i *= sf
28: validated operator stack(6) = stack(5) * stack(4)
33: assign typed builtin (int) stack(5) = stack(6)
37: line 10: 	print(i)
39: call-utility stack(7) = print(stack(5))
45: assign stack(7) = null
```

**x86-64 Machine Code (Disassembled):**

```asm
sub rsp, 0x28
movsd xmm0, qword ptr [rip + 0x5c]
movsd xmm1, qword ptr [rip + 0x5c]
mov eax, 2
cvtsi2sd xmm2, rax
mulsd xmm2, xmm0
cvttsd2si rcx, xmm2
cvtsi2sd xmm0, rcx
mulsd xmm0, xmm1
cvttsd2si rcx, xmm0
mov dword ptr [rsp], eax
mov qword ptr [rsp + 8], rcx
lea rax, [rsp + 0x18]
mov rcx, rsp
mov qword ptr [rax], rcx
mov edx, 1
movabs r8, 0x55555d612230
xor edi, edi
mov rsi, rax
mov rax, r8
call rax
xor eax, eax
add rsp, 0x28
ret
```

Here is the result of the JIT code generator! Some bottlenecks are evident. I can immediately point out that `bunny-jit` applies certain optimizations but often leaves redundant `mov` instructions between registers without changing values, which seems unnecessary. While some issues can be fixed at the compiler level, many redundant operations stem from the lack of analysis at the compilation level rather than the code generation level.

**Step-by-Step Code Analysis:**

1. **Constant Loading:**
*Double* and *int64_t* constants are lazily used, avoiding storing constants in memory and reloading them into registers.  
```asm
movsd xmm0, qword ptr [rip + 0x5c] ; var hf := 2.0
movsd xmm1, qword ptr [rip + 0x5c] ; var sf := 2.5
mov eax, 2                         ; var i := 2
```

2. **First Multiplication (`i *= hf`):**
```asm
cvtsi2sd xmm2, rax  ; cast int64_t to double
mulsd xmm2, xmm0    ; multiply: i:(2) * hf:(2.0)
cvttsd2si rcx, xmm2 ; cast and store result: i = result:(4)
```

3. **Second Multiplication (`i *= sf`):**
Similar to the first.  
```asm
cvtsi2sd xmm0, rcx
mulsd xmm0, xmm1
cvttsd2si rcx, xmm0
```

4. **Preparing for `print` Call:**
A `const Variant** p_args` array is created, containing a single pointer to a `Variant` object with type `Variant::INT` and the computed value.  
```asm
mov dword ptr [rsp], eax     ; Variant::type = eax:(2 == Variant::INT)
mov qword ptr [rsp + 8], rcx ; Variant::_data = rcx
lea rax, [rsp + 0x18]        ; rax = p_args
mov rcx, rsp                 ; rcx = &variant_object
mov qword ptr [rax], rcx     ; rax(p_args)[0] = rcx(&variant_object)
```

It’s worth noting how cleverly `bunny-jit` reused `eax`, which contains the constant `2`. This matches both the value of `var i := 2` and the enumeration value `Variant::INT`.

5. **`print` Call:**
```asm
mov edx, 1                ; edx (p_args_count) = 1
movabs r8, 0x55555d612230 ; r8 = function pointer
xor edi, edi              ; rdi (p_ret) = nullptr (`print` doesn’t return a value)
mov rsi, rax              ; rsi (p_args) = rax
mov rax, r8
call rax                  ; call: print(i)
```

---

And finally, it works! I get the expected output of `10` when running the game in the editor. You can try it yourself — [source code here](https://github.com/bagggage/godot/tree/gdscript_jit).

Time to move forward.

# Native Structs Implementation

I would like to remind you that the Godot developers, for optimizing `Variant`, decided to embed many basic frequently used types directly into the `Variant` class. This is achieved through the `union` of the `Variant::_data` field, which combines various different types. The actual data stored in this field is determined by the type stored in the `Variant::type` field. For built-in structures and native types, the data and structures themselves are stored directly in the `Variant::_data` field. This serves as the starting point for my native struct implementation.

The only drawback in my case is the need to reimplement all the corresponding structures and their operators, but now through machine code generation. As a result, we will have two implementations of `Vector3`—one in C++ (`core/math/vector3.cpp`) and another native one for JIT GDScript. This approach avoids external calls for various operations, such as `Vector3 * float`, by embedding the vector-multiplication operation directly into the machine code of the function, thereby speeding up calculations.

I believe it will be useful to have some generalized information about the types. Therefore, I implemented the `TypeInfo` structure, which provides the necessary information about types from a code generation perspective. It also links the JIT type with its native C++ type and the `Variant::Type` (if applicable). For supporting structs and classes, it also contains information about fields in the form of their types and offsets relative to the beginning of the structure. This will simplify the code generator when adding support for structs in GDScript [*(Proposal #7329)*](https://github.com/godotengine/godot-proposals/issues/7329). 

I initially avoided adding a custom structure to describe data types, as Godot already provides a robust type system. However, this system is tailored for C++ code and lacks information about field offsets within classes/structs, which is crucial for code generation. Additionally, I may need to store information about operators and possibly even methods that can also be fully represented in the machine code of the function. Such information cannot be obtained using existing approaches. Moreover, some useful data is hidden under the `private` modifier and is not exposed through the API. As for `GDScriptDataType`, it does not fully suit this use case, although it could be used as a base. For now, I will leave things as they are.

The following tasks should be implemented:

1. *Fast access to fields*  
2. *Binary/unary operators*  
3. *Native methods implementation*  

## Accessing Fields

This part seems relatively simple. It's enough to get type information and, if it's a *builtin* structure, implement access to the field via a pointer. To do this, it’s necessary to generate the corresponding information about the structure's fields and their types.

For example:  
```gd
position.y += 1
```

The current set of VM opcodes:
```
 0: line 11: 	position.y += 1
 2: get_member stack(3) = ["position"]
 5: get_named validated stack(5) = stack(3)["y"]
 9: validated operator stack(4) = stack(5) + const(1)
 14: set_named validated stack(3)["y"] = stack(4)
 18: set_member ["position"] = stack(3)
```

Now, let's focus on field access for `y`. For the VM, this code is quite costly because it constructs temporary objects on the stack. With JIT, if the value of field `y` is accessed via a pointer (just pointer to `Vector3` struct + offset to field `y`), lazy allocation of `Variant` ensures that all operations are performed directly in processor registers and the result is then stored back into memory. However... wait... not immediately. The compiler forces us to first save the new `y` in our temporary copy and then assign it back to `position`.

However, there's a small caveat. `Vector3`, like `Vector4` and `Vector2`, contains fields of type `float`, while `Variant::FLOAT` is represented as `double`. This needs to be accounted for, and it may result in the generation of several overhead instructions for converting `float` to `double` and back. This is not ideal, but unavoidable.

## Binary/Unary Operators

I have already implemented code generation for binary/unary operators for native types like `bool`, `int64_t`, and `double` (aka `Variant::BOOL`, `Variant::INT`, and `Variant::FLOAT`). I achieved this manually, with some use of C++ templates and macros, and I hope it’s not overly verbose:

```cpp
template<typename L, typename R>
struct NativeBinaryOperators {
    using RetT = typename ReturnType<L, R>::V;
    static constexpr bool is_float = std::is_same_v<RetT, double> || std::is_same_v<RetT, float>;
	...
    static bjit::Value add(bjit::Proc& proc, bjit::Value lhs, bjit::Value rhs) {
        const bjit::Value casted_lhs = cast_l(proc, lhs);
        const bjit::Value casted_rhs = cast_r(proc, rhs);
        return is_float ? proc.dadd(casted_lhs, casted_rhs) : proc.iadd(casted_lhs, casted_rhs);
    }
    static bjit::Value sub(bjit::Proc& proc, bjit::Value lhs, bjit::Value rhs) {
        const bjit::Value casted_lhs = cast_l(proc, lhs);
        const bjit::Value casted_rhs = cast_r(proc, rhs);
        return is_float ? proc.dsub(casted_lhs, casted_rhs) : proc.isub(casted_lhs, casted_rhs);
    }
    static bjit::Value mul(bjit::Proc& proc, bjit::Value lhs, bjit::Value rhs) {
        const bjit::Value casted_lhs = cast_l(proc, lhs);
        const bjit::Value casted_rhs = cast_r(proc, rhs);
        return is_float ? proc.dmul(casted_lhs, casted_rhs) : proc.imul(casted_lhs, casted_rhs);
    }
    static bjit::Value div(bjit::Proc& proc, bjit::Value lhs, bjit::Value rhs) {
        const bjit::Value casted_lhs = cast_l(proc, lhs);
        const bjit::Value casted_rhs = cast_r(proc, rhs);
        return is_float ? proc.ddiv(casted_lhs, casted_rhs) : proc.idiv(casted_lhs, casted_rhs);
    }
	...
};
```

Next, I store the function pointers for operator code generation in a table:  
```cpp
static BinaryOperatorCodeGenFunc binary_operators_table[Variant::VARIANT_MAX][Variant::VARIANT_MAX][Variant::OP_MAX];
```

This is somewhat similar to the operator table in the `Variant` class. So far, everything works, but writing all the necessary code for generating all possible operator combinations for different structures manually seems unpromising. Besides partial duplication of code, having numerous pointers to various functions means that all this code will end up in the final binary file, potentially increasing its size. This is due to GDScript's decision to crudely embed these standard types at a high level, which is beneficial for the VM but quite inconvenient for machine code generation. 

It would be great to have something like an IR (Intermediate Representation) for GDScript, consisting of simpler operations, making it easier and more efficient to implement machine code generation. But that’s not the case, and we’ll have to deal with it.

So far, I haven’t come up with a good solution to this problem. It’s also necessary to decide whether each operator call should be inlined or if it’s better to make a regular function call. In the latter case, we may already have ready pointers to the required operators from C++ code, which is worth considering.

## Native Methods Implementation

By native methods implementation, I primarily mean using direct method calls without relying on the heavy API currently used by the VM (e.g., `MethodBind`, etc.). As I mentioned in my first post, there doesn’t seem to be a safe way in C++ (or am I mistaken?) to obtain the address of a member function of a given class or structure. Thus, it likely becomes necessary to implement custom versions of the required methods, either as regular static functions (and obtain their address) or generate the machine code for these methods directly via a code generator, which is even more resource-intensive. The latter option offers the advantage of applying a so-called `near call` (on x86-64) for function calls. Beyond this, it remains a headache. 

Since Godot doesn’t provide pointers to the methods themselves, but only pointers to certain wrappers (which typically accept either pointers to values or are heavy wrappers using `Variant`), every individual argument must be unpacked from `Variant`, passed to the method, and then the result, if any, is re-packed into a `Variant` object.

This raises a broader question about the native implementation of any method calls in general, not just for `builtin` structures. This process doesn’t differ much from that of other native Godot classes. The only differences lie in scripts and other features, like extensions, which may require special handling and usually demand the use of the same "heavy" API.

# That’s All for Now

For now, this is all I wanted to describe and share. I’ve achieved some results in generating code for native types such as `double`, `bool`, and `int64_t`. However, due to the high-level operations of GDScript, most functionality inevitably boils down to calling external functions that require constructing `Variant` objects and incurring other overheads. I believe these factors consume the majority of performance.

In the end, I’ve been considering changes to the Core to provide a lightweight API for function calls—for example, directly via pointers with raw values as arguments instead of `Variant` objects.

That’s all for now. Next time, I plan to try addressing some of these issues or at least exploring possible solutions and attempting their implementation. The current implementation and all the code can still be found on [GitHub](https://github.com/bagggage/godot).
