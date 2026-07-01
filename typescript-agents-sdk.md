# Loan-Origination Agent — Design Decisions

This document captures the macro-level architectural and product decisions behind
`@clutch-libs/ai-sdk` (the in-house TypeScript wrapper for Anthropic Managed
Agents) and the first production agent built on top of it, `LoanOrigination`.
It is intentionally code-free — it explains _why_ each piece exists, what it
costs us, and what we have committed to. Refer to the actual modules for the
"how".

Key code locations referenced by name (not by line) throughout this document:

- SDK package: `platform/libs/ai-sdk/`
- HAL agent package: `agents/loan-origination/`
- Evals: `agents/loan-origination/evals/`
- Source-of-truth scenarios (legacy Python): `hal/v2/eval/scenarios.yaml` (in
  the `hal-assistant` repository)

---

## 1. Scope

There are three concentric concerns:

1. **The SDK** (`@clutch-libs/ai-sdk`) — a hexagonal TypeScript library that
   abstracts the Anthropic Managed Agents (CMA) HTTP API behind a small
   composition root and a typed domain.
2. **The HAL Loan-Origination Agent** (`agents/loan-origination/`) — the first
   real agent we ship on top of the SDK. Its scope is SMS-based conversation
   with loan applicants throughout the lending lifecycle.
3. **The evals** (`agents/loan-origination/evals/`) — a promptfoo-driven
   behavioral test suite. Every meaningful agent decision (intro, escalation,
   silence, doc handling, lifecycle event) is covered by at least one row.

This document treats all three as one coherent system; trade-offs in any layer
ripple to the others.

---

## 2. Why Managed Agents over the classic Messages API

The Anthropic Messages API would have required us to own:

- session state
- a memory abstraction
- tool dispatch & resource mounting
- multi-agent coordination

Anthropic's Managed Agents (CMA) provides these as primitives. We chose CMA
because:

- The team's prior implementation in Python (`pydantic-ai` + raw Messages API)
  carried a lot of glue code we did not want to re-author in TypeScript.
- CMA's first-class **memory store + session resources** map cleanly onto our
  "per-applicant, per-application" data model.
- CMA's tool surface (custom, MCP, builtin toolsets) lets us pick the cheapest
  integration shape per concern (e.g., stub locally, MCP later).
- Versioned/content-hashed deploys are native to CMA, removing a class of
  drift we would otherwise have to police.

The trade-off is that we are locked into the **CMA beta surface**. We have
already observed transient API instability (see §10), and any future change to
the beta header / endpoint shape needs to be absorbed by the SDK rather than by
each agent.

---

## 3. SDK Architecture (`@clutch-libs/ai-sdk`)

### 3.1 Hexagonal layers

The SDK is organized as a strict hexagon under `platform/libs/ai-sdk/src/`:

- **`domain/`** — entities (Session, Agent, MemoryStore, Tool, ModelConfig),
  pure data shapes, and domain errors. No HTTP. No SDK leakage.
- **`application/`** — ports (interfaces the application layer needs) and
  use-cases. The composition root that orchestrates deploy / invoke / destroy
  lives here.
- **`infra/`** — adapters. The Anthropic HTTP adapter (`infra/anthropic/`) is
  the only place the official `@anthropic-ai/sdk` package is allowed to be
  imported. The promptfoo provider also lives here.

`architecture-boundaries.spec.ts` enforces this with an in-test dependency
check. Cross-layer leakage breaks the build.

### 3.2 Why hexagonal

The decision was deliberate:

- The Anthropic SDK is a moving target. Hex layers give us a single seam to
  swap when their types shift.
- Many of our integrations (memory adapters, promptfoo bridge) exist only on
  the infra side — keeping them out of `domain` keeps tests fast and pure.
- We expect to onboard more agents on top of the SDK. A clean ports/adapters
  shape means each new agent only depends on `domain` + `application`.

### 3.3 The composition root (`AgentsSdk`)

`AgentsSdk` exposes three macro operations:

- **`deploy()`** — sends a spec (model + system prompt + tools + skills) to
  CMA. Content-hashed: redeploying an unchanged spec is a no-op on the
  Anthropic side.
