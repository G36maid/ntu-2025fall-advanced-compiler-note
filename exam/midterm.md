# midtern


## Local optimization & Lazy Code Motion


The following Algorithm 1 is modified from Extending the Value Numbering Algorithm presented in class. 
These modifications ensure that the names of a copy assignment share the same value number, 
specify the name to restore, and define how to generate a new value number. With these adjustments, 
the Algorithm 1 is prepared to address the following two questions.


Algorithm 1

```
for i <- to n , where the block has n instructions

    if i-th instructions is a copy assignment in the form T_i <- L_i
        retereve the value number V_1 for L_1
        construct a hash key k as V1
    else
        get instruction i in the form T_i <- L_i O_pi R_i
        retereve value number V_1 and V2 for L_1 and R_1

        if the operation O_pi commutes and V1 > V2
            swap V1 and V2
        construct a hash key k as {V1,Op1,V2}

    if Ti does not already have a table entry
        create a table entry for Ti

    if the key k already presented in the table
        replace instructions i with "Ti <- x" where is the constent value
            or the first available name stored under key K
    else
        create a new table entry for K
        genetate a new value number and name Ti in k's table entry
        store the new value number and name Ti in k's table entry
        if both V1 and V2 are marked constants
            evalute Li Opi Ri into c
            replace operation i with "Ti <- c"
            store c in the table entry for k  and Ti

    store k's value number  in the table entry for k
```

### Q1

Consider applying Algorithm 1 to the following pseudo code, and the value numbers of (x, y, z) are (0, 1, 2).

Question:

What are the value numbers of (a, b, c, d, e) at the end?

```
x <- 0
y <- 1
z <- 2
a <- x + y
b <- x
c <- x + b
d <- b + x
e <- x + x
```

ANSWER: a=3, b=0, c=4, d=4, e=4

### q2
Consider applying Lazy Code Motion to the following pseudo code.

Question:
How many element(s) are in DEEXPR(B1), EXPRKILL(B1), and UEEXPR(B2), respectively?

Choose the correct pair (D1, K1, U2), where:

1. D1 is the number of element(s) in DEEXPR(B1),

2. K1 is the number of element(s) in EXPRKILL(B1).

3. U2 is the number of element(s) in UEEXPR(B2).

Note:

"br" is a condition branch operation. "br condition l_t l_f" jumps to the basic block with the first label "l_t" if condition is true,
else jumps to basic block with the second label "l_f".

```
.B1:
    r10 <- 1
    r4 <- r10
    r11 <- le r4 r3
    br r11 .B2 .B3
.B2:
    r12 <- add r4 r10
    r4 <- r12
    r13 <- add r1 r2
    r5 <- r13
    r14 <- le r4 r3
    br r14 .B2 .B3
.B3:
```

ANSWER: (2,1,1)

### q3
Question: (Question text missing - cannot provide answer)
Which expression(s) would be rewritten (That is, delete and insert)?

Only r12
Only r13
r12 and r13
none

ANSWER: Only r13

## Different compiler passes, phase ordering, loop transformations

### q1
1. There are 2 categories of compiler optimizations: Code-optimizing passes vs. Scope-enhancing passes. Is inlining a scope-enhancing pass or a code-optimizing pass? Choose one below.
*
Inlining is a scope-enhancing pass.
Inlining is a code-optimizing pass.

ANSWER: Inlining is a scope-enhancing pass.
### q2
2. Choose the WRONG answer below. *
Phase ordering among all the adjacent uni-modular transforms is a non-issue.
The LLVM pass "dead code elimination" should happen at the end of compilation, to optimize away the dead code in the most effective way.
Uni-modular transforms cannot handle non-perfectly nested loops.
Dependence analysis is not polynomial in its complexity.

ANSWER: The LLVM pass "dead code elimination" should happen at the end of compilation, to optimize away the dead code in the most effective way.
### q3
3. Choose the WRONG answer below. *
LLVM's Intermediate Representation uses SSA form.
Dependence analysis focuses on the location instead of value.
Data flow analysis is a framework.
Blocking is a uni-modular transform.

