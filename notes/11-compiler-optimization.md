# Compiler Optimization

This note consolidates several lectures on compiler optimizations, covering local optimizations, Static Single Assignment (SSA), data flow analysis, global common subexpression elimination, constant propagation, induction variable optimizations, loop-invariant code motion, and lazy code motion.

## Lecture 3: Local Optimizations, Intro to SSA

### I. Basic Blocks & Flow Graphs

A **Basic block** is a sequence of 3-address statements where:
- Only the first statement can be reached from outside the block (no branches into the middle of the block).
- All statements are executed consecutively if the first one is (no branches out or halts except perhaps at the end of the block).
- Basic blocks are maximal, meaning they cannot be made larger without violating these conditions.

A **Flow graph** has:
- Nodes: basic blocks.
- Edges: `Bi -> Bj`, if `Bj` can follow `Bi` immediately in some execution. This happens either if the first instruction of `Bj` is the target of a `goto` at the end of `Bi`, or `Bj` physically follows `Bi`, which does not end in an unconditional `goto`.

#### Partitioning into Basic Blocks

To partition code into basic blocks, identify the leader of each basic block:
- The first instruction.
- Any target of a jump.
- Any instruction immediately following a jump.

A basic block starts at a leader and ends at the instruction immediately before the next leader (or the last instruction in the function).

