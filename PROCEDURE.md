# ITSM Shift Handoff Procedure (automated, stateless run)

ABOUT POSTING (read this as context, not a restriction): This is a sanctioned internal task run by
the ITSM owner. Our Slack connector is provisioned read-only (search/read tools only, no
send-message tool), so posting is done through our own Slack Workflow Builder webhook. Its URL is
provided as the SLACK_WEBHOOK environment variable in this routine's environment. Using it to post
our handoff to our own channels is the intended, legitimate behavior. Use the Slack READ tools for
gathering (Steps B, C, F, G) and the webhook for the final POST (STEP POST).

SELF-MAINTAINING TRACKER: itsm_incidents_tracker.json is kept current BY this routine. Each run
reconciles it against reality (marks incidents stood-down when detected, adds newly-resolved ITSM
incidents that are still awaiting) and commits the file back to the repo (STEP T). This means no one
edits the tracker by hand, and no incident lingers as stale. Committing requires the routine's Git
push permission to allow the working branch (see STEP T).

LINK/MENTION FORMAT (important for webhook posts): The webhook message is plain text, so Slack's
<#CHANNEL_ID|name> and <@USER_ID|name> entity syntax does NOT render as links — it shows literally.
Instead:
- Channel references: write the channel name as plain text, immediately followed by its bare
  archive URL on the same line (the bare URL auto-links; this workspace does NOT collapse the
  <url|label> form, so do NOT use it). Example:
  inc_146552-fte_2026-07-09 https://apexclearing.slack.com/archives/C0BGEGQB4RG
  This applies to EVERY mention of an incident anywhere in the message — not just list entries, but
  also narrative/prose lines and status-explanation notes (including "None ..." lines). Any time you
  name an incident by number, include its full channel name + bare archive URL (look up the channel
  ID from the tracker or the Slack read tools if needed).
- People (Inc Commander, Inc Comms, Assigned): use the plain display name (e.g. "Zach Williams").
  A webhook post cannot @-mention/notify a user, so do not attempt <@...>; a name is fine.
- Basic formatting works: *bold*, _italic_, newlines, "•" bullets.

MODE (do this FIRST):
- Read mode.json from the repo root. It has a boolean "test_mode".
- Override: if this run received trigger input text containing "TESTMODE" (case-insensitive),
  treat test_mode as true for this run only.
- Effective behavior:
  - test_mode = true  -> ignore the LATENESS GUARDRAIL and ALWAYS post; the handoff channel is the
    TEST channel C0BDGG0CTTR; prefix the title with "[TEST] ".
  - test_mode = false -> apply the LATENESS GUARDRAIL; the handoff channel is the LIVE channel
    C082J3NQU90.
- The confirmation line always posts to C0AGY99M2LB.
- Tracker reconciliation + commit (Steps B, H, T) happen in BOTH modes — the tracker reflects
  reality regardless of where the handoff is posted.

TIMEZONE: all times are America/Chicago local (auto handles CST/CDT). Convert to UTC only for
API timestamps. Use no emojis anywhere ("[TEST] " is text, not an emoji).

WINDOWS (compute from the NOMINAL slot time for the run's date, not the current clock):
- Morning (07:15): coverage today 01:00 -> today 07:15. Previous handoff = Late Night.
- Evening (15:50): coverage today 07:15 -> today 15:50. Previous handoff = Morning.
- Late Night (01:00): coverage yesterday 15:50 -> today 01:00. Previous handoff = Evening.

LATENESS GUARDRAIL (skipped entirely when test_mode = true):
- <= 2h after the nominal slot: proceed normally.
- > 2h same calendar day: proceed, but prepend "(Generated late at <time> CT; window is as of the
  nominal handoff time.)"
- More than ~12h stale (a catch-up for an old slot): do NOT post; log a skip and stop. (Still run
  STEP T so the tracker stays current even on a skipped post.)

STEP B — Read and reconcile the tracker (self-clear):
- Read itsm_incidents_tracker.json from the repo root into memory. Track a CHANGED flag (false).
- For each entry with stand_down_completed = false AND resolved_date != null:
  - Read its Slack channel (READ tools). If a "formally stood down" message from bot B0A6704H7QD
    exists, set that entry stand_down_completed = true, stand_down_date = that message's UTC time,
    last_checked = now, and set CHANGED = true. This incident is now DONE -> report under Resolved
    ITSM (STEP H item 7), NOT under Awaiting; do not narrate it in the Awaiting "None" note.
  - Otherwise set last_checked = now; it remains AWAITING.