ANSWER: Blocking is a uni-modular transform.
### q4
4. For the following loop:
for (i = 2; i < 100; i++)
    A[i] = A[i-2];

What is the largest number of processors that can be used effectively to execute this loop? Write down the number ONLY, in Arabic numerals. E.g., 20
Note: Just put down the number without any punctuation. I.e., no "20". No period at the end. You won't get any point if you violated the format above.

ANSWER: 2

### q5
For the following loop:
for (i = 2; i < 100; i++)
    A[i] = A[i-2];

Rewrite the code with processor p as a parameter.

ANSWER:
```
for (i = p + 2; i < 100; i = i + 2)
    A[i] = A[i-2];
```

## Loop Unrolling & Global Code Placement
### q1
Consider this Control Flow Graph (CFG) with 13 blocks, please answer the following 3 questions

```
1 -> 2,5,9
2 -> 3
3 -> 3,4
4 -> 13
5 -> 6,7
6 -> 4,8
7 -> 8,12
8 -> 5,13
9 -> 10,11
10 -> 12
11 -> 12
12 -> 13
13 -> none
```

Recall the definition and notation of dominance frontier:

Def. A block w ∈ DF(x) if w has a CFG predecessor t`hat x dominates and x does not strictly dominate w.

DF(x) = {w | x dominates pred(w) AND !(x strictly dominates w)}

Question:

Which of the following is the dominance frontier of block 5?
*
{4, 12, 13}
{4, 5, 12, 13}
{4, 8, 12, 13}
{4, 5, 8, 12}

ANSWER: {4, 5, 12, 13}

### q2
Which of the following statement regarding loop unrolling in regional optimization is INCORRECT? *
Unrolling can move consecutive memory accesses into the same iteration, which may improve memory locality.
If the increased demand for registers due to unrolled loop induces additional register spills and restores, then the resulting memory traffic may overwhelm the potential benefits of unrolling.
With more independent operations in the unrolled loop, the scheduler has a better chance of keeping multiple functional units busy.
Unrolling a loop can increase the code size in both IR and executable, which shortens the compile time.

ANSWER: Unrolling a loop can increase the code size in both IR and executable, which shortens the compile time.

### q3
Consider this Control Flow Graph(CFG) with executed frequency on each edge, please answer the following 1 question
In global code placement,  for the purpose of determining the code layout, the compiler finds hot paths through the CFG—paths that contain the most frequently executed edges.
We adopt the following algorithm to determine the hot paths in the CFG.

Note that <Bi, Bj, ..., Bk>n means a chain with priority n contains Bi, Bj, ..., and Bk.

```
10 -> B0
B0

```

Question:

What is the eventual set of chains of the CFG?
*

{ <B0, B1, B2, B3>0, <B5, B4>4 }
{ <B0, B1, B2, B3>0, <B5, B4>3 }
{ <B0, B1>0, < B2, B3>1, <B5, B4>4 }
{ <B0, B1>0, < B2, B3>1, <B5, B4>3 }


## Data Flow Analysis & Lattice Theory


### q1
Which of the following statements about liveness analysis in compiler optimization is TRUE? *
Liveness analysis is guaranteed to always produce precise results for programs with arbitrary pointer usage, as it can accurately track all possible aliases and their lifetimes throughout the program execution.
The complexity of the iterative fixed-point algorithm for liveness analysis is always O(n*|V|), where n is the number of nodes in the control flow graph and |V| is the number of variables in the program.
Liveness analysis always produces identical results whether performed before or after dead code elimination, as the removal of dead code does not affect the liveness of remaining variables in any way.
If we use the iterative fixed-point algorithm for liveness analysis, we can choose any order in which blocks are evaluated, and it will always produce the same and correct LIVEOUT sets.

ANSWER: If we use the iterative fixed-point algorithm for liveness analysis, we can choose any order in which blocks are evaluated, and it will always produce the same and correct LIVEOUT sets.
### q2
Consider the following Control Flow Graph (CFG). We want to calculate which blocks dominate each block n using two different traversal orders:
Forward order (processing blocks sequentially from B0 to B8)Reverse post-order

Using the iterative fixed-point algorithm, how many iterations are needed to compute the dominators for each traversal order?

Find the correct pair (a, b) where:
a = number of iterations using forward order
b = number of iterations using reverse post-order

Important notes:
1. An iteration counts when all nodes in the CFG have been processed once in the given order
2. The initial setup counts as iteration 0.
3. Even after reaching a fixed point, the algorithm needs one additional iteration to confirm no further changes occur. Include this confirmation iteration in your count.
*
```


