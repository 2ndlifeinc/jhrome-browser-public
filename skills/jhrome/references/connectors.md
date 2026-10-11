# Connectors

A flow asks the outside world for what is not on the page: a certificate password (`ask`), an SMS sent from a phone (`gate`), or its own input. A connector answers. It lives only on this machine, in `<data-dir>/connectors/<name>/` (`jhrome profiles` prints the data dir), so credentials and device ids never enter a repository.

## Files

`SKILL.md` starts with frontmatter:

```markdown
---
name: corporate-fincert
description: One line on what it answers and where the value comes from
inputs:
  - obiz.kbstar.com/corporate-fincert-sign-in
flows:
  - obiz.kbstar.com/corporate-transfer
asks:
  - 사업자 금융인증서 비밀번호 여섯 자리
gates:
  - 등록된 휴대폰에서 화면의 확인코드를 문자로 발신한다
timeoutSeconds: 120
env:
  - EXTRA_VARIABLE_NAME
---
```

- `inputs` are the flow ids whose input it fills, `flows` the other flow ids whose asks or gates it answers. A connector answers a sentence only for a flow listed in one of them, and one with asks or gates but neither list does not load.
- `asks` and `gates` are the exact sentences the flow uses (`jhrome flow` reports a sentence no connector lists). One sentence, one connector: two connectors listing it stop the run.
- `run` is an executable (any language, shebang line). On Windows, where shebangs do not run, write `run.exe`, `run.cmd` or `run.ps1` instead. Keep non-secret settings such as device serials in a file next to it.
- A secret is read only inside `run`, only as `ahcred get --id <item id> --field json` with the item id written as a literal in `run`. No config file, argument or environment variable may change which item is read, and the connector checks the item's `kind` field before answering.
- You never read, print, compare or type a secret yourself, not even to check a connector. Check a connector with a request whose `flow.input` is wrong (it must refuse) and with an `input` request (it returns no secret). Do not dump past sessions or snapshots that might hold one into files.

## Protocol

`run` reads one JSON object on stdin and prints one JSON object on stdout.

| kind | stdin | stdout |
|---|---|---|
| `input` | `{"kind":"input","flow":{"id","input"}}` | `{"value":{...flow input}}`; keys the caller gave win |
| `ask` | `{"kind":"ask","sentence","url","flow":{"id","input"}}` | `{"value":"..."}` |
| `gate` | `{"kind":"gate","sentence","until","url","facts":{...},"flow":{"id","input"}}` | `{"ok":true}` or `{"ok":false,"reason":"..."}` |
| `check` | `{"kind":"check","flow":{"id","input"}}` | `{"ok":true}` or `{"ok":false,"reason":"..."}` |

- `check` comes before the flow touches the page, to every connector listing the flow, and before any `input` request: its `flow.input` holds only what the caller gave, never a connector's preset. Check items by the ids written in `run`, not by a value a preset would supply. Answer `ok: true` only when everything the connector will need for that flow is reachable now: the credential vault opens the item (read a non-secret field and print nothing), the device is online and unlocked. Never send, type or print anything in a check. A reason the user can act on ("the phone is locked; unlock it") is the point: all reasons come back together, so the user fixes everything at once instead of the run failing at the SMS step.

- Choose the value from `flow.input` (for example the business number), never from a guess. Refuse when it does not match exactly one stored item.
- A gate does its one action and returns; the flow waits for `until` itself. Never resend or retry inside a connector; answer `ok: false` when unsure.
- Only `PATH`, `HOME`, `USER`, `LOGNAME`, `LANG`, `LC_ALL`, `TMPDIR`, `SHELL` and the names under `env` reach the process. A non-zero exit, a timeout or non-JSON stdout stops the run, and the last 400 characters of stderr are shown: never write a secret to stderr.
- Read secrets only with `ahcred get --id <item id> --field json`, the id written as a literal in `run`. Never copy a secret into the connector folder.

Test a connector by piping a request into it before running the flow:

```bash
echo '{"kind":"check","flow":{"id":"<flow id>","input":{}}}' | <data-dir>/connectors/<name>/run
echo '{"kind":"input","flow":{"id":"<flow id>","input":{}}}' | <data-dir>/connectors/<name>/run
```

A flow that stops with `Not started: connectors of <flow> are not ready` touched nothing. Tell the user every listed reason, wait until they say it is fixed, then start it again.
