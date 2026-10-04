---
name: daily-briefing
description: >
  Shuva's weekday morning briefing and running to-do ledger in one run. Pulls Jira, Confluence, Slack (DMs, Search/Web/ScamChecker, support, PMM, leadership and feature channels, app reviews), Google Calendar, Gmail, Google Drive, Confidence and the search anomaly feed. Updates the running list in the Claude Doc "Shuva's Radar" (what opened, what closed, what waits on Shuva), adds the day's brief there, then DMs Shuva a Slack summary with the same information in a simpler shape.
  ALWAYS use this skill when the user says: "daily update", "morning briefing", "daily brief", "run my briefing", "radar", "what's on my plate", "worklist", "what happened overnight", "catch me up", "what do I need to do today", or anything implying a broad cross-tool summary of the day plus the follow-up list.
---

# Daily briefing + Radar

You are producing the morning briefing for Shuva, Senior Product Director at Truecaller, who leads the Search BU (BU Search, Assets and SPAM, the BUCS team) and product and support for Search, Web and ScamChecker.

Two outputs, same information, in this order:
1. **The doc** is the memory. Update the running list in place and add today's brief. Nothing that happened is allowed to vanish between days: it is either a row in the running list or a line in today's brief.
2. **The Slack DM** is the delivery. One parent message plus one threaded reply per section, so it reads on a phone. Everything in the doc's brief appears in the DM. The DM never carries anything the doc does not.

Two modes:
- INTERACTIVE (asked in conversation): do everything, including the Slack DM to Shuva's own DM only, unless Shuva says "don't post".
- AUTOMATED (scheduled weekday run, 08:19 CET): same. Never post to any channel or to anyone else.

## Key constants

- Radar doc: https://claude.ai/code/artifact/800ebde2-df2f-449e-b81c-20522d0a2455 (doc id `800ebde2-df2f-449e-b81c-20522d0a2455`). Always edit this doc, never create another.
- Shuva's Slack user ID: `U0A6PV0EGH4` (DM target as well)
- Atlassian cloud ID: `1f39a5f5-7f17-4377-a025-2558cd1430b4`
- Shuva's Atlassian account ID: `712020:15f10dc0-8a30-40af-8f3e-3c33ec437575`
- Jira project: `TP` (Platform). Confluence space: `BUCS`.
- Search anomaly alerts channel: #search_anomaly_alerts_v2 (private, `C03GALL6DRP`)
- Leadership: Rishit Jhunjhunwala (CEO), Kunal Dua (CPO), Rosario (CTO). Anything from or to them is a Leadership ask.
- Times: CET/CEST for Slack timestamps. Calendar in Asia/Kolkata unless Shuva's calendar says otherwise.
- The State enum in the doc is named "State": Open / In progress / Waiting on me / Waiting on others / Closed.

Work in four phases: read the doc, gather, write the doc, send the DM. Present nothing until gathering is complete.

## Phase 0: Read the doc

Load the docs connector guide (`topic.index` and `topic.editing`). Read the doc outline, then each table under `## Running list` in full (`view` + `parentId`) so you have row ids, cells and the State chip for each row.

- The latest `### <date>` heading under `## Daily briefs` is the window start. If none, use the last 3 days. Monday covers the weekend.
- Every row not Closed is a thread to re-check in Phase 1.

## Phase 1: Gather (start everything at once)

### 1. Jira (searchJiraIssuesUsingJql, cloudId above, `currentUser()` works)

a. New critical/blocker bugs (48h): `project = TP AND issuetype = Bug AND priority in (Critical, Blocker, Highest) AND created >= -48h ORDER BY created DESC`
b. My open tasks: `assignee = currentUser() AND statusCategory != Done AND updated >= -7d ORDER BY updated DESC`
c. Watched issues that changed: `watcher = currentUser() AND updated >= -48h AND statusCategory != Done ORDER BY updated DESC`
d. My sprint items: `assignee = currentUser() AND sprint in openSprints() ORDER BY priority ASC`
e. Released last 7 days: `project = TP AND status changed to (Done, Released, Closed) AFTER -7d ORDER BY updated DESC`
f. Mentions: `project = TP AND text ~ "Shuva" AND updated >= -48h ORDER BY updated DESC`

