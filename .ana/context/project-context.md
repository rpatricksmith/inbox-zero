<!-- SCAFFOLD - Setup will fill this file -->

# Project Context

## What This Product Does

**Detected:** pnpm monorepo, with authentication (Better Auth), database (Prisma → postgresql, 63 models), and AI integration (Vercel AI). 1670 source files, 557 test files.
**Detected issues:** 1 warning — run `ana scan` for details

Inbox Zero is the AI email and calendar assistant that actually works. It runs alongside your existing Gmail, Google Workspace, or Microsoft Outlook — no client switch required. It organizes your inbox, pre-drafts replies that sound like you, tracks follow-ups, blocks cold emails, files attachments, and handles calendar with meeting briefs, booking links, and rescheduling. Users manage it from the web app or by chatting with the AI from Slack, Telegram, or Microsoft Teams.

**Target users:** Knowledge workers and professionals who spend too much time on email. 20,000+ active users.

**The gap:** AI email tools either don't capture your voice — the drafts sound robotic so nobody sends them — or they're not trustworthy with sensitive data because they're closed source. Or they force you to switch email clients entirely. Inbox Zero works alongside your existing inbox, the code is open so you can see exactly what it does, and the AI actually sounds like you. That's why people use it instead of hiring a VA.

**Positioning:** Open source and privacy-first is how we build trust. Microsoft/Outlook support and Teams integration are a major active push.

## Architecture

**Detected:** pnpm · 13 packages (inbox-zero-ai, @inboxzero/image-proxy-worker, @inboxzero/image-proxy-aws, @inboxzero/worker, @inbox-zero/api)
**Detected surfaces:** web (apps/web, TypeScript, Next.js), api (packages/api, TypeScript), cli (packages/cli, TypeScript)
**Detected:** 7 directories mapped: .github/, .vscode/, apps/, docker/, docs/, packages/, scripts/
**Detected deployment:** Vercel, GitHub Actions

The monorepo has one dominant application (`apps/web`) with everything else supporting it:

- **`apps/web`** — The full Next.js App Router application. Frontend, API routes, server actions, AI logic, and email provider integrations all live here. This is where 95%+ of development happens.
- **`apps/worker`** — Background worker process (BullMQ).
- **`apps/image-proxy`** / **`apps/image-proxy-aws`** — Cloudflare Worker and AWS Lambda image proxies.
- **`packages/cli`** — Self-hosting CLI (`npx @inbox-zero/cli setup`).
- **`packages/api`** — Public API wrapper for external integrations.
- **Utility packages** — `resend` (transactional email), `tinybird` / `tinybird-ai-analytics` (analytics), `scheduling`, `loops` (marketing), `image-proxy` (shared), `tsconfig`.

**Email Provider Abstraction:** A ~60-method `EmailProvider` interface (`utils/email/types.ts`) with `GmailProvider` and `OutlookProvider` implementations. `createEmailProvider()` factory in `utils/email/provider.ts` selects based on the account. API routes use `withEmailProvider` middleware to get the provider on the request object. Gmail-specific code in `utils/gmail/` (39 files), Outlook-specific in `utils/outlook/` (45 files).

**AI Multi-Provider:** 13 LLM providers supported (OpenAI, Anthropic, Google, Azure, Vertex, Bedrock, Groq, OpenRouter, AI Gateway, Ollama, OpenAI-compatible, plus Codex CLI and Claude Code). Model selection via `utils/llms/model.ts` — each use case has a `ModelType` (default, economy, chat, nano, draft) with env-configured provider/model pairs and comma-separated fallback chains. Users can bring their own API key. LLM calls go through `utils/llms/index.ts` which wraps Vercel AI SDK with fallback chains, cost control, PostHog tracing, and JSON repair.

**Rule Engine:** The core automation system. `utils/ai/choose-rule/run-rules.ts` orchestrates: incoming email → `match-rules.ts` (condition matching) → `ai-choose-rule.ts` (AI classification) → `choose-args.ts` (AI argument generation) → `execute.ts` (action dispatch). Actions are dispatched by type in `utils/ai/actions.ts`.

**Auth:** Better Auth (`utils/auth.ts`) with Google, Microsoft, and Apple social login. Google and Microsoft OAuth tokens double as email API credentials. SSO (SAML/OIDC) and SCIM for enterprise. JWT sessions.

