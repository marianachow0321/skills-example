---
name: zendesk-ticket-manager
display_name: Zendesk Ticket Manager
description: "Creates a Zendesk ticket, or adds a comment to an existing one, with content that renders as real HTML, and optionally sets custom field values. Use when a Zendesk ticket needs formatted rich-text content (headings, bold, italic, lists, links) rather than plain text, or needs custom field values set — triggers: 'create a zendesk ticket with html', 'zendesk ticket with formatting', 'add an html comment to zendesk ticket', 'update zendesk ticket content with html', 'set a custom field on a zendesk ticket'."
icon: "🎫"
trigger: create or update zendesk ticket with html or custom fields
integration: quick_suite__zendesk_suite
inputs:
  - name: subject
    description: "Ticket subject. Used only when creating (ignored on update)."
    type: string
    required: false
  - name: body
    description: "Plain-text fallback for the comment (comment.body). Supply whenever html_body is set; offer to derive one if the user gives only HTML."
    type: string
    required: false
  - name: html_body
    description: "HTML content for the comment (comment.html_body). Rendered as real HTML by Zendesk. Required for the comment path, but omit it for a custom-fields-only update."
    type: string
    required: false
  - name: ticket_id
    description: "Existing ticket to comment on. Provided = UPDATE that ticket; omitted = CREATE a new one."
    type: integer
    required: false
  - name: custom_fields
    description: "Optional [{id, value}] pairs. id is Zendesk's NUMERIC field id — see the Field map."
    type: array
    required: false
  - name: public
    description: "true = public reply, false = internal note. Omit for connector default. Maps to comment.public."
    type: boolean
    required: false
  - name: tags
    description: "Optional ticket tags."
    type: array
    required: false
---

## Overview

Creates a Zendesk ticket, or adds a comment to an existing one, with the comment rendered
as **real HTML** and optionally with **custom field** values set. `ticket_id` decides
which: provide it to update, omit it to create.

A **create** always needs a comment — Zendesk rejects a new ticket without one (422,
"Description: cannot be blank"). An **update** needs **either** a comment **or**
`custom_fields`; an update that sets only a custom field is valid and needs no comment at
all. When a comment is wanted, `html_body` carries it and `body` is the plain-text fallback;
if the user supplies only HTML, offer to derive the `body`.

The connector's declared schema is not a reliable guide to what Zendesk accepts. Build the
payload to match Zendesk's API, not the declared schema:

1. **`html_body` is missing from the runtime `comment` schema** (which lists only `body`,
   `author_id`, `public`, `uploads`) but is accepted and rendered. A schema-strict agent
   drops it and silently loses the user's formatting.
2. **`custom_fields` is declared as an array of *strings*** but Zendesk requires an array
   of `{id, value}` *objects*. An agent that conforms to the declared type produces a
   silent no-op.

## Field map — resolving a field name to its id

Custom fields are set by **numeric field id**, and the Zendesk Suite connector has **no
list-fields action**. Ids are instance-specific: one carried over from another Zendesk either
errors or silently writes to an unrelated field. This skill therefore ships with **no field
ids of its own**.

Resolve from these two sources, stopping at the first that yields the id:

1. **The field map file in the `Zendesk Field Map` space.** Files are named
   `ticket-fields_<subdomain>.csv`, one per Zendesk instance, CSV with a header row
   (`id`, `type`, `title`, `option_values`). If the space holds exactly one, use it. If it
   holds several, pick the one whose `<subdomain>` matches the Zendesk the user is signed in
   to (the host of any Zendesk URL in the conversation, or the `ticket.url` of a ticket
   already touched). Ask only if none or more than one matches — you MUST NOT pick one by
   guessing, because ids from the wrong instance write silently to unrelated fields.

   Read it whole with `file_read` — it is small, often only a few rows — and match on
   title. The map holds only **customer-created** custom fields: standard attributes and
   Zendesk-owned auto-populated fields are excluded at export time, so anything in it is
   both settable through `custom_fields` and safe to set. A very short map is normal, not
   a broken export.
