# Vectorization in LLVM: SLP & Custom Instructions

Notes from the presentation "SLP Pass & Custom Instruction in LLVM Compiler" for the course CSIE5054 Advanced Compiler Design.

## Part 1: Superword-Level Parallelism (SLP)

### Introduction to SLP

-   **Superword-Level Parallelism (SLP)** is a vectorization technique proposed in 2000.
-   It finds parallelism within a single basic block by identifying isomorphic and independent statements that can be packed together and executed with vector/SIMD instructions.
-   LLVM has two main auto-vectorizers: the **Loop Vectorizer** and the **SLP Vectorizer**.
-   Both are enabled by default at optimization levels `-O2`, `-O3`, and `-Os`.
-   The pass pipeline runs the Loop Vectorizer first, then the SLP Vectorizer.

### SLP & Loop-Aware SLP

-   The original SLP concept applies to any basic block.
-   LLVM's implementation is inspired by GCC's **Loop-Aware SLP**, which focuses on finding SLP opportunities *inside* loops, where the payoff is often highest.
-   In contrast to GCC, which integrates loop and SLP vectorization into a single pass, LLVM uses two distinct passes.

### SLP Vectorizer Pass Overview

-   The pass is implemented in the `SLPVectorizer` class, which inherits from `PassInfoMixin`.
-   The entry point is the `run()` method.
-   The core logic resides in `runImpl()`, which initializes a `BoUpSLP` (Bottom-Up SLP) object to analyze the code.
-   The process involves:
    1.  Finding all candidate instructions for vectorization.
    2.  Building a "tree" or graph of related instructions.
    3.  Evaluating the cost of vectorizing the tree versus executing it sequentially.
    4.  If profitable, transforming the instructions into vector form.

### The Cost Model in SLP

-   The decision to vectorize is driven by a **cost model**.
-   The SLP pass calculates the cost of the original scalar instructions and compares it to the estimated cost of the vectorized instructions.
-   Vectorization only proceeds if the cost model predicts a performance benefit (i.e., the vectorized code is "cheaper").
-   The cost is determined using the `TargetTransformInfo` (TTI) interface, which queries the specific backend (e.g., AArch64, RISC-V) for the cost of various instructions.

#### Cost Differences: RISC-V vs. ARM NEON

-   An example was shown where vectorization was profitable for ARM NEON but not for RISC-V.
-   This difference was due to the addressing modes available. NEON has more flexible addressing modes (e.g., register + immediate offset) that allow `load`/`store` instructions to be generated at zero additional cost (`TCC_FREE`).
-   RISC-V, with its more limited addressing modes, required additional instructions to calculate memory addresses, making the vectorized version more expensive.

## Part 2: Supporting Custom Vector Instructions

To improve vectorization performance, a backend can be extended with custom instructions that better match the patterns found by the vectorizer.

### Describing Instructions in TableGen

-   New instructions are defined in `.td` (TableGen) files located in the target's backend directory (e.g., `/lib/Target/RISCV/`).
-   For a custom vector load/store with an immediate offset, the following must be specified:
    1.  **Operands:** Define the inputs (`ins`) and outputs (`outs`) of the instruction, such as the vector register class for the data, the base address register, and the immediate offset.
    2.  **Assembly String:** How the instruction should be printed in assembly code.
    3.  **Selection Pattern:** A pattern that matches a subgraph in the SelectionDAG (the IR representation during instruction selection). This tells the compiler how to replace a sequence of generic operations (like an `add` followed by a `load`) with the new custom instruction.

### SelectionDAG Matching

-   Without a custom instruction, calculating an address and performing a load might involve multiple nodes in the SelectionDAG (e.g., a green node for the base address, a blue node for the offset, and an `add` node to combine them).
-   By defining a pattern, the instruction selector can match this entire subgraph and replace it with a single machine instruction that takes the base and offset as direct operands, improving code efficiency.

## Part 3: LLVM Loop Vectorizer

*This section seems to be from a separate, related presentation.*

### Overview

-   The Loop Vectorizer widens instructions in loops to operate on multiple consecutive iterations simultaneously.
-   The number of iterations processed in parallel is the **Vectorization Factor (VF)**.
-   Like the SLP vectorizer, it uses the **Target Transform Info (TTI)** to guide its decisions.

### LCSSA (Loop-Closed SSA Form)

-   For a loop to be vectorized, it must be in **Loop-Closed SSA Form**.
-   This is a stronger form of SSA which guarantees that any value defined inside a loop is used only inside that loop.
-   If a value needs to be used outside, a PHI node is placed in the loop's exit block to "close" the loop. This simplifies many loop optimizations, including vectorization.

### Structure of the Loop Vectorizer Pass

The vectorization process follows four main steps:

1.  **Legality Check:** Determines if vectorizing the loop is safe.
    -   **Memory Checks:** Ensures that vectorizing won't reorder memory accesses in a way that violates dependencies (e.g., checking for aliasing).
    -   **Scalar Checks:** Ensures the loop structure is simple enough (e.g., has a single induction variable) and that all types are supported.
2.  **Build Vectorization Recipes:** Creates one or more "VPlans" (Vectorization Plans) for different possible Vectorization Factors (VFs).
3.  **Find the Best Plan:** Uses the **cost model** to evaluate each VPlan and select the one predicted to give the most speedup. The cost model queries the TTI for the cost of scalar vs. vector instructions for the target architecture.
4.  **Execute the Plan:** Transforms the original scalar loop into a new vectorized loop based on the chosen VPlan. This involves creating a new loop body with vector instructions and often includes generating pre-loop and post-loop code to handle iterations not divisible by the VF.

### Handling PHI Instructions

-   PHI nodes within a loop present a challenge for vectorization.
-   A common strategy is to convert the PHI into a `select` instruction.
-   For a PHI with `N` incoming values, `N-1` `select` instructions are needed.
-   On vector targets like RISC-V with the V extension, these selects can be implemented efficiently with a masked merge instruction (e.g., `VMERGE_VVM`), where the cost is proportional to the chosen vector length multiplier (LMUL).

## References

-   Writing an LLVM Backend (2014 LLVM Developers’ Meeting)
-   `llvm/lib/Transforms/Vectorize/SLPVectorizer.cpp`
-   Exploiting Superword Level Parallelism with Multimedia Instruction Sets
-   Loop-aware SLP in GCC
