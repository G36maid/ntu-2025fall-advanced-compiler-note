# Midterm exam of Advanced Compiler

## Question 1: Dominance Frontier Computation
```
                   ┌─────┐
                   │ B0  │  (entry)
                   │x=.. │
                   └──┬──┘
                      │
             ┌────────┴───────┐
             │                │
          ┌──▼──┐          ┌──▼──┐
          │ B1  │          │ B2  │
          │x=.. │          │y=.. │
          └──┬──┘          └──┬──┘
             │                │
             │     ┌─────┐    │
             └────►│ B3  │◄───┘
                   │z=x+y│
                   └──┬──┘
                      │
                   ┌──▼──┐
                   │ B4  │  (exit)
                   └─────┘
```

The dominance frontier `DF(B1)` is:
* A) `{B3, B4}`
* B) `{B2, B3}`
* C) `{B4}`
* D) `{B3}`
* E) `{B1, B3}`
* F) `∅` (empty set)
* G) `{B0, B3}`
* H) `{B2, B3, B4}`

Answer: D

Explanation:
- `DF(B1) = {B3}` is correct because `B1` dominates predecessor `B1` of `B3`, but does NOT strictly dominate `B3` itself (path `B0→B2→B3` exists).
- A is wrong: `B4` is dominated by `B3`; since `B1` doesn't dominate `B3`, it cannot have `B4` in its `DF`.
- B is wrong: `B2` is not in `DF(B1)` because `B1` doesn't dominate any predecessor of `B2`.
- C is wrong: `B4`'s only predecessor is `B3`; since `B1` does not dominate `B3`, `B4 ∉ DF(B1)`.
- E is wrong: In this CFG, `B1 ∉ DF(B1)` because `B1` does not dominate any predecessor of `B1`. Note: A node can belong to its own `DF` in cyclic CFGs with loop headers (e.g., loop backedges where `X ∈ DF(X)`).
- F is wrong: `B3` clearly belongs to `DF(B1)`.
- G is wrong: `B0` dominates `B1`, not the other way around; `B0` cannot be in `DF(B1)`.
- H is wrong: `B2` and `B4` don't meet the `DF` criteria.

---

## Question 2: Iterated Dominance Frontier and φ-Placement
```
           ┌─────┐
           │ B0  │
           └──┬──┘
              │
         ┌────┴───┐
         │        │
      ┌──▼──┐  ┌──▼──┐
      │ B1  │  │ B2  │
      │x=1  │  │x=2  │
      └──┬──┘  └──┬──┘
         │        │
         └────┬───┘
              │
           ┌──▼──┐
           │ B3  │◄───┐
           │y=x │     │
           └──┬──┘    │
              │       │
         ┌────┴────┐  │
         │         │  │
      ┌──▼──┐  ┌──▼───┴┐
      │ B4  │  │  B5   │
      │x=3  │  │if(...)│
      └──┬──┘  └───────┘
         │
         │    ┌─────┐
         └───►│ B6  │
              │w=x  │
              └─────┘
```

How many φ-functions for variable `x` are needed in minimal SSA form?
* A) 2 φ-functions at `B3` and `B6`
* B) 3 φ-functions at `B3`, `B6`, and `B5`
* C) 1 φ-function at `B3`
* D) 2 φ-functions at `B3` (placed twice due to iteration)
* E) 3 φ-functions at `B3`, `B5`, and `B6`
* F) 4 φ-functions at `B3`, `B5`, `B6`, and one more at `B3` (iteration effect)
* G) 1 φ-function at `B6` only
* H) 0 φ-functions (reaching definitions analysis suffices)

Answer: C

Only one φ at `B3` is required for `x` in minimal SSA.
No φ is needed at `B6`.

Definitions and CFG facts
* `x` is (re)defined at `B1: x=1`, `B2: x=2`, `B4: x=3`.
* Structure: `B0 → (B1 | B2) → B3`, then from `B3` to `B4` or `B5`; `B5 → B3` (backedge); `B4 → B6`.
* Uses: `B3: y = x`, `B6: w = x`.

Dominators
* `B3` is reached from both `B1` and `B2` ⇒ `dom(B3) = {B0, B3}`.
* `B4` only via `B3` ⇒ `dom(B4) = {B0, B3, B4}`.
* `B5` only via `B3` ⇒ `dom(B5) = {B0, B3, B5}`.
* `B6` only via `B4` (and hence via `B3`) ⇒ `dom(B6) = {B0, B3, B4, B6}`.

