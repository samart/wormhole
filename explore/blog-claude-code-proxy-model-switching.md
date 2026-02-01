# How Claude Code Picks Its Model — and Why Your Proxy Alias Matters

If you're running Claude Code CLI behind a corporate LLM proxy or gateway, you've probably hit a subtle problem: everything *seems* to work, but outputs get truncated, the model claims wrong training dates, or features silently disappear. The root cause is almost always the same — your proxy's model alias doesn't contain the right substrings.

This post explains how model switching actually works inside Claude Code, why the CLI cares about your model ID string, and how to configure proxy aliases that don't silently degrade your experience.

---

## How Model Switching Works in Claude Code

### The Three Layers of Model Selection

Claude Code resolves which model to use through a priority chain. From highest to lowest:

1. **`/model` command** — mid-session interactive switch (or Option+P / Alt+P)
2. **`--model` CLI flag** — set at startup (`claude --model opus`)
3. **`ANTHROPIC_MODEL` env var** — overrides config file
4. **`model` field in `settings.json`** — persistent config
5. **Built-in default** — Sonnet for most users, Opus for Max/Team/Pro plans

At every layer, you can use either a **shorthand alias** or a **full model ID**.

### Shorthands: The Easy Path

The CLI recognizes a small set of aliases:

| Alias | Resolves to (Anthropic Direct) |
|---|---|
| `sonnet` | `claude-sonnet-4-5-20250929` |
| `opus` | `claude-opus-4-5-20251101` |
| `haiku` | `claude-haiku-4-5-20251001` |
| `sonnet[1m]` | `claude-sonnet-4-5-20250929[1m]` |
| `opusplan` | Opus in plan mode, Sonnet otherwise |

When you type `/model opus`, the CLI maps it through a resolver that checks your provider (Anthropic Direct, Bedrock, Vertex, Foundry) and returns the appropriate full ID. On Bedrock, for example, `opus` resolves to `us.anthropic.claude-opus-4-1-20250805-v1:0` instead.

### The Override Env Vars

Before the resolver hits its internal registry, it checks three environment variables:

```bash
ANTHROPIC_DEFAULT_OPUS_MODEL    # what "opus" resolves to
ANTHROPIC_DEFAULT_SONNET_MODEL  # what "sonnet" resolves to
ANTHROPIC_DEFAULT_HAIKU_MODEL   # what "haiku" resolves to
```

If set, these **completely replace** the built-in mapping. This is how you wire shorthands to your proxy's model aliases.

### What Happens to Non-Shorthand Strings

Anything the CLI doesn't recognize as a shorthand passes through **unchanged**. No parsing, no validation, no transformation. If you set:

```bash
export ANTHROPIC_MODEL="acme-claude-opus-4-5"
```

The CLI sends the literal string `"acme-claude-opus-4-5"` as the `model` parameter in the API request. Your proxy receives that string and is responsible for routing it to the right upstream model.

This passthrough behavior is what makes proxy setups possible — but it's also the source of the capability detection problem.

---

## The Capability Detection Problem

Here's where it gets interesting. The resolved model ID string serves **two purposes**:

1. It's sent to the API as the `model` parameter (for routing)
2. It's checked locally with `.includes()` substring matching (for feature gating)

The CLI uses substring checks on the model ID to decide:

- How many output tokens to allow (32K vs 64K)
- What knowledge cutoff date to include in the system prompt
- Whether to enable extended thinking
- Which beta features to request
- Whether plan mode uses special behavior

**There is no API call to discover capabilities.** The CLI literally does `modelId.includes("opus-4-5")` and branches on the result.

### The Complete Substring Check Catalog

Here's every check the CLI runs against your model ID string:

**Max output tokens** (checked in this order, first match wins):

```
modelId.includes("opus-4-5")              → 64K tokens
modelId.includes("opus-4")                → 32K tokens
modelId.includes("sonnet-4") ||
  modelId.includes("haiku-4")             → 64K tokens
(default)                                  → 32K tokens
```

**Knowledge cutoff** (checked in this order):

```
modelId.includes("claude-opus-4-5")        → "May 2025"
modelId.includes("claude-haiku-4")         → "February 2025"
modelId.includes("claude-opus-4") ||
  modelId.includes("claude-sonnet-4-5") ||
  modelId.includes("claude-sonnet-4")      → "January 2025"
(default)                                   → null (omitted from prompt)
```

**Haiku detection** — `modelId.includes("haiku")`:
- Disables extended thinking
- Removes certain beta feature headers
- Skips prompt cache optimization
- Silently upgrades to Sonnet in plan mode

