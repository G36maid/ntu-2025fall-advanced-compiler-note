# Local optimizations, DCE, and LVN framework

Prof. 廖世偉, liao@csie.ntu.edu.tw
Read Dragon Book: 8.4-8.5, 6.2.4

## Learning-by-doing in today’s class:
https://www.youtube.com/watch?v=YhmCG8S5umc

## Reminder: Simple quiz at the end of class today; No need to turn in your answer.
Each time I’ll record the answer for you in a week.
BTW, each week’s lecture is also recorded. Just watch it at your pace.

Though you don’t need to come to class per se, your attendance will benefit yourself: 儀式感很重要:

## Local vs. Data-flow Analysis
*   **Local Analysis**:
    *   Analyze effect of each instruction.
    *   Compose effects of instructions to derive information from beginning of basic block to each instruction.
*   **Data-flow Analysis**:
    *   Analyze effect of each basic block.
    *   Compose effects of basic blocks to derive information at basic block boundaries.

## Intra-procedural and Interprocedural analysis
*   **Intra-procedural analysis**:
    *   **Local analysis**: Within a single basic block.
    *   Doesn’t deal with control flow. Just “linear scan”.
*   **Global analysis**: Works on an entire function.
*   **Interprocedural analysis**: Works on an entire program.

From now on, we use “local and global analysis” instead of “intra-procedural analysis” to avoid confusion with "interprocedural".

## Dead Code Elimination (DCE) in BRIL
"Dead code" = Can’t have any possible effect on the output of a program.
Dead code elimination (DCE) makes the program run faster!

**Example:**
```example.bril#L1-5
main {
  a: int = const 4;
  c: int = const 1; // Dead code
  print a;
}
```
In this example, `c: int = const 1;` is dead code.

**Another example:**
```example2.bril#L1-7
main {
  a: int = const 4;
  b: int = const 2;
  c: int = const 1; // Dead code
  d: int = add a b;
  print d;
}
```
Here, `c: int = const 1;` is still dead code. Dynamic instruction count goes down from 5 to 4 after DCE.

## Write pseudo code on how to do DCE:
**First algorithm (sub-optimal):**
Iterate through all the instructions in a given basic block:
Eliminate an instruction whose destination:
1.  is never used as an instruction’s argument, AND
2.  is NOT live at the exit of the given basic block.
Such an instruction is called Dead Code.

**Note:** Liveness is a global property. This approach relates to Data Flow analysis.

## Algorithm: Use 2 loops below.
```dce_algo.py#L1-8
used = {}
for inst in a func:
  used += instr.args
for inst in a func:  // Here we only delete those “pure” instructions with unused dest
  if instr.dest and \
      instr.dest is NOT in used :
     delete inst
// Impure instructions such as print instruction have side effects.
// In BRIL, impure instructions won’t have “.dest”, so print instruction is never deleted above.
```

## Is this algorithm enough? (Iterative approach)
Consider this case:
```example3.bril#L1-8
main {
  a: int = const 4;
  b: int = const 2;
  c: int = const 1;
  d: int = add a b;
  e: int = add c d; // This instruction's result 'e' is not used.
  print d;
}
```
The 2-loop algorithm alone will only delete `e: int = add c d;`. However, `c: int = const 1;` also becomes dead code after `e` is removed. This shows the 2-loop algorithm is insufficient on its own. The algorithm should iterate until it converges.

## Key Concept: Iterative algorithm in compilers
```iterative_dce.py#L1-3
while program changed:
  run DCE (the 2-loop algorithm) above
```

## Is this algorithm enough? (Control Flow Challenges)
Consider this case with write-write dependence:
```example4.bril#L1-4
main {
  a: int = const 100;
  a: int = const 42:
  print a;
}
```
Here, the first assignment to `a` is dead.

Consider DCE in the presence of control flow:
```example5.bril#L1-9
main {
  a: int = const 100;
  br input .l .k;
.l:
  a: int = const 42:
.k:
  print a;
}
```
DCE in the presence of control flow is challenging! For now, we resort to local DCE.

## Local DCE:
This is a Python-like pseudo code. You should implement the check and elimination yourself.
```local_dce_algo.py#L1-9
last_def_without_any_use_yet = {}  // mapping from variable -> instr
for instr in block:
  // check for uses
  last_def_without_any_use_yet -= instr.args
  // check for defs
  if instr.dest in last_def_without_any_use_yet:
    delete last_def_without_any_use_yet[inst.dest]  // Dead code elimination here
  last_def_without_any_use_yet[instr.dest] = instr
```

## To understand “brili” vs. “brili -p”, see https://capra.cs.cornell.edu/bril/tools/interp.html

## simple.bril example:
```simple.bril#L1-5
main {
  a: int = const 4;
  b: int = const 2;
  c: int = const 1;
  d: int = add a b;
  print d;
}
```
For these 2 passes above: Iterate them until convergence. Once converged, `wc -l` returns 6 instead of 7 (after deleting `c: int = const 1`).

**llvm-as, llvm-dis**

## Run the bril interpreter (brili): We care about the DIC:
```bash
$ bril2json < simple.bril | python3 tdce.py | brili -p
```
Output:
```
6
total_dyn_inst: 4
```
Total dynamic instruction count (DIC) = 4 instead of 5. This indicates success in DCE.

## More real case:
Reduce 148 instructions to 144 instructions through DCE.

## Recap: Dead Code Elimination Algorithm
*   **Globally unused instructions:**
    *   Derive an algorithm for deleting them.
    *   Iterating to convergence.
    *   Then implement it.
*   **Locally killed instructions:**
    *   **Caveat:** Be aware of the limits of local reasoning: Why can't we do this globally?
    *   Derive an algorithm for removing them.
    *   Then implement that too.

## Local Value Numbering (LVN) framework:
A unifying framework for:
*   DCE
*   Copy propagation
*   Constant propagation
*   Common Subexpression Elimination (CSE)

When free, please watch the video on Local Value Numbering:
https://www.cs.cornell.edu/courses/cs6120/2020fa/lesson/3/

If you’re wonderful, feel free to do the tasks (Cornell website says: “due September 21”, but you don’t need to turn it in. It’s optional.)
