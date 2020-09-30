# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure("2") do |config|
  config.vm.box = "ubuntu/bionic64"

  config.vm.box_check_update = true

  config.vm.network "forwarded_port", guest: 8200, host: 8200, host_ip: "127.0.0.1"

  config.vm.synced_folder '.', '/vagrant', disabled: true

  config.vm.provider "virtualbox" do |vb|
    vb.gui = false
    vb.memory = "2048" # 2GB
  end

  config.vm.provision "file", source: "./vault-enterprise_1.4.0+prem_linux_amd64.zip", destination: "/tmp/vault-enterprise_1.4.0+prem_linux_amd64.zip"

  config.vm.provision "shell", inline: <<-SHELL
    apt-get update
    apt-get install -y apt-transport-https unzip    
    mkdir -p /tmp/build
    cd /tmp/build
    mv /tmp/vault-enterprise_1.4.0+prem_linux_amd64.zip /tmp/build/vault-enterprise_1.4.0+prem_linux_amd64.zip
    unzip vault-enterprise_1.4.0+prem_linux_amd64.zip && \
    sudo mv vault /usr/local/bin/ && \
    cd /tmp
    rm -rf /tmp/build
    vault version
    vault -autocomplete-install
    complete -C /usr/local/bin/vault vault
   
    sudo setcap cap_ipc_lock=+ep /usr/local/bin/vault

    useradd --system --home /vault --shell /bin/false vault
    usermod -aG vault vault
    
    mkdir -p /vault/data && \
    mkdir -p /vault/config && \
    chown -R vault:vault /vault

    sudo touch /etc/systemd/system/vault.service
  SHELL

  config.vm.provision "file", source: "./vault.service", destination: "/tmp/vault.service"
  config.vm.provision "file", source: "./vault.license", destination: "/tmp/vault.license"

  config.vm.provision "shell", inline: <<-SHELL
    sudo mv /tmp/vault.service /etc/systemd/system/vault.service 
    sudo mv /tmp/vault.license /vault/config/vault.license
    chown -R vault:vault /vault/config/vault.license
    sudo systemctl daemon-reload
    sudo systemctl enable vault
    sudo systemctl start vault
    sudo systemctl status vault

    # wait for vault to start, then apply the license
    sleep 10

    TOKEN=$(sudo journalctl -u vault | grep "Root Token:" | awk -F ': ' '{print $3}')
    LICENSE=$(cat /vault/config/vault.license)

    echo "Root Token: $TOKEN"
    echo "{\\"text\\":\\"$LICENSE\\"}" > /tmp/license.json
    
    echo "Applying license from /tmp/license.json" && \  
      curl \
        --header "X-Vault-Token: $TOKEN" \
        --request PUT \
        --data @/tmp/license.json \
        http://0.0.0.0:8200/v1/sys/license && \
    echo "Checking license" && \
      curl \
        --header "X-Vault-Token: $TOKEN" \
        http://0.0.0.0:8200/v1/sys/license &

    sudo su root
    echo "export VAULT_ADDR='http://127.0.0.1:8200/'" >> /etc/profile
  SHELL

  config.trigger.after :up do |trigger|
    if RUBY_PLATFORM.include? "darwin"
      trigger.info = "Opening Vault Enterprise UI in Chrome"
      trigger.ruby do
        `if which open; then sleep 4 && open -a "Google Chrome" http://localhost:8200; fi`
      end
    end
  end
  
  config.trigger.after :up do |trigger|
    trigger.info = "Getting Vault Enterprise Unseal Key and Root Token"
    trigger.ruby do
      puts `vagrant ssh -c "sudo journalctl -u vault | grep -E 'Root|Unseal'"`
    end
  end
end
