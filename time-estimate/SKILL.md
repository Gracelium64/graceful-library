---
name: time-estimate
description: Trigger when a build/implementation task is about to start and a time estimate would help (asked directly, or automatically from hackathon-mode/night-shift's own flow), and again when that build finishes, to run a retro comparing the estimate to actual time. Gives a conservative estimate (historical/analogous-build anchored) and a best estimate (mechanical-throughput decomposition) instead of one vague number, then self-calibrates its own defaults from the gap between estimate and actual across builds. Also invoke by name ("time estimate", "/time-estimate") for a one-off estimate outside a build-mode skill.
---

# Time Estimate

Gives two numbers, not one vague range: a **conservative estimate** (what
this looks like next to comparable past builds) and a **best estimate**
(what the actual mechanical work should take, decomposed), plus a one-line
read on what's driving the gap between them. After the build, runs a retro
against the real elapsed time and appends what was learned to two logs — one
global (cross-project), one local to the project the build happened in — so
later estimates get tighter over time instead of restating the same guess.

Has two modes. Which one runs is determined by when it's called, not by the
caller naming it explicitly:

- **`estimate`** — called before implementation starts.
- **`retro`** — called after the build is verified complete.

## Data sources to check before estimating

1. **This project's local log**: `<project-root>/docs/TIME_ESTIMATES.md`.
   Weight entries here most heavily — they reflect what's actually proven
   vs. novel in *this* codebase specifically. In a monorepo of separate
   sub-projects (a hackathon-template repo holding several independent
   builds, for instance), "project root" means the nearest ancestor
   directory that is itself a self-contained build (has its own
   package.json / repo identity / existing `docs/` folder already in use for
   that sub-project) — not necessarily the outermost repo root.
2. **The global cross-project log**: `~/.claude/skills/time-estimate/calibration-log.md`.
   If it doesn't exist yet, create it with the table header from the log format below.
   Filter to entries whose tags overlap the current build's tags (see
   tag vocabulary below) — e.g. a past "n8n-integration" entry from a
   different project is still relevant signal for a new project also
   touching n8n; a past "React-portal-architecture" entry usually isn't.
3. If neither has a relevant entry, fall back to the default multipliers in
   the "Defaults" section below — these get refined over time (see
   "Self-calibration").

## `estimate` mode

1. **Identify which build-mode skill is driving this build** (hackathon-mode,
   night-shift, plain/no mode) — this matters because some modes carry fixed
   wall-clock overhead unrelated to task complexity (night-shift's mandatory
   10-minute inter-phase wait, for instance). Note it; it feeds the
   conservative number's "total wall-clock" side, never the "active work"
   side.
2. **Decompose the plan into 4 buckets** — do this explicitly, don't skip to
   a gut number:
   - **Bucket 1 — mechanical, pattern-repeating work.** Steps that copy an
     already-established pattern (proven once, then repeated N times: same
     screen-wiring shape applied to 3 more screens, same kind of config edit
     applied to 2 more files). Near-zero marginal cost per repetition when
     each step has an automated `Verify:` command. Budget roughly 1-2
     minutes per step unless a relevant log entry says otherwise.
   - **Bucket 2 — genuinely novel architecture.** The piece(s) with no
     established precedent in this codebase to copy. This is the only
     bucket that gets a real risk buffer, because it's the only place a
     genuine unknown can hide. Size the buffer from a matching-tag log
     entry if one exists; otherwise default 5-15 minutes per novel piece,
     scaled up if the novel piece touches more than one file/concept at
     once.
   - **Bucket 3 — human/external-system-gated steps.** Anything gated on
     the user's own pace against a live system (re-importing a workflow,
     manual browser click-through, waiting on an external service).
     **Exclude these from the time estimate entirely** — name them
     separately as "not estimated, depends on you."
   - **Bucket 4 — automated verification.** Build/test runs. Wall-clock-cheap
     (seconds per run) — do not pad for this the way a human QA pass would
     be padded.
3. **Best estimate = sum(bucket 1) + sum(bucket 2).**
4. **Conservative estimate**: anchor on the most analogous past build's
   actual wall-clock time (from either log), or if none exists, apply the
   default conservative multiplier (see Defaults) to the best estimate.
   Add mode-mandated wait overhead (bucket-independent) if the active mode
   has any, and call that out as its own line item, not folded silently in.
