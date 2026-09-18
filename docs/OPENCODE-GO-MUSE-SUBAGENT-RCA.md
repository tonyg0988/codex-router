# OpenCode Go Muse subagent `agent_message` compatibility

## Symptom

Codex could select `opencode-go-responses/muse-spark-1.3-contributor` for a
subagent, but the child failed before producing output. Console Go returned an
HTTP 400 such as:

```text
input[N] did not match any supported type
```

The same model worked for ordinary top-level prompts, and DeepSeek subagents
worked through the same router installation.

## Root cause

Codex represents collaboration handoffs with a private Responses item named
`agent_message`. The router already recovered the readable task payload from
`encrypted_content`, but the paid OpenCode Go Responses compatibility path
forwarded the enclosing private item unchanged.

Console Go implements the public Responses schema. It accepts the equivalent
`message` item with `role: "user"`, but it does not accept Codex's private
`agent_message` discriminator.

A controlled request against the same Muse route established the boundary:

- `agent_message` produced HTTP 400.
- The equivalent public user `message` produced HTTP 200.

## Fix

The existing `agentMessagesAsUserMessages()` adapter is now applied only when
`needsConsoleGoResponsesToolCompatibility(route)` is true:

- on ordinary routed turns, after image bridging and before reasoning/history
  transformations;
- on routed compaction, before strict OpenCode history conversion.

The adapter keeps the recovered content and changes only the private envelope
to the public Responses shape. Native OpenAI traffic, OpenCode Go Chat,
OpenCode Go Messages, paid Zen, and other Free routes do not enter this branch.

## Regression coverage

`test/namespace-relay-routing.test.mjs` sends a realistic collaboration
handoff containing a visible `NEW_TASK` header and a readable payload through
the real router fixture. It checks both streaming modes and compaction:

- no provider-bound `agent_message` remains;
- the public user message retains the task payload;
- the existing Console Go tool compatibility contract remains intact.

Existing inverse-boundary tests continue to assert that native OpenAI and
non-target providers retain their original collaboration envelopes.

## Live verification

After restarting only `codex-router.service`, Codex spawned both routed child
models through its native collaboration API:

- Muse Spark 1.3 Contributor returned the requested marker.
- DeepSeek V4.1 Flash returned the requested marker.

No standalone thread or simulated agent endpoint was used for this check.

## Upgrade and rollback notes

The repaired branch is based on the current upstream `main`, so it also
contains the upstream Union Alpha v4 limits (`autoCompact: 180000` and
`maxOutputTokens: 32768`) that replace the earlier compaction-looping v1
configuration.

The pre-integration branch and stashes were retained locally as recovery
points. The older positional WebSocket replay patch was deliberately not
carried onto the installed branch because it operated on a transport shared by
native and routed traffic. Keeping it out makes the native OpenAI boundary
smaller and easier to reason about.
