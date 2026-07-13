# Theme Webhooks — Partner Integration Guide

Partners receive HTTP callbacks from `okta-web` whenever a lifecycle
event happens on one of their themes. This document describes the
contract and shows how to verify the signature on the receiving side.

## Events

| Event                         | Fired when                                               |
| ----------------------------- | -------------------------------------------------------- |
| `theme.installed`             | A tenant installs the theme                              |
| `theme.uninstalled`           | A tenant uninstalls the theme                            |
| `theme.version_published`     | A new version of the theme is published via the API      |
| `theme.rating_created`        | A tenant leaves a rating (future)                        |

## Registration

```bash
curl -X POST https://okta-web.example/api/partners/webhooks \
  -H "Authorization: Bearer $PARTNER_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "url": "https://hooks.my-partner.com/okta",
    "events": ["theme.installed", "theme.uninstalled"]
  }'
```

The response returns the webhook id and a **one-time** `secret`:

```json
{
  "id": 17,
  "url": "https://hooks.my-partner.com/okta",
  "events": ["theme.installed", "theme.uninstalled"],
  "is_active": true,
  "secret": "Gh7fD3kQ…"
}
```

Store the secret alongside the webhook row in your system —
`okta-web` never reveals it again.

## Request shape

Every webhook delivery is a `POST` with headers:

| Header               | Meaning                                              |
| -------------------- | ---------------------------------------------------- |
| `X-Okta-Event`       | The event name (see table above)                     |
| `X-Okta-Signature`   | HMAC-SHA256 of the raw body using your secret        |
| `Content-Type`       | `application/json`                                   |

Body envelope:

```json
{
  "event": "theme.installed",
  "partner_tenant_id": 42,
  "dispatched_at": "2026-04-17T12:00:00Z",
  "data": {
    "tenant_id": 1001,
    "tenant_slug": "acme",
    "module_slug": "modern-academy",
    "installed_version": "1.2.0",
    "installed_at": "2026-04-17T12:00:00Z"
  }
}
```

## Signature verification (PHP)

```php
$secret   = getenv('OKTA_WEBHOOK_SECRET');
$body     = file_get_contents('php://input');
$received = $_SERVER['HTTP_X_OKTA_SIGNATURE'] ?? '';

$expected = hash_hmac('sha256', $body, $secret);

if (! hash_equals($expected, $received)) {
    http_response_code(401);
    exit('Invalid signature');
}
```

## Signature verification (Node)

```js
import crypto from 'node:crypto';

export function verify(rawBody, signature, secret) {
  const expected = crypto
    .createHmac('sha256', secret)
    .update(rawBody)
    .digest('hex');

  return crypto.timingSafeEqual(
    Buffer.from(expected, 'hex'),
    Buffer.from(signature, 'hex'),
  );
}
```

## Response contract

Return a `2xx` status within **5 seconds** to mark the delivery as
successful. Any other response (or timeout) marks the delivery as
failed; `okta-web` retries failed deliveries hourly for up to **72
hours** via a scheduled job.

Your handler should be **idempotent**: retries may re-deliver the
same event up to 72 times, so dedupe by the `dispatched_at` field or
your own processed-events log.

## Troubleshooting

- Check `partner_webhook_deliveries` from your partner dashboard.
  Each delivery row records the response status and body excerpt.
- Webhooks auto-deactivate after repeated failures — check
  `failure_count` on the webhook row.
- Re-issuing the secret: delete the old webhook and create a new
  one. Secrets cannot be rotated in place.
