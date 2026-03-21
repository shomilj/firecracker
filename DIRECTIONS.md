# System Documentation Directions

Read this entire file before doing anything. You must follow these directions exactly, and every subtask you spawn must also read this file and follow these directions.

## Your Process

1. Read this file first.
2. List the directory you've been assigned using `ls` or Glob.
3. **First, write documentation for your current path.** Write `README.md` at your assigned path describing the architecture of everything underneath it.
4. **Then, decide whether to recurse.** If your README.md fully captures the architecture of everything underneath this path—all the important design decisions, data flows, and system properties—stop here. If there are subdirectories with substantial complexity that deserve their own dedicated documentation (distinct modules, services, or subsystems that you couldn't do justice to in your README), spawn parallel subtasks for each. The goal is comprehensive documentation, not maximum recursion.
5. Every subtask prompt must include the full writing instructions below and say: "First read `DIRECTIONS.md` at the repo root. You are analyzing `<path>`. Follow the directions exactly."

## Retry Logic

If you hit depth or rate limits when spawning subagents, run `sleep 15` in Shell and retry, up to 5 times.

## Writing Instructions

Pass these exact instructions to every subtask:

---

Write comprehensive documentation for the <subsystem> architecture in <subsystem>/README.md. Overwrite whatever is there. Use this structure:

Good: "For authentication, a session-based model is used where the server validates tokens on every incoming request before any handler logic runs. Sessions are stored server-side with a configurable TTL, and the system fails closed—any validation error immediately rejects the request with an unauthorized response rather than falling back to anonymous access."

Bad: "• Session-based auth • Validates per-request • Fails closed"

Guidelines:
- For large-scale systems, ASCII diagrams are often helpful. Include ASCII diagrams where necessary. Be careful about pipe alignment, though.
- Focus on the high and low-level design of the system.
- File names, file paths, and directory maps are not helpful. Do not include these. Focus instead on comprehensively documenting the system design.
- Do not start documents with "This area is", "This subsystem is", etc.; write each document as a standalone piece of documentation describing the system.
- When writing documents, go _extremely_ deep in the description of functionality, design, behavior, and implementation detail. Do not mention function names / file names / etc.

Use a decimal outline style.

---

