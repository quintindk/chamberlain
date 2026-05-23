# Chamberlain

A four-pillar AI ecosystem providing highly contextual, agentic capabilities to a local or cloud-based coding agent. Decouples client execution, API routing, inference, and memory ingestion into discrete microservices.

See [`specification.md`](./specification.md) for the full architecture.

## Pillars

| Pillar    | Role                   | Repo                                                |
| --------- | ---------------------- | --------------------------------------------------- |
| Catchpole | The Gateway            | [`catchpole/`](https://github.com/quintindk/catchpole) |
| Scribe    | The Record Keeper      | [`scribe/`](https://github.com/quintindk/scribe)       |
| Miller    | The Background Grinder | [`miller/`](https://github.com/quintindk/miller)       |
| Bailiff   | The Delegate           | [`bailiff/`](https://github.com/quintindk/bailiff)     |

Each pillar lives in its own repository, wired in here as a git submodule.

## Clone

```bash
git clone --recurse-submodules https://github.com/quintindk/chamberlain
```

Already cloned? Pull submodules with:

```bash
git submodule update --init --recursive
```

## Bring up the rig

The top-level `docker-compose.yaml` orchestrates the four rig-side services (Qdrant + Catchpole + Miller + Scribe) using Docker Compose's `include:` directive. Each pillar's own compose file remains the source of truth.

```bash
# from the repo root
cp catchpole/.env.example catchpole/.env  # then edit secrets
cp miller/.env.example    miller/.env
cp scribe/.env.example    scribe/.env
docker compose up -d
docker compose logs -f
```

Bailiff is intentionally NOT in the top-level compose — it's a client-side component that runs next to the coding agent (typically on a laptop). Bring it up separately from its own folder when needed.

Requires Docker Compose 2.20+ for `include:`.
