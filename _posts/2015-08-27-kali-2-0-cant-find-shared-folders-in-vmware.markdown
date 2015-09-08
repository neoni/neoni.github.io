---
layout: post
title: "Kali 2.0 Can't Find Shared Folders in Vmware"
comments: true
tags: [kali, vmware-tools]
date: 2015-08-27T13:31:30+08:00

---

### Reason: VMware Tools broken with gcc 5.1.0+ 

* during vmware-tools-install.pl, it won't compile vmhgfs with errors of functions not found.  needs to be patched (vmware issue, ignore)

* during vmware-tools-install.pl again with patched script, it compiles just fine.

* but during the Kernel Modules enabling portion, the system hangs / freezes / halts.  Hard reset is only resolve.

* on system boot, system hangs / freezes / halts on vmhgfs initialization.  

### solution

* first use gcc-4.9 instead of gcc-5.+

```sh
ln -sf /usr/bin/gcc-4.9 /usr/bin/gcc
```

* then use [vmware-tools patch](https://github.com/rasa/vmware-tools-patches/)


### Ref

[https://bbs.archlinux.org/viewtopic.php?id=197684](https://bbs.archlinux.org/viewtopic.php?id=197684)