- **`invoke()`** — opens a session (or reuses one), mounts memory stores as
  resources, projects events into memory files, sends the activation message,
  and surfaces the model's tool calls back to the caller.
- **`destroy()`** — tears down session + store, with best-effort cleanup of
  orphan resources.

This is the only surface a consumer (agent code, evals, future MCPs) needs.

### 3.4 Built-in `READ_PROTOCOL`

The SDK auto-appends an "activation protocol" block to every agent's system
prompt. It tells the model: when it sees `Activated. Events in this batch:`,
it must view the listed memory files first, then act. This standardizes the
first-N steps of every turn across agents and means individual agents don't
need to re-author this guidance.

---

## 4. Tooling Surface

CMA exposes three classes of tools. Our SDK supports all three; each agent
picks the right one per concern.

| Class | Use when | Trade-off |
| --- | --- | --- |
| **`customTool`** | The tool's effect is implemented inside our system, OR we want to ship behaviorally before the real integration exists. | The tool runs server-side as a no-op stub; the model just emits a structured call. We capture it on `metadata.toolCalls`. Cheap but contract-only. |
| **`mcpToolset`** | The tool is backed by a real MCP server we host (e.g., conversation-service, notifications). | Real side effects. Requires the MCP to be alive + reachable per session. |
| **`builtinToolset`** | The model needs to view / edit memory files itself (`view`, `edit`, `view_image`). | Mandatory whenever the system prompt instructs the model to read memory paths — otherwise the model lands with no `view` tool and hallucinates. |

`LoanOrigination` ships v1 with `send_message`, `notify`, `no_action` as
**custom-tool stubs** plus the **builtin toolset**. When the MCPs are ready,
the same agent will swap to `mcpToolset('conversation-service')` /
`mcpToolset('notifications')` with no system-prompt change — the tool names
and input shapes were designed to be MCP-compatible from day one.

---

## 5. Memory Model

CMA memory stores are isolated, mountable, per-session resources. We use a
canonical path layout for every agent the SDK hosts:

```
/mnt/memory/system/meta.md                              # partner / agent config
/mnt/memory/system/profile/recipient.md                 # the applicant
/mnt/memory/system/loan-app/<applicationId>.current.json
/mnt/memory/system/loan-app/<applicationId>.previous.json
/mnt/memory/agents/<AgentName>/observations.md          # agent's durable notes
/mnt/memory/agents/shared/summary.md                    # cross-agent summary (optional)
/mnt/memory/system/events/<...>                         # one file per inbound event in the batch
```

Decisions baked in here:

- **Current + previous snapshot** — the agent infers _what changed_ this turn
  by comparing `current` to `previous`. This is how the behavior block model
  detects `once_on_entry` transitions without needing a separate "change feed".
- **APR-0 scrub at write-time** — applications with `apr === 0` are persisted
  as `apr: null, aprStatus: 'not_finalized'`. Quoting "0% APR" / "$0 monthly"
  is a compliance violation, so we never let that value reach the model.
- **Events are files, not message content** — the activation message only
  lists paths; the model views each event file. This survives long batches
  (many events) without context blow-up.

---

## 6. The HAL Loan-Origination Agent

### 6.1 Identity model

The agent's _registered_ name (`LoanOrigination`) is for CMA. The
_applicant-facing_ name lives in `meta.md` (`Agent name`, `Agent title`) and
is read at runtime, with a fallback to `"loan origination assistant"`. This
means:

- The same deployed agent can present as different brands per credit union
  without re-deploying.
- The model never references the deprecated `Hal` identity in messages.

### 6.2 The Behavior Block Model

This is the single most important design choice in the agent and was authored
by Anna (canonical "Behavior Blocks" doc). Instead of branching on raw LOS
statuses, the agent reads a **resolved block** from the snapshot. Each block
has a stable `name` and five behavioral fields:

