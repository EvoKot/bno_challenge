# B&O DevOps Challenge – Self-hosted GitHub Actions Runners

This repo provisions **two self-hosted GitHub Actions runners** for this repository
using **Vagrant + VirtualBox** and **Ansible (ansible_local)**.  
The goal is to spin up an isolated CI environment on a Windows 11 laptop and
smoke-test that both runners can execute a simple workflow.

---

## High-level design

- **Host**: Windows 11
- **Vagrant provider**: VirtualBox
- **VMs**: 2× Ubuntu 22.04 (`runner1`, `runner2`)
- **Provisioning**: `ansible_local` inside each VM
- **Runners**: GitHub Actions self-hosted runners registered to this repo
- **Labels**:
  - common: `self-hosted`, `bno`
  - per-node: `runner1`, `runner2`
- **Workflow**: `.github/workflows/selfhosted-smoketest.yml`  
  Runs a small smoke test on each runner via `workflow_dispatch`.

---

## Prerequisites

On the **Windows 11** host:

- [VirtualBox 7.x](https://www.virtualbox.org/)
- [Vagrant 2.4.x](https://developer.hashicorp.com/vagrant)
- Git + Git Bash
- A GitHub **Personal Access Token (classic)** with at least **`repo`** scope  
  (used only to mint short-lived runner registration tokens).

You’ll also need:

- ~4–6 GB free RAM and ~20 GB disk for the two VMs.
- Internet access from the VMs (NAT is fine).

---

## Setup
1. Clone the repo
```bash```
git clone https://github.com/EvoKot/bno_challenge.git
cd bno_challenge
2. Create your .env from the example
```bash```
cp .env.example .env
 - Edit .env and set your values:
    GH_OWNER=EvoKot          # GitHub username or org (no @)
    GH_REPO=bno_challenge    # repo name only
    GH_PAT=ghp_**********************  # PAT with repo scope
3. Bring up the runners
source .env       # export GH_OWNER / GH_REPO / GH_PAT
vagrant up        # creates & provisions runner1 and runner2
 - What happens:
    1. Vagrant downloads the ubuntu/jammy64 box.
    2. Two VMs are created: runner1, runner2.
    3. ansible_local runs inside each VM using ansible/playbooks/site.yml.
    4. The github_runner role:
    5. Installs required packages (curl, git, jq, tar, ansible, …)
    6. Creates user runner
    7. Downloads the latest GitHub Actions runner
    8. Registers it to GH_OWNER/GH_REPO with labels self-hosted, bno, runnerX
    9. Ensures the runner service is installed and running
4. Verify runners in GitHub
 - In the GitHub UI:
Settings → Actions → Runners
You should see:
runner1 – online or Idle
runner2 – online or Idle
 - Running the smoke test
The smoke test workflow is defined in
.github/workflows/selfhosted-smoketest.yml.
Go to the Actions tab in GitHub.
Select “Self-hosted runners smoke test”.
Click “Run workflow” (it’s workflow_dispatch only).
Observe:
Job on-runner1 runs on runner1.
Job on-runner2 runs on runner2.
- Each job prints basic context (runner name, hostname, kernel version) to prove 
the job actually executed on the self-hosted VM.

## Project Layout
.
├─ Vagrantfile                 # Defines runner1 & runner2 VMs + ansible_local
├─ .env.example                # Template for local env vars (no secrets)
├─ ansible/
│  ├─ playbooks/
│  │  └─ site.yml             # Main playbook, calls github_runner role
│  └─ roles/
│     └─ github_runner/
│        ├─ tasks/main.yml    # Install + configure GitHub Actions runner
│        └─ handlers/main.yml # Service handler (restart, etc.)
├─ .github/
│  └─ workflows/
│     └─ selfhosted-smoketest.yml  # Workflow targeting self-hosted runners
└─ docs/
   └─ architecture.md          # Design/architecture notes
## Re-provisioning and cleanup
1. Re-run provisioning for a VM (e.g. after changing Ansible):
```bash```
source .env
vagrant provision runner1
vagrant provision runner2
2. Stop VMs
`bash
vagrant halt
3. Destroy everything (disk space cleanup):
```bash```
vagrant destroy -f
## Design notes / trade-offs
1. Vagrant + VirtualBox
Chosen to keep the whole setup reproducible on a personal laptop without
touching any cloud account.
2. ansible_local
Avoids extra SSH/WinRM complexity on Windows host; Ansible runs directly
inside each VM.
3. Secrets handling
PAT is kept only in .env (local, git-ignored).
Runner registration is done via GitHub’s short-lived registration tokens.
4. Simplicity over full production
For the exercise I keep things single-host, static, and easy to debug.
In a real environment I’d look into autoscaling runners (containers/VMs),
central secrets management, hardened base images, etc.