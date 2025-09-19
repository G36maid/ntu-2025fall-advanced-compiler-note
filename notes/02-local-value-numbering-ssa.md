# Local Value Numbering & SSA
CSIE 5054: Advanced Compiler Design
Instructor: Shih-Wei Liao

## Table of Contents
*   Local Value Numbering (LVN)
    *   Local Optimizations
    *   Value Numbering
    *   Summary
*   Static Single-Assignment Form (SSA)
    *   Building SSA
    *   Homework 4
    *   SSA construction for Bril

## Local Optimizations

### Dead Code Elimination (DCE)
**Original Code:**
```dce_example1.bril#L1-4
@main {
		a: int = const 1;
		a: int = const 2;
		print a;
}
```
**Optimized Code:**
```dce_example1_opt.bril#L1-3
@main {
		a: int = const 2;
		print a;
}
```

### Copy Propagation
**Original Code:**
```copy_prop_example1.bril#L1-5
@main {
		x: int = const 4;
		copy1: int = id x;
		copy2: int = id copy1;
		print copy2;
}
```
**Optimized Code:**
```copy_prop_example1_opt.bril#L1-3
@main {
		x: int = const 4;
		print x;
}
```
*Note: Whether to eliminate dead code or not in Copy Propagation, Constant Folding, and CSE depends on implementation.*

### Constant Folding
**Original Code:**
```const_fold_example1.bril#L1-5
@main {
		a: int = const 1;
		b: int = const 2;
		sum: int = add a b;
		print sum;
}
```
**Optimized Code:**
```const_fold_example1_opt.bril#L1-3
@main {
		sum: int = const 3;
		print sum;
}
```

### Common Subexpression Elimination (CSE)
**Original Code:**
```cse_example1.bril#L1-9
@main {
		a: int = const 4;
		b: int = const 3;
		two: int = const 2;
		t1: int = mul a two;
		t2: int = mul a two;
		t3: int = mul t2 b;
		result: int = add t1 t3;
		print result;
}
```
**Optimized Code:**
```cse_example1_opt.bril#L1-7
@main {
		a: int = const 4;
		b: int = const 3;
		two: int = const 2;
		t1: int = mul a two;
		t2: int = mul t1 b;
		result: int = add t1 t2;
		print result;
}
```
*Note: This CSE example includes elimination. The renaming t3 to t2 is not strictly needed, but it's a good implementation choice to introduce the concept of register space.*

## Value Numbering: Another Abstraction
Compared to a Directed Acyclic Graph (DAG), value numbering is more explicit with respect to values and times. Each value has its own "number." Common subexpressions will have the same value number.

`var2value`: A current map of variables to their associated value numbers, used to determine the value number of the current expression.
For example: `v1 + v2` is translated to `var2value(r1) + var2value(r2)`.

## Value Numbering Algorithm
**For symbolic execution (Lookup found!):**

| Value as number | Value as expression (hashkey `k`) | Var as name (a.k.a. canonical name or compile-time name) |
|---|---|---|
| `1` | `{V1}{OPi}{V2}` | `x` |

**For symbolic execution (Create entry for `k`):**

| Value as number | Value as expression (hashkey) | Var as name (a.k.a. canonical name) |
|---|---|---|
| `1` | `k: {V1}{OPi}{V2}` | `Ti` |
*E.g., `k` can be a textual string `"{V1}{OPi}{V2}"`.*

## Extending the Value Numbering Algorithm

### Commutative Operations
Variants of a commutative expression, such as `a × b` and `b × a`, should receive the same value number. We can sort the operands to ensure these expressions receive the same value number.

### Constant Folding
Marks hash table entries as constant and stores the constant value alongside the value number. It evaluates constant expressions and replaces them with their constant value.

### Algebraic Identities
Simplifies expressions with variables. For example, `x + 0` and `x` should receive the same value number.

Testing for each identity can significantly slow down LVN. Therefore, LVN implementations should organize these tests into operator-specific decision trees (e.g., a series of if-else statements for each operator).

## The Role of Naming
Variable naming can limit the effectiveness of LVN.

**Example:**
On the left: `c <- x + y` cannot be rewritten to `c <- a` because `a` no longer holds value number 2 after the third operation.
On the right: Each operation defines a unique name, which yields the desired result.

*Note: On the left, tracking `var2value` requires extra effort and clutters the basic LVN algorithm. The subscript denotes the instance of variable write (e.g., `a0` is the first write, `a1` is the second).*