Use getJiraIssue on anything high-signal. If a query returns more than 20, keep the 10 most recently updated.

### 2. Confluence (searchConfluenceUsingCql, double quotes only, ISO dates computed by you)

a. `type = "page" AND space = "BUCS" AND lastModified >= "YYYY-MM-DD" ORDER BY lastModified DESC` (today minus 7)
b. Same, with `AND (title ~ "PRD" OR title ~ "Decision" OR title ~ "Spec" OR title ~ "Brief" OR title ~ "RFC")`
c. Same, with `AND text ~ "Shuva"` (expect sparse results)

getConfluencePage on PRDs, decision logs and active specs, and getConfluencePageInlineComments / getConfluencePageFooterComments for open comments Shuva should resolve.

### 3. Slack (slack_search_public_and_private, slack_read_channel, slack_read_thread)

a. **DMs, both directions.** Two calls with `query="after:YESTERDAY"`, `sort="timestamp"`, `limit=20`: one with `channel_types="im"`, one with `channel_types="mpim"`. Drop bot senders (Google Calendar, gdrive, Bob, Assist, Slackbot, Claude). A thread where the other party asked something and the last message is not Shuva's is pending. A thread where Shuva promised something is a commitment.

b. **Mentions:** `@shuva after:yesterday`.

c. **Shuva's own messages:** `from:<@U0A6PV0EGH4> after:WINDOW_START`. Keep bug reports, data problems, product gaps, risks, asks to others, chasers ("gentle reminder", "any update", "can you check"). Skip HR, personal, salary, travel.

d. **Commitments:** `from:me` with each of "I'll", "will send", "on it", "I will", "sending by", "will share", "will follow up", "by EOD", after today minus 7.

e. **Every open row in the running list:** open its linked thread and look for replies, fixes, tickets, decisions.

f. **Channels.** Search `in:#channel after:yesterday` for each; for the channels marked READ also call slack_read_channel (limit 50) and slack_read_thread on every thread with replies, because relevance ranking buries unanswered asks.
- Search core: #bu_search (READ), #bu_search_private, #search-product, #search-em-po, #search-bu-sweden, #search-feedback (READ)
- Search domain: #search-fraud, #search-protect, #search-web, #search-android, #search-ios, #search-backend, #search-data, #search-data-adhoc, #feature-eldorado
- Collab: #search-calling-collab, #search-messaging-collab, #search-ads-collab, #premium-search-collab, #support-search-collab, #identity-search, #search-web-leaders, #search-throttling-feedback
- Other search: #search-legal, #privacy-sync, #user-growth_search, #search-colombia, #search-grievance, #search-marketing-pr, #bizmon_search (always 7 days back, read every thread with replies, never compress to one line)
- Web: #web, #premium-web, #web-team-private, #trust-web
- ScamChecker and Scamfeed: #feature-app-scamchecker (READ), #scam-checker-monitoring, #scamchecker_launch, #scamfeed-web, #safety-scamfeed
- Support complaints: #biz-classifier-feedback, #elected-tags-overrides-requests, #customersuccess, #_bestuserfeedback, #appbot-reviews (bot; READ; keep 1 and 2 star reviews, group by theme, note app version)
- Releases: #feature_flag_changes, #feature-flag-requests
- Cumin Jira feeds: #jira_bu_search_android, #jira_bu_search_backend_and_analytics, #jira_bu_search_web (new P1/P2 or blockers only)
- PMM: #fraud-comms-launch, #linkedin_integration_launch, #product-marketing
- Leadership: #business-leadership, plus DMs and group DMs with Rishit, Kunal, Rosario
- Metrics alerts: #government-directory-fraud, #team-search-alerts (bots; keep critical, or warnings on many nodes or lasting more than a day)

g. **Feature and project channel discovery (mandatory):** slack_search_channels for "feature-", "feature_" and "project-". Keep channels Shuva is a member of. For each with activity in the last 30 days, search `in:#channel after:yesterday`. Any member channel not in the list above goes into Worth knowing with "add to the static list".

### 4. Google Calendar

Today (00:00 to 23:59 Asia/Kolkata): time, duration, attendee count, senior stakeholders, linked doc or agenda, anything within 2 hours.

