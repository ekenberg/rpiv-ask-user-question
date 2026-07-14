# PLAN

## [FEATURE] Transcript visibility (dialog no longer covers preceding output)

### What we want
The dialog currently renders as a full-height `overlay: true` bottom overlay
(`ask-user-question.ts`), which can overwrite up to 100% of the visible
terminal viewport — including the model's own text that immediately
preceded the tool call, which is almost always the most important context
for answering. This forced a collapse/expand cycle (`Ctrl+]`) on nearly
every invocation.

### Research summary (source-verified against `@earendil-works/pi-tui` and
`@earendil-works/pi-coding-agent`)
`pi-tui`'s overlay compositor (`TUI.compositeOverlays`, `tui.js`) literally
overwrites base-document lines at the overlay's screen position — it does
not push/scroll content, it replaces it in place, and with `maxHeight:
"100%"` that can be the entire viewport. The **default** (non-overlay)
`ctx.ui.custom()` path instead swaps the component into the SAME container
slot as the normal chat input editor (`interactive-mode.js`
`showExtensionCustom`), as a sibling of the chat transcript in the base
document — nothing is ever overwritten, only naturally scrolled, exactly
like any long user message. This is also the path used by pi's own bundled
reference example (`examples/extensions/questionnaire.ts`), which never
passes `overlay: true`. Full research, alternatives considered (capped
overlay `maxHeight`, static row caps), and the decision rationale are in
`IMPL_TRANSCRIPT_VISIBILITY.md` §Part A.

### Decision
Drop `overlay: true` entirely (Option C in the impl doc). Reserve a small,
named, tunable row budget (`CHROME_RESERVE_ROWS`) so `DialogView`'s existing
internal overflow/scroll-to-focus math doesn't assume it owns the full
terminal height (pi's own footer/widget rows sit below our dialog in the
non-overlay document and need headroom). Upstream's `Ctrl+]` collapse
feature is **removed** — see the next entry.

### Status
**Implemented + live-tested.** Verified in a live session: small dialogs
leave the preceding output visible; a tall multi-question/preview dialog
shows its heading and Submit picker without clipping. See
`IMPL_TRANSCRIPT_VISIBILITY.md` §A4 for the exact diff.

---

## [FEATURE] Remove the `Ctrl+]` collapse feature

### What we want
With non-overlay rendering (entry above) the transcript is always visible
above the dialog, so upstream's `Ctrl+]` collapse-to-read feature no longer
has a purpose. Initially it was kept (and its `Esc`-cancels-everything
footgun fixed to `Esc`-expands, plus a stronger pending indicator). Live
testing then showed collapse provides **no** benefit in non-overlay mode:
collapsing only frees rows at the *bottom* of the document (pi commits drawn
transcript in place, so nothing slides down to fill them), revealing no
additional transcript and just leaving an empty gap. To see older lines you
scroll the terminal, which collapse doesn't help with.

### Decision
**Remove it entirely** — the `collapsed` state, the `toggle_collapsed`
action/handler, the `Ctrl+]` intercept, the collapsed-mode lockout, the
collapsed render/indicator, and the collapse legend text in the footer hint
(`hint.collapse`). A dead keybinding that only punches a hole in the layout
is worse than no keybinding. Confirmed with the user after demonstrating the
gap live. (This supersedes the earlier "keep + make safe" plan.)

### Status
**Implemented** — see `IMPL_TRANSCRIPT_VISIBILITY.md` §Part B (revised).

---

## [FEATURE] `C-e` comment on the Submit/Cancel tab

### What we want
When the user has answered all questions and is sitting on the **Submit** tab
(the summary/review tab, tab index `runtime.questions.length`), pressing
`Ctrl+E` should open an inline editor — just like the per-option inline-append
on question tabs — whose buffer becomes a **dialog-level comment** attached to
the terminal action:

- If the user then **Submits**, the comment travels back as extra
  info / instructions for the model (e.g. "FYI I picked X because Y",
  "after doing this also run the lint step").
- If the user instead **Cancels**, the comment travels back as a
  **rationale for cancelling** (so the model understands *why* the
  questionnaire was declined, rather than a bare cancel signal).

### Current state
- Per-**option** inline append (`appendMode`) exists on question tabs:
  `state/state.ts` (`appendMode`, `OptionNote`), `state/key-router.ts`
  (`append_enter`/`append_confirm`/`append_exit` at the `Key.ctrl("e")`
  branch ~line 271), `state/state-reducer.ts` (`appendEnterHandler` etc.),
  `view/components/wrapping-select.ts` (`setAppendMode`), and the footer hint
  `HINT_PART_APPEND = "Ctrl+E to append"` in `view/dialog-builder.ts`,
  injected by `buildHintText` in `view/tab-content-strategy.ts`.
  That addendum is **option-scoped** (`notesByTab`) and shows up per answer as
  `notes` in the envelope (`tool/response-envelope.ts:38`).
