# Agent Platform — Phase 2 Build Instructions
## For use with Claude Code in VS Code

> **How to use:** Open your `agent-platform` project in VS Code. Open Claude Code terminal. Copy-paste each step below as a prompt. **Test after each step before moving to the next.**

---

## PRE-FLIGHT CHECK

Paste this first to make sure everything is running:

```
Make sure Docker is running with `docker-compose up -d`, then run `cd backend && npm run db:seed` to ensure the database is seeded. Start the backend with `npm run dev`. In a second terminal start the frontend with `cd frontend && npm run dev`. Verify http://localhost:3000/api/health returns OK and http://localhost:5173 loads the app.
```

---

## STEP 1 — Authentication (Backend)

```
Add JWT authentication to the backend. Here's what to build:

1. Install jsonwebtoken and bcryptjs: `npm install jsonwebtoken bcryptjs`

2. Create `backend/src/middleware/auth.js`:
   - Export a `verifyToken` middleware that reads the Authorization header (Bearer token)
   - Decodes the JWT using process.env.JWT_SECRET (default: 'dev-secret-change-me')
   - Attaches the decoded payload (user_id, org_id, workspace_id, persona_id, role) to req.user
   - Returns 401 if no token, 403 if invalid

3. Create `backend/src/routes/auth.js`:
   - POST /api/auth/login — accepts { email, password }
   - Looks up user in the users table by email
   - For the prototype, skip password hashing — just check if user exists (we'll add bcrypt later)
   - Returns a JWT containing: user_id, org_id, workspace_id, persona_id, role
   - Also returns the user object and their persona object in the response
   - POST /api/auth/me — takes the JWT token, returns current user + persona

4. Mount the auth routes in `backend/src/index.js` at /api/auth (BEFORE the other routes)
   - Do NOT add auth middleware to existing routes yet — keep them open for now

5. Update the seed script to create a demo user:
   - Email: demo@acme.com, name: "Demo User"
   - Assign to the existing demo org, default workspace, "builder" role, "Business Analyst" persona

6. Add JWT_SECRET=dev-secret-change-me to .env.example

Test: `curl -X POST http://localhost:3000/api/auth/login -H "Content-Type: application/json" -d '{"email":"demo@acme.com"}' ` should return a token + user object.
```

---

## STEP 2 — Authentication (Frontend)

```
Add authentication to the Vue frontend. Here's what to build:

1. Create `frontend/src/stores/auth.js` (Pinia store):
   - State: user (null), persona (null), token (null), loading, error
   - Actions: login(email, password), logout(), loadFromStorage()
   - login calls POST /api/auth/login, stores token in localStorage, sets user + persona
   - logout clears everything from state and localStorage
   - loadFromStorage runs on app init — reads token from localStorage, calls /api/auth/me to hydrate user + persona

2. Update `frontend/src/api/client.js`:
   - Add an Axios request interceptor that reads the token from localStorage
   - If token exists, add header: Authorization: Bearer <token>

3. Create `frontend/src/views/LoginView.vue`:
   - Simple centered login card with email + password fields and a "Sign In" button
   - On submit, call auth store login()
   - On success, redirect to "/"
   - Show error message if login fails
   - Use the existing Tailwind classes from main.css (card, btn-primary)

4. Add a route for /login in router/index.js (no auth required)

5. Add a navigation guard in router/index.js:
   - Before each route (except /login), check if auth store has a token
   - If no token, redirect to /login
   - On app startup in main.js, call auth.loadFromStorage() before mounting the app

6. Update App.vue sidebar:
   - Show the logged-in user's name and persona icon at the bottom of the sidebar
   - Add a "Sign Out" button that calls auth.logout() and redirects to /login

Test: Go to http://localhost:5173 — should redirect to /login. Enter demo@acme.com, click Sign In. Should redirect to dashboard with user info in the sidebar.
```

---

## STEP 3 — Persona-Driven Home Screen

```
Replace the current DashboardView with a persona-driven home screen. The existing backend route GET /api/personas/:id/full already returns the persona with their recommended skills, templates, connectors, and onboarding steps.

