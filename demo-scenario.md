# Demo: AI-Powered Playbook Debugging with AAP + Claude Code

## The Story

A developer is writing a new Ansible playbook and pushes it to git.
They run it via AAP — it fails. Instead of digging through logs manually,
they ask Claude Code to investigate using the AAP MCP server. Claude reads
the job output, diagnoses the bug, fixes the playbook, commits the change,
triggers a project sync, and reruns — all from a single VS Code conversation.

The playbook has **two bugs** at different sophistication levels, making the
debugging session realistic and iterative.

---

## The Bugs

### Bug 1 — Wrong Jinja2 comparison operator (sophisticated)

```yaml
# line 15 — WRONG: =< is not valid Jinja2 syntax
when: ansible_memtotal_mb =< min_memory_mb

# CORRECT
when: ansible_memtotal_mb <= min_memory_mb
```

**Error produced:**
```
The conditional check 'ansible_memtotal_mb =< min_memory_mb' failed.
The error was: template error while templating string: unexpected char '=' at 25
```

### Bug 2 — Typo in file path (classic)

```yaml
# line 27 — WRONG: missing letter 'a'
src: /etc/os-relese

# CORRECT
src: /etc/os-release
```

**Error produced:**
```
fatal: [localhost]: FAILED! => {
    "msg": "Could not find or access '/etc/os-relese' on the Ansible Controller."
}
```

---

## One-time AAP Setup

### 1. Push this repo to git

```bash
git init
git remote add origin <your-git-url>
git add .
git commit -m "Add system_report playbook"
git push -u origin main
```

### 2. Create an AAP Project

AAP → Projects → Add:
- **Name**: `MCP Demo`
- **Source Control Type**: Git
- **Source Control URL**: `<your-git-url>`
- **Update Revision on Launch**: ✅  ← auto-syncs before every job run

### 3. Create a Job Template

AAP → Templates → Add → Job Template:
- **Name**: `System Report`
- **Inventory**: `Demo Inventory`
- **Project**: `MCP Demo`
- **Playbook**: `playbooks/system_report.yml`
- **Credentials**: none required

### 4. Add localhost to Demo Inventory

AAP → Inventories → Demo Inventory → Hosts → Add:
- **Name**: `localhost`
- **Variables**:
  ```yaml
  ansible_connection: local
  ```

---

## Demo Script

### Run 1 — Hit Bug 1

**Prompt:**
> "Run the System Report job template"

Claude launches the job. It fails on the memory check task.

**Prompt:**
> "What went wrong? Show me the error"

Claude calls `jobs_stdout_retrieve` via MCP and reads:
```
TASK [Check memory meets minimum requirement]
fatal: [localhost]: FAILED! => {
    "msg": "The conditional check 'ansible_memtotal_mb =< min_memory_mb' failed.
            The error was: template error while templating string:
            unexpected char '=' at 25"
}
```

Claude diagnoses:
> *"`=<` is not valid Jinja2 — the correct operator is `<=`. This is a flipped
> comparison operator. I'll fix it in the playbook."*

Claude edits `playbooks/system_report.yml` line 15, commits and pushes:
```bash
git add playbooks/system_report.yml
git commit -m "Fix: correct Jinja2 comparison operator =< to <="
git push
```

---

### Run 2 — Hit Bug 2

**Prompt:**
> "Run it again"

Because **Update Revision on Launch** is enabled, AAP syncs the fix from git
automatically before running. The job gets past the memory check — but now
fails on the slurp task.

**Prompt:**
> "What failed this time?"

Claude reads the output:
```
TASK [Read OS release details]
fatal: [localhost]: FAILED! => {
    "msg": "Could not find or access '/etc/os-relese' on the Ansible Controller."
}
```

Claude diagnoses:
> *"Typo in the file path — `/etc/os-relese` is missing the letter `a`,
> it should be `/etc/os-release`. Fixing now."*

Claude edits line 27, commits and pushes:
```bash
git add playbooks/system_report.yml
git commit -m "Fix: correct typo in os-release path"
git push
```

---

### Run 3 — Success

**Prompt:**
> "Run it one more time"

AAP syncs the latest fix and the job completes successfully:

```
TASK [Display system report]
ok: [localhost] => {
    "msg": [
        "=============================",
        "     SYSTEM REPORT",
        "=============================",
        "Hostname : aap-controller",
        "OS       : RedHat 9.4",
        "Version  : 9.4",
        "Kernel   : 5.14.0-427.13.1.el9_4.x86_64",
        "Uptime   : up 3 days, 4 hours, 12 minutes",
        "Memory   : 7821 MB total",
        "Disk /   : 8.3G used of 100G (9% full)",
        "============================="
    ]
}
```

---

## What the Demo Shows

| What the audience sees | What it demonstrates |
|------------------------|----------------------|
| Claude launches jobs without touching AAP UI | MCP gives AI direct access to the automation platform |
| Claude reads raw job output and explains the error | AI reasoning on real operational data |
| Claude identifies a Jinja2 syntax bug from an error message | AI understands Ansible internals, not just error strings |
| Claude edits the file, commits, and pushes | Full developer workflow — AI as a coding partner |
| AAP auto-syncs and reruns without extra steps | "Update on launch" makes the loop seamless |
| Two iterations: find → fix → find → fix → pass | Realistic debugging is iterative; Claude handles it naturally |