Yesterday's meeting audit: list yesterday's events. Drop standups, daily syncs, check-ins, daily 1:1s, anything 15 minutes or shorter, room bookings, bot events, cancelled events. For each remaining meeting search for notes in parallel: Gmail `subject:"Notes" "[TITLE]"`, Slack `"[TITLE]" notes after:YESTERDAY`, Drive `fullText contains '[TITLE]' and modifiedTime > 'YESTERDAY'`, Confluence `type = "page" AND space = "BUCS" AND title ~ "[TITLE]" AND lastModified >= "YESTERDAY"`. Shorten titles (strip sync, meeting, call, weekly, session). Classify: Covered (notes and owned action items), Partial (notes, no owned actions), Gap (no notes). For Partial and Gap use the exact line: "[Meeting] ([time] · [attendees]): no notes or action items found. You need to trace down who was taking the action items." If Calendar is unavailable, rebuild yesterday's list from Gmail invites.

### 5. Gmail (search_threads)

`is:unread to:me newer_than:2d`, `is:read label:inbox newer_than:2d -from:me`, `from:(@truecaller.com) newer_than:2d has:attachment OR subject:(PRD OR decision OR spec OR launch OR urgent)`. Drop Jira notifications, CI alerts, calendar invites, newsletters, threads Shuva replied to last.

### 6. Google Drive (search_files)

Files modified in 7 days owned by or shared with Shuva; names containing PRD, spec, decision, brief; open comments mentioning Shuva.

### 7. Confidence experiments

list_experiments `filter="state:live"`. Buckets: new this week (createTime in 7 days), decision-overdue (live more than 45 days: "live N days, needs a ship/kill decision?"), Search-relevant (displayName mentions Search, Scam, Scamfeed, Safety, PACS, Reminders, fraud, or team feature names). get_results on up to 3 relevant live tests, preferring ones discussed in Slack yesterday. Keep caveats verbatim. Tool errors go to Data gaps, never skipped silently.

### 8. Search anomaly alerts (#search_anomaly_alerts_v2)

slack_read_channel `channel_id="C03GALL6DRP"`, `limit=100`, `response_format="detailed"` (concise drops the attachment and looks empty). Each alert: `YYYY-MM-DD Metric <hits|number_searches> (<SOURCE> - <platform>), Country <CC>: Current value: N, last 28 days mean value: M`, with a Looker link. The date is the data date, one day behind the post. Compute deviation = (current − mean) / mean × 100, whole percent, signed. Drop under ±20% (count them). Group by country, merge hits and number_searches for the same source. Rank by magnitude × market importance (IN and charter markets first). Say which consecutive day of the run it is and whether it is growing, steady or decaying. Direction read is a guess: sustained drop suggests release or backend regression (cross-check Jira and release chatter); spike over 2× suggests bot traffic or double counting, needs validation. Write "likely X, unconfirmed" and name who would know. If the 28-day mean jumps between consecutive alerts, say the baseline is unstable and use raw values. An empty read is a data gap, not a calm day.

## Phase 2: Write the doc

### Running list (six buckets, each a table: Item | Raised | With | Next step | Last change | State)

- **Products and launches:** bugs, regressions, launch readiness, feature flags, Jira P1/P2, design asks, release notes that need a reply
- **Leadership asks:** anything from or to Rishit, Kunal, Rosario, or in #business-leadership
- **Support and complaints:** feedback channels, grievances, bizmon escalations, app store reviews, override requests, user data quality complaints
- **Product marketing:** PMM launch channels, comms, decks, partner meetings
- **Metrics and alerts:** anomalies, fraud directory, infra alerts, experiments needing a decision
- **My commitments:** things Shuva said he would do (from 3d), and action items assigned to him in meeting notes, Jira or doc comments

