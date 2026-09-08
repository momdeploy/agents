---
name: secrets
description: Sets, lists, deletes and rotates momdeploy project secrets — database passwords, API keys, signing keys — with `momdeploy secret put`, `list` and `delete`, without ever putting a value in a command argument. Use when the user wants to store or change a secret, when a deploy stops on a declared secret that was never set, or when a value needs to move out of the code into the platform.
compatibility: The momdeploy CLI ships inside this plugin and is run by its full path. A directory linked to a project, or a project id for `--project`.
---

# momdeploy secrets

A secret is a value the code needs at run time and the repository must not contain. The platform
stores it encrypted and injects it as an environment variable into every process of the project,
on the next deploy, and only if `momdeploy.yaml` lists the name under `secrets:`.

A value is never printed back, not even to whoever set it. There is no command that reads one.

## How to run the CLI

The CLI ships with this plugin and is **not** on the `PATH`. Run it by its full path:

```
"${CLAUDE_PLUGIN_ROOT}/scripts/momdeploy" --version
```

Every command below is written that way. A bare `momdeploy` is not a command here and fails with
`command not found`; if you see that, you dropped the path.

## The one rule that matters

**A secret value is never a command argument.** Arguments end up in shell history, in process
lists and in this transcript, and a transcript is not a place a password can be taken back from.
The CLI has no flag that accepts a value, by design. Use one of three inputs:

```
printf '%s' "$DATABASE_URL" | "${CLAUDE_PLUGIN_ROOT}/scripts/momdeploy" secret put DATABASE_URL
"${CLAUDE_PLUGIN_ROOT}/scripts/momdeploy" secret put TLS_KEY --from-file ./key.pem
"${CLAUDE_PLUGIN_ROOT}/scripts/momdeploy" secret put --from-env-file .env
```

The first reads stdin, which is what to use when the value is already in a variable. The second
reads a file, which is what to use for keys and certificates. The third sets every `NAME=value`
line of a `.env` file at once and takes no name. In a terminal with no stdin and no flag, the CLI
asks for the value with a hidden prompt, which is the user's path, not yours.

One trailing newline is stripped, so `printf` and text files both behave. Nothing else is trimmed.

If the value is not already on disk or in the environment, **do not invent a way to obtain it and
do not make one up**. Ask the user to run the command themselves, and carry on with the rest of
the work.

## Naming

Upper-case latin letters, digits and underscores, starting with a letter, at most 128 characters:
`DATABASE_URL`, `SMTP_PASSWORD`, `API_KEY_2`. Lower case and dashes are rejected. The `MOMDEPLOY_`
prefix is reserved by the platform.

A name cannot be both an `env` key and a secret in the same manifest.

## Declaring

Setting a value is half the job. The platform injects the names the manifest asks for, so every
secret must also appear under `secrets:` in `momdeploy.yaml`:

```yaml
secrets:
  - DATABASE_URL
  - SMTP_PASSWORD
```

After a successful `put` the CLI says so itself when the name is missing from the manifest:
`Note: … is not declared under secrets: …`. Add it, or the running project never sees the value.

The reverse is a hard stop: a name declared under `secrets:` but never set **fails the deploy**.
That is deliberate. A process missing a variable it asked for would otherwise break in a way
nobody can see from outside.

## Listing and deleting

```
"${CLAUDE_PLUGIN_ROOT}/scripts/momdeploy" secret list
"${CLAUDE_PLUGIN_ROOT}/scripts/momdeploy" secret list --json
"${CLAUDE_PLUGIN_ROOT}/scripts/momdeploy" secret delete SMTP_PASSWORD
```

`list` shows names and when they changed, never values. Use it to check what the project has
before setting anything, so an existing secret is not overwritten by accident.

`delete` removes the value from the platform. The running project keeps the variable until the
next deploy, so deleting is not an emergency stop: to cut access now, rotate the credential at its
source. Remember to remove the name from `secrets:` as well, or the next deploy stops on the name
that no longer has a value.

## Which project

Commands take the project from `.momdeploy/project.json` in the directory they run in, or in one
of its parents, the way git finds its repository. Run them from the code.

`--project <id>` overrides the link. That is the path from CI, or from a directory that is not
tied to a project. `momdeploy project list --json` gives the ids.

`Error: not linked: … or pass --project <id>` means neither was available.

## When a secret changes

A new value reaches the running project with the **next deploy**, not immediately. There is no
command to re-deploy without a commit. After changing a secret, push:

```
"${CLAUDE_PLUGIN_ROOT}/scripts/momdeploy" push -m "rotate DATABASE_URL"
```

## Moving a secret out of the code

When a credential is hardcoded in the source, the fix has four steps, in this order:

1. read the value out of the code and set it with `momdeploy secret put`, from stdin or a file;
2. add the name under `secrets:` in `momdeploy.yaml`;
3. change the code to read the environment variable instead of the literal;
4. push.

Say plainly that the old value is still in the repository's history and should be rotated at its
source. Removing a line does not remove it from git.
