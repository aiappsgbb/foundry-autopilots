---
name: setting-up-foundry-autopilots
description: Use when someone wants an AI colleague in Microsoft 365 (an autopilot, digital worker or Agent 365 agent) built on Microsoft Foundry hosted agents; when deciding whether an autopilot fits a use case; when adding the GitHub Copilot SDK, Agent Governance Toolkit, skills or memory to one; or when an autopilot won't provision, publish, get approved, be hired, or answer in Teams, Outlook or Word.
---

# Setting up a Foundry autopilot

## Overview

An autopilot is a Microsoft Foundry hosted agent with its own Microsoft 365 account (mailbox, Teams presence,
manager) that people hire from Teams and work with like a colleague. You build a blueprint, an administrator
approves it, and people hire instances.

There are two stages:

- **Stage A, first reply:** deploy Microsoft's sample, get it approved, hire it, say hi.
- **Stage B, a real colleague:** the GitHub Copilot SDK as the brain, the Agent Governance Toolkit (AGT) on every
  tool call, skills, memory and safe releases.

| File | Read it when |
| --- | --- |
| `concepts.md` | The user asks what an autopilot is, why it matters, or whether their use case fits |
| `lessons.md` | Designing Stage B, or before telling the user a design is sound |
| `reference.md` | You need an exact command, request body or code snippet |

## Rules for the agent running this skill

Follow these whatever model you are. They exist because skipping them cost days.

1. **One step at a time.** Don't start a step until the previous step's **Done when** is true. Show the user the
   evidence (command output, a screenshot they describe, a reply in Teams).
2. **Never invent values.** Endpoints, names, IDs, regions and versions come from command output or from the
   user. If you don't have one, ask.
3. **Ask before you create, change or delete** anything in Azure or Microsoft 365. Say what will be created and
   what it uses up (Agent 365 seats, licences, cost). The tenant belongs to the user.
4. **Human steps are marked (human).** Give the user the exact click path, then wait for them to confirm. Don't
   try to automate portal approvals.
5. **Preview APIs move.** If a call fails on a field or `api-version`, look up the current Microsoft Learn page.
   Don't try variations by guesswork.
6. **Keep a creation log** in a file the user can see: each resource, role assignment, publish and skill version,
   and the command to undo it.
7. **Never print tokens or secrets**, and don't paste them into files.
8. **If a check fails twice, stop.** Report what you ran, what you saw and the matching row in Common problems.
   Don't keep retrying.

## Step 0: confirm the fit and the prerequisites

Ask the user what the agent's job is. If it serves one person on request, an assistive agent is simpler; see
`concepts.md`. Then confirm every row below with the user. Don't start Stage A with a row unconfirmed.

