# Contributing

This repository holds personal Claude Code configuration: the global
instructions in `CLAUDE.md`, the status line script
[statusline-command.sh](statusline-command.sh) and the herdr hook
[hooks/herdr-tab-title.sh](hooks/herdr-tab-title.sh). Open an issue or a pull
request on <https://github.com/leroyguillaume/claude-config>; the short version
is `task lint` and `task test` must be green before you push.

See [README.md](README.md#getting-started) for installing and using it. The
skills have their own repository and contributing guide:
<https://github.com/leroyguillaume/skills>.

## Development setup

[mise](https://mise.jdx.dev/installing-mise.html) is the one tool to install
beyond `git`, activated in your shell as its install page describes:

```bash
brew install mise
```

```bash
curl https://mise.run | sh
```

It installs everything else at the versions [mise.toml](mise.toml) pins, with
their checksums in [mise.lock](mise.lock):

- [pre-commit](https://pre-commit.com) runs the gate, and the Python it
  builds its hook environments with.
- [jq](https://jqlang.org): the hook tests and the status line call it.
- [Task](https://taskfile.dev) runs the commands below, defined in
  [Taskfile.yml](Taskfile.yml); `task --list` shows them.

pre-commit builds the `actionlint` hook from source with the Go on `PATH`, and
downloads a toolchain of its own when there is none.

From a fresh clone:

```bash
mise trust
```

```bash
mise install
```

```bash
task setup
```

`task setup` installs both the `pre-commit` and the `commit-msg` hook types, as
`default_install_hook_types` in the config asks.

The setup is good when this is green:

```bash
task lint
```

## Editing the configuration

1. A new file at the repository root is ignored until it gets its own
   `!/<name>` line in the allow-list [.gitignore](.gitignore).
2. Do not list the skills in [CLAUDE.md](CLAUDE.md): clients list the
   available skills themselves, and a copy there only goes stale.
3. Give each rule one home. `CLAUDE.md` holds what applies to every change
   whatever the technology — meta-rules and guardrails — and never restates a
   skill's convention: where it touches a skill's topic, it says so in one
   line and names the skill.

## Running the tests

Both scripts have a test suite under [tests/](tests/), and a change to a
script comes with the case that covers it. Each needs only `jq`. Run both:

```bash
task test
```

[.gitattributes](.gitattributes) checks shell scripts out with LF line
endings everywhere: Git for Windows converts to CRLF by default, and bash
then fails on the trailing `\r` of every line.

The herdr hook's suite runs the hook against a fake `herdr` placed first on
`PATH`, which serves canned JSON and logs every call, so it needs neither
herdr nor a session; each case covers one condition the hook checks. Run it
alone with:

```bash
tests/herdr-tab-title.sh
```

The status line's suite pipes it sample payloads of what Claude Code sends,
with the host's `date` and with a fake that only takes the BSD form, and
under a comma-decimal locale when one is installed. Run it alone with:

```bash
tests/statusline-command.sh
```

To see the coloured output, pipe it a sample yourself:

```bash
echo '{"model":{"display_name":"Opus"},"context_window":{"used_percentage":42},"rate_limits":{"five_hour":{"used_percentage":61,"resets_at":1893456000}}}' | ./statusline-command.sh
```

## Pre-commit hooks

`task setup` installs the hooks; they must pass before you push.

```bash
task lint
```

```bash
pre-commit run <hook-id> --all-files
```

The hooks are listed in [.pre-commit-config.yaml](.pre-commit-config.yaml);
most are formatters or checks that name the file to fix. These need more:

| Hook | What it checks | How to fix |
| --- | --- | --- |
| `yamllint` | YAML style, per [.yamllint.yaml](.yamllint.yaml), with `--strict` so warning-level rules fail too | Fix the finding; block style only, no `---` opening a file |
| `shellcheck` | Shell scripts, with [shellcheck](https://www.shellcheck.net) bundled by the hook, nothing to install | Fix the finding it names |
| `actionlint` | Workflow files are valid | Fix the expression or key it names |
| `mise-fmt` | `mise.toml` is formatted, with the `mise` on `PATH` | Re-run: it rewrites the file |
| `no-co-authors` (`commit-msg` stage) | No `Co-Authored-By:` trailer or `Generated with` line in the commit message. Runs on `git commit` only: `pre-commit run --all-files`, and so CI, skips it | Rewrite the message; a commit has one author |

`--no-verify` and `SKIP=` are not the fix. A red hook is a real finding: fix
the content, or change the hook configuration in the same pull request and say
why. The same hooks run in CI, so skipping locally only moves the failure
somewhere slower.

## Continuous integration

| Workflow | What it does | Reproduce locally |
| --- | --- | --- |
| [`quality.yaml`](.github/workflows/quality.yaml) | Every job installs its tools with mise. `pre-commit` job: `task lint`. `tests` job: `task test`, on Linux and on Windows under Git Bash, where Claude Code and OpenCode run the scripts | `task lint`, `task test` |
| [`renovate.yaml`](.github/workflows/renovate.yaml) | Weekly self-hosted Renovate run on [renovate.json](renovate.json); `workflow_dispatch` takes a `dryRun` and a `logLevel` | `workflow_dispatch` with `dryRun` |

`quality` needs no secret and only reads the repository, so it runs the same
on a fork's pull request. `renovate` writes with a `RENOVATE_TOKEN` secret, a
token allowed to push branches and open pull requests here: a pull request
opened with `GITHUB_TOKEN` would not trigger `quality`.

Renovate keeps every pin current: the action SHA pins, the pre-commit hook
revisions, and the tool versions in `mise.toml` and `mise.lock`. Pins are SHAs
and tags can move, so a bot bumps them rather than a person remembering to;
grouping folds the minor and patch bumps into one pull request per ecosystem,
each major gets its own, and nothing is automerged.

## Submitting a change

- Commit subjects are imperative, lowercase, with an optional `area:` prefix —
  read `git log --oneline` and follow what is there.
- Pull requests are squash-merged, so the pull request title becomes the
  commit subject on `main`: write it in the same style.
- A pull request needs green CI and no `TODO` left behind in a committed file:
  work that remains goes in an issue.
