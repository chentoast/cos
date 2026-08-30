# OPERATING.md — how a cos run works

You are running one scheduled `cos` session in an isolated cloud environment.
You start cold. This file is the whole procedure. Read `CLAUDE.md` for the hard
guardrails (they always apply). The run type — `morning`, `evening`, or `poll` —
is given in your prompt.

---

## Current rollout stage: SLICE 2 (morning + evening + replies + inference)

Live now: `morning` run, `evening` run, reply processing (§7), inferred action
items (§6).

Not live yet — until this line changes:

- **No coordination chains.** `chains` stays `[]`. Don't build dependency graphs
  or do step-completion detection. If Tony delegates a multi-step task, note in
  the brief "chain tracking goes live next slice — for now I'll just draft what I
  can when you ask." Single Gmail drafts on request are fine (that's §7 tiered
  authority, not a chain).
- **No `poll` run.** Replies are processed at the next morning/evening run only.

---

## 0. Every run, in order

1. **Get onto a real branch with current state.** The sandbox starts in detached
   HEAD and its `origin/main` ref may be slightly stale:
   `git fetch origin && git checkout -B main origin/main`.
2. Read `state/cos.json`, `context.md`, `todos.md`, `reference/writing-style.md`.
   - If `state/cos.json` is missing, unparseable, or missing top-level keys:
     recover the last good version with `git log --oneline -- state/cos.json`
     then `git show <good-sha>:state/cos.json`. Note the recovery — the next
     brief must lead with "state was restored from <date>, these items may be
     stale or missing".
3. Get the current time (`date`), record it as the run's "as of" time.
4. **Process replies and do the work they ask for.** Fetch the current brief
   thread (§7); every message that isn't your own brief is an instruction from
   Tony. This is **identical for morning and evening** — Tony replies at any
   hour, and whichever run fires next owns the follow-through: apply the change,
   *and actually do the delegated work* (write the draft, run the research, make
   the tiered-authority change), then report each in `## From your replies`.
   Never leave a reply sitting for "the other run".
5. Do the run-type body (§3).
6. Write `state/cos.json` (§2), append a record to `runs` (§4).
7. `git add -A && git commit -m "<run type> run <date>" && git push origin main`.
   - If push rejected (someone else pushed): `git pull --rebase origin main`,
     reapply, `git push origin main` again. If it still fails, keep going (you
     already sent the brief) and note it in `runs`.

Never skip step 7. If you sent a brief but didn't commit state, the next run
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
  "dinner_history": [
    {"date": "2026-08-30", "ideas": ["Mapo tofu", "Oyakodon", "Chicken shawarma bowls"]}
  ],
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

