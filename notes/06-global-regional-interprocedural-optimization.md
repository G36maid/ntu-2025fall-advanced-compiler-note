# Global, Regional, and Interprocedural Optimization

This note covers various compiler optimization techniques that operate at different scopes: global, regional, and interprocedural. These optimizations aim to improve program performance by leveraging insights from program behavior and underlying hardware architecture.

## Global Optimization

Global optimization focuses on improving the entire program's performance. One significant example is Global Code Placement, which leverages the architectural characteristics of modern processors.

### Opportunities from Asymmetric Branch Costs

Many processors exhibit asymmetric branch costs, meaning a "fall-through" branch (execution continues to the next sequential instruction) is less costly than a "taken" branch (execution jumps to a different address). This is primarily due to pipelining and control hazards.

*   **Pipelining**: Modern CPUs use pipelines to execute instructions concurrently.
*   **Control Hazard & Pipeline Stall**: When a branch instruction is encountered, the CPU might speculatively fetch instructions based on a prediction (e.g., always untaken). If the prediction is wrong (a "taken" branch when predicted untaken), the pipeline must be flushed, causing a stall or "bubble," which wastes cycles.

### Global Code Placement Intuition

The goal of global code placement is to arrange basic blocks in memory such that frequently executed branches become fall-through branches. By knowing the execution frequency of edges in the Control Flow Graph (CFG), the compiler can position high-frequency blocks sequentially, minimizing the cost of taken branches.

### Three Steps in Global Code Placement

1.  **Analysis**: Estimate the relative execution frequency of each edge on the CFG. This often involves runtime profiling.
2.  **Pre-computing**: Construct a set of CFG paths that include the most frequently executed edges (hot paths) and establish a partial order on the basic blocks.
3.  **Transformation**: Place all basic blocks into a fixed linear order in the executable code.

### Obtaining Profile Data

Accurate profile data is crucial for effective global code placement. Methods for obtaining this data include:

*   **Instrumented executables**: The compiler generates code to count specific events (e.g., procedure entries/exits, taken branches).
*   **Timer interrupts**: Program execution is interrupted at regular intervals, and a histogram of program counter locations is constructed.
*   **Performance counters**: Hardware counters record events like total cycles, cache misses, or taken branches.

The accuracy of this profiling directly impacts the quality of the optimization results.

### Greedy Algorithm to Find Hot Paths

A "hot path" is a CFG path comprising the most frequently executed edges. The greedy algorithm aims to identify these paths and assign them priorities for code layout.

*   **Initialization**: Each basic block starts as a "degenerate chain" with a high priority.
*   **Merging**: Edges are processed from highest to lowest frequency. If an edge `<x, y>` connects the tail of chain `C_x` and the head of chain `C_y`, and `C_x` and `C_y` are distinct, they are merged into a new chain `C_xy`.
*   **Priorities**: Chains are assigned priorities to guide the final code layout, approximating the maximal number of executed forward branches.

### Performing Code Layout

The layout algorithm uses two main heuristics:

1.  Place the blocks of a chain in order to implement fall-through branches.
2.  Choose among alternative placements using the recorded priority numbers for the chains.

Different choices when breaking ties among equal-priority edges can lead to different sets of chains and code layouts, though the algorithm still produces good results.

### Data-flow Analysis vs. Runtime Profiling

*   **Data-flow analysis**: Conservative, accounting for all possibilities, which can lead to more robust but potentially less aggressive optimizations.
*   **Runtime profiling**: Aggressive, assuming future runs will share characteristics with the profiling run, potentially leading to highly optimized but less general results.

## Regional Optimizations

Regional optimizations focus on specific code subsets larger than a basic block but smaller than an entire procedure, such as loops. This limited scope can yield sharper analysis and better transformation results, especially in heavily executed regions.

### Loop Unrolling

Loop unrolling duplicates the loop body multiple times within a single iteration, reducing loop control overhead and potentially exposing more instruction-level parallelism.

*   **Benefits**:
    *   **Improved Instruction Scheduling**: Increases the number of independent operations in the loop body, allowing the CPU to execute more instructions in parallel (especially useful for superscalar architectures).
    *   **Amortized Loop Control Overhead**: Reduces the frequency of loop termination condition checks and branch instructions, saving cycles. This also reduces branch prediction misses.
*   **Drawbacks**:
    *   **Increased Program Size**: Duplicating code can lead to a larger executable, potentially causing instruction cache misses.
    *   **Increased Register Pressure**: The unrolled loop body may require more registers to hold temporary values, potentially leading to spills (saving registers to memory) and additional memory accesses.

### Code Motion (Lazy Code Motion - LCM)

