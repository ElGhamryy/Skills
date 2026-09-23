# Frontend Conventions

## Code Style

- Always use optional chaining `?.` for potentially-undefined access and calls (e.g. `list?.map?.()`, `str?.toLowerCase?.()`).
- Use arrow function expressions assigned to `const` (e.g. `const toggle = () => ...`), not function declarations.
- Name event handlers with the `handleX` prefix (e.g. `handleClick`), not `onX`.
- Do not rename props when passing them through.
- Use descriptive variable and function names.
- Keep comments to one concise line stating intent — not multi-line mechanism explanations.
- Apply KISS, DRY, and SOLID. Prefer simple, readable code; do not over-engineer.
- A constant earns a place in a shared `constants` file only if it is large (a long list, a config map, a lookup table) or used by more than one file. A small single-use value stays in the file that uses it, declared above the component or hook.

## Styling

- Use single-dash kebab-case class names (`class-name`).
- Do not use underscores (`class__name`) or double-dashes (`class--name`) in class names.
- NEVER use `!important`. Win the cascade with your own unique class names, or restructure the markup so the override is not needed.
- **Directional layout uses logical properties, not a mirrored copy.** Write `padding-inline`, `margin-inline-start/end`, `border-inline-start`, `inset-inline-*` — never `left`/`right`/`margin-left`/`padding-right` followed by an RTL block that flips them. One rule then covers both directions. The one sanctioned exception is `transform: translate(±N, …)` for an animated thumb, which does need an explicit `[dir="rtl"]` rule with the value negated.

## Components & Accessibility

- Do not duplicate JSX for repeated elements. Declare an array of config objects (icon, label, value, id key, ...) and map over it in the render.
- NEVER use the array index as a React `key`, and never a value derived from it (``key={`${index}-${list?.length}`}``). An index key binds a component's state to a position, so inserting, removing, or reordering leaves the previous item's state — and any state inside child components — attached to the wrong data. Key on stable identity already present in the data: the item's `id` when it has one, otherwise its own value. Do not generate ids (`uuid` and friends) just to satisfy a key. If an item genuinely carries neither — no `id`, and a value that mutates while editing — document the exception in a comment on the `key` line.
- Add accessibility features on interactive elements: `tabIndex`, `aria-label`, and keyboard handlers where relevant.

## Edge Cases

### Paginated lists

- Deleting the last row on a page must step back one page. Never leave the user on an emptied or out-of-range page. Detect it from the current page's rows, not the total count: `rows?.length === 1 && page > 1`. On page 1 do nothing extra — the refetch renders the empty state. Out-of-range is not cosmetic: the API may reject it and the page can render a not-found state. This applies to bulk delete too — if every row on the page went, step back.
- When the page lives in a URL param, navigate to `page - 1` and let the list refetch from the new param:

```js
if (rows?.length === 1) {
  const currentPage = Number(searchParams?.get("page"));
  if (currentPage > 1) {
    searchParams?.set("page", currentPage - 1);
    navigate({ search: searchParams?.toString() });
  }
}
```

- When the page lives in React state and you use TanStack Query, invalidate but skip the refetch of the page you are leaving — `refetchType: "none"` matters, without it the out-of-range page fires one wasted request before the clamp lands:

```ts
const isPageEmptied = rows?.length === 1 && page > 1;
queryClient.invalidateQueries({
  queryKey: [LIST_QUERY_KEY],
  refetchType: isPageEmptied ? "none" : "active",
});
if (isPageEmptied) setPage(page - 1);
```

- A search, sort or filter change resets the page to 1.
