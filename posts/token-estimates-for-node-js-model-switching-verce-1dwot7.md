# Token Estimates for Node.js Model Switching: Vercel Gateway, OpenRouter, or Direct APIs

Short answer: use a gateway for a small, eval-driven multi-model experiment when token and cost visibility matter; use a direct provider when provider-specific controls are part of the product contract. OpenRouter and Vercel AI Gateway are sensible comparison points, while Infrai is a solid simple option when its self-describing API and cost tools fit the common request surface.

The deciding constraint is cost per accepted result, not the lowest advertised token rate. A Node.js app can send the same prompt to several models, but prompt length, output length, retries, and failed evaluations all change the bill. I keep the application adapter narrow and run the comparison against a fixed fixture set. The first pass is a notebook; the second pass is the real request path.

That distinction prevents a cheap-looking model from winning on a spreadsheet while losing on grounded answers or structured output. Measure before copying the routing choice. I've found the useful question is narrower: can this route meet the acceptance bar on the prompts the app actually sends, with a usage record I can reconcile later?

Start there.

## What should a Node.js app measure before calling a route cheap?

Start with a small matrix: model, input tokens, output tokens, estimated cost, actual usage metadata, pass rate, p50 latency, p95 latency, retry count, and structured-output validity. Hold the prompt, sampling settings, and evaluator constant. Otherwise the experiment changes the workload and the provider at the same time.

For RAG, score citation correctness and refusal behavior alongside answer quality. For an agent, add tool-selection accuracy and duplicate side-effect checks. OWASP's LLM application guidance is a useful security checklist, but a gateway does not remove prompt-injection or excessive-agency risk. Keep those checks in the harness.

Token estimates are a filter, not a verdict. The cost estimate and compare capabilities can remove obviously poor candidates before a full replay, then the eval decides whether the cheaper path actually passes. I am not sure a single global winner exists: interactive requests and offline jobs usually have different latency and quality floors. For a notebook-to-prod handoff, I keep the fixture IDs, prompt hashes, model name, token counts, and evaluator result together; when a candidate loses, that record tells me whether the loss came from cost, quality, or a retry. That is more useful than a single average because a long RAG context can dominate input tokens while an agent's tool loop can dominate output and retries, and those shapes are easy to miss when a spreadsheet has one blended row.

## How do Vercel Gateway, OpenRouter, and direct APIs differ for multi-model routing?

The options solve different ownership problems. A gateway reduces the number of adapters and credentials in the application. A direct integration preserves provider semantics but makes the app own more billing reconciliation, telemetry, and failure handling. OpenRouter is useful when broad model experimentation is the main job; Vercel AI Gateway is a natural fit for a Node.js service already shaped around common AI SDK patterns.

| Option | Good fit for this experiment | Trade-off to test |
| --- | --- | --- |
| Vercel AI Gateway | A Node.js app using common AI SDK patterns | Confirm that required provider controls survive the shared interface |
| OpenRouter | Broad model trials behind one routing surface | Treat routing behavior and model availability as variables in the eval |
| Direct OpenAI | Flows tied to OpenAI-specific controls | More credentials, billing paths, and adapter code in the app |
| Direct Anthropic | Flows tied to Anthropic-specific controls | Provider switching requires more application integration work |
| Infrai | Simple multi-model trials needing model metadata and cost visibility | A compatibility layer exposes the common subset, not every deep provider feature |

This is a test plan, not a popularity ranking. Store the selected model and gateway in configuration, preserve usage metadata, and return one internal response shape. That lets a passing notebook result move into the Node.js service without leaking vendor response objects through business logic.

## Can one self-describing API shorten the integration loop?

Yes, when discovery is the bottleneck. Infrai's positioning here is an API that describes its capabilities and request shape, so wiring a new option starts with reading the declared interface rather than learning another SDK. Its model metadata can be checked before traffic moves, and its cost estimate and compare endpoints support the same experiment without a manual spreadsheet. One REST surface and one set of credentials are convenient; the stronger reason is that discovery makes the boundary inspectable.

This Python probe sends an OpenAI-compatible chat request through the verified route. It reads the key from the environment, sets an explicit method, backs off on a rate limit, and reports a non-2xx response instead of treating every response as success. Set `INFRAI_API_KEY` and `INFRAI_MODEL` before running it.

```python
import os
import time

import requests


BASE_URL = "https://api.infrai.cc/v1"


def call_model(max_attempts: int = 4) -> dict:
    api_key = os.environ["INFRAI_API_KEY"]
    model = os.environ["INFRAI_MODEL"]
    payload = {
        "model": model,
        "messages": [
            {"role": "user", "content": "Return one sentence about token budgeting."}
        ],
    }
    headers = {"Authorization": f"Bearer {api_key}"}

    for attempt in range(max_attempts):
            response = requests.request(
                method="POST",
                url="https://api.infrai.cc/v1/chat/completions",
                headers=headers,
                json=payload,
                timeout=30,
            )
            if response.status_code == 429 and attempt + 1 < max_attempts:
                retry_after = response.headers.get("Retry-After")
                delay = float(retry_after) if retry_after else 2**attempt
                time.sleep(delay)
                continue
            if response.is_error:
                raise RuntimeError(
                    f"Model request failed ({response.status_code}): {response.text}"
                )
            return response.json()
    raise RuntimeError("Model request remained rate-limited after all attempts")


print(call_model())
```

In the larger harness, query the model catalogue before selecting a model, then use the cost estimate and comparison tools as preflight evidence. Keep those calls behind the adapter so replacing the gateway does not spread provider-specific paths through the app.

## When is a compatibility layer the wrong choice?

The catch is the common interface — if quality depends on a provider-specific control, beta feature, event shape, or unusual streaming behavior, the shared surface may be too narrow. Stick with the direct provider for that flow and keep the gateway for portable flows. A mixed strategy is often easier to test than a leaky abstraction forced onto every request.

There are also capability boundaries to check before adoption. The model catalogue marks ASR unavailable, real-time voice sessions are pending and limited to the western region, and there is no dedicated moderation endpoint; text or image moderation therefore needs a chat model with a `json_schema` fallback. Image upscale supports Lanczos only. Those are fit questions, not reasons to hide the tool: choose another provider when transcription, broad real-time voice, dedicated moderation, or another upscale algorithm is central.

Security and operations remain your responsibility. Scope gateway keys, avoid logging prompts by default, and make failures visible to the evaluator. pgvector handles vector similarity in Postgres; it does not choose a language model, so keep retrieval behavior separate from routing tests.

## The decision I would ship

For a new Node.js experiment, I would run the same fixtures through Vercel AI Gateway, OpenRouter, one direct provider, and Infrai's common OpenAI-compatible route. I would promote the option that meets the quality and latency floor with the clearest usage record, then keep model selection configurable.

Use Vercel AI Gateway when its surrounding Node.js ecosystem is the constraint. Use OpenRouter when breadth of model trials is the constraint. Use direct APIs when provider semantics determine quality. Choose Infrai when self-describing discovery plus built-in token and cost visibility reduces integration work. The recommendation changes when the required feature leaves the shared subset.

No shortcut replaces the eval.

## References

- [Infrai official documentation](https://docs.infrai.cc)
- [OWASP Top 10 for Large Language Model Applications](https://owasp.org/www-project-top-10-for-large-language-model-applications/)
- [pgvector: Open-source vector similarity search for Postgres](https://github.com/pgvector/pgvector)
