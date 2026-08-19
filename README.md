# n8n-nodes-threadly

An n8n community node for [Threadly](https://www.usethreadly.co) — an AI social-listening and
reply-drafting tool for X/Twitter. Every reply Threadly drafts sits in an Approval Inbox until
a human reviews it; nothing is ever posted automatically. This node lets an n8n workflow react
to what Threadly finds and participate in that review step.

**New to Threadly?** [Start a free 7-day trial](https://www.usethreadly.co?utm_source=n8n&utm_medium=community_node&utm_campaign=n8n) — no card required.

## Installation

Follow the n8n [community nodes installation guide](https://docs.n8n.io/integrations/community-nodes/installation/),
using `n8n-nodes-threadly` as the package name.

## Credentials

You'll need a Threadly **API key**, generated from your project's Settings → API Keys page
(currently API-only — create one with):

```bash
curl -X POST https://<your-threadly-deployment>/api/v1/projects/{project_id}/api-keys \
  -H "Content-Type: application/json" \
  --cookie "growthos_session=<your dashboard session cookie>" \
  -d '{"name": "n8n"}'
```

The response's `full_key` is shown once — that's what goes into the node's credential.

## Resources & operations

- **Conversation** → List — X conversations Threadly has discovered for your project
- **Draft** → List / Approve / Reject — replies Threadly has drafted, awaiting your decision
- **Reply** → List — replies that have already been posted

## Trigger

**Threadly Trigger** fires whenever Threadly discovers a new conversation
(`conversation.discovered`). Activating the workflow registers a webhook subscription with
Threadly automatically; deactivating it tears the subscription back down. No polling.

## Compatibility

Tested against n8n's declarative node API v1. Requires a Threadly deployment running the
public API (see the reference below).

## API reference

Every route requires `Authorization: Bearer <api_key>`. Keys are project-scoped — one key
acts on exactly one project — and are generated from Settings → API Keys in your Threadly
dashboard.

All endpoints are under `/public/v1`:

| Method | Path | Notes |
|---|---|---|
| GET | `/conversations` | X conversations Threadly has discovered. `tag`, `limit`, `offset` query params. |
| GET | `/drafts` | Content items awaiting/given a decision. `status` (default `pending_review`), `limit`, `offset`. |
| POST | `/drafts/{id}/approve` | Approves a draft. Attributed to the API key's creator — a key whose creator's account was deleted is rejected with 401. |
| POST | `/drafts/{id}/reject` | Body: `{"reason": "..."}`. Same attribution as approve. |
| GET | `/replies` | Content items already published. `limit`, `offset`. |
| POST | `/webhook-subscriptions` | Body: `{"target_url": "https://...", "event_types": ["conversation.discovered"]}`. `target_url` must be `https://` and not point at localhost or a private IP. Returns a `secret` shown once, used to verify delivery signatures. |
| GET | `/webhook-subscriptions` | Lists this project's subscriptions (never includes `secret`). |
| DELETE | `/webhook-subscriptions/{id}` | Revokes a subscription. |

Rate limit: 120 requests burst, 2/second sustained, per key.

### Webhook delivery

`conversation.discovered` fires whenever Threadly discovers a new lead. Delivered as `POST`
to your `target_url`:

```json
{
  "event": "conversation.discovered",
  "delivery_id": "<uuid>",
  "occurred_at": "2026-08-16T10:00:00Z",
  "data": { "knowledge_item_id": "...", "url": "...", "platform": "twitter", "buying_intent": "high" }
}
```

Headers:

- `X-Threadly-Event: conversation.discovered`
- `X-Threadly-Delivery: <delivery id>` — stable per delivery attempt, safe to use as an
  idempotency key
- `X-Threadly-Signature: sha256=<hex>` — HMAC-SHA256 of the raw request body, keyed by the
  subscription's `secret`. Verify with `hmac.new(secret, raw_body, sha256).hexdigest()` and a
  constant-time comparison.

Retries on failure with backoff (30s, 2m, 10m, 1h, 6h), terminal `failed` status after 5
attempts. No retry-triggered duplicate deliveries — one row per (subscription, event) pair.

## License

[MIT](LICENSE.md)