Consider the following example:
```/dev/null/example.txt
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

### II. Local Optimizations (within basic block)

Local optimizations occur within a basic block. Key techniques include:
- **Common subexpression elimination** for array expressions, field access in records, and access to parameters.

#### Graph Abstractions

**Example 1: Expression DAG**
For an expression like `a+a*(b-c)+(b-c)*d`, a Directed Acyclic Graph (DAG) can represent the computations, allowing common subexpressions like `b-c` to be computed once.

**Optimized code:**
```/dev/null/example.txt
t1 = b - c
t2 = a * t1
t3 = a + t2
t4 = t1 * d
t5 = t3 + t4
```

**How well do DAGs hold up across statements?**
**Example 2:**
```/dev/null/example.txt
a = b+c;
b = a-d;
c = b+c;
d = a-d;
```
If we optimize this to:
```/dev/null/example.txt
a = b+c;
d = a-d;
c = d+c;
```
The correctness depends on whether `b` is live on exit from the block.

#### Critique of DAGs
- **Cause of problems:** Assignment statements where the value of a variable depends on TIME.
- **How to fix:** Build the graph in order of execution and attach the variable name to the latest value.
- **Result:** The final graph is less abstract. The key is the variable-to-value mapping across time.

### III. Value Numbering: Another Abstraction

Introduced by John Cocke & Jack Schwartz (1970), Value Numbering is more explicit with respect to VALUES and TIME.
- Each value has its own "number," so common subexpressions have the same value number.
- `var2value` is a current map of variables to values, used to determine the value number of the current expression (e.g., `r1 + r2 => var2value(r1) + var2value(r2)`).

#### Value Numbering: Expression Example
For `a+a*(b-c)+(b-c)*d`:
**Optimized code:**
```/dev/null/example.txt
t4 = b - c
t5 = a * t4
t6 = a + t5
t8 = t4 * d
t9 = t6 + t8
```

#### Value Numbering Algorithm
**Data structure:**
`VALUES = Table of { expression /* [OP, valnum1, valnum2] */, var /* name of variable currently holding expr */ }`

For each instruction `(dst = src1 OP src2)` in execution order:
1. `valnum1 = var2value(src1); valnum2 = var2value(src2)`
2. `IF [OP, valnum1, valnum2] is in VALUES`:
    `v = the index of expression`
    `Replace instruction with: dst = VALUES[v].var`
3. `ELSE`:
    `Add expression = [OP, valnum1, valnum2] and var = tv to VALUES`
    `v = index of new entry; tv is new temporary for v`
    `Replace instruction with: tv = VALUES[valnum1].var OP VALUES[valnum2].var; dst = tv`
4. `set_var2value (dst, v)`

#### More Details
- Initial values of variables are those at the beginning of the basic block.
- `VALUES` and `var2value` can be implemented using hash tables.

#### Value Numbering: Basic Block Example
```/dev/null/example.txt
a = b+c
b = a-d
c = b+c
d = a-d
```
Optimized code (with temporaries `t4`, `t5`, `t6`):
```/dev/null/example.txt
t4 = b + c
a = t4
t5 = t4 - d
b = t5
t6 = t5 + c
c = t6
d = t5
```
Assigning to a temporary and then copying to the destination, while increasing instructions, prevents losing common subexpression elimination opportunities if the destination is overwritten later.

#### Question
How do you extend value numbering to constant folding?
**Answer:** Add a field to the `VALUES` table indicating when an expression is a constant and its value.

#### DAGs vs. Value Numbering
- **DAGs:** Focus on expression structure.
- **Value Numbering:** Distinguishes between variables and values, tracks dynamic state information, and interprets instructions in execution order.

### IV. Intro to SSA

**Global Optimizations** look beyond a single basic block:
- **Global common subexpression elimination**
- **Global constant propagation**
- **Dead code elimination**
- **Loop optimizations:** Reduce code in each iteration, code motion, induction variable elimination.
- **Code hoisting:** Eliminates copies of identical code on parallel paths to reduce code size.

#### Recurring Optimization Theme: Where Is a Variable Defined or Used?
- **Loop-Invariant Code Motion:** Requires knowing if variables are defined only outside the loop or if their definitions inside the loop are consistent.
- **Copy Propagation:** For a use of `X`, it's useful to know if all reaching definitions of `X` are copies from the same variable `Y`, and if `Y` is not redefined since that copy.

#### Appearances of Same Variable Name May Be Unrelated
The values in reused storage locations may be provably independent, allowing the compiler to optimize them as separate values (e.g., `X1 = A2 + 1; Y2 = X1 + B; F = 2; X2 = F2 + 7; C2 = X2 + D`). Compiler renaming makes these versions explicit.

#### Definition-Use and Use-Definition Chains
- **Definition-Use (DU) Chains:** For a given definition of `X`, what are all its uses?
- **Use-Definition (UD) Chains:** For a given use of `X`, what are all its reaching definitions?

These can be expensive (O(NM) space and time) in general.
**One solution:** Limit each variable to **ONE** definition site.

#### Static Single Assignment (SSA)
SSA is an Intermediate Representation (IR) where every variable is assigned a value at most once in the program text.
- **For a basic block:** Visit instructions in program order; assign to a fresh version of the LHS variable; use the most recent version of RHS variables.
    Example: `a = x + y; b = a + x; a = b + 2; c = y + 1; a = c + a` becomes
    `a1 = x + y; b1 = a1 + x; a2 = b1 + 2; c1 = y + 1; a3 = c1 + a2`

#### What about Joins in the CFG?
At control flow merges (joins), a `` (phi) function is used to merge multiple definitions along multiple control paths into a single definition.
Example: If `a` is defined in two different branches before a merge, the merged block will have `a_new = (a_from_branch1, a_from_branch2)`.

#### The  function
- Merges multiple definitions from different control paths into a single definition.
- At a basic block with `p` predecessors, there are `p` arguments to the `` function (`xnew = (x1, x2, ..., xp)`).
- **Minimal SSA:** Insert `` functions only for live variables with multiple outstanding definitions at each join point.

#### “Implementing” 
Conceptually, `` functions could be implemented by copying values, but in practice, compilers handle them differently.

#### Trivial SSA
Each assignment generates a fresh variable, and at each join point, `` functions are inserted for **all** live variables. This generates too many `` functions.

#### Minimal SSA
Each assignment generates a fresh variable, and at each join point, `` functions are inserted for all live variables **with multiple outstanding definitions**.

## Lecture 5: Introduction to Data Flow Analysis

### I. Structure of Data Flow Analysis

- **Local analysis (e.g., value numbering):** Analyzes the effect of each instruction and composes these effects within a basic block.
- **Data flow analysis:** Analyzes the effect of each basic block and composes these effects at basic block boundaries. Local techniques are then applied within blocks.

#### What is Data Flow Analysis? (Cont.)
- **Flow-sensitive:** Sensitive to the control flow in a function.
- **Intraprocedural analysis:** Within a single procedure.
- **Examples of optimizations:** Constant propagation, common subexpression elimination, dead code elimination.

For each variable `x`, data flow analysis determines its value, which definition defines it, and if the definition is still meaningful (live).

#### Static Program vs. Dynamic Execution
- A static program can have infinitely many possible execution paths dynamically.
- Data flow analysis abstracts this by combining information from all instances of the same program point.
- **Example:** For `b = a`, which definition of `a` reaches this statement?

#### Effects of a Basic Block
- A statement `a = b + c` uses `b` and `c`, kills an old definition of `a`, and defines a new `a`.
- The effect of a basic block is composed from its statements.
- A **locally exposed use** is a use not preceded by a definition in the same block.
- A definition in a block kills all other definitions of the same data item.
- A **locally available definition** is the last definition of a data item in a block.

### II. Reaching Definitions

- **Every assignment is a definition.**
- A definition `d` **reaches** a point `p` if there's a path from `d` to `p` where `d` isn't killed (overwritten).
- **Problem statement:** For each point in the program, determine if each definition reaches that point. This uses a bit vector per program point, with vector length equal to the number of definitions.

#### Data Flow Analysis Schema
1. Build a flow graph (nodes = basic blocks, edges = control flow).
2. Set up equations between `in[b]` (information entering block b) and `out[b]` (information exiting block b):
    - **Effect of code in basic block:** A transfer function `fb` relates `in[b]` and `out[b]`.
    - **Effect of flow of control:** Relates `out[b]` to `in[b']` if `b` and `b'` are adjacent.
3. Find a solution to the equations (a fixed point).

#### Effects of a Statement
For a statement `s` (e.g., `d: x = y + z`):
`out[s] = fs(in[s]) = Gen[s]  (in[s] - Kill[s])`
- `Gen[s] = {d}` (definitions generated by `s`).
- `Kill[s]` = set of all other definitions to `x` in the rest of the program.

#### Effects of a Basic Block
The transfer function for a basic block `B` is the composition of its statements' transfer functions:
`out[B] = fB(in[B]) = Gen[B]  (in[B] - Kill[B])`
- `Gen[B]`: locally available definitions (defined locally & reaches end of block).
- `Kill[B]`: set of definitions killed by definitions in `B`.

#### Effects of the Edges (acyclic)
For a join node (multiple predecessors), the `in` set is the union of `out` sets from all predecessors:
`in[b] = out[p1]  out[p2]  ...  out[pn]`

#### Cyclic Graphs
Equations still hold for cyclic graphs. The solution is a **fixed point**.

#### Reaching Definitions: Iterative Algorithm
```/dev/null/algorithm.txt
input: control flow graph CFG = (N, E, Entry, Exit)

// Boundary condition
out[Entry] = 

// Initialization for iterative algorithm
For each basic block B other than Entry
  out[B] = 

// iterate
While (changes to any out[] occur) {
  For each basic block B other than Entry {
    in[B] = U(out[p]), for all predecessors p of B
    out[B] = fB(in[B])    // out[B]=gen[B] U(in[B]-kill[B])
  }
}
```

#### Reaching Definitions: Worklist Algorithm
A more efficient iterative algorithm using a worklist for changed nodes.

### III. Live Variable Analysis

- **Definition:** A variable `v` is **live** at point `p` if its value is used along some path in the flow graph starting at `p`. Otherwise, it's **dead**.
- **Motivation:** Useful for register allocation.
- **Problem statement:** For each basic block, determine if each variable is live in that block. This uses a bit vector for each variable.

#### Live Variables: Effects of a Basic Block (Transfer Function)
- **Insight:** Trace uses backwards to the definitions. This is a **backward analysis**.
- `IN[b] = Use[b]  (OUT[b] - Def[b])`
- `Use[b]`: set of locally exposed uses in `b`.
- `Def[b]`: set of variables defined in `b`.

#### Flow Graph
For a join node (multiple successors), the `out` set is the union of `in` sets from all successors:
`out[b] = in[s1]  in[s2]  ...  in[sn]`

#### Live Variables: Iterative Algorithm
```/dev/null/algorithm.txt
input: control flow graph CFG = (N, E, Entry, Exit)

// Boundary condition
in[Exit] = 

// Initialization for iterative algorithm
For each basic block B other than Exit
  in[B] = 

// iterate
While (changes to any in[] occur) {
  For each basic block B other than Exit {
    out[B] = U(in[s]), for all successors s of B
    in[B] = fB(out[B])    // in[B]=Use[B] U(out[B]-Def[B])
  }
}
```

### IV. Framework

| Feature             | Reaching Definitions | Live Variables       |
| :------------------ | :------------------- | :------------------- |
| **Domain**          | Sets of definitions  | Sets of variables    |
| **Direction**       | Forward              | Backward             |
| **Transfer function** | `fb(x) = Genb  (x –Killb)` | `fb(x) = Useb  (x -Defb)` |
| **Meet Operation ()** | ``                  | ``                  |
| **Boundary Condition** | `out[entry] = `     | `in[exit] = `       |
| **Initial interior points** | `out[b] = `         | `in[b] = `          |

Other data flow analysis problems (e.g., Available Expressions) fit into this general framework.

## Lecture 6: Foundations of Data Flow Analysis

### I. Meet Operator

- **Properties:** Commutative (`x  y = y  x`), idempotent (`x  x = x`), associative (`x  (y  z) = (x  y)  z`).
- There is a **Top element T** (`x  T = x`).
- The meet operator defines a **partial ordering** (`x ≤ y` iff `x  y = x`).
- There exist Top `T` and Bottom `⊥` elements for a semi-lattice.

#### Further Properties of Meet Operators
- `x < y` means `x` is more conservative than `y`.
- For all `a, b`, `a  b ≤ a`.
- If `w ≤ x` and `w ≤ y` then `w ≤ x  y`.

#### Descending Chain
- The height of a lattice is the largest number of `>` relations in a descending chain.
- An important property is a **finite descending chain**. Some infinite lattices (like for constant propagation) can still have a finite descending chain.

### II. Transfer Functions

- **Basic Properties f: V → V:**
    - Has an identity function (`f(x) = x`).
    - Closed under composition (if `f1, f2  F`, then `f1f2  F`).

#### Monotonicity
A framework `(F, V, )` is **monotone** if `x ≤ y` implies `f(x) ≤ f(y)`.
Equivalently, `f(x  y) ≤ f(x)  f(y)`.
- If input in `second iteration ≤ input in first iteration`, then `result in second iteration ≤ result in first iteration`.

#### Distributivity
A framework `(F, V, )` is **distributive** if `f(x  y) = f(x)  f(y)`.
- Reaching Definitions is distributive.
- Constant Propagation is **not** distributive.

### III. Data Flow Analysis

- **Perfect data flow answer:** ` fpi(T)`, for all possibly executed paths `pi` in the program.
- In general, determining all possibly executed paths is **undecidable**.

#### Meet-Over-Paths (MOP)
- **MOP(n) =  fpi(T)**, for all paths `pi` in the data flow graph reaching `n`.
- MOP is conservative (`MOP ≤ Perfect-Solution`) as it considers more paths than necessary (including unexecuted paths).
- **Meet = union:** Definition may reach; Variable may be live.
- **Meet = intersection:** Expression is always available.
- Considering too few paths would not be safe. The desirable solution is as close to MOP as possible.

#### Solving Data Flow Equations
- Any solution satisfying equations is a **Fixed Point Solution (FP)**.
- Iterative algorithms compute the **Maximum Fixed Point (MFP)**, which is the largest solution.
- `FP ≤ MFP ≤ MOP ≤ Perfect-solution`. `FP` and `MFP` are safe.
- If monotone and converges, `IN[b] ≤ MOP[b]`.

#### Partial Correctness of Algorithm
If the data flow framework is monotone and the algorithm converges, then `IN[b] ≤ MOP[b]`.

#### Precision
If the data flow framework is distributive and the algorithm converges, then `IN[b] = MOP[b]` (high precision).
- Monotone but not distributive frameworks behave as if there are additional paths.

#### Additional Property to Guarantee Convergence
A monotone data flow framework converges if there is a **finite descending chain**.
- Guaranteed to converge after at most `(height of lattice) x (number of nodes in flow graph)` iterations.

### IV. Speed of Convergence

- Speed depends on the order of node visits.
- **Reverse Postorder** is a common visit order.
- Number of iterations = `number of back edges in any acyclic path + 2`.

## Lecture 7: Global Common Subexpression Elimination; Constant Propagation/Folding

### I. Available Expressions Analysis

- **Definition:** An expression `E` is **available** at point `P` if along every path to `P`:
    - `E` must be evaluated at least once.
    - No variables in `E` redefined after the last evaluation.
- `E` may have different values on different paths.

#### Formulating the Problem
- **Domain:** A bit vector, with a bit for each textually unique expression.
- **Direction:** Forward.
- **Lattice Elements:** All bit vectors of given length.
- **Meet Operator:** Element-wise minimum (intersection).
- **Top:** All ones `(1,1,...,1)`.
- **Bottom:** All zeros `(0,0,...,0)`.
- **Boundary condition:** `out[entry] = (0,...,0)`.
- **Initialization for interior nodes:** Initialize `out[b] = T` (all expressions) for all interior basic blocks `b`.

#### Transfer Functions
- `out[b] = gen[b]  (in[b] - kill[b])`
- An instruction kills `E` if it defines a variable in `E`.
- An instruction generates `E` if it evaluates `E` and doesn't kill it.

### II. Eliminating CSEs

- **Value Numbering:** Eliminates local common subexpressions.
- **Available Expressions:** Provides the set of expressions available at the start of a block.
- If a CSE is an "available expression," transform the code. This often involves copying the expression to a new temporary variable at each evaluation reaching the redundant use.

#### Limitation: Textually Identical Expressions
- `x + y` and `y + x` are textually different but semantically identical.
- **Solution:** Sort the operands for canonical representation.

#### Further Improvements
- Handle expressions with more than two operands.
- Address textually different but equivalent expressions (e.g., after copy propagation).
- **Solution:** Use multiple passes of GCSE combined with copy propagation.

### III. Constant Propagation/Folding

- At every basic block boundary, determine if each variable `v` is a constant and, if so, its value.

#### Semi-lattice Diagram
- Domain includes `UNDEF`, integer constants, and `NAC` (Not-A-Constant).
- Not a finite domain (unless bounds number of bits), but has finite height (2).
- One such lattice for each variable.

#### Meet Operation in Table Form
- `UNDEF  c2 = c2`
- `c1  c2 = c1` if `c1 = c2`, else `NAC`.
- `NAC  any = NAC`.

#### Transfer Function
- For non-assignment instructions: `OUT[b,x] = IN[b,x]`.
- For `x3 = x1 + x2`: `OUT[b,x3]` is computed based on `IN[b,x1]` and `IN[b,x2]` and the operator. Other variables are propagated unchanged.

#### Not Distributive
Constant Propagation is **not distributive**. This means the iterative solution is not always perfectly precise but is still conservative (safe).

#### Summary of Constant Propagation
- A useful optimization.
- Illustrates abstract execution, an infinite semi-lattice, a non-distributive problem, and a problem where cycles can add information.

## Lecture 8: Induction Variable Optimizations

### I. Finding Loops

- **Goals:** Define a loop in graph-theoretic terms, independent of programming language constructs, with a uniform treatment for all loop types.
- **Intuitive properties:** Single entry point, edges must form at least a cycle. Loops can nest.

#### Important Concept: Dominance
- `x dominates w (x dom w)` if every path from the start node to `w` goes through `x`.
- `x strictly dominates w (x sdom w)` if `x dom w` and `x ≠ w`.
- A **Dominance Tree (D-Tree)** represents these relationships.

#### Natural Loops
- **Single entry-point:** A **header** that dominates all nodes in the loop.
- A **back edge** `t -> h` is an arc whose head `h` dominates its tail `t`.
- The **natural loop** of a back edge `t -> h` is the smallest set of nodes including `t` and `h` with no predecessors outside the set (except predecessors of `h`).

#### Algorithm to Find Natural Loops
1. Find the dominator relations in a flow graph.
2. Identify the back edges.
3. Find the natural loop associated with each back edge.

#### Step 1. Finding Dominators
- Formulated as a data flow analysis problem: `d dom n` iff `d` lies on all paths to `n`.
- **Direction:** Forward.
- **Values:** Sets of basic blocks.
- **Meet operator:** Intersection.
- **Transfer function:** `fb(x) = {b}  x`.
- **Monotone & Distributive:** Yes.
- Converges quickly, often in 1 pass with rPostorder.

#### Step 2. Finding Back Edges
- Construct a **depth-first spanning tree** of the CFG.
- Categorize edges: Advancing (A), Cross (C), Retreating (R).
- A **back edge** `t -> h` is a retreating edge where `h` dominates `t`.
- For reducible flow graphs, retreating edges are back edges.

#### Step 3. Constructing Natural Loops
- For each back edge `t -> h`:
    - Delete `h` from the flow graph.
    - Find nodes that can reach `t`. These nodes, plus `h`, form the natural loop.

#### Inner Loops
- Disjoint loops or nested loops. An **inner loop** contains no other loops.
- Loops sharing the same header are often combined and treated as one.

#### Preheader
A **preheader basic block** is created for every loop to execute code once before the loop, facilitating optimizations.

### II. Overview of Induction Variable Elimination

- **Example:** `for(i=0; i<100; i++) A[i] = 0;`
    - `i` is an induction variable.
    - `t1 = 4 * i`, `t2 = &A + t1` are also induction variables.
- The goal is to transform the loop to eliminate costly operations (`4 * i`) by maintaining auxiliary variables that are updated incrementally.

#### Definitions
- A **basic induction variable** `X` is one whose only definitions within the loop are `X = X+c` or `X = X-c`, where `c` is a constant or loop-invariant.
- An **induction variable** is a basic induction variable `B`, or a variable `A` defined once in the loop whose value is a linear function of some basic induction variable (`A = c1 * B + c2`).
- The **family** of a basic induction variable `B` is the set of induction variables `A` such that `A` is a linear function of `B` each time `A` is assigned.

#### Optimizations
1.  **Strength reduction:** For an induction variable `A = c1 * B + c2`, create a new variable `A'`, initialize it in the preheader, and update it incrementally (`A' = A' + x*c1`) alongside `B`. Replace original assignments to `A` with `A = A'`.
2.  **Optimizing non-basic induction variables:** Apply copy propagation and dead code elimination.
3.  **Optimizing basic induction variables:** Eliminate basic induction variables used only for calculating other induction variables and loop tests. Replace loop tests (`if B > X goto L1` with `if (A' > c1 * X + c2) goto L1`). Recompute `B` from `A'` if `B` is live after the loop.

### III. Further Details

- Basic induction variables can be detected by scanning the loop.
- **Strength Reduction Algorithm:** For each induction variable `A = c1 * B + c2`, a shadow variable `A'` holds `c1 * B + c2`. `A`'s definition is replaced with `A = A'` when executed.

#### Finding Induction Variable Families
- Conditions (C1, C2) determine if a variable `A` is in the family of a basic induction variable `B`, based on single assignments in the loop and dependencies.

## Lecture 9: Loop Invariant Computation and Code Motion

### I. Loop-Invariant Computation and Code Motion

- A **loop-invariant computation** is one whose value doesn't change as long as control stays within the loop.
- **Code motion** moves a loop-invariant statement to the preheader of the loop.

#### Algorithm
- **Observations:**
    - Loop invariants have operands defined outside the loop or are themselves invariant.
    - Not all loop-invariant instructions can be moved.
- **Algorithm steps:**
    1. Find invariant expressions.
    2. Define conditions for code motion.
    3. Perform code transformation.

#### Algorithm: Detecting Loop Invariant Computation
1. Compute reaching definitions.
2. Mark a statement `A = B + C` as `INVARIANT` if all definitions of `B` and `C` reaching it are outside the loop (or if `B, C` are constants).
3. Repeat marking `INVARIANT` if:
    - All reaching definitions of an operand are outside the loop, OR
    - There is exactly one reaching definition for an operand, and it's from a loop-invariant statement inside the loop.
    - Continue until no changes.

### II. Conditions for Code Motion

- **Correctness:** Movement must not change program semantics.
    - The moved code dominates all loop exits.
    - No other definition of the assigned variable exists in the loop.
    - The moved code dominates all blocks in the loop that use the assigned variable (or no other definitions reach the use).
- **Performance:** Code is not slowed down.

#### Code Motion Algorithm
1. Compute reaching definitions, loop invariant computations, and dominators.
2. Find loop exits.
3. Identify candidate statements: loop-invariant, in blocks dominating all exits, assign to variables not assigned elsewhere in the loop, and dominate all uses.
4. Perform a depth-first search of blocks. Move candidates to the preheader if their dependencies have also been moved.

#### More Aggressive Optimizations
- Relaxing the "dominating all exits" constraint if the destination is not live after the loop.
- **Landing pads:** Ensure preheader executes only if the loop is entered.

### III. Partial Redundancy Elimination

- **Sources of Redundancy:** Global common subexpressions, loop-invariant expressions, partially redundant expressions.

#### Recall: Global Common Subexpression Elimination
An expression `b + c` is fully redundant at point `p` if, on every path reaching `p`, `b + c` has been computed and `b, c` haven't been overwritten.

#### Loop Invariant Code Motion
A loop-invariant expression `b + c` inside a loop can be moved to the preheader if its value doesn't change inside the loop and the code is executed at least once.

#### Partial Redundancy
- An occurrence of `E` at `P` is **partially redundant** if `E` is partially available (evaluated on at least one path to `P` with no operands redefined).
- Partially redundant expressions can be eliminated by inserting computations to make them fully redundant.

#### Loop Invariants are Partial Redundancies
A loop-invariant expression is a special case of a partially redundant expression.

#### Partial Redundancy Elimination (PRE)
- A powerful optimization that subsumes global common subexpression elimination (full redundancy) and loop-invariant code motion (partial redundancy for loops).
- Aims to place calculations such that no path re-executes the same expression.

#### Where Can We Insert Computations?
- **Safety:** Never introduce a new expression along any path (e.g., avoid exceptions). Only insert where the expression is **anticipated** (its value will be used along ALL subsequent paths).
- **Performance:** Never increase the number of computations on any path.

## Lecture 10: Lazy Code Motion

### I. Full Redundancy: A Cut Set in a Graph

- **Full redundancy** at point `p`: Expression `a + b` is redundant on all paths to `p`.
- A **cut set** separates `entry` from `p`. Each node in a cut set contains a calculation of `a + b`, and `a, b` are not redefined.

#### Partial Redundancy: Completing a Cut Set
- **Partial redundancy** at `p`: Redundant on some but not all paths.
- Add operations to create a cut set. Moving operations up can eliminate redundancy.
- **Constraint:** No wasted operation. `a + b` is **anticipated** at `B` if its value computed at `B` will be used along ALL subsequent paths.

### II. Lazy Code Motion Algorithm

**Goal:** Place computations as late as possible without introducing redundancy. This minimizes register lifetimes while maximizing redundancy elimination.

#### Preparing the Flow Graph
- **Critical edges:** Source basic block has multiple successors, and destination has multiple predecessors.
- Modify the flow graph by adding a basic block for every edge leading to a block with multiple predecessors (not just critical edges). This simplifies the algorithm.

#### Algorithm Steps (4 Passes)
1.  **Pass 1: Anticipated Expressions (Backward Pass)**
    - `Anticipated[b].in`: Set of expressions anticipated at the entry of `b`.
    - An expression is anticipated if its value will be used along ALL subsequent paths.
    - **Domain:** Sets of expressions. **Meet:** Intersection. **Transfer:** `fb(x) = EUseb  (x - EKillb)`.
    - Places operations at the frontier of anticipation (boundary between not anticipated and anticipated).

2.  **Pass 2: (Will be) Available Expressions (Forward Pass)**
    - Determines if an expression `e` will be available at `p` if it has been "anticipated but not subsequently killed" on all paths to `p`.
    - Helps to refine earliest placement.
    - **Domain:** Sets of expressions. **Meet:** Intersection. **Transfer:** `fb(x) = (Anticipated[b].in  x) - EKillb`.
    - `earliest(b) = anticipated[b].in - available[b].in`. Place expressions at this earliest point.

3.  **Pass 3: Postponable Expressions (Forward Pass)**
    - Delays creating redundancy to reduce register pressure.
    - An expression `e` is postponable at point `p` if all paths to `p` have seen its earliest placement but not a subsequent use.
    - **Domain:** Sets of expressions. **Meet:** Intersection. **Transfer:** `fb(x) = (earliest[b]  x) - EUseb`.
    - `latest(b)` is calculated based on `earliest[b]`, `postponable.in[b]`, and uses in `b` or its successors.

4.  **Pass 4: Used Expressions (Backward Pass)**
    - Like liveness analysis for expressions.
    - `Used.out[b]`: Sets of used (live) expressions at the exit of `b`.
    - Eliminates temporary variable assignments unused beyond the current block.
    - **Domain:** Sets of expressions. **Meet:** Union. **Transfer:** `fb(x) = (EUse[b]  x) - latest[b]`.

#### Code Transformation
- For all basic blocks `b`, if `(x+y)  (latest[b]  used.out[b])`, then at the beginning of `b`, add a new temporary `t = x+y`, and replace every original `x+y` by `t`.

#### Remarks
- Lazy Code Motion is a powerful algorithm that finds many forms of redundancy in a unified framework and illustrates the power of data flow analysis through multiple interconnected problems.

## Lecture 11: Static Single Assignment (SSA)

### I. Review: Static Single Assignment (SSA)

- **Static Single Assignment (SSA)** is an IR where every variable is assigned a value at most once in the program text.
- **Within a basic block:** Easy, assign to a fresh version on LHS, use most recent on RHS.
- **At joins in the CFG:** `` (phi) functions merge multiple definitions into a single definition.
    - `xnew = (x1, x2, x3, … , xp)`
- **Minimal SSA:** Insert `` functions only for live variables with multiple outstanding definitions at join points.

### II. When/Where to Insert 

- Insert a `` function for variable `v` in block `Z` if:
    - `v` was defined more than once before (e.g., in blocks `X` and `Y`, where `X ≠ Y`).
    - There exist non-empty paths `Pxz` from `X` to `Z` and `Pyz` from `Y` to `Z` such that `Z` is the first node common to both paths.
- The **Dominance Frontier** helps determine where `` functions are needed.
- The **Dominance Frontier of a node x (DF(x))** is the set of nodes `w` such that `x` dominates a predecessor of `w`, but `x` does not strictly dominate `w`.

#### Using Dominance Frontier to Compute SSA: Overview
1. Place all `()` functions.
2. Rename all variables.

#### Using Dominance Frontier to Place ()
- Algorithm iteratively places `` functions:
    - For each variable `v`, initialize a worklist `W` with all basic blocks that define `v`.
    - While `W` is not empty, remove a block `n` from `W`.
    - For each block `y` in `DF[n]`:
        - If `y` does not already have a `` for `v`, insert one at the top of `y`.
        - Add `y` to `PHI[v]` (set of blocks with a `` for `v`).
        - If `v` was not originally defined in `y`, add `y` to `W` (because the `` counts as a definition).
- This computes the Iterated Dominance Frontier on the fly, inserting the minimal number of `` functions.

### III. Example

Detailed example of computing Dominance Tree, Dominance Frontiers, and then inserting `` functions and renaming variables.

### IV. Constant Propagation with SSA

- If `v  c` or `v  (c,c,c)` (all inputs are the same constant), replace all uses of `v` with `c`.
- An iterative algorithm can achieve this:
    - Maintain a worklist of all definitions.
    - If a definition `v  c` or `v  (c,…,c)` is processed:
        - Delete the statement.
        - Replace `v` with `c` in all its uses.
        - Add statements that use `v` to the worklist.

#### Conditional Constant Propagation
- A more advanced technique that tracks which blocks are executed and which variables are constants more precisely.
- **Assumptions:** Blocks are unexecuted until proven otherwise; variables are constants until proven otherwise.
- **Lattice for variables:** `⊥` (not executed) -> integer constants -> `T` (not a constant).
- This can detect situations where branches are never taken based on constant values, leading to further optimizations.