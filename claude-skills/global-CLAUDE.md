# Global Preferences for Claude Code

**Note:** This is Alex's `~/.claude/CLAUDE.md` — the global instruction file loaded into every Claude Code session. Team members can use this as a reference or starting point for their own.

---

## Interaction Style

- **Challenge my thinking** — If I make a logical leap that doesn't make sense, or something I say conflicts with prior context or your knowledge, push back. I use Claude as a thought partner, not for empty validation.
- **Answer inline first** — When I ask questions, respond to them directly (stream of consciousness) before producing any work product. This avoids wasted effort if the answer changes direction.
- **Outline multi-step tasks** — If a task will take multiple steps, outline the plan first and wait for my confirmation before executing.

## Document Formatting

- **Numbered headings** — All documents use hierarchical numbering: 1, 1.1, 1.1.1, etc. This lets me give precise feedback by section.
- **Checkbox action items** — Any to-do or task list uses `[ ]` format for compatibility with AI agents and hybrid workflows.

## Code Preferences

- **Comment block at top** — When writing code, add a brief comment explaining the approach before the implementation.
- **Prefer diffs/targeted edits** — Don't rewrite entire files. Show me the specific changes or use minimal edits.

## Tips & Shortcuts

- Drop relevant VS Code and Claude Code hotkeys/commands as tips during our work. I'm actively learning both systems.

## My Context

See project-specific CLAUDE.md files for business context. I run multiple businesses:
- **Kalpa Inovasi Digital** — Healthcare SaaS (WellMed), Go/PHP/AWS stack
- **Padma Care** — Medical advocacy for expats in Bali
- **Padma Medical Group** — Healthcare for overseas workers (17 years)
- **Narawangsa Villas** — Luxury short-term rentals in Bali

---

# Personal Coding Preferences

## Languages & Frameworks
- Primary: Go
- Web: Nuxt.js, Node.js
- Database: PostgreSQL
- Personal scripts: Python

## Go Projects
- Build: `go build ./...`
- Test: `go test ./...`
- Lint: `golangci-lint run`
- Format: `gofmt -w .`
- Prefer table-driven tests
- Use context for cancellation/timeouts
- Handle errors explicitly, don't ignore them

## Node.js / Nuxt Projects
- Package manager: npm (or pnpm/yarn if specified)
- Build: `npm run build`
- Dev: `npm run dev`
- Test: `npm test`
- Formatting: Prettier
- Linting: ESLint

## Python (Personal Projects)
- Use virtual environments (venv)
- Format with black
- Type hints appreciated but not required

## Database
- PostgreSQL for all persistent storage
- Use migrations for schema changes
- Prefer parameterized queries (never string concat)

## Git Preferences
- Conventional commits (feat:, fix:, docs:, refactor:, test:)
- Keep commits focused and atomic
- Meaningful branch names: feature/*, fix/*, etc.