Replies + their delegated work were handled in §0 step 4 — the morning run is a
full working run, not just triage. Anything Tony sent overnight (including
replies to last night's PM brief) gets done here.

1. Read Gmail since watermark. Triage: what's important (see §5), what needs a
   reply, what implies a new action item.
2. Read calendar (today + 3 days).
3. Update `action_items` — add inferred items (permissive, `confirmed: false`),
   dedup against existing, update `last_nagged`.
4. Pick three dinner ideas (§9). Morning brief only.
5. Compose the brief (§6) and **send it as a new self-addressed email**. Subject:
   `Daily briefing — <Wkd Mon DD> (AM)`.
6. Set `current_brief` to the new thread/message, `type: "morning"`.

### `evening`

Purpose: same working run as the morning, **plus** a delta + accountability pass.
Replies + their delegated work were handled in §0 step 4.

1. Read Gmail + calendar since the morning run.
2. Update `action_items` (inferences, dedup) from what landed since morning.
3. **Accountability**: "this morning's plan was X, Y, Z — here's what I see as
   done vs. not" (based on sent mail, calendar, Tony's replies).
4. Compose + send the brief. Lead with `## Since this morning` and `## Done vs.
   not`, then the standard sections (§6). Subject:
   `Daily briefing — <Wkd Mon DD> (PM)`. Set `current_brief`, `type: "evening"`.

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

Plain text / lightweight markdown (no HTML). Readable on a phone. Write it per
`reference/writing-style.md` — plain, specific, no filler. Structure:

```
Daily briefing — Fri Aug 29 (AM)
as of 9:12am

[if any run failed since last brief: lead with it + catch-up]

## From your replies      [omit if none]
- "<what you said>" → <what I did / my reading + question>

## Plan
1. <most important thing> — <why / deadline>
2. ...
   (this is the point of the brief — order matters)

## Needs your call
- <unconfirmed inferred items> — "track this? / done?"
- <stale drafts >24h>

## Calendar
- <today's events; next-3-day notables>

## Chains      [Slice 3 — omit for now]
- <chain>: <where it stands, what's next, what's blocked>

## Open items
- <every open action item, grouped by project if set>

## Dinner      [morning only; see §9]
- <dish> — <protein>, ~<time>, <prereqs to buy>
- <dish> — ...
- <dish> — ...

## FYI
- <lower-priority mail worth knowing about>
```

Evening brief swaps `## Plan` for `## Since this morning` + `## Done vs. not`
first, then the same sections, and drops `## Dinner`. Keep it tight — if a
section is empty, omit it.

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

Match each reply to an item by **natural language** — Tony won't quote ids.

- **Direct instruction** ("close the I-9 item", "drop the hoodie one", "track
  X"): do it this run. Report what you did in this run's brief ("closed I-9 per
  your reply").
- **Implied / ambiguous** ("oh I already did that", a reply that half-mentions
  something): don't change state yet. In this run's brief, state your reading and
  ask ("you mentioned the transfer — treating that as no action needed unless you
  say otherwise"). Act on it after the next reply confirms.
- **Delegation of a multi-step task**: not yet (chains are Slice 3). Draft the
  single most useful email if one applies (§ tiered authority), and say chain
  tracking is coming.
- If a reply is unclear, ask in the brief rather than guessing.

Always list, in the brief, every reply you processed and what you did with it.

**Tiered authority.** These execute on a reply, no second confirmation:

- append/edit `todos.md`
- create/change a **solo** calendar event (never one with other attendees)
- create a Gmail **draft** — written per `reference/writing-style.md` and the
  `context.md` style guide; only assert facts you can point to a source for
  (a real email, a calendar entry) — flag anything you inferred so Tony checks it
- research (web, Gmail history, calendar)

Everything else — anything outward-facing or irreversible — needs explicit
confirmation, and actual sending to a third party is never done by you at all.

---

## 8. Learning

When Tony corrects you in a way that generalizes (draft tone for a person, a
brief-format preference, who matters), append it to the "Learned" section of
`context.md` and commit. Keep entries short.

---

## 9. Dinner suggestions

Part of the **morning** brief only. Three ideas for what to cook that night.
Ideas, not recipes — a dish name plus a short phrase. No ingredient lists, no
steps. Keep each line under ~15 words. Read Tony's constraints from the
"Food & dinner preferences" section of `context.md`; the standing ones as of
this writing:

- **Cooking for two.**
- **30–45 min, weeknight-doable.** Tony picks the labor-intensive meals himself.
- **Every idea has a heavy protein component** — ideally meat plus vegetables,
  as one dish or with a vegetable side. Name the protein in the line.
- **Lean toward the ranked cuisines** in `context.md` (Chinese, Japanese,
  Mediterranean, Levantine, Korean, Mexican) — a lean, not a quota. Variety
  across the week matters more than hitting the ranking every day.
- **Prerequisites:** call out the non-staple ingredients each idea needs to buy
  (e.g. "needs pork, eggplant"). Assume pantry staples are always on hand —
  garlic, onions, common seasonings and spices, oil, soy, vinegar, lemon, rice,
  eggs, flour, butter. Don't narrow ideas by guessing what's in the fridge.
- **No repeats:** don't suggest a dish that's in `dinner_history` from the last
  14 days.
- **Skip the section** if the calendar shows Tony out for dinner or traveling
  that night. Say nothing — don't note the omission.

After sending the brief, append `{"date": "<today>", "ideas": ["<name>", ...]}`
to `dinner_history` and trim it to the last 14 entries.
