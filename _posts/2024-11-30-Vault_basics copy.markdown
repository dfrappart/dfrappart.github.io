---
layout: post
title:  "A look at Hashicorp Vault - Getting started"
date:   2024-11-15 18:00:00 +0200
year: 2024
categories: Vault Security
---

Following our previous Vault 101, here is another Vault article, this time about some vault basics (again)

The Agenda:


1. Vault basics, authentication and first secrets
2. A bit of terraform for vault
3. Vault integration with AD DS

Let's get started!

## 1. Vault basics, authentication and first secrets

```bash

df@df2204lts:~/Documents/myrepo/vaultlab/vault_standalone_vagrant$ vault login
Token (will be hidden): 
Success! You are now authenticated. The token information displayed below
is already stored in the token helper. You do NOT need to run "vault login"
again. Future Vault requests will automatically use this token.

Key                  Value
---                  -----
token                hvs.xxxxxxxxxxxxxxxxxxxxxxxx
token_accessor       xxxxxxxxxxxxxxxxxxxxxxx
token_duration       ∞
token_renewable      false
token_policies       ["root"]
identity_policies    []
policies             ["root"]
df@df2204lts:~/Documents/myrepo/vaultlab/vault_standalone_vagrant$ vault policy list
default
root
df@df2204lts:~/Documents/myrepo/vaultlab/vault_standalone_vagrant$ vault secrets list
Path          Type         Accessor              Description
----          ----         --------              -----------
cubbyhole/    cubbyhole    cubbyhole_08e648c5    per-token private secret storage
identity/     identity     identity_0b7ce7f8     identity store
sys/          system       system_c760460c       system endpoints used for control, policy and debugging
df@df2204lts:~/Documents/myrepo/vaultlab/vault_standalone_vagrant$ vault auth list
Path      Type     Accessor               Description                Version
----      ----     --------               -----------                -------
token/    token    auth_token_447c9a81    token based credentials    n/a

```

```bash

df@df2204lts:~/Documents/myrepo/vaultlab/vault_standalone_vagrant$ vault list auth/userpass/users
Keys
----
david
penny
sheldon
df@df2204lts:~/Documents/myrepo/vaultlab/vault_standalone_vagrant$ vault read auth/userpass/users/david
Key                        Value
---                        -----
token_bound_cidrs          []
token_explicit_max_ttl     0s
token_max_ttl              0s
token_no_default_policy    false
token_num_uses             0
token_period               0s
token_policies             []
token_ttl                  0s
token_type                 default


df@df2204lts:~/Documents/myrepo/vaultlab/vault_standalone_vagrant$ vault auth enable userpass 
Success! Enabled userpass auth method at: userpass/
df@df2204lts:~/Documents/myrepo/vaultlab/vault_standalone_vagrant$ vault auth list
Path         Type        Accessor                  Description                Version
----         ----        --------                  -----------                -------
token/       token       auth_token_447c9a81       token based credentials    n/a
userpass/    userpass    auth_userpass_48819e2f    n/a                        n/a
df@df2204lts:~/Documents/myrepo/vaultlab/vault_standalone_vagrant$ export userpassword="xxxxxxxxxxxxxxxxxxxx"
df@df2204lts:~/Documents/myrepo/vaultlab/vault_standalone_vagrant$ vault write auth/userpass/users/david
Must supply data or use -force
df@df2204lts:~/Documents/myrepo/vaultlab/vault_standalone_vagrant$ vault write auth/userpass/users/david password=$userpassword
Success! Data written to: auth/userpass/users/david
df@df2204lts:~/Documents/myrepo/vaultlab/vault_standalone_vagrant$ vault write auth/userpass/users/sheldon password=$userpassword
Success! Data written to: auth/userpass/users/sheldon
df@df2204lts:~/Documents/myrepo/vaultlab/vault_standalone_vagrant$ vault write auth/userpass/users/penny password=$userpassword
Success! Data written to: auth/userpass/users/penny

```

