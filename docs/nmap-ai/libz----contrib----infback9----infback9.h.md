# `nmap\libz\contrib\infback9\infback9.h`

```cpp
/* infback9.h -- header for using inflateBack9 functions
 * 定义了使用 inflateBack9 函数的头文件
 * 版权声明
 */

/*
 * This header file and associated patches provide a decoder for PKWare's
 * undocumented deflate64 compression method (method 9).  Use with infback9.c,
 * inftree9.h, inftree9.c, and inffix9.h.  These patches are not supported.
 * 这个头文件和相关的补丁提供了用于解码 PKWare 的未记录的 deflate64 压缩方法（方法 9）的解码器。与 infback9.c、inftree9.h、inftree9.c 和 inffix9.h 一起使用。这些补丁不受支持。
 * This should be compiled with zlib, since it uses zutil.h and zutil.o.
 * 这应该与 zlib 一起编译，因为它使用了 zutil.h 和 zutil.o。
 * This code has not yet been tested on 16-bit architectures.  See the
 * comments in zlib.h for inflateBack() usage.  These functions are used
 * identically, except that there is no windowBits parameter, and a 64K
 * window must be provided.  Also if int's are 16 bits, then a zero for
 * the third parameter of the "out" function actually means 65536UL.
 * 这段代码尚未在 16 位架构上进行测试。查看 zlib.h 中对 inflateBack() 的用法的注释。这些函数的用法完全相同，只是没有 windowBits 参数，必须提供一个 64K 的窗口。另外，如果 int 是 16 位，则 "out" 函数的第三个参数为零实际上表示 65536UL。
 * zlib.h must be included before this header file.
 * 必须在包含这个头文件之前包含 zlib.h。
 */

#ifdef __cplusplus
extern "C" {
#endif

ZEXTERN int ZEXPORT inflateBack9 OF((z_stream FAR *strm,
                                    in_func in, void FAR *in_desc,
                                    out_func out, void FAR *out_desc));
ZEXTERN int ZEXPORT inflateBack9End OF((z_stream FAR *strm));
ZEXTERN int ZEXPORT inflateBack9Init_ OF((z_stream FAR *strm,
                                         unsigned char FAR *window,
                                         const char *version,
                                         int stream_size));
#define inflateBack9Init(strm, window) \
        inflateBack9Init_((strm), (window), \
        ZLIB_VERSION, sizeof(z_stream))

#ifdef __cplusplus
}
#endif
```