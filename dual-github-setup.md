# Dual GitHub Account Setup on Mac

**Using iTerm2 Profiles + SSH Keys**
`saiabhiramjaini` (Personal) • `abhiram-neosigma` (Work)

---

## Overview

This document explains how to set up and use two separate GitHub accounts on a single Mac — one personal and one for Neosigma — without ever mixing up identities. The setup uses:

- Separate SSH keys for each GitHub account
- An SSH config file that routes each key to the right account
- Separate gitconfig files that hold each account's name and email
- iTerm2 profiles that automatically load the right identity when you open a terminal

> **Result:** Open the Personal terminal → all git activity uses your personal GitHub. Open the Neosigma terminal → all git activity uses your work GitHub. No manual switching needed.

---

## Part 1 — SSH Keys

### What is an SSH key?

An SSH key is a pair of files — a private key (stays on your Mac) and a public key (uploaded to GitHub). When you connect to GitHub, your Mac uses the private key to prove your identity without needing a password.

### Your SSH keys

You have two separate key pairs, one for each account:

```
~/.ssh/id_ed25519_personal       ← private key for personal account
~/.ssh/id_ed25519_personal.pub   ← public key (uploaded to GitHub personal)

~/.ssh/id_ed25519_neosigma       ← private key for Neosigma account
~/.ssh/id_ed25519_neosigma.pub   ← public key (uploaded to GitHub Neosigma)
```

> **Important:** Each public key must be registered on its own GitHub account only. GitHub does not allow the same key on two different accounts.

### How to generate a new SSH key (if needed)

Run this command, replacing the email with the account's email:

```bash
ssh-keygen -t ed25519 -C "your-email@example.com" -f ~/.ssh/id_ed25519_personal
```

Press Enter twice when asked for a passphrase (to skip it). Then copy the public key and add it to GitHub:

```bash
pbcopy < ~/.ssh/id_ed25519_personal.pub
# Then go to GitHub → Settings → SSH and GPG keys → New SSH key → Paste
```

---

## Part 2 — SSH Config File

### What is the SSH config file?

The file `~/.ssh/config` tells SSH which key to use depending on which "host alias" you connect to. Instead of connecting directly to `github.com`, you connect to a custom alias like `github-personal` or `github-neosigma`, and SSH automatically picks the right key.

### Your SSH config

**Location:** `~/.ssh/config`

```
# Personal GitHub
Host github-personal
    HostName github.com
    User git
    IdentityFile ~/.ssh/id_ed25519_personal

# Neosigma GitHub
Host github-neosigma
    HostName github.com
    User git
    IdentityFile ~/.ssh/id_ed25519_neosigma
```

**What this means:**

- `github-personal` → connects to `github.com` using your personal SSH key
- `github-neosigma` → connects to `github.com` using your Neosigma SSH key

### Testing the SSH connection

Run these commands to verify both accounts are authenticated:

```bash
ssh -T git@github-personal
# Expected: Hi saiabhiramjaini! You've successfully authenticated...

ssh -T git@github-neosigma
# Expected: Hi abhiram-neosigma! You've successfully authenticated...
```

---

## Part 3 — Gitconfig Files

### What is a gitconfig file?

A gitconfig file tells git what name and email to attach to your commits. When someone looks at a commit, they see the author name and email from this file. You have two separate gitconfig files — one per account.

### Your gitconfig files

**Personal — `~/.gitconfig-personal`:**

```ini
[user]
    name = saiabhiramjaini
    email = abhiramjaini28@gmail.com

[url "git@github-personal:"]
    insteadOf = https://github.com/
```

**Neosigma — `~/.gitconfig-neosigma`:**

```ini
[user]
    name = abhiram-neosigma
    email = abhiram@neosigma.ai

[url "git@github-neosigma:"]
    insteadOf = https://github.com/
```

> **The `insteadOf` rule:** This is the magic that lets you use normal `https://github.com/...` clone URLs. Git automatically rewrites them to use the correct SSH alias behind the scenes, so you never have to type `git@github-personal:...` manually.

### Verifying commit author

After making a commit, run this to confirm it's tagged with the right identity:

```bash
git log --oneline --format="%h %an <%ae> %s" -1

# Personal terminal output:
# 8daf380 saiabhiramjaini <abhiramjaini28@gmail.com> feat: updated

# Neosigma terminal output:
# a1b2c3d abhiram-neosigma <abhiram@neosigma.ai> feat: updated
```

---

