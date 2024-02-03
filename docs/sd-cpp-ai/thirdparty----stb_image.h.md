# `stable-diffusion.cpp\thirdparty\stb_image.h`

```
/* stb_image - v2.28 - public domain image loader - http://nothings.org/stb
                                  no warranty implied; use at your own risk

   Do this:
      #define STB_IMAGE_IMPLEMENTATION
   before you include this file in *one* C or C++ file to create the implementation.

   // i.e. it should look like this:
   #include ...
   #include ...
   #include ...
   #define STB_IMAGE_IMPLEMENTATION
   #include "stb_image.h"

   You can #define STBI_ASSERT(x) before the #include to avoid using assert.h.
   And #define STBI_MALLOC, STBI_REALLOC, and STBI_FREE to avoid using malloc,realloc,free

   QUICK NOTES:
      Primarily of interest to game developers and other people who can
          avoid problematic images and only need the trivial interface

      JPEG baseline & progressive (12 bpc/arithmetic not supported, same as stock IJG lib)
      PNG 1/2/4/8/16-bit-per-channel

      TGA (not sure what subset, if a subset)
      BMP non-1bpp, non-RLE
      PSD (composited view only, no extra channels, 8/16 bit-per-channel)

      GIF (*comp always reports as 4-channel)
      HDR (radiance rgbE format)
      PIC (Softimage PIC)
      PNM (PPM and PGM binary only)

      Animated GIF still needs a proper API, but here's one way to do it:
          http://gist.github.com/urraka/685d9a6340b26b830d49

      - decode from memory or through FILE (define STBI_NO_STDIO to remove code)
      - decode from arbitrary I/O callbacks
      - SIMD acceleration on x86/x64 (SSE2) and ARM (NEON)

   Full documentation under "DOCUMENTATION" below.


LICENSE

  See end of file for license information.


*/
# 最近修订历史记录，列出了每个版本的更新内容和日期
RECENT REVISION HISTORY:

      2.28  (2023-01-29) many error fixes, security errors, just tons of stuff
      2.27  (2021-07-11) document stbi_info better, 16-bit PNM support, bug fixes
      2.26  (2020-07-13) many minor fixes
      2.25  (2020-02-02) fix warnings
      2.24  (2020-02-02) fix warnings; thread-local failure_reason and flip_vertically
      2.23  (2019-08-11) fix clang static analysis warning
      2.22  (2019-03-04) gif fixes, fix warnings
      2.21  (2019-02-25) fix typo in comment
      2.20  (2019-02-07) support utf8 filenames in Windows; fix warnings and platform ifdefs
      2.19  (2018-02-11) fix warning
      2.18  (2018-01-30) fix warnings
      2.17  (2018-01-29) bugfix, 1-bit BMP, 16-bitness query, fix warnings
      2.16  (2017-07-23) all functions have 16-bit variants; optimizations; bugfixes
      2.15  (2017-03-18) fix png-1,2,4; all Imagenet JPGs; no runtime SSE detection on GCC
      2.14  (2017-03-03) remove deprecated STBI_JPEG_OLD; fixes for Imagenet JPGs
      2.13  (2016-12-04) experimental 16-bit API, only for PNG so far; fixes
      2.12  (2016-04-02) fix typo in 2.11 PSD fix that caused crashes
      2.11  (2016-04-02) 16-bit PNGS; enable SSE2 in non-gcc x64
                         RGB-format JPEG; remove white matting in PSD;
                         allocate large structures on the stack;
                         correct channel count for PNG & BMP
      2.10  (2016-01-22) avoid warning introduced in 2.09
      2.09  (2016-01-16) 16-bit TGA; comments in PNM files; STBI_REALLOC_SIZED

   See end of file for full revision history.


 ============================    Contributors    =========================

 Image formats                          Extensions, features
    Sean Barrett (jpeg, png, bmp)          Jetro Lauha (stbi_info)
    Nicolas Schulz (hdr, psd)              Martin "SpartanJ" Golini (stbi_info)
    Jonathan Dummer (tga)                  James "moose2000" Brown (iPhone PNG)
    Jean-Marc Lienher (gif)                Ben "Disch" Wenger (io callbacks)
    Tom Seddon (pic)                       Omar Cornut (1/2/4-bit PNG)
    Thatcher Ulrich (psd)                  Nicolas Guillemot (vertical flip)
    Ken Miller (pgm, ppm)                  Richard Mitton (16-bit PSD)
    github:urraka (animated gif)           Junggon Kim (PNM comments)
    Christopher Forseth (animated gif)     Daniel Gibson (16-bit TGA)
                                           socks-the-fox (16-bit PNG)
                                           Jeremy Sawicki (handle all ImageNet JPGs)
# 优化和错误修复
 Optimizations & bugfixes                  
    Fabian "ryg" Giesen                    Anael Seghezzi (is-16-bit query)
    Arseny Kapoulkine                      Simon Breuss (16-bit PNM)
    John-Mark Allen
    Carmelo J Fdez-Aguera

# 错误和警告修复
 Bug & warning fixes
    Marc LeBlanc            David Woo          Guillaume George     Martins Mozeiko
    Christpher Lloyd        Jerry Jansson      Joseph Thomson       Blazej Dariusz Roszkowski
    Phil Jordan                                Dave Moore           Roy Eltham
    Hayaki Saito            Nathan Reed        Won Chun
    Luke Graham             Johan Duparc       Nick Verigakis       the Horde3D community
    Thomas Ruf              Ronny Chevalier                         github:rlyeh
    Janez Zemva             John Bartholomew   Michal Cichon        github:romigrou
    Jonathan Blow           Ken Hamada         Tero Hanninen        github:svdijk
    Eugene Golushkov        Laurent Gomila     Cort Stratton        github:snagar
    Aruelien Pocheville     Sergio Gonzalez    Thibault Reuille     github:Zelex
    Cass Everitt            Ryamond Barbiero                        github:grim210
    Paul Du Bois            Engin Manap        Aldo Culquicondor    github:sammyhw
    Philipp Wiesemann       Dale Weiler        Oriol Ferrer Mesia   github:phprus
    Josh Tobin              Neil Bickford      Matthew Gregan       github:poppolopoppo
    Julian Raschke          Gregory Mullen     Christian Floisand   github:darealshinji
    Baldur Karlsson         Kevin Schmidt      JR Smith             github:Michaelangel007
                            Brad Weinberger    Matvey Cherevko      github:mosra
    Luca Sas                Alexander Veselov  Zack Middleton       [reserved]
    Ryan C. Gordon          [reserved]                              [reserved]
                     DO NOT ADD YOUR NAME HERE

                     Jacko Dirks

  To add your name to the credits, pick a random blank space in the middle and fill it.
  80% of merge conflicts on stb PRs are due to people adding their name at the end
  of the credits.
// 包含防止重复包含的宏定义
#ifndef STBI_INCLUDE_STB_IMAGE_H
#define STBI_INCLUDE_STB_IMAGE_H

// 文档说明
//
// 限制：
//    - 没有12位每通道的JPEG
//    - 没有使用算术编码的JPEG
//    - GIF 图像总是返回 *comp=4
//
// 基本用法（有关HDR用法，请参阅下面的HDR讨论）：
//    int x,y,n;
//    unsigned char *data = stbi_load(filename, &x, &y, &n, 0);
//    // ... 如果数据不为NULL，则处理数据 ...
//    // ... x = 宽度, y = 高度, n = 每像素的8位组件数 ...
//    // ... 将'0'替换为'1'..'4'以强制每像素有那么多组件
//    // ... 但'n'将始终是如果你说0时的数量
//    stbi_image_free(data);
//
// 标准参数：
//    int *x                 -- 输出图像的宽度（以像素为单位）
//    int *y                 -- 输出图像的高度（以像素为单位）
//    int *channels_in_file  -- 输出图像文件中的图像组件数
//    int desired_channels   -- 如果非零，则请求结果中的图像组件数
//
// 图像加载器的返回值是一个指向像素数据的 'unsigned char *'，如果分配失败或图像损坏或无效，则为NULL。像素数据由*y个扫描线和*x个像素组成，每个像素由N个交错的8位组件组成；指向的第一个像素是图像中左上角的像素。无论格式如何，图像扫描线之间或像素之间都没有填充。组件数N为desired_channels（如果desired_channels为非零），否则为*channels_in_file。如果desired_channels为非零，则*channels_in_file具有否则将输出的组件数。例如，如果将desired_channels设置为4，则将始终获得RGBA输出，但您可以检查*channels_in_file以查看它是否是平凡不透明的，例如源图像中只有3个通道。
//
// 具有N个组件的输出图像在每个像素中按以下顺序交错：
//
//     N=#comp     components
//       1           grey
//       2           grey, alpha
//       3           red, green, blue
//       4           red, green, blue, alpha
//
// 如果由于任何原因图像加载失败，返回值将为 NULL，
// 并且 *x, *y, *channels_in_file 将保持不变。可以查询 stbi_failure_reason() 函数，
// 获取一个非常简短、不友好的解释为什么加载失败。定义 STBI_NO_FAILURE_STRINGS
// 可以避免编译这些字符串，定义 STBI_FAILURE_USERMSG 可以得到稍微
// 更友好的消息。
//
// 带调色板的 PNG、BMP、GIF 和 PIC 图像会自动去调色板。
//
// 要查询图像的宽度、高度和组件数，而不必解码整个文件，可以使用 stbi_info 系列函数：
//
//   int x,y,n,ok;
//   ok = stbi_info(filename, &x, &y, &n);
//   // 如果图像是支持的格式，则返回 ok=1 并设置 x, y, n，
//   // 否则返回 0。
//
// 注意，stb_image 在其公共 API 中广泛使用 int 类型表示大小，
// 包括内存缓冲区的大小。这现在是 API 的一部分，因此很难更改而不引起破坏。
// 结果，各种图像加载器都对图像大小有一定限制；这些限制在格式上有所不同，
// 但通常都是接近 2GB 或接近 1GB。当解码后的图像大于此大小时，
// stb_image 解码将失败。
//
// 另外，stb_image 将拒绝具有任何维度设置为大于可配置的 STBI_MAX_DIMENSIONS 的图像文件，
// 默认为 2**24 = 16777216 像素。由于上述内存限制，
// 使得具有这些尺寸的图像正确加载的唯一方法是具有极端的长宽比。
// 无论如何，这里的假设是这样的较大图像可能是畸形的或恶意的。
// 如果确实需要加载具有单独尺寸的图像
// larger than that, and it still fits in the overall size limit, you can
// #define STBI_MAX_DIMENSIONS on your own to be something larger.
//
// ===========================================================================
//
// UNICODE:
//
//   If compiling for Windows and you wish to use Unicode filenames, compile
//   with
//       #define STBI_WINDOWS_UTF8
//   and pass utf8-encoded filenames. Call stbi_convert_wchar_to_utf8 to convert
//   Windows wchar_t filenames to utf8.
//
// ===========================================================================
//
// Philosophy
//
// stb libraries are designed with the following priorities:
//
//    1. easy to use
//    2. easy to maintain
//    3. good performance
//
// Sometimes I let "good performance" creep up in priority over "easy to maintain",
// and for best performance I may provide less-easy-to-use APIs that give higher
// performance, in addition to the easy-to-use ones. Nevertheless, it's important
// to keep in mind that from the standpoint of you, a client of this library,
// all you care about is #1 and #3, and stb libraries DO NOT emphasize #3 above all.
//
// Some secondary priorities arise directly from the first two, some of which
// provide more explicit reasons why performance can't be emphasized.
//
//    - Portable ("ease of use")
//    - Small source code footprint ("easy to maintain")
//    - No dependencies ("ease of use")
//
// ===========================================================================
//
// I/O callbacks
//
// I/O callbacks allow you to read from arbitrary sources, like packaged
// files or some other source. Data read from callbacks are processed
// through a small internal buffer (currently 128 bytes) to try to reduce
// overhead.
//
// The three functions you must define are "read" (reads some bytes of data),
// "skip" (skips some bytes of data), "eof" (reports if the stream is at the end).
//
// ===========================================================================
//
// SIMD support
//
// The JPEG decoder will try to automatically use SIMD kernels on x86 when
// supported by the compiler. For ARM Neon support, you must explicitly
// request it.
//
// (The old do-it-yourself SIMD API is no longer supported in the current
// code.)
//
// On x86, SSE2 will automatically be used when available based on a run-time
// test; if not, the generic C versions are used as a fall-back. On ARM targets,
// the typical path is to have separate builds for NEON and non-NEON devices
// (at least this is true for iOS and Android). Therefore, the NEON support is
// toggled by a build flag: define STBI_NEON to get NEON loops.
//
// If for some reason you do not want to use any of SIMD code, or if
// you have issues compiling it, you can disable it entirely by
// defining STBI_NO_SIMD.
//
// ===========================================================================
//
// HDR image support   (disable by defining STBI_NO_HDR)
//
// stb_image supports loading HDR images in general, and currently the Radiance
// .HDR file format specifically. You can still load any file through the existing
// interface; if you attempt to load an HDR file, it will be automatically remapped
// to LDR, assuming gamma 2.2 and an arbitrary scale factor defaulting to 1;
// both of these constants can be reconfigured through this interface:
//
//     stbi_hdr_to_ldr_gamma(2.2f);
//     stbi_hdr_to_ldr_scale(1.0f);
//
// (note, do not use _inverse_ constants; stbi_image will invert them
// appropriately).
//
// Additionally, there is a new, parallel interface for loading files as
// (linear) floats to preserve the full dynamic range:
//
//    float *data = stbi_loadf(filename, &x, &y, &n, 0);
//
// If you load LDR images through this interface, those images will
// be promoted to floating point values, run through the inverse of
// constants corresponding to the above:
//
// 设置图像加载时的缩放比例为1.0
// 设置图像加载时的 gamma 值为2.2
//
// 最后，给定一个包含图像数据的文件名（或打开的文件或内存块-详见头文件的详细信息），您可以查询“最合适”的接口来使用（即，图像是否为 HDR），使用：
//
//     stbi_is_hdr(char *filename);
//
// ===========================================================================
//
// iPhone PNG 支持:
//
// 我们可选地支持将 iPhone 格式的 PNG（存储预乘的 BGRA）转换回 RGB，即使它们在内部编码方式不同。要启用此转换，请调用
// stbi_convert_iphone_png_to_rgb(1)。
//
// 还要调用 stbi_set_unpremultiply_on_load(1) 来强制每像素进行除法，以去除任何预乘的 alpha *仅*如果图像文件明确指出存在预乘数据（目前仅在 iPhone 图像中发生，并且仅在启用 iPhone 转换为 RGB 处理时发生）。
//
// ===========================================================================
//
// 附加配置
//
//  - 您可以通过在创建实现之前 #define 以下一个或多个符号来抑制任何解码器的实现，以减少代码占用空间。
//
//        STBI_NO_JPEG
//        STBI_NO_PNG
//        STBI_NO_BMP
//        STBI_NO_PSD
//        STBI_NO_TGA
//        STBI_NO_GIF
//        STBI_NO_HDR
//        STBI_NO_PIC
//        STBI_NO_PNM   （.ppm 和 .pgm）
//
//  - 您可以仅请求某些解码器，并抑制所有其他解码器（这将更具前向兼容性，因为添加新解码器不需要显式禁用它们）：
//
//        STBI_ONLY_JPEG
//        STBI_ONLY_PNG
//        STBI_ONLY_BMP
//        STBI_ONLY_PSD
//        STBI_ONLY_TGA
//        STBI_ONLY_GIF
//        STBI_ONLY_HDR
//        STBI_ONLY_PIC
//        STBI_ONLY_PNM   （.ppm 和 .pgm）
//
// 如果使用 STBI_NO_PNG（或者仅使用没有 PNG 的选项），但仍希望 zlib 解码器可用，请定义 STBI_SUPPORT_ZLIB
// 如果定义了 STBI_MAX_DIMENSIONS，stb_image 将拒绝大于该尺寸（宽度或高度）的图像，不进行进一步处理。
// 这是为了让程序在野外设置一个上限，以防止拒绝服务攻击未经信任的数据，因为可以生成一个巨大尺寸的有效图像，并强制 stb_image 分配一个巨大的内存块并花费不成比例的时间解码它。默认设置为 (1 << 24)，即 16777216，但仍然非常大。

#ifndef STBI_NO_STDIO
#include <stdio.h>
#endif // STBI_NO_STDIO

#define STBI_VERSION 1

enum
{
   STBI_default = 0, // 仅用于 desired_channels

   STBI_grey       = 1,
   STBI_grey_alpha = 2,
   STBI_rgb        = 3,
   STBI_rgb_alpha  = 4
};

#include <stdlib.h>
typedef unsigned char stbi_uc;
typedef unsigned short stbi_us;

#ifdef __cplusplus
extern "C" {
#endif

#ifndef STBIDEF
#ifdef STB_IMAGE_STATIC
#define STBIDEF static
#else
#define STBIDEF extern
#endif
#endif

//////////////////////////////////////////////////////////////////////////////
//
// PRIMARY API - works on images of any type
//

//
// 通过文件名、打开文件或内存缓冲加载图像
//

typedef struct
{
   int      (*read)  (void *user,char *data,int size);   // 用 'size' 字节填充 'data'。返回实际读取的字节数
   void     (*skip)  (void *user,int n);                 // 跳过接下来的 'n' 字节，或者如果是负数，则 'unget' 最后的 -n 字节
   int      (*eof)   (void *user);                       // 如果我们在文件/数据的末尾，则返回非零值
} stbi_io_callbacks;

////////////////////////////////////
//
// 8 位每通道接口
//

STBIDEF stbi_uc *stbi_load_from_memory   (stbi_uc           const *buffer, int len   , int *x, int *y, int *channels_in_file, int desired_channels);
// 从回调函数中加载图像数据，返回图像数据指针
STBIDEF stbi_uc *stbi_load_from_callbacks(stbi_io_callbacks const *clbk  , void *user, int *x, int *y, int *channels_in_file, int desired_channels);

#ifndef STBI_NO_STDIO
// 从文件名加载图像数据，返回图像数据指针
STBIDEF stbi_uc *stbi_load            (char const *filename, int *x, int *y, int *channels_in_file, int desired_channels);
// 从文件指针加载图像数据，返回图像数据指针，文件指针在图像后面
STBIDEF stbi_uc *stbi_load_from_file  (FILE *f, int *x, int *y, int *channels_in_file, int desired_channels);
#endif

#ifndef STBI_NO_GIF
// 从内存中加载 GIF 图像数据，返回图像数据指针
STBIDEF stbi_uc *stbi_load_gif_from_memory(stbi_uc const *buffer, int len, int **delays, int *x, int *y, int *z, int *comp, int req_comp);
#endif

#ifdef STBI_WINDOWS_UTF8
// 将宽字符转换为 UTF-8 字符串
STBIDEF int stbi_convert_wchar_to_utf8(char *buffer, size_t bufferlen, const wchar_t* input);
#endif

////////////////////////////////////
//
// 16位每通道接口
//

// 从内存中加载 16 位每通道图像数据，返回图像数据指针
STBIDEF stbi_us *stbi_load_16_from_memory   (stbi_uc const *buffer, int len, int *x, int *y, int *channels_in_file, int desired_channels);
// 从回调函数中加载 16 位每通道图像数据，返回图像数据指针
STBIDEF stbi_us *stbi_load_16_from_callbacks(stbi_io_callbacks const *clbk, void *user, int *x, int *y, int *channels_in_file, int desired_channels);

#ifndef STBI_NO_STDIO
// 从文件名加载 16 位每通道图像数据，返回图像数据指针
STBIDEF stbi_us *stbi_load_16          (char const *filename, int *x, int *y, int *channels_in_file, int desired_channels);
// 从文件指针加载 16 位每通道图像数据，返回图像数据指针
STBIDEF stbi_us *stbi_load_from_file_16(FILE *f, int *x, int *y, int *channels_in_file, int desired_channels);
#endif

////////////////////////////////////
//
// 浮点数每通道接口
//
// 如果未定义 STBI_NO_LINEAR，则声明从内存加载浮点数数据的函数
STBIDEF float *stbi_loadf_from_memory     (stbi_uc const *buffer, int len, int *x, int *y, int *channels_in_file, int desired_channels);
// 如果未定义 STBI_NO_LINEAR，则声明从回调函数加载浮点数数据的函数
STBIDEF float *stbi_loadf_from_callbacks  (stbi_io_callbacks const *clbk, void *user, int *x, int *y,  int *channels_in_file, int desired_channels);

#ifndef STBI_NO_STDIO
// 如果未定义 STBI_NO_STDIO，则声明从文件加载浮点数数据的函数
STBIDEF float *stbi_loadf            (char const *filename, int *x, int *y, int *channels_in_file, int desired_channels);
// 如果未定义 STBI_NO_STDIO，则声明从文件指针加载浮点数数据的函数
STBIDEF float *stbi_loadf_from_file  (FILE *f, int *x, int *y, int *channels_in_file, int desired_channels);
#endif

#ifndef STBI_NO_HDR
// 如果未定义 STBI_NO_HDR，则声明将 HDR 转换为 LDR 的 gamma 函数
STBIDEF void   stbi_hdr_to_ldr_gamma(float gamma);
// 如果未定义 STBI_NO_HDR，则声明将 HDR 转换为 LDR 的 scale 函数
STBIDEF void   stbi_hdr_to_ldr_scale(float scale);
#endif // STBI_NO_HDR

#ifndef STBI_NO_LINEAR
// 如果未定义 STBI_NO_LINEAR，则声明将 LDR 转换为 HDR 的 gamma 函数
STBIDEF void   stbi_ldr_to_hdr_gamma(float gamma);
// 如果未定义 STBI_NO_LINEAR，则声明将 LDR 转换为 HDR 的 scale 函数
STBIDEF void   stbi_ldr_to_hdr_scale(float scale);
#endif // STBI_NO_LINEAR

// 始终定义 stbi_is_hdr，但如果定义了 STBI_NO_HDR，则始终返回 false
STBIDEF int    stbi_is_hdr_from_callbacks(stbi_io_callbacks const *clbk, void *user);
STBIDEF int    stbi_is_hdr_from_memory(stbi_uc const *buffer, int len);
#ifndef STBI_NO_STDIO
// 如果未定义 STBI_NO_STDIO，则声明检查文件是否为 HDR 格式的函数
STBIDEF int      stbi_is_hdr          (char const *filename);
// 如果未定义 STBI_NO_STDIO，则声明检查文件指针是否为 HDR 格式的函数
STBIDEF int      stbi_is_hdr_from_file(FILE *f);
#endif // STBI_NO_STDIO

// 获取加载失败的简短原因
// 在大多数编译器（以及所有现代主流编译器）上，这是线程安全的
STBIDEF const char *stbi_failure_reason  (void);

// 释放加载的图像 -- 这只是 free() 函数
STBIDEF void     stbi_image_free      (void *retval_from_stbi_load);

// 获取图像的尺寸和组件数，而不完全解码
STBIDEF int      stbi_info_from_memory(stbi_uc const *buffer, int len, int *x, int *y, int *comp);
STBIDEF int      stbi_info_from_callbacks(stbi_io_callbacks const *clbk, void *user, int *x, int *y, int *comp);
STBIDEF int      stbi_is_16_bit_from_memory(stbi_uc const *buffer, int len);
// 检查给定的回调函数和用户数据是否表示16位图像
STBIDEF int stbi_is_16_bit_from_callbacks(stbi_io_callbacks const *clbk, void *user);

#ifndef STBI_NO_STDIO
// 从文件名获取图像信息，包括宽度、高度和通道数
STBIDEF int stbi_info(char const *filename, int *x, int *y, int *comp);
// 从文件指针获取图像信息，包括宽度、高度和通道数
STBIDEF int stbi_info_from_file(FILE *f, int *x, int *y, int *comp);
// 检查文件是否为16位图像
STBIDEF int stbi_is_16_bit(char const *filename);
// 从文件指针检查是否为16位图像
STBIDEF int stbi_is_16_bit_from_file(FILE *f);
#endif

// 设置是否在加载时取消预乘alpha通道
STBIDEF void stbi_set_unpremultiply_on_load(int flag_true_if_should_unpremultiply);
// 设置是否将iPhone图像转换为RGB格式
STBIDEF void stbi_convert_iphone_png_to_rgb(int flag_true_if_should_convert);
// 设置是否垂直翻转图像，使输出数组的第一个像素为左下角
STBIDEF void stbi_set_flip_vertically_on_load(int flag_true_if_should_flip);
// 与上述功能相同，但仅适用于调用该函数的线程加载的图像
STBIDEF void stbi_set_unpremultiply_on_load_thread(int flag_true_if_should_unpremultiply);
STBIDEF void stbi_convert_iphone_png_to_rgb_thread(int flag_true_if_should_convert);
STBIDEF void stbi_set_flip_vertically_on_load_thread(int flag_true_if_should_flip);

// ZLIB客户端 - 用于PNG，也可用于其他用途

// 猜测解码后的数据大小并分配内存
STBIDEF char *stbi_zlib_decode_malloc_guesssize(const char *buffer, int len, int initial_size, int *outlen);
// 带有标志的猜测解码后的数据大小并分配内存
STBIDEF char *stbi_zlib_decode_malloc_guesssize_headerflag(const char *buffer, int len, int initial_size, int *outlen, int parse_header);
// 解码数据并分配内存
STBIDEF char *stbi_zlib_decode_malloc(const char *buffer, int len, int *outlen);
// 定义一个函数，用于解码经过 zlib 压缩的数据到指定的输出缓冲区
STBIDEF int   stbi_zlib_decode_buffer(char *obuffer, int olen, const char *ibuffer, int ilen);

// 定义一个函数，用于在不包含头部信息的情况下解码经过 zlib 压缩的数据到动态分配的内存中
STBIDEF char *stbi_zlib_decode_noheader_malloc(const char *buffer, int len, int *outlen);
// 定义一个函数，用于在不包含头部信息的情况下解码经过 zlib 压缩的数据到指定的输出缓冲区
STBIDEF int   stbi_zlib_decode_noheader_buffer(char *obuffer, int olen, const char *ibuffer, int ilen);

#ifdef __cplusplus
}
#endif

//
//
////   end header file   /////////////////////////////////////////////////////
#endif // STBI_INCLUDE_STB_IMAGE_H

#ifdef STB_IMAGE_IMPLEMENTATION

#if defined(STBI_ONLY_JPEG) || defined(STBI_ONLY_PNG) || defined(STBI_ONLY_BMP) \
  || defined(STBI_ONLY_TGA) || defined(STBI_ONLY_GIF) || defined(STBI_ONLY_PSD) \
  || defined(STBI_ONLY_HDR) || defined(STBI_ONLY_PIC) || defined(STBI_ONLY_PNM) \
  || defined(STBI_ONLY_ZLIB)
   #ifndef STBI_ONLY_JPEG
   #define STBI_NO_JPEG
   #endif
   #ifndef STBI_ONLY_PNG
   #define STBI_NO_PNG
   #endif
   #ifndef STBI_ONLY_BMP
   #define STBI_NO_BMP
   #endif
   #ifndef STBI_ONLY_PSD
   #define STBI_NO_PSD
   #endif
   #ifndef STBI_ONLY_TGA
   #define STBI_NO_TGA
   #endif
   #ifndef STBI_ONLY_GIF
   #define STBI_NO_GIF
   #endif
   #ifndef STBI_ONLY_HDR
   #define STBI_NO_HDR
   #endif
   #ifndef STBI_ONLY_PIC
   #define STBI_NO_PIC
   #endif
   #ifndef STBI_ONLY_PNM
   #define STBI_NO_PNM
   #endif
#endif

#if defined(STBI_NO_PNG) && !defined(STBI_SUPPORT_ZLIB) && !defined(STBI_NO_ZLIB)
#define STBI_NO_ZLIB
#endif

// 包含标准头文件
#include <stdarg.h>
#include <stddef.h> // ptrdiff_t on osx
#include <stdlib.h>
#include <string.h>
#include <limits.h>

// 如果不禁用线性插值或 HDR 支持，则包含 math.h 头文件
#if !defined(STBI_NO_LINEAR) || !defined(STBI_NO_HDR)
#include <math.h>  // ldexp, pow
#endif

// 如果未禁用标准输入输出，则包含 stdio.h 头文件
#ifndef STBI_NO_STDIO
#include <stdio.h>
#endif

// 如果未定义 STBI_ASSERT 宏，则使用 assert.h 头文件定义 STBI_ASSERT 宏
#ifndef STBI_ASSERT
#include <assert.h>
#define STBI_ASSERT(x) assert(x)
#endif

#ifdef __cplusplus
#define STBI_EXTERN extern "C"
#else
#define STBI_EXTERN extern
#endif

// 如果不是 MSC 编译器，则定义 stbi_inline 宏
#ifndef _MSC_VER
   #ifdef __cplusplus
   #define stbi_inline inline
   #else
   #define stbi_inline
   #endif
#else
   #define stbi_inline __forceinline
#endif
#ifndef STBI_NO_THREAD_LOCALS
   #if defined(__cplusplus) &&  __cplusplus >= 201103L
      #define STBI_THREAD_LOCAL       thread_local
   #elif defined(__GNUC__) && __GNUC__ < 5
      #define STBI_THREAD_LOCAL       __thread
   #elif defined(_MSC_VER)
      #define STBI_THREAD_LOCAL       __declspec(thread)
   #elif defined (__STDC_VERSION__) && __STDC_VERSION__ >= 201112L && !defined(__STDC_NO_THREADS__)
      #define STBI_THREAD_LOCAL       _Thread_local
   #endif

   #ifndef STBI_THREAD_LOCAL
      #if defined(__GNUC__)
        #define STBI_THREAD_LOCAL       __thread
      #endif
   #endif
#endif

#if defined(_MSC_VER) || defined(__SYMBIAN32__)
typedef unsigned short stbi__uint16;
typedef   signed short stbi__int16;
typedef unsigned int   stbi__uint32;
typedef   signed int   stbi__int32;
#else
#include <stdint.h>
typedef uint16_t stbi__uint16;
typedef int16_t  stbi__int16;
typedef uint32_t stbi__uint32;
typedef int32_t  stbi__int32;
#endif

// should produce compiler error if size is wrong
typedef unsigned char validate_uint32[sizeof(stbi__uint32)==4 ? 1 : -1];

#ifdef _MSC_VER
#define STBI_NOTUSED(v)  (void)(v)
#else
#define STBI_NOTUSED(v)  (void)sizeof(v)
#endif

#ifdef _MSC_VER
#define STBI_HAS_LROTL
#endif

#ifdef STBI_HAS_LROTL
   #define stbi_lrot(x,y)  _lrotl(x,y)
#else
   #define stbi_lrot(x,y)  (((x) << (y)) | ((x) >> (-(y) & 31)))
#endif

#if defined(STBI_MALLOC) && defined(STBI_FREE) && (defined(STBI_REALLOC) || defined(STBI_REALLOC_SIZED))
// ok
#elif !defined(STBI_MALLOC) && !defined(STBI_FREE) && !defined(STBI_REALLOC) && !defined(STBI_REALLOC_SIZED)
// ok
#else
#error "Must define all or none of STBI_MALLOC, STBI_FREE, and STBI_REALLOC (or STBI_REALLOC_SIZED)."
#endif

#ifndef STBI_MALLOC
#define STBI_MALLOC(sz)           malloc(sz)
#define STBI_REALLOC(p,newsz)     realloc(p,newsz)
#define STBI_FREE(p)              free(p)
#endif

#ifndef STBI_REALLOC_SIZED
#define STBI_REALLOC_SIZED(p,oldsz,newsz) STBI_REALLOC(p,newsz)
#endif
// 检测当前编译环境是否为 x86/x64 架构
#if defined(__x86_64__) || defined(_M_X64)
#define STBI__X64_TARGET
#elif defined(__i386) || defined(_M_IX86)
#define STBI__X86_TARGET
#endif

// 如果使用的是 GCC 编译器，并且目标架构为 x86，并且未定义 __SSE2__，并且未定义 STBI_NO_SIMD
#if defined(__GNUC__) && defined(STBI__X86_TARGET) && !defined(__SSE2__) && !defined(STBI_NO_SIMD)
// GCC 编译器在没有使用 -msse2 编译选项的情况下不支持 SSE2 指令集，
// 而使用 -msse2 编译选项会在整个程序中启用 SSE2，这是不太理想的。
// 之前尝试在运行时检测 SSE2 函数，但导致了许多问题。
// GCC/Clang 中架构扩展的暴露方式，遗憾地不太适合单文件库。
// 新的行为：如果使用 -msse2 编译，我们将直接使用 SSE2，否则不使用。
#define STBI_NO_SIMD
#endif

// 如果使用的是 MinGW 编译器，并且目标架构为 x86，并且未定义 STBI_MINGW_ENABLE_SSE2，并且未定义 STBI_NO_SIMD
#if defined(__MINGW32__) && defined(STBI__X86_TARGET) && !defined(STBI_MINGW_ENABLE_SSE2) && !defined(STBI_NO_SIMD)
// 注意，__MINGW32__ 实际上并不表示 32 位，因此我们必须避免 STBI__X64_TARGET
//
// 32 位 MinGW 要求 ESP 寄存器的 16 字节对齐，但这不符合 Windows ABI，VC++ 以及 Windows DLLs 不保持这个不变性。
// 因此，在 32 位 MinGW 上启用 SSE2 在不同时启用 "-mstackrealign" 是危险的。
// 有关更多信息，请参阅 https://github.com/nothings/stb/issues/81
//
// 因此，默认情况下在 32 位 MinGW 上不启用 SSE2。如果您已经阅读到这里并在构建设置中添加了 -mstackrealign，请随意 #define STBI_MINGW_ENABLE_SSE2。
#define STBI_NO_SIMD
#endif

// 如果未定义 STBI_NO_SIMD，并且目标架构为 x86 或 x64
#if !defined(STBI_NO_SIMD) && (defined(STBI__X86_TARGET) || defined(STBI__X64_TARGET))
#define STBI_SSE2
#include <emmintrin.h>

#ifdef _MSC_VER

#if _MSC_VER >= 1400  // 不是 VC6
#include <intrin.h> // __cpuid
static int stbi__cpuid3(void)
{
   int info[4];
   __cpuid(info,1);
   return info[3];
}
#else
static int stbi__cpuid3(void)
{
   int res;
   __asm {
      mov  eax,1
      cpuid
      mov  res,edx
   }
   return res;
}
#endif
// 定义一个宏，用于指定变量的内存对齐方式为 16 字节
#define STBI_SIMD_ALIGN(type, name) __declspec(align(16)) type name

// 如果不是 VC++ 编译器，则使用 GCC 风格的内存对齐方式
#else // assume GCC-style if not VC++
#define STBI_SIMD_ALIGN(type, name) type name __attribute__((aligned(16)))

// 如果不禁用 JPEG 解码并且启用了 SSE2 指令集，则定义一个函数检查 SSE2 是否可用
#if !defined(STBI_NO_JPEG) && defined(STBI_SSE2)
static int stbi__sse2_available(void)
{
   // 调用 stbi__cpuid3 函数获取 CPU 信息，判断是否支持 SSE2 指令集
   int info3 = stbi__cpuid3();
   // 返回 SSE2 是否可用的结果
   return ((info3 >> 26) & 1) != 0;
}
#endif

// ARM NEON
#if defined(STBI_NO_SIMD) && defined(STBI_NEON)
// 如果禁用了 SIMD 并且启用了 NEON，则取消定义 NEON
#undef STBI_NEON
#endif

// 如果定义了 NEON，则包含 arm_neon.h 头文件，并根据编译器类型定义内存对齐方式
#ifdef STBI_NEON
#include <arm_neon.h>
#ifdef _MSC_VER
#define STBI_SIMD_ALIGN(type, name) __declspec(align(16)) type name
#else
#define STBI_SIMD_ALIGN(type, name) type name __attribute__((aligned(16)))
#endif
#endif

// 如果未定义 STBI_SIMD_ALIGN，则定义一个默认的内存对齐方式
#ifndef STBI_SIMD_ALIGN
#define STBI_SIMD_ALIGN(type, name) type name
#endif

// 如果未定义 STBI_MAX_DIMENSIONS，则定义一个默认的最大尺寸
#ifndef STBI_MAX_DIMENSIONS
#define STBI_MAX_DIMENSIONS (1 << 24)
#endif

///////////////////////////////////////////////
//
//  stbi__context 结构体和 start_xxx 函数

// stbi__context 结构体是所有图像使用的基本上下文，包含所有 IO 上下文以及一些基本的图像信息
typedef struct
{
   stbi__uint32 img_x, img_y;
   int img_n, img_out_n;

   stbi_io_callbacks io;
   void *io_user_data;

   int read_from_callbacks;
   int buflen;
   stbi_uc buffer_start[128];
   int callback_already_read;

   stbi_uc *img_buffer, *img_buffer_end;
   stbi_uc *img_buffer_original, *img_buffer_original_end;
} stbi__context;

// 填充缓冲区的函数
static void stbi__refill_buffer(stbi__context *s);

// 初始化一个内存解码上下文
static void stbi__start_mem(stbi__context *s, stbi_uc const *buffer, int len)
{
   // 初始化读取函数为空
   s->io.read = NULL;
   // 读取回调函数标志位设为0
   s->read_from_callbacks = 0;
   // 回调函数已读标志位设为0
   s->callback_already_read = 0;
   // 图像缓冲区指针指向传入的缓冲区起始位置
   s->img_buffer = s->img_buffer_original = (stbi_uc *) buffer;
   // 图像缓冲区结束指针指向传入的缓冲区结束位置
   s->img_buffer_end = s->img_buffer_original_end = (stbi_uc *) buffer+len;
}

// 初始化基于回调的上下文
static void stbi__start_callbacks(stbi__context *s, stbi_io_callbacks *c, void *user)
{
   // 将回调函数结构体赋值给上下文的io字段
   s->io = *c;
   // 设置回调函数的用户数据
   s->io_user_data = user;
   // 缓冲区长度设为初始缓冲区的大小
   s->buflen = sizeof(s->buffer_start);
   // 读取回调函数标志位设为1
   s->read_from_callbacks = 1;
   // 回调函数已读标志位设为0
   s->callback_already_read = 0;
   // 图像缓冲区指针指向初始缓冲区起始位置
   s->img_buffer = s->img_buffer_original = s->buffer_start;
   // 填充缓冲区
   stbi__refill_buffer(s);
   // 图像缓冲区结束指针指向初始缓冲区结束位置
   s->img_buffer_original_end = s->img_buffer_end;
}

#ifndef STBI_NO_STDIO

static int stbi__stdio_read(void *user, char *data, int size)
{
   // 从文件中读取数据
   return (int) fread(data,1,size,(FILE*) user);
}

static void stbi__stdio_skip(void *user, int n)
{
   // 跳过文件中的数据
   int ch;
   fseek((FILE*) user, n, SEEK_CUR);
   ch = fgetc((FILE*) user);  /* have to read a byte to reset feof()'s flag */
   if (ch != EOF) {
      ungetc(ch, (FILE *) user);  /* push byte back onto stream if valid. */
   }
}

static int stbi__stdio_eof(void *user)
{
   // 检查文件是否结束
   return feof((FILE*) user) || ferror((FILE *) user);
}

// 定义标准IO回调函数结构体
static stbi_io_callbacks stbi__stdio_callbacks =
{
   stbi__stdio_read,
   stbi__stdio_skip,
   stbi__stdio_eof,
};

// 初始化文件上下文
static void stbi__start_file(stbi__context *s, FILE *f)
{
   // 使用标准IO回调函数初始化上下文
   stbi__start_callbacks(s, &stbi__stdio_callbacks, (void *) f);
}

//static void stop_file(stbi__context *s) { }

#endif // !STBI_NO_STDIO

static void stbi__rewind(stbi__context *s)
{
   // 重新定位到初始缓冲区的起始位置
   s->img_buffer = s->img_buffer_original;
   // 重新定位到初始缓冲区的结束位置
   s->img_buffer_end = s->img_buffer_original_end;
}

// 定义通道顺序枚举
enum
{
   STBI_ORDER_RGB,
   STBI_ORDER_BGR
};

// 定义图像信息结构体
typedef struct
{
   int bits_per_channel;
   int num_channels;
   int channel_order;
// 如果未定义 STBI_NO_JPEG，则声明 JPEG 相关函数
static int      stbi__jpeg_test(stbi__context *s);
static void    *stbi__jpeg_load(stbi__context *s, int *x, int *y, int *comp, int req_comp, stbi__result_info *ri);
static int      stbi__jpeg_info(stbi__context *s, int *x, int *y, int *comp);

// 如果未定义 STBI_NO_PNG，则声明 PNG 相关函数
static int      stbi__png_test(stbi__context *s);
static void    *stbi__png_load(stbi__context *s, int *x, int *y, int *comp, int req_comp, stbi__result_info *ri);
static int      stbi__png_info(stbi__context *s, int *x, int *y, int *comp);
static int      stbi__png_is16(stbi__context *s);

// 如果未定义 STBI_NO_BMP，则声明 BMP 相关函数
static int      stbi__bmp_test(stbi__context *s);
static void    *stbi__bmp_load(stbi__context *s, int *x, int *y, int *comp, int req_comp, stbi__result_info *ri);
static int      stbi__bmp_info(stbi__context *s, int *x, int *y, int *comp);

// 如果未定义 STBI_NO_TGA，则声明 TGA 相关函数
static int      stbi__tga_test(stbi__context *s);
static void    *stbi__tga_load(stbi__context *s, int *x, int *y, int *comp, int req_comp, stbi__result_info *ri);
static int      stbi__tga_info(stbi__context *s, int *x, int *y, int *comp);

// 如果未定义 STBI_NO_PSD，则声明 PSD 相关函数
static int      stbi__psd_test(stbi__context *s);
static void    *stbi__psd_load(stbi__context *s, int *x, int *y, int *comp, int req_comp, stbi__result_info *ri, int bpc);
static int      stbi__psd_info(stbi__context *s, int *x, int *y, int *comp);
static int      stbi__psd_is16(stbi__context *s);

// 如果未定义 STBI_NO_HDR，则声明 HDR 相关函数
static int      stbi__hdr_test(stbi__context *s);
static float   *stbi__hdr_load(stbi__context *s, int *x, int *y, int *comp, int req_comp, stbi__result_info *ri);
static int      stbi__hdr_info(stbi__context *s, int *x, int *y, int *comp);

// 如果未定义 STBI_NO_PIC，则声明 PIC 相关函数
static int      stbi__pic_test(stbi__context *s);
static void    *stbi__pic_load(stbi__context *s, int *x, int *y, int *comp, int req_comp, stbi__result_info *ri);
// 检查两个整数相加是否会导致溢出，返回1表示有效，0表示溢出
// 负数被视为无效
static int stbi__addsizes_valid(int a, int b)
{



    // 如果任一操作数为负数，则返回0表示无效
    if (b < 0 || a < 0) return 0;
    // 如果a+b的结果小于a或b，则表示溢出，返回0表示无效
    if (a > INT_MAX - b) return 0;
    // 否则返回1表示有效
    return 1;
}
{
   // 如果 b 小于 0，则返回 0
   if (b < 0) return 0;
   // 现在 0 <= b <= INT_MAX，因此也有 0 <= INT_MAX - b <= INTMAX。
   // 并且 "a + b <= INT_MAX"（可能会溢出）等同于 a <= INT_MAX - b（不会溢出）
   return a <= INT_MAX - b;
}

// 如果乘积有效，则返回 1，溢出时返回 0。负因子被视为无效。
static int stbi__mul2sizes_valid(int a, int b)
{
   // 如果 a 或 b 小于 0，则返回 0
   if (a < 0 || b < 0) return 0;
   // 如果 b 等于 0，则返回 1，乘以 0 总是安全的
   if (b == 0) return 1;
   // 检查 a*b 是否不会溢出的可移植方式
   return a <= INT_MAX/b;
}

#if !defined(STBI_NO_JPEG) || !defined(STBI_NO_PNG) || !defined(STBI_NO_TGA) || !defined(STBI_NO_HDR)
// 如果 "a*b + add" 没有负项/因子且不会溢出，则返回 1
static int stbi__mad2sizes_valid(int a, int b, int add)
{
   return stbi__mul2sizes_valid(a, b) && stbi__addsizes_valid(a*b, add);
}
#endif

// 如果 "a*b*c + add" 没有负项/因子且不会溢出，则返回 1
static int stbi__mad3sizes_valid(int a, int b, int c, int add)
{
   return stbi__mul2sizes_valid(a, b) && stbi__mul2sizes_valid(a*b, c) &&
      stbi__addsizes_valid(a*b*c, add);
}

// 如果 "a*b*c*d + add" 没有负项/因子且不会溢出，则返回 1
#if !defined(STBI_NO_LINEAR) || !defined(STBI_NO_HDR) || !defined(STBI_NO_PNM)
static int stbi__mad4sizes_valid(int a, int b, int c, int d, int add)
{
   return stbi__mul2sizes_valid(a, b) && stbi__mul2sizes_valid(a*b, c) &&
      stbi__mul2sizes_valid(a*b*c, d) && stbi__addsizes_valid(a*b*c*d, add);
}
#endif

#if !defined(STBI_NO_JPEG) || !defined(STBI_NO_PNG) || !defined(STBI_NO_TGA) || !defined(STBI_NO_HDR)
// 使用大小溢出检查的 malloc
static void *stbi__malloc_mad2(int a, int b, int add)
{
   // 如果乘积有效，则分配内存，否则返回 NULL
   if (!stbi__mad2sizes_valid(a, b, add)) return NULL;
   return stbi__malloc(a*b + add);
}
#endif

// 使用大小溢出检查的 malloc
static void *stbi__malloc_mad3(int a, int b, int c, int add)
{
   // 如果乘积有效，则分配内存，否则返回 NULL
   if (!stbi__mad3sizes_valid(a, b, c, add)) return NULL;
   return stbi__malloc(a*b*c + add);
}
// 如果未定义 STBI_NO_LINEAR 或 STBI_NO_HDR 或 STBI_NO_PNM，则定义 stbi__malloc_mad4 函数
static void *stbi__malloc_mad4(int a, int b, int c, int d, int add)
{
   // 如果给定的参数不会导致溢出，则返回分配的内存块
   if (!stbi__mad4sizes_valid(a, b, c, d, add)) return NULL;
   return stbi__malloc(a*b*c*d + add);
}

// 返回 1 如果两个有符号整数的和有效（在 -2^31 和 2^31-1 之间，包括边界），在溢出时返回 0
static int stbi__addints_valid(int a, int b)
{
   // 如果 a 和 b 的符号不同，则没有溢出
   if ((a >= 0) != (b >= 0)) return 1;
   // 如果 a 和 b 都小于 0，则返回 a 是否大于等于 INT_MIN - b；INT_MIN - b 不会溢出，因为 b 小于 0
   if (a < 0 && b < 0) return a >= INT_MIN - b;
   // 返回 a 是否小于等于 INT_MAX - b
   return a <= INT_MAX - b;
}

// 返回 1 如果两个有符号短整数的乘积有效，溢出时返回 0
static int stbi__mul2shorts_valid(short a, short b)
{
   // 如果 b 为 0 或 -1，则返回 1；乘以 0 总是 0；检查 -1 是为了避免 SHRT_MIN/b 溢出
   if (b == 0 || b == -1) return 1;
   // 如果 a 和 b 的符号相同，则返回 a 是否小于等于 SHRT_MAX/b；乘积为正数，类似于 mul2sizes_valid
   if ((a >= 0) == (b >= 0)) return a <= SHRT_MAX/b;
   // 如果 b 小于 0，则返回 a 是否小于等于 SHRT_MIN / b；同样，a * b 是否大于等于 SHRT_MIN
   if (b < 0) return a <= SHRT_MIN / b;
   // 返回 a 是否大于等于 SHRT_MIN / b
   return a >= SHRT_MIN / b;
}

// 定义错误处理宏
#ifdef STBI_NO_FAILURE_STRINGS
   #define stbi__err(x,y)  0
#elif defined(STBI_FAILURE_USERMSG)
   #define stbi__err(x,y)  stbi__err(y)
#else
   #define stbi__err(x,y)  stbi__err(x)
#endif

// 定义返回指向浮点数的指针的错误处理宏
#define stbi__errpf(x,y)   ((float *)(size_t) (stbi__err(x,y)?NULL:NULL))
// 定义返回指向无符号字符的指针的错误处理宏
#define stbi__errpuc(x,y)  ((unsigned char *)(size_t) (stbi__err(x,y)?NULL:NULL))

// 释放由 stbi_load 返回的内存块
STBIDEF void stbi_image_free(void *retval_from_stbi_load)
{
   STBI_FREE(retval_from_stbi_load);
}

// 如果未定义 STBI_NO_LINEAR，则定义 stbi__ldr_to_hdr 函数
#ifndef STBI_NO_LINEAR
static float   *stbi__ldr_to_hdr(stbi_uc *data, int x, int y, int comp);
#endif

// 如果未定义 STBI_NO_HDR，则定义 stbi__hdr_to_ldr 函数
#ifndef STBI_NO_HDR
static stbi_uc *stbi__hdr_to_ldr(float   *data, int x, int y, int comp);
#endif

// 全局变量，用于在加载时垂直翻转图像
static int stbi__vertically_flip_on_load_global = 0;
// 设置是否在加载时垂直翻转图像
STBIDEF void stbi_set_flip_vertically_on_load(int flag_true_if_should_flip)
{
   // 将全局变量 stbi__vertically_flip_on_load_global 设置为传入的参数值
   stbi__vertically_flip_on_load_global = flag_true_if_should_flip;
}

// 如果未定义 STBI_THREAD_LOCAL，则使用全局变量 stbi__vertically_flip_on_load_global
#ifndef STBI_THREAD_LOCAL
#define stbi__vertically_flip_on_load  stbi__vertically_flip_on_load_global
// 如果定义了 STBI_THREAD_LOCAL，则使用本地变量 stbi__vertically_flip_on_load_local 或全局变量 stbi__vertically_flip_on_load_global
#else
// 定义线程本地变量和标志位
static STBI_THREAD_LOCAL int stbi__vertically_flip_on_load_local, stbi__vertically_flip_on_load_set;

// 设置线程本地是否在加载时垂直翻转图像
STBIDEF void stbi_set_flip_vertically_on_load_thread(int flag_true_if_should_flip)
{
   // 将本地变量 stbi__vertically_flip_on_load_local 设置为传入的参数值
   stbi__vertically_flip_on_load_local = flag_true_if_should_flip;
   // 设置标志位为已设置
   stbi__vertically_flip_on_load_set = 1;
}

// 定义宏，根据标志位选择使用本地变量或全局变量
#define stbi__vertically_flip_on_load  (stbi__vertically_flip_on_load_set       \
                                         ? stbi__vertically_flip_on_load_local  \
                                         : stbi__vertically_flip_on_load_global)
#endif // STBI_THREAD_LOCAL

// 加载主函数，解析图像数据
static void *stbi__load_main(stbi__context *s, int *x, int *y, int *comp, int req_comp, stbi__result_info *ri, int bpc)
{
   // 初始化 ri 结构体，将其内存清零
   memset(ri, 0, sizeof(*ri)); // make sure it's initialized if we add new fields
   // 设置每个通道的位数为 8
   ri->bits_per_channel = 8; // default is 8 so most paths don't have to be changed
   // 设置通道顺序为 RGB
   ri->channel_order = STBI_ORDER_RGB; // all current input & output are this, but this is here so we can add BGR order
   // 初始化通道数为 0
   ri->num_channels = 0;

   // 使用特定的头部信息测试不同的图片格式
   // 如果是 PNG 格式，则调用相应的加载函数
   #ifndef STBI_NO_PNG
   if (stbi__png_test(s))  return stbi__png_load(s,x,y,comp,req_comp, ri);
   #endif
   // 如果是 BMP 格式，则调用相应的加载函数
   #ifndef STBI_NO_BMP
   if (stbi__bmp_test(s))  return stbi__bmp_load(s,x,y,comp,req_comp, ri);
   #endif
   // 如果是 GIF 格式，则调用相应的加载函数
   #ifndef STBI_NO_GIF
   if (stbi__gif_test(s))  return stbi__gif_load(s,x,y,comp,req_comp, ri);
   #endif
   // 如果是 PSD 格式，则调用相应的加载函数
   #ifndef STBI_NO_PSD
   if (stbi__psd_test(s))  return stbi__psd_load(s,x,y,comp,req_comp, ri, bpc);
   // 如果不支持 PSD 格式，则忽略 bpc 参数
   #else
   STBI_NOTUSED(bpc);
   #endif
   // 如果是 PIC 格式，则调用相应的加载函数
   #ifndef STBI_NO_PIC
   if (stbi__pic_test(s))  return stbi__pic_load(s,x,y,comp,req_comp, ri);
   #endif

   // 尝试加载可能只有 1 或 2 个字节匹配预期的格式
   // 这些格式容易出现误判，所以放在后面尝试
   // 如果是 JPEG 格式，则调用相应的加载函数
   #ifndef STBI_NO_JPEG
   if (stbi__jpeg_test(s)) return stbi__jpeg_load(s,x,y,comp,req_comp, ri);
   #endif
   // 如果是 PNM 格式，则调用相应的加载函数
   #ifndef STBI_NO_PNM
   if (stbi__pnm_test(s))  return stbi__pnm_load(s,x,y,comp,req_comp, ri);
   #endif

   // 如果是 HDR 格式，则调用相应的加载函数
   #ifndef STBI_NO_HDR
   if (stbi__hdr_test(s)) {
      float *hdr = stbi__hdr_load(s, x,y,comp,req_comp, ri);
      return stbi__hdr_to_ldr(hdr, *x, *y, req_comp ? req_comp : *comp);
   }
   #endif

   // 如果是 TGA 格式，则调用相应的加载函数
   #ifndef STBI_NO_TGA
   // test tga last because it's a crappy test!
   if (stbi__tga_test(s))
      return stbi__tga_load(s,x,y,comp,req_comp, ri);
   #endif

   // 如果不是任何已知类型的图片，则返回错误信息
   return stbi__errpuc("unknown image type", "Image not of any known type, or corrupt");
}

// 将 16 位像素数据转换为 8 位像素数据
static stbi_uc *stbi__convert_16_to_8(stbi__uint16 *orig, int w, int h, int channels)
{
   // 定义变量 i，用于循环计数
   int i;
   // 计算图像数据长度，宽度乘以高度乘以通道数
   int img_len = w * h * channels;
   // 定义指向缩小后图像数据的指针
   stbi_uc *reduced;

   // 分配内存给缩小后的图像数据
   reduced = (stbi_uc *) stbi__malloc(img_len);
   // 如果内存分配失败，则返回错误信息
   if (reduced == NULL) return stbi__errpuc("outofmem", "Out of memory");

   // 将原始图像数据每个像素的高位取出，缩小为8位，存入缩小后的图像数据中
   for (i = 0; i < img_len; ++i)
      reduced[i] = (stbi_uc)((orig[i] >> 8) & 0xFF); // top half of each byte is sufficient approx of 16->8 bit scaling

   // 释放原始图像数据内存
   STBI_FREE(orig);
   // 返回缩小后的图像数据
   return reduced;
}

// 将8位图像数据转换为16位图像数据
static stbi__uint16 *stbi__convert_8_to_16(stbi_uc *orig, int w, int h, int channels)
{
   // 定义变量 i，用于循环计数
   int i;
   // 计算图像数据长度，宽度乘以高度乘以通道数
   int img_len = w * h * channels;
   // 定义指向扩大后图像数据的指针
   stbi__uint16 *enlarged;

   // 分配内存给扩大后的图像数据
   enlarged = (stbi__uint16 *) stbi__malloc(img_len*2);
   // 如果内存分配失败，则返回错误信息
   if (enlarged == NULL) return (stbi__uint16 *) stbi__errpuc("outofmem", "Out of memory");

   // 将原始图像数据每个像素的低位和高位复制到16位数据中
   for (i = 0; i < img_len; ++i)
      enlarged[i] = (stbi__uint16)((orig[i] << 8) + orig[i]); // replicate to high and low byte, maps 0->0, 255->0xffff

   // 释放原始图像数据内存
   STBI_FREE(orig);
   // 返回扩大后的图像数据
   return enlarged;
}

// 垂直翻转图像数据
static void stbi__vertical_flip(void *image, int w, int h, int bytes_per_pixel)
{
   // 定义变量 row，用于循环计数
   int row;
   // 计算每行字节数
   size_t bytes_per_row = (size_t)w * bytes_per_pixel;
   // 定义临时存储数组
   stbi_uc temp[2048];
   // 将图像数据转换为字节类型指针
   stbi_uc *bytes = (stbi_uc *)image;

   // 逐行进行垂直翻转
   for (row = 0; row < (h>>1); row++) {
      // 指向当前行和对称行的指针
      stbi_uc *row0 = bytes + row*bytes_per_row;
      stbi_uc *row1 = bytes + (h - row - 1)*bytes_per_row;
      // 交换当前行和对称行的数据
      size_t bytes_left = bytes_per_row;
      while (bytes_left) {
         size_t bytes_copy = (bytes_left < sizeof(temp)) ? bytes_left : sizeof(temp);
         memcpy(temp, row0, bytes_copy);
         memcpy(row0, row1, bytes_copy);
         memcpy(row1, temp, bytes_copy);
         row0 += bytes_copy;
         row1 += bytes_copy;
         bytes_left -= bytes_copy;
      }
   }
}

// 垂直翻转图像数据的切片
#ifndef STBI_NO_GIF
static void stbi__vertical_flip_slices(void *image, int w, int h, int z, int bytes_per_pixel)
{
   // 定义变量slice，用于循环迭代
   int slice;
   // 计算每个切片的大小
   int slice_size = w * h * bytes_per_pixel;

   // 将image强制转换为unsigned char类型的指针bytes
   stbi_uc *bytes = (stbi_uc *)image;
   // 遍历所有切片
   for (slice = 0; slice < z; ++slice) {
      // 垂直翻转当前切片的像素数据
      stbi__vertical_flip(bytes, w, h, bytes_per_pixel);
      // 移动指针到下一个切片的起始位置
      bytes += slice_size;
   }
}
#endif

// 加载并处理8位图像数据
static unsigned char *stbi__load_and_postprocess_8bit(stbi__context *s, int *x, int *y, int *comp, int req_comp)
{
   // 存储加载结果的信息
   stbi__result_info ri;
   // 调用stbi__load_main函数加载图像数据
   void *result = stbi__load_main(s, x, y, comp, req_comp, &ri, 8);

   // 如果加载失败，返回空指针
   if (result == NULL)
      return NULL;

   // 断言每个通道的位数为8或16
   STBI_ASSERT(ri.bits_per_channel == 8 || ri.bits_per_channel == 16);

   // 如果每个通道的位数不是8位，则将16位数据转换为8位数据
   if (ri.bits_per_channel != 8) {
      result = stbi__convert_16_to_8((stbi__uint16 *) result, *x, *y, req_comp == 0 ? *comp : req_comp);
      ri.bits_per_channel = 8;
   }

   // @TODO: move stbi__convert_format to here

   // 如果需要在加载时垂直翻转图像数据
   if (stbi__vertically_flip_on_load) {
      // 计算通道数
      int channels = req_comp ? req_comp : *comp;
      // 垂直翻转图像数据
      stbi__vertical_flip(result, *x, *y, channels * sizeof(stbi_uc));
   }

   // 返回处理后的图像数据
   return (unsigned char *) result;
}

// 加载并处理16位图像数据
static stbi__uint16 *stbi__load_and_postprocess_16bit(stbi__context *s, int *x, int *y, int *comp, int req_comp)
{
   // 定义 stbi__result_info 结构体变量 ri
   stbi__result_info ri;
   // 调用 stbi__load_main 函数加载图片数据，并返回结果
   void *result = stbi__load_main(s, x, y, comp, req_comp, &ri, 16);

   // 如果加载结果为空，则返回空指针
   if (result == NULL)
      return NULL;

   // 断言每个通道的位数为 8 或 16
   STBI_ASSERT(ri.bits_per_channel == 8 || ri.bits_per_channel == 16);

   // 如果每个通道的位数不是 16，则将结果转换为 16 位
   if (ri.bits_per_channel != 16) {
      result = stbi__convert_8_to_16((stbi_uc *) result, *x, *y, req_comp == 0 ? *comp : req_comp);
      ri.bits_per_channel = 16;
   }

   // 如果需要在加载时垂直翻转图片，则进行翻转操作
   if (stbi__vertically_flip_on_load) {
      int channels = req_comp ? req_comp : *comp;
      stbi__vertical_flip(result, *x, *y, channels * sizeof(stbi__uint16));
   }

   // 返回转换后的结果
   return (stbi__uint16 *) result;
}

// 如果未定义 STBI_NO_HDR 和 STBI_NO_LINEAR，则定义 stbi__float_postprocess 函数
#if !defined(STBI_NO_HDR) && !defined(STBI_NO_LINEAR)
static void stbi__float_postprocess(float *result, int *x, int *y, int *comp, int req_comp)
{
   // 如果需要在加载时垂直翻转图片，并且结果不为空，则进行翻转操作
   if (stbi__vertically_flip_on_load && result != NULL) {
      int channels = req_comp ? req_comp : *comp;
      stbi__vertical_flip(result, *x, *y, channels * sizeof(float));
   }
}
#endif

// 如果未定义 STBI_NO_STDIO，则定义 stbi__fopen 函数
#ifndef STBI_NO_STDIO

// 如果在 Windows 系统下，并且定义了 STBI_WINDOWS_UTF8，则定义 stbi_convert_wchar_to_utf8 函数
#if defined(_WIN32) && defined(STBI_WINDOWS_UTF8)
STBI_EXTERN __declspec(dllimport) int __stdcall MultiByteToWideChar(unsigned int cp, unsigned long flags, const char *str, int cbmb, wchar_t *widestr, int cchwide);
STBI_EXTERN __declspec(dllimport) int __stdcall WideCharToMultiByte(unsigned int cp, unsigned long flags, const wchar_t *widestr, int cchwide, char *str, int cbmb, const char *defchar, int *used_default);
#endif

// 如果在 Windows 系统下，并且定义了 STBI_WINDOWS_UTF8，则定义 stbi_convert_wchar_to_utf8 函数
#if defined(_WIN32) && defined(STBI_WINDOWS_UTF8)
STBIDEF int stbi_convert_wchar_to_utf8(char *buffer, size_t bufferlen, const wchar_t* input)
{
    return WideCharToMultiByte(65001 /* UTF8 */, 0, input, -1, buffer, (int) bufferlen, NULL, NULL);
}
#endif

// 定义 stbi__fopen 函数，用于打开文件
static FILE *stbi__fopen(char const *filename, char const *mode)
{
   FILE *f;
#if defined(_WIN32) && defined(STBI_WINDOWS_UTF8)
   // 定义宽字符模式和宽字符文件名数组
   wchar_t wMode[64];
   wchar_t wFilename[1024];
    // 将 UTF-8 编码的文件名转换为宽字符编码
    if (0 == MultiByteToWideChar(65001 /* UTF8 */, 0, filename, -1, wFilename, sizeof(wFilename)/sizeof(*wFilename)))
      return 0;

    // 将 UTF-8 编码的模式转换为宽字符编码
    if (0 == MultiByteToWideChar(65001 /* UTF8 */, 0, mode, -1, wMode, sizeof(wMode)/sizeof(*wMode)))
      return 0;

#if defined(_MSC_VER) && _MSC_VER >= 1400
    // 使用 _wfopen_s 函数打开文件
    if (0 != _wfopen_s(&f, wFilename, wMode))
        f = 0;
#else
   // 使用 _wfopen 函数打开文件
   f = _wfopen(wFilename, wMode);
#endif

#elif defined(_MSC_VER) && _MSC_VER >= 1400
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


STBIDEF stbi_uc *stbi_load(char const *filename, int *x, int *y, int *comp, int req_comp)
{
   // 打开文件
   FILE *f = stbi__fopen(filename, "rb");
   unsigned char *result;
   // 如果文件打开失败，返回错误信息
   if (!f) return stbi__errpuc("can't fopen", "Unable to open file");
   // 从文件加载图像数据
   result = stbi_load_from_file(f,x,y,comp,req_comp);
   // 关闭文件
   fclose(f);
   // 返回图像数据
   return result;
}

STBIDEF stbi_uc *stbi_load_from_file(FILE *f, int *x, int *y, int *comp, int req_comp)
{
   unsigned char *result;
   stbi__context s;
   // 初始化文件上下文
   stbi__start_file(&s,f);
   // 加载并处理 8 位图像数据
   result = stbi__load_and_postprocess_8bit(&s,x,y,comp,req_comp);
   if (result) {
      // 需要将 IO 缓冲区中的所有字符“unget”
      fseek(f, - (int) (s.img_buffer_end - s.img_buffer), SEEK_CUR);
   }
   // 返回图像数据
   return result;
}

STBIDEF stbi__uint16 *stbi_load_from_file_16(FILE *f, int *x, int *y, int *comp, int req_comp)
{
   stbi__uint16 *result;
   stbi__context s;
   // 初始化文件上下文
   stbi__start_file(&s,f);
   // 加载并处理 16 位图像数据
   result = stbi__load_and_postprocess_16bit(&s,x,y,comp,req_comp);
   if (result) {
      // 需要将 IO 缓冲区中的所有字符“unget”
      fseek(f, - (int) (s.img_buffer_end - s.img_buffer), SEEK_CUR);
   }
   // 返回图像数据
   return result;
}

STBIDEF stbi_us *stbi_load_16(char const *filename, int *x, int *y, int *comp, int req_comp)
{
   // 打开文件以供读取
   FILE *f = stbi__fopen(filename, "rb");
   // 如果文件打开失败，则返回错误信息
   if (!f) return (stbi_us *) stbi__errpuc("can't fopen", "Unable to open file");
   // 从文件中加载数据到16位无符号整数指针
   stbi__uint16 *result = stbi_load_from_file_16(f,x,y,comp,req_comp);
   // 关闭文件
   fclose(f);
   // 返回加载的数据
   return result;
}


#endif //!STBI_NO_STDIO

// 从内存中加载16位数据
STBIDEF stbi_us *stbi_load_16_from_memory(stbi_uc const *buffer, int len, int *x, int *y, int *channels_in_file, int desired_channels)
{
   // 创建上下文对象并从内存开始
   stbi__context s;
   stbi__start_mem(&s,buffer,len);
   // 加载并后处理16位数据
   return stbi__load_and_postprocess_16bit(&s,x,y,channels_in_file,desired_channels);
}

// 从回调函数中加载16位数据
STBIDEF stbi_us *stbi_load_16_from_callbacks(stbi_io_callbacks const *clbk, void *user, int *x, int *y, int *channels_in_file, int desired_channels)
{
   // 创建上下文对象并从回调函数开始
   stbi__context s;
   stbi__start_callbacks(&s, (stbi_io_callbacks *)clbk, user);
   // 加载并后处理16位数据
   return stbi__load_and_postprocess_16bit(&s,x,y,channels_in_file,desired_channels);
}

// 从内存中加载8位数据
STBIDEF stbi_uc *stbi_load_from_memory(stbi_uc const *buffer, int len, int *x, int *y, int *comp, int req_comp)
{
   // 创建上下文对象并从内存开始
   stbi__context s;
   stbi__start_mem(&s,buffer,len);
   // 加载并后处理8位数据
   return stbi__load_and_postprocess_8bit(&s,x,y,comp,req_comp);
}

// 从回调函数中加载8位数据
STBIDEF stbi_uc *stbi_load_from_callbacks(stbi_io_callbacks const *clbk, void *user, int *x, int *y, int *comp, int req_comp)
{
   // 创建上下文对象并从回调函数开始
   stbi__context s;
   stbi__start_callbacks(&s, (stbi_io_callbacks *) clbk, user);
   // 加载并后处理8位数据
   return stbi__load_and_postprocess_8bit(&s,x,y,comp,req_comp);
}

#ifndef STBI_NO_GIF
// 从内存中加载GIF数据
STBIDEF stbi_uc *stbi_load_gif_from_memory(stbi_uc const *buffer, int len, int **delays, int *x, int *y, int *z, int *comp, int req_comp)
{
   unsigned char *result;
   // 创建上下文对象并从内存开始
   stbi__context s;
   stbi__start_mem(&s,buffer,len);

   // 加载GIF主要数据
   result = (unsigned char*) stbi__load_gif_main(&s, delays, x, y, z, comp, req_comp);
   // 如果需要垂直翻转加载的数据
   if (stbi__vertically_flip_on_load) {
      stbi__vertical_flip_slices( result, *x, *y, *z, *comp );
   }

   // 返回加载的数据
   return result;
}
#endif

#ifndef STBI_NO_LINEAR
// 加载浮点数据的主要函数
static float *stbi__loadf_main(stbi__context *s, int *x, int *y, int *comp, int req_comp)
{
   // 定义一个指向无符号字符的指针变量 data
   unsigned char *data;
   // 如果未定义 STBI_NO_HDR 宏，则执行以下代码块
   #ifndef STBI_NO_HDR
   // 检测是否为 HDR 格式的图片
   if (stbi__hdr_test(s)) {
      // 如果是 HDR 格式的图片，加载并处理 HDR 数据
      stbi__result_info ri;
      float *hdr_data = stbi__hdr_load(s,x,y,comp,req_comp, &ri);
      // 对加载的 HDR 数据进行后处理
      if (hdr_data)
         stbi__float_postprocess(hdr_data,x,y,comp,req_comp);
      // 返回处理后的 HDR 数据
      return hdr_data;
   }
   #endif
   // 加载并处理 8 位图片数据
   data = stbi__load_and_postprocess_8bit(s, x, y, comp, req_comp);
   // 如果成功加载数据，则将其转换为 HDR 数据
   if (data)
      return stbi__ldr_to_hdr(data, *x, *y, req_comp ? req_comp : *comp);
   // 返回错误信息，表示图片类型未知或损坏
   return stbi__errpf("unknown image type", "Image not of any known type, or corrupt");
}

// 从内存中加载 HDR 图片数据
STBIDEF float *stbi_loadf_from_memory(stbi_uc const *buffer, int len, int *x, int *y, int *comp, int req_comp)
{
   // 创建 stbi__context 结构体对象 s，并初始化为内存中的数据
   stbi__context s;
   stbi__start_mem(&s,buffer,len);
   // 调用 stbi__loadf_main 函数加载 HDR 图片数据
   return stbi__loadf_main(&s,x,y,comp,req_comp);
}

// 从回调函数中加载 HDR 图片数据
STBIDEF float *stbi_loadf_from_callbacks(stbi_io_callbacks const *clbk, void *user, int *x, int *y, int *comp, int req_comp)
{
   // 创建 stbi__context 结构体对象 s，并初始化为回调函数中的数据
   stbi__context s;
   stbi__start_callbacks(&s, (stbi_io_callbacks *) clbk, user);
   // 调用 stbi__loadf_main 函数加载 HDR 图片数据
   return stbi__loadf_main(&s,x,y,comp,req_comp);
}

// 从文件中加载 HDR 图片数据
#ifndef STBI_NO_STDIO
STBIDEF float *stbi_loadf(char const *filename, int *x, int *y, int *comp, int req_comp)
{
   // 定义一个指向 float 的指针变量 result
   float *result;
   // 打开文件并获取文件指针
   FILE *f = stbi__fopen(filename, "rb");
   // 如果文件指针为空，返回错误信息
   if (!f) return stbi__errpf("can't fopen", "Unable to open file");
   // 从文件中加载 HDR 图片数据
   result = stbi_loadf_from_file(f,x,y,comp,req_comp);
   // 关闭文件
   fclose(f);
   // 返回加载的 HDR 图片数据
   return result;
}

// 从文件指针中加载 HDR 图片数据
STBIDEF float *stbi_loadf_from_file(FILE *f, int *x, int *y, int *comp, int req_comp)
{
   // 创建 stbi__context 结构体对象 s，并初始化为文件指针中的数据
   stbi__context s;
   stbi__start_file(&s,f);
   // 调用 stbi__loadf_main 函数加载 HDR 图片数据
   return stbi__loadf_main(&s,x,y,comp,req_comp);
}
#endif // !STBI_NO_STDIO

#endif // !STBI_NO_LINEAR

// 这些 is-hdr-or-not 是独立于 STBI_NO_LINEAR 是否定义的，为了 API 的简单性；如果定义了 STBI_NO_LINEAR，则始终返回 false！

// 从内存中判断是否为 HDR 图片
STBIDEF int stbi_is_hdr_from_memory(stbi_uc const *buffer, int len)
{
   #ifndef STBI_NO_HDR
   // 如果未定义 STBI_NO_HDR，则执行以下代码块
   stbi__context s;
   // 创建 stbi__context 结构体 s
   stbi__start_mem(&s,buffer,len);
   // 从内存中读取数据到 s
   return stbi__hdr_test(&s);
   // 调用 stbi__hdr_test 函数，返回测试结果
   #else
   // 如果定义了 STBI_NO_HDR，则执行以下代码块
   STBI_NOTUSED(buffer);
   // 使用 STBI_NOTUSED 宏避免编译器警告
   STBI_NOTUSED(len);
   // 使用 STBI_NOTUSED 宏避免编译器警告
   return 0;
   // 返回 0
   #endif
}

#ifndef STBI_NO_STDIO
// 如果未定义 STBI_NO_STDIO，则执行以下代码块
STBIDEF int      stbi_is_hdr          (char const *filename)
{
   // 定义文件指针 f，打开文件 filename 以二进制只读方式
   FILE *f = stbi__fopen(filename, "rb");
   // 初始化结果为 0
   int result=0;
   // 如果文件指针有效
   if (f) {
      // 调用 stbi_is_hdr_from_file 函数，将结果赋给 result
      result = stbi_is_hdr_from_file(f);
      // 关闭文件
      fclose(f);
   }
   // 返回结果
   return result;
}

STBIDEF int stbi_is_hdr_from_file(FILE *f)
{
   #ifndef STBI_NO_HDR
   // 如果未定义 STBI_NO_HDR，则执行以下代码块
   // 获取当前文件指针位置
   long pos = ftell(f);
   // 定义结果变量
   int res;
   // 创建 stbi__context 结构体 s
   stbi__context s;
   // 从文件中读取数据到 s
   stbi__start_file(&s,f);
   // 调用 stbi__hdr_test 函数，将结果赋给 res
   res = stbi__hdr_test(&s);
   // 将文件指针移动到之前保存的位置
   fseek(f, pos, SEEK_SET);
   // 返回结果
   return res;
   #else
   // 如果定义了 STBI_NO_HDR，则执行以下代码块
   STBI_NOTUSED(f);
   // 使用 STBI_NOTUSED 宏避免编译器警告
   return 0;
   // 返回 0
   #endif
}
#endif // !STBI_NO_STDIO

STBIDEF int      stbi_is_hdr_from_callbacks(stbi_io_callbacks const *clbk, void *user)
{
   #ifndef STBI_NO_HDR
   // 如果未定义 STBI_NO_HDR，则执行以下代码块
   // 创建 stbi__context 结构体 s
   stbi__context s;
   // 从回调函数中读取数据到 s
   stbi__start_callbacks(&s, (stbi_io_callbacks *) clbk, user);
   // 调用 stbi__hdr_test 函数，返回测试结果
   return stbi__hdr_test(&s);
   #else
   // 如果定义了 STBI_NO_HDR，则执行以下代码块
   STBI_NOTUSED(clbk);
   // 使用 STBI_NOTUSED 宏避免编译器警告
   STBI_NOTUSED(user);
   // 使用 STBI_NOTUSED 宏避免编译器警告
   return 0;
   // 返回 0
   #endif
}

#ifndef STBI_NO_LINEAR
// 如果未定义 STBI_NO_LINEAR，则执行以下代码块
static float stbi__l2h_gamma=2.2f, stbi__l2h_scale=1.0f;

STBIDEF void   stbi_ldr_to_hdr_gamma(float gamma) { stbi__l2h_gamma = gamma; }
// 设置 stbi__l2h_gamma 的值为 gamma
STBIDEF void   stbi_ldr_to_hdr_scale(float scale) { stbi__l2h_scale = scale; }
// 设置 stbi__l2h_scale 的值为 scale
#endif

static float stbi__h2l_gamma_i=1.0f/2.2f, stbi__h2l_scale_i=1.0f;

STBIDEF void   stbi_hdr_to_ldr_gamma(float gamma) { stbi__h2l_gamma_i = 1/gamma; }
// 设置 stbi__h2l_gamma_i 的值为 1/gamma
STBIDEF void   stbi_hdr_to_ldr_scale(float scale) { stbi__h2l_scale_i = 1/scale; }
// 设置 stbi__h2l_scale_i 的值为 1/scale


//////////////////////////////////////////////////////////////////////////////
//
// Common code used by all image loaders
//

enum
{
   STBI__SCAN_load=0,
   STBI__SCAN_type,
   STBI__SCAN_header
};

static void stbi__refill_buffer(stbi__context *s)
// 重新填充缓冲区的函数
{
   // 从输入流中读取数据到缓冲区，返回读取的字节数
   int n = (s->io.read)(s->io_user_data,(char*)s->buffer_start,s->buflen);
   // 更新已读取的回调数据量
   s->callback_already_read += (int) (s->img_buffer - s->img_buffer_original);
   // 如果读取的字节数为0，表示已到文件末尾
   if (n == 0) {
      // 处理文件末尾的情况，将 img_buffer 指向 buffer_start，并设置 img_buffer_end
      s->read_from_callbacks = 0;
      s->img_buffer = s->buffer_start;
      s->img_buffer_end = s->buffer_start+1;
      *s->img_buffer = 0;
   } else {
      // 更新 img_buffer 和 img_buffer_end
      s->img_buffer = s->buffer_start;
      s->img_buffer_end = s->buffer_start + n;
   }
}

stbi_inline static stbi_uc stbi__get8(stbi__context *s)
{
   // 如果 img_buffer 指针小于 img_buffer_end 指针，返回当前指针位置的值并移动指针
   if (s->img_buffer < s->img_buffer_end)
      return *s->img_buffer++;
   // 如果 read_from_callbacks 为真，重新填充缓冲区并返回当前指针位置的值
   if (s->read_from_callbacks) {
      stbi__refill_buffer(s);
      return *s->img_buffer++;
   }
   // 否则返回0
   return 0;
}

#if defined(STBI_NO_JPEG) && defined(STBI_NO_HDR) && defined(STBI_NO_PIC) && defined(STBI_NO_PNM)
// 什么也不做
#else
// 检查是否已到达文件末尾
stbi_inline static int stbi__at_eof(stbi__context *s)
{
   if (s->io.read) {
      // 如果未到达文件末尾，返回0
      if (!(s->io.eof)(s->io_user_data)) return 0;
      // 如果 feof() 为真，检查缓冲区是否等于结束指针
      // 特殊情况：只有特殊的0字符在末尾
      if (s->read_from_callbacks == 0) return 1;
   }

   // 返回 img_buffer 是否大于等于 img_buffer_end
   return s->img_buffer >= s->img_buffer_end;
}
#endif

#if defined(STBI_NO_JPEG) && defined(STBI_NO_PNG) && defined(STBI_NO_BMP) && defined(STBI_NO_PSD) && defined(STBI_NO_TGA) && defined(STBI_NO_GIF) && defined(STBI_NO_PIC)
// 什么也不做
#else
// 跳过指定数量的字节
static void stbi__skip(stbi__context *s, int n)
{
   // 如果 n为0，直接返回
   if (n == 0) return;  // already there!
   // 如果 n为负数，将 img_buffer 指向 img_buffer_end
   if (n < 0) {
      s->img_buffer = s->img_buffer_end;
      return;
   }
   // 如果有读取函数，检查剩余可读字节数是否小于n，若是则跳过
   if (s->io.read) {
      int blen = (int) (s->img_buffer_end - s->img_buffer);
      if (blen < n) {
         s->img_buffer = s->img_buffer_end;
         (s->io.skip)(s->io_user_data, n - blen);
         return;
      }
   }
   // 否则直接移动 img_buffer 指针
   s->img_buffer += n;
}
#endif
#if defined(STBI_NO_PNG) && defined(STBI_NO_TGA) && defined(STBI_NO_HDR) && defined(STBI_NO_PNM)
// 如果定义了 STBI_NO_PNG、STBI_NO_TGA、STBI_NO_HDR 和 STBI_NO_PNM，则什么都不做
// 否则，定义一个函数 stbi__getn，用于从 stbi__context 结构体中读取 n 个字节的数据
#else
static int stbi__getn(stbi__context *s, stbi_uc *buffer, int n)
{
   // 如果 stbi__context 结构体中有读取函数
   if (s->io.read) {
      // 计算当前图像缓冲区中剩余的字节数
      int blen = (int) (s->img_buffer_end - s->img_buffer);
      // 如果剩余字节数小于 n
      if (blen < n) {
         int res, count;

         // 将当前图像缓冲区中的数据拷贝到 buffer 中
         memcpy(buffer, s->img_buffer, blen);

         // 调用读取函数，将剩余数据读取到 buffer 中
         count = (s->io.read)(s->io_user_data, (char*) buffer + blen, n - blen);
         // 判断是否成功读取了 n-blen 个字节
         res = (count == (n-blen));
         // 更新图像缓冲区指针
         s->img_buffer = s->img_buffer_end;
         return res;
      }
   }

   // 如果剩余数据足够，直接从图像缓冲区中读取 n 个字节到 buffer 中
   if (s->img_buffer+n <= s->img_buffer_end) {
      memcpy(buffer, s->img_buffer, n);
      s->img_buffer += n;
      return 1;
   } else
      return 0;
}
#endif

#if defined(STBI_NO_JPEG) && defined(STBI_NO_PNG) && defined(STBI_NO_PSD) && defined(STBI_NO_PIC)
// 如果定义了 STBI_NO_JPEG、STBI_NO_PNG、STBI_NO_PSD 和 STBI_NO_PIC，则什么都不做
// 否则，定义一个函数 stbi__get16be，用于从 stbi__context 结构体中读取一个 16 位大端序的整数
#else
static int stbi__get16be(stbi__context *s)
{
   // 读取一个字节
   int z = stbi__get8(s);
   // 将该字节左移 8 位，再读取一个字节，将两个字节合并成一个 16 位大端序的整数
   return (z << 8) + stbi__get8(s);
}
#endif

#if defined(STBI_NO_PNG) && defined(STBI_NO_PSD) && defined(STBI_NO_PIC)
// 如果定义了 STBI_NO_PNG、STBI_NO_PSD 和 STBI_NO_PIC，则什么都不做
// 否则，定义一个函数 stbi__get32be，用于从 stbi__context 结构体中读取一个 32 位大端序的整数
#else
static stbi__uint32 stbi__get32be(stbi__context *s)
{
   // 读取一个 16 位大端序的整数
   stbi__uint32 z = stbi__get16be(s);
   // 将该整数左移 16 位，再读取一个 16 位大端序的整数，将两个整数合并成一个 32 位大端序的整数
   return (z << 16) + stbi__get16be(s);
}
#endif

#if defined(STBI_NO_BMP) && defined(STBI_NO_TGA) && defined(STBI_NO_GIF)
// 如果定义了 STBI_NO_BMP、STBI_NO_TGA 和 STBI_NO_GIF，则什么都不做
// 否则，定义一个函数 stbi__get16le，用于从 stbi__context 结构体中读取一个 16 位小端序的整数
#else
static int stbi__get16le(stbi__context *s)
{
   // 读取一个字节
   int z = stbi__get8(s);
   // 读取另一个字节，将两个字节合并成一个 16 位小端序的整数
   return z + (stbi__get8(s) << 8);
}
#endif

#ifndef STBI_NO_BMP
static stbi__uint32 stbi__get32le(stbi__context *s)
{
   // 读取一个 16 位小端序的整数
   stbi__uint32 z = stbi__get16le(s);
   // 将该整数左移 16 位，再读取另一个 16 位小端序的整数，将两个整数合并成一个 32 位小端序的整数
   z += (stbi__uint32)stbi__get16le(s) << 16;
   return z;
}
#endif

#define STBI__BYTECAST(x)  ((stbi_uc) ((x) & 255))  // 将整数 x 截断为字节，避免警告

#if defined(STBI_NO_JPEG) && defined(STBI_NO_PNG) && defined(STBI_NO_BMP) && defined(STBI_NO_PSD) && defined(STBI_NO_TGA) && defined(STBI_NO_GIF) && defined(STBI_NO_PIC) && defined(STBI_NO_PNM)
// 如果定义了 STBI_NO_JPEG、STBI_NO_PNG、STBI_NO_BMP、STBI_NO_PSD、STBI_NO_TGA、STBI_NO_GIF、STBI_NO_PIC 和 STBI_NO_PNM，则什么都不做
// 否则，继续下面的代码
#else
//////////////////////////////////////////////////////////////////////////////
//
//  通用转换器，从内置的 img_n 转换为 req_comp
//    各个类型尽可能自动完成此操作（例如，jpeg
//    在内部执行所有情况，因为它需要进行颜色空间转换，而且它从不具有 alpha 通道，所以很少出现这种情况）。png 可以自动
//    交错一个 alpha=255 通道，但对于其他情况则退回到这里
//
//  假设数据缓冲区是通过 malloc 分配的，因此 malloc 一个新的并释放原来的
//  唯一的失败模式是 malloc 失败

static stbi_uc stbi__compute_y(int r, int g, int b)
{
   return (stbi_uc) (((r*77) + (g*150) +  (29*b)) >> 8);
}
#endif

#if defined(STBI_NO_PNG) && defined(STBI_NO_BMP) && defined(STBI_NO_PSD) && defined(STBI_NO_TGA) && defined(STBI_NO_GIF) && defined(STBI_NO_PIC) && defined(STBI_NO_PNM)
// 什么也不做
#else
static unsigned char *stbi__convert_format(unsigned char *data, int img_n, int req_comp, unsigned int x, unsigned int y)
}
#endif

#if defined(STBI_NO_PNG) && defined(STBI_NO_PSD)
// 什么也不做
#else
static stbi__uint16 stbi__compute_y_16(int r, int g, int b)
{
   return (stbi__uint16) (((r*77) + (g*150) +  (29*b)) >> 8);
}
#endif

#if defined(STBI_NO_PNG) && defined(STBI_NO_PSD)
// 什么也不做
#else
static stbi__uint16 *stbi__convert_format16(stbi__uint16 *data, int img_n, int req_comp, unsigned int x, unsigned int y)
}
#endif

#ifndef STBI_NO_LINEAR
static float   *stbi__ldr_to_hdr(stbi_uc *data, int x, int y, int comp)
{
   // 声明整型变量 i, k, n
   int i,k,n;
   // 声明浮点型指针变量 output
   float *output;
   // 如果 data 为空，则返回空指针
   if (!data) return NULL;
   // 分配内存给 output，大小为 x*y*comp*sizeof(float)
   output = (float *) stbi__malloc_mad4(x, y, comp, sizeof(float), 0);
   // 如果 output 为空，则释放 data 并返回错误信息
   if (output == NULL) { STBI_FREE(data); return stbi__errpf("outofmem", "Out of memory"); }
   // 计算非 alpha 通道的数量
   if (comp & 1) n = comp; else n = comp-1;
   // 遍历像素点
   for (i=0; i < x*y; ++i) {
      for (k=0; k < n; ++k) {
         // 对每个像素点进行颜色转换
         output[i*comp + k] = (float) (pow(data[i*comp+k]/255.0f, stbi__l2h_gamma) * stbi__l2h_scale);
      }
   }
   // 如果非 alpha 通道数量小于 comp，则继续处理
   if (n < comp) {
      for (i=0; i < x*y; ++i) {
         // 对每个像素点的 alpha 通道进行颜色转换
         output[i*comp + n] = data[i*comp + n]/255.0f;
      }
   }
   // 释放 data 内存
   STBI_FREE(data);
   // 返回处理后的 output
   return output;
}
#endif

#ifndef STBI_NO_HDR
#define stbi__float2int(x)   ((int) (x))
// 将 HDR 数据转换为 LDR 数据
static stbi_uc *stbi__hdr_to_ldr(float   *data, int x, int y, int comp)
{
   // 声明整型变量 i, k, n
   int i,k,n;
   // 声明无符号字符型指针变量 output
   stbi_uc *output;
   // 如果 data 为空，则返回空指针
   if (!data) return NULL;
   // 分配内存给 output，大小为 x*y*comp
   output = (stbi_uc *) stbi__malloc_mad3(x, y, comp, 0);
   // 如果 output 为空，则释放 data 并返回错误信息
   if (output == NULL) { STBI_FREE(data); return stbi__errpuc("outofmem", "Out of memory"); }
   // 计算非 alpha 通道的数量
   if (comp & 1) n = comp; else n = comp-1;
   // 遍历像素点
   for (i=0; i < x*y; ++i) {
      for (k=0; k < n; ++k) {
         // 对每个像素点进行颜色转换
         float z = (float) pow(data[i*comp+k]*stbi__h2l_scale_i, stbi__h2l_gamma_i) * 255 + 0.5f;
         if (z < 0) z = 0;
         if (z > 255) z = 255;
         output[i*comp + k] = (stbi_uc) stbi__float2int(z);
      }
      if (k < comp) {
         // 对每个像素点的 alpha 通道进行颜色转换
         float z = data[i*comp+k] * 255 + 0.5f;
         if (z < 0) z = 0;
         if (z > 255) z = 255;
         output[i*comp + k] = (stbi_uc) stbi__float2int(z);
      }
   }
   // 释放 data 内存
   STBI_FREE(data);
   // 返回处理后的 output
   return output;
}
#endif

//////////////////////////////////////////////////////////////////////////////
//
//  "baseline" JPEG/JFIF decoder
//
//    simple implementation
//      - doesn't support delayed output of y-dimension
//      - simple interface (only one output format: 8-bit interleaved RGB)
//      - doesn't try to recover corrupt jpegs
// 如果定义了 STBI_NO_JPEG，则不包含 JPEG 解码相关的代码

// 定义快速位数，用于加速哈夫曼解码，较大值处理更多情况，较小值减少缓存占用
#define FAST_BITS   9

// 定义哈夫曼结构体，包含快速查找表、编码、数值、大小、最大编码等信息
typedef struct
{
   stbi_uc  fast[1 << FAST_BITS];
   stbi__uint16 code[256];
   stbi_uc  values[256];
   stbi_uc  size[257];
   unsigned int maxcode[18];
   int    delta[17];
} stbi__huffman;

// 定义 JPEG 解码器结构体，包含上下文、哈夫曼表、量化表、快速交流表等信息
typedef struct
{
   stbi__context *s;
   stbi__huffman huff_dc[4];
   stbi__huffman huff_ac[4];
   stbi__uint16 dequant[4][64];
   stbi__int16 fast_ac[4][1 << FAST_BITS];

   // 组件大小、交错 MCU 大小等信息
   int img_h_max, img_v_max;
   int img_mcu_x, img_mcu_y;
   int img_mcu_w, img_mcu_h;
// 定义 JPEG 图像组件的结构体
struct
{
    int id; // 组件 ID
    int h,v; // 水平和垂直采样因子
    int tq; // 量化表索引
    int hd,ha; // DC 和 AC Huffman 表索引
    int dc_pred; // DC 预测值

    int x,y,w2,h2; // 组件在图像中的位置和大小
    stbi_uc *data; // 组件数据
    void *raw_data, *raw_coeff; // 原始数据和系数
    stbi_uc *linebuf; // 行缓冲
    short *coeff; // 仅用于渐进扫描
    int coeff_w, coeff_h; // 8x8 系数块的数量
} img_comp[4]; // 图像组件数组，最多包含 4 个组件

stbi__uint32 code_buffer; // JPEG 熵编码缓冲
int code_bits; // 有效位数
unsigned char marker; // 在填充熵缓冲时看到的标记
int nomore; // 如果看到标记，则必须停止的标志

int progressive; // 渐进模式标志
int spec_start; // 起始扫描段
int spec_end; // 结束扫描段
int succ_high; // 高位成功
int succ_low; // 低位成功
int eob_run; // EOB 运行
int jfif; // JFIF 标志
int app14_color_transform; // Adobe APP14 标签
int rgb; // RGB 标志

int scan_n, order[4]; // 扫描数量和顺序数组
int restart_interval, todo; // 重启间隔和待处理数量

// 内核函数指针
void (*idct_block_kernel)(stbi_uc *out, int out_stride, short data[64]); // IDCT 块内核
void (*YCbCr_to_RGB_kernel)(stbi_uc *out, const stbi_uc *y, const stbi_uc *pcb, const stbi_uc *pcr, int count, int step); // YCbCr 转 RGB 内核
stbi_uc *(*resample_row_hv_2_kernel)(stbi_uc *out, stbi_uc *in_near, stbi_uc *in_far, int w, int hs); // 水平和垂直重采样内核
} stbi__jpeg; // JPEG 解码器结构体

// 构建哈夫曼表
static int stbi__build_huffman(stbi__huffman *h, int *count)
{
   // 定义整型变量 i, j, k，并初始化 k 为 0
   int i,j,k=0;
   // 定义无符号整型变量 code
   unsigned int code;
   // 为每个符号构建大小列表（根据 JPEG 规范）
   for (i=0; i < 16; ++i) {
      for (j=0; j < count[i]; ++j) {
         // 将大小值存入 h->size 数组中
         h->size[k++] = (stbi_uc) (i+1);
         // 如果 k 大于等于 257，则返回错误信息
         if(k >= 257) return stbi__err("bad size list","Corrupt JPEG");
      }
   }
   // 将 h->size 数组最后一个元素设为 0
   h->size[k] = 0;

   // 计算实际符号（根据 JPEG 规范）
   code = 0;
   k = 0;
   for(j=1; j <= 16; ++j) {
      // 计算添加到 code 上以计算符号 ID 的增量
      h->delta[j] = k - code;
      if (h->size[k] == j) {
         while (h->size[k] == j)
            // 将符号存入 h->code 数组中
            h->code[k++] = (stbi__uint16) (code++);
         // 如果 code-1 大于等于 2^j，则返回错误信息
         if (code-1 >= (1u << j)) return stbi__err("bad code lengths","Corrupt JPEG");
      }
      // 计算该大小的最大代码 + 1，根据需要预先移位
      h->maxcode[j] = code << (16-j);
      code <<= 1;
   }
   // 将 h->maxcode[j] 设为 0xffffffff

   // 构建非规范加速表；255 用作未加速的标志
   memset(h->fast, 255, 1 << FAST_BITS);
   for (i=0; i < k; ++i) {
      int s = h->size[i];
      if (s <= FAST_BITS) {
         int c = h->code[i] << (FAST_BITS-s);
         int m = 1 << (FAST_BITS-s);
         for (j=0; j < m; ++j) {
            // 将索引存入 h->fast 数组中
            h->fast[c+j] = (stbi_uc) i;
         }
      }
   }
   // 返回 1
   return 1;
}

// 构建一个表，一次性解码小 AC 的幅度和值
static void stbi__build_fast_ac(stbi__int16 *fast_ac, stbi__huffman *h)
}
{
   // 定义整型变量 i
   int i;
   // 循环遍历 0 到 (1 << FAST_BITS) 之间的整数
   for (i=0; i < (1 << FAST_BITS); ++i) {
      // 获取 h->fast[i] 的值
      stbi_uc fast = h->fast[i];
      // 初始化 fast_ac[i] 为 0
      fast_ac[i] = 0;
      // 如果 fast 小于 255
      if (fast < 255) {
         // 获取 h->values[fast] 的值
         int rs = h->values[fast];
         // 获取 rs 的高 4 位作为 run，低 4 位作为 magbits
         int run = (rs >> 4) & 15;
         int magbits = rs & 15;
         // 获取 h->size[fast] 的值作为 len
         int len = h->size[fast];

         // 如果 magbits 不为 0 且 len + magbits 小于等于 FAST_BITS
         if (magbits && len + magbits <= FAST_BITS) {
            // 计算 k
            int k = ((i << len) & ((1 << FAST_BITS) - 1)) >> (FAST_BITS - magbits);
            int m = 1 << (magbits - 1);
            // 如果 k 小于 m，则 k 加上 (~0U << magbits) + 1
            if (k < m) k += (~0U << magbits) + 1;
            // 如果结果在 -128 到 127 之间，则将结果存入 fast_ac[i]
            if (k >= -128 && k <= 127)
               fast_ac[i] = (stbi__int16) ((k * 256) + (run * 16) + (len + magbits));
         }
      }
   }
}

// 扩展缓冲区
static void stbi__grow_buffer_unsafe(stbi__jpeg *j)
{
   do {
      // 读取一个字节到 b
      unsigned int b = j->nomore ? 0 : stbi__get8(j->s);
      // 如果 b 为 0xff
      if (b == 0xff) {
         // 读取下一个字节到 c
         int c = stbi__get8(j->s);
         // 消耗填充字节
         while (c == 0xff) c = stbi__get8(j->s);
         // 如果 c 不为 0，则将 c 存入 marker，并设置 nomore 为 1，然后返回
         if (c != 0) {
            j->marker = (unsigned char) c;
            j->nomore = 1;
            return;
         }
      }
      // 将 b 左移 (24 - j->code_bits) 位后与 code_buffer 进行或运算
      j->code_buffer |= b << (24 - j->code_bits);
      j->code_bits += 8;
   } while (j->code_bits <= 24);
}

// (1 << n) - 1
// 定义常量数组 stbi__bmask，存储 2^n - 1 的值
static const stbi__uint32 stbi__bmask[17]={0,1,3,7,15,31,63,127,255,511,1023,2047,4095,8191,16383,32767,65535};

// 从比特流中解码 JPEG 哈夫曼值
stbi_inline static int stbi__jpeg_huff_decode(stbi__jpeg *j, stbi__huffman *h)
{
   // 定义临时变量和两个整型变量
   unsigned int temp;
   int c,k;

   // 如果当前码流中的比特数小于16，则扩展码流缓冲区
   if (j->code_bits < 16) stbi__grow_buffer_unsafe(j);

   // 通过查看顶部的FAST_BITS确定符号ID，如果码值小于等于FAST_BITS
   c = (j->code_buffer >> (32 - FAST_BITS)) & ((1 << FAST_BITS)-1);
   k = h->fast[c];
   if (k < 255) {
      int s = h->size[k];
      if (s > j->code_bits)
         return -1;
      j->code_buffer <<= s;
      j->code_bits -= s;
      return h->values[k];
   }

   // 简单测试是将码流缓冲区向下移动k位，然后与maxcode进行比较。为了加快速度，我们已经将maxcode向左预移，使其末尾有(16-k)个0；换句话说，无论位数如何，它都希望与移位后有16位的内容进行比较；这样我们就不需要在循环内部进行移位。
   temp = j->code_buffer >> 16;
   for (k=FAST_BITS+1 ; ; ++k)
      if (temp < h->maxcode[k])
         break;
   if (k == 17) {
      // 错误！未找到码值
      j->code_bits -= 16;
      return -1;
   }

   if (k > j->code_bits)
      return -1;

   // 将哈夫曼码转换为符号ID
   c = ((j->code_buffer >> (32 - k)) & stbi__bmask[k]) + h->delta[k];
   if(c < 0 || c >= 256) // 符号ID超出范围！
       return -1;
   STBI_ASSERT((((j->code_buffer) >> (32 - h->size[c])) & stbi__bmask[h->size[c]]) == h->code[c]);

   // 将ID转换为符号
   j->code_bits -= k;
   j->code_buffer <<= k;
   return h->values[c];
}

// bias[n] = (-1<<n) + 1
static const int stbi__jbias[16] = {0,-1,-3,-7,-15,-31,-63,-127,-255,-511,-1023,-2047,-4095,-8191,-16383,-32767};

// 结合了JPEG的“接收”和JPEG的“扩展”，因为基线总是扩展它接收到的所有内容。
stbi_inline static int stbi__extend_receive(stbi__jpeg *j, int n)
}
{
   // 定义无符号整型变量 k
   unsigned int k;
   // 定义整型变量 sgn
   int sgn;
   // 如果当前已读取的比特数小于 n，则扩展缓冲区
   if (j->code_bits < n) stbi__grow_buffer_unsafe(j);
   // 如果当前已读取的比特数小于 n，则返回 0，表示从流中读取的比特不足，避免继续读取
   if (j->code_bits < n) return 0; // ran out of bits from stream, return 0s intead of continuing

   // 获取符号位，始终在最高位；如果最高位清零（正数），则 sgn 为 0，如果最高位设置（负数），则 sgn 为 1
   sgn = j->code_buffer >> 31; 
   // 将当前缓冲区左旋 n 位
   k = stbi_lrot(j->code_buffer, n);
   // 更新缓冲区，保留 n 位之外的数据
   j->code_buffer = k & ~stbi__bmask[n];
   // 获取 n 位数据
   k &= stbi__bmask[n];
   // 更新已读取的比特数
   j->code_bits -= n;
   // 返回解码后的值
   return k + (stbi__jbias[n] & (sgn - 1));
}

// 获取一些无符号比特
stbi_inline static int stbi__jpeg_get_bits(stbi__jpeg *j, int n)
{
   // 定义无符号整型变量 k
   unsigned int k;
   // 如果当前已读取的比特数小于 n，则扩展缓冲区
   if (j->code_bits < n) stbi__grow_buffer_unsafe(j);
   // 如果当前已读取的比特数小于 n，则返回 0，表示从流中读取的比特不足，避免继续读取
   if (j->code_bits < n) return 0; // ran out of bits from stream, return 0s intead of continuing
   // 将当前缓冲区左旋 n 位
   k = stbi_lrot(j->code_buffer, n);
   // 更新缓冲区，保留 n 位之外的数据
   j->code_buffer = k & ~stbi__bmask[n];
   // 获取 n 位数据
   k &= stbi__bmask[n];
   // 更新已读取的比特数
   j->code_bits -= n;
   // 返回解码后的值
   return k;
}

// 获取一个比特
stbi_inline static int stbi__jpeg_get_bit(stbi__jpeg *j)
{
   // 定义无符号整型变量 k
   unsigned int k;
   // 如果当前已读取的比特数小于 1，则扩展缓冲区
   if (j->code_bits < 1) stbi__grow_buffer_unsafe(j);
   // 如果当前已读取的比特数小于 1，则返回 0，表示从流中读取的比特不足，避免继续读取
   if (j->code_bits < 1) return 0; // ran out of bits from stream, return 0s intead of continuing
   // 获取当前缓冲区的最高位
   k = j->code_buffer;
   // 左移缓冲区一位
   j->code_buffer <<= 1;
   // 减少已读取的比特数
   --j->code_bits;
   // 返回最高位的值
   return k & 0x80000000;
}

// 给定在蛇形流中位置为 X 的值，在以行优先方式编码的 8x8 矩阵中的位置
static const stbi_uc stbi__jpeg_dezigzag[64+15] =
{
    // 蛇形流到矩阵的映射关系
    0,  1,  8, 16,  9,  2,  3, 10,
    17, 24, 32, 25, 18, 11,  4,  5,
    12, 19, 26, 33, 40, 48, 41, 34,
    27, 20, 13,  6,  7, 14, 21, 28,
    35, 42, 49, 56, 57, 50, 43, 36,
    29, 22, 15, 23, 30, 37, 44, 51,
    58, 59, 52, 45, 38, 31, 39, 46,
    53, 60, 61, 54, 47, 55, 62, 63,
    // 让损坏的输入样本超出末尾
    63, 63, 63, 63, 63, 63, 63, 63,
    63, 63, 63, 63, 63, 63, 63
};

// 解码一个 64 个条目的块
static int stbi__jpeg_decode_block(stbi__jpeg *j, short data[64], stbi__huffman *hdc, stbi__huffman *hac, stbi__int16 *fac, int b, stbi__uint16 *dequant)
{
   // 声明变量 diff, dc, k, t
   int diff,dc,k;
   int t;

   // 如果当前 Huffman 编码位数小于 16，则扩展缓冲区
   if (j->code_bits < 16) stbi__grow_buffer_unsafe(j);
   // 使用 Huffman 解码器解码 DC 系数
   t = stbi__jpeg_huff_decode(j, hdc);
   // 如果解码结果 t 不在 [0, 15] 范围内，则返回错误信息
   if (t < 0 || t > 15) return stbi__err("bad huffman code","Corrupt JPEG");

   // 将 data 数组的前 64 个元素全部置为 0
   memset(data,0,64*sizeof(data[0]));

   // 计算差值 diff，如果 t 不为 0，则调用 stbi__extend_receive 函数，否则 diff 为 0
   diff = t ? stbi__extend_receive(j, t) : 0;
   // 更新 DC 系数，并检查是否合法
   if (!stbi__addints_valid(j->img_comp[b].dc_pred, diff)) return stbi__err("bad delta","Corrupt JPEG");
   dc = j->img_comp[b].dc_pred + diff;
   j->img_comp[b].dc_pred = dc;
   // 检查 DC 和量化表是否合法，计算 data[0] 的值
   if (!stbi__mul2shorts_valid(dc, dequant[0])) return stbi__err("can't merge dc and ac", "Corrupt JPEG");
   data[0] = (short) (dc * dequant[0]);

   // 解码 AC 分量，遵循 JPEG 规范
   k = 1;
   do {
      unsigned int zig;
      int c,r,s;
      // 如果当前 Huffman 编码位数小于 16，则扩展缓冲区
      if (j->code_bits < 16) stbi__grow_buffer_unsafe(j);
      // 从编码缓冲区中读取 c，根据 fac 表查找对应的值 r
      c = (j->code_buffer >> (32 - FAST_BITS)) & ((1 << FAST_BITS)-1);
      r = fac[c];
      if (r) { // 快速 AC 路径
         // 更新 k 和 s，解码 AC 分量并存储到 data 数组中
         k += (r >> 4) & 15; // run
         s = r & 15; // combined length
         if (s > j->code_bits) return stbi__err("bad huffman code", "Combined length longer than code bits available");
         j->code_buffer <<= s;
         j->code_bits -= s;
         // 解码到未经过 zigzag 变换的位置
         zig = stbi__jpeg_dezigzag[k++];
         data[zig] = (short) ((r >> 8) * dequant[zig]);
      } else {
         // 从 Huffman 表解码 AC 分量
         int rs = stbi__jpeg_huff_decode(j, hac);
         if (rs < 0) return stbi__err("bad huffman code","Corrupt JPEG");
         s = rs & 15;
         r = rs >> 4;
         if (s == 0) {
            if (rs != 0xf0) break; // 结束块
            k += 16;
         } else {
            k += r;
            // 解码到未经过 zigzag 变换的位置
            zig = stbi__jpeg_dezigzag[k++];
            data[zig] = (short) (stbi__extend_receive(j,s) * dequant[zig]);
         }
      }
   } while (k < 64);
   // 返回成功
   return 1;
}

// 解码渐进扫描模式下的 DC 分量
static int stbi__jpeg_decode_block_prog_dc(stbi__jpeg *j, short data[64], stbi__huffman *hdc, int b)
{
   // 声明变量 diff 和 dc，用于存储差值和 DC 系数
   int diff,dc;
   // 声明变量 t
   int t;
   // 如果 spec_end 不为 0，则返回错误信息
   if (j->spec_end != 0) return stbi__err("can't merge dc and ac", "Corrupt JPEG");

   // 如果 code_bits 小于 16，则扩展缓冲区
   if (j->code_bits < 16) stbi__grow_buffer_unsafe(j);

   // 如果 succ_high 为 0
   if (j->succ_high == 0) {
      // 第一次扫描 DC 系数，必须在第一位
      // 将 data 数组中的所有值设为 0
      memset(data,0,64*sizeof(data[0])); // 0 all the ac values now
      // 从哈夫曼表 hdc 中解码 t
      t = stbi__jpeg_huff_decode(j, hdc);
      // 如果 t 小于 0 或大于 15，则返回错误信息
      if (t < 0 || t > 15) return stbi__err("can't merge dc and ac", "Corrupt JPEG");
      // 计算差值
      diff = t ? stbi__extend_receive(j, t) : 0;

      // 如果 dc_pred 和 diff 之和无效，则返回错误信息
      if (!stbi__addints_valid(j->img_comp[b].dc_pred, diff)) return stbi__err("bad delta", "Corrupt JPEG");
      // 计算 dc
      dc = j->img_comp[b].dc_pred + diff;
      j->img_comp[b].dc_pred = dc;
      // 如果 dc 乘以 2^succ_low 无效，则返回错误信息
      if (!stbi__mul2shorts_valid(dc, 1 << j->succ_low)) return stbi__err("can't merge dc and ac", "Corrupt JPEG");
      // 将计算结果存入 data 数组中
      data[0] = (short) (dc * (1 << j->succ_low));
   } else {
      // DC 系数的精炼扫描
      // 如果获取到一个比特位，则将 data[0] 加上 1<<succ_low
      if (stbi__jpeg_get_bit(j))
         data[0] += (short) (1 << j->succ_low);
   }
   // 返回 1
   return 1;
}

// @OPTIMIZE: store non-zigzagged during the decode passes,
// and only de-zigzag when dequantizing
// 解码进度扫描 AC 系数
static int stbi__jpeg_decode_block_prog_ac(stbi__jpeg *j, short data[64], stbi__huffman *hac, stbi__int16 *fac)
}

// 将 -128 到 127 的值限制在 0 到 255 之间
stbi_inline static stbi_uc stbi__clamp(int x)
{
   // 通过一个测试来处理两种情况
   if ((unsigned int) x > 255) {
      if (x < 0) return 0;
      if (x > 255) return 255;
   }
   return (stbi_uc) x;
}

// 定义宏 stbi__f2f，将浮点数转换为整数
#define stbi__f2f(x)  ((int) (((x) * 4096 + 0.5)))
// 定义宏 stbi__fsh，将浮点数转换为整数
#define stbi__fsh(x)  ((x) * 4096)

// 源自 jidctint -- DCT_ISLOW
// 定义一维 IDCT 变换的宏，用于计算 DCT 变换后的值
#define STBI__IDCT_1D(s0,s1,s2,s3,s4,s5,s6,s7) \
   int t0,t1,t2,t3,p1,p2,p3,p4,p5,x0,x1,x2,x3; \  // 定义变量
   p2 = s2;                                    \  // 赋值操作
   p3 = s6;                                    \  // 赋值操作
   p1 = (p2+p3) * stbi__f2f(0.5411961f);       \  // 计算操作
   t2 = p1 + p3*stbi__f2f(-1.847759065f);      \  // 计算操作
   t3 = p1 + p2*stbi__f2f( 0.765366865f);      \  // 计算操作
   p2 = s0;                                    \  // 赋值操作
   p3 = s4;                                    \  // 赋值操作
   t0 = stbi__fsh(p2+p3);                      \  // 计算操作
   t1 = stbi__fsh(p2-p3);                      \  // 计算操作
   x0 = t0+t3;                                 \  // 计算操作
   x3 = t0-t3;                                 \  // 计算操作
   x1 = t1+t2;                                 \  // 计算操作
   x2 = t1-t2;                                 \  // 计算操作
   t0 = s7;                                    \  // 赋值操作
   t1 = s5;                                    \  // 赋值操作
   t2 = s3;                                    \  // 赋值操作
   t3 = s1;                                    \  // 赋值操作
   p3 = t0+t2;                                 \  // 计算操作
   p4 = t1+t3;                                 \  // 计算操作
   p1 = t0+t3;                                 \  // 计算操作
   p2 = t1+t2;                                 \  // 计算操作
   p5 = (p3+p4)*stbi__f2f( 1.175875602f);      \  // 计算操作
   t0 = t0*stbi__f2f( 0.298631336f);           \  // 计算操作
   t1 = t1*stbi__f2f( 2.053119869f);           \  // 计算操作
   t2 = t2*stbi__f2f( 3.072711026f);           \  // 计算操作
   t3 = t3*stbi__f2f( 1.501321110f);           \  // 计算操作
   p1 = p5 + p1*stbi__f2f(-0.899976223f);      \  // 计算操作
   p2 = p5 + p2*stbi__f2f(-2.562915447f);      \  // 计算操作
   p3 = p3*stbi__f2f(-1.961570560f);           \  // 计算操作
   p4 = p4*stbi__f2f(-0.390180644f);           \  // 计算操作
   t3 += p1+p4;                                \  // 计算操作
   t2 += p2+p3;                                \  // 计算操作
   t1 += p2+p4;                                \  // 计算操作
   t0 += p1+p3;                                \  // 计算操作

// 对 8x8 块进行 IDCT 变换
static void stbi__idct_block(stbi_uc *out, int out_stride, short data[64])
}

#ifdef STBI_SSE2
// 使用 SSE2 指令集进行整数 IDCT 变换，虽然不是最快的实现方式，但与通用 C 版本产生完全相同的结果，因此是完全“透明”的
#ifdef STBI_NEON

// NEON integer IDCT. should produce bit-identical
// results to the generic C version.
// 使用 NEON 指令集实现整数 IDCT，应该产生与通用 C 版本完全相同的结果
static void stbi__idct_simd(stbi_uc *out, int out_stride, short data[64])
{
   // 定义 NEON 寄存器变量
   int16x8_t row0, row1, row2, row3, row4, row5, row6, row7;

   // 定义旋转系数
   int16x4_t rot0_0 = vdup_n_s16(stbi__f2f(0.5411961f));
   int16x4_t rot0_1 = vdup_n_s16(stbi__f2f(-1.847759065f));
   int16x4_t rot0_2 = vdup_n_s16(stbi__f2f( 0.765366865f));
   int16x4_t rot1_0 = vdup_n_s16(stbi__f2f( 1.175875602f));
   int16x4_t rot1_1 = vdup_n_s16(stbi__f2f(-0.899976223f));
   int16x4_t rot1_2 = vdup_n_s16(stbi__f2f(-2.562915447f));
   int16x4_t rot2_0 = vdup_n_s16(stbi__f2f(-1.961570560f));
   int16x4_t rot2_1 = vdup_n_s16(stbi__f2f(-0.390180644f));
   int16x4_t rot3_0 = vdup_n_s16(stbi__f2f( 0.298631336f));
   int16x4_t rot3_1 = vdup_n_s16(stbi__f2f( 2.053119869f));
   int16x4_t rot3_2 = vdup_n_s16(stbi__f2f( 3.072711026f));
   int16x4_t rot3_3 = vdup_n_s16(stbi__f2f( 1.501321110f));

   // 定义宏，用于进行长乘法
#define dct_long_mul(out, inq, coeff) \
   int32x4_t out##_l = vmull_s16(vget_low_s16(inq), coeff); \
   int32x4_t out##_h = vmull_s16(vget_high_s16(inq), coeff)

   // 定义宏，用于进行长乘加
#define dct_long_mac(out, acc, inq, coeff) \
   int32x4_t out##_l = vmlal_s16(acc##_l, vget_low_s16(inq), coeff); \
   int32x4_t out##_h = vmlal_s16(acc##_h, vget_high_s16(inq), coeff)

   // 定义宏，用于扩展数据位宽
#define dct_widen(out, inq) \
   int32x4_t out##_l = vshll_n_s16(vget_low_s16(inq), 12); \
   int32x4_t out##_h = vshll_n_s16(vget_high_s16(inq), 12)

   // 定义宏，用于宽度相加
#define dct_wadd(out, a, b) \
   int32x4_t out##_l = vaddq_s32(a##_l, b##_l); \
   int32x4_t out##_h = vaddq_s32(a##_h, b##_h)

   // 定义宏，用于宽度相减
// 定义宏，计算两个向量的差值
#define dct_wsub(out, a, b) \
   int32x4_t out##_l = vsubq_s32(a##_l, b##_l); \
   int32x4_t out##_h = vsubq_s32(a##_h, b##_h)

// 宏定义，对两个向量进行蝶形操作，然后使用"shiftop"进行位移，位移量为"s"，并打包结果
#define dct_bfly32o(out0,out1, a,b,shiftop,s) \
   { \
      // 计算两个向量的和
      dct_wadd(sum, a, b); \
      // 计算两个向量的差
      dct_wsub(dif, a, b); \
      // 将结果进行位移和打包
      out0 = vcombine_s16(shiftop(sum_l, s), shiftop(sum_h, s)); \
      out1 = vcombine_s16(shiftop(dif_l, s), shiftop(dif_h, s)); \
   }

// 这三个宏分别映射到一个VTRN.16、VTRN.32和VSWP。不幸的是，编译器是否真正理解这一点是另一回事。
#define dct_trn16(x, y) { int16x8x2_t t = vtrnq_s16(x, y); x = t.val[0]; y = t.val[1]; }
#define dct_trn32(x, y) { int32x4x2_t t = vtrnq_s32(vreinterpretq_s32_s16(x), vreinterpretq_s32_s16(y)); x = vreinterpretq_s16_s32(t.val[0]); y = vreinterpretq_s16_s32(t.val[1]); }
#define dct_trn64(x, y) { int16x8_t x0 = x; int16x8_t y0 = y; x = vcombine_s16(vget_low_s16(x0), vget_low_s16(y0)); y = vcombine_s16(vget_high_s16(x0), vget_high_s16(y0)); }

      // 第一次处理
      dct_trn16(row0, row1); // a0b0a2b2a4b4a6b6
      dct_trn16(row2, row3);
      dct_trn16(row4, row5);
      dct_trn16(row6, row7);

      // 第二次处理
      dct_trn32(row0, row2); // a0b0c0d0a4b4c4d4
      dct_trn32(row1, row3);
      dct_trn32(row4, row6);
      dct_trn32(row5, row7);

      // 第三次处理
      dct_trn64(row0, row4); // a0b0c0d0e0f0g0h0
      dct_trn64(row1, row5);
      dct_trn64(row2, row6);
      dct_trn64(row3, row7);

#undef dct_trn16
#undef dct_trn32
   // 取消定义 dct_trn64 宏

   // 行处理
   // vrshrn_n_s32 只支持最大 16 的位移，我们需要 17 位的位移。
   // 因此先进行一个非四舍五入的 16 位移，然后再进行一个四舍五入的 1 位移。
   dct_pass(vshrn_n_s32, 16);

   {
      // 打包和四舍五入
      uint8x8_t p0 = vqrshrun_n_s16(row0, 1);
      uint8x8_t p1 = vqrshrun_n_s16(row1, 1);
      uint8x8_t p2 = vqrshrun_n_s16(row2, 1);
      uint8x8_t p3 = vqrshrun_n_s16(row3, 1);
      uint8x8_t p4 = vqrshrun_n_s16(row4, 1);
      uint8x8_t p5 = vqrshrun_n_s16(row5, 1);
      uint8x8_t p6 = vqrshrun_n_s16(row6, 1);
      uint8x8_t p7 = vqrshrun_n_s16(row7, 1);

      // 再次说明，这些可以转换为一条指令，但通常不会。
#define dct_trn8_8(x, y) { uint8x8x2_t t = vtrn_u8(x, y); x = t.val[0]; y = t.val[1]; }
#define dct_trn8_16(x, y) { uint16x4x2_t t = vtrn_u16(vreinterpret_u16_u8(x), vreinterpret_u16_u8(y)); x = vreinterpret_u8_u16(t.val[0]); y = vreinterpret_u8_u16(t.val[1]); }
#define dct_trn8_32(x, y) { uint32x2x2_t t = vtrn_u32(vreinterpret_u32_u8(x), vreinterpret_u32_u8(y)); x = vreinterpret_u8_u32(t.val[0]); y = vreinterpret_u8_u32(t.val[1]); }

      // 遗憾的是在这里不能使用交错存储，因为我们每个扫描线只写入 8 字节！

      // 8x8 8 位转置第一步
      dct_trn8_8(p0, p1);
      dct_trn8_8(p2, p3);
      dct_trn8_8(p4, p5);
      dct_trn8_8(p6, p7);

      // 第二步
      dct_trn8_16(p0, p2);
      dct_trn8_16(p1, p3);
      dct_trn8_16(p4, p6);
      dct_trn8_16(p5, p7);

      // 第三步
      dct_trn8_32(p0, p4);
      dct_trn8_32(p1, p5);
      dct_trn8_32(p2, p6);
      dct_trn8_32(p3, p7);

      // 存储
      vst1_u8(out, p0); out += out_stride;
      vst1_u8(out, p1); out += out_stride;
      vst1_u8(out, p2); out += out_stride;
      vst1_u8(out, p3); out += out_stride;
      vst1_u8(out, p4); out += out_stride;
      vst1_u8(out, p5); out += out_stride;
      vst1_u8(out, p6); out += out_stride;
      vst1_u8(out, p7);

#undef dct_trn8_8
// 取消定义 dct_trn8_16
// 取消定义 dct_trn8_32
   }

// 取消定义 dct_long_mul
// 取消定义 dct_long_mac
// 取消定义 dct_widen
// 取消定义 dct_wadd
// 取消定义 dct_wsub
// 取消定义 dct_bfly32o
// 取消定义 dct_pass
}

#endif // STBI_NEON

// 定义 STBI__MARKER_none 为 0xff
// 如果熵流中有待处理的标记，则返回该标记
// 否则，从流中获取一个标记。如果没有标记，则返回 0xff，这是一个无效的标记值
static stbi_uc stbi__get_marker(stbi__jpeg *j)
{
   stbi_uc x;
   if (j->marker != STBI__MARKER_none) { x = j->marker; j->marker = STBI__MARKER_none; return x; }
   x = stbi__get8(j->s);
   if (x != 0xff) return STBI__MARKER_none;
   while (x == 0xff)
      x = stbi__get8(j->s); // 消耗重复的 0xff 填充字节
   return x;
}

// 在每个扫描中，我们将有 scan_n 个分量，分量的顺序由 order[] 指定
#define STBI__RESTART(x)     ((x) >= 0xd0 && (x) <= 0xd7)

// 在重启间隔之后，重置熵解码器和 DC 预测
static void stbi__jpeg_reset(stbi__jpeg *j)
{
   j->code_bits = 0;
   j->code_buffer = 0;
   j->nomore = 0;
   j->img_comp[0].dc_pred = j->img_comp[1].dc_pred = j->img_comp[2].dc_pred = j->img_comp[3].dc_pred = 0;
   j->marker = STBI__MARKER_none;
   j->todo = j->restart_interval ? j->restart_interval : 0x7fffffff;
   j->eob_run = 0;
   // 如果没有 restart_interval，最多不超过 1<<31 个 MCU，这是非常安全的，因为我们甚至不允许 1<<30 个像素
}

static int stbi__parse_entropy_coded_data(stbi__jpeg *z)
}

static void stbi__jpeg_dequantize(short *data, stbi__uint16 *dequant)
{
   int i;
   for (i=0; i < 64; ++i)
      data[i] *= dequant[i];
}

static void stbi__jpeg_finish(stbi__jpeg *z)
{
   // 如果图像是渐进式的
   if (z->progressive) {
      // 对数据进行反量化和逆 DCT 变换
      int i,j,n;
      for (n=0; n < z->s->img_n; ++n) {
         // 计算每个分量的宽度和高度
         int w = (z->img_comp[n].x+7) >> 3;
         int h = (z->img_comp[n].y+7) >> 3;
         for (j=0; j < h; ++j) {
            for (i=0; i < w; ++i) {
               // 获取当前块的数据指针
               short *data = z->img_comp[n].coeff + 64 * (i + j * z->img_comp[n].coeff_w);
               // 对数据进行反量化
               stbi__jpeg_dequantize(data, z->dequant[z->img_comp[n].tq]);
               // 对数据进行逆 DCT 变换
               z->idct_block_kernel(z->img_comp[n].data+z->img_comp[n].w2*j*8+i*8, z->img_comp[n].w2, data);
            }
         }
      }
   }
}

// 处理标记后的扫描头
static int stbi__process_marker(stbi__jpeg *z, int m)
}

// 在遇到 SOS 之后
static int stbi__process_scan_header(stbi__jpeg *z)
{
   // 定义变量 i
   int i;
   // 读取 16 位大端序数据，存入变量 Ls
   int Ls = stbi__get16be(z->s);
   // 读取扫描组件数，存入 z->scan_n
   z->scan_n = stbi__get8(z->s);
   // 检查扫描组件数是否合法
   if (z->scan_n < 1 || z->scan_n > 4 || z->scan_n > (int) z->s->img_n) return stbi__err("bad SOS component count","Corrupt JPEG");
   // 检查扫描长度是否正确
   if (Ls != 6+2*z->scan_n) return stbi__err("bad SOS len","Corrupt JPEG");
   // 遍历扫描组件
   for (i=0; i < z->scan_n; ++i) {
      // 读取组件 id
      int id = stbi__get8(z->s), which;
      // 读取量化表 q
      int q = stbi__get8(z->s);
      // 查找对应的图像组件
      for (which = 0; which < z->s->img_n; ++which)
         if (z->img_comp[which].id == id)
            break;
      // 如果没有找到匹配的组件，返回 0
      if (which == z->s->img_n) return 0; // no match
      // 设置 DC 和 AC 哈夫曼表
      z->img_comp[which].hd = q >> 4;   if (z->img_comp[which].hd > 3) return stbi__err("bad DC huff","Corrupt JPEG");
      z->img_comp[which].ha = q & 15;   if (z->img_comp[which].ha > 3) return stbi__err("bad AC huff","Corrupt JPEG");
      // 记录组件顺序
      z->order[i] = which;
   }

   {
      // 定义变量 aa
      int aa;
      // 读取 spec_start
      z->spec_start = stbi__get8(z->s);
      // 读取 spec_end
      z->spec_end   = stbi__get8(z->s); // should be 63, but might be 0
      // 读取 aa
      aa = stbi__get8(z->s);
      // 分别设置 succ_high 和 succ_low
      z->succ_high = (aa >> 4);
      z->succ_low  = (aa & 15);
      // 检查参数是否合法
      if (z->progressive) {
         if (z->spec_start > 63 || z->spec_end > 63  || z->spec_start > z->spec_end || z->succ_high > 13 || z->succ_low > 13)
            return stbi__err("bad SOS", "Corrupt JPEG");
      } else {
         if (z->spec_start != 0) return stbi__err("bad SOS","Corrupt JPEG");
         if (z->succ_high != 0 || z->succ_low != 0) return stbi__err("bad SOS","Corrupt JPEG");
         z->spec_end = 63;
      }
   }

   // 返回成功
   return 1;
}

// 释放 JPEG 组件
static int stbi__free_jpeg_components(stbi__jpeg *z, int ncomp, int why)
{
   // 循环遍历每个图像分量
   int i;
   for (i=0; i < ncomp; ++i) {
      // 如果图像分量的原始数据存在，则释放内存并将指针置为空
      if (z->img_comp[i].raw_data) {
         STBI_FREE(z->img_comp[i].raw_data);
         z->img_comp[i].raw_data = NULL;
         z->img_comp[i].data = NULL;
      }
      // 如果图像分量的原始系数存在，则释放内存并将指针置为0
      if (z->img_comp[i].raw_coeff) {
         STBI_FREE(z->img_comp[i].raw_coeff);
         z->img_comp[i].raw_coeff = 0;
         z->img_comp[i].coeff = 0;
      }
      // 如果图像分量的行缓冲存在，则释放内存并将指针置为空
      if (z->img_comp[i].linebuf) {
         STBI_FREE(z->img_comp[i].linebuf);
         z->img_comp[i].linebuf = NULL;
      }
   }
   // 返回原因
   return why;
}

// 处理帧头部信息
static int stbi__process_frame_header(stbi__jpeg *z, int scan)
}

// 定义宏，用于比较是否为不同的标记
#define stbi__DNL(x)         ((x) == 0xdc)
#define stbi__SOI(x)         ((x) == 0xd8)
#define stbi__EOI(x)         ((x) == 0xd9)
#define stbi__SOF(x)         ((x) == 0xc0 || (x) == 0xc1 || (x) == 0xc2)
#define stbi__SOS(x)         ((x) == 0xda)

#define stbi__SOF_progressive(x)   ((x) == 0xc2)

// 解码 JPEG 头部信息
static int stbi__decode_jpeg_header(stbi__jpeg *z, int scan)
{
   int m;
   // 初始化变量
   z->jfif = 0;
   z->app14_color_transform = -1; // 有效值为 0,1,2
   z->marker = STBI__MARKER_none; // 初始化缓存的标记为空
   // 获取标记
   m = stbi__get_marker(z);
   // 如果不是 SOI 标记，则返回错误信息
   if (!stbi__SOI(m)) return stbi__err("no SOI","Corrupt JPEG");
   // 如果扫描类型为 STBI__SCAN_type，则返回1
   if (scan == STBI__SCAN_type) return 1;
   // 获取下一个标记
   m = stbi__get_marker(z);
   // 循环直到遇到 SOF 标记
   while (!stbi__SOF(m)) {
      // 处理标记
      if (!stbi__process_marker(z,m)) return 0;
      m = stbi__get_marker(z);
      // 如果标记为空，则继续扫描
      while (m == STBI__MARKER_none) {
         // 一些文件在块之后有额外的填充，因此继续扫描
         if (stbi__at_eof(z->s)) return stbi__err("no SOF", "Corrupt JPEG");
         m = stbi__get_marker(z);
      }
   }
   // 判断是否为渐进式 JPEG
   z->progressive = stbi__SOF_progressive(m);
   // 处理帧头部信息
   if (!stbi__process_frame_header(z, scan)) return 0;
   return 1;
}

// 跳过 JPEG 结尾处的无用数据
static int stbi__skip_jpeg_junk_at_end(stbi__jpeg *j)
{
   // 某些 JPEG 图像末尾可能有垃圾数据，跳过这部分，但如果找到类似有效标记的内容，则从那里继续解析
   while (!stbi__at_eof(j->s)) {
      // 读取一个字节
      int x = stbi__get8(j->s);
      // 如果读取到的字节为 255，可能是一个标记
      while (x == 255) { // 可能是一个标记
         // 如果已经到达文件末尾，则返回标记为 none
         if (stbi__at_eof(j->s)) return STBI__MARKER_none;
         // 继续读取下一个字节
         x = stbi__get8(j->s);
         // 如果读取到的字节不是 0x00 也不是 0xff
         if (x != 0x00 && x != 0xff) {
            // 不是填充的零或者另一个标记的前导，看起来是一个实际的标记，返回该标记
            return x;
         }
         // 填充的零现在 x=0，结束循环，意味着回到常规扫描循环
         // 重复的 0xff 继续尝试读取标记的下一个字节
      }
   }
   // 如果没有找到标记，返回标记为 none
   return STBI__MARKER_none;
}

// 将图像解码为 YCbCr 格式
static int stbi__decode_jpeg_image(stbi__jpeg *j)
{
   // 初始化变量 m
   int m;
   // 循环4次
   for (m = 0; m < 4; m++) {
      // 将图像组件的原始数据和原始系数设置为 NULL
      j->img_comp[m].raw_data = NULL;
      j->img_comp[m].raw_coeff = NULL;
   }
   // 重置间隔为 0
   j->restart_interval = 0;
   // 解码 JPEG 头部信息
   if (!stbi__decode_jpeg_header(j, STBI__SCAN_load)) return 0;
   // 获取标记
   m = stbi__get_marker(j);
   // 循环直到遇到结束标记
   while (!stbi__EOI(m)) {
      // 如果是扫描开始标记
      if (stbi__SOS(m)) {
         // 处理扫描头部信息
         if (!stbi__process_scan_header(j)) return 0;
         // 解析熵编码数据
         if (!stbi__parse_entropy_coded_data(j)) return 0;
         // 如果标记为 none，则跳过 JPEG 末尾的垃圾数据
         if (j->marker == STBI__MARKER_none ) {
            j->marker = stbi__skip_jpeg_junk_at_end(j);
            // 如果在没有遇到标记的情况下到达文件末尾，stbi__get_marker() 将失败并最终返回 0
         }
         // 获取下一个标记
         m = stbi__get_marker(j);
         // 如果是重启标记，则再次获取下一个标记
         if (STBI__RESTART(m))
            m = stbi__get_marker(j);
      } else if (stbi__DNL(m)) {
         // 获取 DNL 标记的长度和高度
         int Ld = stbi__get16be(j->s);
         stbi__uint32 NL = stbi__get16be(j->s);
         // 检查长度和高度是否正确
         if (Ld != 4) return stbi__err("bad DNL len", "Corrupt JPEG");
         if (NL != j->s->img_y) return stbi__err("bad DNL height", "Corrupt JPEG");
         // 获取下一个标记
         m = stbi__get_marker(j);
      } else {
         // 处理其他标记
         if (!stbi__process_marker(j, m)) return 1;
         // 获取下一个标记
         m = stbi__get_marker(j);
      }
   }
   // 如果是渐进式 JPEG，则完成 JPEG 处理
   if (j->progressive)
      stbi__jpeg_finish(j);
   // 返回成功
   return 1;
}

// 静态的 jfif 居中重采样（跨块边界）

typedef stbi_uc *(*resample_row_func)(stbi_uc *out, stbi_uc *in0, stbi_uc *in1,
                                    int w, int hs);

// 定义宏，将 x 除以 4
#define stbi__div4(x) ((stbi_uc) ((x) >> 2))

// 静态函数，重采样行为 1
static stbi_uc *resample_row_1(stbi_uc *out, stbi_uc *in_near, stbi_uc *in_far, int w, int hs)
{
   // 不使用 out, in_far, w, hs
   STBI_NOTUSED(out);
   STBI_NOTUSED(in_far);
   STBI_NOTUSED(w);
   STBI_NOTUSED(hs);
   // 返回输入的近邻像素值
   return in_near;
}

// 静态函数，垂直重采样行为 2
static stbi_uc* stbi__resample_row_v_2(stbi_uc *out, stbi_uc *in_near, stbi_uc *in_far, int w, int hs)
{
   // 需要为每个输入像素生成两个垂直样本
   int i;
   STBI_NOTUSED(hs);
   // 对每个像素进行计算
   for (i=0; i < w; ++i)
      // 计算输出像素值
      out[i] = stbi__div4(3*in_near[i] + in_far[i] + 2);
   // 返回输出像素数组
   return out;
}
}
static stbi_uc*  stbi__resample_row_h_2(stbi_uc *out, stbi_uc *in_near, stbi_uc *in_far, int w, int hs)
{
   // 需要为每个输入样本水平生成两个样本
   int i;
   stbi_uc *input = in_near;

   if (w == 1) {
      // 如果只有一个样本，无法进行任何插值
      out[0] = out[1] = input[0];
      return out;
   }

   out[0] = input[0];
   out[1] = stbi__div4(input[0]*3 + input[1] + 2);
   for (i=1; i < w-1; ++i) {
      int n = 3*input[i]+2;
      out[i*2+0] = stbi__div4(n+input[i-1]);
      out[i*2+1] = stbi__div4(n+input[i+1]);
   }
   out[i*2+0] = stbi__div4(input[w-2]*3 + input[w-1] + 2);
   out[i*2+1] = input[w-1];

   STBI_NOTUSED(in_far);
   STBI_NOTUSED(hs);

   return out;
}

#define stbi__div16(x) ((stbi_uc) ((x) >> 4))

static stbi_uc *stbi__resample_row_hv_2(stbi_uc *out, stbi_uc *in_near, stbi_uc *in_far, int w, int hs)
{
   // 需要为每个输入生成2x2个样本
   int i,t0,t1;
   if (w == 1) {
      out[0] = out[1] = stbi__div4(3*in_near[0] + in_far[0] + 2);
      return out;
   }

   t1 = 3*in_near[0] + in_far[0];
   out[0] = stbi__div4(t1+2);
   for (i=1; i < w; ++i) {
      t0 = t1;
      t1 = 3*in_near[i]+in_far[i];
      out[i*2-1] = stbi__div16(3*t0 + t1 + 8);
      out[i*2  ] = stbi__div16(3*t1 + t0 + 8);
   }
   out[w*2-1] = stbi__div4(t1+2);

   STBI_NOTUSED(hs);

   return out;
}

#if defined(STBI_SSE2) || defined(STBI_NEON)
static stbi_uc *stbi__resample_row_hv_2_simd(stbi_uc *out, stbi_uc *in_near, stbi_uc *in_far, int w, int hs)
{
   // 需要为每个输入生成2x2个样本
   int i=0,t0,t1;

   if (w == 1) {
      out[0] = out[1] = stbi__div4(3*in_near[0] + in_far[0] + 2);
      return out;
   }

   t1 = 3*in_near[0] + in_far[0];
   // 处理每组8个像素，尽可能多地处理
   // 注意，我们无法在此循环中处理行中的最后一个像素
   // 因为我们需要处理滤波器的边界条件。
   for (; i < ((w-1) & ~7); i += 8) {
#elif defined(STBI_NEON)
      // 如果定义了 STBI_NEON，则执行以下代码块
      // 加载并执行垂直滤波处理
      // 这里使用了 3*x + y = 4*x + (y - x) 的计算方式
      uint8x8_t farb  = vld1_u8(in_far + i);
      uint8x8_t nearb = vld1_u8(in_near + i);
      int16x8_t diff  = vreinterpretq_s16_u16(vsubl_u8(farb, nearb));
      int16x8_t nears = vreinterpretq_s16_u16(vshll_n_u8(nearb, 2));
      int16x8_t curr  = vaddq_s16(nears, diff); // 当前行

      // 水平滤波处理与当前行的偏移版本相同
      // "prev" 是当前行向右偏移一个像素；我们需要插入前一个像素值（来自 t1）
      // "next" 是当前行向左偏移一个像素，同时添加下一个 8 像素块的第一个像素
      int16x8_t prv0 = vextq_s16(curr, curr, 7);
      int16x8_t nxt0 = vextq_s16(curr, curr, 1);
      int16x8_t prev = vsetq_lane_s16(t1, prv0, 0);
      int16x8_t next = vsetq_lane_s16(3*in_near[i+8] + in_far[i+8], nxt0, 7);

      // 水平滤波，采用多相实现方式，因为这样更方便：
      // 偶数像素 = 3*cur + prev = cur*4 + (prev - cur)
      // 奇数像素 = 3*cur + next = cur*4 + (next - cur)
      // 注意共享的项
      int16x8_t curs = vshlq_n_s16(curr, 2);
      int16x8_t prvd = vsubq_s16(prev, curr);
      int16x8_t nxtd = vsubq_s16(next, curr);
      int16x8_t even = vaddq_s16(curs, prvd);
      int16x8_t odd  = vaddq_s16(curs, nxtd);

      // 恢复缩放并四舍五入，然后交替存储在偶数/奇数相位中
      uint8x8x2_t o;
      o.val[0] = vqrshrun_n_s16(even, 4);
      o.val[1] = vqrshrun_n_s16(odd,  4);
      vst2_u8(out + i*2, o);
#endif

      // 计算下一次迭代的“previous”值
      t1 = 3*in_near[i+7] + in_far[i+7];
   }

   t0 = t1;
   t1 = 3*in_near[i] + in_far[i];
   out[i*2] = stbi__div16(3*t1 + t0 + 8);

   for (++i; i < w; ++i) {
      t0 = t1;
      t1 = 3*in_near[i]+in_far[i];
      out[i*2-1] = stbi__div16(3*t0 + t1 + 8);
      out[i*2  ] = stbi__div16(3*t1 + t0 + 8);
   }
   out[w*2-1] = stbi__div4(t1+2);

   STBI_NOTUSED(hs);

   return out;
}
#endif

static stbi_uc *stbi__resample_row_generic(stbi_uc *out, stbi_uc *in_near, stbi_uc *in_far, int w, int hs)
{
   // 使用最近邻插值重新采样
   int i,j;
   STBI_NOTUSED(in_far);
   for (i=0; i < w; ++i)
      for (j=0; j < hs; ++j)
         out[i*hs+j] = in_near[i];
   return out;
}

// 这是一个降低精度的 YCbCr 到 RGB 的计算，用于确保代码在 SIMD 和标量中产生相同结果
#define stbi__float2fixed(x)  (((int) ((x) * 4096.0f + 0.5f)) << 8)
static void stbi__YCbCr_to_RGB_row(stbi_uc *out, const stbi_uc *y, const stbi_uc *pcb, const stbi_uc *pcr, int count, int step)
{
   int i;
   for (i=0; i < count; ++i) {
      int y_fixed = (y[i] << 20) + (1<<19); // 四舍五入
      int r,g,b;
      int cr = pcr[i] - 128;
      int cb = pcb[i] - 128;
      r = y_fixed +  cr* stbi__float2fixed(1.40200f);
      g = y_fixed + (cr*-stbi__float2fixed(0.71414f)) + ((cb*-stbi__float2fixed(0.34414f)) & 0xffff0000);
      b = y_fixed                                     +   cb* stbi__float2fixed(1.77200f);
      r >>= 20;
      g >>= 20;
      b >>= 20;
      if ((unsigned) r > 255) { if (r < 0) r = 0; else r = 255; }
      if ((unsigned) g > 255) { if (g < 0) g = 0; else g = 255; }
      if ((unsigned) b > 255) { if (b < 0) b = 0; else b = 255; }
      out[0] = (stbi_uc)r;
      out[1] = (stbi_uc)g;
      out[2] = (stbi_uc)b;
      out[3] = 255;
      out += step;
   }
}

#if defined(STBI_SSE2) || defined(STBI_NEON)
# 将 YCbCr 颜色空间转换为 RGB 颜色空间，使用 SIMD 指令加速计算
static void stbi__YCbCr_to_RGB_simd(stbi_uc *out, stbi_uc const *y, stbi_uc const *pcb, stbi_uc const *pcr, int count, int step)
{
    # 初始化变量 i 为 0
    int i = 0;

#endif
#ifdef STBI_NEON
   // 如果支持 step=3，可以很容易添加，但是否有需求呢？
   if (step == 4) {
      // 这是一个相当直接的实现，没有经过超级优化。
      uint8x8_t signflip = vdup_n_u8(0x80);
      int16x8_t cr_const0 = vdupq_n_s16(   (short) ( 1.40200f*4096.0f+0.5f));
      int16x8_t cr_const1 = vdupq_n_s16( - (short) ( 0.71414f*4096.0f+0.5f));
      int16x8_t cb_const0 = vdupq_n_s16( - (short) ( 0.34414f*4096.0f+0.5f));
      int16x8_t cb_const1 = vdupq_n_s16(   (short) ( 1.77200f*4096.0f+0.5f));

      for (; i+7 < count; i += 8) {
         // 加载
         uint8x8_t y_bytes  = vld1_u8(y + i);
         uint8x8_t cr_bytes = vld1_u8(pcr + i);
         uint8x8_t cb_bytes = vld1_u8(pcb + i);
         int8x8_t cr_biased = vreinterpret_s8_u8(vsub_u8(cr_bytes, signflip));
         int8x8_t cb_biased = vreinterpret_s8_u8(vsub_u8(cb_bytes, signflip));

         // 扩展为 s16
         int16x8_t yws = vreinterpretq_s16_u16(vshll_n_u8(y_bytes, 4));
         int16x8_t crw = vshll_n_s8(cr_biased, 7);
         int16x8_t cbw = vshll_n_s8(cb_biased, 7);

         // 颜色转换
         int16x8_t cr0 = vqdmulhq_s16(crw, cr_const0);
         int16x8_t cb0 = vqdmulhq_s16(cbw, cb_const0);
         int16x8_t cr1 = vqdmulhq_s16(crw, cr_const1);
         int16x8_t cb1 = vqdmulhq_s16(cbw, cb_const1);
         int16x8_t rws = vaddq_s16(yws, cr0);
         int16x8_t gws = vaddq_s16(vaddq_s16(yws, cb0), cr1);
         int16x8_t bws = vaddq_s16(yws, cb1);

         // 撤销缩放，四舍五入，转换为字节
         uint8x8x4_t o;
         o.val[0] = vqrshrun_n_s16(rws, 4);
         o.val[1] = vqrshrun_n_s16(gws, 4);
         o.val[2] = vqrshrun_n_s16(bws, 4);
         o.val[3] = vdup_n_u8(255);

         // 存储，交错存储 r/g/b/a
         vst4_u8(out, o);
         out += 8*4;
      }
   }
#endif



// 循环处理每个像素
for (; i < count; ++i) {
    // 对 Y 值进行固定点表示，进行四舍五入
    int y_fixed = (y[i] << 20) + (1<<19); // rounding
    int r,g,b;
    int cr = pcr[i] - 128;
    int cb = pcb[i] - 128;
    // 计算 RGB 值
    r = y_fixed + cr* stbi__float2fixed(1.40200f);
    g = y_fixed + cr*-stbi__float2fixed(0.71414f) + ((cb*-stbi__float2fixed(0.34414f)) & 0xffff0000);
    b = y_fixed + cb* stbi__float2fixed(1.77200f);
    // 右移 20 位，得到最终的 RGB 值
    r >>= 20;
    g >>= 20;
    b >>= 20;
    // 确保 RGB 值在 0 到 255 之间
    if ((unsigned) r > 255) { if (r < 0) r = 0; else r = 255; }
    if ((unsigned) g > 255) { if (g < 0) g = 0; else g = 255; }
    if ((unsigned) b > 255) { if (b < 0) b = 0; else b = 255; }
    // 将计算得到的 RGB 值写入输出数组
    out[0] = (stbi_uc)r;
    out[1] = (stbi_uc)g;
    out[2] = (stbi_uc)b;
    out[3] = 255;
    out += step;
}



#endif



// 设置 JPEG 解码器的内核函数
static void stbi__setup_jpeg(stbi__jpeg *j)
{
    // 设置 IDCT 内核函数
    j->idct_block_kernel = stbi__idct_block;
    // 设置 YCbCr 到 RGB 转换内核函数
    j->YCbCr_to_RGB_kernel = stbi__YCbCr_to_RGB_row;
    // 设置水平垂直 2 倍重采样内核函数
    j->resample_row_hv_2_kernel = stbi__resample_row_hv_2;

#ifdef STBI_SSE2
    // 如果支持 SSE2 指令集，则使用相应的内核函数
    if (stbi__sse2_available()) {
        j->idct_block_kernel = stbi__idct_simd;
        j->YCbCr_to_RGB_kernel = stbi__YCbCr_to_RGB_simd;
        j->resample_row_hv_2_kernel = stbi__resample_row_hv_2_simd;
    }
#endif

#ifdef STBI_NEON
    // 如果支持 NEON 指令集，则使用相应的内核函数
    j->idct_block_kernel = stbi__idct_simd;
    j->YCbCr_to_RGB_kernel = stbi__YCbCr_to_RGB_simd;
    j->resample_row_hv_2_kernel = stbi__resample_row_hv_2_simd;
#endif
}



// 清理临时组件缓冲区
static void stbi__cleanup_jpeg(stbi__jpeg *j)
{
    // 释放 JPEG 组件
    stbi__free_jpeg_components(j, j->s->img_n, 0);
}



typedef struct
{
    resample_row_func resample;
    stbi_uc *line0,*line1;
    int hs,vs;   // 每个轴的扩展因子
    int w_lores; // 水平像素预扩展
    int ystep;   // 垂直扩展进度
    int ypos;    // 预扩展行数
} stbi__resample;



// 快速 0..255 * 0..255 => 0..255 的四舍五入乘法
static stbi_uc stbi__blinn_8x8(stbi_uc x, stbi_uc y)
{
   // 计算 t 的值，用于解码 JPEG 图像
   unsigned int t = x*y + 128;
   // 对 t 进行位运算，返回解码后的像素值
   return (stbi_uc) ((t + (t >>8)) >> 8);
}

// 加载 JPEG 图像
static stbi_uc *load_jpeg_image(stbi__jpeg *z, int *out_x, int *out_y, int *comp, int req_comp)
}

// 解析 JPEG 图像
static void *stbi__jpeg_load(stbi__context *s, int *x, int *y, int *comp, int req_comp, stbi__result_info *ri)
{
   // 定义结果指针
   unsigned char* result;
   // 分配内存给 JPEG 对象
   stbi__jpeg* j = (stbi__jpeg*) stbi__malloc(sizeof(stbi__jpeg));
   // 检查内存分配是否成功
   if (!j) return stbi__errpuc("outofmem", "Out of memory");
   // 将 JPEG 对象初始化为 0
   memset(j, 0, sizeof(stbi__jpeg));
   // 设置 JPEG 对象的输入流
   j->s = s;
   // 设置 JPEG 对象
   stbi__setup_jpeg(j);
   // 加载 JPEG 图像
   result = load_jpeg_image(j, x,y,comp,req_comp);
   // 释放 JPEG 对象内存
   STBI_FREE(j);
   // 返回加载结果
   return result;
}

// 测试 JPEG 图像
static int stbi__jpeg_test(stbi__context *s)
{
   // 定义结果变量
   int r;
   // 分配内存给 JPEG 对象
   stbi__jpeg* j = (stbi__jpeg*)stbi__malloc(sizeof(stbi__jpeg));
   // 检查内存分配是否成功
   if (!j) return stbi__err("outofmem", "Out of memory");
   // 将 JPEG 对象初始化为 0
   memset(j, 0, sizeof(stbi__jpeg));
   // 设置 JPEG 对象的输入流
   j->s = s;
   // 设置 JPEG 对象
   stbi__setup_jpeg(j);
   // 解码 JPEG 头信息
   r = stbi__decode_jpeg_header(j, STBI__SCAN_type);
   // 重置输入流
   stbi__rewind(s);
   // 释放 JPEG 对象内存
   STBI_FREE(j);
   // 返回解码结果
   return r;
}

// 获取 JPEG 图像信息
static int stbi__jpeg_info_raw(stbi__jpeg *j, int *x, int *y, int *comp)
{
   // 解析 JPEG 头信息
   if (!stbi__decode_jpeg_header(j, STBI__SCAN_header)) {
      // 重置输入流
      stbi__rewind( j->s );
      return 0;
   }
   // 获取图像宽度
   if (x) *x = j->s->img_x;
   // 获取图像高度
   if (y) *y = j->s->img_y;
   // 获取图像通道数
   if (comp) *comp = j->s->img_n >= 3 ? 3 : 1;
   return 1;
}

// 获取 JPEG 图像信息
static int stbi__jpeg_info(stbi__context *s, int *x, int *y, int *comp)
{
   // 定义结果变量
   int result;
   // 分配内存给 JPEG 对象
   stbi__jpeg* j = (stbi__jpeg*) (stbi__malloc(sizeof(stbi__jpeg)));
   // 检查内存分配是否成功
   if (!j) return stbi__err("outofmem", "Out of memory");
   // 将 JPEG 对象初始化为 0
   memset(j, 0, sizeof(stbi__jpeg));
   // 设置 JPEG 对象的输入流
   j->s = s;
   // 获取 JPEG 图像信息
   result = stbi__jpeg_info_raw(j, x, y, comp);
   // 释放 JPEG 对象内存
   STBI_FREE(j);
   // 返回获取结果
   return result;
}
#endif

// public domain zlib decode    v0.2  Sean Barrett 2006-11-18
//    simple implementation
//      - all input must be provided in an upfront buffer
//      - all output is written to a single output buffer (can malloc/realloc)
//    performance
//      - fast huffman

#ifndef STBI_NO_ZLIB
// 定义宏，快速方式比 JPEG 哈夫曼检查更快，但慢速方式更慢
#define STBI__ZFAST_BITS  9 // 在默认表中加速所有情况
#define STBI__ZFAST_MASK  ((1 << STBI__ZFAST_BITS) - 1)
#define STBI__ZNSYMS 288 // 字面/长度字母表中的符号数

// zlib 风格的哈夫曼编码
// （JPEG 从左侧打包，zlib 从右侧打包，因此无法共享代码）
typedef struct
{
   stbi__uint16 fast[1 << STBI__ZFAST_BITS];
   stbi__uint16 firstcode[16];
   int maxcode[17];
   stbi__uint16 firstsymbol[16];
   stbi_uc  size[STBI__ZNSYMS];
   stbi__uint16 value[STBI__ZNSYMS];
} stbi__zhuffman;

// 位反转函数，反转 16 位整数
stbi_inline static int stbi__bitreverse16(int n)
{
  n = ((n & 0xAAAA) >>  1) | ((n & 0x5555) << 1);
  n = ((n & 0xCCCC) >>  2) | ((n & 0x3333) << 2);
  n = ((n & 0xF0F0) >>  4) | ((n & 0x0F0F) << 4);
  n = ((n & 0xFF00) >>  8) | ((n & 0x00FF) << 8);
  return n;
}

// 位反转函数，反转指定位数的整数
stbi_inline static int stbi__bit_reverse(int v, int bits)
{
   STBI_ASSERT(bits <= 16);
   // 对于反转 n 位，反转 16 位然后右移
   // 例如，11 位，反转并右移 5 位
   return stbi__bitreverse16(v) >> (16-bits);
}

// 构建哈夫曼树
static int stbi__zbuild_huffman(stbi__zhuffman *z, const stbi_uc *sizelist, int num)
{
   // 定义变量 i 和 k，初始化 k 为 0
   int i,k=0;
   // 定义变量 code, next_code 数组和 sizes 数组
   int code, next_code[16], sizes[17];

   // DEFLATE 规范用于生成编码
   // 将 sizes 数组初始化为 0
   memset(sizes, 0, sizeof(sizes));
   // 将 z->fast 数组初始化为 0
   memset(z->fast, 0, sizeof(z->fast));
   // 遍历 sizelist 数组，统计每个大小出现的次数
   for (i=0; i < num; ++i)
      ++sizes[sizelist[i]];
   sizes[0] = 0;
   // 检查每个大小是否符合规范
   for (i=1; i < 16; ++i)
      if (sizes[i] > (1 << i))
         return stbi__err("bad sizes", "Corrupt PNG");
   code = 0;
   // 生成编码
   for (i=1; i < 16; ++i) {
      next_code[i] = code;
      z->firstcode[i] = (stbi__uint16) code;
      z->firstsymbol[i] = (stbi__uint16) k;
      code = (code + sizes[i]);
      if (sizes[i])
         if (code-1 >= (1 << i)) return stbi__err("bad codelengths","Corrupt PNG");
      z->maxcode[i] = code << (16-i); // preshift for inner loop
      code <<= 1;
      k += sizes[i];
   }
   z->maxcode[16] = 0x10000; // sentinel
   // 生成编码表
   for (i=0; i < num; ++i) {
      int s = sizelist[i];
      if (s) {
         int c = next_code[s] - z->firstcode[s] + z->firstsymbol[s];
         stbi__uint16 fastv = (stbi__uint16) ((s << 9) | i);
         z->size [c] = (stbi_uc     ) s;
         z->value[c] = (stbi__uint16) i;
         if (s <= STBI__ZFAST_BITS) {
            int j = stbi__bit_reverse(next_code[s],s);
            while (j < (1 << STBI__ZFAST_BITS)) {
               z->fast[j] = fastv;
               j += (1 << s);
            }
         }
         ++next_code[s];
      }
   }
   // 返回成功
   return 1;
}

// 从内存中读取 zlib 数据的实现，用于 PNG 读取
// 因为 PNG 允许任意拆分 zlib 流，结构上让 PNG 调用 ZLIB 再调用 PNG 很烦人
// 所以我们要求 PNG 读取所有 IDAT 并将它们合并成单个内存缓冲区

typedef struct
{
   stbi_uc *zbuffer, *zbuffer_end;
   int num_bits;
   stbi__uint32 code_buffer;

   char *zout;
   char *zout_start;
   char *zout_end;
   int   z_expandable;

   stbi__zhuffman z_length, z_distance;
} stbi__zbuf;

// 检查是否已经到达 zlib 数据的末尾
stbi_inline static int stbi__zeof(stbi__zbuf *z)
{
   return (z->zbuffer >= z->zbuffer_end);
}
}
// 从输入缓冲区中读取一个字节，如果已经到达末尾则返回0
stbi_inline static stbi_uc stbi__zget8(stbi__zbuf *z)
{
   return stbi__zeof(z) ? 0 : *z->zbuffer++;
}

// 填充位数据，直到位数据达到24位或者遇到EOF
static void stbi__fill_bits(stbi__zbuf *z)
{
   do {
      // 如果码缓冲区超过了当前位数所能表示的最大值，则将输入缓冲区指针指向末尾，视为EOF并返回
      if (z->code_buffer >= (1U << z->num_bits)) {
        z->zbuffer = z->zbuffer_end;  /* treat this as EOF so we fail. */
        return;
      }
      // 将一个字节的数据读取到码缓冲区中
      z->code_buffer |= (unsigned int) stbi__zget8(z) << z->num_bits;
      z->num_bits += 8;
   } while (z->num_bits <= 24);
}

// 从输入缓冲区中读取n位数据
stbi_inline static unsigned int stbi__zreceive(stbi__zbuf *z, int n)
{
   unsigned int k;
   // 如果当前位数小于n，则填充位数据
   if (z->num_bits < n) stbi__fill_bits(z);
   // 从码缓冲区中取出n位数据
   k = z->code_buffer & ((1 << n) - 1);
   z->code_buffer >>= n;
   z->num_bits -= n;
   return k;
}

// 通过慢速方法解码哈夫曼数据
static int stbi__zhuffman_decode_slowpath(stbi__zbuf *a, stbi__zhuffman *z)
{
   int b,s,k;
   // 未被快速查找表解析，因此使用慢速方法计算
   // 使用JPEG方法，要求MS位在顶部
   k = stbi__bit_reverse(a->code_buffer, 16);
   for (s=STBI__ZFAST_BITS+1; ; ++s)
      if (k < z->maxcode[s])
         break;
   if (s >= 16) return -1; // 无效的码字！
   // 码字大小为s，因此：
   b = (k >> (16-s)) - z->firstcode[s] + z->firstsymbol[s];
   if (b >= STBI__ZNSYMS) return -1; // 某些数据在某处损坏！
   if (z->size[b] != s) return -1;  // 最初是一个断言，但现在报告失败。
   a->code_buffer >>= s;
   a->num_bits -= s;
   return z->value[b];
}

// 解码哈夫曼数据
stbi_inline static int stbi__zhuffman_decode(stbi__zbuf *a, stbi__zhuffman *z)
{
   int b,s;
   if (a->num_bits < 16) {
      if (stbi__zeof(a)) {
         return -1;   /* 报告意外数据结束的错误。 */
      }
      stbi__fill_bits(a);
   }
   b = z->fast[a->code_buffer & STBI__ZFAST_MASK];
   if (b) {
      s = b >> 9;
      a->code_buffer >>= s;
      a->num_bits -= s;
      return b & 511;
   }
   return stbi__zhuffman_decode_slowpath(a, z);
}

// 扩展输入缓冲区，以便为n个字节腾出空间
static int stbi__zexpand(stbi__zbuf *z, char *zout, int n)
{
   // 定义指针变量 q
   char *q;
   // 定义无符号整型变量 cur, limit, old_limit
   unsigned int cur, limit, old_limit;
   // 将输出缓冲区赋值给 z->zout
   z->zout = zout;
   // 如果输出缓冲区不可扩展，则返回错误信息
   if (!z->z_expandable) return stbi__err("output buffer limit","Corrupt PNG");
   // 计算当前位置 cur
   cur   = (unsigned int) (z->zout - z->zout_start);
   // 计算限制 limit 和旧限制 old_limit
   limit = old_limit = (unsigned) (z->zout_end - z->zout_start);
   // 如果 cur + n 超出无符号整型最大值，则返回错误信息
   if (UINT_MAX - cur < (unsigned) n) return stbi__err("outofmem", "Out of memory");
   // 当 cur + n 大于 limit 时，循环扩展 limit 直到满足条件
   while (cur + n > limit) {
      // 如果 limit 大于无符号整型最大值的一半，则返回错误信息
      if(limit > UINT_MAX / 2) return stbi__err("outofmem", "Out of memory");
      // 将 limit 扩大为原来的两倍
      limit *= 2;
   }
   // 重新分配内存大小为 limit 的空间给 q
   q = (char *) STBI_REALLOC_SIZED(z->zout_start, old_limit, limit);
   // 不使用 old_limit
   STBI_NOTUSED(old_limit);
   // 如果分配内存失败，则返回错误信息
   if (q == NULL) return stbi__err("outofmem", "Out of memory");
   // 更新 zout_start, zout, zout_end 的位置
   z->zout_start = q;
   z->zout       = q + cur;
   z->zout_end   = q + limit;
   // 返回 1
   return 1;
}

// 定义静态常量数组 stbi__zlength_base
static const int stbi__zlength_base[31] = {
   3,4,5,6,7,8,9,10,11,13,
   15,17,19,23,27,31,35,43,51,59,
   67,83,99,115,131,163,195,227,258,0,0 };

// 定义静态常量数组 stbi__zlength_extra
static const int stbi__zlength_extra[31]=
{ 0,0,0,0,0,0,0,0,1,1,1,1,2,2,2,2,3,3,3,3,4,4,4,4,5,5,5,5,0,0,0 };

// 定义静态常量数组 stbi__zdist_base
static const int stbi__zdist_base[32] = { 1,2,3,4,5,7,9,13,17,25,33,49,65,97,129,193,
257,385,513,769,1025,1537,2049,3073,4097,6145,8193,12289,16385,24577,0,0};

// 定义静态常量数组 stbi__zdist_extra
static const int stbi__zdist_extra[32] =
{ 0,0,0,0,1,1,2,2,3,3,4,4,5,5,6,6,7,7,8,8,9,9,10,10,11,11,12,12,13,13};

// 定义函数 stbi__parse_huffman_block，参数为 stbi__zbuf 结构体指针 a
static int stbi__parse_huffman_block(stbi__zbuf *a)
{
   // 定义指向字符的指针 zout，指向输入参数 a 的 zout 字段
   char *zout = a->zout;
   // 无限循环，解压缩数据
   for(;;) {
      // 从 huffman 编码中解码一个值，存储在 z 中
      int z = stbi__zhuffman_decode(a, &a->z_length);
      // 如果 z 小于 256，表示为字节数据
      if (z < 256) {
         // 如果 z 小于 0，返回错误信息
         if (z < 0) return stbi__err("bad huffman code","Corrupt PNG"); // error in huffman codes
         // 如果 zout 超出了 zout_end，扩展输出缓冲区
         if (zout >= a->zout_end) {
            if (!stbi__zexpand(a, zout, 1)) return 0;
            zout = a->zout;
         }
         // 将 z 转换为字符类型，存储到 zout 中
         *zout++ = (char) z;
      } else {
         // 定义指向无符号字符的指针 p，以及长度 len 和距离 dist
         stbi_uc *p;
         int len,dist;
         // 如果 z 为 256，表示结束，返回 1
         if (z == 256) {
            a->zout = zout;
            return 1;
         }
         // 如果 z 大于等于 286，返回错误信息
         if (z >= 286) return stbi__err("bad huffman code","Corrupt PNG"); // per DEFLATE, length codes 286 and 287 must not appear in compressed data
         // 计算长度值
         z -= 257;
         len = stbi__zlength_base[z];
         if (stbi__zlength_extra[z]) len += stbi__zreceive(a, stbi__zlength_extra[z]);
         // 从 huffman 编码中解码一个值，存储在 z 中
         z = stbi__zhuffman_decode(a, &a->z_distance);
         // 如果 z 小于 0 或者大于等于 30，返回错误信息
         if (z < 0 || z >= 30) return stbi__err("bad huffman code","Corrupt PNG"); // per DEFLATE, distance codes 30 and 31 must not appear in compressed data
         // 计算距离值
         dist = stbi__zdist_base[z];
         if (stbi__zdist_extra[z]) dist += stbi__zreceive(a, stbi__zdist_extra[z]);
         // 如果距离大于当前输出缓冲区的长度，返回错误信息
         if (zout - a->zout_start < dist) return stbi__err("bad dist","Corrupt PNG");
         // 如果输出缓冲区不足以存储 len 长度的数据，扩展输出缓冲区
         if (zout + len > a->zout_end) {
            if (!stbi__zexpand(a, zout, len)) return 0;
            zout = a->zout;
         }
         // 指向距离 dist 的指针 p
         p = (stbi_uc *) (zout - dist);
         // 如果距离为 1，表示连续一个字节的情况
         if (dist == 1) { // run of one byte; common in images.
            // 获取 p 指向的值 v，重复 len 次将 v 存储到 zout 中
            stbi_uc v = *p;
            if (len) { do *zout++ = v; while (--len); }
         } else {
            // 将 p 指向的值存储到 zout 中，重复 len 次
            if (len) { do *zout++ = *p++; while (--len); }
         }
      }
   }
}

// 计算 huffman 编码
static int stbi__compute_huffman_codes(stbi__zbuf *a)
{
   // 长度解码表，用于解码长度信息
   static const stbi_uc length_dezigzag[19] = { 16,17,18,0,8,7,9,6,10,5,11,4,12,3,13,2,14,1,15 };
   // 存储长度编码的哈夫曼树
   stbi__zhuffman z_codelength;
   // 存储长度编码的数组，包括长度编码、距离编码和额外的长度编码
   stbi_uc lencodes[286+32+137];//padding for maximum single op
   // 存储长度编码的大小
   stbi_uc codelength_sizes[19];
   int i,n;

   // 读取 HLIT 和 HDIST 的值
   int hlit  = stbi__zreceive(a,5) + 257;
   int hdist = stbi__zreceive(a,5) + 1;
   int hclen = stbi__zreceive(a,4) + 4;
   int ntot  = hlit + hdist;

   // 初始化长度编码大小数组
   memset(codelength_sizes, 0, sizeof(codelength_sizes));
   // 为每个长度编码设置大小
   for (i=0; i < hclen; ++i) {
      int s = stbi__zreceive(a,3);
      codelength_sizes[length_dezigzag[i]] = (stbi_uc) s;
   }
   // 构建长度编码的哈夫曼树
   if (!stbi__zbuild_huffman(&z_codelength, codelength_sizes, 19)) return 0;

   n = 0;
   // 解码长度信息
   while (n < ntot) {
      int c = stbi__zhuffman_decode(a, &z_codelength);
      if (c < 0 || c >= 19) return stbi__err("bad codelengths", "Corrupt PNG");
      if (c < 16)
         lencodes[n++] = (stbi_uc) c;
      else {
         stbi_uc fill = 0;
         if (c == 16) {
            c = stbi__zreceive(a,2)+3;
            if (n == 0) return stbi__err("bad codelengths", "Corrupt PNG");
            fill = lencodes[n-1];
         } else if (c == 17) {
            c = stbi__zreceive(a,3)+3;
         } else if (c == 18) {
            c = stbi__zreceive(a,7)+11;
         } else {
            return stbi__err("bad codelengths", "Corrupt PNG");
         }
         if (ntot - n < c) return stbi__err("bad codelengths", "Corrupt PNG");
         memset(lencodes+n, fill, c);
         n += c;
      }
   }
   // 检查解码的长度信息是否与总长度相等
   if (n != ntot) return stbi__err("bad codelengths","Corrupt PNG");
   // 构建长度编码和距离编码的哈夫曼树
   if (!stbi__zbuild_huffman(&a->z_length, lencodes, hlit)) return 0;
   if (!stbi__zbuild_huffman(&a->z_distance, lencodes+hlit, hdist)) return 0;
   return 1;
}

// 解析未压缩的数据块
static int stbi__parse_uncompressed_block(stbi__zbuf *a)
{
   // 定义一个长度为4的无符号字符数组header，用于存储数据头部信息
   stbi_uc header[4];
   // 定义变量len、nlen、k
   int len,nlen,k;
   // 如果当前比特位数不是8的倍数，则丢弃多余的比特位
   if (a->num_bits & 7)
      stbi__zreceive(a, a->num_bits & 7); // 丢弃
   // 将比特压缩的数据填充到header数组中
   k = 0;
   while (a->num_bits > 0) {
      // 将code_buffer中的低8位存入header数组
      header[k++] = (stbi_uc) (a->code_buffer & 255); // 抑制 MSVC 运行时检查
      // 右移8位，丢弃已经存入header数组的8位
      a->code_buffer >>= 8;
      // 减去已经存入header数组的8位
      a->num_bits -= 8;
   }
   // 如果比特位数小于0，表示数据损坏
   if (a->num_bits < 0) return stbi__err("zlib corrupt","Corrupt PNG");
   // 填充header数组剩余的位置
   while (k < 4)
      header[k++] = stbi__zget8(a);
   // 计算数据长度和反码长度
   len  = header[1] * 256 + header[0];
   nlen = header[3] * 256 + header[2];
   // 检查反码长度是否与数据长度的反码相等
   if (nlen != (len ^ 0xffff)) return stbi__err("zlib corrupt","Corrupt PNG");
   // 检查是否读取数据超出缓冲区
   if (a->zbuffer + len > a->zbuffer_end) return stbi__err("read past buffer","Corrupt PNG");
   // 如果输出缓冲区不足，进行扩展
   if (a->zout + len > a->zout_end)
      if (!stbi__zexpand(a, a->zout, len)) return 0;
   // 将数据从输入缓冲区复制到输出缓冲区
   memcpy(a->zout, a->zbuffer, len);
   // 更新输入缓冲区和输出缓冲区的指针位置
   a->zbuffer += len;
   a->zout += len;
   // 返回成功标志
   return 1;
}

// 解析zlib头部信息
static int stbi__parse_zlib_header(stbi__zbuf *a)
{
   // 读取cmf和flg字段
   int cmf   = stbi__zget8(a);
   int cm    = cmf & 15;
   /* int cinfo = cmf >> 4; */
   int flg   = stbi__zget8(a);
   // 如果到达文件末尾，返回错误
   if (stbi__zeof(a)) return stbi__err("bad zlib header","Corrupt PNG"); // zlib spec
   // 检查cmf和flg字段的校验和是否为31的倍数
   if ((cmf*256+flg) % 31 != 0) return stbi__err("bad zlib header","Corrupt PNG"); // zlib spec
   // 检查是否存在预设字典
   if (flg & 32) return stbi__err("no preset dict","Corrupt PNG"); // preset dictionary not allowed in png
   // 检查压缩方法是否为DEFLATE
   if (cm != 8) return stbi__err("bad compression","Corrupt PNG"); // DEFLATE required for png
   // 返回成功标志
   // 窗口大小 = 1 << (8 + cinfo)... 但是我们完全缓冲输出，所以不需要关心
   return 1;
}

// 默认长度数组
static const stbi_uc stbi__zdefault_length[STBI__ZNSYMS] =
// 长度编码表，根据规范初始化
{
   // 初始化长度编码表，范围为 0 到 143，长度为 8
   int i;   // use <= to match clearly with spec
   for (i=0; i <= 143; ++i)     stbi__zdefault_length[i]   = 8;
   // 初始化长度编码表，范围为 144 到 255，长度为 9
   for (   ; i <= 255; ++i)     stbi__zdefault_length[i]   = 9;
   // 初始化长度编码表，范围为 256 到 279，长度为 7
   for (   ; i <= 279; ++i)     stbi__zdefault_length[i]   = 7;
   // 初始化长度编码表，范围为 280 到 287，长度为 8
   for (   ; i <= 287; ++i)     stbi__zdefault_length[i]   = 8;

   // 初始化距离编码表，范围为 0 到 31，距离为 5
   for (i=0; i <=  31; ++i)     stbi__zdefault_distance[i] = 5;
}
*/

// 解析 zlib 数据
static int stbi__parse_zlib(stbi__zbuf *a, int parse_header)
{
   // 定义变量 final 和 type，用于存储解压缩过程中的最终标志和类型
   int final, type;
   // 如果需要解析头部信息
   if (parse_header)
      // 解析 zlib 头部信息，如果失败则返回 0
      if (!stbi__parse_zlib_header(a)) return 0;
   // 初始化位数为 0
   a->num_bits = 0;
   // 初始化代码缓冲区为 0
   a->code_buffer = 0;
   // 开始循环解压缩过程
   do {
      // 读取最终标志
      final = stbi__zreceive(a,1);
      // 读取类型
      type = stbi__zreceive(a,2);
      // 如果类型为 0
      if (type == 0) {
         // 解析非压缩块，如果失败则返回 0
         if (!stbi__parse_uncompressed_block(a)) return 0;
      } else if (type == 3) {
         // 如果类型为 3，直接返回 0
         return 0;
      } else {
         // 如果类型为 1
         if (type == 1) {
            // 使用固定的码长构建哈夫曼树
            if (!stbi__zbuild_huffman(&a->z_length  , stbi__zdefault_length  , STBI__ZNSYMS)) return 0;
            if (!stbi__zbuild_huffman(&a->z_distance, stbi__zdefault_distance,  32)) return 0;
         } else {
            // 计算哈夫曼编码
            if (!stbi__compute_huffman_codes(a)) return 0;
         }
         // 解析哈夫曼块，如果失败则返回 0
         if (!stbi__parse_huffman_block(a)) return 0;
      }
   } while (!final);
   // 解压缩过程完成，返回 1
   return 1;
}

// 执行 zlib 解压缩
static int stbi__do_zlib(stbi__zbuf *a, char *obuf, int olen, int exp, int parse_header)
{
   // 设置输出缓冲区的起始位置
   a->zout_start = obuf;
   // 设置输出缓冲区的当前位置
   a->zout       = obuf;
   // 设置输出缓冲区的结束位置
   a->zout_end   = obuf + olen;
   // 设置是否可扩展标志
   a->z_expandable = exp;

   // 调用解析 zlib 函数，返回解压缩结果
   return stbi__parse_zlib(a, parse_header);
}

// 根据给定的缓冲区和长度进行 zlib 解码，并返回解码后的数据
STBIDEF char *stbi_zlib_decode_malloc_guesssize(const char *buffer, int len, int initial_size, int *outlen)
{
   // 初始化 zlib 缓冲区
   stbi__zbuf a;
   // 分配初始大小的内存
   char *p = (char *) stbi__malloc(initial_size);
   // 如果内存分配失败，直接返回 NULL
   if (p == NULL) return NULL;
   // 设置 zlib 缓冲区的起始和结束位置
   a.zbuffer = (stbi_uc *) buffer;
   a.zbuffer_end = (stbi_uc *) buffer + len;
   // 执行 zlib 解压缩过程，返回解压缩结果
   if (stbi__do_zlib(&a, p, initial_size, 1, 1)) {
      // 如果需要输出解压后的长度，设置输出长度
      if (outlen) *outlen = (int) (a.zout - a.zout_start);
      // 返回解压后的数据起始位置
      return a.zout_start;
   } else {
      // 解压缩失败，释放内存并返回 NULL
      STBI_FREE(a.zout_start);
      return NULL;
   }
}

// 根据给定的缓冲区和长度进行 zlib 解码，并返回解码后的数据
STBIDEF char *stbi_zlib_decode_malloc(char const *buffer, int len, int *outlen)
{
   // 调用带有初始大小猜测的 zlib 解码函数
   return stbi_zlib_decode_malloc_guesssize(buffer, len, 16384, outlen);
}

// 根据给定的缓冲区和长度进行 zlib 解码，并返回解码后的数据
STBIDEF char *stbi_zlib_decode_malloc_guesssize_headerflag(const char *buffer, int len, int initial_size, int *outlen, int parse_header)
{
   // 定义一个 stbi__zbuf 结构体变量 a
   stbi__zbuf a;
   // 分配内存给指针 p，大小为 initial_size
   char *p = (char *) stbi__malloc(initial_size);
   // 如果分配内存失败，则返回空指针
   if (p == NULL) return NULL;
   // 将 buffer 转换为 stbi_uc 类型，并赋值给 a.zbuffer
   a.zbuffer = (stbi_uc *) buffer;
   // 计算 buffer 的结束位置，并赋值给 a.zbuffer_end
   a.zbuffer_end = (stbi_uc *) buffer + len;
   // 调用 stbi__do_zlib 函数进行 zlib 解码
   if (stbi__do_zlib(&a, p, initial_size, 1, parse_header)) {
      // 如果 outlen 不为空，则将解码后的长度赋值给 outlen
      if (outlen) *outlen = (int) (a.zout - a.zout_start);
      // 返回解码后的数据起始地址
      return a.zout_start;
   } else {
      // 释放解码后的数据起始地址的内存
      STBI_FREE(a.zout_start);
      // 返回空指针
      return NULL;
   }
}

// 解码 zlib 数据到指定缓冲区
STBIDEF int stbi_zlib_decode_buffer(char *obuffer, int olen, char const *ibuffer, int ilen)
{
   // 定义一个 stbi__zbuf 结构体变量 a
   stbi__zbuf a;
   // 将 ibuffer 转换为 stbi_uc 类型，并赋值给 a.zbuffer
   a.zbuffer = (stbi_uc *) ibuffer;
   // 计算 ibuffer 的结束位置，并赋值给 a.zbuffer_end
   a.zbuffer_end = (stbi_uc *) ibuffer + ilen;
   // 调用 stbi__do_zlib 函数进行 zlib 解码
   if (stbi__do_zlib(&a, obuffer, olen, 0, 1))
      // 返回解码后的数据长度
      return (int) (a.zout - a.zout_start);
   else
      // 返回 -1 表示解码失败
      return -1;
}

// 解码 zlib 数据到新分配的内存中
STBIDEF char *stbi_zlib_decode_noheader_malloc(char const *buffer, int len, int *outlen)
{
   // 定义一个 stbi__zbuf 结构体变量 a
   stbi__zbuf a;
   // 分配内存给指针 p，大小为 16384
   char *p = (char *) stbi__malloc(16384);
   // 如果分配内存失败，则返回空指针
   if (p == NULL) return NULL;
   // 将 buffer 转换为 stbi_uc 类型，并赋值给 a.zbuffer
   a.zbuffer = (stbi_uc *) buffer;
   // 计算 buffer 的结束位置，并赋值给 a.zbuffer_end
   a.zbuffer_end = (stbi_uc *) buffer+len;
   // 调用 stbi__do_zlib 函数进行 zlib 解码
   if (stbi__do_zlib(&a, p, 16384, 1, 0)) {
      // 如果 outlen 不为空，则将解码后的长度赋值给 outlen
      if (outlen) *outlen = (int) (a.zout - a.zout_start);
      // 返回解码后的数据起始地址
      return a.zout_start;
   } else {
      // 释放解码后的数据起始地址的内存
      STBI_FREE(a.zout_start);
      // 返回空指针
      return NULL;
   }
}

// 解码 zlib 数据到指定缓冲区，不包含头信息
STBIDEF int stbi_zlib_decode_noheader_buffer(char *obuffer, int olen, const char *ibuffer, int ilen)
{
   // 定义一个 stbi__zbuf 结构体变量 a
   stbi__zbuf a;
   // 将 ibuffer 转换为 stbi_uc 类型，并赋值给 a.zbuffer
   a.zbuffer = (stbi_uc *) ibuffer;
   // 计算 ibuffer 的结束位置，并赋值给 a.zbuffer_end
   a.zbuffer_end = (stbi_uc *) ibuffer + ilen;
   // 调用 stbi__do_zlib 函数进行 zlib 解码
   if (stbi__do_zlib(&a, obuffer, olen, 0, 0))
      // 返回解码后的数据长度
      return (int) (a.zout - a.zout_start);
   else
      // 返回 -1 表示解码失败
      return -1;
}
#endif

// PNG 解码器的基线实现，简单实现，只支持 8 位样本，没有 CRC 检查
// 分配大量中间内存，避免在子系统之间传输数据的问题，避免显式窗口管理
// 使用 stb_zlib，一个具有快速哈夫曼解码的公共领域 zlib 实现
// 定义 PNG 数据块结构体
typedef struct
{
   int length; // 数据块长度
   int type;   // 数据块类型
} stbi__pngchunk;

// 获取 PNG 数据块头信息
static stbi__pngchunk stbi__get_chunk_header(stbi__context *s)
{
   stbi__pngchunk c;
   c.length = stbi__get32be(s); // 读取数据块长度
   c.type   = stbi__get32be(s); // 读取数据块类型
   return c;
}

// 检查 PNG 文件头
static int stbi__check_png_header(stbi__context *s)
{
   static const stbi_uc png_sig[8] = { 137,80,78,71,13,10,26,10 }; // PNG 文件头签名
   int i;
   for (i=0; i < 8; ++i)
      if (stbi__get8(s) != png_sig[i]) return stbi__err("bad png sig","Not a PNG"); // 检查文件头签名是否正确
   return 1;
}

// 定义 PNG 数据结构体
typedef struct
{
   stbi__context *s; // 上下文
   stbi_uc *idata, *expanded, *out; // 数据指针
   int depth; // 深度
} stbi__png;

// PNG 滤波器类型枚举
enum {
   STBI__F_none=0,
   STBI__F_sub=1,
   STBI__F_up=2,
   STBI__F_avg=3,
   STBI__F_paeth=4,
   // 用于第一行扫描线的合成滤波器，避免需要一个全为0的虚拟行
   STBI__F_avg_first,
   STBI__F_paeth_first
};

// 第一行滤波器数组
static stbi_uc first_row_filter[5] =
{
   STBI__F_none,
   STBI__F_sub,
   STBI__F_none,
   STBI__F_avg_first,
   STBI__F_paeth_first
};

// Paeth 滤波器函数
static int stbi__paeth(int a, int b, int c)
{
   int p = a + b - c;
   int pa = abs(p-a);
   int pb = abs(p-b);
   int pc = abs(p-c);
   if (pa <= pb && pa <= pc) return a;
   if (pb <= pc) return b;
   return c;
}

// 深度缩放表
static const stbi_uc stbi__depth_scale_table[9] = { 0, 0xff, 0x55, 0, 0x11, 0,0,0, 0x01 };

// 从解压后的数据创建 PNG 图像
static int stbi__create_png_image_raw(stbi__png *a, stbi_uc *raw, stbi__uint32 raw_len, int out_n, stbi__uint32 x, stbi__uint32 y, int depth, int color)
}

// 创建 PNG 图像
static int stbi__create_png_image(stbi__png *a, stbi_uc *image_data, stbi__uint32 image_data_len, int out_n, int depth, int color, int interlaced)
{
   // 根据深度确定每个像素占用的字节数
   int bytes = (depth == 16 ? 2 : 1);
   // 计算输出数据的总字节数
   int out_bytes = out_n * bytes;
   // 定义最终输出的数据指针
   stbi_uc *final;
   // 定义循环变量 p
   int p;
   // 如果图像没有交错，则直接创建 PNG 图像数据并返回
   if (!interlaced)
      return stbi__create_png_image_raw(a, image_data, image_data_len, out_n, a->s->img_x, a->s->img_y, depth, color);

   // 对图像进行去交错处理
   final = (stbi_uc *) stbi__malloc_mad3(a->s->img_x, a->s->img_y, out_bytes, 0);
   // 如果内存分配失败，则返回错误信息
   if (!final) return stbi__err("outofmem", "Out of memory");
   // 遍历七个子图像
   for (p=0; p < 7; ++p) {
      // 定义子图像的原点和间隔
      int xorig[] = { 0,4,0,2,0,1,0 };
      int yorig[] = { 0,0,4,0,2,0,1 };
      int xspc[]  = { 8,8,4,4,2,2,1 };
      int yspc[]  = { 8,8,8,4,4,2,2 };
      int i,j,x,y;
      // 计算子图像的宽度和高度
      x = (a->s->img_x - xorig[p] + xspc[p]-1) / xspc[p];
      y = (a->s->img_y - yorig[p] + yspc[p]-1) / yspc[p];
      // 如果子图像宽度和高度大于0
      if (x && y) {
         // 计算子图像数据长度
         stbi__uint32 img_len = ((((a->s->img_n * x * depth) + 7) >> 3) + 1) * y;
         // 创建 PNG 图像数据并返回
         if (!stbi__create_png_image_raw(a, image_data, image_data_len, out_n, x, y, depth, color)) {
            STBI_FREE(final);
            return 0;
         }
         // 将子图像数据复制到最终输出数据中
         for (j=0; j < y; ++j) {
            for (i=0; i < x; ++i) {
               int out_y = j*yspc[p]+yorig[p];
               int out_x = i*xspc[p]+xorig[p];
               memcpy(final + out_y*a->s->img_x*out_bytes + out_x*out_bytes,
                      a->out + (j*x+i)*out_bytes, out_bytes);
            }
         }
         // 释放临时数据内存
         STBI_FREE(a->out);
         // 更新图像数据指针和长度
         image_data += img_len;
         image_data_len -= img_len;
      }
   }
   // 将最终输出数据指针赋值给 a->out
   a->out = final;

   return 1;
}

// 计算透明度信息
static int stbi__compute_transparency(stbi__png *z, stbi_uc tc[3], int out_n)
{
   // 获取输入的上下文
   stbi__context *s = z->s;
   // 初始化变量 i 和像素数量
   stbi__uint32 i, pixel_count = s->img_x * s->img_y;
   // 获取输出指针
   stbi_uc *p = z->out;

   // 计算基于颜色的透明度，假设输出中 alpha 值已经是 255
   STBI_ASSERT(out_n == 2 || out_n == 4);

   // 如果输出通道数为 2
   if (out_n == 2) {
      // 遍历每个像素
      for (i=0; i < pixel_count; ++i) {
         // 如果当前像素的第一个通道值等于指定颜色的第一个通道值，则将第二个通道值设为 0，否则设为 255
         p[1] = (p[0] == tc[0] ? 0 : 255);
         // 移动指针到下一个像素
         p += 2;
      }
   } else {
      // 如果输出通道数不为 2
      for (i=0; i < pixel_count; ++i) {
         // 如果当前像素的 RGB 值与指定颜色相同，则将 alpha 值设为 0
         if (p[0] == tc[0] && p[1] == tc[1] && p[2] == tc[2])
            p[3] = 0;
         // 移动指针到下一个像素
         p += 4;
      }
   }
   // 返回 1 表示成功
   return 1;
}

// 计算 16 位 PNG 图像的透明度
static int stbi__compute_transparency16(stbi__png *z, stbi__uint16 tc[3], int out_n)
{
   // 获取输入的上下文
   stbi__context *s = z->s;
   // 初始化变量 i 和像素数量
   stbi__uint32 i, pixel_count = s->img_x * s->img_y;
   // 获取输出指针
   stbi__uint16 *p = (stbi__uint16*) z->out;

   // 计算基于颜色的透明度，假设输出中 alpha 值已经是 65535
   STBI_ASSERT(out_n == 2 || out_n == 4);

   // 如果输出通道数为 2
   if (out_n == 2) {
      // 遍历每个像素
      for (i = 0; i < pixel_count; ++i) {
         // 如果当前像素的第一个通道值等于指定颜色的第一个通道值，则将第二个通道值设为 0，否则设为 65535
         p[1] = (p[0] == tc[0] ? 0 : 65535);
         // 移动指针到下一个像素
         p += 2;
      }
   } else {
      // 如果输出通道数不为 2
      for (i = 0; i < pixel_count; ++i) {
         // 如果当前像素的 RGB 值与指定颜色相同，则将 alpha 值设为 0
         if (p[0] == tc[0] && p[1] == tc[1] && p[2] == tc[2])
            p[3] = 0;
         // 移动指针到下一个像素
         p += 4;
      }
   }
   // 返回 1 表示成功
   return 1;
}

// 扩展 PNG 调色板
static int stbi__expand_png_palette(stbi__png *a, stbi_uc *palette, int len, int pal_img_n)
{
   // 定义变量 i 为循环计数器，pixel_count 为图像像素总数
   stbi__uint32 i, pixel_count = a->s->img_x * a->s->img_y;
   // 定义指针变量 p, temp_out, orig 指向 a->out
   stbi_uc *p, *temp_out, *orig = a->out;

   // 分配内存给 p，大小为 pixel_count * pal_img_n
   p = (stbi_uc *) stbi__malloc_mad2(pixel_count, pal_img_n, 0);
   // 如果分配内存失败，返回错误信息
   if (p == NULL) return stbi__err("outofmem", "Out of memory");

   // 保存 p 的地址到 temp_out
   temp_out = p;

   // 根据 pal_img_n 的值，将原始像素数据转换为 RGB 或 RGBA 格式
   if (pal_img_n == 3) {
      for (i=0; i < pixel_count; ++i) {
         int n = orig[i]*4;
         p[0] = palette[n  ];
         p[1] = palette[n+1];
         p[2] = palette[n+2];
         p += 3;
      }
   } else {
      for (i=0; i < pixel_count; ++i) {
         int n = orig[i]*4;
         p[0] = palette[n  ];
         p[1] = palette[n+1];
         p[2] = palette[n+2];
         p[3] = palette[n+3];
         p += 4;
      }
   }
   // 释放原始像素数据内存
   STBI_FREE(a->out);
   // 将转换后的像素数据地址赋值给 a->out
   a->out = temp_out;

   // 忽略变量 len

   // 返回成功标志
   return 1;
}

// 设置全局变量 stbi__unpremultiply_on_load_global 的值
static int stbi__unpremultiply_on_load_global = 0;
// 设置全局变量 stbi__de_iphone_flag_global 的值
static int stbi__de_iphone_flag_global = 0;

// 设置是否在加载时取消预乘的标志
STBIDEF void stbi_set_unpremultiply_on_load(int flag_true_if_should_unpremultiply)
{
   stbi__unpremultiply_on_load_global = flag_true_if_should_unpremultiply;
}

// 设置是否将 iPhone PNG 转换为 RGB 的标志
STBIDEF void stbi_convert_iphone_png_to_rgb(int flag_true_if_should_convert)
{
   stbi__de_iphone_flag_global = flag_true_if_should_convert;
}

// 如果未定义 STBI_THREAD_LOCAL，则使用全局变量
#ifndef STBI_THREAD_LOCAL
#define stbi__unpremultiply_on_load  stbi__unpremultiply_on_load_global
#define stbi__de_iphone_flag  stbi__de_iphone_flag_global
// 如果定义了 STBI_THREAD_LOCAL，则使用线程本地变量
#else
static STBI_THREAD_LOCAL int stbi__unpremultiply_on_load_local, stbi__unpremultiply_on_load_set;
static STBI_THREAD_LOCAL int stbi__de_iphone_flag_local, stbi__de_iphone_flag_set;

// 设置线程本地变量 stbi__unpremultiply_on_load_local 的值
STBIDEF void stbi_set_unpremultiply_on_load_thread(int flag_true_if_should_unpremultiply)
{
   stbi__unpremultiply_on_load_local = flag_true_if_should_unpremultiply;
   stbi__unpremultiply_on_load_set = 1;
}

// 设置线程本地变量 stbi__de_iphone_flag_local 的值
STBIDEF void stbi_convert_iphone_png_to_rgb_thread(int flag_true_if_should_convert)
{
   stbi__de_iphone_flag_local = flag_true_if_should_convert;
   stbi__de_iphone_flag_set = 1;
}
}
// 定义 stbi__unpremultiply_on_load 宏，根据设置选择本地或全局的值
#define stbi__unpremultiply_on_load  (stbi__unpremultiply_on_load_set           \
                                       ? stbi__unpremultiply_on_load_local      \
                                       : stbi__unpremultiply_on_load_global)
// 定义 stbi__de_iphone_flag 宏，根据设置选择本地或全局的值
#define stbi__de_iphone_flag  (stbi__de_iphone_flag_set                         \
                                ? stbi__de_iphone_flag_local                    \
                                : stbi__de_iphone_flag_global)
#endif // STBI_THREAD_LOCAL

// 解码 iPhone 图像
static void stbi__de_iphone(stbi__png *z)
{
   // 获取 PNG 上下文
   stbi__context *s = z->s;
   // 初始化变量 i 和像素数量
   stbi__uint32 i, pixel_count = s->img_x * s->img_y;
   // 获取输出像素数据指针
   stbi_uc *p = z->out;

   // 如果输出像素通道数为 3，将 bgr 转换为 rgb
   if (s->img_out_n == 3) {
      for (i=0; i < pixel_count; ++i) {
         // 交换 bgr 通道顺序为 rgb
         stbi_uc t = p[0];
         p[0] = p[2];
         p[2] = t;
         p += 3;
      }
   } else {
      // 断言输出像素通道数为 4
      STBI_ASSERT(s->img_out_n == 4);
      // 如果设置了解码前取消预乘标志
      if (stbi__unpremultiply_on_load) {
         // 将 bgr 转换为 rgb 并取消预乘
         for (i=0; i < pixel_count; ++i) {
            stbi_uc a = p[3];
            stbi_uc t = p[0];
            if (a) {
               stbi_uc half = a / 2;
               p[0] = (p[2] * 255 + half) / a;
               p[1] = (p[1] * 255 + half) / a;
               p[2] = ( t   * 255 + half) / a;
            } else {
               p[0] = p[2];
               p[2] = t;
            }
            p += 4;
         }
      } else {
         // 将 bgr 转换为 rgb
         for (i=0; i < pixel_count; ++i) {
            stbi_uc t = p[0];
            p[0] = p[2];
            p[2] = t;
            p += 4;
         }
      }
   }
}

// 定义 PNG 类型宏
#define STBI__PNG_TYPE(a,b,c,d)  (((unsigned) (a) << 24) + ((unsigned) (b) << 16) + ((unsigned) (c) << 8) + (unsigned) (d))

// 解析 PNG 文件
static int stbi__parse_png_file(stbi__png *z, int scan, int req_comp)
}

// 执行 PNG 解码
static void *stbi__do_png(stbi__png *p, int *x, int *y, int *n, int req_comp, stbi__result_info *ri)
{
   // 定义一个指针变量 result，初始化为 NULL
   void *result=NULL;
   // 如果请求的通道数小于 0 或大于 4，则返回错误信息
   if (req_comp < 0 || req_comp > 4) return stbi__errpuc("bad req_comp", "Internal error");
   // 解析 PNG 文件，加载数据到内存
   if (stbi__parse_png_file(p, STBI__SCAN_load, req_comp)) {
      // 如果像素深度小于等于 8，则设置每通道位数为 8
      if (p->depth <= 8)
         ri->bits_per_channel = 8;
      // 如果像素深度为 16，则设置每通道位数为 16
      else if (p->depth == 16)
         ri->bits_per_channel = 16;
      // 否则返回错误信息，不支持该颜色深度
      else
         return stbi__errpuc("bad bits_per_channel", "PNG not supported: unsupported color depth");
      // 将解析后的数据赋值给 result，并清空 p->out
      result = p->out;
      p->out = NULL;
      // 如果请求的通道数不为 0 且不等于解析后的通道数，则进行格式转换
      if (req_comp && req_comp != p->s->img_out_n) {
         // 如果每通道位数为 8，则调用 stbi__convert_format 进行格式转换
         if (ri->bits_per_channel == 8)
            result = stbi__convert_format((unsigned char *) result, p->s->img_out_n, req_comp, p->s->img_x, p->s->img_y);
         // 如果每通道位数为 16，则调用 stbi__convert_format16 进行格式转换
         else
            result = stbi__convert_format16((stbi__uint16 *) result, p->s->img_out_n, req_comp, p->s->img_x, p->s->img_y);
         // 更新解析后的通道数为请求的通道数
         p->s->img_out_n = req_comp;
         // 如果转换结果为 NULL，则返回结果
         if (result == NULL) return result;
      }
      // 将图片的宽度赋值给 x，高度赋值给 y
      *x = p->s->img_x;
      *y = p->s->img_y;
      // 如果 comp 不为空，则将解析后的通道数赋值给 n
      if (n) *n = p->s->img_n;
   }
   // 释放 p->out、p->expanded、p->idata 的内存空间
   STBI_FREE(p->out);      p->out      = NULL;
   STBI_FREE(p->expanded); p->expanded = NULL;
   STBI_FREE(p->idata);    p->idata    = NULL;

   // 返回解析后的结果
   return result;
}

// 加载 PNG 图像数据
static void *stbi__png_load(stbi__context *s, int *x, int *y, int *comp, int req_comp, stbi__result_info *ri)
{
   // 创建 stbi__png 结构体 p，并将输入的上下文赋值给 p.s
   stbi__png p;
   p.s = s;
   // 调用 stbi__do_png 函数进行 PNG 图像加载
   return stbi__do_png(&p, x,y,comp,req_comp, ri);
}

// 检测 PNG 图像格式
static int stbi__png_test(stbi__context *s)
{
   int r;
   // 检查 PNG 文件头部信息
   r = stbi__check_png_header(s);
   // 将文件指针重置到文件开头
   stbi__rewind(s);
   // 返回检测结果
   return r;
}

// 获取 PNG 图像信息（不包含像素数据）
static int stbi__png_info_raw(stbi__png *p, int *x, int *y, int *comp)
{
   // 解析 PNG 文件头部信息
   if (!stbi__parse_png_file(p, STBI__SCAN_header, 0)) {
      // 如果解析失败，则将文件指针重置到文件开头
      stbi__rewind( p->s );
      return 0;
   }
   // 如果 x 不为空，则将图片宽度赋值给 x
   if (x) *x = p->s->img_x;
   // 如果 y 不为空，则将图片高度赋值给 y
   if (y) *y = p->s->img_y;
   // 如果 comp 不为空，则将通道数赋值给 comp
   if (comp) *comp = p->s->img_n;
   return 1;
}

// 获取 PNG 图像信息（包含像素数据）
static int stbi__png_info(stbi__context *s, int *x, int *y, int *comp)
{
   // 创建 stbi__png 结构体 p，并将输入的上下文赋值给 p.s
   stbi__png p;
   p.s = s;
   // 调用 stbi__png_info_raw 函数获取 PNG 图像信息
   return stbi__png_info_raw(&p, x, y, comp);
}

// 检测 PNG 图像是否为 16 位深度
static int stbi__png_is16(stbi__context *s)
{
   // 定义一个 stbi__png 结构体变量 p
   stbi__png p;
   // 将参数 s 赋值给结构体变量 p 的成员变量 s
   p.s = s;
   // 如果无法获取 PNG 图像的原始信息，则返回 0
   if (!stbi__png_info_raw(&p, NULL, NULL, NULL))
       return 0;
   // 如果 PNG 图像的深度不是 16 位，则将文件指针回退并返回 0
   if (p.depth != 16) {
      stbi__rewind(p.s);
      return 0;
   }
   // 返回 1
   return 1;
}
#endif

// Microsoft/Windows BMP image

#ifndef STBI_NO_BMP
// 检测 BMP 图像的原始信息
static int stbi__bmp_test_raw(stbi__context *s)
{
   int r;
   int sz;
   // 检查文件头是否为 'BM'
   if (stbi__get8(s) != 'B') return 0;
   if (stbi__get8(s) != 'M') return 0;
   // 跳过文件大小字段
   stbi__get32le(s); // discard filesize
   // 跳过保留字段
   stbi__get16le(s); // discard reserved
   stbi__get16le(s); // discard reserved
   // 跳过数据偏移字段
   stbi__get32le(s); // discard data offset
   // 获取图像信息头的大小
   sz = stbi__get32le(s);
   // 判断图像信息头的大小是否为合法值
   r = (sz == 12 || sz == 40 || sz == 56 || sz == 108 || sz == 124);
   return r;
}

// 检测 BMP 图像
static int stbi__bmp_test(stbi__context *s)
{
   // 调用 stbi__bmp_test_raw 函数检测 BMP 图像
   int r = stbi__bmp_test_raw(s);
   // 将文件指针回退到起始位置
   stbi__rewind(s);
   return r;
}

// 返回 z 中最高位的索引值（0 到 31）
static int stbi__high_bit(unsigned int z)
{
   int n=0;
   // 如果 z 为 0，则返回 -1
   if (z == 0) return -1;
   // 通过位运算计算最高位的索引值
   if (z >= 0x10000) { n += 16; z >>= 16; }
   if (z >= 0x00100) { n +=  8; z >>=  8; }
   if (z >= 0x00010) { n +=  4; z >>=  4; }
   if (z >= 0x00004) { n +=  2; z >>=  2; }
   if (z >= 0x00002) { n +=  1;/* >>=  1;*/ }
   return n;
}

// 计算 a 中位为 1 的个数
static int stbi__bitcount(unsigned int a)
{
   // 通过位运算计算 a 中位为 1 的个数
   a = (a & 0x55555555) + ((a >>  1) & 0x55555555); // max 2
   a = (a & 0x33333333) + ((a >>  2) & 0x33333333); // max 4
   a = (a + (a >> 4)) & 0x0f0f0f0f; // max 8 per 4, now 8 bits
   a = (a + (a >> 8)); // max 16 per 8 bits
   a = (a + (a >> 16)); // max 32 per 8 bits
   return a & 0xff;
}

// 从 v 中提取 N 位的值（N=bits），并将其扩展为 8 位
static int stbi__shiftsigned(unsigned int v, int shift, int bits)
{
   // 预先计算的乘法表，用于位操作
   static unsigned int mul_table[9] = {
      0,
      0xff/*0b11111111*/, 0x55/*0b01010101*/, 0x49/*0b01001001*/, 0x11/*0b00010001*/,
      0x21/*0b00100001*/, 0x41/*0b01000001*/, 0x81/*0b10000001*/, 0x01/*0b00000001*/,
   };
   // 预先计算的位移表，用于位操作
   static unsigned int shift_table[9] = {
      0, 0,0,1,0,2,4,6,0,
   };
   // 根据位移量对值进行左移或右移
   if (shift < 0)
      v <<= -shift;
   else
      v >>= shift;
   // 断言值小于256
   STBI_ASSERT(v < 256);
   // 对值进行右移，保留指定位数
   v >>= (8-bits);
   // 断言位数在0到8之间
   STBI_ASSERT(bits >= 0 && bits <= 8);
   // 返回经过乘法和位移后的值
   return (int) ((unsigned) v * mul_table[bits]) >> shift_table[bits];
}

typedef struct
{
   int bpp, offset, hsz;
   unsigned int mr,mg,mb,ma, all_a;
   int extra_read;
} stbi__bmp_data;

static int stbi__bmp_set_mask_defaults(stbi__bmp_data *info, int compress)
{
   // 如果压缩类型为3，表示BI_BITFIELDS指定了掩码，不需要覆盖
   if (compress == 3)
      return 1;

   if (compress == 0) {
      // 根据位深度设置默认掩码
      if (info->bpp == 16) {
         info->mr = 31u << 10;
         info->mg = 31u <<  5;
         info->mb = 31u <<  0;
      } else if (info->bpp == 32) {
         info->mr = 0xffu << 16;
         info->mg = 0xffu <<  8;
         info->mb = 0xffu <<  0;
         info->ma = 0xffu << 24;
         info->all_a = 0; // 如果all_a为0，则加载的alpha通道全为0
      } else {
         // 否则使用默认值，全为0
         info->mr = info->mg = info->mb = info->ma = 0;
      }
      return 1;
   }
   return 0; // 错误
}

static void *stbi__bmp_parse_header(stbi__context *s, stbi__bmp_data *info)
}


static void *stbi__bmp_load(stbi__context *s, int *x, int *y, int *comp, int req_comp, stbi__result_info *ri)
}
#endif

// Targa Truevision - TGA
// by Jonathan Dummer
#ifndef STBI_NO_TGA
// 根据像素位数和是否灰度返回通道数，错误返回0
static int stbi__tga_get_comp(int bits_per_pixel, int is_grey, int* is_rgb16)
{
   // 只允许 RGB 或 RGBA（包括 16 位）或灰度图像
   if (is_rgb16) *is_rgb16 = 0;
   switch(bits_per_pixel) {
      case 8:  return STBI_grey;  // 返回灰度图像
      case 16: if(is_grey) return STBI_grey_alpha;  // 如果是灰度图像，返回带 alpha 通道的灰度图像
               // 继续执行下面的代码
      case 15: if(is_rgb16) *is_rgb16 = 1;  // 如果是 16 位 RGB 图像，设置标志位
               return STBI_rgb;  // 返回 RGB 图像
      case 24: // 继续执行下面的代码
      case 32: return bits_per_pixel/8;  // 返回每像素字节数
      default: return 0;  // 默认返回 0
   }
}

static int stbi__tga_info(stbi__context *s, int *x, int *y, int *comp)
{
    int tga_w, tga_h, tga_comp, tga_image_type, tga_bits_per_pixel, tga_colormap_bpp;
    int sz, tga_colormap_type;
    stbi__get8(s);                   // 丢弃 Offset
    tga_colormap_type = stbi__get8(s); // 颜色映射类型
    if( tga_colormap_type > 1 ) {
        stbi__rewind(s);
        return 0;      // 只允许 RGB 或索引颜色映射
    }
    tga_image_type = stbi__get8(s); // 图像类型
    if ( tga_colormap_type == 1 ) { // 使用颜色映射（调色板）的图像
        if (tga_image_type != 1 && tga_image_type != 9) {
            stbi__rewind(s);
            return 0;
        }
        stbi__skip(s,4);       // 跳过第一个颜色映射条目的索引和条目数
        sz = stbi__get8(s);    // 检查每个调色板颜色条目的位数
        if ( (sz != 8) && (sz != 15) && (sz != 16) && (sz != 24) && (sz != 32) ) {
            stbi__rewind(s);
            return 0;
        }
        stbi__skip(s,4);       // 跳过图像 x 和 y 起始位置
        tga_colormap_bpp = sz;
    } else { // 没有颜色映射的“普通”图像 - 只允许 RGB 或灰度图像，可能有 RLE 压缩
        if ( (tga_image_type != 2) && (tga_image_type != 3) && (tga_image_type != 10) && (tga_image_type != 11) ) {
            stbi__rewind(s);
            return 0; // 只允许 RGB 或灰度图像，可能有 RLE 压缩
        }
        stbi__skip(s,9); // 跳过颜色映射规范和图像 x/y 起始位置
        tga_colormap_bpp = 0;
    }
    tga_w = stbi__get16le(s);
    if( tga_w < 1 ) {
        stbi__rewind(s);
        return 0;   // 检查宽度
    }
    # 读取 TGA 文件的高度信息
    tga_h = stbi__get16le(s);
    # 如果高度小于1，则回退到文件开头并返回0，用于测试高度
    if( tga_h < 1 ) {
        stbi__rewind(s);
        return 0;   // test height
    }
    # 读取 TGA 文件的每像素位数
    tga_bits_per_pixel = stbi__get8(s); // bits per pixel
    # 忽略 alpha 位
    stbi__get8(s); // ignore alpha bits
    # 如果使用了颜色映射表
    if (tga_colormap_bpp != 0) {
        # 当使用颜色映射表时，tga_bits_per_pixel 是索引的大小
        # 我认为只有8位或16位索引才有意义
        if((tga_bits_per_pixel != 8) && (tga_bits_per_pixel != 16)) {
            stbi__rewind(s);
            return 0;
        }
        # 获取颜色通道数
        tga_comp = stbi__tga_get_comp(tga_colormap_bpp, 0, NULL);
    } else {
        # 根据每像素位数和图像类型获取颜色通道数
        tga_comp = stbi__tga_get_comp(tga_bits_per_pixel, (tga_image_type == 3) || (tga_image_type == 11), NULL);
    }
    # 如果获取颜色通道数失败，则回退到文件开头并返回0
    if(!tga_comp) {
      stbi__rewind(s);
      return 0;
    }
    # 如果传入了 x 参数，则将 TGA 文件的宽度赋值给 x
    if (x) *x = tga_w;
    # 如果传入了 y 参数，则将 TGA 文件的高度赋值给 y
    if (y) *y = tga_h;
    # 如果传入了 comp 参数，则将颜色通道数赋值给 comp
    if (comp) *comp = tga_comp;
    return 1;                   // seems to have passed everything
// 检查 TGA 文件是否符合要求，返回结果
static int stbi__tga_test(stbi__context *s)
{
   int res = 0; // 初始化结果为 0
   int sz, tga_color_type; // 定义变量 sz 和 tga_color_type
   stbi__get8(s);      //   丢弃 Offset
   tga_color_type = stbi__get8(s);   //   读取颜色类型
   if ( tga_color_type > 1 ) goto errorEnd;   //   只允许 RGB 或索引颜色类型
   sz = stbi__get8(s);   //   读取图像类型
   if ( tga_color_type == 1 ) { // 颜色映射（调色板）图像
      if (sz != 1 && sz != 9) goto errorEnd; // 颜色类型为 1 时，要求图像类型为 1 或 9
      stbi__skip(s,4);       // 跳过第一个颜色映射条目的索引和条目数
      sz = stbi__get8(s);    //   检查每个调色板颜色条目的位数
      if ( (sz != 8) && (sz != 15) && (sz != 16) && (sz != 24) && (sz != 32) ) goto errorEnd;
      stbi__skip(s,4);       // 跳过图像 x 和 y 起始位置
   } else { // 没有颜色映射的“普通”图像
      if ( (sz != 2) && (sz != 3) && (sz != 10) && (sz != 11) ) goto errorEnd; // 只允许 RGB 或灰度，可能带有 RLE 压缩
      stbi__skip(s,9); // 跳过颜色映射规范和图像 x/y 起始位置
   }
   if ( stbi__get16le(s) < 1 ) goto errorEnd;      //   检查宽度
   if ( stbi__get16le(s) < 1 ) goto errorEnd;      //   检查高度
   sz = stbi__get8(s);   //   每像素位数
   if ( (tga_color_type == 1) && (sz != 8) && (sz != 16) ) goto errorEnd; // 对于颜色映射图像，每像素位数是索引的大小
   if ( (sz != 8) && (sz != 15) && (sz != 16) && (sz != 24) && (sz != 32) ) goto errorEnd;

   res = 1; // 如果执行到这里，说明一切正常，返回 1 而不是 0

errorEnd:
   stbi__rewind(s); // 重新定位到起始位置
   return res; // 返回结果
}

// 读取 16 位值并转换为 24 位 RGB
static void stbi__tga_read_rgb16(stbi__context *s, stbi_uc* out)
{
   // 读取两个字节，存储在 px 中
   stbi__uint16 px = (stbi__uint16)stbi__get16le(s);
   // 定义一个掩码，用于提取5位数据
   stbi__uint16 fiveBitMask = 31;
   // 每个通道有5位数据
   int r = (px >> 10) & fiveBitMask;
   int g = (px >> 5) & fiveBitMask;
   int b = px & fiveBitMask;
   // 注意：这里按照 RGB(A) 顺序保存数据，因此后续不需要交换
   out[0] = (stbi_uc)((r * 255)/31);
   out[1] = (stbi_uc)((g * 255)/31);
   out[2] = (stbi_uc)((b * 255)/31);

   // 有些人声称最高位可能用于 alpha 通道
   // （可能是在“图像描述符字节”中设置了 alpha 位）
   // 但这样会使16位测试图像完全透明..
   // 因此，我们将所有15和16位的 TGA 图像视为没有 alpha 通道的 RGB 图像。
}

static void *stbi__tga_load(stbi__context *s, int *x, int *y, int *comp, int req_comp, stbi__result_info *ri)
}
#endif

// *************************************************************************************************
// Photoshop PSD 加载器 -- 由Thatcher Ulrich编写，由Nicolas Schulz整合，由STB调整

#ifndef STBI_NO_PSD
static int stbi__psd_test(stbi__context *s)
{
   // 检查文件头是否为 "8BPS"
   int r = (stbi__get32be(s) == 0x38425053);
   // 将文件指针重置到起始位置
   stbi__rewind(s);
   return r;
}

static int stbi__psd_decode_rle(stbi__context *s, stbi_uc *p, int pixelCount)
}
{
   // 定义变量 count, nleft, len
   int count, nleft, len;

   // 初始化 count 为 0
   count = 0;
   // 当剩余像素数量大于 0 时循环
   while ((nleft = pixelCount - count) > 0) {
      // 读取一个字节作为 len
      len = stbi__get8(s);
      // 如果 len 为 128，则无操作
      if (len == 128) {
         // No-op.
      } else if (len < 128) {
         // 复制接下来的 len+1 个字节
         len++;
         // 如果复制的字节数大于剩余像素数量，返回 0 表示数据损坏
         if (len > nleft) return 0; // corrupt data
         count += len;
         // 循环复制 len 个字节
         while (len) {
            *p = stbi__get8(s);
            p += 4;
            len--;
         }
      } else if (len > 128) {
         stbi_uc   val;
         // 目标中的接下来 -len+1 个字节从源中的下一个字节复制
         // (将 len 解释为负的 8 位整数)
         len = 257 - len;
         // 如果复制的字节数大于剩余像素数量，返回 0 表示数据损坏
         if (len > nleft) return 0; // corrupt data
         val = stbi__get8(s);
         count += len;
         // 循环复制 len 个字节
         while (len) {
            *p = val;
            p += 4;
            len--;
         }
      }
   }

   // 返回 1 表示成功
   return 1;
}

// 解析 Softimage PIC 文件
static void *stbi__psd_load(stbi__context *s, int *x, int *y, int *comp, int req_comp, stbi__result_info *ri, int bpc)
}
#endif

// *************************************************************************************************
// Softimage PIC loader
// by Tom Seddon
//
// See http://softimage.wiki.softimage.com/index.php/INFO:_PIC_file_format
// See http://ozviz.wasp.uwa.edu.au/~pbourke/dataformats/softimagepic/

#ifndef STBI_NO_PIC
// 检查前四个字节是否与给定字符串相同
static int stbi__pic_is4(stbi__context *s,const char *str)
{
   int i;
   for (i=0; i<4; ++i)
      if (stbi__get8(s) != (stbi_uc)str[i])
         return 0;

   return 1;
}

// 测试是否为 Softimage PIC 文件
static int stbi__pic_test_core(stbi__context *s)
{
   int i;

   // 检查文件头是否为指定值
   if (!stbi__pic_is4(s,"\x53\x80\xF6\x34"))
      return 0;

   // 跳过 84 个字节
   for(i=0;i<84;++i)
      stbi__get8(s);

   // 检查文件头是否为 "PICT"
   if (!stbi__pic_is4(s,"PICT"))
      return 0;

   return 1;
}

// 定义 Softimage PIC 数据包结构
typedef struct
{
   stbi_uc size,type,channel;
} stbi__pic_packet;

// 读取通道值
static stbi_uc *stbi__readval(stbi__context *s, int channel, stbi_uc *dest)
{
   // 定义变量 mask 为 0x80，i 为循环计数器
   int mask=0x80, i;

   // 循环4次，每次右移 mask 1位
   for (i=0; i<4; ++i, mask>>=1) {
      // 检查 channel 和 mask 的按位与结果是否为真
      if (channel & mask) {
         // 如果已到达文件末尾，则返回错误信息
         if (stbi__at_eof(s)) return stbi__errpuc("bad file","PIC file too short");
         // 从输入流中读取一个字节，存入 dest 数组中
         dest[i]=stbi__get8(s);
      }
   }

   // 返回 dest 数组
   return dest;
}

static void stbi__copyval(int channel,stbi_uc *dest,const stbi_uc *src)
{
   // 定义变量 mask 为 0x80，i 为循环计数器
   int mask=0x80,i;

   // 循环4次，每次右移 mask 1位
   for (i=0;i<4; ++i, mask>>=1)
      // 检查 channel 和 mask 的按位与结果是否为真
      if (channel&mask)
         // 将 src 数组中的值复制到 dest 数组中
         dest[i]=src[i];
}

static stbi_uc *stbi__pic_load_core(stbi__context *s,int width,int height,int *comp, stbi_uc *result)
{
   // 初始化变量
   int act_comp=0,num_packets=0,y,chained;
   stbi__pic_packet packets[10];

   // 这里将处理一些奇怪的情况，比如数据异常
}

static void *stbi__pic_load(stbi__context *s,int *px,int *py,int *comp,int req_comp, stbi__result_info *ri)
{
   // 定义变量
   stbi_uc *result;
   int i, x,y, internal_comp;
   STBI_NOTUSED(ri);

   // 如果 comp 为空，则使用 internal_comp
   if (!comp) comp = &internal_comp;

   // 跳过92个字节
   for (i=0; i<92; ++i)
      stbi__get8(s);

   // 读取图片的宽度和高度
   x = stbi__get16be(s);
   y = stbi__get16be(s);

   // 检查图片尺寸是否过大
   if (y > STBI_MAX_DIMENSIONS) return stbi__errpuc("too large","Very large image (corrupt?)");
   if (x > STBI_MAX_DIMENSIONS) return stbi__errpuc("too large","Very large image (corrupt?)");

   // 检查文件是否过短
   if (stbi__at_eof(s))  return stbi__errpuc("bad file","file too short (pic header)");
   // 检查图片尺寸是否过大
   if (!stbi__mad3sizes_valid(x, y, 4, 0)) return stbi__errpuc("too large", "PIC image too large to decode");

   // 跳过 ratio、fields、pad 字段
   stbi__get32be(s); //skip `ratio'
   stbi__get16be(s); //skip `fields'
   stbi__get16be(s); //skip `pad'

   // 分配内存存储图片数据，初始化为白色
   result = (stbi_uc *) stbi__malloc_mad3(x, y, 4, 0);
   if (!result) return stbi__errpuc("outofmem", "Out of memory");
   memset(result, 0xff, x*y*4);

   // 加载图片核心数据
   if (!stbi__pic_load_core(s,x,y,comp, result)) {
      STBI_FREE(result);
      result=0;
   }
   *px = x;
   *py = y;
   if (req_comp == 0) req_comp = *comp;
   // 转换图片格式
   result=stbi__convert_format(result,4,req_comp,x,y);

   return result;
}

