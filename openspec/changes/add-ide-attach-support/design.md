# Design

## Context

Claude Code's `/ide` integration is a WebSocket MCP transport between the CLI and an editor extension. The extension binds a loopback port and writes `<port>.lock` into `$CLAUDE_CONFIG_DIR/ide/`, carrying the port, the editor's workspace folders, its pid, and an auth token. The CLI scans that directory, matches its cwd against the recorded workspace folders, resolves a host (`CLAUDE_CODE_IDE_HOST_OVERRIDE`, else a WSL gateway probe, else `127.0.0.1`), and connects.

Every one of those steps assumes the CLI and the extension share a filesystem, a network namespace, and a path layout. A containerised agent breaks all three at once, which leaves two possible topologies.

## Goals / Non-Goals

- **Goal:** `/ide` works for a `claude-docker` session without weakening the container boundary.
- **Goal:** works on a fresh clone with no per-machine editor configuration.
- **Non-Goal:** supporting an IDE that runs on the host. Rejected below.
- **Non-Goal:** starting or managing the IDE from `run.sh`. The attach is driven from the editor; the wrapper's job is to make the container attachable.

## Decision: attach the IDE into the container, do not reach out to the host

The host-IDE topology needs three things, all of which erode the boundary:

- A read-only bind-mount of the host's `~/.claude/ide` into the container. That directory holds one lockfile per open editor window, each with an auth token, so a session scoped to one repo gains credentials for every window the user has open.
- `CLAUDE_CODE_IDE_HOST_OVERRIDE` plus host-gateway resolution, giving the container an authenticated channel into a host process. The wrapper's entire premise is that the agent reaches the host filesystem only through the workspaces it was passed.
- `CLAUDE_CODE_IDE_SKIP_VALID_CHECK`, because `run.sh` mounts workspaces at `/workspaces/<basename>` while the host extension reports host paths. Skipping the check makes the connection succeed but leaves it functionally broken: the agent sends container paths to an editor that cannot resolve them, so diffs and diagnostics silently degrade.

There is also a destructive failure mode. The CLI's stale-lockfile sweep calls `process.kill(pid, 0)` and, on any non-WSL platform, unlinks lockfiles whose pid is not alive. A host editor's pid does not exist in the container's pid namespace, so a read-write mount would have the container delete the host's live lockfiles. A read-only mount degrades this to a logged `unlink` failure, but needing that mitigation at all is a signal.

The attach topology has none of these properties. The IDE server runs inside the container under the same kernel-enforced constraints as the agent (`--cap-drop ALL`, `no-new-privileges`, the same mounts), the lockfile is container-local, the port never leaves the network namespace, and paths match because there is only one filesystem in play. The cwd-versus-workspace-folder check passes on its own, so no override is needed.

The cost is honest and worth stating: attaching downloads an IDE server and extensions into the container at runtime, from vendor endpoints, unpinned. That is a new instance of the runtime code-fetch primitive the threat model already tracks for `npx`, `uvx`, and `tfenv install`, and it persists in the `claude-code-root` volume rather than vanishing with the container. The trade is "the sandbox runs more unpinned code inside its own blast radius" against "the sandbox gets a live channel out to the host", and the first is the one this project is built to contain.

## Decision: fix the uid in the image label, not in per-machine editor config

`docker exec` never runs `ENTRYPOINT`; that is a Docker invariant, not something the wrapper can override. So the attach cannot inherit `entrypoint.sh`'s `runuser` drop and must be told which user to exec as.

Three placements were considered:

1. **`USER` in the Dockerfile.** Impossible. The account is created at runtime by `entrypoint.sh` with `useradd -u "$HOST_UID"`, because the uid has to match whoever invoked `run.sh`. It does not exist at build time. PID 1 also has to start as root to chown the persistent volumes and `runuser`, so a build-time `USER` would break the container outright.
2. **Per-machine attached-container config** (`imageConfigs/claude-code%3alocal.json` for VS Code). Works, but every user has to discover it independently, and the filename is a percent-encoded derivation of the image tag that fails silently when guessed wrong.
3. **`devcontainer.metadata` image label.** Ships with the image, applies on every host, and is the mechanism the Dev Containers spec defines for prebuilt images to carry their own attach configuration. Chosen.

The label deliberately does not set `workspaceFolder`. `workspaceFolder` is primarily an attached-config property and its behaviour when supplied via image metadata is not something this project should depend on; more importantly, pinning one path in an image-scoped label would be wrong for a wrapper whose whole point is arbitrary workspaces. `WORKDIR /workspaces` already gives an attaching editor a sensible default, and it is the one path correct for every session, single or multi-repo. It also satisfies the CLI's cwd check for any repo beneath it, since the match is `cwd === folder || cwd.startsWith(folder + sep)`.

### The root-host exception

When `HOST_UID=0`, `entrypoint.sh` returns early and never creates `claude`, so the label names a user that does not exist. This is the correct outcome to leave alone rather than paper over: on that path `/root` is never chowned away from root, so container root still owns it and an attach-as-root writes fine. The two alternatives were both worse. Creating a second uid-0 account named `claude` puts a root-equivalent login in the image for the sake of a label. Creating a genuine unprivileged `claude` there makes the attach unable to write `/root`, trading a clear failure for a confusing one. The README names the per-machine `"remoteUser": "root"` override instead.

## Decision: name the container per session, do not announce it

`--name` is unconditional rather than gated on an `--ide` flag: naming a container is not an IDE feature, it costs nothing, and gating it would mean users only get a findable container if they knew to ask.

The name is `claude-docker-<workspace>-<session>`, where `<workspace>` is the first workspace's basename (sanitised to Docker's `[a-zA-Z0-9][a-zA-Z0-9_.-]*`) and `<session>` is the existing per-session suffix that `mktemp` already generated for the stage directory. Reusing that suffix keeps uniqueness free: concurrent sessions on the same workspace get distinct names with no extra bookkeeping, and `--rm` releases the name on exit. The variable is renamed `gh_sid` to `session_sid` because it now identifies the session rather than the gh sidecar.

The name is not printed at startup. The `--gh` sidecar prints its name because the user needs it to read the audit log, an action that only makes sense mid-session. Attaching is comparatively rare, so a line on every run would be noise for the majority who never attach. `docker ps --filter name=claude-docker-` is documented instead.
