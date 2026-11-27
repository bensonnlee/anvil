# Step 8: Save to disk

## Goal

Stop `anvil`, even with `kill -9`, start it again, and the data is back.

## Why this step

Memory is lost when the process dies. This step is about the trade-off between safety and speed, and you can now measure it with `hammer`.

## Concepts

- **Two designs.**
  - *Snapshot:* every so often, write the whole dictionary to a file. Compact and fast to load, but you lose everything since the last snapshot.
  - *Append-only log:* every command that changes data is appended to a file. On start, replay the file from the beginning. You lose very little, but the file grows forever.
  - Redis has both. Build the log; it reuses what you already have.
- **You already wrote the file format.** Append each write command in RESP, exactly as it arrived. Replay is: read the file, parse with your parser, run through your execute function.
- **`write` does not mean "on disk".** It copies bytes to the operating system's memory. The OS writes them to the drive later. If the machine loses power first, they are gone.
- **`fsync`** asks the OS to push the file to the drive now. It is slow: often milliseconds, against microseconds for everything else you do.
- **Policies.** *Always:* `fsync` before replying, lose nothing, slowest. *Every second:* lose up to one second. *Never:* let the OS decide.
- **Acknowledged write.** A write the client was told succeeded. The promise of each policy is about which acknowledged writes can be lost.

## Tasks

1. Mark which commands change data. Only those are logged.
2. Append each such command to the log file before replying.
3. On start, if the log exists, replay it before accepting clients.
4. Handle a damaged tail. A crash can leave half a command at the end of the file. Replay everything before it, discard the partial command, and continue.
5. Add the three `fsync` policies as a start-up option.
6. Fix expiry in the log. Replaying `EXPIRE a 10` an hour later would wrongly give the key ten fresh seconds. Log the absolute deadline instead. This needs wall-clock time; revisit your step-7 answer.

## Check it works

- Set keys, Ctrl-C, restart, keys are present.
- Run `hammer`, `kill -9` the server mid-run, restart. It starts cleanly and `DBSIZE` is plausible.
- Cut the last few bytes off the log by hand (`truncate`) and restart. It recovers everything except the damaged command.
- A key set with a 60-second lifetime, then a restart 10 seconds later, has about 50 seconds left.

## Experiments

Record in the README log:

- `hammer` throughput and p99 with a write-heavy mix under each of the three policies.
- Time to restart after loading one million keys.

## Pitfalls

- Logging read commands, which bloats the file for nothing.
- Replay writing to the log again, doubling the file on every restart.
- Replying to the client before the append, under the "always" policy.
- On macOS, `fsync` does not force the drive itself to flush. Look up `F_FULLFSYNC`, try it, and measure the difference.

## Questions to answer before moving on

- Under each policy, exactly which acknowledged writes can be lost on a process crash? On a power cut?
- Why does a slow `fsync` hurt every client and not only the one writing?
- A key is overwritten a million times. How big is the log, and how much of it matters?

## Later (optional)

- **Log rewrite.** Write the current dictionary as a fresh, minimal log into a temporary file, then `rename` it over the old one. Find out why `rename` is the safe way to replace a file.

## Done

Commit. Tick step 8 in `README.md`.
