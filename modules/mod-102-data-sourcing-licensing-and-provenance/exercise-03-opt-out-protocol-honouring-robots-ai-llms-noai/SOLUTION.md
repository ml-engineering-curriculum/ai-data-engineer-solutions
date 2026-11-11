# Exercise 03 — Opt-Out Protocol: Honouring robots.txt, ai.txt, llms.txt, and the noai family (Solution)

## 1. Solution overview

This is an **implementation** exercise. The learner builds an opt-out check that
runs **before** any URL is admitted to the training corpus, treats the opt-out
signal as independent of (and higher precedence than) the licence signal from
Exercise 02, and leaves an auditable per-URL verdict.

A correct protocol:

1. Resolves the publisher's intent from **four signal families**, not one —
   `robots.txt`, HTML / HTTP meta signals (the `noai` / `noimageai` family and
   related `X-Robots-Tag` headers), site-level machine-readable files
   (`ai.txt`, `llms.txt`, and their `-full` variants), and any explicit
   rights-reservation in a response header or embedded metadata.
2. Applies a **publisher-controlled** precedence (strictest wins; silence is
   *not* consent).
3. Re-checks on a defined cadence, because an opt-out that ratchets in only
   counts if the pipeline re-reads it.
4. Emits a per-URL decision record so the dataset can be audited and so
   downstream "delete me" requests can be satisfied without a full retrain.

The cardinal rule: **a permissive licence does not override an opt-out.** A page
can be `CC-BY` and still request `noai`; the opt-out wins.

## 2. Worked answer

### 2.1 Signal reference

Each signal family is standardised or specified to different degrees. The
protocol treats "specified" as a continuum, not a boolean.

| Signal                             | Standardisation                                                                                            | Scope                                             |
|------------------------------------|------------------------------------------------------------------------------------------------------------|---------------------------------------------------|
| `robots.txt` (classic REP)         | IETF **RFC 9309** (Robots Exclusion Protocol).                                                             | Per-host; per-user-agent allow/disallow rules.    |
| `robots.txt` AI-specific agents    | Publisher-declared user-agent tokens (e.g. one per crawler); governed by RFC 9309 matching rules.          | Per-host; per-agent.                              |
| HTML `<meta name="robots">`         | HTML living standard / REP; values include `noindex`, `nofollow`, and vendor-declared `noai`, `noimageai`. | Per-document.                                     |
| HTTP `X-Robots-Tag`                | REP header form of the meta directive; same value grammar.                                                 | Per-response (any MIME type).                     |
| `ai.txt`                            | Community proposal (site-root file, inspired by `robots.txt`).                                                                                                                                                                       | Per-host.                                         |
| `llms.txt` / `llms-full.txt`       | Community proposal (site-root markdown file giving LLMs a map of the site).                                                                                                                                                      | Per-host; informational + optional directives.    |
| TDM rights reservation             | ML-training opt-out signal expressed in response headers and/or embedded metadata, framed by regimes such as the EU DSM Directive Article 4 for text-and-data-mining reservations.                                                                                                                                                                             | Per-response or per-asset. |
| Per-asset embedded metadata        | For images, audio, video: metadata fields in EXIF / XMP / IPTC / C2PA asserting rights or `do-not-train`.  | Per-asset.                                        |

### 2.2 Precedence rules

Apply in order, strictest first. The first rule that fires produces the final
verdict for that URL.

1. **Explicit opt-out at asset level** (EXIF / XMP / IPTC / C2PA / HTTP
   `X-Robots-Tag: noai`): **Deny**.
2. **Explicit opt-out at document level** (HTML meta `noai`, `noimageai`, or
   equivalent): **Deny**.
3. **Explicit opt-out at site level** (`ai.txt` or `robots.txt` disallowing our
   training user-agent token for the URL): **Deny**.
4. **TDM rights reservation signal** on the response or in a linked policy:
   **Deny**.
5. **Classic `robots.txt`** disallowing our training user-agent token for the
   URL: **Deny**.
6. **Site terms of service forbid training-oriented use**: **Deny**. (This is
   out-of-band — recorded in the source onboarding memo from Exercise 01, not
   read from HTTP.)
7. No signal found on any of the above: **Allow** — but only if the licence
   bucket from Exercise 02 is Allow or Conditional; otherwise defer to that
   bucket.

### 2.3 User-agent declaration

The pipeline MUST publish a stable, documented user-agent token and MUST honour
that token in `robots.txt` parsing. Example:

```text
User-Agent: AcmeDataBot/1.0 (+https://example.internal/acmedatabot)
```

- The URL in the user-agent string resolves to a page describing the bot's
  purpose, whether its collections are used for model training, and how to
  opt out via each supported channel.
