# CLAUDE.md

This file provides guidance for AI assistants working with the MCP Kotlin SDK codebase.

## Project Overview

This is the official Kotlin SDK for the [Model Context Protocol (MCP)](https://modelcontextprotocol.io/), a protocol enabling AI models to interact with external tools, resources, and prompts. The SDK is a **Kotlin Multiplatform** library targeting JVM, JS, Wasm, macOS, Linux, Windows, and iOS.

- **Group ID:** `io.modelcontextprotocol`
- **Artifact:** `kotlin-sdk`
- **Current version:** `0.6.0`
- **Latest protocol version:** `2025-03-26`
- **Supported protocol versions:** `2025-03-26`, `2024-11-05`

## Repository Structure

```
kotlin-sdk/
├── kotlin-sdk-core/        # Core MCP protocol types and shared transport logic
├── kotlin-sdk-client/      # MCP client implementation and client-side transports
├── kotlin-sdk-server/      # MCP server implementation and server-side transports
├── kotlin-sdk/             # Umbrella module aggregating core + client + server
├── kotlin-sdk-test/        # Integration tests (client/server end-to-end)
├── samples/
│   ├── kotlin-mcp-client/      # Interactive CLI client example
│   ├── kotlin-mcp-server/      # Multiplatform server example
│   └── weather-stdio-server/   # Practical weather server (stdio transport)
├── buildSrc/               # Custom Gradle convention plugins
├── gradle/
│   └── libs.versions.toml  # Centralized dependency version catalog
├── docs/                   # Dokka-generated API documentation (HTML)
└── .github/workflows/      # CI/CD pipelines
```

### Module Responsibilities

| Module | Package | Purpose |
|--------|---------|---------|
| `kotlin-sdk-core` | `io.modelcontextprotocol.kotlin.sdk` | Protocol types, JSON-RPC, shared transport base |
| `kotlin-sdk-client` | `io.modelcontextprotocol.kotlin.sdk.client` | `Client` class, client transports |
| `kotlin-sdk-server` | `io.modelcontextprotocol.kotlin.sdk.server` | `Server` class, server transports |
| `kotlin-sdk` | — | Re-exports all three modules as one artifact |
| `kotlin-sdk-test` | — | Integration tests only, not published |

## Build System

### Requirements

- **JDK 21** (required; set `JAVA_HOME` accordingly)
- Gradle wrapper included (`./gradlew`)

### Common Commands

```bash
# Build all modules (includes tests)
./gradlew build

# Build without tests
./gradlew assemble

# Run tests only
./gradlew test

# Run ktlint checks
./gradlew ktlintCheck

# Auto-fix ktlint violations
./gradlew ktlintFormat

# Check binary API compatibility
./gradlew apiCheck

# Update API dump files after intentional API changes
./gradlew apiDump

# Generate Dokka HTML documentation
./gradlew dokkaGenerate

# Build a specific module
./gradlew :kotlin-sdk-core:build
./gradlew :kotlin-sdk-test:test
```

### Custom Convention Plugins (buildSrc)

- **`mcp.multiplatform.gradle.kts`** — Configures all Kotlin Multiplatform targets (JVM/JS/Wasm/native), enforces `ExplicitApiMode.Strict`, sets JVM target 1.8, toolchain JDK 21, and generates `LibVersion.kt`
- **`mcp.publishing.gradle.kts`** — Maven Central publishing via JReleaser
- **`mcp.dokka.gradle.kts`** — Dokka API doc configuration
- **`mcp.jreleaser.gradle.kts`** — Release automation

### Dependency Management

All versions are centralized in `gradle/libs.versions.toml`. Key versions:

| Dependency | Version |
|-----------|---------|
| Kotlin | 2.2.0 |
| Kotlinx Serialization | 1.9.0 |
| Kotlinx Coroutines | 1.10.2 |
| Kotlinx IO | 0.8.0 |
| Ktor | 3.2.3 |
| AtomicFU | 0.29.0 |

Always update `libs.versions.toml` rather than hardcoding versions in `build.gradle.kts` files.

## Architecture

### Core Abstractions

**`Protocol`** (`kotlin-sdk-core/src/commonMain/.../shared/Protocol.kt`)
- Abstract base class shared by both `Client` and `Server`
- Manages JSON-RPC 2.0 request/response matching, notification dispatch, and timeouts
- Provides `request()` and `notify()` coroutine functions

**`Transport`** (`kotlin-sdk-core/src/commonMain/.../shared/Transport.kt`)
- Interface: `start()`, `send(message)`, `close()`, `onMessage`, `onError`, `onClose`
- `AbstractTransport` provides base buffering and lifecycle management
- Transport implementations are swappable; clients and servers are transport-agnostic

**`Client`** (`kotlin-sdk-client/src/commonMain/.../client/Client.kt`)
- Entry point for connecting to an MCP server
- `connect(transport)` to establish a session
- Methods: `listTools()`, `callTool()`, `listResources()`, `readResource()`, `listPrompts()`, `getPrompt()`, etc.

**`Server`** (`kotlin-sdk-server/src/commonMain/.../server/Server.kt`)
- Entry point for exposing MCP services
- Register handlers: `addTool()`, `addResource()`, `addPrompt()`
- `connect(transport)` to accept a client connection

### Transport Implementations

| Transport | Client | Server | Notes |
|-----------|--------|--------|-------|
| Stdio | `StdioClientTransport` | `StdioServerTransport` | Process stdin/stdout |
| SSE | `SSEClientTransport` | `SSEServerTransport` | Server-Sent Events via Ktor |
| WebSocket | `WebSocketClientTransport` | `WebSocketMcpServerTransport` | via Ktor |
| Streamable HTTP | `StreamableHttpClientTransport` | — | HTTP with streaming |

Ktor integration helpers: `KtorClient.kt` (client) and `KtorServer.kt` (server).

### Protocol Message Flow

1. Caller invokes `client.callTool(...)` (or similar)
2. `Protocol.request()` creates a JSON-RPC request with an auto-incremented `AtomicLong` ID
3. Message serialized via `McpJson` (kotlinx-serialization) and sent over `Transport`
4. Response matched by request ID; deserialized and returned
5. Notifications (no response expected) go through `Protocol.notify()`

### Type System

All MCP protocol types live in `kotlin-sdk-core/src/commonMain/.../sdk/types.kt`:

- `Method` — sealed interface with `Defined` enum (predefined methods) and `Custom` (arbitrary strings)
- Request/Result pairs: e.g., `CallToolRequest` / `CallToolResult`
- Notifications: e.g., `ToolListChangedNotification`
- `ServerCapabilities` / `ClientCapabilities` — feature negotiation during handshake
- `Tool`, `Resource`, `Prompt` — core MCP entities with JSON schema support
- All types are `@Serializable` and use `JsonObject` for extensible metadata via `_meta`

## Code Conventions

### Kotlin Style

- Follow the [Kotlin Coding Conventions](https://kotlinlang.org/docs/reference/coding-conventions.html)
- Code style enforced by **ktlint** (IntelliJ IDEA style, experimental rules enabled)
- Max line length: **120 characters**
- Indentation: **4 spaces** (2 spaces for JSON/YAML)
- No wildcard imports
- All public APIs **must** have explicit visibility modifiers (enforced by `ExplicitApiMode.Strict`)

### KDoc Requirements

Every public API element (class, function, property, parameter, return type) **must** be documented with KDoc. This is a hard requirement for any new public API.

### API Compatibility

The project uses the **Binary Compatibility Validator** plugin. After any intentional public API change:
1. Run `./gradlew apiDump` to regenerate `.api` dump files
2. Commit the updated dump files alongside the code changes

Breaking changes without an updated dump file will fail the `apiCheck` task in CI.

### Multiplatform Guidelines

- All shared logic goes in `commonMain` source sets
- Platform-specific code only when unavoidable (e.g., process I/O on JVM, native interop)
- JVM target is **Java 8** bytecode (broadest compatibility), toolchain is JDK 21
- Tests in `commonTest` run on all platforms; `jvmTest` for JVM-only tests

### Serialization

- Use `kotlinx-serialization` (not Gson, Jackson, or Moshi)
- The shared `McpJson` instance in `kotlin-sdk-core` is the canonical JSON serializer — use it rather than creating new `Json` instances
- Protocol types use `@Serializable` and often `JsonObject` for extensible fields

### Coroutines

- All async operations use Kotlin coroutines (`suspend` functions)
- Use `kotlinx-coroutines-core` for concurrency primitives
- Use `kotlinx.atomicfu` for atomic operations (not `java.util.concurrent.atomic`)
- Prefer `StateFlow`/`MutableStateFlow` for observable state

## Testing

### Structure

- **Unit tests**: `commonTest/` in each module (run on all platforms)
- **JVM-only tests**: `jvmTest/` for platform-specific behavior
- **Integration tests**: `kotlin-sdk-test/` module — full client/server round trips

### Running Tests

```bash
# All tests
./gradlew test

# Specific module
./gradlew :kotlin-sdk-core:test
./gradlew :kotlin-sdk-test:test

# JVM tests only (faster during development)
./gradlew jvmTest
```

### Test Conventions

- Use Kotlin Multiplatform Test (`kotlin.test`)
- Use `kotlinx-coroutines-test` for testing suspend functions (`runTest { ... }`)
- Use Kotest JSON assertions (`io.kotest:kotest-assertions-json`) for JSON comparisons
- Use Ktor's test host (`ktor-server-test-host`) for HTTP/SSE/WebSocket transport tests
- **Bug fixes must include a regression test**
- **New public APIs must include tests**

## CI/CD

### Workflows

| Workflow | Trigger | Purpose |
|----------|---------|---------|
| `validate-pr.yml` | PR to `main`, manual | Full build + test on macOS with JDK 21 |
| `gradle-publish.yml` | Release created | Build, test, publish to Maven Central |
| `codeql.yml` | Schedule/push | Security code analysis |
| `dependabot.yml` | Schedule | Automated dependency updates |

### PR Validation

All PRs must pass `./gradlew clean build` on macOS with JDK 21. Concurrency is configured to cancel in-progress runs on new commits.

## Commit Message Convention

Write commit messages in English, present tense, imperative mood:
- **Good:** `Fix transport reconnect on connection reset`
- **Bad:** `Fixed transport reconnect` / `Fixes transport reconnect`

Reference issue numbers in parentheses: `Fix SSE reconnect logic (#123)`

## Release Process

Releases are triggered by creating a GitHub Release. The `gradle-publish.yml` workflow:
1. Runs full `clean build`
2. Signs artifacts with GPG
3. Publishes to Maven Central via JReleaser (OSSRH credentials required)

## Key Files Reference

| File | Purpose |
|------|---------|
| `gradle/libs.versions.toml` | All dependency versions — edit here, not in build files |
| `buildSrc/src/main/kotlin/mcp.multiplatform.gradle.kts` | Multiplatform target config and `ExplicitApiMode` |
| `kotlin-sdk-core/src/commonMain/.../sdk/types.kt` | All MCP protocol types |
| `kotlin-sdk-core/src/commonMain/.../shared/Protocol.kt` | JSON-RPC request/response engine |
| `kotlin-sdk-core/src/commonMain/.../shared/Transport.kt` | Transport interface |
| `kotlin-sdk-client/src/commonMain/.../client/Client.kt` | MCP Client entry point |
| `kotlin-sdk-server/src/commonMain/.../server/Server.kt` | MCP Server entry point |
| `.editorconfig` | Editor formatting rules |
| `**/api/*.api` | Binary compatibility dump files — update with `apiDump` after API changes |
