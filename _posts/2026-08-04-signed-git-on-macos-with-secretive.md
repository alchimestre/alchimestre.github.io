---
layout: post
title: "Signed Git on macOS with Secretive"
date: 2026-08-04
---

# Signed Git on macOS with Secretive

Set up SSH auth + git commit/tag signing backed by the Mac's Secure Enclave. Keys never leave the enclave; commits show as **Verified** on GitHub.

## 1. Install Secretive

- From the App Store or [github.com/maxgoedjen/secretive](https://github.com/maxgoedjen/secretive).
- Launch it once. It starts the agent; the socket lives at:
  `~/Library/Containers/com.maxgoedjen.Secretive.SecretAgent/Data/socket.ssh`

## 2. Create a key

- Secretive → `New Key` → **Ed25519** (or ECDSA).
- Copy the public key path: the `.pub` file in `…/SecretAgent/Data/PublicKeys/`.

## 3. Point the shell (and GUI apps) at the agent

Add to `~/.zshrc`:

```sh
export SSH_AUTH_SOCK="$HOME/Library/Containers/com.maxgoedjen.Secretive.SecretAgent/Data/socket.ssh"
```

For GUI apps and new terminals, also set it in launchd (new shells inherit it; resets on reboot — put it in a LaunchAgent if you want it permanent):

```sh
launchctl setenv SSH_AUTH_SOCK ~/Library/Containers/com.maxgoedjen.Secretive.SecretAgent/Data/socket.ssh
```

**Open a new terminal window** — existing shells keep their old `SSH_AUTH_SOCK`.

## 4. SSH config (`~/.ssh/config`)

```sshconfig
Host *
    IdentityAgent ~/Library/Containers/com.maxgoedjen.Secretive.SecretAgent/Data/socket.ssh
    AddKeysToAgent yes
```

`IdentityAgent` covers `ssh` and `git push`. Git commit signing does **not** read it — it uses `$SSH_AUTH_SOCK` (step 3).

## 5. GitHub

1. Settings → **SSH and GPG keys** → **New SSH key** → **Key type: Signing Key**. Paste the `.pub` file contents.
   Authentication keys and signing keys are separate on GitHub: an authentication-only key does **not** verify commits. Register the key as a **Signing Key** (the same key can be uploaded twice, once per type).
2. The commit's **committer email** must be a verified email on your account (Settings → **Emails**). Without it, GitHub shows **Unverified** even with a valid signature.
   - Verify through the GitHub **web UI** — emails added via the API are registered but GitHub sends no verification email, so the link never arrives.
   - `user.email` in git must match exactly, including `+alias` suffixes.
3. Test: `ssh -T git@github.com`

## 6. Git config (`~/.gitconfig`)

```ini
[user]
    name = Your Name
    email = you@example.com
    signingkey = ~/Library/Containers/com.maxgoedjen.Secretive.SecretAgent/Data/PublicKeys/<key-id>.pub

[commit]
    gpgsign = true

[tag]
    gpgsign = true

[gpg]
    format = ssh

[gpg "ssh"]
    allowedSignersFile = ~/.gitallowedsigners
```

Optional — pick the signing key from the agent at commit time instead of pinning a file path (survives key rotation and SSH agent forwarding):

```ini
[gpg "ssh"]
    defaultKeyCommand = ssh-add -L
```

## 7. Allowed signers (local verification)

`ssh-keygen -Y verify` needs an allowlist. Create `~/.gitallowedsigners` with the key line from `ssh-add -L` (prefix with the email that signs):

```
you@example.com ecdsa-sha2-nistp256 AAAAE2Vj…= comment
```

Then verify locally:

```sh
git log --show-signature -1     # expect: Good "git" signature …
```

This is local-only: `allowedSignersFile` is your machine's allowlist. GitHub does not read it — it checks its own key registry and your verified email (step 5). If you rotate keys, keep both lines in this file so old commits still verify locally.

## 8. Verify end to end

```sh
git commit -m "test"            # signs via the enclave
git show --show-signature HEAD  # Good "git" signature with <key>
git push                        # GitHub shows the commit as Verified
```

## 9. Skip repeated Touch ID prompts (time-based unlock)

Out of the box, every sign asks for Touch ID. Secretive can keep a key unlocked for a set window after one authentication:

When a signing request triggers the Touch ID prompt, Secretive also posts a notification. Click it and pick a duration:

- **Don't persist** · **1 minute** · **5 minutes** · **1 hour** · **24 hours**

Authenticate once with Touch ID, and every sign within that window reuses the authorization — no fingerprint. The timer is per-key and monotonic (clock-change safe); it expires on its own, and restarting the agent clears it.

Requires Secretive ≥ 2.2.0 (enabled since, works on 3.0.4).

## Troubleshooting

| Symptom | Cause / fix |
|---|---|
| `error: No private key found for public key "…/PublicKeys/….pub"` | `$SSH_AUTH_SOCK` points at the launchd agent (`/var/run/com.apple.launchd.*/Listeners`), not Secretive. Fix: export it in `~/.zshrc` (step 3) and **open a new terminal**. GUI apps need `launchctl setenv`; non-interactive shells (cron, IDE daemons, a crashed terminal session) never source `~/.zshrc` — set it there too. Diagnose: `echo $SSH_AUTH_SOCK`, `ssh-add -L`. |
| `error: gpg.ssh.allowedSignersFile needs to be configured and exist` | Create `~/.gitallowedsigners` (step 7). |
| Commit signed but not **Verified** on GitHub | Check in order: (1) key registered as **Signing Key**, not just authentication (step 5); (2) committer email verified on GitHub via the web UI (step 5); (3) GitHub reports `no_user`. |
| GitHub shows `unverified_email` | Signature checks out; the committer email isn't verified on the account — Settings → **Emails** → verify through the web UI. |
| GitHub shows `no_user` for an old commit | The signing key wasn't on your GitHub account when the commit was pushed; GitHub caches verification at push time. Re-pushing the commit (amend/rebase + re-sign + force-push) re-verifies it. |
| Local `git log --show-signature` says `Good` but GitHub says unverified | `allowedSignersFile` is local-only. GitHub uses its own registry + verified-email check (step 5) — the two verifications are independent. |
| `ssh: connect to host github.com port 22: Operation timed out` | Port 22 blocked — use SSH over HTTPS (443) or the GitHub CLI: `gh ssh-key add …`. |
| Signing works in the terminal but not in an editor/IDE | The app was started before `launchctl setenv`; restart it, or run it from the terminal. |
