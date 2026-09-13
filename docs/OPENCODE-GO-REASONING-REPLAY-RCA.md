# OpenCode Go reasoning and WebSocket replay RCA

Status: fixed on `fix/opencode-go-reasoning-websocket-replay`, rebased onto
upstream `c1f6e69e`

Date investigated: 2026-09-13

Affected routes:

- `opencode-go/deepseek-v4.1-flash`
- `opencode-go/glm-5.3-flash`
- Other OpenCode Go models that return native Chat Completions reasoning use
  the same reasoning classification, but the live reproduction and endurance
  checks covered the two routes above.

Native OpenAI models were not affected.

## Symptom and impact

During a tool loop, Codex could display the same progress sentence dozens of
times. This was not only a rendering problem: the duplicate assistant messages
were present in the request history sent on the next turn, so the provider
processed them again. A captured task contained one assistant message repeated
49 times and reported roughly 1,436 output tokens and 10.9 million processed
tokens.

DeepSeek V4.1 Flash and GLM-5.3 Flash reproduced the problem through OpenCode
Go. A native `gpt-5.6-sol` control did not.

## Request path

```text
Codex Responses WebSocket
  -> codex-router
  -> LiteLLM Responses-to-Chat adapter
  -> API forwarder
  -> OpenCode Go Chat Completions
```

Responses travel back through the same components. The WebSocket edge retains
completed output items so Codex can send only the new tool result with
`previous_response_id`; the edge then reconstructs the next full request.

## Root cause

Two independent defects compounded.

### 1. OpenCode Go reasoning was not treated as native Chat reasoning

The API forwarder already restores Chat Completions `reasoning_content`, and
the router already has a compatibility transform that turns malformed orphan
reasoning-summary deltas into a valid Responses reasoning lifecycle. OpenCode
Go's DeepSeek and GLM routes were not opted into either contract.

Consequences:

- prior private reasoning could be translated toward visible message history;
- Codex received malformed `response.reasoning_summary_text.delta` events with
  unstable item IDs and no complete reasoning-item lifecycle;
- the model catalog claimed that the routes did not support reasoning
  summaries, even though the upstream returned reasoning.

Upstream merged a generalized native-reasoning contract in `c1f6e69e` while
this investigation was being documented. It supersedes the original
route-specific source patch: the contract now keys on measured upstream model
families and Chat Completions providers, and its protocol-scoped lifecycle
normalizer covers OpenCode Go. This branch is rebased on that implementation
instead of maintaining a competing special case.

### 2. WebSocket continuation reconciliation trusted message IDs too much

LiteLLM's Chat-to-Responses adapter can describe one assistant message twice
with different IDs:

```text
response.output_item.done       message id = msg_streamed
response.completed.output[0]    message id = msg_terminal
```

The content, role, position, and item type are identical. The terminal snapshot
only changes the ID.

The WebSocket continuation cache reconciled items only by `id` or `call_id`, so
it retained both messages. The next tool-result turn replayed both copies. Each
additional tool loop repeated the already duplicated history, producing
multiplicative growth.

The HTTP path was clean because it does not reconstruct continuation state.
The ordinary Codex WebSocket path reproduced the problem.

## Fix

### Reasoning preservation

- Upstream `c1f6e69e` classifies measured thinking-model families behind Chat
  Completions providers in `src/chat-reasoning.mjs` and applies the shared
  reasoning-summary lifecycle normalizer in `src/router.mjs`.
- The DeepSeek V4.1 Flash and GLM-5.3 Flash registry entries advertise
  `supportsReasoningSummaries: true`; their compatibility hashes are bumped so
  the generated catalog refreshes.

### Continuation reconciliation

`src/responses-websocket.mjs` still reconciles by stable key first. It adds a
strict positional fallback only when:

1. the terminal and streamed output arrays have the same length;
2. every item at the same position has the same type; and
3. non-message items with stable keys do not disagree.

For a compatible message position, the complete streamed item is authoritative
when the terminal snapshot only supplies a different message ID.

This is deliberately not content-based deduplication. Legitimate repeated text
must remain repeated, and tool calls must continue to match by their stable
`call_id`/item key.

## Safety boundaries

- Native OpenAI requests do not enter the OpenCode Go transforms.
- Other routed providers keep their existing reasoning policy.
- The positional fallback cannot join arrays with different shapes or reorder
  tool calls whose stable keys disagree.
- No router-authored text is injected into the transcript.
- No manual `config.toml` setting is required for this fix.
- No credential, caller capability URL, or provider response body is logged or
  committed.

## Regression coverage

The deterministic regression is:

```sh
node --test test/responses-websocket.test.mjs
```

The new case creates a streamed message and a terminal snapshot with different
message IDs, follows with a tool result, and asserts that the reconstructed
input contains exactly one assistant message.

