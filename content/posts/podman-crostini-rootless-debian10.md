---
title: "<ChromeOS: nested, unprivileged, podman>"
date: 2021-03-24T15:17:50+01:00
draft: false
tags: [
 "chromeos",
 "crostini",
 "rootless",
 "podman"
]
---
lxc config set penguin security.nesting true
<!--more-->
using the [install instructions](https://podman.io/getting-started/installation )you will probably run into this error

```bash
podman run --rm 2f4357dd9647 /bin/echo "fubar"
Error: mount `proc` to `/proc`: Operation not permitted: OCI permission denied
```

Why is that? Because there are lxcfs mount points below /proc and kernel does not allow it.

```bash
findmnt -R /proc --output "TARGET,SOURCE,FSTYPE"

TARGET                     SOURCE                    FSTYPE
/proc                      proc                      proc
├─/proc/cpuinfo            lxcfs[/proc/cpuinfo]      fuse.lxcfs
├─/proc/diskstats          lxcfs[/proc/diskstats]    fuse.lxcfs
├─/proc/meminfo            lxcfs[/proc/meminfo]      fuse.lxcfs
├─/proc/stat               lxcfs[/proc/stat]         fuse.lxcfs
├─/proc/uptime             lxcfs[/proc/uptime]       fuse.lxcfs
└─/proc/sys/fs/binfmt_misc proc[/sys/fs/binfmt_misc] proc
```

**Two ways to solve this**

1. you can change the penguin container  configuration so that you won't get a permission denied anymore

to do that you open crosh (ctlr+alt+t) and

```bash
# enter the termina vm
vsh termina
# allow nesting containers
lxc config set penguin security.nesting true
# restart penguin lxc container
lxc restart penguin
```

as proposed here: https://bugs.chromium.org/p/chromium/issues/detail?id=1087937

2. umount

```bash
sudo umount /proc/{cpuinfo,diskstats,meminfo,stat,uptime}
```
 
and in order to make that change in the user namespace aswell you can tell podman to redo the namespace this way

```bash
podman system migrate
```

to test that you do

```bash
❯ podman unshare # join user namespace
user in ~
❯ findmnt -R /proc # check from within
TARGET SOURCE FSTYPE OPTIONS
/proc  proc   proc   rw,nosuid,nodev,noexec,relatime
❯ exit # get out again
❯ podman run --rm 2f4357dd9647 /bin/echo "fubar"
fubar
```
