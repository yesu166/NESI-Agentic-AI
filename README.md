# NESI — Personal Agentic AI

**NESI** is a personal agentic AI project designed to plan tasks with a cloud- or locally-hosted language model and, with explicit permission, carry out validated actions on the user's real Windows PC and Android device. A planned application area is a YouTube Creator Agent that assists with original content production, publishing workflows, community replies, and channel analytics.

The goal is a reliable **plan → act → observe → verify** loop—not an unrestricted chatbot that can execute arbitrary commands.

> **Architecture principle:** the model is the reasoning layer; the user's devices are the execution layer. Cloud hosting is optional infrastructure, not the product itself. Never assume a cloud VM can control the user's physical devices unless a secure device agent is connected and the path has been verified.

## Project status

This repository currently documents the architecture and roadmap. It does **not** yet contain the application implementation. Status statements below reflect the previously documented setup and must be re-verified before being treated as current.

- [x] Project identity renamed to **NESI** in this repository's documentation.
- [x] Planned architecture and security boundaries documented.
- [~] Earlier notes reported an AWS EC2 Linux environment, llama.cpp model serving, and a private Tailscale route between the cloud VM and Windows PC. Re-check live infrastructure before relying on these.
- [ ] Windows device agent implemented and connected end-to-end.
- [ ] Structured plan/action/observation/verification loop implemented and tested.
- [ ] Android agent integrated.
- [ ] YouTube Creator Agent implemented with official APIs, creator-style memory, review gates, and analytics feedback.
- [ ] Template-driven Shorts rendering implemented with validated edit plans, reusable templates, and automated quality checks.
- [ ] Two niche profiles evaluated for audience overlap before choosing one versus two channels.
- [ ] Durable memory and task history implemented.

## Intended architecture

```text
                       NESI
              User request / task goal
                         |
                         v
              Planner + policy checks
                         |
                Structured action plan
                         |
             Permission / risk validation
                         |
              Tool and device router
                         |
             Private authenticated network
                    (e.g. Tailscale)
                  /                  \
                 v                    v
         Windows device agent   Android device agent
          allowlisted actions    approved UI actions
                 |                    |
          observe / execute     observe / execute
                 \                    /
                  v                  v
                Results + evidence
                         |
                         v
             Verify, recover, or ask user
```

### Model layer

The model should be swappable. Previous experiments used Qwen3 GGUF models (1.7B and 4B) through llama.cpp. These are historical configuration notes, **not a requirement** and not proof that a model server is currently online.

Keep the model interface provider-neutral so NESI can use a local Ollama/llama.cpp server or an authenticated remote provider without coupling the agent orchestration to one model name.

### Orchestration layer

The planned backend is Python + FastAPI with explicit modules for:

- Task intake and clarification.
- Planning into typed, bounded steps.
- Tool registry and argument schemas.
- Policy, permissions, and risk classification.
- Device routing and execution dispatch.
- Execution receipts, observations, and verification.
- Short-term task state; persistent memory later, after the core loop is reliable.

Use structured schemas for actions rather than extracting free-form shell commands from model output. Treat model responses, screen contents, web pages, documents, and tool outputs as untrusted input.

### Device agents

**Windows agent — next major milestone**

Start with a small, authenticated local agent that can report health and basic system information. Add capabilities one at a time:

1. Health/status and device registration.
2. Allowlisted application launch.
3. Screenshot capture and return.
4. Controlled browser/UI actions.
5. Observe-after-action checks and recovery.

Initial end-to-end test:

> “Open Notepad on my PC.”

NESI should return success only when the Windows agent confirms the action and the result is verified. If the PC is offline, permission is denied, or verification fails, report that accurately.

**Android agent — later milestone**

Integrate after the Windows path is stable. Potential capabilities include screen capture, UI inspection, taps, swipes, and text input through an explicitly permissioned accessibility/UI automation layer. Android integration is a plan, not a currently verified feature.

## Agentic planning contract

Each step should have a predictable shape, for example:

```json
{
  "task_id": "generated-task-id",
  "goal": "Open Notepad",
  "steps": [
    {
      "id": "step-1",
      "action": "launch_application",
      "target": "notepad",
      "risk": "low",
      "requires_confirmation": false,
      "success_condition": "Windows agent reports the process launched"
    }
  ]
}
```

This is an illustrative contract, not an implemented API schema. The runtime should validate every field against an allowlist before dispatch.

A safe agent loop:

