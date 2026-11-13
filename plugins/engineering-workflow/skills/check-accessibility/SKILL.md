---
name: check-accessibility
description: Run an accessibility audit against the app or a set of changed UI files and fix the findings. Covers WCAG 2.2 AA via axe-core, static JSX checks, keyboard and focus review, and MUI-specific patterns. Use when asked to check, audit or fix accessibility, a11y, screen reader support, keyboard navigation, or WCAG compliance.
---

# check-accessibility

Audits against **WCAG 2.2 Level AA** and fixes what can be fixed safely. Three passes —
automated, static, and manual — because automated tooling catches roughly a third of real
barriers, and the other two thirds are the ones that actually stop someone using the app.

## 1. Scope it

Ask what is in scope if it is not obvious: the whole app, one route, or the UI touched by
the current branch. Default to **the diff** when invoked from `pr-gate` or on a feature
branch, and the whole app only when asked.

Identify the stack before installing anything:

```bash
cat package.json | grep -E '"(react|@mui/material|vue|@playwright/test|vitest|next)"'
```

Bronson UI is React with MUI. If a repo is something else, adapt — but say what you found
rather than running React tooling against a Streamlit app.

## 2. Automated pass — axe-core

Against a running app, through Playwright **if the repo already has it** — check for
`@playwright/test` before reaching for axe at all; standing Playwright up from nothing is
its own project, not part of an audit.

Adding a dev dependency is the user's call. Ask first, use the package manager the
lockfile names (`pnpm-lock.yaml` → `pnpm`, `yarn.lock` → `yarn`, otherwise `npm`), and say
whether you will keep the dependency or revert it when the audit is done. If install
scripts need approving, approve the named packages only — never `--all`.

```bash
pnpm add -D @axe-core/playwright   # or npm i -D / yarn add -D, per the lockfile
```

```ts
import AxeBuilder from '@axe-core/playwright'

test('home page has no detectable a11y violations', async ({ page }) => {
  await page.goto('/')
  const { violations } = await new AxeBuilder({ page })
    .withTags(['wcag2a', 'wcag2aa', 'wcag21a', 'wcag21aa', 'wcag22aa'])
    .analyze()
  expect(violations).toEqual([])
})
```

Two environment traps will waste your time here:

- **npm 11 does not run install scripts unless approved.** Playwright's browser download
  is a postinstall. After installing, run `npm install-scripts ls` and approve it, or
  `npx playwright install` explicitly — otherwise the run fails in a way that looks
  unrelated to accessibility.
- **`reuseExistingServer` is on outside CI**, so the suite can silently attach to an
  unrelated dev server already holding the port and report violations for a different
  application. Before trusting a result, check the port and run with `CI=true` so
  Playwright starts exactly what the config declares.

No Playwright and no appetite to add it → run axe against built static output with
`@axe-core/cli`, and say that coverage is limited to the routes you could reach.

## 3. Static pass — JSX

```bash
pnpm add -D eslint-plugin-jsx-a11y   # per the lockfile; ask first
```

**Default to a throwaway config for the run** and leave the repo's ESLint config alone.
Adding these rules to the shared config makes them run in CI, which turns an audit into a
blocking check for everyone — the opposite of what this skill is allowed to do.

Only edit the shared config if the user asks for it, and then at `warn`, or scoped to the
paths that changed, after checking whether CI runs lint. The flat config below (ESLint 9,
what current repos use) is the shape for either case:

```js
import jsxA11y from 'eslint-plugin-jsx-a11y'

export default [
  // ...existing config
  jsxA11y.flatConfigs.recommended,
]
```

Legacy `.eslintrc`: add `plugin:jsx-a11y/recommended` to `extends`.

For a one-off run that leaves the repo config alone, write a throwaway flat config and
point at it — do not reach for `--no-eslintrc` or `--rulesdir`, which ESLint 9 removed.

