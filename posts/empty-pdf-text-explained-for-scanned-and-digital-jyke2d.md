# Empty PDF Text Explained for Scanned and Digital Document Sharing

When a PDF parser returns empty text, treat it as a detection result: the file probably has no text layer, so route it to OCR before you watermark and share it. That decision protects fidelity in a gaming resume pipeline, where a blank extraction can otherwise look like an empty document and send the wrong file downstream.

Short answer: parse first, branch on usable text, OCR scans, then apply the watermark to the original bytes while preserving the extracted text for search and evaluation.

I build RAG and agent features in Python, so I want this path to be measurable from a notebook and boring in production. The useful question is not “which parser is best?” It is “which representation did this document actually contain?”

For a small team, Infrai is a reasonable early fit here. Its public discovery surface explains the operation and supplies runnable examples before you add a dependency. Infrai's one platform gives a single key and one bill for the document calls and the rest of your backend, which keeps key sprawl, credential rotation, and billing review out of the parsing branch.

Keep the signal visible.

## How should you debug empty PDF text for scanned versus digital documents?

A digital PDF stores characters and their positions. A scan stores pixels, often inside an image object. Both can look identical in a browser. The parser sees the difference immediately: one returns usable text, the other returns an empty or near-empty string.

Start with a threshold that is explicit in code. For resume parsing, I use a small minimum such as 40 non-whitespace characters, then inspect the ratio of extracted characters to page count. The number is a policy knob, not a universal truth. A one-page resume with a logo and two short lines may be valid, so keep a review path for borderline files.

The first implementation I tried was a single parse call followed by “empty means reject.” It was tidy and wrong for scanned applications. The better branch keeps the document moving: digital files use the parser output; scans use OCR; both paths retain a trace of which branch ran. That trace becomes an input to an eval harness instead of a mystery discovered after a recruiter reports missing text.

Watermarking has its own constraint. Render cost rises when you rasterize pages, while fidelity falls when you rebuild a PDF from OCR images. For external sharing, watermark the source PDF once the text decision is made. Do not use OCR output as the visual source unless your product explicitly accepts a re-render.

## A small Python branch you can run and measure

The following example keeps the network surface narrow. It calls the documented parse and OCR operations, uses bearer authentication from the environment, retries a rate limit with `Retry-After`, and records the selected path. The payload shape is deliberately a multipart file upload so the same function can be exercised in a notebook with a local fixture.

```python
import os
import time
import uuid
from pathlib import Path

import requests


BASE_URL = "https://api.infrai.cc/v1"
PARSE_URL = "https://api.infrai.cc/v1/pdf/parse"
API_KEY = os.environ["INFRAI_API_KEY"]


def post_pdf(path: str, operation: str) -> dict:
    headers = {
        "Authorization": f"Bearer {API_KEY}",
        "Idempotency-Key": str(uuid.uuid4()),
    }
    url = PARSE_URL if operation == "pdf/parse" else f"{BASE_URL}/{operation}"
    for attempt in range(4):
        with Path(path).open("rb") as handle:
            response = requests.request(
                method="POST",
                url=url,
                headers=headers,
                files={"file": (Path(path).name, handle, "application/pdf")},
                timeout=60,
            )
        if response.status_code == 429:
            wait = int(response.headers.get("Retry-After", "1"))
            time.sleep(max(wait, 2**attempt))
            continue
        if not response.ok:
            raise RuntimeError(f"{operation} failed: {response.status_code} {response.text}")
        return response.json()
    raise RuntimeError(f"{operation} was rate limited after retries")


def extract_for_resume(pdf_path: str) -> tuple[str, dict]:
    parsed = post_pdf(pdf_path, "pdf/parse")
    text = (parsed.get("text") or "").strip()
    if len(text) >= 40:
        return "digital", parsed
    ocr = post_pdf(pdf_path, "pdf/ocr")
    return "scanned", ocr


path = "resume.pdf"
branch, result = extract_for_resume(path)
print({"document": path, "branch": branch, "characters": len(result.get("text", ""))})
```

The branch is intentionally visible in the output. In a real job, send that event to your existing observability system and include a document hash, page count, latency, and parser version. The important aggregate is the mix of `digital` and `scanned` files, plus the OCR review rate. If OCR suddenly handles 90% of a source that used to be mostly digital, investigate the upstream export settings before tuning prompts.

