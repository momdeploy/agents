---
name: deploy
description: Deploys code with momdeploy — creates or clones the project, writes momdeploy.yaml and a Dockerfile, pushes, follows the deploy and reports the URL. Use when the user asks to deploy, publish, ship or host this code, wants a URL for it, asks to push or send changes to momdeploy, mentions `momdeploy init`, `momdeploy push` or `momdeploy secret`, or asks why a momdeploy deploy failed. Not for a project someone merely hands over to work on — that is `/mom:clone`, which clones and stops before deploying.
compatibility: The momdeploy CLI ships inside this plugin and is run by its full path; git and a shell are still needed. Works wherever Claude runs shell commands against a working copy; a chat-only surface has nothing to push.
---

# Deploying with momdeploy

momdeploy takes finished code and returns a URL. A deploy needs three things: the directory is
linked to a project, `momdeploy.yaml` says what to run, and a `Dockerfile` at the repository root
says how to build it. Everything after `momdeploy push` — image build, cluster, domain, TLS — is
the platform's business.

Never mention Kubernetes, Helm, ingress or the platform's GitLab to the user: those are internal
details the CLI deliberately hides. A project is addressed by its name, its id and its domain.

Deploy when asked, and only then. Editing code, answering a question about the project or taking
over a project someone handed you is not a request to deploy: do not run `momdeploy push` on your
own after a change, however finished it looks. The user says "deploy", "ship", "publish" or runs
`/mom:deploy` when they are ready. A project handed over for work is `/mom:clone`'s job: it
clones, reports, offers a local run and stops.

Copy this checklist and tick items off as you go:

```
Deploy progress:
- [ ] Step 1: CLI present and signed in
- [ ] Step 2: project linked, created or cloned
- [ ] Step 3: momdeploy.yaml and Dockerfile at the repository root
- [ ] Step 4: secrets set and declared
- [ ] Step 5: pushed and followed to the end
- [ ] Step 6: URL reported
```

## How to run the CLI

The CLI ships with this plugin and is **not** on the `PATH`. Run it by its full path:

```
"${CLAUDE_PLUGIN_ROOT}/scripts/momdeploy" --version
```

Every command below is written that way. A bare `momdeploy` is not a command here and fails with
`command not found`; if you see that, you dropped the path.

## Step 1: CLI and login

The CLI comes with this plugin. Do not install, download or build it, and do not look for it on
the user's machine: the binary above is the CLI, and it already points at the platform.

```
"${CLAUDE_PLUGIN_ROOT}/scripts/momdeploy" --version
"${CLAUDE_PLUGIN_ROOT}/scripts/momdeploy" project list --json
```

`Error: no token: run momdeploy auth` means nobody is signed in. Run `momdeploy auth`: it prints
a short code and a link **to stderr** and waits for the user to approve the login in a browser.
Relay both, leave the command running, and it exits by itself once approved. Add `--no-browser`
when the machine has no browser. The login is a browser step the user takes; you cannot complete
it for them.

A token in `MOMDEPLOY_TOKEN` replaces the login entirely — if the variable is set, do not run
`momdeploy auth`.

Never print a token, never write one into a file in the repository, never pass one as a command
argument. Do not pass `--api-url` unless the user names a platform themselves: the address is
built into this CLI.

## Step 2: The project

Three ways in, depending on what is already here.

Look for `.momdeploy/project.json`. It links this directory to its project, it is committed with
the code, and it holds no secrets. If it exists, the project exists — go to step 3. Do not edit
this file by hand.

### The code is here and has no project

Create the project from the directory that holds the code:

```
"${CLAUDE_PLUGIN_ROOT}/scripts/momdeploy" init <name> --json
```

The name is lowercase letters, digits and dashes, up to 63 characters. It is a label: it need not
be unique, and the domain comes from the identifier the platform draws, not from the name. Without
an argument the name comes from the directory name. `init` creates the project,
writes a `momdeploy.yaml` skeleton (an existing one is left alone), writes the link file, runs
`git init` if needed, adds the `momdeploy` remote and configures git to push with the momdeploy
token.

From the JSON: `project.domain` is the URL to report at the end, `git.remote_added` and
`git.credential_helper` say whether pushing will work. An empty `project.git_url` means the
platform has no git address yet and nothing can be pushed — say so and stop.

### The project exists and the code is not here

This is the other half of `init`. When the user asks to deploy a project whose code is not on
this machine, bring it down first:

```
"${CLAUDE_PLUGIN_ROOT}/scripts/momdeploy" project clone <project> [directory]
```

The project is named by its id, by its slug, or by the name `project list` shows. Two projects of
one name are refused with their slugs, which are unique — clone by slug then, do not guess.
Without a directory the project's name is used, and a directory that already has files in it is
never written into.

A project the user merely hands over — the dashboard's text ends in this command and asks for
nothing else — is not a deploy: `/mom:clone` clones it, reports and stops, and this skill comes
in later, when the user asks. Either way the clone's report says what the text does not: `URL:`
is where the project answers, and `Access: read-only` appears when the project was shared without
the right to push (`"role": "viewer"` in `--json`). Read the report rather than asking the user
for those facts.

The clone arrives ready to push: the remote is called `momdeploy` rather than `origin`, git is
configured to take the momdeploy token from this CLI, and the link file is written unless the
repository already carries one. From there step 3 and step 5 work as they do anywhere else.