Dominance frontiers of original def blocks
Definition: `Y ∈ DF(X)` iff `X` dominates some predecessor of `Y` and `X` does not strictly dominate `Y`.
* `DF(B1) = {B3}`: `B1` dominates predecessor `B1` of `B3`, but `B1` does not strictly dominate `B3` (there is the path `B0→B2→B3`).
* `DF(B2) = {B3}`: symmetric to `B1`.
* `DF(B4) = ∅`: `B6` has a single predecessor `B4` and `B4` strictly dominates `B6`, so the "not strictly dominate Y" clause fails. Hence `B6 ∉ DF(B4)`.

Initial φ-placement from defs `{B1,B2,B4}` is therefore φ at `B3` only.

Note on φ operands: The φ-function at `B3` has one argument per predecessor. Since `B3` has three predecessors (`B1`, `B2`, and `B5`), the φ will be `x₃ = φ(x₁, x₂, x₃)` where the third argument comes from `B5`. Even though `B5` doesn't redefine `x`, it passes through the current value of `x₃` from the loop.

Why other choices are wrong
* A (`B3` and `B6`): Wrong because `B6` is not in the `DF` of any def (`B4` strictly dominates `B6`), and `B6` is not a join (single predecessor).
* B/E (including `B5`): `B5` is not a join for `x` and does not require a φ.
* D/F (duplicate φ at `B3`): SSA never places two φs for the same variable at the same block.
* G (`B6` only) and H (no φs): Miss the essential join at `B3` where values from `B1` and `B2` (and the loop-back) meet.

---

## Question 3: Post-Dominator Tree and Control Dependence
```
           ┌─────┐
           │ B0  │
           └──┬──┘
              │
           ┌──▼──┐
           │ B1  │
           │if(p)│
           └──┬──┘
              │
         ┌────┴───┐
         │        │
      ┌──▼──┐  ┌──▼──┐
      │ B2  │  │ B3  │
      │ ... │  │exit │
      └──┬──┘  └─────┘
         │
      ┌──▼──┐
      │ B4  │
      │exit │
      └─────┘
```

Which node is `B2` control-dependent on?
* A) `B0`
* B) `B1`
* C) `B4`
* D) `B2` is not control-dependent on any node
* E) `B0` and `B1` (both)
* F) `B1` and `B2` (`B2` depends on itself)
* G) All nodes in the CFG
* H) `B3` (the alternate exit path)

Answer: B

Explanation:

Formal definition: Node `Y` is control-dependent on node `X` iff:
1. There exists a path `X⇒…⇒Y` on which `Y` post-dominates every node after `X` on that path, and
2. `Y` does not post-dominate `X`

Applied here: `B2` is control-dependent on `B1` because there exists path `B1→B2` where `B2` post-dominates all nodes after `B1` on that path (just `B2` itself), AND `B2` does NOT post-dominate `B1` (the path `B1→B3` bypasses `B2`).
- A is wrong: `B0` is not a branch point that controls whether `B2` executes; all paths from `B0` go through `B1`, and `B1`'s decision matters
- C is wrong: `B4` doesn't control `B2`; rather `B2` leads to `B4`
- D is wrong: `B2` clearly depends on `B1`'s branch decision
- E is wrong: `B0` doesn't introduce control dependence (it has no control flow decision affecting `B2`)
- F is wrong: nodes cannot be control-dependent on themselves by standard definition
- G is wrong: control dependence is a specific relationship, not transitive across all nodes
- H is wrong: `B3` is an alternate path, but doesn't control `B2`; `B1` does the controlling

---

## Question 4: Loop Invariant Code Motion with Exceptions
```
           ┌─────┐
           │ B0  │
           └──┬──┘
              │
       ┌──────▼──────┐
       │     B1      │◄────┐
       │  if(i<n)    │     │
       └──┬─────┬────┘     │
          │     │          │
       exit  ┌──▼───┐      │
             │ B2   │      │
             │ t=a/b│      │ (may throw div-by-zero)
             │c[i]=t│      │
             │  i++ │      │
             └──┬───┘      │
                └──────────┘
```
 
Expression `t = a/b` is loop-invariant (`a`, `b` not modified in loop). Under which condition is hoisting `t = a/b` before `B1` ILLEGAL?
* A) When `b` might be zero
* B) When the loop might execute zero iterations (`n ≤ 0`)
* C) When division has side effects (sets CPU flags)
* D) When `a` and `b` are floating-point with NaN handling
* E) B only (must not introduce new exceptions)
* F) A and B together
* G) B, C, and D together
* H) Never illegal - hoisting loop-invariant code is always safe

Answer: E

Explanation:

Loop invariant code motion legality conditions (assuming a language with well-defined arithmetic exceptions, such as Java):
1. No new exceptions: Cannot introduce exceptions on paths where they didn't exist
2. Dominance: Definition must dominate all uses (satisfied here)
3. No conflicting definitions: No redefinitions in loop (satisfied here)

