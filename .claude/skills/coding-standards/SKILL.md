---
name: coding-standards
description: "Invoke when implementing features, writing code, or reviewing code quality. Contains project-specific naming conventions, error handling patterns, import style, and deviations from standard practices."
---

# Coding Standards

## Detected
- Language: TypeScript with Next.js (1670 source files)
- Functions: PascalCase (53%, 1518 sampled)
- Classes: PascalCase (29%)
- Files: PascalCase (79%, 750 sampled)
- Imports: path aliases (`@/`) (near 100%)
- Indentation: spaces, 2 wide
- Error handling: exceptions (nextjs)
- Data fetching: swr
- State management: jotai
- Form handling: react-hook-form
- UI: shadcn/ui (Tailwind)

### Library Rules
- Use `import type` for type-only imports, separate from value imports. Prevents runtime imports of pure types.
- Default to Server Components. Only add `"use client"` when the component needs browser APIs, event handlers, or useState/useEffect. Data fetching belongs in Server Components — no useEffect waterfalls.

## Rules
- Prefer named exports. Default exports only where the framework requires them (e.g., Next.js pages, layouts).
- Use path aliases from tsconfig when configured. Relative imports: never deeper than two levels.
- Avoid `any` — use `unknown` and narrow with type guards. `any` is acceptable only for untyped third-party boundaries. Define an interface for complex types — don't escape the type system.
- Every catch block must do something deliberate: re-throw, return a typed error, or log with context. Empty catch blocks are never acceptable. Intentional graceful degradation — catching a failure and continuing with a fallback — is fine when the degradation is logged and observable.
- Never hardcode API keys, secrets, database URLs, or credentials. Use environment variables or a secrets manager.
- Avoid disabling lint rules inline. When necessary, add a comment explaining why the disable is required.
- Explicit return types on all exported functions. Internal helpers can use inference.
- Use `SafeError` (from `utils/error.ts`) for errors that should surface a user-readable message to the client. All other errors are caught by middleware and reported to Sentry with a generic message.
- Use `createScopedLogger('scope-name')` for module-level loggers. Enrich with `.with({})` for request context. Never use `console.log` directly.

## Gotchas
- Next.js App Router components are Server Components by default. Add `'use client'` only when the component needs browser APIs, event handlers, or React hooks like useState/useEffect.

## Examples
*Not yet captured. Add short snippets showing the RIGHT way.*
