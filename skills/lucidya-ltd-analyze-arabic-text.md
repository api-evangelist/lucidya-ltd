---
name: lucidya-analyze-arabic-text
description: >-
  Run Lucidya's AI models over a batch of texts to get sentiment, Arabic dialect
  and sub-dialect, domain category, and theme/sub-theme classification. Purpose-built
  for Arabic and MENA-market text, and callable as four independent batch endpoints.
api: Lucidya AI API
base_url: https://api.lucidya.com
spec: ../openapi/lucidya-ltd-ai-api-openapi.yml
generated: '2026-08-13'
method: generated
source: openapi/lucidya-ltd-ai-api-openapi.yml + https://docs.lucidya.com/docs/ai-api/12m2bpnec7ycv-rate-limiting
operations:
  - predict_sentiment_batch_ai_sentiment_predict_batch_post
  - predict_dialects_batch_ai_dialects_predict_batch_post
  - predict_domains_batch_ai_domains_predict_batch_post
  - predict_themes_batch_ai_themes_predict_batch_post
---

# Analyze text with the Lucidya AI API

Four stateless batch classifiers over the same request shape. Nothing is stored:
no id is returned and no result is retrievable afterwards, so hold the response.

The AI API requires an **AI-type** key, and the account must additionally hold the
Lucidya CXM Core product before a key can be issued.

## Request shape

All four operations are `POST`, take the required header `luc-authorization`, and
share one body — the `Texts` schema:

```json
{ "texts": ["منتج عظيم", "ما صارت!! الى متى التأخير"] }
```

| Operation | Path | Returns |
|---|---|---|
| `predict_sentiment_batch_ai_sentiment_predict_batch_post` | `POST /sentiment/predict_batch` | `SentimentPredictions` → `sentiments` |
| `predict_dialects_batch_ai_dialects_predict_batch_post` | `POST /dialects/predict_batch` | `DialectsPredictions` → `dialects`, `sub_dialects` |
| `predict_domains_batch_ai_domains_predict_batch_post` | `POST /domains/predict_batch` | `DomainsPredictions` → `domains` |
| `predict_themes_batch_ai_themes_predict_batch_post` | `POST /themes/predict_batch` | `ThemesPredictions` → `themes`, `sub_themes` |

Results come back positionally aligned with the input `texts` array. There is no
per-item id, so preserve input order — do not reorder or deduplicate the batch and
expect to match results back afterwards.

## The two limits that will bite you

Read these before writing a loop. They are much tighter than the rest of Lucidya:

1. **6 requests per minute.** Not 100. The AI API is the only Lucidya product with a
   6/min floor; the other four start at 100/min. An agent that paces for 100 will be
   throttled roughly 94% of the time.
2. **100 texts per request** (batch size), adjustable only at package-creation time.

So the practical ceiling is 600 texts/minute. Chunk the corpus into batches of 100,
issue at most 6 per minute, and pace deliberately — there is no `RateLimit-*` or
`Retry-After` header to read, so you cannot discover your actual ceiling at runtime.

To classify one corpus on all four dimensions you need four calls per batch, which
consumes 4 of the 6 requests in that minute.

## Errors

- `400` — bad request.
- `422` — validation failure. The AI API returns FastAPI-style detail here, a
  **different** envelope from the rest of Lucidya:
  `{"detail":[{"loc":["string"],"msg":"string","type":"string"}]}`.
  Handle both this shape and `{"error":{"status","detail"}}`.
- `500` / `503` / `504` — retry with backoff.

Full registry: `../errors/lucidya-ltd-problem-types.yml`.
Limits: `../rate-limits/lucidya-ltd-rate-limits.yml`.
