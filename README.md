# CareerPath AI — Student Counselling Agent

> **An AI-powered career counselling system built with n8n, GPT-4o Mini, Supabase RAG, and Google Sheets — delivering personalised 12-month career roadmaps to tech and non-tech students in under 15 minutes.**

![Platform](https://img.shields.io/badge/Platform-n8n-orange?style=flat-square)
![AI](https://img.shields.io/badge/AI-GPT--4o%20Mini-412991?style=flat-square&logo=openai)
![DB](https://img.shields.io/badge/RAG-Supabase-3ECF8E?style=flat-square&logo=supabase)
![License](https://img.shields.io/badge/License-MIT-blue?style=flat-square)
![Status](https://img.shields.io/badge/Status-Live-brightgreen?style=flat-square)

---

## Table of Contents

- [Overview](#-overview)
- [Live Demo](#-live-demo)
- [Architecture](#-architecture)
- [Workflow Breakdown](#-workflow-breakdown)
- [Tech Stack](#-tech-stack)
- [Repository Structure](#-repository-structure)
- [Setup & Installation](#-setup--installation)
- [Configuration Reference](#-configuration-reference)
- [RAG Knowledge Base](#-rag-knowledge-base)
- [Agent Behaviour](#-agent-behaviour)
- [Google Sheets CRM](#-google-sheets-crm)
- [Frontend (index.html)](#-frontend-indexhtml)
- [Screenshots](#-screenshots)
- [FAQ](#-faq)
- [License](#-license)

---

## Overview

**CareerPath AI** is a fully automated, multi-agent career counselling system built by [Regex Software Services](https://github.com/navneetsxngh). A student opens a chat widget on the website, shares their name, email, and educational background — and within minutes receives a structured, phase-by-phase, 12-month career roadmap powered by a verified Supabase vector knowledge base.

The system routes students to one of two specialist AI agents:

| Track | For | Counsellor Agent |
|-------|-----|-----------------|
| Tech | CSE, ECE, Mechanical, Civil, IT, Electrical | **Tech Counsellor** (Supabase `tech` table) |
| Non-Tech | Commerce, Arts, Law, Psychology, MBA, Mass Comm | **Non-Tech Counsellor** (Supabase `nontech` table) |

**Key stats from production:**
```json
{
  "students_guided": "5,200+",
  "specialist_agents": 2,
  "avg_session_time_minutes": 12,
  "satisfaction_rate": "98%"
}
```

---

## Live Demo

The frontend is a single-file `index.html` — open it in any browser or serve it statically. The chat widget connects to an n8n webhook in real time.

```
Webhook URL: https://devendradebu.app.n8n.cloud/webhook/<your-id>/chat
```

> Replace the webhook URL in `index.html` with your own n8n production webhook before deploying.

---

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                        Student Browser                       │
│                  index.html  (Chat Widget)                   │
└──────────────────────────┬──────────────────────────────────┘
                           │  HTTP POST (sessionId + message)
                           ▼
┌─────────────────────────────────────────────────────────────┐
│                    n8n Cloud Workflow                       │
│                                                             │
│  ┌──────────────┐    ┌────────────────────┐                 │
│  │ Chat Trigger │───▶│ Basic Info Agent   │                 │
│  │  (Webhook)   │    │  GPT-4o Mini       │                 │
│  └──────────────┘    │  Collects: name,   │                 │
│                      │  email, background │                 │
│                      └────────┬───────────┘                 │
│                               │                             │
│                      ┌────────▼───────────┐                 │
│                      │  Status Check +    │                 │
│                      │  Google Sheets Log │                 │
│                      └────────┬───────────┘                 │
│                               │                             │
│                    ┌──────────▼──────────┐                  │
│                    │   IF: Tech or        │                 │
│                    │   Non-Tech?          │                 │
│                    └──────┬──────┬───────┘                  │
│                           │      │                          │
│              ┌────────────▼┐    ┌▼──────────────┐           │
│              │ Tech Agent  │    │ Non-Tech Agent │           │
│              │ GPT-4o Mini │    │ GPT-4o Mini    │           │
│              │ Supabase    │    │ Supabase       │           │
│              │ (tech table)│    │ (nontech table)│           │
│              └────────────┬┘    └┬──────────────┘           │
│                           │      │                           │
│                    ┌──────▼──────▼───────┐                  │
│                    │  12-Month Roadmap    │                 │
│                    │  Delivered to User  │                  │
│                    └─────────────────────┘                  │
└─────────────────────────────────────────────────────────────┘
```

---

## Workflow Breakdown

The n8n workflow consists of **5 core nodes**:

### Node 1 — Chat Trigger (Webhook)
```json
{
  "node": "Chat Trigger",
  "type": "n8n-nodes-base.webhook",
  "mode": "chatTrigger",
  "accepts": ["sessionId", "message"],
  "authentication": "none"
}
```
Listens for incoming chat messages. Each conversation is identified by a unique `sessionId` generated by the frontend.

---

### Node 2 — Basic Info Collector Agent
```json
{
  "node": "Basic Info Agent",
  "model": "gpt-4o-mini",
  "collects": ["name", "email", "background_type"],
  "context_window": 10,
  "behaviour": "Asks one question at a time. Validates email format. Confirms background before routing."
}
```
A friendly intake agent that gathers the student's basic information conversationally. It uses a 10-message memory window to maintain context and never repeats questions already answered.

**System Prompt excerpt:**
> *"You are a warm and professional intake assistant for Regex Software Services. Collect the student's full name, email address, and whether they come from a Tech or Non-Tech background — one question at a time."*

---

### Node 3 — Status Check + Google Sheets Log
```json
{
  "node": "Status Check",
  "logic": "IF intake_complete == true → route to specialist",
  "on_complete": "Append row to Google Sheet",
  "tracked_fields": ["name", "email", "background", "session_id", "timestamp", "routed_to"]
}
```
Evaluates whether the intake form is complete. If yes, it logs the student's data to a Google Sheet and hands off to the correct specialist agent.

---

### Node 4a — Tech Counsellor Agent
```json
{
  "node": "Tech Counsellor",
  "model": "gpt-4o-mini",
  "rag_source": "Supabase",
  "table": "tech_knowledge",
  "serves": ["CSE", "ECE", "Mechanical", "Civil", "IT", "Electrical"],
  "output": "12-month roadmap with 4 phases, certifications, salary benchmarks"
}
```

### Node 4b — Non-Tech Counsellor Agent
```json
{
  "node": "Non-Tech Counsellor",
  "model": "gpt-4o-mini",
  "rag_source": "Supabase",
  "table": "nontech_knowledge",
  "serves": ["Commerce", "Arts", "Law", "Psychology", "MBA", "Mass Comm"],
  "output": "Bridge-into-tech roadmap with domain-specific pivot strategies"
}
```

Both agents query their respective Supabase vector tables using the student's stated career goal, then compose a personalised roadmap response.

---

## Tech Stack

| Layer | Technology | Purpose |
|-------|------------|---------|
| **Workflow** | [n8n](https://n8n.io) | Visual automation & agent orchestration |
| **AI Model** | GPT-4o Mini (OpenAI) | Conversational intelligence for all agents |
| **Vector DB** | [Supabase](https://supabase.com) + pgvector | RAG knowledge base for career data |
| **CRM** | Google Sheets | Session logging and student tracking |
| **Frontend** | Vanilla HTML/CSS/JS | Chat widget & landing page |
| **Fonts** | Cormorant Garamond + Syne | Typography |
| **Hosting** | Any static host (Vercel, Netlify, GitHub Pages) | Frontend delivery |

---

## Repository Structure

```
Student-Counselling-Agent-using-n8n/
│
├── index.html                  # Full frontend — landing page + chat widget
│
├── workflow/
│   └── *.json                  # Exported n8n workflow file(s) — import directly into n8n
│
├── RAG Documents/
│   ├── tech_careers.pdf        # Knowledge base for tech track (CSE, ECE, etc.)
│   └── nontech_careers.pdf     # Knowledge base for non-tech track (Commerce, Arts, etc.)
│
├── Screenshots/
│   └── *.png                   # Workflow and UI screenshots
│
└── LICENSE                     # MIT License
```

---

## Setup & Installation

### Prerequisites

- [n8n](https://docs.n8n.io/hosting/) — self-hosted **or** n8n Cloud account
- OpenAI API key (GPT-4o Mini access)
- Supabase project with `pgvector` enabled
- Google account (for Sheets integration)

---

### Step 1 — Clone the Repository

```bash
git clone https://github.com/navneetsxngh/Student-Counselling-Agent-using-n8n.git
cd Student-Counselling-Agent-using-n8n
```

---

### Step 2 — Set Up Supabase (RAG Knowledge Base)

1. Create a new [Supabase](https://supabase.com) project.
2. Enable the `pgvector` extension:
```sql
CREATE EXTENSION IF NOT EXISTS vector;
```
3. Create two tables for vector storage:
```sql
-- Tech track knowledge base
CREATE TABLE tech_knowledge (
  id BIGSERIAL PRIMARY KEY,
  content TEXT,
  embedding VECTOR(1536),
  metadata JSONB
);

-- Non-tech track knowledge base
CREATE TABLE nontech_knowledge (
  id BIGSERIAL PRIMARY KEY,
  content TEXT,
  embedding VECTOR(1536),
  metadata JSONB
);
```
4. Upload the PDF documents from `RAG Documents/` using the n8n Supabase Vector Store node (or use the Supabase dashboard with an embedding pipeline).

---

### Step 3 — Import the n8n Workflow

1. Open your n8n instance.
2. Go to **Workflows → Import**.
3. Upload the JSON file from the `workflow/` folder.
4. Connect your credentials:
   - **OpenAI API** → Add your OpenAI key
   - **Supabase** → Add your Supabase URL + service role key
   - **Google Sheets** → Authenticate via OAuth2
5. Activate the workflow. Copy the **Production Webhook URL**.

---

### Step 4 — Configure the Frontend

Open `index.html` and replace the webhook URL on **line ~1100**:

```javascript
// BEFORE
const N8N_WEBHOOK_URL = 'https://devendradebu.app.n8n.cloud/webhook/a90adbf0-facb-4ca0-87ff-2ac832fc78c7/chat';

// AFTER — paste your own webhook URL
const N8N_WEBHOOK_URL = 'https://YOUR-N8N-INSTANCE/webhook/YOUR-WEBHOOK-ID/chat';
```

---

### Step 5 — Deploy the Frontend

Deploy `index.html` to any static host:

```bash
# Vercel (one command)
npx vercel --prod

# Netlify drag-and-drop
# Simply drag index.html into app.netlify.com

# GitHub Pages
# Push to repo → Settings → Pages → Deploy from branch
```

---

## Configuration Reference

All key configuration lives inside the n8n workflow JSON. Below is a summary:

```json
{
  "workflow_name": "Student Counselling Agent",
  "trigger": "Chat Webhook (Public, no auth)",
  "agents": {
    "intake": {
      "model": "gpt-4o-mini",
      "memory_window": 10,
      "goal": "Collect name, email, background_type"
    },
    "tech_counsellor": {
      "model": "gpt-4o-mini",
      "rag_table": "tech_knowledge",
      "top_k": 5,
      "output_format": "4-phase 12-month roadmap"
    },
    "non_tech_counsellor": {
      "model": "gpt-4o-mini",
      "rag_table": "nontech_knowledge",
      "top_k": 5,
      "output_format": "4-phase 12-month roadmap with pivot strategy"
    }
  },
  "crm": {
    "provider": "Google Sheets",
    "columns": ["Timestamp", "Name", "Email", "Background", "Session ID", "Routed To"]
  }
}
```

---

## 📚 RAG Knowledge Base

The `RAG Documents/` folder contains the source documents used to populate the Supabase vector store. These are chunked, embedded (using OpenAI `text-embedding-ada-002`), and stored in Supabase via the n8n **Supabase Vector Store** node.

### Tech Track covers:
- Career paths for CSE, ECE, Mechanical, Civil, IT, Electrical graduates
- Certifications: AWS, GCP, Azure, Cisco, CompTIA, etc.
- Salary benchmarks (fresher → 3 years → 5 years)
- Role-specific skill stacks (SDE, DevOps, Embedded, Data Engineer, etc.)

### Non-Tech Track covers:
- Bridge-into-tech paths for Commerce, Arts, Law, Psychology, MBA students
- Domain crossovers: FinTech, LegalTech, EdTech, MarTech
- Entry-level roles that don't require coding (Business Analyst, Product Manager, UX Researcher, etc.)
- Objection-handling scripts ("I can't code", "Is it too late?")

---

## Agent Behaviour

### Intake Agent Prompt Logic
```
1. Greet the student warmly
2. Ask for their full name
3. Ask for their email address (validate format)
4. Ask: "Are you from a Tech or Non-Tech background?"
5. Confirm the details and hand off to the specialist
```

### Tech Counsellor Output Structure
```
Phase 1 (Months 1-3):   Foundation
Phase 2 (Months 4-6):   Skill Building & Certifications
Phase 3 (Months 7-9):   Real-world Experience
Phase 4 (Months 10-12): Launch & Job Search
+ Recommended certifications (with links)
+ Salary expectations at each phase
+ Immediate next action step
```

### Non-Tech Counsellor Output Structure
```
Your Background Advantage: [How your degree helps]
Bridge Strategy:           [How to cross into tech]
Phase 1-4 Roadmap:         [Same 4-phase structure]
Pivot Roles:               [3-5 realistic entry roles]
First 30-day Action Plan:  [Concrete steps this week]
```

---

## Google Sheets CRM

Every completed intake session is automatically appended to a Google Sheet with the following columns:

| Column | Example Value |
|--------|---------------|
| `Timestamp` | 2025-11-14 10:32:05 |
| `Session ID` | sess_abc123xyz |
| `Name` | Rahul Khatri |
| `Email` | rahul@example.com |
| `Background` | Tech |
| `Routed To` | Tech Counsellor |

This gives you a live CRM of every student who has engaged with the system, usable for follow-up campaigns or analytics.

---

## Frontend (index.html)

The landing page is a single self-contained HTML file (~50KB) with no build step or dependencies required.

### UI Sections

| Section | Description |
|---------|-------------|
| **Hero** | Headline, sub-copy, primary CTA to open chat |
| **Stats Bar** | Live counters: 5,200+ students, 2 agents, 12-min avg, 98% satisfaction |
| **Who We Serve** | Two-column cards for Tech vs Non-Tech tracks |
| **How It Works** | 4-step explainer + live workflow diagram |
| **Features** | 6 capability cards (RAG, routing, memory, CRM, roadmaps, objection handling) |
| **Testimonials** | 3 student story cards |
| **CTA Banner** | Final conversion section |
| **Chat Widget** | Floating button → full chat window powered by n8n |

### Chat Widget Features
- Floating Action Button (FAB) with notification badge
- Animated chat window with typing indicator
- Quick-reply chips for common questions
- Auto-growing textarea input
- Unique `sessionId` generated per browser session
- Powered-by footer: *n8n · GPT-4o Mini · Supabase Vector RAG*

---

## Screenshots

> Screenshots are available in the [`Screenshots/`](./Screenshots/) folder of the repository.

| View | Description |
|------|-------------|
| `workflow.png` | Full n8n workflow canvas |
| `chat_ui.png` | Chat widget open on landing page |
| `roadmap_output.png` | Example counsellor response with 12-month roadmap |
| `sheets_crm.png` | Google Sheets CRM with live student data |

---

## FAQ

**Q: Can I self-host n8n instead of using n8n Cloud?**  
Yes. Install n8n via Docker or npm, import the workflow JSON, and update the webhook URL in `index.html`. All functionality is identical.

**Q: What OpenAI model is used?**  
GPT-4o Mini — chosen for its balance of speed, cost, and quality for multi-turn conversational tasks.

**Q: How do I add more knowledge to the RAG base?**  
Add documents (PDF/text) to the `RAG Documents/` folder, then re-run the embedding pipeline in n8n using the Supabase Vector Store node with your new files.

**Q: Is the webhook secured?**  
The current setup uses a public webhook with no authentication. For production, add n8n's built-in HTTP Basic Auth or Bearer Token validation to the webhook node.

**Q: Can I add more tracks (e.g., a Healthcare track)?**  
Yes — duplicate either counsellor agent node, create a new Supabase table for healthcare knowledge, update the routing logic in the IF node, and add the new track to the intake agent's prompt.

---

## License

This project is licensed under the **MIT License** — see [`LICENSE`](./LICENSE) for details.

```
Copyright (c) 2026 Navneet Singh
```

---

<div align="center">

Built with by **Navneet Singh** · [Regex Software Services](https://github.com/navneetsxngh)

**Stack:** n8n · GPT-4o Mini · Supabase · Google Sheets · Vanilla HTML

</div>
