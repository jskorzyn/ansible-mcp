# Demo: AI-Powered Playbook Debugging with AAP + Claude Code

## The Story

A developer is writing a new Ansible playbook and pushes it to git.
They run it via AAP — it fails. Instead of digging through logs manually,
they ask Claude Code to investigate using the AAP MCP server. Claude reads
the job output, diagnoses the bug, fixes the playbook, commits the change,
and reruns — all from a single VS Code conversation.

The playbook has **three bugs** at different sophistication levels, making the
debugging session realistic and iterative.

---

## The Bugs

### Bug 1 — Wrong Jinja2 comparison operator (sophisticated)

```yaml
# line 13 — WRONG: =< is not valid Jinja2 syntax
when: ansible_memtotal_mb =< min_memory_mb

# CORRECT
when: ansible_memtotal_mb <= min_memory_mb
```

**Error produced:**
```
The conditional check 'ansible_memtotal_mb =< min_memory_mb' failed.
The error was: template error while templating string: unexpected char '=' at 25
```

### Bug 2 — Missing tool in execution environment (realistic)

```yaml
# line 17 — WRONG: iostat (sysstat package) is not installed in the AAP EE container
ansible.builtin.command: iostat -c 1 1

# CORRECT: use a command available in the EE, or install sysstat in a custom EE
```

**Error produced:**
```
fatal: [localhost]: FAILED! => {
    "msg": "[Errno 2] No such file or directory: b'iostat'"
}
```

### Bug 3 — Typo in file path (classic)

```yaml
# line 24 — WRONG: missing letter 'a'
src: /etc/os-relese

# CORRECT
src: /etc/os-release
```

**Error produced:**
```
fatal: [localhost]: FAILED! => {
    "msg": "file not found: /etc/os-relese"
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
- **Name**: `ansible-mcp-demo`
- **Source Control Type**: Git
- **Source Control URL**: `<your-git-url>`
- **Update Revision on Launch**: ✅  ← auto-syncs before every job run

### 3. Create a Job Template

AAP → Templates → Add → Job Template:
- **Name**: `System Report`
- **Inventory**: `Demo Inventory`
- **Project**: `ansible-mcp-demo`
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

Claude edits `playbooks/system_report.yml` line 13, commits and pushes.

---

### Run 2 — Hit Bug 2

**Prompt:**
> "Run it again"

Because **Update Revision on Launch** is enabled, AAP syncs the fix from git
automatically before running. The job gets past the memory check — but now
fails on the CPU utilization task.

**Prompt:**
> "What failed this time?"

Claude reads the output:
```
TASK [Get CPU utilization]
fatal: [localhost]: FAILED! => {
    "msg": "[Errno 2] No such file or directory: b'iostat'"
}
```

Claude diagnoses:
> *"`iostat` is part of the `sysstat` package which isn't installed in the AAP
> execution environment container. I'll replace it with an Ansible fact-based
> approach that works without external tools."*

Claude edits the task to use `ansible_processor_vcpus` and related facts
instead of `iostat`, commits and pushes.

---

### Run 3 — Hit Bug 3

**Prompt:**
> "Run it again"

The job gets past the CPU task — but now fails on the slurp task.

**Prompt:**
> "What failed this time?"

Claude reads the output:
```
TASK [Read OS release details]
fatal: [localhost]: FAILED! => {
    "msg": "file not found: /etc/os-relese"
}
```

Claude diagnoses:
> *"Typo in the file path — `/etc/os-relese` is missing the letter `a`,
> it should be `/etc/os-release`. Fixing now."*

Claude edits line 24, commits and pushes.

---

### Run 4 — Success

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
        "OS       : RedHat 9.7",
        "Version  : 9.7",
        "Kernel   : 5.14.0-570.113.1.el9_6.x86_64",
        "CPU      : 0.50    0.00    0.30    0.10    0.00   99.10",
        "Memory   : 386593 MB total",
        "Disk /   : 374G used of 447G (84% full)",
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
| Claude spots a missing EE package from errno 2 | AI knows execution environment constraints |
| Claude edits the file, commits, and pushes | Full developer workflow — AI as a coding partner |
| AAP auto-syncs and reruns without extra steps | "Update on launch" makes the loop seamless |
| Three iterations: find → fix → find → fix → pass | Realistic debugging is iterative; Claude handles it naturally |
