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

Against a running app, through Playwright if the repo already has it:

```bash
npm i -D @axe-core/playwright
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
npm i -D eslint-plugin-jsx-a11y
npx eslint --no-eslintrc --plugin jsx-a11y --rulesdir /dev/null <paths>
```

Prefer wiring `plugin:jsx-a11y/recommended` into the existing ESLint config if the repo
has one, rather than a one-off invocation — but **do not add it to CI as a blocking job**
unless the user asks. A lint rule that fails everyone's build on day one gets disabled
within a week.

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

- Missing `alt`, `aria-label`, `aria-labelledby`, `aria-describedby`
- Label-to-control association
- Semantic element swaps: `div` with `onClick` becomes `button`
- Heading level corrections and landmark wrapping
- `aria-sort`, `aria-current`, `aria-expanded`, `aria-live` where the state already exists
- Contrast fixes that resolve to an existing theme token
- `prefers-reduced-motion` guards
- Removing `tabIndex` above 0

**Stop and ask** — these are design or product decisions, not defects to patch:

- Anything needing new visible text: button labels, error copy, instructions
- Contrast fixes that change brand colours rather than using a token
- Reordering or restructuring a layout to fix focus order
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