Critical issue: Exception semantics
- Original: `t = a/b` executes only if loop body enters (`if i < n`)
- After hoisting: `t = a/b` always executes before loop test
- Problem: If `n ≤ 0`, loop never executes, so original code never divides
- After hoisting: division executes even when `n ≤ 0`, potentially throwing exception that didn't exist before

Analysis of options:
- A is wrong alone: If loop always executes and `b` is zero, BOTH versions throw; no NEW exception introduced
- Only B holds true; therefore the correct choice is E ("B only"): Zero-trip loop problem - introduces exception where none existed
- C is wrong: CPU flags are typically not considered "observable side effects" for optimization purposes (unless volatile/asm context)
- D is wrong: NaN handling doesn't introduce NEW exceptions; NaN propagates in both versions
- F is redundant: A doesn't add to the illegality; B alone suffices
- G: C and D don't contribute to illegality
- H: Clearly wrong - exception semantics matter

Conclusion: The zero-trip problem makes hoisting illegal. If we can prove `n > 0`, hoisting becomes legal even if `b` might be zero (since the exception would occur in both versions).

---

## Question 5: Dominance Frontier and φ-Placement
```
           ┌─────┐
           │ B0  │  (entry)
           │x=1  │
           └──┬──┘
              │
           ┌──▼──┐
           │ B1  │
           │if(p)│
           └──┬──┘
              │
         ┌────┴───┐
         │        │
      ┌──▼──┐  ┌──▼──┐
      │ B2  │  │ B3  │
      │x=2  │  │y=.. │
      └──┬──┘  └──┬──┘
         │        │
         └────┬───┘
              │
           ┌──▼──┐
           │ B4  │
           │z=x  │
           └─────┘
```

Where must φ-functions for variable `x` be placed?
* A) `B1` only (decision point)
* B) `B1` and `B4` (both join points)
* C) `B2` and `B3` (definition sites)
* D) No φ needed (`B0` dominates all)
* E) `B0` and `B4` (entry and exit)
* F) All blocks need φ-functions
* G) `B4` only (join point after definitions)
* H) `B3` and `B4` (`B3` needs φ even without defining `x`)

Answer: G

Validation:

Step 1: Identify definitions
- `x` is defined in: `{B0, B2}`

Step 2: Compute dominance
- Nodes dominated by `B0`: `{B0, B1, B2, B3, B4}` (all nodes)
- Nodes dominated by `B1`: `{B1, B2, B3, B4}`
- Nodes dominated by `B2`: `{B2}` (only itself)
- Nodes dominated by `B3`: `{B3}` (only itself)
- Dominators of `B4`: `{B0, B1, B4}` (nodes that dominate `B4`)

Step 3: Compute `DF(B0)`
Definition: `DF(B0) = {Y | B0 dominates some predecessor of Y, but B0 does not strictly dominate Y}`
- `B0` dominates all nodes and strictly dominates all except itself
- Therefore `DF(B0) = ∅`

Step 4: Compute `DF(B2)`
Definition: `DF(B2) = {Y | B2 dominates some predecessor of Y, but B2 does not strictly dominate Y}`
- Check `B4`: predecessors = `{B2, B3}`
- `B2` dominates predecessor `B2` (itself) ✓
- `B2` does NOT strictly dominate `B4` (because `B3` can reach `B4` without going through `B2`) ✓
- Therefore `B4 ∈ DF(B2)`

Step 5: Compute `DF⁺`
- `DF⁺({B0, B2}) = DF(B0) ∪ DF(B2) = ∅ ∪ {B4} = {B4}`

Conclusion: Place φ-function at `B4` only.

SSA form: `x₄ = φ(B2: x₂, B3: x₀)`

Note: `B3` passes `x₀` (from `B0`) since `x` is not redefined in `B3`.

Why others wrong:
- A: `B1` is not a join point for `x` definitions (`x` not redefined in both branches yet)
- B: `B1` doesn't need φ (not in dominance frontier)
- C: We place φ at join points, not definition sites
- D: `B0` doesn't dominate uses in a way that eliminates need for φ at `B4`
- E: `B0` is entry, doesn't need φ; but `B4` is correct
- F: Only join points with multiple reaching definitions need φ
- H: `B3` doesn't need φ (no multiple incoming definitions of `x`)

---

## Question 6: Strength Reduction in Loops
```c
  // Original loop
  for (i = 0; i < n; i++) {
      a[i * 4] = i * 4 + 10;
      b[i * 4] = i * 8;
  }

  // After strength reduction
  for (i = 0; i < n; i++) {
      a[???] = ??? + 10;
      b[???] = ???;
  }
```

