# Segment 4 File Sharing

Class: CS631

Subject: # Advanced Programming in the UNIX Environment

Date: 2025-09-14

Teacher: Jan Schaumann

## File Sharing

Since UNIX is a multi-user/multi-tasking system, it is conceivable (and useful) if more than one process can act on a single file simultaneously. In order to understand how this is accomplished, we need to examine some kernel data structures which relate to files.
- The kernel maintains a process table, **each process table entry has a table of file descriptor, which contain**:
	- the file descriptor flags (e.g. FD_CLOEXEC, see fcntl(2))
	- a pointer to a file table entry
- the kernel maintains a file table; each entry contains
	- file status flags (O_APPEND, O_SYNC, O_RDONLY, etc.)
	- current offset
	- a pointer to a vnode table entry
- a vnode structure contains
	- vnode information
	- inode information (such as current file size)

This is what it might look like in a ordinary UNIX system:

![image](../Images/Screenshot%202025-09-28%20at%209.16.15%20PM.png)

We see the process table, with each entry pointing to a file table entry, with each of those entries pointing to v-node table entry, which finally contains the inode information as well as, eventually, a pointer to the actual blocks on disk used for the contents of the file.

If a process opens the same file twice, then we'll get data structures as represented here.  Note that the file table entries are distinct, but pointing to the same vnode table entry.

![image](../Images/Screenshot%202025-09-28%20at%209.49.43%20PM.png)

Similarly, if two different processes opened the same file, then things might look like so.

![image](../Images/Screenshot%202025-09-28%20at%2010.07.29%20PM.png)

After each `write(2)` completes, the current file offset in the file table entry is incremented. If the current file offset is larger than the current file size, we change the current file size in the **inode table entry**. 

![image](../Images/Screenshot%202025-09-28%20at%2010.17.34%20PM.png)

If file was opened with O_APPEND, set corresponding flag in file status flags in file table. For each write, the current file offset is first set to the **current file size from the inode entry**. 

![image](../Images/Screenshot%202025-09-28%20at%2010.20.52%20PM.png)

`lseek(2)` merely adjusts the current file offset in file table entry. To seek to the end of a file, just **copy current file size into current file offset.** To seek to the beginning of the file, simply set the offset to 0.

![image](../Images/Screenshot%202025-09-28%20at%2010.24.55%20PM.png)

But if we have multiple processes possibly accessing the same files at the same time, then we quickly require atomic operations.

As a reminder, atomic operations are operations that are guaranteed to complete in their entirety without another process being able to interfere, or not at all.

![image](../Images/Screenshot%202025-09-28%20at%2010.30.07%20PM.png)

- Let's imagine two processes, both trying to append to their respective buffer contents to the same file.
- Process 1 wants to append "P1 always writes to the end of the file", while process 2 wants to append "Not true! This time P2 wins, P1 loses. Haha!"
- Ok, so let's pretend process 1 gets the CPU first and begins by calling 'lseek(fd, 0, SEEK_END)'.
- As discussed, this copies the current file size to the current offset in the file table entry of process 1.
- But now the scheduler decides that it's time for process 2 to get the CPU, and _that_ process calls 'lseek(fd, 0, SEEK_END)', so of course it copies the current file size -- 128 -- to the current offset of process 2's file table entry.
- Process 2 then issues its write, successfully appending data to the end.
- The new offset is now 173, and since that's larger than 128, it copies 173 to the file size in the inode.
- Content with having written its data, process two happily exits.
- Now process 1 gets the CPU again, continuing where it left off.  Which was right after it had called lseek(2).
- So with it's current offset set to 128, it will write the contents of buf1 at position 128, overwriting the bytes previously written by process 2.
- It adjusts its offset and exits.  And we're left with corrupted data at the end of the file.
- This illustrates the necessity for an atomic operation to append data, and the file status flag O_APPEND solves this problem.

## Atomic Operations

O_APPEND solves the case for writing to the end, but what if we want to write atomically anywhere else in the file.

```c
#include <unistd.h>
ssize_t pread(int fd, void *buf, size_t num, off_t offset);
ssize_t pwrite(int fd, void *buf, size_t num, off_t offset);
```

- Will read and write a file at a specific offset atomically
- Note: The current offset is not changed (these calls will NOT change the current file offset pointed by the file descriptor)
- Returns the number of bytes read/written, -1 on error

## Shell redirection & file status flags

### 1) Basic output redirection (`>`)

- `>` redirects **stdout (fd 1)** to a file using `open(2)` with flags: `O_WRONLY | O_CREAT | O_TRUNC`.
- If the file doesn’t exist → it’s created. If it exists → it’s **truncated to 0** *before* the program runs.

```sh
# Start clean
printf 'hello\n' >file
cat file
# hello
# Overwrite
printf 'WORLD\n' >file
cat file
# WORLD
```

**Why `ls` might say “0 bytes” while writing?**

The shell performs `O_TRUNC` **before** executing the command. If you do:

```sh
ls -l file >file
```

the shell truncates `file` first (size 0). Then `ls` stat’s it and prints “0”, which goes to `file` (not the terminal).

### 2) Append redirection (`>>`)

- `>>` redirects stdout with `O_WRONLY | O_CREAT | O_APPEND`.
- In `O_APPEND` mode, each `write(2)` is appended atomically to the end of the file.

```sh
printf 'first\n' >file
printf 'second\n' >>file
cat file
# first
# second
```

### 3) Redirecting multiple streams (stdout vs stderr)

- **stdout** = fd **1**
- **stderr** = fd **2**
- By default both go to the terminal.
#### 3.1 Redirect only stderr

```sh
# Send errors to the bit bucket; stdout still on terminal
ls no_such_dir 2>/dev/null
```
#### 3.2 Redirect only stdout

