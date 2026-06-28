# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Build & Run

**Prerequisites:** .NET 10 SDK and ffmpeg (jellyfin-ffmpeg)

```bash
# Build the solution
dotnet build

# Run the server (requires web client files)
dotnet run --project Jellyfin.Server --webdir /absolute/path/to/jellyfin-web/dist

# Run without web client
dotnet run --project Jellyfin.Server -- --nowebclient
```

Server runs at `http://localhost:8096` by default. API docs at `/api-docs/swagger/index.html`.

## Tests

```bash
# Run all tests
dotnet test Jellyfin.sln --configuration Release

# Run a specific test project
dotnet test tests/Jellyfin.Naming.Tests

# Run a single test (by name filter)
dotnet test tests/Jellyfin.Naming.Tests --filter "FullyQualifiedName~TestMethodName"
```

Test framework: xunit v3 with Moq and AutoFixture. Integration tests (`Jellyfin.Server.Integration.Tests`) spin up an in-process server; they run sequentially (no parallel execution).

## Code Style & Linting

```bash
# Check formatting (what CI runs)
dotnet format --verify-no-changes --verbosity minimal

# Auto-fix formatting
dotnet format
```

StyleCop and Roslyn analyzers run in Debug builds. Warnings are errors in all builds (except `NU1902`, `NU1903`). In `Debug` configuration, `AnalysisMode` is `AllEnabledByDefault`.

**Banned APIs** (see `BannedSymbols.txt`): `Task<T>.Result` (use `await`), `Guid` equality operators (use `.Equals()`).

## Architecture

Jellyfin is an ASP.NET Core media server. The solution follows a layered architecture:

### Core Abstraction Layers (no implementation)
- **`MediaBrowser.Model`** — DTOs, enums, and configuration models shared across boundaries
- **`MediaBrowser.Common`** — Shared abstractions: `IApplicationHost`, plugin system (`IPlugin`, `BasePlugin<T>`), and infrastructure interfaces
- **`MediaBrowser.Controller`** — Domain interfaces: `ILibraryManager`, `IUserManager`, `IMediaEncoder`, `ILiveTvManager`, etc. Entity types (`BaseItem`, `Movie`, `Series`, `Episode`, `Season`, `Folder`) live here
- **`Emby.Naming`** — File-path parsing and media naming rules (regex-based resolution)

### Implementation Layer
- **`Emby.Server.Implementations`** — Core implementations: `LibraryManager` (scan/resolve/index), `UserDataManager`, media resolvers (in `Library/Resolvers/`), and entry points
- **`Jellyfin.Server.Implementations`** — User management, auth, device tracking, display preferences
- **`Jellyfin.Server.Implementations/Users/UserManager.cs`** — User CRUD and authentication
- **`MediaBrowser.Providers`** — Metadata providers (TMDB, MusicBrainz, OMDB, AudioDb, ListenBrainz) and manager that orchestrates them
- **`MediaBrowser.MediaEncoding`** — FFmpeg encoding, probing (`MediaInfo`), subtitle handling, transcoding

### API Layer
- **`Jellyfin.Api`** — ASP.NET Core controllers (one per domain area), `BaseJellyfinApiController.cs` base class, auth handlers, middleware, and model binders. All controllers inherit from the base class.

### Server Entry Point
- **`Jellyfin.Server`** — Kestrel host, `Program.cs` bootstraps the `CoreAppHost`, `Startup.cs` configures the DI container and middleware pipeline. Handles migrations, setup wizard, and Prometheus metrics.

### Supporting Projects (`src/`)
- **`Jellyfin.Database`** — EF Core `JellyfinDbContext`, entity models, SQLite provider (`Jellyfin.Database.Providers.Sqlite`), and EF migrations
- **`Jellyfin.Drawing` / `Jellyfin.Drawing.Skia`** — Image processing with SkiaSharp
- **`Jellyfin.Extensions`** — Extension methods (string, collection, etc.)
- **`Jellyfin.Networking`** — Network interface discovery, Happy Eyeballs HTTP handler
- **`Jellyfin.LiveTv`** — Live TV recording, guide management, tuner host abstractions
- **`Jellyfin.MediaEncoding.Hls`** — HLS streaming pipeline
- **`Jellyfin.MediaEncoding.Keyframes`** — Keyframe extraction for trickplay

### Plugin System
Plugins implement `IPlugin` / `BasePlugin<TConfig>` and register services via `IPluginServiceRegistrator`. They are discovered from the plugin directory at startup.

### Media Library Pipeline
File discovery → `ILibraryManager` → path-based `Resolvers` (in `Emby.Server.Implementations/Library/Resolvers/`) → `BaseItem` entities → metadata refresh via `MediaBrowser.Providers` → stored via EF Core to SQLite.

## Key Conventions

- All C# files use XML doc comments on public members (required by `GenerateDocumentationFile`)
- Nullable reference types are enabled project-wide
- Use `await` instead of `.Result` on tasks (banned symbol)
- Indentation: 4 spaces for C#, 2 spaces for XML/YAML
- LF line endings, newline at end of file
- Imports: system namespaces sorted first (`dotnet_sort_system_directives_first`)