What is the CORRECT strength-reduced form with auxiliary induction variables?
* A) `t1=0; t2=0; ... a[t1]=t1+10; b[t2]=t2; ... t1+=4; t2+=8;`
* B) `t1=0; t2=0; ... a[t1]=t2+10; b[t1]=t2; ... t1+=4; t2+=8;`
* C) `t=0; ... a[t]=t+10; b[t]=t*2; ... t+=4;`
* D) `t1=0; t2=0; ... a[t1]=t1+10; b[t1]=t2; ... t1+=4; t2+=4;`
* E) `t1=0; t2=10; ... a[t1]=t2; b[t1]=t1*2; ... t1+=4; t2+=4;`
* F) Cannot apply strength reduction (multiplications are necessary)
* G) `t=0; ... a[t]=t+10; b[t]=t+t; ... t+=4;`
* H) `t1=0; t2=0; t3=10; ... a[t1]=t3; b[t1]=t2; ... t1+=4; t2+=8; t3+=4;`

Answer: H

Validation:

Step 1: Identify induction variables
- Basic IV: `i` (loop counter, increments by 1)
- Dependent IVs:
    - `i * 4` (linear relationship with `i`)
    - `i * 8` (linear relationship with `i`)
    - `i * 4 + 10` (affine relationship with `i`)

Step 2: Apply strength reduction transformations

For `a[i * 4] = i * 4 + 10`:
 - Array index: `i * 4`
    - Introduce `t1` for array indexing
    - Initial: `t1 = 0` (when `i=0`)
    - Update: `t1 += 4` (instead of computing `i*4`)
 - RHS: `i * 4 + 10`
    - Introduce `t3` for the expression
    - Initial: `t3 = 10` (when `i=0`, `0*4+10=10`)
    - Update: `t3 += 4` (incremental computation)

For `b[i * 4] = i * 8`:
- Array index: `i * 4` (already have `t1`)
- RHS: `i * 8`
    - Introduce `t2` for this expression
    - Initial: `t2 = 0` (when `i=0`)
    - Update: `t2 += 8` (instead of computing `i*8`)

Step 3: Transformed code
```c
  t1 = 0;      // for i * 4 (array indexing)
  t2 = 0;      // for i * 8
  t3 = 10;     // for i * 4 + 10

  for (i = 0; i < n; i++) {
      a[t1] = t3;
      b[t1] = t2;

      t1 += 4;
      t2 += 8;
      t3 += 4;
  }
```

Verification (`i=0`):
- `t1=0`: `a[0] = 10` ✓ (`0*4+10=10`)
- `t1=0`: `b[0] = 0` ✓ (`0*8=0`)

Verification (`i=1`):
- `t1=4`: `a[4] = 14` ✓ (`1*4+10=14`)
- `t1=4`: `b[4] = 8` ✓ (`1*8=8`)

Verification (`i=2`):
- `t1=8`: `a[8] = 18` ✓ (`2*4+10=18`)
- `t1=8`: `b[8] = 16` ✓ (`2*8=16`)

Conclusion: Answer H is correct - three auxiliary variables (`t1`, `t2`, `t3`) with proper initialization and increments.

Why others wrong:
- A: Critical indexing error: `b[t2]=t2` uses `t2` (which tracks `i*8`) as the array index when the correct index is `i*4` (tracked by `t1`). Should be `b[t1]=t2`. Note: Using `t3` to pre-compute `i*4+10` is optional; `a[t1]=t1+10` is also a valid strength-reduced form, though less optimized.
- B: `a[t1]=t2+10` is WRONG. Uses wrong variable (`t2` is for `i*8`, not `i*4+10`). Also still has addition.
- C: Only uses one auxiliary variable `t`, but needs separate tracking for `i*4`, `i*8`, and `i*4+10`. Also `t*2` defeats the purpose (still has multiplication).
- D: `t2+=4` is WRONG. `t2` tracks `i*8`, so it should increment by 8, not 4.
- E: `t1*2` defeats the purpose (still has multiplication in loop body).
- F: Strength reduction is definitely applicable here.
- G: `t+t` is addition, not multiplication, but still requires computation. Also doesn't handle the `+10` offset correctly.

Key insights:
1. Strength reduction replaces: expensive operations (multiplication) → cheap operations (addition)
2. Auxiliary variables needed: One per distinct linear expression of the loop variable
    - `i * 4` → `t1` (for array indexing)
    - `i * 8` → `t2` (for RHS of `b[]`)
    - `i * 4 + 10` → `t3` (for RHS of `a[]`)
3. Benefits:
    - Eliminates 3 multiplications per iteration
    - Replaces with 3 additions per iteration
    - On most architectures: multiplication is 3-10× slower than addition
4. Cost:
    - 3 additional live variables (increased register pressure)
    - If register pressure causes spills, might not be profitable
5. Compiler considerations:
    - Modern CPUs: integer multiply latency ~3 cycles, but 1/cycle throughput
    - Strength reduction still valuable for reducing instruction count
    - Must analyze register pressure to ensure profitability

---

