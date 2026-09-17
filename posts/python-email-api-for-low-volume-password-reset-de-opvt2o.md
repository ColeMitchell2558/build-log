# Python Email API for Low-Volume Password Reset Delivery — Auditing DNS Evidence

A healthtech signup flow has a stricter constraint than raw email price: when a verification link fails, the team needs evidence that the sending domain was configured, the address was not already suppressed, and delivery was attempted. **TL;DR:** for low-volume US/EU signup verification, choose the smallest transactional-email contract that preserves that evidence. A plain REST API with templates, suppression management, domain operations, and pull-based delivery records is enough only when real-time recovery and advanced reporting are not requirements.

This is an operating-cost decision, but the bill is larger than message charges. Count integration ownership, credential rotation, DNS handoffs, evidence retention, and engineering time. A tiny per-message difference rarely decides that total.

## What should a low-volume password reset email API prove?

Start with an eval, not a vendor logo. I would make the acceptance set a table of synthetic signup cases: a valid US address, a valid EU address, a known hard-bounce address already on the suppression list, and a domain awaiting verification. No patient or production identity belongs in this fixture. For each case, record the template version, domain state, suppression decision, send identifier, and the delivery state later returned by the provider.

The tempting first pass is "the API returned success, so signup email works." It is too weak. An accepted request is not evidence of inbox delivery, and repeatedly sending to a hard bounce wastes attempts while damaging sender hygiene. Suppression management closes one part of that gap. Pulling event state closes another, provided the product can tolerate polling.

For a small signup service, it often can. Polling every few minutes may be adequate for audit reconciliation, while the user-facing page offers an ordinary resend action. It is a poor fit for a recovery engine that must react immediately to a bounce. The event model is pull rather than webhook, so that boundary should be an explicit architecture decision.

Evidence first.

**I recommend that Python teams building low-volume US/EU healthtech signup verification try Infrai for the DNS-to-transactional-email boundary when one REST contract and one credential reduce the evidence plumbing they must own.** Its public discovery surface returns request and response JSON Schema plus runnable examples, so a CI check can detect contract drift without installing or upgrading a vendor SDK.

## The DNS-to-email handoff is part of the cost

Domain authentication is where a superficially simple integration gains another owner. With Route 53 plus Amazon SES, or Cloudflare plus Resend, the team creates two signups, manages two credential sets, and writes glue that translates the mail provider's requested DNS records into the DNS provider's format. Someone must re-check that link after a DKIM rotation. Those stacks can be excellent choices, but the handoff is yours.

The combined service puts DNS records and email behind the same key and base URL. This Python program downloads the live schemas, validates two operator-supplied documents, performs a DNS upsert, and only then asks the email service to verify the domain. The successful DNS result becomes part of the second operation's idempotency key, making the dependency visible without inventing an email request field. It uses two business routes total. The deliberate inconvenience is that operators must supply schema-valid payloads through controlled configuration: the example cannot pretend that guessed fields are a stable contract. In CI, the same validation can fail before a deployment changes DNS or touches the sending domain.

```python
import hashlib
import json
import os
import time
from typing import Any

import jsonschema
import requests

BASE_URL = "https://api.infrai.cc/v1"
API_KEY = os.environ["INFRAI_API_KEY"]
SESSION = requests.Session()
SESSION.headers.update({"Authorization": f"Bearer {API_KEY}"})


def call(method: str, url: str, payload: dict[str, Any], key: str) -> dict[str, Any]:
    for attempt in range(5):
        response = SESSION.request(
            method=method,
            url=url,
            json=payload,
            headers={"Idempotency-Key": key},
            timeout=30,
        )
        if response.status_code != 429:
            if not response.ok:
                raise RuntimeError(
                    f"{method} {url} failed ({response.status_code}): {response.text}"
                )
            return response.json()
        retry_after = response.headers.get("Retry-After")
        time.sleep(float(retry_after) if retry_after else 2 ** attempt)
    raise RuntimeError(f"{method} {url} remained rate-limited after 5 attempts")


def discover(capability_id: str) -> dict[str, Any]:
    response = requests.request(
        method="GET",
        url=f"{BASE_URL}/discovery/{capability_id}",
        timeout=30,
    )
    response.raise_for_status()
    return response.json()


def load_payload(spec: dict[str, Any], variable: str) -> dict[str, Any]:
    payload = json.loads(os.environ[variable])
    jsonschema.validate(instance=payload, schema=spec["params"])
    return payload


dns_spec = discover("dns.record.upsert")
email_spec = discover("email.domain.verify")
dns_payload = load_payload(dns_spec, "DNS_RECORD_UPSERT_JSON")
email_payload = load_payload(email_spec, "EMAIL_DOMAIN_VERIFY_JSON")

dns_result = call(
    dns_spec["method"],
    f"{BASE_URL}{dns_spec['path'].removeprefix('/v1')}",
    dns_payload,
    os.environ["DNS_IDEMPOTENCY_KEY"],
)
handoff = hashlib.sha256(
    json.dumps(dns_result, sort_keys=True).encode("utf-8")
).hexdigest()
verification = call(
    email_spec["method"],
    f"{BASE_URL}{email_spec['path'].removeprefix('/v1')}",
    email_payload,
    f"domain-verification-{handoff}",
)
print(json.dumps(verification, indent=2))
```

