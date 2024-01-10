# `nmap\nsock\src\netutils.h`

```
/* $Id$ */

#ifndef NETUTILS_H
#define NETUTILS_H

#ifdef HAVE_CONFIG_H
#include "nsock_config.h"
#include "nbase_config.h"
#endif

#if HAVE_NETINET_IN_H
#include <netinet/in.h>
#endif

#include "nsock_internal.h"

#ifdef WIN32
#include "nbase_winconfig.h"
/* nbase_winunix.h somehow reason.h to get included */
#include "nbase_winunix.h"
#endif

#if HAVE_SYS_UN_H
#include <sys/un.h>
#endif

#if HAVE_SYS_RESOURCE_H
#include <sys/resource.h>
#else
#ifndef rlim_t
#define rlim_t int
#endif
#endif
/* Maximize the number of file descriptors (including sockets) allowed for this
 * process and return that maximum value (note -- you better not actually open
 * this many -- stdin, stdout, other files opened by libraries you use, etc. all
 * count toward this limit.  Leave a little slack */
rlim_t maximize_fdlimit(void);

/* Get the UNIX domain socket path or empty string if the address family != AF_UNIX. */
const char *get_unixsock_path(const struct sockaddr_storage *addr);

/* Get the peer address string. In case of a Unix domain socket, returns the
 * path to UNIX socket, otherwise it returns string containing
 * "<address>:<port>". */
char *get_peeraddr_string(const struct niod *iod);

/* Get the local bind address string. */
char *get_localaddr_string(const struct niod *iod);

#endif /* NETUTILS_H */
```