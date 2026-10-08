---
name: 1-1-prep
description: Draft a one-page 1:1 prep for one of the user's direct reports (the roster lives in the team skill). Covers the last 7 days of their Linear work, GitHub PRs in the team's repos, Slack, their shared 1:1 doc, and the team's meeting notes. Use before any 1:1, or when asked to "prep me for my 1:1 with X", "what's going on with X", or "catch me up on X before we talk".
user-invocable: true
argument-hint: "<person>"
allowed-tools:
- Bash
- Read
- Write
- Edit
- AskUserQuestion
- ToolSearch
- mcp__claude_ai_Linear__list_issues
- mcp__claude_ai_Linear__list_projects
- mcp__claude_ai_Linear__get_status_updates
- mcp__claude_ai_Slack__slack_search_public_and_private
- mcp__claude_ai_Slack__slack_read_thread
---

# 1:1 Prep Drafter

Draft a one-page prep doc for a 1:1. Scan-in-90-seconds format: what changed in their world,
their topics, open threads, their Linear projects, and a short list of things worth raising.

Everyone in the roster is a direct report, and you run these 1:1s as their manager. The
prep focuses on their individual work, growth, and blockers.

**Read the team reference first:** `<base directory>/../team/SKILL.md`. It has the roster, repos,
meeting-notes docs, standing agendas, and how to use gws, Slack, and Linear.

## Invocation

```
/1-1-prep <person>
```

- **person** (required): name, first name, GitHub login, or email of someone in the team
  reference's roster

## Phase 0: Check gws

Check gws as the team reference says, before anything else. On an auth error, stop; don't
carry on without the docs.

## Phase 1: Resolve person and window

### Person

Match the argument against the team reference's roster: full name, first name, GitHub login,
or email. The roster carries everything later phases need: email (Linear and Slack lookups),
GitHub login, squad, and 1:1 doc.

If the person isn't in the roster, stop and say so. Don't go looking them up elsewhere; ask
the user to add them to the team reference.

### Window

Always the last 7 days, ending today. Nothing in the docs changes the window.

## Phase 2: Read docs

Export each doc as text (the team reference shows how), read it, then delete the export.
All of these docs are rolling: one dated entry per meeting, newest first.

### Shared 1:1 doc

From the person's 1:1 doc (its ID is in the roster), extract:
- Their topics for the upcoming 1:1 (an entry dated today or later, if there is one)
- Open action items from prior entries — **yours and theirs**, tracked separately

Empty action-item bullets can run into the next entry's date heading in the export. Don't
read the heading as an action item.

### Team meeting notes

Read the entries dated in the window from each meeting-notes doc that covers the person
(the team reference says which, and its Docs table has the IDs). Find each doc by ID, not by
name.

Lines are tagged by speaker (the notes tags in the team reference). Pull the lines tagged
with the person, action items naming them, and team-wide asks or decisions that affect their
work.

### Gemini notes

If the team's meetings have Gemini notes (the team reference gives their name filter),
find the ones in the window:

```bash
cd ~/Documents/manager && gws drive files list --params '{"q": "name contains '\''<name filter>'\'' and name contains '\''Notes by Gemini'\'' and mimeType = '\''application/vnd.google-apps.document'\'' and modifiedTime > '\''<start>T00:00:00Z'\''", "fields": "files(id,name,modifiedTime)", "pageSize": 20, "orderBy": "modifiedTime desc"}'
```

The `mimeType` filter matters: each file also appears as a shortcut in your Drive pointing
at the original. Filenames follow `<meeting name> - <YYYY/MM/DD> <time> <tz> - Notes by
Gemini`.

Export each and extract only what involves the person: commitments they made, questions they
raised, and callouts naming them.

## Phase 3: Gather activity signal

Run these in parallel, scoped to the window. Use the person's email and GitHub login from the
roster in the team reference.

### Linear

#### Issues

Make one `list_issues` call for every issue assigned to them. It accepts the email directly,
so there's no user lookup:

```
assignee: <email>
limit: 250
fields: ["id", "title", "url", "status", "statusType", "project", "completedAt", "updatedAt"]
```

Page with `cursor` while `hasNextPage` is true. Everything below comes from this one list:

- **Completed in window:** `completedAt` is in the window
- **Moved forward:** `statusType` is `started` and `updatedAt` is in the window
- **Stale:** `statusType` is `started` and `updatedAt` is more than 7 days ago

#### Projects

Group the issues by `project`. List every project with at least one open issue (any
`statusType` except `completed` or `canceled`). For each one, report `completed/total`,
leaving canceled issues out of both numbers, for example `3/4`.

