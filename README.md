# Hayase — Flatpak packaging

Flatpak packaging for [Hayase](https://hayase.watch) (formerly Miru), an anime
torrent-streaming desktop app. App ID: `watch.hayase.Hayase`.

This repository is **packaging only** — it is not the application's source code.
It builds Hayase from upstream source ([`hayase-app/electron`](https://github.com/hayase-app/electron))
and runs it on upstream's custom Electron runtime, producing an installable
Flatpak.

## Requirements

- `flatpak` and `flatpak-builder`
  - Fedora: `sudo dnf install flatpak flatpak-builder`
  - Debian/Ubuntu: `sudo apt install flatpak flatpak-builder`
  - Arch: `sudo pacman -S flatpak flatpak-builder`
- The Flathub remote, for the runtime and build dependencies:
  ```sh
  flatpak remote-add --if-not-exists --user flathub https://flathub.org/repo/flathub.flatpakrepo
  ```
- **x86_64** — the custom Electron runtime is extracted from an x86_64-only
  release artifact, so this package is x86_64-only (`flathub.json`).
- **A network connection during the build.** Node dependencies are vendored
  offline (`generated-sources.json`), but flatpak-builder still downloads the
  app source, the pnpm CLI, and the Electron runtime while building.

## Build and install

From the directory containing these files:

```sh
flatpak-builder --user --force-clean --install --install-deps-from=flathub \
  --repo=repo builddir watch.hayase.Hayase.yml
```

`--install-deps-from=flathub` pulls the required runtime, SDK, Node SDK
extension, and Electron base app automatically. `--install` installs the result
into your user Flatpak installation.

Then run it:

```sh
flatpak run watch.hayase.Hayase
```

To uninstall:

```sh
flatpak uninstall --user watch.hayase.Hayase
```

## Repository contents

| File | Purpose |
|---|---|
| `watch.hayase.Hayase.yml` | The flatpak-builder manifest (the entry point). |
| `generated-sources.json` | Offline npm/pnpm dependency graph, so the sandboxed build needs no npm network access. Generated with `flatpak-node-generator` (pnpm backend). |
| `flatpak-updater.patch` | Replaces `electron-updater` inside Flatpak (`FLATPAK_ID` set) with Flathub AppStream API + `/app/.updated` checks, `appstream://` software-center updates, and clean relaunch via `flatpak-spawn --latest-version` ([`hayase-app/electron#8`](https://github.com/hayase-app/electron/pull/8)). |
| `watch.hayase.Hayase.desktop` | Desktop entry. |
| `watch.hayase.Hayase.metainfo.xml` | AppStream metadata. |
| `flathub.json` | Flathub build config (restricts to x86_64). Not read by a local build. |

All of these must sit in the same directory as the manifest for the build to
succeed.

## How the build works

1. **App from source** — clones `hayase-app/electron` at a pinned commit,
   applies the updater patch, installs dependencies offline with pnpm, and runs
   `electron-builder`. Main-process chunks are compiled to V8 bytecode exactly
   as in upstream's official builds (a build-only copy of the custom runtime,
   with its loader/codegen fuses relaxed, serves as the bytecode compiler, since
   V8 only accepts bytecode from the exact runtime build that loads it).
2. **Custom Electron runtime** — upstream's custom Electron (which adds HEVC /
   AC3 / E-AC3 codec support and extra security fuses) is not published as a
   standalone download, but the same runtime is embedded in the official `.deb`
   release. The build downloads that checksum-pinned `.deb`, extracts the
   runtime, and hands it to `electron-builder` as its `electronDist`. The app is
   still built entirely from source.
3. **Offline dependencies** — `generated-sources.json` populates the pnpm store
   so no packages are fetched from npm at build time.

## Updating to a new Hayase release

In `watch.hayase.Hayase.yml`:

1. Update the `git` source `commit:` to the new release's commit (upstream no
   longer tags releases; use the commit whose `package.json` version matches
   `https://api.hayase.watch/files/latest-linux.yml`).
2. Update the `.deb` source `url:` and `sha256:` to the matching version.
3. Regenerate `generated-sources.json` from the new `pnpm-lock.yaml`
   (`flatpak-node-generator pnpm`).
4. Confirm `flatpak-updater.patch` still applies against the new commit;
   regenerate it if not. If the Electron version changed, update the
   `electron-v<ver>-linux-x64` names in the manifest.
5. Add a `<release>` entry to `watch.hayase.Hayase.metainfo.xml`.

## Notes

- **Sandbox permissions** are declared in the manifest's `finish-args`: network,
  audio/video, GPU, and read/write access to the common download locations
  (Downloads, Desktop, Documents, Videos). The torrent download folder is
  configurable in-app within those locations.
- **Torrent scratch data** is written to host `/tmp/webtorrent`
  (`--filesystem=/tmp/webtorrent:create`), matching the AppImage and `.deb`
  releases while keeping the rest of `/tmp` isolated inside the sandbox.
- **Flathub submission:** the linter warns about host-command access via
  `flatpak-spawn` (used to launch external players on the host). That warning
  is expected and not a build error; a Flathub reviewer has to accept it.

## Links

- Homepage: <https://hayase.watch>
- Upstream app source: <https://github.com/hayase-app/electron>
- License: BUSL-1.1 (converts to GPL-3.0 on 2029-04-01)
