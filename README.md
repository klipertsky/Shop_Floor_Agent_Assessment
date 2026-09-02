# Shop_Floor_Agent_Assessment

# 🏭 Shop-Floor AI Agent — ABC Cabinet

An AI-powered shop-floor assistant built with **n8n** to help operators identify panels, verify workstation compatibility, retrieve approved SOP instructions, record production events, and escalate issues to a supervisor when information cannot be safely verified.

This project was developed as a practical prototype for the **Junior AI Engineer Practical Assessment – Build a Shop-Floor AI Agent**.

---

## 📌 Project Overview

The Shop-Floor AI Agent follows an agentic workflow:

```text
Operator Input
      ↓
   n8n Webhook
      ↓
    AI Agent
      ↓
 ┌───────────────────────────────┐
 │        Available Tools        │
 │                               │
 │ • Get Panel                   │
 │ • Get Workstation Requirements│
 │ • Search SOP                  │
 │ • Record Event                │
 │ • Escalate to Supervisor      │
 └───────────────────────────────┘
      ↓
Structured Response
      ↓
    Operator UI
```

The AI Agent decides which tools are required based on the operator's request rather than following a completely hard-coded sequence.

---

## 🎯 Objectives

The system is designed to allow an operator to:

* Select a workstation
* Enter or scan a panel code
* Retrieve panel information
* Verify workstation compatibility
* Receive workstation-specific instructions
* Ask questions about a panel or SOP
* View the tools and data sources used
* Record important production events
* Escalate unresolved or safety-related issues

---

## 🤖 AI Agent Capabilities

The AI Agent can:

### 1. Identify a Panel

The agent uses the `get_panel` tool to retrieve structured production data.

Example:

```text
Panel Code: P-1001

Panel: Left Side Panel
Cabinet: CAB-001
Dimensions: 720 × 450 × 18 mm
Material: MDF
Required Operation: Edge Banding
```

---

### 2. Verify Workstation Compatibility

The agent checks the selected workstation using:

```text
get_workstation_requirements(workstation_id)
```

For example:

```text
Panel requires: Drilling
Selected workstation: Edge Banding

Result: WRONG_WORKSTATION
```

The agent will not instruct the operator to process the panel at an incompatible workstation.

---

### 3. Retrieve Approved SOP Instructions

When processing instructions are required, the agent calls:

```text
search_sop(query)
```

Instructions are only provided when supported by the available SOP data.

The agent is explicitly instructed **not to invent machine settings, speeds, tooling parameters, or safety procedures**.

---

### 4. Record Events

Important actions and decisions are recorded using:

```text
record_event(...)
```

Examples include:

* Panel scans
* Successful validation
* Wrong workstation
* Unknown panel
* Unsupported questions
* Errors
* Supervisor escalations

---

### 5. Escalate to a Supervisor

When information is missing, conflicting, or safety-critical, the agent can call:

```text
escalate_to_supervisor(...)
```

Example:

```text
Physical label does not match system information.

Status: ESCALATED
Action: Supervisor verification required
```

---

## 🛠️ Available Tools

| Tool                           | Purpose                                                    |
| ------------------------------ | ---------------------------------------------------------- |
| `get_panel`                    | Retrieves structured panel information                     |
| `get_workstation_requirements` | Retrieves workstation capabilities and accepted operations |
| `search_sop`                   | Searches approved SOP information                          |
| `record_event`                 | Records important shop-floor events                        |
| `escalate_to_supervisor`       | Creates a supervisor escalation                            |

---

## 📊 Mock Production Data

The prototype uses structured mock data instead of real production systems.

### Panels

| Panel Code | Cabinet | Panel            | Dimensions    | Material | Operation    |
| ---------- | ------- | ---------------- | ------------- | -------- | ------------ |
| P-1001     | CAB-001 | Left Side Panel  | 720×450×18 mm | MDF      | Edge Banding |
| P-1002     | CAB-002 | Door Panel       | 700×400×18 mm | Plywood  | Drilling     |
| P-1003     | CAB-003 | Right Side Panel | 720×450×18 mm | MDF      | Edge Banding |

### Workstations

| ID       | Workstation  | Accepted Operation |
| -------- | ------------ | ------------------ |
| EDGE-01  | Edge Banding | Edge Banding       |
| DRILL-01 | Drilling     | Drilling           |

---

## 🧠 Agent Decision Logic

The agent does not blindly execute every available tool.

It determines which information is required before responding.

For example:

```text
Operator:
"Can I process P-1002 at EDGE-01?"
```

The agent may perform:

```text
1. get_panel("P-1002")
2. get_workstation_requirements("EDGE-01")
3. Compare required operation with workstation capability
4. record_event(...)
5. Respond
```

Result:

```text
WRONG_WORKSTATION

Do not process this panel at the selected workstation.

P-1002 requires Drilling, while EDGE-01 is configured
for Edge Banding.
```

This demonstrates the required **multi-tool agentic behavior**.

---

## 🔐 Anti-Hallucination & Safety

Production information must come from structured data or approved SOP content.

The agent is instructed to:

* Never invent panel information
* Never invent dimensions or materials
* Never invent workstation capabilities
* Never invent machine settings
* Never invent spindle speeds
* Never invent feed rates
* Never invent tooling parameters
* Never invent safety procedures
* Never guess when information is unavailable
* Escalate when information is conflicting or safety-critical

If information cannot be verified, the agent responds:

> "That information is not available in the current panel/SOP data, so I won't guess. Please verify with a supervisor."

