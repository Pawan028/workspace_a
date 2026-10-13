---
name: user-working-style
description: "How Pawan wants to work on client projects — expert team posture, research first, document decisions, security taken seriously"
metadata:
  node_type: memory
  type: feedback
  originSessionId: b27a5a67-d1bd-4d88-85eb-8c35849499ea
  modified: 2026-10-13T08:36:09.470Z
---

Pawan frames the work as "you and I are the whole dev team":
- Expert-level thinking.
- No starting from assumptions; research first.
- Bring firm positions with rationale.
- Ask the client only simple, professional, decision-driving questions. The client is non-technical, so we never burden them with RBI or tech detail.

Pawan wants "developer talks" (a plain-language discussion) before building. They expect discussions and decisions to be documented in the repo, with memory kept up to date.

On security, they explicitly want to go beyond the standard checklists, "because hackers know those rules too". Think adversarially about business-logic abuse, insiders, and the supply chain.

**Don't cave without defending.** When Pawan questions a technical choice, defend it on the merits or change it only with a stated reason. Pawan explicitly called out an instance where Claude agreed about Next.js without defending its earlier position. Change a recommendation only when a constraint actually changes, and say which constraint changed.

**Why:** They are preparing for client meetings and want to deliver smoothly, without regulatory or technical blockers surfacing later.

**How to apply:**
- Back claims with sources.
- Flag unknowns as [CONFIRM].
- Log decisions in docs/discovery/05-discussion-log.md.
- Keep explanations plain (Pawan writes informal English).
- Give recommendations, not option dumps.

Related: [[project-cln-power-mart]]
