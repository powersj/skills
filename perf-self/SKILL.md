---
name: perf-self
description: Draft the user's own quarterly self-review as an engineering manager, using the prompts and review tool in the team skill (confirm against the live form each cycle). Gathers the quarter's evidence mostly from the team's results (status updates, initiatives, unplanned work), people leadership (1:1 docs, hiring, growth), docs and initiatives the user owns, cross-team work, and the user's own PRs; frames it against the user's level on the manager track; and keeps a private appendix mapping evidence to the rating the user is aiming for. Use when the self-reflection cycle opens, or to compile self-review evidence mid-quarter.
user-invocable: true
argument-hint: "[quarter, e.g. FY27Q3] [target rating, e.g. 'exceeds expectations']"
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

# Self-review

Draft the user's quarterly self-review. The output is a local draft the user iterates on,
then pastes into the review tool.

**Read the team reference first:** `<base directory>/../team/SKILL.md`, for the roster, the Linear
initiative scope, the repos, the docs, status updates, the review cycle, and the fiscal
calendar. The review format doc and
leveling framework are the same ones `perf-team` uses.

## Context that carries across cycles

- **Role and level:** the user's role and level are in the team reference ("The user"). For
  an engineering manager, frame against their row on the manager track of the leveling
  framework (tab in the team reference), and the next level's row for growth goals.
- **Their team:** the roster and squads in the team reference.
  **A manager's outcomes are mostly delivered through the team**, so most evidence is the
  team's results and what the user did to make them happen.
- **Rating scale and self-review prompts:** in the team reference's Review cycle section.
  The wording and even the number of questions change between cycles, so confirm against the
  live form and the review format doc each time. The steps below assume the usual four
  prompts: outcomes, cultural contribution, one thing to do differently, and goals for next
  quarter. Adapt them to whatever the form asks.
- **Structure:** the submitted answers stay short (bullets, a paragraph each). The long-form
  outcomes and the evidence mapping go in a clearly marked **not-for-submission appendix**,
  used to prepare for the review conversation.

## Step 0: Check gws, and fetch the format

```bash
mkdir -p ~/Documents/manager/perf-reviews && cd ~/Documents/manager
gws drive files export --params '{"fileId": "<review format doc ID>", "mimeType": "text/plain"}' --output perf_review_format.txt
gws sheets spreadsheets values get --params '{"spreadsheetId": "<leveling framework ID>", "range": "<manager track tab>!<range>"}' > leveling_em.json
```

Check gws first, as the team reference says; the doc IDs are in its Docs table. If the format doc's
self-review questions differ from the team reference's list, follow the doc and say what changed.

**Run it after the quarter closes.** Mid-quarter counts undercount work already done, not
just work still to come. If run mid-quarter, re-run every count in Step 1 after the quarter
closes and before submitting.

## Step 1: Gather evidence

Run sources in parallel. `<start>` and `<end>` are the quarter's first and last days.

### Last cycle

- **Last quarter's self-review:** `~/Documents/manager/perf-reviews/<previous quarter>/self-review.md`.
  Its goals are what this quarter's outcomes are measured against first.
- **Last quarter's manager review:** ask the user to paste it or point to it. Note the rating,
  the outcomes their manager chose to highlight (that signals what lands), and the focus
  areas. This quarter's draft should show visible progress on those focus areas.

### The team's results

- **Status updates** for the quarter, wherever the team reference says they live: what
  shipped, what was committed versus done, what didn't go well, and risks that got resolved.
- **Linear initiatives** in the team reference's scope: what completed or moved materially
  in the quarter. Also initiatives the user owns: `list_initiatives` with `owner: "me"`.
- **Unplanned work,** if the team reference says how it's tracked: issues handled in the
  quarter, per squad.

### People leadership

- **1:1 docs** for each report (IDs in the roster), entries in the quarter: growth work,
  feedback given and acted on, hard conversations. **Use these for the user's own framing only;
  never quote a report's private 1:1 content in text that will be shared.** Name the pattern,
  not the person's details.
- **Hiring and staffing** from the meeting notes and status updates:
  roles opened and filled, onboarding, on-call and rotation changes.
- **Promotions or level changes** among reports, if any.

