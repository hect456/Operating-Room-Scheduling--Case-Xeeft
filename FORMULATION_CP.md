# CP-SAT Model: Variables, Objective, Constraints

This is the math behind the model this project is built around
(`src/solvers/cp_sat_interval_solver.py`). FORMULATION.md covers the problem framing,
the evidence behind the priority mechanism, and the case for CP over a bigger MILP; this
document maps directly onto code. Every constraint below carries the same C-number as the
comment next to it in the solver, so the two read side by side.

## 1. Sets and parameters

Unchanged from FORMULATION.md §5: $`C, D, R, H, E`$ and every
$`t_c^{op}, t_c^{clean}, t_c^{tot}, k_{dr}, k_{hd}, k_h, p_c, dd_c, \mu_p, w_c, \alpha, u_{ce}, \kappa_{ed}`$,
plus the bed-pool parameters $`\rho(c)`$, $`\text{los}_c`$, $`\beta_\rho`$, $`\pi^{\text{ovf}}`$.
These last four only exist in this model; a day-bucket formulation has no value
that means "day of surgery" for a multi-day stay to start counting from
(FORMULATION.md, Appendix A.3).

## 2. Decision variables

For every $(c,d,r)$ that survives the eligibility filter (room-service roster C4,
ambulatory-only C5, pediatric block C6, surgeon availability):

$$\text{pr}_{cdr}\in\{0,1\} \qquad \text{start}_{cdr}\in[0,\,k_{dr}]$$

Two interval variables share that same start time but represent two different resources,
each with its own end:

$$
\text{iv}_{cdr}
= \mathtt{NewOptionalIntervalVar}\!\bigl(
    \text{start}_{cdr},\;t_c^{tot},\;\text{end}_{cdr},\;\text{pr}_{cdr}
  \bigr)
\quad \text{room occupancy, size } t_c^{tot}
$$

$$
\text{sgiv}_{cdr}
= \mathtt{NewOptionalIntervalVar}\!\bigl(
    \text{start}_{cdr},\;t_c^{op},\;\text{sgend}_{cdr},\;\text{pr}_{cdr}
  \bigr)
\quad \text{surgeon time, size } t_c^{op}
$$

The room's interval runs through the cleaning buffer ($t_c^{tot}=t_c^{op}+t_c^{clean}$);
the surgeon's ends as soon as the operation does. They have to be two separate
variables, not one shared between C7 and C8: a surgeon is free to scrub into a different
room the moment their own case ends, even while support staff are still cleaning the
first room, and collapsing both checks onto the room's longer interval would wrongly
forbid that. Both intervals are *present* (i.e., actually constrain whatever resource they touch)
exactly when $\text{pr}_{cdr}=1$, which is what lets `NoOverlap` and `Cumulative`
reason correctly over every candidate slot a case could occupy without the solver first
deciding which candidates are real.

Candidates are only generated for $(c,d,r)$ triples that pass eligibility, the model's
main variable-count reduction, done once up front rather than filtered out by
constraints later.

For every case that isn't priority-4:

$$u_c\in\{0,1\}, \qquad \sum_{d,r}\text{pr}_{cdr} + u_c = 1$$

## 3. Objective

$$\min\;Z = T_1 + T_2 + T_3 + T_4$$

$$
T_1 = \sum_{\substack{c:\,dd_c \ge 0 \\ d \in D_c,\; r}} (dd_c + d)\,\text{pr}_{cdr}
\qquad \text{(on-time cases, prefer early days)}
$$

$$
T_2 = \sum_{\substack{c:\,dd_c < 0 \\ d \in D_c,\; r}} (dd_c + \alpha\,d)\,\text{pr}_{cdr}
\qquad \text{(overdue cases, urgency-weighted)}
$$

$$
T_3 = \sum_{c:\,p_c \ne 4} w_c\,u_c
\qquad \text{(non-scheduling penalty)}
$$

$$
T_4 = \pi^{\text{ovf}} \sum_{c:\,\rho(c) \ne \varnothing} \text{overflow}_c
\qquad \text{(bed-overflow penalty)}
$$

with $w_c = \mu_{p_c}\cdot\text{PenaltyCurve}(dd_c) + 1.2\cdot\max_{c'\in C}dd_{c'}$
(`src/model/penalty.py`), computed once and shared by every backend so there is one
place in the codebase that decides what an unscheduled case costs. CP-SAT requires
integer objective coefficients; each term is rounded to the nearest integer at build
time, introducing rounding error well under 1 per term at these coefficient magnitudes
(tens to low thousands).

## 4. Constraints

**C1: at most one occurrence per patient per week:**
$$\sum_{c:\,\text{patient}(c)=n}\sum_{d,r}\text{pr}_{cdr} \le 1 \qquad \forall n$$

**C2: priority-4 cases run on day 1:** $\sum_r \text{pr}_{c,1,r}=1$, every other
candidate slot for that case forced to 0.

**C3: every other case is scheduled exactly once, or counted as unscheduled,** as
written into the $u_c$ definition in §2.

**C4-C6: eligibility** (room-service roster, ambulatory-only rooms, the pediatric
block): resolved during candidate generation; they never appear as constraint rows
because a triple that fails them never gets a variable.

**C7: room capacity, exact, declared over the room interval:**

```math
\mathtt{AddNoOverlap}\!\bigl(\{\,\text{iv}_{cdr} : d,\,r\text{ fixed}\}\bigr)
\qquad \forall\,d,\,r
```

