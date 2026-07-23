# AGENTS.md

rpiv-ask-user-question — a pi extension registering the `ask_user_question` tool:
a tabbed questionnaire dialog (single/multi-select, side-by-side previews,
inline append, Submit review tab).

This is a **fork** of `@juicesharp/rpiv-ask-user-question` (v1.20.0 base), kept
on a separate line of development for the inline-append feature. See DEV.md for
the fork relationship and how this repo relates to upstream and to the
courtesy PR sent to `juicesharp/rpiv-mono` (merged, released in upstream v2.1.0).

## Orientation
- `index.ts` — extension entrypoint; registers the tool.
- `ask-user-question.ts` — tool definition, prompt snippet/guidelines, execute().
- `state/` — the questionnaire state machine: `key-router.ts` (keys → actions),
  `state-reducer.ts` (actions → state+effects), `state.ts` (state shape),
  `questionnaire-session.ts` (runtime that owns the state cell + input cells),
  `row-intent.ts` (sentinel-row metadata), `selectors/` (pure derivations).
- `tool/` — validation, types, response envelope, answer formatting.
- `view/` — TUI components: `components/wrapping-select.ts` (the option list),
  `components/option-list-view.ts`, `components/preview/` (preview pane + markdown),
  `tab-content-strategy.ts` (per-tab layout + hint line), `dialog-builder.ts`.
- `locales/` — i18n strings (only active if `@juicesharp/rpiv-i18n` is installed).
- `package.json` — pi package manifest (`pi.extensions: ["./index.ts"]`).
- `DEV.md` — **read this before changing anything**: branches, install, edit loop.
- `README.md` — install + usage for end users.
- `PLAN.md` — planned enhancements for this fork; design notes + wiring
  checklist, cross-referencing the detailed `IMPL_*.md` docs for shipped
  features (`IMPL_SUBMIT_COMMENT.md`, `IMPL_TRANSCRIPT_VISIBILITY.md`).

## Conventions
- Develop on the `live` branch; `pi install git:git@github.com:ekenberg/rpiv-ask-user-question@live`
  is what the running pi session loads.
- Mirror stable snapshots to `main` with `git push origin live:main`.
- After editing the clone pi loads, `/reload` pi — no reinstall needed.
- This fork tracks upstream `juicesharp/rpiv-mono` conceptually but is **not**
  kept in sync automatically. Re-merging upstream changes is a manual,
  case-by-case decision (upstream `main` has diverged from the v1.20.0 base).

## Hard rules
- The inline-append feature (Ctrl+E), the Submit-tab comment feature
  (Ctrl+E on the Submit tab), and non-overlay rendering (which also removed
  upstream's now-redundant `Ctrl+]` collapse feature) are the three
  behavioral deltas from upstream v1.20.0 in this fork, plus the notes-reopen
  bugfix. Keep those deltas small and documented so future upstream re-merges
  stay tractable. (The notes-reopen bugfix has since been upstreamed — merged
  as PR #111, released in upstream v2.1.0.) See `DEV.md` → "The delta from upstream v1.20.0" and
  `IMPL_SUBMIT_COMMENT.md` / `IMPL_TRANSCRIPT_VISIBILITY.md` for the design
  records.
- Do not author reserved labels (`"Other"`, `"Type something."`, `"Chat about this"`,
  `"Next →"`) in tool calls — they are rejected at validation.
- Keep README/DEV claims consistent with the actual install state in
  `~/.pi/agent/settings.json`.
