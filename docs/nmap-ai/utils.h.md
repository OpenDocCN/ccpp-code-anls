# `nmap\utils.h`

```
/* $Id$ */

#ifndef UTILS_H
#define UTILS_H

#ifndef WIN32
#include <sys/mman.h>
#endif

#include "nbase.h"
#include <assert.h>

/* Arithmatic difference modulo 2^32 */
#ifndef MOD_DIFF
#define MOD_DIFF(a,b) ((u32) (MIN((u32)(a) - (u32 ) (b), (u32 )(b) - (u32) (a))))
#endif

/* Arithmatic difference modulo 2^16 */
#ifndef MOD_DIFF_USHORT
#define MOD_DIFF_USHORT(a,b) ((MIN((unsigned short)((unsigned short)(a) - (unsigned short ) (b)), (unsigned short) ((unsigned short )(b) - (unsigned short) (a)))))
#endif
#ifndef FALSE
#define FALSE 0
#endif
#ifndef TRUE
#define TRUE 1
#endif

#define MAX_PARSE_ARGS 254 /* +1 for integrity checking + 1 for null term */

/* Return num if it is between min and max.  Otherwise return min or max
   (whichever is closest to num). */
template<class T> T box(T bmin, T bmax, T bnum) {
  assert(bmin <= bmax);
  if (bnum >= bmax)
    return bmax;
  if (bnum <= bmin)
    return bmin;
  return bnum;
}

int wildtest(const char *wild, const char *test);

void nmap_hexdump(const unsigned char *cp, unsigned int length);

void genfry(unsigned char *arr, int elem_sz, int num_elem);
void shortfry(unsigned short *arr, int num_elem);
char *chomp(char *string);

int Send(int sd, const void *msg, size_t len, int flags);

int arg_parse(const char *command, char ***argv);
void arg_parse_free(char **argv);

char *cstring_unescape(char *str, unsigned int *len);

void bintohexstr(char *buf, int buflen, const char *src, int srclen);

u8 *parse_hex_string(const char *str, size_t *outlen);

int cpe_get_part(const char *cpe);

char *mmapfile(char *fname, s64 *length, int openflags);

#ifdef WIN32
int win32_munmap(char *filestr, int filelen);
#endif /* WIN32 */

#endif /* UTILS_H */
```