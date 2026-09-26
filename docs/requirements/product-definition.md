# Product Definition — Halyard

**An unofficial background sync client for Proton Drive on Ubuntu ARM64.**

| | |
|---|---|
| **Document** | Product Definition |
| **Status** | Draft v1.0 |
| **Date** | 2026-09-25 |
| **Repository** | `proton-desktop-linux-arm64` |
| **Related docs** | [`technical-blueprint.md`](technical-blueprint.md), `/docs/requirements/domain-model.md` *(not yet created)* |

> **Name is provisional.** "Halyard" is a placeholder chosen to avoid Proton branding. Any final name must not imply Proton endorsement. Throughout this document, "Proton Drive" is used descriptively (nominative use) to identify the service the product works with.

---

## 0. Context & Source Design

This product is modelled on the architecture of Proton's official Windows client, [`ProtonDriveApps/windows-drive`](https://github.com/ProtonDriveApps/windows-drive), which is GPLv3 and has a clean OS-abstraction seam we can exploit.

**How the Windows app is structured:**

| Component | Role |
|---|---|
| `src/Proton.Drive.App` | Platform-neutral core — Account, Drive, Sync, Photos, Onboarding, Settings, Notifications, Update, Localization, Reporting, InterProcessCommunication |
| `src/Proton.Drive.App.Windows` | WPF desktop UI shell |
| `src/Proton.Drive.Native.Windows` | Native Windows interop (auth, Win32) |
| `sync/cs/src/Proton.Drive.Sdk.Sync.{Engine,Adapter,Agent,DataAccess,Shared}` | The sync engine — **platform-neutral** |
| `sync/cs/src/Proton.Drive.Sdk.Sync.Client` | Talks to the Proton Drive API |
| `sync/cs/src/Proton.Drive.Sdk.Sync.Windows` | **The only OS-specific sync project** — FileSystem, Interop, Security, Shell |

**How this product maps onto it:**

| Windows component | Halyard equivalent | Reuse |
|---|---|---|
| `Sync.Engine` / `Adapter` / `DataAccess` | Same responsibilities: tree diffing, state machine, conflict detection, local state DB | **High** — study and adapt the design; reuse code where GPLv3 permits |
| `Sync.Client` (direct API) | **Replaced** by a transport adapter that drives the official `proton-drive` CLI as a subprocess | None |
| `Sync.Windows` (Cloud Files API, Explorer shell) | New `Sync.Linux`: inotify watcher, POSIX permissions/xattr, Nautilus extension | None — written fresh |
| `App.Windows` (WPF) | Avalonia UI, talking to a headless daemon over IPC | None — but the IPC pattern is mirrored |
| `Proton.Drive.Native.Windows` (auth) | **Eliminated** — the CLI owns login and the OS keyring | N/A |

**Why the CLI is the transport.** Proton's official CLI ([`ProtonDriveApps/sdk`](https://github.com/ProtonDriveApps/sdk/blob/main/cli/README.md)) ships a **`linux-arm64` binary**, emits **`--json`** for every command, and stores credentials in the **OS secret store** (`ch.proton.drive/drive-sdk-cli`, overridable via `PROTON_DRIVE_CREDENTIALS_STORE`). Critically, Proton states the CLI *"is not a full replacement — only the applications include a full synchronization engine that runs in the background."*

**That missing sync engine is precisely this product.** The CLI gives us authenticated, end-to-end-encrypted file transfer that Proton maintains. We add the thing it deliberately lacks: continuous, stateful, two-way folder synchronisation.

This also removes the two hardest and riskiest problems from our scope: we never implement SRP authentication, we never touch OpenPGP, and we never see the user's password or keys.

**The alternative was tested and rejected.** Proton also publishes a native **C# Drive SDK** (`client/cs`), which for a .NET application would have been better on every axis — no subprocess, and a real change-event feed. It does not build outside Proton: it depends on `Proton.Cryptography`, which is on a private feed with no public source. A build attempt and the full evidence are in [`technical-blueprint.md` §1.1](technical-blueprint.md). The CLI is therefore not a compromise chosen for convenience — it is the only complete, supported path available to an unofficial client.

