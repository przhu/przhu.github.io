---
layout: post
title: "Mach Message and Bootstrap Server on OS X"
description: ""
category: "Using Mac"
tags: [C, "Mach", "bootstrap server", "OS X", "interprocess communication", "init", "launchd"]
---
{% include JB/setup %}

This passage is mainly about low level microkernel interface provided by Mac OS X,
and all tests are on a late-2011 Mac Book Pro with a Mac OS X 10.7.4 system.
Apple generally discourages users using these raw interfaces since they may
result mess crashes, security problems, resource leaks, etc and says users should
only use mach messages for IPC. `Foundation.framework` and `CoreFoundation.framework`
have their coresponding wrappers, namely `NSMachPort`, `NSMessagePort`, 
`CFMachPort`, `CFMessagePort`, which may be preferred.

Since Apple constantly changes the micokernel interface, my discussion may totally
unappliable to you. However some general idea may be applied also. I think the best
way is to dive in XNU kernel source, launchd source, and refer to old Mach Manual Pages 
(`$KERNEL_SOURCE_TREE/osfmk/man/`, outdated but general usages are described), the
_Mach 3 Server Writer's Guide_ (may be hard to understand if you are not familiar
 with operating system design principles), Apple's _Kernel Programming Guide: Mach Overview_
(also not refering current, some features promised by the guide are not implemented (at least
not exposed as public C API) even today) and/or CMU's orginal mach document, GNU Mach document
for principles, ideas.

