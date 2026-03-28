---
name: rfe.review
description: Review and improve RFEs. Accepts a Jira key (e.g., /rfe.review RHAIRFE-1234) to fetch and review an existing RFE, or reviews local artifacts from /rfe.create. Runs rubric scoring, technical feasibility checks, and auto-revises issues it finds.
user-invocable: true
allowed-tools: Read, Write, Edit, Glob, Grep, Bash, Skill, AskUserQuestion, mcp__atlassian__getJiraIssue
---

You are an RFE review orchestrator. Your job is to review RFEs for quality and technical feasibility, and auto-revise issues when possible.

## Step 0: Resolve Input

Check if `$ARGUMENTS` contains a Jira key (e.g., `RHAIRFE-1234`).

**If a Jira key is provided**: Fetch the RFE from Jira. Try `mcp__atlassian__getJiraIssue` first. If the MCP tool is unavailable, fall back to the REST API script:

```bash
python3 scripts/fetch_issue.py RHAIRFE-1234 --fields summary,description,priority,labels,status,comment --markdown
```

The script outputs JSON to stdout with description and comment bodies already converted to markdown. Parse `fields.description`, `fields.summary`, `fields.priority.name`, and `comments` array.

Write the RFE to `artifacts/rfe-tasks/` as a local artifact using the RFE template format (read `${CLAUDE_SKILL_DIR}/../rfe.create/rfe-template.md` for the format). Update `artifacts/rfes.md` with the RFE summary. Record the Jira key in the artifact metadata so `/rfe.submit` knows to update rather than create.

**Also write a separate comments file** to `artifacts/rfe-tasks/RFE-NNN-comments.md` with the Jira comment history. Format each comment as:

```markdown
# Comments: RHAIRFE-NNNN

## <Author Name> — <date>
<comment body>

## <Author Name> — <date>
<comment body>
```

This file provides stakeholder context to the review forks. It is NOT part of the RFE content and must NOT be pushed back to Jira during submission.

**If no Jira key**: Proceed with existing local artifacts.

## Step 1: Verify Artifacts Exist

Read `artifacts/rfes.md` and list files in `artifacts/rfe-tasks/`. If no RFE artifacts exist and no Jira key was provided, tell the user to run `/rfe.create` first or provide a Jira key (e.g., `/rfe.review RHAIRFE-1234`) and stop.

Check if a prior review report exists at `artifacts/rfe-review-report.md`. If it does, read it — this is a re-review after revisions.

## Step 1.5: Fetch Architecture Context

```bash
bash scripts/fetch-architecture-context.sh
```

The architecture context path for the feasibility fork is `.context/architecture-context/architecture/$LATEST`.

If the fetch fails (network issue, repo unavailable, API rate limit), proceed without architecture context. Note it in the review report.

## Step 2: Run Reviews

Run two independent reviews. These assessments must remain separate — "this RFE is poorly written" is a different concern from "this RFE is technically infeasible."

### Review 1: Rubric Validation

<!-- TEMPORARY: This bootstrap approach clones assess-rfe from GitHub and copies
     the skill locally because the Claude Agent SDK doesn't yet support marketplace
     plugin resolution. Once the SDK or ambient runner adds plugin support, this
     can be replaced with a direct /assess-rfe:assess-rfe plugin invocation. -->

Bootstrap the assess-rfe skill by running:

```bash
bash scripts/bootstrap-assess-rfe.sh
```

This clones the assess-rfe repo into `.context/assess-rfe/` and copies the skills into `.claude/skills/`. If the clone already exists, it reuses it.

When any assess-rfe skill resolves its `{PLUGIN_ROOT}`, it should use the absolute path of `.context/assess-rfe/` in the project working directory.

**If the bootstrap succeeded**: Invoke `/assess-rfe` to score each RFE against the rubric. The plugin owns the scoring logic, criteria, and calibration. Do not reimplement or second-guess its scores.

**If the bootstrap failed** (network issue, git unavailable): Skip rubric validation. Note in the review report that rubric validation was skipped because assess-rfe could not be fetched. Perform a basic quality check instead:
- Does each RFE describe a business need (WHAT/WHY), not a task or technical activity?
- Does each RFE avoid prescribing architecture, technology, or implementation?
- Does each RFE name specific affected customers?
- Does each RFE include evidence-based business justification?
- Is each RFE right-sized for a single strategy feature?

### Stakeholder Context

Both review forks should read any `artifacts/rfe-tasks/RFE-NNN-comments.md` files that exist. Comments from stakeholders provide context about what is intentional in the RFE, what has already been discussed, and what related work exists. This context should inform the review — e.g., if a stakeholder has already confirmed a technology choice is deliberate, the rubric should not penalize it.

### Review 2: Technical Feasibility (Forked)

Invoke the `rfe-feasibility-review` skill on the RFE artifacts. This runs in a forked context with architecture context (if available) to assess whether each RFE is technically feasible without the business context influencing the assessment. If comments files exist in `artifacts/rfe-tasks/`, include them in the feasibility reviewer's context.

## Step 3: Combine Results

Write `artifacts/rfe-review-report.md` with the following structure:

```markdown
# RFE Review Report

**Date**: <date>
**RFEs reviewed**: <count>
**Rubric validation**: <pass/fail/skipped>
**Technical feasibility**: <pass/conditional/fail>

## Summary
<Overall assessment: are these RFEs ready for submission?>

## Per-RFE Results

### RFE-001: <title>

**Rubric score**: <score>/10 <PASS/FAIL> (or "skipped — plugin not installed")
<Include the FULL rubric feedback verbatim — scores, notes, verdict, and actionable suggestions. Do not summarize or paraphrase the assessor's output. The assessor's specific recommendations (e.g., "remove the Technical Approach section") are instructions for the revision step and must not be lost in summarization.>

**Technical feasibility**: <feasible / infeasible / needs RFE revision>
**Strategy considerations**: <none / list of items flagged for /strat.refine>

**Recommendation**: <submit / revise / split / reject>
<Specific actionable suggestions if revision needed>

### RFE-002: <title>
...

## Changes vs. Original
<For Jira-sourced RFEs only: summarize what was modified compared to the original Jira description. List sections added, removed, or edited so the user can see what will change if they submit.>

## Revision History
<If this is a re-review, note what changed since the prior review:>
- What concerns from the prior review were addressed
- What concerns remain
- What new issues the revisions introduced
```

## Step 4: Auto-Revise

Always attempt at least one auto-revision cycle when any criterion scores below full marks. Improve what you can with available information. If a revision requires information you don't have (e.g., named customer accounts), make the best improvement possible and note the gap in Revision Notes for the user. Only skip auto-revision entirely if the RFE is technically infeasible or the problem statement needs to be rethought from scratch.

### Revision Principles

**Only edit sections that directly caused a rubric failure.** If the rubric didn't flag a section, don't touch it. If you're unsure whether a section contributed to a score, leave it alone. Never rewrite the entire artifact from scratch — this destroys author context that wasn't scored.

**Reframe, don't remove.** When the assessor flags sections for HOW violations, the problem may not be the information — it's the framing. Prescriptive architecture and implementation directives can almost always be reframed into non-prescriptive context that preserves useful information while fixing the rubric score. For example, a section that assigns components to architectural roles can be reframed as a flat context list with a disclaimer that engineering should determine the design. Only remove content as a last resort when there is nothing reframeable (pure implementation detail with no business-facing content).

**If content must be removed**, preserve it for Jira-sourced RFEs by writing it to `artifacts/rfe-tasks/RFE-NNN-removed-context.md` so `/rfe.submit` can post it as a comment, prefixed with:

```
*[RFE Creator]* The following technical implementation details were removed from the RFE description during review. This content is better suited for a RHAISTRAT and is preserved here for reference:
```

This file must NOT be merged back into the RFE description.

**When a section mixes WHAT and HOW and the assessor did not flag it**, leave it alone. Do not proactively scan for additional HOW content beyond what the assessor identified.

**Right-sizing is a recommendation, never auto-applied.** If the rubric scores Right-sized at 0 or 1, report the recommendation to split in the review report. Do NOT remove acceptance criteria, scope items, or capabilities from the artifact to force a different shape. Splitting an RFE is a structural decision that changes what the RFE *is* — only the author can make that call.

**Do not invent missing evidence.** If the rubric flags weak business justification due to missing named customers or revenue data, flag the gap in Revision Notes for the author to fill. Do not fabricate evidence.

### Revision Steps

1. Read the **full** review feedback for each failing RFE (the verbatim assessor output in the review report)
2. Read the comments file (`artifacts/rfe-tasks/RFE-NNN-comments.md`) if it exists — stakeholder comments may explain why certain content is intentional
3. For each criterion the assessor flagged, follow its specific recommendations:
   - **Open to HOW**: Reframe flagged sections to remove prescriptive framing while preserving useful context. If content cannot be reframed, remove it and write it to `artifacts/rfe-tasks/RFE-NNN-removed-context.md` for preservation as a Jira comment during `/rfe.submit`
   - **WHY**: Strengthen with available evidence (stakeholder comments, strategic alignment references); flag gaps the author must fill (named customers, revenue data)
   - **Right-sized**: Report the recommendation only; do not split or remove scope. Advise the user to run `/rfe.split` if splitting is needed
   - **WHAT / Not a task**: Follow assessor guidance if provided
4. Add a `### Revision Notes` section at the end of each RFE **only if its content was actually changed** (sections rewritten, reframed, or removed). The Revision Notes should document what changed and why. Do NOT add Revision Notes to artifacts where no content was modified — gaps that require author input (e.g., missing named customers) belong only in the review report, not in the artifact. This distinction matters because the submit script uses the presence of Revision Notes to apply the `rfe-creator-auto-revised` label.
5. Re-run the review (go back to Step 2) on the revised artifacts

**Revision limits**:
- Maximum 2 auto-revision cycles
- If RFEs still fail after 2 cycles, stop and present the review report to the user

## Step 5: Advise the User

Based on the results:
- **All pass**: Tell the user RFEs are ready for `/rfe.submit`.
- **Some need revision after auto-revise failed**: List the remaining issues. Tell the user to edit the artifact files and re-run `/rfe.review`.
- **Fundamental problems**: Recommend re-running `/rfe.create` if the RFEs need to be rethought entirely.

$ARGUMENTS