---

## 1. Product Vision Statement

**Halyard gives Linux users on ARM64 the background folder sync that Proton Drive has never had — a small, honest daemon that keeps a local directory and a Proton Drive folder in step, loses nothing when they disagree, and gets out of the way.**

---

## 2. Core User Roles & Permissions Matrix

Halyard is a single-user desktop application, not a multi-tenant system. "Roles" here describe the distinct ways people interact with it, and what each can reach.

| Role | Who they are | Can do | Cannot do |
|---|---|---|---|
| **Desktop User** | The person signed in to the Ubuntu session. The GUI's default audience. | Sign in/out; add, pause, resume and remove sync pairs; view sync status and errors; trigger "Sync now"; resolve conflicts; change settings; view logs | Alter another Linux user's sync pairs; access Proton account settings (done on the web); bypass conflict safety rules |
| **Power User / Automation Operator** | Same human, working from a terminal or script. **The primary persona.** | Everything the Desktop User can, plus: edit the TOML config directly; drive the daemon via `halyardctl` and its IPC socket; run fully headless with no GUI; set per-pair conflict policy, ignore patterns and poll intervals; read structured JSON status for scripting | Change which OS user the daemon runs as without systemd/root access |
| **System Administrator** | Installs Halyard on a shared or managed machine. | Install/remove the package; enable the systemd **user** unit; set machine-wide defaults; control whether the bundled or system `proton-drive` binary is used | Read any user's Proton session or file contents — credentials live in the per-user keyring and data is E2E encrypted |
| **Maintainer / Packager** | The project side. | Build and publish `.deb`/AppImage for `arm64`; sign releases; publish update metadata | Ship Proton trademarks or imply endorsement |
| **Proton Account (upstream authority)** | Not a person — the remote system of record. | Owns identity, 2FA policy, quota, and the authoritative file tree | — |

**Security property worth stating plainly:** Halyard never handles the user's Proton password, 2FA code, or encryption keys. `proton-drive auth login` opens the system browser; Proton authenticates; the resulting session is written to the GNOME Keyring by the CLI. Halyard only ever knows *whether* a session exists.

---

## 3. Core Features (MVP)

These define v1. Each is a must-have; drop any one and the product doesn't do its job.

### PD-001 — Sync Daemon
A long-running headless process (`halyardd`) started by a **systemd user unit**. It owns all sync state and does all work. The GUI is a client of it, not the other way round. This is what makes the product usable on a headless box and trivially scriptable — and it mirrors the `InterProcessCommunication` split that already exists in the Windows app.

### PD-002 — Account & Session Management
Detect whether a Proton session exists. Offer "Sign in", which invokes `proton-drive auth login` and surfaces the browser flow. Detect expiry and prompt for re-auth without losing sync state. Sign out via `proton-drive auth logout`. **Single account in MVP.**

### PD-003 — Two-Way Folder Sync Engine
The core. For each configured sync pair (one local directory ↔ one Proton Drive path):
- Maintain a persistent **state tree** of what was last known to be in agreement.
- Compare local tree, remote tree and state tree on each cycle to classify every difference as *local change*, *remote change*, *both changed* (conflict), or *no change*.
- Apply creates, updates, moves/renames and deletes in both directions.
- **Full local sync only** — every file in a synced folder exists on local disk. No placeholders.

### PD-004 — Instant Local Push
An `inotify` watcher on each local sync root, with debouncing and atomic-save detection (editors that write-temp-then-rename must not produce spurious delete+create). Local changes upload within seconds.

### PD-005 — Remote Pull On Demand
The CLI emits no change notifications, so remote changes are discovered by listing. Pull is triggered by:
- daemon start,
- an explicit **"Sync now"** (GUI, tray, or `halyardctl sync`),
- reconciliation after a local push,
- an optional low-frequency safety poll (default **15 minutes**, user-configurable, can be disabled).

**Accepted limitation, stated openly in the UI:** changes made on another device may take minutes to appear until you press Sync now. This is a deliberate MVP trade-off in favour of low API load and laptop battery life.

