Linux Kernel Modules: hello, kernel!

# What a Kernel Module is

Kernel Modules are pieces of code that can be loaded into the kernel while the kernel is already running. Modules extend the functionality of the kernel. Many modules are already loaded into the kernel by default (built-in modules) (eg. mouse and keyboard drivers). But what if we need some functionality that is not already built-in? This is where loading new modules without rebooting the system comes in handy (loadable modules). We can do pretty much anything with kernel modules, because they literally adding your code to the kernel which implies that these modules run with kernel privileges (ring 0).

# Hello, kernel!

```c
#include <linux/init.h>
#include <linux/module.h>
#include <linux/sched.h>
#include <asm/current.h>
MODULE_LICENSE("Dual BSD/GPL");


static int print_pid(void){
    printk(KERN_INFO "The process \"%s\" has pid %i\n", current->comm, current->pid);
    return 0;
}

static int hello_init(void){
    printk(KERN_ALERT "Hello, kernel\n");
    print_pid();
    return 0;
}

static void hello_exit(void){
    printk(KERN_ALERT "Goodbye, kernel. I'll miss you\n");
}

module_init(hello_init);
module_exit(hello_exit);

```

`module_init`: executed when the module is loaded into the kernel, used to initialize variables and get information about hardware and other stuff

`module_exit`: executed when the module is been removed from the kernel, used to free up the memory and do a clean-up

## MakeFile

Make needs to run in the kernel source tree directly. It's something like this:
```makefile
obj-m += hello.o

all:
    make -C /lib/modules/$(shell uname -r)/build M=$(PWD) modules

clean:
    make -C /lib/modules/$(shell uname -r)/build M=$(PWD) clean
```
## The `print_pid` function
In the example, the print_pid function, called by `module_init`, prints the **pid** (process id) of the kernel module. We access at this information using the `current`, contained in the file `asm/current.h`:

```
static __always_inline struct task_struct *get_current(void)
{
	return this_cpu_read_stable(current_task);
}

#define current get_current()
```
as we can see, `current` is defined as a the function `get_current`, which returns a `task_struct` **struct**, fetched by the function `this_cpu_read_stable` with `current_task` as parameter.

The `task_struct` struct is declared in [`linux/sched.h`](https://github.com/torvalds/linux/blob/ab159ac569fddf812c0a217d6dbffaa5d93ef88f/include/linux/sched.h#L649), while the function `this_cpu_read_stable` as a comment in `asm/percpu.h` describes, "*makes gcc load the percpu variable every time it is accessed while this_cpu_read_stable() allows the value to be cached. this_cpu_read_stable() is more efficient and can be used if its value is guaranteed to be valid across cpus.  The current users include get_current() and get_thread_info() both of which are actually per-thread variables implemented as per-cpu variables and thus stable for the duration of the respective task."

The important concept is that `task_struct` contains information about the process that is currently running, and in our program we use that in order to know the name and the id of the current process. But *which* is the current process?

As "Linux Device Driver" tell to us, *During the execution of a system call, such as open or read, the current process is the one that invoked the call.*, but currently, the process that is making some system call is `insmod`! (if you don't belive me, you can use `strace` to print every system call that a program does).

In order to prove this we can modify our hello.c file, printing `current` inside the module exit function `hello_exit`. If what I said is true, we should see that the process name (`comm` field in `current` struct, `current->comm`) is `rmmod` and actually, when I typed `sudo rmmod hello.ko` in the kernel I saw `endeavour kernel: The process "rmmod" has pid 16299` 





### Resources
- [LDD3: Building and Running Modules](https://static.lwn.net/images/pdf/LDD3/ch02.pdf)
- [CTF-wiki: Introduction to linux kernel modules](https://csea-iitb.github.io/IITBreachers-wiki/2021/02/28/Intro-to-Linux-Kernel-Modules.html)
- [Linux Source: asm/current.h](https://github.com/torvalds/linux/blob/master/arch/x86/include/asm/current.h)
- [Linux Source: asm/percpu.h](https://github.com/torvalds/linux/blob/arch/x86/include/asm/percpu.h)
- [Linux Source: linux/sched.h](https://github.com/torvalds/linux/blob/master/include/linux/sched.h)
- [SO: Unable to understand how the "current" macro works for x86 architecture](https://stackoverflow.com/questions/53940893/unable-to-understand-how-the-current-macro-works-for-x86-architecture)
