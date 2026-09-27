---
description: "Ethics and accountability for AI agents: who is responsible for agent actions, transparency, consent, fairness, oversight, and an agent card you can adopt."
---

# 08 · Ethics & Accountability

When software acts on people's behalf — sending messages, moving money, making or
preparing decisions about them — the ethical questions stop being abstract. This lesson
treats them as engineering requirements: things you design for, document, and can be
held to.

## Principle: an agent is never the accountable party

An agent cannot be accountable. It can't explain itself under oath, be disciplined, or
make amends. For every agent in production, a named **person or team** is accountable
for what it does, and the organisation deploying it is responsible to the people it
affects. "The AI did it" is never an acceptable explanation — to a customer, a regulator
or a colleague.

Practically, write an accountability map before launch:

| Role | Responsibility |
|---|---|
| **Business owner** | decides the agent should exist, owns its outcomes and its budget |
| **Engineering owner** | builds and operates it; on call for incidents |
| **Policy owners** | own specific rules (refund limits, data handling) and approve changes |
| **Reviewers/approvers** | make the decisions the agent prepares, with authority to override |
| **Affected people's channel** | how a customer or employee can question or appeal an outcome |

## Transparency

- **Disclose that it's an agent.** People interacting with an agent should know it,
  especially when it sends messages in someone's name. Several jurisdictions require
  disclosure of AI interaction in certain contexts; check what applies to you.
- **Label agent-authored content** sent externally, and say how to reach a human.
- **Explainability in practice.** You can't expose model internals, but you *can* show
  the evidence and steps: which records were looked up, which rule allowed the action,
  who approved it. The traces and policy decision records from earlier lessons are what
  make meaningful explanations possible.

## Consent and data

- **Act within the permission the user gave**, for the purpose they gave it. "Summarize
  my inbox" doesn't authorise replying to it.
- **Minimise data** in context, traces and memory (L2-04, L4-02). Don't collect what the
  task doesn't need.
- **Respect people who aren't users.** Agents process data about third parties — email
  senders, customers in a CRM. They didn't consent to your agent; treat their data with
  at least the care your existing policies require.

## Fairness and harm

Agents that prepare or make decisions about people (hiring screens, credit, benefits,
support prioritisation, fraud flags) can reproduce or amplify biases present in models,
data or process. Where an agent touches such decisions:

- Keep a **human decision-maker** with real authority and enough time to use it.
- **Evaluate by group**: include cases representing different populations in your eval
  set and compare outcomes, not just overall pass rates.
- **Check the proxies**: postcode, name, language style and school can stand in for
  protected characteristics.
- In many jurisdictions such uses are regulated as high-risk; involve legal and
  compliance early, not at launch.

## Oversight that is real

Human-in-the-loop only counts if the human can actually catch problems. Watch for
**automation bias** — people approving whatever the system suggests, especially under
time pressure. Countermeasures: show evidence rather than just a recommendation; track
override rates (near-zero overrides can mean great recommendations *or* rubber-stamping —
sample to find out); rotate reviewers; and never set targets that reward approval speed
alone.

## Impact on work

Agents change people's jobs. Involve the people whose work changes in design and
evaluation (L4-05), be honest about intentions, and measure whether the agent actually
reduces toil or just moves it (for example, from doing tasks to checking an agent's
mistakes).

## Worked example: an agent card

An **agent card** is a short, living document published internally for every agent —
the agent equivalent of a model card. Here is one for the Level 3 triage system,
filled in:

```text
AGENT CARD — Support Triage Assistant                     version: triage-v2 (2026-09)

Purpose        Routes incoming support tickets; resolves duplicate-charge refunds up
               to 50 EUR; files bugs; prepares all other actions for agents.
Not for        Account closures, legal/complaint escalations, anything over 50 EUR.
Owner          Support Operations (business) · Platform team (engineering, on call)
Users          Support staff (assist view); customers receive agent-labelled replies.
Tools          get_invoice (read) · search_kb (read) · open_bug (write, internal)
               issue_refund (write, money: auto ≤ 50 EUR, else approval by team lead)
Data           Ticket text, invoice metadata. No payment card data in context.
               Traces redacted; raw content retained 30 days, metrics 13 months.
Controls       Step limit 8 · policy rules R1–R9 · egress allowlist · taint gating ·
               idempotency keys on refunds · kill switch (flag: triage_agent_enabled)
Evaluation     Offline: 5-case suite ×20 trials, gate = no hard violations and outcome
               ≥ current. Online: trajectory rules on all runs, weekly review of 50
               sampled runs + all negative feedback.
Known limits   Misroutes ~3% in offline evals; mixed billing+tech tickets unsupported.
Appeals        Customers: "talk to a person" link in every reply. Staff: flag button.
Last review    2026-09-20 by Support Ops lead, Platform lead, Privacy officer.
```

Every line is checkable: someone can verify the step limit, read the rules, see the
eval results and test the kill switch. That's the difference between an ethics
statement and accountability.

## How It Actually Works

Accountability in organisations works through **traceability plus ownership**: for any
outcome, you can find what happened (records) and who is answerable (roles). Agents
threaten both — actions happen fast, at scale, without a person's name on each one.
The engineering practices in this course restore them: traces and decision records
provide the "what", approvals and policy ownership provide the "who", and the agent
card ties them together in a form non-engineers can read. Ethics becomes operational
when it's attached to artefacts that exist, are reviewed, and can be audited.

## Common mistakes

- **No named owner** — "the AI team" is not a person.
- **Disclosure buried** in terms of service rather than at the point of interaction.
- **Human review that can't realistically catch errors** (too fast, too little evidence).
- **Overall accuracy only**, hiding group-level disparities.
- **Agent cards written once** and never updated as the agent changes.

## Exercise

1. Write an agent card for the Level 1 file organizer or the Level 2 research agent.
   Which fields were hardest to fill honestly?
2. Design the "appeal" path for an agent that prepares decisions about people in your
   organisation: who receives the appeal, what records they see, and how fast they must
   respond.
3. Propose two metrics that would reveal rubber-stamping in an approval workflow, and
   how you'd respond if they did.