| Field | What it drives |
| --- | --- |
| `applicant_facing_meaning` | Semantic frame — paraphrased into the reply, never echoed verbatim |
| `agent_tone` | `positive` / `neutral` / `somber` / `empathetic` |
| `proactive_communication` | `every_change` / `once_on_entry` / `never` |
| `can_mention_loan_terms` | Compliance gate on APR/amount/term/payment |
| `should_try_to_get_applicant_to_complete_tasks` | Whether to surface `pendingActionItems` |

Seven canonical blocks + one fallback (`unknown_combination`). Block-specific
optional fields (`decision_sla`, `adverse_action_notice_delivery`,
`re_engagement_path`) carry block-local semantics.

**Why this matters:** the block model decouples the agent from LOS vocabulary
and from per-status hard-coding. New states are added by the orchestrator
shipping a new block — the agent prompt does not change. Credit unions that
care about different tonality/proactivity per state get it for free by
configuring the block, not by re-prompting.

### 6.3 Decision flow inside a turn

The system prompt encodes a single ordered decision flow:

1. Does `<silence>` apply? → `no_action` (or `notify(stop_word)` + `no_action`
   for STOP).
2. Does `<escalation>` apply? → `notify` AND `send_message` in the same turn.
3. Otherwise → `send_message` only.

This is enforced by a `<turn_completion>` block (a softer replacement for the
original `<tool_use_required>` gate, kept after experiments showed the model
would otherwise leak reasoning text without ever calling a tool).

### 6.4 Tool surface (v1)

Three custom-tool stubs:

- `send_message` — the SMS to the applicant (the only applicant-visible
  channel).
- `notify` — the staff-side escalation, with a `category` enum that the system
  prompt maps from triggers. Categories include `general_escalation`,
  `change_application`, `change_amount_or_term`, `cancel_application`,
  `up_sell`, `distress`, `stop_word`, `no_application`, `action_complete`,
  `preapproval_request`, `ask_for_person`.
- `no_action` — silence, with an explicit `reasoning` field. The reasoning is
  durable and judge-able.

The TCPA constraint ("one outbound per inbound") is enforced by the system
prompt, not by infrastructure — the dispatcher gates outbound timing
downstream; the agent controls the count.

---

## 7. Evaluation Strategy (HAL-1677)

### 7.1 Why promptfoo

Promptfoo gave us a row-oriented harness, structured assertions, and an
LLM-as-judge rubric in a single config. We did not want to invent yet another
eval framework. The integration is a small adapter:
**`AgentsSdkPromptfooProvider`** lives inside the SDK (infra layer) and
exposes the agent's tool calls on `context.providerResponse.metadata` so
assertions can inspect them directly.

### 7.2 Per-row isolation

The provider creates a **fresh memory store + session per row**, seeds the
memory files from `vars.memory.setup`, projects each event into a memory
file, sends the activation, captures the tool calls, then tears down. This
gives us hermetic test rows at the cost of provisioning overhead per row.

Trade-off accepted:

- Multi-turn conversations within a single row are **not supported** — the
  Python source's `conversation_history` is flattened into seeded
  `observations.md` and observed-already messages. Lossy for scenarios that
  depend on fine-grained turn ordering; flagged per scenario when it bites.

### 7.3 Two-tier assertions

Each row carries two complementary asserts:

- A **`javascript` assert** that inspects the structural shape of the turn
  (which tools fired, which categories, optional content-pattern checks).
  This is fast, deterministic, and catches behavior-shape regressions.
- An **`llm-rubric` assert** that grades the actual SMS the applicant would
  receive (post-`transform`) against a prose rubric describing the expected
  behavior. This catches tone/content regressions a regex never could.

The `transform` is critical: the model emits reasoning text BEFORE its tool
calls; without a transform the rubric grades the reasoning, not the
applicant-visible message.

### 7.4 Multiple defensible behaviors

Several scenarios accept **more than one defensible interpretation** of the
prompt. Examples:

- Canceled-block lifecycle event → strict "no_action" OR a single final-
  acknowledgement `send_message`.
- Garbled inbound → `notify(general_escalation)` OR `notify(change_application)`
  OR `notify(change_amount_or_term)` (depending on parseable intent).
- Reminder on `once_on_entry` block → send the nudge OR honor the block and
  `no_action` (with reasoning).

