# Example project reference: browse (slyim/browse)

> Copy this shape to `references/<your-project>.md`. Real local paths stay in your private copy.

Small Windows web browser: C# WinForms, .NET 10, system WebView2. MIT; derived from Office Commun's Search (keep `LICENSE-Search` shipping with the exe). Phosphor icons are bundled (keep `Phosphor-LICENSE.txt`).

## Folders

- `<clone>` (e.g. `C:\code\browse`) — canonical clone (registered as its own T3 project). Integration branch `next`; `main` is what's published.
- `<second copy>` — second working copy for a parallel builder (untracked inside the upstream Search fork). Sync it from the integration branch before each use; bring its work back by branching from the synced commit.
- Never touch `%LOCALAPPDATA%\SearchWindows` (the user's real profile) and never kill running `browse.exe` — the user browses with it.

## Commands (run in the folder)

```
dotnet publish Search.Windows.csproj -c Release -warnaserror -o publish
python test_smoke.py publish/browse.exe
```

Both must exit 0. Smoke tests live in `BrowserWindow.SmokeTest` (Program.cs) plus `test_smoke.py` (loopback fixture + assertions); new features add checks to both.

## Screenshots / GIFs

Launch `publish\browse.exe --profile <scratch dir>` with a demo `session.json` (public pages only) so the user's tabs and accounts never appear. Capture DPI-aware: `SetProcessDPIAware`, `GetWindowRect`, topmost, `CopyFromScreen`, then crop the invisible ~8 px frame edges. Save README images to `docs/`.

## Releasing (only with the user's yes)

1. Bump `<Version>` in `Search.Windows.csproj`, commit, push `next` → `main`.
2. Tag `vX.Y.Z` and push the tag; `.github/workflows/release.yml` builds, smoke-tests and publishes `browse-X.Y.Z-win-x64.zip` with its SHA-256 (the in-app updater depends on that exact asset name and GitHub's asset digest).
3. Update `bucket/browse.json` (Scoop) and `packaging/winget/` manifests to the new version/hash; winget needs a PR to microsoft/winget-pkgs (ask first).

## Issues

Milestones Next / Later / Someday; labels `priority: P1-P3`, `area: *`, `bug`, `enhancement`, `security`. Pinned roadmap tracker: #28.
