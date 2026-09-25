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
      # caddy-dns/cloudflare v0.2.4 rejects malformed tokens at startup and Caddy
      # crash-loops, so the dummy must look like a real one (40 chars of [A-Za-z0-9-])
      vault_cloudflare_api_token: "TEST-DUMMY-TOKEN-NIE-PRAWDZIWY-000000000",
      vault_grafana_admin_password: "test-dummy-password",
      custom_dns_target_ip: "192.168.56.10",
      vault_cloudflared_tunnel_id: "00000000-0000-0000-0000-000000000000",
      vault_cloudflared_credentials_json: "{\"AccountTag\":\"TEST-DUMMY-NIE-PRAWDZIWY\",\"TunnelSecret\":\"VEVTVC1EVU1NWS1TRUNSRVQ=\",\"TunnelID\":\"00000000-0000-0000-0000-000000000000\",\"Endpoint\":\"\"}",
      # bcrypt of "test-dummy-password" (caddy hash-password)
      vault_camera_wall_password_hash: "$2a$14$igZl2JGNhtwWMFmNbIV7iedGrPAIvmefe6GRXlZniKCtkG1FX.jxW",
      # Unreachable TEST-NET cameras: the run checks structure, permissions and
      # idempotence, not video. The passwords exercise percent-encoding of
      # "%", "/", "@" and ":", and the empty user of a password-only account.
      vault_go2rtc_credentials: {
        "dummy" => { "user" => "test-dummy", "password" => "TEST%%dummy/NIE@PRAWDZIWE:1" },
        "dummy_nouser" => { "user" => "", "password" => "TEST%%/dummy" }
      },
      go2rtc_cameras: [
        { "id" => "test1", "label" => "Test 1", "host" => "192.0.2.101", "main_path" => "/stream1",
          "sub_path" => "/stream2", "credentials" => "dummy", "mode" => "mse", "fit" => "contain" },
        { "id" => "test2", "label" => "Test 2", "host" => "192.0.2.102", "sub_path" => "/12",
          "credentials" => "dummy_nouser", "transcode" => true, "mode" => "webrtc", "fit" => "contain" }
      ]
    }
  end
end
