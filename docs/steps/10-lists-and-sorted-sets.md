# Step 10: Lists and sorted sets

## Goal

Values are no longer only strings. `anvil` supports lists (queues) and sorted sets (leaderboards).

| Lists | Sorted sets |
|---|---|
| `LPUSH`, `RPUSH` | `ZADD key score member` |
| `LPOP`, `RPOP` | `ZSCORE key member` |
| `LLEN` | `ZRANK key member` |
| `LRANGE key start stop` | `ZRANGE key start stop` |
| | `ZREM key member` |

Run each in real Redis first and note the replies, including on missing keys.

## Why this step

Redis calls itself a data structure server. The interesting part is choosing the structure that makes each command fast.

## Concepts

- **Typed values.** A key now holds a string, a list, or a sorted set. Look at `std::variant`.
- **Wrong type.** `LPUSH` on a key that holds a string is an error reply starting with `WRONGTYPE`. Every command checks the type first.
- **List.** Needs fast push and pop at both ends. Start with `std::deque`.
- **Sorted set.** Each member has a score. It needs two things at once: find a member's score quickly, and walk members in score order. One structure cannot do both well, so use two that stay in sync: a hash map from member to score, and an ordered structure of (score, member).
- **Skip list.** Redis's ordered structure. A sorted linked list where some nodes also have "express lane" links that skip ahead, giving search in about log(n) steps. Simpler to write than a balanced tree.
- **Rank.** "What position is this member?" must not count from the start. Each express link also stores how many nodes it skips; adding those up along the search path gives the rank.

## Tasks

1. Change the value type to hold more than strings. All existing commands and tests still pass.
2. `WRONGTYPE` checks on every existing command.
3. Lists, using `std::deque`.
4. Sorted sets, first version: your hash table or `std::unordered_map` for member to score, and `std::set` of (score, member) for order. Get every command's behaviour right, with `ZRANK` done the slow way.
5. Check persistence. If step 8 logs commands generically, lists and sorted sets survive a restart with no new code. Verify it.
6. Write a skip list with insert, erase, and in-order walk. Oracle-test it against `std::set`, as in step 9.
7. Add skip counts and a fast `ZRANK`.
8. Swap the skip list into sorted sets.
9. Add a leaderboard workload to `hammer`: `ZADD` with random scores, `ZRANK`, and `ZRANGE 0 9`.

## Check it works

- Every command's reply matches real Redis, including empty and missing-key cases and negative indexes in `LRANGE` and `ZRANGE`.
- `ZADD` on an existing member changes its score and its position, and leaves no stale entry behind.
- Restart with lists and sorted sets present; all data returns.
- `hammer` leaderboard run with one million members: `ZRANK` p99 before and after task 7.

## Pitfalls

- Updating a score in one of the two structures and not the other.
- A list or sorted set that becomes empty must delete its key. Real Redis never keeps empty collections.
- Members with equal scores are ordered by member name. Your comparison must do the same or ranks will be unstable.
- `LRANGE` on a huge list returns a huge reply. That is the "one slow command" problem from step 5 again.

## Questions to answer before moving on

- Why does a sorted set need two structures?
- Why is a skip list search about log(n) steps?
- Which of your commands are now not constant time? What does that mean for other clients?

## Where to go next

Pick what interests you. Each is its own project.

- `kqueue` in place of `poll`, measured with thousands of idle connections.
- Replication: a second `anvil` that follows the first.
- Transactions (`MULTI`/`EXEC`) and publish/subscribe.
- Deterministic simulation testing: run the server against a fake clock, network, and disk with injected faults.
- Open-loop mode and histograms in `hammer`.

## Done

Commit. Tick step 10 in `README.md`.