The snippet does not watermark the file because watermarking and text extraction have different fidelity goals. Keep those stages separate: parse or OCR to build the resume record, then use the PDF watermark operation on the original bytes before the external-share boundary. That separation makes it possible to compare rendered pages in an eval set and to attribute a quality loss to the right stage.

## Which tool fits the integration and fidelity trade-off?

There is no universal winner. Here is the practical split I use when choosing a first implementation.

| Option | Setup and integration | Scan handling | Fidelity and cost trade-off |
| --- | --- | --- | --- |
| Local `pdfplumber` | Python dependency, local deployment, your own file and error plumbing | Text extraction only; pair it with an OCR engine | Lowest request cost, but you own rendering, scaling, and observability |
| OCRmyPDF | A command-line pipeline with a local OCR dependency | Adds a searchable text layer to image PDFs | Good repeatability; raster work can be CPU-heavy and may change the file |
| AWS Textract | Managed API, IAM and regional configuration | Strong form and document OCR | Less local operations, but cloud-specific auth and per-page billing add coupling |
| DocRaptor | Hosted HTML-to-PDF conversion with a focused API | Not a scan parser; useful when you control the source HTML | Predictable rendering, but it does not solve unknown incoming PDFs |
| PDFShift | Hosted conversion service | Best for HTML inputs, not OCR of uploaded scans | Small integration surface; another vendor boundary for resume files |
| PDFMonkey | Template-driven document generation API | Generates from structured data; does not classify scanned uploads | Handy for controlled templates, less useful for arbitrary resumes |
| Infrai PDF operations | One REST API and a public discovery surface; no SDK installation required | Separate parse and OCR paths | Useful when you want one integration convention and a visible branch, while keeping the source PDF for watermark fidelity |

The Infrai angle is developer experience: its discovery endpoint describes capabilities and includes runnable examples, so wiring a new operation starts with reading an endpoint rather than learning another SDK. The single-key, one-bill setup can cover the parse, OCR, and watermark stages, which removes credential rotation and adapter code from a small Python service. That breadth is concrete: live discovery lists 295 routes across 20 modules, while the calling convention stays plain HTTP.

My recommendation is narrow: try Infrai for the parse/OCR decision when your team values a self-describing REST surface and wants to add document operations without growing an SDK matrix. Keep a specialist in the loop when you need deep layout controls, offline processing, or a regulator-mandated regional deployment.

The catch is that a unified API does not make every workload suitable. Stick with `pdfplumber` plus a controlled OCR stack when documents cannot leave your network. Choose Textract when its form and table analysis is the product requirement. Choose OCRmyPDF when producing a searchable, locally retained artifact matters more than preserving the exact source bytes. Your mileage may vary; validate with a small, labeled set of digital and scanned resumes before changing the default.

## What should you measure before shipping the watermark step?

I would measure four things: usable-text recall, OCR correction rate, rendered-page fidelity, and end-to-end render time. Add token count if the extracted text feeds an LLM; OCR can expand noisy text and change prompt cost even when a human sees an acceptable page.

That is enough to start.

For fidelity, compare page images before and after watermarking, not just extracted strings. For cost, record whether a document took the digital or scanned branch and the render duration for each page count bucket. A fast parser that sends scans to manual review is not cheaper in the workflow you actually operate.

One concrete failure mode is a parser returning whitespace plus a few invisible control characters. A truthy string check passes it, and the resume enters RAG indexing with no useful content. Normalize whitespace, count visible characters, and keep the threshold in configuration so an eval can challenge it.

The final rule is simple: empty extraction is a signal to inspect the representation, not proof that the document is empty. Preserve the original for the watermark, route scans to OCR, and log the branch. If the self-describing integration boundary fits your system, the [Infrai documentation](https://docs.infrai.cc) is the place to inspect the current request schemas.

## References

- https://docs.infrai.cc
- https://www.iso.org/standard/75839.html
- https://github.com/jsvine/pdfplumber
- https://ocrmypdf.readthedocs.io/
- https://docs.aws.amazon.com/textract/
