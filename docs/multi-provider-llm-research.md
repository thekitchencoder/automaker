# Multi-Provider LLM Research: Planning Document

> Research findings to seed a planning session for enabling multi-provider LLM support in Automaker — allowing different models from different providers (including local LLMs) for different agents and activities.

---

## Table of Contents

1. [Executive Summary](#executive-summary)
2. [Current Architecture Analysis](#current-architecture-analysis)
3. [Existing Provider Integrations](#existing-provider-integrations)
4. [Per-Activity Model Assignment (Today)](#per-activity-model-assignment-today)
5. [External CLI Tools: Capabilities & Limitations](#external-cli-tools-capabilities--limitations)
6. [Alternative Providers & Local LLMs](#alternative-providers--local-llms)
7. [Integration Patterns](#integration-patterns)
8. [Claude Code-Only Multi-Provider Strategy](#claude-code-only-multi-provider-strategy)
9. [Local LLMs as Intelligent Tools (A2A Pattern)](#local-llms-as-intelligent-tools-a2a-pattern)
10. [Gaps & Opportunities](#gaps--opportunities)
11. [Design Considerations for Planning](#design-considerations-for-planning)
12. [Open Questions](#open-questions)

---

## Executive Summary

Automaker already has a **sophisticated multi-provider architecture** with 6 registered providers (Claude, Cursor, Codex, OpenCode, Gemini, Copilot) and a registry-based `ProviderFactory` for model routing. It also supports **Claude-compatible providers** (OpenRouter, MiniMax, GLM) via API endpoint configuration. Per-phase model selection exists via `PhaseModelConfig`.

**Key finding**: The most pragmatic path forward avoids adding new CLI tools or new provider implementations. Instead, it leverages **Claude Code as the single CLI tool** with `ClaudeCompatibleProvider` configurations pointing at different backends. LM Studio and Ollama both support Anthropic-compatible endpoints natively — no proxy needed. Per-phase provider routing already works via `PhaseModelEntry.providerId`.

**The most compelling architecture** is using local LLMs as **intelligent tools** (via MCP servers or extended subagents) rather than as replacement providers. The frontier model (Opus) stays as orchestrator and delegates grunt work (linting, single-class implementation, test generation, commit messages) to cheap/local models via tool calls. This works today with MCP tools and zero Automaker code changes.

### Key gaps to address:

1. **Subagents don't support per-provider routing** — `AgentDefinition.model` is locked to `'sonnet' | 'opus' | 'haiku' | 'inherit'`, no `providerId` field
2. **Auto-mode doesn't pass subagents through** — `buildExecOpts()` in `agent-executor.ts` omits the `agents` field
3. **No per-pipeline-step provider override** — all steps use the same model
4. **MCP-based delegation is one-shot** — local model can't do multi-turn tool use through an MCP tool call

---

## Current Architecture Analysis

### Provider Registry Pattern

**File**: `apps/server/src/providers/provider-factory.ts`

The `ProviderFactory` uses a registry pattern where providers self-register:

```typescript
registerProvider('claude', {
  factory: () => new ClaudeProvider(),
  aliases: ['anthropic'],
  canHandleModel: (model) => model.startsWith('claude-') || ['opus', 'sonnet', 'haiku'].some(n => model.includes(n)),
  priority: 0,
});
```

Providers are matched by:
1. `canHandleModel()` function (checked in priority order)
2. Model prefix fallback (e.g., `cursor-*` → cursor provider)
3. Default: Claude

### Provider Class Hierarchy

```
BaseProvider (abstract)
├── ClaudeProvider        — SDK-based (Claude Agent SDK)
├── CliProvider (abstract) — CLI subprocess spawning
│   ├── CursorProvider    — cursor-agent CLI
│   ├── CodexProvider     — codex CLI (custom JSONL)
│   ├── OpencodeProvider  — opencode CLI
│   ├── GeminiProvider    — gemini CLI
│   └── CopilotProvider   — GitHub Copilot CLI
```

### Key Interfaces

**`BaseProvider`** (`apps/server/src/providers/base-provider.ts`):
- `getName(): string`
- `executeQuery(options: ExecuteOptions): AsyncGenerator<ProviderMessage>`
- `detectInstallation(): Promise<InstallationStatus>`
- `getAvailableModels(): ModelDefinition[]`
- `supportsFeature(feature: string): boolean`

**`CliProvider`** (`apps/server/src/providers/cli-provider.ts`):
- Adds CLI detection, subprocess spawning, JSONL streaming
- Handles cross-platform CLI detection (PATH, common paths, WSL, npx)
- Embeds system prompts into user prompts for CLI tools that lack system prompt support

**`ExecuteOptions`** (`libs/types/src/provider.ts`):
- `prompt`, `model`, `cwd`, `systemPrompt`, `maxTurns`, `allowedTools`
- `thinkingLevel` (Claude), `reasoningEffort` (Codex/OpenAI)
- `claudeCompatibleProvider` — for routing through alternative Anthropic-compatible endpoints
- `agents` — subagent definitions with per-agent model overrides
- `mcpServers` — MCP server configuration

**`ProviderMessage`** — unified output format all providers must conform to:
- `type: 'assistant' | 'user' | 'error' | 'result'`
- `message.content: ContentBlock[]` with `text`, `tool_use`, `thinking`, `tool_result` blocks

### Model Resolution

**Package**: `libs/model-resolver/`

The `resolveModelString()` function maps aliases to full model IDs:
- `'haiku'` → `'claude-haiku-4-5-20251001'`
- `'sonnet'` → `'claude-sonnet-4-6'`
- `'opus'` → `'claude-opus-4-6'`

Provider routing uses prefix matching:
- `'cursor-*'` → Cursor
- `'codex-*'` → Codex
- `'opencode-*'` → OpenCode
- `'gemini-*'` → Gemini
- `'copilot-*'` → Copilot
- Everything else → Claude (default)

---

## Existing Provider Integrations

### 1. Claude Provider (SDK-based)

**File**: `apps/server/src/providers/claude-provider.ts` (15KB)

- Uses `@anthropic-ai/claude-agent-sdk` directly
- Native multi-turn conversation, tool use, vision, thinking blocks
- No CLI needed — runs as an npm dependency
- Supports **Claude-compatible providers** via `ClaudeCompatibleProvider` config:
  - Custom `ANTHROPIC_BASE_URL` endpoints
  - API key strategies: inline, env var, or credentials store
  - Model mapping (provider models → Claude aliases)
  - Pre-configured templates: Anthropic, OpenRouter, z.AI GLM, MiniMax

### 2. Codex Provider (CLI-based)

**File**: `apps/server/src/providers/codex-provider.ts` (41KB)

- Spawns `codex exec --model <model> --json --full-auto`
- Custom JSONL event stream parsing
- Models: GPT-5.x series, GPT-5.x-codex variants
- Auth: `codex login` (OAuth) or `OPENAI_API_KEY`
- Has its own SDK client (`codex-sdk-client.ts`) and config manager (`codex-config-manager.ts`)
- Supports reasoning effort levels (`none` through `xhigh`)

### 3. Cursor Provider (CLI-based)

**File**: `apps/server/src/providers/cursor-provider.ts` (40KB)

- Spawns `cursor-agent` CLI
- JSONL stream format
- Models: composer-1, cursor-auto, various claude/gpt models via Cursor proxy
- Auth via Cursor's own login system

### 4. OpenCode Provider (CLI-based)

**File**: `apps/server/src/providers/opencode-provider.ts` (50KB)

- Spawns `opencode` CLI with `--output-format stream-json`
- **Dynamic model discovery** via `opencode models` command
- Free tier models: big-pickle, GLM-5, GPT-5-nano, Kimi-K2.5, MiniMax-M2.5
- Dynamic models from authenticated providers: GitHub Copilot, Anthropic, OpenAI, Google, xAI, OpenRouter
- Pattern: `provider/model-name` format (e.g., `github-copilot/gpt-4o`)
- This is the most provider-agnostic CLI tool — acts as a gateway to many backends

### 5. Gemini Provider (CLI-based)

**File**: `apps/server/src/providers/gemini-provider.ts` (12KB)

- Spawns `gemini` CLI with `--output-format stream-json`
- Models: Gemini 2.5/3.x Pro, Flash variants
- Auth: Google account or API key

### 6. Copilot Provider (CLI-based)

**File**: `apps/server/src/providers/copilot-provider.ts` (30KB)

- Uses GitHub Copilot SDK
- Models accessed via GitHub subscription: Claude models, GPT models, Gemini, etc.
- Acts as another multi-model gateway (similar to OpenCode)

---

## Per-Activity Model Assignment (Today)

### PhaseModelConfig

**File**: `libs/types/src/settings.ts` (lines 865-893)

The system already supports per-phase model selection:

```typescript
interface PhaseModelConfig {
  // Quick tasks
  enhancementModel: PhaseModelEntry;        // Feature name/description enhancement
  fileDescriptionModel: PhaseModelEntry;     // File context descriptions
  imageDescriptionModel: PhaseModelEntry;    // Image analysis

  // Validation
  validationModel: PhaseModelEntry;          // GitHub issue validation

  // Generation
  specGenerationModel: PhaseModelEntry;      // App spec generation
  featureGenerationModel: PhaseModelEntry;   // Feature list from specs
  backlogPlanningModel: PhaseModelEntry;     // Backlog planning
  projectAnalysisModel: PhaseModelEntry;     // Project analysis
  ideationModel: PhaseModelEntry;            // Feature ideation

  // Operational
  memoryExtractionModel: PhaseModelEntry;    // Learning extraction
  commitMessageModel: PhaseModelEntry;       // Commit messages
}
```

Each `PhaseModelEntry` supports:

```typescript
interface PhaseModelEntry {
  providerId?: string;          // Links to a ClaudeCompatibleProvider
  model: ModelId;               // Model ID (any provider)
  thinkingLevel?: ThinkingLevel;    // Claude thinking
  reasoningEffort?: ReasoningEffort; // Codex reasoning
}
```

### Feature-Level Model Selection

**File**: `libs/types/src/feature.ts` (line 90)

Each feature card has:
- `model?: string` — specific model override
- `thinkingLevel?: ThinkingLevel`
- `reasoningEffort?: ReasoningEffort`

### SDK Options Factory

**File**: `apps/server/src/lib/sdk-options.ts`

Creates SDK configurations per use-case:
- `createSpecGenerationOptions()` — read-only, extended turns
- `createFeatureGenerationOptions()` — quick, JSON generation
- `createAutoModeOptions()` — full tool access, max turns
- `createChatOptions()` — interactive, full tools
- `createSuggestionsOptions()` — read-only, analysis

Each can be overridden via environment variables:
- `AUTOMAKER_MODEL_SPEC`, `AUTOMAKER_MODEL_FEATURES`, `AUTOMAKER_MODEL_CHAT`, etc.

### Subagent Model Override

**File**: `libs/types/src/provider.ts` (lines 142-151)

```typescript
interface AgentDefinition {
  description: string;
  prompt: string;
  tools?: string[];
  model?: 'sonnet' | 'opus' | 'haiku' | 'inherit';  // Currently Claude-only
}
```

This is limited to Claude model aliases — it cannot reference Cursor, Codex, OpenCode, or local models.

---

## External CLI Tools: Capabilities & Limitations

### Claude Code CLI

- **Invocation**: Via `@anthropic-ai/claude-agent-sdk` (SDK, not CLI subprocess for the main path)
- **Models**: Claude family only (haiku, sonnet, opus)
- **Programmatic API**: Full SDK with `query()` function, streaming, tool use, MCP
- **Multi-provider**: No — Anthropic models only, but supports custom Anthropic-compatible base URLs
- **Key limitation**: No non-Anthropic model support

### OpenCode CLI

- **Invocation**: `opencode --output-format stream-json -p <prompt>`
- **Models**: Dynamic discovery — supports 15+ providers (Anthropic, OpenAI, Google, xAI, GitHub Copilot, OpenRouter, AWS Bedrock, and more)
- **Multi-provider**: YES — the most flexible CLI tool. Any authenticated provider's models are available
- **Local LLM support**: Via OpenRouter or custom providers that expose compatible APIs
- **Key strength**: Acts as a universal gateway. If OpenCode supports a provider, Automaker can use it

### Codex CLI

- **Invocation**: `codex exec --model <model> --json --full-auto`
- **Models**: OpenAI GPT models only
- **Multi-provider**: No — OpenAI ecosystem only
- **Key limitation**: Locked to OpenAI

### Gemini CLI

- **Invocation**: `gemini --output-format stream-json`
- **Models**: Google Gemini models only
- **Multi-provider**: No — Google ecosystem only

### GitHub Copilot CLI

- **Invocation**: Via `@anthropic-ai/claude-agent-sdk` + Copilot SDK
- **Models**: Multi-model via GitHub subscription (Claude, GPT, Gemini)
- **Multi-provider**: Partially — accesses multiple model families through GitHub's proxy
- **Key limitation**: Requires GitHub Copilot subscription

### Detailed CLI Programmatic Usage

#### Claude Code — Headless / SDK Modes

- **`--print` / `-p` flag**: Non-interactive execution, exits after response. Combine with `--output-format json` or `stream-json` for structured parsing
- **`--model`**: Select model (opus, sonnet, haiku)
- **`--system-prompt` / `--append-system-prompt`**: Control system instructions
- **`--allowedTools`**: Pre-approve tools (e.g., `Bash(git diff *)`)
- **`--dangerously-skip-permissions`**: For trusted containers only
- **`--worktree` / `-w`**: Isolated git worktree sessions
- **`--agents`**: Define sub-agents as JSON
- **Agent SDK**: Spawns Claude Code as subprocess, communicates via stdin/stdout JSON. Provides custom tools via in-process MCP servers, subagent configuration, hooks, and permission modes
- **Non-Anthropic workarounds**: LiteLLM proxy (`ANTHROPIC_BASE_URL` override), OpenRouter, or direct Anthropic-compatible providers. Policy note: Anthropic prohibits using OAuth tokens from subscriptions in third-party tools, but using third-party *models* in Claude Code via API keys is permitted

#### OpenCode — Most Flexible Gateway

- **`opencode run`**: Non-interactive mode. Flags: `--model provider/model`, `--format json`, `--quiet`, `--file`, `--session`, `--continue`
- **`opencode serve`**: Headless HTTP server with OpenAPI endpoint (protectable with `OPENCODE_SERVER_PASSWORD`)
- **`opencode run --attach`**: Attach to running server to avoid MCP cold starts
- **`opencode web`**: Web UI mode
- **`opencode acp`**: Agent Client Protocol via stdin/stdout nd-JSON
- **Configuration**: `opencode.json` with `provider`, `model` (format: `provider_id/model_id`), `small_model` for lightweight tasks, `disabled_providers` / `enabled_providers`, model variant cycling
- **Providers supported**: OpenAI, Anthropic, Google Gemini, AWS Bedrock, Azure OpenAI, Groq, OpenRouter, Cerebras, Deep Infra, Fireworks AI, MiniMax, Moonshot AI, DeepSeek, GitLab Duo, **Ollama**, Docker Model Runner, any OpenAI-compatible endpoint

#### Codex CLI — OpenAI with Local Support

- **Models**: GPT-5.3 Codex (primary), GPT-5.1 Codex Max, GPT-5 Codex Mini
- **`--oss` flag**: Enables local model support (defaults to Ollama with `gpt-oss:20b`). Note: hardcodes Ollama-specific `/api/tags` and `/api/pull` calls — incompatible with non-Ollama OpenAI-compatible servers
- **Azure**: Configure via `config.toml` with Azure-specific `base_url`
- **Custom providers**: Via `config.toml` profiles with `model_provider`, `base_url`, `env_key`
- **In-session**: `/model` command for switching

#### Aider — Broadest Provider Support

- **Cloud**: OpenAI, Anthropic, Google, Azure, Cohere, DeepSeek, xAI, Vertex AI, Amazon Bedrock
- **Aggregators**: OpenRouter, GitHub Copilot, Fireworks AI
- **Local**: `aider --model ollama_chat/<model-name>` (auto-manages context window, recommended: qwen2.5-coder, deepseek-coder-v2)
- **Configuration**: API keys via env vars, `.env`, or `--api-key provider=<key>`. YAML config for persistence
- **Architect mode**: Uses capable model for planning, cheaper model for execution
- **Note**: Not currently integrated into Automaker, but its architecture pattern (architect mode, multi-provider config) is worth studying

### Summary Table

| CLI Tool       | Provider Count | Local LLM | OpenAI-Compat API | Custom Endpoints | Headless JSON |
|---------------|---------------|-----------|-------------------|-----------------|--------------|
| Claude (SDK)  | 1 + compat    | Via proxy  | No                | Yes (Anthropic) | Yes (SDK) |
| OpenCode      | 20+           | Ollama native | Yes            | Yes             | Yes |
| Codex         | 1 + Azure     | Ollama (--oss) | Limited       | Via config.toml | Limited |
| Gemini        | 1             | No        | No                | No              | Yes |
| Copilot       | 3+            | No        | No                | No              | Yes |
| Aider         | 15+           | Ollama native | Yes            | Yes             | Limited |

---

## Alternative Providers & Local LLMs

### Cloud Providers Not Yet Integrated

#### OpenRouter
- **Status**: Partially supported via `ClaudeCompatibleProvider` template
- **API format**: Anthropic-compatible (they proxy to Claude)
- **Additional models**: 300+ models from all major providers
- **Key value**: Single API key for access to Claude, GPT, Gemini, Mistral, Llama, DeepSeek, etc.
- **Gap**: Currently only usable through the Claude provider path. Non-Claude models via OpenRouter would require OpenAI-compatible API handling

#### MiniMax
- **Status**: Supported via `ClaudeCompatibleProvider` template (MiniMax M2.1)
- **API format**: Anthropic-compatible endpoint
- **Models**: MiniMax M2.1 (maps to all three Claude slots)
- **Gap**: Only one model available; locked to Anthropic-compatible API

#### DeepSeek
- **API format**: OpenAI-compatible
- **Models**: DeepSeek-R1 (reasoning), DeepSeek-V3 (general), DeepSeek-Coder
- **Integration path**: OpenAI-compatible API provider or via OpenRouter/OpenCode
- **Key value**: Very cost-effective for coding tasks, strong reasoning

#### Mistral
- **API format**: OpenAI-compatible
- **Models**: Mistral Large, Codestral (coding), Mixtral
- **Integration path**: OpenAI-compatible API provider or via OpenRouter

#### Google Gemini (API)
- **Status**: CLI integration exists via Gemini CLI
- **Gap**: No direct API integration (only via CLI subprocess)
- **Alternative**: Available via OpenRouter, OpenCode, or Copilot

#### xAI Grok
- **API format**: OpenAI-compatible
- **Models**: Grok-3
- **Integration path**: OpenAI-compatible API provider or via OpenCode

### Local LLM Options

#### Ollama
- **API format**: OpenAI-compatible (`http://localhost:11434/v1`)
- **Models**: Llama 3, CodeLlama, Mistral, Qwen, DeepSeek, many more
- **Pros**: Free, private, no API keys, runs on consumer hardware
- **Cons**: Quality varies by model size, slower than cloud APIs, limited context windows
- **Integration path**: OpenAI-compatible API provider
- **Best use cases**: Quick tasks (enhancement, description), cost-sensitive operations

#### LM Studio
- **API format**: OpenAI-compatible (`http://localhost:1234/v1`)
- **Models**: Same as Ollama + GUI for model management
- **Pros**: User-friendly, drop-in OpenAI replacement
- **Integration path**: OpenAI-compatible API provider

#### vLLM
- **API format**: OpenAI-compatible
- **Models**: Any HuggingFace model
- **Pros**: Production-grade serving, high throughput, tensor parallelism
- **Cons**: Requires GPU server setup
- **Best use cases**: Self-hosted production deployment
- **Integration path**: OpenAI-compatible API provider

#### llama.cpp Server
- **API format**: OpenAI-compatible
- **Models**: GGUF-quantized models
- **Pros**: Minimal dependencies, runs on CPU
- **Integration path**: OpenAI-compatible API provider

### Key Insight: The OpenAI-Compatible API Pattern

Nearly all local LLM servers and most alternative cloud providers expose an **OpenAI-compatible API** (`/v1/chat/completions`). This is the universal integration point:

```
Ollama       → http://localhost:11434/v1
LM Studio    → http://localhost:1234/v1
vLLM         → http://localhost:8000/v1
llama.cpp    → http://localhost:8080/v1
DeepSeek     → https://api.deepseek.com/v1
Mistral      → https://api.mistral.ai/v1
xAI          → https://api.x.ai/v1
OpenRouter   → https://openrouter.ai/api/v1
Together AI  → https://api.together.xyz/v1
Groq         → https://api.groq.com/openai/v1
```

**A single "OpenAI-compatible API" provider would unlock all of the above.**

### The LiteLLM Bridge Option

[LiteLLM](https://docs.litellm.ai/) is a proxy that translates 100+ providers into both OpenAI (`/v1/chat/completions`) AND Anthropic (`/v1/messages`) format with ~8ms P95 latency overhead. This means any tool that speaks either format can reach any provider. Particularly relevant for Claude Code, which requires Anthropic-format APIs — LiteLLM can translate DeepSeek, Mistral, or local Ollama into Anthropic format.

### API Compatibility Matrix

| Provider | OpenAI Compat | Anthropic Compat | Native SDK |
|---|---|---|---|
| Ollama | Yes (built-in) | Yes (built-in) | Yes |
| LM Studio | Yes (built-in) | Yes (built-in) | Yes |
| llama.cpp | Yes (built-in) | No | No |
| vLLM | Yes (built-in) | No | No |
| MiniMax | Yes | Yes | Yes |
| DeepSeek | Yes (native) | Via proxy | Yes |
| Mistral | Yes (native) | Via proxy | Yes |
| Groq | Yes (native) | Via proxy | No |
| Google Gemini | Via proxy | Via proxy | Yes |

### Cost Comparison (per MTok, input/output)

| Provider/Model | Input | Output | Context | Best For |
|---|---|---|---|---|
| Claude Opus 4.6 | $5.00 | $25.00 | 1M | Complex reasoning, architecture |
| Claude Sonnet 4.6 | $3.00 | $15.00 | 1M | General coding workhorse |
| GPT-5.3 Codex | Subscription | Subscription | 256K | OpenAI ecosystem coding |
| DeepSeek V3.2 | $0.28 | $0.42 | 128K | Budget coding, reasoning |
| MiniMax M2.5 | ~$0.24 | ~$1.20 | 196K | Budget agentic coding |
| Mistral Devstral Small | $0.10 | $0.30 | 131K | Open-source, self-hosted coding |
| Groq GPT-OSS-120B | $0.15 | $0.75 | varies | Speed-critical inference |
| Groq GPT-OSS-20B | $0.10 | $0.50 | varies | Ultra-fast, budget |
| Ollama (local) | Free | Free | Varies | Privacy, offline, zero cost |

---

## Integration Patterns

### Pattern 1: New OpenAI-Compatible API Provider (Direct)

Create a new `OpenAICompatProvider` extending `BaseProvider` that:
- Calls OpenAI-format `/v1/chat/completions` endpoints
- Supports streaming via SSE
- Maps tool calls between OpenAI format and Automaker's `ProviderMessage` format
- Configurable base URL (for local or cloud)

```
Automaker → OpenAICompatProvider → Any OpenAI-compatible endpoint
```

**Pros**: Full control, no CLI dependency, works offline
**Cons**: Must implement tool mapping, conversation management, streaming

### Pattern 2: Leverage OpenCode CLI as Universal Gateway

OpenCode already supports 15+ providers and handles all the API differences. Automaker's `OpencodeProvider` already integrates with it.

```
Automaker → OpencodeProvider → opencode CLI → Any provider opencode supports
```

**Pros**: Already partially working, provider management delegated, dynamic model discovery
**Cons**: Requires opencode CLI installed, subprocess overhead, limited to opencode's supported providers

### Pattern 3: Claude-Compatible Provider Extension

Extend the existing `ClaudeCompatibleProvider` system to also support OpenAI-compatible API format:

```typescript
interface ClaudeCompatibleProvider {
  // ... existing fields ...
  apiFormat?: 'anthropic' | 'openai';  // NEW: Which API format to use
}
```

**Pros**: Reuses existing UI and settings infrastructure
**Cons**: The Claude provider would need to handle two API formats, increasing complexity

### Pattern 4: LiteLLM Proxy as Universal Translator

Run LiteLLM as a local/sidecar service that translates any provider into Anthropic format. This lets the existing Claude provider + `ClaudeCompatibleProvider` config reach any model:

```
Automaker → ClaudeProvider → LiteLLM (localhost:4000) → Any provider
```

**Pros**: Zero provider code changes, reuses existing UI/settings, supports 100+ providers
**Cons**: Additional service dependency, operational complexity, another process to manage

### Pattern 5: Hybrid Approach

Combine patterns based on quality tier:

| Activity | Recommended Provider Path |
|----------|--------------------------|
| Feature implementation (main agent) | Claude SDK / Codex CLI (highest quality) |
| Spec generation | Claude SDK / OpenAI-compat (needs deep reasoning) |
| Enhancement, descriptions | OpenAI-compat / Local LLM (cost-effective, fast) |
| Commit messages | OpenAI-compat / Local LLM (simple task) |
| Validation | Claude SDK / Gemini (needs accuracy) |
| Chat | Any provider (user preference) |

---

## Claude Code-Only Multi-Provider Strategy

The above patterns assume adding new provider code or external CLI tools. A simpler approach exists: **use Claude Code as the single CLI tool and route to different backends via `ClaudeCompatibleProvider` configurations** that swap `ANTHROPIC_BASE_URL`, `ANTHROPIC_API_KEY`, and model IDs per invocation.

This already works for any backend that speaks the Anthropic Messages API — including LM Studio (native Anthropic endpoint support), MiniMax, and OpenRouter. No additional CLI tools needed.

### How It Works Today

The `ClaudeProvider.executeQuery()` method in `apps/server/src/providers/claude-provider.ts` calls the Claude Agent SDK's `query()` function. It passes an `env` object constructed by `buildEnv()`:

```typescript
// claude-provider.ts:228
env: buildEnv(providerConfig, credentials),
```

`buildEnv()` constructs a **clean environment** when a provider is configured:

```typescript
function buildEnv(providerConfig?, credentials?) {
  const env = {};

  if (providerConfig) {
    // Clean switch — only provider settings, no inherited process.env
    env['ANTHROPIC_BASE_URL'] = providerConfig.baseUrl;

    if (providerConfig.useAuthToken) {
      env['ANTHROPIC_AUTH_TOKEN'] = resolvedApiKey;
    } else {
      env['ANTHROPIC_API_KEY'] = resolvedApiKey;
    }

    if (providerConfig.timeoutMs) {
      env['API_TIMEOUT_MS'] = String(providerConfig.timeoutMs);
    }
    if (providerConfig.disableNonessentialTraffic) {
      env['CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC'] = '1';
    }
  }
  // + system vars (PATH, HOME, SHELL, etc.)
  return env;
}
```

The SDK spawns a Claude Code subprocess with this environment. The subprocess sees the custom `ANTHROPIC_BASE_URL` and routes all API calls to that endpoint.

### Per-Phase Provider Routing (Already Supported)

Each `PhaseModelEntry` already has a `providerId` field:

```typescript
interface PhaseModelEntry {
  providerId?: string;   // → links to a ClaudeCompatibleProvider by ID
  model: ModelId;
  thinkingLevel?: ThinkingLevel;
}
```

When a phase model has a `providerId`, `getProviderByModelId()` in `apps/server/src/lib/settings-helpers.ts` looks up the matching `ClaudeCompatibleProvider` from settings and passes it through to `ClaudeProvider.executeQuery()` as `claudeCompatibleProvider`. The full chain:

```
PhaseModelEntry.providerId
  → getProviderByModelId() resolves ClaudeCompatibleProvider
  → ExecuteOptions.claudeCompatibleProvider = provider
  → ClaudeProvider.executeQuery() receives it
  → buildEnv(provider, credentials) constructs clean env
  → SDK subprocess runs with custom ANTHROPIC_BASE_URL
```

This means you can already assign LM Studio or MiniMax to specific phases (enhancement, commit messages, etc.) via the settings UI — configure a `ClaudeCompatibleProvider` with the local base URL, then select its models in the phase model dropdowns.

### What LM Studio Native Anthropic Support Enables

LM Studio exposes an Anthropic-compatible endpoint at `http://localhost:1234/v1` that directly accepts the Claude Messages API format. This means:

- No proxy (LiteLLM) needed
- No additional CLI tools needed
- Configure as a `ClaudeCompatibleProvider` with `baseUrl: "http://localhost:1234/v1"`
- Claude Agent SDK connects to it as if it were Anthropic's API
- Full tool use support (if the local model supports function calling)

### Gaps in the Claude Code-Only Approach

1. **Subagents don't support per-provider routing** — `AgentDefinition` only has `model: 'sonnet' | 'opus' | 'haiku' | 'inherit'`, no `providerId` or `baseUrl` field. Subagents always inherit the parent's environment.

2. **Auto-mode doesn't pass subagents** — `buildExecOpts()` in `agent-executor.ts:730` omits the `agents` field entirely, so auto-mode feature execution can't use subagents at all.

3. **Agent discovery validates model against Claude aliases only** — `agent-discovery.ts:66` restricts model to `['sonnet', 'opus', 'haiku', 'inherit']`. Can't specify a custom provider model ID.

4. **No per-pipeline-step provider override** — `pipeline-orchestrator.ts:117` uses `resolveModelString(feature.model)` for all steps. Can't lint with a cheap local model and implement with Opus.

---

## Local LLMs as Intelligent Tools (A2A Pattern)

Instead of treating local LLMs as alternative *providers* that replace the frontier model, treat them as **intelligent tools** that the frontier model *delegates to*. The expensive Opus model stays in the driver's seat — planning, coordinating, reviewing — and offloads commodity grunt work to cheaper/local models via tool calls.

### Concept

```
┌─────────────────────────────────────────────────┐
│  Claude Opus (frontier model — orchestrator)    │
│  Plans architecture, reviews code, coordinates  │
└───┬──────────┬──────────┬──────────┬────────────┘
    │          │          │          │
    ▼          ▼          ▼          ▼
┌────────┐ ┌────────┐ ┌────────┐ ┌────────────┐
│ Lint   │ │ Impl   │ │ Test   │ │ Format     │
│ Agent  │ │ Agent  │ │ Writer │ │ Agent      │
│(local) │ │(local) │ │(local) │ │(local)     │
└────────┘ └────────┘ └────────┘ └────────────┘
  LM Studio   Ollama    LM Studio   Ollama
```

The frontier model uses `Task` tool calls (subagents) or MCP tool calls to invoke local models for:
- Single class/function implementation from a clear spec
- Linting and formatting checks
- Test generation from interfaces
- Documentation generation
- Commit message drafting
- File description generation

### Implementation Approach A: MCP Tool Server (Works Today, Zero Code Changes)

Write an MCP server that wraps local LLM calls as tools. Register it in Automaker's MCP settings. The frontier model calls these tools naturally.

```typescript
// mcp-local-llm-tools/src/index.ts
import { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";
import { z } from "zod";

const server = new McpServer({ name: "local-llm-tools" });

server.tool(
  "local_implement_class",
  {
    className: z.string(),
    spec: z.string(),
    language: z.string(),
    existingImports: z.string().optional(),
  },
  async ({ className, spec, language, existingImports }) => {
    const response = await fetch("http://localhost:1234/v1/messages", {
      method: "POST",
      headers: { "Content-Type": "application/json", "x-api-key": "lm-studio" },
      body: JSON.stringify({
        model: "qwen2.5-coder-32b",
        max_tokens: 8192,
        messages: [{
          role: "user",
          content: `Implement this ${language} class.\n\nClass: ${className}\nSpec: ${spec}\n${existingImports ? `Imports: ${existingImports}` : ""}\n\nReturn ONLY the implementation code.`
        }]
      })
    });
    const result = await response.json();
    return { content: [{ type: "text", text: result.content[0].text }] };
  }
);

server.tool(
  "local_lint_check",
  {
    code: z.string(),
    language: z.string(),
    rules: z.string().optional(),
  },
  async ({ code, language, rules }) => {
    const response = await fetch("http://localhost:1234/v1/messages", {
      // ... similar pattern, prompt asks for lint issues
    });
    return { content: [{ type: "text", text: result.content[0].text }] };
  }
);

server.tool(
  "local_generate_tests",
  {
    sourceCode: z.string(),
    framework: z.string(),
    coverageGoal: z.string().optional(),
  },
  async ({ sourceCode, framework, coverageGoal }) => {
    // ... local model generates tests
  }
);
```

MCP server configuration in Automaker settings:

```json
{
  "mcpServers": {
    "local-llm-tools": {
      "command": "node",
      "args": ["/path/to/mcp-local-llm-tools/dist/index.js"]
    }
  }
}
```

**Pros**:
- Works today with zero Automaker code changes
- Frontier model decides when to delegate (intelligent routing)
- Each tool can use a different local model/endpoint
- Tools are strongly typed with schemas
- MCP servers are already supported in both chat and auto-mode

**Cons**:
- Tool calls are one-shot (no multi-turn within the local model call)
- Local model can't use tools itself (no recursive tool use)
- Frontier model must craft the delegation prompt carefully
- No streaming from local model back through the tool result

### Implementation Approach B: Subagents with Per-Provider Config (Requires Changes)

Extend `AgentDefinition` so each subagent can target a different `ClaudeCompatibleProvider`. The frontier model spawns subagents via the `Task` tool, and each subagent runs as a full Claude Code subprocess — but pointed at a different backend.

```typescript
// Extended AgentDefinition
interface AgentDefinition {
  description: string;
  prompt: string;
  tools?: string[];
  model?: string;           // Any model ID, not just Claude aliases
  providerId?: string;       // Reference a ClaudeCompatibleProvider by ID
}
```

AGENT.md file example (`~/.claude/agents/local-implementer/AGENT.md`):

```markdown
---
description: Implement a single class or function from a detailed specification
tools: Read, Write, Edit, Bash
model: qwen2.5-coder-32b
providerId: lm-studio-local
---
You are a code implementation specialist. You receive a detailed specification
for a single class or function and implement it precisely. Focus only on the
implementation — do not modify other files or add tests unless asked.
```

The frontier model would then delegate via the `Task` tool:

```
Task: "local-implementer"
Prompt: "Implement the UserRepository class per this spec: ..."
```

The changes required in Automaker:

1. **`libs/types/src/provider.ts`** — Extend `AgentDefinition.model` to accept any string, add `providerId`
2. **`apps/server/src/lib/agent-discovery.ts`** — Parse `providerId` from AGENT.md frontmatter
3. **`apps/server/src/providers/claude-provider.ts`** — When building SDK options, resolve `providerId` for each agent and inject the corresponding `ClaudeCompatibleProvider` env
4. **`apps/server/src/services/agent-executor.ts`** — Pass `agents` in `buildExecOpts()` so auto-mode can use subagents
5. **`apps/server/src/services/auto-mode/facade.ts`** — Load and pass subagents configuration

**Pros**:
- Subagents are full Claude Code sessions — multi-turn, tool use, file access
- Each subagent independently configurable (model, tools, provider)
- Frontier model controls delegation via `Task` tool
- Reuses existing AGENT.md discovery and settings UI

**Cons**:
- Requires Automaker code changes
- Each subagent subprocess adds overhead (~2-5s startup)
- Depends on how Claude Agent SDK handles `agents` with custom env per agent (may need SDK changes)
- Local model quality for multi-turn agentic work is unproven

### Implementation Approach C: A2A Protocol Integration (Future)

Google's Agent-to-Agent (A2A) protocol provides a standardized way for agents to discover, negotiate, and delegate work to each other. Local LLMs could register as A2A-compliant agents that the frontier model discovers and delegates to.

```
┌──────────────────────────┐
│ Claude Opus (A2A Client) │
│ Discovers agents via     │
│ /.well-known/agent.json  │
└───────┬──────────────────┘
        │ A2A task delegation
        ▼
┌──────────────────────────┐
│ Local A2A Agent Server   │
│ Wraps LM Studio/Ollama   │
│ Exposes agent skills:    │
│  - implement-class       │
│  - lint-check            │
│  - generate-tests        │
└──────────────────────────┘
```

This is more future-looking — A2A is still nascent — but the architecture aligns well with the concept. The MCP approach (Approach A) is essentially a simpler version of this that works today.

### Recommended Approach: MCP Now, Subagents Next

**Phase 1 (now)**: Build MCP tool servers that wrap local LLM calls for specific tasks. Test with LM Studio's Anthropic endpoint. Zero Automaker changes required. Validate which tasks local models handle well enough.

**Phase 2 (next)**: Based on Phase 1 learnings, extend `AgentDefinition` with `providerId` support and wire subagents into auto-mode. This gives local models full agentic capability (multi-turn, tools) for tasks where one-shot MCP tools aren't enough.

**Phase 3 (future)**: Standardize on A2A protocol as it matures, allowing any agent (local or remote) to register and be discovered.

### Task Suitability for Local vs Frontier Models

| Task | Frontier Model | Local LLM | Notes |
|------|---------------|-----------|-------|
| Architecture planning | Required | Unsuitable | Needs deep reasoning, full context |
| Multi-file refactoring | Required | Unsuitable | Needs broad codebase awareness |
| Code review | Required | Assist | Local can check style; frontier checks logic |
| Single class implementation | Optional | Good fit | Clear spec → implementation is mechanical |
| Test generation | Optional | Good fit | Interface → tests is pattern-matchable |
| Linting / formatting | Unnecessary | Good fit | Rules-based, local models handle well |
| Commit messages | Unnecessary | Good fit | Diff → description is straightforward |
| File descriptions | Unnecessary | Good fit | Read file → summarize |
| Documentation | Optional | Good fit | Clear input → prose output |
| Bug diagnosis | Required | Unsuitable | Needs reasoning about side effects |

---

## Gaps & Opportunities

### Gap 1: No OpenAI-Compatible API Provider

**Impact**: Cannot use local LLMs, DeepSeek, Mistral, Groq, or any OpenAI-format API directly.

**Current workaround**: Route through OpenRouter via `ClaudeCompatibleProvider` — but this only works for Claude-compatible endpoints, not native OpenAI format.

**Fix**: Create an `OpenAICompatProvider` (Pattern 1 above) or extend `ClaudeCompatibleProvider` to support OpenAI API format.

### Gap 2: Subagent Model Override is Claude-Only

**Impact**: `AgentDefinition.model` only supports `'sonnet' | 'opus' | 'haiku' | 'inherit'`. Cannot assign a Codex model, local LLM, or Gemini model to a specific subagent.

**Location**: `libs/types/src/provider.ts` line 150

**Fix**: Change to `model?: ModelId | 'inherit'` to support any registered provider model.

### Gap 3: PhaseModelConfig Doesn't Cover Agent Roles

**Impact**: The `PhaseModelConfig` covers 11 application phases but doesn't differentiate between agent roles within feature execution (e.g., planner vs implementer vs reviewer).

**Fix**: Add agent-role-level model configuration:
```typescript
interface PhaseModelConfig {
  // ... existing ...
  plannerModel?: PhaseModelEntry;
  implementerModel?: PhaseModelEntry;
  reviewerModel?: PhaseModelEntry;
  testGeneratorModel?: PhaseModelEntry;
}
```

### Gap 4: No Capability-Based Model Routing

**Impact**: Model selection is manual. Users must know which models support vision, tools, thinking, etc.

**Fix**: Add capability-based routing:
```typescript
// "Give me the cheapest model that supports tools and has >100K context"
ProviderFactory.getModelByCaps({ tools: true, minContext: 100000, tier: 'basic' });
```

### Gap 5: Feature Pipeline Steps Can't Use Different Providers

**Impact**: All pipeline steps for a feature use the same model. Cannot use a cheap model for linting and a powerful model for implementation within the same pipeline.

**Location**: `apps/server/src/services/pipeline-orchestrator.ts` line 117
```typescript
const model = resolveModelString(feature.model, DEFAULT_MODELS.claude);
```

**Fix**: Allow per-step model override in `PipelineStep` or `PipelineConfig`.

### Gap 6: Claude-Compatible Provider Limitation

**Impact**: The `ClaudeCompatibleProvider` system assumes the Anthropic API format. Providers like DeepSeek or local Ollama that use OpenAI format cannot be configured through this system.

**Fix**: Either extend `ClaudeCompatibleProvider` with an `apiFormat` field, or create a parallel `OpenAICompatibleProvider` system with its own templates and UI.

---

## Design Considerations for Planning

### 1. Provider Abstraction Completeness

The `BaseProvider` → `ProviderMessage` abstraction is well-designed. Any new provider just needs to normalize output to `ProviderMessage` format. This is a strength to preserve.

### 2. Tool Compatibility Across Providers

Different providers handle tools differently:
- **Claude SDK**: Native tool support with `tool_use`/`tool_result` blocks
- **CLI providers**: Tools provided via CLI's built-in tools or MCP servers
- **OpenAI-compat**: Function calling with `tools` parameter

An OpenAI-compatible provider would need a tool calling adapter, or could run in "text-only" mode for simple tasks.

### 3. Quality Tiers for Cost Optimization

Not all tasks need the most powerful model:

| Tier | Use Cases | Example Models |
|------|-----------|---------------|
| **Premium** | Feature implementation, spec gen | Claude Opus, GPT-5.3 Codex |
| **Standard** | Validation, planning, chat | Claude Sonnet, GPT-5.1, Gemini Pro |
| **Basic** | Enhancement, descriptions, commits | Claude Haiku, local Llama, Ollama |
| **Free** | Testing, development, cost-zero | OpenCode free-tier, local models |

### 4. Fallback Chains

If a preferred provider is unavailable (offline, rate-limited, no API key), the system should gracefully fall back:

```
Preferred: Ollama (local)
  → Fallback 1: OpenCode free-tier
  → Fallback 2: Claude Haiku
  → Fallback 3: Error with suggestion
```

### 5. Configuration UX

The settings UI already supports:
- Per-phase model selection dropdowns
- Claude-compatible provider configuration
- CLI tool status/detection

Would need to add:
- OpenAI-compatible provider configuration (URL, API key, model list)
- Local LLM provider auto-detection (Ollama, LM Studio)
- Per-pipeline-step model override
- Provider health/latency monitoring

### 6. Testing with Local Models

Local LLMs enable testing without API costs:
- `AUTOMAKER_MOCK_AGENT=true` exists for CI
- Local LLMs could serve as a middle ground: real model behavior without API costs
- Useful for development and integration testing

### 7. Security Considerations

- Local LLMs: No data leaves the machine — important for sensitive codebases
- OpenAI-compat providers: API keys stored in credentials.json (same pattern as existing)
- OpenRouter: Single key for many providers — reduces credential management

---

## Open Questions

### Claude Code-Only Strategy

1. **Is the `ClaudeCompatibleProvider` + LM Studio Anthropic endpoint sufficient for all local model use cases?**
   - Works for models that support tool use (Qwen2.5-Coder, DeepSeek-Coder)
   - May not work for simpler models that lack function calling
   - Need to test: what happens when the SDK sends tool definitions to a model that doesn't support them?

2. **Should local provider auto-detection be added?**
   - Probe `localhost:1234` (LM Studio) and `localhost:11434` (Ollama) on startup
   - Auto-create `ClaudeCompatibleProvider` entries for discovered endpoints
   - Or keep it manual — user configures providers explicitly

### Intelligent Tools (A2A) Strategy

3. **MCP tools (Phase 1) vs extended subagents (Phase 2) — when to transition?**
   - MCP tools are one-shot: send prompt, get result. No multi-turn, no tool use within the local model
   - Subagents with `providerId` would be full Claude Code sessions against the local endpoint
   - Which tasks genuinely need multi-turn? (Implementation may; linting doesn't)

4. **How should the frontier model know which tools delegate to local models?**
   - Naming convention: `local_*` prefix (e.g., `local_implement_class`, `local_lint`)
   - Tool descriptions that explicitly say "delegates to a local model"
   - Let the frontier model discover capabilities from tool schemas

5. **Quality validation for local model output?**
   - Frontier model reviews local model output before accepting it
   - Automated checks (compilation, lint pass) before returning tool result
   - Cost of re-doing work if local model output is bad vs savings from delegation

6. **Should MCP tool servers ship with Automaker or be user-configured?**
   - Built-in: Automaker includes a set of local-LLM MCP tools out of the box
   - Templates: Provide example MCP servers users can customize
   - User-only: Document the pattern, let users build their own

### Subagent Architecture

7. **How does the Claude Agent SDK handle per-agent environment overrides?**
   - The SDK's `agents` option passes `AgentDefinition` objects
   - Today there's no per-agent env/baseUrl — agents inherit the parent's env
   - Would need to either: (a) extend the SDK, or (b) have Automaker resolve `providerId` and inject env vars before passing to SDK
   - Need to verify: can `AgentDefinition` carry opaque config that the SDK passes through?

8. **Should `buildExecOpts()` always pass agents, or only when explicitly configured?**
   - Today auto-mode skips agents entirely (`agent-executor.ts:730`)
   - Passing agents adds the `Task` tool to the frontier model's toolkit
   - Risk: frontier model may over-delegate when it shouldn't
   - Mitigation: only inject agents when the user has explicitly configured subagents

### Operational

9. **How to handle local model unavailability during a feature run?**
   - Local model endpoint could be down (LM Studio not running, model not loaded)
   - MCP tool call fails → frontier model gets error → retries or does it itself
   - Subagent fails → feature execution fails (harder to recover)
   - Should there be a health check before delegating?

10. **Cost tracking for mixed-provider execution?**
    - Frontier model usage: tracked by Anthropic API (existing)
    - Local model usage: free but still worth tracking (tokens, time)
    - Could help users understand cost/quality tradeoffs of their delegation config

---

## Appendix: File Reference

### Core Provider Files

| File | Size | Description |
|------|------|-------------|
| `apps/server/src/providers/provider-factory.ts` | 10KB | Registry-based provider routing |
| `apps/server/src/providers/base-provider.ts` | 2KB | Abstract base class |
| `apps/server/src/providers/cli-provider.ts` | 19KB | Abstract CLI provider base |
| `apps/server/src/providers/claude-provider.ts` | 16KB | Claude Agent SDK integration |
| `apps/server/src/providers/codex-provider.ts` | 42KB | OpenAI Codex CLI integration |
| `apps/server/src/providers/cursor-provider.ts` | 40KB | Cursor CLI integration |
| `apps/server/src/providers/opencode-provider.ts` | 50KB | OpenCode CLI (multi-provider gateway) |
| `apps/server/src/providers/gemini-provider.ts` | 13KB | Gemini CLI integration |
| `apps/server/src/providers/copilot-provider.ts` | 31KB | GitHub Copilot SDK integration |
| `apps/server/src/providers/simple-query-service.ts` | 10KB | Simplified query interface |
| `apps/server/src/providers/tool-normalization.ts` | 4KB | Cross-provider tool normalization |

### Type Definitions

| File | Description |
|------|-------------|
| `libs/types/src/provider.ts` | `ExecuteOptions`, `ProviderMessage`, `AgentDefinition` |
| `libs/types/src/settings.ts` | `PhaseModelConfig`, `ClaudeCompatibleProvider`, `ModelProvider` |
| `libs/types/src/model.ts` | `ModelId`, `CLAUDE_MODEL_MAP`, `CODEX_MODEL_MAP` |
| `libs/types/src/provider-utils.ts` | `isCursorModel()`, `isCodexModel()`, etc. |
| `libs/types/src/feature.ts` | `Feature` interface with per-feature model field |

### Configuration & SDK

| File | Description |
|------|-------------|
| `apps/server/src/lib/sdk-options.ts` | SDK options factory (per-use-case presets) |
| `libs/model-resolver/` | Model alias → full model ID resolution |
| `apps/server/src/services/pipeline-orchestrator.ts` | Pipeline step execution |
| `apps/server/src/services/agent-executor.ts` | Core agent execution engine |
| `apps/server/src/services/execution-service.ts` | Feature execution lifecycle |

### Subagent & MCP Infrastructure

| File | Description |
|------|-------------|
| `apps/server/src/lib/agent-discovery.ts` | Discovers AGENT.md files from `~/.claude/agents/` and `.claude/agents/` |
| `apps/server/src/lib/settings-helpers.ts:418` | `getCustomSubagents()` — merges global + project subagents |
| `apps/server/src/lib/settings-helpers.ts:705` | `getProviderByModelId()` — resolves model → ClaudeCompatibleProvider |
| `apps/server/src/services/agent-service.ts:391` | Loads subagents and passes as `agents` in chat mode |
| `apps/server/src/services/agent-executor.ts:730` | `buildExecOpts()` — omits `agents` (auto-mode gap) |
| `apps/server/src/services/auto-mode/facade.ts:285` | Auto-mode execution options — no `agents` field |
| `apps/server/src/routes/settings/routes/discover-agents.ts` | API route for agent discovery |

### Documentation

| File | Description |
|------|-------------|
| `docs/server/providers.md` | Provider architecture reference (includes "Adding New Providers" guide) |
