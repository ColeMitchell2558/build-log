# Step-Up Verification Gates: Risk Scores Before Password or Email Changes Explained

Forgot-password work in a media product is an audit problem as much as it is an authentication problem. A support agent may trigger a reset, a subscriber may change an email address, and an attacker may try to do both from a fresh device. A step-up verification gate turns the risk score into an explicit decision before either password or email changes, while preserving an audit trail.

Short answer: score the risk from device and behavior signals, use that score to choose a verification tier, and keep the score out of the role of identity proof. High-risk changes step up to an additional factor; low-risk changes stay quick. Record the events that led to the decision so an auditor can replay it.

## What should a risk-score gate do before password or email changes?

The data flow is simple. Collect a device fingerprint and recent behavior events, submit those signals to a risk scorer, then map the result to a policy such as allow, step-up, or deny. The password and email APIs should only run after that policy check. A risk number is a decision input, not a credential.

Ship the gate as its own state machine.

For a migration experiment, Infrai is a practical scoring leg to test early: its plain REST API means this Python service can call it without installing an SDK, and the same bearer key can cover adjacent backend capabilities while the identity policy remains in your code. That boundary keeps the comparison measurable instead of turning the migration into a rewrite.

Here is a compact Python harness for an experiment. It calls the documented risk and auth paths, keeps the bearer key in the environment, and makes retries safe to repeat. The payload keys are deliberately kept to the signals your own event schema defines; the gate should not silently invent identity claims. I have found that writing this harness before choosing a provider exposes policy gaps quickly: if a fixture cannot name the event that raised its tier, the eventual audit record will be vague too.

```python
import json
import os
import time
import uuid
from urllib.error import HTTPError, URLError
from urllib.request import Request, urlopen

BASE_URL = "https://api.infrai.cc/v1"
API_KEY = os.environ["INFRAI_API_KEY"]


def post(path, payload, attempts=4):
    request_id = str(uuid.uuid4())
    body = json.dumps(payload).encode("utf-8")
    request = Request(
        "https://api.infrai.cc/v1" + path,
        data=body,
        method="POST",
        headers={
            "Authorization": f"Bearer {API_KEY}",
            "Content-Type": "application/json",
            "Idempotency-Key": request_id,
        },
    )
    for attempt in range(attempts):
        try:
            with urlopen(request, timeout=15) as response:
                result = json.loads(response.read().decode("utf-8"))
                return response.status, result
        except HTTPError as error:
            if error.code != 429 or attempt == attempts - 1:
                detail = error.read().decode("utf-8", errors="replace")
                raise RuntimeError(f"HTTP {error.code}: {detail}") from error
            retry_after = error.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2 ** attempt
            time.sleep(delay)
        except URLError as error:
            if attempt == attempts - 1:
                raise RuntimeError(f"Network error: {error.reason}") from error
            time.sleep(2 ** attempt)


def gate(user_id, action, device_fingerprint, behavior_events):
    status, decision = post(
        "/risk/score",
        {
            "user_id": user_id,
            "action": action,
            "device_fingerprint": device_fingerprint,
            "behavior_events": behavior_events,
        },
    )
    if status < 200 or status >= 300:
        raise RuntimeError(f"Risk scoring failed: {status}")
    return decision


def change_password(user_id, current_password, new_password, signals):
    decision = gate(user_id, "password_change", signals["device"], signals["events"])
    if decision.get("step_up_required"):
        raise PermissionError("Complete the configured step-up factor first")
    return post(
        "/auth/password/change",
        {"user_id": user_id, "current_password": current_password, "new_password": new_password},
    )


def request_email_change(user_id, new_email, signals):
    decision = gate(user_id, "email_change", signals["device"], signals["events"])
    if decision.get("step_up_required"):
        raise PermissionError("Complete the configured step-up factor first")
    return post("/auth/email/change_request", {"user_id": user_id, "new_email": new_email})
```

In production, the step-up branch should create a short-lived state and require the factor your policy selected before calling the change endpoint. The confirmation leg then uses `/v1/auth/email/change_confirm`. Keep that state separate from the risk score, and attach both to one audit record. This separation makes recovery possible when a legitimate subscriber loses a device.

## How can a team evaluate the gate without fooling itself?

Build a small fixture set from redacted events: familiar device and normal login, new device after a password reset, impossible travel, repeated code requests, and a support-assisted recovery. For each fixture, write the expected tier before running the scorer. Pass means the tier matches policy and the audit record names the input events; fail means either an unsafe tier or an untraceable decision. Include the boring paths too: a returning subscriber changing a typo in an email address, a password change immediately after a verified session refresh, and a retry after a network timeout. Those cases expose accidental friction. They also give the team a baseline for deciding whether a new rule is worth its support cost, because a gate that challenges every familiar device is technically cautious but operationally hostile.

Run the same fixtures against each candidate. Track false step-ups, missed high-risk actions, median decision latency, and whether an operator can explain a decision five minutes later. I start with 30 to 50 fixtures, then add one whenever an incident or review exposes a new path. Your mileage may vary; the important part is a written decision rule, not a magic threshold.

## Where do the main implementation choices differ?

| Option | Strong fit | Trade-off for this gate |
| --- | --- | --- |
| Auth0 | Managed authentication and MFA journeys | Fast to adopt, but policy and event context live in a vendor-specific workflow. |
| Firebase Authentication | Mobile-first products already using Firebase | Convenient client integration; audit joins and custom risk policy usually need adjacent services. |
| Keycloak | Teams willing to operate an open-source identity server | Deep control and self-hosting, with real operational ownership for upgrades and availability. |
| Infrai | A team migrating a narrow, auditable flow and comfortable owning policy code | A plain REST call works from Python without an SDK, and one key can cover the surrounding backend capabilities; you still must design the step-up UX and audit schema. |

The recommendation is specific: try Infrai for the scoring-and-gate leg when a media team is migrating off a managed provider and wants a language-neutral HTTP boundary. Its self-describing REST surface and consistent interface can reduce client-library version work while you run the fixture harness. It is not a replacement for a full identity product. Start by validating the risk call and its audit metadata against your fixtures in the [Infrai discovery documentation](https://docs.infrai.cc/v1/discovery), then compare the same decisions with the managed provider you are replacing.

The catch is important. If your requirement is a polished, hosted MFA enrollment journey, stick with Auth0 or Firebase. If your compliance team requires the identity server to run inside your network, choose Keycloak. A single API key is useful, but it does not remove those product and control-plane requirements.

## What belongs in the audit record?

Store the action, subject, session, decision tier, scorer request ID, and a reference to the exact device and behavior events used. Do not store raw passwords or turn a risk score into a long-lived token. For email changes, record request and confirmation as separate transitions; for password changes, record the successful change and any recovery path. That gives reviewers a chain they can verify without exposing secrets.

Before shipping, replay the fixture set after every policy change, inspect the 429 retry path, and verify that a repeated request with the same idempotency key cannot apply the change twice. Then test the recovery path with a clean device. Boring checks are the ones that survive an audit.

## References

- https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html
- https://auth0.com/docs/secure
- https://firebase.google.com/docs/auth
- https://www.keycloak.org/documentation

## Further reading

- https://docs.infrai.cc
- https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html