1. **Understand** — clarify ambiguous or underspecified requests.
2. **Plan** — create a small, bounded sequence of typed actions.
3. **Check** — validate arguments, capability, permission, and risk.
4. **Act** — dispatch only to an enrolled, reachable device agent.
5. **Observe** — capture the execution result and, where appropriate, a screenshot or UI state.
6. **Verify** — evaluate the declared success condition using evidence.
7. **Recover or stop** — retry only when safe; otherwise explain the blocker or request user input.

Use step limits, execution timeouts, cancellation, and a task-level budget to avoid runaway loops. A plan is not evidence that an action happened.


## YouTube Creator Agent (planned)

NESI may include a creator workflow to help manage **the user's own YouTube channel**. The goal is to amplify the creator's ideas and voice—not mass-produce interchangeable videos or simulate fake engagement.

### Creator memory: personalize before fine-tuning

Do not train a foundation model from scratch for the first version. Start with a provider-neutral base model plus a channel profile and retrieval from creator-approved material:

- Channel niche, audience, tone, language, boundaries, and goals.
- The user's own scripts, outlines, captions/transcripts, and approved examples.
- Previously published titles, topics, and available channel analytics.
- Examples of replies that the creator personally approved.
- A correction log recording edits to AI-generated drafts.

Use retrieval-augmented generation (RAG) to provide relevant examples at generation time. Keep facts and source material separate from style preferences. Consider a small LoRA fine-tune only after there is a substantial, curated set of high-quality examples and a measurable reason that retrieval and prompting are insufficient. Never treat audience comments or scraped third-party content as automatically consented training data.

### Content production pipeline

1. **Research** — collect topic ideas and sources; record links and evidence for factual claims.
2. **Select** — rank ideas against the channel niche, audience needs, originality, and production effort.
3. **Draft** — generate multiple hooks, an outline, a script, title options, description, and visual/shot suggestions.
4. **Validate** — check factual claims, originality, asset licensing, sensitive topics, and creator-specific style constraints.
5. **Plan the edit** — emit a typed edit decision list with asset references, in/out timecodes, shot order, captions, overlays, zoom/reframe choices, transitions, music ducking, and target duration.
6. **Render** — apply the validated edit plan through a deterministic renderer such as FFmpeg or a parameterized Remotion composition.
7. **Quality-check** — inspect the rendered file for vertical framing, duration, clipping, caption safe areas, subtitle timing, black/frozen frames, audio peaks, and missing assets.
8. **Review** — show the creator the complete draft and required disclosures before publication.
9. **Publish** — use the official YouTube Data API with user-authorized OAuth permissions; begin with private or unlisted test uploads. Publishing or scheduling must be an explicit configured permission.
10. **Learn** — retrieve authorized channel analytics and use performance as evidence for future experiments, not as a guarantee of virality.

