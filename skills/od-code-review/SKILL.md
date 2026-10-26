---
name: od-code-review
description: Review a change with an architecture reviewer, a test reviewer and a QA reviewer, then open an HTML report.
disable-model-invocation: true
argument-hint: "[PR number | ref]"
---

# OD code review

Three reviewers judge one **review target** in parallel. Each owns one lane, and their findings stay in separate report sections: a clean architecture section must never hide a broken test.

## 1. Resolve the review target

Pick the mode from the argument:

| Argument | Mode | BASE | Diff command | Read a file |
|---|---|---|---|---|
| A number | PR | `git fetch origin <baseRefName>` from `gh pr view <n> --json baseRefName,headRefName,title,url`, then `BASE=origin/<baseRefName>` | `git fetch origin pull/<n>/head`, `HEAD_SHA=$(git rev-parse FETCH_HEAD)`, then `git diff BASE...HEAD_SHA` | `git show HEAD_SHA:<path>` |
| A ref (branch, commit, tag) | Ref | `git rev-parse --verify <ref>^{commit}` | `git diff <ref>...HEAD` | `git show HEAD:<path>` |
| Nothing | Branch work | `git merge-base <default> HEAD`, where `<default>` is `git symbolic-ref --short refs/remotes/origin/HEAD`, else `main`, else `master` | `git diff BASE` plus the untracked files from `git ls-files --others --exclude-standard` | Read the working-tree file |

PR mode leaves the working tree alone: it fetches, it never checks out.

Also collect the commit list (`git log --oneline BASE..<head>`) and the changed-file list (`git diff --stat`).

**Done when** you hold BASE, the head (`HEAD_SHA`, `HEAD`, or the working tree), the diff command, the read-a-file command, the commit list and the changed files. If a ref does not resolve, `gh pr view` fails, or the diff and untracked list are both empty, stop here and tell the user why, in one sentence.

## 2. Start the three reviewers

Send one message with three `Agent` calls, `subagent_type: general-purpose`, no model override. Each reviewer's prompt has exactly three parts, in this order:

1. **Target block**: mode, repo root path, BASE, head, diff command, read-a-file command, commit list, changed files, untracked files.
2. **Role line**: "You are the <role>. Read `<this skill's base directory>/reviewer-brief.md` and `<this skill's base directory>/<role-file>`, then follow them."
3. **Return line**: "Return your findings in the shape the brief defines, and nothing else."

| Role | Role file |
|---|---|
| Architecture reviewer | `architecture-reviewer.md` |
| Test reviewer | `test-reviewer.md` |
| QA reviewer | `qa-reviewer.md` |

**Done when** all three reviewers have returned.

## 3. Build the HTML report

Invoke the `show-me:show-me` skill for its diagram guidance. Then fill [`report-template.html`](report-template.html):

- Each reviewer's findings go into that reviewer's section, in the order the reviewer returned them, with the reviewer's wording.
- A finding with a `Structure` field gets a Mermaid `graph` drawn from it, in the finding card. A `Fix` with a `diff` block goes in a `<pre class="diff">`. Every other finding stays text.
- HTML-escape all code, paths and diff text.
- The diagrams live inside the report page; `show-me`'s own "write a file and `open` it" step is replaced by step 4.

Write the finished page with one `Write` call to `/tmp/od-code-review/<repo>-<branch or prN>-<YYYYmmdd-HHMMSS>.html`, with `/` in the branch name replaced by `-`.

**Done when** every finding from all three reviewers is in the file, and the header counts match the cards.

## 4. Open it and summarise

Run `xdg-open <path> >/dev/null 2>&1 &`. Then reply in the chat with only this:

```
<review target, one line>

| Reviewer | Blocking | Should-fix | Nit |
|---|---|---|---|
| Architecture | n | n | n |
| Tests | n | n | n |
| QA | n | n | n |

Blocking:
- [<reviewer>] `file:line`: <the finding's "What" sentence>

Report: <path>
```

Write "Blocking: none" when there are none. The full findings live in the report only.
