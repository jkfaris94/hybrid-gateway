# Upgrade notes (2026.9.x)

Observed while moving a VPS Gateway + Mac Mini node from 2026.7 to 2026.9.3. Not needed for a fresh install; read when an upgrade breaks the hybrid setup.

- **Reapproval after upgrade.** The node advertises new capabilities and files a reapproval request. `openclaw nodes describe --node <name>` lists current vs pending caps and commands. Existing caps keep working while pending. `openclaw nodes approve <request-id>` needs `operator.admin`.
- **trustedProxies.** If `gateway.trustedProxies` includes the Tailscale range, every node is treated as a proxy and rejected with `403 Proxy client attribution is required`. Keep it at `["127.0.0.1"]`.
- **Gateway service reinstall.** `openclaw gateway install --force` refuses group-writable `~/.config`, `~/.config/systemd`, `~/.config/systemd/user`, and unit files. `chmod 755` the directories and `644` the unit first.
- **Config validation.** 2026.9 refuses to start on an invalid config. Run `openclaw doctor --fix` with the Gateway stopped; it migrates old keys (`env.*` to `env.vars`, exec `security: ask` to `mode`).
- **Backups.** `openclaw backup create --verify` failed on a large state dir. A cold `rsync` snapshot after `openclaw gateway stop` worked.
- **Codex plugin.** Requires Codex app-server 0.149.0 or newer (`npm i -g @openai/codex`); older versions make main turns fall back silently.
- **Node service on macOS.** `openclaw node status` shows the LaunchAgent and pid. The plist wraps node through `~/.openclaw/service-env/ai.openclaw.node-env-wrapper.sh` with a pinned Node path; stdout goes to `~/Library/Logs/openclaw/node.log`.
- **Invoke form.** `openclaw nodes invoke --node <name> --command system.which --params '{"bins":["node"]}' --json`. Approved system commands on a typical node: `system.run`, `system.run.prepare`, `system.which`, `system.execApprovals.get/set`, `browser.proxy`, `ollama.*`.
- **Plugins.** Pinned plugin versions do not follow the core upgrade; update with `openclaw plugins install @openclaw/<plugin>@latest`.
- **ssh quoting.** JSON params inside an inline `ssh host 'cmd'` break in zsh. Send the script instead: `ssh host bash -s < job.sh`.
