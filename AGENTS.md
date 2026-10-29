# Global Agent Instructions

Single source of truth for personal, harness-agnostic instructions and skills. OpenCode loads this file for any workspace under `~`. Claude Code reads it through `~/CLAUDE.md`. Repo-specific guidance for the Metal submodules remains in `frontend/AGENTS.md` and `backend/AGENTS.md`.

## Skills

- Skills live in `agents/skills/<skill-name>/SKILL.md` (the Core).
- Harness-specific config points at the Core (the Shell). No copies.
- Create new personal skills only in `~/agents/skills/`. Never create skills in project-level directories.
- `~/.cursor/skills`, `~/.claude/skills`, `~/.agents/skills`, and `~/.config/opencode/skills` are symlinks to the Core. Writes through them land in the Core.
- `~/.codex/skills` is not linked to the Core; it holds Codex-managed system content. For Codex `init_skill.py`, pass `--path ~/agents/skills`.
- Install plugins (e.g. superpowers) natively per harness. Do not share plugin skills through the Core; plugin hooks, updates, and harness-specific workflows are harness-scoped.
- Prefer the smallest relevant set of skills for the task.
- Read only the specific `SKILL.md` files needed for the current request.

## Pull Requests and Git

- Never merge to local `main`. The user merges manually.
- The end-to-end feature workflow runs on the superpowers plugin (plan, implement, review, fix from feedback, push update commits). Ticket and GitHub mechanics come from personal skills in `~/agents/skills`: `scope-feature` (design and spec), `deliver-ticket` (build to merge gate), `run-project` (project runs), `merge-gate`, `test-feature`, `wrap-up`, `linear`, `pr-create`, `pr-code-review`, `pr-plan-review-comments`, and `commit-push-reply-comments`.
- Per-task commits during an approved execution plan: allowed.
- Otherwise, do not make git commits until I review changes.
- Do not push to the remote repository until I grant explicit approval.
- Exception: invoking a skill whose documented workflow includes committing or pushing (for example `commit-push-reply-comments`, `pr-create`) authorizes the commits and pushes that skill describes. The skill's own guardrails apply. No skill authorizes force-pushes, history rewrites, or merges to `main` — merges to `main` are always manual.
- Document the git command for any git operation outside a skill's workflow. I will run it myself.
- Use GitHub CLI when asked to make pull requests. Be succinct in describing the changes across files.
- Whenever linking pull requests with Linear tickets, use the key words "Resolves" or "Closes".
- Example: `Resolves [ENG-4496](<link to linear ticket>)`

Commit message prefixes:

- `fix:` for bug fixes
- `feat:` for new features
- `perf:` for performance improvements
- `docs:` for documentation changes
- `style:` for formatting changes
- `refactor:` for code refactoring
- `test:` for adding missing tests
- `chore:` for maintenance tasks

Commit message rules:

- Use lowercase for commit messages.
- Keep the summary line concise.
- Include a description for non-obvious changes.
- Reference issue numbers when applicable.

## Git Commit Signing

- Create Git commits outside the sandbox with escalated access.
- Use `git commit -S` for new commits and amended commits.
- Do not disable signing with `--no-gpg-sign` or `commit.gpgsign=false`.
- Run GPG signature checks outside the sandbox. The sandbox cannot reliably access `~/.gnupg`.
- After each commit, run `git log -1 --format='%H %G? %GS %s'` with escalated access.
- Require signature status `G` before reporting commit success.
- Before pushing, audit every commit unique to the branch with escalated access.
- Do not push if a branch commit lacks a valid signature.
- Do not rewrite unsigned history unless the user explicitly authorizes the rewrite and force-push.

## Superpowers Planning Files

- All plans, specs, and progress logs go in `docs/superpowers/` at the **project root**: the directory holding `frontend/` and `backend/` when the workspace is a superproject (for example `metal-combined`), or the repository root for a standalone repo (for example `lemmy`, `lemmy 2`, `lemmy 3`). Never inside `frontend/`, `backend/`, or a worktree of them.
- Keep the superpowers working files (`.superpowers/`) at that same project root, so the review ledger and mockups survive worktree removal.
- Do not commit files under `docs/superpowers/`.
- Keep Superpowers design and implementation plan files local and untracked.

## Test Scope

- Never run the full test suite. This applies to frontend and backend.
- Run only targeted tests: the tests named by the approved plan, or tests directly related to the changed files.
- This rule overrides any skill or workflow that requests a full test suite.

# Metal Platform