### Direction, process, and cross-team work

- **Docs the user owns,** edited in the quarter:
  `gws drive files list --params '{"q": "'\''<the user's email>'\'' in owners and modifiedTime > '\''<start>T00:00:00Z'\''", "fields": "files(id,name,modifiedTime,webViewLink)", "pageSize": 100}'`.
  Vision docs, how-we-work manuals, proposals, and planning docs weigh heavily for a manager.
- **Meeting notes** (doc IDs in the team reference): lines tagged with the user's notes tag, and decisions
  the user made in the Gemini notes.
- **Slack:** `from:<the user's Slack username>`, one month at a time. Cross-team channels, partner
  channels, incident channels, and leadership-facing threads.
- **Linear status updates** the user wrote: `get_status_updates` with `user: "me"` for both
  `type: "project"` and `type: "initiative"`.

### Their own technical work

- **PRs authored:** `gh search prs --author=<the user's GitHub login> --merged-at=<start>..<end> --json repository,number,title,url --limit 200`.
- **PRs reviewed:** `gh search prs --reviewed-by=<the user's GitHub login> --merged-at=<start>..<end> --limit 300 --json repository,author,number,url`,
  excluding their own, grouped by repo.
- **Issues filed:** `list_issues` with `creator: "me"`, `createdAt: <start>`, counted by team.
- **Skills and tooling** they built for the team, such as this skills repo:
  `git -C ~/Documents/skills log --since=<start> --until=<end> --oneline`.

### Highlights

Always ask the user which docs, threads, or moments they consider the quarter's highlights before
choosing outcomes.

## Step 1b: How the user added value

Before choosing anything, answer two questions from the evidence, and write the answers to
the appendix. This is the core of a manager's self-review, and the reason the outcomes get
picked the way they do.

### Is the team delivering in a meaningful way?

Not "did things ship" but "was it the right work, and did it matter":
- **On plan:** the share of the team's completed work (issues and projects) that sits inside
  the team reference's initiative scope, versus outside it. Work outside the initiatives is
  drift; name where it came from.
- **Said versus did:** from the quarter's status updates, how often what was committed
  matched what was done. Name the pattern (matching, over-reaching, sandbagging) with the months.
- **Outcomes, not activity:** for the quarter's main shipped items, the result for
  the team's users or customers, with a number (manual work removed, findings recovered,
  incidents avoided). Flag shipped items that have no stated result.
- **Capacity:** unplanned work's share of the team's completed issues, if the team tracks
  it, and whether it went up or down.
- **Kinds of work:** the rough split by project count, from the project labels in the team
  reference, if it defines any.

### What did the user add that wouldn't have happened without them?

For each, the evidence, and the line between what the user did and what the team did:
- **Direction:** initiatives they created, reshaped, or retired; how they changed how the team
  plans (for example, moving from dated quarterly projects to initiative views per squad);
  vision and how-we-work docs.
- **Decisions and unblocking:** decisions recorded in the meeting notes (lines with the user's notes tag,
  Gemini summaries), escalations they took on, risks that went from red to resolved after
  they acted, asks they made of other teams that landed.
- **People:** reports whose scope grew (new projects led, new areas owned), hires,
  onboarding, promotions, and feedback in 1:1s that visibly changed something.
- **Leverage:** process and tooling that made the whole team faster: rotations, on-call
  changes, standups, automation, skills.
- **Representation:** status updates they wrote for leadership, and cross-team work on the
  team's behalf.

### Where the value was missing

