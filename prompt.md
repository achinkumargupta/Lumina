



# Lumina Knowledge Base — Full Build Prompt for Claude

Build a full-stack web application called **Lumina Knowledge Base** — an intelligent reading companion that lets users import documents, select paragraphs to open an AI-powered Q&A context panel, start discussion threads, search documents, and view analytics. Use OpenAI (GPT-4o or equivalent) for all AI features.

---

## Tech Stack

- **Frontend**: React + TypeScript, Vite, Tailwind CSS, Shadcn UI, Wouter (routing), TanStack Query v5, Framer Motion
- **Backend**: Node.js, Express 5, TypeScript (tsx for dev, esbuild for prod)
- **Database**: PostgreSQL with Drizzle ORM + drizzle-zod for schema validation
- **AI**: OpenAI Node SDK (`openai` package), model `gpt-4o`
- **Icons**: `lucide-react`

---

## Database Schema

Three tables using Drizzle ORM:

### `documents`
```ts
export const documentStatusEnum = pgEnum("document_status", ["draft", "review", "approved", "deprecated"]);

export const documentsTable = pgTable("documents", {
  id: serial("id").primaryKey(),
  title: text("title").notNull(),
  content: text("content").notNull(),
  status: documentStatusEnum("status").default("draft").notNull(),
  tags: text("tags").array().default([]).notNull(),
  createdAt: timestamp("created_at").defaultNow().notNull(),
  updatedAt: timestamp("updated_at").defaultNow().notNull(),
});
```

### `threads`
```ts
export const threadStatusEnum = pgEnum("thread_status", ["open", "merged"]);
export const threadCategoryEnum = pgEnum("thread_category", [
  "clarification", "bug", "test-case", "implementation", "runbook", "incident", "general"
]);

export const threadsTable = pgTable("threads", {
  id: serial("id").primaryKey(),
  documentId: integer("document_id").notNull().references(() => documentsTable.id, { onDelete: "cascade" }),
  paragraphIndex: integer("paragraph_index").notNull(),
  paragraphText: text("paragraph_text").notNull(),
  title: text("title").notNull(),
  category: threadCategoryEnum("category").default("general").notNull(),
  status: threadStatusEnum("status").default("open").notNull(),
  mergedNote: text("merged_note"),
  createdAt: timestamp("created_at").defaultNow().notNull(),
  updatedAt: timestamp("updated_at").defaultNow().notNull(),
});
```

### `thread_messages`
```ts
export const messageRoleEnum = pgEnum("message_role", ["user", "assistant"]);

export const threadMessagesTable = pgTable("thread_messages", {
  id: serial("id").primaryKey(),
  threadId: integer("thread_id").notNull().references(() => threadsTable.id, { onDelete: "cascade" }),
  role: messageRoleEnum("role").notNull(),
  content: text("content").notNull(),
  createdAt: timestamp("created_at").defaultNow().notNull(),
});
```

---

## Backend API Routes (all prefixed `/api`)

### Documents

**GET /api/documents**
- Query params: `status` (filter by status), `tag` (filter by tag)
- Returns all documents ordered by `updatedAt` DESC
- Each document includes: `threadCount` (total threads), `openThreadCount` (open threads only)

**POST /api/documents**
- Body: `{ title, content, status?, tags? }`
- Returns created document with `threadCount: 0, openThreadCount: 0`

**GET /api/documents/:id**
- Returns single document with thread counts

**PUT /api/documents/:id**
- Body: any subset of `{ title, content, status, tags }`
- Updates `updatedAt` automatically
- Returns updated document with thread counts

**DELETE /api/documents/:id**
- Returns 204

**GET /api/documents/:id/threads**
- Query param: `category` (optional filter)
- Returns threads ordered by `paragraphIndex`
- Each thread includes: `messageCount`

**POST /api/documents/:id/threads**
- Body: `{ paragraphIndex, paragraphText, title, category? }`
- Returns created thread with `messageCount: 0`

### Threads

**GET /api/threads/:id**
- Returns thread with full `messages` array and `messageCount`

**DELETE /api/threads/:id**
- Returns 204

**POST /api/threads/:id/messages**
- Body: `{ content }` (user message)
- Saves user message to DB
- Calls OpenAI with system prompt focused on `thread.paragraphText`
- System prompt: *"You are a helpful assistant answering questions about a specific paragraph from a knowledge base document. The paragraph the user is asking about: [paragraphText]. Stay focused on this paragraph and its topic. Be concise but thorough."*
- Includes full prior message history for context
- Saves and returns the assistant's response message

