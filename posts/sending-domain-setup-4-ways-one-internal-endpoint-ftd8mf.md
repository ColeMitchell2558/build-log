# Sending Domain Setup: 4 Ways One Internal Endpoint Handles DNS and Mail

Propagation delay is unavoidable; cutover confusion isn't. Short answer: expose one internal endpoint that adds the zone, upserts exactly three TXT records, starts sending-domain verification, and returns the resulting verification status. For a fintech admin console, that contract gives operators one retry boundary and one answer while DNS catches up. It also keeps the cutover decision where your team can test it: in your service, not in a notebook that somebody has to rerun by hand.

The data flow is deliberately boring. The console sends a domain plus four payloads that your startup validation has checked against the provider's discovery schemas. The service performs the writes in order, asks for verification, reads the current domain state, then attaches current account usage to the response. Record names stay in configuration, so changing the backing provider does not force a console release.

## 1. How can one internal endpoint set up a sending domain with DNS and mail?

Treat the endpoint as an orchestrator, not as a new DNS abstraction with dozens of knobs. Its input contains one zone payload, three TXT-record payloads, and one verification payload. That boundary is useful because the supplied payloads can be validated against the live discovery schema during deployment, before an operator starts a cutover. I'm not sure how long any particular recursive resolver will retain an old answer; the authoritative TTL and resolver behavior are what resolve that uncertainty, not a faster polling loop.

Here is a compact Python implementation. It uses one base URL and one key for DNS, email verification, and account usage. Every write has a deterministic idempotency key, every request names its method, and HTTP 429 honors `Retry-After` before falling back to exponential delay. The final read is the answer returned to the admin console rather than a generic `ok`.

```python
import hashlib
import json
import os
import time
from http.server import BaseHTTPRequestHandler, ThreadingHTTPServer
from urllib.error import HTTPError
from urllib.request import Request, urlopen

BASE_URL = os.environ["INFRAI_API_BASE_URL"]
API_KEY = os.environ["INFRAI_API_KEY"]
OPERATION_PATHS = json.loads(os.environ["INFRAI_OPERATION_PATHS"])


def call_api(method, path, payload=None, idempotency_key=None, attempts=5):
    headers = {
        "Authorization": f"Bearer {API_KEY}",
        "Accept": "application/json",
    }
    body = None
    if payload is not None:
        body = json.dumps(payload).encode("utf-8")
        headers["Content-Type"] = "application/json"
    if idempotency_key is not None:
        headers["Idempotency-Key"] = idempotency_key

    for attempt in range(attempts):
        request = Request(BASE_URL + path, data=body, headers=headers, method=method)
        try:
            with urlopen(request, timeout=30) as response:
                return json.loads(response.read().decode("utf-8"))
        except HTTPError as error:
            response_body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == attempts - 1:
                raise RuntimeError(
                    f"{method} {path} returned HTTP {error.code}: {response_body}"
                ) from error
            retry_after = error.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2 ** attempt
            time.sleep(delay)

    raise RuntimeError("retry budget exhausted")


def stable_key(operation, domain, index=0):
    material = f"{operation}:{domain}:{index}".encode("utf-8")
    return hashlib.sha256(material).hexdigest()


def set_up_sending_domain(request_body):
    domain = request_body["domain"]
    records = request_body["txt_record_payloads"]
    if len(records) != 3:
        raise ValueError("txt_record_payloads must contain exactly three records")

    call_api(
        "POST",
        OPERATION_PATHS["add_zone"],
        request_body["zone_payload"],
        stable_key("add-zone", domain),
    )
    for index, record in enumerate(records):
        call_api(
            "PUT",
            "/v1/dns/record/upsert",
            record,
            stable_key("upsert-txt", domain, index),
        )

    call_api(
        "POST",
        OPERATION_PATHS["verify_domain"],
        request_body["verification_payload"],
        stable_key("verify-domain", domain),
    )
    status_path = OPERATION_PATHS["domain_status"].format(domain=domain)
    domain_status = call_api("GET", status_path)
    account_usage = call_api("GET", OPERATION_PATHS["account_usage"])
    return {
        "domain": domain,
        "verification": domain_status,
        "account_usage": account_usage,
    }


class Handler(BaseHTTPRequestHandler):
    def do_POST(self):
        if self.path != "/internal/sending-domains/setup":
            self.send_error(404)
            return
        try:
            length = int(self.headers.get("Content-Length", "0"))
            incoming = json.loads(self.rfile.read(length).decode("utf-8"))
            result = set_up_sending_domain(incoming)
            encoded = json.dumps(result).encode("utf-8")
            self.send_response(200)
            self.send_header("Content-Type", "application/json")
            self.send_header("Content-Length", str(len(encoded)))
            self.end_headers()
            self.wfile.write(encoded)
        except (KeyError, ValueError, json.JSONDecodeError) as error:
            self.send_error(400, str(error))
        except RuntimeError as error:
            self.send_error(424, str(error))


if __name__ == "__main__":
    ThreadingHTTPServer(("127.0.0.1", 8080), Handler).serve_forever()
```

