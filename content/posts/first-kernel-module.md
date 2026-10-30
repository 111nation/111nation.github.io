+++
date = '2026-10-30T21:06:47+02:00'
draft = false
title = 'FreeBSD - Writing my first kernel module!'
categories = ["Unix System Programming"]
tags = ["c", "systems programming", "unix", "bsd"]
cover = "/covers/first-kernel-module.png"
feature=true
+++
<br />

Today (30 October 2026) marks the day I created my *first ever kernel module!* If you are wondering what a kernel module is, a basic to understand it is that it is a piece of executable code that an operating system kernel—like FreeBSD and Linux—can load to use to give your computer additional functionality such as loading new drivers for that new gaming mouse!

## Making of the Kernel Module

I always wanted to have my break through in open source and operating system development. To quickly get hands-on experience, I used AI as a tool to help guide me through and tutor me through the process of creating this simple module.

The goal for today was just to get a simple `.ko` loadable kernel module that is able to indicate it has been loaded, store driver information as well as indicate when it has been unloaded.

{{< alert type="info" title="Quick Tip" >}}
All source code has been provided under [Source Code](#source-code)
{{</ alert >}}

### Development environment 

In order to successfully achieve writing a kernel module, I needed to pick an operating system kernel to target. I went with FreeBSD because it's an operating system I hold dearly in my heart. I spun up a FreeBSD 15.1 virtual machine using [quickemu](https://github.com/quickemu-project/quickemu), which automates installing and configuring the qemu hyper-visor using a single command (very underrated).

The prerequisites for kernel development are obviously the base development tools such as the FreeBSD `clang` compiler and FreeBSD `make` tool.

Notice in the [Makefile](#makefile) does not seem to appear basic gnu/unix syntax. The FreeBSD organization has their own makefile syntax that is able to bind your source code to the operating system's source as simple as possible. 

The line, `.include <bsd.kmod.mk>`, under the Makefile is a rule to bind the source code with the operating system's build rules to produce a `.ko` loadable kernel module. In order for this operation to successfully compile my kernel module, I needed the full FreeBSD source code under `/usr/src`. I resolved this by cloning the source code from `https://git.freebsd.org/src.git` inside that directory.

### What does it Do?

To keep this article short a relatively high level overview will be provided.

This kernel module simply adds event handlers to detect and display a message when it is loaded using:

```sh
~ $ sudo kldload ./module_sysctl.ko
Module loaded
```

and unloaded using:

```sh
~ $ sudo kldunload module_sysctl
Module unloaded
```

It is also able to display a message when it is queried via:
```sh
~ $ sysctl kern.module_message
Hey im Chloe! This is my kernel module!
```

This code taps into FreeBSD's `sysctl` kernel subsystem. `sysctl` manages stuff like kernel parameters. We use `SYSCTL_STRING` to register our message into this subsystem. This allows the `sysctl` CLI tool to have access this string. We then use a large struct to define events which allow our module to do something when we `MOD_LOAD` and `MOD_UNLOAD`.

Towards the end the `DECLARE_MODULE` macro is used to register `module_sysctl` as a valid kernel module.

## Conclusion

I wish I could talk more about kernel modules, but I am still learning and I do not want to provide you an AI answer! I plan to continue writing a series on kernel development!

## Source Code

### Main Module
{{< alert type="note" title="Disclaimer" >}}
The code was originally generated using Gemini and additional modifications have been done on top. My goal was to actually understand what was given to me, and I used it as a tool to introduce me to kernel development. 
{{</ alert >}}

This is the source code for `module_sysctl.c`

```c
#include <sys/param.h>
#include <sys/module.h>
#include <sys/kernel.h>
#include <sys/systm.h>
#include <sys/sysctl.h>

static char msg[128] = "Hey im Chloe! This is my kernel module!";

SYSCTL_STRING(_kern, OID_AUTO, module_message, CTLFLAG_RW,
		msg, sizeof(msg), "This is a kernel string from module_sysctl!");

static int
module_sysctl_modevent(module_t mod, int type, void *data)
{
    switch (type) {
    case MOD_LOAD:
        uprintf("Module loaded\n");
        break;
    case MOD_UNLOAD:
        uprintf("Module unloaded\n");
        break;
    default:
        return (EOPNOTSUPP);
    }
    return (0);
}

static moduledata_t module_sysctl_conf = {
    "module_sysctl",
	module_sysctl_modevent,
    NULL
};

DECLARE_MODULE(module_sysctl, module_sysctl_conf, SI_SUB_DRIVERS, SI_ORDER_MIDDLE);
```

### Makefile

This is the source code to compile and link the source code `module_sysctl.c` into a loadable kernel module `module_sysctl.ko`

```Makefile
KMOD=  module_sysctl
SRCS=  module_sysctl.c

.include <bsd.kmod.mk>
```
