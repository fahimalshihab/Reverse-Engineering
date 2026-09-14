# `strace`: Understanding a Linux Program Through System Calls

*Reverse Engineering Series — Part 1*

<img width="1672" height="941" alt="image" src="https://github.com/user-attachments/assets/267e840e-f831-45b8-ab69-adc4848b36fb" />


When you run a Linux program, you normally see only the result.

```bash
./demo
```

Maybe it prints:

```text
Done!
```

That's all we see.

But behind that one line, Linux may have performed a surprising amount of work.

The program had to be loaded. Its libraries had to become available. Memory had to be mapped. A file may have been opened. Data may have been written. The terminal had to receive output. Finally, the process had to terminate.

Most of that activity is invisible to us.

But it isn't invisible to the operating system.

And this is where `strace` becomes interesting.

<img width="602" height="602" alt="image" src="https://github.com/user-attachments/assets/2670c389-12ef-444b-96cd-443f34ac7c5a" />


## The idea behind `strace`

`strace` is a Linux utility for tracing the **system calls and signals** associated with a process.

The easiest way to understand it is to think of a program as someone interacting with Linux.

The program might say:

> "Open this file."

Linux responds:

> "Okay. Here's a file descriptor."

The program might then say:

> "Write these bytes."

Linux responds:

> "Done."

Later:

> "Close that file."

And finally:

> "I'm finished."

`strace` lets us observe much of this conversation.

It doesn't show us every instruction executed by the CPU. Instead, it gives us visibility into an important boundary:

<img width="262" height="332" alt="image" src="https://github.com/user-attachments/assets/a7cbcfc3-0b65-47a9-bcce-679d11725a4c" />


That boundary is extremely useful when trying to understand an executable whose source code we don't have.

---

## A very small program

We don't need a complicated program for this.

In fact, keeping the program small is useful because we can easily compare what we wrote with what Linux actually does.

Create `demo.c`:

<img width="775" height="457" alt="image" src="https://github.com/user-attachments/assets/b1e3a2c3-fe3d-4ba1-b9fb-02a45f416ed2" />



Compile it:

```bash
gcc demo.c -o demo
```

Run it normally:

```bash
./demo
```

We get:

```text
Done!
```

Simple.

Now let's ask a different question:

**What did the operating system actually do while this program was running?**

<img width="556" height="353" alt="image" src="https://github.com/user-attachments/assets/0485c057-8c6f-4ef1-8068-57d56c1dd8b5" />


## Now bring in `strace`

Run:

```bash
strace ./demo
```

The terminal will suddenly contain a lot more information.

You may see output similar to:

```text
execve("./demo", ["./demo"], 0x7ffd...) = 0
brk(NULL) = 0x55...
mmap(NULL, 8192, PROT_READ|PROT_WRITE, ...) = 0x7f...
access("/etc/ld.so.preload", R_OK) = -1 ENOENT
openat(AT_FDCWD, "/etc/ld.so.cache", O_RDONLY|O_CLOEXEC) = 3
read(3, "...", 832) = 832
close(3) = 0
...
openat(AT_FDCWD, "/tmp/demo.txt", O_WRONLY|O_CREAT, 0644) = 3
write(3, "Hello strace!\n", 14) = 14
close(3) = 0
write(1, "Done!\n", 6) = 6
exit_group(0) = ?
```

At first glance, this looks intimidating.

Don't worry about the huge output.

We are going to read it like a story.

<img width="723" height="924" alt="image" src="https://github.com/user-attachments/assets/a2332871-b72d-4ba8-b897-a0f2755eb25d" />


## What exactly is a system call?

Before reading the trace, we need one simple concept.

A normal program runs in **user space**.

The Linux kernel runs in **kernel space**.

The program cannot simply perform privileged operating-system operations whenever it wants.

If it needs a kernel-managed resource, it makes a **system call**.

For example:

<img width="402" height="542" alt="image" src="https://github.com/user-attachments/assets/27a33944-73d0-41ea-a82e-8481c99ecc41" />


The system call is the bridge between the application and the kernel.