- The Submit tab has **no** free-text affordance. Its key handling lives in
  `state/key-router.ts` (the block at ~line 243, gated by
  `runtime.isMulti && state.currentTab === runtime.questions.length`). Only
  `submit_nav` (toggle Submit/Cancel), `submit`, and `cancel` are recognized.
  `submit`/`cancel` both funnel through `doneFor(...)` in
  `state/state-reducer.ts` (`submitHandler`/`cancelHandler`).
- The terminal envelope is `QuestionnaireResult`
  (`tool/types.ts:138`: `answers: QuestionAnswer[]`, `cancelled: boolean`).
  There is no dialog-level comment field today — `notes` is per-answer only.

### Why this is a new path (not a reuse of `appendMode`)
`appendMode` is hard-wired to a focused **option row** and writes into the
per-answer `notesByTab` map. The Submit tab has no option rows, and the
comment must attach to the *whole dialog outcome*, not a single answer. So this
needs its own state + effect plumbing rather than reusing `appendMode` as-is.

### Suggested implementation checklist
1. **State** (`state/state.ts`): add a dialog-level field, e.g.
   `submitComment: string` (or `dialogComment`), distinct from `notesByTab`.
   Consider whether Submit and Cancel share one buffer or need two — see
   open questions below.
2. **Key router** (`state/key-router.ts`, Submit-tab block ~line 243): detect
   `matchesKey(data, Key.ctrl("e"))`; if already in comment mode, let
   `KEYBIND_CONFIRM`/`KEYBIND_CANCEL` commit/discard (mirror the existing
   `appendMode` fast-path at ~line 186). Emit new actions
   `comment_enter` / `comment_confirm` / `comment_exit`.
3. **Reducer** (`state/state-reducer.ts`): add handlers. `comment_confirm`
   stores the buffer into the dialog-level field. `submit`/`cancel` handlers
   (or `doneFor`) read that field and pass it into the terminal effect.
4. **Session effect** (`state/questionnaire-session.ts`): extend the resolve
   envelope so the comment is surfaced — likely a new field on
   `QuestionnaireResult` (`tool/types.ts`) and rendered in
   `buildQuestionnaireResponse` / `response-envelope.ts` for both the
   submit and cancel branches (today both cancel branches return
   `cancelled: true` with no comment).
5. **Hint line**: `SubmitTabStrategy.footerRows` (`view/tab-content-strategy.ts`)
   currently does NOT use `buildHintText`; it builds its own footer. Add a
   "Ctrl+E to add a comment" hint there (and the inline-edit hint while
   comment mode is active, mirroring `buildHintText`'s appendMode branch).
6. **Render**: reuse the existing inline-input rendering (the `setAppendMode`
   / `renderInlineInputRow` machinery in `wrapping-select.ts` and
   `option-list-view.ts`) against the submit/cancel row, or draw a dedicated
   comment line in `SubmitTabStrategy`.

### Open questions — resolved (see `IMPL_SUBMIT_COMMENT.md` §1 for the full
rationale)
- **One buffer or two?** Resolved: single shared `submitComment`, attaches to
  whichever terminal action fires (also a question-tab Esc-cancel, since all
  paths funnel through `doneFor`).
- **Enter-finalizes vs commit-then-finalize?** Resolved: commit-then-finalize.
  Enter in comment mode only saves `submitComment` and returns to the picker;
  it does NOT submit/cancel. Deliberate divergence from per-option append
  mode (where Enter finalizes) because Enter on the Submit tab is
  irreversible — a stray keystroke while typing a rationale must never
  submit/cancel the whole dialog.
- **Where in the envelope?** Resolved: `QuestionnaireResult.comment?: string`
  (`tool/types.ts`), surfaced in `buildQuestionnaireResponse`
  (`tool/response-envelope.ts`) as `Additional instructions from the user: …`
  on submit and `Reason given: …` appended to the decline message on cancel.
- **i18n**: resolved — `hint.comment_add`, `hint.comment_editing`,
  `review.comment_header` added to `locales/en.json`.

### Status
**Implemented** — see `IMPL_SUBMIT_COMMENT.md` for the full design record and
file-by-file change list. Touch points also enumerated in `DEV.md` → "The
delta from upstream v1.20.0" (delta #3). Not yet synced to the pi-loaded
clone (`~/.pi/agent/git/github.com/ekenberg/rpiv-ask-user-question/`) or
live-tested — see `DEV.md` → "The edit loop".
