---
layout: post
title: "Sharing memory using Mach part of OS X"
description: ""
category: "Using Mac"
tags: ["C", "Mach", "OS X", "interprocess communication", "shared memory", "semaphore"]
---
{% include JB/setup %}

This passage continues <a href="{{ page.previous.url }}">the previous post</a> about 
low level microkernel interface provided by Mac OS X, and also all tests are on a 
late-2011 Mac Book Pro with a Mac OS X 10.7.4 system. This time is mainly about sharing memory.

For sharing memory, proper synchronization method should be applied. Here I chose the semaphore
provided by Mach (declared in `<mach/semaphore.h>`). The interface is rather straight forward:
`semaphore_create`, `semaphore_signal`, `semaphore_wait`. The only thing may need to explain is
the `sync_policy` parameter in `semaphore_create`, as far as I can see, the only one supported
is `SYNC_POLICY_FIFO`, which I think is OK for most synchronization and fast.

The first way is to use `vm_inherit`. This call sets the specified pages' inheritance attribute:
`NONE`, `COPY`, `SHARE`. If it is set to `NONE`, the pages involved will not be inherited by the 
child task. If it is set to `SHARE`, the pages will be shared by children. `COPY` is the default,
and is aligned with Unix `fork`. OpenBSD also has `minherit` syscall, which is the same in semantic
meaning and OS X also provides it for compability to do the same work as `vm_inherit`.

Another shared memory facility involves a `memory_entry` which I had not seen elsewhere before. They
are not mentioned in old Mach documents. In these documents, `vm_inherit` is the only way to share 
memory without using a pager. Users can ask the pager to provide a 'memory object' and map (`vm_map`)
to the task vm space. However when I was consulting `<mach/vm_map.h>` I found the type used in `vm_map`
is a `mem_entry_name_port_t`. After several confirmation it may be a creation by Apple. For usage,
I found an example usage: [the 'commpage' machanism used by OS X](http://fxr.watson.org/fxr/source/osfmk/i386/commpage/commpage.c?v=xnu-1456.1.26;im=excerpts#L135). 

Also in the system, two set of the API co-exist. The one set is only prefixed with `vm_`, the other 
set is prefixed with `mach_vm_`, `mach_vm_` version is always 64bit(`uint64_t`), 
`vm_` version is 32bit(`int` in Mach-O i386) or 64bit(`long` in Mach-O x86\_64). For this topic,
I have prepared a better sample code (compared to the code provided by previous post). You can 
access it at [this gist](https://gist.github.com/3481836).

Firstly I prepare memory using `mach_vm_allocate` and then call
`mach_make_memory_entry_64` to create a memory entry with allocated area and create a semaphore.
Next, name the memory entry and semaphore via bootstrap server (deprecated, for sample purpose). 
Fork the child and use `mach_vm_map` to map the memory entry in the child process.
Now compare the memory inherited via fork (COW!) and the memory shared explictly.
Then I try to verify the writing. Finally I verify whether the parent process
can see the modification.

Unfortunately XCode 4.2's cc and clang miscompile code when `-O1` or higher. MacPorts' gcc does not
have this problem. If compiled without optimization with XCode 4.2 or compiled with FSF gcc (4.4, 4.6 
from MacPorts both have good results), everything works as it should be. 

## ACKNOWLEDGEMENT

Sorry for long code :). Feel free to comment. 
Names or trademarks are owned by coresponding author(s)/orgnazition(s). Use of these names with best regards and wishes.
