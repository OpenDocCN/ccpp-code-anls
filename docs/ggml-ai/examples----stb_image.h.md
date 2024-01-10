# `ggml\examples\stb_image.h`

```
"""
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
"""
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
    # 贡献者列表，包括他们的贡献和联系方式
    Jean-Marc Lienher (gif)                贡献了 GIF 相关部分
    Ben "Disch" Wenger (io callbacks)      贡献了 IO 回调部分
    Tom Seddon (pic)                       贡献了 PIC 相关部分
    Omar Cornut (1/2/4-bit PNG)            贡献了 1/2/4 位 PNG 相关部分
    Thatcher Ulrich (psd)                  贡献了 PSD 相关部分
    Nicolas Guillemot (vertical flip)      贡献了垂直翻转部分
    Ken Miller (pgm, ppm)                  贡献了 PGM、PPM 相关部分
    Richard Mitton (16-bit PSD)            贡献了 16 位 PSD 相关部分
    github:urraka (animated gif)           贡献了动画 GIF 相关部分
    Junggon Kim (PNM comments)             贡献了 PNM 注释部分
    Christopher Forseth (animated gif)     贡献了动画 GIF 相关部分
    Daniel Gibson (16-bit TGA)             贡献了 16 位 TGA 相关部分
    socks-the-fox (16-bit PNG)             贡献了 16 位 PNG 相关部分
    Jeremy Sawicki (handle all ImageNet JPGs) 贡献了处理所有 ImageNet JPGs 相关部分
    Optimizations & bugfixes               优化和错误修复
    Fabian "ryg" Giesen                    优化和错误修复
    Arseny Kapoulkine                      优化和错误修复
    John-Mark Allen                        优化和错误修复
    Carmelo J Fdez-Aguera                  优化和错误修复
    Bug & warning fixes                    错误和警告修复
    Marc LeBlanc                           错误和警告修复
    David Woo                              错误和警告修复
    Guillaume George                       错误和警告修复
    Martins Mozeiko                        错误和警告修复
    Christpher Lloyd                       错误和警告修复
    Jerry Jansson                          错误和警告修复
    Joseph Thomson                         错误和警告修复
    Blazej Dariusz Roszkowski              错误和警告修复
    Phil Jordan                            错误和警告修复
    Dave Moore                             错误和警告修复
    Roy Eltham                             错误和警告修复
    Hayaki Saito                           错误和警告修复
    Nathan Reed                            错误和警告修复
    Won Chun                               错误和警告修复
    Luke Graham                            错误和警告修复
    Johan Duparc                          错误和警告修复
    Nick Verigakis                        错误和警告修复
    the Horde3D community                错误和警告修复
    Thomas Ruf                            错误和警告修复
    Ronny Chevalier                       错误和警告修复
    github:rlyeh                          错误和警告修复
    Janez Zemva                           错误和警告修复
    John Bartholomew                      错误和警告修复
    Michal Cichon                        错误和警告修复
    github:romigrou                      错误和警告修复
    Jonathan Blow                        错误和警告修复
    Ken Hamada                           错误和警告修复
    Tero Hanninen                        错误和警告修复
    github:svdijk                        错误和警告修复
    Eugene Golushkov                     错误和警告修复
    Laurent Gomila                       错误和警告修复
    Cort Stratton                        错误和警告修复
    github:snagar                        错误和警告修复
    Aruelien Pocheville                  错误和警告修复
    Sergio Gonzalez                      错误和警告修复
    Thibault Reuille                     错误和警告修复
    github:Zelex                         错误和警告修复
    Cass Everitt                         错误和警告修复
    Ryamond Barbiero                     错误和警告修复
    github:grim210                       错误和警告修复
    Paul Du Bois                         错误和警告修复
    Engin Manap                          错误和警告修复
    Aldo Culquicondor                    错误和警告修复
    github:sammyhw                       错误和警告修复
    Philipp Wiesemann                    错误和警告修复
    Dale Weiler                          错误和警告修复
    Oriol Ferrer Mesia                   错误和警告修复
    github:phprus                        错误和警告修复
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
#ifndef STBI_INCLUDE_STB_IMAGE_H
#define STBI_INCLUDE_STB_IMAGE_H

// 定义条件编译，防止头文件重复包含

// DOCUMENTATION
//
// 限制：
//    - 没有12位每通道的JPEG
//    - 没有使用算术编码的JPEG
//    - GIF总是返回*comp=4
//
// 基本用法（有关HDR用法，请参阅下面的HDR讨论）：
//    int x,y,n;
//    unsigned char *data = stbi_load(filename, &x, &y, &n, 0);
//    // ... 如果数据不为空，则处理数据 ...
//    // ... x = 宽度，y = 高度，n = 每像素8位组件的数量 ...
//    // ... 用'0'替换为'1'..'4'以强制每像素有指定数量的组件
//    // ... 但'n'将始终是如果你说0时的数量
//    stbi_image_free(data);
//
// 标准参数：
//    int *x                 -- 输出图像的宽度（像素）
//    int *y                 -- 输出图像的高度（像素）
//    int *channels_in_file  -- 输出图像文件中的图像组件数量
//    int desired_channels   -- 如果非零，则请求结果中的图像组件数量
//
// 图像加载器的返回值是一个'unsigned char *'，指向像素数据，如果分配失败或图像损坏或无效，则返回NULL。像素数据由*y个扫描线组成，每个扫描线有*x个像素，每个像素由N个交错的8位组件组成；指向的第一个像素是图像中最左上角的像素。无论格式如何，图像扫描线之间或像素之间都没有填充。如果desired_channels为非零，则组件数量N为'desired_channels'，否则为*channels_in_file。如果desired_channels为非零，则*channels_in_file具有否则将输出的组件数量。例如，如果将desired_channels设置为4，则将始终获得RGBA输出，但您可以检查*channels_in_file，以查看它是否是平凡不透明的，因为例如源图像中只有3个通道。
//
// 具有N个组件的输出图像，每个像素中的组件按以下顺序交错：
//
// 定义图像的颜色通道数和对应的颜色组合
// 1 表示灰度
// 2 表示灰度和透明度
// 3 表示红色、绿色、蓝色
// 4 表示红色、绿色、蓝色、透明度
//
// 如果由于任何原因图像加载失败，返回值将为 NULL，
// *x、*y、*channels_in_file 将保持不变。可以使用 stbi_failure_reason() 函数查询加载失败的原因。
// 定义 STBI_NO_FAILURE_STRINGS 可以避免编译这些字符串，定义 STBI_FAILURE_USERMSG 可以获得稍微用户友好一些的消息。
//
// 调色板 PNG、BMP、GIF 和 PIC 图像会自动去除调色板。
//
// 要查询图像的宽度、高度和颜色通道数，而不必解码整个文件，可以使用 stbi_info 系列函数：
//
//   int x, y, n, ok;
//   ok = stbi_info(filename, &x, &y, &n);
//   // 如果图像是支持的格式，返回 ok=1 并设置 x、y、n，否则返回 0。
//
// 注意，stb_image 在其公共 API 中广泛使用 int 类型表示大小，包括内存缓冲区的大小。这已经成为 API 的一部分，因此很难在不引起破坏的情况下进行更改。因此，各种图像加载器对图像大小都有一定的限制；这些限制在格式上有所不同，但通常都是接近 2GB 或接近 1GB。当解码后的图像大于这个大小时，stb_image 解码将失败。
//
// 另外，stb_image 将拒绝具有任何维度大于可配置的 STBI_MAX_DIMENSIONS 的图像文件，默认为 2**24 = 16777216 像素。由于上述内存限制，使得具有这些尺寸的图像正确加载的唯一方法是具有极端的长宽比。无论如何，这里的假设是这样的较大图像可能是畸形的或恶意的。如果确实需要加载具有单个维度的较大图像
// 如果定义了 STBI_MAX_DIMENSIONS 并且其值大于这个限制，但仍然符合整体大小限制，你可以自己定义 STBI_MAX_DIMENSIONS 为更大的值。
//
// ===========================================================================
//
// UNICODE:
//
//   如果在 Windows 上编译并且希望使用 Unicode 文件名，编译时使用
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
// 有时我会让 "良好的性能" 在 "易于维护" 之上，为了获得更好的性能，我可能会提供一些不太容易使用的 API，以获得更高的性能，除了易于使用的 API。尽管如此，重要的是要记住，从你作为这个库的客户的角度来看，你关心的只有 #1 和 #3，而 stb 库并不强调 #3 胜过一切。
//
// 一些次要的优先级直接源自前两个，其中一些提供了更明确的原因，说明为什么性能不能被强调。
//
//    - 可移植性（"易于使用"）
//    - 小的源代码占用空间（"易于维护"）
//    - 无依赖性（"易于使用"）
//
// ===========================================================================
//
// I/O 回调
//
// I/O 回调允许你从任意来源读取数据，比如打包文件或其他来源。通过一个小的内部缓冲区（目前为 128 字节）处理从回调读取的数据，以尝试减少开销。
//
// 你必须定义的三个函数是 "read"（读取一些字节的数据），"skip"（跳过一些字节的数据），"eof"（报告流是否已结束）。
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
//   - 如果你使用 STBI_NO_PNG（或者 _ONLY_ 没有 PNG），但仍然希望 zlib 解码器可用，请定义 STBI_SUPPORT_ZLIB
//
//  - 如果你定义了 STBI_MAX_DIMENSIONS，stb_image 将拒绝大于该尺寸的图像（宽度或高度），不进行进一步处理。
//    这是为了让程序在野外设置一个上限，以防止拒绝服务攻击未经信任的数据，因为可以生成一个巨大尺寸的有效图像，
//    强制 stb_image 分配一个巨大的内存块，并花费不成比例的时间来解码它。默认情况下，这被设置为（1 << 24），
//    即 16777216，但这仍然非常大。

#ifndef STBI_NO_STDIO
#include <stdio.h>
#endif // STBI_NO_STDIO

#define STBI_VERSION 1

enum
{
   STBI_default = 0, // 仅用于期望的通道

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
// 主要 API - 适用于任何类型的图像
//

//
// 通过文件名、打开文件或内存缓冲区加载图像
//

typedef struct
{
   int      (*read)  (void *user,char *data,int size);   // 用 'size' 字节填充 'data'。返回实际读取的字节数
   void     (*skip)  (void *user,int n);                 // 跳过接下来的 'n' 字节，或者如果是负数，则 'unget' 最后 -n 字节
   int      (*eof)   (void *user);                       // 如果我们在文件/数据的末尾，则返回非零值
} stbi_io_callbacks;

////////////////////////////////////
//
// 每通道 8 位接口
//

STBIDEF stbi_uc *stbi_load_from_memory   (stbi_uc           const *buffer, int len   , int *x, int *y, int *channels_in_file, int desired_channels);
// 通过回调函数从内存中加载图像数据，返回图像数据指针
STBIDEF stbi_uc *stbi_load_from_callbacks(stbi_io_callbacks const *clbk  , void *user, int *x, int *y, int *channels_in_file, int desired_channels);

// 如果没有禁用标准输入输出，则通过文件名加载图像数据，返回图像数据指针
#ifndef STBI_NO_STDIO
STBIDEF stbi_uc *stbi_load            (char const *filename, int *x, int *y, int *channels_in_file, int desired_channels);
// 通过文件指针加载图像数据，返回图像数据指针，文件指针在图像后面
STBIDEF stbi_uc *stbi_load_from_file  (FILE *f, int *x, int *y, int *channels_in_file, int desired_channels);
#endif

// 如果没有禁用 GIF 格式，则从内存中加载 GIF 图像数据，返回图像数据指针
#ifndef STBI_NO_GIF
STBIDEF stbi_uc *stbi_load_gif_from_memory(stbi_uc const *buffer, int len, int **delays, int *x, int *y, int *z, int *comp, int req_comp);
#endif

// 如果定义了 STBI_WINDOWS_UTF8，则将宽字符转换为 UTF-8 字符
#ifdef STBI_WINDOWS_UTF8
STBIDEF int stbi_convert_wchar_to_utf8(char *buffer, size_t bufferlen, const wchar_t* input);
#endif

////////////////////////////////////
//
// 16位每通道接口
//

// 从内存中加载 16 位每通道图像数据，返回图像数据指针
STBIDEF stbi_us *stbi_load_16_from_memory   (stbi_uc const *buffer, int len, int *x, int *y, int *channels_in_file, int desired_channels);
// 通过回调函数从内存中加载 16 位每通道图像数据，返回图像数据指针
STBIDEF stbi_us *stbi_load_16_from_callbacks(stbi_io_callbacks const *clbk, void *user, int *x, int *y, int *channels_in_file, int desired_channels);

// 如果没有禁用标准输入输出，则通过文件名加载 16 位每通道图像数据，返回图像数据指针
#ifndef STBI_NO_STDIO
STBIDEF stbi_us *stbi_load_16          (char const *filename, int *x, int *y, int *channels_in_file, int desired_channels);
// 通过文件指针加载 16 位每通道图像数据，返回图像数据指针
STBIDEF stbi_us *stbi_load_from_file_16(FILE *f, int *x, int *y, int *channels_in_file, int desired_channels);
#endif

////////////////////////////////////
//
// 浮点数每通道接口
//
#ifndef STBI_NO_LINEAR
   // 从内存加载图像数据，并以浮点数形式返回，同时返回图像的宽、高、通道数和期望的通道数
   STBIDEF float *stbi_loadf_from_memory     (stbi_uc const *buffer, int len, int *x, int *y, int *channels_in_file, int desired_channels);
   // 从回调函数加载图像数据，并以浮点数形式返回，同时返回图像的宽、高、通道数和期望的通道数
   STBIDEF float *stbi_loadf_from_callbacks  (stbi_io_callbacks const *clbk, void *user, int *x, int *y,  int *channels_in_file, int desired_channels);

   #ifndef STBI_NO_STDIO
   // 从文件加载图像数据，并以浮点数形式返回，同时返回图像的宽、高、通道数和期望的通道数
   STBIDEF float *stbi_loadf            (char const *filename, int *x, int *y, int *channels_in_file, int desired_channels);
   // 从文件指针加载图像数据，并以浮点数形式返回，同时返回图像的宽、高、通道数和期望的通道数
   STBIDEF float *stbi_loadf_from_file  (FILE *f, int *x, int *y, int *channels_in_file, int desired_channels);
   #endif
#endif

#ifndef STBI_NO_HDR
   // 将 HDR 图像转换为 LDR 图像的 gamma 校正
   STBIDEF void   stbi_hdr_to_ldr_gamma(float gamma);
   // 将 HDR 图像转换为 LDR 图像的缩放
   STBIDEF void   stbi_hdr_to_ldr_scale(float scale);
#endif // STBI_NO_HDR

#ifndef STBI_NO_LINEAR
   // 将 LDR 图像转换为 HDR 图像的 gamma 校正
   STBIDEF void   stbi_ldr_to_hdr_gamma(float gamma);
   // 将 LDR 图像转换为 HDR 图像的缩放
   STBIDEF void   stbi_ldr_to_hdr_scale(float scale);
#endif // STBI_NO_LINEAR

// stbi_is_hdr 始终被定义，但如果 STBI_NO_HDR 被定义，则始终返回 false
STBIDEF int    stbi_is_hdr_from_callbacks(stbi_io_callbacks const *clbk, void *user);
STBIDEF int    stbi_is_hdr_from_memory(stbi_uc const *buffer, int len);
#ifndef STBI_NO_STDIO
// 检查文件是否为 HDR 格式
STBIDEF int      stbi_is_hdr          (char const *filename);
// 从文件指针检查文件是否为 HDR 格式
STBIDEF int      stbi_is_hdr_from_file(FILE *f);
#endif // STBI_NO_STDIO


// 获取加载失败的简要原因
// 在大多数编译器（以及所有现代主流编译器）上，这是线程安全的
STBIDEF const char *stbi_failure_reason  (void);

// 释放加载的图像数据，这只是调用 free()
STBIDEF void     stbi_image_free      (void *retval_from_stbi_load);

// 获取图像的尺寸和通道数，而不完全解码图像
STBIDEF int      stbi_info_from_memory(stbi_uc const *buffer, int len, int *x, int *y, int *comp);
STBIDEF int      stbi_info_from_callbacks(stbi_io_callbacks const *clbk, void *user, int *x, int *y, int *comp);
STBIDEF int      stbi_is_16_bit_from_memory(stbi_uc const *buffer, int len);
# 从回调函数中判断图像是否为16位
STBIDEF int      stbi_is_16_bit_from_callbacks(stbi_io_callbacks const *clbk, void *user);

# 如果没有禁用标准输入输出，则获取图像信息
STBIDEF int      stbi_info               (char const *filename,     int *x, int *y, int *comp);
STBIDEF int      stbi_info_from_file     (FILE *f,                  int *x, int *y, int *comp);
STBIDEF int      stbi_is_16_bit          (char const *filename);
STBIDEF int      stbi_is_16_bit_from_file(FILE *f);

# 设置是否在加载时解除预乘
STBIDEF void stbi_set_unpremultiply_on_load(int flag_true_if_should_unpremultiply);

# 设置是否将 iPhone 图像转换为 RGB 格式
STBIDEF void stbi_convert_iphone_png_to_rgb(int flag_true_if_should_convert);

# 设置是否在加载时垂直翻转图像
STBIDEF void stbi_set_flip_vertically_on_load(int flag_true_if_should_flip);

# 与上述功能相同，但仅适用于调用该函数的线程加载的图像
# 如果编译器支持线程局部变量，则此函数仅在调用时可用
STBIDEF void stbi_set_unpremultiply_on_load_thread(int flag_true_if_should_unpremultiply);
STBIDEF void stbi_convert_iphone_png_to_rgb_thread(int flag_true_if_should_convert);
STBIDEF void stbi_set_flip_vertically_on_load_thread(int flag_true_if_should_flip);

# ZLIB 客户端 - 由 PNG 使用，也可用于其他目的
STBIDEF char *stbi_zlib_decode_malloc_guesssize(const char *buffer, int len, int initial_size, int *outlen);
STBIDEF char *stbi_zlib_decode_malloc_guesssize_headerflag(const char *buffer, int len, int initial_size, int *outlen, int parse_header);
STBIDEF char *stbi_zlib_decode_malloc(const char *buffer, int len, int *outlen);
// 定义了一个函数，用于解码经过 zlib 压缩的数据到指定的输出缓冲区
STBIDEF int   stbi_zlib_decode_buffer(char *obuffer, int olen, const char *ibuffer, int ilen);

// 定义了一个函数，用于解码经过 zlib 压缩的数据到动态分配的内存中
STBIDEF char *stbi_zlib_decode_noheader_malloc(const char *buffer, int len, int *outlen);
// 定义了一个函数，用于解码经过 zlib 压缩的数据到指定的输出缓冲区
STBIDEF int   stbi_zlib_decode_noheader_buffer(char *obuffer, int olen, const char *ibuffer, int ilen);

// 如果是 C++ 环境，则结束 extern "C" 块
#ifdef __cplusplus
}
#endif

//
//
////   end header file   /////////////////////////////////////////////////////
#endif // STBI_INCLUDE_STB_IMAGE_H

// 如果定义了 STB_IMAGE_IMPLEMENTATION，则执行下面的代码块
#ifdef STB_IMAGE_IMPLEMENTATION

// 如果定义了 STBI_ONLY_JPEG 或其他格式的限制，则进行相应的宏定义
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

// 如果定义了 STBI_NO_PNG 且未定义 STBI_SUPPORT_ZLIB 且未定义 STBI_NO_ZLIB，则定义 STBI_NO_ZLIB
#if defined(STBI_NO_PNG) && !defined(STBI_SUPPORT_ZLIB) && !defined(STBI_NO_ZLIB)
#define STBI_NO_ZLIB
#endif

// 包含标准头文件
#include <stdarg.h>
#include <stddef.h> // ptrdiff_t on osx
#include <stdlib.h>
#include <string.h>
#include <limits.h>

// 如果未定义 STBI_NO_LINEAR 或未定义 STBI_NO_HDR，则包含 math.h 头文件
#if !defined(STBI_NO_LINEAR) || !defined(STBI_NO_HDR)
#include <math.h>  // ldexp, pow
#endif

// 如果未定义 STBI_NO_STDIO，则包含 stdio.h 头文件
#ifndef STBI_NO_STDIO
#include <stdio.h>
#endif

// 如果未定义 STBI_ASSERT，则包含 assert.h 头文件，并定义 STBI_ASSERT 宏
#ifndef STBI_ASSERT
#include <assert.h>
#define STBI_ASSERT(x) assert(x)
#endif

// 如果是 C++ 环境，则定义 STBI_EXTERN 为 extern "C"，否则为 extern
#ifdef __cplusplus
#define STBI_EXTERN extern "C"
#else
#define STBI_EXTERN extern
#endif

// 如果不是 MSC 编译器，则根据是否是 C++ 环境定义 stbi_inline 宏
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

定义了 STBI_THREAD_LOCAL 宏，根据不同的编译器和标准，选择不同的线程局部存储修饰符。


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

根据不同的编译器和平台，定义了不同大小的无符号和有符号整型数据类型。


// should produce compiler error if size is wrong
typedef unsigned char validate_uint32[sizeof(stbi__uint32)==4 ? 1 : -1];

如果 stbi__uint32 的大小不是 4 个字节，应该产生编译错误。


#ifdef _MSC_VER
#define STBI_NOTUSED(v)  (void)(v)
#else
#define STBI_NOTUSED(v)  (void)sizeof(v)
#endif

根据不同的编译器，定义了一个宏，用于标记变量未使用。


#ifdef _MSC_VER
#define STBI_HAS_LROTL
#endif

#ifdef STBI_HAS_LROTL
   #define stbi_lrot(x,y)  _lrotl(x,y)
#else
   #define stbi_lrot(x,y)  (((x) << (y)) | ((x) >> (-(y) & 31)))
#endif

根据编译器是否支持 _lrotl 函数，定义了 stbi_lrot 宏，用于实现循环左移操作。


#if defined(STBI_MALLOC) && defined(STBI_FREE) && (defined(STBI_REALLOC) || defined(STBI_REALLOC_SIZED))
// ok
#elif !defined(STBI_MALLOC) && !defined(STBI_FREE) && !defined(STBI_REALLOC) && !defined(STBI_REALLOC_SIZED)
// ok
#else
#error "Must define all or none of STBI_MALLOC, STBI_FREE, and STBI_REALLOC (or STBI_REALLOC_SIZED)."
#endif

检查是否同时定义了 STBI_MALLOC、STBI_FREE 和 STBI_REALLOC（或 STBI_REALLOC_SIZED），如果没有同时定义或者同时定义了其中一部分，则产生编译错误。


#ifndef STBI_MALLOC
#define STBI_MALLOC(sz)           malloc(sz)
#define STBI_REALLOC(p,newsz)     realloc(p,newsz)
#define STBI_FREE(p)              free(p)
#endif

如果没有定义 STBI_MALLOC，则定义 STBI_MALLOC、STBI_REALLOC 和 STBI_FREE 为标准库的内存分配、重新分配和释放函数。


#ifndef STBI_REALLOC_SIZED
#define STBI_REALLOC_SIZED(p,oldsz,newsz) STBI_REALLOC(p,newsz)
#endif

如果没有定义 STBI_REALLOC_SIZED，则定义 STBI_REALLOC_SIZED 为 STBI_REALLOC。
// 检测当前平台是否为 x86/x64
#if defined(__x86_64__) || defined(_M_X64)
#define STBI__X64_TARGET
#elif defined(__i386) || defined(_M_IX86)
#define STBI__X86_TARGET
#endif

#if defined(__GNUC__) && defined(STBI__X86_TARGET) && !defined(__SSE2__) && !defined(STBI_NO_SIMD)
// 如果是 GCC 编译器，并且目标平台是 x86，并且没有启用 SSE2，并且没有禁用 SIMD
// gcc 不支持 SSE2 内联函数，除非使用 -msse2 编译选项，这意味着它可以在任何地方使用 SSE2。这很不幸，
// 但以前尝试在运行时提供 SSE2 函数时导致了许多问题。GCC/Clang 中架构扩展的暴露方式，很遗憾，实际上不太适合单文件库。
// 新行为：如果使用 -msse2 编译，我们将在没有任何检测的情况下使用 SSE2；如果没有，我们根本不使用它。
#define STBI_NO_SIMD
#endif

#if defined(__MINGW32__) && defined(STBI__X86_TARGET) && !defined(STBI_MINGW_ENABLE_SSE2) && !defined(STBI_NO_SIMD)
// 注意，__MINGW32__ 实际上并不意味着 32 位，所以我们必须避免 STBI__X64_TARGET
//
// 32 位 MinGW 希望 ESP 要 16 字节对齐，但这不在 Windows ABI 中，VC++ 以及 Windows DLLs 不保持这个不变。
// 因此，在 32 位 MinGW 上启用 SSE2 在不同时启用 "-mstackrealign" 是危险的。
// 有关更多信息，请参阅 https://github.com/nothings/stb/issues/81。
//
// 因此，默认情况下在 32 位 MinGW 上不启用 SSE2。如果你已经阅读到这里并在构建设置中添加了 -mstackrealign，请随意 #define STBI_MINGW_ENABLE_SSE2。
#define STBI_NO_SIMD
#endif

#if !defined(STBI_NO_SIMD) && (defined(STBI__X86_TARGET) || defined(STBI__X64_TARGET))
#define STBI_SSE2
#include <emmintrin.h>

#ifdef _MSC_VER

#if _MSC_VER >= 1400  // not VC6
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
// 定义一个宏，用于指定变量的内存对齐方式
#define STBI_SIMD_ALIGN(type, name) __declspec(align(16)) type name

// 如果不是 VC++ 编译器，则使用 GCC 风格的内存对齐方式
#else // assume GCC-style if not VC++
#define STBI_SIMD_ALIGN(type, name) type name __attribute__((aligned(16)))

// 如果不定义 STBI_NO_JPEG 且定义了 STBI_SSE2，则定义一个函数来检查是否支持 SSE2
#if !defined(STBI_NO_JPEG) && defined(STBI_SSE2)
static int stbi__sse2_available(void)
{
   // 获取 CPU 的信息，判断是否支持 SSE2
   int info3 = stbi__cpuid3();
   return ((info3 >> 26) & 1) != 0;
}
#endif

// 如果定义了 STBI_NEON，则包含 ARM NEON 相关的头文件
#ifdef STBI_NEON
#include <arm_neon.h>
// 根据不同的编译器，定义不同的内存对齐方式
#ifdef _MSC_VER
#define STBI_SIMD_ALIGN(type, name) __declspec(align(16)) type name
#else
#define STBI_SIMD_ALIGN(type, name) type name __attribute__((aligned(16)))
#endif
#endif

// 如果没有定义 STBI_SIMD_ALIGN，则使用默认的内存对齐方式
#ifndef STBI_SIMD_ALIGN
#define STBI_SIMD_ALIGN(type, name) type name
#endif

// 如果没有定义 STBI_MAX_DIMENSIONS，则定义一个默认的最大尺寸
#ifndef STBI_MAX_DIMENSIONS
#define STBI_MAX_DIMENSIONS (1 << 24)
#endif

// stbi__context 结构体和 start_xxx 函数
// stbi__context 结构体是所有图像使用的基本上下文，包含所有 IO 上下文，以及一些基本的图像信息
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

// 重新填充缓冲区的函数
static void stbi__refill_buffer(stbi__context *s);

// 初始化一个内存解码上下文
static void stbi__start_mem(stbi__context *s, stbi_uc const *buffer, int len)
{
   // 将指向读取函数的指针设置为 NULL
   s->io.read = NULL;
   // 将读取标志位设置为 0
   s->read_from_callbacks = 0;
   // 将回调已读标志位设置为 0
   s->callback_already_read = 0;
   // 将图像缓冲区指针设置为传入的缓冲区起始地址
   s->img_buffer = s->img_buffer_original = (stbi_uc *) buffer;
   // 将图像缓冲区结束指针设置为传入的缓冲区结束地址
   s->img_buffer_end = s->img_buffer_original_end = (stbi_uc *) buffer+len;
}

// 初始化基于回调的上下文
static void stbi__start_callbacks(stbi__context *s, stbi_io_callbacks *c, void *user)
{
   // 将回调函数和用户数据拷贝到上下文中
   s->io = *c;
   s->io_user_data = user;
   // 设置缓冲区长度为初始缓冲区的大小
   s->buflen = sizeof(s->buffer_start);
   // 将读取标志位设置为 1
   s->read_from_callbacks = 1;
   // 将回调已读标志位设置为 0
   s->callback_already_read = 0;
   // 将图像缓冲区指针设置为初始缓冲区的起始地址
   s->img_buffer = s->img_buffer_original = s->buffer_start;
   // 重新填充缓冲区
   stbi__refill_buffer(s);
   // 将图像缓冲区结束指针设置为初始缓冲区的结束地址
   s->img_buffer_original_end = s->img_buffer_end;
}

#ifndef STBI_NO_STDIO

static int stbi__stdio_read(void *user, char *data, int size)
{
   return (int) fread(data,1,size,(FILE*) user);
}

static void stbi__stdio_skip(void *user, int n)
{
   int ch;
   // 移动文件指针到相对当前位置的偏移量
   fseek((FILE*) user, n, SEEK_CUR);
   // 读取一个字节以重置 feof() 的标志位
   ch = fgetc((FILE*) user);
   if (ch != EOF) {
      // 如果读取成功，将字节推回流中
      ungetc(ch, (FILE *) user);
   }
}

static int stbi__stdio_eof(void *user)
{
   // 检查文件流是否到达文件末尾或者发生错误
   return feof((FILE*) user) || ferror((FILE *) user);
}

// 定义标准 I/O 回调函数
static stbi_io_callbacks stbi__stdio_callbacks =
{
   stbi__stdio_read,
   stbi__stdio_skip,
   stbi__stdio_eof,
};

static void stbi__start_file(stbi__context *s, FILE *f)
{
   // 使用标准 I/O 回调函数初始化上下文
   stbi__start_callbacks(s, &stbi__stdio_callbacks, (void *) f);
}

//static void stop_file(stbi__context *s) { }

#endif // !STBI_NO_STDIO

static void stbi__rewind(stbi__context *s)
{
   // 重新定位图像缓冲区指针到初始位置
   s->img_buffer = s->img_buffer_original;
   // 重新定位图像缓冲区结束指针到初始位置
   s->img_buffer_end = s->img_buffer_original_end;
}

enum
{
   // RGB 顺序
   STBI_ORDER_RGB,
   // BGR 顺序
   STBI_ORDER_BGR
};

typedef struct
{
   // 每个通道的位数
   int bits_per_channel;
   // 通道数
   int num_channels;
   // 通道顺序
   int channel_order;
#ifndef STBI_NO_JPEG
// 检测是否为 JPEG 格式的图像
static int      stbi__jpeg_test(stbi__context *s);
// 加载 JPEG 格式的图像数据
static void    *stbi__jpeg_load(stbi__context *s, int *x, int *y, int *comp, int req_comp, stbi__result_info *ri);
// 获取 JPEG 格式的图像信息
static int      stbi__jpeg_info(stbi__context *s, int *x, int *y, int *comp);
#endif

#ifndef STBI_NO_PNG
// 检测是否为 PNG 格式的图像
static int      stbi__png_test(stbi__context *s);
// 加载 PNG 格式的图像数据
static void    *stbi__png_load(stbi__context *s, int *x, int *y, int *comp, int req_comp, stbi__result_info *ri);
// 获取 PNG 格式的图像信息
static int      stbi__png_info(stbi__context *s, int *x, int *y, int *comp);
// 检测 PNG 图像是否为16位
static int      stbi__png_is16(stbi__context *s);
#endif

#ifndef STBI_NO_BMP
// 检测是否为 BMP 格式的图像
static int      stbi__bmp_test(stbi__context *s);
// 加载 BMP 格式的图像数据
static void    *stbi__bmp_load(stbi__context *s, int *x, int *y, int *comp, int req_comp, stbi__result_info *ri);
// 获取 BMP 格式的图像信息
static int      stbi__bmp_info(stbi__context *s, int *x, int *y, int *comp);
#endif

#ifndef STBI_NO_TGA
// 检测是否为 TGA 格式的图像
static int      stbi__tga_test(stbi__context *s);
// 加载 TGA 格式的图像数据
static void    *stbi__tga_load(stbi__context *s, int *x, int *y, int *comp, int req_comp, stbi__result_info *ri);
// 获取 TGA 格式的图像信息
static int      stbi__tga_info(stbi__context *s, int *x, int *y, int *comp);
#endif

#ifndef STBI_NO_PSD
// 检测是否为 PSD 格式的图像
static int      stbi__psd_test(stbi__context *s);
// 加载 PSD 格式的图像数据
static void    *stbi__psd_load(stbi__context *s, int *x, int *y, int *comp, int req_comp, stbi__result_info *ri, int bpc);
// 获取 PSD 格式的图像信息
static int      stbi__psd_info(stbi__context *s, int *x, int *y, int *comp);
// 检测 PSD 图像是否为16位
static int      stbi__psd_is16(stbi__context *s);
#endif

#ifndef STBI_NO_HDR
// 检测是否为 HDR 格式的图像
static int      stbi__hdr_test(stbi__context *s);
// 加载 HDR 格式的图像数据
static float   *stbi__hdr_load(stbi__context *s, int *x, int *y, int *comp, int req_comp, stbi__result_info *ri);
// 获取 HDR 格式的图像信息
static int      stbi__hdr_info(stbi__context *s, int *x, int *y, int *comp);
#endif

#ifndef STBI_NO_PIC
// 检测是否为 PIC 格式的图像
static int      stbi__pic_test(stbi__context *s);
// 加载 PIC 格式的图像数据
static void    *stbi__pic_load(stbi__context *s, int *x, int *y, int *comp, int req_comp, stbi__result_info *ri);
// 检查给定的两个整数相加是否会导致溢出，如果不会溢出返回1，否则返回0
// 负数被视为无效
static int stbi__addsizes_valid(int a, int b)
{
   // 如果 b 小于 0，则返回 0
   if (b < 0) return 0;
   // 现在 0 <= b <= INT_MAX，因此也有 0 <= INT_MAX - b <= INTMAX。
   // 并且 "a + b <= INT_MAX"（可能会溢出）与 a <= INT_MAX - b（不会溢出）是相同的
   return a <= INT_MAX - b;
}

// 如果乘积有效，则返回 1，溢出则返回 0。负因子被视为无效。
static int stbi__mul2sizes_valid(int a, int b)
{
   // 如果 a 或 b 小于 0，则返回 0
   if (a < 0 || b < 0) return 0;
   // 如果 b 等于 0，则返回 1（乘以 0 总是安全的）
   if (b == 0) return 1;
   // 用可移植的方式检查 a*b 是否没有溢出
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
   // 如果 "a*b + add" 无效，则返回 NULL
   if (!stbi__mad2sizes_valid(a, b, add)) return NULL;
   return stbi__malloc(a*b + add);
}
#endif

// 使用大小溢出检查的 malloc
static void *stbi__malloc_mad3(int a, int b, int c, int add)
{
   // 如果 "a*b*c + add" 无效，则返回 NULL
   if (!stbi__mad3sizes_valid(a, b, c, add)) return NULL;
   return stbi__malloc(a*b*c + add);
}
#if !defined(STBI_NO_LINEAR) || !defined(STBI_NO_HDR) || !defined(STBI_NO_PNM)
# 如果未定义 STBI_NO_LINEAR 或 STBI_NO_HDR 或 STBI_NO_PNM，则定义 stbi__malloc_mad4 函数
static void *stbi__malloc_mad4(int a, int b, int c, int d, int add)
{
   # 如果给定的参数不会导致溢出，则分配内存并返回指针
   if (!stbi__mad4sizes_valid(a, b, c, d, add)) return NULL;
   return stbi__malloc(a*b*c*d + add);
}
#endif

// 返回 1 如果两个有符号整数的和有效（在 -2^31 和 2^31-1 之间，包括边界），溢出时返回 0
static int stbi__addints_valid(int a, int b)
{
   # 如果 a 和 b 的符号不同，则没有溢出
   if ((a >= 0) != (b >= 0)) return 1;
   # 如果 a 和 b 都小于 0，则判断 a 是否大于等于 INT_MIN - b；INT_MIN - b 不会溢出，因为 b < 0
   if (a < 0 && b < 0) return a >= INT_MIN - b;
   # 判断 a 是否小于等于 INT_MAX - b
   return a <= INT_MAX - b;
}

// 返回 1 如果两个有符号短整数的乘积有效，溢出时返回 0
static int stbi__mul2shorts_valid(short a, short b)
{
   # 如果 b 等于 0 或 -1，则返回 1；乘以 0 总是 0；检查 -1 是为了避免 SHRT_MIN/b 溢出
   if (b == 0 || b == -1) return 1;
   # 如果 a 和 b 的符号相同，则判断 a 是否小于等于 SHRT_MAX/b
   if ((a >= 0) == (b >= 0)) return a <= SHRT_MAX/b;
   # 如果 b 小于 0，则判断 a 是否小于等于 SHRT_MIN / b
   if (b < 0) return a <= SHRT_MIN / b;
   # 否则判断 a 是否大于等于 SHRT_MIN / b
   return a >= SHRT_MIN / b;
}

// stbi__err - error
// stbi__errpf - error returning pointer to float
// stbi__errpuc - error returning pointer to unsigned char

#ifdef STBI_NO_FAILURE_STRINGS
   #define stbi__err(x,y)  0
#elif defined(STBI_FAILURE_USERMSG)
   #define stbi__err(x,y)  stbi__err(y)
#else
   #define stbi__err(x,y)  stbi__err(x)
#endif

#define stbi__errpf(x,y)   ((float *)(size_t) (stbi__err(x,y)?NULL:NULL))
#define stbi__errpuc(x,y)  ((unsigned char *)(size_t) (stbi__err(x,y)?NULL:NULL))

STBIDEF void stbi_image_free(void *retval_from_stbi_load)
{
   STBI_FREE(retval_from_stbi_load);
}

#ifndef STBI_NO_LINEAR
# 如果未定义 STBI_NO_LINEAR，则定义 stbi__ldr_to_hdr 函数
static float   *stbi__ldr_to_hdr(stbi_uc *data, int x, int y, int comp);
#endif

#ifndef STBI_NO_HDR
# 如果未定义 STBI_NO_HDR，则定义 stbi__hdr_to_ldr 函数
static stbi_uc *stbi__hdr_to_ldr(float   *data, int x, int y, int comp);
#endif

# 定义一个全局变量，用于在加载时垂直翻转图像
static int stbi__vertically_flip_on_load_global = 0;
# 设置是否在加载时垂直翻转图像的全局变量
STBIDEF void stbi_set_flip_vertically_on_load(int flag_true_if_should_flip)
{
   stbi__vertically_flip_on_load_global = flag_true_if_should_flip;
}

# 如果没有定义 STBI_THREAD_LOCAL，则使用全局变量 stbi__vertically_flip_on_load
#ifndef STBI_THREAD_LOCAL
#define stbi__vertically_flip_on_load  stbi__vertically_flip_on_load_global
#else
# 定义线程本地变量和标志位
static STBI_THREAD_LOCAL int stbi__vertically_flip_on_load_local, stbi__vertically_flip_on_load_set;

# 设置是否在加载时垂直翻转图像的线程本地变量
STBIDEF void stbi_set_flip_vertically_on_load_thread(int flag_true_if_should_flip)
{
   stbi__vertically_flip_on_load_local = flag_true_if_should_flip;
   stbi__vertically_flip_on_load_set = 1;
}

# 定义宏，根据线程本地变量和全局变量来确定是否垂直翻转图像
#define stbi__vertically_flip_on_load  (stbi__vertically_flip_on_load_set       \
                                         ? stbi__vertically_flip_on_load_local  \
                                         : stbi__vertically_flip_on_load_global)
#endif // STBI_THREAD_LOCAL

# 定义加载图像的主要函数
static void *stbi__load_main(stbi__context *s, int *x, int *y, int *comp, int req_comp, stbi__result_info *ri, int bpc)
{
   // 将 ri 指向的内存区域初始化为 0，确保在添加新字段时已经初始化
   memset(ri, 0, sizeof(*ri)); 
   // 设置每个通道的位数为 8，大多数情况下不需要更改
   ri->bits_per_channel = 8; 
   // 设置通道顺序为 RGB，以便可以添加 BGR 顺序
   ri->channel_order = STBI_ORDER_RGB; 
   // 初始化通道数为 0
   ri->num_channels = 0;

   // 使用非常明确的头部测试格式（至少首先是 FOURCC 或独特的魔术数字）
   #ifndef STBI_NO_PNG
   if (stbi__png_test(s))  return stbi__png_load(s,x,y,comp,req_comp, ri);
   #endif
   #ifndef STBI_NO_BMP
   if (stbi__bmp_test(s))  return stbi__bmp_load(s,x,y,comp,req_comp, ri);
   #endif
   #ifndef STBI_NO_GIF
   if (stbi__gif_test(s))  return stbi__gif_load(s,x,y,comp,req_comp, ri);
   #endif
   #ifndef STBI_NO_PSD
   if (stbi__psd_test(s))  return stbi__psd_load(s,x,y,comp,req_comp, ri, bpc);
   #else
   STBI_NOTUSED(bpc);
   #endif
   #ifndef STBI_NO_PIC
   if (stbi__pic_test(s))  return stbi__pic_load(s,x,y,comp,req_comp, ri);
   #endif

   // 然后尝试只有 1 或 2 个字节匹配预期的格式，这些容易产生误报，所以稍后再尝试
   #ifndef STBI_NO_JPEG
   if (stbi__jpeg_test(s)) return stbi__jpeg_load(s,x,y,comp,req_comp, ri);
   #endif
   #ifndef STBI_NO_PNM
   if (stbi__pnm_test(s))  return stbi__pnm_load(s,x,y,comp,req_comp, ri);
   #endif

   #ifndef STBI_NO_HDR
   if (stbi__hdr_test(s)) {
      float *hdr = stbi__hdr_load(s, x,y,comp,req_comp, ri);
      return stbi__hdr_to_ldr(hdr, *x, *y, req_comp ? req_comp : *comp);
   }
   #endif

   #ifndef STBI_NO_TGA
   // 最后测试 tga，因为它的测试方法不太好！
   if (stbi__tga_test(s))
      return stbi__tga_load(s,x,y,comp,req_comp, ri);
   #endif

   // 返回错误信息，表示图像类型未知或损坏
   return stbi__errpuc("unknown image type", "Image not of any known type, or corrupt");
}

// 将 16 位数据转换为 8 位数据
static stbi_uc *stbi__convert_16_to_8(stbi__uint16 *orig, int w, int h, int channels)
{
   // 定义变量 i
   int i;
   // 计算图像长度
   int img_len = w * h * channels;
   // 声明指针变量 reduced
   stbi_uc *reduced;

   // 分配内存给 reduced 指针
   reduced = (stbi_uc *) stbi__malloc(img_len);
   // 如果分配内存失败，则返回错误信息
   if (reduced == NULL) return stbi__errpuc("outofmem", "Out of memory");

   // 将原始图像数据缩小到 8 位
   for (i = 0; i < img_len; ++i)
      reduced[i] = (stbi_uc)((orig[i] >> 8) & 0xFF); // top half of each byte is sufficient approx of 16->8 bit scaling

   // 释放原始图像数据的内存
   STBI_FREE(orig);
   // 返回缩小后的图像数据
   return reduced;
}

// 将 8 位图像数据转换为 16 位
static stbi__uint16 *stbi__convert_8_to_16(stbi_uc *orig, int w, int h, int channels)
{
   // 定义变量 i
   int i;
   // 计算图像长度
   int img_len = w * h * channels;
   // 声明指针变量 enlarged
   stbi__uint16 *enlarged;

   // 分配内存给 enlarged 指针
   enlarged = (stbi__uint16 *) stbi__malloc(img_len*2);
   // 如果分配内存失败，则返回错误信息
   if (enlarged == NULL) return (stbi__uint16 *) stbi__errpuc("outofmem", "Out of memory");

   // 将原始图像数据扩大到 16 位
   for (i = 0; i < img_len; ++i)
      enlarged[i] = (stbi__uint16)((orig[i] << 8) + orig[i]); // replicate to high and low byte, maps 0->0, 255->0xffff

   // 释放原始图像数据的内存
   STBI_FREE(orig);
   // 返回扩大后的图像数据
   return enlarged;
}

// 垂直翻转图像
static void stbi__vertical_flip(void *image, int w, int h, int bytes_per_pixel)
{
   // 定义变量 row
   int row;
   // 计算每行的字节数
   size_t bytes_per_row = (size_t)w * bytes_per_pixel;
   // 声明临时数组 temp 和字节指针 bytes
   stbi_uc temp[2048];
   stbi_uc *bytes = (stbi_uc *)image;

   // 遍历图像的一半，进行垂直翻转
   for (row = 0; row < (h>>1); row++) {
      stbi_uc *row0 = bytes + row*bytes_per_row;
      stbi_uc *row1 = bytes + (h - row - 1)*bytes_per_row;
      // 交换 row0 和 row1 的内容
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

// 垂直翻转图像的切片
#ifndef STBI_NO_GIF
static void stbi__vertical_flip_slices(void *image, int w, int h, int z, int bytes_per_pixel)
{
   // 定义变量 slice，用于循环迭代
   int slice;
   // 计算每个切片的大小
   int slice_size = w * h * bytes_per_pixel;

   // 将 image 转换为字节指针
   stbi_uc *bytes = (stbi_uc *)image;
   // 遍历每个切片
   for (slice = 0; slice < z; ++slice) {
      // 垂直翻转当前切片的像素数据
      stbi__vertical_flip(bytes, w, h, bytes_per_pixel);
      // 移动到下一个切片的起始位置
      bytes += slice_size;
   }
}
#endif

// 加载并进行 8 位后处理
static unsigned char *stbi__load_and_postprocess_8bit(stbi__context *s, int *x, int *y, int *comp, int req_comp)
{
   // 加载主要图像数据
   stbi__result_info ri;
   void *result = stbi__load_main(s, x, y, comp, req_comp, &ri, 8);

   // 如果加载失败，则返回空指针
   if (result == NULL)
      return NULL;

   // 加载器负责确保我们得到 8 或 16 位的数据
   STBI_ASSERT(ri.bits_per_channel == 8 || ri.bits_per_channel == 16);

   // 如果数据不是 8 位，则转换为 8 位
   if (ri.bits_per_channel != 8) {
      result = stbi__convert_16_to_8((stbi__uint16 *) result, *x, *y, req_comp == 0 ? *comp : req_comp);
      ri.bits_per_channel = 8;
   }

   // @TODO: 将 stbi__convert_format 移动到这里

   // 如果需要在加载时垂直翻转图像
   if (stbi__vertically_flip_on_load) {
      int channels = req_comp ? req_comp : *comp;
      stbi__vertical_flip(result, *x, *y, channels * sizeof(stbi_uc));
   }

   // 返回处理后的图像数据
   return (unsigned char *) result;
}

// 加载并进行 16 位后处理
static stbi__uint16 *stbi__load_and_postprocess_16bit(stbi__context *s, int *x, int *y, int *comp, int req_comp)
{
   # 定义 stbi__result_info 结构体变量 ri
   stbi__result_info ri;
   # 调用 stbi__load_main 函数加载图像数据，并返回结果
   void *result = stbi__load_main(s, x, y, comp, req_comp, &ri, 16);

   # 如果加载失败，则返回空指针
   if (result == NULL)
      return NULL;

   # 断言每个通道的位数为 8 或 16
   STBI_ASSERT(ri.bits_per_channel == 8 || ri.bits_per_channel == 16);

   # 如果每个通道的位数不是 16，则将结果转换为 16 位
   if (ri.bits_per_channel != 16) {
      result = stbi__convert_8_to_16((stbi_uc *) result, *x, *y, req_comp == 0 ? *comp : req_comp);
      ri.bits_per_channel = 16;
   }

   # 如果需要垂直翻转加载的图像，则进行翻转操作
   if (stbi__vertically_flip_on_load) {
      int channels = req_comp ? req_comp : *comp;
      stbi__vertical_flip(result, *x, *y, channels * sizeof(stbi__uint16));
   }

   # 返回转换为 16 位的结果
   return (stbi__uint16 *) result;
}

# 如果未定义 STBI_NO_HDR 和 STBI_NO_LINEAR
static void stbi__float_postprocess(float *result, int *x, int *y, int *comp, int req_comp)
{
   # 如果需要垂直翻转加载的图像，并且结果不为空，则进行翻转操作
   if (stbi__vertically_flip_on_load && result != NULL) {
      int channels = req_comp ? req_comp : *comp;
      stbi__vertical_flip(result, *x, *y, channels * sizeof(float));
   }
}

# 如果未定义 STBI_NO_STDIO
# 如果定义了 _WIN32 和 STBI_WINDOWS_UTF8
STBI_EXTERN __declspec(dllimport) int __stdcall MultiByteToWideChar(unsigned int cp, unsigned long flags, const char *str, int cbmb, wchar_t *widestr, int cchwide);
STBI_EXTERN __declspec(dllimport) int __stdcall WideCharToMultiByte(unsigned int cp, unsigned long flags, const wchar_t *widestr, int cchwide, char *str, int cbmb, const char *defchar, int *used_default);
#endif

# 如果定义了 _WIN32 和 STBI_WINDOWS_UTF8
STBIDEF int stbi_convert_wchar_to_utf8(char *buffer, size_t bufferlen, const wchar_t* input)
{
    return WideCharToMultiByte(65001 /* UTF8 */, 0, input, -1, buffer, (int) bufferlen, NULL, NULL);
}

# 定义 stbi__fopen 函数，用于打开文件
static FILE *stbi__fopen(char const *filename, char const *mode)
{
   FILE *f;
#if defined(_WIN32) && defined(STBI_WINDOWS_UTF8)
   # 如果是在 Windows 平台且定义了 STBI_WINDOWS_UTF8
   wchar_t wMode[64];
   wchar_t wFilename[1024];
    # 将文件名从 UTF-8 转换为宽字符格式
    if (0 == MultiByteToWideChar(65001 /* UTF8 */, 0, filename, -1, wFilename, sizeof(wFilename)/sizeof(*wFilename)))
      return 0;
    # 将打开模式从 UTF-8 转换为宽字符格式
    if (0 == MultiByteToWideChar(65001 /* UTF8 */, 0, mode, -1, wMode, sizeof(wMode)/sizeof(*wMode)))
      return 0;
    # 如果是在 Visual Studio 2005 及以上版本
    if (0 != _wfopen_s(&f, wFilename, wMode))
        f = 0;
    # 否则
   f = _wfopen(wFilename, wMode);
#elif defined(_MSC_VER) && _MSC_VER >= 1400
   # 如果是在 Visual Studio 2005 及以上版本
   if (0 != fopen_s(&f, filename, mode))
      f=0;
#else
   # 否则
   f = fopen(filename, mode);
#endif
   return f;
}

STBIDEF stbi_uc *stbi_load(char const *filename, int *x, int *y, int *comp, int req_comp)
{
   # 打开文件
   FILE *f = stbi__fopen(filename, "rb");
   unsigned char *result;
   # 如果文件打开失败
   if (!f) return stbi__errpuc("can't fopen", "Unable to open file");
   # 从文件加载图像数据
   result = stbi_load_from_file(f,x,y,comp,req_comp);
   # 关闭文件
   fclose(f);
   return result;
}

STBIDEF stbi_uc *stbi_load_from_file(FILE *f, int *x, int *y, int *comp, int req_comp)
{
   unsigned char *result;
   stbi__context s;
   # 初始化文件上下文
   stbi__start_file(&s,f);
   # 加载并处理 8 位图像数据
   result = stbi__load_and_postprocess_8bit(&s,x,y,comp,req_comp);
   if (result) {
      # 需要将 IO 缓冲区中的所有字符“unget”
      fseek(f, - (int) (s.img_buffer_end - s.img_buffer), SEEK_CUR);
   }
   return result;
}

STBIDEF stbi__uint16 *stbi_load_from_file_16(FILE *f, int *x, int *y, int *comp, int req_comp)
{
   stbi__uint16 *result;
   stbi__context s;
   # 初始化文件上下文
   stbi__start_file(&s,f);
   # 加载并处理 16 位图像数据
   result = stbi__load_and_postprocess_16bit(&s,x,y,comp,req_comp);
   if (result) {
      # 需要将 IO 缓冲区中的所有字符“unget”
      fseek(f, - (int) (s.img_buffer_end - s.img_buffer), SEEK_CUR);
   }
   return result;
}

STBIDEF stbi_us *stbi_load_16(char const *filename, int *x, int *y, int *comp, int req_comp)
{
   // 打开文件并以二进制只读模式读取文件内容
   FILE *f = stbi__fopen(filename, "rb");
   // 定义结果指针
   stbi__uint16 *result;
   // 如果文件打开失败，则返回错误信息
   if (!f) return (stbi_us *) stbi__errpuc("can't fopen", "Unable to open file");
   // 从文件中加载16位数据
   result = stbi_load_from_file_16(f,x,y,comp,req_comp);
   // 关闭文件
   fclose(f);
   // 返回加载的结果
   return result;
}

#endif //!STBI_NO_STDIO

// 从内存中加载16位数据
STBIDEF stbi_us *stbi_load_16_from_memory(stbi_uc const *buffer, int len, int *x, int *y, int *channels_in_file, int desired_channels)
{
   // 创建上下文对象并从内存开始读取数据
   stbi__context s;
   stbi__start_mem(&s,buffer,len);
   // 调用函数加载并处理16位数据
   return stbi__load_and_postprocess_16bit(&s,x,y,channels_in_file,desired_channels);
}

// 从回调函数中加载16位数据
STBIDEF stbi_us *stbi_load_16_from_callbacks(stbi_io_callbacks const *clbk, void *user, int *x, int *y, int *channels_in_file, int desired_channels)
{
   // 创建上下文对象并从回调函数开始读取数据
   stbi__context s;
   stbi__start_callbacks(&s, (stbi_io_callbacks *)clbk, user);
   // 调用函数加载并处理16位数据
   return stbi__load_and_postprocess_16bit(&s,x,y,channels_in_file,desired_channels);
}

// 从内存中加载8位数据
STBIDEF stbi_uc *stbi_load_from_memory(stbi_uc const *buffer, int len, int *x, int *y, int *comp, int req_comp)
{
   // 创建上下文对象并从内存开始读取数据
   stbi__context s;
   stbi__start_mem(&s,buffer,len);
   // 调用函数加载并处理8位数据
   return stbi__load_and_postprocess_8bit(&s,x,y,comp,req_comp);
}

// 从回调函数中加载8位数据
STBIDEF stbi_uc *stbi_load_from_callbacks(stbi_io_callbacks const *clbk, void *user, int *x, int *y, int *comp, int req_comp)
{
   // 创建上下文对象并从回调函数开始读取数据
   stbi__context s;
   stbi__start_callbacks(&s, (stbi_io_callbacks *) clbk, user);
   // 调用函数加载并处理8位数据
   return stbi__load_and_postprocess_8bit(&s,x,y,comp,req_comp);
}

#ifndef STBI_NO_GIF
// 从内存中加载GIF数据
STBIDEF stbi_uc *stbi_load_gif_from_memory(stbi_uc const *buffer, int len, int **delays, int *x, int *y, int *z, int *comp, int req_comp)
{
   // 定义结果指针
   unsigned char *result;
   // 创建上下文对象并从内存开始读取数据
   stbi__context s;
   stbi__start_mem(&s,buffer,len);
   // 调用函数加载GIF数据
   result = (unsigned char*) stbi__load_gif_main(&s, delays, x, y, z, comp, req_comp);
   // 如果需要垂直翻转加载的数据，则进行翻转操作
   if (stbi__vertically_flip_on_load) {
      stbi__vertical_flip_slices( result, *x, *y, *z, *comp );
   }
   // 返回加载的结果
   return result;
}
#endif

#ifndef STBI_NO_LINEAR
// 加载浮点数据
static float *stbi__loadf_main(stbi__context *s, int *x, int *y, int *comp, int req_comp)
{
   unsigned char *data;
   #ifndef STBI_NO_HDR
   // 如果不定义 STBI_NO_HDR，则进行 HDR 格式检测
   if (stbi__hdr_test(s)) {
      // 如果是 HDR 格式，则加载 HDR 数据并进行后处理
      stbi__result_info ri;
      float *hdr_data = stbi__hdr_load(s,x,y,comp,req_comp, &ri);
      if (hdr_data)
         stbi__float_postprocess(hdr_data,x,y,comp,req_comp);
      return hdr_data;
   }
   #endif
   // 如果不是 HDR 格式，则加载并进行后处理 8 位数据
   data = stbi__load_and_postprocess_8bit(s, x, y, comp, req_comp);
   if (data)
      // 将 LDR 数据转换为 HDR 数据
      return stbi__ldr_to_hdr(data, *x, *y, req_comp ? req_comp : *comp);
   // 返回错误信息
   return stbi__errpf("unknown image type", "Image not of any known type, or corrupt");
}

// 从内存加载 HDR 数据
STBIDEF float *stbi_loadf_from_memory(stbi_uc const *buffer, int len, int *x, int *y, int *comp, int req_comp)
{
   stbi__context s;
   stbi__start_mem(&s,buffer,len);
   return stbi__loadf_main(&s,x,y,comp,req_comp);
}

// 从回调函数加载 HDR 数据
STBIDEF float *stbi_loadf_from_callbacks(stbi_io_callbacks const *clbk, void *user, int *x, int *y, int *comp, int req_comp)
{
   stbi__context s;
   stbi__start_callbacks(&s, (stbi_io_callbacks *) clbk, user);
   return stbi__loadf_main(&s,x,y,comp,req_comp);
}

#ifndef STBI_NO_STDIO
// 从文件加载 HDR 数据
STBIDEF float *stbi_loadf(char const *filename, int *x, int *y, int *comp, int req_comp)
{
   float *result;
   FILE *f = stbi__fopen(filename, "rb");
   if (!f) return stbi__errpf("can't fopen", "Unable to open file");
   result = stbi_loadf_from_file(f,x,y,comp,req_comp);
   fclose(f);
   return result;
}

// 从文件指针加载 HDR 数据
STBIDEF float *stbi_loadf_from_file(FILE *f, int *x, int *y, int *comp, int req_comp)
{
   stbi__context s;
   stbi__start_file(&s,f);
   return stbi__loadf_main(&s,x,y,comp,req_comp);
}
#endif // !STBI_NO_STDIO

#endif // !STBI_NO_LINEAR

// 这些 is-hdr-or-not 是独立定义的，与 STBI_NO_LINEAR 的定义无关，为了 API 的简单性；
// 如果定义了 STBI_NO_LINEAR，则始终返回 false！
// 从内存判断是否为 HDR 数据
STBIDEF int stbi_is_hdr_from_memory(stbi_uc const *buffer, int len)
{
   #ifndef STBI_NO_HDR
   // 如果未定义 STBI_NO_HDR，则执行以下代码
   stbi__context s;
   // 创建 stbi__context 结构体对象 s
   stbi__start_mem(&s,buffer,len);
   // 通过内存数据 buffer 和长度 len 初始化 stbi__context 对象 s
   return stbi__hdr_test(&s);
   // 调用 stbi__hdr_test 函数，传入 stbi__context 对象 s，返回测试结果
   #else
   // 如果定义了 STBI_NO_HDR，则执行以下代码
   STBI_NOTUSED(buffer);
   // 使用 STBI_NOTUSED 宏来避免编译器警告，表示未使用 buffer 参数
   STBI_NOTUSED(len);
   // 使用 STBI_NOTUSED 宏来避免编译器警告，表示未使用 len 参数
   return 0;
   // 返回 0
   #endif
}

#ifndef STBI_NO_STDIO
// 如果未定义 STBI_NO_STDIO，则执行以下代码
STBIDEF int      stbi_is_hdr          (char const *filename)
{
   // 定义函数 stbi_is_hdr，接受文件名参数
   FILE *f = stbi__fopen(filename, "rb");
   // 打开文件名对应的文件，以只读方式打开
   int result=0;
   // 初始化结果为 0
   if (f) {
      // 如果成功打开文件
      result = stbi_is_hdr_from_file(f);
      // 调用 stbi_is_hdr_from_file 函数，传入文件指针 f，获取结果
      fclose(f);
      // 关闭文件
   }
   return result;
   // 返回结果
}

STBIDEF int stbi_is_hdr_from_file(FILE *f)
{
   // 定义函数 stbi_is_hdr_from_file，接受文件指针参数
   #ifndef STBI_NO_HDR
   // 如果未定义 STBI_NO_HDR，则执行以下代码
   long pos = ftell(f);
   // 获取当前文件指针位置
   int res;
   // 定义结果变量
   stbi__context s;
   // 创建 stbi__context 结构体对象 s
   stbi__start_file(&s,f);
   // 通过文件指针 f 初始化 stbi__context 对象 s
   res = stbi__hdr_test(&s);
   // 调用 stbi__hdr_test 函数，传入 stbi__context 对象 s，获取结果
   fseek(f, pos, SEEK_SET);
   // 将文件指针位置设置回初始位置
   return res;
   // 返回结果
   #else
   // 如果定义了 STBI_NO_HDR，则执行以下代码
   STBI_NOTUSED(f);
   // 使用 STBI_NOTUSED 宏来避免编译器警告，表示未使用 f 参数
   return 0;
   // 返回 0
   #endif
}
#endif // !STBI_NO_STDIO

STBIDEF int      stbi_is_hdr_from_callbacks(stbi_io_callbacks const *clbk, void *user)
{
   // 定义函数 stbi_is_hdr_from_callbacks，接受回调函数指针和用户数据参数
   #ifndef STBI_NO_HDR
   // 如果未定义 STBI_NO_HDR，则执行以下代码
   stbi__context s;
   // 创建 stbi__context 结构体对象 s
   stbi__start_callbacks(&s, (stbi_io_callbacks *) clbk, user);
   // 通过回调函数指针和用户数据初始化 stbi__context 对象 s
   return stbi__hdr_test(&s);
   // 调用 stbi__hdr_test 函数，传入 stbi__context 对象 s，获取结果
   #else
   // 如果定义了 STBI_NO_HDR，则执行以下代码
   STBI_NOTUSED(clbk);
   // 使用 STBI_NOTUSED 宏来避免编译器警告，表示未使用 clbk 参数
   STBI_NOTUSED(user);
   // 使用 STBI_NOTUSED 宏来避免编译器警告，表示未使用 user 参数
   return 0;
   // 返回 0
   #endif
}

#ifndef STBI_NO_LINEAR
// 如果未定义 STBI_NO_LINEAR，则执行以下代码
static float stbi__l2h_gamma=2.2f, stbi__l2h_scale=1.0f;
// 定义静态浮点型变量 stbi__l2h_gamma 和 stbi__l2h_scale，并初始化值

STBIDEF void   stbi_ldr_to_hdr_gamma(float gamma) { stbi__l2h_gamma = gamma; }
// 定义函数 stbi_ldr_to_hdr_gamma，接受浮点型参数，用于设置 stbi__l2h_gamma 的值
STBIDEF void   stbi_ldr_to_hdr_scale(float scale) { stbi__l2h_scale = scale; }
// 定义函数 stbi_ldr_to_hdr_scale，接受浮点型参数，用于设置 stbi__l2h_scale 的值
#endif

static float stbi__h2l_gamma_i=1.0f/2.2f, stbi__h2l_scale_i=1.0f;
// 定义静态浮点型变量 stbi__h2l_gamma_i 和 stbi__h2l_scale_i，并初始化值

STBIDEF void   stbi_hdr_to_ldr_gamma(float gamma) { stbi__h2l_gamma_i = 1/gamma; }
// 定义函数 stbi_hdr_to_ldr_gamma，接受浮点型参数，用于设置 stbi__h2l_gamma_i 的值
STBIDEF void   stbi_hdr_to_ldr_scale(float scale) { stbi__h2l_scale_i = 1/scale; }
// 定义函数 stbi_hdr_to_ldr_scale，接受浮点型参数，用于设置 stbi__h2l_scale_i 的值


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
// 定义枚举类型，用于表示加载、类型和头部扫描

static void stbi__refill_buffer(stbi__context *s)
// 定义静态函数 stbi__refill_buffer，接受 stbi__context 结构体指针参数
{
   // 从输入流中读取数据到缓冲区，返回读取的字节数
   int n = (s->io.read)(s->io_user_data,(char*)s->buffer_start,s->buflen);
   // 更新已经读取的回调数据长度
   s->callback_already_read += (int) (s->img_buffer - s->img_buffer_original);
   // 如果读取的字节数为0，表示已到文件末尾
   if (n == 0) {
      // 处理已到文件末尾的情况，将 img_buffer 指向缓冲区起始位置，并设置 img_buffer_end 为起始位置后的一个字节
      s->read_from_callbacks = 0;
      s->img_buffer = s->buffer_start;
      s->img_buffer_end = s->buffer_start+1;
      *s->img_buffer = 0;
   } else {
      // 更新 img_buffer 和 img_buffer_end 的位置
      s->img_buffer = s->buffer_start;
      s->img_buffer_end = s->buffer_start + n;
   }
}

// 从缓冲区中获取一个字节的数据
stbi_inline static stbi_uc stbi__get8(stbi__context *s)
{
   // 如果缓冲区中还有数据，直接返回一个字节的数据并移动指针
   if (s->img_buffer < s->img_buffer_end)
      return *s->img_buffer++;
   // 如果缓冲区中没有数据，且可以从回调函数中读取数据，则调用 stbi__refill_buffer 函数重新填充缓冲区并返回一个字节的数据
   if (s->read_from_callbacks) {
      stbi__refill_buffer(s);
      return *s->img_buffer++;
   }
   // 如果缓冲区中没有数据且无法从回调函数中读取数据，则返回0
   return 0;
}

#if defined(STBI_NO_JPEG) && defined(STBI_NO_HDR) && defined(STBI_NO_PIC) && defined(STBI_NO_PNM)
// 无操作
#else
// 检查是否已到达文件末尾
stbi_inline static int stbi__at_eof(stbi__context *s)
{
   // 如果有读取函数，则调用 eof 函数判断是否已到达文件末尾
   if (s->io.read) {
      if (!(s->io.eof)(s->io_user_data)) return 0;
      // 如果 feof() 为真，则检查缓冲区是否等于结束位置，特殊情况：只有一个特殊的0字符在末尾
      if (s->read_from_callbacks == 0) return 1;
   }
   // 返回缓冲区是否已经到达结束位置
   return s->img_buffer >= s->img_buffer_end;
}
#endif

#if defined(STBI_NO_JPEG) && defined(STBI_NO_PNG) && defined(STBI_NO_BMP) && defined(STBI_NO_PSD) && defined(STBI_NO_TGA) && defined(STBI_NO_GIF) && defined(STBI_NO_PIC)
// 无操作
#else
// 跳过指定数量的字节
static void stbi__skip(stbi__context *s, int n)
{
   // 如果 n 为0，表示已经在目标位置，直接返回
   if (n == 0) return;
   // 如果 n 小于0，将缓冲区指针移动到结束位置
   if (n < 0) {
      s->img_buffer = s->img_buffer_end;
      return;
   }
   // 如果有读取函数，并且剩余的缓冲区长度小于 n，则调用 skip 函数跳过剩余的字节数
   if (s->io.read) {
      int blen = (int) (s->img_buffer_end - s->img_buffer);
      if (blen < n) {
         s->img_buffer = s->img_buffer_end;
         (s->io.skip)(s->io_user_data, n - blen);
         return;
      }
   }
   // 更新缓冲区指针位置
   s->img_buffer += n;
}
#endif
#if defined(STBI_NO_PNG) && defined(STBI_NO_TGA) && defined(STBI_NO_HDR) && defined(STBI_NO_PNM)
// 如果定义了 STBI_NO_PNG、STBI_NO_TGA、STBI_NO_HDR 和 STBI_NO_PNM，则什么都不做
// 否则，定义 stbi__getn 函数
#else
static int stbi__getn(stbi__context *s, stbi_uc *buffer, int n)
{
   // 如果 stbi__context 结构体中有读取函数
   if (s->io.read) {
      // 计算图像缓冲区中剩余的字节数
      int blen = (int) (s->img_buffer_end - s->img_buffer);
      // 如果剩余字节数小于 n
      if (blen < n) {
         int res, count;

         // 将图像缓冲区中的数据拷贝到 buffer 中
         memcpy(buffer, s->img_buffer, blen);

         // 调用读取函数，将数据读取到 buffer 中
         count = (s->io.read)(s->io_user_data, (char*) buffer + blen, n - blen);
         // 判断是否成功读取了 n-blen 字节的数据
         res = (count == (n-blen));
         // 将图像缓冲区指针指向末尾
         s->img_buffer = s->img_buffer_end;
         return res;
      }
   }

   // 如果图像缓冲区中剩余的字节数大于等于 n
   if (s->img_buffer+n <= s->img_buffer_end) {
      // 将图像缓冲区中的数据拷贝到 buffer 中
      memcpy(buffer, s->img_buffer, n);
      // 将图像缓冲区指针向后移动 n 个字节
      s->img_buffer += n;
      return 1;
   } else
      return 0;
}
#endif

#if defined(STBI_NO_JPEG) && defined(STBI_NO_PNG) && defined(STBI_NO_PSD) && defined(STBI_NO_PIC)
// 如果定义了 STBI_NO_JPEG、STBI_NO_PNG、STBI_NO_PSD 和 STBI_NO_PIC，则什么都不做
// 否则，定义 stbi__get16be 函数
#else
static int stbi__get16be(stbi__context *s)
{
   // 调用 stbi__get8 函数获取一个字节
   int z = stbi__get8(s);
   // 将获取的字节左移 8 位，再加上下一个字节的值，得到一个 16 位的值
   return (z << 8) + stbi__get8(s);
}
#endif

#if defined(STBI_NO_PNG) && defined(STBI_NO_PSD) && defined(STBI_NO_PIC)
// 如果定义了 STBI_NO_PNG、STBI_NO_PSD 和 STBI_NO_PIC，则什么都不做
// 否则，定义 stbi__get32be 函数
#else
static stbi__uint32 stbi__get32be(stbi__context *s)
{
   // 调用 stbi__get16be 函数获取一个 16 位的值
   stbi__uint32 z = stbi__get16be(s);
   // 将获取的 16 位值左移 16 位，再加上下一个 16 位值的值，得到一个 32 位的值
   return (z << 16) + stbi__get16be(s);
}
#endif

#if defined(STBI_NO_BMP) && defined(STBI_NO_TGA) && defined(STBI_NO_GIF)
// 如果定义了 STBI_NO_BMP、STBI_NO_TGA 和 STBI_NO_GIF，则什么都不做
// 否则，定义 stbi__get16le 函数
#else
static int stbi__get16le(stbi__context *s)
{
   // 调用 stbi__get8 函数获取一个字节
   int z = stbi__get8(s);
   // 将获取的字节加上下一个字节的值左移 8 位，得到一个 16 位的值
   return z + (stbi__get8(s) << 8);
}
#endif

#ifndef STBI_NO_BMP
static stbi__uint32 stbi__get32le(stbi__context *s)
{
   // 调用 stbi__get16le 函数获取一个 16 位的值
   stbi__uint32 z = stbi__get16le(s);
   // 将获取的 16 位值加上下一个 16 位值的值左移 16 位，得到一个 32 位的值
   z += (stbi__uint32)stbi__get16le(s) << 16;
   return z;
}
#endif

#define STBI__BYTECAST(x)  ((stbi_uc) ((x) & 255))  // 将 int 截断为字节，避免警告

#if defined(STBI_NO_JPEG) && defined(STBI_NO_PNG) && defined(STBI_NO_BMP) && defined(STBI_NO_PSD) && defined(STBI_NO_TGA) && defined(STBI_NO_GIF) && defined(STBI_NO_PIC) && defined(STBI_NO_PNM)
// 如果定义了 STBI_NO_JPEG、STBI_NO_PNG、STBI_NO_BMP、STBI_NO_PSD、STBI_NO_TGA、STBI_NO_GIF、STBI_NO_PIC 和 STBI_NO_PNM，则什么都不做
// 否则，定义以下内容
#else
//////////////////////////////////////////////////////////////////////////////
//
// 通用转换器，从内置的 img_n 转换为 req_comp
//    各个类型尽可能自动完成此操作（例如，jpeg 在内部自动处理所有情况，因为它需要进行颜色空间转换，而且它从不具有 alpha 通道，所以情况很少）。
//    png 可以自动交错一个 alpha=255 通道，但对于其他情况则退回到这个函数
//
//  假设数据缓冲区是通过 malloc 分配的，因此需要 malloc 一个新的缓冲区，然后释放旧的缓冲区
//  唯一的失败模式是 malloc 失败

static stbi_uc stbi__compute_y(int r, int g, int b)
{
   return (stbi_uc) (((r*77) + (g*150) +  (29*b)) >> 8);
}
#endif

#if defined(STBI_NO_PNG) && defined(STBI_NO_BMP) && defined(STBI_NO_PSD) && defined(STBI_NO_TGA) && defined(STBI_NO_GIF) && defined(STBI_NO_PIC) && defined(STBI_NO_PNM)
// 什么都不做
#else
static unsigned char *stbi__convert_format(unsigned char *data, int img_n, int req_comp, unsigned int x, unsigned int y)
}
#endif

#if defined(STBI_NO_PNG) && defined(STBI_NO_PSD)
// 什么都不做
#else
static stbi__uint16 stbi__compute_y_16(int r, int g, int b)
{
   return (stbi__uint16) (((r*77) + (g*150) +  (29*b)) >> 8);
}
#endif

#if defined(STBI_NO_PNG) && defined(STBI_NO_PSD)
// 什么都不做
#else
static stbi__uint16 *stbi__convert_format16(stbi__uint16 *data, int img_n, int req_comp, unsigned int x, unsigned int y)
}
#endif

#ifndef STBI_NO_LINEAR
static float   *stbi__ldr_to_hdr(stbi_uc *data, int x, int y, int comp)
{
   // 声明整型变量 i, k, n
   int i,k,n;
   // 声明指向浮点数的指针 output
   float *output;
   // 如果 data 为空，则返回空指针
   if (!data) return NULL;
   // 分配内存给 output，大小为 x*y*comp*sizeof(float)，并初始化为 0
   output = (float *) stbi__malloc_mad4(x, y, comp, sizeof(float), 0);
   // 如果 output 为空，则释放 data 内存并返回错误信息
   if (output == NULL) { STBI_FREE(data); return stbi__errpf("outofmem", "Out of memory"); }
   // 计算非 alpha 通道的组件数量
   if (comp & 1) n = comp; else n = comp-1;
   // 循环遍历 x*y 次
   for (i=0; i < x*y; ++i) {
      // 循环遍历非 alpha 通道的组件数量
      for (k=0; k < n; ++k) {
         // 计算输出值并存入 output 数组
         output[i*comp + k] = (float) (pow(data[i*comp+k]/255.0f, stbi__l2h_gamma) * stbi__l2h_scale);
      }
   }
   // 如果非 alpha 通道的组件数量小于 comp
   if (n < comp) {
      // 循环遍历 x*y 次
      for (i=0; i < x*y; ++i) {
         // 将 data 中的值转换为浮点数并存入 output 数组
         output[i*comp + n] = data[i*comp + n]/255.0f;
      }
   }
   // 释放 data 内存
   STBI_FREE(data);
   // 返回 output 数组
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
   // 声明指向无符号字符的指针 output
   stbi_uc *output;
   // 如果 data 为空，则返回空指针
   if (!data) return NULL;
   // 分配内存给 output，大小为 x*y*comp，初始化为 0
   output = (stbi_uc *) stbi__malloc_mad3(x, y, comp, 0);
   // 如果 output 为空，则释放 data 内存并返回错误信息
   if (output == NULL) { STBI_FREE(data); return stbi__errpuc("outofmem", "Out of memory"); }
   // 计算非 alpha 通道的组件数量
   if (comp & 1) n = comp; else n = comp-1;
   // 循环遍历 x*y 次
   for (i=0; i < x*y; ++i) {
      // 循环遍历非 alpha 通道的组件数量
      for (k=0; k < n; ++k) {
         // 计算输出值并存入 output 数组
         float z = (float) pow(data[i*comp+k]*stbi__h2l_scale_i, stbi__h2l_gamma_i) * 255 + 0.5f;
         if (z < 0) z = 0;
         if (z > 255) z = 255;
         output[i*comp + k] = (stbi_uc) stbi__float2int(z);
      }
      // 如果 k 小于 comp
      if (k < comp) {
         // 计算输出值并存入 output 数组
         float z = data[i*comp+k] * 255 + 0.5f;
         if (z < 0) z = 0;
         if (z > 255) z = 255;
         output[i*comp + k] = (stbi_uc) stbi__float2int(z);
      }
   }
   // 释放 data 内存
   STBI_FREE(data);
   // 返回 output 数组
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
// 定义了一些关于 JPEG 解码加速的宏和结构体

// 定义了一个结构体 stbi__huffman，用于存储 huffman 解码加速所需的数据
typedef struct
{
   stbi_uc  fast[1 << FAST_BITS];  // 用于快速查找的数组
   stbi__uint16 code[256];  // 存储 huffman 编码
   stbi_uc  values[256];  // 存储 huffman 编码对应的值
   stbi_uc  size[257];  // 存储 huffman 编码的长度
   unsigned int maxcode[18];  // 存储最大编码值
   int    delta[17];   // 存储差值
} stbi__huffman;

// 定义了一个结构体，用于存储 JPEG 解码所需的数据
typedef struct
{
   stbi__context *s;  // 存储上下文信息
   stbi__huffman huff_dc[4];  // 存储 DC 分量的 huffman 解码数据
   stbi__huffman huff_ac[4];  // 存储 AC 分量的 huffman 解码数据
   stbi__uint16 dequant[4][64];  // 存储量化表
   stbi__int16 fast_ac[4][1 << FAST_BITS];  // 用于快速查找的数组

   // 存储图像组件的大小和交错 MCU
   int img_h_max, img_v_max;  // 图像组件的最大高度和宽度
   int img_mcu_x, img_mcu_y;  // MCU 的 x 和 y 坐标
   int img_mcu_w, img_mcu_h;  // MCU 的宽度和高度
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
    short   *coeff;   // 仅用于渐进扫描
    int      coeff_w, coeff_h; // 8x8 系数块的数量
} img_comp[4]; // 图像组件数组，最多包含 4 个组件

stbi__uint32   code_buffer; // JPEG 熵编码缓冲区
int            code_bits;   // 有效位数
unsigned char  marker;      // 在填充熵缓冲区时遇到的标记
int            nomore;      // 如果遇到标记，则必须停止的标志

int            progressive; // 渐进式扫描标志
int            spec_start; // 开始扫描的位置
int            spec_end; // 结束扫描的位置
int            succ_high; // 高位系数的累积
int            succ_low; // 低位系数的累积
int            eob_run; // EOB 运行
int            jfif; // JFIF 标志
int            app14_color_transform; // Adobe APP14 标签
int            rgb; // RGB 标志

int scan_n, order[4]; // 扫描数量和顺序数组
int restart_interval, todo; // 重启间隔和待处理的任务数量

// 内核
void (*idct_block_kernel)(stbi_uc *out, int out_stride, short data[64]); // IDCT 块内核
void (*YCbCr_to_RGB_kernel)(stbi_uc *out, const stbi_uc *y, const stbi_uc *pcb, const stbi_uc *pcr, int count, int step); // YCbCr 转 RGB 内核
stbi_uc *(*resample_row_hv_2_kernel)(stbi_uc *out, stbi_uc *in_near, stbi_uc *in_far, int w, int hs); // 重采样行内核
} stbi__jpeg; // JPEG 解码器结构体

static int stbi__build_huffman(stbi__huffman *h, int *count) // 构建哈夫曼表的函数
{
   // 定义整型变量i、j、k，并初始化k为0
   int i,j,k=0;
   // 定义无符号整型变量code
   unsigned int code;
   // 根据JPEG规范，为每个符号构建大小列表
   for (i=0; i < 16; ++i) {
      for (j=0; j < count[i]; ++j) {
         // 将大小值存入h->size数组中
         h->size[k++] = (stbi_uc) (i+1);
         // 如果大小列表超过257，则返回错误信息
         if(k >= 257) return stbi__err("bad size list","Corrupt JPEG");
      }
   }
   // 将h->size数组的最后一个元素设为0
   h->size[k] = 0;

   // 计算实际符号（根据JPEG规范）
   code = 0;
   k = 0;
   for(j=1; j <= 16; ++j) {
      // 计算要添加到code中以计算符号ID的增量
      h->delta[j] = k - code;
      if (h->size[k] == j) {
         while (h->size[k] == j)
            // 将code值赋给h->code数组
            h->code[k++] = (stbi__uint16) (code++);
         // 如果code-1大于等于2的j次方，则返回错误信息
         if (code-1 >= (1u << j)) return stbi__err("bad code lengths","Corrupt JPEG");
      }
      // 计算该大小的最大code+1，根据需要进行预移位
      h->maxcode[j] = code << (16-j);
      code <<= 1;
   }
   // 将h->maxcode数组的最后一个元素设为0xffffffff
   h->maxcode[j] = 0xffffffff;

   // 构建非规范加速表；255是未加速的标志
   memset(h->fast, 255, 1 << FAST_BITS);
   for (i=0; i < k; ++i) {
      int s = h->size[i];
      if (s <= FAST_BITS) {
         int c = h->code[i] << (FAST_BITS-s);
         int m = 1 << (FAST_BITS-s);
         for (j=0; j < m; ++j) {
            // 将i存入h->fast数组中
            h->fast[c+j] = (stbi_uc) i;
         }
      }
   }
   // 返回1
   return 1;
}

// 构建一个表，一次性解码小AC的幅度和值
static void stbi__build_fast_ac(stbi__int16 *fast_ac, stbi__huffman *h)
{
   // 定义整型变量 i
   int i;
   // 循环，i 从 0 到 (1 << FAST_BITS) - 1
   for (i=0; i < (1 << FAST_BITS); ++i) {
      // 获取 h->fast[i] 的值
      stbi_uc fast = h->fast[i];
      // 初始化 fast_ac[i] 为 0
      fast_ac[i] = 0;
      // 如果 fast < 255
      if (fast < 255) {
         // 获取 h->values[fast] 的值
         int rs = h->values[fast];
         // 获取 rs 的高 4 位，表示连续零的个数
         int run = (rs >> 4) & 15;
         // 获取 rs 的低 4 位，表示霍夫曼编码的长度
         int magbits = rs & 15;
         // 获取 h->size[fast] 的值，表示霍夫曼编码的长度
         int len = h->size[fast];

         // 如果 magbits 不为 0，且 len + magbits 小于等于 FAST_BITS
         if (magbits && len + magbits <= FAST_BITS) {
            // 计算霍夫曼编码后接收扩展码
            int k = ((i << len) & ((1 << FAST_BITS) - 1)) >> (FAST_BITS - magbits);
            int m = 1 << (magbits - 1);
            // 如果 k 小于 m，则 k 加上 (~0U << magbits) + 1
            if (k < m) k += (~0U << magbits) + 1;
            // 如果结果足够小，可以放入 fast_ac 表中
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
      // 读取一个字节
      unsigned int b = j->nomore ? 0 : stbi__get8(j->s);
      // 如果读取的字节为 0xff
      if (b == 0xff) {
         // 获取下一个字节
         int c = stbi__get8(j->s);
         // 消耗填充字节
         while (c == 0xff) c = stbi__get8(j->s); // consume fill bytes
         // 如果 c 不为 0，则设置 marker 为 c，nomore 为 1，然后返回
         if (c != 0) {
            j->marker = (unsigned char) c;
            j->nomore = 1;
            return;
         }
      }
      // 将 b 放入 code_buffer 中
      j->code_buffer |= b << (24 - j->code_bits);
      j->code_bits += 8;
   } while (j->code_bits <= 24);
}

// (1 << n) - 1
// 定义一个常量数组，表示 (1 << n) - 1 的值
static const stbi__uint32 stbi__bmask[17]={0,1,3,7,15,31,63,127,255,511,1023,2047,4095,8191,16383,32767,65535};

// 从比特流中解码 JPEG 霍夫曼值
stbi_inline static int stbi__jpeg_huff_decode(stbi__jpeg *j, stbi__huffman *h)
{
   // 声明无符号整型变量temp
   unsigned int temp;
   // 声明整型变量c和k
   int c,k;

   // 如果当前编码位数小于16，则扩展缓冲区
   if (j->code_bits < 16) stbi__grow_buffer_unsafe(j);

   // 查看顶部的FAST_BITS，并确定它是什么符号ID，如果编码小于等于FAST_BITS
   c = (j->code_buffer >> (32 - FAST_BITS)) & ((1 << FAST_BITS)-1);
   k = h->fast[c];
   // 如果k小于255
   if (k < 255) {
      // 获取符号的大小
      int s = h->size[k];
      // 如果大小大于当前编码位数，则返回-1
      if (s > j->code_bits)
         return -1;
      // 将编码缓冲区左移s位
      j->code_buffer <<= s;
      // 减去编码位数
      j->code_bits -= s;
      // 返回对应的值
      return h->values[k];
   }

   // 简单的测试是将code_buffer向下移动k位，然后与maxcode进行比较
   // 为了加快速度，我们将maxcode向左移动，使其末尾有(16-k)个0；
   // 换句话说，无论位数是多少，它都希望与移位后的16进行比较；
   // 这样我们就不需要在循环内部进行移位。
   temp = j->code_buffer >> 16;
   for (k=FAST_BITS+1 ; ; ++k)
      if (temp < h->maxcode[k])
         break;
   // 如果k等于17，则返回错误，未找到编码
   if (k == 17) {
      j->code_bits -= 16;
      return -1;
   }

   // 如果k大于当前编码位数，则返回-1
   if (k > j->code_bits)
      return -1;

   // 将哈夫曼编码转换为符号ID
   c = ((j->code_buffer >> (32 - k)) & stbi__bmask[k]) + h->delta[k];
   // 如果c小于0或大于等于256，则符号ID超出范围
   if(c < 0 || c >= 256)
       return -1;
   // 断言，确保编码转换后的值与哈夫曼编码匹配
   STBI_ASSERT((((j->code_buffer) >> (32 - h->size[c])) & stbi__bmask[h->size[c]]) == h->code[c]);

   // 将编码位数减去k
   j->code_bits -= k;
   // 将编码缓冲区左移k位
   j->code_buffer <<= k;
   // 返回对应的值
   return h->values[c];
}

// bias[n] = (-1<<n) + 1
// 定义静态整型数组stbi__jbias，用于JPEG解码时的偏置值
static const int stbi__jbias[16] = {0,-1,-3,-7,-15,-31,-63,-127,-255,-511,-1023,-2047,-4095,-8191,-16383,-32767};

// 结合JPEG的“接收”和“扩展”，因为基线总是扩展它接收到的所有内容。
// 定义静态内联函数stbi__extend_receive，用于接收和扩展JPEG数据
stbi_inline static int stbi__extend_receive(stbi__jpeg *j, int n)
{
   unsigned int k; // 声明一个无符号整数变量k
   int sgn; // 声明一个整数变量sgn
   if (j->code_bits < n) stbi__grow_buffer_unsafe(j); // 如果code_bits小于n，则调用stbi__grow_buffer_unsafe函数
   if (j->code_bits < n) return 0; // 如果code_bits小于n，则返回0，表示从流中取出的位数不够，无法继续
   sgn = j->code_buffer >> 31; // 将code_buffer右移31位，得到符号位，0表示正数，1表示负数
   k = stbi_lrot(j->code_buffer, n); // 将code_buffer左循环移位n位，得到k
   j->code_buffer = k & ~stbi__bmask[n]; // 将k与~stbi__bmask[n]按位与，结果赋值给code_buffer
   k &= stbi__bmask[n]; // 将k与stbi__bmask[n]按位与，结果赋值给k
   j->code_bits -= n; // code_bits减去n
   return k + (stbi__jbias[n] & (sgn - 1)); // 返回k加上(stbi__jbias[n]与(sgn-1)的按位与结果)
}

// get some unsigned bits
stbi_inline static int stbi__jpeg_get_bits(stbi__jpeg *j, int n)
{
   unsigned int k; // 声明一个无符号整数变量k
   if (j->code_bits < n) stbi__grow_buffer_unsafe(j); // 如果code_bits小于n，则调用stbi__grow_buffer_unsafe函数
   if (j->code_bits < n) return 0; // 如果code_bits小于n，则返回0，表示从流中取出的位数不够，无法继续
   k = stbi_lrot(j->code_buffer, n); // 将code_buffer左循环移位n位，得到k
   j->code_buffer = k & ~stbi__bmask[n]; // 将k与~stbi__bmask[n]按位与，结果赋值给code_buffer
   k &= stbi__bmask[n]; // 将k与stbi__bmask[n]按位与，结果赋值给k
   j->code_bits -= n; // code_bits减去n
   return k; // 返回k
}

stbi_inline static int stbi__jpeg_get_bit(stbi__jpeg *j)
{
   unsigned int k; // 声明一个无符号整数变量k
   if (j->code_bits < 1) stbi__grow_buffer_unsafe(j); // 如果code_bits小于1，则调用stbi__grow_buffer_unsafe函数
   if (j->code_bits < 1) return 0; // 如果code_bits小于1，则返回0，表示从流中取出的位数不够，无法继续
   k = j->code_buffer; // 将code_buffer赋值给k
   j->code_buffer <<= 1; // 将code_buffer左移1位
   --j->code_bits; // code_bits减1
   return k & 0x80000000; // 返回k与0x80000000的按位与结果
}

// given a value that's at position X in the zigzag stream,
// where does it appear in the 8x8 matrix coded as row-major?
static const stbi_uc stbi__jpeg_dezigzag[64+15] =
{
    // 64个值表示在zigzag流中的位置，对应在8x8矩阵中的位置
    0,  1,  8, 16,  9,  2,  3, 10,
   17, 24, 32, 25, 18, 11,  4,  5,
   12, 19, 26, 33, 40, 48, 41, 34,
   27, 20, 13,  6,  7, 14, 21, 28,
   35, 42, 49, 56, 57, 50, 43, 36,
   29, 22, 15, 23, 30, 37, 44, 51,
   58, 59, 52, 45, 38, 31, 39, 46,
   53, 60, 61, 54, 47, 55, 62, 63,
   // let corrupt input sample past end
   63, 63, 63, 63, 63, 63, 63, 63, // 填充值，用于处理损坏的输入样本
   63, 63, 63, 63, 63, 63, 63 // 填充值，用于处理损坏的输入样本
};

// decode one 64-entry block--
static int stbi__jpeg_decode_block(stbi__jpeg *j, short data[64], stbi__huffman *hdc, stbi__huffman *hac, stbi__int16 *fac, int b, stbi__uint16 *dequant)
{
   // 定义变量 diff, dc, k, t
   int diff,dc,k;
   int t;

   // 如果当前的编码位数小于16，则扩展缓冲区
   if (j->code_bits < 16) stbi__grow_buffer_unsafe(j);
   // 使用霍夫曼解码函数解码 DC 成分
   t = stbi__jpeg_huff_decode(j, hdc);
   // 如果解码结果小于0或大于15，则返回错误信息
   if (t < 0 || t > 15) return stbi__err("bad huffman code","Corrupt JPEG");

   // 将数据数组全部置零，以便后续可以32位一次处理
   memset(data,0,64*sizeof(data[0]));

   // 计算差值 diff，如果 t 不为0，则使用扩展接收函数处理 t，否则 diff 为0
   diff = t ? stbi__extend_receive(j, t) : 0;
   // 如果 DC 预测值与 diff 相加后结果不合法，则返回错误信息
   if (!stbi__addints_valid(j->img_comp[b].dc_pred, diff)) return stbi__err("bad delta","Corrupt JPEG");
   // 计算 DC 值
   dc = j->img_comp[b].dc_pred + diff;
   j->img_comp[b].dc_pred = dc;
   // 如果 DC 值与量化表的乘积不合法，则返回错误信息
   if (!stbi__mul2shorts_valid(dc, dequant[0])) return stbi__err("can't merge dc and ac", "Corrupt JPEG");
   // 将计算得到的 DC 值乘以量化表的值，存入数据数组的第一个位置
   data[0] = (short) (dc * dequant[0]);

   // 解码 AC 成分，参考 JPEG 规范
   k = 1;
   do {
      unsigned int zig;
      int c,r,s;
      // 如果当前的编码位数小于16，则扩展缓冲区
      if (j->code_bits < 16) stbi__grow_buffer_unsafe(j);
      // 从编码缓冲区中取出 FAST_BITS 位，根据其值查找对应的霍夫曼编码
      c = (j->code_buffer >> (32 - FAST_BITS)) & ((1 << FAST_BITS)-1);
      r = fac[c];
      if (r) { // 快速 AC 路径
         // 更新 k，计算 run 的长度
         k += (r >> 4) & 15; // run
         // 获取 combined length
         s = r & 15; // combined length
         // 如果 combined length 大于当前的编码位数，则返回错误信息
         if (s > j->code_bits) return stbi__err("bad huffman code", "Combined length longer than code bits available");
         // 将编码缓冲区左移 s 位，更新编码位数
         j->code_buffer <<= s;
         j->code_bits -= s;
         // 解码并存入 unzigzag'd 位置
         zig = stbi__jpeg_dezigzag[k++];
         data[zig] = (short) ((r >> 8) * dequant[zig]);
      } else {
         // 从霍夫曼表中解码 AC 成分
         int rs = stbi__jpeg_huff_decode(j, hac);
         // 如果解码结果小于0，则返回错误信息
         if (rs < 0) return stbi__err("bad huffman code","Corrupt JPEG");
         s = rs & 15;
         r = rs >> 4;
         if (s == 0) {
            // 如果 s 为0，且 rs 不等于0xf0，则结束块
            if (rs != 0xf0) break; // end block
            k += 16;
         } else {
            // 更新 k，计算 run 的长度
            k += r;
            // 解码并存入 unzigzag'd 位置
            zig = stbi__jpeg_dezigzag[k++];
            data[zig] = (short) (stbi__extend_receive(j,s) * dequant[zig]);
         }
      }
   } while (k < 64);
   // 返回解码结果
   return 1;
}

// 静态函数，用于渐进式扫描模式下解码 DC 成分
static int stbi__jpeg_decode_block_prog_dc(stbi__jpeg *j, short data[64], stbi__huffman *hdc, int b)
{
   // 定义变量 diff 和 dc，用于存储差值和直流分量
   int diff,dc;
   // 定义变量 t
   int t;
   // 如果 j->spec_end 不等于 0，则返回错误信息 "can't merge dc and ac", "Corrupt JPEG"
   if (j->spec_end != 0) return stbi__err("can't merge dc and ac", "Corrupt JPEG");

   // 如果 j->code_bits 小于 16，则调用 stbi__grow_buffer_unsafe 函数
   if (j->code_bits < 16) stbi__grow_buffer_unsafe(j);

   // 如果 j->succ_high 等于 0
   if (j->succ_high == 0) {
      // 对数据进行初始化，将所有的 ac 值都设置为 0
      memset(data,0,64*sizeof(data[0])); // 0 all the ac values now
      // 调用 stbi__jpeg_huff_decode 函数解码直流分量
      t = stbi__jpeg_huff_decode(j, hdc);
      // 如果 t 小于 0 或者 t 大于 15，则返回错误信息 "can't merge dc and ac", "Corrupt JPEG"
      if (t < 0 || t > 15) return stbi__err("can't merge dc and ac", "Corrupt JPEG");
      // 计算差值
      diff = t ? stbi__extend_receive(j, t) : 0;

      // 如果直流分量和差值的和不合法，则返回错误信息 "bad delta", "Corrupt JPEG"
      if (!stbi__addints_valid(j->img_comp[b].dc_pred, diff)) return stbi__err("bad delta", "Corrupt JPEG");
      // 计算直流分量
      dc = j->img_comp[b].dc_pred + diff;
      j->img_comp[b].dc_pred = dc;
      // 如果直流分量乘以 2 的 j->succ_low 次方后不合法，则返回错误信息 "can't merge dc and ac", "Corrupt JPEG"
      if (!stbi__mul2shorts_valid(dc, 1 << j->succ_low)) return stbi__err("can't merge dc and ac", "Corrupt JPEG");
      // 将计算后的直流分量存入数据数组中
      data[0] = (short) (dc * (1 << j->succ_low));
   } else {
      // 对直流分量进行细化扫描
      if (stbi__jpeg_get_bit(j))
         data[0] += (short) (1 << j->succ_low);
   }
   // 返回 1
   return 1;
}

// @OPTIMIZE: store non-zigzagged during the decode passes,
// and only de-zigzag when dequantizing
// 优化注释，存储非蛇形排列的数据，在解码过程中进行处理，只在去量化时进行反蛇形排列

// 对 -128..127 的值进行限制并转换为 0..255
stbi_inline static stbi_uc stbi__clamp(int x)
{
   // 通过一个测试来捕获两种情况
   if ((unsigned int) x > 255) {
      if (x < 0) return 0;
      if (x > 255) return 255;
   }
   return (stbi_uc) x;
}

// 定义宏，将浮点数乘以 4096 并四舍五入取整
#define stbi__f2f(x)  ((int) (((x) * 4096 + 0.5)))
// 定义宏，将浮点数乘以 4096
#define stbi__fsh(x)  ((x) * 4096)

// 源自 jidctint -- DCT_ISLOW
// 衍生自 jidctint -- DCT_ISLOW
#define STBI__IDCT_1D(s0,s1,s2,s3,s4,s5,s6,s7) \ 
   int t0,t1,t2,t3,p1,p2,p3,p4,p5,x0,x1,x2,x3; \  // 定义临时变量
   p2 = s2;                                    \  // 将输入参数赋值给临时变量
   p3 = s6;                                    \  // 将输入参数赋值给临时变量
   p1 = (p2+p3) * stbi__f2f(0.5411961f);       \  // 计算临时变量的值
   t2 = p1 + p3*stbi__f2f(-1.847759065f);      \  // 计算临时变量的值
   t3 = p1 + p2*stbi__f2f( 0.765366865f);      \  // 计算临时变量的值
   p2 = s0;                                    \  // 将输入参数赋值给临时变量
   p3 = s4;                                    \  // 将输入参数赋值给临时变量
   t0 = stbi__fsh(p2+p3);                      \  // 调用函数计算临时变量的值
   t1 = stbi__fsh(p2-p3);                      \  // 调用函数计算临时变量的值
   x0 = t0+t3;                                 \  // 计算临时变量的值
   x3 = t0-t3;                                 \  // 计算临时变量的值
   x1 = t1+t2;                                 \  // 计算临时变量的值
   x2 = t1-t2;                                 \  // 计算临时变量的值
   t0 = s7;                                    \  // 将输入参数赋值给临时变量
   t1 = s5;                                    \  // 将输入参数赋值给临时变量
   t2 = s3;                                    \  // 将输入参数赋值给临时变量
   t3 = s1;                                    \  // 将输入参数赋值给临时变量
   p3 = t0+t2;                                 \  // 计算临时变量的值
   p4 = t1+t3;                                 \  // 计算临时变量的值
   p1 = t0+t3;                                 \  // 计算临时变量的值
   p2 = t1+t2;                                 \  // 计算临时变量的值
   p5 = (p3+p4)*stbi__f2f( 1.175875602f);      \  // 计算临时变量的值
   t0 = t0*stbi__f2f( 0.298631336f);           \  // 计算临时变量的值
   t1 = t1*stbi__f2f( 2.053119869f);           \  // 计算临时变量的值
   t2 = t2*stbi__f2f( 3.072711026f);           \  // 计算临时变量的值
   t3 = t3*stbi__f2f( 1.501321110f);           \  // 计算临时变量的值
   p1 = p5 + p1*stbi__f2f(-0.899976223f);      \  // 计算临时变量的值
   p2 = p5 + p2*stbi__f2f(-2.562915447f);      \  // 计算临时变量的值
   p3 = p3*stbi__f2f(-1.961570560f);           \  // 计算临时变量的值
   p4 = p4*stbi__f2f(-0.390180644f);           \  // 计算临时变量的值
   t3 += p1+p4;                                \  // 计算临时变量的值
   t2 += p2+p3;                                \  // 计算临时变量的值
   t1 += p2+p4;                                \  // 计算临时变量的值
   t0 += p1+p3;                                \  // 计算临时变量的值

static void stbi__idct_block(stbi_uc *out, int out_stride, short data[64])  // 定义函数，参数为输出指针、输出步长、数据数组

#ifdef STBI_SSE2  // 如果定义了 STBI_SSE2
// sse2 integer IDCT. not the fastest possible implementation but it  // 注释：sse2整数IDCT。不是最快的实现，但它
// produces bit-identical results to the generic C version so it's  // 产生与通用C版本完全相同的结果，因此它
// fully "transparent".  // 完全“透明”。
#ifdef STBI_NEON
// 如果编译器定义了 STBI_NEON，则执行以下代码

// NEON integer IDCT. should produce bit-identical
// results to the generic C version.
// NEON 整数 IDCT。应该产生与通用 C 版本完全相同的结果。
static void stbi__idct_simd(stbi_uc *out, int out_stride, short data[64])
{
   // 定义 8 个 16 位整数向量，用于存储 IDCT 运算的中间结果
   int16x8_t row0, row1, row2, row3, row4, row5, row6, row7;

   // 定义 4 个 16 位整数向量，用于存储 IDCT 运算中的旋转系数
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

   // 定义宏，用于执行 16 位整数向量的乘法运算
#define dct_long_mul(out, inq, coeff) \
   int32x4_t out##_l = vmull_s16(vget_low_s16(inq), coeff); \
   int32x4_t out##_h = vmull_s16(vget_high_s16(inq), coeff)

   // 定义宏，用于执行 16 位整数向量的乘加运算
#define dct_long_mac(out, acc, inq, coeff) \
   int32x4_t out##_l = vmlal_s16(acc##_l, vget_low_s16(inq), coeff); \
   int32x4_t out##_h = vmlal_s16(acc##_h, vget_high_s16(inq), coeff)

   // 定义宏，用于将 16 位整数向量扩展为 32 位整数向量
#define dct_widen(out, inq) \
   int32x4_t out##_l = vshll_n_s16(vget_low_s16(inq), 12); \
   int32x4_t out##_h = vshll_n_s16(vget_high_s16(inq), 12)

// wide add
// 定义宏，用于执行 32 位整数向量的加法运算
#define dct_wadd(out, a, b) \
   int32x4_t out##_l = vaddq_s32(a##_l, b##_l); \
   int32x4_t out##_h = vaddq_s32(a##_h, b##_h)

// wide sub
// 定义宏，用于执行 32 位整数向量的减法运算
// 定义宏，计算两个向量的差，并将结果存储在指定的输出向量中
#define dct_wsub(out, a, b) \
   int32x4_t out##_l = vsubq_s32(a##_l, b##_l); \
   int32x4_t out##_h = vsubq_s32(a##_h, b##_h)

// 宏定义，对两个向量进行蝶形运算，然后使用指定的位移操作和位移量进行位移，并打包结果
#define dct_bfly32o(out0,out1, a,b,shiftop,s) \
   { \
      // 计算和并存储在sum中
      dct_wadd(sum, a, b); \
      // 计算差并存储在dif中
      dct_wsub(dif, a, b); \
      // 将和进行位移操作，并打包结果存储在out0中
      out0 = vcombine_s16(shiftop(sum_l, s), shiftop(sum_h, s)); \
      // 将差进行位移操作，并打包结果存储在out1中
      out1 = vcombine_s16(shiftop(dif_l, s), shiftop(dif_h, s)); \
   }

// 这三个宏分别映射到单个VTRN.16、VTRN.32和VSWP。不幸的是，编译器是否真正理解这一点是另一回事。
#define dct_trn16(x, y) { int16x8x2_t t = vtrnq_s16(x, y); x = t.val[0]; y = t.val[1]; }
#define dct_trn32(x, y) { int32x4x2_t t = vtrnq_s32(vreinterpretq_s32_s16(x), vreinterpretq_s32_s16(y)); x = vreinterpretq_s16_s32(t.val[0]); y = vreinterpretq_s16_s32(t.val[1]); }
#define dct_trn64(x, y) { int16x8_t x0 = x; int16x8_t y0 = y; x = vcombine_s16(vget_low_s16(x0), vget_low_s16(y0)); y = vcombine_s16(vget_high_s16(x0), vget_high_s16(y0)); }

      // pass 1
      // 对row0和row1进行VTRN.16操作
      dct_trn16(row0, row1); // a0b0a2b2a4b4a6b6
      // 对row2和row3进行VTRN.16操作
      dct_trn16(row2, row3);
      // 对row4和row5进行VTRN.16操作
      dct_trn16(row4, row5);
      // 对row6和row7进行VTRN.16操作
      dct_trn16(row6, row7);

      // pass 2
      // 对row0和row2进行VTRN.32操作
      dct_trn32(row0, row2); // a0b0c0d0a4b4c4d4
      // 对row1和row3进行VTRN.32操作
      dct_trn32(row1, row3);
      // 对row4和row6进行VTRN.32操作
      dct_trn32(row4, row6);
      // 对row5和row7进行VTRN.32操作
      dct_trn32(row5, row7);

      // pass 3
      // 对row0和row4进行VTRN.64操作
      dct_trn64(row0, row4); // a0b0c0d0e0f0g0h0
      // 对row1和row5进行VTRN.64操作
      dct_trn64(row1, row5);
      // 对row2和row6进行VTRN.64操作
      dct_trn64(row2, row6);
      // 对row3和row7进行VTRN.64操作
      dct_trn64(row3, row7);

// 取消定义宏dct_trn16和dct_trn32
#undef dct_trn16
#undef dct_trn32
   // 取消定义 dct_trn64
   }

   // 行处理
   // vrshrn_n_s32 只支持最多 16 的位移，我们需要 17。所以先进行一个非四舍五入的 16 位移，然后再进行一个 1 的四舍五入位移。
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
#define dct_trn8_8(x, y) { uint8x8x2_t t = vtrn_u8(x, y); x = t.val[0]; y = t.val[1]; }
#define dct_trn8_16(x, y) { uint16x4x2_t t = vtrn_u16(vreinterpret_u16_u8(x), vreinterpret_u16_u8(y)); x = vreinterpret_u8_u16(t.val[0]); y = vreinterpret_u8_u16(t.val[1]); }
#define dct_trn8_32(x, y) { uint32x2x2_t t = vtrn_u32(vreinterpret_u32_u8(x), vreinterpret_u32_u8(y)); x = vreinterpret_u8_u32(t.val[0]); y = vreinterpret_u8_u32(t.val[1]); }

      // 遗憾的是，这里不能使用交错存储，因为我们只向每个扫描线写入 8 个字节！

      // 8x8 8 位转置处理 1
      dct_trn8_8(p0, p1);
      dct_trn8_8(p2, p3);
      dct_trn8_8(p4, p5);
      dct_trn8_8(p6, p7);

      // 处理 2
      dct_trn8_16(p0, p2);
      dct_trn8_16(p1, p3);
      dct_trn8_16(p4, p6);
      dct_trn8_16(p5, p7);

      // 处理 3
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
#undef dct_trn8_16
#undef dct_trn8_32
   }

#undef dct_long_mul
#undef dct_long_mac
#undef dct_widen
#undef dct_wadd
#undef dct_wsub
#undef dct_bfly32o
#undef dct_pass
}

#endif // STBI_NEON

#define STBI__MARKER_none  0xff
// 如果熵流中有待处理的标记，返回该标记；否则，从流中获取一个标记。如果没有标记，返回0xff，这是一个无效的标记值
static stbi_uc stbi__get_marker(stbi__jpeg *j)
{
   stbi_uc x;
   if (j->marker != STBI__MARKER_none) { x = j->marker; j->marker = STBI__MARKER_none; return x; }
   x = stbi__get8(j->s);
   if (x != 0xff) return STBI__MARKER_none;
   while (x == 0xff)
      x = stbi__get8(j->s); // 消耗重复的0xff填充字节
   return x;
}

// 在每个扫描中，我们将有scan_n个分量，分量的顺序由order[]指定
#define STBI__RESTART(x)     ((x) >= 0xd0 && (x) <= 0xd7)

// 在重启间隔之后，重置熵解码器和DC预测
static void stbi__jpeg_reset(stbi__jpeg *j)
{
   j->code_bits = 0;
   j->code_buffer = 0;
   j->nomore = 0;
   j->img_comp[0].dc_pred = j->img_comp[1].dc_pred = j->img_comp[2].dc_pred = j->img_comp[3].dc_pred = 0;
   j->marker = STBI__MARKER_none;
   j->todo = j->restart_interval ? j->restart_interval : 0x7fffffff;
   j->eob_run = 0;
   // 如果没有restart_interval，最多1<<31个MCU？这是非常安全的，因为我们甚至不允许1<<30像素
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
      // 对数据进行反量化和逆DCT变换
      int i,j,n;
      for (n=0; n < z->s->img_n; ++n) {
         // 计算每个分量的宽度和高度
         int w = (z->img_comp[n].x+7) >> 3;
         int h = (z->img_comp[n].y+7) >> 3;
         for (j=0; j < h; ++j) {
            for (i=0; i < w; ++i) {
               // 获取当前分量的数据指针
               short *data = z->img_comp[n].coeff + 64 * (i + j * z->img_comp[n].coeff_w);
               // 对数据进行反量化
               stbi__jpeg_dequantize(data, z->dequant[z->img_comp[n].tq]);
               // 对数据进行逆DCT变换
               z->idct_block_kernel(z->img_comp[n].data+z->img_comp[n].w2*j*8+i*8, z->img_comp[n].w2, data);
            }
         }
      }
   }
}

// 处理标记
static int stbi__process_marker(stbi__jpeg *z, int m)
}

// 在遇到SOS标记后
static int stbi__process_scan_header(stbi__jpeg *z)
{
   // 定义整型变量 i
   int i;
   // 从输入流中读取 16 位大端序的数据，赋值给变量 Ls
   int Ls = stbi__get16be(z->s);
   // 从输入流中读取 8 位数据，赋值给变量 scan_n
   z->scan_n = stbi__get8(z->s);
   // 如果 scan_n 不在 1 到 4 之间，或者大于图像通道数，则返回错误信息
   if (z->scan_n < 1 || z->scan_n > 4 || z->scan_n > (int) z->s->img_n) return stbi__err("bad SOS component count","Corrupt JPEG");
   // 如果 Ls 不等于 6+2*scan_n，则返回错误信息
   if (Ls != 6+2*z->scan_n) return stbi__err("bad SOS len","Corrupt JPEG");
   // 循环遍历 scan_n 次
   for (i=0; i < z->scan_n; ++i) {
      // 从输入流中读取 8 位数据，赋值给变量 id
      int id = stbi__get8(z->s), which;
      // 从输入流中读取 8 位数据，赋值给变量 q
      int q = stbi__get8(z->s);
      // 遍历图像通道数次
      for (which = 0; which < z->s->img_n; ++which)
         // 如果 img_comp 中的 id 与当前读取的 id 相等，则跳出循环
         if (z->img_comp[which].id == id)
            break;
      // 如果 which 等于图像通道数，则返回 0，表示没有匹配
      if (which == z->s->img_n) return 0; // no match
      // 根据读取的 q 更新 img_comp 中的 hd 和 ha
      z->img_comp[which].hd = q >> 4;   if (z->img_comp[which].hd > 3) return stbi__err("bad DC huff","Corrupt JPEG");
      z->img_comp[which].ha = q & 15;   if (z->img_comp[which].ha > 3) return stbi__err("bad AC huff","Corrupt JPEG");
      // 将 which 存入 order 数组中
      z->order[i] = which;
   }

   {
      // 定义整型变量 aa
      int aa;
      // 从输入流中读取 8 位数据，赋值给变量 spec_start
      z->spec_start = stbi__get8(z->s);
      // 从输入流中读取 8 位数据，赋值给变量 spec_end
      z->spec_end   = stbi__get8(z->s); // should be 63, but might be 0
      // 从输入流中读取 8 位数据，赋值给变量 aa
      aa = stbi__get8(z->s);
      // 将 aa 的高 4 位赋值给 succ_high，低 4 位赋值给 succ_low
      z->succ_high = (aa >> 4);
      z->succ_low  = (aa & 15);
      // 如果是渐进式扫描
      if (z->progressive) {
         // 如果 spec_start、spec_end 超出范围，或者 succ_high、succ_low 超出范围，则返回错误信息
         if (z->spec_start > 63 || z->spec_end > 63  || z->spec_start > z->spec_end || z->succ_high > 13 || z->succ_low > 13)
            return stbi__err("bad SOS", "Corrupt JPEG");
      } else {
         // 如果 spec_start 不等于 0，则返回错误信息
         if (z->spec_start != 0) return stbi__err("bad SOS","Corrupt JPEG");
         // 如果 succ_high、succ_low 不等于 0，则返回错误信息
         if (z->succ_high != 0 || z->succ_low != 0) return stbi__err("bad SOS","Corrupt JPEG");
         // 将 spec_end 设为 63
         z->spec_end = 63;
      }
   }

   // 返回 1，表示成功
   return 1;
}

// 释放 JPEG 组件
static int stbi__free_jpeg_components(stbi__jpeg *z, int ncomp, int why)
{
   // 定义整型变量 i
   int i;
   // 循环遍历每个图像分量
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
   // 返回 why 变量的值
   return why;
}

// 处理帧头部信息
static int stbi__process_frame_header(stbi__jpeg *z, int scan)
}

// 使用比较操作符，因为在某些情况下我们处理多个情况（例如 SOF）
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
   z->marker = STBI__MARKER_none; // 将缓存的标记初始化为空
   // 获取标记
   m = stbi__get_marker(z);
   // 如果没有 SOI 标记，则返回错误
   if (!stbi__SOI(m)) return stbi__err("no SOI","Corrupt JPEG");
   // 如果扫描类型为 STBI__SCAN_type，则返回 1
   if (scan == STBI__SCAN_type) return 1;
   // 获取标记
   m = stbi__get_marker(z);
   // 循环直到找到 SOF 标记
   while (!stbi__SOF(m)) {
      // 如果处理标记失败，则返回 0
      if (!stbi__process_marker(z,m)) return 0;
      // 获取下一个标记
      m = stbi__get_marker(z);
      // 如果标记为空，则继续扫描直到文件末尾
      while (m == STBI__MARKER_none) {
         // 一些文件在块之后有额外的填充，因此继续扫描
         if (stbi__at_eof(z->s)) return stbi__err("no SOF", "Corrupt JPEG");
         // 获取下一个标记
         m = stbi__get_marker(z);
      }
   }
   // 判断是否为渐进式 JPEG
   z->progressive = stbi__SOF_progressive(m);
   // 如果处理帧头部信息失败，则返回 0
   if (!stbi__process_frame_header(z, scan)) return 0;
   // 返回 1
   return 1;
}

// 跳过 JPEG 结尾处的无用数据
static int stbi__skip_jpeg_junk_at_end(stbi__jpeg *j)
{
   // 某些 JPEG 图像在结尾可能有垃圾数据，跳过这部分，但如果找到类似有效标记的内容，则从那里继续解析
   while (!stbi__at_eof(j->s)) {
      int x = stbi__get8(j->s);
      while (x == 255) { // 可能是一个标记
         if (stbi__at_eof(j->s)) return STBI__MARKER_none;
         x = stbi__get8(j->s);
         if (x != 0x00 && x != 0xff) {
            // 不是填充的零或另一个标记的前导，看起来像是一个实际的标记，返回它
            return x;
         }
         // 填充的零现在 x=0，结束循环，意味着我们回到常规扫描循环。
         // 重复的 0xff 继续尝试读取标记的下一个字节。
      }
   }
   return STBI__MARKER_none;
}

// 将图像解码为 YCbCr 格式
static int stbi__decode_jpeg_image(stbi__jpeg *j)
{
   # 定义整型变量 m
   int m;
   # 循环4次，对图像组件的原始数据和原始系数进行初始化
   for (m = 0; m < 4; m++) {
      j->img_comp[m].raw_data = NULL;
      j->img_comp[m].raw_coeff = NULL;
   }
   # 重置重启间隔
   j->restart_interval = 0;
   # 解码 JPEG 头部信息
   if (!stbi__decode_jpeg_header(j, STBI__SCAN_load)) return 0;
   # 获取 JPEG 标记
   m = stbi__get_marker(j);
   # 循环直到遇到结束标记
   while (!stbi__EOI(m)) {
      # 如果是扫描开始标记
      if (stbi__SOS(m)) {
         # 处理扫描头部信息
         if (!stbi__process_scan_header(j)) return 0;
         # 解析熵编码数据
         if (!stbi__parse_entropy_coded_data(j)) return 0;
         # 如果标记为 none，则跳过 JPEG 末尾的垃圾数据
         if (j->marker == STBI__MARKER_none ) {
         j->marker = stbi__skip_jpeg_junk_at_end(j);
            // 如果在没有遇到标记的情况下到达文件末尾，stbi__get_marker() 将失败并最终返回 0
         }
         # 获取下一个标记
         m = stbi__get_marker(j);
         # 如果是重启标记，则再次获取标记
         if (STBI__RESTART(m))
            m = stbi__get_marker(j);
      } else if (stbi__DNL(m)) {
         # 获取 DNL 标记的长度和高度
         int Ld = stbi__get16be(j->s);
         stbi__uint32 NL = stbi__get16be(j->s);
         # 如果长度不为4，则返回错误信息
         if (Ld != 4) return stbi__err("bad DNL len", "Corrupt JPEG");
         # 如果高度不等于图像的高度，则返回错误信息
         if (NL != j->s->img_y) return stbi__err("bad DNL height", "Corrupt JPEG");
         # 获取下一个标记
         m = stbi__get_marker(j);
      } else {
         # 处理其他标记
         if (!stbi__process_marker(j, m)) return 1;
         # 获取下一个标记
         m = stbi__get_marker(j);
      }
   }
   # 如果是渐进式扫描，则完成 JPEG 解码
   if (j->progressive)
      stbi__jpeg_finish(j);
   # 返回成功
   return 1;
}

// static jfif-centered resampling (across block boundaries)

# 定义静态函数指针类型 resample_row_func
typedef stbi_uc *(*resample_row_func)(stbi_uc *out, stbi_uc *in0, stbi_uc *in1, int w, int hs);

# 定义宏，将 x 右移2位并转换为 stbi_uc 类型
#define stbi__div4(x) ((stbi_uc) ((x) >> 2))

# 定义静态函数，对输入进行垂直方向的重采样
static stbi_uc *resample_row_1(stbi_uc *out, stbi_uc *in_near, stbi_uc *in_far, int w, int hs)
{
   STBI_NOTUSED(out);
   STBI_NOTUSED(in_far);
   STBI_NOTUSED(w);
   STBI_NOTUSED(hs);
   return in_near;
}

# 定义静态函数，对输入进行垂直方向的重采样，生成两个样本
static stbi_uc* stbi__resample_row_v_2(stbi_uc *out, stbi_uc *in_near, stbi_uc *in_far, int w, int hs)
{
   # 需要为每个输入生成两个垂直方向的样本
   int i;
   STBI_NOTUSED(hs);
   for (i=0; i < w; ++i)
      out[i] = stbi__div4(3*in_near[i] + in_far[i] + 2);
   return out;
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
   // 需要为每个输入生成2x2的样本
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
   // 需要为每个输入生成2x2的样本
   int i=0,t0,t1;

   if (w == 1) {
      out[0] = out[1] = stbi__div4(3*in_near[0] + in_far[0] + 2);
      return out;
   }

   t1 = 3*in_near[0] + in_far[0];
   // 处理每组8个像素，尽可能多地处理
   // 注意，我们不能在这个循环中处理行的最后一个像素
   // 因为我们需要处理滤波器的边界条件。
   for (; i < ((w-1) & ~7); i += 8) {
#elif defined(STBI_NEON)
      // 如果定义了 STBI_NEON，则执行以下代码块，使用 NEON 指令集进行优化

      // 加载并执行垂直滤波处理
      // 这里使用了 3*x + y = 4*x + (y - x) 的计算方式
      uint8x8_t farb  = vld1_u8(in_far + i);  // 从 in_far 数组中加载 8 个 uint8 值到 farb 变量
      uint8x8_t nearb = vld1_u8(in_near + i);  // 从 in_near 数组中加载 8 个 uint8 值到 nearb 变量
      int16x8_t diff  = vreinterpretq_s16_u16(vsubl_u8(farb, nearb));  // 计算 farb 和 nearb 之间的差值，并转换为 int16 类型
      int16x8_t nears = vreinterpretq_s16_u16(vshll_n_u8(nearb, 2));  // 将 nearb 左移 2 位，并转换为 int16 类型
      int16x8_t curr  = vaddq_s16(nears, diff); // 当前行

      // 水平滤波处理与当前行的移位版本相同。"prev" 是当前行向右移动 1 个像素；我们需要插入前一个像素值（来自 t1）。
      // "next" 是当前行向左移动 1 个像素，加入了下一个 8 个像素块的第一个像素。
      int16x8_t prv0 = vextq_s16(curr, curr, 7);  // 将 curr 向右移动 7 个位置，并存储到 prv0 变量
      int16x8_t nxt0 = vextq_s16(curr, curr, 1);  // 将 curr 向左移动 1 个位置，并存储到 nxt0 变量
      int16x8_t prev = vsetq_lane_s16(t1, prv0, 0);  // 将 t1 存储到 prv0 的第 0 个位置，并存储到 prev 变量
      int16x8_t next = vsetq_lane_s16(3*in_near[i+8] + in_far[i+8], nxt0, 7);  // 将计算结果存储到 nxt0 的第 7 个位置，并存储到 next 变量

      // 水平滤波，使用 polyphase 实现，因为这样更方便：
      // 偶数像素 = 3*cur + prev = cur*4 + (prev - cur)
      // 奇数像素 = 3*cur + next = cur*4 + (next - cur)
      // 注意共享的项。
      int16x8_t curs = vshlq_n_s16(curr, 2);  // 将 curr 左移 2 位，并存储到 curs 变量
      int16x8_t prvd = vsubq_s16(prev, curr);  // 计算 prev 和 curr 之间的差值，并存储到 prvd 变量
      int16x8_t nxtd = vsubq_s16(next, curr);  // 计算 next 和 curr 之间的差值，并存储到 nxtd 变量
      int16x8_t even = vaddq_s16(curs, prvd);  // 计算偶数像素的值，并存储到 even 变量
      int16x8_t odd  = vaddq_s16(curs, nxtd);  // 计算奇数像素的值，并存储到 odd 变量

      // 恢复缩放并四舍五入，然后交错存储偶数/奇数相位
      uint8x8x2_t o;  // 定义一个包含两个 uint8x8_t 类型的结构体
      o.val[0] = vqrshrun_n_s16(even, 4);  // 对 even 进行右移 4 位并转换为 uint8 类型，存储到 o 结构体的第一个元素
      o.val[1] = vqrshrun_n_s16(odd,  4);  // 对 odd 进行右移 4 位并转换为 uint8 类型，存储到 o 结构体的第二个元素
      vst2_u8(out + i*2, o);  // 将 o 结构体中的数据存储到 out 数组中，每个元素占用 2 个位置
#endif
// 结束条件编译指令

      // 下一次迭代的“previous”值
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
// 结束条件编译指令

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

// 这是一种降低精度的 YCbCr 到 RGB 的计算方法，用于确保代码在 SIMD 和标量中产生相同的结果
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
# 将 YCbCr 颜色空间转换为 RGB 颜色空间，使用 SIMD 指令加速
static void stbi__YCbCr_to_RGB_simd(stbi_uc *out, stbi_uc const *y, stbi_uc const *pcb, stbi_uc const *pcr, int count, int step)
{
   # 初始化变量 i 为 0
   int i = 0;
   # 结束条件的标记
#endif
#ifdef STBI_NEON
   // 如果支持 step=3，可以很容易地添加支持，但是否有需求呢？
   if (step == 4) {
      // 这是一个相当直接的实现，而且并不是特别优化过的。
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

   for (; i < count; ++i) {
      // 将 y[i] 左移 20 位，再加上 (1<<19)，相当于对 y[i] 进行四舍五入
      int y_fixed = (y[i] << 20) + (1<<19); // rounding
      int r,g,b;
      int cr = pcr[i] - 128;
      int cb = pcb[i] - 128;
      // 计算红色通道的值
      r = y_fixed + cr* stbi__float2fixed(1.40200f);
      // 计算绿色通道的值
      g = y_fixed + cr*-stbi__float2fixed(0.71414f) + ((cb*-stbi__float2fixed(0.34414f)) & 0xffff0000);
      // 计算蓝色通道的值
      b = y_fixed + cb* stbi__float2fixed(1.77200f);
      // 右移 20 位，相当于对 r、g、b 进行取整
      r >>= 20;
      g >>= 20;
      b >>= 20;
      // 如果 r、g、b 超出 0~255 的范围，则进行边界处理
      if ((unsigned) r > 255) { if (r < 0) r = 0; else r = 255; }
      if ((unsigned) g > 255) { if (g < 0) g = 0; else g = 255; }
      if ((unsigned) b > 255) { if (b < 0) b = 0; else b = 255; }
      // 将 r、g、b 分别存入输出数组中
      out[0] = (stbi_uc)r;
      out[1] = (stbi_uc)g;
      out[2] = (stbi_uc)b;
      out[3] = 255;
      // 更新输出数组的位置
      out += step;
   }
}
#endif

// 设置 JPEG 对象的内核函数
static void stbi__setup_jpeg(stbi__jpeg *j)
{
   // 设置 IDCT 块内核函数
   j->idct_block_kernel = stbi__idct_block;
   // 设置 YCbCr 转 RGB 内核函数
   j->YCbCr_to_RGB_kernel = stbi__YCbCr_to_RGB_row;
   // 设置水平垂直 2 倍重采样行内核函数
   j->resample_row_hv_2_kernel = stbi__resample_row_hv_2;

#ifdef STBI_SSE2
   // 如果支持 SSE2，则使用相应的内核函数
   if (stbi__sse2_available()) {
      j->idct_block_kernel = stbi__idct_simd;
      j->YCbCr_to_RGB_kernel = stbi__YCbCr_to_RGB_simd;
      j->resample_row_hv_2_kernel = stbi__resample_row_hv_2_simd;
   }
#endif

#ifdef STBI_NEON
   // 如果支持 NEON，则使用相应的内核函数
   j->idct_block_kernel = stbi__idct_simd;
   j->YCbCr_to_RGB_kernel = stbi__YCbCr_to_RGB_simd;
   j->resample_row_hv_2_kernel = stbi__resample_row_hv_2_simd;
#endif
}

// 清理临时组件缓冲区
static void stbi__cleanup_jpeg(stbi__jpeg *j)
{
   // 释放 JPEG 对象的组件
   stbi__free_jpeg_components(j, j->s->img_n, 0);
}

typedef struct
{
   resample_row_func resample;
   stbi_uc *line0,*line1;
   int hs,vs;   // 每个轴上的扩展因子
   int w_lores; // 水平像素预扩展
   int ystep;   // 垂直扩展进度
   int ypos;    // 预扩展行数
} stbi__resample;

// 快速 0..255 * 0..255 => 0..255 的四舍五入乘法
static stbi_uc stbi__blinn_8x8(stbi_uc x, stbi_uc y)
{
   // 计算 t 的值，t = x*y + 128
   unsigned int t = x*y + 128;
   // 返回 t 经过位运算后的结果
   return (stbi_uc) ((t + (t >>8)) >> 8);
}

// 加载 JPEG 图像
static stbi_uc *load_jpeg_image(stbi__jpeg *z, int *out_x, int *out_y, int *comp, int req_comp)
}

// 加载 JPEG 图像
static void *stbi__jpeg_load(stbi__context *s, int *x, int *y, int *comp, int req_comp, stbi__result_info *ri)
{
   // 声明结果指针
   unsigned char* result;
   // 分配内存给 JPEG 对象
   stbi__jpeg* j = (stbi__jpeg*) stbi__malloc(sizeof(stbi__jpeg));
   // 如果分配内存失败，返回错误信息
   if (!j) return stbi__errpuc("outofmem", "Out of memory");
   // 将分配的内存初始化为 0
   memset(j, 0, sizeof(stbi__jpeg));
   // 设置 JPEG 对象的输入流
   j->s = s;
   // 设置 JPEG 对象的参数
   stbi__setup_jpeg(j);
   // 加载 JPEG 图像
   result = load_jpeg_image(j, x,y,comp,req_comp);
   // 释放 JPEG 对象的内存
   STBI_FREE(j);
   // 返回加载的结果
   return result;
}

// 测试 JPEG 图像
static int stbi__jpeg_test(stbi__context *s)
{
   // 声明结果变量
   int r;
   // 分配内存给 JPEG 对象
   stbi__jpeg* j = (stbi__jpeg*)stbi__malloc(sizeof(stbi__jpeg));
   // 如果分配内存失败，返回错误信息
   if (!j) return stbi__err("outofmem", "Out of memory");
   // 将分配的内存初始化为 0
   memset(j, 0, sizeof(stbi__jpeg));
   // 设置 JPEG 对象的输入流
   j->s = s;
   // 设置 JPEG 对象的参数
   stbi__setup_jpeg(j);
   // 解码 JPEG 头部信息
   r = stbi__decode_jpeg_header(j, STBI__SCAN_type);
   // 重置输入流的位置
   stbi__rewind(s);
   // 释放 JPEG 对象的内存
   STBI_FREE(j);
   // 返回解码结果
   return r;
}

// 获取 JPEG 图像的原始信息
static int stbi__jpeg_info_raw(stbi__jpeg *j, int *x, int *y, int *comp)
{
   // 如果无法解码 JPEG 头部信息，重置输入流的位置并返回 0
   if (!stbi__decode_jpeg_header(j, STBI__SCAN_header)) {
      stbi__rewind( j->s );
      return 0;
   }
   // 如果 x 不为空，将图像宽度赋值给 x
   if (x) *x = j->s->img_x;
   // 如果 y 不为空，将图像高度赋值给 y
   if (y) *y = j->s->img_y;
   // 如果 comp 不为空，根据图像通道数赋值给 comp
   if (comp) *comp = j->s->img_n >= 3 ? 3 : 1;
   // 返回 1
   return 1;
}

// 获取 JPEG 图像的信息
static int stbi__jpeg_info(stbi__context *s, int *x, int *y, int *comp)
{
   // 声明结果变量
   int result;
   // 分配内存给 JPEG 对象
   stbi__jpeg* j = (stbi__jpeg*) (stbi__malloc(sizeof(stbi__jpeg)));
   // 如果分配内存失败，返回错误信息
   if (!j) return stbi__err("outofmem", "Out of memory");
   // 将分配的内存初始化为 0
   memset(j, 0, sizeof(stbi__jpeg));
   // 设置 JPEG 对象的输入流
   j->s = s;
   // 获取 JPEG 图像的信息
   result = stbi__jpeg_info_raw(j, x, y, comp);
   // 释放 JPEG 对象的内存
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
// 定义快速方式比JPEG哈夫曼更快，但慢速方式更慢
#define STBI__ZFAST_BITS  9 // 在默认表中加速所有情况
#define STBI__ZFAST_MASK  ((1 << STBI__ZFAST_BITS) - 1)
#define STBI__ZNSYMS 288 // 字面/长度字母表中的符号数

// zlib风格的哈夫曼编码
// (JPEG从左侧打包，zlib从右侧打包，因此无法共享代码)
typedef struct
{
   stbi__uint16 fast[1 << STBI__ZFAST_BITS];
   stbi__uint16 firstcode[16];
   int maxcode[17];
   stbi__uint16 firstsymbol[16];
   stbi_uc  size[STBI__ZNSYMS];
   stbi__uint16 value[STBI__ZNSYMS];
} stbi__zhuffman;

stbi_inline static int stbi__bitreverse16(int n)
{
  n = ((n & 0xAAAA) >>  1) | ((n & 0x5555) << 1);
  n = ((n & 0xCCCC) >>  2) | ((n & 0x3333) << 2);
  n = ((n & 0xF0F0) >>  4) | ((n & 0x0F0F) << 4);
  n = ((n & 0xFF00) >>  8) | ((n & 0x00FF) << 8);
  return n;
}

stbi_inline static int stbi__bit_reverse(int v, int bits)
{
   STBI_ASSERT(bits <= 16);
   // 要位反转n位，反转16位并移位
   // 例如，11位，位反转并移除5位
   return stbi__bitreverse16(v) >> (16-bits);
}

static int stbi__zbuild_huffman(stbi__zhuffman *z, const stbi_uc *sizelist, int num)
{
   // 定义整型变量i和k，并初始化k为0
   int i,k=0;
   // 定义整型变量code和数组next_code、sizes
   int code, next_code[16], sizes[17];

   // 用0填充sizes数组
   memset(sizes, 0, sizeof(sizes));
   // 用0填充z->fast数组
   memset(z->fast, 0, sizeof(z->fast));
   // 遍历sizelist数组，统计每个元素出现的次数
   for (i=0; i < num; ++i)
      ++sizes[sizelist[i]];
   // 将sizes[0]设置为0
   sizes[0] = 0;
   // 检查sizes数组是否符合DEFLATE规范
   for (i=1; i < 16; ++i)
      if (sizes[i] > (1 << i))
         return stbi__err("bad sizes", "Corrupt PNG");
   // 初始化code为0
   code = 0;
   // 生成Huffman编码
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
   // 设置maxcode[16]为0x10000
   z->maxcode[16] = 0x10000; // sentinel
   // 生成Huffman编码表
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
   // 返回1
   return 1;
}

// 用于PNG读取的内存中的zlib实现
// 因为PNG允许任意分割zlib流，结构上让PNG调用ZLIB再调用PNG很麻烦
// 我们要求PNG读取所有IDAT并将它们合并成单个内存缓冲区

// 定义stbi__zbuf结构体
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

// 内联函数，用于检查zbuf是否已经到达末尾
stbi_inline static int stbi__zeof(stbi__zbuf *z)
{
   return (z->zbuffer >= z->zbuffer_end);
}
# 从输入缓冲区中读取一个字节，如果已经到达缓冲区末尾则返回0
stbi_inline static stbi_uc stbi__zget8(stbi__zbuf *z)
{
   return stbi__zeof(z) ? 0 : *z->zbuffer++;
}

# 填充缓冲区中的位数，直到缓冲区中的位数达到24位或者超过
static void stbi__fill_bits(stbi__zbuf *z)
{
   do {
      if (z->code_buffer >= (1U << z->num_bits)) {
        z->zbuffer = z->zbuffer_end;  /* 将其视为 EOF 以便失败 */
        return;
      }
      z->code_buffer |= (unsigned int) stbi__zget8(z) << z->num_bits;
      z->num_bits += 8;
   } while (z->num_bits <= 24);
}

# 从输入缓冲区中读取 n 位，并返回其值
stbi_inline static unsigned int stbi__zreceive(stbi__zbuf *z, int n)
{
   unsigned int k;
   if (z->num_bits < n) stbi__fill_bits(z);
   k = z->code_buffer & ((1 << n) - 1);
   z->code_buffer >>= n;
   z->num_bits -= n;
   return k;
}

# 通过慢速路径解码哈夫曼编码
static int stbi__zhuffman_decode_slowpath(stbi__zbuf *a, stbi__zhuffman *z)
{
   int b,s,k;
   # 未被快速查找表解析，因此以慢速方式计算
   # 使用 JPEG 方法，要求 MSbits 在顶部
   k = stbi__bit_reverse(a->code_buffer, 16);
   for (s=STBI__ZFAST_BITS+1; ; ++s)
      if (k < z->maxcode[s])
         break;
   if (s >= 16) return -1; # 无效的编码！
   # 编码大小为 s，因此：
   b = (k >> (16-s)) - z->firstcode[s] + z->firstsymbol[s];
   if (b >= STBI__ZNSYMS) return -1; # 某些数据在某处损坏！
   if (z->size[b] != s) return -1;  # 最初是一个断言，但改为报告失败。
   a->code_buffer >>= s;
   a->num_bits -= s;
   return z->value[b];
}

# 解码哈夫曼编码
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

# 扩展输入缓冲区，为 n 个字节腾出空间
static int stbi__zexpand(stbi__zbuf *z, char *zout, int n)
{
   // 定义指向字符的指针变量 q
   char *q;
   // 定义无符号整型变量 cur, limit, old_limit
   unsigned int cur, limit, old_limit;
   // 将 zout 赋值给 z->zout
   z->zout = zout;
   // 如果 z->z_expandable 为假，则返回错误信息 "Corrupt PNG"
   if (!z->z_expandable) return stbi__err("output buffer limit","Corrupt PNG");
   // 将 z->zout - z->zout_start 转换为无符号整型并赋值给 cur
   cur   = (unsigned int) (z->zout - z->zout_start);
   // 将 z->zout_end - z->zout_start 转换为无符号整型并赋值给 limit 和 old_limit
   limit = old_limit = (unsigned) (z->zout_end - z->zout_start);
   // 如果 UINT_MAX - cur 小于 n，则返回错误信息 "Out of memory"
   if (UINT_MAX - cur < (unsigned) n) return stbi__err("outofmem", "Out of memory");
   // 当 cur + n 大于 limit 时执行循环
   while (cur + n > limit) {
      // 如果 limit 大于 UINT_MAX / 2，则返回错误信息 "Out of memory"
      if(limit > UINT_MAX / 2) return stbi__err("outofmem", "Out of memory");
      // 将 limit 增加一倍
      limit *= 2;
   }
   // 重新分配大小为 limit 的内存给 q
   q = (char *) STBI_REALLOC_SIZED(z->zout_start, old_limit, limit);
   // 使用 STBI_NOTUSED 宏处理 old_limit
   STBI_NOTUSED(old_limit);
   // 如果 q 为 NULL，则返回错误信息 "Out of memory"
   if (q == NULL) return stbi__err("outofmem", "Out of memory");
   // 将 q 赋值给 z->zout_start
   z->zout_start = q;
   // 将 q + cur 赋值给 z->zout
   z->zout       = q + cur;
   // 将 q + limit 赋值给 z->zout_end
   z->zout_end   = q + limit;
   // 返回 1
   return 1;
}

// 定义静态常量数组 stbi__zlength_base，初始化为指定的值
static const int stbi__zlength_base[31] = {
   3,4,5,6,7,8,9,10,11,13,
   15,17,19,23,27,31,35,43,51,59,
   67,83,99,115,131,163,195,227,258,0,0 };

// 定义静态常量数组 stbi__zlength_extra，初始化为指定的值
static const int stbi__zlength_extra[31]=
{ 0,0,0,0,0,0,0,0,1,1,1,1,2,2,2,2,3,3,3,3,4,4,4,4,5,5,5,5,0,0,0 };

// 定义静态常量数组 stbi__zdist_base，初始化为指定的值
static const int stbi__zdist_base[32] = { 1,2,3,4,5,7,9,13,17,25,33,49,65,97,129,193,
257,385,513,769,1025,1537,2049,3073,4097,6145,8193,12289,16385,24577,0,0};

// 定义静态常量数组 stbi__zdist_extra，初始化为指定的值
static const int stbi__zdist_extra[32] =
{ 0,0,0,0,1,1,2,2,3,3,4,4,5,5,6,6,7,7,8,8,9,9,10,10,11,11,12,12,13,13};

// 定义函数 stbi__parse_huffman_block，参数为 stbi__zbuf 类型指针 a
static int stbi__parse_huffman_block(stbi__zbuf *a)
{
   // 定义指向字符的指针zout，指向a->zout的地址
   char *zout = a->zout;
   // 无限循环
   for(;;) {
      // 从输入流a中解码出一个值z，使用a->z_length作为参数
      int z = stbi__zhuffman_decode(a, &a->z_length);
      // 如果z小于256
      if (z < 256) {
         // 如果z小于0，返回错误信息"bad huffman code"和"Corrupt PNG"
         if (z < 0) return stbi__err("bad huffman code","Corrupt PNG"); // error in huffman codes
         // 如果zout超出了a->zout_end的范围
         if (zout >= a->zout_end) {
            // 如果无法扩展zout指向的内存空间，返回0
            if (!stbi__zexpand(a, zout, 1)) return 0;
            // 重新指向a->zout
            zout = a->zout;
         }
         // 将z转换成字符类型后存入zout指向的地址，并将zout指针后移
         *zout++ = (char) z;
      } else {
         // 定义指向无符号字符的指针p
         stbi_uc *p;
         // 定义长度len和距离dist
         int len,dist;
         // 如果z等于256
         if (z == 256) {
            // 将a->zout指向zout的地址
            a->zout = zout;
            // 返回1
            return 1;
         }
         // 如果z大于等于286，返回错误信息"bad huffman code"和"Corrupt PNG"
         if (z >= 286) return stbi__err("bad huffman code","Corrupt PNG"); // per DEFLATE, length codes 286 and 287 must not appear in compressed data
         // z减去257
         z -= 257;
         // 根据z的值获取长度len
         len = stbi__zlength_base[z];
         // 如果stbi__zlength_extra[z]不为0，将其加到len上
         if (stbi__zlength_extra[z]) len += stbi__zreceive(a, stbi__zlength_extra[z]);
         // 从输入流a中解码出一个值z，使用a->z_distance作为参数
         z = stbi__zhuffman_decode(a, &a->z_distance);
         // 如果z小于0或者大于等于30，返回错误信息"bad huffman code"和"Corrupt PNG"
         if (z < 0 || z >= 30) return stbi__err("bad huffman code","Corrupt PNG"); // per DEFLATE, distance codes 30 and 31 must not appear in compressed data
         // 根据z的值获取距离dist
         dist = stbi__zdist_base[z];
         // 如果stbi__zdist_extra[z]不为0，将其加到dist上
         if (stbi__zdist_extra[z]) dist += stbi__zreceive(a, stbi__zdist_extra[z]);
         // 如果zout减去a->zout_start的值小于dist，返回错误信息"bad dist"和"Corrupt PNG"
         if (zout - a->zout_start < dist) return stbi__err("bad dist","Corrupt PNG");
         // 如果zout加上len大于a->zout_end，无法扩展zout指向的内存空间，返回0
         if (zout + len > a->zout_end) {
            if (!stbi__zexpand(a, zout, len)) return 0;
            // 重新指向a->zout
            zout = a->zout;
         }
         // 定义指向无符号字符的指针p，指向zout - dist的地址
         p = (stbi_uc *) (zout - dist);
         // 如果dist等于1，表示连续一个字节的重复，常见于图像
         if (dist == 1) { 
            // 获取p指向的值v
            stbi_uc v = *p;
            // 如果len不为0，将v写入zout指向的地址，并重复len次
            if (len) { do *zout++ = v; while (--len); }
         } else {
            // 如果len不为0，将p指向的值写入zout指向的地址，并重复len次
            if (len) { do *zout++ = *p++; while (--len); }
         }
      }
   }
}

// 计算哈夫曼编码
static int stbi__compute_huffman_codes(stbi__zbuf *a)
{
   // 定义长度反扭曲表，用于解压缩
   static const stbi_uc length_dezigzag[19] = { 16,17,18,0,8,7,9,6,10,5,11,4,12,3,13,2,14,1,15 };
   // 定义用于存储编码长度的霍夫曼树
   stbi__zhuffman z_codelength;
   // 定义用于存储长度编码的数组
   stbi_uc lencodes[286+32+137];//padding for maximum single op
   // 定义用于存储编码长度大小的数组
   stbi_uc codelength_sizes[19];
   // 定义变量 i 和 n
   int i,n;

   // 读取 HLIT 和 HDIST 的值
   int hlit  = stbi__zreceive(a,5) + 257;
   int hdist = stbi__zreceive(a,5) + 1;
   // 读取 HCLEN 的值
   int hclen = stbi__zreceive(a,4) + 4;
   // 计算总共需要的编码长度
   int ntot  = hlit + hdist;

   // 将编码长度大小数组初始化为 0
   memset(codelength_sizes, 0, sizeof(codelength_sizes));
   // 循环读取 HCLEN 个值，存储到编码长度大小数组中
   for (i=0; i < hclen; ++i) {
      int s = stbi__zreceive(a,3);
      codelength_sizes[length_dezigzag[i]] = (stbi_uc) s;
   }
   // 构建霍夫曼树
   if (!stbi__zbuild_huffman(&z_codelength, codelength_sizes, 19)) return 0;

   // 初始化 n 为 0，循环直到 n 等于 ntot
   n = 0;
   while (n < ntot) {
      // 解码一个值
      int c = stbi__zhuffman_decode(a, &z_codelength);
      // 检查解码值是否合法
      if (c < 0 || c >= 19) return stbi__err("bad codelengths", "Corrupt PNG");
      // 如果解码值小于 16，则直接存储到 lencodes 数组中
      if (c < 16)
         lencodes[n++] = (stbi_uc) c;
      // 如果解码值大于等于 16，则根据不同情况进行处理
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
         // 检查填充的长度是否合法
         if (ntot - n < c) return stbi__err("bad codelengths", "Corrupt PNG");
         // 填充 lencodes 数组
         memset(lencodes+n, fill, c);
         n += c;
      }
   }
   // 检查解码的总数是否等于 ntot
   if (n != ntot) return stbi__err("bad codelengths","Corrupt PNG");
   // 构建长度和距离的霍夫曼树
   if (!stbi__zbuild_huffman(&a->z_length, lencodes, hlit)) return 0;
   if (!stbi__zbuild_huffman(&a->z_distance, lencodes+hlit, hdist)) return 0;
   return 1;
}

// 解析未压缩的数据块
static int stbi__parse_uncompressed_block(stbi__zbuf *a)
{
   // 定义一个包含4个元素的无符号字符数组header
   stbi_uc header[4];
   // 定义整型变量len、nlen、k
   int len,nlen,k;
   // 如果a->num_bits与7的按位与结果不为0，则调用stbi__zreceive函数，丢弃数据
   if (a->num_bits & 7)
      stbi__zreceive(a, a->num_bits & 7); // discard
   // 将位压缩数据填充到header中
   k = 0;
   while (a->num_bits > 0) {
      header[k++] = (stbi_uc) (a->code_buffer & 255); // suppress MSVC run-time check
      a->code_buffer >>= 8;
      a->num_bits -= 8;
   }
   // 如果a->num_bits小于0，则返回zlib corrupt错误
   if (a->num_bits < 0) return stbi__err("zlib corrupt","Corrupt PNG");
   // 使用正常方式填充header
   while (k < 4)
      header[k++] = stbi__zget8(a);
   // 计算len和nlen的值
   len  = header[1] * 256 + header[0];
   nlen = header[3] * 256 + header[2];
   // 如果nlen不等于(len异或0xffff)，则返回zlib corrupt错误
   if (nlen != (len ^ 0xffff)) return stbi__err("zlib corrupt","Corrupt PNG");
   // 如果a->zbuffer + len大于a->zbuffer_end，则返回read past buffer错误
   if (a->zbuffer + len > a->zbuffer_end) return stbi__err("read past buffer","Corrupt PNG");
   // 如果a->zout + len大于a->zout_end
   if (a->zout + len > a->zout_end)
      // 如果stbi__zexpand函数返回0，则返回0
      if (!stbi__zexpand(a, a->zout, len)) return 0;
   // 将a->zbuffer中的len个字节复制到a->zout中
   memcpy(a->zout, a->zbuffer, len);
   // 更新a->zbuffer和a->zout的值
   a->zbuffer += len;
   a->zout += len;
   // 返回1
   return 1;
}

// 解析zlib头部的函数
static int stbi__parse_zlib_header(stbi__zbuf *a)
{
   // 读取cmf和flg
   int cmf   = stbi__zget8(a);
   int cm    = cmf & 15;
   /* int cinfo = cmf >> 4; */
   int flg   = stbi__zget8(a);
   // 如果到达文件末尾，则返回bad zlib header错误
   if (stbi__zeof(a)) return stbi__err("bad zlib header","Corrupt PNG"); // zlib spec
   // 如果(cm*256+flg) % 31不等于0，则返回bad zlib header错误
   if ((cmf*256+flg) % 31 != 0) return stbi__err("bad zlib header","Corrupt PNG"); // zlib spec
   // 如果flg的第5位为1，则返回no preset dict错误
   if (flg & 32) return stbi__err("no preset dict","Corrupt PNG"); // preset dictionary not allowed in png
   // 如果cm不等于8，则返回bad compression错误
   if (cm != 8) return stbi__err("bad compression","Corrupt PNG"); // DEFLATE required for png
   // 返回1
   // window = 1 << (8 + cinfo)... but who cares, we fully buffer output
   return 1;
}

// 定义stbi__zdefault_length数组
static const stbi_uc stbi__zdefault_length[STBI__ZNSYMS] =
{
   // 定义长度为288的数组，存储Huffman编码的长度
   8,8,8,8,8,8,8,8,8,8,8,8,8,8,8,8, 8,8,8,8,8,8,8,8,8,8,8,8,8,8,8,8,
   8,8,8,8,8,8,8,8,8,8,8,8,8,8,8,8, 8,8,8,8,8,8,8,8,8,8,8,8,8,8,8,8,
   8,8,8,8,8,8,8,8,8,8,8,8,8,8,8,8, 8,8,8,8,8,8,8,8,8,8,8,8,8,8,8,8,
   8,8,8,8,8,8,8,8,8,8,8,8,8,8,8
{
   // 定义变量 final 和 type，用于存储解压缩后的数据和数据类型
   int final, type;
   // 如果需要解析头部信息，则调用 stbi__parse_zlib_header 函数进行解析
   if (parse_header)
      if (!stbi__parse_zlib_header(a)) return 0;
   // 初始化位数为 0，代码缓冲区为 0
   a->num_bits = 0;
   a->code_buffer = 0;
   // 进行解压缩操作
   do {
      // 读取最后一个标志位
      final = stbi__zreceive(a,1);
      // 读取数据类型
      type = stbi__zreceive(a,2);
      // 如果数据类型为 0，则解析未压缩块
      if (type == 0) {
         if (!stbi__parse_uncompressed_block(a)) return 0;
      } else if (type == 3) {
         return 0;
      } else {
         // 如果数据类型为 1，则使用固定的代码长度
         if (type == 1) {
            // 使用固定的代码长度构建哈夫曼树
            if (!stbi__zbuild_huffman(&a->z_length  , stbi__zdefault_length  , STBI__ZNSYMS)) return 0;
            if (!stbi__zbuild_huffman(&a->z_distance, stbi__zdefault_distance,  32)) return 0;
         } else {
            // 否则计算哈夫曼编码
            if (!stbi__compute_huffman_codes(a)) return 0;
         }
         // 解析哈夫曼块
         if (!stbi__parse_huffman_block(a)) return 0;
      }
   } while (!final);
   // 返回解压缩结果
   return 1;
}

// 执行 zlib 解压缩
static int stbi__do_zlib(stbi__zbuf *a, char *obuf, int olen, int exp, int parse_header)
{
   // 设置输出缓冲区的起始位置、当前位置和结束位置，以及是否可扩展
   a->zout_start = obuf;
   a->zout       = obuf;
   a->zout_end   = obuf + olen;
   a->z_expandable = exp;

   // 调用 stbi__parse_zlib 函数进行 zlib 解压缩
   return stbi__parse_zlib(a, parse_header);
}

// 根据给定的大小猜测 zlib 解压缩后的数据大小，并分配内存
STBIDEF char *stbi_zlib_decode_malloc_guesssize(const char *buffer, int len, int initial_size, int *outlen)
{
   // 初始化 stbi__zbuf 结构体
   stbi__zbuf a;
   // 分配初始大小的内存
   char *p = (char *) stbi__malloc(initial_size);
   // 如果内存分配失败，则返回 NULL
   if (p == NULL) return NULL;
   // 设置输入缓冲区和结束位置
   a.zbuffer = (stbi_uc *) buffer;
   a.zbuffer_end = (stbi_uc *) buffer + len;
   // 进行 zlib 解压缩
   if (stbi__do_zlib(&a, p, initial_size, 1, 1)) {
      // 如果需要输出解压缩后的数据大小，则设置 outlen
      if (outlen) *outlen = (int) (a.zout - a.zout_start);
      // 返回解压缩后的数据起始位置
      return a.zout_start;
   } else {
      // 如果解压缩失败，则释放内存并返回 NULL
      STBI_FREE(a.zout_start);
      return NULL;
   }
}

// 根据给定的大小进行 zlib 解压缩，并分配内存
STBIDEF char *stbi_zlib_decode_malloc(char const *buffer, int len, int *outlen)
{
   return stbi_zlib_decode_malloc_guesssize(buffer, len, 16384, outlen);
}

// 根据给定的大小和解析头部标志进行 zlib 解压缩，并分配内存
STBIDEF char *stbi_zlib_decode_malloc_guesssize_headerflag(const char *buffer, int len, int initial_size, int *outlen, int parse_header)
// 定义一个结构体变量a，用于存储zlib解压缩的缓冲区
   stbi__zbuf a;
   // 分配初始大小的内存空间，并将其转换为char指针p
   char *p = (char *) stbi__malloc(initial_size);
   // 如果内存分配失败，则返回NULL
   if (p == NULL) return NULL;
   // 将buffer强制类型转换为stbi_uc类型，并赋值给a的zbuffer
   a.zbuffer = (stbi_uc *) buffer;
   // 将buffer的末尾位置赋值给a的zbuffer_end
   a.zbuffer_end = (stbi_uc *) buffer + len;
   // 调用stbi__do_zlib函数进行zlib解压缩，并根据parse_header参数判断是否解析头部信息
   if (stbi__do_zlib(&a, p, initial_size, 1, parse_header)) {
      // 如果outlen不为空，则将解压后的数据长度赋值给outlen
      if (outlen) *outlen = (int) (a.zout - a.zout_start);
      // 返回解压后的数据起始地址
      return a.zout_start;
   } else {
      // 释放a的zout_start指向的内存空间
      STBI_FREE(a.zout_start);
      // 返回NULL
      return NULL;
   }
}

// 定义一个函数，用于对输入的ibuffer进行zlib解压缩，并将结果存储到obuffer中
STBIDEF int stbi_zlib_decode_buffer(char *obuffer, int olen, char const *ibuffer, int ilen)
{
   // 定义一个结构体变量a，用于存储zlib解压缩的缓冲区
   stbi__zbuf a;
   // 将ibuffer强制类型转换为stbi_uc类型，并赋值给a的zbuffer
   a.zbuffer = (stbi_uc *) ibuffer;
   // 将ibuffer的末尾位置赋值给a的zbuffer_end
   a.zbuffer_end = (stbi_uc *) ibuffer + ilen;
   // 调用stbi__do_zlib函数进行zlib解压缩，并将结果存储到obuffer中
   if (stbi__do_zlib(&a, obuffer, olen, 0, 1))
      // 返回解压后的数据长度
      return (int) (a.zout - a.zout_start);
   else
      // 返回-1，表示解压失败
      return -1;
}

// 定义一个函数，用于对输入的buffer进行zlib解压缩，并返回解压后的数据起始地址
STBIDEF char *stbi_zlib_decode_noheader_malloc(char const *buffer, int len, int *outlen)
{
   // 定义一个结构体变量a，用于存储zlib解压缩的缓冲区
   stbi__zbuf a;
   // 分配16384大小的内存空间，并将其转换为char指针p
   char *p = (char *) stbi__malloc(16384);
   // 如果内存分配失败，则返回NULL
   if (p == NULL) return NULL;
   // 将buffer强制类型转换为stbi_uc类型，并赋值给a的zbuffer
   a.zbuffer = (stbi_uc *) buffer;
   // 将buffer的末尾位置赋值给a的zbuffer_end
   a.zbuffer_end = (stbi_uc *) buffer+len;
   // 调用stbi__do_zlib函数进行zlib解压缩，并根据0参数判断是否解析头部信息
   if (stbi__do_zlib(&a, p, 16384, 1, 0)) {
      // 如果outlen不为空，则将解压后的数据长度赋值给outlen
      if (outlen) *outlen = (int) (a.zout - a.zout_start);
      // 返回解压后的数据起始地址
      return a.zout_start;
   } else {
      // 释放a的zout_start指向的内存空间
      STBI_FREE(a.zout_start);
      // 返回NULL
      return NULL;
   }
}

// 定义一个函数，用于对输入的ibuffer进行zlib解压缩，并将结果存储到obuffer中
STBIDEF int stbi_zlib_decode_noheader_buffer(char *obuffer, int olen, const char *ibuffer, int ilen)
{
   // 定义一个结构体变量a，用于存储zlib解压缩的缓冲区
   stbi__zbuf a;
   // 将ibuffer强制类型转换为stbi_uc类型，并赋值给a的zbuffer
   a.zbuffer = (stbi_uc *) ibuffer;
   // 将ibuffer的末尾位置赋值给a的zbuffer_end
   a.zbuffer_end = (stbi_uc *) ibuffer + ilen;
   // 调用stbi__do_zlib函数进行zlib解压缩，并将结果存储到obuffer中
   if (stbi__do_zlib(&a, obuffer, olen, 0, 0))
      // 返回解压后的数据长度
      return (int) (a.zout - a.zout_start);
   else
      // 返回-1，表示解压失败
      return -1;
}
#endif

// 定义一个结构体，用于存储PNG文件的长度和类型信息
typedef struct
{
   stbi__uint32 length;
   stbi__uint32 type;
# 定义 stbi__pngchunk 结构体
} stbi__pngchunk;

# 从 stbi__context 中获取 PNG 数据块头信息
static stbi__pngchunk stbi__get_chunk_header(stbi__context *s)
{
   stbi__pngchunk c;
   # 读取数据块长度
   c.length = stbi__get32be(s);
   # 读取数据块类型
   c.type   = stbi__get32be(s);
   return c;
}

# 检查 PNG 文件头部是否正确
static int stbi__check_png_header(stbi__context *s)
{
   # PNG 文件的签名
   static const stbi_uc png_sig[8] = { 137,80,78,71,13,10,26,10 };
   int i;
   # 遍历 PNG 文件的签名
   for (i=0; i < 8; ++i)
      # 检查签名是否匹配
      if (stbi__get8(s) != png_sig[i]) return stbi__err("bad png sig","Not a PNG");
   return 1;
}

# 定义 stbi__png 结构体
typedef struct
{
   stbi__context *s;
   stbi_uc *idata, *expanded, *out;
   int depth;
} stbi__png;

# 定义 PNG 数据扫描时使用的滤波器类型
enum {
   STBI__F_none=0,
   STBI__F_sub=1,
   STBI__F_up=2,
   STBI__F_avg=3,
   STBI__F_paeth=4,
   # 用于第一行扫描的合成滤波器，避免需要一个全为0的虚拟行
   STBI__F_avg_first,
   STBI__F_paeth_first
};

# 定义第一行扫描时使用的滤波器类型
static stbi_uc first_row_filter[5] =
{
   STBI__F_none,
   STBI__F_sub,
   STBI__F_none,
   STBI__F_avg_first,
   STBI__F_paeth_first
};

# 计算 Paeth 滤波器的值
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

# 定义深度缩放表
static const stbi_uc stbi__depth_scale_table[9] = { 0, 0xff, 0x55, 0, 0x11, 0,0,0, 0x01 };

# 从压缩后的数据创建 PNG 图像数据
static int stbi__create_png_image_raw(stbi__png *a, stbi_uc *raw, stbi__uint32 raw_len, int out_n, stbi__uint32 x, stbi__uint32 y, int depth, int color)
}

# 创建 PNG 图像数据
static int stbi__create_png_image(stbi__png *a, stbi_uc *image_data, stbi__uint32 image_data_len, int out_n, int depth, int color, int interlaced)
{
   # 根据深度选择每个像素的字节数
   int bytes = (depth == 16 ? 2 : 1);
   # 计算输出像素的字节数
   int out_bytes = out_n * bytes;
   # 定义最终输出的像素数据
   stbi_uc *final;
   int p;
   # 如果图像没有交错，则直接调用函数创建 PNG 图像
   if (!interlaced)
      return stbi__create_png_image_raw(a, image_data, image_data_len, out_n, a->s->img_x, a->s->img_y, depth, color);

   # 对图像进行去交错处理
   final = (stbi_uc *) stbi__malloc_mad3(a->s->img_x, a->s->img_y, out_bytes, 0);
   # 如果内存分配失败，则返回错误信息
   if (!final) return stbi__err("outofmem", "Out of memory");
   for (p=0; p < 7; ++p) {
      int xorig[] = { 0,4,0,2,0,1,0 };
      int yorig[] = { 0,0,4,0,2,0,1 };
      int xspc[]  = { 8,8,4,4,2,2,1 };
      int yspc[]  = { 8,8,8,4,4,2,2 };
      int i,j,x,y;
      # 计算每个通道的起始位置和间隔
      x = (a->s->img_x - xorig[p] + xspc[p]-1) / xspc[p];
      y = (a->s->img_y - yorig[p] + yspc[p]-1) / yspc[p];
      # 如果 x 和 y 都大于 0，则进行处理
      if (x && y) {
         # 计算图像数据的长度
         stbi__uint32 img_len = ((((a->s->img_n * x * depth) + 7) >> 3) + 1) * y;
         # 如果创建 PNG 图像失败，则释放内存并返回错误信息
         if (!stbi__create_png_image_raw(a, image_data, image_data_len, out_n, x, y, depth, color)) {
            STBI_FREE(final);
            return 0;
         }
         # 对每个像素进行处理，将去交错后的像素数据复制到最终输出中
         for (j=0; j < y; ++j) {
            for (i=0; i < x; ++i) {
               int out_y = j*yspc[p]+yorig[p];
               int out_x = i*xspc[p]+xorig[p];
               memcpy(final + out_y*a->s->img_x*out_bytes + out_x*out_bytes,
                      a->out + (j*x+i)*out_bytes, out_bytes);
            }
         }
         # 释放内存并更新图像数据指针和长度
         STBI_FREE(a->out);
         image_data += img_len;
         image_data_len -= img_len;
      }
   }
   # 更新输出像素数据
   a->out = final;

   return 1;
}

static int stbi__compute_transparency(stbi__png *z, stbi_uc tc[3], int out_n)
{
   // 获取输入的上下文指针
   stbi__context *s = z->s;
   // 定义变量 i 和像素数量，计算像素数量
   stbi__uint32 i, pixel_count = s->img_x * s->img_y;
   // 获取输出指针
   stbi_uc *p = z->out;

   // 计算基于颜色的透明度，假设输出中已经有 255 作为 alpha 值
   STBI_ASSERT(out_n == 2 || out_n == 4);

   // 如果输出通道数为 2
   if (out_n == 2) {
      // 遍历每个像素
      for (i=0; i < pixel_count; ++i) {
         // 如果当前像素的第一个通道值等于指定颜色的第一个通道值，则将第二个通道值设为 0，否则设为 255
         p[1] = (p[0] == tc[0] ? 0 : 255);
         p += 2;
      }
   } else {
      // 如果输出通道数不为 2
      for (i=0; i < pixel_count; ++i) {
         // 如果当前像素的 RGB 值等于指定颜色的 RGB 值，则将 alpha 通道值设为 0
         p[3] = 0;
         p += 4;
      }
   }
   // 返回 1 表示处理成功
   return 1;
}

static int stbi__compute_transparency16(stbi__png *z, stbi__uint16 tc[3], int out_n)
{
   // 获取输入的上下文指针
   stbi__context *s = z->s;
   // 定义变量 i 和像素数量，计算像素数量
   stbi__uint32 i, pixel_count = s->img_x * s->img_y;
   // 获取输出指针
   stbi__uint16 *p = (stbi__uint16*) z->out;

   // 计算基于颜色的透明度，假设输出中已经有 65535 作为 alpha 值
   STBI_ASSERT(out_n == 2 || out_n == 4);

   // 如果输出通道数为 2
   if (out_n == 2) {
      // 遍历每个像素
      for (i = 0; i < pixel_count; ++i) {
         // 如果当前像素的第一个通道值等于指定颜色的第一个通道值，则将第二个通道值设为 0，否则设为 65535
         p[1] = (p[0] == tc[0] ? 0 : 65535);
         p += 2;
      }
   } else {
      // 如果输出通道数不为 2
      for (i = 0; i < pixel_count; ++i) {
         // 如果当前像素的 RGB 值等于指定颜色的 RGB 值，则将 alpha 通道值设为 0
         p[3] = 0;
         p += 4;
      }
   }
   // 返回 1 表示处理成功
   return 1;
}

static int stbi__expand_png_palette(stbi__png *a, stbi_uc *palette, int len, int pal_img_n)
{
   # 定义变量 i，计算像素总数
   stbi__uint32 i, pixel_count = a->s->img_x * a->s->img_y;
   # 定义指针变量 p、temp_out、orig，并将 a->out 赋值给 orig
   stbi_uc *p, *temp_out, *orig = a->out;

   # 分配内存给 p，大小为 pixel_count * pal_img_n
   p = (stbi_uc *) stbi__malloc_mad2(pixel_count, pal_img_n, 0);
   # 如果分配内存失败，则返回错误信息 "Out of memory"
   if (p == NULL) return stbi__err("outofmem", "Out of memory");

   # 将 p 赋值给 temp_out
   temp_out = p;

   # 根据 pal_img_n 的值，将原始数据 orig 转换为 RGB 数据存储在 p 中
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
   # 释放 a->out 指向的内存
   STBI_FREE(a->out);
   # 将 temp_out 赋值给 a->out
   a->out = temp_out;

   # 忽略参数 len
   STBI_NOTUSED(len);

   # 返回 1，表示成功
   return 1;
}

# 定义并初始化全局变量 stbi__unpremultiply_on_load_global 和 stbi__de_iphone_flag_global
static int stbi__unpremultiply_on_load_global = 0;
static int stbi__de_iphone_flag_global = 0;

# 设置是否在加载时取消预乘的全局变量
STBIDEF void stbi_set_unpremultiply_on_load(int flag_true_if_should_unpremultiply)
{
   stbi__unpremultiply_on_load_global = flag_true_if_should_unpremultiply;
}

# 设置是否将 iPhone PNG 转换为 RGB 的全局变量
STBIDEF void stbi_convert_iphone_png_to_rgb(int flag_true_if_should_convert)
{
   stbi__de_iphone_flag_global = flag_true_if_should_convert;
}

# 如果未定义 STBI_THREAD_LOCAL，则使用全局变量 stbi__unpremultiply_on_load_global 和 stbi__de_iphone_flag_global
#ifndef STBI_THREAD_LOCAL
#define stbi__unpremultiply_on_load  stbi__unpremultiply_on_load_global
#define stbi__de_iphone_flag  stbi__de_iphone_flag_global
#否则，使用线程本地存储的变量
#else
static STBI_THREAD_LOCAL int stbi__unpremultiply_on_load_local, stbi__unpremultiply_on_load_set;
static STBI_THREAD_LOCAL int stbi__de_iphone_flag_local, stbi__de_iphone_flag_set;

# 设置是否在加载时取消预乘的线程本地存储变量
STBIDEF void stbi_set_unpremultiply_on_load_thread(int flag_true_if_should_unpremultiply)
{
   stbi__unpremultiply_on_load_local = flag_true_if_should_unpremultiply;
   stbi__unpremultiply_on_load_set = 1;
}

# 设置是否将 iPhone PNG 转换为 RGB 的线程本地存储变量
STBIDEF void stbi_convert_iphone_png_to_rgb_thread(int flag_true_if_should_convert)
{
   stbi__de_iphone_flag_local = flag_true_if_should_convert;
   stbi__de_iphone_flag_set = 1;
}
// 定义 stbi__unpremultiply_on_load 宏，根据条件选择本地或全局变量
#define stbi__unpremultiply_on_load  (stbi__unpremultiply_on_load_set           \
                                       ? stbi__unpremultiply_on_load_local      \
                                       : stbi__unpremultiply_on_load_global)
// 定义 stbi__de_iphone_flag 宏，根据条件选择本地或全局变量
#define stbi__de_iphone_flag  (stbi__de_iphone_flag_set                         \
                                ? stbi__de_iphone_flag_local                    \
                                : stbi__de_iphone_flag_global)
#endif // STBI_THREAD_LOCAL

// 定义 stbi__de_iphone 函数，处理 iPhone 图片格式
static void stbi__de_iphone(stbi__png *z)
{
   // 获取 PNG 图片的上下文
   stbi__context *s = z->s;
   // 初始化像素计数和输出指针
   stbi__uint32 i, pixel_count = s->img_x * s->img_y;
   stbi_uc *p = z->out;

   // 如果输出通道数为 3，将 bgr 转换为 rgb
   if (s->img_out_n == 3) {
      for (i=0; i < pixel_count; ++i) {
         stbi_uc t = p[0];
         p[0] = p[2];
         p[2] = t;
         p += 3;
      }
   } else {
      // 断言输出通道数为 4
      STBI_ASSERT(s->img_out_n == 4);
      // 如果需要在加载时取消预乘，将 bgr 转换为 rgb 并取消预乘
      if (stbi__unpremultiply_on_load) {
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
         // 如果不需要取消预乘，将 bgr 转换为 rgb
         for (i=0; i < pixel_count; ++i) {
            stbi_uc t = p[0];
            p[0] = p[2];
            p[2] = t;
            p += 4;
         }
      }
   }
}

// 定义 STBI__PNG_TYPE 宏，将四个字符转换为无符号整数
#define STBI__PNG_TYPE(a,b,c,d)  (((unsigned) (a) << 24) + ((unsigned) (b) << 16) + ((unsigned) (c) << 8) + (unsigned) (d))

// 解析 PNG 文件
static int stbi__parse_png_file(stbi__png *z, int scan, int req_comp)
}

// 执行 PNG 解码
static void *stbi__do_png(stbi__png *p, int *x, int *y, int *n, int req_comp, stbi__result_info *ri)
{
   void *result=NULL;  // 声明一个空指针变量result
   if (req_comp < 0 || req_comp > 4) return stbi__errpuc("bad req_comp", "Internal error");  // 如果请求的通道数小于0或大于4，则返回错误信息
   if (stbi__parse_png_file(p, STBI__SCAN_load, req_comp)) {  // 调用解析PNG文件的函数，加载指定通道数的图像数据
      if (p->depth <= 8)  // 如果图像深度小于等于8
         ri->bits_per_channel = 8;  // 设置结果信息结构体中每通道的位数为8
      else if (p->depth == 16)  // 如果图像深度为16
         ri->bits_per_channel = 16;  // 设置结果信息结构体中每通道的位数为16
      else
         return stbi__errpuc("bad bits_per_channel", "PNG not supported: unsupported color depth");  // 返回不支持的颜色深度错误信息
      result = p->out;  // 将解析得到的图像数据赋值给result
      p->out = NULL;  // 将解析得到的图像数据指针置空
      if (req_comp && req_comp != p->s->img_out_n) {  // 如果请求的通道数不为0且不等于解析得到的图像数据通道数
         if (ri->bits_per_channel == 8)  // 如果每通道的位数为8
            result = stbi__convert_format((unsigned char *) result, p->s->img_out_n, req_comp, p->s->img_x, p->s->img_y);  // 转换图像数据的通道数
         else
            result = stbi__convert_format16((stbi__uint16 *) result, p->s->img_out_n, req_comp, p->s->img_x, p->s->img_y);  // 转换图像数据的通道数
         p->s->img_out_n = req_comp;  // 更新解析得到的图像数据通道数
         if (result == NULL) return result;  // 如果转换后的图像数据为空，则返回空指针
      }
      *x = p->s->img_x;  // 将解析得到的图像宽度赋值给x
      *y = p->s->img_y;  // 将解析得到的图像高度赋值给y
      if (n) *n = p->s->img_n;  // 如果n不为空，则将解析得到的图像通道数赋值给n
   }
   STBI_FREE(p->out);      p->out      = NULL;  // 释放解析得到的图像数据内存并将指针置空
   STBI_FREE(p->expanded); p->expanded = NULL;  // 释放扩展后的图像数据内存并将指针置空
   STBI_FREE(p->idata);    p->idata    = NULL;  // 释放图像数据内存并将指针置空

   return result;  // 返回解析得到的图像数据
}

static void *stbi__png_load(stbi__context *s, int *x, int *y, int *comp, int req_comp, stbi__result_info *ri)
{
   stbi__png p;  // 声明一个PNG结构体变量p
   p.s = s;  // 将传入的上下文指针赋值给p的成员变量s
   return stbi__do_png(&p, x,y,comp,req_comp, ri);  // 调用处理PNG图像的函数
}

static int stbi__png_test(stbi__context *s)
{
   int r;  // 声明一个整型变量r
   r = stbi__check_png_header(s);  // 调用检查PNG文件头的函数
   stbi__rewind(s);  // 重置上下文指针
   return r;  // 返回检查结果
}

static int stbi__png_info_raw(stbi__png *p, int *x, int *y, int *comp)
{
   if (!stbi__parse_png_file(p, STBI__SCAN_header, 0)) {  // 如果解析PNG文件失败
      stbi__rewind( p->s );  // 重置上下文指针
      return 0;  // 返回失败
   }
   if (x) *x = p->s->img_x;  // 如果x不为空，则将图像宽度赋值给x
   if (y) *y = p->s->img_y;  // 如果y不为空，则将图像高度赋值给y
   if (comp) *comp = p->s->img_n;  // 如果comp不为空，则将图像通道数赋值给comp
   return 1;  // 返回成功
}

static int stbi__png_info(stbi__context *s, int *x, int *y, int *comp)
{
   stbi__png p;  // 声明一个PNG结构体变量p
   p.s = s;  // 将传入的上下文指针赋值给p的成员变量s
   return stbi__png_info_raw(&p, x, y, comp);  // 调用处理PNG图像信息的函数
}

static int stbi__png_is16(stbi__context *s)
{
   // 创建 stbi__png 结构体实例 p
   stbi__png p;
   // 将参数 s 赋值给 p 的成员变量 s
   p.s = s;
   // 如果调用 stbi__png_info_raw 函数返回假，则返回 0
   if (!stbi__png_info_raw(&p, NULL, NULL, NULL))
       return 0;
   // 如果 p 的深度不等于 16，则将 s 指针回退并返回 0
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
// 检测 BMP 图像格式的原始函数
static int stbi__bmp_test_raw(stbi__context *s)
{
   int r;
   int sz;
   // 如果读取的第一个字节不是 'B'，则返回 0
   if (stbi__get8(s) != 'B') return 0;
   // 如果读取的第二个字节不是 'M'，则返回 0
   if (stbi__get8(s) != 'M') return 0;
   // 读取 32 位小端序的值并丢弃，表示文件大小
   stbi__get32le(s); // discard filesize
   // 读取 16 位小端序的值并丢弃，表示保留字段
   stbi__get16le(s); // discard reserved
   // 读取 16 位小端序的值并丢弃，表示保留字段
   stbi__get16le(s); // discard reserved
   // 读取 32 位小端序的值并丢弃，表示数据偏移
   stbi__get32le(s); // discard data offset
   // 读取 32 位小端序的值，表示头部大小
   sz = stbi__get32le(s);
   // 判断头部大小是否符合 BMP 格式的标准，如果符合则返回 1，否则返回 0
   r = (sz == 12 || sz == 40 || sz == 56 || sz == 108 || sz == 124);
   return r;
}

// 检测 BMP 图像格式的函数
static int stbi__bmp_test(stbi__context *s)
{
   // 调用 stbi__bmp_test_raw 函数进行检测
   int r = stbi__bmp_test_raw(s);
   // 将指针回退到起始位置
   stbi__rewind(s);
   // 返回检测结果
   return r;
}


// 返回 z 中最高位的索引（0 到 31）
static int stbi__high_bit(unsigned int z)
{
   int n=0;
   // 如果 z 为 0，则返回 -1
   if (z == 0) return -1;
   // 如果 z 大于等于 0x10000，则 n 加 16，z 右移 16 位
   if (z >= 0x10000) { n += 16; z >>= 16; }
   // 如果 z 大于等于 0x00100，则 n 加 8，z 右移 8 位
   if (z >= 0x00100) { n +=  8; z >>=  8; }
   // 如果 z 大于等于 0x00010，则 n 加 4，z 右移 4 位
   if (z >= 0x00010) { n +=  4; z >>=  4; }
   // 如果 z 大于等于 0x00004，则 n 加 2，z 右移 2 位
   if (z >= 0x00004) { n +=  2; z >>=  2; }
   // 如果 z 大于等于 0x00002，则 n 加 1
   if (z >= 0x00002) { n +=  1;/* >>=  1;*/ }
   // 返回最高位的索引
   return n;
}

// 计算 a 中位为 1 的个数
static int stbi__bitcount(unsigned int a)
{
   // 将 a 与 0x55555555 按位与，然后加上 a 右移 1 位与 0x55555555 按位与的结果
   a = (a & 0x55555555) + ((a >>  1) & 0x55555555); // max 2
   // 将 a 与 0x33333333 按位与，然后加上 a 右移 2 位与 0x33333333 按位与的结果
   a = (a & 0x33333333) + ((a >>  2) & 0x33333333); // max 4
   // 将 a 加上 a 右移 4 位的结果，然后与 0x0f0f0f0f 按位与
   a = (a + (a >> 4)) & 0x0f0f0f0f; // max 8 per 4, now 8 bits
   // 将 a 加上 a 右移 8 位的结果，然后与 0xff 按位与
   a = (a + (a >> 8)); // max 16 per 8 bits
   // 将 a 加上 a 右移 16 位的结果，然后与 0xff 按位与
   a = (a + (a >> 16)); // max 32 per 8 bits
   // 返回结果与 0xff 的按位与
   return a & 0xff;
}

// 从 v 中提取 N 位的值（N=bits），然后将其扩展为 8 位长并进行符号扩展
static int stbi__shiftsigned(unsigned int v, int shift, int bits)
{
   // 定义一个静态的无符号整型数组，存储乘法表
   static unsigned int mul_table[9] = {
      0,
      0xff/*0b11111111*/, 0x55/*0b01010101*/, 0x49/*0b01001001*/, 0x11/*0b00010001*/,
      0x21/*0b00100001*/, 0x41/*0b01000001*/, 0x81/*0b10000001*/, 0x01/*0b00000001*/,
   };
   // 定义一个静态的无符号整型数组，存储位移表
   static unsigned int shift_table[9] = {
      0, 0,0,1,0,2,4,6,0,
   };
   // 如果位移小于0，则左移-v
   if (shift < 0)
      v <<= -shift;
   else
      v >>= shift;
   // 断言v小于256
   STBI_ASSERT(v < 256);
   // 右移(8-bits)位
   v >>= (8-bits);
   // 断言bits大于等于0且小于等于8
   STBI_ASSERT(bits >= 0 && bits <= 8);
   // 返回 (int) ((unsigned) v * mul_table[bits]) >> shift_table[bits]
   return (int) ((unsigned) v * mul_table[bits]) >> shift_table[bits];
}

// 定义一个结构体 stbi__bmp_data
typedef struct
{
   int bpp, offset, hsz;
   unsigned int mr,mg,mb,ma, all_a;
   int extra_read;
} stbi__bmp_data;

// 设置位掩码的默认值
static int stbi__bmp_set_mask_defaults(stbi__bmp_data *info, int compress)
{
   // 如果压缩为3，则BI_BITFIELDS指定了掩码，不覆盖
   if (compress == 3)
      return 1;

   // 如果压缩为0
   if (compress == 0) {
      // 如果 bpp 为 16
      if (info->bpp == 16) {
         info->mr = 31u << 10;
         info->mg = 31u <<  5;
         info->mb = 31u <<  0;
      } else if (info->bpp == 32) {
         info->mr = 0xffu << 16;
         info->mg = 0xffu <<  8;
         info->mb = 0xffu <<  0;
         info->ma = 0xffu << 24;
         info->all_a = 0; // 如果 all_a 在最后为0，则加载了alpha通道，但全部为0
      } else {
         // 否则，使用默认值，即全部为0
         info->mr = info->mg = info->mb = info->ma = 0;
      }
      return 1;
   }
   return 0; // 错误
}

// 解析头部信息
static void *stbi__bmp_parse_header(stbi__context *s, stbi__bmp_data *info)
}


// 加载TGA格式的图片
static void *stbi__bmp_load(stbi__context *s, int *x, int *y, int *comp, int req_comp, stbi__result_info *ri)
}
#endif

// Targa Truevision - TGA
// by Jonathan Dummer
#ifndef STBI_NO_TGA
// 根据像素位数和是否为灰度图获取组件数，如果是RGB16则返回1，否则返回0
static int stbi__tga_get_comp(int bits_per_pixel, int is_grey, int* is_rgb16)
{
   // 只允许 RGB 或 RGBA（包括16位）或灰度
   if (is_rgb16) *is_rgb16 = 0;  // 如果是16位RGB，则将is_rgb16设置为0
   switch(bits_per_pixel) {  // 根据像素位数进行判断
      case 8:  return STBI_grey;  // 8位像素返回灰度
      case 16: if(is_grey) return STBI_grey_alpha;  // 如果是16位像素且是灰度，则返回灰度+Alpha
               // 继续执行下面的代码
      case 15: if(is_rgb16) *is_rgb16 = 1;  // 如果是15位像素且是16位RGB，则将is_rgb16设置为1
               return STBI_rgb;  // 返回RGB
      case 24: // 继续执行下面的代码
      case 32: return bits_per_pixel/8;  // 返回每像素的字节数
      default: return 0;  // 默认返回0
   }
}

static int stbi__tga_info(stbi__context *s, int *x, int *y, int *comp)
{
    int tga_w, tga_h, tga_comp, tga_image_type, tga_bits_per_pixel, tga_colormap_bpp;
    int sz, tga_colormap_type;
    stbi__get8(s);                   // 丢弃偏移量
    tga_colormap_type = stbi__get8(s); // 颜色映射类型
    if( tga_colormap_type > 1 ) {
        stbi__rewind(s);
        return 0;      // 只允许RGB或索引颜色映射
    }
    tga_image_type = stbi__get8(s); // 图像类型
    if ( tga_colormap_type == 1 ) { // 有颜色映射的图像
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
        stbi__skip(s,4);       // 跳过图像的x和y原点
        tga_colormap_bpp = sz;
    } else { // 没有颜色映射的“普通”图像 - 只允许RGB或灰度，+/- RLE
        if ( (tga_image_type != 2) && (tga_image_type != 3) && (tga_image_type != 10) && (tga_image_type != 11) ) {
            stbi__rewind(s);
            return 0; // 只允许RGB或灰度，+/- RLE
        }
        stbi__skip(s,9); // 跳过颜色映射规范和图像x/y原点
        tga_colormap_bpp = 0;
    }
    tga_w = stbi__get16le(s);
    if( tga_w < 1 ) {
        stbi__rewind(s);
        return 0;   // 测试宽度
    }
    # 读取 TGA 文件的高度
    tga_h = stbi__get16le(s);
    # 如果高度小于1，则回退到文件开头并返回0
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
            # 回退到文件开头并返回0
            stbi__rewind(s);
            return 0;
        }
        # 根据颜色映射表的位数和类型获取颜色通道数
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
    # 如果传入了 x 参数，则将 tga_w 赋值给 x
    if (x) *x = tga_w;
    # 如果传入了 y 参数，则将 tga_h 赋值给 y
    if (y) *y = tga_h;
    # 如果传入了 comp 参数，则将 tga_comp 赋值给 comp
    if (comp) *comp = tga_comp;
    # 通过所有测试，返回1
    return 1;                   // seems to have passed everything
# 检查 TGA 文件是否符合格式要求，返回结果
static int stbi__tga_test(stbi__context *s)
{
   int res = 0;  // 初始化结果为 0
   int sz, tga_color_type;  // 定义变量 sz 和 tga_color_type
   stbi__get8(s);      //   丢弃 Offset
   tga_color_type = stbi__get8(s);   //   读取颜色类型
   if ( tga_color_type > 1 ) goto errorEnd;   //   只允许 RGB 或索引颜色类型
   sz = stbi__get8(s);   //   读取图像类型
   if ( tga_color_type == 1 ) { // 调色板（索引）图像
      if (sz != 1 && sz != 9) goto errorEnd; // 调色板类型 1 需要图像类型 1 或 9
      stbi__skip(s,4);       // 跳过第一个调色板条目的索引和条目数
      sz = stbi__get8(s);    //   检查每个调色板颜色条目的位数
      if ( (sz != 8) && (sz != 15) && (sz != 16) && (sz != 24) && (sz != 32) ) goto errorEnd;
      stbi__skip(s,4);       // 跳过图像 x 和 y 的起始位置
   } else { // "正常" 图像没有调色板
      if ( (sz != 2) && (sz != 3) && (sz != 10) && (sz != 11) ) goto errorEnd; // 只允许 RGB 或灰度，可能带 RLE 压缩
      stbi__skip(s,9); // 跳过调色板规范和图像 x/y 的起始位置
   }
   if ( stbi__get16le(s) < 1 ) goto errorEnd;      //   检查宽度
   if ( stbi__get16le(s) < 1 ) goto errorEnd;      //   检查高度
   sz = stbi__get8(s);   //   每个像素的位数
   if ( (tga_color_type == 1) && (sz != 8) && (sz != 16) ) goto errorEnd; // 对于调色板图像，位数是索引的大小
   if ( (sz != 8) && (sz != 15) && (sz != 16) && (sz != 24) && (sz != 32) ) goto errorEnd;

   res = 1; // 如果执行到这里，一切正常，返回 1 而不是 0

errorEnd:
   stbi__rewind(s);  // 回到文件开始位置
   return res;  // 返回结果
}

// 读取 16 位值并转换为 24 位 RGB
static void stbi__tga_read_rgb16(stbi__context *s, stbi_uc* out)
{
   // 从输入流中读取16位无符号整数
   stbi__uint16 px = (stbi__uint16)stbi__get16le(s);
   // 定义一个5位掩码
   stbi__uint16 fiveBitMask = 31;
   // 由于有3个通道，每个通道占5位
   int r = (px >> 10) & fiveBitMask;
   int g = (px >> 5) & fiveBitMask;
   int b = px & fiveBitMask;
   // 注意：这里保存的数据是按照 RGB(A) 的顺序，因此后续不需要交换顺序
   out[0] = (stbi_uc)((r * 255)/31);
   out[1] = (stbi_uc)((g * 255)/31);
   out[2] = (stbi_uc)((b * 255)/31);

   // 有些人声称最高位可能用于 alpha 通道
   // （可能是在“图像描述字节”中设置了 alpha 位）
   // 但这样做只会使16位测试图像完全透明..
   // 因此，让我们将所有15和16位的 TGA 图像视为没有 alpha 通道的 RGB 图像。
}

static void *stbi__tga_load(stbi__context *s, int *x, int *y, int *comp, int req_comp, stbi__result_info *ri)
}
#endif

// *************************************************************************************************
// Photoshop PSD loader -- PD by Thatcher Ulrich, integration by Nicolas Schulz, tweaked by STB

#ifndef STBI_NO_PSD
// 检测输入流中是否为 PSD 格式
static int stbi__psd_test(stbi__context *s)
{
   // 检查前4个字节是否为 0x38425053
   int r = (stbi__get32be(s) == 0x38425053);
   // 将输入流指针重置到起始位置
   stbi__rewind(s);
   return r;
}

// 解码 PSD 文件中的 RLE 压缩数据
static int stbi__psd_decode_rle(stbi__context *s, stbi_uc *p, int pixelCount)
{
   // 声明变量 count, nleft, len
   int count, nleft, len;

   // 初始化 count 为 0
   count = 0;
   // 当剩余像素数量大于 0 时，进入循环
   while ((nleft = pixelCount - count) > 0) {
      // 读取一个字节作为 len
      len = stbi__get8(s);
      // 如果 len 为 128，则不执行任何操作
      if (len == 128) {
         // No-op.
      } else if (len < 128) {
         // 复制接下来的 len+1 个字节
         len++;
         // 如果复制的字节数大于剩余像素数量，返回 0 表示数据损坏
         if (len > nleft) return 0; // corrupt data
         // 更新已处理的像素数量
         count += len;
         // 循环复制 len 个字节到目标位置
         while (len) {
            *p = stbi__get8(s);
            p += 4;
            len--;
         }
      } else if (len > 128) {
         stbi_uc   val;
         // 目标位置的接下来 -len+1 个字节从源位置的下一个字节复制
         // (将 len 解释为负的 8 位整数)
         len = 257 - len;
         // 如果复制的字节数大于剩余像素数量，返回 0 表示数据损坏
         if (len > nleft) return 0; // corrupt data
         // 读取源位置的一个字节作为复制值
         val = stbi__get8(s);
         // 更新已处理的像素数量
         count += len;
         // 循环复制 len 个字节到目标位置
         while (len) {
            *p = val;
            p += 4;
            len--;
         }
      }
   }

   // 处理完毕，返回 1
   return 1;
}

// 定义函数 stbi__psd_load，接受参数 s, x, y, comp, req_comp, ri, bpc
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

// 测试 PIC 文件的核心部分是否符合格式要求
static int stbi__pic_test_core(stbi__context *s)
{
   int i;

   // 检查前四个字节是否为特定值
   if (!stbi__pic_is4(s,"\x53\x80\xF6\x34"))
      return 0;

   // 跳过 84 个字节
   for(i=0;i<84;++i)
      stbi__get8(s);

   // 检查接下来四个字节是否为 "PICT"
   if (!stbi__pic_is4(s,"PICT"))
      return 0;

   return 1;
}

// 定义结构体 stbi__pic_packet，包含 size, type, channel 三个成员
typedef struct
{
   stbi_uc size,type,channel;
} stbi__pic_packet;

// 从输入流中读取一个值，存入目标位置
static stbi_uc *stbi__readval(stbi__context *s, int channel, stbi_uc *dest)
{
   // 定义变量 mask，并初始化为 0x80，定义变量 i
   int mask=0x80, i;

   // 循环4次，每次循环右移 mask 1位
   for (i=0; i<4; ++i, mask>>=1) {
      // 如果 channel 与 mask 相与的结果不为 0
      if (channel & mask) {
         // 如果已经到达文件末尾，返回错误信息
         if (stbi__at_eof(s)) return stbi__errpuc("bad file","PIC file too short");
         // 从输入流中读取一个字节，存入 dest 数组中
         dest[i]=stbi__get8(s);
      }
   }

   // 返回 dest 数组
   return dest;
}

// 复制 src 数组中的值到 dest 数组中
static void stbi__copyval(int channel,stbi_uc *dest,const stbi_uc *src)
{
   // 定义变量 mask，并初始化为 0x80，定义变量 i
   int mask=0x80,i;

   // 循环4次，每次循环右移 mask 1位
   for (i=0;i<4; ++i, mask>>=1)
      // 如果 channel 与 mask 相与的结果不为 0
      if (channel&mask)
         // 将 src 数组中的值复制到 dest 数组中
         dest[i]=src[i];
}

// 加载 PIC 图像的核心函数
static stbi_uc *stbi__pic_load_core(stbi__context *s,int width,int height,int *comp, stbi_uc *result)
{
   int act_comp=0,num_packets=0,y,chained;
   stbi__pic_packet packets[10];

   // 这将处理一些奇怪的情况，比如有数据
}

// 加载 PIC 图像的函数
static void *stbi__pic_load(stbi__context *s,int *px,int *py,int *comp,int req_comp, stbi__result_info *ri)
{
   stbi_uc *result;
   int i, x,y, internal_comp;
   STBI_NOTUSED(ri);

   // 如果 comp 为 NULL，则使用 internal_comp
   if (!comp) comp = &internal_comp;

   // 跳过92个字节
   for (i=0; i<92; ++i)
      stbi__get8(s);

   // 读取图像的宽度和高度
   x = stbi__get16be(s);
   y = stbi__get16be(s);

   // 如果图像高度或宽度超出最大限制，返回错误信息
   if (y > STBI_MAX_DIMENSIONS) return stbi__errpuc("too large","Very large image (corrupt?)");
   if (x > STBI_MAX_DIMENSIONS) return stbi__errpuc("too large","Very large image (corrupt?)");

   // 如果已经到达文件末尾，返回错误信息
   if (stbi__at_eof(s))  return stbi__errpuc("bad file","file too short (pic header)");
   // 如果图像尺寸过大，返回错误信息
   if (!stbi__mad3sizes_valid(x, y, 4, 0)) return stbi__errpuc("too large", "PIC image too large to decode");

   // 跳过32位的 ratio
   stbi__get32be(s); //skip `ratio'
   // 跳过16位的 fields
   stbi__get16be(s); //skip `fields'
   // 跳过16位的 pad
   stbi__get16be(s); //skip `pad'

   // 分配内存，用于存储图像数据
   result = (stbi_uc *) stbi__malloc_mad3(x, y, 4, 0);
   if (!result) return stbi__errpuc("outofmem", "Out of memory");
   // 将分配的内存初始化为 0xff
   memset(result, 0xff, x*y*4);

   // 如果加载 PIC 图像的核心函数失败，释放内存并返回 NULL
   if (!stbi__pic_load_core(s,x,y,comp, result)) {
      STBI_FREE(result);
      result=0;
   }
   // 将图像的宽度和高度存入 px 和 py 指针指向的位置
   *px = x;
   *py = y;
   // 如果 req_comp 为 0，则使用 comp
   if (req_comp == 0) req_comp = *comp;
   // 转换图像的格式
   result=stbi__convert_format(result,4,req_comp,x,y);

   // 返回图像数据
   return result;
}

