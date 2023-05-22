This simple Linux kernel tutorial is based on the book ["The Linux Kernel Module Programming Guide"](https://sysprog21.github.io/lkmpg/). We are using Ubuntu to bind the examples.

A Linux kernel module, often referred to as simply a "kernel module" or "LKM," is a piece of code that can be dynamically loaded into the Linux kernel at runtime. The Linux kernel is the core component of the Linux operating system and is responsible for managing system resources and providing essential functionality. Kernel modules allow developers to extend the functionality of the Linux kernel without modifying or recompiling the entire kernel. They can add device drivers, file systems, networking protocols, and various other features to the kernel on-demand, without the need to reboot the system. 


Prerequisits
----
Install compiling tools
```shell
sudo apt-get install build-essential checkinstall
```
Install kernel-headers
```shell
sudo apt-get install linux-headers-$(uname -r)
```
Kernel modules are typically compiled as separate object files with a .ko extension. They contain the code and data required to provide the desired functionality. When a module is loaded into the kernel, it becomes an integral part of the running system, able to interact directly with the kernel and other modules. Modules can be loaded and unloaded dynamically using the `insmod`, `rmmod`, and `modprobe` commands in Linux. They can also be automatically loaded at system startup by configuring the system's module loading mechanisms. One of the significant advantages of kernel modules is their ability to conserve system resources. Modules are loaded into memory only when needed, reducing memory consumption and allowing for greater flexibility in managing system features.

Hello World
---


```c
/*
 * hello-1.c - The simplest kernel module.
 */
#include <linux/module.h> /* Needed by all modules */
#include <linux/kernel.h> /* Needed for KERN_INFO */
int init_module(void)
{
 printk(KERN_INFO "Hello world 1.\n");
 /*
 * A non 0 return means init_module failed; module can't be loaded.
 */
 return 0;
}
void cleanup_module(void)
{
 printk(KERN_INFO "Goodbye world 1.\n");
}
```


Kernel modules in Linux have specific requirements and functions. Traditionally, a module must include two functions: `init_module()` for initialization and `cleanup_module()` for cleanup before removal using `rmmod`. However, starting from kernel 2.3.13, you can use any names for these functions, and the new method is now preferred. Nevertheless, many developers still use `init_module()` and `cleanup_module()` for convenience.

Typically, `init_module()` registers a handler or replaces a kernel function with custom code, often performing specific tasks before calling the original function. On the other hand, `cleanup_module()` undoes the changes made by `init_module()` to ensure safe unloading of the module. To work as a kernel module, every module needs to include `linux/module.h`. Additionally, including `linux/kernel.h` is necessary for the expansion of macros used for the `printk()` log level..
The `printk()` function in the Linux kernel is primarily used as a logging mechanism rather than for communicating information to the user. It allows for logging information and issuing warnings within the kernel. Each `printk()` statement includes a priority level, indicated by `<1>` and the `KERN_ALERT` tag. The kernel provides macros for eight priority levels, making it easier to specify the priority rather than using numeric values. The `linux/kernel.h` header file contains the definitions and meanings of these priorities.

If no priority level is specified, the default priority level, DEFAULT_MESSAGE_LOGLEVEL, is used. It is recommended to use the priority macros instead of cryptic numbers, such as `<4>`. For example, the macro KERN_WARNING indicates a warning level.

When a message's priority is lower than the `console_loglevel` setting, the message is printed on the current terminal. If both syslogd and klogd are running, the message will be appended to `/var/log/messages` regardless of whether it was printed on the console. Using a high priority level like KERN_ALERT ensures that the `printk()` messages are displayed on the console rather than just being logged to a file.

In practical usage, it is important to choose meaningful priority levels for the specific situation when writing real modules.

Compile the Kernel Module
---
There is a generic Makefile structure that is dedicated to compile kernel modules.
```Makefile
obj-m += hello-1.o
all:
 make -C /lib/modules/$(shell uname -r)/build M=$(PWD) modules
clean:
 make -C /lib/modules/$(shell uname -r)/build M=$(PWD) clean
```

Once you compile the modules, it will show you the following results on the terminal.

```shell
hostname:~/…/LinuxKernel/HelloWorldKernel/src $ make
make -C /lib/modules/5.8.0-59-generic/build M=/home/aruna/Documents/LinuxKernel/HelloWorldKernel/src modules
make[1]: Entering directory '/usr/src/linux-headers-5.8.0-59-generic'
  CC [M]  /home/user/Documents/LinuxKernel/HelloWorldKernel/src/hello-1.o
  MODPOST /home/user/Documents/LinuxKernel/HelloWorldKernel/src/Module.symvers
WARNING: modpost: missing MODULE_LICENSE() in /home/user/Documents/LinuxKernel/HelloWorldKernel/src/hello-1.o
  CC [M]  /home/user/Documents/LinuxKernel/HelloWorldKernel/src/hello-1.mod.o
  LD [M]  /home/user/Documents/LinuxKernel/HelloWorldKernel/src/hello-1.ko
make[1]: Leaving directory '/usr/src/linux-headers-5.8.0-59-generic'
```

This will generate kernel object files (\*.ko) which carry extra information than the normal object (\*.o) files. This extra information can be elaborated using `modinfo` command.

```shell
hostname:~/…/LinuxKernel/HelloWorldKernel/src $ modinfo hello-1.ko
filename:       /home/user/Documents/LinuxKernel/HelloWorldKernel/src/hello-1.ko
srcversion:     140276773A3090F6F33891F
depends:        
retpoline:      Y
name:           hello_1
vermagic:       5.8.0-59-generic SMP mod_unload modversions 
```

Hello World example does not contain much of details yet since it just performs a write to the log file.


Insert in to Kernel
---

Now we have the compiled kernel module, we just need to insert it into the linux kernel. For that we are using `insmod` command.

```shell
hostname:~/…/LinuxKernel/HelloWorldKernel/src $  sudo insmod ./hello-1.ko
```

we can look into the kernel log file (`/var/log/syslog`), whether the "Hello World" message got logged in.

```shell
hostname:~/…/src $ sudo cat /var/log/syslog | grep 'Hello world'
May 17 15:43:36 eslab1 kernel: [2585797.085213] Hello world 1.
```

Remove a Kernel
---

We can just remove a kernel module from the kernel by simply using `rmmod` command.

```shell
hostname:~/…/src $  sudo rmmod hello-1
```

Spaning Kernel Modules
----

In reallity, the Kernel module implementations span across multiple source files. Compiling them to one is just a matter of modifying the Makefile. Lets consider the following hello world program that utilize `linux/init.h`. This header file defines new marcos for the methods that we used in `hello_1.c`. Here `init_module()` is replaces by `module_init()` and `cleanup_module()` is replaced by `module_exit()`.


```c
/*
 * hello-2.c - Demonstrating the module_init() and module_exit() macros.
 * This is preferred over using init_module() and cleanup_module().
 */
#include <linux/module.h> /* Needed by all modules */
#include <linux/kernel.h> /* Needed for KERN_INFO */
#include <linux/init.h> /* Needed for the macros */
static int __init hello_2_init(void)
{
 printk(KERN_INFO "Hello, world 2\n");
 return 0;
}
static void __exit hello_2_exit(void)
{
 printk(KERN_INFO "Goodbye, world 2\n");
}
module_init(hello_2_init);
module_exit(hello_2_exit);
```


Now we have to modify the Makefile such that the `hello-2.c` file is included in the sources for compile.

```makefile
obj-m += hello-1.o
obj-m += hello-2.o
all:
 make -C /lib/modules/$(shell uname -r)/build M=$(PWD) modules
clean:
 make -C /lib/modules/$(shell uname -r)/build M=$(PWD) clean
```


In this tutorial, we looked at the process of writing a simple Linux kernel module and linked it to the kernel of the Ubuntu operating system. In the next tutorial, let's look at how to use command line inputs with kernel modules.