Code motion moves a sequence of code to a point where it runs less frequently, thereby reducing the total operation count. Lazy Code Motion (LCM) is a prominent example that specifically targets partially redundant expressions.

*   **Redundancy**:
    *   **Partially Redundant**: An expression `e` is partially redundant at point `p` if it occurs on some, but not all, paths reaching `p`.
    *   **Redundant**: An expression `e` is redundant at `p` if it has already been evaluated on every path leading to `p`.
*   **LCM Goal**: LCM finds partially redundant operations, inserts code to make them fully redundant, and then deletes the newly redundant expressions. It moves loop-invariant code out of loops, speeding up execution.
*   **Assumptions**: Expressions are often represented by names that are not redefined.
*   **LCM Steps**:
    1.  **Solve Data-Flow Problems**:
        *   **Available Expressions**: An expression `e` is available on exit from block `b` if it's evaluated on every path from `n0` to `b`, and its operands aren't redefined.
        *   **Anticipable Expressions**: An expression `e` is anticipable on exit from block `n` if every successor block `m` either evaluates `e` and uses it, or has `e`'s first evaluation equivalent to evaluating `e` at the end of block `n`.
    2.  **Earliest Placement**: For each CFG edge `(i, j)`, compute the set of expressions that can be moved to `(i, j)` but not to any earlier edge.
    3.  **Later Placement**: Determines how late an earliest placement can be deferred while achieving the same effect.
    4.  **Compute INSERT and DELETE Sets**: Based on earliest and later placement, determine which expressions to insert and which to delete.
    5.  **Rewrite the Code**: Insert new computations at appropriate points (end of predecessor block, entry of successor block, or a new split block for edges) and delete redundant computations.

## Interprocedural Optimization

Interprocedural optimization (IPO) considers the entire program across procedure boundaries, addressing the limitations of local and global optimizations that only look within a single procedure.

### Impacts of Procedure Division

*   **Positive**: Limits the code scope a compiler considers at one time, simplifying analysis.
*   **Negative**:
    *   **Limited Visibility**: The compiler has limited understanding of what happens inside a called procedure without examining its body.
    *   **Procedure Linkage Overhead**: Each procedure call incurs overhead for pre-call, post-return, prolog, and epilog code.

### Inline Substitution

Inline substitution replaces a procedure call site with a copy of the callee's body. This eliminates call overhead and exposes more code to global and regional optimizations.

*   **Decision Criteria**:
    *   **Callee Size**: Small callees might be inlined if their body is smaller than the linkage code.
    *   **Caller Size**: Compilers might limit the overall size of any procedure to prevent code bloat.
    *   **Dynamic Call Count**: Inlining frequently called sites provides greater benefit.
    *   **Constant-valued Actual Parameters**: Inlining allows constants to be folded into the callee, enabling further optimizations.
    *   **Static Call Count**: Procedures called from only one site can be inlined without code space growth for the callee.
    *   **Parameter Count**: Fewer parameters might make inlining more attractive.
    *   **Calls in the Procedure**: Procedures with no internal calls are good candidates.
    *   **Loop Nesting Depth**: Call sites within loops execute more frequently and benefit more from inlining.
    *   **Fraction of Execution Time**: Inlining hot call sites is prioritized.
*   **Heuristics**: Compilers use various heuristics, often combining these metrics, to decide which call sites to inline.

### Procedure Placement

Procedure placement rearranges the physical locations of procedures in memory based on the call graph (nodes are functions, edges are call sites) to:

*   **Reduce Virtual-Memory Working-Set Sizes**: Placing frequently called procedures and their data close together limits the number of virtual memory pages that need to be in physical memory, reducing page faults.
*   **Limit Instruction Cache Conflicts**: Arranging frequently interacting procedures contiguously in memory minimizes instruction cache misses, improving execution speed.

This is typically a greedy algorithm, aiming for good rather than optimal placement due to the complexity of the problem.

### Interprocedural Optimization: Compiler Organization

Different compiler architectures can facilitate IPO:

*   **Enlarging Compilation Units**: Combining multiple source files into a single unit for compilation allows the compiler to see more code. However, it cannot introduce dependence between compilation units.
*   **Link-time Optimization (LTO)**: Shifts IPO to the linker stage, where the compiler has access to all statically linked code, enabling whole-program analysis and optimization.

## References

*   Engineering A Compiler (3rd. ed.): Chapters 8.5 (Regional Optimization), 8.6 (Global Optimization), 8.7 (Interprocedural Optimization), 10.3 (Code Motion)
*   Compilers: Principles, Techniques, and Tools (2nd. ed.)
*   Computer Organization and Design RISC-V Edition (2nd. ed.)
*   Computer Architecture : A Quantitative Approach (6nd. ed.)
*   CS 153: Loop Optimization