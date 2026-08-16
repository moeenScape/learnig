# Linux `ss` Command — Practical Guide

## 1. What is `ss`?

`ss` is a Linux command used to inspect **network sockets**.

It can show:

- Listening ports
- Active TCP connections
- UDP sockets
- Which process is using a socket
- TCP connection states
- Socket statistics
- Network information such as addresses and ports

The manual describes `ss` as a utility for dumping socket statistics and notes that it can provide more TCP and state information than `netstat`.

---

## 2. Basic Syntax

```bash
ss [options] [FILTER]
```

Think of the command as:

```text
ss
│
├── options  → what information/protocol to show
│
└── FILTER   → which connections/sockets to show
```

For example:

```bash
ss -t
```

means:

> Show TCP sockets.

And:

```bash
ss -tln
```

means:

> Show TCP listening sockets using numeric addresses/ports.

---

# 3. The Most Important Options

## `-t` — TCP

```bash
ss -t
```

Shows TCP sockets.

Use it when you are interested in TCP connections.

---

## `-u` — UDP

```bash
ss -u
```

Shows UDP sockets.

---

## `-l` — Listening

```bash
ss -l
```

Shows only sockets that are listening.

A listening socket means a program is waiting for incoming connections.

For example:

```text
LISTEN 0 128 0.0.0.0:8080 0.0.0.0:*
```

means something is listening on port `8080`.

---

## `-a` — All

```bash
ss -a
```

Shows both listening and non-listening sockets.

For TCP, non-listening sockets commonly include established connections.

---

## `-n` — Numeric

```bash
ss -n
```

Prevents `ss` from trying to resolve names.

For example, instead of displaying:

```text
http
```

it displays:

```text
80
```

This is usually useful when troubleshooting because it makes the output faster and more predictable.

---

## `-p` — Processes

```bash
sudo ss -p
```

Shows the processes using the sockets.

This is very useful when you want to know:

> Which program is using port 8080?

Example:

```text
users:(("nginx",pid=1234,fd=6))
```

This tells you that `nginx` is using the socket.

---

# 4. The Most Useful Command

For everyday Linux troubleshooting, remember:

```bash
sudo ss -tulpn
```

Break it down:

```text
-t   TCP
-u   UDP
-l   Listening
-p   Process
-n   Numeric
```

So:

```bash
sudo ss -tulpn
```

means:

> Show TCP and UDP listening ports, their numeric addresses/ports, and the processes using them.

This is one of the best commands for answering:

> What ports are currently open/listening on this Linux machine?

---

# 5. Checking a Specific Port

Suppose you want to check port `8080`.

```bash
sudo ss -tulpn | grep :8080
```

For port `5000`:

```bash
sudo ss -tulpn | grep :5000
```

For multiple ports:

```bash
sudo ss -tulpn | grep -E ':(8080|5000|27017|6379)'
```

For your Docker application, these could correspond to:

```text
8080  → frontend
5000  → backend
27017 → MongoDB
6379  → Redis
```

---

# 6. Understanding Listening Addresses

You may see:

```text
0.0.0.0:8080
```

or:

```text
127.0.0.1:8080
```

or:

```text
[::]:8080
```

### `0.0.0.0:8080`

The service is listening on all IPv4 interfaces.

It may be reachable through the machine's network interfaces, depending on firewall/network configuration.

### `127.0.0.1:8080`

The service is listening only on localhost.

Other machines normally cannot connect directly to it.

### `[::]:8080`

The service is listening on IPv6 interfaces.

---

# 7. TCP vs UDP

Use:

```bash
ss -t
```

for TCP.

Use:

```bash
ss -u
```

for UDP.

Use:

```bash
ss -tu
```

for both.

To see listening TCP and UDP sockets:

```bash
ss -tul
```

For a more useful version:

```bash
sudo ss -tulpn
```

---

# 8. Connection States

TCP connections have different states.

The manual lists states including:

```text
established
syn-sent
syn-recv
fin-wait-1
fin-wait-2
time-wait
closed
close-wait
last-ack
listening
closing
```

The most important ones to understand first are:

## LISTEN

The application is waiting for incoming connections.

```text
LISTEN
```

Example:

```text
0.0.0.0:8080
```

A web server commonly has a listening socket.

---

## ESTABLISHED

A TCP connection is currently established between two endpoints.

```text
ESTAB
```

Example conceptually:

```text
client:54321  <---- TCP ---->  server:5000
```

---

## TIME-WAIT

A TCP connection has been closed, but the system keeps some state temporarily.

Seeing some `TIME-WAIT` connections is normal.

---

## CLOSE-WAIT

The remote side has closed the connection, and the local application has not completely closed its side yet.

A large number of persistent `CLOSE-WAIT` connections can indicate an application connection-management problem.

---

# 9. Show Established Connections

```bash
ss -tn
```

Meaning:

```text
-t → TCP
-n → numeric
```

You can specifically ask for established connections:

```bash
ss -tn state established
```

---

# 10. Filter by Port

The manual supports filters such as:

```bash
sport = :PORT
```

and:

