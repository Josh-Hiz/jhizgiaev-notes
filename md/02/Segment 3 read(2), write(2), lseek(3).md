# Segment 3 read(2), write(2), lseek(3)

Class: CS631

Subject: # Advanced Programming in the UNIX Environment

Date: 2025-09-14

Teacher: Jan Schaumann

## Reading files

```c
#include <unistd.h>
ssize_t read(int fd, void *buf, size_t num);
```

- Returns the number of bytes read; 0 on EOF, -1 on error
- `read` begins reading at the current offset, and increments the offset by the number of bytes actually read
- There can be several cases where read returns fewer than the number of bytes requested, for example:
	1. EOF reached before requested number of bytes have been read
	2. reading from a network, buffering can cause delays in arrival of data. There may not be as much data available as you have requested, even though we have not yet reached EOF:
		1. For example, if you're reading from a socket, it's possible that not enough data has arrived yet, so read(2) might return to you however many bytes _are_ available, but upon a subsequent call, there may be new data available
	3. interruption by a signal
	4. record-oriented devices (magtape) may return data one record at a time

## Writing data

```c
#include <unistd.h>
ssize_t write(int fd, void *buf, size_t num);
```

- Returns the number of bytes written if OK, -1 on error
- `write` returns the number of bytes written
- For regular files, write begins writing at the current offset (unless O_APPEND has been specified, in which case the offset is first set to the end of the file)
- After the write, the offset is adjusted by the number of bytes actually written

Some manual pages note that if the real user is not the super-user, then write() clears the set-user-id bit on a file. This prevents penetration of system security by a user who "captures" a writeable set-user-id file owned by the super user.

## L-Seek

```c
#include <sys/types.h>
#include <fcntl.h>
off_t lseek(int fd, off_t offset, int whence);
```

- Returns a new offset if OK; -1 on error
- Upon completion, lseek(2) will move the current offset as indicated and return to you the new offset; a value of -1 indicates an error, as usual.
- The value of `whence` determines how the `offset` is used
	- SEEK_SET bytes from the beginning of the file
	- SEEK_CUR bytes from the current file position
	- SEEK_END bytes from the end of the file
- You can do some weird things with lseek
	- seek to a negative offset
		- So if you want to step back a few bytes, you specify, say, -32 as the offset and SEEK_CUR as 'whence'.
		- Or say you wish to go to a position 64 bytes before the end of the file: pass in -64 and SEEK_END, and there you go.
	- seek 0 bytes from the current position
		- This can be useful when you want to jump to the beginning of the file (lseek 0, SEEK_SET) or the end of the file (lseek 0, SEEK_END), but of course you can also say "go 0 bytes from the current position"
	- seek past the end of the file
		- By trying to seek 0 bytes off the current position, we can determine whether or not the file descriptor we have is seekable at all.
- We cannot seek PIPE's or FIFO's either

### Important note about sparse files

You _can_ seek past the end of a file and thereby create a so-called sparse file, a file with a hole.  But the file system must support this, or else you just get a file with lots of NULL bytes written to disk.

If the filesystem supports sparse files, then the kernel will supply NULL bytes when you read the hole, but they will not actually be on disk.

Different versions of cp(1) or other tools may or may not be able to support sparse files, although that is mostly based on the guess that if you encounter a sufficiently large sequence of NULL bytes the input was probably a sparse file.

## Notes about I/O efficiency

Example program:

```c
#define BUFFSIZE 32768

while((n = read(STDIN_FILENO, buf, BUFFSIZE)) > 0){
	if(write(STDOUT_FILENO, buf, n) != n) {
		fprintf(stderr, "Unable to write: %s\n", strerror(errno));
		exit(EXIT_FAILURE);
	}
}
```

We are processing data from stdin in a loop, reading it into a buffer of size BUFFSIZE.  That means that the smaller BUFFSIZE is, the more iterations of the loop we perform and the more calls to both read(2) and write(2) we perform.  Likewise, as we increase BUFFSIZE, we reduce the number of iterations, reduce the number of syscalls, and thus become more efficient.

The reason that we can't keep gaining efficiency by increasing the buffer has to do with the
filesystem.  The filesystem has a fixed blocksize in which it reads data from the disk, and no matter how large your buffer is, the filesystem cannot read more efficiently than whatever its blocksize is.

There was a way to determine the optimal I/O size, i.e., the filesystem blocksize, of a given
file by running the following:

```c
stat -f "%k" tmp/file1
```

Note that the command-line syntax for the stat(1) command differs between Unix versions -- Linux, for example, uses "stat -c "%o" -- always check your manual page.

Note that of course the results of our little benchmark here will differ from filesystem to filesystem and OS to OS. 