## Summary of LVN
*   The Value Numbering Algorithm finds and replaces redundant expressions for CSE and copy propagation within a basic block.
*   The Extended Value Numbering Algorithm handles commutative operations, constant folding, and algebraic identities.
*   Even with extensions, LVN often doesn't directly perform dead code elimination. To eliminate dead code, we can either tweak LVN with more extensions for a specific IR or iteratively run LVN and DCE.
*   To address the naming problem, a simple solution is to apply LVN to code already in Static Single-Assignment (SSA) form.

## Static Single-Assignment Form (SSA)

### Rules
*   **Each assignment defines a unique name:** Any expression is available at any point after being evaluated.
*   **Each use refers to a single name:** A use can be written with a single name rather than a long list of all definitions that might reach it.
*   **𝜙-function:** Merges definitions along multiple control paths into a single one.

### Building SSA
1.  Dominance Frontier
2.  Insert 𝜙-function
3.  Renaming

### Dominance Frontier
**Recap: Dominance**
*   **Definition:** In a Control Flow Graph (CFG) with entry block `b0`, block `bi` dominates block `bj` if and only if `bi` lies on every path from `b0` to `bj`.
*   A block `bi` dominates another block `bj` if and only if block `bi` dominates all predecessors of block `bj`.
*   `DOM(n)`: The set of blocks that dominate block `n`.

**Immediate Dominator**
*   **Definition:** In a CFG, one block `m` ∈ `DOM(n)`, `m ≠ n`, will be closer (has the shortest path length) to `n` than any other block `x` ∈ `DOM(n)`, `x ≠ n`.
*   `IDOM(n)`: The immediate dominator of `n`.

**Dominator Tree**
A tree that encodes the dominance information for a CFG.
*   **Nodes:** Each node in the CFG.
*   **Edges:** Represent immediate dominance.
*   For a block `n`, `DOM(n)` contains precisely the blocks on the path from `n` to the root of the dominator tree.

**Dominance Frontier Definition**
A block `q` ∈ `DF(p)` if, along some path, `q` is one edge beyond the region that `p` dominates. More precisely:
1.  `q` has a CFG predecessor `x` such that `p` ∈ `DOM(x)` (i.e., `p` dominates `x`).
2.  `p` does not strictly dominate `q` (i.e., `p` ∉ (`DOM(q)` − `q`)).
`DF(p)`: The set of blocks that meet these two criteria.

**Dominance Frontier Algorithm:**
*If `p = IDOM(n)`, then `n` does not belong to `DF(p)`, nor to `DF(m)` for any predecessor `m` of `p`.*
*If `p ≠ IDOM(n)`, then `n` belongs in `DF(p)`. It also belongs in `DF(q)` for any `q` such that `q` ∈ `DOM(p)` and `q` ∉ (`DOM(n)` − `n`). The algorithm finds these latter nodes `q` by running up the dominator tree.*

### Insert 𝜙-function
**Global Names:** Variables that are of interest in multiple blocks.
**Variables in algorithm:**
*   `Globals`: A set of global names.
*   `Blocks(x)`: A set of blocks that contain a definition of `x`.
*   `VARKILL`: A set of variables that are defined in block `m`.

*Why global names? A name `x` cannot need a 𝜙-function unless it is live in multiple blocks.*

### Renaming
In the final SSA form, each global name becomes a base name. Individual definitions of that base name are distinguished by a numerical subscript.
**Example:** For a base name `a`, the first definition encountered by the renaming algorithm will be named `a0`.

*Issues: A variable `x` assigned multiple times within a block, even if not a global name, still needs renaming.*

## Homework 4: SSA Construction for Bril
**Deadline:** 2025/11/7 23:59
**References:** Homework 2 Specification, Homework 2 Github Repo

**Goal:** Transform a given non-SSA Bril program into its SSA form.
**Environment Setup:**
*   Bril toolchain (from HW0)
*   Python 3.7+

**Implementation Tasks:**
*   CFG Construction (`cfg.py`)
*   Dominator Computation (`dominance.py`)
*   𝜙-function Insertion (`ssa_construct.py`)
*   Variable Renaming (`ssa_construct.py`)
*   Validation

### DOs and DONTs
*   **DO:** Make sure you thoroughly understand the algorithm before starting.
*   **DO:** Ensure your Student_ID is correctly entered in `student_id.txt` within the `/src` directory.
*   **DO:** Feel free to modify any files *except* for `is_ssa.py`.
*   **DO NOT:** Modify anything other than the allowed files. Any such changes will be considered cheating.
*   Please note that the GitHub Classroom backend can detect changes to restricted files.
*   Please stay away from plagiarism.
*   AI (instead of TA) will provide individual debugging support for coding.
*   The Professor can provide class-level debug support.
*   If you have any questions relating to the homework, please email `llvm@csie.ntu.edu.tw` with the subject line `'[AC-HW2][Summary of Your Issue]'`.
