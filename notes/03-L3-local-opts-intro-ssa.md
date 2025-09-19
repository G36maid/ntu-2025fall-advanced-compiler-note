# Local Optimizations, Intro to SSA (Lecture 3)

## Table of Contents
*   I. Basic Blocks & Flow Graphs
*   II. Abstraction 1: DAG
*   III. Abstraction 2: Value Numbering
*   IV. Intro to SSA

*Based on ALSU 8.4-8.5, 6.2.4 by Phillip B. Gibbons, 15-745: Local Optimizations, Intro to SSA, Carnegie Mellon*

---

## I. Basic Blocks & Flow Graphs

**Basic Block**: A sequence of 3-address statements with the following properties:
*   Only the first statement can be reached from outside the block (no branches into the middle of the block).
*   All statements are executed consecutively if the first one is (no branches out or halts except perhaps at the end of the block).
*   Basic blocks must be **maximal**, meaning they cannot be made larger without violating these conditions.

**Flow Graph**:
*   **Nodes**: Basic blocks.
*   **Edges**: `Bi -> Bj`, if `Bj` can follow `Bi` immediately in some execution.
    *   Either the first instruction of `Bj` is the target of a `goto` at the end of `Bi`.
    *   Or, `Bj` physically follows `Bi`, which does not end in an unconditional `goto`.

### Partitioning into Basic Blocks

1.  **Identify the leader of each basic block**:
    *   The first instruction in the program.
    *   Any instruction that is the target of a jump.
    *   Any instruction immediately following a jump.
2.  A basic block starts at a leader and ends at the instruction immediately before the next leader (or the last instruction in the program).

### Example Flow Graph

Consider the following code:
```example_code.txt#L1-16
i := n-1

L5: if i<1 goto L1

j := 1

L4: if j>i goto L2
t1 := j-1
t2 := 4*t1
t3 := A[t2]
t6 := 4*j
t7 := A[t6]
if t3<=t7 goto L3

A[t2] := t7
A[t6] := t3

L3: j := j+1

goto L4

L2: i := i-1

goto L5

L1:
```
This code can be partitioned into basic blocks, and a flow graph can be constructed where each node is a basic block and edges represent possible control flow transitions.

---

## II. Local Optimizations (within a basic block)

Common local optimizations include:
*   **Common subexpression elimination**: applicable to array expressions, field access in records, and access to parameters.

### Graph Abstractions: Directed Acyclic Graph (DAG)

**Example 1**:
Given an expression like `a+a*(b-c)+(b-c)*d`
A **Parse Tree** represents the syntactic structure.
An **Expression DAG** can represent the computations, where common subexpressions like `b-c` are represented only once.

**Optimized code from DAG**:
```optimized_dag_code.txt#L1-5
t1 = b - c
t2 = a * t1
t3 = a + t2
t4 = t1 * d
t5 = t3 + t4
```

### How well do DAGs hold up across statements?

**Example 2**:
```dag_issue_example.txt#L1-4
a = b+c;
b = a-d;
c = b+c;
d = a-d;
```
If we try to optimize this with a simple DAG, `c = b+c` might appear as a common subexpression with `a = b+c`. However, the value of `b` changes between these two statements.

**Optimized code (potentially incorrect)**:
```potentially_incorrect_opt.txt#L1-3
a = b+c;
d = a-d;
c = d+c; // Is this correct?
```
This depends on whether `b` is live on exit from the block.

### Critique of DAGs

*   **Cause of problems**: Assignment statements where the value of a variable depends on **time**.
*   **How to fix?**:
    *   Build the graph in order of execution.
    *   Attach the variable name to its latest value.
*   **Result**: The final graph created is not very interesting. The key aspect becomes the `variable -> value` mapping across time, which loses the appeal of the DAG abstraction.

---

## III. Value Numbering: Another Abstraction

(John Cocke & Jack Schwartz in unpublished book: “Programming Languages and their Compilers”, 1970)

*   More explicit with respect to **VALUES** and **TIME**.
*   Each value has its own "number" – common subexpressions mean the same value number.
*   `var2value`: A current map of variables to their values. This is used to determine the value number of the current expression.
    *   Example: `r1 + r2` is translated to `var2value(r1) + var2value(r2)`.

### Value Numbering: Expression Example

Expression: `a+a*(b-c)+(b-c)*d`

Optimized code using value numbering:
```optimized_vn_code.txt#L1-5
t4 = b - c
t5 = a * t4
t6 = a + t5
t8 = t4 * d
t9 = t6 + t8
```

### Value Numbering Algorithm

**Data Structure `VALUES`**: A table storing expression details and the variable currently holding that expression.
```
VALUES = Table of {
  expression   /* [OP, valnum1, valnum2] */
  var          /* name of variable currently holding expr */
}
```

