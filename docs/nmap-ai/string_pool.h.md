# `nmap\string_pool.h`

```
#ifndef STRING_POOL_H
#define STRING_POOL_H

/* The OS database consists of many small strings, many of which appear
   thousands of times. It pays to allocate memory only once for each unique
   string, and have all references point at the one allocated value. */

/* Store a string uniquely. The first time this function is called with a
   certain string, it allocates memory and stores a copy of the string in a
   static pool. Thereafter it will return a pointer to the saved string instead
   of allocating memory for an identical one. */
const char *string_pool_insert(const char *s);

/* Format a string with sprintf and insert it with string_pool_insert. */
const char *string_pool_sprintf(const char *fmt, ...);

/* Store the string in the interval [s,t) */
const char *string_pool_substr(const char *s, const char *t);

/* Store the string in the interval [s,t), stripping whitespace from both ends. */
const char *string_pool_substr_strip(const char *s, const char *t);

/* Skip over whitespace to find the beginning of a word, then read until the
   next whitespace character. Returns NULL if only whitespace is found. */
const char *string_pool_strip_word(const char *s, const char *end);

#endif // STRING_POOL_H
```