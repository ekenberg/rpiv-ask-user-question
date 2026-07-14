# Development

Read this before changing the `ask_user_question` extension. Explains the repo
layout, the fork relationship, and the day-to-day edit loop.

## Fork relationship

- **Upstream:** `juicesharp/rpiv-mono`, package `packages/rpiv-ask-user-question`.
  This fork was cut from upstream tag **`v1.20.0`**.
- **This repo:** `ekenberg/rpiv-ask-user-question` — a standalone pi package
  whose root *is* the package (so `pi install git:...` finds the `pi` manifest
  directly, instead of cloning the whole monorepo).
- A courtesy bugfix PR (notes-reopen) was sent to upstream from a separate
  monorepo clone at `/tmp/rpiv-mono` — see PR #111 in `juicesharp/rpiv-mono`.
  That PR is independent of this fork's inline-append work.

### The delta from upstream v1.20.0

Three behavioral changes live in this fork:

1. **Inline append (Ctrl+E)** — on any single-select option (preview or not),
   `Ctrl+E` opens an inline editor at the end of the option row; typed text
   becomes the answer's `notes`; `Enter` confirms option+addendum in one step
   (auto-advancing in multi-question mode); `Esc` discards. The old `n`
   notes-editor path is retired. Notes are **option-scoped** (`notesByTab`
   holds `{ option, note }`): an addendum typed on option A never surfaces on
   or attaches to option B.
   - Touch points: `state/state.ts` (`OptionNote`, `appendMode`),
     `state/key-router.ts` (`append_enter`/`append_confirm`/`append_exit`),
     `state/state-reducer.ts` (handlers + option-scoped `noteForOption`),
     `state/selectors/{projections,derivations}.ts`, `view/components/option-list-view.ts`,
     `view/components/wrapping-select.ts` (`setAppendMode` + inline render branch),
     `view/tab-content-strategy.ts` (hint line), `view/dialog-builder.ts`,
     `view/components/preview/preview-block-renderer.ts` (affordance text),
     `locales/en.json`.
2. **Notes-reopen bugfix** — `notesEnterHandler` reads `notesByTab` before
   `answers[tab].notes` (matching `switchTabResult`), so reopening the notes
   editor before confirming an option no longer wipes the note. (Kept even
   though the `n` path is retired, for consistency and in case of re-merge.)
3. **Submit-tab comment (Ctrl+E)** — on the Submit tab, `Ctrl+E` opens an
   inline editor (rendered in `SubmitTabStrategy.midRows`, between the answer
   summary and the bottom border); `Enter` **saves** the buffer into
   `state.submitComment` and returns to the Submit/Cancel picker —
   deliberately does NOT finalize the dialog (unlike per-option append,
   where Enter confirms — Enter-finalizes on the Submit tab would let a
   stray keystroke submit/cancel irreversibly). `Esc` discards the
   in-progress edit; the previously saved comment survives. The comment is a
   single dialog-level buffer that travels with whichever terminal action
   fires (`QuestionnaireResult.comment`): extra instructions on submit, a
   rationale on cancel. See `IMPL_SUBMIT_COMMENT.md` for the full design
   record.
   - Touch points: `state/state.ts` (`commentMode`, `submitComment`),
     `state/key-router.ts` (`comment_enter`/`comment_confirm`/`comment_exit`,
     intercepted like `appendMode` plus a Ctrl+E entry in the submit-tab
     block), `state/state-reducer.ts` (handlers, `set_input_focused` effect,
     `doneFor` attaches the comment), `state/questionnaire-session.ts`
     (`initialState`, `runEffect` case, `handleIgnoreInline` guard),
     `state/build-questionnaire.ts` (threads `inlineInput` into
     `DialogConfig` + `extraInvalidatables`), `view/dialog-builder.ts`
     (`DialogConfig.inlineInput`, hint constants), `view/tab-content-strategy.ts`
     (`SubmitTabStrategy.midRows` editor / staged-comment line,
     `footerRows` Ctrl+E hint on the prompt row), `tool/types.ts`
     (`QuestionnaireResult.comment?`), `tool/response-envelope.ts` (surfaces
     the comment in both the submit and cancel envelope text),
     `locales/en.json`.

## Repo layout — two repos, one remote

- **Source repo (this directory):**
  `/home/johan/srv/syncthing/projects/rpiv-ask-user-question`
- **pi's managed clone (what pi loads at runtime):**
  `~/.pi/agent/git/github.com/ekenberg/rpiv-ask-user-question/`

Both track the same GitHub remote (`git@github.com:ekenberg/rpiv-ask-user-question.git`).

## Branches

- `main` — stable snapshot.
- `live` — active development branch. `pi install ...@live` checks out `live`,
  so the clone pi loads is always on `live`. `@live` is just a branch name,
  not a pi keyword.

## Install (idempotent)

```bash
# SSH (reliable on this machine)
pi install git:git@github.com:ekenberg/rpiv-ask-user-question@live
# or HTTPS:
pi install git:github.com/ekenberg/rpiv-ask-user-question@live

/reload            # in a running pi session
```

Clean re-clone if needed: `pi update --extensions`.

**Migrating from the npm install:** if `@juicesharp/rpiv-ask-user-question`
was previously installed from npm, remove it first so the two don't collide
(pi dedupes by identity — npm name vs git URL are *different* identities):

```bash
pi remove npm:@juicesharp/rpiv-ask-user-question
pi install git:git@github.com:ekenberg/rpiv-ask-user-question@live
/reload
```

## The edit loop

1. Edit the clone (what pi loads):
   `~/.pi/agent/git/github.com/ekenberg/rpiv-ask-user-question/<file>`
2. `/reload` in pi, then test with an `ask_user_question` call.
   No reinstall needed — `/reload` re-reads the clone.
3. Repeat until happy.
4. Commit & push from the clone:
   `cd ~/.pi/agent/git/github.com/ekenberg/rpiv-ask-user-question && git commit -am "..." && git push`
5. Mirror into this source repo: `git pull` here (this dir also tracks origin).
   Or commit/push from here instead — both track `origin`; pick one place.

## Publishing changes to `main`

`live` is what's installed. To refresh the stable `main` snapshot:

```bash
git push origin live:main
```

## Testing the state machine without the TUI

The reducer + key-router are pure and can be driven headlessly with `tsx`:

```bash
# tsx is available in the pi tree:
TX=/home/johan/.pi/agent/npm/node_modules/.bin/tsx
# write a .mts driver (ESM — top-level await in deps requires ESM) that imports
# state/key-router.ts and state/state-reducer.ts, then call routeKey()/reduce().
$TX /tmp/your-driver.mts
```

Caveat: files importing *values* from `@earendil-works/pi-coding-agent`
(`view/dialog-builder.ts`, `state/build-questionnaire.ts`) won't resolve
standalone — that module is provided by the pi host at runtime. Type-only
imports are fine. `esbuild` is a quick syntax check that sidesteps resolution:

```bash
/home/johan/.pi/agent/npm/node_modules/.bin/esbuild <file.ts> --format=esm --target=es2022 > /dev/null
```

## Gotchas

- **`@live` must exist on the remote** or install fails with
  `pathspec 'live' did not match`. It does on this repo.
- Editing this source repo does NOT affect pi until you push and the clone
  picks it up (`pi update --extensions`, or `git pull` in the clone).
- `@juicesharp/rpiv-config` is a runtime dependency (listed in
  `package.json` `dependencies`); `pi install` runs `npm install` so it's
  fetched automatically.
- The installed 1.20.0 npm copy and this git fork are **different identities**
  to pi — don't keep both installed or you get two `ask_user_question` tools.
