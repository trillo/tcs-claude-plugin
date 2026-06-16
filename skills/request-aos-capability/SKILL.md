---
description: File an AOS enhancement request when an app requirement needs a platform capability AOS doesn't support yet (webhooks, scheduled jobs, a specific integration, etc.). Use when you hit a Gap in the coverage map — confirm with the user, then file it.
---

# Requesting a new AOS capability

When an app requirement needs something Trillo AOS doesn't provide yet, don't
fake it or quietly work around it — surface it and offer to file an enhancement
request so the platform team can prioritize it.

## Detect the gap

Ground yourself with **`aos_capabilities`** (the coverage map — each capability
is tagged *In place / Partial / Gap*, numbered, with a *"What's needed"* note).
If the requirement maps to a **Gap** (or something not covered at all), it's a
candidate. Note the two dimensions:
- `toolkit_stubs` covers what a **function** can do from code (data/files/…).
- the coverage map covers **platform/runtime** capabilities (triggers,
  webhooks, integrations) — these are the ones a function can't just call.

## Tell the user, then file

1. **State it plainly:** what's needed, that AOS doesn't have it yet, and the
   coverage-map item # if there is one. (Trillo usually turns these around in
   under a day.)
2. **Ask the user before filing — do not send silently** (it sends an email).
3. On their OK, call **`request_aos_capability`**:
   `{ feature, whyNeeded, coverageItem?, suggestedApi?, appId?, requesterEmail? }`
   → emails `aos-enhancement@trillo.io`. Confirm to the user what was filed.

## Then keep moving

Note that the feature is blocked pending the capability, and continue building
the parts of the app you *can* build now.