```
(4, 2)
(4, 3)
(3, 2)
(3, 3)

ANSWER: (CFG not provided - likely (4, 2) or (3, 2) based on typical dominator analysis behavior)

### q3
In compiler theory, Reaching Definitions is a fundamental data-flow analysis used to determine which definitions may reach a given point in the code. Before we dive into the main question, let's review some key concepts:

Key concepts:

1. Reachability: A definition d of variable x reaches operation u if and only if u uses the value of x and there exists a path from d to u along which x is not redefined.
2. REACHES(n): For each node n in the Control Flow Graph (CFG), REACHES(n) is the set that contains the name of every definition that reaches the entry point of the basic block corresponding to n.
3. gen(B): The set of definitions generated in block B. These are new definitions or redefinitions of variables within the current block B.
4. kill(B): The set of definitions from other blocks that are invalidated by new definitions in the current block B. In other words, these are definitions that no longer reach the end of block B due to being overwritten within B.

Question:

Which of the following correctly describes the Reaching Definitions analysis within the data-flow analysis framework, considering the computation of REACHES(n) for each CFG node n? Choose the correct combination of (Direction, Values, Transfer function, Meet operator):

1. Direction: Indicates whether the analysis flows forward or backward through the CFG.

2. Values: Describes the type of information being propagated.

3. Transfer function f(S): Defines how the information changes as it flows through a basic block B, where S is the input state.

4. Meet operator: Specifies how information is combined when multiple paths in the CFG converge.
*
(Backward, Sets of definitions,  f(S) = S ∪ gen(B), Intersection)
(Forward, Sets of definitions, f(S) = (S - gen(B)) ∪ kill(B), Union)
(Forward, Sets of definitions, f(S) = (S - kill(B)) ∪ gen(B), Union)
(Forward, Sets of definitions, f(S) = (S - kill(B)) ∪ gen(B), Intersection)

ANSWER: (Forward, Sets of definitions, f(S) = (S - kill(B)) ∪ gen(B), Union)

### q4
Consider a complete lattice L with finite height and a monotone function f : L → L. Let x0 = ⊥ (bottom element of L) and define the sequence xi+1 = f(xi) for i ≥ 0.
Question:
Which of the following statements is TRUE about this sequence and the least fixed point (lfp) of f?
*
The sequence may oscillate indefinitely between two values without reaching a fixed point if L has infinite width.
The least upper bound of any two consecutive elements in the sequence (xi, xi+1) is always equal to xi+1.
The sequence always reaches the lfp of f in exactly height(L) - 1 iterations.
The lfp of f is always equal to lub({xi | i ≥ 0}), where lub is the least upper bound operation on L.

ANSWER: The lfp of f is always equal to lub({xi | i ≥ 0}), where lub is the least upper bound operation on L.

## SSA/Compiler optimization


### Question 1: Dominance Frontier Computation
The dominance frontier DF(B1) is:
```
B0(x=..) -> B1,B2
B1(x=..) -> B3
B2(y=..) -> B3
B3(z=x+y) -> B4(exit)
```
A) {B3, B4}
B) {B2, B3}
C) {B4}
D) {B3}
E) {B1, B3}
F) ∅ (empty set)
G) {B0, B3}
H) {B2, B3, B4}

ANSWER: D) {B3}

### Question 2: Iterated Dominance Frontier and φ-Placement

How many φ-functions for variable x are needed in minimal SSA form?
```
B0 -> B1,B2
B1(x=1) -> B3
B2(x=2) -> B3
B3(y=x) -> B4,B5
B4(x=3) -> B6
B5(if(..)) -> B3
B6(w=x) -> exit
```
A) 2 φ-functions at B3 and B6
B) 3 φ-functions at B3, B6, and B5
C) 1 φ-function at B3
D) 2 φ-functions at B3 (placed twice due to iteration)
E) 3 φ-functions at B3, B5, and B6
F) 4 φ-functions at B3, B5, B6, and one more at B3 (iteration effect)
G) 1 φ-function at B6 only
H) 0 φ-functions (reaching definitions analysis suffices)

ANSWER: C) 1 φ-function at B3

### Question 3: Post-Dominator Tree and Control Dependence

Which node is B2 control-dependent on?
```
B0 -> B1
B1(if(p)) -> B2, B3
B2 -> B4
B3 exit
B4 exit
```
A) B0
B) B1
C) B4
D) B2 is not control-dependent on any node
E) B0 and B1 (both)
F) B1 and B2 (B2 depends on itself)
G) All nodes in the CFG
H) B3 (the alternate exit path)

ANSWER: B) B1

### Question 4: Loop Invariant Code Motion with Exceptions

Expression t = a/b is loop-invariant (a, b not modified in loop). Under which condition is hoisting t = a/b before B1 ILLEGAL?
```
B0 -> B1
B1(if(i<n>)) -> B2,exit
B2 (t=a/b,c[i]=t,i++)-> b1

