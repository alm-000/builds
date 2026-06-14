# Email Discovery Tool

Snippet: sanitised extract from `email_permutator.py` — email pattern generation and SMTP verification.

---

## Problem

Most B2B and creator outreach is bottlenecked at the contact-finding step: find a real email for the right person. Tools like Hunter work at scale but miss niche targets (festival promoters, small venue operators, independent creators). Manual research doesn't scale — browse a site, look for a booking page, log the email, move on. Multiplied across hundreds of targets, this consumed significant time before any actual outreach happened.

The deeper problem: manual processes introduce inconsistency. Some records have emails, some have contact forms, some have nothing. Data sitting in a spreadsheet degrades immediately — no status tracking, no deduplication, no way to know what was already contacted.

## Approach

A crawl-and-enrich pipeline that takes a list of site URLs or Instagram handles and outputs verified, enriched contact records:

**1. Crawl** — fetches the target site and navigates to contact, about, and booking pages using pattern-based routing.

**2. Extract** — finds email patterns in page source, visible text, and linked pages. Checks linked social profiles for contact info.

**3. Verify** — email permutation generation (common `first.last@domain` patterns) plus SMTP RCPT TO probing to verify mailbox existence without sending a message. Flags low-confidence extractions for manual review rather than silently passing them through. Detects catch-all domains (where any RCPT TO succeeds) and discounts confidence accordingly.

**4. Enrich** — pulls categorisation labels, site name, niche tags, geography signals. Cross-references with directory discovery engine output where available.

**5. Deduplicate and sync** — every record gets a dedupe key by domain. Already-processed domains are skipped. Clean records land in the CRM table with status `ready_for_outreach`.

For Instagram-first targets, the pipeline cross-references public bio links and linked-in-bio tools to find emails not visible on the main profile.

## Snippet — email permutation + SMTP verification

The core of the verification logic: generates candidate email patterns from a name and domain, then probes each via SMTP RCPT TO (free, no third-party API).