5. **Present both numbers** plus one line naming the main driver of the gap
   (usually: how much of bucket 2 is genuinely unproven, or how much
   mode-mandated wait time is baked into the conservative number).
6. **Record a pending entry** (see log format below) in both logs with
   `actual: pending` — the retro step fills this in later. If a build never
   reaches retro (abandoned, scope changed), leave it pending rather than
   guessing; don't backfill a fabricated actual.

## `retro` mode

1. **Compute actual elapsed time**, split into:
   - Active work time (the actual editing/build/verify work).
   - Mode-mandated wait time (e.g. night-shift's inter-phase waits) if any —
     computed separately, never merged into "active."
   - Time spent on bucket-3 (human/external) steps, if it happened inside
     the same session — reported separately again, never used to judge
     whether the *estimate* was accurate, since that time was never part of
     the estimate.
2. **Compare against the pending estimate entry** recorded at start. Compute
   the gap ratio for both the conservative and best numbers against actual
   active-work time.
3. **Root-cause the gap** (only if there is one worth noting — a close
   estimate needs no essay): pick from — novel-architecture risk understated,
   novel-architecture risk overstated, a bucket-1 step turned out not to be
   as mechanical as assumed, mode overhead miscounted, other (name it).
4. **Append a completed entry** to both logs (see format below).
5. **If a tag now has 3+ completed entries** in the global log, recompute
   that tag's default buffer/multiplier as their average and update the
   "Defaults" section of *this file* to match — this is the self-calibration
   step. Only ever touch the numeric defaults for the specific tag that just
   got new data; never rewrite the method sections (the bucket definitions,
   the log format) as part of this step — a structural change to the method
   itself is a deliberate edit the user should see and approve separately,
   not something this step does on its own.
6. **Always show what was appended and, if it happened, what default
   changed and why** — the log writes happen automatically (already
   authorized), but stay visible in the response every time rather than
   silent.

## Log format (both logs use the same row shape)

Markdown table, append-only — never edit or delete a past row, only add new
ones (correcting a mistake means adding a note in a new row, not rewriting
history):

```markdown
| Date | Project | Tags | Mode | Conservative | Best | Active actual | Wait overhead | Bucket-3 (unestimated) | Gap driver | Lesson |
|------|---------|------|------|--------------|------|----------------|---------------|--------------------------|------------|--------|
```

- **Tags**: comma-separated, lowercase-kebab, from the vocabulary below (or a
  new tag if genuinely nothing fits — extend the vocabulary in this file's
  "Tag vocabulary" section when that happens, and say so).
- **Project** column only appears meaningful in the *global* log (the
  project-local log implicitly is that project, so this column can just
  repeat the project name for consistency, or be omitted there — either is
  fine, keep it consistent within a given log file).
- `actual: pending` for an entry not yet retro'd.

### Tag vocabulary (extend as needed, note it when you do)

`n8n-integration`, `portal-architecture` (or the more general
`novel-state-persistence-pattern`), `new-screen-wiring`,
`config-file-extension`, `test-infra-setup`, `api-endpoint-addition`,
`design-system-restyle`, `dependency-upgrade`, `first-time-infra` (generic
catch-all for "never done this kind of integration before, in any stack"), `codebase-documentation` (lesson plans, architecture diagrams, write-ups of an already-read codebase), `compiler-toolchain` (parsers, code generators, transpilers, build CLIs), `language-server` (LSP servers, editor-extension language features), `xr-device` (hardware-in-loop XR builds: headset-only verification of input, latency, tracking).

## Defaults (used only when no relevant log entry exists yet — refine via `retro` mode, not by hand-editing on a whim)

- Conservative multiplier over best estimate, with no matching tag data:
  **3x**. (Historical note: the first-ever calibration point in this file's
  life used a ~4-5x gap, but that was measured against a bucket-1/2 decomposed
  best estimate produced *after the fact* — go with 3x as the forward-looking
  default until enough retro data exists to justify a different number.)
- Novel-architecture buffer (bucket 2), per unproven piece with no matching
  tag data: **5-15 minutes**, scaled toward the top of that range if the
  novel piece spans more than 2-3 files or introduces a pattern with no
  existing test coverage to lean on.
- Mechanical step (bucket 1), per repeated pattern instance: **1-2 minutes**,
  assuming an automated `Verify:` command exists for it.
- Per-tag, `config-file-extension` (14 completed entries, recomputed
  2026-09-30): actual active time averaged **0.41× the best estimate**
  (0.95, 0.80, 0.28, 0.44, 0.14, 0.16, 0.45, 0.27, 0.28, 0.17, 0.60, 0.33, 0.60, 0.28). For this tag, scale the decomposed best estimate by
  ~0.4. The conservative number has overstated every time.
- Per-tag, `first-time-infra` (28 completed entries, recomputed 2026-10-20):
  actual active time averaged **0.49× the best estimate** (0.22 [2026-10-20], 0.80, 0.28, 0.44,
  0.14, 0.27, 1.12, 0.45, 0.27, 0.28, 0.97, 0.17, 0.51, 0.19, 0.45, 0.43, 0.38, 0.36, 0.60, 0.28, 0.25, 0.30, 0.41, 3.3, 0.31, 0.25, 0.10, 0.19;
  the 3.3 is the 2026-10-06 agent-server + theme-tuning build, where scope grew mid-build; without it the average is 0.39;
  the 0.10 is a meta-framework build (stage 1), a multi-day compiler/LSP build sized from a human-hours raw decomposition). Scale the decomposed best estimate by ~0.45, and keep bucket 2's
  buffer: the 2026-09-24 overrun came from a second-framework mirror that needed
  its own layout runtime, a novel piece the plan had budgeted as mechanical.
- Per-tag, `design-system-restyle` (13 completed entries, recomputed
  2026-10-20): actual active time averaged **0.68× the best estimate** (0.22 [2026-10-20], 0.28,
  0.44, 0.27, 1.12, 0.45, 0.97, 0.48, 0.28, 0.36, 0.30, 0.41, 3.3; the 3.3 had an unsurveyed
  second token system, without it 0.49). Survey every :root block before sizing a theme pass. Scale the decomposed best estimate by ~0.5.
- Per-tag, `n8n-integration` (14 completed entries, recomputed 2026-10-06):
  actual active time averaged **0.66× the best estimate** (0.95, 0.14, 1.12, 0.16, 0.28, 0.17, 0.51, 0.60, 0.32, 0.33, 0.43, 0.69, 0.25, 3.3;
  the 3.3 is the n8n → code migration with mid-build scope growth, without it 0.46).
- Per-tag, `new-screen-wiring` (20 completed entries, recomputed 2026-10-20):
  actual active time averaged **0.46× the best estimate** (0.22 [2026-10-20], 0.95, 0.27, 1.12, 0.45, 0.27, 0.28, 0.97, 0.45, 0.48, 0.32, 0.36, 0.43, 0.38, 0.69, 0.28, 0.36, 0.25, 0.30, 0.41).
- Per-tag, `test-infra-setup` (11 completed entries, recomputed 2026-10-20): actual
  active time averaged **0.38× the best estimate** (0.22 [2026-10-20], 0.8, 0.27, 0.97, 0.45, 0.30, 0.41, 0.31, 0.25, 0.10, 0.19). The
  high ones were builds verified on real platforms (browser/emulator/macOS runs).
- Per-tag, `api-endpoint-addition` (23 completed entries, recomputed 2026-10-20):
  actual active time averaged **0.48× the best estimate** (0.22 [2026-10-20], 0.14, 0.16, 0.27, 0.28, 0.51, 0.19, 0.60, 0.48, 0.32, 0.36, 0.43, 0.38, 0.69, 0.28, 0.60, 0.25, 0.30, 0.41, 3.3 (scope-grown build; without it 0.35), 0.31, 0.25, 0.19;
  the 0.60, 0.32, 0.36, 0.38, 0.69, 0.25 and 0.31 were measured against an already tag-scaled best, i.e. ~0.17× / ~0.15× / ~0.16× / ~0.17× / ~0.32× / ~0.10× / ~0.14× their raw decompositions).
  Endpoints added onto the proven route/controller/service + test-harness seams
  have been overestimated every time; scale the decomposed best estimate by ~0.3.
- Per-tag, `codebase-documentation` (7 completed entries, recomputed 2026-10-20):
  actual active time averaged **0.20× the best estimate** (0.22 [2026-10-20], 0.16, 0.17, 0.31, 0.25, 0.10, 0.19). Guides/lesson
  plans written right after reading the code go in one pass; scale the decomposed best estimate by ~0.25.

## Manual invocation

If asked for an estimate or a retro outside of hackathon-mode/night-shift
(e.g. a plain "how long will this take"), run the same `estimate`/`retro`
logic above based on which one makes sense for where the task currently is.
