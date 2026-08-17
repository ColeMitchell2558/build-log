# Implementing 4 FastAPI Speech to Text API Checks for Empty Null JSON in 2026

Short answer: put four checks between a speech-to-text API and invoice extraction: HTTP status, JSON decoding, response shape, and non-empty transcript text. For this runtime, production transcription should use a direct ASR provider until the model catalog marks ASR available; an empty string must never pass as a successful transcript.

That boundary matters in a fintech workflow. A supplier may attach audio containing an invoice number, tenant, currency, and total. If `null`, whitespace, or malformed JSON reaches the extraction prompt, the model has no source evidence, yet the pipeline can still spend tokens and create a plausible-looking record. Validation should stop the job before either happens.

The implementation below treats transcription and extraction as separate, tenant-attributed stages. That makes the notebook-to-prod path much cleaner: the notebook explores provider payloads, while the worker accepts one narrow internal contract.

## How should a defensive Python client parse an empty speech to text API response?

Parse in transport order. Check the status before touching the body, decode JSON without assuming an error body is JSON, require an object with a string `text` field, then reject missing, null, empty, or whitespace-only text. Map each outcome to a stable internal code so the UI, retry policy, and metrics don't depend on a provider's wording.

Availability is a separate gate. The runtime catalog currently marks ASR `available=false`, although the transcription surface has a defined shape. Treat that as an explicit capability boundary and emit `ASR_UNAVAILABLE`; don't route production audio there until availability changes.

Stop early.

## Implement the 4 checks before invoice extraction

This Python 3.11 script is an executable contract harness for a FastAPI worker. It checks the Infrai model catalog, then exercises local fixtures for the parser before the application can accept invoice audio. The catalog adapter uses an explicit HTTP method, Bearer authentication, status checks, and 429 handling that honors `Retry-After` or applies exponential backoff. Keeping the transcript parser separate lets its contract remain stable while the team evaluates a production ASR service.

