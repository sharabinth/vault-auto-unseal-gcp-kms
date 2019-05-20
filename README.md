# Vault Auto Unseal Feature using Google Cloud KMS Setup
This repo has a vagrant file to create an enterprise Vault Cluster with Consul backend.  
Seal stanza is added to the Vault config file to setup auto unseal using Google Cloud KMS.
Vagrant file also creates a secondary cluster with auto unseal with Google KMS using the same KMS ring.  This secondary can be configured as a DR or Performance Replication to perform further tests.

Each cluster contains 2 nodes and each note consists of a Consul Server and Vault server.  
The configuration is used for learning purposes.  This is NOT following the reference architecture for Vault and should not be used for a Production setup.

```
Cluster 1: Vault Primary cluster in DC1 

Cluster 2: Vault Secondary cluster in DC2 

```

All servers are set without TLS.

## Pre-Requisites
Create a folder named as ```ent``` and copy both the Consul and Vault enterprise binary zip files.  Since auto unseal with Google Cloud KMS feature is available in the Open Source version, you could copy this version into the ```ent``` folder.

```e.g., consul-enterprise_1.4.5+prem_linux_amd64.zip```

Rename ```data.txt.example``` to ```data.txt``` and update the values to specify the GCP related information such as Region, Project, Location of the Credentials File, Key Ring and Crypto Key.

## Vault Primary Cluster
2 node cluster is created with each node containing Vault and Consul servers. The server details are shown below

```
vault1   10.100.1.11
vault2   10.100.1.12
```

One of the Consul servers would become the leader.  Similarly one of Vault servers would become the Active node and the other node acts as Read Replica.

## Vault Secondary Cluster
2 node cluster is created with each node containing Vault and Consul servers. The server details are shown below

```
vault-dr1   10.100.2.11
vault-dr2   10.100.2.12
```

The vagrant file also has commented section to create another secondary cluster if required.  Check the content of the Vagrant file.

## Usage
If the ubuntu box is not available then it will take sometime to download the base box for the first time.  After the initial download, servers can be destroyed and recreated quickly with Vagrant

```
$vagrant up

$vagrant status

```

To check the status of the servers ssh into one of the nodes and check the cluster members and identify the leader.

```
$vagrant ssh vault1

vagrant@v1:~$ consul members
Node  Address           Status  Type    Build      Protocol  DC   Segment
v1    10.100.1.11:8301  alive   server  1.5.0+ent  2         dc1  <all>
v2    10.100.1.12:8301  alive   server  1.5.0+ent  2         dc1  <all>

vagrant@v1:~$ consul operator raft list-peers
Node  ID                                    Address           State     Voter  RaftProtocol
v1    1719e2f2-ee6d-6b17-fad9-8857771d9ce3  10.100.1.11:8300  leader    true   3
v2    a76db518-996d-4b8b-aa1c-d79eab584814  10.100.1.12:8300  follower  true   3

vagrant@v1: $consul info

vagrant@v1:~$ vault status
Key                      Value
---                      -----
Recovery Seal Type       gcpckms
Initialized              false
Sealed                   true
Total Recovery Shares    0
Threshold                0
Unseal Progress          0/0
Unseal Nonce             n/a
Version                  n/a
HA Enabled               true

```

If ```vault status``` throws an error then check the Google Cloud related information specified in the ```data.txt``` file.

Vault status would be shown as uninitialised and sealed.  By default the Recovery Seal Type is set to ```gcpkms```.

## Initialising and Unsealing Vault

Perform the following to initialise the Vault cluster and this should unseal both vault nodes due to ```gcpkms``` stanza.  
Initialisation is only required at one of the servers.

Vault can be initialised with Recovery keys.  In this case, the Recovery Seal Type would be set to ```Shamir```
If no recovery keys are requested then Recovery Seal Type would remain as ```gcpkms```
Having Recovery Keys would be useful as a last resort in case ```gcp kms``` is accidently removed.

