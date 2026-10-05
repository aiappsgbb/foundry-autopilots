# Lessons for harness builders

Anyone building an autopilot harness will run into these. Most aren't in the docs yet. Each one says what to do
and why. Where a point comes from our own testing rather than Microsoft's documentation, it says so. Preview
services change, so check these against your tenant.

## Getting hired

Treat the hire as its own project, separate from the deployment. A clean `azd provision` says nothing about
whether anyone can hire the agent. Hiring is asynchronous Microsoft 365 provisioning, and it depends on licences,
approval and scope.

Check the licences before you design anything. Ordinary Agent 365 licences don't make a tenant eligible for
autopilots; the tenant has to be in the Frontier preview. Each instance takes a Frontier seat. In our tenant each
instance was also assigned an E7 licence, so a hire failed with a generic licence error while the portal showed
free Agent 365 seats. Freeing an E7 seat fixed it.

Don't trust the admin portal's buttons. During approval the portal sometimes looked as if consent had reset,
while the audit log showed it had been granted. Check the Registry tab and the audit log, not the state of a
button.

Name the people before you start: who approves (Global or AI Administrator), who hires, and who manages the
instance. Elevated roles granted for the approval should be removed afterwards. Write that down when you grant
them.

Expect the identity to settle unevenly. On hire day the mailbox bounced mail (550 5.1.10) for a while. Exchange
kept the old display name after a rename until it was set with `Set-Mailbox`. A local Teams client cached the old
name even after Entra and Teams had the new one. Check the server side before you rebuild anything.

Clean up in the right order. Azure resources and hired instances live in different control planes. Deleting the
Foundry project leaves the agent user, its mailbox and its seats in place. Delete instances in the Microsoft 365
admin center first, then the Azure side.

## The harness

Start from the platform sample, then change the brain. Microsoft's sample is the fastest way to get provisioning,
identity and publishing right. Its brain is a single Responses API call. Keep the sample's activity handling and
per-turn token exchange, and swap only the model call.

Use the full agent runtime, not a cut-down one. Our first harness turned off most of the Copilot runtime and
exposed a few custom tools. It answered questions but couldn't do a job. Enabling skills, MCP servers, sub-agents
and context compaction is what turned it into something that plans, delegates and finishes work.

Make every channel look the same to the model. Teams messages, emails and Word comments differ in a dozen small
ways. Convert each one into the same text block: channel, sender, To and CC, whether the agent was @mentioned,
subject, and any document. The skills then deal with the work, not with channel quirks.

Give the model a real way to say nothing. In a group chat or on a CC, most events aren't for the agent. Agree an
exact token (we used `NO_REPLY`) that means "send nothing". An empty answer looks like a failure. Make "is this
for me?" a skill of its own, and have the agent save anything useful to memory even when it stays silent.

Send independent asks to sub-agents. An email with three questions works best as three parallel sub-agents whose
results come back as one reply. One long sequential turn is slower and drops questions.

Keep the rules in skills, not in the image. Short `SKILL.md` files (job description, working rules, when to
escalate) loaded from Foundry Skills on each turn let you change behaviour without a release. Keep each one short
and imperative, and test it. In our scenario suite the same model passed 1 of 19 role scenarios without the role
skills and 19 of 19 with them.

Don't build a case-management system first. An early version grew into a framework of cases, states and routers
before it could answer a question. The colleague that worked had two or three skills, a few MCP connections and a
small list of open items.

## Channels

Fetch documents through Microsoft Graph, not the attachment URL. Teams attachments sometimes arrive without a
download URL, with unencoded spaces, or as corrupt re-uploads. Resolve the sharing link through Graph `/shares`,
then check that the bytes are a valid document before handing it to the model.

Pass images as images. If the host forwards only extracted text, the model reports that nothing was attached.
Pass the image data itself. Vision helps, but it isn't exact OCR, so don't promise exact figures from a
screenshot.

Parse recipients in both formats. The Mail MCP server returned recipients as plain addresses in some responses
and as structured objects in others. A parser that handled only one format quietly treated CC'd mail as addressed
to someone else.

Word comment threads: replies work, but resolving a comment doesn't. In replies, `<at>` mention markup showed up
as literal text, so start the reply with the person's name instead.

Meetings: the agent works from transcripts and doesn't join calls. Transcripts were available only for
scheduled meetings the agent was invited to, with transcription on. Ad-hoc "Meet now" calls produced nothing it
could read.

