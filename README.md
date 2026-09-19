# Floor Agent AI — Shop-Floor AI Agent

A tool-calling AI agent that helps shop-floor operators verify panels, follow SOPs, and safely escalate uncertainty — built with **n8n**, **Google Gemini**, and **n8n Data Tables**.

> Built as a practical assessment for a Junior AI Engineer role at ABC Cabinet. Simulates connecting cabinet production data with shop-floor operations, without requiring factory access or real company data.

---

## 🧠 What It Does

At a workstation, an operator can:

1. Select a workstation (e.g. Edge Banding, Drilling)
2. Enter or select a panel code (simulated scan)
3. View grounded panel information — panel code, cabinet ID, name, dimensions, material, required operation
4. Receive workstation-specific instructions pulled from SOP data
5. Ask free-text questions about the panel or SOP
6. See which tools/data sources the agent used to answer
7. Trigger a supervisor escalation when something can't be safely resolved

The core design goal: **this is an agent, not a chatbot.** The LLM decides which tools to call based on the input — it isn't a fixed, hard-coded sequence.

```
Operator Input → Agent Decides → Calls Tool(s) → Reads Result → Decides Next Step → Responds
```

---

## 🏗️ Architecture

**Stack:** n8n (AI Agent node, Tools Agent mode) · Google Gemini (chat model) · n8n Data Tables (structured mock data) · Structured Output Parser

Each tool is built as its **own standalone sub-workflow** (e.g. `TOOL - get_panel`, `TOOL - Get Workstation`, `TOOL - Search SOP`, `TOOL - Record Event`, `TOOL - Escalate Supervisor`), each backed by its own dedicated Data Table, and called from the main `ABC cabinet` workflow via the Tool Workflow connector on the AI Agent node. This keeps each tool independently testable and reusable outside the main flow.

```
Main workflow: "ABC cabinet"

[Form Trigger: workstation, panel code, question]
                    ↓
             [AI Agent (Gemini)]
   ┌────────────┬──────────────────────┬──────────────┬──────────────────┬───────────────────────────┐
TOOL - get_panel TOOL - Get Workstation TOOL - Search SOP TOOL - Record Event TOOL - Escalate Supervisor
(each a sub-workflow → own Data Table)
                    ↓
        [Structured Output Parser]
                    ↓
           [If: escalated?] → two response branches (Form Ending)
```

### Tools

| Tool | Purpose |
|---|---|
| `get_panel(panel_code)` | Retrieves panel specs (cabinet ID, name, dimensions, material, required operation) from structured mock data. Returns empty if the panel code doesn't exist. |
| `get_workstation_requirements(workstation_name)` | Retrieves supported operations for a workstation, used to cross-check panel-to-workstation routing. |
| `search_sop(query)` | Keyword-based lookup against stored SOP content. No embeddings or vector DB — kept intentionally simple. |
| `record_event(event_type, panel_code, details)` | Logs every scan, question, or escalation to a history table. |
| `escalate_to_supervisor(reason, panel_code)` | Triggered whenever data is missing, mismatched, or a question falls outside available SOP/panel data. |

### Grounding & Safety Rules (enforced via system prompt)

- Never state panel facts or operational instructions that didn't come from a tool call
- Never invent machine settings, speeds, tooling parameters, or safety procedures
- If SOP/panel data doesn't support an answer, escalate — don't guess
- Always cite a source, e.g. `Source: Panel P-1002` or `Source: SOP - Edge Banding`
- Call `record_event` exactly once per conversation, after reaching a conclusion

### Output Schema

Every agent response is returned as structured JSON via a Structured Output Parser, so downstream logic (routing, history, UI) doesn't have to parse free text:

```json
{
  "status": "ok | mismatch | not_found | escalated",
  "message": "Human-readable message for the operator",
  "panel_code": "P-1002",
  "workstation": "EDGE-01",
  "source": "Panel P-1002; Workstation Requirements - EDGE-01",
  "escalated": false,
  "event_logged": true
}
```

---

## ✅ Test Results

| # | Test Case | Scenario | Result |
|---|---|---|---|
| 1 | Correct Workstation | Edge Banding panel scanned at Edge Banding workstation | ✅ Verifies panel + workstation, returns grounded instructions with source |
| 2 | Wrong Workstation | Drilling panel scanned at Edge Banding workstation | ✅ Detects mismatch, blocks processing, states correct workstation |
| 3 | Unsupported Question | "What spindle speed should I use?" | ✅ No hallucination — escalates to supervisor |
| 4 | Unknown Panel | Panel code that doesn't exist | ✅ Returns "Panel Not Found," invents nothing |
| 5 | Supervisor Escalation | Physical label vs. system data conflict | ✅ Recognizes unsafe-to-resolve case, triggers escalation |

