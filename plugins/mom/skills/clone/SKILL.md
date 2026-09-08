---
name: clone
description: Takes over an existing momdeploy project handed to you — clones its code with `momdeploy project clone`, reports where it landed and where it answers, offers to run it locally when the README says how, and stops there without deploying. Use when the user pastes the hand-over text from the momdeploy dashboard, gives a momdeploy project id, slug or name to clone, continue or work on, or mentions `momdeploy project clone`. Deploying is `/mom:deploy`, and only when the user asks for it.
compatibility: The momdeploy CLI ships inside this plugin and is run by its full path; git and a shell are still needed. Works wherever Claude runs shell commands against a working copy.
---

# Taking over a momdeploy project

A hand-over is not a request to deploy. The user gives you a project that already exists and
already answers on its domain; what they want is the code on disk and you working on it. This
skill ends when the code is cloned, the user knows where it is and how to run it locally, and the
way to deploy has been named. Nothing is pushed here: `momdeploy push` belongs to `/mom:deploy`,
and `/mom:deploy` runs when the user asks for it.

Never mention Kubernetes, Helm, ingress or the platform's GitLab to the user: those are internal
details the CLI deliberately hides. A project is addressed by its name, its id and its domain.

Copy this checklist and tick items off as you go:

```
Take-over progress:
- [ ] Step 1: CLI present and signed in
- [ ] Step 2: project cloned
- [ ] Step 3: user told where the code is and how the project stands
- [ ] Step 4: local run offered, when the README says how
- [ ] Step 5: user told how to deploy when ready
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

## Step 2: Clone

```
"${CLAUDE_PLUGIN_ROOT}/scripts/momdeploy" project clone <project> [directory]
```

The dashboard hands a project over as a short text that ends in exactly this command, with the
id, and says nothing else about the project. That is deliberate: the report the clone prints
carries the rest, so read it rather than asking the user.

- `Project <name> cloned into <directory>.` — where the code is. Without a directory argument the
  project's name is used, and a directory that already has files in it is never written into.
- `URL:` — where the project answers today.
- `Access: read-only` — the project was shared without the right to push (`"role": "viewer"` in
  `--json`). Without that line, pushing is allowed.
- `Files:` — whether `.momdeploy/project.json` was already in the repository or was written now.
  It links the directory to its project and holds no secrets. Do not edit it by hand.
- `The project's repository is empty` — nothing has been pushed yet: the project exists, the code
  does not.

The project is named by its id, by its slug, or by the name `project list` shows. Two projects of
one name are refused with their slugs, which are unique — clone by slug then, do not guess.
Projects other people shared with you are reachable the same way.

The clone is ready for later: the remote is called `momdeploy` rather than `origin`, and git is
configured to take the momdeploy token from this CLI. Nothing about that needs doing now.

## Step 3: Tell the user

In a few lines: the project is cloned into `<directory>`, it answers at `<URL>`, and it is ready
to work on. If the report printed `Access: read-only`, say the project is shared read-only:
changes can be made here, but only its owner can deploy them. If the repository was empty, say
that too. Change nothing yet, and push nothing.

## Step 4: Offer a local run

Look for how the project runs locally: `README.md` (or `README`, `readme.md`, a `docs/` folder),
then `Makefile`, `package.json` scripts and `docker-compose.yml`. If the README describes a local
start — install, dev server, docker compose, a make target — offer to run it, quoting the
commands you found, and wait for the answer: installs and servers are the user's call, so do not
start them unasked. If nothing describes a local start, say so and do not invent one. `command:`
in `momdeploy.yaml` is what runs on the platform, not a development server.

## Step 5: Name the way to deploy

Tell the user: when they are ready, `/mom:deploy` sends the changes to momdeploy — it commits
everything, pushes, follows the deploy and reports the URL. Until then the work is local: edit,
run, test. If somebody else works on the project too — another machine, another coder —
`/mom:pull` brings their pushes down before you continue.

Do not run `momdeploy push` on your own — not after an edit, not after the tests pass, not because
the change looks finished. A user who says "deploy", "ship", "publish" or "push to momdeploy", or
runs `/mom:deploy`, is asking for a deploy; a user who asks to change the code is not.

## When a command refuses

| What the CLI says | What to do |
| --- | --- |
| `no token: run momdeploy auth` | Run `momdeploy auth`, relay the code and link, wait. |
| `no project …: it does not exist, or nobody has shared it with you` | Wrong id, slug or name, or the project was never shared with this account. `project list` shows what is reachable. |
| `… projects are called "…"; clone one by its slug` | Two projects share that name. Use the slug the message prints. |
| `… is not empty; clone into another directory` | The target directory has files. Clone elsewhere, or pass a directory. |
| `the platform has no git address for project … yet` | There is nothing to clone from; tell the user to check the dashboard. |
| `the project's git rejected the momdeploy token` | The token expired: `momdeploy auth`, then clone again. |

Every command takes `--json` for machine-readable output — parse that instead of the table
layout. Results go to stdout, progress and prompts to stderr.
