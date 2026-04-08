# Agent Instructions

## Overview
This project was scaffolded with `create-allthing`. AI agents (Copilot, Claude, etc.)
should read this document before making changes.

## Project structure
```
.
├── frontend/       React + Vite SPA
├── backend/        API server
├── infra/          Infrastructure as code (docker-compose, scripts, db)
├── mcp/            Model Context Protocol tools
├── docs/           Specifications and architecture docs
└── .claude/        AI agent skills and config
```

## Rules
1. Never commit secrets — use `.env` (gitignored) and `.env.example`.
2. All public API routes must have a corresponding integration test.
3. Docker Compose is the single source of truth for local dev.
4. Prefer small, focused PRs.

## Skills available
Run `npx skills list` to see all registered skills for this project.
