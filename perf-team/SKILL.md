---
name: perf-team
description: Draft quarterly manager reviews for the user's direct reports (the roster lives in the team skill), one person or the whole team. Fetches the current review format and leveling framework fresh, gathers the quarter's evidence (GitHub PRs authored and reviewed, Linear work and issues filed, status updates, Slack, Drive docs, 1:1 docs, the team's status updates, meeting notes), writes an evidence file organized by leveling competency, proposes a rating anchored to the leveling row, then drafts the review. Whole-team runs end with a calibration table. Use when the quarterly review cycle opens, or when asked to draft a performance review, manager review, or rating for a report.
user-invocable: true
argument-hint: "<person | all> [quarter, e.g. FY27Q3]"
allowed-tools:
- Bash
- Read
- Write
- Edit
- Agent
- ToolSearch
- AskUserQuestion
- Skill
- mcp__claude_ai_Linear__list_issues
- mcp__claude_ai_Linear__list_projects
- mcp__claude_ai_Linear__list_initiatives
- mcp__claude_ai_Linear__get_status_updates
- mcp__claude_ai_Slack__slack_search_public_and_private
- mcp__claude_ai_Slack__slack_read_thread
---

# Performance review drafter

Draft quarterly manager reviews for the user's direct reports. The review is the user's judgment;
this skill's job is to make sure that judgment rests on the whole quarter's evidence, anchored
to the leveling framework, and written in the current format.

**Read the team reference first:** `<base directory>/../team/SKILL.md`. It has the roster (emails,
GitHub and Slack handles, squads, 1:1 docs), the Linear initiative scope, the repos, the
meeting-notes docs, status updates, the review cycle (rating scale, template, competencies),
and the fiscal calendar. Don't copy any of it here.

## Invocation

```
/perf-team <person | all> [quarter]
```

- **person:** anyone in the team reference's roster, or `all` for the whole team in one pass.
- **quarter:** `FY27Q3`, or an explicit date range. Defaults to the most recently completed
  fiscal quarter (calendar in the team reference).

**Run it after the quarter closes.** Counts gathered mid-quarter don't just miss the rest of
the quarter; they undercount work already done, because search indexes lag. If the quarter
hasn't ended, say so, draft anyway if the user wants, and re-run every count in Phase 3 after the
quarter closes and before submitting.

Everything this skill writes is private. Output goes to
`~/Documents/manager/perf-reviews/<FYxxQx>/`, never into a git repo.

## Phase 0: Check gws

Check gws as the team reference says. On an auth error, stop.

## Phase 1: Fetch the format and the leveling framework

Always fetch both fresh; they change between cycles. Their doc IDs and sheet ranges are in
the team reference's Docs table.

```bash
mkdir -p ~/Documents/manager/perf-reviews && cd ~/Documents/manager
gws drive files export --params '{"fileId": "<review format doc ID>", "mimeType": "text/plain"}' --output perf_review_format.txt
gws sheets spreadsheets values get --params '{"spreadsheetId": "<leveling framework ID>", "range": "<IC track tab>!<range>"}' > leveling_ic.json
```

- **The format doc is the source of truth** for the review's questions, the rating scale,
  and promotion requirements. If it differs from the template in the team
  reference, follow the doc and tell the user what changed.
- **Leveling:** each row is a level, each column a competency (listed in the team
  reference). Take the row for the person's level, and the next level's row if
  promotion is in question.
- **Levels** are in the team reference's roster. Ratings are anchored to level, so **never
  propose a rating without one.** If a level is missing, ask the user, and record it in the
  team reference.

## Phase 2: Ask what only the user knows

One message, per person (or one for the whole team):

> For <person>'s <quarter> review:
> 1. **Level:** <their level from the team reference; confirm it's current>
> 2. **Self-review:** paste it or share the doc, if they've written one.
> 3. **Promotion:** is promotion in question this cycle?
> 4. **Difference-maker moments:** anything that won't show up in Linear or GitHub? Customer
>    escalations, incidents, a hard conversation handled well.
> 5. **Docs to include:** any design docs, PRDs, or retros I should read?

If promotion is in question, also ask for 2–4 peers or cross-team partners whose feedback
should anchor the case, and any peer feedback already collected. Offer to draft outreach
messages to collect more.

## Phase 3: Gather evidence

Run sources in parallel; for the whole team, fan out one background agent per person with the
same instructions. `<start>` and `<end>` are the quarter's first and last days.

### GitHub

- **Authored, in the team's repos:**
  `gh search prs --repo=<owner/repo> --author=<login> --merged-at=<start>..<end> --json title,number,url,repository,closedAt --limit 200`
  for each team repo in the team reference. Note notable PRs: large, cross-cutting, incident-related.
- **Authored elsewhere:** `gh search prs --author=<login> --merged-at=<start>..<end>` across
  all orgs, minus the team repos. Upstream PRs (to the team reference's upstream projects, or
  any open source) are a strong signal at higher levels.
- **Reviews given:**
  `gh search prs --reviewed-by=<login> --merged-at=<start>..<end> --limit 300 --json repository,author,number,url`,
  excluding PRs they authored. Group by repo: **reviews outside the team repos are a
  cross-team collaboration signal; call them out.** For a sample, read the substance
  (`gh api repos/<owner>/<repo>/pulls/<n>/reviews` and `/comments`, filtered to their login):
  depth versus rubber stamps, mentoring, real catches.
- The `--reviewed-by` count can drop between runs (a search-index quirk). Treat the latest run
  as authoritative.

### Linear

- **Completed:** `list_issues` with `assignee: <email>`, paged, filtered to `completedAt`
  inside the quarter.
- **Filed:** `list_issues` with `creator: <email>`, `createdAt: <start>`, paged, filtered to
  the quarter. This catches issues filed on other teams' boards (the team reference lists the
  usual ones), which an assignee search misses. Count them by team.
