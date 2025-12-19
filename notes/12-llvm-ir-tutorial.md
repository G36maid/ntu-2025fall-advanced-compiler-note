# Bridgers' LLVM IR Tutorial

Notes from the presentation "Tutorial: Bridgers' LLVM IR tutorial" by Vince Bridgers and Felipe de Azevedo Piovezan at the LLVM Developers Conference – Brussels 2019.

## About this tutorial

- Assumes no previous Intermediate Representation (IR) knowledge.
- Not a lecture about compiler theory.
- After the tutorial, you should:
    - Understand common LLVM tools.
    - Be able to write simple IR.
    - Be able to understand the language reference.
    - Use it to inspect compiler-generated IR.

## What is the LLVM IR?

The LLVM Intermediate Representation:
- Is a low-level programming language (RISC-like instruction set).
- Can represent high-level ideas (high-level languages can map cleanly to IR).
- Enables efficient code optimization.

### IR representation

- **Bitcode file:** `*.bc` (binary representation)
- **Human-readable file:** `*.ll` (assembly-like text format)

The `llvm-as` tool assembles `.ll` files into `.bc` files, and `llvm-dis` does the reverse.

This tutorial focuses on the human-readable `.ll` format.

```/dev/null/example.ll#L1-4
def void @foo(i32 %arg) {
  ; You can read me!
  ret void
}
```

## IR & the Compilation Process

A typical compilation flow with LLVM might look like this:

1.  **Frontend (Clang):** C/C++ source code (`.c`, `.cpp`) is compiled into LLVM bitcode (`.bc`).
2.  **Optimizer (opt):** A series of optimization passes are run on the bitcode.
3.  **Linker (llvm-link):** Multiple bitcode files can be linked together.
4.  **Backend (llc):** The optimized bitcode is compiled into target-specific machine code.

## Simplified IR Layout

An LLVM module has a hierarchical structure:

- **Module**
    - Target information (`target datalayout`, `target triple`)
    - Global symbols
        - Global Variables
        - Function declarations/definitions
- **Function**
    - Arguments
    - Basic Blocks
- **Basic Block**
    - Label
    - Phi instructions
    - Instructions
    - Terminator Instruction

### Target Information

An IR module usually starts with strings describing the target machine:

```/dev/null/target.ll#L1-2
target datalayout = "e-m:e-i64:64-f80:128-n8:16:32:64-S128"
target triple = "x86_64-unknown-linux-gnu"
```

These specify details like endianness, pointer sizes, and ABI.

### A Basic Example

Here's C++ code and its corresponding hand-written LLVM IR:

**C++ Code:**
```/dev/null/example.cpp#L1-6
int factorial(int val);

int main(int argc, char** argv)
{
  return factorial(2) * 7 == 42;
}
```

**LLVM IR:**
```/dev/null/example.ll#L1-8
declare i32 @factorial(i32)

define i32 @main(i32 %argc, i8** %argv) {
  %1 = call i32 @factorial(i32 2)
  %2 = mul i32 %1, 7
  %3 = icmp eq i32 %2, 42
  %result = zext i1 %3 to i32
  ret i32 %result
}
```

### Key Concepts in the Example

#### Virtual Registers (`%`)

- Local variables in LLVM IR.
- Can be numbered (`%1`, `%2`) or named (`%result`).
- LLVM IR is in Static Single Assignment (SSA) form, meaning each virtual register is assigned a value only once. There are conceptually an infinite number of them.

#### Strong Typing

- LLVM IR is a strongly typed language.
- Types are specified for all arguments, return values, and instruction operands.
- There are no implicit type conversions. You must use explicit instructions like `zext` (zero-extend) to convert between types.
- The `opt -verify` command can be used to check if an `.ll` file is valid.

#### The Language Reference (LangRef)

