# smoothnas-plugin-aimee

SmoothNAS plugin manifests for [aimee](https://github.com/RakuenSoftware/aimee) —
published as one catalog entry that ships three plugins:

| Plugin | Image (GHCR) | What it is |
| --- | --- | --- |
| **aimee-server** | `ghcr.io/rakuensoftware/aimee-server` | The agent/memory broker. Serves the `/v1` HTTP API backed by a self-contained SQLite store (DB1). Runs standalone. |
| **aimee-kb** | `ghcr.io/rakuensoftware/aimee-kb` | The knowledge base: shared, vector-backed memory (DB2 + pgvector) over `/v1`. Needs an **external** Postgres + embedder. |
| **aimee-combined** | `ghcr.io/rakuensoftware/aimee-server-kb` | Both binaries co-located in one container — the full `/v1` surface with shared/vector memory. Needs an **external** Postgres + embedder. |

These are [SmoothNAS plugin manifests](https://github.com/JBailes/SmoothNAS)
(`smoothnas.io/v1`, `kind: Plugin`). SmoothNAS runs each as a single managed LXC
container. The Docker images are built and published from the aimee repo
(`Dockerfile.server`, `Dockerfile`, `Dockerfile.combined`).

## Install

In the SmoothNAS UI, add this catalog repo: **`RakuenSoftware/smoothnas-plugin-aimee`**.
SmoothNAS reads the latest GitHub release and surfaces all three
`smoothnas-plugin-*.yaml` assets. Install the one(s) you want and set the config
fields described below.

You can also sideload a single manifest by URL or local file — grab any of the
files under [`manifests/`](manifests/).

## Which one do I want?

- **Just the broker, nothing external to run** → **aimee-server** (standalone,
  SQLite only). Federate to a kb later by setting `AIMEE_KB_API_URL`.
- **Shared/vector memory, and you already run (or will run) Postgres+embedder**
  → **aimee-kb** (kb only) or **aimee-combined** (server + kb in one container).

## External dependencies (aimee-kb / aimee-combined)

SmoothNAS plugins are **single-container**. aimee-kb and aimee-combined do **not**
bundle their datastores — they require:

1. **Postgres with pgvector** (DB2) — e.g. `pgvector/pgvector:pg16`.
2. **An embedder service** — the aimee embedder image, serving `/health` + the
   embedding API on `:8080`.

Run those on your own infra (another host/container) reachable from the SmoothNAS
box, then point the plugin at them via config:

- `AIMEE_DB2_URL` — e.g. `postgresql://aimee:PASS@10.0.0.6:5432/aimee_shared`
- `AIMEE_EMBEDDER_URL` — e.g. `http://10.0.0.6:8080`

The manifest defaults (`postgres`, `embedder` hostnames) come from the upstream
`compose.yaml` and only resolve inside that compose network — **override them**.

`LLM_ENDPOINT` / `LLM_MODEL` are optional: set them to an OpenAI-compatible
endpoint to enable the kb's candidate-synthesis and curator passes. Left blank,
those passes stay disabled (they fail closed without an endpoint).

aimee-server has no external dependencies in its default (standalone) shape.

## Ports

Each plugin's `/v1` API port is published on the SmoothNAS host (`hostExpose`),
matching the upstream compose port mapping:

- aimee-server → `:8740`
- aimee-kb → `:8741`
- aimee-combined → `:8740` (the in-container kb stays on loopback `:8741`)

The `/v1` API is a bearer-authenticated JSON API, not a browser UI, so no nginx
route / iframe embed is declared.

## Caveats

### Inbound bearer token
The server's inbound `/v1` bearer token is baked into the image's `aimee.yaml`
as the development default **`aimee-local-dev`**, and there is no environment
override for it. Treat these plugins as trusted-LAN deployments, or publish your
own image with a hardened `aimee.yaml` (and TLS termination) for anything
exposed beyond a trusted network.

### Stack rlimit (aimee-server / aimee-combined)
The server's worker threads need a **64 MB stack**; a small default stack rlimit
can overflow on startup and segfault the process (the upstream Dockerfiles
document `--ulimit stack=67108864`). The plugin manifest has no `ulimits` field,
so if the container crashes on startup, install the optional operator profile in
[`profiles/aimee-stack.yaml`](profiles/aimee-stack.yaml):

```sh
sudo cp profiles/aimee-stack.yaml /etc/smoothnas/plugin-profiles.d/aimee-stack.yaml
```

then add it to the manifest before installing:

```yaml
profiles:
  - aimee-stack
```

Drop the profile file in **first** — referencing a profile that isn't present on
the host fails the install with `profile "aimee-stack" not in catalog`.

## Version pinning

The manifests track `:latest`. For reproducible installs, edit `artifact.image`
to a release tag (e.g. `:0.2.1`) or add an `artifact.digest: sha256:...`.

## Releasing

Pushing a `v*` tag runs [`.github/workflows/release.yml`](.github/workflows/release.yml),
which creates a GitHub release and attaches the three manifests as
`smoothnas-plugin-aimee-*.yaml` assets — exactly what the SmoothNAS catalog
fetches.

---

These manifests reference the [aimee](https://github.com/RakuenSoftware/aimee)
project and are distributed under the same terms.
