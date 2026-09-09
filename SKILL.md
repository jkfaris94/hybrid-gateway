---
name: hybrid-gateway
description: Set up and troubleshoot a secure hybrid OpenClaw architecture where the Gateway runs on a VPS and a Mac or other local machine is a paired node. Covers Tailscale node pairing (single-use join links), exact-request approval and reapproval, remote node exec with least-privilege allowlists, node reconnect, and SSH as a separate fallback. Use when connecting a local node to a remote Gateway, debugging node connectivity or "reapproval pending", or planning a VPS + local hardware split.
metadata:
  openclaw:
    env:
      - name: OPENCLAW_ALLOW_INSECURE_PRIVATE_WS
        description: Optional, advanced direct-tailnet route only. Set to 1 on the node service to allow ws:// to a non-loopback Tailscale address. Not needed on the preferred Tailscale Serve (wss://) route.
        scope: node-service
        required: false
---

# Hybrid Gateway: VPS + Local Node

The Gateway runs on an always-on VPS and owns messaging, agents, and models. A local machine (Mac Mini, desktop, Pi) is a paired **node** for what the VPS lacks: GPU or local models, real browser with a residential IP, macOS tools, local files. A node is a peripheral.

## Safety contract

- **Preferred route:** Gateway stays on loopback; Tailscale Serve (or another stable HTTPS reverse proxy with WebSocket upgrade) gives it a `wss://` URL on the tailnet. Never expose port 18789 to the public internet, never use Tailscale Funnel for it.
- **Pairing:** use a single-use Node-host pairing link from the Control UI (or `openclaw devices join-code`). Never paste Gateway tokens, setup codes, or keys into chat, logs, or shell history.
- **Approval:** inspect the exact request (name, device id, IP, requested commands) and approve that request id only. A capability expansion is a new approval and may stay pending.
- **Exec:** named absolute-path commands plus approval gates. No shell (`sh`, `bash`, `zsh`) in any allowlist.
- **SSH:** optional, separate, least-privilege. It is not a substitute for pairing.

Each step ends with a **Done when** check. Stop at a failed check.

## Prerequisites

Both machines run the same OpenClaw version (`openclaw --version`), both are on one tailnet (`tailscale status` shows both), and the VPS Gateway is running (`openclaw gateway status`).

## Step 1: Keep the Gateway private, publish it with Tailscale Serve

On the VPS:

```bash
openclaw config get gateway.bind          # keep "loopback"
openclaw config get gateway.auth.mode     # "token" or "password"; never "none"
openclaw config get gateway.trustedProxies
tailscale serve --bg --https=443 http://127.0.0.1:18789
tailscale serve status                    # shows https://<vps>.<tailnet>.ts.net (tailnet only)
```

`gateway.trustedProxies` must be `["127.0.0.1"]` (the Serve proxy). Never put the Tailscale range `100.0.0.0/8` in it: every node then counts as a proxy and is rejected with `403 Proxy client attribution is required`.

**Done when:** `openclaw gateway status` shows `bind=loopback`, and `tailscale serve status` lists the `https://…ts.net` route as tailnet only.

### Advanced branch: direct tailnet or LAN, no Serve

Only when all four hold: token or password auth is on; a firewall limits 18789 to the tailnet or known private addresses; there is no public port-forward, Funnel, or cloud-firewall opening; and the node uses the exact Gateway URL. Then `openclaw config set gateway.bind tailnet` (or `lan` if local agent sessions on the VPS still need 127.0.0.1) and restart. `bind=lan` and plaintext `ws://` are not general defaults. If you cannot prove the boundary, use Serve.

## Step 2: Mint a single-use pairing link

Control UI: **Devices → Node host → Create pairing link**. CLI equivalent on the VPS:

```bash
openclaw devices join-code --json > /tmp/join.json && chmod 600 /tmp/join.json
```