// 测试是否为 PIC 图像
static int stbi__pic_test(stbi__context *s)
{
   // 调用 stbi__pic_test_core 函数对输入的数据进行测试，并将结果赋值给 r
   int r = stbi__pic_test_core(s);
   // 将输入流回退到起始位置
   stbi__rewind(s);
   // 返回测试结果
   return r;
}
#endif

// *************************************************************************************************
// GIF loader -- public domain by Jean-Marc Lienher -- simplified/shrunk by stb

#ifndef STBI_NO_GIF
// 定义结构体 stbi__gif_lzw，包含 prefix、first、suffix 三个成员变量
typedef struct
{
   stbi__int16 prefix;
   stbi_uc first;
   stbi_uc suffix;
} stbi__gif_lzw;

// 定义结构体 stbi__gif，包含多个成员变量，用于存储 GIF 文件的相关信息
typedef struct
{
   int w,h;                     // 图像宽高
   stbi_uc *out;                // 输出缓冲区（始终为 4 个分量）
   stbi_uc *background;         // GIF 文件的当前“背景”
   stbi_uc *history;
   int flags, bgindex, ratio, transparent, eflags;
   stbi_uc  pal[256][4];        // 调色板
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

// 对输入流进行原始测试，判断是否为 GIF 格式
static int stbi__gif_test_raw(stbi__context *s)
{
   int sz;
   // 读取并判断 GIF 文件的标识符是否为 "GIF8"
   if (stbi__get8(s) != 'G' || stbi__get8(s) != 'I' || stbi__get8(s) != 'F' || stbi__get8(s) != '8') return 0;
   // 读取并判断 GIF 文件的版本号是否为 "9" 或 "7"
   sz = stbi__get8(s);
   if (sz != '9' && sz != '7') return 0;
   // 读取并判断 GIF 文件的标识符是否为 "a"
   if (stbi__get8(s) != 'a') return 0;
   return 1;
}

// 对输入流进行测试，判断是否为 GIF 格式
static int stbi__gif_test(stbi__context *s)
{
   // 调用 stbi__gif_test_raw 函数进行测试，并将结果返回
   int r = stbi__gif_test_raw(s);
   // 将输入流回退到起始位置
   stbi__rewind(s);
   return r;
}

// 解析 GIF 文件的调色板
static void stbi__gif_parse_colortable(stbi__context *s, stbi_uc pal[256][4], int num_entries, int transp)
{
   int i;
   // 遍历调色板的每个条目，依次读取 RGB 值
   for (i=0; i < num_entries; ++i) {
      pal[i][2] = stbi__get8(s);  // 蓝色分量
      pal[i][1] = stbi__get8(s);  // 绿色分量
      pal[i][0] = stbi__get8(s);  // 红色分量
      pal[i][3] = transp == i ? 0 : 255;  // 如果当前条目是透明色，则 alpha 通道为 0，否则为 255
   }
}

// 解析 GIF 文件的头部信息
static int stbi__gif_header(stbi__context *s, stbi__gif *g, int *comp, int is_info)
{
   // 定义一个无符号字符变量 version
   stbi_uc version;
   // 检查文件头是否为 GIF 格式
   if (stbi__get8(s) != 'G' || stbi__get8(s) != 'I' || stbi__get8(s) != 'F' || stbi__get8(s) != '8')
      return stbi__err("not GIF", "Corrupt GIF");

   // 读取版本号
   version = stbi__get8(s);
   // 检查版本号是否为 7 或 9
   if (version != '7' && version != '9')    return stbi__err("not GIF", "Corrupt GIF");
   // 检查是否为合法的 GIF 文件
   if (stbi__get8(s) != 'a')                return stbi__err("not GIF", "Corrupt GIF");

   // 清空失败原因
   stbi__g_failure_reason = "";
   // 读取图像宽度和高度
   g->w = stbi__get16le(s);
   g->h = stbi__get16le(s);
   // 读取标志位
   g->flags = stbi__get8(s);
   // 读取背景颜色索引
   g->bgindex = stbi__get8(s);
   // 读取 ratio
   g->ratio = stbi__get8(s);
   // 设置透明颜色索引为 -1
   g->transparent = -1;

   // 如果图像宽度或高度超过最大尺寸限制，则返回错误
   if (g->w > STBI_MAX_DIMENSIONS) return stbi__err("too large","Very large image (corrupt?)");
   if (g->h > STBI_MAX_DIMENSIONS) return stbi__err("too large","Very large image (corrupt?)");

   // 如果 comp 不为 0，则设置为 4，需要在解析注释后才能确定是 3 还是 4
   if (comp != 0) *comp = 4;  

   // 如果是信息模式，则返回 1
   if (is_info) return 1;

   // 如果标志位中包含全局颜色表标志，则解析全局颜色表
   if (g->flags & 0x80)
      stbi__gif_parse_colortable(s,g->pal, 2 << (g->flags & 7), -1);

   // 返回 1
   return 1;
}

// 获取 GIF 图像信息
static int stbi__gif_info_raw(stbi__context *s, int *x, int *y, int *comp)
{
   // 分配内存给 g
   stbi__gif* g = (stbi__gif*) stbi__malloc(sizeof(stbi__gif));
   // 如果分配内存失败，则返回错误
   if (!g) return stbi__err("outofmem", "Out of memory");
   // 如果获取 GIF 头信息失败，则释放内存并返回 0
   if (!stbi__gif_header(s, g, comp, 1)) {
      STBI_FREE(g);
      stbi__rewind( s );
      return 0;
   }
   // 如果 x 不为空，则将图像宽度赋值给 x
   if (x) *x = g->w;
   // 如果 y 不为空，则将图像高度赋值给 y
   if (y) *y = g->h;
   // 释放内存
   STBI_FREE(g);
   // 返回 1
   return 1;
}

// 输出 GIF 编码
static void stbi__out_gif_code(stbi__gif *g, stbi__uint16 code)
{
   // 定义指针变量p和c，整型变量idx
   stbi_uc *p, *c;
   int idx;

   // 递归解码前缀，因为链表是反向的，通过反向处理交错图像会很麻烦
   if (g->codes[code].prefix >= 0)
      stbi__out_gif_code(g, g->codes[code].prefix);

   // 如果当前行数大于等于最大行数，则返回
   if (g->cur_y >= g->max_y) return;

   // 计算当前像素索引
   idx = g->cur_x + g->cur_y;
   // 指针p指向输出数据的当前位置
   p = &g->out[idx];
   // 将当前像素位置的历史记录设置为1
   g->history[idx / 4] = 1;

   // 指针c指向颜色表中与当前代码后缀对应的颜色
   c = &g->color_table[g->codes[code].suffix * 4];
   // 如果颜色的透明度大于128，则不渲染透明像素
   if (c[3] > 128) {
      p[0] = c[2];
      p[1] = c[1];
      p[2] = c[0];
      p[3] = c[3];
   }
   // 更新当前x坐标
   g->cur_x += 4;

   // 如果当前x坐标大于等于最大x坐标
   if (g->cur_x >= g->max_x) {
      // 重置当前x坐标为起始x坐标
      g->cur_x = g->start_x;
      // 更新当前y坐标
      g->cur_y += g->step;

      // 如果当前y坐标大于等于最大y坐标且parse大于0
      while (g->cur_y >= g->max_y && g->parse > 0) {
         // 更新步长
         g->step = (1 << g->parse) * g->line_size;
         // 重置当前y坐标为起始y坐标加上步长的一半
         g->cur_y = g->start_y + (g->step >> 1);
         // parse减1
         --g->parse;
      }
   }
}

// 处理GIF光栅数据
static stbi_uc *stbi__process_gif_raster(stbi__context *s, stbi__gif *g)
}

