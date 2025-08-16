#  Multi-Threaded HTTP Web Server in C

A lightweight **HTTP Web Server** written in **C** that handles **multiple clients concurrently** using **POSIX threads (pthreads)**. It parses basic HTTP `GET` requests and serves simple HTML pages. Built as part of an **Operating Systems / Computer Networks** project.

Features
- Multi-threaded client handling with **pthreads**
- Parses basic **HTTP GET** requests
- Serves HTML routes:
  - `/` or `/index` → Home
  - `/about` → About
  - anything else → 404
- **Thread-safe logging** using a mutex (writes to `log/log.txt`)
- **Graceful shutdown** with `SIGINT` (Ctrl+C)

Tech Stack
- C (GCC), POSIX Sockets API
- Pthreads, Mutex
- File I/O, Signal handling


