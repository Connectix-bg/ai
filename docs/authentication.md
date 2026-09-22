# Authentication

# Authentication

Every request carries an application token in the `Authorization` header. There is no OAuth, no signing and no session.

```bash
curl "https://api-sandbox.connectix.bg/me" \
  -H "Authorization: $CONNECTIX_SANDBOX_TOKEN" \
  -H "Accept: application/json"
```

## Raw token or Bearer

Both forms are accepted and mean the same thing:

```
Authorization: 6f1d2c3b-...-9a8b7c6d
Authorization: Bearer 6f1d2c3b-...-9a8b7c6d
```

Use whichever your HTTP client produces by default. Tokens are opaque strings (UUID-shaped); never derive anything from their format.

## Two tokens per application, one per host

Each application in the console has two tokens, and the host decides which one is checked:

| Host | Token that works | Delivery | Billing |
|---|---|---|---|
| `api-sandbox.connectix.bg` | sandbox token | simulated | none |
| the production host | live token | real | real |

A sandbox token sent to the production host, or a live token sent to the sandbox, is simply an unknown token: the API answers **401**. That is by design, so a sandbox integration can never send a real message by accident.

The sandbox token is the one shown on the console's AI tools page and the only one these docs use. The live token stays on the application settings page and is handled by a person, following the go-live guide.

## Integration tokens

Some operations identify the *integration* (a shop, an automation) rather than the application:

- `POST /integration/register` returns an integration `token` for the `secret` you supply.
- `POST /shipments` requires that token as the `?integration=` query parameter.
- `POST /messages` and `POST /messages/fallback` accept the same `?integration=` parameter to attribute the message to an integration.

The application token stays in the `Authorization` header in every case; the integration token is an additional identifier, not a replacement.

## Keeping tokens safe

- Store tokens in environment variables or your secret store, never in source code, config files committed to git, client-side JavaScript or mobile apps.
- Rotate a token from the console if it leaks; the old value stops working immediately.
- Send `Accept: application/json`: without it some error responses are HTML pages.

## Errors

| Status | Meaning |
|---|---|
| 401 | Missing `Authorization` header, unknown token, or the right token on the wrong host |
| 401 | The company behind the token is not active or its traffic is banned |
| 403 | The token is valid but this application is restricted from the resource (e.g. TrustCheck) |

