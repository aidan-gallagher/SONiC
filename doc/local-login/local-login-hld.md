# Local Login HLD

## Table of contents

# Revision

| Rev | Date       | Author          | Description     |
|:---:|:----------:|:---------------:|:--------------- |
| 0.1 | 13/09/2024 | Aidan Gallagher | Initial version |

# About this manual

This document provides a high-level information for the SONiC local user feature and CLI. It describes high-level behavior, internal definition, design of the commands, syntax, and output definition.

# Scope

The scope of this document is to cover definition, design and implementation of SONiC logal login feature.

<!-- omit in toc -->

## Abbreviations

| Term    | Meaning                                   |
|:------- |:----------------------------------------- |
| SONiC   | Software for Open Networking in the Cloud |
| DB      | Database                                  |
| CLI     | Сommand-line Interface                    |
| YANG    | Yet Another Next Generation               |
| SHA-512 | A cryptographic hashing algorithm         |

# 1 Introduction

## 1.1 Feature overview

Currently local user management for SONiC switches is controlled by linux commands (e.g. `adduser`, `passwd`, etc). These command update the linux file (e.g. `/etc/shadow`, `/etc/passwd`, etc) however they do not store any of this information in `/etc/sonic/config_db.json`.

When a new SONiC image is installed, any Linux configuration which is not also stored in `/etc/sonic/config_db.json` will be removed.
This means after an operator upgrades a SONiC device the login credentials will be reset to the default username and password. The operater will have to reconfigure the login credentials; this additional step is often done manually which increases the chances of misconfiguration and wastes time. 

This feature aims to solve the problem by storing a list of users and encrypted passwords into the redis database. 

## 1.2 Requirements

This feature will support saving a list of users and their encrypted passwords in the configuration file.

This feature will introduce CLI commands to:

* Add a user
* Remove a user
* Change the password for a user

The CLI commands will store the state in the redis database. The operator can then run

`config save` to write the state to  `/etc/sonic/config_db.json`.

When a SONiC switch is upgraded to a new image the local login credentials stored in the configuration file will be applied.

If no user information is stored in the database then no action will be taken. This ensures this features maintains backwards compatibility. 

# 2 Design

## 2.1 Overview

When an operator uses the CLI to add a user they will be prompted for a password with a masked input field.

The users password will be encrypted using SHA-512 before it is pushed to the redis database.

## 2.2 Sequence Diagrams

<!-- omit in toc -->

### 2.2.1 Inital flow

![Local login init sequence diagram](images/local-login-init-sequence.drawio.svg "Figure 1: Local login init sequence diagram")

### 2.2.2 Add new user / change password

![Local login config sequence diagram](images/local-login-config-sequence.drawio.svg "Figure 2: Local login config sequence diagram")

<!-- omit in toc -->

### 2.5.2 Delete user

![](images/local-login-delete-sequence.drawio.svg)

## 2.3 Linux commands

When the local-login feature is not used the current behaviour of Linux commands such as  `adduser`, `psswd`, etc will not be changed. Changes made using these commands will persist until a new image wipes them.

When the local-login feature is used the configuration database is the single source of truth. This means any changes made using `adduser`, `psswd`, `etc` will be overwritten by the local-login service. The changes will be overwritten on system boot and whenever the local login database changes.

> I could prevent adduser, psswd, from being used & print a helpful warning when this feature is in use. Is this worth the effort? I don't think it is a pattern implemented with SONiC.

## 2.4 Integration With Password Hardening

The [password hardening feature](../passw_hardening/hld_password_hardening.md) imposes restrictions on what passwords can be used (e.g. min character length, numbers must be used, etc).
This feature should ensure it uses the Linux PAM so it doesn't bypass any constraints impossed by password hardening. 

> **Note**: It seems the password hardening constraints are  bypassed when sudo is used. This feature requires sudo permissions because it adds users and set's other users passwords. 
> Need to work out a way to validate passwords against PAM policy before encrypting it. Maybe best doing that all in Python.

