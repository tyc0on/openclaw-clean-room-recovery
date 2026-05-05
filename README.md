# OpenClaw Clean-Room Recovery

This document captures the recovery path learned from the 2026-05-03 OpenClaw outage.

## Failure Lesson

The failed path was trying to repair OpenClaw by modifying global package files and runtime dependency cache behavior under `/usr/lib/node_modules/openclaw`.

That turned a channel outage into a contaminated install with unclear provenance. Future agents must treat that as a known-bad failure path.

## Hard Boundary

Never hand-edit:

- `/usr/lib/node_modules/openclaw`
- `/usr/lib/node_modules/openclaw/dist`
- `/usr/lib/node_modules/openclaw/package.json`
- any generated OpenClaw runtime dependency cache

The only acceptable way to change the installed package is through the package manager.

## Clean-Room Procedure

1. Stop live OpenClaw processes.

```bash
systemctl --user stop openclaw-gateway.service
systemctl --user stop openclaw-memory-guard.timer openclaw-memory-guard.service 2>/dev/null || true
ps -eo pid,ppid,cmd | rg 'openclaw|npm install' | rg -v 'rg'
```

2. Preserve state without deleting it.

```bash
mv /root/.openclaw "/root/.openclaw-backup-$(date -u +%Y%m%dT%H%M%SZ)"
```

3. Reinstall only through npm.

```bash
npm uninstall -g openclaw
npm install -g openclaw@<version>
openclaw --version
```

4. Initialize a blank local gateway.

```bash
openclaw config set gateway.mode '"local"' --strict-json
openclaw doctor --fix --generate-gateway-token --non-interactive --no-workspace-suggestions
```

5. Proof-gate the empty baseline before restoring old state.

Pass criteria:
- Gateway is active for at least 90 seconds.
- Startup reaches `gateway ready`.
- Journal has no plugin register failures.
- No OpenClaw `npm install` process is still running.
- CPU and memory settle after startup.
- `openclaw doctor --non-interactive --no-workspace-suggestions` shows plugin errors at zero.
- `openclaw gateway probe --json` reports `ok: true`, `connect.rpcOk: true`, no `probe_scope_limited` warning, and at least `read_only` capability.

Useful commands:

```bash
systemctl --user status openclaw-gateway.service --no-pager -l
journalctl --user -u openclaw-gateway.service --since '10 minutes ago' --no-pager -o short-iso
openclaw doctor --non-interactive --no-workspace-suggestions
openclaw gateway probe --json
openclaw channels status --json
```

## Update Restart Probe Drift

After an otherwise clean package update, `openclaw update` can report a misleading port conflict if the restarted gateway is alive but the local CLI device token lacks `operator.read`.

Evidence pattern:
- `systemctl --user status openclaw-gateway.service` shows the gateway active on port `18789`.
- `openclaw gateway status --deep` reaches the gateway but shows `Capability: connected-no-operator-scope`.
- `openclaw status --deep --json` reports `gateway.error = "missing scope: operator.read"`.
- `openclaw gateway probe --json` reports a pending scope upgrade request.

Supported repair path:

```bash
openclaw devices list --json
openclaw gateway probe --json
openclaw devices approve <requestId> --token "$(jq -r '.gateway.auth.token' /root/.openclaw/openclaw.json)" --json
openclaw gateway probe --json
```

Do not patch package files for this. The root cause is local operator device auth drift, not a broken listener.

## Restore Rules

Restore operator state only after the blank baseline passes.

Allowed from backup:
- `workspace/SOUL.md`
- `workspace/MEMORY.md`
- `workspace/IDENTITY.md`
- `workspace/USER.md`
- `credentials/`
- selected channel/model/auth config from `openclaw.json`
- device identity files only if needed

Do not restore:
- `plugin-runtime-deps/`
- `node_modules/`
- `.openclaw-runtime-deps.lock`
- `plugins/installs.json`
- generated registries
- old logs or stability bundles
- session transcripts unless explicitly needed for audit

## Agent Operating Principle

When OpenClaw update/startup breaks, do not try clever package surgery.

The correct agent behavior is:
- preserve evidence,
- isolate the broken state,
- establish a clean baseline,
- restore state incrementally,
- validate after each restore step,
- write down the result.