2. **If it is not in the map, ask before going live.** Tell the user the field is missing from
   the map and offer to check Zendesk directly. Open
   `https://<subdomain>.zendesk.com/api/v2/ticket_fields.json` in the browser — the user's
   signed-in Zendesk session authenticates it — only on their go-ahead, or immediately,
   without asking, if they asked for current data in the first place. The response is large,
   which is why it is never automatic. Once fetched, reuse it for the rest of the
   conversation, then **refresh the map** (Step 4b) so the next conversation does not pay for
   the same lookup. `<subdomain>` is the host of the Zendesk the user is signed in to; every
   entry's `url` in the response repeats it.

Working down this order is normal, not a failure. What you MUST NOT do at any point is guess
an id.

Three things that hold regardless of instance:

1. **A `tagger` needs its option `value`, not its display label** — Zendesk matches on the
   tag, so a label sets nothing.
2. **Standard attributes are not custom fields.** `subject`, `status`, `type`, `priority`,
   `group_id`, `assignee_id`, `custom_status_id` are declared parameters on the connector's
   own actions. You MUST NOT route them through `custom_fields`, because the write silently
   does nothing.
3. **Many "custom" fields are Zendesk-owned and auto-populated** (AI triage topic /
   sentiment / language and their confidences, ticket summaries, approval status, resolution
   tier, messaging reminders). They are deliberately left out of the map. If a live
   `ticket_fields.json` lookup surfaces one — its `key` starts with `standard::`, and its
   option tags are prefixed `zd_` or contain `__` — writing it is usually pointless and may
   leave the owning feature inconsistent. Confirm with the user first.

## Workflow

### Step 1: Confirm the connector is available
- **Mode**: `deterministic`
- **Validate**: the Zendesk Suite connector's `CreateTicket`, `UpdateTicket`, and `ShowTicket` actions appear in the available actions
- **On failure**: If they are not available, say so and stop. You MUST NOT proceed by another route or report a ticket as created, because there is no path to the ticket from here.

### Step 2: Choose create vs. update
- **Mode**: `agentic` · **Input**: `{{ticket_id}}`
- **Validate**: creating → a `subject` exists **and** `html_body` is present (Zendesk requires a comment on create); updating → `ticket_id` is an integer **and** at least one of `html_body` or `custom_fields` is present
- **On failure**: Ask whether to update an existing ticket (which one) or create a new one; if creating without content, ask for the comment; if updating with neither content nor fields, ask what the ticket should carry.

### Step 2b: Resolve custom field names to ids (only when setting custom fields)
- **Mode**: `agentic` · **Tool**: `file_read` on the space document; the browser (signed-in Zendesk session) only as fallback
- **Output**: a numeric id per field, plus the option `value` for any `tagger`
- **Validate**: every id came from a source in the Field map order, and its `type` admits the value you intend to send (table below)
- **On failure**: Name the field, say it could not be resolved, and offer to proceed without it. Accept a numeric id if the user already has one, but do not send them looking for one — a business user has no reason to know it. You MUST NOT guess an id or reuse one from another instance, because a wrong id silently writes to an unrelated field.

1. Work the two sources in the order given in the Field map section, reading the map whole
   with `file_read`.
2. Titles may be in any language, and may not match the user's wording exactly — match
   loosely, then confirm the resolved title back to the user, because a silent mismatch
   writes to the wrong field and Step 4 will report that as success.
3. The map stores `id` as quoted text. Convert it to a **number** when building
   `custom_fields` — Payload constraint 3 depends on it.
4. Shape the value to the field's `type`:

   | `type` | value to send |
   |---|---|
   | `text`, `textarea` | string |
   | `checkbox` | `true` / `false` (boolean, not the string) |
   | `integer`, `decimal` | number |
   | `date` | `"YYYY-MM-DD"` |
   | `regexp` | string matching the field's pattern |
   | `tagger` | one option `value` (never the label) |
   | `multiselect` | array of option `value`s |

