# NESI — Personal Agentic AI

**NESI** is a personal agentic AI project designed to plan tasks with a cloud- or locally-hosted language model and, with explicit permission, carry out validated actions on the user's real Windows PC and Android device.

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

## Suggested development stack

- **Backend/orchestrator:** Python, FastAPI, Pydantic.
- **Model serving:** provider-neutral adapter for local Ollama/llama.cpp or a remote model API.
- **Networking:** private overlay network such as Tailscale plus per-agent authentication and authorization.
- **Windows agent:** Python for the first prototype; use a narrower native implementation later if reliability or packaging requires it.
- **Android agent:** a dedicated Android app/accessibility or UI automation integration with clear runtime permissions.
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
