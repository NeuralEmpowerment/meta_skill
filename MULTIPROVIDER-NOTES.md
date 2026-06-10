# Multi-provider coverage (Task B): IMPLEMENTED, UNVALIDATED

Authored by the orchestrator on the code agent's behalf (the agent hit an
upstream API rate limit and exited after writing the code but before committing
or documenting). The code is preserved and committed here.

## Status
Code added + compiles + NO Claude regression (a Claude query still extracts 12
patterns on this binary). NOT empirically validated for Codex/Gemini, because
the cass corpus on this box is 100% Claude Code sessions (60/60 in a top-60
search). Proving it needs a real Codex/Gemini session indexed into cass.

## What was added (src/cass/client.rs, +233 lines)
`translate_cass_record` now dispatches by provider record shape:
- Claude Code (existing path): `type: user|assistant` with a sibling `message`
  object; content is a string or block array (text/thinking/tool_use/tool_result).
- Codex (OpenAI, originator codex-tui): records wrap content in a `payload`
  object keyed by `type: session_meta|event_msg|response_item|turn_context|
  compacted`. Only `response_item` carries conversation; `payload.type ==
  "message"` -> `{role, content:[{type: input_text|output_text, text}]}`. Codex
  emits each piece as its own record (vs Claude bundling a turn). Implemented as
  `translate_codex_record`. This is schema-grounded, not guessed.

## Validation gap (the real limit)
The Codex branch is UNTESTED against real data (no Codex sessions in the index).
Before claiming Codex support: index a Codex rollout into cass and confirm
`ms build --from-cass ... --auto` yields patterns_extracted > 0. Gemini was not
separately implemented in this pass; its export shape still needs inspection.

## Upstream relevance (#114)
This is exactly the "scope of fix" caveat already noted in issue #114: a complete
fix branches on provider shape. The Codex branch here is a concrete proposal but
should be validated before it is offered upstream as proven.
