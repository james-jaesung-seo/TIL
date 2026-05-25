---
public: true
title: Unattended-Upgrades Judgement One-Pager
source: docker-api-mismatch-troubleshooting/unattended-upgrades-judgement-onepager.md
tier: snippet
category: linux
synced_at: '2026-05-25T16:28:19Z'
---

# Unattended-Upgrades Judgement One-Pager

Use this page to determine whether a host is:
- fully disabled for unattended upgrades,
- generally enabled but Docker/Containerd-protected, or
- exposed to unexpected Docker auto-upgrades.

## Scope

This checklist is for Ubuntu hosts where Docker/Ceph stability matters.
It focuses on these package names:
- `docker.io`
- `docker-ce`
- `docker-ce-cli`
- `containerd`
- `containerd.io`

---

## 1) Run These Commands (in order)

```bash
# A. Global periodic settings
cat /etc/apt/apt.conf.d/20auto-upgrades

# B. Package presence/state
dpkg -l unattended-upgrades

# C. Service control state
systemctl is-enabled unattended-upgrades
systemctl status unattended-upgrades --no-pager -l

# D. Docker/Containerd hold state
apt-mark showhold | grep -Ei 'docker|containerd' || true

# E. Docker blacklist policy file
cat /etc/apt/apt.conf.d/51-ceph-no-docker-autoupgrade

# F. Real dry-run behavior (needs sudo)
sudo unattended-upgrade --dry-run -d 2>&1 | grep -Ei 'docker|containerd' || true
echo "grep_exit_code=$?"

# G. Historical evidence
zgrep -h "Commandline: /usr/bin/unattended-upgrade" /var/log/apt/history.log* 2>/dev/null
```

---

## 2) Interpretation Rules

### Case A — Fully disabled unattended-upgrades

Treat as **fully disabled** when most/all are true:
- `/etc/apt/apt.conf.d/20auto-upgrades` contains `APT::Periodic::Unattended-Upgrade "0";`
- `dpkg -l unattended-upgrades` shows `rc` or not installed
- `systemctl is-enabled unattended-upgrades` returns `masked` or `disabled`
- `systemctl status unattended-upgrades` is inactive/dead

Result:
- No unattended package upgrades are expected.
- Docker will not be auto-upgraded by unattended-upgrades because the feature is off.

### Case B — Enabled globally, but Docker/Containerd protected (recommended pattern)

Treat as **protected** when all are true:
- Global unattended-upgrades is enabled (`Unattended-Upgrade "1"` or service active model in your image)
- `51-ceph-no-docker-autoupgrade` exists and includes Docker/Containerd package blacklist entries
- `apt-mark showhold` includes at least `docker.io` and `containerd` (or your chosen Docker channel package set)
- Dry-run grep does **not** return Docker/Containerd candidates (`grep_exit_code=1`)

Result:
- General unattended-upgrades may still run.
- Docker/Containerd should remain blocked from unattended auto-upgrade.

### Case C — Risk/exposed state

Treat as **risk** if any are true:
- Global unattended-upgrades enabled
- Docker blacklist file missing or incomplete
- Required Docker/Containerd hold entries missing
- Dry-run output includes Docker/Containerd candidates (`grep_exit_code=0`)

Result:
- Host can drift into Docker client/daemon mismatch risk during unattended upgrade windows.

---

## 3) Fast Decision Matrix

- `Unattended-Upgrade "0"` + `masked` + `rc`: **Fully Disabled**
- Enabled + blacklist present + hold present + dry-run excludes Docker: **Protected**
- Enabled + any of (missing blacklist / missing hold / dry-run includes Docker): **Exposed**

---

## 4) Notes and Pitfalls

- `history.log` being empty (`0 bytes`) does **not** prove disabled; check config/service states directly.
- `Lock file is already taken` during dry-run means another apt/unattended process is running; wait or investigate locks.
- `grep` exit code behavior:
  - `0` = matched (`docker/containerd` present in dry-run output)
  - `1` = no match (typically desired for protected hosts)
- If remote checks run through automation (`pssh`), avoid misclassifying `grep` exit code `1` as operational failure.

