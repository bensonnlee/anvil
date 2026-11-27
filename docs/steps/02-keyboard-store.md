# Step 2: Key-value store on the keyboard

## Goal

Run `./build/anvil`, type commands, get Redis-like answers. No network.

    SET name benson     ->  OK
    GET name            ->  benson
    GET missing         ->  (nil)
    DEL name            ->  (integer) 1
    EXISTS name         ->  (integer) 0

## Why this step

The heart of Redis is "look up a key in a dictionary". Build that first, alone, so network bugs in step 3 cannot hide logic bugs.

## Concepts

- **Hash map.** `std::unordered_map<std::string, std::string>` is C++'s `dict`. Lookup, insert, and delete take roughly constant time however many keys there are. You will build your own in step 9.
- **Tokenising.** Splitting `SET name benson` into `["SET", "name", "benson"]`. `std::istringstream` does this for whitespace-separated words.
- **Separate logic from input/output.** Write one function that takes the list of words and returns the reply. It must not read the keyboard or print. In step 3 the same function will be fed from a socket instead, unchanged.

## Tasks

1. Split each input line into words. Print them, one per line, to prove it works.
2. Write the execute function described above. Start with only `PING`, which returns `PONG`.
3. Add the store and `SET`, `GET`.
4. Add `DEL` and `EXISTS`. Both return a count.
5. Handle the bad cases. None of these may crash:
   - Unknown command.
   - Wrong number of arguments (`SET a`, `GET`).
   - Empty line.
6. Make command names case-insensitive (`get`, `GET`, `Get`). Keys stay case-sensitive.

## Check it works

- The table under Goal, typed by hand, gives those answers.
- `printf 'SET a 1\nGET a\nDEL a\nGET a\n' | ./build/anvil` prints `OK`, `1`, `(integer) 1`, `(nil)`.
- Compare your replies and error wording against real `redis-cli` for the same commands.

## Pitfalls

- Reading a missing key with `map[key]` silently *inserts* an empty value. Look up how to check for a key without inserting.
- The reply for "missing" and the reply for "empty string" must be distinguishable inside your program, not just when printed. Look at `std::optional`.
- `SET greeting hello world` cannot work yet because you split on spaces. Leave it broken. Step 4 fixes it properly.

## Questions to answer before moving on

- What does a hash function do, and why does it make lookup fast?
- What happens when two keys hash to the same slot? (A one-sentence guess is enough for now.)
- Why is it useful that the execute function does no input or output?

## Done

Commit. Tick step 2 in `README.md`.
