# Data Flow vs. Data Dependence

This note distinguishes between Data Flow Analysis and Data Dependence Analysis, key concepts in compiler optimization.

## Data Flow Analysis

Data Flow Analysis is a value-based approach that utilizes lattice theory to analyze the effect of each basic block and compose these effects to derive information at basic block boundaries. It helps in understanding program properties to generate efficient code.

### Key Concepts:

*   **Local Analysis vs. Data Flow Analysis**: While local analysis examines the effect of individual instructions, data flow analysis considers the impact of entire basic blocks.
*   **Undecidability**: Rice's theorem states that non-trivial semantic properties of programs are undecidable. Data flow analysis provides useful approximations for most realistic programs.
*   **Applications**:
    *   Identifying dead code for code size reduction.
    *   Moving loop-invariant expressions outside loops.
    *   Pre-computing variable values at compile time if they don't depend on program input.
*   **Example: Liveness Analysis**: A variable `v` is live at point `p` if its value is used along some path in the flow graph starting at `p`. Otherwise, it's dead.
    *   **Data-flow Equation of Liveness**:
        *   `LIVEOUT(n)`: Variables live at the end of block `n`.
        *   `UEVAR(m)`: Variables used in block `m` before any redefinition.
        *   `VARKILL(m)`: Variables defined in block `m`.
        *   A variable is live on exit from a node if it is live on entry to any successor.
        *   A variable is live on entry to a node if it's used by the node, or if it's live on exit and not redefined by the node.
*   **Lattice Theory**: Data flow analysis often uses lattices, which are partially ordered sets where least upper bounds (lub) and greatest lower bounds (glb) exist for all elements. Complete lattices are those where lub and glb exist for all subsets.
*   **Fixed-point Theorem**: For lattices with finite height and monotone constraint functions, the fixed-point theorem guarantees solutions to equation systems, and the most precise solutions can be found by iterative computation until a fixed point is reached.

## Data Dependence Analysis

Data Dependence Analysis is a location-based approach, in contrast to the value-based nature of data flow analysis. It focuses on how operations are constrained by the flow of data. It is crucial for understanding how instructions relate to each other, especially for parallelization and program slicing.

### Key Concepts:

*   **Location-based**: Unlike data flow analysis, which is value-based, data dependence analysis is concerned with memory locations and how data is accessed and modified.
*   **Applications**:
    *   **Parallelization**: Identifying independent operations that can be executed concurrently.
    *   **Slicing**: Used in conjunction with control dependence to extract relevant parts of a program.
*   **Key Theorems**:
    *   For data flow analysis, the **Knaster–Tarski fixed-point theorem** is fundamental.
    *   For data dependence analysis, **Fourier-Motzkin elimination** is a key theorem, especially for solving systems of linear inequalities that arise in dependence testing.
    *   For affine partitioning, **Farkas’ lemma** is significant. Paul Feautrier's work on Parametric Integer Programming is a cornerstone in this area.

## Distinction Summary

| Feature                 | Data Flow Analysis     | Data Dependence Analysis |
| :---------------------- | :--------------------- | :----------------------- |
| **Primary Focus**       | Value-based properties | Location-based ordering  |
| **Theoretical Basis**   | Lattice Theory, Fixed-point Theorem | Fourier-Motzkin elimination, Farkas’ lemma |
| **Typical Applications**| Dead code elimination, loop-invariant code motion, constant propagation | Parallelization, program slicing, memory disambiguation |

This distinction is crucial for understanding different compiler optimization techniques and their underlying mathematical foundations. value (e.g., `x = 5`), it can replace uses of `x` with `5`.