>  **Note**: The password hardening feature and this feature are both features which modify the local login credentials. The hardening CLI and yang could be moved under the new parent 'local-login', however this would break backwards compatibility therefore I expect there would be a big push back from the upstream maintainers.

## 2.5 Security Concerns

On Linux systems encrypted passwords for every user are stored in `/etc/shadow`. This file is only readable by sudo users. This feature introduces storing the encrypted passwords in redisDB and `/etc/sonic/config_db.json`.  
RedisDB is only viewable to users in the redis group (1001) - therefore it is secure to store the passwords there.  
`config_db.json` is readable by non-sudo users therefore it reduces the security to store the passwords there. Below are the existing file permissions.

```
admin@sonic:~$ ls -lah /etc/shadow
-rw------- 1 root shadow 731 Sep 16 14:25 /etc/shadow
admin@sonic:~$ ls -lah /etc/sonic/config_db.json 
-rw-r--r-- 1 root root 17K Sep 23 09:07 /etc/sonic/config_db.json
```

This feature will change the config_db.json file permission to

```
-rw--r---- 1 root redis 731 Sep 16 14:25 /etc/sonic/config_db.json
```

> **_NOTE:_** This is probably have a lot of knock on effects. Need to search through all of SONiC and find things that read the file and ensure they are either sudo or in the redis group.

> **_NOTE:_** Linux systems are designed as multi-user systems. Having `config_db.json` readable to non-sudo users is only a concern is we have non-trusted people on the switch (is that unlikely?). Nevertheless I expect it will be hard to upstream a feature if it is deemed to weaken the security.

## 2.6 CLI

<!-- omit in toc -->

### 2.6.1 Command structure

#### 2.6.1 Config command group

```
$ config local-login setuser <USERNAME>
New password:
Retype new password:
```

```
$ config local-login setuser <USERNAME> --encrypted-password '$y$j9T$b11VwUbTIB3mz4vlv/r2j/$BHx0RS4y56H.2ybeiCZTtuwWBsprCD6FOxsc5AoTO3/'
```

```
$ config local-login deluser <USERNAME>
```

```
$ show local-login users
bob
alice
```



>  Should the local-login CLI and yang live under AAA? Should it be something like this?

```
config aaa authentication setuser <USERNAME>

config aaa authentication deluser <USERNAME>
```

## 2.7 YANG model

New YANG model `sonic-local-login.yang` will be added to provide support for configuring local login credentials.

> NOTE: Should this feature be integrated with the AAA module instead of a stand alone module?

```
module local-login {
    yang-version 1.1;
    namespace "http://github.com/sonic-net/local-login";
    prefix local_login;

    description "LOCAL_LOGIN YANG Module for SONiC-based OS";
    revision 2024-09-13 {
        description "First Revision";
    }

    container sonic-local-login {

      container LOCAL_LOGIN {
        list user {
            leaf username {
              type string
            }
            leaf password {
              type string
            }
        }
    }
```

# 3 Test plan

## 3.1 Systems Testing

* Add new user
  
  1. Login as default admin user
  2. Create new user `testuser`
  3. Set password to `testpassword`
  4. Logout
  5. Login to `testuser` using `testpassword`

* Upgrade Image
  
  1. Change `admin` password to `testpassword`
  2. Upgrade image to fresh install
  3. Login to `admin` using `testpasword`

# 4 Out of Scope

In Linux there is more account management settings than just username and password. This feature is limited to just username and password because that is likely the only setting operators are interested in. The following decisions have been made:

- All configured users will have sudo permissions.
- All configured users will have the default home directory `/home/<USERNAME>`.
- All configured passwords will use SHA-512 encryption.

# 5 Disregarded Ideas

## Modify useradd

A wrapper command could be created around useradd and userdel with the same name, where the wrapper handles updating the database before invoking the underlying command.  
The benefit of this approach is the user doesn't have to learn a new command and the existing Linux approach just works.  
This approach doesn't align with the existing SONiC design of adding adding new commands via sonic-utilities.
