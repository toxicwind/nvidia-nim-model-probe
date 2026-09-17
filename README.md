# nvidia-nim-model-probe

![models](https://img.shields.io/badge/models-82%20swept-blue)
![usable](https://img.shields.io/badge/usable%20now-10%2F82-orange)
![method](https://img.shields.io/badge/method-fail--fast%20ladder-green)
![license](https://img.shields.io/badge/license-MIT-lightgrey)

**We swept all 82 models in NVIDIA's NIM catalog with a fail-fast probe ladder. Only 10 answered. The catalog is an unreliable narrator: it lists embeddings that can never answer chat completions, 21 legacy models that are all gated, and it *mutates between calls* — deepseek-v4-pro was delisted mid-investigation. Availability itself is a per-minute lottery: models flap between alive, 503, and timeout across runs.**

## TL;DR

| Finding | Evidence |
|---|---|
| 10/82 models answer chat completions | `data/probe_results_all_2026-09-14.json` |
| 21/21 legacy models gated, 7/7 embeddings gated | category × verdict table below |
| Availability is nondeterministic | ultra-550b: dead → alive; super-120b: alive → 503 across runs |
| The catalog mutates | 82 → 81 models; `deepseek-v4-pro-0813` delisted within an hour |
| Default model is dying | `super-120b` serves a `deprecation: 2026-10-03T09:00:00Z` header |
| `/v1/models` ≠ "available models" | it's NVIDIA's whole registry: guards, reward models, a physics calibrator, a video detector |
| Guard models are the most callable category | 5/5 guards respond; only 5/32 chat models do |

## WTF the 82 models are

The community treats `/v1/models` as a menu of chat models. It's a registry dump. Crossed with probe verdicts (`alive` = answered our 3-token ping in <5s):

| Category | n | alive | gated/dead | notes |
|---|---|---|---|---|
| Current-gen chat | 32 | 5 | 19 gated, 5 timeout, 1 stall, 1 503, 1 400 | only ~1/6 of "chat" models actually chat |
| Nemotron-3 family | 5 | 1 | ultra-550b alive; super-120b **503**; omni **503**; lightning timeout | flagship tier is flapping |
| Legacy / old-gen | 21 | 0 | **all 21 gated** | llama2, codellama, fuyu, starcoder2, dbrx, phi-3, mixtral… the entire 2023–24 fleet |
| Embedding / retrieval | 7 | 0 | all gated | wrong modality — could *never* answer a chat completion |
| Vision / multimodal | 6 | 1 | 4 gated, 1 timeout | llama-3.2-11b-vision answers |
| Guard / safety | 5 | 2 | all 5 respond somehow | most callable category in the catalog |
| Reward model | 1 | 0 | gated | a reward model, listed as a chat model |
| Oddballs | 5 | 1 | diffusion model answers; Ising calibrator times out; video detector 500s | an Ising physics calibrator and a synthetic-video detector are in the chat catalog |

Of the 10 alive: 3 are guard/safety classifiers, 2 are Riva *translation* models, 1 is vision, 1 is a diffusion model, 1 is a document parser. **~4 are general-purpose chat models.**

## Availability is a per-minute lottery

Same account, same key, same probe — different answers across runs minutes apart:

| Model | run 1 (19-model) | run 2 (82-sweep) |
|---|---|---|
| `nemotron-3-ultra-550b` | flaky-timeout | **alive** 1396ms |
| `nemotron-3-super-120b` | alive 554ms | **503** |
| `deepseek-v4-pro-0813` | alive 2.7s | timeout, then **delisted from catalog** |
| `nemotron-3-nano-omni` | 503 | 503 |
| `glm-5.3-flash` | alive 583ms | alive 3.5s |
| `deepseek-v4-flash` | streaming stall | streaming stall (stable) |

Verdict distribution across all 82: **55 gated-404, 10 alive, 9 timeout, 3 503, 3 error, 2 streaming-stall.** Ranking models by availability from a single sweep is measuring the weather. The only stable properties we found: gated-404s are fast and stable (~300–600ms), streaming-stall is stable, everything else flaps.

## The catalog mutates

Between the 82-model sweep and the per-model GET oracle (~10 min later): **82 → 81 models.** `deepseek-ai/deepseek-v4-pro-0813` vanished from `/v1/models`. EOL models don't 410 anymore — they get delisted (`nemotron-3-nano-30b-a3b`, EOL 2026-09-01, is gone from the catalog). Meanwhile `GET /v1/models/{id}` returns 200 for everything listed — it's an existence oracle, not an entitlement oracle. Gated models 404 only at inference time.

## Deprecation cliff

`nvidia/nemotron-3-super-120b-a12b` — the best default on this API — returns header `deprecation: 2026-10-03T09:00:00Z`. Three weeks out. And it 503'd in our latest sweep.

## API surface findings (fuzzed, 25 probes)

- `GET /v1/models` → 200, 82 ids (mutable). `GET /v1/models/{id}` → 200 listed / 404 fabricated.
- `POST /v1/chat/completions` is the only real inference route. `POST /v1/completions` → 404, images/audio/moderation/batches/files → 404.
- `POST /v1/embeddings` routes model-aware: `nvidia/nv-embed-v1` → 410 with EOL date 2026-08-25. The API is *not* chat-only — but all 7 listed embedding models 404 at inference for this account.
- Chat shape matrix: SSE+stream_options, response_format, tools, logprobs accepted; `n=2`+temp 0 → 400; empty model → 400. `nemotron-parse` (v1) 400s on plain strings — needs structured input.
- Full endpoint matrix: `data/fuzz_endpoints_2026-09-14.json`.

## Method

- **Fail-fast ladder**: 5s non-streaming ping → streaming probe → (retries disabled in the full sweep). Anything slower than 5s is useless for interactive agents.
- **No truncation in diagnostics**: full bodies, headers, timing stages per model (local `probe_audit.jsonl`).
- **Catalog order, not star order**: `--all` probes the live `/v1/models` listing as returned. No cherry-picking.
- **Rate-limit audit**: 429s are recorded, never slept through.
- Independent cross-validation: [aviclaw01/nvclaude](https://github.com/aviclaw01/nvclaude) found the same 404-for-account shape; [sherman-yang/nvidia-model-info](https://github.com/sherman-yang/nvidia-model-info) built a richer paced taxonomy. Forum root-cause threads: [1](https://forums.developer.nvidia.com/t/public-api-endpoints-scope-missing-on-personal-org-llama-gemma-work-kimi-deepseek-qwen-nemotron-all-404/378043) [2](https://forums.developer.nvidia.com/t/newer-nim-models-kimi-k2-6-deepseek-v4-pro-hang-indefinitely-or-404-possible-missing-public-api-endpoints-permission/377777) [3](https://forums.developer.nvidia.com/t/function-not-found-for-account-moonshotai-kimi-k2-6-and-deepseek-ai-deepseek-v4-pro-0813-404/382736)

## Repo contents

- `probe.py` — the fail-fast ladder (`--all`, `--workers`, `--timeout-a`, per-model NVCF header capture, deprecation/EOL parsing)
- `audit_endpoints.py` — deep per-model diagnostics, full bodies + headers, no truncation
- `fuzz_endpoints.py` — 25-probe API surface matrix
- `bench.py` — streaming tool-call benchmark (paper-ranking task)
- `nim.py`, `rank.py`, `auth.py` — public client shims (`NVIDIA_API_KEY` env)
- `data/probe_results_all_2026-09-14.json` — all 82 models, verdicts + timing + headers
- `data/model_oracle_2026-09-14.json` — per-model GET oracle (81 models, all 200)
- `data/fuzz_endpoints_2026-09-14.json` — endpoint surface matrix
- `data/audit_endpoints_2026-09-14.json` — deep per-model audit with full bodies
- `data/probe_results_2026-09-14.json` — earlier 19-model sweep

Account identifiers are redacted in all public data (`REDACTED`).

## Limitations

- Single account/key (free build.nvidia.com tier), single region, 2026-09-14. Entitlements are per-account — your 10 will differ.
- "Usable" = answered a 3-token ping. Content quality, tool-call reliability, and long-context behavior need `bench.py` runs (in progress).
- Fast 404s are stable; alive/timeout/503 verdicts are snapshots, not properties. Re-run before trusting.

## License

MIT

## Consolidated

This repo has been merged into [toxicwind/nvidia-nim](https://github.com/toxicwind/nvidia-nim) (2026-09-17) with full history preserved. New NIM work goes there.