**Opus detection** — `modelId.includes("opus")`:
- Enables opus-specific plan mode gating

### Why This Design?

Substring matching is fast, works across all providers (Bedrock IDs like `us.anthropic.claude-opus-4-5-v1:0` still contain the right substrings), and doesn't require a model metadata API. It's a pragmatic choice — but it means the model ID isn't just a routing label. It's a capability descriptor.

---

## Setting Up Proxy Model IDs (The Right Way)

### The Golden Rule

> **Your proxy's model alias must contain the Anthropic model family string, using dashes (not dots).**

If `"claude-opus-4-5"` appears anywhere in your alias as a substring, every capability check will pass. The CLI doesn't care about prefixes, suffixes, or surrounding text.

### Alias Patterns That Work

**Best: Full dated ID**
```
claude-opus-4-5-20251101
claude-sonnet-4-5-20250929
```

**Good: Foundry-style (no date)**
```
claude-opus-4-5
claude-sonnet-4-5
```

**Good: Company-prefixed or suffixed**
```
acme-claude-opus-4-5
claude-opus-4-5-production
myorg/claude-sonnet-4-5-fast
```

All of these contain the right substrings. The CLI's `.includes()` check finds `"opus-4-5"` in all of them.

### Patterns That Break

| Alias | Problem |
|---|---|
| `claude-opus-4.5` | **Dot** breaks `"opus-4-5"` match → classified as Opus 4.0 (32K tokens, wrong cutoff) |
| `company-smart` | No Anthropic substrings → 32K tokens, no cutoff, no family detection |
| `claude45` | No family substrings → everything falls to defaults |
| `fast-haiku-v2` | Contains `"haiku"` accidentally → thinking disabled even if it's not haiku |
| `opus-latest` | Missing `"claude-"` prefix → cutoff check fails |

### The Dot-vs-Dash Trap

This is the most common misconfiguration. It looks right but isn't:

```
claude-opus-4.5   ← contains "opus-4" but NOT "opus-4-5"
claude-opus-4-5   ← contains both "opus-4-5" and "opus-4"
```

The CLI checks `"opus-4-5"` first (→ 64K tokens). If that fails, it falls through to `"opus-4"` (→ 32K tokens). A dot between `4` and `5` prevents the more-specific check from matching, and you silently get the wrong behavior.

| Alias | Tokens | Cutoff | Classified As |
|---|---|---|---|
| `claude-opus-4-5` | 64K | May 2025 | Opus 4.5 |
| `claude-opus-4.5` | 32K | Jan 2025 | Opus 4.0 |

One character difference. Half the output tokens. Wrong training date. No error message.

---

## Putting It Together: Proxy Configuration Recipes

### Recipe 1: Transparent Proxy (Simplest)

Your proxy forwards to `api.anthropic.com` and passes model IDs through unchanged.

```bash
export ANTHROPIC_BASE_URL="https://proxy.yourcompany.com/v1"
# That's it. Standard shorthands resolve to standard Anthropic IDs.
# /model opus → sends "claude-opus-4-5-20251101" to your proxy
```

### Recipe 2: Proxy With Custom Aliases (Recommended)

Your proxy has its own model names. Embed the Anthropic family string in them:

```bash
# Proxy aliases (configured on the proxy side):
#   acme-claude-opus-4-5    → routes to Opus 4.5
#   acme-claude-sonnet-4-5  → routes to Sonnet 4.5
#   acme-claude-haiku-4-5   → routes to Haiku 4.5

# Client-side config:
export ANTHROPIC_BASE_URL="https://llm-proxy.yourcompany.com/v1"
export ANTHROPIC_DEFAULT_OPUS_MODEL="acme-claude-opus-4-5"
export ANTHROPIC_DEFAULT_SONNET_MODEL="acme-claude-sonnet-4-5"
export ANTHROPIC_DEFAULT_HAIKU_MODEL="acme-claude-haiku-4-5"
```

Now `/model opus` resolves to `"acme-claude-opus-4-5"`, which:
1. Gets sent to your proxy (which knows how to route `acme-claude-opus-4-5`)
2. Contains `"claude-opus-4-5"` so all CLI capability checks pass

### Recipe 3: LiteLLM Gateway

```bash
# Anthropic pass-through endpoint
export ANTHROPIC_BASE_URL="https://litellm-server:4000/anthropic"
export ANTHROPIC_AUTH_TOKEN="sk-litellm-key"

# If LiteLLM uses its own model names, map them:
export ANTHROPIC_DEFAULT_SONNET_MODEL="litellm-claude-sonnet-4-5"
export ANTHROPIC_DEFAULT_HAIKU_MODEL="litellm-claude-haiku-4-5"
```

