# Random fun stuff



## Kernel Developers and weird things they name

Check your /sys/fs/cgroup/memory files.
My output on my vagrant vm is:
```
vagrant@ubuntu-xenial:/sys/fs/cgroup/memory$ ls -l
total 0
-rw-r--r--  1 root root 0 Nov 20 08:10 cgroup.clone_children
--w--w--w-  1 root root 0 Nov 20 08:10 cgroup.event_control
-rw-r--r--  1 root root 0 Nov 20 07:17 cgroup.procs
-r--r--r--  1 root root 0 Nov 20 08:10 cgroup.sane_behavior
drwxr-xr-x  2 root root 0 Nov 20 07:16 init.scope
-rw-r--r--  1 root root 0 Nov 20 08:10 memory.failcnt
--w-------  1 root root 0 Nov 20 08:10 memory.force_empty
-rw-r--r--  1 root root 0 Nov 20 08:10 memory.kmem.failcnt
-rw-r--r--  1 root root 0 Nov 20 08:10 memory.kmem.limit_in_bytes
-rw-r--r--  1 root root 0 Nov 20 08:10 memory.kmem.max_usage_in_bytes
-r--r--r--  1 root root 0 Nov 20 08:10 memory.kmem.slabinfo
-rw-r--r--  1 root root 0 Nov 20 08:10 memory.kmem.tcp.failcnt
-rw-r--r--  1 root root 0 Nov 20 08:10 memory.kmem.tcp.limit_in_bytes
-rw-r--r--  1 root root 0 Nov 20 08:10 memory.kmem.tcp.max_usage_in_bytes
-r--r--r--  1 root root 0 Nov 20 08:10 memory.kmem.tcp.usage_in_bytes
-r--r--r--  1 root root 0 Nov 20 08:10 memory.kmem.usage_in_bytes
-rw-r--r--  1 root root 0 Nov 20 08:10 memory.limit_in_bytes
-rw-r--r--  1 root root 0 Nov 20 08:10 memory.max_usage_in_bytes
-rw-r--r--  1 root root 0 Nov 20 08:10 memory.move_charge_at_immigrate
-r--r--r--  1 root root 0 Nov 20 08:10 memory.numa_stat
-rw-r--r--  1 root root 0 Nov 20 08:10 memory.oom_control
----------  1 root root 0 Nov 20 08:10 memory.pressure_level
-rw-r--r--  1 root root 0 Nov 20 08:10 memory.soft_limit_in_bytes
-r--r--r--  1 root root 0 Nov 20 08:10 memory.stat
-rw-r--r--  1 root root 0 Nov 20 08:10 memory.swappiness
-r--r--r--  1 root root 0 Nov 20 08:10 memory.usage_in_bytes
-rw-r--r--  1 root root 0 Nov 20 07:16 memory.use_hierarchy
-rw-r--r--  1 root root 0 Nov 20 08:10 notify_on_release
-rw-r--r--  1 root root 0 Nov 20 08:10 release_agent
drwxr-xr-x 63 root root 0 Nov 20 07:16 system.slice
-rw-r--r--  1 root root 0 Nov 20 08:10 tasks
drwxr-xr-x  2 root root 0 Nov 20 07:16 user.slice
```

Theres a file called...cgroup.sane\_behavior. It was set to 0 :o

Theres also a memory.swappiness 
Apparently the swappiness is a measure of how aggressively the kernel will swap memory pages. Who would've guessed.

