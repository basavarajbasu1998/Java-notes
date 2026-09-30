# SDLC, Agile/Scrum and AI-SDLC

## SDLC = Software Development Life Cycle
**Analogy:** building a house — requirement (what house?), design (blueprint), construction (build), inspection (test), handover (deploy), repairs (maintenance).

```
Requirement → Analysis → Design → Development → Testing → Deployment → Maintenance
     ▲                                                                      │
     └──────────────────── feedback / new requirements ─────────────────────┘
```

| Model | How it works | Use when |
|---|---|---|
| **Waterfall** | phases strictly in order, no going back | fixed, well-known requirements |
| **V-Model** | each dev phase has a matching test phase | safety-critical |
| **Iterative / Spiral** | repeat small cycles with risk analysis | big risky projects |
| **Agile (Scrum/Kanban)** | short sprints, working software each time | changing requirements (most IT today) |
| **DevOps** | dev + ops automation, CI/CD, monitoring | continuous delivery |

## Scrum flow (the daily life of a 5-year developer)
```
Product Backlog (all wishes, prioritised by Product Owner)
        │  Sprint Planning (team picks stories, estimates story points)
        ▼
Sprint Backlog ──► 2-week SPRINT
                     ├─ Daily Stand-up (15 min: yesterday / today / blockers)
                     ├─ Develop → code review → test
                     └─ Definition of Done met?
        ▼
Sprint Review (demo to stakeholders)  →  Retrospective (what to improve)  → next sprint
```
Roles: **Product Owner** (what/priority), **Scrum Master** (process helper), **Team** (builds). Artifacts: backlog, sprint backlog, increment. **Story** → **Tasks**; **Epic** = many stories. **Velocity** = points done per sprint.

**One story through the lifecycle (Jira "PROJ-101: Place order"):**
```
To Do → In Progress (branch feature/PROJ-101) → Code Review (PR) → QA (test env)
      → Done (merged, deployed) ; bug found later → new Bug ticket → fix → hotfix release
```
Testing levels: **unit → integration → system → UAT**; types: functional, regression, performance, security, smoke, sanity.

## AI-SDLC (AI in every phase)
AI does not replace the phases; it **accelerates** each one while a human stays accountable.

```
Phase            Without AI                 With AI assistant                    Human still does
────────────────────────────────────────────────────────────────────────────────────────────
Requirement      meetings, manual notes     summarise calls, draft user stories   confirm business intent
Planning         manual estimation          break epic into stories, find risks   prioritise, commit
Design           whiteboard                 propose designs, ADRs, diagrams       choose trade-offs
Coding           type everything            autocomplete, generate boilerplate    review every line
Testing          write tests by hand        generate unit tests, edge cases       validate meaning
Code review      reviewer reads all         first-pass bug/security review        final approval
Deploy/Ops       manual runbooks            log summarisation, anomaly detection  on-call decisions
Docs             often skipped              auto-generate/update docs             correctness
```

**Structured AI-SDLC flow (spec-driven — the style of your `plan / spec / design / implement` skills):**
```
Epic ─► /plan     : AI + human decompose epic into independent stories → human approves
 │
Story ─► /spec    : product spec (what & acceptance criteria) → human approves
 │
       ─► /design : prescriptive tech spec (APIs, tables, classes; ambiguities resolved) → human approves
 │
       ─► /implement: code written strictly from approved design (no new decisions)
 │
       ─► tests/review/CI ─► merge ─► /wrap-up (sync docs & code map)

Bug ─► /diagnose (root cause + fix approach, human approves) ─► /fix (code strictly from diagnosis)
```
Key principles:
1. **Human approval gates** between phases (AI proposes, human decides).
2. **Small, well-scoped inputs** → better output ("context engineering").
3. AI code goes through the **same** tests, review, security scans as human code.
4. Never paste secrets/customer data into public AI tools.
5. Measure: cycle time, defect rate, review time.

Risks: hallucinated APIs, insecure code, licence issues, over-trust. Mitigation: tests, static analysis (Sonar), code review, pinned dependency versions.
