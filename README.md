# User-Tailored Inventory

The **second** application that adopts the [`user-tailored-ui`](https://github.com/taylor-made-labs/user-tailored-ui)
foundation, for the Kiroween 2026 **Skeleton Crew** category. It is the "flywheel"
proof: the SAME foundation, adopted again through its public ports only, re-using
everything the foundation owns and costing dramatically less than the first app
(the [Planner](https://github.com/taylor-made-labs/user-tailored-planner)).

A stock-tracking app. You **create, edit, adjust the quantity of, and delete items**
(name, quantity, location, low-stock threshold), then use the **in-app agent** to
customize the catalog into a **low-stock focus view** — hiding healthy items while
preserving every item's identity, the required quantity-adjust control, and the
creator's constraints.

## What makes it distinct from the Planner

| | Planner | Inventory |
| --- | --- | --- |
| domain entity | task (title, order, completed) | item (quantity, location, threshold) |
| signature op | reorder / complete | **adjustQty** (signed delta, clamped at 0) |
| customization story | focused daily **agenda** | **low-stock focus** (hide healthy items) |
| required control | complete-task | **adjust-qty** |
| allowed components | board / agenda / task-card | **grid / list / low-stock-view / item-card** |
| allowed actions | task.create/edit/reorder/complete | **item.create/edit/adjustQty/delete** |

Everything shared — the customization boundary, the validator, the orchestrator, the
policy/preview/apply/reopen/undo flow, the labeled test-double adapters — comes from
the foundation unchanged.

## Architecture (hexagonal)

- **Pure domain** (`src/domain/item.ts`): the `Item` entity and create/edit/adjustQty/
  delete operations. No React, no model, no storage. Sole authority on item identity
  stability and quantity clamping.
- **Foundation, behind one port** (`src/ports/foundation.ts`): all foundation types,
  the orchestrator, the ports and the labeled test-double adapters are imported from
  here and here only — never from `src/foundation/**` directly.
- **Customization** (`src/customization/*`): the Inventory `CreatorPolicy`, the default
  grid view and the low-stock-focus transform + four violation fixtures (all data,
  never code), and the service that wires the foundation orchestrator.
- **Adapters** (`src/adapters/*`): the IndexedDB item store (browser) and an in-memory
  double (tests), plus the TailoredUI capability adapter resolving components to
  data-only render descriptors.
- **UI** (`src/ui/*`, `src/App.tsx`, `src/main.tsx`): React 18 + Vite. Item CRUD + the
  adjust-qty control, and an agent panel (utterance → preview → apply/undo) with inline
  rejection explanations and an honest SIMULATED label on test-double output.

### How the foundation is consumed

`@taylor-made-labs/user-tailored-ui` is **not published to npm yet**, and CI runs
`npm ci` in an isolated runner with no sibling checkout — so a `file:` dependency
cannot resolve there. We therefore **vendor only the foundation's public entrypoint**
into `src/foundation/` (a faithful copy of the foundation's own `src/index.ts` and the
modules it re-exports) and re-export it through the single port
`src/ports/foundation.ts`. When the foundation publishes, that one file becomes
`export * from '@taylor-made-labs/user-tailored-ui';`, `src/foundation/` is deleted,
and no other app code changes. That is the point of the port seam.

## Policy invariants (enforced by the foundation validator)

- **allowedActions**: `item.create`, `item.edit`, `item.adjustQty`, `item.delete`
- **requiredControls**: `adjust-qty` must stay present and operable in every view — no
  customization can strand stock you can no longer restock.
- **allowedComponents**: `grid`, `list`, `low-stock-view`, `item-card`
- **tokenRules**: `fontScale` number in `[1, 2]` (accessibility floor ≥ 1), `density`
  enum `{compact, comfortable}`, `lowStockOnly` boolean (drives the focus filter).

The four violation fixtures each have a passing and a violating test:
remove the adjust control → reject; `fontScale` 2.5 → reject; a non-approved component
(`kanban`) → reject; a disallowed action (`item.archive`) → reject.

## Storage

Business items persist in **IndexedDB only** (`inventory-business` DB), kept strictly
separate from customization history. Never localStorage, never chrome.storage.

## Develop

```bash
npm ci
npm run typecheck   # tsc --noEmit (strict)
npm run lint        # eslint, zero warnings
npm test            # vitest (deterministic, offline)
npm run build       # tsc + vite build
npm run dev         # local dev server
npx tsx src/demo.ts # headless end-to-end demo (no browser, no network)
```

CI (`.github/workflows/ci.yml`) runs `npm ci → typecheck → lint → test → build` and is
the source of truth for "it runs". No secrets, no network beyond npm: the default
inference adapter is the labeled offline test-double.

## License

Dual-licensed `MIT OR Apache-2.0`. See `LICENSE-MIT`, `LICENSE-APACHE`, `NOTICE`.