**It needs a TypeScript parser.** With `--no-config-lookup` nothing else supplies one, so
a config carrying only `jsxA11y.flatConfigs.recommended` reports a parsing error at the
first type annotation in every `.tsx` file and evaluates no jsx-a11y rules at all — the
run then reads as clean, or as a pile of errors that are not accessibility findings.

Check which parser package the repo has first — importing the wrong one fails with
module-not-found before a single file is linted:

```bash
if node -e "require.resolve('typescript-eslint')" 2>/dev/null; then echo meta
elif node -e "require.resolve('@typescript-eslint/parser')" 2>/dev/null; then echo parser-only
else echo no-ts
fi
```

Keep it as `if`/`elif`: an `&&`/`||` chain of the same three checks prints both `meta` and
`parser-only` when the first one succeeds.

Three outcomes, not two. A plain JavaScript React app has neither package, and importing
`@typescript-eslint/parser` there fails with module-not-found before a file is linted —
use the `no-ts` config below, which needs no parser at all.

The probe answers "can the import resolve from here", which is the question that matters.
Under pnpm's isolated `node_modules` that can be `no-ts` on a TypeScript repo whose
workspace package does declare the parser: run it from the package holding the UI code,
not the repo root, and check `package.json` if the answer contradicts the file extensions
you are about to lint.

```js
// tmp-a11y.mjs - "meta": the typescript-eslint package is present
import tseslint from 'typescript-eslint'
import jsxA11y from 'eslint-plugin-jsx-a11y'

export default [
  { files: ['**/*.{ts,tsx,js,jsx}'],
    languageOptions: {
      parser: tseslint.parser,
      parserOptions: { ecmaFeatures: { jsx: true } },
    } },
  jsxA11y.flatConfigs.recommended,
]
```

```js
// tmp-a11y.mjs - "parser-only": the repo has @typescript-eslint/parser alone
import tsParser from '@typescript-eslint/parser'
import jsxA11y from 'eslint-plugin-jsx-a11y'

export default [
  { files: ['**/*.{ts,tsx,js,jsx}'],
    languageOptions: {
      parser: tsParser,
      parserOptions: { ecmaFeatures: { jsx: true } },
    } },
  jsxA11y.flatConfigs.recommended,
]
```

```js
// tmp-a11y.mjs - "no-ts": a plain JavaScript repo, no TypeScript parser needed
import jsxA11y from 'eslint-plugin-jsx-a11y'

export default [
  { files: ['**/*.{js,jsx}'],
    languageOptions: { parserOptions: { ecmaFeatures: { jsx: true } } } },
  jsxA11y.flatConfigs.recommended,
]
```

```bash
npx eslint --config tmp-a11y.mjs --no-config-lookup <paths>
```

Check the output is findings rather than parse errors before you report a count.

**Do not add this to CI as a blocking job** unless the user asks. A lint rule that fails
everyone's build on day one gets disabled within a week.

Static analysis catches the markup-level issues axe cannot see without rendering: missing
`alt`, handlers on non-interactive elements, `tabIndex` misuse, labels not associated with
controls.

## 4. Manual pass — the part tooling misses

Walk the scoped UI and check each of these. These are where real barriers live:

- **Keyboard only.** Tab through every interactive element. Can you reach everything, in
  an order that matches the visual layout, and get back out of every menu, dialog and
  date picker? Is focus visible at every stop?
- **Focus management.** Opening a dialog moves focus into it; closing returns focus to the
  trigger. Focus never lands on `<body>`. Route changes move focus to the new heading.
- **Accessible names.** Every control has one. An icon-only button with no `aria-label`
  reads as "button" and is the single most common finding in a MUI codebase.
- **Headings and landmarks.** One `<h1>` per page, no skipped levels, content inside
  `main`/`nav`/`header` rather than a wall of `div`s.
- **Status and errors.** Form errors are associated with their field via
  `aria-describedby`, not just coloured red. Async results are announced through a live
  region, not silently swapped in.
