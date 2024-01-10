# `nmap\libz\zlib.h`

```
/* zlib.h -- interface of the 'zlib' general purpose compression library
  version 1.2.13, October 13th, 2022

  Copyright (C) 1995-2022 Jean-loup Gailly and Mark Adler

  This software is provided 'as-is', without any express or implied
  warranty.  In no event will the authors be held liable for any damages
  arising from the use of this software.

  Permission is granted to anyone to use this software for any purpose,
  including commercial applications, and to alter it and redistribute it
  freely, subject to the following restrictions:

  1. The origin of this software must not be misrepresented; you must not
     claim that you wrote the original software. If you use this software
     in a product, an acknowledgment in the product documentation would be
     appreciated but is not required.
  2. Altered source versions must be plainly marked as such, and must not be
     misrepresented as being the original software.
  3. This notice may not be removed or altered from any source distribution.

  Jean-loup Gailly        Mark Adler
  jloup@gzip.org          madler@alumni.caltech.edu


  The data format used by the zlib library is described by RFCs (Request for
  Comments) 1950 to 1952 in the files http://tools.ietf.org/html/rfc1950
  (zlib format), rfc1951 (deflate format) and rfc1952 (gzip format).
*/

#ifndef ZLIB_H
#define ZLIB_H

#include "zconf.h"

#ifdef __cplusplus
extern "C" {
#endif

#define ZLIB_VERSION "1.2.13"
#define ZLIB_VERNUM 0x12d0
#define ZLIB_VER_MAJOR 1
#define ZLIB_VER_MINOR 2
#define ZLIB_VER_REVISION 13
#define ZLIB_VER_SUBREVISION 0

/*
    The 'zlib' compression library provides in-memory compression and
  decompression functions, including integrity checks of the uncompressed data.
  This version of the library supports only one compression method (deflation)
  but other algorithms will be added later and will have the same stream
  interface.
    # 如果缓冲区足够大，可以在单个步骤中进行压缩，也可以通过重复调用压缩函数来进行压缩。在后一种情况下，应用程序必须在每次调用之前提供更多的输入和/或消耗输出（提供更多的输出空间）。

    # 默认情况下，内存函数使用的压缩数据格式是 zlib 格式，这是一个包装在 RFC 1950 中记录的 zlib 包装器，包装在 RFC 1951 中记录的 deflate 流周围。

    # 该库还支持使用以 "gz" 开头的函数，以类似于 stdio 的接口读取和写入 gzip (.gz) 格式的文件。gzip 格式与 zlib 格式不同。gzip 是一个在 RFC 1952 中记录的 gzip 包装器，包装在 deflate 流周围。

    # 该库还可以选择在内存中读取和写入 gzip 和原始 deflate 流。

    # zlib 格式旨在在内存和通信通道中紧凑且快速使用。gzip 格式旨在文件系统上进行单个文件压缩，具有比 zlib 更大的标头以维护目录信息，并且使用与 zlib 不同且更慢的检查方法。

    # 该库不安装任何信号处理程序。解码器检查压缩数据的一致性，因此即使在输入损坏的情况下，该库也不应崩溃。
/*
    定义了两个函数指针类型，alloc_func 和 free_func，用于分配和释放内存
    OF((voidpf opaque, uInt items, uInt size)) 表示函数参数列表
*/
typedef voidpf (*alloc_func) OF((voidpf opaque, uInt items, uInt size));
typedef void   (*free_func)  OF((voidpf opaque, voidpf address));

/*
    定义了一个内部状态结构体，但没有给出具体的定义
*/
struct internal_state;

/*
    定义了一个 z_stream 结构体，用于处理压缩和解压缩的数据流
*/
typedef struct z_stream_s {
    z_const Bytef *next_in;     /* 下一个输入字节的指针 */
    uInt     avail_in;  /* next_in 处可用的字节数 */
    uLong    total_in;  /* 到目前为止已读取的输入字节数 */

    Bytef    *next_out; /* 下一个输出字节的指针 */
    uInt     avail_out; /* next_out 处剩余的可用空间 */
    uLong    total_out; /* 到目前为止已输出的字节数 */

    z_const char *msg;  /* 最后的错误消息，如果没有错误则为 NULL */
    struct internal_state FAR *state; /* 应用程序不可见的内部状态 */

    alloc_func zalloc;  /* 用于分配内部状态的函数指针 */
    free_func  zfree;   /* 用于释放内部状态的函数指针 */
    voidpf     opaque;  /* 传递给 zalloc 和 zfree 的私有数据对象 */

    int     data_type;  /* 数据类型的最佳猜测：二进制或文本（对于压缩），或解压缩状态（对于解压缩） */
    uLong   adler;      /* 未压缩数据的 Adler-32 或 CRC-32 值 */
    uLong   reserved;   /* 保留字段，未来使用 */
} z_stream;

/*
    定义了一个 z_streamp 类型的指针，指向 z_stream 结构体
*/
typedef z_stream FAR *z_streamp;

/*
    传递给和从 zlib 程序中传递的 gzip 头信息结构体，参考 RFC 1952 获取更多关于这些字段含义的细节
*/
typedef struct gz_header_s {
    int     text;       /* 如果压缩数据被认为是文本，则为 true */
    uLong   time;       /* 修改时间 */
    int     xflags;     /* 额外标志（写入 gzip 文件时不使用） */
    int     os;         /* 操作系统 */
    Bytef   *extra;     /* 指向额外字段的指针，如果没有则为 Z_NULL */
    uInt    extra_len;  /* 额外字段的长度（extra 不为 Z_NULL 时有效） */
    uInt    extra_max;  /* extra 的可用空间（仅在读取头部时有效） */
    Bytef   *name;      /* 指向以零结尾的文件名的指针，如果没有则为 Z_NULL */
    uInt    name_max;   /* 用于存储文件名的最大长度（仅在读取头部时使用） */
    Bytef   *comment;   /* 指向以零结尾的注释的指针，或者为 Z_NULL */
    uInt    comm_max;   /* 用于存储注释的最大长度（仅在读取头部时使用） */
    int     hcrc;       /* 如果存在或将存在头部 CRC，则为 true */
    int     done;       /* 在读取完 gzip 头部时为 true（在写入 gzip 文件时不使用） */
# 定义了一个结构体 gz_header，用于表示 gzip 文件头部信息
} gz_header;

# 定义了一个指向 gz_header 结构体的指针类型 gz_headerp
typedef gz_header FAR *gz_headerp;

# 下面是一段注释，说明了应用程序在使用 zlib 压缩库时需要注意的一些事项，包括更新输入输出缓冲区的指针和可用长度，初始化内存分配器等

# 定义了一些常量
                        /* constants */
#define Z_NO_FLUSH      0
#define Z_PARTIAL_FLUSH 1
#define Z_SYNC_FLUSH    2
#define Z_FULL_FLUSH    3
#define Z_FINISH        4
#define Z_BLOCK         5
#define Z_TREES         6
/* Allowed flush values; see deflate() and inflate() below for details */

#define Z_OK            0
#define Z_STREAM_END    1
#define Z_NEED_DICT     2
#define Z_ERRNO        (-1)
#define Z_STREAM_ERROR (-2)
#define Z_DATA_ERROR   (-3)
#define Z_MEM_ERROR    (-4)
#define Z_BUF_ERROR    (-5)
#define Z_VERSION_ERROR (-6)
/* Return codes for the compression/decompression functions. Negative values
 * are errors, positive values are used for special but normal events.
 */

#define Z_NO_COMPRESSION         0
#define Z_BEST_SPEED             1
#define Z_BEST_COMPRESSION       9
#define Z_DEFAULT_COMPRESSION  (-1)
/* compression levels */

#define Z_FILTERED            1
#define Z_HUFFMAN_ONLY        2
#define Z_RLE                 3
#define Z_FIXED               4
#define Z_DEFAULT_STRATEGY    0
/* compression strategy; see deflateInit2() below for details */

#define Z_BINARY   0
#define Z_TEXT     1
#define Z_ASCII    Z_TEXT   /* for compatibility with 1.2.2 and earlier */
#define Z_UNKNOWN  2
/* Possible values of the data_type field for deflate() */

#define Z_DEFLATED   8
/* The deflate compression method (the only one supported in this version) */

#define Z_NULL  0  /* for initializing zalloc, zfree, opaque */

#define zlib_version zlibVersion()
/* for compatibility with versions < 1.0.2 */


                        /* basic functions */

ZEXTERN const char * ZEXPORT zlibVersion OF((void));
/* The application can compare zlibVersion and ZLIB_VERSION for consistency.
   If the first character differs, the library code actually used is not
   compatible with the zlib.h header file used by the application.  This check
   is automatically made by deflateInit and inflateInit.
 */
# 初始化压缩流状态
ZEXTERN int ZEXPORT deflateInit OF((z_streamp strm, int level));

     初始化内部压缩流状态。在调用者之前必须初始化字段zalloc、zfree和opaque。如果zalloc和zfree设置为Z_NULL，则deflateInit会更新它们以使用默认分配函数。

     压缩级别必须是Z_DEFAULT_COMPRESSION，或者在0到9之间：1表示最快速度，9表示最佳压缩，0表示根本不压缩（输入数据仅以块方式复制）。Z_DEFAULT_COMPRESSION请求速度和压缩之间的默认折衷（当前等效于级别6）。

     如果成功，deflateInit返回Z_OK，如果内存不足则返回Z_MEM_ERROR，如果级别不是有效的压缩级别则返回Z_STREAM_ERROR，如果zlib库版本（zlib_version）与调用者假定的版本不兼容则返回Z_VERSION_ERROR。如果没有错误消息，则将msg设置为null。deflateInit不执行任何压缩：这将由deflate()执行。

ZEXTERN int ZEXPORT deflate OF((z_streamp strm, int flush));
/*
    deflate尽可能多地压缩数据，并在输入缓冲区变空或输出缓冲区变满时停止。除非强制刷新，否则可能会引入一些输出延迟（读取输入而不产生任何输出）。

    详细语义如下。deflate执行以下一个或两个操作：

  - 从next_in开始压缩更多输入，并相应地更新next_in和avail_in。如果无法处理所有输入（因为输出缓冲区中没有足够的空间），则更新next_in和avail_in，并且在下一次调用deflate()时将在此点恢复处理。

  - 从next_out开始生成更多输出，并相应地更新next_out和avail_out。如果参数flush非零，则会强制执行此操作。
    # 设置 flush 参数会影响压缩比，应仅在必要时设置。即使 flush 为零，也可能会提供一些输出。
    # 在调用 deflate() 之前，应用程序应确保至少有一种操作是可能的，可以提供更多输入和/或消耗更多输出，并相应地更新 avail_in 或 avail_out；在调用之前，avail_out 不应为零。应用程序可以在需要时消耗压缩输出，例如当输出缓冲区已满时（avail_out == 0），或在每次调用 deflate() 后。如果 deflate 返回 Z_OK 并且 avail_out 为零，则必须在在输出缓冲区中腾出空间后再次调用它，因为可能还有更多的输出等待。可以使用 deflatePending() 来确定是否存在更多输出。
    # 通常，flush 参数设置为 Z_NO_FLUSH，这允许 deflate 决定在产生输出之前累积多少数据，以最大化压缩。
    # 如果将参数 flush 设置为 Z_SYNC_FLUSH，则会将所有挂起的输出刷新到输出缓冲区，并且输出会对齐到字节边界，以便解压缩器可以获取到目前为止所有可用的输入数据。（特别是在调用之后，如果在调用之前提供了足够的输出空间，则 avail_in 为零。）刷新可能会降低某些压缩算法的压缩效果，因此应仅在必要时使用。这将完成当前的 deflate 块，并跟随一个空的存储块，其中包括三位加上填充位到下一个字节，然后是四个字节（00 00 ff ff）。
    # 如果 flush 被设置为 Z_PARTIAL_FLUSH，则所有挂起的输出被刷新到输出缓冲区，但输出不会对齐到字节边界。
    # 到目前为止的所有输入数据将对解压缩器可用，就像对于 Z_SYNC_FLUSH 一样。
    # 这完成了当前的压缩块，并跟随一个长度为 10 位的空的固定代码块。这确保输出足够的字节，以便解压缩器在空的固定代码块之前完成块。
    
    # 如果 flush 被设置为 Z_BLOCK，则完成并发射一个压缩块，就像对于 Z_SYNC_FLUSH 一样，但输出不会对齐到字节边界，
    # 并且当前块的最多七位被保留，以便在下一个压缩块完成后作为下一个字节写入。
    # 在这种情况下，解压缩器可能在这一点上没有提供足够的位来完成到目前为止提供给压缩器的数据的解压缩。
    # 它可能需要等待下一个块的发射。这是为了需要控制发射压缩块的高级应用程序。
    
    # 如果 flush 被设置为 Z_FULL_FLUSH，则所有输出都像 Z_SYNC_FLUSH 一样被刷新，并且压缩状态被重置，
    # 以便从这一点重新开始解压缩，如果先前的压缩数据已经损坏或者需要随机访问。过于频繁地使用 Z_FULL_FLUSH 可能严重降低压缩效果。
    
    # 如果 deflate 返回时 avail_out == 0，则必须再次使用相同的 flush 参数值和更多的输出空间（更新的 avail_out）调用此函数，
    # 直到刷新完成（deflate 返回时 avail_out 非零）。在 Z_FULL_FLUSH 或 Z_SYNC_FLUSH 的情况下，确保 avail_out 大于六，以避免由于返回时 avail_out == 0 而重复刷新标记。
    # 如果参数 flush 设置为 Z_FINISH，则处理挂起的输入，刷新挂起的输出，并且如果有足够的输出空间，则 deflate 返回 Z_STREAM_END。
    # 如果 deflate 返回 Z_OK 或 Z_BUF_ERROR，则必须再次调用此函数，使用 Z_FINISH 和更多的输出空间（更新 avail_out），但不再有输入数据，直到它返回 Z_STREAM_END 或错误。
    # 在 deflate 返回 Z_STREAM_END 后，流上唯一可能的操作是 deflateReset 或 deflateEnd。
    
    # 如果所有压缩都要在单个步骤中完成，则可以在 deflateInit 后的第一个 deflate 调用中使用 Z_FINISH。为了在一次调用中完成，avail_out 必须至少是 deflateBound 返回的值（见下文）。然后，deflate 保证返回 Z_STREAM_END。如果提供的输出空间不足，则 deflate 不会返回 Z_STREAM_END，并且必须按上述描述再次调用。
    
    # deflate() 将 strm->adler 设置为迄今为止读取的所有输入的 Adler-32 校验和（即 total_in 字节）。如果正在生成 gzip 流，则 strm->adler 将是迄今为止读取的输入的 CRC-32 校验和。（请参见下文的 deflateInit2。）
    
    # 如果 deflate() 能够对输入数据类型（Z_BINARY 或 Z_TEXT）做出良好的猜测，则可能会更新 strm->data_type。如果不确定，则将数据视为二进制。此字段仅用于信息目的，不以任何方式影响压缩算法。
    # deflate()函数返回值说明：
    # - 如果已经有一些进展（处理了更多输入或产生了更多输出），则返回Z_OK
    # - 如果所有输入都已经被消耗并且所有输出都已经产生（仅当flush设置为Z_FINISH时），则返回Z_STREAM_END
    # - 如果流状态不一致（例如如果next_in或next_out为Z_NULL，或者状态被应用程序无意中覆盖），则返回Z_STREAM_ERROR
    # - 如果没有进展可能（例如avail_in或avail_out为零），则返回Z_BUF_ERROR
    # 注意，Z_BUF_ERROR不是致命的，可以再次调用deflate()函数，提供更多输入和更多输出空间以继续压缩
# 终止压缩流，释放所有动态分配的数据结构
ZEXTERN int ZEXPORT deflateEnd OF((z_streamp strm));
'''
     所有为该流动态分配的数据结构都将被释放。
   此函数丢弃任何未处理的输入，并且不刷新任何未决的输出。

     如果成功，deflateEnd返回Z_OK，如果流状态不一致，则返回Z_STREAM_ERROR，
   如果流过早释放（一些输入或输出被丢弃），则返回Z_DATA_ERROR。在错误情况下，msg
   可能被设置，但然后指向一个静态字符串（不得被释放）。

'''

'''
ZEXTERN int ZEXPORT inflateInit OF((z_streamp strm));

     初始化内部流状态以进行解压缩。字段
   next_in，avail_in，zalloc，zfree和opaque必须由调用者在之前初始化。
   在当前版本的inflate中，提供的输入不会被读取或消耗。
   滑动窗口的分配将被推迟到第一次调用inflate（如果解压缩在第一次调用时未完成）。
   如果zalloc和zfree设置为Z_NULL，则inflateInit将更新它们以使用默认分配函数。

     如果成功，inflateInit返回Z_OK，如果内存不足则返回Z_MEM_ERROR，
   如果zlib库版本与调用者假定的版本不兼容，则返回Z_VERSION_ERROR，
   或者如果参数无效，则返回Z_STREAM_ERROR，例如结构的空指针。如果没有错误消息，msg设置为null。
   inflateInit不执行任何解压缩。实际的解压缩将由inflate()完成。
   因此，next_in和avail_in，next_out和avail_out未使用且未更改。
   inflateInit()的当前实现不处理任何头信息--这将推迟到调用inflate()。
'''

ZEXTERN int ZEXPORT inflate OF((z_streamp strm, int flush));
'''
    # inflate函数尽可能解压缩更多的数据，并在输入缓冲区变空或输出缓冲区变满时停止。除非被强制刷新，否则可能会引入一些输出延迟（读取输入但不产生任何输出）。
    
    # inflate函数的详细语义如下：
    # - 从next_in开始解压更多的输入，并相应地更新next_in和avail_in。如果无法处理所有输入（因为输出缓冲区中没有足够的空间），则相应地更新next_in和avail_in，并且在下一次调用inflate()时将在此点恢复处理。
    # - 从next_out开始生成更多的输出，并相应地更新next_out和avail_out。inflate()提供尽可能多的输出，直到没有更多的输入数据或输出缓冲区中没有更多空间（有关flush参数的信息，请参见下文）。
    
    # 在调用inflate()之前，应用程序应确保至少有一种操作是可能的，即提供更多的输入和/或消耗更多的输出，并相应地更新next_*和avail_*的值。如果调用inflate()的调用者没有同时提供可用的输入和可用的输出空间，可能不会取得任何进展。应用程序可以在希望时消耗未压缩的输出，例如当输出缓冲区已满（avail_out == 0）时，或在每次调用inflate()后。如果inflate返回Z_OK并且avail_out为零，则必须在输出缓冲区中腾出空间后再次调用它，因为可能有更多的输出待处理。
    # 设置inflate()的flush参数可以是Z_NO_FLUSH, Z_SYNC_FLUSH, Z_FINISH, Z_BLOCK或Z_TREES。
    # Z_SYNC_FLUSH请求inflate()尽可能多地将输出刷新到输出缓冲区。
    # Z_BLOCK请求inflate()在到达下一个deflate块边界时停止。
    # 在解码zlib或gzip格式时，这将导致inflate()在头部之后立即返回，而在第一个块之前返回。
    # 在进行原始解压缩时，inflate()将继续处理第一个块，并且在到达该块的末尾或数据用完时返回。
    
    # Z_BLOCK选项有助于附加或组合deflate流。
    # 为了帮助实现这一点，在返回时，inflate()总是将strm->data_type设置为从strm->next_in中取出的最后一个字节中未使用的位数，如果inflate()当前正在解码deflate流中的最后一个块，则再加上64，如果inflate()在解码结束块代码或解码完整头部后立即返回，则再加上128。
    # 直到所有该块的未压缩数据都被写入到strm->next_out后，才会指示结束块。
    # 未使用的位数通常可能大于七，除非data_type的第7位被设置，此时未使用的位数将小于八。
    # 每次inflate()返回时都会根据这里的说明设置data_type，因此可以用于确定当前消耗的输入位数。
    
    # Z_TREES选项的行为与Z_BLOCK相同，但它还会在到达每个deflate块头部的末尾之前返回，而不解码该块中的任何实际数据。
    # 这允许调用者确定以后在deflate块内进行随机访问时的deflate块头部的长度。
    # 当inflate()在到达deflate块头部的末尾后立即返回时，将256添加到strm->data_type的值。
    # inflate()函数可以解压并检查zlib-wrapped或gzip-wrapped的deflate数据。
    # 如果在使用inflateInit2()初始化时请求，头部类型会被自动检测。
    # 除非使用inflateGetHeader()，否则不会保留gzip头部中包含的任何信息。
    # 在处理gzip-wrapped的deflate数据时，strm->adler32会被设置为迄今为止产生的输出的CRC-32。
    # CRC-32会与gzip尾部进行检查，未压缩长度对2^32取模也会进行检查。
    
    # 如果取得了一些进展（处理了更多输入或产生了更多输出），inflate()会返回Z_OK；
    # 如果已经到达压缩数据的末尾并且已经产生了所有未压缩的输出，inflate()会返回Z_STREAM_END；
    # 如果此时需要预设字典，inflate()会返回Z_NEED_DICT；
    # 如果输入数据损坏（输入流不符合zlib格式或检查值不正确），inflate()会返回Z_DATA_ERROR（此时strm->msg指向一个具有更具体错误信息的字符串）；
    # 如果流结构不一致（例如next_in或next_out为Z_NULL，或者状态被应用程序无意中覆盖），inflate()会返回Z_STREAM_ERROR；
    # 如果内存不足，inflate()会返回Z_MEM_ERROR；
    # 如果在使用Z_FINISH时无法取得进展或输出缓冲区空间不足，inflate()会返回Z_BUF_ERROR。
    # 注意，Z_BUF_ERROR不是致命的，可以再次调用inflate()并提供更多输入和更多输出空间以继续解压缩。
    # 如果返回Z_DATA_ERROR，则应用程序可以调用inflateSync()来寻找一个良好的压缩块，以尝试对数据进行部分恢复。
/*
    终止解压缩流，释放所有动态分配的数据结构。该函数丢弃任何未处理的输入，并不刷新任何未决的输出。
    如果成功，inflateEnd返回Z_OK；如果流状态不一致，则返回Z_STREAM_ERROR。
*/
ZEXTERN int ZEXPORT inflateEnd OF((z_streamp strm));

                        /* 高级函数 */

/*
    以下函数仅在一些特殊应用程序中需要。
*/

/*
*/

/*
    设置deflate的压缩字典。strm是指向压缩流的指针，dictionary是指向要设置的字典的指针，dictLength是字典的长度。
*/
ZEXTERN int ZEXPORT deflateSetDictionary OF((z_streamp strm,
                                             const Bytef *dictionary,
                                             uInt  dictLength));

/*
    获取deflate维护的滑动字典。strm是指向压缩流的指针，dictionary是指向要获取的字典的指针，dictLength是字典的长度。
    返回滑动字典的长度，并将相应数量的字节复制到dictionary中。如果dictionary为Z_NULL，则只返回字典长度，不复制任何内容。
    如果dictLength为Z_NULL，则不设置字典长度。
    如果成功，deflateGetDictionary返回Z_OK；如果流状态不一致，则返回Z_STREAM_ERROR。
*/
ZEXTERN int ZEXPORT deflateGetDictionary OF((z_streamp strm,
                                             Bytef *dictionary,
                                             uInt  *dictLength));
# 定义函数deflateCopy，用于将目标流设置为源流的完全副本
# 这个函数在尝试多种压缩策略时可能会很有用，例如在使用过滤器对输入数据进行多种预处理时
# 将被丢弃的流应该通过调用deflateEnd来释放
# 注意，deflateCopy会复制内部的压缩状态，这可能会很大，所以这种策略会很慢并且可能会消耗大量内存
# 如果成功，deflateCopy返回Z_OK；如果内存不足，则返回Z_MEM_ERROR；如果源流状态不一致（例如zalloc为Z_NULL），则返回Z_STREAM_ERROR
# msg在源流和目标流中都不会改变
ZEXTERN int ZEXPORT deflateCopy OF((z_streamp dest,
                                    z_streamp source));

# 定义函数deflateReset，相当于deflateEnd后跟deflateInit，但不释放和重新分配内部的压缩状态
# 流将保持压缩级别和可能已设置的任何其他属性不变
# 如果成功，deflateReset返回Z_OK；如果源流状态不一致（例如zalloc或state为Z_NULL），则返回Z_STREAM_ERROR
ZEXTERN int ZEXPORT deflateReset OF((z_streamp strm));

# 定义函数deflateParams，用于设置压缩级别和策略
ZEXTERN int ZEXPORT deflateParams OF((z_streamp strm,
                                      int level,
                                      int strategy));

# 定义函数deflateTune，用于调整压缩参数
ZEXTERN int ZEXPORT deflateTune OF((z_streamp strm,
                                    int good_length,
                                    int max_lazy,
                                    int nice_length,
                                    int max_chain));
/*
     调整deflate的内部压缩参数。这应该只由了解zlib的deflate用于搜索最佳匹配字符串的算法的人使用，即使是最狂热的优化器也只能为其特定输入数据挤出最后的压缩位。阅读deflate.c源代码以了解max_lazy、good_length、nice_length和max_chain参数的含义。

     deflateTune()可以在deflateInit()或deflateInit2()之后调用，并在成功时返回Z_OK，或在无效的deflate流时返回Z_STREAM_ERROR。
 */

ZEXTERN uLong ZEXPORT deflateBound OF((z_streamp strm,
                                       uLong sourceLen));
/*
     deflateBound()返回对sourceLen字节进行压缩后的压缩大小的上限。必须在deflateInit()或deflateInit2()之后调用，并在使用deflateSetHeader()之后调用。这将用于在单次传递中为压缩分配输出缓冲区，因此在调用deflate()之前会调用它。如果第一个deflate()调用提供了sourceLen输入字节，一个按照deflateBound()返回的大小分配的输出缓冲区，以及flush值Z_FINISH，那么deflate()保证返回Z_STREAM_END。请注意，如果使用的flush选项不是Z_FINISH或Z_NO_FLUSH，则压缩大小可能大于deflateBound()返回的值。
*/

ZEXTERN int ZEXPORT deflatePending OF((z_streamp strm,
                                       unsigned *pending,
                                       int *bits));
/*
     deflatePending()返回已生成但尚未提供的输出字节数和位数。未提供的字节数是由于可用输出空间已被消耗。未提供的位数在0和7之间，它们等待更多位加入以填满一个完整的字节。如果pending或bits为Z_NULL，则这些值未设置。

     如果成功，deflatePending返回Z_OK，如果源流状态不一致，则返回Z_STREAM_ERROR。
 */

ZEXTERN int ZEXPORT deflatePrime OF((z_streamp strm,
                                     int bits,
                                     int value));
/*
     deflatePrime()在deflate输出流中插入位。其意图是在追加到先前的deflate流时，使用此函数从上一个deflate流中剩余的位开始deflate输出。因此，此函数只能用于原始deflate，并且必须在deflateInit2()或deflateReset()之后的第一个deflate()调用之前使用。bits必须小于或等于16，并且value的最低有效位数将被插入输出中。

     如果成功，deflatePrime返回Z_OK，如果内部缓冲区中没有足够的空间插入位，则返回Z_BUF_ERROR，如果源流状态不一致，则返回Z_STREAM_ERROR。
*/

ZEXTERN int ZEXPORT deflateSetHeader OF((z_streamp strm,
                                         gz_headerp head));
/*
     deflateSetHeader()函数为当请求一个gzip流时提供gzip头信息。deflateSetHeader()可以在deflateInit2()或deflateReset()之后，以及在第一次调用deflate()之前调用。提供的gz_header结构中的文本、时间、操作系统、额外字段、名称和注释信息将被写入gzip头部（xflag被忽略——额外标志根据压缩级别设置）。调用者必须确保，如果不是Z_NULL，名称和注释以零字节结尾，如果extra不是Z_NULL，则在那里有extra_len字节可用。如果hcrc为true，则包括gzip头部crc。请注意，当前版本的命令行版本的gzip（直到1.3.x版本）不支持头部crc，并且将报告它是一个“多部分gzip文件”并放弃。

     如果不使用deflateSetHeader，则默认的gzip头部具有文本为false，时间设置为零，操作系统设置为255，没有额外字段、名称或注释字段。通过deflateReset()将gzip头部返回到默认状态。

     如果成功，deflateSetHeader返回Z_OK，如果源流状态不一致，则返回Z_STREAM_ERROR。
*/

/*
*/

ZEXTERN int ZEXPORT inflateSetDictionary OF((z_streamp strm,
                                             const Bytef *dictionary,
                                             uInt  dictLength));
/*
     从给定的未压缩字节序列初始化解压缩字典。如果inflate返回Z_NEED_DICT，则必须立即调用此函数。可以从inflate返回的Adler-32值确定压缩器选择的字典。压缩器和解压缩器必须使用完全相同的字典（参见deflateSetDictionary）。对于原始inflate，可以随时调用此函数来设置字典。如果提供的字典比窗口小，并且窗口中已经有数据，则提供的字典将修正已有的数据。应用程序必须确保提供了用于压缩的字典。

     如果成功，inflateSetDictionary返回Z_OK，如果参数无效（例如，字典为Z_NULL）或流状态不一致，则返回Z_STREAM_ERROR，如果给定的字典与预期的字典不匹配（不正确的Adler-32值），则返回Z_DATA_ERROR。inflateSetDictionary不执行任何解压缩：这将由后续的inflate()调用完成。
*/
ZEXTERN int ZEXPORT inflateGetDictionary OF((z_streamp strm,
                                             Bytef *dictionary,
                                             uInt  *dictLength));
/*
     返回由inflate维护的滑动字典。dictLength设置为字典中的字节数，并将这么多字节复制到字典中。字典必须有足够的空间，其中32768字节总是足够的。如果inflateGetDictionary()使用dictionary等于Z_NULL调用，则仅返回字典长度，并且不复制任何内容。同样，如果dictLength等于Z_NULL，则不设置。

     如果成功，inflateGetDictionary返回Z_OK，如果流状态不一致，则返回Z_STREAM_ERROR。
*/
ZEXTERN int ZEXPORT inflateSync OF((z_streamp strm));
/*
     跳过无效的压缩数据，直到找到可能的完全刷新点（参见上文关于带有 Z_FULL_FLUSH 的 deflate 描述），或者直到跳过所有可用的输入。不提供任何输出。

     inflateSync 在压缩数据中搜索 00 00 FF FF 模式。所有完全刷新点都具有此模式，但并非所有此模式的出现都是完全刷新点。

     如果找到可能的完全刷新点，则 inflateSync 返回 Z_OK；如果没有提供更多输入，则返回 Z_BUF_ERROR；如果没有找到刷新点，则返回 Z_DATA_ERROR；如果流结构不一致，则返回 Z_STREAM_ERROR。在成功的情况下，应用程序可以保存 total_in 的当前值，该值指示找到有效压缩数据的位置。在错误的情况下，应用程序可以重复调用 inflateSync，每次提供更多的输入，直到成功或输入数据结束。
*/

ZEXTERN int ZEXPORT inflateCopy OF((z_streamp dest,
                                    z_streamp source));
/*
     将目标流设置为源流的完全副本。

     当随机访问大流时，此函数可能很有用。通过流的第一次遍历，可以定期记录 inflate 状态，允许在随机访问流时在这些点重新启动 inflate。

     如果成功，inflateCopy 返回 Z_OK；如果内存不足，则返回 Z_MEM_ERROR；如果源流状态不一致（例如 zalloc 为 Z_NULL），则返回 Z_STREAM_ERROR。msg 在源和目标中都不会改变。
*/

ZEXTERN int ZEXPORT inflateReset OF((z_streamp strm));
/*
     这个函数相当于inflateEnd后跟inflateInit，但不会释放和重新分配内部解压缩状态。流将保留可能由inflateInit2设置的属性。

     如果成功，inflateReset返回Z_OK，如果源流状态不一致（例如zalloc或state为Z_NULL），则返回Z_STREAM_ERROR。
*/

ZEXTERN int ZEXPORT inflateReset2 OF((z_streamp strm,
                                      int windowBits));
/*
     这个函数与inflateReset相同，但它还允许更改wrap和window大小的请求。windowBits参数的解释与inflateInit2相同。如果窗口大小被更改，那么为窗口分配的内存将被释放，并且如果需要，窗口将由inflate()重新分配。

     如果成功，inflateReset2返回Z_OK，如果源流状态不一致（例如zalloc或state为Z_NULL），或者windowBits参数无效，则返回Z_STREAM_ERROR。
*/

ZEXTERN int ZEXPORT inflatePrime OF((z_streamp strm,
                                     int bits,
                                     int value));
# 这个函数在inflate输入流中插入位。意图是在字节的中间位置开始解压缩。提供的位将在从next_in使用任何字节之前使用。这个函数应该只与原始解压缩一起使用，并且应该在inflateInit2()或inflateReset()之后的第一个inflate()调用之前使用。bits必须小于或等于16，并且value的最低有效位将被插入输入中。

# 如果bits为负数，则输入流位缓冲区被清空。然后可以再次调用inflatePrime()将位放入缓冲区。这用于在向inflate提供代码之前清除剩余的位。

# 如果成功，inflatePrime返回Z_OK，如果源流状态不一致，则返回Z_STREAM_ERROR。
ZEXTERN long ZEXPORT inflateMark OF((z_streamp strm));
/*
     此函数返回两个值，一个在返回值的低16位中，另一个在将返回值向下移动16位后的剩余高位中获得。如果上半部分的值为-1，下半部分的值为零，则inflate()当前正在解码块外的信息。如果上半部分的值为-1，下半部分的值为非零，则inflate()处于存储块的中间阶段，下半部分的值等于剩余要复制的输入字节数。如果上半部分的值不是-1，则它是当前正在处理的代码（文字或长度/距离对）在输入中距离当前位位置的位数。在这种情况下，下半部分的值是已经为该代码发出的字节数。

     如果inflate()正在等待更多的输入来完成代码的解码，或者已经完成解码但正在等待更多的输出空间来写入文字或匹配数据，则正在处理一个代码。

     inflateMark()用于标记输入数据的随机访问位置，这可能是在位位置，以及记录代码的输出可能跨越随机访问块的情况。可以从avail_in和data_type中确定输入流中的当前位置，如inflate的Z_BLOCK刷新参数的描述中所述。

     inflateMark返回上面提到的值，如果提供的源流状态不一致，则返回-65536。
*/

ZEXTERN int ZEXPORT inflateGetHeader OF((z_streamp strm,
                                         gz_headerp head));
*/
# 初始化内部流状态，用于使用inflateBack()进行解压缩调用
ZEXTERN int ZEXPORT inflateBackInit OF((z_streamp strm, int windowBits,
                                        unsigned char FAR *window));
/*
   初始化内部流状态，用于使用inflateBack()进行解压缩调用。
   在调用之前，strm中的zalloc、zfree和opaque字段必须初始化。
   如果zalloc和zfree为Z_NULL，则使用默认的库派生内存分配例程。
   windowBits是窗口大小的以2为底的对数，范围在8到15之间。
   window是调用者提供的具有该大小的缓冲区。
   除了在确保使用小窗口大小的deflate的特殊应用程序中，windowBits必须为15，
   并且必须提供32K字节窗口以便解压缩一般的deflate流。

   请参阅inflateBack()以了解这些例程的用法。

   inflateBackInit在成功时返回Z_OK，在参数无效时返回Z_STREAM_ERROR，
   在无法分配内部状态时返回Z_MEM_ERROR，
   或者在库的版本与头文件的版本不匹配时返回Z_VERSION_ERROR。
*/

typedef unsigned (*in_func) OF((void FAR *,
                                z_const unsigned char FAR * FAR *));
typedef int (*out_func) OF((void FAR *, unsigned char FAR *, unsigned));

ZEXTERN int ZEXPORT inflateBack OF((z_streamp strm,
                                    in_func in, void FAR *in_desc,
                                    out_func out, void FAR *out_desc));
/*
   使用inflateBackInit()分配的所有内存都将被释放。
   inflateBackEnd()在成功时返回Z_OK，如果流状态不一致则返回Z_STREAM_ERROR。
*/

ZEXTERN int ZEXPORT inflateBackEnd OF((z_streamp strm));
/*
   返回指示编译时选项的标志。
*/
ZEXTERN uLong ZEXPORT zlibCompileFlags OF((void));
    # 定义不同类型的大小，每个类型占两位，00表示16位，01表示32位，10表示64位，11表示其他
    # 1.0: 表示 uInt 的大小
    # 3.2: 表示 uLong 的大小
    # 5.4: 表示 voidpf（指针）的大小
    # 7.6: 表示 z_off_t 的大小
    
    # 编译器、汇编器和调试选项：
    # 8: ZLIB_DEBUG
    # 9: ASMV 或 ASMINF -- 使用 ASM 代码
    # 10: ZLIB_WINAPI -- 导出函数使用 WINAPI 调用约定
    # 11: 0（保留）
    
    # 一次性表构建（代码更小，但如果为 true 则不是线程安全的）：
    # 12: BUILDFIXED -- 需要时构建静态块解码表
    # 13: DYNAMIC_CRC_TABLE -- 需要时构建 CRC 计算表
    # 14,15: 0（保留）
    
    # 库内容（指示缺少的功能）：
    # 16: NO_GZCOMPRESS -- gz* 函数无法压缩（以避免在不需要时链接 deflate 代码）
    # 17: NO_GZIP -- deflate 无法写入 gzip 流，inflate 无法检测和解码 gzip 流（以避免链接 crc 代码）
    # 18-19: 0（保留）
    
    # 操作变体（库功能的变化）：
    # 20: PKZIP_BUG_WORKAROUND -- 稍微宽松的 inflate
    # 21: FASTEST -- 只有一个最低压缩级别的 deflate 算法
    # 22,23: 0（保留）
    
    # gzprintf 使用的 sprintf 变体（零是最佳选择）：
    # 24: 0 = vs*，1 = s* -- 1 表示格式后最多有 20 个参数
    # 25: 0 = *nprintf，1 = *printf -- 1 表示 gzprintf() 不安全！
    # 26: 0 = 返回值，1 = void -- 1 表示返回的字符串长度是推断的
    
    # 剩余部分：
    # 27-31: 0（保留）
#ifndef Z_SOLO

                        /* utility functions */

/*
     The following utility functions are implemented on top of the basic
   stream-oriented functions.  To simplify the interface, some default options
   are assumed (compression level and memory usage, standard memory allocation
   functions).  The source code of these utility functions can be modified if
   you need special options.
*/

ZEXTERN int ZEXPORT compress OF((Bytef *dest,   uLongf *destLen,
                                 const Bytef *source, uLong sourceLen));
/*
     将源缓冲区压缩到目标缓冲区中。sourceLen 是源缓冲区的字节长度。在进入时，destLen 是目标缓冲区的总大小，必须至少是 compressBound(sourceLen) 返回的值。退出时，destLen 是压缩数据的实际大小。compress() 等同于使用 Z_DEFAULT_COMPRESSION 级别的 compress2()。

     如果成功，compress 返回 Z_OK，如果内存不足则返回 Z_MEM_ERROR，如果输出缓冲区空间不足则返回 Z_BUF_ERROR。
*/

ZEXTERN int ZEXPORT compress2 OF((Bytef *dest,   uLongf *destLen,
                                  const Bytef *source, uLong sourceLen,
                                  int level));
/*
     将源缓冲区压缩到目标缓冲区中。level 参数的含义与 deflateInit 中的相同。sourceLen 是源缓冲区的字节长度。在进入时，destLen 是目标缓冲区的总大小，必须至少是 compressBound(sourceLen) 返回的值。退出时，destLen 是压缩数据的实际大小。

     如果成功，compress2 返回 Z_OK，如果内存不足则返回 Z_MEM_ERROR，如果输出缓冲区空间不足则返回 Z_BUF_ERROR，如果级别参数无效则返回 Z_STREAM_ERROR。
*/

ZEXTERN uLong ZEXPORT compressBound OF((uLong sourceLen));
/*
     compressBound()函数返回对sourceLen字节进行compress()或compress2()压缩后的压缩大小的上限。在调用compress()或compress2()之前，可以使用它来分配目标缓冲区。
*/

ZEXTERN int ZEXPORT uncompress OF((Bytef *dest,   uLongf *destLen,
                                   const Bytef *source, uLong sourceLen));
/*
     将源缓冲区解压缩到目标缓冲区中。sourceLen是源缓冲区的字节长度。在进入时，destLen是目标缓冲区的总大小，必须足够大以容纳整个未压缩的数据。（未压缩数据的大小必须由压缩器先前保存，并通过此压缩库范围之外的某种机制传输给解压缩器。）退出时，destLen是未压缩数据的实际大小。

     如果成功，uncompress返回Z_OK，如果内存不足则返回Z_MEM_ERROR，如果输出缓冲区空间不足则返回Z_BUF_ERROR，如果输入数据损坏或不完整则返回Z_DATA_ERROR。在空间不足的情况下，uncompress()将填充输出缓冲区，直到那一点为止。
*/

ZEXTERN int ZEXPORT uncompress2 OF((Bytef *dest,   uLongf *destLen,
                                    const Bytef *source, uLong *sourceLen));
/*
     与uncompress相同，不同之处在于sourceLen是一个指针，其中source的长度是*sourceLen。返回时，*sourceLen是消耗的源字节数。
*/

                        /* gzip文件访问函数 */

/*
     该库支持使用类似stdio的接口读取和写入gzip（.gz）格式的文件，使用以"gz"开头的函数。gzip格式与zlib格式不同。gzip是一个gzip包装器，文档化在RFC 1952中，包装在一个deflate流周围。
*/
/* 定义了一个半透明的 gzip 文件描述符结构体指针 */
typedef struct gzFile_s *gzFile;    /* semi-opaque gzip file descriptor */

/*
*/

/* 
   用文件描述符 fd 关联一个 gzFile。文件描述符可以通过 open、dup、creat、pipe 或 fileno（如果文件之前已经用 fopen 打开）等调用获得。
   mode 参数与 gzopen 中的一样。

   对返回的 gzFile 进行下一次 gzclose 调用也会关闭文件描述符 fd，就像 fclose(fdopen(fd, mode)) 关闭文件描述符 fd 一样。
   如果要保持 fd 开放，使用 fd = dup(fd_keep); gz = gzdopen(fd, mode);。复制的描述符应该保存以避免泄漏，因为 gzdopen 在失败时不会关闭 fd。
   如果使用 fileno() 从 FILE * 获取文件描述符，则必须使用 dup() 来避免双重关闭文件描述符。
   gzclose() 和 fclose() 都会关闭关联的文件描述符，因此它们需要有不同的文件描述符。

   如果内存不足以分配 gzFile 状态，或者指定了无效的模式（未提供 'r'、'w' 或 'a'，或者提供了 '+'），或者 fd 为 -1，则 gzdopen 返回 NULL。
   文件描述符直到下一次 gz* 读取、写入、寻址或关闭操作才会被使用，因此 gzdopen 不会检测 fd 是否无效（除非 fd 为 -1）。
*/
ZEXTERN gzFile ZEXPORT gzdopen OF((int fd, const char *mode));

ZEXTERN int ZEXPORT gzbuffer OF((gzFile file, unsigned size));
/*
     设置此库函数在文件中使用的内部缓冲区大小为size。默认缓冲区大小为8192字节。此函数必须在gzopen()或gzdopen()之后调用，并在读取或写入文件的任何其他调用之前调用。缓冲区内存分配总是延迟到第一次读取或写入。分配三倍大小的缓冲区空间。例如，更大的缓冲区大小，如64K或128K字节，将显着增加解压缩（读取）的速度。

     新的缓冲区大小还影响gzprintf()的最大长度。

     gzbuffer()成功返回0，失败返回-1，例如调用太晚。
*/

ZEXTERN int ZEXPORT gzsetparams OF((gzFile file, int level, int strategy));
/*
     动态更新文件的压缩级别和策略。有关这些参数的含义，请参阅deflateInit2的描述。在应用参数更改之前，先刷新先前提供的数据。

     gzsetparams成功返回Z_OK，如果文件未打开以进行写入，则返回Z_STREAM_ERROR，如果写入刷新的数据时出现错误，则返回Z_ERRNO，如果存在内存分配错误，则返回Z_MEM_ERROR。
*/

ZEXTERN int ZEXPORT gzread OF((gzFile file, voidp buf, unsigned len));
# 从文件中读取并解压最多 len 个未压缩字节到缓冲区中
# 如果输入文件不是 gzip 格式，则 gzread 直接从文件中将给定数量的字节复制到缓冲区中

# 在输入中达到 gzip 流的末尾后，gzread 将继续读取，寻找另一个 gzip 流
# 输入文件中可以串联任意数量的 gzip 流，并且都将被 gzread() 解压缩
# 如果在 gzip 流之后遇到了其他内容，那么剩余的尾部垃圾将被忽略（并且不会返回错误）

# gzread 可用于读取正在同时写入的 gzip 文件
# 在达到输入的末尾时，gzread 将返回可用的数据
# 如果 gzerror 返回的错误代码是 Z_OK 或 Z_BUF_ERROR，则可以使用 gzclearerr 来清除文件结束标志，以便再次尝试 gzread
# Z_OK 表示上次 gzread 完成了一个 gzip 流
# Z_BUF_ERROR 表示输入文件在 gzip 流的中间结束
# 注意，gzread 在遇到不完整的 gzip 流时不会返回 -1。此错误被推迟到 gzclose()，如果最后一个 gzread 在 gzip 流的中间结束，则会返回 Z_BUF_ERROR
# 或者，可以在 gzclose 之前使用 gzerror 来检测这种情况

# gzread 返回实际读取的未压缩字节数，对于文件末尾小于 len 的情况返回小于 len 的值，对于错误返回 -1
# 如果 len 太大而无法适应 int，则不会读取任何内容，返回 -1，并将错误状态设置为 Z_STREAM_ERROR
/*
     从文件中读取并解压缩最多 nitems 个大小为 size 的项目到 buf 中，否则操作方式与 gzread() 相同。这复制了 stdio 的 fread() 接口，使用 size_t 请求和返回类型。如果库定义了 size_t，则 z_size_t 与 size_t 相同。如果没有定义，则 z_size_t 是一个无符号整数类型，可以包含指针。

     gzfread() 返回以 size 大小读取的完整项目数，如果到达文件末尾并且无法读取完整项目，则返回零，或者如果出现错误也返回零。如果返回零，则必须查询 gzerror() 以确定是否出现错误。如果 size 和 nitems 的乘积溢出，即乘积不适合 z_size_t，那么将不会读取任何内容，返回零，并将错误状态设置为 Z_STREAM_ERROR。

     如果到达文件末尾并且仅在末尾有部分项目可用，即剩余的未压缩数据长度不是 size 的倍数，则最终部分项目仍然被读入 buf，并设置文件结束标志。不提供已读取的部分项目的长度，但可以从 gztell() 的结果推断出来。这种行为与常见库中的 fread() 实现的行为相同，但当 size 不为 1 时，它阻止直接使用 gzfread() 读取并发写入的文件，重置并在文件末尾重试。

*/

ZEXTERN int ZEXPORT gzwrite OF((gzFile file, voidpc buf, unsigned len));
/*
     将 buf 中的 len 个未压缩字节压缩并写入文件。gzwrite 返回已写入的未压缩字节数，如果出现错误则返回 0。
*/

ZEXTERN z_size_t ZEXPORT gzfwrite OF((voidpc buf, z_size_t size,
                                      z_size_t nitems, gzFile file));
/*
     从缓冲区 buf 中压缩并写入 nitems 个大小为 size 的项目到文件中，与 stdio 的 fwrite() 接口相同，使用 size_t 请求和返回类型。如果库定义了 size_t，则 z_size_t 与 size_t 相同。如果没有定义，则 z_size_t 是一个无符号整数类型，可以包含指针。

     gzfwrite() 返回实际写入大小为 size 的完整项目数，如果发生错误则返回零。如果 size 和 nitems 的乘积溢出，即乘积不适合 z_size_t，那么将不会写入任何内容，返回零，并将错误状态设置为 Z_STREAM_ERROR。
*/

ZEXTERN int ZEXPORTVA gzprintf Z_ARG((gzFile file, const char *format, ...));
/*
     将参数 (...) 转换、格式化、压缩并写入文件，受格式字符串 format 控制，就像 fprintf 一样。gzprintf 返回实际写入的未压缩字节数，或者在出现错误时返回负的 zlib 错误代码。写入的未压缩字节数限制为 8191，或者比 gzbuffer() 给定的缓冲区大小少一个。调用者应确保不超过此限制。如果超过了，gzprintf() 将返回一个错误（0），并且不会写入任何内容。在这种情况下，可能会发生缓冲区溢出，造成不可预测的后果，这只有在 zlib 使用不安全的函数 sprintf() 或 vsprintf() 编译时才可能发生，因为安全的 snprintf() 或 vsnprintf() 函数不可用。可以使用 zlibCompileFlags() 来确定这一点。
*/

ZEXTERN int ZEXPORT gzputs OF((gzFile file, const char *s));
/*
     将给定的以空字符结尾的字符串 s 压缩并写入文件，不包括结尾的空字符。

     gzputs 返回写入的字符数，如果发生错误则返回-1。
*/

ZEXTERN char * ZEXPORT gzgets OF((gzFile file, char *buf, int len));
/*
     从文件中读取并解压字节到缓冲区，直到读取len-1个字符，或者直到读取并传输到缓冲区的换行符，或者遇到文件结束条件。如果读取了任何字符，或者len为1，那么字符串将以空字符结尾。如果由于文件结束或len小于1而没有读取任何字符，则缓冲区将保持不变。

     gzgets返回一个以空字符结尾的字符串buf，或者在文件结束或出现错误的情况下返回NULL。如果发生错误，则buf中的内容是不确定的。
*/

ZEXTERN int ZEXPORT gzputc OF((gzFile file, int c));
/*
     压缩并将c（转换为无符号字符）写入文件。gzputc返回写入的值，或者在发生错误的情况下返回-1。
*/

ZEXTERN int ZEXPORT gzgetc OF((gzFile file));
/*
     从文件中读取并解压一个字节。gzgetc返回这个字节，或者在文件结束或错误的情况下返回-1。这是为了提高速度而实现的宏。因此，它不执行其他函数所做的所有检查。即它不检查file是否为NULL，也不检查file指向的结构是否已被破坏。
*/

ZEXTERN int ZEXPORT gzungetc OF((int c, gzFile file));
/*
     将c推回流中，以便在下一次读取时作为第一个字符读取。至少始终允许推回一个字符。gzungetc()返回推送的字符，或者在失败时返回-1。如果c为-1，则gzungetc()将失败，并且如果已经推送了一个字符但尚未读取，则可能会失败。如果在gzopen或gzdopen之后立即使用gzungetc，则至少允许推送字符的输出缓冲区大小。（参见上面的gzbuffer。）如果使用gzseek()或gzrewind()重新定位流，则推送的字符将被丢弃。
*/

ZEXTERN int ZEXPORT gzflush OF((gzFile file, int flush));
/*
     刷新所有待定的输出到文件。flush参数与deflate()函数中的参数相同。返回值是zlib错误号（参见下面的gzerror函数）。只有在写入时才允许使用gzflush。

     如果flush参数为Z_FINISH，则剩余数据将被写入，并且输出中的gzip流将被完成。如果再次调用gzwrite()，则将在输出中启动新的gzip流。gzread()能够读取这样的串联gzip流。

     仅在绝对必要时才应调用gzflush，因为如果调用太频繁，它会降低压缩效果。
*/

/*
ZEXTERN z_off_t ZEXPORT gzseek OF((gzFile file,
                                   z_off_t offset, int whence));

     设置下一次对文件进行gzread或gzwrite时相对于whence的偏移量的起始位置。偏移量表示未压缩数据流中的字节数。whence参数的定义与lseek(2)中的相同；不支持值SEEK_END。

     如果文件以读取模式打开，此函数将被模拟，但可能非常慢。如果文件以写入模式打开，仅支持向前搜索；然后gzseek会压缩一系列零直到新的起始位置。

     gzseek返回从未压缩流的开头开始以字节为单位测量的结果偏移位置，或者在错误的情况下返回-1，特别是如果文件以写入模式打开，并且新的起始位置在当前位置之前。
*/

ZEXTERN int ZEXPORT    gzrewind OF((gzFile file));
/*
     倒带文件。此函数仅支持读取。

     gzrewind(file)等同于(int)gzseek(file, 0L, SEEK_SET)。
*/

/*
# 返回文件的下一个 gzread 或 gzwrite 的起始位置
ZEXTERN z_off_t ZEXPORT gztell OF((gzFile file));

     返回文件的下一个 gzread 或 gzwrite 的起始位置。
     这个位置代表了未压缩数据流中的字节数，即使使用 gzdopen() 从文件中间开始读取或追加一个 gzip 流，这个位置也是零。

     gztell(file) 等同于 gzseek(file, 0L, SEEK_CUR)
*/

/*
# 返回文件的当前压缩（实际）读取或写入偏移量
ZEXTERN z_off_t ZEXPORT gzoffset OF((gzFile file));

     返回文件的当前压缩（实际）读取或写入偏移量。这个偏移量包括了在 gzip 流之前的字节数，例如在追加或使用 gzdopen() 读取时。在读取时，偏移量不包括尚未使用的缓冲输入。这个信息可以用于进度指示器。在出错时，gzoffset() 返回 -1。
*/

# 返回 true（1），如果在读取时文件的结束标志已经设置，否则返回 false（0）
ZEXTERN int ZEXPORT gzeof OF((gzFile file));
/*
     如果文件的结束标志在读取时被设置，则返回 true（1），否则返回 false（0）。注意，结束标志只有在读取尝试超出输入的末尾时才会被设置，但是不足以完成读取。因此，就像 feof() 一样，即使没有更多数据可读，gzeof() 也可能返回 false，如果最后的读取请求正好是输入文件中剩余字节数的精确倍数。如果输入文件大小是缓冲区大小的精确倍数，就会发生这种情况。

     如果 gzeof() 返回 true，则读取函数将不再返回更多数据，除非结束标志被 gzclearerr() 重置，并且输入文件自上次检测到文件结束以来已经增长。
*/

# 返回文件是否是直接读取或写入的
ZEXTERN int ZEXPORT gzdirect OF((gzFile file));
# 返回 true (1) 如果文件在读取时直接被复制，或者返回 false (0) 如果文件是被解压缩的 gzip 流
# 如果输入文件为空，gzdirect() 将返回 true，因为输入不包含 gzip 流
# 如果在 gzopen() 或 gzdopen() 后立即使用 gzdirect()，它将导致分配缓冲区以允许读取文件以确定它是否是 gzip 文件。因此，如果使用了 gzbuffer()，应在调用 gzdirect() 之前调用它
# 在写入时，gzdirect() 返回 true (1) 如果请求了透明写入 ("wT" 用于 gzopen() 模式)，否则返回 false (0)。（注意：在写入时不需要使用 gzdirect()。必须显式请求透明写入，因此应用程序已经知道答案。在静态链接时，使用 gzdirect() 将包括所有用于 gzip 文件读取和解压缩的 zlib 代码，这可能不是期望的。）
ZEXTERN int ZEXPORT    gzclose OF((gzFile file));
# 刷新文件的所有待处理输出（如果需要），关闭文件并释放（解）压缩状态。请注意，一旦关闭文件，就不能再使用文件调用 gzerror，因为其结构已被释放。gzclose 不得在同一文件上调用多次，就像不能在同一分配上调用多次 free 一样。
# 如果文件无效，gzclose 将返回 Z_STREAM_ERROR，文件操作错误则返回 Z_ERRNO，内存不足则返回 Z_MEM_ERROR，如果最后一次读取在 gzip 流的中间结束，则返回 Z_BUF_ERROR，成功时返回 Z_OK。
ZEXTERN int ZEXPORT gzclose_r OF((gzFile file));
ZEXTERN int ZEXPORT gzclose_w OF((gzFile file));
"""
Same as gzclose(), but gzclose_r() is only for use when reading, and
gzclose_w() is only for use when writing or appending. The advantage to
using these instead of gzclose() is that they avoid linking in zlib
compression or decompression code that is not used when only reading or only
writing respectively. If gzclose() is used, then both compression and
decompression code will be included the application when linking to a static
zlib library.
"""
ZEXTERN const char * ZEXPORT gzerror OF((gzFile file, int *errnum));
"""
Return the error message for the last error which occurred on file.
errnum is set to zlib error number. If an error occurred in the file system
and not in the compression library, errnum is set to Z_ERRNO and the
application may consult errno to get the exact error code.

The application must not modify the returned string. Future calls to
this function may invalidate the previously returned string. If file is
closed, then the string previously returned by gzerror will no longer be
available.

gzerror() should be used to distinguish errors from end-of-file for those
functions above that do not distinguish those cases in their return values.
"""
ZEXTERN void ZEXPORT gzclearerr OF((gzFile file));
"""
Clear the error and end-of-file flags for file. This is analogous to the
clearerr() function in stdio. This is useful for continuing to read a gzip
file that is being written concurrently.
"""
#endif /* !Z_SOLO */

/* checksum functions */

"""
These functions are not related to compression but are exported
anyway because they might be useful in applications using the compression
library.
"""
ZEXTERN uLong ZEXPORT adler32 OF((uLong adler, const Bytef *buf, uInt len));
/*
     更新 Adler-32 校验和，使用 buf[0..len-1] 字节，并返回更新后的校验和。Adler-32 值在 32 位无符号整数的范围内。如果 buf 是 Z_NULL，则此函数返回校验和的所需初始值。

     Adler-32 校验和几乎和 CRC-32 一样可靠，但计算速度更快。

   使用示例：

     uLong adler = adler32(0L, Z_NULL, 0);

     while (read_buffer(buffer, length) != EOF) {
       adler = adler32(adler, buffer, length);
     }
     if (adler != original_adler) error();
*/

ZEXTERN uLong ZEXPORT adler32_z OF((uLong adler, const Bytef *buf,
                                    z_size_t len));
/*
     与 adler32() 相同，但使用 size_t 长度。
*/

/*
ZEXTERN uLong ZEXPORT adler32_combine OF((uLong adler1, uLong adler2,
                                          z_off_t len2));

     将两个 Adler-32 校验和合并为一个。对于两个字节序列 seq1 和 seq2，长度分别为 len1 和 len2，分别计算了每个的 Adler-32 校验和，adler1 和 adler2。adler32_combine() 返回 seq1 和 seq2 连接后的 Adler-32 校验和，只需要 adler1、adler2 和 len2。注意，z_off_t 类型（类似于 off_t）是有符号整数。如果 len2 为负数，则结果没有意义或实用性。
*/

ZEXTERN uLong ZEXPORT crc32 OF((uLong crc, const Bytef *buf, uInt len));
/*
     使用 buf[0..len-1] 字节更新运行中的 CRC-32，并返回更新后的 CRC-32。CRC-32 值在 32 位无符号整数的范围内。如果 buf 是 Z_NULL，则此函数返回 crc 的所需初始值。在此函数中执行预处理和后处理（取反），因此应用程序不应该执行。

   使用示例：

     uLong crc = crc32(0L, Z_NULL, 0);

     while (read_buffer(buffer, length) != EOF) {
       crc = crc32(crc, buffer, length);
     }
     if (crc != original_crc) error();
*/
# 定义一个函数 crc32_z，用于计算给定数据的 CRC-32 校验值
ZEXTERN uLong ZEXPORT crc32_z OF((uLong crc, const Bytef *buf, z_size_t len));
/*
     与 crc32() 相同，但使用 size_t 类型的长度参数。
*/

/*
ZEXTERN uLong ZEXPORT crc32_combine OF((uLong crc1, uLong crc2, z_off_t len2));

     将两个 CRC-32 校验值合并为一个。对于两个字节序列 seq1 和 seq2，长度分别为 len1 和 len2，分别计算了它们的 CRC-32 校验值 crc1 和 crc2。crc32_combine() 返回将 seq1 和 seq2 连接起来的 CRC-32 校验值，只需要 crc1、crc2 和 len2。
*/

/*
ZEXTERN uLong ZEXPORT crc32_combine_gen OF((z_off_t len2));

     返回与长度 len2 对应的操作符，用于 crc32_combine_op()。
*/

ZEXTERN uLong ZEXPORT crc32_combine_op OF((uLong crc1, uLong crc2, uLong op));
/*
     给出与 crc32_combine() 相同的结果，但使用 op 替代 len2。op 是由 len2 通过 crc32_combine_gen() 生成的。如果生成的 op 被多次使用，这将比 crc32_combine() 更快。
*/


                        /* 各种黑客技巧，不要看 :) */

/* deflateInit 和 inflateInit 是宏，用于检查 zlib 版本和编译器对 z_stream 的视图：
 */
ZEXTERN int ZEXPORT deflateInit_ OF((z_streamp strm, int level, const char *version, int stream_size));
ZEXTERN int ZEXPORT inflateInit_ OF((z_streamp strm, const char *version, int stream_size));
ZEXTERN int ZEXPORT deflateInit2_ OF((z_streamp strm, int  level, int  method, int windowBits, int memLevel, int strategy, const char *version, int stream_size));
ZEXTERN int ZEXPORT inflateInit2_ OF((z_streamp strm, int  windowBits, const char *version, int stream_size));
# 定义了一个名为 inflateBackInit_ 的函数，用于初始化反向解压缩流
# 参数包括压缩流指针 strm，窗口大小 windowBits，窗口数组 window，版本信息 version，流大小 stream_size
ZEXTERN int ZEXPORT inflateBackInit_ OF((z_streamp strm, int windowBits,
                                         unsigned char FAR *window,
                                         const char *version,
                                         int stream_size));
# 如果定义了 Z_PREFIX_SET，则使用带有 z_ 前缀的宏定义
# 否则使用不带前缀的宏定义
# 定义了一系列的宏，用于初始化压缩流和解压缩流
# 包括 deflateInit、inflateInit、deflateInit2、inflateInit2、inflateBackInit
# 每个宏都接受不同的参数，并调用对应的初始化函数
# 参数包括压缩流指针 strm，压缩级别 level，压缩方法 method，窗口大小 windowBits，内存级别 memLevel，策略 strategy，窗口数组 window
# 这些宏都使用了 ZLIB_VERSION 和 sizeof(z_stream) 作为参数
#else
# 如果没有定义 Z_PREFIX_SET，则使用不带前缀的宏定义
# 同样定义了一系列的宏，用于初始化压缩流和解压缩流
# 包括 deflateInit、inflateInit、deflateInit2、inflateInit2、inflateBackInit
# 每个宏都接受不同的参数，并调用对应的初始化函数
# 参数包括压缩流指针 strm，压缩级别 level，压缩方法 method，窗口大小 windowBits，内存级别 memLevel，策略 strategy，窗口数组 window
# 这些宏都使用了 ZLIB_VERSION 和 sizeof(z_stream) 作为参数
#endif
# 如果没有定义 Z_SOLO，则继续执行后面的代码
/* gzgetc() 宏及其支持函数和暴露的数据结构。注意，真正的内部状态比暴露的结构要大得多。这个简化的结构只暴露了足够供 gzgetc() 宏使用的部分。用户不应该修改这些暴露的元素，因为它们的名称或行为可能会在将来发生变化，甚至可能是任意的。它们只能被 gzgetc() 宏使用。你已经被警告过了。 */
struct gzFile_s {
    unsigned have;
    unsigned char *next;
    z_off64_t pos;
};
ZEXTERN int ZEXPORT gzgetc_ OF((gzFile file));  /* 向后兼容性 */
#ifdef Z_PREFIX_SET
#  undef z_gzgetc
#  define z_gzgetc(g) \
          ((g)->have ? ((g)->have--, (g)->pos++, *((g)->next)++) : (gzgetc)(g))
#else
#  define gzgetc(g) \
          ((g)->have ? ((g)->have--, (g)->pos++, *((g)->next)++) : (gzgetc)(g))
#endif

/* 如果定义了 _LARGEFILE64_SOURCE，提供 64 位偏移函数，如果定义了 _FILE_OFFSET_BITS 为 64（如果两者都为真，则应用程序获得 *64 函数，并且常规函数被更改为 64 位）-- 如果在没有大文件支持的系统上设置了这些选项，则 _LFS64_LARGEFILE 也必须为真 */
#ifdef Z_LARGE64
   ZEXTERN gzFile ZEXPORT gzopen64 OF((const char *, const char *));
   ZEXTERN z_off64_t ZEXPORT gzseek64 OF((gzFile, z_off64_t, int));
   ZEXTERN z_off64_t ZEXPORT gztell64 OF((gzFile));
   ZEXTERN z_off64_t ZEXPORT gzoffset64 OF((gzFile));
   ZEXTERN uLong ZEXPORT adler32_combine64 OF((uLong, uLong, z_off64_t));
   ZEXTERN uLong ZEXPORT crc32_combine64 OF((uLong, uLong, z_off64_t));
   ZEXTERN uLong ZEXPORT crc32_combine_gen64 OF((z_off64_t));
#endif

#if !defined(ZLIB_INTERNAL) && defined(Z_WANT64)
#  ifdef Z_PREFIX_SET
#    define z_gzopen z_gzopen64
#    define z_gzseek z_gzseek64
#    define z_gztell z_gztell64
#    define z_gzoffset z_gzoffset64
#    define z_adler32_combine z_adler32_combine64
# 如果定义了 Z_LARGE64，则使用 64 位的 CRC32 和 Adler32 函数
# 否则，使用 32 位的函数
# 如果没有定义 Z_LARGE64，则定义以下 64 位函数
# 定义 64 位的 gzopen 函数
ZEXTERN gzFile ZEXPORT gzopen64 OF((const char *, const char *));
# 定义 64 位的 gzseek 函数
ZEXTERN z_off_t ZEXPORT gzseek64 OF((gzFile, z_off_t, int));
# 定义 64 位的 gztell 函数
ZEXTERN z_off_t ZEXPORT gztell64 OF((gzFile));
# 定义 64 位的 gzoffset 函数
ZEXTERN z_off_t ZEXPORT gzoffset64 OF((gzFile));
# 定义 64 位的 adler32_combine 函数
ZEXTERN uLong ZEXPORT adler32_combine64 OF((uLong, uLong, z_off_t));
# 定义 64 位的 crc32_combine 函数
ZEXTERN uLong ZEXPORT crc32_combine64 OF((uLong, uLong, z_off_t));
# 定义 64 位的 crc32_combine_gen 函数
ZEXTERN uLong ZEXPORT crc32_combine_gen64 OF((z_off_t));
# 否则，定义以下 32 位函数
# 定义 32 位的 gzopen 函数
ZEXTERN gzFile ZEXPORT gzopen OF((const char *, const char *));
# 定义 32 位的 gzseek 函数
ZEXTERN z_off_t ZEXPORT gzseek OF((gzFile, z_off_t, int));
# 定义 32 位的 gztell 函数
ZEXTERN z_off_t ZEXPORT gztell OF((gzFile));
# 定义 32 位的 gzoffset 函数
ZEXTERN z_off_t ZEXPORT gzoffset OF((gzFile));
# 定义 32 位的 adler32_combine 函数
ZEXTERN uLong ZEXPORT adler32_combine OF((uLong, uLong, z_off_t));
# 定义 32 位的 crc32_combine 函数
ZEXTERN uLong ZEXPORT crc32_combine OF((uLong, uLong, z_off_t));
# 定义 32 位的 crc32_combine_gen 函数
ZEXTERN uLong ZEXPORT crc32_combine_gen OF((z_off_t));
# 如果是 Z_SOLO 模式，则定义以下函数
# 定义 adler32_combine 函数
ZEXTERN uLong ZEXPORT adler32_combine OF((uLong, uLong, z_off_t));
# 定义 crc32_combine 函数
ZEXTERN uLong ZEXPORT crc32_combine OF((uLong, uLong, z_off_t));
# 定义 crc32_combine_gen 函数
ZEXTERN uLong ZEXPORT crc32_combine_gen OF((z_off_t));
# 未记录的函数
# 定义 zError 函数
ZEXTERN const char * ZEXPORT zError OF((int));
# 定义 inflateSyncPoint 函数
ZEXTERN int ZEXPORT inflateSyncPoint OF((z_streamp));
# 定义 get_crc_table 函数
ZEXTERN const z_crc_t FAR * ZEXPORT get_crc_table OF((void));
# 定义 inflateUndermine 函数
ZEXTERN int ZEXPORT inflateUndermine OF((z_streamp, int));
# 定义 inflateValidate 函数
ZEXTERN int ZEXPORT inflateValidate OF((z_streamp, int));
# 定义 inflateCodesUsed 函数
ZEXTERN unsigned long ZEXPORT inflateCodesUsed OF((z_streamp));
# 定义一个外部函数，用于重置解压缩器的状态，保持输入和输出缓冲区不变
ZEXTERN int            ZEXPORT inflateResetKeep OF((z_streamp));
# 定义一个外部函数，用于重置压缩器的状态，保持输入和输出缓冲区不变
ZEXTERN int            ZEXPORT deflateResetKeep OF((z_streamp));
# 如果在 Windows 平台并且没有定义 Z_SOLO
ZEXTERN gzFile         ZEXPORT gzopen_w OF((const wchar_t *path,
                                            const char *mode));
#endif
# 如果定义了 STDC 或者 Z_HAVE_STDARG_H
# 并且没有定义 Z_SOLO
# 定义一个外部函数，用于格式化输出到 gzFile
ZEXTERN int            ZEXPORTVA gzvprintf Z_ARG((gzFile file,
                                                  const char *format,
                                                  va_list va));
# 如果是 C++ 环境
}
#endif
# 结束条件编译，防止重复包含
#endif /* ZLIB_H */
```