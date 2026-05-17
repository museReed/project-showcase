# Portfolio Deck — Reed Hsin

## From Building Apps to Building AI Systems

> Senior iOS Engineer → AI Technical Program Manager

- 8+ years shipping at scale: TikTok, Crypto.com, 17Live, Pinkoi, MoFA
- Built a production AI platform: 17 modules, 4 LLM providers, 24/7 operation
- Led cross-functional teams of 10+ across SG, TW, US, UK timezones

---

## My Career Arc

### Timeline

| Period | Role | Key Achievement |
|--------|------|-----------------|
| 2017-2019 | iOS Engineer, MoFA Taiwan (Belize) | Cross-cultural collab, 3 public sector apps |
| 2019-2020 | iOS Engineer, Pinkoi | Login & shipping system renovation |
| 2021-2022 | Senior iOS, 17Live | Live room architecture MVC→MVVM-C, US team sprint 60%→85% |
| 2023-2024 | Senior iOS, Crypto.com | Onboarding module, test coverage 42%→75% |
| 2024-2025 | Senior iOS, TikTok Live | Tech Lead Q&E, Project Owner (10-person team) |
| 2025-now | AI Platform Builder | Muse Platform — solo architect + PM + SWE + QA |

### The Pattern

Every role moved me closer to the intersection of **technical depth × delivery leadership**. TikTok's Project Owner experience was the turning point — I realized I'm best at *driving technical programs to completion*.

---

## Why AI TPM?

### The Real Challenges of AI Projects

AI delivery challenges are **not** about the model itself. They're about:

| Challenge | What It Means |
|-----------|---------------|
| **Pipeline Reliability** | LLMs are non-deterministic; retries, fallbacks, quality checks |
| **Cost Unpredictability** | Token usage varies wildly; need routing + budget controls |
| **Quality Gates** | How do you QA AI output at scale? |
| **Delivery Cadence** | AI experiments ≠ traditional sprints; need different planning |
| **Cross-team Deps** | ML eng × Backend × Product × Data — everyone speaks different languages |

### My Edge

I've experienced these challenges as a **builder**, not just a manager. I designed solutions for each one.

---

## Muse Platform — Architecture Overview

### Four-Layer Clean Architecture

```
┌───────────────────────────────────────────────────────┐
│  INTERFACES                                           │
│  Ops UI (Next.js) │ Telegram Agent │ CLI │ MCP Server │
├───────────────────────────────────────────────────────┤
│  FEATURES — 17 Automated AI Modules                   │
│  Content Collectors │ Personal Productivity │ Knowledge│
├───────────────────────────────────────────────────────┤
│  PLATFORM — Shared Libraries                          │
│  LLM Routing │ Catalog │ Notify │ HTTP │ Collectors   │
├───────────────────────────────────────────────────────┤
│  INFRA — Foundation                                   │
│  Paths │ Secrets │ Logging │ Deploy │ Scheduler       │
└───────────────────────────────────────────────────────┘
```

### Key Numbers

| Metric | Value |
|--------|-------|
| Feature modules | 17 |
| Content sources | 4 (XHS, YouTube, Twitter, RSS) |
| LLM providers | 4 (Anthropic, DeepSeek, Groq, Gemini) |
| Scheduled jobs | 20+ daily |
| Interfaces | 4 (Ops UI, Telegram, CLI, MCP) |
| Architecture layers | 4 (CI-enforced dependency rules) |

---

## Deep Dive #1 — LLM Cost Optimization

### Problem

Claude Agent SDK: **~27,000 token overhead** per call = **83% wasted** on system prompt

### Approach

1. **Abandoned SDK** → Direct Anthropic Messages API
2. **Task-based model routing** — TOML config maps task types to optimal providers
3. **A/B test framework** — 4 configurations tested simultaneously:
   - Baseline (Claude-only)
   - Current-prod (mixed)
   - Groq-free tier
   - Mixed-optimal

### Result

**83% cost reduction** — quality validated through A/B testing

### TPM Takeaway

> Quantify the trade-off → Let data drive the decision → Validate the hypothesis.
> This is what AI TPMs do every day.

---

## Deep Dive #2 — Parallel Delivery at Scale

### Problem

17 features to migrate. Sequential delivery = **weeks**.

### Approach

1. **6-wave plan** (A-G): 3-5 features per wave
2. **Git worktree isolation**: 3-5 Claude subagents work in parallel
3. **Conflict resolution SOP**: `pyproject.toml` is a guaranteed conflict point — documented merge protocol
4. **Quality gates don't bend**: Each agent runs the full 10-step ceremony

### Result

| Metric | Before | After |
|--------|--------|-------|
| Wall-clock per wave | ~27 min | ~9 min |
| Improvement | — | **-66%** |
| Features landed | — | 17/17 |
| Production incidents | — | 0 |

