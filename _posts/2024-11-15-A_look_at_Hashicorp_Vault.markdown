---
layout: post
title:  "A look at Hashicorp Vault - Getting started"
date:   2024-11-15 18:00:00 +0200
year: 2024
categories: Vault Security
---

It's been a long time coming!
I decided to start seriously looking into Hashicorp Vault.
I love the product but by lack of time, never started the journey.
So what better way to start than initiating a few blog posts on the topic &#128523;

This article is Vault 101, so we'll look at the basics.

The Agenda:


1. What is Vault
2. A local lab with Vagrant
3. Basics with Vault

Let's get started!

## 1. What is Vault

I wonder if we really need to introduce Vault. It's becoming something of a must have in the secrets management landscape (amon others things).

But well, just to be thorough, let's describe what we have.

Vault from Hashicorp is a Secrets Engine.
Right, which means ?

Well, if, like me, you're dealing in a Public Cloud environment, you probably came accross times when you needed to store, and even more, manage secrets. Those secrets being password, key, or certificate.

If, again, like me, you're dealing with Azure, you probably also came accross Azure Keyvault, which, all in all, could be described as a `vault as a service`.
Notice the lack of capital `V` here. When I say vault, it means the concepts of vault to securely store secrets.

Well, Vault (with a capital `V`) is also a `vault as a service` but also so much more.

Obviously, because it's so much more, it also comes with much more management to do.

So for now, let's admit that Vault is one way to push the secret management to the next level. 

Considering a (very) higl level view of the architecture, we can consider 3 main elements

- The Vault API
- The Vault Backend Storage
- The Vault core components

![illustration1](/assets/vault/vaulthighlevelview.png)

The Vault API, as one would expect, is the entry point to access Vault components.

Those components are the core of Vault functionnalities. We have for instance, the secrets engines, the authentication methods or the token store. Things that we will have a look at when we go deeper in our Vault journey.

Because all of those components interacts with sensitive data, they're included in what is called the Barrier, the Vault encryption layer.
Inside this barrier occurs the encryption/decryption actions.

We'll note that the backend storage is outside of the barrier, meaning that it's not trusted. And that's why Vault encrypt the data **before** sending data to be written to the storage.

