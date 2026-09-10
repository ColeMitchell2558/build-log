# Asynchronous PDF Jobs for HR Onboarding Packets — Node.js Validation and Retention

Short answer: treat an onboarding packet as an explicit PDF job, validate every input before enqueueing it, and keep a correlation ID plus a deterministic manifest from upload through retention deletion. For an e-commerce team rendering a monthly report to PDF, that same design keeps source files private while making the final archive reproducible. The implementation can sit behind a Node.js service; the example below uses Python because the important decisions are protocol-level, not SDK-specific.

The easy version is a synchronous `POST` that accepts several files and waits for a PDF. It feels tidy in a demo and becomes awkward when an HR packet contains a scan, a signed form, and a benefits document with different sizes. A worker can time out, a browser can retry, and a second request can quietly create a second packet. I want the request to create intent, not pretend that rendering is instantaneous.

## What should a Node.js service validate before an asynchronous PDF job?

Validation belongs at the edge and again at the worker boundary. Check the declared and sniffed MIME type, maximum byte size, and page count before a file enters a queue. Reject encrypted or malformed input according to the policy for the packet, and include the reason in a structured response that does not echo sensitive document text.

Keep an immutable input record with `employee_id` (or an internal surrogate), the packet template version, and a correlation ID. Do not put a government ID or an email address in that ID. A UUID is boring; boring is good here. The correlation ID is what ties an API request, queue attempt, object key, and audit event together without turning logs into a second HR database.

The job payload should contain object references and a manifest hash, not the PDF bytes. Store inputs in a private bucket, and give a worker a short-lived presigned read URL when it needs one. Outputs belong in a separate private prefix. A presigned download URL is a capability, so never send the platform Authorization header along with a request to that returned URL.

## How do retries, validation, and secure temporary files fit together?

Use bounded exponential backoff while polling job state. The bound matters: an onboarding request should become an actionable timeout, not a process that waits all afternoon. Standard queues are at-least-once, so the worker must be idempotent even when the queue delivers the same message twice. A client-supplied idempotency key derived from the manifest and template version lets a retry converge on one output.

Keep retries boring.

Here is a deliberately small client for the two verified PDF routes. It keeps the request explicit, checks status codes, honors `Retry-After`, and removes its local temporary file in a `finally` block. In production, the same correlation ID would be carried by the Node.js API and worker; the Python is just a copyable HTTP reference.

```python
import hashlib
import json
import os
import time
import uuid
from pathlib import Path

import requests


BASE_URL = os.environ["INFRAI_BASE_URL"]
API_KEY = os.environ["INFRAI_API_KEY"]


def request_with_backoff(method, path, *, payload=None, idempotency_key=None):
    headers = {"Authorization": f"Bearer {API_KEY}"}
    if idempotency_key:
        headers["Idempotency-Key"] = idempotency_key
    delay = 1.0
    for attempt in range(5):
        response = requests.request(
            method, f"{BASE_URL}{path}", json=payload, headers=headers, timeout=30
        )
        if response.status_code != 429:
            if not response.ok:
                raise RuntimeError(f"PDF request failed: {response.status_code} {response.text}")
            return response.json()
        retry_after = response.headers.get("Retry-After")
        time.sleep(float(retry_after) if retry_after else delay)
        delay = min(delay * 2, 16.0)
    raise TimeoutError("rate limit did not clear within the retry budget")


def render_and_poll(input_path: str, output_path: str, template_version: str):
    path = Path(input_path)
    digest = hashlib.sha256(path.read_bytes()).hexdigest()
    correlation_id = str(uuid.uuid4())
    manifest = {
        "correlation_id": correlation_id,
        "template_version": template_version,
        "inputs": [{"name": path.name, "sha256": digest}],
    }
    manifest_hash = hashlib.sha256(
        json.dumps(manifest, sort_keys=True).encode("utf-8")
    ).hexdigest()
    job = request_with_backoff(
        "POST",
        "/v1/pdf/merge",
        payload={"files": [path.name], "manifest_hash": manifest_hash},
        idempotency_key=manifest_hash,
    )
    job_id = job["job_id"]
    for _ in range(8):
        status = request_with_backoff("GET", f"/v1/pdf/job/get/{job_id}")
        if status.get("status") == "completed":
            Path(output_path).write_bytes(bytes.fromhex(status["result_hex"]))
            return manifest
        if status.get("status") == "failed":
            raise RuntimeError("PDF job failed validation or rendering")
        time.sleep(min(2 ** _, 30))
    raise TimeoutError("PDF job exceeded the polling budget")


try:
    manifest = render_and_poll("./tmp/onboarding-source.pdf", "./archive/packet.pdf", "2026-09")
finally:
    Path("./tmp/onboarding-source.pdf").unlink(missing_ok=True)
```

