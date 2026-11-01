# Shared project context

## Model

A host can run independent Herdr server sessions. Each session owns spaces
(workspaces), tabs, and panes; a pane may run an agent, shell, tests, or a server.
Agent and pane IDs are scoped to their server. Saved machine profiles target one
session, not all sessions on a host. A client can display multiple machines.

## Repository name and checkout locations

On 2026-10-12 the repository was renamed from `herdr-test` to `herdr-info`.
The current GitHub remote is `https://github.com/thewoolleyman/herdr-info.git`.
The VPS checkout is `/data/projects/herdr-info`; the Mac checkout is
`~/workspace/herdr-info`. Older `herdr-test` paths below and in conversation
exports describe historical locations. After a checkout directory is renamed,
start new agent sessions in its new path.

## Findings from both conversations

- Use plain `herdr` on the Mac plus `herdr machine add vps` for a combined view.
- `herdr --remote vps` attaches directly to the remote session.
- Local is the client's built-in local connection. Saving `localhost` can create
  an unnecessary second connection to the same local session.
- Inspect `herdr machine list` before removing a duplicate using its actual ID.
  Removing a profile disconnects the client without stopping the remote server.
- Check `herdr machine status vps`; use `herdr machine reconnect vps` for SSH
  authentication recovery where supported by the installed release.
- UI selection does not retarget CLI commands in existing panes. Use `hostname`
  to establish execution location, and explicit machine targeting for automation.
- An always-on VPS is useful for ongoing work; a sleeping laptop cannot keep
  executing agents merely because its terminal client was detached.

## Keyboard reference

