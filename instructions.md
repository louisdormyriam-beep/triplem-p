# CI/CD Setup Instructions (Secure)

This document explains how to set up a secure CI/CD pipeline to deploy the repository to your DigitalOcean droplet at 159.89.46.51. It intentionally does NOT include any private keys. Keep private keys secret and add them only to your CI provider's secret store.

## Overview
- Create a dedicated deploy SSH keypair (ed25519 recommended).
- Install the public key on the server (preferably for a restricted `deploy` user, not `root`).
- Store the private key in your CI provider's secret store (example secret name: `DEPLOY_KEY`).
- Use a CI workflow that loads the private key and performs `rsync`/`scp` or remote commands.
- Restrict and rotate keys regularly.

## 1) Generate a deploy key (if you haven't already)
Run on your local machine (do NOT share the private key):

```bash
# Generate ed25519 keypair without passphrase
ssh-keygen -t ed25519 -f ~/.ssh/ci_cd_deploy -C "ci_cd_deploy@triplem" -N ""

# Public: ~/.ssh/ci_cd_deploy.pub
# Private: ~/.ssh/ci_cd_deploy  (KEEP THIS PRIVATE)
```

## 2) Install public key on the server (recommended: `deploy` user)
Recommended: create a limited `deploy` user and add the public key to that user's authorized_keys.

Run these commands from your local machine (uses your existing root access). Replace commands as needed for your distribution.

```bash
# Create deploy user on the droplet (run as root)
ssh root@159.89.46.51 'adduser --disabled-password --gecos "" deploy || true'

# Ensure .ssh directory exists and has correct permissions
ssh root@159.89.46.51 'mkdir -p /home/deploy/.ssh && chown deploy:deploy /home/deploy/.ssh && chmod 700 /home/deploy/.ssh'

# Copy the public key from your machine to the server (appends)
cat ~/.ssh/ci_cd_deploy.pub | ssh root@159.89.46.51 'cat >> /home/deploy/.ssh/authorized_keys && chown deploy:deploy /home/deploy/.ssh/authorized_keys && chmod 600 /home/deploy/.ssh/authorized_keys'

# Verify you can connect as deploy (locally):
ssh -i ~/.ssh/ci_cd_deploy deploy@159.89.46.51 'echo deploy_ok'
```

Notes:
- If you previously installed the key under `root`, you can keep using `root`, but using a restricted `deploy` user is safer.
- If the server uses `ubuntu` or different user creation tools, substitute `adduser` with the appropriate command.

## 3) Add the private key to GitHub Actions (or your CI provider)

Do not paste the key into chat or public places. Copy it locally and paste into the CI secret UI.

Using the GitHub web UI:
- Repository → Settings → Secrets and variables → Actions → New repository secret
- Name: `DEPLOY_KEY`
- Value: the full contents of `~/.ssh/ci_cd_deploy` (the private key file)

Using GitHub CLI (run locally):

```bash
gh secret set DEPLOY_KEY --body "$(cat ~/.ssh/ci_cd_deploy)"
```

Other CI providers have similar secure secret stores — use them.

## 4) Example GitHub Actions workflow
Place this in `.github/workflows/deploy.yml`. Update paths and user (`deploy@159.89.46.51`) as needed.

```yaml
name: Deploy to Droplet
on:
  push:
    branches: [ main ]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Start SSH agent and add key
        uses: webfactory/ssh-agent@v0.8.1
        with:
          ssh-private-key: ${{ secrets.DEPLOY_KEY }}

      - name: Add droplet to known_hosts
        run: |
          mkdir -p ~/.ssh
          ssh-keyscan -H 159.89.46.51 >> ~/.ssh/known_hosts

      - name: Rsync project to server
        run: |
          rsync -az --delete --exclude='.git' ./ deploy@159.89.46.51:/var/www/html/

      # Optionally run remote commands (migrations, restart service, etc.)
      - name: Run remote post-deploy commands
        run: |
          ssh deploy@159.89.46.51 'cd /var/www/html && ./deploy-post.sh || true'
```

Notes:
- Use `deploy@159.89.46.51` instead of `root` whenever possible.
- Ensure the target path (`/var/www/html/`) has appropriate ownership/permissions for `deploy`.
- If you need to build before deploying (e.g. `npm run build`), add the build steps before `rsync` and upload the build output only.

## 5) Restrict the key on the server (recommended)
You can restrict what the key can do by prepending options in `/home/deploy/.ssh/authorized_keys` before the key string. Example:

```
command="/usr/local/bin/deploy-receiver.sh",no-port-forwarding,no-X11-forwarding,no-agent-forwarding,no-pty ssh-ed25519 AAAA... ci_cd_deploy@triplem
```

- `command="..."` forces any connection with that key to run the specified command (useful for narrowly-scoped deployments).
- `no-port-forwarding,no-X11-forwarding,no-agent-forwarding,no-pty` reduces attack surface.
- Test restrictions carefully to avoid locking out legitimate automation.

## 6) Rotate keys and revoke access
- To rotate, generate a new keypair locally, add the new public key to the server, update your CI secret with the new private key, then remove the old public key from `authorized_keys` on the server.

Commands (example):

```bash
# Generate new key
ssh-keygen -t ed25519 -f ~/.ssh/ci_cd_deploy_v2 -N ""

# Upload new public key
cat ~/.ssh/ci_cd_deploy_v2.pub | ssh root@159.89.46.51 'cat >> /home/deploy/.ssh/authorized_keys'

# Update CI secret locally (GitHub example)
gh secret set DEPLOY_KEY --body "$(cat ~/.ssh/ci_cd_deploy_v2)"

# Remove old key from server after verifying new key works
ssh deploy@159.89.46.51 'vim /home/deploy/.ssh/authorized_keys' # manually remove old key line
```

## 7) Verification & Troubleshooting
- Locally verify: `ssh -i ~/.ssh/ci_cd_deploy deploy@159.89.46.51 'echo ok'`
- From CI, check the job logs — the `webfactory/ssh-agent` step should succeed and subsequent SSH/rsync steps should run.
- Common issues: wrong file permissions on `~/.ssh` or `authorized_keys` (use 700 for dir, 600 for file), SELinux/AppArmor restrictions, wrong user or path.

## Security checklist
- Never paste private keys into chat, issue trackers, or public places.
- Use separate deploy keys per environment/repository where possible.
- Use a restricted `deploy` user and limit key capabilities.
- Rotate keys periodically and revoke immediately if compromise suspected.

---

If you want, I can:
- Create the `.github/workflows/deploy.yml` file in this repository adapted to your preferred build path and target directory.
- Run the server-side commands to create the `deploy` user and install the public key (I already installed a key for `root` earlier; tell me if you want me to switch to `deploy`).