// 该函数旨在支持动画GIF，尽管stb_image不支持
// two_back是两帧前的图像，用于特定的处理格式
static stbi_uc *stbi__gif_load_next(stbi__context *s, stbi__gif *g, int *comp, int req_comp, stbi_uc *two_back)
}

// 释放GIF加载过程中的内存
static void *stbi__load_gif_main_outofmem(stbi__gif *g, stbi_uc *out, int **delays)
{
   STBI_FREE(g->out);
   STBI_FREE(g->history);
   STBI_FREE(g->background);

   if (out) STBI_FREE(out);
   if (delays && *delays) STBI_FREE(*delays);
   return stbi__errpuc("outofmem", "Out of memory");
}

// 加载GIF的主要函数
static void *stbi__load_gif_main(stbi__context *s, int **delays, int *x, int *y, int *z, int *comp, int req_comp)
}

// 加载GIF
static void *stbi__gif_load(stbi__context *s, int *x, int *y, int *comp, int req_comp, stbi__result_info *ri)
{
   // 定义指向无符号字符的指针 u，并初始化为 0
   stbi_uc *u = 0;
   // 定义 stbi__gif 结构体 g，并用 0 填充其内存
   stbi__gif g;
   memset(&g, 0, sizeof(g));
   // 忽略未使用的参数 ri
   STBI_NOTUSED(ri);

   // 调用 stbi__gif_load_next 函数加载下一帧 GIF 图像数据，并将结果赋值给 u
   u = stbi__gif_load_next(s, &g, comp, req_comp, 0);
   // 如果 u 等于 s，表示已经到达动画 GIF 的结尾标记，将 u 置为 0
   if (u == (stbi_uc *) s) u = 0;  // end of animated gif marker
   // 如果 u 不为空
   if (u) {
      // 将 g 的宽度赋值给指针 x 指向的变量
      *x = g.w;
      // 将 g 的高度赋值给指针 y 指向的变量
      *y = g.h;

      // 如果请求的通道数不为空且不等于 4，将 u 转换为请求的通道数
      if (req_comp && req_comp != 4)
         u = stbi__convert_format(u, 4, req_comp, g.w, g.h);
   } else if (g.out) {
      // 如果出现错误并且已经分配了图像缓冲区，则释放它
      STBI_FREE(g.out);
   }

   // 释放多帧加载所需的缓冲区
   STBI_FREE(g.history);
   STBI_FREE(g.background);

   // 返回 u
   return u;
}