### PD-006 — Never-Lose-Data Conflict Handling
When a file changed on both sides since the last agreed state, **keep both**. The remote version takes the canonical filename; the local version is renamed in place to `name (conflict 2026-09-25 14-32).ext` and then uploaded as a new file. The conflict is recorded, counted in the tray badge, and listed in the UI until dismissed. **No sync operation ever destroys the only copy of a byte of user data.**

### PD-007 — Sync Pair Configuration & Selective Sync
Add, edit, pause and remove sync pairs. Choose any Proton Drive subfolder as a remote root — you are never forced to sync the whole drive. Per-pair ignore patterns (`.halyardignore`, gitignore syntax) and a global default set (temp files, editor lock files, `node_modules`).

### PD-008 — Desktop Status UI
An **Avalonia** application providing:
- a **system tray icon** with at-a-glance state (idle / syncing / paused / error / conflicts / signed out) and a menu for Sync now, Pause, Open folder, Quit;
- a **main window** listing sync pairs, current transfer activity, recent history, conflicts awaiting attention, and errors in plain language;
- settings for pairs, poll interval, conflict policy and startup behaviour.

The GUI is optional at runtime — killing it must never interrupt sync.

### PD-009 — Nautilus File Manager Integration
A `libnautilus-python` extension that shows **sync-state emblems** on files and folders inside sync roots (synced / syncing / conflicted / ignored / error), plus a context menu with "Sync now" and "Show sync status". Emblems only in MVP — this is the thinnest slice that delivers the "it's integrated" feeling, and it degrades to nothing if Nautilus is absent.

### PD-010 — Control Surface & Observability
- A human-editable **TOML config** at `~/.config/halyard/config.toml` as the source of truth; the GUI edits this file.
- **`halyardctl`** for status, sync, pause, resume, pair add/remove, and conflict listing — all with `--json`.
- Structured rotating logs at `~/.local/state/halyard/`, a `halyardctl diagnostics` bundle for bug reports, and clear mapping from CLI subprocess failures to user-legible errors.

### PD-011 — Packaging for Ubuntu ARM64
A signed **`.deb` for `arm64`** targeting Ubuntu 24.04 LTS and later, installing the daemon, GUI, systemd user unit and Nautilus extension. Self-contained .NET publish so no runtime prerequisite. The `proton-drive` CLI is **detected first, prompted for if missing**, with bundling permitted as a fallback. An in-app update check points at GitHub Releases; MVP does **not** auto-install updates.

---

## 4. Additional Features (post-MVP)

Everything identified during definition that is explicitly *not* in v1.

**Deferred by explicit decision:**
- **Photos backup** — auto-upload of a local photo library to Proton Drive Photos, with de-duplication and album mapping.
- **Sharing & collaboration** — create/revoke public links, manage invitations and shared-with-me folders (the CLI already exposes `sharing status` / `sharing invite`).
- **Proton Docs integration** — open and edit Proton Docs from the file manager.
- **On-demand files (FUSE)** — placeholder files that hydrate on open, matching the Windows Cloud Files experience. The largest single piece of future work.

**Identified as valuable, not yet scheduled:**
- **Multi-account support** — more than one Proton account, each with its own pairs.
- **Sidecar transport** — replace the CLI subprocess with a long-lived Bun process hosting `@protontech/drive-sdk` (MIT, on npm, verified to install and bundle), behind the same transport interface. Removes per-operation process spawn. Its one obstacle is that the SDK's login module is unpublished and explicitly incubating.
- **Real-time remote change detection** — the same sidecar unlocks `subscribeToDriveEvents` / `subscribeToTreeEvents` (`node_created`, `node_updated`, `node_deleted`, `tree_refresh`), removing the on-demand limitation in PD-005 entirely. These events are **not** reachable through the CLI, which is what forces polling in MVP.
- **Bandwidth throttling and scheduling** — caps, quiet hours, metered-connection and battery awareness.
- **Trash and version restore UI** — browse deleted files and prior versions.
- **KDE Dolphin, Thunar and Nemo integrations.**
- **Broader packaging** — x86-64 builds, Flatpak, Snap, AUR, and other distributions.
- **Richer conflict resolution** — side-by-side diff and merge for text files; per-pair remote-wins/local-wins/prompt policies.
- **Symlink, hardlink and sparse-file policy** — currently out of scope and skipped with a warning.
- **Encrypted local state** — encrypt the state database at rest.
- **Migration assistant** — import existing `rclone` Proton Drive remotes and mounts.
- **Automatic updates** — background download and install with rollback.

