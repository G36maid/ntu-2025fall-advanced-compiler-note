# Week 5: Affine Partitioning

**Course Week**: 5
**Topic**: Affine Partitioning
**Reading Assignment**: Chapter 11 of Dragon Book

Affine Partitioning (AP) is a powerful compiler optimization technique that transforms programs, particularly those with nested loops and array accesses, into a form suitable for parallel execution on multi-core architectures. It aims to minimize communication between threads and improve memory locality to reduce cache misses.

## Introduction: Beyond Traditional Dependence Analysis

Traditional parallelization often relies on **dependence distance vectors**, which describe the distance between dependent loop iterations. For a nest of `k` loops, a distance vector `d = <d1, ..., dk>` indicates that iteration `i = <i1, ..., ik>` depends on iteration `i\' = <i1+d1, ..., ik+dk>`. While useful, this approach has limitations.

### Programming Models and Compiler Challenges

The shift to multi-core architectures has brought various programming models, moving beyond the "lazy-boy programming era." These include:
*   **Basic pthread kind of model**: Lock-based synchronization.
*   **Transactional Memory (TM)**: Aims for atomicity, coherence, and isolation of shared mutable states. While a promising concept (resurfaced in 2004 due to multi-cores), it has faced challenges in tuning for both productivity and performance.
*   **Functional**: Symbolic machine.
*   **Data-Parallel**: Graphics Processors.
*   **Data Flow**: Network Processors.
*   **Multicore-for-latency**: Helper threads.

To address the complexities of these models and optimize for performance, compilers perform various bottom-up analyses:
*   **Control Flow Analysis (CFA)**: Basic block, dominance, loop, interval analysis, reducibility.
*   **Data Flow Analysis (DFA)**: Iterative data flow analysis, lattice.
*   **Static Single Assignment (SSA)**: Dominance frontier, SSA & de-SSA, SSA-based analysis.
*   **Dependence Analysis**: Dependence DAG, dependence testing, dependence distance.

## Affine Partitioning: A Unifying Theory

AP provides a more rigorous and generalized framework for transforming code within the "affine domain."

### Problem Statement

AP transforms sequential or parallel code into optimal partitions, where each partition can be mapped to a thread on a multi-core processor. The core idea is to go back to the "first principle" of abstraction: focus on partitioning the workload rather than just analyzing dependences. This results in a powerful, unifying theory that aims to:

*   Minimize communication between threads.
*   Improve memory locality to reduce cache misses.

### Optimization Goals: Parallelism & Locality

AP encompasses and unifies many classic loop transformations, including:

*   Loop Interchange
*   Loop Reversal
*   Loop Skewing
*   Loop Distribution
*   Loop Fusion
*   Loop Reindexing
*   Tiling/Blocking
*   Array Privatization / Expansion / Contraction
*   Statement Reordering

These transformations were traditionally applied in an ad-hoc manner, but AP provides a systematic way to achieve them.

### Unified Transformation Framework

AP views loop transformations as mapping the *old iteration space* to a *new iteration space* using **affine transforms**. This means the new loop structures and their dependencies are described by affine expressions.

**Unimodular Transforms (UT)**, a subset of affine transforms, involve square, integer matrices with a determinant of `+1` or `-1`. UTs handle transformations like interchange, reversal, and skewing. They generally do not suffer from phase-ordering problems (the order in which transformations are applied), which is also true for AP transforms.

### Examples of Unimodular Transforms

*   **Loop Interchange**: Swapping inner and outer loops.
    ```example.c#L1-7
    For J = 1 to 3
      For I = 1 to 4
        A[J,I] = ...

    // After interchange
    For J’ = 1 to 4
      For I’ = 1 to 3
        A[I’,J’] = ...
    ```
*   **Loop Reversal**: Reversing the direction of a loop.
    ```example.c#L1-6
    For J = 1 to 4
      For I = 1 to 3
        A[I,J] = ...

    // After reversal
    For J’ = 1 to 4
      For I’ = -3 to -1
        A[-I’,J’] = ...
    ```
