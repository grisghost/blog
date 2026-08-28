---
title: Pushing a New Project or Updating an Existing One on GitHub
description: >-
  A practical walkthrough for pushing a brand-new local project to GitHub,
  and for pulling and pushing changes to a repo you already cloned.
date: 2026-07-07 22:15:00 +0000
categories: [Development, Dev Guide]
tags: [git, github]
pin: false
---

## Prerequisites

- [Git](https://git-scm.com/downloads) installed locally.
- A [GitHub account](https://github.com/).
- A [Personal Access Token (PAT)](https://github.com/settings/tokens) with `repo` scope. GitHub no longer accepts your account password over HTTPS, so the token is used in its place.

> Treat your PAT like a password. Don't paste it into files you commit, and store it somewhere safe (a password manager or your OS credential store) rather than typing it in every time.
{: .prompt-warning }

## Initial Git Configuration

Before doing anything else, configure your identity. Git attaches your name and email to every commit you make.

```console
$ git config --global user.name "Your Name"
$ git config --global user.email "you@example.com"
```

The `--global` flag applies these settings to all repositories on your machine. To apply them only to a specific project, run the same commands without `--global` from inside that project's directory.

To verify your configuration at any time:

```console
$ git config --global --list
```

## Saving Your Credentials (No Password Every Time)

By default, Git asks for your username and PAT on every push. To avoid that, enable the credential helper:

### On Linux

```console
$ git config --global credential.helper store
```

> `store` saves credentials in plain text at `~/.git-credentials`. It is convenient but not encrypted. On a personal or private server this is usually acceptable; on a shared machine consider using `cache` instead (which keeps them in memory temporarily):
> ```console
> $ git config --global credential.helper 'cache --timeout=3600'
> ```
{: .prompt-warning }

### On macOS

```console
$ git config --global credential.helper osxkeychain
```

Credentials are stored in the macOS Keychain, encrypted.

### On Windows

```console
$ git config --global credential.helper manager
```

Uses the Windows Credential Manager.

---

After the first successful push where you enter your credentials, Git will remember them and you won't be prompted again.

## Part 1 — Pushing a New Project to GitHub

Use this when you have a project on your machine that has never been pushed anywhere.

### 1. Create the Repository on GitHub

1. Go to [github.com/new](https://github.com/new).
2. Give it a name and choose **Public** or **Private**.
3. **Do not** initialize it with a README, `.gitignore`, or license if your local project already has files — that avoids a conflicting history on first push.
4. Click **Create repository**.

### 2. Initialize Git in Your Local Project

```console
$ cd /path/to/your-project
$ git init
$ git add .
$ git commit -m "Initial commit"
```

### 3. Connect the Local Project to GitHub

Copy the HTTPS URL from the repository page, then:

```console
$ git remote add origin https://github.com/YOUR_USERNAME/YOUR_REPO.git
$ git branch -M main
$ git push -u origin main
```

When prompted for credentials:
- **Username**: your GitHub username
- **Password**: paste your **Personal Access Token** (not your account password)

> After the first `push -u`, Git remembers the `origin`/`main` link. Future pushes only need `git push`.
{: .prompt-tip }

## Part 2 — Updating an Existing Project

Use this when you already have a repo cloned locally and want to pull the latest changes, then push your own.

### 1. Check the Current State

```console
$ cd /path/to/your-project
$ git status
```

This shows any uncommitted changes before you pull, so you don't lose local work.

### 2. Pull the Latest Changes

```console
$ git pull origin main
```

> If you have uncommitted local changes that conflict with incoming ones, Git will stop and ask you to commit or stash them first:
> ```console
> $ git stash
> $ git pull origin main
> $ git stash pop
> ```
{: .prompt-info }

### 3. Make Your Changes

Edit your files as needed, then check what changed:

```console
$ git status
$ git diff
```

### 4. Stage and Commit

```console
$ git add .
$ git commit -m "Describe what changed"
```

> Prefer several small, descriptive commits over one giant one — it makes history easier to read and problems easier to trace later.
{: .prompt-tip }

### 5. Push the Changes

```console
$ git push origin main
```

## Changing Your Configuration Later

### Update Your Name or Email

```console
$ git config --global user.name "New Name"
$ git config --global user.email "new@example.com"
```

> Changing the email globally only affects future commits. Commits already made keep the old email attached to them.
{: .prompt-info }

### Change the Default Branch Name

To rename the current local branch:

```console
$ git branch -m old-name new-name
```

To rename the branch you're currently on:

```console
$ git branch -m new-name
```

After renaming, update the remote tracking reference:

```console
$ git push origin -u new-name
```

Then delete the old branch on the remote if needed:

```console
$ git push origin --delete old-name
```

### Change the Remote Repository URL

To see the current remote URL:

```console
$ git remote -v
```

To update it:

```console
$ git remote set-url origin https://github.com/YOUR_USERNAME/NEW_REPO.git
```

To verify it was updated:

```console
$ git remote -v
```

### Update Saved Credentials

If you generated a new PAT or changed your GitHub account, clear the saved credentials and re-enter them on the next push.

**On Linux (store helper):**

```console
$ git credential reject
protocol=https
host=github.com
```

Or simply edit `~/.git-credentials` directly and remove the GitHub line.

**On macOS:**

Open **Keychain Access**, search for `github.com`, and delete the entry. Git will prompt you again on the next push.

**On Windows:**

Open **Credential Manager → Windows Credentials**, find the GitHub entry, and remove it.

## Handling a Rejected Push

If someone else (or another machine of yours) pushed changes you don't have yet, `git push` will be rejected:

```console
$ git pull origin main --rebase
$ git push origin main
```

`--rebase` replays your commits on top of the latest remote history instead of creating an extra merge commit, keeping the log linear.

## Force Pushing (Use With Caution)

`git pull --rebase` fixes a rejected push in the normal case. But sometimes you deliberately need the remote to match your local history exactly — for example, after rewriting commits with `git rebase -i`, or after a mistaken push you need to undo. That's what force pushing is for.

> A force push overwrites the remote branch's history. Any commits that exist on the remote but not in your local branch are **permanently lost** for anyone who hasn't already pulled them. Never force push to a shared branch (like `main`) without warning your collaborators first.
{: .prompt-danger }

### `git push --force-with-lease origin main`

```console
$ git push --force-with-lease origin main
```

The safer option. It checks that the remote branch hasn't changed since you last fetched it — if a teammate pushed something new in the meantime, this command **fails instead of overwriting** their work. Use this by default whenever you need to force push.

### `git push --force origin main` / `git push -f origin main`

```console
$ git push --force origin main
$ git push -f origin main
```

These two are identical (`-f` is just shorthand for `--force`). Unlike `--force-with-lease`, this overwrites the remote branch unconditionally, even if someone else pushed new commits you don't have. Only reach for this when you're certain no one else could have pushed in the meantime — for example, on a personal project with a single contributor.

| Command | Checks remote first? | Safe for shared branches? |
|---|---|---|
| `git push --force-with-lease origin main` | Yes | Safer, but still communicate with collaborators |
| `git push --force origin main` / `git push -f origin main` | No | Avoid — can silently erase others' commits |

## Quick Reference

| Situation | Command |
|---|---|
| Set global username | `git config --global user.name "Name"` |
| Set global email | `git config --global user.email "email"` |
| Save credentials (Linux) | `git config --global credential.helper store` |
| Save credentials (macOS) | `git config --global credential.helper osxkeychain` |
| Save credentials (Windows) | `git config --global credential.helper manager` |
| First-time setup in a new folder | `git init` |
| Link to a GitHub repo | `git remote add origin <url>` |
| Change remote URL | `git remote set-url origin <url>` |
| Rename current branch | `git branch -m new-name` |
| Stage all changes | `git add .` |
| Commit staged changes | `git commit -m "message"` |
| Get remote changes | `git pull origin main` |
| Send local commits | `git push origin main` |
| Force push (safer) | `git push --force-with-lease origin main` |
| Force push (unconditional) | `git push --force origin main` |
| Check status | `git status` |
| See file-level changes | `git diff` |
| View current config | `git config --global --list` |
| View remote URL | `git remote -v` |

## Common Errors

- **`remote origin already exists`** — you already ran `git remote add origin` once. Fix it with:
  ```console
  $ git remote set-url origin https://github.com/YOUR_USERNAME/YOUR_REPO.git
  ```
- **`Updates were rejected because the remote contains work that you do not have locally`** — pull first (see the rebase steps above), then push.
- **`support for password authentication was removed`** — you're using your account password instead of a Personal Access Token. Generate one at [github.com/settings/tokens](https://github.com/settings/tokens) and use it as the password.
