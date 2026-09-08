# momdeploy for Claude

Say "ship this" and get a URL back. Claude creates the project, writes `momdeploy.yaml` and a
`Dockerfile`, pushes the code and reports the address it came up on. Or it picks up a project that
already exists and continues from there.

This repository is a Claude plugin marketplace with one plugin in it. The momdeploy CLI ships
inside the plugin, so there is nothing to install, download or configure first.

| Command | What it does |
| --- | --- |
| `/mom:clone` | Take over an existing project: sign in, clone the code, say where it landed and how to run it locally, and wait for you to ask before deploying |
| `/mom:deploy` | The whole path: sign in, create or clone the project, write the two required files, push, follow the deploy, report the URL |
| `/mom:pull` | Bring down what somebody else pushed to the project, and sort out a branch that diverged from it |
| `/mom:manifest` | Write, fix or review `momdeploy.yaml` and the `Dockerfile` next to it |
| `/mom:secrets` | Set, list, delete and rotate project secrets without a value ever reaching a command argument |

The skills are model-invoked as well, so "deploy this", "why did my deploy fail" or "store this
API key" reach them without anyone typing a slash command.

## Install

```sh
claude plugin marketplace add momdeploy/agents
claude plugin install mom@momdeploy
```

The same from inside a session, with `/plugin marketplace add momdeploy/agents` and
`/plugin install mom@momdeploy`. Restart the session to apply.

To update later:

```sh
claude plugin marketplace update momdeploy
claude plugin update mom@momdeploy
```

## Use

Point Claude at a directory with code and ask it to ship it. The first command that needs an
account starts the sign-in: the CLI prints a short code and a link, you approve it in a browser,
and the token lands on disk. Nothing is pasted into a config file.

To continue a project that already exists, give Claude its id, slug or name, or paste the
hand-over text from the dashboard. Claude clones the code, tells you where it landed and how to
run it locally when the README says, and deploys only when you ask: `/mom:deploy`. Editing code
is not a deploy.

## What is inside

```
.claude-plugin/marketplace.json   the marketplace
plugins/mom/
├── .claude-plugin/plugin.json    the plugin manifest
├── scripts/momdeploy             launcher; picks the build for the current OS
├── platform/                     the CLI, one binary per platform
└── skills/{clone,deploy,manifest,pull,secrets}
```

The CLI covers `auth`, `init`, `project create | list | clone`, `push`, `pull` and
`secret put | delete | list`.

## The CLI

`scripts/momdeploy` is a POSIX script, not a binary. It resolves the real CLI in two steps and
execs it:

1. `MOMDEPLOY_BIN`, when it points at an executable, for testing a build of your own;
2. `platform/momdeploy-<os>-<arch>` otherwise.

Picking at run time is not decoration. The shell that runs the command is not always the machine
the plugin was installed on: in Claude Code it is your own operating system, and in Cowork shell
commands run inside a Linux virtual machine. A single macOS binary would fail there.

Builds shipped: `darwin/arm64`, `darwin/amd64`, `linux/amd64`, `linux/arm64`, `windows/amd64`.
There is no `windows/arm64` build; set `MOMDEPLOY_BIN` there. Windows needs Git Bash or another
POSIX shell for the launcher itself.

### Where it points

The platform address is compiled into the binaries, so a fresh install needs no configuration. The
CLI resolves the address in this order, first match winning:

1. the `--api-url` flag;
2. `MOMDEPLOY_API_URL`;
3. the address saved at login;
4. the address compiled into the binary.

Someone who logged in against another platform keeps that platform until they log in again. This
is deliberate: silently redirecting a saved login would pair one platform's address with another
platform's token, and every call would fail with an authorisation error nobody could explain.

### Signing in

There is no token to configure. `momdeploy auth` prints a short code and a link to stderr and
waits; you approve in a browser, and the CLI writes the token with mode `0600` to the platform's
config directory. On macOS that is `~/Library/Application Support/momdeploy/credentials.json`, on
Linux `~/.config/momdeploy/credentials.json`. `MOMDEPLOY_TOKEN` replaces the login entirely, which
is the path for CI and for agents.

The plugin deliberately declares no `userConfig` form. Values from that form reach hook and monitor
processes and are substituted into MCP and LSP server configuration; they do not reach a command
the Bash tool runs, which is how this CLI is invoked. A form with an address and a token would look
like it worked and would change nothing.

### Why the CLI is not on the PATH

A plugin's `bin/` directory is added to the Bash tool's `PATH`, which would make `momdeploy` a bare
command. claude.ai refuses to host a plugin that has one:

```
Plugin contains a top-level bin/ directory ('bin/momdeploy'). claude.ai-hosted plugins may not
ship bin/ executables because they are added to PATH on the CLI but are not shown on the admin
approval surface. Declare executable entry points via hooks, commands, or mcpServers instead.
```

The reason is review, not packaging: an executable that silently joins the `PATH` never appears on
the surface an administrator approves. So the launcher lives in `scripts/`, and the skills call it
by absolute path, as `${CLAUDE_PLUGIN_ROOT}/scripts/momdeploy`. Claude Code substitutes that
variable in skill text before Claude reads it, so what reaches the model is a real path.

## Developing

Load the plugin straight from a checkout, without installing it:

```sh
claude --plugin-dir ./plugins/mom
```

After editing a skill, `/reload-plugins` picks it up without a restart. Before publishing, run:

```sh
claude plugin validate . --strict
```

### Releasing

1. rebuild the binaries from the CLI repository, pinned to the platform:
   `make release VERSION=<v> API_URL=https://api.momdeploy.com`
2. copy `dist/momdeploy-*` into `plugins/mom/platform/`
3. bump `version` in `plugins/mom/.claude-plugin/plugin.json`
4. `claude plugin validate . --strict`
5. commit and push

Bumping the version is not optional. Clients keep their cached copy until that string changes, so
a release without a bump reaches nobody.

The version lives only in the plugin manifest. Declaring it in the marketplace entry as well would
be masked without warning: the manifest value always wins.