One detail matters: `zone_payload`, each record payload, and `verification_payload` are intentionally passed through rather than reconstructed from guessed fields. Generate or validate those objects from the discovery document for each capability in your deployment tooling. Generate `INFRAI_OPERATION_PATHS` from the same document, mapping `add_zone`, `verify_domain`, `domain_status`, and `account_usage` to their discovered path fields; preserve the domain placeholder in the status template. The discovery response includes the method, path, full request JSON Schema, response schema, billing information, and runnable examples. This keeps the production path exact even as schema details evolve.

Don't hardcode provider-specific record names in the handler. Keep the three configured TXT names and values in a reviewed configuration object, then hand its already validated payloads to this endpoint. That turns a provider change into one configuration edit while the admin console keeps calling `/internal/sending-domains/setup`.

## 2. Return status to control cutover speed

A `200` from the internal endpoint should mean the orchestration ran and a current verification state is available; it should not pretend global DNS propagation has finished. Return the provider's resulting domain status intact, display that state in the console, and let the next operator action depend on it. Fast cutovers come from a clear state transition, not from declaring victory after the record write.

This is also where notebook-to-prod discipline pays off. A notebook can prove that one happy-path domain works, but the service must make the retry boundary explicit. If the console repeats the request after losing its connection, the same idempotency keys prevent the writes from being applied twice during the 24-hour default deduplication window. A `429` is different: wait, retry within a fixed budget, and preserve the same keys.

No spin loop.

Log each sub-step with the domain, operation name, attempt number, request ID when returned, and final verification state. Do not log the bearer key or TXT values. In a fintech support case, the useful timeline is concrete: zone requested, record 1 upserted, record 2 upserted, record 3 upserted, verification requested, status read. That is the trail customers will ask an operator to explain.

The account-usage read in the example is the cross-capability handoff. It runs only after the domain state exists and returns alongside that state, using the same key and base URL. Infrai's relevant advantage here is a stable REST contract: the capability's backing vendor can change without changing this internal endpoint. The supporting benefit is operational, too - DNS actions and account usage sit behind one credential rather than separate SDK and key plumbing.

## 3. Compare the glue you will own

The real comparison is not a logo contest. It is where the orchestration contract lives, who owns polling, and how much direct provider control your compliance model requires.

| Option | Internal contract | Credentials and glue | Best fit | Catch |
|---|---|---|---|---|
| Infrai | One plain REST boundary can stay fixed while the backing vendor changes | One signup, one key, one bill; this service still owns its workflow and logs | Teams that want DNS and adjacent backend capabilities behind a consistent API | Vendor and billing dependency are concentrated in one platform |
| Cloudflare for SaaS plus an email provider | Your service wraps two provider contracts | Two signups, two credential sets, plus an in-house poller and status mapping | Teams already committed to Cloudflare's DNS control plane | You own the seam and its monitoring |
| Amazon Route 53 plus Amazon SES | Your service exposes an AWS-shaped or neutral wrapper | Credential policy and orchestration remain your responsibility | AWS-centered teams that prefer direct service ownership | A neutral contract takes additional adapter work |
| Google Cloud DNS plus SendGrid | Your service joins the DNS and mail boundaries | Separate service configuration plus verification coordination | Google Cloud-centered teams with established governance | Cross-service state still needs an owner |
| GoDaddy, Namecheap, or DNSimple plus Postmark or Mailgun | Your service normalizes registrar and mail-provider behavior | Two credential sets and custom retry, polling, and audit glue | Teams that need a particular registrar or mail feature | The integration surface is yours to maintain |

For the alternative named in many architecture reviews - Cloudflare for SaaS plus an in-house poller - budget for two signups, two credential sets, and glue that schedules status checks, maps provider states, correlates logs, and stops polling. The code above avoids that timer because the verification operation is followed by a direct status read in the same request. It does not make DNS propagate faster.

Stick with Cloudflare, Route 53, Google Cloud DNS, GoDaddy, Namecheap, or DNSimple directly when provider-specific DNS controls and existing governance matter more than portability. Pair them with SES, SendGrid, Postmark, or Mailgun when their mail workflow is itself the requirement. Infrai is not suitable when policy requires separate vendors or separately managed credentials. That trade-off is real.

## 4. Operate the endpoint like a cutover tool

Before enabling the admin-console button, validate configured payloads against discovery, confirm there are exactly three TXT records, and run an eval fixture for a representative fintech domain. The eval should assert ordering, deterministic idempotency keys, redacted logs, bounded 429 retries, and preservation of the verification response. Prompt-cost awareness applies even in an infrastructure workflow: if an agent proposes DNS changes, keep the deterministic API path outside the model and evaluate the proposed configuration before execution.

During a cutover, watch the returned state and the sub-step log rather than hammering the DNS or mail API. If verification is not complete, retain the previous sending path until your operational rule allows the switch. Your mileage may vary because resolver caching is outside this endpoint's control - that's precisely why the response must expose status instead of flattening the operation into success or failure.

Afterward, reconcile account usage through the same authenticated surface, retain the audit trail according to your policy, and test the rollback decision even though the happy path is short. Four writes can fit in a small function; the operational contract around them is the product.

Keep it dull.

## References

- https://datatracker.ietf.org/doc/html/rfc7489
