# CS3 IaC Lab — Ansible Project

Infrastructure as Code lab for the CS3 Cybersecurity Club.
Applies security hardening and deploys a web server to the local Kali machine
using Ansible, targeting `localhost`.

Based on the real CS3-Infra `common` role at https://github.com/Server2090/CS3-Infra

---

## Project Structure

```
ansible-lab/
├── ansible.cfg              ← Ansible settings (inventory path, output format, etc.)
├── requirements.yml         ← Collection dependencies
├── site.yml                 ← Master playbook — entry point for everything
├── .gitignore               ← Files Git must never track (keys, secrets, logs)
│
├── inventory/
│   └── hosts.yml            ← Target: localhost (no SSH needed)
│
└── roles/
    ├── hardening/           ← Baseline security hardening
    │   ├── handlers/
    │   │   └── main.yml     ← SSH restart, fail2ban restart, GRUB update
    │   ├── tasks/
    │   │   ├── main.yml     ← Ordered entry point: includes all task files
    │   │   ├── packages.yml ← Install ufw, fail2ban, openssh-server
    │   │   ├── sysctl.yml   ← Kernel hardening parameters
    │   │   ├── ipv6.yml     ← Disable IPv6 (sysctl + GRUB)
    │   │   ├── ssh.yml      ← Harden sshd_config from template
    │   │   ├── fail2ban.yml ← SSH brute force protection
    │   │   ├── firewall.yml ← UFW: deny in, allow out, allow SSH
    │   │   └── motd.yml     ← Login banner
    │   ├── templates/
    │   │   └── sshd_config.j2  ← Hardened SSH config with Jinja2 variables
    │   └── vars/
    │       └── main.yml     ← ALL config values — edit here, not in tasks
    │
    └── webserver/           ← Nginx web server deployment
        ├── handlers/
        │   └── main.yml     ← Nginx reload/restart
        ├── tasks/
        │   ├── main.yml     ← Entry point: packages → firewall → configure → service
        │   ├── packages.yml ← Install nginx
        │   ├── firewall.yml ← Open port 80 in UFW
        │   ├── configure.yml ← Document root, site config, index.html
        │   └── service.yml  ← Ensure nginx is running and enabled
        ├── templates/
        │   ├── nginx_site.conf.j2  ← Virtual host config with security headers
        │   └── index.html.j2       ← Landing page using Ansible facts
        └── vars/
            └── main.yml     ← nginx_port, document_root, page title, etc.
```

---

## Prerequisites

Kali Linux (2024.x or 2025.x) with internet access.

---

## Setup (run once)

```bash
# 1. Update apt and install Ansible
sudo apt update
sudo apt install -y ansible

# 2. Verify installation
ansible --version

# 3. Clone the repo (or copy the ansible-lab folder to your home directory)
cd ~
git clone https://github.com/Server2090/CS3-Infra.git cs3-lab
cd cs3-lab/ansible-lab

# 4. Initialise git tracking for your own changes
git init
git add .
git commit -m "Initial commit: lab starting point"

# 5. Install collections (already bundled with apt ansible on Kali,
#    but run this as a safety net and in any other environment)
ansible-galaxy collection install -r requirements.yml
```

---

## Usage

```bash
# Always dry-run first — see what WOULD change without touching anything
ansible-playbook site.yml --check --diff

# Apply everything
ansible-playbook site.yml

# Apply only the hardening role
ansible-playbook site.yml --tags hardening

# Apply only the webserver role
ansible-playbook site.yml --tags webserver

# Apply a single task group (e.g. just SSH hardening)
ansible-playbook site.yml --tags hardening_ssh

# List all tasks without running them
ansible-playbook site.yml --list-tasks

# Verbose output for debugging
ansible-playbook site.yml -v
ansible-playbook site.yml -vvv   # very verbose — shows every module call
```

---

## Configuration

**All tunable values live in `vars/main.yml` files — never in task files.**

| File | Controls |
|---|---|
| `roles/hardening/vars/main.yml` | SSH port, fail2ban thresholds, sysctl params, banner text |
| `roles/webserver/vars/main.yml` | Nginx port, document root, page title |

To make a change:
1. Edit the relevant `vars/main.yml`
2. Run `ansible-playbook site.yml --check --diff` to preview
3. Run `ansible-playbook site.yml` to apply
4. `git add . && git commit -m "describe your change"`

---

## Verification Commands

```bash
# Kernel parameters
sysctl net.ipv6.conf.all.disable_ipv6   # should be 1
sysctl net.ipv4.tcp_syncookies           # should be 1
cat /etc/sysctl.d/99-cs3-hardening.conf

# Firewall
sudo ufw status verbose

# fail2ban
sudo systemctl status fail2ban
sudo fail2ban-client status sshd

# SSH
sudo sshd -T | grep -E "passwordauthentication|permitrootlogin|maxauthtries"

# Nginx
sudo systemctl status nginx
sudo nginx -t
firefox http://localhost &
```

---

## Git Workflow

```bash
# After every working change — save your progress
git add .
git commit -m "Short description of what you changed and why"

# See what has changed since last commit
git status
git diff

# See commit history
git log --oneline
```

---

## Relation to CS3-Infra

This lab reproduces a subset of what runs on the real CS3 server:

| CS3-Infra (production) | This lab |
|---|---|
| `ansible/roles/common/tasks/ipv6.yml` | `roles/hardening/tasks/ipv6.yml` |
| `ansible/roles/common/tasks/ssh.yml` | `roles/hardening/tasks/ssh.yml` |
| `ansible/roles/common/tasks/fail2ban/` | `roles/hardening/tasks/fail2ban.yml` |
| `ansible/roles/common/tasks/firewall/` | `roles/hardening/tasks/firewall.yml` |
| `ansible/roles/common/tasks/sysctl/` | `roles/hardening/tasks/sysctl.yml` |

The production version uses remote SSH targets, SOPS-encrypted secrets, and
the OS-dispatch pattern to support multiple distros. The lab simplifies to
`localhost` and Kali/Debian only — the concepts are identical.
