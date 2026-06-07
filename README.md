# smoothnas-plugin-aimee

SmoothNAS plugin manifests for [aimee](https://github.com/RakuenSoftware/aimee) —
published as one catalog entry that ships three plugins:

| Plugin | Image (GHCR) | What it is |
| --- | --- | --- |
| **aimee-server** | `ghcr.io/rakuensoftware/aimee-server` | The agent/memory broker. Serves the `/v1` HTTP API backed by a self-contained SQLite store (DB1). Runs standalone. |
| **aimee-kb** | `ghcr.io/rakuensoftware/aimee-kb` | The knowledge base: shared, vector-backed memory (DB2 + pgvector) over `/v1`. **Self-contained** — bundles its own pgvector Postgres + embedder. |
| **aimee-combined** | `ghcr.io/rakuensoftware/aimee-server-kb` | Both binaries co-located in one container — the full `/v1` surface with shared/vector memory. **Self-contained** — bundles its own pgvector Postgres + embedder. |

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

- **Just the broker, nothing else to run** → **aimee-server** (standalone,
  SQLite only). Federate to a kb later by setting `AIMEE_KB_API_URL`.
- **Shared/vector memory with zero external setup** → **aimee-kb** (kb only) or
  **aimee-combined** (server + kb in one container). Both bundle their Postgres
  and embedder.

## Storage & bundled services

aimee-kb and aimee-combined are compose-style plugins (`services:`): they bring
up their **own** pgvector Postgres and embedder as sibling services — there is no
external database or embedder to stand up. The kb reaches its siblings over the
plugin bridge via service-discovery tokens, so nothing needs configuring.

All three plugins use **tier-bound** volumes (Postgres database, embedder model
cache, kb/server state, DB1, mirror workspaces): at install you choose which
storage tier holds them, keeping the data off the small OS/root device. The
per-volume `slot` (HDD) is the home tier within that pool; smoothfs caches hot
data on faster tiers automatically.

The containers run as **uid 1000** to match SmoothNAS tiers, which present a
uniform owner of uid 1000 — the stock images would otherwise drop to their own
user and be unable to access their data dirs. This requires the runtime from
**SmoothNAS v0.1.20+** (which honours the container user).

`LLM_ENDPOINT` / `LLM_MODEL` are optional: set them to an OpenAI-compatible
endpoint to enable the kb's candidate-synthesis and curator passes. Left blank,
those passes stay disabled (they fail closed without an endpoint).

aimee-server has no bundled services (self-contained SQLite DB1); it optionally
federates to a kb via `AIMEE_KB_API_URL` (+ `AIMEE_KB_API_BEARER_TOKEN`).

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

## Releasing (automatic)

Versioning is automatic. Every push to `main` (except doc-only changes) runs
[`.github/workflows/release.yml`](.github/workflows/release.yml), which:

1. computes the next semver **patch** from the highest existing `v*` tag,
2. stamps that version into each manifest's `metadata.version`, and
3. cuts a `v<version>` GitHub release with the three
   `smoothnas-plugin-aimee-*.yaml` assets — exactly what the SmoothNAS catalog
   fetches from the latest release.

So merging a change to `main` ships a new version on its own; no manual tagging.
To bump the **minor/major** instead of the patch, run the workflow manually
(Actions → release → *Run workflow*) and pass an explicit `version` (e.g.
`0.2.0`).

The committed manifests keep a base `metadata.version`; the **released assets**
are stamped with the actual release version at build time, so the version the
appliance sees always matches the release tag.

---

These manifests reference the [aimee](https://github.com/RakuenSoftware/aimee)
project and are distributed under the same terms.
