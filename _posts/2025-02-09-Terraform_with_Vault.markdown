---
layout: post
title:  "Terraform with Vault"
date:   2025-02-09 18:00:00 +0200
year: 2025
categories: Vault Security
---

In our last [Vault article](/_posts/2025-01-20-Vault_basics.markdown), we went beyond the simple bootstrap of a Vault server (not a cluster yet &#128539).
So we created some users with the userpass auth method, policies and kv stores.
It obvioulsy can take some time to create policies, kv stores and such Vault objects manually, in addition to being error prone.

The good thing here, is that there is a terraform provider for Vault.
In this article we'll have a look at this and recreate what we've done last time, but with some terraform configuration

The Agenda:


1. Preparing the use of Terraform
2. Creating stuff in Vault with Terraform


Let's get started!


## 1. Preparing the use of Terraform

To use Terraform, we need a backend configuration, to avoid the local state. 
And because I'm an Azure guy, I'll use an Azure backend.

Now about the provider, we can check the [documentation](https://registry.terraform.io/providers/hashicorp/vault/latest/docs), and we'll find that there are quite a lot of options to authenticate on a Vault server.

In our case, we start from where we are, meaning with userpass authentication. We'll see the details in the next section.
We want to create a user that will be used for all our terraform configuration.


```bash

yumemaru@df-2404lts:~$ export userpassword="xxxxxxxxxxxxxxxxxxxx"
yumemaru@df-2404lts:~$ vault write auth/userpass/users/terraform password=$userpassword
Success! Data written to: auth/userpass/users/terraform

```

By default, this user only have access in accordance to the default policy, which is not much. So if we want to use it to create resource, we need to create a kind of admin policy. This can also be found on the documentation, and we get a policy as below:

```go

# Read system health check
path "sys/health"
{
  capabilities = ["read", "sudo"]
}

# Create and manage ACL policies broadly across Vault

# List existing policies
path "sys/policies/acl"
{
  capabilities = ["list"]
}

# Create and manage ACL policies
path "sys/policies/acl/*"
{
  capabilities = ["create", "read", "update", "delete", "list", "sudo"]
}

# Enable and manage authentication methods broadly across Vault

# Manage auth methods broadly across Vault
path "auth/*"
{
  capabilities = ["create", "read", "update", "delete", "list", "sudo"]
}

# Create, update, and delete auth methods
path "sys/auth/*"
{
  capabilities = ["create", "update", "delete", "sudo"]
}

# List auth methods
path "sys/auth"
{
  capabilities = ["read"]
}

# Enable and manage the key/value secrets engine at `secret/` path

# List, create, update, and delete key/value secrets
path "secret/*"
{
  capabilities = ["create", "read", "update", "delete", "list", "sudo"]
}

# List, create, update, and delete key/value secrets under the path kvstores/ added by me
path "kvstores/*"
{
  capabilities = ["create", "read", "update", "delete", "list", "sudo"]
}

# Manage secrets engines
path "sys/mounts/*"
{
  capabilities = ["create", "read", "update", "delete", "list", "sudo"]
}

# List existing secrets engines.
path "sys/mounts"
{
  capabilities = ["read"]
}



```
Notice the section where we added `kvstores/*` path

```go

path "kvstores/*"
{
  capabilities = ["create", "read", "update", "delete", "list", "sudo"]
}

```

That's typically where we want to create all our kv stores.

To create the policy and add it to the terraform user we use vault cli.

```bash

df@df-2404lts:~$ vault policy write admin-policy ./admin-policy.hcl
Success! Uploaded policy: admin-policy

df@df-2404lts:~$ vault write auth/userpass/users/terraform policies="admin-policy"
Success! Data written to: auth/userpass/users/terraform

df@df-2404lts:~$ vault read auth/userpass/users/terraform
Key                        Value
---                        -----
policies                   [admin-policy]
token_bound_cidrs          []
token_explicit_max_ttl     0s
token_max_ttl              0s
token_no_default_policy    false
token_num_uses             0
token_period               0s
token_policies             [admin-policy]
token_ttl                  0s
token_type                 default

``` 

And with that, we should be good to go.



## 2. Creating stuff with Terraform

  

### 2.1. Init the terraform config

To get started, we need the provider, and a backend. 
With an `azurerm` backend we have the following configuration:


```go

terraform {
  required_version = ">= 1.10.0"
  required_providers {
    vault = {
      source  = "hashicorp/vault"
      version = "2.20.0"
    }
  }

  backend "azurerm" {}
}

provider "vault" {
  address = "http://192.168.56.11:8200"

  #auth_login_userpass {
  #  username = "terraform"
  #  password_file = "./terraformvaultpwd.txt"
  #}

  auth_login {
    path = "auth/userpass/login/terraform"
    parameters = {
      username = "terraform"
      password = file("./terraformvaultpwd.txt")
    }
  }

}

```

You will notice the commented `auth_login_userpass` which... I was enable to configure correctly &#128517;.

The `auth_login` (which I was able to configure correctly &#128537;) block allows us to specify the userpass path to specify the login user, in this case the `terraform` user that we created in the previous section.

We'll note also the password that we fill through a file which contains only the password of the user. We do need to create this user though.

### 2.2. Create a kv store


Refering to the terraform registry, we can write the terraform configuration for the kv store, in v2 version.
We need to specify the path, and accordingly to what we defined in the ppolicy, we use `kvstores/bebopkv`

```go

resource "vault_mount" "kvv2" {
  path = "kvstores/bebopkv"
  type = "kv-v2"
  options = {
    version = "2"
    type    = "kv-v2"
  }
  description = "This is an KV Version 2 secret engine mount created by terraform for the Bebop crew"
}

```
  
### 2.3. Create another userpass auth
  

Adding an userpass authentication backend is done as below.
Because it's another userpass backend, this time we do need to specify a path, which we did not with the first one in cli.

```go

resource "vault_auth_backend" "userpass" {
  type        = "userpass"
  path        = "yetanotheruserpath"
  description = "This is an userpass auth backend created by terraform, for the bebop crew"

  tune {
    max_lease_ttl      = "90000s"
    listing_visibility = "unauth"
  }
}

```

We won't create users here, because I did not find the corresponding resource. There are however resources for other kind of user, and we will certainly have a look at this in others posts.
  

### 2.4. Template policies
  

Next we look at policies, for users that do not exist yet, but you see the idea.

```go

variable "bebopusers" {
  type = map(object({
    username = string
    password = string

  }))
  default = {
    "user1" = {
      username = "Spike"
      password = "user1"
    }
    "user2" = {
      username = "Faye"
      password = "user2"
    }
    "user3" = {
      username = "Jet"
      password = "user3"
    }
    "user4" = {
      username = "Ed"
      password = ""
    }
    "user5" = {
      username = "Ein"
      password = ""
    }
  }
  
}

resource "vault_policy" "accessbebopkv" {
  for_each = var.bebopusers
  name = "${each.value.username}policy"

  policy = templatefile("${path.root}/policies/bebopkvtemplate.hcl", {
    Kvv2StoreName    = vault_mount.kvv2.path
    UserPassUserName = each.value.username
  })
}



```

Notice the use of the `templatefile` function. For familiar terraform users, this is the same function that you may have used to template stuffs on you cloud provider.

The templated policy looked like this:


```go

path "kvstores/${Kvv2StoreName}/data/${UserPassUserName}/*" {
  capabilities = ["list", "create", "update", "read", "patch", "delete"]
}

path "kvstores/${Kvv2StoreName}/metadata/${UserPassUserName}/*" {
  capabilities = ["list", "create", "update", "read", "patch", "delete"]
}

path "kvstores/${Kvv2StoreName}/*" {
  capabilities = ["list"]
}

path "kvstores/${Kvv2StoreName}/data/+/sharedsecrets/*" {
  capabilities = ["read"]
}

```

Let's apply Now
  

### 2.5. Creating and cheking the resources
  

The plan should gives us something looking like that:


```bash


Terraform used the selected providers to generate the following execution
plan. Resource actions are indicated with the following symbols:
  + create

Terraform will perform the following actions:

  # vault_auth_backend.userpass will be created
  + resource "vault_auth_backend" "userpass" {
      + accessor                  = (known after apply)
      + default_lease_ttl_seconds = (known after apply)
      + description               = "This is an example userpass auth backend created by terraform"
      + id                        = (known after apply)
      + listing_visibility        = (known after apply)
      + max_lease_ttl_seconds     = (known after apply)
      + path                      = "yetanotheruserpath"
      + tune                      = [
          + {
              + allowed_response_headers     = []
              + audit_non_hmac_request_keys  = []
              + audit_non_hmac_response_keys = []
              + listing_visibility           = "unauth"
              + max_lease_ttl                = "90000s"
              + passthrough_request_headers  = []
                # (2 unchanged attributes hidden)
            },
        ]
      + type                      = "userpass"
    }

  # vault_mount.kvv2 will be created
  + resource "vault_mount" "kvv2" {
      + accessor                  = (known after apply)
      + default_lease_ttl_seconds = (known after apply)
      + description               = "This is an KV Version 2 secret engine mount created by terraform"
      + external_entropy_access   = false
      + id                        = (known after apply)
      + max_lease_ttl_seconds     = (known after apply)
      + options                   = {
          + "type"    = "kv-v2"
          + "version" = "2"
        }
      + path                      = "kvstores/bebopkv"
      + seal_wrap                 = (known after apply)
      + type                      = "kv-v2"
    }

  # vault_policy.accessbebopkv["user1"] will be created
  + resource "vault_policy" "accessbebopkv" {
      + id     = (known after apply)
      + name   = "Spikepolicy"
      + policy = <<-EOT
            path "kvstores/kvstores/bebopkv/data/Spike/*" {
              capabilities = ["list", "create", "update", "read", "patch", "delete"]
            }
            
            path "kvstores/kvstores/bebopkv/metadata/Spike/*" {
              capabilities = ["list", "create", "update", "read", "patch", "delete"]
            }
            
            path "kvstores/kvstores/bebopkv/*" {
              capabilities = ["list"]
            }
            
            path "kvstores/kvstores/bebopkv/data/+/sharedsecrets/*" {
              capabilities = ["read"]
            }
        EOT
    }

  # vault_policy.accessbebopkv["user2"] will be created
  + resource "vault_policy" "accessbebopkv" {
      + id     = (known after apply)
      + name   = "Fayepolicy"
      + policy = <<-EOT
            path "kvstores/kvstores/bebopkv/data/Faye/*" {
              capabilities = ["list", "create", "update", "read", "patch", "delete"]
            }
            
            path "kvstores/kvstores/bebopkv/metadata/Faye/*" {
              capabilities = ["list", "create", "update", "read", "patch", "delete"]
            }
            
            path "kvstores/kvstores/bebopkv/*" {
              capabilities = ["list"]
            }
            
            path "kvstores/kvstores/bebopkv/data/+/sharedsecrets/*" {
              capabilities = ["read"]
            }
        EOT
    }

  # vault_policy.accessbebopkv["user3"] will be created
  + resource "vault_policy" "accessbebopkv" {
      + id     = (known after apply)
      + name   = "Jetpolicy"
      + policy = <<-EOT
            path "kvstores/kvstores/bebopkv/data/Jet/*" {
              capabilities = ["list", "create", "update", "read", "patch", "delete"]
            }
            
            path "kvstores/kvstores/bebopkv/metadata/Jet/*" {
              capabilities = ["list", "create", "update", "read", "patch", "delete"]
            }
            
            path "kvstores/kvstores/bebopkv/*" {
              capabilities = ["list"]
            }
            
            path "kvstores/kvstores/bebopkv/data/+/sharedsecrets/*" {
              capabilities = ["read"]
            }
        EOT
    }

  # vault_policy.accessbebopkv["user4"] will be created
  + resource "vault_policy" "accessbebopkv" {
      + id     = (known after apply)
      + name   = "Edpolicy"
      + policy = <<-EOT
            path "kvstores/kvstores/bebopkv/data/Ed/*" {
              capabilities = ["list", "create", "update", "read", "patch", "delete"]
            }
            
            path "kvstores/kvstores/bebopkv/metadata/Ed/*" {
              capabilities = ["list", "create", "update", "read", "patch", "delete"]
            }
            
            path "kvstores/kvstores/bebopkv/*" {
              capabilities = ["list"]
            }
            
            path "kvstores/kvstores/bebopkv/data/+/sharedsecrets/*" {
              capabilities = ["read"]
            }
        EOT
    }

  # vault_policy.accessbebopkv["user5"] will be created
  + resource "vault_policy" "accessbebopkv" {
      + id     = (known after apply)
      + name   = "Einpolicy"
      + policy = <<-EOT
            path "kvstores/kvstores/bebopkv/data/Ein/*" {
              capabilities = ["list", "create", "update", "read", "patch", "delete"]
            }
            
            path "kvstores/kvstores/bebopkv/metadata/Ein/*" {
              capabilities = ["list", "create", "update", "read", "patch", "delete"]
            }
            
            path "kvstores/kvstores/bebopkv/*" {
              capabilities = ["list"]
            }
            
            path "kvstores/kvstores/bebopkv/data/+/sharedsecrets/*" {
              capabilities = ["read"]
            }
        EOT
    }

Plan: 7 to add, 0 to change, 0 to destroy.


```

Once the apply is complete (it should not take long if the Vault server is local), we can check the resources.

To get more familiar with the API, we'll check everything with curl ^^

**Get authentication backend:**

```bash

df@df-2404lts:~/Documents/dfrappart.github.io$ curl --header $vaultheader http://192.168.56.11:8200/v1/sys/auth | jq .data
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100  3251    0  3251    0     0  1011k      0 --:--:-- --:--:-- --:--:-- 1058k
{
  "token/": {},
  "userpass/": {},
  "yetanotheruserpath/": {
    "accessor": "auth_userpass_f11282b1",
    "config": {
      "allowed_response_headers": [
        ""
      ],
      "audit_non_hmac_request_keys": [
        ""
      ],
      "audit_non_hmac_response_keys": [
        ""
      ],
      "default_lease_ttl": 0,
      "force_no_cache": false,
      "listing_visibility": "unauth",
      "max_lease_ttl": 90000,
      "passthrough_request_headers": [
        ""
      ],
      "token_type": "default-service"
    },
    "deprecation_status": "supported",
    "description": "This is an example userpass auth backend created by terraform",
    "external_entropy_access": false,
    "local": false,
    "options": null,
    "plugin_version": "",
    "running_plugin_version": "v1.18.3+builtin.vault",
    "running_sha256": "",
    "seal_wrap": false,
    "type": "userpass",
    "uuid": "4a1c4645-5885-5888-f45a-f9f3d7d6dd51"
  }
}

```

**Get policies**

```bash

df@df-2404lts:~/Documents/dfrappart.github.io$ curl --request LIST --header $vaultheader http://192.168.56.11:8200/v1/sys/policy | jq .data.policies
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   789  100   789    0     0   131k      0 --:--:-- --:--:-- --:--:--  154k
[
  "admin-policy",
  "default",
  "edpolicy",
  "einpolicy",
  "fayepolicy",
  "jetpolicy",
  "spikepolicy",
  "tbbtkv-david",
  "tbbtkv-penny",
  "tbbtkv-sheldon",
  "root"
]

```

Then **get the details of one policy:**

```bash

df@df-2404lts:~/Documents/dfrappart.github.io$ curl --header $vaultheader http://192.168.56.11:8200/v1/sys/policy/fayepolicy | jq .
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100  1153  100  1153    0     0   396k      0 --:--:-- --:--:-- --:--:--  562k
{
  "name": "fayepolicy",
  "rules": "path \"kvstores/kvstores/bebopkv/data/Faye/*\" {\n  capabilities = [\"list\", \"create\", \"update\", \"read\", \"patch\", \"delete\"]\n}\n\npath \"kvstores/kvstores/bebopkv/metadata/Faye/*\" {\n  capabilities = [\"list\", \"create\", \"update\", \"read\", \"patch\", \"delete\"]\n}\n\npath \"kvstores/kvstores/bebopkv/*\" {\n  capabilities = [\"list\"]\n}\n\npath \"kvstores/kvstores/bebopkv/data/+/sharedsecrets/*\" {\n  capabilities = [\"read\"]\n}\n\n",
  "request_id": "0fb59209-2e9f-302f-07f1-64c2b7c8a0b0",
  "lease_id": "",
  "renewable": false,
  "lease_duration": 0,
  "data": {
    "name": "fayepolicy",
    "rules": "path \"kvstores/kvstores/bebopkv/data/Faye/*\" {\n  capabilities = [\"list\", \"create\", \"update\", \"read\", \"patch\", \"delete\"]\n}\n\npath \"kvstores/kvstores/bebopkv/metadata/Faye/*\" {\n  capabilities = [\"list\", \"create\", \"update\", \"read\", \"patch\", \"delete\"]\n}\n\npath \"kvstores/kvstores/bebopkv/*\" {\n  capabilities = [\"list\"]\n}\n\npath \"kvstores/kvstores/bebopkv/data/+/sharedsecrets/*\" {\n  capabilities = [\"read\"]\n}\n\n"
  },
  "wrap_info": null,
  "warnings": null,
  "auth": null,
  "mount_type": "system"
}

```

And from this policy, we can look at the details of the authorized kv store.

**Get kv store:**

```bash

df@df-2404lts:~/Documents/dfrappart.github.io$ curl --header $vaultheader http://192.168.56.11:8200/v1/kvstores/bebopkv/config | j
q .
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   243  100   243    0     0  83706      0 --:--:-- --:--:-- --:--:--  118k
{
  "request_id": "f397f6a8-08be-44da-cf7c-bb5d733dc095",
  "lease_id": "",
  "renewable": false,
  "lease_duration": 0,
  "data": {
    "cas_required": false,
    "delete_version_after": "0s",
    "max_versions": 0
  },
  "wrap_info": null,
  "warnings": null,
  "auth": null,
  "mount_type": "kv"
}

```

Ok that's about all for today, let's summarize

## 3. Summary

In this article, following our previous [Vault basics](/_posts/2025-01-20-Vault_basics.markdown), we added some automation.
Because Vault is also hashicorp, it's quite easy to automate its configuration with its IaC tool, even if it's more config mgmt than IaC, but hey, who cares? 

As for any IaC preparation, we need credentials, and sufficiantly elevated privileges to create the resources we want.
Once that's done, it's terraform as usual if you'll allow me.

Ok that will be all. Hope it was useful.
I'll come back soon for other Vault stuff 