We encode "both are OK" inline in the JS assert rather than picking one and
producing flake. The eval suite is a **specification of acceptable behavior**,
not a single golden output.

### 7.5 Migration cadence

The 66 legacy Python scenarios are migrated in **batches of 5**, one PR each.
This deliberately small unit:

- Lets each PR be reviewed independently.
- Surfaces prompt-design tensions early (e.g., the `reminder.scheduled.fired`
  vs `once_on_entry` tension was found in Batch 7).
- Lets the team correct rubric strictness incrementally instead of
  re-tuning 66 rubrics at once.

Batch order: P3 first (smoke tests), then P2 (refusal/edge), then P1
(lifecycle / escalation / docs / refusal), then P0 (highest-priority
guardrails). P0 is intentionally last because the suite is more mature by
the time we get there, reducing the cost of P0 false-positives.

---

## 8. Key Decisions, Summarized

| Decision | Why | Cost |
| --- | --- | --- |
| Use CMA (managed agents) over raw Messages API | Free memory/session/tool plumbing; matches our data model. | Locked into beta; we absorb instability. |
| Hexagonal SDK | Single seam to swap Anthropic SDK shifts; clean for multi-agent. | More files / indirection for first-time readers. |
| Custom-tool stubs for v1 | Ship HAL before MCPs are ready; no MCP infra blocking. | We trust the model's tool args without runtime side-effects yet. |
| Builtin toolset on every memory-reading agent | Without `view`, the model can't read memory paths and hallucinates. | None — required. |
| Behavior block model (not raw status switch) | Decouples agent from LOS vocab; per-CU tonality without re-prompting. | The orchestrator must produce a resolved block per turn. |
| APR-0 scrub at write time | Compliance — agent must never quote "0% APR". | Tiny normalization complexity in writers. |
| Current + previous snapshots | Lets the agent detect transitions / `once_on_entry` natively. | Doubles snapshot storage per app. |
| Activation protocol auto-appended | Standardizes first-N steps; agents don't re-author them. | A single `READ_PROTOCOL` string is now part of the SDK contract. |
| `<turn_completion>` (softer than `<tool_use_required>`) | Avoids the model leaking reasoning as if it were the reply, without the harsh "text is discarded" framing. | Slightly more prompt; periodic re-tightening when models update. |
| Promptfoo with per-row session/store | Hermetic eval rows; structural + semantic asserts in one harness. | One CMA invoke + one judge call per row → cost grows linearly. |
| Two-tier asserts (JS + llm-rubric) | JS catches structural regressions cheaply; rubric catches tone/content. | Rubrics drift over model versions; periodic re-tuning. |
| Accept multiple defensible behaviors | Reflects reality of how the prompt can be read; reduces flake. | More verbose assertions; reviewers must read both branches. |
| Batches of 5 / PR | Reviewable; surfaces issues early; doesn't blow up CI. | Many PRs to track; chained branch dependency. |

---

## 9. Pros

- **Velocity from primitives.** CMA's memory + session + toolset model meant
  the agent itself was a thin layer of prompt + tool specs. Most of our work
  is product (block model, rubric tuning), not infrastructure.
- **Hexagonal payoff has already shown.** When the Anthropic SDK
  surface shifted (sessions namespace location), the change was contained to
  one adapter file. No agent code changed.
- **Content-hash deploys.** Re-deploying an unchanged agent is free. We rely
  on this in CI and dev loops without thinking about it.
- **One-row-one-session isolation.** Hermetic eval rows are the difference
  between "trust the result" and "is this a flake?". We pay the cost
  willingly.
- **Behavioral specs as code.** The eval suite is now executable
  documentation of how the agent must behave. Future prompt changes get a
  diff against intent, not just a diff against prose.
- **Migration cadence sized to context.** Five-row batches kept each PR
  small enough that we can ship one or two per day even with the LLM judge in
  the loop.

---

## 10. Cons / Known Limitations

- **CMA beta instability.** Transient `session.event stream failed with an
  unmapped error` and `invalid arguments` errors hit a small percentage of
  rows. Retry usually clears it; some require deleting orphan sessions
  manually.