```sh

# Send normal output to file; errors still on terminal

ls /etc >file

```
### 4) Truncation & race when redirecting both to the *same path* (wrong way)

If you naively do:
```sh
# BAD: two independent opens on the same path
# stdout: >file (O_TRUNC)
# stderr: 2>file (O_TRUNC or O_APPEND, but still a separate open)
ls /etc no_such_dir >file 2>file
```

What happens:
- The shell performs **two separate `open(2)` calls** on `file`
- Each open (with `O_TRUNC` for `>file`) sets size/offset to 0
- Outputs from fd 1 and fd 2 race and **overwrite** each other’s bytes because they are **distinct descriptors referring to the same inode but with independent file offsets**.
- Result: partial/overwritten content (symptoms: truncated or interleaved lines, “missing” stderr or stdout).

Even mixing append for one side doesn’t fix the race if the other side truncates first:

```sh
# Still problematic
ls /etc no_such_dir >>file 2>file
```
### 5) Correct way: make stderr follow stdout’s *destination* (descriptor duplication)

Use **descriptor duplication** so stderr goes to **wherever stdout currently goes**, rather than opening the path twice.

```sh
# Send stdout to file, then duplicate fd 1 onto fd 2
ls /etc no_such_dir >file 2>&1
```

- `2>&1` means: **make fd 2 a duplicate of fd 1** (both fds now refer to the **same open file description**, sharing offset/flags).
- Because there is **one open** with **one file offset**, writes from fd 1 and fd 2 won’t overwrite each other via competing offsets; appends use the same description/offset behavior.

**Order matters**:

```sh
# GOOD: stdout first, then make stderr follow it
cmd >file 2>&1
# BAD: duplicate first (stderr->stdout still points to terminal), then move stdout
cmd 2>&1 >file
# Here stderr remains on the terminal while stdout goes to file.
```

### 6) Why stderr often appears “first”

- `stderr` is typically **unbuffered**; `stdout` is often **line-buffered** (when to a terminal) or **fully buffered** (when to a file/pipe).
- So when both point to a terminal, you’ll often **see errors first** even if the program writes later. This is about **stdio buffering**, not kernel scheduling.
### 7) Piping both stdout and stderr

By design, a simple pipe only connects **stdout** of the left command to **stdin** of the right command; **stderr** remains on the terminal unless you redirect it.

```sh
# Pipe both stdout and stderr into grep
cmd 2>&1 | grep something
```

Explanation:
- Shell creates a pipe, connects cmd’s **stdout** to the pipe.
- `2>&1` duplicates stderr onto **where stdout currently points** (the pipe).
- Now **both** streams flow through the pipe to `grep`.
### 8) “How do we ‘open whatever stdout is’?” (it’s not `open(2)`, it’s `dup2(2)`)

- Redirections using a pathname (`> file`, `>> file`) use `open(2)`
- Redirections that copy one fd to another (`2>&1`) use **descriptor duplication**, typically `dup2(2)` (or `dup3(2)`):
- Conceptually: `dup2(1, 2);` // “make fd 2 refer to the same open file description as fd 1”
- This is why `2>&1` follows stdout *wherever it was pointed at* (file, pipe, socket), without reopening a path.
### 9) Quick-reference

```sh
# Overwrite file with stdout
cmd >file # O_CREAT|O_TRUNC
# Append stdout
cmd >>file # O_CREAT|O_APPEND
# Discard stderr
cmd 2>/dev/null
# Send stdout to file and make stderr go to the same place
cmd >file 2>&1 # order matters!
# WRONG: stderr duplicates stdout (to terminal), then stdout goes to file
cmd 2>&1 >file # stderr stays on terminal
# Pipe both stdout and stderr
cmd 2>&1 | other
# Keep stdout to terminal, capture only stderr
cmd 2>errs.txt
```

## File Descriptor Duplication

The dup(2) system call duplicates an existing file descriptor, and you get back a second file handle pointing to the same file table entry as the first one.

```c
#include <unistd.h>
int dup(int oldfd);
int dup2(int oldfd, int newfd);
```

- Returns newfd, -1 on error

Note the difference in scope of the file descriptor flags -- on the left, in the process table -- and the file status flags -- in the middle, in the file table.  Unlike in the case where a process has called 'open(2)' on the same files multiple times and gets back file descriptors pointing to distinct file table entries that then point to the same vnode table entry, here we have the duplicated file descriptor pointing to the _same_ file table entry, meaning the two share the offset and file status flags etc.  The new file descriptor is indeed a duplicate of the original.

Now to allow for redirection of an existing file descriptor, we can use 'dup2' -- the existing file descriptor will be closed, and instead pointed to the original's file table entry.

![image](../Images/Screenshot%202025-09-28%20at%2011.07.37%20PM.png)

## File Descriptor Control

### fcntl

```c
#include <fcntl.h>
int fcntl(int fd, int cmd, ...);
```

- Returns depends on cmd, -1 on error
- `fcntl(2)` is one of those "catch-all" functions with a myriad of purposes. Here, they all relate to changing properties of an already opened file. Some of them are:
	- F_DUPFD - duplicate file descriptors
	- F_GETFD - get file descriptor tags
	- F_SETFD - set file descriptor tags
	- F_GETFL - get file status flags
	- F_SETFL - set file status flags
	- and many more

### ioctl

```c
#include <fcntl.h>
int ioctl(int fd, unsigned long request, ...);
```

- Returns depend on cmd, -1 on error
- Another catch-all function, this one is designed to handle device specifics that can't be specified via any of the previous function calls
- Examples include terminal I/O, magtape access, socket I/O, etc