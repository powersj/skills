---
name: review
description: Review a pull request the way the user does — mostly their direct reports' PRs. Asks whether this is the right change before reading code, then runs parallel reviewers (five fixed lenses plus one per matching reference in references/), disproves weak findings, waits for the AI review bot if the team skill names one (adding its label if it's missing), and returns a private report plus a few short, question-first comments that add to the AI review rather than repeat it. Posts nothing without the user's OK. Use when asked to review a PR, look over a PR, or draft review comments.
user-invocable: true
argument-hint: "<pr-url|pr-number> [--repo <owner/repo>]"
allowed-tools:
- Bash
- Read
- Write
- Agent
- ToolSearch
- AskUserQuestion
- mcp__claude_ai_Linear__get_issue
- mcp__claude_ai_Linear__get_project
---

# Review

Most of these PRs come from the user's direct reports. **Read the team reference first**
(`<base directory>/../team/SKILL.md`) for the user's GitHub login, the roster, where local
clones live, and the AI review bot, if the team has one. The review has two audiences, and they get different things:

- **The user gets everything.** A rigorous private report: is this the right change, what's
  broken, what's unclear, what was checked and found sound.
- **The author gets a few short comments.** Mostly questions. About the code, never the
  person. Nothing that reads as a grade.

The goal isn't to find something to say. It's to ask whether this is the right thing,
check that it works, and help the author get it merged. If a PR is good, approve it with
few or no words.

## Step 1: Resolve the PR

```sh
gh pr view <ref> --json number,title,body,url,author,headRefName,baseRefName,headRefOid,reviews,labels,files
```

A URL gives `<owner>/<repo>` directly. A bare number resolves against `--repo`, or the
current directory's `origin`. `<name>` below is the repo name without the owner.

**Request the AI review.** Skip this if the team reference names no AI review bot. If the
repo has the bot's label (`gh label list -R <owner>/<name> --search <label>`) and the PR
doesn't carry it, add it now, without asking, and tell the user you did:

```sh
gh pr edit <n> -R <owner>/<name> --add-label <label>
```

The bot takes a while, so the steps below run while it works. Your review waits for it in
Step 7, so what you post adds to the AI review instead of repeating it.

## Step 2: Check out the PR for the reviewers

Reviewers read files in full and run tests, so they need the PR's code on disk. Use a
throwaway worktree so nothing touches your own checkouts.

If there's a local clone (see the team reference's Repos section; `origin` is upstream):

```sh
git -C ~/Documents/<name> fetch origin <base> pull/<n>/head
git -C ~/Documents/<name> worktree add --detach /tmp/review-<name>-<n> <headRefOid>
```

Otherwise, clone without blobs:

```sh
gh repo clone <owner>/<name> /tmp/review-<name>-<n> -- --filter=blob:none
git -C /tmp/review-<name>-<n> fetch origin <base> pull/<n>/head
git -C /tmp/review-<name>-<n> checkout --detach <headRefOid>
```

Confirm `git -C /tmp/review-<name>-<n> rev-parse HEAD` equals `headRefOid`. The diff range
is `origin/<base>...HEAD`, excluding `vendor/`, `*.pb.go`, and `*_generated.go`.

## Step 3: Requirements and context

Find the Linear issue ID in the PR body or branch name (pattern `[A-Z]+-[0-9]+`) and fetch
it with `get_issue`: title, description, acceptance criteria. If it belongs to a project,
fetch the project with `get_project` for its goal.

The Linear issue is the requirements. The PR description is the author's claim about what
the code does, to be checked against the code. With no linked issue, fall back to the
description and say so in the report.

## Step 4: Re-review check

Look for the user's earlier review: the latest entry in `reviews` where `author.login` is the
user's GitHub login. If there's none, or its commit equals `headRefOid`, skip this step.

Otherwise the author pushed since your review. Gather your earlier comments and their
replies:

```sh
gh api repos/<owner>/<name>/pulls/<n>/comments --jq '[.[] | {user: .user.login, path, line, body, in_reply_to_id}]'
```

For each thing you asked for, judge it **addressed**, **partial**, **unaddressed**, or
**disputed** (the author pushed back in a reply), and lead the report with a table:

```
| You asked | Verdict | Evidence |
|---|---|---|
| handle nil provider | unaddressed | no change to provider.go |
```

Then scope the rest of the review to what changed since: diff range `<review-sha>...HEAD`.
After a rebase, focus on files your review touched and files new since then.

## Step 5: Pick the references

Each file in `references/` (next to this skill) is a rulebook. Skip files whose names start
with `_`. A reference applies when both of these hold:

- **`repos`** (optional): the PR's `<owner>/<name>` is in the list. No `repos` field means
  any repo.
- **`applies_to`**: any changed file, relative to the repo root, matches any of the globs.

`generic.md` matches everything, so it always runs. A reference with `repos` set only
runs on those repos.

## Step 6: Parallel reviewers

Always fan out, whatever the PR's size. Launch these concurrently with the Agent tool
(general-purpose subagents), in one message:

- **Five lens reviewers**, one per [lens](#lenses)
- **One reference reviewer per matching reference** from Step 5

**Hand off artifacts only.** Each prompt carries the worktree path, PR number, base, diff
range, and the verbatim Linear issue text. No summary of the change, no PR description, no
existing review comments. Reviewers form expectations from the requirements, then read the
code fresh.

<prompt_template>
You are reviewing a pull request through one lens. Assume the diff contains at least one
real problem visible through your lens; your job is to find it. You get no credit for
volume, only for findings that survive scrutiny.

Repo: <owner>/<name>, checked out at <worktree>
PR #<n>, base origin/<base>
Diff: `git -C <worktree> diff <range> -- . ':(exclude)vendor' ':(exclude)*.pb.go' ':(exclude)*_generated.go'`

Requirements (Linear <issue-id>, or "none linked"):
<verbatim issue title + description>

Your lens:
<LENS CHARTER, or for a reference reviewer: "Read <absolute path to reference file>. Walk
every rule in it against the changed lines in files matching its applies_to globs. Apply
each rule's cue and carve-out, and use its severity.">

Method:
1. Read the requirements and form expectations before reading any code.
2. Read the diff, then read every nontrivially changed file in full.
3. For each candidate problem, construct the concrete failure: specific input or state,
   specific wrong outcome.
4. Try to disprove each candidate: look for the guard, invariant, or caller contract that
   makes it a non-issue. Drop what you disprove. Where it's cheap, run the affected
   package's build or tests instead of speculating.
5. Stay in scope: changed code and its direct blast radius. Pre-existing issues only if
   the diff makes them worse or newly reachable.

Return only, for each surviving finding:
- SEVERITY: BLOCKER | MAJOR | MINOR
- WHERE: file:line (a line in the diff)
- CLAIM: one sentence
- SCENARIO: the concrete failure, 1–3 sentences
- CONFIDENCE: confirmed-by-reading | confirmed-by-test | plausible

For each thing only the author can answer:
- QUESTION: one sentence
- WHERE: file:line, or "PR-wide"
- WHY IT MATTERS: what changes depending on the answer

End with "CLEAN: <what you checked and found sound>".
</prompt_template>

### Lenses

**1. Is this the right thing?** Check each acceptance criterion from the Linear issue: met,
partial, or missing. Check each claim in the PR description (fetch it yourself with
`gh pr view`) against the code. Look for scope creep, a simpler way to get the same result,
and whether the change serves the project's goal. Ask what happens next: rollout, existing
users, who gets paged, what follow-up this creates.

**2. Correctness.** Logic errors, off-by-one, nil and zero values, misused APIs, wrong
assumptions about input or external state, unhandled edge cases (empty, huge, duplicate,
concurrent first call), broken invariants across the changed call graph.

**3. Robustness and safety.** Races and unsynchronized shared state, goroutine leaks,
missing cancellation. Swallowed errors, partial writes without cleanup, retries without
backoff or idempotency. Leaked connections, files, or transactions. Authz gaps, injection,
secrets in logs. Unbounded work: queries without limits, fan-out driven by input size,
unbounded memory.

**4. Do the tests prove it?** Could each test fail? A test that passes whether or not the
code works is worse than none. Changed paths are exercised, error paths too, not just the
happy path. Tests assert behavior, not internals. Run the affected tests and report actual
results.

**5. Is it needed, and will it make sense later?** Unneeded code, deps, layers, or
defensive checks for things that can't happen. Duplicated logic or a one-off where the repo
already has a way to do this (look for prior art). Clever control flow. Names and comments
someone new would follow in six months.

## Step 7: Merge and disprove

When the reviewers return:

0. **Wait for the AI review** if the PR carries the bot's label. The team reference gives
   the bot's login, how its summary comment starts, and how to tell it's done with a commit.
   It's done when the latest summary names `headRefOid`:

   ```sh
   gh api repos/<owner>/<name>/issues/<n>/comments --jq '[.[] | select(.user.login == "<bot login>") | .body | select(startswith("<summary prefix>"))] | last'
   ```

   Check every minute or so. After 20 minutes with no summary for the head commit, ask the user whether to keep
   waiting or go ahead without it. If the status isn't `success`, tell the user and ask the
   same.

   Then fetch the bot's inline comments from every round, plus the replies to them. An
   earlier round's comment may already be fixed, or the author may have disputed it:

   ```sh
   gh api --paginate repos/<owner>/<name>/pulls/<n>/comments --jq '[.[] | {id, user: .user.login, path, line, in_reply_to_id, body}]'
   ```

   Keep the bot's comments, and the replies to them. The team reference says how the
   bot's inline comments start.

1. **Dedupe** findings that point at the same line and problem. Keep the highest severity
   and the best scenario.
2. **Disprove** every BLOCKER and MAJOR yourself: read the cited code and its callers, run
   the targeted test if the claim is runnable. Drop what you disprove, and keep a count.
   Mark what you confirm **CONFIRMED**. What you can't settle stays **PLAUSIBLE**, with the
   open question.
3. **MINOR** findings pass through; the reviewer already tried to disprove them once.
4. **Existing comments:** now fetch other reviewers' open comments. Drop anything they
   already raised and you have nothing to add to.
5. **Add to the AI review, don't repeat it.** For each finding, check whether the AI
   review already raised it:
   - **Raised, nothing to add:** drop it from the draft. Note it as "AI review covered" in
     the private report.
   - **Raised, but you have more** (you confirmed it with a test, found a worse scenario, or
     have a better fix): reply on the AI's comment thread instead of opening a new one.
   - **Raised, and the author says it's fixed:** check the fix. If it holds, drop it. If it
     doesn't, reply on that thread with what still breaks.
   - **Not raised:** keep it. This is what your review adds.

   Also check the AI's comments against your own findings. Where it flagged something you
   disproved, say so in the private report so the author isn't left chasing it. Whether to
   reply on that thread is the user's call.

## Step 8: Report and draft

### Private report (for the user)

```
# Review: <owner>/<name>#<n> — <title>

**Author:** <login> · **Linear:** <issue-id> — <title> (or "none linked")
**Reviewers:** 5 lenses + <references used> · **Scope:** full PR (or "since <short sha>")
**Findings:** <raised> raised, <disproved> disproved, <reported> reported
**AI review:** round <r> at <short sha>, <k> comments · <overlap> overlap ours · <disputed> we think are wrong (or "not requested")

## Is this the right thing?
<2–4 sentences: does it do what the issue asks, and is it the right approach?>

## Findings
**[SEVERITY] `file:line` — <claim>** (CONFIRMED | PLAUSIBLE)
<scenario, 1–3 sentences>

## Requirements
<one line per acceptance criterion: met / partial / missing, with file:line>

## For your 1:1, not the PR
<direction, priority, or growth points better raised in person. Omit if none.>

## Clean
<one short paragraph: what was checked and found sound>

## Recommendation
APPROVE / COMMENT / REQUEST CHANGES — <one line why>
```

Findings are terse: claim, scenario, location. No restating the diff, no lectures.

### Draft review (for the author)

Turn the report into what you'd actually post, following [How comments read](#how-comments-read):

- **Confirmed blockers and majors** become comments on the line.
- **Plausible findings** become questions.
- **Minors** mostly get cut. When the PR needs changes anyway, keep the cheap "this isn't
  needed" ones as suggestion blocks.
- **"Is this the right thing" concerns** go in the review body as one or two questions, or
  stay in the 1:1 section if they're about direction rather than this code.
- **Nothing must change?** Approve, with no inline comments and at most a few words.
- **Already in the AI review?** Leave it out, or reply on its thread if you have more to
  say (Step 7.5). The author shouldn't read the same point twice. If the AI review already
  covered everything that matters, your review can just be the event.

Show the draft under the report: body, event, each new comment with its `path:line`, and
each reply with the AI comment it answers.

## Step 9: Post on OK, then clean up

Post only when the user says to in this session. A review is public and can't be unsent. Write
`review.json` with `commit_id` (`headRefOid`), `event`, `body`, and `comments` (`path`,
`line`, `side: "RIGHT"`, `body`), then:

```sh
gh api repos/<owner>/<name>/pulls/<n>/reviews --input review.json
```

Every comment's line must be in the diff, or GitHub rejects the review. Points that can't
anchor to a diff line go in the body.

Post replies on the AI review's threads separately:

```sh
gh api repos/<owner>/<name>/pulls/<n>/comments/<comment-id>/replies -f body='<reply>'
```

Whether or not anything posts, remove the worktree:

```sh
git -C ~/Documents/<name> worktree remove --force /tmp/review-<name>-<n>   # local clone
rm -rf /tmp/review-<name>-<n>                                             # fresh clone
```

## How comments read

The rule: say what you know in as few words as the author needs, and ask only what you
can't find out yourself.

### Questions

Ask only when the answer would change something. A good question does one of six jobs:

| Job | Example |
|---|---|
| How would we know it works? | "Is there anything to grep for that would show success?" |
| I think X, right? | "This only has a 1.29 version, right?" |
| Is this needed? | "Does this need to be 18 or can it be `nodejs`?" |
| Could this be X instead? | "Why not use our image?" |
| What is this for? | "Where does this command come from?" |
| What happens when...? | "When 1.18 is supported, how will we catch the change?" |

- If you know the answer, say it: "No busybox needed for bash." A question means the
  author has a fact you don't.
- Look first, then ask with what you saw: "I see REDIS_IMAGE in the envsubst, but not
  NGINX_IMAGE?"
- Put the proposal in the question so either answer moves things forward: "Can this be
  `py3-pip` instead, or is 3.12 important?"
- If it's rhetorical, answer it: "What's being tested here? Nothing, since the result is
  always true."
- Never "Have you considered...", "What do you think about...", or "Is there a reason you
  chose...".

### Tone

Check the author against the team reference's roster. Most are the user's reports: ask hard
questions about the work, and be careful with the person. Peers and people on other teams
get the same care, with less shorthand, since they don't share the team's context.

- Talk about the code, not the author: "This retries forever if the scan fails," not "You
  didn't handle the failure."
- Question the approach, don't pronounce on it: "Could this be one explicit migration
  instead of doing it in the bot?"
- Thank people by name for real work: "Thanks for digging into this, @<login>." Skip
  praise otherwise.
- Hedge only when you're actually unsure: "I think", "right?". Use "Doesn't need to block,
  but..." for something optional.
- Own your mistakes: "Sorry, missed the trailing newline in my suggestion."
- Anything about priorities, growth, or how they're working goes to the 1:1, not the PR.

### Shape

- One point per comment, on the line it's about. Usually one sentence. Go longer only to
  show why a test can't fail, with the command and its output.
- Give the one reason that matters, not the trail of evidence.
- For a test that can't fail, say what still passes: "Even with no events, this test still
  passes."
- Mechanical fixes are suggestion blocks with a short line of why, or none if it's obvious.
  An empty suggestion block deletes the line.
- Plain words and contractions. Don't coin terms, and don't repeat ones the PR coined; say
  it the way someone new to the code would follow.
- No headers, bold, bullet lists, em dashes, or labels like "nit:", "Heads up:", or
  "Blocking:".
- Keep the count down: cut nits, follow-up ideas, things CI catches, and anything you'd
  hedge with "probably fine".

### Bot-authored PRs

When the author is a bot (for example a dependency-update bot), there's no person to be
careful with. Tell it what to do in short imperatives: "Build from source, don't fetch the
jar." "Pin the action to a commit."

### Review body and event

- **Body:** usually none, at most a sentence or two: "Two small changes and then I think
  this is good to go."
- **Approvals:** no body, or "LGTM!", "Nice catch", "Tested locally and it builds."
- **Event:** COMMENT by default, APPROVE when nothing has to change. REQUEST_CHANGES only
  when it must not merge as is.
- Don't summarize the inline comments, count findings, or list what you checked. That's
  what the private report is for.

### Before posting

For every comment: could it be half as long? Would someone new to this code follow every
word? Would you say it out loud to the author at their desk?

For every question: which job is it? Did you look first? Does the answer change anything?

## Adding a reference

To teach the review a new topic (Python, Helm charts, Terraform, a team convention):

1. Copy `references/_template.md` to `references/<topic>.md`.
2. Set `applies_to` to the globs of files it covers. Use `"**"` for every PR. If the topic
   is specific to one repo, also set `repos`, so its globs don't match look-alike folders
   elsewhere.
3. Write one `##` rule per check, each with a cue, a carve-out, and a severity
   (blocker / major / minor). The carve-out matters most: it keeps the reviewer from
   flagging things that are fine.

The next review picks it up automatically and gives it its own reviewer when a changed file
matches. To retire a topic, delete its file or prefix the name with `_`.

## Constraints

- **Requirements first.** Acceptance criteria are checked one by one, before style.
- **Every finding is disprovable.** No concrete failure scenario means it's a question or
  it's dropped.
- **Precision over volume.** Uncertainty is stated as PLAUSIBLE with the open question,
  never asserted as fact.
- **Report only.** This skill never edits the PR's code and posts nothing without an OK.
  The one exception is adding the AI review bot's label.
- **Add to the AI review, don't duplicate it.** When there is one, wait for it, and post only
  what it missed.
