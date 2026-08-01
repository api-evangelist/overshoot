---
name: overshoot-live-video-qa
description: >-
  Ask vision-language models questions about a live video feed in real time using
  the Overshoot API — create a stream, keep its lease alive while publishing frames
  over LiveKit/WebRTC, query frames with an OpenAI-compatible chat completion, then
  end the stream.
generated: '2026-07-20'
method: generated
source: openapi/overshoot-openapi.yaml
api: Overshoot API
base_url: https://api.overshoot.ai/v1beta
auth: Bearer API key (Authorization: Bearer ovs_...)
operations:
- createStream
- keepaliveStream
- chatCompletions
- deleteStream
- listModels
---

# Real-time video Q&A with Overshoot

Query a live camera or video feed with a vision-language model in real time.

## Prerequisites
- An Overshoot API key (prefix `ovs_`) from https://platform.overshoot.ai/api-keys.
- Prepaid credit balance (inference returns `402` when billing denies a request).

## Steps

1. **Pick a model (optional).** `listModels` — `GET /models` (no auth). Use a model
   whose `status` is `ready` (e.g. `google/gemma-4-26B-A4B-it`, `claude-sonnet-4-6`).
   Use the returned `id` as the `model` field in step 3.

2. **Create a stream.** `createStream` — `POST /streams` with your bearer key.
   The `201` response returns `id`, `state`, a `publish` object (`url` + `token`),
   `expires_at_ms`, and `ttl_seconds`. Use `publish.url` and `publish.token` to
   publish media over WebRTC via LiveKit. The publish token is for LiveKit only —
   it does not replace your API key for HTTP calls.

3. **Keep the lease alive.** `keepaliveStream` — `POST /streams/{stream_id}/keepalive`
   **every 10-20 seconds.** Streams expire after ~45 seconds without a keepalive.
   Each keepalive also pays for elapsed streaming time.

4. **Query the feed.** `chatCompletions` — `POST /chat/completions` (OpenAI-compatible).
   Reference frames inside your message content by URL using the `ovs://` grammar:
   `ovs://streams/{stream_id}?frame_index=...` (or `timestamp_ms`/`offset_ms`, and
   `start_`/`end_` prefixes for segments). Frames are addressable only within the
   600-second retention window; older frames return `frame_evicted`.

5. **End the stream.** `deleteStream` — `DELETE /streams/{stream_id}`. Idempotent on
   already-deleted streams within the lookup window.

## Conventions & error handling
- **Rate limits:** on `429`, honor `x-ratelimit-limit`/`-remaining`/`-reset` and
  `retry-after` (currently 1s); back off exponentially.
- **Regions:** a `409` `region_error` carries `expected_region`/`requested_region` —
  retarget the correct region and retry.
- **Error envelope:** `/chat/completions` returns
  `{ "detail": { "error": { message, type, code } } }` — inspect `code`
  (`stream_not_found`, `frame_evicted`, `model_unavailable`, `billing_denied`, ...).
  See `errors/overshoot-problem-types.yml`.
- **Billing:** money is in integer microcents (1 cent = 1,000,000 microcents).