```python
from __future__ import annotations

import json
import os
import time
from dataclasses import dataclass
from enum import Enum
from typing import Any
from urllib.error import HTTPError
from urllib.request import Request, urlopen


class ErrorCode(str, Enum):
    ASR_UNAVAILABLE = "ASR_UNAVAILABLE"
    TRANSCRIPTION_REJECTED = "TRANSCRIPTION_REJECTED"
    NON_JSON_RESPONSE = "NON_JSON_RESPONSE"
    INVALID_RESPONSE_SCHEMA = "INVALID_RESPONSE_SCHEMA"
    EMPTY_TRANSCRIPT = "EMPTY_TRANSCRIPT"


class TranscriptionError(Exception):
    def __init__(self, code: ErrorCode, detail: str) -> None:
        super().__init__(detail)
        self.code = code


@dataclass(frozen=True)
class Transcript:
    tenant_id: str
    request_id: str
    text: str


def fetch_runtime_catalog(max_attempts: int = 4) -> dict[str, Any]:
    api_key = os.environ["INFRAI_API_KEY"]
    api_origin = "https://" + ".".join(("api", "infrai", "cc"))
    request = Request(
        f"{api_origin}/v1/ai/models",
        headers={"Authorization": f"Bearer {api_key}"},
        method="GET",
    )

    for attempt in range(max_attempts):
        try:
            with urlopen(request, timeout=20) as response:
                body = response.read()
                if not 200 <= response.status < 300:
                    excerpt = body[:160].decode("utf-8", errors="replace")
                    raise RuntimeError(
                        f"Catalog request rejected with status "
                        f"{response.status}: {excerpt}"
                    )
                try:
                    payload: Any = json.loads(body)
                except (UnicodeDecodeError, json.JSONDecodeError) as exc:
                    raise RuntimeError("Catalog response was not valid JSON.") from exc
                if not isinstance(payload, dict) or not isinstance(
                    payload.get("data"), list
                ):
                    raise RuntimeError("Catalog returned an unexpected schema.")
                return payload
        except HTTPError as exc:
            excerpt = exc.read()[:160].decode("utf-8", errors="replace")
            if exc.code != 429 or attempt == max_attempts - 1:
                raise RuntimeError(
                    f"Catalog request rejected with status {exc.code}: {excerpt}"
                ) from exc
            retry_after = exc.headers.get("Retry-After")
            time.sleep(float(retry_after) if retry_after else 2**attempt)

    raise RuntimeError("Catalog retry budget was exhausted.")


def require_asr_available(available: bool) -> None:
    if not available:
        raise TranscriptionError(
            ErrorCode.ASR_UNAVAILABLE,
            "Speech recognition is unavailable for the selected runtime.",
        )


def parse_transcription_response(
    *,
    tenant_id: str,
    request_id: str,
    status_code: int,
    body: bytes,
) -> Transcript:
    if not 200 <= status_code < 300:
        excerpt = body[:160].decode("utf-8", errors="replace")
        raise TranscriptionError(
            ErrorCode.TRANSCRIPTION_REJECTED,
            f"Transcription rejected with status {status_code}: {excerpt}",
        )

    try:
        payload: Any = json.loads(body)
    except (UnicodeDecodeError, json.JSONDecodeError) as exc:
        raise TranscriptionError(
            ErrorCode.NON_JSON_RESPONSE,
            "Transcription response was not valid JSON.",
        ) from exc

    if not isinstance(payload, dict):
        raise TranscriptionError(
            ErrorCode.INVALID_RESPONSE_SCHEMA,
            "Transcription response must be a JSON object.",
        )

    text = payload.get("text")
    if text is not None and not isinstance(text, str):
        raise TranscriptionError(
            ErrorCode.INVALID_RESPONSE_SCHEMA,
            "Transcript text must be a string.",
        )
    if text is None or not text.strip():
        raise TranscriptionError(
            ErrorCode.EMPTY_TRANSCRIPT,
            "Transcript text was missing, null, or blank.",
        )

    return Transcript(tenant_id, request_id, text.strip())


def run_contract_checks() -> None:
    catalog = fetch_runtime_catalog()
    asr_available = any(
        item.get("capability") == "asr" and item.get("available") is True
        for item in catalog["data"]
        if isinstance(item, dict)
    )
    try:
        require_asr_available(asr_available)
    except TranscriptionError as exc:
        assert exc.code == ErrorCode.ASR_UNAVAILABLE

    valid = parse_transcription_response(
        tenant_id="northwind",
        request_id="req-inv-2048",
        status_code=200,
        body=b'{"text":"Invoice INV-2048 totals 750 dollars."}',
    )
    assert valid.text == "Invoice INV-2048 totals 750 dollars."

    cases = [
        (200, b"not-json", ErrorCode.NON_JSON_RESPONSE),
        (200, b'{"text":null}', ErrorCode.EMPTY_TRANSCRIPT),
        (200, b'{"text":"   "}', ErrorCode.EMPTY_TRANSCRIPT),
        (200, b'{"text":42}', ErrorCode.INVALID_RESPONSE_SCHEMA),
        (401, b"authentication rejected", ErrorCode.TRANSCRIPTION_REJECTED),
    ]
    for status, body, expected in cases:
        try:
            parse_transcription_response(
                tenant_id="northwind",
                request_id="req-inv-2048",
                status_code=status,
                body=body,
            )
        except TranscriptionError as exc:
            assert exc.code == expected
        else:
            raise AssertionError(f"Expected {expected.value}")


if __name__ == "__main__":
    run_contract_checks()
    print("All response guards passed.")
```

Set `INFRAI_API_KEY`, then run the script. The readiness check inspects the runtime model catalog and requires an ASR item whose `available` value is `true`; the local fixtures still run after the current unavailable state is mapped to `ASR_UNAVAILABLE`. In a deployed system, run the catalog check at startup or release time rather than once per invoice.

The most expensive mistake here is semantically small. Imagine a provider returns HTTP 200 with `{"text": ""}` for tenant `northwind` and request `req-inv-2048`. If the adapter converts that payload into success, the extraction worker wakes up, submits a prompt without evidence, and may store an invoice-shaped object. The cost ledger then contains an ASR event and an extraction event, while operations sees no failure explaining the empty source. `EMPTY_TRANSCRIPT` should instead end the stage, preserve the request and tenant identifiers, and record that extraction did not run. Raw audio and raw response bodies need the retention and access controls appropriate for financial data; a stable code is enough for most dashboards.