```python
import re
import socket
import smtplib
import dns.resolver
import time
from collections import defaultdict


def generate_permutations(first: str, last: str, domain: str) -> list[str]:
    """
    Generate common email permutations from a name + domain.
    Returns ~8 candidate emails, all lowercased.
    """
    first = re.sub(r'[^a-z]', '', first.lower().strip())
    last = re.sub(r'[^a-z]', '', last.lower().strip())

    if not first or not last or not domain:
        return []

    f = first[0]  # first initial

    patterns = [
        f"{first}.{last}",     # john.smith@
        f"{first}",            # john@
        f"{f}{last}",          # jsmith@
        f"{first}{last[0]}",   # johns@
        f"{first}_{last}",     # john_smith@
        f"{f}.{last}",         # j.smith@
        f"{last}",             # smith@
        f"{last}.{first}",     # smith.john@
    ]

    return [f"{p}@{domain.lower()}" for p in patterns]


_mx_cache: dict[str, str | None] = {}


def _get_mx_host(domain: str) -> str | None:
    """Look up the MX record for a domain. Caches results per domain."""
    if domain in _mx_cache:
        return _mx_cache[domain]
    try:
        records = dns.resolver.resolve(domain, 'MX')
        mx_records = sorted(records, key=lambda r: r.preference)
        if mx_records:
            host = str(mx_records[0].exchange).rstrip('.')
            _mx_cache[domain] = host
            return host
    except (dns.resolver.NoAnswer, dns.resolver.NXDOMAIN,
            dns.resolver.NoNameservers, dns.exception.Timeout):
        pass
    _mx_cache[domain] = None
    return None


SMTP_TIMEOUT = 30
MAX_RETRIES = 3
BACKOFF_BASE = 2


def smtp_verify(email: str, sender_domain: str = "yourco.example.com",
                timeout: int = SMTP_TIMEOUT) -> dict:
    """
    Verify if a mailbox exists via SMTP RCPT TO probe.
    Retries up to MAX_RETRIES times on transient failures.
    Returns {email, valid, reason, mx_host}.

    Pass sender_domain from your environment — it is used in HELO
    and MAIL FROM; it does not need to receive mail.
    """
    result = {"email": email, "valid": False, "reason": "unknown", "mx_host": None}

    domain = email.split("@")[-1] if "@" in email else None
    if not domain:
        result["reason"] = "invalid_format"
        return result

    mx_host = _get_mx_host(domain)
    if not mx_host:
        result["reason"] = "no_mx_record"
        return result

    result["mx_host"] = mx_host

    for attempt in range(MAX_RETRIES):
        try:
            smtp = smtplib.SMTP(timeout=timeout)
            smtp.connect(mx_host, 25)
            smtp.helo(sender_domain)
            smtp.mail(f"verify@{sender_domain}")
            code, message = smtp.rcpt(email)
            smtp.quit()

            if code == 250:
                result["valid"] = True
                result["reason"] = "accepted"
            elif code == 550:
                result["reason"] = "mailbox_not_found"
            elif code == 553:
                result["reason"] = "mailbox_not_allowed"
            elif code == 452:
                result["reason"] = "mailbox_full"
            else:
                result["reason"] = f"smtp_{code}"

            return result

        except (smtplib.SMTPServerDisconnected, smtplib.SMTPConnectError,
                socket.timeout, TimeoutError, OSError):
            if attempt < MAX_RETRIES - 1:
                time.sleep(BACKOFF_BASE ** attempt)
            continue

    result["reason"] = "inconclusive"
    return result


def detect_catch_all(domain: str, timeout: int = 10) -> bool:
    """Test if a domain accepts any address (catch-all).
    Sends RCPT TO for a guaranteed-fake address — if accepted, domain is catch-all.
    """
    fake = f"zzzfake9999test@{domain}"
    result = smtp_verify(fake, timeout=timeout)
    return result["valid"]


def find_email_for_person(first: str, last: str, domain: str,
                          timeout: int = 10) -> dict | None:
    """Generate permutations, SMTP-verify, return best match with confidence score.

    Confidence is discounted for catch-all domains (can't distinguish real from fake).
    Returns {email, confidence, catch_all} or None.
    """
    permutations = generate_permutations(first, last, domain)
    if not permutations:
        return None

    catch_all = detect_catch_all(domain, timeout=timeout)
    results = batch_verify(permutations, timeout=timeout)
    valid = [r for r in results if r["valid"]]

    if not valid:
        return None

    # Prefer first.last, then first-initial+last, then first-only
    first_l = first.lower().strip()
    last_l = last.lower().strip()
    preferred_order = [
        f"{first_l}.{last_l}@{domain.lower()}",
        f"{first_l[0]}{last_l}@{domain.lower()}" if first_l else None,
        f"{first_l}@{domain.lower()}",
    ]

    best = valid[0]
    for pref in preferred_order:
        if pref and any(v["email"] == pref for v in valid):
            best = next(v for v in valid if v["email"] == pref)
            break

    if best["email"] == f"{first_l}.{last_l}@{domain.lower()}":
        confidence = 0.8
    elif first_l and best["email"].startswith(f"{first_l[0]}{last_l}@"):
        confidence = 0.7
    else:
        confidence = 0.65

    if catch_all:
        confidence *= 0.6  # e.g. 0.8 → 0.48 — still surfaced, but flagged

    return {
        "email": best["email"],
        "confidence": round(confidence, 2),
        "catch_all": catch_all,
        "mx_host": best.get("mx_host"),
    }


def batch_verify(emails: list[str], timeout: int = 10) -> list[dict]:
    """Verify a list of emails with rate limiting.
    Groups by domain to reuse MX lookups; adds 1s delay between checks
    to the same MX host to avoid triggering spam guards.
    """
    by_domain: dict[str, list[str]] = defaultdict(list)
    for email in emails:
        domain = email.split("@")[-1] if "@" in email else "invalid"
        by_domain[domain].append(email)

    results = []
    last_check_time: dict[str, float] = {}

    for domain, domain_emails in by_domain.items():
        mx_host = _get_mx_host(domain)
        for email in domain_emails:
            if mx_host and mx_host in last_check_time:
                elapsed = time.time() - last_check_time[mx_host]
                if elapsed < 1.0:
                    time.sleep(1.0 - elapsed)
            result = smtp_verify(email, timeout=timeout)
            results.append(result)
            if mx_host:
                last_check_time[mx_host] = time.time()

    return results
```

**What the snippet shows:** pattern generation, SMTP RCPT TO probing with retry + backoff, catch-all domain detection, confidence scoring, per-MX rate limiting. This is the core verification engine that plugged into the larger crawl-and-enrich pipeline.

## Stack

- **Python** — verification engine (above)
- **n8n** — orchestration, crawling, CRM sync trigger
- **Supabase** — contact records, deduplication, outreach status

## Results

- Removed the manual contact-finding bottleneck from the outreach workflow entirely
- Fed directly into the CRM engine's enrichment and scoring layer — automated path from target identification to personalised outreach
- Contact acquisition foundation for festival, venue, and creator partnership outreach
- Pattern transfers to any context where you are building outreach lists from scratch against niche targets