```
A) When b might be zero
B) When the loop might execute zero iterations (n ≤ 0)
C) When division has side effects (sets CPU flags)
D) When a and b are floating-point with NaN handling
E) B only (must not introduce new exceptions)
F) A and B together
G) B, C, and D together
H) Never illegal - hoisting loop-invariant code is always safe

ANSWER: F) A and B together

### Question 5: Dominance Frontier and φ-Placement

Where must φ-functions for variable x be placed?
```
B0(x=1) -> B1
B1(if(p)) -> B2,B3
B2(x=2) -> B4
B3(y=..) -> B4
B4 (z=x)
```
A) B1 only (decision point)
B) B1 and B4 (both join points)
C) B2 and B3 (definition sites)
D) No φ needed (B0 dominates all)
E) B0 and B4 (entry and exit)
F) All blocks need φ-functions
G) B4 only (join point after definitions)
H) B3 and B4 (B3 needs φ even without defining x)

ANSWER: G) B4 only (join point after definitions)

### Question 6: Strength Reduction in Loops

What is the CORRECT strength-reduced form with auxiliary induction variables?
```
// Orgional loop
for (int i = 0; i < n; i++) {
    a[i * 4] = i * 4 + 10;
    b[i * 4] = i * 8;
}
// After strength reduction
int t1 = 0, t2 = 10;
for (int i = 0; i < n; i++) {
    a[???] = ??? + 10
    b[???] = ???
}
```
A) t1=0; t2=0; ... a[t1]=t1+10; b[t2]=t2; ... t1+=4; t2+=8;
B) t1=0; t2=0; ... a[t1]=t2+10; b[t1]=t2; ... t1+=4; t2+=8;
C) t=0; ... a[t]=t+10; b[t]=t*2; ... t+=4;
D) t1=0; t2=0; ... a[t1]=t1+10; b[t1]=t2; ... t1+=4; t2+=4;
E) t1=0; t2=10; ... a[t1]=t2; b[t1]=t1*2; ... t1+=4; t2+=4;
F) Cannot apply strength reduction (multiplications are necessary)
G) t=0; ... a[t]=t+10; b[t]=t+t; ... t+=4;
H) t1=0; t2=0; t3=10; ... a[t1]=t3; b[t1]=t2; ... t1+=4; t2+=8; t3+=4;

ANSWER: H) t1=0; t2=0; t3=10; ... a[t1]=t3; b[t1]=t2; ... t1+=4; t2+=8; t3+=4;

### Question 7: Strength Reduction Profitability

Under which condition is strength reduction UNPROFITABLE?
```
//Loop with address caluculation
for (i = 0; i < 100; i++) {
    sum += array[i*stride];
}

