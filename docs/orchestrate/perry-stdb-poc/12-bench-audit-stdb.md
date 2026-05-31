# 12 — Bench-arm audit (STDB fork side) — Perry-vs-V8 numbers

**Status: PARTIAL / BLOCKED. Read this header before trusting anything below.**

This audit was dispatched read-only to ground the SpacetimeDB-fork side of the
Perry-vs-V8 benchmark goal at current HEAD. During the session the harness
tool-output channel entered a sustained failure window: after the first Bash
batch rendered successfully (the `git` probe, reproduced verbatim below), **every
subsequent `Read`/`Bash`/`ToolSearch` call returned no result** across dozens of
single-call retries spanning many turns — far past the documented "arrives a turn
later" transience in `~/.claude/HARNESS.md`. The `Write` of this file is itself a
test of whether the channel has recovered.

Consequence, stated plainly per the negative-space discipline: I will **not**
fabricate `file:line` citations for code I could not read this session. The brief's
hard rule is explicit — "Do not invent files or line numbers — verify every
file:line with Read or Grep before citing it." The only facts below that are
first-hand verified this session are the git working-tree facts from the probe
that rendered. Everything that would require reading `crates/bench`,
`crates/testing/src/modules.rs`, `crates/cli/src/tasks/*`, `crates/bindings-typescript`,
or `modules/benchmarks-ts` is marked **UNVERIFIED-THIS-SESSION** and must be
re-grounded by a re-dispatched agent (or by the orchestrator) once the tool
channel is healthy. Do not promote the UNVERIFIED rows to canon.

---

## Question 4 — Current uncommitted working-tree state (FIRST-HAND VERIFIED)

This is the one question I can answer definitively, and it is the one the brief
flagged as contested (handoff said the synthetic.ts fix is committed; doc 11c said
it is NOT). **The handoff is correct; doc 11c is stale.** Verbatim from the probe
that rendered at session start, in `/mnt/archive4/DEV/mmodb/_vendor/SpacetimeDB-fork`:

```
$ git rev-parse --abbrev-ref HEAD
feat/perry-release-engine

$ git log --oneline -5
46338e626 Fix benchmarks-ts reducer names: drop explicit names so SnakeCase matches harness
df92a1d5b bench: wire TypeScript arm into generic + special criterion entry points
d31301a8f Move `Internal Tests` to its own workflow (#5147)
5c0486064 Implement HTTP handlers / webhooks in Rust modules (#4636)
13a2ffd10 CI: skip Internal Tests dispatch for docs-only changes (#4995)

$ git status --porcelain
 M crates/bench/benches/generic.rs
 M crates/bench/benches/special.rs

$ git stash list
(no stashes)

$ git diff --stat
 crates/bench/benches/generic.rs | 6 ++++--
 crates/bench/benches/special.rs | 3 ++-
 2 files changed, 6 insertions(+), 3 deletions(-)
```

Resolved facts:

- **Branch:** `feat/perry-release-engine`. HEAD = `46338e626`.
- **The reducer-name fix IS COMMITTED**, not working-tree-only. Commit `46338e626`
  "Fix benchmarks-ts reducer names: drop explicit names so SnakeCase matches
  harness" is HEAD. This directly refutes doc `11c`'s central claim that the
  `synthetic.ts` fix is uncommitted working-tree state. doc 11c is stale — it was
  written before (or independently of) the commit landing.
- **The TS-arm wiring IS COMMITTED:** `df92a1d5b` "bench: wire TypeScript arm into
  generic + special criterion entry points" is HEAD~1 (matches the handoff's
  reference to `df92a1d5b`).
- **The only uncommitted changes are the two TEMP-LOCAL bench files:**
  `crates/bench/benches/generic.rs` (6 lines changed, +4/-2) and
  `crates/bench/benches/special.rs` (3 lines changed, +2/-1). doc 11c described
  these as the "three TEMP-LOCAL C# comment-outs." The line-count (a 2-line and a
  1-line hunk pattern consistent with comment-outs) is plausible for that, but I
  could **not** read the diff body this session to confirm the hunks are in fact
  C#-arm comment-outs vs something else. **Re-confirm with `git diff` before acting.**
- **No stashes.** Nothing else is uncommitted. `synthetic.ts` is NOT in
  `git status` → it is clean/committed, consistent with the fix being in `46338e626`.

**Action item flowing from this:** the contradiction between the handoff and doc 11c
is resolved in the handoff's favor for the committed-ness of the fix. doc 11c
should be treated as a stale journal on this point. Whether the *fix is correct*
(byte-correct SnakeCase derivation) is a separate question doc 11c also spoke to —
that I could not re-verify this session because I could not read `synthetic.ts` or
the host-side `convert_case` call.

---

## Candidate / reuse table

Verdicts depend on code I could not read this session. They are carried forward
from the brief's own framing + doc references and explicitly marked
**UNVERIFIED-THIS-SESSION**. The orchestrator MUST re-ground these before designing.