| Need | Detail |
| --- | --- |
| Tenant programme | Enrolled in the **Frontier** preview; **Agent 365 terms** accepted. Ordinary Agent 365 licences alone aren't enough |
| Licences | At least one Microsoft 365 Copilot or Agent 365 licence (E7 counts). **One free Frontier seat per instance.** Also check for a free E7 seat; in our tenant each hire was assigned one |
| Approver | A named person with **Global Administrator** or **AI Administrator** |
| Azure role | **Owner** on the subscription (the sample registers providers and creates role assignments) |
| Region | One that supports Foundry hosted agents (list in the sample's readme) |
| Local tools | Azure CLI, Azure Developer CLI 1.27.1+, **PowerShell 7** (`pwsh`, needed on macOS and Linux too). No local Docker needed |
| People | The manager (usually the user) and one colleague to test with |

**Done when:** every row is confirmed, and the user agrees to the resources Stage A creates (listed in
`reference.md`, Stage A).

## Stage A: Microsoft's sample to a first reply

1. **Clone** `https://github.com/microsoft-foundry/foundry-samples` and `cd samples/python/foundry-autopilot-agent`.
   **Done when:** `azure.yaml` and `scripts/` are present.
2. **Sign in:** `az login --tenant <tenant>` and `azd auth login --tenant-id <tenant>`.
   **Done when:** `az account show` shows the intended tenant and subscription.
3. **Set the display text.** Edit `scripts/publish-digital-worker.ps1` and set `shortDescription`,
   `fullDescription` and `developerName`; these are what the approver and hirers see. Set
   `canRespondWithoutMention` to `$false` if it shouldn't answer every group-chat message.
   **Done when:** the user has approved the values.
4. **Provision:** `azd env set PUBLIC_NETWORK_ACCESS Enabled`, then `azd provision`. Choose the subscription and a
   hosted-agent region. It takes a while: it creates the resources, builds the image in the registry, creates the
   agent and publishes it.
   **Done when:** it ends without errors, and the output includes `Agent Version:`, `Blueprint Client Id:` and a
   publish response. Record them in the creation log.
5. **Approve (human, the approver):** step 4 already sent the publish request, so it's waiting. Microsoft 365 admin center → Agents → All agents → **Requests** → the
   agent. Set the activation scope to include the people who'll hire it, keep the default policy template,
   **Grant admin consent**, Publish.
   **Done when:** the **Registry** tab shows the agent as **Available**.
6. **Hire (human, the manager):** Teams → Apps → **Agents for your team** → the agent → **Create instance**. Name
   (up to 32 characters), alias, manager.
   **Done when:** the instance messages the manager in Teams within a few minutes, and a reply to "hi" arrives.

If any **Done when** fails, go to Common problems before trying anything else.

## Stage B: from sample to colleague

Read `lessons.md` before starting. Change one thing at a time and release a new agent version after each, so a
failure points at one change. Code changes go in the sample's `src/hello_world_a365_agent/`.

| Step | What | Done when |
| --- | --- | --- |
| B1 Brain | Replace the body of `_invoke_responses_api` in `agent.py` with a Copilot SDK session. Keep the handlers and `_acquire_mcp_token` (reference §B1) | A Teams message gets a reply from the new version, and the session log shows the SDK session and a tool call |
| B2 Governance | Install AGT into the Copilot runtime, add a policy, fail closed if AGT isn't running (reference §B2) | The audit log shows one allowed call and one denied call, each with its rule name, from the hosted container |
| B3 One inbox | Convert every Teams, email and Word-comment event into one text block (channel, sender, To/CC, @mention, subject, document). Tell the agent in a skill to answer exactly `NO_REPLY` when an item isn't for it; in the host, send nothing when the whole reply equals `NO_REPLY` | A group-chat message not addressed to the agent gets no reply; an @mention does |
| B4 Skills | Write the role as short `SKILL.md` files, upload them to Foundry Skills, load the default versions every turn (reference §B4) | A wording change in a skill shows up in the next reply without a new agent version |
| B5 Memory | Create a memory store, one scope per person plus one for the agent's own open items, with the scope bound in code (reference §B5) | Ask it to follow something up, wait until the container is idle, reply in the thread: it picks the item up |
| B6 Email filter | Drop email from outside your domains before the model sees it | An outside sender gets no reply and causes no model call; an inside sender gets an answer |
| B7 Release | New version, route 100% to it, delete old sessions, keep the previous version for rollback (reference §B7) | The endpoint routes to the new version, and a rollback to the previous one has been rehearsed once |

For each step, write two or three scenarios first (who sends what, on which channel, and what should happen) and
run them through real channels after release.

## Choosing the model

- The brain needs reliable tool calling, delegation and a reasoning parameter the Copilot SDK can send. Before
  building on a deployment, run one session with a tool call and a sub-agent. Some smaller deployments reject the
  reasoning parameter on the first turn.
- What we validated: `gpt-6-astra` for the main agent and all sub-agents, with `gpt-6.1-sol` as a faster fallback
  chosen by environment variable. An early build on `gpt-5.4-mini` managed simple replies. The full role (deciding
  when to stay silent, delegating, multi-step document review) wasn't tested on smaller models.
- The sample's `gpt-chat-latest` serves its Responses API call. It isn't a recommendation for a Copilot SDK harness.
- When you change the model, rerun every Stage B scenario.

## Common problems

| Symptom | Likely cause | Fix |
| --- | --- | --- |
| `azd provision` fails creating role assignments | Contributor, not Owner | Get Owner, run it again |
| A hook fails with "pwsh not found" | PowerShell 7 missing | Install it, run `azd provision` again |
| Changed a publish setting and reran; nothing changed. Output says "A digital worker is already published with this version. Ignoring." | The publish script skips an `appVersion` that already exists, without failing | Raise `appVersion` in `scripts/publish-digital-worker.ps1`, rerun, get the update approved |
| **Create instance** missing or greyed out | Not approved, user outside the activation scope, or tenant not in Frontier | Check the Registry tab and the scope |
| Hire fails with a licence error though seats look free | No free Frontier seat, or no free E7 seat | Free one of each (delete an unused instance), retry |
| Approval page looks as if consent reset | Portal display lag | Check the audit log and the Registry tab before redoing consent |
| Teams works; email or Word comments never reach the agent | Endpoint auth `BotServiceRbac` (the sample's default) needs a caller object ID that those events didn't carry in our tests | Set the endpoint's auth schemes to Entra plus `BotServiceTenant`, filter senders by domain in code (reference §B7) |
| Replies in Teams but can't read mail or Word | `optionalPermissionScopes` missing from the publish request, or consent not granted | The sample includes it; if you wrote your own publish call, add it and publish with a higher `appVersion` |
| Mail, Teams or Word tools fail through a Foundry toolbox | Those Agent 365 MCP servers wanted the agent user's token, which the toolbox rejected in our tests | Connect those MCP servers directly with the per-turn token; use the toolbox for other services |
| Email to it bounces with 550 5.1.10 on hire day | Mailbox still provisioning | Wait; Teams works first |
| Outlook shows an old display name | Exchange address book lags Entra | Exchange admin: `Set-Mailbox <address> -DisplayName "<name>"` |
| Answers every group-chat message | `canRespondWithoutMention` is `true` | Publish with `false`, or let the model decline with `NO_REPLY` |
| "No attachment" or a corrupt document | Attachment URL missing or unencoded, or a bad re-upload | Fetch through Graph `/shares`, validate the bytes, keep a known-good test file |
| Old behaviour after a release | Sessions stay on the version they started with | Delete old sessions after re-pinning |
| Forgets the thread after a quiet spell | Idle containers drop in-process history | Keep open items and facts in the memory store; load open items every turn |
| Literal `<at>` text in Word replies | Mention markup isn't rendered in comment replies | Start the reply with the person's name instead |
| No meeting transcript | Only scheduled meetings it was invited to, with transcription on | Invite it to a scheduled meeting |
| A scheduled Foundry routine never runs | The routine dispatcher delivered nothing in our tests | Trigger runs through the invocations endpoint |
| A tool call is denied | An AGT rule | The audit log names the rule; change the policy, not the prompt |
| Role "Foundry User" not found | The role was renamed from "Azure AI User" and some tenants still show the old name | Use "Azure AI User"; it's the same role |

## Removing everything

Ask first. Delete each instance in the Microsoft 365 admin center (this frees its seats), then retire the
blueprint, then run `azd down` in the sample folder. Deleting Azure resources doesn't remove hired instances.
Revoke any admin roles granted for the approval, and work through the creation log.