The YouTube Data API supports video uploads through `videos.insert`, and the YouTube Analytics API exposes channel performance reports. API access requires appropriate OAuth authorization: [video upload](https://developers.google.com/youtube/data/reference/rest/v3/videos/insert), [OAuth authorization](https://developers.google.com/youtube/v3/guides/authentication), and [YouTube Analytics](https://developers.google.com/youtube/analytics/reference).

### Template-driven Shorts editor

The proposed editor should reuse a locked visual system while varying the substance and timeline for each Short. Do not ask the model to invent arbitrary editing commands; let it emit a validated edit plan consumed by a deterministic renderer.

**Fixed template properties** can include the vertical canvas, typography, brand colors, caption style and safe zones, logo placement, audio loudness target, intro/outro rules, and transition presets.

**Per-video editable properties** can include the topic, hook, narration, approved footage, clip in/out points, scene order, B-roll, crop/reframe coordinates, emphasis words, subtitle timing, music bed, sound effects, and call to action.

Suggested implementation options:

- **Recommended for this project:** Remotion with React/TypeScript for a reusable, inspectable composition and JSON props; FFmpeg for trimming, audio normalization/mixing, encoding, and final media operations. Remotion supports parameterized rendering: [Remotion docs](https://www.remotion.dev/docs/parameterized-rendering).
- **Alternative hosted service:** a template-rendering API such as Creatomate, where a saved template can be rendered with per-video modifications. This can speed up prototyping but adds API cost and a third-party dependency: [template rendering](https://creatomate.com/docs/api/quick-start/create-a-video-by-template).
- **Existing-editor workflow:** create a human-editable draft or project first and keep final approval manual until the automated output is consistently reliable.

Potential automated steps: generate or import a transcript, find candidate clip boundaries, rank useful moments, cut dead air, reframe to 9:16, track the speaker when possible, generate word- or phrase-timed captions, emphasize key terms, duck background music beneath narration, and produce a preview. These should be separate, testable operations—not one opaque prompt.

Every render should retain a manifest of source asset, license/source, selected timecodes, template version, script version, model/settings used, and QC results. Use only footage and audio that the creator owns, is licensed to use, or is otherwise authorized to include.

### Two niches on one channel

Support two named content profiles, but **do not assume one channel is always the right strategy**. The decision depends on whether the same viewer is likely to want both types of content.

- **One channel can work** when both niches share an audience or a clear umbrella promise—for example, AI tools and practical developer builds for people interested in building with technology. Give each series its own label, playlist, color accent, and recurring structure while preserving the channel-wide design system.
- **Separate channels are usually cleaner** when the viewer intent is very different—for example, programming tutorials and unrelated celebrity gossip. A shared editing template does not make two audiences compatible.
- Test a small, clearly labeled batch of each series and compare audience retention, returning viewers, subscribers gained, and comment quality by series before committing.

YouTube's monetization policy permits a consistent intro/outro when the substance differs, but warns that interchangeable, mass-produced or minimally varied videos may be ineligible for monetization. The videos need meaningful original commentary, storytelling, education, or entertainment—not merely different nouns in the same shell: [YouTube monetization policy](https://support.google.com/youtube/answer/1311392).

For NESI, store each niche's topic constraints, vocabulary, audience persona, script pattern, approved source types, and visual theme in a separate `niche_profile`. Both can share rendering infrastructure, but the script and story must be specific to the selected niche.

### Community inbox

Start with an **assistive inbox**, not a bot that replies indiscriminately:

- Read comments from the authorized channel using the official API.
- Classify them as question, praise, constructive criticism, spam, safety concern, or sensitive/uncertain.
- Draft specific replies in the creator's voice, grounded in the video and verified context.
- Show the original comment, proposed response, and a reason/confidence indicator.
- Require creator approval by default; any later automatic replies should be opt-in, rate-limited, and limited to a narrow set of low-risk, approved FAQs.
- Escalate harassment, threats, privacy issues, controversy, legal/medical/financial topics, and ambiguous situations for human review.
- Never post repetitive unsolicited comments, generate fake engagement, impersonate a real person, or use engagement bait to manipulate metrics.

The official API supports listing comment threads and inserting replies, subject to authorization and quota limits: [comment threads](https://developers.google.com/youtube/v3/docs/commentThreads) and [comment replies](https://developers.google.com/youtube/v3/docs/comments/insert).

### Required creator safeguards

- The creator remains responsible for what gets published and for compliance with Community Guidelines and copyright.
- Require disclosure when realistic, meaningfully altered or synthetic content falls under YouTube's disclosure requirements: [YouTube AI disclosure guidance](https://support.google.com/youtube/answer/14328491).
- Avoid generic, interchangeable, mass-produced uploads. YouTube's monetization policy identifies repetitive or mass-produced content as "inauthentic content": [monetization policies](https://support.google.com/youtube/answer/1311392).
- Use original commentary and a distinct creative contribution; keep source links and asset provenance with each draft.
- Protect OAuth refresh tokens as secrets, request only necessary scopes, support disconnect/revocation, and minimize stored commenter data.
- Keep a human-review gate for video publishing, contentious replies, moderation actions, and changes to channel settings.

### First Creator Agent vertical slice

1. Import the user's channel profile and a small set of their own scripts.
2. Generate three distinct video ideas and one script with sources and uncertainty notes.
3. Let the creator edit and approve the script.
4. Generate a metadata package and a production checklist; no public upload yet.
5. Build a comment inbox that drafts replies but does not send them automatically.
6. Only after review and testing, add OAuth upload and scheduled publishing.
7. Use analytics to compare outcomes and update the channel profile—not to automatically repeat a template that merely performed well.

This feature is planned only. It is **not implemented** in the current repository.

## Security model

Device-control software must be conservative by default:

- Do not expose device-control APIs directly to the public internet.
- Use authenticated device enrollment and private networking; network membership alone should not be treated as full authorization.
- Require TLS where supported, authenticate every request, and reject replayed or expired requests.
- Never permit unrestricted shell, PowerShell, Python, or arbitrary-code execution from model output.
- Maintain a strict action allowlist, typed arguments, risk levels, and auditable execution receipts.
- Require explicit user confirmation for destructive, financial, privacy-sensitive, or high-impact actions.
- Keep credentials and API keys in environment variables or a secret manager; never commit them.
- Treat screenshots, OCR, and external content as untrusted data that may contain prompt injection.
- Minimize captured data, avoid retaining sensitive screenshots unnecessarily, and provide a clear stop/disable control.
- Return verifiable evidence to the planner and never claim success without it.

## Development roadmap

### Milestone 1 — Establish the baseline
- Verify the actual model-serving environment and device connectivity.
- Add configuration and health checks with secrets excluded from logs.
- Record a reproducible setup procedure.

### Milestone 2 — Windows agent vertical slice
- Implement authenticated health/status.
- Implement one allowlisted action: launch Notepad.
- Return structured receipts and verify the outcome.
- Add timeout, offline, rejected-action, and failure tests.

### Milestone 3 — Planning and policy
- Define typed request, plan, action, observation, and result schemas.
- Validate model-generated plans before execution.
- Add risk gates, user confirmations, step budgets, cancellation, and logs.

### Milestone 4 — Observe and verify
- Add opt-in screenshot capture.
- Check post-action state and distinguish success, failure, and uncertain outcomes.
- Add safe recovery behavior.

### Milestone 5 — Browser and Android
- Expand Windows UI automation only after the first vertical slice is stable.
- Add Android through a separately permissioned device agent.
- Reuse the same action contract and policy engine.

### Milestone 6 — Memory and reliability
- Add explicit user-controlled memory, task history, and retention controls.
- Add integration tests, structured logging, and a setup guide.
- Benchmark candidate models against real planning/tool-use tasks instead of choosing by parameter count alone.

### Milestone 7 — YouTube Creator Agent
- Build a channel profile and retrieval over the creator's own approved scripts and examples.
- Generate topic ideas, sourced scripts, metadata, and production checklists for manual review.
- Add a comment inbox with classifications and reply drafts; do not send replies automatically in the first version.
- Add YouTube OAuth and official API integration after draft workflows are reliable.
- Add explicit publication approval, AI disclosure checks, asset provenance, privacy controls, and analytics feedback.

## Suggested development stack

- **Backend/orchestrator:** Python, FastAPI, Pydantic.
- **Model serving:** provider-neutral adapter for local Ollama/llama.cpp or a remote model API.
- **Networking:** private overlay network such as Tailscale plus per-agent authentication and authorization.
- **Windows agent:** Python for the first prototype; use a narrower native implementation later if reliability or packaging requires it.
- **Android agent:** a dedicated Android app/accessibility or UI automation integration with clear runtime permissions.
- **YouTube integration (planned):** YouTube Data API v3 and YouTube Analytics API through OAuth 2.0; begin with read-only access and draft-only workflows where possible.
- **Creator memory (planned):** channel profile + retrieval over creator-approved material; add fine-tuning only when a curated dataset justifies it.
- **Shorts rendering (planned):** Remotion + React/TypeScript for editable compositions, FFmpeg for media operations, and validated JSON edit manifests.
- **Niche profiles (planned):** separate per-niche audience, style, topic and source constraints while sharing the render/upload pipeline.
- **Testing:** unit tests for policy/schema logic, mocked device agents, and explicit end-to-end tests against a test device.

These are recommendations for the implementation, not a claim that these components have all been built.

## Repository layout target

```text
NESI-Agentic-AI/
├── README.md
├── backend/
│   ├── app/
│   │   ├── api/
│   │   ├── planner/
│   │   ├── policy/
│   │   ├── tools/
│   │   ├── devices/
│   │   └── verification/
│   └── tests/
├── agents/
│   ├── windows/
│   └── android/
├── docs/
│   ├── architecture.md
│   ├── security.md
│   └── roadmap.md
└── scripts/
```

This is a proposed layout. The current repository does not yet contain these directories.

## Definition of done for the first demo

The first credible demo is deliberately narrow:

1. User asks NESI to open Notepad.
2. Planner emits a structured action.
3. Policy validates it against the allowlist.
4. An authenticated Windows agent receives and executes it.
5. The agent returns a receipt.
6. NESI verifies the result and reports the evidence.
7. If any stage fails, NESI reports the failure instead of fabricating success.

**Current focus:** verify the environment, then build this Windows end-to-end slice before expanding to screenshots, browser automation, Android, or long-term memory.
