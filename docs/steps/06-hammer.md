# Step 6: `hammer`, the load tester

## Goal

A second program, `hammer`, that simulates many users against `anvil` and prints one line per second:

    users   req/s   errors   p50     p95     p99     max
    50      81234   0        0.51ms  0.92ms  1.80ms  6.2ms

## Why this step

From here on every change gets a number. You stop guessing whether something is fast.

## How Locust works, and what you are copying

- A **user** is a loop: pick a task, send it, wait for the answer, record how long it took, optionally pause, repeat.
- You choose **how many users** and a **spawn rate** (users added per second until the target is reached).
- A **task mix** says how often each kind of request is sent, for example 80% `GET` and 20% `SET`.
- It reports **requests per second**, **failures**, and **response-time percentiles**.

## Concepts

- **Throughput and latency.** Throughput is requests completed per second across all users. Latency is how long one request took. They are different measurements and can move in opposite directions.
- **Percentiles.** Sort all latencies. p50 is the middle one. p99 is the one 99% of the way along: 1 request in 100 was slower. Averages hide the slow requests that users notice; percentiles show them.
- **Monotonic clock.** Time requests with `std::chrono::steady_clock`. The normal wall clock can jump.
- **Closed loop.** Each user waits for its reply before sending again. So if the server slows down, the users automatically send less. Remember this limitation; see "Later" below.
- **Threads.** The simplest design is one thread per user, each with an ordinary blocking socket. This is the opposite design from your server, which is useful: you get to feel both.
- **Shared data between threads.** User threads record latencies; a reporter thread reads them every second. That is shared mutable state, a race condition waiting to happen. Protect it with `std::mutex`, or give each user its own list and swap it out under a lock.

## Two programs, shared code

`hammer` must encode requests and parse replies, so it needs your RESP code. Typing two long `clang++` commands is now painful. Write a `Makefile` (or CMake, your choice) that builds `anvil`, `hammer`, and the parser tests, in both a debug build (sanitizers) and a release build (`-O2`).

## Tasks

1. **One request.** `hammer` connects, sends `PING`, parses the reply, prints it, exits.
2. **One user for 10 seconds.** Loop requests. Print the total count at the end.
3. **Time each request.** Store every latency. At the end print p50, p95, p99, max.
4. **Many users.** One thread per user, each with its own connection. Combine results at the end.
5. **Report every second** while running, not only at the end.
6. **Ramp up.** Start users gradually at a spawn rate.
7. **Task mix.** A `GET`/`SET` ratio and a key-space size; each request picks a random key such as `key:4821`.
8. **Options on the command line:** users, spawn rate, duration, `GET` ratio, number of keys, host, port.
9. **Failures.** A refused connection or an error reply is counted, not fatal.

## Check it works

- With 1 user, requests per second is close to 1 divided by the average latency. If not, something is mistimed.
- Run `hammer` against real Redis and `redis-benchmark` against real Redis with the same number of connections. The numbers should be in the same range.
- Kill the server mid-run. `hammer` reports errors and exits cleanly.
- Build `hammer` once with `-fsanitize=thread` and run it. It reports any data race.

## Experiments

Do these and write the results in the README log.

- Run with 1, 10, 50, 200, and 1000 users. Record requests per second and p99 for each. Where does throughput stop rising? What happens to p99 past that point?
- Same runs against real Redis. How far off are you?
- Debug build versus release build of the server.

## Pitfalls

- Benchmarking a sanitizer build. Always measure release builds.
- `hammer` itself being the bottleneck. Watch both processes in `top`. If `hammer` is at its limit, you are measuring the hammer, not the anvil.
- Timing more than the request. Start the clock just before sending and stop just after the reply is fully parsed.
- Reporting an average of per-user percentiles. Percentiles must be computed over all samples together.

## Questions to answer before moving on

- Why can p99 get worse while requests per second stays the same?
- In a closed loop, the server freezes for one second. How many slow requests does each user record? How many would real, independent users have experienced?
- Why does each user need its own connection?

## Later (optional)

- The freeze question above is called **coordinated omission**. Watch Gil Tene's talk "How NOT to Measure Latency", then add an open-loop mode that sends at a fixed rate regardless of replies.
- Replace "store every sample" with a histogram so memory stays constant on long runs.

## Done

Commit. Tick step 6 in `README.md`.
