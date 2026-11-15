# Software factory: services

The services half of this repo's software factory design, in the vocabulary of
the `software-factory` skill (`.agents/skills/software-factory/SKILL.md`).
Tasks are not designed yet; they build on this.

Decided 2026-11-15. Prices and limits were checked then and go stale fast:
re-check them before relying on a number.

## Slots

| Service                | Product                                           |
| ---------------------- | ------------------------------------------------- |
| Ticket tracker         | Linear (Free plan)                                |
| Ticket source          | None yet: only humans file tickets                |
| Git host               | GitHub (`kkrampa/cohort-005-project`, public)     |
| Orchestrator           | GitHub Actions, with a Cloudflare Worker relay    |
| Runner                 | E2B sandboxes, started from a thin Actions job    |
| Observability platform | None, deliberately: the Actions run page suffices |

### Ticket tracker: Linear Free

- Free plan: unlimited members, 2 teams, 250 issues.
- When the issue count nears 250, either archive or delete closed tickets, or
  upgrade to Basic (about $10 per user per month). Check then whether closed or
  archived issues count toward the cap.
- Linear's agent API was a developer preview as of 2026-10. The design does not
  depend on it.

### Ticket source: none yet

- Humans file every ticket at first.
- First candidate to add: a feedback button in the app that files reports
  through the Linear API, throttled, and that looks for an open ticket before
  filing a new one.
- Every ticket source added must answer: what stops it filing the same ticket
  twice? Each one also spends the 250-issue cap.

### Git host: GitHub

- `kkrampa/cohort-005-project`. It must stay public: standard Actions runners
  are free on public repos only.

### Orchestrator: GitHub Actions plus a relay

- Actions receives triggers, starts runs and records which are queued, in
  flight or finished. Concurrency groups keep two runs off the same ticket.
- PR events and clocks trigger Actions directly.
- Linear cannot trigger Actions directly: GitHub's dispatch API needs a bearer
  token and its own payload shape, and Linear's webhooks cannot add an
  `Authorization` header. A **Cloudflare Worker relay** (Free plan: 100,000
  requests a day) verifies `Linear-Signature` and calls `repository_dispatch`.
- Rejected:
  - Linear's GitHub Issues sync: it would mirror every ticket into a second
    tracker.
  - Polling Linear from a scheduled Action: GitHub's scheduled runs fire every
    5 minutes at best and are often late.
- Deferred: Temporal, Inngest or an own server plus database. Revisit when
  Actions' concurrency, 6-hour job limit or retry handling get in the way. The
  runner code carries over unchanged.

### Runner: E2B

- The Actions job is a thin launcher. It creates an E2B sandbox, starts Claude
  Code inside it, streams its output into the job log, and kills the sandbox at
  the end.
- Sandbox size: 2 vCPU / 4 GiB (about $0.17 an hour). The default 512 MiB is
  too small for `npm install` and the test suite.
- Every sandbox gets a strict timeout as the backstop. Billing is wall-clock,
  including time spent waiting on the model.
- Hobby plan: $100 one-time credit, 1-hour max run, about 20 runs at once.
  Pro ($150 a month plus usage) is only needed for longer runs or more runs at
  once.
- Plain Actions runners are the fallback. Runner choice is per task: keep the
  launcher thin so Modal ($30 credit a month) or Daytona (no fixed max run
  length) can be added and compared.

### Observability platform: none, deliberately

- Each run's page shows the streamed Claude Code output, and the full
  transcript is uploaded as an artifact. E2B's dashboard shows sandbox usage.
- Add an OpenTelemetry tracing platform (e.g. Langfuse) when questions span
  many runs: cost per task, failing tools.

## Identity and secrets

| Credential                    | Used by                          | Stored as               |
| ----------------------------- | -------------------------------- | ----------------------- |
| GitHub App private key        | Relay (dispatch), runs (git, PR) | Worker + Actions secret |
| Linear webhook signing secret | Relay                            | Worker secret           |
| Linear API key or OAuth app   | Runs                             | Actions secret          |
| E2B API key                   | Launcher job                     | Actions secret          |
| Claude subscription token     | Claude Code in the sandbox       | Actions secret          |

- **One GitHub App identity from day one**, for the relay and for runs. Events
  caused by the built-in `GITHUB_TOKEN` start no new workflow runs, so a chain
  of runs would silently stop. Never use a personal token.
- **The Claude subscription token enters each E2B sandbox only for that run.**
  Anything the run executes there can read it; this is accepted for a personal
  project. Limit what the sandbox can reach over the network where E2B allows
  it, and revoke and regenerate the token after any run you did not expect.
  Switch to a spend-capped Anthropic API key if this stops being a personal
  project.

## Cost

| Item                           | Cost per month                                                          |
| ------------------------------ | ----------------------------------------------------------------------- |
| Linear, GitHub, Actions, relay | $0                                                                      |
| E2B                            | $0 until the $100 credit is spent, then about $5 to $35 at course scale |
| Claude subscription            | $100 minimum, per the course README                                     |
