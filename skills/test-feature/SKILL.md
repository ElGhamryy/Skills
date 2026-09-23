---
name: test-feature
description: Write unit tests for a feature, page, component, or hook in a React frontend using Vitest (or Jest) and React Testing Library. Use whenever the user asks to "write tests for X", "add unit tests", "test this feature/component/hook", wants to increase test coverage for a folder, or just finished building a feature that has no tests yet. Mocks dependencies at the unit's boundary and, unless the project already does otherwise, organizes every feature's tests in a single flat tests/ directory at the feature root.
---

# Test Feature

Stack and code conventions live in the project's `CLAUDE.md` — this skill only covers the testing workflow. Before writing anything, check the project's test setup: runner (Vitest or Jest) and its config (`globals`, `environment: jsdom`, setup files), React Testing Library, `@testing-library/jest-dom`, `@testing-library/user-event`, and whether it uses MSW or mocks modules directly. Follow what is there. Examples below use Vitest (`vi.*`); swap for `jest.*` on Jest.

## Scope

Given a feature/folder, write tests for every component, hook, and util inside it that doesn't already have a test file. Skip files that already have one — extend an existing test file only if the user is asking to cover new behavior that was added to that file, don't duplicate coverage that's already there. If the user names specific file(s), scope to just those.

## Workflow

1. **Inventory.** List the feature's files and check which already have a test file (`*.test.*` / `*.spec.*`) — colocated, in a `__tests__/` folder, or in the feature's `tests/` directory. Build the list of what actually needs tests.
2. **Place the tests.** If the project already has a test layout, follow it. Otherwise, **put all of a feature's tests directly inside one flat `tests/` directory at the feature root — no subfolders.** Every test file is an immediate child of `tests/`, named after the source file it covers — e.g. `hooks/useCreateItem.js` → `tests/useCreateItem.test.js`, and `components/ItemForm/ItemForm.jsx` → `tests/ItemForm.test.jsx`. Don't mirror the feature's folder structure or scatter `__tests__/` folders per subfolder — one flat `tests/` directory keeps the source tree free of test files and makes the full test surface browsable in one place. If two source files share a name, disambiguate the test filename (e.g. prefix with the parent folder) rather than adding subfolders.
3. **Read the source file fully before writing its test.** Identify: its external dependencies (hooks, API calls, store, i18n, toasts/feedback, router), its conditional branches (loading/empty/error/disabled states), and any business logic (validation, limits, calculations) worth exercising with edge-case inputs.
4. **Write the test** — see conventions below.
5. **Run it** with the project's runner, scoped to the file (e.g. `npx vitest run <path>` or `npx jest <path>`). Fix failures by reading the actual component/hook behavior, not by loosening assertions to match whatever happened to run — a passing test that doesn't verify real behavior is worse than no test.
6. **Report** what got covered and, importantly, what didn't (e.g. a branch that's hard to reach, a dependency too entangled to mock cleanly) — don't silently skip coverage gaps.

## Conventions

**Mocking** — mock at the boundary of the unit under test, not deeper. Concretely:
- Mock the custom hooks a component calls (`vi.mock("../hooks/useX")`, then `useX.mockReturnValue(...)` per test/`beforeEach`) rather than mocking React Query or HTTP client internals underneath them.
- Mock the store bindings directly when a component reads/dispatches, e.g. `vi.mock("react-redux", () => ({ useDispatch: () => vi.fn(), useSelector: (selector) => selector(mockState) }))`.
- Mock the translation hook to return the key as the translated string, so assertions can match on keys.
- Mock toast/feedback helpers and any API modules imported directly. If a helper is exported from a large barrel (e.g. a component library that also exports a toast singleton), partially mock it so real components stay in place: `vi.mock("<library>", async (importOriginal) => ({ ...(await importOriginal()), toast: { success: vi.fn(), error: vi.fn() } }))`.
- If the project uses MSW, prefer its handlers over mocking API modules.
- Reset mocks in `beforeEach` with `vi.clearAllMocks()`, then set default `mockReturnValue`s there so individual tests only override what they need to change.

**Querying library components** — many component libraries style with CSS modules or CSS-in-JS, so class names are hashed and are not stable selectors. Query by what the user perceives — `getByRole`, accessible name, `getByLabelText`, visible text — or by an `id`/`data-*` the consumer set. Class-name selectors are also why swapping a component can break a test suite that looked unrelated to it.

