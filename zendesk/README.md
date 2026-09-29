# Zendesk Ticket Manager — Setup

Do this once per Zendesk instance. The agent never performs these steps; it only consumes the result.

## Quick Start (Amazon Quick Desktop)

1. Create a space named **Zendesk Field Map**.
2. Add a file named `ticket-fields_<subdomain>.csv` — see [Producing the field map](#producing-the-field-map) below, or skip this and let the skill's first live lookup seed the map automatically.
3. **Attach the space to the agent** that runs this skill. Spaces bind to agents, not to skills, so a space that exists but is not attached is invisible.

That's it. On Quick Desktop the skill auto-refreshes the CSV after every live lookup, so the manual export is only needed to seed the map — or not at all if you're fine with the first custom-field request going live.

> On Quick Suite web there is no write action, so the manual export is the only way the map gets updated.

---

## Reference

### Producing the field map

Export `https://<subdomain>.zendesk.com/api/v2/ticket_fields.json` while signed in, then run the projection below. Do **not** put the raw JSON in the space — it is ~74 KB; the projection is under 1 KB.

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

### Projection rules

The script keeps only **active, customer-created custom fields**:

- **Drops Zendesk-owned fields** — those whose `key` starts with `standard::` (AI triage topic/sentiment/language, ticket summaries, approval status, etc.). They are auto-populated and not settable through this skill. In a fresh instance they may be the majority, so a one- or two-row map is normal.
- **Drops system-type rows** (`subject`, `description`, `status`, `priority`, `tickettype`, `group`, `assignee`, `custom_status`). These are standard ticket attributes, not `custom_fields`.
- **Joins option tags with a space** instead of a comma, so `option_values` never introduces embedded commas. Zendesk tags never contain spaces, so the separator is unambiguous.

### Live lookup fallback

Nothing to install. The agent runs in the user's browser, so with the user signed in to Zendesk it can open `https://<subdomain>.zendesk.com/api/v2/ticket_fields.json` and `/api/v2/tickets/{id}/comments.json` directly via session cookie. The only requirement is that the user is logged in to the correct Zendesk instance.