**POST /api/threads/:id/merge**
- Fetches all messages for the thread
- Calls OpenAI to summarize the Q&A into a 2–4 sentence annotation
- System prompt: *"You are a summarizer. Given a Q&A thread about a paragraph, produce a concise note (2-4 sentences) capturing the key insights and clarifications that emerged. Write in a neutral, informative tone as if it's an annotation to the original text."*
- Sets thread `status = 'merged'`, saves `mergedNote`
- Returns updated thread

### AI

**POST /api/ai/suggest-questions**
- Body: `{ paragraphText, documentTitle }`
- System prompt: *"You are a curious reader. Given a paragraph from a knowledge base document, generate 4-5 insightful clarifying questions that a reader might have. Return ONLY a JSON array of question strings."*
- Returns `{ questions: string[] }`

**POST /api/ai/generate-test-cases**
- Body: `{ paragraphText, documentTitle }`
- System prompt: *"You are a senior QA engineer. Given a paragraph from a requirements or specification document, generate thorough test cases covering happy path, edge cases, and negative scenarios. Return ONLY a JSON array: [{"title": "...", "scenario": "...", "expectedResult": "...", "priority": "high|medium|low"}]. Generate 4-6 test cases."*
- Returns `{ testCases: Array<{ title, scenario, expectedResult, priority }> }`

**POST /api/ai/generate-implementation-notes**
- Body: `{ paragraphText, documentTitle }`
- System prompt: *"You are a senior software architect. Given a paragraph from a requirements document, generate implementation notes. Cover architecture considerations, potential pitfalls, suggested approaches, and complexity. Return ONLY a JSON array: [{"area": "...", "note": "...", "complexity": "low|medium|high"}]. Generate 4-6 notes covering: API design, data model, performance, security, edge cases."*
- Returns `{ notes: Array<{ area, note, complexity }> }`

**POST /api/ai/summarize-document**
- Body: `{ content, title }`
- System prompt: *"You are an expert business analyst. Generate: 1. A concise executive summary (3-4 sentences). 2. 5-7 key points. 3. Stakeholder-specific notes for: business (business value, ROI, risks), product (scope, dependencies, open questions), engineering (technical complexity, approach, risks), qa (testability, edge cases, acceptance criteria). Return ONLY JSON: { "summary": "...", "keyPoints": ["..."], "stakeholderNotes": { "business": "...", "product": "...", "engineering": "...", "qa": "..." } }"*
- Returns that JSON structure

### Analytics

**GET /api/analytics**
Returns:
```json
{
  "totalDocuments": 0,
  "totalThreads": 0,
  "openThreads": 0,
  "mergedThreads": 0,
  "threadsByCategory": [{ "category": "clarification", "count": 5 }],
  "documentsByStatus": [{ "status": "draft", "count": 3 }],
  "mostActiveDocuments": [{ "id": 1, "title": "...", "threadCount": 10, "openThreadCount": 3 }],
  "recentActivity": [{ "type": "thread", "description": "...", "documentTitle": "...", "createdAt": "..." }]
}
```
- Most active documents: top 5 by thread count
- Recent activity: last 10 threads created (join with document title)

### Search

**GET /api/search?q=term&type=all|documents|threads**
- Minimum query length: 2 characters
- Document search: case-insensitive match on title OR content. Include an `excerpt` (160 chars surrounding the match)
- Thread search: match on title OR paragraphText. Include `documentId`, `documentTitle`
- Returns `{ results: [], total: 0 }`
- Each result: `{ type: "document"|"thread", id, title, excerpt, documentId, documentTitle }`

---

## Frontend Pages & Components

### Routing (Wouter)
```
/                          → Home page
/analytics                 → Analytics dashboard
/search                    → Search results
/documents/new             → Create document
/documents/:id             → Document reading view
/documents/:id/threads/:threadId → Thread chat view
```

---

### Shared Layout Component
Every page is wrapped in `<Layout>`. It contains:
- A **fixed left sidebar** (width: 288px / `w-72`) with:
  - **Logo area**: Sparkles icon + "Lumina" in serif bold font, links to `/`
  - **Search form**: Text input with Search icon. On submit, navigates to `/search?q=...`
  - **Library section**: List of all documents from `GET /api/documents`. Each item:
    - FileText icon
    - Document title (truncated)
    - A colored dot indicating status: slate=draft, amber=review, green=approved, red=deprecated
    - Active state when current route matches `/documents/:id` or its threads
  - **Footer**: Analytics link + "New Document" primary button (links to `/documents/new`)