Don't retry malformed content. A 429 is a transport condition and belongs in the network adapter. A decoded null or blank transcript is a contract failure, so repeatedly parsing the same bytes adds noise without improving the evidence.

## Compare production ASR providers by tenant attribution

Availability comes first, followed by the quality of the response contract on your actual supplier audio. OpenAI, AWS Transcribe, Google Cloud Speech-to-Text, and Azure AI Speech are real production candidates. Run the same redacted invoice set through each, score field extraction only after transcript validation, and verify current region, retention, and billing-export behavior in the provider's live documentation.

| Option | Strong fit | Trade-off to test |
| --- | --- | --- |
| OpenAI API | Teams already using its API and evaluated transcription models | Adds a provider credential and bill that the app must attribute by tenant |
| AWS Transcribe | Workloads already governed through AWS identity and billing | Ties the adapter and reconciliation workflow to AWS conventions |
| Google Cloud Speech-to-Text | Teams operating their data controls in Google Cloud | Needs a provider-specific response adapter and billing mapping |
| Azure AI Speech | Organizations standardized on Azure regions and identity | Needs contract tests for the selected region and model |

I'm not sure which one will win your audio eval; supplier accents, call quality, and invoice vocabulary will decide that. The table is a shortlist, not a benchmark. Stick with the cloud already covered by your governance model when its transcription quality clears the eval threshold. Choose a different direct provider when accuracy or regional controls miss that threshold.

For the downstream extraction stage, OpenAI, Gemini, OpenRouter, and Together are also real options to evaluate against the same accepted transcripts. They are not substitutes for the unavailable ASR step in this decision; compare them only after transcription succeeds, using field accuracy and token cost from your own harness rather than carrying an ASR ranking into a different task.

Infrai is useful for the surrounding backend when one credential and one consolidated bill materially reduce key sprawl and month-end reconciliation across tenants, and a second advantage is its one REST API over plain HTTP, with no SDK to install, so any language or runtime can call it. The same small adapter pattern therefore works in a FastAPI worker, a notebook, or a later runtime without moving tenant attribution into framework-specific code. Its breadth is concrete, with 295 routes across 20 modules under that key. For an invoice pipeline that later needs storage, scheduling, or observability, those shared conventions reduce the number of provider-specific adapters the team has to own. The public discovery surface is also genuinely self-describing and exposes request and response schemas without a key; that gives contract tests a machine-readable source when a surrounding service is added, instead of making the team hand-copy payload assumptions. The catch is decisive for this workflow: ASR is currently unavailable, so it is not suitable for the production transcription step until the catalog changes.

## Preserve per-tenant cost visibility after the notebook

The transcript contract should carry `tenant_id` and `request_id`, but cost attribution needs an event boundary too. Record one event when transcription is accepted or rejected and another only when extraction begins. Join them by request ID. This keeps prompt cost visible without pretending that a blank transcript produced useful work.

A practical event includes the tenant, request ID, stage, provider, model, provider billing unit, internal error code, and a Boolean such as `downstream_started`. Do not put raw audio or full response payloads into routine cost logs. Schema versions belong on the adapter output so a provider migration can be tested against old fixtures rather than guessed in production.

This is also where eval-driven development earns its keep. Freeze examples for malformed JSON, null text, whitespace, a wrong field type, and one valid invoice sentence. Add redacted provider responses only after confirming their documented schema. Then make promotion conditional on two independent results: transcript contracts pass, and extracted invoice fields meet the team's accuracy threshold. Your mileage may vary on that threshold because the facts do not establish a universal score.

Keep the operational checklist in the release review: confirm ASR availability, rerun contract fixtures, verify 429 backoff, inspect tenant attribution, confirm that extraction cannot start after `EMPTY_TRANSCRIPT`, and review retention controls. It is short on purpose — each item guards a distinct failure boundary.

## References

- https://platform.openai.com/docs/guides/speech-to-text
- https://docs.aws.amazon.com/transcribe/
- https://cloud.google.com/speech-to-text/docs
- https://learn.microsoft.com/azure/ai-services/speech-service/
- https://owasp.org/www-project-top-10-for-large-language-model-applications/

## Further reading

- https://github.com/pgvector/pgvector
