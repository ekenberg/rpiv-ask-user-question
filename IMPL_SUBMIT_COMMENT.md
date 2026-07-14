# Implementation plan: `Ctrl+E` comment on the Submit/Cancel tab

Implements the feature specified in `PLAN.md`. This document is written for an
implementing model: follow it file-by-file, in order. Every touch point, code
snippet, and invariant below was verified against the current source
(fork of upstream v1.20.0, `live` branch) before writing.

**Read `DEV.md` and `AGENTS.md` first.** After editing, the reviewer will
`/reload` pi to test live — no build step exists; this is a TS-source pi
extension. There are no test files in this repo (`vitest` is configured but
no `*.test.ts` exist here — upstream keeps them in the monorepo), so
correctness is by review + live TUI testing.

---

## 1. Decided behavior (do not re-litigate; rationale recorded)

**UX flow (multi-question mode only — the Submit tab only exists when
`isMulti`, i.e. `questions.length > 1`):**

1. User is on the Submit tab (`state.currentTab === questions.length`).
2. `Ctrl+E` opens an inline comment editor (prefilled with any previously
   saved comment). A `Comment:` header + live editor appear between the
   answer summary and the bottom border.
3. **`Enter` saves the comment and returns to the Submit/Cancel picker. It
   does NOT submit or cancel.** `Esc` discards the in-progress edit (the
   previously saved comment survives).
4. While a saved comment exists, a dim `Comment: <text>` line is visible on
   the Submit tab so the user knows it is staged.
5. Enter on the picker (normal path) fires submit/cancel; the saved comment
   travels on the result as `comment` and is surfaced to the model:
   - Submit → appended to the envelope text as `Additional instructions: …`
   - Cancel → decline message becomes
     `User declined to answer questions (reason: …)`

