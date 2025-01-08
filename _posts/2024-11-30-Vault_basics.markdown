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

### 1.1. Where we are

At the end of the previous article, we had a working stand alone Vault server.
At this point, we can start the server, and because we configured auto unseal with Azure keyvault, we get an unseal server after boot

```bash

yumemaru@df-2404lts:~$ vault status
Key                      Value
---                      -----
Seal Type                azurekeyvault
Recovery Seal Type       shamir
Initialized              true
Sealed                   false
Total Recovery Shares    3
Threshold                2
Version                  1.18.3
Build Date               2024-12-16T14:00:53Z
Storage Type             raft
Cluster Name             vault-cluster-6c316402
Cluster ID               9d7a0fd1-34b1-afb9-a239-66ccc4368d71
HA Enabled               true
HA Cluster               https://192.168.56.11:8201
HA Mode                  active
Active Since             2025-01-03T17:13:15.795033947Z
Raft Committed Index     55
Raft Applied Index       55

```

By default, the available authentication method is `token`, so we need to have the root token available to logon.

With the token, we can log as root through the UI:

![illustration1](/assets/vaultbasics/vaultbasics001.png)

![illustration2](/assets/vaultbasics/vaultbasics002.png)

Or with the vault cli:

```bash

yumemaru@df-2404lts:~$ vault login
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

```

### 1.2. Adding users to Vault

At this point, we only have the token authentication method and nothing else.

```bash

yumemaru@df-2404lts:~$ vault auth list
Path      Type     Accessor               Description                Version
----      ----     --------               -----------                -------
token/    token    auth_token_d7e2d457    token based credentials    n/a

```

If we want to start having users, we need to plan for an authentication method. There are a lot of interesting options but for now, we'll use the `userpass` authentication, to illustrate a simple authentication, and we'll focus on others functionnalities of Vault.
That can be done through the vault cli.

```bash

yumemaru@df-2404lts:~$ vault auth enable userpass 
Success! Enabled userpass auth method at: userpass/

``` 

Now we can see the `userpass` in the authentication method.

```bash

yumemaru@df-2404lts:~$ vault auth list
Path         Type        Accessor                  Description                Version
----         ----        --------                  -----------                -------
token/       token       auth_token_d7e2d457       token based credentials    n/a
userpass/    userpass    auth_userpass_671751c5    n/a                        n/a

```

Notice the `Path` output. Everything uses path in Vault so that's important. At this point, we don't have any user, so let's do this. We can either refer to the [`vault cli documentation`](https://developer.hashicorp.com/vault/docs/commands/auth/enable) or the [`api documentation`](https://developer.hashicorp.com/vault/api-docs/auth/userpass)

```bash

yumemaru@df-2404lts:~$ export userpassword="xxxxxxxxxxxxxxxxxxxx"
yumemaru@df-2404lts:~$ vault write auth/userpass/users/david
Must supply data or use -force
yumemaru@df-2404lts:~$ vault write auth/userpass/users/david password=$userpassword
Success! Data written to: auth/userpass/users/david
yumemaru@df-2404lts:~$ vault write auth/userpass/users/sheldon password=$userpassword
Success! Data written to: auth/userpass/users/sheldon

```

The user creation requires to specify a password in this case, as can be expected. 

Using the rest API, it looks like this:

```bash

yumemaru@df-2404lts:~$ curl --request POST --data '{"password": "xxxxxxxxxxxxxxxxxxxx"}' --header "X-Vault-Token:hvs.xxxxxxxxxxxxxxxxxxxxxxxx" http://192.168.56.11:8200/v1/auth/userpass/users/penny

```

We can check the new users in the `userpass` authentication, with the cli, the api or through the UI

```bash

yumemaru@df-2404lts:~$ curl --request LIST --header "X-Vault-Token:hvs.xxxxxxxxxxxxxxxxxxxxxxxx" http://192.168.56.11:8200/v1/auth/userpass/users/ | jq .
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   218  100   218    0     0  88726      0 --:--:-- --:--:-- --:--:--  106k
{
  "request_id": "c1052b0d-fb82-f09b-8b81-12775704f50b",
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

```

```bash

yumemaru@df-2404lts:~$ vault list auth/userpass/users
Keys
----
david
penny
sheldon

```

![illustration3](/assets/vaultbasics/vaultbasics003.png)

![illustration4](/assets/vaultbasics/vaultbasics004.png)

We are now able to logon as one of those user. Let's try it from the UI

![illustration5](/assets/vaultbasics/vaultbasics005.png)

![illustration6](/assets/vaultbasics/vaultbasics006.png)

![illustration7](/assets/vaultbasics/vaultbasics007.png)

And we don't see much, because, well, we did not specify any access.

