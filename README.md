# Fable 5 vs Opus 4.8 — A Real-World Planning Head-to-Head

A controlled, side-by-side comparison of two Anthropic models — **Claude Fable 5** and **Claude Opus 4.8 (1M context)** — given the *exact same* software-planning task and measured on **time, cost, quality, and thoroughness**.

## About this project

Picking the right model for the right phase of work saves real time and money. A common, pragmatic workflow is:

> **Plan** with the most capable model → **implement** with a cheaper/faster one → **review** with a capable model again.

This repo stress-tests the **planning** phase of that workflow. Both models were handed an identical brief — *"split the student portal into a dedicated `/my-library` home and a catalog-only `/workshops`"* — and asked to take it from a rough idea to a validated, implementation-ready spec using **OpenSpec**'s `explore → propose` skills.

The conditions were held as equal as possible:

| Control | Value |
|---|---|
| Task | The same one-paragraph brief (a real feature on a real codebase) |
| Skills | OpenSpec `explore` → `propose` |
| Effort | **Xhigh** for both models |
| Mode | **Planning only** — no code was written |
| Runs | One session per model · 2026-06-13/14 |

Each session's **cost and wall-clock time were captured live from the running session** (wall-clock includes the human's thinking and interaction time, not just model latency). The full breakdown — what each model produced, where they converged, where they diverged, and which to reach for when — is the analysis below.

## The real-world context

This wasn't a toy benchmark — the task is a real feature on a real product I'm building: **Aimplified Academy**, a self-hosted **LMS** (learning-management system). That's what makes the cost and time numbers meaningful: both models were doing actual work I needed done on a live codebase.

**The codebase.** An Nx monorepo — a React frontend (`projects/frontend`) and a Node/TypeScript backend (`projects/services`) over Postgres, built with hexagonal architecture and vertical slicing, plus a Playwright e2e suite. Auth is passwordless (magic-link), and the platform bootstraps its first `owner` at runtime through an onboarding flow.

**The problem.** The `/workshops` page was doing two jobs at once: it showed the learner's **own enrolled courses** (a "My courses" progress rail) *and* the **full catalog** to browse and enroll from. As the catalog grows, that conflation only gets worse — and there's no dedicated home for a learner to come back to and pick up where they left off.

**The feature being planned.** Split those two jobs into two pages:

- **`/workshops`** stays the browse-and-enroll **catalog**, with all its existing routes unchanged.
- **`/my-library`** becomes the learner's **home** — all of their enrolled courses and workshops gathered in one place, organized Udemy-style (a continue-learning shelf, an all-courses view, and a completed view) — and the **default landing page for authenticated users**.

The hard constraint I gave both models: **use only data already in the database — no new tables, columns, or tracking.** That one line is what both planning sessions had to reckon with, and it's where their depth diverged the most (see the analysis).

## What's in this repo

| Path | Contents |
|---|---|
| [`opus-plan/`](./opus-plan/) | **Opus 4.8's** output: the `add-my-library` OpenSpec change (proposal · design · tasks · 3 spec deltas) plus [`planning-with-opus-4.8.md`](./opus-plan/planning-with-opus-4.8.md), a writeup of the session. |
| [`fable-plan/`](./fable-plan/) | **Fable 5's** output: the `split-student-portal-my-library` OpenSpec change (proposal · design · tasks · 3 spec deltas) plus a [session README](./fable-plan/planning-session-with-fable-5/README.md). |
| `README.md` | This file — the full comparative analysis. |

> Both changes solve the same feature, so the folders share the same shape (`proposal.md`, `design.md`, `tasks.md`, `specs/{my-library,course-consumption,workshops}/spec.md`) — read them side by side.

---

## TL;DR

Both models independently arrived at **the same core architecture** — proof both are highly capable planners. The difference is at the edges:

