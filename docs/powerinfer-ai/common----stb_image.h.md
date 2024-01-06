# `PowerInfer\common\stb_image.h`

```
/* stb_image - v2.28 - public domain image loader - http://nothings.org/stb
                                  no warranty implied; use at your own risk
   stb_image是一个公共领域的图像加载器，版本为2.28，可以在http://nothings.org/stb找到
   没有明示的保修；使用需自担风险
*/

   Do this:
      #define STB_IMAGE_IMPLEMENTATION
   before you include this file in *one* C or C++ file to create the implementation.
   在*一个* C 或 C++ 文件中包含此文件之前，需要定义 #define STB_IMAGE_IMPLEMENTATION 来创建实现。

   // i.e. it should look like this:
   #include ...
   #include ...
   #include ...
   #define STB_IMAGE_IMPLEMENTATION
   #include "stb_image.h"
   例如，应该像这样：
   #include ...
   #include ...
   #include ...
   #define STB_IMAGE_IMPLEMENTATION
   #include "stb_image.h"

   You can #define STBI_ASSERT(x) before the #include to avoid using assert.h.
   And #define STBI_MALLOC, STBI_REALLOC, and STBI_FREE to avoid using malloc,realloc,free
   可以在#include之前定义STBI_ASSERT(x)来避免使用assert.h。
   也可以定义STBI_MALLOC、STBI_REALLOC和STBI_FREE来避免使用malloc、realloc、free

   QUICK NOTES:
      Primarily of interest to game developers and other people who can
   快速笔记：
      主要适用于游戏开发人员和其他可以使用此图像加载器的人
# 避免有问题的图像，只需要简单的接口
JPEG 基线和渐进式（12 bpc/arithmetic 不支持，与标准 IJG 库相同）
PNG 1/2/4/8/16 位每通道
TGA（不确定是什么子集，如果是子集）
BMP 非 1bpp，非 RLE
PSD（仅支持合成视图，没有额外通道，8/16 位每通道）
GIF（*comp 总是报告为 4 通道）
HDR（辐射 rgbE 格式）
PIC（Softimage PIC）
PNM（仅支持 PPM 和 PGM 二进制）
动画 GIF 仍需要一个合适的 API，但这是一种方法：
    http://gist.github.com/urraka/685d9a6340b26b830d49
- 从内存或通过文件解码（定义 STBI_NO_STDIO 以删除代码）
- 通过任意 I/O 回调解码
- 在 x86/x64（SSE2）和 ARM（NEON）上的 SIMD 加速
# 完整的文档在下面的"DOCUMENTATION"中
# 许可证信息在文件末尾
# 最近的修订历史记录
# 2.28 (2023-01-29) 修复了许多错误，安全错误，以及大量的其他问题
# 2.27 (2021-07-11) 更好地记录了stbi_info，支持16位PNM，修复了错误
# 2.26 (2020-07-13) 修复了许多次要问题
# 2.25 (2020-02-02) 修复了警告
# 2.24 (2020-02-02) 修复了警告；线程本地的failure_reason和flip_vertically
# 2.23 (2019-08-11) 修复了clang静态分析警告
# 2.22 (2019-03-04) 修复了gif问题，修复了警告
# 2.21 (2019-02-25) 修复了注释中的拼写错误
# 2.20 (2019-02-07) 支持Windows中的utf8文件名；修复了警告和平台条件编译
# 2.19 (2018-02-11) 修复了警告
# 2.18  (2018-01-30) fix warnings
# 修复警告

# 2.17  (2018-01-29) bugfix, 1-bit BMP, 16-bitness query, fix warnings
# 修复错误，添加对1位BMP和16位查询的支持，修复警告

# 2.16  (2017-07-23) all functions have 16-bit variants; optimizations; bugfixes
# 所有函数都有16位变体；优化；修复错误

# 2.15  (2017-03-18) fix png-1,2,4; all Imagenet JPGs; no runtime SSE detection on GCC
# 修复PNG-1,2,4；所有Imagenet JPGs；GCC上不进行运行时SSE检测

# 2.14  (2017-03-03) remove deprecated STBI_JPEG_OLD; fixes for Imagenet JPGs
# 移除不推荐使用的STBI_JPEG_OLD；修复Imagenet JPGs

# 2.13  (2016-12-04) experimental 16-bit API, only for PNG so far; fixes
# 实验性的16位API，目前只支持PNG；修复错误

# 2.12  (2016-04-02) fix typo in 2.11 PSD fix that caused crashes
# 修复2.11版本中导致崩溃的PSD修复中的拼写错误

# 2.11  (2016-04-02) 16-bit PNGS; enable SSE2 in non-gcc x64
# RGB格式的JPEG；移除PSD中的白色衬垫；在堆栈上分配大型结构；修正PNG和BMP的通道数

# 2.10  (2016-01-22) avoid warning introduced in 2.09
# 避免2.09版本引入的警告

# 2.09  (2016-01-16) 16-bit TGA; comments in PNM files; STBI_REALLOC_SIZED
# 16位TGA；PNM文件中的注释；STBI_REALLOC_SIZED
这段代码看起来是一些关于图片处理的库或者工具的作者名单，以及一些优化和bug修复的贡献者名单。这些名单可能是用来感谢他们对代码库的贡献或者记录他们的贡献。
# 以下是一些程序员的名字和他们的贡献者信息
Phil Jordan
Dave Moore
Roy Eltham
Hayaki Saito
Nathan Reed
Won Chun
Luke Graham
Johan Duparc
Nick Verigakis
the Horde3D community
Thomas Ruf
Ronny Chevalier
Janez Zemva
John Bartholomew
Michal Cichon
Jonathan Blow
Ken Hamada
Tero Hanninen
Eugene Golushkov
Laurent Gomila
Cort Stratton
Aruelien Pocheville
Sergio Gonzalez
Thibault Reuille
Cass Everitt
Ryamond Barbiero
Paul Du Bois
Engin Manap
Aldo Culquicondor
Philipp Wiesemann
Dale Weiler
Oriol Ferrer Mesia
Josh Tobin
Neil Bickford
Matthew Gregan
Julian Raschke
Gregory Mullen
Christian Floisand
Baldur Karlsson
Kevin Schmidt
JR Smith
Brad Weinberger
Matvey Cherevko
Luca Sas
Alexander Veselov
Zack Middleton
[reserved]
Ryan C. Gordon
[reserved]
[reserved]
DO NOT ADD YOUR NAME HERE
Jacko Dirks
/*
  添加你的名字到贡献者名单中，选择一个随机的空白位置填写。
  在 stb PRs 中，80% 的合并冲突是因为人们将他们的名字添加到贡献者名单的末尾。
*/

#ifndef STBI_INCLUDE_STB_IMAGE_H
#define STBI_INCLUDE_STB_IMAGE_H

// 文档说明
//
// 限制：
//    - 没有每通道 12 位的 JPEG
//    - 没有算术编码的 JPEG
//    - GIF 总是返回 *comp=4
//
// 基本用法（有关 HDR 用法，请参阅下面的 HDR 讨论）：
//    int x,y,n;
//    unsigned char *data = stbi_load(filename, &x, &y, &n, 0);
//    // ... 如果数据不为空，则处理数据 ...
// 释放之前分配的图像数据内存
stbi_image_free(data);

// 标准参数：
//    int *x                 -- 输出图像宽度（像素）
//    int *y                 -- 输出图像高度（像素）
//    int *channels_in_file  -- 输出图像文件中的图像组件数
//    int desired_channels   -- 如果非零，请求结果中的图像组件数
//
// 图像加载器的返回值是一个指向像素数据的 'unsigned char *' 指针，如果分配失败或图像损坏或无效，则返回 NULL。像素数据由 *y 个扫描线和 *x 个像素组成，每个像素由 N 个交错的 8 位组件组成；指向的第一个像素是图像中最左上角的像素。无论格式如何，图像扫描线或像素之间都没有填充。如果 desired_channels 非零，则组件数 N 为 desired_channels，否则为 *channels_in_file。如果 desired_channels 非零，则 *channels_in_file 为 _would_ have been 的组件数
// 输出图像的通道数，例如如果你将 desired_channels 设置为 4，你将始终得到 RGBA 输出，
// 但你可以检查 *channels_in_file 来查看它是否是不透明的，例如因为源图像只有 3 个通道。
//
// 一个具有 N 个分量的输出图像，每个像素中的分量按照以下顺序交错排列：
//
//     N=#comp     components
//       1           灰度
//       2           灰度，透明度
//       3           红色，绿色，蓝色
//       4           红色，绿色，蓝色，透明度
//
// 如果由于任何原因图像加载失败，返回值将为 NULL，
// *x, *y, *channels_in_file 将保持不变。可以查询 stbi_failure_reason() 函数，
// 得到一个非常简短、不适合最终用户的加载失败原因解释。
// 定义 STBI_NO_FAILURE_STRINGS 可以避免编译这些字符串，定义 STBI_FAILURE_USERMSG
// 可以得到稍微更适合用户的解释。
//
// 对于调色板 PNG、BMP、GIF 和 PIC 图像，会自动去除调色板。
//
// 若要查询图像的宽度、高度和组件数量，而不必解码整个文件，可以使用 stbi_info 系列函数：
//
//   int x,y,n,ok;
//   ok = stbi_info(filename, &x, &y, &n);
//   // 如果图像是支持的格式，则返回 ok=1 并设置 x、y、n，
//   // 否则返回 0。
//
// 注意，stb_image 在其公共 API 中广泛使用 int 类型来表示大小，包括内存缓冲区的大小。这现在是 API 的一部分，因此很难在不引起破坏的情况下进行更改。因此，各种图像加载器对图像大小都有一定的限制；这些限制在格式上有所不同，但通常都是在接近 2GB 或接近 1GB 的范围内。当解码后的图像大于此大小时，stb_image 解码将失败。
//
// 此外，stb_image 还会拒绝具有任何维度设置为大于可配置的 STBI_MAX_DIMENSIONS 的图像文件。
// 设置默认的最大图片尺寸为2**24 = 16777216像素。由于上述内存限制，唯一能够正确加载具有这样尺寸的图片的方式是它具有极端的长宽比。无论如何，这里的假设是这样更大的图片可能是畸形的或恶意的。如果你确实需要加载具有单个尺寸大于这个值的图片，并且它仍然符合总尺寸限制，你可以自己定义STBI_MAX_DIMENSIONS为更大的值。

// UNICODE:

// 如果在Windows上编译并且希望使用Unicode文件名，编译时使用
// #define STBI_WINDOWS_UTF8
// 并传递utf8编码的文件名。调用stbi_convert_wchar_to_utf8将Windows的wchar_t文件名转换为utf8。
// 哲学
//
// stb 库的设计优先级如下：
//
//    1. 易于使用
//    2. 易于维护
//    3. 良好的性能
//
// 有时，我会让“良好的性能”在“易于维护”之上，为了获得最佳性能，我可能会提供一些不太易于使用的 API，以获得更高的性能，除了易于使用的 API。尽管如此，重要的是要记住，从你作为这个库的客户的角度来看，你关心的只有 #1 和 #3，而 stb 库并不将 #3 置于所有其他因素之上。
//
// 一些次要的优先级直接源自前两个优先级，其中一些提供了更明确的原因，说明为什么性能不能被强调。
//
//    - 可移植性（“易于使用”）
//    - 小的源代码占用空间（“易于维护”）
//    - 无依赖性（“易于使用”）
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
// ===========================================================================
//
// 这段代码是关于 I/O 回调和 SIMD 支持的说明。I/O 回调允许从任意来源读取数据，而 SIMD 支持则是指 JPEG 解码器在 x86 平台上自动使用 SIMD 内核。对于 ARM Neon 支持，需要显式请求。
//
// (当前代码不再支持旧的自定义SIMD API。)
//
// 在x86上，如果可用，SSE2将根据运行时测试自动使用；如果不可用，则使用通用的C版本作为后备。在ARM目标上，典型的路径是为NEON和非NEON设备分别构建（至少对于iOS和Android是这样）。因此，NEON支持通过构建标志进行切换：定义STBI_NEON以获取NEON循环。
//
// 如果出于某种原因您不想使用任何SIMD代码，或者如果您在编译时遇到问题，可以通过定义STBI_NO_SIMD来完全禁用它。
//
// ===========================================================================
//
// HDR图像支持（通过定义STBI_NO_HDR来禁用）
//
// stb_image通常支持加载HDR图像，目前特别支持Radiance .HDR文件格式。您仍然可以通过现有的方式加载任何文件。
// 接口说明：如果尝试加载HDR文件，它将自动重新映射为LDR，假定伽马值为2.2，任意比例因子默认为1；这两个常量可以通过该接口重新配置：
//
//     stbi_hdr_to_ldr_gamma(2.2f);
//     stbi_hdr_to_ldr_scale(1.0f);
//
// （注意，不要使用 _inverse_ 常量；stbi_image将适当地反转它们）。
//
// 此外，还有一个新的并行接口，用于将文件加载为（线性）浮点数，以保留完整的动态范围：
//
//    float *data = stbi_loadf(filename, &x, &y, &n, 0);
//
// 如果通过该接口加载LDR图像，这些图像将被提升为浮点值，并通过上述常量的逆运行：
//
//     stbi_ldr_to_hdr_scale(1.0f);
// 将加载的图像数据转换为 HDR 格式，使用伽马值 2.2
stbi_ldr_to_hdr_gamma(2.2f);

// 给定文件名（或打开的文件或内存块），查询最合适的接口来使用，判断图像是否为 HDR 格式
stbi_is_hdr(char *filename);

// iPhone PNG 支持：
// 可选地支持将 iPhone 格式的 PNG（存储预乘的 BGRA）转换为 RGB，即使它们在内部编码方式不同。要启用此转换，调用 stbi_convert_iphone_png_to_rgb(1)。
// 还要调用 stbi_set_unpremultiply_on_load(1) 强制每个像素进行除法运算，以去除任何预乘的 alpha，仅当图像文件明确地包含预乘的 alpha 时。
// 说明了在 iPhone 图像中可能会出现预乘数据的情况，只有当 iPhone 转换为 RGB 处理时才会发生。
//
// ===========================================================================
//
// 附加配置
//
//  - 你可以通过在创建实现之前 #定义以下符号中的一个或多个来抑制任何解码器的实现，以减少代码占用空间。
//
//        STBI_NO_JPEG
//        STBI_NO_PNG
//        STBI_NO_BMP
//        STBI_NO_PSD
//        STBI_NO_TGA
//        STBI_NO_GIF
//        STBI_NO_HDR
//        STBI_NO_PIC
//        STBI_NO_PNM   (.ppm and .pgm)
//  - 您可以仅请求特定的解码器并抑制所有其他解码器
//    （这将更具前瞻性，因为添加新的解码器不需要您显式禁用它们）：
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
//   - 如果您使用 STBI_NO_PNG（或 _ONLY_ 不包括 PNG），并且仍然
//     想要 zlib 解码器可用，请定义 STBI_SUPPORT_ZLIB
//
//  - 如果您定义了 STBI_MAX_DIMENSIONS，stb_image 将拒绝大于该尺寸的图像
//    （宽度或高度）而不进行进一步处理。
// 设置一个上限，防止恶意程序利用巨大尺寸的有效图像来进行拒绝服务攻击，强制 stb_image 分配大块内存并花费不成比例的时间来解码。默认值为 (1 << 24)，即 16777216，但这仍然非常大。

#ifndef STBI_NO_STDIO
#include <stdio.h>
#endif // STBI_NO_STDIO

#define STBI_VERSION 1

enum {
    STBI_default = 0, // 仅用于 desired_channels

    STBI_grey = 1,
    STBI_grey_alpha = 2,
    STBI_rgb = 3,
    STBI_rgb_alpha = 4
```
// 结束大括号，表示代码块的结束
};

// 包含标准库头文件
#include <stdlib.h>
// 定义无符号字符类型 stbi_uc
typedef unsigned char stbi_uc;
// 定义无符号短整型类型 stbi_us

// 如果是 C++ 环境，则使用 extern "C" 修饰
#ifdef __cplusplus
extern "C" {
#endif

// 如果未定义 STBIDEF，则根据 STB_IMAGE_STATIC 的定义来确定 STBIDEF 的值
#ifndef STBIDEF
#ifdef STB_IMAGE_STATIC
#define STBIDEF static
#else
#define STBIDEF extern
#endif
#endif

// 分割线注释
//////////////////////////////////////////////////////////////////////////////
//
// 主要的 API - 适用于任何类型的图像

// 加载图像的函数，可以通过文件名、打开文件或内存缓冲区来加载图像

// stbi_io_callbacks 结构体，包含了读取、跳过和判断文件末尾的函数指针
typedef struct {
    int (*read)(void * user, char * data, int size); // 用 'size' 字节填充 'data'，返回实际读取的字节数
    void (*skip)(void * user, int n); // 跳过接下来的 'n' 个字节，如果是负数则表示取消上次跳过的 '-n' 个字节
    int (*eof)(void * user); // 如果已经到达文件末尾则返回非零值
} stbi_io_callbacks;

// 8 位每通道的接口

// 从内存中加载图像，参数为内存缓冲区、长度、图像宽度、高度、文件中的通道数
STBIDEF stbi_uc * stbi_load_from_memory(stbi_uc const * buffer, int len, int * x, int * y, int * channels_in_file,
// 从回调函数中加载图片数据，返回像素数据指针
STBIDEF stbi_uc * stbi_load_from_callbacks(stbi_io_callbacks const * clbk, void * user, int * x, int * y,
                                           int * channels_in_file, int desired_channels);

#ifndef STBI_NO_STDIO
// 从文件名加载图片数据，返回像素数据指针
STBIDEF stbi_uc * stbi_load(char const * filename, int * x, int * y, int * channels_in_file, int desired_channels);
// 从文件指针加载图片数据，返回像素数据指针，文件指针在图像后立即指向
STBIDEF stbi_uc * stbi_load_from_file(FILE * f, int * x, int * y, int * channels_in_file, int desired_channels);
#endif

#ifndef STBI_NO_GIF
// 从内存中加载 GIF 图像数据，返回像素数据指针
STBIDEF stbi_uc * stbi_load_gif_from_memory(stbi_uc const * buffer, int len, int ** delays, int * x, int * y, int * z,
                                            int * comp, int req_comp);
#endif

#ifdef STBI_WINDOWS_UTF8
// 将宽字符转换为 UTF-8 字符串
STBIDEF int stbi_convert_wchar_to_utf8(char * buffer, size_t bufferlen, const wchar_t * input);
#endif
// 16位每通道接口

// 从内存中加载16位图像数据，返回unsigned short指针
STBIDEF stbi_us * stbi_load_16_from_memory(stbi_uc const * buffer, int len, int * x, int * y, int * channels_in_file,
                                           int desired_channels);
// 从回调函数中加载16位图像数据，返回unsigned short指针
STBIDEF stbi_us * stbi_load_16_from_callbacks(stbi_io_callbacks const * clbk, void * user, int * x, int * y,
                                              int * channels_in_file, int desired_channels);

#ifndef STBI_NO_STDIO
// 从文件中加载16位图像数据，返回unsigned short指针
STBIDEF stbi_us * stbi_load_16(char const * filename, int * x, int * y, int * channels_in_file, int desired_channels);
// 从文件指针中加载16位图像数据，返回unsigned short指针
STBIDEF stbi_us * stbi_load_from_file_16(FILE * f, int * x, int * y, int * channels_in_file, int desired_channels);
#endif

////////////////////////////////////
//
// 每通道浮点数接口
//
#ifndef STBI_NO_LINEAR
// 从内存中加载浮点数图像数据，返回float指针
STBIDEF float * stbi_loadf_from_memory(stbi_uc const * buffer, int len, int * x, int * y, int * channels_in_file,
// 从内存中加载 HDR 图像数据，并返回像素值数组
STBIDEF float * stbi_loadf_from_memory(unsigned char const * buffer, int len, int * x, int * y, int * channels_in_file, int desired_channels);

// 从回调函数中加载 HDR 图像数据，并返回像素值数组
STBIDEF float * stbi_loadf_from_callbacks(stbi_io_callbacks const * clbk, void * user, int * x, int * y, int * channels_in_file, int desired_channels);

#ifndef STBI_NO_STDIO
// 从文件中加载 HDR 图像数据，并返回像素值数组
STBIDEF float * stbi_loadf(char const * filename, int * x, int * y, int * channels_in_file, int desired_channels);
// 从文件指针中加载 HDR 图像数据，并返回像素值数组
STBIDEF float * stbi_loadf_from_file(FILE * f, int * x, int * y, int * channels_in_file, int desired_channels);
#endif
#endif

#ifndef STBI_NO_HDR
// 将 HDR 图像数据转换为 LDR，使用指定的 gamma 值
STBIDEF void stbi_hdr_to_ldr_gamma(float gamma);
// 将 HDR 图像数据转换为 LDR，使用指定的缩放比例
STBIDEF void stbi_hdr_to_ldr_scale(float scale);
#endif // STBI_NO_HDR

#ifndef STBI_NO_LINEAR
// 将 LDR 图像数据转换为 HDR，使用指定的 gamma 值
STBIDEF void stbi_ldr_to_hdr_gamma(float gamma);
// 将 LDR 图像数据转换为 HDR，使用指定的缩放比例
STBIDEF void stbi_ldr_to_hdr_scale(float scale);
#endif // STBI_NO_LINEAR
// 使用回调函数和用户数据判断是否为 HDR 格式的图片，返回 1 表示是 HDR，0 表示不是 HDR
STBIDEF int stbi_is_hdr_from_callbacks(stbi_io_callbacks const * clbk, void * user);

// 从内存中判断是否为 HDR 格式的图片，返回 1 表示是 HDR，0 表示不是 HDR
STBIDEF int stbi_is_hdr_from_memory(stbi_uc const * buffer, int len);

#ifndef STBI_NO_STDIO
// 从文件名判断是否为 HDR 格式的图片，返回 1 表示是 HDR，0 表示不是 HDR
STBIDEF int stbi_is_hdr(char const * filename);

// 从文件指针判断是否为 HDR 格式的图片，返回 1 表示是 HDR，0 表示不是 HDR
STBIDEF int stbi_is_hdr_from_file(FILE * f);
#endif // STBI_NO_STDIO

// 获取失败的简要原因
// 在大多数编译器（以及所有现代主流编译器）上，这是线程安全的
STBIDEF const char * stbi_failure_reason(void);

// 释放加载的图片 -- 这只是 free() 函数的调用
STBIDEF void stbi_image_free(void * retval_from_stbi_load);

// 获取图片的尺寸和通道数，而不完全解码
STBIDEF int stbi_info_from_memory(stbi_uc const * buffer, int len, int * x, int * y, int * comp);
STBIDEF int stbi_info_from_callbacks(stbi_io_callbacks const * clbk, void * user, int * x, int * y, int * comp);
STBIDEF int stbi_is_16_bit_from_memory(stbi_uc const * buffer, int len);
STBIDEF int stbi_is_16_bit_from_callbacks(stbi_io_callbacks const * clbk, void * user);
// 如果没有定义 STBI_NO_STDIO，则定义以下函数
STBIDEF int stbi_info(char const * filename, int * x, int * y, int * comp);
STBIDEF int stbi_info_from_file(FILE * f, int * x, int * y, int * comp);
STBIDEF int stbi_is_16_bit(char const * filename);
STBIDEF int stbi_is_16_bit_from_file(FILE * f);

// 如果图像格式明确注明具有预乘 alpha 通道，则返回文件中存储的颜色。设置此标志以强制取消预乘。如果取消预乘溢出，则结果是未定义的。
STBIDEF void stbi_set_unpremultiply_on_load(int flag_true_if_should_unpremultiply);

// 指示是否应将 iPhone 图像处理回规范格式，还是直接传递它们“原样”
STBIDEF void stbi_convert_iphone_png_to_rgb(int flag_true_if_should_convert);

// 垂直翻转图像，使输出数组中的第一个像素为左下角
STBIDEF void stbi_set_flip_vertically_on_load(int flag_true_if_should_flip);
// 设置在加载线程上只对加载的图像应用预乘
// 如果编译器支持线程局部变量，则此函数仅在调用该函数的线程上可用；
// 如果编译器不支持，调用它将无法链接
STBIDEF void stbi_set_unpremultiply_on_load_thread(int flag_true_if_should_unpremultiply);

// 设置在加载线程上将 iPhone PNG 转换为 RGB
STBIDEF void stbi_convert_iphone_png_to_rgb_thread(int flag_true_if_should_convert);

// 设置在加载线程上翻转图像
STBIDEF void stbi_set_flip_vertically_on_load_thread(int flag_true_if_should_flip);

// ZLIB 客户端 - 由 PNG 使用，也可用于其他目的

// 根据给定的缓冲区和长度，猜测初始大小并解码 ZLIB 数据，返回分配的内存指针和解码后的长度
STBIDEF char * stbi_zlib_decode_malloc_guesssize(const char * buffer, int len, int initial_size, int * outlen);

// 根据给定的缓冲区和长度，猜测初始大小并解码带有标头标志的 ZLIB 数据，返回分配的内存指针和解码后的长度
STBIDEF char * stbi_zlib_decode_malloc_guesssize_headerflag(const char * buffer, int len, int initial_size, int * outlen, int parse_header);

// 根据给定的缓冲区和长度，解码 ZLIB 数据，返回分配的内存指针和解码后的长度
STBIDEF char * stbi_zlib_decode_malloc(const char * buffer, int len, int * outlen);

// 根据给定的输入缓冲区和长度，解码 ZLIB 数据到输出缓冲区，返回解码后的长度
STBIDEF int stbi_zlib_decode_buffer(char * obuffer, int olen, const char * ibuffer, int ilen);

// 根据给定的缓冲区和长度，解码不带标头的 ZLIB 数据，返回分配的内存指针和解码后的长度
STBIDEF char * stbi_zlib_decode_noheader_malloc(const char * buffer, int len, int * outlen);

// 根据给定的输入缓冲区和长度，解码不带标头的 ZLIB 数据到输出缓冲区，返回解码后的长度
STBIDEF int stbi_zlib_decode_noheader_buffer(char * obuffer, int olen, const char * ibuffer, int ilen);

#ifdef __cplusplus
}
#endif
// 结束头文件

#ifdef STBI_INCLUDE_STB_IMAGE_H
// 如果定义了 STBI_INCLUDE_STB_IMAGE_H，则执行以下代码

#ifdef STB_IMAGE_IMPLEMENTATION
// 如果定义了 STB_IMAGE_IMPLEMENTATION，则执行以下代码

#if defined(STBI_ONLY_JPEG) || defined(STBI_ONLY_PNG) || defined(STBI_ONLY_BMP) || defined(STBI_ONLY_TGA) ||                   \
    defined(STBI_ONLY_GIF) || defined(STBI_ONLY_PSD) || defined(STBI_ONLY_HDR) || defined(STBI_ONLY_PIC) ||                    \
    defined(STBI_ONLY_PNM) || defined(STBI_ONLY_ZLIB)
// 如果定义了特定的图片格式，则执行以下代码

#ifndef STBI_ONLY_JPEG
// 如果没有定义 STBI_ONLY_JPEG，则定义 STBI_NO_JPEG
#define STBI_NO_JPEG
#endif
#ifndef STBI_ONLY_PNG
// 如果没有定义 STBI_ONLY_PNG，则定义 STBI_NO_PNG
#define STBI_NO_PNG
#endif
#ifndef STBI_ONLY_BMP
// 如果没有定义 STBI_ONLY_BMP，则定义 STBI_NO_BMP
#define STBI_NO_BMP
// 如果未定义 STBI_ONLY_PSD，则定义 STBI_NO_PSD
#ifndef STBI_ONLY_PSD
#define STBI_NO_PSD
#endif
// 如果未定义 STBI_ONLY_TGA，则定义 STBI_NO_TGA
#ifndef STBI_ONLY_TGA
#define STBI_NO_TGA
#endif
// 如果未定义 STBI_ONLY_GIF，则定义 STBI_NO_GIF
#ifndef STBI_ONLY_GIF
#define STBI_NO_GIF
#endif
// 如果未定义 STBI_ONLY_HDR，则定义 STBI_NO_HDR
#ifndef STBI_ONLY_HDR
#define STBI_NO_HDR
#endif
// 如果未定义 STBI_ONLY_PIC，则定义 STBI_NO_PIC
#ifndef STBI_ONLY_PIC
#define STBI_NO_PIC
#endif
// 如果未定义 STBI_ONLY_PNM，则定义 STBI_NO_PNM
#ifndef STBI_ONLY_PNM
#define STBI_NO_PNM
#endif
#endif
// 如果定义了 STBI_NO_PNG 且未定义 STBI_SUPPORT_ZLIB 且未定义 STBI_NO_ZLIB，则定义 STBI_NO_ZLIB
#define STBI_NO_ZLIB

// 包含一些标准库的头文件
#include <limits.h> // 包含一些整数限制的宏
#include <stdarg.h> // 提供了函数原型的功能
#include <stddef.h> // 在 OSX 上定义了指针差值类型 ptrdiff_t
#include <stdlib.h> // 包含了一些常用的函数，如动态内存分配、随机数生成、排序等
#include <string.h> // 包含了一些字符串处理函数

// 如果未定义 STBI_NO_LINEAR 或未定义 STBI_NO_HDR，则包含数学库的头文件
#if !defined(STBI_NO_LINEAR) || !defined(STBI_NO_HDR)
#include <math.h> // 包含了一些数学函数，如 ldexp、pow 等
#endif

// 如果未定义 STBI_NO_STDIO，则包含标准输入输出库的头文件
#ifndef STBI_NO_STDIO
#include <stdio.h> // 包含了一些输入输出函数
#endif

// 如果未定义 STBI_ASSERT
#ifndef STBI_ASSERT
#include <assert.h>
// 包含 assert.h 头文件，用于断言
#define STBI_ASSERT(x) assert(x)
// 定义宏 STBI_ASSERT，用于断言
#endif

#ifdef __cplusplus
// 如果是 C++ 环境
#define STBI_EXTERN extern "C"
// 定义宏 STBI_EXTERN，用于声明外部 C 函数
#else
// 如果不是 C++ 环境
#define STBI_EXTERN extern
// 定义宏 STBI_EXTERN，用于声明外部函数
#endif

#ifndef _MSC_VER
// 如果不是 Microsoft Visual C++ 编译器
#ifdef __cplusplus
// 如果是 C++ 环境
#define stbi_inline inline
// 定义宏 stbi_inline，用于内联函数
#else
// 如果不是 C++ 环境
#define stbi_inline
// 定义宏 stbi_inline，为空
#endif
#else
// 如果是 Microsoft Visual C++ 编译器
#define stbi_inline __forceinline
// 定义宏 stbi_inline，用于强制内联函数
#endif
#ifndef STBI_NO_THREAD_LOCALS
// 如果没有定义 STBI_NO_THREAD_LOCALS，则进行以下条件编译
#if defined(__cplusplus) && __cplusplus >= 201103L
// 如果是 C++11 及以上版本，则定义 STBI_THREAD_LOCAL 为 thread_local
#define STBI_THREAD_LOCAL thread_local
#elif defined(__GNUC__) && __GNUC__ < 5
// 如果是 GCC 编译器且版本小于 5，则定义 STBI_THREAD_LOCAL 为 __thread
#define STBI_THREAD_LOCAL __thread
#elif defined(_MSC_VER)
// 如果是 MSVC 编译器，则定义 STBI_THREAD_LOCAL 为 __declspec(thread)
#define STBI_THREAD_LOCAL __declspec(thread)
#elif defined(__STDC_VERSION__) && __STDC_VERSION__ >= 201112L && !defined(__STDC_NO_THREADS__)
// 如果是 C11 及以上版本且不是禁用线程，则定义 STBI_THREAD_LOCAL 为 _Thread_local
#define STBI_THREAD_LOCAL _Thread_local
#endif

#ifndef STBI_THREAD_LOCAL
// 如果没有定义 STBI_THREAD_LOCAL，则进行以下条件编译
#if defined(__GNUC__)
// 如果是 GCC 编译器，则定义 STBI_THREAD_LOCAL 为 __thread
#define STBI_THREAD_LOCAL __thread
#endif
#endif
#endif

#if defined(_MSC_VER) || defined(__SYMBIAN32__)
// 如果是 MSVC 编译器或者是 __SYMBIAN32__ 平台，则定义 stbi__uint16 为 unsigned short
typedef unsigned short stbi__uint16;
// 定义了几种不同大小的有符号和无符号整数类型
typedef signed short stbi__int16; // 有符号短整型
typedef unsigned int stbi__uint32; // 无符号整型
typedef signed int stbi__int32; // 有符号整型
#else
#include <stdint.h>
typedef uint16_t stbi__uint16; // 无符号短整型
typedef int16_t stbi__int16; // 有符号短整型
typedef uint32_t stbi__uint32; // 无符号整型
typedef int32_t stbi__int32; // 有符号整型
#endif

// 如果大小不正确，应该产生编译器错误
typedef unsigned char validate_uint32[sizeof(stbi__uint32) == 4 ? 1 : -1]; // 验证无符号整型大小是否为4字节

#ifdef _MSC_VER
#define STBI_NOTUSED(v) (void)(v) // 如果是 MSC 编译器，定义一个宏用于标记变量未使用
#else
#define STBI_NOTUSED(v) (void)sizeof(v) // 如果不是 MSC 编译器，定义一个宏用于标记变量未使用
#endif
#ifdef _MSC_VER
// 如果编译器是 Microsoft Visual C++，则定义 STBI_HAS_LROTL
#define STBI_HAS_LROTL
#endif

#ifdef STBI_HAS_LROTL
// 如果定义了 STBI_HAS_LROTL，则使用 _lrotl 函数进行循环左移操作
#define stbi_lrot(x, y) _lrotl(x, y)
#else
// 如果未定义 STBI_HAS_LROTL，则使用位操作进行循环左移操作
#define stbi_lrot(x, y) (((x) << (y)) | ((x) >> (-(y)&31)))
#endif

#if defined(STBI_MALLOC) && defined(STBI_FREE) && (defined(STBI_REALLOC) || defined(STBI_REALLOC_SIZED))
// 如果同时定义了 STBI_MALLOC、STBI_FREE，并且定义了 STBI_REALLOC 或 STBI_REALLOC_SIZED，则通过
// 否则，如果同时未定义 STBI_MALLOC、STBI_FREE、STBI_REALLOC 和 STBI_REALLOC_SIZED，则通过
#elif !defined(STBI_MALLOC) && !defined(STBI_FREE) && !defined(STBI_REALLOC) && !defined(STBI_REALLOC_SIZED)
// 否则，如果同时未定义 STBI_MALLOC、STBI_FREE、STBI_REALLOC 和 STBI_REALLOC_SIZED，则通过
#else
// 否则，抛出错误，要求同时定义或不定义 STBI_MALLOC、STBI_FREE、STBI_REALLOC 和 STBI_REALLOC_SIZED
#error "Must define all or none of STBI_MALLOC, STBI_FREE, and STBI_REALLOC (or STBI_REALLOC_SIZED)."
#endif

#ifndef STBI_MALLOC
// 如果未定义 STBI_MALLOC，则定义为 malloc 函数
#define STBI_MALLOC(sz) malloc(sz)
// 定义了一个宏，用于重新分配内存
#define STBI_REALLOC(p, newsz) realloc(p, newsz)
// 定义了一个宏，用于释放内存
#define STBI_FREE(p) free(p)
#endif

#ifndef STBI_REALLOC_SIZED
// 如果未定义 STBI_REALLOC_SIZED 宏，则定义为 STBI_REALLOC 宏
#define STBI_REALLOC_SIZED(p, oldsz, newsz) STBI_REALLOC(p, newsz)
#endif

// 检测当前平台是否为 x86/x64
#if defined(__x86_64__) || defined(_M_X64)
// 如果是 x86_64 平台，则定义 STBI__X64_TARGET 宏
#define STBI__X64_TARGET
// 如果是 i386 平台，则定义 STBI__X86_TARGET 宏
#elif defined(__i386) || defined(_M_IX86)
#define STBI__X86_TARGET
#endif

#if defined(__GNUC__) && defined(STBI__X86_TARGET) && !defined(__SSE2__) && !defined(STBI_NO_SIMD)
// 如果是使用 GCC 编译，并且是 x86 平台，并且未定义 __SSE2__ 宏，并且未定义 STBI_NO_SIMD 宏
// 则执行以下代码
// gcc 不支持 SSE2 内联汇编，除非使用 -msse2 编译选项
// 这意味着它可以在所有地方使用 SSE2。这很不幸，
// 但以前尝试在运行时检测 SSE2 函数时导致了许多问题。架构扩展的方式是
// 如果使用的是 GCC/Clang 编译器，并且不适合单文件库，则定义 STBI_NO_SIMD
// 新的行为：如果使用 -msse2 编译，则直接使用 SSE2，不进行检测；如果没有使用，则完全不使用
#define STBI_NO_SIMD
#endif

#if defined(__MINGW32__) && defined(STBI__X86_TARGET) && !defined(STBI_MINGW_ENABLE_SSE2) && !defined(STBI_NO_SIMD)
// 注意，__MINGW32__ 实际上并不意味着 32 位，因此我们必须避免使用 STBI__X64_TARGET
//
// 32 位 MinGW 希望 ESP 寄存器的值是 16 字节对齐的，但这不符合 Windows ABI，VC++ 和 Windows DLLs 也不保持这个不变。
// 因此，在 32 位 MinGW 上启用 SSE2 是危险的，除非同时启用 "-mstackrealign"。
//
// 更多信息请参见 https://github.com/nothings/stb/issues/81
//
// 因此，默认情况下在 32 位 MinGW 上不启用 SSE2。如果你已经阅读到这里并在构建设置中添加了 -mstackrealign，请随意 #define STBI_MINGW_ENABLE_SSE2。
#define STBI_NO_SIMD
#endif
#if !defined(STBI_NO_SIMD) && (defined(STBI__X86_TARGET) || defined(STBI__X64_TARGET))
// 如果未定义STBI_NO_SIMD并且定义了STBI__X86_TARGET或STBI__X64_TARGET，则执行以下代码
#define STBI_SSE2
// 定义STBI_SSE2

#include <emmintrin.h>
// 包含emmintrin.h头文件，该头文件包含了SSE2指令集的函数和宏

#ifdef _MSC_VER
// 如果是在Microsoft Visual C++编译器下编译以下代码

#if _MSC_VER >= 1400 // not VC6
// 如果编译器版本大于等于1400（不包括VC6），则执行以下代码
#include <intrin.h>  // __cpuid
// 包含intrin.h头文件，该头文件包含了CPUID指令的函数__cpuid

static int stbi__cpuid3(void) {
    // 定义stbi__cpuid3函数，用于获取CPU信息
    int info[4];
    __cpuid(info, 1);
    // 使用CPUID指令获取CPU信息，存储在info数组中
    return info[3];
    // 返回CPU信息中的第四个元素
#else
static int stbi__cpuid3(void) {
    // 如果编译器版本不满足上述条件，则执行以下代码
    int res;
    __asm {
      mov  eax,1
      cpuid
    }
    // 使用汇编语言内联指令获取CPU信息
// 将 edx 寄存器的值移动到 res 变量中
mov  res,edx
// 返回 res 变量的值
}
return res;
}
#endif

// 定义一个宏，用于指定变量的内存对齐方式为 16 字节
#define STBI_SIMD_ALIGN(type, name) __declspec(align(16)) type name

// 如果不是 VC++ 编译器，则使用 GCC 风格的语法
#else // assume GCC-style if not VC++
// 定义一个宏，用于指定变量的内存对齐方式为 16 字节
#define STBI_SIMD_ALIGN(type, name) type name __attribute__((aligned(16)))

// 如果不禁用 JPEG 并且支持 SSE2 指令集，则检查 SSE2 是否可用
#if !defined(STBI_NO_JPEG) && defined(STBI_SSE2)
static int stbi__sse2_available(void) {
// 如果我们在 GCC/Clang 上尝试编译这段代码，那意味着 -msse2 已经开启，这意味着编译器可以随意使用 SSE2 指令，我们也可以。
return 1;
}
#endif

#endif
#endif

// ARM NEON
#if defined(STBI_NO_SIMD) && defined(STBI_NEON)
// 如果 STBI_NO_SIMD 被定义，并且 STBI_NEON 也被定义，那么取消 STBI_NEON 的定义
#undef STBI_NEON
#endif

#ifdef STBI_NEON
// 包含 ARM NEON 头文件
#include <arm_neon.h>
#ifdef _MSC_VER
// 如果是在 MSVC 编译器下，使用 __declspec(align(16)) 来指定变量的内存对齐方式
#define STBI_SIMD_ALIGN(type, name) __declspec(align(16)) type name
#else
// 定义了一个宏，用于指定变量的对齐方式
#define STBI_SIMD_ALIGN(type, name) type name __attribute__((aligned(16)))
#endif
#endif

// 如果未定义STBI_SIMD_ALIGN，则重新定义STBI_SIMD_ALIGN宏
#ifndef STBI_SIMD_ALIGN
#define STBI_SIMD_ALIGN(type, name) type name
#endif

// 如果未定义STBI_MAX_DIMENSIONS，则定义STBI_MAX_DIMENSIONS为2^24
#ifndef STBI_MAX_DIMENSIONS
#define STBI_MAX_DIMENSIONS (1 << 24)
#endif

///////////////////////////////////////////////
//
//  stbi__context struct and start_xxx functions

// stbi__context结构是我们所有图像使用的基本上下文，因此它包含所有IO上下文，以及一些基本的图像信息
typedef struct {
    stbi__uint32 img_x, img_y;
```

在这段代码中，宏定义了一些常用的变量和结构体，以及一些基本的图像信息。
// 定义变量img_n和img_out_n，用于存储图片的通道数和输出通道数
int img_n, img_out_n;

// 定义io和io_user_data，用于存储IO回调函数和用户数据
stbi_io_callbacks io;
void * io_user_data;

// 定义read_from_callbacks和buflen，用于标识是否从回调函数中读取数据和缓冲区长度
int read_from_callbacks;
int buflen;

// 定义buffer_start和callback_already_read，用于存储缓冲区起始位置和已读取的回调数据长度
stbi_uc buffer_start[128];
int callback_already_read;

// 定义img_buffer、img_buffer_end、img_buffer_original和img_buffer_original_end，用于存储图片数据和原始图片数据的起始和结束位置
stbi_uc *img_buffer, *img_buffer_end;
stbi_uc *img_buffer_original, *img_buffer_original_end;
} stbi__context;

// 重新填充缓冲区的函数
static void stbi__refill_buffer(stbi__context * s);

// 初始化内存解码上下文
static void stbi__start_mem(stbi__context * s, stbi_uc const * buffer, int len) {
    // 将IO读取函数设置为NULL，表示不使用IO回调函数
    s->io.read = NULL;
    // 将read_from_callbacks设置为0，表示不从回调函数中读取数据
    s->read_from_callbacks = 0;
    // 将callback_already_read设置为0，表示回调函数尚未读取任何数据
    s->callback_already_read = 0;
    // 将img_buffer和img_buffer_original指向传入的buffer，表示图像数据的起始位置
    s->img_buffer = s->img_buffer_original = (stbi_uc *)buffer;
    // 将img_buffer_end和img_buffer_original_end指向buffer加上长度len的位置，表示图像数据的结束位置
    s->img_buffer_end = s->img_buffer_original_end = (stbi_uc *)buffer + len;
}

// 初始化基于回调的上下文
static void stbi__start_callbacks(stbi__context * s, stbi_io_callbacks * c, void * user) {
    // 将io字段设置为传入的回调函数
    s->io = *c;
    // 将io_user_data字段设置为传入的用户数据
    s->io_user_data = user;
    // 将buflen设置为buffer_start的大小
    s->buflen = sizeof(s->buffer_start);
    // 将read_from_callbacks设置为1，表示使用回调函数读取数据
    s->read_from_callbacks = 1;
    // 将callback_already_read设置为0，表示回调函数尚未读取任何数据
    s->callback_already_read = 0;
    // 将img_buffer和img_buffer_original指向buffer_start，表示图像数据的起始位置
    s->img_buffer = s->img_buffer_original = s->buffer_start;
    // 调用stbi__refill_buffer函数，填充缓冲区
    stbi__refill_buffer(s);
    // 将img_buffer_original_end设置为img_buffer_end，表示图像数据的结束位置
    s->img_buffer_original_end = s->img_buffer_end;
}

#ifndef STBI_NO_STDIO

// 从标准IO流中读取数据的函数
static int stbi__stdio_read(void * user, char * data, int size) { return (int)fread(data, 1, size, (FILE *)user); }
// 定义一个函数，用于跳过文件流中的指定字节数
static void stbi__stdio_skip(void * user, int n) {
    int ch;
    // 移动文件流的读取位置
    fseek((FILE *)user, n, SEEK_CUR);
    ch = fgetc((FILE *)user); /* have to read a byte to reset feof()'s flag */
    // 读取一个字节以重置 feof() 的标志位
    if (ch != EOF) {
        // 如果读取的字节不是文件结束符，则将其推回流中
        ungetc(ch, (FILE *)user); /* push byte back onto stream if valid. */
    }
}

// 定义一个函数，用于检查文件流是否已经到达文件末尾
static int stbi__stdio_eof(void * user) { return feof((FILE *)user) || ferror((FILE *)user); }

// 定义一个结构体，包含了读取、跳过和检查文件流末尾的函数指针
static stbi_io_callbacks stbi__stdio_callbacks = {
    stbi__stdio_read,
    stbi__stdio_skip,
    stbi__stdio_eof,
};

// 定义一个函数，用于初始化 stbi__context 结构体，设置回调函数为 stbi__stdio_callbacks，并传入文件流指针
static void stbi__start_file(stbi__context * s, FILE * f) { stbi__start_callbacks(s, &stbi__stdio_callbacks, (void *)f); }
// 重新定位流的位置到初始缓冲区的开头
static void stbi__rewind(stbi__context * s) {
    // 概念上，rewind 应该将流的位置重置到开头，但是我们只将其重置到初始缓冲区的开头，
    // 因为我们只在 'test' 之后使用它，而 'test' 只查看最多 92 个字节
    s->img_buffer = s->img_buffer_original;
    s->img_buffer_end = s->img_buffer_original_end;
}

// 定义通道顺序的枚举
enum { STBI_ORDER_RGB, STBI_ORDER_BGR };

// 定义图像信息的结构体
typedef struct {
    int bits_per_channel; // 每个通道的位数
    int num_channels; // 通道数
    int channel_order; // 通道顺序
} stbi__result_info;
// 如果未定义 STBI_NO_JPEG，则声明 stbi__jpeg_test、stbi__jpeg_load、stbi__jpeg_info 函数
static int stbi__jpeg_test(stbi__context * s);
static void * stbi__jpeg_load(stbi__context * s, int * x, int * y, int * comp, int req_comp, stbi__result_info * ri);
static int stbi__jpeg_info(stbi__context * s, int * x, int * y, int * comp);

// 如果未定义 STBI_NO_PNG，则声明 stbi__png_test、stbi__png_load、stbi__png_info、stbi__png_is16 函数
static int stbi__png_test(stbi__context * s);
static void * stbi__png_load(stbi__context * s, int * x, int * y, int * comp, int req_comp, stbi__result_info * ri);
static int stbi__png_info(stbi__context * s, int * x, int * y, int * comp);
static int stbi__png_is16(stbi__context * s);

// 如果未定义 STBI_NO_BMP，则声明 stbi__bmp_test、stbi__bmp_load、stbi__bmp_info 函数
static int stbi__bmp_test(stbi__context * s);
static void * stbi__bmp_load(stbi__context * s, int * x, int * y, int * comp, int req_comp, stbi__result_info * ri);
static int stbi__bmp_info(stbi__context * s, int * x, int * y, int * comp);
// 检测是否为 TGA 格式的图像
static int stbi__tga_test(stbi__context * s);

// 加载 TGA 格式的图像数据
static void * stbi__tga_load(stbi__context * s, int * x, int * y, int * comp, int req_comp, stbi__result_info * ri);

// 获取 TGA 格式的图像信息
static int stbi__tga_info(stbi__context * s, int * x, int * y, int * comp);

// 检测是否为 PSD 格式的图像
static int stbi__psd_test(stbi__context * s);

// 加载 PSD 格式的图像数据
static void * stbi__psd_load(stbi__context * s, int * x, int * y, int * comp, int req_comp, stbi__result_info * ri, int bpc);

// 获取 PSD 格式的图像信息
static int stbi__psd_info(stbi__context * s, int * x, int * y, int * comp);

// 判断 PSD 图像是否为16位
static int stbi__psd_is16(stbi__context * s);

// 检测是否为 HDR 格式的图像
static int stbi__hdr_test(stbi__context * s);

// 加载 HDR 格式的图像数据
static float * stbi__hdr_load(stbi__context * s, int * x, int * y, int * comp, int req_comp, stbi__result_info * ri);

// 获取 HDR 格式的图像信息
static int stbi__hdr_info(stbi__context * s, int * x, int * y, int * comp);

// 检测是否为 PIC 格式的图像
static int stbi__pic_test(stbi__context * s);
// 定义 stbi__pic_load 函数，用于加载 PIC 格式的图片
static void * stbi__pic_load(stbi__context * s, int * x, int * y, int * comp, int req_comp, stbi__result_info * ri);

// 定义 stbi__pic_info 函数，用于获取 PIC 格式图片的信息
static int stbi__pic_info(stbi__context * s, int * x, int * y, int * comp);
#endif

#ifndef STBI_NO_GIF
// 定义 stbi__gif_test 函数，用于测试是否为 GIF 格式的图片
static int stbi__gif_test(stbi__context * s);
// 定义 stbi__gif_load 函数，用于加载 GIF 格式的图片
static void * stbi__gif_load(stbi__context * s, int * x, int * y, int * comp, int req_comp, stbi__result_info * ri);
// 定义 stbi__load_gif_main 函数，用于加载 GIF 格式的图片的主要函数
static void * stbi__load_gif_main(stbi__context * s, int ** delays, int * x, int * y, int * z, int * comp, int req_comp);
// 定义 stbi__gif_info 函数，用于获取 GIF 格式图片的信息
static int stbi__gif_info(stbi__context * s, int * x, int * y, int * comp);
#endif

#ifndef STBI_NO_PNM
// 定义 stbi__pnm_test 函数，用于测试是否为 PNM 格式的图片
static int stbi__pnm_test(stbi__context * s);
// 定义 stbi__pnm_load 函数，用于加载 PNM 格式的图片
static void * stbi__pnm_load(stbi__context * s, int * x, int * y, int * comp, int req_comp, stbi__result_info * ri);
// 定义 stbi__pnm_info 函数，用于获取 PNM 格式图片的信息
static int stbi__pnm_info(stbi__context * s, int * x, int * y, int * comp);
// 定义 stbi__pnm_is16 函数，用于判断是否为 16 位 PNM 格式的图片
static int stbi__pnm_is16(stbi__context * s);
#endif

// 定义一个静态函数
static
#ifdef STBI_THREAD_LOCAL
// 定义了一个线程本地存储的宏，用于声明线程本地变量
    STBI_THREAD_LOCAL
#endif
    // 定义了一个指向失败原因的全局变量
    const char * stbi__g_failure_reason;

// 返回失败原因的函数
STBIDEF const char * stbi_failure_reason(void) { return stbi__g_failure_reason; }

#ifndef STBI_NO_FAILURE_STRINGS
// 如果没有定义STBI_NO_FAILURE_STRINGS，则定义一个返回错误的函数
static int stbi__err(const char * str) {
    // 将错误原因赋值给全局变量
    stbi__g_failure_reason = str;
    return 0;
}
#endif

// 分配内存的函数
static void * stbi__malloc(size_t size) { return STBI_MALLOC(size); }

// stb_image广泛使用int类型，包括偏移计算。
// 因此，即使在64位目标上，当前代码也只能支持INT_MAX大小的解码图像。
// 对于预期的用例，这不是一个重大的限制。
//
// 确保我们的大小计算不会溢出。因此，需要一些辅助函数来进行大小计算，确保它们是非负的，且不会溢出。

// 如果两个数相加的结果有效，则返回1，溢出时返回0。
// 负数被视为无效。
static int stbi__addsizes_valid(int a, int b) {
    if (b < 0)
        return 0;
    // 现在 0 <= b <= INT_MAX，因此也有
    // 0 <= INT_MAX - b <= INTMAX。
    // 而 "a + b <= INT_MAX"（可能会溢出）等同于 a <= INT_MAX - b（不会溢出）
    return a <= INT_MAX - b;
}

// 如果两个数相乘的结果有效，则返回1，溢出时返回0。
// 负数被视为无效。
static int stbi__mul2sizes_valid(int a, int b) {
// 如果 a 或 b 小于 0，则返回 0
if (a < 0 || b < 0)
    return 0;
// 如果 b 等于 0，则返回 1，因为乘以 0 总是安全的
if (b == 0)
    return 1; 
// 通过可移植的方式检查 a*b 是否会溢出
return a <= INT_MAX / b;
}

#if !defined(STBI_NO_JPEG) || !defined(STBI_NO_PNG) || !defined(STBI_NO_TGA) || !defined(STBI_NO_HDR)
// 如果 "a*b + add" 没有负项/因子并且不会溢出，则返回 1
static int stbi__mad2sizes_valid(int a, int b, int add) {
    return stbi__mul2sizes_valid(a, b) && stbi__addsizes_valid(a * b, add);
}
#endif

// 如果 "a*b*c + add" 没有负项/因子并且不会溢出，则返回 1
static int stbi__mad3sizes_valid(int a, int b, int c, int add) {
    return stbi__mul2sizes_valid(a, b) && stbi__mul2sizes_valid(a * b, c) && stbi__addsizes_valid(a * b * c, add);
}
// 如果"a*b*c*d + add"没有负项/因子并且没有溢出，则返回1
#if !defined(STBI_NO_LINEAR) || !defined(STBI_NO_HDR) || !defined(STBI_NO_PNM)
static int stbi__mad4sizes_valid(int a, int b, int c, int d, int add) {
    // 检查a和b的乘积是否有效
    return stbi__mul2sizes_valid(a, b) && 
           // 检查a*b和c的乘积是否有效
           stbi__mul2sizes_valid(a * b, c) && 
           // 检查a*b*c和d的乘积是否有效
           stbi__mul2sizes_valid(a * b * c, d) &&
           // 检查a*b*c*d和add的和是否有效
           stbi__addsizes_valid(a * b * c * d, add);
}
#endif

#if !defined(STBI_NO_JPEG) || !defined(STBI_NO_PNG) || !defined(STBI_NO_TGA) || !defined(STBI_NO_HDR)
// 使用大小溢出检查进行动态内存分配
static void * stbi__malloc_mad2(int a, int b, int add) {
    // 如果a*b+add的大小溢出，则返回NULL
    if (!stbi__mad2sizes_valid(a, b, add))
        return NULL;
    // 动态分配内存
    return stbi__malloc(a * b + add);
}
#endif

// 使用大小溢出检查进行动态内存分配
static void * stbi__malloc_mad3(int a, int b, int c, int add) {
    // 如果a*b*c+add的大小溢出，则返回NULL
    if (!stbi__mad3sizes_valid(a, b, c, add))
        return NULL;
// 返回分配的内存空间的指针，大小为 a * b * c + add
return stbi__malloc(a * b * c + add);
}

#if !defined(STBI_NO_LINEAR) || !defined(STBI_NO_HDR) || !defined(STBI_NO_PNM)
// 返回分配的内存空间的指针，大小为 a * b * c * d + add
static void * stbi__malloc_mad4(int a, int b, int c, int d, int add) {
    // 检查参数是否有效
    if (!stbi__mad4sizes_valid(a, b, c, d, add))
        return NULL;
    return stbi__malloc(a * b * c * d + add);
}
#endif

// 如果两个有符号整数的和有效（在 -2^31 和 2^31-1 之间，包括边界），返回1；溢出返回0
static int stbi__addints_valid(int a, int b) {
    // 如果 a 和 b 的符号不同，则没有溢出
    if ((a >= 0) != (b >= 0))
        return 1;
    // 如果 a 和 b 都小于 0，则判断 a 是否大于等于 INT_MIN - b
    if (a < 0 && b < 0)
        return a >= INT_MIN - b;
    // 否则判断 a 是否小于等于 INT_MAX - b
    return a <= INT_MAX - b;
}
// 返回1，如果两个有符号短整型的乘积有效，溢出时返回0。
static int stbi__mul2shorts_valid(short a, short b) {
    if (b == 0 || b == -1)
        return 1; // 如果b为0或-1，则返回1；乘以0总是0；检查-1以防止SHRT_MIN/b溢出
    if ((a >= 0) == (b >= 0))
        return a <= SHRT_MAX / b; // 乘积为正数，所以类似于mul2sizes_valid
    if (b < 0)
        return a <= SHRT_MIN / b; // 和a * b >= SHRT_MIN相同
    return a >= SHRT_MIN / b;
}

// stbi__err - 错误
// stbi__errpf - 返回指向浮点数的指针的错误
// stbi__errpuc - 返回指向无符号字符的指针的错误

#ifdef STBI_NO_FAILURE_STRINGS
#define stbi__err(x, y) 0
#elif defined(STBI_FAILURE_USERMSG)
#define stbi__err(x, y) stbi__err(y)
#else
// 定义 stbi__err(x, y) 宏，如果出错则返回错误信息
#define stbi__err(x, y) stbi__err(x)
#endif

// 定义 stbi__errpf(x, y) 宏，如果出错则返回浮点数指针
#define stbi__errpf(x, y) ((float *)(size_t)(stbi__err(x, y) ? NULL : NULL))
// 定义 stbi__errpuc(x, y) 宏，如果出错则返回无符号字符指针
#define stbi__errpuc(x, y) ((unsigned char *)(size_t)(stbi__err(x, y) ? NULL : NULL))

// 释放由 stbi_load 返回的内存
STBIDEF void stbi_image_free(void * retval_from_stbi_load) { STBI_FREE(retval_from_stbi_load); }

// 如果未定义 STBI_NO_LINEAR，则定义 stbi__ldr_to_hdr 函数
#ifndef STBI_NO_LINEAR
static float * stbi__ldr_to_hdr(stbi_uc * data, int x, int y, int comp);
#endif

// 如果未定义 STBI_NO_HDR，则定义 stbi__hdr_to_ldr 函数
#ifndef STBI_NO_HDR
static stbi_uc * stbi__hdr_to_ldr(float * data, int x, int y, int comp);
#endif

// 定义一个全局变量，用于控制是否在加载时垂直翻转图像
static int stbi__vertically_flip_on_load_global = 0;

// 设置是否在加载时垂直翻转图像
STBIDEF void stbi_set_flip_vertically_on_load(int flag_true_if_should_flip) {
    stbi__vertically_flip_on_load_global = flag_true_if_should_flip;
// 如果未定义 STBI_THREAD_LOCAL，则使用全局变量 stbi__vertically_flip_on_load_global
#ifndef STBI_THREAD_LOCAL
#define stbi__vertically_flip_on_load stbi__vertically_flip_on_load_global
// 如果定义了 STBI_THREAD_LOCAL，则使用线程本地变量 stbi__vertically_flip_on_load_local，并设置 stbi__vertically_flip_on_load_set
#else
static STBI_THREAD_LOCAL int stbi__vertically_flip_on_load_local, stbi__vertically_flip_on_load_set;

// 设置是否在加载时垂直翻转图像的标志
STBIDEF void stbi_set_flip_vertically_on_load_thread(int flag_true_if_should_flip) {
    stbi__vertically_flip_on_load_local = flag_true_if_should_flip;
    stbi__vertically_flip_on_load_set = 1;
}

// 定义宏 stbi__vertically_flip_on_load，根据设置的标志选择使用本地变量还是全局变量
#define stbi__vertically_flip_on_load                                                                                          \
    (stbi__vertically_flip_on_load_set ? stbi__vertically_flip_on_load_local : stbi__vertically_flip_on_load_global)
#endif // STBI_THREAD_LOCAL

// 加载图像的主要函数，返回图像数据指针
static void * stbi__load_main(stbi__context * s, int * x, int * y, int * comp, int req_comp, stbi__result_info * ri, int bpc) {
    memset(ri, 0, sizeof(*ri));         // 确保初始化，以防添加新字段
    ri->bits_per_channel = 8;           // 默认为 8，大多数情况下不需要更改
    ri->channel_order = STBI_ORDER_RGB; // 当前输入和输出通道顺序为 RGB，但这里可以添加 BGR 顺序
// 设置图像的通道数为0
ri->num_channels = 0;

// 使用非常明确的头部信息（至少是一个FOURCC或者独特的魔术数字）来测试格式
#ifndef STBI_NO_PNG
    // 如果是PNG格式，则调用stbi__png_load函数加载图像
    if (stbi__png_test(s))
        return stbi__png_load(s, x, y, comp, req_comp, ri);
#endif
#ifndef STBI_NO_BMP
    // 如果是BMP格式，则调用stbi__bmp_load函数加载图像
    if (stbi__bmp_test(s))
        return stbi__bmp_load(s, x, y, comp, req_comp, ri);
#endif
#ifndef STBI_NO_GIF
    // 如果是GIF格式，则调用stbi__gif_load函数加载图像
    if (stbi__gif_test(s))
        return stbi__gif_load(s, x, y, comp, req_comp, ri);
#endif
#ifndef STBI_NO_PSD
    // 如果是PSD格式，则调用stbi__psd_load函数加载图像
    if (stbi__psd_test(s))
        return stbi__psd_load(s, x, y, comp, req_comp, ri, bpc);
#else
// 如果 bpc 没有被使用，则忽略
    STBI_NOTUSED(bpc);
#endif
#ifndef STBI_NO_PIC
    // 如果文件是 PIC 格式，则调用 stbi__pic_load 函数加载文件
    if (stbi__pic_test(s))
        return stbi__pic_load(s, x, y, comp, req_comp, ri);
#endif

// 然后尝试加载只有 1 或 2 个字节匹配预期的格式；这些容易产生误判，所以稍后再尝试
#ifndef STBI_NO_JPEG
    // 如果文件是 JPEG 格式，则调用 stbi__jpeg_load 函数加载文件
    if (stbi__jpeg_test(s))
        return stbi__jpeg_load(s, x, y, comp, req_comp, ri);
#endif
#ifndef STBI_NO_PNM
    // 如果文件是 PNM 格式，则调用 stbi__pnm_load 函数加载文件
    if (stbi__pnm_test(s))
        return stbi__pnm_load(s, x, y, comp, req_comp, ri);
#endif

#ifndef STBI_NO_HDR
```
这段代码是一个条件判断的代码块，根据不同的条件调用不同的函数来加载不同格式的文件。每个条件判断都是根据文件的格式来确定是否调用对应的加载函数。
    // 如果是 HDR 格式的图片，则调用 stbi__hdr_load 函数加载图片数据，并转换为 LDR 格式返回
    if (stbi__hdr_test(s)) {
        float * hdr = stbi__hdr_load(s, x, y, comp, req_comp, ri);
        return stbi__hdr_to_ldr(hdr, *x, *y, req_comp ? req_comp : *comp);
    }
#endif

#ifndef STBI_NO_TGA
    // 如果是 TGA 格式的图片，则调用 stbi__tga_load 函数加载图片数据并返回
    // 注意：TGA 格式的测试放在最后，因为测试方法不够准确
    if (stbi__tga_test(s))
        return stbi__tga_load(s, x, y, comp, req_comp, ri);
#endif

    // 如果不是以上任何已知格式的图片，则返回错误信息
    return stbi__errpuc("unknown image type", "Image not of any known type, or corrupt");
}

// 将 16 位图片数据转换为 8 位图片数据
static stbi_uc * stbi__convert_16_to_8(stbi__uint16 * orig, int w, int h, int channels) {
    int i;
    int img_len = w * h * channels;
    stbi_uc * reduced;
    // 分配内存，用于存储转换后的数据
    reduced = (stbi_uc *)stbi__malloc(img_len);
    // 如果内存分配失败，返回错误信息
    if (reduced == NULL)
        return stbi__errpuc("outofmem", "Out of memory");

    // 将原始数据转换为 8 位到 16 位的数据
    for (i = 0; i < img_len; ++i)
        reduced[i] = (stbi_uc)((orig[i] >> 8) & 0xFF); // 取每个字节的高 8 位作为 16 位数据

    // 释放原始数据的内存
    STBI_FREE(orig);
    // 返回转换后的数据
    return reduced;
}

// 将 8 位数据转换为 16 位数据
static stbi__uint16 * stbi__convert_8_to_16(stbi_uc * orig, int w, int h, int channels) {
    int i;
    int img_len = w * h * channels;
    stbi__uint16 * enlarged;

    // 分配足够的内存，用于存储转换后的数据
    enlarged = (stbi__uint16 *)stbi__malloc(img_len * 2);
    // 如果内存分配失败，返回错误信息
    if (enlarged == NULL)
        return (stbi__uint16 *)stbi__errpuc("outofmem", "Out of memory");
    // 遍历原始图像数据，将每个像素值扩大为16位，高位和低位相同，范围从0~255映射到0~0xffff
    for (i = 0; i < img_len; ++i)
        enlarged[i] = (stbi__uint16)((orig[i] << 8) + orig[i]); // replicate to high and low byte, maps 0->0, 255->0xffff

    // 释放原始图像数据的内存
    STBI_FREE(orig);
    // 返回扩大后的图像数据
    return enlarged;
}

// 垂直翻转图像
static void stbi__vertical_flip(void * image, int w, int h, int bytes_per_pixel) {
    int row;
    // 每行的字节数
    size_t bytes_per_row = (size_t)w * bytes_per_pixel;
    // 临时存储空间
    stbi_uc temp[2048];
    // 图像数据的指针
    stbi_uc * bytes = (stbi_uc *)image;

    // 遍历图像的一半高度
    for (row = 0; row < (h >> 1); row++) {
        // 指向当前行和对称行的指针
        stbi_uc * row0 = bytes + row * bytes_per_row;
        stbi_uc * row1 = bytes + (h - row - 1) * bytes_per_row;
        // 交换当前行和对称行的像素数据
        size_t bytes_left = bytes_per_row;
        while (bytes_left) {
            // 拷贝当前行和对称行的像素数据
            size_t bytes_copy = (bytes_left < sizeof(temp)) ? bytes_left : sizeof(temp);
// 将源地址的数据复制到目标地址，temp为临时存储地址，bytes_copy为要复制的字节数
memcpy(temp, row0, bytes_copy);
// 将row1的数据复制到row0
memcpy(row0, row1, bytes_copy);
// 将temp的数据复制到row1
memcpy(row1, temp, bytes_copy);
// 更新row0和row1的地址指针，使其指向下一段数据
row0 += bytes_copy;
row1 += bytes_copy;
// 更新剩余字节数
bytes_left -= bytes_copy;
// 垂直翻转图像的每个片段
static void stbi__vertical_flip_slices(void * image, int w, int h, int z, int bytes_per_pixel) {
    // 计算每个片段的大小
    int slice_size = w * h * bytes_per_pixel;
    // 将图像数据转换为无符号字符指针
    stbi_uc * bytes = (stbi_uc *)image;
    // 遍历每个片段
    for (slice = 0; slice < z; ++slice) {
        // 对每个片段进行垂直翻转
        stbi__vertical_flip(bytes, w, h, bytes_per_pixel);
        // 更新指针，指向下一个片段的起始位置
        bytes += slice_size;
    }
// 结束条件判断，检查是否到达文件末尾
}
#endif

// 加载并进行后处理，将图像数据转换为8位
static unsigned char * stbi__load_and_postprocess_8bit(stbi__context * s, int * x, int * y, int * comp, int req_comp) {
    // 定义结果信息结构体
    stbi__result_info ri;
    // 调用stbi__load_main函数加载图像数据，并进行后处理
    void * result = stbi__load_main(s, x, y, comp, req_comp, &ri, 8);

    // 如果加载失败，返回空指针
    if (result == NULL)
        return NULL;

    // 加载器负责确保我们得到8位或16位的数据
    STBI_ASSERT(ri.bits_per_channel == 8 || ri.bits_per_channel == 16);

    // 如果数据不是8位，将16位数据转换为8位
    if (ri.bits_per_channel != 8) {
        result = stbi__convert_16_to_8((stbi__uint16 *)result, *x, *y, req_comp == 0 ? *comp : req_comp);
        ri.bits_per_channel = 8;
    }

    // @TODO: 将stbi__convert_format移动到这里
    // 如果需要在加载时垂直翻转图像
    if (stbi__vertically_flip_on_load) {
        // 如果需要指定通道数，则使用指定的通道数，否则使用默认通道数
        int channels = req_comp ? req_comp : *comp;
        // 对加载的图像进行垂直翻转
        stbi__vertical_flip(result, *x, *y, channels * sizeof(stbi_uc));
    }

    // 返回加载的图像数据
    return (unsigned char *)result;
}

// 加载并处理16位图像数据
static stbi__uint16 * stbi__load_and_postprocess_16bit(stbi__context * s, int * x, int * y, int * comp, int req_comp) {
    // 加载主函数，获取加载的图像数据
    stbi__result_info ri;
    void * result = stbi__load_main(s, x, y, comp, req_comp, &ri, 16);

    // 如果加载的图像数据为空，则返回空
    if (result == NULL)
        return NULL;

    // 加载器负责确保我们得到8位或16位的图像数据
    STBI_ASSERT(ri.bits_per_channel == 8 || ri.bits_per_channel == 16);

    // 如果图像数据不是16位的，则将8位图像数据转换为16位
    if (ri.bits_per_channel != 16) {
        result = stbi__convert_8_to_16((stbi_uc *)result, *x, *y, req_comp == 0 ? *comp : req_comp);
    // 设置每个通道的位数为16
    ri.bits_per_channel = 16;
    }

    // @TODO: 将stbi__convert_format16移到这里
    // @TODO: 对于8位到16位的情况，特殊处理RGB到Y（以及RGBA到YA），以保持更多的精度

    // 如果在加载时需要垂直翻转图像
    if (stbi__vertically_flip_on_load) {
        // 获取通道数
        int channels = req_comp ? req_comp : *comp;
        // 对结果进行垂直翻转
        stbi__vertical_flip(result, *x, *y, channels * sizeof(stbi__uint16));
    }

    // 返回16位无符号整数指针类型的结果
    return (stbi__uint16 *)result;
}

// 如果未定义STBI_NO_HDR和STBI_NO_LINEAR
static void stbi__float_postprocess(float * result, int * x, int * y, int * comp, int req_comp) {
    // 如果在加载时需要垂直翻转图像，并且结果不为空
    if (stbi__vertically_flip_on_load && result != NULL) {
        // 获取通道数
        int channels = req_comp ? req_comp : *comp;
        // 对结果进行垂直翻转
        stbi__vertical_flip(result, *x, *y, channels * sizeof(float));
    }
} 
#endif
// 如果没有定义 STBI_NO_STDIO，则执行以下代码

#ifndef STBI_NO_STDIO
// 如果没有定义 STBI_NO_STDIO，则执行以下代码

#if defined(_WIN32) && defined(STBI_WINDOWS_UTF8)
// 如果在 Windows 平台且定义了 STBI_WINDOWS_UTF8，则执行以下代码
STBI_EXTERN __declspec(dllimport) int __stdcall MultiByteToWideChar(unsigned int cp, unsigned long flags, const char * str,
                                                                    int cbmb, wchar_t * widestr, int cchwide);
// 导入 MultiByteToWideChar 函数声明

STBI_EXTERN __declspec(dllimport) int __stdcall WideCharToMultiByte(unsigned int cp, unsigned long flags,
                                                                    const wchar_t * widestr, int cchwide, char * str, int cbmb,
                                                                    const char * defchar, int * used_default);
// 导入 WideCharToMultiByte 函数声明
#endif
// 结束条件编译指令

#if defined(_WIN32) && defined(STBI_WINDOWS_UTF8)
// 如果在 Windows 平台且定义了 STBI_WINDOWS_UTF8，则执行以下代码
STBIDEF int stbi_convert_wchar_to_utf8(char * buffer, size_t bufferlen, const wchar_t * input) {
    // 将宽字符转换为 UTF-8 编码
    return WideCharToMultiByte(65001 /* UTF8 */, 0, input, -1, buffer, (int)bufferlen, NULL, NULL);
}
#endif
// 结束条件编译指令

static FILE * stbi__fopen(char const * filename, char const * mode) {
// 定义一个静态函数 stbi__fopen，用于打开文件
    // 声明文件指针
    FILE * f;
    // 如果是在 Windows 平台且定义了 STBI_WINDOWS_UTF8
#if defined(_WIN32) && defined(STBI_WINDOWS_UTF8)
    // 声明宽字符数组
    wchar_t wMode[64];
    wchar_t wFilename[1024];
    // 将 UTF-8 编码的文件名转换为宽字符编码
    if (0 == MultiByteToWideChar(65001 /* UTF8 */, 0, filename, -1, wFilename, sizeof(wFilename) / sizeof(*wFilename)))
        return 0;
    // 将 UTF-8 编码的模式转换为宽字符编码
    if (0 == MultiByteToWideChar(65001 /* UTF8 */, 0, mode, -1, wMode, sizeof(wMode) / sizeof(*wMode)))
        return 0;
    // 如果是在 Visual Studio 2005 及以上版本
#if defined(_MSC_VER) && _MSC_VER >= 1400
    // 使用 _wfopen_s 打开文件
    if (0 != _wfopen_s(&f, wFilename, wMode))
        f = 0;
    // 否则
#else
    // 使用 _wfopen 打开文件
    f = _wfopen(wFilename, wMode);
#endif

// 如果是在 Windows 平台但未定义 STBI_WINDOWS_UTF8，或者在 Visual Studio 2005 及以上版本
#elif defined(_MSC_VER) && _MSC_VER >= 1400
    // 使用 fopen_s 打开文件
    if (0 != fopen_s(&f, filename, mode))
        f = 0;
// 如果不是在 Windows 平台，使用 fopen 打开文件
#else
    f = fopen(filename, mode);
#endif
    return f;
}

// 从文件加载图像数据
STBIDEF stbi_uc * stbi_load(char const * filename, int * x, int * y, int * comp, int req_comp) {
    // 打开文件
    FILE * f = stbi__fopen(filename, "rb");
    unsigned char * result;
    // 如果文件打开失败，返回错误信息
    if (!f)
        return stbi__errpuc("can't fopen", "Unable to open file");
    // 从文件中加载图像数据
    result = stbi_load_from_file(f, x, y, comp, req_comp);
    // 关闭文件
    fclose(f);
    // 返回加载的图像数据
    return result;
}

// 从文件中加载图像数据
STBIDEF stbi_uc * stbi_load_from_file(FILE * f, int * x, int * y, int * comp, int req_comp) {
    unsigned char * result;
    stbi__context s;
    // 初始化文件上下文
    stbi__start_file(&s, f);
// 从文件中加载并处理8位图像数据
result = stbi__load_and_postprocess_8bit(&s, x, y, comp, req_comp);
if (result) {
    // 需要将IO缓冲区中的所有字符“unget”
    fseek(f, -(int)(s.img_buffer_end - s.img_buffer), SEEK_CUR);
}
return result;
}

// 从文件中加载并处理16位图像数据
STBIDEF stbi__uint16 * stbi_load_from_file_16(FILE * f, int * x, int * y, int * comp, int req_comp) {
    stbi__uint16 * result;
    stbi__context s;
    stbi__start_file(&s, f);
    result = stbi__load_and_postprocess_16bit(&s, x, y, comp, req_comp);
    if (result) {
        // 需要将IO缓冲区中的所有字符“unget”
        fseek(f, -(int)(s.img_buffer_end - s.img_buffer), SEEK_CUR);
    }
    return result;
}
// 从文件中加载16位图像数据，并返回一个指向该数据的指针
STBIDEF stbi_us * stbi_load_16(char const * filename, int * x, int * y, int * comp, int req_comp) {
    // 打开文件
    FILE * f = stbi__fopen(filename, "rb");
    stbi__uint16 * result;
    // 如果文件打开失败，返回错误信息
    if (!f)
        return (stbi_us *)stbi__errpuc("can't fopen", "Unable to open file");
    // 从文件中加载16位图像数据
    result = stbi_load_from_file_16(f, x, y, comp, req_comp);
    // 关闭文件
    fclose(f);
    // 返回加载的图像数据
    return result;
}

// 从内存中加载16位图像数据，并返回一个指向该数据的指针
STBIDEF stbi_us * stbi_load_16_from_memory(stbi_uc const * buffer, int len, int * x, int * y, int * channels_in_file,
                                           int desired_channels) {
    // 创建一个内存上下文
    stbi__context s;
    // 初始化内存上下文
    stbi__start_mem(&s, buffer, len);
    // 加载并处理16位图像数据
    return stbi__load_and_postprocess_16bit(&s, x, y, channels_in_file, desired_channels);
}

// 从回调函数中加载16位图像数据，并返回一个指向该数据的指针
STBIDEF stbi_us * stbi_load_16_from_callbacks(stbi_io_callbacks const * clbk, void * user, int * x, int * y,
// 从内存中加载图片数据，并进行后处理，返回8位数据
STBIDEF stbi_uc * stbi_load_from_memory(stbi_uc const * buffer, int len, int * x, int * y, int * comp, int req_comp) {
    // 创建 stbi__context 结构体
    stbi__context s;
    // 初始化 stbi__context 结构体，使用内存中的数据
    stbi__start_mem(&s, buffer, len);
    // 调用 stbi__load_and_postprocess_8bit 函数，加载并进行后处理，返回8位数据
    return stbi__load_and_postprocess_8bit(&s, x, y, comp, req_comp);
}

// 从回调函数中加载图片数据，并进行后处理，返回8位数据
STBIDEF stbi_uc * stbi_load_from_callbacks(stbi_io_callbacks const * clbk, void * user, int * x, int * y, int * comp,
                                           int req_comp) {
    // 创建 stbi__context 结构体
    stbi__context s;
    // 初始化 stbi__context 结构体，使用回调函数中的数据
    stbi__start_callbacks(&s, (stbi_io_callbacks *)clbk, user);
    // 调用 stbi__load_and_postprocess_8bit 函数，加载并进行后处理，返回8位数据
    return stbi__load_and_postprocess_8bit(&s, x, y, comp, req_comp);
}

// 从回调函数中加载图片数据，并进行后处理，返回16位数据
STBIDEF stbi_us * stbi_load_16_from_callbacks(stbi_io_callbacks const * clbk, void * user, int * x, int * y,
                                              int * channels_in_file, int desired_channels) {
    // 创建 stbi__context 结构体
    stbi__context s;
    // 初始化 stbi__context 结构体，使用回调函数中的数据
    stbi__start_callbacks(&s, (stbi_io_callbacks *)clbk, user);
    // 调用 stbi__load_and_postprocess_16bit 函数，加载并进行后处理，返回16位数据
    return stbi__load_and_postprocess_16bit(&s, x, y, channels_in_file, desired_channels);
}

// 如果没有定义 STBI_NO_GIF，则执行以下代码
#ifndef STBI_NO_GIF
# 从内存中加载 GIF 图像数据，并返回解码后的像素数据
# 参数：buffer - 内存缓冲区指针，len - 缓冲区长度，delays - 帧延迟数组指针，x/y/z - 图像尺寸，comp - 图像通道数，req_comp - 请求的通道数
STBIDEF stbi_uc * stbi_load_gif_from_memory(stbi_uc const * buffer, int len, int ** delays, int * x, int * y, int * z,
                                            int * comp, int req_comp) {
    unsigned char * result;  # 存储解码后的像素数据
    stbi__context s;  # 创建解码上下文
    stbi__start_mem(&s, buffer, len);  # 初始化解码上下文，指定内存缓冲区和长度

    result = (unsigned char *)stbi__load_gif_main(&s, delays, x, y, z, comp, req_comp);  # 调用解码函数，返回解码后的像素数据
    if (stbi__vertically_flip_on_load) {  # 如果需要在加载时垂直翻转图像
        stbi__vertical_flip_slices(result, *x, *y, *z, *comp);  # 调用垂直翻转函数
    }

    return result;  # 返回解码后的像素数据
}
#endif

#ifndef STBI_NO_LINEAR
static float * stbi__loadf_main(stbi__context * s, int * x, int * y, int * comp, int req_comp) {
    unsigned char * data;  # 存储解码后的像素数据
#ifndef STBI_NO_HDR
    if (stbi__hdr_test(s)) {  # 如果是 HDR 格式的图像
// 定义一个 stbi__result_info 结构体变量 ri
stbi__result_info ri;
// 调用 stbi__hdr_load 函数，加载 HDR 数据，并将结果保存在 hdr_data 中
float * hdr_data = stbi__hdr_load(s, x, y, comp, req_comp, &ri);
// 如果 hdr_data 不为空，则对其进行后处理
if (hdr_data)
    stbi__float_postprocess(hdr_data, x, y, comp, req_comp);
// 返回处理后的 HDR 数据
return hdr_data;
// 如果定义了 endif，则执行以下代码
#endif
// 调用 stbi__load_and_postprocess_8bit 函数，加载并对 8 位数据进行后处理
data = stbi__load_and_postprocess_8bit(s, x, y, comp, req_comp);
// 如果 data 不为空，则将其转换为 HDR 数据
if (data)
    return stbi__ldr_to_hdr(data, *x, *y, req_comp ? req_comp : *comp);
// 如果以上条件都不满足，则返回错误信息
return stbi__errpf("unknown image type", "Image not of any known type, or corrupt");

// 定义 stbi_loadf_from_memory 函数，从内存中加载 HDR 数据
STBIDEF float * stbi_loadf_from_memory(stbi_uc const * buffer, int len, int * x, int * y, int * comp, int req_comp) {
    // 定义 stbi__context 结构体变量 s
    stbi__context s;
    // 初始化 s，将 buffer 中的数据加载到 s 中
    stbi__start_mem(&s, buffer, len);
    // 调用 stbi__loadf_main 函数，加载并处理 HDR 数据
    return stbi__loadf_main(&s, x, y, comp, req_comp);
}

// 定义 stbi_loadf_from_callbacks 函数，从回调函数中加载 HDR 数据
STBIDEF float * stbi_loadf_from_callbacks(stbi_io_callbacks const * clbk, void * user, int * x, int * y, int * comp,
// 定义一个函数，从内存中加载图像数据
float * stbi_loadf_from_memory(stbi_uc const * buffer, int len, int * x, int * y, int * comp, int req_comp) {
    // 创建一个 stbi__context 结构体
    stbi__context s;
    // 初始化回调函数，将其与 stbi__context 结构体关联
    stbi__start_callbacks(&s, (stbi_io_callbacks *)clbk, user);
    // 调用 stbi__loadf_main 函数，从内存中加载图像数据
    return stbi__loadf_main(&s, x, y, comp, req_comp);
}

// 如果没有定义 STBI_NO_STDIO 宏，则定义一个从文件中加载图像数据的函数
STBIDEF float * stbi_loadf(char const * filename, int * x, int * y, int * comp, int req_comp) {
    float * result;
    // 打开文件
    FILE * f = stbi__fopen(filename, "rb");
    // 如果文件打开失败，则返回错误信息
    if (!f)
        return stbi__errpf("can't fopen", "Unable to open file");
    // 调用 stbi_loadf_from_file 函数，从文件中加载图像数据
    result = stbi_loadf_from_file(f, x, y, comp, req_comp);
    // 关闭文件
    fclose(f);
    // 返回加载的图像数据
    return result;
}

// 从文件中加载图像数据的函数
STBIDEF float * stbi_loadf_from_file(FILE * f, int * x, int * y, int * comp, int req_comp) {
    // 创建一个 stbi__context 结构体
    stbi__context s;
    // 初始化 stbi__context 结构体，将其与文件关联
    stbi__start_file(&s, f);
// 如果未定义 STBI_NO_STDIO，则返回 stbi__loadf_main 函数的结果
return stbi__loadf_main(&s, x, y, comp, req_comp);
#endif // !STBI_NO_STDIO

#endif // !STBI_NO_LINEAR

// 这些 is-hdr-or-not 是独立定义的，与 STBI_NO_LINEAR 是否定义无关，为了 API 的简单性；如果定义了 STBI_NO_LINEAR，则始终返回 false！

// 如果未定义 STBI_NO_HDR，则从内存中判断是否为 HDR 图像
STBIDEF int stbi_is_hdr_from_memory(stbi_uc const * buffer, int len) {
#ifndef STBI_NO_HDR
    // 创建 stbi__context 对象
    stbi__context s;
    // 从内存中开始读取数据
    stbi__start_mem(&s, buffer, len);
    // 判断是否为 HDR 图像
    return stbi__hdr_test(&s);
#else
    // 如果未定义 STBI_NO_HDR，则忽略 buffer 和 len 参数，直接返回 0
    STBI_NOTUSED(buffer);
    STBI_NOTUSED(len);
    return 0;
#endif
} 
// 结束了一个代码块

#ifndef STBI_NO_STDIO
// 如果没有定义 STBI_NO_STDIO，则定义 stbi_is_hdr 函数
STBIDEF int stbi_is_hdr(char const * filename) {
    // 打开文件以供读取
    FILE * f = stbi__fopen(filename, "rb");
    int result = 0;
    // 如果文件成功打开
    if (f) {
        // 调用 stbi_is_hdr_from_file 函数检查是否为 HDR 格式
        result = stbi_is_hdr_from_file(f);
        // 关闭文件
        fclose(f);
    }
    // 返回结果
    return result;
}

// 从文件中检查是否为 HDR 格式
STBIDEF int stbi_is_hdr_from_file(FILE * f) {
#ifndef STBI_NO_HDR
    // 获取当前文件位置
    long pos = ftell(f);
    int res;
    stbi__context s;
    // 初始化文件上下文
    stbi__start_file(&s, f);
    // 检查是否为 HDR 格式
    res = stbi__hdr_test(&s);
// 将文件指针移动到指定位置
fseek(f, pos, SEEK_SET);
// 返回结果
return res;
#else
// 如果没有定义STBI_NO_STDIO，则忽略文件指针f
STBI_NOTUSED(f);
// 返回0
return 0;
#endif
}
#endif // !STBI_NO_STDIO

// 通过回调函数判断是否为HDR格式图片
STBIDEF int stbi_is_hdr_from_callbacks(stbi_io_callbacks const * clbk, void * user) {
#ifndef STBI_NO_HDR
    // 创建stbi__context对象s
    stbi__context s;
    // 通过回调函数初始化stbi__context对象s
    stbi__start_callbacks(&s, (stbi_io_callbacks *)clbk, user);
    // 判断是否为HDR格式图片
    return stbi__hdr_test(&s);
#else
    // 如果没有定义STBI_NO_HDR，则忽略回调函数clbk和user
    STBI_NOTUSED(clbk);
    STBI_NOTUSED(user);
    // 返回0
    return 0;
#endif
}
// 如果未定义 STBI_NO_LINEAR，则定义两个全局变量 stbi__l2h_gamma 和 stbi__l2h_scale
#ifndef STBI_NO_LINEAR
static float stbi__l2h_gamma = 2.2f, stbi__l2h_scale = 1.0f;

// 将输入的 gamma 值赋给 stbi__l2h_gamma
STBIDEF void stbi_ldr_to_hdr_gamma(float gamma) { stbi__l2h_gamma = gamma; }
// 将输入的 scale 值赋给 stbi__l2h_scale
STBIDEF void stbi_ldr_to_hdr_scale(float scale) { stbi__l2h_scale = scale; }
#endif

// 定义两个全局变量 stbi__h2l_gamma_i 和 stbi__h2l_scale_i
static float stbi__h2l_gamma_i = 1.0f / 2.2f, stbi__h2l_scale_i = 1.0f;

// 将输入的 gamma 值的倒数赋给 stbi__h2l_gamma_i
STBIDEF void stbi_hdr_to_ldr_gamma(float gamma) { stbi__h2l_gamma_i = 1 / gamma; }
// 将输入的 scale 值的倒数赋给 stbi__h2l_scale_i
STBIDEF void stbi_hdr_to_ldr_scale(float scale) { stbi__h2l_scale_i = 1 / scale; }

//////////////////////////////////////////////////////////////////////////////
//
// 所有图像加载器都使用的通用代码
//

// 定义一个枚举类型，包含三个值：STBI__SCAN_load、STBI__SCAN_type、STBI__SCAN_header
enum { STBI__SCAN_load = 0, STBI__SCAN_type, STBI__SCAN_header };
// 重新填充缓冲区的函数
static void stbi__refill_buffer(stbi__context * s) {
    // 从输入流中读取数据到缓冲区
    int n = (s->io.read)(s->io_user_data, (char *)s->buffer_start, s->buflen);
    // 更新已经读取的字节数
    s->callback_already_read += (int)(s->img_buffer - s->img_buffer_original);
    // 如果读取的字节数为0，表示已经到达文件末尾
    if (n == 0) {
        // 处理已经到达文件末尾的情况
        s->read_from_callbacks = 0;
        s->img_buffer = s->buffer_start;
        s->img_buffer_end = s->buffer_start + 1;
        *s->img_buffer = 0;
    } else {
        // 更新图像缓冲区的起始和结束位置
        s->img_buffer = s->buffer_start;
        s->img_buffer_end = s->buffer_start + n;
    }
}

// 从缓冲区中获取一个字节的数据
stbi_inline static stbi_uc stbi__get8(stbi__context * s) {
    // 如果图像缓冲区中还有数据，直接返回一个字节的数据
    if (s->img_buffer < s->img_buffer_end)
        return *s->img_buffer++;
    // 如果需要从回调函数中读取数据
    if (s->read_from_callbacks) {
// 重新填充缓冲区
stbi__refill_buffer(s);
// 返回当前图像缓冲区的值，并将指针移动到下一个位置
return *s->img_buffer++;
// 如果已经到达文件末尾，返回0
if (s->img_buffer >= s->img_buffer_end)
    return 0;
// 如果定义了STBI_NO_JPEG、STBI_NO_HDR、STBI_NO_PIC、STBI_NO_PNM中的任何一个，不执行以下代码
// 否则，检查是否已经到达文件末尾
stbi_inline static int stbi__at_eof(stbi__context * s) {
    if (s->io.read) {
        // 如果未到达文件末尾，返回0
        if (!(s->io.eof)(s->io_user_data))
            return 0;
        // 如果feof()为真，则检查缓冲区是否等于末尾
        // 特殊情况：我们只有在末尾有特殊的0字符
        if (s->read_from_callbacks == 0)
            return 1;
    }
    // 返回缓冲区是否已经到达末尾
    return s->img_buffer >= s->img_buffer_end;
// 如果定义了STBI_NO_JPEG、STBI_NO_PNG、STBI_NO_BMP、STBI_NO_PSD、STBI_NO_TGA、STBI_NO_GIF、STBI_NO_PIC中的任何一个，就什么都不做
// 否则，定义一个跳过指定字节数的函数
static void stbi__skip(stbi__context * s, int n) {
    // 如果要跳过的字节数为0，直接返回
    if (n == 0)
        return; // already there!
    // 如果要跳过的字节数小于0，将图片缓冲区指针指向缓冲区末尾
    if (n < 0) {
        s->img_buffer = s->img_buffer_end;
        return;
    }
    // 如果有IO读取函数
    if (s->io.read) {
        // 计算当前图片缓冲区中剩余的字节数
        int blen = (int)(s->img_buffer_end - s->img_buffer);
        // 如果剩余字节数小于要跳过的字节数，将图片缓冲区指针指向缓冲区末尾，然后调用跳过函数跳过剩余的字节数
        if (blen < n) {
            s->img_buffer = s->img_buffer_end;
            (s->io.skip)(s->io_user_data, n - blen);
            return;
// 如果定义了STBI_NO_PNG、STBI_NO_TGA、STBI_NO_HDR和STBI_NO_PNM，则什么都不做
// 否则，从输入流中读取n个字节到buffer中
static int stbi__getn(stbi__context * s, stbi_uc * buffer, int n) {
    // 如果输入流的读取函数存在
    if (s->io.read) {
        // 计算当前图像缓冲区中剩余的字节数
        int blen = (int)(s->img_buffer_end - s->img_buffer);
        // 如果剩余字节数小于n
        if (blen < n) {
            int res, count;

            // 将当前图像缓冲区中的数据拷贝到buffer中
            memcpy(buffer, s->img_buffer, blen);

            // 从输入流中读取n-blen个字节到buffer中
            count = (s->io.read)(s->io_user_data, (char *)buffer + blen, n - blen);
            // 判断是否成功读取了n-blen个字节
            res = (count == (n - blen));
            // 将图像缓冲区指针指向缓冲区末尾
            s->img_buffer = s->img_buffer_end;
// 如果条件成立，返回 res
return res;
}

// 如果条件不成立，执行以下代码
if (s->img_buffer + n <= s->img_buffer_end) {
    // 将 s->img_buffer 中的 n 个字节复制到 buffer 中
    memcpy(buffer, s->img_buffer, n);
    // 更新 s->img_buffer 的位置
    s->img_buffer += n;
    // 返回 1
    return 1;
} else
    // 如果条件不成立，返回 0
    return 0;
}
#endif

#if defined(STBI_NO_JPEG) && defined(STBI_NO_PNG) && defined(STBI_NO_PSD) && defined(STBI_NO_PIC)
// 什么都不做
#else
// 从输入流中读取两个字节，以大端模式返回
static int stbi__get16be(stbi__context * s) {
    // 读取一个字节
    int z = stbi__get8(s);
    // 将其左移 8 位，再加上下一个字节的值，返回结果
    return (z << 8) + stbi__get8(s);
}
// 如果定义了 STBI_NO_PNG、STBI_NO_PSD 和 STBI_NO_PIC，则什么都不做
// 否则，定义一个函数 stbi__get32be，用于从 stbi__context 中读取 32 位大端格式的数据
static stbi__uint32 stbi__get32be(stbi__context * s) {
    // 读取 16 位大端格式的数据
    stbi__uint32 z = stbi__get16be(s);
    // 将读取的数据左移 16 位，再加上下一个 16 位大端格式的数据
    return (z << 16) + stbi__get16be(s);
}

// 如果定义了 STBI_NO_BMP、STBI_NO_TGA 和 STBI_NO_GIF，则什么都不做
// 否则，定义一个函数 stbi__get16le，用于从 stbi__context 中读取 16 位小端格式的数据
static int stbi__get16le(stbi__context * s) {
    // 读取 8 位数据
    int z = stbi__get8(s);
    // 将读取的数据加上下一个 8 位数据左移 8 位
    return z + (stbi__get8(s) << 8);
}
#ifndef STBI_NO_BMP
// 如果未定义 STBI_NO_BMP，则执行以下代码

static stbi__uint32 stbi__get32le(stbi__context * s) {
    // 从输入流中读取 16 位的数据，然后将其左移 16 位，再加上下一个 16 位的数据，返回 32 位的数据
    stbi__uint32 z = stbi__get16le(s);
    z += (stbi__uint32)stbi__get16le(s) << 16;
    return z;
}
#endif

#define STBI__BYTECAST(x) ((stbi_uc)((x)&255)) // 将整数截断为字节，避免警告

#if defined(STBI_NO_JPEG) && defined(STBI_NO_PNG) && defined(STBI_NO_BMP) && defined(STBI_NO_PSD) && defined(STBI_NO_TGA) &&   \
    defined(STBI_NO_GIF) && defined(STBI_NO_PIC) && defined(STBI_NO_PNM)
// 如果定义了所有的 STBI_NO_XXX 宏，则不执行任何操作
// 否则，执行以下代码
#else
// 如果未定义所有的 STBI_NO_XXX 宏，则执行以下代码
//////////////////////////////////////////////////////////////////////////////
//
//  通用的从内置 img_n 转换为 req_comp 的转换器
//    各个类型尽可能自动执行此操作（例如，jpeg 内部处理所有情况，因为它需要进行颜色空间转换，而且它从不具有 alpha 通道，所以情况非常少）。png 可以自动执行
// 计算亮度值，根据 RGB 值计算亮度值并返回
static stbi_uc stbi__compute_y(int r, int g, int b) { return (stbi_uc)(((r * 77) + (g * 150) + (29 * b)) >> 8); }
// 如果没有定义 STBI_NO_PNG、STBI_NO_BMP、STBI_NO_PSD、STBI_NO_TGA、STBI_NO_GIF、STBI_NO_PIC、STBI_NO_PNM，则执行以下代码
static unsigned char * stbi__convert_format(unsigned char * data, int img_n, int req_comp, unsigned int x, unsigned int y) {
    int i, j;
    unsigned char * good;

    // 如果请求的通道数等于图像的通道数，则直接返回原始数据
    if (req_comp == img_n)
        return data;
    // 断言请求的通道数在 1 到 4 之间
    STBI_ASSERT(req_comp >= 1 && req_comp <= 4);
    // 分配足够大小的内存空间，用于存储转换后的图像数据
    good = (unsigned char *)stbi__malloc_mad3(req_comp, x, y, 0);
    // 如果内存分配失败，则释放之前分配的内存空间，并返回内存不足的错误信息
    if (good == NULL) {
        STBI_FREE(data);
        return stbi__errpuc("outofmem", "Out of memory");
    }

    // 遍历图像的每一行
    for (j = 0; j < (int)y; ++j) {
        // 指向源图像数据的指针
        unsigned char * src = data + j * x * img_n;
        // 指向目标图像数据的指针
        unsigned char * dest = good + j * x * req_comp;

        // 定义宏，用于组合图像通道数
        #define STBI__COMBO(a, b) ((a)*8 + (b))
        // 定义宏，用于处理不同图像通道数的情况
        #define STBI__CASE(a, b)                                                                                                       \
        case STBI__COMBO(a, b):                                                                                                    \
        for (i = x - 1; i >= 0; --i, src += a, dest += b)
        // 根据源图像通道数和目标图像通道数的组合，执行不同的转换操作
        // 避免每个像素点都进行一次判断，而是每行进行一次判断，使用大量的宏来实现
        switch (STBI__COMBO(img_n, req_comp)) {
            // 当源图像通道数为1，目标图像通道数为2时
            STBI__CASE(1, 2) {
                // 执行转换操作，将源图像的数据复制到目标图像，并设置第二个通道的值为255
                dest[0] = src[0];
                dest[1] = 255;
            }  # 结束当前的 case 分支
            break;  # 跳出 switch 语句
            STBI__CASE(1, 3) { dest[0] = dest[1] = dest[2] = src[0]; }  # 当输入格式为 1 通道，输出格式为 3 通道时，将目标数组的前三个元素赋值为源数组的第一个元素
            break;  # 跳出 switch 语句
            STBI__CASE(1, 4) {  # 当输入格式为 1 通道，输出格式为 4 通道时
                dest[0] = dest[1] = dest[2] = src[0];  # 将目标数组的前三个元素赋值为源数组的第一个元素
                dest[3] = 255;  # 将目标数组的第四个元素赋值为 255
            }
            break;  # 跳出 switch 语句
            STBI__CASE(2, 1) { dest[0] = src[0]; }  # 当输入格式为 2 通道，输出格式为 1 通道时，将目标数组的第一个元素赋值为源数组的第一个元素
            break;  # 跳出 switch 语句
            STBI__CASE(2, 3) { dest[0] = dest[1] = dest[2] = src[0]; }  # 当输入格式为 2 通道，输出格式为 3 通道时，将目标数组的前三个元素赋值为源数组的第一个元素
            break;  # 跳出 switch 语句
            STBI__CASE(2, 4) {  # 当输入格式为 2 通道，输出格式为 4 通道时
                dest[0] = dest[1] = dest[2] = src[0];  # 将目标数组的前三个元素赋值为源数组的第一个元素
                dest[3] = src[1];  # 将目标数组的第四个元素赋值为源数组的第二个元素
            }
            break;  # 跳出 switch 语句
            STBI__CASE(3, 4) { dest[0] = src[0];  # 当输入格式为 3 通道，输出格式为 4 通道时，将目标数组的第一个元素赋值为源数组的第一个元素
            // 将源数组中的第二个元素复制到目标数组的第二个位置
            dest[1] = src[1];
            // 将源数组中的第三个元素复制到目标数组的第三个位置
            dest[2] = src[2];
            // 将目标数组的第四个位置设置为255
            dest[3] = 255;
            // 结束当前的case
            }
            break;
            // 当源数组有3个元素时的情况
            STBI__CASE(3, 1) { dest[0] = stbi__compute_y(src[0], src[1], src[2]); }
            break;
            // 当源数组有3个元素时的情况
            STBI__CASE(3, 2) {
                // 计算亮度并赋值给目标数组的第一个位置
                dest[0] = stbi__compute_y(src[0], src[1], src[2]);
                // 将目标数组的第二个位置设置为255
                dest[1] = 255;
            }
            break;
            // 当源数组有4个元素时的情况
            STBI__CASE(4, 1) { dest[0] = stbi__compute_y(src[0], src[1], src[2]); }
            break;
            // 当源数组有4个元素时的情况
            STBI__CASE(4, 2) {
                // 计算亮度并赋值给目标数组的第一个位置
                dest[0] = stbi__compute_y(src[0], src[1], src[2]);
                // 将源数组的第四个元素赋值给目标数组的第二个位置
                dest[1] = src[3];
            }
            break;
            // 当源数组有4个元素时的情况
            STBI__CASE(4, 3) {
                dest[0] = src[0];
                dest[1] = src[1];
                dest[2] = src[2];
            }
            break;
        default:
            // 断言，如果不是以上情况，则抛出错误
            STBI_ASSERT(0);
            // 释放内存
            STBI_FREE(data);
            STBI_FREE(good);
            // 返回错误信息
            return stbi__errpuc("unsupported", "Unsupported format conversion");
        }
#undef STBI__CASE
    }

    // 释放内存
    STBI_FREE(data);
    // 返回结果
    return good;
}
#endif

#if defined(STBI_NO_PNG) && defined(STBI_NO_PSD)
```

在这段代码中，主要是对不同情况下的处理进行了注释解释。包括对数组的赋值、错误处理、内存释放和返回结果等。
// 如果没有定义 STBI_NO_PNG 和 STBI_NO_PSD，则执行以下代码
static stbi__uint16 * stbi__convert_format16(stbi__uint16 * data, int img_n, int req_comp, unsigned int x, unsigned int y) {
    // 声明变量
    int i, j;
    stbi__uint16 * good;

    // 如果请求的通道数和图像的通道数相同，则直接返回数据
    if (req_comp == img_n)
        return data;
    // 断言请求的通道数在1到4之间
    STBI_ASSERT(req_comp >= 1 && req_comp <= 4);

    // 分配新的内存空间来存储转换后的数据
    good = (stbi__uint16 *)stbi__malloc(req_comp * x * y * 2);
    // 如果内存分配失败，则释放原始数据并返回错误信息
    if (good == NULL) {
        STBI_FREE(data);
        return (stbi__uint16 *)stbi__errpuc("outofmem", "Out of memory");
    }

    for (j = 0; j < (int)y; ++j) {
        // 指向源数据的指针
        stbi__uint16 * src = data + j * x * img_n;
        // 指向目标数据的指针
        stbi__uint16 * dest = good + j * x * req_comp;

        // 定义一个宏，用于将两个参数组合成一个新的值
#define STBI__COMBO(a, b) ((a)*8 + (b))
        // 定义一个宏，用于处理不同组合的情况
#define STBI__CASE(a, b)                                                                                                       \
    case STBI__COMBO(a, b):                                                                                                    \
        for (i = x - 1; i >= 0; --i, src += a, dest += b)
        // 将源图像的 img_n 个分量转换为 req_comp 个分量；
        // 避免每个像素使用 switch，所以使用每行扫描和大量宏
        switch (STBI__COMBO(img_n, req_comp)) {
            // 当 img_n 为 1，req_comp 为 2 时
            STBI__CASE(1, 2) {
                // 将源数据的第一个分量赋值给目标数据的第一个分量，第二个分量赋值为 0xffff
                dest[0] = src[0];
                dest[1] = 0xffff;
            }
            break;
            // 当 img_n 为 1，req_comp 为 3 时
            STBI__CASE(1, 3) { 
                // 将源数据的第一个分量赋值给目标数据的三个分量
                dest[0] = dest[1] = dest[2] = src[0]; 
            }
            break;
# 根据不同的情况将源数据转换成目标数据
STBI__CASE(1, 4) {
    # 如果源数据是1个通道，目标数据是4个通道，则将目标数据的前3个通道赋值为源数据的第一个通道的值，第4个通道赋值为0xffff
    dest[0] = dest[1] = dest[2] = src[0];
    dest[3] = 0xffff;
}
break;
STBI__CASE(2, 1) { 
    # 如果源数据是2个通道，目标数据是1个通道，则将目标数据的第一个通道赋值为源数据的第一个通道的值
    dest[0] = src[0]; 
}
break;
STBI__CASE(2, 3) { 
    # 如果源数据是2个通道，目标数据是3个通道，则将目标数据的前3个通道赋值为源数据的第一个通道的值
    dest[0] = dest[1] = dest[2] = src[0]; 
}
break;
STBI__CASE(2, 4) {
    # 如果源数据是2个通道，目标数据是4个通道，则将目标数据的前3个通道赋值为源数据的第一个通道的值，第4个通道赋值为源数据的第二个通道的值
    dest[0] = dest[1] = dest[2] = src[0];
    dest[3] = src[1];
}
break;
STBI__CASE(3, 4) {
    # 如果源数据是3个通道，目标数据是4个通道，则将目标数据的前3个通道分别赋值为源数据的三个通道的值，第4个通道赋值为0xffff
    dest[0] = src[0];
    dest[1] = src[1];
    dest[2] = src[2];
    dest[3] = 0xffff;
}
            # 结束当前的 switch-case 结构
            break;
            # 如果输入通道数为3，输出通道数为1，调用 stbi__compute_y_16 函数计算亮度值
            STBI__CASE(3, 1) { dest[0] = stbi__compute_y_16(src[0], src[1], src[2]); }
            # 结束当前的 switch-case 结构
            break;
            # 如果输入通道数为3，输出通道数为2，调用 stbi__compute_y_16 函数计算亮度值，并设置第二个输出通道为最大值
            STBI__CASE(3, 2) {
                dest[0] = stbi__compute_y_16(src[0], src[1], src[2]);
                dest[1] = 0xffff;
            }
            # 结束当前的 switch-case 结构
            break;
            # 如果输入通道数为4，输出通道数为1，调用 stbi__compute_y_16 函数计算亮度值
            STBI__CASE(4, 1) { dest[0] = stbi__compute_y_16(src[0], src[1], src[2]); }
            # 结束当前的 switch-case 结构
            break;
            # 如果输入通道数为4，输出通道数为2，调用 stbi__compute_y_16 函数计算亮度值，并设置第二个输出通道为输入的第四个通道值
            STBI__CASE(4, 2) {
                dest[0] = stbi__compute_y_16(src[0], src[1], src[2]);
                dest[1] = src[3];
            }
            # 结束当前的 switch-case 结构
            break;
            # 如果输入通道数为4，输出通道数为3，直接将输入的前三个通道值赋给输出的前三个通道
            STBI__CASE(4, 3) {
                dest[0] = src[0];
                dest[1] = src[1];
                dest[2] = src[2];
            }
// 如果条件满足，则跳出当前循环
            break;
        // 如果条件不满足，则执行下面的语句
        default:
            // 断言，如果条件不满足，则终止程序
            STBI_ASSERT(0);
            // 释放内存
            STBI_FREE(data);
            // 释放内存
            STBI_FREE(good);
            // 返回错误信息
            return (stbi__uint16 *)stbi__errpuc("unsupported", "Unsupported format conversion");
        }
// 取消之前定义的宏
#undef STBI__CASE
    }

    // 释放内存
    STBI_FREE(data);
    // 返回结果
    return good;
}
#endif

#ifndef STBI_NO_LINEAR
// 将低动态范围图像数据转换为高动态范围图像数据
static float * stbi__ldr_to_hdr(stbi_uc * data, int x, int y, int comp) {
    int i, k, n;
    float * output;
    // 如果数据为空，则执行下面的语句
    if (!data)
        // 返回空指针
        return NULL;
    // 为输出分配内存空间
    output = (float *)stbi__malloc_mad4(x, y, comp, sizeof(float), 0);
    // 如果分配内存失败，则释放之前分配的内存并返回错误信息
    if (output == NULL) {
        STBI_FREE(data);
        return stbi__errpf("outofmem", "Out of memory");
    }
    // 计算非 alpha 通道的数量
    if (comp & 1)
        n = comp;
    else
        n = comp - 1;
    // 对每个像素的每个通道进行 gamma 校正和缩放，并存储到输出数组中
    for (i = 0; i < x * y; ++i) {
        for (k = 0; k < n; ++k) {
            output[i * comp + k] = (float)(pow(data[i * comp + k] / 255.0f, stbi__l2h_gamma) * stbi__l2h_scale);
        }
    }
    // 如果存在 alpha 通道，则将其存储到输出数组中
    if (n < comp) {
        for (i = 0; i < x * y; ++i) {
            output[i * comp + n] = data[i * comp + n] / 255.0f;
        }
    }
    // 释放 HDR 数据占用的内存
    STBI_FREE(data);
    // 返回 LDR 数据
    return output;
}
#endif

#ifndef STBI_NO_HDR
// 将 HDR 数据转换为 LDR 数据
#define stbi__float2int(x) ((int)(x))
static stbi_uc * stbi__hdr_to_ldr(float * data, int x, int y, int comp) {
    int i, k, n;
    stbi_uc * output;
    if (!data)
        return NULL;
    // 分配 LDR 数据所需的内存
    output = (stbi_uc *)stbi__malloc_mad3(x, y, comp, 0);
    if (output == NULL) {
        // 如果内存分配失败，释放 HDR 数据占用的内存，并返回错误信息
        STBI_FREE(data);
        return stbi__errpuc("outofmem", "Out of memory");
    }
    // 计算非 alpha 通道的数量
    if (comp & 1)
    # 如果 comp 大于 0，则将 n 设置为 comp；否则将 n 设置为 comp - 1
    if (comp > 0) {
        n = comp;
    } else {
        n = comp - 1;
    }
    # 遍历数据数组中的每个元素
    for (i = 0; i < x * y; ++i) {
        # 对每个元素的前 n 个分量进行处理
        for (k = 0; k < n; ++k) {
            # 对数据进行缩放和 gamma 校正，并将结果转换为整数
            float z = (float)pow(data[i * comp + k] * stbi__h2l_scale_i, stbi__h2l_gamma_i) * 255 + 0.5f;
            # 如果结果小于 0，则将其设置为 0
            if (z < 0)
                z = 0;
            # 如果结果大于 255，则将其设置为 255
            if (z > 255)
                z = 255;
            # 将处理后的结果转换为 stbi_uc 类型并存储到输出数组中
            output[i * comp + k] = (stbi_uc)stbi__float2int(z);
        }
        # 如果 k 小于 comp，则对第 k 个分量进行处理
        if (k < comp) {
            # 对数据进行缩放，并将结果转换为整数
            float z = data[i * comp + k] * 255 + 0.5f;
            # 如果结果小于 0，则将其设置为 0
            if (z < 0)
                z = 0;
            # 如果结果大于 255，则将其设置为 255
            if (z > 255)
                z = 255;
            # 将处理后的结果转换为 stbi_uc 类型并存储到输出数组中
            output[i * comp + k] = (stbi_uc)stbi__float2int(z);
        }
    }
    }
    // 释放由stbi_load加载的数据
    STBI_FREE(data);
    // 返回解码后的图像数据
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
//      - doesn't allow partial loading, loading multiple at once
//      - still fast on x86 (copying globals into locals doesn't help x86)
//      - allocates lots of intermediate memory (full size of all components)
//        - non-interleaved case requires this anyway
//        - allows good upsampling (see next)
//    high-quality
```

注释：
- 释放由stbi_load加载的数据
- 返回解码后的图像数据
- 这是一个基准的JPEG/JFIF解码器
- 简单的实现
  - 不支持延迟输出的y维度
  - 简单的接口（只有一个输出格式：8位交错RGB）
  - 不尝试恢复损坏的JPEG文件
  - 不允许部分加载，一次加载多个
  - 在x86上仍然很快（将全局变量复制到本地变量对x86没有帮助）
  - 分配大量的中间内存（所有组件的完整大小）
    - 非交错情况下无论如何都需要这样做
    - 允许良好的上采样（见下文）
- 高质量
// 如果没有定义 STBI_NO_JPEG，则执行以下代码

// 定义快速位数，用于加速哈夫曼解码，较大的值处理更多情况，较小的值减少缓存占用
#define FAST_BITS 9 

// 定义结构体，包含快速查找表、哈夫曼编码、值、大小、最大编码、增量等信息
typedef struct {
    stbi_uc fast[1 << FAST_BITS]; // 快速查找表
    stbi__uint16 code[256]; // 哈夫曼编码
    stbi_uc values[256]; // 值
    stbi_uc size[257]; // 大小
    unsigned int maxcode[18]; // 最大编码
    int delta[17]; // 增量
// 定义了一个结构体 stbi__huffman，用于存储哈夫曼编码相关的信息
} stbi__huffman;

// 定义了一个结构体，用于存储 JPEG 图像解码过程中需要用到的各种信息
typedef struct {
    stbi__context * s; // 指向 stbi__context 结构体的指针，用于存储 JPEG 图像的上下文信息
    stbi__huffman huff_dc[4]; // 存储 DC 分量的哈夫曼编码信息
    stbi__huffman huff_ac[4]; // 存储 AC 分量的哈夫曼编码信息
    stbi__uint16 dequant[4][64]; // 存储量化表信息
    stbi__int16 fast_ac[4][1 << FAST_BITS]; // 存储快速 AC 解码表信息

    // 图像组件的大小，交错的最大 MCU 数
    int img_h_max, img_v_max;
    int img_mcu_x, img_mcu_y;
    int img_mcu_w, img_mcu_h;

    // JPEG 图像组件的定义
    struct {
        int id; // 组件的 ID
        int h, v; // 水平和垂直采样因子
        int tq; // 量化表索引
        int hd, ha; // DC 和 AC 分量的哈夫曼表索引
        int dc_pred;  // 用于存储 DC 预测值

        int x, y, w2, h2;  // 图像的位置和尺寸参数
        stbi_uc * data;  // 指向图像数据的指针
        void *raw_data, *raw_coeff;  // 原始数据和系数数据的指针
        stbi_uc * linebuf;  // 用于存储扫描线数据的缓冲区
        short * coeff;        // 仅用于渐进式扫描
        int coeff_w, coeff_h; // 8x8 系数块的数量
    } img_comp[4];  // 图像组件数组，最多包含 4 个组件

    stbi__uint32 code_buffer; // JPEG 熵编码缓冲区
    int code_bits;            // 有效位数
    unsigned char marker;     // 在填充熵缓冲区时遇到的标记
    int nomore;               // 如果遇到标记，则必须停止的标志

    int progressive;  // 是否为渐进式扫描
    int spec_start;  // 起始扫描位置
    int spec_end;  // 结束扫描位置
    int succ_high;  // 高位 AC 系数的累积计数
    int succ_low;  // 低位 AC 系数的累积计数
    // 定义变量，用于存储解码过程中的一些参数
    int eob_run;
    int jfif;
    int app14_color_transform; // Adobe APP14 tag
    int rgb;

    int scan_n, order[4];
    int restart_interval, todo;

    // 定义指向函数的指针，用于存储不同的内核函数
    void (*idct_block_kernel)(stbi_uc * out, int out_stride, short data[64]);
    void (*YCbCr_to_RGB_kernel)(stbi_uc * out, const stbi_uc * y, const stbi_uc * pcb, const stbi_uc * pcr, int count,
                                int step);
    stbi_uc * (*resample_row_hv_2_kernel)(stbi_uc * out, stbi_uc * in_near, stbi_uc * in_far, int w, int hs);
} stbi__jpeg;

// 静态函数，用于构建哈夫曼编码表
static int stbi__build_huffman(stbi__huffman * h, int * count) {
    int i, j, k = 0;
    unsigned int code;
    // 根据 JPEG 规范，构建每个符号的大小列表
    for (i = 0; i < 16; ++i) {
        for (j = 0; j < count[i]; ++j) {
            // 将当前 i 对应的 count[i] 个大小为 i+1 的值写入 h->size 数组
            h->size[k++] = (stbi_uc)(i + 1);
            // 如果 k 大于等于 257，则说明 size 数组超出范围，返回错误信息
            if (k >= 257)
                return stbi__err("bad size list", "Corrupt JPEG");
        }
    }
    // 将 h->size 数组的第 k 个元素设为 0
    h->size[k] = 0;

    // 计算实际符号（来自 JPEG 规范）
    code = 0;
    k = 0;
    for (j = 1; j <= 16; ++j) {
        // 计算要添加到代码中以计算符号 ID 的增量
        h->delta[j] = k - code;
        // 如果 h->size[k] 等于 j，则循环直到 h->size[k] 不等于 j 为止，将 code 写入 h->code 数组
        if (h->size[k] == j) {
            while (h->size[k] == j)
                h->code[k++] = (stbi__uint16)(code++);
            // 如果 code-1 大于等于 2^j，则说明代码长度错误，返回错误信息
            if (code - 1 >= (1u << j))
                return stbi__err("bad code lengths", "Corrupt JPEG");
        }
    // 计算对于当前大小的最大代码 + 1，并根据需要进行预移位
    h->maxcode[j] = code << (16 - j);
    code <<= 1;
    }
    // 将最大代码设置为 0xffffffff
    h->maxcode[j] = 0xffffffff;

    // 构建非规范加速表；255 用作未加速的标志
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
    // 返回 1 表示成功
    return 1;
}
// 构建一个表，一次性解码小AC的幅度和值。
static void stbi__build_fast_ac(stbi__int16 * fast_ac, stbi__huffman * h) {
    int i;
    for (i = 0; i < (1 << FAST_BITS); ++i) {
        // 获取快速查找表中的值
        stbi_uc fast = h->fast[i];
        // 初始化快速AC表
        fast_ac[i] = 0;
        // 如果快速查找表中的值小于255
        if (fast < 255) {
            // 获取值对应的符号和大小
            int rs = h->values[fast];
            // 获取运行长度和幅度大小
            int run = (rs >> 4) & 15;
            int magbits = rs & 15;
            int len = h->size[fast];

            // 如果幅度大小不为0且长度加上幅度大小小于等于FAST_BITS
            if (magbits && len + magbits <= FAST_BITS) {
                // 幅度码后跟接收扩展码
                int k = ((i << len) & ((1 << FAST_BITS) - 1)) >> (FAST_BITS - magbits);
                int m = 1 << (magbits - 1);
                // 如果k小于m
                if (k < m)
                    // 对k进行符号扩展
                    k += (~0U << magbits) + 1;
// 如果结果足够小，可以将其放入fast_ac表中
if (k >= -128 && k <= 127)
    fast_ac[i] = (stbi__int16)((k * 256) + (run * 16) + (len + magbits));
// 增长缓冲区，不安全的操作
static void stbi__grow_buffer_unsafe(stbi__jpeg * j) {
    do {
        unsigned int b = j->nomore ? 0 : stbi__get8(j->s);
        // 如果读取到0xff，继续读取下一个字节
        if (b == 0xff) {
            int c = stbi__get8(j->s);
            // 消耗填充字节
            while (c == 0xff)
                c = stbi__get8(j->s);
            // 如果c不等于0，设置标记并返回
            if (c != 0) {
                j->marker = (unsigned char)c;
                j->nomore = 1;
                return;
            }
    }
    // 将下一个字节的数据合并到code_buffer中
    j->code_buffer |= b << (24 - j->code_bits);
    // 增加code_bits的值
    j->code_bits += 8;
} while (j->code_bits <= 24);
}

// (1 << n) - 1
// 定义一个包含2^n-1的数组
static const stbi__uint32 stbi__bmask[17] = {0,   1,    3,    7,    15,   31,    63,    127,  255,
                                             511, 1023, 2047, 4095, 8191, 16383, 32767, 65535};

// 从比特流中解码jpeg huffman值
stbi_inline static int stbi__jpeg_huff_decode(stbi__jpeg * j, stbi__huffman * h) {
    unsigned int temp;
    int c, k;

    if (j->code_bits < 16)
        // 如果code_bits小于16，则增加缓冲区的大小
        stbi__grow_buffer_unsafe(j);

    // 查看顶部的FAST_BITS并确定它是什么符号ID，如果代码<= FAST_BITS
    // 从code_buffer中取出前32-FAST_BITS位的值，并且将其与(1 << FAST_BITS) - 1进行与运算
    c = (j->code_buffer >> (32 - FAST_BITS)) & ((1 << FAST_BITS) - 1);
    // 从h->fast数组中获取对应索引c的值
    k = h->fast[c];
    // 如果k小于255，则执行以下操作
    if (k < 255) {
        // 获取h->size数组中索引为k的值
        int s = h->size[k];
        // 如果s大于j->code_bits，则返回-1
        if (s > j->code_bits)
            return -1;
        // 将code_buffer左移s位
        j->code_buffer <<= s;
        // 减去s位
        j->code_bits -= s;
        // 返回h->values数组中索引为k的值
        return h->values[k];
    }

    // 将code_buffer右移16位，赋值给temp
    temp = j->code_buffer >> 16;
    // 循环，从FAST_BITS + 1开始，直到找到合适的maxcode
    for (k = FAST_BITS + 1;; ++k)
        // 如果temp小于h->maxcode数组中索引为k的值，则继续循环
        if (temp < h->maxcode[k])
    // 如果遇到 17，表示错误，未找到对应的代码
    if (k == 17) {
        // 错误！未找到对应的代码
        j->code_bits -= 16;
        return -1;
    }

    // 如果 k 大于当前可用的比特数，返回错误
    if (k > j->code_bits)
        return -1;

    // 将哈夫曼编码转换为符号 ID
    c = ((j->code_buffer >> (32 - k)) & stbi__bmask[k]) + h->delta[k];
    // 如果符号 ID 超出范围，返回错误
    if (c < 0 || c >= 256) // 符号 ID 超出范围！
        return -1;
    // 断言哈夫曼编码是否正确
    STBI_ASSERT((((j->code_buffer) >> (32 - h->size[c])) & stbi__bmask[h->size[c]]) == h->code[c]);

    // 将 ID 转换为符号
    j->code_bits -= k;
    j->code_buffer <<= k;
    return h->values[c];
// bias[n] = (-1<<n) + 1
// 定义一个静态常量数组，用于JPEG解码时的偏置计算

// combined JPEG 'receive' and JPEG 'extend', since baseline
// always extends everything it receives.
// 结合了JPEG的“接收”和“扩展”操作，因为基线总是扩展它接收到的所有数据。

// 定义一个静态内联函数，用于JPEG解码时的数据接收和扩展操作
// 参数j为JPEG解码器指针，参数n为要接收和扩展的位数
stbi_inline static int stbi__extend_receive(stbi__jpeg * j, int n) {
    unsigned int k;
    int sgn;
    // 如果当前缓冲区中的比特数小于n，则扩展缓冲区
    if (j->code_bits < n)
        stbi__grow_buffer_unsafe(j);
    // 如果扩展后的缓冲区中的比特数仍然小于n，则返回0，表示从流中读取的比特不足
    if (j->code_bits < n)
        return 0; // ran out of bits from stream, return 0s intead of continuing

    // 获取符号位，即缓冲区中的最高位，如果最高位为0则为正数，为1则为负数
    sgn = j->code_buffer >> 31; // sign bit always in MSB; 0 if MSB clear (positive), 1 if MSB set (negative)
    // 将缓冲区中的数据左旋n位，得到接收和扩展后的数据
    k = stbi_lrot(j->code_buffer, n);
    // 更新缓冲区中的数据，去除已经接收和扩展的部分
    j->code_buffer = k & ~stbi__bmask[n];
    // 获取接收和扩展后的数据
    k &= stbi__bmask[n];
    // 更新缓冲区中的比特数
    j->code_bits -= n;
// 返回 k 加上 stbi__jbias[n] 与 (sgn - 1) 的按位与结果
return k + (stbi__jbias[n] & (sgn - 1));
}

// 获取一些无符号位
stbi_inline static int stbi__jpeg_get_bits(stbi__jpeg * j, int n) {
    unsigned int k;
    // 如果代码位数小于 n，则增加缓冲区大小
    if (j->code_bits < n)
        stbi__grow_buffer_unsafe(j);
    // 如果代码位数小于 n，则返回 0，表示从流中用尽了位，返回 0 而不是继续
    if (j->code_bits < n)
        return 0; 
    k = stbi_lrot(j->code_buffer, n); // 将代码缓冲区左旋 n 位
    j->code_buffer = k & ~stbi__bmask[n]; // 将代码缓冲区与非位掩码[n]进行按位与操作
    k &= stbi__bmask[n]; // 将 k 与位掩码[n]进行按位与操作
    j->code_bits -= n; // 代码位数减去 n
    return k; // 返回 k
}

// 获取一个位
stbi_inline static int stbi__jpeg_get_bit(stbi__jpeg * j) {
    unsigned int k;
    // 如果代码位数小于 1
        stbi__grow_buffer_unsafe(j);
    // 如果当前缓冲区的位数小于1，则表示从流中读取的位数不够，返回0代替继续读取
    if (j->code_bits < 1)
        return 0; // ran out of bits from stream, return 0s intead of continuing
    // 将当前缓冲区的值赋给k
    k = j->code_buffer;
    // 左移一位，相当于将当前缓冲区的值乘以2
    j->code_buffer <<= 1;
    // 缓冲区的位数减1
    --j->code_bits;
    // 返回k与0x80000000的按位与结果
    return k & 0x80000000;
}

// 给定在zigzag流中位置为X的值，它在以行优先方式编码的8x8矩阵中的位置是哪里？
// 静态数组，存储了zigzag流中位置与8x8矩阵中位置的对应关系
static const stbi_uc stbi__jpeg_dezigzag[64 + 15] = {
    // 省略了数组中的具体数值
    // let corrupt input sample past end
    // 用于处理损坏的输入样本
    63, 63, 63, 63, 63, 63, 63, 63, 63, 63, 63, 63, 63, 63, 63};

// 解码一个64个条目的块
static int stbi__jpeg_decode_block(stbi__jpeg * j, short data[64], stbi__huffman * hdc, stbi__huffman * hac, stbi__int16 * fac,
                                   int b, stbi__uint16 * dequant) {
    // 声明变量 diff, dc, k, t
    int diff, dc, k;
    int t;

    // 如果当前处理的码字位数小于16，则扩展缓冲区
    if (j->code_bits < 16)
        stbi__grow_buffer_unsafe(j);
    // 从哈夫曼表中解码一个值
    t = stbi__jpeg_huff_decode(j, hdc);
    // 如果解码值小于0或大于15，则返回错误信息
    if (t < 0 || t > 15)
        return stbi__err("bad huffman code", "Corrupt JPEG");

    // 将数据数组中的所有AC值置为0，以便后续可以一次处理32位
    memset(data, 0, 64 * sizeof(data[0]));

    // 如果 t 不为0，则使用扩展接收函数处理 t，否则置为0
    diff = t ? stbi__extend_receive(j, t) : 0;
    // 如果无法将当前图像组件的 DC 预测值和 diff 相加，则返回错误信息
    if (!stbi__addints_valid(j->img_comp[b].dc_pred, diff))
        return stbi__err("bad delta", "Corrupt JPEG");
    // 计算 DC 值
    dc = j->img_comp[b].dc_pred + diff;
    j->img_comp[b].dc_pred = dc;
    // 如果无法将 dc 和 dequant[0] 相乘，则返回错误信息
    if (!stbi__mul2shorts_valid(dc, dequant[0]))
        return stbi__err("can't merge dc and ac", "Corrupt JPEG");
    // 将计算得到的值存入数据数组中
    data[0] = (short)(dc * dequant[0]);
    // 解码交流分量，参考 JPEG 规范
    k = 1;  // 初始化变量 k
    do {
        unsigned int zig;  // 定义无符号整数 zig
        int c, r, s;  // 定义整数 c, r, s
        if (j->code_bits < 16)  // 如果码流缓冲区的位数小于 16
            stbi__grow_buffer_unsafe(j);  // 调用函数扩展缓冲区
        c = (j->code_buffer >> (32 - FAST_BITS)) & ((1 << FAST_BITS) - 1);  // 从码流缓冲区中获取 c
        r = fac[c];  // 从 fac 数组中获取 r
        if (r) {  // 如果 r 存在
            k += (r >> 4) & 15;  // 更新 k
            s = r & 15;  // 获取 s
            if (s > j->code_bits)  // 如果 s 大于码流缓冲区的位数
                return stbi__err("bad huffman code", "Combined length longer than code bits available");  // 返回错误信息
            j->code_buffer <<= s;  // 将码流缓冲区左移 s 位
            j->code_bits -= s;  // 更新码流缓冲区的位数
            // 解码到非 ZigZag 位置
            zig = stbi__jpeg_dezigzag[k++];  // 获取非 ZigZag 位置
            data[zig] = (short)((r >> 8) * dequant[zig]);  // 解码并存储数据
        } else {
            // 使用哈夫曼解码器解码 JPEG 数据
            int rs = stbi__jpeg_huff_decode(j, hac);
            // 如果解码结果小于 0，返回错误信息
            if (rs < 0)
                return stbi__err("bad huffman code", "Corrupt JPEG");
            // 获取解码结果的低 4 位和高 4 位
            s = rs & 15;
            r = rs >> 4;
            // 如果低 4 位为 0，表示结束当前块
            if (s == 0) {
                if (rs != 0xf0)
                    break; // 结束块
                k += 16;
            } else {
                k += r;
                // 解码到非 Zigzag 位置
                zig = stbi__jpeg_dezigzag[k++];
                data[zig] = (short)(stbi__extend_receive(j, s) * dequant[zig]);
            }
        }
    } while (k < 64);
    return 1;
}
static int stbi__jpeg_decode_block_prog_dc(stbi__jpeg * j, short data[64], stbi__huffman * hdc, int b) {
    // 解码 JPEG 数据流中的 DC 系数
    int diff, dc;
    int t;
    if (j->spec_end != 0)
        return stbi__err("can't merge dc and ac", "Corrupt JPEG");

    if (j->code_bits < 16)
        stbi__grow_buffer_unsafe(j);

    if (j->succ_high == 0) {
        // 如果是第一次扫描 DC 系数，需要将 AC 系数全部置为 0
        memset(data, 0, 64 * sizeof(data[0])); // 0 all the ac values now
        // 从哈夫曼编码中解码 DC 系数
        t = stbi__jpeg_huff_decode(j, hdc);
        if (t < 0 || t > 15)
            return stbi__err("can't merge dc and ac", "Corrupt JPEG");
        // 计算 DC 系数的差值
        diff = t ? stbi__extend_receive(j, t) : 0;

        // 检查差值是否合法
        if (!stbi__addints_valid(j->img_comp[b].dc_pred, diff))
            return stbi__err("bad delta", "Corrupt JPEG");
        // 计算直流系数
        dc = j->img_comp[b].dc_pred + diff;
        // 更新直流预测值
        j->img_comp[b].dc_pred = dc;
        // 检查直流系数和后继低位是否有效
        if (!stbi__mul2shorts_valid(dc, 1 << j->succ_low))
            // 如果无效，返回错误信息
            return stbi__err("can't merge dc and ac", "Corrupt JPEG");
        // 计算直流系数的值
        data[0] = (short)(dc * (1 << j->succ_low));
    } else {
        // 对直流系数进行细化扫描
        if (stbi__jpeg_get_bit(j))
            // 如果下一个比特为1，直流系数加上2^j->succ_low
            data[0] += (short)(1 << j->succ_low);
    }
    // 返回1表示成功
    return 1;
}

// @OPTIMIZE: 在解码过程中存储非蛇形排列的数据，
// 只有在反量化时才进行蛇形排列
static int stbi__jpeg_decode_block_prog_ac(stbi__jpeg * j, short data[64], stbi__huffman * hac, stbi__int16 * fac) {
    int k;
    // 如果spec_start为0，表示无法合并直流和交流系数，返回错误信息
    if (j->spec_start == 0)
        return stbi__err("can't merge dc and ac", "Corrupt JPEG");
    # 如果 j->succ_high 等于 0
    if (j->succ_high == 0) {
        # 将 j->succ_low 赋值给 shift
        int shift = j->succ_low;

        # 如果 j->eob_run 不为 0
        if (j->eob_run) {
            # 减少 j->eob_run 的值并返回 1
            --j->eob_run;
            return 1;
        }

        # 将 j->spec_start 赋值给 k
        k = j->spec_start;
        # 进入循环
        do {
            # 定义变量 zig
            unsigned int zig;
            # 定义变量 c, r, s
            int c, r, s;
            # 如果 j->code_bits 小于 16
            if (j->code_bits < 16)
                # 调用 stbi__grow_buffer_unsafe 函数
                stbi__grow_buffer_unsafe(j);
            # 将 (j->code_buffer >> (32 - FAST_BITS)) & ((1 << FAST_BITS) - 1) 赋值给 c
            c = (j->code_buffer >> (32 - FAST_BITS)) & ((1 << FAST_BITS) - 1);
            # 将 fac[c] 赋值给 r
            r = fac[c];
            # 如果 r 不为 0
            if (r) {                # fast-AC path
                # 将 (r >> 4) & 15 加到 k 上，赋值给 k
                k += (r >> 4) & 15; # run
                # 将 r & 15 赋值给 s
                s = r & 15;         # combined length
                # 如果 s 大于 j->code_bits
                if (s > j->code_bits)
                return stbi__err("bad huffman code", "Combined length longer than code bits available");
                # 如果组合长度超过了可用的编码位数，返回错误信息
                j->code_buffer <<= s;
                # 将编码缓冲区左移 s 位
                j->code_bits -= s;
                # 减去已经处理的编码位数
                zig = stbi__jpeg_dezigzag[k++];
                # 从 JPEG 解码表中获取 zigzag 顺序的索引
                data[zig] = (short)((r >> 8) * (1 << shift));
                # 将解码后的数据存入 data 数组中
            } else {
                int rs = stbi__jpeg_huff_decode(j, hac);
                # 从哈夫曼表中解码出一个值
                if (rs < 0)
                    return stbi__err("bad huffman code", "Corrupt JPEG");
                    # 如果解码出的值小于 0，返回错误信息
                s = rs & 15;
                # 获取 rs 的低 4 位
                r = rs >> 4;
                # 获取 rs 的高 4 位
                if (s == 0) {
                    # 如果 s 为 0
                    if (r < 15) {
                        # 如果 r 小于 15
                        j->eob_run = (1 << r);
                        # 计算 eob_run 的值
                        if (r)
                            j->eob_run += stbi__jpeg_get_bits(j, r);
                            # 如果 r 不为 0，继续获取位数为 r 的数据
                        --j->eob_run;
                        # 减去 eob_run 的值
                        break;
                        # 跳出循环
                    }
                    k += 16;
                    # k 增加 16
                } else {
                    // 如果不是 EOB，进行数据解码
                    k += r;
                    // 获取解码后的 Zig 值
                    zig = stbi__jpeg_dezigzag[k++];
                    // 对数据进行扩展和移位，存入 data 数组
                    data[zig] = (short)(stbi__extend_receive(j, s) * (1 << shift));
                }
            }
        } while (k <= j->spec_end);
    } else {
        // 对 AC 系数进行细化扫描

        // 计算 bit 值
        short bit = (short)(1 << j->succ_low);

        if (j->eob_run) {
            // 如果 eob_run 不为 0，减少 eob_run 的值
            --j->eob_run;
            // 遍历特定范围内的数据
            for (k = j->spec_start; k <= j->spec_end; ++k) {
                // 获取指向 data 数组中特定位置的指针
                short * p = &data[stbi__jpeg_dezigzag[k]];
                // 如果指针指向的值不为 0
                if (*p != 0)
                    // 如果满足条件，进行进一步处理
                    if (stbi__jpeg_get_bit(j))
                        if ((*p & bit) == 0) {
                            if (*p > 0)
                *p += bit; // 如果条件成立，将指针所指向的值加上bit
                else
                    *p -= bit; // 如果条件不成立，将指针所指向的值减去bit
            }
        }
    } else {
        k = j->spec_start; // 将j->spec_start的值赋给k
        do {
            int r, s;
            int rs = stbi__jpeg_huff_decode(
                j, hac); // 使用stbi__jpeg_huff_decode函数解码JPEG数据，将结果赋给rs
            if (rs < 0)
                return stbi__err("bad huffman code", "Corrupt JPEG"); // 如果rs小于0，返回错误信息
            s = rs & 15; // 将rs与15进行按位与操作，将结果赋给s
            r = rs >> 4; // 将rs右移4位，将结果赋给r
            if (s == 0) { // 如果s等于0
                if (r < 15) { // 如果r小于15
                    j->eob_run = (1 << r) - 1; // 计算eob_run的值
                    if (r)
                        j->eob_run += stbi__jpeg_get_bits(j, r); // 如果r不为0，调用stbi__jpeg_get_bits函数并将结果加到eob_run上
                    r = 64; // 强制结束块
                } else {
                    // r=15 s=0 应该写入16个0，所以我们只需写入15个0，然后写入s（即0），所以这里不需要特殊处理
                }
            } else {
                if (s != 1)
                    return stbi__err("bad huffman code", "Corrupt JPEG");
                // 符号位
                if (stbi__jpeg_get_bit(j))
                    s = bit;
                else
                    s = -bit;
            }

            // 按照r的值前进
            while (k <= j->spec_end) {
                short * p = &data[stbi__jpeg_dezigzag[k++]];
                if (*p != 0) {
# 如果当前位为1，则执行以下操作
if (stbi__jpeg_get_bit(j))
    # 如果当前字节的指定位为0
    if ((*p & bit) == 0) {
        # 如果当前字节的值大于0
        if (*p > 0)
            # 将当前字节的值加上指定位
            *p += bit;
        else
            # 否则将当前字节的值减去指定位
            *p -= bit;
    }
# 如果当前位为0
else {
    # 如果r为0
    if (r == 0) {
        # 将当前字节的值设为s的short类型值
        *p = (short)s;
        # 跳出循环
        break;
    }
    # 否则r减1
    --r;
}
# 循环直到k大于j的spec_end
} while (k <= j->spec_end);
# 返回1
return 1;
// 将一个 -128 到 127 的值限制在 0 到 255 之间，并转换为无符号字符型
stbi_inline static stbi_uc stbi__clamp(int x) {
    // 用一个条件判断来同时处理两种情况的技巧
    if ((unsigned int)x > 255) {
        if (x < 0)
            return 0;
        if (x > 255)
            return 255;
    }
    return (stbi_uc)x;
}

// 将浮点数乘以 4096 并四舍五入到整数
#define stbi__f2f(x) ((int)(((x)*4096 + 0.5)))
// 将浮点数乘以 4096
#define stbi__fsh(x) ((x)*4096)

// 源自 jidctint -- DCT_ISLOW，一维逆离散余弦变换
#define STBI__IDCT_1D(s0, s1, s2, s3, s4, s5, s6, s7)                                                                          \
    int t0, t1, t2, t3, p1, p2, p3, p4, p5, x0, x1, x2, x3;                                                                    \
    p2 = s2;  // 将 s2 赋值给 p2
    # 将变量 s6 的值赋给变量 p3
    p3 = s6;                                                                                                                   \
    # 计算 p2 和 p3 的和，乘以常数 0.5411961，赋给变量 p1
    p1 = (p2 + p3) * stbi__f2f(0.5411961f);                                                                                    \
    # 计算 p1 和 p3 乘以常数 -1.847759065 的和，赋给变量 t2
    t2 = p1 + p3 * stbi__f2f(-1.847759065f);                                                                                   \
    # 计算 p1 和 p2 乘以常数 0.765366865 的和，赋给变量 t3
    t3 = p1 + p2 * stbi__f2f(0.765366865f);                                                                                    \
    # 将变量 s0 的值赋给变量 p2
    p2 = s0;                                                                                                                   \
    # 将变量 s4 的值赋给变量 p3
    p3 = s4;                                                                                                                   \
    # 计算 p2 和 p3 的和，并进行位移操作，赋给变量 t0
    t0 = stbi__fsh(p2 + p3);                                                                                                   \
    # 计算 p2 和 p3 的差，并进行位移操作，赋给变量 t1
    t1 = stbi__fsh(p2 - p3);                                                                                                   \
    # 计算 t0 和 t3 的和，赋给变量 x0
    x0 = t0 + t3;                                                                                                              \
    # 计算 t0 和 t3 的差，赋给变量 x3
    x3 = t0 - t3;                                                                                                              \
    # 计算 t1 和 t2 的和，赋给变量 x1
    x1 = t1 + t2;                                                                                                              \
    # 计算 t1 和 t2 的差，赋给变量 x2
    x2 = t1 - t2;                                                                                                              \
    # 将变量 s7 的值赋给变量 t0
    t0 = s7;                                                                                                                   \
    # 将变量 s5 的值赋给变量 t1
    t1 = s5;                                                                                                                   \
    # 将变量 s3 的值赋给变量 t2
    t2 = s3;                                                                                                                   \
    # 将变量 s1 的值赋给变量 t3
    t3 = s1;                                                                                                                   \
    # 计算 t0 和 t2 的和，赋给变量 p3
    p3 = t0 + t2;                                                                                                              \
    # 计算 t1 和 t3 的和，赋给变量 p4
    p4 = t1 + t3;                                                                                                              \
    # 计算 t0 和 t3 的和，赋给变量 p1
    p1 = t0 + t3;                                                                                                              \
    # 计算 t1 和 t2 的和，赋给变量 p2
    p2 = t1 + t2;                                                                                                              \
    p5 = (p3 + p4) * stbi__f2f(1.175875602f);  \                    # 计算 p5 的值
    t0 = t0 * stbi__f2f(0.298631336f);         \                    # 计算 t0 的值
    t1 = t1 * stbi__f2f(2.053119869f);         \                    # 计算 t1 的值
    t2 = t2 * stbi__f2f(3.072711026f);         \                    # 计算 t2 的值
    t3 = t3 * stbi__f2f(1.501321110f);         \                    # 计算 t3 的值
    p1 = p5 + p1 * stbi__f2f(-0.899976223f);   \                    # 计算 p1 的值
    p2 = p5 + p2 * stbi__f2f(-2.562915447f);   \                    # 计算 p2 的值
    p3 = p3 * stbi__f2f(-1.961570560f);        \                    # 计算 p3 的值
    p4 = p4 * stbi__f2f(-0.390180644f);        \                    # 计算 p4 的值
    t3 += p1 + p4;                               \                  # 更新 t3 的值
    t2 += p2 + p3;                               \                  # 更新 t2 的值
    t1 += p2 + p4;                               \                  # 更新 t1 的值
    t0 += p1 + p3;                               \                  # 更新 t0 的值

static void stbi__idct_block(stbi_uc * out, int out_stride, short data[64]) {   # 定义函数 stbi__idct_block，接受输出指针、输出步长和数据数组
    int i, val[64], *v = val;                   # 定义变量 i、val 数组和指针 v
    stbi_uc * o;                                # 定义指针 o
    short * d = data;                           # 定义指针 d，指向数据数组

    // columns                                   # 对列进行操作
    // 循环8次，每次增加d和v的索引值
    for (i = 0; i < 8; ++i, ++d, ++v) {
        // 如果d数组中的特定位置都是0，则进行快捷处理，避免反量化和IDCT
        if (d[8] == 0 && d[16] == 0 && d[24] == 0 && d[32] == 0 && d[40] == 0 && d[48] == 0 && d[56] == 0) {
            //    没有快捷方式                 0     秒
            //    (1|2|3|4|5|6|7)==0          0     秒
            //    所有分开的情况             -0.047 秒
            //    1 && 2|3 && 4|5 && 6|7:    -0.047 秒
            // 计算d[0]乘以4的结果
            int dcterm = d[0] * 4;
            // 将结果赋值给v数组的特定位置
            v[0] = v[8] = v[16] = v[24] = v[32] = v[40] = v[48] = v[56] = dcterm;
        } else {
            // 对d数组进行一维IDCT变换
            STBI__IDCT_1D(d[0], d[8], d[16], d[24], d[32], d[40], d[48], d[56])
            // 将常数扩大2^12倍，现在将其缩小，但保留2位额外的精度
            x0 += 512;
            x1 += 512;
            x2 += 512;
            x3 += 512;
            v[0] = (x0 + t3) >> 10;
            v[56] = (x0 - t3) >> 10;
            v[8] = (x1 + t2) >> 10;
        // 将计算结果右移 10 位，存入数组 v 的第 48 个位置
        v[48] = (x1 - t2) >> 10;
        // 将计算结果右移 10 位，存入数组 v 的第 16 个位置
        v[16] = (x2 + t1) >> 10;
        // 将计算结果右移 10 位，存入数组 v 的第 40 个位置
        v[40] = (x2 - t1) >> 10;
        // 将计算结果右移 10 位，存入数组 v 的第 24 个位置
        v[24] = (x3 + t0) >> 10;
        // 将计算结果右移 10 位，存入数组 v 的第 32 个位置
        v[32] = (x3 - t0) >> 10;
    }

    for (i = 0, v = val, o = out; i < 8; ++i, v += 8, o += out_stride) {
        // 没有快速情况，因为第一个 1D IDCT 将分量展开
        // 对数组 v 中的元素进行一维 IDCT 变换
        STBI__IDCT_1D(v[0], v[1], v[2], v[3], v[4], v[5], v[6], v[7])
        // 常数将结果扩大了 1<<12，加上第一个循环中的 1<<2，再加上水平和垂直方向上的缩放 sqrt(8)
        // 所以总共需要移除 1<<17，为了四舍五入，需要加上 0.5 * 1<<17，即 65536
        // 同时，结果范围是 -128 到 127，需要加上 128 以将其编码为 0 到 255
        x0 += 65536 + (128 << 17);
        x1 += 65536 + (128 << 17);
        x2 += 65536 + (128 << 17);
        x3 += 65536 + (128 << 17);
        // 将 x3 增加 65536 加上 (128 左移 17) 的结果
        // 尝试计算移位到临时变量，或将临时变量进行或运算，以查看是否有任何超出范围的情况，但这样做速度更慢
        o[0] = stbi__clamp((x0 + t3) >> 17);
        // 将 (x0 + t3) 右移 17 位，并使用 stbi__clamp 函数对结果进行限制
        o[7] = stbi__clamp((x0 - t3) >> 17);
        // 将 (x0 - t3) 右移 17 位，并使用 stbi__clamp 函数对结果进行限制
        o[1] = stbi__clamp((x1 + t2) >> 17);
        // 将 (x1 + t2) 右移 17 位，并使用 stbi__clamp 函数对结果进行限制
        o[6] = stbi__clamp((x1 - t2) >> 17);
        // 将 (x1 - t2) 右移 17 位，并使用 stbi__clamp 函数对结果进行限制
        o[2] = stbi__clamp((x2 + t1) >> 17);
        // 将 (x2 + t1) 右移 17 位，并使用 stbi__clamp 函数对结果进行限制
        o[5] = stbi__clamp((x2 - t1) >> 17);
        // 将 (x2 - t1) 右移 17 位，并使用 stbi__clamp 函数对结果进行限制
        o[3] = stbi__clamp((x3 + t0) >> 17);
        // 将 (x3 + t0) 右移 17 位，并使用 stbi__clamp 函数对结果进行限制
        o[4] = stbi__clamp((x3 - t0) >> 17);
        // 将 (x3 - t0) 右移 17 位，并使用 stbi__clamp 函数对结果进行限制
    }
}

#ifdef STBI_SSE2
// sse2 整数 IDCT。虽然不是最快的实现方式，但它产生与通用 C 版本完全相同的结果，因此它是完全“透明”的。
static void stbi__idct_simd(stbi_uc * out, int out_stride, short data[64]) {
    // 这是为了与我们的常规（通用）整数 IDCT 完全匹配而构建的。
// 定义8个128位整数变量，用于存储数据
__m128i row0, row1, row2, row3, row4, row5, row6, row7;
// 定义一个临时的128位整数变量
__m128i tmp;

// 定义宏，用于生成包含特定值的128位整数变量
#define dct_const(x, y) _mm_setr_epi16((x), (y), (x), (y), (x), (y), (x), (y))

// 定义宏，用于计算乘积和加法操作
#define dct_rot(out0, out1, x, y, c0, c1)                                                                                      
    // 将输入的x和y进行拆分，分别存储到c0##lo和c0##hi中
    __m128i c0##lo = _mm_unpacklo_epi16((x), (y));                                                                             
    __m128i c0##hi = _mm_unpackhi_epi16((x), (y));                                                                             
    // 计算out0的低位和高位
    __m128i out0##_l = _mm_madd_epi16(c0##lo, c0);                                                                             
    __m128i out0##_h = _mm_madd_epi16(c0##hi, c0);                                                                             
    // 计算out1的低位和高位
    __m128i out1##_l = _mm_madd_epi16(c0##lo, c1);                                                                             
    __m128i out1##_h = _mm_madd_epi16(c0##hi, c1)

// 定义宏，用于将输入的16位整数扩展为32位整数
#define dct_widen(out, in)                                                                                                     
    // 将输入的in进行拆分，分别存储到out##_l和out##_h中
    __m128i out##_l = _mm_srai_epi32(_mm_unpacklo_epi16(_mm_setzero_si128(), (in)), 4);                                        
    __m128i out##_h = _mm_srai_epi32(_mm_unpackhi_epi16(_mm_setzero_si128(), (in)), 4)
// 定义宏，用于计算两个__m128i类型的变量的加法
#define dct_wadd(out, a, b)                                                                                                    \
    __m128i out##_l = _mm_add_epi32(a##_l, b##_l);                                                                             \
    __m128i out##_h = _mm_add_epi32(a##_h, b##_h)

// 定义宏，用于计算两个__m128i类型的变量的减法
#define dct_wsub(out, a, b)                                                                                                    \
    __m128i out##_l = _mm_sub_epi32(a##_l, b##_l);                                                                             \
    __m128i out##_h = _mm_sub_epi32(a##_h, b##_h)

// 定义宏，用于进行蝶形运算，加上偏置，然后按位移动并打包
#define dct_bfly32o(out0, out1, a, b, bias, s)                                                                                 \
    {                                                                                                                          \
        // 对a加上偏置
        __m128i abiased_l = _mm_add_epi32(a##_l, bias);                                                                        \
        __m128i abiased_h = _mm_add_epi32(a##_h, bias);                                                                        \
        // 调用宏计算a+b和a-b
        dct_wadd(sum, abiased, b);                                                                                             \
        dct_wsub(dif, abiased, b);                                                                                             \
        // 对结果进行位移和打包
        out0 = _mm_packs_epi32(_mm_srai_epi32(sum_l, s), _mm_srai_epi32(sum_h, s));                                            \
        out1 = _mm_packs_epi32(_mm_srai_epi32(dif_l, s), _mm_srai_epi32(dif_h, s));                                            \
    }

// 8-bit interleave step (for transposes)
// 定义一个宏，用于对8位整数进行交错操作
#define dct_interleave8(a, b)                                                                                                  
    tmp = a;                                                                                                                   
    a = _mm_unpacklo_epi8(a, b);                                                                                              
    b = _mm_unpackhi_epi8(tmp, b)

// 16-bit interleave step (for transposes)
// 定义一个宏，用于对16位整数进行交错操作
#define dct_interleave16(a, b)                                                                                                 
    tmp = a;                                                                                                                   
    a = _mm_unpacklo_epi16(a, b);                                                                                             
    b = _mm_unpackhi_epi16(tmp, b)

// 定义一个宏，用于执行DCT变换的一步操作
#define dct_pass(bias, shift)                                                                                                  
    {                                                                                                                          
        /* even part */                                                                                                        
        // 执行DCT变换的偶数部分操作
        dct_rot(t2e, t3e, row2, row6, rot0_0, rot0_1);                                                                         
        __m128i sum04 = _mm_add_epi16(row0, row4);                                                                             
        __m128i dif04 = _mm_sub_epi16(row0, row4);                                                                             
        dct_widen(t0e, sum04);  // 将sum04扩展为16位，存储在t0e中
        dct_widen(t1e, dif04);  // 将dif04扩展为16位，存储在t1e中
        dct_wadd(x0, t0e, t3e);  // 计算t0e和t3e的和，存储在x0中
        dct_wsub(x3, t0e, t3e);  // 计算t0e和t3e的差，存储在x3中
        dct_wadd(x1, t1e, t2e);  // 计算t1e和t2e的和，存储在x1中
        dct_wsub(x2, t1e, t2e);  // 计算t1e和t2e的差，存储在x2中
        /* odd part */  // 奇数部分
        dct_rot(y0o, y2o, row7, row3, rot2_0, rot2_1);  // 对y0o和y2o进行旋转操作，存储在row7和row3中
        dct_rot(y1o, y3o, row5, row1, rot3_0, rot3_1);  // 对y1o和y3o进行旋转操作，存储在row5和row1中
        __m128i sum17 = _mm_add_epi16(row1, row7);  // 计算row1和row7的和，存储在sum17中
        __m128i sum35 = _mm_add_epi16(row3, row5);  // 计算row3和row5的和，存储在sum35中
        dct_rot(y4o, y5o, sum17, sum35, rot1_0, rot1_1);  // 对y4o和y5o进行旋转操作，存储在sum17和sum35中
        dct_wadd(x4, y0o, y4o);  // 计算y0o和y4o的和，存储在x4中
        dct_wadd(x5, y1o, y5o);  // 计算y1o和y5o的和，存储在x5中
        dct_wadd(x6, y2o, y5o);  // 计算y2o和y5o的和，存储在x6中
        dct_wadd(x7, y3o, y4o);  // 计算y3o和y4o的和，存储在x7中
        dct_bfly32o(row0, row7, x0, x7, bias, shift);  // 对row0和row7进行蝶形运算，存储在x0和x7中
        dct_bfly32o(row1, row6, x1, x6, bias, shift);  // 对row1和row6进行蝶形运算，存储在x1和x6中
        dct_bfly32o(row2, row5, x2, x5, bias, shift);  // 对row2和row5进行蝶形运算，存储在x2和x5中
        dct_bfly32o(row3, row4, x3, x4, bias, shift);  // 对row3和row4进行蝶形运算，存储在x3和x4中
    }

    // 定义旋转常量，用于离散余弦变换
    __m128i rot0_0 = dct_const(stbi__f2f(0.5411961f), stbi__f2f(0.5411961f) + stbi__f2f(-1.847759065f));
    __m128i rot0_1 = dct_const(stbi__f2f(0.5411961f) + stbi__f2f(0.765366865f), stbi__f2f(0.5411961f));
    __m128i rot1_0 = dct_const(stbi__f2f(1.175875602f) + stbi__f2f(-0.899976223f), stbi__f2f(1.175875602f));
    __m128i rot1_1 = dct_const(stbi__f2f(1.175875602f), stbi__f2f(1.175875602f) + stbi__f2f(-2.562915447f));
    __m128i rot2_0 = dct_const(stbi__f2f(-1.961570560f) + stbi__f2f(0.298631336f), stbi__f2f(-1.961570560f));
    __m128i rot2_1 = dct_const(stbi__f2f(-1.961570560f), stbi__f2f(-1.961570560f) + stbi__f2f(3.072711026f));
    __m128i rot3_0 = dct_const(stbi__f2f(-0.390180644f) + stbi__f2f(2.053119869f), stbi__f2f(-0.390180644f));
    __m128i rot3_1 = dct_const(stbi__f2f(-0.390180644f), stbi__f2f(-0.390180644f) + stbi__f2f(1.501321110f));

    // 在列/行传递中使用的舍入偏差，参见stbi__idct_block的解释
    __m128i bias_0 = _mm_set1_epi32(512);
    __m128i bias_1 = _mm_set1_epi32(65536 + (128 << 17));

    // 加载数据
    row0 = _mm_load_si128((const __m128i *)(data + 0 * 8));
    row1 = _mm_load_si128((const __m128i *)(data + 1 * 8));
    row2 = _mm_load_si128((const __m128i *)(data + 2 * 8));
    row3 = _mm_load_si128((const __m128i *)(data + 3 * 8));
    # 从数据中加载特定位置的 128 位数据到寄存器中
    row4 = _mm_load_si128((const __m128i *)(data + 4 * 8));
    row5 = _mm_load_si128((const __m128i *)(data + 5 * 8));
    row6 = _mm_load_si128((const __m128i *)(data + 6 * 8));
    row7 = _mm_load_si128((const __m128i *)(data + 7 * 8));

    // 列处理
    dct_pass(bias_0, 10);

    {
        // 16位 8x8 转置处理第一步
        dct_interleave16(row0, row4);
        dct_interleave16(row1, row5);
        dct_interleave16(row2, row6);
        dct_interleave16(row3, row7);

        // 转置处理第二步
        dct_interleave16(row0, row2);
        dct_interleave16(row1, row3);
        dct_interleave16(row4, row6);
        dct_interleave16(row5, row7);
// 转置处理第三步
dct_interleave16(row0, row1); // 对16个元素的数组进行交错处理
dct_interleave16(row2, row3);
dct_interleave16(row4, row5);
dct_interleave16(row6, row7);

// 行处理
dct_pass(bias_1, 17); // 对行进行DCT变换

{
    // 打包
    __m128i p0 = _mm_packus_epi16(row0, row1); // 将row0和row1中的16位整数打包成8位整数
    __m128i p1 = _mm_packus_epi16(row2, row3);
    __m128i p2 = _mm_packus_epi16(row4, row5);
    __m128i p3 = _mm_packus_epi16(row6, row7);

    // 8位 8x8 转置处理第一步
    dct_interleave8(p0, p2); // 对8个8位整数的数组进行交错处理
}
        dct_interleave8(p1, p3); // 交错处理p1和p3的数据，生成c0g0c1g1...

        // 转置处理第二步
        dct_interleave8(p0, p1); // 交错处理p0和p1的数据，生成a0c0e0g0...
        dct_interleave8(p2, p3); // 交错处理p2和p3的数据，生成b0d0f0h0...

        // 转置处理第三步
        dct_interleave8(p0, p2); // 交错处理p0和p2的数据，生成a0b0c0d0...
        dct_interleave8(p1, p3); // 交错处理p1和p3的数据，生成a4b4c4d4...

        // 存储处理结果
        _mm_storel_epi64((__m128i *)out, p0); // 将p0的数据存储到out指向的内存地址
        out += out_stride; // 更新out指针位置
        _mm_storel_epi64((__m128i *)out, _mm_shuffle_epi32(p0, 0x4e)); // 将p0的数据进行特定的位移后存储到out指向的内存地址
        out += out_stride; // 更新out指针位置
        _mm_storel_epi64((__m128i *)out, p2); // 将p2的数据存储到out指向的内存地址
        out += out_stride; // 更新out指针位置
        _mm_storel_epi64((__m128i *)out, _mm_shuffle_epi32(p2, 0x4e)); // 将p2的数据进行特定的位移后存储到out指向的内存地址
        out += out_stride; // 更新out指针位置
        _mm_storel_epi64((__m128i *)out, p1); // 将p1的数据存储到out指向的内存地址
// 将 out 指针向前移动一个 out_stride 的距离
out += out_stride;
// 将 p1 中的数据按照指定顺序存储到 out 指针指向的内存中
_mm_storel_epi64((__m128i *)out, _mm_shuffle_epi32(p1, 0x4e));
// 将 out 指针向前移动一个 out_stride 的距离
out += out_stride;
// 将 p3 中的数据存储到 out 指针指向的内存中
_mm_storel_epi64((__m128i *)out, p3);
// 将 out 指针向前移动一个 out_stride 的距离
out += out_stride;
// 将 p3 中的数据按照指定顺序存储到 out 指针指向的内存中
_mm_storel_epi64((__m128i *)out, _mm_shuffle_epi32(p3, 0x4e));
// 取消之前定义的宏
#undef dct_const
#undef dct_rot
#undef dct_widen
#undef dct_wadd
#undef dct_wsub
#undef dct_bfly32o
#undef dct_interleave8
#undef dct_interleave16
#undef dct_pass
// 结束条件编译指令
#endif // STBI_SSE2
#ifdef STBI_NEON
// 如果定义了 STBI_NEON，则执行以下 NEON 整数 IDCT 函数

// 定义 NEON 寄存器变量，用于存储 IDCT 运算中的中间结果
static void stbi__idct_simd(stbi_uc * out, int out_stride, short data[64]) {
    int16x8_t row0, row1, row2, row3, row4, row5, row6, row7;

    // 定义 NEON 寄存器变量，用于存储 IDCT 运算中的常数
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
```

// 定义一个宏，用于将输入的低位和高位有符号16位整数乘以一个系数，并将结果存储在指定的输出变量中
#define dct_long_mul(out, inq, coeff)                                                                                          
    int32x4_t out##_l = vmull_s16(vget_low_s16(inq), coeff);                                                                   
    int32x4_t out##_h = vmull_s16(vget_high_s16(inq), coeff)

// 定义一个宏，用于将输入的低位和高位有符号16位整数乘以一个系数，并将结果与累加器中的值相加，然后将结果存储在指定的输出变量中
#define dct_long_mac(out, acc, inq, coeff)                                                                                     
    int32x4_t out##_l = vmlal_s16(acc##_l, vget_low_s16(inq), coeff);                                                          
    int32x4_t out##_h = vmlal_s16(acc##_h, vget_high_s16(inq), coeff)

// 定义一个宏，用于将输入的有符号16位整数扩展为32位整数，并将结果存储在指定的输出变量中
#define dct_widen(out, inq)                                                                                                    
    int32x4_t out##_l = vshll_n_s16(vget_low_s16(inq), 12);                                                                    
    int32x4_t out##_h = vshll_n_s16(vget_high_s16(inq), 12)

// 定义一个宏，用于将两个输入的32位整数进行宽度相加，并将结果存储在指定的输出变量中
#define dct_wadd(out, a, b)                                                                                                    
    int32x4_t out##_l = vaddq_s32(a##_l, b##_l);                                                                               
    int32x4_t out##_h = vaddq_s32(a##_h, b##_h)

// 定义一个宏，用于将两个输入的32位整数进行宽度相减，并将结果存储在指定的输出变量中
#define dct_wsub(out, a, b)                                                                                                    
// 计算两个 int32x4_t 类型的向量 a 和 b 的差，分别存储在 out##_l 和 out##_h 中
int32x4_t out##_l = vsubq_s32(a##_l, b##_l);
int32x4_t out##_h = vsubq_s32(a##_h, b##_h)

// 对 a 和 b 进行蝶形运算，然后使用 "shiftop" 进行位移，位移量为 "s"，并打包结果
#define dct_bfly32o(out0, out1, a, b, shiftop, s) 
{
    // 计算 a 和 b 的和
    dct_wadd(sum, a, b);
    // 计算 a 和 b 的差
    dct_wsub(dif, a, b);
    // 将结果进行位移和打包
    out0 = vcombine_s16(shiftop(sum_l, s), shiftop(sum_h, s));
    out1 = vcombine_s16(shiftop(dif_l, s), shiftop(dif_h, s));
}

// 执行 DCT 变换的一步操作，使用 "shiftop" 进行位移，位移量为 "shift"
#define dct_pass(shiftop, shift) 
{
    // 计算偶数部分
    int16x8_t sum26 = vaddq_s16(row2, row6);
    // 计算长乘法
    dct_long_mul(p1e, sum26, rot0_0);
    // 计算长乘累加
    dct_long_mac(t2e, p1e, row6, rot0_1);
    dct_long_mac(t3e, p1e, row2, rot0_2);
    int16x8_t sum04 = vaddq_s16(row0, row4);
}
        # 计算row0和row4的差值
        int16x8_t dif04 = vsubq_s16(row0, row4);
        # 将t0e扩展为32位，存储sum04的值
        dct_widen(t0e, sum04);
        # 将t1e扩展为32位，存储dif04的值
        dct_widen(t1e, dif04);
        # 计算x0 = t0e + t3e
        dct_wadd(x0, t0e, t3e);
        # 计算x3 = t0e - t3e
        dct_wsub(x3, t0e, t3e);
        # 计算x1 = t1e + t2e
        dct_wadd(x1, t1e, t2e);
        # 计算x2 = t1e - t2e
        dct_wsub(x2, t1e, t2e);
        # 奇数部分
        # 计算row1和row5的和
        int16x8_t sum15 = vaddq_s16(row1, row5);
        # 计算row1和row7的和
        int16x8_t sum17 = vaddq_s16(row1, row7);
        # 计算row3和row5的和
        int16x8_t sum35 = vaddq_s16(row3, row5);
        # 计算row3和row7的和
        int16x8_t sum37 = vaddq_s16(row3, row7);
        # 计算奇数部分的和
        int16x8_t sumodd = vaddq_s16(sum17, sum35);
        # 使用rot1_0对sumodd进行长乘法
        dct_long_mul(p5o, sumodd, rot1_0);
        # 使用rot1_1对p5o和sum17进行长乘加
        dct_long_mac(p1o, p5o, sum17, rot1_1);
        # 使用rot1_2对p5o和sum35进行长乘加
        dct_long_mac(p2o, p5o, sum35, rot1_2);
        # 使用rot2_0对sum37进行长乘法
        dct_long_mul(p3o, sum37, rot2_0);
        # 使用rot2_1对sum15进行长乘法
        dct_long_mul(p4o, sum15, rot2_1);
        # 计算sump13o = p1o + p3o
        dct_wadd(sump13o, p1o, p3o);
        # 计算sump24o = p2o + p4o
        dct_wadd(sump24o, p2o, p4o);
// 使用 dct_wadd 函数将 p2o 和 p3o 相加，并将结果存储到 sump23o 中
dct_wadd(sump23o, p2o, p3o);
// 使用 dct_wadd 函数将 p1o 和 p4o 相加，并将结果存储到 sump14o 中
dct_wadd(sump14o, p1o, p4o);
// 使用 dct_long_mac 函数将 sump13o 与 row7 进行长乘累加操作，并将结果存储到 x4 中
dct_long_mac(x4, sump13o, row7, rot3_0);
// 使用 dct_long_mac 函数将 sump24o 与 row5 进行长乘累加操作，并将结果存储到 x5 中
dct_long_mac(x5, sump24o, row5, rot3_1);
// 使用 dct_long_mac 函数将 sump23o 与 row3 进行长乘累加操作，并将结果存储到 x6 中
dct_long_mac(x6, sump23o, row3, rot3_2);
// 使用 dct_long_mac 函数将 sump14o 与 row1 进行长乘累加操作，并将结果存储到 x7 中
dct_long_mac(x7, sump14o, row1, rot3_3);
// 使用 dct_bfly32o 函数对 row0 和 row7 进行蝶形运算，并将结果存储到 x0 和 x7 中
dct_bfly32o(row0, row7, x0, x7, shiftop, shift);
// 使用 dct_bfly32o 函数对 row1 和 row6 进行蝶形运算，并将结果存储到 x1 和 x6 中
dct_bfly32o(row1, row6, x1, x6, shiftop, shift);
// 使用 dct_bfly32o 函数对 row2 和 row5 进行蝶形运算，并将结果存储到 x2 和 x5 中
dct_bfly32o(row2, row5, x2, x5, shiftop, shift);
// 使用 dct_bfly32o 函数对 row3 和 row4 进行蝶形运算，并将结果存储到 x3 和 x4 中
dct_bfly32o(row3, row4, x3, x4, shiftop, shift);

// 从指定地址加载数据到寄存器 row0
row0 = vld1q_s16(data + 0 * 8);
// 从指定地址加载数据到寄存器 row1
row1 = vld1q_s16(data + 1 * 8);
// 从指定地址加载数据到寄存器 row2
row2 = vld1q_s16(data + 2 * 8);
// 从指定地址加载数据到寄存器 row3
row3 = vld1q_s16(data + 3 * 8);
// 从指定地址加载数据到寄存器 row4
row4 = vld1q_s16(data + 4 * 8);
// 从指定地址加载数据到寄存器 row5
row5 = vld1q_s16(data + 5 * 8);
// 从指定地址加载数据到寄存器 row6
row6 = vld1q_s16(data + 6 * 8);
    // 从数据数组中读取第7行的16位整数数据，存储到row7变量中
    row7 = vld1q_s16(data + 7 * 8);

    // 添加直流偏置
    // 将row0和一个包含1024的向量进行逐元素相加
    row0 = vaddq_s16(row0, vsetq_lane_s16(1024, vdupq_n_s16(0), 0));

    // 列传递
    // 调用dct_pass函数，传递vrshrn_n_s32函数指针和10作为参数
    dct_pass(vrshrn_n_s32, 10);

    // 16位8x8转置
    {
        // 这三个宏分别映射到单个VTRN.16、VTRN.32和VSWP指令
        // 不幸的是，编译器是否真正理解这一点是另一回事
        // 定义一个宏，用于将两个16位整数向量进行转置
        #define dct_trn16(x, y)                                                                                                        \
        {                                                                                                                          \
            int16x8x2_t t = vtrnq_s16(x, y);                                                                                       \
            x = t.val[0];                                                                                                          \
            y = t.val[1];                                                                                                          \
        }
        // 定义一个宏，用于将两个32位整数向量进行转置
        #define dct_trn32(x, y)                                                                                                        \
        {                                                                                                                          \
// 将两个int16x8_t类型的向量x和y进行交错操作，并将结果存储回x和y中
#define dct_trn64(x, y)                                                                                                        \
    {                                                                                                                          \
        int16x8_t x0 = x;                                                                                                      \  // 将x向量存储到x0中
        int16x8_t y0 = y;                                                                                                      \  // 将y向量存储到y0中
        x = vcombine_s16(vget_low_s16(x0), vget_low_s16(y0));                                                                  \  // 将x0和y0的低位部分进行交错操作，并存储回x中
        y = vcombine_s16(vget_high_s16(x0), vget_high_s16(y0));                                                                \  // 将x0和y0的高位部分进行交错操作，并存储回y中
    }

// pass 1
dct_trn16(row0, row1); // a0b0a2b2a4b4a6b6  // 对row0和row1进行交错操作
dct_trn16(row2, row3);  // 对row2和row3进行交错操作
dct_trn16(row4, row5);  // 对row4和row5进行交错操作
dct_trn16(row6, row7);  // 对row6和row7进行交错操作

// pass 2
dct_trn32(row0, row2); // a0b0c0d0a4b4c4d4  // 对row0和row2进行交错操作
        // 对row1和row3进行32位离散余弦变换
        dct_trn32(row1, row3);
        // 对row4和row6进行32位离散余弦变换
        dct_trn32(row4, row6);
        // 对row5和row7进行32位离散余弦变换
        dct_trn32(row5, row7);

        // 第三轮变换
        // 对row0和row4进行64位离散余弦变换
        dct_trn64(row0, row4); // a0b0c0d0e0f0g0h0
        // 对row1和row5进行64位离散余弦变换
        dct_trn64(row1, row5);
        // 对row2和row6进行64位离散余弦变换
        dct_trn64(row2, row6);
        // 对row3和row7进行64位离散余弦变换
        dct_trn64(row3, row7);

        // 取消之前定义的宏
        #undef dct_trn16
        #undef dct_trn32
        #undef dct_trn64
    }

    // 行变换
    // vrshrn_n_s32只支持最多16位的移位，我们需要17位，所以先进行一个非四舍五入的16位移位，然后再进行一个1位的四舍五入移位
    dct_pass(vshrn_n_s32, 16);
    {
        // 对输入的16位有符号整数向右移动1位，并将结果转换为8位无符号整数
        uint8x8_t p0 = vqrshrun_n_s16(row0, 1);
        uint8x8_t p1 = vqrshrun_n_s16(row1, 1);
        uint8x8_t p2 = vqrshrun_n_s16(row2, 1);
        uint8x8_t p3 = vqrshrun_n_s16(row3, 1);
        uint8x8_t p4 = vqrshrun_n_s16(row4, 1);
        uint8x8_t p5 = vqrshrun_n_s16(row5, 1);
        uint8x8_t p6 = vqrshrun_n_s16(row6, 1);
        uint8x8_t p7 = vqrshrun_n_s16(row7, 1);

        // 再次，这些可以转换为一条指令，但通常不这样做。
#define dct_trn8_8(x, y)                                                                                                       \
    {                                                                                                                          \
        // 交错存储两个8位向量的元素
        uint8x8x2_t t = vtrn_u8(x, y);                                                                                         
        x = t.val[0];                                                                                                         
        y = t.val[1];                                                                                                         
    }
#define dct_trn8_16(x, y)                                                                                                      \
// 定义一个宏，用于对两个 uint16x4x2_t 类型的变量进行转置操作
{
    // 将两个 uint8x8_t 类型的变量转换为 uint16x4x2_t 类型，并进行转置操作
    uint16x4x2_t t = vtrn_u16(vreinterpret_u16_u8(x), vreinterpret_u16_u8(y));
    // 将转置后的结果重新转换为 uint8x8_t 类型的变量
    x = vreinterpret_u8_u16(t.val[0]);
    y = vreinterpret_u8_u16(t.val[1]);
}

// 定义一个宏，用于对两个 uint32x2x2_t 类型的变量进行转置操作
{
    // 将两个 uint8x8_t 类型的变量转换为 uint32x2x2_t 类型，并进行转置操作
    uint32x2x2_t t = vtrn_u32(vreinterpret_u32_u8(x), vreinterpret_u32_u8(y));
    // 将转置后的结果重新转换为 uint8x8_t 类型的变量
    x = vreinterpret_u8_u32(t.val[0]);
    y = vreinterpret_u8_u32(t.val[1]);
}

// 对8个变量进行8x8的8位转置操作
dct_trn8_8(p0, p1);
dct_trn8_8(p2, p3);
dct_trn8_8(p4, p5);
dct_trn8_8(p6, p7);
// pass 2
// 对输入的8个值进行DCT变换，并将结果存储在p0, p1, p2, p3, p4, p5, p6, p7中
dct_trn8_16(p0, p2);
dct_trn8_16(p1, p3);
dct_trn8_16(p4, p6);
dct_trn8_16(p5, p7);

// pass 3
// 对输入的8个值进行DCT变换，并将结果存储在p0, p1, p2, p3, p4, p5, p6, p7中
dct_trn8_32(p0, p4);
dct_trn8_32(p1, p5);
dct_trn8_32(p2, p6);
dct_trn8_32(p3, p7);

// store
// 将p0, p1, p2中的值存储到out指向的内存中，并更新out的位置
vst1_u8(out, p0);
out += out_stride;
// 将p1, p2, p3中的值存储到out指向的内存中，并更新out的位置
vst1_u8(out, p1);
out += out_stride;
// 将p2, p3, p4中的值存储到out指向的内存中，并更新out的位置
vst1_u8(out, p2);
out += out_stride;
# 将p3中的值存储到out指向的地址中，然后将out指针向后移动一个out_stride的距离
vst1_u8(out, p3);
out += out_stride;
# 将p4中的值存储到out指向的地址中，然后将out指针向后移动一个out_stride的距离
vst1_u8(out, p4);
out += out_stride;
# 将p5中的值存储到out指向的地址中，然后将out指针向后移动一个out_stride的距离
vst1_u8(out, p5);
out += out_stride;
# 将p6中的值存储到out指向的地址中，然后将out指针向后移动一个out_stride的距离
vst1_u8(out, p6);
out += out_stride;
# 将p7中的值存储到out指向的地址中
vst1_u8(out, p7);

# 取消之前定义的宏
#undef dct_trn8_8
#undef dct_trn8_16
#undef dct_trn8_32
# 取消之前定义的宏
#undef dct_long_mul
#undef dct_long_mac
#undef dct_widen
#undef dct_wadd
#undef dct_wsub
// 取消定义 dct_bfly32o
// 取消定义 dct_pass
#endif // STBI_NEON

// 定义标记值为0xff
#define STBI__MARKER_none 0xff
// 如果熵流中有待处理的标记，返回该标记；否则，从流中获取一个标记。如果没有标记，返回0xff，这是一个无效的标记值
static stbi_uc stbi__get_marker(stbi__jpeg * j) {
    stbi_uc x;
    if (j->marker != STBI__MARKER_none) {
        x = j->marker;
        j->marker = STBI__MARKER_none;
        return x;
    }
    x = stbi__get8(j->s);
    if (x != 0xff)
        return STBI__MARKER_none;
}
// 当 x 等于 0xff 时，消耗重复的 0xff 填充字节
while (x == 0xff)
    x = stbi__get8(j->s);
return x;

// 在每个扫描中，我们将有 scan_n 个分量，分量的顺序由 order[] 指定
#define STBI__RESTART(x) ((x) >= 0xd0 && (x) <= 0xd7)

// 在重启间隔之后，重置熵解码器和 DC 预测
static void stbi__jpeg_reset(stbi__jpeg * j) {
    j->code_bits = 0;
    j->code_buffer = 0;
    j->nomore = 0;
    j->img_comp[0].dc_pred = j->img_comp[1].dc_pred = j->img_comp[2].dc_pred = j->img_comp[3].dc_pred = 0;
    j->marker = STBI__MARKER_none;
    j->todo = j->restart_interval ? j->restart_interval : 0x7fffffff;
    j->eob_run = 0;
    // 如果没有 restart_interval，最多不超过 1<<31 个 MCU，这是非常安全的
}
    // 由于我们甚至不允许 1<<30 像素
}

static int stbi__parse_entropy_coded_data(stbi__jpeg * z) {
    // 重置 JPEG 对象
    stbi__jpeg_reset(z);
    // 如果不是渐进式扫描
    if (!z->progressive) {
        // 如果扫描的组件数为1
        if (z->scan_n == 1) {
            int i, j;
            // 创建一个 64 个 short 类型元素的数组
            STBI_SIMD_ALIGN(short, data[64]);
            int n = z->order[0];
            // 非交错数据，我们只需要一次处理一个块，在简单的扫描线顺序中
            // 要处理的块数取决于该组件实际具有的“像素”数量，与交错 MCU 阻塞等无关
            int w = (z->img_comp[n].x + 7) >> 3;
            int h = (z->img_comp[n].y + 7) >> 3;
            for (j = 0; j < h; ++j) {
                for (i = 0; i < w; ++i) {
                    int ha = z->img_comp[n].ha;
                    // 解码一个块
                    if (!stbi__jpeg_decode_block(z, data, z->huff_dc + z->img_comp[n].hd, z->huff_ac + ha, z->fast_ac[ha], n,
// 检查是否需要进行反量化和逆变换
if (z->img_comp[n].tq >= z->img_comp[n].num_quant_tables || z->img_comp[n].tq >= 4)
    return 0;
// 对图像数据进行逆变换
z->idct_block_kernel(z->img_comp[n].data + z->img_comp[n].w2 * j * 8 + i * 8, z->img_comp[n].w2, data);
// 每个数据块都是一个 MCU，所以减少重启间隔的计数
if (--z->todo <= 0) {
    if (z->code_bits < 24)
        stbi__grow_buffer_unsafe(z);
    // 如果不是重启标记，则直接返回1，以获取损坏的数据而不是没有数据
    if (!STBI__RESTART(z->marker))
        return 1;
    stbi__jpeg_reset(z);
}
// 遍历图像 MCU 的行和列
for (j = 0; j < z->img_mcu_y; ++j) {
// 遍历图像的 MCU 行
for (i = 0; i < z->img_mcu_x; ++i) {
    // 处理扫描中的每个分量
    for (k = 0; k < z->scan_n; ++k) {
        // 获取当前分量的索引
        int n = z->order[k];
        // 根据分量的 H 和 V 值确定扫描出一个 MCU 的数据
        for (y = 0; y < z->img_comp[n].v; ++y) {
            for (x = 0; x < z->img_comp[n].h; ++x) {
                // 计算当前像素在图像中的位置
                int x2 = (i * z->img_comp[n].h + x) * 8;
                int y2 = (j * z->img_comp[n].v + y) * 8;
                int ha = z->img_comp[n].ha;
                // 解码 JPEG 数据块
                if (!stbi__jpeg_decode_block(z, data, z->huff_dc + z->img_comp[n].hd, z->huff_ac + ha,
                                             z->fast_ac[ha], n, z->dequant[z->img_comp[n].tq]))
                    return 0;
                // 对解码后的数据进行反量化和反离散余弦变换
                z->idct_block_kernel(z->img_comp[n].data + z->img_comp[n].w2 * y2 + x2, z->img_comp[n].w2,
                                     data);
            }
        }
    }
    // 处理完所有交错的分量后，得到一个交错的 MCU
}
// 如果重启间隔计数减少到0以下，执行以下操作
if (--z->todo <= 0) {
    // 如果代码位数小于24，增加缓冲区大小
    if (z->code_bits < 24)
        stbi__grow_buffer_unsafe(z);
    // 如果重启标记不成功，返回1
    if (!STBI__RESTART(z->marker))
        return 1;
    // 重置JPEG解码器状态
    stbi__jpeg_reset(z);
}
// 返回1
return 1;
// 如果不是第一扫描，执行以下操作
} else {
    if (z->scan_n == 1) {
        int i, j;
        int n = z->order[0];
        // 非交错数据，一次处理一个块，按照简单的扫描线顺序
        // 需要处理的块数量取决于该分量实际的像素数量，与交错MCU块无关
            // 计算当前图像分量的宽度和高度
            int w = (z->img_comp[n].x + 7) >> 3;
            int h = (z->img_comp[n].y + 7) >> 3;
            // 遍历每个 MCU（Minimum Coded Unit）
            for (j = 0; j < h; ++j) {
                for (i = 0; i < w; ++i) {
                    // 计算当前 MCU 的数据指针
                    short * data = z->img_comp[n].coeff + 64 * (i + j * z->img_comp[n].coeff_w);
                    // 如果尚未开始解码特殊数据，则使用直流哈夫曼解码
                    if (z->spec_start == 0) {
                        if (!stbi__jpeg_decode_block_prog_dc(z, data, &z->huff_dc[z->img_comp[n].hd], n))
                            return 0;
                    } else {
                        // 否则，使用交流哈夫曼解码
                        int ha = z->img_comp[n].ha;
                        if (!stbi__jpeg_decode_block_prog_ac(z, data, &z->huff_ac[ha], z->fast_ac[ha]))
                            return 0;
                    }
                    // 每个数据块都是一个 MCU，所以倒计时重启间隔
                    if (--z->todo <= 0) {
                        // 如果代码位数小于 24，则扩展缓冲区
                        if (z->code_bits < 24)
                            stbi__grow_buffer_unsafe(z);
                        // 如果不是重启标记，则返回 1
                        if (!STBI__RESTART(z->marker))
                            return 1;
                        // 重置 JPEG 解码器状态
                        stbi__jpeg_reset(z);
// 如果扫描模式为非交错模式
if (!z->progressive) {
    // 返回 1，表示成功
    return 1;
} else { // 如果扫描模式为交错模式
    int i, j, k, x, y;
    // 遍历图像的 MCU 行
    for (j = 0; j < z->img_mcu_y; ++j) {
        // 遍历图像的 MCU 列
        for (i = 0; i < z->img_mcu_x; ++i) {
            // 扫描交错的 MCU... 按顺序处理 scan_n 个分量
            for (k = 0; k < z->scan_n; ++k) {
                int n = z->order[k];
                // 扫描出一个 MCU 的该分量的数据；这由该分量的基本 H 和 V 决定
                for (y = 0; y < z->img_comp[n].v; ++y) {
                    for (x = 0; x < z->img_comp[n].h; ++x) {
                        int x2 = (i * z->img_comp[n].h + x);
                        int y2 = (j * z->img_comp[n].v + y);
                        short * data = z->img_comp[n].coeff + 64 * (x2 + y2 * z->img_comp[n].coeff_w);
                        // 如果解码 DC 系数失败，则返回 0
                        if (!stbi__jpeg_decode_block_prog_dc(z, data, &z->huff_dc[z->img_comp[n].hd], n))
                            return 0;
// 如果所有交错的组件都处理完了，那么这是一个交错的 MCU，现在减少重启间隔
if (--z->todo <= 0) {
    // 如果代码位数小于24，增加缓冲区大小
    if (z->code_bits < 24)
        stbi__grow_buffer_unsafe(z);
    // 如果不是重启标记，返回1
    if (!STBI__RESTART(z->marker))
        return 1;
    // 重置 JPEG 解码器状态
    stbi__jpeg_reset(z);
}
    // 定义整型变量 i
    int i;
    // 循环遍历 data 数组的前 64 个元素
    for (i = 0; i < 64; ++i)
        // 将每个元素与 dequant 数组对应位置的元素相乘
        data[i] *= dequant[i];
}

static void stbi__jpeg_finish(stbi__jpeg * z) {
    // 如果是渐进式 JPEG
    if (z->progressive) {
        // 变量定义
        int i, j, n;
        // 遍历图像通道
        for (n = 0; n < z->s->img_n; ++n) {
            // 计算宽度和高度
            int w = (z->img_comp[n].x + 7) >> 3;
            int h = (z->img_comp[n].y + 7) >> 3;
            // 遍历每个 8x8 的块
            for (j = 0; j < h; ++j) {
                for (i = 0; i < w; ++i) {
                    // 获取当前块的数据
                    short * data = z->img_comp[n].coeff + 64 * (i + j * z->img_comp[n].coeff_w);
                    // 对数据进行反量化
                    stbi__jpeg_dequantize(data, z->dequant[z->img_comp[n].tq]);
                    // 对数据进行逆离散余弦变换
                    z->idct_block_kernel(z->img_comp[n].data + z->img_comp[n].w2 * j * 8 + i * 8, z->img_comp[n].w2, data);
                }
            }
        }
    }
}

static int stbi__process_marker(stbi__jpeg * z, int m) {
    int L; // 用于存储标记的长度
    switch (m) {
    case STBI__MARKER_none: // 没有找到标记
        return stbi__err("expected marker", "Corrupt JPEG"); // 返回错误信息

    case 0xDD: // DRI - 指定重启间隔
        if (stbi__get16be(z->s) != 4) // 获取下一个 16 位大端序的值，判断是否为 4
            return stbi__err("bad DRI len", "Corrupt JPEG"); // 返回错误信息
        z->restart_interval = stbi__get16be(z->s); // 将获取的值赋给 restart_interval
        return 1; // 返回 1

    case 0xDB: // DQT - 定义量化表
        L = stbi__get16be(z->s) - 2; // 获取标记的长度，并减去 2
        while (L > 0) { // 循环直到标记的长度为 0
            int q = stbi__get8(z->s); // 获取下一个 8 位的值
            int p = q >> 4, sixteen = (p != 0); // 对获取的值进行位运算
            // 从输入的 q 中取出低 4 位，赋值给 t
            int t = q & 15, i;
            // 如果 p 不等于 0 且不等于 1，则返回错误信息
            if (p != 0 && p != 1)
                return stbi__err("bad DQT type", "Corrupt JPEG");
            // 如果 t 大于 3，则返回错误信息
            if (t > 3)
                return stbi__err("bad DQT table", "Corrupt JPEG");

            // 遍历 64 个元素，将数据存入 z->dequant[t] 数组中
            for (i = 0; i < 64; ++i)
                z->dequant[t][stbi__jpeg_dezigzag[i]] = (stbi__uint16)(sixteen ? stbi__get16be(z->s) : stbi__get8(z->s));
            // 减去相应的长度
            L -= (sixteen ? 129 : 65);
        }
        // 返回 L 是否等于 0
        return L == 0;

    // 如果遇到 0xC4，则执行以下代码
    case 0xC4: // DHT - define huffman table
        // 读取 16 位大端序数据，并减去 2
        L = stbi__get16be(z->s) - 2;
        // 当 L 大于 0 时执行以下代码
        while (L > 0) {
            // 定义变量 v，sizes 数组，以及 i 和 n
            stbi_uc * v;
            int sizes[16], i, n = 0;
            // 从输入的 q 中取出高 4 位，赋值给 tc，取出低 4 位，赋值给 th
            int q = stbi__get8(z->s);
            int tc = q >> 4;
            int th = q & 15;
// 如果 tc 大于 1 或者 th 大于 3，则返回错误信息
if (tc > 1 || th > 3)
    return stbi__err("bad DHT header", "Corrupt JPEG");
// 遍历16次，读取每个大小值并累加到 n 上
for (i = 0; i < 16; ++i) {
    sizes[i] = stbi__get8(z->s);
    n += sizes[i];
}
// 如果 n 大于 256，则返回错误信息
if (n > 256)
    return stbi__err("bad DHT header", "Corrupt JPEG"); // Loop over i < n would write past end of values!
// 减去17，更新剩余长度
L -= 17;
// 如果 tc 等于 0，则构建直流哈夫曼表，否则构建交流哈夫曼表
if (tc == 0) {
    if (!stbi__build_huffman(z->huff_dc + th, sizes))
        return 0;
    v = z->huff_dc[th].values;
} else {
    if (!stbi__build_huffman(z->huff_ac + th, sizes))
        return 0;
    v = z->huff_ac[th].values;
}
// 读取 n 个值到 v 中
for (i = 0; i < n; ++i)
    v[i] = stbi__get8(z->s);
        // 如果 tc 不等于 0，则构建快速 AC 表
        if (tc != 0)
            stbi__build_fast_ac(z->fast_ac[th], z->huff_ac + th);
        // 减去已处理的字节数
        L -= n;
    }
    // 返回 L 是否等于 0
    return L == 0;
}

// 检查是否为注释块或者 APP 块
if ((m >= 0xE0 && m <= 0xEF) || m == 0xFE) {
    // 读取 16 位大端序的长度值
    L = stbi__get16be(z->s);
    // 如果长度小于 2，则返回错误
    if (L < 2) {
        if (m == 0xFE)
            return stbi__err("bad COM len", "Corrupt JPEG");
        else
            return stbi__err("bad APP len", "Corrupt JPEG");
    }
    // 减去已处理的字节数
    L -= 2;

    // 如果是 JFIF APP0 段且长度大于等于 5
    if (m == 0xE0 && L >= 5) {
        // 定义 JFIF APP0 段的标签
        static const unsigned char tag[5] = {'J', 'F', 'I', 'F', '\0'};
            // 初始化变量 ok 为 1
            int ok = 1;
            // 初始化变量 i
            int i;
            // 遍历 tag 数组，比较每个元素和输入流中的数据是否相等，如果不相等则将 ok 置为 0
            for (i = 0; i < 5; ++i)
                if (stbi__get8(z->s) != tag[i])
                    ok = 0;
            // 减去已经读取的字节数
            L -= 5;
            // 如果 ok 为真，则将 z->jfif 置为 1
            if (ok)
                z->jfif = 1;
        } else if (m == 0xEE && L >= 12) { // Adobe APP14 segment
            // 初始化 tag 数组
            static const unsigned char tag[6] = {'A', 'd', 'o', 'b', 'e', '\0'};
            // 初始化变量 ok 为 1
            int ok = 1;
            // 初始化变量 i
            int i;
            // 遍历 tag 数组，比较每个元素和输入流中的数据是否相等，如果不相等则将 ok 置为 0
            for (i = 0; i < 6; ++i)
                if (stbi__get8(z->s) != tag[i])
                    ok = 0;
            // 减去已经读取的字节数
            L -= 6;
            // 如果 ok 为真，则继续读取输入流中的数据
            if (ok) {
                stbi__get8(z->s);                            // version
                stbi__get16be(z->s);                         // flags0
                stbi__get16be(z->s);                         // flags1
                z->app14_color_transform = stbi__get8(z->s); // 从输入流中读取一个字节，赋值给app14_color_transform，表示颜色转换
                L -= 6; // 减去6，用于后续处理
            }
        }

        stbi__skip(z->s, L); // 跳过输入流中的L个字节
        return 1; // 返回1，表示处理成功
    }

    return stbi__err("unknown marker", "Corrupt JPEG"); // 返回错误信息，表示未知标记，JPEG文件损坏
}

// 在看到SOS标记后
static int stbi__process_scan_header(stbi__jpeg * z) {
    int i;
    int Ls = stbi__get16be(z->s); // 从输入流中读取两个字节，赋值给Ls
    z->scan_n = stbi__get8(z->s); // 从输入流中读取一个字节，赋值给scan_n，表示扫描的组件数
    if (z->scan_n < 1 || z->scan_n > 4 || z->scan_n > (int)z->s->img_n) // 判断扫描的组件数是否合法
        return stbi__err("bad SOS component count", "Corrupt JPEG"); // 返回错误信息，表示SOS组件数错误，JPEG文件损坏
    if (Ls != 6 + 2 * z->scan_n) // 判断Ls是否符合规定
    # 返回错误信息，指示SOS长度错误和JPEG文件损坏
    return stbi__err("bad SOS len", "Corrupt JPEG");
    # 遍历扫描的次数
    for (i = 0; i < z->scan_n; ++i) {
        # 读取id和which
        int id = stbi__get8(z->s), which;
        int q = stbi__get8(z->s);
        # 遍历图像通道数
        for (which = 0; which < z->s->img_n; ++which)
            # 如果找到对应的id，跳出循环
            if (z->img_comp[which].id == id)
                break;
        # 如果which等于图像通道数，表示没有匹配项
        if (which == z->s->img_n)
            return 0; # 没有匹配项
        # 设置图像通道的hd值
        z->img_comp[which].hd = q >> 4;
        # 如果hd值大于3，返回错误信息
        if (z->img_comp[which].hd > 3)
            return stbi__err("bad DC huff", "Corrupt JPEG");
        # 设置图像通道的ha值
        z->img_comp[which].ha = q & 15;
        # 如果ha值大于3，返回错误信息
        if (z->img_comp[which].ha > 3)
            return stbi__err("bad AC huff", "Corrupt JPEG");
        # 设置order数组的值
        z->order[i] = which;
    }

    {
        # 定义变量aa
        int aa;
        // 从输入流中读取一个字节，赋值给 spec_start
        z->spec_start = stbi__get8(z->s);
        // 从输入流中读取一个字节，赋值给 spec_end，应该是63，但可能是0
        z->spec_end = stbi__get8(z->s);
        // 从输入流中读取一个字节，赋值给 aa
        aa = stbi__get8(z->s);
        // 将 aa 右移4位，赋值给 succ_high
        z->succ_high = (aa >> 4);
        // 将 aa 与 15 进行与操作，赋值给 succ_low
        z->succ_low = (aa & 15);
        // 如果是渐进式扫描
        if (z->progressive) {
            // 如果 spec_start 大于63，spec_end 大于63，spec_start 大于 spec_end，succ_high 大于13，succ_low 大于13，返回错误
            if (z->spec_start > 63 || z->spec_end > 63 || z->spec_start > z->spec_end || z->succ_high > 13 || z->succ_low > 13)
                return stbi__err("bad SOS", "Corrupt JPEG");
        } else {
            // 如果 spec_start 不等于0，返回错误
            if (z->spec_start != 0)
                return stbi__err("bad SOS", "Corrupt JPEG");
            // 如果 succ_high 不等于0 或 succ_low 不等于0，返回错误
            if (z->succ_high != 0 || z->succ_low != 0)
                return stbi__err("bad SOS", "Corrupt JPEG");
            // 将 spec_end 赋值为63
            z->spec_end = 63;
        }
    }

    // 返回1
    return 1;
}
// 释放 JPEG 图像组件的内存
static int stbi__free_jpeg_components(stbi__jpeg * z, int ncomp, int why) {
    int i;
    for (i = 0; i < ncomp; ++i) {
        // 如果原始数据存在，则释放内存并将指针置为空
        if (z->img_comp[i].raw_data) {
            STBI_FREE(z->img_comp[i].raw_data);
            z->img_comp[i].raw_data = NULL;
            z->img_comp[i].data = NULL;
        }
        // 如果原始系数存在，则释放内存并将指针置为0
        if (z->img_comp[i].raw_coeff) {
            STBI_FREE(z->img_comp[i].raw_coeff);
            z->img_comp[i].raw_coeff = 0;
            z->img_comp[i].coeff = 0;
        }
        // 如果行缓冲存在，则释放内存并将指针置为空
        if (z->img_comp[i].linebuf) {
            STBI_FREE(z->img_comp[i].linebuf);
            z->img_comp[i].linebuf = NULL;
        }
    }
    // 返回释放原因
    return why;
}
static int stbi__process_frame_header(stbi__jpeg * z, int scan) {
    // 获取 JPEG 上下文
    stbi__context * s = z->s;
    int Lf, p, i, q, h_max = 1, v_max = 1, c;
    // 读取帧头部长度
    Lf = stbi__get16be(s);
    // 如果长度小于11，返回错误
    if (Lf < 11)
        return stbi__err("bad SOF len", "Corrupt JPEG"); // JPEG
    // 读取精度
    p = stbi__get8(s);
    // 如果精度不是8位，返回错误
    if (p != 8)
        return stbi__err("only 8-bit", "JPEG format not supported: 8-bit only"); // JPEG baseline
    // 读取图像高度
    s->img_y = stbi__get16be(s);
    // 如果高度为0，返回错误
    if (s->img_y == 0)
        return stbi__err("no header height",
                         "JPEG format not supported: delayed height"); // Legal, but we don't handle it--but neither does IJG
    // 读取图像宽度
    s->img_x = stbi__get16be(s);
    // 如果宽度为0，返回错误
    if (s->img_x == 0)
        return stbi__err("0 width", "Corrupt JPEG"); // JPEG requires
    // 如果图像高度超过最大尺寸限制，返回错误
    if (s->img_y > STBI_MAX_DIMENSIONS)
        return stbi__err("too large", "Very large image (corrupt?)");
    // 如果图像宽度超过最大尺寸限制，返回错误
    if (s->img_x > STBI_MAX_DIMENSIONS)
        # 返回错误信息，表示图像太大，可能损坏
        return stbi__err("too large", "Very large image (corrupt?)");
    # 从输入流中读取一个字节，赋值给变量c
    c = stbi__get8(s);
    # 如果c不等于3、1或4，返回错误信息，表示组件数量不正确，JPEG可能损坏
    if (c != 3 && c != 1 && c != 4)
        return stbi__err("bad component count", "Corrupt JPEG");
    # 将c赋值给s的img_n属性
    s->img_n = c;
    # 循环遍历每个组件
    for (i = 0; i < c; ++i) {
        # 将img_comp[i]的data和linebuf属性置为NULL
        z->img_comp[i].data = NULL;
        z->img_comp[i].linebuf = NULL;
    }

    # 如果Lf不等于8加上3乘以s的img_n，返回错误信息，表示SOF长度不正确，JPEG可能损坏
    if (Lf != 8 + 3 * s->img_n)
        return stbi__err("bad SOF len", "Corrupt JPEG");

    # 将rgb属性置为0
    z->rgb = 0;
    # 循环遍历每个组件
    for (i = 0; i < s->img_n; ++i) {
        # 创建一个静态的包含'R'、'G'、'B'的无符号字符数组
        static const unsigned char rgb[3] = {'R', 'G', 'B'};
        # 从输入流中读取一个字节，赋值给img_comp[i]的id属性
        z->img_comp[i].id = stbi__get8(s);
        # 如果img_comp[i]的id等于rgb[i]，并且img_n等于3，增加rgb属性的值
        if (s->img_n == 3 && z->img_comp[i].id == rgb[i])
            ++z->rgb;
        # 从输入流中读取一个字节，赋值给变量q
        q = stbi__get8(s);
        # 设置图像组件的水平采样因子
        z->img_comp[i].h = (q >> 4);
        # 如果水平采样因子为0或大于4，则返回错误信息
        if (!z->img_comp[i].h || z->img_comp[i].h > 4)
            return stbi__err("bad H", "Corrupt JPEG");
        # 设置图像组件的垂直采样因子
        z->img_comp[i].v = q & 15;
        # 如果垂直采样因子为0或大于4，则返回错误信息
        if (!z->img_comp[i].v || z->img_comp[i].v > 4)
            return stbi__err("bad V", "Corrupt JPEG");
        # 设置图像组件的量化表
        z->img_comp[i].tq = stbi__get8(s);
        # 如果量化表大于3，则返回错误信息
        if (z->img_comp[i].tq > 3)
            return stbi__err("bad TQ", "Corrupt JPEG");
    }

    # 如果不是加载扫描模式，则返回1
    if (scan != STBI__SCAN_load)
        return 1;

    # 如果图像尺寸过大，则返回错误信息
    if (!stbi__mad3sizes_valid(s->img_x, s->img_y, s->img_n, 0))
        return stbi__err("too large", "Image too large to decode");

    # 遍历图像组件，找出最大的水平采样因子
    for (i = 0; i < s->img_n; ++i) {
        if (z->img_comp[i].h > h_max)
            h_max = z->img_comp[i].h;
    // 如果当前组件的垂直采样因子大于v_max，则将v_max更新为当前组件的垂直采样因子
    if (z->img_comp[i].v > v_max)
        v_max = z->img_comp[i].v;
    }

    // 检查平面下采样因子是否为整数比率；我们的重采样器无法处理分数比率
    // 而且我从未见过一个非损坏的JPEG文件实际上使用它们
    for (i = 0; i < s->img_n; ++i) {
        // 如果最大水平采样因子不能整除当前组件的水平采样因子，则返回错误信息
        if (h_max % z->img_comp[i].h != 0)
            return stbi__err("bad H", "Corrupt JPEG");
        // 如果最大垂直采样因子不能整除当前组件的垂直采样因子，则返回错误信息
        if (v_max % z->img_comp[i].v != 0)
            return stbi__err("bad V", "Corrupt JPEG");
    }

    // 计算交错式MCU信息
    z->img_h_max = h_max;
    z->img_v_max = v_max;
    z->img_mcu_w = h_max * 8;
    z->img_mcu_h = v_max * 8;
    // 这些尺寸不能超过17位
    z->img_mcu_x = (s->img_x + z->img_mcu_w - 1) / z->img_mcu_w;
    // 计算图像 MCU 的垂直方向数量
    z->img_mcu_y = (s->img_y + z->img_mcu_h - 1) / z->img_mcu_h;

    for (i = 0; i < s->img_n; ++i) {
        // 计算每个颜色分量的水平位置
        z->img_comp[i].x = (s->img_x * z->img_comp[i].h + h_max - 1) / h_max;
        // 计算每个颜色分量的垂直位置
        z->img_comp[i].y = (s->img_y * z->img_comp[i].v + v_max - 1) / v_max;
        // 分配足够的内存来解码数据，以便处理非常大的数据
        // img_mcu_x, img_mcu_y: <=17 bits; comp[i].h and .v are <=4 (checked earlier)
        // so these muls can't overflow with 32-bit ints (which we require)
        // 计算每个颜色分量的宽度和高度
        z->img_comp[i].w2 = z->img_mcu_x * z->img_comp[i].h * 8;
        z->img_comp[i].h2 = z->img_mcu_y * z->img_comp[i].v * 8;
        // 初始化一些变量
        z->img_comp[i].coeff = 0;
        z->img_comp[i].raw_coeff = 0;
        z->img_comp[i].linebuf = NULL;
        // 分配内存来存储原始数据
        z->img_comp[i].raw_data = stbi__malloc_mad2(z->img_comp[i].w2, z->img_comp[i].h2, 15);
        // 检查内存分配是否成功
        if (z->img_comp[i].raw_data == NULL)
        // 释放 JPEG 组件的内存，并返回错误信息
        return stbi__free_jpeg_components(z, i + 1, stbi__err("outofmem", "Out of memory"));
        // 使用 mmx/sse 对齐块以进行 idct
        z->img_comp[i].data = (stbi_uc *)(((size_t)z->img_comp[i].raw_data + 15) & ~15);
        if (z->progressive) {
            // w2, h2 是 8 的倍数（见上文）
            z->img_comp[i].coeff_w = z->img_comp[i].w2 / 8;
            z->img_comp[i].coeff_h = z->img_comp[i].h2 / 8;
            // 分配内存给原始系数
            z->img_comp[i].raw_coeff = stbi__malloc_mad3(z->img_comp[i].w2, z->img_comp[i].h2, sizeof(short), 15);
            if (z->img_comp[i].raw_coeff == NULL)
                // 如果分配内存失败，释放 JPEG 组件的内存，并返回错误信息
                return stbi__free_jpeg_components(z, i + 1, stbi__err("outofmem", "Out of memory"));
            // 对齐系数以进行 idct
            z->img_comp[i].coeff = (short *)(((size_t)z->img_comp[i].raw_coeff + 15) & ~15);
        }
    }

    return 1;
}

// 使用比较，因为在某些情况下我们处理多个情况（例如 SOF）
#define stbi__DNL(x) ((x) == 0xdc)
#define stbi__SOI(x) ((x) == 0xd8)
// 定义判断是否为结束图像标记的宏
#define stbi__EOI(x) ((x) == 0xd9)
// 定义判断是否为帧标记的宏
#define stbi__SOF(x) ((x) == 0xc0 || (x) == 0xc1 || (x) == 0xc2)
// 定义判断是否为扫描开始标记的宏
#define stbi__SOS(x) ((x) == 0xda)

// 定义判断是否为渐进式扫描帧标记的宏
#define stbi__SOF_progressive(x) ((x) == 0xc2)

// 解码 JPEG 头部信息的函数
static int stbi__decode_jpeg_header(stbi__jpeg * z, int scan) {
    int m;
    z->jfif = 0; // 初始化 JFIF 标记为 0
    z->app14_color_transform = -1; // 初始化应用程序标记14的颜色转换值为-1，有效值为0,1,2
    z->marker = STBI__MARKER_none; // 初始化缓存的标记为空
    m = stbi__get_marker(z); // 获取标记
    if (!stbi__SOI(m)) // 如果不是起始标记，则返回错误
        return stbi__err("no SOI", "Corrupt JPEG");
    if (scan == STBI__SCAN_type) // 如果扫描类型为指定类型，则返回1
        return 1;
    m = stbi__get_marker(z); // 获取标记
    while (!stbi__SOF(m)) { // 循环直到找到帧标记
        if (!stbi__process_marker(z, m)) // 处理标记，如果失败则返回0
            return 0;
        m = stbi__get_marker(z);
        // 循环直到找到有效的标记
        while (m == STBI__MARKER_none) {
            // 一些文件在块之后有额外的填充，因此我们需要扫描
            if (stbi__at_eof(z->s))
                return stbi__err("no SOF", "Corrupt JPEG");
            m = stbi__get_marker(z);
        }
    }
    // 设置是否为渐进式扫描
    z->progressive = stbi__SOF_progressive(m);
    // 处理帧头信息
    if (!stbi__process_frame_header(z, scan))
        return 0;
    return 1;
}

static int stbi__skip_jpeg_junk_at_end(stbi__jpeg * j) {
    // 一些 JPEG 文件在末尾有垃圾数据，跳过它们，但如果找到类似有效标记的内容，则从那里继续
    while (!stbi__at_eof(j->s)) {
        int x = stbi__get8(j->s);
        while (x == 255) { // 可能是一个标记
            // 检查是否已经到达文件末尾，如果是则返回标记为none
            if (stbi__at_eof(j->s))
                return STBI__MARKER_none;
            // 读取一个字节作为标记
            x = stbi__get8(j->s);
            // 如果不是填充的零或者另一个标记的前导，看起来像是一个实际的标记，返回它
            if (x != 0x00 && x != 0xff) {
                return x;
            }
            // 填充的零现在 x=0，结束循环，意味着回到常规扫描循环
            // 重复的 0xff 继续尝试读取标记的下一个字节
        }
    }
    // 如果没有找到标记，返回标记为none
    return STBI__MARKER_none;
}

// 将图像解码为YCbCr格式
static int stbi__decode_jpeg_image(stbi__jpeg * j) {
    int m;
    // 循环处理图像的每个部分
    for (m = 0; m < 4; m++) {
        // 将图像组件的原始数据和原始系数设置为NULL
        j->img_comp[m].raw_data = NULL;
        j->img_comp[m].raw_coeff = NULL;
    }
    // 重置间隔为0
    j->restart_interval = 0;
    // 解码 JPEG 头部信息
    if (!stbi__decode_jpeg_header(j, STBI__SCAN_load))
        return 0;
    // 获取标记
    m = stbi__get_marker(j);
    // 循环直到遇到结束标记
    while (!stbi__EOI(m)) {
        // 如果是扫描开始标记
        if (stbi__SOS(m)) {
            // 处理扫描头部信息
            if (!stbi__process_scan_header(j))
                return 0;
            // 解析熵编码数据
            if (!stbi__parse_entropy_coded_data(j))
                return 0;
            // 如果标记为none，则跳过 JPEG 末尾的垃圾数据
            if (j->marker == STBI__MARKER_none) {
                j->marker = stbi__skip_jpeg_junk_at_end(j);
                // 如果在没有遇到标记的情况下到达文件末尾，stbi__get_marker() 将失败，最终返回0
            }
            // 获取下一个标记
            m = stbi__get_marker(j);
            // 如果是重启标记，则再次获取标记
            if (STBI__RESTART(m))
                m = stbi__get_marker(j);
// 如果遇到DNL标记，读取长度和高度信息，并进行校验
} else if (stbi__DNL(m)) {
    int Ld = stbi__get16be(j->s); // 读取DNL标记的长度信息
    stbi__uint32 NL = stbi__get16be(j->s); // 读取DNL标记的高度信息
    if (Ld != 4) // 如果长度信息不等于4，返回错误信息
        return stbi__err("bad DNL len", "Corrupt JPEG");
    if (NL != j->s->img_y) // 如果高度信息不等于图像的高度，返回错误信息
        return stbi__err("bad DNL height", "Corrupt JPEG");
    m = stbi__get_marker(j); // 获取下一个标记
} else {
    if (!stbi__process_marker(j, m)) // 处理其他标记
        return 1; // 如果处理失败，返回错误
    m = stbi__get_marker(j); // 获取下一个标记
}
}
if (j->progressive) // 如果是渐进式扫描，完成JPEG解码
    stbi__jpeg_finish(j);
return 1; // 返回成功
}

// 静态的JFIF中心重采样（跨块边界）
// 定义一个指向函数的指针，该函数用于重采样行数据
typedef stbi_uc * (*resample_row_func)(stbi_uc * out, stbi_uc * in0, stbi_uc * in1, int w, int hs);

// 定义一个宏，用于将输入除以4
#define stbi__div4(x) ((stbi_uc)((x) >> 2))

// 重采样函数，将输入的近似值直接返回
static stbi_uc * resample_row_1(stbi_uc * out, stbi_uc * in_near, stbi_uc * in_far, int w, int hs) {
    STBI_NOTUSED(out);  // 不使用输出参数
    STBI_NOTUSED(in_far);  // 不使用远端输入参数
    STBI_NOTUSED(w);  // 不使用宽度参数
    STBI_NOTUSED(hs);  // 不使用hs参数
    return in_near;  // 直接返回近端输入参数
}

// 重采样函数，用于在垂直方向上生成两个样本
static stbi_uc * stbi__resample_row_v_2(stbi_uc * out, stbi_uc * in_near, stbi_uc * in_far, int w, int hs) {
    // 需要在输入的每个样本上垂直生成两个样本
    int i;
    STBI_NOTUSED(hs);  // 不使用hs参数
    for (i = 0; i < w; ++i)
        out[i] = stbi__div4(3 * in_near[i] + in_far[i] + 2);  // 对输入进行加权平均并除以4，将结果存入输出数组
    return out;  // 返回输出数组
}
// 对输入的像素数据进行水平重采样，生成两个输出样本
static stbi_uc * stbi__resample_row_h_2(stbi_uc * out, stbi_uc * in_near, stbi_uc * in_far, int w, int hs) {
    // 需要为每个输入样本水平生成两个样本
    int i;
    stbi_uc * input = in_near;

    if (w == 1) {
        // 如果只有一个样本，无法进行任何插值
        out[0] = out[1] = input[0];
        return out;
    }

    out[0] = input[0];
    out[1] = stbi__div4(input[0] * 3 + input[1] + 2);
    for (i = 1; i < w - 1; ++i) {
        int n = 3 * input[i] + 2;
        out[i * 2 + 0] = stbi__div4(n + input[i - 1]);
        out[i * 2 + 1] = stbi__div4(n + input[i + 1]);
    }
    # 将输入数组中的值进行处理后存入输出数组中
    out[i * 2 + 0] = stbi__div4(input[w - 2] * 3 + input[w - 1] + 2);
    out[i * 2 + 1] = input[w - 1];

    # 标记变量in_far和hs未使用
    STBI_NOTUSED(in_far);
    STBI_NOTUSED(hs);

    # 返回输出数组
    return out;
}

# 定义宏stbi__div16，用于对输入值进行位移操作
#define stbi__div16(x) ((stbi_uc)((x) >> 4))

# 定义函数stbi__resample_row_hv_2，用于对输入数组进行2x2采样
static stbi_uc * stbi__resample_row_hv_2(stbi_uc * out, stbi_uc * in_near, stbi_uc * in_far, int w, int hs) {
    # 需要为每个输入值生成2x2的样本
    int i, t0, t1;
    if (w == 1) {
        # 如果输入数组长度为1，则对输入值进行处理后存入输出数组中
        out[0] = out[1] = stbi__div4(3 * in_near[0] + in_far[0] + 2);
        return out;
    }

    # 对输入数组的第一个值进行处理
    t1 = 3 * in_near[0] + in_far[0];
    # 将第一个输出像素值设为输入像素值除以4
    out[0] = stbi__div4(t1 + 2);
    # 遍历每个输入像素
    for (i = 1; i < w; ++i) {
        # 保存上一个输入像素值
        t0 = t1;
        # 计算当前输入像素值
        t1 = 3 * in_near[i] + in_far[i];
        # 计算输出像素值，使用上一个和当前输入像素值
        out[i * 2 - 1] = stbi__div16(3 * t0 + t1 + 8);
        out[i * 2] = stbi__div16(3 * t1 + t0 + 8);
    }
    # 将最后一个输出像素值设为输入像素值除以4
    out[w * 2 - 1] = stbi__div4(t1 + 2);

    # 标记变量未使用
    STBI_NOTUSED(hs);

    # 返回输出像素数组
    return out;
}

#if defined(STBI_SSE2) || defined(STBI_NEON)
static stbi_uc * stbi__resample_row_hv_2_simd(stbi_uc * out, stbi_uc * in_near, stbi_uc * in_far, int w, int hs) {
    # 需要为每个输入像素生成2x2的样本
    int i = 0, t0, t1;

    # 如果输入像素宽度为1
    if (w == 1) {
        // 将输出数组的第一个和第二个元素赋值为 (3 * in_near[0] + in_far[0] + 2) 的四分之一
        out[0] = out[1] = stbi__div4(3 * in_near[0] + in_far[0] + 2);
        // 返回输出数组
        return out;
    }

    // 计算 t1 的值
    t1 = 3 * in_near[0] + in_far[0];
    // 对每组8个像素进行处理，直到不能再处理为止
    // 注意：在这个循环中，我们不能处理行中的最后一个像素，因为我们需要处理滤波器的边界条件。
    for (; i < ((w - 1) & ~7); i += 8) {
#if defined(STBI_SSE2)
        // 加载并执行垂直滤波处理
        // 这里使用了 3*x + y = 4*x + (y - x) 的公式
        __m128i zero = _mm_setzero_si128();  // 将 zero 初始化为全0的128位整数
        __m128i farb = _mm_loadl_epi64((__m128i *)(in_far + i));  // 从 in_far 数组中加载 64 位数据到 farb
        __m128i nearb = _mm_loadl_epi64((__m128i *)(in_near + i));  // 从 in_near 数组中加载 64 位数据到 nearb
        __m128i farw = _mm_unpacklo_epi8(farb, zero);  // 将 farb 和 zero 进行无符号字节整数扩展，存储到 farw
        __m128i nearw = _mm_unpacklo_epi8(nearb, zero);  // 将 nearb 和 zero 进行无符号字节整数扩展，存储到 nearw
        __m128i diff = _mm_sub_epi16(farw, nearw);  // 计算 farw 和 nearw 的差值，存储到 diff
        __m128i nears = _mm_slli_epi16(nearw, 2);  // 将 nearw 左移16位，存储到 nears
        __m128i curr = _mm_add_epi16(nears, diff);  // 计算 nears 和 diff 的和，存储到 curr，表示当前行
// 水平滤波器基于当前行的偏移版本工作。"prev"是当前行向右移动1像素得到的结果；我们需要插入前一个像素的值（来自t1）。
// "next"是当前行向左移动1像素得到的结果，同时加入了下一个8像素块的第一个像素。
__m128i prv0 = _mm_slli_si128(curr, 2); // 将当前行左移2个字节，得到prev0
__m128i nxt0 = _mm_srli_si128(curr, 2); // 将当前行右移2个字节，得到nxt0
__m128i prev = _mm_insert_epi16(prv0, t1, 0); // 在prev0中插入t1，得到prev
__m128i next = _mm_insert_epi16(nxt0, 3 * in_near[i + 8] + in_far[i + 8], 7); // 在nxt0中插入指定的值，得到next

// 水平滤波器，多相实现，因为它很方便：
// 偶数像素 = 3*cur + prev = cur*4 + (prev - cur)
// 奇数像素 = 3*cur + next = cur*4 + (next - cur)
// 注意共享的项。
__m128i bias = _mm_set1_epi16(8); // 创建一个所有元素都是8的向量
__m128i curs = _mm_slli_epi16(curr, 2); // 将当前行左移2位，得到curs
__m128i prvd = _mm_sub_epi16(prev, curr); // 计算prev和curr的差，得到prvd
__m128i nxtd = _mm_sub_epi16(next, curr); // 计算next和curr的差，得到nxtd
__m128i curb = _mm_add_epi16(curs, bias); // 将curs和bias相加，得到curb
        // 计算偶数像素和
        __m128i even = _mm_add_epi16(prvd, curb);
        // 计算奇数像素和
        __m128i odd = _mm_add_epi16(nxtd, curb);

        // 交错排列偶数和奇数像素，然后取消缩放
        __m128i int0 = _mm_unpacklo_epi16(even, odd);
        __m128i int1 = _mm_unpackhi_epi16(even, odd);
        __m128i de0 = _mm_srli_epi16(int0, 4);
        __m128i de1 = _mm_srli_epi16(int1, 4);

        // 打包并写入输出
        __m128i outv = _mm_packus_epi16(de0, de1);
        _mm_storeu_si128((__m128i *)(out + i * 2), outv);
#elif defined(STBI_NEON)
        // 加载并执行垂直滤波处理
        // 这里使用的是 3*x + y = 4*x + (y - x) 的公式
        uint8x8_t farb = vld1_u8(in_far + i);
        uint8x8_t nearb = vld1_u8(in_near + i);
        int16x8_t diff = vreinterpretq_s16_u16(vsubl_u8(farb, nearb));
        int16x8_t nears = vreinterpretq_s16_u16(vshll_n_u8(nearb, 2));
        int16x8_t curr = vaddq_s16(nears, diff); // 当前行
// 水平滤波器基于当前行的偏移版本工作。"prev"是当前行向右移动1像素得到的结果；我们需要插入前一个像素的值（来自t1）。
// "next"是当前行向左移动1像素得到的结果，同时加入下一个8像素块的第一个像素。
int16x8_t prv0 = vextq_s16(curr, curr, 7); // 从当前向量中提取第7个元素，构成新的向量
int16x8_t nxt0 = vextq_s16(curr, curr, 1); // 从当前向量中提取第1个元素，构成新的向量
int16x8_t prev = vsetq_lane_s16(t1, prv0, 0); // 将t1的值插入到prv0向量的第0个位置，构成新的向量
int16x8_t next = vsetq_lane_s16(3 * in_near[i + 8] + in_far[i + 8], nxt0, 7); // 将指定的值插入到nxt0向量的第7个位置，构成新的向量

// 水平滤波器，采用多相实现，因为这样更方便：
// 偶数像素 = 3*cur + prev = cur*4 + (prev - cur)
// 奇数像素 = 3*cur + next = cur*4 + (next - cur)
// 注意共享的项。
int16x8_t curs = vshlq_n_s16(curr, 2); // 将当前向量中的每个元素左移2位，构成新的向量
int16x8_t prvd = vsubq_s16(prev, curr); // 计算prev和curr之间的差值，构成新的向量
int16x8_t nxtd = vsubq_s16(next, curr); // 计算next和curr之间的差值，构成新的向量
int16x8_t even = vaddq_s16(curs, prvd); // 计算偶数像素的值，构成新的向量
int16x8_t odd = vaddq_s16(curs, nxtd); // 计算奇数像素的值，构成新的向量
        // 取消缩放和四舍五入，然后交错存储偶数/奇数相位
        uint8x8x2_t o;
        o.val[0] = vqrshrun_n_s16(even, 4); // 对偶数进行右移4位并转换为无符号8位整数
        o.val[1] = vqrshrun_n_s16(odd, 4); // 对奇数进行右移4位并转换为无符号8位整数
        vst2_u8(out + i * 2, o); // 将o中的值存储到out数组中，每隔一个位置存储一次

        // 下一次迭代的“前一个”值
        t1 = 3 * in_near[i + 7] + in_far[i + 7]; // 计算下一次迭代的t1值
    }

    t0 = t1; // 将t1的值赋给t0
    t1 = 3 * in_near[i] + in_far[i]; // 计算t1的新值
    out[i * 2] = stbi__div16(3 * t1 + t0 + 8); // 将计算结果存储到out数组中

    for (++i; i < w; ++i) { // 循环开始
        t0 = t1; // 将t1的值赋给t0
        t1 = 3 * in_near[i] + in_far[i]; // 计算t1的新值
        out[i * 2 - 1] = stbi__div16(3 * t0 + t1 + 8); // 将计算结果存储到out数组中
// 将输入的数据进行重新采样，使用最近邻插值算法
static stbi_uc * stbi__resample_row_generic(stbi_uc * out, stbi_uc * in_near, stbi_uc * in_far, int w, int hs) {
    int i, j;
    // 未使用参数in_far
    STBI_NOTUSED(in_far);
    // 遍历输出数据的每个像素
    for (i = 0; i < w; ++i)
        // 对每个像素进行水平方向的重复采样
        for (j = 0; j < hs; ++j)
            out[i * hs + j] = in_near[i];
    // 返回重采样后的数据
    return out;
}
// 这是一个降低精度的 YCbCr 到 RGB 转换计算，用于确保代码在 SIMD 和标量计算中产生相同的结果
// 将浮点数转换为定点数，乘以 4096 并四舍五入
#define stbi__float2fixed(x) (((int)((x)*4096.0f + 0.5f)) << 8)
// 将 YCbCr 转换为 RGB，处理一行像素数据
static void stbi__YCbCr_to_RGB_row(stbi_uc * out, const stbi_uc * y, const stbi_uc * pcb, const stbi_uc * pcr, int count,
                                   int step) {
    int i;
    for (i = 0; i < count; ++i) {
        // 将 Y 值左移 20 位并加上 1<<19，进行四舍五入
        int y_fixed = (y[i] << 20) + (1 << 19); // rounding
        int r, g, b;
        int cr = pcr[i] - 128;
        int cb = pcb[i] - 128;
        // 计算 R、G、B 值
        r = y_fixed + cr * stbi__float2fixed(1.40200f);
        g = y_fixed + (cr * -stbi__float2fixed(0.71414f)) + ((cb * -stbi__float2fixed(0.34414f)) & 0xffff0000);
        b = y_fixed + cb * stbi__float2fixed(1.77200f);
        // 将结果右移 20 位，得到最终的 R、G、B 值
        r >>= 20;
        g >>= 20;
        b >>= 20;
        // 如果 R 值超出范围 [0, 255]，则进行修正
        if ((unsigned)r > 255) {
            if (r < 0)
                r = 0;
        // 如果 r 大于 255，则将 r 设为 255；如果 r 小于 0，则将 r 设为 0
        if ((unsigned)r > 255) {
            if (r < 0)
                r = 0;
            else
                r = 255;
        }
        // 如果 g 大于 255，则将 g 设为 255；如果 g 小于 0，则将 g 设为 0
        if ((unsigned)g > 255) {
            if (g < 0)
                g = 0;
            else
                g = 255;
        }
        // 如果 b 大于 255，则将 b 设为 255；如果 b 小于 0，则将 b 设为 0
        if ((unsigned)b > 255) {
            if (b < 0)
                b = 0;
            else
                b = 255;
        }
        // 将 r、g、b 分别转换为无符号字符类型，并赋值给输出数组的前三个元素
        out[0] = (stbi_uc)r;
        out[1] = (stbi_uc)g;
        out[2] = (stbi_uc)b;
        // 输出数组的第四个元素设为 255
        out[3] = 255;
        // 输出数组指针向前移动 step 个位置
        out += step;
    }
}

#if defined(STBI_SSE2) || defined(STBI_NEON)
static void stbi__YCbCr_to_RGB_simd(stbi_uc * out, stbi_uc const * y, stbi_uc const * pcb, stbi_uc const * pcr, int count,
                                    int step) {
    int i = 0;

#ifdef STBI_SSE2
    // 如果 step 等于 4，则执行下面的代码
    if (step == 4) {
        // 这是一个相当直接的实现，不是特别优化。
        // 设置一个 8 位整数的向量，所有元素都是 -128
        __m128i signflip = _mm_set1_epi8(-0x80);
        // 设置一个 16 位整数的向量，所有元素都是 1.40200 * 4096
        __m128i cr_const0 = _mm_set1_epi16((short)(1.40200f * 4096.0f + 0.5f));
        // 设置一个 16 位整数的向量，所有元素都是 -0.71414 * 4096
        __m128i cr_const1 = _mm_set1_epi16(-(short)(0.71414f * 4096.0f + 0.5f));
        // 设置一个 16 位整数的向量，所有元素都是 -0.34414 * 4096
        __m128i cb_const0 = _mm_set1_epi16(-(short)(0.34414f * 4096.0f + 0.5f));
        // 设置一个 16 位整数的向量，所有元素都是 1.77200 * 4096
        __m128i cb_const1 = _mm_set1_epi16((short)(1.77200f * 4096.0f + 0.5f));
        // 设置一个 8 位整数的向量，所有元素都是 128
        __m128i y_bias = _mm_set1_epi8((char)(unsigned char)128);
        __m128i xw = _mm_set1_epi16(255); // 创建一个包含 255 的 16 位整数向量，用作 alpha 通道

        for (; i + 7 < count; i += 8) {
            // 加载数据
            __m128i y_bytes = _mm_loadl_epi64((__m128i *)(y + i)); // 从地址 y + i 处加载 8 个字节到 128 位整数向量 y_bytes
            __m128i cr_bytes = _mm_loadl_epi64((__m128i *)(pcr + i)); // 从地址 pcr + i 处加载 8 个字节到 128 位整数向量 cr_bytes
            __m128i cb_bytes = _mm_loadl_epi64((__m128i *)(pcb + i)); // 从地址 pcb + i 处加载 8 个字节到 128 位整数向量 cb_bytes
            __m128i cr_biased = _mm_xor_si128(cr_bytes, signflip); // 对 cr_bytes 中的每个字节进行按位异或操作，实现减 128 的效果
            __m128i cb_biased = _mm_xor_si128(cb_bytes, signflip); // 对 cb_bytes 中的每个字节进行按位异或操作，实现减 128 的效果

            // 解包成 short（并将 cr、cb 左移 8 位）
            __m128i yw = _mm_unpacklo_epi8(y_bias, y_bytes); // 将 y_bias 和 y_bytes 中的字节解包成 16 位整数，存储到 128 位整数向量 yw
            __m128i crw = _mm_unpacklo_epi8(_mm_setzero_si128(), cr_biased); // 将 cr_biased 中的字节解包成 16 位整数，左移 8 位后存储到 128 位整数向量 crw
            __m128i cbw = _mm_unpacklo_epi8(_mm_setzero_si128(), cb_biased); // 将 cb_biased 中的字节解包成 16 位整数，左移 8 位后存储到 128 位整数向量 cbw

            // 颜色转换
            __m128i yws = _mm_srli_epi16(yw, 4); // 将 yw 中的每个 16 位整数逻辑右移 4 位，存储到 128 位整数向量 yws
            __m128i cr0 = _mm_mulhi_epi16(cr_const0, crw); // 将 cr_const0 和 crw 中的每个 16 位整数进行有符号乘法，高位存储到 128 位整数向量 cr0
            __m128i cb0 = _mm_mulhi_epi16(cb_const0, cbw); // 将 cb_const0 和 cbw 中的每个 16 位整数进行有符号乘法，高位存储到 128 位整数向量 cb0
            __m128i cb1 = _mm_mulhi_epi16(cbw, cb_const1); // 将 cbw 和 cb_const1 中的每个 16 位整数进行有符号乘法，高位存储到 128 位整数向量 cb1
// 使用 SIMD 指令对 crw 和 cr_const1 进行 16 位整数乘法并取高位结果
__m128i cr1 = _mm_mulhi_epi16(crw, cr_const1);
// 对 rws、gwt、bws、gws 进行 16 位整数加法
__m128i rws = _mm_add_epi16(cr0, yws);
__m128i gwt = _mm_add_epi16(cb0, yws);
__m128i bws = _mm_add_epi16(yws, cb1);
__m128i gws = _mm_add_epi16(gwt, cr1);

// 对 rw、bw、gw 进行 16 位整数右移 4 位，进行降尺度
__m128i rw = _mm_srai_epi16(rws, 4);
__m128i bw = _mm_srai_epi16(bws, 4);
__m128i gw = _mm_srai_epi16(gws, 4);

// 将 rw、bw、gw 转换为无符号 8 位整数，并打包成 brb 和 gxb
__m128i brb = _mm_packus_epi16(rw, bw);
__m128i gxb = _mm_packus_epi16(gw, xw);

// 对 brb 和 gxb 进行通道交错转置
__m128i t0 = _mm_unpacklo_epi8(brb, gxb);
__m128i t1 = _mm_unpackhi_epi8(brb, gxb);
__m128i o0 = _mm_unpacklo_epi16(t0, t1);
__m128i o1 = _mm_unpackhi_epi16(t0, t1);
            // 将结果存储到内存中
            _mm_storeu_si128((__m128i *)(out + 0), o0);
            _mm_storeu_si128((__m128i *)(out + 16), o1);
            out += 32;
        }
    }
#endif

#ifdef STBI_NEON
    // 在这个版本中，添加 step=3 的支持会很容易。但是是否有需求呢？
    if (step == 4) {
        // 这是一个相当直接的实现，而且并不是特别优化。
        uint8x8_t signflip = vdup_n_u8(0x80);
        int16x8_t cr_const0 = vdupq_n_s16((short)(1.40200f * 4096.0f + 0.5f));
        int16x8_t cr_const1 = vdupq_n_s16(-(short)(0.71414f * 4096.0f + 0.5f));
        int16x8_t cb_const0 = vdupq_n_s16(-(short)(0.34414f * 4096.0f + 0.5f));
        int16x8_t cb_const1 = vdupq_n_s16((short)(1.77200f * 4096.0f + 0.5f));

        for (; i + 7 < count; i += 8) {
// 从内存中加载数据到寄存器中
uint8x8_t y_bytes = vld1_u8(y + i);
uint8x8_t cr_bytes = vld1_u8(pcr + i);
uint8x8_t cb_bytes = vld1_u8(pcb + i);
int8x8_t cr_biased = vreinterpret_s8_u8(vsub_u8(cr_bytes, signflip));
int8x8_t cb_biased = vreinterpret_s8_u8(vsub_u8(cb_bytes, signflip);

// 将数据类型扩展为16位有符号整数
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
            // 取消缩放，四舍五入，转换为字节
            uint8x8x4_t o; // 创建一个包含四个8位元素的向量结构体
            o.val[0] = vqrshrun_n_s16(rws, 4); // 对rws中的16位有符号整数进行右移4位并转换为无符号8位整数，存入o的第一个元素中
            o.val[1] = vqrshrun_n_s16(gws, 4); // 对gws中的16位有符号整数进行右移4位并转换为无符号8位整数，存入o的第二个元素中
            o.val[2] = vqrshrun_n_s16(bws, 4); // 对bws中的16位有符号整数进行右移4位并转换为无符号8位整数，存入o的第三个元素中
            o.val[3] = vdup_n_u8(255); // 创建一个包含四个相同值255的向量，存入o的第四个元素中

            // 存储，交错存储r/g/b/a
            vst4_u8(out, o); // 将o中的值存储到out指向的内存地址中，以交错的方式存储r/g/b/a
            out += 8 * 4; // 更新out指针的位置，移动到下一个像素的位置
        }
    }
#endif

    for (; i < count; ++i) {
        int y_fixed = (y[i] << 20) + (1 << 19); // 对y[i]进行左移20位，加上1左移19位的结果，进行四舍五入
        int r, g, b;
        int cr = pcr[i] - 128; // 对pcr[i]减去128，得到cr的值
        int cb = pcb[i] - 128; // 对pcb[i]减去128，得到cb的值
        # 计算红色分量
        r = y_fixed + cr * stbi__float2fixed(1.40200f);
        # 计算绿色分量
        g = y_fixed + cr * -stbi__float2fixed(0.71414f) + ((cb * -stbi__float2fixed(0.34414f)) & 0xffff0000);
        # 计算蓝色分量
        b = y_fixed + cb * stbi__float2fixed(1.77200f);
        # 右移20位，相当于除以2^20
        r >>= 20;
        g >>= 20;
        b >>= 20;
        # 如果 r 超出范围 [0, 255]，则进行修正
        if ((unsigned)r > 255) {
            if (r < 0)
                r = 0;
            else
                r = 255;
        }
        # 如果 g 超出范围 [0, 255]，则进行修正
        if ((unsigned)g > 255) {
            if (g < 0)
                g = 0;
            else
                g = 255;
        }
        # 如果 b 超出范围 [0, 255]，则进行修正
        if ((unsigned)b > 255) {
            if (b < 0)
// 如果条件成立，将 b 设置为 0
b = 0;
// 如果条件不成立，将 b 设置为 255
else
    b = 255;
// 将 r、g、b 分别转换为无符号字符类型，并赋值给输出数组的前三个元素
out[0] = (stbi_uc)r;
out[1] = (stbi_uc)g;
out[2] = (stbi_uc)b;
// 将输出数组的第四个元素设置为 255
out[3] = 255;
// 输出数组指针向前移动 step 个位置
out += step;
// 设置 JPEG 结构体的 idct_block_kernel、YCbCr_to_RGB_kernel、resample_row_hv_2_kernel 三个成员变量
static void stbi__setup_jpeg(stbi__jpeg * j) {
    j->idct_block_kernel = stbi__idct_block;
    j->YCbCr_to_RGB_kernel = stbi__YCbCr_to_RGB_row;
    j->resample_row_hv_2_kernel = stbi__resample_row_hv_2;
// 如果定义了 STBI_SSE2，执行以下代码
#ifdef STBI_SSE2
// 检查是否支持 SSE2 指令集，如果支持则使用相应的 SIMD 函数
if (stbi__sse2_available()) {
    j->idct_block_kernel = stbi__idct_simd; // 设置 IDCT 块处理的 SIMD 函数
    j->YCbCr_to_RGB_kernel = stbi__YCbCr_to_RGB_simd; // 设置 YCbCr 转换为 RGB 的 SIMD 函数
    j->resample_row_hv_2_kernel = stbi__resample_row_hv_2_simd; // 设置行水平垂直重采样的 SIMD 函数
}
#endif

#ifdef STBI_NEON
    j->idct_block_kernel = stbi__idct_simd; // 设置 IDCT 块处理的 SIMD 函数
    j->YCbCr_to_RGB_kernel = stbi__YCbCr_to_RGB_simd; // 设置 YCbCr 转换为 RGB 的 SIMD 函数
    j->resample_row_hv_2_kernel = stbi__resample_row_hv_2_simd; // 设置行水平垂直重采样的 SIMD 函数
#endif
}

// 清理临时的组件缓冲区
static void stbi__cleanup_jpeg(stbi__jpeg * j) { stbi__free_jpeg_components(j, j->s->img_n, 0); }

typedef struct {
    resample_row_func resample; // 重采样行函数
    stbi_uc *line0, *line1; // 行数据指针
    int hs, vs;  // 在每个轴上的扩展因子
    int w_lores; // 水平像素在扩展之前
    int ystep;   // 垂直扩展的进度
    int ypos;    // 我们所在的扩展前行
} stbi__resample;

// 快速 0..255 * 0..255 => 0..255 的四舍五入乘法
static stbi_uc stbi__blinn_8x8(stbi_uc x, stbi_uc y) {
    unsigned int t = x * y + 128;
    return (stbi_uc)((t + (t >> 8)) >> 8);
}

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

    is_rgb = z->s->img_n == 3 && (z->rgb == 3 || (z->app14_color_transform == 0 && !z->jfif));

    if (z->s->img_n == 3 && n < 3 && !is_rgb)
        decode_n = 1;
    else
        decode_n = z->s->img_n;

    // 如果没有请求任何分量，则无需执行任何操作；现在检查这一点，以避免稍后访问未初始化的 coutput[0]
    if (decode_n <= 0) {
        stbi__cleanup_jpeg(z);
    }
        // 如果出现错误，返回空指针
        return NULL;
    }

    // 重新采样和颜色转换
    {
        int k;
        unsigned int i, j;
        stbi_uc * output;
        stbi_uc * coutput[4] = {NULL, NULL, NULL, NULL};

        stbi__resample res_comp[4];

        for (k = 0; k < decode_n; ++k) {
            stbi__resample * r = &res_comp[k];

            // 分配足够大的行缓冲区，以便在边缘进行上采样，上采样因子为4
            z->img_comp[k].linebuf = (stbi_uc *)stbi__malloc(z->s->img_x + 3);
            // 如果分配失败，清理JPEG数据并返回
            if (!z->img_comp[k].linebuf) {
                stbi__cleanup_jpeg(z);
// 如果内存不足，返回错误信息
return stbi__errpuc("outofmem", "Out of memory");

// 计算水平和垂直方向的采样因子
r->hs = z->img_h_max / z->img_comp[k].h;
r->vs = z->img_v_max / z->img_comp[k].v;

// 计算垂直方向的步长
r->ystep = r->vs >> 1;

// 计算低分辨率图像的宽度
r->w_lores = (z->s->img_x + r->hs - 1) / r->hs;

// 初始化一些变量
r->ypos = 0;
r->line0 = r->line1 = z->img_comp[k].data;

// 根据采样因子选择不同的重采样函数
if (r->hs == 1 && r->vs == 1)
    r->resample = resample_row_1;
else if (r->hs == 1 && r->vs == 2)
    r->resample = stbi__resample_row_v_2;
else if (r->hs == 2 && r->vs == 1)
    r->resample = stbi__resample_row_h_2;
else if (r->hs == 2 && r->vs == 2)
    r->resample = z->resample_row_hv_2_kernel;
else
    r->resample = stbi__resample_row_generic;
        }

        // 之后不会出现错误，所以这是安全的
        // 分配内存以存储解压后的图像数据
        output = (stbi_uc *)stbi__malloc_mad3(n, z->s->img_x, z->s->img_y, 1);
        if (!output) {
            stbi__cleanup_jpeg(z);
            return stbi__errpuc("outofmem", "Out of memory");
        }

        // 开始重采样
        for (j = 0; j < z->s->img_y; ++j) {
            // 指向输出数据的指针
            stbi_uc * out = output + n * z->s->img_x * j;
            for (k = 0; k < decode_n; ++k) {
                // 获取当前重采样组件
                stbi__resample * r = &res_comp[k];
                // 计算是否需要向下取样
                int y_bot = r->ystep >= (r->vs >> 1);
                // 执行重采样
                coutput[k] = r->resample(z->img_comp[k].linebuf, y_bot ? r->line1 : r->line0, y_bot ? r->line0 : r->line1,
                                         r->w_lores, r->hs);
                // 更新ystep和line0
                if (++r->ystep >= r->vs) {
                    r->ystep = 0;
                    r->line0 = r->line1;
                    // 如果当前行数小于图像组件的高度
                    if (++r->ypos < z->img_comp[k].y)
                        // 更新当前行的起始位置
                        r->line1 += z->img_comp[k].w2;
                }
            }
            // 如果图像组件数大于等于3
            if (n >= 3) {
                // 获取亮度数据
                stbi_uc * y = coutput[0];
                // 如果图像通道数为3
                if (z->s->img_n == 3) {
                    // 如果是 RGB 格式
                    if (is_rgb) {
                        // 遍历图像的每个像素
                        for (i = 0; i < z->s->img_x; ++i) {
                            // 将亮度和色度数据写入输出数组
                            out[0] = y[i];
                            out[1] = coutput[1][i];
                            out[2] = coutput[2][i];
                            out[3] = 255;
                            out += n;
                        }
                    } else {
                        // 转换 YCbCr 到 RGB 格式
                        z->YCbCr_to_RGB_kernel(out, y, coutput[1], coutput[2], z->s->img_x, n);
                    }
                } else if (z->s->img_n == 4) {
                    // 如果图像通道数为4
                    if (z->app14_color_transform == 0) { // CMYK
// 遍历图像的每个像素，处理颜色转换
for (i = 0; i < z->s->img_x; ++i) {
    // 获取当前像素的 alpha 值
    stbi_uc m = coutput[3][i];
    // 对 RGB 通道进行颜色转换，使用 stbi__blinn_8x8 函数
    out[0] = stbi__blinn_8x8(coutput[0][i], m);
    out[1] = stbi__blinn_8x8(coutput[1][i], m);
    out[2] = stbi__blinn_8x8(coutput[2][i], m);
    // 设置 alpha 通道为 255
    out[3] = 255;
    // 更新输出指针位置
    out += n;
}
// 如果颜色转换类型为 YCCK
} else if (z->app14_color_transform == 2) { // YCCK
    // 使用 YCbCr_to_RGB_kernel 函数进行颜色转换
    z->YCbCr_to_RGB_kernel(out, y, coutput[1], coutput[2], z->s->img_x, n);
    // 遍历图像的每个像素，处理颜色转换
    for (i = 0; i < z->s->img_x; ++i) {
        // 获取当前像素的 alpha 值
        stbi_uc m = coutput[3][i];
        // 对 RGB 通道进行颜色转换，使用 stbi__blinn_8x8 函数，并取反
        out[0] = stbi__blinn_8x8(255 - out[0], m);
        out[1] = stbi__blinn_8x8(255 - out[1], m);
        out[2] = stbi__blinn_8x8(255 - out[2], m);
        // 更新输出指针位置
        out += n;
    }
// 如果颜色转换类型为其他
} else { // YCbCr + alpha?  Ignore the fourth channel for now
    // 使用 YCbCr_to_RGB_kernel 函数进行颜色转换，忽略第四个通道
    z->YCbCr_to_RGB_kernel(out, y, coutput[1], coutput[2], z->s->img_x, n);
}
// 如果条件成立，执行以下代码块
} else
    // 循环遍历图像的每个像素
    for (i = 0; i < z->s->img_x; ++i) {
        // 将像素值赋给输出数组的RGB通道
        out[0] = out[1] = out[2] = y[i];
        // 如果通道数为3，则out[3]不使用
        out[3] = 255; // 如果n==3则不使用
        // 输出数组指针移动到下一个像素
        out += n;
    }
} else {
    // 如果是RGB格式
    if (is_rgb) {
        // 如果通道数为1
        if (n == 1)
            // 遍历每个像素，计算灰度值并赋给输出数组
            for (i = 0; i < z->s->img_x; ++i)
                *out++ = stbi__compute_y(coutput[0][i], coutput[1][i], coutput[2][i]);
        else {
            // 如果通道数不为1，遍历每个像素
            for (i = 0; i < z->s->img_x; ++i, out += 2) {
                // 计算灰度值并赋给输出数组的第一个通道，第二个通道赋值255
                out[0] = stbi__compute_y(coutput[0][i], coutput[1][i], coutput[2][i]);
                out[1] = 255;
            }
        }
    } else if (z->s->img_n == 4 && z->app14_color_transform == 0) {
        // 如果图像通道数为4且颜色转换为0
        for (i = 0; i < z->s->img_x; ++i) {
            // 获取Alpha通道值
            stbi_uc m = coutput[3][i];
# 使用 stbi__blinn_8x8 函数处理 coutput[0][i] 的值，并将结果赋给 r
stbi_uc r = stbi__blinn_8x8(coutput[0][i], m);
# 使用 stbi__blinn_8x8 函数处理 coutput[1][i] 的值，并将结果赋给 g
stbi_uc g = stbi__blinn_8x8(coutput[1][i], m);
# 使用 stbi__blinn_8x8 函数处理 coutput[2][i] 的值，并将结果赋给 b
stbi_uc b = stbi__blinn_8x8(coutput[2][i], m);
# 使用 stbi__compute_y 函数计算亮度，并将结果赋给 out[0]
out[0] = stbi__compute_y(r, g, b);
# 将 255 赋给 out[1]
out[1] = 255;
# 将 out 指针向后移动 n 个位置
out += n;

# 如果条件满足，则执行以下代码块
if (z->s->img_n == 4 && z->app14_color_transform == 2) {
    # 使用 stbi__blinn_8x8 函数处理 255 - coutput[0][i] 和 coutput[3][i] 的值，并将结果赋给 out[0]
    out[0] = stbi__blinn_8x8(255 - coutput[0][i], coutput[3][i]);
    # 将 255 赋给 out[1]
    out[1] = 255;
    # 将 out 指针向后移动 n 个位置
    out += n;
} else {
    # 将 coutput[0] 赋给 y
    stbi_uc * y = coutput[0];
    # 如果 n 等于 1，则执行以下代码块
    if (n == 1)
        # 将 y[i] 赋给 out[i]
        for (i = 0; i < z->s->img_x; ++i)
            out[i] = y[i];
    else
        # 否则执行以下代码块
        for (i = 0; i < z->s->img_x; ++i) {
// 将 y[i] 的值赋给 out 指针所指向的地址，并将 out 指针向后移动一个位置
*out++ = y[i];
// 将 255 的值赋给 out 指针所指向的地址，并将 out 指针向后移动一个位置
*out++ = 255;
// 清理 JPEG 数据
stbi__cleanup_jpeg(z);
// 将 z->s->img_x 的值赋给 out_x 指针所指向的地址
*out_x = z->s->img_x;
// 将 z->s->img_y 的值赋给 out_y 指针所指向的地址
*out_y = z->s->img_y;
// 如果 comp 存在
if (comp)
    // 如果 z->s->img_n 大于等于 3，则将 3 赋给 comp 指针所指向的地址，否则将 1 赋给 comp 指针所指向的地址
    *comp = z->s->img_n >= 3 ? 3 : 1; // 报告原始组件数，而不是输出
// 返回 output 指针所指向的地址
return output;
// 为 stbi__jpeg 结构分配内存
static void * stbi__jpeg_load(stbi__context * s, int * x, int * y, int * comp, int req_comp, stbi__result_info * ri) {
    unsigned char * result;
    stbi__jpeg * j = (stbi__jpeg *)stbi__malloc(sizeof(stbi__jpeg));
    // 如果 j 为空
    if (!j)
        // 返回错误信息
        return stbi__errpuc("outofmem", "Out of memory");
// 将结构体 stbi__jpeg 的内存空间初始化为 0
memset(j, 0, sizeof(stbi__jpeg));
// 宏定义，用于标记变量 ri 未使用
STBI_NOTUSED(ri);
// 将结构体 stbi__jpeg 的成员变量 s 赋值为 s
j->s = s;
// 调用函数 stbi__setup_jpeg 对结构体 stbi__jpeg 进行初始化设置
stbi__setup_jpeg(j);
// 调用函数 load_jpeg_image 加载 JPEG 图像，返回加载结果
result = load_jpeg_image(j, x, y, comp, req_comp);
// 释放结构体 stbi__jpeg 占用的内存空间
STBI_FREE(j);
// 返回加载结果
return result;
}

// JPEG 格式测试函数
static int stbi__jpeg_test(stbi__context * s) {
    int r;
    // 分配 stbi__jpeg 结构体所需的内存空间
    stbi__jpeg * j = (stbi__jpeg *)stbi__malloc(sizeof(stbi__jpeg));
    // 如果内存分配失败，返回错误信息
    if (!j)
        return stbi__err("outofmem", "Out of memory");
    // 将结构体 stbi__jpeg 的内存空间初始化为 0
    memset(j, 0, sizeof(stbi__jpeg));
    // 将结构体 stbi__jpeg 的成员变量 s 赋值为 s
    j->s = s;
    // 调用函数 stbi__setup_jpeg 对结构体 stbi__jpeg 进行初始化设置
    stbi__setup_jpeg(j);
    // 调用函数 stbi__decode_jpeg_header 对 JPEG 头部进行解码
    r = stbi__decode_jpeg_header(j, STBI__SCAN_type);
    // 将输入流 s 的位置重置到起始位置
    stbi__rewind(s);
    // 释放结构体 stbi__jpeg 占用的内存空间
    STBI_FREE(j);
// 返回解析后的结果
return r;
}

// 获取 JPEG 图像的原始信息
static int stbi__jpeg_info_raw(stbi__jpeg * j, int * x, int * y, int * comp) {
    // 如果无法解析 JPEG 头部信息，则回退到文件开头并返回 0
    if (!stbi__decode_jpeg_header(j, STBI__SCAN_header)) {
        stbi__rewind(j->s);
        return 0;
    }
    // 如果 x 不为空，则将图像宽度赋值给 x
    if (x)
        *x = j->s->img_x;
    // 如果 y 不为空，则将图像高度赋值给 y
    if (y)
        *y = j->s->img_y;
    // 如果 comp 不为空，则根据图像通道数赋值给 comp
    if (comp)
        *comp = j->s->img_n >= 3 ? 3 : 1;
    // 返回 1 表示成功获取信息
    return 1;
}

// 获取 JPEG 图像的信息
static int stbi__jpeg_info(stbi__context * s, int * x, int * y, int * comp) {
    int result;
    // 分配内存给 JPEG 对象 j
    stbi__jpeg * j = (stbi__jpeg *)(stbi__malloc(sizeof(stbi__jpeg)));
// 如果 j 为空，则返回内存不足的错误信息
    if (!j)
        return stbi__err("outofmem", "Out of memory");
    // 将 j 的内存空间初始化为 0
    memset(j, 0, sizeof(stbi__jpeg));
    // 将 j 的输入流设置为 s
    j->s = s;
    // 调用 stbi__jpeg_info_raw 函数获取 JPEG 图像的信息
    result = stbi__jpeg_info_raw(j, x, y, comp);
    // 释放 j 占用的内存空间
    STBI_FREE(j);
    // 返回获取的 JPEG 图像信息
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
```

// 定义 STBI__ZFAST_BITS 为 9，用于加速默认表中的所有情况
#define STBI__ZFAST_BITS 9 
// 定义 STBI__ZFAST_MASK 为 (1 << STBI__ZFAST_BITS) - 1
#define STBI__ZFAST_MASK ((1 << STBI__ZFAST_BITS) - 1)
// 定义 STBI__ZNSYMS 为 288，表示字面/长度字母表中的符号数
#define STBI__ZNSYMS 288 

// zlib 风格的哈夫曼编码
// (JPEG 从左边打包，zlib 从右边打包，所以不能共享代码)
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
    // 返回变量 n
    return n;
}

// 对输入的 v 进行位反转，返回反转后的结果
stbi_inline static int stbi__bit_reverse(int v, int bits) {
    // 断言 bits 不大于 16
    STBI_ASSERT(bits <= 16);
    // 对于 n 位进行位反转，先反转 16 位，然后右移
    // 例如，11 位，进行位反转并右移 5 位
    return stbi__bitreverse16(v) >> (16 - bits);
}

// 构建哈夫曼树
static int stbi__zbuild_huffman(stbi__zhuffman * z, const stbi_uc * sizelist, int num) {
    int i, k = 0;
    int code, next_code[16], sizes[17];

    // 根据 DEFLATE 规范生成哈夫曼编码
    // 初始化 sizes 和 z->fast 数组
    memset(sizes, 0, sizeof(sizes));
    memset(z->fast, 0, sizeof(z->fast));
    // 统计每个大小的出现次数
    for (i = 0; i < num; ++i)
        ++sizes[sizelist[i]];
    sizes[0] = 0;
    // 循环遍历 i 从 1 到 15
    for (i = 1; i < 16; ++i)
        // 如果 sizes[i] 大于 2 的 i 次方，则返回错误信息
        if (sizes[i] > (1 << i))
            return stbi__err("bad sizes", "Corrupt PNG");
    // 初始化 code 为 0
    code = 0;
    // 再次循环遍历 i 从 1 到 15
    for (i = 1; i < 16; ++i) {
        // 将当前 code 赋值给 next_code[i]
        next_code[i] = code;
        // 将当前 code 赋值给 z->firstcode[i]
        z->firstcode[i] = (stbi__uint16)code;
        // 将当前 k 赋值给 z->firstsymbol[i]
        z->firstsymbol[i] = (stbi__uint16)k;
        // 更新 code 的值
        code = (code + sizes[i]);
        // 如果 sizes[i] 不为 0
        if (sizes[i])
            // 如果 code - 1 大于等于 2 的 i 次方，则返回错误信息
            if (code - 1 >= (1 << i))
                return stbi__err("bad codelengths", "Corrupt PNG");
        // 将 code 左移 16 - i 位，赋值给 z->maxcode[i]，为内部循环做预处理
        z->maxcode[i] = code << (16 - i); // preshift for inner loop
        // 将 code 左移 1 位
        code <<= 1;
        // 更新 k 的值
        k += sizes[i];
    }
    // 将 z->maxcode[16] 的值设为 0x10000，作为结束标志
    z->maxcode[16] = 0x10000; // sentinel
    // 再次循环遍历 i 从 0 到 num
    for (i = 0; i < num; ++i) {
        // 将 sizelist[i] 的值赋给 s
        int s = sizelist[i];
        // 如果 s 不为 0
        if (s) {
// 从内存中读取 zlib 数据的实现，用于 PNG 读取
// 因为 PNG 允许任意分割 zlib 流，结构上让 PNG 调用 ZLIB 再调用 PNG 很麻烦
int c = next_code[s] - z->firstcode[s] + z->firstsymbol[s];
// 计算当前编码的值
stbi__uint16 fastv = (stbi__uint16)((s << 9) | i);
// 将当前符号的快速查找表值计算出来
z->size[c] = (stbi_uc)s;
// 将当前编码的大小存入大小表
z->value[c] = (stbi__uint16)i;
// 将当前编码的值存入值表
if (s <= STBI__ZFAST_BITS) {
    int j = stbi__bit_reverse(next_code[s], s);
    // 如果当前编码小于等于 STBI__ZFAST_BITS，则进行快速查找表的填充
    while (j < (1 << STBI__ZFAST_BITS)) {
        z->fast[j] = fastv;
        j += (1 << s);
    }
}
// 增加下一个编码的值
++next_code[s];
// 结束内层循环
}
// 结束外层循环
return 1;
// 返回结果
}
// 定义了一个结构体 stbi__zbuf，用于存储解压缩过程中的相关信息
typedef struct {
    stbi_uc *zbuffer, *zbuffer_end; // 存储解压缩数据的缓冲区起始和结束位置
    int num_bits; // 存储当前字节的剩余未处理的位数
    stbi__uint32 code_buffer; // 存储当前字节的数据

    char * zout; // 存储解压缩后的数据的起始位置
    char * zout_start; // 存储解压缩后的数据的起始位置
    char * zout_end; // 存储解压缩后的数据的结束位置
    int z_expandable; // 标记解压缩后的数据是否可扩展

    stbi__zhuffman z_length, z_distance; // 存储解压缩过程中的哈夫曼编码信息
} stbi__zbuf;

// 判断解压缩缓冲区是否已经读取完毕
stbi_inline static int stbi__zeof(stbi__zbuf * z) { return (z->zbuffer >= z->zbuffer_end); }

// 从解压缩缓冲区中读取一个字节的数据
stbi_inline static stbi_uc stbi__zget8(stbi__zbuf * z) { return stbi__zeof(z) ? 0 : *z->zbuffer++; }
// 用于填充位流缓冲区，直到缓冲区中的位数达到24位
static void stbi__fill_bits(stbi__zbuf * z) {
    do {
        // 如果位流缓冲区中的位数超过当前位数限制，则将缓冲区标记为结束，以便失败处理
        if (z->code_buffer >= (1U << z->num_bits)) {
            z->zbuffer = z->zbuffer_end; /* treat this as EOF so we fail. */
            return;
        }
        // 将下一个字节的数据填充到位流缓冲区中
        z->code_buffer |= (unsigned int)stbi__zget8(z) << z->num_bits;
        z->num_bits += 8;
    } while (z->num_bits <= 24);
}

// 从位流缓冲区中接收n位数据
stbi_inline static unsigned int stbi__zreceive(stbi__zbuf * z, int n) {
    unsigned int k;
    // 如果位流缓冲区中的位数小于n，则填充位流缓冲区
    if (z->num_bits < n)
        stbi__fill_bits(z);
    // 从位流缓冲区中取出n位数据
    k = z->code_buffer & ((1 << n) - 1);
    z->code_buffer >>= n;
    z->num_bits -= n;
    return k;
}
static int stbi__zhuffman_decode_slowpath(stbi__zbuf * a, stbi__zhuffman * z) {
    int b, s, k;
    // 如果无法通过快速查找表解析，就使用慢速方法计算
    // 使用 JPEG 方法，要求 MS 位在顶部
    k = stbi__bit_reverse(a->code_buffer, 16);
    for (s = STBI__ZFAST_BITS + 1;; ++s)
        if (k < z->maxcode[s])
            break;
    if (s >= 16)
        return -1; // 无效的编码！
    // 编码大小为 s，因此：
    b = (k >> (16 - s)) - z->firstcode[s] + z->firstsymbol[s];
    if (b >= STBI__ZNSYMS)
        return -1; // 某些数据在某个地方损坏了！
    if (z->size[b] != s)
        return -1; // 最初是一个断言，但现在报告失败。
    a->code_buffer >>= s;
    a->num_bits -= s;
    return z->value[b];
}
stbi_inline static int stbi__zhuffman_decode(stbi__zbuf * a, stbi__zhuffman * z) {
    // 定义一个内联函数，用于解码哈夫曼编码
    int b, s;
    // 如果当前比特数小于16，则填充比特流
    if (a->num_bits < 16) {
        // 如果已经到达流的末尾，则返回错误
        if (stbi__zeof(a)) {
            return -1; /* 报告意外的数据结束错误 */
        }
        stbi__fill_bits(a);
    }
    // 从快速查找表中获取编码
    b = z->fast[a->code_buffer & STBI__ZFAST_MASK];
    // 如果编码存在
    if (b) {
        // 获取编码的长度
        s = b >> 9;
        // 将编码缓冲区右移s位
        a->code_buffer >>= s;
        // 减去已使用的比特数
        a->num_bits -= s;
        // 返回解码结果
        return b & 511;
    }
    // 否则调用慢速解码函数
    return stbi__zhuffman_decode_slowpath(a, z);
}
// 扩展压缩数据的输出缓冲区，以便能够容纳 n 个字节
static int stbi__zexpand(stbi__zbuf * z, char * zout, int n) {
    char * q;
    unsigned int cur, limit, old_limit;
    z->zout = zout; // 设置输出缓冲区的起始位置
    if (!z->z_expandable) // 如果输出缓冲区不可扩展，则返回错误
        return stbi__err("output buffer limit", "Corrupt PNG");
    cur = (unsigned int)(z->zout - z->zout_start); // 计算当前输出位置与起始位置的偏移量
    limit = old_limit = (unsigned)(z->zout_end - z->zout_start); // 设置当前输出缓冲区的大小
    if (UINT_MAX - cur < (unsigned)n) // 如果要扩展的大小超出了限制，则返回错误
        return stbi__err("outofmem", "Out of memory");
    while (cur + n > limit) { // 当要扩展的大小超出了当前缓冲区的大小时
        if (limit > UINT_MAX / 2) // 如果当前缓冲区大小已经接近上限，则返回错误
            return stbi__err("outofmem", "Out of memory");
        limit *= 2; // 将当前缓冲区大小扩大一倍
    }
    q = (char *)STBI_REALLOC_SIZED(z->zout_start, old_limit, limit); // 重新分配内存以扩展缓冲区
    STBI_NOTUSED(old_limit); // 标记 old_limit 为未使用，避免编译器警告
    if (q == NULL) // 如果内存分配失败，则返回错误
        return stbi__err("outofmem", "Out of memory");
    // 设置输出缓冲区的起始位置
    z->zout_start = q;
    // 设置输出缓冲区的当前位置
    z->zout = q + cur;
    // 设置输出缓冲区的结束位置
    z->zout_end = q + limit;
    // 返回 1，表示成功
    return 1;
}

// 哈夫曼编码长度的基础值
static const int stbi__zlength_base[31] = {3,  4,  5,  6,  7,  8,  9,  10,  11,  13,  15,  17,  19,  23, 27, 31,
                                           35, 43, 51, 59, 67, 83, 99, 115, 131, 163, 195, 227, 258, 0,  0};

// 哈夫曼编码长度的额外值
static const int stbi__zlength_extra[31] = {0, 0, 0, 0, 0, 0, 0, 0, 1, 1, 1, 1, 2, 2, 2, 2,
                                            3, 3, 3, 3, 4, 4, 4, 4, 5, 5, 5, 5, 0, 0, 0};

// 哈夫曼编码距离的基础值
static const int stbi__zdist_base[32] = {1,    2,    3,    4,    5,    7,     9,     13,    17,  25,   33,
                                         49,   65,   97,   129,  193,  257,   385,   513,   769, 1025, 1537,
                                         2049, 3073, 4097, 6145, 8193, 12289, 16385, 24577, 0,   0};

// 哈夫曼编码距离的额外值
static const int stbi__zdist_extra[32] = {0, 0, 0, 0, 1, 1, 2, 2,  3,  3,  4,  4,  5,  5,  6,
                                          6, 7, 7, 8, 8, 9, 9, 10, 10, 11, 11, 12, 12, 13, 13};

// 解析哈夫曼编码块的函数
static int stbi__parse_huffman_block(stbi__zbuf * a) {
    // 从输入的数据中读取字符
    char * zout = a->zout;
    // 无限循环，直到遇到终止条件
    for (;;) {
        // 使用霍夫曼解码器解码数据
        int z = stbi__zhuffman_decode(a, &a->z_length);
        // 如果解码结果小于256，表示为字符数据
        if (z < 256) {
            // 如果解码结果小于0，表示霍夫曼编码出错
            if (z < 0)
                return stbi__err("bad huffman code", "Corrupt PNG"); // error in huffman codes
            // 如果输出缓冲区已满，进行扩展
            if (zout >= a->zout_end) {
                if (!stbi__zexpand(a, zout, 1))
                    return 0;
                zout = a->zout;
            }
            // 将解码结果存入输出缓冲区
            *zout++ = (char)z;
        } else {
            // 如果解码结果大于等于256，表示为长度和距离数据
            stbi_uc * p;
            int len, dist;
            // 如果解码结果为256，表示数据解压结束
            if (z == 256) {
                a->zout = zout;
                return 1;
            }
            // 如果解码结果大于等于286
// 如果出现错误的哈夫曼编码，返回错误信息
return stbi__err("bad huffman code", "Corrupt PNG"); // 根据 DEFLATE 压缩算法，长度代码 286 和 287 不应出现在压缩数据中
z -= 257;
// 根据长度基础值和额外值计算长度
len = stbi__zlength_base[z];
if (stbi__zlength_extra[z])
    len += stbi__zreceive(a, stbi__zlength_extra[z]);
// 解码距离值
z = stbi__zhuffman_decode(a, &a->z_distance);
// 如果距离值小于 0 或大于等于 30，返回错误信息
if (z < 0 || z >= 30)
    return stbi__err("bad huffman code", "Corrupt PNG"); // 根据 DEFLATE 压缩算法，距离代码 30 和 31 不应出现在压缩数据中
// 根据距离基础值和额外值计算距离
dist = stbi__zdist_base[z];
if (stbi__zdist_extra[z])
    dist += stbi__zreceive(a, stbi__zdist_extra[z]);
// 如果输出位置减去起始输出位置小于距离，返回错误信息
if (zout - a->zout_start < dist)
    return stbi__err("bad dist", "Corrupt PNG");
// 如果输出位置加上长度大于输出结束位置，尝试扩展输出缓冲区
if (zout + len > a->zout_end) {
    if (!stbi__zexpand(a, zout, len))
        return 0;
    zout = a->zout;
}
            p = (stbi_uc *)(zout - dist);
            // 计算指针 p，指向当前输出位置减去距离 dist 的位置
            if (dist == 1) { // run of one byte; common in images.
                // 如果距离为1，表示连续一个字节的情况，这在图像中很常见
                stbi_uc v = *p;
                // 将指针 p 指向的值赋给变量 v
                if (len) {
                    // 如果长度不为0
                    do
                        *zout++ = v;
                    // 将变量 v 的值写入输出缓冲区，并递增 zout 指针
                    while (--len);
                    // 重复上述操作直到长度减为0
                }
            } else {
                // 如果距离不为1
                if (len) {
                    // 如果长度不为0
                    do
                        *zout++ = *p++;
                    // 将指针 p 指向的值写入输出缓冲区，并递增 p 指针
                    while (--len);
                    // 重复上述操作直到长度减为0
                }
            }
        }
    }
}

static int stbi__compute_huffman_codes(stbi__zbuf * a) {
// 计算哈夫曼编码
    // 定义长度反Zigzag数组，用于解码哈夫曼编码
    static const stbi_uc length_dezigzag[19] = {16, 17, 18, 0, 8, 7, 9, 6, 10, 5, 11, 4, 12, 3, 13, 2, 14, 1, 15};
    // 定义哈夫曼编码长度结构
    stbi__zhuffman z_codelength;
    // 定义长度编码数组，用于存储哈夫曼编码
    stbi_uc lencodes[286 + 32 + 137]; // 为最大单操作填充
    // 定义编码长度数组，用于存储哈夫曼编码长度
    stbi_uc codelength_sizes[19];
    // 定义变量 i, n
    int i, n;

    // 读取HLIT值，用于解码哈夫曼编码
    int hlit = stbi__zreceive(a, 5) + 257;
    // 读取HDIST值，用于解码哈夫曼编码
    int hdist = stbi__zreceive(a, 5) + 1;
    // 读取HCLEN值，用于解码哈夫曼编码
    int hclen = stbi__zreceive(a, 4) + 4;
    // 计算哈夫曼编码总数
    int ntot = hlit + hdist;

    // 将编码长度数组初始化为0
    memset(codelength_sizes, 0, sizeof(codelength_sizes));
    // 读取HCLEN个长度编码，存储到编码长度数组中
    for (i = 0; i < hclen; ++i) {
        int s = stbi__zreceive(a, 3);
        codelength_sizes[length_dezigzag[i]] = (stbi_uc)s;
    }
    // 构建哈夫曼编码树
    if (!stbi__zbuild_huffman(&z_codelength, codelength_sizes, 19))
        return 0;

    // 初始化变量n为0
    n = 0;
    # 当 n 小于 ntot 时执行循环
    while (n < ntot) {
        # 从哈夫曼编码表 a 中解码出一个值，存入变量 c，同时更新 z_codelength
        int c = stbi__zhuffman_decode(a, &z_codelength);
        # 如果 c 小于 0 或者大于等于 19，则返回错误信息
        if (c < 0 || c >= 19)
            return stbi__err("bad codelengths", "Corrupt PNG");
        # 如果 c 小于 16，则将其作为长度编码存入 lencodes 数组
        if (c < 16)
            lencodes[n++] = (stbi_uc)c;
        # 如果 c 大于等于 16
        else {
            # 声明并初始化 fill 变量
            stbi_uc fill = 0;
            # 如果 c 等于 16
            if (c == 16) {
                # 从输入流中读取 2 位并加上 3，存入 c
                c = stbi__zreceive(a, 2) + 3;
                # 如果 n 等于 0，则返回错误信息
                if (n == 0)
                    return stbi__err("bad codelengths", "Corrupt PNG");
                # 将 lencodes 数组中前一个元素的值存入 fill
                fill = lencodes[n - 1];
            }
            # 如果 c 等于 17
            else if (c == 17) {
                # 从输入流中读取 3 位并加上 3，存入 c
                c = stbi__zreceive(a, 3) + 3;
            }
            # 如果 c 等于 18
            else if (c == 18) {
                # 从输入流中读取 7 位并加上 11，存入 c
                c = stbi__zreceive(a, 7) + 11;
            }
            # 如果 c 不是 16、17、18 中的任意一个值，则返回错误信息
            else {
                return stbi__err("bad codelengths", "Corrupt PNG");
            }
    // 如果剩余的编码长度小于当前的 c 值，则返回错误
    if (ntot - n < c)
        return stbi__err("bad codelengths", "Corrupt PNG");
    // 将 lencodes + n 到 lencodes + n + c 的内存填充为 fill
    memset(lencodes + n, fill, c);
    // n 值增加 c
    n += c;
    // 循环直到 n 等于 ntot
    while (n < ntot) {
        // ...
    }
    // 如果 n 不等于 ntot，则返回错误
    if (n != ntot)
        return stbi__err("bad codelengths", "Corrupt PNG");
    // 构建 huffman 编码树，存储在 a->z_length 中
    if (!stbi__zbuild_huffman(&a->z_length, lencodes, hlit))
        return 0;
    // 构建 huffman 编码树，存储在 a->z_distance 中
    if (!stbi__zbuild_huffman(&a->z_distance, lencodes + hlit, hdist))
        return 0;
    // 返回成功
    return 1;
}

// 解析未压缩的数据块
static int stbi__parse_uncompressed_block(stbi__zbuf * a) {
    // 读取头部数据
    stbi_uc header[4];
    int len, nlen, k;
    // 如果当前比特位数不是 8 的倍数，则丢弃
    if (a->num_bits & 7)
        stbi__zreceive(a, a->num_bits & 7); // discard
    // 将位压缩数据填充到头部
    k = 0;
    while (a->num_bits > 0) {
        header[k++] = (stbi_uc)(a->code_buffer & 255); // 抑制 MSVC 运行时检查
        a->code_buffer >>= 8;
        a->num_bits -= 8;
    }
    if (a->num_bits < 0)
        return stbi__err("zlib corrupt", "Corrupt PNG");
    // 现在以正常方式填充头部
    while (k < 4)
        header[k++] = stbi__zget8(a);
    len = header[1] * 256 + header[0];
    nlen = header[3] * 256 + header[2];
    if (nlen != (len ^ 0xffff))
        return stbi__err("zlib corrupt", "Corrupt PNG");
    if (a->zbuffer + len > a->zbuffer_end)
        return stbi__err("read past buffer", "Corrupt PNG");
    if (a->zout + len > a->zout_end)
        if (!stbi__zexpand(a, a->zout, len))
            return 0; // 返回0
    memcpy(a->zout, a->zbuffer, len); // 将a->zbuffer中的数据复制到a->zout中，长度为len
    a->zbuffer += len; // 将a->zbuffer指针向后移动len个位置
    a->zout += len; // 将a->zout指针向后移动len个位置
    return 1; // 返回1
}

static int stbi__parse_zlib_header(stbi__zbuf * a) {
    int cmf = stbi__zget8(a); // 从输入流中读取一个字节，存储在cmf中
    int cm = cmf & 15; // 将cmf的低4位存储在cm中
    /* int cinfo = cmf >> 4; */ // 注释掉的代码，不执行
    int flg = stbi__zget8(a); // 从输入流中读取一个字节，存储在flg中
    if (stbi__zeof(a)) // 如果输入流已经结束
        return stbi__err("bad zlib header", "Corrupt PNG"); // 返回错误信息
    if ((cmf * 256 + flg) % 31 != 0) // 如果cmf和flg的组合不符合zlib规范
        return stbi__err("bad zlib header", "Corrupt PNG"); // 返回错误信息
    if (flg & 32) // 如果flg的第5位为1
        return stbi__err("no preset dict", "Corrupt PNG"); // 返回错误信息
    if (cm != 8) // 如果cm不等于8
        return stbi__err("bad compression", "Corrupt PNG"); // 返回错误信息
// 返回值为1，但是不清楚具体作用
return 1;
}

// 定义长度为STBI__ZNSYMS的数组，用于存储默认长度值
static const stbi_uc stbi__zdefault_length[STBI__ZNSYMS] = {
    // 为数组赋初值为8
    8, 8, 8, 8, 8, 8, 8, 8, 8, 8, 8, 8, 8, 8, 8, 8, 8, 8, 8, 8, 8, 8, 8, 8, 8, 8, 8, 8, 8, 8, 8, 8, 8, 8, 8, 8,
    // ...（省略部分赋值）
    8, 8, 8, 8, 8, 8, 8, 8, 8, 8, 8, 8, 8, 8, 8, 8, 8, 8, 8, 8, 8, 8, 8, 8, 8, 8, 8, 8, 8, 8, 8, 8, 8, 8, 8, 8,
    // ...（省略部分赋值）
    8, 8, 8, 8, 8, 8, 8, 8, 8, 8, 8, 8, 8, 8, 8, 8, 8, 8, 8, 8, 8, 8, 8, 8, 8, 8, 8, 8, 8, 8, 8, 8, 8
// 设置长度码的默认值
for (   ; i <= 255; ++i)     stbi__zdefault_length[i]   = 9;
// 设置长度码的默认值
for (   ; i <= 279; ++i)     stbi__zdefault_length[i]   = 7;
// 设置长度码的默认值
for (   ; i <= 287; ++i)     stbi__zdefault_length[i]   = 8;

// 设置距离码的默认值
for (i=0; i <=  31; ++i)     stbi__zdefault_distance[i] = 5;
*/

// 解析 zlib 数据
static int stbi__parse_zlib(stbi__zbuf * a, int parse_header) {
    int final, type;
    // 如果需要解析头部信息，则解析 zlib 头部
    if (parse_header)
        if (!stbi__parse_zlib_header(a))
            return 0;
    // 初始化位数和缓冲区
    a->num_bits = 0;
    a->code_buffer = 0;
    do {
        // 读取最后一个块标志位
        final = stbi__zreceive(a, 1);
        // 读取块的类型
        type = stbi__zreceive(a, 2);
        if (type == 0) {
            // 如果块类型为 0，解析未压缩块
            if (!stbi__parse_uncompressed_block(a))
// 如果类型为0，返回0
        return 0;
// 如果类型为3，返回0
        } else if (type == 3) {
            return 0;
        } else {
            // 如果类型为1，使用固定的编码长度
            if (type == 1) {
                // 使用默认的长度构建哈夫曼编码树
                if (!stbi__zbuild_huffman(&a->z_length, stbi__zdefault_length, STBI__ZNSYMS))
                    return 0;
                // 使用默认的距离构建哈夫曼编码树
                if (!stbi__zbuild_huffman(&a->z_distance, stbi__zdefault_distance, 32))
                    return 0;
            } else {
                // 计算哈夫曼编码
                if (!stbi__compute_huffman_codes(a))
                    return 0;
            }
            // 解析哈夫曼块
            if (!stbi__parse_huffman_block(a))
                return 0;
        }
    } while (!final);
    // 返回1
    return 1;
}
// 使用 zlib 对数据进行解压缩
static int stbi__do_zlib(stbi__zbuf * a, char * obuf, int olen, int exp, int parse_header) {
    // 设置输出缓冲区的起始位置、当前位置和结束位置
    a->zout_start = obuf;
    a->zout = obuf;
    a->zout_end = obuf + olen;
    // 设置是否可扩展
    a->z_expandable = exp;

    // 调用解析 zlib 数据的函数
    return stbi__parse_zlib(a, parse_header);
}

// 根据给定的数据和长度，使用 zlib 进行解压缩，并返回解压缩后的数据
STBIDEF char * stbi_zlib_decode_malloc_guesssize(const char * buffer, int len, int initial_size, int * outlen) {
    stbi__zbuf a;
    // 分配初始大小的内存
    char * p = (char *)stbi__malloc(initial_size);
    if (p == NULL)
        return NULL;
    // 设置输入缓冲区的起始位置和结束位置
    a.zbuffer = (stbi_uc *)buffer;
    a.zbuffer_end = (stbi_uc *)buffer + len;
    // 调用 zlib 解压缩函数
    if (stbi__do_zlib(&a, p, initial_size, 1, 1)) {
        // 如果输出长度指针不为空，设置解压缩后的数据长度
        if (outlen)
            *outlen = (int)(a.zout - a.zout_start);
// 如果条件成立，返回a.zout_start；否则释放a.zout_start的内存并返回NULL
        return a.zout_start;
    } else {
        STBI_FREE(a.zout_start);
        return NULL;
    }
}

// 使用默认大小16384调用stbi_zlib_decode_malloc_guesssize函数
STBIDEF char * stbi_zlib_decode_malloc(char const * buffer, int len, int * outlen) {
    return stbi_zlib_decode_malloc_guesssize(buffer, len, 16384, outlen);
}

// 根据给定的参数调用stbi__do_zlib函数进行解码
STBIDEF char * stbi_zlib_decode_malloc_guesssize_headerflag(const char * buffer, int len, int initial_size, int * outlen,
                                                            int parse_header) {
    stbi__zbuf a;
    // 分配初始大小为initial_size的内存
    char * p = (char *)stbi__malloc(initial_size);
    if (p == NULL)
        return NULL;
    // 设置a的zbuffer和zbuffer_end属性
    a.zbuffer = (stbi_uc *)buffer;
    a.zbuffer_end = (stbi_uc *)buffer + len;
    // 如果stbi__do_zlib函数成功执行，则返回解码后的数据
    if (stbi__do_zlib(&a, p, initial_size, 1, parse_header)) {
// 如果输出长度不为零，则将输出长度设置为a.zout - a.zout_start的值，并返回a.zout_start
if (outlen)
    *outlen = (int)(a.zout - a.zout_start);
return a.zout_start;
// 否则，释放a.zout_start的内存，并返回NULL
} else {
    STBI_FREE(a.zout_start);
    return NULL;
}

// 解压缩缓冲区中的数据，返回解压后的数据长度
STBIDEF int stbi_zlib_decode_buffer(char * obuffer, int olen, char const * ibuffer, int ilen) {
    // 创建stbi__zbuf对象a，并设置其zbuffer和zbuffer_end属性
    stbi__zbuf a;
    a.zbuffer = (stbi_uc *)ibuffer;
    a.zbuffer_end = (stbi_uc *)ibuffer + ilen;
    // 调用stbi__do_zlib函数进行zlib解压缩，如果成功则返回解压后的数据长度，否则返回-1
    if (stbi__do_zlib(&a, obuffer, olen, 0, 1))
        return (int)(a.zout - a.zout_start);
    else
        return -1;
}

// 使用malloc分配内存，解压缩数据并返回解压后的数据
STBIDEF char * stbi_zlib_decode_noheader_malloc(char const * buffer, int len, int * outlen) {
    // 创建一个stbi__zbuf结构体变量a
    stbi__zbuf a;
    // 分配16384字节内存，并将其转换为char指针p
    char * p = (char *)stbi__malloc(16384);
    // 如果分配内存失败，则返回空指针
    if (p == NULL)
        return NULL;
    // 将buffer转换为stbi_uc类型，并赋值给a.zbuffer
    a.zbuffer = (stbi_uc *)buffer;
    // 将buffer的末尾位置赋值给a.zbuffer_end
    a.zbuffer_end = (stbi_uc *)buffer + len;
    // 调用stbi__do_zlib函数进行zlib解压缩
    if (stbi__do_zlib(&a, p, 16384, 1, 0)) {
        // 如果outlen不为空，则将解压后的数据长度赋值给outlen
        if (outlen)
            *outlen = (int)(a.zout - a.zout_start);
        // 返回解压后的数据起始地址
        return a.zout_start;
    } else {
        // 释放a.zout_start指向的内存
        STBI_FREE(a.zout_start);
        // 返回空指针
        return NULL;
    }
}

// 定义stbi_zlib_decode_noheader_buffer函数
STBIDEF int stbi_zlib_decode_noheader_buffer(char * obuffer, int olen, const char * ibuffer, int ilen) {
    // 创建一个stbi__zbuf结构体变量a
    stbi__zbuf a;
    // 将ibuffer转换为stbi_uc类型，并赋值给a.zbuffer
    a.zbuffer = (stbi_uc *)ibuffer;
    // 将ibuffer的末尾位置赋值给a.zbuffer_end
    a.zbuffer_end = (stbi_uc *)ibuffer + ilen;
// 如果使用 stbi__do_zlib 函数成功解压缩数据，则返回解压后数据的长度，否则返回 -1
if (stbi__do_zlib(&a, obuffer, olen, 0, 0))
    return (int)(a.zout - a.zout_start);
else
    return -1;
}
#endif

// 公有领域的“基线”PNG解码器 v0.10 Sean Barrett 2006-11-18
// 简单的实现
//   - 只有8位样本
//   - 没有CRC检查
//   - 分配大量中间内存
//     - 避免在子系统之间流式传输数据的问题
//     - 避免显式窗口管理
// 性能
//   - 使用stb_zlib，一个具有快速哈夫曼解码的PD zlib实现

#ifndef STBI_NO_PNG
typedef struct {
    stbi__uint32 length;
```

// 定义一个结构体 stbi__pngchunk，包含两个成员变量：长度和类型
typedef struct {
    stbi__uint32 length;
    stbi__uint32 type;
} stbi__pngchunk;

// 从 stbi__context 中获取 PNG 数据块头部信息
static stbi__pngchunk stbi__get_chunk_header(stbi__context * s) {
    stbi__pngchunk c;
    // 读取数据块长度
    c.length = stbi__get32be(s);
    // 读取数据块类型
    c.type = stbi__get32be(s);
    return c;
}

// 检查 PNG 文件头部信息是否正确
static int stbi__check_png_header(stbi__context * s) {
    // PNG 文件的签名
    static const stbi_uc png_sig[8] = {137, 80, 78, 71, 13, 10, 26, 10};
    int i;
    // 遍历 PNG 文件的签名
    for (i = 0; i < 8; ++i)
        // 检查文件签名是否正确
        if (stbi__get8(s) != png_sig[i])
            return stbi__err("bad png sig", "Not a PNG");
    return 1;
}

// 定义一个结构体，用于存储 PNG 文件的信息
typedef struct {
// 定义 stbi__png 结构体，包含指向 stbi__context 结构体的指针、指向输入数据的指针、指向扩展数据的指针和深度
typedef struct
{
    stbi__context * s;
    stbi_uc *idata, *expanded, *out;
    int depth;
} stbi__png;

// 定义枚举类型，包含不同的滤波器类型
enum {
    STBI__F_none = 0,
    STBI__F_sub = 1,
    STBI__F_up = 2,
    STBI__F_avg = 3,
    STBI__F_paeth = 4,
    // 用于第一行扫描线的合成滤波器，避免需要一个全为0的虚拟行
    STBI__F_avg_first,
    STBI__F_paeth_first
};

// 定义静态数组，存储第一行扫描线的滤波器类型
static stbi_uc first_row_filter[5] = {STBI__F_none, STBI__F_sub, STBI__F_none, STBI__F_avg_first, STBI__F_paeth_first};

// 定义函数 stbi__paeth，实现 Paeth 滤波算法
static int stbi__paeth(int a, int b, int c) {
    int p = a + b - c;
    // 计算点 p 到 a、b、c 三个点的距离
    int pa = abs(p - a);
    int pb = abs(p - b);
    int pc = abs(p - c);
    // 比较距离，返回距离最近的点
    if (pa <= pb && pa <= pc)
        return a;
    if (pb <= pc)
        return b;
    return c;
}

// 定义深度缩放表
static const stbi_uc stbi__depth_scale_table[9] = {0, 0xff, 0x55, 0, 0x11, 0, 0, 0, 0x01};

// 从压缩后的数据创建 PNG 图像
static int stbi__create_png_image_raw(stbi__png * a, stbi_uc * raw, stbi__uint32 raw_len, int out_n, stbi__uint32 x,
                                      stbi__uint32 y, int depth, int color) {
    // 计算每个像素的字节数
    int bytes = (depth == 16 ? 2 : 1);
    stbi__context * s = a->s;
    stbi__uint32 i, j, stride = x * out_n * bytes;
    stbi__uint32 img_len, img_width_bytes;
    int k;
    // 将 s->img_n 的值复制到本地变量 img_n 中以备后用
    int img_n = s->img_n;

    // 计算输出数据的字节数
    int output_bytes = out_n * bytes;
    // 计算滤镜数据的字节数
    int filter_bytes = img_n * bytes;
    // 设置图像宽度
    int width = x;

    // 断言输出数据的通道数等于图像数据的通道数或者等于图像数据的通道数加一
    STBI_ASSERT(out_n == s->img_n || out_n == s->img_n + 1);
    // 分配输出数据的内存空间，额外的字节用于写入末尾
    a->out = (stbi_uc *)stbi__malloc_mad3(x, y, output_bytes, 0);
    // 如果内存分配失败，则返回错误信息
    if (!a->out)
        return stbi__err("outofmem", "Out of memory");

    // 检查图像数据大小是否合法
    if (!stbi__mad3sizes_valid(img_n, x, depth, 7))
        return stbi__err("too large", "Corrupt PNG");
    // 计算图像每行的字节数
    img_width_bytes = (((img_n * x * depth) + 7) >> 3);
    // 计算图像数据的总字节数
    img_len = (img_width_bytes + 1) * y;

    // 在非交错式 PNG 图像中，我们曾经检查原始数据长度和图像数据长度是否完全匹配，
    // 但问题＃276报告了一种 PNG 图像，其末尾有额外的数据（全为零），
    // 因此始终检查原始数据长度是否小于图像数据长度。
    if (raw_len < img_len)
        # 如果像素不够，返回错误信息
        return stbi__err("not enough pixels", "Corrupt PNG");

    for (j = 0; j < y; ++j) {
        # 获取当前行的输出指针
        stbi_uc * cur = a->out + stride * j;
        # 获取前一行的输出指针
        stbi_uc * prior;
        # 获取滤波器类型
        int filter = *raw++;

        # 如果滤波器类型大于4，返回错误信息
        if (filter > 4)
            return stbi__err("invalid filter", "Corrupt PNG");

        # 如果像素深度小于8，并且图像宽度字节数大于x，返回错误信息
        if (depth < 8) {
            if (img_width_bytes > x)
                return stbi__err("invalid width", "Corrupt PNG");
            # 调整当前指针位置，以便在原地解码
            cur += x * out_n - img_width_bytes; // store output to the rightmost img_len bytes, so we can decode in place
            filter_bytes = 1;
            width = img_width_bytes;
        }
        # 计算前一行的指针位置
        prior = cur - stride; // bugfix: need to compute this after 'cur +=' computation above

        # 如果是第一行，使用特殊的滤波器，不采样前一行
        // 如果 j 等于 0，则将 filter 设置为 first_row_filter[filter]
        if (j == 0)
            filter = first_row_filter[filter];

        // 处理第一个字节
        for (k = 0; k < filter_bytes; ++k) {
            // 根据 filter 的不同情况进行处理
            switch (filter) {
            case STBI__F_none:
                cur[k] = raw[k];
                break;
            case STBI__F_sub:
                cur[k] = raw[k];
                break;
            case STBI__F_up:
                cur[k] = STBI__BYTECAST(raw[k] + prior[k]);
                break;
            case STBI__F_avg:
                cur[k] = STBI__BYTECAST(raw[k] + (prior[k] >> 1));
                break;
            case STBI__F_paeth:
                cur[k] = STBI__BYTECAST(raw[k] + stbi__paeth(0, prior[k], 0));
                break; // 结束当前的 case 分支
            case STBI__F_avg_first: // 如果当前状态是 STBI__F_avg_first
                cur[k] = raw[k]; // 将 raw 数组中的值赋给 cur 数组
                break; // 结束当前的 case 分支
            case STBI__F_paeth_first: // 如果当前状态是 STBI__F_paeth_first
                cur[k] = raw[k]; // 将 raw 数组中的值赋给 cur 数组
                break; // 结束当前的 case 分支
            }
        }

        if (depth == 8) { // 如果深度为 8
            if (img_n != out_n) // 如果输入通道数不等于输出通道数
                cur[img_n] = 255; // 设置当前像素的第 img_n 通道值为 255
            raw += img_n; // 将 raw 指针向后移动 img_n 个位置
            cur += out_n; // 将 cur 指针向后移动 out_n 个位置
            prior += out_n; // 将 prior 指针向后移动 out_n 个位置
        } else if (depth == 16) { // 如果深度为 16
            if (img_n != out_n) { // 如果输入通道数不等于输出通道数
                cur[filter_bytes] = 255;     // 设置当前像素的第 filter_bytes 通道值为 255
                cur[filter_bytes + 1] = 255; // 设置当前像素的第 filter_bytes + 1 通道值为 255
        } 
        // 如果当前像素深度小于8或者图像通道数等于输出通道数
        if (depth < 8 || img_n == out_n) {
            // 计算滤波器的宽度
            int nk = (width - 1) * filter_bytes;
            // 定义一个宏，用于处理不同滤波器类型的情况
#define STBI__CASE(f)                                                                                                          \
    case f:                                                                                                                    \
        for (k = 0; k < nk; ++k)
            // 根据滤波器类型进行不同的处理
            switch (filter) {
            // 如果是"none"滤波器，直接进行内存拷贝
            case STBI__F_none:
                memcpy(cur, raw, nk);
                break;  // 结束当前的 switch 语句块
                STBI__CASE(STBI__F_sub) { cur[k] = STBI__BYTECAST(raw[k] + cur[k - filter_bytes]); }  // 如果当前滤波器类型是 SUB，根据公式计算当前像素的值
                break;  // 结束当前的 switch 语句块
                STBI__CASE(STBI__F_up) { cur[k] = STBI__BYTECAST(raw[k] + prior[k]); }  // 如果当前滤波器类型是 UP，根据公式计算当前像素的值
                break;  // 结束当前的 switch 语句块
                STBI__CASE(STBI__F_avg) { cur[k] = STBI__BYTECAST(raw[k] + ((prior[k] + cur[k - filter_bytes]) >> 1)); }  // 如果当前滤波器类型是 AVG，根据公式计算当前像素的值
                break;  // 结束当前的 switch 语句块
                STBI__CASE(STBI__F_paeth) {
                    cur[k] = STBI__BYTECAST(raw[k] + stbi__paeth(cur[k - filter_bytes], prior[k], prior[k - filter_bytes]));  // 如果当前滤波器类型是 PAETH，根据公式计算当前像素的值
                }
                break;  // 结束当前的 switch 语句块
                STBI__CASE(STBI__F_avg_first) { cur[k] = STBI__BYTECAST(raw[k] + (cur[k - filter_bytes] >> 1)); }  // 如果当前滤波器类型是 AVG_FIRST，根据公式计算当前像素的值
                break;  // 结束当前的 switch 语句块
                STBI__CASE(STBI__F_paeth_first) { cur[k] = STBI__BYTECAST(raw[k] + stbi__paeth(cur[k - filter_bytes], 0, 0)); }  // 如果当前滤波器类型是 PAETH_FIRST，根据公式计算当前像素的值
                break;  // 结束当前的 switch 语句块
            }
#undef STBI__CASE  // 取消之前定义的 STBI__CASE 宏
            raw += nk;  // 将 raw 指针向后移动 nk 个位置
        } else {
            STBI_ASSERT(img_n + 1 == out_n);  // 断言条件 img_n + 1 == out_n 是否成立
# 定义一个宏，用于根据不同的过滤器类型执行不同的操作
#define STBI__CASE(f)                                                                                                          \
    case f:                                                                                                                    \  # 根据传入的过滤器类型执行不同的操作
        for (i = x - 1; i >= 1; --i, cur[filter_bytes] = 255, raw += filter_bytes, cur += output_bytes, prior += output_bytes) \  # 循环遍历每个像素进行处理
            for (k = 0; k < filter_bytes; ++k)  # 遍历每个像素的每个通道进行处理
            switch (filter) {  # 根据传入的过滤器类型进行不同的处理
                STBI__CASE(STBI__F_none) { cur[k] = raw[k]; }  # 如果是无过滤器，则直接赋值
                break;
                STBI__CASE(STBI__F_sub) { cur[k] = STBI__BYTECAST(raw[k] + cur[k - output_bytes]); }  # 如果是sub过滤器，则进行sub操作
                break;
                STBI__CASE(STBI__F_up) { cur[k] = STBI__BYTECAST(raw[k] + prior[k]); }  # 如果是up过滤器，则进行up操作
                break;
                STBI__CASE(STBI__F_avg) { cur[k] = STBI__BYTECAST(raw[k] + ((prior[k] + cur[k - output_bytes]) >> 1)); }  # 如果是avg过滤器，则进行avg操作
                break;
                STBI__CASE(STBI__F_paeth) {
                    cur[k] = STBI__BYTECAST(raw[k] + stbi__paeth(cur[k - output_bytes], prior[k], prior[k - output_bytes]));  # 如果是paeth过滤器，则进行paeth操作
                }
                break;
                STBI__CASE(STBI__F_avg_first) { cur[k] = STBI__BYTECAST(raw[k] + (cur[k - output_bytes] >> 1)); }  # 如果是avg_first过滤器，则进行avg_first操作
                break;
                STBI__CASE(STBI__F_paeth_first) { cur[k] = STBI__BYTECAST(raw[k] + stbi__paeth(cur[k - output_bytes], 0, 0)); }  # 如果是paeth_first过滤器，则进行paeth_first操作
                break;
            }
#undef STBI__CASE

            // the loop above sets the high byte of the pixels' alpha, but for
            // 16 bit png files we also need the low byte set. we'll do that here.
            // 如果深度为16位，需要设置像素的低字节为255
            if (depth == 16) {
                cur = a->out + stride * j; // start at the beginning of the row again
                // 从行的开头开始，为每个像素的低字节设置为255
                for (i = 0; i < x; ++i, cur += output_bytes) {
                    cur[filter_bytes + 1] = 255;
                }
            }
        }
    }

    // we make a separate pass to expand bits to pixels; for performance,
    // this could run two scanlines behind the above code, so it won't
    // intefere with filtering but will still be in the cache.
    // 我们单独进行一次遍历，将位扩展为像素；为了性能考虑，这可能会比上面的代码慢两个扫描行，这样就不会干扰滤波，但仍然会在缓存中。
    if (depth < 8) {
        for (j = 0; j < y; ++j) {
// 定义指向输出数据的指针 cur
stbi_uc * cur = a->out + stride * j;
// 定义指向输入数据的指针 in
stbi_uc * in = a->out + stride * j + x * out_n - img_width_bytes;
// 将1/2/4位数据解压缩成8位缓冲区。这样可以保持常见的8位路径在最小成本下最优化，对于1/2/4位的png保证字节对齐，如果宽度不是8/4/2的倍数，我们将解码虚拟的尾随数据，这些数据将在后续循环中被跳过
stbi_uc scale = (color == 0) ? stbi__depth_scale_table[depth] : 1; // 将灰度值缩放到0..255范围

// 注意，最终的字节可能超出并写入比期望更多的数据。我们可以分配足够的数据，使其永远不会写出内存，但它也可能覆盖下一个扫描线。它可以覆盖下一个扫描线上的非空数据吗？是的，考虑每像素宽度为1位的扫描线。因此，我们需要显式地夹紧最后的数据
if (depth == 4) {
    for (k = x * img_n; k >= 2; k -= 2, ++in) {
        *cur++ = scale * ((*in >> 4));
        *cur++ = scale * ((*in) & 0x0f);
    }
    if (k > 0)
        *cur++ = scale * ((*in >> 4));
}
# 如果深度为2，对每个像素进行处理
} else if (depth == 2) {
    # 对每个像素的每个通道进行处理
    for (k = x * img_n; k >= 4; k -= 4, ++in) {
        # 处理第一个通道
        *cur++ = scale * ((*in >> 6));
        # 处理第二个通道
        *cur++ = scale * ((*in >> 4) & 0x03);
        # 处理第三个通道
        *cur++ = scale * ((*in >> 2) & 0x03);
        # 处理第四个通道
        *cur++ = scale * ((*in) & 0x03);
    }
    # 处理剩余的通道
    if (k > 0)
        *cur++ = scale * ((*in >> 6));
    if (k > 1)
        *cur++ = scale * ((*in >> 4) & 0x03);
    if (k > 2)
        *cur++ = scale * ((*in >> 2) & 0x03);
# 如果深度为1，对每个像素进行处理
} else if (depth == 1) {
    # 对每个像素的每个通道进行处理
    for (k = x * img_n; k >= 8; k -= 8, ++in) {
        # 处理第一个通道
        *cur++ = scale * ((*in >> 7));
        # 处理第二个通道
        *cur++ = scale * ((*in >> 6) & 0x01);
        # 处理第三个通道
        *cur++ = scale * ((*in >> 5) & 0x01);
        # 处理第四个通道
        *cur++ = scale * ((*in >> 4) & 0x01);
        # 处理第五个通道
        *cur++ = scale * ((*in >> 3) & 0x01);
# 将输入数据进行位操作和缩放，并将结果存入cur指向的位置
*cur++ = scale * ((*in >> 2) & 0x01);
*cur++ = scale * ((*in >> 1) & 0x01);
*cur++ = scale * ((*in) & 0x01);
# 如果k大于0，则将输入数据进行位操作和缩放，并将结果存入cur指向的位置
if (k > 0)
    *cur++ = scale * ((*in >> 7));
# 如果k大于1，则将输入数据进行位操作和缩放，并将结果存入cur指向的位置
if (k > 1)
    *cur++ = scale * ((*in >> 6) & 0x01);
# 如果k大于2，则将输入数据进行位操作和缩放，并将结果存入cur指向的位置
if (k > 2)
    *cur++ = scale * ((*in >> 5) & 0x01);
# 如果k大于3，则将输入数据进行位操作和缩放，并将结果存入cur指向的位置
if (k > 3)
    *cur++ = scale * ((*in >> 4) & 0x01);
# 如果k大于4，则将输入数据进行位操作和缩放，并将结果存入cur指向的位置
if (k > 4)
    *cur++ = scale * ((*in >> 3) & 0x01);
# 如果k大于5，则将输入数据进行位操作和缩放，并将结果存入cur指向的位置
if (k > 5)
    *cur++ = scale * ((*in >> 2) & 0x01);
# 如果k大于6，则将输入数据进行位操作和缩放，并将结果存入cur指向的位置
if (k > 6)
    *cur++ = scale * ((*in >> 1) & 0x01);
# 如果图像通道数不等于输出通道数，则执行以下操作
if (img_n != out_n) {
                int q;
                // 定义变量 q
                cur = a->out + stride * j;
                // 将指针 cur 指向 a->out + stride * j
                if (img_n == 1) {
                    // 如果图像通道数为 1
                    for (q = x - 1; q >= 0; --q) {
                        // 从 x-1 开始循环到 0
                        cur[q * 2 + 1] = 255;
                        // 将 cur[q * 2 + 1] 的值设为 255
                        cur[q * 2 + 0] = cur[q];
                        // 将 cur[q * 2 + 0] 的值设为 cur[q]
                    }
                } else {
                    // 如果图像通道数不为 1
                    STBI_ASSERT(img_n == 3);
                    // 断言图像通道数为 3
                    for (q = x - 1; q >= 0; --q) {
                        // 从 x-1 开始循环到 0
                        cur[q * 4 + 3] = 255;
                        // 将 cur[q * 4 + 3] 的值设为 255
                        cur[q * 4 + 2] = cur[q * 3 + 2];
                        // 将 cur[q * 4 + 2] 的值设为 cur[q * 3 + 2]
                        cur[q * 4 + 1] = cur[q * 3 + 1];
                        // 将 cur[q * 4 + 1] 的值设为 cur[q * 3 + 1]
                        cur[q * 4 + 0] = cur[q * 3 + 0];
                        // 将 cur[q * 4 + 0] 的值设为 cur[q * 3 + 0]
                    }
                }
            }
        }
    } else if (depth == 16) {
// 强制将图像数据从大端字节序转换为平台本机字节序。
// 这是在单独的步骤中完成的，因为解码依赖于数据不被修改，但如果小心处理，可能可以在解码过程中每行进行转换。
stbi_uc * cur = a->out;  // 定义指向输出数据的指针
stbi__uint16 * cur16 = (stbi__uint16 *)cur;  // 将输出数据强制转换为16位无符号整数指针

for (i = 0; i < x * y * out_n; ++i, cur16++, cur += 2) {  // 遍历输出数据
    *cur16 = (cur[0] << 8) | cur[1];  // 将每两个字节的数据从大端字节序转换为平台本机字节序
}

return 1;  // 返回成功标志
}

static int stbi__create_png_image(stbi__png * a, stbi_uc * image_data, stbi__uint32 image_data_len, int out_n, int depth,
                                  int color, int interlaced) {
    int bytes = (depth == 16 ? 2 : 1);  // 根据深度确定每个像素的字节数
    int out_bytes = out_n * bytes;  // 计算输出数据的总字节数
    stbi_uc * final;  // 定义最终输出数据的指针
    // 声明整型变量 p
    int p;
    // 如果图像不是交错的，直接返回原始图像数据
    if (!interlaced)
        return stbi__create_png_image_raw(a, image_data, image_data_len, out_n, a->s->img_x, a->s->img_y, depth, color);

    // 对交错的图像进行解交错处理
    // 分配内存空间用于存储解交错后的图像数据
    final = (stbi_uc *)stbi__malloc_mad3(a->s->img_x, a->s->img_y, out_bytes, 0);
    // 如果内存分配失败，返回错误信息
    if (!final)
        return stbi__err("outofmem", "Out of memory");
    // 遍历7个子图像进行解交错处理
    for (p = 0; p < 7; ++p) {
        // 定义每个子图像的原点和间隔
        int xorig[] = {0, 4, 0, 2, 0, 1, 0};
        int yorig[] = {0, 0, 4, 0, 2, 0, 1};
        int xspc[] = {8, 8, 4, 4, 2, 2, 1};
        int yspc[] = {8, 8, 8, 4, 4, 2, 2};
        int i, j, x, y;
        // 计算当前子图像的宽度和高度
        x = (a->s->img_x - xorig[p] + xspc[p] - 1) / xspc[p];
        y = (a->s->img_y - yorig[p] + yspc[p] - 1) / yspc[p];
        // 如果宽度和高度大于0
        if (x && y) {
            // 计算解交错后的图像数据长度
            stbi__uint32 img_len = ((((a->s->img_n * x * depth) + 7) >> 3) + 1) * y;
            // 如果解交错处理失败，返回错误信息
            if (!stbi__create_png_image_raw(a, image_data, image_data_len, out_n, x, y, depth, color)) {
                释放 final 指针指向的内存空间
                返回 0，表示出现错误
            }
            遍历图像的每一行
            for (j = 0; j < y; ++j) {
                遍历图像的每一列
                for (i = 0; i < x; ++i) {
                    计算输出图像中像素的 y 坐标
                    int out_y = j * yspc[p] + yorig[p];
                    计算输出图像中像素的 x 坐标
                    int out_x = i * xspc[p] + xorig[p];
                    将输入图像中指定位置的像素数据复制到输出图像的指定位置
                    memcpy(final + out_y * a->s->img_x * out_bytes + out_x * out_bytes, a->out + (j * x + i) * out_bytes,
                           out_bytes);
                }
            }
            释放 a->out 指针指向的内存空间
            调整图像数据指针和长度，准备处理下一张图像
            image_data += img_len;
            image_data_len -= img_len;
        }
    }
    将 final 指针指向的内存空间赋值给 a->out
    返回 1，表示处理成功
}
static int stbi__compute_transparency(stbi__png * z, stbi_uc tc[3], int out_n) {
    // 获取输入的 PNG 数据
    stbi__context * s = z->s;
    // 计算像素数量
    stbi__uint32 i, pixel_count = s->img_x * s->img_y;
    // 获取输出数据指针
    stbi_uc * p = z->out;

    // 计算基于颜色的透明度，假设输出中已经有 255 作为 alpha 值
    STBI_ASSERT(out_n == 2 || out_n == 4);

    // 如果输出通道数为 2
    if (out_n == 2) {
        // 遍历每个像素
        for (i = 0; i < pixel_count; ++i) {
            // 如果当前像素的颜色与给定颜色相同，则将 alpha 值设为 0，否则设为 255
            p[1] = (p[0] == tc[0] ? 0 : 255);
            p += 2;
        }
    } 
    // 如果输出通道数为 4
    else {
        // 遍历每个像素
        for (i = 0; i < pixel_count; ++i) {
            // 如果当前像素的颜色与给定颜色相同，则将 alpha 值设为 0
            if (p[0] == tc[0] && p[1] == tc[1] && p[2] == tc[2])
                p[3] = 0;
            p += 4;
        }
    }
}
        }
    }
    return 1;
}

static int stbi__compute_transparency16(stbi__png * z, stbi__uint16 tc[3], int out_n) {
    // 获取 PNG 对象的上下文
    stbi__context * s = z->s;
    // 计算像素数量
    stbi__uint32 i, pixel_count = s->img_x * s->img_y;
    // 获取输出数据的指针
    stbi__uint16 * p = (stbi__uint16 *)z->out;

    // 计算基于颜色的透明度，假设输出中已经有 65535 作为 alpha 值
    STBI_ASSERT(out_n == 2 || out_n == 4);

    // 如果输出通道数为 2
    if (out_n == 2) {
        // 遍历每个像素
        for (i = 0; i < pixel_count; ++i) {
            // 如果当前像素的颜色值等于指定的颜色值，则将 alpha 值设为 0，否则设为 65535
            p[1] = (p[0] == tc[0] ? 0 : 65535);
            p += 2;
        }
    } else {
        for (i = 0; i < pixel_count; ++i) {
            // 遍历每个像素
            if (p[0] == tc[0] && p[1] == tc[1] && p[2] == tc[2])
                // 如果当前像素的 RGB 值与目标颜色相同，将 alpha 通道设为 0
                p[3] = 0;
            // 指针移动到下一个像素
            p += 4;
        }
    }
    // 返回 1 表示处理成功
    return 1;
}

static int stbi__expand_png_palette(stbi__png * a, stbi_uc * palette, int len, int pal_img_n) {
    // 计算像素总数
    stbi__uint32 i, pixel_count = a->s->img_x * a->s->img_y;
    // 定义指针变量
    stbi_uc *p, *temp_out, *orig = a->out;

    // 分配内存空间
    p = (stbi_uc *)stbi__malloc_mad2(pixel_count, pal_img_n, 0);
    // 如果内存分配失败，返回错误
    if (p == NULL)
        return stbi__err("outofmem", "Out of memory");

    // 在这里和 free(out) 之间，如果退出会导致内存泄漏
    // 将临时输出指针指向分配的内存空间
    temp_out = p;

# 如果调色板图片的通道数为3
if (pal_img_n == 3) {
    # 遍历每个像素
    for (i = 0; i < pixel_count; ++i) {
        # 计算原始像素值在调色板中的索引
        int n = orig[i] * 4;
        # 将调色板中的RGB值赋给输出像素
        p[0] = palette[n];
        p[1] = palette[n + 1];
        p[2] = palette[n + 2];
        # 指针移动到下一个像素
        p += 3;
    }
} else {
    # 如果调色板图片的通道数不为3
    for (i = 0; i < pixel_count; ++i) {
        # 计算原始像素值在调色板中的索引
        int n = orig[i] * 4;
        # 将调色板中的RGBA值赋给输出像素
        p[0] = palette[n];
        p[1] = palette[n + 1];
        p[2] = palette[n + 2];
        p[3] = palette[n + 3];
        # 指针移动到下一个像素
        p += 4;
    }
}
# 释放a->out指向的内存
STBI_FREE(a->out);
# 将temp_out赋给a->out
a->out = temp_out;
// 使用STBI_NOTUSED宏来避免编译器警告未使用的变量
STBI_NOTUSED(len);

// 返回1，表示成功加载
return 1;
}

// 设置全局变量，用于指示是否在加载时取消预乘
STBIDEF void stbi_set_unpremultiply_on_load(int flag_true_if_should_unpremultiply) {
    stbi__unpremultiply_on_load_global = flag_true_if_should_unpremultiply;
}

// 设置全局变量，用于指示是否需要转换iPhone的PNG格式为RGB格式
STBIDEF void stbi_convert_iphone_png_to_rgb(int flag_true_if_should_convert) {
    stbi__de_iphone_flag_global = flag_true_if_should_convert;
}

// 如果未定义STBI_THREAD_LOCAL，则使用全局变量来表示取消预乘和iPhone标志
#ifndef STBI_THREAD_LOCAL
#define stbi__unpremultiply_on_load stbi__unpremultiply_on_load_global
#define stbi__de_iphone_flag stbi__de_iphone_flag_global
// 如果条件不成立，则定义静态变量 stbi__unpremultiply_on_load_local 和 stbi__unpremultiply_on_load_set
static STBI_THREAD_LOCAL int stbi__unpremultiply_on_load_local, stbi__unpremultiply_on_load_set;
// 如果条件不成立，则定义静态变量 stbi__de_iphone_flag_local 和 stbi__de_iphone_flag_set
static STBI_THREAD_LOCAL int stbi__de_iphone_flag_local, stbi__de_iphone_flag_set;

// 设置是否在加载时解除预乘线程
STBIDEF void stbi_set_unpremultiply_on_load_thread(int flag_true_if_should_unpremultiply) {
    stbi__unpremultiply_on_load_local = flag_true_if_should_unpremultiply;
    stbi__unpremultiply_on_load_set = 1;
}

// 设置是否将 iPhone PNG 转换为 RGB 线程
STBIDEF void stbi_convert_iphone_png_to_rgb_thread(int flag_true_if_should_convert) {
    stbi__de_iphone_flag_local = flag_true_if_should_convert;
    stbi__de_iphone_flag_set = 1;
}

// 定义宏，根据条件返回是否在加载时解除预乘
#define stbi__unpremultiply_on_load                                                                                            \
    (stbi__unpremultiply_on_load_set ? stbi__unpremultiply_on_load_local : stbi__unpremultiply_on_load_global)
// 定义宏，根据条件返回是否将 iPhone PNG 转换为 RGB
#define stbi__de_iphone_flag (stbi__de_iphone_flag_set ? stbi__de_iphone_flag_local : stbi__de_iphone_flag_global)
#endif // STBI_THREAD_LOCAL

// 解析 iPhone PNG 图像
static void stbi__de_iphone(stbi__png * z) {
    # 获取解压缩上下文
    stbi__context * s = z->s;
    # 初始化变量 i 和像素数量
    stbi__uint32 i, pixel_count = s->img_x * s->img_y;
    # 获取输出像素数据指针
    stbi_uc * p = z->out;

    # 如果输出像素通道数为 3，将 bgr 转换为 rgb
    if (s->img_out_n == 3) {
        # 遍历每个像素，交换 bgr 通道顺序为 rgb
        for (i = 0; i < pixel_count; ++i) {
            stbi_uc t = p[0];
            p[0] = p[2];
            p[2] = t;
            p += 3;
        }
    } else {
        # 断言输出像素通道数为 4
        STBI_ASSERT(s->img_out_n == 4);
        # 如果需要在加载时取消预乘，将 bgr 转换为 rgb 并取消预乘
        if (stbi__unpremultiply_on_load) {
            for (i = 0; i < pixel_count; ++i) {
                stbi_uc a = p[3];
                stbi_uc t = p[0];
                # 如果 alpha 不为 0，取消预乘
                if (a) {
                    stbi_uc half = a / 2;
// 如果 alpha 通道不为 0，执行颜色空间转换
if (a != 0) {
    // 对每个像素进行颜色空间转换
    for (i = 0; i < pixel_count; ++i) {
        // 计算新的 RGB 值
        p[0] = (p[2] * 255 + half) / a;
        p[1] = (p[1] * 255 + half) / a;
        p[2] = (t * 255 + half) / a;
        // 移动到下一个像素
        p += 4;
    }
} else {
    // 如果 alpha 通道为 0，执行 bgr 到 rgb 的转换
    for (i = 0; i < pixel_count; ++i) {
        // 交换蓝色和红色通道
        stbi_uc t = p[0];
        p[0] = p[2];
        p[2] = t;
        // 移动到下一个像素
        p += 4;
    }
}
// 定义一个宏，用于将四个字符转换为一个32位的整数
#define STBI__PNG_TYPE(a, b, c, d) (((unsigned)(a) << 24) + ((unsigned)(b) << 16) + ((unsigned)(c) << 8) + (unsigned)(d))

// 解析 PNG 文件
static int stbi__parse_png_file(stbi__png * z, int scan, int req_comp) {
    // 定义一些变量
    stbi_uc palette[1024], pal_img_n = 0; // 调色板和调色板图像的通道数
    stbi_uc has_trans = 0, tc[3] = {0}; // 是否有透明通道和透明通道的颜色
    stbi__uint16 tc16[3]; // 16位的透明通道颜色
    stbi__uint32 ioff = 0, idata_limit = 0, i, pal_len = 0; // 偏移量、数据限制、循环变量、调色板长度
    int first = 1, k, interlace = 0, color = 0, is_iphone = 0; // 是否是第一次解析、循环变量、是否是交错扫描、颜色类型、是否是 iPhone
    stbi__context * s = z->s; // 获取 PNG 数据流

    // 初始化一些变量
    z->expanded = NULL;
    z->idata = NULL;
    z->out = NULL;

    // 检查 PNG 文件头
    if (!stbi__check_png_header(s))
        return 0;

    // 如果只需要扫描类型，则返回1
    if (scan == STBI__SCAN_type)
        return 1;
}
# 进入无限循环
for (;;) {
    # 读取 PNG 文件的块头信息
    stbi__pngchunk c = stbi__get_chunk_header(s);
    # 根据块头信息的类型进行不同的处理
    switch (c.type) {
        # 如果是 iPhone 标记块
        case STBI__PNG_TYPE('C', 'g', 'B', 'I'):
            # 设置 iPhone 标记为 1
            is_iphone = 1;
            # 跳过该块的数据
            stbi__skip(s, c.length);
            break;
        # 如果是图像头块
        case STBI__PNG_TYPE('I', 'H', 'D', 'R'): {
            int comp, filter;
            # 如果不是第一个图像头块，则返回错误
            if (!first)
                return stbi__err("multiple IHDR", "Corrupt PNG");
            first = 0;
            # 如果块长度不为 13，则返回错误
            if (c.length != 13)
                return stbi__err("bad IHDR len", "Corrupt PNG");
            # 读取图像的宽度和高度
            s->img_x = stbi__get32be(s);
            s->img_y = stbi__get32be(s);
            # 如果图像高度超过最大限制，则返回错误
            if (s->img_y > STBI_MAX_DIMENSIONS)
                return stbi__err("too large", "Very large image (corrupt?)");
            # 如果图像宽度超过最大限制
            if (s->img_x > STBI_MAX_DIMENSIONS)
# 如果图像太大，返回错误信息
return stbi__err("too large", "Very large image (corrupt?)");
# 读取图像的深度
z->depth = stbi__get8(s);
# 如果深度不是1、2、4、8、16位，则返回错误信息
if (z->depth != 1 && z->depth != 2 && z->depth != 4 && z->depth != 8 && z->depth != 16)
    return stbi__err("1/2/4/8/16-bit only", "PNG not supported: 1/2/4/8/16-bit only");
# 读取图像的颜色类型
color = stbi__get8(s);
# 如果颜色类型大于6，则返回错误信息
if (color > 6)
    return stbi__err("bad ctype", "Corrupt PNG");
# 如果颜色类型为3且深度为16位，则返回错误信息
if (color == 3 && z->depth == 16)
    return stbi__err("bad ctype", "Corrupt PNG");
# 如果颜色类型为3，则设置调色板图像的通道数为3
if (color == 3)
    pal_img_n = 3;
# 如果颜色类型为奇数，则返回错误信息
else if (color & 1)
    return stbi__err("bad ctype", "Corrupt PNG");
# 读取图像的压缩方法
comp = stbi__get8(s);
# 如果压缩方法不为0，则返回错误信息
if (comp)
    return stbi__err("bad comp method", "Corrupt PNG");
# 读取图像的滤波方法
filter = stbi__get8(s);
# 如果滤波方法不为0，则返回错误信息
if (filter)
    return stbi__err("bad filter method", "Corrupt PNG");
# 读取图像的隔行扫描方法
interlace = stbi__get8(s);
            // 如果交错值大于1，则返回错误信息
            if (interlace > 1)
                return stbi__err("bad interlace method", "Corrupt PNG");
            // 如果图片宽度或高度为0，则返回错误信息
            if (!s->img_x || !s->img_y)
                return stbi__err("0-pixel image", "Corrupt PNG");
            // 如果没有调色板图像，则根据颜色位数计算图片通道数
            s->img_n = (color & 2 ? 3 : 1) + (color & 4 ? 1 : 0);
            // 如果图片尺寸过大，则返回错误信息
            if ((1 << 30) / s->img_x / s->img_n < s->img_y)
                return stbi__err("too large", "Image too large to decode");
            // 如果有调色板图像，则设置图片通道数为1
            s->img_n = 1;
            // 如果图片尺寸过大，则返回错误信息
            if ((1 << 30) / s->img_x / 4 < s->img_y)
                return stbi__err("too large", "Corrupt PNG");
            // 即使有 SCAN_header，也必须扫描以查看是否有 tRNS
            break;
        }

        // 如果是PLTE类型的PNG，则执行以下代码
        case STBI__PNG_TYPE('P', 'L', 'T', 'E'): {
// 如果是第一次读取，返回错误信息
if (first)
    return stbi__err("first not IHDR", "Corrupt PNG");
// 如果颜色表长度大于 256*3，返回错误信息
if (c.length > 256 * 3)
    return stbi__err("invalid PLTE", "Corrupt PNG");
// 计算颜色表长度
pal_len = c.length / 3;
// 如果颜色表长度乘以 3不等于颜色表长度，返回错误信息
if (pal_len * 3 != c.length)
    return stbi__err("invalid PLTE", "Corrupt PNG");
// 遍历颜色表，读取颜色数据
for (i = 0; i < pal_len; ++i) {
    palette[i * 4 + 0] = stbi__get8(s);
    palette[i * 4 + 1] = stbi__get8(s);
    palette[i * 4 + 2] = stbi__get8(s);
    palette[i * 4 + 3] = 255;
}
break;
}

// 处理 tRNS 块
case STBI__PNG_TYPE('t', 'R', 'N', 'S'): {
    // 如果是第一次读取，返回错误信息
    if (first)
        return stbi__err("first not IHDR", "Corrupt PNG");
    // 如果已经有图像数据，处理 tRNS 块
            // 如果在IDAT之后出现tRNS，则PNG文件损坏
            return stbi__err("tRNS after IDAT", "Corrupt PNG");
            // 如果有调色板图像
            if (pal_img_n) {
                // 如果在扫描头部阶段，设置图像通道数为4并返回1
                if (scan == STBI__SCAN_header) {
                    s->img_n = 4;
                    return 1;
                }
                // 如果调色板长度为0，则PNG文件损坏
                if (pal_len == 0)
                    return stbi__err("tRNS before PLTE", "Corrupt PNG");
                // 如果tRNS长度大于调色板长度，则PNG文件损坏
                if (c.length > pal_len)
                    return stbi__err("bad tRNS len", "Corrupt PNG");
                // 设置调色板图像通道数为4
                pal_img_n = 4;
                // 读取tRNS数据并存入调色板中
                for (i = 0; i < c.length; ++i)
                    palette[i * 4 + 3] = stbi__get8(s);
            } else {
                // 如果图像通道数不是奇数，则tRNS带有alpha通道，PNG文件损坏
                if (!(s->img_n & 1))
                    return stbi__err("tRNS with alpha", "Corrupt PNG");
                // 如果tRNS长度不等于图像通道数乘以2，则PNG文件损坏
                if (c.length != (stbi__uint32)s->img_n * 2)
                    return stbi__err("bad tRNS len", "Corrupt PNG");
                // 设置存在透明通道标志为1
                has_trans = 1;
                // 非调色板图像且tRNS为常量alpha值，如果在头部扫描阶段，可以立即停止
                // 如果正在扫描头部信息
                if (scan == STBI__SCAN_header) {
                    // 增加图像通道数
                    ++s->img_n;
                    // 返回1表示成功
                    return 1;
                }
                // 如果深度为16位
                if (z->depth == 16) {
                    // 将值原样复制到tc16数组中
                    for (k = 0; k < s->img_n; ++k)
                        tc16[k] = (stbi__uint16)stbi__get16be(s); // copy the values as-is
                } else {
                    // 对于非16位深度的图像
                    for (k = 0; k < s->img_n; ++k)
                        // 将16位值按位与运算后乘以深度缩放表中的值，存入tc数组中
                        tc[k] = (stbi_uc)(stbi__get16be(s) & 255) *
                                stbi__depth_scale_table[z->depth]; // non 8-bit images will be larger
                }
            }
            break;
        }

        // 如果是IDAT数据块
        case STBI__PNG_TYPE('I', 'D', 'A', 'T'): {
            // 如果是第一个数据块，返回错误
            if (first)
                return stbi__err("first not IHDR", "Corrupt PNG");
            // 如果有调色板图像通道数且调色板长度为0
            // 如果没有调色板，返回错误信息
            return stbi__err("no PLTE", "Corrupt PNG");
            // 如果是 header scan，停止扫描并返回 1
            if (scan == STBI__SCAN_header) {
                if (pal_img_n)
                    s->img_n = pal_img_n;
                return 1;
            }
            // 如果 IDAT 大小超过 2^30 字节，返回错误信息
            if (c.length > (1u << 30))
                return stbi__err("IDAT size limit", "IDAT section larger than 2^30 bytes");
            // 如果偏移量加上 IDAT 大小小于偏移量本身，返回 0
            if ((int)(ioff + c.length) < (int)ioff)
                return 0;
            // 如果偏移量加上 IDAT 大小超过 idata_limit，重新分配内存
            if (ioff + c.length > idata_limit) {
                stbi__uint32 idata_limit_old = idata_limit;
                stbi_uc * p;
                if (idata_limit == 0)
                    idata_limit = c.length > 4096 ? c.length : 4096;
                while (ioff + c.length > idata_limit)
                    idata_limit *= 2;
                STBI_NOTUSED(idata_limit_old);
                p = (stbi_uc *)STBI_REALLOC_SIZED(z->idata, idata_limit_old, idata_limit);
        // 如果指针 p 为空，表示内存不足，返回错误信息
        if (p == NULL)
            return stbi__err("outofmem", "Out of memory");
        // 将指针 p 赋值给 z->idata
        z->idata = p;
    }
    // 如果无法从输入流 s 中获取 c.length 长度的数据，返回错误信息
    if (!stbi__getn(s, z->idata + ioff, c.length))
        return stbi__err("outofdata", "Corrupt PNG");
    // 更新 ioff 的值
    ioff += c.length;
    // 结束 switch 语句
    break;
}

case STBI__PNG_TYPE('I', 'E', 'N', 'D'): {
    stbi__uint32 raw_len, bpl;
    // 如果是第一个数据块，返回错误信息
    if (first)
        return stbi__err("first not IHDR", "Corrupt PNG");
    // 如果扫描模式不是加载模式，返回 1
    if (scan != STBI__SCAN_load)
        return 1;
    // 如果 z->idata 为空，返回错误信息
    if (z->idata == NULL)
        return stbi__err("no IDAT", "Corrupt PNG");
    // 计算每行的字节数，用于初始化解码后的数据大小的初始猜测值，以避免不必要的重新分配
    bpl = (s->img_x * z->depth + 7) / 8; // 每个分量的每行字节数
            // 计算原始数据长度，包括像素和每行的滤波模式
            raw_len = bpl * s->img_y * s->img_n /* pixels */ + s->img_y /* filter mode per row */;
            // 使用 zlib 解码原始数据，并根据需要分配内存
            z->expanded = (stbi_uc *)stbi_zlib_decode_malloc_guesssize_headerflag((char *)z->idata, ioff, raw_len,
                                                                                  (int *)&raw_len, !is_iphone);
            // 如果解码失败，返回错误
            if (z->expanded == NULL)
                return 0; // zlib should set error
            // 释放原始数据内存
            STBI_FREE(z->idata);
            z->idata = NULL;
            // 根据条件设置输出图片的通道数
            if ((req_comp == s->img_n + 1 && req_comp != 3 && !pal_img_n) || has_trans)
                s->img_out_n = s->img_n + 1;
            else
                s->img_out_n = s->img_n;
            // 创建 PNG 图像
            if (!stbi__create_png_image(z, z->expanded, raw_len, s->img_out_n, z->depth, color, interlace))
                return 0;
            // 如果有透明通道，根据深度计算透明度
            if (has_trans) {
                if (z->depth == 16) {
                    if (!stbi__compute_transparency16(z, tc16, s->img_out_n))
                        return 0;
                } else {
                    if (!stbi__compute_transparency(z, tc, s->img_out_n))
                        return 0;
            }
        }
        // 如果是 iPhone 并且有 iPhone 标志，并且输出通道数大于 2，则进行 iPhone 解码
        if (is_iphone && stbi__de_iphone_flag && s->img_out_n > 2)
            stbi__de_iphone(z);
        // 如果有调色板图像
        if (pal_img_n) {
            // 调色板图像的通道数为 3 或 4
            s->img_n = pal_img_n; // 记录实际颜色数
            s->img_out_n = pal_img_n;
            // 如果请求的通道数大于等于 3，则输出通道数为请求的通道数
            if (req_comp >= 3)
                s->img_out_n = req_comp;
            // 如果调色板扩展失败，则返回 0
            if (!stbi__expand_png_palette(z, palette, pal_len, s->img_out_n))
                return 0;
        } else if (has_trans) {
            // 非调色板图像且有透明通道，则图像通道数加一
            ++s->img_n;
        }
        // 释放扩展后的图像数据
        STBI_FREE(z->expanded);
        z->expanded = NULL;
        // 结束 PNG 块，读取并跳过 CRC
        stbi__get32be(s);
            return 1; // 返回1，表示成功
        }

        default:
            // 如果是关键的情况，失败
            if (first) // 如果是第一个出错，返回错误信息
                return stbi__err("first not IHDR", "Corrupt PNG");
            if ((c.type & (1 << 29)) == 0) { // 如果不是已知的 PNG 块类型，返回错误信息
#ifndef STBI_NO_FAILURE_STRINGS
                // 非线程安全
                static char invalid_chunk[] = "XXXX PNG chunk not known"; // 未知的 PNG 块类型
                invalid_chunk[0] = STBI__BYTECAST(c.type >> 24);
                invalid_chunk[1] = STBI__BYTECAST(c.type >> 16);
                invalid_chunk[2] = STBI__BYTECAST(c.type >> 8);
                invalid_chunk[3] = STBI__BYTECAST(c.type >> 0);
#endif
                return stbi__err(invalid_chunk, "PNG not supported: unknown PNG chunk type"); // 返回错误信息，表示不支持未知的 PNG 块类型
            }
            stbi__skip(s, c.length); // 跳过当前块的长度
            break; // 结束当前的 switch 语句
        }
        // PNG块结束，读取并跳过CRC
        stbi__get32be(s);
    }
}

static void * stbi__do_png(stbi__png * p, int * x, int * y, int * n, int req_comp, stbi__result_info * ri) {
    void * result = NULL;
    if (req_comp < 0 || req_comp > 4)
        return stbi__errpuc("bad req_comp", "Internal error");
    // 解析PNG文件
    if (stbi__parse_png_file(p, STBI__SCAN_load, req_comp)) {
        // 如果深度小于等于8，每通道8位
        if (p->depth <= 8)
            ri->bits_per_channel = 8;
        // 如果深度为16，每通道16位
        else if (p->depth == 16)
            ri->bits_per_channel = 16;
        else
            return stbi__errpuc("bad bits_per_channel", "PNG not supported: unsupported color depth");
        result = p->out;
        p->out = NULL;
        // 如果请求的通道数不等于输出图像的通道数
        if (req_comp && req_comp != p->s->img_out_n) {
// 如果每个通道的位数为8，则调用stbi__convert_format函数将结果转换为所需的通道数和格式
if (ri->bits_per_channel == 8)
    result = stbi__convert_format((unsigned char *)result, p->s->img_out_n, req_comp, p->s->img_x, p->s->img_y);
// 如果每个通道的位数不为8，则调用stbi__convert_format16函数将结果转换为所需的通道数和格式
else
    result = stbi__convert_format16((stbi__uint16 *)result, p->s->img_out_n, req_comp, p->s->img_x, p->s->img_y);
// 将输出通道数设置为所需的通道数
p->s->img_out_n = req_comp;
// 如果结果为空，则返回空结果
if (result == NULL)
    return result;

// 将图像的宽度和高度赋值给指针x和y
*x = p->s->img_x;
*y = p->s->img_y;
// 如果n不为空，则将图像的通道数赋值给指针n
if (n)
    *n = p->s->img_n;

// 释放内存并将指针置为空
STBI_FREE(p->out);
p->out = NULL;
STBI_FREE(p->expanded);
p->expanded = NULL;
STBI_FREE(p->idata);
p->idata = NULL;
    // 返回结果
    return result;
}

// 从 stbi__context 中加载 PNG 文件
static void * stbi__png_load(stbi__context * s, int * x, int * y, int * comp, int req_comp, stbi__result_info * ri) {
    // 创建 stbi__png 对象，并将 stbi__context 赋值给 p.s
    stbi__png p;
    p.s = s;
    // 调用 stbi__do_png 函数处理 PNG 文件，并返回结果
    return stbi__do_png(&p, x, y, comp, req_comp, ri);
}

// 检测 stbi__context 中的数据是否为 PNG 格式
static int stbi__png_test(stbi__context * s) {
    int r;
    // 检查 PNG 文件头部信息
    r = stbi__check_png_header(s);
    // 将 stbi__context 指针重置到文件开头
    stbi__rewind(s);
    // 返回检测结果
    return r;
}

// 从 stbi__png 对象中获取 PNG 文件的原始信息
static int stbi__png_info_raw(stbi__png * p, int * x, int * y, int * comp) {
    // 如果无法解析 PNG 文件头部信息，则将 stbi__context 指针重置到文件开头，并返回 0
    if (!stbi__parse_png_file(p, STBI__SCAN_header, 0)) {
        stbi__rewind(p->s);
        return 0;
    }
    // 如果 x 不为空，则将 p->s->img_x 的值赋给 *x
    if (x)
        *x = p->s->img_x;
    // 如果 y 不为空，则将 p->s->img_y 的值赋给 *y
    if (y)
        *y = p->s->img_y;
    // 如果 comp 不为空，则将 p->s->img_n 的值赋给 *comp
    if (comp)
        *comp = p->s->img_n;
    // 返回 1，表示成功
    return 1;
}

// 获取 PNG 图像的信息
static int stbi__png_info(stbi__context * s, int * x, int * y, int * comp) {
    stbi__png p;
    p.s = s;
    // 调用 stbi__png_info_raw 函数获取 PNG 图像的原始信息
    return stbi__png_info_raw(&p, x, y, comp);
}

// 判断 PNG 图像是否为16位
static int stbi__png_is16(stbi__context * s) {
    stbi__png p;
    p.s = s;
    // 如果 stbi__png_info_raw 函数返回 false，则表示不是16位
    if (!stbi__png_info_raw(&p, NULL, NULL, NULL))
        return 0; // 如果条件不满足，返回0
    if (p.depth != 16) { // 如果深度不等于16
        stbi__rewind(p.s); // 重新定位到文件开头
        return 0; // 返回0
    }
    return 1; // 返回1
}
#endif

// Microsoft/Windows BMP image

#ifndef STBI_NO_BMP
static int stbi__bmp_test_raw(stbi__context * s) { // 检测是否为BMP格式的图片
    int r; // 用于存储返回值
    int sz; // 用于存储大小
    if (stbi__get8(s) != 'B') // 如果不是BMP格式，返回0
        return 0;
    if (stbi__get8(s) != 'M') // 如果不是BMP格式，返回0
        return 0;
    stbi__get32le(s); // 丢弃文件大小
    stbi__get16le(s); // 从输入流中读取16位数据并丢弃，这里是丢弃保留字段
    stbi__get16le(s); // 从输入流中读取16位数据并丢弃，这里是丢弃保留字段
    stbi__get32le(s); // 从输入流中读取32位数据并丢弃，这里是丢弃数据偏移量
    sz = stbi__get32le(s); // 从输入流中读取32位数据，存储到变量sz中
    r = (sz == 12 || sz == 40 || sz == 56 || sz == 108 || sz == 124); // 检查sz的值是否符合特定条件，将结果存储到变量r中
    return r; // 返回r的值

static int stbi__bmp_test(stbi__context * s) {
    int r = stbi__bmp_test_raw(s); // 调用stbi__bmp_test_raw函数，将返回值存储到变量r中
    stbi__rewind(s); // 将输入流的位置重置到起始位置
    return r; // 返回r的值
}

// 返回最高位设置的位数，范围在0到31之间
static int stbi__high_bit(unsigned int z) {
    int n = 0; // 初始化变量n为0
    if (z == 0) // 如果z等于0
        return -1; // 返回-1
    if (z >= 0x10000) { // 如果z大于等于65536
    n += 16;  // 将 n 值增加 16
    z >>= 16; // 将 z 右移 16 位
}
if (z >= 0x00100) {  // 如果 z 大于等于十六进制数 0x00100
    n += 8;  // 将 n 值增加 8
    z >>= 8; // 将 z 右移 8 位
}
if (z >= 0x00010) {  // 如果 z 大于等于十六进制数 0x00010
    n += 4;  // 将 n 值增加 4
    z >>= 4; // 将 z 右移 4 位
}
if (z >= 0x00004) {  // 如果 z 大于等于十六进制数 0x00004
    n += 2;  // 将 n 值增加 2
    z >>= 2; // 将 z 右移 2 位
}
if (z >= 0x00002) {  // 如果 z 大于等于十六进制数 0x00002
    n += 1;  // 将 n 值增加 1
    // z >>=  1;  // 将 z 右移 1 位
}
return n;  // 返回 n 的值
// 计算一个32位整数中1的个数
static int stbi__bitcount(unsigned int a) {
    a = (a & 0x55555555) + ((a >> 1) & 0x55555555); // 将奇数位和偶数位相加，最大为2
    a = (a & 0x33333333) + ((a >> 2) & 0x33333333); // 将相邻的两位相加，最大为4
    a = (a + (a >> 4)) & 0x0f0f0f0f;                // 每4位相加，最大为8，现在是8位
    a = (a + (a >> 8));                             // 每8位相加，最大为16
    a = (a + (a >> 16));                            // 每8位相加，最大为32
    return a & 0xff; // 返回低8位
}

// 从v中提取任意对齐的N位值（N=bits），然后将其转换为8位长，并将其分数扩展到完整范围。
static int stbi__shiftsigned(unsigned int v, int shift, int bits) {
    static unsigned int mul_table[9] = {
        0,
        0xff /*0b11111111*/,
        0x55 /*0b01010101*/,
        0x49 /*0b01001001*/,
        0x11 /*0b00010001*/,
```
这段代码是C语言的函数定义，其中包含了一些位操作和移位运算。第一个函数`stbi__bitcount`用于计算一个32位整数中1的个数，通过位操作和移位运算来实现。第二个函数`stbi__shiftsigned`用于从一个整数中提取任意对齐的N位值，并将其转换为8位长，并将其分数扩展到完整范围。在函数内部使用了静态数组`mul_table`来存储一些预先计算好的值。
    # 定义一个包含8个元素的无符号整数数组，表示8位二进制数的值
    0x21 /*0b00100001*/,
    0x41 /*0b01000001*/,
    0x81 /*0b10000001*/,
    0x01 /*0b00000001*/,
};

# 定义一个包含9个元素的无符号整数数组，表示位移量
static unsigned int shift_table[9] = {
    0, 0, 0, 1, 0, 2, 4, 6, 0,
};

# 如果位移量小于0，则将v左移-shift位，否则将v右移shift位
if (shift < 0)
    v <<= -shift;
else
    v >>= shift;

# 断言v的值小于256
STBI_ASSERT(v < 256);

# 将v右移(8 - bits)位
v >>= (8 - bits);

# 断言bits的值在0到8之间
STBI_ASSERT(bits >= 0 && bits <= 8);

# 返回经过乘法表和位移表处理后的值
return (int)((unsigned)v * mul_table[bits]) >> shift_table[bits];
}

# 定义一个结构体，包含bpp、offset和hsz三个成员变量
typedef struct {
    int bpp, offset, hsz;
// 定义了用于存储颜色掩码和额外读取信息的结构体
typedef struct
{
    unsigned int mr, mg, mb, ma, all_a;
    int extra_read;
} stbi__bmp_data;

// 设置颜色掩码的默认值
static int stbi__bmp_set_mask_defaults(stbi__bmp_data * info, int compress) {
    // 如果压缩类型为3，表示BI_BITFIELDS指定了掩码，不需要覆盖默认值
    if (compress == 3)
        return 1;

    // 如果压缩类型为0，根据位深度设置颜色掩码的默认值
    if (compress == 0) {
        if (info->bpp == 16) {
            info->mr = 31u << 10;
            info->mg = 31u << 5;
            info->mb = 31u << 0;
        } else if (info->bpp == 32) {
            info->mr = 0xffu << 16;
            info->mg = 0xffu << 8;
            info->mb = 0xffu << 0;
            info->ma = 0xffu << 24;
            info->all_a = 0; // 如果最终 all_a 为0，则表示加载了alpha通道，但其值全为0
        } else {
            // 否则，使用默认值，即全部为0
            info->mr = info->mg = info->mb = info->ma = 0;
        }
        return 1;
    }
    return 0; // 出错
}

static void * stbi__bmp_parse_header(stbi__context * s, stbi__bmp_data * info) {
    int hsz;
    if (stbi__get8(s) != 'B' || stbi__get8(s) != 'M')
        return stbi__errpuc("not BMP", "Corrupt BMP");
    stbi__get32le(s); // 丢弃文件大小
    stbi__get16le(s); // 丢弃保留字段
    stbi__get16le(s); // 丢弃保留字段
    info->offset = stbi__get32le(s); // 读取图像数据偏移量
    info->hsz = hsz = stbi__get32le(s); // 读取头部大小
    info->mr = info->mg = info->mb = info->ma = 0; // 设置颜色掩码为0
    info->extra_read = 14; // 额外读取14字节
    // 如果偏移量小于0，则返回错误信息
    if (info->offset < 0)
        return stbi__errpuc("bad BMP", "bad BMP");

    // 如果头部大小不是12、40、56、108或124，则返回错误信息
    if (hsz != 12 && hsz != 40 && hsz != 56 && hsz != 108 && hsz != 124)
        return stbi__errpuc("unknown BMP", "BMP type not supported: unknown");
    
    // 根据头部大小不同，读取图片的宽和高
    if (hsz == 12) {
        s->img_x = stbi__get16le(s);
        s->img_y = stbi__get16le(s);
    } else {
        s->img_x = stbi__get32le(s);
        s->img_y = stbi__get32le(s);
    }
    
    // 如果不是1，则返回错误信息
    if (stbi__get16le(s) != 1)
        return stbi__errpuc("bad BMP", "bad BMP");
    
    // 读取图片的位深度
    info->bpp = stbi__get16le(s);
    
    // 如果头部大小不是12，则读取压缩方式，如果是1或2则返回错误信息
    if (hsz != 12) {
        int compress = stbi__get32le(s);
        if (compress == 1 || compress == 2)
            return stbi__errpuc("BMP RLE", "BMP type not supported: RLE");
    }
        // 如果压缩类型大于等于4，表示不支持BMP类型，返回错误信息
        if (compress >= 4)
            return stbi__errpuc("BMP JPEG/PNG",
                                "BMP type not supported: unsupported compression"); // this includes PNG/JPEG modes
        // 如果压缩类型为3且像素深度不是16或32，返回错误信息
        if (compress == 3 && info->bpp != 16 && info->bpp != 32)
            return stbi__errpuc("bad BMP", "bad BMP"); // bitfields requires 16 or 32 bits/pixel
        // 丢弃sizeof字段
        stbi__get32le(s);                              // discard sizeof
        // 丢弃hres字段
        stbi__get32le(s);                              // discard hres
        // 丢弃vres字段
        stbi__get32le(s);                              // discard vres
        // 丢弃colorsused字段
        stbi__get32le(s);                              // discard colorsused
        // 丢弃max important字段
        stbi__get32le(s);                              // discard max important
        // 如果头部大小为40或56
        if (hsz == 40 || hsz == 56) {
            // 如果头部大小为56，再丢弃4个字段
            if (hsz == 56) {
                stbi__get32le(s);
                stbi__get32le(s);
                stbi__get32le(s);
                stbi__get32le(s);
            }
            // 如果像素深度为16或32
            if (info->bpp == 16 || info->bpp == 32) {
                // 如果压缩类型为0，设置BMP掩码默认值
                if (compress == 0) {
                    stbi__bmp_set_mask_defaults(info, compress);
// 如果压缩类型为3，读取颜色掩码信息，并增加额外读取字节数
} else if (compress == 3) {
    info->mr = stbi__get32le(s); // 读取红色掩码
    info->mg = stbi__get32le(s); // 读取绿色掩码
    info->mb = stbi__get32le(s); // 读取蓝色掩码
    info->extra_read += 12; // 增加额外读取字节数
    // 如果红色掩码等于绿色掩码且绿色掩码等于蓝色掩码，返回错误信息
    if (info->mr == info->mg && info->mg == info->mb) {
        // ?!?!?
        return stbi__errpuc("bad BMP", "bad BMP");
    }
} else
    // 如果压缩类型不为3，返回错误信息
    return stbi__errpuc("bad BMP", "bad BMP");
}
} else {
    // V4/V5 头部
    int i;
    // 如果头部大小不为108且不为124，返回错误信息
    if (hsz != 108 && hsz != 124)
        return stbi__errpuc("bad BMP", "bad BMP");
    info->mr = stbi__get32le(s); // 读取红色掩码
    info->mg = stbi__get32le(s); // 读取绿色掩码
            // 从输入流中读取4字节，存入info->mb
            info->mb = stbi__get32le(s);
            // 从输入流中读取4字节，存入info->ma
            info->ma = stbi__get32le(s);
            // 如果压缩模式不是3，根据文档覆盖mr/mg/mb的默认值
            if (compress != 3)
                stbi__bmp_set_mask_defaults(info, compress);
            // 从输入流中读取4字节，丢弃颜色空间信息
            stbi__get32le(s);
            // 循环12次，每次从输入流中读取4字节，丢弃颜色空间参数
            for (i = 0; i < 12; ++i)
                stbi__get32le(s);
            // 如果头部大小为124，依次丢弃渲染意图、配置数据偏移、配置数据大小、保留字段
            if (hsz == 124) {
                stbi__get32le(s);
                stbi__get32le(s);
                stbi__get32le(s);
                stbi__get32le(s);
            }
        }
    }
    // 返回指针1
    return (void *)1;
}

// 从输入流中加载BMP图像
static void * stbi__bmp_load(stbi__context * s, int * x, int * y, int * comp, int req_comp, stbi__result_info * ri) {
    // 输出图像数据
    stbi_uc * out;
    // 定义四个无符号整数变量，分别表示红、绿、蓝、透明度的值，以及一个整数变量 all_a
    unsigned int mr = 0, mg = 0, mb = 0, ma = 0, all_a;
    // 定义一个二维数组，用于存储调色板中每个颜色的 RGBA 值
    stbi_uc pal[256][4];
    // 定义整数变量 psize、i、j、width，以及布尔变量 flip_vertically、pad、target
    int psize = 0, i, j, width;
    int flip_vertically, pad, target;
    // 定义一个结构体变量 info，用于存储 BMP 图像的相关信息
    stbi__bmp_data info;
    // 使用宏定义 STBI_NOTUSED(ri)，表示 ri 参数未使用
    STBI_NOTUSED(ri);

    // 设置 info 结构体中的 all_a 字段为 255
    info.all_a = 255;
    // 调用 stbi__bmp_parse_header 函数解析 BMP 图像头部信息，如果失败则返回 NULL
    if (stbi__bmp_parse_header(s, &info) == NULL)
        return NULL; // 错误码已经设置

    // 根据图像的高度判断是否需要垂直翻转图像
    flip_vertically = ((int)s->img_y) > 0;
    s->img_y = abs((int)s->img_y);

    // 如果图像的高度或宽度超过最大尺寸限制，则返回错误
    if (s->img_y > STBI_MAX_DIMENSIONS)
        return stbi__errpuc("too large", "Very large image (corrupt?)");
    if (s->img_x > STBI_MAX_DIMENSIONS)
        return stbi__errpuc("too large", "Very large image (corrupt?)");

    // 将 info 结构体中的 mr 字段赋值给变量 mr
    mr = info.mr;
    # 从 info 对象中获取 mg、mb、ma、all_a 的值
    mg = info.mg;
    mb = info.mb;
    ma = info.ma;
    all_a = info.all_a;

    # 根据 info 对象中的 hsz 和 bpp 计算 psize
    if (info.hsz == 12) {
        if (info.bpp < 24)
            psize = (info.offset - info.extra_read - 24) / 3;
    } else {
        if (info.bpp < 16)
            psize = (info.offset - info.extra_read - info.hsz) >> 2;
    }

    # 如果 psize 为 0，则根据一定条件判断文件是否损坏，如果是则返回错误信息
    if (psize == 0) {
        # 计算已经读取的字节数和头部限制
        int bytes_read_so_far = s->callback_already_read + (int)(s->img_buffer - s->img_buffer_original);
        int header_limit = 1024;        # 目前实际读取的最大字节数为 256 字节以下。
        int extra_data_limit = 256 * 4; # 通常在这里是一个调色板；256 个条目 * 4 字节是其最大大小。
        # 如果已读取的字节数小于等于 0 或者大于头部限制，则返回错误信息
        if (bytes_read_so_far <= 0 || bytes_read_so_far > header_limit) {
            return stbi__errpuc("bad header", "Corrupt BMP");
        }
        // 我们已经确定 bytes_read_so_far 是正数且合理的。
        // 这个测试的前半部分拒绝了太小的正数偏移量，或者负数，并且保证 info.offset >= bytes_read_so_far > 0。这反过来又确保了在测试的后半部分计算的数字不会溢出。
        if (info.offset < bytes_read_so_far || info.offset - bytes_read_so_far > extra_data_limit) {
            return stbi__errpuc("bad offset", "Corrupt BMP");
        } else {
            stbi__skip(s, info.offset - bytes_read_so_far);
        }
    }

    if (info.bpp == 24 && ma == 0xff000000)
        s->img_n = 3;
    else
        s->img_n = ma ? 4 : 3;
    if (req_comp && req_comp >= 3) // 我们可以直接解码 3 或 4 通道
        target = req_comp;
    else
        target = s->img_n; // 如果他们想要单色，我们将进行后期转换
```
在这个示例中，我们对代码进行了注释，解释了每个语句的作用和意图。
// 检查目标大小是否合理
if (!stbi__mad3sizes_valid(target, s->img_x, s->img_y, 0))
    return stbi__errpuc("too large", "Corrupt BMP");

// 分配内存以存储解压后的图像数据
out = (stbi_uc *)stbi__malloc_mad3(target, s->img_x, s->img_y, 0);
if (!out)
    return stbi__errpuc("outofmem", "Out of memory");

// 如果图像的位深度小于16，则处理调色板
if (info.bpp < 16) {
    int z = 0;
    // 检查调色板大小是否合理
    if (psize == 0 || psize > 256) {
        STBI_FREE(out);
        return stbi__errpuc("invalid", "Corrupt BMP");
    }
    // 读取调色板中的颜色信息
    for (i = 0; i < psize; ++i) {
        pal[i][2] = stbi__get8(s);
        pal[i][1] = stbi__get8(s);
        pal[i][0] = stbi__get8(s);
        // 如果信息头大小不为12，则跳过一个字节
        if (info.hsz != 12)
            stbi__get8(s);
    }
}
        // 设置调色板中第i个颜色的alpha通道为255
        pal[i][3] = 255;
        // 跳过多余的数据
        stbi__skip(s, info.offset - info.extra_read - info.hsz - psize * (info.hsz == 12 ? 3 : 4));
        // 根据不同的位深度计算图像宽度
        if (info.bpp == 1)
            width = (s->img_x + 7) >> 3;
        else if (info.bpp == 4)
            width = (s->img_x + 1) >> 1;
        else if (info.bpp == 8)
            width = s->img_x;
        else {
            // 释放内存并返回错误信息
            STBI_FREE(out);
            return stbi__errpuc("bad bpp", "Corrupt BMP");
        }
        // 计算每行像素数据的填充字节数
        pad = (-width) & 3;
        // 处理位深度为1的情况
        if (info.bpp == 1) {
            for (j = 0; j < (int)s->img_y; ++j) {
                int bit_offset = 7, v = stbi__get8(s);
                for (i = 0; i < (int)s->img_x; ++i) {
                    int color = (v >> bit_offset) & 0x1;
                    out[z++] = pal[color][0];
                    out[z++] = pal[color][1];  // 将颜色值的绿色分量存入输出数组
                    out[z++] = pal[color][2];  // 将颜色值的蓝色分量存入输出数组
                    if (target == 4)  // 如果目标格式是4
                        out[z++] = 255;  // 将 alpha 值设为 255
                    if (i + 1 == (int)s->img_x)  // 如果已经处理完一行
                        break;  // 跳出循环
                    if ((--bit_offset) < 0) {  // 如果位偏移小于0
                        bit_offset = 7;  // 重置位偏移
                        v = stbi__get8(s);  // 从输入流中获取一个字节
                    }
                }
                stbi__skip(s, pad);  // 跳过填充字节
            }
        } else {
            for (j = 0; j < (int)s->img_y; ++j) {  // 遍历图像的每一行
                for (i = 0; i < (int)s->img_x; i += 2) {  // 遍历图像的每一列，每次增加2
                    int v = stbi__get8(s), v2 = 0;  // 从输入流中获取一个字节，初始化第二个字节为0
                    if (info.bpp == 4) {  // 如果每像素位数为4
                        v2 = v & 15;  // 获取低4位作为第二个像素的值
                        v >>= 4;  // 右移4位，获取高4位作为第一个像素的值
                    }
                    // 将调色板中的颜色值写入输出数组
                    out[z++] = pal[v][0];
                    out[z++] = pal[v][1];
                    out[z++] = pal[v][2];
                    // 如果目标色彩通道为4，则将alpha通道值设为255
                    if (target == 4)
                        out[z++] = 255;
                    // 如果当前行已经读取完毕，则跳出循环
                    if (i + 1 == (int)s->img_x)
                        break;
                    // 从输入流中读取下一个像素值
                    v = (info.bpp == 8) ? stbi__get8(s) : v2;
                    // 将调色板中的颜色值写入输出数组
                    out[z++] = pal[v][0];
                    out[z++] = pal[v][1];
                    out[z++] = pal[v][2];
                    // 如果目标色彩通道为4，则将alpha通道值设为255
                    if (target == 4)
                        out[z++] = 255;
                }
                // 跳过填充字节
                stbi__skip(s, pad);
            }
        }
    } else {
        // 初始化位移和计数变量
        int rshift = 0, gshift = 0, bshift = 0, ashift = 0, rcount = 0, gcount = 0, bcount = 0, acount = 0;
        # 初始化变量 z 和 easy
        int z = 0;
        int easy = 0;
        # 跳过指定字节数
        stbi__skip(s, info.offset - info.extra_read - info.hsz);
        # 根据像素位数计算图像宽度
        if (info.bpp == 24)
            width = 3 * s->img_x;
        else if (info.bpp == 16)
            width = 2 * s->img_x;
        else /* bpp = 32 and pad = 0 */
            width = 0;
        # 计算图像每行的填充字节数
        pad = (-width) & 3;
        # 根据像素位数判断是否为简单格式
        if (info.bpp == 24) {
            easy = 1;
        } else if (info.bpp == 32) {
            if (mb == 0xff && mg == 0xff00 && mr == 0x00ff0000 && ma == 0xff000000)
                easy = 2;
        }
        # 如果不是简单格式，检查颜色掩码是否有效
        if (!easy) {
            if (!mr || !mg || !mb) {
                STBI_FREE(out);
                return stbi__errpuc("bad masks", "Corrupt BMP");
            }
            // 将位移量 amt 右移，将高位放在位置 #7
            rshift = stbi__high_bit(mr) - 7;
            // 统计 mr 中的位数
            rcount = stbi__bitcount(mr);
            // 将位移量 amt 右移，将高位放在位置 #7
            gshift = stbi__high_bit(mg) - 7;
            // 统计 mg 中的位数
            gcount = stbi__bitcount(mg);
            // 将位移量 amt 右移，将高位放在位置 #7
            bshift = stbi__high_bit(mb) - 7;
            // 统计 mb 中的位数
            bcount = stbi__bitcount(mb);
            // 将位移量 amt 右移，将高位放在位置 #7
            ashift = stbi__high_bit(ma) - 7;
            // 统计 ma 中的位数
            acount = stbi__bitcount(ma);
            // 如果 rcount、gcount、bcount、acount 中有任何一个大于 8，则返回错误
            if (rcount > 8 || gcount > 8 || bcount > 8 || acount > 8) {
                // 释放内存并返回错误信息
                STBI_FREE(out);
                return stbi__errpuc("bad masks", "Corrupt BMP");
            }
        }
        // 遍历图像的每一行
        for (j = 0; j < (int)s->img_y; ++j) {
            // 如果是简单模式
            if (easy) {
                // 遍历图像的每一列
                for (i = 0; i < (int)s->img_x; ++i) {
                    // 定义一个无符号字符变量 a
                    unsigned char a;
                    // 从输入流中读取一个字节，并存入 out 数组中
                    out[z + 2] = stbi__get8(s);
                    // 从输入流中读取一个字节并存入输出数组的下一个位置
                    out[z + 1] = stbi__get8(s);
                    // 从输入流中读取一个字节并存入输出数组的下一个位置
                    out[z + 0] = stbi__get8(s);
                    // 更新输出数组的索引
                    z += 3;
                    // 如果 easy 等于 2，则从输入流中读取一个字节并存入变量 a，否则存入 255
                    a = (easy == 2 ? stbi__get8(s) : 255);
                    // 将 a 的值与 all_a 进行按位或运算
                    all_a |= a;
                    // 如果目标通道数为 4，则将 a 存入输出数组的下一个位置
                    if (target == 4)
                        out[z++] = a;
                }
            } else {
                // 获取图像每个像素的位深度
                int bpp = info.bpp;
                // 遍历图像的每个像素
                for (i = 0; i < (int)s->img_x; ++i) {
                    // 如果位深度为 16，则从输入流中读取一个 16 位的像素值，否则读取一个 32 位的像素值
                    stbi__uint32 v = (bpp == 16 ? (stbi__uint32)stbi__get16le(s) : stbi__get32le(s));
                    unsigned int a;
                    // 将像素值中的红色通道值存入输出数组的下一个位置
                    out[z++] = STBI__BYTECAST(stbi__shiftsigned(v & mr, rshift, rcount));
                    // 将像素值中的绿色通道值存入输出数组的下一个位置
                    out[z++] = STBI__BYTECAST(stbi__shiftsigned(v & mg, gshift, gcount));
                    // 将像素值中的蓝色通道值存入输出数组的下一个位置
                    out[z++] = STBI__BYTECAST(stbi__shiftsigned(v & mb, bshift, bcount));
                    // 如果存在 alpha 通道，则从像素值中提取 alpha 通道值，否则存入 255
                    a = (ma ? stbi__shiftsigned(v & ma, ashift, acount) : 255);
                    // 将 a 的值与 all_a 进行按位或运算
                    all_a |= a;
                    // 如果目标通道数为 4，则将 a 存入输出数组的下一个位置
                    if (target == 4)
                        out[z++] = STBI__BYTECAST(a);
    // 如果 alpha 通道全为 0，则替换为全为 255
    if (target == 4 && all_a == 0)
        for (i = 4 * s->img_x * s->img_y - 1; i >= 0; i -= 4)
            out[i] = 255;

    // 如果需要垂直翻转图像
    if (flip_vertically) {
        // 临时变量 t
        stbi_uc t;
        // 遍历图像的一半高度
        for (j = 0; j < (int)s->img_y >> 1; ++j) {
            // 指向当前行和对称行的指针
            stbi_uc * p1 = out + j * s->img_x * target;
            stbi_uc * p2 = out + (s->img_y - 1 - j) * s->img_x * target;
            // 交换当前行和对称行的像素数据
            for (i = 0; i < (int)s->img_x * target; ++i) {
                t = p1[i];
                p1[i] = p2[i];
                p2[i] = t;
// 如果请求的压缩格式与目标格式不同，则将输出图像转换为请求的压缩格式
if (req_comp && req_comp != target) {
    out = stbi__convert_format(out, target, req_comp, s->img_x, s->img_y);
    // 如果转换失败，则返回空指针，并且stbi__convert_format会释放输入
    if (out == NULL)
        return out;
}

// 将图像的宽度赋值给指针变量x
*x = s->img_x;
// 将图像的高度赋值给指针变量y
*y = s->img_y;
// 如果压缩格式存在，则将图像的通道数赋值给指针变量comp
if (comp)
    *comp = s->img_n;
// 返回输出图像
return out;
}
#endif

// Targa Truevision - TGA
// 作者：Jonathan Dummer
#ifndef STBI_NO_TGA
// 如果没有定义 STBI_NO_TGA，则执行以下代码

// 根据像素位数和是否灰度，确定返回的颜色通道数，如果是 16 位 RGB，则设置 is_rgb16 为 1
static int stbi__tga_get_comp(int bits_per_pixel, int is_grey, int * is_rgb16) {
    // 只允许 RGB 或 RGBA（包括 16 位）或灰度
    if (is_rgb16)
        *is_rgb16 = 0;
    switch (bits_per_pixel) {
    case 8:
        return STBI_grey;
    case 16:
        if (is_grey)
            return STBI_grey_alpha;
        // 继续执行下面的代码
    case 15:
        if (is_rgb16)
            *is_rgb16 = 1;
        return STBI_rgb;
    case 24: // 继续执行下面的代码
    case 32:
        return bits_per_pixel / 8;
```

    default:
        return 0; // 如果不符合任何条件，返回0
    }
}

static int stbi__tga_info(stbi__context * s, int * x, int * y, int * comp) {
    int tga_w, tga_h, tga_comp, tga_image_type, tga_bits_per_pixel, tga_colormap_bpp;
    int sz, tga_colormap_type;
    stbi__get8(s);                     // 丢弃偏移量
    tga_colormap_type = stbi__get8(s); // 颜色映射类型
    if (tga_colormap_type > 1) {
        stbi__rewind(s);
        return 0; // 只允许RGB或索引颜色映射
    }
    tga_image_type = stbi__get8(s); // 图像类型
    if (tga_colormap_type == 1) {   // 颜色映射（调色板）图像
        if (tga_image_type != 1 && tga_image_type != 9) {
            stbi__rewind(s);
            return 0; // 只允许类型1或9的颜色映射图像
        }
        stbi__skip(s, 4);   // 跳过第一个调色板条目的索引和条目数
        sz = stbi__get8(s); // 检查调色板颜色条目的位数
        if ((sz != 8) && (sz != 15) && (sz != 16) && (sz != 24) && (sz != 32)) {
            stbi__rewind(s);
            return 0;
        }
        stbi__skip(s, 4); // 跳过图像的 x 和 y 起始位置
        tga_colormap_bpp = sz;
    } else { // "normal" image w/o colormap - 只允许 RGB 或灰度，带有或不带有 RLE 压缩
        if ((tga_image_type != 2) && (tga_image_type != 3) && (tga_image_type != 10) && (tga_image_type != 11)) {
            stbi__rewind(s);
            return 0; // 只允许 RGB 或灰度，带有或不带有 RLE 压缩
        }
        stbi__skip(s, 9); // 跳过调色板规范和图像的 x/y 起始位置
        tga_colormap_bpp = 0;
    }
    tga_w = stbi__get16le(s);
    if (tga_w < 1) {
        stbi__rewind(s);
        return 0; // 测试宽度
    }
    // 读取TGA文件的高度
    tga_h = stbi__get16le(s);
    // 如果高度小于1，则回退到文件开头并返回0
    if (tga_h < 1) {
        stbi__rewind(s);
        return 0; // test height
    }
    // 读取TGA文件的每像素位数
    tga_bits_per_pixel = stbi__get8(s); // bits per pixel
    // 忽略alpha位
    stbi__get8(s);                      // ignore alpha bits
    // 如果使用颜色映射表
    if (tga_colormap_bpp != 0) {
        // 当使用颜色映射表时，tga_bits_per_pixel是索引的大小
        // 我认为除了8位或16位索引之外没有其他意义
        if ((tga_bits_per_pixel != 8) && (tga_bits_per_pixel != 16)) {
            // 回退到文件开头并返回0
            stbi__rewind(s);
            return 0;
        }
        // 获取颜色通道数
        tga_comp = stbi__tga_get_comp(tga_colormap_bpp, 0, NULL);
    } else {
        // 获取颜色通道数
        tga_comp = stbi__tga_get_comp(tga_bits_per_pixel, (tga_image_type == 3) || (tga_image_type == 11), NULL);
    }
    // 如果颜色通道数为空
    if (!tga_comp) {
// 将流的读取位置重置到起始位置
stbi__rewind(s);
// 返回0，表示读取成功
return 0;
}
// 如果x不为空，则将tga_w的值赋给x
if (x)
    *x = tga_w;
// 如果y不为空，则将tga_h的值赋给y
if (y)
    *y = tga_h;
// 如果comp不为空，则将tga_comp的值赋给comp
if (comp)
    *comp = tga_comp;
// 返回1，表示通过了所有检查
return 1; 
}

// 检测TGA文件格式是否符合要求
static int stbi__tga_test(stbi__context * s) {
    int res = 0;
    int sz, tga_color_type;
    // 丢弃Offset
    stbi__get8(s);                  
    // 读取颜色类型
    tga_color_type = stbi__get8(s); 
    // 如果颜色类型大于1，则跳转到错误处理
    if (tga_color_type > 1)
        goto errorEnd;         
    // 读取图像类型
    sz = stbi__get8(s);        
    if (tga_color_type == 1) { // 如果 TGA 图像的颜色类型为1，表示使用调色板
        if (sz != 1 && sz != 9)
            goto errorEnd;  // 如果图像类型不是1或9，跳转到错误处理部分
        stbi__skip(s, 4);   // 跳过第一个调色板条目的索引和调色板条目数
        sz = stbi__get8(s); // 获取调色板颜色条目的位数
        if ((sz != 8) && (sz != 15) && (sz != 16) && (sz != 24) && (sz != 32))
            goto errorEnd;  // 如果调色板颜色条目的位数不符合要求，跳转到错误处理部分
        stbi__skip(s, 4); // 跳过图像的 x 和 y 起始位置
    } else {              // 如果不是使用调色板的 "普通" 图像
        if ((sz != 2) && (sz != 3) && (sz != 10) && (sz != 11))
            goto errorEnd; // 只允许 RGB 或灰度图像，且可能使用 RLE 压缩
        stbi__skip(s, 9);  // 跳过调色板规范和图像的 x/y 起始位置
    }
    if (stbi__get16le(s) < 1)
        goto errorEnd; // 检查图像宽度是否大于等于1
    if (stbi__get16le(s) < 1)
        goto errorEnd;  // 检查图像高度是否大于等于1
    sz = stbi__get8(s); // 获取每像素位数
    if ((tga_color_type == 1) && (sz != 8) && (sz != 16))
        goto errorEnd; // 对于使用调色板的图像，每像素位数应该等于调色板索引的位数
// 检查 sz 的取值，如果不是指定的几个值之一，跳转到错误处理的标签
if ((sz != 8) && (sz != 15) && (sz != 16) && (sz != 24) && (sz != 32))
    goto errorEnd;

res = 1; // 如果程序执行到这一步，说明一切正常，可以返回 1 而不是 0

errorEnd:
stbi__rewind(s); // 将文件指针重新定位到起始位置
return res; // 返回结果

// 读取 16 位值并转换为 24 位 RGB
static void stbi__tga_read_rgb16(stbi__context * s, stbi_uc * out) {
    stbi__uint16 px = (stbi__uint16)stbi__get16le(s); // 读取 16 位值
    stbi__uint16 fiveBitMask = 31; // 定义 5 位掩码
    // 有 3 个通道，每个通道有 5 位
    int r = (px >> 10) & fiveBitMask; // 取出红色通道的值
    int g = (px >> 5) & fiveBitMask; // 取出绿色通道的值
    int b = px & fiveBitMask; // 取出蓝色通道的值
    // 注意，这里保存的数据是按照 RGB(A) 的顺序，所以后续不需要再进行通道交换
    out[0] = (stbi_uc)((r * 255) / 31); // 将 5 位值转换为 8 位值并保存到输出数组中
}
    // 将g乘以255再除以31，将结果转换为8位无符号整数，存入out数组的第1个位置
    out[1] = (stbi_uc)((g * 255) / 31);
    // 将b乘以255再除以31，将结果转换为8位无符号整数，存入out数组的第2个位置
    out[2] = (stbi_uc)((b * 255) / 31);

    // 一些人声称最高有效位可能用于alpha通道
    // 但这样做会导致16位测试图像完全透明
    // 因此，我们将所有15和16位的TGA图像视为没有alpha通道的RGB图像
    // 所以让我们将所有15和16位的TGA图像视为没有alpha通道的RGB图像
}

static void * stbi__tga_load(stbi__context * s, int * x, int * y, int * comp, int req_comp, stbi__result_info * ri) {
    // 读取TGA文件头信息
    int tga_offset = stbi__get8(s);
    int tga_indexed = stbi__get8(s);
    int tga_image_type = stbi__get8(s);
    int tga_is_RLE = 0;
    int tga_palette_start = stbi__get16le(s);
    int tga_palette_len = stbi__get16le(s);
    int tga_palette_bits = stbi__get8(s);
    int tga_x_origin = stbi__get16le(s);
    int tga_y_origin = stbi__get16le(s);
    # 读取 TGA 图像的宽度
    int tga_width = stbi__get16le(s);
    # 读取 TGA 图像的高度
    int tga_height = stbi__get16le(s);
    # 读取 TGA 图像的每像素位数
    int tga_bits_per_pixel = stbi__get8(s);
    # 定义变量 tga_comp 和 tga_rgb16，并初始化为 0
    int tga_comp, tga_rgb16 = 0;
    # 读取 TGA 图像是否倒置
    int tga_inverted = stbi__get8(s);
    # 定义变量 tga_data 和 tga_palette，并初始化为 NULL
    unsigned char * tga_data;
    unsigned char * tga_palette = NULL;
    # 定义变量 i 和 j
    int i, j;
    # 定义数组 raw_data，并初始化为 {0}
    unsigned char raw_data[4] = {0};
    # 定义变量 RLE_count 和 RLE_repeating，并初始化为 0
    int RLE_count = 0;
    int RLE_repeating = 0;
    # 定义变量 read_next_pixel，并初始化为 1
    int read_next_pixel = 1;
    # 忽略变量 ri
    STBI_NOTUSED(ri);
    # 忽略变量 tga_x_origin
    STBI_NOTUSED(tga_x_origin); // @TODO
    # 忽略变量 tga_y_origin
    STBI_NOTUSED(tga_y_origin); // @TODO

    # 如果 TGA 图像的高度超过最大尺寸限制，则返回错误信息
    if (tga_height > STBI_MAX_DIMENSIONS)
        return stbi__errpuc("too large", "Very large image (corrupt?)");
    // 如果 TGA 图像宽度超过最大尺寸限制，则返回错误信息
    if (tga_width > STBI_MAX_DIMENSIONS)
        return stbi__errpuc("too large", "Very large image (corrupt?)");

    // 对 TGA 图像类型进行一些预处理
    if (tga_image_type >= 8) {
        tga_image_type -= 8; // 减去8
        tga_is_RLE = 1; // 设置 RLE 标志为1
    }
    tga_inverted = 1 - ((tga_inverted >> 5) & 1); // 对 tga_inverted 进行位运算

    // 如果是调色板图像，则使用调色板的位数
    if (tga_indexed)
        tga_comp = stbi__tga_get_comp(tga_palette_bits, 0, &tga_rgb16);
    else
        tga_comp = stbi__tga_get_comp(tga_bits_per_pixel, (tga_image_type == 3), &tga_rgb16);

    // 如果 tga_comp 为假，说明出现了不应该出现的情况，因为 stbi__tga_test() 应该已经确保了基本的一致性
    if (!tga_comp)
        return stbi__errpuc("bad format", "Can't find out TGA pixelformat");

    // TGA 信息
    // 将 tga_width 的值赋给指针 x
    *x = tga_width;
    // 将 tga_height 的值赋给指针 y
    *y = tga_height;
    // 如果 comp 存在，则将 tga_comp 的值赋给指针 comp
    if (comp)
        *comp = tga_comp;

    // 检查 tga_width、tga_height 和 tga_comp 的乘积是否合法
    if (!stbi__mad3sizes_valid(tga_width, tga_height, tga_comp, 0))
        // 如果不合法，返回错误信息
        return stbi__errpuc("too large", "Corrupt TGA");

    // 分配内存给 tga_data
    tga_data = (unsigned char *)stbi__malloc_mad3(tga_width, tga_height, tga_comp, 0);
    // 如果分配内存失败，返回错误信息
    if (!tga_data)
        return stbi__errpuc("outofmem", "Out of memory");

    // 跳过到数据的起始位置（通常偏移量为 0）
    stbi__skip(s, tga_offset);

    // 如果不是索引色、不是 RLE 压缩、不是 RGB16 格式
    if (!tga_indexed && !tga_is_RLE && !tga_rgb16) {
        // 遍历每一行
        for (i = 0; i < tga_height; ++i) {
            // 如果 tga_inverted 为真，则计算倒序的行数，否则使用 i
            int row = tga_inverted ? tga_height - i - 1 : i;
            // 获取当前行的数据
            stbi_uc * tga_row = tga_data + row * tga_width * tga_comp;
            // 从输入流中读取 tga_width * tga_comp 个字节到 tga_row
            stbi__getn(s, tga_row, tga_width * tga_comp);
        }
    } else {
        // 如果需要加载调色板
        if (tga_indexed) {
            // 如果调色板长度为0，则返回错误
            if (tga_palette_len == 0) { /* you have to have at least one entry! */
                STBI_FREE(tga_data);
                return stbi__errpuc("bad palette", "Corrupt TGA");
            }

            // 跳过需要跳过的数据（通常偏移量为0）
            stbi__skip(s, tga_palette_start);
            // 加载调色板
            tga_palette = (unsigned char *)stbi__malloc_mad2(tga_palette_len, tga_comp, 0);
            // 如果内存分配失败，则返回错误
            if (!tga_palette) {
                STBI_FREE(tga_data);
                return stbi__errpuc("outofmem", "Out of memory");
            }
            // 如果是RGB16格式，则断言调色板的颜色分量为RGB
            if (tga_rgb16) {
                stbi_uc * pal_entry = tga_palette;
                STBI_ASSERT(tga_comp == STBI_rgb);
        // 遍历调色板，读取RGB16颜色数据
        for (i = 0; i < tga_palette_len; ++i) {
            stbi__tga_read_rgb16(s, pal_entry);
            pal_entry += tga_comp;
        }
        // 如果没有调色板，直接读取颜色数据
        } else if (!stbi__getn(s, tga_palette, tga_palette_len * tga_comp)) {
            // 释放内存并返回错误信息
            STBI_FREE(tga_data);
            STBI_FREE(tga_palette);
            return stbi__errpuc("bad palette", "Corrupt TGA");
        }
        // 加载数据
        for (i = 0; i < tga_width * tga_height; ++i) {
            // 如果处于RLE模式，需要获取RLE命令
            if (tga_is_RLE) {
                if (RLE_count == 0) {
                    // 获取下一个字节作为RLE命令
                    int RLE_cmd = stbi__get8(s);
                    RLE_count = 1 + (RLE_cmd & 127);
                    RLE_repeating = RLE_cmd >> 7;
                    read_next_pixel = 1;
            } else if (!RLE_repeating) {
                // 如果不是重复的 RLE，设置读取下一个像素的标志为1
                read_next_pixel = 1;
            }
        } else {
            // 如果不是 RLE 压缩，设置读取下一个像素的标志为1
            read_next_pixel = 1;
        }
        // 如果需要读取像素，立即执行
        if (read_next_pixel) {
            // 加载我们已经有的数据
            if (tga_indexed) {
                // 读取索引，然后进行查找
                int pal_idx = (tga_bits_per_pixel == 8) ? stbi__get8(s) : stbi__get16le(s);
                // 如果索引超出调色板长度，将其设置为0
                if (pal_idx >= tga_palette_len) {
                    pal_idx = 0;
                }
                pal_idx *= tga_comp;
                for (j = 0; j < tga_comp; ++j) {
                    raw_data[j] = tga_palette[pal_idx + j];
                }
                } else if (tga_rgb16) {
                    // 如果是 RGB16 格式，确保 tga_comp 是 STBI_rgb，然后调用 stbi__tga_read_rgb16 函数读取数据
                    STBI_ASSERT(tga_comp == STBI_rgb);
                    stbi__tga_read_rgb16(s, raw_data);
                } else {
                    // 如果不是 RGB16 格式，按照 tga_comp 的值逐个读取数据
                    // 逐个读取数据并存入 raw_data 数组中
                    for (j = 0; j < tga_comp; ++j) {
                        raw_data[j] = stbi__get8(s);
                    }
                }
                // 清除读取标志，准备读取下一个像素
                read_next_pixel = 0;
            } // 结束像素读取

            // 复制数据
            for (j = 0; j < tga_comp; ++j)
                tga_data[i * tga_comp + j] = raw_data[j];

            // 如果处于 RLE 模式，继续计数
            --RLE_count;
        }
// 检查是否需要反转图像
if (tga_inverted) {
    // 遍历图像数据，将每一行的像素颠倒
    for (j = 0; j * 2 < tga_height; ++j) {
        int index1 = j * tga_width * tga_comp;
        int index2 = (tga_height - 1 - j) * tga_width * tga_comp;
        for (i = tga_width * tga_comp; i > 0; --i) {
            // 交换两个像素的数据
            unsigned char temp = tga_data[index1];
            tga_data[index1] = tga_data[index2];
            tga_data[index2] = temp;
            ++index1;
            ++index2;
        }
    }
}
// 清空调色板，如果存在的话
if (tga_palette != NULL) {
    // 释放调色板内存
    STBI_FREE(tga_palette);
}
    // 如果源数据是 RGB16，则不需要交换顺序，否则需要交换 RGB 顺序
    if (tga_comp >= 3 && !tga_rgb16) {
        // 获取 TGA 数据的指针
        unsigned char * tga_pixel = tga_data;
        // 遍历每个像素，交换 RGB 顺序
        for (i = 0; i < tga_width * tga_height; ++i) {
            unsigned char temp = tga_pixel[0];
            tga_pixel[0] = tga_pixel[2];
            tga_pixel[2] = temp;
            tga_pixel += tga_comp;
        }
    }

    // 转换为目标组件数量
    if (req_comp && req_comp != tga_comp)
        tga_data = stbi__convert_format(tga_data, tga_comp, req_comp, tga_width, tga_height);

    // 为了消除错误消息并保持 Microsoft 的 C 编译器的满意，做了一些操作
    tga_palette_start = tga_palette_len = tga_palette_bits = tga_x_origin = tga_y_origin = 0;
    STBI_NOTUSED(tga_palette_start);
    // 操作完成
    return tga_data;
}
#endif

// *************************************************************************************************
// Photoshop PSD loader -- PD by Thatcher Ulrich, integration by Nicolas Schulz, tweaked by STB

#ifndef STBI_NO_PSD
// 检测是否为 PSD 文件，通过判断文件头部是否为 0x38425053
static int stbi__psd_test(stbi__context * s) {
    int r = (stbi__get32be(s) == 0x38425053);
    // 将文件指针重新定位到起始位置
    stbi__rewind(s);
    return r;
}

// 解码 PSD 文件中的 RLE 压缩数据
static int stbi__psd_decode_rle(stbi__context * s, stbi_uc * p, int pixelCount) {
    int count, nleft, len;

    count = 0;
    // 循环解压缩像素数据
    while ((nleft = pixelCount - count) > 0) {
        // 读取 RLE 压缩数据的长度
        len = stbi__get8(s);
        if (len == 128) {
            // 如果长度为128，则不做任何操作
        } else if (len < 128) {
            // 如果长度小于128，则直接复制下一个 len+1 个字节
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
            // 目标中的接下来的 -len+1 个字节将从源的下一个字节复制过来
            // (将 len 解释为一个负的8位整数)
            len = 257 - len;
            if (len > nleft)
                return 0; // 数据损坏
            // 从输入流中获取一个字节的数值
            val = stbi__get8(s);
            // 累加长度到计数器
            count += len;
            // 当长度不为0时，执行循环
            while (len) {
                // 将获取的数值赋给指针所指向的位置
                *p = val;
                // 指针向后移动4个字节
                p += 4;
                // 长度减1
                len--;
            }
        }
    }

    // 返回1，表示加载成功
    return 1;
}

// 从PSD文件中加载数据
static void * stbi__psd_load(stbi__context * s, int * x, int * y, int * comp, int req_comp, stbi__result_info * ri, int bpc) {
    int pixelCount;
    int channelCount, compression;
    int channel, i;
    int bitdepth;
    int w, h;
    stbi_uc * out;
    STBI_NOTUSED(ri);
    // 忽略参数ri，不使用

    // 检查标识符
    if (stbi__get32be(s) != 0x38425053) // "8BPS"
        return stbi__errpuc("not PSD", "Corrupt PSD image");
    // 如果不是PSD格式，返回错误信息

    // 检查文件类型版本
    if (stbi__get16be(s) != 1)
        return stbi__errpuc("wrong version", "Unsupported version of PSD image");
    // 如果版本不支持，返回错误信息

    // 跳过6个保留字节
    stbi__skip(s, 6);

    // 读取图像的通道数（R、G、B、A等）
    channelCount = stbi__get16be(s);
    if (channelCount < 0 || channelCount > 16)
        return stbi__errpuc("wrong channel count", "Unsupported number of channels in PSD image");
    // 如果通道数不在0到16之间，返回错误信息

    // 读取图像的行数和列数
    h = stbi__get32be(s);
    // 从字节流中读取32位整数，存储在变量w中
    w = stbi__get32be(s);

    // 如果高度超过最大限制，则返回错误信息
    if (h > STBI_MAX_DIMENSIONS)
        return stbi__errpuc("too large", "Very large image (corrupt?)");
    // 如果宽度超过最大限制，则返回错误信息
    if (w > STBI_MAX_DIMENSIONS)
        return stbi__errpuc("too large", "Very large image (corrupt?)");

    // 确保图像的深度为8位
    bitdepth = stbi__get16be(s);
    // 如果深度不是8位或16位，则返回错误信息
    if (bitdepth != 8 && bitdepth != 16)
        return stbi__errpuc("unsupported bit depth", "PSD bit depth is not 8 or 16 bit");

    // 确保颜色模式为RGB
    // 有效选项包括：
    //   0: 位图
    //   1: 灰度
    //   2: 索引颜色
    //   3: RGB颜色
    //   4: CMYK颜色
    //   7: 多通道
    // 检查颜色格式是否为 RGB，如果不是则返回错误信息
    if (stbi__get16be(s) != 3)
        return stbi__errpuc("wrong color format", "PSD is not in RGB color format");

    // 跳过模式数据，对于索引颜色模式是调色板，对于其他模式是其他信息
    stbi__skip(s, stbi__get32be(s));

    // 跳过图像资源，如分辨率、画笔路径等
    stbi__skip(s, stbi__get32be(s));

    // 跳过保留数据
    stbi__skip(s, stbi__get32be(s));

    // 检查数据是否被压缩
    // 已知的值：
    //   0: 无压缩
    //   1: RLE 压缩
    compression = stbi__get16be(s);
    if (compression > 1)
        // 如果压缩格式不被识别，返回错误信息
        return stbi__errpuc("bad compression", "PSD has an unknown compression format");

    // 检查图像尺寸是否合法
    if (!stbi__mad3sizes_valid(4, w, h, 0))
        return stbi__errpuc("too large", "Corrupt PSD");

    // 创建目标图像

    // 如果不压缩且位深为16且每通道位数为16，则分配内存空间
    if (!compression && bitdepth == 16 && bpc == 16) {
        out = (stbi_uc *)stbi__malloc_mad3(8, w, h, 0);
        ri->bits_per_channel = 16;
    } else
        // 否则分配内存空间
        out = (stbi_uc *)stbi__malloc(4 * w * h);

    // 如果内存分配失败，返回错误信息
    if (!out)
        return stbi__errpuc("outofmem", "Out of memory");
    pixelCount = w * h;

    // 初始化数据为零
    // memset( out, 0, pixelCount * 4 );
    // 最后是图像数据。
    if (compression) {
        // 使用 .PSD 和 .TIFF 中使用的 RLE 压缩
        // 循环直到获得预期的解压缩字节数：
        //     读取下一个源字节到 n 中。
        //     如果 n 在 0 和 127 之间（包括 0 和 127），就直接复制接下来的 n+1 个字节。
        //     如果 n 在 -127 和 -1 之间（包括 -127 和 -1），就复制接下来的 -n+1 次字节。
        //     如果 n 是 128，就不执行任何操作。
        // 结束循环

        // RLE 压缩的数据之前是每行数据的 2 字节数据计数，我们将直接跳过。
        stbi__skip(s, h * channelCount * 2);

        // 按通道读取 RLE 数据。
        for (channel = 0; channel < 4; channel++) {
            stbi_uc * p;

            p = out + channel;
            // 如果通道数大于等于图像的通道数，填充该通道的默认数据
            if (channel >= channelCount) {
                // 用默认数据填充该通道
                for (i = 0; i < pixelCount; i++, p += 4)
                    *p = (channel == 3 ? 255 : 0);
            } else {
                // 读取RLE数据
                if (!stbi__psd_decode_rle(s, p, pixelCount)) {
                    // 释放内存并返回错误信息
                    STBI_FREE(out);
                    return stbi__errpuc("corrupt", "bad RLE data");
                }
            }
        }
    } else {
        // 到达原始图像数据，每个通道按顺序排列（红色，绿色，蓝色，透明度...）
        // 每个通道包含图像中每个像素的8位（或16位）值。

        // 按通道读取数据
        for (channel = 0; channel < 4; channel++) {
            // 如果通道数大于等于图像的通道数，填充该通道的默认数据
            if (channel >= channelCount) {
                // 用默认数据填充该通道
                // 如果位深度为16且每个通道的位深度也为16
                if (bitdepth == 16 && bpc == 16) {
                    // 如果通道为3，将输出值设置为65535，否则设置为0
                    stbi__uint16 * q = ((stbi__uint16 *)out) + channel;
                    stbi__uint16 val = channel == 3 ? 65535 : 0;
                    // 遍历每个像素，设置每个通道的值为val
                    for (i = 0; i < pixelCount; i++, q += 4)
                        *q = val;
                } else {
                    // 否则，根据通道设置输出值为255或0
                    stbi_uc * p = out + channel;
                    stbi_uc val = channel == 3 ? 255 : 0;
                    // 遍历每个像素，设置每个通道的值为val
                    for (i = 0; i < pixelCount; i++, p += 4)
                        *p = val;
                }
            } else {
                // 如果输入的每个通道的位深度为16
                if (ri->bits_per_channel == 16) { // output bpc
                    // 设置输出值为16位深度的通道值
                    stbi__uint16 * q = ((stbi__uint16 *)out) + channel;
                    // 遍历每个像素，设置每个通道的值为输入流中的16位值
                    for (i = 0; i < pixelCount; i++, q += 4)
                        *q = (stbi__uint16)stbi__get16be(s);
                } else {
                    // 否则，根据输入的位深度设置输出值
                    stbi_uc * p = out + channel;
                    if (bitdepth == 16) { // input bpc
                        // 遍历每个像素，设置每个通道的值为输入流中的16位值
                        for (i = 0; i < pixelCount; i++, p += 4)
    // 如果通道数大于等于4，且每个通道的位数为16
    if (channelCount >= 4) {
        if (ri->bits_per_channel == 16) {
            // 遍历每个像素
            for (i = 0; i < w * h; ++i) {
                // 获取当前像素的指针
                stbi__uint16 * pixel = (stbi__uint16 *)out + 4 * i;
                // 如果当前像素的 alpha 通道不为0且不为最大值65535
                if (pixel[3] != 0 && pixel[3] != 65535) {
                    // 计算 alpha 通道的比例
                    float a = pixel[3] / 65535.0f;
                    // 计算 alpha 通道的倒数
                    float ra = 1.0f / a;
                    // 计算 alpha 通道的倒数的补数
                    float inv_a = 65535.0f * (1 - ra);
                    // 对 RGB 通道进行修正
                    pixel[0] = (stbi__uint16)(pixel[0] * ra + inv_a);
    // 如果 alpha 通道不为 0 且不为 255，则进行颜色混合
    for (i = 0; i < w * h; ++i) {
        // 获取当前像素的指针
        unsigned char * pixel = out + 4 * i;
        // 如果 alpha 通道不为 0 且不为 255，则进行颜色混合
        if (pixel[3] != 0 && pixel[3] != 255) {
            // 计算当前像素的 alpha 值
            float a = pixel[3] / 255.0f;
            // 计算当前像素的反向 alpha 值
            float ra = 1.0f / a;
            // 计算当前像素的反向 alpha 值的补码
            float inv_a = 255.0f * (1 - ra);
            // 对当前像素的 RGB 通道进行颜色混合
            pixel[0] = (unsigned char)(pixel[0] * ra + inv_a);
            pixel[1] = (unsigned char)(pixel[1] * ra + inv_a);
            pixel[2] = (unsigned char)(pixel[2] * ra + inv_a);
        }
    }
    // 转换为期望的输出格式
// 如果请求的通道数不为空且不等于4
if (req_comp && req_comp != 4) {
    // 如果每个通道的位数为16
    if (ri->bits_per_channel == 16)
        // 将输出转换为请求的通道数
        out = (stbi_uc *)stbi__convert_format16((stbi__uint16 *)out, 4, req_comp, w, h);
    else
        // 将输出转换为请求的通道数
        out = stbi__convert_format(out, 4, req_comp, w, h);
    // 如果输出为空，返回空值，stbi__convert_format 在失败时释放输入
    if (out == NULL)
        return out;
}

// 如果 comp 不为空
if (comp)
    // 设置 comp 为 4
    *comp = 4;
// 设置 y 为 h
*y = h;
// 设置 x 为 w
*x = w;

// 返回输出
return out;
}
#endif

// Softimage PIC 加载器
// 作者是Tom Seddon
//
// 参考链接：http://softimage.wiki.softimage.com/index.php/INFO:_PIC_file_format
// 参考链接：http://ozviz.wasp.uwa.edu.au/~pbourke/dataformats/softimagepic/
//
// 如果未定义STBI_NO_PIC，则定义以下函数
static int stbi__pic_is4(stbi__context * s, const char * str) {
    int i;
    for (i = 0; i < 4; ++i)
        if (stbi__get8(s) != (stbi_uc)str[i])
            return 0;

    return 1;
}

// 测试是否为PIC文件格式的核心函数
static int stbi__pic_test_core(stbi__context * s) {
    int i;

    // 如果文件头部不是"\x53\x80\xF6\x34"，则返回0
    if (!stbi__pic_is4(s, "\x53\x80\xF6\x34"))
        return 0;
    // 读取 84 个字节并丢弃
    for (i = 0; i < 84; ++i)
        stbi__get8(s);

    // 检查是否为 PICT 格式
    if (!stbi__pic_is4(s, "PICT"))
        return 0;

    // 返回 1 表示成功
    return 1;
}

// 定义 stbi__pic_packet 结构体
typedef struct {
    stbi_uc size, type, channel;
} stbi__pic_packet;

// 读取通道值
static stbi_uc * stbi__readval(stbi__context * s, int channel, stbi_uc * dest) {
    int mask = 0x80, i;

    // 遍历通道值
    for (i = 0; i < 4; ++i, mask >>= 1) {
        if (channel & mask) {
            // 如果已经到达文件末尾，则返回
            if (stbi__at_eof(s))
static void stbi__copyval(int channel, stbi_uc * dest, stbi_uc * src) {
    // 初始化掩码为0x80
    int mask = 0x80, i;

    // 遍历4次，每次右移一位，检查通道是否匹配掩码
    for (i = 0; i < 4; ++i, mask >>= 1)
        if (channel & mask)
            // 如果通道匹配掩码，则将源数据复制到目标数据
            dest[i] = src[i];
}

static stbi_uc * stbi__pic_load_core(stbi__context * s, int width, int height, int * comp, stbi_uc * result) {
    // 初始化实际通道数为0，数据包数量为0，y坐标，链式标志
    int act_comp = 0, num_packets = 0, y, chained;
    // 初始化数据包数组
    stbi__pic_packet packets[10];
// 这段代码用于处理一些奇怪的情况，比如在多个数据包中有相同通道的数据。
do {
    // 定义一个数据包结构体指针
    stbi__pic_packet * packet;

    // 如果数据包数量达到了最大限制，返回错误
    if (num_packets == sizeof(packets) / sizeof(packets[0]))
        return stbi__errpuc("bad format", "too many packets");

    // 获取下一个数据包
    packet = &packets[num_packets++];

    // 从输入流中读取数据，并存入数据包结构体中
    chained = stbi__get8(s);
    packet->size = stbi__get8(s);
    packet->type = stbi__get8(s);
    packet->channel = stbi__get8(s);

    // 更新通道标志
    act_comp |= packet->channel;

    // 如果已经到达文件末尾，返回错误
    if (stbi__at_eof(s))
        return stbi__errpuc("bad file", "file too short (reading packets)");
    // 如果数据包大小不为8，返回错误
    if (packet->size != 8)
            return stbi__errpuc("bad format", "packet isn't 8bpp");
    } while (chained);

    *comp = (act_comp & 0x10 ? 4 : 3); // has alpha channel?

    for (y = 0; y < height; ++y) {
        int packet_idx;

        for (packet_idx = 0; packet_idx < num_packets; ++packet_idx) {
            stbi__pic_packet * packet = &packets[packet_idx];
            stbi_uc * dest = result + y * width * 4;

            switch (packet->type) {
            default:
                return stbi__errpuc("bad format", "packet has bad compression type");

            case 0: { // uncompressed
                int x;

                for (x = 0; x < width; ++x, dest += 4)
```

注释：

```
            return stbi__errpuc("bad format", "packet isn't 8bpp");
```
如果像素数据不是8位每像素，返回错误信息。

```
    } while (chained);
```
当chained为真时，执行do-while循环。

```
    *comp = (act_comp & 0x10 ? 4 : 3); // has alpha channel?
```
根据act_comp的值判断是否有alpha通道，将结果存入comp指针指向的位置。

```
    for (y = 0; y < height; ++y) {
```
循环遍历图像的高度。

```
        int packet_idx;

        for (packet_idx = 0; packet_idx < num_packets; ++packet_idx) {
```
循环遍历数据包的索引。

```
            stbi__pic_packet * packet = &packets[packet_idx];
            stbi_uc * dest = result + y * width * 4;
```
获取当前数据包的指针和目标位置的指针。

```
            switch (packet->type) {
            default:
                return stbi__errpuc("bad format", "packet has bad compression type");

            case 0: { // uncompressed
                int x;

                for (x = 0; x < width; ++x, dest += 4)
```
根据数据包的类型进行不同的操作，如果是未压缩的数据包，循环遍历图像的宽度。
                    // 如果遇到 Pure RLE 类型的数据包
                    if (!stbi__readval(s, packet->channel, dest))
                        // 如果读取失败，返回 0
                        return 0;
                break;
            }

            case 1: // Pure RLE
            {
                // 剩余像素数量
                int left = width, i;

                // 当还有剩余像素时循环
                while (left > 0) {
                    // 记录重复次数和数值
                    stbi_uc count, value[4];

                    // 读取重复次数
                    count = stbi__get8(s);
                    // 如果已经到达文件末尾，返回错误
                    if (stbi__at_eof(s))
                        return stbi__errpuc("bad file", "file too short (pure read count)");

                    // 如果重复次数大于剩余像素数量，将重复次数设为剩余像素数量
                    if (count > left)
                        count = (stbi_uc)left;

                    // 读取数值
                    if (!stbi__readval(s, packet->channel, value))
                        return 0; // 如果不满足任何情况，返回 0

                    for (i = 0; i < count; ++i, dest += 4) // 循环，每次增加 4 个字节的目标地址
                        stbi__copyval(packet->channel, dest, value); // 调用 stbi__copyval 函数，将 packet 的通道值复制到目标地址
                    left -= count; // 减去已处理的像素数量
                }
            } break;

            case 2: { // Mixed RLE
                int left = width; // 初始化剩余像素数量为图像宽度
                while (left > 0) { // 当剩余像素数量大于 0 时循环
                    int count = stbi__get8(s), i; // 从输入流中读取一个字节作为像素数量
                    if (stbi__at_eof(s)) // 如果已到达文件末尾
                        return stbi__errpuc("bad file", "file too short (mixed read count)"); // 返回错误信息

                    if (count >= 128) { // 如果像素数量大于等于 128
                        stbi_uc value[4]; // 定义一个长度为 4 的无符号字符数组

                        if (count == 128) // 如果像素数量为 128
                            count = stbi__get16be(s); // 从输入流中读取两个字节作为像素数量
                    else
                        // 如果count大于127，则减去127
                        count -= 127;
                    // 如果count大于剩余的长度，返回错误
                    if (count > left)
                        return stbi__errpuc("bad file", "scanline overrun");

                    // 从输入流中读取值到packet->channel中
                    if (!stbi__readval(s, packet->channel, value))
                        return 0;

                    // 根据count的值进行循环，将value复制到dest中
                    for (i = 0; i < count; ++i, dest += 4)
                        stbi__copyval(packet->channel, dest, value);
                } else { // Raw
                    // count加1
                    ++count;
                    // 如果count大于剩余的长度，返回错误
                    if (count > left)
                        return stbi__errpuc("bad file", "scanline overrun");

                    // 根据count的值进行循环，从输入流中读取值到dest中
                    for (i = 0; i < count; ++i, dest += 4)
                        if (!stbi__readval(s, packet->channel, dest))
                            return 0;
                }
                // 更新剩余长度
                left -= count;
    }
    break;
}
}
}
}

return result;
}

// 加载 PIC 格式的图片数据
static void * stbi__pic_load(stbi__context * s, int * px, int * py, int * comp, int req_comp, stbi__result_info * ri) {
    stbi_uc * result;
    int i, x, y, internal_comp;
    STBI_NOTUSED(ri);

    // 如果 comp 为空，则使用 internal_comp
    if (!comp)
        comp = &internal_comp;

    // 跳过前 92 个字节的数据
    for (i = 0; i < 92; ++i)
        stbi__get8(s);
    # 从字节流中读取两个字节，存储到变量 x 中
    x = stbi__get16be(s);
    # 从字节流中读取两个字节，存储到变量 y 中
    y = stbi__get16be(s);

    # 如果图片的高度超过了最大限制，则返回错误信息
    if (y > STBI_MAX_DIMENSIONS)
        return stbi__errpuc("too large", "Very large image (corrupt?)");
    # 如果图片的宽度超过了最大限制，则返回错误信息
    if (x > STBI_MAX_DIMENSIONS)
        return stbi__errpuc("too large", "Very large image (corrupt?)");

    # 如果已经到达文件末尾，则返回文件过短的错误信息
    if (stbi__at_eof(s))
        return stbi__errpuc("bad file", "file too short (pic header)");
    # 如果图片的尺寸过大，则返回错误信息
    if (!stbi__mad3sizes_valid(x, y, 4, 0))
        return stbi__errpuc("too large", "PIC image too large to decode");

    # 跳过 4 个字节的数据
    stbi__get32be(s); // skip `ratio'
    # 跳过 2 个字节的数据
    stbi__get16be(s); // skip `fields'
    # 跳过 2 个字节的数据
    stbi__get16be(s); // skip `pad'

    # 分配内存，用于存储解码后的图片数据
    # 分配的内存大小为 x * y * 4
    result = (stbi_uc *)stbi__malloc_mad3(x, y, 4, 0);
    // 如果结果为空，则返回内存不足的错误信息
    if (!result)
        return stbi__errpuc("outofmem", "Out of memory");
    // 将结果数组初始化为全 0xFF
    memset(result, 0xff, x * y * 4);

    // 如果图片加载核心函数返回失败，则释放结果数组并将其置为 0
    if (!stbi__pic_load_core(s, x, y, comp, result)) {
        STBI_FREE(result);
        result = 0;
    }
    // 将图片的宽度和高度赋值给指针变量
    *px = x;
    *py = y;
    // 如果请求的通道数为 0，则将其赋值为实际通道数
    if (req_comp == 0)
        req_comp = *comp;
    // 将结果数组转换为请求的通道数的格式
    result = stbi__convert_format(result, 4, req_comp, x, y);

    // 返回转换后的结果数组
    return result;
}

// 测试是否为 PIC 格式的图片
static int stbi__pic_test(stbi__context * s) {
    // 调用 PIC 格式测试核心函数
    int r = stbi__pic_test_core(s);
    // 将文件指针重新定位到文件开头
    stbi__rewind(s);
    return r;
}
#endif

// *************************************************************************************************
// GIF loader -- public domain by Jean-Marc Lienher -- simplified/shrunk by stb

#ifndef STBI_NO_GIF
// 定义 GIF 解码所需的数据结构 stbi__gif_lzw
typedef struct {
    stbi__int16 prefix; // LZW 编码的前缀
    stbi_uc first; // LZW 编码的第一个字节
    stbi_uc suffix; // LZW 编码的后缀
} stbi__gif_lzw;

// 定义 GIF 图像的数据结构
typedef struct {
    int w, h; // 图像的宽度和高度
    stbi_uc * out;        // 输出缓冲区（始终为4个分量）
    stbi_uc * background; // GIF 图像的当前“背景”
    stbi_uc * history; // 历史记录
    int flags, bgindex, ratio, transparent, eflags; // 其他标志和参数
// 定义一个包含256个元素的数组，每个元素包含4个字节，用于存储调色板
stbi_uc pal[256][4];
// 定义一个包含256个元素的数组，每个元素包含4个字节，用于存储局部调色板
stbi_uc lpal[256][4];
// 定义一个包含8192个元素的数组，用于存储 GIF 解码时使用的 LZW 编码
stbi__gif_lzw codes[8192];
// 指向颜色表的指针
stbi_uc * color_table;
// 解析标志
int parse, step;
// 局部标志
int lflags;
// 起始坐标
int start_x, start_y;
// 最大坐标
int max_x, max_y;
// 当前坐标
int cur_x, cur_y;
// 行大小
int line_size;
// 延迟
int delay;
} stbi__gif;

// 静态函数，用于测试是否为原始的 GIF 格式
static int stbi__gif_test_raw(stbi__context * s) {
    int sz;
    // 读取并检查 GIF 文件的标识符是否为 "GIF8"
    if (stbi__get8(s) != 'G' || stbi__get8(s) != 'I' || stbi__get8(s) != 'F' || stbi__get8(s) != '8')
        return 0;
    // 读取并检查 GIF 文件的版本号是否为 "9" 或 "7"
    sz = stbi__get8(s);
    if (sz != '9' && sz != '7')
        return 0;
# 如果读取的下一个字节不是 'a'，则返回 0
if (stbi__get8(s) != 'a')
    return 0;
# 否则返回 1
return 1;
}

# 测试是否为 GIF 格式的图像
static int stbi__gif_test(stbi__context * s) {
    # 调用 stbi__gif_test_raw 函数进行测试
    int r = stbi__gif_test_raw(s);
    # 重置读取位置
    stbi__rewind(s);
    # 返回测试结果
    return r;
}

# 解析 GIF 图像的颜色表
static void stbi__gif_parse_colortable(stbi__context * s, stbi_uc pal[256][4], int num_entries, int transp) {
    # 遍历颜色表的每个条目
    int i;
    for (i = 0; i < num_entries; ++i) {
        # 依次读取 RGB 值
        pal[i][2] = stbi__get8(s);
        pal[i][1] = stbi__get8(s);
        pal[i][0] = stbi__get8(s);
        # 如果当前条目是透明色，则 alpha 通道为 0，否则为 255
        pal[i][3] = transp == i ? 0 : 255;
    }
}
# 解析 GIF 文件头部信息
static int stbi__gif_header(stbi__context * s, stbi__gif * g, int * comp, int is_info) {
    stbi_uc version;
    # 检查文件头部是否为 "GIF8"
    if (stbi__get8(s) != 'G' || stbi__get8(s) != 'I' || stbi__get8(s) != 'F' || stbi__get8(s) != '8')
        return stbi__err("not GIF", "Corrupt GIF");

    # 检查版本号是否为 "7" 或 "9"
    version = stbi__get8(s);
    if (version != '7' && version != '9')
        return stbi__err("not GIF", "Corrupt GIF");
    # 检查是否为合法的 GIF 文件
    if (stbi__get8(s) != 'a')
        return stbi__err("not GIF", "Corrupt GIF");

    # 初始化 GIF 结构体的属性
    stbi__g_failure_reason = "";
    g->w = stbi__get16le(s);
    g->h = stbi__get16le(s);
    g->flags = stbi__get8(s);
    g->bgindex = stbi__get8(s);
    g->ratio = stbi__get8(s);
    g->transparent = -1;
    // 如果图像宽度超过最大限制，则返回错误信息
    if (g->w > STBI_MAX_DIMENSIONS)
        return stbi__err("too large", "Very large image (corrupt?)");
    // 如果图像高度超过最大限制，则返回错误信息
    if (g->h > STBI_MAX_DIMENSIONS)
        return stbi__err("too large", "Very large image (corrupt?)");

    // 如果 comp 不为 0，则将其设置为 4，因为在解析注释之前无法确定是 3 还是 4
    if (comp != 0)
        *comp = 4; // can't actually tell whether it's 3 or 4 until we parse the comments

    // 如果是信息模式，则返回 1
    if (is_info)
        return 1;

    // 如果标志中包含 0x80，则解析颜色表
    if (g->flags & 0x80)
        stbi__gif_parse_colortable(s, g->pal, 2 << (g->flags & 7), -1);

    // 返回 1
    return 1;
}

// 解析 GIF 图像的信息
static int stbi__gif_info_raw(stbi__context * s, int * x, int * y, int * comp) {
    // 分配内存给 GIF 结构体
    stbi__gif * g = (stbi__gif *)stbi__malloc(sizeof(stbi__gif));
    // 如果分配内存失败，则返回错误
    if (!g)
        // 如果内存不足，返回内存不足的错误信息
        return stbi__err("outofmem", "Out of memory");
    // 如果解析 GIF 头部失败
    if (!stbi__gif_header(s, g, comp, 1)) {
        // 释放内存
        STBI_FREE(g);
        // 重新定位到文件开头
        stbi__rewind(s);
        // 返回解析失败
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
    // 返回解析成功
    return 1;
}

static void stbi__out_gif_code(stbi__gif * g, stbi__uint16 code) {
    stbi_uc *p, *c;
    int idx;

    // 递归解码前缀，因为链表是反向的，反向处理交错图像会很麻烦
    # 如果当前代码的前缀大于等于0，则调用stbi__out_gif_code函数，传入当前代码的前缀
    if (g->codes[code].prefix >= 0)
        stbi__out_gif_code(g, g->codes[code].prefix);

    # 如果当前y坐标大于等于最大y坐标，则返回
    if (g->cur_y >= g->max_y)
        return;

    # 计算当前像素在输出数组中的索引
    idx = g->cur_x + g->cur_y;
    # 获取当前像素的指针
    p = &g->out[idx];
    # 将当前像素所在的4个像素位置标记为历史像素
    g->history[idx / 4] = 1;

    # 获取当前代码对应的颜色表中的颜色值
    c = &g->color_table[g->codes[code].suffix * 4];
    # 如果颜色的透明度大于128，则不渲染透明像素
    if (c[3] > 128) { 
        p[0] = c[2];  # 设置像素的蓝色分量
        p[1] = c[1];  # 设置像素的绿色分量
        p[2] = c[0];  # 设置像素的红色分量
        p[3] = c[3];  # 设置像素的透明度
    }
    g->cur_x += 4;  # 更新当前x坐标

    # 如果当前x坐标大于等于最大x坐标
    if (g->cur_x >= g->max_x) {
        // 将当前 x 坐标设置为起始 x 坐标
        g->cur_x = g->start_x;
        // 将当前 y 坐标增加步长
        g->cur_y += g->step;

        // 当当前 y 坐标大于等于最大 y 坐标并且解析次数大于 0 时执行循环
        while (g->cur_y >= g->max_y && g->parse > 0) {
            // 计算新的步长
            g->step = (1 << g->parse) * g->line_size;
            // 将当前 y 坐标设置为起始 y 坐标加上步长的一半
            g->cur_y = g->start_y + (g->step >> 1);
            // 解析次数减一
            --g->parse;
        }
    }
}

// 处理 GIF 栅格数据
static stbi_uc * stbi__process_gif_raster(stbi__context * s, stbi__gif * g) {
    // 定义变量
    stbi_uc lzw_cs;
    stbi__int32 len, init_code;
    stbi__uint32 first;
    stbi__int32 codesize, codemask, avail, oldcode, bits, valid_bits, clear;
    stbi__gif_lzw * p;

    // 读取 LZW 编码的代码尺寸
    lzw_cs = stbi__get8(s);
    // 如果代码尺寸大于 12
    if (lzw_cs > 12)
        // 返回空值
        return NULL;
    // 将1左移lzw_cs位，得到clear值
    clear = 1 << lzw_cs;
    // 初始化first为1
    first = 1;
    // 初始化codesize为lzw_cs+1
    codesize = lzw_cs + 1;
    // 初始化codemask为2的codesize次方减1
    codemask = (1 << codesize) - 1;
    // 初始化bits为0
    bits = 0;
    // 初始化valid_bits为0
    valid_bits = 0;
    // 遍历初始化代码
    for (init_code = 0; init_code < clear; init_code++) {
        // 设置codes数组中的prefix为-1
        g->codes[init_code].prefix = -1;
        // 设置codes数组中的first为init_code
        g->codes[init_code].first = (stbi_uc)init_code;
        // 设置codes数组中的suffix为init_code
        g->codes[init_code].suffix = (stbi_uc)init_code;
    }

    // 支持无起始清除代码
    // 初始化avail为clear+2
    avail = clear + 2;
    // 初始化oldcode为-1
    oldcode = -1;

    // 初始化len为0
    len = 0;
    // 无限循环
    for (;;) {
        // 如果valid_bits小于codesize
        if (valid_bits < codesize) {
            // 如果当前块长度为0，表示需要开始一个新的块
            if (len == 0) {
                // 从输入流中读取一个字节，作为新块的长度
                len = stbi__get8(s); // start new block
                // 如果新块长度为0，表示已经到达流的末尾，直接返回输出结果
                if (len == 0)
                    return g->out;
            }
            // 减少当前块的长度
            --len;
            // 将下一个字节的数据合并到 bits 中
            bits |= (stbi__int32)stbi__get8(s) << valid_bits;
            // 更新有效位数
            valid_bits += 8;
        } else {
            // 从 bits 中取出当前编码
            stbi__int32 code = bits & codemask;
            // 右移 bits，丢弃已经使用的编码位
            bits >>= codesize;
            // 更新有效位数
            valid_bits -= codesize;
            // @OPTIMIZE: 是否有一些方法可以加速非清除路径？
            // 如果当前编码是清除码
            if (code == clear) { // clear code
                // 更新编码长度
                codesize = lzw_cs + 1;
                // 更新编码掩码
                codemask = (1 << codesize) - 1;
                // 更新可用编码数
                avail = clear + 2;
                // 重置旧编码
                oldcode = -1;
                // 重置首字节
                first = 0;
            } 
            // 如果当前编码是流结束码
            else if (code == clear + 1) { // end of stream code
                # 跳过指定长度的数据
                stbi__skip(s, len);
                # 当长度大于0时，继续跳过数据
                while ((len = stbi__get8(s)) > 0)
                    stbi__skip(s, len);
                # 返回解码后的图像数据
                return g->out;
            } else if (code <= avail) {
                # 如果是第一个code，返回错误信息
                if (first) {
                    return stbi__errpuc("no clear code", "Corrupt GIF");
                }
                # 如果oldcode存在，创建新的code
                if (oldcode >= 0) {
                    p = &g->codes[avail++];
                    # 如果code数量超过8192，返回错误信息
                    if (avail > 8192) {
                        return stbi__errpuc("too many codes", "Corrupt GIF");
                    }
                    # 设置新code的前缀、首字符和后缀
                    p->prefix = (stbi__int16)oldcode;
                    p->first = g->codes[oldcode].first;
                    p->suffix = (code == avail) ? p->first : g->codes[code].first;
                } else if (code == avail)
                    # 如果code等于avail，返回错误信息
                    return stbi__errpuc("illegal code in raster", "Corrupt GIF");
// 将 code 输出到 GIF 图像数据中
stbi__out_gif_code(g, (stbi__uint16)code);

// 如果可用码字数和码字掩码相等且小于等于 0x0FFF，则增加码字大小和更新码字掩码
if ((avail & codemask) == 0 && avail <= 0x0FFF) {
    codesize++;
    codemask = (1 << codesize) - 1;
}

// 保存当前码字
oldcode = code;

// 如果遇到非法码字，则返回错误信息
} else {
    return stbi__errpuc("illegal code in raster", "Corrupt GIF");
}

// 用于支持动画 GIF，尽管 stb_image 不支持
// dispose 用于指定图像的处理方式
static stbi_uc * stbi__gif_load_next(stbi__context * s, stbi__gif * g, int * comp, int req_comp, stbi_uc * two_back) {
    int dispose;
    // 定义变量，用于存储第一帧、像素索引、像素数量
    int first_frame;
    int pi;
    int pcount;
    STBI_NOTUSED(req_comp);

    // 在第一帧上，任何未写入的像素都将使用背景颜色（非透明）
    first_frame = 0;
    // 如果输出缓冲区为空
    if (g->out == 0) {
        // 读取 GIF 文件头信息
        if (!stbi__gif_header(s, g, comp, 0))
            return 0; // 如果读取失败，返回0，错误原因由stbi__gif_header设置
        // 检查图像大小是否合法
        if (!stbi__mad3sizes_valid(4, g->w, g->h, 0))
            return stbi__errpuc("too large", "GIF image is too large");
        // 计算像素数量
        pcount = g->w * g->h;
        // 分配输出缓冲区、背景颜色缓冲区、历史记录缓冲区的内存空间
        g->out = (stbi_uc *)stbi__malloc(4 * pcount);
        g->background = (stbi_uc *)stbi__malloc(4 * pcount);
        g->history = (stbi_uc *)stbi__malloc(pcount);
        // 如果内存分配失败，返回错误
        if (!g->out || !g->background || !g->history)
            return stbi__errpuc("outofmem", "Out of memory");

        // 图像在开始时被视为“透明” - 即，没有东西覆盖当前背景
        // 背景颜色仅用于未在第一帧渲染的像素，之后“背景”颜色指的是上一帧的颜色。
        // 将输出缓冲区的内容全部设置为0x00，即透明色
        memset(g->out, 0x00, 4 * pcount);
        // 将背景缓冲区的内容全部设置为0x00，即透明色（初始状态为透明）
        memset(g->background, 0x00, 4 * pcount);
        // 将历史记录缓冲区的内容全部设置为0x00
        memset(g->history, 0x00, pcount);
        // 标记为第一帧
        first_frame = 1;
    } else {
        // 第二帧 - 我们如何处理前一帧？
        // 获取图形控制扩展块中的处理方式
        dispose = (g->eflags & 0x1C) >> 2;
        // 计算像素总数
        pcount = g->w * g->h;

        // 如果处理方式为3且没有上上帧图像，则将处理方式设置为2，即默认使用旧背景
        if ((dispose == 3) && (two_back == 0)) {
            dispose = 2;
        }

        // 如果处理方式为3，则使用上一帧的图像
        if (dispose == 3) {
            // 遍历像素，如果该像素在上一帧受到影响，则将输出缓冲区的像素值设置为上上帧的像素值
            for (pi = 0; pi < pcount; ++pi) {
                if (g->history[pi]) {
                    memcpy(&g->out[pi * 4], &two_back[pi * 4], 4);
                }
        } else if (dispose == 2) {
            // 如果 dispose 等于 2，将上一帧改变的内容恢复到该帧的背景中
            for (pi = 0; pi < pcount; ++pi) {
                if (g->history[pi]) {
                    // 如果该像素在历史记录中有改变，将其值恢复为背景中对应位置的值
                    memcpy(&g->out[pi * 4], &g->background[pi * 4], 4);
                }
            }
        } else {
            // 如果不是 dispose 等于 2 的情况，即非 dispose 情况，保持像素不变，它们将成为新的背景
            // 1: 不丢弃
            // 0: 未指定
        }

        // 将 out 中的内容复制到 background 中，作为前一帧撤销后的背景
        memcpy(g->background, g->out, 4 * g->w * g->h);
    }

    // 清空历史记录
    // 用 0x00 填充 g->history 数组，表示上一帧受影响的像素
    memset(g->history, 0x00, g->w * g->h); 

    // 无限循环，解析 GIF 图像数据
    for (;;) {
        // 读取一个字节，表示图像数据的标签
        int tag = stbi__get8(s);
        // 根据标签类型进行处理
        switch (tag) {
        case 0x2C: /* Image Descriptor */
        {
            // 读取图像的位置和尺寸信息
            stbi__int32 x, y, w, h;
            stbi_uc * o;

            x = stbi__get16le(s);
            y = stbi__get16le(s);
            w = stbi__get16le(s);
            h = stbi__get16le(s);
            // 检查图像位置和尺寸是否合法
            if (((x + w) > (g->w)) || ((y + h) > (g->h)))
                return stbi__errpuc("bad Image Descriptor", "Corrupt GIF");

            // 计算每行像素的字节数
            g->line_size = g->w * 4;
            // 计算图像数据的起始位置
            g->start_x = x * 4;
            g->start_y = y * g->line_size;
            // 设置最大 x 坐标为起始 x 坐标加上宽度乘以 4
            g->max_x = g->start_x + w * 4;
            // 设置最大 y 坐标为起始 y 坐标加上高度乘以行大小
            g->max_y = g->start_y + h * g->line_size;
            // 设置当前 x 坐标为起始 x 坐标
            g->cur_x = g->start_x;
            // 设置当前 y 坐标为起始 y 坐标
            g->cur_y = g->start_y;

            // 如果指定矩形的宽度为 0，表示可能看不到任何像素或图像格式不正确；
            // 为了确保捕捉到这种情况，将当前 y 坐标移动到最大 y 坐标（这是 out_gif_code 检查的内容）。
            if (w == 0)
                g->cur_y = g->max_y;

            // 读取一个字节作为标志位
            g->lflags = stbi__get8(s);

            // 如果标志位的第 6 位为 1
            if (g->lflags & 0x40) {
                // 设置步长为行大小的 8 倍，用于隔行扫描的第一个间隔
                g->step = 8 * g->line_size; 
                // 设置解析模式为 3
                g->parse = 3;
            } else {
                // 否则，设置步长为行大小
                g->step = g->line_size;
                // 设置解析模式为 0
                g->parse = 0;
            }

            // 如果逻辑标志位中包含0x80
            if (g->lflags & 0x80) {
                // 解析颜色表
                stbi__gif_parse_colortable(s, g->lpal, 2 << (g->lflags & 7), g->eflags & 0x01 ? g->transparent : -1);
                // 将颜色表赋值给color_table
                g->color_table = (stbi_uc *)g->lpal;
            } 
            // 如果标志位中包含0x80
            else if (g->flags & 0x80) {
                // 将颜色表赋值给color_table
                g->color_table = (stbi_uc *)g->pal;
            } 
            // 如果以上条件都不满足，返回错误信息
            else
                return stbi__errpuc("missing color table", "Corrupt GIF");

            // 处理 GIF 光栅数据
            o = stbi__process_gif_raster(s, g);
            // 如果处理失败，返回空指针
            if (!o)
                return NULL;

            // 如果这是第一帧
            pcount = g->w * g->h;
            if (first_frame && (g->bgindex > 0)) {
                // 如果是第一帧，未绘制的像素使用背景颜色
                for (pi = 0; pi < pcount; ++pi) {
                    if (g->history[pi] == 0) {
// 设置调色板中当前颜色的透明度为255，以防止之前设置的透明色影响；下一帧将根据需要重新设置
g->pal[g->bgindex][3] = 255;
// 将调色板中当前颜色的RGBA值复制到输出图像数据中
memcpy(&g->out[pi * 4], &g->pal[g->bgindex], 4);
// 如果需要，将输出图像数据中的像素索引pi设置为透明色

case 0x21: // 注释扩展
{
    int len;
    int ext = stbi__get8(s);
    if (ext == 0xF9) { // 图形控制扩展
        len = stbi__get8(s);
        if (len == 4) {
            g->eflags = stbi__get8(s);
            // 设置延迟时间，以1/100秒为单位，保存为1/1000秒
            g->delay = 10 * stbi__get16le(s);
// 如果已经设置了透明色，则将其 alpha 值设为 255
if (g->transparent >= 0) {
    g->pal[g->transparent][3] = 255;
}
// 如果 eflags 的第一位为 1
if (g->eflags & 0x01) {
    // 读取透明色索引
    g->transparent = stbi__get8(s);
    // 如果透明色索引有效，则将其 alpha 值设为 0
    if (g->transparent >= 0) {
        g->pal[g->transparent][3] = 0;
    }
} else {
    // 不需要透明色
    stbi__skip(s, 1);
    g->transparent = -1;
}
// 跳过 len 长度的数据
} else {
    stbi__skip(s, len);
    break;
}
// 循环直到遇到值为 0 的 len
while ((len = stbi__get8(s)) != 0) {
            // 跳过指定长度的数据
            stbi__skip(s, len);
        }
        // 结束循环
        break;
    }

    case 0x3B:               // GIF 流终止代码
        // 返回当前位置的指针，使用 '1' 在某些编译器上会产生警告
        return (stbi_uc *)s;

    default:
        // 返回错误指针，表示未知的代码，提示 GIF 文件损坏
        return stbi__errpuc("unknown code", "Corrupt GIF");
    }
}

// 从 GIF 文件加载数据，处理内存不足情况
static void * stbi__load_gif_main_outofmem(stbi__gif * g, stbi_uc * out, int ** delays) {
    // 释放之前分配的内存
    STBI_FREE(g->out);
    STBI_FREE(g->history);
    STBI_FREE(g->background);

    // 如果输出指针不为空
    if (out)
        释放 out 指向的内存空间
        STBI_FREE(out);
    如果 delays 不为空且指向的值不为空
        释放 delays 指向的内存空间
        STBI_FREE(*delays);
    返回一个错误指针，表示内存不足
    return stbi__errpuc("outofmem", "Out of memory");
}

static void * stbi__load_gif_main(stbi__context * s, int ** delays, int * x, int * y, int * z, int * comp, int req_comp) {
    如果输入的数据是 GIF 格式
        初始化变量 layers 为 0
        初始化变量 u 为 0
        初始化变量 out 为 0
        初始化变量 two_back 为 0
        初始化变量 g 为 stbi__gif 结构体
        初始化变量 stride 为 0
        初始化变量 out_size 为 0
        初始化变量 delays_size 为 0

        不使用 out_size 变量
        STBI_NOTUSED(out_size);
        不使用 delays_size 变量
        STBI_NOTUSED(delays_size);
        // 将 g 的内存空间初始化为 0
        memset(&g, 0, sizeof(g));
        // 如果 delays 不为空，则将其值设为 0
        if (delays) {
            *delays = 0;
        }

        // 循环处理每一帧图像数据
        do {
            // 调用 stbi__gif_load_next 函数加载下一帧图像数据
            u = stbi__gif_load_next(s, &g, comp, req_comp, two_back);
            // 如果返回的指针等于 s，表示已经到达动态 GIF 的结尾
            if (u == (stbi_uc *)s)
                u = 0; // end of animated gif marker

            // 如果返回的指针不为空
            if (u) {
                // 将图像的宽度和高度赋值给 x 和 y
                *x = g.w;
                *y = g.h;
                // 帧数加一
                ++layers;
                // 计算图像数据的步长
                stride = g.w * g.h * 4;

                // 如果 out 不为空
                if (out) {
                    // 重新分配内存空间以容纳新的图像数据
                    void * tmp = (stbi_uc *)STBI_REALLOC_SIZED(out, out_size, layers * stride);
                    // 如果分配内存失败，则返回内存不足的错误
                    if (!tmp)
                        return stbi__load_gif_main_outofmem(&g, out, delays);
                    else {
                        // 如果没有指定输出缓冲区，则将 tmp 赋给 out，out_size 为图像数据大小
                        out = (stbi_uc *)tmp;
                        out_size = layers * stride;
                    }

                    if (delays) {
                        // 重新分配 delays 的内存空间，大小为 layers * sizeof(int)，并将指针赋给 new_delays
                        int * new_delays = (int *)STBI_REALLOC_SIZED(*delays, delays_size, sizeof(int) * layers);
                        // 如果内存分配失败，则调用 stbi__load_gif_main_outofmem 函数处理
                        if (!new_delays)
                            return stbi__load_gif_main_outofmem(&g, out, delays);
                        *delays = new_delays;
                        delays_size = layers * sizeof(int);
                    }
                } else {
                    // 分配大小为 layers * stride 的内存空间给 out
                    out = (stbi_uc *)stbi__malloc(layers * stride);
                    // 如果内存分配失败，则调用 stbi__load_gif_main_outofmem 函数处理
                    if (!out)
                        return stbi__load_gif_main_outofmem(&g, out, delays);
                    out_size = layers * stride;
                    if (delays) {
                        // 分配大小为 layers * sizeof(int) 的内存空间给 *delays
                        *delays = (int *)stbi__malloc(layers * sizeof(int));
                        // 如果内存分配失败，则调用 stbi__load_gif_main_outofmem 函数处理
                        if (!*delays)
                return stbi__load_gif_main_outofmem(&g, out, delays);
                // 如果内存不足，调用函数处理内存不足情况并返回结果

                delays_size = layers * sizeof(int);
                // 计算延迟数组的大小

                memcpy(out + ((layers - 1) * stride), u, stride);
                // 将当前帧的数据复制到输出数组中

                if (layers >= 2) {
                    two_back = out - 2 * stride;
                }
                // 如果帧数大于等于2，将两帧前的数据指针指向相应位置

                if (delays) {
                    (*delays)[layers - 1U] = g.delay;
                }
                // 如果延迟数组存在，将当前帧的延迟值存入数组

                // 重复循环直到 u 等于 0

                // 释放临时缓冲区
                STBI_FREE(g.out);
                STBI_FREE(g.history);
                STBI_FREE(g.background);
                // 释放临时缓冲区的内存
// 在加载所有内容后进行最终转换；
if (req_comp && req_comp != 4)
    out = stbi__convert_format(out, 4, req_comp, layers * g.w, g.h);
// 将图像的层数赋值给 z
*z = layers;
// 返回处理后的图像数据
return out;
} else {
    // 如果不是 GIF 图像，则返回错误信息
    return stbi__errpuc("not GIF", "Image was not as a gif type.");
}
}

// 加载 GIF 图像的函数
static void * stbi__gif_load(stbi__context * s, int * x, int * y, int * comp, int req_comp, stbi__result_info * ri) {
    stbi_uc * u = 0;
    stbi__gif g;
    // 初始化 g 结构体
    memset(&g, 0, sizeof(g));
    STBI_NOTUSED(ri);

    // 调用 stbi__gif_load_next 函数加载下一帧 GIF 图像
    u = stbi__gif_load_next(s, &g, comp, req_comp, 0);
    // 如果返回的指针等于 s 指针，表示动画 GIF 图像结束
    if (u == (stbi_uc *)s)
        u = 0; // 动画 GIF 结束标记
    // 如果 u 存在（非空），则将图像的宽度和高度赋值给 x 和 y
    if (u) {
        *x = g.w;
        *y = g.h;

        // 在成功加载后移动转换，以便可以对多个帧执行相同的操作
        if (req_comp && req_comp != 4)
            // 如果请求的通道数不为4，则将图像格式转换为请求的通道数
            u = stbi__convert_format(u, 4, req_comp, g.w, g.h);
    } else if (g.out) {
        // 如果出现错误并且我们分配了图像缓冲区，则释放它
        STBI_FREE(g.out);
    }

    // 释放多帧加载所需的缓冲区
    STBI_FREE(g.history);
    STBI_FREE(g.background);

    // 返回图像数据
    return u;
}
// 检查是否为 Radiance RGBE HDR 文件的核心测试函数
static int stbi__hdr_test_core(stbi__context * s, const char * signature) {
    int i;
    // 遍历文件签名，检查是否与给定签名匹配
    for (i = 0; signature[i]; ++i)
        if (stbi__get8(s) != signature[i])
            return 0;
    // 将文件指针重置到起始位置
    stbi__rewind(s);
    return 1;
}

// 检查是否为 Radiance RGBE HDR 文件的测试函数
static int stbi__hdr_test(stbi__context * s) {
    // 调用核心测试函数，检查文件签名是否为 "#?RADIANCE\n"
    int r = stbi__hdr_test_core(s, "#?RADIANCE\n");
    // 将文件指针重置到起始位置
    stbi__rewind(s);
    // 如果不是 Radiance RGBE HDR 文件，则返回 0
    if (!r) {
// 调用 stbi__hdr_test_core 函数进行 HDR 文件的核心测试
r = stbi__hdr_test_core(s, "#?RGBE\n");
// 将文件指针重新定位到文件开头
stbi__rewind(s);
// 返回测试结果
}
return r;
}

// 定义 HDR 文件读取时的缓冲区大小
#define STBI__HDR_BUFLEN 1024
// 从 stbi__context 中获取一个 token
static char * stbi__hdr_gettoken(stbi__context * z, char * buffer) {
    int len = 0;
    char c = '\0';

    // 从 stbi__context 中获取一个字符
    c = (char)stbi__get8(z);

    // 循环直到文件结束或者遇到换行符
    while (!stbi__at_eof(z) && c != '\n') {
        // 将字符存入缓冲区
        buffer[len++] = c;
        // 如果缓冲区已满，则跳出循环
        if (len == STBI__HDR_BUFLEN - 1) {
            // 清空缓冲区直到行末
            while (!stbi__at_eof(z) && stbi__get8(z) != '\n')
                ;
            break;
        }
        // 从输入流中获取一个字节，转换成字符
        c = (char)stbi__get8(z);
    }

    // 在缓冲区的末尾添加一个空字符，表示字符串结束
    buffer[len] = 0;
    // 返回缓冲区内容
    return buffer;
}

// 将输入的 HDR 数据转换成指定通道数的浮点数输出
static void stbi__hdr_convert(float * output, stbi_uc * input, int req_comp) {
    // 如果输入数据的第四个字节不为0
    if (input[3] != 0) {
        float f1;
        // 指数
        f1 = (float)ldexp(1.0f, input[3] - (int)(128 + 8));
        // 如果请求的通道数小于等于2
        if (req_comp <= 2)
            // 计算输出的第一个通道值
            output[0] = (input[0] + input[1] + input[2]) * f1 / 3;
        else {
            // 计算输出的三个通道值
            output[0] = input[0] * f1;
            output[1] = input[1] * f1;
            output[2] = input[2] * f1;
        }
# 如果 req_comp 等于 2，则将 output[1] 设置为 1
if (req_comp == 2)
    output[1] = 1;
# 如果 req_comp 等于 4，则将 output[3] 设置为 1
if (req_comp == 4)
    output[3] = 1;
# 如果 req_comp 不是 2 或 4，则执行以下操作
} else {
    # 根据 req_comp 的不同取值进行不同的操作
    switch (req_comp) {
    # 如果 req_comp 等于 4，则将 output[3] 设置为 1，并继续执行下一个 case
    case 4:
        output[3] = 1; /* fallthrough */
    # 如果 req_comp 等于 3，则将 output[0]、output[1]、output[2] 设置为 0
    case 3:
        output[0] = output[1] = output[2] = 0;
        break;
    # 如果 req_comp 等于 2，则将 output[1] 设置为 1，并继续执行下一个 case
    case 2:
        output[1] = 1; /* fallthrough */
    # 如果 req_comp 等于 1，则将 output[0] 设置为 0
    case 1:
        output[0] = 0;
        break;
    }
}
// 加载 HDR 图像数据
static float * stbi__hdr_load(stbi__context * s, int * x, int * y, int * comp, int req_comp, stbi__result_info * ri) {
    // 用于存储临时数据的缓冲区
    char buffer[STBI__HDR_BUFLEN];
    // 用于存储分隔后的字符串
    char * token;
    // 用于标记是否数据有效
    int valid = 0;
    // 图像的宽度和高度
    int width, height;
    // 用于存储扫描线数据
    stbi_uc * scanline;
    // 用于存储 HDR 数据
    float * hdr_data;
    // 用于存储长度
    int len;
    // 用于存储计数和值
    unsigned char count, value;
    // 用于循环计数
    int i, j, k, c1, c2, z;
    // 用于存储头部标识符
    const char * headerToken;
    // 忽略未使用的参数
    STBI_NOTUSED(ri);

    // 检查标识符
    headerToken = stbi__hdr_gettoken(s, buffer);
    // 如果标识符不是 "#?RADIANCE" 或 "#?RGBE"，则返回错误
    if (strcmp(headerToken, "#?RADIANCE") != 0 && strcmp(headerToken, "#?RGBE") != 0)
        return stbi__errpf("not HDR", "Corrupt HDR image");

    // 解析头部信息
    for (;;) {
        // 从输入流中获取一个token
        token = stbi__hdr_gettoken(s, buffer);
        // 如果token的第一个字符是0，表示读取结束，跳出循环
        if (token[0] == 0)
            break;
        // 如果token等于"FORMAT=32-bit_rle_rgbe"，则表示格式有效
        if (strcmp(token, "FORMAT=32-bit_rle_rgbe") == 0)
            valid = 1;
    }

    // 如果格式无效，返回错误信息
    if (!valid)
        return stbi__errpf("unsupported format", "Unsupported HDR format");

    // 解析图像的宽度和高度
    // 如果不使用stdio，无法使用sscanf()函数
    token = stbi__hdr_gettoken(s, buffer);
    // 如果token的前3个字符不是"-Y "，表示数据布局不支持，返回错误信息
    if (strncmp(token, "-Y ", 3))
        return stbi__errpf("unsupported data layout", "Unsupported HDR format");
    token += 3;
    // 将token转换为整数，表示图像的高度
    height = (int)strtol(token, &token, 10);
    // 跳过空格
    while (*token == ' ')
        ++token;
    // 如果token的前3个字符不是"+X "，表示数据布局不支持
    if (strncmp(token, "+X ", 3))
    # 返回错误信息，表示不支持的数据布局和不支持的 HDR 格式
    return stbi__errpf("unsupported data layout", "Unsupported HDR format");
    # 将 token 指针向后移动 3 个位置
    token += 3;
    # 将 token 指向的字符串转换为整数，作为图片的宽度
    width = (int)strtol(token, NULL, 10);

    # 如果图片高度超过最大限制，返回错误信息
    if (height > STBI_MAX_DIMENSIONS)
        return stbi__errpf("too large", "Very large image (corrupt?)");
    # 如果图片宽度超过最大限制，返回错误信息
    if (width > STBI_MAX_DIMENSIONS)
        return stbi__errpf("too large", "Very large image (corrupt?)");

    # 将图片宽度赋值给 x
    *x = width;
    # 将图片高度赋值给 y
    *y = height;

    # 如果 comp 不为空，将其赋值为 3
    if (comp)
        *comp = 3;
    # 如果 req_comp 为 0，将其赋值为 3
    if (req_comp == 0)
        req_comp = 3;

    # 如果图片的大小超过了浮点数的最大限制，返回错误信息
    if (!stbi__mad4sizes_valid(width, height, req_comp, sizeof(float), 0))
        return stbi__errpf("too large", "HDR image is too large");
    // 读取数据
    // 分配内存以存储 HDR 数据
    hdr_data = (float *)stbi__malloc_mad4(width, height, req_comp, sizeof(float), 0);
    // 如果内存分配失败，则返回错误信息
    if (!hdr_data)
        return stbi__errpf("outofmem", "Out of memory");

    // 加载图像数据
    // 图像数据以一定数量的 sca 存储
    if (width < 8 || width >= 32768) {
        // 读取扁平数据
        for (j = 0; j < height; ++j) {
            for (i = 0; i < width; ++i) {
                stbi_uc rgbe[4];
                // 从输入流中获取 4 个字节的数据
                stbi__getn(s, rgbe, 4);
                // 将获取的数据转换为 HDR 数据格式
                stbi__hdr_convert(hdr_data + j * width * req_comp + i * req_comp, rgbe, req_comp);
            }
        }
    } else {
        // 读取 RLE 编码数据
        scanline = NULL;
// 遍历图像的高度
for (j = 0; j < height; ++j) {
    // 从输入流中获取下一个字节作为c1
    c1 = stbi__get8(s);
    // 从输入流中获取下一个字节作为c2
    c2 = stbi__get8(s);
    // 从输入流中获取下一个字节作为len
    len = stbi__get8(s);
    // 如果不是运行长度编码，需要使用这些数据作为解码后的像素
    if (c1 != 2 || c2 != 2 || (len & 0x80)) {
        // 创建一个包含4个元素的无符号字符数组rgbe
        stbi_uc rgbe[4];
        // 将c1、c2、len和下一个字节分别存入rgbe数组
        rgbe[0] = (stbi_uc)c1;
        rgbe[1] = (stbi_uc)c2;
        rgbe[2] = (stbi_uc)len;
        rgbe[3] = (stbi_uc)stbi__get8(s);
        // 将rgbe数组转换为hdr_data格式，并指定通道数
        stbi__hdr_convert(hdr_data, rgbe, req_comp);
        // 重置i和j的值
        i = 1;
        j = 0;
        // 释放scanline的内存
        STBI_FREE(scanline);
        // 跳转到主解码循环的开始处
        goto main_decode_loop; // 是的，这没有意义
    }
    // 将len左移8位
    len <<= 8;
}
            # 读取一个字节并将其与len进行按位或操作
            len |= stbi__get8(s);
            # 如果读取的长度与宽度不相等，则释放内存并返回错误信息
            if (len != width) {
                STBI_FREE(hdr_data);
                STBI_FREE(scanline);
                return stbi__errpf("invalid decoded scanline length", "corrupt HDR");
            }
            # 如果扫描行为空，则分配内存
            if (scanline == NULL) {
                scanline = (stbi_uc *)stbi__malloc_mad2(width, 4, 0);
                # 如果分配内存失败，则释放hdr_data内存并返回错误信息
                if (!scanline) {
                    STBI_FREE(hdr_data);
                    return stbi__errpf("outofmem", "Out of memory");
                }
            }

            # 循环遍历RGBA四个通道
            for (k = 0; k < 4; ++k) {
                int nleft;
                i = 0;
                # 当还有剩余像素时，继续读取像素值
                while ((nleft = width - i) > 0) {
                    count = stbi__get8(s);
                    # 如果像素值大于128，则进行处理
                    if (count > 128) {
// 读取一个字节作为值
value = stbi__get8(s);
// 减去128，表示接下来的count个值都是value
count -= 128;
// 如果count为0或者大于剩余的像素数，说明数据损坏，释放内存并返回错误信息
if ((count == 0) || (count > nleft)) {
    STBI_FREE(hdr_data);
    STBI_FREE(scanline);
    return stbi__errpf("corrupt", "bad RLE data in HDR");
}
// 将count个值设置为value
for (z = 0; z < count; ++z)
    scanline[i++ * 4 + k] = value;
} else {
    // 如果count为0或者大于剩余的像素数，说明数据损坏，释放内存并返回错误信息
    if ((count == 0) || (count > nleft)) {
        STBI_FREE(hdr_data);
        STBI_FREE(scanline);
        return stbi__errpf("corrupt", "bad RLE data in HDR");
    }
    // 读取count个值作为像素值
    for (z = 0; z < count; ++z)
        scanline[i++ * 4 + k] = stbi__get8(s);
}
    }
            }  // 结束循环
        }  // 结束循环
        // 遍历每个像素，将 HDR 数据转换为所需的通道数
        for (i = 0; i < width; ++i)
            stbi__hdr_convert(hdr_data + (j * width + i) * req_comp, scanline + i * 4, req_comp);
    }
    // 如果扫描行存在，则释放内存
    if (scanline)
        STBI_FREE(scanline);
    // 返回 HDR 数据
    return hdr_data;
}

// 获取 HDR 图像的信息
static int stbi__hdr_info(stbi__context * s, int * x, int * y, int * comp) {
    char buffer[STBI__HDR_BUFLEN];  // 创建缓冲区
    char * token;  // 创建指针
    int valid = 0;  // 初始化有效标志
    int dummy;  // 创建虚拟变量

    if (!x)  // 如果 x 为空
        x = &dummy;  // 将 x 指向虚拟变量
    // 如果 y 为空，则将其指向 dummy
    if (!y)
        y = &dummy;
    // 如果 comp 为空，则将其指向 dummy
    if (!comp)
        comp = &dummy;

    // 如果不是 HDR 格式的文件，则将文件指针重置并返回 0
    if (stbi__hdr_test(s) == 0) {
        stbi__rewind(s);
        return 0;
    }

    // 循环读取 token，直到遇到空字符
    for (;;) {
        token = stbi__hdr_gettoken(s, buffer);
        // 如果 token 是空字符，则跳出循环
        if (token[0] == 0)
            break;
        // 如果 token 是 "FORMAT=32-bit_rle_rgbe"，则将 valid 置为 1
        if (strcmp(token, "FORMAT=32-bit_rle_rgbe") == 0)
            valid = 1;
    }

    // 如果 valid 为假，则将文件指针重置
    if (!valid) {
        stbi__rewind(s);
    # 返回值为0
    return 0;
    # 读取下一个标记
    token = stbi__hdr_gettoken(s, buffer);
    # 如果标记不是"-Y "，则回退并返回0
    if (strncmp(token, "-Y ", 3)) {
        stbi__rewind(s);
        return 0;
    }
    # 跳过"-Y "，获取高度值
    token += 3;
    *y = (int)strtol(token, &token, 10);
    # 跳过空格
    while (*token == ' ')
        ++token;
    # 如果标记不是"+X "，则回退并返回0
    if (strncmp(token, "+X ", 3)) {
        stbi__rewind(s);
        return 0;
    }
    # 跳过"+X "，获取宽度值
    token += 3;
    *x = (int)strtol(token, NULL, 10);
    # 设置组件数为3
    *comp = 3;
    # 返回1
    return 1;
}
// 如果定义了 STBI_NO_BMP，则不执行以下代码
#ifndef STBI_NO_BMP
// 获取 BMP 图像信息
static int stbi__bmp_info(stbi__context * s, int * x, int * y, int * comp) {
    void * p;
    stbi__bmp_data info;

    // 设置默认 alpha 值为 255
    info.all_a = 255;
    // 解析 BMP 头部信息
    p = stbi__bmp_parse_header(s, &info);
    // 如果解析失败，回到文件开头并返回 0
    if (p == NULL) {
        stbi__rewind(s);
        return 0;
    }
    // 如果 x 不为空，将图像宽度赋值给 x
    if (x)
        *x = s->img_x;
    // 如果 y 不为空，将图像高度赋值给 y
    if (y)
        *y = s->img_y;
    // 如果 comp 不为空
    if (comp) {
        // 如果图像位深为 24 且 alpha 通道为 0xff000000，则设置 comp 为 3
        if (info.bpp == 24 && info.ma == 0xff000000)
            *comp = 3;
```

#ifndef STBI_NO_PSD
// 如果未定义 STBI_NO_PSD，则编译以下代码

static int stbi__psd_info(stbi__context * s, int * x, int * y, int * comp) {
    // 定义变量 channelCount, dummy, depth
    int channelCount, dummy, depth;
    // 如果 x 为空指针，则指向 dummy
    if (!x)
        x = &dummy;
    // 如果 y 为空指针，则指向 dummy
    if (!y)
        y = &dummy;
    // 如果 comp 为空指针，则指向 dummy
    if (!comp)
        comp = &dummy;
    // 如果读取的值不等于 0x38425053，则回退并返回 0
    if (stbi__get32be(s) != 0x38425053) {
        stbi__rewind(s);
        return 0;
    }
```

# 如果读取的数据不是16位大端序的1，就将流倒回，返回0
if (stbi__get16be(s) != 1) {
    stbi__rewind(s);
    return 0;
}
# 跳过6个字节
stbi__skip(s, 6);
# 读取通道数
channelCount = stbi__get16be(s);
# 如果通道数小于0或大于16，就将流倒回，返回0
if (channelCount < 0 || channelCount > 16) {
    stbi__rewind(s);
    return 0;
}
# 读取图像的高度和宽度
*y = stbi__get32be(s);
*x = stbi__get32be(s);
# 读取图像的深度
depth = stbi__get16be(s);
# 如果深度不是8或16，就将流倒回，返回0
if (depth != 8 && depth != 16) {
    stbi__rewind(s);
    return 0;
}
# 如果读取的数据不是16位大端序的3，就将流倒回，返回0
if (stbi__get16be(s) != 3) {
    stbi__rewind(s);
    return 0;
}
    }
    *comp = 4;  // 将指针 comp 指向的地址赋值为 4
    return 1;    // 返回 1，表示函数执行成功

}

static int stbi__psd_is16(stbi__context * s) {
    int channelCount, depth;  // 声明变量 channelCount 和 depth
    if (stbi__get32be(s) != 0x38425053) {  // 如果读取的 32 位大端数据不等于 0x38425053
        stbi__rewind(s);  // 将读取指针回退到起始位置
        return 0;  // 返回 0，表示函数执行失败
    }
    if (stbi__get16be(s) != 1) {  // 如果读取的 16 位大端数据不等于 1
        stbi__rewind(s);  // 将读取指针回退到起始位置
        return 0;  // 返回 0，表示函数执行失败
    }
    stbi__skip(s, 6);  // 跳过 6 个字节
    channelCount = stbi__get16be(s);  // 读取 16 位大端数据并赋值给 channelCount
    if (channelCount < 0 || channelCount > 16) {  // 如果 channelCount 小于 0 或大于 16
        stbi__rewind(s);  // 将读取指针回退到起始位置
        return 0;  // 返回 0，表示函数执行失败
    }
    // 忽略下一个32位大端序的数据
    STBI_NOTUSED(stbi__get32be(s));
    // 忽略下一个32位大端序的数据
    STBI_NOTUSED(stbi__get32be(s));
    // 读取16位大端序的数据，存储在depth变量中
    depth = stbi__get16be(s);
    // 如果depth不等于16，则将文件指针重置，返回0
    if (depth != 16) {
        stbi__rewind(s);
        return 0;
    }
    // 返回1
    return 1;
}
#endif

#ifndef STBI_NO_PIC
// 获取PIC格式图片的信息
static int stbi__pic_info(stbi__context * s, int * x, int * y, int * comp) {
    int act_comp = 0, num_packets = 0, chained, dummy;
    // 存储PIC格式图片的数据包
    stbi__pic_packet packets[10];

    // 如果x为NULL，则将x指向dummy
    if (!x)
        x = &dummy;
    // 如果y为NULL，则将y指向dummy
    // 将 y 的地址指向 dummy 变量的地址
    y = &dummy;
    // 如果 comp 为空，则将其指向 dummy 变量的地址
    if (!comp)
        comp = &dummy;

    // 检查文件是否以特定的字节序列开头，如果不是则返回 0
    if (!stbi__pic_is4(s, "\x53\x80\xF6\x34")) {
        // 将文件指针重置到文件开头
        stbi__rewind(s);
        return 0;
    }

    // 跳过文件的前 88 个字节
    stbi__skip(s, 88);

    // 从文件中读取两个字节作为 x 的值
    *x = stbi__get16be(s);
    // 从文件中读取两个字节作为 y 的值
    *y = stbi__get16be(s);
    // 如果已经到达文件末尾，则将文件指针重置到文件开头并返回 0
    if (stbi__at_eof(s)) {
        stbi__rewind(s);
        return 0;
    }
    // 如果 x 不为 0 并且 (1 << 28) / x 小于 y，则将文件指针重置到文件开头并返回 0
    if ((*x) != 0 && (1 << 28) / (*x) < (*y)) {
        stbi__rewind(s);
        return 0;
    }
    }

    // 跳过8个字节
    stbi__skip(s, 8);

    do {
        stbi__pic_packet * packet;

        // 如果数据包数量达到上限，返回0
        if (num_packets == sizeof(packets) / sizeof(packets[0]))
            return 0;

        // 获取下一个数据包
        packet = &packets[num_packets++];
        // 读取chained、size、type、channel字段
        chained = stbi__get8(s);
        packet->size = stbi__get8(s);
        packet->type = stbi__get8(s);
        packet->channel = stbi__get8(s);
        // 更新活跃通道标志
        act_comp |= packet->channel;

        // 如果已经到达文件末尾，回到文件开头并返回0
        if (stbi__at_eof(s)) {
            stbi__rewind(s);
            return 0;
    }
    // 如果数据包大小不等于8，则将流指针重置并返回0
    if (packet->size != 8) {
        stbi__rewind(s);
        return 0;
    }
} while (chained);

// 根据act_comp的值确定comp的值，然后返回1
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
// 如果定义了 STBI_NO_PNM，则不支持 PNM 格式，直接返回
#ifndef STBI_NO_PNM

// 检测文件是否为 PNM 格式
static int stbi__pnm_test(stbi__context * s) {
    char p, t;
    // 读取文件的前两个字符
    p = (char)stbi__get8(s);
    t = (char)stbi__get8(s);
    // 如果不是以 'P' 开头，或者后面的字符不是 '5' 或 '6'，则不是 PNM 格式，返回 0
    if (p != 'P' || (t != '5' && t != '6')) {
        // 将文件指针重置到起始位置
        stbi__rewind(s);
        return 0;
    }
    // 是 PNM 格式，返回 1
    return 1;
}

// 加载 PNM 格式的文件
static void * stbi__pnm_load(stbi__context * s, int * x, int * y, int * comp, int req_comp, stbi__result_info * ri) {
    stbi_uc * out;
    // 忽略 ri 参数
    STBI_NOTUSED(ri);
    # 通过调用 stbi__pnm_info 函数获取每个通道的位数，并将结果存储在 ri->bits_per_channel 中
    ri->bits_per_channel = stbi__pnm_info(s, (int *)&s->img_x, (int *)&s->img_y, (int *)&s->img_n);
    # 如果 bits_per_channel 为 0，则返回 0
    if (ri->bits_per_channel == 0)
        return 0;

    # 如果图像的高度超过了最大限制，则返回错误信息
    if (s->img_y > STBI_MAX_DIMENSIONS)
        return stbi__errpuc("too large", "Very large image (corrupt?)");
    # 如果图像的宽度超过了最大限制，则返回错误信息
    if (s->img_x > STBI_MAX_DIMENSIONS)
        return stbi__errpuc("too large", "Very large image (corrupt?)");

    # 将图像的宽度和高度分别存储在指针 x 和 y 指向的位置
    *x = s->img_x;
    *y = s->img_y;
    # 如果 comp 不为空，则将图像的通道数存储在指针 comp 指向的位置
    if (comp)
        *comp = s->img_n;

    # 检查图像的大小是否超出了合理范围，如果是，则返回错误信息
    if (!stbi__mad4sizes_valid(s->img_n, s->img_x, s->img_y, ri->bits_per_channel / 8, 0))
        return stbi__errpuc("too large", "PNM too large");

    # 分配内存以存储图像数据，并将结果存储在 out 中
    out = (stbi_uc *)stbi__malloc_mad4(s->img_n, s->img_x, s->img_y, ri->bits_per_channel / 8, 0);
    # 如果分配内存失败，则返回错误信息
    if (!out)
// 如果内存不足，返回“内存不足”的错误信息
return stbi__errpuc("outofmem", "Out of memory");
// 从输入流中读取数据，存储到out中，读取的字节数为s->img_n * s->img_x * s->img_y * (ri->bits_per_channel / 8)
if (!stbi__getn(s, out, s->img_n * s->img_x * s->img_y * (ri->bits_per_channel / 8))) {
    // 如果读取失败，释放out内存，返回“PNM文件被截断”的错误信息
    STBI_FREE(out);
    return stbi__errpuc("bad PNM", "PNM file truncated");
}

// 如果请求的通道数不等于图像的通道数
if (req_comp && req_comp != s->img_n) {
    // 如果每个通道的位数为16
    if (ri->bits_per_channel == 16) {
        // 转换图像格式为16位
        out = (stbi_uc *)stbi__convert_format16((stbi__uint16 *)out, s->img_n, req_comp, s->img_x, s->img_y);
    } else {
        // 转换图像格式
        out = stbi__convert_format(out, s->img_n, req_comp, s->img_x, s->img_y);
    }
    // 如果转换失败，返回空指针，stbi__convert_format在失败时会释放输入
    if (out == NULL)
        return out;
}
// 返回转换后的图像数据
return out;
}

// 判断字符是否为空白字符
static int stbi__pnm_isspace(char c) { return c == ' ' || c == '\t' || c == '\n' || c == '\v' || c == '\f' || c == '\r'; }
// 跳过空白字符，包括空格、制表符等
static void stbi__pnm_skip_whitespace(stbi__context * s, char * c) {
    for (;;) {
        // 跳过空白字符
        while (!stbi__at_eof(s) && stbi__pnm_isspace(*c))
            *c = (char)stbi__get8(s);

        // 如果遇到注释符号'#'，则跳过注释内容直到换行符或回车符
        if (stbi__at_eof(s) || *c != '#')
            break;

        while (!stbi__at_eof(s) && *c != '\n' && *c != '\r')
            *c = (char)stbi__get8(s);
    }
}

// 判断字符是否为数字
static int stbi__pnm_isdigit(char c) { return c >= '0' && c <= '9'; }

// 获取整数值
static int stbi__pnm_getinteger(stbi__context * s, char * c) {
    int value = 0;

    // 循环读取字符，直到遇到非数字字符
    while (!stbi__at_eof(s) && stbi__pnm_isdigit(*c)) {
        // 将字符转换为数字并累加到整数值中
        value = value * 10 + (*c - '0');
        *c = (char)stbi__get8(s); // 从输入流中读取一个字节，并将其转换为字符类型赋值给指针 c
        if ((value > 214748364) || (value == 214748364 && *c > '7')) // 如果 value 大于 214748364 或者等于 214748364 且指针 c 大于 '7'
            return stbi__err("integer parse overflow", "Parsing an integer in the PPM header overflowed a 32-bit int"); // 返回整数解析溢出的错误信息
    }

    return value; // 返回解析出的整数值
}

static int stbi__pnm_info(stbi__context * s, int * x, int * y, int * comp) {
    int maxv, dummy; // 定义变量 maxv 和 dummy
    char c, p, t; // 定义字符变量 c, p, t

    if (!x) // 如果 x 为空
        x = &dummy; // 将 x 指向 dummy 变量
    if (!y) // 如果 y 为空
        y = &dummy; // 将 y 指向 dummy 变量
    if (!comp) // 如果 comp 为空
        comp = &dummy; // 将 comp 指向 dummy 变量

    stbi__rewind(s); // 将输入流指针重置到起始位置
```

// 获取标识符
p = (char)stbi__get8(s); // 从输入流中获取一个字节作为标识符的第一个字符
t = (char)stbi__get8(s); // 从输入流中获取一个字节作为标识符的第二个字符
if (p != 'P' || (t != '5' && t != '6')) { // 如果标识符不是'P'，或者第二个字符不是'5'或'6'，则回退输入流并返回0
    stbi__rewind(s);
    return 0;
}

*comp = (t == '6') ? 3 : 1; // 如果第二个字符是'6'，则设置组件数为3，否则设置为1

c = (char)stbi__get8(s); // 从输入流中获取一个字节
stbi__pnm_skip_whitespace(s, &c); // 跳过空白字符

*x = stbi__pnm_getinteger(s, &c); // 读取图像宽度
if (*x == 0)
    return stbi__err("invalid width", "PPM image header had zero or overflowing width"); // 如果宽度为0，则返回错误
stbi__pnm_skip_whitespace(s, &c); // 跳过空白字符

*y = stbi__pnm_getinteger(s, &c); // 读取图像高度
    // 如果像素的值为0，返回错误信息
    if (*y == 0)
        return stbi__err("invalid width", "PPM image header had zero or overflowing width");
    // 跳过空白字符
    stbi__pnm_skip_whitespace(s, &c);

    // 读取最大像素值
    maxv = stbi__pnm_getinteger(s, &c); 
    // 如果最大像素值大于65535，返回错误信息
    if (maxv > 65535)
        return stbi__err("max value > 65535", "PPM image supports only 8-bit and 16-bit images");
    // 如果最大像素值大于255，返回16
    else if (maxv > 255)
        return 16;
    // 否则返回8
    else
        return 8;
}

// 判断是否为16位图像
static int stbi__pnm_is16(stbi__context * s) {
    // 如果图像信息为16，返回1
    if (stbi__pnm_info(s, NULL, NULL, NULL) == 16)
        return 1;
    // 否则返回0
    return 0;
}
# 定义一个函数，用于获取图像文件的信息，包括宽度、高度和通道数
static int stbi__info_main(stbi__context * s, int * x, int * y, int * comp) {
    # 如果图像是 JPEG 格式，调用 stbi__jpeg_info 函数获取信息
    # 如果成功获取信息，返回 1
    # 如果不是 JPEG 格式，继续执行下面的代码
#ifndef STBI_NO_JPEG
    if (stbi__jpeg_info(s, x, y, comp))
        return 1;
#endif

    # 如果图像是 PNG 格式，调用 stbi__png_info 函数获取信息
    # 如果成功获取信息，返回 1
    # 如果不是 PNG 格式，继续执行下面的代码
#ifndef STBI_NO_PNG
    if (stbi__png_info(s, x, y, comp))
        return 1;
#endif

    # 如果图像是 GIF 格式，调用 stbi__gif_info 函数获取信息
    # 如果成功获取信息，返回 1
    # 如果不是 GIF 格式，继续执行下面的代码
#ifndef STBI_NO_GIF
    if (stbi__gif_info(s, x, y, comp))
        return 1;
#endif

    # 如果图像是 BMP 格式，调用 stbi__bmp_info 函数获取信息
    # 如果成功获取信息，返回 1
    # 如果不是 BMP 格式，继续执行下面的代码
#ifndef STBI_NO_BMP
    if (stbi__bmp_info(s, x, y, comp))
        return 1;
#endif
// 如果定义了 STBI_NO_PSD，则不执行 stbi__psd_info 函数，否则执行该函数获取 PSD 文件信息
#ifndef STBI_NO_PSD
    if (stbi__psd_info(s, x, y, comp))
        return 1;
#endif

// 如果定义了 STBI_NO_PIC，则不执行 stbi__pic_info 函数，否则执行该函数获取 PIC 文件信息
#ifndef STBI_NO_PIC
    if (stbi__pic_info(s, x, y, comp))
        return 1;
#endif

// 如果定义了 STBI_NO_PNM，则不执行 stbi__pnm_info 函数，否则执行该函数获取 PNM 文件信息
#ifndef STBI_NO_PNM
    if (stbi__pnm_info(s, x, y, comp))
        return 1;
#endif

// 如果定义了 STBI_NO_HDR，则不执行 stbi__hdr_info 函数，否则执行该函数获取 HDR 文件信息
#ifndef STBI_NO_HDR
    if (stbi__hdr_info(s, x, y, comp))
        return 1;
#endif
// 如果未定义 STBI_NO_TGA，则进行 TGA 图像格式的检测
#ifndef STBI_NO_TGA
    // 如果是 TGA 格式的图像，则返回 1
    if (stbi__tga_info(s, x, y, comp))
        return 1;
#endif
    // 如果不是已知类型的图像，或者图像损坏，则返回错误信息
    return stbi__err("unknown image type", "Image not of any known type, or corrupt");
}

// 检测是否为 16 位图像
static int stbi__is_16_main(stbi__context * s) {
#ifndef STBI_NO_PNG
    // 如果是 PNG 格式的 16 位图像，则返回 1
    if (stbi__png_is16(s))
        return 1;
#endif

#ifndef STBI_NO_PSD
    // 如果是 PSD 格式的 16 位图像，则返回 1
    if (stbi__psd_is16(s))
        return 1;
#endif
#ifndef STBI_NO_PNM
    // 如果定义了 STBI_NO_PNM，则不执行以下代码
    if (stbi__pnm_is16(s))
        // 如果输入的数据是16位 PNM 格式，则返回1
        return 1;
#endif
    // 否则返回0
    return 0;
}

#ifndef STBI_NO_STDIO
// 获取文件信息，包括宽度、高度和通道数
STBIDEF int stbi_info(char const * filename, int * x, int * y, int * comp) {
    // 打开文件
    FILE * f = stbi__fopen(filename, "rb");
    int result;
    // 如果文件打开失败，则返回错误信息
    if (!f)
        return stbi__err("can't fopen", "Unable to open file");
    // 从文件中获取信息
    result = stbi_info_from_file(f, x, y, comp);
    // 关闭文件
    fclose(f);
    // 返回获取的信息
    return result;
}

// 从文件中获取信息，包括宽度、高度和通道数
STBIDEF int stbi_info_from_file(FILE * f, int * x, int * y, int * comp) {
    int r;
// 创建一个 stbi__context 结构体
stbi__context s;
// 获取当前文件指针的位置
long pos = ftell(f);
// 初始化 stbi__context 结构体
stbi__start_file(&s, f);
// 获取图像信息并存储在 x, y, comp 中，返回结果存储在 r 中
r = stbi__info_main(&s, x, y, comp);
// 将文件指针移动到指定位置
fseek(f, pos, SEEK_SET);
// 返回获取图像信息的结果
return r;
}

// 从文件名获取图像是否为 16 位的信息
STBIDEF int stbi_is_16_bit(char const * filename) {
    // 打开文件
    FILE * f = stbi__fopen(filename, "rb");
    int result;
    // 如果文件打开失败，返回错误信息
    if (!f)
        return stbi__err("can't fopen", "Unable to open file");
    // 从文件中获取图像是否为 16 位的信息
    result = stbi_is_16_bit_from_file(f);
    // 关闭文件
    fclose(f);
    // 返回获取的结果
    return result;
}

// 从文件指针获取图像是否为 16 位的信息
STBIDEF int stbi_is_16_bit_from_file(FILE * f) {
    int r;
// 定义一个 stbi__context 结构体变量 s
stbi__context s;
// 获取当前文件指针的位置
long pos = ftell(f);
// 初始化 stbi__context 结构体变量 s，开始读取文件
stbi__start_file(&s, f);
// 调用 stbi__is_16_main 函数判断文件是否为16位图像
r = stbi__is_16_main(&s);
// 将文件指针重新定位到之前保存的位置
fseek(f, pos, SEEK_SET);
// 返回判断结果
return r;
#endif // !STBI_NO_STDIO

// 从内存中读取图像信息
STBIDEF int stbi_info_from_memory(stbi_uc const * buffer, int len, int * x, int * y, int * comp) {
    // 定义一个 stbi__context 结构体变量 s
    stbi__context s;
    // 初始化 stbi__context 结构体变量 s，开始读取内存中的数据
    stbi__start_mem(&s, buffer, len);
    // 调用 stbi__info_main 函数获取图像信息
    return stbi__info_main(&s, x, y, comp);
}

// 从回调函数中读取图像信息
STBIDEF int stbi_info_from_callbacks(stbi_io_callbacks const * c, void * user, int * x, int * y, int * comp) {
    // 定义一个 stbi__context 结构体变量 s
    stbi__context s;
    // 初始化 stbi__context 结构体变量 s，开始使用回调函数读取数据
    stbi__start_callbacks(&s, (stbi_io_callbacks *)c, user);
    // 调用 stbi__info_main 函数获取图像信息
    return stbi__info_main(&s, x, y, comp);
}
# 从内存中检查数据是否为16位
STBIDEF int stbi_is_16_bit_from_memory(stbi_uc const * buffer, int len) {
    # 创建 stbi__context 结构体，用于处理内存中的数据
    stbi__context s;
    # 初始化 stbi__context 结构体，传入数据指针和数据长度
    stbi__start_mem(&s, buffer, len);
    # 调用 stbi__is_16_main 函数，检查数据是否为16位
    return stbi__is_16_main(&s);
}

# 从回调函数中检查数据是否为16位
STBIDEF int stbi_is_16_bit_from_callbacks(stbi_io_callbacks const * c, void * user) {
    # 创建 stbi__context 结构体，用于处理回调函数中的数据
    stbi__context s;
    # 初始化 stbi__context 结构体，传入回调函数和用户数据
    stbi__start_callbacks(&s, (stbi_io_callbacks *)c, user);
    # 调用 stbi__is_16_main 函数，检查数据是否为16位
    return stbi__is_16_main(&s);
}

# 结束 STB_IMAGE_IMPLEMENTATION 宏定义
#endif // STB_IMAGE_IMPLEMENTATION

# 修订历史记录
# 2.20  (2019-02-07) 支持 Windows 下的 UTF8 文件名；修复警告和平台条件编译
# 2.19  (2018-02-11) 修复警告
# 2.18  (2018-01-30) 修复警告
# 版本 2.17 (2018-01-29) 修改 sbti__shiftsigned 以避免 clang -O2 的 bug
# 支持 1 位 BMP
# 所有函数都有 16 位变体
# 避免警告
# 版本 2.16 (2017-07-23) 所有函数都有 16 位变体
# STBI_NO_STDIO 再次可用
# 编译修复
# 修复 unpremultiply 中的舍入问题
# 优化垂直翻转
# 禁用原始长度验证
# 文档修复
# 版本 2.15 (2017-03-18) 修复 png-1,2,4 bug; 现在所有 Imagenet JPGs 都可以解码
# 修复警告
# 在 gcc 上禁用运行时 SSE 检测
# 统一处理可选的 "return" 值
# zlib 表的线程安全初始化
# 版本 2.14 (2017-03-03) 移除废弃的 STBI_JPEG_OLD; 修复 Imagenet JPGs 的问题
# 版本 2.13 (2016-11-29) 添加 16 位 API，目前仅支持 PNG
# 版本 2.12 (2016-04-02) 修复 2.11 PSD 修复中导致崩溃的拼写错误
# 版本 2.11 (2016-04-02) 在堆栈上分配大型结构
# 移除透明 PSD 的白色衬垫
# 修复了 PNG 和 BMP 报告的通道数
# 在非 GCC 64 位系统中重新启用 SSE2
# 支持 RGB 格式的 JPEG
# 读取 16 位的 PNG（只作为 8 位处理）
# 2.10 版本（2016-01-22）避免了 2.09 版本引入的警告
# 2.09 版本（2016-01-16）允许 PNM 文件中的注释
# 支持 16 位每像素的 TGA（而不是每分量）
# 修复了 TGA 的 info() 方法由于 .hdr 处理而可能中断的问题
# 修复了 BMP 的 info() 方法，共享代码而不是粗糙的解析
# 如果分配器不支持 realloc，可以使用 STBI_REALLOC_SIZED
# 代码清理
# 2.08 版本（2015-09-13）修复了 2.07 版本的问题，将 RGB PSD 读取为 RGBA
# 2.07 版本（2015-09-13）修复了编译器警告
# 部分动画 GIF 支持
# 有限的 16 位每通道 PSD 支持
# 未使用的函数使用 #ifdef 进行条件编译
# 修复了 < 92 字节的 PIC、PNM、HDR、TGA 文件的 bug
# 2.06 版本（2015-04-19）修复了 PSD 返回错误 '*comp' 值的 bug
# 2.05 版本（2015-04-19）修复了渐进式 JPEG 处理中的 bug，修复了警告
# 2.04 版本（2015-04-15）尝试在 MinGW 64 位系统上重新启用 SIMD
# 版本 2.03 (2015-04-12) 增加了额外的损坏检查 (mmozeiko)
# 设置在加载时翻转图像的函数 stbi_set_flip_vertically_on_load (nguillemot)
# 修复了 NEON 支持和 mingw 支持
# 版本 2.02 (2015-01-19) 修复了不正确的断言，修复了警告
# 版本 2.01 (2015-01-17) 修复了各种警告；在没有 -msse2 的情况下抑制了 gcc 32 位的 SIMD
# 版本 2.00b (2014-12-25) 修复了渐进式 JPEG 中的 STBI_MALLOC
# 版本 2.00 (2014-12-25) 优化了 JPG，包括 x86 SSE2 和 NEON SIMD (ryg)
# 渐进式 JPEG (stb)
# PGM/PPM 支持 (Ken Miller)
# STBI_MALLOC,STBI_REALLOC,STBI_FREE
# 修复了 GIF 的 bug -- 似乎从未工作过
# STBI_NO_*, STBI_ONLY_*
# 版本 1.48 (2014-12-14) 修复了错误命名的断言()
# 版本 1.47 (2014-12-14) 1/2/4 位 PNG 支持，直接和调色板两种方式 (Omar Cornut & stb)
# 优化了 PNG (ryg)
# 修复了带有用户指定通道数的隔行式 PNG 中的 bug (stb)
# 版本 1.46 (2014-08-26) 修复了非调色板 PNG 中损坏的 tRNS 块 (颜色键式透明度)
# 版本 1.45 (2014-08-16) 通过包装 malloc 修复了 MSVC-ARM 内部编译器错误
# 版本 1.44 (2014-08-07)
# 从 Ronny Chevalier 进行了各种警告修复
# 版本 1.43 (2014-07-15)
# 修复了在 1.42 中更改的代码中仅适用于 MSVC 编译器的问题
# 版本 1.42 (2014-07-09)
# 不再定义 _CRT_SECURE_NO_WARNINGS（影响用户代码）
# 修复 stbi__cleanup_jpeg 路径的问题
# 添加 STBI_ASSERT 以避免需要 assert.h
# 版本 1.41 (2014-06-25)
# 修复了从 1.36 开始的搜索和替换，导致注释/错误消息混乱
# 版本 1.40 (2014-06-22)
# 修复了 gcc 结构初始化警告
# 版本 1.39 (2014-06-15)
# 修复了 TGA 优化，当 req_comp != TGA 中组件的数量时；
# 修复了 GIF 加载，因为 BMP 没有倒带（糟糕，在我的测试套件中没有 GIF）
# 添加对 BMP 版本 5 的支持（更多被忽略的字段）
# 版本 1.38 (2014-06-06)
# 抑制 MSVC 警告，整数转换截断值
# 修复了意外重命名 I/O 的 'skip' 字段
# 版本 1.37 (2014-06-04)
# 版本 1.36 (2014-06-03)
# 移除重复的 typedef
# 将代码转换为单文件头文件库
# 如果未设置 de-iphone，则加载 iPhone 图像的颜色交换，而不是返回 NULL

# 版本 1.35 (2014-05-27)
# 修复各种警告
# 修复损坏的 STBI_SIMD 路径
# 修复 stbi_load_from_file 中不再将文件指针留在正确位置的 bug
# 修复 32 位 BMP 的非 easy 路径（可能从未使用过）
# TGA 优化由 Arseny Kapoulkine 提供

# 版本 1.34 (未知)
# 在 stbi__resample_row_generic() 中使用 STBI_NOTUSED，修复 tga 失败情况下的另一个泄漏

# 版本 1.33 (2011-07-14)
# 使 stbi_is_hdr 在 STBI_NO_HDR 中工作（如规定的那样），进行一些编译器友好的改进

# 版本 1.32 (2011-07-13)
# 支持所有支持的文件类型的 "info" 函数（SpartanJ 提供）

# 版本 1.31 (2011-06-20)
# 修复了一些泄漏问题，修复了 PNG 处理中的 bug（SpartanJ 提供）

# 版本 1.30 (2011-06-11)
# 添加了通过回调加载文件的能力，以适应自定义输入流（Ben Wenger 提供）
# 该部分代码似乎是一系列更新日志或者变更记录，但是缺少具体的代码内容，无法为每个语句添加注释。
# 从Jean-Marc Lienher添加了GIF支持
# 从James Brown添加了iPhone PNG扩展
# 从Nicolas Schulz和Janez Zemva（i.stbi__err. Janez (U+017D)emva）解决了警告问题
# 1.21 修复了头文件中对'stbi_uc'的使用（由jon blow报告）
# 1.20 由Tom Seddon添加了对Softimage PIC的支持
# 1.19 修复了交错PNG损坏检查中的错误（ryg发现）
# 1.18（2008-08-02）修复了一个线程bug（本地可变静态）
# 1.17 支持交错PNG
# 1.16 重大bug修复 - stbi__convert_format转换了一个多余的像素
# 1.15 为了线程安全性初始化了一些字段
# 1.14 修复了线程安全的转换bug
# 头文件版本（在包含之前定义#define STBI_HEADER_FILE_ONLY）
# 1.13 线程安全
# 1.12 API中的const限定符
# 1.11 支持可安装的IDCT，颜色空间转换例程
# 1.10 修复了64位的问题（不使用"unsigned long"）
# 由Fabian "ryg" Giesen优化的上采样
# 1.09 修复了PSD代码的格式转换问题（全局变量不好）
# 1.08 Nicolas Schulz整合了Thatcher Ulrich的PSD代码
# 1.07    尝试再次修复 C++ 的警告/错误
# 1.06    尝试再次修复 C++ 的警告/错误
# 1.05    修复 TGA 加载以返回正确的 *comp 并使用良好的亮度计算
# 1.04    默认浮点 alpha 为 1，而不是 255；使用 'void *' 作为 stbi_image_free 的参数
# 1.03    修复 STBI_NO_STDIO、STBI_NO_HDR 的 bug
# 1.02    支持（部分）HDR 文件，提供浮点接口以优先访问它们
# 1.01    修复 bug：处理正常位图时可能出现的 bug... 不确定
#         修复 bug：stbi__bmp_load() 和 stbi__tga_load() 函数根本不起作用
# 1.00    跳过 zlib 头部的接口
# 0.99    正确处理调色板中的 alpha
# 0.98    lonesock 的 TGA 加载器；动态添加加载器（未经测试）
# 0.97    处理过大文件的 jpeg 错误；还捕获另一个 malloc 失败
# 0.96    修复无效 v 值的检测 - particleman@mollyrocket 论坛
# 0.95    在头部扫描期间，寻找标记以处理填充
# 0.94    使用 STBI_NO_STDIO 禁用 stdio 使用；将所有 #define 重命名为相同的名称
# 0.93    处理 jpegtran 输出；详细错误信息
# 0.92    读取多种格式的 4、8、16、24、32 位 BMP 文件
# 0.91    输出 24 位 Windows 3.0 BMP 文件
# 0.90    修复更多警告；版本号升级以接近 1.0
# 0.61    由 Marc LeBlanc、Christopher Lloyd 导致的 bug 修复
# 版本更新记录，包括修复的问题和改进的内容
0.60    修复了作为 C++ 编译时的问题
0.59    修复了警告：合并了 Dave Moore 的 -Wall 修复
0.58    修复了 bug：zlib 未压缩模式下的 len/nlen 字节序问题
0.57    修复了 bug：jpg 中标记前最后一个哈夫曼符号大于9位但小于16位的问题
0.56    修复了 bug：zlib 未压缩模式下的 len 与 nlen 问题
0.55    修复了 bug：restart_interval 未初始化为 0 的问题
0.54    允许 'int *comp' 为 NULL
0.53    修复了 png 3->4 的 bug；加快了 png 解码速度
0.52    png 直接处理 req_comp=3,4；进行了一些清理；添加了 jpeg 注释
0.51    遵守了 req_comp 请求，1 组件的 jpeg 返回为 1 组件，对于 'test' 只检查类型，不检查是否支持此变体
0.50    (2006-11-19)
        第一个发布版本

/*
------------------------------------------------------------------------------
此软件可根据两种许可证选择使用，选择您喜欢的。
------------------------------------------------------------------------------
备选方案 A - MIT 许可证
# 版权声明，允许任何人免费获取并使用该软件，包括复制、修改、合并、发布、分发、转让和出售等操作
# 在使用该软件时，需包含以上版权声明和许可声明
# 该软件按原样提供，不提供任何形式的保证，包括但不限于适销性、特定用途适用性和非侵权性的保证
# 作者或版权持有人不对任何索赔、损害或其他责任负责，无论是合同诉讼、侵权行为还是其他情况引起的
# 与软件、使用或其他交易有关的索赔、损害或其他责任
# 另外提供的选择B - 公共领域（www.unlicense.org）
# 这是一个放入公共领域的自由和无限制的软件
# 任何人都可以自由复制、修改、发布、使用、编译、出售或分发该软件
# 该段代码是版权声明和软件许可协议，声明该软件可以以源代码形式或编译后的二进制形式，以任何目的，商业或非商业，以任何方式使用。
# 在承认版权法的司法管辖区，作者将该软件的所有版权利益无偿奉献给公共领域。我们做出这一奉献是为了造福广大公众，对我们的继承人和后继者造成损害。我们打算将这一奉献视为永久放弃对该软件在版权法下现有和未来的所有权利的公开行为。
# 该软件按原样提供，不提供任何形式的保证，包括但不限于适销性、特定用途的适用性和非侵权性的保证。在任何情况下，作者均不对任何索赔、损害或其他责任承担责任，无论是合同行为、侵权行为还是其他行为，因使用该软件或与该软件的使用或其他交易有关而产生、引起或与之有关。
```