*   **Loop Skewing**: Transforming the iteration space to remove dependencies.
    ```example.c#L1-6
    For J = 1 to 4
      For I = -3 to -1
        A[-I,J] = ...

    // After skewing
    For J’ = 1 to 4
      For I’ = J’-3 to J’-1
        A[J’-I’,J’] = ...
    ```

## Limitations of Unimodular Transforms (UT)

While powerful, UTs have limitations that AP addresses:

1.  **Only Handle Perfect Nested Loops**: UTs typically only apply to loops where all statements are within the innermost loop of a perfectly nested structure. Code not in the innermost loop or loops with multiple child loops are not directly supported. For example:
    ```example.c#L1-10
    For I = 1 to X
      For J = 1 to Y
        B[I,J] = ...        // Not in innermost loop
        For L = 1 to W
          A[I,J,L]=...
          ....
        For L = 1 to W2     // Loop with multiple child loops
          ....
    ```
    AP solves this by mapping at the **statement-level**, allowing uniform transformation for multiple loops, such as loop fusion.
2.  **Cannot Reindex at Statement Level**: UTs cannot easily reindex loops at the statement level to change loop-carried dependencies into loop-independent ones. For instance, to transform a dependency `T[I]` on `T[I+1]`:
    ```example.c#L1-5
    For I = 1  to  4
      For J = 1 to 4
        T[I] = ...
        ...   = T[I+1]
    ```
    AP provides a uniform way to achieve loop reindexing.
3.  **Dependences Must Be in Distance Vector Format**: Some complex dependencies cannot be formulated simply as distance vectors but instead appear as linear combinations of induction variables. AP can parallelize such loops, for example, in cases where `C[I, J] = A[J, I]` where the dependency is on `J` for `C` and `I` for `A`.

## Affine Partitioning Approach

For each statement `k` in the program, AP applies a linear transformation `Φk` to its induction variables `i_k`:
`Φk(i_k) = c_k1 * i_k1 + c_k2 * i_k2 + ... + c_ks * i_ks + c_k`

Each statement can have a different transformation. The objective is to find a common outermost loop structure for all statements, or to partition the iteration space for parallel execution.

### Space Partitioning

**Space partitioning** aims to partition iterations to different processors such that there are no loop-carried dependencies after the transformation. This means all data dependences become independent of the outermost loop, allowing synchronization-free parallel execution.

*   **Goal**: Assign each iteration instance `(i, j)` to a processor `P` (e.g., `P = a*i + b*j + c`).
*   **Example**: Consider a loop with statements `S1` and `S2`:
    ```example.c#L1-5
    FOR i = 1 TO n DO
      FOR j = 1 TO n DO
        A[i,j] = A[i,j]+B[i-1,j]; (S1)
        B[i,j] = A[i,j-1]*B[i,j]; (S2)
    ```
    There are dependencies: `S1[i,j]` depends on `B[i-1,j]` (from previous `S2` iteration) and `S2[i,j]` depends on `A[i,j-1]` (from previous `S1` iteration). Specifically, `S1[i,j]` depends on `S2[i,j+1]` and `S2[i,j]` depends on `S1[i+1,j]`.

    If we assume affine partitions `P = a*i + b*j + c` for S1 and `P = d*i + e*j + f` for S2, we can derive conditions for these partitions. A solution could be `P_S1 = i - j` and `P_S2 = i - j + 1`. This effectively transforms the loop into:
    ```example.c#L1-9
    For P = 1-n TO n DO
      For i = 1 TO n DO
        IF 1<=i-P<=n
          A [i, i- P] = A[i,i-P]+B[i-1,i-P]
        IF 1<=i-P+1<=n
          B [ i, i-P+1] = A[i,i-P]*B[i,i-P+1]
    ```

