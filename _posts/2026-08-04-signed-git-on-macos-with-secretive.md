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

1. Settings → **SSH and GPG keys** → **New SSH key**. Paste the `.pub` file contents. An uploaded SSH key covers both authentication and commit signing.
2. Make sure `user.email` in git matches a verified email on your GitHub account (required for the **Verified** badge).
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

## 8. Verify end to end

```sh
git commit -m "test"            # signs via the enclave
git show --show-signature HEAD  # Good "git" signature with <key>
git push                        # GitHub shows the commit as Verified
```

## Troubleshooting

| Symptom | Cause / fix |
|---|---|
| `error: No private key found for public key "…/PublicKeys/….pub"` | `$SSH_AUTH_SOCK` points at the launchd agent (`/var/run/com.apple.launchd.*/Listeners`), not Secretive. Fix: export it in `~/.zshrc` (step 3) and **open a new terminal**. GUI apps need `launchctl setenv`. |
| `error: gpg.ssh.allowedSignersFile needs to be configured and exist` | Create `~/.gitallowedsigners` (step 7). |
| Commit signed but not **Verified** on GitHub | `user.email` doesn't match a verified GitHub email, or the public key isn't uploaded (step 5). |
| `ssh: connect to host github.com port 22: Operation timed out` | Port 22 blocked — use SSH over HTTPS (443) or the GitHub CLI: `gh ssh-key add …`. |
| Signing works in the terminal but not in an editor/IDE | The app was started before `launchctl setenv`; restart it, or run it from the terminal. |
