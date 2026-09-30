# GLPI media type — what was changed and why

Base: the official `templates/media/glpi` from https://git.zabbix.com/projects/ZBX/repos/zabbix
(branch `release/7.4`, commit `8fc4e45de83`, 2026-08-13).

Tested on Zabbix 7.4.15 with GLPI 11.0.10 (API V2 and V1) and GLPI 12.0.0-rc3 (API V2).

## Changes to the webhook

### 1. Resolved problems add a solution to the ticket

- **Before:** on recovery, the webhook wrote `status: 5` (Solved) to the ticket and added the resolution as a followup.
- **Now:** it adds the resolution as a **solution** (`ITILSolution`), and GLPI sets the ticket status itself. The ticket title is still updated, as before.
- **Why:** in GLPI a ticket is solved by adding a solution. The resolution then appears as the ticket's solution instead of a followup, and GLPI applies its own solve logic (solution approval, auto-close delay). In our production GLPI, writing the status directly did not solve tickets that already had requesters, observers or assignees. We could not reproduce that on a clean GLPI 11.0.10 / 12.0.0-rc3, but the solution works in both cases.

### 2. Problem updates no longer change the ticket title

- **Before:** every update (comment, acknowledge, severity change) sent the update subject as the ticket title, so the title became "Updated problem in …".
- **Now:** updates add the followup and keep the urgency in sync, without touching the title.
- **Why:** the original title was lost on every comment; the followup already carries the update information.

### 3. Tickets get a requester

- **Before:** tickets were created without a requester.
- **Now:** the webhook user is set as the requester, or the user given in the new `glpi_requester_id` parameter.
  - API V1: set in the ticket creation request.
  - API V2: added right after creation (`POST /Assistance/Ticket/{id}/TeamMember`), because the V2 ticket creation has no requester field.
- **Why:** tickets without a requester are harder to route, notify and report on in GLPI.
- **Note:** with API V2, GLPI requires the **Assign** ticket permission for this (otherwise it answers 403). Setting the requester never blocks ticket creation; the result is stored in the `__zbx_glpi_requester` event tag (V2), and the new `glpi_debug` parameter makes a failure visible in the action log.

### 4. Readable GLPI errors

- **Before:** API V2 errors were shown as "Unknown INFO. Check debug log for more information."
- **Now:** errors are shown as `HTTP <code>: <message from GLPI>`.
- **Why:** only the V1 error field was read; V2 returns its message in other fields.

### 5. Requests without a body and empty responses

- **Now:** the `Content-Type: application/json` header is sent only when the request has a body, and empty responses are accepted.
- **Why:** GLPI API V2 rejects a `GET` with a JSON content type and no body (`Invalid JSON body`), and answers some requests (`TeamMember`) with an empty `201`. Needed for change 3.

### 6. Zabbix event ID in the ticket

The ticket description ends with `Zabbix event: <event ID>`, to trace the ticket back to the Zabbix event.

## New parameters (optional, empty by default)

| Name | Purpose |
|------|---------|
| `glpi_requester_id` | GLPI user ID to set as the requester. Empty = the webhook user. |
| `glpi_debug` | `1` = a failure to set the requester fails the notification, so it shows in *Reports > Action log*. For troubleshooting only. |

## Changes to the manual (README)

- **GLPI profile (API V2):** add the *Assign* ticket permission (needed for the requester).
- **Zabbix user:** use a dedicated user as the recipient of the GLPI media. Zabbix does not send update notifications to the user who made the update, so if the person commenting on problems is the only recipient, updates never reach GLPI.
- **Action:** new step — the action must have *Operations*, *Recovery operations* **and** *Update operations* via the GLPI media type. Without *Update operations*, problem updates are not sent to GLPI.
- Parameters table, behavior description and a short troubleshooting section updated accordingly. The same text was updated in the media type's built-in description.

Nothing else was changed: API endpoints, authentication, TLS validation, urgency mapping, message templates and retries are the same as the official version.