## Question 7: Strength Reduction Profitability
```c
  // Loop with address calculation
  for (i = 0; i < 100; i++) {
      sum += array[i * stride];
  }

  // Architecture characteristics:
  // - Integer multiplication: 3 cycles latency, 1/cycle throughput
  // - Integer addition: 1 cycle latency, 2/cycle throughput  
  // - 16 general-purpose registers available
  // - Current register pressure: 14 registers in use
```

After applying strength reduction, the code becomes:
```c
  p = &array[0];
  for (i = 0; i < 100; i++) {
      sum += *p;
      p += stride;
  }
```

Under which condition is strength reduction UNPROFITABLE?
* A) `stride = 1` (unit stride)
* B) `stride = 1024` (large stride)
* C) The loop body is very large (50+ instructions)
* D) When the extra live variable `p` causes register spilling
* E) When `stride` is not a compile-time constant
* F) Never unprofitable - strength reduction always improves performance
* G) When the loop has only 2 iterations (`n=2`)
* H) When multiplication has 1-cycle latency

Answer: D

Explanation:

Cost-benefit analysis:

Benefits:
- Eliminates 1 multiplication per iteration (saves ~2 cycles on average)
- 100 iterations × 2 cycles = 200 cycles saved

Costs:
- Introduces new live variable `p` (+1 register pressure)
- Current: 14 registers used, adding `p` → 15 registers
- If adding `p` increases the live set beyond available registers (e.g., `>16`), spills can occur
- Spill cost: Can be several cycles per iteration (loads/stores), often outweighing the saved multiply
- Example: If spills cost 10 cycles/iteration × 100 iterations = 1000 cycles cost

When D occurs: Spill cost > arithmetic benefit → UNPROFITABLE

Why others wrong:
- A: Unit stride still benefits (pointer increment is faster than multiply)
- B: Large stride doesn't affect profitability of add vs. multiply
- C: Large loop body doesn't change the arithmetic cost comparison
- E: Non-constant `stride` can still use strength reduction (runtime value in register)
- F: False - register spilling can make it unprofitable
- G: Small iteration count reduces total benefit, but doesn't make it unprofitable per iteration
- H: With 1-cycle multiplies, arithmetic benefit is negligible; profitability depends on whether extra registers cause spills

Key insight: Strength reduction trades computation for register pressure. Profitable only when register pressure doesn't cause spilling.

---

## Question 8: Loop Invariant Code Motion vs. Strength Reduction
```c
  // Original loop
  for (i = 0; i < n; i++) {
      a[i] = b[i] * factor + offset;
  }

  // factor and offset are loop-invariant (not modified in loop)
  // i is the only loop variable
```

Which optimization applies to which expression?
* A) Strength reduction: `i`; LICM: `factor`, `offset` (but not `b[i] * factor`)
* B) Strength reduction: `i`; LICM: `factor`, `offset`, `b[i] * factor`
* C) Strength reduction: `i`, `b[i]`; LICM: `offset`
* D) Strength reduction: none; LICM: `factor`, `offset`
* E) Strength reduction: `i`, `b[i] * factor`, `offset`; LICM: none
* F) Strength reduction: `i`, `b[i]`, `b[i] * factor`; LICM: `factor + offset`
* G) Strength reduction: `i`, `b[i] * factor`; LICM: `offset`
* H) Both optimizations apply to all expressions

Answer: A

Explanation:

Strength reduction replaces expensive operations with cheaper ones based on induction variable relationships:
- Applies to `i`: Array indexing `a[i]` and `b[i]` can use pointer arithmetic
    - Replace: `base + i * sizeof(type)` → increment pointer by `sizeof(type)` each iteration

Loop Invariant Code Motion (LICM) hoists computations that don't change across iterations:
- `factor`: Loop-invariant, but already a single variable (nothing to hoist)
- `offset`: Loop-invariant, but already a single variable (nothing to hoist)
- `b[i] * factor`: NOT loop-invariant (depends on `i` through `b[i]`)

Transformed code:
```c
  // Strength reduction: pointer arithmetic
  pa = &a[0];
  pb = &b[0];

  // LICM: factor and offset already available
  for (i = 0; i < n; i++) {
      *pa = *pb * factor + offset;
      pa++;  // cheaper than a[i]
      pb++;  // cheaper than b[i]
  }
```

Why others wrong:
- B, E, F, G: `b[i] * factor` is NOT loop-invariant (depends on loop variable `i`)
- C: `b[i]` alone doesn't get strength reduction (the array indexing does)
- D: Strength reduction does apply to array indexing via `i`
- F: Cannot hoist `factor + offset` as a combined expression (used with varying `b[i]`)
- H: Not all expressions benefit from both optimizations

Key insight:
- Strength reduction: Optimizes operations involving induction variables (loop counters)
- LICM: Optimizes expressions that are constant across iterations
- Overlap: Both can optimize array indexing (LICM for base address calculation, strength reduction for offset computation)

