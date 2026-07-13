# Household Specification

This document is the technical counterpart to [`requirements.md`](./requirements.md). Where the
requirements describe *what* the household must do in the language of staff and service, this
document describes *how* it is wired: the topology, the components, and the open-source tooling
chosen for each duty. The metaphor is dropped here in favour of plain engineering terms, with a
decode table to keep the two documents in sync.

It is opinionated on purpose. The Owner asked to compare mental models of the architecture, so every
choice below names a default tool, the runners-up, and the reason for the pick. Tools move fast;
each recommendation is timestamped to mid-2026 and should be re-checked before build.

## 0. Decode: household → system

| In `requirements.md`      | In this specification                                              |
| ------------------------- | ------------------------------------------------------------------ |
| The household / the Court | The multi-agent runtime as a whole (one Owner, local-first)        |
| A member / officer        | An agent: a persona + Charter + toolset + model-routing profile    |
| The Chamberlain           | The orchestrator/supervisor agent (decompose, delegate, arbitrate) |
| The Owner                 | The single human principal                                         |
| A room                    | A channel: terminal (TUI), messaging gateway, voice                |
| Calling someone by name   | Agent-to-agent invocation (A2A) / subagent spawn                   |
| The servant               | Local LLM inference, finite, queued                                |
| The artisans in the city  | External frontier model APIs (paid, irreversible egress)           |
| The household's records   | Private docs/repos exposed via RAG + MCP                           |
| The household's memory    | Typed persistent stores (owner / scratch / skills)                 |
| A member's Charter        | Private system prompt + immutable policy (Owner-authored)          |
| The Roll                  | Public agent registry/directory (standing, not laws)               |
| The gatekeeper            | Egress control: guardrails + PII/DLP + policy decision point       |
| Delicacy / a confidence   | Data sensitivity label that travels with the data (taint)          |

## 1. Architecture at a glance

The household is **one shared control plane with a thin per-member gateway on top**. Each member is a
**Hermes profile** — its own gateway daemon and bot token (so it is a distinct, DM-able Discord
identity), its own Charter, memory peer, skills and cron. All those per-member gateways sit on a
**single host** and share one control plane: **one LiteLLM router that is also the gatekeeper**, **one
Kanban board**, **one Honcho memory workspace**, and **one local servant (LM Studio)**. Members reach
the outside world only through the shared LiteLLM gatekeeper, which fronts a two-tier inference fabric
(local servant + remote artisans).

The frame to hold: not "single gateway vs many" but *unified control plane, lightweight per-member
bot*. The system is operationally singular (one gatekeeper, one board, one machine) while members
remain first-class identities the Owner can message directly or convene into a room.

```
                          ┌──────────────────── ROOMS (channels) ────────────────────┐
                          │  Terminal TUI · Telegram/Slack/Discord/WhatsApp/Signal ·  │
                          │                 Voice (STT/TTS)                           │
                          └───────────────────────────┬──────────────────────────────┘
                                                       │ normalized envelopes
                                                       ▼
┌─────────────────── MEMBERS as Hermes PROFILES (one named bot per member) ─────────────────────────┐
│  Each member = its own gateway daemon + bot token (Owner can DM it directly, no orchestrator):      │
│     Channel bridges · Session mgr · Command queue (lanes) · Cron · Auth & trust                     │
│     Charter (SOUL.md + laws) · own Honcho memory peer · skills · routing profile · sensitivity tier │
│                                                                                                    │
│  ┌──────────────────────────── CHAMBERLAIN (orchestrator profile) ──────────────────────────────┐ │
│  │     decompose great matters → Kanban tasks · assign to members · gather · arbitrate memory    │ │
│  └───────────────────────────────────────────────────────────────────────────────────────────────┘ │
│  MEMBERS: Mason · Smith · Reeve · Steward · Physician · Herald · Huntsman · Confessor              │
│                                                                                                    │
│  COORDINATION:                                                                                      │
│   • KANBAN board (durable queue; comment = inter-agent protocol; Owner reads/writes/blocks) ← CV-1/4│
│   • delegate_task (in-process fork→join quick consult) ← CV-2                                       │
│   • Roll (profile --descriptions + Agent-Card directory; Roll lookup before sharing) ← CV-5/6      │
│   • Group channel (member-bots co-present, @mentionable; Owner watches live + interrupts) ← CV-7   │
│                                                                                                    │
│  SHARED STATE:  Roll (registry) · Memory (owner=Honcho / scratch / skills) · Records index (RAG)   │
└──────────────────────────────┬───────────────────────────────────┬─────────────────────────────────┘
                               │                                   │
                  ┌────────────▼─────────────┐        ┌────────────▼──────────────────┐
                  │  LiteLLM = ROUTER +      │        │      RECORDS / RAG             │
                  │  GATEKEEPER (egress)     │        │  LlamaIndex pipeline +         │
                  │  route + budgets/caps +  │        │  Qdrant vector store +         │
                  │  Presidio PII + prompt-  │        │  incremental file watcher      │
                  │  injection + categorical │        └────────────────────────────────┘
                  │  no-egress by label      │
                  └──────┬─────────────┬──────┘
                         │             │
              ┌──────────▼───┐   ┌─────▼─────────────┐
              │  SERVANT     │   │  ARTISANS (city)  │
              │  LM Studio   │   │  Anthropic/OpenAI │
              │  (local,     │   │  Google/OpenRouter│
              │  OpenAI-API) │   │  (paid, external) │
              │  + queue     │   │                   │
              └──────────────┘   └───────────────────┘

Cross-cutting: Secrets (OpenBao/Infisical) · Observability (Langfuse, OTel GenAI)
One shared control plane (LiteLLM gatekeeper · Kanban · Honcho · host) under N per-member bot gateways
```

