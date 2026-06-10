# Multi-provider session coverage (translate_cass_record)

Status as of 2026-06-10. Covers what each agent provider's `cass export
--format json` records look like and how `ms` decodes them into the flat
`SessionMessage` the miner consumes.

## How it works

`CassClient::get_session` runs `cass export <path> --format json` and maps each
record through `translate_cass_record`. cass returns the provider's RAW JSONL
(it does NOT normalize across providers), so `translate_cass_record` must
detect and decode each provider's native shape. Dispatch is by record shape:

- A record with a `payload` object -> Codex shape -> `translate_codex_record`.
- Otherwise -> Claude Code shape (`type: user|assistant` + sibling `message`).

## Claude Code (`~/.claude/projects/**`, agent=claude_code)

- Conversation turns: `{type: "user"|"assistant", message: {role, content}}`.
- `content` is a string OR an array of typed blocks: `text`, `thinking`
  (dropped), `tool_use` (-> ToolCall), `tool_result` (-> ToolResult).
- This was the original / only supported shape (wall-1).
- Corpus: 191 of ~200 sampled hits. The dominant provider on this box.

## Codex (`~/.codex/sessions/YYYY/MM/DD/rollout-*.jsonl`, agent=codex)

- DIFFERENT shape. Every line is `{timestamp, type, payload}` where top-level
  `type` is one of `session_meta`, `event_msg`, `turn_context`, `compacted`,
  `response_item`. Only `response_item` carries conversation content.
- `payload.type` within `response_item`:
  - `message`: `{role, content: [{type: "input_text"|"output_text", text}]}`.
    Roles seen: assistant (output_text), user/developer (input_text).
  - `function_call`: `{name, arguments (JSON STRING), call_id}` -> ToolCall.
    `arguments` is a JSON-encoded string; we parse it to structured JSON when
    possible, else keep the raw string.
  - `custom_tool_call`: `{name, input, call_id}` (e.g. `apply_patch`) ->
    ToolCall. `input` is often non-JSON (patch text); preserved as a string.
  - `function_call_output` / `custom_tool_call_output` / `tool_search_output`:
    `{call_id, output}` -> ToolResult (role "tool").
  - `reasoning`: chain-of-thought, DROPPED (parity with Claude `thinking`).
  - `tool_search_call`: no skill-relevant content, dropped.
- Unlike Claude (one turn bundles text + tool blocks), Codex emits each piece
  as its own top-level record, so each maps to its own SessionMessage.
- Before this change `translate_cass_record` returned None for every Codex
  record (it only matched `type: user|assistant`), so Codex sessions yielded
  ZERO messages and ZERO mined patterns. This was a real, silent coverage gap.
- Corpus: 9 of ~200 sampled cass hits; 62 rollout files on disk. The two
  cass-indexed Codex sessions happen to be short/aborted ("test 1 2",
  interrupted turns), so the `build --from-cass` search does not reliably pick
  them for a focused query. Decoder coverage is therefore proven by unit tests
  against real-shape records (see `src/cass/client.rs` tests:
  test_translate_codex_message, _function_call_parses_arguments,
  _function_call_output, _custom_tool_call_apply_patch, _reasoning_dropped),
  plus a Claude regression test, all green.

## Gemini (`~/.gemini/**`)

- NOT PRESENT in the cass corpus. `cass search` returns zero hits with
  agent=gemini; the agent-field distribution over 200 sampled hits is
  191 claude_code + 9 codex, 0 gemini.
- On disk, `~/.gemini` contains settings backups plus chat transcripts under
  `~/.gemini/tmp/<project>/chats/session-*.jsonl`. These exist but are NOT
  indexed by cass on this box, so `cass export` cannot surface them through
  the normal `ms build --from-cass` path.
- Coverage decision: NOT extended. Decoding a shape that cass cannot feed us
  would be speculative (we could not prove it against the live pipeline). This
  is a real, documented coverage limit. When/if cass indexes Gemini sessions,
  the same shape-dispatch pattern in `translate_cass_record` is the place to
  add a `translate_gemini_record` branch; capture a real
  `~/.gemini/tmp/*/chats/session-*.jsonl` record shape first, then decode.

## Summary

| Provider | cass-indexed | Decoder | Proof |
|----------|--------------|---------|-------|
| Claude Code | yes (dominant) | wall-1 | regression test + live builds |
| Codex | yes (sparse, short) | added (this change) | 5 unit tests, real shapes |
| Gemini | no | not added (documented) | n/a - cass has no hits |
