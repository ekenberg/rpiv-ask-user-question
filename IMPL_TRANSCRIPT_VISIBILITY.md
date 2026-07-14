# Implementation plan: transcript visibility + collapsed-mode safety

Two related UX problems, researched and planned as a pair (per user request):

1. The dialog, rendered as a full-height bottom overlay, can cover the tail
   of the model's own preceding output — the exact text most relevant to
   answering the question — forcing a collapse/expand cycle on nearly every
   invocation.
2. Collapsing to read that hidden text gives no strong visual cue that a
   questionnaire is still pending, and a reflexive `Esc` while collapsed
   silently cancels the *entire* multi-question dialog instead of just
   leaving "read mode".

Item 2's fix reduces reliance on collapse (item 1 fixes the root cause for
most dialogs) but doesn't eliminate the collapse feature — very tall dialogs
on short terminals still benefit from it — so item 2's safety fix remains
independently necessary.

---

## Part A — Item 1: transcript visibility

### A1. Research: why the overlay covers previous output (source-verified)

Read `@earendil-works/pi-tui` (`dist/tui.js`, `dist/tui.d.ts`) and
`@earendil-works/pi-coding-agent` (`dist/modes/interactive/interactive-mode.js`,
`docs/tui.md`, `docs/extensions.md`, `examples/extensions/questionnaire.ts`)
directly — no assumptions.

**The TUI's rendering model.** `TUI extends Container` (`tui.js`). The
*entire* app — header, chat transcript (`chatContainer`), status, widgets,
the input editor (`editorContainer`), footer — is one `Component` tree.
Every frame, `TUI.doRender()` renders that whole tree to a `lines[]` array
(the full virtual "document", which can be far taller than the terminal),
then only the **last `termHeight` lines** of that document are the visible
viewport (`tui.js` `doRender`, `previousViewportTop` math).

**How overlays composite (`tui.js` `compositeOverlays`, ~line 573).**
`compositeOverlays(lines, termWidth, termHeight)` receives the *already
viewport-sliced* base content and, for each visible overlay, computes
`row`/`col` via `resolveOverlayLayout` (anchor/margin/maxHeight math, ~line
450) relative to that same viewport, then does:

```js
result[idx] = this.compositeLineAt(result[idx], truncatedOverlayLine, col, w, termWidth);
```

i.e. it **overwrites** `result[idx]` — the base document's line at that
screen row — with the overlay's line. This repo's call
(`ask-user-question.ts`): `overlay: true, overlayOptions: { anchor:
"bottom-center", width: "100%", maxHeight: "100%", margin: { left: 0, right:
0, bottom: 0 } }`. With `maxHeight: "100%"` the overlay can claim every row
of the viewport, so a moderately tall dialog (multi-question, previews,
multi-select) can and regularly does overwrite 100% of the visible screen —
including the model's just-printed text that triggered the tool call, which
was never scrolled into terminal-native scrollback (it's still "on screen"
in the *current* viewport slice when the overlay composites over it in the
very same render pass). This is not a bug in pi-tui; `maxHeight: "100%"`
plus `bottom-center` is doing exactly what it's told. It's a **usage**
problem: the dialog claims more screen than the situation needs, discarding
already-useful content instead of coexisting with it.

**The non-overlay default (`interactive-mode.js` `showExtensionCustom`,
~line 1897).** Passing no `overlay` option (or `overlay: false`, the
default) takes a completely different code path:

```js
this.editorContainer.clear();
this.editorContainer.addChild(component);
this.ui.setFocus(component);
```

The component literally **replaces the input editor** — it becomes a normal
sibling `Container` in the SAME base tree
(`ui.addChild(this.chatContainer) ... ui.addChild(this.editorContainer) ...`,
`interactive-mode.js` ~line 497-503, confirmed order: header →
loadedResources → **chatContainer** → pendingMessages → status →
widgetContainerAbove → **editorContainer** → widgetContainerBelow → footer).
No `compositeOverlays` call happens at all (`overlayStack` stays empty). The
document simply grows by however many lines the component renders, and the
viewport (last `termHeight` lines of the *whole* document) naturally shows
`(as much preceding transcript as fits) + (our dialog) + (footer)` — nothing
is overwritten, because there's nothing to composite over.