static int stbi__pic_test(stbi__context *s)
{
   // 调用 stbi__pic_test_core 函数测试输入流 s 是否为 PIC 格式，返回结果
   int r = stbi__pic_test_core(s);
   // 将输入流 s 重置到起始位置
   stbi__rewind(s);
   // 返回 PIC 格式测试结果
   return r;
}
#endif

// *************************************************************************************************
// GIF loader -- public domain by Jean-Marc Lienher -- simplified/shrunk by stb

#ifndef STBI_NO_GIF
// 定义 GIF 解析过程中需要使用的数据结构 stbi__gif_lzw
typedef struct
{
   stbi__int16 prefix;
   stbi_uc first;
   stbi_uc suffix;
} stbi__gif_lzw;

// 定义 GIF 解析过程中需要使用的数据结构 stbi__gif
typedef struct
{
   int w,h;
   stbi_uc *out;                 // 输出缓冲区（始终为 4 个分量）
   stbi_uc *background;          // GIF 中当前的“背景”
   stbi_uc *history;
   int flags, bgindex, ratio, transparent, eflags;
   stbi_uc  pal[256][4];
   stbi_uc lpal[256][4];
   stbi__gif_lzw codes[8192];
   stbi_uc *color_table;
   int parse, step;
   int lflags;
   int start_x, start_y;
   int max_x, max_y;
   int cur_x, cur_y;
   int line_size;
   int delay;
} stbi__gif;

