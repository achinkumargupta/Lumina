# Lumina
Running Lumina Locally — Complete Setup Guide
Prerequisites
Install these first if you don't have them:

Node.js 20+

# Check version
node --version  # should be v20 or higher
# Install via nvm if needed
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash
nvm install 20
nvm use 20
pnpm (this project requires pnpm, not npm or yarn)

npm install -g pnpm
pnpm --version  # confirm it's installed
PostgreSQL 16

# macOS (via Homebrew)
brew install postgresql@16
brew services start postgresql@16
# Ubuntu/Debian
sudo apt install postgresql-16
sudo systemctl start postgresql
Step 1 — Get the Code
Extract the downloaded archive:

tar -xzf lumina-knowledge-base.tar.gz
cd lumina-knowledge-base
Step 2 — Create the Database
# macOS
psql postgres -c "CREATE DATABASE lumina;"
# Ubuntu (switch to postgres user first)
sudo -u postgres psql -c "CREATE DATABASE lumina;"
Then get the connection string. It will look like:

postgresql://your_username@localhost:5432/lumina
On macOS with Homebrew, your username is usually your Mac login name. On Ubuntu it's postgres. Test it:

psql postgresql://your_username@localhost:5432/lumina -c "SELECT 1;"
Step 3 — Set Environment Variables
Create a .env file inside the lumina-knowledge-base folder:

cat > .env << 'EOF'
DATABASE_URL=postgresql://your_username@localhost:5432/lumina
AI_INTEGRATIONS_OPENAI_API_KEY=sk-your-openai-api-key-here
AI_INTEGRATIONS_OPENAI_BASE_URL=https://api.openai.com/v1
EOF
Get your OpenAI API key from https://platform.openai.com/api-keys

Step 4 — Install Dependencies
cd lumina-knowledge-base
pnpm install
This installs all packages across all workspaces (API server, frontend, DB library, etc.) in one go.

Step 5 — Push the Database Schema
This creates all the tables in your PostgreSQL database:

# Load env vars, then push schema
export $(cat .env | xargs)
pnpm --filter @workspace/db run push
You should see Drizzle confirm it created the documents, threads, and thread_messages tables (plus their enums).

Step 6 — Start the API Server
Open Terminal 1:

cd lumina-knowledge-base
export $(cat .env | xargs)
PORT=8080 pnpm --filter @workspace/api-server run dev
You should see:

Server listening on port 8080
Test it:

curl http://localhost:8080/api/documents
# Should return: []
Step 7 — Start the Frontend
Open Terminal 2:

cd lumina-knowledge-base
export $(cat .env | xargs)
PORT=19350 BASE_PATH=/ pnpm --filter @workspace/knowledge-base run dev
You should see:

VITE v7.x.x  ready in ...ms
➜  Local:   http://localhost:19350/
The Vite dev server already has a built-in proxy that forwards all /api requests to http://localhost:8080, so both services communicate automatically.

Step 8 — Open the App
Visit http://localhost:19350 in your browser.

You should see the Lumina welcome screen with an empty library. Click New Document to create your first document.

