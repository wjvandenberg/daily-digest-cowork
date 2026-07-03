# Daily Digest — Setup Guide

**For the person reading this:** Attach this file to a Cowork session and say *"read and go"*. Claude will walk you through building your own daily digest — a one-page briefing that lands on your desk every weekday morning, covering news in the topics you care about and (optionally, if you choose) a snapshot of your calendar and a short to-do list pulled from your work tools.

The digest is **modular**. You decide what's in it:
- **At minimum: a news briefing.** Headlines on whichever topics, sectors, and regions you want covered — picked up from public web sources. Nothing private leaves your machine.
- **Optionally: today's calendar.** A line-by-line view of your meetings.
- **Optionally: a to-do list pulled from your Outlook inbox and Microsoft Teams chats.** This is the more powerful capability, but also the one that needs the most care — it requires you to give Claude read-access to your work mail, which may or may not be allowed under your organisation's policies (we'll check together up front).

Most people start with news-only and add the other pieces later if they want.

**For Claude reading this file:** This is a guided setup. Follow the phases below in order. Ask the user the questions in each phase. Use the `AskUserQuestion` tool for multiple-choice questions and plain chat for free-text questions. Don't ask everything at once — work through one phase at a time. After all questions are answered, build the deliverables described in each phase. The user is likely **not** technically trained, so use plain language and walk through Windows-side steps click-by-click. Default to the minimum-viable news-only digest unless the user explicitly asks for more.

**Question-numbering rule — apply this consistently throughout.** Whenever you ask the user a question that has two or more parts (a batched question, a multi-part follow-up, a list of options requiring a written reply), **number each part 1, 2, 3 …**. This lets the user reply with "1) X, 2) Y, 3) Z" or just "1 - X, 2 - Y" without ambiguity. `AskUserQuestion` tool calls don't need manual numbering (the UI labels them). But every free-text prompt with more than one sub-question, every offered list of split options ("A / B / C — which?"), and every grouped clarification should use explicit numbering. Mixed prose-and-bullets formats are harder to answer than a clean numbered list — prefer numbered lists.

---

## How this works at a high level

The daily digest is two pieces working together:

1. **The skill** — a recipe that tells Claude what to assemble into the briefing (news always; calendar and/or to-do list if the user opted in).
2. **The scheduled task** — an alarm clock in Cowork that runs the skill at a chosen time every weekday and delivers the result somewhere visible.

You'll design both with Claude in this conversation. The skill stays fixed (you can edit it later). The scheduled task fires automatically each morning as long as your PC is on and the Claude Desktop app is running.

---

# PHASE 0 — Scope check + connector setup

The depth of the permission check depends entirely on whether the user wants the news-only digest or also wants the inbox/Teams scan. Ask first what they want, then size the rest accordingly.

## Q0a: What do you want in the digest?

Ask via `AskUserQuestion`: *"Before we start: roughly what do you want in your daily digest?"*

Options:
- **Just news on the topics I care about** (simplest, no work-account access needed)
- **News + today's calendar overview** (needs read-access to your calendar — usually unrestricted)
- **News + calendar + a to-do list from my inbox / Teams** (more powerful, but needs the same access checks below)
- **Not sure — explain it to me**

If "Not sure", explain the three options in plain language, then re-ask.

## Q0b: Connector availability (only if Q0a included calendar or inbox/Teams)

If the user picked "News only", skip this question entirely and go to Q0c.

Frame this practically — the real gate is whether the Microsoft 365 Connector for Cowork is available to the user, not an abstract "IT policy" check. In many organisations the connector is either already installed for everyone, or installable by anyone with admin rights on their own profile, in which case no extra permission step is needed.

Ask via `AskUserQuestion`: *"To pull from your work calendar / inbox / Teams, Cowork needs the Microsoft 365 Connector. Where do you stand?"*

Options:
- **I already have the Microsoft 365 Connector installed in Cowork** → proceed straight to Q0c.
- **I don't have it yet, but I can install it myself without asking anyone** → continue to Q0c and we'll install it as part of the next step.
- **I need to ask my IT department first** → tell the user: *"Pause here. Ask your IT contact whether (i) you can install the Microsoft 365 Connector for Cowork (read-only access to Outlook + Teams + calendar), and (ii) if Google Calendar delivery is on the table later, whether routing work content through a personal Google account is allowed. Come back and we'll resume from this point."* End the session.
- **I'm not sure — let me check** → tell the user: *"Open Cowork → Customize → Connectors. Look for 'Microsoft 365'. If it shows 'Connected', you're set. If it shows 'Connect' as a button, you have the option to install it yourself. If it's not in the list at all or shows a 'locked'/'unavailable' state, your IT department has restricted it and you need to ask them."* Wait for the user's report, then re-ask Q0b.

If user picks one of the first two options, continue.

## Q0c: Connector status check

> *"Open Cowork → **Customize → Connectors** (not 'Connectors' directly — the path is via Customize). Tell me which of these show as 'Connected'."*

For news-only users, the answer is: *no connector is strictly needed for the digest content itself — the news is fetched via web search. But you will need either Google Calendar OR a way to write files locally, depending on the delivery method you pick later, so we'll come back to this in Phase 3 once you've chosen.*

For calendar + inbox/Teams users:
- *Microsoft 365 — required (for Outlook + Teams + calendar)*
- *Google Calendar — only required if you choose the Google delivery method later. Skip checking for now if you haven't decided.*

If a required connector isn't connected, walk the user through:
1. *"In Cowork, go to **Customize → Connectors**."*
2. *"Find the connector by name (Microsoft 365 / Google Calendar) and click 'Connect'."*
3. *"Sign in with the relevant account."*
4. *"Come back here when it shows as Connected."*

**Verify the connector actually works before continuing.** A "Connected" badge in the UI is not a 100% guarantee the connection is healthy — tokens expire, scopes get revoked. Once the user confirms it's Connected, run one low-impact verification call:
- For Microsoft 365: `outlook_email_search` with `folderName: "Inbox", limit: 1` — should return a single message metadata block.
- For Google Calendar: `list_calendars` — should return at least the primary calendar.

If the verification fails (auth error, empty response, NOT_FOUND), tell the user: *"The connector shows as Connected but isn't responding. Disconnect it (in Customize → Connectors → click 'Disconnect' → confirm) and reconnect from scratch. The token may have expired."*

Only continue to Phase 1 once each required connector has passed verification.

---

# PHASE 1 — Design the daily digest

Claude: ask these questions one at a time. Capture the answers; they drive Phase 2.

The Q0a answer already told you the broad scope (news only / news + calendar / news + calendar + inbox-Teams). Ask the follow-ups for each module the user opted in to. Skip the ones they didn't.

## Q1: News topics (always ask — news is the core of every digest)

**Ask the first three together in a single message** so the user can answer in one go without ping-ponging:

> *Three quick questions about the news section:*
> 1. *What topics, fields, or sectors do you want covered?* (e.g. for a tax lawyer: "international tax, transfer pricing, BEPS"; for a marketing manager: "B2B SaaS, brand strategy, growth marketing"; for a doctor: "oncology research, pharma approvals")
> 2. *What countries or regions matter most for those topics?* (e.g. "my country only", "EU-wide", "global with focus on the US and UK")
> 3. *How many separate news categories do you want — typically 2 or 3?* (one per topic cluster)

**Handling the "how many categories" question when the answer isn't obvious:** if the user has *one substantive topic across multiple jurisdictions* (e.g. "data protection law in the EU, UK, and US") or *multiple topics in one region* (e.g. "tax, accounting, audit — all in one country"), they may not know how to split. When you spot this pattern, offer concrete category options derived from their answer rather than re-asking the abstract question. Examples:
- One topic × three jurisdictions → offer: *(a) 1 category lumping all three; (b) 3 categories one per jurisdiction; (c) 2 categories grouped by region/legal-family.*
- Three topics × one region → offer: *(a) 3 categories one per topic; (b) 1 category if they're tightly related.*

**Then, two optional follow-ups** — ask only if the user hasn't already addressed them in their answer:
- *"How many headlines per category, maximum? (Default is 5 if you have no preference.)"*
- *"Anything you specifically want EXCLUDED? E.g. press-release-style PR fluff, particular sources you don't trust, particular subtopics. (Skip if nothing comes to mind.)"*

## Q2: Calendar inclusion (only if Q0a was "News + calendar" or "News + calendar + inbox/Teams")

> *Should the digest list today's meetings, including time, subject, and key attendees?*

Options (`AskUserQuestion`):
- **Yes — full calendar**
- **Yes — but skip personal/private meetings** (Claude will infer from event subject/category whether something looks personal — imperfect, so this is best-effort)
- **No, leave calendar out** (in case they change their mind from Q0a)

## Q3: Inbox / Teams to-do list (only if Q0a was the full version)

If the user chose the news-only or news+calendar option, skip this whole block.

Otherwise, the goal is to gather four things — keep it tight, don't drown the user in six prompts. Suggested order:

**Q3a — Source choice** (`AskUserQuestion`):

> *Pull a to-do list from Outlook inbox, Teams chats, or both? And should I also scan your sent items for promises you made (e.g. "I'll send the contract Monday")?*

Options:
- **Inbox only** (don't scan Sent Items)
- **Inbox + scan Sent Items for promises**
- **Teams only**
- **Both inbox + Teams (with Sent Items scan)**

That collapses two earlier questions into one.

**Q3b — Inbox style** (`AskUserQuestion`, only if Q3a included Inbox):

> *How do you manage your Outlook inbox?*

Options:
- **a) Inbox-as-to-do — I keep emails in the inbox until I've handled them; once handled I file them into subfolders. My inbox is effectively my live to-do list.**
- **b) Inbox-as-archive — most emails stay in the inbox forever; I track tasks somewhere else (a planner, paper, etc.).**
- **c) I tag tasks within Outlook — using categories (color labels) and/or flags (the follow-up flag icon).**
- **d) Outlook Tasks or Microsoft To Do — I maintain an explicit task list separate from the inbox.**

What each option triggers in the skill:

- **(a)** Treat every inbox email as a live task by definition (the "inbox-as-todo" principle). Filed = handled.
- **(b)** Look for explicit follow-up flags or recent date as the signal; don't treat unflagged old inbox items as live.
- **(c)** Ask one clarifying follow-up: *"You said you use Outlook tagging — is that categories (colour labels), flags (the follow-up flag icon), or both? And tell me what each tag/colour means, including any that mean 'done, ignore'. Example mapping: red category = urgent; yellow category = lower priority; green category = done; flagged emails = waiting on someone else."* Bake the mapping into the skill: the skill reads each email's `categories` field AND its `flag.flagStatus` field and routes accordingly.
- **(d) — Important honest answer.** *"Outlook Tasks and Microsoft To Do are a real and common workflow, but the current Microsoft 365 Connector for Cowork does **not** read those task lists. Same architectural reason the connector is read-only for everything Outlook: only inbox, calendar, Teams chats, and SharePoint are exposed; Tasks and To Do are not. Two workarounds: (i) maintain tasks in the inbox by leaving emails there until done — the skill can then treat your inbox as the task list (option (a) above); or (ii) move to a different task app that has its own MCP connector (e.g. Todoist, Notion) and tell me about it under Q4 custom watches."* Then ask the user how they want to proceed.

**Q3c — Projects, items, or work pattern** (chat, free text)

The original framing of this question — "list your top 5-10 projects" — assumes project-based work. That's the most common pattern but not universal. Frame the question to match the user's actual work shape. Open with this:

> *To make the digest accurate, I need to understand what your work looks like. Pick whichever description fits and answer in that frame:*
>
> *(A) **Project-based** — I work on a manageable number of named projects/matters/accounts in parallel (typically 3–30).*
> → *In that case: list your top 5–10 active ones, plus any keywords you want me to search for (counterparty names, project codes, internal abbreviations, technical terms). Examples by job: lawyer (matter names, opposing counsel firms, NDA/MSA/DPA/SOW), tax/accounting (client names, project codes, TP/BEPS/DAC6/CIT), consulting (project codes, workstream tags), sales (account names, deal stage tags), doctor (study names, drug names, principal investigators), engineer (repo names, codenames, ticket prefixes).*
>
> *(B) **Single big matter** — I have one major project/case/deal that dominates my time, plus some smaller things.*
> → *In that case: tell me the name of the big one and any keywords particular to it. We can list the smaller stuff loosely.*
>
> *(C) **High-volume queue** — I process a steady stream of similar items each day (e.g. applications, KYC forms, support tickets, casework, contracts to review, claims to triage).*
> → *In that case: there's nothing project-shaped to search for. We'll skip the keyword sweep entirely and rely on (i) the priority signal you use day-to-day — Outlook categories if you flagged them in Q3b, otherwise some other field — and (ii) optional pattern-flags within the item bodies. Tell me: roughly how many items per day, what kind, and what patterns within them you'd want flagged (e.g. for insurance claims: "fraud indicators", "high-value claim", "litigation pending", "settlement deadline approaching"; for KYC: "PEP flags", "high-risk jurisdictions", "missing documents"; for support: "P1 incident", "VIP customer", "SLA breach risk").*
>
> *(D) **Mixed** — some of A, some of B, some of C.*
> → *Describe it however makes sense.*

This open-ended branching is the heaviest question of the whole setup; let the user take their time. Once they answer, internally classify their work pattern (A / B / C / D) — that classification drives whether the skill uses a keyword sweep, a single-matter focus, a pattern-flag-based scan within bodies, or a hybrid.

**Q3d — Optional: separate hermetic folders** (chat, free text)

> *Optional: any specific Outlook folders — other than Inbox — that you want me to treat as a separate to-do source? E.g. a "Project ideas" folder, a "Reading pile", or a folder where you park items that still need action but are lower priority. Skip if not.*

If the user names a separate folder, **treat it as hermetically isolated** in the skill — items from that folder go in their own section, not mixed with inbox items, and items from anywhere else (even thematically related) don't bleed into that section. (The phrase "the folder is the source of truth, not the topic" is the rule.) If Q3b was "Outlook categories", the same colour filter applies inside that hermetic folder — green = ignore, etc.

## Q4: Custom watches — websites, RSS feeds, sources

> *Are there any specific sources you'd like me to monitor each morning beyond the news scan we set up in Q1? For example a government regulator's website, an RSS feed, a particular blog or newsletter, a GitHub repo for new releases — anything where you want to know "what's new since yesterday".*

Free text. The user may say no. If they describe something, take careful notes — you'll need to add a custom step to the skill.

