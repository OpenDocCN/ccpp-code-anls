# `nmap\libz\examples\gzlog.h`

```cpp
/* gzlog.h
  版权所有 2004 年，2008 年，2012 年 Mark Adler，保留所有权利
  版本 2.2，2012 年 8 月 14 日

  本软件按原样提供，不提供任何明示或暗示的保证。作者不对因使用本软件而产生的任何损害承担责任。

  任何人都可以使用本软件进行任何用途，包括商业应用，并且可以自由修改和重新分发，但需遵守以下限制：

  1. 本软件的来源不得被歪曲；您不得声称自己编写了原始软件。如果您在产品中使用本软件，产品文档中的致谢将不是必需的，但会受到赞赏。
  2. 修改后的源代码必须明确标记，并且不得被误传为原始软件。
  3. 任何源分发中都不得删除或更改本通知。

  Mark Adler    madler@alumni.caltech.edu
 */

/* 版本历史:
   1.0  2004 年 11 月 26 日  第一个版本
   2.0  2008 年 4 月 25 日  为了恢复中断的操作进行了完全重新设计
                             接口略微改变，现在路径是一个前缀
                             压缩现在在 gzlog_write() 中按需进行
                             gzlog_write() 现在始终将日志文件保持为有效的 gzip
   2.1  2012 年 7 月 8 日   修复了 gzlog_compress() 和 gzlog_write() 中的参数检查
   2.2  2012 年 8 月 14 日  清理了有符号比较
 */
/*
   The gzlog object allows writing short messages to a gzipped log file,
   opening the log file locked for small bursts, and then closing it.  The log
   object works by appending stored (uncompressed) data to the gzip file until
   1 MB has been accumulated.  At that time, the stored data is compressed, and
   replaces the uncompressed data in the file.  The log file is truncated to
   its new size at that time.  After each write operation, the log file is a
   valid gzip file that can decompressed to recover what was written.

   The gzlog operations can be interrupted at any point due to an application or
   system crash, and the log file will be recovered the next time the log is
   opened with gzlog_open().
 */

#ifndef GZLOG_H
#define GZLOG_H

/* gzlog object type */
typedef void gzlog;

/* Open a gzlog object, creating the log file if it does not exist.  Return
   NULL on error.  Note that gzlog_open() could take a while to complete if it
   has to wait to verify that a lock is stale (possibly for five minutes), or
   if there is significant contention with other instantiations of this object
   when locking the resource.  path is the prefix of the file names created by
   this object.  If path is "foo", then the log file will be "foo.gz", and
   other auxiliary files will be created and destroyed during the process:
   "foo.dict" for a compression dictionary, "foo.temp" for a temporary (next)
   dictionary, "foo.add" for data being added or compressed, "foo.lock" for the
   lock file, and "foo.repairs" to log recovery operations performed due to
   interrupted gzlog operations.  A gzlog_open() followed by a gzlog_close()
   will recover a previously interrupted operation, if any. */
gzlog *gzlog_open(char *path);
/* Write to a gzlog object.  Return zero on success, -1 if there is a file i/o
   error on any of the gzlog files (this should not happen if gzlog_open()
   succeeded, unless the device has run out of space or leftover auxiliary
   files have permissions or ownership that prevent their use), -2 if there is
   a memory allocation failure, or -3 if the log argument is invalid (e.g. if
   it was not created by gzlog_open()).  This function will write data to the
   file uncompressed, until 1 MB has been accumulated, at which time that data
   will be compressed.  The log file will be a valid gzip file upon successful
   return. */
int gzlog_write(gzlog *log, void *data, size_t len);

/* Force compression of any uncompressed data in the log.  This should be used
   sparingly, if at all.  The main application would be when a log file will
   not be appended to again.  If this is used to compress frequently while
   appending, it will both significantly increase the execution time and
   reduce the compression ratio.  The return codes are the same as for
   gzlog_write(). */
int gzlog_compress(gzlog *log);

/* Close a gzlog object.  Return zero on success, -3 if the log argument is
   invalid.  The log object is freed, and so cannot be referenced again. */
int gzlog_close(gzlog *log);

#endif
```