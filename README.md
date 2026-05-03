# Codemote

Codemote is a personal remote development system for controlling a local Mac agent from mobile clients.

It lets you start Codex tasks, resume existing Codex threads, run constrained shell commands, inspect task history, and manage the local agent lifecycle from a mobile interface. It can run directly on a trusted local network or through an optional public Relay when you are away from your machine.

This project is currently an early MVP and is designed as a self-hosted developer tool, not a hosted SaaS product.

## How It Works

Codemote is split into four main parts:

- **Desktop Manager**: an Electron app that owns the normal Mac Agent lifecycle. It starts, stops, restarts, configures, and monitors the local agent.
- **Mac Agent**: a local Node.js process that exposes WebSocket and Ops APIs, runs Codex tasks, runs allowlisted shell commands, stores task history, and connects outbound to the Relay when remote mode is enabled.
- **Mobile App**: an Expo React Native client for connecting to the agent, browsing workspaces, starting or resuming Codex threads, reviewing logs, and managing tasks.
- **Relay**: an optional HTTPS/WSS service for remote access. It authenticates mobile clients with Google OAuth and forwards WebSocket frames between mobile clients and the Mac Agent.

The Relay is the only public internet surface. The Mac Agent does not require inbound ports when remote mode is enabled; it connects to the Relay using outbound WSS.

## Connection Modes

### Local Direct

Use Local Direct mode when the mobile device can reach the Mac over LAN, Tailscale, or another trusted private network.

```text
Mobile client -> ws://<mac-host>:7381/ws -> Mac Agent
```

### Remote Relay

Use Remote Relay mode when the mobile client cannot directly reach the Mac.

```text
Mobile client -> HTTPS/WSS Relay <- outbound WSS <- Mac Agent
```

The Relay authenticates the mobile client and routes frames to the registered Mac Agent. Relay routing is intentionally narrow: it forwards frames and stores audit metadata, but it is not intended to store prompts, command text, or task output.

## Security Model

Codemote is built around a few important constraints:

- The Desktop Manager owns the local Agent lifecycle.
- Remote task working directories must stay under configured `workspaceRoots`.
- Shell execution is constrained by the existing allowlist and policy layer.
- Local tokens and Relay agent secrets are generated per machine and should stay out of git.
- The Relay should preserve WebSocket text/binary frame types while forwarding.
- The Relay should not persist Codex prompts, command text, or task output.

Runtime config and logs live outside the repository under `~/.mda/`.

## Project Layout

```text
apps/agent/      Mac Agent: local WebSocket/Ops APIs, task runners, Relay client
apps/desktop/    Electron Desktop Manager for agent lifecycle and settings
apps/ios/        Expo React Native mobile client
apps/relay/      Optional public HTTPS/WSS Relay
schemas/         Generated Codex app-server TypeScript schema artifacts
docs/            Local planning and implementation notes
```

## Quick Start

Install dependencies:

```bash
pnpm install
```

Generate Codex app-server schemas if needed:

```bash
pnpm generate:codex-schema
```

Set up the local Agent:

```bash
pnpm agent:setup
```

Start the Desktop Manager:

```bash
pnpm desktop:dev
```

Start the mobile app development server:

```bash
pnpm --filter @mda/ios start -- --host lan --port 8081 -c
```

The first Agent setup creates `~/.mda/config.yaml` with a generated local token. The Desktop Manager can switch between Local Direct and Remote Relay mode later.

## Useful Commands

```bash
pnpm agent:doctor
pnpm agent:setup
pnpm agent:start
pnpm desktop:dev
pnpm desktop:build
pnpm relay:dev
pnpm relay:build
pnpm typecheck
pnpm test
```

## Relay Setup

For a self-hosted Relay:

```bash
export RELAY_PUBLIC_BASE_URL=https://relay.example.com
export GOOGLE_CLIENT_ID=...
export GOOGLE_CLIENT_SECRET=...
export GOOGLE_REDIRECT_URI=https://relay.example.com/auth/google/callback
export ALLOWED_GOOGLE_EMAIL=you@example.com
pnpm relay:start
```

Register a Mac Agent for Relay mode:

```bash
pnpm agent:relay-login -- --relay-url https://relay.example.com --workspace-root ~/Documents/source
```

This opens Google login in a browser, creates the Relay agent, and writes the remote-mode config to `~/.mda/config.yaml`.

Example config shape:

```yaml
remoteEnabled: true
relayUrl: https://relay.example.com
agentId: <agent-id>
agentSecret: <agent-secret>
workspaceRoots:
  - ~/Documents/source
```

## Development Status

Codemote is still a personal, early-stage project. The current focus is:

- reliable local Agent lifecycle management through the Desktop Manager
- mobile-first workspace and task flows
- Codex thread listing and resume support
- constrained shell execution
- Relay-based remote access without exposing inbound ports on the Mac

Before using it beyond a private environment, review the security model, workspace root policy, shell command policy, OAuth settings, and deployment configuration.
