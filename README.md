# loam
Langchain Orchestration and Agent Manager

## Title: **Agentic Workflow Orchestrator for Go Codebases**

---

## 🧠 Purpose

To implement a **developer-focused, multi-agent orchestration service** that:

- Accepts commands via CLI, API, or VS Code agent
- Agentically plans how to perform complex changes to a Go codebase (e.g., add a database, refactor APIs)
- Executes those steps reliably using **Temporal workflows**
- Leverages **LangChain (Python)** for flexible planning and tool invocation

---

## ⚙️ System Components

### 1. **Entry Interface (Frontend Inputs)**

**Type:** HTTP REST API (Go), optionally CLI or gRPC\
**Responsibilities:**

- Accept task input (`POST /orchestrate`)
- Validate request (repo path, command string)
- Start a Temporal workflow with the input

**Inputs:**

```json
{
  "command": "Add vector DB support using Qdrant",
  "repoPath": "/mnt/repos/myservice",
  "caller": "cursor-agent"
}
```

---

### 2. **Temporal Workflow (Go)**

**Type:** Durable orchestrator\
**Responsibilities:**

- Call LangChain to generate a multi-step plan
- Iterate through steps and run execution activities
- Retry, recover, and log all progress
- Store execution context per workflow instance

**Steps (per workflow instance):**

1. `AgenticPlanner`: POST to LangChain agent server (Python)
2. `ParseSteps`: Tokenize steps (YAML/JSON plan)
3. For each step:
   - `ExecuteStep`: Tool-specific action (write file, install module, etc.)
   - `VerifyOutcome`: Optional check per step

---

### 3. **LangChain Planner (Python Service)**

**Type:** Flask/FastAPI service callable by Go\
**Responsibilities:**

- Accept `task` as prompt (e.g. "Add Qdrant support")
- Use LangGraph or function-calling chain to:
  - Analyze the Go project
  - Generate a multi-step execution plan
- Return structured plan (JSON)

**Example Output:**

```json
[
  {"step": "Install qdrant-client-go via go.mod"},
  {"step": "Create vector_store.go with client logic"},
  {"step": "Add indexing code to main.go"},
  {"step": "Write tests for vector integration"}
]
```

---

### 4. **Execution Tooling (Go)**

**Responsibilities:**

- Execute plan steps with proper isolation
- Provide tooling for:
  - Editing Go files (AST-based or templated)
  - Running `go get`, `go test`
  - Committing or staging changes
- Optionally shell out to Python tools for LLM embeddings, if needed

---

### 5. **Observability & Security**

- Use Temporal Web UI for live workflow visualization
- Audit logs per step stored in local DB or cloud log store
- Use Tailscale + MagicDNS + TLS for secure in-network access

---

## 🔐 Interfaces / Integration Points

| Interface        | Protocol | Purpose                        |
| ---------------- | -------- | ------------------------------ |
| CLI Tool         | HTTP     | Dev tool for triggering tasks  |
| Cursor Agent     | HTTP     | Accept commands inside VS Code |
| LangChain Agent  | HTTP     | Receives planning prompt       |
| Git Repo (Local) | FS       | Codebase read/write            |

---

## 💠 Technologies

| Layer         | Tech                           |
| ------------- | ------------------------------ |
| Orchestrator  | **Go**, Temporal SDK           |
| Planner Agent | **Python**, LangChain, FastAPI |
| Interfaces    | HTTP REST, CLI (`cobra`)       |
| Auth/Security | Tailscale, MagicDNS, JWT       |
| Observability | Temporal Web, structured logs  |
| Code Editor   | `go/ast`, `text/template`      |

---

## 🔄 Example Flow

1. Cursor agent sends:\
   `"Add vector search using Qdrant to Go repo /mnt/repos/myservice"`

2. API triggers Temporal workflow:

   - `OrchestrationWorkflow("Add vector search", "/mnt/...")`

3. Planner agent (LangChain) returns structured plan:

   - `"Install dependency"`, `"Write vector_store.go"`, etc.

4. Executor runs each step:

   - Writes code, installs packages, commits changes

5. Final status returned to caller

---
