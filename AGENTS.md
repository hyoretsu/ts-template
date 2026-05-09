# Global Guidelines

This project uses Serena. Lookup `SERENA.md`/`@SERENA.md` for the full protocol.

- Never put multiple components in the same file unless the file is intentionally using the Composition pattern. When a parent component needs private child components, turn it into a folder with `index.tsx`, move each child component to its own file, and place shared local types in `types.ts`.
- Always add ShadCN components through the ShadCN CLI.

## Component Architecture (Frontend)

- **Simple, componentized components.** Break out parts that make sense into their own components — avoid bloated components with multiple responsibilities.
- **Maximize reuse — design-system mindset.** Aggressively extract primitives (Container, Section, IconBadge, CheckList, BrandMark, SectionHeading, etc.) whenever a layout/presentation pattern repeats, even within a single page. Generic primitives go in [frontend/src/components/ui/](frontend/src/components/ui/); domain composites go in `frontend/src/components/<feature>/`. When in doubt, extract.
- **Component file naming in CamelCase (PascalCase).** Component files use PascalCase: `AuthTabs.tsx`, `LoginForm.tsx`, `FormField.tsx`. Files that are not components (utils, hooks, configs) stay in kebab-case.
- **Page component co-location.** If a component is only used by one page (or a single parent component), place it in a `components/` folder adjacent to the file that uses it. Always create an `index.ts` barrel inside that folder. Example: `app/auth/components/AuthTabs.tsx` + `app/auth/components/index.ts`.
- **Fetch data in the deepest possible component.** The request belongs in the component that actually consumes the data. Only lift it to the parent if another sibling also needs the same data.
- **Server Components by default.** Keep `"use client"` at the smallest possible scope — only add it when there is state, an effect, an event handler, or a browser API.
- **Lazzu exception:** the frontend uses Vinext with the intention of `output: "export"` and Tauri in the future — in this mode, the bundle must be 100% pre-rendered client-side, so RSC does not apply and fetching happens in the browser (React Query/SWR/etc.). Treat everything here as a client component, but keep the discipline of small components, design system, and leaf-level fetching.
- **No `React.` namespace.** Never use `import * as React from "react"` or qualify types/hooks with `React.X`. Import everything by name: `import { forwardRef, useState, type HTMLAttributes, type ReactNode } from "react"`. `FormEvent`/`FormEventHandler` are deprecated in current typings — prefer an inline `onSubmit` handler so the type is inferred.

## Backend DTOs (Elysia / TypeBox)

**DTOs are always TypeBox schemas, never TypeScript interfaces.** The type is derived with `typeof XxxDTO.static`:

```ts
// ✅ correct
export const CreateReceiptDTO = t.Object({
    ...CreateReceiptBody.properties,
    companyId: t.Optional(t.String()),
    filename: t.String(),
    userId: t.String(),
});
export type CreateReceiptDTO = typeof CreateReceiptDTO.static;

// ❌ avoid
export interface CreateReceiptDTO {
    companyId?: string;
    filename: string;
    // ...
}
```

To compose DTOs, use a single `t.Object` with a `.properties` spread — **never** `t.Intersect`:

```ts
// ✅ correct
export const ListPriceAlertsDTO = t.Object({ ...ListPriceAlertsQuery.properties, userId: t.String() });

// ❌ avoid
export const ListPriceAlertsDTO = t.Intersect([ListPriceAlertsQuery, t.Object({ userId: t.String() })]);
```

`t.Intersect` produces poorly resolved runtime types during validation; the `.properties` spread produces a correct flat schema.

### DTO Naming vs. Elysia Option

- **Return DTO**: name it `XxxReturn` or `XxxResult` — it reflects the use-case domain, without knowledge of HTTP infrastructure.
- **Elysia controller**: pass the DTO under the `response:` key, **never** `result:`.

```ts
// ✅ correct — the Elysia key is "response"
.get("/revenue", handler, { response: GetRevenueReturn })

// ❌ wrong — "result" is not recognized by Elysia; it generates `any` types in Kubb
.get("/revenue", handler, { result: GetRevenueReturn })
```

Using `result:` makes Elysia ignore the response typing, which makes Kubb generate `XxxQueryResponse = any` in the generated SDK.

## Frontend SDK — Mandatory Rule

**Never edit files inside `frontend/src/lib/api/generated/`.** They are completely overwritten on every regeneration.

Correct flow when adding or changing an endpoint:
1. Change the backend (controller/DTO).
2. `bun run export` in the backend → updates `backend/generated/openapi.json` and `generateOpenApi.ts`.
3. `bun run generate` in the frontend → regenerates the SDK in `generated/`.

If you need a hook for an endpoint that is not yet available on the production server, place it in `frontend/src/lib/api/` (outside `generated/`) and re-export it through `frontend/src/lib/api/index.ts`.

## i18n — Mandatory Rule

The translation files in [frontend/src/i18n/locales/](frontend/src/i18n/locales/) **must remain synchronized**. Today they are `pt-BR.json` and `pt-PT.json`.

**Every time you add, rename, or remove a key, apply the change to ALL locale files in the same operation.** Never leave one locale behind another, even if the final translation still needs review — in that case, use the pt-BR string as a placeholder and leave a comment in the PR, but ensure the key structure is identical across files.

- `pt-BR` is the fallback and the ground truth for types (see [frontend/src/i18n/types.d.ts](frontend/src/i18n/types.d.ts)).
- Keys inside JSON files follow alphabetical order (linter rule).
- When introducing a new namespace, register it in [frontend/src/i18n/config.ts](frontend/src/i18n/config.ts) and in `types.d.ts`.