**Algorithm for each instruction (e.g., `dst = src1 OP src2`) in execution order**:
1.  `valnum1 = var2value(src1); valnum2 = var2value(src2)`
2.  **IF** `[OP, valnum1, valnum2]` is in `VALUES`:
    *   `v =` the index of the expression.
    *   Replace instruction with: `dst = VALUES[v].var`
3.  **ELSE**:
    *   Add `expression = [OP, valnum1, valnum2]` and `var = tv` (where `tv` is a new temporary) to `VALUES`.
    *   `v =` index of the new entry.
    *   Replace instruction with: `tv = VALUES[valnum1].var OP VALUES[valnum2].var`
    *   `dst = tv`
4.  `set_var2value(dst, v)`

### More Details

*   **Initial values of variables**: These are the values at the beginning of the basic block.
*   **Possible implementations**:
    *   Initialization: Create "initial values" for all variables.
    *   Dynamically create them as they are used.
*   **Implementation of `VALUES` and `var2value`**: Typically hash tables.

### Value Numbering: Basic Block Example

Consider the sequence:
```vn_bb_example_original.txt#L1-4
a = b+c
b = a-d
c = b+c
d = a-d
```
After value numbering and optimization:
```vn_bb_example_optimized.txt#L1-6
t4 = b + c  // corresponds to value number for (b+c)
a = t4
t5 = t4 - d // corresponds to value number for (a-d) after a becomes t4
b = t5
t6 = t5 + c // corresponds to value number for (b+c) after b becomes t5
c = t6
d = t5
```
**Question**: Assigning to a temporary and then copying to the destination increases the number of instructions—so why do it?
**Answer**: If `dst` is overwritten later, we would lose the opportunity to eliminate common subexpressions since no variable would hold the result.

### Question: Constant Folding

**How do you extend value numbering to constant folding?**
Example:
```constant_folding_q.txt#L1-3
a = 1
b = 2
c = a+b
```
**Answer**: Can add a field to the `VALUES` table indicating when an expression is a constant and what its value is.

### DAGs vs. Value Numbering

**Comparisons of two abstractions**:
*   **DAGs**: Represent computations, good for immediate common subexpression elimination.
*   **Value numbering**:
    *   Explicitly distinguishes between variables and values.
    *   Accounts for **TIME** by keeping dynamic state information.
    *   Involves interpretation of instructions in order of execution.

---

## IV. Intro to SSA (Static Single Assignment)

**Global Optimizations**: Look beyond the basic block.
*   Global versions of local optimizations:
    *   Global common subexpression elimination
    *   Global constant propagation
    *   Dead code elimination
*   Loop optimizations:
    *   Reduce code to be executed in each iteration.
    *   Code motion.
    *   Induction variable elimination.
*   Other control structures:
    *   Code hoisting: eliminates copies of identical code on parallel paths in a flow graph to reduce code size.

*(We will cover these optimizations in later lectures.)*

### Recurring Optimization Theme: Where Is a Variable Defined or Used?

**Example: Loop-Invariant Code Motion**
```loop_invariant_example.txt#L1-4
...
A = B + C
E = A + D
...
```
*   Are `B`, `C`, and `D` only defined outside the loop?
*   Other definitions of `A` inside the loop?
*   Uses of `A` inside the loop?

**Example: Copy Propagation**
```copy_propagation_example.txt#L1-4
X = Y
...
W = X + Z
```
For a given use of `X`:
*   Are all reaching definitions of `X` copies from the same variable (e.g., `X = Y`)?
*   Where `Y` is not redefined since that copy?
*   If so, substitute use of `X` with use of `Y`.

It would be nice if we could traverse directly between related uses and definitions – this would enable a form of sparse code analysis (skip over "don't care" cases).

### Appearances of Same Variable Name May Be Unrelated

```unrelated_names_example.txt#L1-8
X1 = A2 + 1
Y2 = X1 + B

F = 2

X2 = F2 + 7
C2 = X2 + D
```
*   The values in reused storage locations may be provably independent, in which case the compiler can optimize them as separate values.
*   The compiler could use renaming to make these different versions more explicit.

### Definition-Use and Use-Definition Chains

*   **Definition-Use (DU) Chains**: For a given definition of a variable `X`, what are all of its uses?
*   **Use-Definition (UD) Chains**: For a given use of a variable `X`, what are all of the reaching definitions of `X`?

### Unfortunately DU and UD Chains Can Be Expensive

For `N` definitions and `M` uses, this can take `O(NM)` space and time.

**Example**:
```expensive_chains_example.txt#L1-19
foo(int i, int j) {
  ...
  switch (i) {
  case 0: x=3;break;
  case 1: x=1; break;
  case 2: x=6; break;
  case 3: x=7; break;
  default: x = 11;
  }
  switch (j) {
  case 0: y=x+7; break;
  case 1: y=x+4; break;
  case 2: y=x-2; break;
  case 3: y=x+1; break;
  default: y=x+9;
  }
  ...
}
```
Here, `x` has multiple definitions reaching its uses.

