# Step 9: Your own hash table

## Goal

`std::unordered_map` is gone from the store. Your own hash table takes its place, and it never pauses the server while growing.

## Why this step

You have used a hash map since step 2 without knowing how it works. Building one teaches pointers, memory ownership, and the reason Redis's table is designed the way it is: a single-threaded server cannot afford one slow operation.

## Concepts

- **Buckets.** An array of slots. `hash(key) % number_of_buckets` picks the slot.
- **Hash function.** Turns a string into a number, spread evenly. Start with `std::hash`. Writing FNV-1a yourself is a short optional exercise.
- **Collision.** Two keys land in the same bucket. With *chaining*, each bucket is a linked list of entries. Redis uses chaining; use it too.
- **Load factor.** Entries divided by buckets. As it rises, chains get longer and lookups slower. When it passes about 1, grow.
- **Growing (rehash).** Allocate a bigger array, usually double, and move every entry to its new bucket.
- **The problem.** Moving ten million entries takes long enough to freeze every client. In `hammer` it shows as a huge `max` latency at the moment of growth.
- **Incremental rehash.** Keep two tables, old and new. Every operation moves a few buckets from old to new. Lookups check both. Inserts go to the new one. When the old one is empty, drop it. The work is the same, but spread across thousands of requests.
- **Ownership.** Each entry is allocated on the heap. Something must free it exactly once. Decide whether you use `std::unique_ptr` or manual `new` and `delete`, and keep the address sanitizer on while you develop.

## Tasks

1. **Write down the interface** the server needs: find, insert-or-overwrite, erase, size. Keep it that small.
2. **Fixed-size chaining table.** No growth yet.
3. **Test against an oracle.** A test program that performs thousands of random operations on both your table and `std::unordered_map`, and checks after every one that they agree. Use a fixed random seed so a failure can be repeated.
4. **Grow all at once** when the load factor passes 1. The oracle test must still pass.
5. **Swap it into the server.** `redis-cli` and `hammer` work as before.
6. **Measure the pause.** Use `hammer` to insert several million distinct keys. Watch the `max` column.
7. **Incremental rehash.** Oracle test first, then the server.
8. **Measure again.** The spikes should be gone.
9. **Random key** for active expiry, done properly now: pick a random bucket.

## Check it works

- Oracle test passes with sanitizers on, including a run that forces many growths.
- Sanitizers report no leaks when the test program exits.
- Before and after `max` latency for task 6 and task 8 are written in the README log.
- Throughput is in the same range as with `std::unordered_map`.

## Pitfalls

- During a rehash, find, erase, and overwrite must all look in both tables. Forgetting one gives a bug that only appears while growing, which is why the oracle test must force growth.
- Inserting a key that already exists in the old table creates a duplicate unless you check both first.
- Freeing an entry while something still points to it. The sanitizer will tell you; read its report carefully.
- A bucket count that is a power of two lets you replace `%` with a bit mask. Do that only after it works, and measure whether it matters.

## Questions to answer before moving on

- Why is lookup constant time on average but not in the worst case?
- What exactly does each operation do while a rehash is in progress?
- What did the oracle test catch that you would not have found by hand?

## Look up

- `dict.c` in the Redis source code, after your own version works. Compare the design choices.

## Done

Commit. Tick step 9 in `README.md`.
