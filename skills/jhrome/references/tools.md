# jhrome browser tools

`jhrome tools` lists every tool in one line; `jhrome tools <name>` prints its input schema; `jhrome call <tool> '<json>'` runs it. jhrome's own assistant uses the same tools. Back to [SKILL.md](../SKILL.md).

The Playwright tools are `patchright-core` 1.63.0 `browserTools` without `browser_close`. They name an element with `target`: a snapshot ref (`e12`, `f1e12`) or a unique selector. Results that save a file name it by absolute path under `<data-dir>/playwright-output`; `call` replaces image data with its size.

## Sentence tools (action cache)

| Tool | CLI |
|---|---|
| `browser_act` | `jhrome act` |
| `browser_has` | `jhrome has` |
| `browser_read` | `jhrome read` |
| `browser_capture` | `jhrome capture` |
| `browser_note` | `jhrome call browser_note '<json>'` |

Prefer these for clicking, typing and reading: they remember the element per sentence and check the result on screen.

## CAPTCHA

- `browser_captcha` (`jhrome call browser_captcha`): clicks the reCAPTCHA, hCaptcha or Turnstile checkbox and reports `solved` only once the response token is filled. An image challenge returns `choose_tiles` with a numbered grid PNG at `path`; answer with `{"tiles":[…]}`, or `{"tiles":[]}` to press verify or skip. Pass `{"vision":true}` when you cannot or will not pick tiles yourself: the built-in vision fallback (Antigravity CLI with `gemini-3.6-flash`, needs `agy` installed and logged in) solves the grids and returns the per-grid `grids` log. A grid that changed is shown again instead of clicked. `handoff` after three answered grids means the user takes over.

## Pages and tabs

- `browser_navigate` (`jhrome go`), `browser_navigate_back`, `browser_navigate_forward`, `browser_reload`.
- `browser_tabs`: `{"action":"list"}` (`jhrome tabs`), `{"action":"new","url":"…"}` opens and selects a tab in `Default`, `{"action":"select","index":N}`, `{"action":"close","index":N}`. The last tab cannot be closed.
- `browser_wait_for`: `{"text":"…"}`, `{"textGone":"…"}` or `{"time":seconds}`.

## Observation

- `browser_snapshot` (`jhrome snapshot`): `target`, `depth`, `boxes` (bounding boxes per element).
- `browser_find`: search the current snapshot for text or a regular expression.
- `browser_take_screenshot` (`jhrome screenshot`).
- `browser_network_requests`, `browser_network_request` (headers and body of one request).
- `browser_verify_element_visible`, `browser_verify_text_visible`, `browser_verify_list_visible`, `browser_verify_value`: assertions for checks and QA.

## Element input beyond sentences

- `browser_click`, `browser_type`, `browser_press_sequentially`, `browser_press_key`, `browser_keydown`, `browser_keyup`, `browser_hover`, `browser_drag`, `browser_select_option`, `browser_check`, `browser_uncheck`, `browser_fill_form`.
- Coordinates: `browser_mouse_click_xy`, `browser_mouse_move_xy`, `browser_mouse_drag_xy`, `browser_mouse_down`, `browser_mouse_up`, `browser_mouse_wheel`. Use them for canvas and other surfaces without DOM semantics, with a screenshot to pick coordinates. Move, click and drag follow a curved path; the wheel scrolls in ticks; `browser_mouse_drag_xy` takes the drag `method`.
- `browser_click`, `browser_type`, `browser_hover`, `browser_drag`, `browser_check`, `browser_uncheck`, `browser_select_option` and `browser_fill_form` textboxes, checkboxes, radios and comboboxes act like a user (curved pointer moves, key-by-key typing) and take a few seconds. All but `browser_hover` take an optional `method` (per field in `browser_fill_form`); `jhrome tools <tool>` lists the choices. Check tools click only when the state differs. Selecting types the option label into the focused `<select>`, one value at a time; for several values, a disabled option or a non-`<select>` widget pass `"method":"playwright.injectedScript.selectOptions"`.
- `browser_handle_dialog`: answer a dialog raised by a Playwright tool. Sentence tools answer their own dialogs.
- `browser_file_upload`: after a click opens a file chooser. Paths must be inside `<data-dir>/playwright-output`.

## Page code

- Scripts run in an isolated world. `browser_evaluate` sees the DOM but not page JavaScript globals; pass `isolatedContext: false` to run in the page main world, where page scripts can observe it.
- `browser_run_code_unsafe` runs Playwright code against the page; use `page.evaluate(fn, arg, undefined, false)` there to reach page globals (the third argument is an options object; `false` there throws "Too many arguments").
- `browser_console_messages` returns nothing by default. `runtimeEnable: true` replays the current document's console messages and errors by sending `Runtime.enable` then `Runtime.disable`; the page can detect it while enabled.

## Storage

`browser_cookie_*`, `browser_localstorage_*`, `browser_sessionstorage_*`, `browser_storage_state`, `browser_set_storage_state`. These read and change the current tab's real profile (`browser_cookie_*` and the storage-state tools follow the current tab's profile; the localStorage and sessionStorage tools act on the current page); clearing reaches every site of that profile. Ask before clearing or setting anything.

## jhrome profile

- `jhrome_profiles` (`jhrome profiles`): every profile's directory, name, signed-in Google account and tab indexes, tabs of unknown profile, the current tab, data dir, Chrome user-data-dir, browser version.
- `jhrome_tab_new` (`jhrome tabs new`): `{"profile":"Profile 1","url":"…"}` opens a tab in that profile and selects it. `profile` defaults to `Default`. `{"account":"me@example.com"}` instead picks the one profile recorded as holding that account.
- `jhrome_accounts` (`jhrome accounts`): per profile, the Google account and per registry site the recorded account id, last login page visit and last check. `{"verify":true}` opens each site's account page in the profile to record what it shows; a profile window that is not the last active one comes to the front.
- `jhrome_extension_list`, `jhrome_extension_install` (absolute path of an unpacked extension; stays installed across restarts and loads from that directory, so keep it), `jhrome_extension_uninstall`, `jhrome_extension_reload`, `jhrome_extension_enable`, `jhrome_extension_disable` (`jhrome extensions …`): unpacked extensions of the current tab's profile. `reload` reads the directory again and returns the load error while the loaded version keeps running.
- `jhrome_extension_open_popup`, `jhrome_extension_open_side_panel` (`jhrome extensions popup|sidepanel <id>`): `{"id":"…"}` opens the popup (as an icon click, granting activeTab) or the side panel for the current tab, makes it the current tab and returns its snapshot. Calling again returns the open one. The current tab's window comes to the front; a popup closes when another window takes focus. While a popup or side panel is the current tab, the next one opens for the tab it was opened for; closing that tab closes its popup too.

## Not for agents

`browser_route*`, `browser_network_state_set`, recording, tracing and video tools, `browser_annotate`, `browser_highlight`, `browser_resume`, `browser_resize`, `browser_pdf_save`, `browser_generate_locator`, `browser_get_config` exist for testing and debugging. Use them only when the user asks.

Downloads are not supported.
