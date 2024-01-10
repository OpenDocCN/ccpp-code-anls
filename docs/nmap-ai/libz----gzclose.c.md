# `nmap\libz\gzclose.c`

```
/* gzclose.c -- zlib gzclose() function
 * zlib gzclose()函数的实现
 * 版权所有 (C) 2004, 2010 Mark Adler
 * 分发和使用条件，请参见zlib.h中的版权声明
 */

#include "gzguts.h"

/* gzclose()被放在一个单独的文件中，这样只有在需要时才链接它。
   这样可以使用其他gzclose函数，以避免链接不需要的压缩或解压缩例程。 */
int ZEXPORT gzclose(file)
    gzFile file;
{
#ifndef NO_GZCOMPRESS
    gz_statep state;

    if (file == NULL)
        return Z_STREAM_ERROR;
    state = (gz_statep)file;

    return state->mode == GZ_READ ? gzclose_r(file) : gzclose_w(file);
#else
    return gzclose_r(file);
#endif
}
```