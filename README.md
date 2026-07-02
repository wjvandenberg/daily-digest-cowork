# Daily Digest — a morning briefing built with Claude in Cowork

A one-page AI briefing on your desk every weekday morning. Modular: you choose what's in it (news, calendar, to-dos), and how it lands on your screen (a Google Calendar invite, a browser tab that pops open, an Outlook meeting form). Built once with Claude in Cowork, then it runs itself.

If you got here from my [LinkedIn post](https://www.linkedin.com/in/wjvandenberg/) — welcome. The three files below are everything you need.

---

## What's in this repo

| File | What it is |
|---|---|
| [`Daily Digest — Setup Guide.md`](./Daily%20Digest%20—%20Setup%20Guide.md) | The guide Claude reads. You attach this to a Cowork chat and Claude walks you through several phases of questions and build steps. |
| [`Daily Digest — Quick Start.pdf`](./Daily%20Digest%20—%20Quick%20Start.pdf) | A one-page printable cheat-sheet for humans. Read this in 60 seconds. |
| [`Daily Digest — LinkedIn Carousel.pdf`](./Daily%20Digest%20—%20LinkedIn%20Carousel.pdf) | The 5-slide LinkedIn carousel for sharing. |

---

## What you get

Every weekday morning, Claude prepares a short briefing for you. You choose the scope when you set it up — the digest is **modular**:

- **News only** — headlines on whichever topics, sectors and regions you care about. A simple data-gathering exercise across public web sources. **No connector needed.** The simplest version — what most people start with.
- **News + today's calendar** — add a quick line-by-line view of your meetings. **Requires the Microsoft 365 Connector in Cowork.**
- **News + calendar + a to-do list from your inbox / Teams** — the most powerful version. **Requires the Microsoft 365 Connector in Cowork** (which your organisation may or may not allow).

The skill also recognises the way you actually tag work in Outlook — categories (colour labels), follow-up flags, and folders you use as separate work-piles all map naturally to sections of the digest. If you process a steady stream of similar items (applications, support tickets, casework) rather than discrete projects, the skill adapts to that pattern instead.

---

## How to install

About 5–10 minutes depending on scope. Mostly answering questions; Claude does the building.

1. **Download** [`Daily Digest — Setup Guide.md`](./Daily%20Digest%20—%20Setup%20Guide.md) and save it somewhere easy to find.
2. **Open a fresh Cowork chat.** Attach the `.md` file by drag-and-drop or the paperclip icon.
3. **Type:** *"Read this guide and follow it — set me up with a daily digest."*
4. **Answer Claude's questions** — scope first, then topics, then schedule, then delivery method. Each phase is short.
5. **Claude builds the skill** as a one-click installable `.skill` file, schedules the task, and walks you through a brief Windows-side setup if you chose the local-file delivery method. Test once. Done.

---

## Before you start

For a **news-only** digest you don't need anyone's permission — the digest only touches public web content.

For the **calendar** or **to-do** options, you'll be giving Claude read-access to your work email, chats and/or calendar via the **Microsoft 365 connector in Cowork**. If it isn't already installed, Claude will walk you through connecting it. **Check with your IT department or supervisor first** if your organisation has policies on AI tools reading internal data — regulated sectors (legal, finance, healthcare, government, defence) commonly restrict this. A 30-second check now can save a much bigger conversation later.

---

## The catch — and three delivery workarounds

The interesting design problem of this skill is not the skill itself (Claude is good at reading mail and summarising). It's getting the result in front of you, automatically, every morning, without you having to open Claude.

I expected Outlook to be the obvious channel. But **Microsoft 365's official connector for Claude is read-only**. The AI can read your inbox and calendar, but can't write to them — no `create_event`, no `send_mail`, no Tasks API, no Outlook write of any kind. So the scheduled task has to deliver elsewhere.

Three workarounds do work:

### 1. Google Calendar invite into Outlook

Cowork creates a Google Calendar event with the digest in the meeting body, and invites your work email as an external attendee. Outlook automatically adds it as a tentative meeting. The digest content routes through your personal Google account.

**Cleanest if your digest is news-only.** For inbox/Teams content (potentially sensitive), check whether your organisation allows routing work content through a personal Google account — many firms in regulated sectors don't.

### 2. Local file + Windows browser popup

Cowork writes the digest as an HTML file on your computer. Windows Task Scheduler — a built-in Windows tool — opens that file in your browser at the time you choose. Fully local, nothing leaves your machine.

Most secure option for sensitive content. Requires a one-time five-minute Windows Task Scheduler setup, walked through step-by-step by Claude during install.

### 3. Outlook Classic .ics import

Cowork writes a calendar (`.ics`) file. Windows Task Scheduler opens it in Outlook Classic at the chosen time, which prompts you to click Save to add to your calendar.

**Caveat:** does **not** work in new Outlook (the rewritten app Microsoft is pushing to all users). If your firm is migrating to new Outlook, this method will stop working. Also requires a click each morning, unlike option 1 which is automatic.

---

## "Why not Outlook Tasks or Microsoft To Do as a fourth option?"

Same architectural reason as the others: the Microsoft 365 Connector for Cowork is read-only across the board. Microsoft Graph the underlying API supports task creation, but the connector doesn't expose those endpoints. If that changes, Outlook Tasks / To Do would become a natural fourth delivery method — until then, the three above are the only programmatic delivery paths.

---

## After install

- **Change anything later** (different time, different topics, broader or narrower scope): open Cowork, say *"update my daily digest"*.
- **Pause temporarily:** Cowork → Scheduled section → find the daily-digest task → toggle Enabled off. Re-enable when you want it back.
- **Stop entirely:** same place, delete the task.
- **If a morning run doesn't fire:** open Cowork, re-attach the Setup Guide, say *"my digest didn't fire today, help me debug"*.

---

## The lesson behind the design

The most obvious approach — pushing something into Outlook — doesn't really work anymore. The cleanest delivery comes from going *outside* Outlook entirely. Whether that's a Google bridge or a local file or an .ics drop, the workaround is forced by the read-only nature of the connector, not by anything Outlook-specific.

What an MCP connector gives you depends on the vendor's choices, not on what the underlying product technically supports. That gap drives architecture decisions you wouldn't otherwise make. Worth knowing when you're building anything where AI needs to deliver back into a Microsoft surface.

---

## Iterations + acknowledgements

This skill went through several rounds of real failures before it stabilised. Three concrete examples:

- A "call them back" task listed as still open — the call had already happened the day before, but the tool that reads your inbox and chats can't see phone calls.
- A task listed as still open when the reply had arrived by email — the original ask was in Teams, the resolution in email, and the skill was only scanning one channel.
- A phantom task created by loose summarisation, conflating two unrelated threads into a combined item that didn't actually exist.

Each failure became a rule in the setup guide, baked in as a worked example so future Claude instances hit the anti-pattern before re-discovering it.

If you build one and find a new failure mode, I'd genuinely like to hear. Open an issue or reach me on [LinkedIn](https://www.linkedin.com/in/wjvandenberg/).

---

## License

MIT — do anything you want with the files. Attribution appreciated but not required.

---

*Built with Claude in Cowork by [Wouter van den Berg](https://www.linkedin.com/in/wjvandenberg/). Last updated July 2026.*
