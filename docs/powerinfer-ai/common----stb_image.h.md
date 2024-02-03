# `PowerInfer\common\stb_image.h`

```cpp
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
# 最近的修订历史记录，列出了每个版本的日期和更新内容
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
    # 贡献者列表，包括贡献者姓名和贡献的内容
    Jean-Marc Lienher (gif)                Ben "Disch" Wenger (io callbacks)
    Tom Seddon (pic)                       Omar Cornut (1/2/4-bit PNG)
    Thatcher Ulrich (psd)                  Nicolas Guillemot (vertical flip)
    Ken Miller (pgm, ppm)                  Richard Mitton (16-bit PSD)
    github:urraka (animated gif)           Junggon Kim (PNM comments)
    Christopher Forseth (animated gif)     Daniel Gibson (16-bit TGA)
                                           socks-the-fox (16-bit PNG)
                                           Jeremy Sawicki (handle all ImageNet JPGs)
    Optimizations & bugfixes                Mikhail Morozov (1-bit BMP)
    Fabian "ryg" Giesen                    Anael Seghezzi (is-16-bit query)
    Arseny Kapoulkine                      Simon Breuss (16-bit PNM)
    John-Mark Allen
    Carmelo J Fdez-Aguera
    
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
    # 以下是一些开发者的名字和他们的 GitHub 账号
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
// 如果未定义 STBI_INCLUDE_STB_IMAGE_H，则定义 STBI_INCLUDE_STB_IMAGE_H
#ifndef STBI_INCLUDE_STB_IMAGE_H
#define STBI_INCLUDE_STB_IMAGE_H

// 文档说明
//
// 限制：
//    - 没有每通道 12 位的 JPEG
//    - 没有使用算术编码的 JPEG
//    - GIF 总是返回 *comp=4
//
// 基本用法（有关 HDR 用法，请参阅下面的 HDR 讨论）：
//    int x,y,n;
//    unsigned char *data = stbi_load(filename, &x, &y, &n, 0);
//    // ... 如果数据不为空，则处理数据 ...
//    // ... x = 宽度，y = 高度，n = 每像素的 8 位组件数 ...
//    // ... 将 '0' 替换为 '1'..'4' 以强制每像素有指定数量的组件
//    // ... 但 'n' 将始终是如果你说 0 时的数量
//    stbi_image_free(data);
//
// 标准参数：
//    int *x                 -- 输出图像的宽度（以像素为单位）
//    int *y                 -- 输出图像的高度（以像素为单位）
//    int *channels_in_file  -- 输出图像文件中的图像组件数
//    int desired_channels   -- 如果非零，则请求结果中的图像组件数
//
// 图像加载器的返回值是一个指向像素数据的 'unsigned char *'，如果分配失败或图像损坏或无效，则为 NULL。像素数据由 *y 个扫描线和 *x 个像素组成，每个像素由 N 个交错的 8 位组件组成；指向的第一个像素是图像中最左上角的像素。无论格式如何，图像扫描线之间或像素之间都没有填充。如果 desired_channels 非零，则组件数 N 为 'desired_channels'，否则为 *channels_in_file。如果 desired_channels 非零，则 *channels_in_file 为否则输出的组件数。例如，如果将 desired_channels 设置为 4，则将始终获得 RGBA 输出，但您可以检查 *channels_in_file，以查看它是否是平凡不透明的，例如源图像中只有 3 个通道。
//
// 具有 N 个组件的输出图像中，每个像素的组件按以下顺序交错：
//
// 定义图像的颜色通道数和对应的组件
// 1 表示灰度图，2 表示灰度图带透明通道，3 表示红绿蓝三通道，4 表示带透明通道的红绿蓝三通道
//
// 如果由于任何原因图像加载失败，返回值将为 NULL，
// *x, *y, *channels_in_file 将保持不变。可以查询 stbi_failure_reason() 函数以获取加载失败的简要解释。
// 定义 STBI_NO_FAILURE_STRINGS 可以避免编译这些字符串，定义 STBI_FAILURE_USERMSG 可以获得稍微用户友好一些的解释。
//
// 调色板 PNG、BMP、GIF 和 PIC 图像会自动去除调色板。
//
// 若要查询图像的宽度、高度和通道数，而不必解码整个文件，可以使用 stbi_info 系列函数：
//
//   int x,y,n,ok;
//   ok = stbi_info(filename, &x, &y, &n);
//   // 如果图像是支持的格式，返回 ok=1 并设置 x, y, n，
//   // 否则返回 ok=0。
//
// 注意，stb_image 在其公共 API 中广泛使用 int 类型表示大小，包括内存缓冲区的大小。
// 这已经成为 API 的一部分，因此很难在不引起破坏的情况下进行更改。
// 因此，各种图像加载器都对图像大小有一定限制；这些限制在格式上有所不同，但通常都是接近 2GB 或接近 1GB。
// 当解码后的图像大于这个限制时，stb_image 解码将失败。
//
// 另外，stb_image 将拒绝具有任何维度大于可配置的 STBI_MAX_DIMENSIONS 的图像文件，
// 默认为 2**24 = 16777216 像素。由于上述内存限制，使得具有这些维度的图像正确加载的唯一方法是具有极端的长宽比。
// 无论如何，这里的假设是，这样的大图像可能是畸形的或恶意的。
// 如果确实需要加载具有单个维度的大图像
// 如果定义了 STBI_MAX_DIMENSIONS 并且大于这个值，但仍然适合于整体大小限制，你可以自己定义 STBI_MAX_DIMENSIONS 为更大的值。
//
// ===========================================================================
//
// UNICODE:
//
//   如果在 Windows 上编译并且希望使用 Unicode 文件名，请使用以下编译指令：
//       #define STBI_WINDOWS_UTF8
//   并传递 utf8 编码的文件名。调用 stbi_convert_wchar_to_utf8 将 Windows 的 wchar_t 文件名转换为 utf8。
//
// ===========================================================================
//
// 哲学
//
// stb 库的设计优先级如下：
//
//    1. 易于使用
//    2. 易于维护
//    3. 良好的性能
//
// 有时我会让 "良好的性能" 在 "易于维护" 之上，为了获得最佳性能，我可能会提供一些不太容易使用的 API，以获得更高的性能，除了易于使用的 API。尽管如此，重要的是要记住，从你作为这个库的客户的角度来看，你关心的只有 #1 和 #3，而 stb 库并不强调 #3 胜过一切。
//
// 一些次要的优先级直接源自前两个优先级，其中一些提供了更明确的原因，说明为什么性能不能被强调。
//
//    - 可移植性（"易于使用"）
//    - 小的源代码占用空间（"易于维护"）
//    - 无依赖性（"易于使用"）
//
// ===========================================================================
//
// I/O 回调
//
// I/O 回调允许你从任意来源读取数据，比如打包文件或其他来源。通过一个小的内部缓冲区（目前为 128 字节）处理通过回调读取的数据，以尝试减少开销。
//
// 你必须定义的三个函数是 "read"（读取一些字节的数据），"skip"（跳过一些字节的数据），"eof"（报告流是否已经结束）。
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
// 设置图像的线性缩放因子为1.0
// 设置图像的伽马值为2.2
// 最后，根据文件名（或打开的文件或内存块）包含的图像数据，可以使用 stbi_is_hdr 函数查询“最合适”的接口（即图像是否为 HDR）。
// 支持 iPhone 格式的 PNG 转换为 RGB，即使它们在内部编码方式不同。调用 stbi_convert_iphone_png_to_rgb(1) 启用此转换。
// 同时调用 stbi_set_unpremultiply_on_load(1) 强制每个像素进行除法，以去除任何预乘的 alpha 通道，仅当图像文件明确表示存在预乘数据时（目前仅在 iPhone 图像中发生，并且仅在启用 iPhone 转换为 RGB 处理时发生）。
// 可以通过在创建实现之前 #define 以下符号之一或多个来抑制任何解码器的实现，以减少代码占用空间。
// 可以仅请求特定的解码器，并抑制所有其他解码器（这将更具前向兼容性，因为添加新解码器不需要显式禁用它们）。
//   - 如果使用 STBI_NO_PNG（或者 _ONLY_ 没有 PNG），但仍希望 zlib 解码器可用，请定义 STBI_SUPPORT_ZLIB
//
//  - 如果定义了 STBI_MAX_DIMENSIONS，stb_image 将拒绝大于该尺寸的图像（宽度或高度），不进行进一步处理。
//    这是为了让程序在野外设置一个上限，以防止拒绝服务攻击未经信任的数据，因为可以生成一个
//    巨大尺寸的有效图像，并迫使 stb_image 分配一个巨大的内存块并花费不成比例的时间来解码它。默认情况下，这个值设置为（1 << 24），即 16777216，但这仍然是非常大的。

#ifndef STBI_NO_STDIO
#include <stdio.h>
#endif // STBI_NO_STDIO

#define STBI_VERSION 1

enum {
    STBI_default = 0, // 仅用于期望的通道数

    STBI_grey = 1,
    STBI_grey_alpha = 2,
    STBI_rgb = 3,
    STBI_rgb_alpha = 4
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
// 主要 API - 适用于任何类型的图像
//

//
// 通过文件名、打开文件或内存缓冲区加载图像
//

typedef struct {
    int (*read)(void * user, char * data,
                int size);            // 用 'size' 字节填充 'data'。返回实际读取的字节数
    void (*skip)(void * user, int n); // 跳过接下来的 'n' 字节，或者如果是负数，则 'unget' 最后的 -n 字节
    int (*eof)(void * user);          // 如果我们在文件/数据的末尾，则返回非零值
} stbi_io_callbacks;

////////////////////////////////////
//
// 每通道 8 位接口
//

STBIDEF stbi_uc * stbi_load_from_memory(stbi_uc const * buffer, int len, int * x, int * y, int * channels_in_file,
                                        int desired_channels);
# 通过回调函数加载图像数据，返回无符号字符指针
STBIDEF stbi_uc * stbi_load_from_callbacks(stbi_io_callbacks const * clbk, void * user, int * x, int * y,
                                           int * channels_in_file, int desired_channels);

# 如果没有禁用标准输入输出，则通过文件名加载图像数据，返回无符号字符指针
STBIDEF stbi_uc * stbi_load(char const * filename, int * x, int * y, int * channels_in_file, int desired_channels);
# 如果没有禁用标准输入输出，则通过文件指针加载图像数据，返回无符号字符指针，加载后文件指针指向图像后面
STBIDEF stbi_uc * stbi_load_from_file(FILE * f, int * x, int * y, int * channels_in_file, int desired_channels);

# 如果没有禁用 GIF 格式，则通过内存加载 GIF 图像数据，返回无符号字符指针
STBIDEF stbi_uc * stbi_load_gif_from_memory(stbi_uc const * buffer, int len, int ** delays, int * x, int * y, int * z,
                                            int * comp, int req_comp);

# 如果定义了 STBI_WINDOWS_UTF8，则转换宽字符为 UTF-8 编码
STBIDEF int stbi_convert_wchar_to_utf8(char * buffer, size_t bufferlen, const wchar_t * input);

# 16位每通道接口
# 通过内存加载 16 位每通道图像数据，返回无符号短整型指针
STBIDEF stbi_us * stbi_load_16_from_memory(stbi_uc const * buffer, int len, int * x, int * y, int * channels_in_file,
                                           int desired_channels);
# 通过回调函数加载 16 位每通道图像数据，返回无符号短整型指针
STBIDEF stbi_us * stbi_load_16_from_callbacks(stbi_io_callbacks const * clbk, void * user, int * x, int * y,
                                              int * channels_in_file, int desired_channels);
# 如果没有禁用标准输入输出，则通过文件名加载 16 位每通道图像数据，返回无符号短整型指针
STBIDEF stbi_us * stbi_load_16(char const * filename, int * x, int * y, int * channels_in_file, int desired_channels);
# 如果没有禁用标准输入输出，则通过文件指针加载 16 位每通道图像数据，返回无符号短整型指针
STBIDEF stbi_us * stbi_load_from_file_16(FILE * f, int * x, int * y, int * channels_in_file, int desired_channels);

# 浮点数每通道接口
# 如果没有禁用线性插值，则通过内存加载浮点数每通道图像数据，返回浮点数指针
STBIDEF float * stbi_loadf_from_memory(stbi_uc const * buffer, int len, int * x, int * y, int * channels_in_file,
                                       int desired_channels);
// 从回调函数中加载 HDR 图像数据，返回浮点数指针
STBIDEF float * stbi_loadf_from_callbacks(stbi_io_callbacks const * clbk, void * user, int * x, int * y, int * channels_in_file,
                                          int desired_channels);

// 如果没有定义 STBI_NO_STDIO，则从文件名加载 HDR 图像数据，返回浮点数指针
STBIDEF float * stbi_loadf(char const * filename, int * x, int * y, int * channels_in_file, int desired_channels);
// 如果没有定义 STBI_NO_STDIO，则从文件指针加载 HDR 图像数据，返回浮点数指针
STBIDEF float * stbi_loadf_from_file(FILE * f, int * x, int * y, int * channels_in_file, int desired_channels);

// 如果没有定义 STBI_NO_HDR，则设置 HDR 到 LDR 的 gamma 值
STBIDEF void stbi_hdr_to_ldr_gamma(float gamma);
// 如果没有定义 STBI_NO_HDR，则设置 HDR 到 LDR 的缩放比例
STBIDEF void stbi_hdr_to_ldr_scale(float scale);

// 如果没有定义 STBI_NO_LINEAR，则设置 LDR 到 HDR 的 gamma 值
STBIDEF void stbi_ldr_to_hdr_gamma(float gamma);
// 如果没有定义 STBI_NO_LINEAR，则设置 LDR 到 HDR 的缩放比例
STBIDEF void stbi_ldr_to_hdr_scale(float scale);

// 从回调函数中判断是否为 HDR 图像，返回整数
STBIDEF int stbi_is_hdr_from_callbacks(stbi_io_callbacks const * clbk, void * user);
// 从内存中判断是否为 HDR 图像，返回整数
STBIDEF int stbi_is_hdr_from_memory(stbi_uc const * buffer, int len);
// 如果没有定义 STBI_NO_STDIO，则从文件名判断是否为 HDR 图像，返回整数
STBIDEF int stbi_is_hdr(char const * filename);
// 如果没有定义 STBI_NO_STDIO，则从文件指针判断是否为 HDR 图像，返回整数
STBIDEF int stbi_is_hdr_from_file(FILE * f);

// 获取失败的简要原因，返回常量字符指针
STBIDEF const char * stbi_failure_reason(void);

// 释放加载的图像，使用 free() 函数
STBIDEF void stbi_image_free(void * retval_from_stbi_load);

// 获取图像的尺寸和通道数，不完全解码，返回整数
STBIDEF int stbi_info_from_memory(stbi_uc const * buffer, int len, int * x, int * y, int * comp);
// 从回调函数中获取图像的尺寸和通道数，不完全解码，返回整数
STBIDEF int stbi_info_from_callbacks(stbi_io_callbacks const * clbk, void * user, int * x, int * y, int * comp);
// 从内存中判断是否为 16 位图像，返回整数
STBIDEF int stbi_is_16_bit_from_memory(stbi_uc const * buffer, int len);
// 从回调函数中判断是否为 16 位图像，返回整数
STBIDEF int stbi_is_16_bit_from_callbacks(stbi_io_callbacks const * clbk, void * user);

// 如果没有定义 STBI_NO_STDIO，则从文件名获取图像的尺寸和通道数，不完全解码，返回整数
STBIDEF int stbi_info(char const * filename, int * x, int * y, int * comp);
// 如果没有定义 STBI_NO_STDIO，则从文件指针获取图像的尺寸和通道数，不完全解码，返回整数
STBIDEF int stbi_info_from_file(FILE * f, int * x, int * y, int * comp);
# 检查文件是否为16位图像
STBIDEF int stbi_is_16_bit(char const * filename);
# 从文件指针检查文件是否为16位图像
STBIDEF int stbi_is_16_bit_from_file(FILE * f);
#endif

# 设置是否在加载时取消预乘
STBIDEF void stbi_set_unpremultiply_on_load(int flag_true_if_should_unpremultiply);

# 指示是否应将 iPhone 图像转换为标准格式
STBIDEF void stbi_convert_iphone_png_to_rgb(int flag_true_if_should_convert);

# 垂直翻转图像，使输出数组中的第一个像素为左下角
STBIDEF void stbi_set_flip_vertically_on_load(int flag_true_if_should_flip);

# 与上述相同，但仅适用于调用该函数的线程加载的图像
# 如果编译器支持线程局部变量，则此函数仅在您的编译器支持线程局部变量时可用；
# 如果编译器不支持线程局部变量，则调用它将无法链接
STBIDEF void stbi_set_unpremultiply_on_load_thread(int flag_true_if_should_unpremultiply);
STBIDEF void stbi_convert_iphone_png_to_rgb_thread(int flag_true_if_should_convert);
STBIDEF void stbi_set_flip_vertically_on_load_thread(int flag_true_if_should_flip);

# ZLIB 客户端 - 由 PNG 使用，也可用于其他目的

# 猜测解码后的大小并分配内存
STBIDEF char * stbi_zlib_decode_malloc_guesssize(const char * buffer, int len, int initial_size, int * outlen);
# 猜测解码后的大小并分配内存，带有标志头
STBIDEF char * stbi_zlib_decode_malloc_guesssize_headerflag(const char * buffer, int len, int initial_size, int * outlen,
                                                            int parse_header);
# 分配内存并解码
STBIDEF char * stbi_zlib_decode_malloc(const char * buffer, int len, int * outlen);
# 解码到缓冲区
STBIDEF int stbi_zlib_decode_buffer(char * obuffer, int olen, const char * ibuffer, int ilen);

# 无头部解码并分配内存
STBIDEF char * stbi_zlib_decode_noheader_malloc(const char * buffer, int len, int * outlen);
// 定义了一个函数，用于解压缩没有头部信息的缓冲区
STBIDEF int stbi_zlib_decode_noheader_buffer(char * obuffer, int olen, const char * ibuffer, int ilen);

#ifdef __cplusplus
}
#endif

//
//
////   end header file   /////////////////////////////////////////////////////
#endif // STBI_INCLUDE_STB_IMAGE_H

#ifdef STB_IMAGE_IMPLEMENTATION

#if defined(STBI_ONLY_JPEG) || defined(STBI_ONLY_PNG) || defined(STBI_ONLY_BMP) || defined(STBI_ONLY_TGA) ||                   \
    defined(STBI_ONLY_GIF) || defined(STBI_ONLY_PSD) || defined(STBI_ONLY_HDR) || defined(STBI_ONLY_PIC) ||                    \
    defined(STBI_ONLY_PNM) || defined(STBI_ONLY_ZLIB)
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

#include <limits.h>
#include <stdarg.h>
#include <stddef.h> // ptrdiff_t on osx
#include <stdlib.h>
#include <string.h>

#if !defined(STBI_NO_LINEAR) || !defined(STBI_NO_HDR)
#include <math.h> // ldexp, pow
#endif

#ifndef STBI_NO_STDIO
#include <stdio.h>
#endif

#ifndef STBI_ASSERT
#include <assert.h>
#define STBI_ASSERT(x) assert(x)
#endif

#ifdef __cplusplus
#define STBI_EXTERN extern "C"
#else
#define STBI_EXTERN extern
#endif

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
#if defined(__cplusplus) && __cplusplus >= 201103L
#define STBI_THREAD_LOCAL thread_local
#elif defined(__GNUC__) && __GNUC__ < 5
#define STBI_THREAD_LOCAL __thread
#elif defined(_MSC_VER)
// 定义 STBI_THREAD_LOCAL 宏，用于线程局部存储的声明
#define STBI_THREAD_LOCAL __declspec(thread)
// 如果是 C11 标准及以上，并且不是无线程环境，则使用 _Thread_local
#elif defined(__STDC_VERSION__) && __STDC_VERSION__ >= 201112L && !defined(__STDC_NO_THREADS__)
#define STBI_THREAD_LOCAL _Thread_local
#endif

// 如果没有定义 STBI_THREAD_LOCAL，则根据不同的编译器定义不同的线程局部存储方式
#ifndef STBI_THREAD_LOCAL
// 如果是 GCC 编译器，则使用 __thread
#if defined(__GNUC__)
#define STBI_THREAD_LOCAL __thread
#endif
#endif
#endif

// 根据不同的编译器和平台定义不同大小的整型数据类型
#if defined(_MSC_VER) || defined(__SYMBIAN32__)
typedef unsigned short stbi__uint16;
typedef signed short stbi__int16;
typedef unsigned int stbi__uint32;
typedef signed int stbi__int32;
// 如果不是以上平台，则包含 stdint.h 头文件，使用标准的整型数据类型
#else
#include <stdint.h>
typedef uint16_t stbi__uint16;
typedef int16_t stbi__int16;
typedef uint32_t stbi__uint32;
typedef int32_t stbi__int32;
#endif

// 如果大小不正确，则产生编译错误
typedef unsigned char validate_uint32[sizeof(stbi__uint32) == 4 ? 1 : -1];

// 根据不同的编译器定义 STBI_NOTUSED 宏
#ifdef _MSC_VER
#define STBI_NOTUSED(v) (void)(v)
#else
#define STBI_NOTUSED(v) (void)sizeof(v)
#endif

// 如果是 MSC 编译器，则定义 STBI_HAS_LROTL 宏
#ifdef _MSC_VER
#define STBI_HAS_LROTL
#endif

// 根据不同的宏定义，定义 stbi_lrot 宏
#ifdef STBI_HAS_LROTL
#define stbi_lrot(x, y) _lrotl(x, y)
#else
#define stbi_lrot(x, y) (((x) << (y)) | ((x) >> (-(y)&31)))
#endif

// 如果定义了 STBI_MALLOC 和 STBI_FREE，并且定义了 STBI_REALLOC 或 STBI_REALLOC_SIZED，则通过
// 否则，如果没有定义 STBI_MALLOC 和 STBI_FREE，并且没有定义 STBI_REALLOC 和 STBI_REALLOC_SIZED，则通过
// 否则，产生编译错误
#if defined(STBI_MALLOC) && defined(STBI_FREE) && (defined(STBI_REALLOC) || defined(STBI_REALLOC_SIZED))
// ok
#elif !defined(STBI_MALLOC) && !defined(STBI_FREE) && !defined(STBI_REALLOC) && !defined(STBI_REALLOC_SIZED)
// ok
#else
#error "Must define all or none of STBI_MALLOC, STBI_FREE, and STBI_REALLOC (or STBI_REALLOC_SIZED)."
#endif

// 如果没有定义 STBI_MALLOC，则定义 STBI_MALLOC、STBI_REALLOC 和 STBI_FREE
#ifndef STBI_MALLOC
#define STBI_MALLOC(sz) malloc(sz)
#define STBI_REALLOC(p, newsz) realloc(p, newsz)
#define STBI_FREE(p) free(p)
#endif

// 如果没有定义 STBI_REALLOC_SIZED，则定义 STBI_REALLOC_SIZED 为 STBI_REALLOC
#ifndef STBI_REALLOC_SIZED
#define STBI_REALLOC_SIZED(p, oldsz, newsz) STBI_REALLOC(p, newsz)
#endif

// 检测 x86/x64 架构
#if defined(__x86_64__) || defined(_M_X64)
#define STBI__X64_TARGET
#elif defined(__i386) || defined(_M_IX86)
#define STBI__X86_TARGET
#endif

// 如果是 GCC 编译器，并且是 x86 架构，并且没有禁用 SIMD，则使用 SSE2 指令集
#if defined(__GNUC__) && defined(STBI__X86_TARGET) && !defined(__SSE2__) && !defined(STBI_NO_SIMD)
// gcc doesn't support sse2 intrinsics unless you compile with -msse2,
// 如果未定义 STBI_NO_SIMD 并且目标架构是 x86 或 x64，则定义 STBI_SSE2
#if !defined(STBI_NO_SIMD) && (defined(STBI__X86_TARGET) || defined(STBI__X64_TARGET))
#define STBI_SSE2
// 包含 SSE2 指令集
#include <emmintrin.h>

#ifdef _MSC_VER

#if _MSC_VER >= 1400 // not VC6
#include <intrin.h>  // __cpuid
// 定义一个函数，用于获取 CPU 的信息
static int stbi__cpuid3(void) {
    int info[4];
    __cpuid(info, 1);
    return info[3];
}
#else
// 定义一个函数，用于获取 CPU 的信息
static int stbi__cpuid3(void) {
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

// 如果未定义 STBI_NO_JPEG 并且定义了 STBI_SSE2，则定义一个函数，用于检查 SSE2 是否可用
#if !defined(STBI_NO_JPEG) && defined(STBI_SSE2)
static int stbi__sse2_available(void) {
    int info3 = stbi__cpuid3();
    return ((info3 >> 26) & 1) != 0;
}
#endif

#else // assume GCC-style if not VC++
// 定义一个宏，用于指定变量的内存对齐方式为 16 字节
#define STBI_SIMD_ALIGN(type, name) type name __attribute__((aligned(16)))
// 如果未定义 STBI_NO_JPEG 并且定义了 STBI_SSE2，则定义 stbi__sse2_available 函数
static int stbi__sse2_available(void) {
    // 如果我们在 GCC/Clang 上尝试编译这个代码，那意味着 -msse2 已经开启，这意味着编译器可以随意使用 SSE2 指令，我们也可以使用
    return 1;
}

// 如果定义了 STBI_NO_SIMD 并且定义了 STBI_NEON，则取消定义 STBI_NEON
#if defined(STBI_NO_SIMD) && defined(STBI_NEON)
#undef STBI_NEON
#endif

// 如果定义了 STBI_NEON，则包含 arm_neon.h 头文件，并根据不同的编译器定义 STBI_SIMD_ALIGN 宏
#ifdef STBI_NEON
#include <arm_neon.h>
#ifdef _MSC_VER
#define STBI_SIMD_ALIGN(type, name) __declspec(align(16)) type name
#else
#define STBI_SIMD_ALIGN(type, name) type name __attribute__((aligned(16)))
#endif
#endif

// 如果未定义 STBI_SIMD_ALIGN，则定义 STBI_SIMD_ALIGN 宏
#ifndef STBI_SIMD_ALIGN
#define STBI_SIMD_ALIGN(type, name) type name
#endif

// 如果未定义 STBI_MAX_DIMENSIONS，则定义 STBI_MAX_DIMENSIONS 宏
#ifndef STBI_MAX_DIMENSIONS
#define STBI_MAX_DIMENSIONS (1 << 24)
#endif

// stbi__context 结构体和 start_xxx 函数

// stbi__context 结构体是所有图像使用的基本上下文，因此它包含所有 IO 上下文，以及一些基本的图像信息
typedef struct {
    stbi__uint32 img_x, img_y;
    int img_n, img_out_n;

    stbi_io_callbacks io;
    void * io_user_data;

    int read_from_callbacks;
    int buflen;
    stbi_uc buffer_start[128];
    int callback_already_read;

    stbi_uc *img_buffer, *img_buffer_end;
    stbi_uc *img_buffer_original, *img_buffer_original_end;
} stbi__context;

// 填充缓冲区的函数
static void stbi__refill_buffer(stbi__context * s);

// 初始化内存解码上下文
static void stbi__start_mem(stbi__context * s, stbi_uc const * buffer, int len) {
    s->io.read = NULL;
    s->read_from_callbacks = 0;
    s->callback_already_read = 0;
    s->img_buffer = s->img_buffer_original = (stbi_uc *)buffer;
    s->img_buffer_end = s->img_buffer_original_end = (stbi_uc *)buffer + len;
}

// 初始化基于回调的上下文
static void stbi__start_callbacks(stbi__context * s, stbi_io_callbacks * c, void * user) {
    s->io = *c;
    s->io_user_data = user;
    s->buflen = sizeof(s->buffer_start);
}
    # 设置读取回调函数标志为1，表示将使用回调函数进行读取
    s->read_from_callbacks = 1;
    # 设置回调函数已经读取的标志为0
    s->callback_already_read = 0;
    # 将图像缓冲区指针指向原始图像缓冲区的起始位置
    s->img_buffer = s->img_buffer_original = s->buffer_start;
    # 调用stbi__refill_buffer函数，填充图像缓冲区
    stbi__refill_buffer(s);
    # 将原始图像缓冲区的结束位置赋给img_buffer_original_end
    s->img_buffer_original_end = s->img_buffer_end;
}

#ifndef STBI_NO_STDIO

// 从标准输入流中读取数据
static int stbi__stdio_read(void * user, char * data, int size) { return (int)fread(data, 1, size, (FILE *)user); }

// 从标准输入流中跳过指定字节数
static void stbi__stdio_skip(void * user, int n) {
    int ch;
    fseek((FILE *)user, n, SEEK_CUR);
    ch = fgetc((FILE *)user); /* have to read a byte to reset feof()'s flag */
    if (ch != EOF) {
        ungetc(ch, (FILE *)user); /* push byte back onto stream if valid. */
    }
}

// 检查标准输入流是否已经结束
static int stbi__stdio_eof(void * user) { return feof((FILE *)user) || ferror((FILE *)user); }

// 定义标准输入流的回调函数
static stbi_io_callbacks stbi__stdio_callbacks = {
    stbi__stdio_read,
    stbi__stdio_skip,
    stbi__stdio_eof,
};

// 从文件开始读取数据
static void stbi__start_file(stbi__context * s, FILE * f) { stbi__start_callbacks(s, &stbi__stdio_callbacks, (void *)f); }

// 重新定位输入流到初始位置
static void stbi__rewind(stbi__context * s) {
    // 从概念上讲，rewind应该将流倒回到开头，但我们只将其倒回到初始缓冲区的开头，因为我们只在'test'之后使用它，而'test'只查看最多92个字节
    s->img_buffer = s->img_buffer_original;
    s->img_buffer_end = s->img_buffer_original_end;
}

// 定义颜色通道的顺序
enum { STBI_ORDER_RGB, STBI_ORDER_BGR };

// 定义图像信息结构体
typedef struct {
    int bits_per_channel;
    int num_channels;
    int channel_order;
} stbi__result_info;

#ifndef STBI_NO_JPEG
// JPEG格式的测试函数
static int stbi__jpeg_test(stbi__context * s);
// 从JPEG格式加载图像数据
static void * stbi__jpeg_load(stbi__context * s, int * x, int * y, int * comp, int req_comp, stbi__result_info * ri);
// 获取JPEG格式图像信息
static int stbi__jpeg_info(stbi__context * s, int * x, int * y, int * comp);
#endif

#ifndef STBI_NO_PNG
// PNG格式的测试函数
static int stbi__png_test(stbi__context * s);
// 从PNG格式加载图像数据
static void * stbi__png_load(stbi__context * s, int * x, int * y, int * comp, int req_comp, stbi__result_info * ri);
// 获取PNG格式图像信息
static int stbi__png_info(stbi__context * s, int * x, int * y, int * comp);
// 检查PNG格式是否使用16位深度
static int stbi__png_is16(stbi__context * s);
#endif

#ifndef STBI_NO_BMP
# 检测 BMP 文件格式
static int stbi__bmp_test(stbi__context * s);

# 加载 BMP 文件内容
static void * stbi__bmp_load(stbi__context * s, int * x, int * y, int * comp, int req_comp, stbi__result_info * ri);

# 获取 BMP 文件信息
static int stbi__bmp_info(stbi__context * s, int * x, int * y, int * comp);
#endif

# 如果未定义 STBI_NO_TGA，则执行以下代码
#ifndef STBI_NO_TGA
# 检测 TGA 文件格式
static int stbi__tga_test(stbi__context * s);

# 加载 TGA 文件内容
static void * stbi__tga_load(stbi__context * s, int * x, int * y, int * comp, int req_comp, stbi__result_info * ri);

# 获取 TGA 文件信息
static int stbi__tga_info(stbi__context * s, int * x, int * y, int * comp);
#endif

# 如果未定义 STBI_NO_PSD，则执行以下代码
#ifndef STBI_NO_PSD
# 检测 PSD 文件格式
static int stbi__psd_test(stbi__context * s);

# 加载 PSD 文件内容
static void * stbi__psd_load(stbi__context * s, int * x, int * y, int * comp, int req_comp, stbi__result_info * ri, int bpc);

# 获取 PSD 文件信息
static int stbi__psd_info(stbi__context * s, int * x, int * y, int * comp);

# 判断 PSD 文件是否为16位
static int stbi__psd_is16(stbi__context * s);
#endif

# 如果未定义 STBI_NO_HDR，则执行以下代码
#ifndef STBI_NO_HDR
# 检测 HDR 文件格式
static int stbi__hdr_test(stbi__context * s);

# 加载 HDR 文件内容
static float * stbi__hdr_load(stbi__context * s, int * x, int * y, int * comp, int req_comp, stbi__result_info * ri);

# 获取 HDR 文件信息
static int stbi__hdr_info(stbi__context * s, int * x, int * y, int * comp);
#endif

# 如果未定义 STBI_NO_PIC，则执行以下代码
#ifndef STBI_NO_PIC
# 检测 PIC 文件格式
static int stbi__pic_test(stbi__context * s);

# 加载 PIC 文件内容
static void * stbi__pic_load(stbi__context * s, int * x, int * y, int * comp, int req_comp, stbi__result_info * ri);

# 获取 PIC 文件信息
static int stbi__pic_info(stbi__context * s, int * x, int * y, int * comp);
#endif

# 如果未定义 STBI_NO_GIF，则执行以下代码
#ifndef STBI_NO_GIF
# 检测 GIF 文件格式
static int stbi__gif_test(stbi__context * s);

# 加载 GIF 文件内容
static void * stbi__gif_load(stbi__context * s, int * x, int * y, int * comp, int req_comp, stbi__result_info * ri);

# 加载 GIF 主要内容
static void * stbi__load_gif_main(stbi__context * s, int ** delays, int * x, int * y, int * z, int * comp, int req_comp);

# 获取 GIF 文件信息
static int stbi__gif_info(stbi__context * s, int * x, int * y, int * comp);
#endif

# 如果未定义 STBI_NO_PNM，则执行以下代码
#ifndef STBI_NO_PNM
# 检测 PNM 文件格式
static int stbi__pnm_test(stbi__context * s);

# 加载 PNM 文件内容
static void * stbi__pnm_load(stbi__context * s, int * x, int * y, int * comp, int req_comp, stbi__result_info * ri);
// 定义静态函数，用于获取 PNM 格式图片的信息
static int stbi__pnm_info(stbi__context * s, int * x, int * y, int * comp);
// 定义静态函数，用于检查 PNM 格式图片是否为16位
static int stbi__pnm_is16(stbi__context * s);
#endif

// 定义一个指向失败原因的指针，用于记录失败的原因
static
#ifdef STBI_THREAD_LOCAL
    STBI_THREAD_LOCAL
#endif
    const char * stbi__g_failure_reason;

// 返回失败原因的函数
STBIDEF const char * stbi_failure_reason(void) { return stbi__g_failure_reason; }

#ifndef STBI_NO_FAILURE_STRINGS
// 定义一个函数，用于设置失败原因并返回0
static int stbi__err(const char * str) {
    stbi__g_failure_reason = str;
    return 0;
}
#endif

// 定义一个函数，用于分配内存
static void * stbi__malloc(size_t size) { return STBI_MALLOC(size); }

// stb_image 广泛使用 int 类型，包括偏移计算。
// 因此，即使在64位目标上，我们当前的代码也只支持最大解码图像大小为INT_MAX。
// 对于预期的用例，这不是一个重大限制。
//
// 但是，我们需要确保我们的大小计算不会溢出。因此，我们需要一些辅助函数来进行大小计算，以确保它们是非负的并且不会溢出。

// 如果两个整数相加的结果有效，则返回1，溢出返回0。
// 负数被视为无效。
static int stbi__addsizes_valid(int a, int b) {
    if (b < 0)
        return 0;
    // 现在0 <= b <= INT_MAX，因此也有0 <= INT_MAX - b <= INTMAX。
    // 并且 "a + b <= INT_MAX"（可能会溢出）与 a <= INT_MAX - b（不会溢出）是相同的
    return a <= INT_MAX - b;
}

// 如果两个整数相乘的结果有效，则返回1，溢出返回0。
// 负数被视为无效。
static int stbi__mul2sizes_valid(int a, int b) {
    if (a < 0 || b < 0)
        return 0;
    if (b == 0)
        return 1; // 乘以0总是安全的
    // 检查a*b是否不会溢出的便携方式
    return a <= INT_MAX / b;
}

#if !defined(STBI_NO_JPEG) || !defined(STBI_NO_PNG) || !defined(STBI_NO_TGA) || !defined(STBI_NO_HDR)
// 如果 "a*b + add" 没有负项/因子并且不会溢出，则返回1
static int stbi__mad2sizes_valid(int a, int b, int add) {
    # 返回两个大小是否有效的乘法结果，并且返回两个大小相乘再加上另一个大小是否有效的结果
    return stbi__mul2sizes_valid(a, b) && stbi__addsizes_valid(a * b, add);
// 结束条件判断，如果定义了 STBI_NO_LINEAR、STBI_NO_HDR、STBI_NO_PNM 中的任意一个，则返回 1
#endif

// 如果 "a*b*c + add" 没有负项/因子并且不会溢出，则返回 1
static int stbi__mad3sizes_valid(int a, int b, int c, int add) {
    return stbi__mul2sizes_valid(a, b) && stbi__mul2sizes_valid(a * b, c) && stbi__addsizes_valid(a * b * c, add);
}

// 如果 "a*b*c*d + add" 没有负项/因子并且不会溢出，则返回 1
#if !defined(STBI_NO_LINEAR) || !defined(STBI_NO_HDR) || !defined(STBI_NO_PNM)
static int stbi__mad4sizes_valid(int a, int b, int c, int d, int add) {
    return stbi__mul2sizes_valid(a, b) && stbi__mul2sizes_valid(a * b, c) && stbi__mul2sizes_valid(a * b * c, d) &&
           stbi__addsizes_valid(a * b * c * d, add);
}
#endif

#if !defined(STBI_NO_JPEG) || !defined(STBI_NO_PNG) || !defined(STBI_NO_TGA) || !defined(STBI_NO_HDR)
// 使用大小溢出检查的 malloc
static void * stbi__malloc_mad2(int a, int b, int add) {
    if (!stbi__mad2sizes_valid(a, b, add))
        return NULL;
    return stbi__malloc(a * b + add);
}
#endif

// 使用大小溢出检查的 malloc
static void * stbi__malloc_mad3(int a, int b, int c, int add) {
    if (!stbi__mad3sizes_valid(a, b, c, add))
        return NULL;
    return stbi__malloc(a * b * c + add);
}

#if !defined(STBI_NO_LINEAR) || !defined(STBI_NO_HDR) || !defined(STBI_NO_PNM)
// 使用大小溢出检查的 malloc
static void * stbi__malloc_mad4(int a, int b, int c, int d, int add) {
    if (!stbi__mad4sizes_valid(a, b, c, d, add))
        return NULL;
    return stbi__malloc(a * b * c * d + add);
}
#endif

// 如果两个有符号整数的和有效（在 -2^31 到 2^31-1 之间，包括边界），则返回 1，溢出返回 0
static int stbi__addints_valid(int a, int b) {
    if ((a >= 0) != (b >= 0))
        return 1; // a 和 b 有不同的符号，所以不会溢出
    if (a < 0 && b < 0)
        return a >= INT_MIN - b; // 相当于 a + b >= INT_MIN；INT_MIN - b 不会溢出，因为 b < 0。
    return a <= INT_MAX - b;
}

// 如果两个有符号短整数的乘积有效，则返回 1，溢出返回 0
static int stbi__mul2shorts_valid(short a, short b) {
    # 如果 b 等于 0 或者 -1，则返回 1；乘以 0 总是得到 0；检查 -1 是为了避免 SHRT_MIN/b 溢出
    if (b == 0 || b == -1)
        return 1;
    # 如果 a 和 b 同号，则返回 a 是否小于等于 SHRT_MAX/b；乘积为正数，所以类似于 mul2sizes_valid
    if ((a >= 0) == (b >= 0))
        return a <= SHRT_MAX / b;
    # 如果 b 小于 0，则返回 a 是否小于等于 SHRT_MIN/b；与 a * b >= SHRT_MIN 相同
    if (b < 0)
        return a <= SHRT_MIN / b;
    # 其他情况返回 a 是否大于等于 SHRT_MIN/b
    return a >= SHRT_MIN / b;
// 定义了三个宏，用于处理错误信息
// 如果定义了 STBI_NO_FAILURE_STRINGS，则 stbi__err 返回 0
// 如果定义了 STBI_FAILURE_USERMSG，则 stbi__err 返回 stbi__err(y)
// 否则，stbi__err 返回 stbi__err(x)
#ifdef STBI_NO_FAILURE_STRINGS
#define stbi__err(x, y) 0
#elif defined(STBI_FAILURE_USERMSG)
#define stbi__err(x, y) stbi__err(y)
#else
#define stbi__err(x, y) stbi__err(x)
#endif

// 定义了两个宏，用于处理错误信息并返回指向浮点数的指针
#define stbi__errpf(x, y) ((float *)(size_t)(stbi__err(x, y) ? NULL : NULL))
#define stbi__errpuc(x, y) ((unsigned char *)(size_t)(stbi__err(x, y) ? NULL : NULL))

// 释放由 stbi_load 返回的内存
STBIDEF void stbi_image_free(void * retval_from_stbi_load) { STBI_FREE(retval_from_stbi_load); }

// 如果没有定义 STBI_NO_LINEAR，则定义了 stbi__ldr_to_hdr 函数
#ifndef STBI_NO_LINEAR
static float * stbi__ldr_to_hdr(stbi_uc * data, int x, int y, int comp);
#endif

// 如果没有定义 STBI_NO_HDR，则定义了 stbi__hdr_to_ldr 函数
#ifndef STBI_NO_HDR
static stbi_uc * stbi__hdr_to_ldr(float * data, int x, int y, int comp);
#endif

// 定义了一个静态变量 stbi__vertically_flip_on_load_global，并初始化为 0
static int stbi__vertically_flip_on_load_global = 0;

// 设置是否在加载时垂直翻转图像
STBIDEF void stbi_set_flip_vertically_on_load(int flag_true_if_should_flip) {
    stbi__vertically_flip_on_load_global = flag_true_if_should_flip;
}

// 如果没有定义 STBI_THREAD_LOCAL，则使用全局变量 stbi__vertically_flip_on_load_global
#ifndef STBI_THREAD_LOCAL
#define stbi__vertically_flip_on_load stbi__vertically_flip_on_load_global
// 否则，使用线程本地变量 stbi__vertically_flip_on_load_local
#else
static STBI_THREAD_LOCAL int stbi__vertically_flip_on_load_local, stbi__vertically_flip_on_load_set;

// 设置是否在加载时垂直翻转图像（线程本地）
STBIDEF void stbi_set_flip_vertically_on_load_thread(int flag_true_if_should_flip) {
    stbi__vertically_flip_on_load_local = flag_true_if_should_flip;
    stbi__vertically_flip_on_load_set = 1;
}

// 定义了一个宏，根据是否设置了线程本地变量来决定使用全局变量还是线程本地变量
#define stbi__vertically_flip_on_load                                                                                          \
    (stbi__vertically_flip_on_load_set ? stbi__vertically_flip_on_load_local : stbi__vertically_flip_on_load_global)
#endif // STBI_THREAD_LOCAL

// 定义了 stbi__load_main 函数，用于加载图像数据
static void * stbi__load_main(stbi__context * s, int * x, int * y, int * comp, int req_comp, stbi__result_info * ri, int bpc) {
    memset(ri, 0, sizeof(*ri));         // 确保初始化，以防我们添加新字段
    ri->bits_per_channel = 8;           // 默认为 8，因此大多数路径不必更改
    # 设置通道顺序为RGB，因为当前所有的输入和输出都是这个顺序，但是这里可以添加BGR顺序
    ri->channel_order = STBI_ORDER_RGB; 
    # 设置通道数为0
    ri->num_channels = 0;
// 首先测试具有非常明确头部的格式（至少首先是 FOURCC 或独特的魔术数字）
#ifndef STBI_NO_PNG
    // 如果是 PNG 格式，则调用 stbi__png_load 函数加载并返回图像数据
    if (stbi__png_test(s))
        return stbi__png_load(s, x, y, comp, req_comp, ri);
#endif
#ifndef STBI_NO_BMP
    // 如果是 BMP 格式，则调用 stbi__bmp_load 函数加载并返回图像数据
    if (stbi__bmp_test(s))
        return stbi__bmp_load(s, x, y, comp, req_comp, ri);
#endif
#ifndef STBI_NO_GIF
    // 如果是 GIF 格式，则调用 stbi__gif_load 函数加载并返回图像数据
    if (stbi__gif_test(s))
        return stbi__gif_load(s, x, y, comp, req_comp, ri);
#endif
#ifndef STBI_NO_PSD
    // 如果是 PSD 格式，则调用 stbi__psd_load 函数加载并返回图像数据
    if (stbi__psd_test(s))
        return stbi__psd_load(s, x, y, comp, req_comp, ri, bpc);
#else
    // 如果不是 PSD 格式，则标记 bpc 为未使用
    STBI_NOTUSED(bpc);
#endif
#ifndef STBI_NO_PIC
    // 如果是 PIC 格式，则调用 stbi__pic_load 函数加载并返回图像数据
    if (stbi__pic_test(s))
        return stbi__pic_load(s, x, y, comp, req_comp, ri);
#endif

// 然后测试可能最终尝试仅使用 1 或 2 个字节匹配预期的格式；这些容易产生误报，所以稍后再尝试
#ifndef STBI_NO_JPEG
    // 如果是 JPEG 格式，则调用 stbi__jpeg_load 函数加载并返回图像数据
    if (stbi__jpeg_test(s))
        return stbi__jpeg_load(s, x, y, comp, req_comp, ri);
#endif
#ifndef STBI_NO_PNM
    // 如果是 PNM 格式，则调用 stbi__pnm_load 函数加载并返回图像数据
    if (stbi__pnm_test(s))
        return stbi__pnm_load(s, x, y, comp, req_comp, ri);
#endif

#ifndef STBI_NO_HDR
    // 如果是 HDR 格式，则调用 stbi__hdr_load 函数加载 HDR 图像数据，并将其转换为 LDR 格式
    if (stbi__hdr_test(s)) {
        float * hdr = stbi__hdr_load(s, x, y, comp, req_comp, ri);
        return stbi__hdr_to_ldr(hdr, *x, *y, req_comp ? req_comp : *comp);
    }
#endif

#ifndef STBI_NO_TGA
    // 最后测试 TGA 格式，因为它的测试方法很糟糕！
    // 如果是 TGA 格式，则调用 stbi__tga_load 函数加载并返回图像数据
    if (stbi__tga_test(s))
        return stbi__tga_load(s, x, y, comp, req_comp, ri);
#endif

// 如果不是任何已知类型的图像，或者图像损坏，则返回错误信息
return stbi__errpuc("unknown image type", "Image not of any known type, or corrupt");
}

// 将 16 位图像数据转换为 8 位图像数据
static stbi_uc * stbi__convert_16_to_8(stbi__uint16 * orig, int w, int h, int channels) {
    int i;
    int img_len = w * h * channels;
    stbi_uc * reduced;

    // 分配内存用于存储转换后的图像数据
    reduced = (stbi_uc *)stbi__malloc(img_len);
    // 如果内存分配失败，则返回错误信息
    if (reduced == NULL)
        return stbi__errpuc("outofmem", "Out of memory");
    # 遍历原始数据数组
    for (i = 0; i < img_len; ++i)
        # 对原始数据进行位移和按位与操作，将每个字节的高位截取下来，作为缩小后的数据
        reduced[i] = (stbi_uc)((orig[i] >> 8) & 0xFF); // top half of each byte is sufficient approx of 16->8 bit scaling

    # 释放原始数据的内存空间
    STBI_FREE(orig);
    # 返回缩小后的数据数组
    return reduced;
// 将8位图像数据转换为16位图像数据
static stbi__uint16 * stbi__convert_8_to_16(stbi_uc * orig, int w, int h, int channels) {
    int i;
    int img_len = w * h * channels; // 计算图像数据长度
    stbi__uint16 * enlarged; // 声明一个16位图像数据的指针

    enlarged = (stbi__uint16 *)stbi__malloc(img_len * 2); // 分配内存用于存储16位图像数据
    if (enlarged == NULL)
        return (stbi__uint16 *)stbi__errpuc("outofmem", "Out of memory"); // 如果内存分配失败，则返回错误信息

    for (i = 0; i < img_len; ++i)
        enlarged[i] = (stbi__uint16)((orig[i] << 8) + orig[i]); // 将8位图像数据扩展为16位图像数据，高位和低位相同

    STBI_FREE(orig); // 释放原始8位图像数据的内存
    return enlarged; // 返回16位图像数据
}

// 垂直翻转图像数据
static void stbi__vertical_flip(void * image, int w, int h, int bytes_per_pixel) {
    int row;
    size_t bytes_per_row = (size_t)w * bytes_per_pixel; // 计算每行的字节数
    stbi_uc temp[2048]; // 声明一个临时数组用于交换数据
    stbi_uc * bytes = (stbi_uc *)image; // 将图像数据转换为字节类型指针

    for (row = 0; row < (h >> 1); row++) {
        stbi_uc * row0 = bytes + row * bytes_per_row; // 指向当前行的指针
        stbi_uc * row1 = bytes + (h - row - 1) * bytes_per_row; // 指向对称行的指针
        // 交换row0和row1的数据
        size_t bytes_left = bytes_per_row;
        while (bytes_left) {
            size_t bytes_copy = (bytes_left < sizeof(temp)) ? bytes_left : sizeof(temp); // 计算需要复制的字节数
            memcpy(temp, row0, bytes_copy); // 复制row0的数据到临时数组
            memcpy(row0, row1, bytes_copy); // 将row1的数据复制到row0
            memcpy(row1, temp, bytes_copy); // 将临时数组的数据复制到row1
            row0 += bytes_copy;
            row1 += bytes_copy;
            bytes_left -= bytes_copy;
        }
    }
}

// 如果未定义STBI_NO_GIF，则垂直翻转图像数据的切片
#ifndef STBI_NO_GIF
static void stbi__vertical_flip_slices(void * image, int w, int h, int z, int bytes_per_pixel) {
    int slice;
    int slice_size = w * h * bytes_per_pixel; // 计算每个切片的字节数

    stbi_uc * bytes = (stbi_uc *)image; // 将图像数据转换为字节类型指针
    for (slice = 0; slice < z; ++slice) {
        stbi__vertical_flip(bytes, w, h, bytes_per_pixel); // 对每个切片进行垂直翻转
        bytes += slice_size; // 移动到下一个切片的起始位置
    }
}
#endif

// 加载并后处理8位图像数据
static unsigned char * stbi__load_and_postprocess_8bit(stbi__context * s, int * x, int * y, int * comp, int req_comp) {
    stbi__result_info ri;
    void * result = stbi__load_main(s, x, y, comp, req_comp, &ri, 8); // 调用stbi__load_main函数加载图像数据

    if (result == NULL)
        return NULL; // 如果加载失败，则返回空指针
    // 确保加载器负责确保我们得到8位或16位的数据
    STBI_ASSERT(ri.bits_per_channel == 8 || ri.bits_per_channel == 16);

    // 如果每个通道的位数不是8位，则将16位数据转换为8位数据
    if (ri.bits_per_channel != 8) {
        result = stbi__convert_16_to_8((stbi__uint16 *)result, *x, *y, req_comp == 0 ? *comp : req_comp);
        ri.bits_per_channel = 8;
    }

    // @TODO: 将stbi__convert_format移动到这里

    // 如果在加载时需要垂直翻转图像，则进行翻转操作
    if (stbi__vertically_flip_on_load) {
        int channels = req_comp ? req_comp : *comp;
        stbi__vertical_flip(result, *x, *y, channels * sizeof(stbi_uc));
    }

    // 返回转换后的结果数据
    return (unsigned char *)result;
}

static stbi__uint16 * stbi__load_and_postprocess_16bit(stbi__context * s, int * x, int * y, int * comp, int req_comp) {
    // 加载并进行后处理，返回16位数据
    stbi__result_info ri;
    // 调用stbi__load_main函数加载主要数据
    void * result = stbi__load_main(s, x, y, comp, req_comp, &ri, 16);

    // 如果加载结果为空，则返回空指针
    if (result == NULL)
        return NULL;

    // 加载器负责确保我们得到8位或16位的数据
    STBI_ASSERT(ri.bits_per_channel == 8 || ri.bits_per_channel == 16);

    // 如果通道的位数不是16位，则将8位数据转换为16位数据
    if (ri.bits_per_channel != 16) {
        result = stbi__convert_8_to_16((stbi_uc *)result, *x, *y, req_comp == 0 ? *comp : req_comp);
        ri.bits_per_channel = 16;
    }

    // @TODO: 将stbi__convert_format16移动到这里
    // @TODO: 对于8位转16位的情况，特殊处理RGB到Y（以及RGBA到YA），以保留更多精度

    // 如果在加载时需要垂直翻转，则进行翻转
    if (stbi__vertically_flip_on_load) {
        int channels = req_comp ? req_comp : *comp;
        stbi__vertical_flip(result, *x, *y, channels * sizeof(stbi__uint16));
    }

    // 返回16位数据的指针
    return (stbi__uint16 *)result;
}

#if !defined(STBI_NO_HDR) && !defined(STBI_NO_LINEAR)
static void stbi__float_postprocess(float * result, int * x, int * y, int * comp, int req_comp) {
    // 如果在加载时需要垂直翻转，并且结果不为空，则进行翻转
    if (stbi__vertically_flip_on_load && result != NULL) {
        int channels = req_comp ? req_comp : *comp;
        stbi__vertical_flip(result, *x, *y, channels * sizeof(float));
    }
}
#endif

#ifndef STBI_NO_STDIO

#if defined(_WIN32) && defined(STBI_WINDOWS_UTF8)
// 导入Windows API函数，用于处理UTF-8编码的字符串
STBI_EXTERN __declspec(dllimport) int __stdcall MultiByteToWideChar(unsigned int cp, unsigned long flags, const char * str,
                                                                    int cbmb, wchar_t * widestr, int cchwide);
STBI_EXTERN __declspec(dllimport) int __stdcall WideCharToMultiByte(unsigned int cp, unsigned long flags,
                                                                    const wchar_t * widestr, int cchwide, char * str, int cbmb,
                                                                    const char * defchar, int * used_default);
#endif
#if defined(_WIN32) && defined(STBI_WINDOWS_UTF8)
# 如果定义了_WIN32并且定义了STBI_WINDOWS_UTF8，则执行以下代码
STBIDEF int stbi_convert_wchar_to_utf8(char * buffer, size_t bufferlen, const wchar_t * input) {
    # 将宽字符转换为 UTF-8 编码
    return WideCharToMultiByte(65001 /* UTF8 */, 0, input, -1, buffer, (int)bufferlen, NULL, NULL);
}
#endif

# 打开文件函数，根据文件名和模式返回文件指针
static FILE * stbi__fopen(char const * filename, char const * mode) {
    FILE * f;
    # 如果定义了_WIN32并且定义了STBI_WINDOWS_UTF8，则执行以下代码
#if defined(_WIN32) && defined(STBI_WINDOWS_UTF8)
    wchar_t wMode[64];
    wchar_t wFilename[1024];
    # 将文件名从 UTF-8 转换为宽字符
    if (0 == MultiByteToWideChar(65001 /* UTF8 */, 0, filename, -1, wFilename, sizeof(wFilename) / sizeof(*wFilename)))
        return 0;

    # 将模式从 UTF-8 转换为宽字符
    if (0 == MultiByteToWideChar(65001 /* UTF8 */, 0, mode, -1, wMode, sizeof(wMode) / sizeof(*wMode)))
        return 0;

    # 如果是 MSC 编译器且版本大于等于 1400，则使用 _wfopen_s 函数打开文件
#if defined(_MSC_VER) && _MSC_VER >= 1400
    if (0 != _wfopen_s(&f, wFilename, wMode))
        f = 0;
#else
    # 否则使用 _wfopen 函数打开文件
    f = _wfopen(wFilename, wMode);
#endif

# 如果是 MSC 编译器且版本大于等于 1400，则使用 fopen_s 函数打开文件
#elif defined(_MSC_VER) && _MSC_VER >= 1400
    if (0 != fopen_s(&f, filename, mode))
        f = 0;
# 否则使用 fopen 函数打开文件
#else
    f = fopen(filename, mode);
#endif
    return f;
}

# 从文件加载图像数据，返回无符号字符指针
STBIDEF stbi_uc * stbi_load(char const * filename, int * x, int * y, int * comp, int req_comp) {
    # 打开文件
    FILE * f = stbi__fopen(filename, "rb");
    unsigned char * result;
    # 如果文件打开失败，则返回错误信息
    if (!f)
        return stbi__errpuc("can't fopen", "Unable to open file");
    # 从文件加载图像数据
    result = stbi_load_from_file(f, x, y, comp, req_comp);
    # 关闭文件
    fclose(f);
    return result;
}

# 从文件加载图像数据，返回无符号字符指针
STBIDEF stbi_uc * stbi_load_from_file(FILE * f, int * x, int * y, int * comp, int req_comp) {
    unsigned char * result;
    stbi__context s;
    # 初始化图像上下文
    stbi__start_file(&s, f);
    # 加载并处理 8 位图像数据
    result = stbi__load_and_postprocess_8bit(&s, x, y, comp, req_comp);
    if (result) {
        # 需要将 IO 缓冲区中的所有字符“unget”
        fseek(f, -(int)(s.img_buffer_end - s.img_buffer), SEEK_CUR);
    }
    return result;
}

# 从文件加载 16 位图像数据，返回 16 位无符号整数指针
STBIDEF stbi__uint16 * stbi_load_from_file_16(FILE * f, int * x, int * y, int * comp, int req_comp) {
    stbi__uint16 * result;
    stbi__context s;
    # 初始化图像上下文
    stbi__start_file(&s, f);
    # 调用函数 stbi__load_and_postprocess_16bit，加载并处理 16 位图像数据，将结果保存在 result 中
    result = stbi__load_and_postprocess_16bit(&s, x, y, comp, req_comp);
    # 如果结果不为空（即加载和处理成功）
    if (result) {
        # 需要将 IO 缓冲区中的所有字符都“unget”回去
        # 将文件指针移动到当前位置减去 IO 缓冲区的大小的位置
        fseek(f, -(int)(s.img_buffer_end - s.img_buffer), SEEK_CUR);
    }
    # 返回加载和处理的结果
    return result;
}

STBIDEF stbi_us * stbi_load_16(char const * filename, int * x, int * y, int * comp, int req_comp) {
    // 打开文件
    FILE * f = stbi__fopen(filename, "rb");
    // 定义结果指针
    stbi__uint16 * result;
    // 如果文件打开失败，返回错误信息
    if (!f)
        return (stbi_us *)stbi__errpuc("can't fopen", "Unable to open file");
    // 从文件中加载 16 位数据
    result = stbi_load_from_file_16(f, x, y, comp, req_comp);
    // 关闭文件
    fclose(f);
    // 返回结果指针
    return result;
}

#endif //! STBI_NO_STDIO

STBIDEF stbi_us * stbi_load_16_from_memory(stbi_uc const * buffer, int len, int * x, int * y, int * channels_in_file,
                                           int desired_channels) {
    // 定义上下文
    stbi__context s;
    // 从内存中开始
    stbi__start_mem(&s, buffer, len);
    // 加载并后处理 16 位数据
    return stbi__load_and_postprocess_16bit(&s, x, y, channels_in_file, desired_channels);
}

STBIDEF stbi_us * stbi_load_16_from_callbacks(stbi_io_callbacks const * clbk, void * user, int * x, int * y,
                                              int * channels_in_file, int desired_channels) {
    // 定义上下文
    stbi__context s;
    // 从回调函数中开始
    stbi__start_callbacks(&s, (stbi_io_callbacks *)clbk, user);
    // 加载并后处理 16 位数据
    return stbi__load_and_postprocess_16bit(&s, x, y, channels_in_file, desired_channels);
}

STBIDEF stbi_uc * stbi_load_from_memory(stbi_uc const * buffer, int len, int * x, int * y, int * comp, int req_comp) {
    // 定义上下文
    stbi__context s;
    // 从内存中开始
    stbi__start_mem(&s, buffer, len);
    // 加载并后处理 8 位数据
    return stbi__load_and_postprocess_8bit(&s, x, y, comp, req_comp);
}

STBIDEF stbi_uc * stbi_load_from_callbacks(stbi_io_callbacks const * clbk, void * user, int * x, int * y, int * comp,
                                           int req_comp) {
    // 定义上下文
    stbi__context s;
    // 从回调函数中开始
    stbi__start_callbacks(&s, (stbi_io_callbacks *)clbk, user);
    // 加载并后处理 8 位数据
    return stbi__load_and_postprocess_8bit(&s, x, y, comp, req_comp);
}

#ifndef STBI_NO_GIF
STBIDEF stbi_uc * stbi_load_gif_from_memory(stbi_uc const * buffer, int len, int ** delays, int * x, int * y, int * z,
                                            int * comp, int req_comp) {
    // 定义结果指针
    unsigned char * result;
    // 定义上下文
    stbi__context s;
    // 从内存中开始
    stbi__start_mem(&s, buffer, len);
    // 将指针类型转换为 unsigned char*，调用 stbi__load_gif_main 函数加载 GIF 图像数据，并返回结果
    result = (unsigned char *)stbi__load_gif_main(&s, delays, x, y, z, comp, req_comp);
    // 如果在加载时需要垂直翻转图像，则调用 stbi__vertical_flip_slices 函数进行翻转操作
    if (stbi__vertically_flip_on_load) {
        stbi__vertical_flip_slices(result, *x, *y, *z, *comp);
    }
    // 返回加载后的图像数据
    return result;
}
#endif

#ifndef STBI_NO_LINEAR
// 定义一个函数，用于加载 HDR 图像数据并转换为浮点数格式
static float * stbi__loadf_main(stbi__context * s, int * x, int * y, int * comp, int req_comp) {
    unsigned char * data;
#ifndef STBI_NO_HDR
    // 如果是 HDR 图像，则调用 stbi__hdr_load 函数加载数据
    if (stbi__hdr_test(s)) {
        stbi__result_info ri;
        float * hdr_data = stbi__hdr_load(s, x, y, comp, req_comp, &ri);
        // 如果成功加载 HDR 数据，则进行后处理
        if (hdr_data)
            stbi__float_postprocess(hdr_data, x, y, comp, req_comp);
        return hdr_data;
    }
#endif
    // 如果不是 HDR 图像，则调用 stbi__load_and_postprocess_8bit 函数加载数据
    data = stbi__load_and_postprocess_8bit(s, x, y, comp, req_comp);
    // 如果成功加载数据，则将 LDR 数据转换为 HDR 格式
    if (data)
        return stbi__ldr_to_hdr(data, *x, *y, req_comp ? req_comp : *comp);
    // 如果加载失败，则返回错误信息
    return stbi__errpf("unknown image type", "Image not of any known type, or corrupt");
}

// 从内存中加载 HDR 图像数据
STBIDEF float * stbi_loadf_from_memory(stbi_uc const * buffer, int len, int * x, int * y, int * comp, int req_comp) {
    stbi__context s;
    stbi__start_mem(&s, buffer, len);
    return stbi__loadf_main(&s, x, y, comp, req_comp);
}

// 从回调函数中加载 HDR 图像数据
STBIDEF float * stbi_loadf_from_callbacks(stbi_io_callbacks const * clbk, void * user, int * x, int * y, int * comp,
                                          int req_comp) {
    stbi__context s;
    stbi__start_callbacks(&s, (stbi_io_callbacks *)clbk, user);
    return stbi__loadf_main(&s, x, y, comp, req_comp);
}

#ifndef STBI_NO_STDIO
// 从文件中加载 HDR 图像数据
STBIDEF float * stbi_loadf(char const * filename, int * x, int * y, int * comp, int req_comp) {
    float * result;
    FILE * f = stbi__fopen(filename, "rb");
    // 如果无法打开文件，则返回错误信息
    if (!f)
        return stbi__errpf("can't fopen", "Unable to open file");
    // 调用 stbi_loadf_from_file 函数加载文件中的 HDR 数据
    result = stbi_loadf_from_file(f, x, y, comp, req_comp);
    fclose(f);
    return result;
}

// 从文件指针中加载 HDR 图像数据
STBIDEF float * stbi_loadf_from_file(FILE * f, int * x, int * y, int * comp, int req_comp) {
    stbi__context s;
    stbi__start_file(&s, f);
    return stbi__loadf_main(&s, x, y, comp, req_comp);
}
#endif // !STBI_NO_STDIO

#endif // !STBI_NO_LINEAR

// 这些 is-hdr-or-not 是独立定义的，与 STBI_NO_LINEAR 的定义无关，为了 API 的简单性；
// 如果 STBI_NO_LINEAR 被定义，它总是
// 从内存中检查是否为 HDR 格式的图片
STBIDEF int stbi_is_hdr_from_memory(stbi_uc const * buffer, int len) {
#ifndef STBI_NO_HDR
    // 创建 stbi__context 结构体
    stbi__context s;
    // 从内存中开始读取数据
    stbi__start_mem(&s, buffer, len);
    // 调用 stbi__hdr_test 函数检查是否为 HDR 格式
    return stbi__hdr_test(&s);
#else
    // 如果定义了 STBI_NO_HDR，则不进行 HDR 格式检查
    STBI_NOTUSED(buffer);
    STBI_NOTUSED(len);
    return 0;
#endif
}

// 检查文件是否为 HDR 格式的图片
STBIDEF int stbi_is_hdr(char const * filename) {
    // 打开文件
    FILE * f = stbi__fopen(filename, "rb");
    int result = 0;
    if (f) {
        // 调用 stbi_is_hdr_from_file 函数检查文件是否为 HDR 格式
        result = stbi_is_hdr_from_file(f);
        // 关闭文件
        fclose(f);
    }
    return result;
}

// 从文件中检查是否为 HDR 格式的图片
STBIDEF int stbi_is_hdr_from_file(FILE * f) {
#ifndef STBI_NO_HDR
    // 获取当前文件位置
    long pos = ftell(f);
    int res;
    // 创建 stbi__context 结构体
    stbi__context s;
    // 从文件中开始读取数据
    stbi__start_file(&s, f);
    // 调用 stbi__hdr_test 函数检查是否为 HDR 格式
    res = stbi__hdr_test(&s);
    // 恢复文件位置
    fseek(f, pos, SEEK_SET);
    return res;
#else
    // 如果定义了 STBI_NO_HDR，则不进行 HDR 格式检查
    STBI_NOTUSED(f);
    return 0;
#endif
}

// 从回调函数中检查是否为 HDR 格式的图片
STBIDEF int stbi_is_hdr_from_callbacks(stbi_io_callbacks const * clbk, void * user) {
#ifndef STBI_NO_HDR
    // 创建 stbi__context 结构体
    stbi__context s;
    // 从回调函数中开始读取数据
    stbi__start_callbacks(&s, (stbi_io_callbacks *)clbk, user);
    // 调用 stbi__hdr_test 函数检查是否为 HDR 格式
    return stbi__hdr_test(&s);
#else
    // 如果定义了 STBI_NO_HDR，则不进行 HDR 格式检查
    STBI_NOTUSED(clbk);
    STBI_NOTUSED(user);
    return 0;
#endif
}

// 设置从 LDR 转换到 HDR 的 gamma 值
STBIDEF void stbi_ldr_to_hdr_gamma(float gamma) { stbi__l2h_gamma = gamma; }
// 设置从 LDR 转换到 HDR 的比例尺
STBIDEF void stbi_ldr_to_hdr_scale(float scale) { stbi__l2h_scale = scale; }

// 设置从 HDR 转换到 LDR 的 gamma 值
STBIDEF void stbi_hdr_to_ldr_gamma(float gamma) { stbi__h2l_gamma_i = 1 / gamma; }
// 设置从 HDR 转换到 LDR 的比例尺
STBIDEF void stbi_hdr_to_ldr_scale(float scale) { stbi__h2l_scale_i = 1 / scale; }

// 用于所有图像加载器的通用代码
enum { STBI__SCAN_load = 0, STBI__SCAN_type, STBI__SCAN_header };

// 重新填充缓冲区
static void stbi__refill_buffer(stbi__context * s) {
    // 从输入流中读取数据到缓冲区
    int n = (s->io.read)(s->io_user_data, (char *)s->buffer_start, s->buflen);
    // 更新已读取的回调数据长度
    s->callback_already_read += (int)(s->img_buffer - s->img_buffer_original);
    // 如果 n 等于 0
    if (n == 0) {
        // 在文件末尾，处理方式与从内存中读取相同，但需要处理 s->img_buffer 指向不安全内存的情况，例如 0 字节文件
        s->read_from_callbacks = 0;
        s->img_buffer = s->buffer_start;
        s->img_buffer_end = s->buffer_start + 1;
        *s->img_buffer = 0;
    } else {
        // 否则更新 img_buffer 和 img_buffer_end 的指向位置
        s->img_buffer = s->buffer_start;
        s->img_buffer_end = s->buffer_start + n;
    }
// 定义一个静态内联函数，用于从 stbi__context 结构体中获取一个字节数据
stbi_inline static stbi_uc stbi__get8(stbi__context * s) {
    // 如果图像缓冲区中还有数据，则返回下一个字节数据并将指针向后移动
    if (s->img_buffer < s->img_buffer_end)
        return *s->img_buffer++;
    // 如果使用回调函数读取数据，则重新填充缓冲区并返回下一个字节数据
    if (s->read_from_callbacks) {
        stbi__refill_buffer(s);
        return *s->img_buffer++;
    }
    // 如果没有数据可读，则返回 0
    return 0;
}

// 如果未定义 STBI_NO_JPEG、STBI_NO_HDR、STBI_NO_PIC、STBI_NO_PNM 中的任何一个，则定义一个静态内联函数用于检查是否已到达文件末尾
#if defined(STBI_NO_JPEG) && defined(STBI_NO_HDR) && defined(STBI_NO_PIC) && defined(STBI_NO_PNM)
// 什么也不做
#else
static int stbi__at_eof(stbi__context * s) {
    // 如果使用自定义的读取函数，则调用其 eof 函数判断是否已到达文件末尾
    if (s->io.read) {
        if (!(s->io.eof)(s->io_user_data))
            return 0;
        // 如果 feof() 返回 true，则检查缓冲区是否已经到达末尾
        // 特殊情况：我们只有一个特殊的 0 字符在末尾
        if (s->read_from_callbacks == 0)
            return 1;
    }
    // 如果图像缓冲区已经到达末尾，则返回 1，否则返回 0
    return s->img_buffer >= s->img_buffer_end;
}
#endif

// 如果未定义 STBI_NO_PNG、STBI_NO_TGA、STBI_NO_BMP、STBI_NO_PSD、STBI_NO_TGA、STBI_NO_GIF、STBI_NO_PIC 中的任何一个，则定义一个静态函数用于跳过指定数量的字节
#if defined(STBI_NO_JPEG) && defined(STBI_NO_PNG) && defined(STBI_NO_BMP) && defined(STBI_NO_PSD) && defined(STBI_NO_TGA) &&   \
    defined(STBI_NO_GIF) && defined(STBI_NO_PIC)
// 什么也不做
#else
static void stbi__skip(stbi__context * s, int n) {
    // 如果 n 等于 0，则不做任何操作
    if (n == 0)
        return; // already there!
    // 如果 n 小于 0，则将图像缓冲区指针移动到末尾
    if (n < 0) {
        s->img_buffer = s->img_buffer_end;
        return;
    }
    // 如果使用自定义的读取函数，则检查缓冲区中是否有足够的数据，如果不够则调用其 skip 函数跳过剩余的字节数
    if (s->io.read) {
        int blen = (int)(s->img_buffer_end - s->img_buffer);
        if (blen < n) {
            s->img_buffer = s->img_buffer_end;
            (s->io.skip)(s->io_user_data, n - blen);
            return;
        }
    }
    // 否则直接将图像缓冲区指针向后移动 n 个字节
    s->img_buffer += n;
}
#endif

// 如果未定义 STBI_NO_PNG、STBI_NO_TGA、STBI_NO_HDR、STBI_NO_PNM 中的任何一个，则定义一个静态函数用于从 stbi__context 结构体中获取指定数量的字节数据
#if defined(STBI_NO_PNG) && defined(STBI_NO_TGA) && defined(STBI_NO_HDR) && defined(STBI_NO_PNM)
// 什么也不做
#else
static int stbi__getn(stbi__context * s, stbi_uc * buffer, int n) {
    # 如果存在读取函数
    if (s->io.read) {
        # 计算当前缓冲区中的数据长度
        int blen = (int)(s->img_buffer_end - s->img_buffer);
        # 如果当前缓冲区中的数据长度小于需要读取的数据长度
        if (blen < n) {
            int res, count;
            # 将当前缓冲区中的数据拷贝到目标缓冲区中
            memcpy(buffer, s->img_buffer, blen);
            # 调用读取函数，将剩余数据读取到目标缓冲区中
            count = (s->io.read)(s->io_user_data, (char *)buffer + blen, n - blen);
            # 判断是否成功读取了剩余数据
            res = (count == (n - blen));
            # 更新缓冲区指针
            s->img_buffer = s->img_buffer_end;
            # 返回读取结果
            return res;
        }
    }

    # 如果当前缓冲区中有足够的数据可以读取
    if (s->img_buffer + n <= s->img_buffer_end) {
        # 将数据从当前缓冲区拷贝到目标缓冲区中
        memcpy(buffer, s->img_buffer, n);
        # 更新缓冲区指针
        s->img_buffer += n;
        # 返回读取结果
        return 1;
    } else
        # 如果当前缓冲区中的数据不足以读取，返回读取失败
        return 0;
// 如果定义了 STBI_NO_JPEG、STBI_NO_PNG、STBI_NO_PSD 和 STBI_NO_PIC，则什么也不做
#else
// 定义一个函数，从 stbi__context 结构中读取两个字节，以大端序返回一个整数
static int stbi__get16be(stbi__context * s) {
    int z = stbi__get8(s); // 读取一个字节
    return (z << 8) + stbi__get8(s); // 将两个字节合并成一个整数
}
#endif

// 如果定义了 STBI_NO_PNG、STBI_NO_PSD 和 STBI_NO_PIC，则什么也不做
#else
// 定义一个函数，从 stbi__context 结构中读取四个字节，以大端序返回一个无符号整数
static stbi__uint32 stbi__get32be(stbi__context * s) {
    stbi__uint32 z = stbi__get16be(s); // 读取两个字节
    return (z << 16) + stbi__get16be(s); // 将四个字节合并成一个无符号整数
}
#endif

// 如果定义了 STBI_NO_BMP、STBI_NO_TGA 和 STBI_NO_GIF，则什么也不做
#else
// 定义一个函数，从 stbi__context 结构中读取两个字节，以小端序返回一个整数
static int stbi__get16le(stbi__context * s) {
    int z = stbi__get8(s); // 读取一个字节
    return z + (stbi__get8(s) << 8); // 将两个字节合并成一个整数
}
#endif

// 如果未定义 STBI_NO_BMP，则执行以下代码
#ifndef STBI_NO_BMP
// 定义一个函数，从 stbi__context 结构中读取四个字节，以小端序返回一个无符号整数
static stbi__uint32 stbi__get32le(stbi__context * s) {
    stbi__uint32 z = stbi__get16le(s); // 读取两个字节
    z += (stbi__uint32)stbi__get16le(s) << 16; // 将两个字节合并成一个无符号整数
    return z;
}
#endif

// 定义一个宏，将输入的整数截断为一个字节的无符号整数
#define STBI__BYTECAST(x) ((stbi_uc)((x)&255)) // truncate int to byte without warnings

// 如果定义了 STBI_NO_JPEG、STBI_NO_PNG、STBI_NO_BMP、STBI_NO_PSD、STBI_NO_TGA、STBI_NO_GIF、STBI_NO_PIC 和 STBI_NO_PNM，则什么也不做
#else
// 以下是一个通用的转换函数，将内置的 img_n 转换为 req_comp
// 假设数据缓冲区是通过 malloc 分配的，因此需要重新分配一个新的缓冲区，并释放旧的缓冲区
// 唯一的失败模式是 malloc 失败
static stbi_uc stbi__compute_y(int r, int g, int b) { return (stbi_uc)(((r * 77) + (g * 150) + (29 * b)) >> 8); }
#endif

// 如果定义了 STBI_NO_PNG、STBI_NO_BMP、STBI_NO_PSD、STBI_NO_TGA 和 STBI_NO_GIF，则继续下一行
    # 如果 STBI_NO_PIC 和 STBI_NO_PNM 都被定义，则执行以下代码
// nothing
#else
static unsigned char * stbi__convert_format(unsigned char * data, int img_n, int req_comp, unsigned int x, unsigned int y) {
    int i, j;
    unsigned char * good;

    if (req_comp == img_n)
        return data;
    STBI_ASSERT(req_comp >= 1 && req_comp <= 4);

    good = (unsigned char *)stbi__malloc_mad3(req_comp, x, y, 0);
    if (good == NULL) {
        STBI_FREE(data);
        return stbi__errpuc("outofmem", "Out of memory");
    }

    for (j = 0; j < (int)y; ++j) {
        unsigned char * src = data + j * x * img_n;
        unsigned char * dest = good + j * x * req_comp;

#define STBI__COMBO(a, b) ((a)*8 + (b))  // 定义一个宏，用于将两个值组合成一个值
#define STBI__CASE(a, b)                                                                                                       \
#undef STBI__CASE  // 定义一个宏，用于在switch语句中使用
    }

    STBI_FREE(data);  // 释放原始数据的内存
    return good;  // 返回转换后的数据
}
#endif

#if defined(STBI_NO_PNG) && defined(STBI_NO_PSD)
// nothing
#else
static stbi__uint16 stbi__compute_y_16(int r, int g, int b) { return (stbi__uint16)(((r * 77) + (g * 150) + (29 * b)) >> 8); }
#endif

#if defined(STBI_NO_PNG) && defined(STBI_NO_PSD)
// nothing
#else
static stbi__uint16 * stbi__convert_format16(stbi__uint16 * data, int img_n, int req_comp, unsigned int x, unsigned int y) {
    int i, j;
    stbi__uint16 * good;

    if (req_comp == img_n)
        return data;
    STBI_ASSERT(req_comp >= 1 && req_comp <= 4);

    good = (stbi__uint16 *)stbi__malloc(req_comp * x * y * 2);
    if (good == NULL) {
        STBI_FREE(data);
        return (stbi__uint16 *)stbi__errpuc("outofmem", "Out of memory");
    }

    for (j = 0; j < (int)y; ++j) {
        stbi__uint16 * src = data + j * x * img_n;
        stbi__uint16 * dest = good + j * x * req_comp;

#define STBI__COMBO(a, b) ((a)*8 + (b))  // 定义一个宏，用于将两个值组合成一个值
#define STBI__CASE(a, b)                                                                                                       \
#undef STBI__CASE  // 定义一个宏，用于在switch语句中使用
    }

    STBI_FREE(data);  // 释放原始数据的内存
    return good;  // 返回转换后的数据
}
#endif

#ifndef STBI_NO_LINEAR
static float * stbi__ldr_to_hdr(stbi_uc * data, int x, int y, int comp) {
    int i, k, n;
    float * output;
    if (!data)
        return NULL;  # 如果输入数据为空，则返回空指针
    output = (float *)stbi__malloc_mad4(x, y, comp, sizeof(float), 0);  # 分配内存以存储 HDR 数据
    if (output == NULL) {
        STBI_FREE(data);  # 释放输入数据的内存
        return stbi__errpf("outofmem", "Out of memory");  # 返回内存分配错误
    }
    // compute number of non-alpha components
    if (comp & 1)
        n = comp;  # 计算非 alpha 通道的数量
    else
        n = comp - 1;
    for (i = 0; i < x * y; ++i) {
        for (k = 0; k < n; ++k) {
            output[i * comp + k] = (float)(pow(data[i * comp + k] / 255.0f, stbi__l2h_gamma) * stbi__l2h_scale);  # 将 LDR 数据转换为 HDR 数据
        }
    }
    if (n < comp) {
        for (i = 0; i < x * y; ++i) {
            output[i * comp + n] = data[i * comp + n] / 255.0f;  # 处理 alpha 通道
        }
    }
    STBI_FREE(data);  # 释放输入数据的内存
    return output;  # 返回 HDR 数据
}
#endif

#ifndef STBI_NO_HDR
#define stbi__float2int(x) ((int)(x))
static stbi_uc * stbi__hdr_to_ldr(float * data, int x, int y, int comp) {
    int i, k, n;
    stbi_uc * output;
    if (!data)
        return NULL;  # 如果输入数据为空，则返回空指针
    output = (stbi_uc *)stbi__malloc_mad3(x, y, comp, 0);  # 分配内存以存储 LDR 数据
    if (output == NULL) {
        STBI_FREE(data);  # 释放输入数据的内存
        return stbi__errpuc("outofmem", "Out of memory");  # 返回内存分配错误
    }
    // compute number of non-alpha components
    if (comp & 1)
        n = comp;  # 计算非 alpha 通道的数量
    else
        n = comp - 1;
    for (i = 0; i < x * y; ++i) {
        for (k = 0; k < n; ++k) {
            float z = (float)pow(data[i * comp + k] * stbi__h2l_scale_i, stbi__h2l_gamma_i) * 255 + 0.5f;  # 将 HDR 数据转换为 LDR 数据
            if (z < 0)
                z = 0;
            if (z > 255)
                z = 255;
            output[i * comp + k] = (stbi_uc)stbi__float2int(z);  # 将浮点数转换为整数，并存储为 LDR 数据
        }
        if (k < comp) {
            float z = data[i * comp + k] * 255 + 0.5f;  # 处理 alpha 通道
            if (z < 0)
                z = 0;
            if (z > 255)
                z = 255;
            output[i * comp + k] = (stbi_uc)stbi__float2int(z);  # 将浮点数转换为整数，并存储为 LDR 数据
        }
    }
    STBI_FREE(data);  # 释放输入数据的内存
    return output;  # 返回 LDR 数据
}
#endif
// 定义了一个简单的 JPEG/JFIF 解码器
// 实现简单
// - 不支持延迟输出 y 维度
// - 简单接口（只有一种输出格式：8 位交错 RGB）
// - 不尝试恢复损坏的 JPEG
// - 不允许部分加载，一次加载多个
// - 在 x86 上仍然很快（将全局变量复制到本地变量中对 x86 没有帮助）
// - 分配大量中间内存（所有组件的完整大小）
//   - 非交错情况下无论如何都需要这样做
//   - 允许良好的上采样（见下文）
// 高质量
// - 上采样通道进行双线性插值，甚至跨块
// - 质量整数 IDCT 源自 IJG 的 'slow'
// 性能
// - 快速哈夫曼；合理的整数 IDCT
// - 一些 SIMD 内核用于具有 SSE2/NEON 的目标上的常见路径
// - 使用大量中间内存，可能缓存不佳

#ifndef STBI_NO_JPEG

// 哈夫曼解码加速
#define FAST_BITS 9 // 较大的值处理更多情况；较小的值占用更少的缓存

typedef struct {
    stbi_uc fast[1 << FAST_BITS];
    // 奇怪的是，将其重新打包成 AoS 会导致速度下降 10%，而不是提升
    stbi__uint16 code[256];
    stbi_uc values[256];
    stbi_uc size[257];
    unsigned int maxcode[18];
    int delta[17]; // 旧的 'firstsymbol' - 旧的 'firstcode'
} stbi__huffman;

typedef struct {
    stbi__context * s;
    stbi__huffman huff_dc[4];
    stbi__huffman huff_ac[4];
    stbi__uint16 dequant[4][64];
    stbi__int16 fast_ac[4][1 << FAST_BITS];

    // 组件的大小，交错 MCU
    int img_h_max, img_v_max;
    int img_mcu_x, img_mcu_y;
    int img_mcu_w, img_mcu_h;

    // JPEG 图像组件的定义
    # 定义包含图像压缩信息的结构体
    struct {
        int id;             // 图像id
        int h, v;           // 水平和垂直采样因子
        int tq;             // 量化表索引
        int hd, ha;         // DC和AC哈夫曼表索引
        int dc_pred;        // DC预测值

        int x, y, w2, h2;   // 图像位置和大小
        stbi_uc * data;     // 图像数据
        void *raw_data, *raw_coeff;  // 原始数据和系数
        stbi_uc * linebuf;  // 行缓冲
        short * coeff;      // 仅用于渐进扫描
        int coeff_w, coeff_h;  // 8x8系数块的数量
    } img_comp[4];

    stbi__uint32 code_buffer;   // JPEG熵编码缓冲区
    int code_bits;              // 有效位数
    unsigned char marker;       // 在填充熵缓冲区时看到的标记
    int nomore;                 // 如果看到标记，则必须停止的标志

    int progressive;            // 渐进式
    int spec_start;             // 起始位置
    int spec_end;               // 结束位置
    int succ_high;              // 高位成功
    int succ_low;               // 低位成功
    int eob_run;                // EOB运行
    int jfif;                   // JFIF标志
    int app14_color_transform;  // Adobe APP14标签
    int rgb;                    // RGB标志

    int scan_n, order[4];       // 扫描数量和顺序
    int restart_interval, todo; // 重启间隔和待办事项

    # 内核
    void (*idct_block_kernel)(stbi_uc * out, int out_stride, short data[64]);  // 反量化和逆DCT内核
    void (*YCbCr_to_RGB_kernel)(stbi_uc * out, const stbi_uc * y, const stbi_uc * pcb, const stbi_uc * pcr, int count, int step);  // YCbCr到RGB内核
    stbi_uc * (*resample_row_hv_2_kernel)(stbi_uc * out, stbi_uc * in_near, stbi_uc * in_far, int w, int hs);  // 水平和垂直2倍重采样内核
# 定义一个静态函数，用于构建哈夫曼表
static int stbi__build_huffman(stbi__huffman * h, int * count) {
    int i, j, k = 0;
    unsigned int code;
    # 根据 JPEG 规范构建每个符号的大小列表
    for (i = 0; i < 16; ++i) {
        for (j = 0; j < count[i]; ++j) {
            h->size[k++] = (stbi_uc)(i + 1);
            if (k >= 257)
                return stbi__err("bad size list", "Corrupt JPEG");
        }
    }
    h->size[k] = 0;

    # 计算实际符号（来自 JPEG 规范）
    code = 0;
    k = 0;
    for (j = 1; j <= 16; ++j) {
        # 计算要添加到代码中以计算符号 ID 的增量
        h->delta[j] = k - code;
        if (h->size[k] == j) {
            while (h->size[k] == j)
                h->code[k++] = (stbi__uint16)(code++);
            if (code - 1 >= (1u << j))
                return stbi__err("bad code lengths", "Corrupt JPEG");
        }
        # 计算该大小的最大代码 + 1，根据需要预移位
        h->maxcode[j] = code << (16 - j);
        code <<= 1;
    }
    h->maxcode[j] = 0xffffffff;

    # 构建非规范加速表；255 是未加速的标志
    memset(h->fast, 255, 1 << FAST_BITS);
    for (i = 0; i < k; ++i) {
        int s = h->size[i];
        if (s <= FAST_BITS) {
            int c = h->code[i] << (FAST_BITS - s);
            int m = 1 << (FAST_BITS - s);
            for (j = 0; j < m; ++j) {
                h->fast[c + j] = (stbi_uc)i;
            }
        }
    }
    return 1;
}

# 构建一个表，一次解码小 AC 的幅度和值
static void stbi__build_fast_ac(stbi__int16 * fast_ac, stbi__huffman * h) {
    int i;
    # 遍历 0 到 2^FAST_BITS 之间的所有数
    for (i = 0; i < (1 << FAST_BITS); ++i) {
        # 获取 h->fast[i] 对应的值
        stbi_uc fast = h->fast[i];
        # 初始化 fast_ac[i] 为 0
        fast_ac[i] = 0;
        # 如果 fast < 255，则执行以下操作
        if (fast < 255) {
            # 获取 h->values[fast] 对应的值
            int rs = h->values[fast];
            # 获取 rs 的高 4 位，表示重复次数
            int run = (rs >> 4) & 15;
            # 获取 rs 的低 4 位，表示码字长度
            int magbits = rs & 15;
            # 获取 h->size[fast] 对应的值，表示码字长度
            int len = h->size[fast];

            # 如果 magbits 不为 0 且 len + magbits 小于等于 FAST_BITS，则执行以下操作
            if (magbits && len + magbits <= FAST_BITS) {
                # 计算 magnitude code 后接 receive_extend code 的值
                int k = ((i << len) & ((1 << FAST_BITS) - 1)) >> (FAST_BITS - magbits);
                # 计算 m 的值
                int m = 1 << (magbits - 1);
                # 如果 k 小于 m，则执行以下操作
                if (k < m)
                    k += (~0U << magbits) + 1;
                # 如果结果足够小，可以将其放入 fast_ac 表中
                if (k >= -128 && k <= 127)
                    fast_ac[i] = (stbi__int16)((k * 256) + (run * 16) + (len + magbits));
            }
        }
    }
}
// 扩展缓冲区，用于解码 JPEG 数据
static void stbi__grow_buffer_unsafe(stbi__jpeg * j) {
    do {
        // 读取一个字节
        unsigned int b = j->nomore ? 0 : stbi__get8(j->s);
        // 如果读取的字节为 0xff
        if (b == 0xff) {
            // 继续读取下一个字节
            int c = stbi__get8(j->s);
            // 消耗填充字节
            while (c == 0xff)
                c = stbi__get8(j->s); // consume fill bytes
            // 如果下一个字节不为 0，则表示找到了标记
            if (c != 0) {
                j->marker = (unsigned char)c;
                j->nomore = 1;
                return;
            }
        }
        // 将读取的字节存入代码缓冲区
        j->code_buffer |= b << (24 - j->code_bits);
        j->code_bits += 8;
    } while (j->code_bits <= 24);
}

// (1 << n) - 1
// 预定义的位掩码数组
static const stbi__uint32 stbi__bmask[17] = {0,   1,    3,    7,    15,   31,    63,    127,  255,
                                             511, 1023, 2047, 4095, 8191, 16383, 32767, 65535};

// 从比特流中解码 JPEG 霍夫曼值
stbi_inline static int stbi__jpeg_huff_decode(stbi__jpeg * j, stbi__huffman * h) {
    unsigned int temp;
    int c, k;

    // 如果代码位数小于 16，则扩展缓冲区
    if (j->code_bits < 16)
        stbi__grow_buffer_unsafe(j);

    // 查看顶部的 FAST_BITS，并确定它是什么符号 ID，如果代码 <= FAST_BITS
    c = (j->code_buffer >> (32 - FAST_BITS)) & ((1 << FAST_BITS) - 1);
    k = h->fast[c];
    // 如果 k < 255，则表示找到了对应的值
    if (k < 255) {
        int s = h->size[k];
        // 如果大小大于代码位数，则返回错误
        if (s > j->code_bits)
            return -1;
        // 将代码缓冲区左移 s 位，并减去 s 位
        j->code_buffer <<= s;
        j->code_bits -= s;
        return h->values[k];
    }

    // 简单的测试是将代码缓冲区向下移动，使 k 位有效，然后与 maxcode 进行比较
    // 为了加快速度，我们预先将 maxcode 左移，使其末尾有 (16-k) 个 0
    // 这样，无论位数是多少，它都希望与移位后的值进行比较，以便有 16 位
    // 这样我们就不需要在循环内部进行移位
    temp = j->code_buffer >> 16;
    for (k = FAST_BITS + 1;; ++k)
        if (temp < h->maxcode[k])
            break;
    if (k == 17) {
        // 错误！未找到代码
        j->code_bits -= 16;
        return -1;
    }
    // 如果当前比特位数大于最大编码位数，返回错误
    if (k > j->code_bits)
        return -1;

    // 将哈夫曼编码转换为符号 ID
    c = ((j->code_buffer >> (32 - k)) & stbi__bmask[k]) + h->delta[k];
    // 如果符号 ID超出范围（小于0或大于等于256），返回错误
    if (c < 0 || c >= 256) // symbol id out of bounds!
        return -1;
    // 断言，验证哈夫曼编码是否正确
    STBI_ASSERT((((j->code_buffer) >> (32 - h->size[c])) & stbi__bmask[h->size[c]]) == h->code[c]);

    // 将 ID 转换为符号
    j->code_bits -= k;
    j->code_buffer <<= k;
    // 返回符号对应的值
    return h->values[c];
// bias[n] = (-1<<n) + 1
// 定义一个静态的整型数组，存储 JPEG 解码时的偏置值
static const int stbi__jbias[16] = {0, -1, -3, -7, -15, -31, -63, -127, -255, -511, -1023, -2047, -4095, -8191, -16383, -32767};

// combined JPEG 'receive' and JPEG 'extend', since baseline
// always extends everything it receives.
// 定义一个内联函数，用于接收和扩展 JPEG 数据，因为基线总是扩展它接收到的所有数据
stbi_inline static int stbi__extend_receive(stbi__jpeg * j, int n) {
    unsigned int k;
    int sgn;
    if (j->code_bits < n)
        stbi__grow_buffer_unsafe(j);
    if (j->code_bits < n)
        return 0; // 从流中用尽了比特位，返回0而不是继续
    sgn = j->code_buffer >> 31; // 符号位始终在 MSB 中；如果 MSB 清除（正数），则为0，如果 MSB 设置（负数），则为1
    k = stbi_lrot(j->code_buffer, n);
    j->code_buffer = k & ~stbi__bmask[n];
    k &= stbi__bmask[n];
    j->code_bits -= n;
    return k + (stbi__jbias[n] & (sgn - 1));
}

// get some unsigned bits
// 定义一个内联函数，用于获取一些无符号比特位
stbi_inline static int stbi__jpeg_get_bits(stbi__jpeg * j, int n) {
    unsigned int k;
    if (j->code_bits < n)
        stbi__grow_buffer_unsafe(j);
    if (j->code_bits < n)
        return 0; // 从流中用尽了比特位，返回0而不是继续
    k = stbi_lrot(j->code_buffer, n);
    j->code_buffer = k & ~stbi__bmask[n];
    k &= stbi__bmask[n];
    j->code_bits -= n;
    return k;
}

// 定义一个内联函数，用于获取一个无符号比特位
stbi_inline static int stbi__jpeg_get_bit(stbi__jpeg * j) {
    unsigned int k;
    if (j->code_bits < 1)
        stbi__grow_buffer_unsafe(j);
    if (j->code_bits < 1)
        return 0; // 从流中用尽了比特位，返回0而不是继续
    k = j->code_buffer;
    j->code_buffer <<= 1;
    --j->code_bits;
    return k & 0x80000000;
}

// 给定在蛇形流中位置为 X 的值，在以行优先方式编码的 8x8 矩阵中出现在哪里？
// 定义一个静态的无符号字符数组，存储 JPEG 解码时的蛇形流解码顺序
static const stbi_uc stbi__jpeg_dezigzag[64 + 15] = {
    0, 1, 8, 16, 9, 2, 3, 10, 17, 24, 32, 25, 18, 11, 4, 5, 12, 19, 26, 33, 40, 48, 41, 34, 27, 20, 13, 6, 7, 14, 21, 28, 35,
    # 定义一个包含数字的数组
    42, 49, 56, 57, 50, 43, 36, 29, 22, 15, 23, 30, 37, 44, 51, 58, 59, 52, 45, 38, 31, 39, 46, 53, 60, 61, 54, 47, 55, 62, 63,
    # 在数组末尾添加了一些无效的数据，可能是损坏的输入样本
    63, 63, 63, 63, 63, 63, 63, 63, 63, 63, 63, 63, 63, 63, 63};
// 解码一个64个条目的块
static int stbi__jpeg_decode_block(stbi__jpeg * j, short data[64], stbi__huffman * hdc, stbi__huffman * hac, stbi__int16 * fac,
                                   int b, stbi__uint16 * dequant) {
    int diff, dc, k;
    int t;

    // 如果码位少于16位，则扩展缓冲区
    if (j->code_bits < 16)
        stbi__grow_buffer_unsafe(j);
    // 使用直流哈夫曼解码
    t = stbi__jpeg_huff_decode(j, hdc);
    if (t < 0 || t > 15)
        return stbi__err("bad huffman code", "Corrupt JPEG");

    // 将数据数组清零
    memset(data, 0, 64 * sizeof(data[0]));

    // 如果t不为0，则使用扩展接收函数计算差值
    diff = t ? stbi__extend_receive(j, t) : 0;
    // 检查直流预测值和差值是否有效
    if (!stbi__addints_valid(j->img_comp[b].dc_pred, diff))
        return stbi__err("bad delta", "Corrupt JPEG");
    dc = j->img_comp[b].dc_pred + diff;
    j->img_comp[b].dc_pred = dc;
    // 检查直流和量化表值是否有效
    if (!stbi__mul2shorts_valid(dc, dequant[0]))
        return stbi__err("can't merge dc and ac", "Corrupt JPEG");
    data[0] = (short)(dc * dequant[0]);

    // 解码交流分量，参考JPEG规范
    k = 1;
}
    // 进入循环，解码 JPEG 数据
    do {
        // 定义变量
        unsigned int zig;
        int c, r, s;
        // 如果当前码流不足 16 位，则扩展码流
        if (j->code_bits < 16)
            stbi__grow_buffer_unsafe(j);
        // 从码流中读取 FAST_BITS 位作为索引，获取对应的哈夫曼编码
        c = (j->code_buffer >> (32 - FAST_BITS)) & ((1 << FAST_BITS) - 1);
        r = fac[c];
        // 如果 r 不为 0，则表示是快速 AC 路径
        if (r) {                // fast-AC path
            // 根据 r 的高 4 位更新 k，表示连续零值的个数
            k += (r >> 4) & 15; // run
            // 获取 r 的低 4 位，表示组合长度
            s = r & 15;         // combined length
            // 如果组合长度大于当前码流位数，则返回错误
            if (s > j->code_bits)
                return stbi__err("bad huffman code", "Combined length longer than code bits available");
            // 将码流左移 s 位
            j->code_buffer <<= s;
            // 更新码流位数
            j->code_bits -= s;
            // 解码并存储到 zigzag 位置
            zig = stbi__jpeg_dezigzag[k++];
            data[zig] = (short)((r >> 8) * dequant[zig]);
        } else {
            // 否则，使用哈夫曼解码器解码
            int rs = stbi__jpeg_huff_decode(j, hac);
            // 如果解码结果小于 0，则返回错误
            if (rs < 0)
                return stbi__err("bad huffman code", "Corrupt JPEG");
            // 获取 rs 的低 4 位，表示组合长度
            s = rs & 15;
            // 获取 rs 的高 4 位，表示连续零值的个数
            r = rs >> 4;
            // 如果组合长度为 0，则表示结束当前块
            if (s == 0) {
                if (rs != 0xf0)
                    break; // end block
                k += 16;
            } else {
                // 更新 k，表示连续零值的个数
                k += r;
                // 解码并存储到 zigzag 位置
                zig = stbi__jpeg_dezigzag[k++];
                data[zig] = (short)(stbi__extend_receive(j, s) * dequant[zig]);
            }
        }
    } while (k < 64);
    // 返回解码结果
    return 1;
}
# 解码 JPEG 数据块的 DC 系数，用于渐进式扫描
static int stbi__jpeg_decode_block_prog_dc(stbi__jpeg * j, short data[64], stbi__huffman * hdc, int b) {
    int diff, dc;
    int t;
    if (j->spec_end != 0)
        return stbi__err("can't merge dc and ac", "Corrupt JPEG");

    if (j->code_bits < 16)
        stbi__grow_buffer_unsafe(j);

    if (j->succ_high == 0):
        # 第一次扫描 DC 系数，必须首先进行
        # 将所有 AC 值置为 0
        memset(data, 0, 64 * sizeof(data[0]));
        t = stbi__jpeg_huff_decode(j, hdc);
        if (t < 0 || t > 15)
            return stbi__err("can't merge dc and ac", "Corrupt JPEG");
        diff = t ? stbi__extend_receive(j, t) : 0;

        if (!stbi__addints_valid(j->img_comp[b].dc_pred, diff))
            return stbi__err("bad delta", "Corrupt JPEG");
        dc = j->img_comp[b].dc_pred + diff;
        j->img_comp[b].dc_pred = dc;
        if (!stbi__mul2shorts_valid(dc, 1 << j->succ_low))
            return stbi__err("can't merge dc and ac", "Corrupt JPEG");
        data[0] = (short)(dc * (1 << j->succ_low));
    else:
        # DC 系数的渐进扫描
        if (stbi__jpeg_get_bit(j))
            data[0] += (short)(1 << j->succ_low);
    return 1;
}

# @OPTIMIZE: 在解码过程中存储非蛇形排列的数据，
# 仅在反量化时进行蛇形排列
static int stbi__jpeg_decode_block_prog_ac(stbi__jpeg * j, short data[64], stbi__huffman * hac, stbi__int16 * fac) {
    int k;
    if (j->spec_start == 0)
        return stbi__err("can't merge dc and ac", "Corrupt JPEG");
    # 如果 j->succ_high 等于 0，则执行以下代码块
    if (j->succ_high == 0) {
        # 将 j->succ_low 赋值给 shift
        int shift = j->succ_low;

        # 如果 j->eob_run 不为 0，则将其减一并返回 1
        if (j->eob_run) {
            --j->eob_run;
            return 1;
        }

        # 将 j->spec_start 赋值给 k
        k = j->spec_start;
        # 执行以下循环
        do {
            # 定义无符号整型变量 zig，整型变量 c、r、s
            unsigned int zig;
            int c, r, s;
            # 如果 j->code_bits 小于 16，则调用 stbi__grow_buffer_unsafe(j) 函数
            if (j->code_bits < 16)
                stbi__grow_buffer_unsafe(j);
            # 将 j->code_buffer 右移 (32 - FAST_BITS) 位，并与 ((1 << FAST_BITS) - 1) 进行按位与运算，结果赋值给 c
            c = (j->code_buffer >> (32 - FAST_BITS)) & ((1 << FAST_BITS) - 1);
            # 将 fac[c] 的值赋值给 r
            r = fac[c];
            # 如果 r 不为 0，则执行以下代码块
            if (r) {                // fast-AC path
                # 将 (r >> 4) & 15 的值加给 k，结果赋值给 k
                k += (r >> 4) & 15; // run
                # 将 r & 15 的值赋值给 s
                s = r & 15;         // combined length
                # 如果 s 大于 j->code_bits，则返回错误信息
                if (s > j->code_bits)
                    return stbi__err("bad huffman code", "Combined length longer than code bits available");
                # 将 j->code_buffer 左移 s 位
                j->code_buffer <<= s;
                # 将 j->code_bits 减去 s
                j->code_bits -= s;
                # 将 stbi__jpeg_dezigzag[k++] 的值赋值给 zig
                zig = stbi__jpeg_dezigzag[k++];
                # 将 (r >> 8) * (1 << shift) 的值转换为 short 类型，并赋值给 data[zig]
                data[zig] = (short)((r >> 8) * (1 << shift));
            } else {
                # 将 stbi__jpeg_huff_decode(j, hac) 的返回值赋值给 rs
                int rs = stbi__jpeg_huff_decode(j, hac);
                # 如果 rs 小于 0，则返回错误信息
                if (rs < 0)
                    return stbi__err("bad huffman code", "Corrupt JPEG");
                # 将 rs & 15 的值赋值给 s，将 rs >> 4 的值赋值给 r
                s = rs & 15;
                r = rs >> 4;
                # 如果 s 等于 0，则执行以下代码块
                if (s == 0) {
                    # 如果 r 小于 15，则将 (1 << r) 赋值给 j->eob_run，如果 r 不为 0，则将 stbi__jpeg_get_bits(j, r) 的值加给 j->eob_run
                    j->eob_run = (1 << r);
                    # 如果 r 不为 0，则将 j->eob_run 减一
                    if (r)
                        j->eob_run += stbi__jpeg_get_bits(j, r);
                    --j->eob_run;
                    # 跳出循环
                    break;
                }
                # 否则执行以下代码块
                else {
                    # 将 r 加给 k
                    k += r;
                    # 将 stbi__jpeg_dezigzag[k++] 的值赋值给 zig
                    zig = stbi__jpeg_dezigzag[k++];
                    # 将 stbi__extend_receive(j, s) * (1 << shift) 的值转换为 short 类型，并赋值给 data[zig]
                    data[zig] = (short)(stbi__extend_receive(j, s) * (1 << shift));
                }
            }
        } while (k <= j->spec_end);
    }
    # 返回 1
    return 1;
// 将一个 -128 到 127 的值限制在 0 到 255 之间，并转换为无符号字符型
stbi_inline static stbi_uc stbi__clamp(int x) {
    // 用一个测试来捕捉两种情况的技巧
    if ((unsigned int)x > 255) {
        if (x < 0)
            return 0;
        if (x > 255)
            return 255;
    }
    return (stbi_uc)x;
}

// 定义一个宏，用于将浮点数乘以 4096 并四舍五入到整数
#define stbi__f2f(x) ((int)(((x)*4096 + 0.5)))
// 定义一个宏，用于将浮点数乘以 4096
#define stbi__fsh(x) ((x)*4096)

// 从 jidctint 派生而来 -- DCT_ISLOW
#define STBI__IDCT_1D(s0, s1, s2, s3, s4, s5, s6, s7)                                                                          \
    int t0, t1, t2, t3, p1, p2, p3, p4, p5, x0, x1, x2, x3;                                                                    \
    p2 = s2;                                                                                                                   \
    p3 = s6;                                                                                                                   \
    p1 = (p2 + p3) * stbi__f2f(0.5411961f);                                                                                    \
    t2 = p1 + p3 * stbi__f2f(-1.847759065f);                                                                                   \
    t3 = p1 + p2 * stbi__f2f(0.765366865f);                                                                                    \
    p2 = s0;                                                                                                                   \
    p3 = s4;                                                                                                                   \
    t0 = stbi__fsh(p2 + p3);                                                                                                   \
    t1 = stbi__fsh(p2 - p3);                                                                                                   \
    x0 = t0 + t3;                                                                                                              \
    # 计算 t0 和 t3 的差值
    x3 = t0 - t3;                                                                                                              \
    # 计算 t1 和 t2 的和
    x1 = t1 + t2;                                                                                                              \
    # 计算 t1 和 t2 的差值
    x2 = t1 - t2;                                                                                                              \
    # 保存 s7 的值到 t0
    t0 = s7;                                                                                                                   \
    # 保存 s5 的值到 t1
    t1 = s5;                                                                                                                   \
    # 保存 s3 的值到 t2
    t2 = s3;                                                                                                                   \
    # 保存 s1 的值到 t3
    t3 = s1;                                                                                                                   \
    # 计算 t0 和 t2 的和
    p3 = t0 + t2;                                                                                                              \
    # 计算 t1 和 t3 的和
    p4 = t1 + t3;                                                                                                              \
    # 计算 t0 和 t3 的和
    p1 = t0 + t3;                                                                                                              \
    # 计算 t1 和 t2 的和
    p2 = t1 + t2;                                                                                                              \
    # 计算 p3 和 p4 的和，然后乘以常数 1.175875602f
    p5 = (p3 + p4) * stbi__f2f(1.175875602f);                                                                                  \
    # t0 乘以常数 0.298631336f
    t0 = t0 * stbi__f2f(0.298631336f);                                                                                         \
    # t1 乘以常数 2.053119869f
    t1 = t1 * stbi__f2f(2.053119869f);                                                                                         \
    # t2 乘以常数 3.072711026f
    t2 = t2 * stbi__f2f(3.072711026f);                                                                                         \
    # 对 t3 进行乘法运算
    t3 = t3 * stbi__f2f(1.501321110f);                                                                                         \
    # 对 p1 进行加法和乘法运算
    p1 = p5 + p1 * stbi__f2f(-0.899976223f);                                                                                   \
    # 对 p2 进行加法和乘法运算
    p2 = p5 + p2 * stbi__f2f(-2.562915447f);                                                                                   \
    # 对 p3 进行乘法运算
    p3 = p3 * stbi__f2f(-1.961570560f);                                                                                        \
    # 对 p4 进行乘法运算
    p4 = p4 * stbi__f2f(-0.390180644f);                                                                                        \
    # 对 t3 进行加法运算
    t3 += p1 + p4;                                                                                                             \
    # 对 t2 进行加法运算
    t2 += p2 + p3;                                                                                                             \
    # 对 t1 进行加法运算
    t1 += p2 + p4;                                                                                                             \
    # 对 t0 进行加法运算
    t0 += p1 + p3;
static void stbi__idct_block(stbi_uc * out, int out_stride, short data[64]) {
    int i, val[64], *v = val;  # 定义变量 i, val 数组和指针 v，用于存储中间结果
    stbi_uc * o;  # 定义指针 o，用于存储输出结果
    short * d = data;  # 定义指针 d，指向输入的数据

    // columns  # 对每一列进行操作
    for (i = 0; i < 8; ++i, ++d, ++v) {  # 循环遍历每一列
        // if all zeroes, shortcut -- this avoids dequantizing 0s and IDCTing
        if (d[8] == 0 && d[16] == 0 && d[24] == 0 && d[32] == 0 && d[40] == 0 && d[48] == 0 && d[56] == 0) {
            //    no shortcut                 0     seconds
            //    (1|2|3|4|5|6|7)==0          0     seconds
            //    all separate               -0.047 seconds
            //    1 && 2|3 && 4|5 && 6|7:    -0.047 seconds
            int dcterm = d[0] * 4;  # 计算中间结果 dcterm
            v[0] = v[8] = v[16] = v[24] = v[32] = v[40] = v[48] = v[56] = dcterm;  # 将中间结果赋值给 v 数组
        } else {
            STBI__IDCT_1D(d[0], d[8], d[16], d[24], d[32], d[40], d[48], d[56])  # 调用宏定义的 IDCT 函数，对输入数据进行反量化和反离散余弦变换
            // constants scaled things up by 1<<12; let's bring them back
            // down, but keep 2 extra bits of precision
            x0 += 512;  # 对中间结果进行处理
            x1 += 512;  # 对中间结果进行处理
            x2 += 512;  # 对中间结果进行处理
            x3 += 512;  # 对中间结果进行处理
            v[0] = (x0 + t3) >> 10;  # 对中间结果进行处理
            v[56] = (x0 - t3) >> 10;  # 对中间结果进行处理
            v[8] = (x1 + t2) >> 10;  # 对中间结果进行处理
            v[48] = (x1 - t2) >> 10;  # 对中间结果进行处理
            v[16] = (x2 + t1) >> 10;  # 对中间结果进行处理
            v[40] = (x2 - t1) >> 10;  # 对中间结果进行处理
            v[24] = (x3 + t0) >> 10;  # 对中间结果进行处理
            v[32] = (x3 - t0) >> 10;  # 对中间结果进行处理
        }
    }
    // 遍历8个元素的数组，每次处理一个元素
    for (i = 0, v = val, o = out; i < 8; ++i, v += 8, o += out_stride) {
        // 执行一维逆离散余弦变换（IDCT）操作
        STBI__IDCT_1D(v[0], v[1], v[2], v[3], v[4], v[5], v[6], v[7])
        // 对结果进行修正，移除缩放常量并进行四舍五入
        x0 += 65536 + (128 << 17);
        x1 += 65536 + (128 << 17);
        x2 += 65536 + (128 << 17);
        x3 += 65536 + (128 << 17);
        // 对结果进行修正，移除缩放常量并进行四舍五入
        o[0] = stbi__clamp((x0 + t3) >> 17);
        o[7] = stbi__clamp((x0 - t3) >> 17);
        o[1] = stbi__clamp((x1 + t2) >> 17);
        o[6] = stbi__clamp((x1 - t2) >> 17);
        o[2] = stbi__clamp((x2 + t1) >> 17);
        o[5] = stbi__clamp((x2 - t1) >> 17);
        o[3] = stbi__clamp((x3 + t0) >> 17);
        o[4] = stbi__clamp((x3 - t0) >> 17);
    }
}
#ifdef STBI_SSE2
// 定义了使用 SSE2 指令集的整数 IDCT 函数，虽然不是最快的实现方式，但产生的结果与通用 C 版本完全相同，因此是“透明”的。
static void stbi__idct_simd(stbi_uc * out, int out_stride, short data[64]) {
    // 以下代码是为了与通用整数 IDCT 完全匹配而构建的。
    __m128i row0, row1, row2, row3, row4, row5, row6, row7;
    __m128i tmp;

// 点乘常数：偶数元素=x，奇数元素=y
#define dct_const(x, y) _mm_setr_epi16((x), (y), (x), (y), (x), (y), (x), (y))

// out(0) = c0[even]*x + c0[odd]*y   (c0, x, y 16-bit, out 32-bit)
// out(1) = c1[even]*x + c1[odd]*y
#define dct_rot(out0, out1, x, y, c0, c1)                                                                                      \
    __m128i c0##lo = _mm_unpacklo_epi16((x), (y));                                                                             \
    __m128i c0##hi = _mm_unpackhi_epi16((x), (y));                                                                             \
    __m128i out0##_l = _mm_madd_epi16(c0##lo, c0);                                                                             \
    __m128i out0##_h = _mm_madd_epi16(c0##hi, c0);                                                                             \
    __m128i out1##_l = _mm_madd_epi16(c0##lo, c1);                                                                             \
    __m128i out1##_h = _mm_madd_epi16(c0##hi, c1)

// out = in << 12  (in 16-bit, out 32-bit)
#define dct_widen(out, in)                                                                                                     \
    __m128i out##_l = _mm_srai_epi32(_mm_unpacklo_epi16(_mm_setzero_si128(), (in)), 4);                                        \
    __m128i out##_h = _mm_srai_epi32(_mm_unpackhi_epi16(_mm_setzero_si128(), (in)), 4)

// 宽加法
// 定义宏，将两个__m128i类型的变量相加，结果存储在out##_l和out##_h中
#define dct_wadd(out, a, b)                                                                                                    \
    __m128i out##_l = _mm_add_epi32(a##_l, b##_l);                                                                             \
    __m128i out##_h = _mm_add_epi32(a##_h, b##_h)

// 宽字节减法
#define dct_wsub(out, a, b)                                                                                                    \
    __m128i out##_l = _mm_sub_epi32(a##_l, b##_l);                                                                             \
    __m128i out##_h = _mm_sub_epi32(a##_h, b##_h)

// 蝶形运算，将a和b相加，加上偏置，然后按"s"进行移位和打包
#define dct_bfly32o(out0, out1, a, b, bias, s)                                                                                 \
    {                                                                                                                          \
        // 将a##_l加上偏置
        __m128i abiased_l = _mm_add_epi32(a##_l, bias);                                                                        \
        // 将a##_h加上偏置
        __m128i abiased_h = _mm_add_epi32(a##_h, bias);                                                                        \
        // 调用dct_wadd宏，将abiased和b相加
        dct_wadd(sum, abiased, b);                                                                                             \
        // 调用dct_wsub宏，将abiased和b相减
        dct_wsub(dif, abiased, b);                                                                                             \
        // 将结果右移s位，并打包成16位整数
        out0 = _mm_packs_epi32(_mm_srai_epi32(sum_l, s), _mm_srai_epi32(sum_h, s));                                            \
        out1 = _mm_packs_epi32(_mm_srai_epi32(dif_l, s), _mm_srai_epi32(dif_h, s));                                            \
    }

// 8位交错步骤（用于转置）
#define dct_interleave8(a, b)                                                                                                  \
    # 将变量a的值赋给临时变量tmp
    tmp = a;                                                                                                                   \
    # 使用SSE指令将a和b中的每个字节拆分成两个16位整数，然后将这些整数按顺序排列在一起，存储到a中
    a = _mm_unpacklo_epi8(a, b);                                                                                               \
    # 使用SSE指令将tmp和b中的每个字节拆分成两个16位整数，然后将这些整数按顺序排列在一起，存储到b中
    b = _mm_unpackhi_epi8(tmp, b)
// 定义16位交错步骤（用于转置）
#define dct_interleave16(a, b)                                                                                                 \
    tmp = a;                                                                                                                   \
    a = _mm_unpacklo_epi16(a, b);                                                                                              \
    b = _mm_unpackhi_epi16(tmp, b)

#define dct_pass(bias, shift)                                                                                                  \
    }

    // 创建常量__m128i类型的变量，用于DCT变换
    __m128i rot0_0 = dct_const(stbi__f2f(0.5411961f), stbi__f2f(0.5411961f) + stbi__f2f(-1.847759065f));
    __m128i rot0_1 = dct_const(stbi__f2f(0.5411961f) + stbi__f2f(0.765366865f), stbi__f2f(0.5411961f));
    __m128i rot1_0 = dct_const(stbi__f2f(1.175875602f) + stbi__f2f(-0.899976223f), stbi__f2f(1.175875602f));
    __m128i rot1_1 = dct_const(stbi__f2f(1.175875602f), stbi__f2f(1.175875602f) + stbi__f2f(-2.562915447f));
    __m128i rot2_0 = dct_const(stbi__f2f(-1.961570560f) + stbi__f2f(0.298631336f), stbi__f2f(-1.961570560f));
    __m128i rot2_1 = dct_const(stbi__f2f(-1.961570560f), stbi__f2f(-1.961570560f) + stbi__f2f(3.072711026f));
    __m128i rot3_0 = dct_const(stbi__f2f(-0.390180644f) + stbi__f2f(2.053119869f), stbi__f2f(-0.390180644f));
    __m128i rot3_1 = dct_const(stbi__f2f(-0.390180644f), stbi__f2f(-0.390180644f) + stbi__f2f(1.501321110f));

    // 在列/行传递中的舍入偏差，参见stbi__idct_block的解释。
    __m128i bias_0 = _mm_set1_epi32(512);
    __m128i bias_1 = _mm_set1_epi32(65536 + (128 << 17));

    // 加载数据
    row0 = _mm_load_si128((const __m128i *)(data + 0 * 8));
    row1 = _mm_load_si128((const __m128i *)(data + 1 * 8));
    row2 = _mm_load_si128((const __m128i *)(data + 2 * 8));
    row3 = _mm_load_si128((const __m128i *)(data + 3 * 8));
    row4 = _mm_load_si128((const __m128i *)(data + 4 * 8));
    // 从数据中加载第5行的数据到row5
    row5 = _mm_load_si128((const __m128i *)(data + 5 * 8));
    // 从数据中加载第6行的数据到row6
    row6 = _mm_load_si128((const __m128i *)(data + 6 * 8));
    // 从数据中加载第7行的数据到row7
    row7 = _mm_load_si128((const __m128i *)(data + 7 * 8));

    // 列处理
    dct_pass(bias_0, 10);

    {
        // 16位8x8转置处理第1步
        dct_interleave16(row0, row4);
        dct_interleave16(row1, row5);
        dct_interleave16(row2, row6);
        dct_interleave16(row3, row7);

        // 转置处理第2步
        dct_interleave16(row0, row2);
        dct_interleave16(row1, row3);
        dct_interleave16(row4, row6);
        dct_interleave16(row5, row7);

        // 转置处理第3步
        dct_interleave16(row0, row1);
        dct_interleave16(row2, row3);
        dct_interleave16(row4, row5);
        dct_interleave16(row6, row7);
    }

    // 行处理
    dct_pass(bias_1, 17);
    {
        // 将两个 128 位寄存器中的 16 位整数打包成一个 128 位寄存器，无符号饱和转换
        __m128i p0 = _mm_packus_epi16(row0, row1); // a0a1a2a3...a7b0b1b2b3...b7
        __m128i p1 = _mm_packus_epi16(row2, row3);
        __m128i p2 = _mm_packus_epi16(row4, row5);
        __m128i p3 = _mm_packus_epi16(row6, row7);
    
        // 8 位 8x8 转置，第一步
        dct_interleave8(p0, p2); // a0e0a1e1...
        dct_interleave8(p1, p3); // c0g0c1g1...
    
        // 转置，第二步
        dct_interleave8(p0, p1); // a0c0e0g0...
        dct_interleave8(p2, p3); // b0d0f0h0...
    
        // 转置，第三步
        dct_interleave8(p0, p2); // a0b0c0d0...
        dct_interleave8(p1, p3); // a4b4c4d4...
    
        // 存储
        _mm_storel_epi64((__m128i *)out, p0);
        out += out_stride;
        _mm_storel_epi64((__m128i *)out, _mm_shuffle_epi32(p0, 0x4e));
        out += out_stride;
        _mm_storel_epi64((__m128i *)out, p2);
        out += out_stride;
        _mm_storel_epi64((__m128i *)out, _mm_shuffle_epi32(p2, 0x4e));
        out += out_stride;
        _mm_storel_epi64((__m128i *)out, p1);
        out += out_stride;
        _mm_storel_epi64((__m128i *)out, _mm_shuffle_epi32(p1, 0x4e));
        out += out_stride;
        _mm_storel_epi64((__m128i *)out, p3);
        out += out_stride;
        _mm_storel_epi64((__m128i *)out, _mm_shuffle_epi32(p3, 0x4e));
    }
// 取消定义 dct_const
#undef dct_const
// 取消定义 dct_rot
#undef dct_rot
// 取消定义 dct_widen
#undef dct_widen
// 取消定义 dct_wadd
#undef dct_wadd
// 取消定义 dct_wsub
#undef dct_wsub
// 取消定义 dct_bfly32o
#undef dct_bfly32o
// 取消定义 dct_interleave8
#undef dct_interleave8
// 取消定义 dct_interleave16
#undef dct_interleave16
// 取消定义 dct_pass
#undef dct_pass
}

#endif // STBI_SSE2

#ifdef STBI_NEON

// NEON integer IDCT. should produce bit-identical
// results to the generic C version.
static void stbi__idct_simd(stbi_uc * out, int out_stride, short data[64]) {
    // 定义 NEON 寄存器变量
    int16x8_t row0, row1, row2, row3, row4, row5, row6, row7;

    // 定义 NEON 寄存器变量并初始化
    int16x4_t rot0_0 = vdup_n_s16(stbi__f2f(0.5411961f));
    int16x4_t rot0_1 = vdup_n_s16(stbi__f2f(-1.847759065f));
    int16x4_t rot0_2 = vdup_n_s16(stbi__f2f(0.765366865f));
    int16x4_t rot1_0 = vdup_n_s16(stbi__f2f(1.175875602f));
    int16x4_t rot1_1 = vdup_n_s16(stbi__f2f(-0.899976223f));
    int16x4_t rot1_2 = vdup_n_s16(stbi__f2f(-2.562915447f));
    int16x4_t rot2_0 = vdup_n_s16(stbi__f2f(-1.961570560f));
    int16x4_t rot2_1 = vdup_n_s16(stbi__f2f(-0.390180644f));
    int16x4_t rot3_0 = vdup_n_s16(stbi__f2f(0.298631336f));
    int16x4_t rot3_1 = vdup_n_s16(stbi__f2f(2.053119869f));
    int16x4_t rot3_2 = vdup_n_s16(stbi__f2f(3.072711026f));
    int16x4_t rot3_3 = vdup_n_s16(stbi__f2f(1.501321110f));

    // 定义宏，用于将输入寄存器变量与系数相乘并拓宽为 32 位
#define dct_long_mul(out, inq, coeff)                                                                                          \
    int32x4_t out##_l = vmull_s16(vget_low_s16(inq), coeff);                                                                   \
    int32x4_t out##_h = vmull_s16(vget_high_s16(inq), coeff)

    // 定义宏，用于将输入寄存器变量与系数相乘并累加到累加寄存器变量中
#define dct_long_mac(out, acc, inq, coeff)                                                                                     \
    int32x4_t out##_l = vmlal_s16(acc##_l, vget_low_s16(inq), coeff);                                                          \
    int32x4_t out##_h = vmlal_s16(acc##_h, vget_high_s16(inq), coeff)

    // 定义宏，用于将输入寄存器变量拓宽为 32 位
#define dct_widen(out, inq)                                                                                                    \
    # 将输入的低位16位有符号整数扩展为32位有符号整数，并左移12位
    int32x4_t out##_l = vshll_n_s16(vget_low_s16(inq), 12);
    # 将输入的高位16位有符号整数扩展为32位有符号整数，并左移12位
    int32x4_t out##_h = vshll_n_s16(vget_high_s16(inq), 12)
// 定义宏，用于计算两个 int32x4_t 类型的向量的和
#define dct_wadd(out, a, b)                                                                                                    \
    int32x4_t out##_l = vaddq_s32(a##_l, b##_l);                                                                               \
    int32x4_t out##_h = vaddq_s32(a##_h, b##_h)

// 定义宏，用于计算两个 int32x4_t 类型的向量的差
#define dct_wsub(out, a, b)                                                                                                    \
    int32x4_t out##_l = vsubq_s32(a##_l, b##_l);                                                                               \
    int32x4_t out##_h = vsubq_s32(a##_h, b##_h)

// 定义宏，用于执行蝶形运算，然后使用 "shiftop" 进行移位，移位量为 "s"，并打包结果
#define dct_bfly32o(out0, out1, a, b, shiftop, s)                                                                              \
    {                                                                                                                          \
        // 计算和与差
        dct_wadd(sum, a, b);                                                                                                   
        dct_wsub(dif, a, b);                                                                                                   
        // 执行移位和打包
        out0 = vcombine_s16(shiftop(sum_l, s), shiftop(sum_h, s));                                                             
        out1 = vcombine_s16(shiftop(dif_l, s), shiftop(dif_h, s));                                                             
    }

#define dct_pass(shiftop, shift)                                                                                               \
    }

    // 加载数据
    row0 = vld1q_s16(data + 0 * 8);
    row1 = vld1q_s16(data + 1 * 8);
    row2 = vld1q_s16(data + 2 * 8);
    row3 = vld1q_s16(data + 3 * 8);
    row4 = vld1q_s16(data + 4 * 8);
    row5 = vld1q_s16(data + 5 * 8);
    row6 = vld1q_s16(data + 6 * 8);
    row7 = vld1q_s16(data + 7 * 8);

    // 添加直流偏置
    # 将第0行的每个元素与1024相加，并将结果存储在第0个位置
    row0 = vaddq_s16(row0, vsetq_lane_s16(1024, vdupq_n_s16(0), 0));

    # 对列进行DCT变换
    dct_pass(vrshrn_n_s32, 10);

    # 进行16位8x8矩阵的转置操作
    {
// 定义一个宏，用于对两个 int16x8 类型的向量进行交替交换
#define dct_trn16(x, y)                                                                                                        \
    {                                                                                                                          \
        // 使用 vtrnq_s16 函数对 x 和 y 进行交替交换，并将结果保存在 t 中
        int16x8x2_t t = vtrnq_s16(x, y);                                                                                       \
        // 将 t 中的第一个值赋给 x，将 t 中的第二个值赋给 y
        x = t.val[0];                                                                                                          \
        y = t.val[1];                                                                                                          \
    }
// 定义一个宏，用于对两个 int32x4 类型的向量进行交替交换
#define dct_trn32(x, y)                                                                                                        \
    {                                                                                                                          \
        // 使用 vreinterpretq_s32_s16 函数将 x 和 y 转换为 int32x4 类型，然后使用 vtrnq_s32 函数对转换后的向量进行交替交换，并将结果保存在 t 中
        int32x4x2_t t = vtrnq_s32(vreinterpretq_s32_s16(x), vreinterpretq_s32_s16(y));                                         \
        // 使用 vreinterpretq_s16_s32 函数将 t 中的第一个值转换为 int16x8 类型，并赋给 x；将 t 中的第二个值转换为 int16x8 类型，并赋给 y
        x = vreinterpretq_s16_s32(t.val[0]);                                                                                   \
        y = vreinterpretq_s16_s32(t.val[1]);                                                                                   \
    }
// 定义一个空的宏，用于对两个 int64x2 类型的向量进行交替交换
#define dct_trn64(x, y)                                                                                                        \
    {                                                                                                                          \
        int16x8_t x0 = x;                                                                                                      \  // 将x的值复制给x0
        int16x8_t y0 = y;                                                                                                      \  // 将y的值复制给y0
        x = vcombine_s16(vget_low_s16(x0), vget_low_s16(y0));                                                                  \  // 将x0和y0的低位组合成新的x
        y = vcombine_s16(vget_high_s16(x0), vget_high_s16(y0));                                                                \  // 将x0和y0的高位组合成新的y
    }

        // pass 1
        dct_trn16(row0, row1); // a0b0a2b2a4b4a6b6                                                                             \  // 对row0和row1进行16点DCT变换
        dct_trn16(row2, row3);                                                                                                  \  // 对row2和row3进行16点DCT变换
        dct_trn16(row4, row5);                                                                                                  \  // 对row4和row5进行16点DCT变换
        dct_trn16(row6, row7);                                                                                                  \  // 对row6和row7进行16点DCT变换

        // pass 2
        dct_trn32(row0, row2); // a0b0c0d0a4b4c4d4                                                                             \  // 对row0和row2进行32点DCT变换
        dct_trn32(row1, row3);                                                                                                  \  // 对row1和row3进行32点DCT变换
        dct_trn32(row4, row6);                                                                                                  \  // 对row4和row6进行32点DCT变换
        dct_trn32(row5, row7);                                                                                                  \  // 对row5和row7进行32点DCT变换

        // pass 3
        dct_trn64(row0, row4); // a0b0c0d0e0f0g0h0                                                                             \  // 对row0和row4进行64点DCT变换
        dct_trn64(row1, row5);                                                                                                  \  // 对row1和row5进行64点DCT变换
        dct_trn64(row2, row6);                                                                                                  \  // 对row2和row6进行64点DCT变换
        dct_trn64(row3, row7);                                                                                                  \  // 对row3和row7进行64点DCT变换
// 取消之前定义的 dct_trn16、dct_trn32、dct_trn64
#undef dct_trn16
#undef dct_trn32
#undef dct_trn64
    }

    // 行处理
    // vrshrn_n_s32 只支持最大 16 的位移，我们需要 17。因此先进行一个非四舍五入的 16 位移，然后再进行一个 1 的四舍五入位移。
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

        // 再次，这些可以转换为一条指令，但通常不会。
#define dct_trn8_8(x, y)                                                                                                       \
    {                                                                                                                          \
        uint8x8x2_t t = vtrn_u8(x, y);                                                                                         \
        x = t.val[0];                                                                                                          \
        y = t.val[1];                                                                                                          \
    }
#define dct_trn8_16(x, y)                                                                                                      \
    # 创建一个 2x4 的 uint16 类型的向量 t，其中 t.val[0] 存储 x 和 y 中的偶数索引元素，t.val[1] 存储 x 和 y 中的奇数索引元素
    uint16x4x2_t t = vtrn_u16(vreinterpret_u16_u8(x), vreinterpret_u16_u8(y));
    # 将 t.val[0] 转换为 uint8 类型的向量，并赋值给 x
    x = vreinterpret_u8_u16(t.val[0]);
    # 将 t.val[1] 转换为 uint8 类型的向量，并赋值给 y
    y = vreinterpret_u8_u16(t.val[1]);
// 定义一个宏，用于将两个寄存器中的8位数据进行32位的交错存储
#define dct_trn8_32(x, y)                                                                                                      \
    {                                                                                                                          \
        // 将两个寄存器中的8位数据转换成两个32位的数据，并进行交错存储
        uint32x2x2_t t = vtrn_u32(vreinterpret_u32_u8(x), vreinterpret_u32_u8(y));                                             \
        // 将交错存储后的数据重新转换成8位数据
        x = vreinterpret_u8_u32(t.val[0]);                                                                                     \
        y = vreinterpret_u8_u32(t.val[1]);                                                                                     \
    }

// 8x8 8位转置，第一次通过宏dct_trn8_8进行转置
dct_trn8_8(p0, p1);
dct_trn8_8(p2, p3);
dct_trn8_8(p4, p5);
dct_trn8_8(p6, p7);

// 第二次通过宏dct_trn8_16进行转置
dct_trn8_16(p0, p2);
dct_trn8_16(p1, p3);
dct_trn8_16(p4, p6);
dct_trn8_16(p5, p7);

// 第三次通过宏dct_trn8_32进行转置
dct_trn8_32(p0, p4);
dct_trn8_32(p1, p5);
dct_trn8_32(p2, p6);
dct_trn8_32(p3, p7);

// 将转置后的数据存储到内存中
vst1_u8(out, p0);
out += out_stride;
vst1_u8(out, p1);
out += out_stride;
vst1_u8(out, p2);
out += out_stride;
vst1_u8(out, p3);
out += out_stride;
vst1_u8(out, p4);
out += out_stride;
vst1_u8(out, p5);
out += out_stride;
vst1_u8(out, p6);
out += out_stride;
vst1_u8(out, p7);

// 取消之前定义的宏
#undef dct_trn8_8
#undef dct_trn8_16
#undef dct_trn8_32
}

// 取消之前定义的宏
#undef dct_long_mul
#undef dct_long_mac
#undef dct_widen
#undef dct_wadd
#undef dct_wsub
#undef dct_bfly32o
#undef dct_pass
}

#endif // STBI_NEON

// 定义一个标记常量
#define STBI__MARKER_none 0xff
// 如果熵流中有一个待处理的标记，返回该标记；否则，从流中获取一个标记。如果没有
// 获取标记值，如果标记值不为STBI__MARKER_none，则返回标记值，并将标记值重置为STBI__MARKER_none
static stbi_uc stbi__get_marker(stbi__jpeg * j) {
    stbi_uc x;
    if (j->marker != STBI__MARKER_none) {
        x = j->marker;
        j->marker = STBI__MARKER_none;
        return x;
    }
    x = stbi__get8(j->s); // 从输入流中获取一个字节
    if (x != 0xff)
        return STBI__MARKER_none; // 如果获取的字节不是0xff，则返回STBI__MARKER_none
    while (x == 0xff)
        x = stbi__get8(j->s); // 消耗重复的0xff填充字节
    return x; // 返回获取的字节
}

// 在每个扫描中，我们将有scan_n个分量，分量的顺序由order[]指定
#define STBI__RESTART(x) ((x) >= 0xd0 && (x) <= 0xd7) // 定义宏，用于判断是否为重启标记

// 在重启间隔之后，重置熵解码器和DC预测
static void stbi__jpeg_reset(stbi__jpeg * j) {
    j->code_bits = 0; // 重置码位数
    j->code_buffer = 0; // 重置码缓冲区
    j->nomore = 0; // 重置nomore标志
    j->img_comp[0].dc_pred = j->img_comp[1].dc_pred = j->img_comp[2].dc_pred = j->img_comp[3].dc_pred = 0; // 重置DC预测值
    j->marker = STBI__MARKER_none; // 重置标记值
    j->todo = j->restart_interval ? j->restart_interval : 0x7fffffff; // 重置todo值
    j->eob_run = 0; // 重置eob_run值
    // 如果没有restart_interval，则不超过1<<31个MCU，这是非常安全的，因为我们甚至不允许1<<30个像素
}

// 解析熵编码数据
static int stbi__parse_entropy_coded_data(stbi__jpeg * z) {
    stbi__jpeg_reset(z); // 重置JPEG数据
}

// 反量化
static void stbi__jpeg_dequantize(short * data, stbi__uint16 * dequant) {
    int i;
    for (i = 0; i < 64; ++i)
        data[i] *= dequant[i]; // 对数据进行反量化
}

// JPEG解码完成
static void stbi__jpeg_finish(stbi__jpeg * z) {
    # 如果图像是渐进式的
    if (z->progressive) {
        # 对数据进行反量化和逆离散余弦变换
        int i, j, n;
        # 遍历图像的每个颜色通道
        for (n = 0; n < z->s->img_n; ++n) {
            # 计算每个颜色通道的宽度和高度
            int w = (z->img_comp[n].x + 7) >> 3;
            int h = (z->img_comp[n].y + 7) >> 3;
            # 遍历每个颜色通道的每个 8x8 块
            for (j = 0; j < h; ++j) {
                for (i = 0; i < w; ++i) {
                    # 获取当前 8x8 块的数据
                    short * data = z->img_comp[n].coeff + 64 * (i + j * z->img_comp[n].coeff_w);
                    # 对数据进行反量化
                    stbi__jpeg_dequantize(data, z->dequant[z->img_comp[n].tq]);
                    # 对数据进行逆离散余弦变换
                    z->idct_block_kernel(z->img_comp[n].data + z->img_comp[n].w2 * j * 8 + i * 8, z->img_comp[n].w2, data);
                }
            }
        }
    }
    # 处理 JPEG 文件标记
    static int stbi__process_marker(stbi__jpeg * z, int m) {
        int L;
        switch (m) {
        case STBI__MARKER_none: // 未找到标记
            return stbi__err("expected marker", "Corrupt JPEG");

        case 0xDD: // DRI - 指定重启间隔
            if (stbi__get16be(z->s) != 4)
                return stbi__err("bad DRI len", "Corrupt JPEG");
            z->restart_interval = stbi__get16be(z->s);
            return 1;

        case 0xDB: // DQT - 定义量化表
            L = stbi__get16be(z->s) - 2;
            while (L > 0) {
                int q = stbi__get8(z->s);
                int p = q >> 4, sixteen = (p != 0);
                int t = q & 15, i;
                if (p != 0 && p != 1)
                    return stbi__err("bad DQT type", "Corrupt JPEG");
                if (t > 3)
                    return stbi__err("bad DQT table", "Corrupt JPEG");

                for (i = 0; i < 64; ++i)
                    z->dequant[t][stbi__jpeg_dezigzag[i]] = (stbi__uint16)(sixteen ? stbi__get16be(z->s) : stbi__get8(z->s));
                L -= (sixteen ? 129 : 65);
            }
            return L == 0;
    // 当标记为 0xC4 时，表示定义霍夫曼表
    case 0xC4: // DHT - define huffman table
        // 读取 16 位大端序的值，并减去 2
        L = stbi__get16be(z->s) - 2;
        // 当 L 大于 0 时，执行循环
        while (L > 0) {
            // 定义变量 v，大小为 8 位无符号字符
            stbi_uc * v;
            // 定义数组 sizes，大小为 16，定义变量 i 和 n，并初始化为 0
            int sizes[16], i, n = 0;
            // 读取 8 位无符号字符，赋值给变量 q
            int q = stbi__get8(z->s);
            // 将 q 右移 4 位，赋值给变量 tc
            int tc = q >> 4;
            // 将 q 与 15 进行与操作，赋值给变量 th
            int th = q & 15;
            // 如果 tc 大于 1 或 th 大于 3，则返回错误
            if (tc > 1 || th > 3)
                return stbi__err("bad DHT header", "Corrupt JPEG");
            // 循环 16 次，读取 8 位无符号字符，存入 sizes 数组
            for (i = 0; i < 16; ++i) {
                sizes[i] = stbi__get8(z->s);
                n += sizes[i];
            }
            // 如果 n 大于 256，则返回错误
            if (n > 256)
                return stbi__err("bad DHT header", "Corrupt JPEG"); // Loop over i < n would write past end of values!
            // 减去 17
            L -= 17;
            // 如果 tc 等于 0
            if (tc == 0) {
                // 如果构建霍夫曼表失败，则返回 0
                if (!stbi__build_huffman(z->huff_dc + th, sizes))
                    return 0;
                // 将 z->huff_dc[th].values 赋值给 v
                v = z->huff_dc[th].values;
            } else {
                // 如果构建霍夫曼表失败，则返回 0
                if (!stbi__build_huffman(z->huff_ac + th, sizes))
                    return 0;
                // 将 z->huff_ac[th].values 赋值给 v
                v = z->huff_ac[th].values;
            }
            // 循环 n 次，读取 8 位无符号字符，存入 v 数组
            for (i = 0; i < n; ++i)
                v[i] = stbi__get8(z->s);
            // 如果 tc 不等于 0，则构建快速霍夫曼表
            if (tc != 0)
                stbi__build_fast_ac(z->fast_ac[th], z->huff_ac + th);
            // 减去 n
            L -= n;
        }
        // 返回 L 是否等于 0
        return L == 0;
    }

    // 检查是否为注释块或 APP 块
    # 如果标记字节的值在 0xE0 到 0xEF 之间，或者等于 0xFE
    if ((m >= 0xE0 && m <= 0xEF) || m == 0xFE) {
        # 读取下一个 16 位的大端数据
        L = stbi__get16be(z->s);
        # 如果读取的数据小于 2
        if (L < 2) {
            # 如果标记字节的值等于 0xFE，返回错误信息
            if (m == 0xFE)
                return stbi__err("bad COM len", "Corrupt JPEG");
            # 否则返回错误信息
            else
                return stbi__err("bad APP len", "Corrupt JPEG");
        }
        # 减去 2，得到剩余的长度
        L -= 2;

        # 如果标记字节的值等于 0xE0 并且剩余长度大于等于 5
        if (m == 0xE0 && L >= 5) { // JFIF APP0 segment
            # 定义 JFIF APP0 段的标记
            static const unsigned char tag[5] = {'J', 'F', 'I', 'F', '\0'};
            # 初始化标记是否匹配的变量
            int ok = 1;
            int i;
            # 遍历标记数组
            for (i = 0; i < 5; ++i)
                # 如果读取的字节与标记不匹配，将 ok 置为 0
                if (stbi__get8(z->s) != tag[i])
                    ok = 0;
            # 减去已经读取的长度
            L -= 5;
            # 如果标记匹配，设置 JFIF 标记为 1
            if (ok)
                z->jfif = 1;
        } else if (m == 0xEE && L >= 12) { // Adobe APP14 segment
            # 定义 Adobe APP14 段的标记
            static const unsigned char tag[6] = {'A', 'd', 'o', 'b', 'e', '\0'};
            # 初始化标记是否匹配的变量
            int ok = 1;
            int i;
            # 遍历标记数组
            for (i = 0; i < 6; ++i)
                # 如果读取的字节与标记不匹配，将 ok 置为 0
                if (stbi__get8(z->s) != tag[i])
                    ok = 0;
            # 减去已经读取的长度
            L -= 6;
            # 如果标记匹配
            if (ok) {
                stbi__get8(z->s);                            // version
                stbi__get16be(z->s);                         // flags0
                stbi__get16be(z->s);                         // flags1
                z->app14_color_transform = stbi__get8(z->s); // color transform
                L -= 6;
            }
        }

        # 跳过剩余长度的数据
        stbi__skip(z->s, L);
        # 返回 1
        return 1;
    }

    # 返回未知标记的错误信息
    return stbi__err("unknown marker", "Corrupt JPEG");
# 在我们看到 SOS 之后处理扫描头部
def stbi__process_scan_header(stbi__jpeg * z):
    # 定义变量 Ls，存储从输入流中读取的 16 位大端数据
    int Ls = stbi__get16be(z->s);
    # 读取扫描组件数
    z->scan_n = stbi__get8(z->s);
    # 如果扫描组件数小于 1 或大于 4，或者大于图像通道数，则返回错误
    if (z->scan_n < 1 || z->scan_n > 4 || z->scan_n > (int)z->s->img_n)
        return stbi__err("bad SOS component count", "Corrupt JPEG");
    # 如果 Ls 不等于 6 + 2 * z->scan_n，则返回错误
    if (Ls != 6 + 2 * z->scan_n)
        return stbi__err("bad SOS len", "Corrupt JPEG");
    # 遍历扫描组件数
    for (i = 0; i < z->scan_n; ++i):
        # 读取 id 和 q
        int id = stbi__get8(z->s), which;
        int q = stbi__get8(z->s);
        # 查找与 id 匹配的图像组件
        for (which = 0; which < z->s->img_n; ++which)
            if (z->img_comp[which].id == id)
                break;
        # 如果没有匹配的组件，则返回 0
        if (which == z->s->img_n)
            return 0; // no match
        # 设置图像组件的 hd 和 ha
        z->img_comp[which].hd = q >> 4;
        if (z->img_comp[which].hd > 3)
            return stbi__err("bad DC huff", "Corrupt JPEG");
        z->img_comp[which].ha = q & 15;
        if (z->img_comp[which].ha > 3)
            return stbi__err("bad AC huff", "Corrupt JPEG");
        # 设置 order[i] 为 which
        z->order[i] = which;

    # 处理特殊标记
    {
        int aa;
        # 读取 spec_start 和 spec_end
        z->spec_start = stbi__get8(z->s);
        z->spec_end = stbi__get8(z->s); # should be 63, but might be 0
        aa = stbi__get8(z->s);
        # 设置 succ_high 和 succ_low
        z->succ_high = (aa >> 4);
        z->succ_low = (aa & 15);
        # 如果是渐进式扫描，则进行额外的检查
        if (z->progressive):
            if (z->spec_start > 63 || z->spec_end > 63 || z->spec_start > z->spec_end || z->succ_high > 13 || z->succ_low > 13)
                return stbi__err("bad SOS", "Corrupt JPEG");
        else:
            # 如果不是渐进式扫描，则进行额外的检查
            if (z->spec_start != 0)
                return stbi__err("bad SOS", "Corrupt JPEG");
            if (z->succ_high != 0 || z->succ_low != 0)
                return stbi__err("bad SOS", "Corrupt JPEG");
            z->spec_end = 63;
    }

    # 返回 1
    return 1;

# 释放 JPEG 组件
def stbi__free_jpeg_components(stbi__jpeg * z, int ncomp, int why):
    # 定义变量 i
    int i;
    # 遍历图像组件，释放原始数据和系数
    for (i = 0; i < ncomp; ++i) {
        # 如果图像组件的原始数据存在，则释放内存并将指针置为空
        if (z->img_comp[i].raw_data) {
            STBI_FREE(z->img_comp[i].raw_data);
            z->img_comp[i].raw_data = NULL;
            z->img_comp[i].data = NULL;
        }
        # 如果图像组件的原始系数存在，则释放内存并将指针置为0
        if (z->img_comp[i].raw_coeff) {
            STBI_FREE(z->img_comp[i].raw_coeff);
            z->img_comp[i].raw_coeff = 0;
            z->img_comp[i].coeff = 0;
        }
        # 如果图像组件的行缓冲存在，则释放内存并将指针置为空
        if (z->img_comp[i].linebuf) {
            STBI_FREE(z->img_comp[i].linebuf);
            z->img_comp[i].linebuf = NULL;
        }
    }
    # 返回原因
    return why;
    # 处理帧头部信息
static int stbi__process_frame_header(stbi__jpeg * z, int scan) {
    # 获取上下文对象
    stbi__context * s = z->s;
    # 定义变量
    int Lf, p, i, q, h_max = 1, v_max = 1, c;
    # 读取帧头部长度
    Lf = stbi__get16be(s);
    # 如果长度小于11，返回错误
    if (Lf < 11)
        return stbi__err("bad SOF len", "Corrupt JPEG"); // JPEG
    # 读取精度
    p = stbi__get8(s);
    # 如果精度不是8，返回错误
    if (p != 8)
        return stbi__err("only 8-bit", "JPEG format not supported: 8-bit only"); // JPEG baseline
    # 读取图像高度
    s->img_y = stbi__get16be(s);
    # 如果高度为0，返回错误
    if (s->img_y == 0)
        return stbi__err("no header height",
                         "JPEG format not supported: delayed height"); // Legal, but we don't handle it--but neither does IJG
    # 读取图像宽度
    s->img_x = stbi__get16be(s);
    # 如果宽度为0，返回错误
    if (s->img_x == 0)
        return stbi__err("0 width", "Corrupt JPEG"); // JPEG requires
    # 如果高度或宽度超过最大尺寸限制，返回错误
    if (s->img_y > STBI_MAX_DIMENSIONS)
        return stbi__err("too large", "Very large image (corrupt?)");
    if (s->img_x > STBI_MAX_DIMENSIONS)
        return stbi__err("too large", "Very large image (corrupt?)");
    # 读取分量数
    c = stbi__get8(s);
    # 如果分量数不是3、1或4，返回错误
    if (c != 3 && c != 1 && c != 4)
        return stbi__err("bad component count", "Corrupt JPEG");
    # 设置图像通道数
    s->img_n = c;
    # 初始化图像分量数据和行缓冲区
    for (i = 0; i < c; ++i) {
        z->img_comp[i].data = NULL;
        z->img_comp[i].linebuf = NULL;
    }

    # 如果帧头部长度不符合规范，返回错误
    if (Lf != 8 + 3 * s->img_n)
        return stbi__err("bad SOF len", "Corrupt JPEG");

    # 初始化RGB标志
    z->rgb = 0;
    # 遍历图像分量
    for (i = 0; i < s->img_n; ++i) {
        static const unsigned char rgb[3] = {'R', 'G', 'B'};
        # 读取分量ID
        z->img_comp[i].id = stbi__get8(s);
        # 如果图像分量数为3且ID符合RGB顺序，增加RGB标志
        if (s->img_n == 3 && z->img_comp[i].id == rgb[i])
            ++z->rgb;
        # 读取水平和垂直采样因子
        q = stbi__get8(s);
        z->img_comp[i].h = (q >> 4);
        # 如果水平采样因子不符合规范，返回错误
        if (!z->img_comp[i].h || z->img_comp[i].h > 4)
            return stbi__err("bad H", "Corrupt JPEG");
        z->img_comp[i].v = q & 15;
        # 如果垂直采样因子不符合规范，返回错误
        if (!z->img_comp[i].v || z->img_comp[i].v > 4)
            return stbi__err("bad V", "Corrupt JPEG");
        # 读取量化表索引
        z->img_comp[i].tq = stbi__get8(s);
        # 如果量化表索引超过3，返回错误
        if (z->img_comp[i].tq > 3)
            return stbi__err("bad TQ", "Corrupt JPEG");
    }

    // 如果扫描模式不是加载模式，返回1
    if (scan != STBI__SCAN_load)
        return 1;

    // 检查图像大小是否合法
    if (!stbi__mad3sizes_valid(s->img_x, s->img_y, s->img_n, 0))
        return stbi__err("too large", "Image too large to decode");

    // 遍历图像通道，找到最大的水平和垂直采样因子
    for (i = 0; i < s->img_n; ++i) {
        if (z->img_comp[i].h > h_max)
            h_max = z->img_comp[i].h;
        if (z->img_comp[i].v > v_max)
            v_max = z->img_comp[i].v;
    }

    // 检查平面子采样因子是否为整数比率
    // 我们的重采样器无法处理分数比率
    // 并且我从未见过一个非损坏的 JPEG 文件实际上使用它们
    for (i = 0; i < s->img_n; ++i) {
        if (h_max % z->img_comp[i].h != 0)
            return stbi__err("bad H", "Corrupt JPEG");
        if (v_max % z->img_comp[i].v != 0)
            return stbi__err("bad V", "Corrupt JPEG");
    }

    // 计算交错 MCU 信息
    z->img_h_max = h_max;
    z->img_v_max = v_max;
    z->img_mcu_w = h_max * 8;
    z->img_mcu_h = v_max * 8;
    // 这些大小不能超过17位
    z->img_mcu_x = (s->img_x + z->img_mcu_w - 1) / z->img_mcu_w;
    z->img_mcu_y = (s->img_y + z->img_mcu_h - 1) / z->img_mcu_h;
    for (i = 0; i < s->img_n; ++i) {
        // 遍历图像组件的数量
        z->img_comp[i].x = (s->img_x * z->img_comp[i].h + h_max - 1) / h_max;
        // 计算图像组件的水平采样因子
        z->img_comp[i].y = (s->img_y * z->img_comp[i].v + v_max - 1) / v_max;
        // 计算图像组件的垂直采样因子
        // 为了简化生成，我们将分配足够的内存来解码使用交错MCU和它们的大块（例如，在宽度为33的图像上使用16x16 iMCU）产生的虚假超大数据；我们在颜色空间转换之前不会丢弃额外的数据
        //
        // img_mcu_x, img_mcu_y: <=17 bits; comp[i].h and .v are <=4 (checked earlier)
        // 所以这些乘法不会在32位整数下溢出（我们要求）
        z->img_comp[i].w2 = z->img_mcu_x * z->img_comp[i].h * 8;
        // 计算图像组件的水平大小
        z->img_comp[i].h2 = z->img_mcu_y * z->img_comp[i].v * 8;
        // 计算图像组件的垂直大小
        z->img_comp[i].coeff = 0;
        z->img_comp[i].raw_coeff = 0;
        z->img_comp[i].linebuf = NULL;
        // 分配足够的内存来存储解码后的图像数据
        z->img_comp[i].raw_data = stbi__malloc_mad2(z->img_comp[i].w2, z->img_comp[i].h2, 15);
        if (z->img_comp[i].raw_data == NULL)
            return stbi__free_jpeg_components(z, i + 1, stbi__err("outofmem", "Out of memory"));
        // 为了使用mmx/sse对idct进行块对齐
        z->img_comp[i].data = (stbi_uc *)(((size_t)z->img_comp[i].raw_data + 15) & ~15);
        if (z->progressive) {
            // w2, h2是8的倍数（见上文）
            z->img_comp[i].coeff_w = z->img_comp[i].w2 / 8;
            z->img_comp[i].coeff_h = z->img_comp[i].h2 / 8;
            // 为渐进式扫描分配内存
            z->img_comp[i].raw_coeff = stbi__malloc_mad3(z->img_comp[i].w2, z->img_comp[i].h2, sizeof(short), 15);
            if (z->img_comp[i].raw_coeff == NULL)
                return stbi__free_jpeg_components(z, i + 1, stbi__err("outofmem", "Out of memory"));
            // 对渐进式扫描的系数进行块对齐
            z->img_comp[i].coeff = (short *)(((size_t)z->img_comp[i].raw_coeff + 15) & ~15);
        }
    }

    return 1;
// 使用比较操作符，因为在某些情况下我们处理多个情况（例如 SOF）
#define stbi__DNL(x) ((x) == 0xdc)
#define stbi__SOI(x) ((x) == 0xd8)
#define stbi__EOI(x) ((x) == 0xd9)
#define stbi__SOF(x) ((x) == 0xc0 || (x) == 0xc1 || (x) == 0xc2)
#define stbi__SOS(x) ((x) == 0xda)

#define stbi__SOF_progressive(x) ((x) == 0xc2)

// 解码 JPEG 头部信息
static int stbi__decode_jpeg_header(stbi__jpeg * z, int scan) {
    int m;
    z->jfif = 0; // 初始化 JFIF 为 0
    z->app14_color_transform = -1; // 有效值为 0,1,2
    z->marker = STBI__MARKER_none; // 初始化缓存的标记为空
    m = stbi__get_marker(z); // 获取标记
    if (!stbi__SOI(m)) // 如果不是 SOI 标记，则返回错误
        return stbi__err("no SOI", "Corrupt JPEG");
    if (scan == STBI__SCAN_type) // 如果扫描类型为 STBI__SCAN_type，则返回 1
        return 1;
    m = stbi__get_marker(z); // 获取标记
    while (!stbi__SOF(m)) { // 如果不是 SOF 标记
        if (!stbi__process_marker(z, m)) // 处理标记，如果失败则返回 0
            return 0;
        m = stbi__get_marker(z); // 获取标记
        while (m == STBI__MARKER_none) { // 如果标记为空
            // 一些文件在它们的块之后有额外的填充，所以我们扫描一下
            if (stbi__at_eof(z->s)) // 如果已经到达文件末尾，则返回错误
                return stbi__err("no SOF", "Corrupt JPEG");
            m = stbi__get_marker(z); // 获取标记
        }
    }
    z->progressive = stbi__SOF_progressive(m); // 判断是否为渐进式 JPEG
    if (!stbi__process_frame_header(z, scan)) // 处理帧头信息，如果失败则返回 0
        return 0;
    return 1;
}

// 跳过 JPEG 结尾处的无用数据
static int stbi__skip_jpeg_junk_at_end(stbi__jpeg * j) {
    // 一些 JPEG 文件在结尾处有无用数据，跳过它们，但如果找到类似有效标记的内容，则在那里恢复
    # 当流未到达结尾时执行循环
    while (!stbi__at_eof(j->s)) {
        # 从流中获取一个字节
        int x = stbi__get8(j->s);
        # 如果获取的字节为255，可能是一个标记
        while (x == 255) { // might be a marker
            # 如果流已经到达结尾，返回标记为none
            if (stbi__at_eof(j->s))
                return STBI__MARKER_none;
            # 继续获取下一个字节
            x = stbi__get8(j->s);
            # 如果获取的字节不是0x00也不是0xff
            if (x != 0x00 && x != 0xff) {
                # 不是填充的零或者另一个标记的前导，看起来是一个实际的标记，返回它
                return x;
            }
            # 填充的零现在为x=0，结束循环，意味着我们回到常规扫描循环
            # 重复的0xff继续尝试读取标记的下一个字节
        }
    }
    # 返回标记为none
    return STBI__MARKER_none;
# 将图像解码为 YCbCr 格式
def stbi__decode_jpeg_image(stbi__jpeg * j):
    # 初始化变量 m
    int m;
    # 遍历四次
    for (m = 0; m < 4; m++):
        # 将图像组件的原始数据和系数数据设置为 NULL
        j->img_comp[m].raw_data = NULL;
        j->img_comp[m].raw_coeff = NULL;
    # 重置重启间隔
    j->restart_interval = 0;
    # 解码 JPEG 头部信息
    if (!stbi__decode_jpeg_header(j, STBI__SCAN_load)):
        return 0;
    # 获取标记
    m = stbi__get_marker(j);
    # 循环直到遇到结束标记
    while (!stbi__EOI(m)):
        # 如果是扫描开始标记
        if (stbi__SOS(m)):
            # 处理扫描头部信息
            if (!stbi__process_scan_header(j)):
                return 0;
            # 解析熵编码数据
            if (!stbi__parse_entropy_coded_data(j)):
                return 0;
            # 如果标记为 none，则跳过 JPEG 末尾的垃圾数据
            if (j->marker == STBI__MARKER_none):
                j->marker = stbi__skip_jpeg_junk_at_end(j);
                # 如果在没有遇到标记的情况下到达文件末尾，stbi__get_marker() 将失败并最终返回 0
            m = stbi__get_marker(j);
            # 如果是重启标记
            if (STBI__RESTART(m)):
                m = stbi__get_marker(j);
        # 如果是定义数值线标记
        elif (stbi__DNL(m)):
            # 获取长度和高度
            int Ld = stbi__get16be(j->s);
            stbi__uint32 NL = stbi__get16be(j->s);
            # 如果长度不为 4，则返回错误
            if (Ld != 4):
                return stbi__err("bad DNL len", "Corrupt JPEG");
            # 如果高度不等于图像高度，则返回错误
            if (NL != j->s->img_y):
                return stbi__err("bad DNL height", "Corrupt JPEG");
            m = stbi__get_marker(j);
        else:
            # 处理标记
            if (!stbi__process_marker(j, m)):
                return 1;
            m = stbi__get_marker(j);
    # 如果是渐进式扫描，则完成 JPEG 解码
    if (j->progressive):
        stbi__jpeg_finish(j);
    return 1;

# 静态的 JFIF 居中重采样（跨块边界）
typedef stbi_uc * (*resample_row_func)(stbi_uc * out, stbi_uc * in0, stbi_uc * in1, int w, int hs);

# 定义宏函数 stbi__div4
#define stbi__div4(x) ((stbi_uc)((x) >> 2))

# 静态的重采样函数，处理一行像素数据
static stbi_uc * resample_row_1(stbi_uc * out, stbi_uc * in_near, stbi_uc * in_far, int w, int hs):
    # 未使用的参数
    STBI_NOTUSED(out);
    STBI_NOTUSED(in_far);
    STBI_NOTUSED(w);
    STBI_NOTUSED(hs);
    # 返回近处输入像素数据
    return in_near;
// 对输入的像素进行垂直双线性插值，生成两倍高度的输出像素
static stbi_uc * stbi__resample_row_v_2(stbi_uc * out, stbi_uc * in_near, stbi_uc * in_far, int w, int hs) {
    // 需要为每个输入像素生成两个垂直方向的样本
    int i;
    STBI_NOTUSED(hs); // 忽略未使用的参数 hs
    for (i = 0; i < w; ++i)
        out[i] = stbi__div4(3 * in_near[i] + in_far[i] + 2); // 使用垂直双线性插值计算输出像素值
    return out; // 返回处理后的输出像素
}

// 对输入的像素进行水平双线性插值，生成两倍宽度的输出像素
static stbi_uc * stbi__resample_row_h_2(stbi_uc * out, stbi_uc * in_near, stbi_uc * in_far, int w, int hs) {
    // 需要为每个输入像素生成两个水平方向的样本
    int i;
    stbi_uc * input = in_near; // 将输入像素赋值给 input

    if (w == 1) {
        // 如果只有一个样本，无法进行任何插值
        out[0] = out[1] = input[0]; // 输出像素等于输入像素
        return out; // 返回处理后的输出像素
    }

    out[0] = input[0]; // 输出的第一个像素等于输入的第一个像素
    out[1] = stbi__div4(input[0] * 3 + input[1] + 2); // 使用水平双线性插值计算输出的第二个像素
    for (i = 1; i < w - 1; ++i) {
        int n = 3 * input[i] + 2;
        out[i * 2 + 0] = stbi__div4(n + input[i - 1]); // 使用水平双线性插值计算输出的像素
        out[i * 2 + 1] = stbi__div4(n + input[i + 1]); // 使用水平双线性插值计算输出的像素
    }
    out[i * 2 + 0] = stbi__div4(input[w - 2] * 3 + input[w - 1] + 2); // 使用水平双线性插值计算输出的倒数第二个像素
    out[i * 2 + 1] = input[w - 1]; // 输出的最后一个像素等于输入的最后一个像素

    STBI_NOTUSED(in_far); // 忽略未使用的参数 in_far
    STBI_NOTUSED(hs); // 忽略未使用的参数 hs

    return out; // 返回处理后的输出像素
}

// 对输入的像素进行水平和垂直双线性插值，生成四倍宽高的输出像素
static stbi_uc * stbi__resample_row_hv_2(stbi_uc * out, stbi_uc * in_near, stbi_uc * in_far, int w, int hs) {
    // 需要为每个输入像素生成 2x2 的输出像素
    int i, t0, t1;
    if (w == 1) {
        out[0] = out[1] = stbi__div4(3 * in_near[0] + in_far[0] + 2); // 使用水平和垂直双线性插值计算输出像素
        return out; // 返回处理后的输出像素
    }

    t1 = 3 * in_near[0] + in_far[0];
    out[0] = stbi__div4(t1 + 2); // 使用水平和垂直双线性插值计算输出的第一个像素
    for (i = 1; i < w; ++i) {
        t0 = t1;
        t1 = 3 * in_near[i] + in_far[i];
        out[i * 2 - 1] = stbi__div16(3 * t0 + t1 + 8); // 使用水平和垂直双线性插值计算输出的像素
        out[i * 2] = stbi__div16(3 * t1 + t0 + 8); // 使用水平和垂直双线性插值计算输出的像素
    }
    out[w * 2 - 1] = stbi__div4(t1 + 2); // 使用水平和垂直双线性插值计算输出的最后一个像素

    STBI_NOTUSED(hs); // 忽略未使用的参数 hs

    return out; // 返回处理后的输出像素
}

#if defined(STBI_SSE2) || defined(STBI_NEON)
static stbi_uc * stbi__resample_row_hv_2_simd(stbi_uc * out, stbi_uc * in_near, stbi_uc * in_far, int w, int hs) {
    // 需要为输入的每个样本生成2x2的样本
    int i = 0, t0, t1;

    if (w == 1) {
        out[0] = out[1] = stbi__div4(3 * in_near[0] + in_far[0] + 2);
        return out;
    }

    t1 = 3 * in_near[0] + in_far[0];
    // 处理每组8个像素，尽可能多地处理
    // 注意：在这个循环中，我们无法处理行中的最后一个像素
    // 因为我们需要处理滤波器的边界条件。
    for (; i < ((w - 1) & ~7); i += 8) {
#elif defined(STBI_NEON)
        // 如果定义了 STBI_NEON，则执行以下代码块
        // 加载并执行垂直滤波处理
        // 这里使用了 3*x + y = 4*x + (y - x) 的公式
        uint8x8_t farb = vld1_u8(in_far + i);
        uint8x8_t nearb = vld1_u8(in_near + i);
        int16x8_t diff = vreinterpretq_s16_u16(vsubl_u8(farb, nearb));
        int16x8_t nears = vreinterpretq_s16_u16(vshll_n_u8(nearb, 2));
        int16x8_t curr = vaddq_s16(nears, diff); // 当前行

        // 水平滤波处理与基于当前行的移位版本相同。"prev" 是当前行向右移动 1 个像素；我们需要插入前一个像素值（来自 t1）。
        // "next" 是当前行向左移动 1 个像素，其中添加了下一个 8 个像素块的第一个像素。
        int16x8_t prv0 = vextq_s16(curr, curr, 7);
        int16x8_t nxt0 = vextq_s16(curr, curr, 1);
        int16x8_t prev = vsetq_lane_s16(t1, prv0, 0);
        int16x8_t next = vsetq_lane_s16(3 * in_near[i + 8] + in_far[i + 8], nxt0, 7);

        // 水平滤波，由于方便，使用了多相实现：
        // 偶数像素 = 3*cur + prev = cur*4 + (prev - cur)
        // 奇数像素 = 3*cur + next = cur*4 + (next - cur)
        // 注意共享的项。
        int16x8_t curs = vshlq_n_s16(curr, 2);
        int16x8_t prvd = vsubq_s16(prev, curr);
        int16x8_t nxtd = vsubq_s16(next, curr);
        int16x8_t even = vaddq_s16(curs, prvd);
        int16x8_t odd = vaddq_s16(curs, nxtd);

        // 撤销缩放和四舍五入，然后交错存储偶数/奇数相位
        uint8x8x2_t o;
        o.val[0] = vqrshrun_n_s16(even, 4);
        o.val[1] = vqrshrun_n_s16(odd, 4);
        vst2_u8(out + i * 2, o);
#endif

        // 下一次迭代的“前一个”值
        t1 = 3 * in_near[i + 7] + in_far[i + 7];
    }

    t0 = t1;
    t1 = 3 * in_near[i] + in_far[i];
    out[i * 2] = stbi__div16(3 * t1 + t0 + 8);
    # 遍历从 i+1 到 w 的范围
    for (++i; i < w; ++i) {
        # 保存 t1 的值到 t0
        t0 = t1;
        # 计算 t1 的新值
        t1 = 3 * in_near[i] + in_far[i];
        # 将计算结果存入输出数组中
        out[i * 2 - 1] = stbi__div16(3 * t0 + t1 + 8);
        out[i * 2] = stbi__div16(3 * t1 + t0 + 8);
    }
    # 将最后一个元素的值存入输出数组中
    out[w * 2 - 1] = stbi__div4(t1 + 2);

    # 忽略未使用的变量 hs

    # 返回输出数组
    return out;
}
#endif

# 通用的行重采样函数，使用最近邻插值
static stbi_uc * stbi__resample_row_generic(stbi_uc * out, stbi_uc * in_near, stbi_uc * in_far, int w, int hs) {
    # 定义变量 i, j
    int i, j;
    # 声明未使用的参数 in_far
    STBI_NOTUSED(in_far);
    # 遍历输入的像素数据，进行重采样
    for (i = 0; i < w; ++i)
        for (j = 0; j < hs; ++j)
            out[i * hs + j] = in_near[i];
    # 返回重采样后的像素数据
    return out;
}

# 这是一个降低精度的 YCbCr-to-RGB 转换计算，用于确保代码在 SIMD 和标量中产生相同的结果
# 定义宏 stbi__float2fixed，将浮点数转换为固定点数
#define stbi__float2fixed(x) (((int)((x)*4096.0f + 0.5f)) << 8)
static void stbi__YCbCr_to_RGB_row(stbi_uc * out, const stbi_uc * y, const stbi_uc * pcb, const stbi_uc * pcr, int count,
                                   int step) {
    # 定义变量 i
    int i;
    # 遍历像素数据，进行 YCbCr 到 RGB 转换
    for (i = 0; i < count; ++i) {
        # 对 Y 值进行固定点数转换
        int y_fixed = (y[i] << 20) + (1 << 19); # 四舍五入
        int r, g, b;
        int cr = pcr[i] - 128;
        int cb = pcb[i] - 128;
        # 计算 RGB 值
        r = y_fixed + cr * stbi__float2fixed(1.40200f);
        g = y_fixed + (cr * -stbi__float2fixed(0.71414f)) + ((cb * -stbi__float2fixed(0.34414f)) & 0xffff0000);
        b = y_fixed + cb * stbi__float2fixed(1.77200f);
        r >>= 20;
        g >>= 20;
        b >>= 20;
        # 对 RGB 值进行范围限制
        if ((unsigned)r > 255) {
            if (r < 0)
                r = 0;
            else
                r = 255;
        }
        if ((unsigned)g > 255) {
            if (g < 0)
                g = 0;
            else
                g = 255;
        }
        if ((unsigned)b > 255) {
            if (b < 0)
                b = 0;
            else
                b = 255;
        }
        # 将计算得到的 RGB 值写入输出数组
        out[0] = (stbi_uc)r;
        out[1] = (stbi_uc)g;
        out[2] = (stbi_uc)b;
        out[3] = 255;
        out += step;
    }
}

# 如果支持 SSE2 或 NEON 指令集，则使用 SIMD 加速 YCbCr 到 RGB 转换
#if defined(STBI_SSE2) || defined(STBI_NEON)
static void stbi__YCbCr_to_RGB_simd(stbi_uc * out, stbi_uc const * y, stbi_uc const * pcb, stbi_uc const * pcr, int count,
                                    int step) {
    # 定义变量 i
    int i = 0;

#ifdef STBI_SSE2
    // 当 step == 3 时，在最终交错中非常丑陋，我不确定它在实践中是否有用（例如，你不会在纹理中使用它）。
    // 因此，只加速 step == 4 的情况。
    }
#endif

#ifdef STBI_NEON
    // 在这个版本中，添加 step=3 的支持会比较容易。但是是否有需求呢？
    if (step == 4) {
        // 这是一个相当直接的实现，而且并不是特别优化过的。
        uint8x8_t signflip = vdup_n_u8(0x80);  // 创建一个包含 0x80 的 8 位无符号整数向量
        int16x8_t cr_const0 = vdupq_n_s16((short)(1.40200f * 4096.0f + 0.5f));  // 创建一个包含计算结果的 8 个 16 位有符号整数向量
        int16x8_t cr_const1 = vdupq_n_s16(-(short)(0.71414f * 4096.0f + 0.5f));  // 创建一个包含计算结果的 8 个 16 位有符号整数向量
        int16x8_t cb_const0 = vdupq_n_s16(-(short)(0.34414f * 4096.0f + 0.5f));  // 创建一个包含计算结果的 8 个 16 位有符号整数向量
        int16x8_t cb_const1 = vdupq_n_s16((short)(1.77200f * 4096.0f + 0.5f));  // 创建一个包含计算结果的 8 个 16 位有符号整数向量

        for (; i + 7 < count; i += 8) {
            // 加载
            uint8x8_t y_bytes = vld1_u8(y + i);  // 从地址 y + i 处加载 8 个 8 位无符号整数，存储到向量中
            uint8x8_t cr_bytes = vld1_u8(pcr + i);  // 从地址 pcr + i 处加载 8 个 8 位无符号整数，存储到向量中
            uint8x8_t cb_bytes = vld1_u8(pcb + i);  // 从地址 pcb + i 处加载 8 个 8 位无符号整数，存储到向量中
            int8x8_t cr_biased = vreinterpret_s8_u8(vsub_u8(cr_bytes, signflip));  // 将 cr_bytes 中的值减去 signflip 中的值，然后转换为 8 位有符号整数
            int8x8_t cb_biased = vreinterpret_s8_u8(vsub_u8(cb_bytes, signflip));  // 将 cb_bytes 中的值减去 signflip 中的值，然后转换为 8 位有符号整数

            // 扩展为 s16
            int16x8_t yws = vreinterpretq_s16_u16(vshll_n_u8(y_bytes, 4));  // 将 y_bytes 中的值左移 4 位，然后转换为 16 位有符号整数
            int16x8_t crw = vshll_n_s8(cr_biased, 7);  // 将 cr_biased 中的值左移 7 位，然后转换为 16 位有符号整数
            int16x8_t cbw = vshll_n_s8(cb_biased, 7);  // 将 cb_biased 中的值左移 7 位，然后转换为 16 位有符号整数

            // 颜色转换
            int16x8_t cr0 = vqdmulhq_s16(crw, cr_const0);  // 两个 16 位有符号整数向量的高位乘法
            int16x8_t cb0 = vqdmulhq_s16(cbw, cb_const0);  // 两个 16 位有符号整数向量的高位乘法
            int16x8_t cr1 = vqdmulhq_s16(crw, cr_const1);  // 两个 16 位有符号整数向量的高位乘法
            int16x8_t cb1 = vqdmulhq_s16(cbw, cb_const1);  // 两个 16 位有符号整数向量的高位乘法
            int16x8_t rws = vaddq_s16(yws, cr0);  // 两个 16 位有符号整数向量的加法
            int16x8_t gws = vaddq_s16(vaddq_s16(yws, cb0), cr1);  // 两个 16 位有符号整数向量的加法
            int16x8_t bws = vaddq_s16(yws, cb1);  // 两个 16 位有符号整数向量的加法

            // 撤销缩放，四舍五入，转换为字节
            uint8x8x4_t o;  // 创建一个包含 4 个 8 位无符号整数向量的结构体
            o.val[0] = vqrshrun_n_s16(rws, 4);  // 对 16 位有符号整数向量进行右移和无符号转换
            o.val[1] = vqrshrun_n_s16(gws, 4);  // 对 16 位有符号整数向量进行右移和无符号转换
            o.val[2] = vqrshrun_n_s16(bws, 4);  // 对 16 位有符号整数向量进行右移和无符号转换
            o.val[3] = vdup_n_u8(255);  // 创建一个包含 255 的 8 位无符号整数向量

            // 存储，交错存储 r/g/b/a
            vst4_u8(out, o);  // 将结构体 o 中的值存储到地址 out 处
            out += 8 * 4;  // 更新地址 out
        }
    }
#endif
    // 循环遍历处理每个像素点
    for (; i < count; ++i) {
        // 对亮度进行处理，左移20位并加上1<<19，相当于进行四舍五入
        int y_fixed = (y[i] << 20) + (1 << 19); // rounding
        int r, g, b;
        // 计算红色通道的值
        int cr = pcr[i] - 128;
        r = y_fixed + cr * stbi__float2fixed(1.40200f);
        // 计算绿色通道的值
        g = y_fixed + cr * -stbi__float2fixed(0.71414f) + ((cb * -stbi__float2fixed(0.34414f)) & 0xffff0000);
        // 计算蓝色通道的值
        b = y_fixed + cb * stbi__float2fixed(1.77200f);
        // 右移20位，相当于进行取整
        r >>= 20;
        g >>= 20;
        b >>= 20;
        // 如果超出了颜色值范围，进行边界处理
        if ((unsigned)r > 255) {
            if (r < 0)
                r = 0;
            else
                r = 255;
        }
        if ((unsigned)g > 255) {
            if (g < 0)
                g = 0;
            else
                g = 255;
        }
        if ((unsigned)b > 255) {
            if (b < 0)
                b = 0;
            else
                b = 255;
        }
        // 将处理后的颜色值写入输出数组
        out[0] = (stbi_uc)r;
        out[1] = (stbi_uc)g;
        out[2] = (stbi_uc)b;
        out[3] = 255;
        // 更新输出数组的位置
        out += step;
    }
// 结束预处理指令
#endif

// 设置 JPEG 的内核
static void stbi__setup_jpeg(stbi__jpeg * j) {
    // 设置 IDCT 块内核
    j->idct_block_kernel = stbi__idct_block;
    // 设置 YCbCr 转换为 RGB 的内核
    j->YCbCr_to_RGB_kernel = stbi__YCbCr_to_RGB_row;
    // 设置水平和垂直 2 倍重采样的内核
    j->resample_row_hv_2_kernel = stbi__resample_row_hv_2;

    // 如果支持 SSE2
#ifdef STBI_SSE2
    if (stbi__sse2_available()) {
        // 使用 SIMD 实现的 IDCT 块内核
        j->idct_block_kernel = stbi__idct_simd;
        // 使用 SIMD 实现的 YCbCr 转换为 RGB 的内核
        j->YCbCr_to_RGB_kernel = stbi__YCbCr_to_RGB_simd;
        // 使用 SIMD 实现的水平和垂直 2 倍重采样的内核
        j->resample_row_hv_2_kernel = stbi__resample_row_hv_2_simd;
    }
#endif

    // 如果支持 NEON
#ifdef STBI_NEON
    // 使用 SIMD 实现的 IDCT 块内核
    j->idct_block_kernel = stbi__idct_simd;
    // 使用 SIMD 实现的 YCbCr 转换为 RGB 的内核
    j->YCbCr_to_RGB_kernel = stbi__YCbCr_to_RGB_simd;
    // 使用 SIMD 实现的水平和垂直 2 倍重采样的内核
    j->resample_row_hv_2_kernel = stbi__resample_row_hv_2_simd;
#endif
}

// 清理临时的分量缓冲区
static void stbi__cleanup_jpeg(stbi__jpeg * j) { stbi__free_jpeg_components(j, j->s->img_n, 0); }

// 定义结构体 stbi__resample
typedef struct {
    resample_row_func resample;
    stbi_uc *line0, *line1;
    int hs, vs;  // 每个轴上的扩展因子
    int w_lores; // 水平像素预扩展
    int ystep;   // 垂直扩展的进度
    int ypos;    // 预扩展行数
} stbi__resample;

// 快速 0..255 * 0..255 => 0..255 的四舍五入乘法
static stbi_uc stbi__blinn_8x8(stbi_uc x, stbi_uc y) {
    unsigned int t = x * y + 128;
    return (stbi_uc)((t + (t >> 8)) >> 8);
}

// 加载 JPEG 图像
static stbi_uc * load_jpeg_image(stbi__jpeg * z, int * out_x, int * out_y, int * comp, int req_comp) {
    int n, decode_n, is_rgb;
    z->s->img_n = 0; // 使 stbi__cleanup_jpeg 安全

    // 验证 req_comp
    if (req_comp < 0 || req_comp > 4)
        return stbi__errpuc("bad req_comp", "Internal error");

    // 从任何来源加载 JPEG 图像，但保留在 YCbCr 格式
    if (!stbi__decode_jpeg_image(z)) {
        stbi__cleanup_jpeg(z);
        return NULL;
    }

    // 确定要生成的实际分量数
    n = req_comp ? req_comp : z->s->img_n >= 3 ? 3 : 1;
    # 检查图像是否为 RGB 格式
    is_rgb = z->s->img_n == 3 && (z->rgb == 3 || (z->app14_color_transform == 0 && !z->jfif));

    # 如果图像为非 RGB 格式且请求的组件数小于 3，则解码数量为 1；否则解码数量为图像的组件数
    if (z->s->img_n == 3 && n < 3 && !is_rgb)
        decode_n = 1;
    else
        decode_n = z->s->img_n;

    # 如果没有请求任何组件，则无需处理；现在检查以避免后续访问未初始化的 coutput[0]
    if (decode_n <= 0) {
        # 清理 JPEG 数据并返回空指针
        stbi__cleanup_jpeg(z);
        return NULL;
    }

    # 重新采样和颜色转换
// 释放 JPEG 对象占用的内存空间
static void * stbi__jpeg_load(stbi__context * s, int * x, int * y, int * comp, int req_comp, stbi__result_info * ri) {
    // 分配内存空间用于存储 JPEG 对象
    unsigned char * result;
    stbi__jpeg * j = (stbi__jpeg *)stbi__malloc(sizeof(stbi__jpeg));
    // 如果内存分配失败，返回错误信息
    if (!j)
        return stbi__errpuc("outofmem", "Out of memory");
    // 将分配的内存空间清零
    memset(j, 0, sizeof(stbi__jpeg));
    // 忽略未使用的参数
    STBI_NOTUSED(ri);
    // 将 JPEG 对象的输入流设置为给定的上下文
    j->s = s;
    // 设置 JPEG 对象
    stbi__setup_jpeg(j);
    // 加载 JPEG 图像
    result = load_jpeg_image(j, x, y, comp, req_comp);
    // 释放 JPEG 对象占用的内存空间
    STBI_FREE(j);
    // 返回加载的图像数据
    return result;
}

// 检测给定输入流是否为 JPEG 格式
static int stbi__jpeg_test(stbi__context * s) {
    int r;
    // 分配内存空间用于存储 JPEG 对象
    stbi__jpeg * j = (stbi__jpeg *)stbi__malloc(sizeof(stbi__jpeg));
    // 如果内存分配失败，返回错误信息
    if (!j)
        return stbi__err("outofmem", "Out of memory");
    // 将分配的内存空间清零
    memset(j, 0, sizeof(stbi__jpeg));
    // 将 JPEG 对象的输入流设置为给定的上下文
    j->s = s;
    // 设置 JPEG 对象
    stbi__setup_jpeg(j);
    // 解码 JPEG 头信息
    r = stbi__decode_jpeg_header(j, STBI__SCAN_type);
    // 将输入流重置到起始位置
    stbi__rewind(s);
    // 释放 JPEG 对象占用的内存空间
    STBI_FREE(j);
    // 返回解码结果
    return r;
}

// 获取 JPEG 图像的原始信息
static int stbi__jpeg_info_raw(stbi__jpeg * j, int * x, int * y, int * comp) {
    // 如果无法解析 JPEG 头信息，将输入流重置到起始位置并返回错误
    if (!stbi__decode_jpeg_header(j, STBI__SCAN_header)) {
        stbi__rewind(j->s);
        return 0;
    }
    // 如果 x 不为空，将图像宽度赋值给 x
    if (x)
        *x = j->s->img_x;
    // 如果 y 不为空，将图像高度赋值给 y
    if (y)
        *y = j->s->img_y;
    // 如果 comp 不为空，根据图像通道数赋值给 comp
    if (comp)
        *comp = j->s->img_n >= 3 ? 3 : 1;
    // 返回成功获取图像信息
    return 1;
}

// 获取 JPEG 图像的信息
static int stbi__jpeg_info(stbi__context * s, int * x, int * y, int * comp) {
    int result;
    // 分配内存空间用于存储 JPEG 对象
    stbi__jpeg * j = (stbi__jpeg *)(stbi__malloc(sizeof(stbi__jpeg)));
    // 如果内存分配失败，返回错误信息
    if (!j)
        return stbi__err("outofmem", "Out of memory");
    // 将分配的内存空间清零
    memset(j, 0, sizeof(stbi__jpeg));
    // 将 JPEG 对象的输入流设置为给定的上下文
    j->s = s;
    // 获取 JPEG 图像的原始信息
    result = stbi__jpeg_info_raw(j, x, y, comp);
    // 释放 JPEG 对象占用的内存空间
    STBI_FREE(j);
    // 返回获取信息的结果
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

// fast-way is faster to check than jpeg huffman, but slow way is slower
// 定义 STBI__ZFAST_BITS 为 9，用于加速默认表中的所有情况
#define STBI__ZFAST_BITS 9 
// 定义 STBI__ZFAST_MASK 为 (1 << STBI__ZFAST_BITS) - 1
#define STBI__ZFAST_MASK ((1 << STBI__ZFAST_BITS) - 1)
// 定义 STBI__ZNSYMS 为 288，表示字面/长度字母表中的符号数量
#define STBI__ZNSYMS 288 

// zlib 风格的哈夫曼编码
// （JPEG 从左侧打包，zlib 从右侧打包，因此无法共享代码）
typedef struct {
    stbi__uint16 fast[1 << STBI__ZFAST_BITS];
    stbi__uint16 firstcode[16];
    int maxcode[17];
    stbi__uint16 firstsymbol[16];
    stbi_uc size[STBI__ZNSYMS];
    stbi__uint16 value[STBI__ZNSYMS];
} stbi__zhuffman;

// 将 16 位整数 n 的比特位反转
stbi_inline static int stbi__bitreverse16(int n) {
    n = ((n & 0xAAAA) >> 1) | ((n & 0x5555) << 1);
    n = ((n & 0xCCCC) >> 2) | ((n & 0x3333) << 2);
    n = ((n & 0xF0F0) >> 4) | ((n & 0x0F0F) << 4);
    n = ((n & 0xFF00) >> 8) | ((n & 0x00FF) << 8);
    return n;
}

// 将整数 v 的 bits 位反转
stbi_inline static int stbi__bit_reverse(int v, int bits) {
    STBI_ASSERT(bits <= 16);
    // 对于反转 n 位，反转 16 位并进行位移
    // 例如，11 位，反转位并移除 5 位
    return stbi__bitreverse16(v) >> (16 - bits);
}

// 构建哈夫曼树
static int stbi__zbuild_huffman(stbi__zhuffman * z, const stbi_uc * sizelist, int num) {
    int i, k = 0;
    int code, next_code[16], sizes[17];

    // 根据 DEFLATE 规范生成哈夫曼编码
    memset(sizes, 0, sizeof(sizes));
    memset(z->fast, 0, sizeof(z->fast));
    for (i = 0; i < num; ++i)
        ++sizes[sizelist[i]];
    sizes[0] = 0;
    for (i = 1; i < 16; ++i)
        if (sizes[i] > (1 << i))
            return stbi__err("bad sizes", "Corrupt PNG");
    code = 0;
    for (i = 1; i < 16; ++i) {
        next_code[i] = code;
        z->firstcode[i] = (stbi__uint16)code;
        z->firstsymbol[i] = (stbi__uint16)k;
        code = (code + sizes[i]);
        if (sizes[i])
            if (code - 1 >= (1 << i))
                return stbi__err("bad codelengths", "Corrupt PNG");
        z->maxcode[i] = code << (16 - i); // 内部循环的预移位
        code <<= 1;
        k += sizes[i];
    }
    z->maxcode[16] = 0x10000; // 哨兵
}
    # 遍历大小列表中的元素
    for (i = 0; i < num; ++i) {
        # 获取当前大小
        int s = sizelist[i];
        # 如果大小不为0
        if (s) {
            # 计算当前码字的索引
            int c = next_code[s] - z->firstcode[s] + z->firstsymbol[s];
            # 将大小和索引组合成一个16位的值
            stbi__uint16 fastv = (stbi__uint16)((s << 9) | i);
            # 设置码字大小和值
            z->size[c] = (stbi_uc)s;
            z->value[c] = (stbi__uint16)i;
            # 如果大小小于等于STBI__ZFAST_BITS
            if (s <= STBI__ZFAST_BITS) {
                # 对码字进行反转
                int j = stbi__bit_reverse(next_code[s], s);
                # 将fastv值存入fast数组中
                while (j < (1 << STBI__ZFAST_BITS)) {
                    z->fast[j] = fastv;
                    j += (1 << s);
                }
            }
            # 增加下一个码字的索引
            ++next_code[s];
        }
    }
    # 返回1表示执行成功
    return 1;
// 定义了一个结构体 stbi__zbuf，用于存储 zlib-from-memory 实现的 PNG 读取
// 这是因为 PNG 允许任意拆分 zlib 流，而且在结构上让 PNG 调用 ZLIB 再调用 PNG 很麻烦
// 所以我们要求 PNG 读取所有的 IDAT 并将它们组合成一个单一的内存缓冲区
typedef struct {
    stbi_uc *zbuffer, *zbuffer_end; // zlib 缓冲区的起始和结束指针
    int num_bits; // 当前缓冲区中的位数
    stbi__uint32 code_buffer; // 缓冲区中的代码

    char * zout; // 输出缓冲区的指针
    char * zout_start; // 输出缓冲区的起始指针
    char * zout_end; // 输出缓冲区的结束指针
    int z_expandable; // 输出缓冲区是否可扩展

    stbi__zhuffman z_length, z_distance; // 霍夫曼编码
} stbi__zbuf;

// 判断 zlib 缓冲区是否已经读取完毕
stbi_inline static int stbi__zeof(stbi__zbuf * z) { return (z->zbuffer >= z->zbuffer_end); }

// 从 zlib 缓冲区中获取一个字节
stbi_inline static stbi_uc stbi__zget8(stbi__zbuf * z) { return stbi__zeof(z) ? 0 : *z->zbuffer++; }

// 填充缓冲区中的位
static void stbi__fill_bits(stbi__zbuf * z) {
    do {
        if (z->code_buffer >= (1U << z->num_bits)) {
            z->zbuffer = z->zbuffer_end; /* 将其视为 EOF 以便失败 */
            return;
        }
        z->code_buffer |= (unsigned int)stbi__zget8(z) << z->num_bits;
        z->num_bits += 8;
    } while (z->num_bits <= 24);
}

// 从 zlib 缓冲区中接收指定位数的数据
stbi_inline static unsigned int stbi__zreceive(stbi__zbuf * z, int n) {
    unsigned int k;
    if (z->num_bits < n)
        stbi__fill_bits(z);
    k = z->code_buffer & ((1 << n) - 1);
    z->code_buffer >>= n;
    z->num_bits -= n;
    return k;
}

// 通过慢速路径解码霍夫曼编码
static int stbi__zhuffman_decode_slowpath(stbi__zbuf * a, stbi__zhuffman * z) {
    int b, s, k;
    // 通过慢速路径计算未被快速表解析的数据
    // 使用 JPEG 方法，要求 MSbits 在顶部
    k = stbi__bit_reverse(a->code_buffer, 16);
    for (s = STBI__ZFAST_BITS + 1;; ++s)
        if (k < z->maxcode[s])
            break;
    if (s >= 16)
        return -1; // 无效的代码！
    // 代码大小为 s，所以：
    b = (k >> (16 - s)) - z->firstcode[s] + z->firstsymbol[s];
    if (b >= STBI__ZNSYMS)
        return -1; // 某些数据在某个地方损坏了！
    if (z->size[b] != s)
        return -1; // 最初是一个断言，但现在报告失败。
}
    # 将a对象的code_buffer右移s位
    a->code_buffer >>= s;
    # 将a对象的num_bits减去s
    a->num_bits -= s;
    # 返回z对象的value列表中索引为b的元素
    return z->value[b];
# 解码哈夫曼编码，将压缩数据解压缩成原始数据
stbi_inline static int stbi__zhuffman_decode(stbi__zbuf * a, stbi__zhuffman * z) {
    int b, s;
    # 如果当前比特数小于16，则需要填充比特流
    if (a->num_bits < 16) {
        # 如果已经到达数据流末尾，则报告意外的数据结束错误
        if (stbi__zeof(a)) {
            return -1; /* report error for unexpected end of data. */
        }
        stbi__fill_bits(a);
    }
    # 从快速查找表中获取编码对应的值
    b = z->fast[a->code_buffer & STBI__ZFAST_MASK];
    if (b) {
        s = b >> 9;
        a->code_buffer >>= s;
        a->num_bits -= s;
        return b & 511;
    }
    # 如果快速查找表中没有对应的值，则使用慢速查找方法解码
    return stbi__zhuffman_decode_slowpath(a, z);
}

# 扩展输出缓冲区，为n个字节腾出空间
static int stbi__zexpand(stbi__zbuf * z, char * zout, int n) // need to make room for n bytes
{
    char * q;
    unsigned int cur, limit, old_limit;
    z->zout = zout;
    # 如果输出缓冲区不可扩展，则返回PNG文件损坏的错误
    if (!z->z_expandable)
        return stbi__err("output buffer limit", "Corrupt PNG");
    cur = (unsigned int)(z->zout - z->zout_start);
    limit = old_limit = (unsigned)(z->zout_end - z->zout_start);
    # 如果当前位置cur加上n超过了限制，则报告内存不足的错误
    if (UINT_MAX - cur < (unsigned)n)
        return stbi__err("outofmem", "Out of memory");
    # 当当前位置cur加上n超过了限制时，不断扩大限制直到能够容纳n个字节
    while (cur + n > limit) {
        if (limit > UINT_MAX / 2)
            return stbi__err("outofmem", "Out of memory");
        limit *= 2;
    }
    # 重新分配扩展后的输出缓冲区
    q = (char *)STBI_REALLOC_SIZED(z->zout_start, old_limit, limit);
    STBI_NOTUSED(old_limit);
    if (q == NULL)
        return stbi__err("outofmem", "Out of memory");
    z->zout_start = q;
    z->zout = q + cur;
    z->zout_end = q + limit;
    return 1;
}

# 哈夫曼编码长度的基础值表
static const int stbi__zlength_base[31] = {3,  4,  5,  6,  7,  8,  9,  10,  11,  13,  15,  17,  19,  23, 27, 31,
                                           35, 43, 51, 59, 67, 83, 99, 115, 131, 163, 195, 227, 258, 0,  0};

# 哈夫曼编码长度的额外值表
static const int stbi__zlength_extra[31] = {0, 0, 0, 0, 0, 0, 0, 0, 1, 1, 1, 1, 2, 2, 2, 2,
                                            3, 3, 3, 3, 4, 4, 4, 4, 5, 5, 5, 5, 0, 0, 0};
# 定义长度码的基础值，用于计算长度码
static const int stbi__zdist_base[32] = {1,    2,    3,    4,    5,    7,     9,     13,    17,  25,   33,
                                         49,   65,   97,   129,  193,  257,   385,   513,   769, 1025, 1537,
                                         2049, 3073, 4097, 6145, 8193, 12289, 16385, 24577, 0,   0};

# 定义长度码的额外位数，用于计算长度码
static const int stbi__zdist_extra[32] = {0, 0, 0, 0, 1, 1, 2, 2,  3,  3,  4,  4,  5,  5,  6,
                                          6, 7, 7, 8, 8, 9, 9, 10, 10, 11, 11, 12, 12, 13, 13};

# 解析哈夫曼块的函数
static int stbi__parse_huffman_block(stbi__zbuf * a) {
    char * zout = a->zout;
    }
}

# 计算哈夫曼编码的函数
static int stbi__compute_huffman_codes(stbi__zbuf * a) {
    # 定义长度码的反扭曲数组
    static const stbi_uc length_dezigzag[19] = {16, 17, 18, 0, 8, 7, 9, 6, 10, 5, 11, 4, 12, 3, 13, 2, 14, 1, 15};
    # 定义哈夫曼编码结构
    stbi__zhuffman z_codelength;
    # 定义长度码数组
    stbi_uc lencodes[286 + 32 + 137]; # 为最大单操作填充
    # 定义长度码大小数组
    stbi_uc codelength_sizes[19];
    int i, n;

    # 读取长度码的数量
    int hlit = stbi__zreceive(a, 5) + 257;
    # 读取距离码的数量
    int hdist = stbi__zreceive(a, 5) + 1;
    # 读取长度码长度的数量
    int hclen = stbi__zreceive(a, 4) + 4;
    # 计算总数量
    int ntot = hlit + hdist;

    # 初始化长度码大小数组
    memset(codelength_sizes, 0, sizeof(codelength_sizes));
    # 读取长度码长度
    for (i = 0; i < hclen; ++i) {
        int s = stbi__zreceive(a, 3);
        codelength_sizes[length_dezigzag[i]] = (stbi_uc)s;
    }
    # 构建哈夫曼编码
    if (!stbi__zbuild_huffman(&z_codelength, codelength_sizes, 19))
        return 0;

    n = 0;
    # 当 n 小于 ntot 时执行循环
    while (n < ntot) {
        # 从哈夫曼树 a 中解码出一个值，存入 c 中
        int c = stbi__zhuffman_decode(a, &z_codelength);
        # 如果 c 小于 0 或者大于等于 19，则返回错误信息
        if (c < 0 || c >= 19)
            return stbi__err("bad codelengths", "Corrupt PNG");
        # 如果 c 小于 16，则将 c 存入 lencodes 数组中，然后 n 自增
        if (c < 16)
            lencodes[n++] = (stbi_uc)c;
        # 如果 c 大于等于 16，则根据不同的情况进行处理
        else {
            # 声明并初始化 fill 变量
            stbi_uc fill = 0;
            # 如果 c 等于 16，则从哈夫曼树 a 中接收 2 位，并加上 3，存入 c 中
            if (c == 16) {
                c = stbi__zreceive(a, 2) + 3;
                # 如果 n 等于 0，则返回错误信息
                if (n == 0)
                    return stbi__err("bad codelengths", "Corrupt PNG");
                # 将 lencodes[n-1] 的值存入 fill 中
                fill = lencodes[n - 1];
            } 
            # 如果 c 等于 17，则从哈夫曼树 a 中接收 3 位，并加上 3，存入 c 中
            else if (c == 17) {
                c = stbi__zreceive(a, 3) + 3;
            } 
            # 如果 c 等于 18，则从哈夫曼树 a 中接收 7 位，并加上 11，存入 c 中
            else if (c == 18) {
                c = stbi__zreceive(a, 7) + 11;
            } 
            # 如果 c 不是 16、17、18 中的任意一个值，则返回错误信息
            else {
                return stbi__err("bad codelengths", "Corrupt PNG");
            }
            # 如果 ntot - n 小于 c，则返回错误信息
            if (ntot - n < c)
                return stbi__err("bad codelengths", "Corrupt PNG");
            # 将 fill 的值填充到 lencodes 数组中的 n 位置开始的 c 个元素中
            memset(lencodes + n, fill, c);
            # n 自增 c
            n += c;
        }
    }
    # 如果 n 不等于 ntot，则返回错误信息
    if (n != ntot)
        return stbi__err("bad codelengths", "Corrupt PNG");
    # 构建哈夫曼树 a->z_length，使用 lencodes 数组中的前 hlit 个元素
    if (!stbi__zbuild_huffman(&a->z_length, lencodes, hlit))
        return 0;
    # 构建哈夫曼树 a->z_distance，使用 lencodes 数组中的第 hlit 个元素开始的 hdist 个元素
    if (!stbi__zbuild_huffman(&a->z_distance, lencodes + hlit, hdist))
        return 0;
    # 返回 1，表示执行成功
    return 1;
}

static int stbi__parse_uncompressed_block(stbi__zbuf * a) {
    stbi_uc header[4]; // 创建一个长度为4的无符号字符数组header
    int len, nlen, k; // 定义整型变量len, nlen, k
    if (a->num_bits & 7)
        stbi__zreceive(a, a->num_bits & 7); // 如果a->num_bits与7按位与的结果不为0，则调用stbi__zreceive函数，丢弃结果
    // 将位压缩数据填充到header中
    k = 0; // 初始化k为0
    while (a->num_bits > 0) { // 当a->num_bits大于0时执行循环
        header[k++] = (stbi_uc)(a->code_buffer & 255); // 将a->code_buffer的低8位存入header数组中
        a->code_buffer >>= 8; // a->code_buffer右移8位
        a->num_bits -= 8; // a->num_bits减8
    }
    if (a->num_bits < 0) // 如果a->num_bits小于0
        return stbi__err("zlib corrupt", "Corrupt PNG"); // 返回zlib corrupt错误
    // 现在正常填充header
    while (k < 4) // 当k小于4时执行循环
        header[k++] = stbi__zget8(a); // 将stbi__zget8(a)的结果存入header数组中
    len = header[1] * 256 + header[0]; // 计算len的值
    nlen = header[3] * 256 + header[2]; // 计算nlen的值
    if (nlen != (len ^ 0xffff)) // 如果nlen不等于(len异或0xffff)
        return stbi__err("zlib corrupt", "Corrupt PNG"); // 返回zlib corrupt错误
    if (a->zbuffer + len > a->zbuffer_end) // 如果a->zbuffer加上len大于a->zbuffer_end
        return stbi__err("read past buffer", "Corrupt PNG"); // 返回read past buffer错误
    if (a->zout + len > a->zout_end) // 如果a->zout加上len大于a->zout_end
        if (!stbi__zexpand(a, a->zout, len)) // 如果stbi__zexpand函数返回false
            return 0; // 返回0
    memcpy(a->zout, a->zbuffer, len); // 将a->zbuffer中的len个字节复制到a->zout中
    a->zbuffer += len; // a->zbuffer增加len
    a->zout += len; // a->zout增加len
    return 1; // 返回1
}

static int stbi__parse_zlib_header(stbi__zbuf * a) {
    int cmf = stbi__zget8(a); // 从输入流中读取一个字节，存入cmf
    int cm = cmf & 15; // cm为cmf的低4位
    /* int cinfo = cmf >> 4; */
    int flg = stbi__zget8(a); // 从输入流中读取一个字节，存入flg
    if (stbi__zeof(a)) // 如果输入流已经结束
        return stbi__err("bad zlib header", "Corrupt PNG"); // 返回bad zlib header错误
    if ((cmf * 256 + flg) % 31 != 0) // 如果(cm * 256 + flg)对31取模不等于0
        return stbi__err("bad zlib header", "Corrupt PNG"); // 返回bad zlib header错误
    if (flg & 32) // 如果flg的第6位为1
        return stbi__err("no preset dict", "Corrupt PNG"); // 返回no preset dict错误
    if (cm != 8) // 如果cm不等于8
        return stbi__err("bad compression", "Corrupt PNG"); // 返回bad compression错误
    // window = 1 << (8 + cinfo)... but who cares, we fully buffer output
    return 1; // 返回1
}

static const stbi_uc stbi__zdefault_length[STBI__ZNSYMS] = {
    8, 8, 8, 8, 8, 8, 8, 8, 8, 8, 8, 8, 8, 8, 8, 8, 8, 8, 8, 8, 8, 8, 8, 8, 8, 8, 8, 8, 8, 8, 8, 8, 8, 8, 8, 8, 8,
    # 定义一个包含大量数字的数组
    # 这个数组可能是某种数据的表示，但缺乏上下文无法确定具体含义
// 定义固定距离码长度的数组
static const stbi_uc stbi__zdefault_distance[32] = {5, 5, 5, 5, 5, 5, 5, 5, 5, 5, 5, 5, 5, 5, 5, 5,
                                                    5, 5, 5, 5, 5, 5, 5, 5, 5, 5, 5, 5, 5, 5, 5, 5};
/*
Init algorithm:
{
   int i;   // use <= to match clearly with spec
   // 初始化长度码
   for (i=0; i <= 143; ++i)     stbi__zdefault_length[i]   = 8;
   for (   ; i <= 255; ++i)     stbi__zdefault_length[i]   = 9;
   for (   ; i <= 279; ++i)     stbi__zdefault_length[i]   = 7;
   for (   ; i <= 287; ++i)     stbi__zdefault_length[i]   = 8;

   // 初始化距离码
   for (i=0; i <=  31; ++i)     stbi__zdefault_distance[i] = 5;
}
*/

// 解析 zlib 数据
static int stbi__parse_zlib(stbi__zbuf * a, int parse_header) {
    int final, type;
    // 如果需要解析头部
    if (parse_header)
        if (!stbi__parse_zlib_header(a))
            return 0;
    a->num_bits = 0;
    a->code_buffer = 0;
    do {
        final = stbi__zreceive(a, 1);
        type = stbi__zreceive(a, 2);
        if (type == 0) {
            if (!stbi__parse_uncompressed_block(a))
                return 0;
        } else if (type == 3) {
            return 0;
        } else {
            if (type == 1) {
                // 使用固定码长度
                if (!stbi__zbuild_huffman(&a->z_length, stbi__zdefault_length, STBI__ZNSYMS))
                    return 0;
                if (!stbi__zbuild_huffman(&a->z_distance, stbi__zdefault_distance, 32))
                    return 0;
            } else {
                if (!stbi__compute_huffman_codes(a))
                    return 0;
            }
            if (!stbi__parse_huffman_block(a))
                return 0;
        }
    } while (!final);
    return 1;
}

// 执行 zlib 解压缩
static int stbi__do_zlib(stbi__zbuf * a, char * obuf, int olen, int exp, int parse_header) {
    a->zout_start = obuf;
    a->zout = obuf;
    a->zout_end = obuf + olen;
    a->z_expandable = exp;

    return stbi__parse_zlib(a, parse_header);
}

// 猜测 zlib 解码后的数据大小并分配内存
STBIDEF char * stbi_zlib_decode_malloc_guesssize(const char * buffer, int len, int initial_size, int * outlen) {
    # 创建一个 stbi__zbuf 结构体变量 a
    stbi__zbuf a;
    # 分配一块大小为 initial_size 的内存，并将其转换为 char* 类型的指针 p
    char * p = (char *)stbi__malloc(initial_size);
    # 如果内存分配失败，则返回空指针
    if (p == NULL)
        return NULL;
    # 将 buffer 转换为 stbi_uc* 类型，并赋值给 a.zbuffer
    a.zbuffer = (stbi_uc *)buffer;
    # 将 buffer 的末尾地址赋值给 a.zbuffer_end
    a.zbuffer_end = (stbi_uc *)buffer + len;
    # 调用 stbi__do_zlib 函数进行 zlib 解压缩
    if (stbi__do_zlib(&a, p, initial_size, 1, 1)) {
        # 如果 outlen 不为空，则将解压后的数据长度赋值给 outlen
        if (outlen)
            *outlen = (int)(a.zout - a.zout_start);
        # 返回解压后的数据起始地址
        return a.zout_start;
    } else {
        # 释放解压缩过程中分配的内存
        STBI_FREE(a.zout_start);
        # 返回空指针
        return NULL;
    }
# 用于解码经过 zlib 压缩的数据并分配内存
STBIDEF char * stbi_zlib_decode_malloc(char const * buffer, int len, int * outlen) {
    return stbi_zlib_decode_malloc_guesssize(buffer, len, 16384, outlen);
}

# 用于根据给定的大小和标志位解码经过 zlib 压缩的数据并分配内存
STBIDEF char * stbi_zlib_decode_malloc_guesssize_headerflag(const char * buffer, int len, int initial_size, int * outlen,
                                                            int parse_header) {
    # 创建一个缓冲区对象
    stbi__zbuf a;
    # 分配初始大小的内存
    char * p = (char *)stbi__malloc(initial_size);
    if (p == NULL)
        return NULL;
    # 设置缓冲区对象的属性
    a.zbuffer = (stbi_uc *)buffer;
    a.zbuffer_end = (stbi_uc *)buffer + len;
    # 调用 zlib 解压函数进行解压
    if (stbi__do_zlib(&a, p, initial_size, 1, parse_header)) {
        if (outlen)
            *outlen = (int)(a.zout - a.zout_start);
        return a.zout_start;
    } else {
        STBI_FREE(a.zout_start);
        return NULL;
    }
}

# 用于解码经过 zlib 压缩的数据到指定的缓冲区
STBIDEF int stbi_zlib_decode_buffer(char * obuffer, int olen, char const * ibuffer, int ilen) {
    # 创建一个缓冲区对象
    stbi__zbuf a;
    # 设置缓冲区对象的属性
    a.zbuffer = (stbi_uc *)ibuffer;
    a.zbuffer_end = (stbi_uc *)ibuffer + ilen;
    # 调用 zlib 解压函数进行解压
    if (stbi__do_zlib(&a, obuffer, olen, 0, 1))
        return (int)(a.zout - a.zout_start);
    else
        return -1;
}

# 用于解码经过 zlib 压缩的数据并分配内存，不包含头部信息
STBIDEF char * stbi_zlib_decode_noheader_malloc(char const * buffer, int len, int * outlen) {
    # 创建一个缓冲区对象
    stbi__zbuf a;
    # 分配初始大小的内存
    char * p = (char *)stbi__malloc(16384);
    if (p == NULL)
        return NULL;
    # 设置缓冲区对象的属性
    a.zbuffer = (stbi_uc *)buffer;
    a.zbuffer_end = (stbi_uc *)buffer + len;
    # 调用 zlib 解压函数进行解压
    if (stbi__do_zlib(&a, p, 16384, 1, 0)) {
        if (outlen)
            *outlen = (int)(a.zout - a.zout_start);
        return a.zout_start;
    } else {
        STBI_FREE(a.zout_start);
        return NULL;
    }
}

# 用于解码经过 zlib 压缩的数据到指定的缓冲区，不包含头部信息
STBIDEF int stbi_zlib_decode_noheader_buffer(char * obuffer, int olen, const char * ibuffer, int ilen) {
    # 创建一个缓冲区对象
    stbi__zbuf a;
    # 设置缓冲区对象的属性
    a.zbuffer = (stbi_uc *)ibuffer;
    a.zbuffer_end = (stbi_uc *)ibuffer + ilen;
    # 调用 zlib 解压函数进行解压
    if (stbi__do_zlib(&a, obuffer, olen, 0, 0))
        return (int)(a.zout - a.zout_start);
    else
        return -1;
}
#endif
# 定义 PNG 数据块头部结构
typedef struct {
    stbi__uint32 length;
    stbi__uint32 type;
} stbi__pngchunk;

# 读取 PNG 数据块头部信息
static stbi__pngchunk stbi__get_chunk_header(stbi__context * s) {
    stbi__pngchunk c;
    c.length = stbi__get32be(s);
    c.type = stbi__get32be(s);
    return c;
}

# 检查 PNG 文件头部信息
static int stbi__check_png_header(stbi__context * s) {
    static const stbi_uc png_sig[8] = {137, 80, 78, 71, 13, 10, 26, 10};
    int i;
    for (i = 0; i < 8; ++i)
        if (stbi__get8(s) != png_sig[i])
            return stbi__err("bad png sig", "Not a PNG");
    return 1;
}

# 定义 PNG 数据结构
typedef struct {
    stbi__context * s;
    stbi_uc *idata, *expanded, *out;
    int depth;
} stbi__png;

# 定义 PNG 滤波器类型
enum {
    STBI__F_none = 0,
    STBI__F_sub = 1,
    STBI__F_up = 2,
    STBI__F_avg = 3,
    STBI__F_paeth = 4,
    # 用于第一行扫描线的合成滤波器，避免需要一个全为0的虚拟行
    STBI__F_avg_first,
    STBI__F_paeth_first
};

# 第一行滤波器类型数组
static stbi_uc first_row_filter[5] = {STBI__F_none, STBI__F_sub, STBI__F_none, STBI__F_avg_first, STBI__F_paeth_first};

# 计算 Paeth 滤波器
static int stbi__paeth(int a, int b, int c) {
    int p = a + b - c;
    int pa = abs(p - a);
    int pb = abs(p - b);
    int pc = abs(p - c);
    if (pa <= pb and pa <= pc)
        return a;
    if (pb <= pc)
        return b;
    return c;
}

# 深度缩放表
static const stbi_uc stbi__depth_scale_table[9] = {0, 0xff, 0x55, 0, 0x11, 0, 0, 0, 0x01};

# 从解压后的数据创建 PNG 数据
# 创建 PNG 图像的原始数据
def stbi__create_png_image_raw(a, raw, raw_len, out_n, x, y, depth, color):
    # 计算每个像素的字节数
    bytes = (depth == 16 ? 2 : 1)
    # 获取上下文
    s = a->s
    # 计算图像行的字节数
    stride = x * out_n * bytes
    # 计算图像数据的长度和宽度字节数
    img_len, img_width_bytes = 0, 0
    # 初始化变量
    k, img_n, output_bytes, filter_bytes, width = 0, 0, 0, 0, 0

    # 检查输出通道数是否与图像通道数匹配
    STBI_ASSERT(out_n == s->img_n || out_n == s->img_n + 1)
    # 分配内存给输出图像数据
    a->out = (stbi_uc *)stbi__malloc_mad3(x, y, output_bytes, 0)
    # 检查内存分配是否成功
    if (!a->out)
        return stbi__err("outofmem", "Out of memory")

    # 检查图像数据长度是否合法
    if (!stbi__mad3sizes_valid(img_n, x, depth, 7))
        return stbi__err("too large", "Corrupt PNG")
    # 计算图像宽度的字节数
    img_width_bytes = (((img_n * x * depth) + 7) >> 3)
    # 计算图像数据的总长度
    img_len = (img_width_bytes + 1) * y

    # 检查原始数据长度是否足够
    if (raw_len < img_len)
        return stbi__err("not enough pixels", "Corrupt PNG")

    # 定义宏
#define STBI__CASE(f)                                                                                                          \
    case f:                                                                                                                    \
        for (k = 0; k < nk; ++k)
            switch (filter) {
            // "none" filter turns into a memcpy here; make that explicit.
            // 如果过滤器是"none"，这里会变成一个memcpy操作；这里明确说明一下
            case STBI__F_none:
                // 使用memcpy函数将raw中的数据拷贝到cur中，拷贝长度为nk
                memcpy(cur, raw, nk);
                // 跳出switch语句
                break;
                // 如果过滤器是"sub"，执行以下操作
                STBI__CASE(STBI__F_sub) { cur[k] = STBI__BYTECAST(raw[k] + cur[k - filter_bytes]); }
                // 跳出switch语句
                break;
                // 如果过滤器是"up"，执行以下操作
                STBI__CASE(STBI__F_up) { cur[k] = STBI__BYTECAST(raw[k] + prior[k]); }
                // 跳出switch语句
                break;
                // 如果过滤器是"avg"，执行以下操作
                STBI__CASE(STBI__F_avg) { cur[k] = STBI__BYTECAST(raw[k] + ((prior[k] + cur[k - filter_bytes]) >> 1)); }
                // 跳出switch语句
                break;
                // 如果过滤器是"paeth"，执行以下操作
                STBI__CASE(STBI__F_paeth) {
                    cur[k] = STBI__BYTECAST(raw[k] + stbi__paeth(cur[k - filter_bytes], prior[k], prior[k - filter_bytes]));
                }
                // 跳出switch语句
                break;
                // 如果过滤器是"avg_first"，执行以下操作
                STBI__CASE(STBI__F_avg_first) { cur[k] = STBI__BYTECAST(raw[k] + (cur[k - filter_bytes] >> 1)); }
                // 跳出switch语句
                break;
                // 如果过滤器是"paeth_first"，执行以下操作
                STBI__CASE(STBI__F_paeth_first) { cur[k] = STBI__BYTECAST(raw[k] + stbi__paeth(cur[k - filter_bytes], 0, 0)); }
                // 跳出switch语句
                break;
            }
# 取消之前定义的 STBI__CASE 宏
#undef STBI__CASE
# 将 raw 的值增加 nk
raw += nk;
# 如果条件成立，执行以下代码块
} else {
    # 断言 img_n + 1 等于 out_n
    STBI_ASSERT(img_n + 1 == out_n);
    # 定义 STBI__CASE 宏，用于处理不同的滤波器类型
    # 根据不同的滤波器类型，执行相应的操作
    # 没有滤波器
    STBI__CASE(STBI__F_none) { cur[k] = raw[k]; }
    # 差值滤波
    STBI__CASE(STBI__F_sub) { cur[k] = STBI__BYTECAST(raw[k] + cur[k - output_bytes]); }
    # 上方滤波
    STBI__CASE(STBI__F_up) { cur[k] = STBI__BYTECAST(raw[k] + prior[k]); }
    # 平均值滤波
    STBI__CASE(STBI__F_avg) { cur[k] = STBI__BYTECAST(raw[k] + ((prior[k] + cur[k - output_bytes]) >> 1)); }
    # Paeth 滤波
    STBI__CASE(STBI__F_paeth) { cur[k] = STBI__BYTECAST(raw[k] + stbi__paeth(cur[k - output_bytes], prior[k], prior[k - output_bytes])); }
    # 平均值滤波（首行）
    STBI__CASE(STBI__F_avg_first) { cur[k] = STBI__BYTECAST(raw[k] + (cur[k - output_bytes] >> 1)); }
    # Paeth 滤波（首行）
    STBI__CASE(STBI__F_paeth_first) { cur[k] = STBI__BYTECAST(raw[k] + stbi__paeth(cur[k - output_bytes], 0, 0)); }
    # 取消之前定义的 STBI__CASE 宏
    # the loop above sets the high byte of the pixels' alpha, but for
    # 16 bit png files we also need the low byte set. we'll do that here.
    if (depth == 16) {
        # 重新设置 cur 的值
        cur = a->out + stride * j; // start at the beginning of the row again
        # 遍历像素，设置低字节为 255
        for (i = 0; i < x; ++i, cur += output_bytes) {
            cur[filter_bytes + 1] = 255;
        }
    }
}
    // 如果深度为16，则进行位到像素的扩展
    // 为了性能考虑，这个过程可以在上面的代码运行两个扫描线之后进行，这样就不会干扰滤波，但仍然会在缓存中
    } else if (depth == 16) {
        // 强制将图像数据从大端序转换为平台本地序
        // 这是在一个单独的步骤中完成的，因为解码依赖于数据不被修改，但如果小心处理，可能可以在解码过程中每行进行
        stbi_uc * cur = a->out;
        stbi__uint16 * cur16 = (stbi__uint16 *)cur;

        for (i = 0; i < x * y * out_n; ++i, cur16++, cur += 2) {
            *cur16 = (cur[0] << 8) | cur[1];
        }
    }

    // 返回1表示解码成功
    return 1;
# 创建 PNG 图像
def stbi__create_png_image(stbi__png * a, stbi_uc * image_data, stbi__uint32 image_data_len, int out_n, int depth,
                                  int color, int interlaced):
    # 计算每个像素的字节数
    int bytes = (depth == 16 ? 2 : 1)
    # 计算输出图像的总字节数
    int out_bytes = out_n * bytes
    # 定义最终图像数据的指针
    stbi_uc * final
    # 定义循环变量 p
    int p
    # 如果不是隔行扫描，则调用 stbi__create_png_image_raw 函数创建 PNG 图像
    if (!interlaced)
        return stbi__create_png_image_raw(a, image_data, image_data_len, out_n, a->s->img_x, a->s->img_y, depth, color)

    # 对图像进行去隔行扫描
    final = (stbi_uc *)stbi__malloc_mad3(a->s->img_x, a->s->img_y, out_bytes, 0)
    # 如果内存分配失败，则返回错误
    if (!final)
        return stbi__err("outofmem", "Out of memory")
    # 遍历七个隔行扫描的 pass
    for (p = 0; p < 7; ++p):
        # 定义隔行扫描的原点和间隔
        int xorig[] = {0, 4, 0, 2, 0, 1, 0}
        int yorig[] = {0, 0, 4, 0, 2, 0, 1}
        int xspc[] = {8, 8, 4, 4, 2, 2, 1}
        int yspc[] = {8, 8, 8, 4, 4, 2, 2}
        int i, j, x, y
        # 计算当前 pass 的图像宽度和高度
        x = (a->s->img_x - xorig[p] + xspc[p] - 1) / xspc[p]
        y = (a->s->img_y - yorig[p] + yspc[p] - 1) / yspc[p]
        # 如果宽度和高度大于 0，则创建 PNG 图像
        if (x && y):
            stbi__uint32 img_len = ((((a->s->img_n * x * depth) + 7) >> 3) + 1) * y
            # 如果创建 PNG 图像失败，则释放内存并返回错误
            if (!stbi__create_png_image_raw(a, image_data, image_data_len, out_n, x, y, depth, color)):
                STBI_FREE(final)
                return 0
            # 将创建的 PNG 图像数据复制到最终图像数据中
            for (j = 0; j < y; ++j):
                for (i = 0; i < x; ++i):
                    int out_y = j * yspc[p] + yorig[p]
                    int out_x = i * xspc[p] + xorig[p]
                    memcpy(final + out_y * a->s->img_x * out_bytes + out_x * out_bytes, a->out + (j * x + i) * out_bytes,
                           out_bytes)
            # 释放创建的 PNG 图像数据
            STBI_FREE(a->out)
            # 更新剩余的图像数据和长度
            image_data += img_len
            image_data_len -= img_len
    # 将最终图像数据赋值给 a->out
    a->out = final

    return 1

# 计算透明度
def stbi__compute_transparency(stbi__png * z, stbi_uc tc[3], int out_n):
    # 获取 PNG 对象的上下文
    stbi__context * s = z->s
    # 定义变量 i，计算像素总数为图像宽度乘以图像高度
    stbi__uint32 i, pixel_count = s->img_x * s->img_y;
    # 定义指针 p，指向解压后的图像数据
    stbi_uc * p = z->out;

    # 断言输出通道数为 2 或 4
    STBI_ASSERT(out_n == 2 || out_n == 4);

    # 如果输出通道数为 2
    if (out_n == 2) {
        # 遍历每个像素
        for (i = 0; i < pixel_count; ++i) {
            # 如果当前像素的第一个通道值等于指定颜色值的第一个通道值，则将第二个通道值设为 0，否则设为 255
            p[1] = (p[0] == tc[0] ? 0 : 255);
            # 指针移动到下一个像素
            p += 2;
        }
    } 
    # 如果输出通道数为 4
    else {
        # 遍历每个像素
        for (i = 0; i < pixel_count; ++i) {
            # 如果当前像素的 RGB 值等于指定颜色值的 RGB 值，则将 alpha 通道值设为 0
            if (p[0] == tc[0] && p[1] == tc[1] && p[2] == tc[2])
                p[3] = 0;
            # 指针移动到下一个像素
            p += 4;
        }
    }
    # 返回 1，表示处理成功
    return 1;
# 计算16位PNG图像的透明度
def stbi__compute_transparency16(stbi__png * z, stbi__uint16 tc[3], int out_n):
    # 获取上下文
    stbi__context * s = z->s
    # 计算像素数量
    stbi__uint32 i, pixel_count = s->img_x * s->img_y
    # 获取输出指针
    stbi__uint16 * p = (stbi__uint16 *)z->out

    # 计算基于颜色的透明度，假设输出中已经有65535作为alpha值
    STBI_ASSERT(out_n == 2 || out_n == 4)

    if (out_n == 2):
        for (i = 0; i < pixel_count; ++i):
            p[1] = (p[0] == tc[0] ? 0 : 65535)
            p += 2
    else:
        for (i = 0; i < pixel_count; ++i):
            if (p[0] == tc[0] and p[1] == tc[1] and p[2] == tc[2]):
                p[3] = 0
            p += 4
    return 1

# 扩展PNG调色板
def stbi__expand_png_palette(stbi__png * a, stbi_uc * palette, int len, int pal_img_n):
    # 计算像素数量
    stbi__uint32 i, pixel_count = a->s->img_x * a->s->img_y
    # 获取指针
    stbi_uc *p, *temp_out, *orig = a->out

    # 分配内存
    p = (stbi_uc *)stbi__malloc_mad2(pixel_count, pal_img_n, 0)
    if (p == NULL):
        return stbi__err("outofmem", "Out of memory")

    # 在这里和free(out)之间，退出会导致内存泄漏
    temp_out = p

    if (pal_img_n == 3):
        for (i = 0; i < pixel_count; ++i):
            n = orig[i] * 4
            p[0] = palette[n]
            p[1] = palette[n + 1]
            p[2] = palette[n + 2]
            p += 3
    else:
        for (i = 0; i < pixel_count; ++i):
            n = orig[i] * 4
            p[0] = palette[n]
            p[1] = palette[n + 1]
            p[2] = palette[n + 2]
            p[3] = palette[n + 3]
            p += 4
    # 释放原输出内存
    STBI_FREE(a->out)
    # 更新输出指针
    a->out = temp_out

    STBI_NOTUSED(len)

    return 1

# 全局变量，用于设置加载时是否取消预乘
static int stbi__unpremultiply_on_load_global = 0
static int stbi__de_iphone_flag_global = 0

# 设置加载时是否取消预乘
STBIDEF void stbi_set_unpremultiply_on_load(int flag_true_if_should_unpremultiply):
    stbi__unpremultiply_on_load_global = flag_true_if_should_unpremultiply
# 设置是否需要将 iPhone PNG 转换为 RGB 格式
STBIDEF void stbi_convert_iphone_png_to_rgb(int flag_true_if_should_convert) {
    stbi__de_iphone_flag_global = flag_true_if_should_convert;
}

# 如果没有定义 STBI_THREAD_LOCAL，则使用全局变量，否则使用线程本地变量
#ifndef STBI_THREAD_LOCAL
# 将全局变量赋值给宏定义
#define stbi__unpremultiply_on_load stbi__unpremultiply_on_load_global
#define stbi__de_iphone_flag stbi__de_iphone_flag_global
#else
# 定义线程本地变量和设置标志位
static STBI_THREAD_LOCAL int stbi__unpremultiply_on_load_local, stbi__unpremultiply_on_load_set;
static STBI_THREAD_LOCAL int stbi__de_iphone_flag_local, stbi__de_iphone_flag_set;

# 设置是否需要在加载时取消预乘线程
STBIDEF void stbi_set_unpremultiply_on_load_thread(int flag_true_if_should_unpremultiply) {
    stbi__unpremultiply_on_load_local = flag_true_if_should_unpremultiply;
    stbi__unpremultiply_on_load_set = 1;
}

# 设置是否需要将 iPhone PNG 转换为 RGB 格式的线程
STBIDEF void stbi_convert_iphone_png_to_rgb_thread(int flag_true_if_should_convert) {
    stbi__de_iphone_flag_local = flag_true_if_should_convert;
    stbi__de_iphone_flag_set = 1;
}

# 定义宏，根据设置的标志位选择使用线程本地变量还是全局变量
#define stbi__unpremultiply_on_load                                                                                            \
    (stbi__unpremultiply_on_load_set ? stbi__unpremultiply_on_load_local : stbi__unpremultiply_on_load_global)
#define stbi__de_iphone_flag (stbi__de_iphone_flag_set ? stbi__de_iphone_flag_local : stbi__de_iphone_flag_global)
#endif // STBI_THREAD_LOCAL

# iPhone PNG 转换为 RGB 格式的具体实现
static void stbi__de_iphone(stbi__png * z) {
    stbi__context * s = z->s;
    stbi__uint32 i, pixel_count = s->img_x * s->img_y;
    stbi_uc * p = z->out;

    # 如果输出通道数为 3，则将 bgr 转换为 rgb
    if (s->img_out_n == 3) { // convert bgr to rgb
        for (i = 0; i < pixel_count; ++i) {
            stbi_uc t = p[0];
            p[0] = p[2];
            p[2] = t;
            p += 3;
        }
    } else {
        // 断言输出通道数为4
        STBI_ASSERT(s->img_out_n == 4);
        if (stbi__unpremultiply_on_load) {
            // 将bgr转换为rgb并取消预乘
            for (i = 0; i < pixel_count; ++i) {
                // 获取alpha通道值
                stbi_uc a = p[3];
                // 临时保存蓝色通道值
                stbi_uc t = p[0];
                if (a) {
                    // 计算非预乘后的rgb值
                    stbi_uc half = a / 2;
                    p[0] = (p[2] * 255 + half) / a;
                    p[1] = (p[1] * 255 + half) / a;
                    p[2] = (t * 255 + half) / a;
                } else {
                    // alpha通道为0时，直接交换蓝色和红色通道值
                    p[0] = p[2];
                    p[2] = t;
                }
                p += 4;
            }
        } else {
            // 将bgr转换为rgb
            for (i = 0; i < pixel_count; ++i) {
                // 临时保存蓝色通道值
                stbi_uc t = p[0];
                // 交换蓝色和红色通道值
                p[0] = p[2];
                p[2] = t;
                p += 4;
            }
        }
    }
}

这是一个代码块的结束。


#define STBI__PNG_TYPE(a, b, c, d) (((unsigned)(a) << 24) + ((unsigned)(b) << 16) + ((unsigned)(c) << 8) + (unsigned)(d))

定义了一个宏，用于将四个字节转换为一个无符号整数。


static int stbi__parse_png_file(stbi__png * z, int scan, int req_comp) {

定义了一个静态函数，用于解析 PNG 文件。


    stbi_uc palette[1024], pal_img_n = 0;
    stbi_uc has_trans = 0, tc[3] = {0};
    stbi__uint16 tc16[3];
    stbi__uint32 ioff = 0, idata_limit = 0, i, pal_len = 0;
    int first = 1, k, interlace = 0, color = 0, is_iphone = 0;
    stbi__context * s = z->s;

声明了一系列变量，用于存储解析 PNG 文件时需要用到的数据。


    z->expanded = NULL;
    z->idata = NULL;
    z->out = NULL;

将指针初始化为 NULL。


    if (!stbi__check_png_header(s))
        return 0;

检查 PNG 文件头部是否正确，如果不正确则返回 0。


    if (scan == STBI__SCAN_type)
        return 1;

如果扫描类型为 STBI__SCAN_type，则返回 1。


#ifndef STBI_NO_FAILURE_STRINGS
                // not threadsafe
                static char invalid_chunk[] = "XXXX PNG chunk not known";
                invalid_chunk[0] = STBI__BYTECAST(c.type >> 24);
                invalid_chunk[1] = STBI__BYTECAST(c.type >> 16);
                invalid_chunk[2] = STBI__BYTECAST(c.type >> 8);
                invalid_chunk[3] = STBI__BYTECAST(c.type >> 0);
#endif

如果未定义 STBI_NO_FAILURE_STRINGS，则定义一个错误信息字符串。


                return stbi__err(invalid_chunk, "PNG not supported: unknown PNG chunk type");
            }
            stbi__skip(s, c.length);
            break;
        }
        // end of PNG chunk, read and skip CRC
        stbi__get32be(s);
    }
}

处理未知的 PNG 块类型，跳过该块并读取 CRC。


static void * stbi__do_png(stbi__png * p, int * x, int * y, int * n, int req_comp, stbi__result_info * ri) {

定义了一个静态函数，用于处理 PNG 文件。


    void * result = NULL;
    if (req_comp < 0 || req_comp > 4)
        return stbi__errpuc("bad req_comp", "Internal error");

如果请求的通道数小于 0 或大于 4，则返回错误信息。
    # 如果成功解析 PNG 文件
    if (stbi__parse_png_file(p, STBI__SCAN_load, req_comp)) {
        # 如果深度小于等于8
        if (p->depth <= 8)
            # 设置每通道的位数为8
            ri->bits_per_channel = 8;
        # 如果深度为16
        else if (p->depth == 16)
            # 设置每通道的位数为16
            ri->bits_per_channel = 16;
        else
            # 返回错误信息，表示不支持的颜色深度
            return stbi__errpuc("bad bits_per_channel", "PNG not supported: unsupported color depth");
        # 将解析结果赋值给result
        result = p->out;
        # 将p->out置为NULL
        p->out = NULL;
        # 如果请求的通道数不等于解析结果的通道数
        if (req_comp && req_comp != p->s->img_out_n) {
            # 如果每通道的位数为8
            if (ri->bits_per_channel == 8)
                # 转换格式为请求的通道数
                result = stbi__convert_format((unsigned char *)result, p->s->img_out_n, req_comp, p->s->img_x, p->s->img_y);
            else
                # 转换格式为请求的通道数
                result = stbi__convert_format16((stbi__uint16 *)result, p->s->img_out_n, req_comp, p->s->img_x, p->s->img_y);
            # 更新解析结果的通道数为请求的通道数
            p->s->img_out_n = req_comp;
            # 如果转换结果为NULL，则返回NULL
            if (result == NULL)
                return result;
        }
        # 更新x和y为解析结果的宽和高
        *x = p->s->img_x;
        *y = p->s->img_y;
        # 如果n不为空，则更新n为解析结果的通道数
        if (n)
            *n = p->s->img_n;
    }
    # 释放p->out的内存
    STBI_FREE(p->out);
    # 将p->out置为NULL
    p->out = NULL;
    # 释放p->expanded的内存
    STBI_FREE(p->expanded);
    # 将p->expanded置为NULL
    p->expanded = NULL;
    # 释放p->idata的内存
    STBI_FREE(p->idata);
    # 将p->idata置为NULL
    p->idata = NULL;
    # 返回解析结果
    return result;
}
// 用于加载 PNG 图像的函数，返回图像数据指针
static void * stbi__png_load(stbi__context * s, int * x, int * y, int * comp, int req_comp, stbi__result_info * ri) {
    stbi__png p; // 创建 PNG 对象
    p.s = s; // 将输入的上下文赋值给 PNG 对象
    return stbi__do_png(&p, x, y, comp, req_comp, ri); // 调用处理 PNG 图像的函数
}

// 用于测试是否为 PNG 图像的函数，返回测试结果
static int stbi__png_test(stbi__context * s) {
    int r; // 定义变量 r
    r = stbi__check_png_header(s); // 检查 PNG 头部信息
    stbi__rewind(s); // 重置输入上下文
    return r; // 返回测试结果
}

// 用于获取 PNG 图像信息的函数，返回获取结果
static int stbi__png_info_raw(stbi__png * p, int * x, int * y, int * comp) {
    if (!stbi__parse_png_file(p, STBI__SCAN_header, 0)) { // 解析 PNG 文件头部信息
        stbi__rewind(p->s); // 重置输入上下文
        return 0; // 返回获取信息失败
    }
    if (x)
        *x = p->s->img_x; // 获取图像宽度
    if (y)
        *y = p->s->img_y; // 获取图像高度
    if (comp)
        *comp = p->s->img_n; // 获取图像通道数
    return 1; // 返回获取信息成功
}

// 用于获取 PNG 图像信息的函数，返回获取结果
static int stbi__png_info(stbi__context * s, int * x, int * y, int * comp) {
    stbi__png p; // 创建 PNG 对象
    p.s = s; // 将输入的上下文赋值给 PNG 对象
    return stbi__png_info_raw(&p, x, y, comp); // 调用获取 PNG 图像信息的函数
}

// 用于判断 PNG 图像是否为 16 位的函数，返回判断结果
static int stbi__png_is16(stbi__context * s) {
    stbi__png p; // 创建 PNG 对象
    p.s = s; // 将输入的上下文赋值给 PNG 对象
    if (!stbi__png_info_raw(&p, NULL, NULL, NULL)) // 获取 PNG 图像信息
        return 0; // 返回判断结果为假
    if (p.depth != 16) { // 判断图像深度是否为 16
        stbi__rewind(p.s); // 重置输入上下文
        return 0; // 返回判断结果为假
    }
    return 1; // 返回判断结果为真
}
#endif

// Microsoft/Windows BMP image

#ifndef STBI_NO_BMP
// 用于测试是否为 BMP 图像的函数，返回测试结果
static int stbi__bmp_test_raw(stbi__context * s) {
    int r; // 定义变量 r
    int sz; // 定义变量 sz
    if (stbi__get8(s) != 'B') // 判断是否为 BMP 图像
        return 0; // 返回测试结果为假
    if (stbi__get8(s) != 'M') // 判断是否为 BMP 图像
        return 0; // 返回测试结果为假
    stbi__get32le(s); // 读取并丢弃文件大小
    stbi__get16le(s); // 读取并丢弃保留字段
    stbi__get16le(s); // 读取并丢弃保留字段
    stbi__get32le(s); // 读取并丢弃数据偏移
    sz = stbi__get32le(s); // 读取图像信息头大小
    r = (sz == 12 || sz == 40 || sz == 56 || sz == 108 || sz == 124); // 判断图像信息头大小是否符合 BMP 格式
    return r; // 返回测试结果
}

// 用于测试是否为 BMP 图像的函数，返回测试结果
static int stbi__bmp_test(stbi__context * s) {
    int r = stbi__bmp_test_raw(s); // 调用测试 BMP 图像的函数
    stbi__rewind(s); // 重置输入上下文
    return r; // 返回测试结果
}

// 返回 z 的最高位的索引值
static int stbi__high_bit(unsigned int z) {
    int n = 0; // 定义变量 n
    if (z == 0) // 判断 z 是否为 0
        return -1; // 返回 -1
    if (z >= 0x10000) { // 判断 z 是否大于等于 0x10000
        n += 16; // n 加 16
        z >>= 16; // z 右移 16 位
    }
    if (z >= 0x00100) { // 判断 z 是否大于等于 0x00100
        n += 8; // n 加 8
        z >>= 8; // z 右移 8 位
    }
    # 如果 z 大于等于 16，则将 n 增加 4，同时 z 右移 4 位
    if (z >= 0x00010) {
        n += 4;
        z >>= 4;
    }
    # 如果 z 大于等于 4，则将 n 增加 2，同时 z 右移 2 位
    if (z >= 0x00004) {
        n += 2;
        z >>= 2;
    }
    # 如果 z 大于等于 2，则将 n 增加 1
    if (z >= 0x00002) {
        n += 1; /* >>=  1;*/
    }
    # 返回 n 的值
    return n;
}

# 计算一个无符号整数的二进制表示中1的个数
static int stbi__bitcount(unsigned int a) {
    a = (a & 0x55555555) + ((a >> 1) & 0x55555555); // 最多2个1
    a = (a & 0x33333333) + ((a >> 2) & 0x33333333); // 最多4个1
    a = (a + (a >> 4)) & 0x0f0f0f0f;                // 每4位最多8个1，现在有8位
    a = (a + (a >> 8));                             // 每8位最多16个1
    a = (a + (a >> 16));                            // 每8位最多32个1
    return a & 0xff;
}

# 从v中提取任意对齐的N位值（N=bits），然后将其扩展为8位长并分数扩展到完整范围
static int stbi__shiftsigned(unsigned int v, int shift, int bits) {
    static unsigned int mul_table[9] = {
        0,
        0xff /*0b11111111*/,
        0x55 /*0b01010101*/,
        0x49 /*0b01001001*/,
        0x11 /*0b00010001*/,
        0x21 /*0b00100001*/,
        0x41 /*0b01000001*/,
        0x81 /*0b10000001*/,
        0x01 /*0b00000001*/,
    };
    static unsigned int shift_table[9] = {
        0, 0, 0, 1, 0, 2, 4, 6, 0,
    };
    if (shift < 0)
        v <<= -shift;
    else
        v >>= shift;
    STBI_ASSERT(v < 256);
    v >>= (8 - bits);
    STBI_ASSERT(bits >= 0 && bits <= 8);
    return (int)((unsigned)v * mul_table[bits]) >> shift_table[bits];
}

# 定义一个结构体 stbi__bmp_data
typedef struct {
    int bpp, offset, hsz;
    unsigned int mr, mg, mb, ma, all_a;
    int extra_read;
} stbi__bmp_data;

# 设置 stbi__bmp_data 结构体的默认值
static int stbi__bmp_set_mask_defaults(stbi__bmp_data * info, int compress) {
    # 如果压缩方式为3，则BI_BITFIELDS指定了掩码，不要覆盖
    if (compress == 3)
        return 1;
    # 如果不需要压缩
    if (compress == 0) {
        # 如果颜色深度为16位
        if (info->bpp == 16) {
            # 设置红色掩码为31位左移10位
            info->mr = 31u << 10;
            # 设置绿色掩码为31位左移5位
            info->mg = 31u << 5;
            # 设置蓝色掩码为31位左移0位
            info->mb = 31u << 0;
        } else if (info->bpp == 32) {
            # 设置红色掩码为0xff左移16位
            info->mr = 0xffu << 16;
            # 设置绿色掩码为0xff左移8位
            info->mg = 0xffu << 8;
            # 设置蓝色掩码为0xff左移0位
            info->mb = 0xffu << 0;
            # 设置alpha通道掩码为0xff左移24位
            info->ma = 0xffu << 24;
            # 如果all_a为0，则表示加载了alpha通道但全部为0
            info->all_a = 0; 
        } else {
            # 否则使用默认值，即全部为0
            info->mr = info->mg = info->mb = info->ma = 0;
        }
        # 返回1表示成功
        return 1;
    }
    # 返回0表示错误
    return 0; 
# 解析 BMP 文件头部信息
static void * stbi__bmp_parse_header(stbi__context * s, stbi__bmp_data * info) {
    int hsz;
    # 检查文件头部是否以 'BM' 开头，如果不是则返回错误信息
    if (stbi__get8(s) != 'B' || stbi__get8(s) != 'M')
        return stbi__errpuc("not BMP", "Corrupt BMP");
    # 丢弃文件大小信息
    stbi__get32le(s); // discard filesize
    # 丢弃保留字段信息
    stbi__get16le(s); // discard reserved
    stbi__get16le(s); // discard reserved
    # 读取偏移量信息
    info->offset = stbi__get32le(s);
    info->hsz = hsz = stbi__get32le(s);
    info->mr = info->mg = info->mb = info->ma = 0;
    info->extra_read = 14;

    # 检查偏移量是否合法
    if (info->offset < 0)
        return stbi__errpuc("bad BMP", "bad BMP");

    # 检查头部信息是否合法
    if (hsz != 12 && hsz != 40 && hsz != 56 && hsz != 108 && hsz != 124)
        return stbi__errpuc("unknown BMP", "BMP type not supported: unknown");
    # 根据头部信息读取图片宽高
    if (hsz == 12) {
        s->img_x = stbi__get16le(s);
        s->img_y = stbi__get16le(s);
    } else {
        s->img_x = stbi__get32le(s);
        s->img_y = stbi__get32le(s);
    }
    # 检查图片是否为单色位图
    if (stbi__get16le(s) != 1)
        return stbi__errpuc("bad BMP", "bad BMP");
    # 读取每像素位数
    info->bpp = stbi__get16le(s);
    }
    return (void *)1;
}

# 加载 BMP 图像数据
static void * stbi__bmp_load(stbi__context * s, int * x, int * y, int * comp, int req_comp, stbi__result_info * ri) {
    stbi_uc * out;
    unsigned int mr = 0, mg = 0, mb = 0, ma = 0, all_a;
    stbi_uc pal[256][4];
    int psize = 0, i, j, width;
    int flip_vertically, pad, target;
    stbi__bmp_data info;
    STBI_NOTUSED(ri);

    info.all_a = 255;
    # 解析 BMP 文件头部信息，如果出错则返回空
    if (stbi__bmp_parse_header(s, &info) == NULL)
        return NULL; # error code already set

    # 检查是否需要垂直翻转图片
    flip_vertically = ((int)s->img_y) > 0;
    s->img_y = abs((int)s->img_y);

    # 检查图片宽高是否过大
    if (s->img_y > STBI_MAX_DIMENSIONS)
        return stbi__errpuc("too large", "Very large image (corrupt?)");
    if (s->img_x > STBI_MAX_DIMENSIONS)
        return stbi__errpuc("too large", "Very large image (corrupt?)");

    # 读取颜色通道信息
    mr = info.mr;
    mg = info.mg;
    mb = info.mb;
    ma = info.ma;
    all_a = info.all_a;
    // 如果信息头大小为12
    if (info.hsz == 12) {
        // 如果每像素位数小于24，则计算调色板大小
        if (info.bpp < 24)
            psize = (info.offset - info.extra_read - 24) / 3;
    } else {
        // 如果每像素位数小于16，则计算调色板大小
        if (info.bpp < 16)
            psize = (info.offset - info.extra_read - info.hsz) >> 2;
    }
    // 如果调色板大小为0
    if (psize == 0) {
        // 接受头部之后的一些额外字节，但如果偏移量指向头部结束之前或者意味着大量额外数据，则拒绝文件
        int bytes_read_so_far = s->callback_already_read + (int)(s->img_buffer - s->img_buffer_original);
        int header_limit = 1024;        // 目前实际读取的最大值低于256字节。
        int extra_data_limit = 256 * 4; // 通常在这里的是调色板；256个条目*4字节是其最大大小。
        if (bytes_read_so_far <= 0 || bytes_read_so_far > header_limit) {
            return stbi__errpuc("bad header", "Corrupt BMP");
        }
        // 我们已经确定bytes_read_so_far是正数且合理的。
        // 这个测试的前半部分拒绝了太小的正偏移量，或者负偏移量，并保证info.offset >= bytes_read_so_far > 0。这反过来
        // 确保了在测试的后半部分计算的数字不会溢出。
        if (info.offset < bytes_read_so_far || info.offset - bytes_read_so_far > extra_data_limit) {
            return stbi__errpuc("bad offset", "Corrupt BMP");
        } else {
            stbi__skip(s, info.offset - bytes_read_so_far);
        }
    }

    // 如果每像素位数为24且ma为0xff000000，则图像通道数为3，否则为4或3
    if (info.bpp == 24 && ma == 0xff000000)
        s->img_n = 3;
    else
        s->img_n = ma ? 4 : 3;
    // 如果请求的通道数大于等于3，则目标通道数为请求的通道数，否则为图像的通道数
    if (req_comp && req_comp >= 3) // we can directly decode 3 or 4
        target = req_comp;
    else
        target = s->img_n; // if they want monochrome, we'll post-convert

    // 检查目标通道数和图像大小是否合理
    if (!stbi__mad3sizes_valid(target, s->img_x, s->img_y, 0))
        return stbi__errpuc("too large", "Corrupt BMP");
    # 分配足够大小的内存空间，用于存储解压后的图像数据
    out = (stbi_uc *)stbi__malloc_mad3(target, s->img_x, s->img_y, 0);
    # 如果内存分配失败，则返回错误信息
    if (!out)
        return stbi__errpuc("outofmem", "Out of memory");
    }

    // 如果目标通道数为4且所有的alpha通道值都为0，则将其替换为255
    if (target == 4 && all_a == 0)
        for (i = 4 * s->img_x * s->img_y - 1; i >= 0; i -= 4)
            out[i] = 255;

    # 如果需要垂直翻转图像
    if (flip_vertically) {
        stbi_uc t;
        for (j = 0; j < (int)s->img_y >> 1; ++j) {
            stbi_uc * p1 = out + j * s->img_x * target;
            stbi_uc * p2 = out + (s->img_y - 1 - j) * s->img_x * target;
            for (i = 0; i < (int)s->img_x * target; ++i) {
                t = p1[i];
                p1[i] = p2[i];
                p2[i] = t;
            }
        }
    }

    # 如果需要转换通道数，并且目标通道数不等于原通道数
    if (req_comp && req_comp != target) {
        # 转换图像数据的通道数
        out = stbi__convert_format(out, target, req_comp, s->img_x, s->img_y);
        # 如果转换失败，则返回空指针，stbi__convert_format 在失败时会释放输入数据
        if (out == NULL)
            return out;
    }

    # 返回图像的宽度和高度
    *x = s->img_x;
    *y = s->img_y;
    # 如果需要返回通道数，则返回通道数
    if (comp)
        *comp = s->img_n;
    # 返回解压后的图像数据
    return out;
}
#endif

// Targa Truevision - TGA
// by Jonathan Dummer
#ifndef STBI_NO_TGA
// 返回STBI_rgb或其他，错误时返回0
static int stbi__tga_get_comp(int bits_per_pixel, int is_grey, int * is_rgb16) {
    // 只允许RGB或RGBA（包括16位）或灰度
    if (is_rgb16)
        *is_rgb16 = 0;
    switch (bits_per_pixel) {
    case 8:
        return STBI_grey;
    case 16:
        if (is_grey)
            return STBI_grey_alpha;
        // 继续执行
    case 15:
        if (is_rgb16)
            *is_rgb16 = 1;
        return STBI_rgb;
    case 24: // 继续执行
    case 32:
        return bits_per_pixel / 8;
    default:
        return 0;
    }
}

static int stbi__tga_info(stbi__context * s, int * x, int * y, int * comp) {
    int tga_w, tga_h, tga_comp, tga_image_type, tga_bits_per_pixel, tga_colormap_bpp;
    int sz, tga_colormap_type;
    stbi__get8(s);                     // 丢弃偏移量
    tga_colormap_type = stbi__get8(s); // 调色板类型
    if (tga_colormap_type > 1) {
        stbi__rewind(s);
        return 0; // 只允许RGB或索引
    }
    tga_image_type = stbi__get8(s); // 图像类型
    if (tga_colormap_type == 1) {   // 调色板（调色板）图像
        if (tga_image_type != 1 && tga_image_type != 9) {
            stbi__rewind(s);
            return 0;
        }
        stbi__skip(s, 4);   // 跳过第一个调色板条目的索引和条目数
        sz = stbi__get8(s); // 检查调色板颜色条目的位数
        if ((sz != 8) && (sz != 15) && (sz != 16) && (sz != 24) && (sz != 32)) {
            stbi__rewind(s);
            return 0;
        }
        stbi__skip(s, 4); // 跳过图像x和y的原点
        tga_colormap_bpp = sz;
    } else { // 如果不是带有颜色映射的“普通”图像 - 只允许 RGB 或灰度，可能带有 RLE 压缩
        if ((tga_image_type != 2) && (tga_image_type != 3) && (tga_image_type != 10) && (tga_image_type != 11)) {
            stbi__rewind(s);
            return 0; // 只允许 RGB 或灰度，可能带有 RLE 压缩
        }
        stbi__skip(s, 9); // 跳过颜色映射规范和图像 x/y 起始位置
        tga_colormap_bpp = 0;
    }
    tga_w = stbi__get16le(s);
    if (tga_w < 1) {
        stbi__rewind(s);
        return 0; // 测试宽度
    }
    tga_h = stbi__get16le(s);
    if (tga_h < 1) {
        stbi__rewind(s);
        return 0; // 测试高度
    }
    tga_bits_per_pixel = stbi__get8(s); // 每像素位数
    stbi__get8(s);                      // 忽略 alpha 位
    if (tga_colormap_bpp != 0) {
        if ((tga_bits_per_pixel != 8) && (tga_bits_per_pixel != 16)) {
            // 使用颜色映射时，tga_bits_per_pixel 是索引的大小
            // 我认为只有 8 或 16 位索引才有意义
            stbi__rewind(s);
            return 0;
        }
        tga_comp = stbi__tga_get_comp(tga_colormap_bpp, 0, NULL);
    } else {
        tga_comp = stbi__tga_get_comp(tga_bits_per_pixel, (tga_image_type == 3) || (tga_image_type == 11), NULL);
    }
    if (!tga_comp) {
        stbi__rewind(s);
        return 0;
    }
    if (x)
        *x = tga_w;
    if (y)
        *y = tga_h;
    if (comp)
        *comp = tga_comp;
    return 1; // 似乎已经通过了所有测试
// 检测 TGA 文件是否符合格式要求
static int stbi__tga_test(stbi__context * s) {
    int res = 0;  // 初始化返回结果为 0
    int sz, tga_color_type;  // 定义变量 sz 和 tga_color_type
    stbi__get8(s);  // 读取一个字节并丢弃，即 Offset
    tga_color_type = stbi__get8(s);  // 读取一个字节，表示颜色类型
    if (tga_color_type > 1)  // 如果颜色类型大于 1
        goto errorEnd;  // 跳转到错误处理部分
    sz = stbi__get8(s);  // 读取一个字节，表示图像类型
    if (tga_color_type == 1) {  // 如果颜色类型为 1，即使用调色板的图像
        if (sz != 1 && sz != 9)  // 如果图像类型不是 1 或 9
            goto errorEnd;  // 跳转到错误处理部分
        stbi__skip(s, 4);  // 跳过调色板的第一个颜色索引和颜色数量
        sz = stbi__get8(s);  // 读取一个字节，表示调色板中每个颜色的位数
        if ((sz != 8) && (sz != 15) && (sz != 16) && (sz != 24) && (sz != 32))  // 如果调色板中每个颜色的位数不符合要求
            goto errorEnd;  // 跳转到错误处理部分
        stbi__skip(s, 4);  // 跳过图像的 x 和 y 起始位置
    } else {  // 如果不是使用调色板的图像
        if ((sz != 2) && (sz != 3) && (sz != 10) && (sz != 11))  // 如果图像类型不是 2、3、10 或 11
            goto errorEnd;  // 跳转到错误处理部分
        stbi__skip(s, 9);  // 跳过调色板规范和图像的 x/y 起始位置
    }
    if (stbi__get16le(s) < 1)  // 如果图像宽度小于 1
        goto errorEnd;  // 跳转到错误处理部分
    if (stbi__get16le(s) < 1)  // 如果图像高度小于 1
        goto errorEnd;  // 跳转到错误处理部分
    sz = stbi__get8(s);  // 读取一个字节，表示每个像素的位数
    if ((tga_color_type == 1) && (sz != 8) && (sz != 16))  // 如果是使用调色板的图像，并且每个像素的位数不是 8 或 16
        goto errorEnd;  // 跳转到错误处理部分
    if ((sz != 8) && (sz != 15) && (sz != 16) && (sz != 24) && (sz != 32))  // 如果每个像素的位数不符合要求
        goto errorEnd;  // 跳转到错误处理部分

    res = 1;  // 如果执行到这里，表示一切正常，将返回结果设为 1
errorEnd:
    stbi__rewind(s);  // 将流重置到起始位置
    return res;  // 返回结果
}

// 读取 16 位值并转换为 24 位 RGB
static void stbi__tga_read_rgb16(stbi__context * s, stbi_uc * out) {
    stbi__uint16 px = (stbi__uint16)stbi__get16le(s);  // 读取 16 位像素值
    stbi__uint16 fiveBitMask = 31;  // 定义 5 位掩码
    // 每个像素有 3 个通道，每个通道占 5 位
    int r = (px >> 10) & fiveBitMask;  // 计算红色通道值
    int g = (px >> 5) & fiveBitMask;  // 计算绿色通道值
    int b = px & fiveBitMask;  // 计算蓝色通道值
    // 注意，这里保存的数据是按照 RGB(A) 的顺序，所以后面不需要再进行交换
    out[0] = (stbi_uc)((r * 255) / 31);
    out[1] = (stbi_uc)((g * 255) / 31);
    out[2] = (stbi_uc)((b * 255) / 31);

    // 有些人声称最高有效位可能用于 alpha 通道
    // （可能是在“图像描述字节”中设置了 alpha 位）
    // 但这只会使16位测试图像完全透明...
    // 因此，让我们将所有的15位和16位 TGA 图像视为没有 alpha 通道的 RGB 图像。
    # 读取 TGA 文件的头部信息
    int tga_offset = stbi__get8(s);  # 读取 TGA 文件的偏移量
    int tga_indexed = stbi__get8(s);  # 读取 TGA 文件是否使用调色板
    int tga_image_type = stbi__get8(s);  # 读取 TGA 文件的图像类型
    int tga_is_RLE = 0;  # 初始化 RLE 压缩标志
    int tga_palette_start = stbi__get16le(s);  # 读取调色板的起始位置
    int tga_palette_len = stbi__get16le(s);  # 读取调色板的长度
    int tga_palette_bits = stbi__get8(s);  # 读取调色板的位数
    int tga_x_origin = stbi__get16le(s);  # 读取图像的 X 起始位置
    int tga_y_origin = stbi__get16le(s);  # 读取图像的 Y 起始位置
    int tga_width = stbi__get16le(s);  # 读取图像的宽度
    int tga_height = stbi__get16le(s);  # 读取图像的高度
    int tga_bits_per_pixel = stbi__get8(s);  # 读取每个像素的位数
    int tga_comp, tga_rgb16 = 0;  # 初始化图像通道数和 RGB16 标志
    int tga_inverted = stbi__get8(s);  # 读取图像是否翻转
    # int tga_alpha_bits = tga_inverted & 15; // the 4 lowest bits - unused (useless?)  # 读取图像的 alpha 位数
    # 图像数据
    unsigned char * tga_data;  # 定义图像数据指针
    unsigned char * tga_palette = NULL;  # 初始化调色板指针为空
    int i, j;  # 定义循环变量
    unsigned char raw_data[4] = {0};  # 初始化原始数据数组
    int RLE_count = 0;  # 初始化 RLE 计数
    int RLE_repeating = 0;  # 初始化 RLE 重复标志
    int read_next_pixel = 1;  # 初始化读取下一个像素标志
    STBI_NOTUSED(ri);  # 标记 ri 未使用
    STBI_NOTUSED(tga_x_origin); // @TODO  # 标记 tga_x_origin 未使用
    STBI_NOTUSED(tga_y_origin); // @TODO  # 标记 tga_y_origin 未使用

    if (tga_height > STBI_MAX_DIMENSIONS)  # 如果图像高度超出最大限制
        return stbi__errpuc("too large", "Very large image (corrupt?)");  # 返回错误信息
    if (tga_width > STBI_MAX_DIMENSIONS)  # 如果图像宽度超出最大限制
        return stbi__errpuc("too large", "Very large image (corrupt?)");  # 返回错误信息

    # 进行一些预处理
    if (tga_image_type >= 8):  # 如果图像类型大于等于8
        tga_image_type -= 8  # 减去8
        tga_is_RLE = 1  # 设置 RLE 压缩标志为1
    tga_inverted = 1 - ((tga_inverted >> 5) & 1)  # 计算图像是否翻转

    # 如果使用调色板，则使用调色板的位数
    if (tga_indexed)
        tga_comp = stbi__tga_get_comp(tga_palette_bits, 0, &tga_rgb16)
    else
        tga_comp = stbi__tga_get_comp(tga_bits_per_pixel, (tga_image_type == 3), &tga_rgb16)

    if (!tga_comp)  # 如果图像通道数为0
        return stbi__errpuc("bad format", "Can't find out TGA pixelformat");  # 返回错误信息
    # TGA 信息
    # 将 tga_width 的值赋给指针变量 x
    *x = tga_width;
    # 将 tga_height 的值赋给指针变量 y
    *y = tga_height;
    # 如果 comp 不为 0，则将 tga_comp 的值赋给指针变量 comp
    if (comp)
        *comp = tga_comp;

    # 检查 tga_width、tga_height 和 tga_comp 的乘积是否合法
    if (!stbi__mad3sizes_valid(tga_width, tga_height, tga_comp, 0))
        # 如果不合法，返回错误信息
        return stbi__errpuc("too large", "Corrupt TGA");

    # 分配内存给 tga_data，大小为 tga_width * tga_height * tga_comp
    tga_data = (unsigned char *)stbi__malloc_mad3(tga_width, tga_height, tga_comp, 0);
    # 如果分配内存失败，返回错误信息
    if (!tga_data)
        return stbi__errpuc("outofmem", "Out of memory");

    # 跳过数据的起始位置（通常偏移为 0）
    stbi__skip(s, tga_offset);

    # 如果不是索引颜色、不是RLE压缩、不是RGB16格式
    if (!tga_indexed && !tga_is_RLE && !tga_rgb16) {
        # 遍历 tga_height 行数据
        for (i = 0; i < tga_height; ++i) {
            # 如果 tga_inverted 为真，则行号为 tga_height - i - 1，否则为 i
            int row = tga_inverted ? tga_height - i - 1 : i;
            # 获取当前行的数据，存入 tga_row
            stbi_uc * tga_row = tga_data + row * tga_width * tga_comp;
            # 从输入流中读取 tga_width * tga_comp 个字节，存入 tga_row
            stbi__getn(s, tga_row, tga_width * tga_comp);
        }
    }

    # 如果 tga_comp 大于等于 3 且不是 RGB16 格式
    if (tga_comp >= 3 && !tga_rgb16) {
        # 获取 tga_data 的起始地址，存入 tga_pixel
        unsigned char * tga_pixel = tga_data;
        # 遍历 tga_width * tga_height 个像素
        for (i = 0; i < tga_width * tga_height; ++i) {
            # 交换 tga_pixel 的第一个和第三个字节的值
            unsigned char temp = tga_pixel[0];
            tga_pixel[0] = tga_pixel[2];
            tga_pixel[2] = temp;
            # 移动 tga_pixel 到下一个像素
            tga_pixel += tga_comp;
        }
    }

    # 如果 req_comp 存在且不等于 tga_comp
    if (req_comp && req_comp != tga_comp)
        # 转换 tga_data 的格式，使其符合 req_comp
        tga_data = stbi__convert_format(tga_data, tga_comp, req_comp, tga_width, tga_height);

    # 为了消除错误信息，保持 Microsoft 的 C 编译器的兼容性
    tga_palette_start = tga_palette_len = tga_palette_bits = tga_x_origin = tga_y_origin = 0;
    STBI_NOTUSED(tga_palette_start);
    # 返回 tga_data
    return tga_data;
}
#endif

// *************************************************************************************************
// Photoshop PSD loader -- PD by Thatcher Ulrich, integration by Nicolas Schulz, tweaked by STB

#ifndef STBI_NO_PSD
// 检测是否为 PSD 格式的文件
static int stbi__psd_test(stbi__context * s) {
    // 判断文件头是否为 0x38425053
    int r = (stbi__get32be(s) == 0x38425053);
    // 将文件指针重新定位到文件开头
    stbi__rewind(s);
    return r;
}

// 解码 PSD 文件中的 RLE 压缩数据
static int stbi__psd_decode_rle(stbi__context * s, stbi_uc * p, int pixelCount) {
    int count, nleft, len;

    count = 0;
    while ((nleft = pixelCount - count) > 0) {
        len = stbi__get8(s);
        if (len == 128) {
            // 无操作
        } else if (len < 128) {
            // 字面复制接下来的 len+1 个字节
            len++;
            if (len > nleft)
                return 0; // 数据损坏
            count += len;
            while (len) {
                *p = stbi__get8(s);
                p += 4;
                len--;
            }
        } else if (len > 128) {
            stbi_uc val;
            // 目标中的接下来的 -len+1 个字节从源中的下一个字节复制
            // (将 len 解释为负的 8 位整数)
            len = 257 - len;
            if (len > nleft)
                return 0; // 数据损坏
            val = stbi__get8(s);
            count += len;
            while (len) {
                *p = val;
                p += 4;
                len--;
            }
        }
    }

    return 1;
}

// 加载 PSD 文件
static void * stbi__psd_load(stbi__context * s, int * x, int * y, int * comp, int req_comp, stbi__result_info * ri, int bpc) {
    int pixelCount;
    int channelCount, compression;
    int channel, i;
    int bitdepth;
    int w, h;
    stbi_uc * out;
    STBI_NOTUSED(ri);

    // 检查标识符
    if (stbi__get32be(s) != 0x38425053) // "8BPS"
        return stbi__errpuc("not PSD", "Corrupt PSD image");

    // 检查文件类型版本
    if (stbi__get16be(s) != 1)
        return stbi__errpuc("wrong version", "Unsupported version of PSD image");
    // 跳过 PSD 文件中的 6 个保留字节
    stbi__skip(s, 6);

    // 读取图像的通道数（R、G、B、A 等）
    channelCount = stbi__get16be(s);
    if (channelCount < 0 || channelCount > 16)
        return stbi__errpuc("wrong channel count", "Unsupported number of channels in PSD image");

    // 读取图像的行数和列数
    h = stbi__get32be(s);
    w = stbi__get32be(s);

    if (h > STBI_MAX_DIMENSIONS)
        return stbi__errpuc("too large", "Very large image (corrupt?)");
    if (w > STBI_MAX_DIMENSIONS)
        return stbi__errpuc("too large", "Very large image (corrupt?)");

    // 确保图像的深度为 8 位
    bitdepth = stbi__get16be(s);
    if (bitdepth != 8 && bitdepth != 16)
        return stbi__errpuc("unsupported bit depth", "PSD bit depth is not 8 or 16 bit");

    // 确保颜色模式为 RGB
    // 有效选项包括：
    //   0: 位图
    //   1: 灰度
    //   2: 索引颜色
    //   3: RGB 颜色
    //   4: CMYK 颜色
    //   7: 多通道
    //   8: 双色调
    //   9: Lab 颜色
    if (stbi__get16be(s) != 3)
        return stbi__errpuc("wrong color format", "PSD is not in RGB color format");

    // 跳过模式数据（对于索引颜色是调色板；对于其他模式是其他信息）
    stbi__skip(s, stbi__get32be(s));

    // 跳过图像资源（分辨率、笔工具路径等）
    stbi__skip(s, stbi__get32be(s));

    // 跳过保留数据
    stbi__skip(s, stbi__get32be(s));

    // 查看数据是否被压缩
    // 已知的值包括：
    //   0: 无压缩
    //   1: RLE 压缩
    compression = stbi__get16be(s);
    if (compression > 1)
        return stbi__errpuc("bad compression", "PSD has an unknown compression format");

    // 检查大小
    if (!stbi__mad3sizes_valid(4, w, h, 0))
        return stbi__errpuc("too large", "Corrupt PSD");

    // 创建目标图像
    # 如果不压缩且位深为16且每通道位深为16
    if (!compression && bitdepth == 16 && bpc == 16) {
        # 分配内存空间，每像素8位，宽度w，高度h，总共8*w*h字节
        out = (stbi_uc *)stbi__malloc_mad3(8, w, h, 0);
        # 设置每通道位深为16
        ri->bits_per_channel = 16;
    } else
        # 分配内存空间，每像素4个字节，宽度w，高度h
        out = (stbi_uc *)stbi__malloc(4 * w * h);

    # 如果分配内存失败，返回错误信息
    if (!out)
        return stbi__errpuc("outofmem", "Out of memory");
    # 计算像素总数
    pixelCount = w * h;

    # 初始化数据为0
    # memset( out, 0, pixelCount * 4 );

    # 最后，图像数据
    # 如果使用了压缩
    if (compression) {
        # RLE压缩，用于.PSD和.TIFF格式
        # 循环直到得到期望的解压缩字节数：
        #     读取下一个源字节n
        #     如果n在0到127之间（包括0和127），就直接复制接下来的n+1个字节
        #     否则，如果n在-127到-1之间（包括-127和-1），就复制接下来的-byte -n+1次
        #     否则，如果n为128，不执行任何操作
        # 结束循环

        # RLE压缩数据之前有每行数据的2字节数据计数，我们将直接跳过
        stbi__skip(s, h * channelCount * 2);

        # 按通道读取RLE数据
        for (channel = 0; channel < 4; channel++) {
            stbi_uc * p;

            p = out + channel;
            if (channel >= channelCount) {
                # 用默认数据填充该通道
                for (i = 0; i < pixelCount; i++, p += 4)
                    *p = (channel == 3 ? 255 : 0);
            } else {
                # 读取RLE数据
                if (!stbi__psd_decode_rle(s, p, pixelCount)) {
                    STBI_FREE(out);
                    return stbi__errpuc("corrupt", "bad RLE data");
                }
            }
        }
    } else {
        // 如果代码执行到这里，说明已经到达原始图像数据部分。每个通道按顺序排列（红色、绿色、蓝色、透明度...）
        // 每个通道包含图像中每个像素的8位（或16位）值。

        // 按通道读取数据
        for (channel = 0; channel < 4; channel++) {
            if (channel >= channelCount) {
                // 用默认数据填充该通道
                if (bitdepth == 16 && bpc == 16) {
                    stbi__uint16 * q = ((stbi__uint16 *)out) + channel;
                    stbi__uint16 val = channel == 3 ? 65535 : 0;
                    for (i = 0; i < pixelCount; i++, q += 4)
                        *q = val;
                } else {
                    stbi_uc * p = out + channel;
                    stbi_uc val = channel == 3 ? 255 : 0;
                    for (i = 0; i < pixelCount; i++, p += 4)
                        *p = val;
                }
            } else {
                if (ri->bits_per_channel == 16) { // 输出 bpc
                    stbi__uint16 * q = ((stbi__uint16 *)out) + channel;
                    for (i = 0; i < pixelCount; i++, q += 4)
                        *q = (stbi__uint16)stbi__get16be(s);
                } else {
                    stbi_uc * p = out + channel;
                    if (bitdepth == 16) { // 输入 bpc
                        for (i = 0; i < pixelCount; i++, p += 4)
                            *p = (stbi_uc)(stbi__get16be(s) >> 8);
                    } else {
                        for (i = 0; i < pixelCount; i++, p += 4)
                            *p = stbi__get8(s);
                    }
                }
            }
        }
    }

    // 从 PSD 文件中移除奇怪的白色底板
    // 如果通道数大于等于4
    if (channelCount >= 4) {
        // 如果每个通道的位数为16
        if (ri->bits_per_channel == 16) {
            // 遍历每个像素
            for (i = 0; i < w * h; ++i) {
                // 将像素转换为16位整数指针
                stbi__uint16 * pixel = (stbi__uint16 *)out + 4 * i;
                // 如果像素的alpha通道不为0且不为最大值65535
                if (pixel[3] != 0 && pixel[3] != 65535) {
                    // 计算alpha通道的比例
                    float a = pixel[3] / 65535.0f;
                    // 计算alpha通道的倒数
                    float ra = 1.0f / a;
                    // 计算alpha通道的倒数的补数
                    float inv_a = 65535.0f * (1 - ra);
                    // 对RGB通道进行alpha混合
                    pixel[0] = (stbi__uint16)(pixel[0] * ra + inv_a);
                    pixel[1] = (stbi__uint16)(pixel[1] * ra + inv_a);
                    pixel[2] = (stbi__uint16)(pixel[2] * ra + inv_a);
                }
            }
        } else {
            // 如果每个通道的位数不为16
            for (i = 0; i < w * h; ++i) {
                // 将像素转换为无符号字符指针
                unsigned char * pixel = out + 4 * i;
                // 如果像素的alpha通道不为0且不为最大值255
                if (pixel[3] != 0 && pixel[3] != 255) {
                    // 计算alpha通道的比例
                    float a = pixel[3] / 255.0f;
                    // 计算alpha通道的倒数
                    float ra = 1.0f / a;
                    // 计算alpha通道的倒数的补数
                    float inv_a = 255.0f * (1 - ra);
                    // 对RGB通道进行alpha混合
                    pixel[0] = (unsigned char)(pixel[0] * ra + inv_a);
                    pixel[1] = (unsigned char)(pixel[1] * ra + inv_a);
                    pixel[2] = (unsigned char)(pixel[2] * ra + inv_a);
                }
            }
        }
    }

    // 转换为所需的输出格式
    if (req_comp && req_comp != 4) {
        // 如果通道数不为4且不为0
        if (ri->bits_per_channel == 16)
            // 如果每个通道的位数为16，调用16位格式转换函数
            out = (stbi_uc *)stbi__convert_format16((stbi__uint16 *)out, 4, req_comp, w, h);
        else
            // 如果每个通道的位数不为16，调用通用格式转换函数
            out = stbi__convert_format(out, 4, req_comp, w, h);
        // 如果转换失败，返回空指针
        if (out == NULL)
            return out; // stbi__convert_format frees input on failure
    }

    // 如果comp不为空，将其值设为4
    if (comp)
        *comp = 4;
    // 将y的值设为h
    *y = h;
    // 将x的值设为w
    *x = w;

    // 返回输出数据
    return out;
}
#endif

// *************************************************************************************************
// Softimage PIC loader
// by Tom Seddon
//
// See http://softimage.wiki.softimage.com/index.php/INFO:_PIC_file_format
// See http://ozviz.wasp.uwa.edu.au/~pbourke/dataformats/softimagepic/

#ifndef STBI_NO_PIC
// 检查读取的四个字节是否与给定字符串相同
static int stbi__pic_is4(stbi__context * s, const char * str) {
    int i;
    for (i = 0; i < 4; ++i)
        if (stbi__get8(s) != (stbi_uc)str[i])
            return 0;

    return 1;
}

// 检查文件是否符合 Softimage PIC 文件格式
static int stbi__pic_test_core(stbi__context * s) {
    int i;

    // 检查文件头部是否匹配指定的四个字节
    if (!stbi__pic_is4(s, "\x53\x80\xF6\x34"))
        return 0;

    // 跳过 84 个字节
    for (i = 0; i < 84; ++i)
        stbi__get8(s);

    // 检查文件头部是否包含 "PICT" 字符串
    if (!stbi__pic_is4(s, "PICT"))
        return 0;

    return 1;
}

// 定义 Softimage PIC 数据包结构
typedef struct {
    stbi_uc size, type, channel;
} stbi__pic_packet;

// 读取通道数据
static stbi_uc * stbi__readval(stbi__context * s, int channel, stbi_uc * dest) {
    int mask = 0x80, i;

    for (i = 0; i < 4; ++i, mask >>= 1) {
        if (channel & mask) {
            if (stbi__at_eof(s))
                return stbi__errpuc("bad file", "PIC file too short");
            dest[i] = stbi__get8(s);
        }
    }

    return dest;
}

// 复制通道数据
static void stbi__copyval(int channel, stbi_uc * dest, const stbi_uc * src) {
    int mask = 0x80, i;

    for (i = 0; i < 4; ++i, mask >>= 1)
        if (channel & mask)
            dest[i] = src[i];
}

// 加载 Softimage PIC 核心数据
static stbi_uc * stbi__pic_load_core(stbi__context * s, int width, int height, int * comp, stbi_uc * result) {
    int act_comp = 0, num_packets = 0, y, chained;
    stbi__pic_packet packets[10];

    // 这将处理一些奇怪的情况，比如在多个数据包中有相同通道的数据
    // 使用 do-while 循环来处理数据包，直到 chained 为假
    do {
        // 声明并初始化一个 stbi__pic_packet 结构体指针
        stbi__pic_packet * packet;

        // 如果数据包数量超过了预设的最大数量，返回错误信息
        if (num_packets == sizeof(packets) / sizeof(packets[0]))
            return stbi__errpuc("bad format", "too many packets");

        // 获取当前数据包的指针，并增加数据包数量
        packet = &packets[num_packets++];

        // 依次读取 chained、size、type、channel 的值
        chained = stbi__get8(s);
        packet->size = stbi__get8(s);
        packet->type = stbi__get8(s);
        packet->channel = stbi__get8(s);

        // 将当前数据包的 channel 加入到 act_comp 中
        act_comp |= packet->channel;

        // 如果已经到达文件末尾，返回文件过短的错误信息
        if (stbi__at_eof(s))
            return stbi__errpuc("bad file", "file too short (reading packets)");
        // 如果当前数据包的 size 不为 8，返回数据包格式错误的错误信息
        if (packet->size != 8)
            return stbi__errpuc("bad format", "packet isn't 8bpp");
    } while (chained); // 当 chained 为真时继续循环

    // 根据 act_comp 的值判断是否有 alpha 通道，将结果存入 comp 中
    *comp = (act_comp & 0x10 ? 4 : 3); // has alpha channel?

    // 返回处理结果
    return result;
}

# 加载 PIC 图像文件
static void * stbi__pic_load(stbi__context * s, int * px, int * py, int * comp, int req_comp, stbi__result_info * ri) {
    stbi_uc * result;
    int i, x, y, internal_comp;
    STBI_NOTUSED(ri);

    if (!comp)
        comp = &internal_comp;

    # 跳过前 92 个字节
    for (i = 0; i < 92; ++i)
        stbi__get8(s);

    # 读取图像宽度和高度
    x = stbi__get16be(s);
    y = stbi__get16be(s);

    # 检查图像尺寸是否过大
    if (y > STBI_MAX_DIMENSIONS)
        return stbi__errpuc("too large", "Very large image (corrupt?)");
    if (x > STBI_MAX_DIMENSIONS)
        return stbi__errpuc("too large", "Very large image (corrupt?)");

    # 检查文件是否过短
    if (stbi__at_eof(s))
        return stbi__errpuc("bad file", "file too short (pic header)");
    # 检查图像尺寸是否过大
    if (!stbi__mad3sizes_valid(x, y, 4, 0))
        return stbi__errpuc("too large", "PIC image too large to decode");

    stbi__get32be(s); # 跳过 `ratio'
    stbi__get16be(s); # 跳过 `fields'
    stbi__get16be(s); # 跳过 `pad'

    # 创建 RGBA 格式的中间缓冲区
    result = (stbi_uc *)stbi__malloc_mad3(x, y, 4, 0);
    if (!result)
        return stbi__errpuc("outofmem", "Out of memory");
    memset(result, 0xff, x * y * 4);

    # 加载 PIC 图像的核心部分
    if (!stbi__pic_load_core(s, x, y, comp, result)) {
        STBI_FREE(result);
        result = 0;
    }
    *px = x;
    *py = y;
    if (req_comp == 0)
        req_comp = *comp;
    result = stbi__convert_format(result, 4, req_comp, x, y);

    return result;
}

# 测试是否为 PIC 图像
static int stbi__pic_test(stbi__context * s) {
    int r = stbi__pic_test_core(s);
    stbi__rewind(s);
    return r;
}
#endif

# *************************************************************************************************
# GIF loader -- public domain by Jean-Marc Lienher -- simplified/shrunk by stb

# 如果未定义 STBI_NO_GIF，则进行 GIF 加载
#ifndef STBI_NO_GIF
typedef struct {
    stbi__int16 prefix;
    stbi_uc first;
    stbi_uc suffix;
} stbi__gif_lzw;

typedef struct {
    int w, h;
    stbi_uc * out;        # 输出缓冲区（始终为 4 个分量）
    stbi_uc * background; # GIF 图像的当前“背景”
    stbi_uc * history;
    # 定义整型变量 flags，用于存储图像的标志信息
    int flags, 
    # 定义整型变量 bgindex，用于存储图像的背景颜色索引
    bgindex, 
    # 定义整型变量 ratio，用于存储图像的比率
    ratio, 
    # 定义整型变量 transparent，用于存储图像的透明度
    transparent, 
    # 定义整型变量 eflags，用于存储图像的扩展标志
    eflags;
    # 定义一个 256x4 的数组 pal，用于存储图像的调色板
    stbi_uc pal[256][4];
    # 定义一个 256x4 的数组 lpal，用于存储图像的局部调色板
    stbi_uc lpal[256][4];
    # 定义一个 8192 大小的 stbi__gif_lzw 结构体数组 codes，用于存储图像的 LZW 编码
    stbi__gif_lzw codes[8192];
    # 定义一个 stbi_uc 类型的指针 color_table，用于存储图像的颜色表
    stbi_uc * color_table;
    # 定义整型变量 parse，用于存储图像的解析信息
    int parse, 
    # 定义整型变量 step，用于存储图像的步长
    step;
    # 定义整型变量 lflags，用于存储图像的局部标志
    int lflags;
    # 定义整型变量 start_x，start_y，max_x，max_y，cur_x，cur_y，line_size，delay，用于存储图像的位置、大小、延迟等信息
    int start_x, start_y, max_x, max_y, cur_x, cur_y, line_size, delay;
# 定义一个静态的 GIF 结构体
} stbi__gif;

# 测试是否为原始的 GIF 格式
static int stbi__gif_test_raw(stbi__context * s) {
    int sz;
    # 检查文件头是否为 "GIF8"
    if (stbi__get8(s) != 'G' || stbi__get8(s) != 'I' || stbi__get8(s) != 'F' || stbi__get8(s) != '8')
        return 0;
    # 获取版本号
    sz = stbi__get8(s);
    # 检查版本号是否为 '9' 或 '7'
    if (sz != '9' && sz != '7')
        return 0;
    # 检查是否为动画 GIF
    if (stbi__get8(s) != 'a')
        return 0;
    return 1;
}

# 测试是否为 GIF 格式
static int stbi__gif_test(stbi__context * s) {
    # 调用原始测试函数
    int r = stbi__gif_test_raw(s);
    # 重置文件指针
    stbi__rewind(s);
    return r;
}

# 解析 GIF 颜色表
static void stbi__gif_parse_colortable(stbi__context * s, stbi_uc pal[256][4], int num_entries, int transp) {
    int i;
    # 遍历颜色表
    for (i = 0; i < num_entries; ++i) {
        # 读取颜色值
        pal[i][2] = stbi__get8(s);
        pal[i][1] = stbi__get8(s);
        pal[i][0] = stbi__get8(s);
        # 设置透明颜色
        pal[i][3] = transp == i ? 0 : 255;
    }
}

# 解析 GIF 头部信息
static int stbi__gif_header(stbi__context * s, stbi__gif * g, int * comp, int is_info) {
    stbi_uc version;
    # 检查文件头是否为 "GIF8"
    if (stbi__get8(s) != 'G' || stbi__get8(s) != 'I' || stbi__get8(s) != 'F' || stbi__get8(s) != '8')
        return stbi__err("not GIF", "Corrupt GIF");

    # 获取版本号
    version = stbi__get8(s);
    # 检查版本号是否为 '7' 或 '9'
    if (version != '7' && version != '9')
        return stbi__err("not GIF", "Corrupt GIF");
    # 检查是否为动画 GIF
    if (stbi__get8(s) != 'a')
        return stbi__err("not GIF", "Corrupt GIF");

    # 初始化 GIF 结构体
    stbi__g_failure_reason = "";
    g->w = stbi__get16le(s);
    g->h = stbi__get16le(s);
    g->flags = stbi__get8(s);
    g->bgindex = stbi__get8(s);
    g->ratio = stbi__get8(s);
    g->transparent = -1;

    # 检查图片尺寸是否过大
    if (g->w > STBI_MAX_DIMENSIONS)
        return stbi__err("too large", "Very large image (corrupt?)");
    if (g->h > STBI_MAX_DIMENSIONS)
        return stbi__err("too large", "Very large image (corrupt?)");

    # 设置颜色通道数
    if (comp != 0)
        *comp = 4; # 在解析注释之前无法确定是 3 还是 4

    # 如果只是获取信息，则返回 1
    if (is_info)
        return 1;

    # 解析全局颜色表
    if (g->flags & 0x80)
        stbi__gif_parse_colortable(s, g->pal, 2 << (g->flags & 7), -1);

    return 1;
}
static int stbi__gif_info_raw(stbi__context * s, int * x, int * y, int * comp) {
    // 分配内存给 GIF 结构体
    stbi__gif * g = (stbi__gif *)stbi__malloc(sizeof(stbi__gif));
    // 如果内存分配失败，返回错误信息
    if (!g)
        return stbi__err("outofmem", "Out of memory");
    // 读取 GIF 头信息
    if (!stbi__gif_header(s, g, comp, 1)) {
        // 释放内存
        STBI_FREE(g);
        // 重新定位到文件开头
        stbi__rewind(s);
        return 0;
    }
    // 如果 x 不为空，将 g 的宽度赋值给 x
    if (x)
        *x = g->w;
    // 如果 y 不为空，将 g 的高度赋值给 y
    if (y)
        *y = g->h;
    // 释放内存
    STBI_FREE(g);
    return 1;
}

static void stbi__out_gif_code(stbi__gif * g, stbi__uint16 code) {
    stbi_uc *p, *c;
    int idx;

    // 递归解码前缀，因为链表是反向的，反向处理交错图像会很麻烦
    if (g->codes[code].prefix >= 0)
        stbi__out_gif_code(g, g->codes[code].prefix);

    // 如果当前行数超过最大行数，直接返回
    if (g->cur_y >= g->max_y)
        return;

    // 计算当前像素索引
    idx = g->cur_x + g->cur_y;
    p = &g->out[idx];
    g->history[idx / 4] = 1;

    // 获取颜色表中的颜色值
    c = &g->color_table[g->codes[code].suffix * 4];
    // 如果透明度大于128，不渲染透明像素
    if (c[3] > 128) {
        p[0] = c[2];
        p[1] = c[1];
        p[2] = c[0];
        p[3] = c[3];
    }
    g->cur_x += 4;

    // 如果当前行数超过最大行数
    if (g->cur_x >= g->max_x) {
        g->cur_x = g->start_x;
        g->cur_y += g->step;

        // 处理当前行数超过最大行数的情况
        while (g->cur_y >= g->max_y && g->parse > 0) {
            g->step = (1 << g->parse) * g->line_size;
            g->cur_y = g->start_y + (g->step >> 1);
            --g->parse;
        }
    }
}

static stbi_uc * stbi__process_gif_raster(stbi__context * s, stbi__gif * g) {
    stbi_uc lzw_cs;
    stbi__int32 len, init_code;
    stbi__uint32 first;
    stbi__int32 codesize, codemask, avail, oldcode, bits, valid_bits, clear;
    stbi__gif_lzw * p;

    // 读取 LZW 编码的代码尺寸
    lzw_cs = stbi__get8(s);
    // 如果代码尺寸大于12，返回空指针
    if (lzw_cs > 12)
        return NULL;
    // 计算清除码
    clear = 1 << lzw_cs;
    first = 1;
    codesize = lzw_cs + 1;
    codemask = (1 << codesize) - 1;
    bits = 0;
    valid_bits = 0;
}
    # 对于每个编码，设置前缀为-1，第一个字符为当前编码，后缀为当前编码
    for (init_code = 0; init_code < clear; init_code++) {
        g->codes[init_code].prefix = -1;
        g->codes[init_code].first = (stbi_uc)init_code;
        g->codes[init_code].suffix = (stbi_uc)init_code;
    }

    // 支持没有起始清除编码
    avail = clear + 2;
    oldcode = -1;

    len = 0;
// 该函数旨在支持动画 GIF，尽管 stb_image 不支持
// two_back 是两帧前的图像，用于非常特定的处理格式
static stbi_uc * stbi__gif_load_next(stbi__context * s, stbi__gif * g, int * comp, int req_comp, stbi_uc * two_back) {
    int dispose; // 处理方式
    int first_frame; // 是否第一帧
    int pi; // 像素索引
    int pcount; // 像素数量
    STBI_NOTUSED(req_comp); // 不使用 req_comp

    // 在第一帧上，任何未写入的像素都会得到背景颜色（非透明）
    first_frame = 0;
    if (g->out == 0) {
        // 如果输出为空，则读取 GIF 头信息
        if (!stbi__gif_header(s, g, comp, 0))
            return 0; // stbi__g_failure_reason 由 stbi__gif_header 设置
        // 检查图像大小是否合法
        if (!stbi__mad3sizes_valid(4, g->w, g->h, 0))
            return stbi__errpuc("too large", "GIF image is too large");
        pcount = g->w * g->h;
        // 分配内存
        g->out = (stbi_uc *)stbi__malloc(4 * pcount);
        g->background = (stbi_uc *)stbi__malloc(4 * pcount);
        g->history = (stbi_uc *)stbi__malloc(pcount);
        if (!g->out || !g->background || !g->history)
            return stbi__errpuc("outofmem", "Out of memory");

        // 图像在开始时被视为“透明” - 即，没有东西覆盖当前背景；
        // 背景颜色仅用于第一帧未渲染的像素，之后“背景”颜色指的是上一帧的颜色。
        memset(g->out, 0x00, 4 * pcount);
        memset(g->background, 0x00, 4 * pcount); // 背景的状态（开始透明）
        memset(g->history, 0x00, pcount);        // 上一帧受影响的像素
        first_frame = 1;
    } else {
        // 第二帧 - 我们如何处理前一帧？
        dispose = (g->eflags & 0x1C) >> 2;  // 从eflags中获取dispose值
        pcount = g->w * g->h;  // 计算像素总数

        if ((dispose == 3) && (two_back == 0)) {
            dispose = 2; // 如果没有图像可以回退，则默认为旧背景
        }

        if (dispose == 3) { // 使用上一帧图像
            for (pi = 0; pi < pcount; ++pi) {
                if (g->history[pi]) {
                    memcpy(&g->out[pi * 4], &two_back[pi * 4], 4);  // 将上一帧图像复制到当前帧
                }
            }
        } else if (dispose == 2) {
            // 恢复上一帧更改到之前帧的背景；
            for (pi = 0; pi < pcount; ++pi) {
                if (g->history[pi]) {
                    memcpy(&g->out[pi * 4], &g->background[pi * 4], 4);  // 将更改的像素恢复到之前的背景
                }
            }
        } else {
            // 这是一个非处理情况，所以只需保留像素，它们将成为新的背景
            // 1：不处理
            // 0：未指定
        }

        // 背景是撤消前一帧后的输出；
        memcpy(g->background, g->out, 4 * g->w * g->h);  // 将当前帧的输出作为新的背景
    }

    // 清除我的历史记录；
    memset(g->history, 0x00, g->w * g->h); // 上一帧受影响的像素

    }
# 释放 GIF 结构体中的输出、历史记录和背景内存
static void * stbi__load_gif_main_outofmem(stbi__gif * g, stbi_uc * out, int ** delays) {
    STBI_FREE(g->out);  # 释放输出内存
    STBI_FREE(g->history);  # 释放历史记录内存
    STBI_FREE(g->background);  # 释放背景内存

    if (out)
        STBI_FREE(out);  # 如果输出不为空，则释放输出内存
    if (delays && *delays)
        STBI_FREE(*delays);  # 如果延迟不为空，则释放延迟内存
    return stbi__errpuc("outofmem", "Out of memory");  # 返回内存不足的错误信息
}

# 加载 GIF 主函数，处理图像数据
static void * stbi__load_gif_main(stbi__context * s, int ** delays, int * x, int * y, int * z, int * comp, int req_comp) {
    } else {
        return stbi__errpuc("not GIF", "Image was not as a gif type.");  # 如果不是 GIF 类型的图像，则返回错误信息
    }
}

# 加载 GIF 图像
static void * stbi__gif_load(stbi__context * s, int * x, int * y, int * comp, int req_comp, stbi__result_info * ri) {
    stbi_uc * u = 0;  # 初始化图像数据指针为 0
    stbi__gif g;  # 创建 GIF 结构体
    memset(&g, 0, sizeof(g));  # 将 GIF 结构体清零
    STBI_NOTUSED(ri);  # 标记 ri 未使用

    u = stbi__gif_load_next(s, &g, comp, req_comp, 0);  # 加载 GIF 的下一帧图像
    if (u == (stbi_uc *)s)
        u = 0;  # 如果图像数据指针等于 s，则置为 0，表示动画 GIF 结束
    if (u) {
        *x = g.w;  # 将图像宽度赋值给 x
        *y = g.h;  # 将图像高度赋值给 y

        # 在成功加载后进行格式转换，以便处理多帧图像
        if (req_comp && req_comp != 4)
            u = stbi__convert_format(u, 4, req_comp, g.w, g.h);  # 如果请求的通道数不为 4，则进行格式转换
    } else if (g.out) {
        # 如果出现错误并且分配了图像缓冲区，则释放它
        STBI_FREE(g.out);  # 释放图像缓冲区内存
    }

    # 释放多帧加载所需的缓冲区
    STBI_FREE(g.history);  # 释放历史记录内存
    STBI_FREE(g.background);  # 释放背景内存

    return u;  # 返回图像数据指针
}

# 获取 GIF 图像信息
static int stbi__gif_info(stbi__context * s, int * x, int * y, int * comp) { return stbi__gif_info_raw(s, x, y, comp); }  # 获取 GIF 图像的原始信息
#endif

# *************************************************************************************************
# Radiance RGBE HDR 加载器
# 原作者：Nicolas Schulz
#ifndef STBI_NO_HDR
# 测试 HDR 文件的核心函数
static int stbi__hdr_test_core(stbi__context * s, const char * signature) {
    int i;
    for (i = 0; signature[i]; ++i)
        if (stbi__get8(s) != signature[i])  # 如果读取的字节与签名不匹配
            return 0;  # 返回 0，表示不是 HDR 文件
    stbi__rewind(s);  # 重新定位文件指针到文件开头
    return 1;  # 返回 1，表示是 HDR 文件
}

# 测试是否为 HDR 文件
static int stbi__hdr_test(stbi__context * s) {
    # 调用 stbi__hdr_test_core 函数测试是否为 RADIANCE 格式的 HDR 文件，并将结果存储在 r 变量中
    int r = stbi__hdr_test_core(s, "#?RADIANCE\n");
    # 将文件指针重新定位到文件开头
    stbi__rewind(s);
    # 如果不是 RADIANCE 格式的 HDR 文件，则再次调用 stbi__hdr_test_core 函数测试是否为 RGBE 格式的 HDR 文件，并将结果存储在 r 变量中
    if (!r) {
        r = stbi__hdr_test_core(s, "#?RGBE\n");
        # 将文件指针重新定位到文件开头
        stbi__rewind(s);
    }
    # 返回最终的测试结果
    return r;
    # 定义 STBI__HDR_BUFLEN 常量为 1024
#define STBI__HDR_BUFLEN 1024
# 定义函数 stbi__hdr_gettoken，接受 stbi__context 类型的参数 z 和字符指针类型的参数 buffer
static char * stbi__hdr_gettoken(stbi__context * z, char * buffer) {
    int len = 0;  # 初始化 len 变量为 0
    char c = '\0';  # 初始化 c 变量为空字符

    c = (char)stbi__get8(z);  # 从输入流中获取一个字节，转换成字符类型并赋值给 c

    while (!stbi__at_eof(z) && c != '\n') {  # 当未到达文件末尾且字符不是换行符时执行循环
        buffer[len++] = c;  # 将字符 c 存入 buffer 中，并递增 len
        if (len == STBI__HDR_BUFLEN - 1) {  # 如果 len 达到 STBI__HDR_BUFLEN - 1
            // flush to end of line
            while (!stbi__at_eof(z) && stbi__get8(z) != '\n')  # 刷新到行末
                ;
            break;
        }
        c = (char)stbi__get8(z);  # 从输入流中获取一个字节，转换成字符类型并赋值给 c
    }

    buffer[len] = 0;  # 在 buffer 的末尾添加空字符
    return buffer;  # 返回 buffer
}

# 定义函数 stbi__hdr_convert，接受 float 类型指针 output、stbi_uc 类型指针 input、整型 req_comp 作为参数
static void stbi__hdr_convert(float * output, stbi_uc * input, int req_comp) {
    if (input[3] != 0) {  # 如果输入的第四个元素不为 0
        float f1;  # 定义浮点数 f1
        // Exponent
        f1 = (float)ldexp(1.0f, input[3] - (int)(128 + 8));  # 计算指数
        if (req_comp <= 2)  # 如果请求的通道数小于等于 2
            output[0] = (input[0] + input[1] + input[2]) * f1 / 3;  # 计算输出的第一个元素
        else:  # 否则
            output[0] = input[0] * f1;  # 计算输出的第一个元素
            output[1] = input[1] * f1;  # 计算输出的第二个元素
            output[2] = input[2] * f1;  # 计算输出的第三个元素
        if (req_comp == 2)  # 如果请求的通道数为 2
            output[1] = 1;  # 输出的第二个元素为 1
        if (req_comp == 4)  # 如果请求的通道数为 4
            output[3] = 1;  # 输出的第四个元素为 1
    } else {  # 否则
        switch (req_comp) {  # 根据请求的通道数进行判断
        case 4:
            output[3] = 1; /* fallthrough */  # 输出的第四个元素为 1
        case 3:
            output[0] = output[1] = output[2] = 0;  # 输出的前三个元素为 0
            break;
        case 2:
            output[1] = 1; /* fallthrough */  # 输出的第二个元素为 1
        case 1:
            output[0] = 0;  # 输出的第一个元素为 0
            break;
        }
    }
}

# 定义函数 stbi__hdr_load，接受 stbi__context 类型指针 s、整型指针 x、y、comp、req_comp 和 stbi__result_info 类型指针 ri 作为参数
static float * stbi__hdr_load(stbi__context * s, int * x, int * y, int * comp, int req_comp, stbi__result_info * ri) {
    char buffer[STBI__HDR_BUFLEN];  # 定义长度为 STBI__HDR_BUFLEN 的字符数组 buffer
    char * token;  # 定义字符指针 token
    int valid = 0;  # 初始化 valid 变量为 0
    int width, height;  # 定义宽度和高度变量
    stbi_uc * scanline;  # 定义 stbi_uc 类型指针 scanline
    float * hdr_data;  # 定义浮点数指针 hdr_data
    int len;  # 定义整型变量 len
    unsigned char count, value;  # 定义无符号字符 count 和 value
    int i, j, k, c1, c2, z;  # 定义整型变量 i、j、k、c1、c2、z
    const char * headerToken;  # 定义常量字符指针 headerToken
    STBI_NOTUSED(ri);  # 使用 STBI_NOTUSED 宏

    // Check identifier
    headerToken = stbi__hdr_gettoken(s, buffer);  # 调用 stbi__hdr_gettoken 函数，将返回值赋给 headerToken
    if (strcmp(headerToken, "#?RADIANCE") != 0 && strcmp(headerToken, "#?RGBE") != 0)  # 如果 headerToken 不等于 "#?RADIANCE" 且不等于 "#?RGBE"
        return stbi__errpf("not HDR", "Corrupt HDR image");  # 返回错误信息
    # 解析头部信息
    while True:
        # 获取下一个 token
        token = stbi__hdr_gettoken(s, buffer)
        # 如果 token 为空，则跳出循环
        if (token[0] == 0):
            break
        # 如果 token 为 "FORMAT=32-bit_rle_rgbe"，则设置 valid 为 1
        if (strcmp(token, "FORMAT=32-bit_rle_rgbe") == 0):
            valid = 1

    # 如果 valid 为假，则返回错误信息
    if (!valid):
        return stbi__errpf("unsupported format", "Unsupported HDR format")

    # 解析宽度和高度
    # 不能使用 sscanf()，因为我们没有使用 stdio！
    token = stbi__hdr_gettoken(s, buffer)
    # 如果 token 不以 "-Y " 开头，则返回错误信息
    if (strncmp(token, "-Y ", 3)):
        return stbi__errpf("unsupported data layout", "Unsupported HDR format")
    token += 3
    # 将 token 转换为整数作为高度
    height = (int)strtol(token, &token, 10)
    while (*token == ' '):
        ++token
    # 如果 token 不以 "+X " 开头，则返回错误信息
    if (strncmp(token, "+X ", 3)):
        return stbi__errpf("unsupported data layout", "Unsupported HDR format")
    token += 3
    # 将 token 转换为整数作为宽度
    width = (int)strtol(token, NULL, 10)

    # 如果高度或宽度超过最大限制，则返回错误信息
    if (height > STBI_MAX_DIMENSIONS):
        return stbi__errpf("too large", "Very large image (corrupt?)")
    if (width > STBI_MAX_DIMENSIONS):
        return stbi__errpf("too large", "Very large image (corrupt?)")

    # 将宽度和高度分别赋值给指针 x 和 y
    *x = width;
    *y = height;

    # 如果 comp 不为空，则将其赋值为 3
    if (comp):
        *comp = 3
    # 如果 req_comp 为 0，则将其赋值为 3
    if (req_comp == 0):
        req_comp = 3

    # 如果图像尺寸过大，则返回错误信息
    if (!stbi__mad4sizes_valid(width, height, req_comp, sizeof(float), 0)):
        return stbi__errpf("too large", "HDR image is too large")

    # 读取数据
    hdr_data = (float *)stbi__malloc_mad4(width, height, req_comp, sizeof(float), 0)
    # 如果分配内存失败，则返回错误信息
    if (!hdr_data):
        return stbi__errpf("outofmem", "Out of memory")

    # 加载图像数据
    # 图像数据存储为一定数量的 sca
    if (width < 8 || width >= 32768):
        # 读取扁平数据
        for (j = 0; j < height; ++j):
            for (i = 0; i < width; ++i):
                stbi_uc rgbe[4]
                # 主解码循环
                stbi__getn(s, rgbe, 4)
                stbi__hdr_convert(hdr_data + j * width * req_comp + i * req_comp, rgbe, req_comp)

    # 返回 hdr_data
    return hdr_data
# 检查并获取 HDR 文件的信息
static int stbi__hdr_info(stbi__context * s, int * x, int * y, int * comp) {
    char buffer[STBI__HDR_BUFLEN];  # 创建一个缓冲区
    char * token;  # 创建一个指向字符的指针
    int valid = 0;  # 初始化有效性标志为 0
    int dummy;  # 创建一个虚拟变量

    if (!x)  # 如果 x 为空
        x = &dummy;  # 将 x 指向虚拟变量 dummy
    if (!y)  # 如果 y 为空
        y = &dummy;  # 将 y 指向虚拟变量 dummy
    if (!comp)  # 如果 comp 为空
        comp = &dummy;  # 将 comp 指向虚拟变量 dummy

    if (stbi__hdr_test(s) == 0) {  # 如果不是 HDR 文件
        stbi__rewind(s);  # 重置文件指针
        return 0;  # 返回 0
    }

    for (;;) {  # 无限循环
        token = stbi__hdr_gettoken(s, buffer);  # 获取 HDR 文件中的 token
        if (token[0] == 0)  # 如果 token 的第一个字符为 0
            break;  # 退出循环
        if (strcmp(token, "FORMAT=32-bit_rle_rgbe") == 0)  # 如果 token 为 "FORMAT=32-bit_rle_rgbe"
            valid = 1;  # 设置有效性标志为 1
    }

    if (!valid) {  # 如果不是有效的 HDR 文件
        stbi__rewind(s);  # 重置文件指针
        return 0;  # 返回 0
    }
    token = stbi__hdr_gettoken(s, buffer);  # 获取 HDR 文件中的 token
    if (strncmp(token, "-Y ", 3)) {  # 如果 token 不以 "-Y " 开头
        stbi__rewind(s);  # 重置文件指针
        return 0;  # 返回 0
    }
    token += 3;  # 将 token 指针向后移动 3 个位置
    *y = (int)strtol(token, &token, 10);  # 将 token 转换为整数，存入 y 中
    while (*token == ' ')  # 当 token 指向的字符为空格时
        ++token;  # token 指针向后移动一位
    if (strncmp(token, "+X ", 3)) {  # 如果 token 不以 "+X " 开头
        stbi__rewind(s);  # 重置文件指针
        return 0;  # 返回 0
    }
    token += 3;  # 将 token 指针向后移动 3 个位置
    *x = (int)strtol(token, NULL, 10);  # 将 token 转换为整数，存入 x 中
    *comp = 3;  # 设置 comp 为 3
    return 1;  # 返回 1
}
#endif // STBI_NO_HDR

#ifndef STBI_NO_BMP
static int stbi__bmp_info(stbi__context * s, int * x, int * y, int * comp) {
    void * p;  # 创建一个指向 void 的指针
    stbi__bmp_data info;  # 创建一个 stbi__bmp_data 结构体变量

    info.all_a = 255;  # 设置 info 结构体的 all_a 成员为 255
    p = stbi__bmp_parse_header(s, &info);  # 解析 BMP 文件头部信息
    if (p == NULL) {  # 如果指针 p 为空
        stbi__rewind(s);  # 重置文件指针
        return 0;  # 返回 0
    }
    if (x)  # 如果 x 不为空
        *x = s->img_x;  # 将图像宽度赋值给 x
    if (y)  # 如果 y 不为空
        *y = s->img_y;  # 将图像高度赋值给 y
    if (comp) {  # 如果 comp 不为空
        if (info.bpp == 24 && info.ma == 0xff000000)  # 如果位深为 24 且透明度为 0xff000000
            *comp = 3;  # 设置 comp 为 3
        else
            *comp = info.ma ? 4 : 3;  # 否则根据透明度设置 comp 为 4 或 3
    }
    return 1;  # 返回 1
}
#endif

#ifndef STBI_NO_PSD
static int stbi__psd_info(stbi__context * s, int * x, int * y, int * comp) {
    int channelCount, dummy, depth;  # 创建通道数、虚拟变量和深度变量

    if (!x)  # 如果 x 为空
        x = &dummy;  # 将 x 指向虚拟变量 dummy
    if (!y)  # 如果 y 为空
        y = &dummy;  # 将 y 指向虚拟变量 dummy
    if (!comp)  # 如果 comp 为空
        comp = &dummy;  # 将 comp 指向虚拟变量 dummy
    if (stbi__get32be(s) != 0x38425053) {  # 如果 PSD 文件标识不匹配
        stbi__rewind(s);  # 重置文件指针
        return 0;  # 返回 0
    }
    if (stbi__get16be(s) != 1) {  # 如果版本号不为 1
        stbi__rewind(s);  # 重置文件指针
        return 0;  # 返回 0
    }
    stbi__skip(s, 6);  # 跳过 6 个字节
    # 读取通道数，使用大端序读取两个字节
    channelCount = stbi__get16be(s);
    # 如果通道数小于0或大于16，则将文件指针重置到起始位置并返回0
    if (channelCount < 0 || channelCount > 16) {
        stbi__rewind(s);
        return 0;
    }
    # 读取图像的高度，使用大端序读取四个字节
    *y = stbi__get32be(s);
    # 读取图像的宽度，使用大端序读取四个字节
    *x = stbi__get32be(s);
    # 读取图像的深度，使用大端序读取两个字节
    depth = stbi__get16be(s);
    # 如果深度不是8或16，则将文件指针重置到起始位置并返回0
    if (depth != 8 && depth != 16) {
        stbi__rewind(s);
        return 0;
    }
    # 读取图像的组件数，使用大端序读取两个字节
    if (stbi__get16be(s) != 3) {
        stbi__rewind(s);
        return 0;
    }
    # 设置图像的组件数为4
    *comp = 4;
    # 返回1表示成功
    return 1;
# 检查是否为16位PSD文件
static int stbi__psd_is16(stbi__context * s) {
    int channelCount, depth;
    # 检查文件头是否为0x38425053
    if (stbi__get32be(s) != 0x38425053) {
        stbi__rewind(s);
        return 0;
    }
    # 检查版本号是否为1
    if (stbi__get16be(s) != 1) {
        stbi__rewind(s);
        return 0;
    }
    # 跳过6个字节
    stbi__skip(s, 6);
    # 读取通道数
    channelCount = stbi__get16be(s);
    # 检查通道数是否在0到16之间
    if (channelCount < 0 || channelCount > 16) {
        stbi__rewind(s);
        return 0;
    }
    # 跳过4个字节
    STBI_NOTUSED(stbi__get32be(s));
    STBI_NOTUSED(stbi__get32be(s));
    # 读取深度
    depth = stbi__get16be(s);
    # 检查深度是否为16
    if (depth != 16) {
        stbi__rewind(s);
        return 0;
    }
    return 1;
}
#endif

#ifndef STBI_NO_PIC
# 获取PIC文件信息
static int stbi__pic_info(stbi__context * s, int * x, int * y, int * comp) {
    int act_comp = 0, num_packets = 0, chained, dummy;
    stbi__pic_packet packets[10];

    # 如果x为NULL，则指向dummy
    if (!x)
        x = &dummy;
    # 如果y为NULL，则指向dummy
    if (!y)
        y = &dummy;
    # 如果comp为NULL，则指向dummy
    if (!comp)
        comp = &dummy;

    # 检查文件头是否为"\x53\x80\xF6\x34"
    if (!stbi__pic_is4(s, "\x53\x80\xF6\x34")) {
        stbi__rewind(s);
        return 0;
    }

    # 跳过88个字节
    stbi__skip(s, 88);

    # 读取宽度和高度
    *x = stbi__get16be(s);
    *y = stbi__get16be(s);
    # 检查是否已到达文件末尾
    if (stbi__at_eof(s)) {
        stbi__rewind(s);
        return 0;
    }
    # 检查宽度是否合法
    if ((*x) != 0 && (1 << 28) / (*x) < (*y)) {
        stbi__rewind(s);
        return 0;
    }

    # 跳过8个字节
    stbi__skip(s, 8);

    do {
        stbi__pic_packet * packet;

        # 如果包数量达到上限，则返回0
        if (num_packets == sizeof(packets) / sizeof(packets[0]))
            return 0;

        packet = &packets[num_packets++];
        chained = stbi__get8(s);
        packet->size = stbi__get8(s);
        packet->type = stbi__get8(s);
        packet->channel = stbi__get8(s);
        act_comp |= packet->channel;

        # 检查是否已到达文件末尾
        if (stbi__at_eof(s)) {
            stbi__rewind(s);
            return 0;
        }
        # 检查包大小是否为8
        if (packet->size != 8) {
            stbi__rewind(s);
            return 0;
        }
    } while (chained);

    # 设置通道数
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

static int stbi__pnm_test(stbi__context * s) {
    // 读取两个字符，检查是否以 'P' 开头，且第二个字符是 '5' 或 '6'
    char p, t;
    p = (char)stbi__get8(s);
    t = (char)stbi__get8(s);
    if (p != 'P' || (t != '5' && t != '6')) {
        // 如果不符合条件，回退读取的字符，并返回 0
        stbi__rewind(s);
        return 0;
    }
    // 符合条件，返回 1
    return 1;
}

static void * stbi__pnm_load(stbi__context * s, int * x, int * y, int * comp, int req_comp, stbi__result_info * ri) {
    stbi_uc * out;
    STBI_NOTUSED(ri);

    // 读取 PNM 图像信息，包括图像宽度、高度、通道数，并计算每通道的位数
    ri->bits_per_channel = stbi__pnm_info(s, (int *)&s->img_x, (int *)&s->img_y, (int *)&s->img_n);
    if (ri->bits_per_channel == 0)
        return 0;

    // 检查图像是否过大
    if (s->img_y > STBI_MAX_DIMENSIONS)
        return stbi__errpuc("too large", "Very large image (corrupt?)");
    if (s->img_x > STBI_MAX_DIMENSIONS)
        return stbi__errpuc("too large", "Very large image (corrupt?)");

    // 设置输出图像的宽度和高度
    *x = s->img_x;
    *y = s->img_y;
    // 设置输出图像的通道数
    if (comp)
        *comp = s->img_n;

    // 检查图像大小是否合法
    if (!stbi__mad4sizes_valid(s->img_n, s->img_x, s->img_y, ri->bits_per_channel / 8, 0))
        return stbi__errpuc("too large", "PNM too large");

    // 分配内存存储图像数据
    out = (stbi_uc *)stbi__malloc_mad4(s->img_n, s->img_x, s->img_y, ri->bits_per_channel / 8, 0);
    if (!out)
        return stbi__errpuc("outofmem", "Out of memory");
    // 读取图像数据
    if (!stbi__getn(s, out, s->img_n * s->img_x * s->img_y * (ri->bits_per_channel / 8))) {
        STBI_FREE(out);
        return stbi__errpuc("bad PNM", "PNM file truncated");
    }
    # 如果请求的通道数不为空且不等于图像的通道数
    if (req_comp && req_comp != s->img_n) {
        # 如果每个通道的位数为16
        if (ri->bits_per_channel == 16) {
            # 将输出数据转换为请求的通道数
            out = (stbi_uc *)stbi__convert_format16((stbi__uint16 *)out, s->img_n, req_comp, s->img_x, s->img_y);
        } else {
            # 将输出数据转换为请求的通道数
            out = stbi__convert_format(out, s->img_n, req_comp, s->img_x, s->img_y);
        }
        # 如果转换失败，则返回空
        if (out == NULL)
            return out; // stbi__convert_format frees input on failure
    }
    # 返回输出数据
    return out;
// 定义一个静态函数，用于判断字符是否为空白字符
static int stbi__pnm_isspace(char c) { return c == ' ' || c == '\t' || c == '\n' || c == '\v' || c == '\f' || c == '\r'; }

// 定义一个静态函数，用于跳过空白字符
static void stbi__pnm_skip_whitespace(stbi__context * s, char * c) {
    for (;;) {
        while (!stbi__at_eof(s) && stbi__pnm_isspace(*c))
            *c = (char)stbi__get8(s);

        if (stbi__at_eof(s) || *c != '#')
            break;

        while (!stbi__at_eof(s) && *c != '\n' && *c != '\r')
            *c = (char)stbi__get8(s);
    }
}

// 定义一个静态函数，用于判断字符是否为数字
static int stbi__pnm_isdigit(char c) { return c >= '0' && c <= '9'; }

// 定义一个静态函数，用于获取整数值
static int stbi__pnm_getinteger(stbi__context * s, char * c) {
    int value = 0;

    while (!stbi__at_eof(s) && stbi__pnm_isdigit(*c)) {
        value = value * 10 + (*c - '0');
        *c = (char)stbi__get8(s);
        if ((value > 214748364) || (value == 214748364 && *c > '7'))
            return stbi__err("integer parse overflow", "Parsing an integer in the PPM header overflowed a 32-bit int");
    }

    return value;
}

// 定义一个静态函数，用于解析 PNM 图像的信息
static int stbi__pnm_info(stbi__context * s, int * x, int * y, int * comp) {
    int maxv, dummy;
    char c, p, t;

    if (!x)
        x = &dummy;
    if (!y)
        y = &dummy;
    if (!comp)
        comp = &dummy;

    stbi__rewind(s);

    // 获取标识符
    p = (char)stbi__get8(s);
    t = (char)stbi__get8(s);
    if (p != 'P' || (t != '5' && t != '6')) {
        stbi__rewind(s);
        return 0;
    }

    *comp = (t == '6') ? 3 : 1; // '5' 表示 1 个分量的 .pgm；'6' 表示 3 个分量的 .ppm

    c = (char)stbi__get8(s);
    stbi__pnm_skip_whitespace(s, &c);

    *x = stbi__pnm_getinteger(s, &c); // 读取宽度
    if (*x == 0)
        return stbi__err("invalid width", "PPM image header had zero or overflowing width");
    stbi__pnm_skip_whitespace(s, &c);

    *y = stbi__pnm_getinteger(s, &c); // 读取高度
    if (*y == 0)
        return stbi__err("invalid width", "PPM image header had zero or overflowing width");
    stbi__pnm_skip_whitespace(s, &c);
    # 从输入流中读取最大值
    maxv = stbi__pnm_getinteger(s, &c); // read max value
    # 如果最大值大于65535，则返回错误信息
    if (maxv > 65535)
        return stbi__err("max value > 65535", "PPM image supports only 8-bit and 16-bit images");
    # 如果最大值大于255，则返回16
    else if (maxv > 255)
        return 16;
    # 否则返回8
    else
        return 8;
// 检查是否为16位 PNM 图像
static int stbi__pnm_is16(stbi__context * s) {
    // 如果图像信息为16位，则返回1，否则返回0
    if (stbi__pnm_info(s, NULL, NULL, NULL) == 16)
        return 1;
    return 0;
}
#endif

// 获取图像信息的主要函数
static int stbi__info_main(stbi__context * s, int * x, int * y, int * comp) {
#ifndef STBI_NO_JPEG
    // 如果是 JPEG 图像，则调用 stbi__jpeg_info 函数获取信息
    if (stbi__jpeg_info(s, x, y, comp))
        return 1;
#endif

#ifndef STBI_NO_PNG
    // 如果是 PNG 图像，则调用 stbi__png_info 函数获取信息
    if (stbi__png_info(s, x, y, comp))
        return 1;
#endif

#ifndef STBI_NO_GIF
    // 如果是 GIF 图像，则调用 stbi__gif_info 函数获取信息
    if (stbi__gif_info(s, x, y, comp))
        return 1;
#endif

#ifndef STBI_NO_BMP
    // 如果是 BMP 图像，则调用 stbi__bmp_info 函数获取信息
    if (stbi__bmp_info(s, x, y, comp))
        return 1;
#endif

#ifndef STBI_NO_PSD
    // 如果是 PSD 图像，则调用 stbi__psd_info 函数获取信息
    if (stbi__psd_info(s, x, y, comp))
        return 1;
#endif

#ifndef STBI_NO_PIC
    // 如果是 PIC 图像，则调用 stbi__pic_info 函数获取信息
    if (stbi__pic_info(s, x, y, comp))
        return 1;
#endif

#ifndef STBI_NO_PNM
    // 如果是 PNM 图像，则调用 stbi__pnm_info 函数获取信息
    if (stbi__pnm_info(s, x, y, comp))
        return 1;
#endif

#ifndef STBI_NO_HDR
    // 如果是 HDR 图像，则调用 stbi__hdr_info 函数获取信息
    if (stbi__hdr_info(s, x, y, comp))
        return 1;
#endif

// 最后检查 TGA 图像，因为它的检测方式比较差
#ifndef STBI_NO_TGA
    // 如果是 TGA 图像，则调用 stbi__tga_info 函数获取信息
    if (stbi__tga_info(s, x, y, comp))
        return 1;
#endif
    // 如果都不是已知类型的图像，则返回错误信息
    return stbi__err("unknown image type", "Image not of any known type, or corrupt");
}

// 检查是否为16位图像的主要函数
static int stbi__is_16_main(stbi__context * s) {
#ifndef STBI_NO_PNG
    // 如果是 PNG 图像，则调用 stbi__png_is16 函数检查是否为16位图像
    if (stbi__png_is16(s))
        return 1;
#endif

#ifndef STBI_NO_PSD
    // 如果是 PSD 图像，则调用 stbi__psd_is16 函数检查是否为16位图像
    if (stbi__psd_is16(s))
        return 1;
#endif

#ifndef STBI_NO_PNM
    // 如果是 PNM 图像，则调用 stbi__pnm_is16 函数检查是否为16位图像
    if (stbi__pnm_is16(s))
        return 1;
#endif
    // 如果不是16位图像，则返回0
    return 0;
}

#ifndef STBI_NO_STDIO
// 从文件中获取图像信息
STBIDEF int stbi_info(char const * filename, int * x, int * y, int * comp) {
    // 打开文件
    FILE * f = stbi__fopen(filename, "rb");
    int result;
    // 如果文件打开失败，则返回错误信息
    if (!f)
        return stbi__err("can't fopen", "Unable to open file");
    // 调用 stbi_info_from_file 函数获取文件信息
    result = stbi_info_from_file(f, x, y, comp);
    // 关闭文件
    fclose(f);
    return result;
}

// 从文件中获取图像信息
STBIDEF int stbi_info_from_file(FILE * f, int * x, int * y, int * comp) {
    int r;
    stbi__context s;
    // 获取当前文件指针位置
    long pos = ftell(f);
    // 初始化图像上下文
    stbi__start_file(&s, f);
    // 调用 stbi__info_main 函数获取图像信息
    r = stbi__info_main(&s, x, y, comp);
    // 将文件指针位置设置回初始位置
    fseek(f, pos, SEEK_SET);
    return r;
}
# 检查文件是否为16位图像
STBIDEF int stbi_is_16_bit(char const * filename) {
    # 打开文件
    FILE * f = stbi__fopen(filename, "rb");
    int result;
    # 如果文件打开失败，返回错误信息
    if (!f)
        return stbi__err("can't fopen", "Unable to open file");
    # 调用函数检查文件是否为16位图像
    result = stbi_is_16_bit_from_file(f);
    # 关闭文件
    fclose(f);
    return result;
}

# 从文件中检查是否为16位图像
STBIDEF int stbi_is_16_bit_from_file(FILE * f) {
    int r;
    stbi__context s;
    # 获取文件当前位置
    long pos = ftell(f);
    # 初始化文件上下文
    stbi__start_file(&s, f);
    # 调用函数检查文件是否为16位图像
    r = stbi__is_16_main(&s);
    # 恢复文件位置
    fseek(f, pos, SEEK_SET);
    return r;
}
#endif // !STBI_NO_STDIO

# 从内存中获取图像信息
STBIDEF int stbi_info_from_memory(stbi_uc const * buffer, int len, int * x, int * y, int * comp) {
    stbi__context s;
    # 初始化内存上下文
    stbi__start_mem(&s, buffer, len);
    # 调用函数获取图像信息
    return stbi__info_main(&s, x, y, comp);
}

# 从回调函数中获取图像信息
STBIDEF int stbi_info_from_callbacks(stbi_io_callbacks const * c, void * user, int * x, int * y, int * comp) {
    stbi__context s;
    # 初始化回调函数上下文
    stbi__start_callbacks(&s, (stbi_io_callbacks *)c, user);
    # 调用函数获取图像信息
    return stbi__info_main(&s, x, y, comp);
}

# 从内存中检查是否为16位图像
STBIDEF int stbi_is_16_bit_from_memory(stbi_uc const * buffer, int len) {
    stbi__context s;
    # 初始化内存上下文
    stbi__start_mem(&s, buffer, len);
    # 调用函数检查是否为16位图像
    return stbi__is_16_main(&s);
}

# 从回调函数中检查是否为16位图像
STBIDEF int stbi_is_16_bit_from_callbacks(stbi_io_callbacks const * c, void * user) {
    stbi__context s;
    # 初始化回调函数上下文
    stbi__start_callbacks(&s, (stbi_io_callbacks *)c, user);
    # 调用函数检查是否为16位图像
    return stbi__is_16_main(&s);
}

#endif // STB_IMAGE_IMPLEMENTATION
# 版权声明和许可证声明，规定了软件的使用条件
# 版权声明和许可证声明应包含在所有软件副本或实质部分中
# 软件按"原样"提供，不提供任何形式的保证，包括但不限于对适销性、特定用途的适用性和非侵权的保证
# 作者或版权持有人不对任何索赔、损害或其他责任负责，无论是合同诉讼、侵权行为还是其他情况，由软件引起或与之相关
# 可选方案B - 公共领域（www.unlicense.org）
# 这是自由的、不受限制的软件，放入公共领域
# 任何人都可以自由复制、修改、发布、使用、编译、出售或分发这个软件，无论是以源代码形式还是编译后的二进制形式，无论是商业用途还是非商业用途，以任何方式
# 在承认版权法的司法管辖区，软件的作者或作者将软件的任何和所有版权利益奉献给公共领域
# 我们做出这一奉献是为了造福广大公众，对我们的继承人和后继者造成损害。我们打算将这一奉献作为对版权法下对该软件的所有现有和未来权利的永久放弃的公开行为
# 软件按"原样"提供，不提供任何形式的保证，包括但不限于对适销性、特定用途的适用性和非侵权的保证
# 作者不对任何索赔、损害或其他责任负责，无论是合同诉讼、侵权行为还是其他情况，由软件引起或与之相关
------------------------------------------------------------------------------

这是一个分隔符，用于标记代码块的开始。
```