**Structure** — nest `describe` blocks by scenario/feature area (e.g. `describe("Update option")`, `describe("Form validation")`), not by function name. Prefer `screen.getByRole`/`getByText`/`getByDisplayValue` queries over test IDs unless the element has no accessible text. Use `fireEvent` for simple interactions where the codebase already does; `userEvent` for more realistic multi-step interactions.

**What to cover, per file type:**

| File type | Cover |
|---|---|
| Component | Renders with expected content/props; each conditional branch (loading, empty, error, disabled, permission-gated); user interactions and their resulting state/UI changes; anything that calls a mutation/handler gets a test asserting it's called with the right arguments. |
| Hook | Initial return shape; state transitions across calls (`renderHook` + `act`); parameter variations; error/edge-case inputs (nulls, boundary values, invalid combinations) and what the hook does with them — don't just test the happy path. |
| Pure function/util | Table of input → expected output, including boundary and invalid inputs. |
| Store slice | Each action/reducer case; selector outputs against representative state shapes. |

## Every test must earn its place

Before writing a test, name the specific regression it would catch. If you can't — if the test would still pass after someone quietly broke the behavior it's supposedly checking — don't write it. A suite full of tests that can't fail for a real reason is worse than a smaller suite that can: it inflates the file, slows CI, and gives false confidence that the feature is covered.

This means **fewer, sharper tests** beats one test per line of code. Don't pad a file to "look thorough" — a component with three meaningful behaviors needs three good tests, not ten.

## Anti-patterns to avoid

- **Don't test implementation details.** Asserting on an internal variable name or exact internal call sequence breaks on harmless refactors and proves nothing about behavior. Assert on rendered output, called side effects, and return values instead.
- **Don't over-mock.** If every dependency down to the DOM is mocked, the test just checks that your mocks return what you told them to. Mock only what's outside the unit under test.
- **Don't test the mock instead of the code.** If a test's only assertion is "the function I mocked was called" and the mock's return value is hardcoded with nothing derived from real logic, it's verifying your test setup, not the component. Assert on what the component *does* with the result.
- **Don't test the framework or library instead of your code.** A test that checks a library `Button` renders, or that a native browser API behaves as documented, is testing someone else's already-tested code. A component library's styling (padding, radius, colour, hover/disabled states) is covered by its own suite, so a feature test asserting it is both redundant and a false alarm waiting to happen. Only test the logic and wiring you actually wrote.

  **Two carve-outs that ARE your code:**
  - **The contract bridge.** Where a library API differs from what the call site used to expect, the adapter is yours. If a component emits `onChange(value)` where the old one emitted `(event, value)`, assert your handler receives the **value** and writes the right state. A handler reading the wrong parameter silently no-ops and no appearance test will catch it.
  - **The rendered element, where accessibility depends on it.** If a typography component picks its tag from `variant`, a metric styled like a heading ships as an `<h2>` — visually identical to the `<p>` it should be, which is exactly why review misses it. Where a tag or ARIA state carries meaning, assert it: `expect(screen.getByText("42").tagName).toBe("P")`, or `getByRole("button", { expanded: false })` over a chevron's rotation.
- **Don't write near-duplicate tests.** Multiple tests that exercise the exact same branch with cosmetically different input (e.g. `"foo"` vs `"bar"` when neither value affects control flow) add maintenance cost with no added protection. One representative case per distinct behavior is enough; only add more when different inputs genuinely take different paths.
- **Don't assert the trivially true.** `expect(component).toBeDefined()` or checking a static heading renders on a page with no conditional logic doesn't protect against any realistic regression — skip it unless that render path has actual branching behind it.
- **Don't skip loading/empty/error states.** These are exactly the states that regress silently in review; per [implement-feature](../implement-feature/SKILL.md)'s verification step, they matter as much as the happy path. (This one cuts the other way from padding — real behavior, not filler.)
- **A structural CSS invariant can be guarded in a test, and sometimes should be.** jsdom applies no stylesheet, so a rendered-appearance assertion is impossible — but reading the stylesheet as text and asserting on it works. Reach for it only where a wrong selector is silent and expensive: e.g. an accordion that styled its chevron with a bare descendant selector (`.itemOpen .chevron`), so every nested accordion's chevron mirrored its ancestor's state instead of its own. A guard asserting no state selector uses a bare descendant combinator fails if that regresses. Don't use this to pin ordinary values — a test asserting `padding: 8px` just duplicates the stylesheet and breaks on every legitimate change.
- **Don't write snapshot-only tests.** A snapshot that nobody reads on failure just gets blindly updated — prefer explicit assertions on the specific content/behavior that matters.