AI-powered financial research platform with document processing, company enrichment, and intelligent chat assistants.

## Repository Structure

The checkout root contains two git submodules. Paths in this file are relative to that root.

```
.
├── frontend/    # wizard-web (Next.js 15 + React 19)
└── backend/     # metal-backend (Go + Rust + Node.js microservices)
```

Both are git submodules. After cloning, run:
```bash
git submodule update --init --recursive
```

## Tech Stack

### Frontend (wizard-web)
- **Framework**: Next.js 15 with Turbopack, React 19, TypeScript
- **UI**: shadcn/ui + Radix primitives + Tailwind CSS
- **Data Fetching**: SWR for client-side, server components for SSR
- **Tables**: AG Grid
- **Auth**: PropelAuth
- **Analytics**: PostHog, Sentry

### Backend (metal-backend)
- **API Gateway**: Node.js + Fastify (black-sabbath-api, port 8000)
- **Core API**: Go + Gin (metallica-api, port 8081)
- **Document Processing**: Rust (iron-maiden-worker)
- **Event Worker**: Go (danzig-worker)
- **Database**: MongoDB
- **Cache/PubSub**: Redis
- **Vector Search**: Vespa
- **Cloud**: AWS (S3, SQS, SNS, Kinesis, Bedrock, Textract), Azure

## Development Setup

### Prerequisites
- Node.js >= 20.0.0 (check `frontend/.nvmrc`)
- Go 1.24+
- Rust (for iron-maiden-worker)
- Docker & Docker Compose
- Depot CLI (optional, for faster builds)

### Frontend
```bash
cd frontend
npm install
npm run dev          # Start dev server with Turbopack (http://localhost:3000)
```

### Backend
```bash
cd backend
# With Depot (faster):
depot bake -f docker-compose.yaml --load && docker-compose up

# Without Depot:
docker-compose up --build
```

## Common Commands

### Frontend
| Command | Description |
|---------|-------------|
| `npm run dev` | Start dev server with Turbopack |
| `npm run build` | Production build |
| `npm run test` | Run Jest tests |
| `npm run lint` | Run ESLint |
| `npm run check` | TypeScript type check |
| `npm run format` | Format with Prettier |

### Backend
| Command | Description |
|---------|-------------|
| `docker-compose up` | Start all services |
| `cd services/metallica-api && go test ./...` | Run Go API tests |
| `cd services/black-sabbath-api && npm test` | Run Node gateway tests |
| `cd services/iron-maiden-worker/worker && cargo test` | Run Rust worker tests |

## Architecture Overview

### API Flow
```
Browser → Frontend (Next.js :3000)
              ↓
         API Gateway (black-sabbath-api :8000)
              ↓
         Core API (metallica-api :8081)
              ↓
    ┌─────────┼─────────┐
    ↓         ↓         ↓
 MongoDB    Redis    Vespa
```

### Event Processing
```
API → SNS/SQS → danzig-worker (Go)
              → iron-maiden-worker (Rust) → S3/Embeddings
```

### Authentication
- PropelAuth handles auth, sets HTTP-only `token` cookie
- Frontend reads cookie for server-side requests
- Client-side uses SWR with `credentials: "include"`

## Key Directories

### Frontend (`frontend/src/`)
- `app/` - Next.js App Router pages and layouts
- `components/` - 150+ React components (shadcn/ui in `ui/`)
- `lib/mobile-constants.ts` - Shared mobile safe-area padding constants (headers & footers)
- `lib/` - Utilities (`request.ts` for API calls, `server-request.ts` for SSR)
- `hooks/` - 40+ custom React hooks
- `providers/` - Context providers (auth, feature flags, notifications)
- `types/` - TypeScript type definitions
- `lib/errors` - error handling patterns

### Backend
- `services/metallica-api/controllers/` - 120+ API endpoint handlers
- `services/metallica-api/models/` - MongoDB document schemas
- `services/metallica-api/services/` - Business logic layer
- `services/black-sabbath-api/routes/` - API gateway routes
- `lib/go/` - Shared Go packages (LLM clients, search, agents)
- `prompts/` - AI prompt templates and tool schemas

## Environment Variables

### Frontend (`.env.local`)
```env
NEXT_PUBLIC_SERVER_URL=http://localhost:8000   # Backend API
NEXT_PUBLIC_AUTH_URL=<propelauth-url>
```

### Backend
See `backend/docker-compose.yaml` for required env vars. Key ones:
- MongoDB, Redis, Vespa connection strings
- AWS credentials (S3, SQS, Bedrock)
- LLM API keys (OpenAI, Anthropic, Azure)

