# Repository Guidelines

## Project Structure & Module Organization
- This workspace has three top-level folders: `backend/`, `web/`, and `docs/`.
- `backend/` contains the Kotlin/Spring Boot API (`src/main/kotlin/com/thomas/notiguide`), with resources in `src/main/resources` and tests in `src/test/kotlin`.
- `web/` contains the Next.js admin app, primarily under `src/app`, with static assets in `public/`.
- `docs/` stores planning notes, walkthroughs, and changelog entries.
- `backend/` and `web/` are separate Git repositories; commit changes in the repo you modified.

## Build, Test, and Development Commands
```bash
# Backend
cd backend
docker compose up -d      # start PostgreSQL + Redis
./gradlew bootRun         # run API locally
./gradlew test            # run backend tests
./gradlew build           # compile + test + package

# Frontend
cd web
yarn dev                  # run Next.js dev server
yarn build                # production build
yarn lint                 # biome check
yarn format               # biome format --write
```

## Coding Style & Naming Conventions
- Kotlin: follow existing style (4-space indentation, `PascalCase` types, `camelCase` members, domain-first packages).
- Prefer coroutine-based handlers (`suspend`) in controllers/services, matching current WebFlux usage.
- Use named imports at file top; avoid fully qualified imports in code.
- TypeScript/React: Biome is authoritative (`web/biome.json`), including 2-space indentation and import organization.
- Next.js App Router files should follow framework naming (`page.tsx`, `layout.tsx`); non-route components use `PascalCase`.

## Testing Guidelines
- Backend testing stack: JUnit 5 + Spring Boot Test + Reactor Test + Kotlin coroutine test libraries.
- Place backend tests in `backend/src/test/kotlin` and name files `*Test.kt`.
- No enforced coverage gate is configured; new backend behavior should include focused unit/integration tests.
- Frontend test tooling is not configured yet; include manual verification steps in PRs for UI changes.

## Commit & Pull Request Guidelines
- Follow Conventional Commit style used in history (for example, `feat: add rate limiter`, `fix: fix Redis connection issue`).
- Keep commits scoped and atomic; avoid mixing backend and frontend refactors in one commit when possible.
- PRs should include: concise summary, linked issue/task, test evidence (`./gradlew test`, `yarn lint`), and screenshots for UI changes.
- Update `docs/CHANGELOGS.md` for every code change merged.

## Security & Configuration Tips
- Keep secrets in environment files (for example `backend/.env`); never commit credentials or private keys.
- Use `backend/compose.yaml` for local infra and `application-dev.yaml` for development-safe configuration.

## Notes
- All file changes should be specify in `docs/CHANGELOGS.md`, even what is skipped from the implementation steps.
- Do not attempt to build anything after implementation. There is a specific audit flow for that.
- Use named imports at the top of the file only. Do not use full qualified imports.
- CSS should be written to seperate files if they are too complex and lengthy. Avoid lengthy Tailwind class inline.
- Do not use any other UI framework than shadcn/ui.
- From then on, all frontend code should be written with bilingual support (English and Vietnamese)
