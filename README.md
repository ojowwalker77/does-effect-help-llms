Hypothesis

     Effect-TS's explicit error types, dependency injection (Layers), and structured concurrency help LLMs fix bugs faster and with fewer tokens compared to traditional TypeBox/Zod patterns.

     Architecture

     does-llms-want-effect/
     ├── codebases/
     │   ├── baseline/          # Bun + TypeBox + traditional patterns
     │   │   └── src/
     │   └── effect/            # Bun + Effect-TS
     │       └── src/
     ├── scenarios/
     │   ├── 01-missing-error-case/
     │   │   ├── bug.patch      # Git patch to introduce bug
     │   │   ├── prompt.md      # Exact prompt for Claude
     │   │   └── test.ts        # Verification test
     │   └── ... (10-20 scenarios)
     ├── harness/
     │   ├── runner.ts          # Spawns Claude Code, captures metrics
     │   ├── analyze.ts         # Statistical analysis
     │   └── results/           # JSON output
     └── README.md

     Codebase Spec (~600 LOC each)

     Domain: Simple Order API

     - Users (CRUD)
     - Orders (create, list, status transitions)
     - Email notifications (mock external service)
     - SQLite persistence

     Baseline Stack (no Effect)

     - Bun.serve native
     - TypeBox for validation
     - better-sqlite3 / bun:sqlite
     - Manual error handling (try/catch, union returns)
     - Constructor-based DI

     Effect Stack

     - Bun.serve native
     - @effect/schema
     - Effect SQL (or manual with Effect wrapper)
     - Effect error channel
     - Layer-based DI

     Bug Scenarios (20 total, 4 per category)

     Category A: Error Handling (4)

     1. Missing error case in order status transition
     2. Swallowed DB error on user creation
     3. Wrong error type returned from validation
     4. Error lost in Promise.all (only first reported)

     Category B: Dependency Injection (4)

     5. Service using wrong DB instance
     6. Circular dependency causing undefined
     7. Missing service initialization
     8. Stale closure capturing old service reference

     Category C: Concurrency/Async (4)

     9. Race condition in order counter
     10. Missing await on email send
     11. Parallel requests corrupting shared state
     12. Timeout not cancelling downstream operations

     Category D: Resource Management (4)

     13. DB connection not closed on error
     14. File handle leak in export
     15. Transaction not rolled back on failure
     16. Connection pool exhausted under load

     Category E: Type Inference (4)

     17. Generic type mismatch in service method
     18. Union type not narrowed correctly
     19. Schema inference losing optional fields
     20. Discriminated union check incomplete

     Measurement Metrics
     ┌─────────────────┬────────────────────────────────┐
     │     Metric      │          How Measured          │
     ├─────────────────┼────────────────────────────────┤
     │ Token count     │ Parse Claude Code output/logs  │
     ├─────────────────┼────────────────────────────────┤
     │ Wall clock time │ Process start → test pass      │
     ├─────────────────┼────────────────────────────────┤
     │ Tool calls      │ Count from conversation log    │
     ├─────────────────┼────────────────────────────────┤
     │ Success         │ Test file passes (exit code 0) │
     └─────────────────┴────────────────────────────────┘
     Harness Design

     // harness/runner.ts
     interface ScenarioResult {
       scenario: string
       codebase: 'baseline' | 'effect'
       runNumber: 1 | 2 | 3
       success: boolean
       tokensUsed: number
       wallClockMs: number
       toolCalls: number
       conversationTurns: number
     }

     // Run each scenario 3x per codebase for variance
     // Total runs: 20 scenarios × 2 codebases × 3 runs = 120 runs

     Execution Protocol

     1. Fresh git worktree per run
     2. Apply bug.patch
     3. Spawn Claude Code: claude --print "$(cat prompt.md)" --output-format json
     4. Timeout: 10 min (no token limit - measure actual usage)
     5. Run test.ts to verify fix
     6. Parse JSON output for token counts, tool calls
     7. Log all metrics to results/

     Analysis Plan

     - Mean/median comparison per category
     - Statistical significance (t-test or Mann-Whitney)
     - Token efficiency = success_rate / avg_tokens
     - Qualitative: Review Claude's reasoning in failures

     Decisions Made

     - Timeout: 10 min, no token limit (measure actual usage)
     - Runs: 3 per scenario for variance (120 total runs)
     - Scenarios: 20 (4 per category)
     - Model: Claude Opus 4.5 for both

     Estimated Cost

     ~120 runs × ~$5-15 per run (Opus) = $600-1800
     (Depends on average tokens per run)

     Implementation Order

     Phase 1: Codebases

     1. Create baseline codebase (Bun + TypeBox + SQLite)
       - Bun.serve HTTP server
       - User/Order CRUD
       - TypeBox schemas
       - Manual DI pattern
     2. Create effect codebase (equivalent logic)
       - Same API surface
       - @effect/schema
       - Layer-based services
       - Effect error channel
     3. Write shared test suite (works against both)

     Phase 2: Bug Scenarios

     4. Design 20 bugs, create git patches + prompts
     5. Verify each bug breaks tests, fix restores them

     Phase 3: Harness

     6. Build runner.ts (subprocess, metric parsing)
     7. Build analyze.ts (stats, charts)

     Phase 4: Execution

     8. Pilot run (3 scenarios) - validate setup
     9. Full benchmark (120 runs)
     10. Analysis and writeup
