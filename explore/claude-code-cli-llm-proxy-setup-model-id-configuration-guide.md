# Claude Code CLI — LLM Proxy Setup & Model ID Configuration Guide

> How to configure your company's LLM proxy aliases so Claude Code CLI correctly detects model capabilities, enables the right features, and avoids silent degradation.

**Source:** Analysis of Claude Code CLI bundle v2.1.29. Substring detection patterns may change in future versions.

---

## 1. TL;DR — The Golden Rule

> **Your proxy's model alias must contain the Anthropic model family string using dashes, not dots.**

The CLI uses **substring matching** (`.includes()`) on the model ID to determine capabilities. If the right substrings aren't present, features silently degrade.

### Minimum viable aliases

| Model | Minimum alias that works | What it must contain |
|---|---|---|
| Opus 4.5 | `claude-opus-4-5` | The substring `"claude-opus-4-5"` with dashes |
| Sonnet 4.5 | `claude-sonnet-4-5` | The substring `"claude-sonnet-4-5"` with dashes |
| Haiku 4.5 | `claude-haiku-4-5` | The substring `"claude-haiku-4-5"` with dashes |
| Sonnet 4.0 | `claude-sonnet-4` | The substring `"claude-sonnet-4"` |
| Opus 4.1 | `claude-opus-4-1` | The substring `"claude-opus-4-1"` |

### What breaks without these substrings

| Missing substring | Consequence |
|---|---|
| `"opus-4-5"` | Max output tokens drops from 64K → 32K |
| `"claude-opus-4-5"` | Knowledge cutoff wrong ("January 2025" instead of "May 2025") |
| `"haiku"` | Extended thinking enabled (wrong for haiku), all betas applied |
| `"opus"` | Plan mode gating doesn't activate |
| All family strings | 32K tokens, no cutoff, no family-specific behavior |

---

## 2. Recommended Proxy Alias Patterns

### Tier 1: Full dated ID (best)

```
claude-opus-4-5-20251101
claude-sonnet-4-5-20250929
claude-haiku-4-5-20251001
```

All capability checks work. Most specific. Matches what Anthropic's own resolver produces.

### Tier 2: Foundry-style / no date (good)

```
claude-opus-4-5
claude-sonnet-4-5
claude-haiku-4-5
```

All checks work. Shorter. Forward-compatible with new date-stamped versions.

### Tier 3: Company-prefixed (good)

```
mycompany-claude-opus-4-5
mycompany/claude-opus-4-5
acme-claude-sonnet-4-5-fast
```

All checks work — the Anthropic model name is embedded as a substring. Your prefix is ignored by the CLI's substring matching. This is the recommended pattern when you need custom routing identifiers.

### Tier 4: Company-suffixed (good)

```
claude-opus-4-5-mycompany
claude-opus-4-5-fast
claude-sonnet-4-5-production
```

All checks work — the Anthropic model name is still present as a leading substring.

### Patterns to AVOID

| Pattern | Problem | Impact |
|---|---|---|
| `claude-opus-4.5` | **Dot** breaks `"opus-4-5"` match | 32K tokens instead of 64K, wrong cutoff |
| `claude45` | No family substrings | No capability detection at all |
| `opus-latest` | Missing `claude-` prefix | Cutoff and family slug checks fail |
| `my-fast-model` | No model family info | Everything falls to defaults |
| `company-smart` | No Anthropic substrings | No detection, 32K tokens |
| `gpt-4o` | Wrong API protocol entirely | Incompatible request format |
| `fast-haiku-v2` | Contains `"haiku"` accidentally | Loses thinking, fewer betas — even if it's not haiku |

---

## 3. Proxy Setup — Common Scenarios

### Scenario A: Reverse proxy to Anthropic API

Your proxy forwards to `api.anthropic.com` but clients use a different base URL. Model IDs pass through unchanged.

```bash
export ANTHROPIC_BASE_URL="https://proxy.yourcompany.com/v1"
# Use standard shorthands: /model opus, /model sonnet, /model haiku
# They resolve to standard Anthropic IDs and get sent to your proxy URL
```