The link is single-use and expires. Move it to the node privately (scp, or read it on the node's own screen). Never relay it through a chat transcript.

**Done when:** the link exists on the node machine and nowhere in chat or shell history.

## Step 3: Connect the node

On the node machine, the join URL carries the Gateway address:

```bash
openclaw connect --target-file ~/join.txt --display-name "Mac Mini"            # foreground proof
openclaw connect --target-file ~/join.txt --display-name "Mac Mini" --service  # install as LaunchAgent / systemd
openclaw node status                                                            # service, pid, log path
```

`--target-file` reads the link from a private file and deletes it. The service form installs `~/Library/LaunchAgents/ai.openclaw.node.plist` on macOS or a systemd user unit on Linux, with env in `~/.openclaw/service-env/`. Node log: `~/Library/Logs/openclaw/node.log` (macOS).

Advanced branch (Step 1 direct route): `openclaw node install --host <gateway-tailscale-ip> --port 18789 --no-tls --display-name "Mac Mini"` and set `OPENCLAW_ALLOW_INSECURE_PRIVATE_WS=1` in the node service env. Do it only on a proven private route.

**Done when:** `openclaw node status` on the node shows the service running, and `openclaw nodes pending` on the VPS shows one new request from it.

## Step 4: Approve the exact request

On the VPS:

```bash
openclaw nodes pending                      # request id, node name, IP, requested caps
openclaw nodes describe --node "Mac Mini"   # device, model, PATH, current vs pending caps and commands
openclaw nodes approve <request-id>         # that id only; needs operator.admin
openclaw nodes status
```

Compare the request with the machine you just installed: display name, IP, hardware model, requested commands. Approve nothing you did not just create.

After an upgrade the node advertises new capabilities (for example `claude-sessions`, `mcp`, `fs.listDir`) and files a **reapproval** request. Existing caps keep working while it is pending. Approving it grants Gateway agents that new surface (including Claude Code session and terminal control), so read `describe` first and leave it pending if the workflow does not need it.

**Done when:** `openclaw nodes status` shows the node `paired · connected` with no unreviewed request accepted.

## Step 5: Route exec and allowlist named binaries

```bash
openclaw config set tools.exec.node "Mac Mini"
openclaw approvals get --node "Mac Mini"                   # defaults + current allowlist
openclaw approvals allowlist add --agent main --node "Mac Mini" "/usr/bin/uname"
openclaw approvals allowlist add --agent main --node "Mac Mini" "/opt/homebrew/bin/ollama"
openclaw exec-policy show                                  # requested vs host vs effective
```

Agents then run `exec host=node command="/usr/bin/uname -a"`. Keep node defaults at `security=allowlist, ask=on-miss` so anything off the list raises an approval instead of running. One pattern per binary, absolute path, `--agent main` rather than `*`. Never allowlist a shell, an interpreter with arbitrary code, or `rm`. Direct probe from the VPS:

```bash
openclaw nodes invoke --node "Mac Mini" --command system.which --params '{"bins":["ollama"]}' --json
```

**Done when:** `approvals get` shows only binaries with a named use, and an unlisted command produces a pending approval rather than output.

## Step 6: SSH fallback, separately secured (optional)

Use SSH for file transfer, a full login shell (nvm, Homebrew PATH), or when the node is offline. On the node create a dedicated non-root user; on the VPS:

```bash
ssh-keygen -t ed25519 -f ~/.ssh/id_node -N ""
ssh-copy-id -i ~/.ssh/id_node.pub <node-user>@<node-tailscale-ip>   # verify the host-key fingerprint on first connect
printf 'Host my-node\n  HostName <node-tailscale-ip>\n  User <node-user>\n  IdentityFile ~/.ssh/id_node\n  IdentitiesOnly yes\n' >> ~/.ssh/config
ssh my-node -- true
rsync -avn ./data/ my-node:~/data/          # preview first; drop -n only after reading the plan
```

Never set `StrictHostKeyChecking no`. Pass JSON or quotes as a script (`ssh my-node bash -s < job.sh`), not inline. Routine exec and transfer patterns live in the companion `remote-node-ssh` skill; adopt it only after Steps 1 to 5 pass.

**Done when:** `ssh my-node -- true` exits 0 with a verified host key and a non-root user.

## Diagnose by the first failed transition

| Symptom | Meaning | Action |
|---|---|---|
| No connection attempt in `node.log` | route | fix DNS, Serve, TLS, firewall; `tailscale status` on both |
| `403 Proxy client attribution is required` | trustedProxies too wide | set `gateway.trustedProxies` to `["127.0.0.1"]`, restart |
| `Cannot connect over plaintext ws://` | direct route without TLS | use Serve `wss://`, or the advanced env override on a proven private route |
| auth error, pending list empty | credential | recreate the join link (Step 2); old codes expire |
| `pairing required` | route and auth fine | `nodes pending` → approve the exact id |
| `reapproval pending` after upgrade | new caps requested | `nodes describe`, then approve or leave pending |
| paired but disconnected | node side | `openclaw node status` on the node, machine sleep, Wi-Fi; prefer Ethernet, disable sleep |
| exec denied or `command not found` | allowlist or PATH | `approvals get --node`; use the absolute path from `nodes describe` PATH |

A request id is stale after any retry that changed the node's identity or requested caps. Re-list before approving. Order: route → auth → pairing → disconnect.

## Verify

```bash
openclaw doctor
openclaw security audit
openclaw security audit --deep
openclaw nodes status
```

Confirm the node reconnects after the last change, the Gateway is unreachable from outside the tailnet, and no log line holds a token. Resolve critical findings; document intentional warnings. Version-specific upgrade notes: `references/upgrade-notes.md`.

## Tested with

OpenClaw 2026.9.3 (Gateway and node), Node 24.18, Tailscale 1.x, macOS 26 (Mac16,10) node, Ubuntu 24.04 VPS.

## License

MIT
