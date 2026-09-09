# skills

Claude Code skills, kept in one place so they're easy to share and reuse.

Each skill is a folder at the repo root containing a `SKILL.md`. Claude Code picks
them up automatically for anyone working inside a clone of this repo — no install step.

## Available skills

<!-- skills:start -->

| Skill | What it does | Requires |
| --- | --- | --- |
| [`basecamp`](basecamp/) | Interact with Basecamp via the Basecamp CLI — projects, todos, cards, messages, files, schedule, check-ins, and the rest of the API surface. | [`basecamp` CLI](#requirements), authenticated |
| [`create-basecamp-draft`](create-basecamp-draft/) | Turn a plain-text meeting summary into an unpublished Basecamp message draft, with real person mentions resolved from the project. | `basecamp` skill |

<!-- skills:end -->

## Using these skills

Clone the repo and run Claude Code from inside it:

```bash
git clone https://github.com/pedro-lopes-px/skills.git
cd skills
claude
```

Type `/` and the skills above appear in the list.

To use them **everywhere** instead of just in this repo, symlink them into your
user-level skills directory:

```bash
ln -s "$PWD/create-basecamp-draft" ~/.claude/skills/create-basecamp-draft
```

## Requirements

Some skills wrap external tools. Install what the skill you want needs:

- **Basecamp CLI** — required by `basecamp` and `create-basecamp-draft`.
  Verify with `basecamp auth status`.