// 测试输入流 s 是否为原始 GIF 格式
static int stbi__gif_test_raw(stbi__context *s)
{
   int sz;
   // 检查 GIF 文件头部是否为 "GIF8"
   if (stbi__get8(s) != 'G' || stbi__get8(s) != 'I' || stbi__get8(s) != 'F' || stbi__get8(s) != '8') return 0;
   // 读取版本号
   sz = stbi__get8(s);
   // 检查版本号是否为 '9' 或 '7'
   if (sz != '9' && sz != '7') return 0;
   // 检查是否为合法的 GIF 文件
   if (stbi__get8(s) != 'a') return 0;
   // 返回测试结果
   return 1;
}

// 测试输入流 s 是否为 GIF 格式
static int stbi__gif_test(stbi__context *s)
{
   // 调用 stbi__gif_test_raw 函数测试输入流 s 是否为原始 GIF 格式
   int r = stbi__gif_test_raw(s);
   // 将输入流 s 重置到起始位置
   stbi__rewind(s);
   // 返回测试结果
   return r;
}

// 解析 GIF 颜色表
static void stbi__gif_parse_colortable(stbi__context *s, stbi_uc pal[256][4], int num_entries, int transp)
{
   int i;
   // 遍历颜色表，读取 RGB 值
   for (i=0; i < num_entries; ++i) {
      pal[i][2] = stbi__get8(s);
      pal[i][1] = stbi__get8(s);
      pal[i][0] = stbi__get8(s);
      // 设置透明颜色的 Alpha 通道为 0，其他为 255
      pal[i][3] = transp == i ? 0 : 255;
   }
}