For more details, please refer to [Vault architecture page](https://developer.hashicorp.com/vault/docs/internals/architecture).

Now, still about the architecture, there are considerations for High availability, but I prefer to keep that for another time.

Which leaves us with one topic to introduce a little bit more: the backend storage.

Vault writes everything that need to be persisted in its backend storage, which makes this storage highly critical.
We already mentionned that everything was encrypted before being written. Let's focus a bit more on what we can use as a backend storage.

We have a lot of choice actually, but only 2 categories, from Vault architecture point of view:

- Integrated storage
- External storage

In the external storage, we can find many well known storage options, either in the relational database family, or in the cloud storage area. Example include:

- Azure storage
- AWS S3
- Google storage
- Postgresql
- MySql
- Cassandra

A more exhaustive list can be found on the [documentation](https://developer.hashicorp.com/vault/docs/configuration/storage).

All of thos options are well proven in traditional and cloud native architecture and it can makes sense to choose from one of those.
However, the Integrated storage option is nowadays the recommended solution from Hashicorp. Taking into considerations the following tableextracted again fro mthe documentation it does make sense:

|  | Integrated Storage	| External Storage |
|-|-|-|
|HashiCorp Supported | Yes | Limited support |
|Operation	| Operationally simpler with no additional software installation required.	| Must install and configure the external storage environment outside of Vault. For high availability, the external storage should be clustered. |
|Networking	| One less network hop.	| Extra network hop between Vault and the external storage system (e.g., Consul cluster).
|Troubleshooting and monitoring	| Integrated Storage is a part of Vault; therefore, Vault is the only system you need to monitor and troubleshoot. | The source of failure could be the external storage; therefore, you need to check the health of both Vault and the external storage. This requires expertise in the chosen storage backend and additional monitoring of that storage. |
|Data location	| The encrypted Vault data is stored on the same host where the Vault server process runs.	| The encrypted Vault data is stored where the external storage is located. Therefore, the Vault server and the data storage are hosted on physically separate hosts. |

Now that we've read the marketing part, let's dig a little in the underlying technology of this integrated storage.

It relies on the [Raft Consensus Algorithm](https://raft.github.io/), which, simply put, allows to replicate data and maintain consistency between multiple nodes. When data is submitted to the leader node by a client, it will interact with the others nodes to replicate the data, validate it was propagated and then commit and validate the new state. This is called the distributed consensus. 

In the schema below, `node1` is the leader, and the other nodes are followers.

![illustration2](/assets/vault/consensus.png)

A very nice explanation is available on the [secret live of data](http://thesecretlivesofdata.com/raft/).

Another critical point to remember is the number of nodes, which should always be odd. That's related to the leader election, important in the high availability also. But we'll keep that for another time.
Now it's time to start experimenting a bit

## 2. A local lab with Vagrant

### 2.1. Basic configuration review

It's very easy to have a ephemeral lab of Vault, thanks to the dev server.

But that's not what we would like to achieve. Instead, we will create a local lab with the help of another Hashicorp tool which is Vagrant.

There are many doc and tutorials on this tool, and I honestly found the configuration for this lab somewhere on the Internet.
To get started, we need a Vagrantfile file, somehow similar to docker, except it's all virtual machine. The vagrant file content is like this:

```ruby

VAULTSERVER_COUNT = 1
IMAGEVAULT = "bento/ubuntu-22.04"

Vagrant.configure("2") do |config|

  (1..VAULTSERVER_COUNT).each do |i|
    config.vm.define "vaultserver#{i}" do |vaultservers|
      vaultservers.vm.box = IMAGEVAULT
      vaultservers.vm.hostname = "vaultserver#{i}"
      vaultservers.vm.network  :private_network, ip: "192.168.56.#{i+10}"
      vaultservers.vm.provision "file", source: "./config/vaultconfig.hcl", destination: "/tmp/config.hcl"
      vaultservers.vm.provision "shell", privileged: true,  path: "scripts/vault_install.sh"
    end
  end
end


```

We are using the vagrant box `bento/ubuntu-22.04` which, as its name implies, gives us an ubuntu in our virtual machine.
We can note also the IP configuration with the line `vaultservers.vm.network  :private_network, ip: "192.168.56.#{i+10}"`, which match the network range of our virtual box configuration.

![illustration3](/assets/vault/vaultvbox001.png)

Next, we copy a `vaultconfig.hcl` file to the vagrant box, before using a `vault_install.sh` script

```go

storage "raft" {
  path    = "/opt/vault/data"
  node_id = "node1"
}

listener "tcp" {
  address     = "0.0.0.0:8200"
  tls_disable = "true"
}

api_addr = "http://0.0.0.0:8200"
cluster_addr = "https://serverip:8201"
ui = true

```


```bash

#!/bin/sh

echo "127.0.1.1 $(hostname)" >> /etc/hosts

# Get current IP adress

export currentip=$(/sbin/ip -o -4 addr list enp0s8 | awk '{print $4}' | cut -d/ -f1)

# Add Hashicorp repository

wget -O- https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list

# Install vault

sudo apt update
sudo apt install vault net-tools apt-transport-https gnupg curl wget bat -y

# Set up UFW

sudo ufw allow OpenSSH
sudo ufw allow https

for i in 8200/tcp 8201/tcp 8250/tcp 443/tcp
do sudo ufw allow $i
done

#sudo ufw enable -y

# prepare vault configuration

sed -i 's/serverip/'$currentip'/g' /tmp/config.hcl
mv /tmp/config.hcl /etc/vault.d/config.hcl

# Create user without login for vault service
sudo adduser vault --shell=/bin/false --no-create-home --disabled-password --gecos GECOS

# granting access to vault user
chmod 700 -R /opt/vault
chown vault -R /opt/vault

```

Initiating the creation of our virtual box vm with the `vagrant up` command, we should have an output looking like this:

```bash

yumemaru@df2204lts:~/vaultlab/vault_standalone_vagrant$ vagrant up
Bringing machine 'vaultserver1' up with 'virtualbox' provider...
==> vaultserver1: Importing base box 'bento/ubuntu-22.04'...
==> vaultserver1: Matching MAC address for NAT networking...
==> vaultserver1: Checking if box 'bento/ubuntu-22.04' version '202309.08.0' is up to date...
==> vaultserver1: Setting the name of the VM: vault_standalone_vagrant_vaultserver1_1731619580815_26084
==> vaultserver1: Clearing any previously set network interfaces...
==> vaultserver1: Preparing network interfaces based on configuration...
    vaultserver1: Adapter 1: nat
    vaultserver1: Adapter 2: hostonly
==> vaultserver1: Forwarding ports...
    vaultserver1: 22 (guest) => 2222 (host) (adapter 1)
==> vaultserver1: Booting VM...
========================truncated========================
    vaultserver1: No VM guests are running outdated hypervisor (qemu) binaries on this host.
    vaultserver1: Rules updated
    vaultserver1: Rules updated (v6)
    vaultserver1: Rules updated
    vaultserver1: Rules updated (v6)
    vaultserver1: Rules updated
    vaultserver1: Rules updated (v6)
    vaultserver1: Rules updated
    vaultserver1: Rules updated (v6)
    vaultserver1: Rules updated
    vaultserver1: Rules updated (v6)
    vaultserver1: Rules updated
    vaultserver1: Rules updated (v6)
    vaultserver1: adduser: The user `vault' already exists.

```

We should be able to access the box with a `vagrant ssh` command now and get access to our playground:

```bash


df@df2204lts:~/Documents/myrepo/vaultlab/vault_standalone_vagrant$ vagrant ssh
Welcome to Ubuntu 22.04.3 LTS (GNU/Linux 5.15.0-83-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

  System information as of Thu Nov 14 09:30:21 PM UTC 2024

  System load:  0.02978515625      Processes:             157
  Usage of /:   13.9% of 30.34GB   Users logged in:       0
  Memory usage: 13%                IPv4 address for eth0: 10.0.2.15
  Swap usage:   0%                 IPv4 address for eth1: 192.168.56.11


This system is built by the Bento project by Chef Software
More information can be found at https://github.com/chef/bento

```
We should find the vault config file in the `/etc/vault.d/config.hcl` file

```bash

vagrant@vaultserver1:~$ batcat /etc/vault.d/config.hcl 
───────┬───────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
       │ File: /etc/vault.d/config.hcl
───────┼───────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
   1   │ storage "raft" {
   2   │   path    = "/opt/vault/data"
   3   │   node_id = "node1"
   4   │ }
   5   │ 
   6   │ listener "tcp" {
   7   │   address     = "0.0.0.0:8200"
   8   │   tls_disable = "true"
   9   │ }
  10   │ 
  11   │ api_addr = "http://0.0.0.0:8200"
  12   │ cluster_addr = "https://192.168.56.11:8201"
  13   │ ui = true
───────┴───────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
vagrant@vaultserver1:~$ 

```

We want to start our vault server based on this config so we will use the `vault server` command:

```bash

vagrant@vaultserver1:~$ vault server -config /etc/vault.d/config.hcl 
Error initializing storage of type raft: failed to create fsm: failed to open bolt file: error checking raft FSM db file "/opt/vault/data/vault.db": stat /opt/vault/data/vault.db: permission denied
2024-11-14T21:34:51.595Z [INFO]  proxy environment: http_proxy="" https_proxy="" no_proxy=""

```

And it does not work !

But it makes sense, there is  no reason that we should access the path `/opt/vault/data` with the vagrant user.
Let's just verify tha the configuration is ok by launching the vault command with sudo.

```bash

vagrant@vaultserver1:~$ sudo vault server -config /etc/vault.d/config.hcl &
[1] 3737
vagrant@vaultserver1:~$ ==> Vault server configuration:

Administrative Namespace: 
             Api Address: http://0.0.0.0:8200
                     Cgo: disabled
         Cluster Address: https://192.168.56.11:8201
   Environment Variables: HOME, LANG, LOGNAME, LS_COLORS, MAIL, PATH, SHELL, SUDO_COMMAND, SUDO_GID, SUDO_UID, SUDO_USER, TERM, USER
              Go Version: go1.22.8
              Listener 1: tcp (addr: "0.0.0.0:8200", cluster address: "0.0.0.0:8201", disable_request_limiter: "false", max_request_duration: "1m30s", max_request_size: "33554432", tls: "disabled")
               Log Level: 
                   Mlock: supported: true, enabled: true
           Recovery Mode: false
                 Storage: raft (HA available)
                 Version: Vault v1.18.1, built 2024-10-29T14:21:31Z
             Version Sha: f479e5c85462477c9334564bc8f69531cdb03b65

==> Vault server started! Log data will stream in below:

2024-11-14T21:42:28.438Z [INFO]  proxy environment: http_proxy="" https_proxy="" no_proxy=""
2024-11-14T21:42:28.442Z [INFO]  incrementing seal generation: generation=1
2024-11-14T21:42:28.489Z [INFO]  core: Initializing version history cache for core
2024-11-14T21:42:28.489Z [INFO]  events: Starting event system

```

Checking the status of vault with the vault status command, we get a proper answer after setting the `VAULT_ADDR` environment variable:

```bash

vagrant@vaultserver1:~$ vault status
WARNING! VAULT_ADDR and -address unset. Defaulting to https://127.0.0.1:8200.
Error checking seal status: Get "https://127.0.0.1:8200/v1/sys/seal-status": http: server gave HTTP response to HTTPS client
vagrant@vaultserver1:~$ export VAULT_ADDR="http://192.168.56.11:8200"
vagrant@vaultserver1:~$ vault status
2024-11-14T21:43:17.341Z [INFO]  core: security barrier not initialized
2024-11-14T21:43:17.341Z [INFO]  core: seal configuration missing, not initialized
Key                Value
---                -----
Seal Type          shamir
Initialized        false
Sealed             true
Total Shares       0
Threshold          0
Unseal Progress    0/0
Unseal Nonce       n/a
Version            1.18.1
Build Date         2024-10-29T14:21:31Z
Storage Type       raft
HA Enabled         true

```

So it does work, but it's not "reboot persistent", and well, it's really not a good idea to start apps as root so let's try to do better.

```bash

vagrant@vaultserver1:~$ export VAULT_ADDR="http://192.168.56.11:8200"
vagrant@vaultserver1:~$ vault status
Error checking seal status: Get "http://192.168.56.11:8200/v1/sys/seal-status": dial tcp 192.168.56.11:8200: connect: connection refused

```

### 2.2. Configuring Vault as a service

This time, we will add a configuration in the `vault_install.sh` so that Vault is configured as a service.
For this, we can find what we need on the [vault documentation](https://developer.hashicorp.com/vault/docs/run-as-service).

In our case, we will add the following in our script:

```bash
sudo tee -a /etc/systemd/system/vault.service > /dev/null <<EOT
[Unit]
Description="Hashicorp Vault"
Documentation="https://developer.hashicorp.com/vault/docs"
Requires=network-online.target
After=network-online.target
ConditionFileNotEmpty=/etc/vault.d/config.hcl
[Service]
User=vault
Group=vault
ProtectSystem=full
ProtectHome=read-only
PrivateTmp=yes
PrivateDevices=yes
SecureBits=keep-caps
AmbientCapabilities=CAP_IPC_LOCK
Capabilities=CAP_IPC_LOCK+ep
CapabilityBoundingSet=CAP_SYSLOG CAP_IPC_LOCK
NoNewPrivileges=yes
ExecStart=/usr/bin/vault server -config=/etc/vault.d/config.hcl
ExecReload=/bin/kill --signal HUP $MAINPID
KillMode=process
KillSignal=SIGINT
Restart=on-failure
RestartSec=5
TimeoutStopSec=30
StartLimitInterval=60
StartLimitBurst=3
LimitNOFILE=65536
LimitMEMLOCK=infinity

[Install]
WantedBy=multi-user.target
EOT
# Enable and start the service
sudo systemctl daemon-reload
sudo systemctl start vault.service
sudo systemctl enable vault.service

```

To avoid using the root account, we specified a vault user earlier with some authorization:

```bash

# granting access to vault user
chmod 700 -R /opt/vault
chown vault -R /opt/vault

```

After rebuilding our box, we have a working vault service:

```bash

vagrant@vaultserver1:~$ systemctl status vault
● vault.service - "Hashicorp Vault"
     Loaded: loaded (/etc/systemd/system/vault.service; enabled; vendor preset: enabled)
     Active: active (running) since Thu 2024-11-14 22:07:24 UTC; 40s ago
       Docs: https://developer.hashicorp.com/vault/docs
   Main PID: 2537 (vault)
      Tasks: 8 (limit: 2219)
     Memory: 134.6M
        CPU: 157ms
     CGroup: /system.slice/vault.service
             └─2537 /usr/bin/vault server -config=/etc/vault.d/config.hcl

Nov 14 22:07:24 vaultserver1 vault[2537]:                    Mlock: supported: true, enabled: true
Nov 14 22:07:24 vaultserver1 vault[2537]:            Recovery Mode: false
Nov 14 22:07:24 vaultserver1 vault[2537]:                  Storage: raft (HA available)
Nov 14 22:07:24 vaultserver1 vault[2537]:                  Version: Vault v1.18.1, built 2024-10-29T14:21:31Z
Nov 14 22:07:24 vaultserver1 vault[2537]:              Version Sha: f479e5c85462477c9334564bc8f69531cdb03b65
Nov 14 22:07:24 vaultserver1 vault[2537]: ==> Vault server started! Log data will stream in below:
Nov 14 22:07:24 vaultserver1 vault[2537]: 2024-11-14T22:07:24.836Z [INFO]  proxy environment: http_proxy="" https_proxy="" no_proxy=""
Nov 14 22:07:24 vaultserver1 vault[2537]: 2024-11-14T22:07:24.859Z [INFO]  incrementing seal generation: generation=1
Nov 14 22:07:24 vaultserver1 vault[2537]: 2024-11-14T22:07:24.922Z [INFO]  core: Initializing version history cache for core
Nov 14 22:07:24 vaultserver1 vault[2537]: 2024-11-14T22:07:24.922Z [INFO]  events: Starting event system

```

```bash
yumemaru@df2204lts:~/vaultlab/vault_standalone_vagrant$ vault status
Key                Value
---                -----
Seal Type          shamir
Initialized        false
Sealed             true
Total Shares       0
Threshold          0
Unseal Progress    0/0
Unseal Nonce       n/a
Version            1.18.1
Build Date         2024-10-29T14:21:31Z
Storage Type       raft
HA Enabled         true

```

Ok, the playground is ready, let's begin the serious stuff now ^^


## 3. Basics with Vault

### 3.1. Unsealing vault

### 3.2. Change the root token

### 3.3. Vault auto unseal with key vault

### 3.4. Basic authentication & secrets management

## 4. Summary

In this article 



