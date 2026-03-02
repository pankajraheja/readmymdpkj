We need a fully working "Lead Qualification Bot" agent that chains: Salesforce connector (pull leads) → Lead Scorer skill (classify) → Email Drafter skill (generate follow-up). User clicks "Run Agent" and sees step-by-step execution with rich mock output. Here's what to build:

=== BACKEND ===

1. Add the agent run simulator to `backend/src/routes/agents.js`:

   POST /api/agents/:id/run
   - This is the core demo endpoint. It simulates a deterministic agent execution.
   - Step-by-step logic:

   a. Create an agent_run record: { agent_id, status: 'running', triggered_by: req.body.user_id or null }

   b. Fetch the agent's linked skills (from agent_skills table, ordered by execution_order)
      and linked connectors (from agent_connectors table)

   c. For EACH linked connector, create a run_step record:
      - step_type: "connector"
      - connector_id: the connector's ID
      - status: "success"
      - If the connector name contains "Salesforce", set:
        - output_data: Pick ONE random lead from this list (the agent "pulled" this lead):
          { name: "Sarah Chen", company: "Meridian Health Systems", title: "VP of Operations", deal_value: 185000, stage: "Negotiation", lead_score: 87, source: "Webinar" }
          { name: "Marcus Johnson", company: "TechForge Industries", title: "CTO", deal_value: 340000, stage: "Discovery", lead_score: 62, source: "LinkedIn" }
          { name: "David Okafor", company: "Atlas Financial Partners", title: "Head of Strategy", deal_value: 520000, stage: "Qualification", lead_score: 45, source: "Cold Outreach" }
          { name: "Rachel Kim", company: "Vertex Cloud Solutions", title: "VP Engineering", deal_value: 275000, stage: "Closed Won", lead_score: 95, source: "Inbound Demo" }
          { name: "Carlos Rivera", company: "Summit Manufacturing", title: "IT Director", deal_value: 210000, stage: "Proposal Sent", lead_score: 82, source: "Webinar" }
        - tokens_used: 0 (connectors don't use tokens)
        - duration_ms: random between 120-350
      - For non-Salesforce connectors: generic output_data: { records_fetched: 1 }

   d. Store the pulled lead data — the next skills will reference it.

   e. For EACH linked skill (in execution_order), create a run_step record:
      - step_type: "skill"
      - skill_id: the skill's ID

      If the skill name contains "Lead Scorer":
        - Use the lead data from step (d) to produce a classification
        - output_data based on the lead's lead_score:
          score >= 80: { classification: "🔥 Hot", score: <lead_score>, confidence: 0.94, reasoning: "<name> at <company> is in <stage> with a $<deal_value> deal. High engagement score of <lead_score> and sourced via <source>. Recommend immediate outreach.", priority: "P1", recommended_action: "Schedule demo within 48 hours" }
          score 50-79: { classification: "🌤️ Warm", score: <lead_score>, confidence: 0.87, reasoning: "<name> at <company> shows interest (<stage>) but needs nurturing. Deal value $<deal_value> is promising. Source: <source>.", priority: "P2", recommended_action: "Send case study + schedule follow-up in 1 week" }
          score < 50: { classification: "❄️ Cold", score: <lead_score>, confidence: 0.91, reasoning: "<name> at <company> is early stage (<stage>). Low engagement score of <lead_score>. Source: <source> suggests low intent.", priority: "P3", recommended_action: "Add to nurture campaign, revisit in 30 days" }
        - tokens_used: random 150-250
        - duration_ms: random 300-600

      If the skill name contains "Email Drafter":
        - Use the lead data AND the lead scorer output to generate a follow-up email
        - output_data with a realistic email:
          For Hot leads:
          {
            subject: "Following up on your interest — let's schedule a walkthrough",
            body: "Hi <name>,\n\nGreat connecting with you! I saw that <company> is evaluating solutions in the <deal context> space, and I think we're a strong fit.\n\nBased on your needs, I've put together a quick overview of how our platform can help your team:\n\n• Reduce agent deployment time by 80%\n• Integrate directly with your existing <source>-sourced pipeline\n• Full enterprise security with Azure-native infrastructure\n\nWould you be available for a 20-minute walkthrough this Thursday or Friday?\n\nBest regards,\nAlex Morgan\nEnterprise Account Executive",
            tone: "direct, high-urgency",
            word_count: 98
          }
          For Warm leads: similar but softer tone, offers a case study, less urgent CTA
          For Cold leads: even softer, offers to add to newsletter, long-term nurture
        - tokens_used: random 250-400
        - duration_ms: random 500-900

      For any OTHER skill: generic mock output as before

   f. After all steps complete, update the agent_run record:
      - status: "completed"
      - total_tokens: sum of all step tokens
      - total_duration_ms: sum of all step durations
      - completed_at: now

   g. Return the full run object with ALL steps embedded:
      {
        run_id: "...",
        agent_id: "...",
        status: "completed",
        total_tokens: 487,
        total_duration_ms: 1423,
        started_at: "...",
        completed_at: "...",
        steps: [
          { step_number: 1, step_type: "connector", name: "Salesforce", status: "success", output_data: {...}, duration_ms: 230, tokens_used: 0 },
          { step_number: 2, step_type: "skill", name: "Lead Scorer", status: "success", output_data: {...}, duration_ms: 412, tokens_used: 187 },
          { step_number: 3, step_type: "skill", name: "Email Drafter", status: "success", output_data: {...}, duration_ms: 781, tokens_used: 300 }
        ]
      }

   GET /api/runs/:runId/steps
   - Returns all run_steps for a given run, ordered by step_number
   - Join with skills table (for skill name) and connectors table (for connector name)

2. Update the seed data in `backend/scripts/seed.js`:

   After creating the "Lead Qualification Agent" template and forking it (or create the agent directly), make sure there is a pre-built agent called "Lead Qualification Bot" with:
   - builder_tier: "no-code"
   - status: "active"
   - description: "Pulls leads from Salesforce, scores them, and drafts personalized follow-up emails"
   - Linked skills (via agent_skills):
     - Lead Scorer at execution_order: 1
     - Email Drafter at execution_order: 2
   - Linked connectors (via agent_connectors):
     - Salesforce at execution_order: 0
   
   Also make sure the "Lead Scorer" skill has a published version in the seed:
   - version_number: "1.0.0"
   - status: "published"
   - prompt_config: {
       "system_prompt": "You are a B2B lead scoring engine. Analyze the lead's title, company size signals, deal stage, engagement score, and source channel. Return a classification (Hot/Warm/Cold), a confidence score, reasoning, and a recommended next action.",
       "user_template": "Score this lead:\n\nName: {{name}}\nCompany: {{company}}\nTitle: {{title}}\nDeal Value: ${{deal_value}}\nStage: {{stage}}\nEngagement Score: {{lead_score}}/100\nSource: {{source}}\n\nClassify as Hot, Warm, or Cold with reasoning."
     }
   - model_settings: { "model": "gpt-4o", "temperature": 0.3, "max_tokens": 512 }

=== FRONTEND ===

3. Create `frontend/src/views/AgentDetailView.vue` (route: /agents/:id, add to router):

   HEADER:
   - Breadcrumb: "← Back to Agents"
   - Agent name (large text), description below
   - Status badge (active=green, draft=yellow, archived=gray)
   - Builder tier badge (no-code=green, low-code=orange, pro-code=red)
   - "▶ Run Agent" button — prominent, teal/brand colored, right-aligned in header

   TABS: Overview | Runs

   TAB 1 — Overview:
   
   Left side — "Execution Pipeline" (this is the star of the demo):
   - Visual pipeline showing the execution order as a vertical flow:
     Step 0: 🔌 Salesforce → "Pull lead data" (connector badge)
     ↓
     Step 1: ⚡ Lead Scorer → "Classify hot/warm/cold" (skill badge)
     ↓
     Step 2: ⚡ Email Drafter → "Generate follow-up email" (skill badge)
   - Each step is a card showing: icon, name, type badge (connector/skill), brief description
   - Show execution_order numbers

   Right side — "Agent Configuration":
   - Linked Skills section: list of skills with "Remove" button and "Add Skill" button
   - Linked Connectors section: list of connectors with "Remove" button and "Add Connector" button
   - "Add Skill" opens a modal listing all available skills — click one to link it (POST to agent_skills)
   - "Add Connector" opens a modal listing all available connectors — click one to link it

   TAB 2 — Runs:
   - List of past runs for this agent (GET /api/runs?agent_id=:id or filter from agent_runs)
   - Each run row shows: run ID (short), status badge, total tokens, total duration, timestamp
   - Clicking a run row EXPANDS it inline to show the step-by-step trace:

     ┌─────────────────────────────────────────────────────────┐
     │ Run #a3f2... — Completed ✓ — 487 tokens — 1,423ms      │
     ├─────────────────────────────────────────────────────────┤
     │ Step 1  🔌 Salesforce          ✓ success    230ms  0t   │
     │   → Pulled: Sarah Chen @ Meridian Health ($185,000)     │
     │                                                         │
     │ Step 2  ⚡ Lead Scorer          ✓ success    412ms  187t │
     │   → Classification: 🔥 Hot (94% confidence)             │
     │   → Action: Schedule demo within 48 hours               │
     │                                                         │
     │ Step 3  ⚡ Email Drafter        ✓ success    781ms  300t │
     │   → Subject: "Following up on your interest..."         │
     │   → 98 words, direct tone                               │
     └─────────────────────────────────────────────────────────┘

   - For each step, show a summary line extracted from output_data:
     - Connector steps: show the lead name + company + deal value
     - Lead Scorer steps: show the classification emoji + confidence + recommended action
     - Email Drafter steps: show the subject line + word count + tone
   - Use colored left-borders on each step: green for success, red for failure
   - Make the full output_data viewable via a "Show Raw Output" toggle that expands to show formatted JSON

4. Wire the "▶ Run Agent" button:
   - On click, call POST /api/agents/:id/run
   - Show a brief loading state on the button ("Running...")
   - On completion:
     - Switch to the Runs tab automatically
     - The new run should appear at the top of the list, already expanded
     - Show a success banner at the top: "Agent completed successfully — 487 tokens, 1.4s"

5. Update `frontend/src/views/AgentsView.vue`:
   - Make each agent row/card clickable → navigates to /agents/:id
   - The "Lead Qualification Bot" should be visible in the list with its description and tier badge

=== TEST FLOW ===

After building, test this exact demo flow:

1. Go to http://localhost:5173/agents
2. See "Lead Qualification Bot" in the list with no-code badge
3. Click it → goes to agent detail page
4. See the Execution Pipeline: Salesforce → Lead Scorer → Email Drafter in a visual flow
5. See linked skills and connectors listed in the configuration panel
6. Click "▶ Run Agent"
7. See button show "Running..." briefly
8. Runs tab activates, new run appears expanded showing:
   - Step 1: Salesforce pulled a lead (e.g., "Sarah Chen @ Meridian Health, $185,000")
   - Step 2: Lead Scorer classified as "🔥 Hot (94% confidence)" with recommended action
   - Step 3: Email Drafter generated a subject line and email body
9. Total tokens and duration shown in the run header
10. Click "▶ Run Agent" again → different lead appears (randomized from the pool)
11. Both runs now visible in the Runs tab, expandable
12. Click "Show Raw Output" on any step → see the full JSON output