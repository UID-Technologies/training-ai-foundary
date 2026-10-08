# Lab 03 — Configure system instructions

| | |
| --- | --- |
| **Duration** | 10 minutes |
| **Previous lab** | [Lab 02](./Lab-02-deploy-chat-model.md) |
| **Next lab** | [Lab 04 — Few-shot and structured output](./Lab-04-few-shot-structured-output.md) |

---

## Objective

Turn the generic chat model into a **customer-support triage assistant** with defined categories, priorities, and JSON output rules.

---

## Before you begin

- [ ] Deployment **`support-chat-model`** is available  
- [ ] Chat playground is open for that deployment  

---

## Step 1 — Open a clean playground session

1. Return to deployment **`support-chat-model`**.  
2. Open the **chat playground**.  
3. **Start a new conversation** (no prior messages).  
4. Locate **System instructions**, **System message**, or **Instructions** (panel or settings).

---

## Step 2 — Test without system instructions

Leave system instructions **empty**. Send:

```text
My payment was deducted twice.
Please analyze this complaint.
```

Observe the reply (paragraph, list, or casual tone).

**Ask:** Would this format be easy for a backend API to parse automatically?

---

## Step 3 — Add system instructions (Version 1)

Paste the following into **system instructions**:

```text
You are an enterprise customer-support triage assistant.

Your responsibility is to analyze customer complaints
and return structured information.

Allowed categories:
billing, delivery, product, account, other

Allowed priorities:
low, medium, high, critical

Allowed sentiments:
positive, neutral, negative

Rules:
1. Classify complaints using only provided information.
2. Never invent missing information.
3. Use "other" when the category is unclear.
4. Use "unknown" for unavailable details when needed.
5. Keep the summary under 50 words.
6. Return JSON containing exactly:
   category, priority, sentiment, summary,
   requires_human_review.
7. Do not include explanations outside JSON.
```

Save or apply if the UI requires it.

---

## Step 4 — Test with a specific complaint

Send:

```text
My payment was deducted twice for order 8291.
I need help resolving this issue.
```

**Illustrative acceptable response** (not guaranteed verbatim):

```json
{
  "category": "billing",
  "priority": "medium",
  "sentiment": "negative",
  "summary": "Customer reports a duplicate payment for order 8291.",
  "requires_human_review": false
}
```

---

## Step 5 — Compare behavior

| Without system instructions | With system instructions |
| --------------------------- | ------------------------- |
| Free-form natural language | Role, enums, and JSON shape defined |
| Unpredictable structure | Intended machine-readable fields |

**Trainer note:** Prompting for JSON does **not** guarantee valid JSON or correct field values. Lab 04 adds examples and schema enforcement where supported.

---

## Verification checklist

- [ ] System instructions saved for this playground session  
- [ ] Response includes the five fields: `category`, `priority`, `sentiment`, `summary`, `requires_human_review`  
- [ ] Category and priority use allowed enum values  
- [ ] You kept **Version 1** instructions—you will extend them in Labs 04–08  

---

## Expected result

The assistant returns JSON-shaped triage output aligned with billing for the duplicate-payment scenario.

---

## Common issues

| Symptom | Action |
| -------- | ------ |
| Model adds markdown fences (\`\`\`json) | Note for Lab 04 structured output; retry with “JSON only, no markdown” |
| Extra prose outside JSON | Reinforce rule 7; use fresh conversation |
| Wrong category | Expected until few-shot and schema in Lab 04 |

---

## Lab 03 complete

Keep these system instructions. Continue to **[Lab 04](./Lab-04-few-shot-structured-output.md)**.