Now I assume readers reaching here have basic mach port and mach messaging knowledge. 
In mach messaging one important problem is how other programmers provide their ports' send or 
send once right to other tasks. Thus the system provides bootstrap server for this purpose (
netname server in _Server Writer's Guide_). Bootstrap subsystem is declared in `<servers/bootstrap.h>`.
The service is now provided by launchd (pid 1) and the header file includes several deprecated
declaration, which upsets me for some time. Finally I have managed to get the full picture.

	kern_return_t bootstrap_check_in(
	        mach_port_t bp, 
	        const name_t service_name,
	        mach_port_t *sp);

Use `bootstrap_port` external variable as bp, your favorite string as `service_name`,
 this API directly gives you a port with receive right in sp. The header document 
says registering the name using launchd.plist with launchd is required, which is not
true. The full story is, if call this from a normal unix process executed from terminal, 
launchd treats this process as a 'legacy' task, and dynamically bind the name with the port.
Currently it also does the binding for not 'legacy' job (Cocoa app?) but emits a warning 
which "deprecated the usage due to performance reasons". 

	kern_return_t bootstrap_look_up(
	        mach_port_t bp,
	        const name_t service_name,
	        mach_port_t *sp);

Use this API to receive a port with send right. Mach allows one task hold multiple
reference to the same send right, but mach limits total send right reference count 
and dead port reference count a task can hold, so pay attention to right reference leak. 

	AVAILABLE_MAC_OS_X_VERSION_10_0_AND_LATER_BUT_DEPRECATED_IN_MAC_OS_X_VERSION_10_5
	kern_return_t
	bootstrap_register(mach_port_t bp, name_t service_name, mach_port_t sp);

This API is deprecated since launchd was introduced. It requires the user provides a
send right of sp, which is very convenient for transfering `semaphore_t`, `lockset_t`,
even `task_t` and more. The header documents said please use `mach_msg` to send them
directly. Now take this scene into account: process A gives process B a send right,
then B call `bootstrap_register` to make the send right available to others (by default
this will be available to any process with the same UID). This is a common problem
need to be deeply considered by server developers. For IPC between co-operative processes,
I think `bootstrap_subset` routine is suitable for namespace isolation here.

	kern_return_t bootstrap_subset(
	        mach_port_t bp,
	        mach_port_t requestor_port,
	        mach_port_t *subset_port);

The first argument is still the bootstrap port, the second argument is the task port (
send right!), which in most case , is `mach_task_self()`, the last argument is the returned 
(created) subset port. Names registered with subset port is only available in the subset and 
its subset while names registered with original bootstrap port is available to the subset 
also. Since only root user is allowed to use `bootstrap_parent` to access given bootstrap port's 
parent (documented in the header), a normal task is not able to change the subset by acrossing
the parent. The system uses this technique to isolate users to there own subset.

After creating a subset, set the new subset as the `bootstrap_port`:

	task_set_bootstrap_port(task, port)

In Unix, the common way to create a new process is `fork()`, so I have to know after `fork`,
how mach ports are inherited. After examined XNU kernel source, I know the forked process
only gets a task special port (`mach_task_self()`) and inherits only task special ports including 
bootstrap port and some other ports (Ref. `<mach/task_special_ports.h>`, not sure whether they are 
all set and set). For the detail, code is listed below.

{% highlight c %}
/*
  API Illustrating only.
 */

#include <servers/bootstrap.h>
#include <unistd.h>
#include <mach/mach_interface.h>

/*
  extern mach_port_t bootstrap_port;
 */

int main()
{
        pid_t pid;
        mach_port_t subset_port, service;

        /* create a subset_port */
        bootstrap_subset(bootstrap_port, mach_task_self(), &subset_port);
        /* set the task's bootstrap_port at kernel */
        task_set_bootstrap_port(mach_task_self(), subset_port);
        /* now the external variable `bootstrap_port' does not refer to the real bootstrap port registered
           at kernel, it refers the original bootstrap port. Kernel routines does not know these external 
           variables.
         */
        /*
           Ref. fork_mach_init() in /libsyscall/mach/mach_init.c in XNU source
           This routine called explicitly by _cthread_fork_child(), which calls mach_init_ports()
           For new exectables(exec family), mach_init() (also in that file) is called by static images or by dyld.
         */

        pid = fork();
        /* At mach side, only task special ports are created(mach_task_self)/copyed(bootstrap, host...) */
        if (pid == 0) {
                bootstrap_check_in(bootstrap_port, "CHILD", &service);
                /* Results in subset */
        } else {
                bootstrap_check_in(bootstrap_port, "PARENT", &service);
                /* Results in the original set */
                /* explicitly add one in subset */
                bootstrap_check_in(subset_port, "PARENT-1", &service);
                /*
                   explicitly set the external variable, 
                     bootstrap_port = subset_port; 
                   is also sufficient.
                 */
                task_get_bootstrap_port(mach_task_self(), &bootstrap_port);
                bootstrap_check_in(bootstrap_port, "PARENT-2", &service);
                /* now also subset */
                /* 
                  WANRING: The above code illustrate a port-right user reference leak
                 */
        }

        pause();
}
{% endhighlight %}

To check the results, I placed `pause()` at the end of the code. 
Use `launchctl` utility to check the results.

	launchctl bslist

which lists all mach ports binded with name available in the system. While

	sudo launchctl bstree

prints the Mach bootstrap hierarchy tree. In the system tree, there are 
some subsets, noticably, the root set `System/`, the 
`com.apple.launchd.peruser.$UID (Per-user)/` subset. Executes the sample
code and uses the `bstree` command to verify what I said in the code.

At last

	launchctl bsexec PID command ...

is used for userspace explictly start a command using the same bootstrap subset
as process idetified by PID and it requires privileges for `task_for_pid()`
call. This is another hard topic and can be found from web and mainly used 
by administrators and debuggers. As previously noted, bootstrap subset is 
for namespace isolation, this level of security is enough. 

In summary, this post roughly discussed bootstrap server, bootstrap subset and
how to use bootstrap server to initiate mach communication in Mac OS X. I will
post another article soon about my more experience in exploring OS X's Mach which
mainly about sharing memory.

## ACKNOWLEDGEMENT

Sorry for another long post. Feel free to comment. 
Names or trademarks are owned by coresponding author(s)/orgnazition(s). Use of these names with best regards and wishes.
