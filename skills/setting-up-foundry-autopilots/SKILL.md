---
name: setting-up-foundry-autopilots
description: Use when setting up, deploying or hiring an AI colleague (an autopilot) in Microsoft 365 with Microsoft Foundry hosted agents and Agent 365, adding the GitHub Copilot SDK or the Agent Governance Toolkit to one, or when an autopilot won't publish, can't be hired, or doesn't answer in Teams, Outlook or Word.
---

# Setting up a Foundry autopilot

## Overview

An **autopilot** is a Microsoft Foundry hosted agent that works in Microsoft 365 under its own identity: its own
Entra account, mailbox and Teams presence, and a manager. You build a *blueprint* (a hosted agent), publish it,
an administrator approves it, and people *hire* instances of it from Teams.

There are two stages:

- **Stage A, first reply.** Microsoft's sample, `microsoft-foundry/foundry-samples`, folder
  `samples/python/foundry-autopilot-agent`, provisions everything with one command. You approve, hire and say hi.
  Budget an hour plus the admin's approval.
- **Stage B, a real colleague.** Swap the sample's brain for the GitHub Copilot SDK, put the Agent Governance
  Toolkit (AGT) in front of every tool call, and add versioned skills and a memory store. These are the parts that
  make it decide what needs it, work for several minutes, follow up, and stay inside rules you can read.

Exact commands, request bodies and code are in `reference.md` next to this file.

## Before you start

Missing any of these stops you at the hire, often after everything else has worked.

| Need | Detail |
| --- | --- |
| Tenant programme | Enrolled in the **Frontier** preview; **Agent 365 terms** accepted |
| Licences | At least one Microsoft 365 Copilot or Agent 365 licence (E7 counts). **One free Agent 365 Frontier seat per instance** you hire (eligible tenants get 25). We also saw each hire take an E7 seat; check both |
| Approver | Someone with **Global Administrator** or **AI Administrator** |
| Your Azure role | **Owner** on the subscription. The sample registers resource providers and creates role assignments, which Contributor can't do |
| Region | One with Foundry hosted agents, for example East US 2, Sweden Central, UK South (full list in the sample's readme) |
| Local tools | Azure CLI, Azure Developer CLI 1.27.1 or later, **PowerShell 7** (`pwsh`: the sample's scripts need it on macOS and Linux too). No local Docker: the image builds in your registry |
| People | You (you'll be the manager) and one colleague to message it |

## Stage A: Microsoft's sample to a first reply

1. **Clone** `microsoft-foundry/foundry-samples` and open `samples/python/foundry-autopilot-agent`.
2. **Sign in** with both CLIs to the tenant (`az login --tenant`, `azd auth login --tenant-id`).
3. **Provision:** `azd env set PUBLIC_NETWORK_ACCESS Enabled`, then `azd provision`. Pick the subscription and a
   hosted-agent region when asked. This creates the Foundry project, a model, a registry and Application Insights;
   builds the image; creates the agent (which creates its blueprint and identity); and publishes it. Check: it ends
   without errors, and the output includes `Agent Version:` and `Blueprint Client Id:`.
4. **Approve (admin):** Microsoft 365 admin center → Agents → All agents → **Requests** → your agent. Choose who can
   hire (the activation scope), keep the default policy template, **Grant admin consent**, then Publish. Check: the
   **Registry** tab shows it as **Available**.
5. **Hire:** Teams → Apps → **Agents for your team** → your agent → **Create instance**. Give it a name (up to 32
   characters), alias and manager. Check: within a few minutes it opens a Teams chat with you. Say hi.

## Stage B: from sample to colleague

Do these one at a time, releasing a new agent version after each so you can tell what broke.

1. **Brain:** replace the sample's model call with a GitHub Copilot SDK session. You get planning, tools, skills,
   sub-agents and context compaction from the engine behind GitHub Copilot CLI (reference §B1).
2. **Governance:** install AGT into the Copilot runtime and write a policy (reference §B2). Every tool call,
   including sub-agents' calls, is allowed or denied by a named rule before it runs.
3. **One inbox:** turn every channel event (a Teams message, an email, a Word comment) into the same text block with
   sender, To/CC, @mention and any document. Let the model reply with an agreed token, such as `NO_REPLY`, when the
   item isn't for it, and post exactly one reply in the same place.
4. **Skills:** write the job description and working rules as short `SKILL.md` files. Upload them to Foundry Skills
   and load the default versions on every turn. A behaviour fix then takes effect from the next message, with no
   rebuild (reference §B3).
5. **Memory:** create a Foundry memory store with one scope per person and one for the agent's own open items. Load
   the open items into every turn, so a reply that arrives later is picked up by a fresh session (reference §B4).
6. **Email filter:** anyone can email a mailbox. Drop email from outside your domains before the model sees it.
7. **Release:** each change is a new agent version. Route 100% of traffic to it, and route back to roll back
   (reference §B5).

## Common problems

| Symptom | Cause | Fix |
| --- | --- | --- |
| **Create instance** missing or greyed out | Not approved, you're not in the activation scope, or the tenant isn't in Frontier | Check the Registry tab and the scope |
| Hiring fails with a licence error, though seats look free | No free Frontier seat, or (in our tenant) no free E7 seat | Free one of each, retry |
| `azd provision` fails while creating role assignments | You're Contributor, not Owner | Get Owner, run it again |
| A hook fails with "pwsh not found" | PowerShell 7 isn't installed | Install it, run `azd provision` again |
| Replies in Teams, but can't read mail or Word | Publish request lacked `optionalPermissionScopes`, or consent wasn't granted in the approval wizard | The sample includes it; if you wrote your own publish call, add it and publish with a higher `appVersion` |
| Email to it bounces with 550 5.1.10 on the day of the hire | The mailbox is still provisioning | Wait; Teams works first |
| Outlook shows an old display name | Exchange's address book lags Entra | Exchange admin: `Set-Mailbox <address> -DisplayName "<name>"` |
| Answers every group-chat message | `canRespondWithoutMention` is `true` (the sample's default) | Publish with `false`, or let the model decline items not addressed to it |
| No meeting transcript | Transcripts exist only for meetings it was invited to, not Meet now | Invite it to a scheduled meeting with transcription on |
| Forgets the conversation after a quiet spell | The container's conversation history doesn't survive going idle | Keep durable facts and open items in the memory store |
| A scheduled Foundry routine never runs | In our tests the routine dispatcher delivered nothing | Trigger the run yourself through the invocations endpoint |
| Changed a publish setting, reran `azd provision`, nothing changed | The sample's publish script skips a publish whose `appVersion` already exists, without failing | Raise `appVersion` in `scripts/publish-digital-worker.ps1`, rerun, get the update approved |
| A tool call is denied | An AGT rule said no | The session log names the rule (reference §B5) |
| Role "Foundry User" not found | The rename is still rolling out | Use "Azure AI User" (same role) |