## Part 4 — iTerm2 Profiles

### What are iTerm2 profiles?

iTerm2 profiles let you open a terminal window that is pre-configured with specific settings. You have set up two profiles — Personal and Neosigma — each of which automatically loads the correct SSH key and gitconfig when opened.

### How each profile is configured

In **iTerm2 → Settings → Profiles**, each profile has a "Send text at start" field that runs automatically when a new terminal opens:

**Personal profile — Send text at start:**

```bash
ssh-add ~/.ssh/id_ed25519_personal 2>/dev/null; export GIT_CONFIG_GLOBAL=~/.gitconfig-personal; clear
```

**Neosigma profile — Send text at start:**

```bash
ssh-add ~/.ssh/id_ed25519_neosigma 2>/dev/null; export GIT_CONFIG_GLOBAL=~/.gitconfig-neosigma; clear
```

**What these commands do:**

- `ssh-add` — loads the correct SSH key into the SSH agent so it's ready to use
- `export GIT_CONFIG_GLOBAL` — tells git to use the matching gitconfig file for this terminal session
- `clear` — cleans up the terminal screen after setup

### How to open a profile

In the Mac menu bar (at the top of the screen), click **Profiles** → then select **Personal** or **Neosigma**. A new terminal window opens already configured for that account.

> **Tip:** Always use the Personal profile for personal repos and the Neosigma profile for work repos. Never mix them up.

---

## Part 5 — Daily Usage

### Cloning a repo

Just copy the HTTPS URL from GitHub and paste it normally — the `insteadOf` rule handles the rest automatically:

**In Personal terminal:**

```bash
git clone https://github.com/saiabhiramjaini/repo-name.git
# Git rewrites this to: git@github-personal:saiabhiramjaini/repo-name.git
```

**In Neosigma terminal:**

```bash
git clone https://github.com/neosigma-org/repo-name.git
# Git rewrites this to: git@github-neosigma:neosigma-org/repo-name.git
```

### Fixing an existing repo's remote

If you cloned a repo before this setup, the remote URL may not use the SSH alias. Fix it with:

```bash
# Check current remote
git remote -v

# Fix for personal repo
git remote set-url origin git@github-personal:saiabhiramjaini/repo-name.git

# Fix for Neosigma repo
git remote set-url origin git@github-neosigma:neosigma-org/repo-name.git
```

### Pushing changes

Push works normally — just make sure you're in the right terminal profile:

```bash
git add .
git commit -m "your message"
git push origin main
```

---

## Part 6 — Quick Reference Summary

| Setting | Personal | Neosigma | Notes |
|---|---|---|---|
| **SSH Key** | `id_ed25519_personal` | `id_ed25519_neosigma` | Separate keys |
| **Host Alias** | `github-personal` | `github-neosigma` | In `~/.ssh/config` |
| **Git Name** | `saiabhiramjaini` | `abhiram-neosigma` | Commit author |
| **Git Email** | `abhiramjaini28@gmail.com` | `abhiram@neosigma.ai` | Commit email |
| **Gitconfig** | `.gitconfig-personal` | `.gitconfig-neosigma` | Per-profile |
| **iTerm2 Profile** | Personal | Neosigma | Auto-loads config |

---

## Troubleshooting

### Permission denied (publickey)

This means GitHub rejected your SSH key. Check:

- Run `ssh -T git@github-personal` to test the connection
- Make sure the public key is added to the correct GitHub account under **Settings → SSH keys**
- Make sure you are in the correct iTerm2 profile for that account

### Key is already in use

GitHub does not allow the same public key on two accounts. If you see this error, generate a new key for that account:

```bash
ssh-keygen -t ed25519 -C "your-email@example.com" -f ~/.ssh/id_ed25519_personal
pbcopy < ~/.ssh/id_ed25519_personal.pub
# Add the new key to GitHub → Settings → SSH keys
```

### Wrong author on commits

If commits show the wrong name or email, you may have opened the wrong profile. Check which config is active:

```bash
git config user.email     # should match the account you want
echo $GIT_CONFIG_GLOBAL   # should show the correct gitconfig file path
```

### Push asks for username and password

This means the repo's remote URL is still using HTTPS without the alias rewrite. Fix it:

```bash
git remote set-url origin git@github-personal:saiabhiramjaini/repo-name.git
```

---

> **Remember:** The golden rule — always open the correct iTerm2 profile before doing any git work. Personal terminal for personal projects, Neosigma terminal for work projects.
