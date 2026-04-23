# nuget-maui-windows

## Feature exercised

This probe exercises Mend SCA detection of a .NET 8 MAUI project that targets the Windows 10 SDK-versioned TFM (`net8.0-windows10.0.19041.0`), using standard NuGet `PackageReference` with a `packages.lock.json` for deterministic lockfile-driven resolution.

## Probe metadata

| Field          | Value                                              |
|----------------|----------------------------------------------------|
| pattern        | nuget-maui-windows                                 |
| target         | local                                              |
| package manager| NuGet (PackageReference)                           |
| target framework | net8.0-windows10.0.19041.0                       |
| lock file      | packages.lock.json (RestorePackagesWithLockFile=true) |
| seed           | nuget-maui-windows                                 |

## Purpose

Validates that Mend correctly handles:
1. The long platform-versioned TFM `net8.0-windows10.0.19041.0` — Mend must not truncate or misparse the OS version suffix when grouping dependencies under a target framework bucket.
2. MAUI's deep transitive closure (`Microsoft.Maui.Controls` pulls `Microsoft.Maui.Controls.Core`, `Microsoft.Maui.Controls.Xaml`, `Microsoft.Maui.Core`, `Microsoft.Maui.Essentials`, `Microsoft.Maui.Graphics`) resolved via lockfile.
3. `CommunityToolkit.Maui` 7.0.1 resolved alongside MAUI itself — both share the same `Microsoft.Maui.*` transitive subtree without duplication.
4. `Microsoft.Extensions.Logging` declared directly even though it is also a MAUI transitive dep — Mend must report it once at the resolved version (8.0.0).
5. `Newtonsoft.Json` 13.0.3 as a leaf dependency with no Windows-specific transitives, confirming that non-platform packages resolve cleanly inside a platform-targeted project.

## Expected dependency tree

### Direct dependencies (net8.0-windows10.0.19041.0)

| Package | Requested | Resolved |
|---------|-----------|----------|
| `Microsoft.Maui.Controls` | 8.0.82 | 8.0.82 |
| `CommunityToolkit.Maui` | 7.0.1 | 7.0.1 |
| `Microsoft.Extensions.Logging` | 8.0.0 | 8.0.0 |
| `Newtonsoft.Json` | 13.0.3 | 13.0.3 |

### Key transitive relationships

- `Microsoft.Maui.Controls` 8.0.82
  - `Microsoft.Maui.Controls.Core` 8.0.82
    - `Microsoft.Maui.Core` 8.0.82
      - `Microsoft.Maui.Essentials` 8.0.82
      - `Microsoft.Maui.Graphics` 8.0.82
      - `Microsoft.Extensions.DependencyInjection` 8.0.0
      - `Microsoft.Extensions.Logging.Abstractions` 8.0.0
    - `Microsoft.Maui.Essentials` 8.0.82 (shared)
    - `Microsoft.Maui.Graphics` 8.0.82 (shared)
  - `Microsoft.Maui.Controls.Xaml` 8.0.82
    - `Microsoft.Maui.Controls.Core` 8.0.82 (shared)
  - `Microsoft.Maui.Core` 8.0.82 (shared)
- `CommunityToolkit.Maui` 7.0.1
  - `CommunityToolkit.Maui.Core` 7.0.1
    - `Microsoft.Maui.Controls` 8.0.82 (shared)
    - `Microsoft.Maui.Core` 8.0.82 (shared)
- `Microsoft.Extensions.Logging` 8.0.0
  - `Microsoft.Extensions.DependencyInjection.Abstractions` 8.0.0
  - `Microsoft.Extensions.Logging.Abstractions` 8.0.0 (shared)
- `Newtonsoft.Json` 13.0.3 (leaf — no children)

### Mend-specific assertions

- All packages must appear under the `net8.0-windows10.0.19041.0` TFM bucket — not under `net8.0` or any truncated variant.
- `dependencyFile` for every entry should reference `MauiWindowsProbe.csproj` or `packages.lock.json`.
- `dependencyType` for direct deps: `DIRECT`; for transitives: `TRANSITIVE`.
- `Newtonsoft.Json` has no `children` array (leaf node).
- `Microsoft.Extensions.Logging.Abstractions` must appear once even though both `Microsoft.Maui.Core` and `Microsoft.Extensions.Logging` pull it transitively.