// Architecture characteristics
// interger Multiple : 3cycle latency, 1/cycle throughput
// interger add : 1cycle latency, 2/cycle throughput
// 16 general-purpose registers
// Current register pressure: 14 in use

// After strength reduction
p = &array[0]
for (i = 0; i < 100; i++) {
    sum += *p;
    p += stride;
}
```
A) stride = 1 (unit stride)
B) stride = 1024 (large stride)
C) The loop body is very large (50+ instructions)
D) When the extra live variable p causes register spilling
E) When stride is not a compile-time constant
F) Never unprofitable - strength reduction always improves performance
G) When the loop has only 2 iterations (n=2)
H) When multiplication has 1-cycle latency

ANSWER: D) When the extra live variable p causes register spilling

### Question 8: Loop Invariant Code Motion vs. Strength Reduction

Which optimization applies to which expression?
```
//original loop
for (i = 0; i < 100; i++) {
    a[i] = b [i] * factor + offset;
}
```
A) Strength reduction: i; LICM: factor, offset (but not b[i] * factor)
B) Strength reduction: i; LICM: factor, offset, b[i] * factor
C) Strength reduction: i, b[i]; LICM: offset
D) Strength reduction: none; LICM: factor, offset
E) Strength reduction: i, b[i] * factor, offset; LICM: none
F) Strength reduction: i, b[i], b[i] * factor; LICM: factor + offset
G) Strength reduction: i, b[i] * factor; LICM: offset
H) Both optimizations apply to all expressions

ANSWER: A) Strength reduction: i; LICM: factor, offset (but not b[i] * factor)
### Question 9: Horner's Rule for Polynomial Evaluation

How many multiplications does each form require?
```
//Evaluate polynomial 2x^3 + 3x^2 + 4x + 5
// Form A (naive)
result = 2*x*x*x + 3*x*x + 4*x + 5;
// Form B (Horners'rule)
result = ((2*x + 3)*x + 4)*x + 5;
```
A) Form A: 6 muls; Form B: 3 muls
B) Form A: 4 muls; Form B: 3 muls
C) Form A: 3 muls; Form B: 3 muls (equal)
D) Form A: 6 muls; Form B: 4 muls
E) Form A: 5 muls; Form B: 3 muls
F) Form A: 7 muls; Form B: 3 muls
G) Form A: 3 muls; Form B: 2 muls
H) Depends on compiler optimization level

ANSWER: A) Form A: 6 muls; Form B: 3 muls
### Question 10: SSA-Based Constant Folding and Dead Code Elimination

After constant folding and dead code elimination, what code remains?
```
.B0:
    x1=5
    y1=10
    if (x1<3) goto B1 else goto B2
.B1:
    z1 = x1 + y1
    w1 = z1 * 2
    goto B3
.B2:
    z2 = x1 - y1
    w2 = z2 * 3
    goto B3
.B3:
    z3 = φ(z1,z2)
    w3 = φ(w1,w2)
    result = (z3+w3)
```
A) B0: if(5<3) ...; B1: ...; B2: z₂=-5; w₂=-15; B3: result=-20
B) B0: x₁=5; y₁=10; B2: z₂=-5; w₂=-15; B3: result=-20
C) B0: x₁=5; y₁=10; B3: result=φ(-20, -20)
D) All blocks remain (cannot eliminate φ-functions)
E) B0: goto B2; B2: result=-20
F) B0: result=-20
G) Only B3 remains with result computation
H) B2: z₂=-5; w₂=-15; B3: result=-20

ANSWER: F) B0: result=-20
### Question 11: SSA-Based Value Range Propagation and DCE

After value range analysis proves a₂ is always ≤ 14, which code remains?
```
B0:
    a1=10
    b1=20
    goto B1
