# Containers

Containers are a way to isolate processes and trick them to thinks they are the
only ones running on the machine.
Containers are not virtual machines: processes running in containers are normal
processes running on the host kernel (there is not guest kernel, kernel is
shared with the host).
Therefore, there is no performance drop.

## Namespaces

Linux implements PID namespaces: when a process creates a new namespace, it
becomes the root of its tree and all sub-processes receive two PIDs (old and
new one).
A child namespace cannot interact with the parent process tree.
Exploiting this technology, container applications use their own independent
process trees.

There are many types of namespaces, below some examples.

1. Unix Timeshared System (UTS) gives each process tree a host and a domain
    name.
2. Mount namespaces allow different views of the file system.
3. Network namespaces offer separate network stacks to processes.
4. User namespaces let processes inside containers to have different user and
    group identifiers.

