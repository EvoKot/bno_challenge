# -*- mode: ruby -*-
# vi: set ft=ruby :
#
# B&O DevOps Challenge - Vagrantfile
#
# Creates two Ubuntu 22.04 VMs and provisions each as a GitHub Actions self-hosted runner.
# Before running `vagrant up`, set these environment variables on the host:
#   GH_OWNER  -> your GitHub username or org (no @)
#   GH_REPO   -> your PRIVATE repository name (only the name, not 'owner/name')
#   GH_PAT    -> a GitHub Personal Access Token (classic) with 'repo' scope to mint runner registration tokens

Vagrant.configure("2") do |config|
  config.vm.box = "ubuntu/jammy64"   # Ubuntu 22.04

  servers = [
    { :hostname => "runner1", :ip => "192.168.56.11", :memory => 2048, :cpus => 2 },
    { :hostname => "runner2", :ip => "192.168.56.12", :memory => 2048, :cpus => 2 }
  ]

  servers.each do |srv|
    config.vm.define srv[:hostname] do |node|
      node.vm.hostname = srv[:hostname]
      node.vm.network "private_network", ip: srv[:ip]

      node.vm.provider "virtualbox" do |vb|
        vb.name = "bno-#{srv[:hostname]}"
        vb.memory = srv[:memory]
        vb.cpus   = srv[:cpus]
      end

      # Keep the project synced into /vagrant inside the guest
      node.vm.synced_folder ".", "/vagrant"

      # Ensure Ansible and a few utilities exist in the guest (ansible_local runs inside the VM)
      node.vm.provision "shell", inline: <<-SHELL
        sudo apt-get update -y
        sudo DEBIAN_FRONTEND=noninteractive apt-get install -y ansible curl jq tar git
      SHELL

      # Run Ansible inside the VM
      node.vm.provision "ansible_local" do |ansible|
        ansible.playbook = "/vagrant/ansible/playbooks/site.yml"

        # The following variables are injected into Ansible.
        # Make sure they exist in your environment before 'vagrant up'.
        ansible.extra_vars = {
          gh_owner: ENV['GH_OWNER'],
          gh_repo:  ENV['GH_REPO'],
          gh_pat:   ENV['GH_PAT'],
          runner_extra_labels: "bno"
        }

        ansible.limit = srv[:hostname]
      end
    end
  end
end