The [LLVM Language Reference Manual](https://llvm.org/docs/LangRef.html) is the authoritative source for all instructions, types, and attributes. It's an essential resource for understanding and writing IR.

## Control Flow

### Recursive Factorial Example

**C Code:**
```/dev/null/factorial.c#L1-7
// Precondition: val is non-negative.
int factorial(int val) {
  if (val == 0)
    return 1;
  return val * factorial(val - 1);
}
```

**LLVM IR:**
```/dev/null/factorial.ll#L1-13
; Precondition: %val is non-negative.
define i32 @factorial(i32 %val) {
  %is_base_case = icmp eq i32 %val, 0
  br i1 %is_base_case, label %base_case, label %recursive_case

base_case:
  ret i32 1

recursive_case:
  %1 = add i32 -1, %val
  %2 = call i32 @factorial(i32 %1)
  %3 = mul i32 %val, %2
  ret i32 %3
}
```

### Basic Blocks

- A sequence of non-terminator instructions followed by exactly one terminator instruction.
- **Terminator instructions** transfer control flow. Examples include:
    - `ret`: Return from a function.
    - `br`: Conditional or unconditional branch to another basic block.
    - `switch`: Multi-way branch.
    - `unreachable`: Indicates a point that should never be reached.
- The first basic block in a function is the `entry` block.
- Each basic block has a label, which can be explicit (`base_case:`) or implicit (`%0:`).

### Control Flow Graph (CFG)

The connections between basic blocks form the CFG. You can visualize it using `opt`:
- `opt -analyze -dot-cfg <input.ll>`: Generates a `.dot` file including instructions.
- `opt -analyze -dot-cfg-only <input.ll>`: Generates a `.dot` file without instructions.

## Static Single Assignment (SSA) and Loops

Because every variable is assigned exactly once, traditional loops with mutating variables require a special construct.

### Iterative Factorial Example

**C Code:**
```/dev/null/iterative_factorial.c#L1-6
int factorial(int val) {
  int temp = 1;
  for (int i = 2; i <= val; ++i)
    temp *= i;
  return temp;
}
```

A direct translation would violate SSA rules by re-assigning `%i` and `%temp`.

### The `phi` Instruction

The `phi` instruction is used to select a value based on which predecessor basic block control came from. This is how SSA form is maintained in the presence of loops and branches.

`<result> = phi <type> [ <val0>, <label0> ], [ <val1>, <label1> ], ...`

**LLVM IR with `phi`:**
```/dev/null/iterative_factorial.ll#L1-15
define i32 @factorial(i32 %val) {
entry:
  br label %check_for_condition

check_for_condition:
  %current_i = phi i32 [ 2, %entry ], [ %i_plus_one, %for_body ]
  %temp      = phi i32 [ 1, %entry ], [ %new_temp, %for_body ]
  %i_leq_val = icmp sle i32 %current_i, %val
  br i1 %i_leq_val, label %for_body, label %end_loop

for_body:
  %new_temp = mul i32 %temp, %current_i
  %i_plus_one = add i32 %current_i, 1
  br label %check_for_condition

end_loop:
  ret i32 %temp
}
```

### Memory Operations (`alloca`, `store`, `load`)

Another way to handle mutable state is by using memory. Frontends often generate this "unoptimized" form first.
- `alloca <type>`: Allocates memory on the stack frame. Returns a pointer.
- `store <type> <value>, <type>* <pointer>`: Writes a value to memory.
- `load <type>, <type>* <pointer>`: Reads a value from memory.

The `mem2reg` optimization pass in `opt` can often convert this stack-based IR into the `phi`-based SSA form, which is more amenable to other optimizations.

## Global Variables

- Global variables are declared at the module level.
- Their names are prefixed with `@`.
- They are always pointers.
- They can be `global` (mutable) or `constant`.

```/dev/null/global.ll#L1-4
@gv = global i16 46
; ... Inside a function:
%val = load i16, i16* @gv
store i16 0, i16* @gv
```

## Aggregate Types

### Arrays

- Defined by a constant size and an element type.
- Example: `[17 x i8]` is an array of 17 bytes.

### Structs

- A collection of heterogeneous types.
- Can be named for clarity.
- Example: `%MyStruct = type { i8, i32, [3 x i32] }`

## Pointers and the `getelementptr` (GEP) Instruction

The `getelementptr` (GEP) instruction is used for pointer arithmetic. It calculates pointer offsets without accessing memory.

`<result> = getelementptr <type>, <type>* <ptr>, [ <index_type> <index> ]+`

### GEP Fundamentals

1.  **First Index:** The first index always steps through the base pointer `<ptr>`. The `<type>` of the instruction determines the size of the step. It does *not* change the resulting pointer type.
2.  **Further Indices:** Subsequent indices step *inside* aggregate types (arrays or structs). They change the resulting pointer type.
3.  **Struct Indices:** Indices into structs must be constants.
4.  **No Memory Access:** GEP only performs address calculations; it never loads from or stores to memory.

**Example: Accessing a struct field**

```/dev/null/gep_struct.ll#L1-6
%MyStruct = type { i8, i32 }
@my_global = global %MyStruct { i8 1, i32 2 }

; Inside a function:
; Get a pointer to the i32 field of @my_global
%ptr_to_i32 = getelementptr %MyStruct, %MyStruct* @my_global, i32 0, i32 1
```

- `i32 0`: Dereferences the `%MyStruct*` pointer. We want the first (0th) element pointed to by `@my_global`.
- `i32 1`: Selects the second field (index 1) of the `%MyStruct` type.

## Final Remarks

- LLVM IR is constantly evolving.
- This tutorial covered fundamental topics that are unlikely to change soon.
- Further exploration:
    - Constants and constant expressions
    - Intrinsics
    - Metadata
    - Vector instructions
- Next steps: Learn to manipulate IR programmatically using the LLVM libraries.
