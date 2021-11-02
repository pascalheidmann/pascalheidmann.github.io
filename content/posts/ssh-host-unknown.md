---
title: "SSH Host Unknown"
date: 2021-11-02T22:46:50+01:00
draft: false
tags: ["ssh"]
---

## Problem

Your ssh config `Include`s another config file with hosts. When trying to connect to a host in that referenced config the host is unknown.

### Example
```ini
Host example.tld
    Hostname example.tld
    User myuser
    
Include ~/some-other-ssh-config
```

## Solution
Put include statements first as those will be interpreted as sub-configs to the `Host` entry.

### Modified example
```ini
Include ~/some-other-ssh-config

Host example.tld
    Hostname example.tld
    User myuser
```