```bash



df@df2204lts:~/Documents/myrepo/vaultlab/vault_standalone_vagrant$ curl --header "X-Vault-Token:hvs.xxxxxxxxxxxxxxxxxxxxxxxx" http://192.168.56.11:8200/v1/sys/auth/userpass/ | jq .
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100  1027  100  1027    0     0  1022k      0 --:--:-- --:--:-- --:--:-- 1002k
{
  "deprecation_status": "supported",
  "type": "userpass",
  "description": "",
  "accessor": "auth_userpass_48819e2f",
  "uuid": "da487b46-7230-86e2-a752-ea2b9bfd19b3",
  "plugin_version": "",
  "running_sha256": "",
  "local": false,
  "seal_wrap": false,
  "external_entropy_access": false,
  "options": null,
  "running_plugin_version": "v1.18.1+builtin.vault",
  "config": {
    "default_lease_ttl": 0,
    "force_no_cache": false,
    "max_lease_ttl": 0,
    "token_type": "default-service"
  },
  "request_id": "7542baf5-e205-2352-c50c-a91a32b33bea",
  "lease_id": "",
  "renewable": false,
  "lease_duration": 0,
  "data": {
    "accessor": "auth_userpass_48819e2f",
    "config": {
      "default_lease_ttl": 0,
      "force_no_cache": false,
      "max_lease_ttl": 0,
      "token_type": "default-service"
    },
    "deprecation_status": "supported",
    "description": "",
    "external_entropy_access": false,
    "local": false,
    "options": null,
    "plugin_version": "",
    "running_plugin_version": "v1.18.1+builtin.vault",
    "running_sha256": "",
    "seal_wrap": false,
    "type": "userpass",
    "uuid": "da487b46-7230-86e2-a752-ea2b9bfd19b3"
  },
  "wrap_info": null,
  "warnings": null,
  "auth": null,
  "mount_type": "system"
}

df@df2204lts:~/Documents/myrepo/vaultlab/vault_standalone_vagrant$ curl --request LIST --header "X-Vault-Token:hvs.xxxxxxxxxxxxxxxxxxxxxxxx" http://192.168.56.11:8200/v1/auth/userpass/users | jq .
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   218  100   218    0     0   302k      0 --:--:-- --:--:-- --:--:--  212k
{
  "request_id": "0ac05ed2-06e8-bf31-75b5-b593473896af",
  "lease_id": "",
  "renewable": false,
  "lease_duration": 0,
  "data": {
    "keys": [
      "david",
      "penny",
      "sheldon"
    ]
  },
  "wrap_info": null,
  "warnings": null,
  "auth": null,
  "mount_type": "userpass"
}

df@df2204lts:~/Documents/myrepo/vaultlab/vault_standalone_vagrant$ curl --header "X-Vault-Token:hvs.xxxxxxxxxxxxxxxxxxxxxxxx" http://192.168.56.11:8200/v1/auth/userpass/users/david | jq .
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   376  100   376    0     0   197k      0 --:--:-- --:--:-- --:--:--  367k
{
  "request_id": "293cbe23-9403-2898-091a-cab6f408e80e",
  "lease_id": "",
  "renewable": false,
  "lease_duration": 0,
  "data": {
    "token_bound_cidrs": [],
    "token_explicit_max_ttl": 0,
    "token_max_ttl": 0,
    "token_no_default_policy": false,
    "token_num_uses": 0,
    "token_period": 0,
    "token_policies": [],
    "token_ttl": 0,
    "token_type": "default"
  },
  "wrap_info": null,
  "warnings": null,
  "auth": null,
  "mount_type": "userpass"
}


df@df2204lts:~/Documents/myrepo/vaultlab/vault_standalone_vagrant$ vault secrets enable kv
Success! Enabled the kv secrets engine at: kv/
df@df2204lts:~/Documents/myrepo/vaultlab/vault_standalone_vagrant$ vault secrets list
Path          Type         Accessor              Description
----          ----         --------              -----------
cubbyhole/    cubbyhole    cubbyhole_08e648c5    per-token private secret storage
identity/     identity     identity_0b7ce7f8     identity store
kv/           kv           kv_7c21795d           n/a
sys/          system       system_c760460c       system endpoints used for control, policy and debugging


df@df2204lts:~/Documents/dfrappart.github.io$ vault kv put kv/david/secret1 target=thisisasecret
Success! Data written to: kv/david/secret1
df@df2204lts:~/Documents/dfrappart.github.io$ vault kv get kv/david/secret1
===== Data =====
Key       Value
---       -----
target    thisisasecret
df@df2204lts:~/Documents/dfrappart.github.io$ vault kv put kv/sheldon/secret1 target=iworkondarkmatternow!
Success! Data written to: kv/sheldon/secret1
df@df2204lts:~/Documents/dfrappart.github.io$ vault kv get kv/sheldon/secret1
===== Data =====
Key       Value
---       -----
target    iworkondarkmatternow!
df@df2204lts:~/Documents/dfrappart.github.io$ vault kv put kv/penny/secret1 target=imnotanactressiworkforbigpharma
Success! Data written to: kv/penny/secret1
df@df2204lts:~/Documents/dfrappart.github.io$ vault kv get kv/penny/secret1
===== Data =====
Key       Value
---       -----
target    imnotanactressiworkforbigpharma


```

## 2. A bit of terraform for vault


## 3. Vault integration with AD DS


## 4. Summary

In this article, 