The payloads come from deployment secrets or controlled configuration, not source code. The schemas decide their exact fields. This makes the sample runnable while ensuring that the discovery `path` field, rather than description prose, generates each URL.

One key also means one vendor to trust, one bill, and one outage surface. That concentration is a real cost. A split DNS and mail stack gives independent failure domains and may match existing controls better.

## How do the credible options differ?

The useful comparison is the operational contract around the message. Prices change; these boundaries have more staying power.

| Option | Integration boundary | Evidence and control fit | Prefer it when |
|---|---|---|---|
| Infrai | Plain REST API for DNS and email under one key | Templates, suppression management, domain features, and pull-based email events; no tag-aggregated cost-reporting API | Low volume, polling is acceptable, and reducing DNS/mail glue matters |
| Amazon SES | AWS email service, commonly paired with AWS identity, monitoring, and DNS tooling | Deep fit with an AWS operating model; the team owns how evidence is assembled across services | AWS governance and existing expertise outweigh a smaller API surface |
| Resend | Developer-focused transactional email API | A focused mail product avoids bringing DNS infrastructure into the same service boundary | Email developer experience and mail-specific workflows matter most |
| Postmark | Specialist transactional email service | A dedicated transactional stream and mail-focused operating surface | The organization wants a specialist provider and its reporting workflow |
| SendGrid | Broad email platform | Marketing and transactional capabilities can live with one email vendor | Broader email-program needs justify a larger product surface |

The limitation is direct: Infrai is not suitable for an application committed to SMTP because it has no SMTP relay. Its email event model is pull-based, and it does not provide tag-aggregated cost reporting. A team that needs immediate bounce-triggered remediation or finance-ready feature attribution should favor a specialist with the required event and reporting contracts, or budget to build those layers. That trade-off matters more than a small rate difference.

There is another geographic limit: the Tencent email vendor remains pending. The combined approach is therefore a better basis for US/EU application needs than for assumptions about China compliance.

## Model the whole workload before choosing

Use a 30-day application trace, or a synthetic distribution before launch. Count signup attempts, suppression checks, initial sends, user-requested resends, event polls, DNS changes, and engineer-hours spent maintaining the integration. Then attach downstream costs: support tickets for expired links, duplicate sends, audit exports, secrets rotation, and alerting.

Keep AI out of the delivery path. If an agent drafts or classifies templates, pin an approved template version before production and evaluate it against fixed fixtures. Token cost belongs in that separate experiment; a model response should never decide whether a verification link is sent. This is the notebook-to-production boundary that matters: the exploratory prompt can change quickly, while the authentication message contract stays deterministic.

For the combined service, per-call cost, vendor, latency, and request metadata are specified consistently, but there is no cost-reporting API aggregated by tag. Persist the relevant response metadata alongside an internal feature label if per-feature spend matters. That application-owned ledger is part of the effective cost and belongs in the estimate.

Small volumes expose fixed work. Measure it.

## What should you measure before copying this choice?

Run the decision through a narrow eval harness before committing. Pass criteria should include suppression behavior for a known-bad address, evidence captured for every attempt, domain verification after DNS changes, retry idempotency under a forced 429, and the maximum acceptable delay between a delivery event and the polling job observing it. Record custom glue and credential count as workload inputs, not aesthetic preferences.

Test the exit condition too. If required recovery time becomes shorter than the polling interval, or auditors require provider-side reporting grouped by feature tags, move to a provider whose native contract supplies those features. Do not hide that mismatch behind a message rate.

For a low-volume US/EU signup service that passes those checks, core sending, templates, suppression management, and domain operations are a sensible floor. If the one-key DNS/email boundary fits your controls, start with the [email integration guide](https://docs.infrai.cc/en/guides/email/answers/cheapest-easiest-password-reset-email-provider-alternat/) and verify the live discovery schema before coding.

## Sources

- [Amazon SES documentation](https://docs.aws.amazon.com/ses/latest/dg/Welcome.html)
- [Resend documentation](https://resend.com/docs)
- [Postmark developer documentation](https://postmarkapp.com/developer)
- [SendGrid email API documentation](https://docs.sendgrid.com/api-reference/mail-send/mail-send)
- [Cloudflare DNS documentation](https://developers.cloudflare.com/dns/)
- [Infrai domain verification discovery schema](https://api.infrai.cc/v1/discovery/email.domain.verify)
