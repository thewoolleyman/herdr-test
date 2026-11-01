# Herdr experiments and notes

Consolidated notes from separate MacBook and Linux VPS Herdr experiments.
The original `~/tmp/ai/herdr-test` directories were empty, without Git repositories;
the project history lived in separate Codex conversations. This repository preserves
reviewed conversation exports and a shared working summary.

## Contents

- [Shared context](CONTEXT.md): concepts, workflows, findings, and open checks.
- [MacBook conversation](history/macbook.md): Android access, SSH keys, and clients.
- [VPS conversation](history/vps.md): spaces, agents, and multi-machine navigation.
- [Consolidation record](CONSOLIDATION.md): provenance and publication boundaries.

## Latest VPS status

On 2026-11-01 the VPS CLI and running server were upgraded from Herdr 0.9.3 to
0.9.5 using live handoff. All 28 pane-shell processes, 26 tabs, and six workspaces
survived. The client briefly disconnected; the user reconnected and reported that
it looked good. See [the upgrade record](CONTEXT.md#vps-upgrade-to-095-2026-11-01)
for preservation checks, integration updates, and latency measurements.

## Local Mac plus remote VPS

Run these in a **Mac shell**, with `vps` configured as an SSH alias:

```sh
herdr machine add vps
herdr
```

Select **Local** for the Mac and **vps** for remote work. Running commands in a VPS
pane executes them on the VPS even when the outer client runs on the Mac.

To attach only to the remote session:

```sh
herdr --remote vps
# Or a specific named session:
herdr --remote vps --session work
```

From an Android SSH client, connect to the desired host and run `herdr` there.
Detach with **Ctrl+B, then Q**; the server keeps its panes running while the host
remains operational. Detaching does not exit an enclosing SSH shell.

## Working copies

The GitHub repository is [thewoolleyman/herdr-info](https://github.com/thewoolleyman/herdr-info).
Clone it under `~/workspace/herdr-info` on each machine. Commit and
push changes from one clone, then pull them in the other. Git synchronizes these
notes; it does not move running Herdr panes, agents, or Codex session databases.

## References

- [How to work with Herdr](https://herdr.dev/docs/how-to-work/)
- [Connecting machines](https://herdr.dev/docs/connecting-machines/)
- [Persistence and remote access](https://herdr.dev/docs/persistence-remote/)
- [Concepts](https://herdr.dev/docs/concepts/)

Commands and app capabilities may change; check current help before modifying
running sessions. No application code or automated test suite is included.
