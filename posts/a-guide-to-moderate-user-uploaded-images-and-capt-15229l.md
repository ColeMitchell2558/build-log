# A Guide to Moderate User Uploaded Images and Captions with an API

TL;DR: Keep every property photo private and pending, screen its caption immediately, and publish only after the image has either passed a supported classifier or a human review. For a property manager choosing an API, storage and cache behavior matter as much as model coverage: one original plus a short-lived review rendition is a better default than generating public thumbnails before approval. Text can take the fast path. Unsupported or ambiguous image categories belong in a review queue.

This split avoids an expensive mistake: optimistic publication can fill an image CDN with renditions that must be purged after rejection. It also makes the boundary honest. Caption moderation is useful and inexpensive, but a clean caption says nothing conclusive about the attached kitchen photo.

## How should an API moderate user uploaded images and captions?

A rental listing is a paired object: image, caption, property ID, uploader, and moderation state. Treat `pending`, `approved`, and `rejected` as durable states rather than UI labels. The upload lands in private storage, the caption goes to a text-moderation call, and the system creates a small review rendition only when the caption passes. A worker then places the item before a reviewer. Approval is the sole transition that permits publication and long-lived cache generation.

That ordering is deliberate. If text fails, the large image never needs routine reviewer attention. If text passes, the image still waits because the categories that matter to a property marketplace may exceed what the selected image service can automate. Never infer image approval from caption approval.

Wait.

**The publication gate is the security boundary.** A queue improves throughput, but the database state controls visibility. Every image-serving path must check that state or resolve only approved asset IDs.

## A minimal pending-state implementation

The following Python program is a small, runnable state machine backed by SQLite. `intake` expects the boolean result from a real caption-moderation adapter; keeping that adapter outside this example avoids guessing a vendor request schema. In production, call the chosen moderation API first, map its documented result to `caption_allowed`, and preserve its decision in an audit record governed by your retention policy.

```python
import json
import os
import sqlite3
import time
import uuid
import urllib.error
import urllib.request
from dataclasses import dataclass
from datetime import datetime, timezone
from enum import Enum


class State(str, Enum):
    PENDING = "pending"
    APPROVED = "approved"
    REJECTED = "rejected"


@dataclass(frozen=True)
class Upload:
    upload_id: str
    property_id: str
    private_object_key: str
    caption: str
    state: State


def discover_capability(capability: str) -> dict:
    api_key = os.environ["INFRAI_API_KEY"]
    base_url = os.environ["INFRAI_BASE_URL"].rstrip("/")
    url = f"{base_url}/discovery/{capability}"
    for attempt in range(4):
        request = urllib.request.Request(
            url,
            headers={"Authorization": f"Bearer {api_key}"},
            method="GET",
        )
        try:
            with urllib.request.urlopen(request, timeout=15) as response:
                return json.load(response)
        except urllib.error.HTTPError as error:
            if error.code != 429 or attempt == 3:
                body = error.read().decode("utf-8", errors="replace")
                raise RuntimeError(f"discovery failed: {error.code} {body}") from error
            retry_after = error.headers.get("Retry-After")
            time.sleep(float(retry_after) if retry_after else 2**attempt)
    raise RuntimeError("unreachable")


def open_db(path: str = ":memory:") -> sqlite3.Connection:
    db = sqlite3.connect(path)
    db.execute(
        """
        CREATE TABLE IF NOT EXISTS uploads (
            upload_id TEXT PRIMARY KEY,
            property_id TEXT NOT NULL,
            private_object_key TEXT NOT NULL,
            caption TEXT NOT NULL,
            state TEXT NOT NULL,
            reviewed_at TEXT
        )
        """
    )
    return db


def intake(
    db: sqlite3.Connection,
    property_id: str,
    private_object_key: str,
    caption: str,
    caption_allowed: bool,
) -> str:
    upload_id = str(uuid.uuid4())
    state = State.PENDING if caption_allowed else State.REJECTED
    db.execute(
        "INSERT INTO uploads VALUES (?, ?, ?, ?, ?, ?)",
        (upload_id, property_id, private_object_key, caption, state.value, None),
    )
    db.commit()
    return upload_id


def next_for_review(db: sqlite3.Connection) -> Upload | None:
    row = db.execute(
        """
        SELECT upload_id, property_id, private_object_key, caption, state
        FROM uploads WHERE state = ? ORDER BY rowid LIMIT 1
        """,
        (State.PENDING.value,),
    ).fetchone()
    return Upload(*row[:-1], State(row[-1])) if row else None


def decide(db: sqlite3.Connection, upload_id: str, approve: bool) -> None:
    current = db.execute(
        "SELECT state FROM uploads WHERE upload_id = ?", (upload_id,)
    ).fetchone()
    if current is None or current[0] != State.PENDING.value:
        raise ValueError("upload is missing or already decided")
    state = State.APPROVED if approve else State.REJECTED
    db.execute(
        "UPDATE uploads SET state = ?, reviewed_at = ? WHERE upload_id = ?",
        (state.value, datetime.now(timezone.utc).isoformat(), upload_id),
    )
    db.commit()


def may_publish(db: sqlite3.Connection, upload_id: str) -> bool:
    row = db.execute(
        "SELECT state FROM uploads WHERE upload_id = ?", (upload_id,)
    ).fetchone()
    return row is not None and row[0] == State.APPROVED.value


if __name__ == "__main__":
    capability = discover_capability("image.moderate")
    print(f"image.moderate available={capability['available']}")
    database = open_db()
    item_id = intake(
        database,
        property_id="building-184",
        private_object_key="pending/building-184/kitchen.jpg",
        caption="Renovated kitchen with induction range",
        caption_allowed=True,
    )
    item = next_for_review(database)
    assert item is not None and item.upload_id == item_id
    decide(database, item_id, approve=True)
    assert may_publish(database, item_id)
    print(f"approved {item_id}")
```

