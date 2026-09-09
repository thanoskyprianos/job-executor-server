# Job Executor Server & Job Commander

A multithreaded client–server system for remotely submitting, queueing and executing
shell jobs. A central **server** maintains a bounded queue of jobs and a pool of worker
threads that run them (each job in its own forked child process), while any number of
lightweight **commander** clients connect over TCP sockets to issue jobs and manage the
server.

* **Author:** Athanasios Kyprianos (1115202200082)
* **Course project:** K24 – HW2 (Systems Programming 2024)
* **Language:** C/C++ (C++98-compatible, pthreads)

---

## Table of Contents

1. [Overview](#overview)
2. [Features](#features)
3. [Requirements](#requirements)
4. [Build](#build)
5. [Usage](#usage)
6. [Commands](#commands)
7. [Architecture](#architecture)
8. [Wire Protocol](#wire-protocol)
9. [Concurrency & Synchronization](#concurrency--synchronization)
10. [Project Structure](#project-structure)
11. [Exit Codes](#exit-codes)

---

## Overview

```
                 TCP sockets                     fork + execvp
 jobCommander ───────────────► jobExecutorServer ──────────────► /bin/prog arg1 arg2 ...
 jobCommander ───────────────►   ┌───────────────┐
 jobCommander ───────────────►   │  bounded queue│────► worker pool
        ...                      └───────────────┘
```

The server listens on a TCP port and for every incoming commander connection spawns a
dedicated **commander thread** that reads one request, handles it and replies. Jobs are
placed into a bounded buffer (implemented as a `std::map`, FIFO by increasing job ID)
and picked up by **worker threads**. Each worker `fork()`s and `execvp()`s the job,
redirecting the child's stdout to a temporary file so the output can be streamed back
to the commander that submitted the job.

A single `jobCommander` invocation sends exactly one command and exits, so multiple
commanders can drive the server concurrently (e.g. from shell scripts).

## Features

- TCP-based client/server communication with a simple binary protocol
- Bounded job buffer with blocking on both producers and consumers
- Configurable worker thread pool and concurrency level
- Per-connection commander threads; every job's output routed back to the correct client
- Job introspection: poll queued jobs, stop a queued job by ID
- Runtime concurrency adjustment via `setConcurrency`
- Graceful shutdown: `exit` waits for running jobs to finish before terminating
- Clean signal handling (`Ctrl+C` / `SIGUSR1`) with proper mutex/condvar destruction

## Requirements

- Linux (uses POSIX sockets, pthreads, `fork`/`execvp`)
- `g++` and `make`
- Linker flag `-lpthread`

## Build

```bash
make            # or: make all      — builds both binaries into ./bin/
make clean      # removes binaries and object files
make help       # prints the usage cheat sheet
```

Output binaries:

| Binary               | Role        |
| -------------------- | ----------- |
| `./bin/jobExecutorServer` | The server |
| `./bin/jobCommander`     | Client CLI |

## Usage

Start the server first:

```bash
./bin/jobExecutorServer [portNum] [bufferSize] [threadPoolSize]
```

| Argument        | Meaning                                        |
| --------------- | ---------------------------------------------- |
| `portNum`       | TCP port the server listens on                 |
| `bufferSize`    | Maximum number of jobs waiting in the buffer   |
| `threadPoolSize`| Number of worker threads                       |

Then use the commander from any host:

```bash
./bin/jobCommander [serverName] [portNum] [jobCommanderInputCommand]
```

### Examples

```bash
# server on localhost, buffer of 10 jobs, 4 worker threads
./bin/jobExecutorServer 8080 10 4

# submit jobs
./bin/jobCommander localhost 8080 issueJob ls -la /tmp
./bin/jobCommander localhost 8080 issueJob /bin/sleep 20
./bin/jobCommander localhost 8080 issueJob touch /tmp/hello

# inspect the queue
./bin/jobCommander localhost 8080 poll
# JOB <job_1, ls -la /tmp>
# JOB <job_2, /bin/sleep 20>
# ...

# remove a queued job
./bin/jobCommander localhost 8080 stop job_2

# change how many jobs may run at once
./bin/jobCommander localhost 8080 setConcurrency 2

# shut the server down (after running jobs finish)
./bin/jobCommander localhost 8080 exit
```

## Commands

### Commander

| Command              | Arguments           | Description                                        |
| -------------------- | ------------------- | -------------------------------------------------- |
| `issueJob`           | `<job> [args...]`   | Submit a job (executable + arguments)              |
| `setConcurrency`     | `<N>`               | Set server concurrency level to `N`                |
| `stop`               | `job_<id>`          | Remove a queued (not running) job from the buffer  |
| `poll`               | –                   | Print all queued jobs                              |
| `exit`               | –                   | Terminate server after running jobs finish         |

### Server responses

| Command            | Typical response                              |
| ------------------ | --------------------------------------------- |
| `issueJob`         | `JOB <job_1, args> SUBMITTED` + job output    |
| `setConcurrency`   | `CONCURRENCY SET AT N`                        |
| `stop`             | `JOB <job_N> REMOVED` or `JOB <job_N> NOTFOUND` |
| `poll`             | one `JOB <job_N, args>` triplet per line      |
| `exit`             | `SERVER TERMINATED`                           |

When a queued job is discarded (stopped, or server shuts down), the commander that
submitted it receives `JOB STOPPED BEFORE EXECUTION` / `SERVER TERMINATED BEFORE
EXECUTION` on its socket.

## Architecture

The code is organized around namespaces, one per responsibility:

```
jobExecutorServer
│
├── main thread (server.cpp)
│     ├── listens/accepts TCP connections
│     ├── spawns threadPool worker threads
│     └── spawns one detached commander thread per connection
│
├── commander thread ──► Fetcher::headers() reads the command code
│     └── dispatches to Fetcher::issueJob() / setConcurrency() / stop()
│     └── calls Executor::* which use Respondent::* to reply
│
├── worker thread ──► waits on Cond::runtimeWorker
│     ├── Executor::next() pops the FIFO job from the buffer
│     ├── fork() + execvp() with stdout → /tmp/<pid>.output
│     └── Respondent::jobOutput() streams the file back to the client
│
└── Executor::internal  (shared state, guarded by Mutex::runtime / Mutex::concurrency)
      running, runningJobs, incJobId, concurrencyLevel, bufferSize, jobsBuffer
```

### The Job buffer

`Executor::internal::jobsBuffer` is a `std::map<uint32_t, Job*>` keyed by an
ever-increasing job ID. Because keys only grow, iterating from `begin()` yields FIFO
order, while lookups for `stop`/`poll` are O(log n).

### Shared state (`Executor::internal`)

| Variable           | Purpose                                                        |
| ------------------ | -------------------------------------------------------------- |
| `running`          | Server shutdown flag                                           |
| `runningJobs`      | Jobs currently executing (bounded by `concurrencyLevel`)       |
| `incJobId`         | Monotonic ID source assigned on buffer insertion               |
| `concurrencyLevel` | Max jobs running at once (adjustable at runtime)               |
| `bufferSize`       | Buffer capacity (fixed at startup)                             |
| `jobsBuffer`       | The job queue described above                                  |

## Wire Protocol

All multi-byte integers are sent in network byte order (big-endian, `htonl`/`ntohl`).
The commander sends a request, then calls `shutdown(sock, SHUT_WR)` to signal EOF.
The server responds with zero or more length-prefixed messages:

```
[uint32 len][payload bytes]...
```

### Requests

**`issueJob`** — code `10`

```
10
len(arg_1) arg_1
len(arg_2) arg_2
...
(EOF via SHUT_WR)
```

**`setConcurrency`** — code `11`

```
11
N (uint32_t)
(EOF)
```

**`stop`** — code `12`

```
12
jobId (uint32_t)
(EOF)
```

**`poll`** — code `13` / **`exit`** — code `14`

```
13 | 14
(EOF)
```

Command codes are defined in `include/codes.hpp`.

## Concurrency & Synchronization

Two mutexes and two condition variables coordinate the threads
(`include/server/sync.hpp`, `src/server/sync.cpp`):

| Primitive              | Protects                                             |
| ---------------------- | ---------------------------------------------------- |
| `Mutex::runtime`       | `jobsBuffer`, `running`, job submission/stopping, polling |
| `Mutex::concurrency`   | `runningJobs`, `concurrencyLevel`                    |
| `Cond::runtimeWorker`  | Wakes workers when a job enters the buffer           |
| `Cond::runtimeCommander` | Wakes commanders when buffer space frees up        |

- **Commanders** lock `runtime` to insert a job (blocking on `runtimeCommander` while
  the buffer is full) or to stop/poll jobs.
- **Workers** wait on `runtimeWorker` while the buffer is empty; when a job arrives
  they re-check `concurrency` and pop the FIFO head, then broadcast
  `runtimeCommander` to announce free buffer space.
- **Exit:** the commander thread handling `exit` sets `running = false`, broadcasts
  both condition variables, and delivers `SIGUSR1` to the main thread. The signal
  handler joins all workers, destroys the synchronization primitives, notifies the
  commanders of still-queued jobs, replies `SERVER TERMINATED` to the exit caller and
  cleans up.

## Project Structure

```
.
├── Makefile                  # build, clean, help targets
├── help                      # usage cheat sheet printed by `make help`
├── README.md                 # this file
├── include/
│   ├── codes.hpp             # shared error & command codes
│   ├── commander/
│   │   └── requester.hpp     # client-side request sender interface
│   └── server/
│       ├── executor.hpp      # Job struct + shared buffer state
│       ├── fetcher.hpp       # server-side request parser interface
│       ├── respondent.hpp    # server-side response writer interface
│       └── sync.hpp          # mutexes & condition variables
├── src/
│   ├── commander/
│   │   ├── commander.cpp     # CLI entry point, socket setup
│   │   └── requester.cpp     # request encoding/writing
│   └── server/
│       ├── server.cpp        # main, accept loop, worker/commander routines, signals
│       ├── executor.cpp      # buffer operations (issue/stop/next), Job
│       ├── fetcher.cpp       # request decoding/reading
│       ├── respondent.cpp    # response encoding/writing, output streaming
│       └── sync.cpp          # mutex/condvar definitions
├── syspro2024_hw2_completion_report.pdf   # assignment completion report
├── bin/                      # (created by make) executables
└── build/                    # (created by make) object files
```

## Exit Codes

Error codes shared by both binaries (`include/codes.hpp`):

| Code | Constant        | Meaning                          |
| ---- | --------------- | -------------------------------- |
| 1    | `USAGE_ERROR`   | Invalid command-line usage       |
| 2    | `HOSTNAME_ERROR`| Hostname resolution failure      |
| 3    | `VALUE_ERROR`   | Invalid numeric argument         |
| 4    | `NETWORK_ERROR` | Network failure                  |
| 5    | `PROCCESS_ERROR`| Read/write (processing) failure  |
| 6    | `THREAD_ERROR`  | Thread creation/join failure     |
| 7    | `SOCKET_ERROR`  | Socket operation failure         |
| 8    | `EXEC_ERROR`    | `fork`/`exec` failure            |

Codes `10`–`15` are the protocol command codes used by the wire protocol above.