---

## Question 9: Horner's Rule for Polynomial Evaluation
```c
  // Evaluate polynomial: 2x³ + 3x² + 4x + 5

  // Form A (naive):
  result = 2*x*x*x + 3*x*x + 4*x + 5;

  // Form B (Horner's rule):
  result = ((2*x + 3)*x + 4)*x + 5;
```

How many multiplications does each form require?
* A) Form A: 6 muls; Form B: 3 muls
* B) Form A: 4 muls; Form B: 3 muls
* C) Form A: 3 muls; Form B: 3 muls (equal)
* D) Form A: 6 muls; Form B: 4 muls
* E) Form A: 5 muls; Form B: 3 muls
* F) Form A: 7 muls; Form B: 3 muls
* G) Form A: 3 muls; Form B: 2 muls
* H) Depends on compiler optimization level

Answer: A

Explanation:

Assumption: Counting multiplications for naive evaluation without common subexpression elimination (CSE). Each power of `x` is computed independently.

Form A (naive evaluation):
- `2*x*x*x`: 3 multiplications (`2×x`, then `×x`, then `×x`)
- `3*x*x`: 2 multiplications (`3×x`, then `×x`)
- `4*x`: 1 multiplication
- Total: 6 multiplications

Form B (Horner's rule):
- `2*x`: 1 multiplication
- `(2*x + 3)*x`: 1 multiplication
- `((2*x+3)*x + 4)*x`: 1 multiplication
- Total: 3 multiplications

Key insight: Horner's rule reduces multiplications from `O(n²)` to `O(n)` for degree-`n` polynomial by reusing intermediate results through nested evaluation.

Why others wrong:
- B-G: Incorrect counting
- H: Multiplication count is determined by the algorithm structure, not optimization level

Benefit: For high-degree polynomials, savings are substantial (degree 10: 55 muls → 10 muls).

---

## Question 10: SSA-Based Constant Folding and Dead Code Elimination
```c
  // Original code in SSA form:
  B0:
    x₁ = 5
    y₁ = 10
    if (x₁ < 3) goto B1 else goto B2

  B1:
    z₁ = x₁ + y₁
    w₁ = z₁ * 2
    goto B3

  B2:
    z₂ = x₁ - y₁
    w₂ = z₂ * 3
    goto B3

  B3:
    z₃ = φ(z₁, z₂)
    w₃ = φ(w₁, w₂)
    result = w₃ + z₃
```

After constant folding and dead code elimination, what code remains?
* A) `B0: if(5<3) ...; B1: ...; B2: z₂=-5; w₂=-15; B3: result=-20`
* B) `B0: x₁=5; y₁=10; B2: z₂=-5; w₂=-15; B3: result=-20`
* C) `B0: x₁=5; y₁=10; B3: result=φ(-20, -20)`
* D) All blocks remain (cannot eliminate φ-functions)
* E) `B0: goto B2; B2: result=-20`
* F) `B0: result=-20`
* G) Only `B3` remains with result computation
* H) `B2: z₂=-5; w₂=-15; B3: result=-20`

Answer: F

Explanation:

Step 1: Constant folding in `B0`
- `x₁ = 5`, `y₁ = 10` are constants
- `if (5 < 3)` → `FALSE` (constant condition)
- `B1` is unreachable, only `B2` executes

Step 2: Propagate to `B2`
- `z₂ = 5 - 10 = -5`
- `w₂ = -5 * 3 = -15`

Step 3: Simplify `B3`
- φ-functions have only one reachable predecessor (`B2`)
- `z₃ = φ(⊥, -5) = -5` (`B1` unreachable)
- `w₃ = φ(⊥, -15) = -15`
- `result = -15 + (-5) = -20`

Step 4: Dead code elimination
- `B1`: unreachable → delete
- `B2`: intermediate values only → inline
- `B3`: φ-functions collapse → eliminate
- Final: `result = -20`

Why others wrong:
- A: Keeps unnecessary blocks and condition
- B: Retains eliminated variables `x₁`, `y₁`
- C: φ-functions should be eliminated (single reachable path)
- D: DCE removes unreachable code including φ-functions
- E, G, H: Retain unnecessary intermediate computations

Key insight: SSA + constant folding + DCE chain together: constants → unreachable blocks → simplified φ-functions → complete elimination.

---

## Question 11: SSA-Based Value Range Propagation and DCE
```
  // Original code in SSA form:
  B0:
    a₁ = 10
    b₁ = 20
    goto B1

  B1:
    a₂ = φ(a₁, a₃)
    if (a₂ > 100) goto B2 else goto B3

  B2:
    c₁ = a₂ / 2
    goto B4

  B3:
    a₃ = a₂ + 1
    if (a₃ < 15) goto B1 else goto B4

  B4:
    a₄ = φ(a₂, a₃)
    c₂ = φ(c₁, 0)
    result = a₄ + c₂
```

