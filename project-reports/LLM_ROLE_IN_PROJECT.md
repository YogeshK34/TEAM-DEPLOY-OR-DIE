# 🤖 Role of LLM in This Project

The short answer: **AST and LLM are two separate layers that work together in a pipeline.**
AST does the structural groundwork first, and the LLM enhances, enriches, and reasons on top of it.

---

## 🔀 The Full Pipeline

```
Upload .py file
      ↓
 [LAYER 1] AST (mvp_engine.py) — Structural analysis
      → Extracts functions, classes, imports, args, return logic
      → Detects logic bugs via code structure (e.g., is_even uses % == 1)
      → Generates a baseline: unit tests + edge tests + semantic suites
      ↓
 [LAYER 2] Gemini (gemini-2.5-flash-lite) — AI enrichment
      → Receives the AST baseline + raw source code as context
      → Re-generates semantic suites with deeper reasoning
      → Produces richer, more human-like test scenarios
      ↓
 [LAYER 3] Gemini AI Fixations — Bug fix suggestions
      → For every logic issue AST flagged, Gemini writes a fix suggestion
      → Returns: currentCode, suggestedCode, explanation, severity
      ↓
 Final response returned to frontend
```

---

## 📋 LLM's 3 Specific Jobs

| **Job** | **LLM Used** | **Where** |
|---|---|---|
| **1. Enhance test generation** — Takes AST baseline + source code, re-generates semantic suites with more natural test cases | `gemini-2.5-flash-lite` | `tryGenerateWithGemini()` in `app/api/generate-tests/route.ts` |
| **2. AI Fix Suggestions** — For each bug flagged by AST, Gemini writes a code fix with explanation and confidence score | `gemini-2.5-flash-lite` | `enrichFixSuggestions()` in `app/api/generate-tests/route.ts` |
| **3. User Story tests** — When user pastes a user story (not code), OpenAI/Gemini parses it and generates acceptance criteria tests | `gpt-4.1-mini` / `gemini-2.5-flash-lite` | `app/api/generate-userstory-tests/route.ts` |

---

## 🧩 The Key Insight — AST Feeds the LLM

The LLM is **never sent the raw code blind**. The prompt in `buildGeminiSourcePrompt()` explicitly includes:

```ts
// route.ts lines 124-125:
"Baseline AST analysis and local semantic context:"
JSON.stringify({ summary: baseline.summary, semanticSuites: baseline.semanticSuites })
```

Gemini is given:
- The **raw source code** (to understand implementation)
- The **AST-generated baseline** (already-detected structure, bug flags, semantic intent)

This makes the LLM's output much more accurate — it's not starting from scratch, it's **upgrading what AST already found**.

---

## 🛡️ Fallback Design

If Gemini is unavailable (no API key or failure), the system **gracefully falls back** to the pure AST output:

```ts
// route.ts line 319
if (!apiKey) return { ...baseline, provider: { provider: 'local', model: 'mvp_engine' } }
```

So **the app always works**, even without an LLM — it just uses the AST-only baseline.