- A **textured background overlay** (subtle noise/grain texture via CSS)
- A **main content area** that renders `children`

---

### Home Page (`/`)
An editorial welcome screen shown when no document is selected. Center the content with a card:
- Book Open icon (orange/amber primary color)
- Heading: "Welcome to Lumina" (serif font, large)
- Subtext: "Your intelligent reading companion. Import texts, explore dense paragraphs, and ask clarifying questions without losing your place."
- "Start Writing" button (dark, links to `/documents/new`)

The background should have a warm, sandy/parchment gradient texture.

---

### Create Document Page (`/documents/new`)
A centered, distraction-free editor:
- Page title: "New Document"
- Form fields:
  - **Title** (text input, large, prominent, serif font)
  - **Status** (select: Draft, Review, Approved, Deprecated)
  - **Tags** (text input, comma-separated, hint: "tag1, tag2, tag3")
  - **Content** (textarea, tall, for the document body — plain text where double newlines separate paragraphs)
- Submit button: "Create Document"
- On success, navigate to `/documents/:newId`

---

### Document View Page (`/documents/:id`)

This is the core reading interface. It has two regions side by side:

#### Left: Main Reading Area (scrollable)
- Max width 3xl, centered, generous padding
- **Document header**:
  - Status badge (color-coded pill): draft=slate, review=amber, approved=green, deprecated=red
  - Tags (muted pill badges)
  - Settings icon button (opens Edit modal)
  - Document title in large serif font (4xl–5xl)
- **Content**: Split `document.content` by `\n\n` into paragraphs. Each rendered via `<Paragraph>` component.

#### Right: Context Panel (slides in from right, 450px wide)
Appears when a paragraph is selected. Uses Framer Motion for slide-in animation (`x: 50 → 0`). Disappears when closed.

#### Paragraph Component
Each paragraph:
- Clickable. On click: sets `selectedParagraphIndex`
- Visual states:
  - **Default**: subtle hover background
  - **Selected**: accent background + left primary-color bar (Framer Motion animated)
- **Left gutter indicators** (shown on hover or always on small screens):
  - If paragraph has open threads: circular badge with thread count
  - Otherwise: faint message icon
- **Below the paragraph** (if merged threads exist):
  - Indented block with left border (primary color, 20% opacity)
  - For each merged thread: checkmark icon + "Synthesized from: [thread title]" + AI-generated `mergedNote` in serif italic

#### Edit Document Modal
Opens when Settings icon is clicked. Framer Motion scale animation (`scale: 0.95 → 1`):
- Title input
- Status select
- Tags input (comma-separated)
- Save / Cancel buttons
- Calls `PUT /api/documents/:id` (title, status, tags only — content is not editable from this modal)

---

### Context Panel Component
A 450px sliding right panel. Three tabs:

#### Tab 1: "Insights" (Sparkles icon)
- Automatically calls `POST /api/ai/suggest-questions` when a paragraph is selected
- Shows 4–5 AI-generated questions as cards
- Each card: question text + "Start thread →" affordance
- Clicking a card: creates a thread with that question as title and `category: "clarification"`, then navigates to the thread view
- Loading state: pulsing Sparkles icon + "Analyzing text..."

#### Tab 2: "Threads" (MessageSquare icon)
- Shows count badge if open threads exist
- "Start New Thread" dashed button → expands inline form:
  - Title input ("What's your question?")
  - Category dropdown: Clarification, Bug, Test Case, Implementation, Runbook, Incident, General
  - Create / Cancel buttons
- Lists all open threads for this paragraph:
  - Category badge (color-coded: blue=clarification, red=bug, purple=test-case, indigo=implementation, teal=runbook, orange=incident, gray=general)
  - Thread title
  - Message count + relative time
  - Click → navigates to thread view

#### Tab 3: "AI Tools" (Wand2 icon)
Three independent action buttons, each showing results inline:

**Generate Test Cases**
- Button with FlaskConical icon (purple)
- Results: list of test case cards showing: title, priority badge (high=red, medium=amber, low=green), scenario, expected result
- Each card has "Create Thread" link that creates a thread with `category: "test-case"`

**Implementation Notes**
- Button with Code2 icon (indigo)
- Results: list of note cards showing: area name, complexity badge, note text
- Each card has "Create Thread" link with `category: "implementation"`

