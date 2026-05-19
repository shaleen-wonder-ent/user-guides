# Stage 2b — Naive Bayes ML (Self-Learning Classifier)

> The cheap statistical layer that sits between hand-written rules (Stage 2)
> and the expensive Claude call (Stage 3) in the DocAI email-classification
> pipeline.
>
> **Runs in-process inside the existing `docai-worker` Container App.**
> No separate ML service, no GPU, no Foundry endpoint, no AML deployment.

---

## 1. Where it sits in the pipeline

```
Email Ingestion Worker ─► Service Bus (docai-jobs)
                                  │
                                  ▼
                  ┌─────────────────────────────────────────────────┐
                  │  DocAI Worker (Container App, KEDA-scaled)      │
                  │                                                 │
                  │   Stage 1   TRASH Regex            ~ 1 ms       │
                  │      │                                          │
                  │      ▼                                          │
                  │   Stage 2   Insurance Domain Rules ~ 10 ms      │
                  │      │                                          │
                  │      ▼                                          │
                  │   Stage 2b  Naive Bayes ML  ◄── this doc        │
                  │             (in-process, model.pkl in RAM)      │
                  │             ~ 20–50 ms                          │
                  │      │                                          │
                  │      ▼                                          │
                  │   if confidence ≥ 0.85 → label & exit           │
                  │   else                 → Stage 3 (Claude)       │
                  └─────────────────────────────────────────────────┘
```

Output of every stage is one of the 5 DocAI categories:

`ADHOC_REQUEST` · `ADHOC_RESPONSE` · `RENEWAL_RESPONSE` · `OTHERS` · `TRASH`

---

## 2. What Naive Bayes actually is

A **probabilistic text classifier** — one of the oldest, simplest, most
boring ML algorithms there is. Given a piece of text, it computes:

$$
P(\text{category} \mid \text{words}) \propto P(\text{category}) \cdot \prod_{i} P(\text{word}_i \mid \text{category})
$$

In plain English: *"Given that I see the words `policy`, `renewal`, `2026`,
`bind`, how likely is this to be a `RENEWAL_RESPONSE` vs the other 4
categories?"*

### Key properties

| Property | Value |
|---|---|
| Algorithm family | Linear, probabilistic |
| Training data | Past emails + operator-confirmed labels |
| Model artifact | A `model.pkl` file containing word-probability tables |
| Model size | ~ 5–20 MB |
| Inference cost | A few hundred dictionary lookups + multiplications |
| Inference hardware | CPU only — no GPU, no accelerator |
| Inference latency | ~ 20–50 ms per email |
| Memory at runtime | ~ 50–100 MB resident |
| Library | scikit-learn `MultinomialNB` (Python) or ML.NET equivalent |

> **It is not deep learning. It is not an LLM. It has no neural network,
> no embeddings, no GPU dependency.** It's basically counting words and
> multiplying probabilities.

---

## 3. Where it runs (the important answer)

### Inference: same Container App as everything else

It runs **inside the same `docai-worker` container** that already runs
Stage 1 regex and Stage 2 rules. Same pod, same process, same memory.

```
docai-worker  (Container App · Workload Profile: Consumption · scale-to-zero)
┌──────────────────────────────────────────────────────────┐
│ on_message(sb_msg):                                      │
│     text = load_email(sb_msg.blob_uri)                   │
│                                                          │
│     if stage1_regex(text):    return "TRASH"             │
│     if stage2_rules(text):    return rule_label          │
│                                                          │
│     label, conf = NB_MODEL.predict_proba(text)  ◄────────┤── here
│     if conf >= 0.85:          return label               │
│                                                          │
│     return stage3_claude(text)                           │
└──────────────────────────────────────────────────────────┘
        │
        └── NB_MODEL is loaded from Blob ONCE at container startup
            and held in RAM for the lifetime of the replica.
```

### Why NOT a separate service