1. Update `frontend/src/views/DashboardView.vue`:

   SECTION A — Welcome Header:
   - "Welcome back, {user.name}" with the persona icon and name
   - Show their builder tier as a colored badge (no-code=green, low-code=orange, pro-code=red)
   
   SECTION B — Onboarding Checklist (if persona has onboarding steps):
   - Fetch persona full data from GET /api/personas/{persona_id}/full
   - Show onboarding steps as a horizontal progress bar or checklist cards
   - Each step shows: step title, description, a "Mark Complete" toggle
   - Store completion state in localStorage keyed by user_id (no backend needed yet)
   
   SECTION C — Recommended Templates (from persona.recommended_templates):
   - Card grid showing the persona's recommended templates
   - Each card: template name, description, required skill count, "Use Template" button
   - "Use Template" calls POST /api/templates/{id}/fork (already exists in backend)
   - On success, redirect to /agents/{new_agent_id}
   
   SECTION D — Recommended Skills (from persona.recommended_skills):
   - Horizontal scrollable row of skill cards
   - Each card: skill name, category badge, usage count
   - Click navigates to /skills/{skill_id}
   
   SECTION E — Recommended Connectors (from persona.recommended_connectors):
   - Row of connector cards showing name, category, auth type
   - "Connect" button (visual only for now, no real OAuth)
   
   SECTION F — Quick Stats (keep the existing dashboard API):
   - Move the existing KPI cards (total agents, skills, runs) to a compact row at the bottom

Keep the Tailwind styling consistent with existing views. Use the card, badge, btn-primary classes from main.css.

Test: Login as demo@acme.com. The home screen should show personalized content for the "Business Analyst" persona — their recommended templates (like Document Q&A Agent, Weekly Report Generator), skills (Data Transformer, Report Generator), and connectors (PostgreSQL, Google Sheets).
```

---

## STEP 4 — Template Fork Flow

```
Build the full template fork flow — user picks a template, forks it into a new agent, and lands on the agent detail page.

1. Update `frontend/src/views/TemplatesView.vue`:
   - Replace the basic table with a card grid (similar to SkillsView)
   - Each card shows: name, description, required skills (as badges), required connectors (as badges)
   - "Use This Template" button on each card
   - Clicking it calls POST /api/templates/{id}/fork with the user's workspace_id from auth store
   - On success (returns new agent), redirect to /agents/{agent_id}

2. Create `frontend/src/views/AgentDetailView.vue`:
   - Route: /agents/:id
   - Fetch agent data from GET /api/agents/{id} (already exists)
   - Fetch linked skills from GET /api/agents/{id}/skills (already exists)
   - Fetch linked connectors from GET /api/agents/{id}/connectors (already exists)
   - Fetch versions from GET /api/agents/{id}/versions (already exists)
   
   Layout with tabs:
   TAB 1 — Overview:
   - Agent name (editable inline), description, status badge, tier badge
   - Linked skills as a list with execution order
   - Linked connectors as a list
   - "Add Skill" button → opens a modal listing all skills, click to link
   - "Add Connector" button → opens a modal listing all connectors, click to link
   
   TAB 2 — Versions:
   - Version history table (version number, status, created date)
   - "Promote to Published" button on draft versions
   
   TAB 3 — Runs:
   - Filtered run history for this agent from GET /api/runs?agent_id={id}
   - Show status, duration, tokens, timestamp
   
3. Add the /agents/:id route to router/index.js

4. Update AgentsView.vue:
   - Make each agent row clickable → navigates to /agents/{agent_id}
   - Add a "New Agent" button that opens a create modal (similar to SkillCreateModal)
   - Fields: name, description, builder_tier dropdown

Test: Go to Templates. Click "Use This Template" on "Lead Qualification Agent". Should fork and redirect to a new agent detail page showing the linked skills and connectors from the template.
```

---

## STEP 5 — Connector Detail & Instance Management

```
Build connector detail pages so users can "connect" integrations to their workspace.

1. Update `backend/src/routes/connectors.js`:
   - GET /api/connectors/:id/detail — returns connector + all instances for the user's workspace
   - POST /api/connectors/:id/instances — creates a connector_instance record:
     { workspace_id, connector_id, config: {}, credentials: {}, status: 'active' }
   - PUT /api/connectors/:id/instances/:instanceId — update config/status
   - DELETE /api/connectors/:id/instances/:instanceId — remove instance
   - GET /api/connectors/:id/instances — list instances for this connector in a workspace
   (Use the existing connector_instances table from the schema)

2. Update `frontend/src/views/ConnectorsView.vue`:
   - Card grid layout (like Skills)
   - Each card: connector name, category badge, auth_type, "Connected" or "Connect" button
   - Show a green checkmark if there's an active instance for this connector in the user's workspace
   
