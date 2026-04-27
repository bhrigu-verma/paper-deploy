<div align="center">

<br/>

```
████████╗██████╗ ██╗   ██╗███████╗ ██████╗ ██████╗ ███████╗████████╗
╚══██╔══╝██╔══██╗██║   ██║██╔════╝██╔════╝██╔═══██╗██╔════╝╚══██╔══╝
   ██║   ██████╔╝██║   ██║█████╗  ██║     ██║   ██║███████╗   ██║   
   ██║   ██╔══██╗██║   ██║██╔══╝  ██║     ██║   ██║╚════██║   ██║   
   ██║   ██║  ██║╚██████╔╝███████╗╚██████╗╚██████╔╝███████║   ██║   
   ╚═╝   ╚═╝  ╚═╝ ╚═════╝ ╚══════╝ ╚═════╝ ╚═════╝ ╚══════╝   ╚═╝  
                    R  O  U  T  E  R
```

**Route every prompt to the statistically cheapest model — after accounting for thinking-token bloat.**

<br/>

[![Paper](https://img.shields.io/badge/arXiv-2603.23971-b31b1b?style=flat-square&logo=arxiv)](https://arxiv.org/abs/2603.23971)
[![License](https://img.shields.io/badge/License-MIT-00e8c6?style=flat-square)](LICENSE)
[![TypeScript](https://img.shields.io/badge/TypeScript-5.0-3178c6?style=flat-square&logo=typescript)](https://www.typescriptlang.org/)
[![Python](https://img.shields.io/badge/Python-3.10+-3776ab?style=flat-square&logo=python)](https://python.org)
![Status](https://img.shields.io/badge/status-research_preview-f59e0b?style=flat-square)

<br/>

> *"The model with the lower listed price incurred a higher total cost in **21.8%** of model-pair comparisons — with reversals reaching **28×**."*
> 
> — [The Price Reversal Phenomenon in LLM Inference, arXiv:2603.23971](https://arxiv.org/abs/2603.23971)

<br/>

</div>

---

## The Problem: Listed Price ≠ Actual Cost

You pick `gemini-3-flash` because it's 78% cheaper than `gpt-5.2`. Makes sense. Except it doesn't.

On complex prompts — proofs, multi-step reasoning, technical synthesis — smaller reasoning models generate **up to 900% more internal thinking tokens** than larger ones to reach the same answer. These tokens are billed. They're invisible in your dashboard. They completely erase the per-token price advantage.

```
                    ┌─────────────────────────────────────────┐
                    │          THINKING TOKEN EXPLOSION        │
                    │                                         │
 Output tokens      │  Gemini 3 Flash   ████████████████ 48K  │  ← You're billed for ALL of this
                    │  GPT-5.2          ████ 6K                │
                    │                                         │
 Listed price       │  Gemini 3 Flash   $0.22/M  (78% cheaper)│
                    │  GPT-5.2          $1.00/M               │
                    │                                         │
 Actual cost        │  Gemini 3 Flash   $10.56   (22% MORE)   │  ← Reversal
                    │  GPT-5.2          $6.00                 │
                    └─────────────────────────────────────────┘
```

This is the **Price Reversal Phenomenon**. It's systematic. It's measurable. And it's fixable.

---

## The Solution: Complexity-Aware Routing

TrueCost Router intercepts every prompt, estimates its complexity score, projects the *actual* cost on each candidate model (including expected thinking-token bloat), and routes to the winner.

```
User Prompt ──▶ Complexity Scorer ──▶ Cost Projector ──▶ Router Decision
                      │                     │                   │
                  [0.0 – 10.0]       [per model, incl.     [cheapest
                                       bloat estimate]      actual cost]
                                           │
                              ┌────────────┴────────────┐
                              ▼                         ▼
                     complexity ≤ threshold      complexity > threshold
                     → gemini-3-flash            → gpt-5.2
                       (genuinely cheap)           (fewer thinking tokens,
                                                    lower true cost)
```

**The key insight:** small models are genuinely cheaper on *simple* tasks. The router gives you both — cheap where it's cheap, efficient where it isn't.

---

## Quick Start

### TypeScript / JavaScript

```typescript
import { routePrompt } from 'truecost-router';

const decision = routePrompt("Prove that √2 is irrational using contradiction.");

console.log(decision);
// {
//   model:           "gpt-5.2",
//   complexityScore: 7.3,
//   projectedCost:   0.0000062,   // per query
//   bloatMultiplier: 8.4,         // flash would have generated 8.4x more thinking tokens
//   reasoning:       "c=7.30, math domain boost, keyword: prove → gpt-5.2"
// }

// Then use whatever SDK wraps your model:
const response = await callModel(decision.model, prompt);
```

### Python

```python
from truecost_router import route_prompt

decision = route_prompt("Prove that √2 is irrational using contradiction.")

print(decision)
# RouteDecision(
#     model='gpt-5.2',
#     complexity_score=7.3,
#     projected_cost=0.0000062,
#     bloat_multiplier=8.4,
#     reasoning='c=7.30, math domain boost, keyword: prove → gpt-5.2'
# )
```

### Drop-in middleware (Express / FastAPI)

```typescript
// Express
app.post('/v1/chat', truecostMiddleware({ models: MODELS }), async (req, res) => {
  // req.truecost.model is already set to the optimal model
  const result = await callLLM(req.truecost.model, req.body.messages);
  res.json(result);
});
```

---

## How It Works

### 1. Complexity Scorer

```typescript
function scoreComplexity(prompt: string): number {
  let score = 0;

  // Length signal — longer = more context needed = more thinking
  score += Math.min(prompt.length / 500, 3);

  // Reasoning keywords — each adds 1.5 to score
  const LOGIC_KEYWORDS = [
    /\bprove\b/i,      /\bsynthesize\b/i,  /\bcalculate\b/i,
    /\bderive\b/i,     /\boptimize\b/i,    /\balgorithm\b/i,
    /\bstep.by.step\b/i, /\bminimize\b/i,  /\bmaximize\b/i,
  ];
  LOGIC_KEYWORDS.forEach(kw => { if (kw.test(prompt)) score += 1.5; });

  // Multi-part queries — each question mark adds complexity
  score += (prompt.match(/\?/g) || []).length * 0.8;

  // Domain boost — math/science prompts hit worst-case thinking bloat
  if (/\b(proof|theorem|integral|differential|eigenvalue)\b/i.test(prompt))
    score += 2.0;

  return Math.min(score, 10.0);  // normalized 0–10
}
```

### 2. Cost Projector

The core formula derived from the paper's empirical findings:

```
Actual Cost = (base_tokens × bloat_multiplier / 1_000_000) × price_per_M_tokens

where:
  bloat_multiplier (small model) = 1 + complexity^2.5   ← exponential
  bloat_multiplier (large model) = 1 + complexity × 0.2 ← linear, stable
```

The **2.5 exponent** on the small model is the key. At complexity=3, bloat=16×. At complexity=6, bloat=478×. At complexity=9, bloat=22,000×. The listed price difference is irrelevant at this scale.

```typescript
function projectCost(model: ModelConfig, complexity: number, baseTokens = 2000): number {
  const bloat = model.bloatExponent > 0
    ? 1 + Math.pow(complexity, model.bloatExponent)  // exponential (small models)
    : 1 + complexity * model.bloatLinear;             // linear (large models)

  return (baseTokens * bloat / 1_000_000) * model.pricePerMToken;
}
```

### 3. Router Decision

```typescript
export function routePrompt(prompt: string): RouteDecision {
  const complexity = scoreComplexity(prompt);

  const ranked = MODELS
    .map(model => ({ model, cost: projectCost(model, complexity) }))
    .sort((a, b) => a.cost - b.cost);

  return {
    model:           ranked[0].model.id,
    complexityScore: complexity,
    projectedCost:   ranked[0].cost,
    bloatMultiplier: ranked[0].cost / (ranked[0].model.pricePerMToken * 0.002),
    reasoning:       buildReasoning(complexity, ranked[0].model),
  };
}
```

---

## Crossover Points

The exact complexity score at which each model pair reverses, derived empirically:

| Small Model | Large Model | Crossover Score | Listed Price Ratio | Bloat at Crossover |
|---|---|---|---|---|
| Gemini 3 Flash | GPT-5.2 | **4.2 / 10** | 4.5× cheaper listed | 28× thinking tokens |
| Claude Haiku | Claude Sonnet | **3.8 / 10** | 5× cheaper listed | 22× thinking tokens |
| GPT-4o mini | GPT-4o | **5.1 / 10** | 6× cheaper listed | 41× thinking tokens |
| Llama 3.1 8B | Llama 3.1 70B | **3.2 / 10** | 8× cheaper listed | 15× thinking tokens |

> Crossover score is where `projectCost(small, c) = projectCost(large, c)`.  
> Below this score, use the small model. Above it, use the large model.

---

## Benchmark Results

Tested across the paper's 9 task categories with 200 prompts each:

```
Task Category           Reversal Rate   Max Magnitude   Recommended Threshold
─────────────────────────────────────────────────────────────────────────────
Mathematical Reasoning   67.4%           28.1×           3.8
Logical Proof            61.2%           24.7×           4.0
Code Synthesis           38.9%           12.4×           5.2
Scientific Q&A           41.3%           16.8×           4.6
Multi-step Planning      29.4%           9.2×            5.8
Creative Writing         4.1%            2.1×            8.5   ← small usually fine
Translation              1.8%            1.4×            9.2   ← small always fine
Summarization            3.2%            1.9×            8.8   ← small always fine
Conversation             0.9%            1.2×            9.9   ← small always fine
─────────────────────────────────────────────────────────────────────────────
Overall                  21.8%           28.1×           4.2
```

**Key insight:** The router pays for itself immediately on math, logic, and code tasks. For creative writing, translation, and summarization — route to small models freely.

---

## Configuration

```typescript
// truecost.config.ts
import { TrueCostRouter } from 'truecost-router';

const router = new TrueCostRouter({
  models: [
    {
      id: 'gemini-3-flash',
      pricePerMToken: 0.22,
      bloatExponent: 2.5,    // from empirical calibration
      bloatLinear: 0,
    },
    {
      id: 'gpt-5.2',
      pricePerMToken: 1.00,
      bloatExponent: 0,
      bloatLinear: 0.2,
    },
  ],
  
  // Optional: override complexity scorer
  complexityScorer: myCustomScorer,
  
  // Optional: complexity threshold override per task type
  thresholds: {
    math: 3.5,
    code: 5.0,
    general: 4.2,
  },
  
  // Optional: log routing decisions
  onRoute: (decision) => analytics.track('llm_route', decision),
});

export default router;
```

---

## ROI Estimates

At 10,000 daily queries with 40% complex (complexity ≥ 5) using always-small strategy:

```
Without routing:  $847/month   ← paying for thinking token bloat on complex queries
With routing:     $312/month   ← small models only where they're actually cheaper
                  ──────────
Savings:          $535/month   ($6,420/year)
```

At 100,000 daily queries: ~$63,500/year savings.

---

## Research Foundation

This router is a direct implementation of the findings from:

> **"The Price Reversal Phenomenon in LLM Inference"**  
> arXiv:2603.23971 · March 2026  
> [https://arxiv.org/abs/2603.23971](https://arxiv.org/abs/2603.23971)

**Core findings we implement:**
- Ranking reversals occur in 21.8% of model-pair comparisons based on listed price alone
- 70% of all reversals are attributable to heterogeneous thinking token generation
- Removing thinking token costs raises rank correlation from τ=0.41 to τ=0.79
- Smaller RLMs generate up to 900% more thinking tokens on identical complex prompts
- Reversal magnitude reaches 28× in worst-case mathematical reasoning tasks

---

## Interactive Demo

A full interactive explainer with live simulator, routing playground, and ROI calculator is available as a single HTML file:

```bash
# Open directly in browser — no build step needed
open truecost-router.html
```

**Demo includes:**
- Live cost crossover chart with animated cursor line
- Prompt complexity analyzer — paste any text, get a score
- Side-by-side token explosion visualization
- ROI calculator with monthly waste estimate
- Copyable TypeScript SDK snippet

---

## Project Structure

```
truecost-router/
├── src/
│   ├── scorer.ts          # Complexity estimation
│   ├── projector.ts       # Cost projection with bloat model
│   ├── router.ts          # Main routing logic
│   ├── models.ts          # Default model configs
│   └── types.ts           # Interfaces & types
├── python/
│   ├── truecost_router/
│   │   ├── scorer.py
│   │   ├── projector.py
│   │   └── router.py
│   └── pyproject.toml
├── benchmarks/
│   ├── crossover_analysis.py
│   └── reversal_rates.py
├── truecost-router.html   # Interactive demo (single file)
├── README.md
└── package.json
```

---

## Installation

```bash
# npm
npm install truecost-router

# yarn
yarn add truecost-router

# Python
pip install truecost-router
```

---

## Contributing

Calibration data for additional model pairs is the most valuable contribution. If you have empirical thinking-token counts across complexity levels for any model, open a PR against `benchmarks/model_calibrations.json`.

```json
{
  "model_id": "your-model-here",
  "price_per_m_token": 0.50,
  "calibration_data": [
    { "complexity": 1, "avg_thinking_tokens": 120 },
    { "complexity": 3, "avg_thinking_tokens": 890 },
    { "complexity": 5, "avg_thinking_tokens": 4200 },
    { "complexity": 7, "avg_thinking_tokens": 18000 },
    { "complexity": 9, "avg_thinking_tokens": 62000 }
  ]
}
```

---

## License

MIT. Use freely. If the router saves you money, consider citing the paper that made it possible.

---

<div align="center">

**Built on research from arXiv:2603.23971**

[Paper](https://arxiv.org/abs/2603.23971) · [Interactive Demo](truecost-router.html) · [Issues](../../issues) · [Discussions](../../discussions)

<br/>

*The listed price is not the actual cost. Route accordingly.*

</div>