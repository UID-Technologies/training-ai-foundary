# Microsoft Foundry Fundamentals — Hands-On Workshop

**Audience:** Beginner to intermediate  
**Total duration:** ~105 minutes (guided discussion included)  
**Delivery mode:** Instructor-led or self-paced  

This workshop expands the Microsoft Foundry presentation (slides 31–40) into eight connected labs. Participants use **one project** and **one chat deployment** for the entire session.

---

## Workshop scenario: AI Customer Support Triage Assistant

An e-commerce company receives hundreds of support requests daily. Teams manually read each message, classify the issue, set priority, and route to the right department.

**Your goal:** Build an AI-assisted triage flow in Microsoft Foundry that:

- Classifies customer messages into business categories  
- Returns structured JSON for downstream systems  
- Respects safety and privacy rules  
- Can be evaluated and improved without changing the model  

**Recommended naming (use consistently across labs):**

| Setting | Sample value |
| -------- | ------------- |
| Resource group | `rg-foundry-training-dev` |
| Foundry resource | `foundry-training-dev` |
| Project | `foundry-support-lab` |
| Deployment name | `support-chat-model` |
| Structured output schema name | `customer_support_triage` |

---

## Lab plan

| Lab | Topic | Duration | File |
| --- | ----- | -------- | ---- |
| 01 | Create and configure a Foundry project | 15 min | [Lab-01-create-foundry-project.md](./Lab-01-create-foundry-project.md) |
| 02 | Explore catalog and deploy a chat model | 15 min | [Lab-02-deploy-chat-model.md](./Lab-02-deploy-chat-model.md) |
| 03 | Configure system instructions | 10 min | [Lab-03-system-instructions.md](./Lab-03-system-instructions.md) |
| 04 | Few-shot prompting and structured outputs | 15 min | [Lab-04-few-shot-structured-output.md](./Lab-04-few-shot-structured-output.md) |
| 05 | Test constraints and missing information | 10 min | [Lab-05-constraints-missing-info.md](./Lab-05-constraints-missing-info.md) |
| 06 | Test safety controls and prompt injection | 15 min | [Lab-06-safety-prompt-injection.md](./Lab-06-safety-prompt-injection.md) |
| 07 | Evaluate quality, consistency, and latency | 15 min | [Lab-07-evaluate-quality.md](./Lab-07-evaluate-quality.md) |
| 08 | Improve prompt performance without changing models | 10 min | [Lab-08-improve-prompt-performance.md](./Lab-08-improve-prompt-performance.md) |

**Shorter sessions:** Demonstrate Labs 06–08 while participants complete Labs 01–05.

---

## Prerequisites

Before starting Lab 01, confirm:

- [ ] Azure subscription access (training or personal, as approved)  
- [ ] Permission to use an existing Foundry resource **or** create a new one  
- [ ] Approved **region** with model availability for your tenant  
- [ ] Sufficient **deployment quota** for a small chat model  
- [ ] Modern browser and access to [Microsoft Foundry portal](https://ai.azure.com/)  
- [ ] Microsoft Entra ID account for sign-in  

**Corporate workshops:** An administrator should pre-provision the Foundry resource, quota, and (optionally) the project so lab time is not spent on access errors.

---

## End-to-end flow

```text
Lab 01  Project + navigation
   ↓
Lab 02  Deploy support-chat-model + playground smoke test
   ↓
Lab 03  System instructions (triage role + JSON rules)
   ↓
Lab 04  Few-shot examples + JSON Schema structured output
   ↓
Lab 05  Ambiguous inputs + anti-hallucination rules
   ↓
Lab 06  Safety, privacy, prompt injection
   ↓
Lab 07  Test dataset + accuracy / structure / consistency
   ↓
Lab 08  Priority definitions + re-test
```

---

## Workshop completion checklist

Use this at the end of Lab 08 (or as an instructor sign-off):

- [ ] Created or accessed a Foundry project  
- [ ] Deployed or selected an available chat model (`support-chat-model`)  
- [ ] Successfully tested the model playground  
- [ ] Configured system instructions for triage  
- [ ] Added few-shot examples  
- [ ] Tested JSON structured output (schema if supported)  
- [ ] Tested incomplete customer requests  
- [ ] Tested privacy and prompt-injection scenarios  
- [ ] Evaluated classification and JSON structure  
- [ ] Improved priority classification rules  

---

## Instructor preparation

1. Walk through Labs 01–08 once on the **exact** subscription, region, and model participants will use.  
2. Confirm playground supports **system instructions**, **few-shot** (or inline examples), and **structured output / JSON Schema** for your chosen model.  
3. Pre-create project and deployment if participant permissions are limited.  
4. Keep default **content safety** settings enabled; do not disable guardrails for demos.

### Official references

- [Create Foundry resources and deploy a model](https://learn.microsoft.com/azure/ai-foundry/how-to/create-projects)  
- [Microsoft Foundry playgrounds](https://learn.microsoft.com/azure/ai-foundry/how-to/playground)  
- [Evaluate generative AI applications](https://learn.microsoft.com/azure/ai-foundry/how-to/evaluate-generative-ai-app)  

---

## Trainer appendix

- [Cumulative system prompt](./appendix-cumulative-system-prompt.md) — full instruction stack after each lab (for Lab 07–08 and instructor demos)

## Source document

The original combined guide is preserved as [test.md](./test.md). Individual lab files are the **trainer-ready** versions with navigation, checklists, and cleaned copy-paste blocks.
