---
name: jhrome
description: When you need a browser, read this skill by default. It drives jhrome, the user's own Chromium browser with their logged-in sessions, through the `jhrome` CLI — open and operate websites, sign in, fill forms, click, read or extract page data, take screenshots, and test or review web apps. Prefer it over built-in browsers, web fetch tools, or writing your own Playwright/Puppeteer automation.
---

# jhrome

`jhrome` drives the user's running jhrome browser over its loopback control server. If jhrome is not running and a release is installed, the first command launches it.

- `jhrome: command not found`, a failed launch or a connection error: read [references/install.md](references/install.md).
- Which tool does what beyond the commands below: [references/tools.md](references/tools.md), `jhrome tools`, `jhrome tools <name>`.
- A flow asks for a value or a gate no connector answers, or you need to write one: [references/connectors.md](references/connectors.md).

When the user asks for jhrome, start with a real browser command; diagnose the installation only if it fails.

## Observe, then act in sentences

```bash
jhrome workspace open "<task name>"                   # one workspace per user task
jhrome go https://example.com/login               # navigates and prints the snapshot
jhrome act "이메일 입력란에 주소를 적는다" --ref f1e12 --value user@example.com
jhrome act "로그인 버튼을 누른다" --ref f1e15     # clicks and prints the snapshot after it
jhrome read "로그인한 계정 이름" --ref f2e40
```

