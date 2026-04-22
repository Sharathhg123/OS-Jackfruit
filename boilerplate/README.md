## 4. Engineering Analysis

### Isolation Mechanisms

My runtime achieves isolation using three Linux namespaces created via `clone()` with flags `CLONE_NEWPID | CLONE_NEWUTS | CLONE_NEWNS`:

- **CLONE_NEWPID** – Each container gets its own PID namespace. The first process inside sees itself as PID 1 and cannot see or signal host processes.
- **CLONE_NEWUTS** – Each container gets its own hostname, set via `sethostname()`.
- **CLONE_NEWNS** – Each container gets its own mount namespace. Combined with `chroot(rootfs)`, the container sees only its assigned Alpine Linux filesystem.

The host kernel is still fully shared across all containers – same system call table, same scheduler, same physical memory management. Namespaces only restrict what each process can see, not what the kernel does internally.

### Supervisor and Process Lifecycle

A long-running supervisor is essential for three reasons:
1. It must stay alive to call `waitpid()` on exiting children – if the parent exits first, orphaned children are re-parented to init, making metadata tracking impossible.
2. It maintains container state across the full lifetime.
3. It accepts CLI commands at any time without blocking.

**Lifecycle:** `clone()` creates child with new namespaces → child calls `container_setup()` which mounts rootfs, calls `chroot()`, and `exec()`s the workload → parent stores `host_pid` and sets state `running` → `SIGCHLD` fires on child exit → parent's handler calls `waitpid(-1, &status, WNOHANG)` to reap all finished children → state updated to `stopped` or `killed`.

Without `WNOHANG`, the supervisor would block on `waitpid` and be unable to service other containers. Without calling `waitpid`, exited containers become zombies.

### IPC, Threads, and Synchronization

I use two distinct IPC mechanisms:

**Pipes** capture log output. A `pipe(pipefd)` is created before `clone()`. The child inherits the write end and we `dup2` it onto `stdout` and `stderr`. The parent reads from the read end. This suits logging because data flows in only one direction as a continuous byte stream.

**UNIX domain socket** handles CLI commands. The supervisor `bind()`s to `/tmp/engine_supervisor.sock` and `listen()`s. Each CLI invocation connects, sends a command, reads the response, and disconnects.

**Bounded Buffer Synchronization:** A circular array of `BUF_SLOTS` slots protected by:
- `pthread_mutex_t lock` – only one thread modifies buffer state at a time
- `pthread_cond_t not_empty` – consumer sleeps when buffer is empty
- `pthread_cond_t not_full` – producer sleeps when buffer is full

This guarantees no lost data, no corruption, and no deadlock.

### Memory Management and Enforcement

**RSS (Resident Set Size)** measures physical memory pages currently in RAM for a process: `get_mm_rss(task->mm) * PAGE_SIZE`. It does not include: pages swapped to disk, memory allocated but never touched, or shared library pages.

**Soft limit** is advisory – the process continues, giving the operator time to react. The `soft_warned` flag ensures the warning fires only once.

**Hard limit** is enforcement – the process is unconditionally killed with `SIGKILL`.

Enforcement belongs in **kernel space** because: user-space monitors can be delayed by the scheduler; a user-space monitor can itself be killed; only the kernel can atomically read `mm_struct` fields and issue `SIGKILL` in a non-preemptible context.

### Scheduling Behavior

Linux uses the **CFS (Completely Fair Scheduler)**. Every runnable process has a `vruntime` tracked in a red-black tree. CFS always runs the process with the smallest `vruntime`. When a process runs, its `vruntime` increases proportional to actual CPU time divided by its scheduling weight. Weight is set by nice value (nice=0 → weight 1024, nice=+5 → weight 335, nice=-5 → weight 3121).

In my experiments:
- **Equal priority CPU-bound:** Both containers received equal CPU shares (CFS fairness).
- **Different nice values:** Higher priority (nice=-5) received ~45% more CPU than lower priority (nice=+10).
- **CPU-bound vs I/O-bound:** The CPU-bound container dominated because I/O-bound processes voluntarily yield the CPU when blocking on I/O syscalls.
