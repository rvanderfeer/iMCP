# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Build, lint, test

This is an Xcode project (no SwiftPM root). Use `xcodebuild` against the schemes listed in `iMCP.xcodeproj`.

```bash
# Build the app + bundled CLI (matches CI)
xcodebuild -scheme iMCP -configuration Debug -destination "platform=macOS" build

# Lint (CI runs this with --strict and fails the build on any finding)
swift format lint --strict --recursive .

# Apply formatting in place (config in .swift-format: 4 spaces, 120 col, AlwaysUseLowerCamelCase disabled)
swift format -i -r .

# Run the CLI test target
xcodebuild -scheme imcp-serverTests -destination "platform=macOS" test

# Run a single test
xcodebuild -scheme imcp-serverTests -destination "platform=macOS" \
  test -only-testing:imcp-serverTests/ServiceGroupConfigurationTests/testServiceReturningNormallyExitsGroupCleanly
```

Schemes: `iMCP` (app + bundled `imcp-server`), `imcp-serverTests`. macOS deployment target is 15.1; CI builds on macOS 26 / Xcode 26.0.

Release flow (notarization, signing, GitHub release) lives in `Scripts/release.sh` — see its `print_usage` for subcommands. Don't invent ad-hoc release steps; extend the script.

## Architecture

iMCP is a **two-process system** in one Xcode project:

- **`iMCP` target** (`App/`) — a SwiftUI MenuBarExtra macOS app. Holds all entitlements (Calendar, Contacts, WeatherKit, etc.), owns the UI, and runs the actual tool implementations.
- **`imcp-server` target** (`CLI/main.swift`) — a single-file CLI that MCP clients (Claude Desktop, Cursor, Claude Code) spawn over **stdio transport**. It does *not* implement tools; it's a stdin/stdout ↔ TCP proxy (`StdioProxy` actor) that discovers the running app via Bonjour (`_mcp._tcp`, domain `local.`) and forwards JSON-RPC frames.
- The CLI binary is **copied into the app bundle** at `iMCP.app/Contents/MacOS/imcp-server` (see "Copy Executables" build phase). Both ends advertise/discover the same Bonjour service.

This split exists because MCP stdio clients can't hold persistent macOS permissions — only the app (sandboxed, signed, user-prompted) can. Keep this constraint in mind: don't move tool execution into the CLI.

### Service / Tool model

Every capability is a `Service` (`App/Models/Service.swift`) that exposes a list of `Tool` values built with the `@ToolBuilder` result builder. A `Tool` wraps an async closure returning any `Encodable`; the wrapper re-encodes the result through `JSONEncoder` → `Ontology` (Schema.org JSON-LD) → MCP `Value` (`App/Models/Tool.swift`). Tool results are therefore always JSON-LD documents, not ad-hoc dicts.

`ServiceRegistry` (an `enum` inside `App/Controllers/ServerController.swift` near line 50) is the **single registration point**. Adding a service requires editing it in two places:

1. Append to the `services: [any Service]` array.
2. Add a matching `ServiceConfig` in `configureServices(...)` — this wires the SwiftUI menu icon (SF Symbol name), color, and the `@AppStorage`-backed `Binding<Bool>` toggle.

Forgetting step 2 means the service runs but doesn't appear in the menu bar UI; forgetting step 1 means the toggle exists but no tools are registered.

WeatherKit is gated behind the `WEATHERKIT_AVAILABLE` Swift compilation condition (set on the iMCP target only). Code touching `WeatherService` must be wrapped in `#if WEATHERKIT_AVAILABLE`.

### Service lifecycle pitfall

`CLI/main.swift` configures its `ServiceLifecycle.ServiceGroup` with `successTerminationBehavior: .gracefullyShutdownGroup`. The library default `.cancelGroup` causes a **fatal top-level crash** whenever `MCPService.run()` returns normally (e.g. when the Bonjour connection drops mid-session). This was the bug fixed in commit `15a540d`. Regression coverage lives in `CLITests/ServiceGroupConfigurationTests.swift` — touch this carefully.

### Cross-process integrations

- `App/Integrations/ClaudeDesktop.swift` reads/writes `~/Library/Application Support/Claude/claude_desktop_config.json` to inject the `imcp-server` command into the user's MCP client config. Preserves other servers in the file.
- `App/Services/Messages.swift` accesses `~/Library/Messages/chat.db` via `NSOpenPanel` — the user-granted file URL is what the sandbox uses to allow read access. Message decoding uses the `Madrid` Swift package (Apple `typedstream` reverse-engineered).

### Connection approval

When a new MCP client connects via Bonjour, `ServerController` shows a SwiftUI approval dialog (`ConnectionApprovalView`) and persists the user's choice per-client in `@AppStorage`. The "Always trust this client" flow lives in `ServerController.swift`.

## Working with SourceKit warnings

There is a Cursor rule (`.cursor/rules/ignore-sourcekit-warnings.mdc`) — also applies here: ignore spurious `Cannot find type '____' in scope` / `No such module '___'` diagnostics. They're SourceKit indexer noise; the types and modules do exist. Do **not** try to "fix" them by installing new Swift packages. If a build genuinely fails, trust `xcodebuild` output over the editor's red squiggles.

## Xcode coding assistant config

When using Xcode's built-in Claude coding assistant, its `.claude` directory lives at `~/Library/Developer/Xcode/CodingAssistant/ClaudeAgentConfig/.claude` — that's the path to edit for Xcode-side settings, agents, and hooks (separate from this repo's `.claude/` and from `~/.claude/`).

## Code style

- `.swift-format` enforces 4-space indent, 120-column lines, `lineBreakBeforeEachArgument: true`, trailing commas on multi-element collections, and explicitly disables `AlwaysUseLowerCamelCase` (so `MCP`, `JSON`, etc. are allowed in identifiers).
- CI runs `swift format lint --strict` — any finding fails the build.
