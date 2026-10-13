# Agents as Tools — Component Flow

This document shows how agents and tools connect to each other in the three patterns demonstrated in the notebook.

---

## Pattern 1: Orchestrator with Specialist Agents as Tools

The primary pattern: an orchestrator agent delegates work to specialist sub-agents that are each wrapped as a `@tool`.

```mermaid
graph TD
    User([User Query]) --> Orchestrator

    subgraph Orchestrator Agent
        Orchestrator["🤖 Orchestrator\n(us.amazon.nova-pro-v1:0)\nRoutes queries to the right specialist"]
    end

    subgraph Tools registered on Orchestrator
        T1["🔧 research_assistant\n(@tool)"]
        T2["🔧 product_recommendation_assistant\n(@tool)"]
        T3["🔧 trip_planning_assistant\n(@tool)"]
        T4["🔧 file_write\n(strands_tools)"]
    end

    Orchestrator -->|"research question"| T1
    Orchestrator -->|"product inquiry"| T2
    Orchestrator -->|"travel planning"| T3
    Orchestrator -->|"write result to disk"| T4

    subgraph Specialist Agents spawned inside each tool
        A1["🤖 research_agent\n(us.amazon.nova-pro-v1:0)\nFactual info & citations"]
        A2["🤖 product_agent\n(us.amazon.nova-pro-v1:0)\nPersonalised recommendations"]
        A3["🤖 travel_agent\n(us.amazon.nova-pro-v1:0)\nDetailed itineraries"]
    end

    T1 --> A1
    T2 --> A2
    T3 --> A3

    A1 -->|str response| T1
    A2 -->|str response| T2
    A3 -->|str response| T3

    T1 -->|tool result| Orchestrator
    T2 -->|tool result| Orchestrator
    T3 -->|tool result| Orchestrator
    T4 -->|tool result| Orchestrator

    Orchestrator --> FinalResponse([Final Response to User])
```

---

## Pattern 2: Single-Query Tool Dispatch

How the orchestrator handles a single-domain query (e.g. "I'm looking for hiking boots").

```mermaid
sequenceDiagram
    actor User
    participant Orch as Orchestrator Agent
    participant Tool as product_recommendation_assistant (@tool)
    participant Agent as product_agent (nova-pro)

    User->>Orch: "I'm looking for hiking boots"
    Note over Orch: Identifies query type → product recommendation
    Orch->>Tool: call product_recommendation_assistant(query)
    Tool->>Agent: Agent("I'm looking for hiking boots")
    Agent-->>Tool: response (AgentResult)
    Tool-->>Orch: str(response)
    Orch-->>User: Formatted recommendations
```

---

## Pattern 3: Multi-Tool Dispatch (Parallel Specialists)

When a query spans multiple domains the orchestrator calls several specialist tools in sequence and combines the results.

```mermaid
sequenceDiagram
    actor User
    participant Orch as Orchestrator Agent
    participant R as research_assistant (@tool)
    participant RA as research_agent (nova-pro)
    participant T as trip_planning_assistant (@tool)
    participant TA as travel_agent (nova-pro)

    User->>Orch: "Research Spain and plan a 7-day trip"
    Note over Orch: Query spans research + travel domains

    Orch->>R: call research_assistant("Research Spain")
    R->>RA: Agent("Research Spain")
    RA-->>R: research result
    R-->>Orch: str(research result)

    Orch->>T: call trip_planning_assistant("Plan 7-day Spain trip")
    T->>TA: Agent("Plan 7-day Spain trip")
    TA-->>T: itinerary
    T-->>Orch: str(itinerary)

    Orch-->>User: Combined research + itinerary response
```

---

## Pattern 4: Sequential Agent Pipeline (Research → Summary)

Agents chained directly (without an orchestrator) so the output of one feeds the input of the next.

```mermaid
graph LR
    User([Topic: Generative AI]) --> RA

    subgraph Step 1 — Research
        RA["🤖 research_agent\n(claude-sonnet-4-5)\nGathers comprehensive info"]
    end

    RA -->|"research_response\n.message['content'][0]['text']"| SA

    subgraph Step 2 — Summarise
        SA["🤖 summary_agent\n(claude-sonnet-4-5)\nDistils key points"]
    end

    SA --> Output([Concise Summary])
```

---

## Component Inventory

| Component | Type | Model | Responsibility |
|---|---|---|---|
| `orchestrator` | `Agent` | nova-pro-v1:0 | Routes queries; owns all tools |
| `research_assistant` | `@tool` wrapping `Agent` | nova-pro-v1:0 | Factual research with citations |
| `product_recommendation_assistant` | `@tool` wrapping `Agent` | nova-pro-v1:0 | Product suggestions |
| `trip_planning_assistant` | `@tool` wrapping `Agent` | nova-pro-v1:0 | Travel itineraries |
| `file_write` | `strands_tools` built-in tool | — | Writes output to disk |
| `research_agent` (pipeline) | `Agent` | claude-sonnet-4-5 | Deep research for sequential pattern |
| `summary_agent` (pipeline) | `Agent` | claude-sonnet-4-5 | Summarises research output |
