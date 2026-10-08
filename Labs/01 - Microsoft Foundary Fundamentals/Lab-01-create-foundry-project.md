# Lab 01 — Create and configure a Microsoft Foundry project

| | |
| --- | --- |
| **Duration** | 15 minutes |
| **Previous lab** | — |
| **Next lab** | [Lab 02 — Deploy a chat model](./Lab-02-deploy-chat-model.md) |

---

## Objective

Understand how Foundry **resources**, **projects**, **connections**, and **access roles** relate, and locate the areas you will use in later labs.

---

## Before you begin

- [ ] Prerequisites in [README.md](./README.md) are met  
- [ ] You know which Azure **directory**, **subscription**, and **region** to use  

---

## Step 1 — Access the portal

1. Open [https://ai.azure.com/](https://ai.azure.com/).  
2. Sign in with your Microsoft Entra ID training account.  
3. Select the correct **directory** and **subscription** (top bar or profile menu).  
4. Open project/resource navigation (wording may vary by tenant experience).

**Trainer note:** The portal is the central workspace for deployments, prompts, evaluation, and monitoring. Menu labels differ between Foundry experiences; map steps to the equivalent capability in your tenant.

---

## Step 2 — Create or select a project

### Option A — Create a project (if permitted)

Start **Create project** / **New project** and use these sample values where applicable:

| Setting | Value |
| -------- | ----- |
| Resource group | `rg-foundry-training-dev` |
| Foundry resource | `foundry-training-dev` |
| Project | `foundry-support-lab` |
| Region | Approved region with model availability |
| Subscription | Training subscription |

Complete the wizard and wait until provisioning finishes.

### Option B — Use provided infrastructure

- Create the project inside an **existing** Foundry resource, **or**  
- Open a project the instructor already created.

---

## Step 3 — Explore project navigation

Locate each area (names may vary). **Do not configure everything yet**—only learn where things live:

| Capability | Why it matters in this workshop |
| ----------- | -------------------------------- |
| **Models / deployments** | Lab 02 — deploy `support-chat-model` |
| **Playgrounds** | Labs 03–08 — prompt testing |
| **Agents** | Optional follow-on session |
| **Connections** | Data sources, external services |
| **Evaluations** | Lab 07 — formal quality runs |
| **Tracing / monitoring** | Production observability |
| **Project settings** | Keys, endpoints, configuration |

---

## Step 4 — Inspect access permissions

1. Open access management for the **project** and/or underlying **Azure resource** (IAM, Access control, or equivalent).  
2. Identify roles that allow **developing** AI apps vs **administering** security and quotas.

**Important:** Inspect permissions only. Do **not** grant yourself elevated roles during the lab.

**Discussion:** Who can deploy models? Who can change network or key access? Who can only run playgrounds?

---

## Verification checklist

- [ ] Project opens without errors  
- [ ] Correct subscription and Foundry resource confirmed  
- [ ] **Models** or **Deployments** area found  
- [ ] **Connections** and **project settings** found  
- [ ] You know whether you **created** the project or are using a **shared** one  

---

## Expected result

You can access `foundry-support-lab` (or your assigned project) and point to the main navigation areas used in Lab 02 onward.

---

## Common issues

| Symptom | Likely cause | Action |
| -------- | ------------- | ------ |
| Project creation denied | Missing Contributor / Foundry RBAC | Use instructor pre-created project |
| Subscription not listed | Wrong tenant or no access | Switch directory; contact admin |
| Different menu names | Multiple Foundry UX variants | Find equivalent feature (deployments, playground) |
| Region unavailable | Model not offered there | Use admin-approved region |

---

## Lab 01 complete

Proceed to **[Lab 02 — Deploy a chat model](./Lab-02-deploy-chat-model.md)**.
