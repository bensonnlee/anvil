# Step 3: TCP server for one client

## Goal

`anvil` listens on port 6379. You connect with `nc localhost 6379`, type the step-2 commands, and get the same answers. One client at a time.

## Why this step

Same calls you used in Python, but in C you see everything Python hid: byte order, buffers, return codes, and the fact that TCP has no idea what a "message" is.

## Concepts

- **File descriptor.** A small integer the operating system gives you to name an open socket or file. Python wrapped it in a socket object; in C you pass the integer to every call.
- **The calls.** `socket`, `bind`, `listen`, `accept`, `read`, `write`, `close`. Same order and meaning as Python's `socket.socket()`, `.bind()`, `.listen()`, `.accept()`, `.recv()`, `.send()`, `.close()`.
- **Two sockets.** The listening socket only accepts connections. `accept` returns a *new* descriptor for each client. You read and write on that one.
- **Addresses.** `bind` takes a `sockaddr_in` struct holding the address and port. The port must be converted with `htons`, because the network uses a different byte order from your CPU.
- **Errors.** No exceptions. Every call returns `-1` on failure and sets `errno`. Check every return value. `perror` prints the reason.
- **TCP is a stream of bytes, not messages.** One `read` may give you half a command, exactly one, or three and a half. This is the most important idea in this step.
- **`read` returning 0** means the client closed the connection.
- **`write` may send fewer bytes than you asked.** Its return value says how many. Loop until all are sent.

## Tasks

1. **Echo server.** Accept one client, read bytes, write the same bytes back, exit when the client disconnects. Test with `nc`.
2. **Keep serving.** After a client disconnects, go back to `accept` for the next one.
3. **Restart quickly.** Stop and restart the server immediately. If you see "address already in use", look up `SO_REUSEADDR`.
4. **Lines from a stream.** Give the connection a buffer. Append whatever `read` returns. While the buffer contains a complete line (ends in `\n`), remove it from the buffer and process it. Leftover bytes stay for next time.
5. **Connect step 2.** Pass each line through your tokeniser and execute function. Write the reply followed by a newline.
6. **Write everything.** Put the "loop until all bytes are sent" logic in one function and use it for every reply.
7. **Survive rude clients.** Writing to a client that already left kills your process with a signal called `SIGPIPE`. Look up how to ignore it.

## Check it works

- `nc localhost 6379`, then type `SET a 1`, `GET a`.
- Several commands in one read: `printf 'SET a 1\nGET a\nGET a\n' | nc localhost 6379` prints three replies.
- Half a command per read: write a short Python client (you know how) that sends `SET a`, sleeps one second, then sends ` 1\n`. The server must reply `OK` once, after the second part.
- Kill `nc` with Ctrl-C mid-session. The server keeps running and accepts the next client.
- Open two `nc` sessions at once. The second one hangs until the first disconnects. Understand why. Step 5 fixes it.

## Pitfalls

- The bytes from `read` are not a C string. There is no terminating zero. Always use the count `read` returned.
- `nc` sends `\n`. Other clients send `\r\n`. Strip a trailing `\r`.
- Forgetting `htons` gives you a server listening on a different port than you think.
- Forgetting `close` on a client descriptor leaks it. Run long enough and `accept` starts failing.

## Questions to answer before moving on

- Why are there two different descriptors, one from `socket` and one from `accept`?
- Your `read` returned 7 bytes: `GET a\nG`. What do you do with the trailing `G`?
- Why does the second `nc` hang? Which line of your code is the server stuck on?

## Look up

- `man 2 socket`, `man 2 bind`, `man 2 accept`, `man 2 read`, `man 2 write`.
- *Beej's Guide to Network Programming* (free online). The clearest introduction to the C socket API.

## Done

Commit. Tick step 3 in `README.md`.
