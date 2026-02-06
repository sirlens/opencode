# OpenCode AI Agent Guidelines

This document provides context and instructions for AI agents working on the OpenCode codebase.

## Project Overview

OpenCode is an open-source AI coding agent built with a monorepo structure using Bun and Turborepo. It follows a client-server architecture.

### Key Packages
- `packages/opencode`: Core business logic, headless server (Hono), and TUI (SolidJS + OpenTUI).
- `packages/app`: Web application frontend (SolidJS + Vite).
- `packages/desktop`: Desktop application (Tauri wrapping the web app).
- `packages/ui`: Shared UI components.
- `packages/sdk`: SDK for client-server communication.
- `infra`: Infrastructure code (SST).

## Architecture & Core Logic

### Agents (`packages/opencode/src/agent/`)
- Agents define behavior, prompts, and permissions.
- Modes: `primary`, `subagent`.
- Core agents: `build` (default), `plan` (read-only), `explore` (fast navigation).

### Tools (`packages/opencode/src/tool/`)
- Tools are the primary way agents interact with the system.
- Implementation: Use `Tool.define(name, { description, parameters, execute })`.
- All tool inputs must be validated with Zod schemas.

### Providers (`packages/opencode/src/provider/`)
- LLM integration layer using Vercel AI SDK.
- Agnostic to model providers (Anthropic, OpenAI, Gemini, Bedrock, etc.).

### Server & SDK
- Server: `packages/opencode/src/server/server.ts`.
- **Note**: When modifying server endpoints, run `./script/generate.ts` to regenerate the SDK.

## Development Commands

- **Install**: `bun install`
- **Root Dev**: `bun dev` (runs TUI in `packages/opencode`)
- **Headless Server**: `bun dev serve`
- **Web App**: `bun run --cwd packages/app dev`
- **Desktop App**: `bun run --cwd packages/desktop tauri dev`
- **Typecheck**: `bun run typecheck`
- **Test**: `bun test` (run in `packages/opencode` for core logic)

## Style Guide & Conventions

### General Principles
- **Early Returns**: Avoid `else` statements; use early returns for control flow.
- **Naming**: Prefer single-word concise names (camelCase for variables, PascalCase for namespaces).
- **Functions**: Keep logic in one function unless it's clearly reusable/composable.
- **Async**: Prefer `const foo = await condition ? 1 : 2` over `let`.
- **Types**: Use precise types; avoid `any`. Rely on type inference where possible.
- **Bun APIs**: Use Bun-native APIs (e.g., `Bun.file()`, `Bun.password()`) when applicable.
- **Parallelism**: ALWAYS USE PARALLEL TOOLS WHEN APPLICABLE.

### State Management (Frontend)
- **SolidJS**: Prefer `createStore` over multiple `createSignal` calls for complex state.

## Testing Guidelines
- Avoid mocks; test actual implementations.
- Do not duplicate logic into tests.
- Run `bun test` in the relevant package directory.

## Maintenance
- To regenerate the JavaScript SDK: `./packages/sdk/js/script/build.ts`.
- The default branch is `dev`. Use `dev` or `origin/dev` for diffs.
