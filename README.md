# Agent Workspace Bundles — Study

## The question

> What does it take to package everything an agent did in a session (the
> files it made, the code it ran, the tools and APIs it used, the environment
> it ran in, and the secrets it needed) as one folder that can be saved,
> shared, and opened somewhere else, with the secrets left behind?

Two halves:

1. **The bundle.** How existing formats describe a folder of work, its
   provenance, its tool configuration, and its secrets.
2. **The sandbox.** What rebuilds and runs the environment the work came from:
   container runtimes, microVMs, user-space kernels, and WebAssembly. Docker is
   no longer the only answer, and for agent sandboxes it is rarely the fastest
   or safest one.

Home of the question: the flave master spec, §5.7 (the sandbox directories),
§5.8 (`config/`, `.env`, `.secrets`); the exploration
[`Inspiration-for-the-Sandbox-and-Settings-Design.md`](https://github.com/lossless-group/flave/blob/development/context-v/explorations/Inspiration-for-the-Sandbox-and-Settings-Design.md); and the
exploration
[`A-Flave-Is-the-Zip-for-an-Agent-Session.md`](https://github.com/lossless-group/flave/blob/development/context-v/explorations/A-Flave-Is-the-Zip-for-an-Agent-Session.md).

## Pin criterion

- **(a) It defines or runs part of the bundle.** A format a flave could
  borrow from, or a runtime that could rebuild a flave's `env/`.
- **(b) Open source** under a license we can read and build on.
- **(c) Live.** Pushed to in 2026.

All pins are shallow (`--depth 1`); several of these repos are large (gVisor
is over 1 GB). First pinned to the upstream HEAD of 2026-11-19; refreshed to
upstream HEAD on 2026-11-23 (table below). Fetch one before reading it:

```sh
git submodule update --init --depth 1 <name>    # e.g. gvisor
```

### Pins last refreshed 2026-11-23

Seven of thirteen repos had no new commits. Commit counts come from the GitHub
compare API, because the shallow clones cannot count history locally. No repo
cut a release inside its window; "latest release" is the newest GitHub release
at refresh time.

| Repo | Pinned commit date, old → new | Commits | Latest release |
|---|---|---|---|
| ro-crate | 2026-11-17 → 2026-11-20 | 4 | 1.3.0 (2026-06-22) |
| agent-file | 2026-03-24 (unchanged) | 0 | none published |
| mcpb | 2026-04-22 (unchanged) | 0 | v2.1.2 (2025-12-04) |
| datapackage | 2026-08-25 (unchanged) | 0 | v2.0 (2024-10-23) |
| devcontainers-spec | 2026-03-20 (unchanged) | 0 | none published |
| containerd | 2026-11-19 → 2026-11-22 | 2 | v2.4.2 (2026-10-13) |
| podman | 2026-11-19 → 2026-11-21 | 6 | v6.2.0 (2026-11-10) |
| apple-container | 2026-11-13 (unchanged) | 0 | 1.7.1 (2026-11-06) |
| gvisor | 2026-11-20 → 2026-11-23 | 51 | release-20261116.0 (2026-11-17) |
| firecracker | 2026-11-18 → 2026-11-20 | 10 | v1.18.0 (2026-11-09) |
| e2b | 2026-11-18 (unchanged) | 0 | @e2b/cli 2.24.0 (2026-11-13) |
| cloudflare-sandbox-sdk | 2026-11-02 (unchanged) | 0 | @cloudflare/sandbox 1.0.3 (2026-10-22) |
| pyodide | 2026-11-19 → 2026-11-20 | 3 | 314.0.8 (2026-11-13) |

What moved, for this study's question:

- **gVisor** added a `passt` mode to its network gofer for rootless sandbox
  networking (`runsc/gofer/net/passt.go`). passt opens host sockets for the
  sandbox; netgofer applies the same egress filter as proxy mode to outbound
  packets, and the file notes the filter only holds if passt runs without
  address translation. This bears on the isolation row: a rootless sandbox
  with filtered egress.
- **Firecracker** hardened the MMDS (metadata service) HTTP parsing in
  `src/vmm/src/dumbo/tcp/` and fixed dirty-page tracking so a failed full
  snapshot no longer loses pages (`src/vmm/src/vstate/memory.rs`). Snapshot
  and restore is the mechanism a flave would use to resume a sandbox.
- **ro-crate**: website and tools-list edits only (`docs/_data/tools_list.yml`).
  The spec is unchanged at 1.3.0. The local commit count of 2194 is an artifact
  of the shallow clone being unshallowed; the real gap is 4 commits.
- **containerd, podman, pyodide**: small fixes, nothing that changes the
  entries above. Podman's repo now lives at `podman-container-tools/podman`
  (the old `containers/podman` URL redirects there); `.gitmodules` already
  uses the new URL.
- No license changes and no archived repos.

### Added 2026-11-26

- **Paperclip** (`paperclipai/paperclip`), pinned at `dadbc49` (2026-11-26);
  latest release `v2026.1117.0` (2026-11-18). Meets all three criteria: it
  defines a bundle format (the Agent Companies spec,
  `docs/companies/companies-spec.md`) and drives several of the runtimes above
  through plugins. Its profile is the study's first:
  [`Profile__Paperclip.md`](./context-v/profiles/Profile__Paperclip.md).

## What we are looking at, repo by repo

Working checklist for each entry:

- **The manifest.** One file at the root, or metadata spread across files?
- **Provenance.** Can it say where a file came from and what produced it?
- **Secrets.** How does it declare a secret it needs without holding it?
- **Rebuild.** How small is the recipe, and how long from recipe to running?
- **Isolation.** What stops code from a stranger's bundle from hurting the host?

### Bundle formats

| Bet | Entry | License |
|---|---|---|
| A folder becomes a crate with one `ro-crate-metadata.json` describing every file and its origin, plus an optional HTML preview | [RO-Crate](./ro-crate) | Apache-2.0 |
| An agent's memory, tools, prompts and model settings in one file; secrets nulled on export | [Letta Agent File (`.af`)](./agent-file) | Apache-2.0 |
| A local MCP server plus `manifest.json`; sensitive `user_config` fields go to the OS keychain | [MCP Bundles (`.mcpb`)](./mcpb) | See repo (no SPDX id) |
| Typed data files with schemas, described by a `datapackage.json` | [Frictionless Data Package](./datapackage) | Unlicense |
| `devcontainer.json`: a small recipe for a full development environment | [Dev Container spec](./devcontainers-spec) | CC-BY-4.0 |

### Bundle format and orchestrator

| Bet | Entry | License |
|---|---|---|
| A whole agent "company" as a markdown package (`COMPANY.md`, `TEAM.md`, `AGENTS.md`, `PROJECT.md`, `TASK.md` around Agent Skills' `SKILL.md`), sources pinned to commit + sha256, secrets exported as declared inputs with no values; plus encrypted workspace snapshots for resume and nine sandbox-provider plugins (E2B, Cloudflare, Daytona, Modal, Kubernetes, …). [Profile](./context-v/profiles/Profile__Paperclip.md) | [Paperclip](./paperclip) | MIT |

### Sandbox runtimes

| Bet | Entry | License |
|---|---|---|
| The container runtime under Docker and Kubernetes, usable without Docker (`nerdctl` is its Docker-compatible CLI) | [containerd](./containerd) | Apache-2.0 |
| Daemonless, rootless containers with a Docker-compatible CLI | [Podman](./podman) | Apache-2.0 |
| Apple's container tool for macOS: each container runs in its own lightweight VM | [Apple `container`](./apple-container) | Apache-2.0 |
| A user-space kernel that intercepts system calls, for running untrusted code; rootless networking through passt as of November 2026 | [gVisor](./gvisor) | Apache-2.0 |
| MicroVMs that boot in well under a second; the isolation behind AWS Lambda and many agent sandboxes | [Firecracker](./firecracker) | Apache-2.0 |
| Open-source cloud sandboxes for agent-run code, built on Firecracker | [E2B](./e2b) | Apache-2.0 |
| Sandboxes on Cloudflare Containers, driven from a Worker | [Cloudflare Sandbox SDK](./cloudflare-sandbox-sdk) | See repo (no SPDX id) |
| CPython compiled to WebAssembly: run Python in a browser with nothing installed | [Pyodide](./pyodide) | MPL-2.0 |

## Beyond Docker: the landscape

The pinned runtimes, plus what else exists, by how much they isolate:

| Layer | Examples | Why it matters for a flave |
|---|---|---|
| **Low-level OCI runtimes** | `runc` (Docker's default), `crun` (C; faster start, less memory), `youki` (Rust) | What actually starts a container. Swappable under containerd or Podman |
| **Container engines** | containerd + `nerdctl`, Podman, Docker | Same OCI images, different daemons and security defaults. Podman runs rootless with no daemon |
| **Desktop VMs for containers** | Apple `container`, Colima (open source), OrbStack (closed source), Docker Desktop | On a Mac, containers always run inside a Linux VM; these make that VM fast. Apple's gives each container its own VM |
| **Sandboxed kernels** | gVisor | Stronger isolation than a container, without a full VM |
| **MicroVMs** | Firecracker, Cloud Hypervisor, Kata Containers | A real VM boundary that starts in a fraction of a second. The default for running code nobody has reviewed |
| **Hosted agent sandboxes** | E2B, Cloudflare Sandbox, Daytona, Modal, Vercel Sandbox | What chat products and agent frameworks call to run code. Mostly Firecracker or gVisor underneath |
| **WebAssembly** | Pyodide, Wasmtime, WasmEdge, StackBlitz WebContainers | Runs in a browser or a tiny runtime with nothing installed. A reader could re-run a flave's Python script from a web page |
| **Environments, not isolation** | Nix flakes, devenv, Flox, `uv` | A different question from every row above: *what is installed*, not *what the code can reach*. A script in a Nix shell runs as you, with your files and keys. Pair with a row above to run untrusted code; flave-ai itself uses a Nix flake |

The likely flave answer: `env/` stays a **recipe** (lockfiles, optionally a
`devcontainer.json` or OCI image reference), and the host chooses the runtime.
A received flave's scripts run in a microVM, gVisor, or WebAssembly, never
directly on the reader's machine (master spec §10, M6).

## Not pinned (yet)

- **Runtimes:** runc, crun, youki, Kata Containers, Cloud Hypervisor, Colima,
  Wasmtime, WasmEdge, `kubernetes-sigs/agent-sandbox` (used by Paperclip's
  Kubernetes plugin). Listed above; pin when a question needs their source.
- **Hosted sandboxes:** Daytona, Modal, Vercel Sandbox, OrbStack (closed
  source), StackBlitz WebContainers (closed source).
- **Formats:** BagIt (checksum manifests for archives), Nix flakes and devenv.
- **git-annex** (big files as pointers in git) is already pinned in
  `study-sync-and-content-version-control`.

## Reading checklist

- [ ] RO-Crate vs. flave's `_sources.yaml` + `runs/`: could a flave *be* an
      RO-Crate, with `ro-crate-metadata.json` generated from them?
- [ ] `.mcpb` and `.af`: exact placeholder syntax and export behavior for
      secrets; adopt one for `${secret:NAME}` rather than invent it.
- [ ] Dev Container spec: is `devcontainer.json` the right optional file for
      `env/`, next to plain lockfiles?
- [ ] For each runtime: cold-start time, isolation boundary, and whether it
      runs on macOS, Linux and in a browser.
- [ ] E2B and the Cloudflare Sandbox SDK: how a chat-style agent gets files in
      and out of a sandbox, which is where a flave export would hook in.
- [ ] Paperclip's `sources[]` object vs. flave's `_sources.yaml`: adopt it
      as-is? And should a flave also be a valid Agent Companies package?
- [ ] Paperclip's `e2b/`, `cloudflare/` and `daytona/` sandbox plugins as
      working answers to the files-in-and-out question above.