After value range analysis proves `a₂` is always `≤ 14`, which code remains?
* A) All blocks remain unchanged
* B) `B0-B1-B3-B4` remain; `B2` deleted; `c₂=0` in `B4`
* C) `B2` is eliminated; `B4: a₄=φ(⊥,a₃); c₂=0; result=a₄`
* D) Entire loop eliminated; `B0: result=14`
* E) `B2` eliminated; `B3`: produces values `11,12,13,14`
* F) `B0: a₁=10; B1-B3: loop executes; B4: result=a₄`
* G) Only `B4: result=14` remains
* H) Cannot eliminate `B2` (may be reachable)

Answer: B

Explanation:

Value range analysis:
- Loop: `a₁=10` → `a₂=10` → `a₃=11` → `a₂=11` → ... → `a₃=14` → exit
- Range of `a₂`: `[10, 14]` (always `≤ 14`)

Dead code elimination:
- `if (a₂ > 100)`: always `FALSE` (`a₂ ≤ 14`)
- `B2` unreachable → eliminate
- `c₁` never defined → dead

Simplified `B4`:
- `a₄ = φ(⊥, a₃)` → `a₄ = a₃` (`B2` unreachable)
- `c₂ = φ(⊥, 0)` → `c₂ = 0`
- `result = a₃ + 0 = a₃`

Remaining code:
```
B0: a₁=10; goto B1
B1: a₂=φ(a₁,a₃); if(a₂>100) goto B2 else goto B3  // condition always FALSE, goes to B3
B3: a₃=a₂+1; if(a₃<15) goto B1 else goto B4
B4: a₄=a₃; c₂=0; result=a₄
```

Why others wrong:
- A: `B2` is eliminated
- C: Wrong φ notation and incomplete
- D: Loop cannot be fully eliminated (final value depends on execution)
- E, F: Too vague, doesn't specify elimination correctly
- G: Loop must execute to compute final value (14)
- H: VRP proves `B2` unreachable

Key insight: Value range propagation enables DCE by proving branches always/never taken.

---

## Question 12: SSA Constant Propagation with CFG
```
           ┌─────┐
           │ B0  │
           │x=7  │
           │y=3  │
           └──┬──┘
              │
           ┌──▼────┐
           │ B1    │
           │if(x>5)│
           └──┬────┘
              │
         ┌────┴────┐
         │         │
      ┌──▼──┐  ┌──▼──┐
      │ B2  │  │ B3  │
      │z=x+y│  │z=x*2│
      └──┬──┘  └──┬──┘
         │        │
         └────┬───┘
              │
           ┌──▼─────┐
           │ B4     │
           │w=φ(z,z)│
           │r=w-y   │
           └────────┘
```

After constant propagation and dead code elimination, which blocks remain?
* A) All blocks `B0-B4` remain
* B) `B0, B3, B4` remain (`B2` eliminated)
* C) Only `B0` remains with `r=7`
* D) `B0, B1, B2, B4` remain
* E) `B0, B2, B4` remain (`B3` eliminated)
* F) `B0, B4` remain (`B1-B3` eliminated)
* G) Only `B4` remains with `r=7`
* H) Cannot eliminate any blocks (φ-function prevents DCE)

Answer: E

Explanation:

Constant propagation:
- `B0`: `x=7`, `y=3`
- `B1`: `if(7>5)` → `TRUE` (constant condition)
- Branch `B1→B2` taken, `B1→B3` not taken

Dead code elimination:
- `B3` unreachable → eliminate
- `B2` always executes

Simplified CFG:
```
B0: x=7, y=3
B2: z=10 (computed from 7+3)
B4: w=10 (φ collapses, only B2 reaches), r=7 (computed from 10-3)
```

Remaining blocks: `B0, B2, B4`

Why others wrong:
- A: `B3` is unreachable (dead)
- B: `B3` is eliminated (false branch not taken), `B2` remains
- C, G: Intermediate blocks needed (control flow structure)
- D: `B1` can be simplified but branch still evaluated
- F: `B2` must remain (contains computation)
- H: φ-functions simplify when one predecessor is unreachable

Key insight: Constant branches enable DCE of unreachable paths; φ-functions collapse to single value when only one predecessor is reachable.

---

