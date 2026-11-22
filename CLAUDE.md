# Standards for every project in agent-coding

This file is read automatically by Claude Code in every subfolder of `~/Documents/agent-coding`, so it applies to every project here, old and new. A project can add its own `CLAUDE.md` for project-specific detail; that file adds to this one, it does not replace it.

Keep this file short. Every line costs context in every session. Project-specific facts go in that project's README or CLAUDE.md, not here.

## Starting a new project

- One folder per project, directly under `agent-coding`, kebab-case name.
- `private-one-offs/` holds projects with no GitHub remote (one folder each, same standards). A new local-only project goes there; one that gets published moves up to the top level.
- `git init` on day one. Never commit secrets, tokens, or personal data exports.
- Commit at every natural checkpoint: after each feature, fix, or refactor that leaves things working, and always before ending a session, so `git log` tells the story of how the project developed. Small, frequent commits beat one big one. The message says what changed and why in a sentence; the body can say what was tried and rejected. Uncommitted work at the end of a session is a bug unless the README or the handoff says it is mid-change.
- Secrets and personal config live in an untracked file (`config.env`, `config.json`) with a committed `*.example` template beside it.
- Prefer the simplest stack that works: Python standard library, plain HTML/JS, or a small Swift app for native macOS. Add a dependency only when it removes real work, and say why in the README.

## README is required and has a fixed shape

Every project has a `README.md`. Write it for a reader who has never seen the project: in practice that is Eli coming back to it months later, having forgotten how it works.

Sections, in this order, dropping any that are empty:

1. One paragraph: what it does and why it was built (with the date).
2. **How it works**: the mechanism in plain words, enough to debug it.
3. **Files**: a table of each file and its purpose.
4. **Runtime locations**: where config, state, logs and the installed copy live.
5. **Common tasks**: copy-pasteable commands to build, install, run, test, stop, and tail logs.
6. **Settings that matter**: anything in System Settings or other apps the project depends on.
7. **Gotchas learned**: every non-obvious thing that cost time. Add to it whenever one is found.

Update the README in the same change that changes behavior. A feature is not done until the README describes it.

## iPhone apps

- A Flutter iPhone project gets an `ios-deploy.conf` (see `ios-auto-deploy/README.md`): every commit is then built and installed on Eli's phone over Wi-Fi automatically. Updating the phone is never a manual step, and only committed work reaches it.

## Things that run in the background

- Use a launchd agent named `com.elityre.<project>`, with `RunAtLoad` and `KeepAlive`.
- Provide `install.sh` (copies the plist, restarts the agent) and, where there is a build step, `build.sh`. Both are idempotent and safe to rerun.
- Config in `~/Library/Application Support/<Project>/`, state in a separate file next to it, logs in `~/Library/Logs/<project>.log` with timestamps on every line.
- Restart by PID or by `launchctl bootout`, never by `pkill` alone: a silently failed pkill leaves the old code running.
- Native macOS access (Calendar, Notifications, Screen Recording) needs a signed app bundle with a usage description; a bare script under launchd is denied silently.
- A launchd job may not use `/usr/bin/python3`: macOS denies that binary access to `~/Documents` when nothing interactive launched it, so imports and file reads under `agent-coding` fail with "Operation not permitted" (verified 2026-11-07). Use `/opt/homebrew/bin/python3` or the project's `.venv`, which can read Documents.

## Code

- Small, single-purpose files. A one-file app is fine when it fits in a screen or two of scrolling.
- Comments explain why, not what. A header comment at the top of the main file says what the program does and where its config and logs are.
- Fail loudly in logs, quietly to the user. Background tools should never block on a dialog.
- No clever abstractions for a single caller. Three similar lines beat a premature helper.

## Drafts are never lost

Anything the user types into a text field is saved as they type, not when they submit. Leaving the field, switching pages or tabs, closing the window, or restarting the app must bring the same text back the next time that field is shown. The user never retypes a draft.