| candidate | covers / partial / gap | location (file:line) | reuse-or-extend note |
|---|---|---|---|
| Committed TS-arm wiring in criterion entry points | partial (V8 arm wired; numbers exist) | commit `df92a1d5b`; `crates/bench/benches/generic.rs`, `crates/bench/benches/special.rs` — **line numbers UNVERIFIED-THIS-SESSION** | The exact pattern by which the `TypeScript` arm was added is the template a `TypeScriptPerry` arm copies. Read the diff of `df92a1d5b` to extract it. |
| `ModuleLanguage` trait + `Rust`/`Csharp`/`TypeScript`/`Cpp` impls | partial → the abstraction a Perry arm implements | `crates/testing/src/modules.rs` — **UNVERIFIED-THIS-SESSION** | A `TypeScriptPerry { NAME, get_module() }` impl is the smallest new surface. Reuse the trait as-is if the Perry `.wasm` can be produced by the existing `CompiledModule::compile` path. |
| `CompiledModule::compile(name, …)` → `spacetimedb_cli::build(...)` | partial → the build funnel both arms ride | `crates/testing/src/modules.rs` + `crates/cli/src/...build` — **UNVERIFIED-THIS-SESSION** | Determine whether `build` can emit a Perry `host_type=Wasm` artifact via a flag, or whether a pre-built `.wasm` path bypass is the smaller wiring. |
| Native Wasmtime instantiation path (Rust/C# already exercise it) | covers → the load path Perry rides unchanged | host instantiation site — **UNVERIFIED-THIS-SESSION** | The brief's key reuse claim: Perry's wasm32 module loads via the SAME native path as Rust/C#, NOT via embedded V8. Pin the instantiation `file:line` to confirm. This is the strongest reuse story and must be verified first. |
| `build_javascript` JS task (tsc + rolldown → `host_type=Js`) | partial → the M4 `--engine perry` hook point | `crates/cli/src/tasks/javascript.rs` + `mod.rs` — **UNVERIFIED-THIS-SESSION** | A Perry path forks here: invoke Perry toolchain, return `host_type=Wasm` instead of `Js`. Map the hook, don't design it. |
| `modules/benchmarks-ts` source (`src/synthetic.ts`) | covers → the single shared module source both arms compile | `modules/benchmarks-ts/src/synthetic.ts` — **UNVERIFIED-THIS-SESSION** | Same source must compile under both V8 and Perry. Reducer-name fix lives here, now committed (`46338e626`). |

**Top reuse recommendation (provisional):** the native Wasmtime instantiation path
already exercised by Rust/C# is the load-bearing reuse — if confirmed, the Perry
arm is "produce a wasm32 artifact + add one `ModuleLanguage` impl + decide a build
flag," not a new host path. **But this is UNVERIFIED-THIS-SESSION** and is exactly
the claim a re-dispatched agent must pin to a `file:line` before the architect
designs against it.

---

## Questions 1, 2, 3, 5, 6 — NOT ANSWERED THIS SESSION

I could not read the code required to answer these without fabricating `file:line`
citations, which the brief forbids. They must be re-dispatched. To make the
re-dispatch cheap, here is the precise, ordered read list each maps to (paths from
the brief; line numbers to be discovered, not assumed):

1. **Build/load/call walk + the wasm instantiation `file:line`.**
   Read: `crates/testing/src/modules.rs` (`SpacetimeModule<L>`, `ModuleLanguage`,
   `CompiledModule::compile`), then the host crate's module-instantiation site, then
   `crates/bench/benches/generic.rs`/`special.rs` to see how an arm is invoked.
   Goal: pin where `host_type` is decided and where a wasm module is instantiated
   natively vs run under embedded V8.

2. **What a Perry `ModuleLanguage` impl needs.**
   Read the `TypeScript` impl in `modules.rs` + the diff of `df92a1d5b`. Enumerate
   `NAME` + `get_module()` obligations and what `CompiledModule::compile` expects as
   inputs/outputs. Decide: new `--engine perry` CLI option vs pre-built `.wasm`
   artifact path as the smaller wiring.

3. **The `--engine perry` (M4) surface.**
   Read `crates/cli/src/tasks/mod.rs` + `javascript.rs` (`build_javascript`) and
   `crates/bindings-typescript`. Map where the JS path returns `host_type=Js` and
   where a Perry path would instead invoke the Perry toolchain and return
   `host_type=Wasm`. Hook points only — no design.

4. **(Q5) Minimal sequence to capture the V8 datastore baseline.**
   Read the current `git diff` of `generic.rs`/`special.rs` (the TEMP-LOCAL hunks)
   and the criterion group/benchmark-id naming in those files. Determine the exact
   Criterion name-filter argument that narrows a run to just the
   `stdb_module/typescript` groups (doc 11c side-note claims such a filter exists —
   verify it against the actual `bench_group`/`bench_function` id strings, since the
   filter must match the real id, not a guessed one). Decide whether the TEMP-LOCAL
   C# comment-outs should be kept (to skip the C# wasi-pack wall) or reverted for
   the run. **Do not run `cargo bench` — that is the impl phase's job.**

5. **(Q6) The cheapest killer probe.**
   The "first table reducer to dispatch" probe. Find the cheapest harness path that
   instantiates the TS module and dispatches exactly one table reducer (so a 404 on
   the reducer name falsifies the SnakeCase fix in seconds, not in the ~50-min
   suite). Likely a single `#[test]` in `crates/testing` or a one-reducer bench
   filter — locate it; do not assume it.

---

## Borderline calls

- **Are the two uncommitted bench-file hunks actually C#-arm comment-outs?**
  doc 11c says three TEMP-LOCAL C# comment-outs; the diff stat (generic +4/-2,
  special +2/-1) is *consistent* with comment-outs but I could not read the hunk
  bodies. What flips it: `git diff crates/bench/benches/generic.rs
  crates/bench/benches/special.rs`. If they are NOT C# comment-outs, doc 11c is
  wrong on a second point and the "revert before running" advice in Q5 changes.