| Option we did NOT pick | Why not |
|---|---|
| Separate Container App for NB | Adds a network hop (~5–20 ms) for a 20 ms call. Net loss. |
| Azure ML managed online endpoint | Built for GPU / heavy models. Overkill, slower cold start, more $$. |
| AI Foundry hosted deployment | Foundry is for LLMs / embeddings. NB is not an LLM. |
| AKS GPU pool | NB doesn't use a GPU. Period. |

**Conclusion:** in-process is strictly better. No new resource, no new
RBAC, no new VNet rule, no new failure mode.

### Training: small scheduled job (separate from inference)

Training is **not** in the request path. It's a nightly batch job:

```
Container Apps Job (cron: 0 2 * * *)  — runs ~5 min nightly on 2 vCPU
┌──────────────────────────────────────────────────────────┐
│ 1. Read labeled emails from SQL Ledger:                  │
│       SELECT body, subject, attachments, final_label     │
│       FROM classifications                               │
│       WHERE labeled_at > now - 90 days                   │
│       AND   source IN ('operator', 'auto_confirmed')     │
│                                                          │
│ 2. Train MultinomialNB on that dataset (sklearn)         │
│ 3. Validate on a held-out 10% split                      │
│ 4. If F1 ≥ previous model F1 - 0.02 (no regression):     │
│       version = today's date, e.g. 2026-05-19            │
│       upload model.pkl → Blob:                           │
│         models/naive-bayes/{version}/model.pkl           │
│       update pointer:                                    │
│         models/naive-bayes/current → {version}           │
│ 5. Emit metrics: training_accuracy, training_size, drift │
└──────────────────────────────────────────────────────────┘
```

A **Container Apps Job** is the right primitive for this — it's exactly
the same Container Apps Environment, but the workload type is `Job` instead
of `App`, so it runs to completion and shuts down. No always-on cost.

---

## 4. The "self-learning" loop

This is where the *self-learning* label comes from — it's a closed feedback
loop between operators and the model:

```
                  ┌──────────────────────────────────────────┐
                  │           DocAI Worker (inference)       │
                  │   NB predicts label + confidence         │
                  └──────────────┬───────────────────────────┘
                                 │
              ┌──────────────────┼──────────────────┐
              ▼                  ▼                  ▼
        confidence ≥ 95 %   75–95 %             < 75 %
              │                  │                  │
              ▼                  ▼                  ▼
        Done Queue       Review Queue        Stage 3 (Claude)
        (auto-applied)   (operator decides)  (high-quality re-label)
              │                  │                  │
              └──────────────────┴──────────────────┘
                                 │
                                 ▼
                       SQL Ledger (append-only)
                       {email_id, predicted, final_label, source, ts}
                                 │
                                 ▼
                  Nightly Container Apps Job retrains
                  the NB model on the latest 90 days
                  of *final* labels.
                                 │
                                 ▼
                       New model.pkl in Blob
                                 │
                                 ▼
                  Workers reload within 1 h
                  (next pull or next replica start).
```

The model **improves every night** because:

- Every operator correction adds a high-quality labeled example.
- Every Stage 3 (Claude) decision adds a high-quality labeled example.
- Drift (new sender patterns, new product names, new templates) gets
  absorbed naturally — no manual rule edits.

---

## 5. How workers pick up a new model

No restart, no redeploy required:

1. On startup, the worker reads `models/naive-bayes/current` from Blob
   (a tiny pointer file containing the active version, e.g. `2026-05-19`).
2. It downloads `models/naive-bayes/2026-05-19/model.pkl` into memory.
3. Every 1 hour, a background thread re-reads the `current` pointer.
   If the version changed, it atomically swaps the in-memory model.
4. If load fails, the worker keeps using the previous model and emits an
   alert — it **never serves uninitialized**.

Because models are **versioned and immutable**, rollback is just:

```bash
az storage blob upload --overwrite \
  --account-name <stg> --container models \
  --name naive-bayes/current --data "2026-05-12"
```