3. Create `frontend/src/views/ConnectorDetailView.vue`:
   - Route: /connectors/:id
   - Shows connector info: name, category, auth type, description
   - "Connections" section: list of active instances
   - "New Connection" button → opens a simple form modal:
     - Connection name (text input)
     - Config (JSON textarea for now — we'll make this nicer later)
     - "Connect" button → POST to create instance
   - Each instance shows: name, status (active/error/inactive), created date, "Disconnect" button
   
4. Add the /connectors/:id route to router/index.js

Test: Navigate to Connectors. Click on "Salesforce". Click "New Connection", enter a name and dummy config JSON, click Connect. Should see the new connection appear with a green "active" status.
```

---

## STEP 6 — Agent Run Simulator

```
Build a "Run Agent" feature so users can trigger and watch an agent execution with mock data.

1. Add to `backend/src/routes/agents.js`:
   - POST /api/agents/:id/run — simulates an agent execution:
     a. Creates an agent_run record (status: 'running')
     b. For each linked skill (from agent_skills), creates a run_step record:
        - Simulates execution: random tokens (100-500), random latency (200-2000ms), status: 'success'
        - If the agent has connectors, add a run_step for each connector call too
     c. Updates the agent_run status to 'completed', sets total tokens and total duration
     d. Returns the full run with all steps
   - GET /api/runs/:runId/steps — returns all run_steps for a given run

2. Update the Agent Detail View (AgentDetailView.vue):
   - Add a prominent "▶ Run Agent" button in the header
   - Clicking it calls POST /api/agents/{id}/run
   - While running, show a loading spinner
   - On completion, show a "Run Complete" toast/banner with summary (tokens, duration, status)
   - Auto-switch to the "Runs" tab and highlight the new run

3. Create a "Run Detail" expandable row or modal in the Runs tab:
   - Click a run row → expands to show all run_steps
   - Each step: step number, skill/connector name, status (with color), tokens, duration, timestamp
   - Visual: a vertical timeline of steps with colored dots

4. Also update `frontend/src/views/RunsView.vue`:
   - Make it a richer view with:
     - Filter by status (all, completed, failed, running)
     - Filter by agent (dropdown)
     - Click a run row → expand to show steps inline
   
Test: Go to an agent detail page. Click "Run Agent". Should see a simulated run complete with step-by-step breakdown showing each skill and connector that was called.
```

---

## STEP 7 — Dashboard Wiring with Real Data

```
Wire the Dashboard's bottom stats section to real backend data using the existing /api/dashboard endpoints.

1. The backend already has these routes in `backend/src/routes/dashboard.js`:
   - GET /api/dashboard/summary — total agents, skills, connectors, templates, runs
   - GET /api/dashboard/persona-adoption — agents per persona
   - GET /api/dashboard/skill-reuse — most-used skills

2. Update DashboardView.vue (the "Quick Stats" section at the bottom, or a separate /dashboard/admin route):
   - Wire the KPI cards to GET /api/dashboard/summary (using the org_id from the auth store)
   - Add a "Skill Reuse Leaderboard" section using GET /api/dashboard/skill-reuse
     - Show skill name, usage_count as a horizontal bar chart or ranked list
   - Add a "Persona Adoption" section using GET /api/dashboard/persona-adoption
     - Show persona name, agent count as a simple bar or donut chart
   - Use a simple chart library: install `chart.js` and `vue-chartjs` in the frontend:
     `npm install chart.js vue-chartjs`
   - Create a BarChart.vue component wrapper for reuse

Test: Dashboard should show real counts from the database. After forking templates and running agents, the numbers should update.
```

---

## STEP 8 — Polish & Navigation Cleanup

```
Final polish pass to make everything feel connected.

1. Update App.vue sidebar navigation:
   - Highlight the active route
   - Add icon emojis to each nav item:
     📊 Dashboard, 🤖 Agents, ⚡ Skills, 🔌 Connectors, 📋 Templates, 👤 Personas, 📜 Runs
   - Show the user's persona name and tier badge at the bottom above "Sign Out"

2. Add breadcrumbs to detail pages:
   - SkillDetailView: Skills > {Skill Name}
   - AgentDetailView: Agents > {Agent Name}
   - ConnectorDetailView: Connectors > {Connector Name}
   - Each segment is clickable and navigates back

3. Add a global toast/notification system:
   - Create a composable `frontend/src/composables/useToast.js`
   - Provides: showToast(message, type) where type is 'success', 'error', 'info'
   - Renders a fixed-position toast at top-right that auto-dismisses after 3 seconds
   - Wire it into: create skill, fork template, run agent, connect connector, login/logout

4. Add loading skeletons:
   - Create a `SkeletonLoader.vue` component (gray pulsing rectangles)
   - Use it in place of "Loading..." text across all views

5. Add empty states to all list views:
   - If no items, show an illustration/emoji + helpful message + "Create" CTA button
   - Skills, Agents, Connectors, Templates, Runs should all have empty states

Test: Navigate through the full flow: Login → Dashboard (persona content) → Fork template → Agent detail → Add skill → Run agent → View run steps → Check dashboard stats. Everything should feel connected.
```

---

## TESTING CHECKLIST

After completing all steps, verify this end-to-end flow:

1. ✅ `http://localhost:5173` → redirects to `/login`
2. ✅ Login with `demo@acme.com` → redirects to Dashboard
3. ✅ Dashboard shows personalized content for Business Analyst persona
4. ✅ Onboarding checklist visible with steps
5. ✅ Recommended templates show (Document Q&A, Weekly Report Generator)
6. ✅ Click "Use Template" → forks → redirects to new Agent detail page
7. ✅ Agent detail shows linked skills and connectors from the template
8. ✅ Can add more skills/connectors to the agent
9. ✅ Click "Run Agent" → simulated run completes with step breakdown
10. ✅ Go to Connectors → click Salesforce → create a connection instance
11. ✅ Go to Skills → click a skill → see Prompt Studio → create version → test it
12. ✅ Dashboard stats reflect real data from the database
13. ✅ Sidebar shows user name, persona, sign out works
14. ✅ All detail pages have breadcrumbs
15. ✅ Toast notifications on key actions

---

## FILE REFERENCE — What Already Exists

**Backend (Express + PostgreSQL):**
```
backend/
  src/
    index.js              ← Main server, mounts all routes at /api/*
    config/db.js          ← PostgreSQL pool, query() helper, schema=prototype_builder
    middleware/errorHandler.js
    models/base.js        ← Generic CRUD factory (findAll, findById, create, update, remove)
    models/index.js       ← All entity models using factory
    routes/_factory.js    ← Generic CRUD route factory (GET, POST, PUT, DELETE)
    routes/skills.js      ← FULL: CRUD + versions + test + duplicate + compose + stats
    routes/agents.js      ← CRUD + versions + linked skills/connectors
    routes/templates.js   ← CRUD + fork endpoint (POST /:id/fork)
    routes/personas.js    ← CRUD + GET /:id/full (hydrates onboarding, skills, templates, connectors)
    routes/dashboard.js   ← summary, persona-adoption, skill-reuse
    routes/connectors.js  ← Basic CRUD only (needs detail + instances)
    routes/runs.js        ← CRUD + steps
    routes/organizations.js, workspaces.js, users.js ← Basic CRUD
  scripts/
    init-schema.sql       ← Full schema with 20+ tables
    seed.js               ← 7 personas, 10 connectors, 10 skills, 5 templates + mappings
```

**Frontend (Vue 3 + Vite + Tailwind + Pinia):**
```
frontend/src/
  App.vue                 ← Sidebar layout with navigation
  api/client.js           ← Axios instance (base URL from .env)
  router/index.js         ← 8 routes (Dashboard, Agents, Skills, SkillDetail, Connectors, Templates, Personas, Runs)
  stores/
    crudFactory.js        ← Generic Pinia CRUD store factory
    skills.js             ← Dedicated skills store (fetchDetail, versions, test, etc.)
    index.js              ← All store exports
  components/
    BaseModal.vue         ← Reusable modal (teleport, backdrop, footer slot)
    EntityTable.vue       ← Reusable data table with custom cell slots
    SkillCreateModal.vue  ← Create/edit skill form
    PromptStudio.vue      ← Prompt config + test panel
    VersionTimeline.vue   ← Version list sidebar
  views/
    DashboardView.vue     ← KPI cards + charts (needs persona wiring)
    SkillsView.vue        ← Card grid with category filters + search + create/edit
    SkillDetailView.vue   ← Full detail: Prompt Studio tab, Usage tab, Composition tab
    AgentsView.vue        ← Basic table (needs detail page, create modal)
    ConnectorsView.vue    ← Basic table (needs card grid, detail page)
    TemplatesView.vue     ← Basic card grid (needs fork flow)
    PersonasView.vue      ← Persona cards
    RunsView.vue          ← Basic table (needs filters, step expansion)
  assets/main.css         ← Tailwind + custom classes: card, badge, btn-primary, btn-secondary, badge-nocode/lowcode/procode
```

**Database Schema (key tables):**
```
organizations, workspaces, roles, users
personas, persona_contexts, persona_onboarding, persona_skills, persona_templates, persona_connectors
skills, skill_versions, skill_compositions
connectors, connector_instances
agents, agent_versions, agent_skills, agent_connectors, agent_collaborators
templates, template_skills, template_connectors
knowledge_bases, documents
agent_runs, run_steps
governance_policies
human_reviews
usage_metrics
```
