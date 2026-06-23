# Chamberlain

<p align="center">
  <img src="docs/images/chamberlain.jpg" alt="Chamberlain" width="480">
</p>

A four-pillar AI ecosystem providing highly contextual, agentic capabilities to a local or cloud-based coding agent. Decouples client execution, API routing, inference, and memory ingestion into discrete microservices.

See [`docs/specification.md`](./docs/specification.md) for the full architecture and [`docs/requirements.md`](./docs/requirements.md) for the actors, scenarios, and requirements.

## Pillars

| | Pillar | Role | Repo |
| :---: | --- | --- | --- |
| <img src="docs/images/catchpole.jpg" alt="Catchpole" width="80"> | **Catchpole** | The Gateway | [`catchpole/`](https://github.com/quintindk/catchpole) |
| <img src="docs/images/scribe.jpg" alt="Scribe" width="80"> | **Scribe** | The Record Keeper | [`scribe/`](https://github.com/quintindk/scribe) |
| <img src="docs/images/miller.jpg" alt="Miller" width="80"> | **Miller** | The Background Grinder | [`miller/`](https://github.com/quintindk/miller) |
| <img src="docs/images/bailiff.jpg" alt="Bailiff" width="80"> | **Bailiff** | The Delegate | [`bailiff/`](https://github.com/quintindk/bailiff) |
| | **Crier** | The Town Voice (embedded NPU SLM library) | [`crier/`](./crier) — scaffolded in-tree, pending repo |

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