---

## 5. Non-Functional Requirements

### Security & Privacy
- **NFR-S1** — Halyard never reads, stores, logs or transmits the user's Proton password, 2FA code, or private keys. All authentication is delegated to `proton-drive auth login`.
- **NFR-S2** — Sessions live in the OS secret store via the CLI's own mechanism. Halyard does not copy session material out of the keyring.
- **NFR-S3** — No telemetry, no analytics, no crash reporting to any third party. Diagnostics bundles are written locally and shared only by explicit user action.
- **NFR-S4** — Logs must redact file *contents* and full paths outside the sync root; file names within sync roots may be logged at debug level only.
- **NFR-S5** — Config, state DB and logs are created mode `0600` under the user's home; the daemon runs as the user, never as root.
- **NFR-S6** — Subprocess invocation of the CLI must use argument arrays, never shell string interpolation, so filenames cannot become command injection.

### Reliability & Data Integrity
- **NFR-R1** — **No data loss is the top-priority requirement.** No sync operation may leave the only copy of user data deleted or overwritten. Conflicts always produce a preserved copy.
- **NFR-R2** — The state database must survive an abrupt kill, power loss and full disks. Use SQLite in WAL mode with transactional state transitions; an interrupted sync resumes cleanly on restart.
- **NFR-R3** — The daemon must self-heal from CLI failures: exponential backoff on transient errors, clear terminal state on permanent ones, and never a silent stall. Any pair stuck for more than one cycle surfaces an error.
- **NFR-R4** — Disk-full, permission-denied and quota-exceeded must pause the affected pair with an actionable message, not crash the daemon.
- **NFR-R5** — A full re-scan must converge to the same result as incremental sync. A "verify and repair" path must exist for when state and reality disagree.

### Performance (ARM64 laptop/desktop budget)
- **NFR-P1** — Idle daemon: **under 80 MB RSS** and **under 0.5% CPU** on an ARM64 laptop with 50,000 tracked files.
- **NFR-P2** — Local change detected to upload started: **under 5 seconds** for a single file.
- **NFR-P3** — Full re-scan of 50,000 local files: **under 30 seconds** on NVMe.
- **NFR-P4** — Transfer throughput within **80% of raw `proton-drive` CLI** throughput for the same payload — the orchestration layer must not be the bottleneck.
- **NFR-P5** — Tray icon and main window must reflect state changes within 1 second; the GUI must never block on daemon I/O.
- **NFR-P6** — No polling loop may prevent laptop suspend or measurably shorten battery life at idle.

### Usability
- **NFR-U1** — First-run to first synced file in **under three minutes**, including sign-in.
- **NFR-U2** — Every error shown to the user states what happened, what it affects, and what to do next. No raw stack traces or CLI exit codes in the primary UI.
- **NFR-U3** — The config file and the GUI are equally authoritative; an external edit to the config is picked up without restarting the daemon.
- **NFR-U4** — The remote-latency limitation (PD-005) must be visible in the UI, not buried in documentation.
- **NFR-U5** — Uninstalling leaves all synced files intact on disk.

### Accessibility
- **NFR-A1** — The Avalonia UI is fully keyboard-navigable with a visible focus indicator and a logical tab order.
- **NFR-A2** — Text contrast meets **WCAG 2.1 AA** (4.5:1 body, 3:1 large text) in both light and dark themes; follows the system GTK theme preference.
- **NFR-A3** — Sync state is never conveyed by colour alone — icon shape and text label always accompany it, including Nautilus emblems.
- **NFR-A4** — Controls expose accessible names and roles to AT-SPI so Orca can read them.
- **NFR-A5** — The UI respects the system font scale and reduced-motion preference.

