# Building an AI Incident Investigator with an MCP Server and a Harness

I built this application for two purposes:
1. I wanted to understand how to build an MCP server
2. I wanted to understand what harness engineering is and how it works 

## The Problem

This project simulates investigating an incident using (fake) security logs. Its a multi-step process, that takes data from multiple sources and melds them into a final report, an excellent place to use an MCP server. If an AI system says a deployment caused a spike in HTTP 500s, an engineer should be able to ask:

- Which logs support that?
- Which metrics changed?
- Did a deployment actually happen at the right time?
- Did the system check alternative causes?
- What tool calls did the model make?
- How much did the investigation cost?
- Where did the final confidence score come from?

## The Solution

The application lets a user select a synthetic software incident, run an investigation, inspect the generated report, and review the underlying trace.

The current demo supports three sample incidents:

- Database connection exhaustion after a checkout deployment
- API latency regression after a recommendation rollout
- Payment provider degradation during checkout

For each incident, a backend harness coordinates specialized agents:

- A **supervisor** creates the investigation plan and final report.
- An **evidence investigator** calls operational tools through an MCP interface.
- A **reliability reviewer** checks whether required citations are present and records caveats.
- A **human reviewer** can accept the report or provide feedback to run a revised investigation.

The final output is a structured report with a title, summary, root cause, confidence score, evidence citations, reviewer challenges, and recommendations.

## The Harness

The harness is the core of the project.

In this app, a harness means the code around the model that controls state, routing, tools, validation, review, observability, and fallback behavior. The LLM is only one part of the system.

The investigation state includes fields:

- `run_id`
- `title`
- `description`
- `scenario`
- `human_feedback`
- `evidence`
- `reviewer_challenges`
- `needs_more_evidence`
- `review_rounds`
- `report`
- `trace_events`

![Simplified harness trace flow](/images/simple-trace-flow.svg)

The workflow is explicit:

1. The **supervisor** creates an investigation plan.
2. The **evidence investigator** gathers logs, metrics, deployments, and alerts through MCP tools.
3. The **reviewer** checks whether required citations are present.
4. The graph conditionally routes back for more evidence when needed.
5. The **supervisor** generates a final report.


## The MCP Server

The MCP server is the boundary between the AI harness and operational data.

The project uses `FastMCP` to expose the tools:

- `search_logs`
- `get_service_metrics`
- `get_database_metrics`
- `get_deployment_history`
- `get_recent_alerts`
- `list_scenarios`

Each tool reads from local JSON fixtures under `mcp_server/fixtures`. The fixture store validates the scenario, loads the relevant JSON file, filters records by service/database/query, and returns structured dictionaries.

### MCP Server vs. Fixture Store

  `mcp_server/tools/fixture_store.py` contains the actual implementation:

  - Reads local JSON fixture files.
  - Validates the selected scenario.
  - Filters records by service, database, or query.
  - Returns Python lists/dicts.
  - Can be called directly by backend code or tests.

![MCP_fixture](/images/MCP_fixture.png)

  Example: `search_logs(...)` here actually loads `logs.json` and filters records.

  `mcp_server/server.py` contains the MCP server interface:

  - Creates the `FastMCP` server.
  - Registers functions as MCP tools with `@mcp.tool()`.
  - Gives external MCP clients a standardized tool surface.
  - Delegates the real work to `fixture_store.py`.

![MCP_tool](/images/MCP_tool.png)

The model does not directly inspect arbitrary files or databases. It interacts with a constrained tool interface. That makes the system easier to reason about:

- Tool inputs are explicit.
- Tool outputs are structured.
- Evidence records include stable citations.
- The trace can show each tool invocation and result count.

For example, the database exhaustion scenario includes citations such as:

```text
logs.json#log-002
database_metrics.json#db-001
database_metrics.json#db-002
deployments.json#dep-001
```

The reliability reviewer checks for required citations before accepting the evidence base. If a required citation is missing, the reviewer records a challenge instead of silently trusting the conclusion.



When `langgraph` is installed, the harness uses a `StateGraph`. The project also has a fallback runner that executes the same staged workflow when LangGraph is unavailable in local development.

### StateGraph (warning: this is the part I really don't understand yet, it gets complicated quickly)

  `StateGraph` is LangGraph’s way of defining the investigation as an explicit state machine. Each node reads from and writes to a shared typed state object, and edges define what step runs next.

  In this app, `StateGraph` connects **(nodes)** supervisor planning, MCP evidence gathering, reliability review, and final report generation. The **edges** are transitions between nodes (edges are actions) in this case "review", "investigate", "finalize". 