- Rename the current space (workspace) in the Herdr client UI with **Ctrl+B,
  release, then Shift+W** using the default bindings. Enter confirms the name.
  Verified against the [keyboard documentation](https://herdr.dev/docs/keyboard/)
  on 2026-10-06; custom bindings may differ.
- Rename the current tab with **Ctrl+B, release, then Shift+T**.
- Resize terminal splits with **Ctrl+B, release, then r**, followed by **h/l**
  for width or **j/k** for height; **Esc** exits. This is pane resizing, not
  sidebar resizing ([upstream README](https://github.com/motionharvest/herdr)).

### Sidebar toggle when nested in tmux

The user subsequently clarified on 2026-10-16 that the failed sidebar attempt
used **x** instead of **b**. This explains the reported failure; the tmux behavior
below is a separate controlled finding, not the established cause of that incident.

Verified on 2026-10-16 with Herdr 0.9.3 and tmux 3.5a in a disposable tmux
server and isolated Herdr config/runtime/session. With default bindings:

- Direct Herdr input: **Ctrl+B, release, then lowercase b** toggles the sidebar.
- Herdr inside tmux: **Ctrl+B, Ctrl+B, release Ctrl, then lowercase b**.
  tmux consumes the first Ctrl+B; its `send-prefix` binding forwards the second
  Ctrl+B to Herdr. One Ctrl+B followed by b did not toggle Herdr in this setup.
- Uppercase **B** did not toggle it. The collapsed sidebar remained a narrow
  strip; it did not disappear completely.

Captured terminal rendering verified collapse and restoration using the doubled
prefix, and direct pane input verified the single-prefix Herdr binding. Custom
prefixes or additional nesting layers need separate checks. The first test was
blocked by fresh-config onboarding; the successful test used `onboarding = false`.
Only disposable test servers were stopped. Existing user sessions were untouched.
See the [keyboard reference](https://herdr.dev/docs/keyboard/).

## Workspace sidebar branch selection

Verified on 2026-10-12 against the
[v0.9.3 workspace source](https://github.com/herdrdev/herdr/blob/v0.9.3/src/workspace.rs)
and Git refresh implementation: the workspace's Git identity uses the current
working directory of the **root pane in the first tab in current tab order**.
If that directory cannot be resolved, it falls back to the workspace's stored
identity directory. The sidebar branch is cached Git information for that
directory; it does not aggregate repositories across tabs or follow the selected
tab. Reordering or closing the first tab can therefore change which repository
supplies the branch after refresh. Changing the root pane's directory can also
change it. These directories belong to the host running the panes, even when the
client UI runs on another machine. A read-only inspection of the VPS `livespec`
workspace found its first tab's pane in the `livespec-overseer` repository, whose
branch was `master` at inspection time.

## Sidebar width configuration

On 2026-10-10, changing the VPS config's `ui.sidebar_width` from the default 26
to 36 and `ui.sidebar_max_width` from 36 to 46 did not visibly resize the user's
sidebar after the suggested reload. The user confirmed the visible Herdr client
runs on the Mac and connects to the VPS. The VPS overrides were reverted.

The user then opened a reverse SSH tunnel from a local Mac terminal, using its
existing `vps` alias:

```sh
ssh -NT -o ExitOnForwardFailure=yes -o ServerAliveInterval=30 -R 127.0.0.1:22222:localhost:22 vps
```

This exposes the Mac's SSH service only on the VPS loopback interface; connecting
back still requires a valid Mac account and SSH authentication. Existing SSH
authentication worked, so no temporary authorized key was installed.

The Mac was verified to run Herdr 0.9.3 with `herdr --remote vps`. Its config had
no sidebar width overrides, and its saved client preferences contained no manual
sidebar width. The Mac config was backed up outside this repository and changed:

```toml
[ui]
sidebar_width = 36
sidebar_max_width = 46
```

The default width increased from 26 to 36 columns; the maximum increased from 36
to 46. TOML parsing and a comparison confirmed that other settings were preserved.
The repository was cloned to `~/workspace/herdr-test` on the Mac and pulled with
`git pull --ff-only` before recording the change there.
The connected VPS accepted `herdr server reload-config` with status `applied` and
no diagnostics. In v0.9.3 this broadcasts a reload to attached clients, which
reread their own local config. The on-screen result has not been independently
observed; if necessary, use **Ctrl+B, release, then Shift+R** in the Mac client.

The [v0.9.3 client reload code](https://github.com/herdrdev/herdr/blob/v0.9.3/src/client/shell/config.rs)
preserves a manually selected sidebar width on reload. Layout clamps that width
to `sidebar_min_width` and `sidebar_max_width`. A double-click on the sidebar
divider resets a manual width to the configured default, per the
[mouse handler](https://github.com/herdrdev/herdr/blob/v0.9.3/src/client/shell/mouse.rs).
These are client-owned settings: a Mac client attached to the VPS reads the Mac
config, whereas `ssh` followed by `herdr` on the VPS reads the VPS config.

## Agent status legend

Verified on 2026-10-08 against the
[v0.9.3 client source](https://github.com/herdrdev/herdr/blob/v0.9.3/src/client/shell.rs):

| Status | Theme color | Default icon | Symbol mode | Meaning |
| --- | --- | --- | --- | --- |
| Blocked | Red | ● | × | Needs input, approval, or a decision |
| Working | Yellow | ● | ◐ | Actively working |
| Done | Teal | ● | ✓ | Finished; not yet viewed in this client |
| Idle | Green | ○ | ○ | Waiting for work; no unseen completion indicator |
| Unknown | Muted gray (`overlay0`) | · | · | State cannot be determined confidently |

Actual shades depend on the theme. The v0.9.3 mobile renderer uses blue for Done.
`ui.status_indicators = "symbols"` selects the alternate static glyphs.
See also [status concepts](https://herdr.dev/docs/concepts/).

With `agent_panel_sort = "priority"`, Blocked agents sort to the top of the agent
panel, but the panel keeps its scroll offset as rows reorder. A red × on a
workspace with no × visible in the agent panel usually means the panel is
scrolled down; scroll it to the top. Rows from a machine whose connection is not
Online sort last and render dimmed gray, per the v0.9.3
[aggregate sort](https://github.com/herdrdev/herdr/blob/v0.9.3/src/client/shell/aggregate_navigation.rs).
Observed 2026-10-14.

## Pane text selection

For iTerm2 intercepting right-clicks on Herdr tabs, check **Settings → Profiles →
active profile → Terminal** and enable **Enable mouse reporting** and
**Report mouse clicks & drags**. The
[iTerm2 Terminal profile documentation](https://iterm2.com/documentation-preferences-profiles-terminal.html)
verified on 2026-10-12 confirms that disabling click/drag reporting makes mouse
buttons perform iTerm2's default action instead of reporting them to an app.
Holding Option temporarily disables mouse reporting. Apply these settings in
iTerm2 on the Mac, regardless of which host owns the Herdr session. The user
reported that this advice did not resolve right-click tab renaming on 2026-10-12
and chose the keyboard shortcut instead. The actual settings and delivered mouse
events were not inspected; do not treat the documented options as a verified fix.
Rename the current tab with **Ctrl+B, release, then Shift+T**, type its name,
and press **Enter**.

Checked against local default config and public docs on 2026-10-08:
`[ui] mouse_capture = true` enables Herdr mouse handling; `copy_on_select = true`
copies its selection automatically. Both default to true. Reload edited config
with **Ctrl+B, release, then Shift+R**. Herdr selection is scoped to pane text;
outer-terminal selection can include sidebar and other UI text. Mouse gestures
must reach Herdr for its selection to apply; enabling capture cannot constrain
selection performed locally by the Android SSH app. Tablet gesture behavior
remains unverified. Keyboard fallback: **Ctrl+B, then [**, move with **h/j/k/l**,
start selection with **Space**, extend it, and copy with **y**.
See [config reference](https://herdr.dev/docs/config-reference/) and
[keyboard copy mode](https://herdr.dev/docs/keyboard/#copy-mode).

## Moving terminal work between workspaces

### Reordering tabs within a workspace

Correction verified 2026-10-16: lack of a `herdr tab move` CLI command does not
mean keyboard reordering is unavailable. The installed Herdr 0.9.3
`--default-config` exposes `move_tab_previous` and `move_tab_next`, both unbound
by default, with examples `alt+shift+left` and `alt+shift+right`. The
[Omarchy quattro config](https://github.com/omacom/omarchy/blob/quattro/config/herdr/config.toml)
uses that pair, as does the
[Herdr Web UI v0.4.6 release](https://github.com/devswha/herdr-web-ui/releases/tag/v0.4.6).
These are concrete examples, not evidence of a universal community convention.

For Termux toolbar use, the user adopted prefix followed by an unmodified arrow:

```toml
[keys]
move_tab_previous = "prefix+left"
move_tab_next = "prefix+right"
```

The pre-change VPS config had no key overrides; the default prefix Left/Right
slots are free (pane focus uses prefix h/l). A temporary config containing these
bindings passed the installed `herdr config check` with `config: ok`, exit 0.
After user approval on 2026-10-16, these bindings were applied to the VPS client
config and reloaded successfully with no diagnostics. The idempotent installer
and fragment live in the private VPS configuration repo (`9c9930a`); it manages
only these two keys, backs up changed config locally, and validates preservation
of all other parsed settings. A second installer run made no changes.

The Android configuration repo (`dda9a0e`) adds a Termux toolbar `PREFIX` macro
that sends `CTRL b`. The user clarified that the original request for Shift+B
meant Control+B. Tap PREFIX, then LEFT/RIGHT; swipe up on PREFIX to access `/`.
The toolbar was installed and reloaded; a second run was unchanged. Actual
touchscreen rendering/tapping remains unverified.

A disposable Herdr client inside an isolated tmux server verified the keys:
starting order `1, Second, Third`, prefix+Left moved the focused Third tab before
Second, and prefix+Right restored the original order. The test forwarded the
tmux prefix with two Ctrl+B strokes before each arrow. Only test servers were
stopped; existing user tabs and sessions were not used for the test.

Bare arrows should remain available to shell/editor/agent cursor movement.
Herdr also uses them within navigation and resize modes; those modal bindings
are separate from prefix actions. See
[configuration semantics](https://herdr.dev/docs/configuration/#keybindings).

### Moving panes between workspaces

Local Herdr 0.9.3 help checked on 2026-10-06 exposes no `herdr tab move`
subcommand. The supported CLI workaround for a single-pane tab is
`herdr pane move <pane_id> --new-tab --workspace <workspace_id> --label <name>`.
Discover IDs with `herdr workspace list` and `herdr pane list` on the host/session
that owns the work. This moves a running pane into a new destination tab; it does
not transfer a whole split layout. See the
[CLI reference](https://herdr.dev/docs/cli-reference/).

Verified on 2026-10-12 while regrouping single-pane tabs by repository: adding
`--no-focus` to the move command preserves user focus. Moves append destination
tabs; processing source tabs in order preserves their relative order after the
destination's existing tabs. Workspace-qualified pane and tab IDs change, while
terminal IDs remain stable. Moving the last pane out automatically removes the
empty source workspace. Verification of 14 moves confirmed stable terminal IDs,
tab labels and order, and unchanged existing destination tabs and unrelated
workspaces.

## Android

The Herdr Mobile app discussed in the Mac conversation is the WNT app with package
ID `dev.herdr.mobile`. Its published privacy policy stated password-only SSH
authentication during this investigation. The initial suggestion of this app did
not mention that limitation; the later answer corrected it. This is an app
limitation, not a limitation of Herdr's SSH workflow.

Termius or Termux can provide key-authenticated SSH access followed by `herdr`.
The tradeoff is terminal navigation and rendering on a small touchscreen instead
of app-specific agent views and attachment controls. No Android client was tested
end to end in these sessions.

The user clarified that the tablet client is Termius, not Termux. Termius's
[official mobile tips](https://www.linkedin.com/posts/termius_vibecoding-ai-termius-activity-7447815156828450816-gkrd)
describe long-press plus vertical drag as arrow-key input (checked 2026-10-06).
Gesture interception is therefore a plausible explanation for failed Herdr border
dragging, but the tablet's app version and actual input events are unverified.
Do not apply Termux-specific implementation findings to Termius.

Separately, Termux touchscreen behavior was checked against upstream source on 2026-10-06:
long-press starts text selection, and finger movement does not send held-button
mouse drag events. This explains why touch taps can work while Herdr border
dragging fails. Physical mouse input has a separate drag-reporting path; using
an external mouse remains untested on the user's tablet.
See [Termux TerminalView source](https://github.com/termux/termux-app/blob/master/terminal-view/src/main/java/com/termux/view/TerminalView.java).
For the workspace sidebar, Herdr exposes `ui.sidebar_width`,
`ui.sidebar_min_width`, and `ui.sidebar_max_width` in its
[config reference](https://herdr.dev/docs/config-reference/).
These are Herdr UI settings: when SSH is followed by running `herdr` on the VPS,
the Herdr client runs on the VPS even though the Android SSH app renders it.

Sources:
- [Herdr Mobile privacy policy](https://herdrmobile.zephwu.com/privacy-policy)
- [Herdr Mobile store listing](https://play.google.com/store/apps/details?id=dev.herdr.mobile)
- [Termius Android](https://www.termius.com/free-ssh-client-for-android)
- [Termux](https://github.com/termux/termux-app)

## Verified at consolidation

Both machines report Herdr 0.9.3. The two original project directories contain no
files or Git history. One relevant Codex conversation was found on each host.
The VPS Codex memory directory contained no files, and no project-specific Claude
memory directory was found there. Local global memory included an older Herdr
installation/workspace observation; current filesystem and version checks take
precedence over that observation.

## Test-created workspace pollution (2026-10-12)

Read-only inspection of the VPS default session found 70 workspaces labelled
`topic` and four labelled `supervisor`, each containing one pane whose cwd was a
deleted pytest temporary repository. These are separate workspaces; `fabro`
itself held five panes. A sampled leftover pane had only its retained zsh shell
in the foreground. No leftover workspace or live session was closed.

The matching tests belong to `livespec-overseer`, plan `herdr-overseer`, anchored
by epic `overseer-uzvcbn` (Support herdr and tmux overseer panes with verified
automatic selection). The initial-start implementation is owned by child
`overseer-zbpqnj` (Exact owned-pane initial start and uncertain-launch preservation).
Its public start path now detects the invoking process's terminal through real
ancestry and selects a native Herdr driver. Several legacy CLI tests substitute
only `TmuxIO`; their setup changes cwd and installs a fake Claude executable but
does not substitute the native ownership/router boundary. This provides a mechanism
for tests run from a Herdr-owned process to create real `topic` workspaces despite
the tmux fake, consistent with the observed leftovers. A controlled reproduction
was not run during this investigation. The plan's live-verification research already requires isolated named
sessions and cleanup; ordinary unit-suite isolation must enforce the same
production-session protection. Clearing environment selectors alone is
insufficient because backend selection uses process ancestry.

## Interactive latency investigation (2026-11-01)

Read-only checks on the VPS at approximately 21:33–21:38 CET found host-side
contention while the user reported slow typing and pane switching. These are
point-in-time observations, not a controlled comparison with the earlier snappy
state:

- The VPS exposed 18 vCPUs; load averages were approximately 52–64. CPU pressure
  (`/proc/pressure/cpu`, `some avg10`) was approximately 44–55%. Herdr's server
  main thread accumulated 235 ms of runnable scheduling wait during one roughly
  two-second probe. Idle CPU in an aggregate sample does not establish responsive
  scheduling for the thread handling an interactive request.
- Twenty local `herdr workspace list` calls ranged from about 8 to 424 ms;
  a separate earlier call took 1.75 seconds. These calls ran directly on the VPS,
  so the variation exists before adding client/network latency. A separate
  seven-call sample had a 12 ms median, illustrating the intermittent stalls.
- An hourly `/etc/cron.hourly/fstrim` job ran `fstrim /` and was still active
  after approximately 18 minutes. Its kernel stack showed it waiting inside
  `ext4_trim_fs` / `blkdev_issue_discard`. Disk samples showed hundreds of discard
  requests per second, with approximately 75–85% device utilization in one
  window; a later window showed 16–22% host I/O wait. The root filesystem also
  had the `discard` mount option, and a separate weekly `fstrim.timer` existed.
  The long trim is a plausible latency contributor, not a proven sole cause.
  Follow-up inspection found the hourly script owned by `root:root`, last
  modified on 2026-01-06, and unclaimed by `dpkg-query -S`. Its observed parent
  chain was system cron → run-parts → script → fstrim. Who installed the script
  remains unknown. No `fstrim` process remained at the follow-up check.
- Memory had approximately 58 GiB available, memory pressure was zero, and the
  root filesystem was 57% full. Sampled CPU steal was zero, and inspected Herdr
  cgroups and ancestors had no CPU quota throttling. These checks did not support
  RAM exhaustion, full root storage, or a cgroup CPU quota as the immediate cause.
- Herdr 0.9.3 served seven workspaces, 55 tabs, 58 panes, and 35 detected agents.
  A separate three-second `/proc/stat` sample measured approximately 76 process
  creations per second across the host, consistent with substantial process churn.
  Both inspected VPS client processes were descendants of Tailscale SSH. Their
  existence does not establish which client the user was viewing; client/device
  and network contributions were not measured end to end.
- An existing diagnostic service used BPF only for process-exit events. Its
  presence alone is not evidence that tracing caused the slowdown. Recent client
  warnings mostly concerned failed notification audio playback; these were not
  established as the cause of typing/pane-switch latency.

Initial suggested next checks, not applied: reduce concurrent background tests and
polling/process churn, review whether the custom hourly trim is needed alongside
the weekly timer and online discard, and compare local API and interactive
latency after trim completes. A controlled before/after measurement is needed
to attribute improvement to a specific change. No live sessions were stopped,
and no host configuration was changed during that initial pass.

### Follow-up: cross-vCPU synchronization stalls

At 22:12 CET, with **no fstrim process running**, the lag was still reproducible:
40 local CLI calls had a 170 ms median and a 3.96 second maximum. CPU pressure
was approximately 75%, while I/O wait averaged approximately 1%. A new hourly
trim started later, around 22:17; its recurrence cannot explain the already
measured stalls while no trim was running.

Further profiling established a specific mechanism:

- A system-wide CPU-clock profile contained 7,805 samples. Approximately 36%
  landed in `smp_call_function_many_cond`, predominantly through TLB flushes
  during memory-map changes. Of 2,804 sampled stacks containing that function,
  1,264 came through `madvise`; the other callers included process exit, fork,
  exec, and `munmap`. Claude/Bun's `mi-scavenger`, `HeapHelper`, and `JITWorker`
  threads were prominent callers. This attributes sampled work, not exclusive
  blame to one application or release.
- A separate Herdr server profile contained 810 samples. Approximately 43%
  landed in the same cross-vCPU synchronization function; all 352 stacks
  containing it came through `munmap`. A syscall trace independently observed
  Herdr unmaps taking hundreds of milliseconds, including 390 ms on the server's
  main thread, alongside long futex waits. Futex duration alone is not proof of
  a bug, since a worker can legitimately sleep waiting for work.
- Direct Unix-socket requests excluded CLI startup and SSH. In parallel samples
  of 25 valid requests each, `ping` had a 0.8 ms median, while `workspace.list`
  had a 477 ms median and a 3.69 second maximum. The
  [v0.9.3 API server source](https://github.com/herdrdev/herdr/blob/v0.9.3/src/api/server.rs)
  handles `ping` directly but dispatches workspace requests to the app loop.
  Another workspace request reached 5.52 seconds. These observations locate
  substantial delay in the server/app path without establishing every lock's owner.
- Kernel CSD tracepoints, filtered specifically to `flush_tlb_func`, showed
  queued remote TLB callbacks taking up to approximately 373 ms to reach entry.
  Per-vCPU mean queue delays in that window ranged from approximately 4 to 36 ms.
  A separate unfiltered callback trace had longer tails, but those must not all
  be labelled TLB flushes.
- A reversible affinity experiment confined the existing Herdr server threads
  to one vCPU briefly. Workspace-request median latency was 56 ms before,
  4.9 ms during, and 65.5 ms after restoration. P95 was approximately
  585 / 238 / 547 ms respectively. Host CPU pressure remained high throughout.
  This supports a material contribution from cross-vCPU coordination, but did
  not eliminate latency tails. All server threads were verified restored to
  their original 18-vCPU affinity. A permanent single-vCPU pin was **not**
  applied: new pane processes could inherit that affinity and lose parallelism.

The likely underlying amplifier is VM CPU scheduling or throttling, with two
important qualifications about the available evidence:

- Across three five-second windows, `/proc/stat` accounted for only
  56.58 / 47.85 / 55.85 CPU-seconds, including idle, against approximately
  90 wall-clock CPU-seconds expected from 18 vCPUs. This is consistent with
  unreported lost execution time, but guest accounting alone cannot distinguish
  host contention, provider CPU throttling, and a guest accounting defect.
- CPUID leaf `0x40000001` returned EAX `0x010000db`: KVM steal-time reporting,
  paravirtual TLB flush, and paravirtual IPI feature bits were all absent.
  The running kernel also had `CONFIG_PARAVIRT_TIME_ACCOUNTING` disabled.
  **The earlier 0% steal measurement does not rule out host contention.** See
  the kernel's [KVM CPUID documentation](https://docs.kernel.org/virt/kvm/x86/cpuid.html).

Disposable memory-discard probes outside Herdr normally completed in
microseconds, but longer samples also encountered 112–413 ms outliers, including
in a single-vCPU variant. They therefore corroborate general scheduling tails;
they do not establish that every stall requires a cross-vCPU TLB flush. No
before-regression baseline or hypervisor-side counters were available.

The immediate bottleneck is established: memory-map cleanup spends substantial
time on cross-vCPU synchronization, and Herdr's interactive app path stalls.
Provider-side confirmation is still needed to distinguish host CPU scheduling
contention from throttling as the underlying cause. A concrete escalation should
request host vCPU scheduling/CPU-limit counters, investigate the missing CPU-time
accounting, and check the unavailable KVM paravirtual features. Reducing allocator
purging and process churn is a potential mitigation, not a verified complete fix;
the [mimalloc documentation](https://github.com/microsoft/mimalloc#environment)
describes purge delay and decommit controls, but they were not changed in live
agents. No Herdr sessions were stopped or upgraded during those probes. Raw profiles remained outside
this public repository; only reviewed measurements are recorded here.

### Upstream fixes identified before choosing a remedy

The installed 0.9.3 predates relevant fixes released in 0.9.4 and retained in
0.9.5, per the [0.9.5 changelog](https://github.com/herdrdev/herdr/blob/v0.9.5/CHANGELOG.md).
Reviewed changes include avoiding per-cell allocations during terminal text
reads ([PR 4845](https://github.com/herdrdev/herdr/pull/4845)) and avoiding
unneeded terminal/scroll-metric reads in client snapshots
([PR 4847](https://github.com/herdrdev/herdr/pull/4847)). The latter separates
metadata snapshots from terminal reads, relevant to the observed app-loop waits.
These are concrete application-side leads; an upgrade followed by the same
latency probes should precede treating provider contention as the sole cause.
At that stage no upgrade comparison had been performed. The subsequent upgrade
and its measurements are recorded below; a complete remedy remains unproven.

The [session-state documentation](https://github.com/herdrdev/herdr/blob/v0.9.5/docs/next/website/src/content/docs/session-state.mdx)
labels live handoff experimental and best effort. Plain `herdr update` can leave
a compatible old server running, so merely updating the binary does not establish
that server performance fixes are active. The old sending server's handoff pane
limit and compatibility must be checked before any live replacement. These
checks informed the subsequently authorized upgrade recorded below.

## VPS upgrade to 0.9.5 (2026-11-01)

The user authorized a safe upgrade while cleaning up running work. The VPS CLI
and default session's running server were upgraded from **0.9.3 to 0.9.5** using
`herdr update --handoff </dev/null` on the stable channel. This replaced the
running server through live handoff; no stop/restart fallback was used. The
non-interactive updater was checked to refuse a stop-server fallback after a
failed handoff. The old server advertised live-handoff support, protocol 22,
and endpoint generation 1. At the final pre-upgrade snapshot it had 28 panes,
below its 64-pane handoff limit.

Before the upgrade, the binary, Herdr configuration, saved session metadata,
live snapshot, and direct pane-shell PID/start-time identities were backed up
privately under `~/.local/state/herdr-upgrade-backups/<timestamp>/`. Agent settings
were also backed up before integration installation. Runtime snapshots, hooks,
settings backups, and raw profiles were not added to this public repository.

Post-upgrade verification found:

- The installed CLI and running server both report 0.9.5. Status reports them
  compatible, with no pending restart or stale server binary.
- **All 28 original pane-shell processes survived**, verified by matching PIDs
  and process start times. All six workspace IDs, 26 tab IDs, and 28 pane IDs
  remained present, with the same pane locations and working directories.
- Internal terminal IDs were regenerated during handoff. Automation caching
  terminal IDs must refresh them from the current snapshot; unchanged pane IDs
  and surviving shell processes provide the continuity evidence here.
- The Herdr config file was byte-for-byte unchanged. All server threads had
  their original 18-vCPU affinity before the upgrade; the earlier diagnostic
  single-vCPU restriction was not left in place.
- The user observed the expected client disconnection, reconnected, and reported
  that it looked good. Tabs and workspaces did **not** need to be closed. In-flight
  requests and subscriptions can be interrupted even when pane processes survive.

The updater identified stale installed agent integrations. After backing up
settings, `herdr integration install claude` and `herdr integration install codex`
updated them to Claude integration v11 and Codex integration v10. Status reports
both current, with no outdated installed integrations. Claude's unrelated hooks
and non-hook settings were verified preserved; the parsed Codex config was
unchanged. These changes were made on the VPS, not on the Mac.

Thirty direct Unix-socket `workspace.list` requests were measured on either side
of the upgrade, excluding CLI startup and SSH/network latency:

| Measurement | Before: 0.9.3 | After: 0.9.5 |
| --- | ---: | ---: |
| Median | 317.53 ms | 32.44 ms |
| P95 (sorted sample index) | 2,954.49 ms | 531.31 ms |
| Maximum | 4,835.08 ms | 748.80 ms |

The observed median improved by approximately 9.8 times. Cleanup and background
work continued during these samples, so this is not an isolated benchmark of
the version change and does not prove that every source of VPS lag is resolved.
The upgrade supplies the upstream allocation and terminal-read fixes identified
above, and the user confirmed that the resulting client experience looked good.

The Mac version was not rechecked or upgraded in this operation. For future
upgrades, verify current handoff support and limits again, retain private backups,
and compare live pane processes before and after. The successful result here
does not make experimental live handoff a guarantee for every future session.

## Open checks

- Verify Android key authentication and interactive Herdr rendering on the device.
- Verify current saved-machine connectivity before treating historical profile
  listings as current state. The VPS conversation suggested removing a duplicate
  localhost profile but did not record confirmation that it was removed.
- Recheck mobile app capabilities before choosing an app based on these notes.