### 1.1 Chosen harness (don't build the runtime from scratch)

The hardest 80% — gateway daemon, multi-channel bridges, per-session queues, subagent spawning,
skill loading, cron, voice transcription, model-agnostic routing — is already solved by mature
local-first personal-assistant harnesses. We adopt one rather than reinvent it.

| Option                                | What it gives the household                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                | Verdict                                                                                                                                             |
| ------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Hermes Agent** (Nous Research, MIT) | **Profiles** — multiple independent *named* agents, each with its own bot token/gateway, config, API keys, persona (`SOUL.md`), memory, sessions, skills and cron (the members); **Kanban board** — durable multi-agent task queue where named profiles collaborate and every handoff/comment is an inspectable, human-writable row (the Chamberlain's fan-out + visible Court); `delegate_task` for in-process quick consults; **skills-from-experience** (techniques, MR/FR-9); FTS5 cross-session search; **native Honcho** per-profile user-modeling (owner-knowledge); built-in cron (speaking first); voice-memo transcription (quiet room); any-model routing; serverless hibernation for "servant comes and goes". | **Primary backbone.** Maps onto the household more directly than anything else found.                                                               |
| **Honcho** (Plastic Labs, OSS)        | Dialectic *theory-of-mind* user modeling: a continually-learned representation of the Owner, queryable per agent.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          | **Adopt** as the owner-knowledge memory layer (already first-class in Hermes).                                                                      |
| Claude Code / GitHub Copilot CLI      | Strong tool-using coding agents with MCP.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  | **Sub-harness** for the Mason and Smith (workroom trades), invoked as tools.                                                                        |
| OpenClaw (ex-Clawdbot)                | Same category as Hermes; 24+ channels, plugin architecture, sandboxed tools.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                               | **Alternative only.** Noted for completeness; the ecosystem is fragmenting and being absorbed by larger vendors, so it is not the recommended base. |

Everything below either *configures* the chosen harness or *bolts a specialist component beside it*.

## 2. Component specifications

### 2.1 The members and the Chamberlain (FR-2, FR-14, CV-4)

- **Each member is a Hermes Profile**, not an in-process persona: a fully independent named agent
  with its own bot token/gateway, `config.yaml`, API keys, persona (`SOUL.md` = Charter), memory
  (its own Honcho peer), sessions, skills, cron and routing profile. This is what lets the Owner DM a
  member directly (§2.2) and what keeps one member's state from bleeding into another's (MR-3).
- **Member self-description for routing:** each profile is created with a `--description` of its
  trade, so the Chamberlain's orchestrator knows what each member is good at when sharing out work —
  this is the machine-readable half of the Roll (§2.7).
- **Two ways to orchestrate, sharing one Kanban board — both first-class:**
  - **Chamberlain fan-out (the default for large matters; FR-14, CV-4):** the Owner hands a big,
    multi-trade goal to the **Chamberlain**, an *orchestrator profile* that breaks it into **Kanban
    tasks**, assigns each to the right member, and gathers the results — unattended. This is the
    "arrange a parent's 70th birthday" path (requirements §2.15): drop it on the Chamberlain and walk
    away.
  - **Owner-as-orchestrator (on demand; requirements §2.15 variant):** for matters the Owner wants to
    drive personally, the Owner opens a room, convenes the relevant members, and dispatches the work
    themselves (§2.2). The Owner *is* the orchestrator for that matter.
  - These are not in tension: both create and comment on rows in the **same Kanban board**, so the
    Owner can drop a large matter on the Chamberlain *and* step in to drive a specific one — even take
    over a Chamberlain-run task mid-flight by commenting or `block`/`unblock`-ing its rows.
- **Quick consult (CV-2):** `delegate_task` — an in-process fork→join call where one member borrows
  a short answer from another and control returns to the caller. No board row, no orchestrator.
- **The Chamberlain carries no trade and is not a bottleneck:** quick consults (`delegate_task`),
  direct Owner↔member DMs, and Owner-driven rooms never pass through it; it owns automated
  decomposition and memory arbitration (§2.6), and the Owner can always orchestrate without it.
- **If an explicit, inspectable decomposition graph is wanted**, **LangGraph** can host the
  Chamberlain's planning state machine while Hermes remains the gateway and Kanban the work queue.
- **Each member declares:** trade, home rooms, sensitivity tier, speak-first policy, home/city
  leaning — the columns in requirements §1.2, expressed as profile config + Charter + Roll entry.

### 2.2 The rooms / channels — Discord-centric (FR-1, FR-13, RM-1…5, OR-1/2/5)

**Discord is the primary venue, with channels as rooms.** Hermes maps Discord cleanly: each DM,
thread, and channel is its own session namespace, so a channel *is* a room and the Owner can spin up
new rooms at will.

- **Direct Owner↔member conversation (RM-2, requirements §2.12):** each member is its own profile/bot,
  so the Owner pulls a member into a room by **DMing that member's bot** — e.g. DM the **Confessor** in
  the quiet room, no orchestrator in the path. Native to the Profiles model.
- **Channels as named rooms (RM-1, RM-3):** a Discord channel maps to a room (e.g. `#messaging-hall`,
  `#workroom`, `#quiet-room`). Conversation stays confined to its channel (each channel/thread is a
  separate Hermes session); cross-room carry is deliberate, not a default leak.
- **Owner-as-orchestrator — a third coordination mode (CV-4 variant, requirements §2.15):** the Owner
  creates a new channel and invites the relevant member-bots (e.g. Herald + Steward + Huntsman for a
  birthday), then drives the discussion personally. Configured as a **free-response channel**
  (`DISCORD_FREE_RESPONSE_CHANNELS`) with `group_sessions_per_user: false` so the room is one shared
  transcript every present member sees. This complements — does not replace — the Chamberlain's
  automated fan-out (Kanban) and peer consults (`delegate_task`).
- **The Court in session — visible inter-agent comms (CV-7, see §2.11):** in a shared channel the
  member-bots are co-present and `@mentionable`; the Owner watches hand-offs live and interrupts a
  drifting or hallucinating exchange on the spot. The Kanban comment thread is the durable record
  behind the live view.
- **Reach away from home / servant asleep (RM-4, OR-1/2):** Discord (and other bridged channels) keep
  the Owner reachable wherever they are and while the servant is down, for members that can work
  without it.
- **Voice (quiet room, FR-13):** **faster-whisper** (CTranslate2) or **whisper.cpp** for offline STT,
  plus a local TTS (e.g. Piper) for spoken replies; Hermes handles Discord voice messages natively.

### 2.3 The servant — local inference (CR-1, FR-12, OR-3/5)

- **Engine: LM Studio** as the local servant — it exposes an OpenAI-compatible server that LiteLLM
  treats as just another backend, so the same routing/gatekeeping path covers home and city uniformly.
  If raw multi-member concurrency outgrows LM Studio, **vLLM** (continuous batching) is the scale-up
  swap behind the same LiteLLM seam; Ollama/llama.cpp are lighter alternatives. The servant choice is
  isolated behind LiteLLM, so it can change without touching members.
- **The visible queue (FR-12):** a single FIFO work queue in front of the servant, its depth exposed
  to every member. Hermes' lane-aware command queue provides the serialization primitive; the queue
  *depth/ETA* is surfaced to agents so each decides for itself whether to wait or go to the city. No
  agent is appointed to ration — the routing decision is per-member (see §2.4).

### 2.4 Routing & thrift (CR-1…6, FR-5/6/7, OR-4)

- **Gateway:** **LiteLLM** (OpenAI-compatible proxy). Provides: model routing, fallback chains,
  **per-member and per-task budgets / spend caps**, cost tracking, and a single seam to record
  *where each piece of work ran and why* (OR-4).
- **Routing policy (the decision):**
  1. Default to the **servant** (CR-1, thrift first).
  2. Escalate to an **artisan** when the task exceeds local capability (CR-2/FR-6) **or** the queue
     is too deep / servant absent (CR-5/FR-7) — these are two independent signals.
  3. **Sensitivity overrides thrift (CR-3/CR-6):** if the matter's delicacy forbids egress, the work
     stays local and waits, however long the queue or however absent the servant.
- **Servant-unavailable matrix (CR-4):** each (member, task-class) declares one of `send-out`,
  `hold`, `set-aside`. A confidential member rightly chooses `hold`. Encoded as policy (§2.6), not
  ad-hoc code.

### 2.5 Records / RAG (FR-3, FR-4, OR-3)

- **Pipeline:** **LlamaIndex** for ingestion, chunking, retrieval and *response synthesis* (members
  must return a finished reply, not raw clippings — FR-3).
- **Vector store:** **Qdrant** (self-hosted, fast, good filtering — needed to enforce sensitivity
  filters on retrieval); pgvector is an acceptable lighter-weight alternative if Postgres is already
  present.
- **Freshness (FR-4, OR-3):** a file-system / repo watcher drives **incremental, idempotent**
  re-indexing (content-hash dedup so re-running changes nothing already correct). Records are exposed
  to members as **MCP** resources/tools so any member can consult them uniformly.

### 2.6 Memory (MR-1…7)

Three stores kept apart (MR-1), each with provenance and a sensitivity label:

| Memory kind                  | Purpose                                      | Tooling                                     | Notes                                                      |
| ---------------------------- | -------------------------------------------- | ------------------------------------------- | ---------------------------------------------------------- |
| **Owner-knowledge** (shared) | What the household knows of the Owner (MR-2) | **Honcho** dialectic user model             | Read-shared among permitted members; write-guarded (MR-4). |
| **Member scratch** (private) | Each member's own working notes (MR-3)       | Hermes per-agent session store (FTS5)       | Never visible to other members.                            |
| **Skills / techniques**      | Learned methods (FR-9)                       | Hermes skills + **agentskills.io** standard | Each skill scoped *member-only* or *household-wide*.       |

- **Sensitivity-aware reads (MR-6, SR-5/SR-6):** every remembered item carries a delicacy label; the
  label — not the writer's identity — governs who and which outsiders may ever read it. This closes
  the "back-stair leak" (requirements §2.7): a member preparing an egress call cannot pull a
  higher-sensitivity shared note into a city-bound prompt.
- **Contradiction handling (MR-5):** a temporal knowledge-graph memory (**Graphiti / Zep**) detects
  when two remembered truths disagree and raises them to the Chamberlain, who has final say over the
  shared store. Graphiti's bi-temporal model also satisfies provenance/"when said" (MR-7).
- **Write-guard (MR-4):** writes to shared memory pass a higher bar (Chamberlain or policy approval)
  than private scratch writes.

### 2.7 The Court conversations (CV-1…8, RM-2)

- **Call-by-name / handoff (CV-1, RM-2):** done through the **Kanban board** — a member (or the
  Chamberlain) creates a task assigned to another member by name; the assignee, a named profile, picks
  it up as its own OS process. The handoff is a durable, inspectable row, and because handing over
  work can hand over the Owner's money, it is visible by construction (CV-7).
- **Quick consult (CV-2):** `delegate_task` — a bounded "borrow an answer" fork→join that returns
  control to the caller and stays in-room. Whose delicacy applies is set by the *answering* member's
  tier plus the label on what is shared. (Use Kanban when the work crosses boundaries or needs the
  Owner; use `delegate_task` when it's a short answer feeding back into the caller.)
- **Broadcast / news that should spread (CV-3):** a member posts to a shared group channel or a
  Kanban comment that relevant members read, with the delicate *origin* stripped per the Roll
  (requirements §2.14).
- **The Roll (CV-5/6, SR-1, requirements §2.13):** two coupled artifacts — each member profile's
  `--description` (the machine-readable standing the Kanban orchestrator routes on) plus a shared
  registry derived from Charters exposing each member's trade and discretion but **not** its private
  laws. Before sharing, a member consults the Roll to share only what's needed. An A2A **Agent Card**
  directory or MCP discovery resource is the clean way to publish it.
- **Charters (CV-8, SR-8):** each member's private `SOUL.md` persona plus immutable laws (spend caps,
  "never do X unbidden"), known to that profile alone. Laws are enforced as policy (§2.8), not mere
  prompt text, so they cannot be talked out of the agent.
- **Visibility (CV-7) — live and durable, see §2.11:** member-bots share a group channel where the
  Owner watches exchanges in real time and can stop a drifting run; the Kanban comment thread (the
  "inter-agent protocol", human-readable and human-writable) is the durable record; Langfuse traces
  cost and tool calls. Three layers: live oversight, durable audit, deep trace.

### 2.8 The gatekeeper, laws & discretion (SR-1…9, CR-3/6, MR-6)

Defence in depth, in three lines as the requirements demand:

1. **Discretion (first line, SR-3):** each member, guided by the Roll, shares only what's needed.
   This is behavioural (prompt + Roll lookup), reinforced by §2.6 sensitivity reads.
2. **Laws (Charter, SR-8):** hard per-member constraints — spend caps, forbidden actions — enforced
   by a **policy engine: Open Policy Agent (Rego)** or **Cedar**. Spend caps also enforced by LiteLLM
   budgets. Policy is the single decision point for the servant-unavailable matrix (CR-4) too.
3. **Gatekeeper (last line, SR-2/4/5) — co-located in the LiteLLM router:** because *all* artisan
   (city) traffic is LLM calls and every LLM call goes through LiteLLM, the egress chokepoint lives in
   the router itself, as a **LiteLLM guardrail** that has the final word regardless of member intent:
   - **PII / DLP:** LiteLLM's **Presidio** guardrail (pre-call hook) detects and masks/blocks sensitive
     entities before the payload leaves for an artisan.
   - **Content guardrails:** LiteLLM prompt-injection detection; optionally **Llama Guard** /
     **NeMo Guardrails** as additional rails for output safety classification.
   - **Hard rule (SR-2) via label, not regex:** the most delicate tiers (Physician's health,
     Confessor's confidences, Steward's money) are *categorically* non-egressable. This is enforced by
     passing the request's **sensitivity label as LiteLLM metadata**; a guardrail rule blocks any
     city-bound call whose label exceeds the egress threshold outright — no cost saving or queue depth
     overrides it. (Regex PII masking is the second net, not the primary control.)
   - **Travelling delicacy (SR-5):** the label rides with the data as request metadata through every
     hop, so the router judges the payload by what it *is*, not who carries it.
   - **Residual path to close:** a member tool that calls a third party *directly* (email, web POST,
     non-LLM API) bypasses a router-only gatekeeper. The "city" in the requirements is specifically the
     artisans/LLMs, so this is out of the literal scope — but high-sensitivity members should have such
     outbound tools disabled in their Charter (enforced as a law, below) so the LiteLLM chokepoint
     remains the *only* door to the outside.
- **Secrets (SR-9):** **OpenBao** (community Vault fork) or **Infisical** holds keys for servant,
  artisans and channels — including LiteLLM's virtual keys and the per-member Discord bot tokens. No
  secret in source or prompt.

### 2.9 Speaking first / proactivity (FR-10, SR-7, RM-5, OR-2)

- **Triggers:** Hermes built-in **cron** for scheduled nudges (a bill due, a lapsed streak, a
  birthday) and event hooks for happenings; delivery to any channel, reaching the Owner away from
  home (RM-5) and while the servant sleeps (OR-2).
- **Per-member leave (SR-7):** whether a member may speak first, in which rooms, and how
  insistently, is a per-member setting in its Charter — the Confessor set to *never*, the Steward and
  Physician permitted. Enforced by policy, not a global rule.

### 2.10 Onboarding a new member (FR-11)

- Adding a member = `hermes profile create <name> --description "<trade>"` + its Charter (`SOUL.md`),
  Roll entry, routing profile and room assignments. Because each member is an independent profile
  over shared memory (Honcho) and shared skills, a newcomer draws on the household's memory and
  established techniques from its first turn without disturbing the others.

### 2.11 Observability & Owner oversight (OR-4, CV-7)

Three layers, from real-time intervention to deep forensics:

1. **Live oversight (catch drift as it happens):** member-bots share a group channel (Discord /
   Telegram) where the Owner watches inter-agent exchanges stream in. If a conversation drifts or a
   member begins to hallucinate, the Owner stops it on the spot — **interrupt-and-redirect** (type a
   new message), **`/steer`** (inject a correction without cancelling), **`Ctrl+C`**, the **`/agents`
   overlay** (live subagent tree with kill/pause controls), or **`kanban block`** to freeze a task and
   comment a correction. This is the mechanism your instinct pointed at, and it is the household's
   primary guard against runaway agent loops.
2. **Durable audit (what was passed, and what it may cost):** the Kanban comment thread — the
   inter-agent protocol — is a permanent, human-readable/writable SQLite record of every handoff,
   surviving restarts and context compaction.
3. **Deep trace:** **Langfuse** (OpenTelemetry GenAI-native) records which member ran each step, home
   or city, token cost, and tool calls — the substrate for "where each piece of work was done and why"
   (OR-4).

## 3. Requirement → component traceability

Every requirement ID has a home. (Components: GW=gateway/Hermes, CH=Chamberlain, RT=LiteLLM routing,
SV=servant, RAG=records, MEM=memory, GK=gatekeeper, POL=policy, ROLL=registry, OBS=observability.)

| ID    | Covered by                                            |
| ----- | ----------------------------------------------------- |
| FR-1  | GW channels (§2.2)                                    |
| FR-2  | Members as Profiles + Charters (§2.1)                 |
| FR-3  | RAG response synthesis (§2.5)                         |
| FR-4  | Incremental indexer (§2.5)                            |
| FR-5  | RT routing decision (§2.4)                            |
| FR-6  | RT escalate-on-capability (§2.4)                      |
| FR-7  | RT escalate-on-busy/absent (§2.4)                     |
| FR-8  | MEM three stores (§2.6)                               |
| FR-9  | Skills, scoped member/household (§2.6)                |
| FR-10 | Cron + hooks (§2.9)                                   |
| FR-11 | Config-driven onboarding (§2.10)                      |
| FR-12 | Visible servant queue (§2.3)                          |
| FR-13 | Voice STT (§2.2)                                      |
| FR-14 | Chamberlain decompose+arbitrate (§2.1, §2.6)          |
| MR-1  | Three separated stores (§2.6)                         |
| MR-2  | Honcho owner-knowledge share (§2.6)                   |
| MR-3  | Per-agent scratch (§2.6)                              |
| MR-4  | Shared-write guard (§2.6)                             |
| MR-5  | Graphiti/Zep contradiction → CH (§2.6)                |
| MR-6  | Sensitivity labels on reads (§2.6, §2.8)              |
| MR-7  | Provenance / bi-temporal (§2.6)                       |
| CR-1  | Servant-first (§2.4)                                  |
| CR-2  | Artisan reachable (§2.4)                              |
| CR-3  | Delicacy overrides thrift (§2.4, §2.8)                |
| CR-4  | Unavailable matrix via POL (§2.4)                     |
| CR-5  | Busy ≠ hard, separate signal (§2.4)                   |
| CR-6  | Egress only if label permits (§2.4, §2.8)             |
| CV-1  | Kanban handoff by name (§2.7)                         |
| CV-2  | Quick consult (§2.7)                                  |
| CV-3  | Broadcast w/ origin stripped (§2.7)                   |
| CV-4  | CH share-out, no bottleneck (§2.1, §2.7)              |
| CV-5  | Roll registry (§2.7)                                  |
| CV-6  | Roll lookup before sharing (§2.7)                     |
| CV-7  | Live group channel + Kanban audit + OBS (§2.7, §2.11) |
| CV-8  | Private Charters (§2.7)                               |
| RM-1  | Rooms (§2.2)                                          |
| RM-2  | Direct DM to member bot (§2.2, §2.7)                  |
| RM-3  | Room confinement (§2.2)                               |
| RM-4  | Reach away from home (§2.2)                           |
| RM-5  | Speak-first away from home (§2.9)                     |
| SR-1  | Member standing in Roll (§2.7, §2.8)                  |
| SR-2  | Categorical non-egress tiers (§2.8)                   |
| SR-3  | Discretion first line (§2.8)                          |
| SR-4  | Gatekeeper last line (§2.8)                           |
| SR-5  | Travelling delicacy label (§2.8, §2.6)                |
| SR-6  | Back-stair leak closed (§2.6, §2.8)                   |
| SR-7  | Per-member speak-first (§2.9)                         |
| SR-8  | Charter laws via POL + LiteLLM caps (§2.8)            |
| SR-9  | Secrets manager (§2.8)                                |
| OR-1  | Reachable when servant asleep (§2.2)                  |
| OR-2  | Speak-first when servant asleep (§2.9)                |
| OR-3  | Idempotent re-index (§2.5)                            |
| OR-4  | Provenance of where work ran (§2.11)                  |
| OR-5  | Continuity across servant churn (§2.2)                |

## 4. Where my mental model may differ from the Owner's

Points most worth arguing about before build — these are the seams where a different architect would
reasonably diverge:

1. **RESOLVED — members as separate named agents, not in-process personas.** The Owner's model won
   here: each member is a Hermes **Profile** (own bot token, own memory peer, own process), so the
   Owner can DM a member directly (e.g. the Confessor) with no orchestrator in the path, and members
   collaborate over the durable **Kanban** board rather than a fragile in-process subagent swarm. The
   Roll is therefore a real directory of named agents, not just an in-process lookup. This replaces
   the earlier "one runtime, many personas" framing.
2. **RESOLVED — visibility is live, not just after-the-fact.** Inter-agent exchanges surface in a
   shared group channel the Owner watches in real time, with interrupt/`/steer`/`block` controls to
   stop drift or hallucination mid-run (§2.11). Langfuse remains for deep forensics, but the primary
   oversight is the live channel — the Owner's point, now the spec's position.
3. **RESOLVED — gatekeeper co-located in the LiteLLM router.** Since all city traffic is LLM calls
   through LiteLLM, egress control lives there as a guardrail (Presidio + categorical label block),
   not as a separate service. The corollary: high-sensitivity members must have direct-outbound tools
   disabled in their Charter so LiteLLM stays the only door. This is the Owner's offload, adopted.
4. **RESOLVED — "single gateway" reframed as one shared control plane.** A Hermes profile *is* a
   gateway, so distinct member bots mean N gateways — but they share one LiteLLM gatekeeper, one
   Kanban board, one Honcho workspace and one host. The system is operationally singular while members
   stay first-class identities. Cost: N bot tokens + N light processes. A lighter variant could still
   collapse low-sensitivity members behind a shared bot and reserve dedicated bots for the delicate
   ones (Confessor, Physician, Steward) — a deployment dial, not an architecture change.
5. **Sensitivity as a first-class label vs. emergent from Charters.** I make delicacy an explicit
   taint passed as request metadata and enforced at the LiteLLM chokepoint. An alternative keeps it
   implicit in member behaviour. I chose explicit because SR-5/SR-6 demand the gatekeeper judge data
   "by what it truly is, not who carries it."
6. **Memory contradiction engine.** I add a temporal knowledge graph (Graphiti/Zep) specifically for
   MR-5/MR-7. A lighter design would handle contradictions purely at Chamberlain reasoning time
   without a dedicated graph store.
7. **Servant engine (LM Studio now, vLLM later).** LM Studio is the default servant behind LiteLLM;
   if multi-member concurrency outgrows it, vLLM swaps in behind the same seam. The "many hands, one
   bench" story (§2.3) is the signal to make that swap.

---
*Tooling currency: choices reflect the open-source landscape as of mid-2026 and should be
re-validated at build time. Primary backbone: Hermes Agent (MIT). Memory: Honcho. OpenClaw noted as
an alternative, not the base.*
