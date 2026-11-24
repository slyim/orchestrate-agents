# orchestrate-agents

A [Claude Code](https://claude.com/claude-code) skill that turns Claude into the lead of a small team of coding agents: it plans the work as GitHub issues, runs parallel subagents (Gemini 4 Neon through T3 Code's `delegate_task` by default) in folders that can't collide, reviews and verifies what comes back, merges on a local branch, runs the tests, and asks before anything ships. It also keeps you in the loop: asks at real decision points and explains the parts of the codebase each round touches.

Built while developing [browse](https://github.com/slyim/browse), a small Windows browser: in one session it filed 50 issues and landed about 25 of them through rounds of parallel agents.

## Install

```sh
git clone https://github.com/slyim/orchestrate-agents ~/.claude/skills/orchestrate-agents
```

Claude Code picks it up automatically. Trigger it with things like "use agents for this", "orchestrate the bug fixes", "audit this repo and turn it into issues".

## What's inside

- `SKILL.md`: the loop (plan in issues → partition → start agents → review → merge → report → ship only with a yes), the agent brief template, and lessons learned.
- `references/example-project.md`: what a per-project reference looks like (folders, build/test commands, release steps). Write your own as `references/<project>.md`; `.gitignore` keeps them out of the repo.

## Requirements

- Claude Code with subagents. The default agent target uses T3 Code's orchestrator; any provider/model works, so tell Claude which.
- `gh` CLI for issues.

## License

MIT
