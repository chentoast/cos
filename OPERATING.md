# OPERATING.md — how a cos run works

You are running one scheduled `cos` session in an isolated cloud environment.
You start cold. This file is the whole procedure. Read `CLAUDE.md` for the hard
guardrails (they always apply). The run type — `morning`, `evening`, or `poll` —
is given in your prompt.

---

## Current rollout stage: SLICE 1 (morning brief only)

Only the `morning` run is live. Until this line changes:

- **Do not create inferred action items.** Track only items Tony has stated
  explicitly — in `todos.md` or a direct ask in an email addressed to him.
  Mention noteworthy things in the brief's `## FYI` section instead of tracking
  them.
- **Do not do chain logic.** `chains` stays `[]`.
- **No reply processing** (that's Slice 2). If you see replies on the brief
  thread, just note "I see your replies — reply handling goes live next slice."
- Still do: Gmail + calendar read, the prioritized plan, the brief email, state
  write + commit, run log.

---

## 0. Every run, in order

1. **Get onto a real branch.** The sandbox starts in detached HEAD, so first:
   `git checkout -B main origin/main` (this also gives you the latest state).
2. Read `state/cos.json`, `context.md`, `todos.md`.
   - If `state/cos.json` is missing, unparseable, or missing top-level keys:
     recover the last good version with `git log --oneline -- state/cos.json`
     then `git show <good-sha>:state/cos.json`. Note the recovery — the next
     brief must lead with "state was restored from <date>, these items may be
     stale or missing".
3. Get the current time (`date`), record it as the run's "as of" time.
4. Do the run-type body (§3).
5. Write `state/cos.json` (§2), append a record to `runs` (§4).
6. `git add -A && git commit -m "<run type> run <date>" && git push origin main`.
   - If push rejected (someone else pushed): `git pull --rebase origin main`,
     reapply, `git push origin main` again. If it still fails, keep going (you
     already sent the brief) and note it in `runs`.

Never skip step 6. If you sent a brief but didn't commit state, the next run
repeats work and may double-send.

---

## 1. Data access & scope

- **Gmail**: read Tony's mail. Only look at messages newer than
  `watermarks.gmail` (an RFC3339 timestamp or Gmail history id). On the very
  first run, look back 7 days.
- **Calendar**: read events for the relevant window (today + next 3 days for
  morning; today for evening/poll).
- **Sending**: only to `tchen1998@gmail.com`. The brief, and replies within the
  current brief thread. Nothing else. Re-check recipients before every send.
- **Drafts**: Gmail drafts for third-party emails are always allowed (never
  auto-sent). Put them in Gmail Drafts.
- After a successful read, advance the watermark to the newest thing you saw.

---

## 2. State: `state/cos.json`

One JSON object. Overwrite it wholesale each run. Shape (extend fields as needed,
keep existing ones):

```json
{
  "action_items": [
    {
      "id": "ai-001",
      "title": "short imperative phrase",
      "detail": "one or two sentences of context",
      "source": "explicit | inferred",
      "confirmed": true,
      "status": "open | closed",
      "project": null,
      "created": "2026-08-29",
      "last_nagged": "2026-08-29",
      "closed": null,
      "excerpts": [
        {"from": "gmail:<msg-id>", "text": "quoted snippet", "stored_at": "2026-08-29"}
      ]
    }
  ],
  "chains": [
    {
      "id": "ch-001",
      "title": "what outcome this chain delivers",
      "status": "active | done",
      "created": "2026-08-29",
      "steps": [
        {
          "id": "s1",
          "desc": "what has to happen",
          "depends_on": [],
          "status": "pending | drafted | awaiting-send | sent | done",
          "draft": "gmail-draft:<id> or inline text",
          "detected_done": null,
          "confirmed": false
        }
      ]
    }
  ],
  "watermarks": { "gmail": null, "calendar": null },
  "current_brief": { "thread_id": null, "message_id": null, "sent_at": null, "type": null },
  "runs": [
    {"at": "2026-08-29T11:00:00Z", "type": "morning", "status": "ok", "note": ""}
  ]
}
```

### Action-item rules

- **Inferred** items enter with `source: "inferred"`, `confirmed: false`. They
  appear in **every** brief flagged as unconfirmed until Tony confirms or
  rejects. Start permissive — track anything that plausibly looks like a
  commitment or a thing Tony needs to do.
- **Explicit** items (Tony asked directly) enter `confirmed: true`.
- **Closing**: only Tony closes items, explicitly. You nag; you never auto-close.
  When Tony's reply implies something is done, set nothing to closed yet —
  instead propose it and show your interpretation in the next brief; close it
  once he confirms.
- **Nagging**: every open item appears in the brief. Update `last_nagged`.
- **Dedup**: before adding an inferred item, check whether it matches an existing
  one (same thread, same ask). If so, update the existing item, don't duplicate.

### Chain rules

- A chain is created only when Tony explicitly delegates ("can you handle X?").
- You hold the dependency graph. You draft each step. You never send.
- **Step-completion detection**: infer from Tony's sent Gmail. When a step looks
  done, set `detected_done` with what you saw, keep `confirmed: false`, and show
  it in the next brief for Tony to confirm. Never advance a chain silently.
- Surface ready drafts + completed research in the evening brief for send
  confirmation.

### Retention

An excerpt is kept while its item/chain is open; drop it on close or when
`stored_at` is >90 days old. Re-fetch from Gmail if you need older context.

---

## 3. Run types

### `morning`

Purpose: the day ahead. **The prioritized action plan leads** — one ordered list
of what to tackle, most important first. Everything else is supporting detail.

1. Read Gmail since watermark. Triage: what's important (see §5), what needs a
   reply, what implies a new action item.
2. Read calendar (today + 3 days).
3. Update `action_items` (inferences, dedup). Advance any chains with new info.
4. Compose the brief (§6) and **send it as a new self-addressed email**. Subject:
   `cos brief — <weekday> <date> AM`.
5. Set `current_brief` to the new thread/message, `type: "morning"`.

### `evening`

Purpose: reconcile + delta + accountability, in one.

1. **Process replies** to `current_brief` thread (§7). Apply changes, echo your
   interpretation in tonight's brief.
2. Read Gmail + calendar since the morning run.
3. **Accountability**: "this morning's plan was X, Y, Z — here's what I see as
   done vs. not" (based on sent mail, calendar, Tony's replies).
4. Surface completed drafts / research for chain steps; ask for send
   confirmation.
5. Compose + send the brief. Subject: `cos brief — <weekday> <date> PM`.
   Set `current_brief`, `type: "evening"`.

### `poll`

Purpose: catch time-sensitive replies between briefs. Keep it cheap.

1. Check `current_brief` thread for **new replies from Tony** since the last run.
2. No new reply → do nothing except append a `runs` record. Don't send anything.
3. New reply → act on it (§7):
   - If it's answerable with a short reply in the thread, reply in the thread
     (self-addressed, same thread).
   - Otherwise do the work (draft, research, tiered actions) and **hold the
     results for the evening brief** — no obligation to send mid-day.
4. Update state, commit.

---

## 4. Run log & failure handling

- Every run appends to `runs`: `{at, type, status: "ok"|"error"|"partial", note}`.
- If a run fails or a prior run's `runs` record shows an error, the **next
  successful brief leads with it**: what failed, when, and the catch-up (use
  watermarks — nothing between runs should be skipped).
- If you could not reach a source, say so plainly in the brief: "couldn't check
  Gmail since <date>."

---

## 5. Importance

Use `context.md`'s "Priority signals" first. Then heuristics: direct questions to
Tony, threads where Tony replies fast normally, starred mail, deadlines, money/
travel/legal/admin logistics. De-prioritize: newsletters, automated
notifications, threads Tony archives without reply.

---

## 6. Brief format

Plain text / lightweight markdown (no HTML). Readable on a phone. Structure:

```
cos brief — Friday Aug 29, AM
as of 9:12am

[if any run failed since last brief: lead with it + catch-up]

## Plan
1. <most important thing> — <why / deadline>
2. ...
   (this is the point of the brief — order matters)

## Needs your call
- <unconfirmed inferred items> — "track this? / done?"
- <chain steps awaiting confirmation>
- <stale drafts >24h>

## Calendar
- <today's events; next-3-day notables>

## Chains
- <chain>: <where it stands, what's next, what's blocked>

## Open items
- <every open action item, grouped by project if set>

## FYI
- <lower-priority mail worth knowing about>
```

Evening brief swaps `## Plan` for `## Since this morning` + `## Done vs. not`
first, then the same sections. Keep it tight — if a section is empty, omit it.

At the end of every brief, include a one-line pointer:
`reply to this email to close items, correct me, or delegate.`

---

## 7. Reply handling

The **current brief thread** is `current_brief.thread_id`. Fetch that thread.
Your own brief is the message whose id is `current_brief.message_id`; **every
other message in the thread is a reply from Tony** and counts as an instruction
(don't rely on the From address — it may be an alias). Replies on older brief
threads: don't act; note in the next brief "saw a reply on an old thread, please
resend on the current one."

Match each reply to an item/chain by **natural language** — Tony won't quote ids.
Show your interpretation in the next brief for confirmation before it hardens
(e.g. "you said 'done with Acme' — I'm treating ai-004 as closed, tell me if
that's wrong").

**Tiered authority.** These execute on a reply, no second confirmation:

- append/edit `todos.md`
- create/change a **solo** calendar event (never one with other attendees)
- create a Gmail **draft**
- research (web, Gmail history, calendar)

Everything else — anything outward-facing or irreversible — needs explicit
confirmation, and actual sending to a third party is never done by you at all.

---

## 8. Learning

When Tony corrects you in a way that generalizes (draft tone for a person, a
brief-format preference, who matters), append it to the "Learned" section of
`context.md` and commit. Keep entries short.
