---
name: perf-feedback
description: Draft evidence-backed feedback that the user gives in a review cycle — upward feedback on their own manager (manager surveys) or peer feedback (the review tool's "wants your feedback" requests). Parses the pasted form, confirms the scores, anonymity, and candour the user chose, sweeps Slack, Linear, Drive, and meeting notes for concrete dated examples, then writes plain-language answers that match the scores, with a private record of the evidence behind each claim. Use when the user pastes an upward-feedback survey or a peer feedback request. For reviews of the user's direct reports, use perf-team; for the user's own self-review, perf-self.
user-invocable: true
argument-hint: "[subject name] [feedback window, e.g. Aug–Oct 2026]"
allowed-tools:
- Bash
- Read
- Write
- Edit
- Agent
- ToolSearch
- AskUserQuestion
- Skill
- mcp__claude_ai_Slack__slack_search_public_and_private
- mcp__claude_ai_Slack__slack_search_users
- mcp__claude_ai_Slack__slack_read_thread
- mcp__claude_ai_Linear__list_projects
- mcp__claude_ai_Linear__list_issues
- mcp__claude_ai_Linear__get_status_updates
---

# Feedback

Turn a pasted feedback form into evidence-backed answers the user can paste. Two kinds:
- **Upward:** a survey about the user's own manager.
- **Peer:** a request for feedback on someone the user worked with, often about one project.

**Read the team reference first:** `<base directory>/../team/SKILL.md`, for the user's Slack ID, the
team's channels, meeting-notes docs, the review cycle, the glossary, and the fiscal calendar. Adjectives without evidence are
useless; this skill's job is the evidence.

## Step 1: Parse the form, and ask what only the user knows

From the pasted form, pull out: the questions, which are required and which optional, the
**visibility statement** (anonymized and pooled, or visible to the subject), and the
deadline.

Then ask, in one `AskUserQuestion`, not one at a time:
- **Scores:** which option the user picked for each rating question. The written answers must
  match them.
- **Anonymity:** identity-safe, or fine to be identifiable? This decides which evidence is
  usable. DM-only anecdotes and personal logistics identify the user even in an "anonymized"
  pool.
- **Candour,** for any "could do better" question: honest but gentle, direct, or both
  versions to choose from.
- **For peer requests:** which project the feedback is about, if the form doesn't say. Offer
  candidates from the collaboration signals (Linear projects the subject led that the user's team
  touched), plus "not sure, infer from the evidence".

## Step 2: Sweep for evidence

Spawn one background agent to gather concrete, dated examples. It should:
- **Resolve the subject's Slack user ID** first (`slack_search_users`).
- **Search their messages** in channels the user shares with them, plus any DM or group DM with
  the user (their Slack ID is in the team reference), across the window. Read the 5–8 most
  substantive threads in full.
- **For upward feedback,** organize by the survey's dimensions, listed in the team
  reference's Review cycle section. Add **advocacy:** shielding the team and speaking up for it. Confirm the dimensions
  against the form; they change.
- **For peer feedback,** first establish which project the collaboration centered on.
  Linear projects the subject led are the strongest signal (`list_projects` with
  `member: <their email>`, then their status updates). Then gather what they did well, and
  factual could-do-better observations.
- **Also check** docs the subject owns from the window (`gws drive files list` with their
  email in owners), and the meeting notes for decisions they drove.
- **Return** each item with its theme, date, channel or doc, a 1–2 sentence description, and
  permalinks for the strongest.

Follow the team reference's Slack rules: search by sender, split long windows by month. DM
filters like `to:me` are unreliable; an agent that has already read a thread is the fastest
way to recover its permalink.

## Step 3: Draft the answers

- **Match the scores.** A "Strongly agree" reinforces with one or two concrete examples. A
  "Neutral" needs an honest explanation.
- **One theme per answer.** "What's one thing" means one thing, with supporting examples,
  not a list of virtues.
- **Name the context.** Peer feedback names the project and what it was in one line;
  reviewers may read it with no shared context.
- **Plain language.** Short sentences, everyday words, no acronyms. Watch for words with a
  team-specific meaning (the team reference's Glossary), and pick another word for the
  everyday sense.
- **Apply the anonymity filter.** Identity-safe: public-channel events only, and generalize
  personal anecdotes. Identifiable: the DM-level examples are usually the strongest; use them.
- **Could-do-better answers:** a factual pattern, concrete instances, and one actionable
  suggestion. Keep it about process. Don't re-litigate a mistake the subject already owned
  publicly. When the user asked for two candour levels, both versions rest on the same facts, so
  they choose on tone alone.
- **Never quote a third party unfavorably,** even if the evidence has it. Leave it out and
  note why in the record.

Then invoke `Skill` with `writing-clarity`, then `writing-google-style`, on the
answers. Keep every fact and example.

Show each answer under a `Text to copy:` label, as plain prose with no fenced code block and
no trailing whitespace.

## Step 4: Record

Write `~/Documents/manager/perf-reviews/<FYxxQx>/feedback-<subject>.md` (quarter from the team
reference's calendar), outside any git repo. It holds:
- The form's questions and the scores the user gave.
- Every answer, plus unused alternates.
- **Evidence anchors:** permalinks and doc links behind each claim, so anything can be
  checked before submitting.
- Anything deliberately left out, and why.
- The deadline, and a note if it has passed.

> "Answers are ready to paste, above. The record with the evidence behind each one is at
> `<path>`."
