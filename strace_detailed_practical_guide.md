# `strace` — Detailed Practical Guide

## 1. What is `strace`?

`strace` is a Linux diagnostic tool used to **trace system calls and signals** made by a process.

Think of it as a window between an application and the Linux kernel:

```text
Application
     |
     | system calls
     v
Linux Kernel
     |
     +-- files
     +-- network
     +-- processes
     +-- devices
     +-- resources
     +-- signals
```

For a Java application:

```text
Java Application
       |
      JVM
       |
       v
Linux system calls
       |
       v
Linux kernel
```

`strace` observes this system-call boundary.

---

# 2. Why Use `strace`?

`strace` is useful when a program:

- Hangs
- Starts slowly
- Cannot find a file
- Gets `Permission denied`
- Cannot connect to a service
- Performs unexpected file I/O
- Performs unexpected network operations
- Creates or waits for processes
- Receives signals
- Appears to be doing nothing
- Has unclear operating-system-level behavior

It can also help with performance investigation.

However:

> `strace` is not a replacement for a Java profiler.

For Java performance analysis, use it together with tools such as JFR/JMC, `pidstat`, `iostat`, and application metrics.

---

# 3. What Is a System Call?

A system call is how a user-space program asks the Linux kernel to perform an operation.

Conceptually:

```text
User space
-----------------------------
Java
JVM
Application
-----------------------------
          |
          | system call
          v
Kernel space
-----------------------------
Linux kernel
-----------------------------
          |
          +-- filesystem
          +-- network
          +-- process management
          +-- devices
```

Common system calls include:

```text
openat()
read()
write()
close()
connect()
accept()
socket()
execve()
clone()
futex()
poll()
epoll_wait()
```

---

# 4. Understanding a `strace` Line

A typical line looks like:

```text
read(3, "hello", 4096) = 5
```

Break it down:

```text
read
  |
  +-- system call

(3, "hello", 4096)
  |
  +-- arguments

= 5
  |
  +-- return value
```

The process asked Linux to read from file descriptor `3`, and the kernel returned `5`.

Another example:

```text
openat(..., "/tmp/test.txt", ...) = 3
```

The file was successfully opened and file descriptor `3` was returned.

A failure might look like:

```text
openat(..., "/tmp/missing.txt", ...) = -1 ENOENT
```

`ENOENT` means the requested file or directory does not exist.

---

# 5. Basic Usage

## Trace a new command

```bash
strace ls
```

This runs `ls` under `strace` and prints the system calls it makes.

## Trace a Java application

```bash
strace java -jar app.jar
```

For real applications, the output can be very large, so filtering is usually better.

---

# 6. Attach to a Running Process

Find a Java process:

```bash
pgrep -af java
```

Example:

```text
12345 java -jar myapp.jar
```

Attach:

```bash
sudo strace -p 12345
```

This lets you inspect an already-running application without restarting it.

You may need `sudo` because Linux restricts tracing of processes owned by another user.

---

# 7. Trace File Activity

Use:

```bash
sudo strace -p <PID> -e trace=file
```

You may see:

```text
openat(...)
stat(...)
access(...)
readlink(...)
unlink(...)
rename(...)
```

Useful questions include:

> Which configuration file is my application opening?

> Which file is the application trying to access?

> Why does the application say a file does not exist?

---

# 8. Trace Read and Write Operations

Use:

```bash
sudo strace -p <PID> -e trace=read,write
```

For positional I/O too:

```bash
sudo strace -p <PID> -e trace=read,write,pread64,pwrite64
```

You may see:

```text
read(5, ..., 4096) = 4096
write(6, ..., 1024) = 1024
```

This proves that the process is making read/write system calls.

## Important: `read()` does not always mean physical disk access

Linux has a page cache:

```text
Application
    |
    | read()
    v
Linux
    |
    +---- data already in page cache (RAM)
    |
    +---- storage device if data is not cached
```

Therefore:

```text
read() system call
       !=
physical SSD/HDD read
```

Use `iostat` and other storage metrics when you need to determine actual device activity.

---

# 9. Trace Network Activity

Use:

```bash
sudo strace -p <PID> -e trace=network
```

You may see:

```text
socket(...)
connect(...)
accept(...)
sendto(...)
recvfrom(...)
sendmsg(...)
recvmsg(...)
```

For example:

```text
connect(5, ...) = 0
```

means the process successfully performed a connection operation.

This is useful for investigating HTTP clients, database connections, Redis, Kafka, TCP services, and connection failures.

---

# 10. Trace Process Creation

Common process-related system calls include:

```text
clone()
fork()
vfork()
execve()
```

Trace them with:

```bash
strace -e trace=process command
```

This helps determine when an application starts another process.

---

# 11. Trace Signals

`strace` also traces Linux signals.

Examples:

```text
SIGINT
SIGTERM
SIGKILL
SIGSEGV
SIGPIPE
SIGCHLD
```

Signals are notifications delivered to processes.

This is useful for investigating:

- Graceful shutdown
- Ctrl+C
- Process termination
- Crashes
- Child process changes
- Signal-related problems

---

# 12. Follow Child Processes and Threads

Use:

```bash
strace -f -p <PID>
```

`-f` tells `strace` to follow processes/threads created by the traced process.

This can be particularly useful for Java because a JVM normally has many threads.

---

# 13. Get a System-Call Summary

Instead of printing every call:

```bash
strace -c command
```

For an existing process:

```bash
sudo strace -c -p <PID>
```

You may get output conceptually like:

```text
% time   seconds   calls   errors   syscall
------   -------   -----   ------   --------
45.2     0.52      10000   0        futex
20.1     0.23       5000   0        read
15.4     0.18       3000   0        write
...
```

This helps answer:

> Which system calls are occurring most often and where is system-call time being spent?

---

# 14. Save Output to a File

Instead of printing everything:

```bash
strace -o trace.log command
```

Then:

```bash
less trace.log
```

For a running process:

```bash
sudo strace -o trace.log -p <PID>
```

---

# 15. Use Filters

Avoid tracing everything on a busy application unless necessary.

Useful filters:

### Files

```bash
-e trace=file
```

### Network

```bash
-e trace=network
```

### Processes

```bash
-e trace=process
```

### Reads/writes

```bash
-e trace=read,write
```

### Specific calls

```bash
strace -e trace=openat,read,write,close command
```

---

# 16. Debug "File Not Found"

Suppose your Java application reports:

```text
Configuration file not found
```

Run:

```bash
sudo strace -f -e trace=file -p <PID>
```

You might find:

```text
openat(..., "/app/config/application.yml", ...) = -1 ENOENT
```

Now you know:

```text
Path:
 /app/config/application.yml

Error:
 ENOENT
```

This is much faster than guessing which path the application is using.

---

# 17. Debug Permission Problems

Suppose the application says:

```text
Permission denied
```

Trace files:

```bash
sudo strace -f -e trace=file -p <PID>
```

You might see:

```text
openat(..., "/var/log/app.log", ...) = -1 EACCES
```

`EACCES` indicates a permissions/access problem.

Then inspect:

```bash
ls -l /var/log/app.log
```

and the process user/group.

---

# 18. Debug Connection Refused

Suppose a Java application cannot connect to another service.

Run:

```bash
sudo strace -f -e trace=network -p <PID>
```

You may see:

```text
connect(...) = -1 ECONNREFUSED
```

This tells you the connection was refused.

Then check whether the destination service is listening:

```bash
ss -tulpn
```

---

# 19. Debug a Hanging Program

If a Java application appears stuck:

```bash
sudo strace -p <PID>
```

Possible calls include:

```text
futex(...)
```

which can indicate synchronization/waiting.

```text
epoll_wait(...)
```

which can indicate waiting for I/O events.

```text
poll(...)
```

which can indicate waiting on file descriptors/events.

```text
connect(...)
```

which can indicate network activity.

`strace` therefore helps answer:

> What is this process waiting for at the operating-system level?

It does not, by itself, explain the complete Java-level reason.

---

# 20. Java Performance Investigation

Suppose:

```bash
pgrep -af java
```

returns:

```text
12345 java -jar daily-sync-hub.jar
```

Start with:

```bash
sudo strace -c -p 12345
```

Then, if file I/O is suspected:

```bash
sudo strace -f -e trace=file -p 12345
```

For read/write activity:

```bash
sudo strace -p 12345 -e trace=read,write,pread64,pwrite64
```

Check process-level I/O:

```bash
pidstat -d -p 12345 1
```

Check storage-device activity:

```bash
iostat -xz 1
```

Use JFR/JMC for Java-level profiling.

---

# 21. Is My Java Program I/O-Bound or Disk-Bound?

Do not conclude:

> "My program uses `read()`, so it is disk-bound."

Instead, collect evidence at several levels:

```text
Java application
       |
       v
    JFR/JMC
       |
       | What are Java threads doing?
       v
     strace
       |
       | What system calls are occurring?
       v
    pidstat
       |
       | How much I/O is the process generating?
       v
    iostat
       |
       | Is the storage device actually busy?
       v
Storage device
```

A stronger disk-bound diagnosis would have evidence such as:

```text
CPU utilization     → relatively low
Process disk I/O    → high
Disk utilization    → high
I/O latency         → high
Application latency → high
```

Then you can reasonably say storage is a bottleneck.

---

# 22. `strace` vs `pidstat` vs `iostat` vs JFR

These tools answer different questions.

## `strace`

```bash
sudo strace -p <PID>
```

Question:

> What system calls is this process making?

---

## `pidstat`

```bash
pidstat -d -p <PID> 1
```

