# Chamberlain — Copilot Instructions

This is a **meta-repository**. It contains the architecture specification and wires the four pillars in as git submodules. Application code lives **inside the submodules**, not here.

Authoritative architecture reference: [`docs/specification.md`](../docs/specification.md). Read it before making non-trivial changes. Actors, scenarios, and requirements live in [`docs/requirements.md`](../docs/requirements.md).

## Repository shape

```
chamberlain/              ← this meta-repo (quintindk/chamberlain)
├── docs/
│   ├── specification.md  ← the source of truth for the architecture
│   └── requirements.md   ← actors, scenarios, and requirements
├── catchpole/            ← submodule → quintindk/catchpole (Gateway, LiteLLM)
├── scribe/               ← submodule → quintindk/scribe    (RAG MCP, Qdrant)
├── miller/               ← submodule → quintindk/miller    (Ingestion cron)
└── bailiff/              ← submodule → quintindk/bailiff   (Client MCP)
```

Each submodule is an **independent git repo on its own `main` branch**. The meta-repo only tracks the pinned commit SHA of each.

## Working with submodules

Always clone with submodules, or hydrate after a plain clone:

```sh
git clone --recurse-submodules https://github.com/quintindk/chamberlain
# or, post-clone:
git submodule update --init --recursive
```

**To change code in a pillar:**

1. `cd <pillar>` (you are now in that pillar's own repo).
2. Make changes, commit, push to the pillar's `origin/main`.
3. `cd ..` back to the meta-repo.
4. `git add <pillar> && git commit -m "Bump <pillar> to <short-sha>"` to advance the pinned SHA.
5. Push the meta-repo.

Never commit changes from inside a pillar via the meta-repo's index — the meta-repo only stores submodule pointers, not file contents.

## Architecture in one paragraph

A coding agent (e.g. Copilot CLI) talks to **Bailiff** (client-side MCP, single `ask` tool). Bailiff POSTs to OpenAI `/v1/responses` on **Catchpole** (LiteLLM proxy) with an MCP tool block pointing at Scribe. Catchpole's `catchpole/auto` router decides LOCAL (LM Studio) vs CLOUD (GitHub Copilot) per request, but forces LOCAL whenever the request carries MCP tools so private archive content never escapes. LM Studio has **Scribe** (Qdrant MCP over streamable-http) attached at the inference layer for RAG over the codebase. **Miller** runs out-of-band on cron, diffing repos and upserting embeddings into the same Qdrant DB that Scribe reads from. The net effect: the agent receives a synthesised answer, never raw vector chunks, saving client context tokens. The pillars are named after a Knight Court of the Domain (KCD) administrative theme — keep that vocabulary in user-facing copy.

Request path: `Agent → Bailiff → Catchpole → LM Studio ↔ Scribe ↔ Qdrant`
Ingestion path: `GitHub → Miller → Qdrant`

## Per-pillar conventions

- **Catchpole** is the existing reference implementation. Mirror its conventions when scaffolding the others:
  - Run via `docker compose up -d` from the pillar root.
  - Config split: a YAML for the runtime (`litellm_config.yaml`) + a Python module (`catchpole.py`) mounted into the container as a custom provider.
  - All tunables come from env vars with a `CATCHPOLE_*` prefix and sensible defaults; document them in the pillar's `README.md`.
  - `.env.example` committed, `.env` git-ignored.
  - Logging via a small `_log()` helper to stderr with a `[<pillar>]` prefix.
- **Scribe** and **Bailiff** both expose MCP via **FastMCP with streamable-http transport** (the legacy SSE transport is deprecated and not reliably picked up by current MCP clients). Bailiff exposes a single tool `ask(query: str) -> str` returning synthesised markdown.
- **Miller** is a lightweight Python container running cron (10-minute cadence), based on `maholick/github-qdrant-sync`. It must be idempotent — re-runs on unchanged repos are a no-op.

## Commands (meta-repo level)

There is no build, test, or lint at the meta-repo level. The only meta-repo operations are submodule pointer bumps and editing `docs/specification.md` / `docs/requirements.md` / `README.md`.

Pillar-level commands live in each pillar's `README.md`. Catchpole today:

```sh
cd catchpole
cp .env.example .env   # then edit secrets
docker compose up -d   # proxy on http://localhost:4000
```

## Editorial conventions

- British English in prose (`synchronisation`, `synthesised`).
- No en-dashes (`–`) or em-dashes (`—`); use hyphens, commas, or parentheses. They mangle in terminals and copy/paste.
- Markdown paragraphs stay on a single line (no hard-wrapping); renderers handle the wrap, copy-paste targets like Confluence/Teams do not.
- Keep the KCD pillar names (Catchpole, Scribe, Miller, Bailiff) as the canonical identifiers in code, configs, and docs.