Some common examples are:

```text
openat()    → open a file
read()      → read data
write()     → write data
close()     → close a descriptor
mmap()      → create a memory mapping
clone()     → create a process/thread
execve()    → execute a program
socket()    → create a socket
connect()   → connect a socket
wait4()     → wait for a child process
exit_group()→ terminate a process
```

There are many more.

But we don't need to memorize them.

The important thing is to recognize what they represent when they appear in a trace.

---

## Reading an `strace` line

Let's take the most interesting line from our program:

```text
openat(AT_FDCWD, "/tmp/demo.txt", O_WRONLY|O_CREAT, 0644) = 3
```

It looks complicated.

It really isn't.

We can break it down:

```text
openat(
    AT_FDCWD,
    "/tmp/demo.txt",
    O_WRONLY|O_CREAT,
    0644
) = 3
```

The general structure is:

```text
system_call(arguments...) = return_value
```

So here:

```text
openat()                         ← system call
AT_FDCWD                        ← directory reference
"/tmp/demo.txt"                 ← target path
O_WRONLY|O_CREAT                ← flags
0644                            ← permissions/mode
= 3                             ← return value
```

The program is essentially asking Linux:

> "Please open `/tmp/demo.txt` for writing, creating it if necessary."

Linux answers:

> "Okay. Use descriptor 3."

That `3` is important.

---

## File descriptors: what is `3`?

Linux represents many resources using **file descriptors**.

The first three are normally:

```text
0 → standard input
1 → standard output
2 → standard error
```

Additional resources normally receive descriptors such as:

```text
3
4
5
6
...
```

So when we see:

```text
openat(...) = 3
```

the process now has a resource associated with descriptor `3`.

Our next syscall is:

```text
write(3, "Hello strace!\n", 14) = 14
```

Notice the `3`.

That's the same descriptor.

So we can connect the two calls:

```text
openat("/tmp/demo.txt") = 3
                 │
                 ▼
          File Descriptor 3
                 │
                 ▼
write(3, "Hello strace!\n", 14)
                 │
                 ▼
             close(3)
```

This is one of the most important habits when reading `strace`:

> **Track the values returned by one syscall and see where they are used later.**

A single number can connect several apparently unrelated lines.

<img width="360" height="552" alt="image" src="https://github.com/user-attachments/assets/fd010621-25d3-446b-811c-e27343645253" />


## `write()` tells us what happened next

Our source contains:

```c
write(fd, "Hello strace!\n", 14);
```

The trace shows:

```text
write(3, "Hello strace!\n", 14) = 14
```

We can read it almost like English:

> Write 14 bytes to descriptor 3. The operation successfully wrote 14 bytes.

The last `14` is the requested byte count.

The final:

```text
= 14
```

is the return value.

This distinction matters.

For example:

```text
write(3, "Hello strace!\n", 14) = 14
```

means the operation succeeded completely.

But if we encountered:

```text
write(3, "...", 14) = 7
```

only 7 bytes were written.

And if we saw:

```text
write(3, "...", 14) = -1 EPIPE
```

the operation failed.

The return value isn't decoration.

**It is evidence.**

---

## Then the program closes the file

Our source says:

```c
close(fd);
```

And `strace` shows:

```text
close(3) = 0
```

The program is finished with descriptor `3`.

So the entire file operation becomes:

```text
                 Program
                    │
                    ▼
        openat("/tmp/demo.txt")
                    │
                    │ = 3
                    ▼
             File Descriptor 3
                    │
                    ▼
      write(3, "Hello strace!")
                    │
                    ▼
               close(3)
```

Three lines of `strace` have now told us exactly what happened.

---

## But wait… we never wrote `openat()`

Look back at our source code:

```c
int fd = open("/tmp/demo.txt", O_CREAT | O_WRONLY, 0644);
```

We wrote:

```text
open()
```

But `strace` showed:

```text
openat()
```

Why?

Because a **library function is not necessarily identical to the underlying system call**.

This is a very important concept.

Our application source interacts with the C library:

```text
open()
```

The library provides an interface for the programmer.

