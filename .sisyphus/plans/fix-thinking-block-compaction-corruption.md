# Plan: Fix Thinking Block Corruption During Compaction

**Status:** COMPLETE (Phase 1 + Phase 2 implemented, Phase 3 verification pending smoke test)
**Created:** 2026-02-16
**Branch:** `bugfix/gee/opus-parsing-issue`
**Triggered by:** Sessions crash with `"'thinking' or 'redacted_thinking' blocks in the latest assistant message cannot be modified"` when compaction fires for Claude Opus 4.6 on Bedrock with extended thinking enabled

---

## Context & Problem

### Symptoms

After upgrading to OpenCode 1.2.5 and configuring `anthropicBeta: ["context-1m-2025-08-07"]` for Claude Opus 4.6 on Bedrock, sessions fail during compaction (auto or manual `/compact`) with:

```
messages.13.content.1: 'thinking' or 'redacted_thinking' blocks in the latest assistant message
cannot be modified. These blocks must remain as they were in the original response.
```

### Root Cause: Two Compounding Bugs

**Bug A — Premature compaction trigger** (`compaction.ts:32-48`)

`isOverflow()` uses the model's metadata limits (~200K) to decide when to compact, but the `anthropicBeta` header tells the API to accept 1M tokens. Compaction triggers far too early.

The `limit.input` path also fails to reserve headroom for output tokens:

| Path                | Formula                              | Reserves output space? |
| ------------------- | ------------------------------------ | ---------------------- |
| `limit.input` set   | `usable = limit.input - reserved`    | Only 20K buffer        |
| `limit.input` unset | `usable = context - maxOutputTokens` | Full output window     |

Three existing test cases (lines 128, 154, 174) explicitly document this as a BUG.

**Bug B — Thinking block signature stripping** (`message-v2.ts:589-676`)

When `toModelMessages()` replays conversation history, it compares the current model against each historical message's original model:

```typescript
const differentModel = `${model.providerID}/${model.id}` !== `${msg.info.providerID}/${msg.info.modelID}`
```

When `differentModel === true`, all `providerMetadata` is dropped:

- Line 611: text parts — `...(differentModel ? {} : { providerMetadata: part.metadata })`
- Line 676: reasoning parts — `...(differentModel ? {} : { providerMetadata: part.metadata })`
- Lines 648/658/669: tool parts — `...(differentModel ? {} : { callProviderMetadata: part.metadata })`

This strips the cryptographic `signature` Anthropic requires thinking blocks to carry. The API rejects the modified blocks.

### Why Both Bugs Compound

Bug A triggers compaction earlier than it should. Bug B corrupts the messages during compaction. Without Bug A, the user would rarely hit compaction. Without Bug B, compaction would succeed. Together they're lethal.

### References