This is the simplest setup. No alias changes needed. The CLI resolves shorthands to canonical IDs, and your proxy forwards them to Anthropic.

### Scenario B: Proxy with custom model aliases (recommended fix)

Your proxy maps custom names to Anthropic models. **Embed the Anthropic model name in your alias:**

| Proxy alias | Routes to |
|---|---|
| `acme-claude-opus-4-5` | Opus 4.5 |
| `acme-claude-sonnet-4-5` | Sonnet 4.5 |
| `acme-claude-haiku-4-5` | Haiku 4.5 |

Then configure the CLI to use your aliases via environment variables:

```bash
export ANTHROPIC_BASE_URL="https://llm-proxy.yourcompany.com/v1"
export ANTHROPIC_DEFAULT_OPUS_MODEL="acme-claude-opus-4-5"
export ANTHROPIC_DEFAULT_SONNET_MODEL="acme-claude-sonnet-4-5"
export ANTHROPIC_DEFAULT_HAIKU_MODEL="acme-claude-haiku-4-5"
```

Now `/model opus` resolves to `"acme-claude-opus-4-5"`, which:
1. Gets sent to your proxy (which knows how to route it)
2. Contains `"claude-opus-4-5"` so all CLI capability checks pass

### Scenario C: Proxy aliases you can't change

Your proxy uses aliases like `company-fast` and `company-smart` and you can't rename them.

**Option 1 — Override shorthands (partial fix):**
```bash
export ANTHROPIC_DEFAULT_OPUS_MODEL="company-smart"
export ANTHROPIC_DEFAULT_HAIKU_MODEL="company-fast"
```
`/model opus` now sends `"company-smart"` to the proxy. But the CLI still won't detect it as opus-class — you lose capability-specific behavior (32K tokens, no cutoff, no plan gating).

**Option 2 — Override shorthands + override max tokens (better):**
```bash
export ANTHROPIC_DEFAULT_OPUS_MODEL="company-smart"
export CLAUDE_CODE_MAX_OUTPUT_TOKENS=64000
```
This fixes the token limit but not the knowledge cutoff or family detection.

**Option 3 — Request alias changes from your proxy admin.** This is the only complete fix. Point them to §2 of this document.

### Scenario D: Pinning to a specific model version

Override the shorthand to point to a specific dated ID:

```bash
# Pin "sonnet" to Sonnet 4.0 instead of the default 4.5
export ANTHROPIC_DEFAULT_SONNET_MODEL="claude-sonnet-4-20250514"

# Or set a single model for everything
export ANTHROPIC_MODEL="claude-opus-4-5-20251101"
```

### Scenario E: Controlling sub-agent models

Sub-agents (spawned via the Task tool) resolve models independently. Force them all to a specific model:

```bash
export CLAUDE_CODE_SUBAGENT_MODEL="claude-sonnet-4-5"
# All sub-agents use Sonnet 4.5 regardless of their configured defaults
```

Shorthands work here too: `export CLAUDE_CODE_SUBAGENT_MODEL="sonnet"`

---

## 4. Environment Variable Quick Reference

| Variable | Purpose | Respects substring matching? |
|---|---|---|
| `ANTHROPIC_BASE_URL` | Point CLI to your proxy endpoint | **No** — routing only |
| `ANTHROPIC_MODEL` | Set model ID directly (overrides config) | **Yes** — value becomes the resolved string |
| `ANTHROPIC_DEFAULT_OPUS_MODEL` | Override what `/model opus` resolves to | **Yes** — value must contain family substrings |
| `ANTHROPIC_DEFAULT_SONNET_MODEL` | Override what `/model sonnet` resolves to | **Yes** |
| `ANTHROPIC_DEFAULT_HAIKU_MODEL` | Override what `/model haiku` resolves to | **Yes** |
| `CLAUDE_CODE_SUBAGENT_MODEL` | Force model for all sub-agents | **Yes** — goes through resolver |
| `CLAUDE_CODE_MAX_OUTPUT_TOKENS` | Override max output tokens directly | **No** — bypasses model detection |

**Priority order for model selection:**
```
1. /model interactive selection (highest)
2. CLI --model arg
3. ANTHROPIC_MODEL env var
4. settings.model in config
5. Default based on plan tier (lowest)
```