Question:

> How much I/O activity is this process generating?

---

## `iostat`

```bash
iostat -xz 1
```

Question:

> What is happening on the storage device?

---

## JFR/JMC

Question:

> What is the Java application doing internally?

Useful for:

- Java threads
- CPU hotspots
- Garbage collection
- Locks
- Java I/O
- Thread states
- Application behavior

---

# 23. `strace` and Database Applications

Suppose:

```text
Java
 |
 +-- MongoDB
 +-- Redis
 +-- Oracle/MySQL/PostgreSQL
 +-- filesystem
 +-- external HTTP APIs
```

`strace` can show network system calls:

```bash
sudo strace -f -e trace=network -p <PID>
```

But it generally cannot tell you:

> "This SQL query took 300 ms."

For that, use database monitoring, JDBC/application metrics, APM, JFR, or database-specific tools.

`strace` operates at the operating-system system-call level.

---

# 24. `strace` and Docker

You can use `strace` when troubleshooting a process inside a Docker container if the required permissions and tools are available.

For example:

```bash
docker exec -it <container> ps
```

Then:

```bash
docker exec -it <container> strace -p <PID>
```

Attaching to another process may require additional privileges depending on the container security configuration.

Do not automatically use privileged containers in production just to run `strace`; understand the security implications first.

---

# 25. Practical Java Troubleshooting Workflow

### Step 1 — Find the Java process

```bash
pgrep -af java
```

### Step 2 — Check CPU

```bash
top -p <PID>
```

or:

```bash
pidstat -p <PID> 1
```

### Step 3 — Check process disk I/O

```bash
pidstat -d -p <PID> 1
```

### Step 4 — Check storage

```bash
iostat -xz 1
```

### Step 5 — Check system calls

```bash
sudo strace -c -p <PID>
```

### Step 6 — If file activity is suspected

```bash
sudo strace -f -e trace=file -p <PID>
```

### Step 7 — If network activity is suspected

```bash
sudo strace -f -e trace=network -p <PID>
```

### Step 8 — Use Java profiling

Use JFR/JMC to identify Java-level hotspots and thread states.

---

# 26. Most Important Commands to Memorize

### Trace a command

```bash
strace command
```

### Attach to a running process

```bash
sudo strace -p PID
```

### Follow child processes/threads

```bash
strace -f ...
```

### Get a summary

```bash
sudo strace -c -p PID
```

### Save output

```bash
strace -o trace.log ...
```

### Trace files

```bash
sudo strace -f -e trace=file -p PID
```

### Trace network

```bash
sudo strace -f -e trace=network -p PID
```

### Trace reads/writes

```bash
sudo strace -e trace=read,write command
```

---

# 27. Quick Cheat Sheet

| Goal | Command |
|---|---|
| Trace a command | `strace command` |
| Attach to process | `sudo strace -p PID` |
| Follow children/threads | `strace -f ...` |
| Summary | `strace -c ...` |
| Save output | `strace -o trace.log ...` |
| File operations | `-e trace=file` |
| Network operations | `-e trace=network` |
| Process operations | `-e trace=process` |
| Read/write | `-e trace=read,write` |
| Process disk I/O | `pidstat -d -p PID 1` |
| Storage device | `iostat -xz 1` |
| Socket information | `ss -tulpn` |

---

# 28. Mental Model

Remember:

```text
                 Application
                      |
                      v
                     JVM
                      |
                      v
               System calls
                      |
          +-----------+-----------+
          |           |           |
         File       Network     Process
          |           |           |
          v           v           v
       Kernel       Kernel      Kernel
          |
          v
      Hardware/
      filesystem
```

`strace` watches this boundary:

```text
Application → System Calls → Linux Kernel
                    ^
                    |
                  strace
```

The key question it answers is:

> **"What is this process actually asking Linux to do?"**

---

# 29. Final Summary

`strace` is primarily a **system-call and signal tracing tool**.

It is useful for discovering:

- What files a process accesses
- What it reads and writes
- What network operations it performs
- What processes it creates
- What errors system calls return
- What a process is waiting for
- What signals it receives
- Which system calls dominate system-call activity

For Java troubleshooting:

```bash
# Find Java process
pgrep -af java

# Attach
sudo strace -p <PID>

# Summary
sudo strace -c -p <PID>

# File activity
sudo strace -f -e trace=file -p <PID>

# Network activity
sudo strace -f -e trace=network -p <PID>

# Process I/O
pidstat -d -p <PID> 1

# Storage device
iostat -xz 1
```

Remember the distinction:

```text
strace
→ What system calls does the process make?

pidstat
→ How much CPU/I/O activity does the process generate?

iostat
→ What is happening on the storage device?

JFR/JMC
→ What is the Java application doing internally?
```

Use these tools together when diagnosing performance rather than relying on `strace` alone.