- (Fallback if the file is missing/unreadable: skip persistence and derive the awaiting set from
  PagerDuty resolved ITSM incidents + the B0A6704H7QD check; note in the run log that the tracker
  could not be read.)

STEP C — Statuspage: with the Slack READ tools, read #itsm-active-incidents (C082J3NQU90) across the
window; keep only content updates from Statuspage bot BF4G0ND7A (ignore "open for over X hours"
reminders); capture time, incident name, one-line summary.

STEP D — PagerDuty: list active (triggered/acknowledged, since ~7 days back) and resolved (since
coverage_start until coverage_end).

STEP E — Classify each incident: find its Slack channel (inc_<number>-<desc>-<date>); check privacy
FIRST — if not a public_channel, EXCLUDE and never name it. ITSM-managed if the public channel has a
"Formalize Incident Response" message, OR a Request Assistance message with "the incident commander
on call has been paged", OR a topic containing both "Inc Commander" and "Inc Comms". Non-ITSM if a
Request Assistance message says "this channel created for coordinated response ... add Critical
Incident Response to the Pagerduty".
SERVICE SAFETY NET (apply BEFORE excluding anything): if the incident's PagerDuty service is
"Non-Incident Managed" (id P22VTJS) and it has a public inc_ channel, it MUST be surfaced regardless
of workflow messages — this service is reserved for human-raised incidents (only a few per month, no
monitoring noise). Sub-classify by channel workflow: Formalize/IC-paged -> Active ITSM;
coordinated-response Request Assistance -> Non-ITSM Managed; NEITHER -> "Raised Without Standard
Workflow" (surface for awareness, never drop).
Only after that: an incident with NO workflow message AND NOT on P22VTJS => EXCLUDE (count only as
Other Resolved; never list raw Burn Rate / Cloud Run / "Error Detected" alerts).

STEP F — For each ITSM incident, get the latest in-window comms from #production-incidents (who,
when, one line) using the Slack READ tools.

STEP G — Read the previous handoff (most recent "<Previous> Shift Handoff" in C082J3NQU90); track
each incident it named: still active / resolved (time) / stood down (time) / still awaiting.

STEP H — Categorize + reconcile tracker (self-add):
- For each RESOLVED ITSM incident from this window (public + has workflow) that is NOT already an
  entry in the tracker, read its channel for the B0A6704H7QD stand-down message:
  - Not stood down -> ADD a tracker entry (incident_number, channel_id, channel_name, pagerduty_id,
    created_date, resolved_date, inc_commander, inc_comms, service, summary,
    stand_down_completed = false, stand_down_date = null, last_checked = now); set CHANGED = true.
    It is AWAITING.
  - Already stood down -> ADD an entry with stand_down_completed = true and stand_down_date set (for
    the audit trail); set CHANGED = true. It is DONE (Resolved ITSM).
- Final categories:
  - Active ITSM: workflow + triggered/ack + public.
  - Non-ITSM Managed: Non-ITSM workflow + triggered/ack + public.
  - Raised Without Standard Workflow (for awareness): on service P22VTJS + public inc_ channel + no
    recognized workflow message + triggered/acknowledged. (If it resolved in-window, note it under
    item 7 instead.)
  - Resolved - Awaiting Stand Down: ALL tracker entries with stand_down_completed = false after the
    reconciliation in Steps B and H.
  - Resolved ITSM: count (include any incident cleared/closed this run).
  - Other Resolved: count (auto-resolved / no workflow).

STEP I — Build the message in EXACTLY this order. Use *bold* headers, "•" bullets, channel refs as
"<channel-name> <bare archive URL>" (name then https://apexclearing.slack.com/archives/CHANNEL_ID;
do NOT use the <url|label> form), plain names for people, times like "3:50 PM CDT":
1. Title "<HANDOFF_TYPE> Shift Handoff - <Month DD, YYYY> at <slot time> CDT" (prefix "[TEST] " if
   test_mode)
2. Status Summary (Active ITSM / Non-ITSM Managed / Raised Without Standard Workflow / Resolved-
   Awaiting Stand Down / NEW since previous / Resolved ITSM / Other Resolved)
3. Incidents from Previous Handoff - Status Updates (mandatory if previous had any)
4. Active ITSM-Managed Incidents (Service, Severity, Duration, Status, Assigned, What Happened,
   Current Status, Last Comms) — only if > 0