// 解析 GIF 文件头部信息
static int stbi__gif_header(stbi__context *s, stbi__gif *g, int *comp, int is_info)
{
   // 读取 GIF 文件的版本信息
   stbi_uc version;
   // 检查文件头是否为 "GIF8"
   if (stbi__get8(s) != 'G' || stbi__get8(s) != 'I' || stbi__get8(s) != 'F' || stbi__get8(s) != '8')
      return stbi__err("not GIF", "Corrupt GIF");

   // 读取版本号
   version = stbi__get8(s);
   // 检查版本号是否为 '7' 或 '9'
   if (version != '7' && version != '9')    return stbi__err("not GIF", "Corrupt GIF");
   // 检查是否为 'a'
   if (stbi__get8(s) != 'a')                return stbi__err("not GIF", "Corrupt GIF");

   // 初始化失败原因为空
   stbi__g_failure_reason = "";
   // 读取图像宽度和高度
   g->w = stbi__get16le(s);
   g->h = stbi__get16le(s);
   // 读取标志位
   g->flags = stbi__get8(s);
   // 读取背景颜色索引
   g->bgindex = stbi__get8(s);
   // 读取像素宽高比
   g->ratio = stbi__get8(s);
   // 初始化透明颜色索引为 -1
   g->transparent = -1;

   // 检查图像宽度和高度是否过大
   if (g->w > STBI_MAX_DIMENSIONS) return stbi__err("too large","Very large image (corrupt?)");
   if (g->h > STBI_MAX_DIMENSIONS) return stbi__err("too large","Very large image (corrupt?)");

   // 如果 comp 不为 0，则设置为 4，需要在解析注释后才能确定是 3 还是 4
   if (comp != 0) *comp = 4;

   // 如果是信息模式，则返回 1
   if (is_info) return 1;

   // 如果标志位中包含 0x80，则解析颜色表
   if (g->flags & 0x80)
      stbi__gif_parse_colortable(s,g->pal, 2 << (g->flags & 7), -1);

   // 返回 1
   return 1;
}