**Messaging Channels:** Slack, Teams, and Telegram integrations under `utils/messaging/`. `MessagingProvider` enum, `MessagingChannel`/`MessagingRoute` models, with `rule-notifications.ts` as the dispatch hub.

**Validation:** Zod schemas in `utils/actions/*.validation.ts` (33 files). Types inferred with `z.infer`. Server actions use `next-safe-action`.

**Data fetching:** SWR on the client. Mutations via server actions (not POST API routes, with exceptions for mobile).

**Dual deployment model:** The hosted product runs on Vercel, but self-hosters run via Docker with the CLI (`packages/cli`). Both paths are first-class — features can't assume Vercel-specific infrastructure. The repo maintains Docker configs (`docker/`), a CLI setup wizard, and Google/Microsoft emulators for local development.

## Where to Make Changes

| Task | Where to go |
|------|-------------|
| New AI feature | `utils/ai/` — new subdirectory. For assistant chat tools: `utils/ai/assistant/tools/`. Wire with `createGenerateText`/`createGenerateObject` from `utils/llms/index.ts`. |
| New email action | Add `ActionType` enum value in `prisma/schema.prisma`, handler in `utils/ai/actions.ts` (switch dispatch), fields on `Action`/`ExecutedAction` if needed. |
| New API route | `app/api/` — use `withAuth`, `withEmailAccount`, or `withEmailProvider` middleware. For mutations, prefer server actions in `utils/actions/`. |
| New page/UI feature | `app/(app)/[emailAccountId]/` for email-scoped pages. Shared components in `components/`. |
| New messaging channel | `utils/messaging/providers/<name>/`. Add to `MessagingProvider` enum. Wire into `rule-notifications.ts`. |
| Modify rule engine | `utils/ai/choose-rule/` — `run-rules.ts` (orchestrator), `match-rules.ts` (matching), `execute.ts` (actions). |
| New validation schema | `utils/actions/<feature>.validation.ts` |
| Environment variables | Add to `.env.example`, `env.ts`, and `turbo.json`. Prefix client-side with `NEXT_PUBLIC_`. |

**Active development right now** (high-churn files): rule notifications and messaging channels, follow-up reminders, AI draft replies, and the AI assistant chat.

## Key Decisions

- **Two-way sync between prompt files and DB rules** — Users write rules as a prompt file, which gets parsed into discrete DB rules. The LLM sees DB rules, not the prompt file. This is messy (noted in ARCHITECTURE.md) but intentional: it enables per-rule tracking, static action execution without LLM interference, and precise condition matching. Known tech debt — may be reworked.
- **EmailAccount as the primary workspace boundary, not User** — Most features (rules, chat, labels, trackers, settings) are scoped to EmailAccount. A User can have multiple EmailAccounts across providers.
- **Conversation tracking via meta-rule** — The rule engine uses a synthetic `CONVERSATION_TRACKING_META_RULE_ID` that first matches conversations, then resolves to a specific SystemType (TO_REPLY, AWAITING_REPLY, FYI, ACTIONED). Two-phase approach is intentional.
- **Multi-provider AI with per-use-case model types** — Instead of one model for everything, each use case (default, economy, chat, nano, draft) can target a different provider/model via env vars, with fallback chains.
- **Self-hosting is first-class** — Docker setup, CLI, and emulators are maintained alongside the Vercel deployment. Features must work in both environments.

## What Looks Wrong But Is Intentional

- **`Newsletter` model doesn't mean newsletters** — It tracks any sender (approved, unsubscribed, auto-archived). Schema comments acknowledge the name is wrong; rename is planned.
- **`ColdEmail` model is deprecated but present** — Being migrated to GroupItem learned patterns. Kept for backward compatibility.
- **`Group` has unused fields** — `name` and `prompt` are vestigial from when Groups were standalone entities. Now they must be attached to a Rule.
- **`automate` field on Rule is always true** — Deprecated. All rules are automated now. Kept for historical data.
- **`PremiumTier` has `@map` aliases that don't match** — STARTER_MONTHLY maps to "BUSINESS_MONTHLY" in the DB, PROFESSIONAL_MONTHLY to "BUSINESS_PLUS_MONTHLY". Renamed tiers with backward-compatible storage.
- **`withEmailProvider` vs `withEmailAccount` look redundant** — Different purposes: `withEmailAccount` authenticates and loads account data; `withEmailProvider` additionally creates the Gmail/Outlook API client.
- **Session model exists but isn't used** — Better Auth is configured with JWT sessions only.

