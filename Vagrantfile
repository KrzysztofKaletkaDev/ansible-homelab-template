# -*- mode: ruby -*-
# ==============================================================================
# Jednorazowa VM do testowania site.yml PRZED wdrożeniem na realny hosta.
# Użycie:
#   vagrant up        -> stawia VM i od razu odpala site.yml
#   vagrant provision  -> ponownie uruchamia playbook na już postawionej VM
#   vagrant destroy -f -> kasuje VM całkowicie
# ==============================================================================

Vagrant.configure("2") do |config|
  config.vm.box = "almalinux/9"
  config.vm.hostname = "ansible-test-node"

  config.vm.network "private_network", ip: "192.168.56.10"

  # Zasoby VM — dostosuj do możliwości swojej maszyny
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

    # Vagrant tworzy własny inventory w locie i przypisuje tę VM
    # do grupy core_nodes, dokładnie tej, którą celuje site.yml
    ansible.groups = {
      "core_nodes" => ["default"]
    }

    # Box almalinux/9 loguje się jako "vagrant", nie "sysadmin" z group_vars/all/vars.yml
    # — nadpisujemy to tylko na potrzeby testu, żeby nie ruszać właściwego configu
    ansible.extra_vars = {
      ansible_user: "vagrant",
      vault_cloudflare_api_token: "TEST-DUMMY-TOKEN-NIE-PRAWDZIWY",
      custom_dns_target_ip: "192.168.56.10"
    }
  end
end
