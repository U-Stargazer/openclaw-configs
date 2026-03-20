# OpenClaw Community Configs

Two production-ready `openclaw.json` configurations — one for daily free-tier use, one for a full hybrid local + cloud stack. Both have been security-hardened, performance-tuned, and compared against every publicly available config before release.

---

## Which config should I use?

| | Daily (free) | Advanced (hybrid) |
|---|---|---|
| **File** | `openclaw_daily_public.json` | `openclaw_advanced_public.json` |
| **Cost** | $0 — Groq + Gemini free tiers only | Mostly free local, pay-per-call cloud |
| **Hardware** | Any laptop | 16GB RAM recommended |
| **Ollama required** | Yes | Yes |
| **Cloud agents** | No | 6 (Claude, GPT, Gemini, Kimi, Perplexity, Grok) |
| **Best for** | Students, everyday use, tight budgets | Developers, power users, multi-agent workflows |

---

## Quick start

### 1. Install dependencies

```bash
# Ollama (runs models locally)
curl -fsSL https://ollama.com/install.sh | sh

# Pull the models used by your chosen config
# Daily config:
ollama pull phi4
ollama pull qwen3:7b
ollama pull llama3.3

# Advanced config (additional models):
ollama pull minimax-m2.5
ollama pull deepseek-v3.2
ollama pull qwen3-coder
ollama pull glm-4.7-flash
ollama pull llava
```

### 2. Copy the config

```bash
# Pick one:
cp openclaw_daily_public.json    ~/.openclaw/openclaw.json
cp openclaw_advanced_public.json ~/.openclaw/openclaw.json
```

### 3. Add your API keys

Create a `.env` file in your OpenClaw config directory — **never paste keys into the JSON**:

```env
# Daily config — both are free tiers, no credit card needed
GROQ_API_KEY=gsk_...        # → console.groq.com
GEMINI_API_KEY=AIza...      # → aistudio.google.com/app/apikey

# Advanced config — add whichever cloud agents you want
ANTHROPIC_API_KEY=sk-ant-...   # → console.anthropic.com
OPENAI_API_KEY=sk-...          # → platform.openai.com/api-keys
GOOGLE_API_KEY=AIza...         # → aistudio.google.com/app/apikey
MOONSHOT_API_KEY=sk-...        # → platform.moonshot.ai  (Kimi)
XAI_API_KEY=xai-...            # → console.x.ai          (Grok)
PERPLEXITY_API_KEY=pplx-...    # → perplexity.ai/settings/api
```

### 4. Generate TLS certs

```bash
openclaw tls generate
```

### 5. Start the gateway

```bash
openclaw gateway start
```

---

## What's in these configs

### Security (both configs)

- Gateway bound to `127.0.0.1` with TLS 1.3 — never exposed to the internet
- 30+ blocked shell patterns covering pipe-to-shell, eval, base64 decode, PowerShell bypass, reverse shells, and Windows LOLBins
- Immutable audit log with SHA-256 integrity checks
- Docker sandbox with all capabilities dropped, `noNewPrivileges`, read-only root, `nobody:nogroup` user
- 13 sensitive pattern regexes scanning file content for AWS keys, GitHub tokens, Stripe keys, MongoDB URIs, and more
- Blocked file extensions: `.env`, `.pem`, `.ssh`, `.key`, `.pfx`, `.ovpn`, `.tfvars`, `.npmrc`, and more
- Prompt injection detection + jailbreak refusal patterns

### Daily config highlights

- **100% free** — Groq (Llama 3.1/3.3) + Gemini 2.5 Flash free tiers + local Ollama
- `maxCostPerSession: 0` — hard enforces zero cloud spend
- Hardware-conscious: conservative CPU quota, OOM handler, WSL2 memory reclaim, storage tiering across SSD + HDD
- `keepAlive: 10m` — prevents Ollama cold-start reloads every few minutes
- HEARTBEAT.md template — passive health check written every 6 hours
- Study slot uses Gemini 2.5 Flash (free, far better reading comprehension than local models for long documents)

### Advanced config highlights

