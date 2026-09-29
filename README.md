<div align="center">

# kvstore

**A high-performance, Redis-style in-memory key-value store in C++ — single-threaded, event-driven, and speaking the Redis wire protocol.**

[![C++](https://img.shields.io/badge/C%2B%2B-17-00599C?logo=cplusplus&logoColor=white)](https://isocpp.org/)
[![Platform](https://img.shields.io/badge/platform-Linux-333333?logo=linux&logoColor=white)](https://www.kernel.org/)
[![I/O](https://img.shields.io/badge/I%2FO-poll()%20multiplexing-4c1)](https://man7.org/linux/man-pages/man2/poll.2.html)
[![Protocol](https://img.shields.io/badge/protocol-RESP-DC382D?logo=redis&logoColor=white)](https://redis.io/docs/reference/protocol-spec/)

</div>

---

## Table of contents

- [Overview](#overview)
- [Why event-driven?](#why-event-driven)
- [Architecture](#architecture)
- [How it works](#how-it-works)
  - [The event loop](#the-event-loop)
  - [RESP protocol parsing](#resp-protocol-parsing)
  - [Pipelining](#pipelining)
  - [Storage](#storage)
- [Supported commands](#supported-commands)
- [Getting started](#getting-started)
- [Usage](#usage)
- [Benchmarks](#benchmarks)
- [Project structure](#project-structure)
- [Design decisions](#design-decisions)

---

## Overview

`kvstore` is an in-memory key-value store written from scratch in C++, modeled on the
core of Redis. It accepts connections over TCP, speaks the **RESP** wire protocol (so
standard clients like `redis-cli` and `redis-benchmark` work against it unmodified),
and serves `SET` / `GET` from an in-memory hash map.

Its defining property is its concurrency model: a **single thread** handles thousands
of concurrent connections through a `poll()`-based event loop and non-blocking sockets
— no thread per client, no locks on the data path.

This is a systems-programming project focused on the parts that make an in-memory store
fast: the I/O model, the protocol parser, and request pipelining.

## Why event-driven?

The naïve way to serve many clients is one thread (or process) per connection. That
model is simple but scales poorly: thousands of threads mean thousands of stacks,
heavy context-switching, and synchronization overhead on shared state.

`kvstore` takes the approach Redis itself uses — **I/O multiplexing on a single
thread**:

- One thread monitors *all* sockets at once with `poll()` and wakes only for those
  ready to read or write.
- Sockets are **non-blocking**, so the loop never stalls on a slow client.
- Because a single thread owns the data, `SET`/`GET` need **no locking** — the hot path
  is a plain hash-map lookup.

The result is high throughput with a small, predictable resource footprint.

## Architecture

```
                 ┌──────────────────────────────────────────────┐
                 │                 Event loop                     │
   clients ────▶ │   poll()  ──▶  ready fds  ──▶  handle events   │
   (TCP)         │     ▲                              │           │
                 │     │                              ▼           │
                 │  register/                  ┌──────────────┐   │
                 │  update fds                 │ per-conn I/O │   │
                 │     │                        │ read buffer  │   │
                 │     └────────────────────────│ write buffer │   │
                 │                              └──────┬───────┘   │
                 │                                     ▼           │
                 │                          ┌────────────────────┐ │
                 │                          │  RESP parser        │ │
                 │                          │  (may yield 0..N    │ │
                 │                          │   complete commands)│ │
                 │                          └─────────┬──────────┘ │
                 │                                    ▼            │
                 │                          ┌────────────────────┐ │
                 │                          │  command dispatch   │ │
                 │                          │  SET / GET / ...     │ │
                 │                          └─────────┬──────────┘ │
                 │                                    ▼            │
                 │                          ┌────────────────────┐ │
                 │                          │  hash map storage   │ │
                 │                          │  unordered_map      │ │
                 │                          └────────────────────┘ │
                 └──────────────────────────────────────────────┘
```

Each connection carries its own read and write buffers. The loop reads whatever the
kernel has available, hands the read buffer to the RESP parser, executes every
**complete** command the parser finds, and queues replies into the write buffer to be
flushed when the socket is writable.

## How it works

### The event loop

The server keeps a set of file descriptors — the listening socket plus every client
socket — and calls `poll()` to wait for readiness across all of them at once.

- **Listening socket readable** → `accept()` new connections, set them non-blocking,
  and add them to the poll set.
- **Client socket readable** → read available bytes into that connection's read buffer
  and parse.
- **Client socket writable** → flush pending bytes from the write buffer.

Nothing blocks: a client that sends half a command or reads slowly holds up only its
own buffers, never the loop.

### RESP protocol parsing

Clients send commands encoded in **RESP** (the Redis Serialization Protocol) — arrays
of bulk strings. `kvstore` implements a RESP parser so it is wire-compatible with the
standard Redis tooling.

Critically, the parser is **incremental**. A single `read()` may contain a partial
command, exactly one command, or several commands. The parser:

- consumes as many **complete** commands as the buffer holds,
- leaves any trailing **partial** command in the buffer until more bytes arrive,

so a command split across two reads is handled correctly, and a read containing many
commands is fully drained.

### Pipelining

Because the parser extracts *all* complete commands from a single read, `kvstore`
naturally supports **pipelining** — a client can send many commands back-to-back
without waiting for each reply. Instead of assuming one socket read maps to one
command, the server processes the whole batch and writes the batched replies.

This is where the throughput comes from: pipelining amortizes syscall and round-trip
overhead across many operations, which is exactly what `redis-benchmark -P 10`
exercises.

### Storage

Keys and values live in an in-memory `std::unordered_map<std::string, std::string>`.
Since one thread owns the map, reads and writes are lock-free and operate at hash-map
speed.

## Supported commands

| Command             | Description                        |
|---------------------|------------------------------------|
| `SET key value`     | Store a value for a key            |
| `GET key`           | Retrieve the value for a key       |
| `DEL key`           | Delete a key *(if implemented)*    |
| `PING`              | Health check → `PONG`              |

> Commands speak RESP, so `redis-cli` and `redis-benchmark` work out of the box.

## Getting started

### Prerequisites

- A Linux environment (native or WSL)
- A C++17 compiler (`g++` or `clang++`)
- `cmake` and `make`
- `redis-tools` (for `redis-cli` / `redis-benchmark`)

### Build

```bash
git clone https://github.com/<you>/kvstore.git
cd kvstore

cmake -B build -DCMAKE_BUILD_TYPE=Release
cmake --build build
```

### Run

```bash
./build/kvstore --port 6380
```

## Usage

Connect with the standard Redis CLI:

```bash
redis-cli -p 6380
127.0.0.1:6380> SET hello world
OK
127.0.0.1:6380> GET hello
"world"
```

## Benchmarks

Throughput is measured with the standard `redis-benchmark` tool, using pipelining to
exercise the batched-command path:

```bash
redis-benchmark -p 6380 -t set,get -n 1000000 -P 10
```

| Workload            | Pipelining | Throughput            |
|---------------------|------------|-----------------------|
| `SET` / `GET`       | `-P 10`    | ~400K requests/sec    |

> Measured locally on `<your environment — CPU / OS / WSL, fill in>`. Numbers are
> single-node, loopback, and reflect the effect of pipelining; without `-P` the figure
> is bounded by round-trip latency rather than server throughput.

## Project structure

```
/src
  main.cpp            # entry point, arg parsing, server bootstrap
  server.cpp/.h       # event loop: poll(), accept, read/write dispatch
  connection.cpp/.h   # per-connection read/write buffers and state
  resp.cpp/.h         # incremental RESP parser
  commands.cpp/.h     # command dispatch (SET / GET / ...)
  store.cpp/.h        # in-memory hash map storage
CMakeLists.txt
```

## Design decisions and limitations

**Decisions**

- **Single-threaded event loop** over thread-per-client — trades multi-core use for a
  simple, lock-free, cache-friendly hot path, matching Redis's own design.
- **RESP compatibility** — reusing the Redis protocol means the entire Redis client and
  benchmarking ecosystem works against this server for free.
- **Incremental parsing** — correctness under partial reads and pipelined batches is
  handled explicitly rather than assumed.

## License

Released under the MIT License. See [`LICENSE`](LICENSE).