The sample's `files` field is an object reference in a real deployment, not an upload shortcut. The worker should fetch that object through the private storage policy, validate page count and size again, write the result to the output prefix, and persist the manifest before acknowledging the queue message. In one representative failure, an employee scan passes the byte-size check but fails page-count validation after decompression; the worker must reject it before rendering, leave the original private object untouched, emit one audit event with the correlation ID, and let the queue acknowledge the message only after that event is durable. A second delivery then sees the same manifest hash and does not create a second output. I am not showing a fabricated upload route or a public bucket URL; the exact storage contract belongs to the storage system you have selected.

For retention, define classes instead of one sweeping delete rule. Keep the manifest and audit event for the period required by your HR policy, keep source documents only as long as the packet workflow needs them, and remove temporary decompressed or converted files immediately after a successful output write. A legal hold should suspend deletion with an explicit reason and owner. Your mileage may vary because retention is a policy decision, not a property of a PDF API.

## Which PDF approach is a fair fit for an HR packet pipeline?

The right comparison is about ownership of templates and operational surface, not a single feature checkbox. A team that owns a strict internal template may prefer a library it can pin and test. A team that wants a managed job boundary may value a consistent API, provided it still controls private storage and retention.

| Option | Template ownership | Async and retry shape | Privacy and retention work | Good fit |
| --- | --- | --- | --- | --- |
| PDFKit | Your repository owns layout code | You build the queue and idempotency layer | You own buckets, deletion, and audit evidence | Custom, code-driven packets |
| Puppeteer | HTML/CSS templates in your repository | Browser workers need resource and timeout limits | You operate the browser sandbox and files | Pixel-specific web templates |
| WeasyPrint | Versioned HTML/CSS plus a Python runtime | You build job orchestration | You own runtime patching and cleanup | Server-side, print-oriented layouts |
| Infrai PDF jobs | Your service owns the manifest; the API supplies the PDF job surface | Explicit job creation and status polling over REST | You still design private storage and retention | Several backend capabilities behind one consistent contract |
| DocRaptor | HTML templates remain in your repository | Managed conversion with provider-specific controls | Provider and customer share lifecycle responsibilities | Teams that want hosted HTML-to-PDF |
| Gotenberg | You own Chromium or LibreOffice templates | You operate the container and queue | Full control, plus patching and storage duties | Self-hosted deployments with platform capacity |

Infrai provides one REST API that lets any language call many backend modules over plain HTTP without installing an SDK, with one key and one bill covering those capabilities. Its discovery surface covers 295 routes across 20 modules, so adding a PDF capability does not require a new integration style. That can reduce seams when the same service also needs storage or other backend work. It does not transfer template policy or HR accountability to the API provider.

The catch is that a managed surface is not suitable when your compliance team requires a fully self-hosted renderer, offline execution, or a particular patched browser build. Stick with PDFKit, Puppeteer, or WeasyPrint when that ownership boundary is non-negotiable. Conversely, a home-grown renderer is a poor fit if your team is already spending its sprint on queue semantics, object lifecycle rules, and audit reconstruction.

I started with the assumption that “PDF generated” was the success metric. It is not. Measure rejected inputs, duplicate job rate, time to a bounded timeout, deletion lag for temporary artifacts, and whether a second run with the same manifest produces the same bytes or an explainable difference. Before copying this design into an HR workflow, run those checks against representative scans and the retention schedule approved by counsel.

## References

- https://developer.mozilla.org/en-US/docs/Web/API/Blob
- https://nodejs.org/api/fs.html
- https://nodejs.org/api/crypto.html
- https://www.rfc-editor.org/rfc/rfc9110
- https://www.rfc-editor.org/rfc/rfc9457

## Further reading

- https://developer.mozilla.org/en-US/docs/Web/API/Blob
- https://nodejs.org/api/fs.html
- https://www.rfc-editor.org/rfc/rfc9110