- **5 local sub-agents**, each tuned to a benchmark-backed model:

  | Agent | Model | Benchmark | Why |
  |---|---|---|---|
  | `swe` | minimax-m2.5 | SWE-bench 80.2 | Highest open-model score for resolving real GitHub issues |
  | `math-reasoning` | phi4 | MATH 80.4% | Best score per GB of RAM for logic tasks |
  | `fast-autocomplete` | qwen3:7b | HumanEval 76.0 | Fastest sub-8B model, highest HumanEval in class |
  | `security` | deepseek-v3.2 | Long-context sweep | Sparse Attention + agentic tool-use training |
  | `docs-writer` | llama3.3 | Prose quality | Best long-form narrative clarity of local open models |

- **6 cloud agents** with per-agent privacy guards — each has `allowProprietaryCode: false` and `allowSecrets: false`
- **12-rule orchestrator routing table** — matches task intent to the best agent automatically:
  - Secrets detected → always local security agent
  - Live CVE / version info needed → Perplexity Sonar (grounded web results)
  - Document > 80K tokens → Gemini (1M context + caching)
  - Multi-file refactor → Claude Sonnet or MiniMax (based on sensitivity)
  - Math / algorithms → Phi-4 (local, best MATH score)
  - Batch / parallel tasks → Kimi K2.5 (10× cheaper than Opus for bulk work)
  - Small edit / < 2K tokens → Qwen3 7B (fastest, never worth a cloud call)
- **Privacy gate** — applied before every cloud route; blocks cloud if secrets, PII, or proprietary code detected
- **Cost guards** — warns at $0.50/call, blocks at $5.00/call, daily budget cap of $10, falls back to local when exhausted

---

## Customise for your hardware

### Daily config — key limits to adjust

```jsonc
// In system.performance:
"cpuQuota": 0.5,          // raise if your CPU is faster
"parallelAgents": 1,       // raise to 2 on newer machines

// In sandbox.docker:
"memoryLimitMB": 1536,    // raise to 2048+ if you have more RAM
"cpuLimit": 1.8,

// In models.providers.ollama:
"numThread": 4             // set to your physical core count (run: nproc --all)
```

### Advanced config — key limits to adjust

```jsonc
// In agents.defaults.limits:
"concurrentAgents": 4,    // lower to 2 if you have less than 16GB RAM
"tokenBudget": 250000,

// In sandbox.docker:
"memoryLimitMB": 2560,
"cpuLimit": 2.5
```

---

## WSL2 users (Windows)

The `%USERNAME%` in all paths is a Windows environment variable — OpenClaw expands it automatically. You don't need to replace it manually.

If WSL2 is consuming too much RAM over time, the configs include:

```jsonc
"wsl2": {
  "memoryReclaim": true,
  "pageCacheTimeout": 180,
  "autoShrinkVhd": true
}
```

This is not standard WSL2 config — OpenClaw applies these settings to its WSL2 runtime on startup.

---

## What these configs don't include (yet)

These are intentionally left out so you can add your own:

- **Messaging channels** (Telegram, WhatsApp, Discord) — add a `channels` block with your bot token
- **Workspace identity files** (SOUL.md, USER.md, MEMORY.md) — the community's 3-tier memory architecture
- **Gateway auth token** — add `gateway.auth.mode: "token"` and a strong random string before exposing to any network
- **Cron / scheduled tasks** — autonomous workflows run on a schedule

---

## Security notes

- **Never commit your `.env` file** — add it to `.gitignore` immediately
- Cloud agents send data to external providers. The privacy gate blocks cloud routing if secrets or proprietary code are detected, but you are responsible for what you send
- Cloud pricing changes frequently — verify rates at each provider's website before use
- The `imageModel` schema key is the correct OpenClaw key for vision tasks. If your version uses a different key, run `openclaw models set-image google/gemini-2.5-flash` to set it correctly

---

## Contributing

Found a better model for a specific task? Benchmarks updated? Open a PR with the rationale and benchmark source. All model choices in these configs include a `//rationale` comment explaining why that model was picked over alternatives.

---

## Licence

MIT — use freely, modify for your setup, share improvements back.