## State and memory

Assume the container forgets. A hosted container that goes idle loses its in-process conversation history, and
two containers with the same SDK session ID don't share it. Put anything that has to survive in the Foundry
memory store: facts about people, and the agent's own open items. Load the open items into every turn.

Scope memory by person, and bind the scope in code. Use one scope per person (keyed on their Entra object ID)
and one for the agent's own work. Build the remember and recall tools for each conversation, bound to the
authenticated sender, so the model never chooses a scope. Load personal memories only in one-to-one chats.

Some channels don't tell you who sent the message. Email and Word events often carry only a UPN, not an object
ID. When you can't map a sender to a memory scope, give the agent its open items and leave personal memory out.

Make effects idempotent. Foundry's durable execution restarts a handler from the beginning rather than resuming
it. A send that happened just before a restart will happen again unless you check first. We saw one duplicate
Teams follow-up and couldn't reproduce it, so log enough to tell.

## Governance

Use one layer for action rules. We first had allow-lists in Python as well as AGT policy, and the two disagreed.
Put every allow and deny for tools, recipients and writes in the AGT policy. Code only handles platform
boundaries, such as the tenant check and the email domain filter.

Shape tools so the policy can see them. AGT matches top-level string arguments. Recipients passed as an array are
invisible to it, so a "company addresses only" rule can't be enforced on the stock send tools. Deny those tools
and give the agent a `send_email` with a single `to`.

Deny high-risk writes in policy, not only in the prompt. The procurement agent's skill says never to change bank
details, and the policy denies the write tool as well. If the model gets talked round, the call still fails and
the log names the rule.

Fail closed, and prove it in the container. Policy tests passing locally don't show that the extension loaded in
the hosted runtime. Deny every tool call unless AGT reports `running`. After release, check the audit log for one
allowed call and one denied call.

Know what each mode does. In `enforce` mode AGT's heuristic text scanner blocked ordinary compliance language. In
`advisory` mode the deterministic allow and deny rules still apply, and only the scanners report instead of block.
Write down which mode you run and why.

Sub-agents are governed too. AGT checks tool calls from sub-agents, so delegation isn't a way around the policy.

## Models

Test the exact model and provider combination. The Copilot SDK sends a reasoning parameter. A deployment that
doesn't accept it (we hit this with `gpt-4.1-mini`) fails on the first turn. Check a model with one real
session, including a tool call and a sub-agent, before you build on it.

Use the same model for sub-agents unless you've tested otherwise. Our agents ran `gpt-6-astra` for the main agent
and every sub-agent, with `gpt-6.1-sol` deployed as a faster fallback that you switch to by environment variable.
An early build on `gpt-5.4-mini` handled simple replies. We didn't test the full role (deciding when to stay
silent, delegation, multi-step document review) on smaller models. If you change model, run your scenario suite
again.

The sample's model isn't your model. The official sample deploys `gpt-chat-latest` for its Responses API call.
That says nothing about how a Copilot SDK harness behaves on it.

## Testing and release

Only count what you've seen working live. A feature that passes unit tests but hasn't run through a real
Foundry invocation, with logs, isn't done. Hosted agents fail on identity, networking and channel shape, and unit
tests reach none of those.

Use real channels for anything identity-related. Sessions you create through the API run under a different
identity from sessions a Teams or email event creates. Microsoft 365 tool calls can pass in one and fail in the
other.

Check the endpoint's auth schemes against your channels. The sample sets `BotServiceRbac`. In our tests, email
and Word comment events didn't carry the caller's object ID, so that scheme rejected them while Teams worked. We
switched to `BotServiceTenant` (plus Entra) and filtered senders by domain in code.

Release by pinning, and clear old sessions. Sessions stay on the version they started with. After routing
traffic to a new version, delete the old sessions, or people keep talking to the old code. Keep the previous
version so you can roll back by re-pinning it.

Check side effects, not just replies. "Didn't reply to a CC" can be correct and still wrong, if it also failed to
remember what the CC said. Acceptance tests should check the memory write, the tracked changes and the follow-up,
as well as the message.

Keep a record of everything you create. Note every role assignment, publication, skill version, memory store and
licence change as you go, with the command to undo it. Preview platforms make that list hard to rebuild later.

Don't trust the routine scheduler yet. The Foundry routine we set up never delivered a scheduled run. Trigger
scheduled work yourself through the agent's invocations endpoint until that's resolved.
