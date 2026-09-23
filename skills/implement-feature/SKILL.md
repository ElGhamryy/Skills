---
name: implement-feature
description: Plan and implement new features in a React frontend. Use when the user asks to implement a new feature, page, component, hook, store slice, or API integration. Enforces step-by-step planning with pseudocode, asks clarifying questions before coding, and follows the project's existing file/folder conventions.
---

# Implement Feature

Tech stack and code conventions live in the project's `CLAUDE.md` and `.claude/rules/` — read them first and do not repeat them here. This skill adds the workflow and default structure.

**The project wins.** Every convention below is a default for when the project has no pattern of its own. If the codebase already does something differently and consistently, follow the codebase.

## Workflow

1. **Clarify** — ask about anything ambiguous; never assume.
2. **Locate** — map the work to the project's file tree. Find the closest existing feature and use it as the reference.
3. **Plan** — write a numbered pseudocode plan covering files, data flow, state, routes, edge cases, and loading/empty/error states.
4. **Confirm** — wait for user approval. In plan mode, also wait after each implemented step.
5. **Implement** — follow project rules.
6. **Verify** — manually walk loading/empty/error states, and RTL if the app supports it. Check: no new imports from a library the project is migrating away from; no consumer CSS overriding a shared component's own styling; no hardcoded px/hex where a design token exists (resolve the token to confirm its value, and snap to the nearest token rather than keeping the literal); no inline styles, no renamed props, no stray `console.log`. New styles use logical properties (`padding-inline`, `margin-inline-start`) rather than `left`/`right` plus a mirrored RTL block.
7. **Changelog** — the last step of every feature. See below.

## Changelog

If the project keeps a changelog, add an entry for the finished feature in the **existing format and location** — copy the structure of the most recent entry. Create a new file only if that is how the project already splits it (per month, per release, …).

Ask the user for the release version and deployment date — never guess them. If entries link a ticket, take the ticket id from the branch name when it carries one (`ABC-295-some-feature` → `ABC-295`); otherwise ask.

If the project has no changelog, skip this step.

## Clarifying questions (common gaps)

- Does a similar component or hook already exist to reuse?
- Is there a design (Figma, screenshot) to match, and which states does it cover?

## Component library first

If the project has a design system or shared component library, it is the source of truth. Find it (a `design-system/`, `ui/`, or `components/` folder, or a package in `package.json`) and **search its exports before you build anything.** Legacy code that bypasses it is migration debt, not a pattern to copy.

- If the library genuinely has no equivalent, **stop and flag it as a gap** — never hand-roll a lookalike with matching CSS. This is the single most common failure.
- **The library owns its components' styling.** Padding, radius, colour, border, shadow, and hover/selected/disabled states belong to the component. Never write consumer CSS to override them — if the render differs from the design, flag the divergence instead. Consumer CSS covers layout *between* components only: containers, gaps, margins, grid.
- **Don't lean on a component's internal spacing** for your layout. If a component's internal padding changes, anything that relied on it silently breaks. The container declares its own `gap`.
- If the project wraps a library component (e.g. a pagination wrapper that adds translation and data fallbacks), use the wrapper, not the raw component.

### Contracts to check when switching components

When replacing one component library with another (or a legacy component with a new one), check every call site for these — each is a silent bug, not a compile error:

| Contract | What goes wrong |
|---|---|
| Handler signature | `onChange(value)` vs `onChange(event, value)`. A handler written for the other signature reads the wrong parameter and silently does nothing. |
| `error` prop type | Some fields take a boolean, others **render the value** as the message. Passing `""` or a boolean to the second kind shows nothing or the wrong thing. Pass a message or `undefined`. |
| Where refs and extra props go | Toggles/checkboxes often forward refs and spread props onto a visually hidden `<input>`. A tooltip or popover that clones its child to attach a ref ends up with a 1px hit target — wrap the child in a real element. |
| Default rendered tag | A text component that defaults to `<p>`, nested in something that already renders a `<p>`, triggers `validateDOMNesting`. Fix at the call site (e.g. `as="span"`), never in the shared component. |
| Compound component nesting | State CSS pinned to an exact depth (e.g. a trigger that must be the **immediate** child of its item) breaks when you add a wrapper. |

## Component conventions