**Summarize Full Document**
- Button with FileText icon
- Only enabled when `documentContent` is available
- Results: summary text, bulleted key points, and 4 stakeholder sections:
  - BUSINESS (blue left border)
  - PRODUCT (purple left border)
  - ENGINEERING (indigo left border)
  - QA (green left border)

**Panel logic**:
- Auto-switch to "Threads" tab if paragraph has open threads on selection, otherwise default to "Insights"
- Reset AI tool states when paragraph changes

---

### Thread View Page (`/documents/:id/threads/:threadId`)

A focused chat interface:
- **Header**: Back link to document, thread title, category badge, "Merge Thread" button
- **Source paragraph quote**: The `thread.paragraphText` shown in a styled blockquote with BookOpen icon
- **Message list** (scrollable):
  - User messages: right-aligned, primary background
  - Assistant messages: left-aligned, muted/card background, with Sparkles icon
  - Timestamps for each message
  - Auto-scroll to bottom on new messages
- **Input area** (fixed at bottom):
  - Textarea for message input
  - Send button (disabled when empty or loading)
  - Sends to `POST /api/threads/:id/messages`
- **Merge Thread**: Button that calls `POST /api/threads/:id/merge`. On success: shows the `mergedNote`, marks thread as merged (disables input), navigates back to document

---

### Analytics Page (`/analytics`)

Dashboard with stat cards + data sections:

**Top stat cards** (4 cards in a row):
- Total Documents
- Total Threads
- Open Threads
- Merged Threads

**Charts / Lists** (below stats):
- **Documents by Status**: Simple visual breakdown (bar or pill indicators) — draft/review/approved/deprecated counts
- **Threads by Category**: List with category name, count, and a proportional bar
- **Most Active Documents**: List of top 5 documents with thread count and open thread count
- **Recent Activity**: Timeline of last 10 thread creation events — thread title + document name + relative timestamp

---

### Search Page (`/search`)

- Reads `?q=` from URL on mount, auto-executes search
- Has a search input (re-searchable)
- Toggle buttons to filter: All / Documents / Threads
- Results displayed as cards:
  - Document results: FileText icon, title, excerpt snippet, link to `/documents/:id`
  - Thread results: MessageSquare icon, thread title, category badge, parent document name, link to `/documents/:id/threads/:threadId`
- Empty state when no results
- "Searching..." loading state

---

## Visual Design & Styling

### Color Palette
Use CSS custom properties in `:root` for the theme. The design aesthetic is "editorial" — warm, parchment-like tones:

- Primary color: Warm amber/orange (e.g., `hsl(38, 92%, 50%)`)
- Background: Warm off-white (`hsl(40, 30%, 97%)`)
- Card: Slightly warm white
- Sidebar: Slightly cooler off-white, subtle border
- Foreground text: Near-black with warm tint
- Muted text: Medium warm gray

### Typography
- **Headings & document content**: Serif font (e.g., Georgia, or load via Google Fonts: "Lora" or "Playfair Display")
- **UI elements**: System sans-serif (Inter or similar)
- Document paragraphs should have generous `line-height` (1.8+) and `text-foreground/80`

### Texture / Background
Add a subtle noise/grain texture overlay to the background:
```css
.texture-overlay {
  position: fixed;
  inset: 0;
  background-image: url("data:image/svg+xml,..."); /* SVG noise pattern */
  opacity: 0.03;
  pointer-events: none;
}
```

### Animations (Framer Motion)
- Context panel: slides in from right (`x: 50 → 0`, fade in)
- Context panel close: slides out right quickly (`duration: 0.2`)
- Edit modal: scale in (`scale: 0.95 → 1`) with backdrop blur
- Suggested question cards: staggered fade-up (`delay: i * 0.1`)
- Paragraph selected bar: `scaleY: 0 → 1`
- Merged notes: fade + slide up

### Status Color Mapping
- `draft`: slate (`bg-slate-400/20 text-slate-600`)
- `review`: amber (`bg-amber-400/20 text-amber-600`)
- `approved`: green (`bg-green-500/20 text-green-600`)
- `deprecated`: red (`bg-red-400/20 text-red-600`)

### Category Color Mapping
- `clarification`: blue
- `bug`: red
- `test-case`: purple
- `implementation`: indigo
- `runbook`: teal
- `incident`: orange
- `general`: gray

---

## Key Implementation Notes

1. **Paragraph splitting**: Split `document.content` by `'\n\n'` (double newline) and filter empty strings. Each chunk is one paragraph. The `paragraphIndex` in threads corresponds to the array index.