- **Projects led:** `list_projects` with `member: <email>`, keeping those where they're the
  lead. Note each project's initiative (in or out of the team reference's scope), whether it
  completed, and its `delivery` or `exploratory` label.
- **Status updates written:** `get_status_updates` with `type: "project"`,
  `user: <email>`, `createdAt: <start>`, paged. Their quality is evidence: clear, numeric,
  honest about health.

### Slack

Search `from:<username>` one month at a time, following the team reference's Slack rules.
Look for:
- Incidents they led or root-caused, and their role in each.
- Cross-team help: messages in other teams' channels.
- Partner work: partner channels.
- Design discussions they drove, and unblocking others.

The team reference's Slack section has the channel patterns for incidents, other teams, and
partners.

For a per-channel tally, delegate to a subagent: it needs full pagination.

### Drive

- **Docs they own, edited in the quarter:**
  `gws drive files list --params '{"q": "'\''<email>'\'' in owners and modifiedTime > '\''<start>T00:00:00Z'\''", "fields": "files(id,name,modifiedTime,webViewLink)", "pageSize": 100}'`.
  Design docs, proposals, and problem statements weigh heavily for Scope and Impact.
- Any docs the user named in Phase 2.

### The team's own records

These are what make a review specific:
- **Their 1:1 doc** (ID in the roster), entries in the quarter: topics they raised, action
  items they owned and whether they closed, feedback the user gave and what changed after.
- **Status updates** for the quarter, wherever the team reference says they live: what
  shipped that they led, what they carried in flight, and any risks on their projects.
- **Meeting notes** (doc IDs in the team reference): lines tagged with their name, and action
  items assigned to them in Gemini notes.

## Phase 4: Write the evidence file

Write `~/Documents/manager/perf-reviews/<FYxxQx>/<first-last>_evidence.md`, so the user can check it
before anything is drafted. Organize it by leveling competency:

- **Scope and Impact:** outcomes tied to the business, named projects and initiatives,
  design docs authored.
