# Build

A skill for building, reviewing, and assessing full-stack apps (FastAPI + React + Docker) against one set of conventions. It has **three modes**, chosen by the first argument:

| Command | Mode | What it does |
|---------|------|--------------|
| `/build new [project] [phase]` | **new** | Scaffold/continue a phased build (the phases below) |
| `/build review [scope]` | **review** | Review the current git branch's changes against the conventions (read-only) |
| `/build assess [subtree]` | **assess** | Audit the whole repo for conformance with the conventions (read-only) |

`review` and `assess` check against [`conventions.md`](conventions.md) — the audit rubric distilled from the phase docs (which stay canonical). See [`SKILL.md`](SKILL.md) for how each mode runs.

## Building (mode: `new`)

Sequential phases, fed to an agent one at a time, in order.

1. Start at Phase 01 and work through sequentially
2. Each phase assumes all prior phases are complete
3. Each phase ends with a checklist — verify before moving to the next
4. Make the **decisions** below as they come up, and pull in the matching **reference doc** when the app needs that capability

## Decisions to Make

These are the branch points — choices that change the structure, dependencies, or deploy. Decide each as early as its "when" column; the default is the simplest option.

| Decision | When | Options (default first) | Affects |
|----------|------|-------------------------|---------|
| **Deployment path** | Phase 01 (commit early) | **A: Raspberry Pi** · **B: Azure** · **C: AWS** | Project structure, async model, DB hosting, CI/CD, Dockerfiles, Terraform — see Phase 08 |
| **Auth strategy** | Phase 02 | **Email + password** · *+ Google sign-in* · *+ mandatory email verification* | User model (`google_sub`, `email_verified`, optional `hashed_password`), auth endpoints, config, frontend auth UI, frontend build args |
| **Background workers** | When async work appears | **None (return inline)** · *Workers* | Redis in compose, `worker/` module + handlers, `events.publish()` calls, deploy (SNS Lambdas on AWS, KEDA worker on Azure) |
| **File storage** | When the app stores files | **Local filesystem** · *Azure Blob* | `STORAGE_BACKEND` config, backend class in lifespan, `azure-storage-blob` dep (Azure only), compose volume |
| **GitHub feedback** | Phase 09 (optional) | **Skip** · *Enable* | `feedback_github_*` config, `/api/feedback` route, feedback modal |
| **Agent auth & API** | Phase 10 (optional) | **Skip** · *Enable* | `/api/auth/agent-token`, OpenAPI exposure, profile-page token UI + downloadable skill |
| **Ports** | Phase 01 | API `8020`, frontend `8095` (defaults) | compose, vite proxy, nginx |

### Deployment paths at a glance

| | **A: Raspberry Pi** | **B: Azure** | **C: AWS** |
|---|---|---|---|
| **Best for** | Internal/family apps | Public apps, always-warm option | Public apps, cheapest at idle |
| **Hosting** | Self-hosted Docker + Cloudflare Tunnel | Container Apps (scale-to-zero) | Lambda + API Gateway + CloudFront |
| **Async work** | In-process / host | Always-on Redis + KEDA worker | SNS → per-handler Lambdas |
| **Database** | MongoDB on Pi (shared) | MongoDB Atlas | MongoDB Atlas |
| **Cost (idle)** | Free (hardware) | ~$35–50/mo | ~$0–5/mo |
| **Example** | `calendarapp`, `track` | `toolshed` | `recall` |

## Phases

| Phase | Name | Scope | Description |
|-------|------|-------|-------------|
| 01 | [Project Setup](phases/01-project-setup.md) | Full-stack | Scaffold the project, configure dependencies, choose ports + deployment path |
| 02 | [Backend Foundation](phases/02-backend-foundation.md) | Backend | FastAPI app, auth system, health check |
| 03 | [Backend Domain](phases/03-backend-domain.md) | Backend | Database models, DTOs, resource routes, business logic |
| 04 | [Backend Testing](phases/04-backend-testing.md) | Backend | pytest fixtures, integration tests |
| 05 | [Frontend Foundation](phases/05-frontend-foundation.md) | Frontend | React app skeleton, API client, auth, routing |
| 06 | [Frontend Features](phases/06-frontend-features.md) | Frontend | Feature components, URL design, mobile-first patterns |
| 07 | [Frontend Testing](phases/07-frontend-testing.md) | Frontend | Vitest + React Testing Library setup and patterns |
| 08 | [Docker & Deploy](phases/08-docker-and-deploy.md) | DevOps | Dockerfile, Compose, nginx, deployment (Pi, Azure, or AWS) |
| 09 | [Cross-App Features](phases/09-cross-app-features.md) | Full-stack | GitHub feedback |
| 10 | [Agent Auth & API](phases/10-agent-auth-and-api.md) | Backend | Agent token endpoint, OpenAPI schema exposure |

## Reference Documents

Pull these in during any phase when the app needs the capability:

| Document | When to Use |
|----------|-------------|
| [File Storage](references/file-storage.md) | App stores user-uploaded files or generated assets |
| [Background Workers](references/background-workers.md) | App needs async processing outside the request/response cycle |
| [Google Auth](references/google-auth.md) | App needs Google sign-in and/or mandatory email verification |

## Notes

- Phases 02–04 (backend) and 05–07 (frontend) can run in parallel if two agents work simultaneously
- `review` and `assess` are read-only — they report severity-rated findings (Violation / Gap / Note / Conforms), they don't change code
- When the phase docs change, update [`conventions.md`](conventions.md) so the audit rubric stays in sync