```bash
dport = :PORT
```

### Source port

```bash
ss -tn 'sport = :5000'
```

### Destination port

```bash
ss -tn 'dport = :5000'
```

You can also use:

```bash
ss -tn '( sport = :5000 or dport = :5000 )'
```

This finds TCP connections where either the source or destination port is `5000`.

---

# 11. Filter by Connection State

Show established TCP connections:

```bash
ss -tn state established
```

Show listening TCP sockets:

```bash
ss -tn state listening
```

Show TIME-WAIT connections:

```bash
ss -tn state time-wait
```

Show CLOSE-WAIT connections:

```bash
ss -tn state close-wait
```

---

# 12. Useful Real-World Commands

## Find listening ports

```bash
sudo ss -tulpn
```

---

## Find who is using port 8080

```bash
sudo ss -tulpn | grep :8080
```

---

## Find who is using port 5000

```bash
sudo ss -tulpn | grep :5000
```

---

## Show all TCP connections

```bash
ss -t -a
```

The manual gives this as an example for displaying all TCP sockets.

---

## Show all UDP sockets

```bash
ss -u -a
```

---

## Show a summary

```bash
ss -s
```

This prints summary statistics instead of parsing a large socket list.

---

## Show processes

```bash
sudo ss -tulpn
```

The `-p` option shows processes using sockets.

---

# 13. `ss` Compared with `netstat`

`ss` can provide information similar to `netstat`.

A common modern replacement for:

```bash
netstat -tulpn
```

is:

```bash
ss -tulpn
```

The `ss` manual specifically describes it as providing information similar to `netstat` and more TCP/state information.

---

# 14. Understanding a Typical Output

Suppose you run:

```bash
sudo ss -tulpn
```

and get:

```text
Netid State  Local Address:Port  Peer Address:Port
tcp   LISTEN 0.0.0.0:8080       0.0.0.0:*       users:(("nginx",pid=1234))
```

Read it as:

```text
tcp
 ↓
TCP socket

LISTEN
 ↓
Waiting for connections

0.0.0.0:8080
 ↓
Listening on port 8080 on IPv4 interfaces

nginx
 ↓
Process using the socket
```

---

# 15. Important Options from the Manual

| Option | Meaning |
|---|---|
| `-a` | All sockets |
| `-l` | Listening sockets |
| `-n` | Numeric addresses/ports |
| `-p` | Processes using sockets |
| `-t` | TCP |
| `-u` | UDP |
| `-s` | Summary statistics |
| `-o` | Timer information |
| `-e` | Extended information |
| `-m` | Socket memory usage |
| `-i` | Internal TCP information |
| `-4` | IPv4 only |
| `-6` | IPv6 only |
| `-x` | Unix domain sockets |
| `-K` | Attempt to forcibly close supported sockets |

---

# 16. Options You Should Learn First

Don't try to memorize the entire manual.

Learn these first:

```bash
ss -t
ss -u
ss -l
ss -a
ss -n
ss -p
```

Then combine them:

```bash
sudo ss -tulpn
```

After that, learn filtering:

```bash
ss -tn state established
```

and:

```bash
ss -tn 'dport = :5000'
```

---

# 17. Quick Cheat Sheet

### What ports are listening?

```bash
sudo ss -tulpn
```

### Is port 8080 listening?

```bash
sudo ss -tulpn | grep :8080
```

### Is port 5000 listening?

```bash
sudo ss -tulpn | grep :5000
```

### Show TCP connections

```bash
ss -tn
```

### Show established TCP connections

```bash
ss -tn state established
```

### Show all TCP sockets

```bash
ss -ta
```

### Show all UDP sockets

```bash
ss -ua
```

### Show socket summary

```bash
ss -s
```

### Show processes

```bash
sudo ss -tulpn
```

---

# 18. A Simple Mental Model

Think about a server running a web application:

```text
                    Linux Server
                         |
             +-----------+-----------+
             |                       |
         Port 8080                Port 5000
             |                       |
           nginx                  Backend
             |                       |
          Frontend                  API
```

Run:

```bash
sudo ss -tulpn
```

and `ss` helps you answer:

```text
What ports are listening?
        ↓
Which IP are they listening on?
        ↓
TCP or UDP?
        ↓
Which process owns the socket?
        ↓
What connections are currently active?
```

That is the core purpose of `ss`.

---

# 19. Recommended Learning Order

Learn in this order:

1. `ss -t`
2. `ss -u`
3. `ss -l`
4. `ss -n`
5. `ss -p`
6. `ss -tulpn`
7. TCP states
8. `state established`
9. `sport` / `dport`
10. IP/address filtering
11. `-o`, `-i`, `-m` for advanced troubleshooting

You do **not** need to memorize all 575 lines of the manual. The goal is to understand the concepts and know which option to use when troubleshooting a network problem.

## Based on the original manual

The source document defines `ss` as a socket-statistics utility, documents the command syntax, options, state filters, expressions, host syntax, and usage examples. The guide above reorganizes those concepts into a beginner-friendly learning path rather than reproducing the manual. 