- **Software Development, System Design, Code Quality:** PR substance and architecture.
- **Autonomy and Prioritization:** high-leverage work picked up unprompted, or waited on.
- **Problem-Solving:** debugs, incidents, ambiguous problems.
- **Collaboration and Communication:** reviews given, cross-team work (called out
  separately), docs and comments, Slack help, status-update quality.
- **Accountability and Adaptability:** follow-through on 1:1 action items, handling
  reprioritization.
- **Gaps:** what needs attention.

Every item links to its evidence (PR, issue, doc, Slack permalink) and is marked **at**,
**above**, or **below** the level row, quoting the row's language. Include the counts (PRs
authored and reviewed, issues completed and filed by team, docs) with the date they were
taken.

## Phase 5: Propose a rating

Use the rating scale in the team reference's Review cycle section, checked against the
format doc.

Walk each competency column of the level row and mark the evidence at, above, or below.
Propose a rating with a 2–3 sentence rationale citing 2–3 specific competencies, **before
drafting**, and wait for the user to agree or change it.

- Use the full range over time; compressing everyone into the middle tiers defeats the
  system.
- Distribution guidance is a calibration reference, not a quota. Don't force it on a small
  team, and never move an individual's rating to fit it.

## Phase 6: Draft the review

Write `~/Documents/manager/perf-reviews/<FYxxQx>/<first-last>_review.md`, matching the fetched
format exactly. Start from the manager review template in the team reference; where it and
the format doc differ, the format doc wins. The template marks which sections are
employee-facing and which are private.

### Writing guidelines

- **Direct, factual, specific.** Past tense for outcomes; named projects and numbers. No
  filler ("great team player", "went above and beyond"); replace it with the evidence.
- **Outcomes lead with impact, not activity.** "Found why the nightly import dropped 40% of
  records, and fixed it upstream" beats "worked on the import."
- **Link, don't describe.** Full PR references (`<owner>/<repo>#<n>`, linked), Linear IDs,
  doc titles. Never "the April doc."
- **Don't overstate contribution.** Credit what they did, not what their project did.
- **Focus areas are about growth.** Specific, tied to a competency they're below or
  stretching on. At or above the "meets expectations" tier, keep to real priorities; below
  it, say plainly what success looks like next quarter.
- **Private calibration fields** read differently: compare to level expectations
  in the leveling framework's own words.
- **Promotion readiness defaults to not ready** unless there's pattern-level evidence of working
  at the next level's competencies and a business need. Reference the next-level row
  explicitly.

## Phase 7: Edit for clarity

Invoke `Skill` with `writing-clarity`, then `writing-google-style`, on the
employee-facing fields, as the template marks them. Keep every fact, number, link,
and name; where a style rule conflicts with the format doc, the format doc wins. Leave the
rating and any private calibration field untouched.

## Phase 8: Hand it over

> "Evidence: `<path>`. Draft: `<path>`, at <rating> because <one-sentence anchor to a
> competency>. Edit directly, or tell me what to change."

For revisions, edit in place; don't re-gather evidence unless asked.

## Whole team (`all`)

1. Fetch the format and leveling framework once (Phase 1).
2. Ask Phase 2's questions for everyone in one message.
3. Gather evidence for everyone in parallel, one background agent per person.
4. Write each evidence file, then present a **calibration table** before any drafting:

   | Person | Level | Proposed rating | One-line anchor | Promotion |
   |---|---|---|---|---|

5. Draft once the user has calibrated across the team.

## Key reminders

- **Re-fetch the format doc and leveling framework every cycle.**
- **No rating without a level and the level row.** Ask if it's missing.
- **Run after the quarter closes**, or re-run the counts before submitting.
- **Cross-team work is its own signal:** reviews outside the team repos, issues filed on
  other boards, partner channels, other teams' docs.
- **Ratings reflect performance.** Never adjust one to hit a distribution.
- **Everything is private.** Output stays in `~/Documents/manager/perf-reviews/`; don't commit
  it, and don't put review content or ratings in any skill file. Levels live in the team
  reference; nothing else about a person's review is kept in a skill.