// 解析 GIF 图像的信息
static int stbi__gif_info_raw(stbi__context *s, int *x, int *y, int *comp)
{
   // 分配内存给 GIF 结构体
   stbi__gif* g = (stbi__gif*) stbi__malloc(sizeof(stbi__gif));
   // 如果内存分配失败，则返回错误
   if (!g) return stbi__err("outofmem", "Out of memory");
   // 解析 GIF 头部信息
   if (!stbi__gif_header(s, g, comp, 1)) {
      // 释放内存，重置指针位置，返回 0
      STBI_FREE(g);
      stbi__rewind( s );
      return 0;
   }
   // 如果 x 不为空，则将图像宽度赋值给 x
   if (x) *x = g->w;
   // 如果 y 不为空，则将图像高度赋值给 y
   if (y) *y = g->h;
   // 释放内存，返回 1
   STBI_FREE(g);
   return 1;
}

// 输出 GIF 编码
static void stbi__out_gif_code(stbi__gif *g, stbi__uint16 code)
{
   // 定义指向当前像素和颜色表的指针，以及索引变量
   stbi_uc *p, *c;
   int idx;

   // 递归解码前缀，因为链表是反向的，反向处理交错图像会很麻烦
   if (g->codes[code].prefix >= 0)
      stbi__out_gif_code(g, g->codes[code].prefix);

   // 如果当前行数超过最大行数，则返回
   if (g->cur_y >= g->max_y) return;

   // 计算当前像素索引
   idx = g->cur_x + g->cur_y;
   // 指向当前像素的指针，并标记历史记录
   p = &g->out[idx];
   g->history[idx / 4] = 1;

   // 指向颜色表中对应索引的颜色值
   c = &g->color_table[g->codes[code].suffix * 4];
   // 如果颜色值的透明度大于128，则不渲染透明像素
   if (c[3] > 128) {
      p[0] = c[2];
      p[1] = c[1];
      p[2] = c[0];
      p[3] = c[3];
   }
   g->cur_x += 4;

   // 如果当前列数超过最大列数
   if (g->cur_x >= g->max_x) {
      // 重置列数，增加行数
      g->cur_x = g->start_x;
      g->cur_y += g->step;

      // 处理超出最大行数的情况
      while (g->cur_y >= g->max_y && g->parse > 0) {
         g->step = (1 << g->parse) * g->line_size;
         g->cur_y = g->start_y + (g->step >> 1);
         --g->parse;
      }
   }
}

