---
description: The Input step — capture the app's requirement document and write it to Trillo AI. Use when starting a new app, when the user describes what the app should do, or hands over a requirements PDF/doc/image. Must be done before the SoftwareSpec step.
---

# Step: Input — capture the requirement

`Input` is the first step: the natural-language requirement that the
SoftwareSpec is derived from. It must exist before `SoftwareSpec` unlocks.

## Gather the requirement

- If the user describes the app in chat, use that.
- If they hand you a **PDF / doc / image**, read it yourself (you're
  multimodal) and extract the requirement.
- A few light clarifying questions are fine, but don't over-interview — the
  SoftwareSpec step will structure and fill gaps.

## Write it

Produce the requirement as **HTML** (`<p>`, headings, lists) and write it to
the Input activity:

```
md_update  modelClassName="Input"  appId=<appId>  content="<p>…the requirement…</p>"
```

The server stores the HTML, marks `Input` **COMPLETED**, and unlocks
`SoftwareSpec`. (It also tolerates the requirement wrapped in a
`{title, description, content}` object — but plain HTML in `content` is
simplest.)

## Confirm + hand off

Call `app_status` → confirm `Input: COMPLETED` and `SoftwareSpec: READY`.
Briefly recap the captured requirement, then ask the user whether to generate
the **SoftwareSpec** (the **software-spec** skill).