**One solution**: Limit each variable to **ONE** definition site.

### Static Single Assignment (SSA)

*   SSA is an Intermediate Representation (IR) where every variable is assigned a value at most once in the program text.
*   **Easy for a basic block (reminiscent of Value Numbering)**:
    *   Visit each instruction in program order.
    *   LHS: assign to a fresh version of the variable.
    *   RHS: use the most recent version of each variable.

**Example:**
**Original:**
```ssa_basic_example_original.txt#L1-5
a = x + y
b = a + x
a = b + 2
c = y + 1
a = c + a
```
**SSA Form:**
```ssa_basic_example_ssa.txt#L1-5
a1 = x + y
b1 = a1 + x
a2 = b1 + 2
c1 = y + 1
a3 = c1 + a2
```

### What about Joins in the CFG?

When control flow merges, a variable might have multiple definitions reaching a single use site.
Example:
```ssa_cfg_join_original.txt#L1-10
c = 12
if (i) {
  a = x + y
  b = a + x
} else {
  a = b + 2
  c = y + 1
}
a = c + a
```
At the merge point before `a = c + a`, `a` could come from two different assignments, and `c` could also come from two different assignments.

**Solution**: Use a notational convention (fiction): a **Φ (Phi) function**.

### Merging at Joins: the Φ function

**Original (simplified):**
```phi_merge_original.txt#L1-8
c1 = 12
if (i) {
  a1 = x + y
  b1 = a1 + x
} else {
  a2 = b + 2
  c2 = y + 1
}
a4 = c? + a? // Ambiguous `a` and `c`
```
**With Φ functions:**
```phi_merge_ssa.txt#L1-10
c1 = 12
if (i) {
  a1 = x + y
  b1 = a1 + x
} else {
  a2 = b + 2
  c2 = y + 1
}
a3 = Φ(a1, a2)
c3 = Φ(c1, c2)
b2 = Φ(b1, b) // If 'b' is live from outside the 'if'
a4 = c3 + a3
```

### The Φ function

*   Φ merges multiple definitions along multiple control paths into a single definition.
*   At a basic block with `p` predecessors, there are `p` arguments to the Φ function:
    `x_new = Φ(x1, x2, x3, ..., xp)`
*   **How do we choose which `xi` to use?**: We don't really care! The Φ function logically represents the merge.
*   **How do we emit code for this?**: Φ functions are typically "implemented" by inserting `move` instructions on the edges leading into the basic block containing the Φ function.

### "Implementing" Φ

The Φ function is never really implemented this way in machine code, but it could be conceptualized as moves:
**SSA with Φ:**
```phi_impl_ssa.txt#L1-7
a3 = Φ(a1,a2)
c3 = Φ(c1,c2)
a4 = c3 + a3
```
**Conceptual Desugaring (Not actual IR transformation):**
```phi_impl_desugared.txt#L1-10
if (i) {
  a1 = x + y
  b1 = a1 + x
  a3 = a1 // Move a1 to a3
  c3 = c1 // Move c1 to c3
} else {
  a2 = b + 2
  c2 = y + 1
  a3 = a2 // Move a2 to a3
  c3 = c2 // Move c2 to c3
}
a4 = c3 + a3
```

### Trivial SSA

*   Each assignment generates a fresh variable.
*   At each join point, insert Φ functions for all live variables.

Example:
**Original:**
```trivial_ssa_original.txt#L1-4
x = 1
y = x
y = 2
z = y + x
```
**Trivial SSA:**
```trivial_ssa_form.txt#L1-6
x1 = 1
y1 = x1
y2 = 2
x2 = Φ(x1, x1) // If x is live and has multiple definitions reaching a join (even if same source)
y3 = Φ(y1, y2)
z1 = y3 + x2
```
In general, too many Φ functions are inserted in trivial SSA.

### Minimal SSA

*   Each assignment generates a fresh variable.
*   At each join point, insert Φ functions for all live variables *with multiple outstanding definitions* (i.e., definitions from different paths that reach the join).

Example:
**Original:**
```minimal_ssa_original.txt#L1-4
x = 1
y = x
y = 2
z = y + x
```
**Minimal SSA:**
```minimal_ssa_form.txt#L1-5
x1 = 1
y1 = x1
y2 = 2
y3 = Φ(y1, y2) // Only y needs a Phi, as x1 is the only def for x
z1 = y3 + x1
```

---

## Today's Class Recap

*   I. Basic Blocks & Flow Graphs
*   II. Abstraction 1: DAG
*   III. Abstraction 2: Value Numbering
*   IV. Intro to SSA

## Upcoming Class (Wednesday)

*   LLVM Compiler: Further Details
    *   *Suggestion: Play around a bit with LLVM before class.*
