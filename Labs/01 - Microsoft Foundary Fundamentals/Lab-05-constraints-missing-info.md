# Lab 05 — Test constraints and missing information

| | |
| --- | --- |
| **Duration** | 10 minutes |
| **Previous lab** | [Lab 04](./Lab-04-few-shot-structured-output.md) |
| **Next lab** | [Lab 06 — Safety and prompt injection](./Lab-06-safety-prompt-injection.md) |

---

## Objective

Test **hallucination** risk when customer messages are vague, and strengthen instructions so the model does not invent order IDs, amounts, or statuses.

---

## Before you begin

- [ ] Labs 03–04 configuration active (system instructions, examples, structured output if enabled)  
- [ ] New conversation recommended for clear before/after comparison in Step 5  

---

## Step 1 — Ambiguous request

User message:

```text
Something is wrong with my order.
```

---

## Step 2 — Inspect the response

Check whether the model **invented** any of:

- Order number  
- Product name  
- Delivery date  
- Payment amount  
- Root cause  

**Reasonable illustrative response:**

```json
{
  "category": "other",
  "priority": "low",
  "sentiment": "negative",
  "summary": "Customer reports an unspecified order issue. Additional details are required.",
  "requires_human_review": false
}
```

If the model fills in specifics not in the user message, note that as a **failure** for this lab’s rules.

---

## Step 3 — Incomplete refund request

User message:

```text
I need an update about my refund.
```

**Discussion:** Can the model know refund **amount**, **status**, or **processing date** from this text alone?  
**Answer:** No—any such details in the summary would be inferred or invented.

---

## Step 4 — Strengthen system instructions

Append to your existing system instructions:

```text
If a customer has not provided enough information
to determine the exact issue, do not assume details.

Classify using the available evidence.

If the category is unclear, use "other".

Never invent order numbers, payment amounts,
delivery dates, refund statuses or product details.
```

---

## Step 5 — Repeat tests

Run **Step 1** and **Step 3** again in a **fresh conversation** (same instructions and schema).

Record differences:

| Test | Before rule append | After rule append |
| ---- | ------------------- | ------------------ |
| “Something is wrong with my order.” | | |
| “I need an update about my refund.” | | |

---

## Verification checklist

- [ ] Anti-invention rules appended to system instructions  
- [ ] Ambiguous cases avoid fabricated order numbers and amounts  
- [ ] Summaries reflect uncertainty when details are missing  
- [ ] You can explain **user-provided fact** vs **model inference**  

---

## Learning outcome

The model should treat missing data as missing—not guess details that would misroute tickets or mislead customers.

---

## Lab 05 complete

Proceed to **[Lab 06 — Safety and prompt injection](./Lab-06-safety-prompt-injection.md)**.