- **Containers** — use whatever the project uses. If the component library ships no layout primitive (`Box`/`Stack`), a plain `<div className="…">` with layout in the stylesheet is correct. Remember a `Stack` supplied `display: flex` + `flex-direction` implicitly and a `div` supplies neither.
- **Text** — if the library has a typography component, every standalone user-facing string goes through it with a library `variant`; no raw text styled by hardcoded font CSS. Text a library parent already styles (button label, chip, menu item, alert message) stays as-is — don't double-wrap.
  - **`variant` sets the look, the tag sets the meaning.** If the typography component picks its HTML tag from `variant`, pass the tag explicitly whenever the meaning differs — a metric styled like a heading must not become a document heading. It looks identical, which is what makes it dangerous.
- **Buttons, tooltips, date inputs, feedback toasts** — the library's version, not a legacy shared one and not a raw element. Check whether the library's toast/alert helper translates its message; if not, translate at the call site.
- **List keys** — never key a mapped list on the array index, or on anything derived from it. Key on identity the data already carries: the item's `id` when it has one, otherwise the item's own value. Do not mint ids (`uuid` and friends) just to produce a key. Index keys attach component state to a position, so a removal shows the removed row's state on its neighbour. If an item carries neither an `id` nor a stable value, document the exception in a comment on the `key` line.

## Data flow

Default, when the project uses a server-state library such as TanStack Query and has no other pattern:

```
component → query/mutation hook (colocated in the feature) → API module (the project's HTTP client)
```

The hook imports the API function and uses it as `queryFn` / `mutationFn` directly. No extra "controller" or "service" layer in between unless the project already has one.

Hook naming: `useGetXxx`, `useCreateXxx`, `useUpdateXxx`, `useDeleteXxx`, `useExportXxx`.

### Three hook folders, split by primitive

A feature's hooks never share one folder. Split them by **what the hook calls**, not what it is named:

```
<feature>/
  queries/      # every hook whose body calls useQuery / useInfiniteQuery
  mutations/    # every hook whose body calls useMutation
  hooks/        # everything else — composite/orchestration hooks, useQueryClient-only
                # hooks, feature flags, form and handler hooks
```

- **The primitive decides, never the name.** A `useGetXxx` that calls `useMutation` (fired on demand, not cached) lives in `mutations/`, with a one-line comment saying why.
- **`useQueryClient` alone is not a query.** A hook that only invalidates or reads the cache is orchestration and belongs in `hooks/`.
- **No barrel files.** Import the hook by its own path; an `index` per folder just hides which primitive you are reaching for.

## Translation keys

If the app is translated, **never inline a translation key string.** Every key goes into a `SCREAMING_CASE` constant and the call site reads the constant:

```jsx
// no
<Text>{t("dashboard.filters.tooltip_title")}</Text>

// yes
const TOOLTIP_TITLE_KEY = "dashboard.filters.tooltip_title";
<Text>{t(TOOLTIP_TITLE_KEY)}</Text>
```

- **Scope follows the usual rule.** A key used by one file is declared above the component in that file; a key used by two or more, or a whole group of them, moves to the feature's constants file.
- **Name for the string, not the location.** `SAVE_AND_APPLY_KEY`, not `BUTTON_2_KEY`. Suffix `_KEY` for one key, `_KEYS` for a map.
- **Why:** a dotted key is a magic string the compiler and linter cannot see. Named once, a rename is one edit, a typo shows up as an undefined identifier instead of a silently untranslated key rendering its own path to the user, and every string the screen can show is greppable in one place.
- This covers anything fed to the translation function, including keys passed to toast helpers and to a field's `error` prop.

## Component file organization

Organize non-trivial component files into labeled `//#region <Name>` / `//#endregion` blocks so each concern is scannable and collapsible.

- Only region what earns it — small components need no regions; add them as a file grows past a scannable size.
- Rename regions to fit the feature; keep the coarse ordering (state → derived → queries → handlers → setup → effects → JSX).
- Annotate non-obvious refs/values with a one-line intent comment (why it exists), not a mechanism explanation.

## File placement

Follow the project's existing layout — find where the nearest similar feature put each concern. Default when there is no clear pattern:

| Concern                | Location                                                  |
| ---------------------- | --------------------------------------------------------- |
| Page (top-level route) | `<pages>/<FeatureName>/` + colocated subfolders           |
| Reusable component     | The shared components folder                              |
| `useQuery` hook        | The feature's `queries/` folder                           |
| `useMutation` hook     | The feature's `mutations/` folder                         |
| Any other custom hook  | The feature's `hooks/` folder, or the global hooks folder |
| API endpoint           | The project's API/services folder, one module per resource |
| Styles                 | Sibling stylesheet next to the component                  |

Colocate everything inside the feature folder; promote to a shared folder only when reused across features.
