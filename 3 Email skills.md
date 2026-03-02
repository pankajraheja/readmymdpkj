We need to make the "Email Drafter" skill fully demo-able with rich mock data. The skill already exists in the seed data. Here's what to build:

=== BACKEND ===

1. Update the skill test endpoint POST /api/skills/:skillId/test in `backend/src/routes/skills.js`:
   - Instead of returning a generic "[Mock Output]" string, make it skill-aware
   - If the skill's name contains "Email Drafter" (case insensitive), return a realistic mock email response
   - The mock should use the input provided and generate a believable output like:

   If input is: { "customer_name": "Sarah Chen", "inquiry": "What's the pricing for your Enterprise plan?", "tone": "professional" }
   
   Return a mock result like:
   """
   Subject: Re: Enterprise Plan Pricing Inquiry

   Hi Sarah,

   Thank you for your interest in our Enterprise plan. I'd be happy to walk you through the details.

   Our Enterprise plan starts at $2,400/year per seat and includes:
   • Unlimited AI agent deployments
   • Priority support with a dedicated success manager
   • Custom connector development
   • SSO via Azure Entra ID and advanced RBAC
   • 99.9% uptime SLA

   For teams of 50+, we offer volume discounts. I'd love to schedule a quick call to understand your team's needs and put together a tailored proposal.

   Would Tuesday at 2pm EST work for a 20-minute walkthrough?

   Best regards,
   Alex Morgan
   Enterprise Account Executive
   """

   - Have 3-4 different mock responses that rotate based on the input content (check if inquiry mentions "pricing", "demo", "support", "integration" and return different templates)
   - Each response should include realistic token counts (200-400) and latency (400-800ms)
   - For any OTHER skill, keep the existing generic mock behavior

2. Make sure the seed data for "Email Drafter" skill has a proper published version:
   - Check if the seed script already creates a skill_version for Email Drafter
   - If not, add one to the seed script with:
     version_number: "1.0.0"
     status: "published"
     prompt_config: {
       "system_prompt": "You are a professional email writer for a B2B SaaS company. Write clear, warm, and action-oriented emails. Always include a specific next step or CTA. Keep emails under 200 words.",
       "user_template": "Write a reply to {{customer_name}} who asked: {{inquiry}}\n\nTone: {{tone}}\nInclude a specific call-to-action."
     }
     model_settings: {
       "model": "gpt-4o",
       "temperature": 0.7,
       "max_tokens": 1024
     }

3. Also seed a second draft version "2.0.0" for Email Drafter with a slightly different prompt:
   - status: "draft"
   - prompt_config with system_prompt that adds "Use the customer's first name. Reference their specific question in the opening line."
   - This shows versioning in the demo

=== FRONTEND ===

4. Update the PromptStudio component (`frontend/src/components/PromptStudio.vue`):
   - In the test input textarea, add a placeholder that's specific and helpful:
     'Try: {"customer_name": "Sarah Chen", "inquiry": "What is the pricing for your Enterprise plan?", "tone": "professional"}'
   - When test results come back, render the "result" field with proper whitespace (it already uses <pre> with whitespace-pre-wrap, just verify it looks good)

5. Update the SkillDetailView (`frontend/src/views/SkillDetailView.vue`):
   - When the page loads and there's a published version, auto-select it (already does this, verify)
   - Make sure the stats bar shows the usage_count from the skill (it already does, verify)

=== TEST FLOW ===

After building, test this exact demo flow:

1. Go to http://localhost:5173/skills
2. Click the "Email Drafter" card → goes to detail page
3. See the published v1.0.0 selected in the version sidebar
4. See the system prompt and user template pre-filled in the Prompt Studio
5. In the test input, paste: {"customer_name": "Sarah Chen", "inquiry": "What is the pricing for your Enterprise plan?", "tone": "professional"}
6. Click "▶ Run Test"
7. See a realistic email response appear in the results panel with token count and latency
8. Click v2.0.0 draft in the sidebar → see the different prompt config
9. Run test again with the draft version → should still return a good mock response
10. Click "Publish" on v2.0.0 → it becomes published, v1.0.0 becomes deprecated