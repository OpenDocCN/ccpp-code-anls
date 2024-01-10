# `ggml\examples\dr_wav.h`

```
/*
WAV音频加载器和写入器。选择公共领域或MIT-0。请参阅本文件末尾的许可声明。
dr_wav - v0.12.16 - 2020-12-02

David Reid - mackron@gmail.com

GitHub: https://github.com/mackron/dr_libs
*/

/*
版本说明 - 版本0.12
============================
版本0.12包括对自定义块处理的重大更改。


对块回调的更改
-------------------------
dr_wav支持在遇到块时触发回调（除了WAVE和FMT块）。回调已更新，包括容器（RIFF或Wave64）和包含有关波形文件中数据格式的信息的FMT块。

以前，没有直接方法来确定容器，因此也没有办法区分块头中的不同ID（RIFF和Wave64容器以不同方式编码块ID）。`container`参数可用于知道要使用哪个ID。

有时，在触发块回调时了解数据格式可能很有用。现在，将指向`drwav_fmt`对象的指针传递到块回调中，这将为您提供有关数据格式的信息。要确定采样格式，请使用`drwav_fmt_get_format()`。这将返回`DR_WAVE_FORMAT_*`之一的标记。
*/

/*
介绍
============
这是一个单文件库。要使用它，在一个.c文件中做如下操作。

    ```c
    #define DR_WAV_IMPLEMENTATION
    #include "dr_wav.h"
    ```

然后，您可以像使用任何其他头文件一样，在程序的其他部分中#include此文件。要读取音频数据，请执行以下操作：

    ```c
    drwav wav;
    if (!drwav_init_file(&wav, "my_song.wav", NULL)) {
        // 打开WAV文件时出错。
    }

    drwav_int32* pDecodedInterleavedPCMFrames = malloc(wav.totalPCMFrameCount * wav.channels * sizeof(drwav_int32));
    // 读取实际解码的样本数，将解码后的 PCM 帧数据存储到 pDecodedInterleavedPCMFrames 中
    size_t numberOfSamplesActuallyDecoded = drwav_read_pcm_frames_s32(&wav, wav.totalPCMFrameCount, pDecodedInterleavedPCMFrames);

    // 结束 WAV 文件的解码，释放相关资源
    drwav_uninit(&wav);
// 以单个操作快速打开和读取音频数据
unsigned int channels;  // 声道数
unsigned int sampleRate;  // 采样率
drwav_uint64 totalPCMFrameCount;  // PCM帧总数
// 打开并读取PCM帧数据，返回浮点数指针
float* pSampleData = drwav_open_file_and_read_pcm_frames_f32("my_song.wav", &channels, &sampleRate, &totalPCMFrameCount, NULL);
if (pSampleData == NULL) {
    // 打开和读取WAV文件出错
}

...

// 释放内存
drwav_free(pSampleData);

// 读取PCM帧数据
size_t framesRead = drwav_read_pcm_frames(&wav, wav.totalPCMFrameCount, pDecodedInterleavedPCMFrames);

// 读取音频数据的原始字节，如果dr_wav不支持特定的数据格式，则可能会有用
size_t bytesRead = drwav_read_raw(&wav, bytesToRead, pRawDataBuffer);

// dr_wav也可以用于输出WAV文件，目前不支持压缩格式
// 使用drwav_init_write()、drwav_init_file_write()等来初始化，使用drwav_write_pcm_frames()来写入样本，或者使用drwav_write_raw()来写入"data"块中的原始数据
drwav_data_format format;
format.container = drwav_container_riff;  // <-- drwav_container_riff = 普通WAV文件，drwav_container_w64 = Sony Wave64
format.format = DR_WAVE_FORMAT_PCM;  // <-- 任何DR_WAVE_FORMAT_*代码
format.channels = 2;  // 声道数
format.sampleRate = 44100;  // 采样率
format.bitsPerSample = 16;  // 每个样本的位数
drwav_init_file_write(&wav, "data/recording.wav", &format, NULL);

...

// 写入PCM帧数据
drwav_uint64 framesWritten = drwav_write_pcm_frames(pWav, frameCount, pSamples);
/*
dr_wav 提供了对 Sony Wave64 格式的无缝支持。解码器将自动检测它，并且应该可以在没有任何手动干预的情况下正常工作。

构建选项
=============
#include 这些选项之前。

#define DR_WAV_NO_CONVERSION_API
  禁用转换 API，如 `drwav_read_pcm_frames_f32()` 和 `drwav_s16_to_f32()`。

#define DR_WAV_NO_STDIO
  禁用从文件初始化解码器的 API，如 `drwav_init_file()`、`drwav_init_file_write()` 等。

注：
=====
- 样本始终是交错的。
- 默认读取函数不执行任何数据转换。使用 `drwav_read_pcm_frames_f32()`、`drwav_read_pcm_frames_s32()` 和 `drwav_read_pcm_frames_s16()` 来读取和转换音频数据为 32 位浮点、有符号 32 位整数和有符号 16 位整数样本。经过测试和支持的内部格式包括以下内容：
  - 无符号 8 位 PCM
  - 有符号 12 位 PCM
  - 有符号 16 位 PCM
  - 有符号 24 位 PCM
  - 有符号 32 位 PCM
  - IEEE 32 位浮点
  - IEEE 64 位浮点
  - A-law 和 u-law
  - Microsoft ADPCM
  - IMA ADPCM (DVI，格式代码 0x11)
- dr_wav 将尽力读取 WAV 文件，即使它不严格符合 WAV 格式。

*/

#ifndef dr_wav_h
#define dr_wav_h

#ifdef __cplusplus
extern "C" {
#endif

#define DRWAV_STRINGIFY(x)      #x
#define DRWAV_XSTRINGIFY(x)     DRWAV_STRINGIFY(x)

#define DRWAV_VERSION_MAJOR     0
#define DRWAV_VERSION_MINOR     12
#define DRWAV_VERSION_REVISION  16
#define DRWAV_VERSION_STRING    DRWAV_XSTRINGIFY(DRWAV_VERSION_MAJOR) "." DRWAV_XSTRINGIFY(DRWAV_VERSION_MINOR) "." DRWAV_XSTRINGIFY(DRWAV_VERSION_REVISION)

#include <stddef.h> /* For size_t. */

/* Sized types. */
typedef   signed char           drwav_int8;
typedef unsigned char           drwav_uint8;
typedef   signed short          drwav_int16;
typedef unsigned short          drwav_uint16;
typedef   signed int            drwav_int32;
// 定义无符号 32 位整数类型
typedef unsigned int            drwav_uint32;
// 如果是 MSC 编译器
#if defined(_MSC_VER)
    // 定义有符号 64 位整数类型
    typedef   signed __int64    drwav_int64;
    // 定义无符号 64 位整数类型
    typedef unsigned __int64    drwav_uint64;
// 如果不是 MSC 编译器
#else
    // 如果是 clang 或者 GCC 版本大于 4.6
    #if defined(__clang__) || (defined(__GNUC__) && (__GNUC__ > 4 || (__GNUC__ == 4 && __GNUC_MINOR__ >= 6)))
        // 忽略 long long 类型的警告
        #pragma GCC diagnostic push
        #pragma GCC diagnostic ignored "-Wlong-long"
        // 如果是 clang 编译器，忽略 C++11 long long 类型的警告
        #if defined(__clang__)
            #pragma GCC diagnostic ignored "-Wc++11-long-long"
        #endif
    #endif
    // 定义有符号 64 位整数类型
    typedef   signed long long  drwav_int64;
    // 定义无符号 64 位整数类型
    typedef unsigned long long  drwav_uint64;
    // 如果是 clang 或者 GCC 版本大于 4.6，恢复警告设置
    #if defined(__clang__) || (defined(__GNUC__) && (__GNUC__ > 4 || (__GNUC__ == 4 && __GNUC_MINOR__ >= 6)))
        #pragma GCC diagnostic pop
    #endif
#endif
// 如果是 64 位系统，使用无符号 64 位整数类型作为指针类型
#if defined(__LP64__) || defined(_WIN64) || (defined(__x86_64__) && !defined(__ILP32__)) || defined(_M_X64) || defined(__ia64) || defined (_M_IA64) || defined(__aarch64__) || defined(__powerpc64__)
    typedef drwav_uint64        drwav_uintptr;
// 如果不是 64 位系统，使用无符号 32 位整数类型作为指针类型
#else
    typedef drwav_uint32        drwav_uintptr;
#endif
// 定义 8 位无符号整数类型
typedef drwav_uint8             drwav_bool8;
// 定义 32 位无符号整数类型
typedef drwav_uint32            drwav_bool32;
// 定义 TRUE 和 FALSE 常量
#define DRWAV_TRUE              1
#define DRWAV_FALSE             0

// 如果没有定义 DRWAV_API
#if !defined(DRWAV_API)
    #if defined(DRWAV_DLL)
        #if defined(_WIN32)
            #define DRWAV_DLL_IMPORT  __declspec(dllimport)  // 如果定义了DRWAV_DLL并且是在Windows平台，则定义为dll导入
            #define DRWAV_DLL_EXPORT  __declspec(dllexport)  // 如果定义了DRWAV_DLL并且是在Windows平台，则定义为dll导出
            #define DRWAV_DLL_PRIVATE static  // 如果定义了DRWAV_DLL并且是在Windows平台，则定义为静态
        #else
            #if defined(__GNUC__) && __GNUC__ >= 4
                #define DRWAV_DLL_IMPORT  __attribute__((visibility("default")))  // 如果定义了DRWAV_DLL并且是在非Windows平台且使用了GCC 4以上版本，则定义为默认可见性
                #define DRWAV_DLL_EXPORT  __attribute__((visibility("default")))  // 如果定义了DRWAV_DLL并且是在非Windows平台且使用了GCC 4以上版本，则定义为默认可见性
                #define DRWAV_DLL_PRIVATE __attribute__((visibility("hidden")))  // 如果定义了DRWAV_DLL并且是在非Windows平台且使用了GCC 4以上版本，则定义为隐藏
            #else
                #define DRWAV_DLL_IMPORT  // 如果定义了DRWAV_DLL并且是在非Windows平台且使用了GCC 4以下版本，则为空
                #define DRWAV_DLL_EXPORT  // 如果定义了DRWAV_DLL并且是在非Windows平台且使用了GCC 4以下版本，则为空
                #define DRWAV_DLL_PRIVATE static  // 如果定义了DRWAV_DLL并且是在非Windows平台且使用了GCC 4以下版本，则定义为静态
            #endif
        #endif

        #if defined(DR_WAV_IMPLEMENTATION) || defined(DRWAV_IMPLEMENTATION)
            #define DRWAV_API  DRWAV_DLL_EXPORT  // 如果定义了DR_WAV_IMPLEMENTATION或DRWAV_IMPLEMENTATION，则定义为dll导出
        #else
            #define DRWAV_API  DRWAV_DLL_IMPORT  // 否则定义为dll导入
        #endif
        #define DRWAV_PRIVATE DRWAV_DLL_PRIVATE  // 定义为dll私有
    #else
        #define DRWAV_API extern  // 否则定义为外部链接
        #define DRWAV_PRIVATE static  // 否则定义为静态
    #endif
#endif

typedef drwav_int32 drwav_result;
#define DRWAV_SUCCESS                        0   /* 操作成功 */
#define DRWAV_ERROR                         -1   /* 通用错误 */
#define DRWAV_INVALID_ARGS                  -2   /* 无效参数 */
#define DRWAV_INVALID_OPERATION             -3   /* 无效操作 */
#define DRWAV_OUT_OF_MEMORY                 -4   /* 内存不足 */
#define DRWAV_OUT_OF_RANGE                  -5   /* 超出范围 */
#define DRWAV_ACCESS_DENIED                 -6   /* 访问被拒绝 */
#define DRWAV_DOES_NOT_EXIST                -7   /* 不存在 */
#define DRWAV_ALREADY_EXISTS                -8   /* 已存在 */
#define DRWAV_TOO_MANY_OPEN_FILES           -9   /* 打开文件过多 */
#define DRWAV_INVALID_FILE                  -10  /* 无效文件 */
#define DRWAV_TOO_BIG                       -11  /* 太大 */
#define DRWAV_PATH_TOO_LONG                 -12  /* 路径太长 */
#define DRWAV_NAME_TOO_LONG                 -13  /* 名称太长 */
#define DRWAV_NOT_DIRECTORY                 -14  /* 不是目录 */
#define DRWAV_IS_DIRECTORY                  -15  /* 是目录 */
#define DRWAV_DIRECTORY_NOT_EMPTY           -16  /* 目录不为空 */
#define DRWAV_END_OF_FILE                   -17  /* 文件结束 */
#define DRWAV_NO_SPACE                      -18  /* 空间不足 */
#define DRWAV_BUSY                          -19  /* 忙 */
#define DRWAV_IO_ERROR                      -20  /* IO 错误 */
#define DRWAV_INTERRUPT                     -21  /* 中断 */
#define DRWAV_UNAVAILABLE                   -22  /* 不可用 */
#define DRWAV_ALREADY_IN_USE                -23  /* 已被使用 */
#define DRWAV_BAD_ADDRESS                   -24  /* 错误地址 */
#define DRWAV_BAD_SEEK                      -25  /* 错误寻址 */
#define DRWAV_BAD_PIPE                      -26  /* 错误管道 */
#define DRWAV_DEADLOCK                      -27  /* 死锁 */
#define DRWAV_TOO_MANY_LINKS                -28  /* 链接过多 */
#define DRWAV_NOT_IMPLEMENTED               -29  /* 未实现 */
#define DRWAV_NO_MESSAGE                    -30  /* 无消息 */
#define DRWAV_BAD_MESSAGE                   -31  /* 错误消息 */
#define DRWAV_NO_DATA_AVAILABLE             -32  /* 无可用数据 */
#define DRWAV_INVALID_DATA                  -33  /* 无效数据 */
#define DRWAV_TIMEOUT                       -34  /* 超时 */
#define DRWAV_NO_NETWORK                    -35  /* 无网络 */
#define DRWAV_NOT_UNIQUE                    -36  /* 非唯一 */
#define DRWAV_NOT_SOCKET                    -37  /* 非套接字 */
#define DRWAV_NO_ADDRESS                    -38  /* 无地址 */
#define DRWAV_BAD_PROTOCOL                  -39  /* 错误协议 */
/* 定义一些错误码 */
#define DRWAV_PROTOCOL_UNAVAILABLE          -40
#define DRWAV_PROTOCOL_NOT_SUPPORTED        -41
#define DRWAV_PROTOCOL_FAMILY_NOT_SUPPORTED -42
#define DRWAV_ADDRESS_FAMILY_NOT_SUPPORTED  -43
#define DRWAV_SOCKET_NOT_SUPPORTED          -44
#define DRWAV_CONNECTION_RESET              -45
#define DRWAV_ALREADY_CONNECTED             -46
#define DRWAV_NOT_CONNECTED                 -47
#define DRWAV_CONNECTION_REFUSED            -48
#define DRWAV_NO_HOST                       -49
#define DRWAV_IN_PROGRESS                   -50
#define DRWAV_CANCELLED                     -51
#define DRWAV_MEMORY_ALREADY_MAPPED         -52
#define DRWAV_AT_END                        -53

/* 常见的数据格式 */
#define DR_WAVE_FORMAT_PCM          0x1
#define DR_WAVE_FORMAT_ADPCM        0x2
#define DR_WAVE_FORMAT_IEEE_FLOAT   0x3
#define DR_WAVE_FORMAT_ALAW         0x6
#define DR_WAVE_FORMAT_MULAW        0x7
#define DR_WAVE_FORMAT_DVI_ADPCM    0x11
#define DR_WAVE_FORMAT_EXTENSIBLE   0xFFFE

/* 常量 */
#ifndef DRWAV_MAX_SMPL_LOOPS
#define DRWAV_MAX_SMPL_LOOPS        1
#endif

/* 传递给 drwav_init_ex() 等函数的标志 */
#define DRWAV_SEQUENTIAL            0x00000001

/* 获取 drwav 库的版本信息 */
DRWAV_API void drwav_version(drwav_uint32* pMajor, drwav_uint32* pMinor, drwav_uint32* pRevision);
DRWAV_API const char* drwav_version_string(void);

/* 定义枚举类型 */
typedef enum
{
    drwav_seek_origin_start,    /* 从文件开始处寻找 */
    drwav_seek_origin_current   /* 从当前位置寻找 */
} drwav_seek_origin;

typedef enum
{
    drwav_container_riff,   /* RIFF 容器格式 */
    drwav_container_w64,    /* W64 容器格式 */
    drwav_container_rf64     /* RF64 容器格式 */
} drwav_container;

/* 定义结构体 */
typedef struct
{
    union
    {
        drwav_uint8 fourcc[4];  /* 4 字节的标识符 */
        drwav_uint8 guid[16];    /* 16 字节的全局唯一标识符 */
    } id;

    /* 数据块的大小（字节数） */
    drwav_uint64 sizeInBytes;

    /*
    RIFF = 2 字节对齐。
    W64  = 8 字节对齐。
    */
    unsigned int paddingSize;  /* 填充大小 */
} drwav_chunk_header;
    /* 需要对不被 dr_wav 原生支持的数据格式提供支持。*/

    /* 音频数据的格式标签。*/
    drwav_uint16 formatTag;

    /* 音频数据的通道数。当设置为1时为单声道，2为立体声，以此类推。*/
    drwav_uint16 channels;

    /* 采样率。通常设置为类似44100的值。*/
    drwav_uint32 sampleRate;

    /* 每秒的平均字节数。可能不需要，但为了信息目的而保留。*/
    drwav_uint32 avgBytesPerSec;

    /* 块对齐。这等于通道数乘以每个样本的字节数。*/
    drwav_uint16 blockAlign;

    /* 每个样本的位数。*/
    drwav_uint16 bitsPerSample;

    /* 扩展数据的大小。仅用于内部验证，但为了信息目的而保留。*/
    drwav_uint16 extendedSize;

    /*
    每个样本的有效位数。当 <formatTag> 等于 WAVE_FORMAT_EXTENSIBLE 时，<bitsPerSample>
    总是向上舍入到最接近的8的倍数。这个变量包含有关每个样本的确切有效位数的信息。主要用于信息目的。
    */
    drwav_uint16 validBitsPerSample;

    /* 通道掩码。目前未使用。*/
    drwav_uint32 channelMask;

    /* 子格式，与波形文件中指定的完全相同。*/
    drwav_uint8 subFormat[16];
} drwav_fmt;

// 获取格式信息
DRWAV_API drwav_uint16 drwav_fmt_get_format(const drwav_fmt* pFMT);


/*
数据读取回调函数。返回值是实际读取的字节数。

pUserData   [in]  传递给 drwav_init() 等函数的用户数据。
pBufferOut  [out] 输出缓冲区。
bytesToRead [in]  要读取的字节数。

返回实际读取的字节数。

返回值小于 bytesToRead 表示流的结束。在填充整个 bytesToRead 或者到达流的末尾之前，不要从此回调函数返回。
*/
typedef size_t (* drwav_read_proc)(void* pUserData, void* pBufferOut, size_t bytesToRead);

/*
数据写入回调函数。返回值是实际写入的字节数。

pUserData    [in] 传递给 drwav_init_write() 等函数的用户数据。
pData        [out] 要写入的数据的指针。
bytesToWrite [in] 要写入的字节数。

返回实际写入的字节数。

如果返回值与 bytesToWrite 不同，表示出现错误。
*/
typedef size_t (* drwav_write_proc)(void* pUserData, const void* pData, size_t bytesToWrite);

/*
数据定位回调函数。

pUserData [in] 传递给 drwav_init() 等函数的用户数据。
offset    [in] 要移动的字节数，相对于原点。永远不会是负数。
origin    [in] 定位的原点 - 当前位置或流的起始位置。

返回定位是否成功。

相对于起始位置或当前位置由 "origin" 参数决定，该参数将是 drwav_seek_origin_start 或 drwav_seek_origin_current。
*/
typedef drwav_bool32 (* drwav_seek_proc)(void* pUserData, int offset, drwav_seek_origin origin);

/*
当 drwav_init_ex() 发现一个块时的回调函数。

pChunkUserData    [in] 传递给 drwav_init_ex() 等函数的 pChunkUserData 参数的用户数据。
# 定义一个函数指针，用于在读取时调用
onRead            [in] A pointer to the function to call when reading.
# 定义一个函数指针，用于在寻址时调用
onSeek            [in] A pointer to the function to call when seeking.
# 传递给 drwav_init_ex() 和相关函数的用户数据
pReadSeekUserData [in] The user data that was passed to the pReadSeekUserData parameter of drwav_init_ex() and family.
# 指向包含有关块的基本头信息的对象的指针。用于识别块。
pChunkHeader      [in] A pointer to an object containing basic header information about the chunk. Use this to identify the chunk.
# WAV 文件是 RIFF 还是 Wave64 容器的标志。如果不确定差异，请假定为 RIFF。
container         [in] Whether or not the WAV file is a RIFF or Wave64 container. If you're unsure of the difference, assume RIFF.
# 指向包含“fmt”块内容的对象的指针。
pFMT              [in] A pointer to the object containing the contents of the "fmt" chunk.

# 返回读取和寻址的字节数。
Returns the number of bytes read + seeked.

# 要从块中读取数据，请调用 onRead()，将 pReadSeekUserData 作为第一个参数传入。对于寻址也是一样，使用 onSeek()。返回值必须是您已读取的字节数加上已寻址的字节数。

# 使用 `container` 参数来区分 `pChunkHeader->id` 中的字段。如果容器是 `drwav_container_riff` 或 `drwav_container_rf64`，则应使用 `id.fourcc`，否则应使用 `id.guid`。

# `pFMT` 参数可用于确定波形文件的数据格式。使用 `drwav_fmt_get_format()` 来获取采样格式，它将是 `DR_WAVE_FORMAT_*` 之一的标识符。

# 读取指针将位于块头之后的第一个字节。不得尝试读取超出块边界的内容。
The read pointer will be sitting on the first byte after the chunk's header. You must not attempt to read beyond the boundary of the chunk.
*/
# 定义一个函数指针，用于在读取和寻址时调用
typedef drwav_uint64 (* drwav_chunk_proc)(void* pChunkUserData, drwav_read_proc onRead, drwav_seek_proc onSeek, void* pReadSeekUserData, const drwav_chunk_header* pChunkHeader, drwav_container container, const drwav_fmt* pFMT);

# 用于内部使用的结构。仅用于使用 drwav_init_memory() 打开的加载器。
typedef struct
{
    # 用户数据
    void* pUserData;
    # 分配内存的回调函数
    void* (* onMalloc)(size_t sz, void* pUserData);
    # 重新分配内存的回调函数
    void* (* onRealloc)(void* p, size_t sz, void* pUserData);
    # 释放内存的回调函数
    void  (* onFree)(void* p, void* pUserData);
} drwav_allocation_callbacks;

# 用于内部使用的结构。仅用于使用 drwav_init_memory() 打开的加载器。
typedef struct
{
    # 声明一个指向无符号8位整数的指针变量，用于存储数据的地址
    const drwav_uint8* data;
    # 声明一个用于存储数据大小的变量，以字节为单位
    size_t dataSize;
    # 声明一个用于存储当前读取位置的变量，以字节为单位
    size_t currentReadPos;
} drwav__memory_stream;

/* 用于内部使用的结构。仅用于使用 drwav_init_memory_write() 打开的写入器。 */
typedef struct
{
    void** ppData;
    size_t* pDataSize;
    size_t dataSize;
    size_t dataCapacity;
    size_t currentWritePos;
} drwav__memory_stream_write;

typedef struct
{
    drwav_container container;  /* RIFF, W64. */
    drwav_uint32 format;        /* DR_WAVE_FORMAT_* */
    drwav_uint32 channels;
    drwav_uint32 sampleRate;
    drwav_uint32 bitsPerSample;
} drwav_data_format;


/* 有关 'smpl' 块的详细信息，请参阅以下内容：https://sites.google.com/site/musicgapi/technical-documents/wav-file-format#smpl */
typedef struct
{
    drwav_uint32 cuePointId;
    drwav_uint32 type;
    drwav_uint32 start;
    drwav_uint32 end;
    drwav_uint32 fraction;
    drwav_uint32 playCount;
} drwav_smpl_loop;

 typedef struct
{
    drwav_uint32 manufacturer;
    drwav_uint32 product;
    drwav_uint32 samplePeriod;
    drwav_uint32 midiUnityNotes;
    drwav_uint32 midiPitchFraction;
    drwav_uint32 smpteFormat;
    drwav_uint32 smpteOffset;
    drwav_uint32 numSampleLoops;
    drwav_uint32 samplerData;
    drwav_smpl_loop loops[DRWAV_MAX_SMPL_LOOPS];
} drwav_smpl;

typedef struct
{
    /* 当需要更多数据时调用的函数指针。 */
    drwav_read_proc onRead;

    /* 当需要写入数据时调用的函数指针。仅在以写入模式打开 drwav 对象时使用。 */
    drwav_write_proc onWrite;

    /* 当需要对 wav 文件进行寻址时调用的函数指针。 */
    drwav_seek_proc onSeek;

    /* 传递给回调函数的用户数据。 */
    void* pUserData;

    /* 分配回调。 */
    drwav_allocation_callbacks allocationCallbacks;


    /* WAV 文件是否格式化为标准的 RIFF 文件或 W64。 */
    drwav_container container;


    /* 包含与 wav 文件完全相符的格式信息的结构。 */
    drwav_fmt fmt;
    /* 采样率。通常设置为44100。 */
    drwav_uint32 sampleRate;

    /* 声道数。单声道流设置为1，立体声设置为2，以此类推。 */
    drwav_uint16 channels;

    /* 每个样本的位数。通常设置为16、24等。 */
    drwav_uint16 bitsPerSample;

    /* 等于 fmt.formatTag，或者如果 fmt.formatTag 等于 65534 (WAVE_FORMAT_EXTENSIBLE) 则等于 fmt.subFormat 指定的值。 */
    drwav_uint16 translatedFormatTag;

    /* 构成音频数据的 PCM 帧的总数。 */
    drwav_uint64 totalPCMFrameCount;


    /* 数据块的字节数。 */
    drwav_uint64 dataChunkDataSize;
    
    /* 数据块第一个字节在流中的位置。用于定位。 */
    drwav_uint64 dataChunkDataPos;

    /* 数据块中剩余的字节数。 */
    drwav_uint64 bytesRemaining;


    /*
    仅在顺序写模式中使用。在初始化时跟踪“数据”块的期望大小。对于非顺序写和以读模式打开的 drwav 对象，始终设置为0。用于验证。
    */
    drwav_uint64 dataChunkDataSizeTargetWrite;

    /* 跟踪 wav 写入器是否以顺序模式初始化。 */
    drwav_bool32 isSequentialWrite;


    /* smpl 块。 */
    drwav_smpl smpl;


    /* 在使用 drwav_init_memory() 打开解码器时避免 DRWAV_MALLOC() 的一个小技巧。 */
    drwav__memory_stream memoryStream;
    drwav__memory_stream_write memoryStreamWrite;

    /* 压缩格式的通用数据。这些数据在所有块压缩格式之间共享。 */
    struct
    {
        drwav_uint64 iCurrentPCMFrame;  /* 下一个由 drwav_read_*() 读取的 PCM 帧的索引。与 "totalPCMFrameCount" 一起使用，以确保我们不会在最后一个块的末尾读取过多的样本。 */
    } compressed;
    
    /* Microsoft ADPCM 特定数据。 */
    # 定义一个结构体，用于存储 MSADPCM 解码器的相关信息
    struct
    {
        # 块中剩余的字节数
        drwav_uint32 bytesRemainingInBlock;
        # 预测器数组，用于存储预测值
        drwav_uint16 predictor[2];
        # 增量数组，用于存储增量值
        drwav_int32  delta[2];
        # 在解码过程中，用于存储样本数据的缓存
        drwav_int32  cachedFrames[4];  /* Samples are stored in this cache during decoding. */
        # 缓存中的帧数
        drwav_uint32 cachedFrameCount;
        # 用于存储每个通道的前两个样本的数组
        drwav_int32  prevFrames[2][2]; /* The previous 2 samples for each channel (2 channels at most). */
    } msadpcm;

    # 定义一个结构体，用于存储 IMA ADPCM 解码器的相关信息
    /* IMA ADPCM specific data. */
    struct
    {
        # 块中剩余的字节数
        drwav_uint32 bytesRemainingInBlock;
        # 预测器数组，用于存储预测值
        drwav_int32  predictor[2];
        # 步长索引数组，用于存储步长索引值
        drwav_int32  stepIndex[2];
        # 在解码过程中，用于存储样本数据的缓存
        drwav_int32  cachedFrames[16]; /* Samples are stored in this cache during decoding. */
        # 缓存中的帧数
        drwav_uint32 cachedFrameCount;
    } ima;
/*
初始化一个预先分配的 drwav 对象以进行读取。

pWav                         [out]          要初始化的 drwav 对象的指针。
onRead                       [in]           当需要从客户端读取数据时调用的函数。
onSeek                       [in]           当客户端数据的读取位置需要移动时调用的函数。
onChunk                      [in, optional] 在初始化时枚举块时调用的函数。
pUserData, pReadSeekUserData [in, optional] 指向应用程序定义数据的指针，将传递给 onRead 和 onSeek。
pChunkUserData               [in, optional] 指向应用程序定义数据的指针，将传递给 onChunk。
flags                        [in, optional] 用于控制加载方式的一组标志。

如果成功返回 true；否则返回 false。

使用 drwav_uninit() 关闭加载器。

这是初始化 WAV 文件的最低级别函数。您还可以使用 drwav_init_file() 和 drwav_init_memory() 分别从文件或内存块打开流。

flags 的可能值：
  DRWAV_SEQUENTIAL: 加载时不执行向后查找。这将禁用块回调，并且将导致此函数在找到数据块后立即返回。数据块后的任何块将被忽略。

drwav_init() 等效于 "drwav_init_ex(pWav, onRead, onSeek, NULL, pUserData, NULL, 0);".

WAVE 或 FMT 块不会调用 onChunk 回调。在函数返回后，可以从 pWav->fmt 读取 FMT 块的内容。

另请参阅：drwav_init_file()、drwav_init_memory()、drwav_uninit()
*/
DRWAV_API drwav_bool32 drwav_init(drwav* pWav, drwav_read_proc onRead, drwav_seek_proc onSeek, void* pUserData, const drwav_allocation_callbacks* pAllocationCallbacks);
/*
初始化一个预先分配的 drwav 对象用于写入。

onRead             [in]           当需要读取数据时调用的函数。
onSeek             [in]           当写入位置需要移动时调用的函数。
onChunk            [in]           当需要处理块数据时调用的函数。
pReadSeekUserData  [in, optional] 一个指向应用程序定义数据的指针，将传递给 onRead 和 onSeek。
pChunkUserData     [in, optional] 一个指向应用程序定义数据的指针，将传递给 onChunk。
flags              [in]           标志位。
pAllocationCallbacks [in, optional] 用于自定义内存分配的回调函数。

如果成功返回 true；否则返回 false。

使用 drwav_uninit() 关闭写入器。

这是用于初始化 WAV 文件的最低级别函数。您还可以使用 drwav_init_file_write() 和 drwav_init_memory_write() 从文件或内存块中打开流。

如果总样本数已知，可以使用 drwav_init_write_sequential()。这避免了 dr_wav 执行后处理步骤来存储总样本数和数据块的大小，这需要进行向后查找。

另请参阅：drwav_init_file_write()、drwav_init_memory_write()、drwav_uninit()
*/
DRWAV_API drwav_bool32 drwav_init_write(drwav* pWav, const drwav_data_format* pFormat, drwav_write_proc onWrite, drwav_seek_proc onSeek, void* pUserData, const drwav_allocation_callbacks* pAllocationCallbacks);
DRWAV_API drwav_bool32 drwav_init_write_sequential(drwav* pWav, const drwav_data_format* pFormat, drwav_uint64 totalSampleCount, drwav_write_proc onWrite, void* pUserData, const drwav_allocation_callbacks* pAllocationCallbacks);
DRWAV_API drwav_bool32 drwav_init_write_sequential_pcm_frames(drwav* pWav, const drwav_data_format* pFormat, drwav_uint64 totalPCMFrameCount, drwav_write_proc onWrite, void* pUserData, const drwav_allocation_callbacks* pAllocationCallbacks);

/*
确定要写入的整个数据的目标大小（包括所有头和块）的实用函数。
*/
/*
Returns the target size in bytes.

Useful if the application needs to know the size to allocate.

Only writing to the RIFF chunk and one data chunk is currently supported.

See also: drwav_init_write(), drwav_init_file_write(), drwav_init_memory_write()
*/
DRWAV_API drwav_uint64 drwav_target_write_size_bytes(const drwav_data_format* pFormat, drwav_uint64 totalSampleCount);

/*
Uninitializes the given drwav object.

Use this only for objects initialized with drwav_init*() functions (drwav_init(), drwav_init_ex(), drwav_init_write(), drwav_init_write_sequential()).
*/
DRWAV_API drwav_result drwav_uninit(drwav* pWav);


/*
Reads raw audio data.

This is the lowest level function for reading audio data. It simply reads the given number of
bytes of the raw internal sample data.

Consider using drwav_read_pcm_frames_s16(), drwav_read_pcm_frames_s32() or drwav_read_pcm_frames_f32() for
reading sample data in a consistent format.

pBufferOut can be NULL in which case a seek will be performed.

Returns the number of bytes actually read.
*/
DRWAV_API size_t drwav_read_raw(drwav* pWav, size_t bytesToRead, void* pBufferOut);

/*
Reads up to the specified number of PCM frames from the WAV file.

The output data will be in the file's internal format, converted to native-endian byte order. Use
drwav_read_pcm_frames_s16/f32/s32() to read data in a specific format.

If the return value is less than <framesToRead> it means the end of the file has been reached or
you have requested more PCM frames than can possibly fit in the output buffer.

This function will only work when sample data is of a fixed size and uncompressed. If you are
using a compressed format consider using drwav_read_raw() or drwav_read_pcm_frames_s16/s32/f32().

pBufferOut can be NULL in which case a seek will be performed.
*/
DRWAV_API drwav_uint64 drwav_read_pcm_frames(drwav* pWav, drwav_uint64 framesToRead, void* pBufferOut);
# 以小端格式读取给定的 PCM 帧，将结果存储到输出缓冲区中
DRWAV_API drwav_uint64 drwav_read_pcm_frames_le(drwav* pWav, drwav_uint64 framesToRead, void* pBufferOut);

# 以大端格式读取给定的 PCM 帧，将结果存储到输出缓冲区中
DRWAV_API drwav_uint64 drwav_read_pcm_frames_be(drwav* pWav, drwav_uint64 framesToRead, void* pBufferOut);

# 定位到给定的 PCM 帧，成功返回 true，否则返回 false
DRWAV_API drwav_bool32 drwav_seek_to_pcm_frame(drwav* pWav, drwav_uint64 targetFrameIndex);

# 写入原始音频数据，返回实际写入的字节数，如果与 bytesToWrite 不同，表示出现错误
DRWAV_API size_t drwav_write_raw(drwav* pWav, size_t bytesToWrite, const void* pData);

# 写入 PCM 帧，返回实际写入的 PCM 帧数
# 输入样本需要以本机字节顺序存储。在大端架构上，输入数据将被转换为小端。使用 drwav_write_raw() 可以写入原始音频数据而不进行任何转换。
DRWAV_API drwav_uint64 drwav_write_pcm_frames(drwav* pWav, drwav_uint64 framesToWrite, const void* pData);
DRWAV_API drwav_uint64 drwav_write_pcm_frames_le(drwav* pWav, drwav_uint64 framesToWrite, const void* pData);
DRWAV_API drwav_uint64 drwav_write_pcm_frames_be(drwav* pWav, drwav_uint64 framesToWrite, const void* pData);

# 转换工具
#ifndef DR_WAV_NO_CONVERSION_API

# 读取一块音频数据并将其转换为有符号 16 位 PCM 样本
# 如果 pBufferOut 为 NULL，则会执行一个 seek 操作
# 返回实际读取的 PCM 帧数。如果返回值小于 framesToRead，则表示已到达文件末尾。
DRWAV_API drwav_uint64 drwav_read_pcm_frames_s16(drwav* pWav, drwav_uint64 framesToRead, drwav_int16* pBufferOut);
DRWAV_API drwav_uint64 drwav_read_pcm_frames_s16le(drwav* pWav, drwav_uint64 framesToRead, drwav_int16* pBufferOut);
DRWAV_API drwav_uint64 drwav_read_pcm_frames_s16be(drwav* pWav, drwav_uint64 framesToRead, drwav_int16* pBufferOut);
# 将无符号8位PCM样本转换为有符号16位PCM样本的低级函数
DRWAV_API void drwav_u8_to_s16(drwav_int16* pOut, const drwav_uint8* pIn, size_t sampleCount);

# 将有符号24位PCM样本转换为有符号16位PCM样本的低级函数
DRWAV_API void drwav_s24_to_s16(drwav_int16* pOut, const drwav_uint8* pIn, size_t sampleCount);

# 将有符号32位PCM样本转换为有符号16位PCM样本的低级函数
DRWAV_API void drwav_s32_to_s16(drwav_int16* pOut, const drwav_int32* pIn, size_t sampleCount);

# 将IEEE 32位浮点样本转换为有符号16位PCM样本的低级函数
DRWAV_API void drwav_f32_to_s16(drwav_int16* pOut, const float* pIn, size_t sampleCount);

# 将IEEE 64位浮点样本转换为有符号16位PCM样本的低级函数
DRWAV_API void drwav_f64_to_s16(drwav_int16* pOut, const double* pIn, size_t sampleCount);

# 将A-law样本转换为有符号16位PCM样本的低级函数
DRWAV_API void drwav_alaw_to_s16(drwav_int16* pOut, const drwav_uint8* pIn, size_t sampleCount);

# 将u-law样本转换为有符号16位PCM样本的低级函数
DRWAV_API void drwav_mulaw_to_s16(drwav_int16* pOut, const drwav_uint8* pIn, size_t sampleCount);

# 读取音频数据块并将其转换为IEEE 32位浮点样本
# 如果pBufferOut为NULL，则执行一次seek
# 返回实际读取的PCM帧数
# 如果返回值小于<framesToRead>，表示已经到达文件末尾
DRWAV_API drwav_uint64 drwav_read_pcm_frames_f32(drwav* pWav, drwav_uint64 framesToRead, float* pBufferOut);
DRWAV_API drwav_uint64 drwav_read_pcm_frames_f32le(drwav* pWav, drwav_uint64 framesToRead, float* pBufferOut);
DRWAV_API drwav_uint64 drwav_read_pcm_frames_f32be(drwav* pWav, drwav_uint64 framesToRead, float* pBufferOut);
# 将无符号8位PCM样本转换为IEEE 32位浮点样本的低级函数
DRWAV_API void drwav_u8_to_f32(float* pOut, const drwav_uint8* pIn, size_t sampleCount);

# 将有符号16位PCM样本转换为IEEE 32位浮点样本的低级函数
DRWAV_API void drwav_s16_to_f32(float* pOut, const drwav_int16* pIn, size_t sampleCount);

# 将有符号24位PCM样本转换为IEEE 32位浮点样本的低级函数
DRWAV_API void drwav_s24_to_f32(float* pOut, const drwav_uint8* pIn, size_t sampleCount);

# 将有符号32位PCM样本转换为IEEE 32位浮点样本的低级函数
DRWAV_API void drwav_s32_to_f32(float* pOut, const drwav_int32* pIn, size_t sampleCount);

# 将IEEE 64位浮点样本转换为IEEE 32位浮点样本的低级函数
DRWAV_API void drwav_f64_to_f32(float* pOut, const double* pIn, size_t sampleCount);

# 将A-law样本转换为IEEE 32位浮点样本的低级函数
DRWAV_API void drwav_alaw_to_f32(float* pOut, const drwav_uint8* pIn, size_t sampleCount);

# 将u-law样本转换为IEEE 32位浮点样本的低级函数
DRWAV_API void drwav_mulaw_to_f32(float* pOut, const drwav_uint8* pIn, size_t sampleCount);

# 读取音频数据块并将其转换为有符号32位PCM样本
# 如果pBufferOut为NULL，则执行seek操作
# 返回实际读取的PCM帧数
# 如果返回值小于<framesToRead>，表示已经到达文件末尾
DRWAV_API drwav_uint64 drwav_read_pcm_frames_s32(drwav* pWav, drwav_uint64 framesToRead, drwav_int32* pBufferOut);
DRWAV_API drwav_uint64 drwav_read_pcm_frames_s32le(drwav* pWav, drwav_uint64 framesToRead, drwav_int32* pBufferOut);
DRWAV_API drwav_uint64 drwav_read_pcm_frames_s32be(drwav* pWav, drwav_uint64 framesToRead, drwav_int32* pBufferOut);
/* 将无符号8位PCM样本转换为有符号32位PCM样本的低级函数。 */
DRWAV_API void drwav_u8_to_s32(drwav_int32* pOut, const drwav_uint8* pIn, size_t sampleCount);

/* 将有符号16位PCM样本转换为有符号32位PCM样本的低级函数。 */
DRWAV_API void drwav_s16_to_s32(drwav_int32* pOut, const drwav_int16* pIn, size_t sampleCount);

/* 将有符号24位PCM样本转换为有符号32位PCM样本的低级函数。 */
DRWAV_API void drwav_s24_to_s32(drwav_int32* pOut, const drwav_uint8* pIn, size_t sampleCount);

/* 将IEEE 32位浮点样本转换为有符号32位PCM样本的低级函数。 */
DRWAV_API void drwav_f32_to_s32(drwav_int32* pOut, const float* pIn, size_t sampleCount);

/* 将IEEE 64位浮点样本转换为有符号32位PCM样本的低级函数。 */
DRWAV_API void drwav_f64_to_s32(drwav_int32* pOut, const double* pIn, size_t sampleCount);

/* 将A-law样本转换为有符号32位PCM样本的低级函数。 */
DRWAV_API void drwav_alaw_to_s32(drwav_int32* pOut, const drwav_uint8* pIn, size_t sampleCount);

/* 将u-law样本转换为有符号32位PCM样本的低级函数。 */
DRWAV_API void drwav_mulaw_to_s32(drwav_int32* pOut, const drwav_uint8* pIn, size_t sampleCount);

#endif  /* DR_WAV_NO_CONVERSION_API */


/* 高级便利助手 */

#ifndef DR_WAV_NO_STDIO
/*
使用stdio初始化用于读取的波形文件的辅助函数。

这会保持内部的FILE对象，直到调用drwav_uninit()为止。如果您正在缓存drwav对象，请记住这一点，因为操作系统可能限制应用程序在任何给定时间内打开的文件句柄数量。
*/
DRWAV_API drwav_bool32 drwav_init_file(drwav* pWav, const char* filename, const drwav_allocation_callbacks* pAllocationCallbacks);
// 初始化一个 wave 文件用于读取，使用标准输入输出
// 参数：pWav - 指向 drwav 结构体的指针，用于存储 wave 文件的信息
// 参数：filename - 文件名
// 参数：onChunk - 指向处理 chunk 的回调函数的指针
// 参数：pChunkUserData - 传递给 onChunk 回调函数的用户数据指针
// 参数：flags - 标志位
// 参数：pAllocationCallbacks - 指向内存分配回调函数的指针
DRWAV_API drwav_bool32 drwav_init_file_ex(drwav* pWav, const char* filename, drwav_chunk_proc onChunk, void* pChunkUserData, drwav_uint32 flags, const drwav_allocation_callbacks* pAllocationCallbacks);

// 初始化一个 wave 文件用于读取，使用宽字符文件名
// 参数：pWav - 指向 drwav 结构体的指针，用于存储 wave 文件的信息
// 参数：filename - 宽字符文件名
// 参数：pAllocationCallbacks - 指向内存分配回调函数的指针
DRWAV_API drwav_bool32 drwav_init_file_w(drwav* pWav, const wchar_t* filename, const drwav_allocation_callbacks* pAllocationCallbacks);

// 初始化一个 wave 文件用于读取，使用宽字符文件名，并提供处理 chunk 的回调函数
// 参数：pWav - 指向 drwav 结构体的指针，用于存储 wave 文件的信息
// 参数：filename - 宽字符文件名
// 参数：onChunk - 指向处理 chunk 的回调函数的指针
// 参数：pChunkUserData - 传递给 onChunk 回调函数的用户数据指针
// 参数：flags - 标志位
// 参数：pAllocationCallbacks - 指向内存分配回调函数的指针
DRWAV_API drwav_bool32 drwav_init_file_ex_w(drwav* pWav, const wchar_t* filename, drwav_chunk_proc onChunk, void* pChunkUserData, drwav_uint32 flags, const drwav_allocation_callbacks* pAllocationCallbacks);

/*
用于使用标准输入输出初始化一个 wave 文件用于写入的辅助函数。

这会一直持有内部的 FILE 对象，直到调用 drwav_uninit()。如果你缓存了 drwav 对象，要记住这一点，因为操作系统可能限制应用程序在任何给定时间打开的文件句柄数量。
*/
// 初始化一个 wave 文件用于写入，使用标准输入输出
// 参数：pWav - 指向 drwav 结构体的指针，用于存储 wave 文件的信息
// 参数：filename - 文件名
// 参数：pFormat - 指向数据格式的指针
// 参数：pAllocationCallbacks - 指向内存分配回调函数的指针
DRWAV_API drwav_bool32 drwav_init_file_write(drwav* pWav, const char* filename, const drwav_data_format* pFormat, const drwav_allocation_callbacks* pAllocationCallbacks);

// 初始化一个 wave 文件用于写入，使用标准输入输出，按顺序写入
// 参数：pWav - 指向 drwav 结构体的指针，用于存储 wave 文件的信息
// 参数：filename - 文件名
// 参数：pFormat - 指向数据格式的指针
// 参数：totalSampleCount - 总采样数
// 参数：pAllocationCallbacks - 指向内存分配回调函数的指针
DRWAV_API drwav_bool32 drwav_init_file_write_sequential(drwav* pWav, const char* filename, const drwav_data_format* pFormat, drwav_uint64 totalSampleCount, const drwav_allocation_callbacks* pAllocationCallbacks);

// 初始化一个 wave 文件用于写入，使用标准输入输出，按顺序写入 PCM 帧
// 参数：pWav - 指向 drwav 结构体的指针，用于存储 wave 文件的信息
// 参数：filename - 文件名
// 参数：pFormat - 指向数据格式的指针
// 参数：totalPCMFrameCount - 总 PCM 帧数
// 参数：pAllocationCallbacks - 指向内存分配回调函数的指针
DRWAV_API drwav_bool32 drwav_init_file_write_sequential_pcm_frames(drwav* pWav, const char* filename, const drwav_data_format* pFormat, drwav_uint64 totalPCMFrameCount, const drwav_allocation_callbacks* pAllocationCallbacks);

// 初始化一个 wave 文件用于写入，使用宽字符文件名
// 参数：pWav - 指向 drwav 结构体的指针，用于存储 wave 文件的信息
// 参数：filename - 宽字符文件名
// 参数：pFormat - 指向数据格式的指针
// 参数：pAllocationCallbacks - 指向内存分配回调函数的指针
DRWAV_API drwav_bool32 drwav_init_file_write_w(drwav* pWav, const wchar_t* filename, const drwav_data_format* pFormat, const drwav_allocation_callbacks* pAllocationCallbacks);

// 初始化一个 wave 文件用于写入，使用宽字符文件名，按顺序写入
// 参数：pWav - 指向 drwav 结构体的指针，用于存储 wave 文件的信息
// 参数：filename - 宽字符文件名
// 参数：pFormat - 指向数据格式的指针
// 参数：totalSampleCount - 总采样数
// 参数：pAllocationCallbacks - 指向内存分配回调函数的指针
DRWAV_API drwav_bool32 drwav_init_file_write_sequential_w(drwav* pWav, const wchar_t* filename, const drwav_data_format* pFormat, drwav_uint64 totalSampleCount, const drwav_allocation_callbacks* pAllocationCallbacks);
/*
Helper for initializing a writer for sequential PCM frames to a file.

This function initializes a writer for writing sequential PCM frames to a file. It takes a pointer to a drwav object, the filename to write to, the format of the audio data, the total number of PCM frames to write, and optional allocation callbacks.

Parameters:
    pWav: A pointer to the drwav object to initialize.
    filename: The filename of the file to write to.
    pFormat: A pointer to a drwav_data_format object describing the format of the audio data.
    totalPCMFrameCount: The total number of PCM frames to write.
    pAllocationCallbacks: Optional allocation callbacks.

Returns:
    A boolean indicating the success of the initialization.
*/
DRWAV_API drwav_bool32 drwav_init_file_write_sequential_pcm_frames_w(drwav* pWav, const wchar_t* filename, const drwav_data_format* pFormat, drwav_uint64 totalPCMFrameCount, const drwav_allocation_callbacks* pAllocationCallbacks);
#endif  /* DR_WAV_NO_STDIO */

/*
Helper for initializing a loader from a pre-allocated memory buffer.

This function initializes a loader from a pre-allocated memory buffer. It does not create a copy of the data, so it is up to the application to ensure the buffer remains valid for the lifetime of the drwav object. The buffer should contain the contents of the entire wave file, not just the sample data.

Parameters:
    pWav: A pointer to the drwav object to initialize.
    data: A pointer to the pre-allocated memory buffer containing the entire wave file.
    dataSize: The size of the memory buffer in bytes.
    pAllocationCallbacks: Optional allocation callbacks.

Returns:
    A boolean indicating the success of the initialization.
*/
DRWAV_API drwav_bool32 drwav_init_memory(drwav* pWav, const void* data, size_t dataSize, const drwav_allocation_callbacks* pAllocationCallbacks);
DRWAV_API drwav_bool32 drwav_init_memory_ex(drwav* pWav, const void* data, size_t dataSize, drwav_chunk_proc onChunk, void* pChunkUserData, drwav_uint32 flags, const drwav_allocation_callbacks* pAllocationCallbacks);

/*
Helper for initializing a writer which outputs data to a memory buffer.

This function initializes a writer for writing audio data to a memory buffer. dr_wav will manage the memory allocations, but it is up to the caller to free the data with drwav_free(). The buffer will remain allocated even after drwav_uninit() is called. The buffer should not be considered valid until after drwav_uninit() has been called.

Parameters:
    pWav: A pointer to the drwav object to initialize.
    ppData: A pointer to a pointer to the memory buffer. On output, this will be set to point to the allocated memory buffer.
    pDataSize: A pointer to the size of the allocated memory buffer. On output, this will be set to the size of the allocated memory buffer.
    pFormat: A pointer to a drwav_data_format object describing the format of the audio data.
    pAllocationCallbacks: Optional allocation callbacks.

Returns:
    A boolean indicating the success of the initialization.
*/
DRWAV_API drwav_bool32 drwav_init_memory_write(drwav* pWav, void** ppData, size_t* pDataSize, const drwav_data_format* pFormat, const drwav_allocation_callbacks* pAllocationCallbacks);
DRWAV_API drwav_bool32 drwav_init_memory_write_sequential(drwav* pWav, void** ppData, size_t* pDataSize, const drwav_data_format* pFormat, drwav_uint64 totalSampleCount, const drwav_allocation_callbacks* pAllocationCallbacks);
DRWAV_API drwav_bool32 drwav_init_memory_write_sequential_pcm_frames(drwav* pWav, void** ppData, size_t* pDataSize, const drwav_data_format* pFormat, drwav_uint64 totalPCMFrameCount, const drwav_allocation_callbacks* pAllocationCallbacks);


#ifndef DR_WAV_NO_CONVERSION_API
/*
/*
以单个操作打开并读取整个 wav 文件。

返回值是一个包含音频数据的堆分配缓冲区。使用 drwav_free() 来释放缓冲区。
*/
DRWAV_API drwav_int16* drwav_open_and_read_pcm_frames_s16(drwav_read_proc onRead, drwav_seek_proc onSeek, void* pUserData, unsigned int* channelsOut, unsigned int* sampleRateOut, drwav_uint64* totalFrameCountOut, const drwav_allocation_callbacks* pAllocationCallbacks);
DRWAV_API float* drwav_open_and_read_pcm_frames_f32(drwav_read_proc onRead, drwav_seek_proc onSeek, void* pUserData, unsigned int* channelsOut, unsigned int* sampleRateOut, drwav_uint64* totalFrameCountOut, const drwav_allocation_callbacks* pAllocationCallbacks);
DRWAV_API drwav_int32* drwav_open_and_read_pcm_frames_s32(drwav_read_proc onRead, drwav_seek_proc onSeek, void* pUserData, unsigned int* channelsOut, unsigned int* sampleRateOut, drwav_uint64* totalFrameCountOut, const drwav_allocation_callbacks* pAllocationCallbacks);
#ifndef DR_WAV_NO_STDIO
/*
以单个操作打开并解码整个 wav 文件。

返回值是一个包含音频数据的堆分配缓冲区。使用 drwav_free() 来释放缓冲区。
*/
DRWAV_API drwav_int16* drwav_open_file_and_read_pcm_frames_s16(const char* filename, unsigned int* channelsOut, unsigned int* sampleRateOut, drwav_uint64* totalFrameCountOut, const drwav_allocation_callbacks* pAllocationCallbacks);
DRWAV_API float* drwav_open_file_and_read_pcm_frames_f32(const char* filename, unsigned int* channelsOut, unsigned int* sampleRateOut, drwav_uint64* totalFrameCountOut, const drwav_allocation_callbacks* pAllocationCallbacks);
DRWAV_API drwav_int32* drwav_open_file_and_read_pcm_frames_s32(const char* filename, unsigned int* channelsOut, unsigned int* sampleRateOut, drwav_uint64* totalFrameCountOut, const drwav_allocation_callbacks* pAllocationCallbacks);
// 从文件名打开并读取 PCM 帧，返回带有 int16 类型的数据指针
DRWAV_API drwav_int16* drwav_open_file_and_read_pcm_frames_s16_w(const wchar_t* filename, unsigned int* channelsOut, unsigned int* sampleRateOut, drwav_uint64* totalFrameCountOut, const drwav_allocation_callbacks* pAllocationCallbacks);

// 从文件名打开并读取 PCM 帧，返回带有 float 类型的数据指针
DRWAV_API float* drwav_open_file_and_read_pcm_frames_f32_w(const wchar_t* filename, unsigned int* channelsOut, unsigned int* sampleRateOut, drwav_uint64* totalFrameCountOut, const drwav_allocation_callbacks* pAllocationCallbacks);

// 从文件名打开并读取 PCM 帧，返回带有 int32 类型的数据指针
DRWAV_API drwav_int32* drwav_open_file_and_read_pcm_frames_s32_w(const wchar_t* filename, unsigned int* channelsOut, unsigned int* sampleRateOut, drwav_uint64* totalFrameCountOut, const drwav_allocation_callbacks* pAllocationCallbacks);
#endif

/*
在单个操作中从内存块中打开并解码整个 wav 文件。

返回值是一个堆分配的缓冲区，包含音频数据。使用 drwav_free() 释放缓冲区。
*/
DRWAV_API drwav_int16* drwav_open_memory_and_read_pcm_frames_s16(const void* data, size_t dataSize, unsigned int* channelsOut, unsigned int* sampleRateOut, drwav_uint64* totalFrameCountOut, const drwav_allocation_callbacks* pAllocationCallbacks);

// 从内存块中打开并读取 PCM 帧，返回带有 float 类型的数据指针
DRWAV_API float* drwav_open_memory_and_read_pcm_frames_f32(const void* data, size_t dataSize, unsigned int* channelsOut, unsigned int* sampleRateOut, drwav_uint64* totalFrameCountOut, const drwav_allocation_callbacks* pAllocationCallbacks);

// 从内存块中打开并读取 PCM 帧，返回带有 int32 类型的数据指针
DRWAV_API drwav_int32* drwav_open_memory_and_read_pcm_frames_s32(const void* data, size_t dataSize, unsigned int* channelsOut, unsigned int* sampleRateOut, drwav_uint64* totalFrameCountOut, const drwav_allocation_callbacks* pAllocationCallbacks);
#endif

// 释放由 dr_wav 内部分配的数据
DRWAV_API void drwav_free(void* p, const drwav_allocation_callbacks* pAllocationCallbacks);

// 将 wav 流中的字节转换为本机字节顺序的大小类型
DRWAV_API drwav_uint16 drwav_bytes_to_u16(const drwav_uint8* data);
/* 将字节数组转换为有符号16位整数 */
DRWAV_API drwav_int16 drwav_bytes_to_s16(const drwav_uint8* data);
/* 将字节数组转换为无符号32位整数 */
DRWAV_API drwav_uint32 drwav_bytes_to_u32(const drwav_uint8* data);
/* 将字节数组转换为有符号32位整数 */
DRWAV_API drwav_int32 drwav_bytes_to_s32(const drwav_uint8* data);
/* 将字节数组转换为无符号64位整数 */
DRWAV_API drwav_uint64 drwav_bytes_to_u64(const drwav_uint8* data);
/* 将字节数组转换为有符号64位整数 */
DRWAV_API drwav_int64 drwav_bytes_to_s64(const drwav_uint8* data);

/* 用于比较 GUID 以检查 Wave64 块类型 */
DRWAV_API drwav_bool32 drwav_guid_equal(const drwav_uint8 a[16], const drwav_uint8 b[16]);

/* 用于比较四字符代码以检查 RIFF 块类型 */
DRWAV_API drwav_bool32 drwav_fourcc_equal(const drwav_uint8* a, const char* b);

#ifdef __cplusplus
}
#endif
#endif  /* dr_wav_h */


/************************************************************************************************************************************************************
 ************************************************************************************************************************************************************

 IMPLEMENTATION

 ************************************************************************************************************************************************************
 ************************************************************************************************************************************************************/
#if defined(DR_WAV_IMPLEMENTATION) || defined(DRWAV_IMPLEMENTATION)
#ifndef dr_wav_c
#define dr_wav_c

#include <stdlib.h>
#include <string.h> /* For memcpy(), memset() */
#include <limits.h> /* For INT_MAX */

#ifndef DR_WAV_NO_STDIO
#include <stdio.h>
#include <wchar.h>
#endif

/* Standard library stuff. */
#ifndef DRWAV_ASSERT
#include <assert.h>
#define DRWAV_ASSERT(expression)           assert(expression)
#endif
#ifndef DRWAV_MALLOC
#define DRWAV_MALLOC(sz)                   malloc((sz))
#endif
#ifndef DRWAV_REALLOC
#define DRWAV_REALLOC(p, sz)               realloc((p), (sz))
#endif
#ifndef DRWAV_FREE
#define DRWAV_FREE(p)                      free((p))  # 定义释放内存的宏，使用 free() 函数
#endif
#ifndef DRWAV_COPY_MEMORY
#define DRWAV_COPY_MEMORY(dst, src, sz)    memcpy((dst), (src), (sz))  # 定义内存拷贝的宏，使用 memcpy() 函数
#endif
#ifndef DRWAV_ZERO_MEMORY
#define DRWAV_ZERO_MEMORY(p, sz)           memset((p), 0, (sz))  # 定义内存清零的宏，使用 memset() 函数
#endif
#ifndef DRWAV_ZERO_OBJECT
#define DRWAV_ZERO_OBJECT(p)               DRWAV_ZERO_MEMORY((p), sizeof(*p))  # 定义对象清零的宏，使用 DRWAV_ZERO_MEMORY() 宏
#endif

#define drwav_countof(x)                   (sizeof(x) / sizeof(x[0]))  # 定义计算数组元素个数的宏
#define drwav_align(x, a)                  ((((x) + (a) - 1) / (a)) * (a))  # 定义对齐操作的宏
#define drwav_min(a, b)                    (((a) < (b)) ? (a) : (b))  # 定义取最小值的宏
#define drwav_max(a, b)                    (((a) > (b)) ? (a) : (b))  # 定义取最大值的宏
#define drwav_clamp(x, lo, hi)             (drwav_max((lo), drwav_min((hi), (x)))  # 定义将值限制在指定范围内的宏

#define DRWAV_MAX_SIMD_VECTOR_SIZE         64  /* 64 for AVX-512 in the future. */  # 定义最大 SIMD 向量大小

/* CPU architecture. */
#if defined(__x86_64__) || defined(_M_X64)
    #define DRWAV_X64  # 定义 x86-64 架构
#elif defined(__i386) || defined(_M_IX86)
    #define DRWAV_X86  # 定义 x86 架构
#elif defined(__arm__) || defined(_M_ARM)
    #define DRWAV_ARM  # 定义 ARM 架构
#endif

#ifdef _MSC_VER
    #define DRWAV_INLINE __forceinline  # 定义 MSC 编译器下的内联函数
#elif defined(__GNUC__)
    /*
    I've had a bug report where GCC is emitting warnings about functions possibly not being inlineable. This warning happens when
    the __attribute__((always_inline)) attribute is defined without an "inline" statement. I think therefore there must be some
    case where "__inline__" is not always defined, thus the compiler emitting these warnings. When using -std=c89 or -ansi on the
    command line, we cannot use the "inline" keyword and instead need to use "__inline__". In an attempt to work around this issue
    I am using "__inline__" only when we're compiling in strict ANSI mode.
    */
    #if defined(__STRICT_ANSI__)
        #define DRWAV_INLINE __inline__ __attribute__((always_inline))  # 定义严格 ANSI 模式下的内联函数
    #else
        #define DRWAV_INLINE inline __attribute__((always_inline))  # 定义内联函数
    #endif
#elif defined(__WATCOMC__)
    #define DRWAV_INLINE __inline  # 定义 Watcom 编译器下的内联函数
#else
    # 定义一个宏，用于在编译时将函数体直接插入到调用处
    #define DRWAV_INLINE
#endif

#if defined(SIZE_MAX)
    #define DRWAV_SIZE_MAX  SIZE_MAX
#else
    #if defined(_WIN64) || defined(_LP64) || defined(__LP64__)
        #define DRWAV_SIZE_MAX  ((drwav_uint64)0xFFFFFFFFFFFFFFFF)
    #else
        #define DRWAV_SIZE_MAX  0xFFFFFFFF
    #endif
#endif

#if defined(_MSC_VER) && _MSC_VER >= 1400
    #define DRWAV_HAS_BYTESWAP16_INTRINSIC
    #define DRWAV_HAS_BYTESWAP32_INTRINSIC
    #define DRWAV_HAS_BYTESWAP64_INTRINSIC
#elif defined(__clang__)
    #if defined(__has_builtin)
        #if __has_builtin(__builtin_bswap16)
            #define DRWAV_HAS_BYTESWAP16_INTRINSIC
        #endif
        #if __has_builtin(__builtin_bswap32)
            #define DRWAV_HAS_BYTESWAP32_INTRINSIC
        #endif
        #if __has_builtin(__builtin_bswap64)
            #define DRWAV_HAS_BYTESWAP64_INTRINSIC
        #endif
    #endif
#elif defined(__GNUC__)
    #if ((__GNUC__ > 4) || (__GNUC__ == 4 && __GNUC_MINOR__ >= 3))
        #define DRWAV_HAS_BYTESWAP32_INTRINSIC
        #define DRWAV_HAS_BYTESWAP64_INTRINSIC
    #endif
    #if ((__GNUC__ > 4) || (__GNUC__ == 4 && __GNUC_MINOR__ >= 8))
        #define DRWAV_HAS_BYTESWAP16_INTRINSIC
    #endif
#endif

// 获取 drwav 版本号
DRWAV_API void drwav_version(drwav_uint32* pMajor, drwav_uint32* pMinor, drwav_uint32* pRevision)
{
    if (pMajor) {
        *pMajor = DRWAV_VERSION_MAJOR;
    }

    if (pMinor) {
        *pMinor = DRWAV_VERSION_MINOR;
    }

    if (pRevision) {
        *pRevision = DRWAV_VERSION_REVISION;
    }
}

// 获取 drwav 版本号字符串
DRWAV_API const char* drwav_version_string(void)
{
    return DRWAV_VERSION_STRING;
}

/*
这些限制用于初始化解码器时的基本验证。如果超出这些限制，首先：你到底在干什么？！（告诉我，我会很好奇！）其次，你可以在 dr_wav 实现之前通过 #define 调整这些限制。
*/
#ifndef DRWAV_MAX_SAMPLE_RATE
#define DRWAV_MAX_SAMPLE_RATE       384000
#endif
#ifndef DRWAV_MAX_CHANNELS
#define DRWAV_MAX_CHANNELS          256
#endif
#ifndef DRWAV_MAX_BITS_PER_SAMPLE
#define DRWAV_MAX_BITS_PER_SAMPLE   64
#endif

// 如果未定义 DRWAV_MAX_BITS_PER_SAMPLE，则定义为 64


static const drwav_uint8 drwavGUID_W64_RIFF[16] = {0x72,0x69,0x66,0x66, 0x2E,0x91, 0xCF,0x11, 0xA5,0xD6, 0x28,0xDB,0x04,0xC1,0x00,0x00};    /* 66666972-912E-11CF-A5D6-28DB04C10000 */
static const drwav_uint8 drwavGUID_W64_WAVE[16] = {0x77,0x61,0x76,0x65, 0xF3,0xAC, 0xD3,0x11, 0x8C,0xD1, 0x00,0xC0,0x4F,0x8E,0xDB,0x8A};    /* 65766177-ACF3-11D3-8CD1-00C04F8EDB8A */
// 定义了两个常量数组，分别表示 W64 格式的 RIFF 和 WAVE

/*static const drwav_uint8 drwavGUID_W64_JUNK[16] = {0x6A,0x75,0x6E,0x6B, 0xF3,0xAC, 0xD3,0x11, 0x8C,0xD1, 0x00,0xC0,0x4F,0x8E,0xDB,0x8A};*/    /* 6B6E756A-ACF3-11D3-8CD1-00C04F8EDB8A */
// 注释掉的常量数组

static const drwav_uint8 drwavGUID_W64_FMT [16] = {0x66,0x6D,0x74,0x20, 0xF3,0xAC, 0xD3,0x11, 0x8C,0xD1, 0x00,0xC0,0x4F,0x8E,0xDB,0x8A};    /* 20746D66-ACF3-11D3-8CD1-00C04F8EDB8A */
static const drwav_uint8 drwavGUID_W64_FACT[16] = {0x66,0x61,0x63,0x74, 0xF3,0xAC, 0xD3,0x11, 0x8C,0xD1, 0x00,0xC0,0x4F,0x8E,0xDB,0x8A};    /* 74636166-ACF3-11D3-8CD1-00C04F8EDB8A */
static const drwav_uint8 drwavGUID_W64_DATA[16] = {0x64,0x61,0x74,0x61, 0xF3,0xAC, 0xD3,0x11, 0x8C,0xD1, 0x00,0xC0,0x4F,0x8E,0xDB,0x8A};    /* 61746164-ACF3-11D3-8CD1-00C04F8EDB8A */
static const drwav_uint8 drwavGUID_W64_SMPL[16] = {0x73,0x6D,0x70,0x6C, 0xF3,0xAC, 0xD3,0x11, 0x8C,0xD1, 0x00,0xC0,0x4F,0x8E,0xDB,0x8A};    /* 6C706D73-ACF3-11D3-8CD1-00C04F8EDB8A */
// 定义了多个常量数组，分别表示 W64 格式的 FMT、FACT、DATA 和 SMPL

static DRWAV_INLINE drwav_bool32 drwav__guid_equal(const drwav_uint8 a[16], const drwav_uint8 b[16])
{
    int i;
    for (i = 0; i < 16; i += 1) {
        if (a[i] != b[i]) {
            return DRWAV_FALSE;
        }
    }
    return DRWAV_TRUE;
}
// 定义了一个内联函数，用于比较两个 GUID 是否相等

static DRWAV_INLINE drwav_bool32 drwav__fourcc_equal(const drwav_uint8* a, const char* b)
{
    return
        a[0] == b[0] &&
        a[1] == b[1] &&
        a[2] == b[2] &&
        a[3] == b[3];
}
// 定义了一个内联函数，用于比较两个 FourCC 是否相等

static DRWAV_INLINE int drwav__is_little_endian(void)
{
#if defined(DRWAV_X86) || defined(DRWAV_X64)
    return DRWAV_TRUE;

// 如果定义了 DRWAV_X86 或 DRWAV_X64，则返回 DRWAV_TRUE
#elif defined(__BYTE_ORDER) && defined(__LITTLE_ENDIAN) && __BYTE_ORDER == __LITTLE_ENDIAN
    # 如果是小端字节序，返回真
    return DRWAV_TRUE;
#else
    # 否则，创建一个整数 n 并赋值为 1
    int n = 1;
    # 返回 n 的第一个字节是否为 1
    return (*(char*)&n) == 1;
#endif
}

# 将 2 字节的数据转换为 16 位无符号整数
static DRWAV_INLINE drwav_uint16 drwav__bytes_to_u16(const drwav_uint8* data)
{
    return (data[0] << 0) | (data[1] << 8);
}

# 将 2 字节的数据转换为 16 位有符号整数
static DRWAV_INLINE drwav_int16 drwav__bytes_to_s16(const drwav_uint8* data)
{
    return (short)drwav__bytes_to_u16(data);
}

# 将 4 字节的数据转换为 32 位无符号整数
static DRWAV_INLINE drwav_uint32 drwav__bytes_to_u32(const drwav_uint8* data)
{
    return (data[0] << 0) | (data[1] << 8) | (data[2] << 16) | (data[3] << 24);
}

# 将 4 字节的数据转换为 32 位有符号整数
static DRWAV_INLINE drwav_int32 drwav__bytes_to_s32(const drwav_uint8* data)
{
    return (drwav_int32)drwav__bytes_to_u32(data);
}

# 将 8 字节的数据转换为 64 位无符号整数
static DRWAV_INLINE drwav_uint64 drwav__bytes_to_u64(const drwav_uint8* data)
{
    return
        ((drwav_uint64)data[0] <<  0) | ((drwav_uint64)data[1] <<  8) | ((drwav_uint64)data[2] << 16) | ((drwav_uint64)data[3] << 24) |
        ((drwav_uint64)data[4] << 32) | ((drwav_uint64)data[5] << 40) | ((drwav_uint64)data[6] << 48) | ((drwav_uint64)data[7] << 56);
}

# 将 8 字节的数据转换为 64 位有符号整数
static DRWAV_INLINE drwav_int64 drwav__bytes_to_s64(const drwav_uint8* data)
{
    return (drwav_int64)drwav__bytes_to_u64(data);
}

# 将 16 字节的数据转换为 GUID
static DRWAV_INLINE void drwav__bytes_to_guid(const drwav_uint8* data, drwav_uint8* guid)
{
    int i;
    for (i = 0; i < 16; ++i) {
        guid[i] = data[i];
    }
}

# 将 16 位无符号整数进行字节交换
static DRWAV_INLINE drwav_uint16 drwav__bswap16(drwav_uint16 n)
{
#ifdef DRWAV_HAS_BYTESWAP16_INTRINSIC
    #if defined(_MSC_VER)
        return _byteswap_ushort(n);
    #elif defined(__GNUC__) || defined(__clang__)
        return __builtin_bswap16(n);
    #else
        #error "This compiler does not support the byte swap intrinsic."
    #endif
#else
    return ((n & 0xFF00) >> 8) |
           ((n & 0x00FF) << 8);
#endif
}

# 将 32 位无符号整数进行字节交换
static DRWAV_INLINE drwav_uint32 drwav__bswap32(drwav_uint32 n)
{
#ifdef DRWAV_HAS_BYTESWAP32_INTRINSIC
    #if defined(_MSC_VER)
        return _byteswap_ulong(n);
    #elif defined(__GNUC__) || defined(__clang__)  /* 如果是使用 GCC 或者 Clang 编译器 */
        #if defined(DRWAV_ARM) && (defined(__ARM_ARCH) && __ARM_ARCH >= 6) && !defined(DRWAV_64BIT)   /* 如果是 ARM 架构并且版本大于等于 6，并且不是 64 位 */
            /* 用于 ARM 架构的内联汇编优化实现。在我的测试中，GCC 在使用 __builtin_bswap32() 时没有生成优化的代码。 */
            drwav_uint32 r;  /* 定义一个无符号 32 位整数 r */
            __asm__ __volatile__ (  /* 内联汇编代码块开始 */
            #if defined(DRWAV_64BIT)
                "rev %w[out], %w[in]" : [out]"=r"(r) : [in]"r"(n)   /* <-- 这部分代码未经测试。如果社区中有人能够测试这部分代码，将不胜感激！ */
            #else
                "rev %[out], %[in]" : [out]"=r"(r) : [in]"r"(n)  /* 使用内联汇编执行反转操作 */
            #endif
            );
            return r;  /* 返回反转后的结果 */
        #else
            return __builtin_bswap32(n);  /* 如果不是 ARM 架构或者版本小于 6，则使用内置的字节交换函数 */
        #endif
    #else
        #error "This compiler does not support the byte swap intrinsic."  /* 如果不是使用支持的编译器，则报错 */
    #endif
#else
    return ((n & 0xFF000000) >> 24) |  // 将 n 的高 8 位右移 24 位
           ((n & 0x00FF0000) >>  8) |  // 将 n 的次高 8 位右移 8 位
           ((n & 0x0000FF00) <<  8) |  // 将 n 的次低 8 位左移 8 位
           ((n & 0x000000FF) << 24);   // 将 n 的低 8 位左移 24 位
#endif
}

static DRWAV_INLINE drwav_uint64 drwav__bswap64(drwav_uint64 n)
{
#ifdef DRWAV_HAS_BYTESWAP64_INTRINSIC
    #if defined(_MSC_VER)
        return _byteswap_uint64(n);    // 使用内置函数交换 n 的字节顺序
    #elif defined(__GNUC__) || defined(__clang__)
        return __builtin_bswap64(n);    // 使用内置函数交换 n 的字节顺序
    #else
        #error "This compiler does not support the byte swap intrinsic."  // 报错信息，提示编译器不支持字节交换操作
    #endif
#else
    /* Weird "<< 32" bitshift is required for C89 because it doesn't support 64-bit constants. Should be optimized out by a good compiler. */
    return ((n & ((drwav_uint64)0xFF000000 << 32)) >> 56) |   // 将 n 的高 8 位左移 32 位，然后右移 56 位
           ((n & ((drwav_uint64)0x00FF0000 << 32)) >> 40) |   // 将 n 的次高 8 位左移 32 位，然后右移 40 位
           ((n & ((drwav_uint64)0x0000FF00 << 32)) >> 24) |   // 将 n 的次低 8 位左移 32 位，然后右移 24 位
           ((n & ((drwav_uint64)0x000000FF << 32)) >>  8) |   // 将 n 的低 8 位左移 32 位，然后右移 8 位
           ((n & ((drwav_uint64)0xFF000000      )) <<  8) |   // 将 n 的高 8 位左移 8 位
           ((n & ((drwav_uint64)0x00FF0000      )) << 24) |   // 将 n 的次高 8 位左移 24 位
           ((n & ((drwav_uint64)0x0000FF00      )) << 40) |   // 将 n 的次低 8 位左移 40 位
           ((n & ((drwav_uint64)0x000000FF      )) << 56);    // 将 n 的低 8 位左移 56 位
#endif
}


static DRWAV_INLINE drwav_int16 drwav__bswap_s16(drwav_int16 n)
{
    return (drwav_int16)drwav__bswap16((drwav_uint16)n);  // 调用 drwav__bswap16 函数交换 n 的字节顺序，并转换为 int16 类型
}

static DRWAV_INLINE void drwav__bswap_samples_s16(drwav_int16* pSamples, drwav_uint64 sampleCount)
{
    drwav_uint64 iSample;
    for (iSample = 0; iSample < sampleCount; iSample += 1) {
        pSamples[iSample] = drwav__bswap_s16(pSamples[iSample]);  // 遍历采样数据，调用 drwav__bswap_s16 函数交换每个采样数据的字节顺序
    }
}


static DRWAV_INLINE void drwav__bswap_s24(drwav_uint8* p)
{
    drwav_uint8 t;
    t = p[0];
    p[0] = p[2];
    p[2] = t;
}

static DRWAV_INLINE void drwav__bswap_samples_s24(drwav_uint8* pSamples, drwav_uint64 sampleCount)
{
    drwav_uint64 iSample;
    for (iSample = 0; iSample < sampleCount; iSample += 1) {
        drwav_uint8* pSample = pSamples + (iSample*3);
        drwav__bswap_s24(pSample);  // 遍历采样数据，调用 drwav__bswap_s24 函数交换每个采样数据的字节顺序
    }
}
static DRWAV_INLINE drwav_int32 drwav__bswap_s32(drwav_int32 n)
{
    // 交换32位有符号整数的字节顺序
    return (drwav_int32)drwav__bswap32((drwav_uint32)n);
}

static DRWAV_INLINE void drwav__bswap_samples_s32(drwav_int32* pSamples, drwav_uint64 sampleCount)
{
    drwav_uint64 iSample;
    for (iSample = 0; iSample < sampleCount; iSample += 1) {
        // 对32位有符号整数样本数组中的每个样本进行字节顺序交换
        pSamples[iSample] = drwav__bswap_s32(pSamples[iSample]);
    }
}


static DRWAV_INLINE float drwav__bswap_f32(float n)
{
    union {
        drwav_uint32 i;
        float f;
    } x;
    x.f = n;
    // 交换32位浮点数的字节顺序
    x.i = drwav__bswap32(x.i);

    return x.f;
}

static DRWAV_INLINE void drwav__bswap_samples_f32(float* pSamples, drwav_uint64 sampleCount)
{
    drwav_uint64 iSample;
    for (iSample = 0; iSample < sampleCount; iSample += 1) {
        // 对32位浮点数样本数组中的每个样本进行字节顺序交换
        pSamples[iSample] = drwav__bswap_f32(pSamples[iSample]);
    }
}


static DRWAV_INLINE double drwav__bswap_f64(double n)
{
    union {
        drwav_uint64 i;
        double f;
    } x;
    x.f = n;
    // 交换64位浮点数的字节顺序
    x.i = drwav__bswap64(x.i);

    return x.f;
}

static DRWAV_INLINE void drwav__bswap_samples_f64(double* pSamples, drwav_uint64 sampleCount)
{
    drwav_uint64 iSample;
    for (iSample = 0; iSample < sampleCount; iSample += 1) {
        // 对64位浮点数样本数组中的每个样本进行字节顺序交换
        pSamples[iSample] = drwav__bswap_f64(pSamples[iSample]);
    }
}


static DRWAV_INLINE void drwav__bswap_samples_pcm(void* pSamples, drwav_uint64 sampleCount, drwav_uint32 bytesPerSample)
{
    /* Assumes integer PCM. Floating point PCM is done in drwav__bswap_samples_ieee(). */
    // 假设是整数PCM。浮点PCM在drwav__bswap_samples_ieee()中处理。
    switch (bytesPerSample)
    {
        case 2: /* s16, s12 (loosely packed) */
        {
            // 如果是 s16 或 s12 格式，调用函数将样本数据进行字节交换
            drwav__bswap_samples_s16((drwav_int16*)pSamples, sampleCount);
        } break;
        case 3: /* s24 */
        {
            // 如果是 s24 格式，调用函数将样本数据进行字节交换
            drwav__bswap_samples_s24((drwav_uint8*)pSamples, sampleCount);
        } break;
        case 4: /* s32 */
        {
            // 如果是 s32 格式，调用函数将样本数据进行字节交换
            drwav__bswap_samples_s32((drwav_int32*)pSamples, sampleCount);
        } break;
        default:
        {
            // 如果是其他格式，输出不支持的格式的错误信息
            /* Unsupported format. */
            DRWAV_ASSERT(DRWAV_FALSE);
        } break;
    }
}

static DRWAV_INLINE void drwav__bswap_samples_ieee(void* pSamples, drwav_uint64 sampleCount, drwav_uint32 bytesPerSample)
{
    switch (bytesPerSample)
    {
    #if 0   /* Contributions welcome for f16 support. */
        case 2: /* f16 */
        {
            // 如果是 2 字节的 f16 格式，调用 drwav__bswap_samples_f16 函数
            drwav__bswap_samples_f16((drwav_float16*)pSamples, sampleCount);
        } break;
    #endif
        case 4: /* f32 */
        {
            // 如果是 4 字节的 f32 格式，调用 drwav__bswap_samples_f32 函数
            drwav__bswap_samples_f32((float*)pSamples, sampleCount);
        } break;
        case 8: /* f64 */
        {
            // 如果是 8 字节的 f64 格式，调用 drwav__bswap_samples_f64 函数
            drwav__bswap_samples_f64((double*)pSamples, sampleCount);
        } break;
        default:
        {
            // 不支持的格式
            /* Unsupported format. */
            DRWAV_ASSERT(DRWAV_FALSE);
        } break;
    }
}

static DRWAV_INLINE void drwav__bswap_samples(void* pSamples, drwav_uint64 sampleCount, drwav_uint32 bytesPerSample, drwav_uint16 format)
{
    switch (format)
    {
        case DR_WAVE_FORMAT_PCM:
        {
            // 如果是 PCM 格式，调用 drwav__bswap_samples_pcm 函数
            drwav__bswap_samples_pcm(pSamples, sampleCount, bytesPerSample);
        } break;

        case DR_WAVE_FORMAT_IEEE_FLOAT:
        {
            // 如果是 IEEE_FLOAT 格式，调用 drwav__bswap_samples_ieee 函数
            drwav__bswap_samples_ieee(pSamples, sampleCount, bytesPerSample);
        } break;

        case DR_WAVE_FORMAT_ALAW:
        case DR_WAVE_FORMAT_MULAW:
        {
            // 如果是 ALAW 或 MULAW 格式，调用 drwav__bswap_samples_s16 函数
            drwav__bswap_samples_s16((drwav_int16*)pSamples, sampleCount);
        } break;

        case DR_WAVE_FORMAT_ADPCM:
        case DR_WAVE_FORMAT_DVI_ADPCM:
        default:
        {
            // 不支持的格式
            /* Unsupported format. */
            DRWAV_ASSERT(DRWAV_FALSE);
        } break;
    }
}


static void* drwav__malloc_default(size_t sz, void* pUserData)
{
    (void)pUserData;
    // 默认的内存分配函数
    return DRWAV_MALLOC(sz);
}

static void* drwav__realloc_default(void* p, size_t sz, void* pUserData)
{
    (void)pUserData;
    // 默认的内存重新分配函数
    return DRWAV_REALLOC(p, sz);
}

static void drwav__free_default(void* p, void* pUserData)
{
    (void)pUserData;
    // 默认的内存释放函数
    DRWAV_FREE(p);
}
static void* drwav__malloc_from_callbacks(size_t sz, const drwav_allocation_callbacks* pAllocationCallbacks)
{
    // 如果分配回调为空，则返回空指针
    if (pAllocationCallbacks == NULL) {
        return NULL;
    }

    // 如果分配回调中的 onMalloc 函数不为空，则调用该函数进行内存分配
    if (pAllocationCallbacks->onMalloc != NULL) {
        return pAllocationCallbacks->onMalloc(sz, pAllocationCallbacks->pUserData);
    }

    /* 尝试使用 realloc()。*/
    // 如果分配回调中的 onRealloc 函数不为空，则调用该函数进行内存重新分配
    if (pAllocationCallbacks->onRealloc != NULL) {
        return pAllocationCallbacks->onRealloc(NULL, sz, pAllocationCallbacks->pUserData);
    }

    // 如果以上条件都不满足，则返回空指针
    return NULL;
}

static void* drwav__realloc_from_callbacks(void* p, size_t szNew, size_t szOld, const drwav_allocation_callbacks* pAllocationCallbacks)
{
    // 如果分配回调为空，则返回空指针
    if (pAllocationCallbacks == NULL) {
        return NULL;
    }

    // 如果分配回调中的 onRealloc 函数不为空，则调用该函数进行内存重新分配
    if (pAllocationCallbacks->onRealloc != NULL) {
        return pAllocationCallbacks->onRealloc(p, szNew, pAllocationCallbacks->pUserData);
    }

    /* 尝试模拟 realloc() 以使用 malloc()/free()。*/
    // 如果分配回调中的 onMalloc 和 onFree 函数都不为空，则使用 malloc() 分配新内存，并释放旧内存
    if (pAllocationCallbacks->onMalloc != NULL && pAllocationCallbacks->onFree != NULL) {
        void* p2;

        p2 = pAllocationCallbacks->onMalloc(szNew, pAllocationCallbacks->pUserData);
        if (p2 == NULL) {
            return NULL;
        }

        if (p != NULL) {
            DRWAV_COPY_MEMORY(p2, p, szOld);
            pAllocationCallbacks->onFree(p, pAllocationCallbacks->pUserData);
        }

        return p2;
    }

    // 如果以上条件都不满足，则返回空指针
    return NULL;
}

static void drwav__free_from_callbacks(void* p, const drwav_allocation_callbacks* pAllocationCallbacks)
{
    // 如果指针为空或分配回调为空，则直接返回
    if (p == NULL || pAllocationCallbacks == NULL) {
        return;
    }

    // 如果分配回调中的 onFree 函数不为空，则调用该函数释放内存
    if (pAllocationCallbacks->onFree != NULL) {
        pAllocationCallbacks->onFree(p, pAllocationCallbacks->pUserData);
    }
}

static drwav_allocation_callbacks drwav_copy_allocation_callbacks_or_defaults(const drwav_allocation_callbacks* pAllocationCallbacks)
{
    // 如果分配回调不为空，则进行复制
    if (pAllocationCallbacks != NULL) {
        /* 复制。*/
        return *pAllocationCallbacks;
    }
    } else {
        /* 如果条件不成立，使用默认值。 */
        // 创建一个包含默认值的内存分配回调结构体
        drwav_allocation_callbacks allocationCallbacks;
        // 设置用户数据为空
        allocationCallbacks.pUserData = NULL;
        // 设置内存分配函数为默认的 malloc 函数
        allocationCallbacks.onMalloc  = drwav__malloc_default;
        // 设置内存重新分配函数为默认的 realloc 函数
        allocationCallbacks.onRealloc = drwav__realloc_default;
        // 设置内存释放函数为默认的 free 函数
        allocationCallbacks.onFree    = drwav__free_default;
        // 返回内存分配回调结构体
        return allocationCallbacks;
    }
}

// 检查给定的格式标签是否为压缩格式
static DRWAV_INLINE drwav_bool32 drwav__is_compressed_format_tag(drwav_uint16 formatTag)
{
    return
        formatTag == DR_WAVE_FORMAT_ADPCM ||
        formatTag == DR_WAVE_FORMAT_DVI_ADPCM;
}

// 计算 RIFF 格式的块的填充大小
static unsigned int drwav__chunk_padding_size_riff(drwav_uint64 chunkSize)
{
    return (unsigned int)(chunkSize % 2);
}

// 计算 W64 格式的块的填充大小
static unsigned int drwav__chunk_padding_size_w64(drwav_uint64 chunkSize)
{
    return (unsigned int)(chunkSize % 8);
}

// 读取 MSADPCM 格式的 PCM 帧
static drwav_uint64 drwav_read_pcm_frames_s16__msadpcm(drwav* pWav, drwav_uint64 samplesToRead, drwav_int16* pBufferOut);

// 读取 IMA 格式的 PCM 帧
static drwav_uint64 drwav_read_pcm_frames_s16__ima(drwav* pWav, drwav_uint64 samplesToRead, drwav_int16* pBufferOut);

// 内部函数，初始化写入 WAV 文件
static drwav_bool32 drwav_init_write__internal(drwav* pWav, const drwav_data_format* pFormat, drwav_uint64 totalSampleCount);

// 读取块头信息
static drwav_result drwav__read_chunk_header(drwav_read_proc onRead, void* pUserData, drwav_container container, drwav_uint64* pRunningBytesReadOut, drwav_chunk_header* pHeaderOut)
{
    if (container == drwav_container_riff || container == drwav_container_rf64) {
        drwav_uint8 sizeInBytes[4];

        // 读取块的 FourCC 标识符
        if (onRead(pUserData, pHeaderOut->id.fourcc, 4) != 4) {
            return DRWAV_AT_END;
        }

        // 读取块的大小
        if (onRead(pUserData, sizeInBytes, 4) != 4) {
            return DRWAV_INVALID_FILE;
        }

        pHeaderOut->sizeInBytes = drwav__bytes_to_u32(sizeInBytes);
        pHeaderOut->paddingSize = drwav__chunk_padding_size_riff(pHeaderOut->sizeInBytes);
        *pRunningBytesReadOut += 8;
    } else {
        // 读取 8 个字节的数据到 sizeInBytes 数组中
        drwav_uint8 sizeInBytes[8];

        // 读取 16 个字节的数据到 pHeaderOut->id.guid 中
        if (onRead(pUserData, pHeaderOut->id.guid, 16) != 16) {
            // 如果读取的字节数不等于 16，则返回 DRWAV_AT_END
            return DRWAV_AT_END;
        }

        // 读取 8 个字节的数据到 sizeInBytes 数组中
        if (onRead(pUserData, sizeInBytes, 8) != 8) {
            // 如果读取的字节数不等于 8，则返回 DRWAV_INVALID_FILE
            return DRWAV_INVALID_FILE;
        }

        // 计算 sizeInBytes 转换成无符号 64 位整数后减去 24，赋值给 pHeaderOut->sizeInBytes
        pHeaderOut->sizeInBytes = drwav__bytes_to_u64(sizeInBytes) - 24;    /* <-- Subtract 24 because w64 includes the size of the header. */
        // 计算填充大小并赋值给 pHeaderOut->paddingSize
        pHeaderOut->paddingSize = drwav__chunk_padding_size_w64(pHeaderOut->sizeInBytes);
        // 将已读取的字节数增加 24
        *pRunningBytesReadOut += 24;
    }

    // 返回成功
    return DRWAV_SUCCESS;
    /* 从当前位置向前定位 */
static drwav_bool32 drwav__seek_forward(drwav_seek_proc onSeek, drwav_uint64 offset, void* pUserData)
{
    /* 剩余需要定位的字节数 */
    drwav_uint64 bytesRemainingToSeek = offset;
    /* 当还有需要定位的字节数时循环 */
    while (bytesRemainingToSeek > 0) {
        /* 如果剩余需要定位的字节数大于 0x7FFFFFFF */
        if (bytesRemainingToSeek > 0x7FFFFFFF) {
            /* 调用 onSeek 函数向前定位 0x7FFFFFFF 个字节 */
            if (!onSeek(pUserData, 0x7FFFFFFF, drwav_seek_origin_current)) {
                return DRWAV_FALSE;
            }
            /* 更新剩余需要定位的字节数 */
            bytesRemainingToSeek -= 0x7FFFFFFF;
        } else {
            /* 如果剩余需要定位的字节数小于等于 0x7FFFFFFF */
            /* 调用 onSeek 函数向前定位剩余的字节数 */
            if (!onSeek(pUserData, (int)bytesRemainingToSeek, drwav_seek_origin_current)) {
                return DRWAV_FALSE;
            }
            /* 将剩余需要定位的字节数置为 0 */
            bytesRemainingToSeek = 0;
        }
    }

    return DRWAV_TRUE;
}

/* 从文件开始位置向前定位 */
static drwav_bool32 drwav__seek_from_start(drwav_seek_proc onSeek, drwav_uint64 offset, void* pUserData)
{
    /* 如果偏移量小于等于 0x7FFFFFFF */
    if (offset <= 0x7FFFFFFF) {
        /* 调用 onSeek 函数从文件开始位置向前定位偏移量个字节 */
        return onSeek(pUserData, (int)offset, drwav_seek_origin_start);
    }

    /* 如果偏移量大于 0x7FFFFFFF */
    /* 调用 onSeek 函数从文件开始位置向前定位 0x7FFFFFFF 个字节 */
    if (!onSeek(pUserData, 0x7FFFFFFF, drwav_seek_origin_start)) {
        return DRWAV_FALSE;
    }
    /* 更新偏移量 */
    offset -= 0x7FFFFFFF;

    /* 循环向前定位 */
    for (;;) {
        /* 如果偏移量小于等于 0x7FFFFFFF */
        if (offset <= 0x7FFFFFFF) {
            /* 调用 onSeek 函数从当前位置向前定位偏移量个字节 */
            return onSeek(pUserData, (int)offset, drwav_seek_origin_current);
        }

        /* 如果偏移量大于 0x7FFFFFFF */
        /* 调用 onSeek 函数从当前位置向前定位 0x7FFFFFFF 个字节 */
        if (!onSeek(pUserData, 0x7FFFFFFF, drwav_seek_origin_current)) {
            return DRWAV_FALSE;
        }
        /* 更新偏移量 */
        offset -= 0x7FFFFFFF;
    }

    /* 不应该执行到这里 */
    /*return DRWAV_TRUE; */
}

/* 读取 fmt chunk */
static drwav_bool32 drwav__read_fmt(drwav_read_proc onRead, drwav_seek_proc onSeek, void* pUserData, drwav_container container, drwav_uint64* pRunningBytesReadOut, drwav_fmt* fmtOut)
{
    drwav_chunk_header header;
    drwav_uint8 fmt[16];

    /* 读取 chunk header */
    if (drwav__read_chunk_header(onRead, pUserData, container, pRunningBytesReadOut, &header) != DRWAV_SUCCESS) {
        return DRWAV_FALSE;
    }

    /* 跳过非 fmt 类型的 chunk */
    # 当容器为 RIFF 或 RF64 且不是 "fmt "，或者容器为 W64 且不是 W64_FMT 时，执行循环
    while (((container == drwav_container_riff || container == drwav_container_rf64) && !drwav__fourcc_equal(header.id.fourcc, "fmt ")) || (container == drwav_container_w64 && !drwav__guid_equal(header.id.guid, drwavGUID_W64_FMT))) {
        # 如果无法向前跳过指定大小的数据块，返回 DRWAV_FALSE
        if (!drwav__seek_forward(onSeek, header.sizeInBytes + header.paddingSize, pUserData)) {
            return DRWAV_FALSE;
        }
        # 更新已读取的字节数
        *pRunningBytesReadOut += header.sizeInBytes + header.paddingSize;

        /* 尝试读取下一个数据块的头部信息 */
        if (drwav__read_chunk_header(onRead, pUserData, container, pRunningBytesReadOut, &header) != DRWAV_SUCCESS) {
            return DRWAV_FALSE;
        }
    }


    /* 验证 */
    # 如果容器为 RIFF 或 RF64，则验证头部信息的 fourcc 是否为 "fmt "
    if (container == drwav_container_riff || container == drwav_container_rf64) {
        if (!drwav__fourcc_equal(header.id.fourcc, "fmt ")) {
            return DRWAV_FALSE;
        }
    } else {
        # 如果容器为 W64，则验证头部信息的 guid 是否为 W64_FMT
        if (!drwav__guid_equal(header.id.guid, drwavGUID_W64_FMT)) {
            return DRWAV_FALSE;
        }
    }


    # 读取 fmt 数据块的内容
    if (onRead(pUserData, fmt, sizeof(fmt)) != sizeof(fmt)) {
        return DRWAV_FALSE;
    }
    # 更新已读取的字节数
    *pRunningBytesReadOut += sizeof(fmt);

    # 解析 fmt 数据块的内容并填充到 fmtOut 结构体中
    fmtOut->formatTag      = drwav__bytes_to_u16(fmt + 0);
    fmtOut->channels       = drwav__bytes_to_u16(fmt + 2);
    fmtOut->sampleRate     = drwav__bytes_to_u32(fmt + 4);
    fmtOut->avgBytesPerSec = drwav__bytes_to_u32(fmt + 8);
    fmtOut->blockAlign     = drwav__bytes_to_u16(fmt + 12);
    fmtOut->bitsPerSample  = drwav__bytes_to_u16(fmt + 14);

    fmtOut->extendedSize       = 0;
    fmtOut->validBitsPerSample = 0;
    fmtOut->channelMask        = 0;
    memset(fmtOut->subFormat, 0, sizeof(fmtOut->subFormat));
    # 如果头部大小大于16，则执行以下代码块
    if (header.sizeInBytes > 16) {
        # 创建一个包含2个元素的数组，用于存储 fmt_cbSize
        drwav_uint8 fmt_cbSize[2];
        # 用于记录已读取的字节数
        int bytesReadSoFar = 0;

        # 从输入流中读取 fmt_cbSize 的数据，如果读取的字节数不等于 fmt_cbSize 的大小，则返回 DRWAV_FALSE
        if (onRead(pUserData, fmt_cbSize, sizeof(fmt_cbSize)) != sizeof(fmt_cbSize)) {
            return DRWAV_FALSE;    /* Expecting more data. */
        }
        # 更新已读取的字节数
        *pRunningBytesReadOut += sizeof(fmt_cbSize);

        # 更新已读取的字节数
        bytesReadSoFar = 18;

        # 从 fmt_cbSize 中读取 extendedSize，并存入 fmtOut 结构体
        fmtOut->extendedSize = drwav__bytes_to_u16(fmt_cbSize);
        # 如果 extendedSize 大于0，则执行以下代码块
        if (fmtOut->extendedSize > 0) {
            # 简单验证
            if (fmtOut->formatTag == DR_WAVE_FORMAT_EXTENSIBLE) {
                if (fmtOut->extendedSize != 22) {
                    return DRWAV_FALSE;
                }
            }

            # 如果 formatTag 为 DR_WAVE_FORMAT_EXTENSIBLE，则执行以下代码块
            if (fmtOut->formatTag == DR_WAVE_FORMAT_EXTENSIBLE) {
                # 创建一个包含22个元素的数组，用于存储 fmtext
                drwav_uint8 fmtext[22];
                # 从输入流中读取 fmtext 的数据，如果读取的字节数不等于 fmtOut->extendedSize，则返回 DRWAV_FALSE
                if (onRead(pUserData, fmtext, fmtOut->extendedSize) != fmtOut->extendedSize) {
                    return DRWAV_FALSE;    /* Expecting more data. */
                }

                # 从 fmtext 中读取 validBitsPerSample、channelMask 和 subFormat，并存入 fmtOut 结构体
                fmtOut->validBitsPerSample = drwav__bytes_to_u16(fmtext + 0);
                fmtOut->channelMask        = drwav__bytes_to_u32(fmtext + 2);
                drwav__bytes_to_guid(fmtext + 6, fmtOut->subFormat);
            } else {
                # 如果 formatTag 不为 DR_WAVE_FORMAT_EXTENSIBLE，则执行以下代码块
                # 在输入流中定位到 fmtOut->extendedSize 处
                if (!onSeek(pUserData, fmtOut->extendedSize, drwav_seek_origin_current)) {
                    return DRWAV_FALSE;
                }
            }
            # 更新已读取的字节数
            *pRunningBytesReadOut += fmtOut->extendedSize;

            # 更新已读取的字节数
            bytesReadSoFar += fmtOut->extendedSize;
        }

        # 定位到剩余的字节处。对于 w64，剩余的字节将根据块大小定义。
        if (!onSeek(pUserData, (int)(header.sizeInBytes - bytesReadSoFar), drwav_seek_origin_current)) {
            return DRWAV_FALSE;
        }
        # 更新已读取的字节数
        *pRunningBytesReadOut += (header.sizeInBytes - bytesReadSoFar);
    }
    # 如果头部有填充数据
    if (header.paddingSize > 0) {
        # 如果无法移动到指定位置
        if (!onSeek(pUserData, header.paddingSize, drwav_seek_origin_current)) {
            # 返回失败
            return DRWAV_FALSE;
        }
        # 更新已读取的字节数
        *pRunningBytesReadOut += header.paddingSize;
    }

    # 返回成功
    return DRWAV_TRUE;
}

static size_t drwav__on_read(drwav_read_proc onRead, void* pUserData, void* pBufferOut, size_t bytesToRead, drwav_uint64* pCursor)
{
    size_t bytesRead;

    DRWAV_ASSERT(onRead != NULL);  // 确保 onRead 函数指针不为空
    DRWAV_ASSERT(pCursor != NULL);  // 确保 pCursor 指针不为空

    bytesRead = onRead(pUserData, pBufferOut, bytesToRead);  // 调用 onRead 函数读取数据
    *pCursor += bytesRead;  // 更新游标位置
    return bytesRead;  // 返回读取的字节数
}

#if 0
static drwav_bool32 drwav__on_seek(drwav_seek_proc onSeek, void* pUserData, int offset, drwav_seek_origin origin, drwav_uint64* pCursor)
{
    DRWAV_ASSERT(onSeek != NULL);  // 确保 onSeek 函数指针不为空
    DRWAV_ASSERT(pCursor != NULL);  // 确保 pCursor 指针不为空

    if (!onSeek(pUserData, offset, origin)) {
        return DRWAV_FALSE;  // 如果 onSeek 函数调用失败，返回假
    }

    if (origin == drwav_seek_origin_start) {
        *pCursor = offset;  // 如果是从文件开始位置偏移，直接设置游标位置
    } else {
        *pCursor += offset;  // 否则在当前游标位置上偏移
    }

    return DRWAV_TRUE;  // 返回真
}
#endif

static drwav_uint32 drwav_get_bytes_per_pcm_frame(drwav* pWav)
{
    /*
    The bytes per frame is a bit ambiguous. It can be either be based on the bits per sample, or the block align. The way I'm doing it here
    is that if the bits per sample is a multiple of 8, use floor(bitsPerSample*channels/8), otherwise fall back to the block align.
    */
    if ((pWav->bitsPerSample & 0x7) == 0) {
        /* Bits per sample is a multiple of 8. */
        return (pWav->bitsPerSample * pWav->fmt.channels) >> 3;  // 如果每个样本的位数是8的倍数，使用位数乘以通道数除以8，否则使用块对齐
    } else {
        return pWav->fmt.blockAlign;  // 否则使用块对齐
    }
}

DRWAV_API drwav_uint16 drwav_fmt_get_format(const drwav_fmt* pFMT)
{
    if (pFMT == NULL) {
        return 0;  // 如果格式信息为空，返回0
    }

    if (pFMT->formatTag != DR_WAVE_FORMAT_EXTENSIBLE) {
        return pFMT->formatTag;  // 如果格式标签不是扩展格式，返回格式标签
    } else {
        return drwav__bytes_to_u16(pFMT->subFormat);    /* Only the first two bytes are required. */  // 否则返回子格式的前两个字节
    }
}

static drwav_bool32 drwav_preinit(drwav* pWav, drwav_read_proc onRead, drwav_seek_proc onSeek, void* pReadSeekUserData, const drwav_allocation_callbacks* pAllocationCallbacks)
{
    if (pWav == NULL || onRead == NULL || onSeek == NULL) {
        return DRWAV_FALSE;  // 如果参数为空，返回假
    }
    # 将 pWav 指向的内存清零
    DRWAV_ZERO_MEMORY(pWav, sizeof(*pWav));
    # 设置 pWav 的 onRead 回调函数
    pWav->onRead    = onRead;
    # 设置 pWav 的 onSeek 回调函数
    pWav->onSeek    = onSeek;
    # 设置 pWav 的 pUserData 指针
    pWav->pUserData = pReadSeekUserData;
    # 复制或设置 pAllocationCallbacks 到 pWav 的 allocationCallbacks
    pWav->allocationCallbacks = drwav_copy_allocation_callbacks_or_defaults(pAllocationCallbacks);

    # 检查分配回调函数的有效性，如果无效则返回假
    if (pWav->allocationCallbacks.onFree == NULL || (pWav->allocationCallbacks.onMalloc == NULL && pWav->allocationCallbacks.onRealloc == NULL)) {
        return DRWAV_FALSE;    /* Invalid allocation callbacks. */
    }

    # 返回真
    return DRWAV_TRUE;
    }
    /* 初始化内部状态，假设 drwav_preinit() 已经被调用 */
static drwav_bool32 drwav_init__internal(drwav* pWav, drwav_chunk_proc onChunk, void* pChunkUserData, drwav_uint32 flags)
{
    drwav_uint64 cursor;    /* <-- 用于跟踪字节位置，以便能够定位到特定位置 */
    drwav_bool32 sequential;
    drwav_uint8 riff[4];
    drwav_fmt fmt;
    unsigned short translatedFormatTag;
    drwav_bool32 foundDataChunk;
    drwav_uint64 dataChunkSize = 0; /* <-- 重要！不要在其他地方显式设置为0。数据块大小的计算取决于容器的不同路径。 */
    drwav_uint64 sampleCountFromFactChunk = 0;  /* 与 dataChunkSize 相同 - 确保这是唯一初始化为0的地方。 */
    drwav_uint64 chunkSize;

    cursor = 0;
    sequential = (flags & DRWAV_SEQUENTIAL) != 0;

    /* 前4个字节应该是 RIFF 标识符。 */
    if (drwav__on_read(pWav->onRead, pWav->pUserData, riff, sizeof(riff), &cursor) != sizeof(riff)) {
        return DRWAV_FALSE;
    }

    /*
    前4个字节可用于识别容器。对于 RIFF 文件，它将以 "RIFF" 开头，对于 w64 文件，它将以 "riff" 开头。
    */
    if (drwav__fourcc_equal(riff, "RIFF")) {
        pWav->container = drwav_container_riff;
    } else if (drwav__fourcc_equal(riff, "riff")) {
        int i;
        drwav_uint8 riff2[12];

        pWav->container = drwav_container_w64;

        /* 检查 GUID 的其余部分是否有效。 */
        if (drwav__on_read(pWav->onRead, pWav->pUserData, riff2, sizeof(riff2), &cursor) != sizeof(riff2)) {
            return DRWAV_FALSE;
        }

        for (i = 0; i < 12; ++i) {
            if (riff2[i] != drwavGUID_W64_RIFF[i+4]) {
                return DRWAV_FALSE;
            }
        }
    } else if (drwav__fourcc_equal(riff, "RF64")) {
        pWav->container = drwav_container_rf64;
    } else {
        return DRWAV_FALSE;   /* 未知或不支持的容器类型。 */
    }


    if (pWav->container == drwav_container_riff || pWav->container == drwav_container_rf64) {
        drwav_uint8 chunkSizeBytes[4];
        drwav_uint8 wave[4];

        /* RIFF/WAVE */
        if (drwav__on_read(pWav->onRead, pWav->pUserData, chunkSizeBytes, sizeof(chunkSizeBytes), &cursor) != sizeof(chunkSizeBytes)) {
            return DRWAV_FALSE;
        }

        if (pWav->container == drwav_container_riff) {
            if (drwav__bytes_to_u32(chunkSizeBytes) < 36) {
                return DRWAV_FALSE;    /* 块大小应该至少为36字节。 */
            }
        } else {
            if (drwav__bytes_to_u32(chunkSizeBytes) != 0xFFFFFFFF) {
                return DRWAV_FALSE;    /* 对于RF64，块大小应该设置为-1/0xFFFFFFFF。实际大小稍后获取。 */
            }
        }

        if (drwav__on_read(pWav->onRead, pWav->pUserData, wave, sizeof(wave), &cursor) != sizeof(wave)) {
            return DRWAV_FALSE;
        }

        if (!drwav__fourcc_equal(wave, "WAVE")) {
            return DRWAV_FALSE;    /* 期望"WAVE"。 */
        }
    } else {
        drwav_uint8 chunkSizeBytes[8];
        drwav_uint8 wave[16];

        /* W64 */
        if (drwav__on_read(pWav->onRead, pWav->pUserData, chunkSizeBytes, sizeof(chunkSizeBytes), &cursor) != sizeof(chunkSizeBytes)) {
            return DRWAV_FALSE;
        }

        if (drwav__bytes_to_u64(chunkSizeBytes) < 80) {
            return DRWAV_FALSE;
        }

        if (drwav__on_read(pWav->onRead, pWav->pUserData, wave, sizeof(wave), &cursor) != sizeof(wave)) {
            return DRWAV_FALSE;
        }

        if (!drwav__guid_equal(wave, drwavGUID_W64_WAVE)) {
            return DRWAV_FALSE;
        }
    }


    /* 对于RF64，"ds64"块必须在下一个"fmt "块之前出现。 */
    # 如果 WAV 文件的容器格式是 rf64，则执行以下操作
    if (pWav->container == drwav_container_rf64):
        # 读取8个字节的大小信息
        drwav_uint8 sizeBytes[8];
        # 声明一个变量用于存储剩余的字节数
        drwav_uint64 bytesRemainingInChunk;
        # 声明一个变量用于存储 chunk 的头信息
        drwav_chunk_header header;
        # 读取 chunk 的头信息
        drwav_result result = drwav__read_chunk_header(pWav->onRead, pWav->pUserData, pWav->container, &cursor, &header);
        # 如果读取失败，则返回 DRWAV_FALSE
        if (result != DRWAV_SUCCESS):
            return DRWAV_FALSE;

        # 如果 chunk 的标识符不是 "ds64"，则返回 DRWAV_FALSE
        if (!drwav__fourcc_equal(header.id.fourcc, "ds64")):
            return DRWAV_FALSE; /* Expecting "ds64". */

        # 计算剩余的字节数
        bytesRemainingInChunk = header.sizeInBytes + header.paddingSize;

        # 跳过 RIFF chunk 的大小信息
        if (!drwav__seek_forward(pWav->onSeek, 8, pWav->pUserData)):
            return DRWAV_FALSE;
        bytesRemainingInChunk -= 8;
        cursor += 8;

        # 读取 "data" chunk 的大小信息
        if (drwav__on_read(pWav->onRead, pWav->pUserData, sizeBytes, sizeof(sizeBytes), &cursor) != sizeof(sizeBytes)):
            return DRWAV_FALSE;
        bytesRemainingInChunk -= 8;
        dataChunkSize = drwav__bytes_to_u64(sizeBytes);

        # 读取与 FACT chunk 相同的计数信息
        if (drwav__on_read(pWav->onRead, pWav->pUserData, sizeBytes, sizeof(sizeBytes), &cursor) != sizeof(sizeBytes)):
            return DRWAV_FALSE;
        bytesRemainingInChunk -= 8;
        sampleCountFromFactChunk = drwav__bytes_to_u64(sizeBytes);

        # 跳过剩余的信息
        if (!drwav__seek_forward(pWav->onSeek, bytesRemainingInChunk, pWav->pUserData)):
            return DRWAV_FALSE;
        cursor += bytesRemainingInChunk;

    # 读取 "fmt " chunk 的信息
    if (!drwav__read_fmt(pWav->onRead, pWav->onSeek, pWav->pUserData, pWav->container, &cursor, &fmt)):
        return DRWAV_FALSE;    /* Failed to read the "fmt " chunk. */
    /* 基本验证。*/
    if ((fmt.sampleRate    == 0 || fmt.sampleRate    > DRWAV_MAX_SAMPLE_RATE)     ||
        (fmt.channels      == 0 || fmt.channels      > DRWAV_MAX_CHANNELS)        ||
        (fmt.bitsPerSample == 0 || fmt.bitsPerSample > DRWAV_MAX_BITS_PER_SAMPLE) ||
        fmt.blockAlign == 0) {
        return DRWAV_FALSE; /* 可能是无效的 WAV 文件。*/
    }


    /* 转换内部格式。*/
    translatedFormatTag = fmt.formatTag;
    if (translatedFormatTag == DR_WAVE_FORMAT_EXTENSIBLE) {
        translatedFormatTag = drwav__bytes_to_u16(fmt.subFormat + 0);
    }


    /*
    我们需要枚举每个块的原因有两个：
      1) "data" 块可能不是接下来的块
      2) 我们可能想要将每个块报告给客户端
    
    为了正确地将每个块报告给客户端，我们需要循环直到文件结束。
    */
    foundDataChunk = DRWAV_FALSE;

    /* 我们关心的下一个块是 "data" 块。这不一定是下一个块，所以我们需要循环。*/
    for (;;)
    }

    /* 如果我们没有找到数据块，返回错误。*/
    if (!foundDataChunk) {
        return DRWAV_FALSE;
    }

    /* 我们可能已经跳过了数据块。如果是这样，我们需要回退。如果在顺序模式下运行，我们可以假设我们已经坐在数据块上。*/
    if (!sequential) {
        if (!drwav__seek_from_start(pWav->onSeek, pWav->dataChunkDataPos, pWav->pUserData)) {
            return DRWAV_FALSE;
        }
        cursor = pWav->dataChunkDataPos;
    }
    

    /* 此时我们应该坐在原始音频数据的第一个字节上。*/

    pWav->fmt                 = fmt;
    pWav->sampleRate          = fmt.sampleRate;
    pWav->channels            = fmt.channels;
    pWav->bitsPerSample       = fmt.bitsPerSample;
    pWav->bytesRemaining      = dataChunkSize;
    pWav->translatedFormatTag = translatedFormatTag;
    # 设置 WAV 文件的数据块大小
    pWav->dataChunkDataSize   = dataChunkSize;

    # 如果 Fact Chunk 中的采样数不为 0，则设置总的 PCM 帧数为 Fact Chunk 中的采样数
    if (sampleCountFromFactChunk != 0) {
        pWav->totalPCMFrameCount = sampleCountFromFactChunk;
    } else {
        # 否则，根据数据块大小和每个 PCM 帧的字节数计算总的 PCM 帧数
        pWav->totalPCMFrameCount = dataChunkSize / drwav_get_bytes_per_pcm_frame(pWav);

        # 如果是 ADPCM 格式
        if (pWav->translatedFormatTag == DR_WAVE_FORMAT_ADPCM) {
            # 计算总的块头大小
            drwav_uint64 totalBlockHeaderSizeInBytes;
            drwav_uint64 blockCount = dataChunkSize / fmt.blockAlign;

            # 确保任何尾部不完整的块被计算在内
            if ((blockCount * fmt.blockAlign) < dataChunkSize) {
                blockCount += 1;
            }

            # 每个字节解码两个样本。数据块中会有 blockCount 个块头，这足够计算总的 PCM 帧数
            totalBlockHeaderSizeInBytes = blockCount * (6*fmt.channels);
            pWav->totalPCMFrameCount = ((dataChunkSize - totalBlockHeaderSizeInBytes) * 2) / fmt.channels;
        }
        # 如果是 DVI ADPCM 格式
        if (pWav->translatedFormatTag == DR_WAVE_FORMAT_DVI_ADPCM) {
            # 计算总的块头大小
            drwav_uint64 totalBlockHeaderSizeInBytes;
            drwav_uint64 blockCount = dataChunkSize / fmt.blockAlign;

            # 确保任何尾部不完整的块被计算在内
            if ((blockCount * fmt.blockAlign) < dataChunkSize) {
                blockCount += 1;
            }

            # 每个字节解码两个样本。数据块中会有 blockCount 个块头，这足够计算总的 PCM 帧数
            totalBlockHeaderSizeInBytes = blockCount * (4*fmt.channels);
            pWav->totalPCMFrameCount = ((dataChunkSize - totalBlockHeaderSizeInBytes) * 2) / fmt.channels;

            # 头部包括每个通道的一个解码样本，作为初始预测样本
            pWav->totalPCMFrameCount += blockCount;
        }
    }

    # 一些格式只支持特定数量的通道
    # 如果音频格式为 ADPCM 或 DVI_ADPCM
    if (pWav->translatedFormatTag == DR_WAVE_FORMAT_ADPCM || pWav->translatedFormatTag == DR_WAVE_FORMAT_DVI_ADPCM) {
        # 如果音频通道数大于2，则返回假
        if (pWav->channels > 2) {
            return DRWAV_FALSE;
        }
    }
#ifdef DR_WAV_LIBSNDFILE_COMPAT
    /*
    如果定义了 DR_WAV_LIBSNDFILE_COMPAT，则执行以下代码块，这段代码用于模拟 libsndfile 的逻辑，以便正确运行正确性测试。
    在我使用的 libsndfile 版本中（来自 libsndfile 网站上的 Windows 安装程序），似乎 libsndfile 用于 MS-ADPCM 的总样本计数是不正确的。
    他们似乎是根据块数计算总样本计数，然而这会导致在最后一个块末尾包含额外的静音样本。正确的方法是检查“fact”块，它应该始终存在于压缩格式中，并且应该始终包含样本计数。
    下面这段小代码块仅用于模拟 libsndfile 逻辑，以便我可以正确地运行我的正确性测试，并且默认情况下是禁用的。
    */
    if (pWav->translatedFormatTag == DR_WAVE_FORMAT_ADPCM) {
        drwav_uint64 blockCount = dataChunkSize / fmt.blockAlign;
        pWav->totalPCMFrameCount = (((blockCount * (fmt.blockAlign - (6*pWav->channels))) * 2)) / fmt.channels;  /* x2 because two samples per byte. */
    }
    if (pWav->translatedFormatTag == DR_WAVE_FORMAT_DVI_ADPCM) {
        drwav_uint64 blockCount = dataChunkSize / fmt.blockAlign;
        pWav->totalPCMFrameCount = (((blockCount * (fmt.blockAlign - (4*pWav->channels))) * 2) + (blockCount * pWav->channels)) / fmt.channels;
    }
#endif

    // 返回 DRWAV_TRUE
    return DRWAV_TRUE;
}

// 初始化 drwav 结构体
DRWAV_API drwav_bool32 drwav_init(drwav* pWav, drwav_read_proc onRead, drwav_seek_proc onSeek, void* pUserData, const drwav_allocation_callbacks* pAllocationCallbacks)
{
    // 调用 drwav_init_ex 函数
    return drwav_init_ex(pWav, onRead, onSeek, NULL, pUserData, NULL, 0, pAllocationCallbacks);
}

// 初始化 drwav 结构体（扩展版）
DRWAV_API drwav_bool32 drwav_init_ex(drwav* pWav, drwav_read_proc onRead, drwav_seek_proc onSeek, drwav_chunk_proc onChunk, void* pReadSeekUserData, void* pChunkUserData, drwav_uint32 flags, const drwav_allocation_callbacks* pAllocationCallbacks)
{
    # 如果 drwav_preinit 函数返回 false，则返回 DRWAV_FALSE
    if (!drwav_preinit(pWav, onRead, onSeek, pReadSeekUserData, pAllocationCallbacks)) {
        return DRWAV_FALSE;
    }
    
    # 调用 drwav_init__internal 函数初始化 WAV 文件，返回初始化结果
    return drwav_init__internal(pWav, onChunk, pChunkUserData, flags);
static drwav_uint32 drwav__riff_chunk_size_riff(drwav_uint64 dataChunkSize)
{
    // 计算 RIFF 格式的 chunk 大小，包括 "WAVE" 头部和 "fmt " chunk 的大小
    drwav_uint64 chunkSize = 4 + 24 + dataChunkSize + drwav__chunk_padding_size_riff(dataChunkSize); /* 4 = "WAVE". 24 = "fmt " chunk. */
    // 如果 chunk 大小超过 0xFFFFFFFFUL，则将其截断为 0xFFFFFFFFUL
    if (chunkSize > 0xFFFFFFFFUL) {
        chunkSize = 0xFFFFFFFFUL;
    }

    // 返回 chunk 大小，由于上面的截断，这里可以安全地进行类型转换
    return (drwav_uint32)chunkSize; /* Safe cast due to the clamp above. */
}

static drwav_uint32 drwav__data_chunk_size_riff(drwav_uint64 dataChunkSize)
{
    // 如果数据 chunk 大小不超过 0xFFFFFFFFUL，则直接返回数据 chunk 大小
    if (dataChunkSize <= 0xFFFFFFFFUL) {
        return (drwav_uint32)dataChunkSize;
    } else {
        // 否则返回 0xFFFFFFFFUL
        return 0xFFFFFFFFUL;
    }
}

static drwav_uint64 drwav__riff_chunk_size_w64(drwav_uint64 dataChunkSize)
{
    // 计算 W64 格式的 chunk 大小，包括 "fmt " chunk 的大小和数据 chunk 的大小
    drwav_uint64 dataSubchunkPaddingSize = drwav__chunk_padding_size_w64(dataChunkSize);
    return 80 + 24 + dataChunkSize + dataSubchunkPaddingSize;   /* +24 because W64 includes the size of the GUID and size fields. */
}

static drwav_uint64 drwav__data_chunk_size_w64(drwav_uint64 dataChunkSize)
{
    // 返回 W64 格式的数据 chunk 大小，包括数据 chunk 的大小
    return 24 + dataChunkSize;        /* +24 because W64 includes the size of the GUID and size fields. */
}

static drwav_uint64 drwav__riff_chunk_size_rf64(drwav_uint64 dataChunkSize)
{
    // 计算 RF64 格式的 chunk 大小，包括 "WAVE" 头部、"ds64" chunk 和 "fmt " chunk 的大小
    drwav_uint64 chunkSize = 4 + 36 + 24 + dataChunkSize + drwav__chunk_padding_size_riff(dataChunkSize); /* 4 = "WAVE". 36 = "ds64" chunk. 24 = "fmt " chunk. */
    // 如果 chunk 大小超过 0xFFFFFFFFUL，则将其截断为 0xFFFFFFFFUL
    if (chunkSize > 0xFFFFFFFFUL) {
        chunkSize = 0xFFFFFFFFUL;
    }

    // 返回 chunk 大小
    return chunkSize;
}

static drwav_uint64 drwav__data_chunk_size_rf64(drwav_uint64 dataChunkSize)
{
    // 返回 RF64 格式的数据 chunk 大小，即数据 chunk 的大小
    return dataChunkSize;
}

static size_t drwav__write(drwav* pWav, const void* pData, size_t dataSize)
{
    // 断言 pWav 不为空
    DRWAV_ASSERT(pWav          != NULL);
    // 断言 pWav->onWrite 不为空
    DRWAV_ASSERT(pWav->onWrite != NULL);

    /* Generic write. Assumes no byte reordering required. */
    // 调用 onWrite 回调函数写入数据
    return pWav->onWrite(pWav->pUserData, pData, dataSize);
}

static size_t drwav__write_u16ne_to_le(drwav* pWav, drwav_uint16 value)
{
    // 断言 pWav 不为空
    DRWAV_ASSERT(pWav          != NULL);
    // 断言 pWav->onWrite 不为空
    DRWAV_ASSERT(pWav->onWrite != NULL);
    # 如果不是小端字节序，则将value进行16位大小端字节序转换
    if (!drwav__is_little_endian()) {
        value = drwav__bswap16(value);
    }
    
    # 将value写入到WAV文件中，写入2个字节
    return drwav__write(pWav, &value, 2);
}
# 写入一个 32 位无符号整数到 WAV 文件，将其转换为小端字节序
static size_t drwav__write_u32ne_to_le(drwav* pWav, drwav_uint32 value)
{
    DRWAV_ASSERT(pWav          != NULL);  # 确保 WAV 对象不为空
    DRWAV_ASSERT(pWav->onWrite != NULL);  # 确保写入回调函数不为空

    if (!drwav__is_little_endian()) {  # 如果不是小端字节序，则进行字节序转换
        value = drwav__bswap32(value);
    }

    return drwav__write(pWav, &value, 4);  # 调用写入函数写入数据
}

# 写入一个 64 位无符号整数到 WAV 文件，将其转换为小端字节序
static size_t drwav__write_u64ne_to_le(drwav* pWav, drwav_uint64 value)
{
    DRWAV_ASSERT(pWav          != NULL);  # 确保 WAV 对象不为空
    DRWAV_ASSERT(pWav->onWrite != NULL);  # 确保写入回调函数不为空

    if (!drwav__is_little_endian()) {  # 如果不是小端字节序，则进行字节序转换
        value = drwav__bswap64(value);
    }

    return drwav__write(pWav, &value, 8);  # 调用写入函数写入数据
}

# 初始化 WAV 文件写入前的预处理
static drwav_bool32 drwav_preinit_write(drwav* pWav, const drwav_data_format* pFormat, drwav_bool32 isSequential, drwav_write_proc onWrite, drwav_seek_proc onSeek, void* pUserData, const drwav_allocation_callbacks* pAllocationCallbacks)
{
    if (pWav == NULL || onWrite == NULL) {
        return DRWAV_FALSE;  # 如果 WAV 对象或写入回调函数为空，则返回失败
    }

    if (!isSequential && onSeek == NULL) {
        return DRWAV_FALSE;  # 如果不是顺序写入模式且没有设置 seek 回调函数，则返回失败
    }

    # 不支持压缩格式，需要在启用之前添加对 "fact" 块的支持
    if (pFormat->format == DR_WAVE_FORMAT_EXTENSIBLE) {
        return DRWAV_FALSE;
    }
    if (pFormat->format == DR_WAVE_FORMAT_ADPCM || pFormat->format == DR_WAVE_FORMAT_DVI_ADPCM) {
        return DRWAV_FALSE;
    }

    DRWAV_ZERO_MEMORY(pWav, sizeof(*pWav));  # 清空 WAV 对象的内存
    pWav->onWrite   = onWrite;  # 设置写入回调函数
    pWav->onSeek    = onSeek;   # 设置 seek 回调函数
    pWav->pUserData = pUserData;  # 设置用户数据
    pWav->allocationCallbacks = drwav_copy_allocation_callbacks_or_defaults(pAllocationCallbacks);  # 复制或设置默认的内存分配回调函数

    if (pWav->allocationCallbacks.onFree == NULL || (pWav->allocationCallbacks.onMalloc == NULL && pWav->allocationCallbacks.onRealloc == NULL)) {
        return DRWAV_FALSE;    # 如果内存分配回调函数无效，则返回失败
    }

    pWav->fmt.formatTag = (drwav_uint16)pFormat->format;  # 设置格式标签
    pWav->fmt.channels = (drwav_uint16)pFormat->channels;  # 设置通道数
    pWav->fmt.sampleRate = pFormat->sampleRate;  # 设置采样率
    # 计算每秒的平均字节数，用于确定数据传输速率
    pWav->fmt.avgBytesPerSec = (drwav_uint32)((pFormat->bitsPerSample * pFormat->sampleRate * pFormat->channels) / 8);
    # 计算每个采样数据的字节数
    pWav->fmt.blockAlign = (drwav_uint16)((pFormat->channels * pFormat->bitsPerSample) / 8);
    # 设置每个采样数据的位数
    pWav->fmt.bitsPerSample = (drwav_uint16)pFormat->bitsPerSample;
    # 扩展大小设置为0
    pWav->fmt.extendedSize = 0;
    # 设置是否为顺序写入
    pWav->isSequentialWrite = isSequential;

    # 返回成功标志
    return DRWAV_TRUE;
    }

    // 函数假设在调用之前已经调用了drwav_preinit_write()
    static drwav_bool32 drwav_init_write__internal(drwav* pWav, const drwav_data_format* pFormat, drwav_uint64 totalSampleCount)
    {
        // 运行位置，用于跟踪写入的字节数
        size_t runningPos = 0;
        // 初始数据块大小
        drwav_uint64 initialDataChunkSize = 0;
        // FMT 块大小
        drwav_uint64 chunkSizeFMT;

        /*
        "RIFF" 和 "data" 块的初始值取决于是否在顺序模式下初始化。在顺序模式下，我们直接设置为最终值，因为可以从总样本数计算出来。
        在非顺序模式下，我们将其全部初始化为零，并在drwav_uninit()中使用向后查找来填充。
        */
        if (pWav->isSequentialWrite) {
            initialDataChunkSize = (totalSampleCount * pWav->fmt.bitsPerSample) / 8;

            /*
            RIFF 容器对样本数有限制。drwav 不允许这种情况发生。对于 Wave64，没有实际的限制，所以为了简单起见，我不对其进行任何验证。
            */
            if (pFormat->container == drwav_container_riff) {
                if (initialDataChunkSize > (0xFFFFFFFFUL - 36)) {
                    return DRWAV_FALSE; // 没有足够的空间存储每个样本。
                }
            }
        }

        pWav->dataChunkDataSizeTargetWrite = initialDataChunkSize;

        /* "RIFF" 块。*/
        if (pFormat->container == drwav_container_riff) {
            drwav_uint32 chunkSizeRIFF = 28 + (drwav_uint32)initialDataChunkSize;   // +28 = "WAVE" + [sizeof "fmt " chunk]
            runningPos += drwav__write(pWav, "RIFF", 4);  // 写入 "RIFF" 标识
            runningPos += drwav__write_u32ne_to_le(pWav, chunkSizeRIFF);  // 写入 RIFF 块大小
            runningPos += drwav__write(pWav, "WAVE", 4);  // 写入 "WAVE" 标识
    # 如果容器格式是 W64
    } else if (pFormat->container == drwav_container_w64) {
        # 计算整个文件的大小，包括头部和数据块的大小，W64 包括 GUID 和大小字段的大小
        drwav_uint64 chunkSizeRIFF = 80 + 24 + initialDataChunkSize;            /* +24 because W64 includes the size of the GUID and size fields. */
        # 写入 RIFF 标识和大小
        runningPos += drwav__write(pWav, drwavGUID_W64_RIFF, 16);
        runningPos += drwav__write_u64ne_to_le(pWav, chunkSizeRIFF);
        # 写入 WAVE 标识
        runningPos += drwav__write(pWav, drwavGUID_W64_WAVE, 16);
    } else if (pFormat->container == drwav_container_rf64) {
        # 写入 RF64 标识
        runningPos += drwav__write(pWav, "RF64", 4);
        # 写入 0xFFFFFFFF，因为 RF64 总是这个值，真正的值在 "ds64" 块中设置
        runningPos += drwav__write_u32ne_to_le(pWav, 0xFFFFFFFF);               /* Always 0xFFFFFFFF for RF64. Set to a proper value in the "ds64" chunk. */
        # 写入 WAVE 标识
        runningPos += drwav__write(pWav, "WAVE", 4);
    }

    
    /* "ds64" chunk (RF64 only). */
    # 如果容器格式是 RF64
    if (pFormat->container == drwav_container_rf64) {
        # 计算 ds64 块的大小
        drwav_uint32 initialds64ChunkSize = 28;                                 /* 28 = [Size of RIFF (8 bytes)] + [Size of DATA (8 bytes)] + [Sample Count (8 bytes)] + [Table Length (4 bytes)]. Table length always set to 0. */
        # 计算 RIFF 块的大小
        drwav_uint64 initialRiffChunkSize = 8 + initialds64ChunkSize + initialDataChunkSize;    /* +8 for the ds64 header. */
        # 写入 ds64 标识和大小
        runningPos += drwav__write(pWav, "ds64", 4);
        runningPos += drwav__write_u32ne_to_le(pWav, initialds64ChunkSize);     /* Size of ds64. */
        runningPos += drwav__write_u64ne_to_le(pWav, initialRiffChunkSize);     /* Size of RIFF. Set to true value at the end. */
        runningPos += drwav__write_u64ne_to_le(pWav, initialDataChunkSize);     /* Size of DATA. Set to true value at the end. */
        runningPos += drwav__write_u64ne_to_le(pWav, totalSampleCount);         /* Sample count. */
        runningPos += drwav__write_u32ne_to_le(pWav, 0);                        /* Table length. Always set to zero in our case since we're not doing any other chunks than "DATA". */
    }


    /* "fmt " chunk. */
    // 如果容器格式是RIFF或RF64
    if (pFormat->container == drwav_container_riff || pFormat->container == drwav_container_rf64) {
        // 设置FMT块的大小为16
        chunkSizeFMT = 16;
        // 写入"fmt "标识符
        runningPos += drwav__write(pWav, "fmt ", 4);
        // 写入FMT块大小
        runningPos += drwav__write_u32ne_to_le(pWav, (drwav_uint32)chunkSizeFMT);
    } else if (pFormat->container == drwav_container_w64) {
        // 设置FMT块的大小为40
        chunkSizeFMT = 40;
        // 写入W64格式的FMT块标识符
        runningPos += drwav__write(pWav, drwavGUID_W64_FMT, 16);
        // 写入FMT块大小
        runningPos += drwav__write_u64ne_to_le(pWav, chunkSizeFMT);
    }

    // 写入音频格式相关信息
    runningPos += drwav__write_u16ne_to_le(pWav, pWav->fmt.formatTag);
    runningPos += drwav__write_u16ne_to_le(pWav, pWav->fmt.channels);
    runningPos += drwav__write_u32ne_to_le(pWav, pWav->fmt.sampleRate);
    runningPos += drwav__write_u32ne_to_le(pWav, pWav->fmt.avgBytesPerSec);
    runningPos += drwav__write_u16ne_to_le(pWav, pWav->fmt.blockAlign);
    runningPos += drwav__write_u16ne_to_le(pWav, pWav->fmt.bitsPerSample);

    // 设置数据块的起始位置
    pWav->dataChunkDataPos = runningPos;

    /* "data" chunk. */
    // 根据容器格式写入数据块信息
    if (pFormat->container == drwav_container_riff) {
        // 计算数据块大小
        drwav_uint32 chunkSizeDATA = (drwav_uint32)initialDataChunkSize;
        // 写入"data"标识符
        runningPos += drwav__write(pWav, "data", 4);
        // 写入数据块大小
        runningPos += drwav__write_u32ne_to_le(pWav, chunkSizeDATA);
    } else if (pFormat->container == drwav_container_w64) {
        // 计算数据块大小
        drwav_uint64 chunkSizeDATA = 24 + initialDataChunkSize;     /* +24 because W64 includes the size of the GUID and size fields. */
        // 写入W64格式的"data"标识符
        runningPos += drwav__write(pWav, drwavGUID_W64_DATA, 16);
        // 写入数据块大小
        runningPos += drwav__write_u64ne_to_le(pWav, chunkSizeDATA);
    } else if (pFormat->container == drwav_container_rf64) {
        // 写入"data"标识符
        runningPos += drwav__write(pWav, "data", 4);
        // 写入数据块大小，总是设置为0xFFFFFFFF，因为RF64的真实数据块大小在ds64块中指定
        runningPos += drwav__write_u32ne_to_le(pWav, 0xFFFFFFFF);
    }

    /*
    /*
    runningPos 变量在上面的部分中被递增，但未被使用，这导致一些静态分析工具将其检测为无用存储。为了安全起见，我将其保留在原样，以防以后想要扩展此函数以包括其他标签，并且想要跟踪运行位置的情况。下面的代码应该可以让静态分析工具静音。
    */
    (void)runningPos;

    /* 为客户端方便设置一些属性。 */
    pWav->container = pFormat->container;
    pWav->channels = (drwav_uint16)pFormat->channels;
    pWav->sampleRate = pFormat->sampleRate;
    pWav->bitsPerSample = (drwav_uint16)pFormat->bitsPerSample;
    pWav->translatedFormatTag = (drwav_uint16)pFormat->format;

    返回 DRWAV_TRUE;
}

/*
初始化写入流，用于非顺序写入
参数：
    pWav: 指向 drwav 结构体的指针
    pFormat: 指向 drwav_data_format 结构体的指针，描述了要写入的音频格式
    onWrite: 写入回调函数指针
    onSeek: 定位回调函数指针
    pUserData: 用户数据指针
    pAllocationCallbacks: 内存分配回调函数指针
返回值：
    成功返回 DRWAV_TRUE，失败返回 DRWAV_FALSE
*/
DRWAV_API drwav_bool32 drwav_init_write(drwav* pWav, const drwav_data_format* pFormat, drwav_write_proc onWrite, drwav_seek_proc onSeek, void* pUserData, const drwav_allocation_callbacks* pAllocationCallbacks)
{
    if (!drwav_preinit_write(pWav, pFormat, DRWAV_FALSE, onWrite, onSeek, pUserData, pAllocationCallbacks)) {
        return DRWAV_FALSE;
    }

    return drwav_init_write__internal(pWav, pFormat, 0);               /* DRWAV_FALSE = Not Sequential */
}

/*
初始化写入流，用于顺序写入
参数：
    pWav: 指向 drwav 结构体的指针
    pFormat: 指向 drwav_data_format 结构体的指针，描述了要写入的音频格式
    totalSampleCount: 总样本数
    onWrite: 写入回调函数指针
    pUserData: 用户数据指针
    pAllocationCallbacks: 内存分配回调函数指针
返回值：
    成功返回 DRWAV_TRUE，失败返回 DRWAV_FALSE
*/
DRWAV_API drwav_bool32 drwav_init_write_sequential(drwav* pWav, const drwav_data_format* pFormat, drwav_uint64 totalSampleCount, drwav_write_proc onWrite, void* pUserData, const drwav_allocation_callbacks* pAllocationCallbacks)
{
    if (!drwav_preinit_write(pWav, pFormat, DRWAV_TRUE, onWrite, NULL, pUserData, pAllocationCallbacks)) {
        return DRWAV_FALSE;
    }

    return drwav_init_write__internal(pWav, pFormat, totalSampleCount); /* DRWAV_TRUE = Sequential */
}

/*
初始化写入流，用于顺序写入 PCM 帧
参数：
    pWav: 指向 drwav 结构体的指针
    pFormat: 指向 drwav_data_format 结构体的指针，描述了要写入的音频格式
    totalPCMFrameCount: 总 PCM 帧数
    onWrite: 写入回调函数指针
    pUserData: 用户数据指针
    pAllocationCallbacks: 内存分配回调函数指针
返回值：
    成功返回 DRWAV_TRUE，失败返回 DRWAV_FALSE
*/
DRWAV_API drwav_bool32 drwav_init_write_sequential_pcm_frames(drwav* pWav, const drwav_data_format* pFormat, drwav_uint64 totalPCMFrameCount, drwav_write_proc onWrite, void* pUserData, const drwav_allocation_callbacks* pAllocationCallbacks)
{
    if (pFormat == NULL) {
        return DRWAV_FALSE;
    }

    return drwav_init_write_sequential(pWav, pFormat, totalPCMFrameCount*pFormat->channels, onWrite, pUserData, pAllocationCallbacks);
}

/*
计算写入目标数据的大小（以字节为单位）
参数：
    pFormat: 指向 drwav_data_format 结构体的指针，描述了要写入的音频格式
    totalSampleCount: 总样本数
返回值：
    写入目标数据的大小（以字节为单位）
*/
DRWAV_API drwav_uint64 drwav_target_write_size_bytes(const drwav_data_format* pFormat, drwav_uint64 totalSampleCount)
{
    /* 将 totalSampleCount 转换为 drwav_int64 以兼容 VC6。实际上没有问题，因为没有人会耗尽整个 63 位。 */
    drwav_uint64 targetDataSizeBytes = (drwav_uint64)((drwav_int64)totalSampleCount * pFormat->channels * pFormat->bitsPerSample/8.0);
    drwav_uint64 riffChunkSizeBytes;
    drwav_uint64 fileSizeBytes = 0;
    # 如果容器格式是RIFF，则计算RIFF块的大小
    if (pFormat->container == drwav_container_riff) {
        riffChunkSizeBytes = drwav__riff_chunk_size_riff(targetDataSizeBytes);
        # 计算文件总大小，加8是因为WAV不包括ChunkID和ChunkSize字段的大小
        fileSizeBytes = (8 + riffChunkSizeBytes);   
    } 
    # 如果容器格式是W64，则计算W64块的大小
    else if (pFormat->container == drwav_container_w64) {
        riffChunkSizeBytes = drwav__riff_chunk_size_w64(targetDataSizeBytes);
        # 文件总大小等于W64块的大小
        fileSizeBytes = riffChunkSizeBytes;
    } 
    # 如果容器格式是RF64，则计算RF64块的大小
    else if (pFormat->container == drwav_container_rf64) {
        riffChunkSizeBytes = drwav__riff_chunk_size_rf64(targetDataSizeBytes);
        # 计算文件总大小，加8是因为WAV不包括ChunkID和ChunkSize字段的大小
        fileSizeBytes = (8 + riffChunkSizeBytes);   
    }

    # 返回文件总大小
    return fileSizeBytes;
#ifndef DR_WAV_NO_STDIO

/* drwav_result_from_errno() is only used for fopen() and wfopen() so putting it inside DR_WAV_NO_STDIO for now. If something else needs this later we can move it out. */
#include <errno.h>
static drwav_result drwav_result_from_errno(int e)
{
    switch (e)
    {
        case 0: return DRWAV_SUCCESS;  // 如果错误码为0，返回成功
    #ifdef EPERM
        case EPERM: return DRWAV_INVALID_OPERATION;  // 如果错误码为EPERM，返回无效操作
    #endif
    #ifdef ENOENT
        case ENOENT: return DRWAV_DOES_NOT_EXIST;  // 如果错误码为ENOENT，返回文件不存在
    #endif
    #ifdef ESRCH
        case ESRCH: return DRWAV_DOES_NOT_EXIST;  // 如果错误码为ESRCH，返回文件不存在
    #endif
    #ifdef EINTR
        case EINTR: return DRWAV_INTERRUPT;  // 如果错误码为EINTR，返回中断
    #endif
    #ifdef EIO
        case EIO: return DRWAV_IO_ERROR;  // 如果错误码为EIO，返回IO错误
    #endif
    #ifdef ENXIO
        case ENXIO: return DRWAV_DOES_NOT_EXIST;  // 如果错误码为ENXIO，返回文件不存在
    #endif
    #ifdef E2BIG
        case E2BIG: return DRWAV_INVALID_ARGS;  // 如果错误码为E2BIG，返回无效参数
    #endif
    #ifdef ENOEXEC
        case ENOEXEC: return DRWAV_INVALID_FILE;  // 如果错误码为ENOEXEC，返回无效文件
    #endif
    #ifdef EBADF
        case EBADF: return DRWAV_INVALID_FILE;  // 如果错误码为EBADF，返回无效文件
    #endif
    #ifdef ECHILD
        case ECHILD: return DRWAV_ERROR;  // 如果错误码为ECHILD，返回错误
    #endif
    #ifdef EAGAIN
        case EAGAIN: return DRWAV_UNAVAILABLE;  // 如果错误码为EAGAIN，返回不可用
    #endif
    #ifdef ENOMEM
        case ENOMEM: return DRWAV_OUT_OF_MEMORY;  // 如果错误码为ENOMEM，返回内存不足
    #endif
    #ifdef EACCES
        case EACCES: return DRWAV_ACCESS_DENIED;  // 如果错误码为EACCES，返回访问被拒绝
    #endif
    #ifdef EFAULT
        case EFAULT: return DRWAV_BAD_ADDRESS;  // 如果错误码为EFAULT，返回错误地址
    #endif
    #ifdef ENOTBLK
        case ENOTBLK: return DRWAV_ERROR;  // 如果错误码为ENOTBLK，返回错误
    #endif
    #ifdef EBUSY
        case EBUSY: return DRWAV_BUSY;  // 如果错误码为EBUSY，返回忙碌
    #endif
    #ifdef EEXIST
        case EEXIST: return DRWAV_ALREADY_EXISTS;  // 如果错误码为EEXIST，返回已存在
    #endif
    #ifdef EXDEV
        case EXDEV: return DRWAV_ERROR;  // 如果错误码为EXDEV，返回错误
    #endif
    #ifdef ENODEV
        case ENODEV: return DRWAV_DOES_NOT_EXIST;  // 如果错误码为ENODEV，返回设备不存在
    #endif
    #ifdef ENOTDIR
        case ENOTDIR: return DRWAV_NOT_DIRECTORY;  // 如果错误码为ENOTDIR，返回不是目录
    #endif
    #ifdef EISDIR
        case EISDIR: return DRWAV_IS_DIRECTORY;  // 如果错误码为EISDIR，返回是目录
    #endif
    #ifdef EINVAL
        case EINVAL: return DRWAV_INVALID_ARGS;  // 如果错误码为EINVAL，返回无效参数
    #endif
    #ifdef ENFILE
            // 如果错误码为 ENFILE，则返回 DRWAV_TOO_MANY_OPEN_FILES
            case ENFILE: return DRWAV_TOO_MANY_OPEN_FILES;
    #endif
    #ifdef EMFILE
            // 如果错误码为 EMFILE，则返回 DRWAV_TOO_MANY_OPEN_FILES
            case EMFILE: return DRWAV_TOO_MANY_OPEN_FILES;
    #endif
    #ifdef ENOTTY
            // 如果错误码为 ENOTTY，则返回 DRWAV_INVALID_OPERATION
            case ENOTTY: return DRWAV_INVALID_OPERATION;
    #endif
    #ifdef ETXTBSY
            // 如果错误码为 ETXTBSY，则返回 DRWAV_BUSY
            case ETXTBSY: return DRWAV_BUSY;
    #endif
    #ifdef EFBIG
            // 如果错误码为 EFBIG，则返回 DRWAV_TOO_BIG
            case EFBIG: return DRWAV_TOO_BIG;
    #endif
    #ifdef ENOSPC
            // 如果错误码为 ENOSPC，则返回 DRWAV_NO_SPACE
            case ENOSPC: return DRWAV_NO_SPACE;
    #endif
    #ifdef ESPIPE
            // 如果错误码为 ESPIPE，则返回 DRWAV_BAD_SEEK
            case ESPIPE: return DRWAV_BAD_SEEK;
    #endif
    #ifdef EROFS
            // 如果错误码为 EROFS，则返回 DRWAV_ACCESS_DENIED
            case EROFS: return DRWAV_ACCESS_DENIED;
    #endif
    #ifdef EMLINK
            // 如果错误码为 EMLINK，则返回 DRWAV_TOO_MANY_LINKS
            case EMLINK: return DRWAV_TOO_MANY_LINKS;
    #endif
    #ifdef EPIPE
            // 如果错误码为 EPIPE，则返回 DRWAV_BAD_PIPE
            case EPIPE: return DRWAV_BAD_PIPE;
    #endif
    #ifdef EDOM
            // 如果错误码为 EDOM，则返回 DRWAV_OUT_OF_RANGE
            case EDOM: return DRWAV_OUT_OF_RANGE;
    #endif
    #ifdef ERANGE
            // 如果错误码为 ERANGE，则返回 DRWAV_OUT_OF_RANGE
            case ERANGE: return DRWAV_OUT_OF_RANGE;
    #endif
    #ifdef EDEADLK
            // 如果错误码为 EDEADLK，则返回 DRWAV_DEADLOCK
            case EDEADLK: return DRWAV_DEADLOCK;
    #endif
    #ifdef ENAMETOOLONG
            // 如果错误码为 ENAMETOOLONG，则返回 DRWAV_PATH_TOO_LONG
            case ENAMETOOLONG: return DRWAV_PATH_TOO_LONG;
    #endif
    #ifdef ENOLCK
            // 如果错误码为 ENOLCK，则返回 DRWAV_ERROR
            case ENOLCK: return DRWAV_ERROR;
    #endif
    #ifdef ENOSYS
            // 如果错误码为 ENOSYS，则返回 DRWAV_NOT_IMPLEMENTED
            case ENOSYS: return DRWAV_NOT_IMPLEMENTED;
    #endif
    #ifdef ENOTEMPTY
            // 如果错误码为 ENOTEMPTY，则返回 DRWAV_DIRECTORY_NOT_EMPTY
            case ENOTEMPTY: return DRWAV_DIRECTORY_NOT_EMPTY;
    #endif
    #ifdef ELOOP
            // 如果错误码为 ELOOP，则返回 DRWAV_TOO_MANY_LINKS
            case ELOOP: return DRWAV_TOO_MANY_LINKS;
    #endif
    #ifdef ENOMSG
            // 如果错误码为 ENOMSG，则返回 DRWAV_NO_MESSAGE
            case ENOMSG: return DRWAV_NO_MESSAGE;
    #endif
    #ifdef EIDRM
            // 如果错误码为 EIDRM，则返回 DRWAV_ERROR
            case EIDRM: return DRWAV_ERROR;
    #endif
    #ifdef ECHRNG
            // 如果错误码为 ECHRNG，则返回 DRWAV_ERROR
            case ECHRNG: return DRWAV_ERROR;
    #endif
    #ifdef EL2NSYNC
            // 如果错误码为 EL2NSYNC，则返回 DRWAV_ERROR
            case EL2NSYNC: return DRWAV_ERROR;
    #endif
    #ifdef EL3HLT
            // 如果错误码为 EL3HLT，则返回 DRWAV_ERROR
            case EL3HLT: return DRWAV_ERROR;
    #endif
    #ifdef EL3RST
            // 如果错误码为 EL3RST，则返回 DRWAV_ERROR
            case EL3RST: return DRWAV_ERROR;
    #endif
    #ifdef ELNRNG
            // 如果错误码为 ELNRNG，则返回 DRWAV_OUT_OF_RANGE
            case ELNRNG: return DRWAV_OUT_OF_RANGE;
    #endif
    #ifdef EUNATCH
            // 如果错误码为 EUNATCH，则返回 DRWAV_ERROR
            case EUNATCH: return DRWAV_ERROR;
    #endif
    #ifdef ENOCSI
            // 如果错误码为 ENOCSI，则返回 DRWAV_ERROR
            case ENOCSI: return DRWAV_ERROR;
    #endif
    #ifdef EL2HLT
            // 如果错误码为 EL2HLT，则返回 DRWAV_ERROR
            case EL2HLT: return DRWAV_ERROR;
    #endif
    #ifdef EBADE
            // 如果错误码为 EBADE，则返回 DRWAV_ERROR
            case EBADE: return DRWAV_ERROR;
    #endif
    #ifdef EBADR
            // 如果错误码为 EBADR，则返回 DRWAV_ERROR
            case EBADR: return DRWAV_ERROR;
    #endif
    #ifdef EXFULL
            // 如果错误码为 EXFULL，则返回 DRWAV_ERROR
            case EXFULL: return DRWAV_ERROR;
    #endif
    #ifdef ENOANO
            // 如果错误码为 ENOANO，则返回 DRWAV_ERROR
            case ENOANO: return DRWAV_ERROR;
    #endif
    #ifdef EBADRQC
            // 如果错误码为 EBADRQC，则返回 DRWAV_ERROR
            case EBADRQC: return DRWAV_ERROR;
    #endif
    #ifdef EBADSLT
            // 如果错误码为 EBADSLT，则返回 DRWAV_ERROR
            case EBADSLT: return DRWAV_ERROR;
    #endif
    #ifdef EBFONT
            // 如果错误码为 EBFONT，则返回 DRWAV_INVALID_FILE
            case EBFONT: return DRWAV_INVALID_FILE;
    #endif
    #ifdef ENOSTR
            // 如果错误码为 ENOSTR，则返回 DRWAV_ERROR
            case ENOSTR: return DRWAV_ERROR;
    #endif
    #ifdef ENODATA
            // 如果错误码为 ENODATA，则返回 DRWAV_NO_DATA_AVAILABLE
            case ENODATA: return DRWAV_NO_DATA_AVAILABLE;
    #endif
    #ifdef ETIME
            // 如果错误码为 ETIME，则返回 DRWAV_TIMEOUT
            case ETIME: return DRWAV_TIMEOUT;
    #endif
    #ifdef ENOSR
            // 如果错误码为 ENOSR，则返回 DRWAV_NO_DATA_AVAILABLE
            case ENOSR: return DRWAV_NO_DATA_AVAILABLE;
    #endif
    #ifdef ENONET
            // 如果错误码为 ENONET，则返回 DRWAV_NO_NETWORK
            case ENONET: return DRWAV_NO_NETWORK;
    #endif
    #ifdef ENOPKG
            // 如果错误码为 ENOPKG，则返回 DRWAV_ERROR
            case ENOPKG: return DRWAV_ERROR;
    #endif
    #ifdef EREMOTE
            // 如果错误码为 EREMOTE，则返回 DRWAV_ERROR
            case EREMOTE: return DRWAV_ERROR;
    #endif
    #ifdef ENOLINK
            // 如果错误码为 ENOLINK，则返回 DRWAV_ERROR
            case ENOLINK: return DRWAV_ERROR;
    #endif
    #ifdef EADV
            // 如果错误码为 EADV，则返回 DRWAV_ERROR
            case EADV: return DRWAV_ERROR;
    #endif
    #ifdef ESRMNT
            // 如果错误码为 ESRMNT，则返回 DRWAV_ERROR
            case ESRMNT: return DRWAV_ERROR;
    #endif
    #ifdef ECOMM
            // 如果错误码为 ECOMM，则返回 DRWAV_ERROR
            case ECOMM: return DRWAV_ERROR;
    #endif
    #ifdef EPROTO
            // 如果错误码为 EPROTO，则返回 DRWAV_ERROR
            case EPROTO: return DRWAV_ERROR;
    #endif
    #ifdef EMULTIHOP
            // 如果错误码为 EMULTIHOP，则返回 DRWAV_ERROR
            case EMULTIHOP: return DRWAV_ERROR;
    #endif
    #ifdef EDOTDOT
            // 如果错误码为 EDOTDOT，则返回 DRWAV_ERROR
            case EDOTDOT: return DRWAV_ERROR;
    #endif
    #ifdef EBADMSG
            // 如果错误码为 EBADMSG，则返回 DRWAV_BAD_MESSAGE
            case EBADMSG: return DRWAV_BAD_MESSAGE;
    #endif
    #ifdef EOVERFLOW
            // 如果错误码为 EOVERFLOW，则返回 DRWAV_TOO_BIG
            case EOVERFLOW: return DRWAV_TOO_BIG;
    #endif
    #ifdef ENOTUNIQ
            // 如果错误码为 ENOTUNIQ，则返回 DRWAV_NOT_UNIQUE
            case ENOTUNIQ: return DRWAV_NOT_UNIQUE;
    #endif
    #ifdef EBADFD
            // 如果错误码为 EBADFD，则返回 DRWAV_ERROR
            case EBADFD: return DRWAV_ERROR;
    #endif
    #ifdef EREMCHG
            // 如果错误码为 EREMCHG，则返回 DRWAV_ERROR
            case EREMCHG: return DRWAV_ERROR;
    #endif
    #ifdef ELIBACC
            // 如果错误码为 ELIBACC，则返回 DRWAV_ACCESS_DENIED
            case ELIBACC: return DRWAV_ACCESS_DENIED;
    #endif
    #ifdef ELIBBAD
            // 如果错误码为 ELIBBAD，则返回 DRWAV_INVALID_FILE
            case ELIBBAD: return DRWAV_INVALID_FILE;
    #endif
    #ifdef ELIBSCN
            // 如果错误码为 ELIBSCN，则返回 DRWAV_INVALID_FILE
            case ELIBSCN: return DRWAV_INVALID_FILE;
    #endif
    #ifdef ELIBMAX
            // 如果错误码为 ELIBMAX，则返回 DRWAV_ERROR
            case ELIBMAX: return DRWAV_ERROR;
    #endif
    #ifdef ELIBEXEC
            // 如果错误码为 ELIBEXEC，则返回 DRWAV_ERROR
            case ELIBEXEC: return DRWAV_ERROR;
    #endif
    #ifdef EILSEQ
            // 如果错误码为 EILSEQ，则返回 DRWAV_INVALID_DATA
            case EILSEQ: return DRWAV_INVALID_DATA;
    #endif
    #ifdef ERESTART
            // 如果错误码为 ERESTART，则返回 DRWAV_ERROR
            case ERESTART: return DRWAV_ERROR;
    #endif
    #ifdef ESTRPIPE
            // 如果错误码为 ESTRPIPE，则返回 DRWAV_ERROR
            case ESTRPIPE: return DRWAV_ERROR;
    #endif
    #ifdef EUSERS
            // 如果错误码为 EUSERS，则返回 DRWAV_ERROR
            case EUSERS: return DRWAV_ERROR;
    #endif
    #ifdef ENOTSOCK
            // 如果错误码为 ENOTSOCK，则返回 DRWAV_NOT_SOCKET
            case ENOTSOCK: return DRWAV_NOT_SOCKET;
    #endif
    #ifdef EDESTADDRREQ
            // 如果错误码为 EDESTADDRREQ，则返回 DRWAV_NO_ADDRESS
            case EDESTADDRREQ: return DRWAV_NO_ADDRESS;
    #endif
    #ifdef EMSGSIZE
            // 如果错误码为 EMSGSIZE，则返回 DRWAV_TOO_BIG
            case EMSGSIZE: return DRWAV_TOO_BIG;
    #endif
    #ifdef EPROTOTYPE
            // 如果错误码为 EPROTOTYPE，则返回 DRWAV_BAD_PROTOCOL
            case EPROTOTYPE: return DRWAV_BAD_PROTOCOL;
    #endif
    #ifdef ENOPROTOOPT
            // 如果错误码为 ENOPROTOOPT，则返回 DRWAV_PROTOCOL_UNAVAILABLE
            case ENOPROTOOPT: return DRWAV_PROTOCOL_UNAVAILABLE;
    #endif
    #ifdef EPROTONOSUPPORT
            // 如果错误码为 EPROTONOSUPPORT，则返回 DRWAV_PROTOCOL_NOT_SUPPORTED
            case EPROTONOSUPPORT: return DRWAV_PROTOCOL_NOT_SUPPORTED;
    #endif
    #ifdef ESOCKTNOSUPPORT
            // 如果错误码为 ESOCKTNOSUPPORT，则返回 DRWAV_SOCKET_NOT_SUPPORTED
            case ESOCKTNOSUPPORT: return DRWAV_SOCKET_NOT_SUPPORTED;
    #endif
    #ifdef EOPNOTSUPP
            // 如果错误码为 EOPNOTSUPP，则返回 DRWAV_INVALID_OPERATION
            case EOPNOTSUPP: return DRWAV_INVALID_OPERATION;
    #endif
    #ifdef EPFNOSUPPORT
            // 如果错误码为 EPFNOSUPPORT，则返回 DRWAV_PROTOCOL_FAMILY_NOT_SUPPORTED
            case EPFNOSUPPORT: return DRWAV_PROTOCOL_FAMILY_NOT_SUPPORTED;
    #endif
    #ifdef EAFNOSUPPORT
            // 如果错误码为 EAFNOSUPPORT，则返回 DRWAV_ADDRESS_FAMILY_NOT_SUPPORTED
            case EAFNOSUPPORT: return DRWAV_ADDRESS_FAMILY_NOT_SUPPORTED;
    #endif
    #ifdef EADDRINUSE
            // 如果错误码为 EADDRINUSE，则返回 DRWAV_ALREADY_IN_USE
            case EADDRINUSE: return DRWAV_ALREADY_IN_USE;
    #endif
    #ifdef EADDRNOTAVAIL
            // 如果错误码为 EADDRNOTAVAIL，则返回 DRWAV_ERROR
            case EADDRNOTAVAIL: return DRWAV_ERROR;
    #endif
    #ifdef ENETDOWN
            // 如果错误码为 ENETDOWN，则返回 DRWAV_NO_NETWORK
            case ENETDOWN: return DRWAV_NO_NETWORK;
    #endif
    #ifdef ENETUNREACH
            // 如果发生网络不可达错误，返回无网络错误
            case ENETUNREACH: return DRWAV_NO_NETWORK;
    #endif
    #ifdef ENETRESET
            // 如果发生网络重置错误，返回无网络错误
            case ENETRESET: return DRWAV_NO_NETWORK;
    #endif
    #ifdef ECONNABORTED
            // 如果发生连接中止错误，返回无网络错误
            case ECONNABORTED: return DRWAV_NO_NETWORK;
    #endif
    #ifdef ECONNRESET
            // 如果发生连接重置错误，返回连接重置错误
            case ECONNRESET: return DRWAV_CONNECTION_RESET;
    #endif
    #ifdef ENOBUFS
            // 如果发生缓冲区不足错误，返回无空间错误
            case ENOBUFS: return DRWAV_NO_SPACE;
    #endif
    #ifdef EISCONN
            // 如果已连接，返回已连接错误
            case EISCONN: return DRWAV_ALREADY_CONNECTED;
    #endif
    #ifdef ENOTCONN
            // 如果未连接，返回未连接错误
            case ENOTCONN: return DRWAV_NOT_CONNECTED;
    #endif
    #ifdef ESHUTDOWN
            // 如果发生关闭错误，返回一般错误
            case ESHUTDOWN: return DRWAV_ERROR;
    #endif
    #ifdef ETOOMANYREFS
            // 如果发生过多引用错误，返回一般错误
            case ETOOMANYREFS: return DRWAV_ERROR;
    #endif
    #ifdef ETIMEDOUT
            // 如果发生超时错误，返回超时错误
            case ETIMEDOUT: return DRWAV_TIMEOUT;
    #endif
    #ifdef ECONNREFUSED
            // 如果连接被拒绝，返回连接被拒绝错误
            case ECONNREFUSED: return DRWAV_CONNECTION_REFUSED;
    #endif
    #ifdef EHOSTDOWN
            // 如果主机不可用，返回无主机错误
            case EHOSTDOWN: return DRWAV_NO_HOST;
    #endif
    #ifdef EHOSTUNREACH
            // 如果主机不可达，返回无主机错误
            case EHOSTUNREACH: return DRWAV_NO_HOST;
    #endif
    #ifdef EALREADY
            // 如果操作已经在进行中，返回进行中错误
            case EALREADY: return DRWAV_IN_PROGRESS;
    #endif
    #ifdef EINPROGRESS
            // 如果操作正在进行中，返回进行中错误
            case EINPROGRESS: return DRWAV_IN_PROGRESS;
    #endif
    #ifdef ESTALE
            // 如果发生文件状态错误，返回无效文件错误
            case ESTALE: return DRWAV_INVALID_FILE;
    #endif
    #ifdef EUCLEAN
            // 如果发生清理错误，返回一般错误
            case EUCLEAN: return DRWAV_ERROR;
    #endif
    #ifdef ENOTNAM
            // 如果不是名称错误，返回一般错误
            case ENOTNAM: return DRWAV_ERROR;
    #endif
    #ifdef ENAVAIL
            // 如果不可用错误，返回一般错误
            case ENAVAIL: return DRWAV_ERROR;
    #endif
    #ifdef EISNAM
            // 如果是名称错误，返回一般错误
            case EISNAM: return DRWAV_ERROR;
    #endif
    #ifdef EREMOTEIO
            // 如果远程 I/O 错误，返回 I/O 错误
            case EREMOTEIO: return DRWAV_IO_ERROR;
    #endif
    #ifdef EDQUOT
            // 如果超出配额，返回无空间错误
            case EDQUOT: return DRWAV_NO_SPACE;
    #endif
    #ifdef ENOMEDIUM
            // 如果没有介质，返回不存在错误
            case ENOMEDIUM: return DRWAV_DOES_NOT_EXIST;
    #endif
    #ifdef EMEDIUMTYPE
            // 如果介质类型错误，返回一般错误
            case EMEDIUMTYPE: return DRWAV_ERROR;
    #endif
    #ifdef ECANCELED
            // 如果操作被取消，返回取消错误
            case ECANCELED: return DRWAV_CANCELLED;
    #endif
    #ifdef ENOKEY
        // 如果错误码为 ENOKEY，则返回 DRWAV_ERROR
        case ENOKEY: return DRWAV_ERROR;
    #endif
    #ifdef EKEYEXPIRED
        // 如果错误码为 EKEYEXPIRED，则返回 DRWAV_ERROR
        case EKEYEXPIRED: return DRWAV_ERROR;
    #endif
    #ifdef EKEYREVOKED
        // 如果错误码为 EKEYREVOKED，则返回 DRWAV_ERROR
        case EKEYREVOKED: return DRWAV_ERROR;
    #endif
    #ifdef EKEYREJECTED
        // 如果错误码为 EKEYREJECTED，则返回 DRWAV_ERROR
        case EKEYREJECTED: return DRWAV_ERROR;
    #endif
    #ifdef EOWNERDEAD
        // 如果错误码为 EOWNERDEAD，则返回 DRWAV_ERROR
        case EOWNERDEAD: return DRWAV_ERROR;
    #endif
    #ifdef ENOTRECOVERABLE
        // 如果错误码为 ENOTRECOVERABLE，则返回 DRWAV_ERROR
        case ENOTRECOVERABLE: return DRWAV_ERROR;
    #endif
    #ifdef ERFKILL
        // 如果错误码为 ERFKILL，则返回 DRWAV_ERROR
        case ERFKILL: return DRWAV_ERROR;
    #endif
    #ifdef EHWPOISON
        // 如果错误码为 EHWPOISON，则返回 DRWAV_ERROR
        case EHWPOISON: return DRWAV_ERROR;
    #endif
        // 如果错误码不在上述定义的范围内，则返回 DRWAV_ERROR
        default: return DRWAV_ERROR;
    }
}

static drwav_result drwav_fopen(FILE** ppFile, const char* pFilePath, const char* pOpenMode)
{
#if _MSC_VER && _MSC_VER >= 1400
    errno_t err;
#endif

    if (ppFile != NULL) {
        *ppFile = NULL;  /* Safety. */  // 将指针指向的地址设置为 NULL，确保安全
    }

    if (pFilePath == NULL || pOpenMode == NULL || ppFile == NULL) {
        return DRWAV_INVALID_ARGS;  // 如果文件路径、打开模式或文件指针为空，则返回参数无效
    }

#if _MSC_VER && _MSC_VER >= 1400
    err = fopen_s(ppFile, pFilePath, pOpenMode);  // 使用安全的方式打开文件
    if (err != 0) {
        return drwav_result_from_errno(err);  // 如果打开文件失败，则返回对应的错误码
    }
#else
#if defined(_WIN32) || defined(__APPLE__)
    *ppFile = fopen(pFilePath, pOpenMode);  // 打开文件
#else
    #if defined(_FILE_OFFSET_BITS) && _FILE_OFFSET_BITS == 64 && defined(_LARGEFILE64_SOURCE)
        *ppFile = fopen64(pFilePath, pOpenMode);  // 打开文件（64 位系统）
    #else
        *ppFile = fopen(pFilePath, pOpenMode);  // 打开文件
    #endif
#endif
    if (*ppFile == NULL) {
        drwav_result result = drwav_result_from_errno(errno);  // 获取对应的错误码
        if (result == DRWAV_SUCCESS) {
            result = DRWAV_ERROR;   /* Just a safety check to make sure we never ever return success when pFile == NULL. */
        }

        return result;  // 返回错误码
    }
#endif

    return DRWAV_SUCCESS;  // 返回成功
}

/*
_wfopen() isn't always available in all compilation environments.

    * Windows only.
    * MSVC seems to support it universally as far back as VC6 from what I can tell (haven't checked further back).
    * MinGW-64 (both 32- and 64-bit) seems to support it.
    * MinGW wraps it in !defined(__STRICT_ANSI__).
    * OpenWatcom wraps it in !defined(_NO_EXT_KEYS).

This can be reviewed as compatibility issues arise. The preference is to use _wfopen_s() and _wfopen() as opposed to the wcsrtombs()
fallback, so if you notice your compiler not detecting this properly I'm happy to look at adding support.
*/
#if defined(_WIN32)
    #if defined(_MSC_VER) || defined(__MINGW64__) || (!defined(__STRICT_ANSI__) && !defined(_NO_EXT_KEYS))
        #define DRWAV_HAS_WFOPEN  // 定义宏 DRWAV_HAS_WFOPEN
    #endif
#endif
static drwav_result drwav_wfopen(FILE** ppFile, const wchar_t* pFilePath, const wchar_t* pOpenMode, const drwav_allocation_callbacks* pAllocationCallbacks)
{
    if (ppFile != NULL) {
        *ppFile = NULL;  /* Safety. */
    }

    if (pFilePath == NULL || pOpenMode == NULL || ppFile == NULL) {
        return DRWAV_INVALID_ARGS;
    }

#if defined(DRWAV_HAS_WFOPEN)
    {
        /* 在 Windows 上使用 _wfopen()。*/
    #if defined(_MSC_VER) && _MSC_VER >= 1400
        errno_t err = _wfopen_s(ppFile, pFilePath, pOpenMode);
        if (err != 0) {
            return drwav_result_from_errno(err);
        }
    #else
        *ppFile = _wfopen(pFilePath, pOpenMode);
        if (*ppFile == NULL) {
            return drwav_result_from_errno(errno);
        }
    #endif
        (void)pAllocationCallbacks;
    }
#else
    /*
    在 Windows 以外的系统上使用 fopen()。需要进行转换。这很麻烦，因为 fopen() 是与语言环境相关的。我能想到的唯一方法是使用 wcsrtombs()。注意，wcstombs() 显然不是线程安全的，因为它使用一个静态全局的 mbstate_t 对象来维护状态。我已经检查了这一点，使用 -std=c89 是可以工作的，但如果有人遇到编译错误，我会考虑改进兼容性。
    */
    {
        // 定义多字节状态结构体
        mbstate_t mbs;
        // 多字节字符长度
        size_t lenMB;
        // 临时存放文件路径的宽字符指针
        const wchar_t* pFilePathTemp = pFilePath;
        // 存放多字节文件路径的指针
        char* pFilePathMB = NULL;
        // 存放打开模式的多字节字符串
        char pOpenModeMB[32] = {0};

        /* Get the length first. */
        // 将多字节状态结构体清零
        DRWAV_ZERO_OBJECT(&mbs);
        // 获取多字节字符长度
        lenMB = wcsrtombs(NULL, &pFilePathTemp, 0, &mbs);
        // 如果获取长度失败，返回对应的错误码
        if (lenMB == (size_t)-1) {
            return drwav_result_from_errno(errno);
        }

        // 分配内存存放多字节文件路径
        pFilePathMB = (char*)drwav__malloc_from_callbacks(lenMB + 1, pAllocationCallbacks);
        // 如果内存分配失败，返回内存不足的错误码
        if (pFilePathMB == NULL) {
            return DRWAV_OUT_OF_MEMORY;
        }

        // 重新指向宽字符文件路径
        pFilePathTemp = pFilePath;
        // 将多字节状态结构体清零
        DRWAV_ZERO_OBJECT(&mbs);
        // 将宽字符文件路径转换为多字节文件路径
        wcsrtombs(pFilePathMB, &pFilePathTemp, lenMB + 1, &mbs);

        /* The open mode should always consist of ASCII characters so we should be able to do a trivial conversion. */
        // 将打开模式转换为多字节字符串
        {
            size_t i = 0;
            for (;;) {
                if (pOpenMode[i] == 0) {
                    pOpenModeMB[i] = '\0';
                    break;
                }

                pOpenModeMB[i] = (char)pOpenMode[i];
                i += 1;
            }
        }

        // 打开文件
        *ppFile = fopen(pFilePathMB, pOpenModeMB);

        // 释放多字节文件路径内存
        drwav__free_from_callbacks(pFilePathMB, pAllocationCallbacks);
    }

    // 如果文件指针为空，返回错误码
    if (*ppFile == NULL) {
        return DRWAV_ERROR;
    }
#endif

    return DRWAV_SUCCESS;
}


static size_t drwav__on_read_stdio(void* pUserData, void* pBufferOut, size_t bytesToRead)
{
    // 从标准输入流中读取数据到输出缓冲区
    return fread(pBufferOut, 1, bytesToRead, (FILE*)pUserData);
}

static size_t drwav__on_write_stdio(void* pUserData, const void* pData, size_t bytesToWrite)
{
    // 将数据写入标准输出流
    return fwrite(pData, 1, bytesToWrite, (FILE*)pUserData);
}

static drwav_bool32 drwav__on_seek_stdio(void* pUserData, int offset, drwav_seek_origin origin)
{
    // 在标准输入流中进行定位
    return fseek((FILE*)pUserData, offset, (origin == drwav_seek_origin_current) ? SEEK_CUR : SEEK_SET) == 0;
}

DRWAV_API drwav_bool32 drwav_init_file(drwav* pWav, const char* filename, const drwav_allocation_callbacks* pAllocationCallbacks)
{
    // 初始化文件读取
    return drwav_init_file_ex(pWav, filename, NULL, NULL, 0, pAllocationCallbacks);
}


static drwav_bool32 drwav_init_file__internal_FILE(drwav* pWav, FILE* pFile, drwav_chunk_proc onChunk, void* pChunkUserData, drwav_uint32 flags, const drwav_allocation_callbacks* pAllocationCallbacks)
{
    drwav_bool32 result;

    // 预初始化 WAV 结构
    result = drwav_preinit(pWav, drwav__on_read_stdio, drwav__on_seek_stdio, (void*)pFile, pAllocationCallbacks);
    if (result != DRWAV_TRUE) {
        fclose(pFile);
        return result;
    }

    // 初始化 WAV 结构
    result = drwav_init__internal(pWav, onChunk, pChunkUserData, flags);
    if (result != DRWAV_TRUE) {
        fclose(pFile);
        return result;
    }

    return DRWAV_TRUE;
}

DRWAV_API drwav_bool32 drwav_init_file_ex(drwav* pWav, const char* filename, drwav_chunk_proc onChunk, void* pChunkUserData, drwav_uint32 flags, const drwav_allocation_callbacks* pAllocationCallbacks)
{
    FILE* pFile;
    if (drwav_fopen(&pFile, filename, "rb") != DRWAV_SUCCESS) {
        return DRWAV_FALSE;
    }

    /* This takes ownership of the FILE* object. */
    // 初始化文件读取，接管 FILE* 对象的所有权
    return drwav_init_file__internal_FILE(pWav, pFile, onChunk, pChunkUserData, flags, pAllocationCallbacks);
}
# 使用宽字符文件名初始化 WAV 文件，使用默认的内存分配器
DRWAV_API drwav_bool32 drwav_init_file_w(drwav* pWav, const wchar_t* filename, const drwav_allocation_callbacks* pAllocationCallbacks)
{
    # 调用 drwav_init_file_ex_w 函数，传入 NULL 参数，flags 为 0
    return drwav_init_file_ex_w(pWav, filename, NULL, NULL, 0, pAllocationCallbacks);
}

# 使用宽字符文件名初始化 WAV 文件，可以指定回调函数和标志
DRWAV_API drwav_bool32 drwav_init_file_ex_w(drwav* pWav, const wchar_t* filename, drwav_chunk_proc onChunk, void* pChunkUserData, drwav_uint32 flags, const drwav_allocation_callbacks* pAllocationCallbacks)
{
    # 打开文件，以只读方式打开二进制文件
    FILE* pFile;
    if (drwav_wfopen(&pFile, filename, L"rb", pAllocationCallbacks) != DRWAV_SUCCESS) {
        return DRWAV_FALSE;
    }

    /* This takes ownership of the FILE* object. */
    # 调用 drwav_init_file__internal_FILE 函数，传入打开的文件指针
    return drwav_init_file__internal_FILE(pWav, pFile, onChunk, pChunkUserData, flags, pAllocationCallbacks);
}

# 使用内部 FILE 对象初始化 WAV 文件的写入操作
static drwav_bool32 drwav_init_file_write__internal_FILE(drwav* pWav, FILE* pFile, const drwav_data_format* pFormat, drwav_uint64 totalSampleCount, drwav_bool32 isSequential, const drwav_allocation_callbacks* pAllocationCallbacks)
{
    drwav_bool32 result;

    # 预初始化写入操作
    result = drwav_preinit_write(pWav, pFormat, isSequential, drwav__on_write_stdio, drwav__on_seek_stdio, (void*)pFile, pAllocationCallbacks);
    if (result != DRWAV_TRUE) {
        fclose(pFile);
        return result;
    }

    # 初始化写入操作
    result = drwav_init_write__internal(pWav, pFormat, totalSampleCount);
    if (result != DRWAV_TRUE) {
        fclose(pFile);
        return result;
    }

    return DRWAV_TRUE;
}

# 使用文件名初始化 WAV 文件的写入操作
static drwav_bool32 drwav_init_file_write__internal(drwav* pWav, const char* filename, const drwav_data_format* pFormat, drwav_uint64 totalSampleCount, drwav_bool32 isSequential, const drwav_allocation_callbacks* pAllocationCallbacks)
{
    FILE* pFile;
    if (drwav_fopen(&pFile, filename, "wb") != DRWAV_SUCCESS) {
        return DRWAV_FALSE;
    }

    /* This takes ownership of the FILE* object. */
    # 调用 drwav_init_file_write__internal_FILE 函数，传入打开的文件指针
    return drwav_init_file_write__internal_FILE(pWav, pFile, pFormat, totalSampleCount, isSequential, pAllocationCallbacks);
}
static drwav_bool32 drwav_init_file_write_w__internal(drwav* pWav, const wchar_t* filename, const drwav_data_format* pFormat, drwav_uint64 totalSampleCount, drwav_bool32 isSequential, const drwav_allocation_callbacks* pAllocationCallbacks)
{
    FILE* pFile;
    // 以写入二进制模式打开文件，如果失败则返回 DRWAV_FALSE
    if (drwav_wfopen(&pFile, filename, L"wb", pAllocationCallbacks) != DRWAV_SUCCESS) {
        return DRWAV_FALSE;
    }

    /* This takes ownership of the FILE* object. */
    // 将文件指针对象的所有权转移到 drwav_init_file_write__internal_FILE 函数
    return drwav_init_file_write__internal_FILE(pWav, pFile, pFormat, totalSampleCount, isSequential, pAllocationCallbacks);
}

DRWAV_API drwav_bool32 drwav_init_file_write(drwav* pWav, const char* filename, const drwav_data_format* pFormat, const drwav_allocation_callbacks* pAllocationCallbacks)
{
    // 调用内部函数，totalSampleCount 为 0，isSequential 为 DRWAV_FALSE
    return drwav_init_file_write__internal(pWav, filename, pFormat, 0, DRWAV_FALSE, pAllocationCallbacks);
}

DRWAV_API drwav_bool32 drwav_init_file_write_sequential(drwav* pWav, const char* filename, const drwav_data_format* pFormat, drwav_uint64 totalSampleCount, const drwav_allocation_callbacks* pAllocationCallbacks)
{
    // 调用内部函数，isSequential 为 DRWAV_TRUE
    return drwav_init_file_write__internal(pWav, filename, pFormat, totalSampleCount, DRWAV_TRUE, pAllocationCallbacks);
}

DRWAV_API drwav_bool32 drwav_init_file_write_sequential_pcm_frames(drwav* pWav, const char* filename, const drwav_data_format* pFormat, drwav_uint64 totalPCMFrameCount, const drwav_allocation_callbacks* pAllocationCallbacks)
{
    // 如果 pFormat 为空，则返回 DRWAV_FALSE
    if (pFormat == NULL) {
        return DRWAV_FALSE;
    }

    // 调用内部函数，totalPCMFrameCount 乘以通道数作为 totalSampleCount
    return drwav_init_file_write_sequential(pWav, filename, pFormat, totalPCMFrameCount*pFormat->channels, pAllocationCallbacks);
}

DRWAV_API drwav_bool32 drwav_init_file_write_w(drwav* pWav, const wchar_t* filename, const drwav_data_format* pFormat, const drwav_allocation_callbacks* pAllocationCallbacks)
{
    // 调用内部函数，totalSampleCount 为 0，isSequential 为 DRWAV_FALSE
    return drwav_init_file_write_w__internal(pWav, filename, pFormat, 0, DRWAV_FALSE, pAllocationCallbacks);
}
// 使用宽字符文件名初始化一个可写的 WAV 文件，按顺序写入数据
DRWAV_API drwav_bool32 drwav_init_file_write_sequential_w(drwav* pWav, const wchar_t* filename, const drwav_data_format* pFormat, drwav_uint64 totalSampleCount, const drwav_allocation_callbacks* pAllocationCallbacks)
{
    // 调用内部函数，初始化一个可写的 WAV 文件
    return drwav_init_file_write_w__internal(pWav, filename, pFormat, totalSampleCount, DRWAV_TRUE, pAllocationCallbacks);
}

// 使用宽字符文件名初始化一个按顺序写入 PCM 帧的 WAV 文件
DRWAV_API drwav_bool32 drwav_init_file_write_sequential_pcm_frames_w(drwav* pWav, const wchar_t* filename, const drwav_data_format* pFormat, drwav_uint64 totalPCMFrameCount, const drwav_allocation_callbacks* pAllocationCallbacks)
{
    // 如果数据格式为空，则返回假
    if (pFormat == NULL) {
        return DRWAV_FALSE;
    }

    // 调用初始化按顺序写入 WAV 文件的函数，传入 PCM 帧数乘以通道数作为总样本数
    return drwav_init_file_write_sequential_w(pWav, filename, pFormat, totalPCMFrameCount*pFormat->channels, pAllocationCallbacks);
}
#endif  /* DR_WAV_NO_STDIO */

// 从内存中读取数据的回调函数
static size_t drwav__on_read_memory(void* pUserData, void* pBufferOut, size_t bytesToRead)
{
    // 将用户数据转换为 drwav 结构体指针
    drwav* pWav = (drwav*)pUserData;
    size_t bytesRemaining;

    // 断言 WAV 结构体指针不为空
    DRWAV_ASSERT(pWav != NULL);
    // 断言内存流数据大小大于等于当前读取位置
    DRWAV_ASSERT(pWav->memoryStream.dataSize >= pWav->memoryStream.currentReadPos);

    // 计算剩余的字节数
    bytesRemaining = pWav->memoryStream.dataSize - pWav->memoryStream.currentReadPos;
    // 如果要读取的字节数大于剩余的字节数，则将要读取的字节数设为剩余的字节数
    if (bytesToRead > bytesRemaining) {
        bytesToRead = bytesRemaining;
    }

    // 如果要读取的字节数大于 0
    if (bytesToRead > 0) {
        // 将内存流数据复制到输出缓冲区中
        DRWAV_COPY_MEMORY(pBufferOut, pWav->memoryStream.data + pWav->memoryStream.currentReadPos, bytesToRead);
        // 更新当前读取位置
        pWav->memoryStream.currentReadPos += bytesToRead;
    }

    // 返回实际读取的字节数
    return bytesToRead;
}

// 在内存中定位的回调函数
static drwav_bool32 drwav__on_seek_memory(void* pUserData, int offset, drwav_seek_origin origin)
{
    // 将用户数据转换为 drwav 结构体指针
    drwav* pWav = (drwav*)pUserData;
    // 断言 WAV 结构体指针不为空
    DRWAV_ASSERT(pWav != NULL);
    # 如果是相对当前位置的偏移
    if (origin == drwav_seek_origin_current) {
        # 如果偏移量大于0
        if (offset > 0) {
            # 如果当前读取位置加上偏移量大于数据大小，返回假
            if (pWav->memoryStream.currentReadPos + offset > pWav->memoryStream.dataSize) {
                return DRWAV_FALSE; /* Trying to seek too far forward. */
            }
        } else {
            # 如果当前读取位置小于负偏移量，返回假
            if (pWav->memoryStream.currentReadPos < (size_t)-offset) {
                return DRWAV_FALSE; /* Trying to seek too far backwards. */
            }
        }

        # 更新当前读取位置
        pWav->memoryStream.currentReadPos += offset;
    } else {
        # 如果是绝对位置的偏移
        if ((drwav_uint32)offset <= pWav->memoryStream.dataSize) {
            # 设置当前读取位置为偏移量
            pWav->memoryStream.currentReadPos = offset;
        } else {
            return DRWAV_FALSE; /* Trying to seek too far forward. */
        }
    }
    
    # 返回真
    return DRWAV_TRUE;
# 内存写入回调函数，用于向内存中写入数据
static size_t drwav__on_write_memory(void* pUserData, const void* pDataIn, size_t bytesToWrite)
{
    # 将 void 指针转换为 drwav 指针
    drwav* pWav = (drwav*)pUserData;
    size_t bytesRemaining;

    # 断言确保 pWav 不为空
    DRWAV_ASSERT(pWav != NULL);
    # 断言确保内存流的数据容量大于等于当前写入位置
    DRWAV_ASSERT(pWav->memoryStreamWrite.dataCapacity >= pWav->memoryStreamWrite.currentWritePos);

    # 计算剩余可写入的字节数
    bytesRemaining = pWav->memoryStreamWrite.dataCapacity - pWav->memoryStreamWrite.currentWritePos;
    if (bytesRemaining < bytesToWrite) {
        /* Need to reallocate. */
        # 需要重新分配内存
        void* pNewData;
        size_t newDataCapacity = (pWav->memoryStreamWrite.dataCapacity == 0) ? 256 : pWav->memoryStreamWrite.dataCapacity * 2;

        # 如果翻倍仍不够，就将容量设置为刚好能够写入数据的最小容量
        if ((newDataCapacity - pWav->memoryStreamWrite.currentWritePos) < bytesToWrite) {
            newDataCapacity = pWav->memoryStreamWrite.currentWritePos + bytesToWrite;
        }

        # 重新分配内存
        pNewData = drwav__realloc_from_callbacks(*pWav->memoryStreamWrite.ppData, newDataCapacity, pWav->memoryStreamWrite.dataCapacity, &pWav->allocationCallbacks);
        if (pNewData == NULL) {
            return 0;
        }

        *pWav->memoryStreamWrite.ppData = pNewData;
        pWav->memoryStreamWrite.dataCapacity = newDataCapacity;
    }

    # 将数据写入内存
    DRWAV_COPY_MEMORY(((drwav_uint8*)(*pWav->memoryStreamWrite.ppData)) + pWav->memoryStreamWrite.currentWritePos, pDataIn, bytesToWrite);

    # 更新当前写入位置
    pWav->memoryStreamWrite.currentWritePos += bytesToWrite;
    # 如果数据大小小于当前写入位置，则更新数据大小
    if (pWav->memoryStreamWrite.dataSize < pWav->memoryStreamWrite.currentWritePos) {
        pWav->memoryStreamWrite.dataSize = pWav->memoryStreamWrite.currentWritePos;
    }

    # 更新数据大小
    *pWav->memoryStreamWrite.pDataSize = pWav->memoryStreamWrite.dataSize;

    # 返回写入的字节数
    return bytesToWrite;
}

# 内存写入回调函数，用于在内存中进行定位
static drwav_bool32 drwav__on_seek_memory_write(void* pUserData, int offset, drwav_seek_origin origin)
{
    # 将 void 指针转换为 drwav 指针
    drwav* pWav = (drwav*)pUserData;
    # 断言确保 pWav 不为空
    DRWAV_ASSERT(pWav != NULL);
    # 如果是相对当前位置的偏移
    if (origin == drwav_seek_origin_current):
        # 如果偏移量大于0
        if (offset > 0):
            # 如果当前写入位置加上偏移量大于数据大小
            if (pWav->memoryStreamWrite.currentWritePos + offset > pWav->memoryStreamWrite.dataSize):
                # 限制偏移量，避免超出数据范围
                offset = (int)(pWav->memoryStreamWrite.dataSize - pWav->memoryStreamWrite.currentWritePos)  /* Trying to seek too far forward. */
        else:
            # 如果当前写入位置小于负偏移量
            if (pWav->memoryStreamWrite.currentWritePos < (size_t)-offset):
                # 限制偏移量，避免超出数据范围
                offset = -(int)pWav->memoryStreamWrite.currentWritePos  /* Trying to seek too far backwards. */

        # 更新当前写入位置
        pWav->memoryStreamWrite.currentWritePos += offset;  # This will never underflow thanks to the clamps above.
    else:
        # 如果是绝对位置的偏移
        if ((drwav_uint32)offset <= pWav->memoryStreamWrite.dataSize):
            # 设置当前写入位置为偏移量
            pWav->memoryStreamWrite.currentWritePos = offset
        else:
            # 如果偏移量超出数据大小
            pWav->memoryStreamWrite.currentWritePos = pWav->memoryStreamWrite.dataSize  /* Trying to seek too far forward. */

    # 返回成功
    return DRWAV_TRUE;
}

// 从内存中初始化 WAV 文件
DRWAV_API drwav_bool32 drwav_init_memory(drwav* pWav, const void* data, size_t dataSize, const drwav_allocation_callbacks* pAllocationCallbacks)
{
    // 调用 drwav_init_memory_ex 函数，传入 NULL 参数
    return drwav_init_memory_ex(pWav, data, dataSize, NULL, NULL, 0, pAllocationCallbacks);
}

// 从内存中初始化 WAV 文件，带有额外参数
DRWAV_API drwav_bool32 drwav_init_memory_ex(drwav* pWav, const void* data, size_t dataSize, drwav_chunk_proc onChunk, void* pChunkUserData, drwav_uint32 flags, const drwav_allocation_callbacks* pAllocationCallbacks)
{
    // 如果数据为空或数据大小为0，则返回假
    if (data == NULL || dataSize == 0) {
        return DRWAV_FALSE;
    }

    // 调用 drwav_preinit 函数，传入相应参数
    if (!drwav_preinit(pWav, drwav__on_read_memory, drwav__on_seek_memory, pWav, pAllocationCallbacks)) {
        return DRWAV_FALSE;
    }

    // 设置内存流的数据和大小
    pWav->memoryStream.data = (const drwav_uint8*)data;
    pWav->memoryStream.dataSize = dataSize;
    pWav->memoryStream.currentReadPos = 0;

    // 调用内部初始化函数
    return drwav_init__internal(pWav, onChunk, pChunkUserData, flags);
}

// 从内存中初始化 WAV 文件的写入操作，带有额外参数
static drwav_bool32 drwav_init_memory_write__internal(drwav* pWav, void** ppData, size_t* pDataSize, const drwav_data_format* pFormat, drwav_uint64 totalSampleCount, drwav_bool32 isSequential, const drwav_allocation_callbacks* pAllocationCallbacks)
{
    // 如果数据指针为空，则返回假
    if (ppData == NULL || pDataSize == NULL) {
        return DRWAV_FALSE;
    }

    // 重置数据指针和数据大小
    *ppData = NULL; /* Important because we're using realloc()! */
    *pDataSize = 0;

    // 调用 drwav_preinit_write 函数，传入相应参数
    if (!drwav_preinit_write(pWav, pFormat, isSequential, drwav__on_write_memory, drwav__on_seek_memory_write, pWav, pAllocationCallbacks)) {
        return DRWAV_FALSE;
    }

    // 设置内存流的写入相关参数
    pWav->memoryStreamWrite.ppData = ppData;
    pWav->memoryStreamWrite.pDataSize = pDataSize;
    pWav->memoryStreamWrite.dataSize = 0;
    pWav->memoryStreamWrite.dataCapacity = 0;
    pWav->memoryStreamWrite.currentWritePos = 0;

    // 调用内部初始化写入函数
    return drwav_init_write__internal(pWav, pFormat, totalSampleCount);
}

// 从内存中初始化 WAV 文件的写入操作
DRWAV_API drwav_bool32 drwav_init_memory_write(drwav* pWav, void** ppData, size_t* pDataSize, const drwav_data_format* pFormat, const drwav_allocation_callbacks* pAllocationCallbacks)
{
    # 调用内部函数 drwav_init_memory_write__internal，初始化内存写入
    return drwav_init_memory_write__internal(pWav, ppData, pDataSize, pFormat, 0, DRWAV_FALSE, pAllocationCallbacks);
}

DRWAV_API drwav_bool32 drwav_init_memory_write_sequential(drwav* pWav, void** ppData, size_t* pDataSize, const drwav_data_format* pFormat, drwav_uint64 totalSampleCount, const drwav_allocation_callbacks* pAllocationCallbacks)
{
    return drwav_init_memory_write__internal(pWav, ppData, pDataSize, pFormat, totalSampleCount, DRWAV_TRUE, pAllocationCallbacks);
}

DRWAV_API drwav_bool32 drwav_init_memory_write_sequential_pcm_frames(drwav* pWav, void** ppData, size_t* pDataSize, const drwav_data_format* pFormat, drwav_uint64 totalPCMFrameCount, const drwav_allocation_callbacks* pAllocationCallbacks)
{
    if (pFormat == NULL) {
        return DRWAV_FALSE;
    }

    return drwav_init_memory_write_sequential(pWav, ppData, pDataSize, pFormat, totalPCMFrameCount*pFormat->channels, pAllocationCallbacks);
}



DRWAV_API drwav_result drwav_uninit(drwav* pWav)
{
    drwav_result result = DRWAV_SUCCESS;

    if (pWav == NULL) {
        return DRWAV_INVALID_ARGS;
    }

    /*
    If the drwav object was opened in write mode we'll need to finalize a few things:
      - Make sure the "data" chunk is aligned to 16-bits for RIFF containers, or 64 bits for W64 containers.
      - Set the size of the "data" chunk.
    */
    }

#ifndef DR_WAV_NO_STDIO
    /*
    If we opened the file with drwav_open_file() we will want to close the file handle. We can know whether or not drwav_open_file()
    was used by looking at the onRead and onSeek callbacks.
    */
    if (pWav->onRead == drwav__on_read_stdio || pWav->onWrite == drwav__on_write_stdio) {
        fclose((FILE*)pWav->pUserData);
    }
#endif

    return result;
}



DRWAV_API size_t drwav_read_raw(drwav* pWav, size_t bytesToRead, void* pBufferOut)
{
    size_t bytesRead;

    if (pWav == NULL || bytesToRead == 0) {
        return 0;
    }

    if (bytesToRead > pWav->bytesRemaining) {
        bytesToRead = (size_t)pWav->bytesRemaining;
    }
    # 如果输出缓冲区不为空
    if (pBufferOut != NULL) {
        # 通过回调函数读取数据到输出缓冲区
        bytesRead = pWav->onRead(pWav->pUserData, pBufferOut, bytesToRead);
    } else {
        # 需要进行定位。如果失败，需要读取并丢弃数据以确保获得正确的字节数。
        bytesRead = 0;
        while (bytesRead < bytesToRead) {
            # 计算需要定位的字节数
            size_t bytesToSeek = (bytesToRead - bytesRead);
            if (bytesToSeek > 0x7FFFFFFF) {
                bytesToSeek = 0x7FFFFFFF;
            }

            # 调用定位回调函数
            if (pWav->onSeek(pWav->pUserData, (int)bytesToSeek, drwav_seek_origin_current) == DRWAV_FALSE) {
                break;
            }

            bytesRead += bytesToSeek;
        }

        # 当程序执行到这里时，可能需要读取并丢弃一些数据。
        while (bytesRead < bytesToRead) {
            # 读取并丢弃数据
            drwav_uint8 buffer[4096];
            size_t bytesSeeked;
            size_t bytesToSeek = (bytesToRead - bytesRead);
            if (bytesToSeek > sizeof(buffer)) {
                bytesToSeek = sizeof(buffer);
            }

            bytesSeeked = pWav->onRead(pWav->pUserData, buffer, bytesToSeek);
            bytesRead += bytesSeeked;

            if (bytesSeeked < bytesToSeek) {
                break;  # 到达文件末尾
            }
        }
    }

    # 更新剩余字节数
    pWav->bytesRemaining -= bytesRead;
    # 返回已读取的字节数
    return bytesRead;
}

DRWAV_API drwav_uint64 drwav_read_pcm_frames_le(drwav* pWav, drwav_uint64 framesToRead, void* pBufferOut)
{
    drwav_uint32 bytesPerFrame;  /* 用于存储每个 PCM 帧的字节数 */
    drwav_uint64 bytesToRead;   /* 用于存储要读取的字节数，意图是使用 uint64 而不是 size_t，以便在 32 位构建上进行检查，确保没有读取过多。 */

    if (pWav == NULL || framesToRead == 0) {  /* 检查输入参数是否有效 */
        return 0;
    }

    /* 不能用于压缩格式。 */
    if (drwav__is_compressed_format_tag(pWav->translatedFormatTag)) {  /* 检查是否为压缩格式 */
        return 0;
    }

    bytesPerFrame = drwav_get_bytes_per_pcm_frame(pWav);  /* 获取每个 PCM 帧的字节数 */
    if (bytesPerFrame == 0) {  /* 检查每个 PCM 帧的字节数是否为 0 */
        return 0;
    }

    /* 不要尝试读取超过输出缓冲区可能容纳的样本数。 */
    bytesToRead = framesToRead * bytesPerFrame;  /* 计算要读取的字节数 */
    if (bytesToRead > DRWAV_SIZE_MAX) {  /* 检查要读取的字节数是否超过最大限制 */
        bytesToRead = (DRWAV_SIZE_MAX / bytesPerFrame) * bytesPerFrame; /* 将要读取的字节数舍入到整数帧边界 */
    }

    /*
    在这里进行显式检查，只是为了明确表明，如果没有要读取的字节，我们不希望尝试读取任何内容。可能会出现由于溢出而评估为 0 的情况。
    */
    if (bytesToRead == 0) {  /* 检查要读取的字节数是否为 0 */
        return 0;
    }

    return drwav_read_raw(pWav, (size_t)bytesToRead, pBufferOut) / bytesPerFrame;  /* 读取原始 PCM 数据并返回读取的 PCM 帧数 */
}

DRWAV_API drwav_uint64 drwav_read_pcm_frames_be(drwav* pWav, drwav_uint64 framesToRead, void* pBufferOut)
{
    drwav_uint64 framesRead = drwav_read_pcm_frames_le(pWav, framesToRead, pBufferOut);  /* 调用 drwav_read_pcm_frames_le 函数读取 PCM 帧 */

    if (pBufferOut != NULL) {
        drwav__bswap_samples(pBufferOut, framesRead*pWav->channels, drwav_get_bytes_per_pcm_frame(pWav)/pWav->channels, pWav->translatedFormatTag);  /* 如果输出缓冲区不为空，则进行字节交换 */
    }

    return framesRead;  /* 返回读取的 PCM 帧数 */
}

DRWAV_API drwav_uint64 drwav_read_pcm_frames(drwav* pWav, drwav_uint64 framesToRead, void* pBufferOut)
{
    if (drwav__is_little_endian()) {  /* 检查是否为小端字节序 */
        return drwav_read_pcm_frames_le(pWav, framesToRead, pBufferOut);  /* 如果是小端字节序，则调用 drwav_read_pcm_frames_le 函数读取 PCM 帧 */
    } else {
        # 如果不是小端格式，则调用drwav_read_pcm_frames_be函数读取指定帧数的数据到输出缓冲区
        return drwav_read_pcm_frames_be(pWav, framesToRead, pBufferOut);
    }
# 定义一个函数，用于将指针移动到第一个 PCM 帧
DRWAV_API drwav_bool32 drwav_seek_to_first_pcm_frame(drwav* pWav)
{
    # 如果是写模式，则不支持寻址，返回假
    if (pWav->onWrite != NULL) {
        return DRWAV_FALSE; /* No seeking in write mode. */
    }

    # 如果无法通过回调函数进行寻址，则返回假
    if (!pWav->onSeek(pWav->pUserData, (int)pWav->dataChunkDataPos, drwav_seek_origin_start)) {
        return DRWAV_FALSE;
    }

    # 如果是压缩格式，则需要清除缓存数据
    if (drwav__is_compressed_format_tag(pWav->translatedFormatTag)) {
        pWav->compressed.iCurrentPCMFrame = 0;

        # 根据不同的压缩格式，清除相应的缓存数据
        if (pWav->translatedFormatTag == DR_WAVE_FORMAT_ADPCM) {
            DRWAV_ZERO_OBJECT(&pWav->msadpcm);
        } else if (pWav->translatedFormatTag == DR_WAVE_FORMAT_DVI_ADPCM) {
            DRWAV_ZERO_OBJECT(&pWav->ima);
        } else {
            DRWAV_ASSERT(DRWAV_FALSE);  /* If this assertion is triggered it means I've implemented a new compressed format but forgot to add a branch for it here. */
        }
    }
    
    # 设置剩余字节数为数据块的大小
    pWav->bytesRemaining = pWav->dataChunkDataSize;
    # 返回真
    return DRWAV_TRUE;
}

# 定义一个函数，用于将指针移动到指定的 PCM 帧
DRWAV_API drwav_bool32 drwav_seek_to_pcm_frame(drwav* pWav, drwav_uint64 targetFrameIndex)
{
    /* Seeking should be compatible with wave files > 2GB. */

    # 如果 WAV 对象为空或者无法进行寻址，则返回假
    if (pWav == NULL || pWav->onSeek == NULL) {
        return DRWAV_FALSE;
    }

    # 如果是写模式，则不支持寻址，返回假
    if (pWav->onWrite != NULL) {
        return DRWAV_FALSE;
    }

    # 如果没有样本，则直接返回真
    if (pWav->totalPCMFrameCount == 0) {
        return DRWAV_TRUE;
    }

    # 确保目标帧索引在有效范围内
    if (targetFrameIndex >= pWav->totalPCMFrameCount) {
        targetFrameIndex  = pWav->totalPCMFrameCount - 1;
    }

    # 对于压缩格式，使用慢速通用的寻址方法
    # 如果是向前寻址，则直接向前寻址；如果是向后寻址，则需要回到开头
    if (drwav__is_compressed_format_tag(pWav->translatedFormatTag)) {
        /* 如果是压缩格式的音频，需要进行特殊处理 */

        /*
        如果是向前查找，简单地读取样本，直到找到所需的样本。如果是向后查找，首先需要回到开头，然后执行与向前查找相同的操作。
        */
        if (targetFrameIndex < pWav->compressed.iCurrentPCMFrame) {
            if (!drwav_seek_to_first_pcm_frame(pWav)) {
                return DRWAV_FALSE;
            }
        }

        if (targetFrameIndex > pWav->compressed.iCurrentPCMFrame) {
            drwav_uint64 offsetInFrames = targetFrameIndex - pWav->compressed.iCurrentPCMFrame;

            drwav_int16 devnull[2048];
            while (offsetInFrames > 0) {
                drwav_uint64 framesRead = 0;
                drwav_uint64 framesToRead = offsetInFrames;
                if (framesToRead > drwav_countof(devnull)/pWav->channels) {
                    framesToRead = drwav_countof(devnull)/pWav->channels;
                }

                if (pWav->translatedFormatTag == DR_WAVE_FORMAT_ADPCM) {
                    framesRead = drwav_read_pcm_frames_s16__msadpcm(pWav, framesToRead, devnull);
                } else if (pWav->translatedFormatTag == DR_WAVE_FORMAT_DVI_ADPCM) {
                    framesRead = drwav_read_pcm_frames_s16__ima(pWav, framesToRead, devnull);
                } else {
                    DRWAV_ASSERT(DRWAV_FALSE);  /* 如果触发了这个断言，意味着我实现了一个新的压缩格式，但忘记在这里添加一个分支。 */
                }

                if (framesRead != framesToRead) {
                    return DRWAV_FALSE;
                }

                offsetInFrames -= framesRead;
            }
        }
    } else {
        // 定义变量，存储总字节数、当前字节位置、目标字节位置和偏移量
        drwav_uint64 totalSizeInBytes;
        drwav_uint64 currentBytePos;
        drwav_uint64 targetBytePos;
        drwav_uint64 offset;

        // 计算总字节数
        totalSizeInBytes = pWav->totalPCMFrameCount * drwav_get_bytes_per_pcm_frame(pWav);
        // 断言总字节数大于等于剩余字节数
        DRWAV_ASSERT(totalSizeInBytes >= pWav->bytesRemaining);

        // 计算当前字节位置
        currentBytePos = totalSizeInBytes - pWav->bytesRemaining;
        // 计算目标字节位置
        targetBytePos  = targetFrameIndex * drwav_get_bytes_per_pcm_frame(pWav);

        // 如果当前字节位置小于目标字节位置
        if (currentBytePos < targetBytePos) {
            /* Offset forwards. */
            // 计算偏移量（向前）
            offset = (targetBytePos - currentBytePos);
        } else {
            /* Offset backwards. */
            // 如果当前字节位置大于等于目标字节位置，将文件指针移动到第一个 PCM 帧
            if (!drwav_seek_to_first_pcm_frame(pWav)) {
                return DRWAV_FALSE;
            }
            // 设置偏移量为目标字节位置
            offset = targetBytePos;
        }

        // 循环直到偏移量为 0
        while (offset > 0) {
            // 计算偏移量的 32 位整数值
            int offset32 = ((offset > INT_MAX) ? INT_MAX : (int)offset);
            // 调用 onSeek 函数，移动文件指针
            if (!pWav->onSeek(pWav->pUserData, offset32, drwav_seek_origin_current)) {
                return DRWAV_FALSE;
            }

            // 更新剩余字节数和偏移量
            pWav->bytesRemaining -= offset32;
            offset -= offset32;
        }
    }

    // 返回成功
    return DRWAV_TRUE;
# 写入原始 PCM 数据到 WAV 文件
DRWAV_API size_t drwav_write_raw(drwav* pWav, size_t bytesToWrite, const void* pData)
{
    size_t bytesWritten;

    # 如果 WAV 对象为空，要写入的字节数为0，或者数据为空，则返回0
    if (pWav == NULL || bytesToWrite == 0 || pData == NULL) {
        return 0;
    }

    # 调用回调函数写入数据，并获取实际写入的字节数
    bytesWritten = pWav->onWrite(pWav->pUserData, pData, bytesToWrite);
    # 更新数据块的大小
    pWav->dataChunkDataSize += bytesWritten;

    # 返回实际写入的字节数
    return bytesWritten;
}


# 以小端格式写入 PCM 帧到 WAV 文件
DRWAV_API drwav_uint64 drwav_write_pcm_frames_le(drwav* pWav, drwav_uint64 framesToWrite, const void* pData)
{
    drwav_uint64 bytesToWrite;
    drwav_uint64 bytesWritten;
    const drwav_uint8* pRunningData;

    # 如果 WAV 对象为空，要写入的帧数为0，或者数据为空，则返回0
    if (pWav == NULL || framesToWrite == 0 || pData == NULL) {
        return 0;
    }

    # 计算要写入的字节数
    bytesToWrite = ((framesToWrite * pWav->channels * pWav->bitsPerSample) / 8);
    # 如果要写入的字节数超过最大限制，则返回0
    if (bytesToWrite > DRWAV_SIZE_MAX) {
        return 0;
    }

    bytesWritten = 0;
    pRunningData = (const drwav_uint8*)pData;

    # 循环写入数据
    while (bytesToWrite > 0) {
        size_t bytesJustWritten;
        drwav_uint64 bytesToWriteThisIteration;

        bytesToWriteThisIteration = bytesToWrite;
        DRWAV_ASSERT(bytesToWriteThisIteration <= DRWAV_SIZE_MAX);  /* <-- This is checked above. */

        # 调用写入原始 PCM 数据的函数
        bytesJustWritten = drwav_write_raw(pWav, (size_t)bytesToWriteThisIteration, pRunningData);
        # 如果没有写入任何数据，则退出循环
        if (bytesJustWritten == 0) {
            break;
        }

        # 更新剩余要写入的字节数和已写入的字节数
        bytesToWrite -= bytesJustWritten;
        bytesWritten += bytesJustWritten;
        pRunningData += bytesJustWritten;
    }

    # 返回实际写入的 PCM 帧数
    return (bytesWritten * 8) / pWav->bitsPerSample / pWav->channels;
}

# 以大端格式写入 PCM 帧到 WAV 文件
DRWAV_API drwav_uint64 drwav_write_pcm_frames_be(drwav* pWav, drwav_uint64 framesToWrite, const void* pData)
{
    drwav_uint64 bytesToWrite;
    drwav_uint64 bytesWritten;
    drwav_uint32 bytesPerSample;
    const drwav_uint8* pRunningData;

    # 如果 WAV 对象为空，要写入的帧数为0，或者数据为空，则返回0
    if (pWav == NULL || framesToWrite == 0 || pData == NULL) {
        return 0;
    }

    # 计算要写入的字节数
    bytesToWrite = ((framesToWrite * pWav->channels * pWav->bitsPerSample) / 8);
    # 如果要写入的字节数超过最大限制，则返回0
    if (bytesToWrite > DRWAV_SIZE_MAX) {
        return 0;
    }
    # 初始化已写入的字节数
    bytesWritten = 0;
    # 将指针转换为指向常量的指针
    pRunningData = (const drwav_uint8*)pData;

    # 计算每个采样的字节数
    bytesPerSample = drwav_get_bytes_per_pcm_frame(pWav) / pWav->channels;
    
    # 循环直到所有需要写入的字节数为0
    while (bytesToWrite > 0) {
        # 创建临时缓冲区
        drwav_uint8 temp[4096];
        drwav_uint32 sampleCount;
        size_t bytesJustWritten;
        drwav_uint64 bytesToWriteThisIteration;

        # 每次迭代需要写入的字节数
        bytesToWriteThisIteration = bytesToWrite;
        # 断言，确保每次迭代需要写入的字节数不超过最大值
        DRWAV_ASSERT(bytesToWriteThisIteration <= DRWAV_SIZE_MAX);  /* <-- This is checked above. */

        '''
        WAV 文件总是小端字节序。我们需要在大端字节序的架构上进行字节交换。由于我们的输入缓冲区是只读的，我们需要
        使用中间缓冲区进行转换。
        '''
        # 计算每次迭代需要处理的采样数
        sampleCount = sizeof(temp)/bytesPerSample;

        # 如果每次迭代需要写入的字节数大于采样数乘以每个采样的字节数，则将每次迭代需要写入的字节数设置为采样数乘以每个采样的字节数
        if (bytesToWriteThisIteration > ((drwav_uint64)sampleCount)*bytesPerSample) {
            bytesToWriteThisIteration = ((drwav_uint64)sampleCount)*bytesPerSample;
        }

        # 将数据从输入缓冲区复制到临时缓冲区，并进行字节交换
        DRWAV_COPY_MEMORY(temp, pRunningData, (size_t)bytesToWriteThisIteration);
        drwav__bswap_samples(temp, sampleCount, bytesPerSample, pWav->translatedFormatTag);

        # 将数据写入 WAV 文件
        bytesJustWritten = drwav_write_raw(pWav, (size_t)bytesToWriteThisIteration, temp);
        # 如果没有写入任何数据，则跳出循环
        if (bytesJustWritten == 0) {
            break;
        }

        # 更新剩余需要写入的字节数和已写入的字节数
        bytesToWrite -= bytesJustWritten;
        bytesWritten += bytesJustWritten;
        pRunningData += bytesJustWritten;
    }

    # 返回写入的比特数
    return (bytesWritten * 8) / pWav->bitsPerSample / pWav->channels;
}

DRWAV_API drwav_uint64 drwav_write_pcm_frames(drwav* pWav, drwav_uint64 framesToWrite, const void* pData)
{
    # 如果系统是小端序，调用小端序写入函数
    if (drwav__is_little_endian()) {
        return drwav_write_pcm_frames_le(pWav, framesToWrite, pData);
    } else {
        # 否则调用大端序写入函数
        return drwav_write_pcm_frames_be(pWav, framesToWrite, pData);
    }
}


static drwav_uint64 drwav_read_pcm_frames_s16__msadpcm(drwav* pWav, drwav_uint64 framesToRead, drwav_int16* pBufferOut)
{
    drwav_uint64 totalFramesRead = 0;

    DRWAV_ASSERT(pWav != NULL);
    DRWAV_ASSERT(framesToRead > 0);

    /* TODO: Lots of room for optimization here. */
    # TODO: 这里有很多优化的空间。

    }

    return totalFramesRead;
}


static drwav_uint64 drwav_read_pcm_frames_s16__ima(drwav* pWav, drwav_uint64 framesToRead, drwav_int16* pBufferOut)
{
    drwav_uint64 totalFramesRead = 0;
    drwav_uint32 iChannel;

    static drwav_int32 indexTable[16] = {
        -1, -1, -1, -1, 2, 4, 6, 8,
        -1, -1, -1, -1, 2, 4, 6, 8
    };

    static drwav_int32 stepTable[89] = {
        7,     8,     9,     10,    11,    12,    13,    14,    16,    17, 
        19,    21,    23,    25,    28,    31,    34,    37,    41,    45, 
        50,    55,    60,    66,    73,    80,    88,    97,    107,   118, 
        130,   143,   157,   173,   190,   209,   230,   253,   279,   307,
        337,   371,   408,   449,   494,   544,   598,   658,   724,   796,
        876,   963,   1060,  1166,  1282,  1411,  1552,  1707,  1878,  2066, 
        2272,  2499,  2749,  3024,  3327,  3660,  4026,  4428,  4871,  5358,
        5894,  6484,  7132,  7845,  8630,  9493,  10442, 11487, 12635, 13899, 
        15289, 16818, 18500, 20350, 22385, 24623, 27086, 29794, 32767 
    };

    DRWAV_ASSERT(pWav != NULL);
    DRWAV_ASSERT(framesToRead > 0);

    /* TODO: Lots of room for optimization here. */
    # TODO: 这里有很多优化的空间。

    }

    return totalFramesRead;
}


#ifndef DR_WAV_NO_CONVERSION_API
static unsigned short g_drwavAlawTable[256] = {
    # 一系列十六进制数值
    0xEA80, 0xEB80, 0xE880, 0xE980, 0xEE80, 0xEF80, 0xEC80, 0xED80, 0xE280, 0xE380, 0xE080, 0xE180, 0xE680, 0xE780, 0xE480, 0xE580, 
    0xF540, 0xF5C0, 0xF440, 0xF4C0, 0xF740, 0xF7C0, 0xF640, 0xF6C0, 0xF140, 0xF1C0, 0xF040, 0xF0C0, 0xF340, 0xF3C0, 0xF240, 0xF2C0, 
    0xAA00, 0xAE00, 0xA200, 0xA600, 0xBA00, 0xBE00, 0xB200, 0xB600, 0x8A00, 0x8E00, 0x8200, 0x8600, 0x9A00, 0x9E00, 0x9200, 0x9600, 
    0xD500, 0xD700, 0xD100, 0xD300, 0xDD00, 0xDF00, 0xD900, 0xDB00, 0xC500, 0xC700, 0xC100, 0xC300, 0xCD00, 0xCF00, 0xC900, 0xCB00, 
    0xFEA8, 0xFEB8, 0xFE88, 0xFE98, 0xFEE8, 0xFEF8, 0xFEC8, 0xFED8, 0xFE28, 0xFE38, 0xFE08, 0xFE18, 0xFE68, 0xFE78, 0xFE48, 0xFE58, 
    0xFFA8, 0xFFB8, 0xFF88, 0xFF98, 0xFFE8, 0xFFF8, 0xFFC8, 0xFFD8, 0xFF28, 0xFF38, 0xFF08, 0xFF18, 0xFF68, 0xFF78, 0xFF48, 0xFF58, 
    0xFAA0, 0xFAE0, 0xFA20, 0xFA60, 0xFBA0, 0xFBE0, 0xFB20, 0xFB60, 0xF8A0, 0xF8E0, 0xF820, 0xF860, 0xF9A0, 0xF9E0, 0xF920, 0xF960, 
    0xFD50, 0xFD70, 0xFD10, 0xFD30, 0xFDD0, 0xFDF0, 0xFD90, 0xFDB0, 0xFC50, 0xFC70, 0xFC10, 0xFC30, 0xFCD0, 0xFCF0, 0xFC90, 0xFCB0, 
    0x1580, 0x1480, 0x1780, 0x1680, 0x1180, 0x1080, 0x1380, 0x1280, 0x1D80, 0x1C80, 0x1F80, 0x1E80, 0x1980, 0x1880, 0x1B80, 0x1A80, 
    0x0AC0, 0x0A40, 0x0BC0, 0x0B40, 0x08C0, 0x0840, 0x09C0, 0x0940, 0x0EC0, 0x0E40, 0x0FC0, 0x0F40, 0x0CC0, 0x0C40, 0x0DC0, 0x0D40, 
    0x5600, 0x5200, 0x5E00, 0x5A00, 0x4600, 0x4200, 0x4E00, 0x4A00, 0x7600, 0x7200, 0x7E00, 0x7A00, 0x6600, 0x6200, 0x6E00, 0x6A00, 
    0x2B00, 0x2900, 0x2F00, 0x2D00, 0x2300, 0x2100, 0x2700, 0x2500, 0x3B00, 0x3900, 0x3F00, 0x3D00, 0x3300, 0x3100, 0x3700, 0x3500, 
    0x0158, 0x0148, 0x0178, 0x0168, 0x0118, 0x0108, 0x0138, 0x0128, 0x01D8, 0x01C8, 0x01F8, 0x01E8, 0x0198, 0x0188, 0x01B8, 0x01A8, 
    0x0058, 0x0048, 0x0078, 0x0068, 0x0018, 0x0008, 0x0038, 0x0028, 0x00D8, 0x00C8, 0x00F8, 0x00E8, 0x0098, 0x0088, 0x00B8, 0x00A8, 
    0x0560, 0x0520, 0x05E0, 0x05A0, 0x0460, 0x0420, 0x04E0, 0x04A0, 0x0760, 0x0720, 0x07E0, 0x07A0, 0x0660, 0x0620, 0x06E0, 0x06A0, 
    # 这是一组十六进制数值
    0x02B0, 0x0290, 0x02F0, 0x02D0, 0x0230, 0x0210, 0x0270, 0x0250, 0x03B0, 0x0390, 0x03F0, 0x03D0, 0x0330, 0x0310, 0x0370, 0x0350
// 静态变量，存储了 8 位 mu-law 编码对应的 16 位 PCM 值
static unsigned short g_drwavMulawTable[256] = {
    // mu-law 编码对应的 16 位 PCM 值
    0x8284, 0x8684, 0x8A84, 0x8E84, 0x9284, 0x9684, 0x9A84, 0x9E84, 0xA284, 0xA684, 0xAA84, 0xAE84, 0xB284, 0xB684, 0xBA84, 0xBE84, 
    0xC184, 0xC384, 0xC584, 0xC784, 0xC984, 0xCB84, 0xCD84, 0xCF84, 0xD184, 0xD384, 0xD584, 0xD784, 0xD984, 0xDB84, 0xDD84, 0xDF84, 
    0xE104, 0xE204, 0xE304, 0xE404, 0xE504, 0xE604, 0xE704, 0xE804, 0xE904, 0xEA04, 0xEB04, 0xEC04, 0xED04, 0xEE04, 0xEF04, 0xF004, 
    0xF0C4, 0xF144, 0xF1C4, 0xF244, 0xF2C4, 0xF344, 0xF3C4, 0xF444, 0xF4C4, 0xF544, 0xF5C4, 0xF644, 0xF6C4, 0xF744, 0xF7C4, 0xF844, 
    0xF8A4, 0xF8E4, 0xF924, 0xF964, 0xF9A4, 0xF9E4, 0xFA24, 0xFA64, 0xFAA4, 0xFAE4, 0xFB24, 0xFB64, 0xFBA4, 0xFBE4, 0xFC24, 0xFC64, 
    0xFC94, 0xFCB4, 0xFCD4, 0xFCF4, 0xFD14, 0xFD34, 0xFD54, 0xFD74, 0xFD94, 0xFDB4, 0xFDD4, 0xFDF4, 0xFE14, 0xFE34, 0xFE54, 0xFE74, 
    0xFE8C, 0xFE9C, 0xFEAC, 0xFEBC, 0xFECC, 0xFEDC, 0xFEEC, 0xFEFC, 0xFF0C, 0xFF1C, 0xFF2C, 0xFF3C, 0xFF4C, 0xFF5C, 0xFF6C, 0xFF7C, 
    0xFF88, 0xFF90, 0xFF98, 0xFFA0, 0xFFA8, 0xFFB0, 0xFFB8, 0xFFC0, 0xFFC8, 0xFFD0, 0xFFD8, 0xFFE0, 0xFFE8, 0xFFF0, 0xFFF8, 0x0000, 
    0x7D7C, 0x797C, 0x757C, 0x717C, 0x6D7C, 0x697C, 0x657C, 0x617C, 0x5D7C, 0x597C, 0x557C, 0x517C, 0x4D7C, 0x497C, 0x457C, 0x417C, 
    0x3E7C, 0x3C7C, 0x3A7C, 0x387C, 0x367C, 0x347C, 0x327C, 0x307C, 0x2E7C, 0x2C7C, 0x2A7C, 0x287C, 0x267C, 0x247C, 0x227C, 0x207C, 
    0x1EFC, 0x1DFC, 0x1CFC, 0x1BFC, 0x1AFC, 0x19FC, 0x18FC, 0x17FC, 0x16FC, 0x15FC, 0x14FC, 0x13FC, 0x12FC, 0x11FC, 0x10FC, 0x0FFC, 
    0x0F3C, 0x0EBC, 0x0E3C, 0x0DBC, 0x0D3C, 0x0CBC, 0x0C3C, 0x0BBC, 0x0B3C, 0x0ABC, 0x0A3C, 0x09BC, 0x093C, 0x08BC, 0x083C, 0x07BC, 
    0x075C, 0x071C, 0x06DC, 0x069C, 0x065C, 0x061C, 0x05DC, 0x059C, 0x055C, 0x051C, 0x04DC, 0x049C, 0x045C, 0x041C, 0x03DC, 0x039C, 
    0x036C, 0x034C, 0x032C, 0x030C, 0x02EC, 0x02CC, 0x02AC, 0x028C, 0x026C, 0x024C, 0x022C, 0x020C, 0x01EC, 0x01CC, 0x01AC, 0x018C, 
};
    # 16进制数值列表，用于某种特定的计算或操作
    0x0174, 0x0164, 0x0154, 0x0144, 0x0134, 0x0124, 0x0114, 0x0104, 0x00F4, 0x00E4, 0x00D4, 0x00C4, 0x00B4, 0x00A4, 0x0094, 0x0084, 
    0x0078, 0x0070, 0x0068, 0x0060, 0x0058, 0x0050, 0x0048, 0x0040, 0x0038, 0x0030, 0x0028, 0x0020, 0x0018, 0x0010, 0x0008, 0x0000
};

// 将 A-law 格式的采样数据转换为有符号 16 位整数
static DRWAV_INLINE drwav_int16 drwav__alaw_to_s16(drwav_uint8 sampleIn)
{
    return (short)g_drwavAlawTable[sampleIn];
}

// 将 u-law 格式的采样数据转换为有符号 16 位整数
static DRWAV_INLINE drwav_int16 drwav__mulaw_to_s16(drwav_uint8 sampleIn)
{
    return (short)g_drwavMulawTable[sampleIn];
}

// 将 PCM 格式的采样数据转换为有符号 16 位整数
static void drwav__pcm_to_s16(drwav_int16* pOut, const drwav_uint8* pIn, size_t totalSampleCount, unsigned int bytesPerSample)
{
    unsigned int i;

    /* 8 位采样数据的特殊情况，因为它被视为无符号的 */
    if (bytesPerSample == 1) {
        drwav_u8_to_s16(pOut, pIn, totalSampleCount);
        return;
    }

    /* 更为常见格式的略微更优化的实现 */
    if (bytesPerSample == 2) {
        for (i = 0; i < totalSampleCount; ++i) {
           *pOut++ = ((const drwav_int16*)pIn)[i];
        }
        return;
    }
    if (bytesPerSample == 3) {
        drwav_s24_to_s16(pOut, pIn, totalSampleCount);
        return;
    }
    if (bytesPerSample == 4) {
        drwav_s32_to_s16(pOut, (const drwav_int32*)pIn, totalSampleCount);
        return;
    }

    /* 不支持超过 64 位每个采样的情况 */
    if (bytesPerSample > 8) {
        DRWAV_ZERO_MEMORY(pOut, totalSampleCount * sizeof(*pOut));
        return;
    }

    /* 通用、较慢的转换器 */
    for (i = 0; i < totalSampleCount; ++i) {
        drwav_uint64 sample = 0;
        unsigned int shift  = (8 - bytesPerSample) * 8;

        unsigned int j;
        for (j = 0; j < bytesPerSample; j += 1) {
            DRWAV_ASSERT(j < 8);
            sample |= (drwav_uint64)(pIn[j]) << shift;
            shift  += 8;
        }

        pIn += j;
        *pOut++ = (drwav_int16)((drwav_int64)sample >> 48);
    }
}

// 将 IEEE 格式的采样数据转换为有符号 16 位整数
static void drwav__ieee_to_s16(drwav_int16* pOut, const drwav_uint8* pIn, size_t totalSampleCount, unsigned int bytesPerSample)
{
    if (bytesPerSample == 4) {
        drwav_f32_to_s16(pOut, (const float*)pIn, totalSampleCount);
        return;
    } else if (bytesPerSample == 8) {
        // 如果每个样本占用8个字节，将输入的64位浮点数转换为16位有符号整数，并存储到输出缓冲区中
        drwav_f64_to_s16(pOut, (const double*)pIn, totalSampleCount);
        // 返回
        return;
    } else {
        /* 只支持32位和64位浮点数。在所有其他情况下输出静音。欢迎为16位浮点数提供贡献。 */
        // 在所有其他情况下，将输出缓冲区中的数据清零，表示输出静音
        DRWAV_ZERO_MEMORY(pOut, totalSampleCount * sizeof(*pOut));
        // 返回
        return;
    }
# 读取带符号16位PCM格式的音频数据帧
static drwav_uint64 drwav_read_pcm_frames_s16__pcm(drwav* pWav, drwav_uint64 framesToRead, drwav_int16* pBufferOut)
{
    # 每个音频帧的字节数
    drwav_uint32 bytesPerFrame;
    # 总共读取的帧数
    drwav_uint64 totalFramesRead;
    # 临时存储音频数据的数组
    drwav_uint8 sampleData[4096];

    /* Fast path. */
    # 如果是PCM格式且每个样本是16位，或者输出缓冲区为空，则直接调用drwav_read_pcm_frames函数
    if ((pWav->translatedFormatTag == DR_WAVE_FORMAT_PCM && pWav->bitsPerSample == 16) || pBufferOut == NULL) {
        return drwav_read_pcm_frames(pWav, framesToRead, pBufferOut);
    }
    
    # 计算每个音频帧的字节数
    bytesPerFrame = drwav_get_bytes_per_pcm_frame(pWav);
    if (bytesPerFrame == 0) {
        return 0;
    }

    # 初始化总共读取的帧数
    totalFramesRead = 0;
    
    # 循环读取音频帧数据
    while (framesToRead > 0) {
        # 读取音频帧数据
        drwav_uint64 framesRead = drwav_read_pcm_frames(pWav, drwav_min(framesToRead, sizeof(sampleData)/bytesPerFrame), sampleData);
        if (framesRead == 0) {
            break;
        }

        # 将PCM格式的音频数据转换为带符号16位整数
        drwav__pcm_to_s16(pBufferOut, sampleData, (size_t)(framesRead*pWav->channels), bytesPerFrame/pWav->channels);

        # 更新输出缓冲区指针和剩余帧数
        pBufferOut      += framesRead*pWav->channels;
        framesToRead    -= framesRead;
        totalFramesRead += framesRead;
    }

    return totalFramesRead;
}

# 读取IEEE浮点格式的音频数据帧
static drwav_uint64 drwav_read_pcm_frames_s16__ieee(drwav* pWav, drwav_uint64 framesToRead, drwav_int16* pBufferOut)
{
    # 总共读取的帧数
    drwav_uint64 totalFramesRead;
    # 临时存储音频数据的数组
    drwav_uint8 sampleData[4096];
    # 每个音频帧的字节数
    drwav_uint32 bytesPerFrame;

    # 如果输出缓冲区为空，则直接调用drwav_read_pcm_frames函数
    if (pBufferOut == NULL) {
        return drwav_read_pcm_frames(pWav, framesToRead, NULL);
    }

    # 计算每个音频帧的字节数
    bytesPerFrame = drwav_get_bytes_per_pcm_frame(pWav);
    if (bytesPerFrame == 0) {
        return 0;
    }

    # 初始化总共读取的帧数
    totalFramesRead = 0;
    # 当还有帧需要读取时进入循环
    while (framesToRead > 0) {
        # 从 WAV 文件中读取指定数量的 PCM 帧到 sampleData 中，并返回实际读取的帧数
        drwav_uint64 framesRead = drwav_read_pcm_frames(pWav, drwav_min(framesToRead, sizeof(sampleData)/bytesPerFrame), sampleData);
        # 如果没有读取到任何帧，则跳出循环
        if (framesRead == 0) {
            break;
        }

        # 将 sampleData 中的 IEEE 格式数据转换为有符号 16 位整数，并存储到 pBufferOut 中
        drwav__ieee_to_s16(pBufferOut, sampleData, (size_t)(framesRead*pWav->channels), bytesPerFrame/pWav->channels);

        # 更新 pBufferOut 的位置指针
        pBufferOut      += framesRead*pWav->channels;
        # 更新剩余需要读取的帧数
        framesToRead    -= framesRead;
        # 更新总共已经读取的帧数
        totalFramesRead += framesRead;
    }

    # 返回总共已经读取的帧数
    return totalFramesRead;
# 读取带有 A-law 压缩的 PCM 数据并将其解压缩成 16 位有符号整数格式
static drwav_uint64 drwav_read_pcm_frames_s16__alaw(drwav* pWav, drwav_uint64 framesToRead, drwav_int16* pBufferOut)
{
    # 用于存储总共读取的帧数
    drwav_uint64 totalFramesRead;
    # 用于存储样本数据的缓冲区
    drwav_uint8 sampleData[4096];
    # 每帧的字节数
    drwav_uint32 bytesPerFrame;

    # 如果输出缓冲区为空，则直接调用 drwav_read_pcm_frames 函数读取数据
    if (pBufferOut == NULL) {
        return drwav_read_pcm_frames(pWav, framesToRead, NULL);
    }

    # 获取每帧的字节数
    bytesPerFrame = drwav_get_bytes_per_pcm_frame(pWav);
    # 如果每帧的字节数为 0，则返回 0
    if (bytesPerFrame == 0) {
        return 0;
    }

    # 初始化总共读取的帧数为 0
    totalFramesRead = 0;
    
    # 循环读取数据直到读取完指定的帧数
    while (framesToRead > 0) {
        # 读取指定数量的帧数到 sampleData 缓冲区中
        drwav_uint64 framesRead = drwav_read_pcm_frames(pWav, drwav_min(framesToRead, sizeof(sampleData)/bytesPerFrame), sampleData);
        # 如果没有读取到任何帧数，则跳出循环
        if (framesRead == 0) {
            break;
        }

        # 将 A-law 压缩的样本数据解压缩成 16 位有符号整数格式
        drwav_alaw_to_s16(pBufferOut, sampleData, (size_t)(framesRead*pWav->channels));

        # 更新输出缓冲区的位置和剩余需要读取的帧数
        pBufferOut      += framesRead*pWav->channels;
        framesToRead    -= framesRead;
        totalFramesRead += framesRead;
    }

    # 返回总共读取的帧数
    return totalFramesRead;
}

# 读取带有 μ-law 压缩的 PCM 数据并将其解压缩成 16 位有符号整数格式
static drwav_uint64 drwav_read_pcm_frames_s16__mulaw(drwav* pWav, drwav_uint64 framesToRead, drwav_int16* pBufferOut)
{
    # 用于存储总共读取的帧数
    drwav_uint64 totalFramesRead;
    # 用于存储样本数据的缓冲区
    drwav_uint8 sampleData[4096];
    # 每帧的字节数
    drwav_uint32 bytesPerFrame;

    # 如果输出缓冲区为空，则直接调用 drwav_read_pcm_frames 函数读取数据
    if (pBufferOut == NULL) {
        return drwav_read_pcm_frames(pWav, framesToRead, NULL);
    }

    # 获取每帧的字节数
    bytesPerFrame = drwav_get_bytes_per_pcm_frame(pWav);
    # 如果每帧的字节数为 0，则返回 0
    if (bytesPerFrame == 0) {
        return 0;
    }

    # 初始化总共读取的帧数为 0
    totalFramesRead = 0;

    # 循环读取数据直到读取完指定的帧数
    while (framesToRead > 0) {
        # 读取指定数量的帧数到 sampleData 缓冲区中
        drwav_uint64 framesRead = drwav_read_pcm_frames(pWav, drwav_min(framesToRead, sizeof(sampleData)/bytesPerFrame), sampleData);
        # 如果没有读取到任何帧数，则跳出循环
        if (framesRead == 0) {
            break;
        }

        # 将 μ-law 压缩的样本数据解压缩成 16 位有符号整数格式
        drwav_mulaw_to_s16(pBufferOut, sampleData, (size_t)(framesRead*pWav->channels));

        # 更新输出缓冲区的位置和剩余需要读取的帧数
        pBufferOut      += framesRead*pWav->channels;
        framesToRead    -= framesRead;
        totalFramesRead += framesRead;
    }

    # 返回总共读取的帧数
    return totalFramesRead;
}
# 读取指定数量的帧并将其解码为有符号16位整数格式的PCM数据
def drwav_read_pcm_frames_s16(drwav* pWav, drwav_uint64 framesToRead, drwav_int16* pBufferOut)
    # 如果输入的WAV文件指针为空或者要读取的帧数为0，则返回0
    if (pWav == NULL || framesToRead == 0):
        return 0;

    # 如果输出缓冲区为空，则调用drwav_read_pcm_frames函数
    if (pBufferOut == NULL):
        return drwav_read_pcm_frames(pWav, framesToRead, NULL);

    # 如果要读取的样本数超过输出缓冲区的最大容量，则调整要读取的帧数
    if (framesToRead * pWav->channels * sizeof(drwav_int16) > DRWAV_SIZE_MAX):
        framesToRead = DRWAV_SIZE_MAX / sizeof(drwav_int16) / pWav->channels;

    # 根据WAV文件的格式标签，调用相应的解码函数
    if (pWav->translatedFormatTag == DR_WAVE_FORMAT_PCM):
        return drwav_read_pcm_frames_s16__pcm(pWav, framesToRead, pBufferOut);

    if (pWav->translatedFormatTag == DR_WAVE_FORMAT_IEEE_FLOAT):
        return drwav_read_pcm_frames_s16__ieee(pWav, framesToRead, pBufferOut);

    if (pWav->translatedFormatTag == DR_WAVE_FORMAT_ALAW):
        return drwav_read_pcm_frames_s16__alaw(pWav, framesToRead, pBufferOut);

    if (pWav->translatedFormatTag == DR_WAVE_FORMAT_MULAW):
        return drwav_read_pcm_frames_s16__mulaw(pWav, framesToRead, pBufferOut);

    if (pWav->translatedFormatTag == DR_WAVE_FORMAT_ADPCM):
        return drwav_read_pcm_frames_s16__msadpcm(pWav, framesToRead, pBufferOut);

    if (pWav->translatedFormatTag == DR_WAVE_FORMAT_DVI_ADPCM):
        return drwav_read_pcm_frames_s16__ima(pWav, framesToRead, pBufferOut);

    # 如果格式标签不匹配任何已知格式，则返回0
    return 0;

# 读取指定数量的帧并将其解码为小端序的有符号16位整数格式的PCM数据
def drwav_read_pcm_frames_s16le(drwav* pWav, drwav_uint64 framesToRead, drwav_int16* pBufferOut)
    # 调用drwav_read_pcm_frames_s16函数读取PCM数据
    framesRead = drwav_read_pcm_frames_s16(pWav, framesToRead, pBufferOut)
    # 如果输出缓冲区不为空且系统不是小端序，则交换字节序
    if (pBufferOut != NULL && drwav__is_little_endian() == DRWAV_FALSE):
        drwav__bswap_samples_s16(pBufferOut, framesRead*pWav->channels);
    # 返回实际读取的帧数
    return framesRead;

# 读取指定数量的帧并将其解码为大端序的有符号16位整数格式的PCM数据
def drwav_read_pcm_frames_s16be(drwav* pWav, drwav_uint64 framesToRead, drwav_int16* pBufferOut)
    # 从 WAV 文件中读取指定数量的 PCM 帧，返回实际读取的帧数
    drwav_uint64 framesRead = drwav_read_pcm_frames_s16(pWav, framesToRead, pBufferOut);
    # 检查输出缓冲区是否存在并且系统是否为小端序
    if (pBufferOut != NULL && drwav__is_little_endian() == DRWAV_TRUE) {
        # 如果是小端序，交换输出缓冲区中的 16 位样本的字节序
        drwav__bswap_samples_s16(pBufferOut, framesRead*pWav->channels);
    }

    # 返回实际读取的帧数
    return framesRead;
// 将无符号8位整数数组转换为有符号16位整数数组
DRWAV_API void drwav_u8_to_s16(drwav_int16* pOut, const drwav_uint8* pIn, size_t sampleCount)
{
    int r;  // 用于存储中间结果的变量
    size_t i;  // 用于循环计数的变量
    for (i = 0; i < sampleCount; ++i) {  // 遍历每个样本
        int x = pIn[i];  // 获取输入数组中的值
        r = x << 8;  // 将输入值左移8位
        r = r - 32768;  // 减去32768
        pOut[i] = (short)r;  // 将结果存入输出数组
    }
}

// 将有符号24位整数数组转换为有符号16位整数数组
DRWAV_API void drwav_s24_to_s16(drwav_int16* pOut, const drwav_uint8* pIn, size_t sampleCount)
{
    int r;  // 用于存储中间结果的变量
    size_t i;  // 用于循环计数的变量
    for (i = 0; i < sampleCount; ++i) {  // 遍历每个样本
        int x = ((int)(((unsigned int)(((const drwav_uint8*)pIn)[i*3+0]) << 8) | ((unsigned int)(((const drwav_uint8*)pIn)[i*3+1]) << 16) | ((unsigned int)(((const drwav_uint8*)pIn)[i*3+2])) << 24)) >> 8;  // 复杂的位运算，将输入值转换为有符号整数
        r = x >> 8;  // 右移8位
        pOut[i] = (short)r;  // 将结果存入输出数组
    }
}

// 将有符号32位整数数组转换为有符号16位整数数组
DRWAV_API void drwav_s32_to_s16(drwav_int16* pOut, const drwav_int32* pIn, size_t sampleCount)
{
    int r;  // 用于存储中间结果的变量
    size_t i;  // 用于循环计数的变量
    for (i = 0; i < sampleCount; ++i) {  // 遍历每个样本
        int x = pIn[i];  // 获取输入数组中的值
        r = x >> 16;  // 右移16位
        pOut[i] = (short)r;  // 将结果存入输出数组
    }
}

// 将32位浮点数数组转换为有符号16位整数数组
DRWAV_API void drwav_f32_to_s16(drwav_int16* pOut, const float* pIn, size_t sampleCount)
{
    int r;  // 用于存储中间结果的变量
    size_t i;  // 用于循环计数的变量
    for (i = 0; i < sampleCount; ++i) {  // 遍历每个样本
        float x = pIn[i];  // 获取输入数组中的值
        float c;  // 用于存储中间结果的变量
        c = ((x < -1) ? -1 : ((x > 1) ? 1 : x));  // 对输入值进行限制在[-1, 1]范围内
        c = c + 1;  // 加1
        r = (int)(c * 32767.5f);  // 乘以32767.5并转换为整数
        r = r - 32768;  // 减去32768
        pOut[i] = (short)r;  // 将结果存入输出数组
    }
}

// 将64位浮点数数组转换为有符号16位整数数组
DRWAV_API void drwav_f64_to_s16(drwav_int16* pOut, const double* pIn, size_t sampleCount)
{
    int r;  // 用于存储中间结果的变量
    size_t i;  // 用于循环计数的变量
    for (i = 0; i < sampleCount; ++i) {  // 遍历每个样本
        double x = pIn[i];  // 获取输入数组中的值
        double c;  // 用于存储中间结果的变量
        c = ((x < -1) ? -1 : ((x > 1) ? 1 : x));  // 对输入值进行限制在[-1, 1]范围内
        c = c + 1;  // 加1
        r = (int)(c * 32767.5);  // 乘以32767.5并转换为整数
        r = r - 32768;  // 减去32768
        pOut[i] = (short)r;  // 将结果存入输出数组
    }
}

// 将A-law编码数组转换为有符号16位整数数组
DRWAV_API void drwav_alaw_to_s16(drwav_int16* pOut, const drwav_uint8* pIn, size_t sampleCount)
{
    size_t i;  // 用于循环计数的变量
    for (i = 0; i < sampleCount; ++i) {  // 遍历每个样本
        pOut[i] = drwav__alaw_to_s16(pIn[i]);  // 调用内部函数将A-law编码转换为有符号16位整数
    }
}

// 将μ-law编码数组转换为有符号16位整数数组
DRWAV_API void drwav_mulaw_to_s16(drwav_int16* pOut, const drwav_uint8* pIn, size_t sampleCount)
{
    size_t i;  // 用于循环计数的变量
    # 遍历从 0 到 sampleCount-1 的整数
    for (i = 0; i < sampleCount; ++i) {
        # 将 pIn[i] 中的 mu-law 编码转换为有符号 16 位整数，并赋值给 pOut[i]
        pOut[i] = drwav__mulaw_to_s16(pIn[i]);
    }
}
// 将 PCM 格式的音频数据转换为 32 位浮点数格式
static void drwav__pcm_to_f32(float* pOut, const drwav_uint8* pIn, size_t sampleCount, unsigned int bytesPerSample)
{
    unsigned int i;

    /* 对于 8 位样本数据，因为它被视为无符号数，所以有特殊处理 */
    if (bytesPerSample == 1) {
        drwav_u8_to_f32(pOut, pIn, sampleCount);
        return;
    }

    /* 对于常见格式，有稍微更优化的实现 */
    if (bytesPerSample == 2) {
        drwav_s16_to_f32(pOut, (const drwav_int16*)pIn, sampleCount);
        return;
    }
    if (bytesPerSample == 3) {
        drwav_s24_to_f32(pOut, pIn, sampleCount);
        return;
    }
    if (bytesPerSample == 4) {
        drwav_s32_to_f32(pOut, (const drwav_int32*)pIn, sampleCount);
        return;
    }

    /* 不支持超过 64 位每样本的数据 */
    if (bytesPerSample > 8) {
        DRWAV_ZERO_MEMORY(pOut, sampleCount * sizeof(*pOut));
        return;
    }

    /* 通用、较慢的转换器 */
    for (i = 0; i < sampleCount; ++i) {
        drwav_uint64 sample = 0;
        unsigned int shift  = (8 - bytesPerSample) * 8;

        unsigned int j;
        for (j = 0; j < bytesPerSample; j += 1) {
            DRWAV_ASSERT(j < 8);
            sample |= (drwav_uint64)(pIn[j]) << shift;
            shift  += 8;
        }

        pIn += j;
        *pOut++ = (float)((drwav_int64)sample / 9223372036854775807.0);
    }
}

// 将 IEEE 格式的音频数据转换为 32 位浮点数格式
static void drwav__ieee_to_f32(float* pOut, const drwav_uint8* pIn, size_t sampleCount, unsigned int bytesPerSample)
{
    if (bytesPerSample == 4) {
        unsigned int i;
        for (i = 0; i < sampleCount; ++i) {
            *pOut++ = ((const float*)pIn)[i];
        }
        return;
    } else if (bytesPerSample == 8) {
        drwav_f64_to_f32(pOut, (const double*)pIn, sampleCount);
        return;
    } else {
        /* 只支持32位和64位浮点数。在所有其他情况下输出静音。欢迎为16位浮点数做出贡献。 */
        // 用零填充输出数组，大小为样本数乘以每个样本的大小
        DRWAV_ZERO_MEMORY(pOut, sampleCount * sizeof(*pOut));
        // 返回
        return;
    }
    # 读取 PCM 格式的音频帧数据并转换为 32 位浮点数格式
    static drwav_uint64 drwav_read_pcm_frames_f32__pcm(drwav* pWav, drwav_uint64 framesToRead, float* pBufferOut)
    {
        drwav_uint64 totalFramesRead;  // 总共读取的帧数
        drwav_uint8 sampleData[4096];  // 用于存储样本数据的缓冲区

        drwav_uint32 bytesPerFrame = drwav_get_bytes_per_pcm_frame(pWav);  // 获取每帧的字节数
        if (bytesPerFrame == 0) {  // 如果每帧字节数为 0，则返回 0
            return 0;
        }

        totalFramesRead = 0;  // 初始化总帧数为 0

        while (framesToRead > 0) {  // 循环读取帧数据，直到读取完指定的帧数
            drwav_uint64 framesRead = drwav_read_pcm_frames(pWav, drwav_min(framesToRead, sizeof(sampleData)/bytesPerFrame), sampleData);  // 读取 PCM 格式的帧数据
            if (framesRead == 0) {  // 如果读取的帧数为 0，则跳出循环
                break;
            }

            drwav__pcm_to_f32(pBufferOut, sampleData, (size_t)framesRead*pWav->channels, bytesPerFrame/pWav->channels);  // 将 PCM 格式的帧数据转换为 32 位浮点数格式

            pBufferOut      += framesRead*pWav->channels;  // 更新输出缓冲区的位置
            framesToRead    -= framesRead;  // 更新剩余需要读取的帧数
            totalFramesRead += framesRead;  // 更新总共读取的帧数
        }

        return totalFramesRead;  // 返回总共读取的帧数
    }

    # 读取 MSADPCM 格式的音频帧数据并转换为 32 位浮点数格式
    static drwav_uint64 drwav_read_pcm_frames_f32__msadpcm(drwav* pWav, drwav_uint64 framesToRead, float* pBufferOut)
    {
        /*
        We're just going to borrow the implementation from the drwav_read_s16() since ADPCM is a little bit more complicated than other formats and I don't
        want to duplicate that code.
        */
        drwav_uint64 totalFramesRead = 0;  // 总共读取的帧数
        drwav_int16 samples16[2048];  // 用于存储 16 位样本数据的缓冲区
        while (framesToRead > 0) {  // 循环读取帧数据，直到读取完指定的帧数
            drwav_uint64 framesRead = drwav_read_pcm_frames_s16(pWav, drwav_min(framesToRead, drwav_countof(samples16)/pWav->channels), samples16);  // 读取 MSADPCM 格式的帧数据
            if (framesRead == 0) {  // 如果读取的帧数为 0，则跳出循环
                break;
            }

            drwav_s16_to_f32(pBufferOut, samples16, (size_t)(framesRead*pWav->channels));   /* <-- Safe cast because we're clamping to 2048. */  // 将 16 位样本数据转换为 32 位浮点数格式

            pBufferOut      += framesRead*pWav->channels;  // 更新输出缓冲区的位置
            framesToRead    -= framesRead;  // 更新剩余需要读取的帧数
            totalFramesRead += framesRead;  // 更新总共读取的帧数
        }

        return totalFramesRead;  // 返回总共读取的帧数
    }

    # 读取 IMA 格式的音频帧数据并转换为 32 位浮点数格式
    static drwav_uint64 drwav_read_pcm_frames_f32__ima(drwav* pWav, drwav_uint64 framesToRead, float* pBufferOut)
    {
        /*
        We're just going to borrow the implementation from the drwav_read_s16() since ADPCM is a little bit more complicated than other formats and I don't
        want to duplicate that code.
        */
        drwav_uint64 totalFramesRead = 0;  // 总共读取的帧数
        drwav_int16 samples16[2048];  // 用于存储 16 位样本数据的缓冲区
        while (framesToRead > 0) {  // 循环读取帧数据，直到读取完指定的帧数
            drwav_uint64 framesRead = drwav_read_pcm_frames_s16(pWav, drwav_min(framesToRead, drwav_countof(samples16)/pWav->channels), samples16);  // 读取 IMA 格式的帧数据
            if (framesRead == 0) {  // 如果读取的帧数为 0，则跳出循环
                break;
            }

            drwav_s16_to_f32(pBufferOut, samples16, (size_t)(framesRead*pWav->channels));   /* <-- Safe cast because we're clamping to 2048. */  // 将 16 位样本数据转换为 32 位浮点数格式

            pBufferOut      += framesRead*pWav->channels;  // 更新输出缓冲区的位置
            framesToRead    -= framesRead;  // 更新剩余需要读取的帧数
            totalFramesRead += framesRead;  // 更新总共读取的帧数
        }

        return totalFramesRead;  // 返回总共读取的帧数
    }
    /*
    从 drwav_read_s16() 中借用实现，因为 IMA-ADPCM 比其他格式复杂一些，我不想重复那段代码。
    */
    // 初始化已读取的总帧数为 0
    drwav_uint64 totalFramesRead = 0;
    // 创建一个长度为 2048 的 int16 数组
    drwav_int16 samples16[2048];
    // 当需要读取的帧数大于 0 时，进入循环
    while (framesToRead > 0) {
        // 读取 PCM 帧数据到 samples16 数组中
        drwav_uint64 framesRead = drwav_read_pcm_frames_s16(pWav, drwav_min(framesToRead, drwav_countof(samples16)/pWav->channels), samples16);
        // 如果没有读取到帧数据，跳出循环
        if (framesRead == 0) {
            break;
        }

        // 将 int16 类型的 samples16 数组转换为 float32 类型的 pBufferOut 数组
        drwav_s16_to_f32(pBufferOut, samples16, (size_t)(framesRead*pWav->channels));   /* <-- Safe cast because we're clamping to 2048. */

        // 更新 pBufferOut 指针位置
        pBufferOut      += framesRead*pWav->channels;
        // 更新剩余需要读取的帧数
        framesToRead    -= framesRead;
        // 更新已读取的总帧数
        totalFramesRead += framesRead;
    }

    // 返回已读取的总帧数
    return totalFramesRead;
}


static drwav_uint64 drwav_read_pcm_frames_f32__ieee(drwav* pWav, drwav_uint64 framesToRead, float* pBufferOut)
{
    drwav_uint64 totalFramesRead;  // 用于记录总共读取的帧数
    drwav_uint8 sampleData[4096];  // 用于存储采样数据的缓冲区
    drwav_uint32 bytesPerFrame;  // 每帧的字节数

    /* Fast path. */
    // 如果是 IEEE_FLOAT 格式并且每个采样是 32 位，则直接调用 drwav_read_pcm_frames 函数
    if (pWav->translatedFormatTag == DR_WAVE_FORMAT_IEEE_FLOAT && pWav->bitsPerSample == 32) {
        return drwav_read_pcm_frames(pWav, framesToRead, pBufferOut);
    }
    
    bytesPerFrame = drwav_get_bytes_per_pcm_frame(pWav);  // 获取每帧的字节数
    if (bytesPerFrame == 0) {  // 如果每帧的字节数为 0，则返回 0
        return 0;
    }

    totalFramesRead = 0;  // 初始化总共读取的帧数为 0

    while (framesToRead > 0) {  // 循环读取帧数大于 0
        drwav_uint64 framesRead = drwav_read_pcm_frames(pWav, drwav_min(framesToRead, sizeof(sampleData)/bytesPerFrame), sampleData);  // 读取帧数
        if (framesRead == 0) {  // 如果读取的帧数为 0，则跳出循环
            break;
        }

        drwav__ieee_to_f32(pBufferOut, sampleData, (size_t)(framesRead*pWav->channels), bytesPerFrame/pWav->channels);  // 将采样数据转换为 32 位浮点数

        pBufferOut      += framesRead*pWav->channels;  // 更新输出缓冲区的位置
        framesToRead    -= framesRead;  // 更新剩余需要读取的帧数
        totalFramesRead += framesRead;  // 更新总共读取的帧数
    }

    return totalFramesRead;  // 返回总共读取的帧数
}

static drwav_uint64 drwav_read_pcm_frames_f32__alaw(drwav* pWav, drwav_uint64 framesToRead, float* pBufferOut)
{
    drwav_uint64 totalFramesRead;  // 用于记录总共读取的帧数
    drwav_uint8 sampleData[4096];  // 用于存储采样数据的缓冲区
    drwav_uint32 bytesPerFrame = drwav_get_bytes_per_pcm_frame(pWav);  // 获取每帧的字节数
    if (bytesPerFrame == 0) {  // 如果每帧的字节数为 0，则返回 0
        return 0;
    }

    totalFramesRead = 0;  // 初始化总共读取的帧数为 0

    while (framesToRead > 0) {  // 循环读取帧数大于 0
        drwav_uint64 framesRead = drwav_read_pcm_frames(pWav, drwav_min(framesToRead, sizeof(sampleData)/bytesPerFrame), sampleData);  // 读取帧数
        if (framesRead == 0) {  // 如果读取的帧数为 0，则跳出循环
            break;
        }

        drwav_alaw_to_f32(pBufferOut, sampleData, (size_t)(framesRead*pWav->channels));  // 将采样数据转换为 32 位浮点数

        pBufferOut      += framesRead*pWav->channels;  // 更新输出缓冲区的位置
        framesToRead    -= framesRead;  // 更新剩余需要读取的帧数
        totalFramesRead += framesRead;  // 更新总共读取的帧数
    }

    return totalFramesRead;  // 返回总共读取的帧数
}

static drwav_uint64 drwav_read_pcm_frames_f32__mulaw(drwav* pWav, drwav_uint64 framesToRead, float* pBufferOut)
    # 定义一个变量用于存储读取的总帧数
    drwav_uint64 totalFramesRead;
    # 定义一个数组用于存储采样数据
    drwav_uint8 sampleData[4096];

    # 获取每个 PCM 帧的字节数
    drwav_uint32 bytesPerFrame = drwav_get_bytes_per_pcm_frame(pWav);
    # 如果每个帧的字节数为 0，则返回 0
    if (bytesPerFrame == 0) {
        return 0;
    }

    # 初始化总帧数为 0
    totalFramesRead = 0;

    # 当需要读取的帧数大于 0 时，执行循环
    while (framesToRead > 0) {
        # 读取 PCM 帧数据
        drwav_uint64 framesRead = drwav_read_pcm_frames(pWav, drwav_min(framesToRead, sizeof(sampleData)/bytesPerFrame), sampleData);
        # 如果没有读取到帧数据，则跳出循环
        if (framesRead == 0) {
            break;
        }

        # 将 mu-law 编码的数据转换为 32 位浮点数
        drwav_mulaw_to_f32(pBufferOut, sampleData, (size_t)(framesRead*pWav->channels));

        # 更新输出缓冲区指针和剩余需要读取的帧数
        pBufferOut      += framesRead*pWav->channels;
        framesToRead    -= framesRead;
        # 更新总帧数
        totalFramesRead += framesRead;
    }

    # 返回总帧数
    return totalFramesRead;
}
# 读取 32 位浮点 PCM 格式的音频数据
DRWAV_API drwav_uint64 drwav_read_pcm_frames_f32(drwav* pWav, drwav_uint64 framesToRead, float* pBufferOut)
{
    # 如果音频流为空或者要读取的帧数为 0，则返回 0
    if (pWav == NULL || framesToRead == 0) {
        return 0;
    }

    # 如果输出缓冲区为空，则调用 drwav_read_pcm_frames 函数
    if (pBufferOut == NULL) {
        return drwav_read_pcm_frames(pWav, framesToRead, NULL);
    }

    /* 不要尝试读取超过输出缓冲区可能容纳的样本数。 */
    if (framesToRead * pWav->channels * sizeof(float) > DRWAV_SIZE_MAX) {
        framesToRead = DRWAV_SIZE_MAX / sizeof(float) / pWav->channels;
    }

    # 根据音频流的格式标签调用相应的读取函数
    if (pWav->translatedFormatTag == DR_WAVE_FORMAT_PCM) {
        return drwav_read_pcm_frames_f32__pcm(pWav, framesToRead, pBufferOut);
    }

    if (pWav->translatedFormatTag == DR_WAVE_FORMAT_ADPCM) {
        return drwav_read_pcm_frames_f32__msadpcm(pWav, framesToRead, pBufferOut);
    }

    if (pWav->translatedFormatTag == DR_WAVE_FORMAT_IEEE_FLOAT) {
        return drwav_read_pcm_frames_f32__ieee(pWav, framesToRead, pBufferOut);
    }

    if (pWav->translatedFormatTag == DR_WAVE_FORMAT_ALAW) {
        return drwav_read_pcm_frames_f32__alaw(pWav, framesToRead, pBufferOut);
    }

    if (pWav->translatedFormatTag == DR_WAVE_FORMAT_MULAW) {
        return drwav_read_pcm_frames_f32__mulaw(pWav, framesToRead, pBufferOut);
    }

    if (pWav->translatedFormatTag == DR_WAVE_FORMAT_DVI_ADPCM) {
        return drwav_read_pcm_frames_f32__ima(pWav, framesToRead, pBufferOut);
    }

    return 0;
}

# 读取 32 位浮点 PCM 格式的小端字节序音频数据
DRWAV_API drwav_uint64 drwav_read_pcm_frames_f32le(drwav* pWav, drwav_uint64 framesToRead, float* pBufferOut)
{
    # 调用 drwav_read_pcm_frames_f32 函数读取音频数据
    drwav_uint64 framesRead = drwav_read_pcm_frames_f32(pWav, framesToRead, pBufferOut);
    # 如果输出缓冲区不为空且系统是大端字节序，则交换字节序
    if (pBufferOut != NULL && drwav__is_little_endian() == DRWAV_FALSE) {
        drwav__bswap_samples_f32(pBufferOut, framesRead*pWav->channels);
    }

    return framesRead;
}

# 读取 32 位浮点 PCM 格式的大端字节序音频数据
DRWAV_API drwav_uint64 drwav_read_pcm_frames_f32be(drwav* pWav, drwav_uint64 framesToRead, float* pBufferOut)
{
    # 从 WAV 文件中读取指定数量的 PCM 帧，返回实际读取的帧数
    drwav_uint64 framesRead = drwav_read_pcm_frames_f32(pWav, framesToRead, pBufferOut);
    # 检查输出缓冲区是否存在并且系统是否为小端序
    if (pBufferOut != NULL && drwav__is_little_endian() == DRWAV_TRUE) {
        # 如果是小端序，交换输出缓冲区中的 32 位浮点数样本的字节顺序
        drwav__bswap_samples_f32(pBufferOut, framesRead*pWav->channels);
    }

    # 返回实际读取的帧数
    return framesRead;
}
// 将无符号8位整数数组转换为32位浮点数数组
DRWAV_API void drwav_u8_to_f32(float* pOut, const drwav_uint8* pIn, size_t sampleCount)
{
    size_t i;

    // 检查输入指针是否为空，如果为空则返回
    if (pOut == NULL || pIn == NULL) {
        return;
    }

#ifdef DR_WAV_LIBSNDFILE_COMPAT
    /*
    看起来libsndfile对于u8到f32的转换逻辑与dr_wav略有不同，我认为是不正确的。看起来libsndfile执行的转换类似于"f32 = (u8 / 256) * 2 - 1"，然而我认为应该是"f32 = (u8 / 255) * 2 - 1"（注意除数为256与255）。我使用libsndfile作为测试的基准，因此我将这个块留在这里只是为了我的自动正确性测试。默认情况下禁用。
    */
    for (i = 0; i < sampleCount; ++i) {
        *pOut++ = (pIn[i] / 256.0f) * 2 - 1;
    }
#else
    for (i = 0; i < sampleCount; ++i) {
        float x = pIn[i];
        x = x * 0.00784313725490196078f;    /* 0..255 to 0..2 */
        x = x - 1;                          /* 0..2 to -1..1 */

        *pOut++ = x;
    }
#endif
}

// 将有符号16位整数数组转换为32位浮点数数组
DRWAV_API void drwav_s16_to_f32(float* pOut, const drwav_int16* pIn, size_t sampleCount)
{
    size_t i;

    // 检查输入指针是否为空，如果为空则返回
    if (pOut == NULL || pIn == NULL) {
        return;
    }

    for (i = 0; i < sampleCount; ++i) {
        *pOut++ = pIn[i] * 0.000030517578125f;
    }
}

// 将有符号24位整数数组转换为32位浮点数数组
DRWAV_API void drwav_s24_to_f32(float* pOut, const drwav_uint8* pIn, size_t sampleCount)
{
    size_t i;

    // 检查输入指针是否为空，如果为空则返回
    if (pOut == NULL || pIn == NULL) {
        return;
    }

    for (i = 0; i < sampleCount; ++i) {
        double x;
        drwav_uint32 a = ((drwav_uint32)(pIn[i*3+0]) <<  8);
        drwav_uint32 b = ((drwav_uint32)(pIn[i*3+1]) << 16);
        drwav_uint32 c = ((drwav_uint32)(pIn[i*3+2]) << 24);

        x = (double)((drwav_int32)(a | b | c) >> 8);
        *pOut++ = (float)(x * 0.00000011920928955078125);
    }
}

// 将有符号32位整数数组转换为32位浮点数数组
DRWAV_API void drwav_s32_to_f32(float* pOut, const drwav_int32* pIn, size_t sampleCount)
{
    size_t i;
    // 检查输入指针是否为空，如果为空则返回
    if (pOut == NULL || pIn == NULL) {
        return;
    }
    # 遍历从 0 到 sampleCount 的索引
    for (i = 0; i < sampleCount; ++i) {
        # 将 pIn[i] 的值除以 2147483648.0 转换为浮点数，然后赋值给 pOut，并将指针向后移动一位
        *pOut++ = (float)(pIn[i] / 2147483648.0);
    }
}

DRWAV_API void drwav_f64_to_f32(float* pOut, const double* pIn, size_t sampleCount)
{
    size_t i;

    if (pOut == NULL || pIn == NULL) {
        return;  // 如果输出指针或输入指针为空，则直接返回
    }

    for (i = 0; i < sampleCount; ++i) {
        *pOut++ = (float)pIn[i];  // 将双精度浮点数转换为单精度浮点数
    }
}

DRWAV_API void drwav_alaw_to_f32(float* pOut, const drwav_uint8* pIn, size_t sampleCount)
{
    size_t i;

    if (pOut == NULL || pIn == NULL) {
        return;  // 如果输出指针或输入指针为空，则直接返回
    }

    for (i = 0; i < sampleCount; ++i) {
        *pOut++ = drwav__alaw_to_s16(pIn[i]) / 32768.0f;  // 将 A-law 编码转换为单精度浮点数
    }
}

DRWAV_API void drwav_mulaw_to_f32(float* pOut, const drwav_uint8* pIn, size_t sampleCount)
{
    size_t i;

    if (pOut == NULL || pIn == NULL) {
        return;  // 如果输出指针或输入指针为空，则直接返回
    }

    for (i = 0; i < sampleCount; ++i) {
        *pOut++ = drwav__mulaw_to_s16(pIn[i]) / 32768.0f;  // 将 μ-law 编码转换为单精度浮点数
    }
}



static void drwav__pcm_to_s32(drwav_int32* pOut, const drwav_uint8* pIn, size_t totalSampleCount, unsigned int bytesPerSample)
{
    unsigned int i;

    /* Special case for 8-bit sample data because it's treated as unsigned. */
    if (bytesPerSample == 1) {
        drwav_u8_to_s32(pOut, pIn, totalSampleCount);  // 处理8位样本数据
        return;
    }

    /* Slightly more optimal implementation for common formats. */
    if (bytesPerSample == 2) {
        drwav_s16_to_s32(pOut, (const drwav_int16*)pIn, totalSampleCount);  // 处理16位样本数据
        return;
    }
    if (bytesPerSample == 3) {
        drwav_s24_to_s32(pOut, pIn, totalSampleCount);  // 处理24位样本数据
        return;
    }
    if (bytesPerSample == 4) {
        for (i = 0; i < totalSampleCount; ++i) {
           *pOut++ = ((const drwav_int32*)pIn)[i];  // 处理32位样本数据
        }
        return;
    }


    /* Anything more than 64 bits per sample is not supported. */
    if (bytesPerSample > 8) {
        DRWAV_ZERO_MEMORY(pOut, totalSampleCount * sizeof(*pOut));  // 不支持超过64位的样本数据
        return;
    }


    /* Generic, slow converter. */
}
    # 遍历总样本数次数
    for (i = 0; i < totalSampleCount; ++i) {
        # 初始化样本值为0
        drwav_uint64 sample = 0;
        # 计算位移量，用于将字节转换为样本值
        unsigned int shift  = (8 - bytesPerSample) * 8;

        # 遍历每个字节
        unsigned int j;
        for (j = 0; j < bytesPerSample; j += 1) {
            # 断言确保字节索引在合理范围内
            DRWAV_ASSERT(j < 8);
            # 将字节转换为样本值，并根据位移量进行位运算
            sample |= (drwav_uint64)(pIn[j]) << shift;
            shift  += 8;
        }

        # 更新输入指针位置
        pIn += j;
        # 将样本值转换为32位有符号整数，并存储到输出指针指向的位置
        *pOut++ = (drwav_int32)((drwav_int64)sample >> 32);
    }
# 将 IEEE 格式的数据转换为 32 位有符号整数
static void drwav__ieee_to_s32(drwav_int32* pOut, const drwav_uint8* pIn, size_t totalSampleCount, unsigned int bytesPerSample)
{
    # 如果每个样本占用 4 个字节，则调用 drwav_f32_to_s32() 函数进行转换
    if (bytesPerSample == 4) {
        drwav_f32_to_s32(pOut, (const float*)pIn, totalSampleCount);
        return;
    } 
    # 如果每个样本占用 8 个字节，则调用 drwav_f64_to_s32() 函数进行转换
    else if (bytesPerSample == 8) {
        drwav_f64_to_s32(pOut, (const double*)pIn, totalSampleCount);
        return;
    } 
    # 如果不是 4 或 8 个字节，则输出静音
    else {
        /* Only supporting 32- and 64-bit float. Output silence in all other cases. Contributions welcome for 16-bit float. */
        DRWAV_ZERO_MEMORY(pOut, totalSampleCount * sizeof(*pOut));
        return;
    }
}

# 读取 PCM 格式的 32 位有符号整数数据帧
static drwav_uint64 drwav_read_pcm_frames_s32__pcm(drwav* pWav, drwav_uint64 framesToRead, drwav_int32* pBufferOut)
{
    drwav_uint64 totalFramesRead;
    drwav_uint8 sampleData[4096];
    drwav_uint32 bytesPerFrame;

    # 如果是 PCM 格式且每个样本占用 32 位，则调用 drwav_read_pcm_frames() 函数进行读取
    /* Fast path. */
    if (pWav->translatedFormatTag == DR_WAVE_FORMAT_PCM && pWav->bitsPerSample == 32) {
        return drwav_read_pcm_frames(pWav, framesToRead, pBufferOut);
    }
    
    bytesPerFrame = drwav_get_bytes_per_pcm_frame(pWav);
    if (bytesPerFrame == 0) {
        return 0;
    }

    totalFramesRead = 0;

    while (framesToRead > 0) {
        drwav_uint64 framesRead = drwav_read_pcm_frames(pWav, drwav_min(framesToRead, sizeof(sampleData)/bytesPerFrame), sampleData);
        if (framesRead == 0) {
            break;
        }

        drwav__pcm_to_s32(pBufferOut, sampleData, (size_t)(framesRead*pWav->channels), bytesPerFrame/pWav->channels);

        pBufferOut      += framesRead*pWav->channels;
        framesToRead    -= framesRead;
        totalFramesRead += framesRead;
    }

    return totalFramesRead;
}

# 读取 MSADPCM 格式的 32 位有符号整数数据帧
static drwav_uint64 drwav_read_pcm_frames_s32__msadpcm(drwav* pWav, drwav_uint64 framesToRead, drwav_int32* pBufferOut)
{
    # 由于 ADPCM 比其他格式复杂，因此直接借用 drwav_read_s16() 的实现
    /*
    We're just going to borrow the implementation from the drwav_read_s16() since ADPCM is a little bit more complicated than other formats and I don't
    want to duplicate that code.
    */
}
    # 初始化已读取的总帧数
    drwav_uint64 totalFramesRead = 0;
    # 创建一个包含2048个int16类型的数组
    drwav_int16 samples16[2048];
    # 当需要读取的帧数大于0时，执行循环
    while (framesToRead > 0) {
        # 从WAV文件中读取PCM帧数据，并将结果保存到samples16数组中
        drwav_uint64 framesRead = drwav_read_pcm_frames_s16(pWav, drwav_min(framesToRead, drwav_countof(samples16)/pWav->channels), samples16);
        # 如果没有读取到帧数据，则跳出循环
        if (framesRead == 0) {
            break;
        }

        # 将int16类型的样本数据转换为int32类型，并保存到pBufferOut中
        drwav_s16_to_s32(pBufferOut, samples16, (size_t)(framesRead*pWav->channels));   /* <-- Safe cast because we're clamping to 2048. */

        # 更新pBufferOut的位置
        pBufferOut      += framesRead*pWav->channels;
        # 更新需要读取的帧数
        framesToRead    -= framesRead;
        # 更新已读取的总帧数
        totalFramesRead += framesRead;
    }

    # 返回已读取的总帧数
    return totalFramesRead;
}

static drwav_uint64 drwav_read_pcm_frames_s32__ima(drwav* pWav, drwav_uint64 framesToRead, drwav_int32* pBufferOut)
{
    /*
    我们将借用 drwav_read_s16() 的实现，因为 IMA-ADPCM 比其他格式复杂一些，我不想重复编写那些代码。
    */
    drwav_uint64 totalFramesRead = 0;  // 初始化总帧数为0
    drwav_int16 samples16[2048];  // 创建一个包含2048个int16类型的数组
    while (framesToRead > 0) {  // 当需要读取的帧数大于0时执行循环
        drwav_uint64 framesRead = drwav_read_pcm_frames_s16(pWav, drwav_min(framesToRead, drwav_countof(samples16)/pWav->channels), samples16);  // 读取PCM帧并存储到samples16数组中
        if (framesRead == 0) {  // 如果读取的帧数为0，则跳出循环
            break;
        }

        drwav_s16_to_s32(pBufferOut, samples16, (size_t)(framesRead*pWav->channels));   /* <-- Safe cast because we're clamping to 2048. */
        // 将int16类型的数据转换为int32类型的数据，并存储到pBufferOut中

        pBufferOut      += framesRead*pWav->channels;  // 更新pBufferOut的位置
        framesToRead    -= framesRead;  // 更新需要读取的帧数
        totalFramesRead += framesRead;  // 更新总帧数
    }

    return totalFramesRead;  // 返回总帧数
}

static drwav_uint64 drwav_read_pcm_frames_s32__ieee(drwav* pWav, drwav_uint64 framesToRead, drwav_int32* pBufferOut)
{
    drwav_uint64 totalFramesRead;  // 初始化总帧数
    drwav_uint8 sampleData[4096];  // 创建一个包含4096个uint8类型的数组

    drwav_uint32 bytesPerFrame = drwav_get_bytes_per_pcm_frame(pWav);  // 获取每帧的字节数
    if (bytesPerFrame == 0) {  // 如果每帧的字节数为0，则返回0
        return 0;
    }

    totalFramesRead = 0;  // 初始化总帧数为0

    while (framesToRead > 0) {  // 当需要读取的帧数大于0时执行循环
        drwav_uint64 framesRead = drwav_read_pcm_frames(pWav, drwav_min(framesToRead, sizeof(sampleData)/bytesPerFrame), sampleData);  // 读取PCM帧并存储到sampleData数组中
        if (framesRead == 0) {  // 如果读取的帧数为0，则跳出循环
            break;
        }

        drwav__ieee_to_s32(pBufferOut, sampleData, (size_t)(framesRead*pWav->channels), bytesPerFrame/pWav->channels);
        // 将IEEE格式的数据转换为int32类型的数据，并存储到pBufferOut中

        pBufferOut      += framesRead*pWav->channels;  // 更新pBufferOut的位置
        framesToRead    -= framesRead;  // 更新需要读取的帧数
        totalFramesRead += framesRead;  // 更新总帧数
    }

    return totalFramesRead;  // 返回总帧数
}

static drwav_uint64 drwav_read_pcm_frames_s32__alaw(drwav* pWav, drwav_uint64 framesToRead, drwav_int32* pBufferOut)
{
    drwav_uint64 totalFramesRead;  // 初始化总帧数
    drwav_uint8 sampleData[4096];  // 创建一个包含4096个uint8类型的数组
    # 获取每个 PCM 帧的字节数
    drwav_uint32 bytesPerFrame = drwav_get_bytes_per_pcm_frame(pWav);
    # 如果每个 PCM 帧的字节数为 0，则返回 0
    if (bytesPerFrame == 0) {
        return 0;
    }

    # 初始化已读取的总帧数
    totalFramesRead = 0;

    # 当需要读取的帧数大于 0 时，执行循环
    while (framesToRead > 0) {
        # 从 WAV 文件中读取 PCM 帧数据
        drwav_uint64 framesRead = drwav_read_pcm_frames(pWav, drwav_min(framesToRead, sizeof(sampleData)/bytesPerFrame), sampleData);
        # 如果未成功读取任何帧，则跳出循环
        if (framesRead == 0) {
            break;
        }

        # 将 ALAW 格式的数据转换为有符号 32 位整数格式
        drwav_alaw_to_s32(pBufferOut, sampleData, (size_t)(framesRead*pWav->channels));

        # 更新输出缓冲区指针位置
        pBufferOut      += framesRead*pWav->channels;
        # 更新剩余需要读取的帧数
        framesToRead    -= framesRead;
        # 更新已读取的总帧数
        totalFramesRead += framesRead;
    }

    # 返回已读取的总帧数
    return totalFramesRead;
}

static drwav_uint64 drwav_read_pcm_frames_s32__mulaw(drwav* pWav, drwav_uint64 framesToRead, drwav_int32* pBufferOut)
{
    drwav_uint64 totalFramesRead;  // 用于记录总共读取的帧数
    drwav_uint8 sampleData[4096];  // 用于存储采样数据的缓冲区

    drwav_uint32 bytesPerFrame = drwav_get_bytes_per_pcm_frame(pWav);  // 获取每个 PCM 帧的字节数
    if (bytesPerFrame == 0) {  // 如果每个 PCM 帧的字节数为 0，则返回 0
        return 0;
    }

    totalFramesRead = 0;  // 初始化总共读取的帧数为 0

    while (framesToRead > 0) {  // 当还有帧需要读取时循环
        drwav_uint64 framesRead = drwav_read_pcm_frames(pWav, drwav_min(framesToRead, sizeof(sampleData)/bytesPerFrame), sampleData);  // 读取 PCM 帧数据
        if (framesRead == 0) {  // 如果没有读取到任何帧，则跳出循环
            break;
        }

        drwav_mulaw_to_s32(pBufferOut, sampleData, (size_t)(framesRead*pWav->channels));  // 将 mu-law 编码转换为 32 位有符号整数

        pBufferOut      += framesRead*pWav->channels;  // 更新输出缓冲区指针
        framesToRead    -= framesRead;  // 更新剩余需要读取的帧数
        totalFramesRead += framesRead;  // 更新总共读取的帧数
    }

    return totalFramesRead;  // 返回总共读取的帧数
}

DRWAV_API drwav_uint64 drwav_read_pcm_frames_s32(drwav* pWav, drwav_uint64 framesToRead, drwav_int32* pBufferOut)
{
    if (pWav == NULL || framesToRead == 0) {  // 如果 WAV 对象为空，或者需要读取的帧数为 0，则返回 0
        return 0;
    }

    if (pBufferOut == NULL) {  // 如果输出缓冲区为空，则调用 drwav_read_pcm_frames 函数
        return drwav_read_pcm_frames(pWav, framesToRead, NULL);
    }

    /* Don't try to read more samples than can potentially fit in the output buffer. */
    if (framesToRead * pWav->channels * sizeof(drwav_int32) > DRWAV_SIZE_MAX) {  // 如果需要读取的样本数超过输出缓冲区的最大容量，则调整需要读取的帧数
        framesToRead = DRWAV_SIZE_MAX / sizeof(drwav_int32) / pWav->channels;
    }

    if (pWav->translatedFormatTag == DR_WAVE_FORMAT_PCM) {  // 如果是 PCM 格式，则调用相应的函数进行读取
        return drwav_read_pcm_frames_s32__pcm(pWav, framesToRead, pBufferOut);
    }

    if (pWav->translatedFormatTag == DR_WAVE_FORMAT_ADPCM) {  // 如果是 ADPCM 格式，则调用相应的函数进行读取
        return drwav_read_pcm_frames_s32__msadpcm(pWav, framesToRead, pBufferOut);
    }

    if (pWav->translatedFormatTag == DR_WAVE_FORMAT_IEEE_FLOAT) {  // 如果是 IEEE 浮点格式，则调用相应的函数进行读取
        return drwav_read_pcm_frames_s32__ieee(pWav, framesToRead, pBufferOut);
    }

    if (pWav->translatedFormatTag == DR_WAVE_FORMAT_ALAW) {  // 如果是 A-law 格式，则调用相应的函数进行读取
        return drwav_read_pcm_frames_s32__alaw(pWav, framesToRead, pBufferOut);
    }
}
    # 如果音频文件的格式为 mulaw，则调用相应的函数读取 PCM 帧并返回
    if (pWav->translatedFormatTag == DR_WAVE_FORMAT_MULAW) {
        return drwav_read_pcm_frames_s32__mulaw(pWav, framesToRead, pBufferOut);
    }

    # 如果音频文件的格式为 DVI ADPCM，则调用相应的函数读取 PCM 帧并返回
    if (pWav->translatedFormatTag == DR_WAVE_FORMAT_DVI_ADPCM) {
        return drwav_read_pcm_frames_s32__ima(pWav, framesToRead, pBufferOut);
    }

    # 如果音频文件的格式不是 mulaw 或 DVI ADPCM，则返回 0
    return 0;
}

This is the end of a function or a code block. No specific action is performed here.


DRWAV_API drwav_uint64 drwav_read_pcm_frames_s32le(drwav* pWav, drwav_uint64 framesToRead, drwav_int32* pBufferOut)
{
    drwav_uint64 framesRead = drwav_read_pcm_frames_s32(pWav, framesToRead, pBufferOut);
    if (pBufferOut != NULL && drwav__is_little_endian() == DRWAV_FALSE) {
        drwav__bswap_samples_s32(pBufferOut, framesRead*pWav->channels);
    }

    return framesRead;
}

This function reads 32-bit signed PCM frames in little-endian format.


DRWAV_API drwav_uint64 drwav_read_pcm_frames_s32be(drwav* pWav, drwav_uint64 framesToRead, drwav_int32* pBufferOut)
{
    drwav_uint64 framesRead = drwav_read_pcm_frames_s32(pWav, framesToRead, pBufferOut);
    if (pBufferOut != NULL && drwav__is_little_endian() == DRWAV_TRUE) {
        drwav__bswap_samples_s32(pBufferOut, framesRead*pWav->channels);
    }

    return framesRead;
}

This function reads 32-bit signed PCM frames in big-endian format.


DRWAV_API void drwav_u8_to_s32(drwav_int32* pOut, const drwav_uint8* pIn, size_t sampleCount)
{
    size_t i;

    if (pOut == NULL || pIn == NULL) {
        return;
    }

    for (i = 0; i < sampleCount; ++i) {
        *pOut++ = ((int)pIn[i] - 128) << 24;
    }
}

This function converts 8-bit unsigned PCM samples to 32-bit signed PCM samples.


DRWAV_API void drwav_s16_to_s32(drwav_int32* pOut, const drwav_int16* pIn, size_t sampleCount)
{
    size_t i;

    if (pOut == NULL || pIn == NULL) {
        return;
    }

    for (i = 0; i < sampleCount; ++i) {
        *pOut++ = pIn[i] << 16;
    }
}

This function converts 16-bit signed PCM samples to 32-bit signed PCM samples.


DRWAV_API void drwav_s24_to_s32(drwav_int32* pOut, const drwav_uint8* pIn, size_t sampleCount)
{
    size_t i;

    if (pOut == NULL || pIn == NULL) {
        return;
    }

    for (i = 0; i < sampleCount; ++i) {
        unsigned int s0 = pIn[i*3 + 0];
        unsigned int s1 = pIn[i*3 + 1];
        unsigned int s2 = pIn[i*3 + 2];

        drwav_int32 sample32 = (drwav_int32)((s0 << 8) | (s1 << 16) | (s2 << 24));
        *pOut++ = sample32;
    }
}

This function converts 24-bit signed PCM samples to 32-bit signed PCM samples.


DRWAV_API void drwav_f32_to_s32(drwav_int32* pOut, const float* pIn, size_t sampleCount)
{
    size_t i;

    if (pOut == NULL || pIn == NULL) {
        return;
    }

This function converts 32-bit floating point PCM samples to 32-bit signed PCM samples.
    # 遍历从 0 到 sampleCount-1 的整数
    for (i = 0; i < sampleCount; ++i) {
        # 将 pIn[i] 的值乘以 2147483648.0 转换为 32 位有符号整数，并存入 pOut 指向的位置，然后 pOut 指针向后移动
        *pOut++ = (drwav_int32)(2147483648.0 * pIn[i]);
    }
}

// 将双精度浮点数数组转换为有符号32位整数数组
DRWAV_API void drwav_f64_to_s32(drwav_int32* pOut, const double* pIn, size_t sampleCount)
{
    size_t i;

    // 检查输入指针是否为空
    if (pOut == NULL || pIn == NULL) {
        return;
    }

    // 遍历输入数组，将双精度浮点数转换为有符号32位整数
    for (i = 0; i < sampleCount; ++i) {
        *pOut++ = (drwav_int32)(2147483648.0 * pIn[i]);
    }
}

// 将A-law编码的无符号8位整数数组转换为有符号32位整数数组
DRWAV_API void drwav_alaw_to_s32(drwav_int32* pOut, const drwav_uint8* pIn, size_t sampleCount)
{
    size_t i;

    // 检查输入指针是否为空
    if (pOut == NULL || pIn == NULL) {
        return;
    }

    // 遍历输入数组，将A-law编码的无符号8位整数转换为有符号32位整数
    for (i = 0; i < sampleCount; ++i) {
        *pOut++ = ((drwav_int32)drwav__alaw_to_s16(pIn[i])) << 16;
    }
}

// 将μ-law编码的无符号8位整数数组转换为有符号32位整数数组
DRWAV_API void drwav_mulaw_to_s32(drwav_int32* pOut, const drwav_uint8* pIn, size_t sampleCount)
{
    size_t i;

    // 检查输入指针是否为空
    if (pOut == NULL || pIn == NULL) {
        return;
    }

    // 遍历输入数组，将μ-law编码的无符号8位整数转换为有符号32位整数
    for (i= 0; i < sampleCount; ++i) {
        *pOut++ = ((drwav_int32)drwav__mulaw_to_s16(pIn[i])) << 16;
    }
}

// 读取PCM帧并关闭WAV文件，返回有符号16位整数数组
static drwav_int16* drwav__read_pcm_frames_and_close_s16(drwav* pWav, unsigned int* channels, unsigned int* sampleRate, drwav_uint64* totalFrameCount)
{
    drwav_uint64 sampleDataSize;
    drwav_int16* pSampleData;
    drwav_uint64 framesRead;

    DRWAV_ASSERT(pWav != NULL);

    // 计算样本数据的大小
    sampleDataSize = pWav->totalPCMFrameCount * pWav->channels * sizeof(drwav_int16);
    // 如果样本数据大小超过最大限制，则释放资源并返回空指针
    if (sampleDataSize > DRWAV_SIZE_MAX) {
        drwav_uninit(pWav);
        return NULL;    /* File's too big. */
    }

    // 分配内存并检查是否成功
    pSampleData = (drwav_int16*)drwav__malloc_from_callbacks((size_t)sampleDataSize, &pWav->allocationCallbacks); /* <-- Safe cast due to the check above. */
    if (pSampleData == NULL) {
        drwav_uninit(pWav);
        return NULL;    /* Failed to allocate memory. */
    }

    // 读取PCM帧并检查是否成功
    framesRead = drwav_read_pcm_frames_s16(pWav, (size_t)pWav->totalPCMFrameCount, pSampleData);
    if (framesRead != pWav->totalPCMFrameCount) {
        drwav__free_from_callbacks(pSampleData, &pWav->allocationCallbacks);
        drwav_uninit(pWav);
        return NULL;    /* There was an error reading the samples. */
    }

    // 关闭WAV文件
    drwav_uninit(pWav);
}
    # 如果 sampleRate 参数不为空，则将 pWav 结构体中的 sampleRate 赋值给 sampleRate 指针所指向的变量
    if (sampleRate) {
        *sampleRate = pWav->sampleRate;
    }
    # 如果 channels 参数不为空，则将 pWav 结构体中的 channels 赋值给 channels 指针所指向的变量
    if (channels) {
        *channels = pWav->channels;
    }
    # 如果 totalFrameCount 参数不为空，则将 pWav 结构体中的 totalPCMFrameCount 赋值给 totalFrameCount 指针所指向的变量
    if (totalFrameCount) {
        *totalFrameCount = pWav->totalPCMFrameCount;
    }

    # 返回 pSampleData 指针所指向的数据
    return pSampleData;
    }
    
    // 读取浮点型 PCM 格式的音频帧并关闭 WAV 文件
    static float* drwav__read_pcm_frames_and_close_f32(drwav* pWav, unsigned int* channels, unsigned int* sampleRate, drwav_uint64* totalFrameCount)
    {
        drwav_uint64 sampleDataSize;
        float* pSampleData;
        drwav_uint64 framesRead;

        DRWAV_ASSERT(pWav != NULL);

        // 计算样本数据的大小
        sampleDataSize = pWav->totalPCMFrameCount * pWav->channels * sizeof(float);
        // 如果样本数据大小超过最大限制，则关闭 WAV 文件并返回空指针
        if (sampleDataSize > DRWAV_SIZE_MAX) {
            drwav_uninit(pWav);
            return NULL;    /* File's too big. */
        }

        // 分配内存并读取 PCM 帧数据
        pSampleData = (float*)drwav__malloc_from_callbacks((size_t)sampleDataSize, &pWav->allocationCallbacks); /* <-- Safe cast due to the check above. */
        if (pSampleData == NULL) {
            drwav_uninit(pWav);
            return NULL;    /* Failed to allocate memory. */
        }

        framesRead = drwav_read_pcm_frames_f32(pWav, (size_t)pWav->totalPCMFrameCount, pSampleData);
        // 如果读取的帧数与总帧数不一致，则释放内存、关闭 WAV 文件并返回空指针
        if (framesRead != pWav->totalPCMFrameCount) {
            drwav__free_from_callbacks(pSampleData, &pWav->allocationCallbacks);
            drwav_uninit(pWav);
            return NULL;    /* There was an error reading the samples. */
        }

        // 关闭 WAV 文件
        drwav_uninit(pWav);

        // 如果需要，设置采样率、通道数和总帧数
        if (sampleRate) {
            *sampleRate = pWav->sampleRate;
        }
        if (channels) {
            *channels = pWav->channels;
        }
        if (totalFrameCount) {
            *totalFrameCount = pWav->totalPCMFrameCount;
        }

        // 返回 PCM 帧数据
        return pSampleData;
    }

    // 读取有符号 32 位整型 PCM 格式的音频帧并关闭 WAV 文件
    static drwav_int32* drwav__read_pcm_frames_and_close_s32(drwav* pWav, unsigned int* channels, unsigned int* sampleRate, drwav_uint64* totalFrameCount)
    {
        drwav_uint64 sampleDataSize;
        drwav_int32* pSampleData;
        drwav_uint64 framesRead;

        DRWAV_ASSERT(pWav != NULL);

        // 计算样本数据的大小
        sampleDataSize = pWav->totalPCMFrameCount * pWav->channels * sizeof(drwav_int32);
        // 如果样本数据大小超过最大限制，则关闭 WAV 文件并返回空指针
        if (sampleDataSize > DRWAV_SIZE_MAX) {
            drwav_uninit(pWav);
            return NULL;    /* File's too big. */
        }

        // 分配内存并读取 PCM 帧数据
        pSampleData = (drwav_int32*)drwav__malloc_from_callbacks((size_t)sampleDataSize, &pWav->allocationCallbacks); /* <-- Safe cast due to the check above. */
    # 如果样本数据为空，则释放 WAV 对象并返回空
    if (pSampleData == NULL) {
        drwav_uninit(pWav);
        return NULL;    /* Failed to allocate memory. */
    }

    # 读取 PCM 样本帧
    framesRead = drwav_read_pcm_frames_s32(pWav, (size_t)pWav->totalPCMFrameCount, pSampleData);
    # 如果读取的帧数不等于总帧数，则释放样本数据和 WAV 对象并返回空
    if (framesRead != pWav->totalPCMFrameCount) {
        drwav__free_from_callbacks(pSampleData, &pWav->allocationCallbacks);
        drwav_uninit(pWav);
        return NULL;    /* There was an error reading the samples. */
    }

    # 释放 WAV 对象
    drwav_uninit(pWav);

    # 如果 sampleRate 不为空，则将 WAV 对象的采样率赋值给它
    if (sampleRate) {
        *sampleRate = pWav->sampleRate;
    }
    # 如果 channels 不为空，则将 WAV 对象的声道数赋值给它
    if (channels) {
        *channels = pWav->channels;
    }
    # 如果 totalFrameCount 不为空，则将 WAV 对象的总帧数赋值给它
    if (totalFrameCount) {
        *totalFrameCount = pWav->totalPCMFrameCount;
    }

    # 返回样本数据
    return pSampleData;
# 打开并读取 16 位有符号整数格式的 PCM 数据帧
DRWAV_API drwav_int16* drwav_open_and_read_pcm_frames_s16(drwav_read_proc onRead, drwav_seek_proc onSeek, void* pUserData, unsigned int* channelsOut, unsigned int* sampleRateOut, drwav_uint64* totalFrameCountOut, const drwav_allocation_callbacks* pAllocationCallbacks)
{
    # 创建一个 drwav 结构体
    drwav wav;

    # 如果 channelsOut 不为空，则将其值设为 0
    if (channelsOut) {
        *channelsOut = 0;
    }
    # 如果 sampleRateOut 不为空，则将其值设为 0
    if (sampleRateOut) {
        *sampleRateOut = 0;
    }
    # 如果 totalFrameCountOut 不为空，则将其值设为 0
    if (totalFrameCountOut) {
        *totalFrameCountOut = 0;
    }

    # 初始化 drwav 结构体
    if (!drwav_init(&wav, onRead, onSeek, pUserData, pAllocationCallbacks)) {
        return NULL;
    }

    # 调用内部函数读取 PCM 数据帧并关闭文件
    return drwav__read_pcm_frames_and_close_s16(&wav, channelsOut, sampleRateOut, totalFrameCountOut);
}

# 打开并读取 32 位有符号整数格式的 PCM 数据帧
DRWAV_API drwav_int32* drwav_open_and_read_pcm_frames_s32(drwav_read_proc onRead, drwav_seek_proc onSeek, void* pUserData, unsigned int* channelsOut, unsigned int* sampleRateOut, drwav_uint64* totalFrameCountOut, const drwav_allocation_callbacks* pAllocationCallbacks)
{
    # 创建一个 drwav 结构体
    drwav wav;

    # 如果 channelsOut 不为空，则将其值设为 0
    if (channelsOut) {
        *channelsOut = 0;
    }
    # 如果 sampleRateOut 不为空，则将其值设为 0
    if (sampleRateOut) {
        *sampleRateOut = 0;
    }
    # 如果 totalFrameCountOut 不为空，则将其值设为 0
    if (totalFrameCountOut) {
        *totalFrameCountOut = 0;
    }

    # 初始化 drwav 结构体
    if (!drwav_init(&wav, onRead, onSeek, pUserData, pAllocationCallbacks)) {
        return NULL;
    }
    # 调用函数 drwav__read_pcm_frames_and_close_s32() 读取 PCM 帧并关闭 WAV 文件
    return drwav__read_pcm_frames_and_close_s32(&wav, channelsOut, sampleRateOut, totalFrameCountOut);
#ifndef DR_WAV_NO_STDIO
// 以16位有符号整数格式打开文件并读取PCM帧
DRWAV_API drwav_int16* drwav_open_file_and_read_pcm_frames_s16(const char* filename, unsigned int* channelsOut, unsigned int* sampleRateOut, drwav_uint64* totalFrameCountOut, const drwav_allocation_callbacks* pAllocationCallbacks)
{
    drwav wav;

    // 如果channelsOut不为空，则将其值设为0
    if (channelsOut) {
        *channelsOut = 0;
    }
    // 如果sampleRateOut不为空，则将其值设为0
    if (sampleRateOut) {
        *sampleRateOut = 0;
    }
    // 如果totalFrameCountOut不为空，则将其值设为0
    if (totalFrameCountOut) {
        *totalFrameCountOut = 0;
    }

    // 如果无法初始化文件，则返回空指针
    if (!drwav_init_file(&wav, filename, pAllocationCallbacks)) {
        return NULL;
    }

    // 调用内部函数读取PCM帧并关闭文件
    return drwav__read_pcm_frames_and_close_s16(&wav, channelsOut, sampleRateOut, totalFrameCountOut);
}

// 以32位浮点数格式打开文件并读取PCM帧
DRWAV_API float* drwav_open_file_and_read_pcm_frames_f32(const char* filename, unsigned int* channelsOut, unsigned int* sampleRateOut, drwav_uint64* totalFrameCountOut, const drwav_allocation_callbacks* pAllocationCallbacks)
{
    drwav wav;

    // 如果channelsOut不为空，则将其值设为0
    if (channelsOut) {
        *channelsOut = 0;
    }
    // 如果sampleRateOut不为空，则将其值设为0
    if (sampleRateOut) {
        *sampleRateOut = 0;
    }
    // 如果totalFrameCountOut不为空，则将其值设为0
    if (totalFrameCountOut) {
        *totalFrameCountOut = 0;
    }

    // 如果无法初始化文件，则返回空指针
    if (!drwav_init_file(&wav, filename, pAllocationCallbacks)) {
        return NULL;
    }

    // 调用内部函数读取PCM帧并关闭文件
    return drwav__read_pcm_frames_and_close_f32(&wav, channelsOut, sampleRateOut, totalFrameCountOut);
}

// 以32位有符号整数格式打开文件并读取PCM帧
DRWAV_API drwav_int32* drwav_open_file_and_read_pcm_frames_s32(const char* filename, unsigned int* channelsOut, unsigned int* sampleRateOut, drwav_uint64* totalFrameCountOut, const drwav_allocation_callbacks* pAllocationCallbacks)
{
    drwav wav;

    // 如果channelsOut不为空，则将其值设为0
    if (channelsOut) {
        *channelsOut = 0;
    }
    // 如果sampleRateOut不为空，则将其值设为0
    if (sampleRateOut) {
        *sampleRateOut = 0;
    }
    // 如果totalFrameCountOut不为空，则将其值设为0
    if (totalFrameCountOut) {
        *totalFrameCountOut = 0;
    }

    // 如果无法初始化文件，则返回空指针
    if (!drwav_init_file(&wav, filename, pAllocationCallbacks)) {
        return NULL;
    }

    // 调用内部函数读取PCM帧并关闭文件
    return drwav__read_pcm_frames_and_close_s32(&wav, channelsOut, sampleRateOut, totalFrameCountOut);
}
#endif
# 从文件名打开并读取带有16位有符号整数的 PCM 帧
def drwav_open_file_and_read_pcm_frames_s16_w(const wchar_t* filename, unsigned int* channelsOut, unsigned int* sampleRateOut, drwav_uint64* totalFrameCountOut, const drwav_allocation_callbacks* pAllocationCallbacks)
{
    # 创建一个 drwav 结构体
    drwav wav;

    # 如果 sampleRateOut 不为空，则将其值设为0
    if (sampleRateOut) {
        *sampleRateOut = 0;
    }
    # 如果 channelsOut 不为空，则将其值设为0
    if (channelsOut) {
        *channelsOut = 0;
    }
    # 如果 totalFrameCountOut 不为空，则将其值设为0
    if (totalFrameCountOut) {
        *totalFrameCountOut = 0;
    }

    # 如果无法初始化文件，则返回空
    if (!drwav_init_file_w(&wav, filename, pAllocationCallbacks)) {
        return NULL;
    }

    # 读取 PCM 帧并关闭文件
    return drwav__read_pcm_frames_and_close_s16(&wav, channelsOut, sampleRateOut, totalFrameCountOut);
}

# 从文件名打开并读取带有32位有符号整数的 PCM 帧
def drwav_open_file_and_read_pcm_frames_s32_w(const wchar_t* filename, unsigned int* channelsOut, unsigned int* sampleRateOut, drwav_uint64* totalFrameCountOut, const drwav_allocation_callbacks* pAllocationCallbacks)
{
    # 创建一个 drwav 结构体
    drwav wav;

    # 如果 sampleRateOut 不为空，则将其值设为0
    if (sampleRateOut) {
        *sampleRateOut = 0;
    }
    # 如果 channelsOut 不为空，则将其值设为0
    if (channelsOut) {
        *channelsOut = 0;
    }
    # 如果 totalFrameCountOut 不为空，则将其值设为0
    if (totalFrameCountOut) {
        *totalFrameCountOut = 0;
    }

    # 如果无法初始化文件，则返回空
    if (!drwav_init_file_w(&wav, filename, pAllocationCallbacks)) {
        return NULL;
    }

    # 读取 PCM 帧并关闭文件
    return drwav__read_pcm_frames_and_close_s32(&wav, channelsOut, sampleRateOut, totalFrameCountOut);
}

# 从文件名打开并读取带有32位浮点数的 PCM 帧
def drwav_open_file_and_read_pcm_frames_f32_w(const wchar_t* filename, unsigned int* channelsOut, unsigned int* sampleRateOut, drwav_uint64* totalFrameCountOut, const drwav_allocation_callbacks* pAllocationCallbacks)
{
    # 创建一个 drwav 结构体
    drwav wav;

    # 如果 sampleRateOut 不为空，则将其值设为0
    if (sampleRateOut) {
        *sampleRateOut = 0;
    }
    # 如果 channelsOut 不为空，则将其值设为0
    if (channelsOut) {
        *channelsOut = 0;
    }
    # 如果 totalFrameCountOut 不为空，则将其值设为0
    if (totalFrameCountOut) {
        *totalFrameCountOut = 0;
    }

    # 如果无法初始化文件，则返回空
    if (!drwav_init_file_w(&wav, filename, pAllocationCallbacks)) {
        return NULL;
    }

    # 读取 PCM 帧并关闭文件
    return drwav__read_pcm_frames_and_close_f32(&wav, channelsOut, sampleRateOut, totalFrameCountOut);
}
# 从内存中打开并读取 16 位有符号整数格式的 PCM 数据帧
def drwav_open_memory_and_read_pcm_frames_s16(const void* data, size_t dataSize, unsigned int* channelsOut, unsigned int* sampleRateOut, drwav_uint64* totalFrameCountOut, const drwav_allocation_callbacks* pAllocationCallbacks)
{
    # 创建一个 drwav 结构体
    drwav wav;

    # 如果 channelsOut 不为空，则将其值设为 0
    if (channelsOut) {
        *channelsOut = 0;
    }
    # 如果 sampleRateOut 不为空，则将其值设为 0
    if (sampleRateOut) {
        *sampleRateOut = 0;
    }
    # 如果 totalFrameCountOut 不为空，则将其值设为 0
    if (totalFrameCountOut) {
        *totalFrameCountOut = 0;
    }

    # 如果初始化内存中的 drwav 对象失败，则返回空指针
    if (!drwav_init_memory(&wav, data, dataSize, pAllocationCallbacks)) {
        return NULL;
    }

    # 调用内部函数读取 PCM 数据帧并关闭 drwav 对象
    return drwav__read_pcm_frames_and_close_s16(&wav, channelsOut, sampleRateOut, totalFrameCountOut);
}

# 从内存中打开并读取 32 位有符号整数格式的 PCM 数据帧
def drwav_open_memory_and_read_pcm_frames_s32(const void* data, size_t dataSize, unsigned int* channelsOut, unsigned int* sampleRateOut, drwav_uint64* totalFrameCountOut, const drwav_allocation_callbacks* pAllocationCallbacks)
{
    # 创建一个 drwav 结构体
    drwav wav;

    # 如果 channelsOut 不为空，则将其值设为 0
    if (channelsOut) {
        *channelsOut = 0;
    }
    # 如果 sampleRateOut 不为空，则将其值设为 0
    if (sampleRateOut) {
        *sampleRateOut = 0;
    }
    # 如果 totalFrameCountOut 不为空，则将其值设为 0
    if (totalFrameCountOut) {
        *totalFrameCountOut = 0;
    }

    # 如果初始化内存中的 drwav 对象失败，则返回空指针
    if (!drwav_init_memory(&wav, data, dataSize, pAllocationCallbacks)) {
        return NULL;
    }

    # 调用内部函数读取 PCM 数据帧并关闭 drwav 对象
    return drwav__read_pcm_frames_and_close_s32(&wav, channelsOut, sampleRateOut, totalFrameCountOut);
}

# 从内存中打开并读取 32 位浮点数格式的 PCM 数据帧
def drwav_open_memory_and_read_pcm_frames_f32(const void* data, size_t dataSize, unsigned int* channelsOut, unsigned int* sampleRateOut, drwav_uint64* totalFrameCountOut, const drwav_allocation_callbacks* pAllocationCallbacks)
{
    # 创建一个 drwav 结构体
    drwav wav;

    # 如果 channelsOut 不为空，则将其值设为 0
    if (channelsOut) {
        *channelsOut = 0;
    }
    # 如果 sampleRateOut 不为空，则将其值设为 0
    if (sampleRateOut) {
        *sampleRateOut = 0;
    }
    # 如果 totalFrameCountOut 不为空，则将其值设为 0
    if (totalFrameCountOut) {
        *totalFrameCountOut = 0;
    }

    # 如果初始化内存中的 drwav 对象失败，则返回空指针
    if (!drwav_init_memory(&wav, data, dataSize, pAllocationCallbacks)) {
        return NULL;
    }

    # 调用内部函数读取 PCM 数据帧并关闭 drwav 对象
    return drwav__read_pcm_frames_and_close_f32(&wav, channelsOut, sampleRateOut, totalFrameCountOut);
}
#ifndef DR_WAV_NO_CONVERSION_API */
// 如果未定义 DR_WAV_NO_CONVERSION_API，则执行以下代码

DRWAV_API void drwav_free(void* p, const drwav_allocation_callbacks* pAllocationCallbacks)
{
    // 如果传入了自定义内存分配回调函数，则使用该回调函数释放内存
    if (pAllocationCallbacks != NULL) {
        drwav__free_from_callbacks(p, pAllocationCallbacks);
    } else {
        // 否则使用默认的内存释放函数
        drwav__free_default(p, NULL);
    }
}

DRWAV_API drwav_uint16 drwav_bytes_to_u16(const drwav_uint8* data)
{
    // 将字节数组转换为 16 位无符号整数
    return drwav__bytes_to_u16(data);
}

DRWAV_API drwav_int16 drwav_bytes_to_s16(const drwav_uint8* data)
{
    // 将字节数组转换为 16 位有符号整数
    return drwav__bytes_to_s16(data);
}

DRWAV_API drwav_uint32 drwav_bytes_to_u32(const drwav_uint8* data)
{
    // 将字节数组转换为 32 位无符号整数
    return drwav__bytes_to_u32(data);
}

DRWAV_API drwav_int32 drwav_bytes_to_s32(const drwav_uint8* data)
{
    // 将字节数组转换为 32 位有符号整数
    return drwav__bytes_to_s32(data);
}

DRWAV_API drwav_uint64 drwav_bytes_to_u64(const drwav_uint8* data)
{
    // 将字节数组转换为 64 位无符号整数
    return drwav__bytes_to_u64(data);
}

DRWAV_API drwav_int64 drwav_bytes_to_s64(const drwav_uint8* data)
{
    // 将字节数组转换为 64 位有符号整数
    return drwav__bytes_to_s64(data);
}


DRWAV_API drwav_bool32 drwav_guid_equal(const drwav_uint8 a[16], const drwav_uint8 b[16])
{
    // 判断两个 GUID 是否相等
    return drwav__guid_equal(a, b);
}

DRWAV_API drwav_bool32 drwav_fourcc_equal(const drwav_uint8* a, const char* b)
{
    // 判断两个 FourCC 是否相等
    return drwav__fourcc_equal(a, b);
}

#endif  /* dr_wav_c */
#endif  /* DR_WAV_IMPLEMENTATION */

/*
RELEASE NOTES - v0.11.0
=======================
Version 0.11.0 has breaking API changes.

Improved Client-Defined Memory Allocation
-----------------------------------------
The main change with this release is the addition of a more flexible way of implementing custom memory allocation routines. The
existing system of DRWAV_MALLOC, DRWAV_REALLOC and DRWAV_FREE are still in place and will be used by default when no custom
allocation callbacks are specified.

To use the new system, you pass in a pointer to a drwav_allocation_callbacks object to drwav_init() and family, like this:

    void* my_malloc(size_t sz, void* pUserData)
    {
        return malloc(sz);
    }
    // 重新分配内存的自定义函数，用于drwav库的内存分配
    void* my_realloc(void* p, size_t sz, void* pUserData)
    {
        return realloc(p, sz);
    }
    // 释放内存的自定义函数，用于drwav库的内存释放
    void my_free(void* p, void* pUserData)
    {
        free(p);
    }

    // 创建一个包含自定义内存分配和释放函数的结构体
    drwav_allocation_callbacks allocationCallbacks;
    // 将自定义数据指针赋值给结构体中的pUserData成员
    allocationCallbacks.pUserData = &myData;
    // 将自定义的内存分配函数赋值给结构体中的onMalloc成员
    allocationCallbacks.onMalloc  = my_malloc;
    // 将自定义的内存重新分配函数赋值给结构体中的onRealloc成员
    allocationCallbacks.onRealloc = my_realloc;
    // 将自定义的内存释放函数赋值给结构体中的onFree成员
    allocationCallbacks.onFree    = my_free;
    // 使用自定义的内存分配和释放函数初始化drwav库的文件读取
    drwav_init_file(&wav, "my_file.wav", &allocationCallbacks);
# 这个新系统的优势在于它允许您指定用户数据，这些数据将传递给分配例程。

# 传递空值给分配回调对象将导致 dr_wav 使用默认值，这与 DRWAV_MALLOC、DRWAV_REALLOC 和 DRWAV_FREE 相同，相当于在以前的版本中的工作方式。

# 现在每个打开 drwav 对象的 API 都需要这个额外的参数。这些包括以下内容：

# 以下 API 现在返回本机字节顺序的数据，这提高了在大端架构上的兼容性。

# 以前，以下 API 返回小端音频数据。现在它们返回本机字节顺序的数据。这提高了在大端架构上的兼容性。
    # 使用 drwav 库中的函数打开并读取 PCM 帧数据，返回带符号 16 位整数
    drwav_open_and_read_pcm_frames_s16()
    
    # 使用 drwav 库中的函数打开并读取 PCM 帧数据，返回带符号 32 位整数
    drwav_open_and_read_pcm_frames_s32()
    
    # 使用 drwav 库中的函数打开并读取 PCM 帧数据，返回 32 位浮点数
    drwav_open_and_read_pcm_frames_f32()
    
    # 使用 drwav 库中的函数打开文件并读取 PCM 帧数据，返回带符号 16 位整数
    drwav_open_file_and_read_pcm_frames_s16()
    
    # 使用 drwav 库中的函数打开文件并读取 PCM 帧数据，返回带符号 32 位整数
    drwav_open_file_and_read_pcm_frames_s32()
    
    # 使用 drwav 库中的函数打开文件并读取 PCM 帧数据，返回 32 位浮点数
    drwav_open_file_and_read_pcm_frames_f32()
    
    # 使用 drwav 库中的函数打开文件并读取 PCM 帧数据，返回带符号 16 位整数（支持写入）
    drwav_open_file_and_read_pcm_frames_s16_w()
    
    # 使用 drwav 库中的函数打开文件并读取 PCM 帧数据，返回带符号 32 位整数（支持写入）
    drwav_open_file_and_read_pcm_frames_s32_w()
    
    # 使用 drwav 库中的函数打开文件并读取 PCM 帧数据，返回 32 位浮点数（支持写入）
    drwav_open_file_and_read_pcm_frames_f32_w()
    
    # 使用 drwav 库中的函数打开内存并读取 PCM 帧数据，返回带符号 16 位整数
    drwav_open_memory_and_read_pcm_frames_s16()
    
    # 使用 drwav 库中的函数打开内存并读取 PCM 帧数据，返回带符号 32 位整数
    drwav_open_memory_and_read_pcm_frames_s32()
    
    # 使用 drwav 库中的函数打开内存并读取 PCM 帧数据，返回 32 位浮点数
    drwav_open_memory_and_read_pcm_frames_f32()
# 添加了一些 API 用于显式控制音频数据的读取或写入的大端或小端字节顺序
drwav_read_pcm_frames_le()
drwav_read_pcm_frames_be()
drwav_read_pcm_frames_s16le()
drwav_read_pcm_frames_s16be()
drwav_read_pcm_frames_f32le()
drwav_read_pcm_frames_f32be()
drwav_read_pcm_frames_s32le()
drwav_read_pcm_frames_s32be()
drwav_write_pcm_frames_le()
drwav_write_pcm_frames_be()

# 已移除的 API
# 在版本 0.10.0 中已经移除了以下 API，这些 API 在版本 0.10.0 中已经被弃用
drwav_open()
drwav_open_ex()
drwav_open_write()
drwav_open_write_sequential()
drwav_open_file()
drwav_open_file_ex()
drwav_open_file_write()
drwav_open_file_write_sequential()
drwav_open_memory()
drwav_open_memory_ex()
drwav_open_memory_write()
drwav_open_memory_write_sequential()
drwav_close()

# 发布说明 - v0.10.0
# 版本 0.10.0 有破坏性的 API 更改。在此版本中没有重大的 bug 修复，所以如果你受到影响，你不需要升级。

# 已移除的 API
# 在版本 0.9.0 中已经弃用的以下 API 在版本 0.10.0 中已经完全移除
drwav_read()
drwav_read_s16()
drwav_read_f32()
drwav_read_s32()
drwav_seek_to_sample()
drwav_write()
drwav_open_and_read_s16()
drwav_open_and_read_f32()
drwav_open_and_read_s32()
drwav_open_file_and_read_s16()
drwav_open_file_and_read_f32()
drwav_open_file_and_read_s32()
drwav_open_memory_and_read_s16()
drwav_open_memory_and_read_f32()
drwav_open_memory_and_read_s32()
drwav::totalSampleCount

# 请查看此文件底部的版本 0.9.0 的发布说明以获取替代 API。

# 已弃用的 API
# 以下 API 已被弃用。drwav_init*() 和 drwav_init*() 之间存在混乱和完全任意的差异
/*
drwav_open*(), where drwav_init*() initializes a pre-allocated drwav object, whereas drwav_open*() will first allocated a
drwav object on the heap and then initialize it. drwav_open*() has been deprecated which means you must now use a pre-
allocated drwav object with drwav_init*(). If you need the previous functionality, you can just do a malloc() followed by
a called to one of the drwav_init*() APIs.
*/
/*
REVISION HISTORY
================
v0.12.16 - 2020-12-02
  - Fix a bug when trying to read more bytes than can fit in a size_t.

v0.12.15 - 2020-11-21
  - Fix compilation with OpenWatcom.

v0.12.14 - 2020-11-13
  - Minor code clean up.

v0.12.13 - 2020-11-01
  - Improve compiler support for older versions of GCC.

v0.12.12 - 2020-09-28
  - Add support for RF64.
  - Fix a bug in writing mode where the size of the RIFF chunk incorrectly includes the header section.

v0.12.11 - 2020-09-08
  - Fix a compilation error on older compilers.

v0.12.10 - 2020-08-24
  - Fix a bug when seeking with ADPCM formats.

v0.12.9 - 2020-08-02
  - Simplify sized types.

v0.12.8 - 2020-07-25
  - Fix a compilation warning.

v0.12.7 - 2020-07-15
  - Fix some bugs on big-endian architectures.
  - Fix an error in s24 to f32 conversion.
*/
# 版本 v0.12.6 - 2020-06-23
  - 修改 drwav_read_*()，允许将 NULL 作为输出缓冲区传入，相当于向前查找。
  - 修复尝试解码无效的 IMA-ADPCM 文件时的缓冲区溢出问题。
  - 为实现部分添加包含保护。

# 版本 v0.12.5 - 2020-05-27
  - 微小的文档修复。

# 版本 v0.12.4 - 2020-05-16
  - 用 DRWAV_ASSERT() 替换 assert()。
  - 添加编译时和运行时版本查询。
    - DRWAV_VERSION_MINOR
    - DRWAV_VERSION_MAJOR
    - DRWAV_VERSION_REVISION
    - DRWAV_VERSION_STRING
    - drwav_version()
    - drwav_version_string()

# 版本 v0.12.3 - 2020-04-30
  - 修复 VC6 的编译错误。

# 版本 v0.12.2 - 2020-04-21
  - 修复一个 bug，即 drwav_init_file() 在尝试加载错误文件后不关闭文件句柄。

# 版本 v0.12.1 - 2020-04-13
  - 修复一些啰嗦的警告。

# 版本 v0.12.0 - 2020-04-04
  - API 更改：向 chunk 回调添加容器和格式参数。
  - 微小的文档更新。

# 版本 v0.11.5 - 2020-03-07
  - 修复在 Visual Studio .NET 2003 中的编译错误。

# 版本 v0.11.4 - 2020-01-29
  - 修复一些静态分析警告。
  - 修复从 A-law 编码流中读取 f32 样本时的 bug。

# 版本 v0.11.3 - 2020-01-12
  - 对一些 f32 格式转换例程进行微小更改。
  - 当到达文件结尾时，对 ADPCM 转换进行微小的 bug 修复。

# 版本 v0.11.2 - 2019-12-02
  - 修复当使用自定义内存分配器但没有自定义 realloc() 实现时可能导致的崩溃问题。
  - 修复整数溢出 bug。
  - 修复空指针解引用 bug。
  - 添加对采样率、通道和每样本位数的限制，以加强一些验证。

# 版本 v0.11.1 - 2019-10-07
  - 内部代码清理。

# 版本 v0.11.0 - 2019-10-06
  - API 更改：添加对用户定义内存分配例程的支持。该系统允许程序指定其自己的内存分配例程，并带有一个用于客户特定上下文数据的用户数据指针。这为以下 API 添加了额外的参数：
    - drwav_init()
    # 初始化一个 WAV 文件读取器，使用默认参数
    drwav_init_ex()
    # 从文件初始化一个 WAV 文件读取器
    drwav_init_file()
    # 从文件初始化一个 WAV 文件读取器，使用额外参数
    drwav_init_file_ex()
    # 从文件初始化一个 WAV 文件读取器，使用宽字符文件名
    drwav_init_file_w()
    # 从文件初始化一个 WAV 文件读取器，使用额外参数和宽字符文件名
    drwav_init_file_w_ex()
    # 从内存初始化一个 WAV 文件读取器
    drwav_init_memory()
    # 从内存初始化一个 WAV 文件读取器，使用额外参数
    drwav_init_memory_ex()
    # 初始化一个 WAV 文件写入器
    drwav_init_write()
    # 初始化一个顺序写入的 WAV 文件写入器
    drwav_init_write_sequential()
    # 初始化一个顺序写入 PCM 帧的 WAV 文件写入器
    drwav_init_write_sequential_pcm_frames()
    # 从文件初始化一个 WAV 文件写入器
    drwav_init_file_write()
    # 从文件初始化一个顺序写入的 WAV 文件写入器
    drwav_init_file_write_sequential()
    # 从文件初始化一个顺序写入 PCM 帧的 WAV 文件写入器
    drwav_init_file_write_sequential_pcm_frames()
    # 从文件初始化一个 WAV 文件写入器，使用宽字符文件名
    drwav_init_file_write_w()
    # 从文件初始化一个顺序写入的 WAV 文件写入器，使用宽字符文件名
    drwav_init_file_write_sequential_w()
    # 从文件初始化一个顺序写入 PCM 帧的 WAV 文件写入器，使用宽字符文件名
    drwav_init_file_write_sequential_pcm_frames_w()
    # 从内存初始化一个 WAV 文件写入器
    drwav_init_memory_write()
    # 从内存初始化一个顺序写入的 WAV 文件写入器
    drwav_init_memory_write_sequential()
    # 从内存初始化一个顺序写入 PCM 帧的 WAV 文件写入器
    drwav_init_memory_write_sequential_pcm_frames()
    # 打开并读取一个 WAV 文件的 PCM 帧，返回有符号 16 位整数
    drwav_open_and_read_pcm_frames_s16()
    # 打开并读取一个 WAV 文件的 PCM 帧，返回 32 位浮点数
    drwav_open_and_read_pcm_frames_f32()
    # 打开并读取一个 WAV 文件的 PCM 帧，返回有符号 32 位整数
    drwav_open_and_read_pcm_frames_s32()
    # 打开文件并读取一个 WAV 文件的 PCM 帧，返回有符号 16 位整数
    drwav_open_file_and_read_pcm_frames_s16()
    # 打开文件并读取一个 WAV 文件的 PCM 帧，返回 32 位浮点数
    drwav_open_file_and_read_pcm_frames_f32()
    # 打开文件并读取一个 WAV 文件的 PCM 帧，返回有符号 32 位整数
    drwav_open_file_and_read_pcm_frames_s32()
    # 打开文件并读取一个 WAV 文件的 PCM 帧，返回有符号 16 位整数，使用宽字符文件名
    drwav_open_file_and_read_pcm_frames_s16_w()
    # 打开文件并读取一个 WAV 文件的 PCM 帧，返回 32 位浮点数，使用宽字符文件名
    drwav_open_file_and_read_pcm_frames_f32_w()
    # 打开文件并读取一个 WAV 文件的 PCM 帧，返回有符号 32 位整数，使用宽字符文件名
    drwav_open_file_and_read_pcm_frames_s32_w()
    # 从内存打开并读取一个 WAV 文件的 PCM 帧，返回有符号 16 位整数
    drwav_open_memory_and_read_pcm_frames_s16()
    # 从内存打开并读取一个 WAV 文件的 PCM 帧，返回 32 位浮点数
    drwav_open_memory_and_read_pcm_frames_f32()
    # 从内存打开并读取一个 WAV 文件的 PCM 帧，返回有符号 32 位整数
    drwav_open_memory_and_read_pcm_frames_s32()
    # 设置额外参数为 NULL，使用默认参数，即使用 DRWAV_MALLOC、DRWAV_REALLOC 和 DRWAV_FREE
    Set this extra parameter to NULL to use defaults which is the same as the previous behaviour. Setting this NULL will use
    DRWAV_MALLOC, DRWAV_REALLOC and DRWAV_FREE.
    # 添加对显式字节序的 PCM 帧读取和写入的支持
    Add support for reading and writing PCM frames in an explicit endianness. New APIs:
    # 从 WAV 文件读取 PCM 帧，返回小端字节序
    drwav_read_pcm_frames_le()
    # 从 WAV 文件读取 PCM 帧，返回大端字节序
    drwav_read_pcm_frames_be()
    # 从 WAV 文件读取 PCM 帧，返回小端有符号 16 位整数
    drwav_read_pcm_frames_s16le()
    # 从 WAV 文件读取 PCM 帧，返回大端有符号 16 位整数
    drwav_read_pcm_frames_s16be()
    # 从 WAV 文件读取 PCM 帧，返回小端 32 位浮点数
    drwav_read_pcm_frames_f32le()
    # 从 WAV 文件读取 PCM 帧，返回大端 32 位浮点数
    drwav_read_pcm_frames_f32be()
    # 从 WAV 文件读取 PCM 帧，返回小端有符号 32 位整数
    drwav_read_pcm_frames_s32le()
    # 从 WAV 文件读取 PCM 帧，返回大端有符号 32 位整数
    drwav_read_pcm_frames_s32be()
    # 向 WAV 文件写入 PCM 帧，使用小端字节序
    drwav_write_pcm_frames_le()
    # 向 WAV 文件写入 PCM 帧，使用大端字节序
    drwav_write_pcm_frames_be()
    # 移除已废弃的 API
    Remove deprecated APIs.
    # API 变更：以下 API 现在返回本机字节序的数据。之前返回小端数据。
    API CHANGE: The following APIs now return native-endian data. Previously they returned little-endian data.
    # 读取 WAV 文件中的 PCM 帧
    drwav_read_pcm_frames()
    
    # 读取 WAV 文件中的 16 位有符号整数格式的 PCM 帧
    drwav_read_pcm_frames_s16()
    
    # 读取 WAV 文件中的 32 位有符号整数格式的 PCM 帧
    drwav_read_pcm_frames_s32()
    
    # 读取 WAV 文件中的 32 位浮点数格式的 PCM 帧
    drwav_read_pcm_frames_f32()
    
    # 打开 WAV 文件并读取其中的 16 位有符号整数格式的 PCM 帧
    drwav_open_and_read_pcm_frames_s16()
    
    # 打开 WAV 文件并读取其中的 32 位有符号整数格式的 PCM 帧
    drwav_open_and_read_pcm_frames_s32()
    
    # 打开 WAV 文件并读取其中的 32 位浮点数格式的 PCM 帧
    drwav_open_and_read_pcm_frames_f32()
    
    # 打开 WAV 文件并读取其中的 16 位有符号整数格式的 PCM 帧
    drwav_open_file_and_read_pcm_frames_s16()
    
    # 打开 WAV 文件并读取其中的 32 位有符号整数格式的 PCM 帧
    drwav_open_file_and_read_pcm_frames_s32()
    
    # 打开 WAV 文件并读取其中的 32 位浮点数格式的 PCM 帧
    drwav_open_file_and_read_pcm_frames_f32()
    
    # 打开 WAV 文件并读取其中的 16 位有符号整数格式的 PCM 帧（宽字符版本）
    drwav_open_file_and_read_pcm_frames_s16_w()
    
    # 打开 WAV 文件并读取其中的 32 位有符号整数格式的 PCM 帧（宽字符版本）
    drwav_open_file_and_read_pcm_frames_s32_w()
    
    # 打开 WAV 文件并读取其中的 32 位浮点数格式的 PCM 帧（宽字符版本）
    drwav_open_file_and_read_pcm_frames_f32_w()
    
    # 打开内存中的 WAV 数据并读取其中的 16 位有符号整数格式的 PCM 帧
    drwav_open_memory_and_read_pcm_frames_s16()
    
    # 打开内存中的 WAV 数据并读取其中的 32 位有符号整数格式的 PCM 帧
    drwav_open_memory_and_read_pcm_frames_s32()
    
    # 打开内存中的 WAV 数据并读取其中的 32 位浮点数格式的 PCM 帧
    drwav_open_memory_and_read_pcm_frames_f32()
# 版本 v0.10.1 - 2019-08-31
  - 正确处理部分尾随的 ADPCM 块。

# 版本 v0.10.0 - 2019-08-04
  - 移除了已弃用的 API。
  - 为文件加载 API 添加了 wchar_t 变体：
      drwav_init_file_w()
      drwav_init_file_ex_w()
      drwav_init_file_write_w()
      drwav_init_file_write_sequential_w()
  - 添加了 drwav_target_write_size_bytes()，用于根据格式和样本数计算 WAV 文件的总字节数。
  - 添加了用于在顺序写模式下指定 PCM 帧数而不是样本数的 API：
      drwav_init_write_sequential_pcm_frames()
      drwav_init_file_write_sequential_pcm_frames()
      drwav_init_file_write_sequential_pcm_frames_w()
      drwav_init_memory_write_sequential_pcm_frames()
  - 弃用了 drwav_open*() 和 drwav_close()：
      drwav_open()
      drwav_open_ex()
      drwav_open_write()
      drwav_open_write_sequential()
      drwav_open_file()
      drwav_open_file_ex()
      drwav_open_file_write()
      drwav_open_file_write_sequential()
      drwav_open_memory()
      drwav_open_memory_ex()
      drwav_open_memory_write()
      drwav_open_memory_write_sequential()
      drwav_close()
  - 小的文档更新。

# 版本 v0.9.2 - 2019-05-21
  - 修复警告。

# 版本 v0.9.1 - 2019-05-05
  - 添加对 C89 的支持。
  - 将许可证更改为公共领域或 MIT-0 的选择。

# 版本 v0.9.0 - 2018-12-16
  - API 更改：添加了新的读取 API，用于按 PCM 帧而不是样本进行读取。旧的 API 已被弃用。
    # 在 v0.10.0 版本中将被移除的内容。已弃用的 API 及其替代项：
    # 将 drwav_read() 替换为 drwav_read_pcm_frames()
    # 将 drwav_read_s16() 替换为 drwav_read_pcm_frames_s16()
    # 将 drwav_read_f32() 替换为 drwav_read_pcm_frames_f32()
    # 将 drwav_read_s32() 替换为 drwav_read_pcm_frames_s32()
    # 将 drwav_seek_to_sample() 替换为 drwav_seek_to_pcm_frame()
    # 将 drwav_write() 替换为 drwav_write_pcm_frames()
    # 将 drwav_open_and_read_s16() 替换为 drwav_open_and_read_pcm_frames_s16()
    # 将 drwav_open_and_read_f32() 替换为 drwav_open_and_read_pcm_frames_f32()
    # 将 drwav_open_and_read_s32() 替换为 drwav_open_and_read_pcm_frames_s32()
    # 将 drwav_open_file_and_read_s16() 替换为 drwav_open_file_and_read_pcm_frames_s16()
    # 将 drwav_open_file_and_read_f32() 替换为 drwav_open_file_and_read_pcm_frames_f32()
    # 将 drwav_open_file_and_read_s32() 替换为 drwav_open_file_and_read_pcm_frames_s32()
    # 将 drwav_open_memory_and_read_s16() 替换为 drwav_open_memory_and_read_pcm_frames_s16()
    # 将 drwav_open_memory_and_read_f32() 替换为 drwav_open_memory_and_read_pcm_frames_f32()
    # 将 drwav_open_memory_and_read_s32() 替换为 drwav_open_memory_and_read_pcm_frames_s32()
    # 将 drwav::totalSampleCount 替换为 drwav::totalPCMFrameCount
    # API 变更：将 drwav_open_and_read_file_*() 重命名为 drwav_open_file_and_read_*()
    # API 变更：将 drwav_open_and_read_memory_*() 重命名为 drwav_open_memory_and_read_*()
    # 添加对 smpl 块的内置支持
    # 添加在初始化时为文件中的每个块触发回调的支持
    # 通过 drwav_init_ex() 等一系列 API 来启用此功能
    # 更加健壮地处理无效的 FMT 块
# 版本 v0.8.5 - 2018-09-11
  - 修复潜在的堆栈溢出问题。

# 版本 v0.8.4 - 2018-08-07
  - 改进了 64 位检测。

# 版本 v0.8.3 - 2018-08-05
  - 修复了在较旧版本的 GCC 上构建 C++ 的问题。

# 版本 v0.8.2 - 2018-08-02
  - 修复了一些大端模式的 bug。

# 版本 v0.8.1 - 2018-06-29
  - 添加了对顺序写入 API 的支持。
  - 在写入模式下禁用寻址。
  - 修复了 Wave64 的 bug。
  - 修正了拼写错误。

# 版本 v0.8 - 2018-04-27
  - 修复 bug。
  - 开始使用主次修订版本号。

# 版本 v0.7f - 2018-02-05
  - 将 ADPCM 格式限制为最多 2 个通道。

# 版本 v0.7e - 2018-02-02
  - 修复了一个崩溃问题。

# 版本 v0.7d - 2018-02-01
  - 为所有压缩格式设置 drwav.bytesPerSample 为 0。
  - 修复了读取 16 位浮点 WAV 文件时的崩溃问题。在这种情况下，dr_wav 将对所有格式转换读取 API (*_s16, *_s32, *_f32 APIs) 输出静音。
  - 修复了一些除零错误。

# 版本 v0.7b - 2018-01-22
  - 修复了压缩格式的寻址错误。
  - 修复了 DR_WAV_NO_CONVERSION_API 时的编译错误。

# 版本 v0.7a - 2017-11-17
  - 修复了一些 GCC 警告。

# 版本 v0.7 - 2017-11-04
  - 添加了写入 API。

# 版本 v0.6 - 2017-08-16
  - API 更改：将 dr_* 类型重命名为 drwav_*。
  - 添加了对 malloc()、realloc() 等自定义实现的支持。
  - 添加了对 Microsoft ADPCM 的支持。
  - 添加了对 IMA ADPCM (DVI, 格式代码 0x11) 的支持。
  - 优化了 drwav_read_s16()。
  - 修复了 bug。

# 版本 v0.5g - 2017-07-16
  - 将布尔值的底层类型更改为无符号数。

# 版本 v0.5f - 2017-04-04
  - 修复了 drwav_open_and_read_s16() 和相关函数的一个小 bug。

# 版本 v0.5e - 2016-12-29
  - 添加了作为有符号 16 位整数读取样本的支持。使用 _s16() 系列的 API。
  - 对文档进行了一些小修正。

# 版本 v0.5d - 2016-12-28
  - 使用 drwav_int* 和 drwav_uint* 大小类型以改善编译器支持。

# 版本 v0.5c - 2016-11-11
  - 正确处理在 FMT 块之前出现的 JUNK 块。

# 版本 v0.5b - 2016-10-23
  - 对 drwav_bool8 和 drwav_bool32 类型进行了一些小改动。
/*
版本 v0.5a - 2016-10-11
  - 修复了由于参数顺序不正确而导致的 drwav_open_and_read() 和相关函数的 bug。
  - 提高了 A-law 和 mu-law 的效率。

版本 v0.5 - 2016-09-29
  - API 更改。交换了 drwav_open_and_read*() 中 "channels" 和 "sampleRate" 参数的顺序。这样做的原因是为了
    与 dr_audio 和 dr_flac 保持一致。

版本 v0.4b - 2016-09-18
  - 修正了文档中的拼写错误。

版本 v0.4a - 2016-09-18
  - 修正了一个拼写错误。
  - 将日期格式更改为 ISO 8601（YYYY-MM-DD）

版本 v0.4 - 2016-07-13
  - API 更改。使 onSeek 与 dr_flac 保持一致。
  - API 更改。将 drwav_seek() 重命名为 drwav_seek_to_sample()，以便清晰地表达并与 dr_flac 保持一致。
  - 增加了对 Sony Wave64 的支持。

版本 v0.3a - 2016-05-28
  - API 更改。在 onSeek 回调中返回 drwav_bool32 而不是 int。
  - 修复了一个内存泄漏。

版本 v0.3 - 2016-05-22
  - 为了保持一致性，进行了大量的 API 更改。

版本 v0.2a - 2016-05-16
  - 修复了 Linux/GCC 构建问题。

版本 v0.2 - 2016-05-11
  - 增加了对以有符号 32 位 PCM 读取数据的支持，以保持与 dr_flac 的一致性。

版本 v0.1a - 2016-05-07
  - 修复了在 drwav_open_file() 中的一个 bug，即如果加载器初始化失败，则文件句柄将不会被关闭。

版本 v0.1 - 2016-05-04
  - 初始版本发布。
*/

/*
此软件可选择以下许可证之一。选择您喜欢的。

===============================================================================
备选方案 1 - 公有领域（www.unlicense.org）
===============================================================================
这是释放到公有领域的自由且不受限制的软件。

任何人都可以自由地复制、修改、发布、使用、编译、出售或分发此软件，无论是以源代码形式还是编译后的二进制形式，无论是出于任何目的，商业或非商业，以及通过任何方式。

在承认版权法的司法管辖区，本软件的作者或作者将此软件的任何和所有版权利益捐赠给公众。
*/
# 这部分代码是软件的许可声明，包括 Unlicense 和 MIT No Attribution 两种许可方式
# Unlicense 许可方式声明
# 该软件被放弃版权，以利于公众，对我们的继承人和后继者不利。我们打算将此放弃声明为对此软件在版权法下所有现有和未来权利的永久放弃行为。
# 软件按"原样"提供，不提供任何形式的保证，包括但不限于对适销性、特定用途的适用性和非侵权的保证。在任何情况下，作者均不对任何索赔、损害或其他责任承担责任，无论是合同诉讼、侵权行为还是其他方式，由此软件或使用软件引起的、或与软件或使用软件有关的索赔、损害或其他责任。
# 更多信息，请参考 <http://unlicense.org/>

# MIT No Attribution 许可方式声明
# 版权声明，允许任何人免费获取此软件及相关文档文件（以下简称"软件"），在不受限制的情况下使用、复制、修改、合并、发布、分发、许可和/或出售软件的副本，并允许软件的受让人这样做。
# 软件按"原样"提供，不提供任何形式的保证，包括但不限于对适销性、特定用途的适用性和非侵权的保证。在任何情况下，作者或版权持有人均不对任何索赔、损害或其他责任承担责任，无论是合同诉讼、侵权行为还是其他方式，由此软件或使用软件引起的、或与软件或使用软件有关的索赔、损害或其他责任。
```