# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What is OpenClaude

OpenClaude is a fork of the Claude Code source that adds an OpenAI-compatible provider shim, allowing Claude Code's full tool system to work with any LLM (GPT-4o, DeepSeek, Gemini, Ollama, Codex, etc.). The shim translates between Anthropic SDK formats and the OpenAI Chat Completions API transparently.

## Build & Run

```bash
bun install              # install dependencies
bun run build            # bundle → dist/cli.mjs (via scripts/build.ts)
bun run dev              # build + run
node dist/cli.mjs        # run built artifact directly
```

## Config Home

OpenClaude uses `~/.openclaude` as its default config home.
Do not assume it should auto-share `~/.claude` with Claude Code: reusing the legacy directory can pull in incompatible plugins, marketplace caches, and settings, which may break startup or terminal input.
If you explicitly need the old behavior for troubleshooting or migration, opt in with `OPENCLAUDE_USE_LEGACY_CLAUDE_HOME=1` or override the path with `CLAUDE_CONFIG_DIR`.

## Testing & Quality

```bash
bun test src/services/api/*.test.ts   # provider shim tests (openaiShim, codexShim)
bun run typecheck                     # tsc --noEmit (strict mode)
bun run smoke                         # build + --version sanity check
bun run doctor:runtime                # validate provider env + API reachability
bun run hardening:check               # smoke + runtime doctor
bun run hardening:strict              # typecheck + hardening:check
```

## Provider Profiles

```bash
bun run profile:init                  # auto-detect provider, writes .openclaude-profile.json
bun run dev:profile                   # launch using persisted profile
bun run dev:openai                    # OpenAI (requires OPENAI_API_KEY)
bun run dev:ollama                    # Ollama (localhost:11434)
bun run dev:codex                     # Codex (ChatGPT auth)
bun run dev:fast                      # quick: llama3.2:3b
bun run dev:code                      # coding: qwen2.5-coder:7b
```

Required env vars: `CLAUDE_CODE_USE_OPENAI=1`, `OPENAI_MODEL`, and optionally `OPENAI_API_KEY` / `OPENAI_BASE_URL`.

## Architecture

The core addition is a provider shim layer in `src/services/api/`:

```
Claude Code tool system
  → src/services/api/client.ts          — routes to Anthropic SDK or shim based on CLAUDE_CODE_USE_OPENAI
    → src/services/api/openaiShim.ts    — Anthropic ↔ OpenAI format translation (chat completions)
    → src/services/api/codexShim.ts     — Codex /responses API translation
    → src/services/api/providerConfig.ts — provider detection, Codex auth, model alias resolution
```

**openaiShim.ts** is the main translation layer (~700 lines). It converts:
- Anthropic message blocks → OpenAI messages
- Anthropic tool_use/tool_result → OpenAI function calls
- OpenAI SSE streaming → Anthropic stream events
- System prompt arrays → OpenAI system messages

**providerConfig.ts** handles provider routing: detects Codex aliases (`codexplan` → `gpt-5.4`, `codexspark` → `gpt-5.3-codex-spark`), resolves credentials from env vars or `~/.codex/auth.json`, and determines transport type (`chat_completions` vs `codex_responses`).

**client.ts** is the original Anthropic SDK client — the only modification is an early check for `CLAUDE_CODE_USE_OPENAI=1` to route through the shim instead.

Other modified files for provider integration:
- `src/utils/model/providers.ts` — added 'openai' provider type
- `src/utils/model/configs.ts` — openai model mappings
- `src/utils/model/model.ts` — respects OPENAI_MODEL for defaults
- `src/utils/auth.ts` — recognizes OpenAI as valid 3P provider

## Build System

`scripts/build.ts` uses Bun's bundler to produce a single `dist/cli.mjs`. It inlines feature flags (all disabled for the open build — VOICE_MODE, PROACTIVE, KAIROS, etc.) and MACRO.* globals (version, build-time constants). The `bin/openclaude` entry point loads `dist/cli.mjs` or tells the user to build first.

## Key Constraints

- **Zero added dependencies** — the shim works with the existing dependency set
- **No thinking mode** — Anthropic extended thinking is disabled for OpenAI models
- **No prompt caching** — Anthropic-specific cache headers are skipped
- **Token limit default** — 32K max output; some models may cap lower
- **TypeScript strict mode** — `tsconfig.json` has `strict: true`, target ES2022, JSX react-jsx
- **Path aliases** — `src/*` maps to `./src/*` (used in imports like `src/utils/auth.js`)
