## ADDED Requirements

### Requirement: Image-declared attach configuration

The container image SHALL carry a `devcontainer.metadata` label declaring the attach configuration an IDE needs to drive Claude Code's `/ide` integration inside the container. The label SHALL name `claude` as the `remoteUser` and SHALL request the `anthropic.claude-code` extension under `customizations.vscode.extensions`.

`claude` is the account `entrypoint.sh` creates at container start with `useradd -u "$HOST_UID"`, so it resolves to the same uid the agent itself runs as. Its home is `/root` (`-d /root`), which is why the IDE server lands under `/root/.vscode-server` and the extension's lockfile under `/root/.claude/ide`, the same config dir the agent reads. Both halves of the label are load-bearing and for the same reason: an attaching IDE `exec`s a fresh process from the image config and never runs `ENTRYPOINT`, so it inherits neither the privilege drop nor any knowledge of where the agent lives.

Without `remoteUser`, the attach lands as container root. `entrypoint.sh` has already chowned `/root` to `HOST_UID`, and the container runs with `--cap-drop ALL` and therefore no `CAP_DAC_OVERRIDE`, so container root holds `CAP_DAC_READ_SEARCH` only: it can traverse and read `/root` but cannot write it. Without the extension request, the extension installs into the user's local editor instead of the container, where it cannot serve an in-container agent.

The label SHALL NOT declare `workspaceFolder`. The image's `WORKDIR` is `/workspaces`, which is where `run.sh` mounts every workspace and is therefore the one folder correct for any session; it also satisfies the CLI's cwd check for any repo beneath it.

#### Scenario: attach execs as the agent's own uid

- **GIVEN** a running agent container started by `run.sh` with `HOST_UID` set to a non-zero uid
- **WHEN** an IDE that honours `devcontainer.metadata` attaches to it
- **THEN** the IDE's processes run as `claude`, the same uid the agent runs as
- **AND** the IDE server installs under `/root/.vscode-server` without a permission error

#### Scenario: lockfile written by the IDE is readable by the agent

- **GIVEN** an attached IDE with the Claude Code extension active in the container
- **WHEN** the extension writes its lockfile under `/root/.claude/ide/`
- **THEN** the lockfile is owned by `claude`
- **AND** the agent reads it and `/ide` connects, without any `CLAUDE_CODE_IDE_*` override being set

#### Scenario: extension is installed in the container, not locally

- **GIVEN** a user whose local editor does not have the Claude Code extension installed
- **WHEN** they attach to the agent container
- **THEN** `anthropic.claude-code` is installed into the container's extension directory
- **AND** no lockfile discovery depends on the user's local editor

#### Scenario: root host is a documented exception

- **GIVEN** a host whose invoking uid is 0, so `run.sh` passes `HOST_UID=0` and `entrypoint.sh` skips user creation
- **WHEN** an IDE attaches using the label's `remoteUser`
- **THEN** the attach fails because `claude` does not exist in that container
- **AND** the documented remedy is a per-machine `"remoteUser": "root"` override, which is correct on that path because `/root` is never chowned away from root

#### Scenario: host uid collides with an account baked into the image

- **GIVEN** a non-zero `HOST_UID` that already has a passwd entry in the image, so `entrypoint.sh` skips creating `claude`
- **WHEN** an IDE attaches using the label's `remoteUser`
- **THEN** the attach fails because `claude` does not exist in that container
- **AND** the documented remedy is a per-machine `"remoteUser"` override naming the existing account, which carries the same uid as the agent

### Requirement: Agent container is addressable by name

`run.sh` SHALL pass `--name` to the agent container's `run` invocation. The name SHALL be `claude-docker-<workspace>-<session>`, where `<workspace>` is the basename of the first workspace sanitised to Docker's permitted character set (`[a-zA-Z0-9][a-zA-Z0-9_.-]*`) and `<session>` is the per-session suffix already generated for the stage directory. The name SHALL be applied unconditionally, not gated on a flag.

Reusing the stage-directory suffix makes the name unique per session without extra bookkeeping, so concurrent sessions (including two on the same workspace) do not collide, and `--rm` releases the name on exit. The wrapper SHALL NOT print the name at startup; attaching is rare enough that a line on every run would be noise.

#### Scenario: container is self-describing in an attach picker

- **GIVEN** the user runs `claude-docker ~/src/my-repo`
- **WHEN** they list running containers
- **THEN** the agent container's name begins `claude-docker-my-repo-` rather than a random Docker-assigned name
- **AND** `docker ps --filter name=claude-docker-` lists it

#### Scenario: concurrent sessions do not collide

- **GIVEN** one `claude-docker ~/repo` session already running
- **WHEN** the user starts a second `claude-docker ~/repo` session
- **THEN** the second container is named with a different session suffix and starts successfully

#### Scenario: an unusual workspace basename does not break the run

- **GIVEN** a workspace whose basename contains characters Docker does not permit in a container name, or begins with a non-alphanumeric character
- **WHEN** the user runs `claude-docker` against it
- **THEN** the name is sanitised to a permitted form and the container starts

### Requirement: No host-IDE bridge

`run.sh` SHALL NOT provide any flag, mount, or environment forwarding that connects the in-container agent to an IDE running on the host. Specifically it SHALL NOT bind-mount the host's `<config-dir>/ide` directory into the container, and SHALL NOT set `CLAUDE_CODE_IDE_HOST_OVERRIDE`, `CLAUDE_CODE_IDE_SKIP_VALID_CHECK`, or `CLAUDE_CODE_AUTO_CONNECT_IDE`.

The host IDE lockfile directory holds one lockfile per open editor window, each carrying an auth token, so mounting it would hand a session scoped to one repository the credentials of every window the user has open. Combined with a host-reachable transport, it would give the container an authenticated channel into a host process, which contradicts the wrapper's premise that the agent reaches the host only through the workspaces it was passed. The attach topology is the supported path and needs none of it.

#### Scenario: host lockfile directory is never mounted

- **WHEN** `claude-docker` starts a session with any combination of flags
- **THEN** no bind-mount targets `/root/.claude/ide`
- **AND** the container's `/root/.claude/ide` contains only lockfiles written inside the container, if any

#### Scenario: IDE host overrides are never forwarded

- **GIVEN** a host with `CLAUDE_CODE_IDE_HOST_OVERRIDE` set in the environment
- **WHEN** the user runs `claude-docker`
- **THEN** the variable is not forwarded into the container
