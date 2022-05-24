---
title: "Initial Work on Heavybell"
date: 2022-05-18T08:20:25+12:00
draft: true
author: "Leon Greeff"
categories:
    - Go
    - Plan9
---

I've grown weary of using Linux Distros. I've hopped around a bit when I stared but I've now setteled on Fedora.
Using Linux is the same way I feel about using Windows. I don't like to use it but I have to because there is nothing better.

Years ago I wanted to create my own Linux Distro. It had a Makefile based package management system, similar to FreeBSD ports.
It was not as much fun as I thought and it would be. It revealed to me just how ugly Distros are under the hood.

I also stared playing around with Plan9. It had some great ideas and some of it made it's way into Linux.
It was a research OS built around namespaces and a file system protocol called 9P.

The simplicity of the system was beautiful but I don't think I can use it for my daily tasks.

I want build something better than what is available. I want to finish what Plan9 started.

- Create a general computing environment around namespaces and 9P
- Add support for 3D graphics
- Use containers for external software, e.g Chrome

Whether this is feasible I do not know. I don't expect this to be easy.

## Where to start

Although Linux does have namespaces and union mounts it is not the same as Plan9.
I have a feeling that creating a system with these versions will be a mess.
(Maybe that is something I should try before dismissing it)

Let's start with writing something like `emu` from Inferno. This will allow us to run our environment anywhere. From the `emu` man page.
> The emulator runs as an application under the machine's native operating system,
> and provides system services and a Dis virtual machine for Inferno applications.

I'll write this in Go and remove the Dis virtual machine.

`emu` starts by calling some OS specific initialization code via `libinit()`.
This creates some signals and sets up a Dis module `/dis/emuinit.dis` as it's main process then calls `emuinit()`.

`emuinit()` then binds some devices, sets up the root namespace, and finaly starts the `disinit` module with `kproc()`.

`kproc()` creates a new process with `pthread_create()` using `disinit` as the function and then loops.
```
        kproc("main", disinit, imod, KPDUPFDG|KPDUPPG|KPDUPENVG);

        for(;;)
                ospause();
```

Let's start by looking at how Go creates threads.

## Goroutines vs pthreads

A `goroutine` is a lightweight thread managed by the Go runtime. The main features are.
- Built-in primitives to communicate safely between routines (channels)
- Avoid having to resort to mutex locking when sharing data structures
- Multiplexed onto a small number of OS threads, rather than a 1:1 mapping.
- Runtime uses resizable, bounded stacks.

From the `pthreads` man page
> Threads share the same global memory (data and heap segments), but each thread has its own stack (automatic variables).

Why do we need to create a thread in the first place? Why not just call `fork()`?
A thread has it's own stack and registers and shares data with the parent process.
Forking makes a copy of the parent process and communication between the processes has to be done via some channel that has to be built.

Now I need to figure out if this will work for what we want to do, or if we need a Go wrapper for pthreads