5. If the field only turned up via live lookup and is Zendesk-owned (`key` starting
   `standard::`), confirm the user means to override that feature before writing.

### Step 3a: Create (no ticket_id)
- **Mode**: `deterministic` · **Tool**: the Zendesk Suite connector's `CreateTicket` action
- **Input**: `{{subject}}`, `{{body}}`, `{{html_body}}`, plus `{{custom_fields}}`, `{{public}}`, `{{tags}}` if given
- **Validate**: response has a numeric `ticket.id`; any custom fields set are reflected back
- **On failure**: If rejected because `html_body` is unknown, report the error verbatim. You MUST NOT retry without it, because that looks like success while losing the formatting.

```json
{
  "ticket": {
    "subject": "{{subject}}",
    "comment": { "body": "{{body}}", "html_body": "{{html_body}}" },
    "custom_fields": [ { "id": 1234567890, "value": "example value" } ],
    "tags": ["example-tag"]
  }
}
```

`1234567890` is a placeholder. You MUST resolve real ids via Step 2b and MUST NOT reuse any
id from this document, because ids are instance-specific.

### Step 3b: Update (ticket_id given)
- **Mode**: `deterministic` · **Tool**: the Zendesk Suite connector's `UpdateTicket` action
- **Input**: `{{ticket_id}}`, `{{body}}`, `{{html_body}}`, plus `{{custom_fields}}`, `{{public}}`, `{{tags}}` if given
- **Validate**: response shows the new comment / audit entry; custom fields reflect new values
- **On failure**: As 3a — report verbatim, never silently drop `html_body`.

`ticket_id` is a **path parameter** (`PUT /api/v2/tickets/{ticket_id}`) passed as its own
tool argument, **not** a body key. The body is only the `ticket` wrapper.

On update, `{{tags}}` goes in as **`additional_tags`**, not `tags`. `ticket.tags` on a PUT
**replaces** the ticket's whole tag list — "add the tag urgent" sent as `tags` would wipe
every existing tag with no error. `additional_tags` appends. Use `tags` on update only if
the user explicitly asks to replace all tags, and confirm first.

```json
{
  "ticket": {
    "comment": { "body": "{{body}}", "html_body": "{{html_body}}" },
    "custom_fields": [ { "id": 1234567890, "value": "example value" } ],
    "additional_tags": ["example-tag"]
  }
}
```

### Payload constraints (both branches)

These are load-bearing: without them both cases fail silently — no error, just lost
formatting or an unset field.

1. `html_body` goes in as a **sibling of `body`** inside `comment`. You MUST NOT drop it
   for being absent from the runtime schema.
2. `custom_fields` MUST be an **array of objects**, never of strings. If they are
   stringified to satisfy the declared type, Zendesk ignores them and the field stays
   empty, with no error.
3. Each `custom_fields[].id` MUST stay a **number** — do not quote it.
4. Send everything verbatim. You MUST NOT rewrite, reformat, or "clean up" the user's HTML
   or field values.

### Step 4: Verify what was stored
- **Mode**: `agentic` · **Tool**: the Zendesk Suite connector's `ShowTicket` action; the browser (signed-in Zendesk session) for the HTML check
- **On failure**: State which of the two below was confirmed and which wasn't. You MUST NOT report HTML as verified when it wasn't, because a false confirmation defeats this step.

**Custom fields — via `ShowTicket`.** `custom_fields` is in the returned `TicketObject`.
Confirm each target id holds the intended value. If one is unchanged, suspect in order:
(a) the objects were stringified, (b) a `tagger` got a label instead of an option value,
(c) the value's type does not match the field's `type` (e.g. the string `"true"` sent to a
`checkbox`), (d) the id is from another instance. Say which you ruled out.

