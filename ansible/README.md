# Mikrotik configuration

First, you need to install network module:

``` bash
ansible-galaxy collection install community.network
ansible-galaxy collection install community.routeros
```

## What works

- Configuring users with file `environment/credentials.yml`.

## Variables

Change files content:

- `environment/credentials.yml` - user accounts, that must be in devices;
- `environment/inventory` - configured devices;
- `vault.key` - key for securing secrets.

RSA public keys should be located in `environment/rsa` and named as user name.

> In `ansible_user` string add after username `+2000wt` to set termonal width: (ex.: `admin+2000wt`).

## How to run

``` bash
ansible-playbook playbooks/run.yml
```

## How to secure secrets

File:

``` bash
ansible-vault encrypt environment/inventory
```

String:

``` bash
$ ansible-vault encrypt_string 'example'
!vault |
          $ANSIBLE_VAULT;1.1;AES256
          39363166313965346264616439333566346465353738336139353539653135366565663630323638
          3165393463383236353433373334626238343938616439380a643637376633633135363631303334
          34316239326163653539346439363131343032653463393037383734653261306564633835343264
          3431323433323534650a303136303132626532363162356631303034616663663834633837613138
          3435
Encryption successful
```
