---
name: <short-name>
description: <one line — what this reference checks>
repos:            # optional: only run on these repos; omit to run on any repo
  - <owner/repo>
applies_to:
  - "<glob of files this applies to, for example **/*.py or charts/**>"
---

# <Title>

Each `##` heading is one rule, with an intent, a detection cue, a carve-out (when not to
flag it), and a default severity. Flag only what's in the diff.

## <Rule name>
<What should be true.> Cue: <what to look for in the diff>. Carve-out: <when not to flag
it>. Severity: blocker | major | minor.