## Question 13: SSA Constant Propagation with Nested Branches
```
           ┌─────┐
           │ B0  │
           │a=4  │
           │b=6  │
           └──┬──┘
              │
           ┌──▼────┐
           │ B1    │
           │if(a<5)│
           └──┬────┘
              │
         ┌────┴─────┐
         │          │
      ┌──▼────┐  ┌──▼──┐
      │ B2    │  │ B3  │
      │c=a+b  │  │c=a*b│
      │if(c>8)│  └──┬──┘
      └──┬────┘     │
         │          │
    ┌────┴───┐      │
    │        │      │
  ┌─▼──┐  ┌─▼──┐    │
  │ B4 │  │ B5 │    │
  │d=10│  │d=20│    │
  └─┬──┘  └─┬──┘    │
    │       │       │
    └───┬───┘       │
        │           │
 ┌──▼────────────┐  │
 │ B6            │◄─┘
 │d₂=φ(B4:d,B5:d,│
 │    B3:⊥)      │
 │result=d₂+c    │
 └───────────────┘
```

After constant propagation, what is the value of `result` in `B6`?
* A) `result = 30`
* B) `result = 34`
* C) `result = 20`
* D) `result = 14`
* E) `result = ⊤` (overdefined)
* F) `result = 28`
* G) Cannot determine
* H) `result = 24`

Answer: C

Explanation:

Execution path:
- `B0`: `a=4`, `b=6`
- `B1`: `(4<5)=TRUE` → take `B2`, `B3` unreachable
- `B2`: `c=10`, `(10>8)=TRUE` → take `B4`, `B5` unreachable
- `B4`: `d=10`
- `B6`: `d₂=φ(B4:10, B5:⊥, B3:⊥)=10`, `result=10+10=20`

Why others wrong:
- A,B,D,F,H: Incorrect calculation
- E,G: φ collapses to single value (only `B4` reachable)

Key insight: Constant branches eliminate unreachable blocks; φ-functions simplify when only one predecessor is reachable.

---

## Question 14: SSA Dead Code Elimination with CFG
```
           ┌─────┐
           │ B0  │
           │x=2  │
           │y=8  │
           └──┬──┘
              │
           ┌──▼─────┐
           │ B1     │
           │z=x*y   │
           │if(z>10)│
           └──┬─────┘
              │
         ┌────┴────┐
         │         │
      ┌──▼──┐  ┌──▼──┐
      │ B2  │  │ B3  │
      │w=z-5│  │w=z+5│
      └──┬──┘  └──┬──┘
         │        │
         └────┬───┘
              │
           ┌──▼────────┐
           │ B4        │
           │w₂=φ(w₁,w₃)│
           │r=w₂*2     │
           └───────────┘
```

After constant propagation and DCE, what remains at `B4`?
* A) `w₂=φ(11,21); r=w₂*2`
* B) `w₂=11; r=22`
* C) `w₂=21; r=42`
* D) `r=42`
* E) All code remains unchanged
* F) `w₂=φ(11,11); r=22`
* G) `r=22`
* H) Cannot determine

Answer: G

Explanation:

Constant propagation:
- `B0`: `x=2`, `y=8`
- `B1`: `z=16`, `(16>10)=TRUE` → `B2` taken, `B3` unreachable
- `B2`: `w=11`
- `B4`: `φ(11,⊥)=11`, `r=22`

Simplified: `B4` becomes `w₂=11; r=22`, further simplified to `r=22` (`w₂` unused after).

Why others wrong:
- A: φ simplifies (`B3` unreachable)
- B: `w₂` can be eliminated
- C: `B3` unreachable
- D: Wrong path
- E: DCE removes dead code
- F: φ with duplicate args still exists
- H: Fully determinable

---

## Question 15: SSA Constant Folding with Multiple φ-Functions
```
           ┌─────┐
           │ B0  │
           │a=6  │
           │b=3  │
           └──┬──┘
              │
           ┌──▼────┐
           │ B1    │
           │if(a>b)│
           └──┬────┘
              │
         ┌────┴────┐
         │         │
      ┌──▼──┐  ┌──▼──┐
      │ B2  │  │ B3  │
      │x=a-b│  │x=b-a│
      │y=10 │  │y=20 │
      └──┬──┘  └──┬──┘
         │        │
         └────┬───┘
              │
           ┌──▼─────────┐
           │ B4         │
           │x₂=φ(x₁,x₃) │
           │y₂=φ(y₁,y₃) │
           │result=x₂+y₂│
           └────────────┘
```

After constant propagation, what is `result` at `B4`?
* A) `result = 13`
* B) `result = 23`
* C) `result = 17`
* D) `result = -17`
* E) `result = ⊤`
* F) `result = 30`
* G) `result = 10`
* H) `result = 7`

Answer: A

Explanation:

Execution:
- `B0`: `a=6`, `b=3`
- `B1`: `(6>3)=TRUE` → `B2`, `B3` unreachable
- `B2`: `x=3`, `y=10`
- `B4`: `x₂=3`, `y₂=10`, `result=13`

Why others wrong:
- B-H: Wrong path or calculation
- E: Fully determinable via constant propagation

---
