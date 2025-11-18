# Architecture & reasoning

**Goal:** Two Ubuntu 22.04 VMs act as self-hosted GitHub Actions runners for a private repository.

- Vagrant orchestrates the VMs (VirtualBox provider).
- Ansible (ansible_local) runs inside each VM to configure the runner.
- GitHub API is used to mint a short-lived registration token using your PAT.
- The Actions runner is installed under `/opt/actions-runner`, configured as user `runner`,
  then installed as a service with `svc.sh`.

Labels: `bno` and the host name (`runner1` or `runner2`).

Re-running the playbook is safe (idempotent); it checks for `/opt/actions-runner/.runner`
before re-registering the runner.
