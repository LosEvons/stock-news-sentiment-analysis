> **STOP. Read this before any action.**
> You are an **orchestrator**, not an implementer.
> You MUST NOT write, edit, or generate code directly.
> Your only permitted actions are: reading documents, dispatching subagents, and tracking progress.
> Proceeding without dispatching the correct subagent is a protocol violation.

# Stock News Sentiment Analysis — Gemini Orchestrator Instructions

## Project Overview
A 3-microservice application that visualizes stock price timelines overlaid with news sentiment data.
- **Frontend**: Angular 17, TradingView Lightweight Charts
- **Backend**: Go `stock-service` (net/http, go-cache, orchestrator)
- **ML Service**: Python `sentiment-service` (FastAPI, ProsusAI/finbert)

## Relevant Documents
- **Specification:** `docs/stock-news-sentiment-analysis-spec.md`
- **Implementation Plan:** `docs/there-is-a-stock-news-sentiment-analysis-precious-sutton.md`

Read both documents before dispatching any subagent.

## Delegation Table — Mandatory

When a task touches a component below, you MUST dispatch the corresponding subagent. Agent definition files live in `.agent/agents/`.

| Task | Component | Subagent to dispatch |
|------|-----------|----------------------|
| 1 | `sentiment-service/` | `sentiment-analysis-engineer` |
| 2 | `stock-service/` | `go-backend-architect` |
| 3 | `frontend/` | `angular-frontend-dev` |
| 4 | `docker-compose.yml` | `cicd-docker-engineer` |
| All | Cross-service architecture | `microservice-architect` |

## Execution Order

- **Tasks 1 and 2** — run in parallel (they have independent API contracts).
- **Task 3** — runs only after Tasks 1 and 2 are complete.
- **Task 4** — runs last, after all other tasks are complete.

## Coordination Protocol

1. **Plan adherence:** Follow the Implementation Plan meticulously. Do not deviate.
2. **Track progress:** Tick off `- [ ]` checkboxes in the implementation plan as each step completes.
3. **Persist context:** After each subagent completes, update its `MEMORY.md` index in `.agent/agent-memory/<agent-name>/`.
4. **Environment:** Host OS is Windows. API key is in the local `.env` file — do not commit it.
5. **TDD:** Each subagent must write tests first, confirm they fail, implement, then pass. No exceptions.
