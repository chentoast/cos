# cos

A single-user chief-of-staff agent for Tony. Runs as scheduled Claude routines
(cloud): a morning brief, an evening reconcile, and an hourly poll for replies.
It reads Gmail + Google Calendar, produces a plain-text brief emailed to self,
and tracks action items and multi-hop coordination chains in `state/cos.json`.
It drafts but never sends to third parties.

- **Runtime procedure:** [`OPERATING.md`](OPERATING.md)
- **Guardrails + repo map:** [`CLAUDE.md`](CLAUDE.md)
- **Config (priority signals, style, learned):** [`context.md`](context.md)
- **Todos:** [`todos.md`](todos.md)
- **State:** [`state/cos.json`](state/cos.json)
- **Design rationale:** [`DESIGN.md`](DESIGN.md)

Status: v0 prototype. Slack integration is deferred to v1.
