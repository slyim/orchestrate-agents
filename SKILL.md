---
name: orchestrate-agents
description: Plan, file and run a project's work as a team of parallel subagents (default Gemini 4 Neon via T3 delegate_task) while Claude acts as planner, reviewer and merger. Use this whenever the user asks to "use agents", "use neon agents", "orchestrate", "run multiple agents", "plan and run", "delegate this", "audit and fix the bugs", "turn this into issues", or hands over a list of features/bugs to implement in parallel — even if they don't say "orchestrate". Also use it when continuing such a session after agents report back.
---

# Orchestrate agents

You are the lead: you plan, write briefs, keep agents from colliding, review what comes back, merge, test, and report. Agents do the bulk of the coding. The user decides what ships.

If `references/<project>.md` exists for the project you're working on (e.g. `references/browse.md`), read it for that project's folders, build/test commands and release steps. `references/example-project.md` shows the shape of one; write a new one after the first session on a project so the next session starts informed.

## The loop

1. **Plan in issues.** Every unit of work gets a GitHub issue before or as it starts, so commits can close it and the user can see the board.
   - In-progress work: file short tracking issues yourself (title, why, checklist).
   - Unknown bugs: start a read-only **audit agent** that returns ranked, verified findings plus "fix batches" grouped by disjoint files/functions. File each finding as an issue (severity → priority label + milestone).
   - Big future features: start a **roadmap agent** that verifies feasibility against real APIs and writes issue drafts as JSON (`title, labels, priority, milestone, body`). Review, then file them with a script (create milestones + labels first), and file a pinned tracking issue whose checklist links every issue number.
   - Never put private data in issues (emails, screenshots that show accounts, local usernames).
2. **Partition the work so agents can't collide.** This is the part that goes wrong otherwise.
   - One agent that *builds* per folder. Two builders in one folder corrupt each other's `bin/obj` and test runs.
   - Read-only agents (audit, research) can share a folder; tell them to read committed code (`git show HEAD:<file>`) since others are editing the working tree.
   - Give parallel coders disjoint files, or disjoint methods in the same file, and name the other agent's territory in each brief.
   - Work that touches code another agent is mid-way through waits for the next round. Say so to the user rather than starting it.
3. **Start agents** (see "Starting agents" below), in the background. Don't poll; completions wake you.
4. **Review each result before trusting it.**
   - Read the diff size and the risky parts (anything that deletes/replaces files, handles downloads, auth, permissions, money, concurrency).
   - Verify claims that can be checked cheaply: e.g. icon codepoints against the official CSS, a downloaded asset's hash against the source.
   - Rerun the build and tests yourself after any merge; an agent's "exit 0" covers only its own copy.
   - If an agent ends without a real report, inspect its folder yourself (`git status`, diff, build, tests) instead of re-running it.
   - Fold small follow-ups (a one-line guard, a residual risk the agent flagged) into the code yourself rather than spawning another agent.
5. **Merge on a local branch.** Agents never commit. You do:
   - Commit a folder's work on the integration branch (e.g. `next`), with `Closes #n` lines.
   - For work done in a second folder (a synced copy): branch from the exact commit that copy was synced from, copy files over, `git checkout` any file whose diff is line-endings only (`git diff --ignore-cr-at-eol --stat` shows the real changes), commit, then merge.
   - If the integration branch's working tree is busy with a running agent, merge in a separate `git worktree` on a new branch and merge that back later.
   - Resolve conflicts by keeping both sides' intent; where a later round added safety (guards, toggles, exemptions), keep it around the code the other round changed. Build + full test suite before committing the merge.
   - After merging, re-sync the second folder from the merged branch before its next agent starts.
6. **Report** after each event in a compact form: what finished (issue links), what you verified and what wasn't verifiable, what's running (agent thread links), what's next. Keep it short; the user is watching many things.
7. **Ship only with a yes.** Pushing, tagging, releasing, publishing packages, or submitting to external registries need the user's explicit go-ahead in this session. Filing issues/labels/milestones on the user's own repo is fine when they asked for planning.

## Keep the user in the loop (they're learning from this)

The user wants to understand the project and the agents' work, not just receive results. So:

- **Ask at real decision points** with AskUserQuestion (short options, recommended first) instead of silently picking a default: product/UX choices (e.g. settings as a window vs a page, what a toggle does by default), scope trade-offs, anything user-visible or hard to undo. Don't ask about things the code or conventions already settle.
- **Before each round**, say in plain words what each agent will change and *where* in the code (file → class/method), and why the split avoids collisions.
- **After each merge**, explain the piece of architecture that changed in 2–4 sentences: which component owns it, how data/events flow, what a reader should look at first. Link `file:line`.
- **Offer a project map** when starting on a codebase or after big changes: components, what each file owns, key flows (startup, navigation, tab lifecycle, persistence), where tests live.
- Point to agent threads by link so the user can watch an agent work live.

## Starting agents

- Call `orchestrator_capabilities` once per thread (search the model name, e.g. `neon`). Default target: `providerInstanceId: "antigravity"`, `model: "gemini-4-neon"`, `options: {thought_level: "high"}` (`"xhigh"` for audits). If the user names another model/provider, use that.
- `delegate_task` can't target a different T3 project, and `isolation: "worktree"` only copies this thread's repo's *tracked* files. When the real code lives elsewhere, use plain shared isolation and give each agent its folder explicitly (a clone registered as its own project, plus a synced copy for a second parallel builder).
- Keep each agent's job to one coherent batch (a feature, or 3–6 related bugs).

### Brief template

Agents start with no context, so the brief carries everything:

```
Work ONLY in <folder> (<what it is, branch>). <One-line product description, stack>.
Files: <key files and what lives in each>. Read them first.
You are the only agent building in this folder. Another agent works on <their area> in <other folder> — stay out of <methods/files>.
Do NOT commit, push, or change git state. Do NOT touch <user's real data/profile> or kill <user's running app>; for manual tries use <safe sandbox, e.g. --profile <temp>>.

Task: <issue numbers — tell them to read `gh issue view <n> -R <repo>`>, smallest complete fix each, existing style.
- <per item: the defect/feature, the intended fix, edge cases that matter>

Tests: <what new checks to add, in which test files, and what each must prove>.
Verify: <exact build + test commands>, both exit 0.
Report: per issue what changed (file/method), how tested, final output lines, anything unverified.
```

Things worth stating explicitly because agents get them wrong otherwise: don't guess external facts (codepoints, package ids, API names) — look them up and cite; keep hooks into shared files small; reuse existing helpers; say which behavior from earlier rounds must be preserved.

## Lessons to keep

- An agent can die mid-task (provider quota, crash). Diff its folder against the commit it was synced from, check it builds, then ask the user which provider/model should continue (never switch silently) and brief the replacement to review and finish the partial work rather than restart. The same model is often available through a second provider with its own quota.
- Before a release, check what only shows up on CI: smaller screens, no GPU, different timing. Tests that assume your machine's screen will fail there.
- A user's screenshot of a bug is evidence: diagnose from code first (grep the handler), file the issue with cause + fix, then schedule it.
- Re-sync before re-use: a folder that an agent left must be synced to the merged state before the next agent starts there, or its next diff will silently revert merged work.
- Shell quoting breaks on big heredocs containing quotes; write generated scripts with the Write tool, then run them.
- Windows repos: files mix LF/CRLF; when patching by script, match the file's newline style.
- Record durable facts (repo layout, push target, user preferences) in memory once per session, not every turn.
