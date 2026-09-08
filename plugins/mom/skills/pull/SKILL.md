---
name: pull
description: Brings a momdeploy project's latest code into the working copy with `momdeploy pull` — what somebody else pushed from another machine, another coder or another agent — and sorts out a branch that has diverged from the platform. Use when the user asks to update, refresh, sync or pull the project, wants the latest changes, says somebody else worked on it, or when `momdeploy push` is refused because the platform has commits this branch does not.
compatibility: The momdeploy CLI ships inside this plugin and is run by its full path; git and a shell are still needed. A directory linked to a project, cloned or initialised earlier.
---

# Pulling a momdeploy project

A project can have more than one pair of hands on it: the coder on a second machine, a coder the
project was shared with, an agent in its own sandbox. Each of them pushes to the same project, and
`momdeploy pull` brings what they pushed into this directory. It fetches the current branch from
the project's git and fast-forwards to it. It commits nothing and deploys nothing; pulling is not
a deploy, and `/mom:deploy` still waits for the user to ask.

Never mention Kubernetes, Helm, ingress or the platform's GitLab to the user: those are internal
details the CLI deliberately hides.

## How to run the CLI

The CLI ships with this plugin and is **not** on the `PATH`. Run it by its full path:

```
"${CLAUDE_PLUGIN_ROOT}/scripts/momdeploy" pull
```

A bare `momdeploy` is not a command here and fails with `command not found`; if you see that, you
dropped the path. `Error: no token: run momdeploy auth` means nobody is signed in: run
`momdeploy auth`, relay the code and the link it prints to stderr, and wait for the user to approve
the login in a browser.

## Step 1: Pull and read the outcome

Run it in the project's directory. The report ends one of five ways, and each says what it means:

- `Pulled N commits into <branch>; <branch> is now at <sha>`, followed by the commits' subjects
  and the number of files changed — the code moved forward. Tell the user what came in.
- `Already up to date` — nobody pushed anything new. Nothing to do.
- `Nothing to pull: <branch> is ahead of the platform by N commits` — this directory has commits
  the platform does not. That is what `momdeploy push` sends, and only the user decides when.
- `Nothing to pull: the platform has no branch <branch> yet` — the platform never saw this
  branch. Usually the user is on a branch of their own; do not switch branches for them.
- `<branch> and the platform have diverged` — step 2.

`--json` gives the same as data: `before`, `after`, `commits`, `files`, `ahead`, `rebased` and
`log`. Parse that rather than the text when you need the numbers.

## Step 2: A diverged branch

The directory has commits of its own and so does the platform. A plain pull refuses on purpose:
it never creates a merge commit. Tell the user both counts from the message, then run

```
"${CLAUDE_PLUGIN_ROOT}/scripts/momdeploy" pull --rebase
```

which replays this directory's commits on top of the platform's. It is safe to try: a rebase that
runs into a conflict is undone by the CLI itself, and the directory comes back exactly as it was.
The error then names the conflicting files and the git commands to resolve them by hand. Report
the files to the user and stop; resolve conflicts yourself only if they ask you to.

## Step 3: Uncommitted changes in the way

Git will not overwrite the user's uncommitted edits to a file the platform also changed, and the
CLI says so. Do not discard them and do not stash them silently. Say what is in the way and offer
the two ways out: commit the edits, or set them aside with `git stash`, pull, then `git stash
pop`. Do either only when the user agrees.

## After the pull

Say what arrived, in the words of the commit subjects, and whether anything the user was working on
is affected. Then continue with whatever they asked for. Pulling changes nothing on the platform:
deploying is `/mom:deploy`, and only when the user asks.

## When a command refuses

| What the CLI says | What to do |
| --- | --- |
| `not linked: run momdeploy init in the directory with the code` | You are outside the project directory. `cd` into it. |
| `the repository has no commits yet` | Nothing to pull into. `momdeploy project clone` into an empty directory brings the code down (`/mom:clone`). |
| `HEAD is detached` | `git switch <branch>`, then pull again. |
| `remote momdeploy is missing` | `git remote add momdeploy <git_url>`; `momdeploy project list --json` shows the address. |
| `the project's git rejected the momdeploy token` | The token expired: `momdeploy auth`, then pull again. |
| `… have diverged …` | Step 2: `pull --rebase`. |
| `cannot pull over your uncommitted changes …` | Step 3: commit or stash, with the user's agreement. |
| `your commits conflict with the platform's in …` | The rebase was undone. Report the files; resolve by hand only if asked. |

Results go to stdout, progress and prompts to stderr.
