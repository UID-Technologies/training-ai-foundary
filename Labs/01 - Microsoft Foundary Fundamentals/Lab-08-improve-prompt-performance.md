# Lab 08 — Improve the assistant without changing the model

| | |
| --- | --- |
| **Duration** | 10 minutes |
| **Previous lab** | [Lab 07](./Lab-07-evaluate-quality.md) |
| **Next lab** | — (see [README completion checklist](./README.md)) |

---

## Objective

Show that **prompt refinement** (clear business rules) can fix mis-prioritization **without** deploying a larger model or increasing cost.

---

## Before you begin

- [ ] Lab 07 baseline metrics recorded (especially priority mistakes)  
- [ ] Same deployment **`support-chat-model`**—do **not** change model or deployment during comparison  

---

## Step 1 — Identify the problem

Routine complaints may be labeled **high** priority when the customer sounds upset but impact is low.

Example user message:

```text
My order is arriving one day late.
```

Note the model’s priority **before** adding priority definitions (optional: run once and record).

---

## Step 2 — Add priority definitions

Append to system instructions (keep all prior Lab 03–06 content):

```text
Priority classification rules:

LOW:
General questions, minor inconvenience,
or no significant urgency.

MEDIUM:
Standard support issues requiring action,
including ordinary delivery delays,
payment questions and product problems.

HIGH:
Significant business impact, confirmed
financial loss, or explicitly urgent
issues requiring prompt intervention.

CRITICAL:
Severe incidents involving immediate
safety concerns or major operational
disruption requiring urgent escalation.

Do not assign HIGH or CRITICAL merely
because the customer sounds frustrated.

Use the evidence in the request to
determine priority.
```

---

## Step 3 — Test priority scenarios

Use **new conversations** for each:

**Test A**

```text
My order is one day late.
```

**Test B**

```text
My card was charged twice and the amount
is causing an urgent financial problem.
```

**Test C**

```text
Our business account is unavailable
and 200 employees cannot access
a critical service.
```

### Expected priorities (exercise target)

| Scenario | Expected priority |
| -------- | ----------------- |
| Routine one-day delivery delay | medium |
| Duplicate charge with urgent financial impact | high |
| Large-scale business service outage | critical |

Record actual priorities in a small table.

---

## Step 4 — Compare before and after

Without changing the model:

1. Re-run a subset of Lab 07 cases where priority was wrong, **or**  
2. Re-run TC02 (delivery) and TC06 (billing) and compare priority only.

| Test | Priority before Step 2 | Priority after Step 2 | Match target? |
| ---- | ---------------------- | --------------------- | ------------- |
| One day late | | | |
| Duplicate charge + financial urgency | | | |
| 200 employees blocked | | | |

**Discussion:** Improvement must be **measured**, not assumed.

---

## Verification checklist

- [ ] Priority rules appended; model/deployment unchanged  
- [ ] Three scenario tests recorded  
- [ ] At least one before/after comparison documented  
- [ ] [README workshop checklist](./README.md) reviewed for full completion  

---

## Expected outcome

More aligned priority labels on clear scenarios, with awareness that edge cases still need evaluation and human review.

---

## Workshop wrap-up

1. Complete the **[Workshop completion checklist](./README.md#workshop-completion-checklist)**.  
2. Optional next session topics: Foundry **agents**, tool calling to ticket systems, **API** integration, **tracing**, continuous evaluation.

---

## Lab 08 complete

You finished the Microsoft Foundry Fundamentals hands-on track.