---

## 5. Quick Decision Flowchart

```
Are you using standard shorthands? (opus / sonnet / haiku / opusplan)
│
├─ YES → Are you using ANTHROPIC_BASE_URL to point to a proxy?
│        ├─ YES → Does your proxy accept standard Anthropic model IDs?
│        │        ├─ YES → ✅ You're done. No alias changes needed.
│        │        └─ NO  → Set ANTHROPIC_DEFAULT_*_MODEL env vars
│        │                 to your proxy's aliases. Make sure aliases
│        │                 contain the Anthropic family string (see §2).
│        └─ NO  → ✅ Direct Anthropic API. Everything works.
│
└─ NO → Does your model ID contain the family string with DASHES?
         (e.g., "claude-opus-4-5" somewhere in the string)
         │
         ├─ YES → ✅ All capability checks will match.
         │
         └─ NO → Does it at least contain the broad family word?
                  ("opus", "haiku", or "sonnet-4" / "haiku-4")
                  │
                  ├─ YES → ⚠️ Partial detection only:
                  │         • Family-level gating works (betas, thinking)
                  │         • Version-specific features WRONG
                  │           (max tokens, cutoff, family slug)
                  │         Fix: CLAUDE_CODE_MAX_OUTPUT_TOKENS=64000
                  │
                  └─ NO → ❌ No capability detection at all.
                           • 32K max tokens
                           • No knowledge cutoff
                           • All betas enabled (even if model doesn't support them)
                           Fix: Change proxy alias to include family string,
                           OR use ANTHROPIC_DEFAULT_*_MODEL with proper names.
```

---

## 6. Edge Cases & Gotchas for Proxy Admins

### Accidental substring matches

Any alias containing `"haiku"` or `"opus"` — even unintentionally — triggers family detection:

```
"super-haiku-fast"        → CLI treats as haiku → no thinking, fewer betas
"opus-router"             → CLI treats as opus-class → plan gating activates
"not-a-haiku-really"      → CLI treats as haiku!
```

There is no opt-out. Choose aliases that don't accidentally contain these words.

### The dot-vs-dash trap (most common mistake)

`claude-opus-4.5` looks right but breaks version detection. The CLI checks for `"opus-4-5"` (with dashes). A dot after `4` means `"opus-4-5"` fails but `"opus-4"` matches — silently classifying it as Opus 4.0:

| Alias | Max tokens | Cutoff | Classification |
|---|---|---|---|
| `claude-opus-4-5` | 64K ✅ | May 2025 ✅ | Opus 4.5 ✅ |
| `claude-opus-4.5` | 32K ❌ | Jan 2025 ❌ | Opus 4.0 ❌ |

### The `[1m]` suffix and proxy routing

When a user selects `sonnet[1m]`, the CLI appends `[1m]` to the model ID: `"claude-sonnet-4-5-20250929[1m]"`. Your proxy must either:
1. Strip `[1m]` before forwarding to the Anthropic API
2. Recognize it as a routing hint for 1M context window

The suffix doesn't affect CLI capability detection but may cause API errors if your proxy doesn't handle it.

### Silent haiku → Sonnet upgrade in plan mode

When haiku is selected and the CLI enters plan mode, it silently switches to Sonnet. If your alias accidentally contains "haiku", the CLI may switch to a different model without notification.

### 32K token limit for unrecognized aliases

Unrecognized aliases get 32K max output tokens — half the 64K available for properly-identified models. This silent degradation can cause truncated outputs. Override with `CLAUDE_CODE_MAX_OUTPUT_TOKENS=64000` if needed.

### No knowledge cutoff for unrecognized aliases

The system prompt won't include the model's knowledge cutoff date. This can lead to the model making incorrect claims about how recent its training data is.

### Display gives no warning

When using a passthrough alias, the UI just shows your literal string (e.g., `"company-smart"`). There's no visual indicator that capability detection is degraded.

---

## 7. Compatibility Matrix

Full reference showing how each model ID format interacts with CLI capability checks.

✅ = correct  ❌ = wrong  ⚠️ = fragile/coincidental

