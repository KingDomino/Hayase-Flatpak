# Hayase-Flatpak

Flatpak packaging for [Hayase](https://hayase.watch) (formerly Miru), an anime
torrent-streaming desktop app. App ID: `watch.hayase.Hayase`.

This repository is **packaging only,** it is not the application's source code, nor does it contain the source code.
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
- **x86_64.** The custom Electron runtime is extracted from an x86_64-only
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
| `disable-updater.patch` | Disables the app's built-in self-updater (Flatpak delivers updates). |
| `flatpak-plugins-dir.patch` | Stores user-installed plugins under the writable user-data dir instead of the read-only `/app`. |
| `flatpak-external-player.patch` | Launches external video players on the host via `flatpak-spawn`. |
| `watch.hayase.Hayase.desktop` | Desktop entry. |
| `watch.hayase.Hayase.metainfo.xml` | AppStream metadata. |
| `flathub.json` | Flathub build config (restricts to x86_64). Not read by a local build. |

All of these must sit in the same directory as the manifest for the build to
succeed.

## Updating to a new Hayase release
Note: These updates only apply to the shell, all remote content is updated automatically.

In `watch.hayase.Hayase.yml`:

1. Update the `git` source `commit:` to the new release's commit.
2. Update the `.deb` source `url:` and `sha256:` to the matching version.
3. Regenerate `generated-sources.json` from the new `pnpm-lock.yaml`
   (`flatpak-node-generator pnpm`).
4. Confirm the three patches still apply against the new commit; regenerate any
   that don't.
5. Add a `<release>` entry to `watch.hayase.Hayase.metainfo.xml`.

## Notes

- **Sandbox permissions** are declared in the manifest's `finish-args`: network,
  audio/video, GPU, and read/write access to the common download locations
  (Downloads, Desktop, Documents, Videos). The torrent download folder is
  configurable in-app within those locations.
- **Torrent scratch data** is written to the sandbox's private `/tmp`
  (`/tmp/webtorrent`), which is a per-instance tmpfs cleared when the app fully
  quits — so it never accumulates on persistent disk.
- **Flathub submission:** two `finish-args` entries (host-command access via
  `flatpak-spawn`, and `login1` system access) require a reviewer-granted
  exception. They are expected and are not build errors.

## Links

- Homepage: <https://hayase.watch>
- Upstream app source: <https://github.com/hayase-app/electron>
- Hayase's License: BUSL-1.1 (converts to GPL-3.0 on 2029-04-01)
