# personal-finance-sync

**Theme:** full-stack · AI/automation

## Problem

Maintaining a live view of balances and transactions across multiple UK bank accounts and credit cards requires logging into each provider separately. Open Banking APIs solve this but add complexity: OAuth flows, token expiry, refresh cycles, and consent windows must all be managed correctly or the connection goes stale.

## Approach

A Python service integrating with the TrueLayer Open Banking aggregation API to pull balances and transactions from multiple connected providers into a central store.

**Key design decisions:**

- **Proactive token refresh** — tokens are checked for expiry (with a configurable buffer, e.g. 5 minutes) before each API call. If close to expiry, the refresh happens transparently and the new tokens are persisted. No silent failures from expired tokens mid-run.
- **Consent tracking** — TrueLayer consent windows are 90 days. The connection record stores `consent_expires_at` so a scheduled job can warn before connections silently stop working.
- **Scoped connection model** — each bank connection is a separate record with its own access/refresh token pair, supporting multiple providers simultaneously.
- **No pandas** — transaction pivots and aggregations are done with stdlib `collections` and `dict` comprehensions. The processing stays fast and the dependency footprint stays small.

## Architecture

```
truelayer.py           — OAuth lifecycle: auth URL, code exchange, token refresh
  ensure_valid_token() — checks expiry, refreshes if needed, persists new tokens
  fetch_accounts()     — list current/savings accounts
  fetch_cards()        — list credit cards
  fetch_account_transactions() — transactions in date range
sheets.py              — no-pandas pivot: transactions → category totals → spreadsheet
```

## Real results / metrics

- Pulls from multiple providers in a single run (~2–3 seconds per provider for balance + last 30 days of transactions).
- Token refresh has run unattended for months without a manual re-auth (within the 90-day consent window).
- Category pivot produces a month-by-month summary across all accounts in under a second — no pandas, no large dependencies.

## Snippet

OAuth token lifecycle — auth URL generation, code exchange, and proactive refresh:

```python
SCOPES = [
    "info", "accounts", "balance",
    "transactions", "cards", "offline_access",
]

def get_auth_url(provider: str | None = None) -> str:
    """Generate TrueLayer auth URL. Optionally pre-select a provider."""
    params = {
        "response_type": "code",
        "client_id": TRUELAYER_CLIENT_ID,
        "scope": " ".join(SCOPES),
        "redirect_uri": TRUELAYER_REDIRECT_URI,
        "response_mode": "form_post",
    }
    if provider:
        params["providers"] = f"uk-ob-{provider}"
    return f"{TRUELAYER_AUTH_BASE}/?{urlencode(params)}"


def ensure_valid_token(connection: dict) -> str:
    """Check token expiry, refresh if needed, return valid access_token."""
    db = get_client()
    expires_at = connection.get("token_expires_at")

    if expires_at:
        expiry = datetime.fromisoformat(expires_at.replace("Z", "+00:00"))
        now = datetime.now(timezone.utc)
        if now + timedelta(seconds=TOKEN_REFRESH_BUFFER_SECS) < expiry:
            return connection["access_token"]

    # Token expired or about to — refresh
    tokens = refresh_access_token(connection["refresh_token"])
    new_expires = datetime.now(timezone.utc) + timedelta(seconds=tokens["expires_in"])

    db.table("connections").update({
        "access_token":    tokens["access_token"],
        "refresh_token":   tokens.get("refresh_token", connection["refresh_token"]),
        "token_expires_at": new_expires.isoformat(),
    }).eq("id", connection["id"]).execute()

    return tokens["access_token"]
```
