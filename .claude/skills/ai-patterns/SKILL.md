---
name: ai-patterns
description: "Invoke when building features that call LLM APIs, handling AI responses, managing prompts, or integrating AI SDKs. Contains error handling, security, prompt management, and observability patterns."
---

# AI Patterns

## Detected
- AI SDK: Vercel AI
- Also detected: OpenAI, Vercel AI (Anthropic), Vercel AI (OpenAI), Vercel AI (Google), Vercel AI (Google Vertex), Vercel AI (Bedrock), Vercel AI (Azure), Vercel AI (Groq), Vercel AI (Perplexity), Vercel AI (Gateway), Vercel AI (MCP), Vercel AI (OpenAI Compatible), Vercel AI (OpenRouter)

## Rules
- All LLM calls MUST go through the wrapper functions in `utils/llms/index.ts` (`createGenerateText`, `createGenerateObject`, `chatCompletionStream`, `toolCallAgentStream`). Never call Vercel AI SDK's `generateText`/`generateObject`/`streamText` directly — the wrapper handles multi-provider fallback chains, cost control, usage tracking, prompt hardening, sensitive data policy enforcement, and JSON repair.
- Use the `label` parameter on every LLM call for cost attribution and debugging. Labels feed PostHog tracing, usage tracking, and log correlation. Unlabeled calls can't be attributed or optimized.
- Each use case has a ModelType (default, economy, chat, nano, draft) configured via env vars with comma-separated fallback chains per type. Users can bring their own API key and choose providers per task. Don't hardcode provider/model — use the ModelType system.
- User automation prompts flow through a two-stage system: users write a "prompt file" in plain English, which gets parsed into discrete rules stored in the database. The LLM sees the DB rules, not the prompt file. This enables per-rule execution tracking and static action execution. The two-way sync between prompt file and DB rules is known tech debt but intentional.
- System feature prompts (cold email detection, conversation tracking) are centralized in `utils/ai/` and not user-editable. Keep them there — don't scatter system prompts across route handlers or components.
- Never interpolate raw user input into system prompts. User content goes in user messages with clear role boundaries. System instructions stay immutable.
- Treat all LLM output as untrusted. Validate and sanitize before using in database queries, HTML rendering, or business logic.
- Handle LLM errors by type: retry rate limits with backoff, truncate input for context overflow, log content filter triggers, fail gracefully for API outages.
- Use structured output (JSON mode, tool_use) for data extraction. Never regex-parse free-text LLM responses for application data.
- Centralize prompt templates — don't scatter prompt strings across business logic. Prompts should be versionable, testable, and reviewable independently.
- Log model, token count, and latency per LLM call. You can't optimize cost or debug quality without knowing what each request consumed.

## Gotchas
- Use `generateObject()` for structured output and `streamText()` for streaming responses. Don't use `generateText()` with manual JSON parsing.
- OpenAI SDK supports `maxRetries` in the client constructor. Use `response_format: { type: 'json_object' }` for structured output instead of parsing free text.

## Examples
*Not yet captured. Add short snippets showing the RIGHT way.*