- The pipeline matches this token against `robots.txt` rules per RFC 9309
  (longest-match path, group-selection, `Allow` beats `Disallow` at equal
  specific­ity).
- A pipeline that fails to declare a stable user-agent cannot honour targeted
  opt-outs and therefore cannot pass this exercise.

### 2.4 Reference implementation (Python, standard library only)

This is a minimal, statically-valid reference — it compiles and runs without
third-party packages. Production code would swap in a cached HTTP client, a
proper `robots.txt` library (e.g. one aware of RFC 9309 group matching), and
real `ai.txt` / `llms.txt` parsers once those specs stabilise.

```python
"""Opt-out protocol — reference implementation.

Call `check_url(url)`; returns (verdict, reason, evidence).
Verdict is one of: "allow", "deny".
"""

from __future__ import annotations

import dataclasses
import enum
import logging
import urllib.parse
import urllib.request
import urllib.robotparser

LOG = logging.getLogger(__name__)

USER_AGENT = "AcmeDataBot/1.0 (+https://example.internal/acmedatabot)"
TRAINING_AGENT_TOKEN = "AcmeDataBot"   # what robots.txt / ai.txt match against


class Verdict(str, enum.Enum):
    ALLOW = "allow"
    DENY = "deny"


@dataclasses.dataclass(frozen=True)
class Decision:
    verdict: Verdict
    reason: str            # which rule fired
    evidence_url: str      # where the signal was read from
    raw_snippet: str       # ≤200 chars of the signal, for the audit log


def _fetch(url: str, timeout: float = 10.0) -> tuple[int, dict[str, str], str]:
    req = urllib.request.Request(url, headers={"User-Agent": USER_AGENT})
    try:
        with urllib.request.urlopen(req, timeout=timeout) as resp:
            body = resp.read(65536).decode("utf-8", errors="replace")
            headers = {k.lower(): v for k, v in resp.headers.items()}
            return resp.status, headers, body
    except Exception as exc:
        LOG.warning("fetch failed for %s: %s", url, exc)
        return 0, {}, ""


def _host_root(url: str) -> str:
    parts = urllib.parse.urlsplit(url)
    return f"{parts.scheme}://{parts.netloc}"


def _check_robots_txt(url: str) -> Decision | None:
    rp = urllib.robotparser.RobotFileParser()
    rp.set_url(f"{_host_root(url)}/robots.txt")
    try:
        rp.read()
    except Exception as exc:
        LOG.info("robots.txt unreachable for %s: %s", url, exc)
        return None
    if not rp.can_fetch(TRAINING_AGENT_TOKEN, url):
        return Decision(
            Verdict.DENY,
            reason="robots.txt disallows training user-agent token",
            evidence_url=f"{_host_root(url)}/robots.txt",
            raw_snippet=f"User-Agent: {TRAINING_AGENT_TOKEN} => disallowed",
        )
    return None


def _check_ai_txt(url: str) -> Decision | None:
    # `ai.txt` is a community proposal; treat any Disallow for our token as Deny.
    ai_url = f"{_host_root(url)}/ai.txt"
    status, _headers, body = _fetch(ai_url)
    if status != 200 or not body:
        return None
    lines = [ln.strip() for ln in body.splitlines() if ln.strip()
             and not ln.strip().startswith("#")]
    current_agents: list[str] = []
    for ln in lines:
        if ":" not in ln:
            continue
        key, _, value = (p.strip() for p in ln.partition(":"))
        key_l = key.lower()
        if key_l == "user-agent":
            current_agents = [v.strip() for v in value.split(",")]
        elif key_l == "disallow":
            if any(a in ("*", TRAINING_AGENT_TOKEN) for a in current_agents):
                if value == "/" or urllib.parse.urlsplit(url).path.startswith(value):
                    return Decision(
                        Verdict.DENY,
                        reason="ai.txt disallow matches path",
                        evidence_url=ai_url,
                        raw_snippet=ln[:200],
                    )
    return None


def _check_llms_txt(url: str) -> Decision | None:
    # `llms.txt` is informational by design, but publishers sometimes include
    # an explicit allow/deny directive. Treat only explicit deny as Deny; absence
    # of allow is NOT consent.
    for name in ("llms-full.txt", "llms.txt"):
        feed_url = f"{_host_root(url)}/{name}"
        status, _headers, body = _fetch(feed_url)
        if status != 200 or not body:
            continue
        lowered = body.lower()
        if "disallow: training" in lowered or "noai" in lowered:
            return Decision(
                Verdict.DENY,
                reason=f"{name} declares an opt-out directive",
                evidence_url=feed_url,
                raw_snippet=body[:200],
            )
    return None


def _check_document_level(url: str) -> Decision | None:
    status, headers, body = _fetch(url)
    if status == 0:
        return None
    # HTTP response header form.
    xr = headers.get("x-robots-tag", "").lower()
    if any(tok in xr for tok in ("noai", "noimageai", "notrain", "nosnippet")):
        return Decision(
            Verdict.DENY,
            reason="X-Robots-Tag contains an opt-out directive",
            evidence_url=url,
            raw_snippet=f"X-Robots-Tag: {xr[:180]}",
        )
    # HTML meta form (naive regex on head is deliberate — pipeline should
    # replace with a proper HTML parser).
    import re
    m = re.search(
        r'<meta\s+name="robots"\s+content="([^"]+)"',
        body,
        re.IGNORECASE,
    )
    if m:
        content = m.group(1).lower()
        if any(tok in content for tok in ("noai", "noimageai", "notrain")):
            return Decision(
                Verdict.DENY,
                reason="HTML <meta name=robots> contains an opt-out directive",
                evidence_url=url,
                raw_snippet=f'<meta ... content="{content[:160]}">',
            )
    # TDM reservation header (publisher-specific; name not yet stabilised).
    for h in ("tdm-reservation", "tdm-policy", "x-tdm-reservation"):
        if h in headers and headers[h].strip() in ("1", "true", "yes"):
            return Decision(
                Verdict.DENY,
                reason=f"{h} header asserts TDM reservation",
                evidence_url=url,
                raw_snippet=f"{h}: {headers[h][:180]}",
            )
    return None


# Precedence order — strictest first. First rule that fires is final.
CHECKS = (
    _check_document_level,   # per-response: X-Robots-Tag, meta, TDM header
    _check_ai_txt,           # site-level ai.txt
    _check_llms_txt,         # site-level llms.txt / llms-full.txt
    _check_robots_txt,       # classic REP
)


def check_url(url: str) -> Decision:
    for check in CHECKS:
        decision = check(url)
        if decision is not None:
            return decision
    return Decision(
        Verdict.ALLOW,
        reason="no opt-out signal found in any checked channel",
        evidence_url=url,
        raw_snippet="",
    )
```

