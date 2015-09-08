---
layout: post
title: "Debugging Forks"
comments: true
tags: [debug, fork]
date: 2015-09-08T10:47:56+08:00
---

### debug the parent process and the child process

By default, when a program forks, gdb will continue to debug the parent process and the child process will run unimpeded.

If you want to follow the child process instead of the parent process, use the command set `follow-fork-mode`.

```python
set follow-fork-mode mode
    Set the debugger response to a program call of fork or vfork. A call to fork or vfork creates a new process.  
    The mode argument can be:
    parent
        The original process is debugged after a fork. The child process runs unimpeded. This is the default.
    child
        The new process is debugged after a fork. The parent process runs unimpeded. 

show follow-fork-mode
    Display the current debugger response to a fork or vfork call. 
```

On Linux, if you want to debug both the parent and child processes, use the command set `detach-on-fork`.

```
set detach-on-fork mode
    Tells gdb whether to detach one of the processes after a fork, or retain debugger control over them both.
    on
        The child process (or parent process, depending on the value of follow-fork-mode) will be detached and allowed to run independently. This is the default.
    off
        Both processes will be held under the control of gdb. One process (child or parent, depending on the value of follow-fork-mode) is debugged as usual, while the other is held suspended. 

show detach-on-fork
    Show whether detach-on-fork mode is on/off. 
```

If you choose to set `detach-on-fork` mode off, then gdb will retain control of all forked processes (including nested forks). 

---

### switch between process

You can list the forked processes under the control of gdb by using the `info inferiors` command, and switch from one fork to another by using the inferior command.

To quit debugging one of the forked processes, you can either detach from it by using the `detach inferiors` command (allowing it to run independently), or kill it using the `kill inferiors` command. 

see [Debugging Multiple Inferiors and Programs](https://sourceware.org/gdb/onlinedocs/gdb/Inferiors-and-Programs.html#Inferiors-and-Programs)

---

### debug exec mode 

If you ask to debug a child process and a vfork is followed by an exec, gdb executes the new target up to the first breakpoint in the new target. If you have a breakpoint set on main in your original program, the breakpoint will also be set on the child process's main.

On some systems, when a child process is spawned by vfork, you cannot debug the child or parent until an exec call completes.

If you issue a run command to gdb after an exec call executes, the new target restarts. To restart the parent process, use the `file` command with the parent executable name as its argument. 

By default, after an exec call executes, gdb discards the symbols of the previous executable image. You can change this behaviour with the set `follow-exec-mode` command.

```python
set follow-exec-mode mode
    Set debugger response to a program call of exec. An exec call replaces the program image of a process.
    follow-exec-mode can be:
    new
        gdb creates a new inferior and rebinds the process to this new inferior. The program the process was running before the exec call can be restarted afterwards by restarting the original inferior.
        For example:
                       (gdb) info inferiors
                       (gdb) info inferior
                         Id   Description   Executable
                       * 1    <null>        prog1
                       (gdb) run
                       process 12020 is executing new program: prog2
                       Program exited normally.
                       (gdb) info inferiors
                         Id   Description   Executable
                       * 2    <null>        prog2
                         1    <null>        prog1
    same
        gdb keeps the process bound to the same inferior. The new executable image replaces the previous executable loaded in the inferior. Restarting the inferior after the exec call, with e.g., the run command, restarts the executable the process was running after the exec call. This is the default mode.
        For example:
                       (gdb) info inferiors
                         Id   Description   Executable
                       * 1    <null>        prog1
                       (gdb) run
                       process 12020 is executing new program: prog2
                       Program exited normally.
                       (gdb) info inferiors
                         Id   Description   Executable
                       * 1    <null>        prog2
```

You can use the catch command to make gdb stop whenever a fork, vfork, or exec call is made. See Setting Catchpoints. 

### Ref

[Debugging Forks](https://sourceware.org/gdb/onlinedocs/gdb/Forks.html)

[Debugging Multiple Inferiors and Programs](https://sourceware.org/gdb/onlinedocs/gdb/Inferiors-and-Programs.html#Inferiors-and-Programs)