B1:
    a2 = φ(a1,a3)
    if (a2 > 100) goto B2 else goto B3
B2:
    c1 = a2 /2
    goto B4
B3:
    a3 = a2 + 1
    if (a3 < 15) goto B1 else goto B4
B4:
    a4 = φ(a2,a3)
    c2 = φ(c1,0)
    result = a4 + c2
```
A) All blocks remain unchanged
B) B0-B1-B3-B4 remain; B2 deleted; c₂=0 in B4
C) B2 is eliminated; B4: a₄=φ(⊥,a₃); c₂=0; result=a₄
D) Entire loop eliminated; B0: result=14
E) B2 eliminated; B3: produces values 11,12,13,14
F) B0: a₁=10; B1-B3: loop executes; B4: result=a₄
G) Only B4: result=14 remains
H) Cannot eliminate B2 (may be reachable)

ANSWER: B) B0-B1-B3-B4 remain; B2 deleted; c₂=0 in B4
### Question 12: SSA Constant Propagation with CFG

After constant propagation and dead code elimination, which blocks remain?
```
B0(x=7,y=3) -> B1
B1(if(x>5)) -> B2 ,B3
B2(z=x+y) -> B4
B3 (Z=x*2) -> B4
B4 (w=φ(z,z),r=w-y)
```
A) All blocks B0-B4 remain
B) B0, B3, B4 remain (B2 eliminated)
C) Only B0 remains with r=7
D) B0, B1, B2, B4 remain
E) B0, B2, B4 remain (B3 eliminated)
F) B0, B4 remain (B1-B3 eliminated)
G) Only B4 remains with r=7
H) Cannot eliminate any blocks (φ-function prevents DCE)

ANSWER: E) B0, B2, B4 remain (B3 eliminated)
### Question 13: SSA Constant Propagation with Nested Branches

After constant propagation, what is the value of result in B6?
```
B0(a=4,b=6) -> B1
B1(if(a>5)) -> B2 ,B3
B2(c=a+b if c>8) -> B4 , B5
B3 (Z=a*b) -> B6
B4 (d=10)
B5 (d=20)
B6 ( d2 = φ (B4:d,B5:d,B3:T) result = d2+c)

```
A) result = 30
B) result = 34
C) result = 20
D) result = 14
E) result = ⊤ (overdefined)
F) result = 28
G) Cannot determine
H) result = 24

ANSWER: E) result = ⊤ (overdefined)
### Question 14: SSA Dead Code Elimination with CFG

After constant propagation and DCE, what remains at B4?
```
B0(x=2,y=8) -> B1
B1(z=x*y , if(z>10)) -> B2 , B3
B2(w=z-5)->B4
B3(w=z+5)->B4
B4(w2=φ(w1,w3) , r=w2*2)
```
A) w₂=φ(11,21); r=w₂*2
B) w₂=11; r=22
C) w₂=21; r=42
D) r=42
E) All code remains unchanged
F) w₂=φ(11,11); r=22
G) r=22
H) Cannot determine

ANSWER: G) r=22
### Question 15: SSA Constant Folding with Multiple φ-Functions

After constant propagation, what is result at B4?
```
B0(a=6,b=3) -> B1
B1(if(a>b)) -> B2 , B3
B2(x=a-b,y=10)->B4
B3(x=b-a,y=20)->B4
B4(x2=φ(x1,x3) , y2=φ(y1,y3) , result = x2 + y2)
```

A) result = 13
B) result = 23
C) result = 17
D) result = -17
E) result = ⊤
F) result = 30
G) result = 10
H) result = 7

ANSWER: A) result = 13