// 处理 GIF 栅格数据
static stbi_uc *stbi__process_gif_raster(stbi__context *s, stbi__gif *g)
}

// 此函数设计用于支持动画 GIF，尽管 stb_image 不支持
// two_back 是两帧前的图像，用于特定的处理格式
static stbi_uc *stbi__gif_load_next(stbi__context *s, stbi__gif *g, int *comp, int req_comp, stbi_uc *two_back)
}

// 释放 GIF 加载过程中的内存
static void *stbi__load_gif_main_outofmem(stbi__gif *g, stbi_uc *out, int **delays)
{
   STBI_FREE(g->out);
   STBI_FREE(g->history);
   STBI_FREE(g->background);

   if (out) STBI_FREE(out);
   if (delays && *delays) STBI_FREE(*delays);
   return stbi__errpuc("outofmem", "Out of memory");
}

// 加载 GIF 主函数
static void *stbi__load_gif_main(stbi__context *s, int **delays, int *x, int *y, int *z, int *comp, int req_comp)
}

// 加载 GIF 图像
static void *stbi__gif_load(stbi__context *s, int *x, int *y, int *comp, int req_comp, stbi__result_info *ri)
}
{
   // 定义指向无符号字符的指针 u，并初始化为 0
   stbi_uc *u = 0;
   // 定义 stbi__gif 结构体 g，并使用 memset 函数将其初始化为 0
   stbi__gif g;
   memset(&g, 0, sizeof(g));
   // 忽略未使用的参数 ri
   STBI_NOTUSED(ri);

   // 调用 stbi__gif_load_next 函数加载 GIF 图像的下一帧，并将结果赋值给 u
   u = stbi__gif_load_next(s, &g, comp, req_comp, 0);
   // 如果 u 等于 s，则表示已经到达动画 GIF 的结尾标记，将 u 置为 0
   if (u == (stbi_uc *) s) u = 0;  // end of animated gif marker
   // 如果 u 不为空
   if (u) {
      // 将 GIF 图像的宽度赋值给 x
      *x = g.w;
      // 将 GIF 图像的高度赋值给 y

      *y = g.h;

      // 在成功加载后进行颜色格式转换，以便处理多帧图像
      if (req_comp && req_comp != 4)
         u = stbi__convert_format(u, 4, req_comp, g.w, g.h);
   } else if (g.out) {
      // 如果出现错误并且已分配图像缓冲区，则释放它
      STBI_FREE(g.out);
   }

   // 释放用于加载多帧图像所需的缓冲区
   STBI_FREE(g.history);
   STBI_FREE(g.background);

   // 返回加载的图像数据
   return u;
}

