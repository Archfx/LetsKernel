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
 * hello−1.c − The simplest kernel module.
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
obj−m += hello−1.o
all:
 make −C /lib/modules/$(shell uname −r)/build M=$(PWD) modules
clean:
 make −C /lib/modules/$(shell uname −r)/build M=$(PWD) clean
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
May 17 15:43:36 hostname kernel: [2585797.085213] Hello world 1.
```

Remove a Kernel
---

We can just remove a kernel module from the kernel by simply using `rmmod` command.

```shell
hostname:~/…/src $  sudo rmmod hello-1
```

Command Line Args
----

Finally, let's look at how to pass command line arguments to the programs during the `insmod` process. To enable passing command line arguments to your module, you need to declare the variables that will store the argument values as global variables. Then, you can use the `module_param()` macro, defined in `linux/moduleparam.h` to set up the mechanism. During runtime, when using `insmod`, the variables will be populated with the provided command line arguments. For example, `./insmod mymodule.ko myvariable=5` would assign the value 5 to the variable `myvariable`. To ensure clarity, it is recommended to place the variable declarations and macros at the beginning of the module. The below example code will help clarify the process.

```c
int myint = 3;
module_param(myint, int, 0);
```

The `module_param()` macro expects three arguments: the variable's name, its type, and the permissions for the corresponding file in the sysfs. Integer types can be signed or unsigned as needed. If you want to work with arrays of integers or strings, you can refer to the `module_param_array()` and `module_param_string()` macros. Arrays are indeed supported for passing command line arguments to modules. To keep track of the number of parameters, you now need to provide a pointer to a count variable as the third parameter. Alternatively, you have the option to ignore the count and pass NULL instead. Here, following example demonstrate both possibilitie,

```c
int myintarray[2];
module_param_array(myintarray, int, NULL, 0); /* not interested in count */
int myshortarray[4];
int count;
module_parm_array(myshortarray, short, , 0); /* put count into "count" variable */
```

One practical use of this approach is to set default values for module variables, such as a port or IO address. By assigning default values to the variables, you can perform autodetection if the variables still hold the default values. Otherwise, you can retain the current values. This concept will be further explained in subsequent sections.

Additionally, there is a macro function called `MODULE_PARM_DESC()` that serves the purpose of documenting the arguments that a module can accept. It requires two parameters: the variable name and a descriptive string that provides information about the variable. This allows for better documentation and understanding of the module's usage.

Following is the complete code that can be used to pass command line parameters

```c
/*
 * hello-5.c - Demonstrates command line argument passing to a module.
 */
#include <linux/module.h>
#include <linux/moduleparam.h>
#include <linux/kernel.h>
#include <linux/init.h>
#include <linux/stat.h>
MODULE_LICENSE("GPL");
MODULE_AUTHOR("Peter Jay Salzman");
static short int myshort = 1;
static int myint = 420;
static long int mylong = 9999;
static char *mystring = "blah";
static int myintArray[2] = { -1, -1 };
static int arr_argc = 0;
/*
 * module_param(foo, int, 0000)
 * The first param is the parameters name
 * The second param is it's data type
 * The final argument is the permissions bits,
 * for exposing parameters in sysfs (if non-zero) at a later stage.
 */
module_param(myshort, short, S_IRUSR | S_IWUSR | S_IRGRP | S_IWGRP);
MODULE_PARM_DESC(myshort, "A short integer");
module_param(myint, int, S_IRUSR | S_IWUSR | S_IRGRP | S_IROTH);
MODULE_PARM_DESC(myint, "An integer");
module_param(mylong, long, S_IRUSR);
MODULE_PARM_DESC(mylong, "A long integer");
module_param(mystring, charp, 0000);
MODULE_PARM_DESC(mystring, "A character string");
/*
 * module_param_array(name, type, num, perm);
 * The first param is the parameter's (in this case the array's) name
 * The second param is the data type of the elements of the array
 * The third argument is a pointer to the variable that will store the number
 * of elements of the array initialized by the user at module loading time
 * The fourth argument is the permission bits
 */
module_param_array(myintArray, int, &arr_argc, 0000);
MODULE_PARM_DESC(myintArray, "An array of integers");
static int __init hello_5_init(void)
{
 int i;
 printk(KERN_INFO "Hello, world 5\n=============\n");
 printk(KERN_INFO "myshort is a short integer: %hd\n", myshort);
 printk(KERN_INFO "myint is an integer: %d\n", myint);
 printk(KERN_INFO "mylong is a long integer: %ld\n", mylong);
 printk(KERN_INFO "mystring is a string: %s\n", mystring);
 for (i = 0; i < (sizeof myintArray / sizeof (int)); i++)
 {
 printk(KERN_INFO "myintArray[%d] = %d\n", i, myintArray[i]);
 }
 printk(KERN_INFO "got %d arguments for myintArray.\n", arr_argc);
 return 0;
}
static void __exit hello_5_exit(void)
{
 printk(KERN_INFO "Goodbye, world 5\n");
}
module_init(hello_5_init);
module_exit(hello_5_exit);
```


After correctly referring to the `hello-5.c` in the Makefile, you can build the `hello-5.ko` and now you can pass command line args into the kernel module as we discussed earlier.


```shell
hostname:~/…/src $ sudo insmod hello-5.ko mystring="bebop" mybyte=255 myintArray=-1
hostname:~/…/src $ sudo cat /var/log/syslog 
```

<code>
xxx hostname kernel: [5435833.841255] Hello, world 5 <br>
xxx hostname kernel: [5435833.841255] ============= <br>
xxx hostname kernel: [5435833.841256] myshort is a short integer: 1 <br>
xxx hostname kernel: [5435833.841257] myint is an integer: 420 <br>
xxx hostname kernel: [5435833.841258] mylong is a long integer: 9999 <br>
xxx hostname kernel: [5435833.841258] mystring is a string: bebop <br>
xxx hostname kernel: [5435833.841259] myintArray[0] = -1 <br>
xxx hostname kernel: [5435833.841260] myintArray[1] = -1 <br>
xxx hostname kernel: [5435833.841260] got 1 arguments for myintArray. <br>
</code>


In this tutorial, we looked at the process of writing a simple Linux kernel module and linked it to the kernel of the Ubuntu operating system. Finally, we looked at how to use command line inputs with kernel modules. In the text post, we will look at how to write a simple kernel driver to control external devices.