// 定义静态函数 stbi__gif_info，用于获取 GIF 图像的信息
static int stbi__gif_info(stbi__context *s, int *x, int *y, int *comp)
{
   // 调用 stbi__gif_info_raw 函数获取原始的 GIF 图像信息
   return stbi__gif_info_raw(s,x,y,comp);
}
#endif

// *************************************************************************************************
// Radiance RGBE HDR loader
// originally by Nicolas Schulz
#ifndef STBI_NO_HDR
// 核心函数，用于测试是否为 HDR 格式的图像
static int stbi__hdr_test_core(stbi__context *s, const char *signature)
{
   int i;
   // 遍历签名字符串，逐个比较读取的字符是否与签名相符
   for (i=0; signature[i]; ++i)
      if (stbi__get8(s) != signature[i])
          return 0;
   // 将读取指针重新定位到起始位置
   stbi__rewind(s);
   return 1;
}

// 测试函数，用于检测是否为 HDR 格式的图像
static int stbi__hdr_test(stbi__context* s)
{
   // 检测是否为 #?RADIANCE 格式的 HDR 图像
   int r = stbi__hdr_test_core(s, "#?RADIANCE\n");
   // 将读取指针重新定位到起始位置
   stbi__rewind(s);
   // 如果不是 #?RADIANCE 格式的 HDR 图像
   if(!r) {
       // 检测是否为 #?RGBE 格式的 HDR 图像
       r = stbi__hdr_test_core(s, "#?RGBE\n");
       // 将读取指针重新定位到起始位置
       stbi__rewind(s);
   }
   return r;
}