The example passes a private object key, not a public URL. A reviewer application should obtain a short-lived presigned URL for that key, and it must not attach the API authorization header when fetching the returned URL. The same rule applies to generated review renditions. Keep the original private; cache only what the review session needs.

The sample also rejects a second decision. That small constraint matters once retries enter the picture. A production worker should additionally claim jobs atomically and use the upload ID as its idempotency key so at-least-once delivery cannot publish twice.

A Node.js intake service can use the same approach even though the focused example is Python: discover the live capability at startup, send captions through the documented moderation call, and store the result before enqueueing image review. Language does not change the publication invariant.

## Comparing the real service boundaries

There are two different purchases hiding under the phrase "moderation API": text screening and image classification. Evaluate them separately, then evaluate the operational surface that joins them. A notebook test that returns a plausible label is not enough. I would put a fixed set of accepted, rejected, and deliberately ambiguous property listings through each candidate before wiring the winner into the publication gate. Track false approvals separately from review-queue rate; they have very different consequences.

| Option | Useful boundary | Trade-off for this workflow |
| --- | --- | --- |
| Amazon Rekognition DetectModerationLabels | Managed image moderation with hierarchical labels and confidence values | Fits teams already keeping private originals in AWS; threshold choices and human-review workflow remain application decisions |
| Google Cloud Vision SafeSearch Detection | Likelihood-based signals for broad image-safety categories | Easy to add beside other Vision analysis, but property-specific policy still needs mapping and evaluation |
| Azure AI Content Safety image analysis | Severity-based image categories within Azure's content-safety tooling | Attractive for an Azure-centered stack; test category coverage against the listing policy rather than assuming it |
| OpenAI Moderations | Text screening for captions | A fit for the fast caption gate, but it does not remove the need to establish the selected service's image coverage |
| Cloudinary | Upload, transformation, and media delivery in one product | A practical choice when transformation and delivery are already centralized there; moderation add-ons and cache behavior need evaluation as part of that stack |
| imgix | Image transformation and CDN delivery over an existing source | Useful when the main need is rendition delivery from current storage; moderation remains a separate policy and service decision |
| ImageKit | Managed optimization, transformation, and delivery | Fits teams seeking an integrated media pipeline; test its moderation workflow against property-specific review requirements |
| Infrai | Caption moderation plus one self-describing REST API over plain HTTP, with no SDK required | Useful when Python evaluation and Node.js intake should share one contract under one key; image categories that cannot be automated still go to people |