A room runs one case at a time, and the next case cannot start until the previous one's
cleaning buffer has elapsed. That buffer is already inside $`\text{iv}_{cdr}`$'s length.

**C8: surgeon, exact non-overlap on the surgeon's own interval, plus a daily cap:**

```math
\mathtt{AddNoOverlap}\!\bigl(\{\,\text{sgiv}_{cdr} : \text{surgeon}(c)=h,\;d\text{ fixed}\}\bigr)
\qquad \forall\,h,\,d
```

```math
\sum_{\substack{c:\,\text{surgeon}(c)=h \\ r}} t_c^{op}\,\text{pr}_{cdr}
\;\le\; k_{hd}
\qquad \forall\,h,\,d
```

Declared over $`\text{sgiv}_{cdr}`$ (size $`t_c^{op}`$), not the room's interval (see §2).
The minutes cap is kept alongside the `NoOverlap` because non-overlap alone bounds
concurrency, not total hours worked.

**C9: surgeon weekly time limit:**

```math
\sum_{\substack{c:\,\text{surgeon}(c)=h \\ d,\,r}} t_c^{op}\,\text{pr}_{cdr}
\;\le\; k_h
\qquad \forall\,h
```

**C10: shared equipment, exact concurrency:**

```math
\mathtt{AddCumulative}\!\bigl(
  \{\,\text{iv}_{cdr} : u_{ce}=1,\;d\text{ fixed}\},\;
  \text{demands}=1,\;
  \text{capacity}=\kappa_{ed}
\bigr)
\qquad \forall\,e,\,d
```
Declared over the room interval, since the equipment sits in the room for the full
$t_c^{tot}$, cleaning included, unlike the surgeon who leaves early. This checks literal
time overlap rather than a day-count, which is the constraint FORMULATION.md §3 argues
for and RESULTS.md measures.

**C11: recovery/ICU beds, with an explicit horizon-overflow term.** For each case $c$
with $\rho(c)\ne\varnothing$, channel whichever $(d,r)$ slot gets chosen into a day
index via `OnlyEnforceIf` (whenever $\text{pr}_{cdr}=1$):

$$
\text{dayof}_c = d_{\mathrm{idx}}
$$

The bed interval's presence literal differs by case type:

```math
\text{bed}_c = \begin{cases}
  \mathtt{NewOptionalIntervalVar}\!\bigl(
    \text{dayof}_c,\;\text{los}_c,\;\text{dayof}_c+\text{los}_c,\;1-u_c
  \bigr) & p_c \ne 4 \\[6pt]
  \mathtt{NewIntervalVar}\!\bigl(
    \text{dayof}_c,\;\text{los}_c,\;\text{dayof}_c+\text{los}_c
  \bigr) & p_c = 4
\end{cases}
```

Priority-4 cases have no $u_c$ (they are always scheduled by C2), so their bed interval
is mandatory rather than conditional. For all other cases, the presence literal $1-u_c$
ensures the interval only occupies capacity when the case is actually scheduled.

$$
\mathtt{AddCumulative}\!\bigl(
  \{\,\text{bed}_c : \rho(c)=\rho\},\;
  \text{demands}=1,\;
  \text{capacity}=\beta_\rho
\bigr)
\qquad \forall\,\rho
$$

Bed capacity $\beta_\rho$ is constant across the week. A stay starting late in the
horizon (a 2-day stay starting Friday, say) can run past it into what would be the
weekend, a regime this constant-capacity model has no separate capacity for. Rather than
silently approximate that or forbid it outright, every day of overflow is charged in the
objective via `AddMaxEquality`:

$$
\text{overflow}_c
= \max\!\bigl(0,\;(\text{dayof}_c+\text{los}_c) - n_{\text{days}}\bigr)
\qquad \Rightarrow\quad
\pi^{\text{ovf}}\cdot\text{overflow}_c \;\text{added to Term 4}
$$

Setting $\pi^{\text{ovf}}=0$ recovers the simpler, silent behavior. The default of 50 makes
crossing the boundary discouraged but not infeasible, an instance-level policy choice
(`PlanningInstance.weekend_bed_overflow_penalty`).

## 5. Why C7–C11 don't need an explicit channeling constraint between room and surgeon

C7, C10, and C11 are declared over the room interval $\text{iv}_{cdr}$; C8 is declared
over the surgeon interval $\text{sgiv}_{cdr}$. Both share the same `start` variable, so
CP-SAT propagates "this case's start is pinned by its room's schedule" into "...which
also constrains its surgeon's schedule," and vice versa, without an explicit constraint
linking the two. It falls directly out of sharing one `start` per candidate slot, with
two different `end`s for the two different resources that slot touches.

## 6. Search

CP-SAT runs with its default parallel portfolio (`num_search_workers`, capped at
`min(16, os.cpu_count())`) and a relative gap target rather than a hand-written
branching strategy (see FORMULATION.md §3). The cap at 16 is deliberate: below 8 cores
the cap has no effect; between 8 and 16 the full count is used; above 16, clause-sharing
overhead begins to outweigh the added search diversity for instances of this size. The
reported gap is the genuine bound-vs-incumbent gap, computed even when the status is
`Optimal`: with a relative gap target set, CP-SAT's `Optimal` means "proven within that
tolerance," the same convention Gurobi uses, not necessarily a literal 0%.
