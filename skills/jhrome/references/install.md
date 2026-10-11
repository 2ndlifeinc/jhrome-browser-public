# Install and connect jhrome

Read this only when `jhrome` is missing, cannot launch, or cannot connect. For browsing, go back to [SKILL.md](../SKILL.md).

## Is it installed?

```bash
command -v jhrome || ls /usr/local/bin/jhrome ~/.local/bin/jhrome     # macOS
jhrome version
jhrome cli status
jhrome skills status
```

- macOS: `jhrome cli install` links `/usr/local/bin/jhrome` (or `~/.local/bin/jhrome` when `/usr/local/bin` is missing) to the CLI inside the installed release, asking for an administrator when the folder is not writable. Older installers wrote `~/.local/bin/jhrome`; `jhrome cli install` replaces it. If `cli status` reports `pathConfigured: false`, add that folder to PATH and retry.
- Windows: the installer adds `%LOCALAPPDATA%\Programs\jhrome` (which holds `jhrome.cmd`) to the user PATH. Open a new shell after installing.
- Skill: `jhrome skills install` copies this skill from `2ndlifeinc/jhrome-browser-public` into `~/.agents/skills/jhrome` and links it for the coding agents found (Claude Code, Codex, Pi) with the `skills` CLI; `jhrome skills update` fetches the latest published one. `jhrome skills get [install|tools|connectors]` prints the copy that matches the installed CLI when the two differ (`skills status` reports `mismatch`).

## Install

Builds are published at https://github.com/2ndlifeinc/jhrome-browser-public (releases `latest-darwin-arm64`, `latest-win32-x64`). Each holds `update-manifest.json` with the asset URL and its sha256. Install only with the user's agreement; the download is about 300 MB.

macOS (Apple silicon):

```bash
work=$(mktemp -d)
curl -fsSL https://github.com/2ndlifeinc/jhrome-browser-public/releases/download/latest-darwin-arm64/update-manifest.json -o "$work/manifest.json"
url=$(python3 -c 'import json,sys; print(json.load(open(sys.argv[1]))["assets"][0]["url"])' "$work/manifest.json")
sha=$(python3 -c 'import json,sys; print(json.load(open(sys.argv[1]))["assets"][0]["sha256"])' "$work/manifest.json")
curl -fL "$url" -o "$work/jhrome.zip"
echo "$sha  $work/jhrome.zip" | shasum -a 256 -c
ditto -x -k "$work/jhrome.zip" "$work/release"
bash "$work/release/scripts/macos-install.sh" "$work/release"
```

This installs into `~/Applications/jhrome/<version>/` (`current` points at it), writes `~/.local/bin/jhrome`, `~/Applications/jhrome/Start jhrome.command`, and links the skill.

Windows: download the `.exe` named in the `latest-win32-x64` manifest and run it. It installs into `%LOCALAPPDATA%\Programs\jhrome`, adds a desktop shortcut and `jhrome.cmd`, and links the skill.

Update an installed build: `jhrome upgrade` checks, `jhrome upgrade --apply` installs the newer build.

Remove jhrome (macOS): `jhrome uninstall` lists what it would remove; `jhrome uninstall --yes` closes jhrome, removes its command and skill, and moves the app to the Trash. Add `--purge-data` to also trash the profiles and settings, only when the user asks for it.

Site cache (sentences of well-known sites that replay without a ref): `npx skills add 2ndlifeinc/jhrome-site-cache-public -g -y`. Run it again to update; jhrome picks it up without a restart.

## Connect

The CLI finds a running jhrome through `control-endpoint.json` in the data dir (macOS `~/Library/Application Support/jhrome/data`, Windows `%LOCALAPPDATA%\jhrome\data`, or `--data-dir`). When none is running it launches the installed release.

| Message | Meaning and fix |
|---|---|
| `jhrome is not running and no installed release could be launched. Start jhrome first.` | No release next to the CLI. Install, or start jhrome (`Start jhrome.command`, the Windows desktop shortcut) and retry |
| `브라우저를 준비하고 있습니다.` / `Browser is starting.` | The browser is still starting. Retry after a few seconds |
| `jhrome is already running. Return to its existing window.` | Another jhrome already runs on this data dir. Use it; do not start a second one |
| Exit code 3 | The control server refused the CLI (401/403). Quit jhrome from its window, then retry |

`--data-dir` never falls back to the default instance: with no instance running on that data dir, the CLI launches one for it.

Then go back to the original task in [SKILL.md](../SKILL.md).