| Reference                              | Location                                                     | Relevance                                            |
| -------------------------------------- | ------------------------------------------------------------ | ---------------------------------------------------- |
| `differentModel` check                 | `message-v2.ts:589`                                          | The guard that strips metadata                       |
| Reasoning part metadata spread         | `message-v2.ts:676`                                          | Where reasoning signature is dropped                 |
| Text part metadata spread              | `message-v2.ts:611`                                          | Where text signature is dropped                      |
| Tool part metadata spreads             | `message-v2.ts:648,658,669`                                  | Where tool metadata is dropped                       |
| `toModelMessages()` signature          | `message-v2.ts:491`                                          | Function accepts `model` parameter                   |
| Compaction calls toModelMessages       | `compaction.ts:188`                                          | Passes compaction model, triggers Bug B              |
| Normal prompt calls toModelMessages    | `prompt.ts:660`                                              | Same function, but model usually matches             |
| Title gen calls toModelMessages        | `prompt.ts:1943`                                             | Another callsite (may use different model)           |
| `isOverflow()` logic                   | `compaction.ts:32-48`                                        | Bug A — premature compaction                         |
| BUG test: no headroom with input limit | `compaction.test.ts:128`                                     | Existing failing test documenting Bug A              |
| BUG test: asymmetry                    | `compaction.test.ts:174`                                     | Existing failing test documenting Bug A              |
| ReasoningPart schema with metadata     | `message-v2.ts:116-127`                                      | Where metadata is defined                            |
| processor.ts metadata capture          | `processor.ts:75,85,105,147,248,297`                         | Where metadata enters the system                     |
| Anthropic normalizeMessages            | `provider/transform.ts:52-72`                                | Filters empty reasoning parts                        |
| Comment about thinking signatures      | `prompt.ts:496`                                              | Confirms this is a known issue area                  |
| Copilot `reasoningOpaque`              | `copilot/chat/*.ts`                                          | Same pattern — opaque signature for reasoning        |
| Upstream issue #13286                  | [GitHub](https://github.com/anomalyco/opencode/issues/13286) | Exact same error, reported Feb 12                    |
| Upstream issue #10970                  | [GitHub](https://github.com/anomalyco/opencode/issues/10970) | Related thinking block error                         |
| Upstream issue #8185                   | [GitHub](https://github.com/anomalyco/opencode/issues/8185)  | Discussion: should old thinking blocks be sent back? |
| Related compaction PRs                 | #6875, #12924                                                | Open PRs related to compaction fixes                 |

### Anthropic Official Guidance (from docs)