---

## 🧪 Test Cases

The prototype supports the following assessment scenarios.

### Test 1 — Correct Workstation

```text
Panel: P-1001
Workstation: EDGE-01
```

Expected:

```text
Status: APPROVED
```

The panel requires Edge Banding and the selected workstation supports Edge Banding.

---

### Test 2 — Wrong Workstation

```text
Panel: P-1002
Workstation: EDGE-01
```

Expected:

```text
Status: WRONG_WORKSTATION
```

The agent must prevent processing at the incorrect workstation.

---

### Test 3 — Unsupported Question

```text
Operator:
"What spindle speed should I use?"
```

Expected:

```text
Status: UNSUPPORTED
```

The agent must not invent a spindle speed.

---

### Test 4 — Unknown Panel

```text
Panel: P-9999
```

Expected:

```text
Status: PANEL_NOT_FOUND
```

The agent must not fabricate panel information.

---

### Test 5 — Physical Label Mismatch

```text
Operator:
"The physical label doesn't match the system information."
```

Expected:

```text
Status: ESCALATED
```

The issue is sent for supervisor verification.

---

## 🧾 Structured Output

The AI Agent returns a structured response containing:

```json
{
  "status": "APPROVED",
  "message": "Panel is compatible with the selected workstation.",
  "panel": {},
  "workstation": {},
  "instructions": [],
  "next_action": "Proceed according to the approved SOP.",
  "escalation": null,
  "sources": [],
  "tool_trace": []
}
```

Possible statuses:

```text
APPROVED
WRONG_WORKSTATION
PANEL_NOT_FOUND
UNSUPPORTED
ESCALATED
ERROR
```

---

## 🔎 Tool Trace

The system provides a simple execution trace without exposing private chain-of-thought.

Example:

```text
✓ get_panel("P-1001")
  Status: success
  Source: Panel P-1001

✓ get_workstation_requirements("EDGE-01")
  Status: success
  Source: Workstation EDGE-01

✓ search_sop("edge banding")
  Status: success
  Source: SOP - Edge Banding

✓ record_event(...)
  Status: success
```

This allows an operator or reviewer to understand which tools and sources were used.

---

## ⚙️ Technology Stack

* **n8n** — Workflow automation and AI orchestration
* **Gemini** — LLM for the AI Agent
* **n8n AI Agent** — Agentic tool selection and decision-making
* **Structured Output Parser** — Consistent JSON responses
* **n8n Data Tables / structured mock data** — Production and event data
* **Webhook** — Operator/API input

---

## 📁 Suggested Project Structure

```text
shop-floor-ai-agent/
│
├── README.md
│
├── workflows/
│   ├── shop-floor-ai-agent.json
│   ├── tool-get-panel.json
│   ├── tool-get-workstation.json
│   ├── tool-search-sop.json
│   ├── tool-record-event.json
│   └── tool-escalate-supervisor.json
│
├── data/
│   ├── panels.json
│   ├── workstations.json
│   └── sops.json
│
└── screenshots/
    ├── workflow.png
    ├── ai-agent.png
    └── tool-trace.png
```

---

## 🚀 Setup

### 1. Install / Run n8n

Create an n8n instance and open the n8n editor.

### 2. Import the Workflows

Import the workflow JSON files from the `workflows/` directory.

### 3. Configure the LLM

Configure the Gemini model credentials in n8n.

Do **not** commit API keys or credentials to GitHub.

### 4. Configure the Tools

Each tool should be available to the main AI Agent as a callable workflow.

The tool workflows should be published/active before testing.

### 5. Load Mock Data

Add the panel, workstation, and SOP data provided in the `data/` directory.

### 6. Test the Agent

Run the required test cases:

```text
P-1001 + EDGE-01
P-1002 + EDGE-01
P-9999
"What spindle speed should I use?"
Physical label mismatch
```

---

## 🧩 Agent Architecture

The main workflow follows:

```text
Webhook
   ↓
Input Normalization
   ↓
AI Agent
   │
   ├── Get Panel
   │
   ├── Get Workstation
   │
   ├── Search SOP
   │
   ├── Record Event
   │
   └── Escalate Supervisor
   ↓
Structured Output
   ↓
Webhook Response
```

The important design principle is that **the AI Agent decides which tools are necessary based on the request and available information**.

---

## 📝 Technical Decisions

### Why structured data instead of asking the LLM?

Production facts need to be deterministic and verifiable.

The LLM is responsible for reasoning about the request and selecting tools, while production data comes from structured sources.

### Why no vector database?

The assessment only requires a small SOP dataset. A simple keyword/structured search is sufficient for the prototype and keeps the architecture simple.

### How is hallucination prevented?

The system prompt explicitly prohibits unsupported production claims, while tools provide the authoritative panel, workstation, and SOP information.

### What happens when a tool fails?

The agent should not assume the missing result. It should return an error or escalate when the missing information prevents a safe decision.

---

## 📈 Future Improvements

If more development time were available, possible improvements would include:

* Real barcode/QR scanner integration
* Operator authentication
* Production database integration
* Real-time machine status
* Better SOP search
* Supervisor dashboard
* Persistent audit history
* Analytics and reporting
* Role-based permissions
* Improved operator UI
* Production deployment with monitoring

---

## ⏱️ Development Time

**Estimated development time:** [Add your actual time here]

---

##

## 👨‍💻 Author

**Klipert Tedlos**

Junior AI Engineer Candidate

##
