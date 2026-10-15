# Install and connect jhrome

Read this only when `jhrome` is missing, cannot launch, or cannot connect. For browsing, go back to [SKILL.md](../SKILL.md).

## Is it installed?

```bash
command -v jhrome || ls /usr/local/bin/jhrome ~/.local/bin/jhrome     # macOS
jhrome version
jhrome cli status
jhrome skills status
```

- macOS: `jhrome cli install` links `/usr/local/bin/jhrome` (or `~/.local/bin/jhrome` when `/usr/local/bin` is missing) to the CLI inside `Jhrome.app`, asking for an administrator when the folder is not writable. Older installers wrote `~/.local/bin/jhrome`; `jhrome cli install` replaces it. If `cli status` reports `pathConfigured: false`, add that folder to PATH and retry.
- Windows: the installer adds `%LOCALAPPDATA%\Programs\Jhrome` (which holds `jhrome.cmd`) to the user PATH. Open a new shell after installing.
- Skill: `jhrome skills install` copies this skill from `2ndlifeinc/jhrome-browser-public` into `~/.agents/skills/jhrome` and links it for the coding agents found (Claude Code, Codex, Pi) with the `skills` CLI; `jhrome skills update` fetches the latest published one. `jhrome skills get [install|tools|connectors]` prints the copy that matches the installed CLI when the two differ (`skills status` reports `mismatch`).

## Install

Builds are published at https://github.com/2ndlifeinc/jhrome-browser-public (releases `latest-darwin-arm64`, `latest-win32-x64`). Each holds `update-manifest.json` with the asset URL and its sha256. Install only with the user's agreement; the download is about 300 MB.

macOS (Apple silicon and Intel):

```bash
work=$(mktemp -d)
curl -fsSL https://github.com/2ndlifeinc/jhrome-browser-public/releases/download/latest-darwin-arm64/update-manifest.json -o "$work/manifest.json"
read -r url sha < <(python3 -c 'import json,sys; a=next(a for a in json.load(open(sys.argv[1]))["assets"] if a["name"].endswith(".dmg")); print(a["url"], a["sha256"])' "$work/manifest.json")
curl -fL "$url" -o "$work/jhrome.dmg"
echo "$sha  $work/jhrome.dmg" | shasum -a 256 -c
mkdir "$work/mount"
hdiutil attach -nobrowse -readonly -mountpoint "$work/mount" "$work/jhrome.dmg"
ditto --noextattr --norsrc "$work/mount/Jhrome.app" /Applications/Jhrome.app
hdiutil detach "$work/mount"
open /Applications/Jhrome.app
"/Applications/Jhrome.app/Contents/Frameworks/Jhrome Framework.framework/Versions/Current/Resources/jhrome/cli/bin/jhrome" cli install
```

This installs `/Applications/Jhrome.app`; the CLI, skill, and runtime are inside it, so an app update updates them too. An older install in `~/Applications/jhrome/` moves onto the app by itself at its next update and is removed once it quits.

Windows: download the `.exe` named in the `latest-win32-x64` manifest and run it. It installs into `%LOCALAPPDATA%\Programs\Jhrome`, adds a desktop shortcut, `jhrome.cmd` and an entry in Settings > Apps. Then run `jhrome skills install` for the skill.

Update an installed build: `jhrome upgrade` checks, `jhrome upgrade --apply` installs the newer build.

Remove jhrome: `jhrome uninstall` lists what it would remove; `jhrome uninstall --yes` closes jhrome and removes its command and skill. On macOS it moves the app to the Trash, on Windows it runs `Uninstall Jhrome.exe` silently (also in Settings > Apps). Add `--purge-data` to also remove the profiles and settings, only when the user asks for it.

Site cache (sentences of well-known sites that replay without a ref): `npx skills add 2ndlifeinc/jhrome-site-cache-public -g -y`. Run it again to update; jhrome picks it up without a restart.

## Connect

The CLI finds a running jhrome through `control-endpoint.json` in the data dir (macOS `~/Library/Application Support/Jhrome/data`, Windows `%LOCALAPPDATA%\Jhrome\data`, or `--data-dir`). When none is running it launches the installed release.

| Message | Meaning and fix |
|---|---|
| `jhrome is not running and no installed release could be launched. Start jhrome first.` | No release next to the CLI. Install, or start jhrome (open `Jhrome.app`, the Windows desktop shortcut) and retry |
| `브라우저를 준비하고 있습니다.` / `Browser is starting.` | The browser is still starting. Retry after a few seconds |
| `jhrome is already running. Return to its existing window.` | Another jhrome already runs on this data dir. Use it; do not start a second one |
| Exit code 3 | The control server refused the CLI (401/403). Quit jhrome from its window, then retry |

`--data-dir` never falls back to the default instance: with no instance running on that data dir, the CLI launches one for it.

A call that timed out, failed or was slow: read `logs/runtime.<date>.<n>.log` in the data dir. Each JSON line is one tool call with `waitMs` (time queued behind earlier calls) and `ms` (its own run); a timed-out call also lists every tab with `responsive` and the free memory. `jhrome status` lists `browser.warnings`; one about cookies not being saved means sign-ins will not survive a restart, and it says how to fix it. Browser warnings and errors are in `logs/chrome.log`, and those of the session before the last restart in `logs/chrome.previous.log`.

Then go back to the original task in [SKILL.md](../SKILL.md).
