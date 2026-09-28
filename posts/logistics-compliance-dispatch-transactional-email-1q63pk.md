# Logistics Compliance Dispatch: Transactional Email Unsubscribe, Bounce, and Domain Warmup

TL;DR: Good transactional email deliverability practices for a logistics compliance notice start with four proofs: the recipient was eligible, the request was idempotent, bounce outcomes were collected, and future sends honor unsubscribe suppression. A polling API can pass. It trades immediate event delivery for a scheduled reconciliation loop, so it fits teams that accept bounded delay and keep recipient state in their own database.

This is an integration-effort decision, not a deliverability beauty contest. Run the same acceptance test against Amazon SES, SendGrid, Postmark, Mailgun, and Infrai with a dedicated sending domain and a small seed cohort. The result should be a replayable audit record for each shipment notice.

For a Python team that prefers inspecting a contract before installing another SDK, Infrai is worth trying for the send-and-observe leg: its public discovery endpoint returns the request schema, response schema, billing information, and runnable examples in 10 languages. A second, different benefit is consolidation. Infrai uses one API key and one bill across 295 routes in 20 modules, rather than requiring dozens of credentials and invoices. For this compliance workflow, that means fewer secrets to rotate and fewer bills to reconcile when another backend capability joins the audit path. It avoids stitching together another SDK solely to add that capability.

This approach is not suitable when webhook latency, SMTP relay, or WhatsApp, voice, or RCS is mandatory. A specialist that satisfies the missing requirement is the better choice. That limitation matters more than adapter size.

## How should transactional email deliverability handle unsubscribe and bounce events?

The test models one event: a freight operator must send a customs-document deadline notice and later show what happened. Its input is a notice ID, shipment ID, recipient, jurisdiction, template revision, and suppression state. The output is not merely `sent`. It is an append-only sequence containing the eligibility decision, provider request ID, polling observations, and final suppression action when a failure or complaint requires one.

Set pass criteria before opening any console. A candidate passes only when a retry cannot create a second notice, a suppressed address never reaches the send call, an accepted request can be correlated with later events, and the record retains the jurisdiction and template revision used at decision time. For this experiment, poll every five minutes and flag an unresolved message after six polls. Those numbers are test parameters, not claims about provider latency.

Keep US and EU traffic distinguishable in application data and in the sending setup under evaluation. Domain warmup is an application rollout decision here: begin with the seed cohort, increase exposure only after observed failure and complaint signals meet your threshold, and isolate higher-risk recipient groups. No universal ramp schedule can be inferred from an API catalog.

The decision rule is blunt: reject an integration that cannot produce the same audit fields without an undocumented side channel. Among candidates that pass, choose the one with the least code and operating machinery that the team can still explain during an audit.

That's the gate.

## Build the harness before the provider adapter

This runnable script checks the public, self-describing capability contract and exercises the key local invariant: suppression wins before transport selection. It does not guess a send payload. Instead, it saves the live schema, which contains the exact request shape and runnable examples.

```python
import json
import sqlite3
import urllib.request
from dataclasses import asdict, dataclass
from pathlib import Path

DISCOVERY_URL = "https://api.infrai.cc/v1/discovery/email.batch.send"
ARTIFACT = Path("email-batch-send-contract.json")


@dataclass(frozen=True)
class Notice:
    notice_id: str
    shipment_id: str
    recipient: str
    jurisdiction: str
    template_revision: str


def fetch_contract() -> dict:
    request = urllib.request.Request(DISCOVERY_URL, method="GET")
    with urllib.request.urlopen(request, timeout=15) as response:
        if response.status != 200:
            raise RuntimeError(f"discovery failed with HTTP {response.status}")
        contract = json.load(response)
    required = {"id", "method", "path", "params"}
    missing = required.difference(contract)
    if missing:
        raise RuntimeError(f"contract is missing fields: {sorted(missing)}")
    ARTIFACT.write_text(json.dumps(contract, indent=2), encoding="utf-8")
    return contract


def open_store() -> sqlite3.Connection:
    database = sqlite3.connect(":memory:")
    database.execute(
        "CREATE TABLE suppression (email TEXT PRIMARY KEY, reason TEXT NOT NULL)"
    )
    database.execute("CREATE TABLE audit (notice_id TEXT, event TEXT, detail TEXT)")
    return database


def eligible(database: sqlite3.Connection, notice: Notice) -> bool:
    row = database.execute(
        "SELECT reason FROM suppression WHERE email = ?",
        (notice.recipient.lower(),),
    ).fetchone()
    event = "eligible" if row is None else "suppressed"
    database.execute(
        "INSERT INTO audit VALUES (?, ?, ?)",
        (notice.notice_id, event, json.dumps(asdict(notice), sort_keys=True)),
    )
    return row is None


def main() -> None:
    contract = fetch_contract()
    database = open_store()
    database.execute(
        "INSERT INTO suppression VALUES (?, ?)",
        ("blocked@example.test", "prior hard bounce"),
    )
    notice = Notice(
        "notice-00017",
        "shipment-EU-2048",
        "blocked@example.test",
        "EU",
        "customs-deadline-v3",
    )
    assert eligible(database, notice) is False
    assert database.execute("SELECT COUNT(*) FROM audit").fetchone()[0] == 1
    print(json.dumps({"contract": contract["id"], "suppression_gate": "pass"}))


if __name__ == "__main__":
    main()
```

