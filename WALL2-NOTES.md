# WALL 2: auto-synthesis of SKILL.md from `ms build --from-cass --auto`

## Verdict: FEASIBLE and WIRED.

`ms build --from-cass <q> --auto` previously extracted patterns and wrote
`patterns.json` + `build-manifest.json`, then returned without ever producing a
`SKILL.md`. The skill compiler `generate_skill_md(draft: &BrennerSkillDraft)`
(src/cass/brenner.rs) is a pure function and was only reachable from the guided
wizard path. This is now fixed: `run_auto` synthesizes a draft and emits SKILL.md.

## Why non-interactive synthesis is possible

The guided wizard only needs a human for one thing: typing the cognitive-move
tag + a free-text description per move (the `a [tag] [description]` REPL command
in run_interactive). There is NO LLM in the loop; `build_skill_draft` itself is a
pure map from `CognitiveMove` -> `SkillRule`.

The patterns that `--auto` already extracts (`ExtractedPattern`) carry strictly
more structured data than a hand-typed move: `pattern_type`, `evidence[]` with
real session snippets, `confidence`, `frequency`, `description`, `tags`. So
instead of fabricating cognitive moves, `run_auto` now constructs a
`BrennerSkillDraft` DIRECTLY from `filtered_patterns` via a new pure helper
`synthesize_draft_from_patterns(name, query, &[ExtractedPattern])`, then feeds it
to the existing `generate_skill_md`. No interactive or LLM input is required.

## The patch (minimal, focused; src/cli/commands/build.rs)

1. New helper `synthesize_draft_from_patterns`: maps each `ExtractedPattern` to a
   `SkillRule { id, description, evidence, confidence }`. Description falls back to
   a `pattern_type`-derived label when `pattern.description` is None; evidence is
   up to 3 non-empty `EvidenceRef.snippet`s. Adds a calibration note recording
   that no human review of cognitive moves occurred. Robust to missing fields, no
   panics (all `unwrap_or_else` / `filter_map`).
2. In `run_auto`, after writing `build-manifest.json` and before completing:
   build the draft from `filtered_patterns`, call `generate_skill_md`, write
   `SKILL.md` into `output_dir`. Skill name = `args.name` or the query.
3. Add `skill_path` to the machine-readable JSON output.

Existing `patterns.json` / `build-manifest.json` output is untouched. The struct
fields used are the real ones (verified against BrennerSkillDraft, SkillRule,
ExtractedPattern, EvidenceRef, PatternType definitions; none invented).

## Proof (binary at /data/tmp/cargo-target/release/ms)

Query: "agent mail lock coordination", --name wall2.

BEFORE: `ms list` -> 1 skill (am-lock-safety). No SKILL.md emitted by --auto.

AFTER: build completes, 13 patterns extracted, JSON now includes
  "skill_path": ".../builds/agentmaillockcoordination/SKILL.md"
A real SKILL.md (3410 bytes, 13 rules) is written, with confidence scores and
real mined evidence (e.g. `am agent`, `br ready` command sequences, error
excerpts) plus the calibration note. Content is substantive mined material, not
a stub.

Note on registration: like the guided flow, `build --auto` writes SKILL.md to the
build output dir; it does NOT auto-register the skill into the `ms` store
(`ms list` still shows am-lock-safety only). Registration/import is a separate
step, out of scope for "make --auto synthesize a skill." The pre-existing
am-lock-safety skill is unchanged; wall2 is a fresh distinct artifact.

## Build

`CARGO_TARGET_DIR=/data/tmp/cargo-target cargo +nightly build --release` -> clean,
9m02s, no warnings introduced by the patch.