## Key Files

- Database schema: `apps/web/prisma/schema.prisma` (63 models)
- Deployment config: `apps/web/vercel.json`
- CI pipeline: `.github/workflows/ai-evals.yml`, `.github/workflows/api-release.yml`, `.github/workflows/build-changelog.yml` + 9 more
- AI model selection: `apps/web/utils/llms/model.ts`
- AI provider config: `apps/web/utils/llms/config.ts`
- LLM call wrapper: `apps/web/utils/llms/index.ts`
- Rule engine orchestrator: `apps/web/utils/ai/choose-rule/run-rules.ts`
- Rule matching: `apps/web/utils/ai/choose-rule/match-rules.ts`
- Action dispatch: `apps/web/utils/ai/actions.ts`
- Email provider interface: `apps/web/utils/email/types.ts`
- Email provider factory: `apps/web/utils/email/provider.ts`
- Auth config: `apps/web/utils/auth.ts`
- API middleware: `apps/web/utils/middleware.ts`
- Env variables: `apps/web/env.ts`
- Messaging dispatch: `apps/web/utils/messaging/rule-notifications.ts`
- Assistant chat: `apps/web/utils/ai/assistant/chat.ts`

## Active Constraints

- **Microsoft/Outlook and Teams parity** — The biggest active push. Lots of work to match Gmail features on the Outlook side and build out Teams as a messaging channel.
- **Mobile app in development** — Auth flows and mobile-specific UI are being built.
- **AI draft quality measurement** — Actively tracking what users send vs what the AI drafts (DraftSendLog + similarity scores), feeding back into the model via ReplyMemory.
- **Self-hosting is first-class** — Docker setup, CLI, and Google/Microsoft emulators are maintained. Features need to work in both Vercel and self-hosted environments.
- **Prompt file ↔ DB rules architecture** — Known tech debt. May be reworked. See Key Decisions.

## Domain Vocabulary

- **EmailAccount** — The primary workspace entity. Most features scope to an EmailAccount, not a User. One user can have multiple EmailAccounts (Gmail + Outlook).
- **Rule** — An automation rule with conditions (AI instructions, static patterns, group membership) and actions. Has `conditionalOperator` (AND/OR) and optional `systemType` for built-in rules.
- **SystemType** — Built-in rule types: TO_REPLY, FYI, AWAITING_REPLY, ACTIONED (conversation tracking), COLD_EMAIL, NEWSLETTER, MARKETING, CALENDAR, RECEIPT, NOTIFICATION.
- **Action** — A specific operation attached to a Rule: ARCHIVE, LABEL, DRAFT_EMAIL, FORWARD, NOTIFY_MESSAGING_CHANNEL, etc.
- **ExecutedRule / ExecutedAction** — Records of a rule matching and its actions executing against a specific message. Status: APPLIED, APPLYING, SKIPPED, ERROR.
- **Group / GroupItem** — Learned pattern collections attached to a Rule. GroupItem types: FROM, SUBJECT, BODY. Source: AI-learned, user-added, or label feedback. Can be exclude patterns.
- **Newsletter** — Misleading name. Actually represents any tracked sender with status (APPROVED, UNSUBSCRIBED, AUTO_ARCHIVED). Rename planned.
- **ThreadTracker** — Follow-up tracking. Types: AWAITING (waiting for their reply), NEEDS_REPLY (you need to reply), NEEDS_ACTION (you need to act).
- **ReplyMemory** — AI-learned facts/preferences scoped to GLOBAL, SENDER, DOMAIN, or TOPIC. Feeds into draft quality over time.
- **DraftSendLog** — Links an AI draft to what the user actually sent, with similarity score. Drives ReplyMemory learning.
- **Knowledge** — User-uploaded knowledge base entries (title + content) that inform AI replies.
- **MessagingChannel** — Connection to a Slack/Teams/Telegram workspace.
- **MessagingRoute** — Maps a channel to a purpose (RULE_NOTIFICATIONS, MEETING_BRIEFS, DIGESTS, etc.) and target.
- **ModelType** — AI model use-case tiers: default, economy, chat, nano, draft. Each can target different providers.
- **Category** — User-defined sender categories for inbox organization. Rules can filter by category.