2. **Thread filtering in document view**: When rendering paragraphs, filter threads with `threads.filter(t => t.paragraphIndex === index)`. The context panel shows only threads for the currently selected paragraph.

3. **Context panel auto-question fetch**: Fire `suggest-questions` mutation immediately when `paragraphIndex` changes (in a `useEffect`). Also reset test-case and implementation note states on paragraph change.

4. **Merge flow**: After merging, the `mergedNote` appears as an annotation below the paragraph in the document view (because the thread status becomes `merged` and `mergedNote` is populated). Invalidate the threads query after merge.

5. **Search navigation**: The search input in the sidebar submits and navigates to `/search?q=...`. The search page reads from the URL, auto-fetches on mount.

6. **Relative timestamps**: Format dates as relative time ("2 hours ago", "just now", etc.).

7. **Tag input/output**: Tags are stored as a PostgreSQL `text[]` array. In UI forms, accept comma-separated strings and split/join on display.

8. **Auto-scroll in thread view**: Use a `useEffect` that scrolls the message container to bottom whenever messages update.

9. **TanStack Query v5**: Use only the object form: `useQuery({ queryKey: [...], queryFn: ... })`. Invalidate related queries after mutations.

10. **OpenAI response parsing**: All AI endpoints return JSON embedded in the model response. Always wrap `JSON.parse()` in try/catch with a sensible fallback (empty array or empty object).

---

## Data Fetch Patterns

Use TanStack Query v5 on the frontend. Create custom hooks:
- `useListDocuments()` — GET /api/documents
- `useGetDocument(id)` — GET /api/documents/:id
- `useCreateDocument()` — POST /api/documents
- `useUpdateDocument()` — PUT /api/documents/:id
- `useDeleteDocument()` — DELETE /api/documents/:id
- `useListThreads(documentId)` — GET /api/documents/:id/threads
- `useGetThread(threadId)` — GET /api/threads/:id
- `useCreateThread(documentId)` — POST /api/documents/:id/threads
- `useDeleteThread()` — DELETE /api/threads/:id
- `useSendMessage(threadId)` — POST /api/threads/:id/messages
- `useMergeThread()` — POST /api/threads/:id/merge
- `useSuggestQuestions()` — POST /api/ai/suggest-questions
- `useGenerateTestCases()` — POST /api/ai/generate-test-cases
- `useGenerateImplementationNotes()` — POST /api/ai/generate-implementation-notes
- `useSummarizeDocument()` — POST /api/ai/summarize-document
- `useAnalytics()` — GET /api/analytics
- `useSearch(query, type)` — GET /api/search

---

## Environment Variables

- `DATABASE_URL` — PostgreSQL connection string
- `OPENAI_API_KEY` — OpenAI API key

---

## Summary of All Features

1. **Document library** in sidebar with status indicators (dot color) and live document list
2. **Create documents** with title, content, status, and comma-separated tags
3. **Edit document metadata** (title, status, tags) via modal from document view
4. **Delete documents** (cascades to threads and messages)
5. **Paragraph-level selection** with animated highlight bar and left gutter thread indicators
6. **AI question suggestions** auto-generated on paragraph selection (Insights tab)
7. **Thread creation** from suggested questions or manually (Threads tab)
8. **Thread categories**: clarification, bug, test-case, implementation, runbook, incident, general
9. **AI chat per thread** — context-aware, paragraph-focused, with full message history
10. **Thread merge** — AI summarizes conversation into a 2-4 sentence annotation that appears permanently below the paragraph
11. **Merged notes display** inline below paragraphs, with serif italic styling
12. **AI test case generation** (QA Tools tab) — 4-6 test cases with priority
13. **AI implementation notes** (QA Tools tab) — 4-6 architectural notes with complexity
14. **Full document summarization** — executive summary + key points + 4 stakeholder sections
15. **"Create Thread" shortcut** from any AI-generated test case or implementation note
16. **Global search** across document titles/content and thread titles/paragraphs
17. **Search with excerpts** showing context around the matched term
18. **Search filtering** by type (All / Documents / Threads)
19. **Analytics dashboard** — totals, threads by category, documents by status, most active, recent activity
20. **Status filtering** on document list (filter by status or tag via query params)
21. **Relative timestamps** throughout the app
22. **Responsive sidebar** with active document highlighting
23. **Animated context panel** slides in/out without leaving reading view
24. **Tag support** on documents — displayed as pill badges, filterable
25. **Error and loading states** throughout (spinners, error messages, empty states)