| Model ID Pattern | Class detect | Max tokens | Cutoff | Family slug | Thinking | **Overall** |
|---|---|---|---|---|---|---|
| **Shorthands** | | | | | | |
| `opus` | ✅ opus | ✅ 64K | ✅ May 2025 | ✅ opus-4-5 | ✅ yes | ✅ |
| `sonnet` | ✅ — | ✅ 64K | ✅ Jan 2025 | ✅ sonnet-4-5 | ✅ yes | ✅ |
| `haiku` | ✅ haiku | ✅ 64K | ✅ Feb 2025 | ✅ haiku-4-5 | ✅ no | ✅ |
| **Full dated IDs** | | | | | | |
| `claude-opus-4-5-20251101` | ✅ opus | ✅ 64K | ✅ May 2025 | ✅ opus-4-5 | ✅ yes | ✅ |
| `claude-opus-4-1-20250805` | ✅ opus | ✅ 32K | ✅ Jan 2025 | ✅ opus-4-1 | ✅ yes | ✅ |
| `claude-haiku-4-5-20251001` | ✅ haiku | ✅ 64K | ✅ Feb 2025 | ✅ haiku | ✅ no | ✅ |
| `claude-sonnet-4-5-20250929` | ✅ — | ✅ 64K | ✅ Jan 2025 | ✅ sonnet-4-5 | ✅ yes | ✅ |
| `claude-sonnet-4-20250514` | ✅ — | ✅ 64K | ✅ Jan 2025 | ✅ sonnet-4 | ✅ yes | ✅ |
| **Foundry-style (no date)** | | | | | | |
| `claude-opus-4-5` | ✅ opus | ✅ 64K | ✅ May 2025 | ✅ opus-4-5 | ✅ yes | ✅ |
| `claude-haiku-4-5` | ✅ haiku | ✅ 64K | ✅ Feb 2025 | ✅ haiku | ✅ no | ✅ |
| `claude-sonnet-4-5` | ✅ — | ✅ 64K | ✅ Jan 2025 | ✅ sonnet-4-5 | ✅ yes | ✅ |
| **Company-prefixed** | | | | | | |
| `acme-claude-opus-4-5` | ✅ opus | ✅ 64K | ✅ May 2025 | ✅ opus-4-5 | ✅ yes | ✅ |
| `acme-claude-haiku-4-5` | ✅ haiku | ✅ 64K | ✅ Feb 2025 | ✅ haiku | ✅ no | ✅ |
| **Bedrock-style** | | | | | | |
| `us.anthropic.claude-opus-4-5-...-v1:0` | ✅ opus | ✅ 64K | ✅ May 2025 | ✅ opus-4-5 | ✅ yes | ✅ |
| **Vertex-style** | | | | | | |
| `claude-opus-4-5@20251101` | ✅ opus | ✅ 64K | ✅ May 2025 | ✅ opus-4-5 | ✅ yes | ✅ |
| **Dot notation** | | | | | | |
| `claude-opus-4.5` | ✅ opus | ❌ 32K | ❌ Jan 2025 | ❌ opus-4 | ✅ yes | **⚠️** |
| `claude-haiku-4.5` | ✅ haiku | ✅ 64K | ✅ Feb 2025 | ✅ haiku | ✅ no | ✅* |
| `claude-sonnet-4.5` | ✅ — | ✅ 64K | ⚠️ Jan 2025 | ⚠️ sonnet-4 | ✅ yes | **⚠️** |
| **Arbitrary aliases** | | | | | | |
| `claude45` | ❌ none | ❌ 32K | ❌ null | ❌ none | ✅ yes | **❌** |
| `company-smart` | ❌ none | ❌ 32K | ❌ null | ❌ none | ✅ yes | **❌** |
| `company-fast` | ❌ none | ❌ 32K | ❌ null | ❌ none | ✅ yes | **❌** |
| `fast-haiku-v2` | ✅ haiku | ❌ 32K | ❌ null | ❌ none | ✅ no | **⚠️** |
| `my-opus-variant` | ✅ opus | ❌ 32K | ❌ null | ❌ none | ✅ yes | **⚠️** |

