# Week 5: Affine Partitioning

**Course Week**: 5
**Topic**: Affine Partitioning

Affine Partitioning (AP) is a powerful compiler optimization technique that transforms programs, particularly those with nested loops and array accesses, into a form suitable for parallel execution on multi-core architectures. It aims to minimize communication between threads and improve memory locality to reduce cache misses.

## Introduction: Beyond Traditional Dependence Analysis

Traditional parallelization often relies on **dependence distance vectors**, which describe the distance between dependent loop iterations. For a nest of `k` loops, a distance vector `d = <d1, ..., dk>` indicates that iteration `i = <i1, ..., ik>` depends on iteration `i' = <i1+d1, ..., ik+dk>`. While useful, this approach has limitations.

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

**Unimodular Transforms (UT)**, a subset of affine transforms, involve square, integer matrices with a determinant of `+1` or `-1`. UTs handle transformations like interchange, reversal, and skewing. They generally do not suffer from phase-ordering problems (the order in which transformations are applied).

## Limitations of Unimodular Transforms (UT)

While powerful, UTs have limitations that AP addresses:

1.  **Only Handle Perfect Nested Loops**: UTs typically only apply to loops where all statements are within the innermost loop of a perfectly nested structure. Code not in the innermost loop or loops with multiple child loops are not directly supported. AP solves this by mapping at the **statement-level**.
2.  **Cannot Reindex at Statement Level**: UTs cannot easily reindex loops at the statement level to change loop-carried dependencies into loop-independent ones. AP provides a uniform way to achieve loop reindexing.
3.  **Dependences Must Be in Distance Vector Format**: Some complex dependencies cannot be formulated simply as distance vectors but instead appear as linear combinations of induction variables. AP can parallelize such loops.

## Affine Partitioning Approach

For each statement `k` in the program, AP applies a linear transformation `Φk` to its induction variables `i_k`:
`Φk(i_k) = c_k1 * i_k1 + c_k2 * i_k2 + ... + c_ks * i_ks + c_k`

Each statement can have a different transformation. The objective is to find a common outermost loop structure for all statements, or to partition the iteration space for parallel execution.

### Space Partitioning

**Space partitioning** aims to partition iterations to different processors such that there are no loop-carried dependencies after the transformation. This means all data dependences become independent of the outermost loop, allowing synchronization-free parallel execution.

*   **Goal**: Assign each iteration instance `(i, j)` to a processor `P` (e.g., `P = a*i + b*j + c`).
*   **Example**: For a loop with statements `S1` and `S2`, if a dependence exists (e.g., `S1[i,j]` depends on `S2[i,j+1]`), we find affine partitions `P_S1` and `P_S2` such that the dependence constraint `P_S1 <= P_S2` is satisfied, and ideally `P_S1 = P_S2` for parallel execution within the same partition.

### Time Partitioning

**Time partitioning** is used to parallelize loops even when full synchronization-free parallelism (space partitioning) cannot be found. The goal is to transform dependencies into loop-carried dependencies for a new outermost loop `P`, requiring synchronization only at the end of each iteration of `P`. This is useful for wavefront dependencies, where computation proceeds in "waves" across the iteration space.

*   **Goal**: Ensure that for all dependencies `x1 -> x2`, the transformed time `T(x1)` must be less than or equal to `T(x2)`, meaning `x1` executes no later than `x2` in the new time dimension.

### Legal Affine Partition

A transformation `Φ` is a **legal affine partition** if for every dependence from instance `x1` (of statement `S1` with iteration vector `I1`) to instance `x2` (of statement `S2` with iteration vector `I2`), `x1` must be executed no later than `x2`. This means:

*   The transformed iteration vector `Φ(I1)` must be lexicographically ordered before or equal to `Φ(I2)`.
*   If `Φ(I1) = Φ(I2)`, then `S1` must appear before `S2` in the transformed loop body.

## Overall Algorithm to Maximize Degrees of Parallelism

1.  **Construct Program Dependence Graph (PDG)**: Identify data and control dependencies.
2.  **Partition Statements into Strongly Connected Components (SCCs)**: Each SCC represents a group of statements that are mutually dependent.
3.  **Apply Space Partitioning**: For each SCC, try to find synchronization-free parallelism.
4.  **Apply Time Partitioning**: If space partitioning is not fully successful, apply time partitioning to find parallelism that requires synchronization between iterations of the outermost loop.

## Key Concepts

*   **Lexicographical Order**: Used to compare iteration vectors. A vector `A` is lexicographically smaller than `B` if at the first position where they differ, `A` has a smaller element.
*   **Lexicographically Positive Vector**: A vector `V` is lexicographically positive if its first non-zero element is positive. This is crucial for ensuring that loop-carried dependences remain valid after transformation.
*   **Theorem**: A positive combination of two legal affine partitions is also a legal affine partition. This allows combining different partitions to achieve more complex transformations.

Affine Partitioning provides a systematic and powerful framework for optimizing programs for parallel execution by rigorously analyzing and transforming iteration spaces and data dependencies.