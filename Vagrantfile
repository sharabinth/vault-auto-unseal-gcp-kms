# -*- mode: ruby -*-
# vi: set ft=ruby :

### Define environment variables to pass on to provisioner

# Define Vault Primary HA cluster details
VAULT_HA_SERVER_IP_PREFIX = ENV['VAULT_HA_SERVER_IP_PREFIX'] || "10.100.1.1"
VAULT_HA_SERVER_IPS = ENV['VAULT_HA_SERVER_IPS'] || '"10.100.1.11", "10.100.1.12"'

# Define Vault Secondary cluster details 
# This cluster can be configured as a DR cluster or a Performance Replica based on test requirements
VAULT_DR_SERVER_IP_PREFIX = ENV['VAULT_DR_SERVER_IP_PREFIX'] || "10.100.2.1"
VAULT_DR_SERVER_IPS = ENV['VAULT_DR_SERVER_IPS'] || '"10.100.2.11", "10.100.2.12"'

# Uncomment the below if a 3rd cluster is required
#VAULT_REPLICA_SERVER_IP_PREFIX = ENV['VAULT_REPLICA_SERVER_IP_PREFIX'] || "10.100.3.1"
#VAULT_REPLICA_SERVER_IPS = ENV['VAULT_REPLICA_SERVER_IPS'] || '"10.100.3.11", "10.100.3.12"'

# Specify the AWS KMS key value from the data.txt file as there are separate keys for Primary and Secondary
VAULT_PRIMARY_KMS_KEY = ENV['VAULT_PRIMARY_KMS_KEY'] || "aws_primary_kms"
VAULT_SECONDARY_KMS_KEY = ENV['VAULT_SECONDARY_KMS_KEY'] || "aws_secondary_kms"

Vagrant.configure("2") do |config|
  config.vm.box = "ubuntu/bionic64"
  config.vm.box_version = "20190411.0.0"

  # Consul UI for Primary would be available at port 8500 and can be accessed from any of the servers e.g., http://10.100.1.11:8500 
  # UI will only work if a leader is elected successfully
  # Vault UI for Primary will be available at port 8200 and can be accessed from any of the primary servers e.g., http://10.100.1.11:8200

  # Set up the 2 node Vault Primary HA cluster servers.  Each node will have a Consul server and a Vault server.
  # This is not recommended for Prod as separate Consul servers and Vault Server + Consul Client are required for Prod setup.
  (1..2).each do |i|
    config.vm.define "vault#{i}" do |v1|
      v1.vm.hostname = "v#{i}"
      
      #v1.vm.network "private_network", ip: "10.100.1.1#{i}"
      v1.vm.network "private_network", ip: VAULT_HA_SERVER_IP_PREFIX+"#{i}"
      v1.vm.provision "shell", 
              path: "scripts/setupConsulServer.sh",
              env: {'VAULT_HA_SERVER_IPS' => VAULT_HA_SERVER_IPS, 'VAULT_DC' => 'dc1'}

      v1.vm.provision "shell", 
              path: "scripts/setupVaultServer.sh",
              env: {'VAULT_KMS_KEY' => VAULT_PRIMARY_KMS_KEY}
    end
  end

  # Set up the 2 node Vault Secondary HA cluster servers
  # (1..2).each do |i|
  #   config.vm.define "vault-dr#{i}" do |v1|
  #     v1.vm.hostname = "v-dr#{i}"
      
  #     v1.vm.network "private_network", ip: VAULT_DR_SERVER_IP_PREFIX+"#{i}"
  #     v1.vm.provision "shell", 
  #             path: "scripts/setupConsulServer.sh",
  #             env: {'VAULT_HA_SERVER_IPS' => VAULT_DR_SERVER_IPS, 'VAULT_DC' => 'dc2'}

  #     v1.vm.provision "shell", 
  #             path: "scripts/setupVaultServer.sh",
  #             env: {'VAULT_KMS_KEY' => VAULT_SECONDARY_KMS_KEY}
  #   end
  # end

  # Uncomment the below if a 3rd HA cluster is required for testing
  # set up the 2 node Vault Secondary Replication servers
  # (1..2).each do |i|
  #   config.vm.define "vault-pr#{i}" do |v1|
  #     v1.vm.hostname = "v-pr#{i}"
      
  #     v1.vm.network "private_network", ip: VAULT_REPLICA_SERVER_IP_PREFIX+"#{i}"
  #     v1.vm.provision "shell", 
  #             path: "setupConsulServer.sh",
  #             env: {'VAULT_HA_SERVER_IPS' => VAULT_REPLICA_SERVER_IPS, 'VAULT_DC' => 'dc3'}

  #     v1.vm.provision "shell", 
  #             path: "setupVaultServer.sh",
  #             env: {'VAULT_KMS_KEY' => VAULT_SECONDARY_KMS_KEY}
  #   end
  # end

end
