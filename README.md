# Zephr Share Secret — GitHub Action

Zero-knowledge one-time secret sharing for CI pipelines. Encrypt secrets on the runner, share them between jobs, and retrieve them exactly once — the server never sees plaintext.

Designed for zero-knowledge secret handoff between independent systems: AI agents, CI/CD pipelines, GitHub Actions, and human operators.

- [zephr.io](https://zephr.io)
- [API docs](https://zephr.io/docs)
- [npm package](https://www.npmjs.com/package/zephr)
- [PyPI package](https://pypi.org/project/zephr)

---

## How it works

1. A 256-bit key is generated on the runner. It never reaches Zephr's servers.
2. Your secret is encrypted with AES-GCM on the runner
3. Only the ciphertext is uploaded to Zephr
4. The link embeds the key in the URL fragment, which is never transmitted to servers
5. First retrieval atomically consumes the record. A second request returns 410.

---

## Usage

### Create a secret

```yaml
- uses: zephr-io/share-secret@v1
  id: create
  with:
    action: create
    secret: ${{ secrets.DB_PASSWORD }}
    api-key: ${{ secrets.ZEPHR_API_KEY }}
    hint: DB_PASSWORD_PROD
    expiry-minutes: '15'
```

### Retrieve a secret

```yaml
- uses: zephr-io/share-secret@v1
  id: retrieve
  with:
    action: retrieve
    link: ${{ steps.create.outputs.secret-link }}
    api-key: ${{ secrets.ZEPHR_API_KEY }}

- run: echo "Got the secret"
  env:
    DB_PASSWORD: ${{ steps.retrieve.outputs.plaintext }}
```

### Cross-job sharing

GitHub doesn't let you pass `secrets.*` between jobs natively. Zephr bridges the gap:

```yaml
jobs:
  share:
    runs-on: ubuntu-latest
    outputs:
      secret-link: ${{ steps.create.outputs.secret-link }}
    steps:
      - uses: zephr-io/share-secret@v1
        id: create
        with:
          action: create
          secret: ${{ secrets.DEPLOY_KEY }}
          api-key: ${{ secrets.ZEPHR_API_KEY }}
          expiry-minutes: '15'

  deploy:
    needs: share
    runs-on: ubuntu-latest
    steps:
      - uses: zephr-io/share-secret@v1
        id: retrieve
        with:
          action: retrieve
          link: ${{ needs.share.outputs.secret-link }}
          api-key: ${{ secrets.ZEPHR_API_KEY }}

      - run: ./deploy.sh
        env:
          DEPLOY_KEY: ${{ steps.retrieve.outputs.plaintext }}
```

### Split mode

Return the URL and encryption key as separate outputs for delivery through independent channels:

```yaml
- uses: zephr-io/share-secret@v1
  id: create
  with:
    action: create
    secret: ${{ secrets.API_KEY }}
    api-key: ${{ secrets.ZEPHR_API_KEY }}
    split: 'true'

# URL and key available separately:
# ${{ steps.create.outputs.secret-url }}
# ${{ steps.create.outputs.secret-key }}
```

### Webhook callback

Get notified when a secret is consumed — no polling needed:

```yaml
- uses: zephr-io/share-secret@v1
  id: create
  with:
    action: create
    secret: ${{ secrets.DB_PASSWORD }}
    api-key: ${{ secrets.ZEPHR_API_KEY }}
    hint: DB_PASSWORD_PROD
    callback-url: https://my-server.example.com/zephr-events
    callback-secret: ${{ secrets.ZEPHR_CALLBACK_SECRET }}
```

When the secret is retrieved, Zephr POSTs a signed event:

```json
{
  "event":       "secret.consumed",
  "event_id":    "550e8400-e29b-41d4-a716-446655440000",
  "secret_id":   "Ht7kR2mNqP3wXvYz8aB4cD",
  "occurred_at": "2026-03-22T14:32:00.000Z",
  "hint":        "DB_PASSWORD_PROD"
}
```

Verify the `X-Zephr-Signature` header — HMAC-SHA256 hex digest of the raw JSON body, signed with your `callback-secret`. Use timing-safe comparison. See [examples/webhook-receiver](https://github.com/zephr-io/zephr-sdk/tree/main/examples/webhook-receiver) for runnable receivers.

Fire-and-forget in v1 — no retries. 5-second timeout. Redirects blocked.

### Idempotency

The Action auto-generates an `Idempotency-Key` on every create — retries are safe by default. If a request times out and is replayed, the server returns the cached response without creating a duplicate. Cache TTL: 24 hours.

Pass your own key via the `idempotency-key` input for application-level retry control.

---

## Inputs

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `action` | Yes | — | `create` or `retrieve` |
| `secret` | Create only | — | Plaintext secret to encrypt |
| `link` | Retrieve only | — | Full Zephr link to retrieve |
| `api-key` | No | — | Zephr API key for authenticated requests |
| `hint` | No | — | Plaintext label for routing and audit logs (1-128 printable ASCII) |
| `expiry-minutes` | No | `60` | Expiry in minutes: 5, 15, 30, 60, 1440, 10080, or 43200 |
| `split` | No | `false` | Return URL and key as separate outputs |
| `callback-url` | No | — | HTTPS webhook URL for consumption events |
| `callback-secret` | No | — | HMAC-SHA256 signing secret for webhook (required with `callback-url`) |
| `idempotency-key` | No | — | Caller-generated idempotency key (1-64 alphanumeric + hyphens) |

Sub-hour expiry values (5, 15, 30) require a Dev or Pro API key. Expiry values above 60 minutes require a free account or higher.

---

## Outputs

| Output | Mode | Description |
|--------|------|-------------|
| `secret-link` | Create (standard) | Full shareable link |
| `secret-url` | Create (split) | Secret URL without key |
| `secret-key` | Create (split) | Encryption key |
| `secret-id` | Create | 22-char base64url identifier |
| `expires-at` | Create | ISO 8601 expiration timestamp |
| `plaintext` | Retrieve | Decrypted secret |
| `hint` | Retrieve | Plaintext hint label (if set at creation) |

---

## Authentication

Add `ZEPHR_API_KEY` as a [repository secret](https://docs.github.com/en/actions/security-guides/encrypted-secrets) and pass it as the `api-key` input. The Action works without an API key for anonymous use, but authenticated requests unlock higher limits and longer expiry.

| Tier | Create limit | Expiry options | Max size |
|------|-------------|----------------|----------|
| Anonymous | 3/day | 1h | 6 KB |
| Free | 50/month | 1h, 24h, 7d, 30d | 20 KB |
| Dev ($15/mo) | 2,000/month | 5m, 15m, 30m, 1h, 24h, 7d, 30d | 200 KB |
| Pro ($39/mo) | 50,000/month | 5m, 15m, 30m, 1h, 24h, 7d, 30d | 1 MB |

**Getting an API key:** Log in at [zephr.io/account](https://zephr.io/account), open the API Keys tab, and create a key. The raw key is shown exactly once. Copy it immediately.

---

## Security

- **Zero-knowledge**: the server never receives your plaintext or encryption keys
- **Local encryption**: AES-GCM-256 on the runner before any network call
- **Key isolation**: keys travel in the URL fragment ([RFC 3986 §3.5](https://datatracker.ietf.org/doc/html/rfc3986#section-3.5)), which is never transmitted to servers
- **One-time access**: the record is permanently deleted after a single retrieval
- **Output masking**: `secret-link`, `secret-url`, `secret-key`, and `plaintext` outputs are masked via `core.setSecret()` — they never appear in workflow logs
- **Input masking**: `secret`, `link`, `api-key`, and `callback-secret` inputs are also masked defensively
- **No filesystem persistence**: secrets flow through action outputs only, never written to disk
- **API key**: always pass via `${{ secrets.ZEPHR_API_KEY }}` — never hardcode

---

## Issues

[Open an issue](https://github.com/zephr-io/share-secret/issues)

## License

MIT
