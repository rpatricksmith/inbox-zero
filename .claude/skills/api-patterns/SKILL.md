---
name: api-patterns
description: "Invoke when implementing API routes, request handling, middleware, or error responses. Contains validation, error format, route architecture, and authorization patterns."
---

# API Patterns

## Detected
- Framework: Next.js
- Validation: zod (95%)
- Auth: Better Auth

### Library Rules
- Verify Stripe webhook signatures using `stripe.webhooks.constructEvent()` with the raw body and signing secret. Never trust webhook payloads without signature verification — they can be forged.
- Use `.safeParse()` instead of `.parse()` for user input validation. `.parse()` throws on invalid input — `.safeParse()` returns a result object with typed errors, enabling structured error responses.
- Server Components fetch data directly from service functions or the database. Route Handlers are for EXTERNAL clients (webhooks, mobile apps, third-party integrations). Never call your own Route Handlers from Server Components — that adds an unnecessary network hop.

## Rules
- Use `withError` for public/unauthenticated routes, `withAuth` for user-level routes, `withEmailAccount` for email-account-scoped routes, `withEmailProvider` when the route needs to make email API calls. Always pass a scope string as the first argument: `withEmailAccount('labels', async (request) => { ... })`.
- Server actions (mutations) use `actionClient` (email-account-scoped), `actionClientUser` (user-scoped), or `adminActionClient` from `utils/actions/safe-action.ts`. Each validates auth, sets audit context, and instruments with Sentry. Never create raw server actions without these clients.
- Export the response type from GET route handlers for client-side type safety: `export type MyResponse = Awaited<ReturnType<typeof getData>>`.
- Validate all input at the API boundary. Parse request bodies, query params, and path params with the project's validation library before any processing.
- Return a consistent error response shape from every endpoint. Never leak stack traces, database errors, or internal paths in production responses.
- Keep route handlers thin. Validation, then service call, then response. Business logic and data access belong in separate modules.
- Verify the requesting user owns the requested resource. An authenticated user should not access another user's data by changing an ID in the URL.

## Gotchas
- Don't call Route Handlers from Server Components. Call the data function directly — the Route Handler is for external clients, not internal server-side calls.
- Always verify webhook signatures before processing Stripe events. Use `stripe.webhooks.constructEvent()` with the raw body — never trust the payload without verification.
- Use `.safeParse()` in route handlers, not `.parse()`. `.parse()` throws on invalid input — use `.safeParse()` and return a 400 with validation error details.

## Examples
*Not yet captured. Add short snippets showing the RIGHT way.*
