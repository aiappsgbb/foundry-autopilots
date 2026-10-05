# Autopilots: what they are and when to build one

## What an autopilot is

Most agents today work for whoever is talking to them. You open a chat, ask for something, and the agent acts
with your permissions. That model is fine for a personal assistant. It falls apart once the agent is part of a
team.

An autopilot is a Foundry hosted agent with its own place in the organisation. Each hired instance gets two
Entra objects:

| Object | What it is | What it gives the autopilot |
| --- | --- | --- |
| Agent identity | A service principal | Credentials for the infrastructure it runs on: the model, Foundry, your registry |
| Agent user account | A user object | A display name, mailbox, calendar, OneDrive, Teams presence, a place in the org chart, and a manager |

The second object is the one that matters. With it, the autopilot sends mail, edits a document or posts in a
group chat as itself. Nobody has to be signed in, and nobody else's permissions leak into the thread.

Microsoft's docs put it in one line: capabilities describe an autopilot, identity establishes it. Memory,
planning and proactivity are features. Having a user account and a manager is what makes it a colleague.

## Blueprints, instances and who does what

You never build an autopilot directly. You build a blueprint (a hosted agent published with
`publishAsAutopilot: true`), and people hire instances of it. One blueprint can become many instances, each with
its own identity, manager and access. Update the blueprint and every instance gets the new version. Block it and
they all stop.

Four roles are involved, and the first attempt usually stalls on the hand-offs between them:

| Stage | Who | Produces |
| --- | --- | --- |
| Provision | Azure administrator (Owner on the subscription) | Foundry project, model, registry, monitoring |
| Build and publish | Developer | A hosted agent version and a publish request with declared Microsoft 365 scopes |
| Approve and consent | Microsoft 365 administrator (Global or AI Administrator) | The blueprint in the Agent 365 registry, with admin consent for its scopes |
| Hire and manage | Any person in the activation scope | An instance with its own mailbox and Teams presence. The person who hires it becomes its manager |

## Is an autopilot the right shape?

Foundry can build three kinds of agent. Pick the smallest one that does the job.

| Kind | Acts as | Use it when |
| --- | --- | --- |
| Assistive | The signed-in user | One person asks, the agent helps that person. Meeting prep, drafting, search |
| Background service | Itself, app-only, no Microsoft 365 actions | Event-driven automation outside Microsoft 365, such as restarting a VM on an alert |
| Autopilot | Itself, with its own Microsoft 365 account | The agent holds a role in a team: it's in group chats, gets email, edits shared documents, follows up later |

Build an autopilot when at least one of these is true:

- The work happens in group settings, where acting "on behalf of" one member is wrong.
- The agent has to act when nobody is in the loop, such as a reply arriving overnight or a reminder falling due.
- People should be able to address it like a colleague: @mention it, CC it, assign it a comment in Word.
- Someone has to be accountable for it. An autopilot has a manager, an audit trail and an off switch.

If none of these apply, an assistive agent is simpler and you avoid the licensing and approval overhead.

## Use cases that work well

The best fit is a role with a clear remit, written rules, and a lot of small requests from many people. Most of
those requests are routine. A few have to go to a human.

We built two to prove the pattern. Both run from the same code, with different skills and policies:

- **A compliance officer.** People @mention it in Teams group chats, CC it on email, and assign it Word comments.
  It reviews a policy draft against the house rules and returns tracked changes with a comment per issue. For a
  three-question email it sends each question to its own sub-agent, asks HR for a missing fact, and replies once.
  When HR answers later, it picks the item up again and tells its manager on Teams. It stays quiet in group chat
  unless someone is talking to it. It sends email only inside the company, enforced by a governance rule and not
  just by the prompt.
- **A procurement officer.** It shortlists approved suppliers for a request in Teams, drafts an RFP in Word and
  works through comment threads on it. When an urgent "change our supplier's bank details" email arrives, it
  refuses, tells its manager why, and keeps the item open. Separately, the governance policy denies the write,
  whatever the model decides.

Other roles with the same shape: a release manager for a delivery team, a contracts desk, a security
questionnaire responder, an onboarding coordinator, a sales-desk analyst who answers pricing questions from a
price book.

## Why this is a big deal

Three things change when an agent has an identity and a manager.

**It works where the work already happens.** Nobody opens a new app. People email, chat and comment as they
always have, and the agent is one more participant. That's most of the adoption problem solved.

**It can be governed like a person, and more tightly.** Admins approve the blueprint and grant its scopes. Each
instance has an owner. Every tool call can go through a policy engine (the Agent Governance Toolkit) that allows
or denies it by a named rule before it runs. A person's judgement can't be audited that precisely.

**It holds work over time.** With a memory store and open items, the agent picks a thread up hours later, when
the person it was waiting for replies. A chat assistant can't do that, because its work ends with the chat.

What it isn't, yet: a meeting participant (it works from transcripts afterwards) or a replacement for formal
approvals. Keep sign-offs with a named human and design the agent to route to them.

## Further reading

- [What is an autopilot in Microsoft Foundry?](https://learn.microsoft.com/azure/foundry/agents/concepts/autopilot-overview)
- [Autopilot lifecycle](https://learn.microsoft.com/azure/foundry/agents/concepts/autopilot-lifecycle)
- [Quickstart: build your first autopilot](https://learn.microsoft.com/azure/foundry/agents/how-to/agent-365)