### TPM Takeaway

> Like managing 3-5 scrum teams in parallel:
> Dependency mapping + clear contracts + conflict resolution protocol + quality gates that don't compromise for speed.

---

## Deep Dive #3 — Platform Engineering

### Problem

17 modules, each needing routing, scheduling, monitoring, QA. **Manual config doesn't scale.**

### Solution: `muse.toml` Manifest System

```toml
[module]
layer = "features"
kind = ["job"]

[job]
schedule = "0 */6 * * *"
entry = "run_pipeline"

[verify]
status = "ok"
max_errors = 0
duration_max = "5m"
```

### How It Works

```
muse.toml (per module)
    ↓ muse_catalog reads all manifests
    ↓ ops-server auto-mounts routers
    ↓ scheduler auto-registers jobs
    ↓ CLI auto-validates on CI
    ↓ Ops UI auto-generates feature cards
```

**Adding a new module = write one muse.toml. Everything else is automatic.**

### TPM Takeaway

> IDP core value = **guardrails + self-service**.
> Teams ship fast while staying within quality boundaries.
> This is what platform teams at Google, Meta, Uber do — I built one from scratch.

---

## Quality System Design

### Multi-Layer Quality Gates

```
Layer 1: LOCAL
  └─ 4-agent automated review
     (code / test / error-handling / comments)

Layer 2: CI
  └─ Gemini code review + Ruff lint
     (required merge checks)

Layer 3: VERIFICATION
  └─ Log-Based Verification Gate
     (parse JSONL logs → validate pipeline health)

Layer 4: MANUAL
  └─ Ops UI QA Checklist
     (human sign-off → Mark Ready gate)

Layer 5: MERGE
  └─ All 4 layers green
     → auto-merge → branch auto-delete
```

### Result

- Zero production incidents across all 17 modules
- Quality and speed are NOT trade-offs — **if you automate the gates**

---

## Bridging: TikTok Project Owner Experience

### Interactive Creator Competition

- Cross-functional team: **10 members** (PM, Designer, Backend, iOS, Android, QA)
- Drove increases in: daily gift-sending users, follower conversions, creator retention

### What I Learned

| Lesson | How It Applies to TPM |
|--------|----------------------|
| Aligning 10 people requires shared metrics | OKR-driven program management |
| Technical decisions impact timelines | TPM must understand engineering trade-offs |
| AB testing validates everything | Data-driven decision making |
| Cross-timezone coordination (SG/TW) | Stakeholder management at scale |

### 17Live: US Team Sprint Improvement

- Problem: Sprint completion rate **60%** (vs Taiwan team's 90%+)
- Solution: Design review process + API spec alignment + rotating sync-ups
- Result: **60% → 85%** in 6 sprints

---

## What I Bring to an AI TPM Role

### Capability Matrix

| From iOS Engineering | From Muse Platform | → AI TPM Value |
|---|---|---|
| Clean Architecture, MVVM-C | 4-layer system design | Technical credibility with engineers |
| Project Owner (10-person) | 6-wave parallel delivery | Program delivery at scale |
| Sprint 60%→85% improvement | Workflow gates, automation | Process improvement |
| Performance optimization | LLM cost optimization, A/B test | Data-driven decisions |
| Cross-timezone (SG/HK/UK/US) | Multi-agent coordination | Stakeholder management |

---

## Key Differentiators

### vs. PM Candidates
> I can read the code, design the architecture, and quantify technical trade-offs.

### vs. Engineering Candidates
> I've led cross-functional delivery, designed processes, and optimized team velocity.

### vs. Other AI TPM Candidates
> I've **built** a production AI system end-to-end.
> I understand AI delivery challenges from first principles, not slide decks.

### The One-Liner

> "Most AI TPM candidates have managed AI projects. I've *built* one — and then built the platform to manage it."

---

## What I'm Looking For

- AI product team where **technical depth matters**
- Opportunity to drive **delivery of AI/ML features at scale**
- Team that values **systematic process improvement**
- Environment where I can **bridge engineering and product**

---

## Appendix: Tech Stack

| Category | Technology |
|----------|------------|
| Language | Python 3.12+ |
| Package Manager | uv (workspace monorepo) |
| Web Framework | FastAPI |
| Frontend | Next.js (Ops UI) |
| LLM | Anthropic Messages API, DeepSeek, Groq, Gemini |
| Database | SQLite (per-feature, with FTS5) |
| Browser Automation | Playwright + CDP |
| CI/CD | GitHub Actions (Gemini review + Ruff lint) |
| Scheduling | launchd (macOS) |
| Notifications | Telegram Bot API |
| Testing | pytest, VCR.py |
| Hardware | Mac Mini M2 Pro 32GB (prod) |