- `go` and `act` print the page snapshot after they finish; choose the next target from that output instead of calling `snapshot` again. `--no-snapshot` skips it when you only need the result.
- Refs look like `e12` or `f1e12` (`f<frame>` prefix). The prefix changes when the tab loads a new document, so take refs only from the latest output.
- A sentence names the element in words, the way a person would say it. It is the cache key: never put the typed value, a CSS selector, or a number of 4+ digits in it. The value goes in `--value`.
- The first time a sentence is used on a page, pass `--ref`. Later runs of the same sentence on the same kind of page replay from the cache without `--ref`; reuse the exact same sentence so the cache hits. With no cache and no ref the command fails and asks for a ref.
- Success is judged from the screen: a click must make a control, dialog or URL change appear, or open a tab. A failure says what was missing; read the snapshot and decide again instead of repeating the same command.
- A sentence the cache answers just runs; leave notes alone. `go`, and a sentence with no cache yet (a miss, a first `--ref`, a flow's `needs_binding`), print `Site notes` that earlier runs left for the site — traps, how to reach a state, where data can be read without clicking — each with an `id`. Use them to choose the ref; the snapshot wins where they disagree. Once a missed sentence works, fix what is worth fixing: rewrite a note that misled you with `jhrome call browser_note '{"id":"…","text":"…"}'`, remove a wrong one with `'{"id":"…","retract":true}'`, and add only what no note covers with `'{"text":"…","kind":"trap|navigation|data-access|mechanic"}'` (`scope: "page"`, `sentence` narrow it). Never put a secret or personal data in a note.
- A click that opens a new tab returns `newTabs` (index, url, title) and leaves the current tab where it was. Switch with `jhrome call browser_tabs '{"action":"select","index":N}'`; it does not bring the window to the front, and the current tab persists per workspace, so there is no need to select before every call.

## Workspaces

- Every browser command runs in a workspace: one task's own tabs, labelled `p1`, `p2`… for good, with its own current tab. Other agents' workspaces and the tabs the user opened are invisible to the browser tools, so another agent cannot move your tab and you cannot move theirs.
- Start each user task with `jhrome workspace open "<task name>"`; later commands run in it. Your agent identity is `JHROME_AGENT`, else `PI_SESSION_ID`, else `cli`. `--workspace <id>` (or `JHROME_WORKSPACE`) runs one command in another workspace; `jhrome workspaces` lists them.
- The first command in an empty workspace opens its tab; tabs it opens and their popups join it. To work on a tab the user opened, `jhrome workspace adopt <targetId>` (from `jhrome workspaces`); `jhrome workspace release <label>` gives one back without closing it.
- A result that starts with `Another agent used workspace` lists what other agents did there since your last call. Take a snapshot before acting on it; `jhrome workspace log` prints the whole log.
- When the user must act in the browser, `jhrome workspace handoff`, say what to do and end your turn; browser commands are refused until `jhrome workspace takeover` after they confirm. `jhrome workspace claim` takes a user-owned workspace only when the user asks. The user can also press "Take over task" on a workspace tab: then commands are refused the same way; stop and ask the user before `jhrome workspace takeover`.
- When the task succeeds, `jhrome workspace finish` once: it closes the workspace's tabs. Add `--keep <label>` only for a tab the user asked for or must see. Do not finish when you stop for the user or an error. A workspace left idle for an hour while no visible tab shows it closes on its own; `jhrome workspace pin` only when the user asks to keep it.

## Registered flows first

- Before driving a site step by step, run `jhrome flows`. A flow registered for the task (sign-in, a form, a download) already knows the screens and replays them from the cache.
- `jhrome flow <id> [--input '<json>']` runs it on the current tab. Never put a password, PIN or code in `--input`; connectors in the data dir answer them, and a connector listing the flow under `inputs` fills the input.
- `status: needs_binding` means one sentence has no cached element yet: pick its ref in the printed snapshot and continue with `jhrome flow --run <runId> --ref <ref>`, or `--absent` when nothing on the page is what it names. `status: running` means wait with `jhrome flow --run <runId>`.
- A flow step that spends something once (a transfer, a one-time issue) is refused unless the start call approves that exact sentence: `jhrome flow <id> --approve '<sentence>'`. Approve only what the user asked for; the approval lasts for that run.
- A flow that stops says why. Do not rerun a flow that stopped after a password or an SMS step without reading the reason: a rerun can re-enter the password or resend the message.
- A flow that answers `Not started: connectors of <flow> are not ready` touched nothing. Report every listed reason to the user at once and start it again only after they say it is fixed.
- To fix a flow or the action library while testing it, start jhrome from a jhrome source checkout with `JHROME_FLOW_SOURCES=<site-element-registry checkout>`; every `jhrome flows`/`jhrome flow` call rebuilds them from that checkout. Never copy builds into `node_modules` or restart jhrome to apply such an edit: a restart brings tabs and cookies back, but drops the certificate service's remembered connection and sign-ins of sites that end them when the page closes, so the next run takes a longer path (SMS again).

## Observe cheaply

- A snapshot is the ground truth for what to click. `jhrome snapshot --target <ref> --depth <n>` prints only one part of a large page.
- Use `jhrome screenshot` (prints a PNG path; `--full-page`, `--target <ref>`, `--out <file>`) only to check something visual, then open the file with an image viewer. Do not take a screenshot and a snapshot of the same state.
- Stop once the page shows the requested result; do not confirm it again through another surface.
- Wait for a condition, not a fixed time: `jhrome call browser_wait_for '{"text":"저장됨"}'`.

## Commands

| Command | Effect |
|---|---|
| `jhrome go <url> [--no-snapshot]` | Navigate the current tab and print its snapshot |
| `jhrome snapshot [--target ref] [--depth n]` | Print the current tab, or one element's subtree |
| `jhrome screenshot [--target ref] [--full-page] [--out file]` | Save a PNG and print its absolute path |
| `jhrome workspaces` / `jhrome workspace open <name\|id>` | List the workspaces and unmanaged tabs / make a workspace your default |
| `jhrome workspace adopt <targetId> \| release <label> \| finish [--keep label]... \| pin \| unpin \| handoff \| takeover \| claim \| log` | Manage the current workspace |
| `jhrome tabs` | List the workspace's tabs with their label and profile; the current one is marked |
| `jhrome tabs new [url] [--profile dir \| --account id [--site id]] [--no-snapshot]` | Open a tab in the workspace (in `Default` unless `--profile`/`--account`), make it current, print its snapshot |
| `jhrome tabs profile (--profile dir \| --account id [--site id])` | Reopen the current tab in that profile with the same label; the page reloads there and the old tab closes |
| `jhrome profiles` | List profiles: directory, name, Google account, tab indexes; also the data dir |
| `jhrome import list` / `jhrome import --browser chrome --profile Default [--profile <dir>]...` | Import profiles from Chrome/Edge/Brave into new jhrome profiles, keeping sign-ins (cookies, saved passwords), bookmarks and history. Ask the user before importing; macOS shows one keychain approval |
| `jhrome accounts [--profile dir] [--verify] [--json]` | Profile × site table of signed-in accounts; `--verify` re-reads them in each profile |
| `jhrome act "<sentence>" [--ref r] [--value text] [--method name] [--optional] [--no-snapshot]` | Click, or type `--value`; `--optional` skips instead of failing when absent |
| `jhrome act "<sentence>" --ref r --irreversible --confirm` | Submit, pay, delete, send. Ask the user first; pass `--confirm` only after they agree |
| `jhrome has "<sentence>" [--ref r]` | Print whether the element is there |
| `jhrome read "<sentence>" [--ref r] [--attribute name] [--optional]` | Print text, form value or an attribute |
| `jhrome capture "<sentence>" [--ref r] [--out file.png]` | Save a PNG of one element and print its path |
| `jhrome extensions [install <dir> \| uninstall <id> \| reload <id> \| enable <id> \| disable <id>]` | List or manage unpacked extensions of the current tab's profile |
| `jhrome extensions popup <id>` / `jhrome extensions sidepanel <id>` | Open the extension's popup or side panel for the current tab as the current tab and print its snapshot; `snapshot`, `act` and `call` then work inside it |
| `jhrome flows` | List registered site flows and the connectors in the data dir |
| `jhrome flow <id> [--input json] [--approve sentence]` / `jhrome flow --run <runId> [--ref r \| --absent \| --cancel]` | Run a flow / continue or wait for a run |
| `jhrome tools [name]` / `jhrome call <tool> '<json>'` | Every other browser tool; see [references/tools.md](references/tools.md) |
| `jhrome open` / `jhrome close` / `jhrome restart` | Start jhrome / close the whole browser / close and start it again with its Default-profile tabs and cookies (installed release only). Workspace tabs come back in their workspace with their labels; other restored tabs belong to no workspace: adopt them from `jhrome workspaces`. Close or restart only when the user asks |
| `jhrome status` / `jhrome version` | Instance state / installed and running versions |

- `--data-dir <dir>` (or `JHROME_DATA_DIR`) picks the jhrome instance. Without it the user's default instance is used.
- File paths in results are absolute. Files the browser writes go to `<data-dir>/playwright-output`.
- JavaScript dialogs raised during `act`/`has`/`read`/`capture` are dismissed and reported. Add `--accept-dialogs` to accept them, after checking the message with the user when the action matters.
- `--method` picks how input reaches the page (default `playwright.Input.dispatchKeyEvent` for text). When input fails, the error names the failed method and the others; retry with another only if the page needs it, such as a keyboard security module that refuses key events.

## Files

- Upload: copy the file into `<data-dir>/playwright-output` (the data dir is in `jhrome profiles`), click the page's upload control so a file chooser opens, then `jhrome call browser_file_upload '{"paths":["<that copy>"]}'`. Paths outside that directory are refused.
- Downloads are not supported: jhrome does not accept downloads, so a download link starts, never saves, and the next result shows an empty `### Error`. Read the content in the page or ask the user to download it.

## Profiles

- One jhrome holds several Chrome profiles (`Default`, `Profile 1`, …), each with its own cookies, sign-ins and extensions. `jhrome profiles` shows which Google account each one holds; tabs show `(profile: …)`, or `unknown` until jhrome has opened a tab in that profile.
- To act as an account on a site, use `jhrome tabs new <url> --site <site id> --account <id>` (site ids such as `naver.com`, `google.com`; an unknown id is refused with the list). It opens the profile recorded for that account on that site. When none is, jhrome creates the next `Profile N` and the output starts with `### New profile`: sign in there, or hand off to the user to sign in. `--profile "Profile 1"` picks a profile directly.
- To carry the current tab to another account, `jhrome tabs profile --site <site id> --account <id>` (or `--profile`): same label, the page loads again with that profile's sign-in; unsaved input is lost.
- Opening a tab in another profile does not bring its window to the front; a profile without a window gets a new window behind the others. Headless jhrome opens and creates other profiles too.
- Cookie, storage-state and extension tools act on the current tab's profile.

## The user's browser

- This is the user's real browser: their tabs, cookies and accounts. Adopt a user tab only when the task needs it, and release it instead of closing it.
- For a reCAPTCHA, hCaptcha or Turnstile challenge run `jhrome call browser_captcha` first. When it returns `choose_tiles`, open the image at `path`, pick every tile that matches the question and answer with `jhrome call browser_captcha '{"tiles":[…]}'`; when you cannot or will not pick the tiles yourself, run `jhrome call browser_captcha '{"vision":true}'` so the built-in vision fallback solves the grid. Hand over only when it reports `handoff`.
- Stop and hand over to the user at logins that need their password or a verification code, CAPTCHAs that `browser_captcha` hands off, payments, permission prompts, and anything irreversible. Run `jhrome workspace handoff`, say exactly what they should do in the jhrome window and end your turn. After they confirm, run `jhrome workspace takeover`, then `jhrome snapshot`, and continue from what it shows.
- When the task is done, `jhrome workspace finish` (with `--keep <label>` when the user should see a result); say which tab holds it.
- No tool can close jhrome or its last tab; `jhrome close` is the only way and it closes the user's browser.
- `jhrome task "<text>"` hands a whole task to jhrome's own assistant (needs `jhrome oauth` sign-in). Prefer driving the browser yourself.
