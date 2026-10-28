---
name: worklog
description: Build a daily timesheet entry from everything you did on a given date — GitHub commits, PRs and reviews, plus Jira and Confluence when connected — with a mandatory check-in for work that leaves no digital trace and a suggested number of hours to log.
---

# Worklog

Scan everything produced on the **target date** across GitHub (and Jira/Confluence
when the Atlassian MCP is connected), then produce a per-project summary, ask about
untracked work, and suggest how many hours to log.

## Required flow

Always run in this order. Never skip a step:

1. Normalize the target date (ask if not given).
2. Collect data from the integrations (sections 1–3).
3. **"Other work" checkpoint** (section 4): ask the user and **wait for the answer**.
4. Estimate hours (section 5).
5. Write the final report (Output section) — only after steps 3 and 4.

## Target date

Accept any format (`2026-06-02`, `06/02`, "today", "yesterday") and normalize to
`YYYY-MM-DD`. The window is the **local** day `00:00:00`–`23:59:59`. GitHub timestamps
are UTC: convert them to the local timezone (`date +%Z`) before filtering.

```bash
gh api user --jq .login
```

## 1. GitHub (`gh` CLI)

Main sweep via the Events API (commits, PRs, reviews, comments; includes private
events when authenticated as yourself):

```bash
DAY=2026-06-02
gh api --paginate /users/<login>/events --jq ".[] | select(.created_at | startswith(\"$DAY\"))"
```

Also query the day before and after in UTC when the local offset pushes events across
midnight.

| Event | Activity |
|-------|----------|
| `PushEvent` | Commits (`payload.commits[].message`) + repo |
| `PullRequestEvent` | PR opened/updated/merged (`payload.action`) |
| `PullRequestReviewEvent` | Review by me (`payload.review.state`) |
| `PullRequestReviewCommentEvent` | Comment on a PR diff |
| `IssueCommentEvent` | Comment on a PR/issue |
| `CreateEvent` | Branch/tag created |

Complement and dedupe (by URL/sha) with targeted searches:

```bash
gh search prs     --author=@me      --created=$DAY     --json title,url,repository,state
gh search prs     --author=@me      --updated=$DAY     --json title,url,repository,state
gh search prs     --reviewed-by=@me --updated=$DAY     --json title,url,repository
gh search commits --author=@me      --author-date=$DAY --json repository,sha,commit
```

Group by **repository**.

## 2. Jira (Atlassian MCP, optional)

Skip silently if the Atlassian tools are not available. Use
`getAccessibleAtlassianResources` for the `cloudId`, then `searchJiraIssuesUsingJql`
(JQL dates use `yyyy/MM/dd`):

```
(assignee = currentUser() OR status CHANGED BY currentUser() AFTER "2026/06/02" BEFORE "2026/06/03")
AND updated >= "2026/06/02" AND updated <= "2026/06/02 23:59"
ORDER BY updated DESC
```

For each candidate, `getJiraIssue` with `expand=changelog` and `fields=comment`, and
keep only status changes and comments **made by me on the target date**. Group by
project key.

## 3. Confluence (Atlassian MCP, optional)

`searchConfluenceUsingCql` (CQL dates use `yyyy-MM-dd`):

```
(creator = currentUser() OR contributor = currentUser()) AND type = page
AND lastModified >= "2026-06-02" AND lastModified < "2026-06-03"
```

Distinguish **created** from **edited** by comparing the creation date with the target
date. Group by space.

## 4. "Other work" checkpoint (mandatory)

Integrations only see the digital trail. **Before** writing the report, ask with
`AskUserQuestion` (both questions in one call):

1. **Meetings** — "Any meetings or calls on <date>?"
   Options: "No meetings" + "Other" (which, with whom, how long).
2. **Untracked work** — "Anything the integrations can't see?"
   E.g. environment setup, research, pairing, helping a teammate, local docs, devops.
   Options: "Nothing else" + "Other" (describe, with approximate time).

Rules:
- Never write the final report before the user answers.
- If the answer is "nothing", omit the section — but never skip the question.
- Never invent items; log only what the user reports, with the stated duration.

| Rationalization | Reality |
|-----------------|---------|
| "The data shows no meetings" | The question exists for what leaves no trace. Always ask. |
| "I'll write the report and ask later" | A report without this has wrong hours. Ask first. |

## 5. Hours

Suggest the value to log, showing the reasoning:

1. **Active window**: first to last activity timestamp of the day (local time), across all sources.
2. **Breaks**: subtract gaps longer than ~90 min with no events.
3. **Other work**: add meetings and untracked work that fall outside the active window; don't double count what falls inside.
4. **Weighting**: commits/PRs/reviews/tickets/pages signal focused work. Use them to adjust the range, not to inflate it.
5. **Result**: a range plus a suggestion rounded to 0.5h (e.g. range ≈ 6–7h → **suggestion: 6.5h**). Say clearly it is an estimate.

With several projects in the day, split the suggestion proportionally to each project's load.

## Output

Write the report in the user's language. Omit empty sections and sources.

Each bullet states **what** was done and the **substance** (which problem, which
solution, which module), not just counts:
- Bad: "3 commits on branch X"
- Good: "`fix/PROJ-512-gateway-validation` [PROJ-512] — 3 commits (api): payload validation fix, null edge-case test, error handler refactor; PR #142 opened."

Tickets always with key **and** short title; status as `From → To`. Merge repos, Jira
projects and Confluence spaces that are the same project.

Sections, separated by `---`:

1. **Header** — full date with weekday.
2. **Summary** — 2–4 sentences with the thread of the day.
3. **Activities by project** — subsections per source (GitHub / Jira / Confluence / Other).
4. **Hours** — reasoning, range and highlighted suggestion.
5. **Ready to paste** — compact version per project, no links, ready for the timesheet tool.