// 获取 GIF 图像信息
static int stbi__gif_info(stbi__context *s, int *x, int *y, int *comp)
{
   return stbi__gif_info_raw(s,x,y,comp);
}
#endif

// *************************************************************************************************
// Radiance RGBE HDR loader
// originally by Nicolas Schulz
#ifndef STBI_NO_HDR
// 检测 HDR 文件的签名
static int stbi__hdr_test_core(stbi__context *s, const char *signature)
{
   int i;
   // 遍历签名字符串，逐个比较读取的字符是否与签名相符
   for (i=0; signature[i]; ++i)
      if (stbi__get8(s) != signature[i])
          return 0;
   // 将读取指针重置到起始位置
   stbi__rewind(s);
   return 1;
}

// 检测 HDR 文件的签名
static int stbi__hdr_test(stbi__context* s)
{
   // 检测是否为 #?RADIANCE\n 格式的 HDR 文件
   int r = stbi__hdr_test_core(s, "#?RADIANCE\n");
   // 将读取指针重置到起始位置
   stbi__rewind(s);
   // 如果不是 #?RADIANCE\n 格式的 HDR 文件，则检测是否为 #?RGBE\n 格式的 HDR 文件
   if(!r) {
       r = stbi__hdr_test_core(s, "#?RGBE\n");
       // 将读取指针重置到起始位置
       stbi__rewind(s);
   }
   return r;
}