Eventually, the request reaches the kernel through the relevant system-call interface, which on modern Linux commonly appears as:

```text
openat()
```

A simplified model is:

```text
Application
     │
     ▼
libc
     │
     ▼
system call
     │
     ▼
Linux kernel
```

This is why source-level function names and syscall names don't always match.

When reverse engineering, that distinction matters.

---

## Now let's investigate the `printf()`

Our program contains:

```c
printf("Done!\n");
```

But we don't see:

```text
printf()
```

in the trace.

Instead, we see:

```text
write(1, "Done!\n", 6) = 6
```

Why?

Because `printf()` is a user-space library function.

Eventually, the output needs to reach the terminal.

That requires a system call.

The simplified path is:

```text
printf()
   │
   ▼
libc
   │
   ▼
write()
   │
   ▼
Linux kernel
   │
   ▼
Terminal
```

And `1` is important here.

Remember:

```text
0 → stdin
1 → stdout
2 → stderr
```

So:

```text
write(1, "Done!\n", 6)
```

means the process is writing the message to standard output.

<img width="360" height="502" alt="image" src="https://github.com/user-attachments/assets/860ac2a5-1a9c-4f3e-a09d-e9e16c152c19" />

**The function visible in source code may eventually result in a different syscall visible to `strace`.**



## The strange calls before our file operation

If you look at the beginning of the trace, you'll notice many calls we never wrote:

```text
execve()
mmap()
openat()
read()
close()
brk()
...
```

Why?

Because our program doesn't begin with the first line inside `main()`.

There is startup work.

The executable has to be loaded, the runtime environment initialized, and shared libraries made available.

For a dynamically linked executable, the dynamic loader can perform many operations before our application code performs its visible work.

This means that when analyzing `strace`, we need to distinguish between:

```text
Program startup
```

and:

```text
Application-specific behavior
```

For example, an early:

```text
openat(..., "/etc/ld.so.cache", ...)
```

doesn't necessarily mean our C program explicitly decided to inspect that file.

It may be part of the dynamic-linking process.

This is one reason why a raw `strace` output can be noisy.

---

## `execve()` — the beginning of execution

One of the first important calls is usually:

```text
execve("./demo", ["./demo"], ...) = 0
```

Conceptually, the process is asking Linux to execute the specified program.

The first argument is:

```text
"./demo"
```

The second contains the argument vector:

```text
["./demo"]
```

The final result:

```text
= 0
```

indicates success.

A useful simplified picture is:

```text
              ./demo
                │
                ▼
              execve()
                │
                ▼
        Executable loaded
                │
                ▼
          Program starts
```

There is an important subtlety here:

`execve()` doesn't create a new process by itself.

It replaces the executable image of the calling process.

This becomes particularly important when we later encounter process creation followed by `execve()`.

---

## `mmap()` — why is memory showing up?

Our tiny program contains only a few lines, yet the trace may contain many calls to:

```text
mmap()
```

For example:

```text
mmap(NULL, 8192, PROT_READ|PROT_WRITE, ...)
```

At a high level, `mmap()` creates a mapping in the process's virtual address space.

Programs use memory mappings for many normal purposes, including:

* executable code
* shared libraries
* data
* anonymous memory
* file-backed mappings

A typical program startup might therefore look like:

```text
execve()
   │
   ▼
Load executable
   │
   ├── mmap()
   ├── mmap()
   ├── openat()
   ├── read()
   └── close()
```

This is another place where context matters.

Seeing `mmap()` does not automatically mean the program is doing something unusual.

Normal applications use memory mappings constantly.

---

## Memory permissions

You may encounter:

```text
PROT_READ
PROT_WRITE
PROT_EXEC
```

These indicate the permissions associated with a memory mapping.

Think of them simply as:

```text
PROT_READ
    ↓
Memory can be read

PROT_WRITE
    ↓
Memory can be modified

PROT_EXEC
    ↓
Memory can be executed
```

For example:

```text
PROT_READ|PROT_WRITE
```

means readable and writable.

A mapping involving:

```text
PROT_READ|PROT_EXEC
```

is readable and executable.

Memory permissions can become interesting during security analysis, but they should always be interpreted in context.

A syscall is an observation—not a verdict.

---

## Errors are part of the story

One of the easiest mistakes when learning `strace` is to pay attention only to successful operations.

Don't.

Failures can be extremely informative.

For example:

```text
openat(AT_FDCWD, "/tmp/missing.conf", O_RDONLY)
    = -1 ENOENT (No such file or directory)
```

The program tried to open:

```text
/tmp/missing.conf
```

Linux replied:

```text
ENOENT
```

which means the requested file or directory does not exist.

Another example:

```text
openat(AT_FDCWD, "/root/secret.txt", O_RDONLY)
    = -1 EACCES (Permission denied)
```

Now we know:

> The program attempted to access the file, but permission checks prevented the operation.

Failures can reveal program logic.

Imagine:

```text
openat(..., "/etc/app.conf") = -1 ENOENT
openat(..., "/usr/local/etc/app.conf") = -1 ENOENT
openat(..., "/tmp/app.conf") = 3
```

We can reasonably infer that the program tried several locations until one succeeded.

We didn't need the source code to observe that behavior.

<img width="360" height="502" alt="image" src="https://github.com/user-attachments/assets/1da88d9e-bde7-4c31-9f42-442898c9b462" />

 **Failed system calls can reveal decision-making inside a program.**


## Let's make the trace easier to read

The complete trace is useful, but sometimes we don't want everything.

If we're interested in filesystem activity:

```bash
strace -e trace=file ./demo
```

Now the output focuses on file-related system calls.

If we're interested in process operations:

```bash
strace -e trace=process ./demo
```

For memory:

```bash
strace -e trace=memory ./demo
```

For networking:

```bash
strace -e trace=network ./demo
```

Or we can select specific calls:

```bash
strace -e openat,write,close ./demo
```

This is extremely useful during analysis.

Instead of staring at thousands of lines, we can ask a much more specific question:

> **Show me the part of the program's behavior I'm interested in.**

---

## Following child processes

Our current program doesn't create another process, but this is important enough to demonstrate.

Suppose a program creates a child.

Running:

```bash
strace ./program
```

may not give us the complete picture.

We can use:

```bash
strace -f ./program
```

The `-f` option tells `strace` to follow processes created by the traced process.

Conceptually:

```text
                Parent
                  │
                  │ clone()/fork()
                  ▼
                Child
                  │
                  │ execve()
                  ▼
             Another program
```

Now the activity of the child becomes part of our investigation.

This matters because a program may delegate important work to another process.

---

## Saving the trace

For a real investigation, we don't always want the output scrolling past our terminal.

We can save it:

```bash
strace -o trace.txt ./demo
```

Now:

```bash
less trace.txt
```

We can search:

```bash
grep 'openat' trace.txt
```

or:

```bash
grep 'execve' trace.txt
```

or:

```bash
grep 'mmap' trace.txt
```

We can also combine several interesting calls:

```bash
grep -E 'openat|execve|connect|socket' trace.txt
```

The trace becomes something we can investigate rather than something we simply watch disappear up the terminal.

---

## Seeing more data

Sometimes `strace` doesn't show the complete contents of a string or buffer.

For example, output may be shortened.

We can increase the displayed string size:

```bash
strace -s 200 ./demo
```

The important point is that `-s` controls how much string data `strace` displays.

It doesn't mean `strace` suddenly understands the application's data format.

It simply lets us see more of the data being passed through the syscall interface.

---

## Timestamps make the trace a timeline

We can add timestamps:

```bash
strace -tt ./demo
```

Now we may see:

```text
10:42:31.123456 openat(...) = 3
10:42:31.123521 write(...) = 14
10:42:31.123549 close(3) = 0
```

Now the trace isn't just:

> What happened?

It becomes:

> **What happened, and when?**

We can also ask how long a syscall took:

```bash
strace -T ./demo
```

which can produce:

```text
openat(...) = 3 <0.000021>
write(...) = 14 <0.000008>
close(3) = 0 <0.000006>
```

The value in angle brackets represents the time spent inside that syscall.

This can be useful for identifying calls that block or take significantly longer than others.

---

## What does `-c` tell us?

Instead of looking at every individual syscall, we can ask `strace` for statistics:

```bash
strace -c ./demo
```

The result includes information such as:

```text
% time     seconds  usecs/call     calls    errors syscall
------ ----------- ----------- --------- --------- ----------------
  0.00    0.000000           0         1           read
  0.00    0.000000           0         2           write
  0.00    0.000000           0         3           close
  0.00    0.000000           0         3           fstat
  0.00    0.000000           0         8           mmap
  0.00    0.000000           0         3           mprotect
  0.00    0.000000           0         1           munmap
  0.00    0.000000           0         3           brk
  0.00    0.000000           0         2           pread64
  0.00    0.000000           0         1         1 access
  0.00    0.000000           0         1           execve
  0.00    0.000000           0         1           arch_prctl
  0.00    0.000000           0         1           set_tid_address
  0.00    0.000000           0         3           openat
  0.00    0.000000           0         1           set_robust_list
  0.00    0.000000           0         1           prlimit64
  0.00    0.000000           0         1           getrandom
  0.00    0.000000           0         1           rseq
------ ----------- ----------- --------- --------- ----------------
100.00    0.000000           0        37         1 total

```

This provides a summary of syscall activity.

Rather than asking:

> "What happened on this particular line?"

we can ask:

> "What kind of system-call activity dominated this execution?"

It's a different perspective on the same program.

<img width="1536" height="1024" alt="image" src="https://github.com/user-attachments/assets/98be312d-ea26-4605-8543-db73f60d6f90" />


## A useful way to think about the entire trace

At this point, we can stop thinking of `strace` output as a giant list.

Think of it as a timeline.

For our tiny program, the interesting portion looks roughly like:

```text
execve()
   │
   ▼
Program initialization
   │
   ├── mmap()
   ├── openat()
   ├── read()
   └── close()
   │
   ▼
Application code
   │
   ▼
openat("/tmp/demo.txt")
   │
   ▼
write(3, ...)
   │
   ▼
close(3)
   │
   ▼
write(1, "Done!\n", ...)
   │
   ▼
exit_group()
```

Now we have transformed raw syscall output into a behavioral model.

---

## One small program, several layers of behavior

Our source code was tiny:

```c
int fd = open("/tmp/demo.txt", O_CREAT | O_WRONLY, 0644);

write(fd, "Hello strace!\n", 14);

close(fd);

printf("Done!\n");
```

But the runtime behavior involves several layers:


<img width="592" height="512" alt="image" src="https://github.com/user-attachments/assets/b72a428b-028a-4258-ba15-94b161831591" />



This is why `strace` is so useful.

It gives us a view that sits between the high-level source code and the low-level CPU instructions.

---

## A syscall is not enough — the sequence matters

Consider:

```text
socket()
```

By itself, that tells us:

> A socket was created.

But:

```text
socket()
connect()
sendto()
recvfrom()
close()
```

tells us much more.

We can interpret the sequence as:

```text
Create socket
      │
      ▼
Establish connection
      │
      ▼
Send data
      │
      ▼
Receive data
      │
      ▼
Close connection
```

The same idea applies to files:

```text
openat()
   ↓
read()
   ↓
close()
```

This tells a story.

And processes:

```text
clone()
   ↓
execve()
   ↓
wait4()
```

tells another story.

The real value of `strace` comes from **connecting these events together**.

---

## What `strace` can reveal

From a runtime trace, we can often answer questions such as:

```text
What files does the program access?

What configuration paths does it try?

What processes does it create?

Does it execute another program?

Does it communicate over the network?

What memory mappings does it create?

What permissions does it request?

Which operations fail?

What resources does it open?

How long do particular system calls take?

How frequently are different syscalls used?
```

These are extremely useful questions when investigating an unknown Linux executable.

---

## But there is a boundary

There is also something important that `strace` does **not** show directly.