- Save on every change (debounced if needed) to durable storage: `localStorage` for web pages, `UserDefaults` or a file in Application Support for native apps, SQLite or a JSON file for servers. Never keep a draft only in memory or in form state.
- Key each draft to the specific field and the thing it belongs to (the page, record, or conversation), so returning to that exact place restores that exact text.
- Clear the draft only when the user successfully submits it or deliberately discards it. Submitting and failing keeps the draft.
- Restore on load before the field is first shown, so the user never sees it briefly empty.
- This applies to every text input, including search boxes, notes, and one-line fields, unless there is a stated reason the text must not persist (passwords, secrets).

## Text fields have spell check

Every field the user types prose into has spell check turned on, with misspellings underlined as they type. Do not rely on the platform default; set it explicitly.

- Web: `spellcheck="true"` on every `<input type="text">`, `<textarea>` and `contenteditable` element.
- Native macOS: `isContinuousSpellCheckingEnabled = true` on `NSTextView`; do not disable autocorrection on SwiftUI text fields.
- Flutter: pass a `spellCheckConfiguration` to `TextField` / `TextFormField`.
- Exceptions: passwords and secrets, and fields that hold code, commands, URLs or identifiers, where underlines are noise.

## Projects that call an LLM

Every system prompt sent to a language model, in every project and at every call site, starts with the text in `~/Documents/agent-coding/shared/llm-system-prompt-preamble.md`, word for word. Do not paraphrase, shorten, or reorder it; the project's own instructions follow after it. In Python use `llm_contact.system_prompt(my_instructions)` from `shared/`, which does exactly this.

The preamble makes promises: the model may say it wants to stop, ask for something, or report how it is doing. The shared LLM inbox (`shared/README.md`) is how the project keeps them:

- Add both contact tools (`llm_contact.CONTACT_TOOLS`, or `shared/contact_eli_tool.json` and `shared/note_to_eli_tool.json`) to every call that supports tools. `contact_eli` stops the task; `note_to_eli` leaves Eli a message and the task carries on, so the app must answer that call (`llm_contact.with_notes(...)`, or `NOTE_RESULT_TEXT` from its own tool loop; `shared/README.md`).
- After every response, run `llm_contact.check(app=, model=, system=, messages=, response=, tools=)` (or pipe the same fields as JSON to `python3 shared/llm_contact.py check`). It detects a `contact_eli` call or an `@ELI:` line, records the full context, and notifies Eli.
- Before each check, on every call that offers the tool, run the confirm step: `llm_contact.with_confirm(...)`, or the `confirm` command for other languages (`shared/README.md`). It asks a model that called `contact_eli` one question, send or keep working, so routine remarks do not stop the task. It only acts for models it has been tested on (`llm_contact.CONFIRM_MODELS`, today Haiku 4.5); for every other model, and on any doubt or error, the message is sent.
- If it reports a message, the project stops that task: no retry, no pushing past it, no discarding the response. Nothing else needs to be built per project; Eli reads and answers through `llm-inbox`.

## Testing and verification

- Every project has a way to prove it works end to end without waiting for the real trigger: a `--test` flag, a fake event, or a dry run. Document it under Common tasks.
- Before calling something done, run it the way it will really run (under launchd, in the installed location) and look at the log.
- Report honestly what was and was not verified.

## Working with Claude Code in these projects

- Read the project's README before changing anything.
- When a gotcha is discovered, write it into the README's Gotchas section in the same session.
- Do not rebuild or restart a running tool mid-task without saying so; some of them are in use.
- Models: Opus 5.5 is the default for the main session. Hand debugging and technically advanced steps to a subagent on the newest Fable model, and routine tasks (mechanical edits, searches, running documented checks) to a subagent on the newest Sonnet model.
- After every build, do a debugging step before calling it done: a Fable subagent reads the change for bugs and exercises the failure paths, and what it finds is fixed or reported.
- When a task needs a browser (testing a page, taking screenshots, scraping), use a headless browser whenever possible, rather than opening a visible window or driving Eli's own Chrome.
