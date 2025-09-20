# Week 5: Loop Transformations for Parallelism and Locality

**Course Week**: 5
**Topic**: Loop Transformations for Parallelism and Locality

This note explores various loop transformations used in compilers to enhance both parallelism and memory locality, which are crucial for performance on shared-memory multiprocessors. It revisits concepts from Affine Partitioning and introduces Blocking (Tiling) as a key technique.

## Introduction: Iteration Space and Performance Goals

### Iteration Space
For `n`-deep nested loops, the **iteration space** can be visualized as an `n`-dimensional polytope. Each iteration corresponds to a coordinate in this space. By convention, iteration indices are incremented in the loop. The sequential execution order follows a lexicographic order (e.g., `[0,0], [0,1], ..., [0,7], [1,1], ...`).

### Performance on Shared-Memory Multiprocessors
On shared-address space multiprocessors, achieving high performance requires optimizing for two primary factors:
*   **Parallelism**: Distributing computations across multiple processors.
*   **Locality**: Ensuring that data accessed by a processor is kept close to that processor (e.g., in its cache) to minimize slow memory accesses.

**Important**: Parallelism alone does not guarantee speedup; locality is equally critical.

*   **Parallel performance**: Focuses on minimizing communication between processors by having operations using the same data executed on the same processor.
*   **Sequential performance**: Focuses on minimizing cache misses by having operations using the same data executed close in time.

## Loop Transformations

Compilers employ various loop transformations to restructure the iteration space for better parallelism and locality.

### Loop Permutation (Loop Interchange)

Loop interchange swaps the order of nested loops. This can change the memory access pattern, significantly impacting cache performance and exposing different levels of parallelism.

*   **Example**:
    ```
    // Original loop
    for I = 1 to 4
       for J = 1 to 3
          Z[I,J] = Z[I-1,J]

    // After interchange
    for J’ = 1 to 3
       for I’ = 1 to 4
          Z[I’,J’] = Z[I’-1,J’]
    ```
    The original accesses `Z[I,J]` and `Z[I-1,J]` might be stride-1 if `J` is the innermost loop and arrays are column-major, or stride-N (N = size of J dimension) if arrays are row-major. Interchanging changes this access pattern.

### Affine Partitioning for Maximum Parallelism

As discussed in the previous note, Affine Partitioning (AP) is a powerful framework that generalizes many loop transformations. It can find optimal parallelization schemes by mapping iteration instances to processor IDs and execution times.

*   **Algorithm**: Finds affine partition mappings for each instruction, e.g., `S1: Execute iteration (i, j) on processor i-j.`
*   **SPMD Code Generation**: AP can generate Single Program, Multiple Data (SPMD) code that, despite its complexity, is mechanically derived from the affine partition mappings.
*   **Maximizing Rank**: For communication-free parallelism, AP aims to find affine mappings `C1*i1 + c1` and `C2*i2 + c2` such that for every pair of data-dependent accesses `F1*i1 + f1` and `F2*i2 + f2`, if `F1*i1 + f1 = F2*i2 + f2`, then `C1*i1 + c1 = C2*i2 + c2`. The objective is to maximize the rank of `C1, C2`, as the rank of partitioning equals the degree of parallelism.
    *   Rank 0: All iterations map to the same processor (sequential).
    *   Rank 1: One dimension of parallelism (e.g., each column processes independently).
    *   Rank 2: Two dimensions of parallelism (e.g., each cell processes independently).

*   **Example**: For `Z[I,J] = Z[I-1,J]`, an affine partitioning `p = j` means `Z[I,J]` and `Z[I-1,J]` for the same `J` value are processed on the same processor `p`. This transforms the inner loop `J` into a parallel dimension.

## Blocking (Tiling) for Locality

**Blocking**, also known as **Tiling**, is a crucial loop transformation for improving data locality, especially for deeply nested loops that operate on large data structures, such as matrix multiplication. It works by dividing the iteration space into smaller "blocks" or "tiles" and processing each block completely before moving to the next.

### Why Blocking?

Without blocking, a program might access data elements that are far apart in memory, leading to many cache misses. Blocking reorders computations to maximize data reuse within the faster levels of the memory hierarchy (caches).

### Blocking Example: Matrix Multiplication

Consider standard matrix multiplication: `Z[i,j] = Z[i,j] + X[i,k] * Y[k,j]`

```
// Before Blocking
for (i = 0; i < n; i++) {
   for (j = 0; j < n; j++) {
      for (k = 0; k < n; k++) {
         Z[i,j] = Z[i,j] + X[i,k]*Y[k,j];
      }
   }
}
```

This code has poor cache performance for large `n` because the inner `k` loop iterates through `X[i,k]` and `Y[k,j]` sequentially, potentially evicting cache lines for `Z[i,j]` or other rows/columns before they are fully reused.

```
// After Blocking (with block size B)
for (ii = 0; ii < n; ii = ii+B) {        // Outer block loop for i
   for (jj = 0; jj < n; jj = jj+B) {    // Outer block loop for j
      for (kk = 0; kk < n; kk = kk+B) { // Outer block loop for k
         for (i = ii; i < min(n, ii+B); i++) {
            for (j = jj; j < min(n, jj+B); j++) {
               for (k = kk; k < min(n, kk+B); k++) {
                  Z[i,j] = Z[i,j] + X[i,k] * Y[k,j];
               }
            }
         }
      }
   }
}
```

In the blocked version, the computation is reorganized to work on `B x B` sub-matrices. The innermost loops now iterate only within a small block. This ensures that a `B x B` tile of `X`, `Y`, and `Z` can be loaded into cache and largely reused before new data needs to be fetched from main memory. This significantly reduces cache misses.

### Experimental Results

Empirical data, such as results from the NAS Parallel Benchmarks (e.g., `chotst` kernel), show that:
*   **Unimodular Transformations + Blocking**: Provide good speedup.
*   **Affine Partitioning + Blocking**: Often yield even better speedup, demonstrating the enhanced capabilities of AP to handle more complex loop structures and dependencies, leading to better parallelism *and* locality.

## Conclusions

*   **Parallelism is plentiful in numeric code, but locality is paramount for achieving actual speedup.**
*   **Two kinds of transforms are essential:**
    *   **Affine Partitioning**: Maximizes the degree of parallelism without communication by mapping operations using the same data to the same processor.
    *   **Blocking (Tiling)**: Exploits locality across multiple dimensions by reorganizing loop nests to enable data reuse within cache, minimizing cache misses.

By combining these powerful loop transformations, compilers can significantly enhance the performance of complex numerical applications on modern multi-core architectures.