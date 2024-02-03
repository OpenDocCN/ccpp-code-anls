# `ggml\examples\stb_image_write.h`

```cpp
# 定义了 stb_image_write 的版本信息和使用条款
# 包含该文件的实现
# 注意：在包含该文件的文件中，需要定义 #define STB_IMAGE_WRITE_IMPLEMENTATION
# 该库用于将图像写入 C 标准输入输出或回调函数
# PNG 输出不是最优的；它比一个良好优化的实现写入的文件要大 20-50%；
# 尽管提供自定义的 zlib 压缩函数（参见 STBIW_ZLIB_COMPRESS）可以缓解这一问题。
# 该库设计用于源代码紧凑性和简单性，而不是最优图像文件大小或运行时性能。
# 可以在 #include 之前定义 STBIW_ASSERT(x) 来避免使用 assert.h
# 可以定义 STBIW_MALLOC()、STBIW_REALLOC() 和 STBIW_FREE() 来替换 malloc、realloc、free
# 可以定义 STBIW_MEMMOVE() 来替换 memmove()
# 可以定义 STBIW_ZLIB_COMPRESS 来使用自定义的 zlib 风格压缩函数进行 PNG 压缩（而不是内置的压缩函数），
# 它必须具有以下签名：unsigned char * my_compress(unsigned char *data, int data_len, int *out_len, int quality)
# 返回的数据将使用 STBIW_FREE()（默认为 free()）释放，因此它必须使用 STBIW_MALLOC()（默认为 malloc()）进行堆分配
# 如果在 Windows 上编译并希望使用 Unicode 文件名，请使用 #define STBIW_WINDOWS_UTF8，并传递 utf8 编码的文件名。
# 调用 stbiw_convert_wchar_to_utf8 将 Windows 的 wchar_t 文件名转换为 utf8。
/*
   以下是一些贡献者的名单，他们为该代码库做出了贡献
   包括不同格式的图片处理和bug修复
   请参考文件末尾的许可信息
*/

#ifndef INCLUDE_STB_IMAGE_WRITE_H
#define INCLUDE_STB_IMAGE_WRITE_H

#include <stdlib.h>

// 如果 STB_IMAGE_WRITE_STATIC 导致问题，请尝试将 STBIWDEF 定义为 'inline' 或 'static inline'
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

#ifndef STB_IMAGE_WRITE_STATIC  // C++ 禁止静态前向声明
STBIWDEF int stbi_write_tga_with_rle;
STBIWDEF int stbi_write_png_compression_level;
STBIWDEF int stbi_write_force_png_filter;
#endif

#ifndef STBI_WRITE_NO_STDIO
STBIWDEF int stbi_write_png(char const *filename, int w, int h, int comp, const void  *data, int stride_in_bytes);
STBIWDEF int stbi_write_bmp(char const *filename, int w, int h, int comp, const void  *data);
STBIWDEF int stbi_write_tga(char const *filename, int w, int h, int comp, const void  *data);
STBIWDEF int stbi_write_hdr(char const *filename, int w, int h, int comp, const float *data);
# 定义了一个函数，用于将数据写入 JPG 格式的文件
STBIWDEF int stbi_write_jpg(char const *filename, int x, int y, int comp, const void  *data, int quality);

# 如果定义了 STBIW_WINDOWS_UTF8，则定义一个函数，用于将宽字符转换为 UTF-8 编码
#ifdef STBIW_WINDOWS_UTF8
STBIWDEF int stbiw_convert_wchar_to_utf8(char *buffer, size_t bufferlen, const wchar_t* input);
#endif
#endif

# 定义了一个函数指针类型 stbi_write_func，用于写入数据的回调函数
typedef void stbi_write_func(void *context, void *data, int size);

# 定义了一系列函数，用于将数据写入不同格式的文件
STBIWDEF int stbi_write_png_to_func(stbi_write_func *func, void *context, int w, int h, int comp, const void  *data, int stride_in_bytes);
STBIWDEF int stbi_write_bmp_to_func(stbi_write_func *func, void *context, int w, int h, int comp, const void  *data);
STBIWDEF int stbi_write_tga_to_func(stbi_write_func *func, void *context, int w, int h, int comp, const void  *data);
STBIWDEF int stbi_write_hdr_to_func(stbi_write_func *func, void *context, int w, int h, int comp, const float *data);
STBIWDEF int stbi_write_jpg_to_func(stbi_write_func *func, void *context, int x, int y, int comp, const void  *data, int quality);

# 定义了一个函数，用于设置写入时是否垂直翻转图像
STBIWDEF void stbi_flip_vertically_on_write(int flip_boolean);

#endif//INCLUDE_STB_IMAGE_WRITE_H

# 如果定义了 STB_IMAGE_WRITE_IMPLEMENTATION，则进行以下操作
#ifdef STB_IMAGE_WRITE_IMPLEMENTATION

# 如果在 Windows 平台下，则定义一些预处理指令
#ifdef _WIN32
   #ifndef _CRT_SECURE_NO_WARNINGS
   #define _CRT_SECURE_NO_WARNINGS
   #endif
   #ifndef _CRT_NONSTDC_NO_DEPRECATE
   #define _CRT_NONSTDC_NO_DEPRECATE
   #endif
#endif

# 如果没有定义 STBI_WRITE_NO_STDIO，则包含标准输入输出库的头文件
#ifndef STBI_WRITE_NO_STDIO
#include <stdio.h>
#endif // STBI_WRITE_NO_STDIO

# 包含一些标准的头文件
#include <stdarg.h>
#include <stdlib.h>
#include <string.h>
#include <math.h>

# 如果定义了 STBIW_MALLOC 和 STBIW_FREE，并且定义了 STBIW_REALLOC 或 STBIW_REALLOC_SIZED，则通过
# 预处理指令进行检查
#if defined(STBIW_MALLOC) && defined(STBIW_FREE) && (defined(STBIW_REALLOC) || defined(STBIW_REALLOC_SIZED))
// ok
# 如果没有定义 STBIW_MALLOC 和 STBIW_FREE，并且没有定义 STBIW_REALLOC 和 STBIW_REALLOC_SIZED，则通过
# 预处理指令进行检查
#elif !defined(STBIW_MALLOC) && !defined(STBIW_FREE) && !defined(STBIW_REALLOC) && !defined(STBIW_REALLOC_SIZED)
// ok
# 否则，抛出错误
#else
#error "Must define all or none of STBIW_MALLOC, STBIW_FREE, and STBIW_REALLOC (or STBIW_REALLOC_SIZED)."
#endif

# 如果没有定义 STBIW_MALLOC，则定义 STBIW_MALLOC 为 malloc 函数
#ifndef STBIW_MALLOC
#define STBIW_MALLOC(sz)        malloc(sz)
# 如果没有定义 STBIW_REALLOC_SIZED，则定义 STBIW_REALLOC 和 STBIW_FREE 为 realloc 和 free 函数
#define STBIW_REALLOC(p,newsz)  realloc(p,newsz)
#define STBIW_FREE(p)           free(p)
#endif

# 如果没有定义 STBIW_REALLOC_SIZED
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

// 如果定义了 STB_IMAGE_WRITE_STATIC 宏，则静态定义了三个变量，否则定义为外部变量
#ifdef STB_IMAGE_WRITE_STATIC
static int stbi_write_png_compression_level = 8;
static int stbi_write_tga_with_rle = 1;
static int stbi_write_force_png_filter = -1;
#else
int stbi_write_png_compression_level = 8;
int stbi_write_tga_with_rle = 1;
int stbi_write_force_png_filter = -1;
#endif

// 静态定义了一个变量，用于控制是否在写入时垂直翻转图像
static int stbi__flip_vertically_on_write = 0;

// 定义了一个函数，用于设置是否在写入时垂直翻转图像
STBIWDEF void stbi_flip_vertically_on_write(int flag)
{
   stbi__flip_vertically_on_write = flag;
}

// 定义了一个结构体，用于存储写入操作的回调函数和上下文
typedef struct
{
   stbi_write_func *func;
   void *context;
   unsigned char buffer[64];
   int buf_used;
} stbi__write_context;

// 初始化一个基于回调函数的上下文
static void stbi__start_write_callbacks(stbi__write_context *s, stbi_write_func *c, void *context)
{
   s->func    = c;
   s->context = context;
}

// 如果未定义 STBI_WRITE_NO_STDIO 宏，则定义了一些与标准输入输出相关的函数和宏
#ifndef STBI_WRITE_NO_STDIO

// 定义了一个函数，用于将数据写入标准输出流
static void stbi__stdio_write(void *context, void *data, int size)
{
   fwrite(data,1,size,(FILE*) context);
}

// 如果定义了 _WIN32 和 STBIW_WINDOWS_UTF8 宏，则定义了一些与 Windows 平台相关的函数和宏
#if defined(_WIN32) && defined(STBIW_WINDOWS_UTF8)
#ifdef __cplusplus
#define STBIW_EXTERN extern "C"
#else
#define STBIW_EXTERN extern
#endif
STBIW_EXTERN __declspec(dllimport) int __stdcall MultiByteToWideChar(unsigned int cp, unsigned long flags, const char *str, int cbmb, wchar_t *widestr, int cchwide);
STBIW_EXTERN __declspec(dllimport) int __stdcall WideCharToMultiByte(unsigned int cp, unsigned long flags, const wchar_t *widestr, int cchwide, char *str, int cbmb, const char *defchar, int *used_default);

// 定义了一个函数，用于将宽字符转换为 UTF-8 编码
STBIWDEF int stbiw_convert_wchar_to_utf8(char *buffer, size_t bufferlen, const wchar_t* input)
{
   return WideCharToMultiByte(65001 /* UTF8 */, 0, input, -1, buffer, (int) bufferlen, NULL, NULL);
}
#endif

// 定义了一个函数，用于打开文件
static FILE *stbiw__fopen(char const *filename, char const *mode)
{
   FILE *f;
#if defined(_WIN32) && defined(STBIW_WINDOWS_UTF8)
   # 如果是在 Windows 平台且定义了 STBIW_WINDOWS_UTF8
   wchar_t wMode[64];
   # 定义一个宽字符数组 wMode，用于存储文件打开模式
   wchar_t wFilename[1024];
   # 定义一个宽字符数组 wFilename，用于存储文件名
   if (0 == MultiByteToWideChar(65001 /* UTF8 */, 0, filename, -1, wFilename, sizeof(wFilename)/sizeof(*wFilename)))
      return 0;
   # 如果将 UTF-8 编码的文件名转换为宽字符失败，则返回 0

   if (0 == MultiByteToWideChar(65001 /* UTF8 */, 0, mode, -1, wMode, sizeof(wMode)/sizeof(*wMode)))
      return 0;
   # 如果将 UTF-8 编码的模式转换为宽字符失败，则返回 0

#if defined(_MSC_VER) && _MSC_VER >= 1400
   if (0 != _wfopen_s(&f, wFilename, wMode))
      f = 0;
   # 如果使用安全的方式打开宽字符文件失败，则将 f 置为 0
#else
   f = _wfopen(wFilename, wMode);
   # 使用宽字符文件名和模式打开文件
#endif

#elif defined(_MSC_VER) && _MSC_VER >= 1400
   if (0 != fopen_s(&f, filename, mode))
      f=0;
   # 如果使用安全的方式打开文件失败，则将 f 置为 0
#else
   f = fopen(filename, mode);
   # 使用文件名和模式打开文件
#endif
   return f;
   # 返回文件指针 f
}

static int stbi__start_write_file(stbi__write_context *s, const char *filename)
{
   FILE *f = stbiw__fopen(filename, "wb");
   # 以二进制写入模式打开文件，并将文件指针赋值给 f
   stbi__start_write_callbacks(s, stbi__stdio_write, (void *) f);
   # 调用 stbi__start_write_callbacks 函数，传入写入回调函数和文件指针
   return f != NULL;
   # 返回文件指针是否不为空的结果
}

static void stbi__end_write_file(stbi__write_context *s)
{
   fclose((FILE *)s->context);
   # 关闭文件
}

#endif // !STBI_WRITE_NO_STDIO

typedef unsigned int stbiw_uint32;
# 定义一个无符号整型 stbiw_uint32
typedef int stb_image_write_test[sizeof(stbiw_uint32)==4 ? 1 : -1];
# 定义一个数组 stb_image_write_test，用于测试 stbiw_uint32 的大小是否为 4 字节

static void stbiw__writefv(stbi__write_context *s, const char *fmt, va_list v)
# 定义一个函数 stbiw__writefv，接受写入上下文、格式字符串和可变参数列表作为参数
{
   // 当格式字符串未结束时循环执行
   while (*fmt) {
      // 根据格式字符串的内容执行不同的操作
      switch (*fmt++) {
         // 如果是空格，则跳过
         case ' ': break;
         // 如果是1，则将下一个参数转换为无符号字符，并调用函数写入数据
         case '1': { unsigned char x = STBIW_UCHAR(va_arg(v, int));
                     s->func(s->context,&x,1);
                     break; }
         // 如果是2，则将下一个参数转换为整数，并将其拆分为两个无符号字符写入数据
         case '2': { int x = va_arg(v,int);
                     unsigned char b[2];
                     b[0] = STBIW_UCHAR(x);
                     b[1] = STBIW_UCHAR(x>>8);
                     s->func(s->context,b,2);
                     break; }
         // 如果是4，则将下一个参数转换为32位整数，并将其拆分为四个无符号字符写入数据
         case '4': { stbiw_uint32 x = va_arg(v,int);
                     unsigned char b[4];
                     b[0]=STBIW_UCHAR(x);
                     b[1]=STBIW_UCHAR(x>>8);
                     b[2]=STBIW_UCHAR(x>>16);
                     b[3]=STBIW_UCHAR(x>>24);
                     s->func(s->context,b,4);
                     break; }
         // 默认情况下断言失败并返回
         default:
            STBIW_ASSERT(0);
            return;
      }
   }
}

// 格式化写入函数，使用可变参数列表
static void stbiw__writef(stbi__write_context *s, const char *fmt, ...)
{
   va_list v;
   va_start(v, fmt);
   stbiw__writefv(s, fmt, v);
   va_end(v);
}

// 写入缓冲区中的数据到输出流中
static void stbiw__write_flush(stbi__write_context *s)
{
   if (s->buf_used) {
      s->func(s->context, &s->buffer, s->buf_used);
      s->buf_used = 0;
   }
}

// 写入一个无符号字符到输出流中
static void stbiw__putc(stbi__write_context *s, unsigned char c)
{
   s->func(s->context, &c, 1);
}

// 写入一个无符号字符到输出流中，如果缓冲区已满则先执行写入缓冲区的操作
static void stbiw__write1(stbi__write_context *s, unsigned char a)
{
   if ((size_t)s->buf_used + 1 > sizeof(s->buffer))
      stbiw__write_flush(s);
   s->buffer[s->buf_used++] = a;
}

// 写入三个无符号字符到输出流中，如果缓冲区已满则先执行写入缓冲区的操作
static void stbiw__write3(stbi__write_context *s, unsigned char a, unsigned char b, unsigned char c)
{
   int n;
   if ((size_t)s->buf_used + 3 > sizeof(s->buffer))
      stbiw__write_flush(s);
   n = s->buf_used;
   s->buf_used = n+3;
   s->buffer[n+0] = a;
   s->buffer[n+1] = b;
   s->buffer[n+2] = c;
}

// 写入像素数据到输出流中
static void stbiw__write_pixel(stbi__write_context *s, int rgb_dir, int comp, int write_alpha, int expand_mono, unsigned char *d)
{
   // 定义背景颜色为紫色
   unsigned char bg[3] = { 255, 0, 255}, px[3];
   int k;

   // 如果写入 alpha 值小于 0，则调用 stbiw__write1 函数写入数据的最后一个字节
   if (write_alpha < 0)
      stbiw__write1(s, d[comp - 1]);

   // 根据通道数进行不同的处理
   switch (comp) {
      case 2: // 2 pixels = mono + alpha, alpha is written separately, so same as 1-channel case
      case 1:
         // 如果需要扩展为单色图像，则调用 stbiw__write3 函数写入数据
         if (expand_mono)
            stbiw__write3(s, d[0], d[0], d[0]); // monochrome bmp
         else
            stbiw__write1(s, d[0]);  // monochrome TGA
         break;
      case 4:
         if (!write_alpha) {
            // 对于没有 alpha 通道的情况，使用背景颜色进行合成
            for (k = 0; k < 3; ++k)
               px[k] = bg[k] + ((d[k] - bg[k]) * d[3]) / 255;
            stbiw__write3(s, px[1 - rgb_dir], px[1], px[1 + rgb_dir]);
            break;
         }
         /* FALLTHROUGH */
      case 3:
         // 写入 RGB 数据
         stbiw__write3(s, d[1 - rgb_dir], d[1], d[1 + rgb_dir]);
         break;
   }
   // 如果需要写入 alpha 值，则调用 stbiw__write1 函数写入数据的最后一个字节
   if (write_alpha > 0)
      stbiw__write1(s, d[comp - 1]);
}

// 写入像素数据
static void stbiw__write_pixels(stbi__write_context *s, int rgb_dir, int vdir, int x, int y, int comp, void *data, int write_alpha, int scanline_pad, int expand_mono)
{
   stbiw_uint32 zero = 0;
   int i,j, j_end;

   // 如果高度小于等于 0，则直接返回
   if (y <= 0)
      return;

   // 如果需要垂直翻转，则修改垂直方向的步长
   if (stbi__flip_vertically_on_write)
      vdir *= -1;

   // 根据垂直方向的步长确定循环的起止条件
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

// 写入输出文件
static int stbiw__outfile(stbi__write_context *s, int rgb_dir, int vdir, int x, int y, int comp, int expand_mono, void *data, int alpha, int pad, const char *fmt, ...)
{
   // 如果 y 或者 x 小于 0，则返回 0
   if (y < 0 || x < 0) {
      return 0;
   } else {
      // 创建一个 va_list 对象 v，并初始化为 fmt 之后的可变参数
      va_list v;
      va_start(v, fmt);
      // 调用 stbiw__writefv 函数，将格式化的数据写入到输出流中
      stbiw__writefv(s, fmt, v);
      // 结束可变参数的使用
      va_end(v);
      // 调用 stbiw__write_pixels 函数，将像素数据写入到输出流中
      stbiw__write_pixels(s,rgb_dir,vdir,x,y,comp,data,alpha,pad, expand_mono);
      // 返回 1
      return 1;
   }
}

// 写入 BMP 文件的核心函数
static int stbi_write_bmp_core(stbi__write_context *s, int x, int y, int comp, const void *data)
{
   // 如果通道数不等于 4
   if (comp != 4) {
      // 写入 RGB 位图
      int pad = (-x*3) & 3;
      // 调用 stbiw__outfile 函数，写入 BMP 文件
      return stbiw__outfile(s,-1,-1,x,y,comp,1,(void *) data,0,pad,
              "11 4 22 4" "4 44 22 444444",
              'B', 'M', 14+40+(x*3+pad)*y, 0,0, 14+40,  // 文件头
               40, x,y, 1,24, 0,0,0,0,0,0);             // 位图头
   } else {
      // RGBA 位图需要 V4 头
      // 使用 32bpp 和 alpha 掩码的 BI_BITFIELDS 模式
      // (大多数阅读器中直接使用 BI_RGB 和 alpha 掩码无效)
      // 调用 stbiw__outfile 函数，写入 BMP 文件
      return stbiw__outfile(s,-1,-1,x,y,comp,1,(void *)data,1,0,
         "11 4 22 4" "4 44 22 444444 4444 4 444 444 444 444",
         'B', 'M', 14+108+x*y*4, 0, 0, 14+108, // 文件头
         108, x,y, 1,32, 3,0,0,0,0,0, 0xff0000,0xff00,0xff,0xff000000u, 0, 0,0,0, 0,0,0, 0,0,0, 0,0,0); // 位图 V4 头
   }
}

// 将 BMP 写入到函数
STBIWDEF int stbi_write_bmp_to_func(stbi_write_func *func, void *context, int x, int y, int comp, const void *data)
{
   // 创建一个写入上下文对象 s，并初始化为 0
   stbi__write_context s = { 0 };
   // 开始写入回调
   stbi__start_write_callbacks(&s, func, context);
   // 调用 stbi_write_bmp_core 函数，将 BMP 写入到函数
   return stbi_write_bmp_core(&s, x, y, comp, data);
}

// 如果没有定义 STBI_WRITE_NO_STDIO
#ifndef STBI_WRITE_NO_STDIO
// 将 BMP 写入到文件
STBIWDEF int stbi_write_bmp(char const *filename, int x, int y, int comp, const void *data)
{
   // 创建一个写入上下文对象 s，并初始化为 0
   stbi__write_context s = { 0 };
   // 如果成功打开文件
   if (stbi__start_write_file(&s,filename)) {
      // 调用 stbi_write_bmp_core 函数，将 BMP 写入到文件
      int r = stbi_write_bmp_core(&s, x, y, comp, data);
      // 结束写入文件
      stbi__end_write_file(&s);
      return r;
   } else
      return 0;
}
#endif //!STBI_WRITE_NO_STDIO

// 写入 TGA 文件的核心函数
static int stbi_write_tga_core(stbi__write_context *s, int x, int y, int comp, void *data)
}
// 将 TGA 数据写入指定的输出函数
STBIWDEF int stbi_write_tga_to_func(stbi_write_func *func, void *context, int x, int y, int comp, const void *data)
{
   // 创建写入上下文对象
   stbi__write_context s = { 0 };
   // 初始化写入回调函数
   stbi__start_write_callbacks(&s, func, context);
   // 调用核心的 TGA 写入函数
   return stbi_write_tga_core(&s, x, y, comp, (void *) data);
}

#ifndef STBI_WRITE_NO_STDIO
// 将 TGA 数据写入文件
STBIWDEF int stbi_write_tga(char const *filename, int x, int y, int comp, const void *data)
{
   // 创建写入上下文对象
   stbi__write_context s = { 0 };
   // 如果成功打开文件进行写入
   if (stbi__start_write_file(&s,filename)) {
      // 调用核心的 TGA 写入函数
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

// 定义取两个数中的较大值的宏
#define stbiw__max(a, b)  ((a) > (b) ? (a) : (b))

#ifndef STBI_WRITE_NO_STDIO

// 将线性颜色转换为 RGBE 格式
static void stbiw__linear_to_rgbe(unsigned char *rgbe, float *linear)
{
   int exponent;
   // 计算颜色分量的最大值
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
   STBIW_ASSERT(length <= 128); // 与规范不一致，但与官方代码一致
   // 写入长度字节和数据
   s->func(s->context, &lengthbyte, 1);
   s->func(s->context, data, length);
}
static void stbiw__write_hdr_scanline(stbi__write_context *s, int width, int ncomp, unsigned char *scratch, float *scanline)
{
    // 写入 HDR 格式的扫描线数据
}

static int stbi_write_hdr_core(stbi__write_context *s, int x, int y, int comp, float *data)
{
   if (y <= 0 || x <= 0 || data == NULL)
      return 0;
   else {
      // 每个分量都单独存储。为完整的输出扫描线分配临时空间。
      unsigned char *scratch = (unsigned char *) STBIW_MALLOC(x*4);
      int i, len;
      char buffer[128];
      char header[] = "#?RADIANCE\n# Written by stb_image_write.h\nFORMAT=32-bit_rle_rgbe\n";
      s->func(s->context, header, sizeof(header)-1);

#ifdef __STDC_LIB_EXT1__
      len = sprintf_s(buffer, sizeof(buffer), "EXPOSURE=          1.0000000000000\n\n-Y %d +X %d\n", y, x);
#else
      len = sprintf(buffer, "EXPOSURE=          1.0000000000000\n\n-Y %d +X %d\n", y, x);
#endif
      s->func(s->context, buffer, len);

      for(i=0; i < y; i++)
         stbiw__write_hdr_scanline(s, x, comp, scratch, data + comp*x*(stbi__flip_vertically_on_write ? y-1-i : i));
      STBIW_FREE(scratch);
      return 1;
   }
}

STBIWDEF int stbi_write_hdr_to_func(stbi_write_func *func, void *context, int x, int y, int comp, const float *data)
{
   stbi__write_context s = { 0 };
   stbi__start_write_callbacks(&s, func, context);
   return stbi_write_hdr_core(&s, x, y, comp, (float *) data);
}

STBIWDEF int stbi_write_hdr(char const *filename, int x, int y, int comp, const float *data)
{
   stbi__write_context s = { 0 };
   if (stbi__start_write_file(&s,filename)) {
      int r = stbi_write_hdr_core(&s, x, y, comp, (float *) data);
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
# 定义宏，返回动态数组的指针减去2的地址，即数组的元数据
#define stbiw__sbraw(a) ((int *) (void *) (a) - 2)
# 定义宏，返回动态数组的元数据中的数组长度
#define stbiw__sbm(a)   stbiw__sbraw(a)[0]
# 定义宏，返回动态数组的元数据中的数组当前元素个数
#define stbiw__sbn(a)   stbiw__sbraw(a)[1]

# 定义宏，判断动态数组是否需要扩容
#define stbiw__sbneedgrow(a,n)  ((a)==0 || stbiw__sbn(a)+n >= stbiw__sbm(a))
# 定义宏，判断动态数组是否可能需要扩容
#define stbiw__sbmaybegrow(a,n) (stbiw__sbneedgrow(a,(n)) ? stbiw__sbgrow(a,n) : 0)
# 定义宏，对动态数组进行扩容
#define stbiw__sbgrow(a,n)  stbiw__sbgrowf((void **) &(a), (n), sizeof(*(a)))

# 定义宏，向动态数组中添加元素
#define stbiw__sbpush(a, v)      (stbiw__sbmaybegrow(a,1), (a)[stbiw__sbn(a)++] = (v))
# 定义宏，返回动态数组中的元素个数
#define stbiw__sbcount(a)        ((a) ? stbiw__sbn(a) : 0)
# 定义宏，释放动态数组的内存
#define stbiw__sbfree(a)         ((a) ? STBIW_FREE(stbiw__sbraw(a)),0 : 0)

# 定义函数，对数组进行扩容
static void *stbiw__sbgrowf(void **arr, int increment, int itemsize)
{
   # 计算新的数组长度
   int m = *arr ? 2*stbiw__sbm(*arr)+increment : increment+1;
   # 重新分配内存
   void *p = STBIW_REALLOC_SIZED(*arr ? stbiw__sbraw(*arr) : 0, *arr ? (stbiw__sbm(*arr)*itemsize + sizeof(int)*2) : 0, itemsize * m + sizeof(int)*2);
   # 断言内存分配成功
   STBIW_ASSERT(p);
   if (p) {
      if (!*arr) ((int *) p)[1] = 0;
      *arr = (void *) ((int *) p + 2);
      stbiw__sbm(*arr) = m;
   }
   return *arr;
}

# 定义函数，将压缩数据写入输出缓冲区
static unsigned char *stbiw__zlib_flushf(unsigned char *data, unsigned int *bitbuffer, int *bitcount)
{
   while (*bitcount >= 8) {
      stbiw__sbpush(data, STBIW_UCHAR(*bitbuffer));
      *bitbuffer >>= 8;
      *bitcount -= 8;
   }
   return data;
}

# 定义函数，反转二进制位
static int stbiw__zlib_bitrev(int code, int codebits)
{
   int res=0;
   while (codebits--) {
      res = (res << 1) | (code & 1);
      code >>= 1;
   }
   return res;
}

# 定义函数，计算匹配长度
static unsigned int stbiw__zlib_countm(unsigned char *a, unsigned char *b, int limit)
{
   int i;
   for (i=0; i < limit && i < 258; ++i)
      if (a[i] != b[i]) break;
   return i;
}

# 定义函数，计算哈希值
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
#define stbiw__zlib_flush() (out = stbiw__zlib_flushf(out, &bitbuf, &bitcount))
#define stbiw__zlib_add(code,codebits) \
      (bitbuf |= (code) << bitcount, bitcount += (codebits), stbiw__zlib_flush())
#define stbiw__zlib_huffa(b,c)  stbiw__zlib_add(stbiw__zlib_bitrev(b,c),c)
// 定义默认的哈夫曼表
#define stbiw__zlib_huff1(n)  stbiw__zlib_huffa(0x30 + (n), 8)
#define stbiw__zlib_huff2(n)  stbiw__zlib_huffa(0x190 + (n)-144, 9)
#define stbiw__zlib_huff3(n)  stbiw__zlib_huffa(0 + (n)-256,7)
#define stbiw__zlib_huff4(n)  stbiw__zlib_huffa(0xc0 + (n)-280,8)
#define stbiw__zlib_huff(n)  ((n) <= 143 ? stbiw__zlib_huff1(n) : (n) <= 255 ? stbiw__zlib_huff2(n) : (n) <= 279 ? stbiw__zlib_huff3(n) : stbiw__zlib_huff4(n))
#define stbiw__zlib_huffb(n) ((n) <= 143 ? stbiw__zlib_huff1(n) : stbiw__zlib_huff2(n)

#define stbiw__ZHASH   16384

#endif // STBIW_ZLIB_COMPRESS

STBIWDEF unsigned char * stbi_zlib_compress(unsigned char *data, int data_len, int *out_len, int quality)
{
#ifdef STBIW_ZLIB_COMPRESS
   // 如果用户提供了 zlib 压缩实现，则使用该实现
   return STBIW_ZLIB_COMPRESS(data, data_len, out_len, quality);
#endif // STBIW_ZLIB_COMPRESS
}

static unsigned int stbiw__crc32(unsigned char *buffer, int len)
{
#ifdef STBIW_CRC32
    return STBIW_CRC32(buffer, len);
#endif
}

#define stbiw__wpng4(o,a,b,c,d) ((o)[0]=STBIW_UCHAR(a),(o)[1]=STBIW_UCHAR(b),(o)[2]=STBIW_UCHAR(c),(o)[3]=STBIW_UCHAR(d),(o)+=4)
#define stbiw__wp32(data,v) stbiw__wpng4(data, (v)>>24,(v)>>16,(v)>>8,(v));
#define stbiw__wptag(data,s) stbiw__wpng4(data, s[0],s[1],s[2],s[3])

static void stbiw__wpcrc(unsigned char **data, int len)
{
   unsigned int crc = stbiw__crc32(*data - len - 4, len+4);
   stbiw__wp32(*data, crc);
}

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
   // 定义静态数组 mapping 和 firstmap
   static int mapping[] = { 0,1,2,3,4 };
   static int firstmap[] = { 0,1,0,5,6 };
   // 根据当前行数 y 的值选择使用 mapping 还是 firstmap
   int *mymap = (y != 0) ? mapping : firstmap;
   int i;
   // 根据 filter_type 选择不同的类型
   int type = mymap[filter_type];
   // 计算当前行的像素数据的起始位置
   unsigned char *z = pixels + stride_bytes * (stbi__flip_vertically_on_write ? height-1-y : y);
   // 根据是否需要垂直翻转来确定步长的正负
   int signed_stride = stbi__flip_vertically_on_write ? -stride_bytes : stride_bytes;

   // 如果类型为 0，直接拷贝像素数据到行缓冲区
   if (type==0) {
      memcpy(line_buffer, z, width*n);
      return;
   }

   // 根据不同的类型进行像素数据的处理
   // 第一个像素的处理方式不同，需要单独处理
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
   // 根据不同的类型对剩余像素数据进行处理
   switch (type) {
      case 1: for (i=n; i < width*n; ++i) line_buffer[i] = z[i] - z[i-n]; break;
      case 2: for (i=n; i < width*n; ++i) line_buffer[i] = z[i] - z[i-signed_stride]; break;
      case 3: for (i=n; i < width*n; ++i) line_buffer[i] = z[i] - ((z[i-n] + z[i-signed_stride])>>1); break;
      case 4: for (i=n; i < width*n; ++i) line_buffer[i] = z[i] - stbiw__paeth(z[i-n], z[i-signed_stride], z[i-signed_stride-n]); break;
      case 5: for (i=n; i < width*n; ++i) line_buffer[i] = z[i] - (z[i-n]>>1); break;
      case 6: for (i=n; i < width*n; ++i) line_buffer[i] = z[i] - stbiw__paeth(z[i-n], 0,0); break;
   }
}

// 定义一个函数，将 PNG 图像数据写入内存
STBIWDEF unsigned char *stbi_write_png_to_mem(const unsigned char *pixels, int stride_bytes, int x, int y, int n, int *out_len)
{
    // ...（省略部分代码）
}
# 定义函数 stbi_write_png，用于将数据写入 PNG 文件
STBIWDEF int stbi_write_png(char const *filename, int x, int y, int comp, const void *data, int stride_bytes)
{
   FILE *f;  # 声明文件指针 f
   int len;  # 声明整型变量 len
   unsigned char *png = stbi_write_png_to_mem((const unsigned char *) data, stride_bytes, x, y, comp, &len);  # 调用 stbi_write_png_to_mem 函数，将数据写入内存中的 PNG 格式
   if (png == NULL) return 0;  # 如果 png 为空，则返回 0

   f = stbiw__fopen(filename, "wb");  # 打开文件名为 filename 的文件，以写入二进制的方式
   if (!f) { STBIW_FREE(png); return 0; }  # 如果文件打开失败，则释放内存中的 png 数据，并返回 0
   fwrite(png, 1, len, f);  # 将内存中的 png 数据写入文件
   fclose(f);  # 关闭文件
   STBIW_FREE(png);  # 释放内存中的 png 数据
   return 1;  # 返回 1
}
#endif

# 定义函数 stbi_write_png_to_func，用于将数据通过指定的函数写入 PNG 文件
STBIWDEF int stbi_write_png_to_func(stbi_write_func *func, void *context, int x, int y, int comp, const void *data, int stride_bytes)
{
   int len;  # 声明整型变量 len
   unsigned char *png = stbi_write_png_to_mem((const unsigned char *) data, stride_bytes, x, y, comp, &len);  # 调用 stbi_write_png_to_mem 函数，将数据写入内存中的 PNG 格式
   if (png == NULL) return 0;  # 如果 png 为空，则返回 0
   func(context, png, len);  # 调用指定的函数，将内存中的 png 数据写入指定的上下文中
   STBIW_FREE(png);  # 释放内存中的 png 数据
   return 1;  # 返回 1
}

# JPEG writer
# 基于 Jon Olick 的 jo_jpeg.cpp 实现的 JPEG 写入器
# 公有领域的简单、极简的 JPEG 写入器 - http://www.jonolick.com/code.html
static const unsigned char stbiw__jpg_ZigZag[] = { 0,1,5,6,14,15,27,28,2,4,7,13,16,26,29,42,3,8,12,17,25,30,41,43,9,11,18,
      24,31,40,44,53,10,19,23,32,39,45,52,54,20,22,33,38,46,51,55,60,21,34,37,47,50,56,59,61,35,36,48,49,57,58,62,63 };  # 定义 JPEG 中的 ZigZag 数组

# 定义函数 stbiw__jpg_writeBits，用于写入比特数据
static void stbiw__jpg_writeBits(stbi__write_context *s, int *bitBufP, int *bitCntP, const unsigned short *bs) {
   int bitBuf = *bitBufP, bitCnt = *bitCntP;  # 声明整型变量 bitBuf 和 bitCnt，并赋值为传入参数的值
   bitCnt += bs[1];  # 将 bs 数组的第二个元素加到 bitCnt 上
   bitBuf |= bs[0] << (24 - bitCnt);  # 将 bs 数组的第一个元素左移 (24 - bitCnt) 位后的结果与 bitBuf 进行或运算
   while(bitCnt >= 8) {  # 当 bitCnt 大于等于 8 时执行循环
      unsigned char c = (bitBuf >> 16) & 255;  # 将 bitBuf 右移 16 位后的结果与 255 进行与运算，并赋值给无符号字符 c
      stbiw__putc(s, c);  # 调用 stbiw__putc 函数，将字符 c 写入到指定的上下文中
      if(c == 255) {  # 如果 c 等于 255
         stbiw__putc(s, 0);  # 调用 stbiw__putc 函数，将字符 0 写入到指定的上下文中
      }
      bitBuf <<= 8;  # 将 bitBuf 左移 8 位
      bitCnt -= 8;  # 将 bitCnt 减去 8
   }
   *bitBufP = bitBuf;  # 将 bitBuf 的值赋回传入参数的地址
   *bitCntP = bitCnt;  # 将 bitCnt 的值赋回传入参数的地址
}
# 计算 JPEG 的离散余弦变换
static void stbiw__jpg_DCT(float *d0p, float *d1p, float *d2p, float *d3p, float *d4p, float *d5p, float *d6p, float *d7p) {
   float d0 = *d0p, d1 = *d1p, d2 = *d2p, d3 = *d3p, d4 = *d4p, d5 = *d5p, d6 = *d6p, d7 = *d7p;
   float z1, z2, z3, z4, z5, z11, z13;

   float tmp0 = d0 + d7;
   float tmp7 = d0 - d7;
   float tmp1 = d1 + d6;
   float tmp6 = d1 - d6;
   float tmp2 = d2 + d5;
   float tmp5 = d2 - d5;
   float tmp3 = d3 + d4;
   float tmp4 = d3 - d4;

   # Even part
   float tmp10 = tmp0 + tmp3;   # phase 2
   float tmp13 = tmp0 - tmp3;
   float tmp11 = tmp1 + tmp2;
   float tmp12 = tmp1 - tmp2;

   d0 = tmp10 + tmp11;       # phase 3
   d4 = tmp10 - tmp11;

   z1 = (tmp12 + tmp13) * 0.707106781f; # c4
   d2 = tmp13 + z1;       # phase 5
   d6 = tmp13 - z1;

   # Odd part
   tmp10 = tmp4 + tmp5;       # phase 2
   tmp11 = tmp5 + tmp6;
   tmp12 = tmp6 + tmp7;

   # The rotator is modified from fig 4-8 to avoid extra negations.
   z5 = (tmp10 - tmp12) * 0.382683433f; # c6
   z2 = tmp10 * 0.541196100f + z5; # c2-c6
   z4 = tmp12 * 1.306562965f + z5; # c2+c6
   z3 = tmp11 * 0.707106781f; # c4

   z11 = tmp7 + z3;      # phase 5
   z13 = tmp7 - z3;

   *d5p = z13 + z2;         # phase 6
   *d3p = z13 - z2;
   *d1p = z11 + z4;
   *d7p = z11 - z4;

   *d0p = d0;  *d2p = d2;  *d4p = d4;  *d6p = d6;
}

# 计算值的比特位数
static void stbiw__jpg_calcBits(int val, unsigned short bits[2]) {
   int tmp1 = val < 0 ? -val : val;
   val = val < 0 ? val-1 : val;
   bits[1] = 1;
   while(tmp1 >>= 1) {
      ++bits[1];
   }
   bits[0] = val & ((1<<bits[1])-1);
}

# 将 JPEG 写入函数
STBIWDEF int stbi_write_jpg_to_func(stbi_write_func *func, void *context, int x, int y, int comp, const void *data, int quality)
{
   stbi__write_context s = { 0 };
   stbi__start_write_callbacks(&s, func, context);
   return stbi_write_jpg_core(&s, x, y, comp, (void *) data, quality);
}

# 如果未定义 STBI_WRITE_NO_STDIO
#ifndef STBI_WRITE_NO_STDIO
// 定义函数 stbi_write_jpg，用于将数据写入 JPG 文件
STBIWDEF int stbi_write_jpg(char const *filename, int x, int y, int comp, const void *data, int quality)
{
   // 创建写入上下文对象，并初始化为 0
   stbi__write_context s = { 0 };
   // 如果成功打开文件准备写入
   if (stbi__start_write_file(&s,filename)) {
      // 调用 stbi_write_jpg_core 函数，将数据写入 JPG 文件
      int r = stbi_write_jpg_core(&s, x, y, comp, data, quality);
      // 结束文件写入操作
      stbi__end_write_file(&s);
      // 返回写入结果
      return r;
   } else
      // 如果打开文件失败，返回 0
      return 0;
}
#endif

#endif // STB_IMAGE_WRITE_IMPLEMENTATION
# 修订历史记录，列出每个版本的更新内容和日期
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
# 该软件提供两种许可证选择
# ALTERNATIVE A - MIT 许可证
# 版权归 Sean Barrett 所有
# 允许任何人免费获取该软件及相关文档文件的副本，无限制地使用、复制、修改、合并、发布、分发、许可或出售该软件的副本，并允许被授予该软件的人员这样做，但需要包含上述版权声明和许可声明
# 该软件按原样提供，不提供任何形式的保证，包括但不限于适销性、特定用途适用性和非侵权性的保证。在任何情况下，作者或版权持有人均不对任何索赔、损害或其他责任负责，无论是合同诉讼、侵权行为还是其他行为引起的，与软件或使用或其他方式相关的索赔、损害或其他责任
# ALTERNATIVE B - Public Domain (www.unlicense.org)
# 这是一款自由的、不受限制的软件，放入公共领域
# 任何人都可以自由复制、修改、发布、使用、编译、出售或分发该软件，无论是以源代码形式还是编译后的二进制形式，无论是商业用途还是非商业用途，以及通过任何方式
# 在承认版权法的司法管辖区，该软件的作者或作者将该软件的所有版权利益捐赠给公共领域。我们做出这一捐赠是为了使公众受益，对我们的继承人和后继者造成损害。我们打算将这一捐赠视为
/*
永久放弃对此软件在版权法下的所有现有和未来权利的公开行为。
本软件按原样提供，不提供任何形式的保证，明示或暗示，包括但不限于对适销性、特定用途的适用性和非侵权的保证。在任何情况下，作者均不对任何索赔、损害或其他责任承担责任，无论是合同诉讼、侵权行为还是其他行为，因使用本软件或与本软件的使用或其他交易有关而产生。
------------------------------------------------------------------------------
*/
```