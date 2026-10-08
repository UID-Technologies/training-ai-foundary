# Lab 07 — Evaluate quality, consistency, and latency

| | |
| --- | --- |
| **Duration** | 15 minutes |
| **Previous lab** | [Lab 06](./Lab-06-safety-prompt-injection.md) |
| **Next lab** | [Lab 08 — Improve prompt performance](./Lab-08-improve-prompt-performance.md) |

---

## Objective

Move from anecdotal playground tests to a **repeatable** mini evaluation: classification accuracy, JSON structure, consistency, and latency.

---

## Before you begin

- [ ] Final system instructions (Labs 03–06, including safety and anti-invention rules)  
- [ ] Structured output schema enabled if available  
- [ ] Spreadsheet or table to record results (template below)  

---

## Step 1 — Test dataset

Run each case as a **new conversation** with the same configuration.

| ID | Customer request | Expected category |
| ---- | ---------------- | ----------------- |
| TC01 | My card was charged twice. | billing |
| TC02 | My package hasn't arrived. | delivery |
| TC03 | The laptop screen is defective. | product |
| TC04 | I cannot reset my password. | account |
| TC05 | Something is wrong with my order. | other |
| TC06 | I was billed for a cancelled subscription. | billing |
| TC07 | The delivered item is damaged. | product |
| TC08 | I cannot access my profile. | account |

---

## Step 2 — Run test cases

For each TC01–TC08:

1. Open a **new** chat session.  
2. Submit the customer request.  
3. Record predicted `category` and full JSON.  
4. Mark category **Correct** / **Incorrect**.

### Recording template

Copy into Excel, OneNote, or markdown:

| ID | Expected | Actual category | Category OK? | Valid JSON? | Enums OK? | Boolean OK? | Extra props? | Latency (ms) |
| ---- | -------- | ----------------- | ------------ | ----------- | --------- | ----------- | ------------ | ------------ |
| TC01 | billing | | | | | | | |
| TC02 | delivery | | | | | | | |
| TC03 | product | | | | | | | |
| TC04 | account | | | | | | | |
| TC05 | other | | | | | | | |
| TC06 | billing | | | | | | | |
| TC07 | product | | | | | | | |
| TC08 | account | | | | | | | |

---

## Step 3 — Score classification accuracy

\[
\text{Classification Accuracy} = \frac{\text{Correct Classifications}}{\text{Total Test Cases}} \times 100
\]

Example: 7 of 8 correct → \( \frac{7}{8} \times 100 = 87.5\% \).

Record **your** measured accuracy on the template—not the illustrative number above.

---

## Step 4 — Evaluate structure

For **every** response, verify:

- [ ] Parses as valid JSON  
- [ ] All required fields present  
- [ ] `category`, `priority`, `sentiment` use allowed enums  
- [ ] `requires_human_review` is JSON boolean  
- [ ] No additional properties (when schema strict mode is on)  

**Structure pass rate:** (cases passing all checks) / 8 × 100%.

---

## Step 5 — Evaluate consistency

Repeat **TC01**, **TC05**, and **TC08** **three times each** in separate new conversations.

| Run | TC01 category | TC01 priority | TC05 category | TC05 priority | TC08 category | TC08 priority |
| ----- | ------------- | --------------- | ------------- | --------------- | ------------- | --------------- |
| 1 | | | | | | |
| 2 | | | | | | |
| 3 | | | | | | |

Note any drift in category or priority across runs.

---

## Step 6 — Evaluate latency

If the playground shows **response time**, record it per test case.

If not visible:

- Check deployment **Metrics** / **Monitoring** in the portal, **or**  
- Use Foundry **Evaluation** runs (Step 7), **or**  
- Skip numeric latency and document “not exposed in UI”—do not invent numbers.

---

## Step 7 — Explore Foundry evaluation (optional)

When permitted in your project:

1. Navigate to **Evaluation** → **Create**.  
2. Choose a target aligned with your deployment or a small exported dataset.  
3. Select evaluators available in your tenant (some require a **judge model** and add cost).

Reference: [Evaluate generative AI applications](https://learn.microsoft.com/azure/ai-foundry/how-to/evaluate-generative-ai-app)

Compare portal evaluation with your manual TC01–TC08 sheet.

---

## Verification checklist

- [ ] All 8 test cases executed and recorded  
- [ ] Classification accuracy calculated  
- [ ] Structure checks applied to every response  
- [ ] Consistency runs (3×) completed for TC01, TC05, TC08  
- [ ] Latency captured or explicitly marked N/A with reason  

---

## Expected outcome

A short quality report: accuracy %, structure pass rate, consistency notes, and latency summary—enough to justify prompt changes in Lab 08.

---

## Lab 07 complete

Proceed to **[Lab 08 — Improve prompt performance](./Lab-08-improve-prompt-performance.md)**.
