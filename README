# Precedent

An AI agent that reviews items in a chosen domain (default: code review of
pull requests) and **improves over time by learning from its own corrected
mistakes** — no model fine-tuning involved. Corrections are stored as
embedded vectors and retrieved alongside static domain guidelines on every
future review; a correction that's a strong semantic match to a new input
overrides the agent's default judgment.

## Live demo

Use the public app here: **https://precedent-review.precedent-review.workers.dev**

## Why this exists

Most "AI reviews your PR" demos are static — the same input always gets
the same output. Precedent's core claim is narrower and more falsifiable:
**correct it once, and it stops making that specific mistake on similar
future inputs, without retraining anything.** The correction-rate-per-batch
metric (see below) is how that claim gets measured rather than asserted.

## Architecture

```
                    ┌─────────────────────┐
   PR description → │   POST /review      │
                    └──────────┬───────────┘
                               │
                 ┌─────────────┴─────────────┐
                 │                            │
         retrieve_guidelines()        retrieve_corrections()
                 │                            │
        ChromaDB "guidelines"         ChromaDB "corrections"
         (static domain rules)      (past human corrections,
                                      embedded on the ORIGINAL
                                      input text for matching)
                 │                            │
                 └─────────────┬──────────────┘
                               │
                     generate_review()
              (Groq llama-3.3-70b via OpenAI SDK,
               or MOCK_MODE canned-but-grounded response)
                               │
                    verdict + reasoning + citations
                               │
                    stored in SQLite `reviews` table
```

```
   Human disagrees → POST /correct
                          │
              embed(original input_text)
                          │
         stored in ChromaDB "corrections" collection
         (metadata: wrong_output, correct_output, reason)
                          │
        SQLite row updated: corrected_flag=True,
                    embedding_ref=<chroma doc id>
```

### Why the correction is embedded on the *input text*, not the full narrative

Early in development this embedded `INPUT + WRONG_OUTPUT + CORRECT_OUTPUT +
REASON` as one document. That's wrong: future queries only have a bare new
input, so matching a bare query against a diluted, multi-field document
score systematically lower than input-to-input matching would. Fixed by
embedding just the input text for retrieval purposes, with the rest kept as
Chroma metadata and pulled in for the LLM prompt separately.

### Similarity threshold (`CORRECTION_SIMILARITY_THRESHOLD`)

Both collections use cosine distance explicitly (`hnsw:space: cosine` in
Chroma's collection metadata) — 0 = identical direction, 1 = orthogonal,
2 = opposite. A correction is treated as a **strong match** (told to the
LLM as authoritative, overriding default guideline judgment) when its
cosine distance to the new input is below this threshold; otherwise it's
surfaced as "loosely related, consider but don't treat as binding."

The shipped default (0.35) is a reasoned starting point, not a tuned one —
`scripts/calibrate_threshold.py` embeds a few known-similar and
known-unrelated PR pairs and prints their actual distances so you can pick
a defensible number once real embeddings are running (this couldn't be
tuned in the build sandbox — see Known Limitations).

### Evaluation metric: correction rate per batch

`GET /stats` groups reviews into batches of 10, in submission order, and
reports `corrected / total` per batch. This is the number that answers
"is it actually learning" without hand-waving — see `DEMO_SCRIPT.md` for
how to produce and narrate a live declining trend.

## Tech stack

| Layer | Choice | Why |
|---|---|---|
| Backend | FastAPI | async-friendly, auto-generated OpenAPI docs |
| Frontend | React + Tailwind (Vite) | fast dev loop, no heavyweight framework needed for one dashboard |
| Vector store | ChromaDB (local, persistent) | no external service — works fully offline |
| LLM | Groq (`llama-3.3-70b-versatile`) via OpenAI SDK | genuinely free tier, OpenAI-compatible so swapping providers later is a one-line base_url change |
| Embeddings | `sentence-transformers/all-MiniLM-L6-v2` | runs locally, no paid embedding API |
| DB | SQLite via SQLAlchemy | zero-setup, sufficient for a single-writer demo app |
| Deployment | Cloudflare Worker | stable public URL without hosting-provider branding |

## Project structure

```
precedent/
├── backend/
│   └── app/
│       ├── main.py           FastAPI app, CORS, startup hook, static frontend mount
│       ├── config.py         env-driven settings (MOCK_MODE, thresholds, etc.)
│       ├── database.py       SQLAlchemy engine/session
│       ├── models/review.py  the `reviews` table
│       ├── schemas.py        Pydantic request/response models + validation
│       ├── vectorstore.py    ChromaDB client, two collections
│       ├── retrieval.py      guideline + correction retrieval, seeding
│       ├── llm.py            Groq call + MOCK_MODE fallback, prompt construction
│       ├── seed_data.py      15 placeholder code-review guidelines
│       ├── demo_data.py      30 sample PRs, deliberately ordered (see below)
│       └── routes/reviews.py /review, /correct, /history, /stats
├── frontend/
│   └── src/                  React dashboard (see Design System below)
├── scripts/
│   ├── build.sh               installs deps, builds frontend
│   ├── run.sh                 starts the single-origin server
│   ├── deploy_tunnel.sh        starts the Cloudflare Tunnel
│   ├── run_demo.py            feeds the 30 sample PRs through a live server
│   └── calibrate_threshold.py  helper for tuning the similarity threshold
├── DEPLOYMENT.md              exact per-OS cloudflared install + tunnel steps
├── DEMO_SCRIPT.md             live walkthrough script for your viva
└── README.md                  this file
```

## Design system (frontend)

Deliberately not the default shadcn/light-template look. Dark developer-
tool aesthetic (near-black background, split dashboard layout — review
feed + correction memory panel, not a centered chat box). Font pairing:
**Sora** (UI/headings) + **IBM Plex Mono** (code and input content).
Color system doubles as a visual language: **teal** = what the agent
decided, **amber** = what a human corrected — so at a glance you can see
how much of the dashboard is agent-original vs. human-corrected judgment.

## Running it

```bash
./scripts/build.sh
./scripts/run.sh
# -> http://localhost:8000
```

`backend/.env` controls `MOCK_MODE` (default: true — no Groq key needed).
The current public demo is available at https://precedent-review.precedent-review.workers.dev.
See `DEPLOYMENT.md` for the local backend deployment workflow.

## Known limitations (stated plainly, not hidden)

- **Similarity threshold is a reasoned default, not an empirically tuned
  one.** The build environment couldn't reach huggingface.co to run the
  real embedding model, so `CORRECTION_SIMILARITY_THRESHOLD=0.35` needs
  validation with `scripts/calibrate_threshold.py` once you're running
  with real embeddings.
- **The public demo runs through a Cloudflare Worker proxy** — the original
  Python/Chroma backend remains available for local full-stack deployment.
- **Groq's free tier is rate-limited to 30 req/min.** `MOCK_MODE` exists
  specifically so a live demo never depends on that limit holding.
- **SQLite is single-writer.** Fine for a demo/single-user tool; would
  need a real database for concurrent users.