- **Fable 5** bought you a *more thorough artifact*: it caught two real latent bugs, wrote ~56% more scenarios, and closed every open question — at **2.9× the cost and 2.2× the time**.
- **Opus 4.8** bought you *efficiency and sharpness*: a tighter, literal-to-brief plan with a cleaner core insight and finer task slices, **3× cheaper and 2× faster** — but it shipped two latent issues for a later phase to catch.

> **Bottom line:** Fable wins the *plan*; Opus wins the *dollar*. For most features Opus 4.8 Xhigh is the better default planner; reserve Fable 5 Xhigh for high-stakes or legacy-heavy work where a missed edge case is expensive.

---

## 1. Headline metrics (from the live sessions)

| Metric | 🟣 Fable 5 | 🟢 Opus 4.8 (1M) | Delta |
|---|---|---|---|
| **Session cost** | **$19.23** | **$6.54** | Fable **2.94× more** |
| **Wall-clock (incl. user thinking)** | **58m 02s** | **26m 38s** | Fable **2.18× longer** |
| Context used | 183.4k (18%) | 194.4k (19%) | ~even (Opus +6%) |
| Branch | `ux/my-courses` | `ux/my-courses-opus` | — |
| Change produced | `split-student-portal-my-library` | `add-my-library` | — |
| `openspec validate` | ✅ valid | ✅ valid (strict) | both pass |

The cost gap (2.94×) is *wider* than the wall-clock gap (2.18×) despite near-identical final context. That tells us Fable burned tokens faster than wall-clock implies — consistent with its **subagent fan-out** strategy (see §6).

---

## 2. Output volume & structure

| Output | 🟣 Fable | 🟢 Opus | Note |
|---|---|---|---|
| Requirements (total) | **11** | 9 | Fable wider scope |
| **Scenarios (total)** | **50** | 32 | Fable **+56%** |
| Tasks | 15 | **19** | Opus finer-grained |
| Task groups | 8 | 6 | — |
| Design decisions | 8 | 8 | tie |
| Risk/trade-off bullets | 7 | 6 | — |
| Change-folder words | 5,789 | 5,214 | Fable +11% |
| `BREAKING` flagged | 2 | 0 (additive framing) | scope/interpretation |

**Scenarios per capability** — where Fable's extra coverage lands:

| Spec (capability) | 🟣 Fable | 🟢 Opus |
|---|---|---|
| `course-consumption` | **20** (3 reqs) | 9 (1 req) |
| `my-library` | 18 (4 reqs) | **20** (6 reqs) |
| `workshops` | **12** (4 reqs) | 3 (2 reqs) |

Opus actually edges Fable on the *new* capability (`my-library`, 20 vs 18). Fable's lead comes entirely from **touching more of the surrounding system** — it reworked `course-consumption` (3 requirements vs 1) and `workshops` (12 scenarios vs 3).

---

## 3. The core design — they converged

Independently, both models reached an almost identical architecture. This is the most important finding: **on the core problem, the two are interchangeable.**

| Decision | Both models chose |
|---|---|
| Surface `lastCompletedAt` + `enrolledAt` | from **existing columns**, no migration |
| Ordering / grouping | a **pure, unit-tested frontend function**; API stays an unordered set |
| Continue-learning rail/hero | ≤2 in-progress courses w/ a completion, by last-completion desc |
| "All" tab | every enrolled course; never-started fall back to `enrolledAt` |
| "Completed" tab | `progress === 100` |
| Tabs | **route-based `NavLink`s + `aria-current`**, *not* the in-page Tabs widget |
| Slice reuse | reuse `WorkshopsUseCasesProvider` + `useMyCourses` (**no new gateway**) |
| Landing | `/` and post-login → `/my-library` |
| Shared card | one `CourseCard` for library + catalog |

The divergence below is all *around* this shared spine.

---

## 4. Where Fable went deeper (the $13 premium)