**Decision — "commit-then-finalize" (deliberate divergence from append
mode):** per-option append mode uses Enter-finalizes ("Enter to answer with
addendum"). We do NOT mirror that here because Enter on the Submit tab
finalizes the ENTIRE dialog irreversibly; a stray Enter while typing a
rationale must not submit/cancel the questionnaire. So in comment mode Enter
only saves. The alternative (Enter = finalize focused picker row + comment)
was considered and rejected for this reason.

**Decision — one shared buffer:** a single dialog-level `submitComment`
attaches to whichever terminal action fires (per PLAN.md's lean). No separate
submit-vs-cancel fields.

**Single-question mode:** unaffected. No Submit tab exists there; per-option
`Ctrl+E` (answer `notes`) already covers it.

**Known accepted edge cases (do not fix; note in code comments only where
indicated):**
- Collapsed mode (`Ctrl+]`) during comment mode, then `Esc` → cancels the
  whole dialog. Identical to existing append-mode behavior; out of scope.
- `Tab`/arrows inside comment mode are forwarded to the inline editor (same
  as append mode); you cannot tab-switch while editing — `Esc` out first.
- If the summary + comment editor overflow a short terminal, the submit tab
  top-anchors (existing `focusedItemRowRange → undefined` behavior); the
  editor may scroll off on very short terminals. Accepted for v1.

---

## 2. `state/state.ts` — two new state fields

Add to `QuestionnaireState`, after `appendMode`:

```ts
	/**
	 * Submit-tab comment mode (Ctrl+E on the Submit tab): an inline editor is
	 * shown between the answer summary and the bottom border; Enter SAVES the
	 * buffer into `submitComment` and returns to the picker (it deliberately
	 * does NOT finalize the dialog — Enter-finalizes would make a stray
	 * keystroke submit/cancel irreversibly). Esc discards the in-progress
	 * edit. Mutually exclusive with `inputMode`/`appendMode` by construction
	 * (those exist only on question tabs).
	 */
	commentMode: boolean;
```

and after `notesByTab` (or near `submitChoiceIndex`):

```ts
	/**
	 * Dialog-level comment staged on the Submit tab. Travels on the terminal
	 * result as `QuestionnaireResult.comment` for BOTH submit (extra
	 * instructions) and cancel (rationale). Empty string = no comment.
	 */
	submitComment: string;
```

## 3. `state/key-router.ts` — three new actions + two routing changes

### 3a. Action union

Add to `QuestionnaireAction` (next to the `append_*` members):

```ts
	/** Enter submit-tab comment mode (Ctrl+E on the Submit tab). */
	| { kind: "comment_enter" }
	/** Save the comment buffer and return to the picker; `comment` is the raw editor buffer. */
	| { kind: "comment_confirm"; comment: string }
	/** Leave comment mode discarding the in-progress edit. */
	| { kind: "comment_exit" }
```

### 3b. Comment-mode intercept

In `routeKey`, add a block **immediately after** the `if (state.appendMode) { … }`
block (and before `if (state.chatFocused)`), mirroring the append intercept:

```ts
	if (state.commentMode) {
		if (kb.matches(data, KEYBIND_CONFIRM)) {
			// Deliberately does NOT finalize the dialog — see commentConfirmHandler.
			return { kind: "comment_confirm", comment: runtime.inputBuffer };
		}
		if (kb.matches(data, KEYBIND_CANCEL)) return { kind: "comment_exit" };
		// Everything else (printable chars, arrows, backspace) is forwarded to the
		// inline editor via the session's `handleIgnoreInline` fast path.
		return { kind: "ignore" };
	}
```

### 3c. Ctrl+E entry on the Submit tab

Inside the existing submit-tab block
(`if (runtime.isMulti && state.currentTab === runtime.questions.length) { … }`),
add as the **first** line of the block body (before the KEYBIND_CANCEL check):

```ts
		// Ctrl+E stages a dialog-level comment (submit instructions / cancel
		// rationale). Available regardless of unanswered questions (D1 allows
		// partial submission).
		if (matchesKey(data, Key.ctrl("e"))) return { kind: "comment_enter" };
```

`Key` / `matchesKey` are already imported (used by the append-mode Ctrl+E at
~line 271). No new imports.

## 4. `state/state-reducer.ts` — handlers, effect, doneFor, tab-switch reset

### 4a. New effect

Extend the `Effect` union (the set is compiler-enforced closed; the session's
`runEffect` switch must gain a matching case — step 5):

```ts
	| { kind: "set_input_focused"; focused: boolean }
```

Rationale: comment mode renders the session's `inlineInput` directly as a
component (like the notes editor renders `notesInput`), so its `focused` flag
must be driven declaratively, mirroring `set_notes_focused`.

### 4b. Handlers

Add next to the `append*Handler`s:

```ts
const commentEnterHandler: Handler<"comment_enter"> = (state, _action, _ctx) => ({
	// Prefill with the previously saved comment so Ctrl+E re-opens for editing.
	// The session's set_input_buffer effect also moves the cursor to end-of-buffer.
	state: { ...state, commentMode: true },
	effects: [
		{ kind: "set_input_buffer", value: state.submitComment },
		{ kind: "set_input_focused", focused: true },
	],
});

const commentExitHandler: Handler<"comment_exit"> = (state, _action, _ctx) => ({
	// Discard the in-progress edit; the previously SAVED comment survives.
	state: { ...state, commentMode: false },
	effects: [
		{ kind: "clear_input_buffer" },
		{ kind: "set_input_focused", focused: false },
	],
});

const commentConfirmHandler: Handler<"comment_confirm"> = (state, action, _ctx) => ({
	// Save-only: deliberately does NOT emit `done`. Submit/Cancel stay explicit
	// picker actions so a stray Enter while typing can never finalize the dialog.
	// Empty (after trim) clears a previously saved comment.
	state: { ...state, submitComment: action.comment.trim(), commentMode: false },
	effects: [
		{ kind: "clear_input_buffer" },
		{ kind: "set_input_focused", focused: false },
	],
});
```

Register all three in `HANDLERS` (`comment_enter`, `comment_confirm`,
`comment_exit`) — the mapped type makes omission a compile error.

### 4c. `doneFor` — attach the comment to the terminal result

`doneFor` is the single funnel for submit AND cancel (also chat-confirm and
question-tab Esc). Replace its body:

```ts
function doneFor(state: QuestionnaireState, ctx: ApplyContext, cancelled: boolean): ApplyResult {
	const result: QuestionnaireResult = {
		answers: orderedAnswers(state, ctx.questions),
		cancelled,
		...(state.submitComment.length > 0 ? { comment: state.submitComment } : {}),
	};
	return { state, effects: [{ kind: "done", result }] };
}
```

Note: this means a question-tab `Esc` cancel ALSO carries a staged comment if
one exists — correct (the comment is dialog-level context for the outcome).

### 4d. `switchTabResult` — defensive mode reset

In the `transitioned` literal (which already resets `inputMode`,
`appendMode`, `notesVisible`, `chatFocused`), add:

```ts
		commentMode: false,
```

Do **NOT** reset `submitComment` — it is dialog-level and must survive tab
switches (the `...state` spread preserves it).

## 5. `state/questionnaire-session.ts` — init, effect case, fast path

### 5a. `initialState()`: add both fields

```ts
		commentMode: false,
		submitComment: "",
```

### 5b. `runEffect`: new case (exhaustive switch — compiler will demand it)

```ts
			case "set_input_focused":
				this.inlineInput.focused = effect.focused;
				return;
```

(`Input.focused` is a public settable — the notes path already does
`this.notesInput.focused = …`.)

### 5c. `handleIgnoreInline`: include comment mode in the guard

```ts
		if (!this.state.inputMode && !this.state.appendMode && !this.state.commentMode) return;
```

## 6. `tool/types.ts` — result field

Add to `QuestionnaireResult`:

```ts
	/**
	 * Dialog-level comment typed on the Submit tab (Ctrl+E). Present only when
	 * non-empty. On submit: extra instructions for the model. On cancel: the
	 * user's rationale for declining.
	 */
	comment?: string;
```

`isQuestionnaireResult` needs no change (extra optional field).

## 7. `tool/response-envelope.ts` — surface the comment in BOTH branches

Add constants next to `DECLINE_MESSAGE`:

```ts
export const SUBMIT_COMMENT_PREFIX = "Additional instructions from the user:";
export const CANCEL_REASON_PREFIX = "Reason given:";
```

Rewrite `buildQuestionnaireResponse`:

```ts
export function buildQuestionnaireResponse(result: QuestionnaireResult | null | undefined, params: QuestionParams) {
	if (!result || result.cancelled) {
		const comment = result?.comment;
		const text =
			comment && comment.length > 0 ? `${DECLINE_MESSAGE}. ${CANCEL_REASON_PREFIX} ${comment}` : DECLINE_MESSAGE;
		return buildToolResult(text, {
			answers: result?.answers ?? [],
			cancelled: true,
			...(comment && comment.length > 0 ? { comment } : {}),
		});
	}
	const segments: string[] = [];
	for (let i = 0; i < params.questions.length; i++) {
		const a = result.answers.find((x) => x.questionIndex === i);
		if (a) segments.push(buildAnswerSegment(a));
	}
	if (segments.length === 0) {
		return buildToolResult(DECLINE_MESSAGE, { answers: result.answers, cancelled: true });
	}
	const tail = result.comment && result.comment.length > 0 ? ` ${SUBMIT_COMMENT_PREFIX} ${result.comment}` : "";
	return buildToolResult(`${ENVELOPE_PREFIX} ${segments.join(" ")} ${ENVELOPE_SUFFIX}${tail}`, result);
}
```

(The `segments.length === 0` fallback intentionally stays comment-less — it
is an all-empty-answers edge case, not a user-driven cancel.)

`buildAnswerSegment` and `buildToolResult` are unchanged.

## 8. `view/dialog-builder.ts` — hint constant + thread `inlineInput`

### 8a. Hint constants (next to `HINT_PART_APPEND`)

```ts
export const HINT_PART_COMMENT = "Ctrl+E to add a comment";
export const HINT_COMMENT_EDITING = "Enter to save comment · Esc to discard";
```

### 8b. `DialogConfig`: add the field (next to `notesInput`)

```ts
	inlineInput: Input;
```

### 8c. `DialogView` constructor: pass it to `SubmitTabStrategy`

```ts
		this.submitStrategy = config.isMulti
			? new SubmitTabStrategy({
					theme: config.theme,
					questions: config.questions,
					submitPicker: config.submitPicker,
					inlineInput: config.inlineInput,
				})
			: undefined;
```

## 9. `view/tab-content-strategy.ts` — submit-tab editor, staged line, hint

### 9a. Constant + imports

Near `const NOTES_HEADER = "Notes:";` add:

```ts
const COMMENT_HEADER = "Comment:";
```

Extend the `./dialog-builder.js` import list with `HINT_PART_COMMENT` and
`HINT_COMMENT_EDITING`. (`Input` is already imported as a type; `Text`,
`Spacer`, `Container` already imported.)

### 9b. `SubmitTabStrategyConfig`: add the input

```ts
export interface SubmitTabStrategyConfig {
	theme: Theme;
	questions: readonly QuestionData[];
	submitPicker: Component | undefined;
	inlineInput: Input;
}
```

### 9c. `SubmitTabStrategy.midRows` — the editor / staged-comment line

`midRows` renders between the body's trailing Spacer and the bottom border
and is NOT part of the fixed `footerRowCount = 5` invariant — this is the
same slot the question tabs use for the notes editor, so variable height here
is already handled by `DialogView.render`'s overflow logic. Replace:

```ts
	midRows(state: DialogState): Component[] {
		// Comment editor (Ctrl+E): rendered in the variable mid slot so the fixed
		// footerRowCount = 5 invariant is untouched (same pattern as the question
		// tabs' notes editor).
		if (state.commentMode) {
			return [
				new Text(this.config.theme.fg("muted", t("review.comment_header", COMMENT_HEADER)), 1, 0),
				this.config.inlineInput,
				new Spacer(1),
			];
		}
		// Staged-comment feedback: after Enter saves, show it so the user knows the
		// comment will travel with Submit/Cancel.
		if (state.submitComment.length > 0) {
			return [
				new Text(
					this.config.theme.fg("dim", `${t("review.comment_header", COMMENT_HEADER)} ${state.submitComment}`),
					1,
					0,
				),
				new Spacer(1),
			];
		}
		return [];
	}
```

### 9d. `SubmitTabStrategy.footerRows` — advertise Ctrl+E on the prompt row

Keep `footerRowCount = 5`. Modify the prompt row only: while editing, the
prompt row becomes the comment-mode hint; otherwise the Ctrl+E affordance is
appended to the existing ready/incomplete prompt. Also swap the prompt's
`Text` for `OneLineClippedText` so the longer line can never word-wrap and
break the 5-row invariant (same reasoning as the question-tab hint):

```ts
		const base =
			missing.length === 0
				? this.config.theme.fg("muted", t("review.ready", READY_PROMPT))
				: this.config.theme.fg(
						"warning",
						`${t("review.incomplete", INCOMPLETE_WARNING_PREFIX)} ${missing.join(", ")}`,
					);
		const promptText = state.commentMode
			? this.config.theme.fg("dim", t("hint.comment_editing", HINT_COMMENT_EDITING))
			: `${base} ${this.config.theme.fg("dim", `· ${t("hint.comment_add", HINT_PART_COMMENT)}`)}`;
		const out: Component[] = [new Spacer(1), new OneLineClippedText(promptText, 1), new Spacer(1)];
```

(The rest of `footerRows` — submitPicker / two-Spacer fallback — unchanged.
`OneLineClippedText` is defined at the top of this same file.)

Do **not** touch `buildHintText` or `QuestionTabStrategy` — comment mode is
submit-tab-only.

## 10. `state/build-questionnaire.ts` — wire `inlineInput` through

Two one-line changes in `QuestionnaireBuilder`:

- `buildDialog(...)`: add `inlineInput: this.inlineInput,` to the
  `DialogConfig` literal (next to `notesInput: this.notesInput`).
- `buildAdapter(...)`: change
  `extraInvalidatables: [this.notesInput]` →
  `extraInvalidatables: [this.notesInput, this.inlineInput]`
  (the inline input is now rendered directly on the submit tab, so it must be
  reachable by `invalidate()`).

## 11. `locales/en.json` — four new keys

```json
  "hint.comment_add": "Ctrl+E to add a comment",
  "hint.comment_editing": "Enter to save comment · Esc to discard",
  "review.comment_header": "Comment:"
```

(Keys must match the `t(key, fallback)` calls in steps 9c/9d exactly. Only
`en.json` exists in this fork; no other locale files to update.)

## 12. Explicitly NOT changed (verified non-issues)

- **`selectActiveView` / `ActiveView`**: comment mode only exists while
  `currentTab === questions.length`, so `activeView` is already `"submit"`.
  No new variant.
- **`selectOptionListProps` / option-list leak**: during comment mode the
  adapter still pushes the live `inputBuffer` (the comment text) into the
  LAST question's `OptionListView` (submit tab clamps to the last pane), but
  that component is never rendered on the submit tab (`SubmitTabStrategy`
  owns the body), and `appendActive` stays false, so nothing shows. On
  exit, `clear_input_buffer` empties it before any question tab renders.
- **`selectPreviewPaneProps`**: preview pane not rendered on submit tab; no
  gating needed for `commentMode`.
- **`selectSubmitPickerProps`**: the picker keeps its `❯` pointer while the
  comment editor is open. Cosmetic; accepted for v1.
- **Height math**: `footerRowCount` stays 4 (question) / 5 (submit);
  `maxFooterRowCount` unchanged → no layout shift on question tabs.
- **`events.ts`**: no payload change (the event fires at prompt time, before
  any comment can exist).
- **`tool/validate-questionnaire.ts`, `row-intent.ts`, sentinel labels**:
  untouched — no new rows, no new reserved labels.

## 13. Manual verification checklist (for the reviewer, post-/reload)

Multi-question dialog (≥2 questions):

1. Submit tab shows `… · Ctrl+E to add a comment` on the prompt row; the
   incomplete warning variant also shows it.
2. `Ctrl+E` → `Comment:` header + editor appear above the bottom border;
   prompt row reads `Enter to save comment · Esc to discard`; typing works,
   arrows/backspace/paste work; cursor visible at end.
3. `Enter` → editor closes, dim `Comment: <text>` line remains; picker
   navigable; **dialog did NOT close**.
4. `Ctrl+E` again → prefilled with the saved comment; `Esc` → edit discarded,
   saved comment still shown.
5. `Ctrl+E`, clear all text, `Enter` → staged line disappears (comment
   cleared).
6. Submit with a comment → tool result text ends with
   `… Additional instructions from the user: <comment>`; `details.comment`
   present.
7. Cancel (picker row 2) with a comment → text is
   `User declined to answer questions. Reason given: <comment>`;
   `details.cancelled === true`, `details.comment` present.
8. Submit/cancel with no comment → output byte-identical to pre-change
   behavior (no `comment` key in details, no text suffix).
9. Tab away from Submit and back → staged comment line still there.
10. Question tabs: append mode (Ctrl+E on an option), notes, "Type
    something.", multi-select, collapse (Ctrl+]) all behave exactly as
    before.
11. Single-question dialog: unchanged (no submit tab).
12. `Esc` on a question tab with a staged comment → decline message includes
    the reason (dialog-level comment travels with any cancel).

## 14. Documentation follow-ups (same commit)

- `PLAN.md`: mark the feature "Implemented — see IMPL_SUBMIT_COMMENT.md",
  resolve the open questions with the decisions from §1.
- `README.md`: add a bullet under Features describing the Submit-tab comment
  (mirroring the inline-append bullet, noting Enter saves / does not submit).
- `DEV.md`: extend "The delta from upstream v1.20.0" with this feature and
  its touch points.
- `AGENTS.md`: the hard rule says inline-append is "the single behavioral
  delta" — update it to enumerate both deltas.
