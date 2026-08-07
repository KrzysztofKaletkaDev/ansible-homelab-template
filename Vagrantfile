# -*- mode: ruby -*-
# ==============================================================================
# One-off VM for testing site.yml BEFORE deploying to a real host.
# Usage:
#   vagrant up        -> spins up the VM and immediately runs site.yml
#   vagrant provision  -> reruns the playbook on an already provisioned VM
#   vagrant destroy -f -> destroys the VM completely
# ==============================================================================

Vagrant.configure("2") do |config|
  config.vm.box = "almalinux/9"
  config.vm.hostname = "ansible-test-node"

  config.vm.network "private_network", ip: "192.168.56.10"

  # VM resources — adjust to your machine's capabilities
  config.vm.provider "libvirt" do |lv|
    lv.memory = 2048
    lv.cpus = 2
  end

  config.vm.provider "virtualbox" do |vb|
    vb.memory = 2048
    vb.cpus = 2
  end

  config.vm.provision "shell", inline: <<-SHELL
  dnf install -y python3-firewall firewalld
  systemctl enable --now firewalld
SHELL

  config.vm.provision "ansible" do |ansible|
    ansible.playbook = "site.yml"
    ansible.vault_password_file = ".vault_pass"

    # Vagrant creates its own inventory on the fly and assigns this VM
    # to the core_nodes group, exactly the one targeted by site.yml
    ansible.groups = {
      "core_nodes" => ["default"]
    }

    # The almalinux/9 box logs in as "vagrant", not "sysadmin" from group_vars/all/vars.yml
    # — we override this only for the test, so as not to touch the actual config
    ansible.extra_vars = {
      ansible_user: "vagrant",
      vault_cloudflare_api_token: "TEST-DUMMY-TOKEN-NIE-PRAWDZIWY",
      custom_dns_target_ip: "192.168.56.10"
    }
  end
end
