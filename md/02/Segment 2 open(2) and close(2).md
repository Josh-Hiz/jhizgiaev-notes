# Segment 2 open(2) and close(2)

Class: CS631

Subject: # Advanced Programming in the UNIX Environment

Date: 2025-09-14

Teacher: Jan Schaumann

## Standard I/O

Almost all UNIX file I/O can be performed using these five functions:
1. open(2)
2. close(2)
3. read(2)
4. write(2)
5. lseek(2)

To create a file, use the `creat(2)` system call. This returns a file descriptor that is in write-only mode. This was made obsolete with `open(2)`. The command is the same as:
```c
open(path, O_CREAT | O_TRUNC | O_WRONLY, mode);
```

```c
int open(const char *pathname, int oflag, .../* mode_t mode */); 
```
*oflag* must be one (and only one) of:
- O_RDONLY - read only
- O_WRONLY - write only
- O_RDWR  - read and write
and they more be OR'd with any of these:
- O_APPEND - append on each write
- O_CREAT - create the file if it doesn't exist; requires *mode* argument
- O_EXCL - error if O_CREAT and file already exists (atomic)
- O_TRUNC - truncate size to 0
- O_NONBLOCK - do not block on open or for data to become available
- O_SYNC - wait for physical I/O to complete
additional *oflags* may be supported by some platforms:
- O_DIRECTORY - if path resolves to a non-directory file, fail and set errno to ENOTDIR
- O_DSYNC - wait for physical I/O for data, except file attributes
- O_EXEC - open file for execute only, fail if it is a directory
- O_NOFOLLOW - do not follow symlinks
- O_PATH - obtain a file descriptor purely for fd-level operations
- O_RSYNC - block read operations on any pending writes
- O_SEARCH - open for search only, fail if it is a regular file
- ...

There is also openat(2):
```c
int openat(int dirfd, const char *pathname, int oflag, .../* mode_t mode */);
```
openat(2) is used to handle relative pathnames from different working directories in an atomic fashion. Here, *pathname* is determined relative to the directory associated with the file descriptor *fd* instead of the current working directory. 

open(2) may fail for a large number of reasons, some common include:
1. EEXIST - O_CREAT | O_EXCL was specified, but the file exists
2. EMFILE - process has already reached max number of open file descriptors
3. ENOENT - file does not exist
4. EPERM - lack of permissions
5. ...

We can close the file descriptor with close(2)
- closing a file descriptor releases any record locks on that file
- file descriptors not explicitly closed are closed by the kernel when process terminates
- to avoid leaking file descriptors, always close(2) them within the same scope
- close can fail, and a -1 is returned on error

This difference arises from filesystem support for sparse files and differing block-reporting units. On Linux (ext4), the lseek then write produces a true sparse file, only a few blocks are allocated. On NetBSD (UFS/FFS), the file is also sparse but incurs additional metadata/indirect-block overhead and uses 512-byte reporting units, so “96 blocks” ≈ 48 KiB. On your macOS volume, the hole was materialized into real zero bytes—typical of volumes that don’t preserve holes (e.g., HFS+ or certain network/backup mounts)—so roughly 20,000 × 512 B ≈ 10 MB were allocated; on APFS one would ordinarily see Linux-like sparse behavior.