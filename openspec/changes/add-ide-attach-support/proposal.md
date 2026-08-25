## Why

Attaching an IDE to a running `claude-docker` session is the only way to get Claude Code's `/ide` integration without punching a hole in the container boundary. The alternative, pointing the in-container agent at an IDE running on the host, requires bind-mounting the host's `~/.claude/ide` lockfile directory (exposing the auth token of every open editor window) and giving the container a reachable WebSocket into a host process. Attaching inverts that: the IDE server, the extension, the lockfile, and the agent all live inside the container, and nothing crosses the boundary.

Today the attach fails, for two reasons that are both invisible from the error message:

1. `docker exec` never runs `ENTRYPOINT`, so an attaching IDE bypasses `entrypoint.sh`'s privilege drop and lands as container root. But `entrypoint.sh` has already chowned `/root` to `HOST_UID`, and `run.sh` starts the container with `--cap-drop ALL` and no `DAC_OVERRIDE`. Container root therefore holds `DAC_READ_SEARCH` only: it can stat and read its way through the IDE server's preflight checks and then dies on the first `mkdir` with a bare `Permission denied`. Even if the server did install, the extension would write a `0600` lockfile owned by root that the agent, running as `HOST_UID`, could not read.
2. The agent container is unnamed, so it appears in the IDE's attach picker under a random Docker name (`frosty_raman`) with nothing tying it to the workspace it is serving.

Both are fixed in the image and the wrapper, so a fresh clone works without per-machine editor configuration.

## What Changes

- Add a `devcontainer.metadata` label to the image declaring `remoteUser: claude`, the unprivileged account `entrypoint.sh` creates at runtime with the invoking user's uid. An attaching IDE that honours the label execs as the same uid the agent runs as, so the IDE server can write `/root/.vscode-server` and the lockfile it produces is readable by the agent.
- Have the label also request the `anthropic.claude-code` extension, so it installs into the container rather than the user's local editor. Installing it locally is the default and produces a misleading "Installed extension to VS Code" success with no lockfile behind it.
- Name the agent container `claude-docker-<workspace>-<session>`, derived from the first workspace's basename and the existing per-session identity, so it is greppable (`docker ps --filter name=claude-docker-`) and self-describing in an attach picker. Unique per session, so concurrent sessions do not collide.
- Document the attach workflow in `README.md`, including the `Permission denied` failure mode, its cause, and why the host-IDE alternative is not offered.

Not in scope: any wrapper flag, mount, or env var for connecting the in-container agent to an IDE running on the host. That path is explicitly rejected below, not merely unimplemented.

## Capabilities

### New Capabilities

- `ide-attach`: the image and wrapper affordances that let an IDE attach to a running agent container and drive Claude Code's `/ide` integration entirely inside it, plus the standing prohibition on the host-IDE alternative.

### Modified Capabilities

None. The label is additive metadata and `--name` adds an argument to the existing `run` invocation; no documented requirement of another capability changes.

## Impact

- `Dockerfile` — one `LABEL` near the existing `ENTRYPOINT`. Metadata only: no new layer content, no new package, no change to the runtime process tree or the capability set.
- `run.sh` — renames the session identity variable from `gh_sid` to `session_sid` (it is no longer gh-specific), derives a container name from it, and adds `--name` to the agent `run`. No change to mounts, env, capabilities, or the privilege drop.
- `README.md` — a new section covering the attach workflow, and the security rationale for preferring it over a host IDE.
- No change to the security posture. The privilege drop, capability set, and credential opt-ins are untouched; the label narrows what an attach can do (unprivileged instead of root) rather than widening it.
- Users on a root host (`HOST_UID=0`) are a documented exception: `entrypoint.sh` skips user creation on that path, so `claude` does not exist and the label's `remoteUser` cannot resolve. Attaching as root is correct there, and the README names the one-line per-machine override.
