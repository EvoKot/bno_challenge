# Architecture & reasoning

## 1. Overview

Goal: provide a **reproducible, self-contained CI environment** where
GitHub Actions workflows for this repository run on **two self-hosted runners**
living on a Windows 11 laptop.

Stack:

- Host: Windows 11
- Virtualisation: VirtualBox + Vagrant
- Guest OS: Ubuntu 22.04 (jammy)
- Config management: Ansible (`ansible_local`)
- CI: GitHub Actions (self-hosted runners)

---

## 2. Components

### 2.1 Host (Windows 11)

- Runs VirtualBox and Vagrant.
- Provides local networking and disk for the VMs.
- Developer interacts via Git Bash and VS Code.

### 2.2 Vagrant VMs

Defined in `Vagrantfile`:

- `runner1`
  - Box: `ubuntu/jammy64`
  - Hostname: `runner1`
  - Labels: `self-hosted`, `bno`, `runner1`

- `runner2`
  - Box: `ubuntu/jammy64`
  - Hostname: `runner2`
  - Labels: `self-hosted`, `bno`, `runner2`

Each VM uses:

- **NAT** for outbound Internet access (GitHub, apt repos).
- **Host-only** adapter for deterministic addressing and easier debugging.

Provisioning uses `ansible_local` so the playbook executes inside the VM.

### 2.3 Ansible

Location: `ansible/`

- **Playbook**: `playbooks/site.yml`
  - Target: `all` (both VMs)
  - `pre_tasks`: validate that `gh_owner`, `gh_repo`, `gh_pat` are provided
    (coming from the host environment via Vagrant).
  - Role: `github_runner`

- **Role**: `roles/github_runner`
  - Installs required packages (curl, git, jq, tar, ansible, …)
  - Creates Linux user `runner`
  - Creates `/opt/actions-runner`
  - Queries GitHub API for the latest runner release
  - Downloads and extracts the runner
  - Uses `gh_pat` to obtain a short-lived **registration token**
  - Runs `config.sh` as user `runner` to register the node
  - Installs and starts the runner service
  - Ensures the service is running (idempotent)

### 2.4 GitHub Actions

- Repo: `GH_OWNER/GH_REPO`
- Runners: two self-hosted agents registered with labels:
  - `self-hosted`, `bno`, `runner1`
  - `self-hosted`, `bno`, `runner2`
- Workflow: `.github/workflows/selfhosted-smoketest.yml`
  - `on: workflow_dispatch`
  - Two jobs:
    - `on-runner1` – `runs-on: [self-hosted, bno, runner1]`
    - `on-runner2` – `runs-on: [self-hosted, bno, runner2]`
  - Each prints basic system info and exits.

---

## 3. Provisioning flow

1. Developer creates `.env` (based on `.env.example`) and runs:

   ```bash
    source .env
    vagrant up
2. Vagrant:
    Downloads the Ubuntu base box (first run only).
    Creates runner1 and runner2.
    Exports GH_OWNER, GH_REPO, GH_PAT into the guest environment.
    Triggers ansible_local provisioning.
3. Inside each VM, Ansible:
    Validates the GitHub variables.
    Installs dependencies and GitHub Actions runner.
    Registers the VM as a self-hosted runner to the repo.
    Starts the runner service.
4. In Github:
    The repo now shows two online self-hosted runners.
    Workflows that target the labels are scheduled onto these VMs.
## 4. Networking
1. Outbound:
    - Both VMs use NAT to access:
        - github.com/api.github.com
        - Ubuntu package mirros
2. Inbound / debugging:
    - Vagrant exposes SSH on localhost (e.g. ports 2222, 2200).
    - Shared folder /vagrant maps the repo from the host into each VM
      for visibility/debugging.
No ingress from outside the laptop is required; all traffic is
host↔GitHub and VM↔GitHub.
## 5. Security considerations
1. GitHub PAT:
    - Scope reduced to repo (enough to mint runner registration tokens).
    - Stored only in the local .env file (git-ignored).
    - Never hard-coded in playbooks or Vagrantfile.
2. Runners:
    - Dedicated VMs, not shared between repositories.
    - Runner process executed as non-root user runner.
    - Destroying the VMs via vagrant destroy fully wipes the environment.

### *** For a Production Setup ***
I would additionally consider:
1. Storing secrets in a dedicated secrets manager instead of .env.
2. Using short-lived OIDC or GitHub Apps instead of PATs.
3. Hardening images (CIS baselines, patch automation).
4. Autoscaling runners and isolation per workload.