Run the focused reasoning and routing checks with:

```sh
node --test \
  test/chat-reasoning.test.mjs \
  test/registry.test.mjs \
  test/responses-websocket.test.mjs \
  test/routing.test.mjs
npm run check
```

The offline LiteLLM bridge proof is enabled when a Python environment with the
repository's pinned LiteLLM is available:

```sh
MODEL_ROUTER_TEST_LITELLM_PYTHON=/path/to/python \
  node --test test/chat-reasoning.test.mjs
```

`npm test` ran 3,975 tests during the investigation: 3,947 passed, 27 skipped,
and the only failure was the desktop-panel test because the optional
`playwright` package was absent from this checkout. The focused tests and
`npm run check` passed.

Live Codex endurance checks completed without repetition:

| Route | Sequential tool turns | Input-token growth | Failures |
| --- | ---: | ---: | ---: |
| OpenCode Go DeepSeek V4.1 Flash | 12 | 46,075 -> 47,495 (+1,420) | 0 |
| OpenCode Go GLM-5.3 Flash | 12 | 44,582 -> 45,572 (+990) | 0 |
| Native `gpt-5.6-sol` control | 4 | linear | 0 |

The live checks spend provider quota and are not part of the default automated
suite.

## Updating from upstream without losing the fix

This installation checkout keeps the public repository as `origin` and the
personal fork as `fork`. Keep `main` as an unmodified mirror and carry the fix
only on its named branch. The branch was first written against `e8041d3`; the
documented rebase onto `c1f6e69e` removed the now-redundant route-specific
reasoning implementation while retaining the WebSocket fix and registry
metadata:

```sh
git fetch origin
git switch main
git merge --ff-only origin/main
git push fork main

git switch fix/opencode-go-reasoning-websocket-replay
git branch compare/opencode-replay-before-rebase
git rebase main
git range-diff \
  origin/main...compare/opencode-replay-before-rebase \
  origin/main...fix/opencode-go-reasoning-websocket-replay
```

Then run the focused tests and `npm run check` before updating the fork branch:

```sh
git diff --check
node --test test/responses-websocket.test.mjs
node --test test/chat-reasoning.test.mjs test/registry.test.mjs test/routing.test.mjs
npm run check
git push --force-with-lease fork fix/opencode-go-reasoning-websocket-replay
```

Use `--force-with-lease` only for the rebased fix branch. Never force-push
`main`. Delete the temporary comparison branch only after reviewing the
range-diff.

In a fresh clone of the personal fork, name the public repository `upstream`
and substitute `upstream/main` for `origin/main` in the commands above.

## Comparing behavior after an update

Before restarting the installed service, inspect exactly what remains local:

```sh
git log --left-right --cherry-pick --oneline origin/main...HEAD
git diff --stat origin/main...HEAD
git diff origin/main...HEAD -- \
  src/chat-reasoning.mjs \
  src/responses-websocket.mjs \
  src/router.mjs \
  config/opencode/go \
  test
```

After tests pass, restart and check the installation:

```sh
./bin/model-router codex update
./bin/model-router codex doctor
```

For a live canary, use a new task and alternate a short progress message with a
simple tool call for at least four turns. Confirm that each progress message
appears once, reasoning stays in Codex's reasoning UI, and input usage grows
roughly linearly. Run the same small canary with a native OpenAI model as the
control.

## Rollback

Because `main` stays clean, rollback does not require deleting work or resetting
the repository:

```sh
git switch main
./bin/model-router codex update
./bin/model-router codex doctor
```

To restore the patch, switch back to
`fix/opencode-go-reasoning-websocket-replay` and run the same update and doctor
commands. Fully quit and reopen Codex only when catalog metadata changed and a
picker refresh is required.

## Related reports

- [codex-router #292: OpenCode Go missing reasoning content](https://github.com/duolahypercho/codex-router/issues/292)
- [codex-router #653: replayed output and visible self-check text](https://github.com/duolahypercho/codex-router/issues/653)
- [OpenAI Codex #24500: DeepSeek reasoning content](https://github.com/openai/codex/issues/24500)
- [OpenAI Codex #15608: WebSocket duplicate seed context](https://github.com/openai/codex/issues/15608)
- [OpenAI Codex #42787: WebSocket response-chain state leak](https://github.com/openai/codex/issues/42787)
- [OpenCode #22329: DeepSeek tool-loop repetition](https://github.com/anomalyco/opencode/issues/22329)
- [OpenCode #35784: GLM read loop](https://github.com/anomalyco/opencode/issues/35784)

These reports cover neighboring failure modes. The exact two-defect chain in
this document was isolated with boundary captures at the API forwarder,
LiteLLM Responses output, and the router WebSocket continuation cache.
