---
name: router
title: "Phantom AI: prepaid LLM inference and model router"
description: "OpenAI- and Anthropic-compatible large language model (LLM) chat over hundreds of models, plus embeddings, images, speech, transcription, rerank and video. phantom-1 picks a model per request. Paid in Solana USDC over x402 or MPP."
use_case: "Use for LLM chat, Anthropic Messages or OpenAI Responses calls, embeddings, image generation, text to speech, transcription, reranking or video, paid from prepaid USDC credit with no account, or when an agent wants a model picked for it."
category: ai_ml
service_url: https://phantom.codes
version: v1
openapi:
  path: openapi.json
---

Phantom AI serves large language model (LLM) chat from many providers in
three request formats: OpenAI Chat Completions (`POST
/v1/chat/completions`), Anthropic Messages (`POST /v1/messages`) and OpenAI
Responses (`POST /v1/responses`). All three stream. The same key pays for
embeddings (`/v1/embeddings`), images (`/v1/images/generations`), speech
(`/v1/audio/speech`), transcription (`/v1/audio/transcriptions`), reranking
(`/v1/rerank`) and video (`/v1/video/generations`). A video job takes its
cost up front; poll `/v1/video/generations/{id}` until it finishes, and a
failed job gets its money back. `GET /v1/models` is free and lists each
model id with its price.

Send `model: "phantom-1"` and the router picks a model for each request from
three tiers: Economy, Standard and Frontier. `x-phantom-routed` names the
model it picked, and `x-phantom-route-receipt` adds the tier and the reason.
`phantom-1:free` runs free models only and costs nothing, within daily
limits. To pick a model yourself, copy an id from `/v1/models`. Ids change
when providers ship new models, so read the list first.

## Buy credit and pay for calls

You pay once for credit, then spend it with an API key.

1. Buy credit:

   ```sh
   pay curl -X POST https://phantom.codes/v1/x402/top-up \
     -H 'content-type: application/json' -d '{"amount_usd": 1}'
   ```

   Phantom answers with a 402 for that amount in Solana USDC, over x402 or
   the Machine Payments Protocol (MPP). `pay` pays it and resends the
   request. The paid response contains
   `api_key`, `recovery_code`, `credit_usd` (what this payment added),
   `workspace_id` and `recovery`. You can buy $0.10 to $10,000. An empty
   body buys $1.

   The key comes in its own workspace, which holds the credit. Save the
   `recovery_code`: you see it once. It gives the key back at
   `POST /v1/purchase/recover`, and it is the workspace's management
   credential, so `recovery` is a ready recovery file for
   `POST /v1/workspaces/recover`.
2. Call the chat routes with `Authorization: Bearer <api_key>`. Use plain
   `curl` or any HTTP client here; you already paid, so `pay` has nothing
   to do. Each call costs the listed token price of the model that answered,
   plus 5%. On a non-streaming call, `x-phantom-cost-usd` and
   `x-phantom-balance-usd` show the cost and what is left. A stream sends its
   cost in an `event: phantom.receipt` before it ends.
   `GET /v1/key/balance` shows the credit at any time.

A paid response with `"status": "pending"` (a 503 or 409) means the transfer
is still confirming. Send the same request again or poll `status_url`, and
don't pay twice. Send the `recovery_code` on the poll in the
`x-phantom-recovery-code` header to collect the new key. A top-up to a key
you already hold needs no code.

To add credit to a key you hold, put it in the top-up body as
`target_api_key`. The credit goes to that key's workspace. Keep the key out
of the `Authorization` header, because `pay` replaces that header when it
pays over MPP. Phantom checks the key before it asks for money and refuses
an unknown or deactivated key with a 400. Sending the same paid request
twice adds the credit once.

Each priced response includes an Ed25519 receipt in `x-phantom-receipt` with
the model, tokens and cost. Check it against the public key at
`GET /v1/receipts/key`.

## Managing the key

These routes take the same Bearer key and cost nothing, except where noted.

- `GET` and `PATCH /v1/key/budget` read and set a spend cap per period and a
  per-minute rate.
- `/v1/key/route` holds a route policy that picks the model for
  `model: "auto"`. `POST /v1/key/route/test` shows what a request would get
  without calling a model. `/v1/key/lineup` sets which models each
  `phantom-1` tier may use.
- `GET /v1/key/spend` breaks down what the key spent by model and day.
- `POST /v1/key/rotate` gives you a new key with the same settings in the
  same workspace and retires the old one.
- `DELETE /v1/key` revokes the key. The credit stays in its workspace.
- `POST /v1/purchase/recover` gives a key back from its `recovery_code`.
- `POST /v1/messages/count_tokens` counts an Anthropic request's tokens, and
  `GET /v1/models/status` says whether a model is answering now.
- `GET /v1/privacy` lists what Phantom stores about the key and what it is
  linked to. Phantom keeps token counts and costs, never prompt or
  completion text.
- `POST /v1/receipts/anchor` writes a receipt's hash to Solana for $0.01.

To add keys that share the credit, or a monthly Pro or Max plan for
`phantom-1`, use the workspace's recovery credential with `/v1/workspaces`
and `/v1/plan`.

## Spend-aware usage

- Start with $0.10 to $1 and top up when `/v1/key/balance` runs low.
- Use `phantom-1` unless the task needs a particular model, and
  `phantom-1:free` when cost matters more than quality.
- Set the smallest `max_tokens` that works. Output tokens cost the most.
- Send only the history the task needs. You pay for input tokens too.
- Check `/v1/models` prices before you pick a large model by hand.
- Retry only when a request failed before inference. Phantom charges
  nothing for a failed upstream call.
