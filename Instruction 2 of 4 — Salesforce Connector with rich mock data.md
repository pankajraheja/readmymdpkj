We need to make the "Salesforce" connector fully demo-able. Users should be able to "connect" it, then pull mock lead data that looks real. The connector already exists in the seed data. Here's what to build:

=== BACKEND ===

1. Update `backend/src/routes/connectors.js` — add these endpoints:

   GET /api/connectors/:id/detail
   - Returns the connector record plus all connector_instances for it
   - Include a count of agents using this connector (join through agent_connectors)

   POST /api/connectors/:id/instances
   - Creates a connector_instance record in the connector_instances table
   - Accepts: { workspace_id, name, config, credentials, status }
   - The credentials field stores mock creds (for demo, just accept whatever JSON the user sends)
   - Default status: "active"

   GET /api/connectors/:id/instances
   - Returns all instances for this connector (optionally filtered by workspace_id query param)

   PUT /api/connectors/:id/instances/:instanceId
   - Updates an instance's config, credentials, or status

   DELETE /api/connectors/:id/instances/:instanceId
   - Removes the instance

   POST /api/connectors/:id/instances/:instanceId/test
   - This is the key demo endpoint: "test connection"
   - Returns a mock successful connection test result:
     { status: "connected", latency_ms: 142, message: "Successfully authenticated to Salesforce (sandbox)" }

   POST /api/connectors/:id/instances/:instanceId/pull
   - This simulates pulling data from the connector
   - If the connector name contains "Salesforce" (case insensitive), return rich mock lead data:
     Return an array of 5 leads, randomly picked from this pool of 10:

     { name: "Sarah Chen", company: "Meridian Health Systems", title: "VP of Operations", email: "s.chen@meridianhealth.com", deal_value: 185000, stage: "Negotiation", lead_score: 87, last_activity: "2026-02-25", source: "Webinar" }
     { name: "Marcus Johnson", company: "TechForge Industries", title: "CTO", email: "mjohnson@techforge.io", deal_value: 340000, stage: "Discovery", lead_score: 62, last_activity: "2026-02-27", source: "LinkedIn" }
     { name: "Emily Nakamura", company: "Pacific Retail Group", title: "Director of Digital", email: "e.nakamura@pacificretail.com", deal_value: 95000, stage: "Proposal Sent", lead_score: 78, last_activity: "2026-02-20", source: "Referral" }
     { name: "David Okafor", company: "Atlas Financial Partners", title: "Head of Strategy", email: "dokafor@atlasfinancial.com", deal_value: 520000, stage: "Qualification", lead_score: 45, last_activity: "2026-02-28", source: "Cold Outreach" }
     { name: "Rachel Kim", company: "Vertex Cloud Solutions", title: "VP Engineering", email: "rkim@vertexcloud.com", deal_value: 275000, stage: "Closed Won", lead_score: 95, last_activity: "2026-02-18", source: "Inbound Demo" }
     { name: "James Whitfield", company: "Beacon Logistics", title: "COO", email: "jwhitfield@beaconlogistics.com", deal_value: 150000, stage: "Negotiation", lead_score: 71, last_activity: "2026-02-26", source: "Trade Show" }
     { name: "Aisha Patel", company: "Horizon Edtech", title: "CEO", email: "apatel@horizonedtech.com", deal_value: 420000, stage: "Discovery", lead_score: 58, last_activity: "2026-02-24", source: "Partner Referral" }
     { name: "Carlos Rivera", company: "Summit Manufacturing", title: "IT Director", email: "crivera@summitmfg.com", deal_value: 210000, stage: "Proposal Sent", lead_score: 82, last_activity: "2026-02-22", source: "Webinar" }
     { name: "Nina Volkov", company: "Crestline Media Group", title: "CMO", email: "nvolkov@crestlinemedia.com", deal_value: 165000, stage: "Qualification", lead_score: 39, last_activity: "2026-02-15", source: "Google Ads" }
     { name: "Thomas Brennan", company: "Ironclad Insurance", title: "VP Partnerships", email: "tbrennan@ironcladins.com", deal_value: 390000, stage: "Closed Won", lead_score: 91, last_activity: "2026-02-19", source: "Inbound Demo" }

   - Return format: { records: [...5 random leads], total_count: 847, pulled_at: "<current ISO timestamp>", source: "Salesforce Sandbox" }
   - For any NON-Salesforce connector, return a generic mock: { records: [{ id: 1, data: "Sample record" }], total_count: 1, pulled_at: "...", source: "<connector name>" }

