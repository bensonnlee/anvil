# Step 4: Speak the Redis protocol (RESP)

## Goal

The real `redis-cli` connects to `anvil` and works. Values with spaces work.

## Why this step

Your line protocol cannot carry a value containing a space or newline. RESP solves this by saying how long each piece is before sending it. Once you speak it, every existing Redis tool becomes a test tool for your server.

## Concepts

RESP has five kinds of value. Every line ends in `\r\n`. The first byte says the kind.

| First byte | Kind | Example on the wire |
|---|---|---|
| `+` | Simple string | `+OK\r\n` |
| `-` | Error | `-ERR unknown command\r\n` |
| `:` | Integer | `:1\r\n` |
| `$` | Bulk string: length, then that many bytes | `$5\r\nhello\r\n` |
| `*` | Array: count, then that many values | `*2\r\n$3\r\nGET\r\n$1\r\na\r\n` |

- **"Not found"** is a bulk string with length -1: `$-1\r\n`.
- **Requests** are always an array of bulk strings. `GET a` is the array example in the table.
- **Replies** use whichever kind fits: `+OK` for `SET`, a bulk string for `GET`, an integer for `DEL`.
- **Length-prefixed** means the parser never searches the value for a terminator, so the value can contain any bytes.
- **Incremental parsing.** The buffer may hold half a request. The parser has three possible outcomes: a complete command plus how many bytes it used, "need more bytes", or "this is not valid RESP".
- **Pipelining.** A client may send many requests without waiting. The buffer can hold several; parse until you get "need more bytes".

## Tasks

1. **Parser as a pure function.** Input: a buffer of bytes. Output: one of the three outcomes above. No sockets, no globals. Put it in its own file.
2. **Test program for the parser.** A second small executable that calls the parser and checks results. Cover:
   - One complete command.
   - Every prefix of a valid command (cut it after 1 byte, 2 bytes, and so on). Each must give "need more bytes", never a crash or a wrong command.
   - Two commands back to back. First call returns the first and its length; parsing the rest returns the second.
   - Invalid input: wrong first byte, a length that is not a number, a negative count.
3. **Reply encoders.** One small function per reply kind that produces the bytes.
4. **Change the execute function** to return a typed reply instead of display text like `(nil)`. That text was `redis-cli`'s formatting, not the protocol.
5. **Swap it into the server** in place of the line splitter. The buffer handling from step 3 stays the same shape.
6. **Add `PING` and `ECHO`.**
7. **Unknown commands** return an error reply and keep the connection open. `redis-cli` sends a few commands of its own when it connects (such as `COMMAND DOCS`); an error reply is fine and it carries on.

## Check it works

- The parser test program passes.
- `redis-cli` connects. `SET greeting "hello world"` then `GET greeting` returns the full value.
- `redis-cli GET missing` prints `(nil)`.
- `redis-cli NOPE` prints an error and the next command still works.
- The raw `printf ... | nc` command from step 0 gets the same bytes back from your server as from real Redis.

## Pitfalls

- After the N bytes of a bulk string there is one more `\r\n`. The command is not complete until that has arrived too.
- Never trust a length from the network. `$999999999999\r\n` must not make you allocate that much memory. Pick a maximum and return "invalid" above it.
- Separate "need more bytes" from "invalid" carefully. Confusing them makes the server either hang forever or drop valid clients.

## Questions to answer before moving on

- Why can a bulk string contain `\r\n` inside it without confusing the parser?
- Why is the parser easier to test as a function that knows nothing about sockets?
- A client sends three commands in one packet. Walk through exactly what your server does.

## Look up

- The Redis protocol specification (search "RESP protocol spec" on redis.io). You are implementing RESP2.

## Done

Commit. Tick step 4 in `README.md`.
