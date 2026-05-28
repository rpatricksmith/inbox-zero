# Design Principles

<!-- Starting principles for AI-augmented development.
     Edit to match your team's philosophy, or replace entirely.
     Ana reads this to understand HOW your team thinks. -->

## Name the disease, not the symptom

Before fixing something, state the root cause in one sentence. A fix that addresses the cause is one fix forever. A fix that addresses the symptom is the first of many.

## Surface tradeoffs before committing

The user isn't asking for a scope, a plan, or code — they're asking for an outcome. Every approach has costs; if the obvious path undermines that outcome, say so before building. Show them the paths, not just the fastest one.

## Every change should be foundation, not scaffolding

Foundation is code you build on top of. Scaffolding is code you tear down later. The test: would a senior engineer approve this — not just for correctness, but for craft? If the answer is "this works, but it's not how we'd do it if we had time" — you don't have time NOT to do it right.

## Both Paths Must Work

Every feature ships to two environments: Vercel (hosted) and Docker (self-hosted). If it relies on a platform-specific primitive — serverless functions, edge middleware, managed queues — there must be a fallback. "Works on Vercel" is half-shipped.

## Provider-Agnostic by Default

New features go through the `EmailProvider` interface, not Gmail or Outlook directly. Provider-specific code lives in `utils/gmail/` or `utils/outlook/`, never in shared logic. If a feature only works on one provider, that's a known gap documented in the PR, not a silent omission.

## Ship it, then sharpen it

We ship features incomplete and iterate. 2-3 releases a week. Don't wait for perfect. But what ships must work — incomplete is fine, broken isn't.

## AI should feel like you

Draft quality and tone matching matter more than speed. If the AI reply doesn't sound like the user, they won't send it, and the feature is useless. We actively measure draft-to-send similarity.

## Work where users already are

The assistant goes to Slack, Telegram, Teams. Don't build features that only work in the web app. The assistant is the product, not the app.