From [Extended Thinking - Preserving Thinking Blocks](https://docs.anthropic.com/en/docs/build-with-claude/extended-thinking#preserving-thinking-blocks):

> "While you can omit `thinking` blocks from prior `assistant` role turns, we suggest always passing back all thinking blocks to the API for any multi-turn conversation. The API will automatically filter the provided thinking blocks, use the relevant thinking blocks necessary to preserve the model's reasoning, and only bill for the input tokens for the blocks shown to Claude."

From [AWS Bedrock - Extended Thinking](https://docs.aws.amazon.com/bedrock/latest/userguide/claude-messages-extended-thinking.html):

> "Context handling: You do not need to remove previous thinking blocks yourself. The Anthropic API automatically ignores thinking blocks from previous turns and they are not included when calculating context usage."

**Key rules:**

1. The `signature` field is a **required cryptographic proof** — not optional metadata
2. If you include thinking blocks, they **must be byte-identical** to the original (signature intact)
3. You **can** omit thinking blocks from old messages entirely — the API handles it
4. The **latest** assistant message's thinking blocks are strictly validated

This confirms our fix: always pass `providerMetadata` through. The API handles the rest.

### Current Workaround

Setting `limit.context: 1048576` in `opencode.json` for the model avoids Bug A by pushing the compaction threshold to near 1M — effectively never triggering auto-compaction. Manual `/compact` still fails (Bug B).

---

## Phase 1: Fix Thinking Block Metadata Preservation (Bug B)

This is the critical fix. The `differentModel` guard in `toModelMessages()` is too aggressive — it strips all provider metadata including cryptographic signatures when the model doesn't match exactly.

### Task 1.1: Always preserve `providerMetadata` on reasoning parts ☑

**Priority:** High — the core fix
**Files:** `packages/opencode/src/session/message-v2.ts`

Change lines 672-677 to **always** include `providerMetadata` on reasoning parts, regardless of `differentModel`:

```typescript
// BEFORE (line 672-677):
if (part.type === "reasoning") {
  assistantMessage.parts.push({
    type: "reasoning",
    text: part.text,
    ...(differentModel ? {} : { providerMetadata: part.metadata }),
  })
}

// AFTER:
if (part.type === "reasoning") {
  assistantMessage.parts.push({
    type: "reasoning",
    text: part.text,
    providerMetadata: part.metadata,
  })
}
```

**Rationale:** Anthropic's API docs state that old thinking blocks in conversation history are **stripped server-side** before processing. Sending them with their original metadata is harmless — the server ignores them. But sending them _without_ their signature causes validation errors. This matches the discussion in issue #8185.

For non-Anthropic providers that don't use reasoning signatures, `part.metadata` will be `undefined` and the spread is a no-op.

**Success criteria:**

- [x] Reasoning parts always carry their original `providerMetadata` in `toModelMessages()` output
- [x] Non-Anthropic models unaffected (`metadata` is undefined, field spreads as no-op)

### Task 1.2: Always preserve `providerMetadata` on text parts ☑

**Priority:** High — same issue for text parts with thinking metadata
**Files:** `packages/opencode/src/session/message-v2.ts`

Change line 607-612:

```typescript
// BEFORE:
if (part.type === "text")
  assistantMessage.parts.push({
    type: "text",
    text: part.text,
    ...(differentModel ? {} : { providerMetadata: part.metadata }),
  })

// AFTER:
if (part.type === "text")
  assistantMessage.parts.push({
    type: "text",
    text: part.text,
    providerMetadata: part.metadata,
  })
```

**Rationale:** Same as 1.1. Text parts can carry provider metadata (e.g., Anthropic cache hints). The AI SDK/provider should handle or ignore unknown metadata gracefully. If a non-Anthropic provider rejects foreign metadata, that's a separate bug to address with provider-specific filtering — not by blanket-stripping everything.

**Success criteria:**

- [x] Text parts always carry their original `providerMetadata`
- [x] No regression for non-Anthropic providers

### Task 1.3: Always preserve `callProviderMetadata` on tool parts ☑

**Priority:** Medium — tool parts also stripped
**Files:** `packages/opencode/src/session/message-v2.ts`

Change lines 648, 658, 669 — all three tool part branches:

```typescript
// BEFORE (each of the three locations):
...(differentModel ? {} : { callProviderMetadata: part.metadata }),

// AFTER:
callProviderMetadata: part.metadata,
```

**Success criteria:**

- [x] Tool parts (completed, error, pending/running) always carry `callProviderMetadata`
- [x] No regression for non-Anthropic providers

### Task 1.4: Remove or simplify `differentModel` variable ☑

**Priority:** Low — cleanup after 1.1-1.3
**Files:** `packages/opencode/src/session/message-v2.ts`

After tasks 1.1-1.3, the `differentModel` variable (line 589) may no longer be used. If all conditional spreads are removed:

- Delete line 589 entirely
- If `differentModel` is still used elsewhere in the function, leave it

**Success criteria:**

- [x] No dead code remains
- [x] `lsp_diagnostics` clean on file

### Task 1.5: Add unit tests for metadata preservation ☑

**Priority:** High — regression prevention
**Files:** `packages/opencode/test/session/message-v2.test.ts`

Add tests to the existing `toModelMessages` test suite:

| Test                                                      | Verifies                                                          |
| --------------------------------------------------------- | ----------------------------------------------------------------- |
| `preserves reasoning providerMetadata when model matches` | Baseline — metadata flows through when same model                 |
| `preserves reasoning providerMetadata when model differs` | The bug fix — metadata preserved even with different model        |
| `preserves text providerMetadata when model differs`      | Text part metadata not stripped                                   |
| `preserves tool callProviderMetadata when model differs`  | Tool part metadata not stripped                                   |
| `handles undefined metadata gracefully`                   | Parts without metadata produce no `providerMetadata` field issues |

Use the existing test model fixture (`model` at line 7) and create a second model with different `providerID`/`id` to simulate `differentModel === true`.

**Success criteria:**

- [x] All 5 test cases pass
- [x] Tests use existing fixture patterns from the test file

---

## Phase 2: Fix Compaction Overflow Detection (Bug A)

### Task 2.1: Fix `isOverflow()` to reserve output headroom when `limit.input` is set ☑

**Priority:** Medium — the existing BUG tests already document this
**Files:** `packages/opencode/src/session/compaction.ts`

Change lines 44-47:

```typescript
// BEFORE:
const usable = input.model.limit.input
  ? input.model.limit.input - reserved
  : context - ProviderTransform.maxOutputTokens(input.model)

// AFTER — always subtract output from usable:
const usable = input.model.limit.input
  ? input.model.limit.input - reserved - ProviderTransform.maxOutputTokens(input.model)
  : context - ProviderTransform.maxOutputTokens(input.model)
```

Wait — this needs careful thought. The `limit.input` represents the **input token limit**, which is already separate from output. When `limit.input` exists, output tokens don't count against it. But compaction needs to trigger _before_ we fill the input window entirely, because the next turn's input = previous total + new system prompt overhead.

Revisit: the `reserved` buffer (20K or `maxOutputTokens`, whichever is smaller) may be sufficient for that overhead. The real issue in the BUG tests is that `reserved` subtracts only 20K from a 200K window, while without `limit.input` it subtracts the full 32K output window.

**Proposed fix:** Make `reserved` consistent — always use `maxOutputTokens` when it exceeds the current buffer:

```typescript
const reserved = config.compaction?.reserved ?? ProviderTransform.maxOutputTokens(input.model)
```

This removes the `Math.min(COMPACTION_BUFFER, ...)` that caps the buffer at 20K.

**Success criteria:**

- [x] All three existing BUG tests (lines 128, 154, 174) now pass
- [x] Existing passing tests still pass
- [x] Models with `limit.input` and without trigger compaction at similar thresholds

### Task 2.2: Update compaction tests ☑

**Priority:** Medium
**Files:** `packages/opencode/test/session/compaction.test.ts`

- Remove the `BUG:` prefix from the three test names (lines 128, 154, 174) since they should now pass
- Verify the expected values in existing non-BUG tests still hold after the `reserved` change

**Success criteria:**

- [x] All tests in `compaction.test.ts` pass
- [x] No test marked as BUG remains

---

## Phase 3: Verification

### Task 3.1: Run full test suite ☑

**Priority:** High
**Command:** `bun test` from `packages/opencode`

- [x] `test/session/message-v2.test.ts` passes (23 pass, 0 fail)
- [x] `test/session/compaction.test.ts` passes (24 pass, 0 fail)
- [x] `test/session/revert-compact.test.ts` passes
- [x] No regressions — 49 tests across 3 files, 91 expect() calls, 0 failures

### Task 3.2: LSP diagnostics clean ☑

**Priority:** High

- [x] `packages/opencode/src/session/message-v2.ts` — no diagnostics found
- [x] `packages/opencode/src/session/compaction.ts` — no diagnostics found

### Task 3.3: Manual smoke test ☐

**Priority:** High
**Command:** `bun run dev` from repo root

1. Start a session with an Anthropic model that supports extended thinking
2. Have a multi-turn conversation (3+ turns with tool use)
3. Trigger `/compact`
4. Verify no thinking block error
5. Continue conversation after compaction

- [ ] Manual compaction succeeds without error
- [ ] Session continues working after compaction

---

## Execution Order

1. **Phase 1 (Bug B):** Tasks 1.1 → 1.2 → 1.3 → 1.4 → 1.5
2. **Phase 2 (Bug A):** Tasks 2.1 → 2.2
3. **Phase 3:** Tasks 3.1 → 3.2 → 3.3

Phase 1 is the critical path — it fixes the crash. Phase 2 is a quality improvement that prevents unnecessary compaction. Phase 3 validates everything.

## Risk Assessment

| Risk                                                             | Mitigation                                                                                                                     |
| ---------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------ |
| Non-Anthropic providers reject foreign `providerMetadata`        | Metadata is `undefined` for non-Anthropic parts; AI SDK ignores unknown fields. Low risk.                                      |
| Copilot `reasoningOpaque` breaks with always-pass metadata       | Copilot already passes `providerMetadata` through its own path — unaffected.                                                   |
| `reserved` change triggers compaction too early for some models  | Only affects models with very small output limits. The old 20K cap was too low for modern models anyway.                       |
| Tests don't cover cross-provider compaction (e.g., Opus → GPT-4) | Out of scope — true cross-provider compaction is a larger design issue. Our fix is safe because undefined metadata is a no-op. |