*   **Second Space Partition Example**: Parallelizing loops without simple distance vectors.
    ```example.c#L1-4
    For I = 1 to N DO
      For J = 1 to I DO
        A[I, J] = B[I, J]       (S1)
        C[I, J]= A[J, I]        (S2)
    ```
    Here, the dependency `C[I,J]=A[J,I]` means `S2[I,J]` depends on `S1[J,I]`.
    We can find two independent solutions for affine partitions: `Φ1(I,J) = I` and `Φ2(I,J) = J`. This leads to a parallel structure like:
    ```example.c#L1-8
    For P = 1 to N DO
      For I = 1 to N DO
        For J = 1 to I DO
          IF P == I
            A[P, J] = B[P, J]
          IF P == J
            C[I, P] = A[P,I]
    ```
    This example also extends to 2-order affine partitions, which might require sorting `P1, P2` at runtime.

### Time Partitioning

**Time partitioning** is used to parallelize loops even when full synchronization-free parallelism (space partitioning) cannot be found. The goal is to transform dependencies into loop-carried dependencies for a new outermost loop `P`, requiring synchronization only at the end of each iteration of `P`. This is useful for wavefront dependencies, where computation proceeds in "waves" across the iteration space.

*   **Goal**: Ensure that for all dependencies `x1 -> x2`, the transformed time `T(x1)` must be less than or equal to `T(x2)`, meaning `x1` executes no later than `x2` in the new time dimension. This implies all dependencies become loop-carried dependencies for the new `P` loop.
*   **Example**: Consider loops with wavefront dependencies:
    ```example.c#L1-4
    For i=1  to n
      For j=1 to m
        A[i,j]=B[i-1,m-j+1];    (S1)
        B[i,j]=A[i,j-1];        (S2)
    ```
    No space partition can be found for full synchronization-free parallelism. A time partition might be `P_S1 = 2*i` and `P_S2 = 2*i + 1`, ensuring `S1` in iteration `i` happens before `S2` in iteration `i`, and allowing dependencies to propagate across `P` iterations.

### Legal Affine Partition

A transformation `Φ` is a **legal affine partition** if for every dependence from instance `x1` (of statement `S1` with iteration vector `I1`) to instance `x2` (of statement `S2` with iteration vector `I2`), `x1` must be executed no later than `x2`. This means:

*   The transformed iteration vector `Φ(I1)` must be lexicographically ordered before or equal to `Φ(I2)`.
*   If `Φ(I1) = Φ(I2)`, then `S1` must appear before `S2` in the transformed loop body.
This ensures that the instance `Φ(I1)` should be executed no later than `Φ(I2)` in loop `P` after the affine partition.

## Overall Algorithm to Maximize Degrees of Parallelism

1.  **Construct Program Dependence Graph (PDG)**: Identify data and control dependencies.
2.  **Partition Statements into Strongly Connected Components (SCCs)**: Each SCC represents a group of statements that are mutually dependent.
3.  **Apply Space Partitioning**: For each SCC, try to find synchronization-free parallelism.
4.  **Apply Time Partitioning**: If space partitioning is not fully successful, apply time partitioning to find parallelism that requires synchronization between iterations of the outermost loop.

## Key Concepts

*   **Lexicographical Order**: Used to compare iteration vectors. A vector `A` is lexicographically smaller than `B` if at the first position where they differ, `A` has a smaller element. For example, `<0,1>` is lexicographically smaller than `<0,2>`, and `<0,3>` is smaller than `<1,0>`.
*   **Lexicographically Positive Vector**: A vector `V` is lexicographically positive if its first non-zero element is positive. This is crucial for ensuring that loop-carried dependences remain valid after transformation. For instance, `<1,0> - <0,1> = <1,-1>` which is lexicographically positive. `For loop-carried dependency, the distance vector must be lexicographically positive`.
*   **Theorem**: A positive combination of two legal affine partitions is also a legal affine partition. This allows combining different partitions to achieve more complex transformations because affine partition is linear.

Affine Partitioning provides a systematic and powerful framework for optimizing programs for parallel execution by rigorously analyzing and transforming iteration spaces and data dependencies.