// 从输入流中获取 HDR 文件的标记
#define STBI__HDR_BUFLEN  1024
static char *stbi__hdr_gettoken(stbi__context *z, char *buffer)
{
   int len=0;
   char c = '\0';

   // 读取一个字符
   c = (char) stbi__get8(z);

   // 循环读取字符，直到遇到换行符或文件结束
   while (!stbi__at_eof(z) && c != '\n') {
      buffer[len++] = c;
      // 如果达到缓冲区长度上限，跳过当前行剩余内容
      if (len == STBI__HDR_BUFLEN-1) {
         // 跳过当前行剩余内容
         while (!stbi__at_eof(z) && stbi__get8(z) != '\n')
            ;
         break;
      }
      // 继续读取下一个字符
      c = (char) stbi__get8(z);
   }

   // 在缓冲区末尾添加字符串结束符
   buffer[len] = 0;
   return buffer;
}
static void stbi__hdr_convert(float *output, stbi_uc *input, int req_comp)
{
   // 如果输入的第四个字节不为0
   if ( input[3] != 0 ) {
      float f1;
      // 指数
      f1 = (float) ldexp(1.0f, input[3] - (int)(128 + 8));
      // 如果请求的通道数小于等于2
      if (req_comp <= 2)
         // 对RGB通道进行加权平均
         output[0] = (input[0] + input[1] + input[2]) * f1 / 3;
      else {
         // 分别对RGB通道进行加权
         output[0] = input[0] * f1;
         output[1] = input[1] * f1;
         output[2] = input[2] * f1;
      }
      // 如果请求的通道数为2，设置第二个通道为1
      if (req_comp == 2) output[1] = 1;
      // 如果请求的通道数为4，设置第四个通道为1
      if (req_comp == 4) output[3] = 1;
   } else {
      switch (req_comp) {
         case 4: output[3] = 1; /* fallthrough */
         case 3: output[0] = output[1] = output[2] = 0;
                 break;
         case 2: output[1] = 1; /* fallthrough */
         case 1: output[0] = 0;
                 break;
      }
   }
}

static float *stbi__hdr_load(stbi__context *s, int *x, int *y, int *comp, int req_comp, stbi__result_info *ri)
{
}

static int stbi__hdr_info(stbi__context *s, int *x, int *y, int *comp)
{
   char buffer[STBI__HDR_BUFLEN];
   char *token;
   int valid = 0;
   int dummy;

   // 如果x、y、comp为空指针，则指向dummy
   if (!x) x = &dummy;
   if (!y) y = &dummy;
   if (!comp) comp = &dummy;

   // 如果不是HDR格式，回到文件开头并返回0
   if (stbi__hdr_test(s) == 0) {
       stbi__rewind( s );
       return 0;
   }

   // 循环读取token，查找是否为32-bit_rle_rgbe格式
   for(;;) {
      token = stbi__hdr_gettoken(s,buffer);
      if (token[0] == 0) break;
      if (strcmp(token, "FORMAT=32-bit_rle_rgbe") == 0) valid = 1;
   }

   // 如果不是有效格式，回到文件开头并返回0
   if (!valid) {
       stbi__rewind( s );
       return 0;
   }
   // 读取Y值
   token = stbi__hdr_gettoken(s,buffer);
   if (strncmp(token, "-Y ", 3)) {
       stbi__rewind( s );
       return 0;
   }
   token += 3;
   *y = (int) strtol(token, &token, 10);
   while (*token == ' ') ++token;
   // 读取X值
   if (strncmp(token, "+X ", 3)) {
       stbi__rewind( s );
       return 0;
   }
   token += 3;
   *x = (int) strtol(token, NULL, 10);
   *comp = 3;
   return 1;
}
#endif // STBI_NO_HDR

#ifndef STBI_NO_BMP
static int stbi__bmp_info(stbi__context *s, int *x, int *y, int *comp)
{
   // 声明指针变量 p
   void *p;
   // 声明结构体变量 info
   stbi__bmp_data info;

   // 设置 info 结构体的 all_a 字段为 255
   info.all_a = 255;
   // 调用 stbi__bmp_parse_header 函数解析 BMP 头部信息，返回结果赋给 p
   p = stbi__bmp_parse_header(s, &info);
   // 如果 p 为空指针，则回退到文件开头并返回 0
   if (p == NULL) {
      stbi__rewind( s );
      return 0;
   }
   // 如果 x 不为空，则将 s->img_x 赋给 x
   if (x) *x = s->img_x;
   // 如果 y 不为空，则将 s->img_y 赋给 y
   if (y) *y = s->img_y;
   // 如果 comp 不为空
   if (comp) {
      // 如果 info 结构体的 bpp 字段为 24 并且 ma 字段为 0xff000000，则将 3 赋给 comp
      if (info.bpp == 24 && info.ma == 0xff000000)
         *comp = 3;
      // 否则，如果 ma 字段不为 0，则将 4 赋给 comp，否则将 3 赋给 comp
      else
         *comp = info.ma ? 4 : 3;
   }
   // 返回 1
   return 1;
}
#endif

#ifndef STBI_NO_PSD
// 解析 PSD 文件信息
static int stbi__psd_info(stbi__context *s, int *x, int *y, int *comp)
{
   // 声明变量 channelCount, dummy, depth
   int channelCount, dummy, depth;
   // 如果 x 为空，则将 dummy 的地址赋给 x
   if (!x) x = &dummy;
   // 如果 y 为空，则将 dummy 的地址赋给 y
   if (!y) y = &dummy;
   // 如果 comp 为空，则将 dummy 的地址赋给 comp
   if (!comp) comp = &dummy;
   // 如果读取的前四个字节不是 '8BPS'，则回退到文件开头并返回 0
   if (stbi__get32be(s) != 0x38425053) {
       stbi__rewind( s );
       return 0;
   }
   // 如果读取的下两个字节不是 1，则回退到文件开头并返回 0
   if (stbi__get16be(s) != 1) {
       stbi__rewind( s );
       return 0;
   }
   // 跳过 6 个字节
   stbi__skip(s, 6);
   // 读取通道数
   channelCount = stbi__get16be(s);
   // 如果通道数小于 0 或大于 16，则回退到文件开头并返回 0
   if (channelCount < 0 || channelCount > 16) {
       stbi__rewind( s );
       return 0;
   }
   // 将读取的两个字节赋给 y
   *y = stbi__get32be(s);
   // 将读取的两个字节赋给 x
   *x = stbi__get32be(s);
   // 读取深度
   depth = stbi__get16be(s);
   // 如果深度不是 8 或 16，则回退到文件开头并返回 0
   if (depth != 8 && depth != 16) {
       stbi__rewind( s );
       return 0;
   }
   // 如果读取的两个字节不是 3，则回退到文件开头并返回 0
   if (stbi__get16be(s) != 3) {
       stbi__rewind( s );
       return 0;
   }
   // 将 4 赋给 comp
   *comp = 4;
   // 返回 1
   return 1;
}

// 判断 PSD 文件是否为 16 位
static int stbi__psd_is16(stbi__context *s)
{
   // 声明变量 channelCount, depth
   int channelCount, depth;
   // 如果读取的前四个字节不是 '8BPS'，则回退到文件开头并返回 0
   if (stbi__get32be(s) != 0x38425053) {
       stbi__rewind( s );
       return 0;
   }
   // 如果读取的下两个字节不是 1，则回退到文件开头并返回 0
   if (stbi__get16be(s) != 1) {
       stbi__rewind( s );
       return 0;
   }
   // 跳过 6 个字节
   stbi__skip(s, 6);
   // 读取通道数
   channelCount = stbi__get16be(s);
   // 如果通道数小于 0 或大于 16，则回退到文件开头并返回 0
   if (channelCount < 0 || channelCount > 16) {
       stbi__rewind( s );
       return 0;
   }
   // 跳过两个 32 位整数
   STBI_NOTUSED(stbi__get32be(s));
   STBI_NOTUSED(stbi__get32be(s));
   // 读取深度
   depth = stbi__get16be(s);
   // 如果深度不是 16，则回退到文件开头并返回 0
   if (depth != 16) {
       stbi__rewind( s );
       return 0;
   }
   // 返回 1
   return 1;
}
#endif

#ifndef STBI_NO_PIC
// 解析 PIC 文件信息
static int stbi__pic_info(stbi__context *s, int *x, int *y, int *comp)
{
   // 初始化变量，用于记录实际通道数、数据包数量、是否链式传输、虚拟变量
   int act_comp=0,num_packets=0,chained,dummy;
   // 创建数据包数组
   stbi__pic_packet packets[10];

   // 如果 x 为空，则指向虚拟变量 dummy
   if (!x) x = &dummy;
   // 如果 y 为空，则指向虚拟变量 dummy
   if (!y) y = &dummy;
   // 如果 comp 为空，则指向虚拟变量 dummy
   if (!comp) comp = &dummy;

   // 检查文件是否为 PIC 格式
   if (!stbi__pic_is4(s,"\x53\x80\xF6\x34")) {
      // 回退文件指针并返回
      stbi__rewind(s);
      return 0;
   }

   // 跳过 88 字节
   stbi__skip(s, 88);

   // 读取图片宽度和高度
   *x = stbi__get16be(s);
   *y = stbi__get16be(s);
   // 如果已到达文件末尾，则回退文件指针并返回
   if (stbi__at_eof(s)) {
      stbi__rewind( s);
      return 0;
   }
   // 检查宽度和高度是否合法
   if ( (*x) != 0 && (1 << 28) / (*x) < (*y)) {
      stbi__rewind( s );
      return 0;
   }

   // 跳过 8 字节
   stbi__skip(s, 8);

   // 循环读取数据包
   do {
      stbi__pic_packet *packet;

      // 如果数据包数量达到上限，则返回
      if (num_packets==sizeof(packets)/sizeof(packets[0]))
         return 0;

      // 获取当前数据包
      packet = &packets[num_packets++];
      chained = stbi__get8(s);
      packet->size    = stbi__get8(s);
      packet->type    = stbi__get8(s);
      packet->channel = stbi__get8(s);
      act_comp |= packet->channel;

      // 如果已到达文件末尾，则回退文件指针并返回
      if (stbi__at_eof(s)) {
          stbi__rewind( s );
          return 0;
      }
      // 如果数据包大小不为 8，则回退文件指针并返回
      if (packet->size != 8) {
          stbi__rewind( s );
          return 0;
      }
   } while (chained);

   // 根据通道信息确定通道数
   *comp = (act_comp & 0x10 ? 4 : 3);

   return 1;
}
#endif

// *************************************************************************************************
// Portable Gray Map and Portable Pixel Map loader
// by Ken Miller
//
// PGM: http://netpbm.sourceforge.net/doc/pgm.html
// PPM: http://netpbm.sourceforge.net/doc/ppm.html
//
// Known limitations:
//    Does not support comments in the header section
//    Does not support ASCII image data (formats P2 and P3)

#ifndef STBI_NO_PNM

// 检测是否为 PNM 格式
static int      stbi__pnm_test(stbi__context *s)
{
   char p, t;
   p = (char) stbi__get8(s);
   t = (char) stbi__get8(s);
   // 如果不是 PNM 格式，则回退文件指针并返回
   if (p != 'P' || (t != '5' && t != '6')) {
       stbi__rewind( s );
       return 0;
   }
   return 1;
}

// 加载 PNM 格式图片
static void *stbi__pnm_load(stbi__context *s, int *x, int *y, int *comp, int req_comp, stbi__result_info *ri)
{
   // 定义指向无符号字符的指针变量 out
   stbi_uc *out;
   // 使用宏函数 STBI_NOTUSED，标记变量 ri 未使用
   STBI_NOTUSED(ri);

   // 调用 stbi__pnm_info 函数获取 PNM 图像信息，包括图像宽度、高度、通道数和每通道位数
   ri->bits_per_channel = stbi__pnm_info(s, (int *)&s->img_x, (int *)&s->img_y, (int *)&s->img_n);
   // 如果每通道位数为 0，则返回 0
   if (ri->bits_per_channel == 0)
      return 0;

   // 如果图像高度超过最大限制，则返回错误信息
   if (s->img_y > STBI_MAX_DIMENSIONS) return stbi__errpuc("too large","Very large image (corrupt?)");
   // 如果图像宽度超过最大限制，则返回错误信息
   if (s->img_x > STBI_MAX_DIMENSIONS) return stbi__errpuc("too large","Very large image (corrupt?)");

   // 将图像宽度赋值给指针变量 x
   *x = s->img_x;
   // 将图像高度赋值给指针变量 y
   *y = s->img_y;
   // 如果 comp 不为空，则将图像通道数赋值给指针变量 comp
   if (comp) *comp = s->img_n;

   // 检查 PNM 图像大小是否合法，如果不合法则返回错误信息
   if (!stbi__mad4sizes_valid(s->img_n, s->img_x, s->img_y, ri->bits_per_channel / 8, 0))
      return stbi__errpuc("too large", "PNM too large");

   // 分配内存空间用于存储图像数据
   out = (stbi_uc *) stbi__malloc_mad4(s->img_n, s->img_x, s->img_y, ri->bits_per_channel / 8, 0);
   // 如果内存分配失败，则返回错误信息
   if (!out) return stbi__errpuc("outofmem", "Out of memory");
   // 从输入流中读取图像数据到 out
   if (!stbi__getn(s, out, s->img_n * s->img_x * s->img_y * (ri->bits_per_channel / 8))) {
      // 释放内存空间并返回错误信息
      STBI_FREE(out);
      return stbi__errpuc("bad PNM", "PNM file truncated");
   }

   // 如果请求的通道数不为空且与图像通道数不同，则进行通道数转换
   if (req_comp && req_comp != s->img_n) {
      // 如果每通道位数为 16，则调用 stbi__convert_format16 进行格式转换
      if (ri->bits_per_channel == 16) {
         out = (stbi_uc *) stbi__convert_format16((stbi__uint16 *) out, s->img_n, req_comp, s->img_x, s->img_y);
      } else {
         // 否则调用 stbi__convert_format 进行格式转换
         out = stbi__convert_format(out, s->img_n, req_comp, s->img_x, s->img_y);
      }
      // 如果转换失败，则返回空指针
      if (out == NULL) return out; // stbi__convert_format frees input on failure
   }
   // 返回图像数据
   return out;
}

// 判断字符是否为空格或制表符等空白字符
static int stbi__pnm_isspace(char c)
{
   return c == ' ' || c == '\t' || c == '\n' || c == '\v' || c == '\f' || c == '\r';
}

// 跳过输入流中的空白字符
static void stbi__pnm_skip_whitespace(stbi__context *s, char *c)
{
   for (;;) {
      // 循环直到遇到非空白字符
      while (!stbi__at_eof(s) && stbi__pnm_isspace(*c))
         *c = (char) stbi__get8(s);

      // 如果遇到注释符号 '#'，则跳过注释内容
      if (stbi__at_eof(s) || *c != '#')
         break;

      while (!stbi__at_eof(s) && *c != '\n' && *c != '\r' )
         *c = (char) stbi__get8(s);
   }
}

// 判断字符是否为数字字符
static int stbi__pnm_isdigit(char c)
{
   return c >= '0' && c <= '9';
}

// 从输入流中获取整数
static int stbi__pnm_getinteger(stbi__context *s, char *c)
{
   // 初始化一个整型变量 value 为 0
   int value = 0;

   // 当未到达文件结尾且当前字符为数字时循环
   while (!stbi__at_eof(s) && stbi__pnm_isdigit(*c)) {
      // 更新 value 为原值乘以 10 加上当前字符减去 '0' 的值
      value = value*10 + (*c - '0');
      // 更新当前字符为下一个字符
      *c = (char) stbi__get8(s);
      // 如果 value 大于 214748364 或者等于 214748364 且当前字符大于 '7'，则返回错误信息
      if((value > 214748364) || (value == 214748364 && *c > '7'))
          return stbi__err("integer parse overflow", "Parsing an integer in the PPM header overflowed a 32-bit int");
   }

   // 返回解析出的整数值
   return value;
}

// 获取 PNM 图像的信息
static int      stbi__pnm_info(stbi__context *s, int *x, int *y, int *comp)
{
   int maxv, dummy;
   char c, p, t;

   // 如果 x 为 NULL，则指向 dummy 变量
   if (!x) x = &dummy;
   // 如果 y 为 NULL，则指向 dummy 变量
   if (!y) y = &dummy;
   // 如果 comp 为 NULL，则指向 dummy 变量
   if (!comp) comp = &dummy;

   // 将文件指针重置到文件开头
   stbi__rewind(s);

   // 获取标识符
   p = (char) stbi__get8(s);
   t = (char) stbi__get8(s);
   // 如果标识符不为 'P' 或者 t 不为 '5' 或 '6'，则将文件指针重置到文件开头并返回 0
   if (p != 'P' || (t != '5' && t != '6')) {
       stbi__rewind(s);
       return 0;
   }

   // 根据 t 的值确定图像的通道数
   *comp = (t == '6') ? 3 : 1;  // '5' is 1-component .pgm; '6' is 3-component .ppm

   // 获取下一个字符
   c = (char) stbi__get8(s);
   // 跳过空白字符
   stbi__pnm_skip_whitespace(s, &c);

   // 读取图像宽度
   *x = stbi__pnm_getinteger(s, &c); // read width
   // 如果宽度为 0，则返回错误信息
   if(*x == 0)
       return stbi__err("invalid width", "PPM image header had zero or overflowing width");
   // 跳过空白字符
   stbi__pnm_skip_whitespace(s, &c);

   // 读取图像高度
   *y = stbi__pnm_getinteger(s, &c); // read height
   // 如果高度为 0，则返回错误信息
   if (*y == 0)
       return stbi__err("invalid width", "PPM image header had zero or overflowing width");
   // 跳过空白字符
   stbi__pnm_skip_whitespace(s, &c);

   // 读取最大像素值
   maxv = stbi__pnm_getinteger(s, &c);  // read max value
   // 如果最大像素值大于 65535，则返回错误信息
   if (maxv > 65535)
      return stbi__err("max value > 65535", "PPM image supports only 8-bit and 16-bit images");
   // 如果最大像素值大于 255，则返回 16
   else if (maxv > 255)
      return 16;
   // 否则返回 8
   else
      return 8;
}

// 判断 PNM 图像是否为 16 位
static int stbi__pnm_is16(stbi__context *s)
{
   // 如果获取 PNM 图像信息返回值为 16，则返回 1，否则返回 0
   if (stbi__pnm_info(s, NULL, NULL, NULL) == 16)
       return 1;
   return 0;
}
#endif

// 获取图像信息的主函数
static int stbi__info_main(stbi__context *s, int *x, int *y, int *comp)
{
   // 如果不定义 STBI_NO_JPEG，则调用 stbi__jpeg_info 函数获取 JPEG 图像信息
   #ifndef STBI_NO_JPEG
   if (stbi__jpeg_info(s, x, y, comp)) return 1;
   #endif

   // 如果不定义 STBI_NO_PNG，则调用 stbi__png_info 函数获取 PNG 图像信息
   #ifndef STBI_NO_PNG
   if (stbi__png_info(s, x, y, comp))  return 1;
   #endif

   // 如果不定义 STBI_NO_GIF，则调用 stbi__gif_info 函数获取 GIF 图像信息
   #ifndef STBI_NO_GIF
   if (stbi__gif_info(s, x, y, comp))  return 1;
   #endif

   // 如果不定义 STBI_NO_BMP，则调用 stbi__bmp_info 函数获取 BMP 图像信息
   #ifndef STBI_NO_BMP
   if (stbi__bmp_info(s, x, y, comp))  return 1;
   #endif

   // 如果不定义 STBI_NO_PSD，则调用 stbi__psd_info 函数获取 PSD 图像信息
   #ifndef STBI_NO_PSD
   if (stbi__psd_info(s, x, y, comp))  return 1;
   #endif

   // 如果不定义 STBI_NO_PIC，则调用 stbi__pic_info 函数获取 PIC 图像信息
   #ifndef STBI_NO_PIC
   if (stbi__pic_info(s, x, y, comp))  return 1;
   #endif

   // 如果不定义 STBI_NO_PNM，则调用 stbi__pnm_info 函数获取 PNM 图像信息
   #ifndef STBI_NO_PNM
   if (stbi__pnm_info(s, x, y, comp))  return 1;
   #endif

   // 如果不定义 STBI_NO_HDR，则调用 stbi__hdr_info 函数获取 HDR 图像信息
   #ifndef STBI_NO_HDR
   if (stbi__hdr_info(s, x, y, comp))  return 1;
   #endif

   // 最后测试 TGA 图像，因为测试方法不够好
   #ifndef STBI_NO_TGA
   if (stbi__tga_info(s, x, y, comp))
       return 1;
   #endif
   // 返回错误信息，表示图像类型未知或损坏
   return stbi__err("unknown image type", "Image not of any known type, or corrupt");
}

// 检查图像是否为 16 位
static int stbi__is_16_main(stbi__context *s)
{
   // 如果不定义 STBI_NO_PNG，则调用 stbi__png_is16 函数检查 PNG 图像是否为 16 位
   #ifndef STBI_NO_PNG
   if (stbi__png_is16(s))  return 1;
   #endif

   // 如果不定义 STBI_NO_PSD，则调用 stbi__psd_is16 函数检查 PSD 图像是否为 16 位
   #ifndef STBI_NO_PSD
   if (stbi__psd_is16(s))  return 1;
   #endif

   // 如果不定义 STBI_NO_PNM，则调用 stbi__pnm_is16 函数检查 PNM 图像是否为 16 位
   #ifndef STBI_NO_PNM
   if (stbi__pnm_is16(s))  return 1;
   #endif
   return 0;
}

// 从文件名获取图像信息
#ifndef STBI_NO_STDIO
STBIDEF int stbi_info(char const *filename, int *x, int *y, int *comp)
{
    // 打开文件
    FILE *f = stbi__fopen(filename, "rb");
    int result;
    // 如果文件打开失败，返回错误信息
    if (!f) return stbi__err("can't fopen", "Unable to open file");
    // 从文件获取图像信息
    result = stbi_info_from_file(f, x, y, comp);
    // 关闭文件
    fclose(f);
    return result;
}

// 从文件获取图像信息
STBIDEF int stbi_info_from_file(FILE *f, int *x, int *y, int *comp)
{
   int r;
   stbi__context s;
   long pos = ftell(f);
   // 初始化图像上下文
   stbi__start_file(&s, f);
   // 获取图像信息
   r = stbi__info_main(&s,x,y,comp);
   // 恢复文件读取位置
   fseek(f,pos,SEEK_SET);
   return r;
}

// 检查图像是否为 16 位
STBIDEF int stbi_is_16_bit(char const *filename)
{
    // 打开文件
    FILE *f = stbi__fopen(filename, "rb");
    int result;
    // 如果文件打开失败，返回错误信息
    if (!f) return stbi__err("can't fopen", "Unable to open file");
    // 检查图像是否为 16 位
    result = stbi_is_16_bit_from_file(f);
    // 关闭文件
    fclose(f);
    return result;
}
}
// 检查文件是否包含16位数据
STBIDEF int stbi_is_16_bit_from_file(FILE *f)
{
   // 定义变量 r
   int r;
   // 创建 stbi__context 结构体变量 s
   stbi__context s;
   // 获取当前文件指针位置
   long pos = ftell(f);
   // 初始化文件流
   stbi__start_file(&s, f);
   // 检查文件是否包含16位数据
   r = stbi__is_16_main(&s);
   // 将文件指针位置设置回之前的位置
   fseek(f,pos,SEEK_SET);
   // 返回结果
   return r;
}
#endif // !STBI_NO_STDIO

// 从内存中获取图像信息
STBIDEF int stbi_info_from_memory(stbi_uc const *buffer, int len, int *x, int *y, int *comp)
{
   // 创建 stbi__context 结构体变量 s
   stbi__context s;
   // 初始化内存流
   stbi__start_mem(&s,buffer,len);
   // 获取图像信息
   return stbi__info_main(&s,x,y,comp);
}

// 从回调函数中获取图像信息
STBIDEF int stbi_info_from_callbacks(stbi_io_callbacks const *c, void *user, int *x, int *y, int *comp)
{
   // 创建 stbi__context 结构体变量 s
   stbi__context s;
   // 初始化回调函数流
   stbi__start_callbacks(&s, (stbi_io_callbacks *) c, user);
   // 获取图像信息
   return stbi__info_main(&s,x,y,comp);
}

// 从内存中检查是否包含16位数据
STBIDEF int stbi_is_16_bit_from_memory(stbi_uc const *buffer, int len)
{
   // 创建 stbi__context 结构体变量 s
   stbi__context s;
   // 初始化内存流
   stbi__start_mem(&s,buffer,len);
   // 检查是否包含16位数据
   return stbi__is_16_main(&s);
}

// 从回调函数中检查是否包含16位数据
STBIDEF int stbi_is_16_bit_from_callbacks(stbi_io_callbacks const *c, void *user)
{
   // 创建 stbi__context 结构体变量 s
   stbi__context s;
   // 初始化回调函数流
   stbi__start_callbacks(&s, (stbi_io_callbacks *) c, user);
   // 检查是否包含16位数据
   return stbi__is_16_main(&s);
}

#endif // STB_IMAGE_IMPLEMENTATION
# 这部分代码是版权声明和许可证信息，不属于实际的程序代码
# 以下是两种不同的许可证选择，一种是 MIT 许可证，另一种是 Public Domain 许可证
# MIT 许可证允许在特定条件下使用、修改、发布、销售等，但不提供任何担保
# Public Domain 许可证将软件释放到公共领域，允许任何人以任何方式使用，作者放弃对软件的版权
```