---

## 🧩 Data Model (n8n Data Tables)

Each tool has its own dedicated Data Table:

| Table | Columns | Used by |
|---|---|---|
| `get_panel` | 9 columns — panel_code, cabinet_id, panel_name, width_mm, height_mm, thickness_mm, material, required_operation, + metadata | `get_panel` tool |
| `get_workstation_requirements` | 5 columns — workstation_name, workstation_id, supported_operations, + metadata | `get_workstation_requirements` tool |
| `search_sop` | 4 columns — workstation, topic, instructions, + metadata | `search_sop` tool |
| `record_event` | 7 columns — timestamp, event_type, panel_code, workstation, details, result, + metadata | `record_event` tool |
| `escalate_to_supervisor` | 7 columns — timestamp, reason, panel_code, workstation, details, status, + metadata | `escalate_to_supervisor` tool |

---

## 🛠️ Key Engineering Challenges Solved

- **Hard-coded pipeline → real agent.** An early version fetched panel/workstation data unconditionally and used a single-shot LLM chain instead of genuine tool selection. Rebuilt around n8n's AI Agent node so the model decides per-request which tools are needed.
- **"Max iterations reached" infinite loop.** Root cause: `record_event` was set to an *Update* operation (which requires a match condition) instead of *Insert*, so it errored on every call and the agent kept retrying. Fixed the operation type and added a "call once per conversation" rule to the system prompt.
- **SOP search returning nothing.** An AI-fillable toggle was mistakenly applied to the filter's *column name* instead of its *value*, so the query text was being treated as a table column. Fixed by making the column fixed and the value AI-controlled.
- **False-positive panel matches.** The `get_panel` tool had no filter condition at all, so it returned *every* row regardless of input — causing invalid panel codes to appear "confirmed." Fixed by adding an exact-match condition on `panel_code`.
- **Structured, reliable output.** Added a Structured Output Parser with a fixed schema so the workflow can branch reliably (success vs. mismatch vs. escalation) instead of parsing free text.

---

## 🚫 Not Included (By Design)

Per assessment scope — kept deliberately simple:

- No authentication or user management
- No vector databases / embeddings / advanced RAG
- No multi-agent systems or MCP
- No Docker / Kubernetes / cloud infrastructure
- No real barcode scanner or factory integration (panel scanning is simulated via form input)

---

## 📦 Running This Project

> **Note:** n8n's workflow export only includes workflow structure (nodes, connections, parameters) — it does **not** include Data Table contents. Workflows and Data Tables are exported/imported separately.

**1. Import the workflows** — this project is 6 separate n8n workflows:
   - `ABC cabinet` — the main workflow (Form Trigger → AI Agent → response branches)
   - `TOOL - get_panel`
   - `TOOL - Get Workstation`
   - `TOOL - Search SOP`
   - `TOOL - Record Event`
   - `TOOL - Escalate Supervisor`

   Import all 6 into your n8n instance, then re-link each Tool Workflow node in the `ABC cabinet` workflow's AI Agent to the corresponding imported sub-workflow (n8n references sub-workflows by ID, so this link needs to be reconnected after import).

**2. Create 5 Data Tables** with these exact names so each tool's node resolves correctly:
   - `get_panel` (9 columns)
   - `get_workstation_requirements` (5 columns)
   - `search_sop` (4 columns)
   - `record_event` (7 columns)
   - `escalate_to_supervisor` (7 columns)

   See the [Data Model](#-data-model-n8n-data-tables) section above for column details.

**3. Seed the reference tables** using the corresponding files in `/data`:
   - `data/get_panel.json`
   - `data/get_workstation_requirements.json`
   - `data/search_sop.json`

   `record_event` and `escalate_to_supervisor` don't need seeding — they populate at runtime as the agent logs scans, questions, and escalations.

**4. Add a Google Gemini credential** (free tier via [Google AI Studio](https://aistudio.google.com)) and connect it to the AI Agent's Chat Model input in the `ABC cabinet` workflow.

**5. Open the Form Trigger's production URL** to use the app.

---

## 📄 License

This is a portfolio / assessment project using entirely fictional data. No proprietary or real company data is included.
