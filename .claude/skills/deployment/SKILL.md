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
*No universal deployment rules — deployment conventions are platform-specific. Run `claude --agent ana-setup` to configure for your deployment platform.*

## Gotchas
- Vercel serverless functions have execution time limits. Long-running operations (LLM calls, file processing, batch jobs) should use streaming to send the first byte quickly, or offload to background functions with `waitUntil()`.

## Examples
*Not yet captured. Add short snippets showing the RIGHT way.*