\* `claude-haiku-4.5` works because haiku checks use broader patterns (`"haiku-4"`, `"claude-haiku-4"`) that don't include the minor version.

---

## Appendix A: How the CLI Detects Model Capabilities

The CLI does **not** parse version numbers. It uses `.includes()` substring checks on the resolved model ID string.

### Resolution pipeline

```
Input string (e.g., "opus", "claude-opus-4-5", "acme-claude-opus-4-5")
    │
    ▼
b2() — Core resolver
    ├─ Recognized shorthand? ("opus", "sonnet", "haiku", "opusplan", "sonnet[1m]")
    │   ├─ Yes → Resolve via dedicated function → full provider-specific model ID
    │   └─ No  → Pass through unchanged as literal string
    │
    ▼
Resolved model ID string
    │
    ├─ Sent to API as the model parameter
    └─ Used for substring-based capability detection
```

Non-shorthand strings pass through **completely unchanged**. The CLI sends whatever literal string you provide directly to the API.

### Complete substring check catalog

| # | Check | Substring pattern | What it controls |
|---|---|---|---|
| 1 | Beta gating | `.includes("haiku")` | Excludes from `claude-code-20250219` beta |
| 2 | Beta gating | `.includes("haiku")` | Excludes from custom `ANTHROPIC_BETAS` |
| 3 | Thinking gate | Excludes haiku | Enables/disables extended thinking blocks |
| 4 | Cache optimization | Detects haiku | Skips prompt cache optimization |
| 5 | Plan mode upgrade | Detects haiku | Silently upgrades haiku → Sonnet |
| 6 | Opus-class | `.includes("opus")` | Enables opus-in-plan-mode gating |
| 7 | Max tokens | `.includes("opus-4-5")` | → 64K output tokens |
| 8 | Max tokens | `.includes("opus-4")` | → 32K output tokens (fallback) |
| 9 | Max tokens | `.includes("sonnet-4")` or `.includes("haiku-4")` | → 64K output tokens |
| 10 | Max tokens | *(else)* | → 32K output tokens (default) |
| 11 | Cutoff | `.includes("claude-opus-4-5")` | → "May 2025" knowledge cutoff |
| 12 | Cutoff | `.includes("claude-haiku-4")` | → "February 2025" knowledge cutoff |
| 13 | Cutoff | `.includes("claude-opus-4")` / `"claude-sonnet-4-5"` / `"claude-sonnet-4"` | → "January 2025" |
| 14 | Cutoff | *(else)* | → `null` (no cutoff in prompt) |
| 15 | Family slug | `.includes("claude-opus-4-5")` | → `"claude-opus-4-5"` |
| 16 | Family slug | `.includes("claude-opus-4-1")` | → `"claude-opus-4-1"` |
| 17 | Family slug | `.includes("claude-opus-4")` | → `"claude-opus-4"` (fallback) |

### Check evaluation order

Several checks use **early returns in priority order**. More-specific patterns are tested first:

**Max output tokens:**
```
1. "opus-4-5"                    → 64K    ← most specific, checked first
2. "opus-4"                      → 32K    ← catches opus-4-0, opus-4-1
3. "sonnet-4" OR "haiku-4"      → 64K
4. (default)                     → 32K
```

**Knowledge cutoff:**
```
1. "claude-opus-4-5"            → "May 2025"       ← checked first
2. "claude-haiku-4"             → "February 2025"
3. "claude-opus-4" / "claude-sonnet-4-5" / "claude-sonnet-4" → "January 2025"
4. (else)                       → null
```

This ordering is why `"opus-4-5"` works but `"opus-4.5"` doesn't — the dot prevents matching the more-specific pattern, and the less-specific `"opus-4"` fallback catches it with different behavior.

### Behavioral impact

