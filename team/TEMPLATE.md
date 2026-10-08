---
name: team
description: Reference data for the user's team, <team name>, and the shared conventions the other skills follow — who the user is, the roster of direct reports, where the team's work is tracked, repos, docs, the review cycle, the AI review bot, a glossary, standing 1:1 agendas, and how to use the tools. Read by the other skills so this lives in one place. Not a workflow; use it when a task needs to know who's on the team, where the team's work is tracked, or where a doc lives.
user-invocable: false
---

<!--
Copy this file to team/SKILL.md and fill it in. team/SKILL.md is gitignored: it holds
names, emails, doc IDs, and other company data that must stay private.

Delete any section the team doesn't use. The skills skip what isn't here.
-->

# Team: <team name>

Reference data and shared conventions, read by the other skills. The skills say "the user";
this file says who that is. **Edit this file, not the skills, when anything here changes:**
someone joins, leaves, changes squad, or is promoted; a doc moves; a tracker view changes.

## The user

| Name | Email | GitHub | Slack | Role and level | Notes tag |
|---|---|---|---|---|---|
| <name> | <email> | `<login>` | `<username>` (`<user ID>`) | <role>, **<level>** | `[<tag>]` |

The notes tag is how meeting notes attribute lines to the user.

## Direct reports

| Person | Email | GitHub | Squad | Level | 1:1 doc ID |
|---|---|---|---|---|---|
| <name> | <email> | `<login>` | <squad> | <level> | `<doc ID>` |

- **Slack username:** <how to derive it, for example the part of the email before `@`>.
- **Notes tag:** <how to derive it, for example the first name in lowercase>.
- **1:1 docs:** `https://docs.google.com/document/d/<ID>/edit`. Rolling, newest entry first.
- **Matching a name:** full name, first name, GitHub login, or email. If it matches no one
  here, stop and ask the user; don't look people up elsewhere.
- **Levels** anchor performance ratings. Update a row when someone is promoted.

## Linear

### Where the work comes from

<What's in scope for the team: Linear team key, initiative views or projects per squad.
Nothing outside them is in scope.>

| Source | Squad |
|---|---|
| <view or initiative URL, and how to query it> | <squad> |

- **Team:** <name>, key `<KEY>`.
- **Project labels:** <labels that sort projects by kind of work, if any>.
- **Unplanned work:** <how it's tracked, if it is>.
- **Other teams' boards** the team often files on: <keys>.

### Using the Linear tools

- The tools are deferred: load them before the first call, by name, for example
  `ToolSearch "select:mcp__claude_ai_Linear__list_issues,mcp__claude_ai_Linear__list_projects"`.
- Page with `cursor` while `hasNextPage` is true.
- Use only URLs Linear returns (request the `url` field). Never build one from a name; the
  slug ends in an ID.

## Slack

- Use the Slack connector's `slack_search_public_and_private`; it reaches public and private
  channels and group DMs, with thread context and resolved names. `slack_read_thread` reads a
  full thread. Load them by name with `ToolSearch`.
- **Search by sender, not by name:** `from:<username>`. A first name matches other people.
- `after:` and `before:` are both **exclusive**. Searches fail past 20 pages (400 results):
  split long windows by month.
- If the Slack tools aren't available, **say so** and report Slack as unavailable. Silently
  leaving it out reads as "nothing happened in Slack."
- **Channels worth knowing:**
  - Incidents: <pattern>
  - Other teams: <pattern>
  - Partners: <pattern>
  - Tracker project updates post to: <channel>

## Repos and git

- **Team repos:** `<owner/repo>`, `<owner/repo>`.
- **Upstream projects** the team contributes to: `<owner/repo>`.
- **Local clones** live at `~/Documents/<repo>`. `origin` is the upstream repo.

Remotes, personal repos, and the default branch are in the `git-conventions` skill.

## Docs

| Doc | ID | Notes |
|---|---|---|
| <meeting notes doc> | `<ID>` | <which meetings; how lines are tagged> |
| <status update doc> | `<ID>` | <how it's organized> |
| <review format doc> | `<ID>` | The review format: questions, rating scale, promotion requirements. Fetch fresh each cycle. |
| <leveling framework> | `<ID>` | <sheet tabs and ranges> |

<Gemini notes, if the meetings have them: the Drive name filter.>

**Meeting notes for a person:** <which docs cover them>.

## Status updates

<Where the team's status updates live and how they're organized: what shipped, what was
committed versus done, risks.>

## Working files and gws

- **Working directory:** `~/Documents/manager`. Run `gws` from there (it only writes
  `--output` files inside the current directory), write drafts there, and delete temporary
  exports when done. Keep it outside any git repo: drafts can be private.
- **Check gws before any Drive work:**

  ```bash
  mkdir -p ~/Documents/manager && cd ~/Documents/manager && gws drive files list --params '{"pageSize": 1}'
  ```

  On an auth error (`invalid_grant`, `invalid_rapt`), **stop**, and tell the user to run
  `! gws auth login`. Pick up once they confirm.
- **Export a doc as text:**
  `gws drive files export --params '{"fileId": "<id>", "mimeType": "text/plain"}' --output <id>.txt`.

## Calendar

<Fiscal year and quarters, for example: the fiscal year runs Feb 1 to Jan 31; Q1 is Feb–Apr.>

## Review cycle

- **Tool:** <review tool>.
- **Format and leveling:** the review format doc and leveling framework in the Docs table.
  Fetch both fresh each cycle; they win over anything below.
- **Leveling tabs:** IC track `<tab>!<range>`; manager track `<tab>!<range>`.
- **Competencies** (one column each): <list>.

### Rating scale

- **<N> <label>:** <definition>

### Self-review prompts

1. <prompt>

### Manager review template

<Which sections are employee-facing and which are private.>

```markdown
# <Person> — Performance Review <Quarter>

## 1. <section>
```

### Upward survey dimensions

- **<dimension>:** <definition>

## AI review bot

- **Label:** `<label>`. Adding it to a PR requests a review.
- **Login:** `<bot login>`.
- **Summary comment** starts `<prefix>`. <When it's done with a commit.>
- **Inline comments** open with `<prefix>`.

## Glossary

Words with a team-specific meaning. In writing for other audiences, pick another word for the
everyday sense.

- **<term>:** <meaning here>

## Standing agendas

Topics the user raises in every 1:1 with that person. Edit this table whenever the user
supplies a standing agenda item.

| Person | Standing agenda |
|---|---|
| <name> | |