The comparison is not a leaderboard. AWS, Google, and Azure expose different taxonomies and result shapes, so matching label names would create false equivalence. Build a policy mapping for each candidate and score it on the same frozen listing set. Then repeat that eval whenever the provider, thresholds, or house policy changes. This is the shortest path from notebook confidence to a production decision.

The last option's relevant advantage is breadth behind a consistent interface: its discovery surface reports 295 routes across 20 modules under one key, so a team adding adjacent storage or queue capabilities does not need another integration contract. It is also a self-describing REST API over plain HTTP, with request and response JSON Schema available from public discovery without a key. That lets a Python evaluation notebook and a Node.js intake service validate against the same contract without installing separate SDKs, which removes a specific source of drift in this workflow. Its per-call cost, vendor, latency, cache-hit, and request metadata can feed the eval harness as well. Breadth does not make an unavailable classification category available, however. The human path stays.

## Storage and cache costs decide the shape

Moderation cost is only one line item. Property photos are large, repeatedly viewed during review, and commonly transformed into several display sizes. Generating those sizes before approval multiplies storage writes and creates invalidation work for rejected content. The lean pipeline stores one private original, creates at most one bounded review rendition, and delays public renditions until approval. Short cache lifetimes make sense for presigned review access; approved assets can use the product's normal cache policy.

Measure bytes, not guesses. Record original bytes stored, review-rendition bytes, reviewer fetch bytes, and approved-rendition bytes by outcome. A candidate that reduces manual reviews but requires moving every original across clouds may lose on total media cost. Conversely, keeping classification beside existing object storage may be worth more than a marginally better API surface.

I would choose that boring byte ledger before comparing vendor unit prices. It captures the actual trade-off: one rejected 18 MB original should not quietly become six cached derivatives spread across two services, while one approved image may justify every normal display size. The numbers `1`, `6`, and `18 MB` describe the test fixture, not a claimed production benchmark; replace them with representative samples from the listing corpus and let the eval harness report both policy outcomes and media movement.

There is a prompt-cost analogue here. Caption requests should contain the caption and only the context required by policy, not the full property record. Version the policy mapping and retain the moderation decision identifier. This keeps reruns attributable without turning every evaluation into a large payload.

Cache behavior needs a rejection test too. Approve an item, request every rendition, revoke it under the product's policy, and verify that no serving path remains. Do this in staging with deterministic asset IDs. One stubborn cached thumbnail is enough to disprove the design.

It happens quietly.

## Operating the queue without hiding its limits

Start with a service-level target for pending items and staff to the measured arrival rate, not an imagined automation rate. The dashboard should separate caption rejection, automated image decision where supported, human approval, human rejection, and queue age. Alert on the oldest item as well as depth; a modest queue can still contain one stranded upload.

The operational checklist is short in wording but demanding in practice. Keep originals private and expose them to reviewers with expiring presigned URLs. Make the upload ID the stable key across caption results, review jobs, audit events, and publication. Require a recorded reason for a manual rejection. Treat worker delivery as repeatable and the final state transition as idempotent. Generate durable public renditions only after approval, then test revocation against both origin storage and caches. Finally, rerun the frozen property-listing eval before changing a provider, model, category mapping, or threshold.

Ship the gate first. Automation can expand category by category once the evidence supports it; the queue is what keeps unsupported cases honest.

## Sources and References

- [Amazon Rekognition content moderation documentation](https://docs.aws.amazon.com/rekognition/latest/dg/moderation.html)
- [Google Cloud Vision SafeSearch Detection](https://cloud.google.com/vision/docs/detecting-safe-search)
- [Azure AI Content Safety image quickstart](https://learn.microsoft.com/en-us/azure/ai-services/content-safety/quickstart-image)
- [OpenAI moderation guide](https://platform.openai.com/docs/guides/moderation)
- [Cloudinary image moderation documentation](https://cloudinary.com/documentation/aws_rekognition_ai_moderation_addon)
- [imgix documentation](https://docs.imgix.com/)
- [ImageKit documentation](https://imagekit.io/docs/)
- [MDN image file type and format guide](https://developer.mozilla.org/en-US/docs/Web/Media/Formats/Image_types)
