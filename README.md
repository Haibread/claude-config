# claude-config

Personal configuration for [Claude Code](https://claude.com/claude-code), also
read by [OpenCode](https://opencode.ai) and [pi](https://pi.dev): the global
instructions, the status line and a herdr hook.

## Description

[`CLAUDE.md`](CLAUDE.md) holds the rules Claude Code, OpenCode and pi apply to
every project unless a project's own instructions override one. The repository
doubles as Claude Code's `~/.claude` directory, which OpenCode reads too. It is
written for one person's machines and addresses that person directly, so
adopting it means rewriting it.

The conventions those rules defer to are Agent Skills, in a repository of their
own: [skills](https://github.com/leroyguillaume/skills), cloned into
`~/.claude/skills`.

Two shell scripts complete the setup:

- [`statusline-command.sh`](statusline-command.sh) is Claude Code's status
  line. It prints three lines: the model with a bar of the context window
  used, then a bar of the 5-hour rate limit with its reset time, and a bar of
  the 7-day rate limit with its reset day and time. A rate-limit line is
  omitted when Claude Code reports no such limit. Bars turn yellow from 50%
  and red from 80%.
- [`hooks/herdr-tab-title.sh`](hooks/herdr-tab-title.sh) is a `Stop` hook for
  sessions run in [herdr](https://herdr.dev): it renames the tab after the
  terminal title Claude Code gives the session, so a tab reads "Fix the flaky CI job"
  instead of "2". It only renames a tab that holds that one pane and
  still has herdr's default numeric label or the label the hook gave it for
  another session, so a label someone chose is never overwritten. A tab keeps
  the first title of each session and takes the next one's after a `/clear`:
  the hook records the label it sets and the session it set it for under
  `$XDG_STATE_HOME/claude/herdr-tab-title/` (`~/.local/state` by default). It
  does nothing in a subagent, outside herdr, or before the session has a title
  of its own.

## Getting started

### Prerequisites

- `git`.
- [`jq`](https://jqlang.org/download/), for the status line; `jq` and herdr
  for the tab-title hook.
- [Claude Code](https://docs.claude.com/en/docs/claude-code),
  [OpenCode](https://opencode.ai/docs/) or [pi](https://pi.dev).
- On Windows, [Git for Windows](https://gitforwindows.org): the scripts are
  bash, and run in its Git Bash, with `jq` on its `PATH`.

### Installation

Claude Code reads its configuration from `~/.claude`. Move any existing
`~/.claude` aside first, then clone:

```bash
mv ~/.claude ~/.claude.bak
```

```bash
git clone git@github.com:leroyguillaume/claude-config.git ~/.claude
```

The [`.gitignore`](.gitignore) is an allow-list: it ignores everything except
what the repository versions, so Claude Code's sessions, caches and history
land beside the tracked files without showing up in `git status`. A deny-list
would have to grow every time Claude Code writes a new kind of state, and the
cost of forgetting one is a committed credential.

To keep an existing `~/.claude` and put it under version control instead,
initialise it in place:

```bash
cd ~/.claude
git init
git remote add origin git@github.com:leroyguillaume/claude-config.git
git fetch origin
git checkout -f main
```

Then install the skills into `~/.claude/skills`, which this repository ignores,
as the [skills README](https://github.com/leroyguillaume/skills#installation)
describes:

```bash
git clone git@github.com:leroyguillaume/skills.git ~/.claude/skills
```

#### OpenCode

OpenCode reads `~/.claude/CLAUDE.md` as its global rules when there is no
`~/.config/opencode/AGENTS.md`, and the skills in `~/.claude/skills/`. Clone
both repositories as above, and both apply.
`OPENCODE_DISABLE_CLAUDE_CODE=1` turns that off.

OpenCode's `task` tool takes no model: a sub-agent runs on the model of the
agent it is launched as. For the choice between Sonnet and Opus that
`CLAUDE.md` asks for at every launch, define one sub-agent per model in
`~/.config/opencode/opencode.json`, with ids from `opencode models`:

```json
{
  "agent": {
    "sonnet": {
      "mode": "subagent",
      "description": "Well-scoped work whose outcome is easy to check: search, mechanical edits, applying a known convention.",
      "model": "anthropic/claude-sonnet-5"
    },
    "opus": {
      "mode": "subagent",
      "description": "Work that needs judgement: design, debugging with no lead, review, trade-offs.",
      "model": "anthropic/claude-opus-5"
    }
  }
}
```

On Windows, OpenCode runs commands in PowerShell when it finds one. Point it
at Git Bash in the same file, so the skills' commands run:

```json
{
  "shell": "C:\\Program Files\\Git\\bin\\bash.exe"
}
```

#### pi

pi reads its global rules from its agent directory, `~/.pi/agent`, not from
`~/.claude`. Clone this repository as above, then link `CLAUDE.md` there as
`AGENTS.md`:

```bash
ln -s ~/.claude/CLAUDE.md ~/.pi/agent/AGENTS.md
```

pi reads the skills from `~/.agents/skills`, not `~/.claude/skills`: install
them there, as the
[skills README](https://github.com/leroyguillaume/skills#pi-as-agentsskills)
describes.

pi has no sub-agents of its own. The `Agent` call `CLAUDE.md` delegates to
comes from an extension, such as
[pi-subagents](https://www.npmjs.com/package/@tintinweb/pi-subagents), whose
`Agent` tool takes `model` by name:

```bash
pi install npm:@tintinweb/pi-subagents
```

The status line and the herdr hook are Claude Code's, and pi runs neither.

### Configuration

`CLAUDE.md` is read by Claude Code, by OpenCode from `~/.claude`, and by pi
through the link above. For any other client, copy the rules you want into its own instructions file (such as
`AGENTS.md`).

The rest goes in `~/.claude/settings.json`, which the repository does not
version.

To show the status line, point Claude Code at the script:

```json
{
  "statusLine": {
    "type": "command",
    "command": "bash ~/.claude/statusline-command.sh"
  }
}
```

To name herdr tabs after their session, register the hook under `Stop`. It
runs in the background, so it never delays the end of a turn:

```json
{
  "hooks": {
    "Stop": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "sh ~/.claude/hooks/herdr-tab-title.sh",
            "async": true
          }
        ]
      }
    ]
  }
}
```

Both scripts sit in `~/.claude` when the repository is cloned there; otherwise
use the path of your copy.

Claude Code lists every skill's name and description to the model within a
size budget, a fraction of the model's context window. The descriptions of
the skills together exceed the default, and past it the least-used skills are
listed without their description, so the model sees less of when to load them.
Raise the fraction:

```json
{
  "skillListingBudgetFraction": 0.02
}
```

The budget scales with the context window, so the fraction that fits depends
on the model. Run `/skill-doctor` in Claude Code to see what each skill costs
in the listing, and raise the fraction until it fits.

Claude Code ends commit messages with a co-author trailer and pull request
descriptions with a "Generated with" line; `CLAUDE.md` forbids both. An
instruction can lose to the harness's own, so turn them off at the source:

```json
{
  "attribution": {
    "commit": "",
    "pr": ""
  }
}
```

### Usage

`CLAUDE.md` applies unconditionally in Claude Code, OpenCode and pi, from the
next session on. To pull new revisions:

```bash
git -C ~/.claude pull --ff-only
```

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md).

## License

MIT — see [LICENSE](LICENSE).