## Known Issues & Don'ts

<!-- Add mistakes Claude makes here so it learns not to repeat them -->
<!-- Example: "Don't use useState for server data - use SWR instead" -->

## Code Conventions

### Frontend
- TypeScript strict mode
- Prettier: 2 spaces, 80 char max line width
- Path alias: `@/*` → `src/*`
- Components: PascalCase files, named exports
- Hooks: `use-*.ts` with camelCase

### Backend (Go)
- `gofmt` for formatting
- `golangci-lint` for linting
- Google Wire for dependency injection
- Controllers return Gin handlers
- Models use MongoDB struct tags

## Testing Commands

Run targeted test files only. Never run the full suite. See Test Scope.

### Frontend
```bash
cd frontend
npm run test           # Run all tests
npm run test:watch     # Watch mode
```
- Jest + React Testing Library
- happy-dom for lightweight DOM

### Backend
```bash
# Go services
cd backend/services/metallica-api && go test ./...

# Integration tests
cd backend/tests/integration && docker-compose up
```

## AI/LLM Integration

The backend supports multiple LLM providers:
- OpenAI (GPT-4)
- Anthropic (Claude)
- Google (Gemini)
- Azure OpenAI
- AWS Bedrock

Agent orchestration via custom `gent` framework in `lib/go/gent/`.
Prompt templates stored in `backend/prompts/`.

## MCP Integrations

This repo has MCP servers configured for team tools. Use `/mcp` to authenticate.

### Linear
Create, search, and update issues directly:
- View assigned issues and priorities
- Create issues from bug reports or feature requests
- Update status, add comments, assign users

## Common Tasks

### Adding a new API endpoint
1. Create controller in `backend/services/metallica-api/controllers/`
2. Add route in `backend/services/metallica-api/server/router.go`
3. Add proxy route in `backend/services/black-sabbath-api/routes/`
4. Create types in `frontend/src/types/`
5. Add API call in `frontend/src/lib/request.ts`

### Adding a new React component
1. Create in `frontend/src/components/`
2. Use shadcn/ui primitives from `components/ui/`
3. Follow naming: PascalCase for components, kebab-case for files

### Adding a new database model
1. Create model in `backend/services/metallica-api/models/`
2. Add indexes in the model's `init()` function
3. Create service layer in `backend/services/metallica-api/services/`

## Useful Links

- Frontend README: `frontend/README.md`
- Backend README: `backend/README.md`
- Chat Architecture: `backend/docs/backend-chat-architecture.md`
- Ingestion Pipeline: `backend/docs/ingestion-pipeline.md`

## Repository scope

The top-level repository stays on `main`. Never create new branches in the root directory.

When changing the frontend, work on a branch in the frontend repository. Use `cd frontend`.

When changing the backend, work on a branch in the backend repository. Use `cd backend`.

# Notes

- whenever considering access to data entities, always consider user permissions
- Repo-specific code and workflow guidance for the submodules remains in `frontend/AGENTS.md` and `backend/AGENTS.md`.

# TypeScript

When creating typescript interfaces, prefer required fields instead of conditional fields if the field is available at all callsites. If a field is not available at all callsites, document where the field is not available.

Avoid using `any` unless absolutely necessary. Prefer to use `unknown`.

Use TypeScript for all frontend code. Prefer interfaces over types.

Write concise, technical code with accurate examples.

Use functional components with TypeScript interfaces.

Use explicit return types for all functions.

Use the function keyword for pure functions.

Avoid unnecessary curly braces in conditionals.

Use declarative JSX.

Implement proper TypeScript discriminated unions for message types.

When a function accepts more than 2 parameters, use a single params object (interface or inline type) instead of ordered positional arguments.

- 2 or fewer params: positional is fine.
- 3 or more params: use a params object with an explicit interface.

Exceptions:

- React component props (already an object via JSX).
- Callbacks or event handlers where a library dictates the signature (e.g. Express middleware `(req, res, next)`).
- Simple utility functions where all params have different, unambiguous types (e.g. `clamp(value, min, max)`).

```typescript
interface BuildPayloadParams {
  file: IntegrationFile;
  siteId: string;
  folderPath: string;
  connectionId: string;
  context: SyncContext;
}

function buildPayload(params: BuildPayloadParams): Payload {}
```

# React

Always follow the rule of hooks when working with React.

Avoid editing arrays in place that are used for component state or component props.

