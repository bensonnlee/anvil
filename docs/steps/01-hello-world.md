# Step 1: Hello world and the toolchain

## Goal

A C++ file in `src/` that compiles with strict flags, reads lines from the keyboard, and echoes them back until you press Ctrl-D.

## Why this step

This step gets the compiler, the warnings, and the memory checker working before anything is hard.

## Concepts

- **Compile and link.** The compiler turns each `.cpp` file into machine code; the linker joins them into one executable. With one file, `clang++` does both in one command.
- **Warnings.** `-Wall -Wextra` make the compiler report suspicious code. Treat every warning as a bug.
- **Sanitizers.** `-fsanitize=address,undefined` adds runtime checks. If your program reads memory it should not, it stops and prints exactly where. In Python that kind of bug raises an exception; in C++ without sanitizers it silently corrupts data.
- **Standard input and EOF.** Ctrl-D in the terminal means "no more input". Your read loop must end cleanly when that happens.

## The build command

    clang++ -std=c++20 -Wall -Wextra -g -fsanitize=address,undefined src/main.cpp -o build/anvil

Look up what each flag does before you use it. `-g` is the one that makes sanitizer reports show line numbers.

## Tasks

1. Create `src/main.cpp` that prints one line and exits. Build it into `build/` and run it.
2. Change it to read a line with `std::getline` and print it back, in a loop, until EOF.
3. Break it on purpose, to learn what the tools tell you:
   - Declare a variable and never use it. Read the warning.
   - Make a `std::vector` with 3 elements and read element 10 with `[]`. Run it and read the sanitizer report from top to bottom.
   - Remove the bug.
4. Refresh these by writing a few throwaway lines for each, then delete them:
   - `std::string`: length, comparing, `substr`, appending.
   - `std::vector`: `push_back`, looping, `size`.
   - Passing by value, by reference (`&`), and by const reference (`const &`). Print something that proves which one copies.

## Check it works

- `echo "hi" | ./build/anvil` prints `hi` and exits.
- Running it and pressing Ctrl-D exits with no error.
- The build prints zero warnings.

## Pitfalls

- `std::cin >> word` reads one word, not one line. You want the whole line.
- `build/` is in `.gitignore`, so the binary is never committed. That is intended.

## Questions to answer before moving on

- What is the difference between an error at compile time and a sanitizer report at run time?
- When you pass a `std::string` to a function by value, what happens to the original if the function changes it?
- Why prefer `const std::string&` for a parameter you only read?

## Done

Commit. Tick step 1 in `README.md`.
