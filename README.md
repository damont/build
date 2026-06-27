# Build Phases

Sequential phases for building a full-stack application. Feed these to an agent one at a time, in order.

## How to Use

1. Start at Phase 01 and work through sequentially
2. Each phase assumes all prior phases are complete
3. Each phase ends with a checklist — verify before moving to the next
4. Pull in **reference documents** (`docs/references/`) when the app needs specific capabilities (file storage, background workers)

## Phases

| Phase | Name | Scope | Description |
|-------|------|-------|-------------|
| 01 | [Project Setup](01-project-setup.md) | Full-stack | Scaffold the project, configure dependencies, choose ports |
| 02 | [Backend Foundation](02-backend-foundation.md) | Backend | FastAPI app, auth system, health check |
| 03 | [Backend Domain](03-backend-domain.md) | Backend | Database models, DTOs, resource routes, business logic |
| 04 | [Backend Testing](04-backend-testing.md) | Backend | pytest fixtures, integration tests |
| 05 | [Frontend Foundation](05-frontend-foundation.md) | Frontend | React app skeleton, API client, auth, routing |
| 06 | [Frontend Features](06-frontend-features.md) | Frontend | Feature components, URL design, mobile-first patterns |
| 07 | [Frontend Testing](07-frontend-testing.md) | Frontend | Vitest + React Testing Library setup and patterns |
| 08 | [Docker & Deploy](08-docker-and-deploy.md) | DevOps | Dockerfile, Compose, nginx, deployment (Pi, Azure, or AWS) |
| 09 | [Cross-App Features](09-cross-app-features.md) | Full-stack | GitHub feedback |
| 10 | [Agent Auth & API](10-agent-auth-and-api.md) | Backend | Agent token endpoint, OpenAPI schema exposure |

## Reference Documents

Pull these in during any phase when the app needs the capability:

| Document | When to Use |
|----------|-------------|
| [File Storage](../references/file-storage.md) | App needs to store user-uploaded files or generated assets |
| [Background Workers](../references/background-workers.md) | App needs async processing outside request/response cycle |

## Notes

- Phases 02-04 (backend) and 05-07 (frontend) can run in parallel if two agents are working simultaneously
- The canonical full reference is `docs/app-architecture-guide.md` — consult it for anything not covered in the phases
