# Step 7: Key expiry

## Goal

Keys can be given a lifetime and disappear when it ends, including keys nobody ever reads again.

| Command | Reply |
|---|---|
| `EXPIRE key seconds` | `1` if set, `0` if the key does not exist |
| `TTL key` | seconds left; `-1` if no expiry; `-2` if no such key |
| `PERSIST key` | removes the expiry |
| `SET key value EX seconds` | set with a lifetime in one command |

A plain `SET` on an existing key removes its expiry.

## Why this step

Redis is mostly used as a cache, and a cache needs old entries to go away. It is also your first feature that happens on a timer instead of in response to a client.

## Concepts

- **Deadline, not duration.** Store the absolute time at which the key dies. Then checking is one comparison with "now".
- **Lazy expiry.** Whenever a key is accessed, check its deadline first. If it has passed, delete it and behave as if it never existed. Cheap and exact, but a key nobody touches again stays in memory forever.
- **Active expiry.** A few times a second, look at a small random sample of keys that have a deadline and delete the dead ones. Redis samples 20; if more than a quarter were dead, it immediately samples again.
- **Why sample instead of scanning everything?** One thread. A full scan of ten million keys freezes every client. Same lesson as the "one slow command" question in step 5.
- **Waking up without a client.** `poll` takes a timeout. Use it so the loop wakes up regularly even when no client sends anything.

## Tasks

1. Decide where deadlines live. The simplest design is a second map from key to deadline.
2. `EXPIRE` and `TTL`, with lazy expiry on every command that reads or writes a key.
3. Every command that deletes or overwrites a key keeps the two maps consistent.
4. `PERSIST` and `SET ... EX`.
5. Give `poll` a timeout and run an active expiry pass on each wake-up.
6. Make the amount of work per pass bounded, however many keys exist.

## Check it works

- `SET a 1`, `EXPIRE a 2`, `TTL a`, wait, `GET a` returns nil, `TTL a` returns `-2`.
- `SET a 1`, `EXPIRE a 100`, `SET a 2`, `TTL a` returns `-1`.
- Active expiry: add a `hammer` mode that only writes fresh keys with a 1-second lifetime and never reads them. Watch `DBSIZE` from `redis-cli` during the run. With active expiry it levels off. Turn active expiry off and it climbs without limit.
- `hammer` p99 with and without active expiry running. It should be almost the same.

## Pitfalls

- Forgetting the lazy check in one command (`EXISTS`, `DEL`, `TTL` itself).
- Leaving a deadline behind after its key is deleted, so a later key with the same name dies unexpectedly.
- "Pick a random key" is slow on `std::unordered_map`. Accept a simple approach for now and note it. Your own hash table in step 9 can do it properly.
- Mixing seconds and milliseconds. Put the unit in every variable name.

## Questions to answer before moving on

- Why are both lazy and active expiry needed? What breaks with only one?
- Which clock did you use for deadlines, `steady_clock` or the wall clock? What happens with each if the computer's time is changed? Remember your answer; step 8 forces the choice.

## Done

Commit. Tick step 7 in `README.md`.
