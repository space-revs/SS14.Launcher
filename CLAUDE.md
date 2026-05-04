# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository overview

This is the **SS14 Launcher** — the desktop application users run to log in, browse servers, download the matching engine + content versions, and start the game. It is the entry point for connecting to any Space Station 14 server; the actual game (`RobustToolbox` engine + content pack) is downloaded on demand and side-loaded.

Built on **Avalonia 11 + ReactiveUI** (MVVM), targeting **.NET 10** (`Launcher.props` pins `<TargetFramework>net10.0</TargetFramework>`). Uses central package management via `Directory.Packages.props`. Nullable reference types are enabled.

Submodule: `Robust.LoaderApi/` (the contract between the launcher and the in-process loader). Initialize with `git submodule update --init --recursive`.

## Common commands

```bash
# Restore + build everything (the SS14.Launcher.sln solution)
dotnet restore
dotnet build --configuration Release

# Run the launcher in dev mode
dotnet run --project SS14.Launcher

# Tests (the only test project)
dotnet test SS14.Launcher.Tests/SS14.Launcher.Tests.csproj -v n

# Single test
dotnet test SS14.Launcher.Tests/SS14.Launcher.Tests.csproj --filter "FullyQualifiedName~UriHelperTests"

# Produce release artifacts (mirrors what publish-release.yml does)
./publish.py linux                       # all Linux RIDs
./publish.py windows --x64-only          # win-x64 only
./publish.py osx                         # macOS x64 + arm64
./publish.py windows linux osx           # everything
```

`publish.py` orchestrates `dotnet publish` for each RID, sets the Windows subsystem on the produced exes (`exe_set_subsystem.py`), copies in a bundled .NET runtime (`download_net_runtime.py`), and zips the result into `bin/publish/`. CI in `.github/workflows/build-test.yml` runs `dotnet build --configuration Release --no-restore` and `dotnet test` on every push/PR.

## Useful environment variables (development)

From `Readme.md`:

- `SS14_LAUNCHER_APPDATA_NAME=launcherTest` — change the user-data directory the launcher reads/writes, so dev work doesn't trash your real launcher install.
- `SS14_LAUNCHER_OVERRIDE_AUTH=https://.../` — point at a local/dev auth API instead of `auth.spacestation14.com`.

## Architecture

### Three executables in one solution

The repo produces three distinct binaries that work together at runtime:

1. **`SS14.Launcher/`** — the actual launcher app (Avalonia GUI). This is what most code lives in.
2. **`SS14.Launcher.Bootstrap/`** (Windows-only) — a tiny native-AOT/win-targeted shim that finds the bundled .NET runtime and launches `SS14.Launcher.exe` against it. Lets the launcher ship without requiring a system .NET install.
3. **`SS14.Loader/`** — a separate process the launcher spawns to actually run the game. The loader uses `Robust.LoaderApi` (the submodule) to load the downloaded RobustToolbox engine + content into a fresh AppDomain/process, isolating it from the launcher.

`SS14.Launcher.FakeRobustToolbox/` is *only* used in test/dev builds to satisfy assembly references without pulling in the real engine.

### Inside `SS14.Launcher/`

- **`Program.cs`** — entry point. Bootstraps Serilog, Splat (DI), `LauncherMessaging` (single-instance named-pipe IPC for `ss14://` URI handoff), then hands off to Avalonia.
- **`ConfigConstants.cs`** — central registry of URLs (auth, hub, build server, news feed, account management). Has `UrlFallbackSet` for redundancy. Edit here when changing infra endpoints.
- **`Models/`** — non-UI logic, organized by concern:
  - `Connector.cs` — orchestrates "connect to server / launch local bundle". The state machine that fetches build manifests, runs the updater, and spawns `SS14.Loader`.
  - `Updater.cs` (+ `Updater.Manifest.cs`, `Updater.Zip.cs`) — downloads and caches engine + content versions.
  - `EngineManager/` — manages installed RobustToolbox engine versions on disk.
  - `ContentManagement/` — content (game) version cache, Sqlite-backed.
  - `Data/` — user data store (favorites, logins, settings) — Sqlite via Microsoft.Data.Sqlite + Dapper.
  - `Logins/` — login token management; refresh policy lives in `ConfigConstants` (`TokenRefreshThreshold`, `TokenRefreshInterval`).
  - `OverrideAssets/` — runtime asset overrides (theming, branding) loaded from a server-distributed bundle.
  - `ServerStatus/` — polling logic for the server browser.
- **`ViewModels/`** + **`Views/`** — strict MVVM. Views are XAML, view-models are `ReactiveObject` subclasses. `ViewLocator.cs` resolves a view-model to its view by namespace convention (`*ViewModel` → `*View`). New screens go: ViewModel in `ViewModels/`, XAML pair in `Views/`.
- **`Localization/`** — Fluent (`.ftl`) translations via Linguini; weblate-managed.
- **`Api/`** — typed clients for the auth/hub/account HTTP APIs.

### SQLite and migrations

The launcher embeds three independent SQLite databases (user data, content cache, override assets), each with its own SQL migration folder embedded as resources (see `SS14.Launcher.csproj`):

- `Models/Data/Migrations/*.sql`
- `Models/ContentManagement/Migrations/*.sql`
- `Models/OverrideAssets/Migrations/*.sql`

To change a schema, **add a new numbered SQL file** in the appropriate Migrations folder — never edit existing migrations, since they've already run on users' machines. The csproj globs `*.sql` automatically.

### Dependencies worth knowing

- **Avalonia + ReactiveUI** for UI; `ReactiveUI.Fody` weaves `[Reactive]` properties.
- **`HotAvalonia`** is enabled in Debug builds — XAML hot-reload while running.
- **`Microsoft.Data.Sqlite.Core` + `Dapper`** for storage; `SQLitePCLRaw.bundle_e_sqlite3` ships the native lib (FreeBSD/`UseSystemSqlite=True` switches to the system library).
- **`NSec.Cryptography` / `SpaceWizards.Sodium` / `libsodium`** for token signing/verification.
- **`Robust.Shared.AuthLib`** — shared auth-token format with the engine.
- **`HappyEyeballsHttp.cs`** — custom dual-stack IPv4/IPv6 HTTP connection logic (do not replace with default `HttpClient` behavior; mirrored from RobustToolbox for parity).

## Conventions

- **Nullable reference types are on everywhere.** Don't suppress with `!` casually; honor the annotations.
- **`.editorconfig`** is minimal — primarily 4-space indent.
- **Versioning.** Bump `<Version>` in `Launcher.props` and `CurrentLauncherVersion` in `ConfigConstants.cs` together. The note in `Launcher.props` lists the other places that must update for a TFM change.
- **Releases** are cut by pushing a `v*` tag, which triggers `.github/workflows/publish-release.yml` to run `publish.py` and upload zips per platform.
- **Sandbox / trim warnings.** `RobustILLink` is enabled for full-release publishes; new reflection or dynamic code paths may need linker hints in `RobustLinkRoots` / `RobustLinkAssemblies` (see csproj).
