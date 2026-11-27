# Step 5: Many clients at once

## Goal

Fifty clients connected at the same time, all served by one thread. `redis-benchmark` runs against `anvil`.

This is the hardest step and the core idea of Redis. Expect it to take several sessions.

## Why this step

In step 3 the server is stuck inside `read` waiting for one client while everyone else waits. There are two fixes:

- **One thread per client.** Simple to write, but every thread touches the same dictionary, so you need locks, and mistakes become race conditions.
- **One thread, never wait.** Ask the operating system which sockets are ready, handle only those, repeat. No locks, because only one thing ever runs at a time. Redis does this. So will you.

## Concepts

- **Blocking.** A normal `read` sleeps until data arrives. That sleep is the whole problem.
- **Non-blocking socket.** Set with `fcntl` and `O_NONBLOCK`. Now `read` returns immediately. If there is no data it returns `-1` with `errno` set to `EAGAIN`. That is not an error; it means "nothing right now".
- **`poll`.** You hand it a list of descriptors and what you care about for each (readable, writable). It sleeps until at least one is ready and tells you which.
- **Event loop.** Forever: call `poll`, handle each ready descriptor, go round again.
- **Per-connection state.** Each client needs its own descriptor, read buffer (bytes received, not yet parsed), and write buffer (reply bytes not yet sent).
- **Writes can also be "not ready".** If a client is slow to receive, `write` sends only part or returns `EAGAIN`. Keep the rest in the write buffer and ask `poll` to tell you when that socket is writable.

## The loop, in words

1. Build the list for `poll`: the listening socket (interested in readable), plus every client (readable, and also writable if its write buffer is not empty).
2. Call `poll`.
3. Listening socket readable: `accept` new clients until it returns `EAGAIN`. Make each non-blocking. Create its state.
4. Client readable: `read` into its read buffer. Parse and execute every complete command. Append each reply to its write buffer.
5. Client writable: `write` as much of the write buffer as it takes. Remove what was sent.
6. Client closed or errored: `close` it and delete its state.

## Tasks

Build it in this order. Keep it working after each one.

1. A struct for per-connection state. Move the step-3 buffer into it.
2. Make the listening socket non-blocking. Loop on `poll` with only that socket. Accept clients and print a line; do not serve them yet.
3. Add clients to the `poll` list. Handle readable: read, parse, execute. For now, write replies directly as in step 3.
4. Add the write buffer. Replies go into it. Send only when `poll` says writable.
5. Handle disconnects and errors. No leaked descriptors, no stale state.
6. Add `DBSIZE` (number of keys). You will want it for testing later.

## Check it works

- Two `redis-cli` sessions at once, both responsive.
- An idle `nc localhost 6379` left open does not slow anyone else.
- `redis-benchmark -t set,get -n 100000 -c 50 -q` completes. It may print a warning that `CONFIG` failed; that is fine.
- Pipelining: add `-P 16` to the command above.
- Run the benchmark and Ctrl-C it halfway. The server keeps running and has no connections left open.
- Run the same benchmark against real Redis and compare the numbers.

## Pitfalls

- Treating `EAGAIN` as a failure and closing the connection.
- Asking for "writable" when you have nothing to write. A socket is almost always writable, so `poll` returns instantly forever and your CPU sits at 100%.
- Removing a connection from a container while looping over that container.
- Set `O_NONBLOCK` on every accepted socket explicitly. Whether it is inherited from the listening socket differs between operating systems.
- Measure with a release build (`-O2`, no sanitizers). Sanitizers make it several times slower.

## Questions to answer before moving on

- Why is no lock needed around the dictionary?
- One command takes a full second to run. What happens to the other 49 clients? What does that imply about which commands a Redis-like server can afford?
- Why do you only ask for "writable" when the write buffer has bytes in it?
- `poll` gets the full list on every call. What happens to its cost with 10,000 mostly idle clients? (macOS has `kqueue` and Linux has `epoll` for this. Optional upgrade after step 6, when you can measure it.)

## Look up

- `man 2 poll`, `man 2 fcntl`.
- *Build Your Own Redis* (build-your-own.org), the event loop chapters. Read after your own attempt, not before.

## Done

Commit. Tick step 5 in `README.md`.
