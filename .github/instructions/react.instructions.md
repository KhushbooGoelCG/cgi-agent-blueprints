---
description: 'Guidelines for building react+typescript web applications'
applyTo: "src/*-app/**"
---

# Project Instructions: React + Vite + TypeScript

You are an expert Senior Frontend Engineer. Follow these rules for all code generation.

## 1. Project Structure
Strictly adhere to this folder structure:
- `src/assets/`: Static files (images, icons).
- `src/components/ui/`: Low-level UI (Shadcn/Radix).
- `src/components/common/`: Shared reusable components.
- `src/features/[feature-name]/`: Domain-driven logic (components, hooks, services).
- `src/hooks/`: Global custom hooks.
- `src/layouts/`: Page wrappers.
- `src/pages/`: Route views/screens.
- `src/services/`: API/External integrations.
- `src/store/`: Global state (Zustand/Redux).
- `src/types/`: Global TypeScript interfaces.
- `src/utils/`: Helper functions.
- `src/lib/`: Third-party config/setup.

## 2. Tech Stack & Patterns
- **Framework**: React 19
- **Build Tool**: Vite.
- **Styling**: Vanilla CSS (component-scoped CSS files + global styles).
- **Routing:** React Router v7
- **State Management**: [Specify your choice, e.g., Zustand].
- **Data Fetching**: [Specify your choice, e.g., TanStack Query].
- **HTTP client:** Native Fetch API (configured wrapper in `src/lib/http.ts`)
- **Testing:** Vitest + React Testing Library

## 3. Naming Conventions

| Construct | Convention | Example |
|-----------|-----------|---------|
| Components | PascalCase | `CustomerCard.tsx` |
| Pages | PascalCase + `Page` suffix | `CustomerPage.tsx` |
| Hooks | camelCase + `use` prefix | `useCustomer.ts` |
| Services | camelCase + `Service` suffix | `customerService.ts` |
| Context | PascalCase + `Context` suffix | `AuthContext.tsx` |
| Store | camelCase + `Store` suffix | `customerStore.ts` |
| Types/Interfaces | PascalCase | `Customer`, `ApiResponse<T>` |
| Type files | camelCase + `.types.ts` | `customer.types.ts` |
| Util functions | camelCase | `formatDate.ts` |
| CSS files | Same name as component + `.css` | `CustomerCard.css` |
| CSS classes | kebab-case, prefixed with component name | `.customer-card-header` |
| Env variables | `VITE_` prefix, SCREAMING_SNAKE_CASE | `VITE_API_URL` |

## 4. TypeScript Rules
- Use `interface` for object definitions, `type` for unions/aliases.
- Avoid `any` at all costs. Use `unknown` if necessary.
- Use Discriminated Unions for complex states.
- Export types from `src/types/` for global use.

## 5. Component Guidelines
- Use **Arrow Functions** for components.
- Use **Named Exports** (no default exports for components).
- Follow the "Feature-First" pattern: if a component is specific to one domain, put it in `features/`.
- Prop types must be defined explicitly above the component.
- Do not put business logic inside components — extract to hooks or services.

## 6. Implementation Details
- Use Path Aliases: Use `@/` for `src/` (e.g., `@/components/Button`).
- Performance: Use `useMemo` and `useCallback` only when necessary for expensive calculations or preventing re-renders of heavy children.
- Data: Always handle Loading and Error states in UI components.

## 7. Hooks Rules
- One hook per file under `src/hooks/`.
- Hooks that call the API must use **React Query** (`useQuery`, `useMutation`).
- Never fetch data directly inside a component — always delegate to a hook.
- Always handle loading, error, and empty states in the hook's return value.

## 8. Services Rules
- All API calls live in `src/services/` — never inline `fetch` calls in
  components or hooks.
- Always use the configured fetch wrapper from `src/lib/http.ts` — never
  call `fetch()` directly in services.
- Always check `response.ok` and throw a typed error if the request failed —
  never assume a response is successful.
- Functions are async and return typed promises.

```ts
// ✅ correct
import type { Customer } from '@/types/customer.types';

// ❌ avoid
const data: any = await fetchCustomer();
```

## 9. Environment Variables
- All env vars must be prefixed with `VITE_` to be exposed to the client.
- Access via `import.meta.env.VITE_*` — never use `process.env`.
- Never hardcode API URLs or secrets in source files.

## 10. Path Aliases
Use `@/` as the alias for `src/`. Always prefer aliases over relative paths
that go up more than one level.

```ts
// ✅ correct
import { Button } from '@/components/Button';

// ❌ avoid
import { Button } from '../../../components/Button';
```

## 11. CSS & Styling Rules
- Use **Vanilla CSS** — no CSS frameworks or preprocessors.
- Each component/page must have its own CSS file (e.g., `CustomerCard.tsx` → `CustomerCard.css`).
- Import the CSS file at the top of the component: `import './CustomerCard.css';`
- Global/shared styles live in `src/index.css` or `src/styles/global.css`.
- Use **kebab-case** for class names, prefixed with the component name to avoid collisions (e.g., `.customer-card-wrapper`).
- No inline styles — use CSS classes only.
- Keep CSS files co-located with their component in the same folder.

```tsx
// ✅ correct
import './CustomerCard.css';

export const CustomerCard = () => (
  <div className="customer-card">
    <h2 className="customer-card-title">...</h2>
  </div>
);
```

## 12. General Rules
- Never use `// @ts-ignore` — fix the type error instead.
- No `console.log` in committed code — use a proper logger utility if needed.
- No inline styles — use CSS classes only.
- Components must not exceed 150 lines — split into sub-components if larger.
- Do not install new packages without noting it explicitly in the response.