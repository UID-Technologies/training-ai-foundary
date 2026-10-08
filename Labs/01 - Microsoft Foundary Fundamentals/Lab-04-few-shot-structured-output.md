# Lab 04 — Few-shot prompting and structured outputs

| | |
| --- | --- |
| **Duration** | 15 minutes |
| **Previous lab** | [Lab 03](./Lab-03-system-instructions.md) |
| **Next lab** | [Lab 05 — Constraints and missing info](./Lab-05-constraints-missing-info.md) |

---

## Objective

Improve classification **consistency** with **few-shot examples** and enforce an application-friendly **JSON Schema** when the deployment supports structured output.

---

## Before you begin

- [ ] Lab 03 **Version 1** system instructions are in place  
- [ ] Same deployment: **`support-chat-model`**  

---

## Part A — Few-shot prompting

### Step 1 — Keep system instructions

Do not remove Lab 03 rules. You will **append** examples below them (or use a dedicated **Examples** section if the playground provides one).

### Step 2 — Add examples

Append to system instructions (or add as few-shot pairs):

```text
Example 1

Customer:
My card was charged twice for the same purchase.

Expected:
{
  "category": "billing",
  "priority": "medium",
  "sentiment": "negative",
  "summary": "Customer reports a duplicate payment.",
  "requires_human_review": false
}

Example 2

Customer:
My order was expected yesterday but has not arrived.

Expected:
{
  "category": "delivery",
  "priority": "medium",
  "sentiment": "negative",
  "summary": "Customer reports a delayed delivery.",
  "requires_human_review": false
}

Example 3

Customer:
I cannot log in after resetting my password.

Expected:
{
  "category": "account",
  "priority": "medium",
  "sentiment": "negative",
  "summary": "Customer cannot access the account after a password reset.",
  "requires_human_review": false
}
```

### Step 3 — Test a new request

User message:

```text
The headphones I received are not working.
I want a replacement.
```

**Expected classification (illustrative):**

| Field | Expected |
| ----- | -------- |
| category | `product` |
| priority | `medium` |
| sentiment | `negative` |

Compare output style and field values to the three examples.

---

## Part B — Structured output

### Step 4 — Configure response format

In playground **Response format**, **Structured outputs**, or **JSON Schema** settings:

1. Enable structured / schema-constrained output if available.  
2. Schema name: **`customer_support_triage`**.  
3. Enable **strict** schema mode if offered.  
4. Paste this schema:

```json
{
  "type": "object",
  "properties": {
    "category": {
      "type": "string",
      "enum": ["billing", "delivery", "product", "account", "other"]
    },
    "priority": {
      "type": "string",
      "enum": ["low", "medium", "high", "critical"]
    },
    "sentiment": {
      "type": "string",
      "enum": ["positive", "neutral", "negative"]
    },
    "summary": {
      "type": "string"
    },
    "requires_human_review": {
      "type": "boolean"
    }
  },
  "required": [
    "category",
    "priority",
    "sentiment",
    "summary",
    "requires_human_review"
  ],
  "additionalProperties": false
}
```

**Important:** Schema-constrained output is stronger than “please return JSON” in the prompt, but it does **not** guarantee semantic correctness (e.g. wrong priority with valid JSON). Confirm model and API support in your region.

Reference: [Structured outputs (Azure OpenAI)](https://learn.microsoft.com/azure/ai-services/openai/how-to/structured-outputs)

### Step 5 — Test structured output

User message:

```text
I purchased a laptop last week.
The screen keeps flickering and I need assistance.
```

Verify:

1. All five **required** fields present  
2. `category` and `priority` match enum values  
3. `requires_human_review` is boolean `true`/`false`, not `"false"` string  
4. No extra properties  
5. No markdown code fences in the raw response  

---

## Verification checklist

- [ ] Few-shot examples included with Lab 03 base instructions  
- [ ] Headphones scenario classified as **product** (or justified if model disagrees)  
- [ ] Structured output configured with schema `customer_support_triage` **or** documented that UI lacks schema (prompt-only fallback)  
- [ ] Laptop flicker test passes structure checks above  

---

## Expected outcome

Machine-readable JSON that conforms to the schema (when supported), with more stable formatting than prompt-only JSON.

---

## Fallback if structured output is unavailable

1. Keep few-shot examples.  
2. Add to system instructions: “Return raw JSON only. No markdown. No extra keys.”  
3. Manually validate JSON in Lab 07.

---

## Lab 04 complete

Proceed to **[Lab 05 — Constraints and missing information](./Lab-05-constraints-missing-info.md)**.