Where performant, wrap functions defined inside of components in `useCallback` or `useMemo`.

When making updates to SWR data, use optimistic updates and the right mutation calls to update the SWR cache.

# Coding patterns

- Never delete commented code blocks without asking first.
- Prefer npm over pnpm or yarn.
- Always prefer simple solutions.
- Avoid duplication. Check for similar code before adding new logic.
- Write code that accounts for dev, staging, and prod.
- Only make changes that are requested or clearly related.
- When fixing a bug, exhaust the existing implementation first. If you introduce a new pattern, remove the old one.
- Keep the codebase clean and organized.
- Avoid scripts in files when the script is likely to run once.
- Prefer iteration and modularization over duplication.
- Mock data only in tests. Never mock data for dev or prod.
- Never add stubbing or fake data in code that affects dev or prod.
- Never overwrite `.env` files without asking and confirming.
- Never add or remove dependencies without asking and confirming.
- Never add or remove environment variables without asking and confirming.
- Use descriptive variable names with auxiliary verbs (e.g. `isLoading`, `hasError`).
- Handle edge cases.
- When a best practice is used during generation, suggest an addition to `AGENTS.md`.
- Prioritize simplicity and cleanliness in generated code.

# Coding workflow

- Focus on the areas of code relevant to the task.
- Do not touch code that is unrelated to the task.
- Write thorough tests for all major functionality.
- Avoid major changes to patterns after a feature works, unless explicitly structured.
- Always think about other methods and areas of code that a change might affect.
- Every time you apply a rule, state the rule in the output. You can abbreviate it.

# UI and styling

- Use Shadcn UI and Radix for components.
- Use `npx shadcn@latest add <component_name>` to add new Shadcn components.
- Implement Tailwind CSS v4.1 for styling.
- When adding new Shadcn components, document the installation command.

# Performance

- Minimize bundle size using code splitting.
- Implement proper lazy loading for non-critical components.
- Optimize content script injection.
- Use proper caching strategies.
- Implement proper cleanup for event listeners and observers.

# Testing

- Write unit tests for utilities and components.
- Implement E2E tests for critical flows.
- Test across Chrome, Safari, and Firefox.
- Test memory usage and performance.

# Security

- Implement Content Security Policy.
- Sanitize user inputs.
- Handle sensitive data properly.
- Follow security and privacy best practices.
- Implement proper CORS handling.

# Documentation

- Maintain a clear README with setup instructions.
- Document API interactions and data flows.
- Keep manifest.json well-documented.
- Include comments for complex logic.
- Document permission requirements.

# Development workflow

- Use proper version control.
- Implement a proper code review process.
- Test in multiple environments.
- Follow semantic versioning for releases.
- Maintain a changelog.

# General
- Avoid deleting relevant comments
- Document decisions with succinct inline comments
- Keep comments succinct and inline; stay comprehensive

## Avoid Temporal Language
Avoid temporal or change-history framing in docs, comments, SKILL.md files, reference material, and explanatory text unless the user explicitly asks for migration/history context.
Write behavior in present-tense, state-based terms. Describe what is true, required, supported, or expected without contrasting it with prior behavior.
Avoid words and phrases like: currently, now, still, no longer, used to, previously, formerly, historically, legacy, old behavior, new behavior, before, after, instead.
Prefer:

"The tool supports X."
"Use Y when Z."
"If the target is unknown, ask."
"The schema must include ..."
Avoid:

"Currently the tool supports X."
"This used to do A, but now does B."
"The old behavior was ..."
"If the target is still unknown ..."
Exception: temporal language is allowed only when it is necessary for changelogs, migration notes, deprecations, incident timelines, or genuinely date-sensitive behavior.

# Communication Style

Write all text in Simplified Technical English (adapted from ASD-STE100). This covers chat responses, explanations, plans, summaries, commit messages, and PR descriptions.

Rules:
- One word, one meaning. Do not swap in a synonym for variety.
- Short sentences. Keep each sentence under about 20 words.
- Active voice. Write "the handler validates the token", not "the token is validated by the handler".
- Short paragraphs. Cover one topic per paragraph.
- No hedging or filler. Cut phrases like "it's worth noting that", "in order to", "essentially", "basically".
- Keep necessary technical terms (library names, APIs, domain vocabulary). The goal is clarity, not a restricted dictionary.

This does not change commit message format conventions (e.g. Conventional Commits prefixes like `feat:`, `fix:`) or required structured output (JSON, code, diffs). It governs the prose around them.