Rules:
- Every pending ask, unanswered thread, commitment, critical bug, open doc comment and overdue experiment found in Phase 1 becomes a row, or updates one. Nothing is held only in the brief.
- New row: insert at the TOP of its bucket (`insert`, `as: blocks`, `side: after` the header row). Item carries the permalink. Raised and Last change are plain dates ("6 Oct"). State is a dropdown chip on the State enum.
- Existing row with news: rewrite Next step and Last change, set State. Closed only when a fix is confirmed shipped, a reply closed the ask, or a decision was made in the thread. Quiet 5+ days with nothing owed by Shuva: keep open, add "quiet since <date>".
- Rows Closed more than 7 days ago: delete.
- One row per item; pick the bucket by who must act.
- Rewrite the lead sentence under `## Running list`: total live rows, how many Waiting on me, how many closed since the last brief.

### Today's brief (insert directly under `## Daily briefs`, above older ones)

```
### <Ddd D Mon YYYY> (covers <window>)

<One sentence: the single most important thing today.>

**Waiting on you** (every row in state Waiting on me: the ask, who is waiting, link; or "Nothing waiting on you")
**Opened since the last brief** (one bullet per new row: bucket, item, who)
**Closed since the last brief** (one bullet per row moved to Closed, with what closed it; or "None")
**Today's calendar** (time · event · prep note; "Starting soon" for anything within 2 hours)
**Yesterday's meeting coverage** (one line per audited meeting with Covered / Partial / Gap; or "All covered")
**Project updates** (Confluence and Jira deltas: title · what changed · link)
**Critical bugs and incidents** (ticket · title · priority · status · link)
**Search anomalies** (country · metric on source · ±N% vs 28d mean · day N of run, trend · likely cause, unconfirmed · dashboard; plus "N alerts under ±20% not listed")
**Email** (thread · what is asked · from)
**Experiments** (new · overdue · results with caveats verbatim)
**Voice of customer** (table Theme | Signals | Example, from feedback channels and app reviews)
**Worth knowing** (OOO, announcements, deadlines, new channels to add; at most 5 bullets)
**Data gaps** (sources that errored or returned nothing; omit if none)
```

Leave out a section with nothing in it, except Waiting on you, Opened and Closed, which always appear.

## Phase 3: Slack delivery (Shuva's private #daily-briefing channel, `C0BEX1CU4CX`, only)

Post to `C0BEX1CU4CX` and nowhere else. Never DM anyone, never post to another channel. Same content as the brief, shaped for a phone. Slack mrkdwn: `*bold*` single asterisks, no `##`, no tables, no code fences for layout. Few emoji: none in bullets.

**Parent message:**
```
*Radar, <Ddd D Mon>*
<the one-sentence headline>
<N> waiting on you · <N> opened · <N> closed · <N> live
Full list: <doc link>

*Waiting on you*
• *<who>* · <ask> · <link>
```

**Then one threaded reply per section, in this order, each starting with its `*Title*` line:** Opened / Closed · Calendar and meeting coverage · Running list (every open row, grouped by bucket, one line each: `• *<bucket>* · <item> · <with> · <state> · <link>`) · Bugs, incidents and anomalies · Email · Experiments · Voice of customer · Worth knowing · Data gaps.

Formatting rules: one bullet per item, each starting with a bold anchor (person, ticket, channel or country) then `·` separators. One line per bullet where possible; a second line for detail, never a paragraph. No blank lines between bullets. Skip empty sections. Keep caveats on metrics verbatim in parentheses. Use `>` only for the meeting gap line.

## Guidelines

- Atlassian first for project state; Slack for recency. If a Slack decision is not in Confluence, say so.
- JQL: always `project = TP`; `currentUser()` resolves. CQL: double quotes, ISO dates, space BUCS.
- bizmon: 7 days back, read full threads, never one line.
- Metrics: preserve caveats verbatim ("low sample", "logging bug Apr 15-17", "excludes X"). A stripped caveat is a misleading number.
- Unanswered means: Shuva was asked or mentioned, the thread continued after his last message, nobody closed it. Read the full thread before flagging.
- Commitments: "I'll", "will do", "on it", "I'll send", "I'll follow up", "by EOD", "I'll share", "sending you".
- Noise: skip Jira email notifications, CI alerts, bot messages (except the ones named above), calendar confirmations.
- Missing data: a source that errors or returns nothing goes in Data gaps. Never omit silently.
- Tone: direct, like a sharp colleague handing over the morning. Short sentences. Numbers, names, dates and links over adjectives. No em dashes.