### Compatibility & Maintainability
- **NFR-C1** — Ubuntu **24.04 LTS and later**, `arm64`, GNOME/Wayland primary; X11 must work; other desktops degrade gracefully (no tray means the GUI window is still fully functional).
- **NFR-C2** — The `proton-drive` CLI is an **external dependency whose output format may change**. All parsing goes through a single versioned adapter with a compatibility check on startup and a contract test suite pinned to a known CLI version.
- **NFR-C3** — The transport is an interface (`IDriveTransport`), with the CLI as one implementation, so a native client can replace it without touching the engine.
- **NFR-C4** — The sync engine must be testable headlessly with a fake transport and a temp-directory filesystem — no network, no Proton account, no GUI in CI.
- **NFR-C5** — Licensed **GPLv3**, consistent with reuse of `windows-drive` source.

---

## 6. MVP Scope vs. Phase 2 Scope

| | **MVP (v1.0)** | **Phase 2 and beyond** |
|---|---|---|
| **Transport** | `proton-drive` CLI subprocess, `--json`, behind `IDriveTransport` | Native Drive SDK/API client; real change notifications |
| **File model** | Full local sync — every file on disk | On-demand files via FUSE; selective per-file pinning |
| **Sync direction** | Two-way, continuous | Unchanged |
| **Remote change detection** | On-demand + optional 15-min safety poll | Push/event-driven or efficient delta sync |
| **Accounts** | One Proton account | Multiple accounts, each with own pairs |
| **Conflicts** | Conflicted-copy, always keep both | Configurable policies; diff/merge UI for text |
| **Auth** | CLI-brokered browser login; keyring session | Unchanged (deliberately) |
| **UI** | Avalonia tray + status window; daemon-first | Richer activity view; transfer queue control; onboarding polish |
| **File manager** | Nautilus emblems + basic context menu | Dolphin, Thunar, Nemo; share-link and Docs actions in the menu |
| **Photos** | Not included | Photos backup with dedup and albums |
| **Sharing** | Not included | Links, invitations, shared-with-me |
| **Proton Docs** | Not included | Open/edit from file manager |
| **Bandwidth control** | None | Throttling, scheduling, metered/battery awareness |
| **Trash / versions** | Not surfaced | Browse and restore |
| **Platforms** | Ubuntu 24.04+ `arm64` | x86-64; Fedora/Arch; Flatpak, Snap, AUR |
| **Updates** | Manual — check and notify | Background auto-update with rollback |
| **Control surface** | TOML config + `halyardctl` + IPC | Unchanged, extended |

---

## 7. Risks & Mitigations

