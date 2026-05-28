---
name: deployment
description: "Invoke when working on deployment configuration, CI/CD pipelines, environment variables, or release processes. Contains project-specific deploy platform conventions."
---

# Deployment

## Detected
- Platform: Vercel
- Config: apps/web/vercel.json
- CI: GitHub Actions

## Rules
- Push to main goes live on Vercel — no separate staging. Vercel preview deployments on every PR serve as the review environment before merging. Docker images are published on push to main tagged with commit SHA; formal releases get version tags.
- CI runs lint, Prisma enum checks, server action export checks, client redirect checks, unit tests, integration tests, and a full Next.js build on every PR. All must pass before merge.
- When adding new workspace packages, add the `package.json` COPY line to `docker/Dockerfile.prod` and `docker/Dockerfile.local`.
- Env vars: add to `.env.example`, `env.ts`, and `turbo.json`. Prefix client-side with `NEXT_PUBLIC_`. Features must work in both Vercel and self-hosted Docker environments — don't assume Vercel-specific infrastructure.

## Gotchas
- Vercel serverless functions have execution time limits. Long-running operations (LLM calls, file processing, batch jobs) should use streaming to send the first byte quickly, or offload to background functions with `waitUntil()`.

## Examples
*Not yet captured. Add short snippets showing the RIGHT way.*
