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

STEP B2 — Awaiting-stand-down SAFETY SWEEP (do NOT rely only on the tracker or the current window;
this is what catches incidents that resolved in a previous shift's window or that a missed/late run
never captured — e.g. an incident resolved yesterday evening but not formally stood down):
- Search Slack for public channels whose name starts with "inc_" created/active in the last ~5 days.
  Naming VARIES — inc_<number>-..., inc_asc-<number>-..., etc. — so match on the incident number
  appearing anywhere in the name, not a fixed prefix.
- For each such channel, identify its PagerDuty incident (number in the channel name / the PD card).
  If that incident is RESOLVED and the channel has NO "formally stood down" message from bot
  B0A6704H7QD, it is AWAITING stand down.
- Ensure every such incident is present in the tracker (ADD it if missing, stand_down_completed =
  false, filling channel_id/name, pagerduty_id, dates, IC/comms, service, summary; set CHANGED =
  true). It then appears in "Resolved - Awaiting Stand Down" regardless of which window it resolved in
  or whether an earlier run caught it.

STEP C — Statuspage: with the Slack READ tools, read #itsm-active-incidents (C082J3NQU90) across the
window; keep only content updates from Statuspage bot BF4G0ND7A (ignore "open for over X hours"
reminders); capture time, incident name, one-line summary.

STEP D — PagerDuty: list active (triggered/acknowledged, since ~7 days back) and resolved (since
coverage_start until coverage_end).

STEP E — Classify each incident: find its Slack channel by searching for the incident NUMBER among
public channels whose name starts with "inc_" (naming varies: inc_<number>-..., inc_asc-<number>-...,
and the channel may have been renamed after creation — match on the number appearing anywhere in the
name, not a fixed "inc_<number>" prefix). Check privacy FIRST — if not a public_channel, EXCLUDE and
never name it. ITSM-managed if the public channel has a
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

STEP I — Build the message in the CHECKLIST format below. Webhook-plain-text rules: *bold* for
headers/labels, "☐" for open items, channel refs as "<channel-name> <bare archive URL>" (name then
https://apexclearing.slack.com/archives/CHANNEL_ID; do NOT use <#...|...> or <url|label> — both show
literally via the webhook), plain names for people, times like "3:50 PM CDT". Prefix the title with
"[TEST] " only if test_mode.

TOP:
  *[TEST] Incident Manager Handoff - Status & Open Items*
  <HANDOFF_TYPE> - <Month DD, YYYY> - <slot time> CDT (coverage <start> - <end>)
  Summary: <#active> active - <#awaiting> awaiting stand down - <#stood down this window> stood down -
  <#new managed> new managed - <#other resolved> other resolved

PER-INCIDENT BLOCK — one per incident that must be surfaced (Active ITSM, Non-ITSM Managed, Raised
Without Standard Workflow, anything Awaiting Stand Down, and anything stood down this window). Order:
attention-needed/active first, then awaiting stand down, then closed-this-window last.

  *<n>. #<number> - <channel-name> <bare archive URL>*
  Status: <state>
  Severity: <sev> - Service: <service>
  Assigned: <name> (<role>)
  Latest comms: <who / when / one-line of the most recent update>. If NO public comms have been posted
    to #production-incidents, say exactly: "INTERNAL ONLY - no public comms update has been posted".
  Open Items
  ☐ <action>
  ☐ <action>

OPEN ITEMS by category (for anything NOT resolved, the FIRST item is always the comms item):
- Comms item (NOT-resolved incidents only): if no public comms posted yet ->
  "☐ POST AN INTERNAL COMMS UPDATE (none posted yet - incident is not resolved)"; otherwise ->
  "☐ Post the next internal comms update in #production-incidents".
- Active ITSM: continue investigation / confirm current root-cause status; confirm recovery plan,
  owner, and timeline; confirm a mitigation is in place to prevent further impact; (if client impact)
  confirm client comms are current; resolve in PagerDuty and run Stand Down once recovered.
- Raised Without Standard Workflow (P22VTJS, no workflow): confirm the correct escalation policy /
  incident workflow was used; engage an incident manager if impact warrants; confirm remediation and
  owner; resolve in PagerDuty once verified.
- Non-ITSM Managed: monitor the channel and engage an IC if it becomes incident-managed; confirm
  current status and owner.
- Resolved - Awaiting Stand Down (no comms item - it is resolved): verify the PagerDuty incident is
  resolved; run the Stand Down Workflow; confirm fully closed after stand down; (if applicable)
  confirm any downstream/client deadline was met.
- Closed this window (stood down): show the Status line and "Open Items: ☐ None - fully closed this
  window."

AFTER the incident blocks, add:
  *Resolved - Awaiting Stand Down:* if none outstanding, write "None outstanding" and a short
  parenthetical of what cleared (e.g. "#147390 stood down 8:20 AM; #147470 stood down July 28").
  *External Status Page Updates:* MANDATORY. Each in-window Statuspage (BF4G0ND7A) content update as
  "<time> - <incident name> - <one-line>"; if none, write "None posted during coverage (<range>)".
  *Other Resolved:* "<count> auto-resolved (monitoring only; no ITSM workflow)".
  *Status Key:* Active = open incident needing attention - Awaiting Stand Down = PD resolved, Stand
  Down not run - Stood down = fully closed - Other Resolved = auto-resolved monitoring.

Previous-handoff incidents: reflect their CURRENT status inline in the blocks above (e.g. one that
moved from Awaiting to stood down appears as closed-this-window). No separate previous-handoff section
is needed in this format.

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
each surfaced incident has a "Latest comms" line, and every NOT-resolved incident's first Open Item is
the internal-comms-update item; Statuspage section present; channel refs are name + bare archive URL
(not <#...> entities, not <url|label>); "☐" used for open items; only genuine no-channel/no-workflow
noise excluded; no private channels; no emojis; webhook returned {"ok":true}; tracker committed if
CHANGED.
