# Lab 02 — Explore the model catalog and deploy a chat model

| | |
| --- | --- |
| **Duration** | 15 minutes |
| **Previous lab** | [Lab 01](./Lab-01-create-foundry-project.md) |
| **Next lab** | [Lab 03 — System instructions](./Lab-03-system-instructions.md) |

---

## Objective

Select a suitable **chat** model, deploy it as `support-chat-model`, and confirm inference works in the playground.

---

## Before you begin

- [ ] Lab 01 complete — you can open your Foundry project  
- [ ] You know your **region** and have **deployment quota** (or an instructor-provided deployment)  

---

## Step 1 — Open the model catalog

1. In the project, go to **Models**, **Model catalog**, or **Deployments → Deploy model**.  
2. Filter or search for a **chat-capable** model available in your subscription and region.  
3. Prefer a **low-cost** model approved for training (e.g. **GPT-4o mini** or equivalent where available).

---

## Step 2 — Review model details

Inspect before deploying:

| Property | What to check |
| -------- | -------------- |
| Capability | Chat / text generation |
| Context window | Enough for system prompt + examples + user message |
| Deployment availability | Deployable in your environment |
| Region | Offered in your selected region |
| Pricing | Token-based cost acceptable for workshop |
| Quota | Capacity available for a small deployment |
| Structured output | Supported modes (needed in Lab 04) |

**Trainer note:** Catalog visibility does not guarantee deployability in every region or SKU.

---

## Step 3 — Deploy the model

If no deployment exists yet, deploy with:

```text
Deployment name: support-chat-model
Model:            <your selected chat model>
Deployment type:  Supported option for your region (e.g. Global Standard)
Capacity:         Small training allocation (per admin guidance)
```

Review settings and start deployment. Wait until status is **Succeeded**.

If the instructor pre-provisioned the deployment, open **`support-chat-model`** and review its configuration instead of creating a second deployment.

---

## Step 4 — Open the playground and smoke test

1. Select deployment **`support-chat-model`**.  
2. Open **Chat playground** / **Playground**.  
3. Submit this user message:

```text
Explain Microsoft Foundry in two sentences.
Use simple language suitable for a beginner.
```

---

## Step 5 — Inspect the response

Identify where the UI shows (if available):

- Deployment / model name  
- Token usage  
- Latency or timing  
- Temperature and other parameters  

**Discussion:** A successful reply proves **inference**, not yet reliable **business triage**. That starts in Lab 03.

---

## Verification checklist

- [ ] Deployment `support-chat-model` exists and is healthy  
- [ ] Playground returns a coherent two-sentence answer  
- [ ] You noted whether **structured output** is listed for this model (for Lab 04)  

---

## Expected result

The model returns a short, beginner-friendly explanation of Microsoft Foundry (wording may vary between runs).

---

## Troubleshooting

| Symptom | Action |
| -------- | ------ |
| Deployment unavailable in region | Pick another supported model or region per admin |
| Quota exceeded | Lower capacity, use shared deployment, or request quota |
| HTTP 429 (throttling) | Reduce concurrent requests; retry with backoff |
| Unauthorized | Verify Entra sign-in, subscription, and RBAC on the resource |

---

## Lab 02 complete

Proceed to **[Lab 03 — System instructions](./Lab-03-system-instructions.md)**.