The same evidence, read honestly: initiatives that stalled, repeated slips, drift that went
unchallenged, 1:1 action items the user owned that stayed open, problems they heard about late.
These are the candidates for question 3 (what I'd do differently). Don't soften them here;
the appendix is private.

Then ask the user the counterfactual directly, in Step 2: "Where would the team be this quarter
if you hadn't been there?" Their answer, checked against this evidence, is usually the
strongest outcome.

## Step 2: Ask the user

One message, pre-filled from the evidence:

> For your <quarter> self-review:
> 1. **Outcomes:** the record shows <4–6 candidates, each with a one-line piece of
>    evidence>. Which 2–3 are you proudest of, and what's missing that the record doesn't
>    capture?
> 2. **Your value:** where would the team be this quarter if you hadn't been there? Here's
>    what the evidence shows you added: <top 3 from Step 1b>, and where it was missing:
>    <top 2>. Does that match how you see it?
> 3. **Cultural contribution:** candidates: <2–3>. Which one?
> 4. **Do differently:** what's one thing you'd do differently, and why? Candidates from
>    where the value was missing: <…>. <If last review had focus areas: "Last review's focus
>    areas were <…>; does one apply?">
> 5. **Goals:** your top 2–3 goals for next quarter, raw. I'll shape them.
> 6. **Target rating:** are you making a case for a particular rating?

If an answer is vague, ask one follow-up ("What did that enable?", "How will you know it
worked?"), not more.

## Step 3: Select and structure

- **Lead with outcomes that answer last quarter's goals.** The reviewer reads them side by
  side.
- **Then the pattern for the target rating.** For Greater, the case is the pattern: repeated
  scope beyond the role, not one big item.
- **Build outcomes from Step 1b.** The strongest outcomes pair a meaningful team result with
  what the user added to it.
- **Say what the user did, not just what the team shipped.** "Restructured the team into two
  squads with their own initiative views, and both squads shipped …" — the team's result plus
  the manager's contribution to it.
- **Do differently:** one genuine, evidenced self-critique, forward-framed: the change, why
  it matters, the learning, and proof it's already improving. Not a humble-brag, and not a
  list of past failures.
- **Goals:** 2–3, scoped to what the user directly owns or decides. Scope out anything measured
  against another team's or partner's timeline. Each needs a success criterion a reviewer
  can check: `**<Goal>** — By <time>: <measurable outcome>`.

## Step 4: Style conventions

- **Verb-led outcomes:** the action, then the evidence, then the impact.
- **Link, don't describe.** Docs, initiatives, and PRs by title or ID, linked. Full PR references
  (`<owner>/<repo>#<n>`, linked), never a bare `#<n>`. Never "the April doc."
- **No local file paths** in shared text.
- **Credit precisely.** Name reports who did the work; claim the part the user did (set
  direction, unblocked, reorganized, hired, decided). Never claim a report's PR or idea.
- **One plain sentence of impact first** for anything leadership reads; numbers up front.
- **Goals must be directly ownable.**
- **No unexplained acronyms,** no boilerplate headings.

When the user corrects the draft's style, add the correction here so the next cycle doesn't
relearn it.

## Step 5: Draft

Write `~/Documents/manager/perf-reviews/<FYxxQx>/self-review.md`:

```markdown
# Self-review — <Quarter>

Previous reviews: <links to earlier self-reviews and manager reviews, newest first>

## 1. Outcomes delivered this quarter
- **<Outcome>** — <one sentence of context or impact, linked>

## 2. Cultural contribution
<a short paragraph>

## 3. What I'd do differently
<2–3 sentences: the change, why, the learning, and the sign it's improving>

## 4. Goals for next quarter
- **<Goal>** — By <time>: <measurable outcome>

---

## Appendix (NOT for submission)

### How I added value
<Step 1b's three parts: is the team delivering meaningfully (with the numbers), what the user
added, and where the value was missing, each item linked to its evidence>

### Outcomes in full
<the long-form version of each outcome, with every link and number>

### Evidence for <target rating>
<each outcome mapped to the rating's clauses (expanded scope, higher-stakes work, more
complex problems, pattern not occasional) and to their level row's competency columns>

### Counts
<PRs authored and reviewed, issues filed by team, docs, standing obligations, with the
date the counts were taken>
```

The submitted text shows the evidence; it doesn't name the rating.

## Step 6: Edit for clarity

Invoke `Skill` with `writing-clarity`, then `writing-google-style`, on sections 1–4.
Keep every fact, number, link, and name.

## Step 7: Hand it over

Show each answer under a `Text to copy:` label, as plain prose with no fenced code block, so
it pastes cleanly into the review tool. Then:

> "Draft at `<path>`, with the appendix for your review conversation. Edit directly, or tell
> me what to change."

Expect a few rounds. Re-sync any edits the user makes elsewhere back into the markdown, so next
quarter's Step 1 reads what was actually submitted.