**Examples of things the user might describe** (so you know what's possible):
- "Watch a country's tax authority website for new rulings on a topic" — Claude uses WebSearch for this, anchored to the previous digest's timestamp.
- "Watch a specific RSS feed" — Claude uses WebFetch on the RSS URL.
- "Track a specific Outlook folder where I park ideas" — Claude reads that folder (mind the hermetic-folder rule from Q3, *and* only ask this if they opted into the inbox/Teams version).
- "Watch a specific GitHub repo for new releases" — Claude uses WebSearch / WebFetch.
- "Watch a specific Substack newsletter sender (e.g. by sender email)" — Claude searches the user's mailbox across folders for that sender and reads the bodies; useful for newsletters that get filtered to Junk or a subscriptions subfolder.

**Important fallback rule — graceful degradation when RSS doesn't exist.** When the user names a source (court website, regulator, blog, news site), don't promise an RSS feed unless you've verified one exists. Many official sources — especially national courts and government bodies — don't publish RSS at all. The right pattern is:
1. First try WebFetch on the canonical RSS URL for the source (common patterns: `/rss`, `/feed`, `/rss.xml`, `/news.rss`).
2. If WebFetch returns an actual feed (XML or JSON content), use it.
3. If WebFetch returns 404, an empty body, or HTML instead of a feed, **silently fall back to WebSearch** anchored to the previous digest's timestamp, restricted to that source's domain (`site:example.gov`).
4. Don't tell the user upfront "I'll watch your RSS feed" — say "I'll watch this source" — because the implementation may end up being WebSearch, and they don't need to care which.

Bake this fallback into the skill's methodology for the custom watch step so it works on every future run.

## Q5: Final catch-all — anything else at all?

Before moving to Phase 2, ask once more with a wide net. The previous questions framed everything around news, calendar, inbox, custom watches — useful framing but it can blind the user to other things they'd love in a morning digest. Ask:

> *One last question before I build this — is there anything else you'd want to see in your daily digest that we haven't covered yet? Think wide:*
>
> 1. *Personal lists or trackers — a Todoist / Notion / Microsoft To Do board, a habit tracker, a reading list, a project board you keep elsewhere.*
> 2. *Recurring personal reminders — "remind me on Mondays to submit my timesheet", "every Friday a checklist of weekend prep items", birthdays/anniversaries of people you want to remember.*
> 3. *Daily context items — local weather forecast, commute traffic, currency rates or stocks you watch, sports scores, sunrise/sunset, today's word, news ticker from a specific publication.*
> 4. *Health and routines — yesterday's step count if you sync a fitness app, calendar of meals planned for today, medication reminders, gym sessions you've booked.*
> 5. *Inspiration and learning — a daily quote, a curated reading from a substack, a single new word in a language you're learning, a daily prompt for journalling.*
> 6. *Anything else at all — it doesn't have to fit a category. If you'd want it in a morning briefing, tell me and I'll see what's possible.*

Free text. Treat each item the user names as a candidate for either (a) a new custom watch step in the skill (if it's information that has to be fetched), (b) a recurring static-content section (e.g. a fixed checklist that appears every Friday), or (c) a connector-driven section (e.g. Todoist's MCP if installed). For anything you can't actually deliver — be honest about the limit and suggest the closest alternative.

**Common patterns and how to deliver them:**
- *Weather forecast* → WebSearch for `weather [user's city] today` or use a weather connector if installed. One-line summary at the top of the digest works well.
- *Todoist / Notion / Microsoft To Do tasks* → if the corresponding MCP connector is installed, read tasks from there directly. If not, ask the user to install the connector, or explain it's not currently fetchable.
- *Recurring static reminders* (e.g. "every Monday: submit timesheet") → bake into the skill as a day-of-week conditional that adds a fixed line to the digest on matching days. No data fetching needed; the skill just emits the text on the right weekday.
- *Birthdays / anniversaries* → if the user keeps these in their calendar, the calendar step (Q2) catches them automatically. If kept elsewhere (a contacts app, a separate sheet), ask the user to keep them in their calendar with all-day events, then the existing calendar step handles it.
- *Stocks / currency rates / sports scores* → WebSearch or a dedicated MCP connector if installed (e.g. financial data, sports). Snippet-level summary in a dedicated digest sub-section.
- *Daily quote / language word / journaling prompt* → static content the skill can rotate from a list, or fetch from a free public source if the user has one in mind.

If the user names something that needs a connector you can detect via the MCP registry, briefly mention they can install it (e.g. *"Todoist has a connector available — install it via Customize → Connectors and we can pull tasks directly"*). Don't force the install during the setup; just flag the option.

If the user says no, move on to Phase 2.

# PHASE 2 — Build the skill

Now compose a customized `SKILL.md` for this user. Save it to a folder the user can access (ask them where to put it — default suggestion: `C:\Users\[username]\Documents\Daily Digest\`).

## What MUST be in the skill regardless of the user's answers

These are universal principles. Bake them in.

**A scoping note:** the principles below assume the digest includes inbox/Teams scanning. For a **news-only** digest, most of them don't apply (there's no Inbox to triage, no Teams chats to read, no closure to detect). The relevant principles for news-only are: principle 5 (one-page hard cap), principle 10 (output discipline), principle 13 (anchor recurring sections to last digest's timestamp), and principle 14 (skip empty sections). The rest apply once the user opts in to inbox/Teams scanning.

### Core principle 1 — Quote, don't synthesise

When the skill lists a task in the final digest, it must be traceable to verbatim text from the source (email body, Teams message, calendar event). If you can't quote the source, don't list the task. This prevents "phantom tasks" — items invented by loose summarisation that don't actually exist in the user's mailbox.

### Core principle 2 — Cross-channel forward-temporal closure

Before listing any task as still open, look forward in time from when it was raised, across all channels:
- **Same channel forward** — any subsequent replies?
- **Cross-channel forward** — a Teams ask may be resolved by an email reply (and vice-versa). Always check both.
- **Calendar forward** — a meeting/call on the matter after the ask date typically resolves or progresses it.
- **Sent-items forward** — did the user deliver, respond, or close the loop?

Drop items that show closure. List ambiguous items under **"Need to confirm"** rather than dropping silently or listing as live.

### Core principle 3 — Phone calls are a closure blind spot

The connector cannot see placed phone calls. "Call me / let's hop on a call / can we speak" type asks have no reliable closure signal in the data. Default these to **"Need to confirm"**, not "live" — the call may well have happened off-system.

### Core principle 4 — Folder source isolation

If the user named a specific Outlook folder in Q3d as a separate to-do source (e.g. a "Project ideas" folder), items in the corresponding output section must come **only from that folder**. Even thematically-related items in the Inbox do not belong in that section. The folder is the source of truth, not the topic. Add a pre-emit sanity check: every bullet in that section must have a `parentFolderId` matching the named folder.

### Core principle 5 — One page hard cap

The digest is one screen, period. Ruthlessly prioritise; cut from "lower priority" first. A digest that doesn't fit on one screen has failed at its job.

### Core principle 6 — Treat what's in inbox vs filed differently (only if Q3b answer was (a))

If the user works inbox-as-to-do (Q3b = (a)), then anything still in their Inbox is live by definition; time is irrelevant — a year-old Inbox email may be perfectly live, a two-day-old filed email is closed. Folder state is the primary signal, not date.

### Core principle 7 — Use a subagent for the inbox heavy lift

The combined Inbox scan + past-month sweep can run to 100+ messages, which overflows the main Claude context if processed inline. The skill must spawn a **subagent** (via the `Agent` tool, `general-purpose` type) to do the bulk inbox processing — paginated batches of 25 with `folderName: "Inbox"`, plus the past-month sweep, plus the per-candidate closure workflow — and return a tight structured triage (urgent / lower-priority / waiting / need-to-confirm / resolved). The main thread synthesises the final digest from the subagent's structured output. **Without this, the skill will fail with context overflow on real-world mailboxes.**

### Core principle 8 — Double-check pass on items classified as closed

"Run everything twice" is expensive. Skip it. But there's a cheap, high-value version: on the second pass, **re-read only the items the first pass classified as closed**. False-closed is the costliest failure mode of this skill (silently dropping a task the user still needed). If anything is ambiguous on second read, move it back to live, or into "Need to confirm". This is mandatory.

### Core principle 9 — Salutation triage for inbox messages

The salutation in the email body is the **primary** signal of who an email is actually addressed to — not the To/CC fields, which are routinely noisy. **Include** emails where the user is addressed by name in the salutation (their name in any language they work in — "Dear [name]", "Hi [name]", "Hola [name]", "Bonjour [name]", etc.). **Exclude** emails where the salutation addresses someone else by name, even when the user is on the To or CC list. Group salutations ("Dear all", "Hi everyone", "Hola a todos") count if there's a clear group task. This filter is the single biggest determinant of whether the digest reads clean (one page) or noisy (two pages of irrelevant CCs).

### Core principle 10 — Output discipline

Hard rules for what the digest actually says:
- One page maximum. If too many items, ruthlessly cut from "Lower priority" first.
- Every bullet starts with a verb (Draft, Review, Send, Follow up on, Ask, Confirm). No passive descriptions.
- **No "X days overdue" labels.** Time is not a staleness signal when the inbox is the to-do list. Note last activity in the thread if useful, but never frame it as days-overdue.
- Skip sections entirely when nothing's there — don't write "no updates" placeholders.
- For news, headline + source URL only. Max bullets per the user's Q1 follow-up (headlines-per-category cap; default 5).

### Core principle 11 — Today's context first

Before triaging the Inbox, pull three things in parallel that form the lens for the rest of the run:
- **Today's calendar.** Meeting subjects and attendees are workstream signals — they tell you what's actively in flux. If a meeting at 10am names project X with the user's external counsel, that's the live thread for project X today; tasks related to X should be read against that context.
- **Recent Teams from key colleagues** (past 24-48h). The strongest single signal of what's in flux right now and frequently contains closure evidence for Inbox items (e.g. "thanks for the call yesterday — I'll send the redline this evening" closes the "review redline" task even if the email thread doesn't reflect it yet).
- **Recent sent emails from the user** (past 24-48h). Anything the user said yesterday morning may close a request listed elsewhere.

Without this lens, the rest of the methodology runs blind.

### Core principle 12 — "Need to confirm" is the safety valve

When the closure-detection workflow is ambiguous, surface the item under **Need to confirm**, not as live and not silently dropped. A false-positive (listing a closed task) burns trust slowly; a false-negative (silently dropping a live one) is the cardinal sin. "Need to confirm" makes ambiguity visible at zero cost. Use it liberally.

### Core principle 13 — Anchor recurring items to the previous digest's timestamp

For any section that's about "what's new" — news, legislation watches, custom feeds — anchor strictly to the previous digest's run-date. Look up the previous digest in the chat history. Only include items dated after that. If an item was already covered yesterday, don't repeat it. If no previous digest exists, default to past 7 days. If nothing new for a watch section, skip the section entirely.

### Core principle 14 — "Resolved since last digest" is a verification surface

Maintain a small section called "Resolved since last digest" that lists items closure-detection caught and removed from the live list (typically 2–5 bullets, often skipped). This isn't for completeness — it exists so the user can spot-check that closure detection is working. If the user ever sees a task in "Resolved" that's actually still open, that's diagnostic gold for tuning the skill. Without this surface, the user has no way to detect false-closures.

---

## Worked failure modes — concrete examples to bake in to the skill

These are real failure patterns earned from running the skill in production. Include them as worked examples in the skill so future Claude instances have anti-patterns to match against.

**Failure 1 — The phone call already happened.** Counterparty Teams message: "can we hop on a call?". User placed the call same day. Closure lived in the placed call, not in any message. Skill listed "call counterparty back" as a live urgent task the next morning. → Rule: "call me" / "let's talk" / "give me a call" type asks default to **Need to confirm**, with a note "may have been resolved by a placed call — verify".

**Failure 2 — Resolved by email after a Teams ask.** Internal Teams message: "what's the right delivery channel for the execution version, email or secure portal?". User replied via *email* directly to external counsel and got confirmation that way. Skill, scanning Teams only, listed it as still open. → Rule: cross-channel forward checking is mandatory; a question raised in one channel may be resolved in another.

**Failure 3 — Phantom task from synthesis drift.** Subagent processing Teams chats saw a message mentioning one document type and another mentioning a separately scoped task; loose summary conflated them into a phantom task that combined both. The actual task was simpler and unrelated to the conflated framing. → Rule: every listed task must be traceable to verbatim text from the source. If you can't quote it, don't list it.

**Failure 4 — Folder bleed-in.** User has a "Project ideas" folder where they park self-notes about products they want to build. Skill correctly listed those as a separate to-do section, but *also* included a personal Inbox note ("buy bread") that was thematically a self-note — pulled it into the Project ideas section because it looked similar. → Rule: hermetic folder source isolation. Project-folder bullets must have `parentFolderId` matching the named folder; everything else goes into the regular Inbox-triage sections.

**Failure 5 — Wrong-attribution to-do owner.** Skill listed "Waiting: next draft of contract X from counterparty Y". Wrong owner — the actual scribe was external counsel Z, who circulates an updated draft after each meeting; counterparty Y doesn't write the draft. The skill could have seen this by looking at meeting attendees + recent emails, but it inferred ownership from one stale email instead. → Rule: when assigning a "waiting on" owner, check the most recent few emails on the matter and the calendar to identify who's actually driving the document, not just who was the first sender.

## Customisation map

Use the user's answers to assemble the right combination of methodology steps:

- **Q1 (news topics)** → always include a news scan step with one sub-section per category the user listed. Each sub-section has its own keyword list and max-bullets cap. Anchor to previous-digest timestamp (Core principle 13).
- **Q2 = Yes (calendar)** → include calendar pull, with timezone matching the user's location. Include all-day-midnight-marker handling.
- **Q2 = Yes-private-skip** → add a "skip personal-looking events" filter, best-effort.
- **Q3 includes Inbox** (only if Q0a was the full version) → include a paginated Inbox scan (batches of 25 with `folderName: "Inbox"`; `limit: 50` overflows context) + a past-month general sweep for closure evidence. Use a subagent for the heavy lift (Core principle 7).
- **Q3 includes Teams** → include broad Teams search + targeted "key colleagues × project keywords" searches.
- **Q3 sent-items follow-up = Yes** → include a sent-items commitment scan (past 2 months, queries like `"I will" OR "I'll" OR "we will"` etc., plus equivalents in any other languages the user works in — ask them).
- **Q3 inbox style = a (inbox-as-to-do)** → mark the inbox-as-to-do principle as PRIMARY. Drop any time-based staleness logic.
- **Q3 separate folder** → add a hermetic sub-step for that folder with the folder-isolation rule (Core principle 4).
- **Q4 = anything custom** → add a dedicated methodology step for it; explain in the skill *why* this step exists (so future iterations don't accidentally drop it). Anchor to previous-digest timestamp if it's a recurring watch.
- **Q5 = anything else from the catch-all** → for each item the user named, decide which output section it belongs in (personal lists may get their own section; weather/quotes/learning items typically go at the very top or bottom of the digest as a short header/footer line; recurring static reminders are emitted by a day-of-week conditional). Add a methodology step for each fetched item, and a content-emission step for each static item. As with Q4, explain in the skill *why* each step exists.

## Output format the skill should produce

```
## [Date] — Morning Overview

### Meetings today
- HH:MM–HH:MM — [subject] ([key attendees])
- ...

### Urgent / overdue
- **[Verb-led task]** — [person/project] — [why urgent]
- ...

### Lower priority
- [Verb-led task] — [context]
- ...

### Other open items
- Waiting: [what] — [who owes it] ([since when])
- **Need to confirm:**
  - [item] — [why ambiguous]

### Resolved since last digest (optional)
- [matter] — closed because [reason]

### [Any custom sections from Q5]

### News — [Category 1]
- [headline] — [source URL]

### News — [Category 2]
- [headline] — [source URL]
```

Rules: bullets only, no paragraphs. Verb-led ("Draft", "Review", "Send", "Follow up on", "Ask"). One page max. News items: headline + source URL, that's it.

## Compose, package, and present as an installable .skill file

Compose the SKILL.md based on the customisation map. Then **package it as a one-click-installable `.skill` file** — a `.skill` is just a zip archive of the skill folder with a specific extension, but Cowork renders it as an install card with a "Save skill" button, so the user gets a one-click install instead of having to fiddle in the Customize section or zip anything by hand.

**Mandatory attribution footer** — every generated SKILL.md **must** end with this exact footer as its final content, after a horizontal rule (`---`). This is provenance, not marketing: it tells the user (and any future Claude picking up the skill) where the skill's design comes from, and gives them a way back to the source guide if they lose the link. Keep it verbatim — no expansion, no logo, no additional CTAs:

```markdown
---

*Generated from the daily-digest guide by Wouter van den Berg — linkedin.com/in/wjvandenberg. Customise freely; the guide lives at github.com/wjvandenberg/daily-digest-cowork.*
```

The user may edit or delete the footer — that's fine. Don't warn them off it, don't lock it. It's a courtesy line, not a required marker.

**Do not deliver a loose SKILL.md and ask the user to add it manually** — non-technical users don't know what a skill folder is, don't know how to zip one, and often don't even know Cowork *has* a Customize section. The `.skill` file makes all of that invisible.

**Steps:**

1. **Layout in the working folder.** Save the SKILL.md inside a subfolder called `daily-digest` (the folder name becomes the skill identifier). So the source path is:
   ```
   C:\Users\[username]\Documents\Daily Digest\daily-digest\SKILL.md
   ```
2. **Package** the folder into a `.skill` archive at:
   ```
   C:\Users\[username]\Documents\Daily Digest\daily-digest.skill
   ```
   **Assume the user does NOT have the skill-creator skill installed** — most non-technical users won't, and the guide should not depend on it. A `.skill` file is simply a zip archive of the skill folder with a `.skill` extension instead of `.zip`, so plain `zip` via bash is the primary method:
   ```bash
   cd "/sessions/<session>/mnt/Documents/Daily Digest/daily-digest"
   zip -j "/sessions/<session>/mnt/Documents/Daily Digest/daily-digest.skill" SKILL.md
   ```
   If the daily-digest folder has additional files beyond SKILL.md (e.g. reference sub-files under `assets/`), use `zip -r` from the parent folder instead of `zip -j` — the archive needs the folder structure preserved:
   ```bash
   cd "/sessions/<session>/mnt/Documents/Daily Digest"
   zip -r "daily-digest.skill" "daily-digest/"
   ```
   *(Optional, if skill-creator happens to be installed on this session — you'll know because you'll see it in the available skills list — you can use `python -m scripts.package_skill <folder> <out>` instead. Same output, slightly nicer packaging. But do not INSTALL skill-creator for the user just to produce this file; the bash zip does the same job.)*
3. **Sanity-check** the archive: it should be a couple of KB, unzipping cleanly should yield `SKILL.md` (and any other skill files) at the top level. If unzipping produces a nested `daily-digest/` folder rather than SKILL.md at the root, that's usually fine — Cowork's installer handles both layouts — but the flat layout installs slightly faster.
4. **Present** the `.skill` file to the user with a `present_files` block pointing at `C:\Users\[username]\Documents\Daily Digest\daily-digest.skill`. This renders in Cowork as an install card with a **Save skill** button — one click and the skill is installed.
5. **Tell the user, verbatim:** *"This is a `.skill` file — one-click installable. Click **Save skill** on the card above and the daily-digest skill is installed into Cowork; you'll then be able to invoke it from any chat by saying 'morning', 'daily digest', 'tasks', or any of the trigger phrases we set up. The raw SKILL.md source is also in your `Daily Digest` folder if you ever want to edit it by hand later — just re-run this guide's packaging step to produce a fresh `.skill` after edits."*

**Before packaging**, give the user a brief summary of the customisations the SKILL.md reflects (news categories, whether inbox/Teams is included, any Q5 catch-all additions) and ask them to skim and confirm. If they want changes, edit the SKILL.md and re-run the packaging step. Don't skip this confirmation — it's cheaper to catch a wrong category or missing source now than to discover it in the first live digest tomorrow morning.

# PHASE 3 — Design the scheduled task

## Q6: What time should it run, and on which days?

> *What time every day should the daily digest run? And which days — every weekday, or specific days?*

Free text. Defaults to suggest: 08:45 CET on weekdays.

**Important warning to relay to the user:** *"The Cowork scheduled task itself runs in the cloud, so the timing works regardless of whether your PC is on. But if you choose Option B (local HTML) or Option C (Outlook .ics), the file the task writes only appears on your local disk when the Cowork desktop app is running. And the Windows Task Scheduler job that opens it only fires when your laptop is on and unlocked. Question Q7b will handle the trigger-timing choice — for now, just pick a Cowork task time that gives you comfortable headroom (e.g. 08:45 if you're usually online by 09:00–09:30, or earlier if you're an early bird)."*

## Q7: Which delivery method?

Use `AskUserQuestion` to present these three options. For the **Google Calendar** option, the warning text below MUST be shown to the user — don't paraphrase, surface the security concern verbatim:

**Option A — Google Calendar invite into Outlook.**

> Cowork creates a Google Calendar event with the digest in the meeting body, and invites your work Outlook email as an external attendee. Outlook auto-adds it as a tentative meeting. The digest text routes through your personal Google Calendar.
>
> **⚠️ Security warning:** If your digest contains content pulled from your work inbox or Teams chats (e.g. confidential matters, client information, internal commercial details), that content is routed through your personal Google account. Many firms — especially in legal, finance, healthcare, and government sectors — do not allow this. Check your IT department's policies before choosing this method. If you only want news in your digest, this risk is lower.

**Option B — Local HTML file + Windows browser popup.**

> Cowork writes the digest as a local HTML file to a folder on your PC. Windows Task Scheduler opens that file in your default browser at the time you choose. Fully local, nothing ever leaves your computer. Most secure option for sensitive content. Requires a one-time five-minute setup of a Windows Task Scheduler job.
>
> **⚠️ Timing note:** The digest file is written by Cowork's cloud task (~5–15 min depending on inbox size and network). Windows Task Scheduler is separate: it needs your laptop to be *on and unlocked* to fire. If you open your laptop at very different times day to day, a fixed-time Windows trigger will sometimes miss — you'll open the laptop and no digest pops up. Question Q7b (next) will pick the right trigger type to make this reliable.

**Option C — Outlook Classic .ics import.**

> Cowork writes a calendar (.ics) file. Windows Task Scheduler opens it in Outlook Classic, which prompts you to add it to your Outlook calendar. Sometimes the most "Outlook-native" feel.
>
> **⚠️ Compatibility warning:** This method does **not** work in new Outlook (the rewritten app Microsoft is pushing to all users). If your firm is migrating you to new Outlook — or has already — this method will silently stop working. Also requires you to click "Save & Close" each morning to actually add the event to your calendar, unlike option A which is fully automatic.
>
> **⚠️ Timing note:** Same laptop-must-be-on constraint as Option B — Question Q7b (next) picks the trigger type accordingly.

**Why not Outlook Tasks / Microsoft To Do as a fourth option?** Users often ask: *"Can the digest items become Outlook Tasks or Microsoft To Do items so I tick them off as I go?"* The answer is no — the Microsoft 365 Connector for Cowork is **read-only** across the board (no `create_event`, no `send_mail`, no `create_task`, no Tasks API at all). The same architectural constraint that forces the Google-Calendar-invite workaround in option A also blocks writing to Outlook Tasks or Microsoft To Do. Microsoft Graph the underlying API supports both reading and writing tasks, but the connector doesn't expose those endpoints. If this changes (the connector gains write capability), Outlook Tasks / To Do would become a natural fourth delivery method — worth re-checking the connector registry periodically. Until then, the three options above are the only programmatic delivery paths.

## Q7b: How consistent is your morning routine? *(only if Option B or C)*

**Why this matters.** Cowork's cloud task writes the digest file at a fixed time. Windows Task Scheduler then needs to fire *after* that, when the laptop is on. If the laptop opens at wildly different times day to day, a fixed clock trigger misses more often than it fires. This question picks the right Windows trigger type.

Ask via `AskUserQuestion`:

> *When do you typically open your laptop or start your workday?*
>
> **(a) Same time every day, within 30 minutes.** *(You open the laptop around 08:30 / 09:00 / etc, almost always.)*
> **(b) Varies by 1+ hours.** *(Some days you're on at 07:30, some days not until 10:00.)*
> **(c) Sometimes I don't open it until afternoon.** *(Truly variable.)*
> **(d) Not sure / no fixed pattern.**

**How the answer drives the Windows Task Scheduler setup in Path B / C:**

- **Answer (a) — consistent routine** → **Fixed-time CalendarTrigger** at digest-time + 30 min (e.g. digest runs 08:45 CET, Windows Task Scheduler fires 09:15), with **`<StartWhenAvailable>true</StartWhenAvailable>`** (Windows's "Run task as soon as possible after a scheduled start is missed" — handles the laptop-was-off-at-fire-time case). Simple, one trigger, works for the vast majority of consistent-routine users. Use the XML from B.3.a as-is.
- **Answer (b), (c), or (d) — variable routine** → **LogonTrigger (or UnlockTrigger)** — fires whenever the user next logs in or unlocks their machine — wrapped by a **freshness-check helper script** so it doesn't open yesterday's digest by mistake. This is the more robust setup; the trade-off is one extra helper file and slightly more explanation during install. Path B.3 branches here to the wrapper variant, walked through in B.3.b.

Note in your session tracking which branch you're on — everything downstream in Path B (or Path C) depends on this.

## Q8 — (only if Option A or C) — What time should the invite/event be for?

> *What time should the meeting in the invite be set for? This is different from when the digest runs.*

Free text. Default: 08:00 (or whenever the user typically opens email).

**Explain to the user:** *"Set this earlier than the time the digest runs from Q6. For example: digest runs at 08:45, but the meeting is set for 08:00. That way the calendar entry sits before your morning meetings (so it doesn't conflict with 08:30 or 09:00 standups). The invite will arrive at 08:45 marked as already past, but it still appears on your calendar at 08:00 — that's fine and intentional."*

# PHASE 4 — Build the scheduled task + any delivery artifacts

Branch on the user's Q7 answer.

## Path A — Google Calendar invite

### A.1: Confirm the user's Google Calendar is connected to Cowork

If not, walk the user through connecting it explicitly:
1. *"In Cowork, go to **Customize → Connectors** (not just 'Connectors' — the path is via the Customize menu)."*
2. *"Find 'Google Calendar' in the list and click 'Connect'."*
3. *"A browser window will open asking you to sign in to Google. Use the Google account you want to send the daily invites FROM. This doesn't need to be your work account — it just needs to be a Google account you control."*
4. *"Grant Cowork the calendar write permission when prompted."*
5. *"Come back to Cowork and tell me when it shows as 'Connected'."*

Once the user reports it's connected, verify it works by running `list_calendars`. If the response is empty or errors, walk them through disconnecting (Customize → Connectors → Google Calendar → Disconnect) and reconnecting. If it returns at least one calendar, you're good.

### A.2: Confirm the work email address

Ask: *"What's your work email address where the invite should arrive? (For example: jane.smith@yourfirm.com)"*

### A.3: Create the Cowork scheduled task

Use the `create_scheduled_task` tool. Settings:

- `taskId`: `daily-digest`
- `cronExpression`: derived from Q6 (e.g. `45 8 * * 1-5` for 08:45 weekdays in local time)
- `description`: one-line summary
- `prompt`: the canonical prompt below

**Canonical scheduled task prompt for Path A** — adapt the time and email to the user's answers:

```
Run the user's daily-digest skill (saved at [path-to-skill]). Each run has no memory of prior runs; the skill itself is the source of truth.

After producing the digest, render it into the table-based Outlook-tested HTML template (single line of HTML, no source-code newlines, <tr><td> per bullet, height="14" spacer rows between sections — see the skill's "Scheduled delivery format" appendix if there is one, otherwise use the format below).

Then create a Google Calendar event via mcp__7038610b...__create_event with:
- summary: "Daily Digest [today's date in DD Month YYYY format]"
- startTime: [today]T[invite-time]:00 (e.g. T08:00:00)
- endTime: [today]T[invite-time + 15min]:00
- timeZone: "[user's IANA timezone, e.g. America/New_York or Europe/London]"
- visibility: "private"   (MANDATORY — Google marks the event Private, and when the invite lands in Outlook it maps to Sensitivity=Private so anyone with shared-calendar/delegate access to the user's mailbox sees "Private Appointment" instead of the digest text)
- attendeeEmails: ["[user-work-email]"]
- notificationLevel: "EXTERNAL_ONLY"
- addGoogleMeetUrl: false  (CRITICAL — without this Google auto-attaches Meet)
- description: the HTML body

Do not produce chat output. Delivery is the calendar invite.
```

### A.4: The Outlook-tested HTML template

If the skill doesn't already have a delivery-format appendix, embed this in the scheduled task prompt. Outlook's calendar description renderer strips most CSS; the only reliable pattern is `<table>` with explicit `height="14"` spacer rows. Use this skeleton:

```
<table cellpadding="0" cellspacing="0" border="0" style="font-family:Calibri,sans-serif;font-size:11pt;line-height:1.3;border-collapse:collapse;color:#222;"><tbody>
<tr><td><b style="font-size:13pt;">[Date] &mdash; Morning Overview</b></td></tr>
<tr><td height="14" style="height:14px;line-height:14px;font-size:11pt;">&nbsp;</td></tr>
<tr><td><b>Meetings today</b></td></tr>
<tr><td>&bull; [item]</td></tr>
...
<tr><td height="14" style="height:14px;line-height:14px;font-size:11pt;">&nbsp;</td></tr>
<tr><td><b>Urgent / overdue</b></td></tr>
... etc ...
</tbody></table>
```

Critical: use HTML entities (`&mdash;`, `&bull;`, `&nbsp;`, `&deg;`), not raw Unicode. Use `<b>` and `<i>` for emphasis. Hyperlinks via `<a href="…">…</a>`. No `<ul>/<li>` (Outlook adds huge default margins). No `<p>` (Outlook adds paragraph spacing). No `<br>`. The whole description must be ONE line of HTML (no source-code newlines — Outlook converts whitespace between tags into phantom paragraph breaks).

### A.5: One manual one-off the user must do (current MCP limitation)

Tell the user: *"There's one thing the connector doesn't let me set: Show as Busy vs. Free. Every new event defaults to Busy. The morning after the first run, open the event in Google Calendar, change Busy → Free, save. The setting sticks per event. Until the connector exposes this field, you'll do this once per day — about three seconds of clicking. I'm sorry, it's a real gap in the API."*

### A.6: Test it

Trigger the scheduled task manually (in Cowork's "Scheduled" section, click "Run now"). Check Outlook for the invite. If it doesn't arrive, check Outlook's "External meetings" folder or junk mail — corporate filtering sometimes routes external invites there the first time.

## Path B — Local HTML file + Windows Task Scheduler

This is two pieces that work together. Make sure the user understands the architecture before you start: Cowork writes the file; Windows opens it. They're independent jobs.

### B.1: The working folder — Claude creates it, don't ask the user to

**Standard location** — always use `C:\Users\[username]\Documents\Daily Digest\`. This one folder holds everything: the digest file that Cowork writes daily, the Task Scheduler XML, any PowerShell wrapper scripts, the source SKILL.md, and the packaged `.skill` file. One folder to back up, one folder to inspect if anything breaks.

**Do not ask the user to create this folder manually.** Non-technical users get stuck on "browse to Documents, right-click, New Folder, name it exactly right, click into it" — it's four steps of friction for something Claude can do in one command. Claude creates the folder automatically:

1. If Cowork already has write access to the user's Documents folder (a session started with Documents as a connected folder), just `mkdir -p` the subfolder via bash. Path mapping: `C:\Users\[username]\Documents\Daily Digest` → `/sessions/<session>/mnt/Documents/Daily Digest`.
2. If Cowork doesn't yet have Documents as a connected folder, call `request_cowork_directory` with `C:\Users\[username]\Documents` — the user gets a folder-picker to approve. Once mounted, `mkdir` the subfolder.
3. Confirm to the user: *"I've created `C:\Users\[your-username]\Documents\Daily Digest\` — that's where the daily digest files, the Task Scheduler task, and the installable skill will all live. If you ever want to see or back up everything, that one folder has it."*

**Only if the user wants a different location** (e.g. because their organisation has a redirected Documents folder, or they want the digest on a specific drive), ask them to type the full Windows path (e.g. `D:\Work\Daily Digest`) and use `request_cowork_directory` to mount that. Then substitute the alternative path in every subsequent reference.

### B.1b: Browser handler — use the default, no need to ask which browser

**Do not ask the user which browser they want or fish for the browser's .exe path.** Every non-technical user knows how to open a browser but few know where its .exe lives, and the answer differs across Chrome / Edge / Firefox / Brave / Opera / per-user vs machine-wide installs. Instead, let Windows handle it: the Task Scheduler task opens the digest HTML via Windows's built-in file-association mechanism, which dispatches to the user's default browser (whatever that is). Same UX outcome as opening any HTML link, no configuration required.

**How the two Path B variants achieve this:**
- **Fixed-time variant (B.3.a)** — the Task Scheduler XML calls `cmd.exe /c start "" "path.html"` directly. Windows's file-association handling routes the .html to the default browser. No PowerShell wrapper needed for this variant.
- **Log-on variant (B.3.b)** — needs a small PowerShell wrapper (for the freshness check and one-per-day sentinel), and the wrapper uses `Start-Process $HtmlPath` — same default-browser dispatch, just wrapped in a script.

Either way, the browser question doesn't need to be asked. Whichever browser the user has set as default handles the .html — Chrome, Edge, Firefox, Brave, Opera, whatever.

**Only if the user explicitly asks** to force a specific browser (rare — "always open in Chrome, my default is Edge but I want the digest in Chrome"), fall back to asking for the .exe path and hardcoding it. This should be a footnote scenario, not the default flow.

### B.2: Create the Cowork scheduled task

Settings:
- `taskId`: `daily-digest`
- `cronExpression`: from Q6
- `prompt`:

```
Run the user's daily-digest skill (saved at [path-to-skill]). Each run has no memory of prior runs; the skill is the source of truth.

After producing the digest, write it as a self-contained HTML file to:
[user-chosen-folder]\Daily Digest.html

Overwrite the file each run. Use simple modern CSS — full browsers handle CSS well, so you don't need the Outlook table-trickery; use proper <h1>, <h2>, <ul>, <li>, hyperlinks, etc. Aim for a clean, readable single-page layout.

Do not produce chat output. Delivery is the file.
```

### B.3: Create the Windows Task Scheduler XML

**⚠️ Windows encoding gotchas — read both before generating files:**

**(1) Task Scheduler XML must be UTF-16 LE with BOM.** See the detailed note immediately below.

**(2) PowerShell wrapper `.ps1` files must be pure ASCII (or UTF-8 with BOM).** When Claude generates any wrapper `.ps1` file mentioned later in this guide (B.3.b `Open Daily Digest.ps1`, Path C Tier 2 `Open Daily Digest ICS.ps1`), use **plain ASCII characters only** — no em-dashes (`—`), no en-dashes (`–`), no smart quotes (`"..."`), no non-ASCII punctuation. Reason: Windows PowerShell 5.1 (the default on Windows 10/11) reads `.ps1` files as **Windows-1252**, not UTF-8. Any UTF-8 non-ASCII character in a comment or string will render as mojibake (`â€"` for em-dash) and PowerShell crashes with:
> ```
> Unexpected token 'silent' in expression or statement.
> Missing closing ')' in expression.
> The string is missing the terminator: ".
> ```
> before the script's first `Write-Log` call — so the user sees a command window flash and no log file, with no diagnostic. Two safe patterns:
> - **Preferred:** write the `.ps1` using ASCII characters only. Use plain hyphens `-`, straight double quotes `"`, standard punctuation. Semantically identical, no encoding risk.
> - **Alternative:** if you must include non-ASCII characters, write the file with a UTF-8 BOM (first three bytes `EF BB BF`). Windows PowerShell 5.1 respects the BOM and reads as UTF-8.
>
> If the user hits those specific PowerShell parse errors when running the wrapper, the file was written UTF-8 without BOM. Rewrite with ASCII-only content and the errors disappear.

---

**Detailed Task Scheduler XML encoding note** — applies to ALL Task Scheduler XML files below (B.3.a, B.3.b, and Path C):

> Task Scheduler is **very** picky about file encoding. It requires the XML to be saved as **UTF-16 LE with a BOM** (byte-order mark). If you save it as UTF-8 (the default of most editors and Python's default `open()`), Task Scheduler rejects the import with `ERROR: unable to switch the encoding` at import time — the XML declaration says `encoding="UTF-16"` but the byte stream doesn't match, and Task Scheduler refuses to guess.
>
> **When Claude writes the XML file, it MUST use this exact pattern:**
> ```python
> path = r"C:\Users\...\Daily Digest Popup.xml"
> with open(path, "wb") as f:
>     f.write(b"\xff\xfe")                    # UTF-16 LE BOM — mandatory
>     f.write(xml_body.encode("utf-16-le"))   # body as UTF-16 LE — mandatory
> ```
> Do NOT use `open(path, "w", encoding="utf-16")` — Python's `utf-16` codec writes an extra BOM that confuses Task Scheduler. Use `utf-16-le` explicitly and write the BOM manually as shown.
>
> **How to verify the file is right:** the first two bytes should be `FF FE`, and every character in the file should be followed by a null byte (`00`). If you `cat` the file and see plain ASCII, it's UTF-8 — regenerate.

**Branch on Q7b's answer** — the XML differs materially between the two paths.

---

#### B.3.a — For Q7b answer (a): consistent routine → fixed-time trigger

Create a file in the same folder named `Daily Digest Popup.xml` with this content (adapt the `<StartBoundary>` time, the `<DaysOfWeek>` list, and the `[username]` in the file path). Note `<StartWhenAvailable>true</StartWhenAvailable>` — that's the "Run task as soon as possible after a scheduled start is missed" setting; it's what makes the trigger robust to the laptop being off at fire time.

```xml
<?xml version="1.0" encoding="UTF-16"?>
<Task version="1.4" xmlns="http://schemas.microsoft.com/windows/2004/02/mit/task">
  <RegistrationInfo>
    <Description>Open Daily Digest HTML in browser at [chosen time] [days].</Description>
  </RegistrationInfo>
  <Triggers>
    <CalendarTrigger>
      <StartBoundary>[YYYY-MM-DD]T[HH:MM]:00</StartBoundary>
      <Enabled>true</Enabled>
      <ScheduleByWeek>
        <DaysOfWeek>
          [Insert <Monday/>, <Tuesday/>, etc. for each chosen day]
        </DaysOfWeek>
        <WeeksInterval>1</WeeksInterval>
      </ScheduleByWeek>
    </CalendarTrigger>
  </Triggers>
  <Principals>
    <Principal id="Author">
      <LogonType>InteractiveToken</LogonType>
      <RunLevel>LeastPrivilege</RunLevel>
    </Principal>
  </Principals>
  <Settings>
    <MultipleInstancesPolicy>IgnoreNew</MultipleInstancesPolicy>
    <DisallowStartIfOnBatteries>false</DisallowStartIfOnBatteries>
    <StopIfGoingOnBatteries>false</StopIfGoingOnBatteries>
    <AllowHardTerminate>true</AllowHardTerminate>
    <StartWhenAvailable>true</StartWhenAvailable>
    <AllowStartOnDemand>true</AllowStartOnDemand>
    <Enabled>true</Enabled>
    <ExecutionTimeLimit>PT5M</ExecutionTimeLimit>
    <Priority>7</Priority>
  </Settings>
  <Actions Context="Author">
    <Exec>
      <Command>cmd.exe</Command>
      <Arguments>/c start "" "C:\Users\[username]\Documents\Daily Digest\Daily Digest.html"</Arguments>
    </Exec>
  </Actions>
</Task>
```

**Why `cmd.exe /c start ""`?** For `.html` files, Windows's file association reliably points to whichever browser the user has set as their default — Chrome, Edge, Firefox, Brave, Opera, whatever. Unlike `.ics` in Path C (where New Outlook hijacks the association), `.html` associations are stable and honoured. So the simple `start` command works for every browser without needing to know the browser's install path or writing any PowerShell.

**This variant does NOT need a PowerShell wrapper.** No `.ps1` file, no execution-policy prompts, no user consent needed. The wrapper only shows up in B.3.b (the log-on variant), where the freshness check and daily sentinel logic can't be expressed in a bare `cmd` line.

Substitute `[username]` for the actual Windows username in the file path. That's the only per-machine configuration.

**Time-of-day suggestion:** set the CalendarTrigger's `<StartBoundary>` to digest-time + 30 minutes. So if Q6 said "digest runs at 08:45", set the Windows trigger to 09:15. This gives Cowork's cloud task headroom for a slow inbox run.

**Skip to B.4** for the walk-through of importing the XML.

---

#### B.3.b — For Q7b answer (b), (c), or (d): variable routine → log-on trigger + freshness check

Two files this time: a **PowerShell wrapper** that decides whether to open the digest, and a **Task Scheduler XML** that fires the wrapper on log-on. Unlike B.3.a (fixed-time), this variant *needs* PowerShell because we need conditional logic — "only open the digest if today's file is present AND it hasn't been opened yet today" — which a bare `cmd` line can't express.

**Before installing this wrapper, explain in plain language what you're doing** — don't dwell on the "PowerShell script" framing, just describe the outcome. Say to them, verbatim:

> *"I'll save a small helper file in your Daily Digest folder. When the Task Scheduler task fires, this helper does three things:*
> *1. Check whether today's daily digest file has been written yet (Cowork writes it during the morning cloud run).*
> *2. If it's there and hasn't been opened yet today, open it in your default browser.*
> *3. Write a small "opened" marker so it doesn't pop up again if you lock/unlock your machine later.*
>
> *You can open the helper file in Notepad any time — it's plain text, about 30 lines. Windows may show a one-time permission prompt the first time it runs. If your organisation blocks helper scripts entirely, we can switch you to the fixed-time trigger (B.3.a) instead — that variant doesn't use a helper file at all."*

Only proceed after the user acknowledges. If they'd rather not have Windows run any helper scripts, switch to the fixed-time variant (B.3.a) instead.

**File 1** — save as `Open Daily Digest.ps1` in `[user-chosen-folder]`:

```powershell
# Open Daily Digest.ps1
# Opens today's digest in the default browser, if fresh. Silent if stale, missing, or already opened today.

$HtmlPath = "C:\Users\[username]\Documents\Daily Digest\Daily Digest.html"
$LogPath = Join-Path $env:TEMP "daily-digest-open.log"

function Write-Log([string]$m) {
    $ts = (Get-Date).ToString("yyyy-MM-dd HH:mm:ss")
    Add-Content -Path $LogPath -Value "$ts  $m"
}

Write-Log "===== Trigger fired ====="

if (-not (Test-Path $HtmlPath)) {
    Write-Log "No digest file at $HtmlPath — silent exit."
    exit 0
}

$File = Get-Item $HtmlPath
$FreshnessCutoff = (Get-Date).Date  # today at midnight
if ($File.LastWriteTime -lt $FreshnessCutoff) {
    Write-Log ("Digest is stale (LastWriteTime " + $File.LastWriteTime + " < today " + $FreshnessCutoff + ") — silent exit.")
    exit 0
}

# Sentinel: has today's digest already been opened by this trigger?
$SentinelPath = Join-Path $env:LOCALAPPDATA "daily-digest-opened-$(Get-Date -Format yyyy-MM-dd).flag"
if (Test-Path $SentinelPath) {
    Write-Log "Already opened today (sentinel exists) — silent exit."
    exit 0
}

# Open it in the default browser and mark opened
Start-Process $HtmlPath  # Windows dispatches to the default browser — no need to know which one
New-Item -Path $SentinelPath -ItemType File -Force | Out-Null
Write-Log "Opened $HtmlPath — sentinel written."
```

**Why the sentinel?** The log-on trigger can fire multiple times a day (log-on, unlock, reconnect after sleep). The sentinel file — one per calendar day — prevents the browser popping up every time the user unlocks their machine. Once opened for the day, subsequent triggers see the sentinel and silent-exit.

**File 2** — save as `Daily Digest Popup.xml` in `[user-chosen-folder]`:

```xml
<?xml version="1.0" encoding="UTF-16"?>
<Task version="1.4" xmlns="http://schemas.microsoft.com/windows/2004/02/mit/task">
  <RegistrationInfo>
    <Description>Open Daily Digest HTML on log-on or unlock, if today's digest is present.</Description>
  </RegistrationInfo>
  <Triggers>
    <LogonTrigger>
      <Enabled>true</Enabled>
    </LogonTrigger>
    <SessionStateChangeTrigger>
      <Enabled>true</Enabled>
      <StateChange>SessionUnlock</StateChange>
    </SessionStateChangeTrigger>
  </Triggers>
  <Principals>
    <Principal id="Author">
      <LogonType>InteractiveToken</LogonType>
      <RunLevel>LeastPrivilege</RunLevel>
    </Principal>
  </Principals>
  <Settings>
    <MultipleInstancesPolicy>IgnoreNew</MultipleInstancesPolicy>
    <DisallowStartIfOnBatteries>false</DisallowStartIfOnBatteries>
    <StopIfGoingOnBatteries>false</StopIfGoingOnBatteries>
    <AllowHardTerminate>true</AllowHardTerminate>
    <StartWhenAvailable>true</StartWhenAvailable>
    <AllowStartOnDemand>true</AllowStartOnDemand>
    <Enabled>true</Enabled>
    <ExecutionTimeLimit>PT5M</ExecutionTimeLimit>
    <Priority>7</Priority>
  </Settings>
  <Actions Context="Author">
    <Exec>
      <Command>powershell.exe</Command>
      <Arguments>-NoProfile -ExecutionPolicy Bypass -File "[user-chosen-folder]\Open Daily Digest.ps1"</Arguments>
    </Exec>
  </Actions>
</Task>
```

**What this does:**
- Fires on log-on (first sign-in of the day) **and** on session unlock (Windows returning from lock screen), covering both "I closed my laptop overnight" and "I locked it for lunch".
- The PS1 wrapper decides whether to actually open the browser: only if the digest file exists AND was written today AND hasn't been opened yet today.
- If the user opens the laptop at 07:00 (before Cowork ran) → no file yet, silent exit. When they next unlock at 09:30, file is present, sentinel absent, browser opens. Later unlocks that day → sentinel exists, silent.

**Alternative:** If the user prefers "log-on only, no unlock trigger" (some people find the unlock trigger too eager), drop the `<SessionStateChangeTrigger>` block. Log-on only means the digest opens once per Windows sign-in — which for many people is effectively "once per day" already.

### B.4: Walk the user through importing the XML — step by step

This is where non-technical users are most likely to get lost. Be patient and very specific. Pause after each step and ask the user to confirm they've completed it before moving to the next.

**Step 1.** Press `Win + R` on the keyboard. A small "Run" dialog appears. Type `taskschd.msc` and press Enter. The Task Scheduler window opens.

**Step 2 — critical: click Task Scheduler Library, not the parent.** In the left-hand pane, you'll see "Task Scheduler (Local)" at the top with a small triangle/chevron `▷` next to it. **You must click that chevron to expand the tree**, then **click "Task Scheduler Library"** (the child folder that appears below). Do NOT leave "Task Scheduler (Local)" selected — that shows the summary overview and hides the actual task list, making Step 6 impossible to complete. If done right, the middle pane switches from an "Overview of Task Scheduler" summary to a scrolling list of tasks.

**Step 3.** Look at the **RIGHT-hand pane**, headed **"Actions"**. In that list, "Import Task..." is roughly the **4th item** from the top, just below "Create Task…". Click it — a file-picker opens.

**Step 4.** Browse to `[user-chosen-folder]` and select `Daily Digest Popup.xml`. Click Open. A Properties dialog appears, pre-filled with the task settings.

**Step 5.** In the Properties dialog, do not change anything. Just click **OK** at the bottom.

**Step 6 — verify the task landed.** Look in the middle pane. You should see a list of tasks, sorted alphabetically by Name. The list contains all the user's user-space tasks (system tasks live in sub-folders under `\Microsoft\`, not here). Scroll or press PgDn to find **"Daily Digest Popup"**. The Triggers column should show a summary of when the task will fire.

**If the middle pane still shows the "Overview of Task Scheduler" summary (with "Task Status" and "Active Tasks" sections)** — you're on "Task Scheduler (Local)" (the parent), not "Task Scheduler Library" (the child). Go back to Step 2, expand the tree, and click **"Task Scheduler Library"** — the middle pane will then flip to the task-list view.

**If you can't find the task by name:** press F5 to refresh, or click the "Name" column header to force an alphabetical sort. If it's still missing, the import may have quietly filed it in a sub-folder — expand `\Microsoft\` in the left pane and browse for it. Rare but possible.

**Step 7.** **Test it now.** Right-click the **"Daily Digest Popup"** task in the list → click **Run**. Within a second, a browser tab should pop open showing your Daily Digest file. If you see the digest, the setup works.

**If the user gets stuck on any step:** ask them to take a screenshot of the Task Scheduler window (the whole window — Windows shortcut `Win + Shift + S` for snip, then save or paste) and share it back in the Cowork chat. With the screenshot you can see exactly which panel they're looking at, whether the task was actually imported, what error message (if any) is shown, and what to suggest next. This is significantly more efficient than back-and-forth text descriptions of buttons.

**Common problems and fixes:**
- *"Nothing happens when I click Run"* — most likely the path to the browser's .exe in the XML is wrong. Right-click the task → Properties → Actions tab → check the Program/script field. Update if needed.
- *"A browser error page appears"* — the file path in the Arguments field is wrong. Same place, check it.
- *"The task says Last Run Result: 0x80070001 or similar error code"* — usually a permissions or path issue. Right-click → Properties → General tab → check "Run only when user is logged on" is selected.
- *"I can't find Import Task in the Actions panel"* — make sure "Task Scheduler Library" is selected in the left pane (not "Task Scheduler (Local)" above it). The Actions panel options change depending on what's selected.

### B.5: Set up the file the FIRST time

The scheduled Cowork task only fires at the chosen time; until then there's no file yet. Have the user manually run the Cowork scheduled task once now (Scheduled section → "Run now") so the initial `Daily Digest.html` exists. Then test B.4 step 7.

### B.6: One nuance to flag

If the browser tab pops up while the user is in the middle of typing or focused on another app, it grabs focus (annoying). Mitigation: most browsers accept an `--app=file:///...` flag to open the page in a frameless app-style window which behaves differently. Or accept the focus steal as a feature ("the digest demands your attention, that's the point"). Ask the user which they prefer.

## Path C — Outlook Classic .ics import

Same shape as Path B, but the file is a `.ics` and the opener is Outlook Classic.

### C.1: Confirm Outlook Classic is installed

Ask: *"Are you using classic Outlook or new Outlook? If you don't know — open Outlook. If the title bar at the top says 'Outlook' with no further qualifier, it's classic; if it says 'New Outlook' or has a 'New Outlook' toggle, it's new Outlook. If new Outlook is the only one installed, this method won't work; we'll need to switch to Path A or B."*

### C.2 — C.5: same structure as B, but

- The Cowork scheduled task writes a `.ics` file using the iCalendar (RFC 5545) format. Set **two** iCalendar properties on the event, both mandatory for privacy and Free/Busy:
  - `TRANSP:TRANSPARENT` — marks the event as **Free** rather than Busy (bonus: solves the Busy/Free issue from Path A).
  - `CLASS:PRIVATE` — marks the event as **Private sensitivity** in Outlook. This is important: without it the event defaults to Normal/Public, meaning anyone who has delegate access, "reviewer" access, or "free/busy details" access to the user's calendar (a manager, an assistant, a shared-mailbox reader) can see the *entire digest text* including inbox/Teams to-dos. With `CLASS:PRIVATE`, Outlook hides the subject, body, and location from anyone other than the calendar owner — even users who can otherwise see the calendar just see "Private Appointment" for that block. This matches the `visibility: "private"` setting used in Path A.

- **The digest content goes into TWO fields on the event, not just one.** Both are mandatory:
  - `DESCRIPTION:` — plain-text fallback for clients that don't render HTML (Apple Calendar, etc.). Format: verbatim digest content with `\n` for line breaks and RFC 5545 escaping (comma → `\,`, semicolon → `\;`, backslash → `\\`).
  - `X-ALT-DESC;FMTTYPE=text/html:` — HTML alternative. Outlook Classic honours this when the `.ics` is opened via `OUTLOOK.EXE /ical`, and renders the meeting-form body with the full formatting (bold section headers, bold + italic inline emphasis, proper section spacing). **Verified: identical rendering to what Google Calendar produces for Path A.**
  - Use the **same Outlook-tested HTML template** as Path A (section A.4 below) for the `X-ALT-DESC` value. One line of HTML, no source newlines, `<table>` with `<tr><td>` per bullet and `height="14"` spacer rows between sections. Reuse it verbatim — no separate Path C template needed.
  - When emitting the two fields, apply RFC 5545 escaping to both the plain-text and HTML content. In particular the HTML block will contain many commas (inside style attributes, CSS values, natural prose in bullets) — every one of them must become `\,` in the `.ics` line, otherwise the parser truncates the field at the first unescaped comma and Outlook falls back to plain-text DESCRIPTION.
  - **Do not fold the X-ALT-DESC line** per RFC 5545's 75-char line-folding rule unless you're confident the folded output survives round-trip. Modern parsers (Outlook Classic 2019+, Microsoft 365) accept unfolded long lines without issue, so keep the entire value on a single line for reliability.

  Reference implementation (Python) — reuses `html_body` variable which is the same template from Path A:
  ```python
  def ical_esc(s):
      return s.replace("\\", "\\\\").replace(",", "\\,").replace(";", "\\;").replace("\n", "\\n")
  ics_lines = [
      # ... other VEVENT properties ...
      "CLASS:PRIVATE",
      "TRANSP:TRANSPARENT",
      f"DESCRIPTION:{ical_esc(plain_text_digest)}",
      f"X-ALT-DESC;FMTTYPE=text/html:{ical_esc(html_body_one_line)}",
  ]
  ```

**⚠️ Critical: do NOT use `cmd.exe /c start "" "…ics"` to open the file.** That relies on the Windows file association for `.ics`. On modern Windows installations, **New Outlook** (the rewritten Microsoft 365 app) grabs the `.ics` association by default and either fails silently or opens a preview that doesn't behave like Outlook Classic's meeting form. Even after installing/reinstalling Outlook Classic, Windows may still route `.ics` to New Outlook or to the Windows Calendar UWP app.

**The reliable pattern is to call OUTLOOK.EXE directly with the `/ical` switch:**

**Two-tier approach — try the hardcoded path first, fall back to PowerShell only if needed.** Path C needs OUTLOOK.EXE's full path in the Task Scheduler action. That path is the same on the vast majority of modern Windows machines with Microsoft 365 or Office 2019/2021/2024 installed, so start with the hardcoded assumption. Only fall back to a self-detecting PowerShell wrapper if the hardcoded path turns out not to match on this particular user's machine.

**Tier 1 (default) — hardcoded OUTLOOK.EXE path in the XML.** Simple, no scripting, no additional files, no PowerShell execution-policy prompts. Use this pattern unless it doesn't work.

**Before you use it, verify the path exists** — ask the user: *"Before I set this up, one quick check. Open File Explorer, paste `C:\Program Files\Microsoft Office\root\Office16` into the address bar (top of the window) and press Enter. Do you see a file called `OUTLOOK.EXE` in that folder?"*

- **If yes** → proceed with the Tier 1 XML (below).
- **If no** → try these alternative folders one at a time (same File Explorer probe): `C:\Program Files (x86)\Microsoft Office\root\Office16`, then `C:\Program Files\Microsoft Office\Office16`, then `C:\Program Files (x86)\Microsoft Office\Office16`. Substitute the folder that contains OUTLOOK.EXE into the `<Command>` field.
- **If none of these have it** → skip to Tier 2 (PowerShell wrapper).

**Tier 1 XML — Task Scheduler `<Exec>` block for the fixed-time variant (Q7b answer (a)):**
```xml
<Exec>
  <Command>C:\Program Files\Microsoft Office\root\Office16\OUTLOOK.EXE</Command>
  <Arguments>/ical "C:\Users\[username]\Documents\Daily Digest\Daily Digest.ics"</Arguments>
</Exec>
```
The `/ical` switch is mandatory — without it, Outlook opens with "The command line argument is not valid. Verify the switch you are using." (See earlier debugging notes.)

**For Q7b answers (b), (c), (d) — log-on / unlock triggers — you still need a PowerShell wrapper for the freshness-check and sentinel logic** (same reason as Path B.3.b), so hardcoding OUTLOOK.EXE in the XML doesn't apply there. Use the Tier 2 wrapper below for the log-on variant regardless of whether Tier 1 worked for fixed-time.

---

**Tier 2 (fallback) — self-detecting PowerShell wrapper.** Use this when the hardcoded path in Tier 1 didn't match, OR for the log-on/unlock trigger variants (which need PowerShell anyway).

**Before installing the wrapper, explain to the user in plain language what you're doing** — no need to dwell on the "PowerShell script" framing, just describe the outcome. Say to them, verbatim:

> *"I'll set this up so I can find where Outlook Classic lives on your PC automatically, rather than making you look. I'll save a small helper file in your Daily Digest folder that does two things when the Task Scheduler task fires:*
> *1. Locate Outlook Classic on your PC.*
> *2. Open today's daily digest with it.*
>
> *You can open the helper file in Notepad any time — it's plain text, about 30 lines. Windows may show a one-time permission prompt the first time it runs. If your organisation blocks helper scripts entirely, we can fall back to the hardcoded Outlook path from Tier 1 above."*

Only proceed after the user acknowledges. If they say they'd rather not have Windows run any helper scripts, stick with Tier 1 and let the user manually update the hardcoded path if Outlook ever moves.

**Tier 2 Task Scheduler `<Exec>` block** — same for fixed-time (fallback) and log-on variants:
```xml
<Exec>
  <Command>powershell.exe</Command>
  <Arguments>-NoProfile -ExecutionPolicy Bypass -File "C:\Users\[username]\Documents\Daily Digest\Open Daily Digest ICS.ps1"</Arguments>
</Exec>
```

**Save `Open Daily Digest ICS.ps1` in the Daily Digest folder** — auto-detects OUTLOOK.EXE via the `App Paths` registry key, falls back to the common file-system paths, errors only if no Outlook Classic install can be found:

  ```powershell
  # Open Daily Digest ICS.ps1 — Path C wrapper.
  # Auto-detects OUTLOOK.EXE and calls it with /ical on today's .ics.

  $IcsPath = "C:\Users\[username]\Documents\Daily Digest\Daily Digest.ics"
  $LogPath = Join-Path $env:TEMP "daily-digest-ics.log"

  function Write-Log([string]$m) {
      $ts = (Get-Date).ToString("yyyy-MM-dd HH:mm:ss")
      Add-Content -Path $LogPath -Value "$ts  $m"
  }

  Write-Log "===== Trigger fired ====="

  if (-not (Test-Path $IcsPath)) {
      Write-Log "No ICS file at $IcsPath — silent exit."
      exit 0
  }

  # (For log-on variant: freshness + sentinel checks belong here, same shape as B.3.b's Open Daily Digest.ps1)

  # Locate OUTLOOK.EXE. Preferred: App Paths registry key.
  $OutlookPath = $null
  try {
      $key = Get-ItemProperty "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\App Paths\OUTLOOK.EXE" -ErrorAction Stop
      $OutlookPath = $key.'(default)'
  } catch { }

  if (-not $OutlookPath -or -not (Test-Path $OutlookPath)) {
      # Fall back to well-known install paths.
      foreach ($candidate in @(
          "C:\Program Files\Microsoft Office\root\Office16\OUTLOOK.EXE",
          "C:\Program Files (x86)\Microsoft Office\root\Office16\OUTLOOK.EXE",
          "C:\Program Files\Microsoft Office\Office16\OUTLOOK.EXE",
          "C:\Program Files (x86)\Microsoft Office\Office16\OUTLOOK.EXE",
          "C:\Program Files\Microsoft Office\Office15\OUTLOOK.EXE",
          "C:\Program Files (x86)\Microsoft Office\Office15\OUTLOOK.EXE"
      )) {
          if (Test-Path $candidate) { $OutlookPath = $candidate; break }
      }
  }

  if (-not $OutlookPath) {
      Write-Log "OUTLOOK.EXE not found in registry or known install locations — Outlook Classic may not be installed."
      exit 1
  }

  Write-Log "Using OUTLOOK.EXE at $OutlookPath"
  & $OutlookPath /ical $IcsPath
  Write-Log "Launched — waiting for Outlook to display the meeting form."
  ```

- **Two failure modes worth knowing about, for debugging** (both are historical — the wrapper design above avoids them by construction, but this documents *why* the design is this shape):
  - **Nothing happens (command window flashes and closes)** — the XML was using the `cmd.exe /c start "" ".ics"` pattern, which routes via file association. New Outlook now grabs the .ics association by default and silently rejects the file. Fix: use the wrapper pattern above, which calls OUTLOOK.EXE directly.
  - **Outlook Classic opens with dialog "The command line argument is not valid. Verify the switch you are using."** — OUTLOOK.EXE was called without the `/ical` switch. Fix: the wrapper above always uses `/ical` — do not remove it.

- **When the wrapper is used (Tier 2), it doubles for both trigger variants.** For Q7b answer (a) with Tier 2 (fallback because Tier 1 hardcoded path didn't match), the Task Scheduler XML has a fixed-time CalendarTrigger firing the wrapper. For Q7b answers (b), (c), (d) — where the wrapper is used regardless — the XML has a LogonTrigger + SessionStateChange trigger firing the wrapper, and the wrapper (identical shape to B.3.b's Open Daily Digest.ps1) also does the freshness check and one-per-day sentinel. One file, both use cases.
- When either variant fires, Outlook Classic opens a meeting form. The user must click **"Save & Close"** to actually add it to the calendar. (This is the click-each-morning friction unique to Path C.)

# PHASE 5 — Test the full chain

Once everything is built, do a manual end-to-end test:

1. **For Path A:** Trigger the Cowork scheduled task manually. Wait one minute. Check Outlook inbox + calendar.
2. **For Path B:** Trigger the Cowork scheduled task manually. Wait one minute. Open the HTML file in a browser — does it look right? Then trigger the Windows Task Scheduler job manually — does a browser tab pop open with the digest?
3. **For Path C:** Trigger the Cowork scheduled task manually. Wait one minute. Open the .ics file by double-clicking — does Outlook Classic open the meeting form correctly?

If anything fails, debug with the user. Common issues:
- **Scheduled task didn't fire** → in Cowork's Scheduled section, check "Last run" status. If it shows an error, the prompt is malformed.
- **File didn't get written** → folder permission issue. Have user check that Cowork has write access to the chosen folder.
- **Email/calendar invite didn't arrive (Path A)** → IT department may be filtering external invites. Check junk mail / external invites folder.
- **Windows Task Scheduler runs but nothing visible (Path B)** → the `cmd.exe /c start` file-association route failed. Double-click the .html file directly in File Explorer — if it opens in your default browser, the association is fine and the issue is in the XML Arguments field (check the file path). If it doesn't open in your default browser, the .html file association itself is broken (rare); reset it in Settings → Default apps.
- **Windows Task Scheduler runs but nothing visible (Path C)** → most likely OUTLOOK.EXE path in the XML doesn't match this machine (Tier 1 XML hardcoded the wrong path). Switch to Tier 2 (self-detecting wrapper) or update the `<Command>` in the XML to the correct OUTLOOK.EXE path.

# PHASE 6 — Wrap up

Summarise for the user what's now in place. Present the skill itself as a `.skill` installable, not as a raw `.md` path — much easier UX:

- **Skill** — present the `.skill` file via the `present_files` tool so it renders as a one-click "Save skill" install card in Cowork. Mention the underlying `SKILL.md` path for users who want to edit it by hand later.
- **Scheduled task** in Cowork: runs at `[time] [days]` (and on-blackout-dates if the user mentioned any).
- **Delivery method**: [A / B / C].
- **Daily one-off the user does manually** (if Path A): change Busy → Free on the event each morning (5 seconds).
- **Daily one-off the user does manually** (if Path C): click Save & Close on the Outlook meeting form each morning.

Tell the user:
- *"To change anything later — what's scanned, what topics, what time — just open a new Cowork chat and say 'update my daily digest'. I'll know what to do."*
- *"To pause the digest temporarily — go to Cowork's Scheduled section, find the daily-digest task, toggle Enabled off. Re-enable it when you want it back."*
- *"To stop entirely — same place, delete the task."*
- *"If a morning run doesn't fire — open Cowork, re-attach the Setup Guide, say 'my digest didn't fire today, help me debug'."*

Confirm that the user is happy with the setup. Offer to also produce a one-page printable cheat-sheet of the setup details (file paths, schedule times) if they'd find it useful.

---

# Appendix — quick reference for Claude

**Tools you'll need (load via ToolSearch if deferred):**
- `mcp__474428d3-…__outlook_email_search` — for inbox/folder scans
- `mcp__474428d3-…__outlook_calendar_search` — for the calendar pull
- `mcp__474428d3-…__chat_message_search` — for Teams
- `mcp__474428d3-…__read_resource` — for full email bodies + folder listing
- `mcp__7038610b-…__create_event` — Google Calendar (Path A)
- `WebSearch`, `mcp__workspace__web_fetch` — for news
- `mcp__scheduled-tasks__create_scheduled_task` — to wire up the cron
- `Write` — to save the SKILL.md and the XML/HTML/ICS deliverables
- `mcp__cowork__request_cowork_directory` — to gain access to the user's target folder if not already mounted

**Known limitations to surface to the user when relevant:**
- Microsoft 365 connector is **read-only** for Outlook (no write to calendar, no send_mail, no Tasks API). That's why we need the Google Calendar / local-file workarounds.
- Google Calendar's MCP doesn't expose `transparency` (Busy/Free). User clicks Busy → Free manually until that changes.
- WebFetch returns empty content on JavaScript-rendered pages (e.g. most modern government sites). Fall back to WebSearch for content from those.
- Some PDFs (typical of government-portal documents) can't be deep-fetched via WebFetch *or* Claude in Chrome — they open in extension PDF viewers that the text-extraction tool refuses to read. WebSearch snippets are the only path for these.
- The Outlook connector batch size: `limit: 50` overflows context; use `limit: 25` per batch and paginate.
- `folderName` is incompatible with `sender` + `query` — the connector rejects the combination. Drop `folderName` when using `sender` + `query`.

**The one-line Claude prompt the user will use after attaching this file:**

> *"Read this guide and follow it — set me up with a daily digest."*

Or any variation. Recognise the intent.

**Maintenance notes (for future editors of this guide):**

- **Browser question (B.1b)** currently lives inside Path B only. If a future revision changes the default delivery method or merges paths, move the browser question to a more prominent place — orphaning it in Path B was fine as long as Path B is one of three peer options, but if Path B becomes the default-recommended path, the browser question should probably move to Phase 3 alongside the time-of-day question.
- **Claude in Chrome fallback** for JavaScript-heavy pages is mentioned but not relied on as the primary path. If `WebFetch` ever gains JS-rendering support, the section about Claude in Chrome can be simplified or removed.
- **Microsoft 365 connector capabilities** are tracked at the Anthropic connector registry — if it ever gains write tools (create_event, send_mail, Tasks API), the entire "delivery workaround" complexity in Phase 4 can be retired in favour of a direct Outlook event creation, and Path A (Google bridge) can be deprecated. Re-check the registry periodically.