// 定义 HDR 缓冲区的长度
#define STBI__HDR_BUFLEN  1024
// 从输入流中获取 token
static char *stbi__hdr_gettoken(stbi__context *z, char *buffer)
{
   int len=0;
   char c = '\0';

   // 读取一个字符
   c = (char) stbi__get8(z);

   // 循环读取字符，直到遇到换行符或者到达文件末尾
   while (!stbi__at_eof(z) && c != '\n') {
      buffer[len++] = c;
      // 如果缓冲区长度达到上限，跳过当前行剩余内容
      if (len == STBI__HDR_BUFLEN-1) {
         // 跳过当前行剩余内容
         while (!stbi__at_eof(z) && stbi__get8(z) != '\n')
            ;
         break;
      }
      // 读取下一个字符
      c = (char) stbi__get8(z);
   }

   // 在缓冲区末尾添加字符串结束符
   buffer[len] = 0;
   return buffer;
}
# 将输入的无符号字符指针转换为浮点数指针，并根据请求的通道数进行转换
static void stbi__hdr_convert(float *output, stbi_uc *input, int req_comp)
{
   # 如果输入的第四个元素不为0
   if ( input[3] != 0 ) {
      float f1;
      # 计算指数
      f1 = (float) ldexp(1.0f, input[3] - (int)(128 + 8));
      # 如果请求的通道数小于等于2，则计算输出的第一个元素
      if (req_comp <= 2)
         output[0] = (input[0] + input[1] + input[2]) * f1 / 3;
      # 如果请求的通道数大于2，则分别计算输出的三个元素
      else {
         output[0] = input[0] * f1;
         output[1] = input[1] * f1;
         output[2] = input[2] * f1;
      }
      # 如果请求的通道数为2，则设置输出的第二个元素为1
      if (req_comp == 2) output[1] = 1;
      # 如果请求的通道数为4，则设置输出的第四个元素为1
      if (req_comp == 4) output[3] = 1;
   } else {
      # 如果输入的第四个元素为0，则根据请求的通道数进行不同的处理
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

# 加载 HDR 图像
static float *stbi__hdr_load(stbi__context *s, int *x, int *y, int *comp, int req_comp, stbi__result_info *ri)
}

# 获取 HDR 图像的信息
static int stbi__hdr_info(stbi__context *s, int *x, int *y, int *comp)
{
   # 创建缓冲区
   char buffer[STBI__HDR_BUFLEN];
   char *token;
   int valid = 0;
   int dummy;

   # 如果 x 为 NULL，则指向 dummy
   if (!x) x = &dummy;
   # 如果 y 为 NULL，则指向 dummy
   if (!y) y = &dummy;
   # 如果 comp 为 NULL，则指向 dummy
   if (!comp) comp = &dummy;

   # 如果不是 HDR 格式的文件，则返回 0
   if (stbi__hdr_test(s) == 0) {
       stbi__rewind( s );
       return 0;
   }

   # 循环读取 token，并判断是否为有效的 HDR 格式
   for(;;) {
      token = stbi__hdr_gettoken(s,buffer);
      if (token[0] == 0) break;
      if (strcmp(token, "FORMAT=32-bit_rle_rgbe") == 0) valid = 1;
   }

   # 如果不是有效的 HDR 格式，则返回 0
   if (!valid) {
       stbi__rewind( s );
       return 0;
   }
   # 读取 token，并判断是否为 "-Y "，如果不是则返回 0
   token = stbi__hdr_gettoken(s,buffer);
   if (strncmp(token, "-Y ", 3)) {
       stbi__rewind( s );
       return 0;
   }
   token += 3;
   # 将 token 转换为整数，并赋值给 y
   *y = (int) strtol(token, &token, 10);
   # 跳过空格
   while (*token == ' ') ++token;
   # 读取 token，并判断是否为 "+X "，如果不是则返回 0
   if (strncmp(token, "+X ", 3)) {
       stbi__rewind( s );
       return 0;
   }
   token += 3;
   # 将 token 转换为整数，并赋值给 x
   *x = (int) strtol(token, NULL, 10);
   # 设置通道数为 3
   *comp = 3;
   return 1;
}
#endif // STBI_NO_HDR

# 获取 BMP 图像的信息
#ifndef STBI_NO_BMP
static int stbi__bmp_info(stbi__context *s, int *x, int *y, int *comp)
{
   // 定义指针变量 p
   void *p;
   // 定义 BMP 数据结构 info
   stbi__bmp_data info;

   // 设置 info 结构体中的 all_a 字段为 255
   info.all_a = 255;
   // 调用 stbi__bmp_parse_header 函数解析 BMP 头部信息，返回结果赋给 p
   p = stbi__bmp_parse_header(s, &info);
   // 如果 p 为空指针，则回退到文件开头并返回 0
   if (p == NULL) {
      stbi__rewind( s );
      return 0;
   }
   // 如果 x 不为空，则将 s->img_x 赋给 *x
   if (x) *x = s->img_x;
   // 如果 y 不为空，则将 s->img_y 赋给 *y
   if (y) *y = s->img_y;
   // 如果 comp 不为空
   if (comp) {
      // 如果 info 结构体中的 bpp 字段为 24 并且 ma 字段为 0xff000000，则将 3 赋给 *comp
      if (info.bpp == 24 && info.ma == 0xff000000)
         *comp = 3;
      // 否则，如果 ma 字段不为 0，则将 4 赋给 *comp，否则将 3 赋给 *comp
      else
         *comp = info.ma ? 4 : 3;
   }
   // 返回 1
   return 1;
}
#endif

#ifndef STBI_NO_PSD
// 定义函数 stbi__psd_info
static int stbi__psd_info(stbi__context *s, int *x, int *y, int *comp)
{
   // 定义变量 channelCount, dummy, depth
   int channelCount, dummy, depth;
   // 如果 x 为空，则将 dummy 的地址赋给 x
   if (!x) x = &dummy;
   // 如果 y 为空，则将 dummy 的地址赋给 y
   if (!y) y = &dummy;
   // 如果 comp 为空，则将 dummy 的地址赋给 comp
   if (!comp) comp = &dummy;
   // 如果读取的 PSD 文件标识不是 0x38425053，则回退到文件开头并返回 0
   if (stbi__get32be(s) != 0x38425053) {
       stbi__rewind( s );
       return 0;
   }
   // 如果读取的版本号不是 1，则回退到文件开头并返回 0
   if (stbi__get16be(s) != 1) {
       stbi__rewind( s );
       return 0;
   }
   // 跳过 6 个字节
   stbi__skip(s, 6);
   // 读取通道数赋给 channelCount
   channelCount = stbi__get16be(s);
   // 如果通道数小于 0 或大于 16，则回退到文件开头并返回 0
   if (channelCount < 0 || channelCount > 16) {
       stbi__rewind( s );
       return 0;
   }
   // 读取高度赋给 *y
   *y = stbi__get32be(s);
   // 读取宽度赋给 *x
   *x = stbi__get32be(s);
   // 读取深度赋给 depth
   depth = stbi__get16be(s);
   // 如果深度不是 8 也不是 16，则回退到文件开头并返回 0
   if (depth != 8 && depth != 16) {
       stbi__rewind( s );
       return 0;
   }
   // 如果读取的值不是 3，则回退到文件开头并返回 0
   if (stbi__get16be(s) != 3) {
       stbi__rewind( s );
       return 0;
   }
   // 将 4 赋给 *comp
   *comp = 4;
   // 返回 1
   return 1;
}

// 定义函数 stbi__psd_is16
static int stbi__psd_is16(stbi__context *s)
{
   // 定义变量 channelCount, depth
   int channelCount, depth;
   // 如果读取的 PSD 文件标识不是 0x38425053，则回退到文件开头并返回 0
   if (stbi__get32be(s) != 0x38425053) {
       stbi__rewind( s );
       return 0;
   }
   // 如果读取的版本号不是 1，则回退到文件开头并返回 0
   if (stbi__get16be(s) != 1) {
       stbi__rewind( s );
       return 0;
   }
   // 跳过 6 个字节
   stbi__skip(s, 6);
   // 读取通道数赋给 channelCount
   channelCount = stbi__get16be(s);
   // 如果通道数小于 0 或大于 16，则回退到文件开头并返回 0
   if (channelCount < 0 || channelCount > 16) {
       stbi__rewind( s );
       return 0;
   }
   // 跳过 8 个字节
   STBI_NOTUSED(stbi__get32be(s));
   STBI_NOTUSED(stbi__get32be(s));
   // 读取深度赋给 depth
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
// 定义函数 stbi__pic_info
static int stbi__pic_info(stbi__context *s, int *x, int *y, int *comp)
{
   // 定义变量，用于记录实际通道数、数据包数量、是否链式传输、虚拟变量
   int act_comp=0,num_packets=0,chained,dummy;
   // 定义数据包数组，最多包含10个数据包
   stbi__pic_packet packets[10];

   // 如果 x 为 NULL，则将 x 指向虚拟变量 dummy
   if (!x) x = &dummy;
   // 如果 y 为 NULL，则将 y 指向虚拟变量 dummy
   if (!y) y = &dummy;
   // 如果 comp 为 NULL，则将 comp 指向虚拟变量 dummy
   if (!comp) comp = &dummy;

   // 如果不是 PPM 或 PGM 格式，则回退到文件开头并返回 0
   if (!stbi__pic_is4(s,"\x53\x80\xF6\x34")) {
      stbi__rewind(s);
      return 0;
   }

   // 跳过 88 个字节
   stbi__skip(s, 88);

   // 读取图像的宽度和高度
   *x = stbi__get16be(s);
   *y = stbi__get16be(s);
   // 如果已经到达文件末尾，则回退到文件开头并返回 0
   if (stbi__at_eof(s)) {
      stbi__rewind( s);
      return 0;
   }
   // 如果图像宽度不为 0 且 (1 << 28) / 图像宽度 小于 图像高度，则回退到文件开头并返回 0
   if ( (*x) != 0 && (1 << 28) / (*x) < (*y)) {
      stbi__rewind( s );
      return 0;
   }

   // 跳过 8 个字节
   stbi__skip(s, 8);

   // 循环读取数据包
   do {
      stbi__pic_packet *packet;

      // 如果数据包数量达到上限，则返回 0
      if (num_packets==sizeof(packets)/sizeof(packets[0]))
         return 0;

      // 获取当前数据包
      packet = &packets[num_packets++];
      // 读取链式传输标志、数据包大小、数据包类型、数据包通道
      chained = stbi__get8(s);
      packet->size    = stbi__get8(s);
      packet->type    = stbi__get8(s);
      packet->channel = stbi__get8(s);
      // 记录数据包通道到实际通道数的映射
      act_comp |= packet->channel;

      // 如果已经到达文件末尾，则回退到文件开头并返回 0
      if (stbi__at_eof(s)) {
          stbi__rewind( s );
          return 0;
      }
      // 如果数据包大小不为 8，则回退到文件开头并返回 0
      if (packet->size != 8) {
          stbi__rewind( s );
          return 0;
      }
   } while (chained);

   // 根据实际通道数确定图像通道数
   *comp = (act_comp & 0x10 ? 4 : 3);

   // 返回 1，表示加载成功
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

// 检测文件是否为 PPM 或 PGM 格式
static int      stbi__pnm_test(stbi__context *s)
{
   char p, t;
   p = (char) stbi__get8(s);
   t = (char) stbi__get8(s);
   // 如果文件格式不为 PPM 或 PGM，则回退到文件开头并返回 0
   if (p != 'P' || (t != '5' && t != '6')) {
       stbi__rewind( s );
       return 0;
   }
   // 返回 1，表示文件格式正确
   return 1;
}

// 加载 PPM 或 PGM 格式的图像数据
static void *stbi__pnm_load(stbi__context *s, int *x, int *y, int *comp, int req_comp, stbi__result_info *ri)
{
   // 定义指向无符号字符的指针变量 out
   stbi_uc *out;
   // 使用 STBI_NOTUSED 宏处理 ri 变量
   STBI_NOTUSED(ri);

   // 调用 stbi__pnm_info 函数获取 PNM 图像信息，并将结果赋值给 ri->bits_per_channel
   ri->bits_per_channel = stbi__pnm_info(s, (int *)&s->img_x, (int *)&s->img_y, (int *)&s->img_n);
   // 如果 bits_per_channel 为 0，则返回 0
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

   // 检查图像大小是否合法，如果不合法则返回错误信息
   if (!stbi__mad4sizes_valid(s->img_n, s->img_x, s->img_y, ri->bits_per_channel / 8, 0))
      return stbi__errpuc("too large", "PNM too large");

   // 分配内存给 out 变量，用于存储图像数据
   out = (stbi_uc *) stbi__malloc_mad4(s->img_n, s->img_x, s->img_y, ri->bits_per_channel / 8, 0);
   // 如果内存分配失败，则返回错误信息
   if (!out) return stbi__errpuc("outofmem", "Out of memory");
   // 从输入流中读取图像数据到 out 变量
   if (!stbi__getn(s, out, s->img_n * s->img_x * s->img_y * (ri->bits_per_channel / 8))) {
      STBI_FREE(out);
      return stbi__errpuc("bad PNM", "PNM file truncated");
   }

   // 如果 req_comp 不为空且不等于图像通道数，则进行格式转换
   if (req_comp && req_comp != s->img_n) {
      // 如果 bits_per_channel 为 16，则调用 stbi__convert_format16 进行格式转换
      if (ri->bits_per_channel == 16) {
         out = (stbi_uc *) stbi__convert_format16((stbi__uint16 *) out, s->img_n, req_comp, s->img_x, s->img_y);
      } else {
         // 否则调用 stbi__convert_format 进行格式转换
         out = stbi__convert_format(out, s->img_n, req_comp, s->img_x, s->img_y);
      }
      // 如果格式转换失败，则返回错误信息
      if (out == NULL) return out; // stbi__convert_format frees input on failure
   }
   // 返回存储图像数据的指针变量 out
   return out;
}

// 判断字符是否为空白字符
static int      stbi__pnm_isspace(char c)
{
   return c == ' ' || c == '\t' || c == '\n' || c == '\v' || c == '\f' || c == '\r';
}

// 跳过空白字符
static void     stbi__pnm_skip_whitespace(stbi__context *s, char *c)
{
   for (;;) {
      // 跳过空白字符
      while (!stbi__at_eof(s) && stbi__pnm_isspace(*c))
         *c = (char) stbi__get8(s);

      // 如果遇到注释，则跳过注释行
      if (stbi__at_eof(s) || *c != '#')
         break;

      while (!stbi__at_eof(s) && *c != '\n' && *c != '\r' )
         *c = (char) stbi__get8(s);
   }
}

// 判断字符是否为数字
static int      stbi__pnm_isdigit(char c)
{
   return c >= '0' && c <= '9';
}

// 从输入流中获取整数
static int      stbi__pnm_getinteger(stbi__context *s, char *c)
{
   # 初始化变量 value 为 0
   int value = 0;

   # 当未到达文件末尾且当前字符为数字时，执行循环
   while (!stbi__at_eof(s) && stbi__pnm_isdigit(*c)) {
      # 将当前值乘以 10，加上当前字符对应的数字值
      value = value*10 + (*c - '0');
      # 读取下一个字符
      *c = (char) stbi__get8(s);
      # 如果 value 大于 214748364 或者等于 214748364 且当前字符大于 '7'，则返回错误
      if((value > 214748364) || (value == 214748364 && *c > '7'))
          return stbi__err("integer parse overflow", "Parsing an integer in the PPM header overflowed a 32-bit int");
   }

   # 返回解析出的整数值
   return value;
}

# 获取 PNM 图像的信息
static int      stbi__pnm_info(stbi__context *s, int *x, int *y, int *comp)
{
   int maxv, dummy;
   char c, p, t;

   # 如果 x 为 NULL，则指向 dummy 变量
   if (!x) x = &dummy;
   # 如果 y 为 NULL，则指向 dummy 变量
   if (!y) y = &dummy;
   # 如果 comp 为 NULL，则指向 dummy 变量
   if (!comp) comp = &dummy;

   # 将文件指针重置到文件开头
   stbi__rewind(s);

   # 获取标识符
   p = (char) stbi__get8(s);
   t = (char) stbi__get8(s);
   # 如果标识符不是 'P' 或者后续字符不是 '5' 或 '6'，则将文件指针重置到文件开头并返回 0
   if (p != 'P' || (t != '5' && t != '6')) {
       stbi__rewind(s);
       return 0;
   }

   # 根据 t 的值确定图像的组件数
   *comp = (t == '6') ? 3 : 1;  # '5' is 1-component .pgm; '6' is 3-component .ppm

   # 读取下一个字符
   c = (char) stbi__get8(s);
   # 跳过空白字符
   stbi__pnm_skip_whitespace(s, &c);

   # 读取图像宽度
   *x = stbi__pnm_getinteger(s, &c); // read width
   # 如果宽度为 0，则返回错误
   if(*x == 0)
       return stbi__err("invalid width", "PPM image header had zero or overflowing width");
   # 跳过空白字符
   stbi__pnm_skip_whitespace(s, &c);

   # 读取图像高度
   *y = stbi__pnm_getinteger(s, &c); // read height
   # 如果高度为 0，则返回错误
   if (*y == 0)
       return stbi__err("invalid width", "PPM image header had zero or overflowing width");
   # 跳过空白字符
   stbi__pnm_skip_whitespace(s, &c);

   # 读取最大像素值
   maxv = stbi__pnm_getinteger(s, &c);  // read max value
   # 如果最大像素值大于 65535，则返回错误；如果大于 255，则返回 16；否则返回 8
   if (maxv > 65535)
      return stbi__err("max value > 65535", "PPM image supports only 8-bit and 16-bit images");
   else if (maxv > 255)
      return 16;
   else
      return 8;
}

# 判断 PNM 图像是否为 16 位
static int stbi__pnm_is16(stbi__context *s)
{
   # 如果获取 PNM 图像信息返回 16，则返回 1；否则返回 0
   if (stbi__pnm_info(s, NULL, NULL, NULL) == 16)
       return 1;
   return 0;
}
#endif

# 获取图像的主要信息
static int stbi__info_main(stbi__context *s, int *x, int *y, int *comp)
{
   #ifndef STBI_NO_JPEG
   # 如果未定义 STBI_NO_JPEG，则执行 stbi__jpeg_info 函数，获取 JPEG 图像信息
   if (stbi__jpeg_info(s, x, y, comp)) return 1;
   #endif

   #ifndef STBI_NO_PNG
   # 如果未定义 STBI_NO_PNG，则执行 stbi__png_info 函数，获取 PNG 图像信息
   if (stbi__png_info(s, x, y, comp))  return 1;
   #endif

   #ifndef STBI_NO_GIF
   # 如果未定义 STBI_NO_GIF，则执行 stbi__gif_info 函数，获取 GIF 图像信息
   if (stbi__gif_info(s, x, y, comp))  return 1;
   #endif

   #ifndef STBI_NO_BMP
   # 如果未定义 STBI_NO_BMP，则执行 stbi__bmp_info 函数，获取 BMP 图像信息
   if (stbi__bmp_info(s, x, y, comp))  return 1;
   #endif

   #ifndef STBI_NO_PSD
   # 如果未定义 STBI_NO_PSD，则执行 stbi__psd_info 函数，获取 PSD 图像信息
   if (stbi__psd_info(s, x, y, comp))  return 1;
   #endif

   #ifndef STBI_NO_PIC
   # 如果未定义 STBI_NO_PIC，则执行 stbi__pic_info 函数，获取 PIC 图像信息
   if (stbi__pic_info(s, x, y, comp))  return 1;
   #endif

   #ifndef STBI_NO_PNM
   # 如果未定义 STBI_NO_PNM，则执行 stbi__pnm_info 函数，获取 PNM 图像信息
   if (stbi__pnm_info(s, x, y, comp))  return 1;
   #endif

   #ifndef STBI_NO_HDR
   # 如果未定义 STBI_NO_HDR，则执行 stbi__hdr_info 函数，获取 HDR 图像信息
   if (stbi__hdr_info(s, x, y, comp))  return 1;
   #endif

   // test tga last because it's a crappy test!
   #ifndef STBI_NO_TGA
   # 如果未定义 STBI_NO_TGA，则执行 stbi__tga_info 函数，获取 TGA 图像信息
   if (stbi__tga_info(s, x, y, comp))
       return 1;
   #endif
   # 如果未识别出图像类型，则返回错误信息
   return stbi__err("unknown image type", "Image not of any known type, or corrupt");
}

static int stbi__is_16_main(stbi__context *s)
{
   #ifndef STBI_NO_PNG
   # 如果未定义 STBI_NO_PNG，则执行 stbi__png_is16 函数，判断是否为 16 位 PNG 图像
   if (stbi__png_is16(s))  return 1;
   #endif

   #ifndef STBI_NO_PSD
   # 如果未定义 STBI_NO_PSD，则执行 stbi__psd_is16 函数，判断是否为 16 位 PSD 图像
   if (stbi__psd_is16(s))  return 1;
   #endif

   #ifndef STBI_NO_PNM
   # 如果未定义 STBI_NO_PNM，则执行 stbi__pnm_is16 函数，判断是否为 16 位 PNM 图像
   if (stbi__pnm_is16(s))  return 1;
   #endif
   # 如果不是 16 位图像，则返回 0
   return 0;
}

#ifndef STBI_NO_STDIO
STBIDEF int stbi_info(char const *filename, int *x, int *y, int *comp)
{
    # 打开文件
    FILE *f = stbi__fopen(filename, "rb");
    int result;
    # 如果文件打开失败，则返回错误信息
    if (!f) return stbi__err("can't fopen", "Unable to open file");
    # 从文件中获取图像信息
    result = stbi_info_from_file(f, x, y, comp);
    # 关闭文件
    fclose(f);
    return result;
}

STBIDEF int stbi_info_from_file(FILE *f, int *x, int *y, int *comp)
{
   int r;
   stbi__context s;
   long pos = ftell(f);
   # 初始化图像上下文
   stbi__start_file(&s, f);
   # 获取图像信息
   r = stbi__info_main(&s,x,y,comp);
   # 恢复文件指针位置
   fseek(f,pos,SEEK_SET);
   return r;
}

STBIDEF int stbi_is_16_bit(char const *filename)
{
    # 打开文件
    FILE *f = stbi__fopen(filename, "rb");
    int result;
    # 如果文件打开失败，则返回错误信息
    if (!f) return stbi__err("can't fopen", "Unable to open file");
    # 判断是否为 16 位图像
    result = stbi_is_16_bit_from_file(f);
    # 关闭文件
    fclose(f);
    return result;
}
// 定义一个函数，用于检查文件是否包含16位图像数据
STBIDEF int stbi_is_16_bit_from_file(FILE *f)
{
   int r;
   stbi__context s;
   // 获取文件当前位置
   long pos = ftell(f);
   // 从文件开始创建一个内存上下文
   stbi__start_file(&s, f);
   // 检查文件是否包含16位图像数据
   r = stbi__is_16_main(&s);
   // 将文件指针移动到之前保存的位置
   fseek(f,pos,SEEK_SET);
   // 返回检查结果
   return r;
}
#endif // !STBI_NO_STDIO

// 从内存中获取图像信息
STBIDEF int stbi_info_from_memory(stbi_uc const *buffer, int len, int *x, int *y, int *comp)
{
   stbi__context s;
   // 从内存开始创建一个内存上下文
   stbi__start_mem(&s,buffer,len);
   // 获取图像信息
   return stbi__info_main(&s,x,y,comp);
}

// 从回调函数中获取图像信息
STBIDEF int stbi_info_from_callbacks(stbi_io_callbacks const *c, void *user, int *x, int *y, int *comp)
{
   stbi__context s;
   // 从回调函数开始创建一个内存上下文
   stbi__start_callbacks(&s, (stbi_io_callbacks *) c, user);
   // 获取图像信息
   return stbi__info_main(&s,x,y,comp);
}

// 从内存中检查是否包含16位图像数据
STBIDEF int stbi_is_16_bit_from_memory(stbi_uc const *buffer, int len)
{
   stbi__context s;
   // 从内存开始创建一个内存上下文
   stbi__start_mem(&s,buffer,len);
   // 检查内存中是否包含16位图像数据
   return stbi__is_16_main(&s);
}

// 从回调函数中检查是否包含16位图像数据
STBIDEF int stbi_is_16_bit_from_callbacks(stbi_io_callbacks const *c, void *user)
{
   stbi__context s;
   // 从回调函数开始创建一个内存上下文
   stbi__start_callbacks(&s, (stbi_io_callbacks *) c, user);
   // 检查回调函数中是否包含16位图像数据
   return stbi__is_16_main(&s);
}

#endif // STB_IMAGE_IMPLEMENTATION
# 这部分代码是软件的许可声明，包括两种选择：MIT许可和公有领域（Unlicense）
# MIT许可声明
# 软件被提供"AS IS"，没有任何形式的保证，包括但不限于适销性、特定用途的适用性和非侵权性。
# 作者或版权持有人不对任何索赔、损害或其他责任负责，无论是合同诉讼、侵权行为还是其他情况，由此产生、由此引起或与软件或软件使用的其他交易有关。
# 公有领域声明
# 软件被释放到公有领域，任何人都可以自由复制、修改、发布、使用、编译、出售或分发该软件，无论是以源代码形式还是编译后的二进制形式，无论是商业用途还是非商业用途，以任何方式。
# 在承认版权法的司法管辖区，软件的作者或作者将软件的所有版权利益捐赠给公共领域。作者们做出这一捐赠是为了使公众受益，对我们的继承人和后继者造成损害。我们打算将这一捐赠作为对版权法下对该软件的所有现有和未来权利的永久放弃的公开行为。
# 软件被提供"AS IS"，没有任何形式的保证，包括但不限于适销性、特定用途的适用性和非侵权性。
# 作者不对任何索赔、损害或其他责任负责，无论是合同诉讼、侵权行为还是其他情况，由此产生、由此引起或与软件或软件使用的其他交易有关。
```