within ~1 hour every replica is back on the old model.

---

## 6. Why this approach (vs just always calling Claude)

| Metric | Claude on every email | With Stage 2b NB filter |
|---|---|---|
| % of emails that need Claude | 100 % | ~ 10–20 % |
| Avg latency per email | 1–3 s | 50 ms (cheap path) or 1–3 s (LLM path) |
| Cost per 1 M emails | $$$$ (LLM tokens) | $ (CPU only) + $$ (Claude on hard ones) |
| Cold-start friendly | Yes | Yes — model loads in < 1 s |
| Improves over time | Only with prompt changes | **Yes — automatically, nightly** |
| Explainability | LLM rationale (verbose) | Top-N contributing words (concise) |

The whole point of Stage 2b is: **don't pay for an LLM call when a
20-millisecond word-count math problem can answer it confidently.**

---

## 7. Identity, secrets, network

Inherits the worker's setup — nothing new is added:

- **Identity:** System-Assigned Managed Identity on the `docai-worker` Container App.
- **Permissions added for this stage:**
  - `Storage Blob Data Reader` on the `models` container (to download `model.pkl`).
  - (Training job only) `Storage Blob Data Contributor` on the `models` container.
  - (Training job only) Read access to SQL Ledger labels table.
- **Network:** Model artifact pulled via Blob **Private Endpoint** — never traverses the public internet.
- **Secrets:** None. There are no API keys, tokens, or endpoints involved — the model is a file.

---

## 8. Observability

| Signal | Source | Why it matters |
|---|---|---|
| `nb_inference_latency_ms` (p50/p95) | App Insights custom metric | Detect model bloat, GC pauses |
| `nb_confidence_histogram` | App Insights custom metric | Detect drift — if confidence collapses, dataset has shifted |
| `nb_stage3_escalation_rate` | App Insights custom metric | % of messages sent to Claude — should stay roughly stable |
| `nb_model_version_in_use` | App Insights custom property | Confirm all replicas converged to the new model |
| `nb_training_f1_score` | Training job → App Insights | Alert if a retrain *regresses* below threshold |
| `nb_training_dataset_size` | Training job → App Insights | Alert if labeled data volume drops sharply |

**Alerts:**

- Stage 3 escalation rate > 35 % for 30 min → model is losing confidence; investigate drift.
- Training F1 drops > 2 pp vs previous run → block model promotion automatically (built into the job).
- No retrain in > 48 h → training job failed; page on-call.

---

## 9. Confidence thresholds (tunable)

| Confidence band | Routing | Rationale |
|---|---|---|
| ≥ 95 % | Done Queue (auto-apply NB label) | Operator never sees it; goes straight to Action Agent |
| 75 – 95 % | Review Queue (operator confirms / corrects) | Most valuable training signal |
| < 75 % | Stage 3 (Claude) | NB can't decide → spend the LLM call |

These are config values, not code — adjust per category to trade off
**autonomy** (high threshold = more human in the loop) vs **cost**
(low threshold = more Claude calls).

---

## 10. TL;DR

- **What:** A tiny statistical classifier (Naive Bayes, scikit-learn `MultinomialNB`).
- **Where it runs:** **In-process inside the existing `docai-worker` Container App.** No separate service.
- **Why no GPU / Foundry / AML:** Naive Bayes is not an LLM. It's CPU math on word counts. Adding a network hop or a managed endpoint would make it *slower and more expensive*, not faster.
- **Training:** Nightly **Container Apps Job** reads operator-confirmed labels from SQL Ledger, retrains, and publishes a versioned `model.pkl` to Blob.
- **Self-learning:** Every operator correction and every Claude decision becomes tomorrow's training data. The model gets quietly better every night, with zero code changes.
- **Why it exists at all:** To answer 80–90 % of emails for ~$0 so we only spend Claude tokens on the genuinely hard ones.
