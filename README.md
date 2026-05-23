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
