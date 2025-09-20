# Week 5: Dependence and Parallelization

**Course Week**: 5
**Topic**: Dependence and Parallelization

This note covers the fundamental concepts of dependence analysis and how it is applied to parallelize numerical applications, particularly focusing on loop-level parallelism and interprocedural parallelization.

## I. Basic Parallelization

Parallelization aims to execute different parts of a program concurrently to improve performance. A common target for parallelization is loops, especially "DoAll" loops.

*   **DoAll Loops**: These are loops whose iterations are entirely independent of each other. This means there are no data dependence edges that cross iteration boundaries (i.e., no loop-carried dependences). The number of iterations in such loops often scales with the problem size, making them excellent candidates for distributing work across multiple processors.

### Data Dependence Edges

A crucial step in parallelization is identifying **data dependence edges**. A dependence exists if two operations access the same memory location, and at least one of them is a write.

*   **Loop-Carried Dependence**: A data dependence edge is loop-carried if it connects operations in different iterations of a loop. If a loop has no loop-carried dependences, it is a DoAll loop.

### Recall: Types of Data Dependences

1.  **True Dependence (RAW - Read After Write)**: An operation reads a value that was written by a previous operation.
    ```
    a = ... // Write
    ... = a // Read
    ```
2.  **Anti-Dependence (WAR - Write After Read)**: An operation writes to a location that was previously read by another operation. This can often be resolved through **privatization** (giving each iteration its own copy of a variable).
    ```
    ... = a // Read
    a = ... // Write
    ```
3.  **Output Dependence (WAW - Write After Write)**: An operation writes to a location that was previously written by another operation. This can also often be resolved through privatization.
    ```
    a = ... // Write 1
    a = ... // Write 2
    ```

## II. Formulating Data Dependence Analysis

Data dependence analysis involves determining if there exist iteration instances where the same memory location is accessed. This is particularly challenging for array accesses.

### Affine Array Accesses

Data dependence analysis primarily focuses on **affine array accesses**, where array indices are expressed as linear functions of loop indices and integer constants.

*   **Form**: `c_n * i_n + c_{n-1} * i_{n-1} + ... + c_1 * i_1 + c_0`
    *   `i_x`: loop indices
    *   `c_x`: integer constants
    *   `c_0`: a constant term

### Formulating Dependences (Example)

Consider a loop:
```
FOR i := 2 to 5 do
   A[i-2] = A[i] + 1;
```
A dependence exists between `A[i]` (read) and `A[i-2]` (write) if there exist two iterations `ir` (for read) and `iw` (for write) within the loop bounds such that they access the same array element:
`∃ integers iw, ir such that 2 ≤ iw, ir ≤ 5 and ir = iw - 2`

To check for a dependence between two write accesses `A[i-2]` and `A[i-2]`, we look for:
`∃ integers iw, iv such that 2 ≤ iw, iv ≤ 5 and iw - 2 = iv - 2`
Additionally, to rule out self-dependences within the same iteration, we add the constraint `iw ≠ iv`.

### Memory Disambiguation

Determining precisely whether two memory accesses refer to the same location is known as **memory disambiguation**. This problem is generally **undecidable at compile time** for arbitrary programs.

### Domain of Data Dependence Analysis

Due to undecidability, dependence analysis is typically restricted to scenarios where:

*   Loop bounds are known.
*   Array indices are affine functions of loop variables.

For multi-dimensional arrays, dependence is checked for each dimension separately. If an array index is non-affine, that dimension is often ignored, or a conservative assumption of dependence is made.

### Complexity of Data Dependence Analysis

Checking for data dependence between two array accesses essentially boils down to solving a system of linear Diophantine equations (equations where only integer solutions are sought) subject to loop bound constraints.

*   This is equivalent to **Integer Linear Programming (ILP)**, which is an NP-complete problem. This means finding an exact solution for large, complex systems can be computationally expensive.

### Data Dependence Analysis Algorithm

Compilers use a multi-pronged approach to manage the complexity of ILP:

1.  **Simple Tests**: A series of quick, relatively inexpensive tests are applied first.
    *   **GCD Test**: Checks if the greatest common divisor of the coefficients of the loop variables divides the constant term in the dependence equation. If not, there's no integer solution.
    *   **Range Overlap Test**: Checks if the ranges of array indices accessed by different iterations overlap.
    *   **Memoization**: Stores results of simple tests to avoid re-computation.
2.  **Fourier-Motzkin Elimination**: If simple tests are inconclusive, a more expensive algorithm like Fourier-Motzkin Elimination is used.
    *   This technique eliminates variables from a set of linear inequalities to check if a real solution exists. If no real solution exists, then there is no integer solution either.
    *   **Heuristics for Integer Solutions**: If a real solution is found but no integer solution, heuristics are applied. For example, if `x = 0.5` is a solution, two subproblems are created by adding `x ≤ 0` and `x ≥ 1` as constraints.

### Relaxing Dependences

Even if a dependence is detected, it might be possible to transform the code to remove it, enabling parallelization.

*   **Privatization**: Creating a private copy of a variable or array for each loop iteration eliminates anti-dependences and output dependences.
    ```
    // Scalar Privatization
    for i = 1 to n
       t = (A[i] + B[i]) / 2; // 't' can be privatized per iteration
       C[i] = t * t;

    // Array Privatization
    for i = 1 to n
       for j = 1 to n
          t[j] = (A[i,j] + B[i,j]) / 2; // 't' array can be privatized per outer iteration 'i'
       for j = 1 to n
          C[i,j] = t[j] * t[j];
    ```
*   **Reduction**: Operations like summation (`sum = sum + A[i]`) involve a loop-carried true dependence. These can be parallelized using special techniques (e.g., parallel reduction algorithms) that combine partial results.

## III. Interprocedural Parallelization

As per Amdahl's Law, coarse-grain parallelism is crucial for significant speedups. Interprocedural analysis extends dependence analysis across function call boundaries.

*   **Interprocedural Symbolic Analysis**: Finds affine array indices that are affine expressions of outer loop indices, even across procedure calls.
*   **Interprocedural Parallelization Analysis**:
    *   Uses **summaries of array regions accessed** by functions. If regions accessed by different parallel tasks do not intersect, then parallelism exists.
    *   Identifies privatizable scalar variables and arrays across procedure boundaries.
    *   Finds scalar and array reductions across procedure boundaries.

## Conclusions

*   **Basic Parallelization**: DoAll loops (loops without loop-carried data dependences) are primary targets.
*   **Data Dependence for Affine Indices**: This problem is equivalent to integer linear programming, solved using a combination of simple tests and more expensive algorithms like Fourier-Motzkin Elimination.
*   **Coarse-Grain Parallelism**: Essential for achieving significant speedups, often requiring interprocedural analysis.
*   **User Interaction**: For unresolved or complex dependences, compilers may ask users for input or directives to guide parallelization.