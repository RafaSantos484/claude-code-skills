# Claude Code Skills

A small collection of custom [Claude Code](https://docs.claude.com/en/docs/claude-code) skills — reusable instruction sets that Claude loads automatically when a task matches, so it follows a consistent, battle-tested procedure instead of improvising each time.

## What's here

| Skill | What it does |
|---|---|
| [`gitlab-api`](gitlab-api/) | Drives GitLab's REST and GraphQL APIs — issues, merge requests, branches, pipelines, labels, epics/work items, members. Authenticates only via a `GITLAB_TOKEN` environment variable (never from files or CLI config), auto-detects the GitLab host from context instead of assuming `gitlab.com`, and knows when to prefer GraphQL over REST (e.g. epics, vulnerabilities — resources where REST is deprecated or incomplete). |
| [`pr-review`](pr-review/) | Turns Claude into a rigorous, evidence-only code reviewer against a live git repository — reads the repo's own conventions first (`CLAUDE.md`, linter config, etc.), verifies claims by actually running tests/lint instead of guessing, and outputs a structured review with severity-tagged findings. |
| [`export-user-prompts`](export-user-prompts/) | Exports just the human-authored messages from the current session's transcript to a plain `.txt` file — useful for handing a clean, unbiased requirements doc to a separate review session without leaking the original session's own reasoning. |
| [`sonarqube`](sonarqube/) | Runs a SonarQube analysis (coverage generation plus the SonarScanner CLI via Docker) and triages what it reports — issues, coverage, duplication — via the Web API. Authenticates only via a `SONAR_TOKEN` environment variable, judges the codebase exclusively on Overall Code metrics (never the New Code period or the quality-gate endpoint), and either fixes a valid finding in code or transitions a false positive/accepted one with a recorded justification — never both. Opt-in only: it runs when explicitly asked, not as a side effect of other work. |

## Installation

Claude Code loads skills from `~/.claude/skills/` (global, all projects) or `.claude/skills/` inside a specific repo (project-scoped).

```bash
git clone <this-repo-url> /tmp/claude-skills
cp -r /tmp/claude-skills/<skill-name> ~/.claude/skills/
```

Or clone straight into place:

```bash
git clone <this-repo-url> ~/.claude/skills-shared
ln -s ~/.claude/skills-shared/<skill-name> ~/.claude/skills/<skill-name>
```

Claude picks up a skill automatically once its folder is in a recognized skills directory — no restart or registration step needed. Each skill's `SKILL.md` frontmatter (`description:`) is what Claude matches against your request to decide when to trigger it.

## Structure

Each skill is a folder containing:

- `SKILL.md` — the entry point: frontmatter (`name`, `description`, optionally `allowed-tools`) plus the instructions Claude follows.
- `references/` (optional) — supplementary docs too detailed for the main file, which Claude reads on demand.
- `scripts/` (optional) — standalone helper scripts the skill points users to for repeatable manual runs (e.g. `sonarqube/scripts/run-scanner.sh`), as distinct from commands Claude runs itself inline.

## Contributing

These skills encode specific workflow opinions (e.g. `pr-review`'s evidence-only rule, `gitlab-api`'s env-var-only auth, `sonarqube`'s Overall-Code-only rule). If you adapt one, keep the reasoning ("why") comments intact where present — they exist so future edits don't accidentally undo a deliberate constraint.

## License

[MIT](LICENSE) — permissive, simple, and standard for this kind of content: mostly instructional markdown with embedded code/shell examples, no patent-sensitive material that would call for Apache 2.0, and no reason to restrict reuse or adaptation.