To find projects they lead, call `list_projects` with `member: <email>` and
`fields: ["name", "lead", "status"]`. For each project they lead, check the latest status
update (`get_status_updates` with `type: "project"`, `project: <id>`, `limit: 1`). Don't put
updates in the project list. Use them for "Worth raising": a missing or stale update (older
than 14 days), an at-risk health, or a promised date that has passed.

### GitHub

Only look at the team repos in the team reference:

```bash
for repo in <team repos>; do
  gh search prs --repo="$repo" --author=<login> --updated=<start>..<end> --json title,number,url,state,closedAt --limit 100
  gh search prs --repo="$repo" --reviewed-by=<login> --updated=<start>..<end> --json title,number,url,author,state --limit 100
done
```

For open PRs that look stuck, check age and review state with
`gh pr view <n> --repo <repo> --json createdAt,isDraft,reviewDecision`.

### Slack

Follow the team reference's Slack rules: search by sender (`from:<username>`), never by
first name.

**Run two searches.** Results cap at 20 per page, and a busy day crowds out the rest of the
week. Since `after:` and `before:` are both exclusive:

```
# 1. Today
filters: from:<username> on:<today>

# 2. Rest of the window
filters: from:<username> after:<start - 1 day> before:<today>
```

Use `natural_language_query: "What has <person> been working on, shipping, or blocked on?"`
and `response_format: "concise"` for both.

For what others said about them, run a third search for the full name in quotes
(`keywords: ["\"<Full Name>\""]`, `filters: after:<start - 1 day>`), and check each hit is
the right person before using it. This also turns up their Linear project updates, if they
post to a channel (the team reference's Slack section).

Look for:
- **Wins** others called out — material to acknowledge
- **Blockers** they raised
- **Cross-team asks** involving them
- **Escalations** they handled
- **Disagreements** they were party to — often the most useful 1:1 material, handle with
  care

Results include surrounding thread context and resolved display names, so quote accurately
and link the permalink rather than paraphrasing.

## Phase 4: Synthesize

- **Last 7 days** — 2–3 bullets on what actually changed. Highlights, not an inventory.
- **Their topics** — from the 1:1 doc
- **Open threads** — unresolved follow-ups, split into *yours* and *theirs*
- **Linear projects** — one bullet per project from [Projects](#projects): name and issues
  done, nothing else
- **Worth raising** — short, each tied to evidence:
  - **Wins to acknowledge** — specific, with a link
  - **Concerns** — stale work, capacity, slipping dates, missed commitments
  - **Cross-team** — praise or friction from other teams
  - **Decisions needed** — things stuck pending one of you
- **Standing agenda** — from the team reference's standing agendas

## Phase 5: Draft

Write to `~/Documents/manager/1-1-prep_<first-last>_<YYYY-MM-DD>.md`, using today's date and
the person's name in lowercase with a hyphen (for example `jane-doe`). One page, hard
limit.

```markdown
# 1:1 Prep — <Person> — <today's date>
_window: <start> → <end>_

## Last 7 days
- <bullet>

## Their topics
- <from the 1:1 doc, or "nothing added since last 1:1">

## Open threads
**Mine:** <follow-up I owe them>
**Theirs:** <follow-up they owe me>

## Linear projects
- <project name>: 3/4 done

## Worth raising
- **Win:** <specific thing, with link>
- **Concern:** <specific thing, with link>
- **Decision needed:** <what's stuck, and on whom>

## Standing agenda
- <from the team reference's standing agendas, or omit>
```

### Writing guidelines

- **Terse.** A pre-meeting scan, not a script. 6–10 bullets total, plus the project list.
- **Evidence-linked.** Every "worth raising" bullet names the PR, issue, doc, or Slack
  thread. No generic impressions.
- **No padding.** Omit any section with nothing material.
- **Wins first, concerns specific.** Say what you're seeing, not a vibe.
- **Never lift material from a third party's 1:1 notes**, even if a shared doc references
  them.
- **Coach, don't grade.** Frame concerns as questions or support you can offer, not as a
  verdict on their performance.

## Phase 6: Output

> "Prep at `<path>`. ~90 sec read. Flag anything to drop or expand."

Don't draft talking points beyond the worth-raising bullets. The user runs the meeting.

## Standing agenda

Topics the user raises in every 1:1 with a person live in the team reference's Standing
agendas table. **Add to it whenever the user supplies a standing agenda item.**

## Key reminders

- **Window is always the last 7 days** — these 1:1s are weekly
- **Stop if gws isn't logged in** — the user fixes it, not the skill
- **Write the prep to `~/Documents/manager`**
- **Header date is today's date** — when the prep was generated
- People come only from the team reference — add new people there
- One page max. If it doesn't fit, cut.