Using the cli, by refering to the [doc](https://developer.hashicorp.com/vault/docs/commands/login), we get the same result, with an explixit deny message, which echoes what we know about Vault: by default, everything is denied.

```bash

df@df-2404lts:~$ vault login -method=userpass username=david
Password (will be hidden): 
Success! You are now authenticated. The token information displayed below
is already stored in the token helper. You do NOT need to run "vault login"
again. Future Vault requests will automatically use this token.

Key                    Value
---                    -----
token                  hvs.xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
token_accessor         yyyyyyyyyyyyyyyyyyyyyyyy
token_duration         768h
token_renewable        true
token_policies         ["default"]
identity_policies      []
policies               ["default"]
token_meta_username    david
df@df-2404lts:~$ vault list auth/userpass/users
Error listing auth/userpass/users: Error making API request.

URL: GET http://192.168.56.11:8200/v1/auth/userpass/users?list=true
Code: 403. Errors:

* 1 error occurred:
        * permission denied

```

The interesting information here is the policies, which specify `["default"]`. We'll have a look at the Vault policies in the coming sections. For now, let's start playing with secrets.

### 1.3. Storing secrets in Vault

In this section, we, at last, start storing secrets in Vault. Toget start, let's enable a secret engine first.

Again, there are many different [secret engines](https://developer.hashicorp.com/vault/docs/secrets) in Vault, but we'll start with the simple [kv engine](https://developer.hashicorp.com/vault/docs/secrets/kv).

There are 2 versions for the kv engine. v1 and v2. The version 2 allows additional features such as secret versioning, so we'll use this one.

To enable the kv v2 engine, we can use the vault cli.

```bash

df@df-2404lts:~$ vault secrets enable -version 2 -path /tbbtkv kv
Success! Enabled the kv secrets engine at: /tbbtkv/

```

And list the secrets engine:

```bash

df@df-2404lts:~$ vault secrets list
Path          Type         Accessor              Description
----          ----         --------              -----------
cubbyhole/    cubbyhole    cubbyhole_d908d63e    per-token private secret storage
identity/     identity     identity_08b8e281     identity store
sys/          system       system_b41078b4       system endpoints used for control, policy and debugging
tbbtkv/       kv           kv_18591b83           n/a

```

We can see the newly enabled engine. Notice the path `tbbtkv/`. Notice also the description that we forgot to specify. Let's fix this.

```bash

df@df-2404lts:~$ vault secrets tune -description "A kv engine for the tbbt guys" /tbbtkv
Success! Tuned the secrets engine at: tbbtkv/
df@df-2404lts:~$ vault secrets list
Path          Type         Accessor              Description
----          ----         --------              -----------
cubbyhole/    cubbyhole    cubbyhole_d908d63e    per-token private secret storage
identity/     identity     identity_08b8e281     identity store
sys/          system       system_b41078b4       system endpoints used for control, policy and debugging
tbbtkv/       kv           kv_18591b83           A kv engine for the tbbt guys

```

Now let's create some secrets!

```bash

df@df-2404lts:~$ vault kv put tbbtkv/david/secret1 target=thisisasecret
====== Secret Path ======
tbbtkv/data/david/secret1

======= Metadata =======
Key                Value
---                -----
created_time       2025-01-03T23:37:19.65449147Z
custom_metadata    <nil>
deletion_time      n/a
destroyed          false
version            1

```

We can also use the API.

```bash

df@df-2404lts:~$ curl --request POST --header $vaultheader --data '{"data":{"secret1":"iworkondarkmatternow"}}' http://192.168.56.11:8200/v1/tbbtkv/data/sheldon/secret1
{"request_id":"f7939771-d9b9-3545-3858-7a7b1bab5621","lease_id":"","renewable":false,"lease_duration":0,"data":{"created_time":"2025-01-03T23:52:59.692039011Z","custom_metadata":null,"deletion_time":"","destroyed":false,"version":1},"wrap_info":null,"warnings":null,"auth":null,"mount_type":"kv"}


```

or the UI

![illustration8](/assets/vaultbasics/vaultbasics008.png)

![illustration9](/assets/vaultbasics/vaultbasics009.png)

![illustration10](/assets/vaultbasics/vaultbasics010.png)

![illustration11](/assets/vaultbasics/vaultbasics011.png)

We can also read those secrets with the API

```bash

df@df-2404lts:~$ curl --header $vaultheader http://192.168.56.11:8200/v1/tbbtkv/data/david/secret1 | jq .
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   343  100   343    0     0   138k      0 --:--:-- --:--:-- --:--:--  167k
{
  "request_id": "75cd44b6-7ee7-9371-14a5-10229efc7c36",
  "lease_id": "",
  "renewable": false,
  "lease_duration": 0,
  "data": {
    "data": {
      "target": "thisisasecret"
    },
    "metadata": {
      "created_time": "2025-01-03T23:37:19.65449147Z",
      "custom_metadata": null,
      "deletion_time": "",
      "destroyed": false,
      "version": 1
    }
  },
  "wrap_info": null,
  "warnings": null,
  "auth": null,
  "mount_type": "kv"
}
df@df-2404lts:~$ curl --header $vaultheader http://192.168.56.11:8200/v1/tbbtkv/data/sheldon/secret1 | jq .
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   352  100   352    0     0   112k      0 --:--:-- --:--:-- --:--:--  171k
{
  "request_id": "3dc4e6ee-0d67-aab1-0e35-55540799579c",
  "lease_id": "",
  "renewable": false,
  "lease_duration": 0,
  "data": {
    "data": {
      "secret1": "iworkondarkmatternow"
    },
    "metadata": {
      "created_time": "2025-01-03T23:52:59.692039011Z",
      "custom_metadata": null,
      "deletion_time": "",
      "destroyed": false,
      "version": 1
    }
  },
  "wrap_info": null,
  "warnings": null,
  "auth": null,
  "mount_type": "kv"
}

```

Or with the cli.

```bash

df@df-2404lts:~$ vault kv get -mount="tbbtkv" "penny"
== Secret Path ==
tbbtkv/data/penny

======= Metadata =======
Key                Value
---                -----
created_time       2025-01-08T07:28:14.00630809Z
custom_metadata    <nil>
deletion_time      n/a
destroyed          false
version            1

===== Data =====
Key        Value
---        -----
secret1    imnotanactressiworkforbigpharma

```

Let's try with a secret from Sheldon

```bash

df@df-2404lts:~$ vault kv get -mount="tbbtkv" "sheldon"
No value found at tbbtkv/data/sheldon
df@df-2404lts:~$ vault kv get -mount="tbbtkv" "sheldon/"
No value found at tbbtkv/data/sheldon

```

Wait, what?
Let's dig a little bit.

### 1.4. Understanding the path

So we know that we can get the secret from the API.

```bash

df@df-2404lts:~$ curl --header $vaultheader http://192.168.56.11:8200/v1/tbbtkv/data/sheldon/secret1 | jq .
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   352  100   352    0     0   112k      0 --:--:-- --:--:-- --:--:--  171k
{
  "request_id": "3dc4e6ee-0d67-aab1-0e35-55540799579c",
  "lease_id": "",
  "renewable": false,
  "lease_duration": 0,
  "data": {
    "data": {
      "secret1": "iworkondarkmatternow"
    },
    "metadata": {
      "created_time": "2025-01-03T23:52:59.692039011Z",
      "custom_metadata": null,
      "deletion_time": "",
      "destroyed": false,
      "version": 1
    }
  },
  "wrap_info": null,
  "warnings": null,
  "auth": null,
  "mount_type": "kv"
}

```

We can note that the path contains the secret name `http://192.168.56.11:8200/v1/tbbtkv/data/sheldon/secret1`

So from the cli we should add the full path with the secret name.

```bash

df@df-2404lts:~$ vault kv get -mount="tbbtkv" "sheldon/secret1"
======= Secret Path =======
tbbtkv/data/sheldon/secret1

======= Metadata =======
Key                Value
---                -----
created_time       2025-01-03T23:52:59.692039011Z
custom_metadata    <nil>
deletion_time      n/a
destroyed          false
version            1

===== Data =====
Key        Value
---        -----
secret1    iworkondarkmatternow


```

We can also do this with the full path, instead of specifying the `-mount` argument.


```bash

df@df-2404lts:~$ vault kv get tbbtkv/sheldon/secret1
======= Secret Path =======
tbbtkv/data/sheldon/secret1

======= Metadata =======
Key                Value
---                -----
created_time       2025-01-03T23:52:59.692039011Z
custom_metadata    <nil>
deletion_time      n/a
destroyed          false
version            1

===== Data =====
Key        Value
---        -----
secret1    iworkondarkmatternow


```

Let's create a new secret for Sheldon from the API:

```bash

df@df-2404lts:~$ curl --request POST --header $vaultheader --data '{"data": {"secret2": "istillfancystringstheorythough"}}' http://192.168.56.11:8200/v1/tbbtkv/data/sheldon/secret2
{"request_id":"15b255b5-bf40-5c51-8ad9-3433033d2e31","lease_id":"","renewable":false,"lease_duration":0,"data":{"created_time":"2025-01-08T10:40:23.166706673Z","custom_metadata":null,"deletion_time":"","destroyed":false,"version":1},"wrap_info":null,"warnings":null,"auth":null,"mount_type":"kv"}
df@df-2404lts:~$ curl --header $vaultheader http://192.168.56.11:8200/v1/tbbtkv/data/sheldon/secret2 | jq .
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   362  100   362    0     0  44968      0 --:--:-- --:--:-- --:--:-- 45250
{
  "request_id": "afff8fc6-77fa-fde5-de2b-d7488642133d",
  "lease_id": "",
  "renewable": false,
  "lease_duration": 0,
  "data": {
    "data": {
      "secret2": "istillfancystringstheorythough"
    },
    "metadata": {
      "created_time": "2025-01-08T10:40:23.166706673Z",
      "custom_metadata": null,
      "deletion_time": "",
      "destroyed": false,
      "version": 1
    }
  },
  "wrap_info": null,
  "warnings": null,
  "auth": null,
  "mount_type": "kv"
}


```

And from the cli

```bash


df@df-2404lts:~$ vault kv put tbbtkv/sheldon/secret3 secret3=bazinga
======= Secret Path =======
tbbtkv/data/sheldon/secret3

======= Metadata =======
Key                Value
---                -----
created_time       2025-01-08T10:45:56.564217129Z
custom_metadata    <nil>
deletion_time      n/a
destroyed          false
version            1

```

Notice the end of the command where we specified `secret3=bazinga` as opposite to the previous occurence of secret creation with the cli here we  specified `target=<secret_value>`.

Checking on the portal we can see all 3 secrets under the tbbtkv/sheldon path

![illustration12](/assets/vaultbasics/vaultbasics012.png)

We can see that the key identifying the secret is `secret3`. 

![illustration13](/assets/vaultbasics/vaultbasics013.png)

While it is target in our first secret.

```bash

df@df-2404lts:~$ vault kv get tbbtkv/david/secret1
====== Secret Path ======
tbbtkv/data/david/secret1

======= Metadata =======
Key                Value
---                -----
created_time       2025-01-03T23:37:19.65449147Z
custom_metadata    <nil>
deletion_time      n/a
destroyed          false
version            1

===== Data =====
Key       Value
---       -----
target    thisisasecret


```

![illustration14](/assets/vaultbasics/vaultbasics014.png)

Adding a secret for Penny that follows the same principles as the secrets created from the cli ou the API, we have to write the path as below.

![illustration15](/assets/vaultbasics/vaultbasics015.png)

For those who wonders, It is not possible to create a path that ends with a `/`

![illustration16](/assets/vaultbasics/vaultbasics016.png)

We can clearly see that we have a secret which path is `tbbtkv/penny`, and others secrets which path are `tbbtkv/penny/<secret_name>`

Which means that we need to consider the path. It should contains the name of the secret. In our case, when we created our `penny` from the UI, we basically named the secret `penny`. The key is really not important, It can be anything, as long as the path is clearly identified.

![illustration17](/assets/vaultbasics/vaultbasics017.png)

![illustration18](/assets/vaultbasics/vaultbasics018.png)

![illustration19](/assets/vaultbasics/vaultbasics019.png)

![illustration20](/assets/vaultbasics/vaultbasics020.png)

Moving on, let's have a look at how we can grant access to this Vault server.

### 1.5. Managing access to Vault

Now that we have a kv engine available, we want our user to start using it. In the previous section, we did everything as the root user, which is not really the aim right ^^.

So we need to define access, which is done with the policies.
Using the vault cli, we can see that there are 2 built-in policies

```bash

yumemaru@df-2404lts:~$ vault policy list
default
root

```

## source


```bash




yumemaru@df-2404lts:~$ vault secrets list
Path          Type         Accessor              Description
----          ----         --------              -----------
cubbyhole/    cubbyhole    cubbyhole_08e648c5    per-token private secret storage
identity/     identity     identity_0b7ce7f8     identity store
sys/          system       system_c760460c       system endpoints used for control, policy and debugging
yumemaru@df-2404lts:~$ vault auth list
Path      Type     Accessor               Description                Version
----      ----     --------               -----------                -------
token/    token    auth_token_447c9a81    token based credentials    n/a

```

```bash


yumemaru@df-2404lts:~$ vault read auth/userpass/users/david
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



```

```bash



yumemaru@df-2404lts:~$ curl --header "X-Vault-Token:hvs.xxxxxxxxxxxxxxxxxxxxxxxx" http://192.168.56.11:8200/v1/sys/auth/userpass/ | jq .
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

yumemaru@df-2404lts:~$ curl --request LIST --header "X-Vault-Token:hvs.xxxxxxxxxxxxxxxxxxxxxxxx" http://192.168.56.11:8200/v1/auth/userpass/users | jq .
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

yumemaru@df-2404lts:~$ curl --header "X-Vault-Token:hvs.xxxxxxxxxxxxxxxxxxxxxxxx" http://192.168.56.11:8200/v1/auth/userpass/users/david | jq .
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


yumemaru@df-2404lts:~$ vault secrets enable kv
Success! Enabled the kv secrets engine at: kv/
yumemaru@df-2404lts:~$ vault secrets list
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