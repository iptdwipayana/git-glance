# CLAUDE.md

Guidance for Claude Code when working in this repository.

## Project

**Git Glance** — a web app to search GitHub users and repositories by `username` or
`username/repository`. Personal learning project: the goal is to learn the stack below and
practice agent-assisted engineering, so **clarity and correctness matter more than speed**.

## Working agreement

1. **Plan before implementing.** For anything beyond a one-file edit, state the plan (files to
   touch, layer placement, public API) and wait for approval.
2. **Never add a dependency without asking.** Name the package, why it's needed, and the
   alternative that avoids it.
3. **Never run `git commit` unless explicitly asked.** Staging + committing on request is fine;
   committing on your own initiative is not.
4. **Ask instead of assuming.** If a requirement is ambiguous (naming, UX behaviour, error
   states), ask one focused question rather than picking a direction silently.
5. **Don't create files that weren't asked for** — no extra READMEs, no summary docs, no
   `.example` scaffolding.
6. **Report uncertainty.** If something wasn't verified (a type checks, a test passes), say so
   instead of implying it was.

## Commands

Package manager is **pnpm**. Do not use `npm` or `yarn`.

```bash
pnpm dev              # start dev server
pnpm build            # typecheck + production build
pnpm preview          # serve the production build

pnpm test             # vitest, single run
pnpm test:watch       # vitest, watch mode
pnpm test path/to/file.test.ts   # single file

pnpm check            # biome check (lint + format, read-only)
pnpm check:fix        # biome check --write
pnpm typecheck        # tsc --noEmit
```

Before saying work is done, run `pnpm check` and `pnpm typecheck`, plus `pnpm test` if any
tested code changed.

## Tech stack

| Concern      | Choice                                    |
| ------------ | ----------------------------------------- |
| Language     | TypeScript (strict)                       |
| UI           | React                                     |
| Routing      | TanStack Router (file-based routes)       |
| Server state | TanStack Query                            |
| Styling      | Tailwind CSS                              |
| Components   | shadcn/ui                                 |
| Testing      | Vitest                                    |
| Lint/format  | Biome                                     |
| Git hooks    | Husky + lint-staged + commitlint          |
| Environment  | Dev Container                             |

## Architecture

Layered architecture, applied **per feature**. Each feature owns its four layers.

```
src/
├─ app/                        # providers, router bootstrap, global styles
├─ routes/                     # TanStack Router file routes — thin, no logic
├─ features/
│  └─ <feature-name>/
│     ├─ domain/               # types, schemas, pure rules. No React, no fetch.
│     ├─ application/          # use-case hooks (TanStack Query), orchestration
│     ├─ infrastructure/       # HTTP clients, GitHub adapters, DTO → domain mappers
│     └─ presentation/         # components + views for this feature
├─ shared/
│  ├─ ui/                      # shadcn/ui components (generated, then owned by us)
│  ├─ lib/                     # cross-feature utilities
│  ├─ hooks/                   # cross-feature hooks
│  └─ config/                  # env, constants, query client
└─ tests/                      # all test files, mirroring the paths above
```

**Dependency direction — enforce this strictly:**

```
presentation → application → domain
                   ↓
             infrastructure → domain
```

- `domain` imports nothing from the other layers. It must be testable without React or network.
- `infrastructure` maps GitHub API responses into domain types. Raw API shapes never leak upward.
- `application` exposes hooks (`useUserSearch`, `useRepository`) — this is the only surface
  presentation is allowed to touch.
- `presentation` contains no `fetch`, no query keys, no API types.
- Features do not import from each other. Shared code moves to `src/shared/`.
- A feature stays flat until it needs a layer — don't create empty `domain/` folders.

Route files should do routing only: params, loaders, and rendering a presentation component.

## GitHub API

Unauthenticated REST (`https://api.github.com`). This is a hard constraint on design:

- **60 requests/hour per IP** for core REST; **10 requests/minute** for search endpoints.
- Debounce search input (~400ms) and never fire a query on every keystroke.
- Set generous `staleTime` (5+ minutes) so navigation reuses cache instead of refetching.
- Handle `403` with rate-limit headers as a distinct, user-visible state — not a generic error.
- Handle `404` (no such user/repo) separately from network failure.
- Never add a token to client code, `.env`, or committed files.

## Conventions

**TypeScript**

- `strict: true`. `any` is not allowed — use `unknown` and narrow.
- Prefer `type` for unions/props, `interface` for extendable object contracts.
- Named exports everywhere except TanStack Router route files.
- Validate external data (GitHub responses) at the infrastructure boundary before it becomes a
  domain type.

**React**

- Function components only. Props typed inline as `type Props = {...}`.
- Derive state during render rather than syncing it in `useEffect`.
- Colocate: a component used by one feature lives in that feature's `presentation/`.

**TanStack Query**

- Query keys are defined in the feature's `application/` layer as a `const` factory, never
  inlined at the call site.
- No `useEffect`-based data fetching anywhere.

**Styling**

- Tailwind utilities in JSX; no separate CSS files beyond globals.
- Reach for a shadcn/ui primitive before writing a custom component.
- Compose class names with the project's `cn()` helper.

**Naming**

- Files and folders: `kebab-case`. Components: `PascalCase`. Hooks: `useCamelCase`.
- Test files live under `src/tests/`, mirroring the path of their subject — never beside the
  file they test. `src/features/user-search/domain/parse-query.ts` is tested by
  `src/tests/features/user-search/domain/parse-query.test.ts`.

## Testing

Vitest. Test **hooks, utilities, mappers, and business logic** — not presentational components,
unless asked.

- Cover the domain layer and infrastructure mappers thoroughly; they're pure and cheap to test.
- Mock at the network boundary (MSW or a stubbed client), never mock TanStack Query itself.
- Test behaviour and edge cases (empty results, rate limit, 404), not implementation details.
- New utility or hook ⇒ write its test in the same change.

## Git workflow

**Branches**

```
main                    # production
develop                 # integration
feature/GG-123-short-description
chore/GG-123-short-description
refactor/GG-123-short-description
fix/GG-123-short-description
release/vX.X.X
```

Work branches off `develop`. Never commit directly to `main` or `develop`.

**Commits** — smart commits, validated by commitlint:

```txt
type(scope): summary

GG-123 #comment detailed message of what changes #in-review

<footer>
```

- The ticket ID in the body is the **Subtask** ID, not the branch's ID. Use it when the work sits
  under a Story or a Task as a Subtask; ask if the Subtask isn't obvious from context.
- When the work is a **Task directly under an Epic**, that Task is already the branch — omit the
  ticket ID from the body entirely and write the detail as plain prose (smart-commit directives
  like `#comment` need an issue key, so they're dropped too).
- `type` follows conventional commits: `feat`, `fix`, `chore`, `refactor`, `test`, `docs`, `style`, `ci`.
- `scope` is the feature or area touched (e.g. `search`, `repo-detail`, `ci`).
- Summary: imperative mood, lowercase, no trailing period.
- Never use `--no-verify`. If a hook fails, fix the cause.
- No Claude/AI attribution lines in commit messages.

## Never

- Add a dependency, commit, or push without being asked.
- Put `fetch` calls or API types in `presentation/`.
- Import across features.
- Introduce a state management library — TanStack Query covers server state.
- Ship code that hasn't passed `pnpm check` and `pnpm typecheck`.

---

_The architecture section reflects intended structure; update it here whenever the real layout
diverges._