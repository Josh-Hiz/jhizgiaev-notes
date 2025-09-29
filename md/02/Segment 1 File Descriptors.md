# Segment 1 File Descriptors

Class: CS631

Subject: # Advanced Programming in the UNIX Environment

Date: 2025-09-14

Teacher: Jan Schaumann

## File Descriptors

- A file descriptor is a small, non-negative integer which identifies a file to the kernel.
- `stdin, stdout, stderr` are 0,1, and 2 respectively, but relying on magic numbers is bad practice, use STDIN_FILENO, STDOUT_FILENO, and STDERR_FILENO instead.

## Lessons

"How many file descriptors can a process open?"

- You can't always rely on values being defined for you
- The defined value may not actually apply to your process
- Constants required by the standard may, while present, not actually be useful
- Use sysconf(3) or getrlimit(2) for runtime values, but keep in mind that the result may change from invocation to the next