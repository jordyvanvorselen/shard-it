# shard-it: Design Documentation

> Framework-Agnostic Test Sharder - Comprehensive Design & Architecture

**Version:** 1.0
**Date:** 2025-11-13
**Status:** Design Phase

---

## Table of Contents

1. [Executive Summary](#executive-summary)
2. [Research Findings](#research-findings)
3. [Public API Design](#public-api-design)
4. [Adapter Protocol Specification](#adapter-protocol-specification)
5. [Programming Language Recommendations](#programming-language-recommendations)
6. [High-Level Architecture](#high-level-architecture)
7. [Implementation Roadmap](#implementation-roadmap)
8. [Distribution Strategy](#distribution-strategy)
9. [Usage Examples](#usage-examples)
10. [Success Criteria](#success-criteria)

---

## Executive Summary

### Vision

Build a **framework-agnostic test sharder** that enables parallel test execution across multiple CI runners or local processes. The tool focuses on defining an elegant public interface first, with architecture evolving from real-world usage patterns.

### Core Principles

- **API-First Design**: The public interface is the contract; implementation details can evolve
- **Framework Agnostic**: Works with JUnit, pytest, Jest, and any test framework via adapters
- **Language-Native Adapters**: Adapters written in the framework's language for maximum compatibility
- **Zero-Config by Default**: Works immediately with sensible defaults, customizable when needed
- **CI/CD Optimized**: First-class integration with GitHub Actions, GitLab CI, CircleCI, etc.

### Key Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| **Core Language** | Rust | Performance (20ms startup), single binary, cross-platform |
| **Adapter Language** | Native to framework | Maximum compatibility, easier contributions |
| **IPC Protocol** | JSON-RPC over stdio | Language-agnostic, structured, versioned |
| **Adapter Discovery** | Explicit `--adapter` flag | Clear, predictable, no surprises |
| **Sharding Strategy** | Round-robin (v1) | Simple, deterministic, good enough to start |
| **Distribution** | Bundled (v1), separate (v2+) | Simple initial setup, scalable later |

### Initial Target

- **Primary Audience**: Java/JUnit 5 projects
- **Integration Level**: Medium (discover tests, filter execution)
- **Timing Optimization**: v2.0 feature (start with count-based splitting)
- **Usage Pattern**: Wrapper around test command

---

## Research Findings

### Existing Test Sharding Solutions

#### Playwright Test Sharding
- **CLI Format**: `npx playwright test --shard=X/Y` (1-based indexing)
- **Approach**: Static, deterministic file-level splitting
- **Granularity**: File-level or test-level (with `fullyParallel: true`)
- **Performance**: Reduces 5m26s to 1m25s with 4 shards
- **Key Insight**: The `--shard=X/Y` format is becoming the de facto standard

#### Jest Test Sharding
- **CLI Format**: `jest --shard=X/Y` (introduced Jest v28)
- **Approach**: Static file-level distribution
- **Limitation**: Incompatible with global coverage thresholds
- **Key Insight**: Simple implementation - collects files, divides by shard count

#### CircleCI Test Splitting
- **CLI Format**: `circleci tests run`
- **Configuration**: `parallelism: N` in config YAML
- **Strategies**: Timing-based (historical data), name-based, file-size-based
- **Environment Variables**: `CIRCLE_NODE_INDEX`, `CIRCLE_NODE_TOTAL`
- **Key Insight**: Dynamic time-based sharding requires historical data but optimally balances loads

#### Knapsack Pro
- **Approach**: SaaS-based dynamic queue mode
- **Architecture**: API tracks timing data, creates optimal splits
- **Modes**: Regular (deterministic) and Queue (dynamic)
- **Key Insight**: Commercial service demonstrates value of timing optimization

#### Bazel Test Sharding
- **Interface**: Environment variables (0-based indexing)
  - `TEST_TOTAL_SHARDS`, `TEST_SHARD_INDEX`
- **Key Insight**: Environment variables are universal for CI integration

#### Key Patterns Identified

1. **CLI Format**: `--shard=X/Y` is emerging standard (Jest, Playwright)
2. **Indexing**: Most tools use 1-based (more intuitive)
3. **Environment Variables**: CI platforms prefer env vars for configuration
4. **Discovery**: Deterministic ordering is critical for reproducibility
5. **Static vs Dynamic**: Start simple (count-based), evolve to time-based
6. **Integration**: Tools either integrate with frameworks or wrap them

### CLI Design Patterns (Modern Developer Tools)

#### Principles from ripgrep, esbuild, Vite

1. **Zero-Config by Default**: Works immediately, no setup required
2. **Progressive Disclosure**: Simple start, progressively reveal advanced features
3. **Context Awareness**: Detects TTY, respects `NO_COLOR`, adapts to environment
4. **Helpful Errors**: Explain what happened and how to fix it
5. **Composability**: Proper stdin/stdout/stderr separation, meaningful exit codes
6. **Human-First**: Readable output by default, `--json` for machines

#### Configuration Hierarchy (Priority Order)

1. Command-line flags (highest priority)
2. Environment variables
3. Project-level config (`.shardit.json`, version-controlled)
4. User-level config (`~/.config/shard-it/config.json`)
5. Built-in defaults (lowest priority)

#### Anti-Patterns to Avoid

- ❌ Silent success without confirmation
- ❌ Cryptic error messages
- ❌ Required configuration before first use
- ❌ Non-standard flag parsing
- ❌ Forced interactivity in non-TTY contexts
- ❌ Mixing data and messages on stdout

### Framework-Agnostic Architecture Patterns

#### Event-Driven Adapter Pattern

Successful tools (Serenity/JS, Wallaby.js) use:

```
Test Framework → Framework Adapter → Unified Event Bus → Core Logic → Output
```

**Components:**
1. **Framework Adapters**: Translate framework-specific APIs to domain events
2. **Plugin Registry**: Auto-detection and loading of adapters
3. **Abstraction Layer**: Unified interface all adapters implement

#### Multi-Language Tool Strategies

**Hybrid Approach** (Recommended):
- **Core Engine**: Rust/Go for CLI, performance-critical paths
- **Language-Specific Adapters**: Native to target framework
- **Communication**: JSON-RPC or MessagePack over stdio

**Example Architecture:**
```
shard-it (Rust binary)
  ├── Spawns: shard-it-junit (Java adapter)
  ├── Spawns: shard-it-pytest (Python adapter)
  └── Spawns: shard-it-jest (Node.js adapter)
```

#### Distribution Patterns

**npm with Optional Dependencies** (esbuild pattern):
```json
{
  "optionalDependencies": {
    "@shard-it/darwin-x64": "^1.0.0",
    "@shard-it/linux-x64": "^1.0.0",
    "@shard-it/win32-x64": "^1.0.0"
  }
}
```

Benefits:
- Automatic platform detection
- Small package size (only downloads relevant binary)
- Works with npx for zero-install usage

---

## Public API Design

### Core CLI Interface

```bash
USAGE:
    shard-it [OPTIONS] -- <COMMAND>

OPTIONS:
    -s, --shard <X/Y>              Shard specification (1-based index/total)
                                   [env: SHARD_INDEX/SHARD_TOTAL]
    -a, --adapter <ADAPTER>        Framework adapter to use (junit, pytest, jest, etc.)
                                   [env: SHARDIT_ADAPTER]
    -p, --pattern <PATTERN>        Test file/class pattern (glob or regex)
                                   [default: adapter-specific]
    -e, --exclude <PATTERN>        Exclusion pattern
    -c, --config <PATH>            Path to config file
                                   [default: .shardit.{json,yaml,toml}]

    --dry-run                      Show shard assignment without running tests
    --list-adapters                List all available adapters
    --adapter-info                 Show detailed adapter information

    --json                         Output in JSON format (for machine consumption)
    --quiet                        Suppress informational output
    --verbose                      Verbose logging (for debugging)

    -h, --help                     Print help information
    -V, --version                  Print version information

ARGS:
    <COMMAND>                      Test command to execute (after --)

ENVIRONMENT:
    SHARD_INDEX                    Shard index (1-based) [overridden by --shard]
    SHARD_TOTAL                    Total number of shards [overridden by --shard]
    SHARDIT_CONFIG                 Path to config file [overridden by --config]
    SHARDIT_ADAPTER                Default adapter [overridden by --adapter]
    NO_COLOR                       Disable colored output
    CI                             Auto-enable CI-friendly output

EXAMPLES:
    # Basic sharding with JUnit
    shard-it --shard=1/4 --adapter=junit -- mvn test

    # CI environment (GitHub Actions)
    shard-it --shard=${{ matrix.shard }} --adapter=junit -- ./gradlew test

    # Preview shard assignment
    shard-it --shard=2/4 --adapter=junit --dry-run

    # Custom test patterns
    shard-it --shard=1/4 --adapter=junit --pattern="**/unit/**/*Test.java" -- mvn test

    # Check available adapters
    shard-it --list-adapters
```

### Configuration File Format

**Location Priority:**
1. `.shardit.json` (project root)
2. `.shardit.yaml` (project root)
3. `.shardit.toml` (project root)
4. Path specified by `--config` or `SHARDIT_CONFIG`
5. `~/.config/shard-it/config.json` (user-level)

**Example Configuration (.shardit.json):**

```json
{
  "$schema": "https://shard-it.dev/schema/v1/config.json",

  "adapter": "junit",

  "discovery": {
    "pattern": "**/*Test.java",
    "exclude": ["**/integration/**", "**/fixtures/**"],
    "sort": "path"
  },

  "shard": {
    "strategy": "round-robin",
    "defaultTotal": 4
  },

  "adapter-config": {
    "junit": {
      "includeEngines": ["junit-jupiter"],
      "excludeTags": ["slow", "flaky"]
    }
  },

  "output": {
    "format": "human",
    "colors": true
  }
}
```

### Exit Codes

```
0   Success (tests ran in this shard, all passed)
1   Test failures
2   Shard-it error (invalid configuration, adapter not found, etc.)
3   No tests in this shard (warning, not error)
64  Command line usage error
```

### Command Examples

```bash
# Basic usage (explicit adapter)
shard-it --shard=1/4 --adapter=junit -- mvn test

# Environment variable support (for CI)
export SHARD_INDEX=1
export SHARD_TOTAL=4
shard-it --adapter=junit -- mvn test

# With test pattern filtering
shard-it --shard=1/4 --adapter=junit --pattern="**/*Test.java" -- mvn test

# With exclusions
shard-it --shard=1/4 --adapter=junit --exclude="**/integration/**" -- mvn test

# Dry run to preview shard assignment
shard-it --shard=1/4 --adapter=junit --dry-run

# List available adapters
shard-it --list-adapters

# Show adapter capabilities
shard-it --adapter=junit --adapter-info

# JSON output for machine consumption
shard-it --shard=1/4 --adapter=junit --json -- mvn test

# Gradle instead of Maven
shard-it --shard=1/4 --adapter=junit -- ./gradlew test

# With verbose logging
shard-it --shard=1/4 --adapter=junit --verbose -- mvn test
```

---

## Adapter Protocol Specification

### Overview

Adapters communicate with the core binary via **JSON-RPC 2.0 over stdio**. This provides:
- Language-agnostic protocol
- Structured, versioned communication
- Bidirectional request/response pattern
- Error handling built-in

### Communication Flow

```
Core Binary ──┐
              ├──> spawn adapter process
              │
              ├──> Request: discover_tests
              │    { "jsonrpc": "2.0", "method": "discover_tests", "params": {...}, "id": 1 }
              │
              ├<── Response: test list
              │    { "jsonrpc": "2.0", "result": { "tests": [...] }, "id": 1 }
              │
              ├──> Request: shard_tests
              │    { "jsonrpc": "2.0", "method": "shard_tests", "params": {...}, "id": 2 }
              │
              ├<── Response: sharded test list
              │    { "jsonrpc": "2.0", "result": { "tests": [...] }, "id": 2 }
              │
              └──> Execute test command with filtered tests
```

### Required Methods

All adapters must implement these JSON-RPC methods:

#### 1. `get_info()`

**Purpose**: Return adapter metadata and capabilities

**Request:**
```json
{
  "jsonrpc": "2.0",
  "method": "get_info",
  "params": {},
  "id": 1
}
```

**Response:**
```json
{
  "jsonrpc": "2.0",
  "result": {
    "name": "junit",
    "version": "1.0.0",
    "description": "JUnit 5 adapter for Java projects",
    "capabilities": {
      "discovery": true,
      "filtering": true,
      "metadata": true
    },
    "supported_patterns": {
      "default": "**/*Test.java",
      "examples": ["**/*Test.java", "**/*Tests.java", "**/Test*.java"]
    }
  },
  "id": 1
}
```

**TypeScript Interface:**
```typescript
interface AdapterInfo {
  name: string;
  version: string;
  description: string;
  capabilities: {
    discovery: boolean;   // Can discover tests
    filtering: boolean;   // Can filter tests before execution
    metadata: boolean;    // Provides test metadata (tags, annotations)
  };
  supported_patterns: {
    default: string;
    examples: string[];
  };
}
```

#### 2. `discover_tests(params)`

**Purpose**: Discover all tests matching patterns

**Request:**
```json
{
  "jsonrpc": "2.0",
  "method": "discover_tests",
  "params": {
    "patterns": ["**/*Test.java"],
    "exclude": ["**/integration/**"],
    "project_root": "/home/user/my-project",
    "config": {
      "includeEngines": ["junit-jupiter"]
    }
  },
  "id": 2
}
```

**Response:**
```json
{
  "jsonrpc": "2.0",
  "result": {
    "tests": [
      {
        "id": "com.example.UserServiceTest#shouldCreateUser",
        "display_name": "UserServiceTest > shouldCreateUser",
        "source": {
          "file": "src/test/java/com/example/UserServiceTest.java",
          "line": 23
        },
        "metadata": {
          "tags": ["unit", "service"],
          "annotations": ["@Test"]
        }
      },
      {
        "id": "com.example.UserServiceTest#shouldDeleteUser",
        "display_name": "UserServiceTest > shouldDeleteUser",
        "source": {
          "file": "src/test/java/com/example/UserServiceTest.java",
          "line": 45
        },
        "metadata": {
          "tags": ["unit", "service"],
          "annotations": ["@Test"]
        }
      }
    ],
    "total_count": 2
  },
  "id": 2
}
```

**TypeScript Interfaces:**
```typescript
interface DiscoverTestsRequest {
  patterns: string[];
  exclude?: string[];
  project_root: string;
  config?: object;  // Adapter-specific config
}

interface DiscoverTestsResponse {
  tests: TestDescriptor[];
  total_count: number;
}

interface TestDescriptor {
  id: string;             // Unique identifier
  display_name: string;   // Human-readable name
  source: {
    file: string;         // Relative path from project root
    line?: number;        // Optional line number
  };
  metadata?: {
    tags?: string[];      // Test tags/categories
    annotations?: string[]; // Framework annotations
    estimated_duration?: number; // milliseconds (for future timing)
  };
}
```

#### 3. `shard_tests(params)`

**Purpose**: Divide tests into shards

**Request:**
```json
{
  "jsonrpc": "2.0",
  "method": "shard_tests",
  "params": {
    "tests": [
      { "id": "com.example.Test1#method1", "display_name": "...", "source": {...} },
      { "id": "com.example.Test2#method1", "display_name": "...", "source": {...} }
    ],
    "shard_index": 1,
    "shard_total": 4,
    "strategy": "round-robin"
  },
  "id": 3
}
```

**Response:**
```json
{
  "jsonrpc": "2.0",
  "result": {
    "tests": [
      { "id": "com.example.Test1#method1", "display_name": "...", "source": {...} }
    ],
    "shard_index": 1,
    "shard_total": 4,
    "test_count": 1
  },
  "id": 3
}
```

**TypeScript Interfaces:**
```typescript
interface ShardTestsRequest {
  tests: TestDescriptor[];
  shard_index: number;    // 1-based
  shard_total: number;
  strategy: "round-robin" | "sequential";
}

interface ShardTestsResponse {
  tests: TestDescriptor[];
  shard_index: number;
  shard_total: number;
  test_count: number;
}
```

#### 4. `filter_command(params)`

**Purpose**: Modify test command to run only specified tests

**Request:**
```json
{
  "jsonrpc": "2.0",
  "method": "filter_command",
  "params": {
    "original_command": ["mvn", "test"],
    "tests": [
      { "id": "com.example.Test1#method1", "display_name": "...", "source": {...} },
      { "id": "com.example.Test2#method2", "display_name": "...", "source": {...} }
    ],
    "project_root": "/home/user/my-project"
  },
  "id": 4
}
```

**Response:**
```json
{
  "jsonrpc": "2.0",
  "result": {
    "command": ["mvn", "test", "-Dtest=Test1#method1,Test2#method2"],
    "environment": {
      "MAVEN_OPTS": "-Xmx2g"
    }
  },
  "id": 4
}
```

**TypeScript Interfaces:**
```typescript
interface FilterCommandRequest {
  original_command: string[];
  tests: TestDescriptor[];
  project_root: string;
}

interface FilterCommandResponse {
  command: string[];
  environment?: Record<string, string>;
}
```

### Error Handling

Adapters should return JSON-RPC error responses for failures:

```json
{
  "jsonrpc": "2.0",
  "error": {
    "code": -32000,
    "message": "Test discovery failed",
    "data": {
      "detail": "Could not find pom.xml or build.gradle in project root"
    }
  },
  "id": 1
}
```

**Standard Error Codes:**
- `-32700`: Parse error (invalid JSON)
- `-32600`: Invalid request
- `-32601`: Method not found
- `-32602`: Invalid params
- `-32603`: Internal error
- `-32000` to `-32099`: Server-defined errors (adapter-specific)

---

## Programming Language Recommendations

### Core Binary: Rust (Recommended)

**Rationale:**

1. **Performance**: ~20ms cold start vs ~130ms for Node.js
2. **Distribution**: Single binary with no runtime dependencies
3. **Cross-Platform**: Excellent cross-compilation support
4. **Memory Safety**: Prevents entire classes of bugs without GC overhead
5. **Ecosystem**: Mature CLI libraries (clap, serde, tokio)
6. **Future-Proof**: Growing ecosystem, modern tooling

**Key Dependencies:**

```toml
[dependencies]
clap = { version = "4.0", features = ["derive"] }      # CLI parsing
serde = { version = "1.0", features = ["derive"] }     # Serialization
serde_json = "1.0"                                      # JSON support
tokio = { version = "1.0", features = ["full"] }       # Async runtime
jsonrpc-core = "18.0"                                   # JSON-RPC
globset = "0.4"                                         # Glob matching
colored = "2.0"                                         # Terminal colors
anyhow = "1.0"                                          # Error handling
```

**Startup Performance Comparison:**

| Language | Cold Start | Memory | Binary Size |
|----------|-----------|--------|-------------|
| Rust | ~20ms | 15 MB | 3-5 MB |
| Go | ~40ms | 16 MB | 5-8 MB |
| Node.js | ~130ms | 64 MB | Requires runtime |

**Alternative: Go**

If team prefers simpler syntax and faster iteration:
- Simpler learning curve
- Fast compilation
- Also produces single binary
- Trade-off: Slightly larger binaries, garbage collection

### Adapters: Native to Framework Language

**JUnit 5 Adapter: Java**

**Why Java:**
- Direct access to JUnit Platform Launcher API
- No impedance mismatch with framework
- Community contributions easier (Java devs write Java)
- Can leverage Maven/Gradle integration naturally

**Key Dependencies:**
```xml
<dependency>
    <groupId>org.junit.platform</groupId>
    <artifactId>junit-platform-launcher</artifactId>
    <version>1.10.0</version>
</dependency>
<dependency>
    <groupId>org.junit.platform</groupId>
    <artifactId>junit-platform-engine</artifactId>
    <version>1.10.0</version>
</dependency>
```

**Future Adapters:**
- **pytest adapter**: Python (uses pytest's collection hooks)
- **Jest adapter**: TypeScript/JavaScript (uses Jest's test discovery API)
- **RSpec adapter**: Ruby (uses RSpec's configuration system)

---

## High-Level Architecture

### System Architecture Diagram

```
┌─────────────────────────────────────────────────────────────┐
│                        User's Terminal                       │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                  shard-it (Rust Binary)                     │
│                                                              │
│  ┌────────────────┐  ┌──────────────┐  ┌─────────────────┐ │
│  │  CLI Parser    │  │ Config Loader│  │  Adapter Manager│ │
│  │   (clap)       │  │ (serde)      │  │                 │ │
│  └────────────────┘  └──────────────┘  └─────────────────┘ │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐  │
│  │           JSON-RPC Client (over stdio)                │  │
│  └──────────────────────────────────────────────────────┘  │
└───────────────────┬──────────────────────────────────────────┘
                    │ spawn process
                    │ JSON-RPC messages via stdin/stdout
                    ▼
┌─────────────────────────────────────────────────────────────┐
│            Adapter Process (Language-Specific)              │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐  │
│  │           JSON-RPC Server (over stdio)                │  │
│  └──────────────────────────────────────────────────────┘  │
│                                                              │
│  ┌────────────────┐  ┌──────────────┐  ┌─────────────────┐ │
│  │ Test Discovery │  │  Test Filter │  │  Cmd Builder    │ │
│  │ (framework API)│  │  (sharding)  │  │ (modify mvn cmd)│ │
│  └────────────────┘  └──────────────┘  └─────────────────┘ │
│                                                              │
│  Example: shard-it-junit (Java)                             │
│  - Uses JUnit Platform Launcher API                         │
│  - Discovers tests via ClasspathTestDiscovery               │
│  - Filters by tags, annotations                             │
│  - Builds Maven/Gradle test filter args                     │
└───────────────────┬──────────────────────────────────────────┘
                    │ returns modified command
                    ▼
┌─────────────────────────────────────────────────────────────┐
│                  shard-it (executes command)                │
│                                                              │
│     exec: mvn test -Dtest=TestA#method1,TestB#*            │
└───────────────────┬──────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────┐
│                  Maven/Gradle Test Execution                │
│                  (runs filtered tests)                       │
└─────────────────────────────────────────────────────────────┘
```

### Core Components

#### 1. CLI Parser (Rust)
- Parse command-line arguments using clap
- Load and validate configuration files
- Validate shard parameters (1 <= index <= total)
- Handle special modes (--dry-run, --list-adapters, --adapter-info)
- Set up logging and output formatting

#### 2. Config Loader (Rust)
- Search for config files in priority order
- Support JSON, YAML, TOML formats
- Merge configuration sources (CLI > env > project > user > defaults)
- Validate configuration schema
- Provide sensible defaults

#### 3. Adapter Manager (Rust)
- Locate adapter binary (bundled in ./adapters/ directory)
- Validate adapter exists and is executable
- Spawn adapter process
- Establish JSON-RPC connection over stdio
- Handle adapter lifecycle (startup, shutdown, crashes)
- Cache adapter info for performance

#### 4. JSON-RPC Client (Rust)
- Send requests to adapter via stdin
- Receive responses from adapter via stdout
- Handle request/response correlation (ID matching)
- Implement timeout handling
- Parse and validate responses
- Protocol versioning support

#### 5. Sharding Logic (Rust)
- Sort tests deterministically (by ID, alphabetically)
- Implement round-robin distribution
- Implement sequential distribution
- Handle edge cases:
  - More shards than tests
  - Empty shards
  - Single test
- Validate shard assignments

#### 6. Command Executor (Rust)
- Receive modified command from adapter
- Set environment variables
- Execute test command using system shell
- Stream stdout/stderr to user in real-time
- Capture and propagate exit code
- Handle interruption (Ctrl-C) gracefully

### Data Flow

```
1. User runs: shard-it --shard=1/4 --adapter=junit -- mvn test

2. Core parses CLI arguments
   - shard_index = 1, shard_total = 4
   - adapter = "junit"
   - command = ["mvn", "test"]

3. Core loads configuration
   - Check for .shardit.json
   - Merge with CLI args
   - Apply defaults

4. Core locates adapter
   - Find ./adapters/shard-it-junit
   - Verify executable

5. Core spawns adapter process
   - spawn("./adapters/shard-it-junit")
   - Establish stdin/stdout pipes

6. Core sends JSON-RPC: get_info
   Adapter responds with capabilities

7. Core sends JSON-RPC: discover_tests
   Request: { patterns: ["**/*Test.java"], project_root: "." }
   Adapter uses JUnit Platform API to discover tests
   Response: { tests: [...100 tests...], total_count: 100 }

8. Core performs sharding
   - Sort tests by ID (deterministic)
   - Round-robin distribution
   - Shard 1 gets tests [0, 4, 8, 12, ...] = 25 tests

9. Core sends JSON-RPC: filter_command
   Request: { original_command: ["mvn", "test"], tests: [...25 tests...] }
   Adapter builds Maven filter argument
   Response: { command: ["mvn", "test", "-Dtest=Test1#*,Test5#*,..."] }

10. Core executes modified command
    - Set environment variables
    - exec: mvn test -Dtest=Test1#*,Test5#*,...
    - Stream output to terminal

11. Core captures exit code and propagates to shell
```

### Adapter Implementation (JUnit Example)

```java
public class JUnitAdapter {
    private JSONRPCServer server;
    private LauncherDiscoveryRequest discoveryRequest;

    public static void main(String[] args) {
        JUnitAdapter adapter = new JUnitAdapter();
        adapter.start();
    }

    public void start() {
        // Read JSON-RPC requests from stdin
        // Write JSON-RPC responses to stdout
        server = new JSONRPCServer(System.in, System.out);

        server.register("get_info", this::getInfo);
        server.register("discover_tests", this::discoverTests);
        server.register("shard_tests", this::shardTests);
        server.register("filter_command", this::filterCommand);

        server.listen();
    }

    public AdapterInfo getInfo() {
        return new AdapterInfo(
            "junit",
            "1.0.0",
            "JUnit 5 adapter for Java projects",
            new Capabilities(true, true, true),
            new SupportedPatterns("**/*Test.java",
                Arrays.asList("**/*Test.java", "**/*Tests.java"))
        );
    }

    public DiscoverTestsResponse discoverTests(DiscoverTestsRequest req) {
        // Use JUnit Platform Launcher API
        LauncherDiscoveryRequest request = LauncherDiscoveryRequestBuilder
            .request()
            .selectors(selectClasspathRoots(req.projectRoot))
            .filters(
                includeClassNamePatterns(convertGlobToRegex(req.patterns))
            )
            .build();

        Launcher launcher = LauncherFactory.create();
        TestPlan testPlan = launcher.discover(request);

        List<TestDescriptor> tests = testPlan.getRoots().stream()
            .flatMap(root -> testPlan.getDescendants(root).stream())
            .filter(TestIdentifier::isTest)
            .map(this::convertToTestDescriptor)
            .collect(Collectors.toList());

        return new DiscoverTestsResponse(tests, tests.size());
    }

    public FilterCommandResponse filterCommand(FilterCommandRequest req) {
        // Build Maven/Gradle test filter
        String testFilter = req.tests.stream()
            .map(test -> convertToMavenFilter(test))
            .collect(Collectors.joining(","));

        List<String> command = new ArrayList<>(req.originalCommand);
        command.add("-Dtest=" + testFilter);

        return new FilterCommandResponse(command, null);
    }
}
```

---

## Implementation Roadmap

### Phase 1: Define & Validate API (Weeks 1-2)

**Goals:**
- ✅ Define CLI interface
- ✅ Define JSON-RPC adapter protocol
- Write comprehensive API documentation
- Create example usage scenarios
- Get feedback from potential users

**Deliverables:**
- Complete DESIGN.md (this document)
- API specification document
- Protocol specification (JSON-RPC schemas)
- Example usage guide
- Early adopter feedback

### Phase 2: Build Core Binary (Weeks 3-4)

**Goals:**
- Implement Rust CLI binary
- Basic adapter management
- Core sharding logic
- Dry-run and info modes

**Tasks:**
1. Set up Rust project structure
2. Implement CLI parsing with clap
3. Add configuration file loading (JSON, YAML)
4. Build JSON-RPC client for adapter communication
5. Implement round-robin sharding algorithm
6. Add --dry-run mode for debugging
7. Add --list-adapters, --adapter-info commands
8. Unit tests for core logic
9. Integration tests with mock adapter

**Deliverables:**
- `shard-it` binary (Linux, macOS, Windows)
- Core library crate for reuse
- Comprehensive test suite
- CI/CD pipeline (GitHub Actions)

### Phase 3: Build JUnit Adapter (Weeks 5-6)

**Goals:**
- Implement JUnit 5 adapter in Java
- Test discovery using JUnit Platform API
- Maven and Gradle integration
- JSON-RPC server implementation

**Tasks:**
1. Set up Java project structure (Maven)
2. Add JUnit Platform dependencies
3. Implement JSON-RPC server (stdin/stdout)
4. Implement `get_info` method
5. Implement `discover_tests` using JUnit Platform Launcher
6. Implement `shard_tests` (round-robin distribution)
7. Implement `filter_command` for Maven (-Dtest=...)
8. Add Gradle support (-Dtest.single=...)
9. Handle JUnit tags and annotations
10. Unit tests for adapter logic
11. Integration tests with real Java projects

**Deliverables:**
- `shard-it-junit` binary (packaged as JAR or native binary)
- Adapter test suite
- Example Java projects demonstrating usage

### Phase 4: Integration & Testing (Week 7)

**Goals:**
- End-to-end testing
- Performance benchmarks
- CI/CD integration examples
- Bug fixes and polish

**Tasks:**
1. E2E tests with real Java projects
2. Test on multiple platforms (Linux, macOS, Windows)
3. Test with various JUnit versions
4. Performance benchmarks (startup time, discovery time)
5. Create GitHub Actions workflow examples
6. Create GitLab CI examples
7. Test edge cases:
   - Empty shards
   - More shards than tests
   - Large test suites (1000+ tests)
   - Failed test discovery
8. Memory profiling
9. Error message quality audit

**Deliverables:**
- Comprehensive test results
- Performance benchmarks report
- CI/CD integration guides
- Bug fixes and improvements

### Phase 5: Documentation & Release (Week 8)

**Goals:**
- Complete documentation
- Publish v1.0.0
- Community enablement

**Tasks:**
1. Write user documentation
   - Installation guide
   - Quick start guide
   - CLI reference
   - Configuration reference
   - Troubleshooting guide
2. Write adapter development guide
   - Protocol specification
   - Language-specific templates
   - Testing guide
3. Create example projects
   - Simple Java/JUnit project
   - Multi-module Maven project
   - Gradle project
   - GitHub Actions integration
4. Set up project website (GitHub Pages)
5. Create release notes
6. Publish GitHub release with binaries
7. Announce on relevant channels

**Deliverables:**
- Complete documentation site
- v1.0.0 release on GitHub
- Binaries for Linux, macOS, Windows
- Example projects repository
- Adapter development guide

### Phase 6: Community & Expansion (Post-v1.0)

**Goals:**
- Enable community contributions
- Expand adapter ecosystem
- Iterate based on feedback

**Tasks:**
1. Create contributor guide
2. Set up issue templates
3. Define governance model
4. Accept and review adapter contributions
5. Build additional official adapters (pytest, jest)
6. Collect user feedback
7. Plan v2.0 features (timing-based sharding)
8. Regular maintenance releases

**Deliverables:**
- Active community
- Multiple language adapters
- Regular updates and improvements

---

## Distribution Strategy

### Version 1.0: Bundled Distribution

**Single Release Artifact:**

```
shard-it-v1.0.0-linux-x64.tar.gz
├── shard-it                    # Core binary (Rust)
└── adapters/
    └── shard-it-junit          # JUnit adapter (packaged Java)
```

**Platform-Specific Releases:**
- Linux x64: `shard-it-v1.0.0-linux-x64.tar.gz`
- Linux ARM64: `shard-it-v1.0.0-linux-arm64.tar.gz`
- macOS x64: `shard-it-v1.0.0-darwin-x64.tar.gz`
- macOS ARM64: `shard-it-v1.0.0-darwin-arm64.tar.gz`
- Windows x64: `shard-it-v1.0.0-windows-x64.zip`

**Installation Methods:**

#### 1. Manual Download (GitHub Releases)
```bash
# Download and extract
curl -L https://github.com/yourorg/shard-it/releases/download/v1.0.0/shard-it-v1.0.0-linux-x64.tar.gz | tar xz

# Move to PATH
sudo mv shard-it /usr/local/bin/
sudo mkdir -p /usr/local/lib/shard-it
sudo mv adapters /usr/local/lib/shard-it/
```

#### 2. Shell Script Installer
```bash
curl -fsSL https://shard-it.dev/install.sh | sh
```

The install script:
- Detects platform and architecture
- Downloads correct binary from GitHub releases
- Verifies checksum
- Installs to appropriate location
- Updates PATH if needed

#### 3. Package Managers

**Homebrew (macOS/Linux):**
```bash
brew install shard-it
```

**Cargo (Rust ecosystem):**
```bash
cargo install shard-it
```

**Scoop (Windows):**
```powershell
scoop install shard-it
```

### Version 2.0+: Modular Distribution

**Separate Core and Adapters:**

- Core: `shard-it` (standalone)
- Adapters: Installed on-demand
  - `shard-it install-adapter junit`
  - `shard-it install-adapter pytest`
  - `shard-it install-adapter <github-url>` (community adapters)

**Adapter Registry:**
- Official adapters hosted on GitHub releases
- Community adapters discoverable via registry file
- Automatic version checking and updates

### Packaging Considerations

**Adapter Packaging Options:**

1. **JAR with bundled JRE** (JUnit adapter)
   - Self-contained, no Java installation required
   - Larger download (~50 MB)
   - Works everywhere

2. **JAR requiring system Java**
   - Smaller download (~5 MB)
   - Requires Java 11+ on system
   - More common in Java ecosystem

3. **Native binary (GraalVM)**
   - Compiled to native binary using GraalVM
   - Fast startup, small size
   - Requires build for each platform

**Recommendation for v1.0:** JAR requiring system Java (simpler, smaller, Java developers have JDK)

### Update Mechanism

**Version Checking:**
```bash
# Check for updates (cached for 24 hours)
shard-it --version
# shard-it 1.0.0
# A new version is available: 1.1.0
# Run 'shard-it update' to upgrade

# Force check
shard-it --check-update

# Update (downloads and replaces binary)
shard-it update
```

**Auto-Update (opt-in):**
- Environment variable: `SHARDIT_AUTO_UPDATE=true`
- Checks once per day
- Downloads in background
- Prompts before replacing

---

## Usage Examples

### Basic Local Development

```bash
# Run 1/4 of tests locally for faster iteration
shard-it --shard=1/4 --adapter=junit -- mvn test

# Preview which tests would run
shard-it --shard=1/4 --adapter=junit --dry-run

# Run with custom pattern
shard-it --shard=1/4 --adapter=junit --pattern="**/unit/**/*Test.java" -- mvn test
```

### GitHub Actions Integration

```yaml
name: Tests

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest

    strategy:
      matrix:
        shard: [1, 2, 3, 4]

    steps:
      - uses: actions/checkout@v3

      - name: Setup Java
        uses: actions/setup-java@v3
        with:
          java-version: '17'
          distribution: 'temurin'

      - name: Install shard-it
        run: |
          curl -L https://github.com/yourorg/shard-it/releases/download/v1.0.0/shard-it-linux-x64.tar.gz | tar xz
          sudo mv shard-it /usr/local/bin/
          sudo mkdir -p /usr/local/lib/shard-it
          sudo mv adapters /usr/local/lib/shard-it/

      - name: Run tests (shard ${{ matrix.shard }}/4)
        run: |
          shard-it --shard=${{ matrix.shard }}/4 --adapter=junit -- mvn test

      - name: Upload test results
        if: always()
        uses: actions/upload-artifact@v3
        with:
          name: test-results-${{ matrix.shard }}
          path: target/surefire-reports/
```

### GitLab CI Integration

```yaml
test:
  parallel: 4

  image: maven:3.9-eclipse-temurin-17

  before_script:
    - curl -L https://github.com/yourorg/shard-it/releases/download/v1.0.0/shard-it-linux-x64.tar.gz | tar xz
    - mv shard-it /usr/local/bin/
    - mkdir -p /usr/local/lib/shard-it
    - mv adapters /usr/local/lib/shard-it/

  script:
    - shard-it --shard=${CI_NODE_INDEX}/${CI_NODE_TOTAL} --adapter=junit -- mvn test

  artifacts:
    reports:
      junit: target/surefire-reports/TEST-*.xml
```

### CircleCI Integration

```yaml
version: 2.1

jobs:
  test:
    docker:
      - image: cimg/openjdk:17.0

    parallelism: 4

    steps:
      - checkout

      - run:
          name: Install shard-it
          command: |
            curl -L https://github.com/yourorg/shard-it/releases/download/v1.0.0/shard-it-linux-x64.tar.gz | tar xz
            sudo mv shard-it /usr/local/bin/
            sudo mkdir -p /usr/local/lib/shard-it
            sudo mv adapters /usr/local/lib/shard-it/

      - run:
          name: Run tests
          command: |
            # CircleCI uses 0-based indexing, convert to 1-based
            SHARD_INDEX=$((CIRCLE_NODE_INDEX + 1))
            shard-it --shard=${SHARD_INDEX}/${CIRCLE_NODE_TOTAL} --adapter=junit -- mvn test

      - store_test_results:
          path: target/surefire-reports

workflows:
  test:
    jobs:
      - test
```

### Configuration File Example

```json
{
  "adapter": "junit",

  "discovery": {
    "pattern": "**/*Test.java",
    "exclude": [
      "**/integration/**",
      "**/e2e/**",
      "**/performance/**"
    ],
    "sort": "path"
  },

  "shard": {
    "strategy": "round-robin",
    "defaultTotal": 4
  },

  "adapter-config": {
    "junit": {
      "includeEngines": ["junit-jupiter"],
      "excludeTags": ["slow", "flaky", "manual"],
      "includeTags": ["unit", "fast"],
      "configFile": "junit-platform.properties"
    }
  },

  "output": {
    "format": "human",
    "colors": true,
    "verbose": false
  }
}
```

### Advanced Scenarios

#### Running Only Unit Tests

```bash
# Using configuration
cat > .shardit.json << EOF
{
  "adapter": "junit",
  "adapter-config": {
    "junit": {
      "includeTags": ["unit"]
    }
  }
}
EOF

shard-it --shard=1/4 --adapter=junit -- mvn test

# Or using CLI flags
shard-it --shard=1/4 --adapter=junit --pattern="**/unit/**/*Test.java" -- mvn test
```

#### Gradle Project

```bash
shard-it --shard=1/4 --adapter=junit -- ./gradlew test
```

#### Multi-Module Maven Project

```bash
# Test specific module
shard-it --shard=1/4 --adapter=junit -- mvn test -pl my-module

# Test all modules
shard-it --shard=1/4 --adapter=junit -- mvn test
```

#### JSON Output for Reporting

```bash
shard-it --shard=1/4 --adapter=junit --json -- mvn test > results.json
```

#### Dry Run for Debugging

```bash
# See exactly which tests will run in each shard
for i in 1 2 3 4; do
  echo "=== Shard $i/4 ==="
  shard-it --shard=$i/4 --adapter=junit --dry-run
done
```

---

## Success Criteria

### API Success Criteria

- ✅ **Intuitive CLI**: New users can run `shard-it --help` and understand immediately
- ✅ **Zero Configuration**: Works for standard projects without any setup
- ✅ **Optional Configuration**: Config file is optional but enables advanced use cases
- ✅ **CI Integration**: Works seamlessly in GitHub Actions, GitLab CI, CircleCI
- ✅ **Clear Errors**: Error messages guide users to solutions
- ✅ **Discoverable**: `--list-adapters`, `--adapter-info`, `--dry-run` help users

### Technical Success Criteria

- ✅ **Deterministic Sharding**: Same shard = same tests every time (reproducible)
- ✅ **Fast Startup**: Core binary starts in <100ms
- ✅ **Edge Case Handling**:
  - Empty shards (more shards than tests)
  - No tests found
  - Invalid patterns
  - Adapter failures
- ✅ **Cross-Platform**: Works on Linux, macOS, Windows
- ✅ **Proper Exit Codes**: 0 (success), 1 (test failure), 2 (tool error)
- ✅ **Real-Time Output**: Streams test output as it runs

### Ecosystem Success Criteria

- ✅ **JUnit Adapter**: Works with Maven and Gradle
- ✅ **Comprehensive Docs**:
  - Installation guide
  - Quick start
  - CLI reference
  - Adapter development guide
- ✅ **Example Projects**: At least 3 real-world examples
- ✅ **Community Ready**: Clear contributing guide, issue templates
- ✅ **Adoption**: At least 3 real projects using it successfully

### Performance Benchmarks

**Target Metrics:**

| Metric | Target | Rationale |
|--------|--------|-----------|
| Startup Time | <100ms | Perceived as instant |
| Test Discovery | <5s for 1000 tests | Fast enough for CI |
| Sharding Overhead | <1s | Negligible compared to test execution |
| Memory Usage | <50 MB | Lightweight enough to run alongside tests |

### Quality Gates

**Before v1.0 release:**

1. **Code Quality**
   - [ ] All tests passing
   - [ ] No compiler warnings
   - [ ] Code coverage >80%
   - [ ] Security audit (cargo audit / OWASP dependency check)

2. **Documentation**
   - [ ] API documentation complete
   - [ ] User guide complete
   - [ ] Adapter development guide complete
   - [ ] All examples working

3. **Testing**
   - [ ] Unit tests for core and adapter
   - [ ] Integration tests with real projects
   - [ ] E2E tests on all platforms
   - [ ] Performance benchmarks meet targets

4. **User Validation**
   - [ ] At least 3 early adopters testing pre-release
   - [ ] Feedback incorporated
   - [ ] No critical issues reported

---

## Next Steps

### Immediate Actions (Week 1)

1. **Validate API Design**
   - Share this design doc with potential users
   - Gather feedback on CLI interface
   - Iterate on protocol if needed

2. **Set Up Project Structure**
   - Create GitHub repository
   - Initialize Rust project
   - Set up CI/CD (GitHub Actions)
   - Create project roadmap

3. **Prototype Core Binary**
   - Basic CLI parsing
   - Mock adapter communication
   - Validate approach

### Questions to Resolve

1. **Naming**: Is "shard-it" the final name? Alternative: "shardify", "test-shard", "parallel-test"
2. **License**: Open source license choice (MIT, Apache 2.0, GPL)?
3. **Hosting**: GitHub organization or personal repo?
4. **Adapter Packaging**: JAR vs native binary for JUnit adapter?

### Decisions Needed

- [ ] Final project name
- [ ] Open source license
- [ ] Repository hosting
- [ ] Initial contributors/maintainers
- [ ] Release cadence (monthly, quarterly?)

---

## Appendix

### Research Sources

1. **Test Sharding Tools**
   - Playwright Test Sharding: https://playwright.dev/docs/test-sharding
   - Jest Sharding: https://jestjs.io/docs/cli#--shard
   - CircleCI Test Splitting: https://circleci.com/docs/parallelism-faster-jobs/
   - Knapsack Pro: https://knapsackpro.com/
   - Bazel Test Sharding: https://bazel.build/reference/test-encyclopedia

2. **CLI Design**
   - CLI Guidelines: https://clig.dev/
   - 12 Factor CLI Apps: https://medium.com/@jdxcode/12-factor-cli-apps-dd3c227a0e46
   - Command Line Interface Guidelines: https://www.gnu.org/prep/standards/

3. **Framework Integration**
   - JUnit Platform Launcher API: https://junit.org/junit5/docs/current/user-guide/#launcher-api
   - Serenity/JS Architecture: https://serenity-js.org/handbook/design/
   - Wallaby.js Architecture: https://wallabyjs.com/

4. **JSON-RPC**
   - JSON-RPC 2.0 Specification: https://www.jsonrpc.org/specification

### Glossary

- **Adapter**: Language-specific component that integrates with a test framework
- **Shard**: A subset of tests distributed to run in parallel
- **Shard Index**: The current shard number (1-based in our design)
- **Shard Total**: The total number of shards
- **Test Descriptor**: Structured representation of a test with metadata
- **Round-Robin**: Distribution strategy where tests are assigned sequentially to shards
- **Sequential**: Distribution strategy where tests are divided into contiguous blocks
- **JSON-RPC**: Remote procedure call protocol using JSON encoding

### Version History

| Version | Date | Changes |
|---------|------|---------|
| 1.0 | 2025-11-13 | Initial design document |

---

**End of Design Document**
