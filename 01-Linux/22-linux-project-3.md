<a id="top"></a>

# 🚀 Linux Project #3: Lock down secrets and a helper tool

> **Source:** <https://www.escbash.com/topics/linux-project-3>
> **Contents:** 1 hands-on project task (no lessons or quizzes)
> **Skills used:** [08 Users and permissions](08-users-and-permissions.md) · [16 Shell configuration](16-shell-configuration.md) · [18 Links](18-links.md)

## 📑 Table of Contents

| # | Section | Type |
|---|---------|------|
| 1 | [Lock down secrets and a helper tool](#project) | 🛠️ Project |
| — | [Cheat sheet](#cheat-sheet) | ⚡ Summary |

<br>

---

<br>

<a id="project"></a>

## 🛠️ Project: Lock down secrets and a helper tool

A developer handed over a server in a rough state: a secrets file anyone can read, a wide-open SSH folder, and a deploy helper that only runs by full path. Clean it up.

### Set up the project

```bash
mkdir -p /root/app
echo "DB_PASSWORD=supersecret" > /root/app/secrets.env
chmod 644 /root/app/secrets.env
mkdir -p /root/.ssh
chmod 755 /root/.ssh
echo "PRIVATE KEY DATA" > /root/.ssh/id_rsa
chmod 644 /root/.ssh/id_rsa
echo "ssh-rsa AAAAB3Nza-example esc-bash" > /root/.ssh/id_rsa.pub
chmod 644 /root/.ssh/id_rsa.pub
mkdir -p /root/tools
echo "#!/bin/bash" > /root/tools/deploy-helper.sh
echo "echo deploying esc bash app" >> /root/tools/deploy-helper.sh
chmod +x /root/tools/deploy-helper.sh
```

### What to fix

| # | Target | Required |
|---|--------|----------|
| 1 | `/root/app/secrets.env` | owner read/write only → **600** |
| 2 | `/root/.ssh` | owner-only access → **700** |
| 3 | `/root/.ssh/id_rsa` (private key) | owner read/write only → **600** |
| 4 | `/root/.ssh/id_rsa.pub` (public key) | owner rw, others read → **644** |
| 5 | deploy helper | runnable as **`deploy-helper`** (not the full path) from anywhere, **in a brand-new shell** |
| 6 | `/root/.bashrc` | alias **`deploy`** that runs the helper |

The site runs 6 checks.

### Worked answer

> **📝 Note:** The site shows **no solution** for the project tasks, so this is my own answer built from chapters 8, 16 and 18. I haven't run it in the lab; treat it as a starting point.

**Permissions (chapter 8)**

```bash
chmod 600 /root/app/secrets.env
chmod 700 /root/.ssh
chmod 600 /root/.ssh/id_rsa
chmod 644 /root/.ssh/id_rsa.pub
```

**Put the helper on `PATH` under the name `deploy-helper`**

The file is called `deploy-helper.sh`, but you must type `deploy-helper`, so it needs a differently named entry in a `PATH` folder. A symlink (chapter 18) into `/usr/local/bin`, already on `PATH`, works in any new shell:

```bash
ln -sfn /root/tools/deploy-helper.sh /usr/local/bin/deploy-helper
```

*Alternative* (chapter 16): a personal bin folder, with the `PATH` export placed at the **top** of `.bashrc` because of the non-interactive guard:

```bash
mkdir -p /root/bin
ln -sfn /root/tools/deploy-helper.sh /root/bin/deploy-helper
grep -qxF 'export PATH="$HOME/bin:$PATH"' /root/.bashrc || sed -i '1i export PATH="$HOME/bin:$PATH"' /root/.bashrc
```

**The alias (chapter 16)**

```bash
grep -q '^alias deploy=' /root/.bashrc || echo "alias deploy='/root/tools/deploy-helper.sh'" >> /root/.bashrc
```

**Verify**

```bash
ls -l /root/app/secrets.env /root/.ssh /root/.ssh/id_rsa /root/.ssh/id_rsa.pub
bash -lc 'deploy-helper'          # new login shell finds it by name
grep '^alias deploy=' /root/.bashrc
```

<br>

---

<br>

<a id="cheat-sheet"></a>

## ⚡ Cheat sheet

| Target | Mode |
|--------|------|
| secrets / `.env` / private key | `600` |
| `~/.ssh` | `700` |
| public key | `644` |
| scripts anyone can run | `755` |

| Goal | How |
|------|-----|
| run a tool by bare name | executable file/symlink in a `PATH` folder |
| keep it across sessions | symlink in `/usr/local/bin`, or `PATH` export at the top of `~/.bashrc` |
| shortcut command | `alias name='command'` in `~/.bashrc` |
| link syntax | `ln -sfn TARGET LINK_NAME` (target first) |

<br>

[↑ Back to top](#top)