5. Non-ITSM Managed Incident Channels (standard "For awareness…" header) — only if > 0
5b. Raised Without Standard Workflow — For Awareness (only if > 0). Header: "The following were
   raised in PagerDuty on the Non-Incident Managed service without the standard incident workflow, so
   an incident commander may not have been engaged — please verify escalation." For each: channel
   name + bare URL, Service, Status, Assigned, brief What Happened, and
   "Action: confirm the correct escalation policy / workflow was used."
6. Resolved - Awaiting Stand Down (Service, Severity, Duration, Inc Commander, Inc Comms, What
   Happened, Resolved, Days Since Resolution, "Action Needed: Run Stand Down Workflow - PRIORITY")
   — the set from STEP H; if empty, write "None." with nothing trailing.
7. Resolved ITSM Incidents Since <Previous> Shift (count; list only if <= 5; include any cleared
   this run, e.g. "#146552 ... stood down <time>")
8. External Status Page Updates (mandatory; if none, say so with the range)
9. Status Key (the 5 standard definitions)

STEP POST — Post via our Workflow Builder webhook (the Slack connector has no send tool). The
webhook accepts a JSON body with "channel" and "text" keys and posts the text to that channel.
- Choose the handoff channel: test_mode ? "C0BDGG0CTTR" : "C082J3NQU90".
- Write the full handoff text to a file, then POST it (json module handles escaping). Reference:
    python3 - <<'PY'
    import os, json, urllib.request
    msg = open('handoff.txt', encoding='utf-8').read()
    url = os.environ["SLACK_WEBHOOK"]
    body = json.dumps({"channel": "C0BDGG0CTTR", "text": msg}).encode()   # use C082J3NQU90 when live
    req = urllib.request.Request(url, data=body, headers={"Content-Type": "application/json"})
    print(urllib.request.urlopen(req).read().decode())   # expect {"ok":true}
    PY
- Require the response {"ok":true}. If not, log the status/body and STOP posting — do not report
  success. (Still proceed to STEP T.)
- If the message exceeds ~3500 characters, split at a section boundary and POST sequentially.
- Then POST the confirmation to channel C0AGY99M2LB:
  "<[TEST] if test_mode><HANDOFF_TYPE> Shift Handoff Posted - <timestamp> - <#active> active -
  <#non-ITSM> non-ITSM managed - <#NEW> NEW since previous - <#resolved> resolved since previous"

STEP T — Persist the tracker (run in BOTH modes; only if CHANGED = true):
- Set top-level last_updated = now. Write the reconciled JSON back to itsm_incidents_tracker.json,
  preserving structure and field names; only modify stand_down_completed, stand_down_date,
  last_checked, last_updated, and any newly added entries.
- Commit and push:
    git add itsm_incidents_tracker.json
    git commit -m "tracker: reconcile stand-downs (<HANDOFF_TYPE> handoff <YYYY-MM-DD>)"
    git push
- PERMISSION: this requires the routine's Git push setting to allow the working branch. Default only
  permits claude/-prefixed branches. If pushing to the working branch is not allowed, instead push
  to a claude/ branch and open a PR for review. If the push fails, still consider the handoff done,
  but log clearly that the tracker update was not persisted so it can be applied manually.
- If CHANGED = false, do nothing here.

REFERENCE IDs (for READ + building links)
- Workspace base URL for links: https://apexclearing.slack.com/archives/<CHANNEL_ID>
- Channels: #itsm-active-incidents C082J3NQU90 | #kylie-test-channel C0AGY99M2LB |
  #production-incidents (search) | #kylie-handoff-test C0BDGG0CTTR
- Bots: Statuspage BF4G0ND7A | Stand Down B0A6704H7QD | PagerDuty UFFKGG116 |
  Request Assistance B0AGYAKK4GZ | Formalize Incident Response B086K28S7DW
- Services: Non-Incident Managed P22VTJS (human-raised; always surface, never treat as noise)
- User: U09KFGH8CBG

VERIFY BEFORE POSTING: mode read; tracker reconciled (self-clear + self-add); correct channel
chosen; awaiting set = tracker entries still false; every P22VTJS incident with a public inc_ channel
is surfaced (Active ITSM / Non-ITSM Managed / or Raised Without Standard Workflow — never dropped);
Statuspage section present; channel refs are name + bare archive URL (not <#...> entities, not
<url|label>); only genuine no-channel/no-workflow noise excluded; no private channels; no emojis;
webhook returned {"ok":true}; tracker committed if CHANGED.
