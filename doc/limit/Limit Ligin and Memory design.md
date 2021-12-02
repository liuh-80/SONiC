# SONiC Limit user session and memory

# Table of Contents
- [Table of Contents](#table-of-contents)
- [About this Manual](#about-this-manual)
- [1 Functional Requirements](#1-functional-requirement)
  * [1.1 Limit the number of logins per user/group/system](#11-limit-the-number-of-logins-per-user/group/system)
  * [1.2 Limit memory usage per user/group/system](#12-limit-memory-usage-per-user/group/system)
- [2 Configuration and Management Requirements](#2-configuration-and-management-requirements)
  * [2.1 SONiC CLI](#21-sonic-cli)
  * [2.2 Config DB](#22-config-db)
- [3 Design](#design)
  * [3.1 Login Limit Implementation](#31-login-limit-implementation)
  * [3.2 Memory Limit Implementation](#32-memory-limit-implementation)
- [4 Error handling](#error-handling)
- [5 Serviceability and Debug](#serviceability-and-debug)
- [6 Unit Test](#unit-test)
- [8 References](#references)


# About this Manual
This document provides a detailed description on the new features for:
 - Limit the number of logins per user/group/system.
 - Limit memory usage per user/group/system.

# 1 Functional Requirement
## 1.1 Limit the number of logins per user/group/system
 - Can set max login session count per user/group/system.
 - When exceed maximum login count, login failed with error message.

## 1.2 Limit memory usage per user/group/system
 - Can set max memory usage per user/group/system.
 - When exceed maximum memory usage, the OOM process will be paused or terminated.

# 2 Configuration and Management Requirements
## 2.1 SONiC CLI
 - Manage limit
```
    config limit {login | memory} { add | del } {user | group | global} <number>
```

 - Show limit
```
    show limit {login | memory}
```

## 2.2 Config DB
 - Login limit and memory limit are fully configurable by config DB.

# 3 Design
 - Design diagram:

```mermaid
graph TD;
%% SONiC CLI update config DB
CLI[SONiC CLI] -- update limit setting --> CONFDB(Config DB);

%% HostCfgd subscribe config DB change
CONFDB --> HOSTCFGD[Hostcfgd];

%% HostCfgd Update config files
HOSTCFGD -- update limits.conf --> PAMCFG[limits.conf];
HOSTCFGD -- update cgconfig.conf --> CGRCFG[cgconfig.conf];
HOSTCFGD -- update cgrules.conf --> CGCFG[cgrules.conf];

%% pam_limits.so will handle login limit
PAMCFG -.-> LIMITLIB[pam_limits.so];
LIMITLIB -- login --- USERSESSION(user session);

%% cgrulesengd daemon will migrate procress to cgroups by uid and gid
CGRCFG -.-> CGRCFGD[cgrulesengd];
CGRCFGD -- migrate process --- APP;

%% cgroup check memory limit and trigger OOM killer
CGCFG -.-> CGROUP[cgroup];
CGROUP -- OOM killer --- APP(user process);
```

## 3.1 Login limit Implementation
 - Enable PAM plugin pam_limits.so to support login limit.

## 3.2 Memory limit Implementation
 - Use cgroup-tools to support memory limit.
 - cgrulesengd will run in background to migrate process to cgroup.

# 4 Error handling
 - pam_limits.so will return errors as per [PAM](#pam) respectively.

# 5 Serviceability and Debug
 - pam_limits.so can be debugged by enabling the debug flag in PAM config file.
 - cgroup cgrulesengd can be debugged by enabling the CGROUP_LOGLEVEL environment variable.

# 6 Unit Test

- TODO: add end to end test case.

# 7 References
## pam_limits.so
https://man7.org/linux/man-pages/man8/pam_limits.8.html
## cgroup
https://man7.org/linux/man-pages/man7/cgroups.7.html
## cgroup-tools
https://manpages.debian.org/testing/cgroup-tools/cgrules.conf.5.en.html
https://manpages.debian.org/bullseye/cgroup-tools/cgconfig.conf.5.en.html
https://manpages.debian.org/experimental/cgroup-tools/cgrulesengd.8.en.html