- **Promptfoo template expansion gap.** The `{{__runIdx}}` placeholder in
  `vars.memory.profileId` is sometimes NOT expanded by promptfoo, producing
  sessions named with the literal template string. Subsequent runs collide
  on the orphan and fail. Mitigation today is manual cleanup; the real fix
  belongs in the SDK's promptfoo provider (defensive UUID fallback or
  pre-run orphan sweep).
- **Workspace scoping.** The Anthropic API key, the Console UI, and the
  deployed agent must be in the same workspace. A mismatch produces "agent
  not visible" symptoms even when deploys succeed. Diagnosing this requires
  external (IT) help.
- **Cleanup is best-effort.** When a stream fails mid-turn, the teardown
  also tends to fail. Stranded sessions accumulate and bill memory until
  swept.
- **Rubric is overly literal periodically.** LLM judges occasionally fail
  rows where the agent's behavior is fine but phrased differently than the
  rubric anticipated (e.g., "we received" vs "thank you" as ack equivalents;
  "decline" as the applicant's choice verb vs "declined" as credit decision).
  Mitigation is explicit rubric guidance for what is NOT a violation.
- **Multi-turn flattening.** Scenarios with rich `conversation_history`
  collapse to a single seeded `observations.md`. We lose turn-by-turn
  sequencing. Flagged in the README; reasonable for now but not forever.
- **Cost grows linearly.** Every row = 1 CMA invoke + 1 judge call. A
  44-row suite is ~$X per run and ~30–45 minutes wall-clock at
  `max-concurrency=1`. The provider can be concurrent but per-row isolation
  makes that an Anthropic-side rate-limit conversation.
- **Locked into TypeScript on the SDK side.** This is fine for our stack,
  but a future Python agent would have to either re-author the SDK or call
  CMA directly.

---

## 11. Open Issues / Future Work

In rough priority order:

1. **Defensive `__runIdx` expansion in the promptfoo provider.** Eliminates
   the orphan-session class of flakes. Single highest-leverage SDK fix.
2. **Auto-sweep of orphan sessions/stores before each eval run.** Belt to
   the suspenders fix above.
3. **MCP swap for `send_message` / `notify`.** Once
   `conversation-service` and `notifications` MCPs ship, replace the custom
   tool stubs. System prompt does not change.
4. **Resolve the reminder-vs-block tension in the prompt.** Today
   `reminder.scheduled.fired` is ambiguously documented vs
   `proactive_communication: once_on_entry`. Pick one and update both the
   prompt and the affected scenarios.
5. **Port the G1–G6 / Q1–Q7 structured rubric from `hal/v2/eval/judge.py`.**
   Currently the eval LLM judge is per-row prose; the Python suite had a
   structured rubric with explicit dimensions. Bringing it over gives us
   numeric scoring per row across stable dimensions.
6. **Severity metadata on rows.** Today we can't easily filter "run only
   P0". Adding `severity` as `metadata` per row gives us
   `--filter-metadata severity=P0` for fast pre-deploy gates.
7. **Multi-turn eval support.** Some scenarios truly need multi-turn. The
   provider would need to keep the session alive across rows that share a
   conversation key. Significant change; defer until a real need.
8. **Generator script for snapshots.** When the suite grows past ~80 rows,
   hand-authoring each `loan-app/<id>.current.json` becomes lossy. A small
   helper that maps the legacy Python `customer.status` onto the canonical
   `PLATFORM_DEFAULT_BLOCKS` will land in the batch that first needs it.

---

## 12. References

- **Anthropic Managed Agents (CMA)** — the beta product we sit on.
  Beta header: `managed-agents-2026-04-01`.
- **Anna's "Behavior Blocks" doc** — the canonical 7-block model used by
  `PLATFORM_DEFAULT_BLOCKS`.
- **`hal/v2/eval/scenarios.yaml`** — source-of-truth for the 66 behavioral
  scenarios being migrated to promptfoo under HAL-1677.
- **`agents/loan-origination/evals/README.md`** — the per-PR migration
  status board and how-to-run.
- **`platform/libs/ai-sdk/README.md`** — SDK quickstart, available
  toolsets, deploy/invoke API surface.
