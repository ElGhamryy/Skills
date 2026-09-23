---
name: implement-feature-typescript
description: Plan and implement new features in a React frontend using TypeScript for all newly created files, in a codebase that is still partly JavaScript. ONLY use this skill when the user explicitly states the feature will use TypeScript (e.g. "implement X in TypeScript", "this feature will use TypeScript", "new TS page/component/hook"). For every other feature request use the `implement-feature` skill instead. New files are .ts/.tsx; existing and shared JS files stay JavaScript and are never converted.
---

# Implement Feature (TypeScript)

Extends the `implement-feature` skill — **read its `SKILL.md` first** (`../implement-feature/SKILL.md`). Its workflow, clarifying questions, component conventions, file organization, data flow, and placement defaults all apply unless overridden below. This file adds only the TypeScript rules.

Before writing TS, read the project's own TypeScript setup: `tsconfig.json`, the ESLint config, and any TypeScript doc the repo keeps. **Trust the config files over the docs** — if a doc says `strict: true` and `tsconfig.json` says otherwise, the config is what `tsc` enforces.

## What's TS, what stays JS

- New, self-contained files only — `.tsx` for JSX, `.ts` for everything else.
- Existing `.js`/`.jsx` stay JavaScript: never rename, migrate, or retro-type them. A new `.tsx` importing old `.js` modules is the normal case (`allowJs` is on).
- If the change belongs in an existing JS file, write JavaScript — don't spin off a new TS file to dodge that. If the task only works cleanly by converting a file, stop and ask; migration is opt-in.
- Follow the project's naming for entry files and barrels. If a page entry and a barrel could both be called `index`, keep them apart (e.g. `Index.tsx` for the page, `index.ts` for barrels) so they never collide under `forceConsistentCasingInFileNames`.
- Feature types go in a `types/` folder — see [Where types live](#where-types-live).
- Parts of the app the project keeps in JS (often the store and router) stay JS.
- A **new** endpoint group is TypeScript: an endpoints module plus sibling `<area>.types.ts` files. Existing JS service files stay JS — a JS HTTP client instance types fine from a `.ts` caller, so `api.get<Response>(url)` is all a typed endpoint needs.
- At **Plan**, name every legacy JS module the new code imports. Do **not** plan types for them — see below.

## Typed libraries are not legacy JS

If the project's component library or design system is written in TypeScript (or ships types), none of the legacy rules below apply to it.

- **Never `@ts-expect-error` a typed library call site.** If `tsc` complains, you have a real type error: a wrong prop name, a wrong handler signature, or a `variant`/`size` outside the union. Read the component's types and fix the call.
- Import prop types by name where you need them (`import type { TableColumn } from "<library>"`).
- Prop unions are the spec. `size`, `variant` and friends are string-literal unions — let the compiler tell you what exists, and don't widen a param to `string` to dodge it.
- Legacy shared JS components **do** need the handling below. For new work reach for the typed library equivalent first, and only render the legacy one where the library genuinely can't do the job, suppressing at the usage site with a reason.

## Legacy JS is off-limits

The TypeScript work is scoped **strictly to purely new files**. Legacy JS is consumed as-is and never typed. This is absolute — there is no "just this once" case, and no amount of `tsc` noise justifies breaking it.

Never do any of the following for an existing `.js`/`.jsx` module:

- Declare, wrap, shim, or re-export it with a prop/return type — **no boundary file of typed re-exports**, no `as unknown as ComponentType<Props>`, no typed adapter or facade.
- Write a `.d.ts` that merely restates its props — see [the `.d.ts` test](#the-dts-test) for the one narrow case that is allowed.
- Edit it — including "harmless" fixes like correcting a wrong JSDoc `@param` that is making inference misbehave.
- Restructure new code around its inferred signature (spreading props through a variable, passing dummy `startIcon={null}` / `className=""` / `children={null}` values) just to satisfy the compiler.

Import it plainly and use it.

### The consequence, and the only sanctioned handling

With `allowJs` on, TS **does** read legacy JS and infers its props. Every param destructured without a default becomes **required**, and JSDoc is trusted — so a wrong `@param {array}` types a whole options object as `any[]`. New `.tsx` that renders a legacy component therefore produces errors that are not the new code's fault and cannot be fixed under this rule.

Suppress them at the usage site with `@ts-expect-error` **and a reason**, one per reported position:

```tsx
{
  /* @ts-expect-error legacy JS component — TS infers all of its props as required */
}
<LegacyTable tableHeader={header} tableBody={body} loading={isLoading} />;
```

Placement must match where `tsc` reports the error, which differs by error kind:

- **Whole-element** errors (missing required props, `TS2739`/`TS2740`/`TS2741` on the tag) → JSX comment `{/* @ts-expect-error … */}` on the line before the **opening tag**.
- **Single-attribute** errors (a nested object literal missing a key) → `// @ts-expect-error …` on the line before that **attribute**.
- **Hook/function calls** → `// @ts-expect-error …` before the offending argument or property line.

Run `tsc` to find the real position instead of guessing — a directive on the wrong line fails twice, as the original error plus `TS2578 Unused '@ts-expect-error' directive`.

Do not chase these errors by adding props. Fixing one required prop just surfaces the next, and the fix may be blocked outright by a lint rule (`react/no-children-prop` rejects `children={null}`).

### The `.d.ts` test

A declaration file for a legacy JS module is allowed **only when it states a signature inference cannot produce and a reader gains real information** — generic hook overloads, a callable translation signature. Two tests, both must pass:

1. Does it say something `tsc` could not infer on its own?
2. Would a reader who opens it learn what the module actually accepts?

**Banned by name:** a declaration whose props are `unknown` with an `[key: string]: unknown` index signature. It accepts any prop object including typos, nothing verifies it still matches the `.js`, and — unlike a suppression — it never expires. It silences errors while looking like typing, which is worse than the error.

**Escalation.** Four or more `@ts-expect-error` sites for the same legacy component is the signal to replace that component with the typed library equivalent, not to declare it.

## Strictness

Never change `tsconfig.json`, the ESLint config, or the bundler config for a feature.

Check what the config actually enforces. If `strict` is off, `tsc` never flags a missed null — handle nullability by writing `| undefined` and `?.`, not by turning on a stricter flag. If lint rules like `no-explicit-any` are off, re-arm them for your files via `--rule` (see [Verify](#verify)).

- **No `any`** — at an untyped boundary use `unknown` and narrow before use.
- **No `!`** — model optionality with `| undefined` and `?.`.
- **No `@ts-ignore`** — if unavoidable, `@ts-expect-error` with a reason.
- Explicit props, params, and non-trivial return types. No `React.FC` — type the props parameter.

## Where types live

Two rules decide every type's home: **the wire format belongs to the service, the UI shape belongs to the feature**, and **imports only ever point from a feature into the services folder, never back**. A service file that imports from a page/feature is a defect — move the type, don't work around it.

```
<services>/<domain>/
  <domain>.ts                 # the endpoints
  <area>.types.ts             # request params, response bodies, payloads
  <otherArea>.types.ts        # split by endpoint group once one file passes ~150 lines

<pages>/<Feature>/
  types/<feature>.types.ts    # UI shapes; re-exports the wire types its consumers need
  <subFeature>/types/<subFeature>.types.ts
```

- **Name a types file by its area, never `types.ts`.** A feature always uses a `types/` folder with `<area>.types.ts` inside it; only a single component's own props file may be a bare `types.ts` beside the component. Splitting by area keeps a 240-line god-file from forming.
- **Re-export rather than reach across.** A feature types module re-exports the wire types its own consumers use (`export type { Thing } from "<services>/…"`), so components import one path and the service stays the single definition.
- Props used by one component live in that component's file → shapes shared across the feature go in the feature's `types/` folder → promote out of the feature only when a second feature needs them.

## Typing

- **One consumer means it stays with that consumer.** This governs helpers, constants, option lists and local types alike. Count the importers before creating or adding to a shared file: `helpers.ts` holds only what **two or more** files import. Single-consumer code lives in the file that uses it even when it runs 25 lines, and a one-line `map` is inlined at its call site rather than named. A `helpers.ts` of thin single-use wrappers is the failure mode to avoid.
- Do not transform API data on the way to the UI. Render the field the endpoint returned; no `String()`/`Number()` round-trips to suit a component, no `||` fallback that invents a display value out of another field (an id standing in for a missing name), no composite stitched from two fields unless the design asks for it. If a field is absent, render nothing.
- Type the **data** crossing a JS boundary — never the JS module itself (see [Legacy JS is off-limits](#legacy-js-is-off-limits)):
  - **API responses** — type them on the endpoint, not the hook: `api.get<ThingsResponse>(url)`. The query hook then needs no annotation at all.
  - **JS store** — a selector off an untyped store yields `any`; type the selector's state parameter with the slice shape you declared.
- `import type` / `export type` for type-only imports and re-exports (`isolatedModules` requires the explicit form).
- `type` over `interface` unless you need declaration merging. No `I` prefix. String-literal unions over `enum`.
- **Generic type parameters are `T` + a descriptive name**, read like a variable: `TItem`, `TUserId`, `TResponse`. Never a bare `T`, `U` or `K`. This applies to type parameters only; type aliases keep plain names (`User`, not `TUser`).

  ```ts
  const pickById = <TItem extends { id: string }>(items: TItem[], id: string): TItem | undefined =>
    items?.find?.((item) => item?.id === id);
  ```
- **`satisfies` only when something consumes the narrower type.** Both forms run the same check, including excess properties and a missing key in a `Record<Union, _>` — the only difference is the resulting type: `const X: T = {…}` gives `X` the type `T`, `const X = {…} satisfies T` leaves `X` with its inferred type. So `satisfies` pays only where the inferred type is _strictly_ more specific **and** a call site reads that specificity. Two cases that qualify:
  - `T` has an index signature — `Record<string, Cfg>` throws the keys away, `satisfies Record<string, Cfg>` keeps them, so `keyof typeof X` works and an unknown key is an error rather than a silent `Cfg`.
  - The literal set is a strict subset of `T` and something is derived from it (`type Key = typeof X[number]["value"]`).

  Where the value already covers the whole union — an options array listing every member, a config map keyed by the full union — the two forms produce the _identical_ type and the annotation reads better, because the contract is stated before the value instead of after it. Don't retrofit `satisfies` onto those; verify with a probe (`const p: never = X[0].value` prints the inferred type) rather than assuming it narrowed something.

- `ReactNode` for children; `ChangeEvent<HTMLInputElement>` / `MouseEvent<HTMLButtonElement>` for handler params.
- No `.d.ts` for stylesheets, images, or other assets if the bundler's client types already declare them (e.g. `vite/client`).
- If `tsconfig.json` declares no test globals, test files import `describe`/`it`/`expect`/`vi` from the test runner and the `jest-dom` matchers explicitly — implicit globals fail typecheck.

## Server state

Defaults for TanStack Query (or similar) when the project has no pattern of its own:

- **One `queryKeys.ts` at the feature root, exporting a key factory** — not loose `SCREAMING_CASE` strings. Nest it so a family can be invalidated by its prefix, and give every builder the params type its endpoint already takes. A builder typed `params?: unknown` throws away the only check worth having.

```ts
export const featureKeys = {
  all: [ROOT] as const,
  items: {
    all: () => [ROOT, "items"] as const, // invalidates the whole family
    lists: () => [ROOT, "items", "list"] as const, // every filter/page variant
    list: (params: ListParams) => [ROOT, "items", "list", params] as const,
    summary: () => [ROOT, "items", "summary"] as const,
  },
} as const;

export const featureMutationKeys = {
  syncItems: () => [ROOT, "sync"] as const,
} as const;
```

Flat string keys force every write to enumerate the reads it invalidates by hand, and that list silently goes stale the day a fourth query is added. A prefix cannot.

- **Three hook folders, split by primitive** — `queries/`, `mutations/`, `hooks/`, exactly as in `implement-feature`. The primitive decides, never the name. No barrel files.
- **A `mappers/` module only when a response is actually reshaped.** `select: (res) => res?.data` is a passthrough and belongs inline. A folder of one-line wrappers is the failure mode, not the goal.

## Translation keys

Same rule as `implement-feature`: never inline a translation key string; every key is a `SCREAMING_CASE` constant, scoped by the one-consumer rule. TypeScript additions:

- **Type a key map `Record<string, string>`, not a wider shape**, and key it off the same constants the rest of the feature uses (e.g. backend error-code constants → key constants), so neither side is a loose literal.
- **Why it matters more in TS:** if the translation function is untyped JS taking a plain `string`, TypeScript can never check a key. A typo in an inline key ships as the raw dotted path rendered to the user; behind a named constant it is an undefined identifier the compiler catches.

## Typed test doubles

Tests are held to the same bar as production code.

- `vi.mocked(fn)` (or `jest.mocked(fn)`) to reach a mocked function. Never `(Service.fn as any).mockResolvedValue(…)`.
- `importOriginal<typeof import("<module>")>()` when partially mocking a module, so the untouched exports keep their types.
- **A fixture factory, not a cast.** Put `makeThing(overrides: Partial<Thing> = {}): Thing` in `tests/fixtures.ts` and build test data from it. `as unknown as Thing` is banned in tests exactly as it is in source — it is the reason a renamed field breaks nothing until runtime.
- Assert against the key factory (`featureKeys.items.all()`), never against a hardcoded key string.
- A test that deliberately feeds an invalid value documents it in a comment on that line. Reach for `@ts-expect-error` only if the repo config actually reports an error there — under `strict: false` a `null` passed where `boolean` is declared does not, and the directive becomes `TS2578`.

## Patterns worth naming

- **`Record<Union, Config>` over an `if`/`switch` ladder.** A fixed set of variants becomes a config map keyed by the string-literal union, mapped over in the render. Add a member to the union and the map fails to compile; a ladder silently falls through.
- **A discriminated union when state has two genuinely different shapes.** Discriminate on a literal `mode`/`kind` field rather than carrying both shapes' fields as optional.
- **Hoist a prop group once three or more components take it.** Declare it once and intersect: `type FooProps = SharedChromeProps & { … }`.
- **Bind every form schema to its form-values type.** With Yup the strong form is `Yup.ObjectSchema<FormValues>`; where the schema uses `when()` branches or an untyped `.test()` that annotation will not compile, and the working form is a key-coverage guard:

```ts
const buildSchemaShape = (mode: Mode): Record<keyof FormValues, Yup.AnySchema> => ({ … });
export const schema = (mode: Mode) => Yup.object(buildSchemaShape(mode));
```

A field with nothing to validate still gets an entry (`Yup.mixed<TValue>()`) so the shape stays exhaustive. Type the form hook with the same values type (`useFormik<FormValues>`, `useForm<FormValues>`). **`as unknown as Yup.ObjectSchema<…>` is banned** — the cast removes the exact check that makes the annotation worth writing.

- **A TS refactor is not a library migration.** Say which one you are doing at Plan. Converting a file to `.tsx` while it keeps imports from a library the project is moving away from is declared debt that goes in the plan — never something that ships quietly inside a typed file.
- **Storybook stories are typed too**: `Meta<typeof Component>` and `StoryObj<typeof meta>`, or the args are unchecked.

## Verify

Judge these commands only on the files this feature touched. Pre-existing errors elsewhere are not this feature's job — don't fix them, don't report them, don't let them block finishing.

- Lint — only your files, not the whole project; `--rule` re-arms what the repo config may disable:
  ```
  npx eslint --rule '{"@typescript-eslint/no-explicit-any":"error","@typescript-eslint/no-non-null-assertion":"error"}' <changed files or the feature folder>
  ```
- Types — run the repo's own config, which is what CI enforces:

  ```
  npx tsc --noEmit
  ```

  Whole-project by design; it can't be file-scoped without losing the tsconfig. Read only the diagnostics whose paths are yours.

  **If the repo is not `strict`, do not gate on `npx tsc --noEmit --strict`.** It reports errors in new files that come from legacy JS inference and cannot be fixed under this skill's rules — `actions = []` in a legacy component infers as `never[]`, so any array passed to it is unassignable. Worse, a `@ts-expect-error` added to silence a `--strict`-only error becomes `TS2578` under the repo config, so satisfying `--strict` actively breaks the real gate.

- Stylesheets — every new stylesheet compiles (e.g. `npx sass --no-source-map <file.scss> /dev/null` for SCSS).

- **Nullability** — when `strict` is off, check the changed files by hand for the three things it would have caught:
  1. every value read off a paginated or optional response is handled as possibly absent before use (`?? 0`, not `a?.b > 1`, which compares against `undefined`);
  2. every array index and `.find()` result is treated as possibly missing;
  3. no comparison or index built from an optional value (`MAP[maybeUndefined]`).

  Running `npx tsc --noEmit --strict` and reading only your own paths is a useful way to _find_ these — but it is a reading aid, not a gate, and **a strict-only diagnostic is not on its own a reason to change working code.** Fix it when the value really can be absent at runtime; leave it when a runtime guard the compiler cannot see already covers it (a query's `enabled`, an early return, a route that cannot render without the param). Never add a `@ts-expect-error` for a strict-only error.

- **Dependency direction** — grep the new services folder for imports from pages/features; it returns nothing.

Done when all of these are clean, the feature's own tests pass, and `git status` shows adds — not renames.
