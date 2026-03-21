# Architecture Documentation Directions

Read this entire file before doing anything. You must follow these directions exactly, and every subtask you spawn must also read this file and follow these directions.

## Your Process

1. Read this file first.
2. List the directory you've been assigned using `ls` or Glob.
3. Count lines of code using `find <path> -name '*.ts' -o -name '*.py' -o -name '*.go' -o -name '*.rs' | xargs wc -l` (adjust extensions as needed).
4. Decide whether to decompose or analyze:
   - If the directory contains **multiple subdirectories that look like distinct modules or services**, spawn a parallel subtask for each. Do not write anything yourself—just delegate.
   - If the directory is a **cohesive module under ~10,000 lines**, analyze it and write `ARCHITECTURE.md` in that directory.
   - If the directory is a **large module over ~10,000 lines**, find logical subcomponents and spawn subtasks for each.
5. Every subtask prompt must say: "First read `DIRECTIONS.md` at the repo root. You are analyzing `<path>`. Follow the directions exactly."

## Retry Logic

If you hit depth or rate limits when spawning subagents, run `sleep 15` in Shell and retry, up to 5 times.

## Writing Style

Write like a staff engineer authoring a technical design document—precise, authoritative, comprehensive. Use full paragraphs as the default. Bullet points, numbered lists, and tables are acceptable when they genuinely clarify structure, but use them conservatively. Every sentence must be complete—never use sentence fragments, even in lists.

Bad: "• Session-based auth • Validates per-request • Fails closed"

Good: "The authentication system uses a session-based model where the server validates tokens on every incoming request before any handler logic runs. Sessions are stored server-side with a configurable TTL, and the system fails closed—any validation error immediately rejects the request with an unauthorized response rather than falling back to anonymous access. This design prioritizes security over availability, accepting that a session store outage will block all authenticated traffic rather than risk exposing user data to unauthenticated requests."

## Absolutely No Code References

Do not mention file names, function names, class names, variable names, or any code symbols. Focus entirely on architectural concepts: how components interact, what invariants are maintained, how data flows between systems, what the failure modes are, and why the design makes these tradeoffs. A reader should understand the system without ever looking at the code.

## Required Sections

Every ARCHITECTURE.md must have these sections in this order:

**Purpose & Boundaries.** What is this system's responsibility? What is explicitly not its responsibility? What would break in the larger system if this component failed?

**Interfaces & Contracts.** What does it expose to the outside world? What does it consume from other systems? What invariants must callers maintain, and what guarantees does it provide in return?

**Data Flow.** What data enters the system, from where, and in what form? What transformations occur? What data exits, and to where?

**Control Flow.** What triggers execution? What is the critical path through the system? What are the branching points and their conditions?

**State & Lifecycle.** What state does this system own? How is state initialized, modified, and cleaned up? What happens during startup, steady-state operation, shutdown, and recovery?

**Failure Modes.** How does this system fail? What failures are handled explicitly versus propagated? What assumptions, if violated, cause silent corruption?

**Operational Characteristics.** What are the resource consumption patterns? What are the scaling bottlenecks? What observability exists—logging, metrics, traces?

**Design Rationale.** What problem does this design solve? What tradeoffs were made? What constraints shaped the design?

## Diagrams

Include ASCII diagrams for system topology, data flow, and state machines where they help. Every diagram must have explanatory prose before and after it.

## Depth

Each document should be substantial—at least 1500 words for any non-trivial system. Depth matters more than breadth.
