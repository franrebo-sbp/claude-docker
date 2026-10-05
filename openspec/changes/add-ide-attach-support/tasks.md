## 1. Image attach metadata

- [x] 1.1 Add a `LABEL devcontainer.metadata` to the Dockerfile next to the existing `ENTRYPOINT`/`CMD`, with `remoteUser` set to `claude` and `customizations.vscode.extensions` set to `["anthropic.claude-code"]`. Single-line JSON so the label value stays a valid one-liner.
- [x] 1.2 Comment the label with the reason it exists: `docker exec` bypasses `ENTRYPOINT`, so an attach never inherits the `runuser` drop; `/root` is chowned to `HOST_UID` and the container has no `CAP_DAC_OVERRIDE`, so container root cannot write it.
- [x] 1.3 Note the `HOST_UID=0` exception in the same comment, so a future reader does not "fix" the label by inventing a build-time `claude` user.
- [x] 1.4 Deliberately omit `workspaceFolder`: `WORKDIR /workspaces` already covers every session and an image-scoped label must not pin one repo.

## 2. Container naming

- [x] 2.1 Rename `gh_sid` to `session_sid` in `run.sh` and update its comment: the suffix now identifies the session, not the gh sidecar. Update both `GH_PROXY_NETWORK` and `GH_PROXY_SIDECAR` uses.
- [x] 2.2 Derive `AGENT_CONTAINER="claude-docker-<workspace>-<session_sid>"` from `SEEN_NAMES[0]`, sanitising the basename to `[a-zA-Z0-9_.-]` and prefixing it when it would otherwise start with a non-alphanumeric character.
- [x] 2.3 Add `--name "$AGENT_CONTAINER"` to the agent `run` invocation. Unconditional, and with no startup message.

## 3. Documentation

- [x] 3.1 Add a `docs/usage.md` section covering the attach workflow: name the container, attach, confirm `whoami` returns `claude`, run `claude`, `/ide` connects.
- [x] 3.2 Document the `mkdir: Permission denied` failure mode and its cause, so an operator who hits it on an older image can identify it.
- [x] 3.3 Document the `HOST_UID=0` exception and the per-machine `"remoteUser": "root"` override.
- [x] 3.4 State why the host-IDE alternative is not offered, referencing the threat model's framing.
- [x] 3.5 Add the IDE server and its extensions to the threat model's runtime code-fetch bullet: attaching downloads unpinned vendor code into the container, and it persists in `claude-code-root`.

## 4. Verification

- [x] 4.1 `shellcheck run.sh entrypoint.sh smoke/*.sh` passes.
- [x] 4.2 `hadolint --config .hadolint.yaml Dockerfile` passes.
- [x] 4.3 `docker build -t claude-code:local .` succeeds, and `docker inspect -f '{{index .Config.Labels "devcontainer.metadata"}}' claude-code:local` emits parseable JSON naming `claude`.
- [x] 4.4 Start a session and confirm the container name matches `claude-docker-<workspace>-<suffix>` and that `docker ps --filter name=claude-docker-` finds it.
- [x] 4.5 Start two concurrent sessions on the same workspace and confirm both start with distinct names.
- [x] 4.6 Attach an IDE to a running session on a fresh machine with no `imageConfigs/` entry. Confirm `whoami` reports `claude`, the extension appears in the container's extension directory, and no permission error occurs during server install.
- [x] 4.7 In the attached session run `claude`, then `/ide`. Confirm it reports a connection, that `/root/.claude/ide/` holds a lockfile owned by `claude`, and that no `CLAUDE_CODE_IDE_*` variable is set in the container.
- [x] 4.8 Confirm by inspection that no code path in `run.sh` mounts a host `ide/` directory or forwards a `CLAUDE_CODE_IDE_*` variable.
- [x] 4.9 `IMAGE=claude-code:local bash smoke/smoke.sh --uid="$(id -u)" --optins=aws,glab,tfe` passes.
- [x] 4.10 `openspec validate add-ide-attach-support --strict` passes and every task above is checked off.
