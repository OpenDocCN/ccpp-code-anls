# `stable-diffusion.cpp\thirdparty\stb_image_write.h`

```
/* stb_image_write - v1.16 - public domain - http://nothings.org/stb
   writes out PNG/BMP/TGA/JPEG/HDR images to C stdio - Sean Barrett 2010-2015
                                     no warranty implied; use at your own risk

   Before #including,

       #define STB_IMAGE_WRITE_IMPLEMENTATION

   in the file that you want to have the implementation.

   Will probably not work correctly with strict-aliasing optimizations.

ABOUT:

   This header file is a library for writing images to C stdio or a callback.

   The PNG output is not optimal; it is 20-50% larger than the file
   written by a decent optimizing implementation; though providing a custom
   zlib compress function (see STBIW_ZLIB_COMPRESS) can mitigate that.
   This library is designed for source code compactness and simplicity,
   not optimal image file size or run-time performance.

BUILDING:

   You can #define STBIW_ASSERT(x) before the #include to avoid using assert.h.
   You can #define STBIW_MALLOC(), STBIW_REALLOC(), and STBIW_FREE() to replace
   malloc,realloc,free.
   You can #define STBIW_MEMMOVE() to replace memmove()
   You can #define STBIW_ZLIB_COMPRESS to use a custom zlib-style compress function
   for PNG compression (instead of the builtin one), it must have the following signature:
   unsigned char * my_compress(unsigned char *data, int data_len, int *out_len, int quality);
   The returned data will be freed with STBIW_FREE() (free() by default),
   so it must be heap allocated with STBIW_MALLOC() (malloc() by default),

UNICODE:

   If compiling for Windows and you wish to use Unicode filenames, compile
   with
       #define STBIW_WINDOWS_UTF8
   and pass utf8-encoded filenames. Call stbiw_convert_wchar_to_utf8 to convert
   Windows wchar_t filenames to utf8.
*/
/*
CREDITS:

   Sean Barrett           -    PNG/BMP/TGA
   Baldur Karlsson        -    HDR
   Jean-Sebastien Guay    -    TGA monochrome
   Tim Kelsey             -    misc enhancements
   Alan Hickman           -    TGA RLE
   Emmanuel Julien        -    initial file IO callback implementation
   Jon Olick              -    original jo_jpeg.cpp code
   Daniel Gibson          -    integrate JPEG, allow external zlib
   Aarni Koskela          -    allow choosing PNG filter

   bugfixes:
      github:Chribba
      Guillaume Chereau
      github:jry2
      github:romigrou
      Sergio Gonzalez
      Jonas Karlsson
      Filip Wasil
      Thatcher Ulrich
      github:poppolopoppo
      Patrick Boettcher
      github:xeekworx
      Cap Petschulat
      Simon Rodriguez
      Ivan Tikhonov
      github:ignotion
      Adam Schackart
      Andrew Kensler

LICENSE

  See end of file for license information.
*/

#ifndef INCLUDE_STB_IMAGE_WRITE_H
#define INCLUDE_STB_IMAGE_WRITE_H

#include <stdlib.h>

// if STB_IMAGE_WRITE_STATIC causes problems, try defining STBIWDEF to 'inline' or 'static inline'
#ifndef STBIWDEF
#ifdef STB_IMAGE_WRITE_STATIC
#define STBIWDEF  static
#else
#ifdef __cplusplus
#define STBIWDEF  extern "C"
#else
#define STBIWDEF  extern
#endif
#endif
#endif

#ifndef STB_IMAGE_WRITE_STATIC  // C++ forbids static forward declarations
STBIWDEF int stbi_write_tga_with_rle; // 声明 TGA RLE 写入标志
STBIWDEF int stbi_write_png_compression_level; // 声明 PNG 压缩级别
STBIWDEF int stbi_write_force_png_filter; // 声明 PNG 强制滤波器
#endif

#ifndef STBI_WRITE_NO_STDIO
STBIWDEF int stbi_write_png(char const *filename, int w, int h, int comp, const void  *data, int stride_in_bytes, const char* parameters = NULL); // 声明写入 PNG 文件函数
STBIWDEF int stbi_write_bmp(char const *filename, int w, int h, int comp, const void  *data); // 声明写入 BMP 文件函数
STBIWDEF int stbi_write_tga(char const *filename, int w, int h, int comp, const void  *data); // 声明写入 TGA 文件函数
STBIWDEF int stbi_write_hdr(char const *filename, int w, int h, int comp, const float *data); // 声明写入 HDR 文件函数
// 定义函数 stbi_write_jpg，用于将数据写入 JPEG 文件
STBIWDEF int stbi_write_jpg(char const *filename, int x, int y, int comp, const void  *data, int quality);

// 如果定义了 STBIW_WINDOWS_UTF8，则定义函数 stbiw_convert_wchar_to_utf8，用于将宽字符转换为 UTF-8 字符串
#ifdef STBIW_WINDOWS_UTF8
STBIWDEF int stbiw_convert_wchar_to_utf8(char *buffer, size_t bufferlen, const wchar_t* input);
#endif
#endif

// 定义函数指针类型 stbi_write_func，用于写入数据
typedef void stbi_write_func(void *context, void *data, int size);

// 定义函数 stbi_write_png_to_func，用于将数据写入 PNG 文件
STBIWDEF int stbi_write_png_to_func(stbi_write_func *func, void *context, int w, int h, int comp, const void  *data, int stride_in_bytes);
// 定义函数 stbi_write_bmp_to_func，用于将数据写入 BMP 文件
STBIWDEF int stbi_write_bmp_to_func(stbi_write_func *func, void *context, int w, int h, int comp, const void  *data);
// 定义函数 stbi_write_tga_to_func，用于将数据写入 TGA 文件
STBIWDEF int stbi_write_tga_to_func(stbi_write_func *func, void *context, int w, int h, int comp, const void  *data);
// 定义函数 stbi_write_hdr_to_func，用于将数据写入 HDR 文件
STBIWDEF int stbi_write_hdr_to_func(stbi_write_func *func, void *context, int w, int h, int comp, const float *data);
// 定义函数 stbi_write_jpg_to_func，用于将数据写入 JPEG 文件
STBIWDEF int stbi_write_jpg_to_func(stbi_write_func *func, void *context, int x, int y, int comp, const void  *data, int quality);

// 定义函数 stbi_flip_vertically_on_write，用于在写入时垂直翻转图像
STBIWDEF void stbi_flip_vertically_on_write(int flip_boolean);

#endif//INCLUDE_STB_IMAGE_WRITE_H

// 如果定义了 STB_IMAGE_WRITE_IMPLEMENTATION，则执行以下代码
#ifdef STB_IMAGE_WRITE_IMPLEMENTATION

// 如果在 Windows 平台下
#ifdef _WIN32
   // 防止 CRT 函数安全警告
   #ifndef _CRT_SECURE_NO_WARNINGS
   #define _CRT_SECURE_NO_WARNINGS
   #endif
   // 防止 CRT 非标准函数警告
   #ifndef _CRT_NONSTDC_NO_DEPRECATE
   #define _CRT_NONSTDC_NO_DEPRECATE
   #endif
#endif

// 如果未定义 STBI_WRITE_NO_STDIO，则包含标准输入输出库
#ifndef STBI_WRITE_NO_STDIO
#include <stdio.h>
#endif // STBI_WRITE_NO_STDIO

// 包含可变参数、标准库、字符串处理、数学库
#include <stdarg.h>
#include <stdlib.h>
#include <string.h>
#include <math.h>

// 如果定义了 STBIW_MALLOC 和 STBIW_FREE，并且定义了 STBIW_REALLOC 或 STBIW_REALLOC_SIZED，则通过
// 宏定义检查通过
#if defined(STBIW_MALLOC) && defined(STBIW_FREE) && (defined(STBIW_REALLOC) || defined(STBIW_REALLOC_SIZED))
// ok
// 如果未定义 STBIW_MALLOC 和 STBIW_FREE，并且未定义 STBIW_REALLOC 和 STBIW_REALLOC_SIZED，则通过
// 宏定义检查通过
#elif !defined(STBIW_MALLOC) && !defined(STBIW_FREE) && !defined(STBIW_REALLOC) && !defined(STBIW_REALLOC_SIZED)
// ok
// 否则报错，必须同时定义或不定义 STBIW_MALLOC、STBIW_FREE 和 STBIW_REALLOC（或 STBIW_REALLOC_SIZED）
#else
#error "Must define all or none of STBIW_MALLOC, STBIW_FREE, and STBIW_REALLOC (or STBIW_REALLOC_SIZED)."
#endif

// 如果未定义 STBIW_MALLOC，则定义 STBIW_MALLOC 为 malloc
#ifndef STBIW_MALLOC
#define STBIW_MALLOC(sz)        malloc(sz)
// 如果未定义 STBIW_REALLOC，则定义 STBIW_REALLOC 为 realloc
#define STBIW_REALLOC(p,newsz)  realloc(p,newsz)
// 如果未定义 STBIW_FREE，则定义 STBIW_FREE 为 free
#define STBIW_FREE(p)           free(p)
#endif

// 如果未定义 STBIW_REALLOC_SIZED
#ifndef STBIW_REALLOC_SIZED
// 定义了一个宏，用于重新分配内存大小
#define STBIW_REALLOC_SIZED(p,oldsz,newsz) STBIW_REALLOC(p,newsz)
#endif

// 如果未定义 STBIW_MEMMOVE 宏，则定义为 memmove 函数
#ifndef STBIW_MEMMOVE
#define STBIW_MEMMOVE(a,b,sz) memmove(a,b,sz)
#endif

// 如果未定义 STBIW_ASSERT 宏，则包含 assert.h 头文件，并定义为 assert 函数
#ifndef STBIW_ASSERT
#include <assert.h>
#define STBIW_ASSERT(x) assert(x)
#endif

// 定义了一个宏，用于将参数 x 转换为无符号字符型
#define STBIW_UCHAR(x) (unsigned char) ((x) & 0xff)

// 如果定义了 STB_IMAGE_WRITE_STATIC 宏，则静态定义了三个变量，否则定义为全局变量
#ifdef STB_IMAGE_WRITE_STATIC
static int stbi_write_png_compression_level = 8;
static int stbi_write_tga_with_rle = 1;
static int stbi_write_force_png_filter = -1;
#else
int stbi_write_png_compression_level = 8;
int stbi_write_tga_with_rle = 1;
int stbi_write_force_png_filter = -1;
#endif

// 静态变量，用于标记是否在写入时垂直翻转图像
static int stbi__flip_vertically_on_write = 0;

// 设置是否在写入时垂直翻转图像的函数
STBIWDEF void stbi_flip_vertically_on_write(int flag)
{
   stbi__flip_vertically_on_write = flag;
}

// 定义一个结构体，用于存储写入回调函数的上下文信息
typedef struct
{
   stbi_write_func *func;
   void *context;
   unsigned char buffer[64];
   int buf_used;
} stbi__write_context;

// 初始化基于回调函数的上下文
static void stbi__start_write_callbacks(stbi__write_context *s, stbi_write_func *c, void *context)
{
   s->func    = c;
   s->context = context;
}

// 如果未定义 STBI_WRITE_NO_STDIO 宏，则定义一个使用标准 I/O 的写入函数
static void stbi__stdio_write(void *context, void *data, int size)
{
   fwrite(data,1,size,(FILE*) context);
}

// 如果在 Windows 系统下并且定义了 STBIW_WINDOWS_UTF8 宏，则定义一些函数和宏
#if defined(_WIN32) && defined(STBIW_WINDOWS_UTF8)
#ifdef __cplusplus
#define STBIW_EXTERN extern "C"
#else
#define STBIW_EXTERN extern
#endif
STBIW_EXTERN __declspec(dllimport) int __stdcall MultiByteToWideChar(unsigned int cp, unsigned long flags, const char *str, int cbmb, wchar_t *widestr, int cchwide);
STBIW_EXTERN __declspec(dllimport) int __stdcall WideCharToMultiByte(unsigned int cp, unsigned long flags, const wchar_t *widestr, int cchwide, char *str, int cbmb, const char *defchar, int *used_default);

// 将宽字符转换为 UTF-8 编码的函数
STBIWDEF int stbiw_convert_wchar_to_utf8(char *buffer, size_t bufferlen, const wchar_t* input)
{
   return WideCharToMultiByte(65001 /* UTF8 */, 0, input, -1, buffer, (int) bufferlen, NULL, NULL);
}
#endif

// 使用标准 I/O 打开文件的函数
static FILE *stbiw__fopen(char const *filename, char const *mode)
{
   FILE *f;
#if defined(_WIN32) && defined(STBIW_WINDOWS_UTF8)
   // 如果是在 Windows 系统下，并且定义了 STBIW_WINDOWS_UTF8
   wchar_t wMode[64];
   wchar_t wFilename[1024];
   // 将 UTF-8 编码的文件名转换为宽字符编码
   if (0 == MultiByteToWideChar(65001 /* UTF8 */, 0, filename, -1, wFilename, sizeof(wFilename)/sizeof(*wFilename)))
      return 0;

   // 将 UTF-8 编码的模式转换为宽字符编码
   if (0 == MultiByteToWideChar(65001 /* UTF8 */, 0, mode, -1, wMode, sizeof(wMode)/sizeof(*wMode)))
      return 0;

#if defined(_MSC_VER) && _MSC_VER >= 1400
   // 如果是在 Visual Studio 2005 及以上版本
   // 使用 _wfopen_s 函数打开文件
   if (0 != _wfopen_s(&f, wFilename, wMode))
      f = 0;
#else
   // 使用 _wfopen 函数打开文件
   f = _wfopen(wFilename, wMode);
#endif

#elif defined(_MSC_VER) && _MSC_VER >= 1400
   // 如果是在 Visual Studio 2005 及以上版本
   // 使用 fopen_s 函数打开文件
   if (0 != fopen_s(&f, filename, mode))
      f=0;
#else
   // 使用 fopen 函数打开文件
   f = fopen(filename, mode);
#endif
   // 返回文件指针
   return f;
}

// 开始写文件
static int stbi__start_write_file(stbi__write_context *s, const char *filename)
{
   // 打开文件进行写操作
   FILE *f = stbiw__fopen(filename, "wb");
   // 设置写回调函数和上下文
   stbi__start_write_callbacks(s, stbi__stdio_write, (void *) f);
   // 返回是否成功打开文件
   return f != NULL;
}

// 结束写文件
static void stbi__end_write_file(stbi__write_context *s)
{
   // 关闭文件
   fclose((FILE *)s->context);
}

#endif // !STBI_WRITE_NO_STDIO

// 定义无符号整型 stbiw_uint32
typedef unsigned int stbiw_uint32;
// 检查 stbiw_uint32 是否为 4 字节大小
typedef int stb_image_write_test[sizeof(stbiw_uint32)==4 ? 1 : -1];

// 写入格式化数据
static void stbiw__writefv(stbi__write_context *s, const char *fmt, va_list v)
{
   // 循环遍历格式字符串，直到遇到结束符
   while (*fmt) {
      // 根据格式字符执行不同的操作
      switch (*fmt++) {
         // 如果是空格，则跳过
         case ' ': break;
         // 如果是 '1'，则写入一个无符号字符
         case '1': { unsigned char x = STBIW_UCHAR(va_arg(v, int));
                     // 调用指定的函数写入数据
                     s->func(s->context,&x,1);
                     break; }
         // 如果是 '2'，则写入两个字节
         case '2': { int x = va_arg(v,int);
                     unsigned char b[2];
                     b[0] = STBIW_UCHAR(x);
                     b[1] = STBIW_UCHAR(x>>8);
                     // 调用指定的函数写入数据
                     s->func(s->context,b,2);
                     break; }
         // 如果是 '4'，则写入四个字节
         case '4': { stbiw_uint32 x = va_arg(v,int);
                     unsigned char b[4];
                     b[0]=STBIW_UCHAR(x);
                     b[1]=STBIW_UCHAR(x>>8);
                     b[2]=STBIW_UCHAR(x>>16);
                     b[3]=STBIW_UCHAR(x>>24);
                     // 调用指定的函数写入数据
                     s->func(s->context,b,4);
                     break; }
         // 如果是其他字符，断言失败并返回
         default:
            STBIW_ASSERT(0);
            return;
      }
   }
}

// 写入格式化数据
static void stbiw__writef(stbi__write_context *s, const char *fmt, ...)
{
   va_list v;
   va_start(v, fmt);
   // 调用带可变参数的写入函数
   stbiw__writefv(s, fmt, v);
   va_end(v);
}

// 刷新写入缓冲区
static void stbiw__write_flush(stbi__write_context *s)
{
   // 如果缓冲区中有数据
   if (s->buf_used) {
      // 调用指定的函数写入缓冲区数据
      s->func(s->context, &s->buffer, s->buf_used);
      s->buf_used = 0;
   }
}

// 写入一个字节
static void stbiw__putc(stbi__write_context *s, unsigned char c)
{
   // 调用指定的函数写入一个字节数据
   s->func(s->context, &c, 1);
}

// 写入一个字节数据
static void stbiw__write1(stbi__write_context *s, unsigned char a)
{
   // 如果缓冲区剩余空间不足一个字节
   if ((size_t)s->buf_used + 1 > sizeof(s->buffer))
      // 刷新缓冲区
      stbiw__write_flush(s);
   // 写入一个字节数据到缓冲区
   s->buffer[s->buf_used++] = a;
}

// 写入三个字节数据
static void stbiw__write3(stbi__write_context *s, unsigned char a, unsigned char b, unsigned char c)
{
   int n;
   // 如果缓冲区剩余空间不足三个字节
   if ((size_t)s->buf_used + 3 > sizeof(s->buffer))
      // 刷新缓冲区
      stbiw__write_flush(s);
   n = s->buf_used;
   s->buf_used = n+3;
   // 写入三个字节数据到缓冲区
   s->buffer[n+0] = a;
   s->buffer[n+1] = b;
   s->buffer[n+2] = c;
}

// 写入像素数据
static void stbiw__write_pixel(stbi__write_context *s, int rgb_dir, int comp, int write_alpha, int expand_mono, unsigned char *d)
{
   // 定义背景颜色为紫色，像素颜色数组和循环变量
   unsigned char bg[3] = { 255, 0, 255}, px[3];
   int k;

   // 如果写入 alpha 值小于 0，则写入数据的最后一个通道值
   if (write_alpha < 0)
      stbiw__write1(s, d[comp - 1]);

   // 根据通道数进行不同的处理
   switch (comp) {
      case 2: // 2 个像素 = 单色 + alpha，alpha 值单独写入，与 1 通道情况相同
      case 1:
         // 如果需要扩展为单色图像
         if (expand_mono)
            stbiw__write3(s, d[0], d[0], d[0]); // 单色 BMP
         else
            stbiw__write1(s, d[0]);  // 单色 TGA
         break;
      case 4:
         if (!write_alpha) {
            // 对于没有 alpha 通道的情况，使用粉色背景进行合成
            for (k = 0; k < 3; ++k)
               px[k] = bg[k] + ((d[k] - bg[k]) * d[3]) / 255;
            stbiw__write3(s, px[1 - rgb_dir], px[1], px[1 + rgb_dir]);
            break;
         }
         /* FALLTHROUGH */
      case 3:
         stbiw__write3(s, d[1 - rgb_dir], d[1], d[1 + rgb_dir]);
         break;
   }
   // 如果需要写入 alpha 值，则写入数据的最后一个通道值
   if (write_alpha > 0)
      stbiw__write1(s, d[comp - 1]);
}

// 写入像素数据
static void stbiw__write_pixels(stbi__write_context *s, int rgb_dir, int vdir, int x, int y, int comp, void *data, int write_alpha, int scanline_pad, int expand_mono)
{
   // 定义零值，循环变量
   stbiw_uint32 zero = 0;
   int i,j, j_end;

   // 如果高度小于等于 0，则直接返回
   if (y <= 0)
      return;

   // 如果需要垂直翻转写入
   if (stbi__flip_vertically_on_write)
      vdir *= -1;

   // 根据垂直方向确定循环起始和结束位置
   if (vdir < 0) {
      j_end = -1; j = y-1;
   } else {
      j_end =  y; j = 0;
   }

   // 遍历像素数据并写入
   for (; j != j_end; j += vdir) {
      for (i=0; i < x; ++i) {
         unsigned char *d = (unsigned char *) data + (j*x+i)*comp;
         stbiw__write_pixel(s, rgb_dir, comp, write_alpha, expand_mono, d);
      }
      stbiw__write_flush(s);
      s->func(s->context, &zero, scanline_pad);
   }
}

// 输出文件
static int stbiw__outfile(stbi__write_context *s, int rgb_dir, int vdir, int x, int y, int comp, int expand_mono, void *data, int alpha, int pad, const char *fmt, ...)
{
   // 如果 x 或 y 小于 0，则返回 0
   if (y < 0 || x < 0) {
      return 0;
   } else {
      // 初始化可变参数列表 v
      va_list v;
      // 初始化可变参数列表 v，传入 fmt
      va_start(v, fmt);
      // 使用可变参数列表 v 写入数据到 s
      stbiw__writefv(s, fmt, v);
      // 结束可变参数列表 v
      va_end(v);
      // 写入像素数据到 s
      stbiw__write_pixels(s,rgb_dir,vdir,x,y,comp,data,alpha,pad, expand_mono);
      // 返回 1
      return 1;
   }
}

// 写入 BMP 文件核心函数
static int stbi_write_bmp_core(stbi__write_context *s, int x, int y, int comp, const void *data)
{
   // 如果通道数不为 4
   if (comp != 4) {
      // 写入 RGB 位图
      int pad = (-x*3) & 3;
      // 写入 BMP 文件
      return stbiw__outfile(s,-1,-1,x,y,comp,1,(void *) data,0,pad,
              "11 4 22 4" "4 44 22 444444",
              'B', 'M', 14+40+(x*3+pad)*y, 0,0, 14+40,  // 文件头
               40, x,y, 1,24, 0,0,0,0,0,0);             // 位图头
   } else {
      // RGBA 位图需要 V4 头
      // 使用 32bpp 和 alpha 掩码的 BI_BITFIELDS 模式
      // (大多数阅读器不支持带 alpha 掩码的 BI_RGB 模式)
      // 写入 BMP 文件
      return stbiw__outfile(s,-1,-1,x,y,comp,1,(void *)data,1,0,
         "11 4 22 4" "4 44 22 444444 4444 4 444 444 444 444",
         'B', 'M', 14+108+x*y*4, 0, 0, 14+108, // 文件头
         108, x,y, 1,32, 3,0,0,0,0,0, 0xff0000,0xff00,0xff,0xff000000u, 0, 0,0,0, 0,0,0, 0,0,0, 0,0,0); // 位图 V4 头
   }
}

// 将 BMP 写入函数
STBIWDEF int stbi_write_bmp_to_func(stbi_write_func *func, void *context, int x, int y, int comp, const void *data)
{
   // 初始化写入上下文 s
   stbi__write_context s = { 0 };
   // 开始写入回调
   stbi__start_write_callbacks(&s, func, context);
   // 调用写入 BMP 文件核心函数
   return stbi_write_bmp_core(&s, x, y, comp, data);
}

#ifndef STBI_WRITE_NO_STDIO
// 写入 BMP 文件
STBIWDEF int stbi_write_bmp(char const *filename, int x, int y, int comp, const void *data)
{
   // 初始化写入上下文 s
   stbi__write_context s = { 0 };
   // 如果成功打开文件
   if (stbi__start_write_file(&s,filename)) {
      // 调用写入 BMP 文件核心函数
      int r = stbi_write_bmp_core(&s, x, y, comp, data);
      // 结束写入文件
      stbi__end_write_file(&s);
      return r;
   } else
      return 0;
}
#endif //!STBI_WRITE_NO_STDIO

// 写入 TGA 文件核心函数
static int stbi_write_tga_core(stbi__write_context *s, int x, int y, int comp, void *data)
}
// 将 TGA 图像数据写入指定的输出函数
STBIWDEF int stbi_write_tga_to_func(stbi_write_func *func, void *context, int x, int y, int comp, const void *data)
{
   // 创建写入上下文对象
   stbi__write_context s = { 0 };
   // 初始化写入回调函数
   stbi__start_write_callbacks(&s, func, context);
   // 调用 TGA 写入核心函数
   return stbi_write_tga_core(&s, x, y, comp, (void *) data);
}

#ifndef STBI_WRITE_NO_STDIO
// 将 TGA 图像数据写入指定的文件
STBIWDEF int stbi_write_tga(char const *filename, int x, int y, int comp, const void *data)
{
   // 创建写入上下文对象
   stbi__write_context s = { 0 };
   // 如果成功打开文件进行写入
   if (stbi__start_write_file(&s,filename)) {
      // 调用 TGA 写入核心函数
      int r = stbi_write_tga_core(&s, x, y, comp, (void *) data);
      // 结束文件写入
      stbi__end_write_file(&s);
      return r;
   } else
      return 0;
}
#endif

// *************************************************************************************************
// Radiance RGBE HDR writer
// by Baldur Karlsson

// 定义获取两个数中的最大值的宏
#define stbiw__max(a, b)  ((a) > (b) ? (a) : (b))

#ifndef STBI_WRITE_NO_STDIO

// 将线性颜色转换为 RGBE 格式
static void stbiw__linear_to_rgbe(unsigned char *rgbe, float *linear)
{
   int exponent;
   // 获取颜色分量中的最大值
   float maxcomp = stbiw__max(linear[0], stbiw__max(linear[1], linear[2]));

   // 如果最大值小于阈值，则设置为 0
   if (maxcomp < 1e-32f) {
      rgbe[0] = rgbe[1] = rgbe[2] = rgbe[3] = 0;
   } else {
      // 计算归一化系数和指数
      float normalize = (float) frexp(maxcomp, &exponent) * 256.0f/maxcomp;

      // 将线性颜色转换为 RGBE 格式
      rgbe[0] = (unsigned char)(linear[0] * normalize);
      rgbe[1] = (unsigned char)(linear[1] * normalize);
      rgbe[2] = (unsigned char)(linear[2] * normalize);
      rgbe[3] = (unsigned char)(exponent + 128);
   }
}

// 写入运行长度编码数据
static void stbiw__write_run_data(stbi__write_context *s, int length, unsigned char databyte)
{
   // 计算长度字节
   unsigned char lengthbyte = STBIW_UCHAR(length+128);
   // 断言长度不超过 255
   STBIW_ASSERT(length+128 <= 255);
   // 写入长度字节和数据字节
   s->func(s->context, &lengthbyte, 1);
   s->func(s->context, &databyte, 1);
}

// 写入非运行长度编码数据
static void stbiw__write_dump_data(stbi__write_context *s, int length, unsigned char *data)
{
   // 计算长度字节
   unsigned char lengthbyte = STBIW_UCHAR(length);
   // 断言长度不超过 128
   STBIW_ASSERT(length <= 128); // inconsistent with spec but consistent with official code
   // 写入长度字节和数据
   s->func(s->context, &lengthbyte, 1);
   s->func(s->context, data, length);
}
// 写入 HDR 格式图像扫描线
static void stbiw__write_hdr_scanline(stbi__write_context *s, int width, int ncomp, unsigned char *scratch, float *scanline)
}

// 写入 HDR 核心函数
static int stbi_write_hdr_core(stbi__write_context *s, int x, int y, int comp, float *data)
{
   // 检查参数是否有效
   if (y <= 0 || x <= 0 || data == NULL)
      return 0;
   else {
      // 每个分量单独存储。为完整输出扫描线分配临时空间。
      unsigned char *scratch = (unsigned char *) STBIW_MALLOC(x*4);
      int i, len;
      char buffer[128];
      char header[] = "#?RADIANCE\n# Written by stb_image_write.h\nFORMAT=32-bit_rle_rgbe\n";
      // 调用回调函数写入头部信息
      s->func(s->context, header, sizeof(header)-1);

#ifdef __STDC_LIB_EXT1__
      // 使用 sprintf_s 格式化字符串
      len = sprintf_s(buffer, sizeof(buffer), "EXPOSURE=          1.0000000000000\n\n-Y %d +X %d\n", y, x);
#else
      // 使用 sprintf 格式化字符串
      len = sprintf(buffer, "EXPOSURE=          1.0000000000000\n\n-Y %d +X %d\n", y, x);
#endif
      // 调用回调函数写入格式化后的字符串
      s->func(s->context, buffer, len);

      // 循环写入每一行扫描线
      for(i=0; i < y; i++)
         stbiw__write_hdr_scanline(s, x, comp, scratch, data + comp*x*(stbi__flip_vertically_on_write ? y-1-i : i));
      // 释放临时空间
      STBIW_FREE(scratch);
      return 1;
   }
}

// 将 HDR 数据写入回调函数
STBIWDEF int stbi_write_hdr_to_func(stbi_write_func *func, void *context, int x, int y, int comp, const float *data)
{
   stbi__write_context s = { 0 };
   // 初始化写入回调函数
   stbi__start_write_callbacks(&s, func, context);
   return stbi_write_hdr_core(&s, x, y, comp, (float *) data);
}

// 将 HDR 数据写入文件
STBIWDEF int stbi_write_hdr(char const *filename, int x, int y, int comp, const float *data)
{
   stbi__write_context s = { 0 };
   // 开始写入文件
   if (stbi__start_write_file(&s,filename)) {
      // 写入 HDR 数据到文件
      int r = stbi_write_hdr_core(&s, x, y, comp, (float *) data);
      // 结束写入文件
      stbi__end_write_file(&s);
      return r;
   } else
      return 0;
}
#endif // STBI_WRITE_NO_STDIO


//////////////////////////////////////////////////////////////////////////////
//
// PNG writer
//

#ifndef STBIW_ZLIB_COMPRESS
// stretchy buffer; stbiw__sbpush() == vector<>::push_back() -- stbiw__sbcount() == vector<>::size()
// 定义一个宏，用于获取动态数组的起始地址
#define stbiw__sbraw(a) ((int *) (void *) (a) - 2)
// 定义一个宏，用于获取动态数组的总容量
#define stbiw__sbm(a)   stbiw__sbraw(a)[0]
// 定义一个宏，用于获取动态数组的当前元素个数
#define stbiw__sbn(a)   stbiw__sbraw(a)[1]

// 定义一个宏，用于判断是否需要扩容动态数组
#define stbiw__sbneedgrow(a,n)  ((a)==0 || stbiw__sbn(a)+n >= stbiw__sbm(a))
// 定义一个宏，用于判断是否可能需要扩容动态数组
#define stbiw__sbmaybegrow(a,n) (stbiw__sbneedgrow(a,(n)) ? stbiw__sbgrow(a,n) : 0)
// 定义一个宏，用于扩容动态数组
#define stbiw__sbgrow(a,n)  stbiw__sbgrowf((void **) &(a), (n), sizeof(*(a)))

// 定义一个宏，用于向动态数组中添加元素
#define stbiw__sbpush(a, v)      (stbiw__sbmaybegrow(a,1), (a)[stbiw__sbn(a)++] = (v))
// 定义一个宏，用于获取动态数组中元素的个数
#define stbiw__sbcount(a)        ((a) ? stbiw__sbn(a) : 0)
// 定义一个宏，用于释放动态数组的内存
#define stbiw__sbfree(a)         ((a) ? STBIW_FREE(stbiw__sbraw(a)),0 : 0)

// 定义一个函数，用于扩容动态数组
static void *stbiw__sbgrowf(void **arr, int increment, int itemsize)
{
   // 计算新的容量
   int m = *arr ? 2*stbiw__sbm(*arr)+increment : increment+1;
   // 重新分配内存
   void *p = STBIW_REALLOC_SIZED(*arr ? stbiw__sbraw(*arr) : 0, *arr ? (stbiw__sbm(*arr)*itemsize + sizeof(int)*2) : 0, itemsize * m + sizeof(int)*2);
   STBIW_ASSERT(p);
   if (p) {
      if (!*arr) ((int *) p)[1] = 0;
      *arr = (void *) ((int *) p + 2);
      stbiw__sbm(*arr) = m;
   }
   return *arr;
}

// 定义一个函数，用于将位缓冲区中的数据刷新到字节数组中
static unsigned char *stbiw__zlib_flushf(unsigned char *data, unsigned int *bitbuffer, int *bitcount)
{
   // 将位缓冲区中的数据刷新到字节数组中
   while (*bitcount >= 8) {
      stbiw__sbpush(data, STBIW_UCHAR(*bitbuffer));
      *bitbuffer >>= 8;
      *bitcount -= 8;
   }
   return data;
}

// 定义一个函数，用于反转位码
static int stbiw__zlib_bitrev(int code, int codebits)
{
   int res=0;
   while (codebits--) {
      res = (res << 1) | (code & 1);
      code >>= 1;
   }
   return res;
}

// 定义一个函数，用于计算两个字节数组的匹配长度
static unsigned int stbiw__zlib_countm(unsigned char *a, unsigned char *b, int limit)
{
   int i;
   for (i=0; i < limit && i < 258; ++i)
      if (a[i] != b[i]) break;
   return i;
}

// 定义一个函数，用于计算字节数组的哈希值
static unsigned int stbiw__zhash(unsigned char *data)
{
   stbiw_uint32 hash = data[0] + (data[1] << 8) + (data[2] << 16);
   hash ^= hash << 3;
   hash += hash >> 5;
   hash ^= hash << 4;
   hash += hash >> 17;
   hash ^= hash << 25;
   hash += hash >> 6;
   return hash;
}
// 定义宏函数 stbiw__zlib_flush()，用于刷新输出缓冲区
#define stbiw__zlib_flush() (out = stbiw__zlib_flushf(out, &bitbuf, &bitcount))
// 定义宏函数 stbiw__zlib_add(code,codebits)，用于向输出缓冲区添加数据
#define stbiw__zlib_add(code,codebits) \
      (bitbuf |= (code) << bitcount, bitcount += (codebits), stbiw__zlib_flush())
// 定义宏函数 stbiw__zlib_huffa(b,c)，用于向输出缓冲区添加经过反转的哈夫曼编码
#define stbiw__zlib_huffa(b,c)  stbiw__zlib_add(stbiw__zlib_bitrev(b,c),c)
// 定义默认的哈夫曼表
#define stbiw__zlib_huff1(n)  stbiw__zlib_huffa(0x30 + (n), 8)
#define stbiw__zlib_huff2(n)  stbiw__zlib_huffa(0x190 + (n)-144, 9)
#define stbiw__zlib_huff3(n)  stbiw__zlib_huffa(0 + (n)-256,7)
#define stbiw__zlib_huff4(n)  stbiw__zlib_huffa(0xc0 + (n)-280,8)
#define stbiw__zlib_huff(n)  ((n) <= 143 ? stbiw__zlib_huff1(n) : (n) <= 255 ? stbiw__zlib_huff2(n) : (n) <= 279 ? stbiw__zlib_huff3(n) : stbiw__zlib_huff4(n))
#define stbiw__zlib_huffb(n) ((n) <= 143 ? stbiw__zlib_huff1(n) : stbiw__zlib_huff2(n))
// 定义常量 stbiw__ZHASH 为 16384

// 结束条件，结束 STBIW_ZLIB_COMPRESS 的定义
#endif // STBIW_ZLIB_COMPRESS

// 定义函数 stbi_zlib_compress，用于压缩数据
STBIWDEF unsigned char * stbi_zlib_compress(unsigned char *data, int data_len, int *out_len, int quality)
{
#ifdef STBIW_ZLIB_COMPRESS
   // 如果用户提供了 zlib 压缩实现，则使用用户提供的实现
   return STBIW_ZLIB_COMPRESS(data, data_len, out_len, quality);
#endif // STBIW_ZLIB_COMPRESS
}

// 计算 CRC32 校验值的函数
static unsigned int stbiw__crc32(unsigned char *buffer, int len)
{
#ifdef STBIW_CRC32
    return STBIW_CRC32(buffer, len);
#endif
}

// 定义宏函数 stbiw__wpng4，用于将四个字节写入数据
#define stbiw__wpng4(o,a,b,c,d) ((o)[0]=STBIW_UCHAR(a),(o)[1]=STBIW_UCHAR(b),(o)[2]=STBIW_UCHAR(c),(o)[3]=STBIW_UCHAR(d),(o)+=4)
// 定义宏函数 stbiw__wp32，用于将 32 位整数写入数据
#define stbiw__wp32(data,v) stbiw__wpng4(data, (v)>>24,(v)>>16,(v)>>8,(v));
// 定义宏函数 stbiw__wptag，用于将标签写入数据
#define stbiw__wptag(data,s) stbiw__wpng4(data, s[0],s[1],s[2],s[3])

// 计算 CRC 校验值并写入数据的函数
static void stbiw__wpcrc(unsigned char **data, int len)
{
   unsigned int crc = stbiw__crc32(*data - len - 4, len+4);
   stbiw__wp32(*data, crc);
}

// Paeth 预测算法
static unsigned char stbiw__paeth(int a, int b, int c)
{
   int p = a + b - c, pa = abs(p-a), pb = abs(p-b), pc = abs(p-c);
   if (pa <= pb && pa <= pc) return STBIW_UCHAR(a);
   if (pb <= pc) return STBIW_UCHAR(b);
   return STBIW_UCHAR(c);
}
// 定义一个静态函数，用于对 PNG 图像的一行进行编码
static void stbiw__encode_png_line(unsigned char *pixels, int stride_bytes, int width, int height, int y, int n, int filter_type, signed char *line_buffer)
{
   // 定义两个静态数组，用于映射不同的滤波器类型
   static int mapping[] = { 0,1,2,3,4 };
   static int firstmap[] = { 0,1,0,5,6 };
   // 根据当前行数选择映射数组
   int *mymap = (y != 0) ? mapping : firstmap;
   int i;
   // 根据是否垂直翻转选择起始位置
   int type = mymap[filter_type];
   unsigned char *z = pixels + stride_bytes * (stbi__flip_vertically_on_write ? height-1-y : y);
   int signed_stride = stbi__flip_vertically_on_write ? -stride_bytes : stride_bytes;

   // 如果滤波器类型为0，直接复制像素数据到行缓冲
   if (type==0) {
      memcpy(line_buffer, z, width*n);
      return;
   }

   // 根据滤波器类型对像素数据进行处理
   for (i = 0; i < n; ++i) {
      switch (type) {
         case 1: line_buffer[i] = z[i]; break;
         case 2: line_buffer[i] = z[i] - z[i-signed_stride]; break;
         case 3: line_buffer[i] = z[i] - (z[i-signed_stride]>>1); break;
         case 4: line_buffer[i] = (signed char) (z[i] - stbiw__paeth(0,z[i-signed_stride],0)); break;
         case 5: line_buffer[i] = z[i]; break;
         case 6: line_buffer[i] = z[i]; break;
      }
   }
   // 根据滤波器类型对剩余像素数据进行处理
   switch (type) {
      case 1: for (i=n; i < width*n; ++i) line_buffer[i] = z[i] - z[i-n]; break;
      case 2: for (i=n; i < width*n; ++i) line_buffer[i] = z[i] - z[i-signed_stride]; break;
      case 3: for (i=n; i < width*n; ++i) line_buffer[i] = z[i] - ((z[i-n] + z[i-signed_stride])>>1); break;
      case 4: for (i=n; i < width*n; ++i) line_buffer[i] = z[i] - stbiw__paeth(z[i-n], z[i-signed_stride], z[i-signed_stride-n]); break;
      case 5: for (i=n; i < width*n; ++i) line_buffer[i] = z[i] - (z[i-n]>>1); break;
      case 6: for (i=n; i < width*n; ++i) line_buffer[i] = z[i] - stbiw__paeth(z[i-n], 0,0); break;
   }
}

// 将 PNG 图像数据写入内存
STBIWDEF unsigned char *stbi_write_png_to_mem(const unsigned char *pixels, int stride_bytes, int x, int y, int n, int *out_len, const char* parameters)
{
}

// 如果未定义 STBI_WRITE_NO_STDIO，则包含标准输入输出库
#ifndef STBI_WRITE_NO_STDIO
// 定义函数 stbi_write_png，用于将数据写入 PNG 文件
STBIWDEF int stbi_write_png(char const *filename, int x, int y, int comp, const void *data, int stride_bytes, const char* parameters)
{
   // 打开文件指针
   FILE *f;
   // 定义变量 len
   int len;
   // 调用 stbi_write_png_to_mem 函数将数据写入内存中的 PNG 格式
   unsigned char *png = stbi_write_png_to_mem((const unsigned char *) data, stride_bytes, x, y, comp, &len, parameters);
   // 如果写入失败，则返回 0
   if (png == NULL) return 0;

   // 打开文件，以二进制写入模式
   f = stbiw__fopen(filename, "wb");
   // 如果文件打开失败，则释放内存并返回 0
   if (!f) { STBIW_FREE(png); return 0; }
   // 将 PNG 数据写入文件
   fwrite(png, 1, len, f);
   // 关闭文件
   fclose(f);
   // 释放 PNG 数据内存
   STBIW_FREE(png);
   // 返回 1 表示成功
   return 1;
}
#endif

// 定义函数 stbi_write_png_to_func，用于将数据写入指定函数
STBIWDEF int stbi_write_png_to_func(stbi_write_func *func, void *context, int x, int y, int comp, const void *data, int stride_bytes)
{
   // 定义变量 len
   int len;
   // 调用 stbi_write_png_to_mem 函数将数据写入内存中的 PNG 格式
   unsigned char *png = stbi_write_png_to_mem((const unsigned char *) data, stride_bytes, x, y, comp, &len, NULL);
   // 如果写入失败，则返回 0
   if (png == NULL) return 0;
   // 调用指定函数将 PNG 数据写入指定上下文
   func(context, png, len);
   // 释放 PNG 数据内存
   STBIW_FREE(png);
   // 返回 1 表示成功
   return 1;
}

// JPEG 写入器
// 基于 Jon Olick 的 jo_jpeg.cpp 实现
// 公有领域的简单、极简 JPEG 写入器 - http://www.jonolick.com/code.html
static const unsigned char stbiw__jpg_ZigZag[] = { 0,1,5,6,14,15,27,28,2,4,7,13,16,26,29,42,3,8,12,17,25,30,41,43,9,11,18,
      24,31,40,44,53,10,19,23,32,39,45,52,54,20,22,33,38,46,51,55,60,21,34,37,47,50,56,59,61,35,36,48,49,57,58,62,63 };

// 写入比特数据
static void stbiw__jpg_writeBits(stbi__write_context *s, int *bitBufP, int *bitCntP, const unsigned short *bs) {
   // 获取比特缓冲和比特计数
   int bitBuf = *bitBufP, bitCnt = *bitCntP;
   // 更新比特计数
   bitCnt += bs[1];
   // 将比特数据写入比特缓冲
   bitBuf |= bs[0] << (24 - bitCnt);
   // 当比特计数大于等于 8 时，将比特缓冲中的数据写入输出流
   while(bitCnt >= 8) {
      unsigned char c = (bitBuf >> 16) & 255;
      stbiw__putc(s, c);
      // 如果写入的数据为 255，则写入一个额外的 0
      if(c == 255) {
         stbiw__putc(s, 0);
      }
      // 左移比特缓冲，减少比特计数
      bitBuf <<= 8;
      bitCnt -= 8;
   }
   // 更新比特缓冲和比特计数
   *bitBufP = bitBuf;
   *bitCntP = bitCnt;
}
// 执行 JPEG 的离散余弦变换
static void stbiw__jpg_DCT(float *d0p, float *d1p, float *d2p, float *d3p, float *d4p, float *d5p, float *d6p, float *d7p) {
   // 从指针中获取对应的值
   float d0 = *d0p, d1 = *d1p, d2 = *d2p, d3 = *d3p, d4 = *d4p, d5 = *d5p, d6 = *d6p, d7 = *d7p;
   float z1, z2, z3, z4, z5, z11, z13;

   // 计算 DCT 变换的过程
   float tmp0 = d0 + d7;
   float tmp7 = d0 - d7;
   float tmp1 = d1 + d6;
   float tmp6 = d1 - d6;
   float tmp2 = d2 + d5;
   float tmp5 = d2 - d5;
   float tmp3 = d3 + d4;
   float tmp4 = d3 - d4;

   // 偶数部分的计算
   float tmp10 = tmp0 + tmp3;   // phase 2
   float tmp13 = tmp0 - tmp3;
   float tmp11 = tmp1 + tmp2;
   float tmp12 = tmp1 - tmp2;

   d0 = tmp10 + tmp11;       // phase 3
   d4 = tmp10 - tmp11;

   z1 = (tmp12 + tmp13) * 0.707106781f; // c4
   d2 = tmp13 + z1;       // phase 5
   d6 = tmp13 - z1;

   // 奇数部分的计算
   tmp10 = tmp4 + tmp5;       // phase 2
   tmp11 = tmp5 + tmp6;
   tmp12 = tmp6 + tmp7;

   // 旋转器的修改，避免额外的否定
   z5 = (tmp10 - tmp12) * 0.382683433f; // c6
   z2 = tmp10 * 0.541196100f + z5; // c2-c6
   z4 = tmp12 * 1.306562965f + z5; // c2+c6
   z3 = tmp11 * 0.707106781f; // c4

   z11 = tmp7 + z3;      // phase 5
   z13 = tmp7 - z3;

   *d5p = z13 + z2;         // phase 6
   *d3p = z13 - z2;
   *d1p = z11 + z4;
   *d7p = z11 - z4;

   *d0p = d0;  *d2p = d2;  *d4p = d4;  *d6p = d6;
}

// 计算给定值的比特数
static void stbiw__jpg_calcBits(int val, unsigned short bits[2]) {
   int tmp1 = val < 0 ? -val : val;
   val = val < 0 ? val-1 : val;
   bits[1] = 1;
   while(tmp1 >>= 1) {
      ++bits[1];
   }
   bits[0] = val & ((1<<bits[1])-1);
}

// 将 JPEG 数据写入函数
STBIWDEF int stbi_write_jpg_to_func(stbi_write_func *func, void *context, int x, int y, int comp, const void *data, int quality)
{
   // 初始化写入上下文
   stbi__write_context s = { 0 };
   // 开始写入回调
   stbi__start_write_callbacks(&s, func, context);
   // 调用 JPEG 写入核心函数
   return stbi_write_jpg_core(&s, x, y, comp, (void *) data, quality);
}

// 如果未定义 STBI_WRITE_NO_STDIO
#ifndef STBI_WRITE_NO_STDIO
// 定义函数 stbi_write_jpg，用于将数据写入 JPG 文件
STBIWDEF int stbi_write_jpg(char const *filename, int x, int y, int comp, const void *data, int quality)
{
   // 初始化写入上下文对象 s
   stbi__write_context s = { 0 };
   // 如果成功打开文件准备写入
   if (stbi__start_write_file(&s,filename)) {
      // 调用 stbi_write_jpg_core 函数写入 JPG 文件核心部分
      int r = stbi_write_jpg_core(&s, x, y, comp, data, quality);
      // 结束文件写入
      stbi__end_write_file(&s);
      // 返回写入结果
      return r;
   } else
      // 如果打开文件失败，返回 0
      return 0;
}
#endif

#endif // STB_IMAGE_WRITE_IMPLEMENTATION
/* Revision history
      1.16  (2021-07-11)
             make Deflate code emit uncompressed blocks when it would otherwise expand
             support writing BMPs with alpha channel
      1.15  (2020-07-13) unknown
      1.14  (2020-02-02) updated JPEG writer to downsample chroma channels
      1.13
      1.12
      1.11  (2019-08-11)

      1.10  (2019-02-07)
             support utf8 filenames in Windows; fix warnings and platform ifdefs
      1.09  (2018-02-11)
             fix typo in zlib quality API, improve STB_I_W_STATIC in C++
      1.08  (2018-01-29)
             add stbi__flip_vertically_on_write, external zlib, zlib quality, choose PNG filter
      1.07  (2017-07-24)
             doc fix
      1.06 (2017-07-23)
             writing JPEG (using Jon Olick's code)
      1.05   ???
      1.04 (2017-03-03)
             monochrome BMP expansion
      1.03   ???
      1.02 (2016-04-02)
             avoid allocating large structures on the stack
      1.01 (2016-01-16)
             STBIW_REALLOC_SIZED: support allocators with no realloc support
             avoid race-condition in crc initialization
             minor compile issues
      1.00 (2015-09-14)
             installable file IO function
      0.99 (2015-09-13)
             warning fixes; TGA rle support
      0.98 (2015-04-08)
             added STBIW_MALLOC, STBIW_ASSERT etc
      0.97 (2015-01-18)
             fixed HDR asserts, rewrote HDR rle logic
      0.96 (2015-01-17)
             add HDR output
             fix monochrome BMP
      0.95 (2014-08-17)
             add monochrome TGA output
      0.94 (2014-05-31)
             rename private functions to avoid conflicts with stb_image.h
      0.93 (2014-05-27)
             warning fixes
      0.92 (2010-08-01)
             casts to unsigned char to fix warnings
      0.91 (2010-07-17)
             first public release
      0.90   first internal release
*/

/*
------------------------------------------------------------------------------
# 这个软件提供两种许可证选择
# ------------------------------------------------------------------------------
# 选择 A - MIT 许可证
# 版权所有 (c) 2017 Sean Barrett
# 在遵守以下条件的情况下，免费授予任何获得此软件及相关文档文件（以下简称“软件”）副本的人使用、复制、修改、合并、发布、分发、许可、销售软件的权利，并允许获得软件的人员这样做：
# 上述版权声明和此许可声明应包含在所有副本或实质部分的软件中。
# 本软件按“原样”提供，不提供任何形式的明示或暗示担保，包括但不限于适销性、特定用途适用性和非侵权性的担保。在任何情况下，作者或版权持有人均不对任何索赔、损害或其他责任负责，无论是合同诉讼、侵权行为还是其他情况，由此产生、由此引起或与软件或使用或其他交易有关。
# ------------------------------------------------------------------------------
# 选择 B - 公共领域（www.unlicense.org）
# 这是自由且不受限制的软件，放入公共领域。
# 任何人都可以自由复制、修改、发布、使用、编译、销售或分发此软件，无论是以源代码形式还是编译后的二进制形式，无论是商业用途还是非商业用途，以及通过任何方式。
# 在承认版权法的司法管辖区，本软件的作者或作者将此软件的所有版权利益捐赠给公共领域。我们做出这一奉献是为了使公众受益，对我们的继承人和后继者造成损害。我们打算将这一奉献视为
# 永久放弃对此软件在版权法下的所有现有和未来权利的公开行为。
# 本软件按原样提供，不附带任何形式的保证，明示或暗示，包括但不限于适销性、特定用途适用性和非侵权性的保证。在任何情况下，作者均不对任何索赔、损害或其他责任承担责任，无论是合同行为、侵权行为还是其他行为，起因于、由此引起或与本软件或本软件的使用或其他交易有关。
```