An empty repository clones too, and that is a second way to start a project: clone it, write the
code and the two files, push.

Projects other people shared with you are reachable the same way. `project list` shows them, and
`clone` takes them; only the owner can invite or revoke.

## Step 3: The two required files

`momdeploy push` refuses to push without both, because the build would fail where the user cannot
see the logs.

**`momdeploy.yaml`** at the repository root — what to run. Read the code first: language, start
command, the port the server listens on, background workers, scheduled jobs, environment
variables, which of them are secrets. The format is strict, and an unknown key stops the deploy;
the `/mom:manifest` skill has the full v1 specification and the rules for names,
`env` and `secrets`. Follow it rather than guessing.

**`Dockerfile`** at the repository root — how to build. Write one that matches the code:

- multi-stage: build in one stage, copy only the artefacts into a slim runtime image;
- install production dependencies from the lockfile that is in the repository;
- the process must listen on `0.0.0.0`, on the same port as `port:` in the manifest;
- no secrets, `.env` files or credentials in the image — secrets arrive at run time;
- the final stage must contain whatever the manifest's `command:` needs to run.

## Step 4: Secrets

Values the code needs at run time and the repository must not contain. The rule that matters: **a
secret value never appears in a command argument** — arguments end up in shell history, in `ps`
and in this transcript. Read them from a file or from stdin:

```
"${CLAUDE_PLUGIN_ROOT}/scripts/momdeploy" secret put DATABASE_URL --from-file ./db-url.txt
"${CLAUDE_PLUGIN_ROOT}/scripts/momdeploy" secret put --from-env-file .env
printf '%s' "$DATABASE_URL" | "${CLAUDE_PLUGIN_ROOT}/scripts/momdeploy" secret put DATABASE_URL
```

If the value is not already on disk or in the environment, do not invent a way to obtain it: ask
the user to run the command themselves and carry on with the rest.

The `/mom:secrets` skill covers the input modes, deletion and rotation in full.

Every secret must also be listed by name under `secrets:` in `momdeploy.yaml`. The platform
injects declared names only, and a name that is declared but never set stops the deploy.
`momdeploy secret list` shows names, never values. A changed secret reaches the project with the
next push.

## Step 5: Push and follow

```
"${CLAUDE_PLUGIN_ROOT}/scripts/momdeploy" push -m "<what changed>"
```

This stages everything, commits, pushes the current branch, then follows the deploy and prints the
log until it ends. Exit code `0` means the application was updated; `1` means the build failed, a
newer push took over, or the deploy never started. Use `--no-wait` only when the user explicitly
does not want to wait.

`Nothing to push: <branch> is already at <sha>` with exit code `0` is a normal outcome, not an
error: there was nothing new to send.

A project shared read-only cannot be pushed — the platform refuses, and retrying changes nothing.
Make the changes, then hand them to the owner as a diff or a branch instead of pushing.

A push refused because the platform has commits this branch does not means somebody else pushed
first. Bring their commits down with `momdeploy pull` (`/mom:pull` covers a diverged branch), then
push again.

Build logs are not available to users — the log gives step names and their outcome. When a step
fails, look for the cause in the manifest and the Dockerfile, fix it, and push again.

## Step 6: Report

Tell the user the URL (`project.domain` from `init`, the `URL:` line of `project clone`, or
`momdeploy project list`), what was deployed, and what is left for them to do: set a secret,
point a domain, fill a placeholder.

## When a command refuses

| What the CLI says | What to do |
| --- | --- |
| `no token: run momdeploy auth` | Run `momdeploy auth`, relay the code and link, wait. |
| `not linked: run momdeploy init in the directory with the code` | You are outside the project directory, or the project was never created. |
| `not a git repository` | `git init`, then re-run `momdeploy init` or add the remote it names. |
| `remote momdeploy is missing` | `git remote add momdeploy <git_url>`; `momdeploy project list --json` shows the address. |
| `momdeploy.yaml and Dockerfile are missing at the repository root` | Write the missing file — step 3. |
| `HEAD is detached` | `git switch -c main`, then push again. |
| `the project's git rejected the momdeploy token` | The token expired: `momdeploy auth`, then push again. |
| `Access: read-only` in the clone report, or a push refused with `403` | The project was shared as viewer. Do not retry; hand the changes to the owner. |
| `deploy failed at the build` | The build broke: check the manifest and the Dockerfile against the failing step's name. |
| `a newer push took over` | Another push superseded this one; follow that deploy instead. |
| `the platform has commits this branch does not — somebody else pushed` | `momdeploy pull` first (`--rebase` when this branch has commits of its own), then push again. |
| `no project …: it does not exist, or nobody has shared it with you` | Wrong id, slug or name, or the project was never shared with this account. `project list` shows what is reachable. |
| `… projects are called "…"; clone one by its slug` | Two projects share that name. Use the slug the message prints. |
| `… is not empty; clone into another directory` | The target directory has files. Clone elsewhere, or use the directory that already holds the code. |
| the deploy never appeared | The push arrived but the platform did not pick it up; tell the user to check the dashboard. |

Every command takes `--json` for machine-readable output — parse that instead of the table
layout. Results go to stdout, progress and prompts to stderr.
