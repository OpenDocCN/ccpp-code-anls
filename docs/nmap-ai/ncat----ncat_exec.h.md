# `nmap\ncat\ncat_exec.h`

```cpp
/* $Id$ */
#ifndef NCAT_EXEC_H
#define NCAT_EXEC_H

/* fork and exec a child process with netexec. Close the given file descriptor
   in the parent process. Return the child's PID or -1 on error. */
extern int netrun(struct fdinfo *info, char *cmdexec);

/* exec the given command line. Before the exec, redirect stdin, stdout, and
   stderr to the given file descriptor. Never returns. */
extern void netexec(struct fdinfo *info, char *cmdexec);

#ifdef WIN32
/* Set a pseudo-signal handler that is called when a thread representing a
   child process dies. This is only used on Windows. */
extern void set_pseudo_sigchld_handler(void (*handler)(void));
#endif

#endif
```