This is also the path used by pi's own bundled reference implementation,
`examples/extensions/questionnaire.ts` ("Multi-step wizard tool" per
`docs/extensions.md`'s example table) — it never passes `overlay: true`.
Overlay mode is documented as "Experimental" (`docs/extensions.md` §"Overlay
Mode (Experimental)"); non-overlay is the established default path.

**Conclusion:** the user's own description — *"toggling injects/removes the
popup ... so previous text is always visible and lines up with top of
popup"* — is *exactly* what the non-overlay path already does, natively, via
existing pi-coding-agent primitives. No new pi-tui/pi-coding-agent API is
needed.

### A2. Options considered

| # | Approach | Mechanism | Verdict |
|---|---|---|---|
| A | Cap `overlayOptions.maxHeight` to e.g. `"70%"`, keep `overlay: true` | Percentage is re-resolved against live `termHeight` every render (`parseSizeValue`, `tui.js`), so it's resize-adaptive. Must also cap `DialogView`'s own internal `getTerminalRows()` budget to the *same* value, or `compositeOverlays`'s `overlayLines.slice(0, maxHeight)` (`tui.js` ~line 588) truncates the dialog's own footer off the *bottom* — losing the Submit picker. | Works, but always reserves a *fixed fraction* of the screen even for a tiny 1-question dialog with no previews — doesn't fully solve "forces toggling on practically every invocation" for the common small-dialog case. |
| B | Static row-count `maxHeight` (e.g. `maxHeight: N`) | `OverlayOptions.maxHeight` is `SizeValue = number \| \`${number}%\`` (`tui.d.ts`); a plain number is **not** re-resolved against terminal size on resize (only percentage strings are). | Rejected: breaks on terminal resize. |
| **C** | **Drop `overlay: true` entirely — default `ctx.ui.custom()` (editor-replace) mode** | See A1. Dialog becomes part of the normal document flow; transcript is *never* overwritten, only naturally scrolled (same as any long input). Small dialogs leave almost the whole transcript visible; only pathologically tall dialogs push much of it out of view — same trade-off every terminal chat app already makes for a long user message. | **Recommended.** Matches the user's own description of the desired behavior. Uses the non-experimental, pi-team-authored reference path. |

**Decision: Option C.**

### A3. Implication / risk this introduces, and how it's mitigated

Under Option C, `DialogView`'s own internal "fit inside `getTerminalRows()`"
overflow logic (`view/dialog-builder.ts` `DialogView.render`, the 3-region
scroll-to-focus partition — this machinery is unchanged and does all the
real work) must be told a slightly *smaller* budget than the raw terminal
height, because pi's own persistent chrome (`footer`, and any
`widgetContainerBelow`/`statusContainer` content from `setFooter`/`setWidget`/
`setStatus` — ours or another active extension's) sits **after**
`editorContainer` in the document (see the confirmed child order in A1) and
must still fit in the same viewport. If `DialogView` assumes it owns 100% of
`tui.terminal.rows` and actually only `rows - 1` or `rows - 2` are free
before the footer needs to render, the viewport (bottom-anchored to the
*whole* document, footer last) will clip the **top** of the dialog (its own
heading/question text) instead — the footer is structurally safe (it's
always last), but our own dialog's header isn't, unless we leave headroom.

Mitigation: reserve a small, named, easily-tunable constant
(`CHROME_RESERVE_ROWS`) subtracted from `tui.terminal.rows` before it's
handed to `DialogView` as its row budget, with a floor
(`MIN_DIALOG_ROWS`) so tiny terminals don't get a negative/zero budget
(`DialogView`'s existing "terminal too small — show just chrome" fallback
still applies below the floor). This is a heuristic, not a precise
measurement — pi-coding-agent exposes no "remaining rows for
`editorContainer`" API (checked `ExtensionUIContext` in
`core/extensions/types.d.ts`: only `tui.terminal.rows`/`.columns` are
available). **Residual risk, explicitly flagged for live testing:** if
another active extension adds a taller `setWidget`/`setFooter`, the reserved
constant may be too small and the dialog's heading could clip by a row or
two on a terminal exactly that tall — cosmetic only (the interactive
middle/footer of the dialog, where focus lives, stays visible via
`DialogView`'s existing scroll-to-focus centering), not a functional break.
Report if seen; the constant is a one-line tune.

Everything else about `DialogView`'s sizing/overflow/scroll-to-focus
machinery, `QuestionTabStrategy`/`SubmitTabStrategy`, and the whole state
machine is **unchanged** — this is a compositing-model change, not a dialog
logic change.

### A4. Implementation

**`ask-user-question.ts`** — drop overlay entirely:

```ts
const result = await ctx.ui.custom<QuestionnaireResult>((tui, theme, _kb, done) => {
	const session = new QuestionnaireSession({
		tui,
		theme,
		params: typed,
		itemsByTab,
		done,
	});
	return session.component;
});
```

(Remove the `overlay: true, overlayOptions: {...}` second argument entirely
— matches `examples/extensions/questionnaire.ts`'s call shape exactly.)

**`state/build-questionnaire.ts`** — reserve chrome headroom in the row
budget fed to `DialogView`:

```ts
/**
 * Non-overlay dialogs (see IMPL_TRANSCRIPT_VISIBILITY.md §A3) render as a
 * normal sibling of pi's chat transcript, ABOVE pi's own footer/widget rows
 * in the document — not as a full-viewport overlay. Reserve a small, tunable
 * budget so DialogView's internal "fit inside N rows" overflow math doesn't
 * assume it owns the entire terminal height; otherwise the outer viewport
 * (bottom-anchored to the whole document, footer last) would clip the TOP of
 * the dialog (its own heading) instead of leaving chrome room. Heuristic —
 * pi-coding-agent exposes no exact "remaining rows" API — tune up if the
 * dialog heading clips with other extensions' widgets/footer active.
 */
const CHROME_RESERVE_ROWS = 2;
const MIN_DIALOG_ROWS = 6;
```

and change:

```ts
private readonly getTerminalRows = () => this.tui.terminal.rows;
```

to:

```ts
private readonly getTerminalRows = () => Math.max(MIN_DIALOG_ROWS, this.tui.terminal.rows - CHROME_RESERVE_ROWS);
```

No other file needs to change for item 1 — `DialogView`, the state machine,
and every question/submit-tab feature (including the just-shipped Ctrl+E
submit comment) are untouched; they only ever consumed `getTerminalRows()`
as an opaque budget.

### A5. What happens to "collapsed" mode — REMOVED (revised after live testing)

**Initial plan:** keep collapse, and make it safer (Esc-expands) + more
visible. **Revised decision (this is what shipped):** remove it entirely.

Live testing disproved the premise that collapse still helps. The hypothesis
was that collapsing a tall dialog would let the bottom-anchored viewport
slide *up* and pull earlier transcript down into the freed rows. It does
not: pi commits already-drawn transcript in place (top-down), so when the
dialog block shrinks, only the rows *below* it (footer) move up and the
bottom of the screen goes empty — no additional transcript is revealed,
regardless of transcript length. Since non-overlay mode already keeps the
transcript visible above the dialog, and older lines are reached by
scrolling the terminal (which collapse can't help with), collapse provides
**zero** benefit in this fork — it only ever mattered for uncovering the
*overlay* we deleted. A keybinding that just leaves a layout gap is worse
than none, so the whole feature is removed. Part B below is retained as the
historical record of the superseded "keep + make safe" plan; see "Part B
revision" immediately under the heading for what actually shipped.

---

## Part B — Item 2: ~~collapsed-mode safety~~ → REMOVED

> **REVISION (shipped):** the plan below ("keep collapse, fix Esc, strengthen
> the indicator") was implemented first, then **superseded**: after live
> testing showed collapse reveals nothing in non-overlay mode (see §A5), the
> user confirmed removing it. What actually shipped is the **deletion** of
> the entire collapse feature: the `collapsed` state field
> (`state/state.ts`), the `toggle_collapsed` action + `Ctrl+]` intercept +
> collapsed lockout (`state/key-router.ts`), `toggleCollapsedHandler` + its
> HANDLERS entry (`state/state-reducer.ts`), `collapsedRender` + the render
> branch + the `collapsed: false` init (`state/questionnaire-session.ts`),
> the collapse hint constants (`view/dialog-builder.ts`), the collapse legend
> in `buildHintText` (`view/tab-content-strategy.ts`), and the
> `hint.collapse`/`hint.expand*`/`hint.collapsed_*` locale keys
> (`locales/en.json`, `locales/zh.json`). The original plan text is kept
> below for the design record.

### B1. Original behavior (source-verified, this repo) — the code being removed

### B1. Current behavior (source-verified, this repo)

- Collapse toggle: `Ctrl+]` → `{ kind: "toggle_collapsed" }`, intercepted
  **first**, unconditionally, in `routeKey` (`state/key-router.ts`, before
  every other inner-mode check) — so collapse can happen mid-append,
  mid-comment-edit, etc., and resumes exactly where it left off on expand
  (only `state.collapsed` flips; no buffers are touched).
- Collapsed-mode lockout (`state/key-router.ts`, right after the `Ctrl+]`
  intercept):
  ```ts
  if (state.collapsed) {
  	if (kb.matches(data, KEYBIND_CANCEL)) return { kind: "cancel" };
  	return { kind: "ignore" };
  }
  ```
  i.e. while collapsed, **`Esc` cancels the entire dialog** — the *only*
  live keystroke. Comment at the call site: *"swallow every keystroke
  except cancel so the user can read the now-uncovered transcript without
  accidentally mutating answers or notes"* — a deliberate quick-cancel
  escape hatch, not a bug.
- Collapsed render (`state/questionnaire-session.ts`, `collapsedRender`):
  one dim line, `theme.fg("dim", " Ctrl+] to expand · Esc to cancel ")`
  (`COLLAPSED_HINT` from `view/dialog-builder.ts`).

### B2. The problem

`Esc` is overloaded app-wide as "back out one level" — every other inner
mode in this state machine treats it that way (`notesExitHandler`,
`appendExitHandler`, `commentExitHandler` all just leave the sub-mode).
Collapsed mode is the **only** state where `Esc` instead terminates the
*entire* multi-question dialog. That inconsistency is exactly what bit the
user: collapse to peek at transcript, reflexively hit `Esc` expecting to
leave "read mode", and the whole questionnaire — including any staged
Submit-tab comment, all answered questions — is gone.

Compounding it: the collapsed hint line is a single **dim** row, visually
identical in weight to the rest of the (also dim) chat chrome, easy to lose
entirely once the user has scrolled their eyes up into the transcript they
collapsed the dialog to read.

### B3. Decision

1. **Re-map `Esc` while collapsed to expand** (same action as `Ctrl+]`,
   i.e. `{ kind: "toggle_collapsed" }` instead of `{ kind: "cancel" }`) —
   makes collapsed mode consistent with every other "Esc backs out one
   level" mode in the app. A deliberate cancel is still one extra keystroke
   away (expand, then `Esc` from the fully-rendered dialog, which still
   cancels as before) — a quick-cancel-from-collapsed shortcut is traded for
   never silently discarding a mid-flight questionnaire by reflex. This is
   the direct answer to *"research if keyboard input could be blocked"*:
   yes, trivially — reuse the existing `toggle_collapsed` action, no new
   state or action needed.
2. **Strengthen the collapsed-line visual indicator**: bold + `warning`
   theme color (same color already used for the Submit tab's "unanswered
   questions" warning — consistent "pay attention" language across the
   extension) instead of `dim`, plus a `❓` glyph, plus live
   answered/total progress (e.g. `2/3 answered`) computed from
   `state.answers.size` / `this.questions.length` — so a glance at the
   bottom of a long scrollback tells the user both *that* something is
   pending and *how far along* they are, without expanding.

### B4. Implementation

**`view/dialog-builder.ts`** — replace the single composed constant with
smaller, individually-translatable pieces; `HINT_PART_CANCEL` is no longer
part of the collapsed hint (Esc no longer cancels there) but stays exported
(still used by the normal expanded-dialog footer hint):

```ts
export const HINT_PART_EXPAND = "Ctrl+] to expand";
export const HINT_COLLAPSED_PENDING = "❓ Question pending";
```

Remove `export const COLLAPSED_HINT = [HINT_PART_EXPAND, HINT_PART_CANCEL].join(" · ");`
(its only consumer is rewritten below).

**`state/questionnaire-session.ts`** — rewrite `collapsedRender`:

```ts
const collapsedRender = (_width: number): string[] => {
	const answered = this.state.answers.size;
	const total = this.questions.length;
	const progress = total > 1 ? ` (${answered}/${total} ${t("hint.collapsed_answered", "answered")})` : "";
	const label = `${t("hint.collapsed_pending", HINT_COLLAPSED_PENDING)}${progress} — ${t("hint.expand", HINT_PART_EXPAND)}`;
	return [theme.bold(theme.fg("warning", ` ${label} `))];
};
```

(update the `dialog-builder.js` import list: drop `COLLAPSED_HINT`, add
`HINT_COLLAPSED_PENDING`.)

**`state/key-router.ts`** — collapsed lockout:

```ts
	// Collapsed-mode lockout: while collapsed, swallow every keystroke except
	// expand, so the user can read the now-uncovered transcript without
	// accidentally mutating answers or notes. Esc expands (same as Ctrl+])
	// instead of cancelling the whole dialog — matches the "Esc backs out one
	// level" convention every other inner mode uses (notes/append/comment
	// exit handlers); a reflexive Esc while just trying to leave "read mode"
	// must not silently discard the entire questionnaire. A deliberate cancel
	// is still available: expand, then Esc from the fully-rendered dialog.
	if (state.collapsed) {
		if (kb.matches(data, KEYBIND_CANCEL)) return { kind: "toggle_collapsed" };
		return { kind: "ignore" };
	}
```

No new `QuestionnaireAction` variant, no reducer change — `toggle_collapsed`
and `toggleCollapsedHandler` already exist and are reused as-is.

**`locales/en.json`** — add, remove the now-orphaned key:

```json
  "hint.expand": "Ctrl+] to expand",
  "hint.collapsed_pending": "❓ Question pending",
  "hint.collapsed_answered": "answered",
```

remove `"hint.expand_line"` (its only call site is rewritten above).

---

## Manual verification checklist (visual — **cannot be self-verified**; requires the user's live terminal after `/reload`)

Unlike the Submit-tab comment feature (verifiable via the returned text
envelope), both items here are pure rendering/interaction changes with no
text-observable signal from a tool call. The implementing/verifying agent
**cannot** confirm these visually — please check on `/reload`:

1. **[PASSED]** Single small question (no previews) — preceding assistant
   text remains visible above the dialog (not painted over).
2. **[PASSED]** Multi-question, multi-select-with-previews dialog tall
   enough to fill most of a normal terminal — dialog heading and Submit-tab
   picker both fully visible (not clipped top or bottom). If the heading
   ever clips with other extensions' widgets active, bump
   `CHROME_RESERVE_ROWS`.
3. Resize the terminal (grow/shrink) while the dialog is open — layout
   should re-flow without clipping the picker/hint row.
4. **Collapse removed:** pressing `Ctrl+]` should now do **nothing** (it's an
   ordinary un-bound key — the dialog ignores it), and the footer hint line
   should **no longer** contain `Ctrl+] to collapse`. `Esc` always cancels
   the dialog from any tab (there is no collapsed state to intercept it).
5. Sanity-check every other existing behavior is unaffected: multi-select,
   previews, per-option Ctrl+E append, Submit-tab Ctrl+E comment (both
   verified in the prior feature), "Type something.", chat-escape row.

## Rollback

If non-overlay mode causes any regression not caught above (e.g. an
unexpected focus-stealing interaction with another extension's active
overlay — noted as a low-probability edge case: both overlay and non-overlay
`ctx.ui.custom()` already unconditionally call `this.ui.setFocus(component)`
today, so this is not a *new* risk, just worth naming), revert is a single
targeted diff: restore the `overlay: true, overlayOptions: {...}` block in
`ask-user-question.ts` and the plain `() => this.tui.terminal.rows` in
`build-questionnaire.ts`. Note that reverting non-overlay mode would also
re-justify the removed collapse feature (it only mattered under the overlay),
so a full revert should restore collapse too — both live on the same
conceptual change.

## Documentation follow-ups (same commit)

- `PLAN.md`: two new `## [FEATURE]` entries summarizing both items, status,
  and linking here.
- `DEV.md`: extend "The delta from upstream v1.20.0" with a 4th entry.
- `README.md`: mention non-overlay rendering (+ collapse removal) under
  Features.
- `AGENTS.md`: hard rule now enumerates three deltas.

**Revision note:** Part B originally shipped as "keep collapse + make it
safe"; live testing (see §A5) then showed collapse is useless in non-overlay
mode, so it was removed entirely with the user's confirmation. The docs
above reflect the removal.