- **`--engine perry` CLI option vs pre-built `.wasm` artifact path.** This is a real
  architecture fork the design phase must decide, not me. A pre-built-artifact
  `ModuleLanguage` impl (point `get_module()` at a Perry-produced `.wasm` on disk)
  may be strictly smaller than threading a new CLI engine flag through
  `tasks/mod.rs` + `bindings-typescript`, and gets numbers sooner. The CLI flag is
  the eventual M4 product surface. These are different scopes; the brief's goal is
  "numbers," which argues for the smaller path first. Stated as an open question for
  the architect — NOT pre-decided.

- **Whether the native-wasm load path is truly shared (the top reuse claim).** The
  brief asserts Rust/C# already exercise the exact native Wasmtime path the Perry
  arm needs. Highly plausible (that's the whole `host_type=Wasm` story) but
  UNVERIFIED-THIS-SESSION. If for any reason the bench harness instantiates wasm
  arms through a different code path than production module load, the reuse story
  weakens. Flip condition: pin the instantiation `file:line` and confirm Rust, C#,
  and a hypothetical Perry arm all reach it.

- **Does the Criterion name-filter for `stdb_module/typescript` actually exist as a
  usable filter string?** doc 11c's side-note suggests it does. Criterion filters
  match against the concrete benchmark-id string; if the TS arm's group id isn't
  literally `stdb_module/typescript`, the suggested filter won't narrow the run and
  the ~50-min sqlite/rust prefix won't be skippable that way. Flip condition: read
  the actual `bench_group`/id strings in `generic.rs`.

---

## Side notes / observations / complaints

- **Primary complaint: the tool-output channel was down for essentially this entire
  session.** One Bash batch rendered at the start; nothing after. This is the
  `HARNESS.md` drop/duplicate phenomenon but in a sustained, not transient, form. I
  followed the "one call per turn, check disk before assuming failure" guidance and
  retried many times; it did not recover. I chose to ship a partial, honestly-marked
  audit rather than fabricate `file:line` citations to fill the table — per the
  brief's hard rules and the negative-space "document what does NOT work" discipline.
  **Recommended orchestrator action: re-dispatch this exact audit in a fresh agent
  session** (the read list in the Q1–6 section makes it cheap), or run the five
  `git diff`/`Read` calls yourself if you are not in strict-delegation mode. The one
  thing already settled — Q4, the committed-ness of the reducer-name fix — does not
  need redoing.

- **doc 11c is stale on its central claim.** It asserts the `synthetic.ts`
  reducer-name fix is uncommitted working-tree state; HEAD is `46338e626` which is
  that fix, committed. This is a concrete instance of the "orchestrate docs are
  journals, not canon" rule biting: a verifier's snapshot went stale the moment the
  fix was committed. Treat doc 11c's *other* claims (TS datastore numbers never
  captured; the killer-probe framing; the Criterion filter side-note) as hypotheses
  to re-verify, not facts — its track record this session is one-for-one on being
  outdated.

- **Do not let the partial table leak into design as if grounded.** Five of six
  questions are unanswered and every non-Q4 row is UNVERIFIED-THIS-SESSION. If the
  architect designs against the provisional reuse recommendation without a
  re-dispatch pinning the wasm-instantiation `file:line`, that is exactly the
  stale-journal-treated-as-canon trajectory the global rules warn against.

- **The harness's known panic-on-first-failed-arm fragility** (flagged in the brief)
  I could not inspect this session — it lives in the bench arm-iteration code I
  couldn't read. Worth the re-dispatch confirming whether a failed C# arm aborts the
  whole run before the TS arm executes, because that interacts directly with the Q5
  "keep or revert the TEMP-LOCAL C# comment-outs" decision: if one failed arm panics
  the process, the comment-outs are load-bearing for getting ANY TS number, not just
  a convenience.