| Behavior | Trigger | When TRUE | When FALSE |
|---|---|---|---|
| **Fewer betas** | `includes("haiku")` | Excluded from advanced betas | Gets all betas |
| **Extended thinking** | Excludes haiku | Thinking blocks with budget | No thinking blocks |
| **Cache optimization** | Haiku detect | Skipped | Normal caching |
| **Plan mode upgrade** | Haiku in plan mode | Silent upgrade to Sonnet | Stays on selected model |
| **Opus plan gating** | `includes("opus")` | Opus-in-plan-mode logic | Normal execution |
| **Max output tokens** | Version-specific | 32K or 64K | 32K default |
| **Knowledge cutoff** | Family-specific | Correct date in system prompt | No cutoff mentioned |

---

## Appendix B: Dot Notation Deep Dive

This section explains exactly why dot notation (`claude-opus-4.5`) partially breaks and which models are affected.

### `claude-opus-4.5` — Misclassified as Opus 4.0

| Check | Substring | Present? | Expected | Actual |
|---|---|---|---|---|
| Opus-class | `"opus"` | ✅ | Yes | ✅ Correct |
| Max tokens (4.5) | `"opus-4-5"` | ❌ | Yes | Falls through... |
| Max tokens (4.0) | `"opus-4"` | ✅ | — | ❌ **32K instead of 64K** |
| Cutoff (4.5) | `"claude-opus-4-5"` | ❌ | Yes | Falls through... |
| Cutoff (4.0) | `"claude-opus-4"` | ✅ | — | ❌ **"Jan 2025" instead of "May 2025"** |
| Family slug | `"claude-opus-4-5"` | ❌ | Yes | ❌ **Classified as `"claude-opus-4"`** |

**Root cause:** `"claude-opus-4.5"` contains `"opus-4"` but not `"opus-4-5"`. The dash between major and minor version is critical.

### `claude-haiku-4.5` — Works (broader check patterns)

| Check | Substring | Present? | Actual |
|---|---|---|---|
| Haiku detection | `"haiku"` | ✅ | ✅ Correct |
| Max tokens | `"haiku-4"` | ✅ | ✅ 64K |
| Cutoff | `"claude-haiku-4"` | ✅ | ✅ "Feb 2025" |

**Why it works:** Haiku checks don't include the minor version in the substring. `"haiku-4"` matches both `"haiku-4-5"` and `"haiku-4.5"`.

### `claude-sonnet-4.5` — Accidentally correct (fragile)

| Check | Substring | Present? | Actual |
|---|---|---|---|
| Max tokens | `"sonnet-4"` | ✅ | ✅ 64K |
| Cutoff (4.5) | `"claude-sonnet-4-5"` | ❌ | Falls through... |
| Cutoff (4.0) | `"claude-sonnet-4"` | ✅ | ⚠️ "Jan 2025" (same result by coincidence) |

**Why it's fragile:** Sonnet 4.0 and 4.5 currently share the same cutoff. If they diverge in a future CLI version, this would silently break.

### Summary

| Dot notation pattern | Severity |
|---|---|
| `claude-opus-4.5` | **Broken** — wrong tokens, wrong cutoff |
| `claude-sonnet-4.5` | **Fragile** — correct by coincidence |
| `claude-haiku-4.5` | **Safe** — broader patterns cover it |

**Fix for all:** Replace dots with dashes → `claude-opus-4-5`, `claude-sonnet-4-5`, `claude-haiku-4-5`.

---

## Appendix C: The Opus 4.0 vs 4.5 Disambiguation

Both `"opus-4"` and `"opus-4-5"` contain the substring `"opus-4"`. The CLI handles this by checking `"opus-4-5"` first. But if your ID contains `"opus-4"` followed by anything other than `"-5"`, it falls through:

```
"claude-opus-4-5"    → "opus-4-5" matches first     → 64K  ✅
"claude-opus-4.5"    → "opus-4-5" fails, "opus-4"   → 32K  ❌
"claude-opus-4_5"    → "opus-4-5" fails, "opus-4"   → 32K  ❌
"claude-opus-4"      → "opus-4-5" fails, "opus-4"   → 32K  ✅ (correct for 4.0)
"claude-opus-4-1"    → "opus-4-5" fails, "opus-4"   → 32K  ✅ (correct for 4.1)
```

The only character that works between `4` and `5` is a **dash** (`-`).

---

*Analysis based on Claude Code CLI v2.1.29 bundle. Always verify with actual CLI behavior if critical.*