### Recipe 4: Aliases You Can't Change

Your proxy uses opaque names like `company-smart` and `company-fast`. You can't rename them.

```bash
# Map shorthands to your proxy aliases
export ANTHROPIC_DEFAULT_OPUS_MODEL="company-smart"
export ANTHROPIC_DEFAULT_HAIKU_MODEL="company-fast"

# Manually fix the token limit (partial workaround)
export CLAUDE_CODE_MAX_OUTPUT_TOKENS=64000
```

This fixes truncation but **not** knowledge cutoff or family-specific behavior. The only complete fix is changing the proxy aliases to contain the Anthropic model name. Point your proxy admin to the alias naming guidelines above.

### Recipe 5: Persistent Config via settings.json

Instead of shell exports, set everything in your user settings:

```json
// ~/.claude/settings.json
{
  "model": "opus",
  "env": {
    "ANTHROPIC_BASE_URL": "https://proxy.yourcompany.com/v1",
    "ANTHROPIC_DEFAULT_OPUS_MODEL": "acme-claude-opus-4-5",
    "ANTHROPIC_DEFAULT_SONNET_MODEL": "acme-claude-sonnet-4-5",
    "ANTHROPIC_DEFAULT_HAIKU_MODEL": "acme-claude-haiku-4-5"
  }
}
```

Or in project-level settings (`.claude/settings.json`) to share with your team.

### Recipe 6: Controlling Sub-Agent Models

Sub-agents spawned by the Task tool resolve models independently. Force all of them to a specific model:

```bash
export CLAUDE_CODE_SUBAGENT_MODEL="claude-sonnet-4-5"
```

This overrides any per-agent model configuration. Useful when your proxy only supports certain models.

---

## Environment Variable Quick Reference

| Variable | Purpose | Affects capability detection? |
|---|---|---|
| `ANTHROPIC_BASE_URL` | Route requests to your proxy | No — routing only |
| `ANTHROPIC_MODEL` | Set model ID directly | Yes — becomes the resolved string |
| `ANTHROPIC_DEFAULT_OPUS_MODEL` | Override what `opus` resolves to | Yes — value is substring-checked |
| `ANTHROPIC_DEFAULT_SONNET_MODEL` | Override what `sonnet` resolves to | Yes |
| `ANTHROPIC_DEFAULT_HAIKU_MODEL` | Override what `haiku` resolves to | Yes |
| `CLAUDE_CODE_SUBAGENT_MODEL` | Force model for all sub-agents | Yes |
| `CLAUDE_CODE_MAX_OUTPUT_TOKENS` | Override max output tokens | No — bypasses detection |

---

## Common Gotchas

### Accidental substring matches

Any alias containing `"haiku"` or `"opus"` triggers family detection — even accidentally:

```
"super-haiku-fast"       → treated as haiku → thinking disabled
"opus-router"            → treated as opus  → plan gating activates
"not-a-haiku-really"     → treated as haiku
```

### The `[1m]` suffix

When a user selects `sonnet[1m]`, the CLI appends `[1m]` to the model ID. Your proxy must either strip it or handle it as a routing hint. It doesn't affect capability detection but will cause API errors if your proxy chokes on it.

### Silent degradation

When capability detection fails, there's no warning in the UI. The status bar shows your literal alias string (`company-smart`) with no indication that features are degraded. The only symptoms are truncated outputs and incorrect model behavior.

### Haiku in plan mode

When haiku is the active model and you enter plan mode, the CLI silently upgrades to Sonnet. If your alias accidentally matches `"haiku"`, your proxy will receive sonnet requests it might not expect.

---

## Decision Flowchart

```
Using standard shorthands? (opus / sonnet / haiku)
│
├─ YES → Using ANTHROPIC_BASE_URL for a proxy?
│        ├─ YES → Proxy accepts standard Anthropic model IDs?
│        │        ├─ YES → Done. No config needed.
│        │        └─ NO  → Set ANTHROPIC_DEFAULT_*_MODEL env vars.
│        │                 Ensure aliases contain family substrings.
│        └─ NO  → Direct API. Everything works out of the box.
│
└─ NO → Does your model ID contain the family string with DASHES?
         (e.g., "claude-opus-4-5" as a substring)
         ├─ YES → All capability checks will pass.
         └─ NO  → Capability detection is degraded.
                   Fix: change alias, or use CLAUDE_CODE_MAX_OUTPUT_TOKENS
                   as a partial workaround.
```

---

*Based on Claude Code CLI v2.1.29 behavior. Substring detection patterns may change in future versions — verify against actual CLI behavior for production configurations.*