- **Colour is never the only signal.** Error, selected and required states carry text or
  an icon too.
- **Tables and data grids.** Real headers with scope, a caption or accessible name, and
  sort state exposed via `aria-sort`.
- **Motion.** Animation respects `prefers-reduced-motion`.

### MUI specifics

- `IconButton` needs `aria-label`. MUI does not infer one from the icon.
- `TextField` needs `label`, or `aria-label` when visually label-less. A `placeholder` is
  not a label — it disappears on input and is not reliably announced.
- `Dialog` needs `aria-labelledby` pointing at its `DialogTitle`.
- `Tooltip` on a disabled control is unreachable — a disabled element fires no events.
  Wrap it in a `span`, or keep the control enabled and explain the constraint.
- Custom `sx` colour overrides bypass the theme's contrast decisions. Check any
  hand-written colour against 4.5:1 for body text and 3:1 for large text and UI borders.
- `DataGrid` is reasonable out of the box; custom `renderCell` content is where names and
  roles get lost.

## 5. Fix

**Fix without asking** — these are mechanical and have one correct answer:

- `alt=""` on an image that is clearly decorative. Writing *descriptive* alt text, and
  any `tabIndex` or focus-order change, goes in "Stop and ask" — only the author knows
  what the image is for, or what the intended focus order is.
- Missing `aria-label`, `aria-labelledby`, `aria-describedby` where the accessible name
  is already in the markup to copy
- Label-to-control association
- Semantic element swaps: `div` with `onClick` becomes `button` — always with
  `type="button"` and a style reset (or MUI's `ButtonBase`). A bare `<button>` inside a
  form defaults to `type="submit"` and will submit it on click.
- Heading level corrections and landmark wrapping
- `aria-sort`, `aria-current`, `aria-expanded`, `aria-live` where the state already exists
- Contrast fixes that resolve to an existing theme token
- `prefers-reduced-motion` guards


**Stop and ask** — these are design or product decisions, not defects to patch:

- Anything needing new visible text: button labels, error copy, instructions
- Contrast fixes that change brand colours rather than using a token
- Reordering or restructuring a layout to fix focus order
- Any `tabIndex` change, including removing a positive one. A `tabIndex={3}` is usually a
  deliberate order somebody set; report it with what it does, and let the author decide
- Writing descriptive `alt` text for an image that is not plainly decorative
- Replacing a component wholesale
- Anything where the accessible behaviour is genuinely ambiguous

Never fix a finding by suppressing it. No `eslint-disable` on a jsx-a11y rule, no
`axe.configure` exclusion, no `aria-hidden` on something a user needs. If a finding is a
false positive, say which one and why, and leave it visible.

## 6. Verify

Re-run both automated passes and report before and after counts. Then re-check by keyboard
the specific interactions you changed — an `aria-label` that satisfies axe can still be
wrong for the control it names.

## 7. Report

```
Scope: apps/portal/src/intake (14 files, 3 routes)
Standard: WCAG 2.2 AA

axe-core:      18 violations → 2
jsx-a11y:      11 errors → 0
Manual:        6 findings → 4 fixed, 2 need a decision

Fixed
  - 7 IconButtons had no accessible name (toolbar, grid row actions)
  - Intake dialog did not return focus to its trigger on close
  - Validation errors were colour-only; now linked via aria-describedby
  ...

Needs a decision
  - The "Status" column communicates state by chip colour only. Adding a text label
    changes the column width — do you want the label, an icon, or a tooltip?
  - Filter bar focus order runs right-to-left because the DOM order is reversed for
    layout. Fixing it means restructuring the flex container.

Not fixed (false positive)
  - axe flags the decorative header gradient as a contrast failure. It sits behind no
    text; left as is.
```

Report what you actually ran. If you could only reach three of eleven routes because the
rest need a seeded login, say that — a clean result on an unreachable app is not a pass.