### 2.5 Per-URL decision record (audit schema)

```json
{
  "url": "https://example.com/post/42",
  "checked_at": "2026-11-11T12:34:56Z",
  "user_agent_token": "AcmeDataBot",
  "verdict": "deny",
  "reason": "ai.txt disallow matches path",
  "evidence_url": "https://example.com/ai.txt",
  "raw_snippet": "User-Agent: * Disallow: /",
  "signal_channels_checked": [
    "x-robots-tag", "meta-robots", "tdm-header",
    "ai.txt", "llms.txt", "robots.txt"
  ],
  "licence_bucket": "allow",          // from Exercise 02 — kept, but overridden
  "overrides_licence": true,
  "recheck_due": "2027-02-11T00:00:00Z"
}
```

Every record is appended to `opt-out-log.jsonl` on the dataset artifact; the
record is the primary evidence that the deny was honoured even if the source
site goes away later.

### 2.6 Re-check cadence

- Full site re-sweep: **quarterly** or before every full retrain, whichever is
  sooner.
- Per-URL re-check on cache expiry (default 30 days) or when the record is
  touched for incremental training.
- Any re-check that flips `allow → deny` triggers removal of the source rows
  from the next training snapshot. That is the hinge that makes opt-out
  *operational* rather than ceremonial — `recheck_due` in the decision record
  is what the re-sweep queries.

### 2.7 Non-goals

- The protocol does **not** validate that the publisher has standing to opt
  specific content out — it treats the publisher's declaration as authoritative
  for content served from their host.
- The protocol does **not** decide what to do about historical data that was
  admitted before the publisher published an opt-out. Deletion-on-opt-out is a
  separate control (mod-110).

## 3. Validation steps

1. **Unit tests at the signal boundaries.**
   - A fixture host with `robots.txt` disallowing `AcmeDataBot` → Deny.
   - A fixture host with an HTML `<meta name="robots" content="noai">` →
     Deny, with precedence higher than the host's `robots.txt` Allow.
   - A fixture asset with `X-Robots-Tag: noai` → Deny.
   - A fixture page with `CC-BY` licence and a `noai` meta tag → Deny
     (opt-out beats licence).
   - A fixture page with **no** signals → Allow if and only if the licence
     bucket from Exercise 02 is Allow / Conditional.
2. **Precedence integration test.** Compose two conflicting signals (e.g.
   `robots.txt` Allow + HTML meta `noai`) and assert the stricter one wins.
3. **User-agent declaration page resolves.** `curl` the URL baked into the
   `User-Agent` string — it must 200 and describe the bot. A bot with a
   dangling URL is not honouring the opt-out protocol in good faith.