**HTML — via the browser, not the connector.** `ShowTicket` returns a `TicketObject`, which
has no comments array, and no connector action lists comments — so the connector alone cannot
confirm `html_body`. The browser can. Take `<subdomain>` from the `ticket.url` in the Step 3
response (`https://<subdomain>.zendesk.com/api/v2/tickets/{id}.json`) and open
`https://<subdomain>.zendesk.com/api/v2/tickets/{ticket_id}/comments.json?sort_order=desc`
in the signed-in session. The **first** element is then the latest comment — the one you just
wrote (without `sort_order=desc` the list is oldest-first and the latest is the *last*
element; on update, checking the first element of the default order examines the wrong
comment).

Compare **structurally, not literally**. Zendesk normalises stored HTML: it wraps the content
in `<div class="zd-comment" dir="auto">`, strips unsupported tags and attributes (`<style>`,
`<script>`, event handlers), and may reorder attributes. The check is: the elements you sent
(`h2`, `strong`, `ul`, `a`, …) appear as elements in `html_body`, and `body` holds their
text with **no** markup. Markup appearing literally in `body` means `html_body` was dropped
and the HTML landed as text — *confirmed failure*. Elements missing from `html_body` because
Zendesk sanitised them is not a write failure; mention it as a note. Otherwise *confirmed*.

If the browser is unavailable, fall back to the weaker signal and say so:

- **Create path only:** `TicketObject.description` is the read-only *first* comment as plain
  text. Literal markup there (`<strong>`, `&lt;strong&gt;`) means `html_body` did not take
  effect — report the failure. Absence of tags is **inconclusive**, since a plain-text `body`
  fallback also yields a tag-free description. Useless on update, where `description` still
  refers to the first comment.
- Otherwise ask the user to check the rendering in Zendesk and report *unverified*.

### Step 4b: Refresh the field map (only if Step 2b did a live lookup)
- **Mode**: `agentic` · **Tool**: the space file write/update action (Amazon Quick Desktop)
- **Input**: the `ticket_fields.json` response already fetched in Step 2b
- **Output**: `ticket-fields_<subdomain>.csv` in the `Zendesk Field Map` space, replacing any
  existing file of that name. `<subdomain>` comes from the `url` of any entry in the response.
- **Validate**: a `file_read` of the written file returns the header row plus one row per
  active, customer-created custom field
- **On failure**: If no write action is available (Quick Suite web has none), skip this step
  and instead print the projected CSV rows so the user can re-upload by hand. Say plainly
  that the map was not updated. You MUST NOT report the map as refreshed unless the
  read-back succeeded. This step runs *after* the ticket outcome is known and MUST NOT
  delay or alter Steps 3–4; a failure here never changes what you report about the ticket.

Build the CSV from the fetched JSON, applying the same projection SETUP.md uses:

1. Keep only fields with `active == true` and `key == null` (Zendesk-owned fields carry a
   `key` starting `standard::`).
2. Drop system types: `subject`, `description`, `status`, `priority`, `tickettype`, `group`,
   `assignee`, `custom_status`.
3. One row per remaining field: `id` (as a string), `type`, `title`, and `option_values` —
   the `value` of each `custom_field_options` entry joined with a single space, or empty.
4. Header row `id,type,title,option_values`; quote every cell.

Write it, read it back, and tell the user how many fields the map now holds. This is the only
place the skill writes anywhere other than Zendesk. You MUST NOT write the raw JSON or add
columns — the map's value is that it is small enough to read exactly.

### Step 5: Report
- **Mode**: `agentic`

State: created or updated; the exact JSON body sent; ticket id and URL; custom fields with
their **verified** stored values; the HTML status as exactly one of *confirmed* (markup seen
in the latest comment's `html_body`), *confirmed failure* (markup in `body` or `description`),
or *unverified* (no browser check possible — tell them how to confirm); and, if Step 4b ran,
whether the field map was refreshed and how many fields it now holds, or why it was not.
