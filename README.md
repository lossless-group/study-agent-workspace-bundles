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
§5.8 (`config/`, `.env`, `.secrets`) and §5.9 (inspiration), and the
exploration
[`A-Flave-Is-the-Zip-for-an-Agent-Session.md`](https://github.com/lossless-group/flave/blob/development/context-v/explorations/A-Flave-Is-the-Zip-for-an-Agent-Session.md).

## Pin criterion

- **(a) It defines or runs part of the bundle.** A format a flave could
  borrow from, or a runtime that could rebuild a flave's `env/`.
- **(b) Open source** under a license we can read and build on.
- **(c) Live.** Pushed to in 2026.

All pins are shallow (`--depth 1`); several of these repos are large (gVisor
is over 1 GB). Each is pinned to the upstream HEAD of 2026-11-19. Fetch one
before reading it:

```sh
git submodule update --init --depth 1 <name>    # e.g. gvisor
```

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

### Sandbox runtimes

| Bet | Entry | License |
|---|---|---|
| The container runtime under Docker and Kubernetes, usable without Docker (`nerdctl` is its Docker-compatible CLI) | [containerd](./containerd) | Apache-2.0 |
| Daemonless, rootless containers with a Docker-compatible CLI | [Podman](./podman) | Apache-2.0 |
| Apple's container tool for macOS: each container runs in its own lightweight VM | [Apple `container`](./apple-container) | Apache-2.0 |
| A user-space kernel that intercepts system calls, for running untrusted code | [gVisor](./gvisor) | Apache-2.0 |
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
  Wasmtime, WasmEdge. Listed above; pin when a question needs their source.
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
