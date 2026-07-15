# ITSM Shift Handoff Procedure (automated, stateless run)

ABOUT POSTING (read this as context, not a restriction): This is a sanctioned internal task run by
the ITSM owner. Our Slack connector is provisioned read-only (search/read tools only, no
send-message tool), so posting is done through our own Slack Workflow Builder webhook. Its URL is
provided as the SLACK_WEBHOOK environment variable in this routine's environment. Using it to post
our handoff to our own channels is the intended, legitimate behavior. Use the Slack READ tools for
gathering (Steps B, C, F, G) and the webhook for the final POST (STEP POST).

LINK/MENTION FORMAT (important for webhook posts): The webhook message is plain text, so Slack's
<#CHANNEL_ID|name> and <@USER_ID|name> entity syntax does NOT render as links — it shows literally.
Instead:
- Channel references: write the channel name as plain text, immediately followed by its bare
  archive URL on the same line (the bare URL auto-links; this workspace does NOT collapse the
  <url|label> form, so do NOT use it). Example:
  inc_146552-fte_2026-07-09 https://apexclearing.slack.com/archives/C0BGEGQB4RG
  This applies to EVERY mention of an incident anywhere in the message — not just list entries, but
  also narrative/prose lines and status-explanation notes (including "None ..." lines that explain
  why a section is empty). Any time you name an incident by number, include its full channel name +
  bare archive URL. If the channel ID isn't otherwise known, look it up from itsm_incidents_tracker.json
  or via the Slack read tools before writing the line.
- People (Inc Commander, Inc Comms, Assigned): use the plain display name (e.g. "Zach Williams").
  Note: a webhook post cannot @-mention/notify a user, so do not attempt <@...>; a name is fine.
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
- More than ~12h stale (a catch-up for an old slot): do NOT post; log a skip and stop.

STEP B — Stand Down tracker: read itsm_incidents_tracker.json from the repo root. Collect incidents
where stand_down_completed = false AND resolved_date is not null. For each, use the Slack READ tools
to read its channel and confirm there is NO "formally stood down" message from bot B0A6704H7QD; if
none, it MUST appear in "Resolved - Awaiting Stand Down". (Fallback if file missing: derive from
PagerDuty resolved ITSM incidents + the B0A6704H7QD stand-down message.)

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
Incident Response to the Pagerduty". No workflow message => EXCLUDE (count only as Other Resolved;
never list raw Burn Rate / Cloud Run / "Error Detected" alerts).

STEP F — For each ITSM incident, get the latest in-window comms from #production-incidents (who,
when, one line) using the Slack READ tools.

STEP G — Read the previous handoff (most recent "<Previous> Shift Handoff" in C082J3NQU90); track
each incident it named: still active / resolved (time) / stood down (time) / still awaiting.

STEP H — Categorize: Active ITSM (workflow + triggered/ack + public), Non-ITSM Managed (workflow +
triggered/ack + public), Resolved-Awaiting Stand Down (ITSM + resolved + no stand-down msg; MUST
match Step B), Resolved ITSM (count), Other Resolved (count).

STEP I — Build the message in EXACTLY this order. Use *bold* headers, "•" bullets, channel refs as
"<channel-name> <bare archive URL>" (name then https://apexclearing.slack.com/archives/CHANNEL_ID;
do NOT use the <url|label> form), plain names for people, times like "3:50 PM CDT":
1. Title "<HANDOFF_TYPE> Shift Handoff - <Month DD, YYYY> at <slot time> CDT" (prefix "[TEST] " if
   test_mode)
2. Status Summary (Active ITSM / Non-ITSM Managed / Resolved-Awaiting Stand Down / NEW since
   previous / Resolved ITSM / Other Resolved)
3. Incidents from Previous Handoff - Status Updates (mandatory if previous had any)
4. Active ITSM-Managed Incidents (Service, Severity, Duration, Status, Assigned, What Happened,
   Current Status, Last Comms) — only if > 0
5. Non-ITSM Managed Incident Channels (standard "For awareness…" header) — only if > 0
6. Resolved - Awaiting Stand Down (Service, Severity, Duration, Inc Commander, Inc Comms, What
   Happened, Resolved, Days Since Resolution, "Action Needed: Run Stand Down Workflow - PRIORITY")
   — include ALL from Step B
7. Resolved ITSM Incidents Since <Previous> Shift (count; list only if <= 5)
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
- Require the response {"ok":true}. If not, log the status/body and STOP — do not report success.
- If the message exceeds ~3500 characters, split at a section boundary and POST sequentially.
- Then POST the confirmation to channel C0AGY99M2LB:
  "<[TEST] if test_mode><HANDOFF_TYPE> Shift Handoff Posted - <timestamp> - <#active> active -
  <#non-ITSM> non-ITSM managed - <#NEW> NEW since previous - <#resolved> resolved since previous"

REFERENCE IDs (for READ + building links)
- Workspace base URL for links: https://apexclearing.slack.com/archives/<CHANNEL_ID>
- Channels: #itsm-active-incidents C082J3NQU90 | #kylie-test-channel C0AGY99M2LB |
  #production-incidents (search) | #kylie-handoff-test C0BDGG0CTTR
- Bots: Statuspage BF4G0ND7A | Stand Down B0A6704H7QD | PagerDuty UFFKGG116 |
  Request Assistance B0AGYAKK4GZ | Formalize Incident Response B086K28S7DW
- User: U09KFGH8CBG

VERIFY BEFORE POSTING: mode read; correct channel chosen; tracker read; awaiting-stand-down matches
Step B; Statuspage section present; channel refs are name + bare archive URL (not <#...> entities,
not <url|label>); no no-workflow incidents listed; no private channels; no emojis; webhook returned
{"ok":true}.