**① Caught the `Math.round` completion bug — scope-independent, a real defect.**
`Enrollment.progress()` rounds, so 200/201 completions → 99.5 → **100**. Fable spotted that `progress === 100` was therefore *not* a sound "completed" predicate, switched it to `Math.floor`, and added scenarios for it. **Opus missed this** and even wrote `progress === 100 (⇔ resumeLessonSlug === null)` in its design — an equivalence that is provably false both for the rounding case *and* for a 0-lesson course. This is a genuine latent bug Opus's plan would have shipped.

**② Discovered the catalog *excludes* enrolled courses.**
While writing spec deltas, Fable found `ListShownCoursesUseCase` filters out the caller's enrolled courses (server-side only — invisible from the frontend). Because Fable had asked the user "should the catalog badge enrolled courses?" (answer: yes), it *needed* the catalog to include them, so it removed the filter and dropped the now-dead `EnrollmentsRepository` dependency. **Opus never examined the catalog use case** — its narrower "catalog = enroll-only" reading happened to sidestep the issue, but it planned blind to it.

**③ Exhaustive edge-case coverage.** Fable tested *dynamic* behavior ("mark a lesson complete → course jumps to the top"), the 0-lesson course, owner-Dashboard-link loading states, "enrolled overrides closed enrollment," etc. It also closed **all** open questions during exploration ("Open Questions: None"). Opus left 2 (minor, with defaults).

---

## 5. Where Opus was sharper (the efficiency dividend)

**① The single best insight in either session.** Opus's design compares timestamps as **lexicographic ISO-8601 strings** — no `Date` parsing, no timezone class of bugs. That's a more defensive, more elegant call than Fable's `Date`-object sorting. Opus's data-flow diagram ("one column dropped at step 2 of a 6-layer chain") is also the clearest single artifact produced by either model.

**② Higher brief fidelity.** The brief said "in `/workshops` we leave everything like it is." Opus honored that literally (`add-my-library`, additive, 0 BREAKING). Fable reinterpreted it as a *split* with 2 BREAKING changes. Both are defensible — Opus stayed closer to what was literally asked.

