# Repository Guidelines

## Project Structure & Module Organization

This repository is in the planning stage. `README.md` introduces the distributed-systems laboratory, `docs/project-plan.md` defines the proposed architecture and milestones, and `LICENSE` contains the MIT license. No source, test, or asset directories exist yet.

The proposed stack is React, TypeScript, and Vite for presentation; Python FastAPI for experiment control; and Docker Compose with RabbitMQ, PostgreSQL, and Python workers for the first experiment. Treat these as planned choices, with FastAPI provisional. Document new directory layouts and entry points in the README when implementation begins.

## Build, Test, and Development Commands

No build scripts, dependency manifests, test runners, or local startup commands are configured yet. Useful checks from the repository root are:

- `git status --short`: inspect pending changes.
- `git diff --check`: catch whitespace errors in tracked changes.
- `git diff -- docs/ README.md`: review documentation changes.

When adding executable components, document exact installation, startup, test, and teardown commands alongside their configuration. Do not assume `npm test` or `docker compose up` works before the corresponding files exist.

## Coding Style & Naming Conventions

Use descriptive Markdown headings, concise explanations, fenced command examples, and relative links. Follow the existing lowercase, hyphenated documentation naming pattern, such as `docs/project-plan.md`.

No code formatter, linter, or indentation standard is configured. Establish language-specific conventions and tool configuration with the first implementation; keep subsequent changes consistent with them.

## Testing Guidelines

No testing framework or coverage threshold exists yet. The first vertical slice should verify normal delivery, worker failure before commit, failure after commit before acknowledgment, redelivery, and transactional idempotency. Assert observable deliveries and committed business effects. Document reproducible setup, reset, execution, and expected outcomes. Define test filenames and runner commands when selecting frameworks.

## Commit & Pull Request Guidelines

Git history currently contains only `Initial commit`; no recurring commit convention is established. Use concise, imperative subjects and keep changes focused.

Pull requests should explain the problem, resulting behavior, validation performed, and relevant issues or plan milestones. Include screenshots for interface changes and reproduction steps for experiments. Update documentation when behavior or setup changes.

## Experiment Design

Prioritize one complete message-delivery experiment before expanding scope. Record hypotheses, measurements, tradeoffs, and limitations. Clearly distinguish real observations, injected faults, and simulations. Capture consequential architectural choices in lightweight decision records.
