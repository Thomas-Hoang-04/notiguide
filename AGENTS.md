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
- The frontend should be written in a way that is easy to understand and maintain.
- The frontend UI should be modern, coherent to current design (refer strictly to `docs/walkthrough/Web Styles.md`), with breathability.
- Only use the UI components existing in the project. Do not create new UI components. Only crawl new components from shadcn/ui if necessary.

### Vietnamese copy rules (i18n)

`vi.json` must read like text written by a native Vietnamese speaker, not a translation of `en.json`. Treat the EN copy as the source of meaning, never the source of phrasing.

- **Don't over-qualify nouns.** Use plain `cookie` — not `cookie phiên`, not `một cookie`. Drop the determiner `một` ("a/one") unless cardinality matters; Vietnamese rarely requires it.
- **Action confirmations stay terse.** Don't add the verb back when context implies it: `Đã xong`, not `Đã bật xong`.
- **Avoid `bấm` for committing a primary action button.** Prefer `xác nhận` (e.g., `Xác nhận Cho phép`). `bấm` is acceptable for taps on icons/links.
- **Avoid `hộp xác nhận`** — sounds technical. Describe behavior instead (e.g., `trình duyệt sẽ hỏi quyền truy cập`).
- **Prefer `kích hoạt` over `bật`** for "enable/turn on" in feature/permission contexts.
- **Don't frame requests as "you change your browser, we don't change anything."** In a security/permission context users may misread it as evasive. Focus on the user's choice, scope (this site only), and reversibility instead.

**Structural mirroring (both directions):** `en.json` and `vi.json` must mirror each other structurally — same keys, same number of paragraphs (`<p>` blocks), same rich-text tags (`<bold>`, `<shield>`, etc.) in equivalent positions. When you change one side's structure, change the other immediately. Wording is per-language; structure must match.

When unsure about Vietnamese phrasing, propose the copy in plain text in chat and get approval before writing to `vi.json`. Do not iterate edits blindly.

<!-- gitnexus:start -->
# GitNexus — Code Intelligence

This project is indexed by GitNexus as **notiguide** (14505 symbols, 25760 relationships, 300 execution flows). Use the GitNexus MCP tools to understand code, assess impact, and navigate safely.

> If any GitNexus tool warns the index is stale, run `npx gitnexus analyze` in terminal first.

## Always Do

- **MUST run impact analysis before editing any symbol.** Before modifying a function, class, or method, run `gitnexus_impact({target: "symbolName", direction: "upstream"})` and report the blast radius (direct callers, affected processes, risk level) to the user.
- **MUST run `gitnexus_detect_changes()` before committing** to verify your changes only affect expected symbols and execution flows.
- **MUST warn the user** if impact analysis returns HIGH or CRITICAL risk before proceeding with edits.
- When exploring unfamiliar code, use `gitnexus_query({query: "concept"})` to find execution flows instead of grepping. It returns process-grouped results ranked by relevance.
- When you need full context on a specific symbol — callers, callees, which execution flows it participates in — use `gitnexus_context({name: "symbolName"})`.

## When Debugging

1. `gitnexus_query({query: "<error or symptom>"})` — find execution flows related to the issue
2. `gitnexus_context({name: "<suspect function>"})` — see all callers, callees, and process participation
3. `READ gitnexus://repo/notiguide/process/{processName}` — trace the full execution flow step by step
4. For regressions: `gitnexus_detect_changes({scope: "compare", base_ref: "main"})` — see what your branch changed

## When Refactoring

- **Renaming**: MUST use `gitnexus_rename({symbol_name: "old", new_name: "new", dry_run: true})` first. Review the preview — graph edits are safe, text_search edits need manual review. Then run with `dry_run: false`.
- **Extracting/Splitting**: MUST run `gitnexus_context({name: "target"})` to see all incoming/outgoing refs, then `gitnexus_impact({target: "target", direction: "upstream"})` to find all external callers before moving code.
- After any refactor: run `gitnexus_detect_changes({scope: "all"})` to verify only expected files changed.

## Never Do

- NEVER edit a function, class, or method without first running `gitnexus_impact` on it.
- NEVER ignore HIGH or CRITICAL risk warnings from impact analysis.
- NEVER rename symbols with find-and-replace — use `gitnexus_rename` which understands the call graph.
- NEVER commit changes without running `gitnexus_detect_changes()` to check affected scope.

## Tools Quick Reference

| Tool | When to use | Command |
|------|-------------|---------|
| `query` | Find code by concept | `gitnexus_query({query: "auth validation"})` |
| `context` | 360-degree view of one symbol | `gitnexus_context({name: "validateUser"})` |
| `impact` | Blast radius before editing | `gitnexus_impact({target: "X", direction: "upstream"})` |
| `detect_changes` | Pre-commit scope check | `gitnexus_detect_changes({scope: "staged"})` |
| `rename` | Safe multi-file rename | `gitnexus_rename({symbol_name: "old", new_name: "new", dry_run: true})` |
| `cypher` | Custom graph queries | `gitnexus_cypher({query: "MATCH ..."})` |

## Impact Risk Levels

| Depth | Meaning | Action |
|-------|---------|--------|
| d=1 | WILL BREAK — direct callers/importers | MUST update these |
| d=2 | LIKELY AFFECTED — indirect deps | Should test |
| d=3 | MAY NEED TESTING — transitive | Test if critical path |

## Resources

| Resource | Use for |
|----------|---------|
| `gitnexus://repo/notiguide/context` | Codebase overview, check index freshness |
| `gitnexus://repo/notiguide/clusters` | All functional areas |
| `gitnexus://repo/notiguide/processes` | All execution flows |
| `gitnexus://repo/notiguide/process/{name}` | Step-by-step execution trace |

## Self-Check Before Finishing

Before completing any code modification task, verify:
1. `gitnexus_impact` was run for all modified symbols
2. No HIGH/CRITICAL risk warnings were ignored
3. `gitnexus_detect_changes()` confirms changes match expected scope
4. All d=1 (WILL BREAK) dependents were updated

## Keeping the Index Fresh

After committing code changes, the GitNexus index becomes stale. Re-run analyze to update it:

```bash
npx gitnexus analyze
```

If the index previously included embeddings, preserve them by adding `--embeddings`:

```bash
npx gitnexus analyze --embeddings
```

To check whether embeddings exist, inspect `.gitnexus/meta.json` — the `stats.embeddings` field shows the count (0 means no embeddings). **Running analyze without `--embeddings` will delete any previously generated embeddings.**

> Claude Code users: A PostToolUse hook handles this automatically after `git commit` and `git merge`.

## CLI

| Task | Read this skill file |
|------|---------------------|
| Understand architecture / "How does X work?" | `.claude/skills/gitnexus/gitnexus-exploring/SKILL.md` |
| Blast radius / "What breaks if I change X?" | `.claude/skills/gitnexus/gitnexus-impact-analysis/SKILL.md` |
| Trace bugs / "Why is X failing?" | `.claude/skills/gitnexus/gitnexus-debugging/SKILL.md` |
| Rename / extract / split / refactor | `.claude/skills/gitnexus/gitnexus-refactoring/SKILL.md` |
| Tools, resources, schema reference | `.claude/skills/gitnexus/gitnexus-guide/SKILL.md` |
| Index, status, clean, wiki CLI commands | `.claude/skills/gitnexus/gitnexus-cli/SKILL.md` |

<!-- gitnexus:end -->
