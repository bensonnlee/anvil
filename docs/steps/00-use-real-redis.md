# Step 0: Use real Redis

## Goal

You have used the real thing for 30 minutes and know what you are copying. No code yet.

## Concepts

- **Server and client.** `redis-server` is a long-running program holding data in memory. `redis-cli` is a small program that connects to it, sends a command, and prints the reply.
- **Port.** The server listens on TCP port 6379 on your own machine (`localhost`).
- **Protocol.** Client and server exchange plain bytes in a format called RESP. You will implement it in step 4.

## Tasks

1. Install with `brew install redis`.
2. Run `redis-server` in one terminal tab. Leave it in the foreground so you can see its log. Stop it with Ctrl-C.
3. Run `redis-cli` in a second tab and try each of these. Write down what each replies:
   - `PING`
   - `SET name benson`, `GET name`, `GET missing`
   - `DEL name`, then `DEL name` again
   - `EXISTS name`
   - `SET counter 10`, `INCR counter`
   - `SET temp hello`, `EXPIRE temp 5`, `TTL temp` a few times, then `GET temp` after 5 seconds
   - `SET greeting "hello world"`
   - `NOTACOMMAND`
4. See the raw protocol. Without `redis-cli`, run:
   - `nc localhost 6379`, type `PING`, press Enter. Note the exact reply.
   - `printf '*2\r\n$3\r\nGET\r\n$4\r\nname\r\n' | nc localhost 6379`
5. See what a load tester prints: `redis-benchmark -t set,get -n 100000 -q`.
6. Set a key, stop the server with Ctrl-C, start it again, and `GET` the key.

## Questions to answer before moving on

- What are the different *kinds* of reply you saw (text, number, nothing, error)? How did `redis-cli` display each?
- In task 4, what do you think `*2`, `$3`, and `\r\n` mean?
- In task 6, was the key still there? Look in the folder where you ran `redis-server` for a new file. What does that tell you?
- What two numbers did `redis-benchmark` report, and what does each measure?

## Note for later

Your own server will also use port 6379. Real Redis must be stopped when you run yours, or you will get "address already in use".

## Done

Tick step 0 in `README.md`. Nothing to commit except the tick and a log line.
