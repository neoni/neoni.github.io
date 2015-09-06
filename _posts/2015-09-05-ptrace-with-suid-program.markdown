---
layout: post
title: "Suid program can't work with ptrace"
comments: true
tags: [debug, ptrace]
description: hi wkwedlsdjk
date: 2015-09-05T21:00:36+08:00
---

If we resume the process with PTRACE_CONT without detaching works and parent process doesn't terminated, it will work like in the gdb, then suid will not work. So we have to detach the work at first.

PTRACE_DETACH is a restarting operation; therefore it requires the tracee to be in ptrace-stop. If the tracee is in signal-delivery- stop, a signal can be injected. Otherwise, the sig parameter may be silently ignored.


```C
ptrace(PTRACE_CONT, child, NULL, 0);
kill(child, SIGSTOP);
waitpid(child, NULL, 0);
int erro = ptrace(PT_DETACH, child, NULL, NULL);
printf("ret: %d\n", erro);
```


### Ref

[PTRACE_DETACH fails after PTRACE_CONT with errno=ESRCH](http://stackoverflow.com/questions/20510300/ptrace-detach-fails-after-ptrace-cont-with-errno-esrch)