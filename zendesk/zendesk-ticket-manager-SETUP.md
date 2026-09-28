# Zendesk Ticket Manager — per-instance setup

Human setup steps for the `zendesk-ticket-manager` skill. The agent never performs these;
it only consumes the result. Do this once per Zendesk instance.


Whoever onboards a Zendesk instance does this once. Either route alone is sufficient; doing
both gives a cheap primary and a never-stale fallback.

**On Amazon Quick Desktop** the two routes combine: after any live lookup the skill rewrites
`ticket-fields_<subdomain>.csv` in the space itself (Step 4b), so the manual export below is
only needed to seed the map the first time — or not at all, if you are happy for the first
custom-field request to go live. On Quick Suite web there is no write action, so the
manual export is the only way the map gets updated.

**Space route.** Three steps, and the third is the one that gets forgotten:

1. Create a space named `Zendesk Field Map`.
2. Add the field map file, named `ticket-fields_<subdomain>.csv`. Produce it by exporting
   `https://<subdomain>.zendesk.com/api/v2/ticket_fields.json` while signed in and running
   the projection below. You MUST NOT put the raw export in the space — it is ~74 KB, most
   of it option lists, against well under 1 KB for the projection, and the whole point of
   this route is that the map is small enough to read exactly.

   The projection keeps only **active, customer-created custom fields**:

   - **Drops Zendesk-owned fields** — those whose `key` starts with `standard::` (AI triage
     topic / sentiment / language and their confidences, ticket summaries, approval status,
     resolution tier, messaging reminders). They are auto-populated by Zendesk features and
     are not something a user sets through this skill. In a fresh instance they are the
     majority of "custom" fields, so the map can legitimately be one or two rows.
   - **Drops system-type rows** (`subject`, `description`, `status`, `priority`,
     `tickettype`, `group`, `assignee`, `custom_status`). These are standard ticket
     attributes, never `custom_fields`; leaving them in only invites a title match that
     routes to the wrong place.
   - **Joins option tags with a space** rather than a comma, so `option_values` introduces
     no embedded commas; `@csv` still quotes any field that needs it. Zendesk tags never
     contain spaces, so the separator is unambiguous.
3. **Attach the space to the agent** that runs this skill. Spaces bind to agents, not to
   skills, so a space that exists but is not attached is invisible here.

```bash
SUB=your-subdomain
{
  printf 'id,type,title,option_values\n'
  jq -r '.ticket_fields[] | select(.active==true) | select(.key == null)
         | select(.type | IN("subject","description","status","priority",
                             "tickettype","group","assignee","custom_status") | not)
         | [ (.id|tostring), .type, .title,
             (if .custom_field_options
              then ([.custom_field_options[].value] | join(" "))
              else "" end) ] | @csv' ticket_fields.json
} > "ticket-fields_${SUB}.csv"
```

**Live lookup route.** Nothing to install. The agent runs in the user's browser (Amazon Quick
Desktop / browser agent), so with the user signed in to Zendesk it can open
`https://<subdomain>.zendesk.com/api/v2/ticket_fields.json` and
`/api/v2/tickets/{id}/comments.json` directly — the session cookie authenticates the request.
The only requirement is that the user is logged in to the right Zendesk instance in that
browser before asking.