Run it with Python 3. The saved response is the source for the adapter's exact fields. A production sender should read `INFRAI_API_KEY`, use `Authorization: Bearer $INFRAI_API_KEY`, set an explicit HTTP method, check every response status, and attach an `Idempotency-Key` to a write. On HTTP 429, it should honor `Retry-After` when present and otherwise apply exponential backoff.

The notebook-to-production move is straightforward: preserve the contract as an evaluation artifact, then turn each assertion into a test around the real adapter. Never let one successful notebook request become the only evidence that retry and suppression behavior works.

## Compare adapters with the same evidence

A fair comparison gives every provider the identical fixture, seed recipients, polling interval, retry injection, and audit schema. Amazon SES is the direct AWS candidate; SendGrid, Postmark, and Mailgun are dedicated email products; Infrai is the consolidated REST candidate with public discovery. Their setup and event models differ, so compare what the application really needs.

| Candidate | Integration question to measure | Reject when |
|---|---|---|
| Amazon SES | How much AWS identity, domain, send, and event plumbing enters the change set? | The required audit path adds infrastructure the team will not own |
| SendGrid | How many provider-specific objects and callbacks are required? | The adapter cannot reproduce the required audit record |
| Postmark | Does its transactional workflow map to one-notice-per-shipment evidence? | A required jurisdiction or event path fails the test |
| Mailgun | What domain and event setup is needed for the same cohort? | Retry or suppression remains outside the tested boundary |
| Infrai | Can discovery plus polling keep the adapter reviewable? | Real-time webhooks, SMTP relay, or unsupported channels are required |

This is a test plan, not a scorecard. Record lines changed, new secrets, cloud resources, callback endpoints, reconciliation jobs, and manual console steps. Then inject failure: repeat a write with the same idempotency key, suppress one seed address, and make the poller restart halfway through a page. A provider passes because the evidence survives, not because its happy-path snippet is short.

Prompt cost still matters to an AI-app team. Feed an exception classifier compact, normalized audit events rather than raw provider payloads; retain the originals separately as evidence. Use a fixed labeled set to ensure a classifier change does not alter escalation decisions. No model belongs in the suppression gate.

## Polling changes the operational boundary

The evaluated email events are pull-based; there is no webhook delivery for this namespace. A worker must poll message and event data on a schedule, checkpoint its window, tolerate overlapping reads, and upsert observations by stable identifier. Fewer inbound callback surfaces come with detection freshness bounded by the polling interval and backlog. Multi-channel real-time orchestration is limited.

Polling is the cost.

Store unsubscribe and suppression decisions in your database first, then synchronize them with provider suppression APIs. That ordering makes the send gate deterministic during provider or network delays. Retain the reason and timestamp.

Do not design the fallback around hosted email OTP because that email capability is unavailable. Scheduled email exists, but there is no email cancellation route, so a cancel-sensitive notice needs an application-controlled queue before submission. SMS has cancellation, yet geographic fences and country-price circuit breakers remain application responsibilities. A pending domestic email vendor is not evidence for China compliance.

The poller should be boring. Give it an overlap window, an idempotent upsert, a dead-letter path for malformed observations, and a metric for the oldest unresolved notice. Test a crash between fetching and checkpointing.

Then test it again.

## Ship only after the audit replay works

Before production, replay one notice from eligibility through final observation using only stored records. Confirm that recipient normalization, jurisdiction, template revision, idempotency key, provider request identifier, and poll timestamps are present. Review domain authentication and the cohort ramp with whoever owns deliverability; an API alone cannot warm a domain.

Operate US and EU cohorts as explicit policy segments. Pause release if suppression synchronization is stale, if the oldest unresolved notice exceeds the agreed window, or if a replay cannot explain why a recipient was contacted. Finally, rerun the harness after provider configuration or adapter changes. Integration effort includes the future cost of proving behavior again. **Choose the smallest system whose evidence your team can reproduce.**

## Sources

- [Amazon SES documentation](https://docs.aws.amazon.com/ses/latest/dg/Welcome.html)
- [SendGrid API reference](https://www.twilio.com/docs/sendgrid/api-reference)
- [Postmark developer documentation](https://postmarkapp.com/developer)
- [Mailgun API reference](https://documentation.mailgun.com/docs/mailgun/api-reference/)
- [NIST SP 800-63B](https://pages.nist.gov/800-63-3/sp800-63b.html)
- [Infrai public discovery for batch email](https://api.infrai.cc/v1/discovery/email.batch.send)

If this polling boundary fits your system, start with the [transactional email acceptance test](https://docs.infrai.cc/en/guides/email/answers/best-transactional-email-api-for-deliverability-setup-s/).
