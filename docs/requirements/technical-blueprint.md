# Technical Blueprint — Halyard

**Background sync client for Proton Drive on Ubuntu ARM64.**

| | |
|---|---|
| **Document** | Technical Blueprint |
| **Status** | Draft v1.1 — transport verified by build |
| **Date** | 2026-09-25 |
| **Repository** | `cgaine/proton-desktop-linux-arm64` |
| **Inputs** | [`product-definition.md`](product-definition.md) |
| **Domain model** | `/docs/requirements/domain-model.md` — **not yet written**. Section 4's entities are the blueprint's working model and must be reconciled when it exists. |

> **v1.1 corrects v1.0.** The first draft proposed referencing Proton's C# Drive SDK directly. A build attempt proved that impossible — it depends on a closed-feed package. The CLI-based transport chosen in the product definition stands. Section 1.1 records the evidence so nobody spends the day again.

---

## 1. Transport verification

Everything in this section was executed, not inferred. Commit `f28a93ce` of [`ProtonDriveApps/sdk`](https://github.com/ProtonDriveApps/sdk), .NET SDK 10.0.301, Node 24.15.0.

### 1.1 The C# SDK cannot be built externally — rejected

`client/cs/src/Proton.Drive.Sdk` looked ideal: MIT, `net10.0`, a full `Events/` namespace, and the SDK `windows-drive` is named for. It does not build.

```
$ dotnet build -c Release -r linux-arm64        # client/cs/src/Proton.Drive.Sdk
error NU1100: Unable to resolve 'Proton.Cryptography (>= 0.31.0)' for 'net10.0'.
error NU1100: Unable to resolve 'Proton.Cryptography (>= 0.31.0)' for 'net10.0/linux-arm64'.
PackageSourceMapping is enabled, the following source(s) were not considered: nuget.org
Build FAILED.  4 Error(s)
```

| Finding | Evidence |
|---|---|
| `Proton.Cryptography` is not on nuget.org | `api.nuget.org` → `BlobNotFound`; a search for "Proton" returns 63 unrelated packages, none from Proton AG |
| It has no source in the repo | Only a `PackageReference` in `Proton.Sdk.csproj` and a `PackageVersion` in `config/cs/Packages.props` |
| It is routed to a private feed | `nuget.config` maps `Proton.*` to a source key `Proton` that the file never defines — the internal feed is stripped from the public repo |
| It ships as native backends | `GoCryptoVersion 0.31.0`, `RustCryptoVersion 0.31.0-rust.1`, selected by `$(CryptoBackend)` |
| `windows-drive` consumes the SDK the same way | `sync/cs/Directory.Packages.props`: `<PackageVersion Include="Proton.Drive.Sdk" Version="0.26.0-rust" />` — a package, not a project reference. Also not public. |
| No public feed exists | No GitHub releases; org packages API requires auth; no feed documented in any README |

**Conclusion.** `client/cs` is published for transparency and audit, not for consumption. Building it needs Proton's internal feed. **Rejected — not available to an unofficial client.** Do not retry without a package source for `Proton.Cryptography`.

*Incidental:* the repo does not check out on Windows without `git config --global core.longpaths true` — `incubating/client/swift/…` exceeds `MAX_PATH`. The C# and JS trees are unaffected.

### 1.2 The JS SDK is genuinely consumable — verified

```
$ npm install @protontech/drive-sdk @protontech/crypto
added 14 packages … found 0 vulnerabilities
$ npx esbuild entry.mjs --bundle --platform=node --format=esm
BUNDLE OK (503395 bytes)
```

`@protontech/drive-sdk@0.21.3`, **MIT**, dependencies `ttag` + `@noble/hashes`, peer `@protontech/crypto` (public, 2.1.3). Its `ProtonDriveClient` exposes **55 methods**, including:

- **Events** — `subscribeToDriveEvents`, `subscribeToTreeEvents`, `iterateEvents`, `getEventScheduler`; `DriveEventType` = `node_created`, `node_updated`, `node_deleted`, `shared_with_me_updated`, `tree_refresh`, `tree_remove`, `fast_forward`
- **Structure** — `moveNodes`, `renameNode`, `createFolder`, `getNode`, `getNodeHierarchy`, `iterateFolderChildren`, `getAvailableName`
- **Content** — `getFileUploader`, `getFileDownloader`, `getFileRevisionUploader`, `iterateRevisions`, `restoreRevision`
- **Lifecycle** — `trashNodes`, `restoreNodes`, `deleteNodes`, `emptyTrash`
- **Backpressure** — `SDKEvent` = `transfersPaused`, `transfersResumed`, `requestsThrottled`, `requestsUnthrottled`

Two caveats that keep it out of MVP:

1. **Node cannot load it directly.** The package uses extensionless ESM imports (`./internal/uids`), which Node's resolver rejects — it is built for Bun and bundlers. A bundling step or Bun is mandatory.
2. **Auth is not published.** The CLI gets login from `proton-drive-sdk-account` (`file:../incubating/account/js`), which is `"private": true` and absent from npm. Its source is in-repo under MIT, but it is explicitly *incubating* — "not guaranteed to have stable interface across releases". Using it means owning the login flow against an unstable module, re-introducing exactly what the product definition removed from scope.

### 1.3 The CLI — confirmed, and richer than assumed

33 commands, enumerated from `cli/src/commands/`:

| Group | Commands |
|---|---|
| `auth` | `login`, `logout` |
| `filesystem` | `list`, `info`, `size`, `upload`, `download`, `copy`, **`move`**, **`rename`**, `delete`, `trash`, `restore`, `empty-trash`, `create-folder` |
| `sharing` | `status`, `invite`, `leave`, `remove`, `set-url`, `remove-url`, `report` |
| `invitation` | `list`, `accept`, `reject` |
| `album` / `photo` | `create`, `list`, `add-photo`, … / `upload`, `download`, `timeline` |
| `takeout` | `run` |

**`filesystem move` and `filesystem rename` exist.** This settles the product definition's highest-priority open question: renames are server-side, not delete-plus-reupload.

`filesystem list --json` prints the SDK's `NodeEntity` directly; `filesystem info` documents itself as "full node metadata including latest revision details" — so revision ids are available for cheap remote change detection.

**But the CLI exposes no event command.** `cli/src/events/manager.ts` calls `subscribeToDriveEvents` internally for its own cache coherence; there is no `events` group in the registry. **Events are unreachable through the CLI**, which is what forces polling in MVP.

---

## 2. Architecture decision records

### ADR-001 — Transport: drive the `proton-drive` CLI; plan a sidecar for Phase 2

**Status:** Accepted. Confirms the product definition. (v1.0's C# SDK proposal is withdrawn — §1.1.)

**Decision.** MVP uses `CliDriveTransport`, invoking the official `proton-drive` `linux-arm64` binary with argument arrays and `--json`. Phase 2 introduces `SidecarDriveTransport`: a long-lived Bun process hosting `@protontech/drive-sdk`, added behind the same interface when event-driven sync becomes the priority.

**Why the CLI for MVP.**

- It is the only **complete and supported** external path. Auth, crypto, session storage and retry are Proton's, and the binary is signed and shipped for `linux-arm64`.
- Its file-operation surface is sufficient: move, rename, copy, trash, restore, revisions (§1.3).
- It needs no second language in the build, no vendored incubating module, and no login flow of our own — decisive for a solo maintainer (R5).

**What it costs, accepted with eyes open.**

- **A process per operation.** The binary is a Bun bundle; spawn cost is real and is the dominant risk (R4).
- **No events → polling.** PD-005's on-demand model stands, and with it R8.

**Why the sidecar is Phase 2, not MVP.** It is the only route to real-time sync and it is proven feasible (§1.2) — but its auth module is unpublished and incubating. Taking that on at MVP trades a schedule risk the project can absorb for a correctness risk in the one area the product definition deliberately kept out of scope.

**Rejected:** the C# SDK (§1.1, unbuildable); reimplementing the API (re-introduces SRP and OpenPGP).

### ADR-002 — `IDriveTransport` is the seam, and it is load-bearing

**Status:** Accepted.

`IDriveTransport` in `Halyard.Core` is the only thing the engine knows. Given ADR-001 explicitly plans to swap the implementation, the seam is not speculative insurance — it is the migration path. One contract suite (§11) runs against the in-memory fake, the CLI transport and, when it exists, the sidecar. The interface is shaped around the **union** of both capability sets, with `SubscribeAsync` returning an empty stream on the CLI so the engine's event path is exercised from day one rather than bolted on later.

### ADR-003 — Daemon-first, GUI as a detached client

**Status:** Accepted. Confirms the product definition.

`halyardd` owns all state and does all work; `halyardctl` and the Avalonia GUI are clients over a Unix domain socket. Serves the power-user persona, makes headless the default rather than a mode, and mirrors the `InterProcessCommunication` split in `windows-drive`. Killing the GUI cannot interrupt sync.

### ADR-004 — SQLite for sync state, not flat files

**Status:** Accepted — deliberate deviation from the team's POC default.

The team convention is JSON flat files at POC phase. Wrong here, and not for phase reasons: the state tree needs **atomic multi-row transitions** (NFR-R2), **indexed lookup by path and remote id** over 50,000+ rows (NFR-P3), and **crash-consistency under `kill -9`**. A JSON file rewritten per change fails all three and makes NFR-R1 — no data loss — unachievable. **SQLite in WAL mode from commit one.** No flat-file phase, no `UseSqlDatabase` toggle.

### ADR-005 — Unix domain socket, newline-delimited JSON

**Status:** Accepted.

Socket at `$XDG_RUNTIME_DIR/halyard/daemon.sock`, mode `0600`, so the OS enforces "this user only" and no auth layer is needed. NDJSON over gRPC: payloads are small and low-frequency, `halyardctl --json` passes frames through nearly unchanged, and the protocol stays debuggable with `socat`.

### ADR-006 — Credentials stay with the CLI

**Status:** Accepted.

The CLI writes its session to the OS secret store under `ch.proton.drive/drive-sdk-cli`, controllable via `PROTON_DRIVE_CREDENTIALS_STORE`. **Halyard never reads it.** We track only whether `auth login` has succeeded and whether an operation failed as unauthenticated. This is the architectural reason the product can promise it never touches passwords or keys, and it is worth protecting: nothing in `Halyard.*` may ever parse the credential store.

---

## 3. Architecture overview

A single-user, single-machine **local daemon**. No server, no network service of our own, no multi-tenancy. The only remote is Proton Drive, reached only through the CLI.

```
┌───────────────────────────────────────────────────────────────────┐
│  User session (systemd --user)                                    │
│                                                                   │
│   ┌──────────────┐   ┌──────────────┐   ┌──────────────────────┐  │
│   │ Avalonia GUI │   │  halyardctl  │   │  Nautilus extension  │  │
│   │ (tray+window)│   │    (CLI)     │   │      (Python)        │  │
│   └───────┬──────┘   └──────┬───────┘   └──────────┬───────────┘  │
│           └─────────────────┴──────────────────────┘              │
│                    Unix domain socket (NDJSON)                    │
│   ┌─────────────────────────▼───────────────────────────────────┐ │
│   │                      halyardd (daemon)                      │ │
│   │   IPC server │ Scheduler │ Hosted services │ Composition    │ │
│   │  ┌────────────────────────────────────────────────────────┐ │ │
│   │  │                 Sync Engine (reconciler)               │ │ │
│   │  └──────┬──────────────────┬──────────────────┬──────────┘ │ │
│   │  ┌──────▼──────┐   ┌───────▼───────┐   ┌──────▼─────────┐  │ │
│   │  │ State store │   │ Drive         │   │ Linux platform │  │ │
│   │  │  (SQLite)   │   │ IDriveTrans.  │   │ (inotify/xdg)  │  │ │
│   │  └─────────────┘   └───────┬───────┘   └──────┬─────────┘  │ │
│   └────────────────────────────┼──────────────────┼────────────┘ │
│                    ┌───────────▼──────────┐  ┌────▼───────────┐  │
│                    │  proton-drive binary │  │  Local filesys │  │
│                    │  (subprocess, --json)│  │                │  │
│                    └───────────┬──────────┘  └────────────────┘  │
│                                │         ┌──────────────────┐    │
│                                │         │ GNOME Keyring    │    │
│                                │◄────────┤ (CLI owns this)  │    │
└────────────────────────────────┼─────────┴──────────────────┘────┘
                                 │ HTTPS, E2E encrypted
                        ┌────────▼────────┐
                        │  Proton Drive   │
                        └─────────────────┘
```

**Pattern:** layered / ports-and-adapters. Domain and engine are pure and testable with no filesystem, no subprocess and no SQLite. Every external dependency is an interface in `Halyard.Core`, implemented in an infrastructure project, composed in the daemon.

---

## 4. Solution structure

`Halyard.slnx` (`.slnx`, matching `windows-drive`).

```
proton-desktop-linux-arm64/
├─ Halyard.slnx
├─ Directory.Build.props            # TFM net10.0, nullable, analyzers, warnings-as-errors
├─ Directory.Packages.props         # central package management
├─ Halyard.ruleset
├─ src/
│  ├─ Halyard.Core/                 # domain + all abstractions. Zero I/O.
│  ├─ Halyard.Sync.Engine/          # reconciliation + orchestration (application layer)
│  ├─ Halyard.Sync.State/           # SQLite state store (infrastructure)
│  ├─ Halyard.Drive/                # IDriveTransport impls + CLI process management
│  ├─ Halyard.Platform.Linux/       # inotify, POSIX, XDG, D-Bus notifications
│  ├─ Halyard.Ipc/                  # wire contracts, shared by daemon and clients
│  ├─ Halyard.Daemon/               # halyardd — host, DI, hosted services, IPC server
│  ├─ Halyard.Cli/                  # halyardctl
│  └─ Halyard.App/                  # Avalonia GUI
├─ sidecar/                         # Phase 2 — Bun + @protontech/drive-sdk. Empty in MVP.
├─ integration/
│  └─ nautilus/halyard_nautilus.py  # Python file-manager extension
├─ packaging/
│  ├─ debian/                       # control, rules, postinst, prerm
│  ├─ systemd/halyard.service       # systemd --user unit
│  └─ desktop/                      # .desktop entry, icons
├─ tests/
│  ├─ Halyard.Core.Tests/
│  ├─ Halyard.Sync.Engine.Tests/    # the deep suite — property-based, adversarial
│  ├─ Halyard.Sync.State.Tests/
│  ├─ Halyard.Drive.Tests/          # transport contract suite + CLI fixtures
│  ├─ Halyard.Platform.Linux.Tests/
│  └─ Halyard.Daemon.Tests/         # end-to-end over a real socket + temp dirs
└─ docs/requirements/
```

No `extern/` submodule — v1.0 planned one for the C# SDK and §1.1 removed the need.

### 4.1 Project responsibilities

| Project | Responsibility | References |
|---|---|---|
| **Halyard.Core** | Entities, value objects, enums, domain rules, and **every abstraction**: `IDriveTransport`, `IStateStore`, `IFileSystem`, `IFileWatcher`, `IProcessRunner`, `IClock`, `INotifier`. No I/O. | — |
| **Halyard.Sync.Engine** | The reconciler: three-way comparison, change classification, conflict detection, operation planning, execution orchestration. Behind Core's interfaces only. | Core |
| **Halyard.Sync.State** | SQLite implementation of `IStateStore`. Schema, versioned migrations, transactions, queries. `Microsoft.Data.Sqlite`. | Core |
| **Halyard.Drive** | `CliDriveTransport`, the `proton-drive` process wrapper, `--json` parsing, exit-code and stderr mapping, version compatibility check. Later `SidecarDriveTransport`. **The only project that may start a process or parse Proton JSON.** | Core |
| **Halyard.Platform.Linux** | inotify watcher, POSIX filesystem ops, atomic writes, XDG paths, desktop notifications, systemd readiness. | Core |
| **Halyard.Ipc** | Request/response/event DTOs and NDJSON framing. Referenced by daemon and both clients so the wire type is defined once. | Core *(DTOs only)* |
| **Halyard.Daemon** | Composition root. DI, hosted services, scheduler, IPC server, logging, config binding. The only place concrete types are chosen. | all of the above |
| **Halyard.Cli** | `halyardctl`. Argument parsing, IPC client, human and `--json` rendering. | Ipc |
| **Halyard.App** | Avalonia GUI. Tray, main window, view models, IPC client. No business logic. | Ipc |

### 4.2 Dependency direction

```
                    Halyard.Core
                         ▲
        ┌────────────┬───┴────┬──────────────────┐
   Sync.Engine   Sync.State  Drive        Platform.Linux
        ▲            ▲        ▲                  ▲
        └────────────┴────┬───┴──────────────────┘
                          │
                   Halyard.Daemon ──────► Halyard.Ipc ◄─── Halyard.Cli
                                                      ◄─── Halyard.App
```

Rules, enforced by an architecture test in `Halyard.Core.Tests`:

1. **Core references nothing** — not SQLite, not Avalonia, not `System.Diagnostics.Process`.
2. **No infrastructure project references another.** They meet in the daemon.
3. **`Sync.Engine` references only `Core`.** If it needs something, that something becomes an interface in Core.
4. **No `System.Diagnostics.Process` outside `Halyard.Drive`**, and no Proton JSON shape is deserialised anywhere else. This is the analogue of the team's "no `fetch` outside `api/`" rule, and it is what makes ADR-002's swap a one-project change.
5. **No Avalonia type outside `Halyard.App`.**
6. **`Halyard.Cli` and `Halyard.App` never reference `Halyard.Daemon`** — only `Halyard.Ipc`.

---

## 5. Domain model (working)

Provisional until `/docs/requirements/domain-model.md` exists. Types live in `Halyard.Core`.

### 5.1 Entities

| Type | Purpose | Key fields |
|---|---|---|
| `SyncPair` | A configured local↔remote mapping. Aggregate root. | `SyncPairId`, `LocalRoot`, `RemotePath`, `IsPaused`, `ConflictPolicy`, `IgnoreRules`, `LastPolledAt` |
| `SyncNode` | One file or folder's last-agreed state. The state tree. | `SyncPairId`, `RelativePath`, `NodeType`, `RemoteNodeUid`, `RemoteRevisionId`, `LocalSize`, `LocalMtime`, `ContentHash`, `Status`, `UpdatedAt` |
| `PendingOperation` | A durable unit of work. The outbox. | `OperationId`, `SyncPairId`, `Kind`, `RelativePath`, `TargetPath`, `Attempt`, `NextAttemptAt`, `LastError`, `State` |
| `ConflictRecord` | A preserved divergence awaiting acknowledgement. | `ConflictId`, `SyncPairId`, `RelativePath`, `PreservedLocalPath`, `DetectedAt`, `AcknowledgedAt` |

### 5.2 Value objects and enums

`SyncPairId`, `OperationId`, `ConflictId` (typed ids over GUID). `RelativePath` — normalised, validated, always `/`-separated, never absolute, never containing `..`. `ContentHash` (BLAKE2b/SHA-256). `IgnoreRuleSet` (compiled gitignore-syntax matcher).

```
NodeType        = File | Folder
NodeStatus      = Synced | PendingUpload | PendingDownload | Conflicted | Ignored | Error
OperationKind   = Upload | Download | CreateRemoteFolder | CreateLocalFolder
                | MoveRemote | MoveLocal | TrashRemote | DeleteLocal
OperationState  = Queued | InFlight | Succeeded | Failed | Abandoned
ConflictPolicy  = PreserveBoth        // MVP default and only supported value
                | RemoteWins | LocalWins | Prompt   // reserved, Phase 2
ChangeSide      = None | Local | Remote | Both
```

`MoveRemote` is first-class because §1.3 confirmed `filesystem move` and `filesystem rename`. Without them a rename would degrade to delete-plus-reupload — the single largest avoidable cost in a sync engine, and now avoided.

### 5.3 Core abstractions

```csharp
public interface IDriveTransport
{
    Task<RemoteNode> GetNodeAsync(RemotePath path, CancellationToken ct);
    IAsyncEnumerable<RemoteNode> ListChildrenAsync(RemotePath path, CancellationToken ct);
    Task<RemoteNode> CreateFolderAsync(RemotePath parent, string name, CancellationToken ct);
    Task<RemoteNode> UploadAsync(string localPath, RemotePath destination,
                                 IProgress<TransferProgress>? progress, CancellationToken ct);
    Task DownloadAsync(RemotePath source, string localPath,
                       IProgress<TransferProgress>? progress, CancellationToken ct);
    Task MoveAsync(RemotePath source, RemotePath destination, CancellationToken ct);
    Task RenameAsync(RemotePath path, string newName, CancellationToken ct);
    Task TrashAsync(RemotePath path, CancellationToken ct);
    Task<AuthState> GetAuthStateAsync(CancellationToken ct);

    /// CLI transport yields nothing and completes. Sidecar transport streams changes.
    IAsyncEnumerable<RemoteChange> SubscribeAsync(CancellationToken ct);
}
```

`SubscribeAsync` exists in MVP and returns an empty stream. That is deliberate: the engine's event-driven path is written, wired and tested from day one, so Phase 2 is a transport swap rather than an engine change. `RemoteChange` is Core's own shape — `Created`, `Updated`, `Deleted`, `TreeRefresh` — matching the SDK's `DriveEventType` values found in §1.2 so the future mapping is mechanical.

Note the interface takes **paths and local file paths**, not streams. The CLI uploads and downloads files by path; forcing a `Stream` abstraction would mean spooling through temp files for no gain. The sidecar can serve the same shape.

---

## 6. The sync engine

### 6.1 Three-way reconciliation

Three observations per path: **L** (local now), **R** (remote now), **S** (last-agreed state).

| L vs S | R vs S | Classification | Action |
|---|---|---|---|
| same | same | unchanged | none |
| changed | same | local change | push (`Upload` / `MoveRemote` / `TrashRemote`) |
| same | changed | remote change | pull (`Download` / `MoveLocal` / `DeleteLocal`) |
| changed | changed, **same result** | convergent | update S only, no transfer |
| changed | changed, **different** | **conflict** | preserve both (§6.4) |
| absent | absent | tombstone | remove from S |

Local change detection: **size or mtime differs → candidate; confirm with `ContentHash`.** Hash-confirm avoids re-uploading a merely-touched file and makes the convergent case detectable. Remote side compares `RemoteRevisionId` from `filesystem list --json` first — cheap and authoritative.

Deciding *delete* versus *never-seen* is the classic trap. A path absent from L but **present in S** was deleted locally; absent from L and **absent from S** is new remotely. This is why S must be transactionally correct — a lost state row turns a delete into a re-download, or a re-upload of a file the user deleted.

### 6.2 Move detection

With `MoveAsync`/`RenameAsync` available, the reconciler must actually *detect* moves rather than seeing delete+create. Within one reconcile pass, a disappeared path and an appeared path matching on `ContentHash` and size are paired into a single `MoveRemote`. Folder moves are detected before their contents, so moving a 10,000-file directory is one operation rather than 20,000. This is the largest single performance lever in the engine and it is why `OperationKind` carries move variants at all.

### 6.3 Pipeline

```
 inotify events ──► Debouncer ──┐
                                ├─► Rescan(pair, scope) ─► Reconciler ─► Plan
 Poll timer / Sync now ─────────┤                                          │
 SubscribeAsync (Phase 2) ──────┘                                          ▼
                                                            PendingOperations (SQLite)
                                                    Channel<Operation>     │
                                                        ┌──────────────────▼────┐
                                                        │  Transfer workers (N) │
                                                        └───────────┬───────────┘
                                                                    ▼
                                            apply → update S in one transaction → emit IPC event
```

- **Debouncer** — coalesces inotify bursts over a 500 ms quiet window per pair, and recognises write-temp-then-`rename` so an atomic editor save produces one `Upload`, not `Delete`+`Create`. Prefer `IN_CLOSE_WRITE` over `IN_MODIFY`.
- **Planning is pure.** The reconciler takes three trees and returns operations. No I/O. This is what makes §11's property suite possible.
- **The outbox is durable.** Operations persist before execution, so a crash mid-transfer resumes rather than re-planning from a half-applied state.
- **Apply-and-record is one transaction.** The state row advances in the same SQLite transaction that marks the operation succeeded. There is no window where the file moved but S disagrees.
- **Ordering.** Folder creates before children; deletes after; moves before content changes at the destination.
- **Concurrency.** Bounded workers (default 4), per-path exclusivity. Because each operation is a process (§7), worker count is also the process-concurrency cap.

### 6.4 Conflicts — PD-006

On `Both` with divergent results:

1. Rename the local file **in place** to `name (conflict <yyyy-MM-dd HH-mm>).ext` — local, no network, cannot fail halfway.
2. Download the remote version to the canonical name.
3. Queue an `Upload` of the preserved copy.
4. Insert a `ConflictRecord`; surface over IPC; badge the tray.

Order is not negotiable: **the local file is preserved before anything is downloaded over it.** If the process dies after step 1 the user has two files and a re-plan reconciles them; after an overwrite, data would be gone. For remote-side name collisions the CLI's own `--conflict-strategy` and the SDK's `getAvailableName` apply; the *local* naming format is ours because it is user-visible.

### 6.5 Deletes and the safety net

Local deletes move to `~/.local/share/halyard/trash/<pair>/<timestamp>/<path>` rather than unlinking (R3), retained 30 days, swept at startup. Remote deletes use `filesystem trash`, never `delete` — Proton's trash is the safety net on that side and `filesystem restore` exists.

---

## 7. The CLI transport

`Halyard.Drive` is the blast radius for every CLI assumption.

### 7.1 Invocation

- **Argument arrays only**, never a shell string — a filename containing `;` or `$(…)` must never become a command (NFR-S6).
- `--json` on every call; parse with source-generated `System.Text.Json` contexts into `Halyard.Drive`'s own DTOs, then map to Core types. **Proton's JSON shapes never leave this project.**
- Map exit codes and stderr to a Core `TransportFailure` taxonomy: `Unauthenticated`, `NotFound`, `Conflict`, `QuotaExceeded`, `RateLimited`, `Transient`, `Permanent`. The engine reacts to that taxonomy, never to an exit code.
- Timeouts per operation class; kill the process group on cancellation so a hung transfer cannot leak.
- `--skip-thumbnails` on upload by default — thumbnail generation is wasted work for a sync client and costs CPU on ARM.

### 7.2 Version compatibility

`proton-drive version` is read at startup and checked against a supported range. Outside it, the daemon **refuses to sync and says so** rather than parsing unknown output into a state tree (R2). The supported range is a constant in `Halyard.Drive` updated alongside the recorded fixtures in §11.

### 7.3 Discovery and process cost

The binary is located by config `drive.cli_path`, then `$PATH`, then known install locations; if missing, the daemon reports an actionable error and stays up (PD-011 — detect and prompt).

Process spawn is the dominant performance unknown (R4). Two mitigations, in order:

1. **Batch at the command level.** `filesystem upload ./dir/* /remote` accepts multiple paths, and the CLI has its own `transferQueue`. Prefer one invocation over N wherever the operation set allows.
2. **Amortise via the REPL.** The CLI has an interactive shell mode (`cli/src/cli/repl.ts`) with command abbreviation. If step 0's measurement shows spawn cost dominating, a long-lived REPL process driven over stdin/stdout is the fallback — worse to parse, far cheaper per operation. Both live behind `IDriveTransport`; neither changes the engine.

If neither closes the gap, that is the signal to pull the Phase-2 sidecar forward.

---

## 8. Data store — ADR-004

**SQLite via `Microsoft.Data.Sqlite`**, one database per user at `~/.local/share/halyard/state.db`.

```
PRAGMA journal_mode = WAL;      -- crash-consistency + concurrent readers (NFR-R2)
PRAGMA synchronous  = NORMAL;   -- WAL-safe; FULL is unnecessary and costs on ARM flash
PRAGMA foreign_keys = ON;
PRAGMA busy_timeout = 5000;
```

| Table | Purpose | Notes |
|---|---|---|
| `SchemaVersion` | Migration bookkeeping | single row |
| `SyncPairs` | Configured pairs | config file is source of truth; this is the runtime projection |
| `SyncNodes` | The state tree **S** | PK `(SyncPairId, RelativePath)`; index on `(SyncPairId, RemoteNodeUid)` and on `(SyncPairId, ContentHash)` for move detection (§6.2) |
| `PendingOperations` | The outbox | index on `(State, NextAttemptAt)` |
| `Conflicts` | Unacknowledged conflicts | |
| `TrashEntries` | Local trash index for sweeping | |

`SyncNodes` is the hot table: 50,000+ rows, read fully per rescan, written per applied operation. `RelativePath` is stored normalised so ordinal comparison is correct and prefix queries drive subtree operations.

**Migrations:** versioned SQL scripts as embedded resources, applied in order inside a transaction at startup — the team's existing convention, and right here. No ORM: the access pattern is a handful of hand-tuned statements, and an ORM would add a dependency, startup cost and query unpredictability for nothing. Dapper is permitted for mapping; raw `SqliteCommand` on hot paths.

The state DB is **derived data**. On corruption or failed migration, rename it aside and rebuild by full rescan — slow but always correct, and a tested path (§11), not a theory.

Not stored here: credentials (ADR-006), configuration (§9), logs (§13), file content (the filesystem).

---

## 9. Configuration

**TOML** at `~/.config/halyard/config.toml`, via `Tomlyn`. The file is the source of truth; the GUI edits it; the daemon watches and reloads without restart (NFR-U3).

The team's standard `appsettings.json` schema (`Jwt`, `AllowedOrigins`, `AdminEmails`, `UseSqlDatabase`, `BlobStorage`, `DataPath`) **does not apply** and must not be imported. Every key presumes a hosted web API: no JWT because we issue no tokens, no CORS because we serve no browser, no blob storage, no data-store toggle (ADR-004). Carrying it over would be cargo-culting.

```toml
[daemon]
worker_count       = 4          # concurrent transfers == concurrent CLI processes
debounce_ms        = 500
log_level          = "information"

[drive]
transport          = "cli"      # "cli" (MVP) | "sidecar" (Phase 2)
cli_path           = ""         # empty = discover on PATH
skip_thumbnails    = true

[sync]
poll_interval      = "15m"      # remote safety poll; 0 disables (PD-005)
poll_on_start      = true

[[sync_pair]]
name               = "documents"
local_root         = "~/ProtonDrive/Documents"
remote_path        = "/my-files/Documents"
paused             = false
conflict_policy    = "preserve-both"
ignore             = ["*.tmp", ".~lock.*", "node_modules/", ".git/"]
```

Precedence: command-line flag → environment (`HALYARD_*`) → config file → defaults. Validated on load and reload; an invalid file **keeps the previous good configuration running** and reports the error rather than stopping sync.

---

## 10. IPC contract

Unix domain socket, `$XDG_RUNTIME_DIR/halyard/daemon.sock`, mode `0600`. NDJSON, one object per line, source-generated `System.Text.Json` contexts.

```
→ {"id":"1","method":"status.get"}
← {"id":"1","ok":true,"result":{"state":"syncing","pairs":[...],"conflicts":2}}

→ {"id":"3","method":"path.status","params":{"paths":["/home/u/ProtonDrive/a.txt"]}}   # Nautilus
← {"event":"transfer.progress","params":{"path":"a.txt","sent":81920,"total":204800}}
← {"event":"conflict.detected","params":{"pair":"documents","path":"notes.md"}}
```

| Method | Purpose |
|---|---|
| `status.get` | Daemon and per-pair state |
| `sync.now` | Force reconcile, optionally one pair — **the primary remote-pull trigger in MVP** |
| `pair.list` / `pair.add` / `pair.remove` / `pair.pause` / `pair.resume` | Pair management |
| `conflict.list` / `conflict.acknowledge` | Conflict handling |
| `path.status` | Batch path → status. The Nautilus hot path; fast and non-blocking. |
| `auth.status` / `auth.login` / `auth.logout` | Session lifecycle — `auth.login` shells the CLI's browser flow |
| `diagnostics.collect` | Build a support bundle |

Events pushed to subscribers: `sync.state`, `transfer.progress`, `conflict.detected`, `error.raised`, `auth.expired`.

The protocol is **versioned** (`halyard.version`). A GUI newer than a running daemon after a package upgrade must fail with a clear message, not misbehave.

---

## 11. Security

Most of the posture comes from *not* handling things. §3's diagram has no path from Halyard to a password.

- **Credentials — ADR-006.** The CLI owns the session in the OS keyring. Halyard never reads, copies or logs it, and nothing in `Halyard.*` may parse the credential store. Headless re-auth is surfaced as `auth.expired`, never worked around.
- **Command injection — NFR-S6.** Argument arrays only (§7.1). A filename is data. Covered by a test using adversarial names (`; rm -rf ~`, `$(id)`, newlines, NUL).
- **Path validation.** `RelativePath` rejects absolute paths, `..` and NUL. A remote node whose name would escape the sync root is refused and reported, not written. This is the one place a hostile remote reaches the local filesystem — validated in Core, tested adversarially.
- **Logs.** A redacting enricher strips absolute paths outside sync roots and token-shaped strings. File names inside sync roots at `Debug` only (NFR-S4). **CLI stderr is treated as untrusted text** — captured, redacted and never interpolated into a shell or a log format string.
- **Telemetry.** We add none, and send none. The CLI's own telemetry is Proton's and governed by the user's Proton account settings — state this plainly in the README rather than implying we can switch it off.
- **File modes.** Config, state DB, logs and socket `0600`; directories `0700`. Asserted at startup and corrected.
- **Never root.** A `systemd --user` unit; the daemon refuses to start as uid 0.
- **Symlinks** not followed across the sync root boundary; skipped with a warning in MVP.

---

## 12. Testing strategy

NFR-R1 — no data loss — is the acceptance gate, so the suite is part of the architecture.

| Layer | Approach |
|---|---|
| **`Halyard.Core`** | Unit tests, plus an **architecture test** (NetArchTest) enforcing §4.2: Core references nothing; no `Process` outside `Halyard.Drive`; no Avalonia outside `Halyard.App`; clients never reference the daemon. |
| **`Halyard.Sync.Engine`** | The deep suite. The reconciler is pure, so: **property-based tests** (FsCheck) over generated (L, R, S) triples asserting the invariants below; table-driven tests for every row of §6.1; **move-detection tests** (§6.2); adversarial cases — case-only renames, NFC/NFD pairs, file replaced by directory, move cycles, clock skew, mtime granularity. |
| **`Halyard.Sync.State`** | Real SQLite in a temp dir. Crash simulation: abort mid-transaction, assert recovery. Migration tests forward from each shipped version. |
| **`Halyard.Drive`** | One **contract suite** run against the in-memory fake (PR) and the real binary with a live test account (nightly). Plus **recorded `--json` fixtures** per supported CLI version, so an output change fails a fast test instead of corrupting a state tree (R2). Plus the injection tests from §11. |
| **`Halyard.Daemon`** | End-to-end over a real socket with temp dirs and the fake transport: add a pair, create files, assert convergence; kill and restart mid-sync, assert resumption. |

**Invariants asserted by the property suite:**

1. **No data loss.** Every distinct content present at the start exists somewhere at the end — local, remote, conflict copy, or trash.
2. **Convergence.** Running to quiescence twice equals once (idempotence).
3. **No spurious transfers.** An unchanged tree plans zero operations.
4. **Move preservation.** A pure rename plans exactly one `MoveRemote` and zero transfers.
5. **Crash safety.** Interrupting after any single operation and re-planning converges to the same final state.

Engine tests need no network, no Proton account and no GUI, so they run on every push (NFR-C4).

---

## 13. Platform integration

| Concern | Implementation | Notes |
|---|---|---|
| **File watching** | `inotify` via `FileSystemWatcher`, P/Invoke fallback if recursive behaviour proves insufficient | One watch per directory |
| **Watch limits** | Read `/proc/sys/fs/inotify/max_user_watches` at startup; compare to directory count; if short, error with the `sysctl` fix | A truncated watch set is a silent sync failure — the worst class of bug (R13) |
| **Atomic local writes** | Write `.halyard-tmp-<guid>` in the destination directory, `fsync`, `rename(2)` | Same-filesystem rename is atomic. Never write in place. |
| **Paths** | XDG: config `~/.config/halyard`, data `~/.local/share/halyard`, runtime `$XDG_RUNTIME_DIR/halyard`, logs `~/.local/state/halyard` | Honour env vars |
| **Notifications** | `org.freedesktop.Notifications` over D-Bus | Degrade silently if absent |
| **Service lifecycle** | `systemd --user`, `Type=notify`, `sd_notify` readiness and watchdog | `Microsoft.Extensions.Hosting.Systemd` |
| **Case / unicode** | Treat local as case-sensitive; detect remote collisions differing only by case or NFC/NFD and surface as errors rather than aliasing | R17 |

### 13.1 Nautilus extension

`integration/nautilus/halyard_nautilus.py` — a `nautilus-python` extension implementing `InfoProvider` (emblems) and `MenuProvider`. It connects to the same socket and asks for the status of displayed paths; it never scans and never reads the state DB.

It runs **inside the Nautilus process**, so it must never block. Non-blocking IPC with a short timeout, per-path cache with a TTL, `update_file_info` returns immediately with whatever is cached and refreshes asynchronously. A slow or dead daemon degrades to *no emblem*, never a frozen file manager. Packaged as an optional dependency (`python3-nautilus`).

---

## 14. Build, packaging and release

Replaces the skill's Azure topology — there is no server. The artefact is a package on a user's machine.

```bash
dotnet publish src/Halyard.Daemon -c Release -r linux-arm64 --self-contained \
       -p:PublishSingleFile=true -p:PublishTrimmed=true
dotnet publish src/Halyard.App    -c Release -r linux-arm64 --self-contained
```

Self-contained so there is no .NET runtime prerequisite (PD-011). NativeAOT for the daemon is a *measured* option, not a default — with the C# SDK gone the dependency set is small and AOT-friendly, and NFR-P1's 80 MB idle budget is the reason to try. The GUI stays JIT in MVP.

**Package:** one `halyard_<version>_arm64.deb`.

| Content | Path |
|---|---|
| Daemon | `/usr/lib/halyard/halyardd` |
| GUI | `/usr/lib/halyard/halyard` |
| CLI symlink | `/usr/bin/halyardctl` |
| systemd user unit | `/usr/lib/systemd/user/halyard.service` |
| Desktop entry + icons | `/usr/share/applications/`, `/usr/share/icons/hicolor/` |
| Nautilus extension | `/usr/share/nautilus-python/extensions/` |

`Recommends: python3-nautilus`. **`proton-drive` is not bundled in MVP** — detected at runtime and prompted for, per PD-011, which keeps us clear of redistributing Proton's binary (R6). `postinst` runs `systemctl --user daemon-reload` but does **not** auto-enable the service. **Uninstall never touches `~/ProtonDrive` or any sync root** (NFR-U5); purge removes config and state only.

**CI/CD** on GitHub Actions **`ubuntu-24.04-arm`** runners — native ARM64, no QEMU, and the suite runs where inotify and file modes actually behave.

| Workflow | Trigger | Does |
|---|---|---|
| `ci.yml` | push, PR | build, analyzers, unit + property + architecture tests, CLI fixture tests |
| `integration.yml` | nightly | live-account transport contract suite against the real binary |
| `cli-watch.yml` | weekly | fetch the latest `proton-drive`, run the contract suite, open an issue on drift — turns R2 into a scheduled check instead of a user bug report |
| `release.yml` | tag `v*` | publish, build `.deb`, sign, GitHub Release, apt metadata |

**Channels:** dev (`dotnet run`), beta (pre-release `.deb`, apt `beta`), stable (Release `.deb`, apt `stable`). MVP notifies about updates; it does not install them.

---

## 15. Observability

`Microsoft.Extensions.Logging` + Serilog. Two sinks: **journald** (so `journalctl --user -u halyard` works as any Linux admin expects) and a rolling file at `~/.local/state/halyard/logs/`, 7 × 10 MB. Structured throughout, with `SyncPairId`, `OperationId` and `RelativePath` as properties, so `journalctl -o json` filtering works.

Every CLI invocation logs at `Debug` with its argument array (redacted), duration and exit code — the audit trail for diagnosing transport problems, and the raw material for the R4 measurement.

`halyardctl diagnostics` collects: version, `proton-drive version`, redacted config, schema version and row counts, last 1,000 log lines, pending-operation and conflict summaries, watch-limit status, environment facts. Written locally for review (§11).

Counters via `status.get`: files tracked, pending operations, transfer rates, conflicts, last successful sync per pair, **and time-since-last-successful-remote-poll** — the key health signal, because a stalled poll loop looks exactly like "nothing changed".

---

## 16. Product-definition deltas

Verification confirmed the product definition's transport decision, so v1.0's large delta list is withdrawn. What remains is small and factual.

| Item | Change |
|---|---|
| **Open Question 2** — CLI throughput ceiling | **Refined, still open.** The binary is a Bun bundle, so spawn cost is real; multi-path commands and REPL mode are the two mitigations (§7.3). Still the week-0 measurement. |
| **Open Question 3** — server-side move? | **Answered: yes.** `filesystem move` and `filesystem rename` exist. Renames are one operation; `MoveRemote`/`MoveLocal` are first-class (§5.2, §6.2). |
| **Open Question 4** — revision id or hash? | **Answered: revision ids exist.** `filesystem list --json` emits the SDK's `NodeEntity`; `filesystem info` includes latest revision details. Local hashing stays for change confirmation. |
| **"Native transport"** (Additional features) | **Made concrete.** A Bun sidecar over `@protontech/drive-sdk` — MIT, on npm, verified to install and bundle (§1.2). Not the C# SDK, which is unbuildable externally (§1.1). |
| **"Real-time remote change detection"** (Additional features) | **Made concrete.** `subscribeToDriveEvents` / `subscribeToTreeEvents`, `DriveEventType` — reachable only via the sidecar, not the CLI (§1.3). |
| **R2** mitigation | Add: the CLI wraps `@protontech/drive-sdk`, so its JSON tracks that package's `NodeEntity`. Pin a supported version range and keep recorded fixtures (§7.2, §12). |

Unchanged: CLI transport, full local sync, PD-005's on-demand model, preserve-both conflicts, daemon-first, single account, Avalonia, GPLv3, ARM64, and every deferred feature. R4 and R8 remain open.

---

## 17. Risks and mitigations

Product-level risks R1, R3, R5, R6, R7, R9, R10 stand as recorded in the product definition. R4 and R8 stand — they were not closed. Architecture-level additions:

| # | Risk | Impact | Likelihood | Mitigation | Owner |
|---|---|---|---|---|---|
| **R11** | **The transport upgrade path is narrower than it looks.** The C# SDK is unbuildable (§1.1) and the sidecar's auth module is unpublished and incubating — so if the CLI proves too slow, the fallback is not free. | High | Medium | Measure at step 0, before the engine exists (§18). Exhaust §7.3's two mitigations first. Keep `IDriveTransport` genuinely narrow so the sidecar remains a one-project change. Track `proton-drive-sdk-account` for publication — it graduating changes this risk materially. | Maintainer |
| **R12** | **Poll loop stalls silently.** A failed poll that is retried forever looks identical to "nothing changed remotely". | High | Medium | Track time-since-last-successful-poll as a first-class health metric (§15); surface in UI and `status.get` past a threshold; never let a poll failure be log-only. | Maintainer |
| **R13** | **inotify watch exhaustion.** A large tree exceeds `max_user_watches`; watches fail silently and local changes are missed. | High | Medium | Count directories against the limit at startup and before adding a pair; refuse with the exact `sysctl` remedy rather than degrading. Periodic rescan as backstop. Never treat a failed registration as a warning. | Maintainer |
| **R14** | **State DB corruption or failed migration** leaves the engine planning against a wrong S — the fastest route to data loss. | Critical | Low | WAL + transactional apply-and-record; integrity check at startup; treat the DB as derived and rebuild by rescan. Recovery is a tested path (§12). | Maintainer |
| **R15** | **Trim/AOT breaks something only in the packaged build**, not under `dotnet run`. | Medium | Low | Source-generated JSON contexts throughout. CI smoke-tests the *published, packaged* artefact. GUI stays JIT in MVP. | Maintainer |
| **R16** | **Nautilus extension blocks the file manager** — it runs in-process. | Medium | Medium | Non-blocking IPC with short timeout, per-path cache, async refresh; degrade to no emblem. Test against a deliberately stalled daemon. | Maintainer |
| **R17** | **Case and unicode collisions** between a case-sensitive local filesystem and Proton Drive. | Medium | Medium | Detect and surface as a per-path error rather than aliasing two nodes to one path. Settle normalisation policy before beta; detection ships in MVP. | Maintainer |
| **R18** | **GNOME has no tray by default** — `StatusNotifierItem` needs a shell extension, so the tray-centric UX (PD-008) may not appear at all. | Medium | High | Detect tray host at startup; if absent, fall back to a window plus notifications and say so once. Never let the tray be the only route to any function. | Maintainer |
| **R19** | **CLI process-per-operation cost compounds with concurrency** — 4 workers means 4 Bun processes, each with its own memory, against NFR-P1's 80 MB idle and a laptop battery. | Medium | Medium | Worker count is the process cap and is configurable. Measure RSS under load at step 0, not at beta. Prefer multi-path invocations (§7.3). | Maintainer |

---

## 18. Code organisation conventions

Team conventions, unchanged: one type per file; file name matches type name; one interface per file; no bundling of unrelated entities; folders by layer and responsibility; no cross-layer shortcuts (§4.2).

Project-specific additions, each enforced by the architecture test:

- **No `System.Diagnostics.Process` outside `Halyard.Drive`.**
- **No Proton JSON shape deserialised outside `Halyard.Drive`.**
- **No Avalonia type outside `Halyard.App`.**
- **No SQL outside `Halyard.Sync.State`.**
- **No direct `System.IO` in `Halyard.Sync.Engine`** — it goes through `IFileSystem`, which is what makes the engine testable in memory.
- **No blocking calls in the Nautilus extension.**

---

## 19. Issue hierarchy

The Roadmap → Epic → Feature → User Story relationship is expressed with **GitHub sub-issues**, so the hierarchy lives in GitHub rather than in checklists in issue bodies. The repository is `cgaine/proton-desktop-linux-arm64`.

The sub-issues API takes the child's **database id**, not its issue number. Resolve the id first:

```bash
# 1. Resolve the child issue number to its database id
CHILD_ID=$(gh api repos/cgaine/proton-desktop-linux-arm64/issues/<child-number> --jq '.id')

# 2. Attach it to the parent
gh api -X POST repos/cgaine/proton-desktop-linux-arm64/issues/<parent-number>/sub_issues \
  -F sub_issue_id="$CHILD_ID"

# List a parent's children
gh api repos/cgaine/proton-desktop-linux-arm64/issues/<parent-number>/sub_issues \
  --jq '.[] | "\(.number)\t\([.labels[].name] | join(","))\t\(.title)"'
```

`github-issue-creation` attaches a deliverable's pieces as sub-issues of the parent. The `bug-fix` skill attaches a bug issue as a sub-issue of the affected feature issue when one can be identified; otherwise the bug stands alone. Both must pass the resolved database id, as above.

---

## 20. Build order

Dependency-ordered, and chosen so everything after step 4 is independently shippable — the mitigation for R5.

| Step | Deliverable | Proves |
|---|---|---|
| **0** | **Transport spike.** Drive the real `proton-drive` binary on ARM64: time 1,000 small-file uploads one-per-process vs. batched vs. REPL; measure RSS at 4 concurrent. | **Whether ADR-001 survives.** Everything downstream assumes the answer. Do this before writing an engine. |
| **1** | `Halyard.Core` + `Halyard.Drive` + contract suite (fake + live account) + JSON fixtures | Auth, list, upload, download, **move**, **rename**, trash all work and are pinned |
| **2** | `Halyard.Sync.State` + migrations + crash tests | State survives what NFR-R2 requires |
| **3** | `Halyard.Sync.Engine` + property suite incl. move detection | The invariants in §12 hold. **The gate for everything downstream.** |
| **4** | `Halyard.Daemon` + `Halyard.Ipc` + `Halyard.Cli` | **First shippable product** — headless sync, the power-user persona served end to end |
| **5** | `Halyard.Platform.Linux` — inotify, notifications, systemd hardening | Local changes sync instantly; watch limits handled |
| **6** | `Halyard.App` — tray and main window | PD-008 |
| **7** | Nautilus extension | PD-009 — the most droppable item if schedule bites |
| **8** | `.deb`, apt repo, release workflow | PD-011 |
| **9+** | *(Phase 2)* Bun sidecar → event-driven sync | Closes R8 and PD-005's limitation |

---

## 21. Output checklist

- [x] Transport verified by execution, with the rejected option's evidence recorded (§1)
- [x] Architecture pattern named and justified (§3, ADR-001…006)
- [x] Project structure, responsibilities and dependency direction defined (§4)
- [x] Layering: domain / application / infrastructure / host (§4.1)
- [x] Domain entities and core abstractions specified (§5)
- [x] Core algorithm — reconciliation, move detection, pipeline, conflicts, deletes (§6)
- [x] Transport implementation: invocation, safety, version pinning, process cost (§7)
- [x] Data store decided with rationale and schema; migration strategy (§8, ADR-004)
- [x] Configuration strategy, with the inapplicable team schema explicitly rejected (§9)
- [x] IPC contract, transport, framing and versioning (§10, ADR-005)
- [x] Security: credentials, injection, path validation, logging, file modes (§11)
- [x] Testing strategy tied to NFR-R1, with invariants stated (§12)
- [x] Platform integration: inotify, XDG, systemd, D-Bus, Nautilus (§13)
- [x] Build, packaging, CI and release channels (§14)
- [x] Observability and the health signal that matters (§15)
- [x] Deltas back to `product-definition.md` listed and applied (§16)
- [x] Architecture risks with mitigations and owners (§17)
- [x] Code-organisation conventions, machine-enforced (§18)
- [x] Issue hierarchy with the exact `gh api` sub-issue commands (§19)
- [x] Build order with the riskiest assumption first (§20)
- [ ] Cross-referenced with `/docs/requirements/domain-model.md` — **blocked: not yet written** (§5 is provisional)
