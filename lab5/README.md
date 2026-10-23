# Lab 5: Reverse Engineering

## Contents

- [Objective](#objective)
- [Requisites](#requisites)
- [Running the container](#running-the-container)
- [The challenge](#the-challenge)
- [Analysis tools](#analysis-tools)
  - [GDB](#gdb)
  - [Radare2](#radare2)
- [Deliverables](#deliverables)
- [References](#references)

## Objective

In this lab we will analyze a program without access to its source code. The
program is deliberately obfuscated: the part of its code that decides the
result is not present in the executable in readable form. That code only
appears in memory while the program runs.

The goal is to determine what the program does, which condition it checks, and
how to satisfy that condition so the program produces a flag. You will deliver
the flag together with a short explanation of how you obtained it.

## Requisites

- `podman` installed on your machine.
- An Internet connection. The program downloads a small key file when it runs,
  so the container needs access to the network.
- An x86-64 Linux host.

## Running the container

Start an interactive Bash shell in the container:

```bash
podman run -it --name challenge-container \
  ghcr.io/fib-gei-si/obfuscated-binary-simple/obfuscated-binary-simple:latest
```

The shell runs as `root` inside the container. The challenge executable is
`/challenge`. Run it once, before you start your analysis, to see how it
behaves:

```bash
/challenge
```

The program asks for your email address. Enter your UPC email address.
The instructor uses the address to validate the flag.

To re-enter the container after you exit its shell:

```bash
podman start --attach --interactive challenge-container
```

Remove the container when you finish:

```bash
podman rm challenge-container
```

## The challenge

`/challenge` prints a banner and asks for your email address. It then acquires
a key, decrypts an embedded payload, and runs that payload. What you see in the
executable is only the loader. The code that decides whether you get the flag
is hidden and only becomes visible in memory at run time.

The payload performs a check on the system. When the check succeeds, the
program prints `Delivery Flag:` followed by a line of the form:

```text
email@address sha256_hash
```

When the check fails, the program prints an error and no flag. Your task is to
find out what the payload checks and to make the check succeed.

You do not need to break the key, the hash, or the download. You need to read
the executable, both statically and at run time, and to understand the
condition that decides between success and failure.

## Analysis tools

The container includes two tools for the analysis. Use whichever you prefer.
Both can open `/challenge`.

### GDB

GDB is a command-line debugger. It can start `/challenge` under your control,
inspect registers and memory, and step through the code.

```bash
gdb /challenge
```

Useful commands:

```text
info files
info functions
disassemble main
break main
run
next
step
info registers
x/i $pc
continue
backtrace
quit
```

Because the payload is generated in memory at run time, break after the program
decrypts the payload and inspect the code it is about to run. A breakpoint
around the `Verifying system status` message is a good place to start.

### Radare2

Radare2 (`r2`) can inspect the executable statically and also debug it.

```bash
r2 -A /challenge     # static analysis
r2 -d /challenge     # run it under the r2 debugger
```

Static-analysis commands:

```text
iI
afl
s main
pdf
q
```

Debugger commands:

```text
db main
dc
ds
dr
pdf
q
```

Use `?` inside Radare2 for command help.

## Deliverables

Submit the following through Atenea, in the section for this lab:

- The flag printed by `/challenge`, in the form
  `email@address sha256_hash`.
- A short explanation of how you obtained the flag: what the hidden payload
  checks, where you found the check, and the steps you followed to satisfy it.

## References

- GDB documentation: <https://sourceware.org/gdb/current/onlinedocs/gdb.html/>
- Radare2 book: <https://book.radare.org/>
- Practical Binary Analysis: <https://practicalbinaryanalysis.com/>