2. Update the Salesforce connector seed data in `backend/scripts/seed.js`:
   - Make sure the Salesforce connector has these fields set:
     category: "crm"
     auth_type: "oauth2"
     config_schema: {
       "fields": [
         { "name": "instance_url", "type": "text", "label": "Salesforce Instance URL", "placeholder": "https://yourcompany.my.salesforce.com", "required": true },
         { "name": "client_id", "type": "text", "label": "Connected App Client ID", "required": true },
         { "name": "client_secret", "type": "password", "label": "Connected App Client Secret", "required": true },
         { "name": "sandbox", "type": "boolean", "label": "Use Sandbox", "default": true }
       ]
     }
   - This config_schema will be used by the frontend to render a proper form instead of raw JSON

=== FRONTEND ===

3. Rewrite `frontend/src/views/ConnectorsView.vue`:
   - Replace the basic table with a card grid (same style as SkillsView)
   - Each card shows:
     - Connector name (bold)
     - Category as a colored badge (crm=blue, ticketing=orange, communication=green, data=purple)
     - Auth type (small text: "OAuth2", "API Key", etc.)
     - A status indicator: green dot + "Connected" if any active instance exists, gray dot + "Not Connected" otherwise
   - To know if connected: after loading connectors, make a GET /api/connectors/:id/instances for each (or better: add a query param to the list endpoint that includes instance count)
   - Clicking a card navigates to /connectors/:id

4. Create `frontend/src/views/ConnectorDetailView.vue`:
   - Route: /connectors/:id (add to router/index.js)
   - Layout:

   HEADER:
   - Back breadcrumb: "← Back to Connectors"
   - Connector name (large), category badge, auth type badge
   - Description text

   SECTION A — Connection Form:
   - If the connector has config_schema, render a dynamic form based on it:
     - For each field in config_schema.fields:
       - type "text" → text input
       - type "password" → password input
       - type "boolean" → toggle/checkbox
     - Use the field's label, placeholder, and required properties
   - If no config_schema, fall back to a JSON textarea
   - "Connect" button that POSTs to /api/connectors/:id/instances with the form data as config and credentials
   - On success, show a success message and refresh the instances list

   SECTION B — Active Connections (instances list):
   - Table or card list of existing instances
   - Each row: instance name, status (green/red dot), created date
   - "Test Connection" button → calls POST /api/connectors/:id/instances/:instanceId/test
     - Shows result inline: "✓ Connected (142ms)" in green, or "✗ Connection failed" in red
   - "Pull Sample Data" button → calls POST /api/connectors/:id/instances/:instanceId/pull
     - Opens an inline panel or modal showing the returned records as a styled table
     - For Salesforce leads: show columns for Name, Company, Title, Deal Value (formatted as $xxx,xxx), Stage, Lead Score
     - Color the lead score: green (>75), orange (50-75), red (<50)
     - Color the stage: green for "Closed Won", blue for "Negotiation", yellow for "Proposal Sent", gray for others
   - "Disconnect" button → DELETE the instance, confirm first

   SECTION C — Usage:
   - "Agents using this connector" — list of agents linked via agent_connectors
   - Same pattern as SkillDetailView's usage tab

5. Create a dedicated Pinia store `frontend/src/stores/connectors.js`:
   - Similar to the skills store pattern
   - State: items, current, instances, loading, saving, error
   - Actions: fetchAll, fetchDetail (calls /connectors/:id/detail), createInstance, deleteInstance, testConnection, pullData
   - Update stores/index.js to use this instead of the generic CRUD factory for connectors

=== TEST FLOW ===

After building, test this exact demo flow:

1. Go to http://localhost:5173/connectors
2. See the card grid — Salesforce shows as "Not Connected" with gray dot
3. Click the Salesforce card → goes to detail page
4. See the dynamic form with: Instance URL, Client ID, Client Secret, Use Sandbox toggle
5. Fill in:
   - Instance URL: https://acme-corp.my.salesforce.com
   - Client ID: 3MVG9d8...(any text)
   - Client Secret: (any text)
   - Use Sandbox: checked
6. Click "Connect" → instance created, appears in the Active Connections list with green dot
7. Click "Test Connection" → shows "✓ Connected (142ms)" in green
8. Click "Pull Sample Data" → shows a table of 5 leads with realistic names, companies, deal values
9. Go back to Connectors list → Salesforce now shows green dot + "Connected"