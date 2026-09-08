---
name: manifest
description: Writes, fixes and reviews momdeploy.yaml — the momdeploy project manifest, format v1, with the web, workers, jobs, env and secrets blocks — and the Dockerfile it requires. Use when creating or editing momdeploy.yaml, when a momdeploy deploy fails on the manifest or on an unknown key, or when the user asks what a momdeploy project has to declare.
compatibility: Reference material for momdeploy manifest format v1. No tools required.
---

# momdeploy.yaml, format v1

`momdeploy.yaml` sits at the repository root and describes the project in the user's own terms:
which processes to run, with which environment, on which schedule. The platform reads it on every
deploy. Cluster vocabulary — Kubernetes, Helm, ingress, replicas — is not part of this format and
must not appear in it.

The file is named exactly `momdeploy.yaml`; `momdeploy.yml` is not looked for. Indent with spaces
only. `momdeploy push` refuses to push unless both this file and a `Dockerfile` are at the
repository root.

## The CLI in this skill

Two commands appear below, in the section on secrets. The CLI ships with this plugin and is not
on the `PATH`, so both are written with its full path, `${CLAUDE_PLUGIN_ROOT}/scripts/momdeploy`.

## Full example

```yaml
version: v1

web:
  app:
    command: node server.js
    port: 3000
    health: /healthz

workers:
  queue:
    command: node worker.js
    env:
      QUEUE_CONCURRENCY: "4"

jobs:
  cleanup:
    schedule: "0 3 * * 1"
    command: node scripts/cleanup.js

env:
  MYSQL_DB_NAME: my-project

secrets:
  - MYSQL_PASSWORD
```

## version

Required. The only accepted value today is the string `v1` — no quotes needed. `version: 1` as a
number is an error. An unknown version is rejected outright.

## Processes

Three blocks, each a map of process name to description. The block a process sits in decides what
it is; there is no `type:` field. To turn a worker into a web service, move it into `web`.

Process names: lowercase letters, digits and dashes, starting with a letter, at most 63
characters.

### web — the HTTP process

Takes traffic and gets the project's domain. In v1 there is **exactly one** entry; two or more are
rejected. Two web services today means two projects.

| Field | Required | Meaning |
| --- | --- | --- |
| `command` | yes | Start command, run through `sh -c`, so pipes, `&&` and quoted arguments work. |
| `port` | yes | The port the process listens on. It must listen on `0.0.0.0`, not on localhost. |
| `health` | no | HTTP path probed for liveness, for example `/healthz`. |
| `env` | no | Variables for this process, on top of the root `env`. |

### workers — long-running background processes

Queue consumers, schedulers, anything without incoming traffic. Started alongside the web process
and kept running. A worker has no port and no domain; `port:` in a worker is a validation error
that tells you to move it to `web`.

| Field | Required | Meaning |
| --- | --- | --- |
| `command` | yes | Start command through `sh -c`. An empty string is an error. |
| `env` | no | Variables for this process, on top of the root `env`. |

### jobs — scheduled processes

Run on a schedule, do the work, exit.

| Field | Required | Meaning |
| --- | --- | --- |
| `schedule` | yes | Five-field cron (minute, hour, day of month, month, day of week) or `@hourly`, `@daily`, `@weekly`, `@monthly`. Always UTC. |
| `command` | yes | Start command through `sh -c`. An empty string is an error. |
| `env` | no | Variables for this process, on top of the root `env`. |

A new run does not start while the previous one is still going; a failing job is retried a limited
number of times; an overlong one is stopped by timeout. Those limits belong to the platform and
cannot be expressed in the manifest.

## env

The root `env` applies to every process. A process's own `env` adds to it, and wins on a clash.

Values are always strings, but quoting is optional: `PORT: 3000` and `DEBUG: no` become the
strings `"3000"` and `"no"` rather than a number, a boolean or an error. Values are literal —
`VAR_1: $HOME` is the five characters `$HOME`, there is no substitution.

Put non-sensitive configuration here. Anything a leak would hurt goes into `secrets`.

## secrets

A list of **names only**. Values never appear in this file, in the repository or in the image;
the platform stores them and injects them as environment variables into every process.

```yaml
secrets:
  - DATABASE_URL
  - SMTP_PASSWORD
```

Names are uppercase letters, digits and underscores, starting with a letter, at most 128
characters: `MYSQL_PASSWORD`, `API_KEY_2`. Lowercase and dashes are rejected. The `MOMDEPLOY_`
prefix is reserved by the platform. The same name cannot be both a secret and an `env` key.

Values are set with the CLI, from a file or from stdin, never as a command argument:

```
"${CLAUDE_PLUGIN_ROOT}/scripts/momdeploy" secret put DATABASE_URL --from-file ./db-url.txt
printf '%s' "$DATABASE_URL" | "${CLAUDE_PLUGIN_ROOT}/scripts/momdeploy" secret put DATABASE_URL
```

A secret that is set on the platform but not listed here is not injected. A name listed here but
never set stops the deploy — deliberately, because a process missing a variable it asked for would
otherwise fail in a way nobody can see from outside. A changed secret reaches the project with the
next deploy.

## Validation

The platform reads the manifest strictly: an unknown key is an error, so a typo such as
`enviroments` stops the deploy instead of being silently ignored. Error messages point at the line
of the file. There is no local `momdeploy validate` command, and no published JSON Schema.

Check before pushing:

- `version: v1` present;
- exactly one entry under `web`, with `command` and `port`;
- `port` matches the port the code actually listens on, and the code binds `0.0.0.0`;
- every worker and job has a non-empty `command`; jobs have a valid five-field cron;
- no `port:` outside `web`;
- no secret values anywhere in the file, and every name under `secrets:` actually set;
- no name is both an `env` key and a secret;
- process names lowercase-with-dashes, secret names UPPER_SNAKE_CASE.

## The Dockerfile next to it

The manifest says what to run, the `Dockerfile` at the repository root says how to build it. Both
are required. Write it from the code, not from a template:

- multi-stage: build in one stage, copy only the artefacts into a slim runtime image;
- install production dependencies from the lockfile that is in the repository;
- no secrets, `.env` files or credentials in the image; secrets arrive at run time;
- do not copy build caches, host `node_modules` or `.git` — add a `.dockerignore`;
- the final stage must contain whatever the manifest's `command:` names, interpreter included.

## What is not in v1

Deliberately absent, so do not invent them: `resources` (memory, CPU), `services` (databases and
other backing services), `domains` (custom domains), a release phase for migrations, more than one
web service. If the project needs one of these, say so plainly instead of writing a key the
validator will reject.