| # | Risk | Impact | Likelihood | Mitigation |
|---|---|---|---|---|
| **R1** | **Proton ships its official Linux GUI client** (publicly targeted late 2026 – early 2027) and makes this redundant within months. | High | High | Accepted deliberately. Keep MVP small and fast to build so the value window is worth it. Design the transport seam so Halyard can sit on top of Proton's engine if that becomes the better base. Position as a daemon/power-user tool, the niche the official client is least likely to serve first. Treat a graceful retirement — files stay on disk, uninstall is clean — as a feature, not a failure. |
| **R2** | **The CLI's JSON output or command surface changes** and silently breaks parsing. | High | Medium | Isolate all parsing in one adapter. Pin and assert a supported CLI version range at startup, refusing to sync with a loud message rather than corrupting state on unknown output. Maintain contract tests against recorded CLI fixtures and run them against the latest CLI on a schedule. Note the coupling: the CLI is a thin wrapper over `@protontech/drive-sdk`, and `filesystem list --json` prints that package's `NodeEntity` directly — so the JSON tracks an unstable 0.x npm package, not a documented CLI contract. |
| **R3** | **Data loss from a sync-engine bug** — the existential risk for any sync product. | Critical | Medium | Make NFR-R1 the acceptance gate: nothing ships that can delete the only copy. Property-based and adversarial tests (interrupt at every state transition, clock skew, case collisions, unicode normalisation, rename-vs-delete races). A `--dry-run` mode. A local trash directory for anything the engine removes, retained 30 days. Beta on non-critical data first. |
| **R4** | **CLI performance or rate limits** make sync of large trees impractical — one subprocess per operation is far heavier than a persistent API client. | High | Medium | Prototype throughput and per-operation overhead **before** committing to the architecture (spike in week 1). Batch operations where the CLI allows, reuse its interactive shell mode to amortise process startup, and cap concurrency. If the ceiling is too low, escalate the native transport into MVP. |
| **R5** | **Solo/very small team versus an 11-feature MVP** including a sync engine, an Avalonia GUI and a Nautilus extension. | High | High | Build strictly in dependency order: transport spike → engine + state DB → daemon + `halyardctl` → tray → main window → Nautilus → packaging. Every stage after the daemon is independently shippable, so an early release is always possible. Keep the Nautilus extension to emblems only. Ruthlessly refuse Phase 2 scope. |
| **R6** | **Legal/trademark exposure** from an unofficial client using the Proton name. | Medium | Low | Non-Proton product name and icon. Descriptive use only ("works with Proton Drive"), with a prominent "not affiliated with or endorsed by Proton" notice in the README, about box and package description. No Proton logos or assets. GPLv3 honoured for all reused `windows-drive` code, with attribution. |
| **R7** | **Reused GPLv3 code carries obligations** and its design assumes an API client we no longer have. | Medium | Medium | Ship under GPLv3 from commit one — no ambiguity. Treat `windows-drive` primarily as a *design reference* for the engine's state machine; copy code only where the boundary is clean and record provenance in file headers. |
| **R8** | **On-demand latency frustrates users** who expect changes from another device to appear immediately. | Medium | High | Set the expectation in the UI, not just the docs (NFR-U4). Make "Sync now" reachable in one click from the tray and one command from the shell. Ship the safety poll on by default. Prioritise real change detection early in Phase 2. |
| **R9** | **ARM64 gaps** — .NET/Avalonia on `arm64` Wayland, tray support under GNOME (which needs a shell extension), or a missing `libnautilus-python`. | Medium | Medium | Validate the whole stack on target hardware in week 1. Detect a missing tray host and fall back to a normal window plus a desktop notification, rather than appearing to do nothing. Declare the Nautilus extension an optional package dependency. |
| **R10** | **Session expiry or 2FA re-prompt stalls a headless machine** with no browser available. | Medium | Medium | Detect expiry explicitly and expose it as a first-class state via `halyardctl status --json` and a desktop notification, so automation can alarm on it. Document the headless re-auth path. Never silently retry into a lockout. |

---

## Open Questions

Questions 3 and 4 were resolved by reading the CLI and SDK source; see [`technical-blueprint.md` §1](technical-blueprint.md).

1. **Final product name and icon** — needed before any public release (R6). *Open.*
2. **CLI throughput ceiling** — *open, and now the only blocking unknown.* The binary is a Bun bundle, so per-operation spawn cost is real. Two mitigations exist before the sidecar is needed: multi-path commands (`filesystem upload ./dir/* /remote`) and the CLI's interactive REPL mode. The week-0 spike must measure all three.
3. ~~**CLI move/rename semantics**~~ — **Answered: server-side move and rename both exist** (`filesystem move`, `filesystem rename`). A rename is one operation, not download + upload + delete. Move and rename are first-class operation kinds in the engine, and folder moves are detected before their contents so relocating a large directory costs one call.
4. ~~**Remote metadata granularity**~~ — **Answered: revision ids are available.** `filesystem list --json` emits the SDK's full `NodeEntity`, and `filesystem info` returns "full node metadata including latest revision details". Remote change detection compares revision ids; local content hashing is retained to confirm local changes.
5. **Case-sensitivity and unicode normalisation** between the Linux filesystem and Proton Drive — *open*, needs a documented, tested policy. Collision **detection** ships in MVP regardless of which policy is chosen.