```
$vagrant ssh vault1

vagrant@v1: $vault status

vagrant@v1:~$ vault operator init
Recovery Key 1: W7KR4i/Aeu+Qw18CaBOVcGkM777zdIWeZEBhxTZE3LBW
Recovery Key 2: tZ8/72ZmMn0tKDzujiN6pu0aVt9C+Ot78b+m94XtHAVn
Recovery Key 3: aRdjC5o2gXUo5AeYSBI5bGLMVsFQ7wtwnHA0e2FiUQI1
Recovery Key 4: EWpgEBqAi3IpPDiT7mEg6SETXjzg3OvKKN3BWiYAkHca
Recovery Key 5: RwVQ+fs/CT/yRnOO4KIXvJ+ESLVPWyKkG1np8U6iNswP

Initial Root Token: s.E6zuhm6X9Fkx0WI7Ew5iUbmD

Success! Vault is initialized

Recovery key initialized with 5 key shares and a key threshold of 3. Please
securely distribute the key shares printed above.

vagrant@v1:~$ vault status
Key                      Value
---                      -----
Recovery Seal Type       shamir
Initialized              true
Sealed                   false
Total Recovery Shares    5
Threshold                3
Version                  1.1.2+prem
Cluster Name             vault-cluster-401b3cf6
Cluster ID               219835c5-b661-6ebf-a771-d04618e67871
HA Enabled               true
HA Cluster               https://10.100.1.12:8201
HA Mode                  standby
Active Node Address      http://10.100.1.12:8200

vagrant@v1: $exit

$vagrant ssh vault2

vagrant@v2:~$ vault status
Key                      Value
---                      -----
Recovery Seal Type       shamir
Initialized              true
Sealed                   false
Total Recovery Shares    5
Threshold                3
Version                  1.1.2+prem
Cluster Name             vault-cluster-401b3cf6
Cluster ID               219835c5-b661-6ebf-a771-d04618e67871
HA Enabled               true
HA Cluster               https://10.100.1.12:8201
HA Mode                  active
Last WAL                 16

```

## Testing Auto Unseal Feature

Once the Vault is initialised, it would be unsealed by the use of AWS KMS.   This is verified in the previous step.

When Vault is restarted, it would automatically unseal using AWS KMS.

```
vagrant@v1: $ sudo systemctl restart vault
vagrant@v1:~$ sudo systemctl status vault
 vault.service - "Vault secret management tool"
   Loaded: loaded (/etc/systemd/system/vault.service; enabled; vendor preset: enabled)
   Active: active (running) since Mon 2019-05-20 11:07:51 UTC; 6s ago
 Main PID: 2768 (vault)
    Tasks: 9 (limit: 1152)
   CGroup: /system.slice/vault.service
           └─2768 /usr/local/bin/vault server -config=/etc/vault/vault.hcl

May 20 11:07:57 v1 vault[2768]:                      Cgo: disabled
May 20 11:07:57 v1 vault[2768]:          Cluster Address: https://10.100.1.11:8201
May 20 11:07:57 v1 vault[2768]:               Listener 1: tcp (addr: "0.0.0.0:8200", cluster address: "0.0.0.0:8201", max_request_duration: "1m30s", max_request_size: "33
May 20 11:07:57 v1 vault[2768]:                Log Level: info
May 20 11:07:57 v1 vault[2768]:                    Mlock: supported: true, enabled: true
May 20 11:07:57 v1 vault[2768]:                  Storage: consul (HA available)
May 20 11:07:57 v1 vault[2768]:                  Version: Vault v1.1.2+prem
May 20 11:07:57 v1 vault[2768]:              Version Sha: c1bd8914ddf4a3e3f7d0b683b0c5670076166932
May 20 11:07:57 v1 vault[2768]: ==> Vault server started! Log data will stream in below:
May 20 11:07:57 v1 vault[2768]: 2019-05-20T11:07:57.499Z [INFO]  core: stored unseal keys supported, attempting fetch



vagrant@v1:~$ vault status
Key                                    Value
---                                    -----
Recovery Seal Type                     shamir
Initialized                            true
Sealed                                 false
Total Recovery Shares                  5
Threshold                              3
Version                                1.1.2+prem
Cluster Name                           vault-cluster-401b3cf6
Cluster ID                             219835c5-b661-6ebf-a771-d04618e67871
HA Enabled                             true
HA Cluster                             https://10.100.1.12:8201
HA Mode                                standby
Active Node Address                    http://10.100.1.12:8200
Performance Standby Node               true
Performance Standby Last Remote WAL    0


vagrant@v1: $exit

$vagrant ssh vault2

vagrant@v2:~$ vault status
Key                      Value
---                      -----
Recovery Seal Type       shamir
Initialized              true
Sealed                   false
Total Recovery Shares    5
Threshold                3
Version                  1.1.2+prem
Cluster Name             vault-cluster-401b3cf6
Cluster ID               219835c5-b661-6ebf-a771-d04618e67871
HA Enabled               true
HA Cluster               https://10.100.1.12:8201
HA Mode                  active
Last WAL                 16

```

In the above, it should be noted that Node 1 has now become the Standby.  When Node 1 was restarted, Node 2 became the leader.

## Accessing UI

Use one of the server nodes to access the Consul UI on port 8500 and the Vault UI on port 8200.  The UI for Consul will not work if the leader is not elected.

e.g., Consul UI http://10.100.1.11:8500 

e.g., Vault UI http://10.100.2.11:8500 