Quick Reference — All Commands Together
# Terminal 0 — one-time setup
tar -xzf lumina-knowledge-base.tar.gz
cd lumina-knowledge-base
pnpm install
export $(cat .env | xargs)
pnpm --filter @workspace/db run push
# Terminal 1 — API server (keep running)
export $(cat .env | xargs) && PORT=8080 pnpm --filter @workspace/api-server run dev
# Terminal 2 — Frontend (keep running)
export $(cat .env | xargs) && PORT=19350 BASE_PATH=/ pnpm --filter @workspace/knowledge-base run dev
Common Issues
Problem	Fix
pnpm: command not found	Run npm install -g pnpm
DATABASE_URL must be set	Make sure you ran `export $(cat .env
AI_INTEGRATIONS_OPENAI_BASE_URL must be set	Same — env vars must be exported in Terminal 1
ECONNREFUSED on API calls	API server isn't running — check Terminal 1
relation "documents" does not exist	Schema wasn't pushed — repeat Step 5
Port already in use	Change PORT=8080 or PORT=19350 to any free port (update both terminals to match)
Error: Use pnpm instead	You ran npm install — use pnpm install instead




## Prod support

Build a Production/UAT Issue Diagnosis Tool — a full-stack web app for SRE/ops engineers 
to diagnose infrastructure incidents using an AI assistant.
## Stack
- React + Vite frontend (TypeScript)
- Express backend (TypeScript)
- PostgreSQL with Drizzle ORM
- OpenAI GPT-4 for AI chat (streaming via SSE)
- React Query for data fetching
- Tailwind CSS + shadcn/ui components
- Wouter for client-side routing
## Visual Design
Dark ops/terminal theme:
- Background: #0a0a0f (near black)
- Primary accent: cyan (#06b6d4)
- Card backgrounds: #0f0f1a
- Borders: rgba(255,255,255,0.08)
- Monospace font for diagnostic output (JetBrains Mono or similar)
- Subtle cyan glow effects on active/focused elements
- Font: display font for headings (e.g. Syne or Inter), mono for terminal-like text
## Pages & Routes
1. `/` — Dashboard: stats (open/investigating/resolved session counts), quick-start 
   template cards (one per issue category), recent sessions list
2. `/new` — New Diagnosis: form with incident title input, environment selector 
   (Production/UAT/Development as clickable cards), category selector (Autosys/Polypaths/
   Microservice/Database/Network/Other as pill buttons), and a Diagnostic Skill selector 
   (see Skills system below)
3. `/sessions` — Session List: table/list of all sessions with status badge, env badge, 
   category, created date, and link to session
4. `/session/:id` — Session Detail: two-panel layout:
   - Left panel (280px): session metadata (id, title, env badge, category, created date), 
     status dropdown (open/investigating/resolved), active skill display with expandable 
     runbook viewer, assign/change skill picker
   - Right panel: AI chat interface with message history, streaming message input, 
     auto-scroll to latest message
5. `/skills` — Skills Management: grid of skill cards (name, category badge, systems tags, 
   description, "View Runbook" expandable), "New Skill" button opens create form, 
   edit/delete per card
## Database Schema (PostgreSQL + Drizzle)
Table: `skills`
- id (serial PK)
- name (text, not null)
- description (text)
- category (text) — "Autosys" | "Polypaths" | "Microservice" | "Database" | "Other"
- systems (text[]) — array of system names
- runbook (text) — full markdown runbook
- isDefault (boolean, default false)
- createdAt, updatedAt (timestamps)
Table: `conversations`
- id (serial PK)
- createdAt, updatedAt
Table: `messages`
- id (serial PK)
- conversationId (FK → conversations)
- role ("user" | "assistant" | "system")
- content (text)
- createdAt
Table: `diagnostic_sessions`
- id (serial PK)
- title (text, not null)
- environment ("production" | "uat" | "development")
- status ("open" | "investigating" | "resolved")
- category (text)
- skillId (FK → skills, nullable)
- conversationId (FK → conversations, nullable)
- createdAt, updatedAt
## REST API Endpoints
- GET/POST /api/skills
- GET/PUT/DELETE /api/skills/:id
- GET/POST /api/sessions
- GET/PATCH/DELETE /api/sessions/:id
- GET/POST /api/openai/conversations
- GET/DELETE /api/openai/conversations/:id
- GET /api/openai/conversations/:id/messages
- POST /api/openai/conversations/:id/messages  ← streams SSE response
- GET /api/healthz
## AI / Skills System
When a session has a skillId, fetch that skill and inject its runbook into the AI system 
prompt dynamically before every conversation. The system prompt should tell the AI it is 
an expert infrastructure engineer, and then include the runbook steps verbatim.
Base system prompt (always included):
"You are an expert infrastructure and operations engineer specializing in diagnosing 
production and UAT issues across Polypaths (fixed income analytics), Autosys (job 
scheduler), microservices, databases, and data pipelines. Guide the engineer through 
diagnosis methodically. Ask clarifying questions. Provide specific commands to run. 
Reference exact log locations, config files, and common failure patterns."
If a skill is attached, append:
"## Active Runbook: [skill name]
[full runbook markdown]
Follow this runbook's steps to guide the diagnosis."
Stream the AI response back as Server-Sent Events (SSE) with `data: {"chunk": "..."}` 
format. Save completed messages to the messages table.
## Seed 5 Default Skills
1. **Autosys Job Failure** (category: Autosys)
   Systems: Autosys Event Server, Autosys Scheduler, Job logs, Application server
   Runbook: Steps to check job status (autorep -j), identify ON_ICE/zombie jobs, 
   check job logs, verify dependencies, restart/force-start jobs, escalation paths.
2. **Polypaths Calculation Error** (category: Polypaths)
   Systems: Polypaths app server, Market data feed, Database, Upstream data sources
   Runbook: Steps to check calculation logs, verify market data currency, check DB 
   connectivity, validate curve/instrument config, test calculation manually, resolutions 
   for missing data/stale prices/curve failures.
3. **Microservice 5xx / Outage** (category: Microservice)
   Systems: API Gateway, Load balancer, Service instances, Health check endpoints, 
   Application logs, APM/Monitoring
   Runbook: Check health endpoints, check logs for errors, check JVM heap/disk/memory, 
   check DB connection pool, check upstream dependencies, check recent deployments. 
   Resolutions for OOM/connection pool exhaustion/bad deployment/502-504 errors.
4. **Database Blocking / Deadlock** (category: Database)
   Systems: Database server, Application logs, DBA tools, Monitoring
   Runbook: SQL queries for SQL Server/Oracle/PostgreSQL to find blocking sessions and 
   long-running queries, how to kill blocking sessions, check connection pool, 
   resolutions for deadlocks/long queries/disk full.
5. **Data Pipeline Failure** (category: Other)
   Systems: Message queue (Kafka/MQ/RabbitMQ), ETL scheduler, Source systems, 
   Target database, Pipeline logs
   Runbook: Check pipeline status in scheduler, check Kafka consumer lag, verify source 
   system connectivity, check target DB, identify data quality issues (nulls/schema 
   mismatch/duplicates). Resolutions for consumer lag/poison messages/backfill.
## Status & Environment Badges
- open → amber badge
- investigating → blue badge
- resolved → green badge
- production → red badge
- uat → orange badge
- development → gray badge
## New Session Form — Skill Selector Behavior
Show all skills as selectable cards below the category selector. Clicking a card selects 
it (highlighted cyan border). A "None" option appears first. Filter displayed skills to 
match the chosen category when possible. Support ?skill=<id> URL param to preselect. 
Pass skillId in the POST /api/sessions payload.
## Session Detail — Skill Panel
In the left metadata panel, show:
- If skill assigned: skill name with BookOpen icon, systems as small monospace tags, 
  "View Runbook" toggle to expand full markdown-rendered runbook, "Change Skill" button 
  that opens a popover/dropdown to pick a different skill (calls PATCH /api/sessions/:id)
- If no skill: "No Skill Selected" with "Assign Skill" button
## Skills Page — Card Features
- Search/filter bar at top
- Category badge colors: Autosys=cyan, Polypaths=purple, Microservice=green, 
  Database=amber, Other=gray
- Systems shown as small monospace chip tags
- "View Runbook" expands inline below the card, renders markdown with react-markdown
- "New Skill" → inline create form or modal: name, description, category select, 
  systems (comma-separated input, rendered as tags), runbook (large monospace textarea)
- Edit button per card
- Delete button (with confirmation dialog) — disabled for isDefault skills
- isDefault skills show a lock icon or "Built-in" badge
## Component Architecture
- `Layout.tsx` — sidebar nav with links: Dashboard, All Sessions, Skills, New Diagnosis
- `ChatInterface.tsx` — handles SSE streaming, auto-scroll, message list rendering
- `Badges.tsx` — StatusBadge and EnvBadge reusable components
- Pages: dashboard.tsx, new-session.tsx, session-detail.tsx, session-list.tsx, skills.tsx






Tier 1 — Changes the game
1. Live Tool Execution (the big one)
Right now the AI tells you what commands to run — you go run them manually, copy-paste the output back. The real value is the AI runs the tools itself against your microservices. Add the semantic tool retrieval layer we discussed, connect it to your 1000 endpoints, and the AI can investigate autonomously. This turns it from a chat assistant into an actual autonomous agent.

2. Real-time Collaboration
Two engineers can't work the same session simultaneously. Add WebSocket-based live sync so multiple people see each other's messages and the AI responses in real time — essential for high-severity P1 incidents where multiple people are involved.

3. Alerting Integration (PagerDuty / OpsGenie)
Instead of engineers manually creating sessions when something breaks, the app auto-creates a session the moment an alert fires — pre-filled with the alert name, affected service, environment, and the matching skill already selected. Zero friction from alert to diagnosis.

Tier 2 — High value, moderate effort
4. AI-Generated RCA / Post-Incident Report
At the end of a resolved session, a button generates a structured Root Cause Analysis document from the chat history — what happened, timeline, root cause, resolution steps, prevention recommendations. Exportable as PDF or Markdown. Saves 30–60 minutes of post-incident write-up.

5. Similar Incidents Suggestion
When you start a new session, use vector similarity to search past resolved sessions and surface "3 similar incidents from the last 90 days" with links. Engineers don't reinvent the wheel for recurring issues.

6. Session Timeline View
Alongside the chat, show a structured visual timeline: 09:12 — Session opened → 09:15 — AI identified DB blocking query → 09:23 — Resolution applied → 09:31 — Resolved. Much easier for post-incident review than reading a raw chat log.

7. Runbook Auto-Improvement
Track which steps in a runbook actually led to resolution (via a "this fixed it" button mid-chat). Over time, surface which steps are most effective and flag runbook sections that are never useful.

Tier 3 — Polish and completeness
8. Authentication & Multi-user
Currently anyone with the URL can access everything. Add login so sessions belong to engineers, you get "sessions I worked on" filtering, and an audit trail of who did what.

9. Metrics Dashboard
Mean Time To Resolve (MTTR) by category and environment, most frequent failure types, busiest incident times, which skills are used most. Useful for ops managers and SRE leads.

10. Command Block UX
When the AI suggests a command, render it in a terminal-style block with a one-click copy button and a structured "paste output here" reply box — rather than the engineer copy-pasting into a plain text message.

11. Session Linking
Link related sessions — e.g., an Autosys failure that was caused by a database deadlock. Shows a "Related Incidents" section in the left panel connecting the two.

What I'd build first
If I were prioritizing: live tool execution first (it's the core differentiator), then alerting integration (removes the manual session creation step), then RCA generation (saves real post-incident time). Those three together make this a serious production ops platform rather than a smart chat interface.

Want me to start on any of these?