Suppose the program performs:

```c
int result = complicated_calculation();
```

If that calculation happens entirely in user space, there may be no corresponding syscall for the calculation itself.

So we should visualize `strace` like this:


<img width="319" height="492" alt="image" src="https://github.com/user-attachments/assets/6fd58ab8-8e7e-4414-9fe3-dae5ac22f047" />


`strace` gives us excellent visibility into the **program-to-kernel interaction**.

It does not expose every instruction executed by the program.

That distinction is important.

---

## A practical `strace` session

For a real binary, a useful starting point might be:

```bash
strace -f -tt -T -s 200 -o trace.txt ./sample
```

This combines several useful capabilities:

```text
-f    → follow child processes
-tt   → detailed timestamps
-T    → syscall execution time
-s    → display longer strings
-o    → save the trace
```

Then we can investigate the resulting file:

```bash
less trace.txt
```

Search for files:

```bash
grep 'openat' trace.txt
```

Search for process execution:

```bash
grep 'execve' trace.txt
```

Search for networking:

```bash
grep -E 'socket|connect|send|recv' trace.txt
```

Search for memory operations:

```bash
grep -E 'mmap|mprotect|munmap' trace.txt
```

We have now moved from simply **running `strace`** to actually **investigating a trace**.

<img width="592" height="472" alt="image" src="https://github.com/user-attachments/assets/5e417a48-bd39-42d5-a9ac-d60f84ba6921" />


## The important mindset

When you first start using `strace`, it is tempting to memorize syscall names.

That's not the goal.

Instead, ask questions.

When you see:

```text
openat(...)
```

ask:

> **What is the program opening?**

When you see:

```text
read(...)
```

ask:

> **What is it reading, and from which descriptor?**

When you see:

```text
write(...)
```

ask:

> **Where is the data going?**

When you see:

```text
execve(...)
```

ask:

> **What program is being executed?**

When you see:

```text
connect(...)
```

ask:

> **Where is the process connecting?**

When you see:

```text
mmap(...)
```

ask:

> **What kind of memory is being mapped, and with what permissions?**

And when you see:

```text
= -1
```

ask:

> **Why did the operation fail?**

That mindset is far more valuable than memorizing a syscall cheat sheet.

---

## From lines to behavior

At the beginning of this article, we had:

```bash
./demo
```

and saw:

```text
Done!
```

After tracing it, we can describe much more:


<img width="1148" height="786" alt="image" src="https://github.com/user-attachments/assets/9eaae00e-4483-40af-ba14-af5316a31a2d" />


The source code was small.

The operating-system interaction was much larger.

And that's exactly why `strace` is useful.

---

## Final thoughts

`strace` doesn't magically reveal the source code of a program.

Instead, it gives us something different:

**evidence of what the process asks the operating system to do.**

Files being opened.

Data being read.

Data being written.

Processes being created.

Programs being executed.

Memory being mapped.

Sockets being created.

Connections being attempted.

Operations succeeding.

Operations failing.

And all of these events can be connected into a timeline.

The most useful question is therefore not:

> **"What does this syscall mean?"**

but:

> **"What does this sequence of syscalls tell me about the program?"**

Once you start looking at `strace` that way, the output stops looking like a wall of cryptic text.

It starts looking like a conversation between a program and Linux.

And `strace` lets us listen in.

---

### Quick reference

```text
strace ./program
    Basic tracing

strace -f ./program
    Follow child processes

strace -o trace.txt ./program
    Save trace to a file

strace -s 200 ./program
    Display longer strings

strace -tt ./program
    Detailed timestamps

strace -T ./program
    Show syscall duration

strace -c ./program
    Show syscall statistics

strace -e trace=file ./program
    Trace filesystem-related calls

strace -e trace=network ./program
    Trace network-related calls

strace -e trace=process ./program
    Trace process-related calls

strace -e trace=memory ./program
    Trace memory-related calls
```

**The program may hide its implementation, but its interaction with the operating system leaves traces.**

That is where `strace` becomes a powerful lens for understanding Linux programs.
