# anvil

A Redis clone and a load tester, written by hand in C++ to learn how they work.

- `anvil`: the server. Keeps a dictionary in memory and answers commands over the network.
- `hammer`: the load tester. Spawns many fake users against `anvil` and reports how fast it answers.

## Roadmap

Each step ends with a working program and a commit. Each has a guide in `docs/steps/` with the concepts, ordered tasks, checks, pitfalls, and questions. The guides contain no code.

- [ ] 0. [Use real Redis](docs/steps/00-use-real-redis.md)
- [ ] 1. [Hello world and the toolchain](docs/steps/01-hello-world.md)
- [ ] 2. [Key-value store on the keyboard](docs/steps/02-keyboard-store.md)
- [ ] 3. [TCP server for one client](docs/steps/03-tcp-server.md)
- [ ] 4. [Speak the Redis protocol (RESP)](docs/steps/04-resp-protocol.md)
- [ ] 5. [Many clients at once](docs/steps/05-many-clients.md)
- [ ] 6. [`hammer`, the load tester](docs/steps/06-hammer.md)
- [ ] 7. [Key expiry](docs/steps/07-expiry.md)
- [ ] 8. [Save to disk](docs/steps/08-persistence.md)
- [ ] 9. [My own hash table](docs/steps/09-hash-table.md)
- [ ] 10. [Lists and sorted sets](docs/steps/10-lists-and-sorted-sets.md)

## How to work a step

1. Read the guide's Goal and Concepts. Ask Claude about anything unclear.
2. Do the tasks in order. Keep the program working after each one.
3. Stuck for 20 minutes: ask Claude for a hint, not the answer.
4. Run every item under "Check it works".
5. Answer the questions out loud or to Claude.
6. Ask Claude to review the code, fix what it finds, commit, tick the box.

## Log

One line per session: date, what I did, what confused me.