4. **Re-sweep flips correctly.** Mutate a fixture's `ai.txt` from blank to a
   disallow-all; run the re-sweep; assert the per-URL record flips to `deny`
   and the corresponding row is pulled from the next snapshot plan.
5. **Logs include evidence snippet.** Every Deny in `opt-out-log.jsonl` has a
   non-empty `raw_snippet` so a reviewer six months later can see why.
6. **No signal silently maps to Allow.** Grep the log for `verdict=allow,
   reason=no opt-out signal found` and confirm those URLs also have an Allow /
   Conditional bucket in the Exercise 02 licence decision.

## 4. Rubric or review checklist

Score each row 0–2. Passing solution ≥ 20/26 and zero critical misses.

| # | Dimension                                                                                           | Critical? |
|---|-----------------------------------------------------------------------------------------------------|-----------|
| 1 | Pipeline declares a stable user-agent token and a public URL describing the bot                     | Yes       |
| 2 | Four signal families are all checked: REP (robots.txt), meta/header, `ai.txt`/`llms.txt`, TDM       | Yes       |
| 3 | Precedence table is written and matches the implementation                                          | Yes       |
| 4 | Opt-out beats licence (worked example present)                                                      | Yes       |
| 5 | Silence is *not* consent — "no signal found" only passes if the licence bucket already permits     | Yes       |
| 6 | Per-URL decision record captures verdict, reason, evidence URL, and a signal snippet                | Yes       |
| 7 | Re-check cadence is defined; `recheck_due` field exists                                             | Yes       |
| 8 | HTML parsing is at least acknowledged as "swap in a real parser" if a regex is used                  | No        |
| 9 | `robots.txt` matching follows RFC 9309 semantics (longest match, group selection)                   | No        |
| 10 | Open follow-up notes appear anywhere the spec has not been freshly re-read                         | No        |
| 11 | Pre-training retrain hook checks for new denies and excludes affected rows                          | Yes       |
| 12 | TDM reservation handled as its own signal, not merged with REP                                      | No        |
| 13 | The protocol refuses to grade opt-out standing — treats publisher declaration as authoritative      | No        |

## 5. Common mistakes

- **Only checking `robots.txt`.** A `robots.txt` Allow does not override an
  HTML `noai` meta tag on a specific page.
- **Treating "no signal" as consent.** Silence is not consent; the licence
  bucket must still be Allow / Conditional for the URL to pass.
- **Bot has no user-agent URL.** You cannot honour a targeted opt-out if
  publishers cannot address you.
- **Ignoring the asset-level channel** (EXIF / XMP / IPTC / C2PA). For images,
  audio, and video, the signal is on the asset, not the page.
- **One-time check.** Opt-out is stateful; without a re-sweep, publishers who
  opt out tomorrow are still in your corpus.
- **Letting licence override opt-out.** The licence tells you what you *may*
  do; the opt-out tells you what the publisher *has asked* you not to do.
  Opt-out wins.
- **Opaque audit log.** An audit entry that only says `verdict=deny` is not
  evidence; the raw snippet is the evidence.
- **Baking the training-agent token into HTTP but not into `ai.txt` matching.**
  Publishers address you through both channels — the token has to match in both.

## 6. References

- **RFC 9309 — Robots Exclusion Protocol.** IETF standardisation of
  `robots.txt`. <https://www.rfc-editor.org/rfc/rfc9309.html>
- **HTML `<meta name="robots">` / HTTP `X-Robots-Tag`.** Documented as part of
  the REP ecosystem; look up the current vendor-registered values for `noai`,
  `noimageai`, and related tokens before pinning a specific token list.
- **`ai.txt` community specification.** (Verify the current reference URL
  before citing; there are multiple competing drafts.)
- **`llms.txt` / `llms-full.txt` community specification.** (Verify the
  current reference URL before citing.)
- **TDM Reservation Protocol / EU DSM Directive Article 4.** The regime that
  motivates the TDM-reservation header family.
- NIST AI Risk Management Framework —
  <https://www.nist.gov/itl/ai-risk-management-framework>
  (GOVERN 1.4 — documented policies; MAP 4.1 — appropriateness of data use.)
- OWASP Machine Learning Security Top 10 —
  <https://owasp.org/www-project-machine-learning-security-top-10/>
  (ML02 Data Poisoning Attack: honouring opt-out shrinks the unvetted scrape
  surface where poisoned content lives.)
- MITRE ATLAS — <https://atlas.mitre.org/>
  (Browse the matrix for current data-supply-chain technique IDs before citing
  a specific ID in an incident write-up.)
- Local anchors: Exercise 01 for source selection, Exercise 02 for the licence
  precedence this protocol overrides, mod-110 for deletion-on-opt-out of
  historical rows.
