● Based on my investigation of the Claude Flow codebase, I can now provide you with a comprehensive answer about its
   exact mechanisms:

  Claude Flow: Core Mechanisms & Architecture

  1. Entry Points & Command Structure

  Claude Flow operates through multiple entry points:

  - Main CLI Dispatcher (bin/claude-flow.js): Cross-platform dispatcher that detects the best available runtime
  (Node.js, tsx, or compiled binaries)
  - Simple CLI (src/cli/simple-cli.js): JavaScript wrapper that avoids TypeScript issues and handles command routing
   via command-registry.js
  - Command Registry: Dynamic command loading system that maps CLI commands to handler functions

  2. Swarm Coordination Mechanisms

  SwarmCoordinator (src/coordination/swarm-coordinator.ts) is the central orchestration system:

  Key Components:
  - Agent Registry: Map of active agents with type, status, capabilities, and metrics
  - Objectives Map: High-level goals decomposed into tasks
  - Task Queue: Pending/running/completed tasks with dependencies
  - Background Workers: Interval-based task polling and health checks
  - Work Stealing: Load balancing when parallel execution is enabled
  - Circuit Breaker: Fault tolerance for failing agents

  Configuration:
  {
    maxAgents: 10,
    maxConcurrentTasks: 5,
    taskTimeout: 300000, // 5 minutes
    enableMonitoring: true,
    enableWorkStealing: true,
    coordinationStrategy: 'hybrid' // centralized | distributed | hybrid
  }

  3. Memory & State Management

  Persistent Memory System (src/memory/shared-memory.js):

  Database Structure:
  - Storage: SQLite database at .swarm/memory.db and .hive-mind/hive.db
  - Tables:
    - memory_store: key-value storage with namespace isolation
    - metadata: system information
    - migrations: schema version tracking

  Features:
  - LRU Cache: In-memory cache (1000 entries, 50MB) for hot data
  - Namespace Isolation: Organize memories by domain (e.g., "swarm", "hive-collective")
  - TTL & Expiration: Auto-cleanup of expired entries
  - Compression: Optional data compression for large values
  - Performance Indexing: Multi-column indexes for fast queries

  ReasoningBank Integration (via agentic-flow@1.5.13):
  - Semantic Search: Hash-based embeddings (1024-dim) for API-key-free search
  - MMR Ranking: Maximal Marginal Relevance scoring
  - Query Latency: 2-3ms average
  - Storage Format: patterns, embeddings, trajectories, links tables

  4. Hooks System & Feedback Loops

  AgenticHookManager (src/services/agentic-flow-hooks/hook-manager.ts):

  Hook Types:
  - pre-task: Fired before task execution (validation, resource prep)
  - post-task: After task completion (cleanup, metrics)
  - pre-edit: Before file modifications
  - post-edit: After file changes (memory updates)
  - session-restore: Load previous context
  - session-end: Persist state and export metrics

  Hook Registration System:
  {
    id: 'hook-id',
    type: 'pre-task',
    priority: 100, // Higher priority executes first
    filter: { /* matching criteria */ },
    handler: async (payload, context) => { /* hook logic */ }
  }

  Hook Execution Flow:
  1. Filter Matching: HookMatcher validates applicability (2-3x performance vs naive filtering)
  2. Priority Sorting: Hooks execute in priority order
  3. Pipeline Processing: Results can modify payload for next hook
  4. Error Handling: Failed hooks don't block execution (logged)
  5. Metrics Collection: Execution time, success/failure rates

  5. Agent Communication Protocols

  BaseAgent (src/cli/agents/base-agent.ts):

  Agent Lifecycle:
  initialize() → idle → assignTask() → busy → executeTask() →
    [success] → completed → idle
    [failure] → error handling → retry/failed

  Communication Channels:
  - EventEmitter: Node.js events for agent state changes
  - EventBus: Global event system for cross-agent messages
  - DistributedMemory: Shared memory for data exchange
  - Heartbeat: Periodic status updates (configurable interval)

  Agent State Tracking:
  - Current tasks, task history, error history
  - Collaborators (peer agents), child agents (hierarchical)
  - Workload percentage, health score
  - Metrics: tasks completed/failed, total duration

  6. File Interactions

  Key Data Files:
  - .swarm/memory.db: SQLite database for swarm memory
  - .hive-mind/hive.db: Hive-mind collective memory
  - .hive-mind/config.json: Queen/worker configuration
  - .hive-mind/sessions/: Saved sessions for resumption
  - ./swarm-runs/{swarmId}/: Per-swarm execution logs and configs

  Configuration Files:
  - config.json: Queen capabilities, consensus algorithms, communication protocols
  - implementation-state.json: Progress tracking across sessions
  - Session auto-save: Every 30 seconds during execution

  7. Integration with External Systems

  Agentic-Flow Backend (dependency):
  - Provides ReasoningBank memory (replaces WASM)
  - SQLite persistence with better-sqlite3
  - Semantic search with hash embeddings
  - 400KB per pattern with embeddings

  MCP Tools (Model Context Protocol):
  - 100+ tools exposed via MCP server
  - Categories: swarm, memory, neural, github, monitoring
  - Integration: claude mcp add claude-flow npx claude-flow@alpha mcp start

  GitHub Integration:
  - 6 specialized modes: pr-manager, issue-tracker, release-manager, etc.
  - Workflow automation, PR coordination, repo analysis

  8. Feedback Loop Architecture

  User Command → CLI Dispatcher → Command Handler →
    [Swarm Coordinator initiates] →
      Agent Registration → Task Decomposition →
        Task Assignment → Agent Execution →
          [Hooks: pre-task] →
            Actual Work (Claude Code API calls, file ops) →
          [Hooks: post-task] →
        Memory Storage (patterns, results) →
      Task Completion Events →
    Metrics Collection → Status Updates →
  User Results

  Cross-Session Continuity:
  1. Session start: Load memories from SQLite
  2. During execution: Store decisions/patterns
  3. Session end: Persist full state
  4. Next session: Resume from saved state

  9. Performance Optimizations

  - Parallel Execution: 2.8-4.4x speed improvement (claimed)
  - Work Stealing: Dynamic load balancing across agents
  - LRU Caching: 50MB in-memory cache with 2-3ms hit latency
  - Indexed Queries: Multi-column SQLite indexes
  - Hook Matcher: 2-3x faster than naive filtering
  - Background Workers: Async task polling (5s intervals)

  ---
  Summary

  Claude Flow is a multi-layered orchestration system that combines:
  1. CLI framework for user interaction
  2. Swarm coordinator for agent management
  3. SQLite-backed memory for persistence
  4. Hook system for workflow automation
  5. Event-driven architecture for agent communication
  6. Integration layer for external tools (MCP, GitHub, etc.)