**③ Better handoff ergonomics.** Opus produced **19 finer tasks** (vs Fable's 15 chunkier ones) — e.g. it split rail / enrolled-tab / completed-tab into separate one-TDD-cycle tasks where Fable bundled them. For the "cheaper model implements one commit at a time" workflow, finer slices are easier to execute.

**④ Leaner, smoother process.** Opus read files directly and inline. Fable leaned on parallel subagents and hit friction (subagents returning "I delivered a report" instead of the report; a `ctx_batch_execute` blowing the token limit; a bad `rg` flag producing a false negative) — all recovered, but that recovery is part of the 2.9× bill.

---

## 6. Why the cost/time differ: two planning *strategies*

| | 🟣 Fable 5 | 🟢 Opus 4.8 |
|---|---|---|
| Exploration style | **Subagent fan-out** (multiple parallel Explore/Plan agents + `ctx` sweeps) | **Direct inline reading** (named the 8 key files, traced the read path) |
| Consequence | broader coverage, more tokens, more friction to recover from | targeted, fewer tokens, faster, tighter |
| Result | thorough but expensive | efficient but slightly less exhaustive |

This is the crux: the strategy *explains* the numbers. Fan-out finds more (the catalog exclusion, the rounding bug) but pays for parallel token burn and lossy subagent round-trips. Direct reading is cheaper and sharper but covers less ground.

---

## 7. Latent issues each plan would pass downstream

| Issue | 🟣 Fable | 🟢 Opus |
|---|---|---|
| `progress===100` unsound under `Math.round` | ✅ fixed (floor) | ❌ shipped |
| 0-lesson course mis-flagged "Completed" | ✅ handled | ⚠️ wrong equivalence in design |
| Catalog excludes enrolled courses | ✅ addressed | ⚪ not examined (sidestepped by scope) |
| Open questions left for later | 0 | 2 (minor, defaulted) |
| Meta-doc self-miscount | said "13 tasks" → actually 15 | said my-library "7 req/22 scen" → actually 6/20 |

*(Both meta-docs have a small counting slip — a wash.)*

---

## 8. My scorecard (1–5)

Split into **artifact quality** (the plan itself) vs **efficiency** (what it cost to get it).

| Dimension | 🟣 Fable | 🟢 Opus | Edge |
|---|---|---|---|
| **— Artifact quality —** | | | |
| Core architecture | 5 | 5 | tie |
| Latent-bug / hidden-constraint discovery | 5 | 2.5 | **Fable** |
| Scenario / test coverage | 5 | 3.5 | **Fable** |
| Decision closure (open Qs) | 5 | 4 | Fable |
| Risk analysis depth | 4.5 | 4 | Fable |
| Brief fidelity (literal scope) | 3.5 | 5 | **Opus** |
| Task granularity for handoff | 3.5 | 4.5 | Opus |
| Engineering elegance (sharp calls) | 4.5 | 5 | Opus |
| Communication / diagrams | 4.5 | 5 | Opus |
| **Artifact subtotal (avg)** | **4.5** | **4.3** | Fable (slim) |
| **— Efficiency —** | | | |
| Cost efficiency ($/value) | 2 | 5 | **Opus** |
| Speed | 2.5 | 5 | **Opus** |
| Process smoothness | 3 | 4.5 | Opus |
| **Efficiency subtotal (avg)** | **2.5** | **4.8** | Opus (large) |

**Cost efficiency check:** $0.385/scenario (Fable) vs **$0.204/scenario (Opus)** → Opus is **1.9× more cost-efficient**, and 1.4× faster per scenario (1.20 vs 0.86 scenarios/min). Fable's scenarios aren't all equal, though — two of them encode bug fixes worth more than a routine scenario.

---

## 9. Wrap-up & recommendation

**Result:** A near-tie on the *artifact* (Fable 4.5 vs Opus 4.3) and a blowout on *efficiency* (Opus 4.8 vs Fable 2.5). Fable earns its premium through latent-bug discovery and exhaustive coverage; Opus earns its lead through speed, cost, brief-fidelity, and a sharper core design. Both shipped a valid, implementable, architecturally-sound plan — they did not disagree on *how* to build the feature, only on *how much ground to cover and how hard to dig*.

### When to use each

**🟢 Reach for Opus 4.8 (Xhigh) by default — most planning:**
- 3× cheaper, 2× faster — you can plan many features, or iterate, for one Fable run.
- Stays literal to the brief; minimal scope creep.
- Finer, TDD-cycle-sized tasks → easiest handoff to a cheaper implementer.
- Best when the codebase is well-understood / lower-risk **and you have a real review phase** to catch the occasional latent issue.

**🟣 Reach for Fable 5 (Xhigh) when the stakes justify ~3×:**
- The feature touches **subtle or legacy logic** where latent bugs hide (it found the `Math.round` defect and the hidden catalog filter that three exploration passes missed).
- You want the plan to be a **complete source of truth** — every edge case specced, every question closed — so a cheaper implementer (and even the reviewer) has little left to catch.
- Correctness and exhaustiveness matter more than budget or turnaround.

### Fit to your 3-phase workflow (plan → implement-cheap → review)

The economic question is **where you'd rather pay to catch a subtle bug.** Plan with Fable and you front-load the cost but the rounding bug never reaches code. Plan with Opus and you save ~$13 per feature, betting your **review phase** (a powerful model again) catches what the leaner plan missed. For routine features that bet pays off handsomely. For a gnarly one, paying Fable at design time — the cheapest place to fix anything — is the safer trade.

> ⚠️ **Caveat:** this is **one feature, one run each**. Some of Fable's extra scope was prompted by a question *it chose to ask* (the enrolled-badge decision), and some of the gap is run-to-run variance and tooling friction. Treat the *direction* (Fable = thorough/expensive, Opus = efficient/sharp) as the signal, not the exact 2.9×.
