# `whisper.cpp\examples\dr_wav.h`

```cpp
/*
WAV音频加载器和写入器。选择公共领域或MIT-0。请查看本文件末尾的许可声明。
dr_wav - v0.12.16 - 2020-12-02

David Reid - mackron@gmail.com

GitHub: https://github.com/mackron/dr_libs
*/

/*
版本0.12的发布说明
============================
版本0.12包括对自定义块处理的重大更改。

对块回调的更改
-------------------------
dr_wav支持在遇到块时触发回调（除了WAVE和FMT块）。回调已更新，包括容器（RIFF或Wave64）和包含有关wave文件中数据格式的信息的FMT块。

以前，没有直接方法确定容器，因此无法区分块头中的不同ID（RIFF和Wave64容器以不同方式编码块ID）。`container`参数可用于知道要使用哪个ID。

有时，在触发块回调时了解数据格式可能很有用。现在，将指向`drwav_fmt`对象的指针传递到块回调中，这将为您提供有关数据格式的信息。要确定采样格式，请使用`drwav_fmt_get_format()`。这将返回`DR_WAVE_FORMAT_*`之一的标记。
*/

/*
介绍
============
这是一个单文件库。要使用它，请在一个.c文件中执行以下操作。

    ```c
    #define DR_WAV_IMPLEMENTATION
    #include "dr_wav.h"
    ```cpp

然后，您可以像使用任何其他头文件一样，在程序的其他部分中#include此文件。要读取音频数据，请执行以下操作：

    ```c
    drwav wav;
    if (!drwav_init_file(&wav, "my_song.wav", NULL)) {
        // 打开WAV文件时出错。
    }

    drwav_int32* pDecodedInterleavedPCMFrames = malloc(wav.totalPCMFrameCount * wav.channels * sizeof(drwav_int32));
*/
    // 读取 WAV 文件中的所有 PCM 样本帧，并返回实际解码的样本数
    size_t numberOfSamplesActuallyDecoded = drwav_read_pcm_frames_s32(&wav, wav.totalPCMFrameCount, pDecodedInterleavedPCMFrames);

    // 关闭 WAV 文件
    drwav_uninit(&wav);
// 打开并读取音频数据的示例代码
unsigned int channels; // 声道数
unsigned int sampleRate; // 采样率
drwav_uint64 totalPCMFrameCount; // PCM帧总数
// 打开并读取 WAV 文件的 PCM 帧数据，返回浮点数数组
float* pSampleData = drwav_open_file_and_read_pcm_frames_f32("my_song.wav", &channels, &sampleRate, &totalPCMFrameCount, NULL);
if (pSampleData == NULL) {
    // 如果打开和读取 WAV 文件出错，则输出错误信息
    // Error opening and reading WAV file.
}

...

// 释放音频数据内存
drwav_free(pSampleData);

// 读取 PCM 帧数据
size_t framesRead = drwav_read_pcm_frames(&wav, wav.totalPCMFrameCount, pDecodedInterleavedPCMFrames);

// 读取原始音频数据的字节
size_t bytesRead = drwav_read_raw(&wav, bytesToRead, pRawDataBuffer);

// 输出 WAV 文件，不支持压缩格式
drwav_data_format format;
format.container = drwav_container_riff; // WAV 文件格式
format.format = DR_WAVE_FORMAT_PCM; // PCM 格式
format.channels = 2; // 声道数
format.sampleRate = 44100; // 采样率
format.bitsPerSample = 16; // 每个样本的位数
// 初始化写入 WAV 文件
drwav_init_file_write(&wav, "data/recording.wav", &format, NULL);

...

// 写入 PCM 帧数据
drwav_uint64 framesWritten = drwav_write_pcm_frames(pWav, frameCount, pSamples);
/*
dr_wav 提供了对 Sony Wave64 格式的无缝支持。解码器将自动检测它，无需任何手动干预即可正常工作。

构建选项
=============
在包含此文件之前定义这些选项。

#define DR_WAV_NO_CONVERSION_API
  禁用转换 API，如 `drwav_read_pcm_frames_f32()` 和 `drwav_s16_to_f32()`。

#define DR_WAV_NO_STDIO
  禁用从文件初始化解码器的 API，如 `drwav_init_file()`、`drwav_init_file_write()` 等。

注意事项
=====
- 样本始终是交错的。
- 默认读取函数不执行任何数据转换。使用 `drwav_read_pcm_frames_f32()`、`drwav_read_pcm_frames_s32()` 和 `drwav_read_pcm_frames_s16()`
  来读取和转换音频数据为 32 位浮点、有符号 32 位整数和有符号 16 位整数样本。经过测试和支持的内部格式包括以下内容：
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
# 定义无符号32位整数类型
typedef unsigned int            drwav_uint32;
# 如果编译器为 MSC_VER
#if defined(_MSC_VER)
    # 定义有符号64位整数类型
    typedef   signed __int64    drwav_int64;
    # 定义无符号64位整数类型
    typedef unsigned __int64    drwav_uint64;
#else
    # 如果编译器为 clang 或者是 GCC 版本大于 4.6
    # 忽略 long long 类型的警告
    # 如果是 clang 编译器，还需要忽略 c++11 long long 类型的警告
    # 定义有符号64位整数类型
    typedef   signed long long  drwav_int64;
    # 定义无符号64位整数类型
    typedef unsigned long long  drwav_uint64;
    # 恢复之前忽略的 long long 类型的警告
#endif
# 如果是 64 位系统
#if defined(__LP64__) || defined(_WIN64) || (defined(__x86_64__) && !defined(__ILP32__)) || defined(_M_X64) || defined(__ia64) || defined (_M_IA64) || defined(__aarch64__) || defined(__powerpc64__)
    # 定义无符号指针类型为 64 位整数类型
    typedef drwav_uint64        drwav_uintptr;
# 如果是 32 位系统
#else
    # 定义无符号指针类型为 32 位整数类型
    typedef drwav_uint32        drwav_uintptr;
#endif
# 定义8位布尔类型
typedef drwav_uint8             drwav_bool8;
# 定义32位布尔类型
typedef drwav_uint32            drwav_bool32;
# 定义 TRUE 和 FALSE 常量
#define DRWAV_TRUE              1
#define DRWAV_FALSE             0

# 如果未定义 DRWAV_API
#if !defined(DRWAV_API)
    #if defined(DRWAV_DLL)
        #if defined(_WIN32)
            #define DRWAV_DLL_IMPORT  __declspec(dllimport)
            #define DRWAV_DLL_EXPORT  __declspec(dllexport)
            #define DRWAV_DLL_PRIVATE static
        #else
            #if defined(__GNUC__) && __GNUC__ >= 4
                #define DRWAV_DLL_IMPORT  __attribute__((visibility("default")))
                #define DRWAV_DLL_EXPORT  __attribute__((visibility("default")))
                #define DRWAV_DLL_PRIVATE __attribute__((visibility("hidden")))
            #else
                #define DRWAV_DLL_IMPORT
                #define DRWAV_DLL_EXPORT
                #define DRWAV_DLL_PRIVATE static
            #endif
        #endif

        #if defined(DR_WAV_IMPLEMENTATION) || defined(DRWAV_IMPLEMENTATION)
            #define DRWAV_API  DRWAV_DLL_EXPORT
        #else
            #define DRWAV_API  DRWAV_DLL_IMPORT
        #endif
        #define DRWAV_PRIVATE DRWAV_DLL_PRIVATE
    #else
        #define DRWAV_API extern
        #define DRWAV_PRIVATE static
    #endif

- 如果定义了DRWAV_DLL，则执行以下代码块
    - 如果定义了_WIN32，则定义DRWAV_DLL_IMPORT为__declspec(dllimport)，定义DRWAV_DLL_EXPORT为__declspec(dllexport)，定义DRWAV_DLL_PRIVATE为static
    - 否则，如果定义了__GNUC__并且__GNUC__大于等于4，则定义DRWAV_DLL_IMPORT为__attribute__((visibility("default")))，定义DRWAV_DLL_EXPORT为__attribute__((visibility("default")))，定义DRWAV_DLL_PRIVATE为__attribute__((visibility("hidden")))
    - 否则，定义DRWAV_DLL_IMPORT为空，定义DRWAV_DLL_EXPORT为空，定义DRWAV_DLL_PRIVATE为static
    - 如果定义了DR_WAV_IMPLEMENTATION或者定义了DRWAV_IMPLEMENTATION，则定义DRWAV_API为DRWAV_DLL_EXPORT
    - 否则，定义DRWAV_API为DRWAV_DLL_IMPORT
    - 定义DRWAV_PRIVATE为DRWAV_DLL_PRIVATE
- 否则，定义DRWAV_API为extern，定义DRWAV_PRIVATE为static
#ifndef

// 定义 drwav_result 类型为 drwav_int32
typedef drwav_int32 drwav_result;
// 定义 DRWAV_SUCCESS 为 0，表示成功
#define DRWAV_SUCCESS                        0
// 定义 DRWAV_ERROR 为 -1，表示通用错误
#define DRWAV_ERROR                         -1   /* A generic error. */
// 定义 DRWAV_INVALID_ARGS 为 -2，表示参数无效
#define DRWAV_INVALID_ARGS                  -2
// 定义 DRWAV_INVALID_OPERATION 为 -3，表示操作无效
#define DRWAV_INVALID_OPERATION             -3
// 定义 DRWAV_OUT_OF_MEMORY 为 -4，表示内存不足
#define DRWAV_OUT_OF_MEMORY                 -4
// 定义 DRWAV_OUT_OF_RANGE 为 -5，表示超出范围
#define DRWAV_OUT_OF_RANGE                  -5
// 定义 DRWAV_ACCESS_DENIED 为 -6，表示访问被拒绝
#define DRWAV_ACCESS_DENIED                 -6
// 定义 DRWAV_DOES_NOT_EXIST 为 -7，表示文件不存在
#define DRWAV_DOES_NOT_EXIST                -7
// 定义 DRWAV_ALREADY_EXISTS 为 -8，表示文件已存在
#define DRWAV_ALREADY_EXISTS                -8
// 定义 DRWAV_TOO_MANY_OPEN_FILES 为 -9，表示打开文件过多
#define DRWAV_TOO_MANY_OPEN_FILES           -9
// 定义 DRWAV_INVALID_FILE 为 -10，表示文件无效
#define DRWAV_INVALID_FILE                  -10
// 定义 DRWAV_TOO_BIG 为 -11，表示文件过大
#define DRWAV_TOO_BIG                       -11
// 定义 DRWAV_PATH_TOO_LONG 为 -12，表示路径过长
#define DRWAV_PATH_TOO_LONG                 -12
// 定义 DRWAV_NAME_TOO_LONG 为 -13，表示名称过长
#define DRWAV_NAME_TOO_LONG                 -13
// 定义 DRWAV_NOT_DIRECTORY 为 -14，表示不是目录
#define DRWAV_NOT_DIRECTORY                 -14
// 定义 DRWAV_IS_DIRECTORY 为 -15，表示是目录
#define DRWAV_IS_DIRECTORY                  -15
// 定义 DRWAV_DIRECTORY_NOT_EMPTY 为 -16，表示目录不为空
#define DRWAV_DIRECTORY_NOT_EMPTY           -16
// 定义 DRWAV_END_OF_FILE 为 -17，表示文件结束
#define DRWAV_END_OF_FILE                   -17
// 定义 DRWAV_NO_SPACE 为 -18，表示空间不足
#define DRWAV_NO_SPACE                      -18
// 定义 DRWAV_BUSY 为 -19，表示忙碌
#define DRWAV_BUSY                          -19
// 定义 DRWAV_IO_ERROR 为 -20，表示 IO 错误
#define DRWAV_IO_ERROR                      -20
// 定义 DRWAV_INTERRUPT 为 -21，表示中断
#define DRWAV_INTERRUPT                     -21
// 定义 DRWAV_UNAVAILABLE 为 -22，表示不可用
#define DRWAV_UNAVAILABLE                   -22
// 定义 DRWAV_ALREADY_IN_USE 为 -23，表示已在使用
#define DRWAV_ALREADY_IN_USE                -23
// 定义 DRWAV_BAD_ADDRESS 为 -24，表示地址错误
#define DRWAV_BAD_ADDRESS                   -24
// 定义 DRWAV_BAD_SEEK 为 -25，表示寻址错误
#define DRWAV_BAD_SEEK                      -25
// 定义 DRWAV_BAD_PIPE 为 -26，表示管道错误
#define DRWAV_BAD_PIPE                      -26
// 定义 DRWAV_DEADLOCK 为 -27，表示死锁
#define DRWAV_DEADLOCK                      -27
// 定义 DRWAV_TOO_MANY_LINKS 为 -28，表示链接过多
#define DRWAV_TOO_MANY_LINKS                -28
// 定义 DRWAV_NOT_IMPLEMENTED 为 -29，表示未实现
#define DRWAV_NOT_IMPLEMENTED               -29
// 定义 DRWAV_NO_MESSAGE 为 -30，表示无消息
#define DRWAV_NO_MESSAGE                    -30
// 定义 DRWAV_BAD_MESSAGE 为 -31，表示消息错误
#define DRWAV_BAD_MESSAGE                   -31
// 定义 DRWAV_NO_DATA_AVAILABLE 为 -32，表示无可用数据
#define DRWAV_NO_DATA_AVAILABLE             -32
// 定义 DRWAV_INVALID_DATA 为 -33，表示数据无效
#define DRWAV_INVALID_DATA                  -33
// 定义 DRWAV_TIMEOUT 为 -34，表示超时
#define DRWAV_TIMEOUT                       -34
// 定义 DRWAV_NO_NETWORK 为 -35，表示无网络
#define DRWAV_NO_NETWORK                    -35
// 定义 DRWAV_NOT_UNIQUE 为 -36，表示不唯一
#define DRWAV_NOT_UNIQUE                    -36
// 定义 DRWAV_NOT_SOCKET 为 -37，表示不是套接字
#define DRWAV_NOT_SOCKET                    -37
// 定义 DRWAV_NO_ADDRESS 为 -38，表示无地址
#define DRWAV_NO_ADDRESS                    -38
// 定义 DRWAV_BAD_PROTOCOL 为 -39，表示协议错误
#define DRWAV_BAD_PROTOCOL                  -39
// 定义一些错误码
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

// 定义一些常见的数据格式
/* Common data formats. */
#define DR_WAVE_FORMAT_PCM          0x1
#define DR_WAVE_FORMAT_ADPCM        0x2
#define DR_WAVE_FORMAT_IEEE_FLOAT   0x3
#define DR_WAVE_FORMAT_ALAW         0x6
#define DR_WAVE_FORMAT_MULAW        0x7
#define DR_WAVE_FORMAT_DVI_ADPCM    0x11
#define DR_WAVE_FORMAT_EXTENSIBLE   0xFFFE

// 定义常量
/* Constants. */
#ifndef DRWAV_MAX_SMPL_LOOPS
#define DRWAV_MAX_SMPL_LOOPS        1
#endif

// 定义一些标志位
/* Flags to pass into drwav_init_ex(), etc. */
#define DRWAV_SEQUENTIAL            0x00000001

// 声明函数
DRWAV_API void drwav_version(drwav_uint32* pMajor, drwav_uint32* pMinor, drwav_uint32* pRevision);
DRWAV_API const char* drwav_version_string(void);

// 定义枚举类型
typedef enum
{
    drwav_seek_origin_start,
    drwav_seek_origin_current
} drwav_seek_origin;

typedef enum
{
    drwav_container_riff,
    drwav_container_w64,
    drwav_container_rf64
} drwav_container;

// 定义结构体
typedef struct
{
    union
    {
        drwav_uint8 fourcc[4];
        drwav_uint8 guid[16];
    } id;

    /* The size in bytes of the chunk. */
    drwav_uint64 sizeInBytes;

    /*
    RIFF = 2 byte alignment.
    W64  = 8 byte alignment.
    */
    unsigned int paddingSize;
} drwav_chunk_header;

typedef struct
{
    /*
    The format tag exactly as specified in the wave file's "fmt" chunk. This can be used by applications
    */
    /* 需要支持 dr_wav 本身不支持的数据格式时使用的变量 */
    drwav_uint16 formatTag;

    /* 音频数据的通道数。当设置为1时为单声道，2为立体声，以此类推 */
    drwav_uint16 channels;

    /* 采样率。通常设置为类似44100的值 */
    drwav_uint32 sampleRate;

    /* 每秒的平均字节数。可能不常用，但这里提供信息用途 */
    drwav_uint32 avgBytesPerSec;

    /* 块对齐。等于通道数乘以每个样本的字节数 */
    drwav_uint16 blockAlign;

    /* 每个样本的位数 */
    drwav_uint16 bitsPerSample;

    /* 扩展数据的大小。仅用于内部验证，但这里提供信息用途 */
    drwav_uint16 extendedSize;

    /*
    每个样本的有效位数。当 <formatTag> 等于 WAVE_FORMAT_EXTENSIBLE 时，<bitsPerSample>
    总是向上取整到最接近的8的倍数。这个变量包含有关每个样本的有效位数的信息。主要用于信息目的。
    */
    drwav_uint16 validBitsPerSample;

    /* 通道掩码。目前未使用 */
    drwav_uint32 channelMask;

    /* 子格式，与波形文件中指定的完全相同 */
    drwav_uint8 subFormat[16];
/*
drwav_fmt 结构体定义，用于存储 WAV 文件的格式信息
*/
} drwav_fmt;

/*
获取 drwav_fmt 结构体中的格式信息
*/
DRWAV_API drwav_uint16 drwav_fmt_get_format(const drwav_fmt* pFMT);


/*
数据读取回调函数。返回值是实际读取的字节数。

pUserData   [in] 传递给 drwav_init() 等函数的用户数据
pBufferOut  [out] 输出缓冲区
bytesToRead [in] 要读取的字节数

返回实际读取的字节数。

返回值小于 bytesToRead 表示流的结束。在填充整个 bytesToRead 或到达流的末尾之前，不要从此回调函数返回。
*/
typedef size_t (* drwav_read_proc)(void* pUserData, void* pBufferOut, size_t bytesToRead);

/*
数据写入回调函数。返回值是实际写入的字节数。

pUserData    [in] 传递给 drwav_init_write() 等函数的用户数据
pData        [out] 要写入的数据指针
bytesToWrite [in] 要写入的字节数

返回实际写入的字节数。

如果返回值与 bytesToWrite 不同，表示出现错误。
*/
typedef size_t (* drwav_write_proc)(void* pUserData, const void* pData, size_t bytesToWrite);

/*
数据定位回调函数。

pUserData [in] 传递给 drwav_init() 等函数的用户数据
offset    [in] 要移动的字节数，相对于原点。永远不会是负数。
origin    [in] 定位的原点 - 当前位置或流的起始位置。

返回定位是否成功。

相对于起始位置或当前位置的确定由 "origin" 参数决定，该参数将是 drwav_seek_origin_start 或 drwav_seek_origin_current。
*/
typedef drwav_bool32 (* drwav_seek_proc)(void* pUserData, int offset, drwav_seek_origin origin);

/*
当 drwav_init_ex() 发现一个块时的回调函数。

pChunkUserData    [in] 传递给 drwav_init_ex() 等函数的 pChunkUserData 参数
*/
/*
    onRead：读取时调用的函数指针。
    onSeek：寻址时调用的函数指针。
    pReadSeekUserData：传递给drwav_init_ex()等函数的用户数据。
    pChunkHeader：包含有关块的基本头信息的对象的指针。用于识别块。
    container：WAV文件是RIFF还是Wave64容器。如果不确定差异，请假定为RIFF。
    pFMT：包含“fmt”块内容的对象的指针。

    返回读取和寻址的字节数总和。

    要从块中读取数据，请调用onRead()，将pReadSeekUserData作为第一个参数传入。对于寻址也是一样，使用onSeek()。返回值必须是您已读取和寻址的总字节数。

    使用`container`参数来区分`pChunkHeader->id`中的字段。如果容器是`drwav_container_riff`或`drwav_container_rf64`，则应使用`id.fourcc`，否则应使用`id.guid`。

    `pFMT`参数可用于确定波形文件的数据格式。使用`drwav_fmt_get_format()`获取采样格式，它将是`DR_WAVE_FORMAT_*`标识符之一。

    读取指针将位于块头之后的第一个字节。不得尝试读取超出块边界的内容。
*/
typedef drwav_uint64 (* drwav_chunk_proc)(void* pChunkUserData, drwav_read_proc onRead, drwav_seek_proc onSeek, void* pReadSeekUserData, const drwav_chunk_header* pChunkHeader, drwav_container container, const drwav_fmt* pFMT);

typedef struct
{
    void* pUserData;
    void* (* onMalloc)(size_t sz, void* pUserData);
    void* (* onRealloc)(void* p, size_t sz, void* pUserData);
    void  (* onFree)(void* p, void* pUserData);
} drwav_allocation_callbacks;

/* 仅用于使用drwav_init_memory()打开的加载器的内部使用的结构。 */
typedef struct
{
    # 声明一个指向无符号8位整数的指针变量data
    const drwav_uint8* data;
    # 声明一个变量dataSize，用于存储数据的大小
    size_t dataSize;
    # 声明一个变量currentReadPos，用于存储当前读取位置的偏移量
    size_t currentReadPos;
/* Structure for managing memory streams. */
typedef struct
{
    void** ppData;          /* Pointer to the pointer to the data. */
    size_t* pDataSize;      /* Pointer to the size of the data. */
    size_t dataSize;        /* Size of the data. */
    size_t dataCapacity;    /* Capacity of the data. */
    size_t currentWritePos; /* Current write position in the data. */
} drwav__memory_stream;

/* Structure for internal use. Only used for writers opened with drwav_init_memory_write(). */
typedef struct
{
    void** ppData;          /* Pointer to the pointer to the data. */
    size_t* pDataSize;      /* Pointer to the size of the data. */
    size_t dataSize;        /* Size of the data. */
    size_t dataCapacity;    /* Capacity of the data. */
    size_t currentWritePos; /* Current write position in the data. */
} drwav__memory_stream_write;

typedef struct
{
    drwav_container container;  /* RIFF, W64. */
    drwav_uint32 format;        /* DR_WAVE_FORMAT_* */
    drwav_uint32 channels;      /* Number of channels. */
    drwav_uint32 sampleRate;    /* Sample rate. */
    drwav_uint32 bitsPerSample; /* Bits per sample. */
} drwav_data_format;

/* Structure for defining loop points in WAV files. */
typedef struct
{
    drwav_uint32 cuePointId;   /* Cue point ID. */
    drwav_uint32 type;         /* Type of loop. */
    drwav_uint32 start;        /* Start point of loop. */
    drwav_uint32 end;          /* End point of loop. */
    drwav_uint32 fraction;     /* Fraction. */
    drwav_uint32 playCount;    /* Number of times to play the loop. */
} drwav_smpl_loop;

/* Structure for defining sampler information in WAV files. */
typedef struct
{
    drwav_uint32 manufacturer;      /* Manufacturer. */
    drwav_uint32 product;           /* Product. */
    drwav_uint32 samplePeriod;      /* Sample period. */
    drwav_uint32 midiUnityNotes;    /* MIDI unity notes. */
    drwav_uint32 midiPitchFraction; /* MIDI pitch fraction. */
    drwav_uint32 smpteFormat;       /* SMPTE format. */
    drwav_uint32 smpteOffset;       /* SMPTE offset. */
    drwav_uint32 numSampleLoops;    /* Number of sample loops. */
    drwav_uint32 samplerData;       /* Sampler data. */
    drwav_smpl_loop loops[DRWAV_MAX_SMPL_LOOPS]; /* Array of sample loops. */
} drwav_smpl;

typedef struct
{
    /* Pointer to the function to call when more data is needed. */
    drwav_read_proc onRead;

    /* Pointer to the function to call when data needs to be written. Only used when the drwav object is opened in write mode. */
    drwav_write_proc onWrite;

    /* Pointer to the function to call when the wav file needs to be seeked. */
    drwav_seek_proc onSeek;

    /* The user data to pass to callbacks. */
    void* pUserData;

    /* Allocation callbacks. */
    drwav_allocation_callbacks allocationCallbacks;

    /* Whether or not the WAV file is formatted as a standard RIFF file or W64. */
    drwav_container container;

    /* Structure containing format information exactly as specified by the wav file. */
    drwav_fmt fmt;
    /* 采样率。通常设置为类似44100的值。 */
    drwav_uint32 sampleRate;

    /* 声道数。对于单声道流设置为1，立体声设置为2，以此类推。 */
    drwav_uint16 channels;

    /* 每个样本的位数。通常设置为16、24等。 */
    drwav_uint16 bitsPerSample;

    /* 等于 fmt.formatTag，或者如果 fmt.formatTag 等于65534（WAVE_FORMAT_EXTENSIBLE）则等于 fmt.subFormat 指定的值。 */
    drwav_uint16 translatedFormatTag;

    /* 构成音频数据的 PCM 帧的总数。 */
    drwav_uint64 totalPCMFrameCount;


    /* 数据块的字节大小。 */
    drwav_uint64 dataChunkDataSize;
    
    /* 数据块中第一个字节的位置。用于定位。 */
    drwav_uint64 dataChunkDataPos;

    /* 数据块中剩余的字节数。 */
    drwav_uint64 bytesRemaining;


    /*
    仅在顺序写模式下使用。在初始化时跟踪“data”块的期望大小。对于非顺序写和以读模式打开的 drwav 对象，始终设置为0。用于验证。
    */
    drwav_uint64 dataChunkDataSizeTargetWrite;

    /* 跟踪 wav 写入器是否在顺序模式下初始化。 */
    drwav_bool32 isSequentialWrite;


    /* smpl 块。 */
    drwav_smpl smpl;


    /* 一个避免在使用 drwav_init_memory() 打开解码器时进行 DRWAV_MALLOC() 的技巧。 */
    drwav__memory_stream memoryStream;
    drwav__memory_stream_write memoryStreamWrite;

    /* 压缩格式的通用数据。这些数据在所有块压缩格式之间共享。 */
    struct
    {
        drwav_uint64 iCurrentPCMFrame;  /* 下一个由 drwav_read_*() 读取的 PCM 帧的索引。与“totalPCMFrameCount”一起使用，以确保我们不会在最后一个块的末尾读取过多的样本。 */
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
        # 在解码过程中用于缓存样本的数组
        drwav_int32  cachedFrames[4];  /* Samples are stored in this cache during decoding. */
        # 缓存的帧数
        drwav_uint32 cachedFrameCount;
        # 为每个通道存储前两个样本的数组
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
        # 在解码过程中用于缓存样本的数组
        drwav_int32  cachedFrames[16]; /* Samples are stored in this cache during decoding. */
        # 缓存的帧数
        drwav_uint32 cachedFrameCount;
    } ima;
/*
初始化一个预先分配的 drwav 对象用于读取。

pWav                         [out]          被初始化的 drwav 对象的指针。
onRead                       [in]           当需要从客户端读取数据时调用的函数。
onSeek                       [in]           当客户端数据的读取位置需要移动时调用的函数。
onChunk                      [in, optional] 在初始化时枚举块时调用的函数。
pUserData, pReadSeekUserData [in, optional] 一个指向应用程序定义数据的指针，将传递给 onRead 和 onSeek。
pChunkUserData               [in, optional] 一个指向应用程序定义数据的指针，将传递给 onChunk。
flags                        [in, optional] 用于控制加载方式的一组标志。

如果成功返回 true；否则返回 false。

使用 drwav_uninit() 关闭加载器。

这是初始化 WAV 文件的最低级函数。您也可以使用 drwav_init_file() 和 drwav_init_memory() 分别从文件或内存块打开流。

flags 的可能值：
  DRWAV_SEQUENTIAL: 在加载时永远不执行向后查找。这将禁用块回调，并且将导致此函数在找到数据块后立即返回。数据块后的任何块将被忽略。

drwav_init() 等效于 "drwav_init_ex(pWav, onRead, onSeek, NULL, pUserData, NULL, 0);".

WAVE 或 FMT 块不会调用 onChunk 回调。在函数返回后，可以从 pWav->fmt 读取 FMT 块的内容。

另请参阅：drwav_init_file()、drwav_init_memory()、drwav_uninit()
*/
DRWAV_API drwav_bool32 drwav_init(drwav* pWav, drwav_read_proc onRead, drwav_seek_proc onSeek, void* pUserData, const drwav_allocation_callbacks* pAllocationCallbacks);
# 初始化一个预先分配的 drwav 对象用于写入
DRWAV_API drwav_bool32 drwav_init_ex(drwav* pWav, drwav_read_proc onRead, drwav_seek_proc onSeek, drwav_chunk_proc onChunk, void* pReadSeekUserData, void* pChunkUserData, drwav_uint32 flags, const drwav_allocation_callbacks* pAllocationCallbacks);

/*
初始化一个预先分配的 drwav 对象用于写入。

onWrite   [in]           当需要写入数据时调用的函数。
onSeek    [in]           当写入位置需要移动时调用的函数。
pUserData [in, optional] 一个指向应用程序定义数据的指针，将传递给 onWrite 和 onSeek。

如果成功返回 true；否则返回 false。

使用 drwav_uninit() 关闭写入器。

这是初始化 WAV 文件的最低级函数。您还可以使用 drwav_init_file_write() 和 drwav_init_memory_write() 从文件或内存块中打开流。

如果总样本数已知，可以使用 drwav_init_write_sequential()。这避免了 dr_wav 执行后处理步骤来存储总样本数和数据块的大小，这需要进行反向查找。

另请参阅：drwav_init_file_write()，drwav_init_memory_write()，drwav_uninit()
*/
DRWAV_API drwav_bool32 drwav_init_write(drwav* pWav, const drwav_data_format* pFormat, drwav_write_proc onWrite, drwav_seek_proc onSeek, void* pUserData, const drwav_allocation_callbacks* pAllocationCallbacks);
DRWAV_API drwav_bool32 drwav_init_write_sequential(drwav* pWav, const drwav_data_format* pFormat, drwav_uint64 totalSampleCount, drwav_write_proc onWrite, void* pUserData, const drwav_allocation_callbacks* pAllocationCallbacks);
DRWAV_API drwav_bool32 drwav_init_write_sequential_pcm_frames(drwav* pWav, const drwav_data_format* pFormat, drwav_uint64 totalPCMFrameCount, drwav_write_proc onWrite, void* pUserData, const drwav_allocation_callbacks* pAllocationCallbacks);

/*
确定要写入的整个数据的目标大小（包括所有头部和块）的实用函数。
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
*/ 
# 读取 Little-Endian 格式的 PCM 帧数据
DRWAV_API drwav_uint64 drwav_read_pcm_frames_le(drwav* pWav, drwav_uint64 framesToRead, void* pBufferOut);

# 读取 Big-Endian 格式的 PCM 帧数据
DRWAV_API drwav_uint64 drwav_read_pcm_frames_be(drwav* pWav, drwav_uint64 framesToRead, void* pBufferOut);

# 定位到给定的 PCM 帧
# 如果成功则返回 true；否则返回 false
DRWAV_API drwav_bool32 drwav_seek_to_pcm_frame(drwav* pWav, drwav_uint64 targetFrameIndex);

# 写入原始音频数据
# 返回实际写入的字节数。如果与 bytesToWrite 不同，表示出现错误
DRWAV_API size_t drwav_write_raw(drwav* pWav, size_t bytesToWrite, const void* pData);

# 写入 PCM 帧数据
# 返回实际写入的 PCM 帧数
# 输入样本需要是本机字节顺序。在大端架构上，输入数据将被转换为小端。使用 drwav_write_raw() 来写入原始音频数据而不执行任何转换
DRWAV_API drwav_uint64 drwav_write_pcm_frames(drwav* pWav, drwav_uint64 framesToWrite, const void* pData);
DRWAV_API drwav_uint64 drwav_write_pcm_frames_le(drwav* pWav, drwav_uint64 framesToWrite, const void* pData);
DRWAV_API drwav_uint64 drwav_write_pcm_frames_be(drwav* pWav, drwav_uint64 framesToWrite, const void* pData);

# 转换工具
#ifndef DR_WAV_NO_CONVERSION_API

# 读取一块音频数据并将其转换为有符号 16 位 PCM 样本
# 如果 pBufferOut 为 NULL，则执行定位操作
# 返回实际读取的 PCM 帧数
# 如果返回值小于 framesToRead，则表示已到达文件末尾
DRWAV_API drwav_uint64 drwav_read_pcm_frames_s16(drwav* pWav, drwav_uint64 framesToRead, drwav_int16* pBufferOut);
DRWAV_API drwav_uint64 drwav_read_pcm_frames_s16le(drwav* pWav, drwav_uint64 framesToRead, drwav_int16* pBufferOut);
DRWAV_API drwav_uint64 drwav_read_pcm_frames_s16be(drwav* pWav, drwav_uint64 framesToRead, drwav_int16* pBufferOut);
/* Low-level function for converting unsigned 8-bit PCM samples to signed 16-bit PCM samples. */
DRWAV_API void drwav_u8_to_s16(drwav_int16* pOut, const drwav_uint8* pIn, size_t sampleCount);

/* Low-level function for converting signed 24-bit PCM samples to signed 16-bit PCM samples. */
DRWAV_API void drwav_s24_to_s16(drwav_int16* pOut, const drwav_uint8* pIn, size_t sampleCount);

/* Low-level function for converting signed 32-bit PCM samples to signed 16-bit PCM samples. */
DRWAV_API void drwav_s32_to_s16(drwav_int16* pOut, const drwav_int32* pIn, size_t sampleCount);

/* Low-level function for converting IEEE 32-bit floating point samples to signed 16-bit PCM samples. */
DRWAV_API void drwav_f32_to_s16(drwav_int16* pOut, const float* pIn, size_t sampleCount);

/* Low-level function for converting IEEE 64-bit floating point samples to signed 16-bit PCM samples. */
DRWAV_API void drwav_f64_to_s16(drwav_int16* pOut, const double* pIn, size_t sampleCount);

/* Low-level function for converting A-law samples to signed 16-bit PCM samples. */
DRWAV_API void drwav_alaw_to_s16(drwav_int16* pOut, const drwav_uint8* pIn, size_t sampleCount);

/* Low-level function for converting u-law samples to signed 16-bit PCM samples. */
DRWAV_API void drwav_mulaw_to_s16(drwav_int16* pOut, const drwav_uint8* pIn, size_t sampleCount);


/*
Reads a chunk of audio data and converts it to IEEE 32-bit floating point samples.

pBufferOut can be NULL in which case a seek will be performed.

Returns the number of PCM frames actually read.

If the return value is less than <framesToRead> it means the end of the file has been reached.
*/
DRWAV_API drwav_uint64 drwav_read_pcm_frames_f32(drwav* pWav, drwav_uint64 framesToRead, float* pBufferOut);
DRWAV_API drwav_uint64 drwav_read_pcm_frames_f32le(drwav* pWav, drwav_uint64 framesToRead, float* pBufferOut);
DRWAV_API drwav_uint64 drwav_read_pcm_frames_f32be(drwav* pWav, drwav_uint64 framesToRead, float* pBufferOut);
/* Low-level function for converting unsigned 8-bit PCM samples to IEEE 32-bit floating point samples. */
DRWAV_API void drwav_u8_to_f32(float* pOut, const drwav_uint8* pIn, size_t sampleCount);

/* Low-level function for converting signed 16-bit PCM samples to IEEE 32-bit floating point samples. */
DRWAV_API void drwav_s16_to_f32(float* pOut, const drwav_int16* pIn, size_t sampleCount);

/* Low-level function for converting signed 24-bit PCM samples to IEEE 32-bit floating point samples. */
DRWAV_API void drwav_s24_to_f32(float* pOut, const drwav_uint8* pIn, size_t sampleCount);

/* Low-level function for converting signed 32-bit PCM samples to IEEE 32-bit floating point samples. */
DRWAV_API void drwav_s32_to_f32(float* pOut, const drwav_int32* pIn, size_t sampleCount);

/* Low-level function for converting IEEE 64-bit floating point samples to IEEE 32-bit floating point samples. */
DRWAV_API void drwav_f64_to_f32(float* pOut, const double* pIn, size_t sampleCount);

/* Low-level function for converting A-law samples to IEEE 32-bit floating point samples. */
DRWAV_API void drwav_alaw_to_f32(float* pOut, const drwav_uint8* pIn, size_t sampleCount);

/* Low-level function for converting u-law samples to IEEE 32-bit floating point samples. */
DRWAV_API void drwav_mulaw_to_f32(float* pOut, const drwav_uint8* pIn, size_t sampleCount);


/*
Reads a chunk of audio data and converts it to signed 32-bit PCM samples.

pBufferOut can be NULL in which case a seek will be performed.

Returns the number of PCM frames actually read.

If the return value is less than <framesToRead> it means the end of the file has been reached.
*/
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

这会保持内部FILE对象，直到调用drwav_uninit()为止。如果您正在缓存drwav对象，请记住这一点，因为操作系统可能限制应用程序在任何给定时间打开的文件句柄数量。
*/
DRWAV_API drwav_bool32 drwav_init_file(drwav* pWav, const char* filename, const drwav_allocation_callbacks* pAllocationCallbacks);
# 初始化一个 wave 文件用于读取，接受文件名、回调函数、回调函数参数、标志和内存分配回调函数
DRWAV_API drwav_bool32 drwav_init_file_ex(drwav* pWav, const char* filename, drwav_chunk_proc onChunk, void* pChunkUserData, drwav_uint32 flags, const drwav_allocation_callbacks* pAllocationCallbacks);
# 初始化一个 wave 文件用于读取，接受宽字符文件名和内存分配回调函数
DRWAV_API drwav_bool32 drwav_init_file_w(drwav* pWav, const wchar_t* filename, const drwav_allocation_callbacks* pAllocationCallbacks);
# 初始化一个 wave 文件用于读取，接受宽字符文件名、回调函数、回调函数参数、标志和内存分配回调函数

DRWAV_API drwav_bool32 drwav_init_file_ex_w(drwav* pWav, const wchar_t* filename, drwav_chunk_proc onChunk, void* pChunkUserData, drwav_uint32 flags, const drwav_allocation_callbacks* pAllocationCallbacks);

'''
用于使用 stdio 初始化一个用于写入的 wave 文件的辅助函数。

这会持有内部的 FILE 对象，直到调用 drwav_uninit()。如果您正在缓存 drwav 对象，请记住这一点，因为操作系统可能会限制应用程序在任何给定时间内打开的文件句柄数量。
'''
# 初始化一个用于写入的 wave 文件，接受文件名、数据格式、内存分配回调函数
DRWAV_API drwav_bool32 drwav_init_file_write(drwav* pWav, const char* filename, const drwav_data_format* pFormat, const drwav_allocation_callbacks* pAllocationCallbacks);
# 初始化一个用于写入的 wave 文件，接受文件名、数据格式、总样本数和内存分配回调函数
DRWAV_API drwav_bool32 drwav_init_file_write_sequential(drwav* pWav, const char* filename, const drwav_data_format* pFormat, drwav_uint64 totalSampleCount, const drwav_allocation_callbacks* pAllocationCallbacks);
# 初始化一个用于写入的 wave 文件，接受文件名、数据格式、总 PCM 帧数和内存分配回调函数
DRWAV_API drwav_bool32 drwav_init_file_write_sequential_pcm_frames(drwav* pWav, const char* filename, const drwav_data_format* pFormat, drwav_uint64 totalPCMFrameCount, const drwav_allocation_callbacks* pAllocationCallbacks);
# 初始化一个用于写入的 wave 文件，接受宽字符文件名、数据格式和内存分配回调函数
DRWAV_API drwav_bool32 drwav_init_file_write_w(drwav* pWav, const wchar_t* filename, const drwav_data_format* pFormat, const drwav_allocation_callbacks* pAllocationCallbacks);
# 初始化一个用于写入的 wave 文件，接受宽字符文件名、数据格式、总样本数和内存分配回调函数
DRWAV_API drwav_bool32 drwav_init_file_write_sequential_w(drwav* pWav, const wchar_t* filename, const drwav_data_format* pFormat, drwav_uint64 totalSampleCount, const drwav_allocation_callbacks* pAllocationCallbacks);
# 初始化一个用于顺序写入 PCM 帧的 WAV 文件的函数，接受文件名、数据格式、总 PCM 帧数和内存分配回调
DRWAV_API drwav_bool32 drwav_init_file_write_sequential_pcm_frames_w(drwav* pWav, const wchar_t* filename, const drwav_data_format* pFormat, drwav_uint64 totalPCMFrameCount, const drwav_allocation_callbacks* pAllocationCallbacks);
#endif  /* DR_WAV_NO_STDIO */

/*
用于从预先分配的内存缓冲区初始化加载器的辅助函数。

这不会创建数据的副本。应用程序需要确保缓冲区在 drwav 对象的生命周期内保持有效。

缓冲区应包含整个波形文件的内容，而不仅仅是样本数据。
*/
DRWAV_API drwav_bool32 drwav_init_memory(drwav* pWav, const void* data, size_t dataSize, const drwav_allocation_callbacks* pAllocationCallbacks);
DRWAV_API drwav_bool32 drwav_init_memory_ex(drwav* pWav, const void* data, size_t dataSize, drwav_chunk_proc onChunk, void* pChunkUserData, drwav_uint32 flags, const drwav_allocation_callbacks* pAllocationCallbacks);

/*
用于初始化将数据输出到内存缓冲区的写入器的辅助函数。

dr_wav 将管理内存分配，但调用者需要使用 drwav_free() 释放数据。

即使调用 drwav_uninit() 后，缓冲区仍将保持分配。在调用 drwav_uninit() 后，缓冲区不应被视为有效。
*/
DRWAV_API drwav_bool32 drwav_init_memory_write(drwav* pWav, void** ppData, size_t* pDataSize, const drwav_data_format* pFormat, const drwav_allocation_callbacks* pAllocationCallbacks);
DRWAV_API drwav_bool32 drwav_init_memory_write_sequential(drwav* pWav, void** ppData, size_t* pDataSize, const drwav_data_format* pFormat, drwav_uint64 totalSampleCount, const drwav_allocation_callbacks* pAllocationCallbacks);
DRWAV_API drwav_bool32 drwav_init_memory_write_sequential_pcm_frames(drwav* pWav, void** ppData, size_t* pDataSize, const drwav_data_format* pFormat, drwav_uint64 totalPCMFrameCount, const drwav_allocation_callbacks* pAllocationCallbacks);


#ifndef DR_WAV_NO_CONVERSION_API
/*
/*
以单个操作打开并读取整个 wav 文件。

返回值是一个堆分配的缓冲区，包含音频数据。使用 drwav_free() 释放缓冲区。
*/
DRWAV_API drwav_int16* drwav_open_and_read_pcm_frames_s16(drwav_read_proc onRead, drwav_seek_proc onSeek, void* pUserData, unsigned int* channelsOut, unsigned int* sampleRateOut, drwav_uint64* totalFrameCountOut, const drwav_allocation_callbacks* pAllocationCallbacks);
DRWAV_API float* drwav_open_and_read_pcm_frames_f32(drwav_read_proc onRead, drwav_seek_proc onSeek, void* pUserData, unsigned int* channelsOut, unsigned int* sampleRateOut, drwav_uint64* totalFrameCountOut, const drwav_allocation_callbacks* pAllocationCallbacks);
DRWAV_API drwav_int32* drwav_open_and_read_pcm_frames_s32(drwav_read_proc onRead, drwav_seek_proc onSeek, void* pUserData, unsigned int* channelsOut, unsigned int* sampleRateOut, drwav_uint64* totalFrameCountOut, const drwav_allocation_callbacks* pAllocationCallbacks);
#ifndef DR_WAV_NO_STDIO
/*
以单个操作打开并解码整个 wav 文件。

返回值是一个堆分配的缓冲区，包含音频数据。使用 drwav_free() 释放缓冲区。
*/
DRWAV_API drwav_int16* drwav_open_file_and_read_pcm_frames_s16(const char* filename, unsigned int* channelsOut, unsigned int* sampleRateOut, drwav_uint64* totalFrameCountOut, const drwav_allocation_callbacks* pAllocationCallbacks);
DRWAV_API float* drwav_open_file_and_read_pcm_frames_f32(const char* filename, unsigned int* channelsOut, unsigned int* sampleRateOut, drwav_uint64* totalFrameCountOut, const drwav_allocation_callbacks* pAllocationCallbacks);
DRWAV_API drwav_int32* drwav_open_file_and_read_pcm_frames_s32(const char* filename, unsigned int* channelsOut, unsigned int* sampleRateOut, drwav_uint64* totalFrameCountOut, const drwav_allocation_callbacks* pAllocationCallbacks);
// 从文件名打开并读取 PCM 帧数据，返回带有 int16 类型的数据指针
DRWAV_API drwav_int16* drwav_open_file_and_read_pcm_frames_s16_w(const wchar_t* filename, unsigned int* channelsOut, unsigned int* sampleRateOut, drwav_uint64* totalFrameCountOut, const drwav_allocation_callbacks* pAllocationCallbacks);

// 从文件名打开并读取 PCM 帧数据，返回带有 float 类型的数据指针
DRWAV_API float* drwav_open_file_and_read_pcm_frames_f32_w(const wchar_t* filename, unsigned int* channelsOut, unsigned int* sampleRateOut, drwav_uint64* totalFrameCountOut, const drwav_allocation_callbacks* pAllocationCallbacks);

// 从文件名打开并读取 PCM 帧数据，返回带有 int32 类型的数据指针
DRWAV_API drwav_int32* drwav_open_file_and_read_pcm_frames_s32_w(const wchar_t* filename, unsigned int* channelsOut, unsigned int* sampleRateOut, drwav_uint64* totalFrameCountOut, const drwav_allocation_callbacks* pAllocationCallbacks);

// 从内存块中打开并解码整个 WAV 文件
// 返回值是一个堆分配的缓冲区，包含音频数据。使用 drwav_free() 释放缓冲区。
DRWAV_API drwav_int16* drwav_open_memory_and_read_pcm_frames_s16(const void* data, size_t dataSize, unsigned int* channelsOut, unsigned int* sampleRateOut, drwav_uint64* totalFrameCountOut, const drwav_allocation_callbacks* pAllocationCallbacks);

// 从内存块中打开并解码整个 WAV 文件，返回带有 float 类型的数据指针
DRWAV_API float* drwav_open_memory_and_read_pcm_frames_f32(const void* data, size_t dataSize, unsigned int* channelsOut, unsigned int* sampleRateOut, drwav_uint64* totalFrameCountOut, const drwav_allocation_callbacks* pAllocationCallbacks);

// 从内存块中打开并解码整个 WAV 文件，返回带有 int32 类型的数据指针
DRWAV_API drwav_int32* drwav_open_memory_and_read_pcm_frames_s32(const void* data, size_t dataSize, unsigned int* channelsOut, unsigned int* sampleRateOut, drwav_uint64* totalFrameCountOut, const drwav_allocation_callbacks* pAllocationCallbacks);

// 释放由 dr_wav 内部分配的数据
DRWAV_API void drwav_free(void* p, const drwav_allocation_callbacks* pAllocationCallbacks);

// 将 WAV 流中的字节转换为本机字节序的指定类型
DRWAV_API drwav_uint16 drwav_bytes_to_u16(const drwav_uint8* data);
// 将字节数组转换为有符号16位整数
DRWAV_API drwav_int16 drwav_bytes_to_s16(const drwav_uint8* data);

// 将字节数组转换为无符号32位整数
DRWAV_API drwav_uint32 drwav_bytes_to_u32(const drwav_uint8* data);

// 将字节数组转换为有符号32位整数
DRWAV_API drwav_int32 drwav_bytes_to_s32(const drwav_uint8* data);

// 将字节数组转换为无符号64位整数
DRWAV_API drwav_uint64 drwav_bytes_to_u64(const drwav_uint8* data);

// 将字节数组转换为有符号64位整数
DRWAV_API drwav_int64 drwav_bytes_to_s64(const drwav_uint8* data);

// 比较两个GUID，用于检查Wave64块的类型
DRWAV_API drwav_bool32 drwav_guid_equal(const drwav_uint8 a[16], const drwav_uint8 b[16]);

// 比较四字符代码，用于检查RIFF块的类型
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

// 标准库相关
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
#ifndef DRWAV_FREE
// 定义释放内存的宏，使用 free 函数
#define DRWAV_FREE(p)                      free((p))
#endif

#ifndef DRWAV_COPY_MEMORY
// 定义复制内存的宏，使用 memcpy 函数
#define DRWAV_COPY_MEMORY(dst, src, sz)    memcpy((dst), (src), (sz))
#endif

#ifndef DRWAV_ZERO_MEMORY
// 定义清零内存的宏，使用 memset 函数
#define DRWAV_ZERO_MEMORY(p, sz)           memset((p), 0, (sz))
#endif

#ifndef DRWAV_ZERO_OBJECT
// 定义清零对象的宏，调用清零内存的宏
#define DRWAV_ZERO_OBJECT(p)               DRWAV_ZERO_MEMORY((p), sizeof(*p))
#endif

// 计算数组元素个数的宏
#define drwav_countof(x)                   (sizeof(x) / sizeof(x[0]))

// 对齐到指定大小的宏
#define drwav_align(x, a)                  ((((x) + (a) - 1) / (a)) * (a))

// 返回两个数中较小的数的宏
#define drwav_min(a, b)                    (((a) < (b)) ? (a) : (b))

// 返回两个数中较大的数的宏
#define drwav_max(a, b)                    (((a) > (b)) ? (a) : (b))

// 将数值限制在指定范围内的宏
#define drwav_clamp(x, lo, hi)             (drwav_max((lo), drwav_min((hi), (x))))

// 定义最大 SIMD 向量大小
#define DRWAV_MAX_SIMD_VECTOR_SIZE         64  /* 64 for AVX-512 in the future. */

/* CPU architecture. */
#if defined(__x86_64__) || defined(_M_X64)
    #define DRWAV_X64
#elif defined(__i386) || defined(_M_IX86)
    #define DRWAV_X86
#elif defined(__arm__) || defined(_M_ARM)
    #define DRWAV_ARM
#endif

#ifdef _MSC_VER
    // 定义 MSC 编译器下的内联函数
    #define DRWAV_INLINE __forceinline
#elif defined(__GNUC__)
    /*
    I've had a bug report where GCC is emitting warnings about functions possibly not being inlineable. This warning happens when
    the __attribute__((always_inline)) attribute is defined without an "inline" statement. I think therefore there must be some
    case where "__inline__" is not always defined, thus the compiler emitting these warnings. When using -std=c89 or -ansi on the
    command line, we cannot use the "inline" keyword and instead need to use "__inline__". In an attempt to work around this issue
    I am using "__inline__" only when we're compiling in strict ANSI mode.
    */
    #if defined(__STRICT_ANSI__)
        // 在严格 ANSI 模式下定义 GCC 编译器下的内联函数
        #define DRWAV_INLINE __inline__ __attribute__((always_inline))
    #else
        // 在非严格 ANSI 模式下定义 GCC 编译器下的内联函数
        #define DRWAV_INLINE inline __attribute__((always_inline))
    #endif
#elif defined(__WATCOMC__)
    // 定义 Watcom 编译器下的内联函数
    #define DRWAV_INLINE __inline
#else
    #define DRWAV_INLINE

这行代码定义了一个宏，用于指示编译器将函数作为内联函数进行处理。
#endif

#if defined(SIZE_MAX)
    // 如果定义了 SIZE_MAX，使用 SIZE_MAX 作为 DRWAV_SIZE_MAX 的值
    #define DRWAV_SIZE_MAX  SIZE_MAX
#else
    // 如果未定义 SIZE_MAX，则根据不同平台设置 DRWAV_SIZE_MAX 的值
    #if defined(_WIN64) || defined(_LP64) || defined(__LP64__)
        // 对于 64 位平台，设置 DRWAV_SIZE_MAX 为 64 位最大值
        #define DRWAV_SIZE_MAX  ((drwav_uint64)0xFFFFFFFFFFFFFFFF)
    #else
        // 对于非 64 位平台，设置 DRWAV_SIZE_MAX 为 32 位最大值
        #define DRWAV_SIZE_MAX  0xFFFFFFFF
    #endif
#endif

#if defined(_MSC_VER) && _MSC_VER >= 1400
    // 如果是 MSVC 编译器且版本大于等于 1400，启用字节交换的内联函数
    #define DRWAV_HAS_BYTESWAP16_INTRINSIC
    #define DRWAV_HAS_BYTESWAP32_INTRINSIC
    #define DRWAV_HAS_BYTESWAP64_INTRINSIC
#elif defined(__clang__)
    // 如果是 Clang 编译器，检查是否支持内建函数
    #if defined(__has_builtin)
        // 如果支持 __builtin_bswap16，启用字节交换的内联函数
        #if __has_builtin(__builtin_bswap16)
            #define DRWAV_HAS_BYTESWAP16_INTRINSIC
        #endif
        // 如果支持 __builtin_bswap32，启用字节交换的内联函数
        #if __has_builtin(__builtin_bswap32)
            #define DRWAV_HAS_BYTESWAP32_INTRINSIC
        #endif
        // 如果支持 __builtin_bswap64，启用字节交换的内联函数
        #if __has_builtin(__builtin_bswap64)
            #define DRWAV_HAS_BYTESWAP64_INTRINSIC
        #endif
    #endif
#elif defined(__GNUC__)
    // 如果是 GCC 编译器，根据版本设置字节交换的内联函数支持
    #if ((__GNUC__ > 4) || (__GNUC__ == 4 && __GNUC_MINOR__ >= 3))
        // GCC 版本大于等于 4.3 支持 32 位和 64 位字节交换的内联函数
        #define DRWAV_HAS_BYTESWAP32_INTRINSIC
        #define DRWAV_HAS_BYTESWAP64_INTRINSIC
    #endif
    #if ((__GNUC__ > 4) || (__GNUC__ == 4 && __GNUC_MINOR__ >= 8))
        // GCC 版本大于等于 4.8 支持 16 位字节交换的内联函数
        #define DRWAV_HAS_BYTESWAP16_INTRINSIC
    #endif
#endif

// 获取 dr_wav 版本信息
DRWAV_API void drwav_version(drwav_uint32* pMajor, drwav_uint32* pMinor, drwav_uint32* pRevision)
{
    // 获取主版本号
    if (pMajor) {
        *pMajor = DRWAV_VERSION_MAJOR;
    }

    // 获取次版本号
    if (pMinor) {
        *pMinor = DRWAV_VERSION_MINOR;
    }

    // 获取修订版本号
    if (pRevision) {
        *pRevision = DRWAV_VERSION_REVISION;
    }
}

// 返回 dr_wav 版本字符串
DRWAV_API const char* drwav_version_string(void)
{
    return DRWAV_VERSION_STRING;
}

/*
这些限制用于初始化解码器时的基本验证。如果超出这些限制，首先：你到底在做什么？！（告诉我，我会很好奇！）其次，您可以在 dr_wav 实现之前通过 #define 调整这些限制。
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



// 如果未定义 DRWAV_MAX_BITS_PER_SAMPLE，则设置最大比特率为 64



static const drwav_uint8 drwavGUID_W64_RIFF[16] = {0x72,0x69,0x66,0x66, 0x2E,0x91, 0xCF,0x11, 0xA5,0xD6, 0x28,0xDB,0x04,0xC1,0x00,0x00};    /* 66666972-912E-11CF-A5D6-28DB04C10000 */
static const drwav_uint8 drwavGUID_W64_WAVE[16] = {0x77,0x61,0x76,0x65, 0xF3,0xAC, 0xD3,0x11, 0x8C,0xD1, 0x00,0xC0,0x4F,0x8E,0xDB,0x8A};    /* 65766177-ACF3-11D3-8CD1-00C04F8EDB8A */
//static const drwav_uint8 drwavGUID_W64_JUNK[16] = {0x6A,0x75,0x6E,0x6B, 0xF3,0xAC, 0xD3,0x11, 0x8C,0xD1, 0x00,0xC0,0x4F,0x8E,0xDB,0x8A};    /* 6B6E756A-ACF3-11D3-8CD1-00C04F8EDB8A */
static const drwav_uint8 drwavGUID_W64_FMT [16] = {0x66,0x6D,0x74,0x20, 0xF3,0xAC, 0xD3,0x11, 0x8C,0xD1, 0x00,0xC0,0x4F,0x8E,0xDB,0x8A};    /* 20746D66-ACF3-11D3-8CD1-00C04F8EDB8A */
static const drwav_uint8 drwavGUID_W64_FACT[16] = {0x66,0x61,0x63,0x74, 0xF3,0xAC, 0xD3,0x11, 0x8C,0xD1, 0x00,0xC0,0x4F,0x8E,0xDB,0x8A};    /* 74636166-ACF3-11D3-8CD1-00C04F8EDB8A */
static const drwav_uint8 drwavGUID_W64_DATA[16] = {0x64,0x61,0x74,0x61, 0xF3,0xAC, 0xD3,0x11, 0x8C,0xD1, 0x00,0xC0,0x4F,0x8E,0xDB,0x8A};    /* 61746164-ACF3-11D3-8CD1-00C04F8EDB8A */
static const drwav_uint8 drwavGUID_W64_SMPL[16] = {0x73,0x6D,0x70,0x6C, 0xF3,0xAC, 0xD3,0x11, 0x8C,0xD1, 0x00,0xC0,0x4F,0x8E,0xDB,0x8A};    /* 6C706D73-ACF3-11D3-8CD1-00C04F8EDB8A */



// 定义 W64 文件格式的 GUID



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



// 比较两个 GUID 是否相等



static DRWAV_INLINE drwav_bool32 drwav__fourcc_equal(const drwav_uint8* a, const char* b)
{
    return
        a[0] == b[0] &&
        a[1] == b[1] &&
        a[2] == b[2] &&
        a[3] == b[3];
}



// 比较两个 FourCC 是否相等



static DRWAV_INLINE int drwav__is_little_endian(void)
{
#if defined(DRWAV_X86) || defined(DRWAV_X64)
    return DRWAV_TRUE;



// 检查系统是否为小端序
#elif defined(__BYTE_ORDER) && defined(__LITTLE_ENDIAN) && __BYTE_ORDER == __LITTLE_ENDIAN
    // 如果是小端字节序则返回真
    return DRWAV_TRUE;
#else
    // 否则，检查字节序
    int n = 1;
    // 判断字节序是否为小端
    return (*(char*)&n) == 1;
#endif
}

// 将字节数组转换为16位无符号整数
static DRWAV_INLINE drwav_uint16 drwav__bytes_to_u16(const drwav_uint8* data)
{
    return (data[0] << 0) | (data[1] << 8);
}

// 将字节数组转换为16位有符号整数
static DRWAV_INLINE drwav_int16 drwav__bytes_to_s16(const drwav_uint8* data)
{
    return (short)drwav__bytes_to_u16(data);
}

// 将字节数组转换为32位无符号整数
static DRWAV_INLINE drwav_uint32 drwav__bytes_to_u32(const drwav_uint8* data)
{
    return (data[0] << 0) | (data[1] << 8) | (data[2] << 16) | (data[3] << 24);
}

// 将字节数组转换为32位有符号整数
static DRWAV_INLINE drwav_int32 drwav__bytes_to_s32(const drwav_uint8* data)
{
    return (drwav_int32)drwav__bytes_to_u32(data);
}

// 将字节数组转换为64位无符号整数
static DRWAV_INLINE drwav_uint64 drwav__bytes_to_u64(const drwav_uint8* data)
{
    return
        ((drwav_uint64)data[0] <<  0) | ((drwav_uint64)data[1] <<  8) | ((drwav_uint64)data[2] << 16) | ((drwav_uint64)data[3] << 24) |
        ((drwav_uint64)data[4] << 32) | ((drwav_uint64)data[5] << 40) | ((drwav_uint64)data[6] << 48) | ((drwav_uint64)data[7] << 56);
}

// 将字节数组转换为64位有符号整数
static DRWAV_INLINE drwav_int64 drwav__bytes_to_s64(const drwav_uint8* data)
{
    return (drwav_int64)drwav__bytes_to_u64(data);
}

// 将字节数组转换为GUID
static DRWAV_INLINE void drwav__bytes_to_guid(const drwav_uint8* data, drwav_uint8* guid)
{
    int i;
    // 复制字节数组到GUID
    for (i = 0; i < 16; ++i) {
        guid[i] = data[i];
    }
}

// 交换16位无符号整数的字节序
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
    // 手动交换字节序
    return ((n & 0xFF00) >> 8) |
           ((n & 0x00FF) << 8);
#endif
}

// 交换32位无符号整数的字节序
static DRWAV_INLINE drwav_uint32 drwav__bswap32(drwav_uint32 n)
{
#ifdef DRWAV_HAS_BYTESWAP32_INTRINSIC
    #if defined(_MSC_VER)
        return _byteswap_ulong(n);
    #elif defined(__GNUC__) || defined(__clang__)   # 如果是使用 GCC 或者 Clang 编译器
        #if defined(DRWAV_ARM) && (defined(__ARM_ARCH) && __ARM_ARCH >= 6) && !defined(DRWAV_64BIT)   # 如果是 ARM 架构且 ARM 版本大于等于 6，并且不是 64 位
            /* 内联汇编优化的 ARM 实现。在我的测试中，GCC 在使用 __builtin_bswap32() 时没有生成优化的代码。 */
            drwav_uint32 r;   # 定义一个无符号 32 位整数 r
            __asm__ __volatile__ (   # 内联汇编代码块开始
            #if defined(DRWAV_64BIT)
                "rev %w[out], %w[in]" : [out]"=r"(r) : [in]"r"(n)   /* <-- 这部分代码未经测试。如果社区中有人可以测试这部分代码，将不胜感激！ */
            #else
                "rev %[out], %[in]" : [out]"=r"(r) : [in]"r"(n)
            #endif
            );   # 内联汇编代码块结束
            return r;   # 返回 r
        #else
            return __builtin_bswap32(n);   # 返回使用内建函数 __builtin_bswap32() 处理后的 n
        #endif
    #else
        #error "This compiler does not support the byte swap intrinsic."   # 报错信息，表示该编译器不支持字节交换内建函数
    #endif
// 如果编译器支持字节交换64位整数的内联函数
static DRWAV_INLINE drwav_uint64 drwav__bswap64(drwav_uint64 n)
{
#ifdef DRWAV_HAS_BYTESWAP64_INTRINSIC
    // 如果是 Microsoft Visual C++ 编译器
    #if defined(_MSC_VER)
        // 使用 _byteswap_uint64 函数进行字节交换
        return _byteswap_uint64(n);
    // 如果是 GCC 或者 Clang 编译器
    #elif defined(__GNUC__) || defined(__clang__)
        // 使用 __builtin_bswap64 函数进行字节交换
        return __builtin_bswap64(n);
    // 其他编译器不支持字节交换
    #else
        #error "This compiler does not support the byte swap intrinsic."
    #endif
// 如果编译器不支持字节交换64位整数的内联函数
#else
    /* C89 不支持64位常量，所以需要使用 "<< 32" 进行位移操作，好的编译器会进行优化 */
    // 通过位运算进行字节交换
    return ((n & ((drwav_uint64)0xFF000000 << 32)) >> 56) |
           ((n & ((drwav_uint64)0x00FF0000 << 32)) >> 40) |
           ((n & ((drwav_uint64)0x0000FF00 << 32)) >> 24) |
           ((n & ((drwav_uint64)0x000000FF << 32)) >>  8) |
           ((n & ((drwav_uint64)0xFF000000      )) <<  8) |
           ((n & ((drwav_uint64)0x00FF0000      )) << 24) |
           ((n & ((drwav_uint64)0x0000FF00      )) << 40) |
           ((n & ((drwav_uint64)0x000000FF      )) << 56);
#endif
}

// 将16位有符号整数进行字节交换
static DRWAV_INLINE drwav_int16 drwav__bswap_s16(drwav_int16 n)
{
    // 调用 drwav__bswap16 函数进行字节交换，并转换为有符号整数
    return (drwav_int16)drwav__bswap16((drwav_uint16)n);
}

// 将16位有符号整数数组进行字节交换
static DRWAV_INLINE void drwav__bswap_samples_s16(drwav_int16* pSamples, drwav_uint64 sampleCount)
{
    drwav_uint64 iSample;
    // 遍历每个样本
    for (iSample = 0; iSample < sampleCount; iSample += 1) {
        // 调用 drwav__bswap_s16 函数进行字节交换
        pSamples[iSample] = drwav__bswap_s16(pSamples[iSample]);
    }
}

// 将24位无符号整数进行字节交换
static DRWAV_INLINE void drwav__bswap_s24(drwav_uint8* p)
{
    drwav_uint8 t;
    // 交换字节顺序
    t = p[0];
    p[0] = p[2];
    p[2] = t;
}

// 将24位无符号整数数组进行字节交换
static DRWAV_INLINE void drwav__bswap_samples_s24(drwav_uint8* pSamples, drwav_uint64 sampleCount)
{
    drwav_uint64 iSample;
    // 遍历每个样本
    for (iSample = 0; iSample < sampleCount; iSample += 1) {
        // 获取当前样本的指针
        drwav_uint8* pSample = pSamples + (iSample*3);
        // 调用 drwav__bswap_s24 函数进行字节交换
        drwav__bswap_s24(pSample);
    }
}
# 交换32位有符号整数的字节顺序
static DRWAV_INLINE drwav_int32 drwav__bswap_s32(drwav_int32 n)
{
    return (drwav_int32)drwav__bswap32((drwav_uint32)n);
}

# 交换32位有符号整数数组的字节顺序
static DRWAV_INLINE void drwav__bswap_samples_s32(drwav_int32* pSamples, drwav_uint64 sampleCount)
{
    drwav_uint64 iSample;
    遍历样本数组，交换每个样本的字节顺序
    for (iSample = 0; iSample < sampleCount; iSample += 1) {
        pSamples[iSample] = drwav__bswap_s32(pSamples[iSample]);
    }
}

# 交换32位浮点数的字节顺序
static DRWAV_INLINE float drwav__bswap_f32(float n)
{
    union {
        drwav_uint32 i;
        float f;
    } x;
    将浮点数转换为整数，交换字节顺序，再转换回浮点数
    x.f = n;
    x.i = drwav__bswap32(x.i);

    return x.f;
}

# 交换32位浮点数数组的字节顺序
static DRWAV_INLINE void drwav__bswap_samples_f32(float* pSamples, drwav_uint64 sampleCount)
{
    drwav_uint64 iSample;
    遍历样本数组，交换每个样本的字节顺序
    for (iSample = 0; iSample < sampleCount; iSample += 1) {
        pSamples[iSample] = drwav__bswap_f32(pSamples[iSample]);
    }
}

# 交换64位浮点数的字节顺序
static DRWAV_INLINE double drwav__bswap_f64(double n)
{
    union {
        drwav_uint64 i;
        double f;
    } x;
    将双精度浮点数转换为整数，交换字节顺序，再转换回双精度浮点数
    x.f = n;
    x.i = drwav__bswap64(x.i);

    return x.f;
}

# 交换64位浮点数数组的字节顺序
static DRWAV_INLINE void drwav__bswap_samples_f64(double* pSamples, drwav_uint64 sampleCount)
{
    drwav_uint64 iSample;
    遍历样本数组，交换每个样本的字节顺序
    for (iSample = 0; iSample < sampleCount; iSample += 1) {
        pSamples[iSample] = drwav__bswap_f64(pSamples[iSample]);
    }
}

# 交换PCM样本的字节顺序
static DRWAV_INLINE void drwav__bswap_samples_pcm(void* pSamples, drwav_uint64 sampleCount, drwav_uint32 bytesPerSample)
{
    /* Assumes integer PCM. Floating point PCM is done in drwav__bswap_samples_ieee(). */
    假设是整数PCM样本，浮点PCM样本在drwav__bswap_samples_ieee()中处理
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
            // 不支持的格式，断言失败
            /* Unsupported format. */
            DRWAV_ASSERT(DRWAV_FALSE);
        } break;
    }
static DRWAV_INLINE void drwav__bswap_samples_ieee(void* pSamples, drwav_uint64 sampleCount, drwav_uint32 bytesPerSample)
{
    // 根据字节每样本大小进行不同的处理
    switch (bytesPerSample)
    {
    #if 0   /* Contributions welcome for f16 support. */
        case 2: /* f16 */
        {
            // 如果是 2 字节大小的样本，调用相应的函数进行处理
            drwav__bswap_samples_f16((drwav_float16*)pSamples, sampleCount);
        } break;
    #endif
        case 4: /* f32 */
        {
            // 如果是 4 字节大小的样本，调用相应的函数进行处理
            drwav__bswap_samples_f32((float*)pSamples, sampleCount);
        } break;
        case 8: /* f64 */
        {
            // 如果是 8 字节大小的样本，调用相应的函数进行处理
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
    // 根据音频格式进行不同的处理
    switch (format)
    {
        case DR_WAVE_FORMAT_PCM:
        {
            // 如果是 PCM 格式，调用相应的函数进行处理
            drwav__bswap_samples_pcm(pSamples, sampleCount, bytesPerSample);
        } break;

        case DR_WAVE_FORMAT_IEEE_FLOAT:
        {
            // 如果是 IEEE_FLOAT 格式，调用相应的函数进行处理
            drwav__bswap_samples_ieee(pSamples, sampleCount, bytesPerSample);
        } break;

        case DR_WAVE_FORMAT_ALAW:
        case DR_WAVE_FORMAT_MULAW:
        {
            // 如果是 ALAW 或 MULAW 格式，调用相应的函数进行处理
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
    // 默认的内存分配函数
    (void)pUserData;
    return DRWAV_MALLOC(sz);
}

static void* drwav__realloc_default(void* p, size_t sz, void* pUserData)
{
    // 默认的内存重新分配函数
    (void)pUserData;
    return DRWAV_REALLOC(p, sz);
}

static void drwav__free_default(void* p, void* pUserData)
{
    // 默认的内存释放函数
    (void)pUserData;
    DRWAV_FREE(p);
}
static void* drwav__malloc_from_callbacks(size_t sz, const drwav_allocation_callbacks* pAllocationCallbacks)
{
    // 如果分配回调为空，则返回空指针
    if (pAllocationCallbacks == NULL) {
        return NULL;
    }

    // 如果有分配内存的回调函数，则调用该函数
    if (pAllocationCallbacks->onMalloc != NULL) {
        return pAllocationCallbacks->onMalloc(sz, pAllocationCallbacks->pUserData);
    }

    /* 尝试使用 realloc()。*/
    // 如果有重新分配内存的回调函数，则调用该函数
    if (pAllocationCallbacks->onRealloc != NULL) {
        return pAllocationCallbacks->onRealloc(NULL, sz, pAllocationCallbacks->pUserData);
    }

    return NULL;
}

static void* drwav__realloc_from_callbacks(void* p, size_t szNew, size_t szOld, const drwav_allocation_callbacks* pAllocationCallbacks)
{
    // 如果分配回调为空，则返回空指针
    if (pAllocationCallbacks == NULL) {
        return NULL;
    }

    // 如果有重新分配内存的回调函数，则调用该函数
    if (pAllocationCallbacks->onRealloc != NULL) {
        return pAllocationCallbacks->onRealloc(p, szNew, pAllocationCallbacks->pUserData);
    }

    /* 尝试模拟 realloc() 使用 malloc()/free()。*/
    // 如果有分配内存和释放内存的回调函数，则尝试模拟 realloc() 的行为
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

    return NULL;
}

static void drwav__free_from_callbacks(void* p, const drwav_allocation_callbacks* pAllocationCallbacks)
{
    // 如果指针为空或分配回调为空，则直接返回
    if (p == NULL || pAllocationCallbacks == NULL) {
        return;
    }

    // 如果有释放内存的回调函数，则调用该函数
    if (pAllocationCallbacks->onFree != NULL) {
        pAllocationCallbacks->onFree(p, pAllocationCallbacks->pUserData);
    }
}


static drwav_allocation_callbacks drwav_copy_allocation_callbacks_or_defaults(const drwav_allocation_callbacks* pAllocationCallbacks)
{
    // 如果分配回调不为空，则复制回调函数
    if (pAllocationCallbacks != NULL) {
        /* 复制。*/
        return *pAllocationCallbacks;
    } else {
        /* 如果条件不成立，使用默认值。 */
        // 创建一个包含默认值的内存分配回调结构体
        drwav_allocation_callbacks allocationCallbacks;
        // 设置用户数据为空指针
        allocationCallbacks.pUserData = NULL;
        // 设置默认的内存分配函数
        allocationCallbacks.onMalloc  = drwav__malloc_default;
        // 设置默认的内存重新分配函数
        allocationCallbacks.onRealloc = drwav__realloc_default;
        // 设置默认的内存释放函数
        allocationCallbacks.onFree    = drwav__free_default;
        // 返回设置好的内存分配回调结构体
        return allocationCallbacks;
    }
# 检查给定的格式标签是否为压缩格式标签
static DRWAV_INLINE drwav_bool32 drwav__is_compressed_format_tag(drwav_uint16 formatTag)
{
    return
        formatTag == DR_WAVE_FORMAT_ADPCM ||
        formatTag == DR_WAVE_FORMAT_DVI_ADPCM;
}

# 计算 RIFF 格式的块的填充大小
static unsigned int drwav__chunk_padding_size_riff(drwav_uint64 chunkSize)
{
    return (unsigned int)(chunkSize % 2);
}

# 计算 W64 格式的块的填充大小
static unsigned int drwav__chunk_padding_size_w64(drwav_uint64 chunkSize)
{
    return (unsigned int)(chunkSize % 8);
}

# 读取 MSADPCM 格式的 PCM 帧
static drwav_uint64 drwav_read_pcm_frames_s16__msadpcm(drwav* pWav, drwav_uint64 samplesToRead, drwav_int16* pBufferOut);

# 读取 IMA 格式的 PCM 帧
static drwav_uint64 drwav_read_pcm_frames_s16__ima(drwav* pWav, drwav_uint64 samplesToRead, drwav_int16* pBufferOut);

# 内部函数，初始化写入 WAV 文件
static drwav_bool32 drwav_init_write__internal(drwav* pWav, const drwav_data_format* pFormat, drwav_uint64 totalSampleCount);

# 读取块头信息
static drwav_result drwav__read_chunk_header(drwav_read_proc onRead, void* pUserData, drwav_container container, drwav_uint64* pRunningBytesReadOut, drwav_chunk_header* pHeaderOut)
{
    # 如果容器类型为 RIFF 或 RF64
    if (container == drwav_container_riff || container == drwav_container_rf64) {
        # 读取四个字节的块标识符
        drwav_uint8 sizeInBytes[4];
        if (onRead(pUserData, pHeaderOut->id.fourcc, 4) != 4) {
            return DRWAV_AT_END;
        }
        
        # 读取四个字节的块大小
        if (onRead(pUserData, sizeInBytes, 4) != 4) {
            return DRWAV_INVALID_FILE;
        }
        
        # 将字节数组转换为无符号整数
        pHeaderOut->sizeInBytes = drwav__bytes_to_u32(sizeInBytes);
        # 计算块的填充大小
        pHeaderOut->paddingSize = drwav__chunk_padding_size_riff(pHeaderOut->sizeInBytes);
        # 更新已读取的字节数
        *pRunningBytesReadOut += 8;
    } else {
        // 读取 8 个字节的数据到 sizeInBytes 数组中
        drwav_uint8 sizeInBytes[8];

        // 读取 16 个字节的数据到 pHeaderOut->id.guid 中，如果读取失败返回 DRWAV_AT_END
        if (onRead(pUserData, pHeaderOut->id.guid, 16) != 16) {
            return DRWAV_AT_END;
        }

        // 读取 8 个字节的数据到 sizeInBytes 数组中，如果读取失败返回 DRWAV_INVALID_FILE
        if (onRead(pUserData, sizeInBytes, 8) != 8) {
            return DRWAV_INVALID_FILE;
        }

        // 计算数据大小，减去 24 是因为 w64 文件包含头部的大小
        pHeaderOut->sizeInBytes = drwav__bytes_to_u64(sizeInBytes) - 24;    /* <-- Subtract 24 because w64 includes the size of the header. */
        // 计算填充大小
        pHeaderOut->paddingSize = drwav__chunk_padding_size_w64(pHeaderOut->sizeInBytes);
        // 更新已读取的字节数
        *pRunningBytesReadOut += 24;
    }

    // 返回成功
    return DRWAV_SUCCESS;
# 向前定位函数，用于在数据流中向前查找指定偏移量的位置
static drwav_bool32 drwav__seek_forward(drwav_seek_proc onSeek, drwav_uint64 offset, void* pUserData)
{
    # 初始化剩余需要查找的字节数
    drwav_uint64 bytesRemainingToSeek = offset;
    # 循环直到所有需要查找的字节数都被处理完
    while (bytesRemainingToSeek > 0) {
        # 如果剩余需要查找的字节数大于 0x7FFFFFFF
        if (bytesRemainingToSeek > 0x7FFFFFFF) {
            # 调用 onSeek 函数向前查找 0x7FFFFFFF 字节
            if (!onSeek(pUserData, 0x7FFFFFFF, drwav_seek_origin_current)) {
                return DRWAV_FALSE;
            }
            # 更新剩余需要查找的字节数
            bytesRemainingToSeek -= 0x7FFFFFFF;
        } else {
            # 如果剩余需要查找的字节数小于等于 0x7FFFFFFF
            # 调用 onSeek 函数向前查找剩余的字节数
            if (!onSeek(pUserData, (int)bytesRemainingToSeek, drwav_seek_origin_current)) {
                return DRWAV_FALSE;
            }
            # 将剩余需要查找的字节数置为 0，结束查找
            bytesRemainingToSeek = 0;
        }
    }

    return DRWAV_TRUE;
}

# 从文件开始位置向前查找函数
static drwav_bool32 drwav__seek_from_start(drwav_seek_proc onSeek, drwav_uint64 offset, void* pUserData)
{
    # 如果偏移量小于等于 0x7FFFFFFF
    if (offset <= 0x7FFFFFFF) {
        # 调用 onSeek 函数从文件开始位置向前查找指定偏移量
        return onSeek(pUserData, (int)offset, drwav_seek_origin_start);
    }

    # 如果偏移量大于 0x7FFFFFFF
    # 向前查找 0x7FFFFFFF 字节
    if (!onSeek(pUserData, 0x7FFFFFFF, drwav_seek_origin_start)) {
        return DRWAV_FALSE;
    }
    # 更新偏移量
    offset -= 0x7FFFFFFF;

    # 循环直到偏移量为 0
    for (;;) {
        # 如果偏移量小于等于 0x7FFFFFFF
        if (offset <= 0x7FFFFFFF) {
            # 调用 onSeek 函数从当前位置向前查找剩余的偏移量
            return onSeek(pUserData, (int)offset, drwav_seek_origin_current);
        }

        # 如果偏移量大于 0x7FFFFFFF
        # 向前查找 0x7FFFFFFF 字节
        if (!onSeek(pUserData, 0x7FFFFFFF, drwav_seek_origin_current)) {
            return DRWAV_FALSE;
        }
        # 更新偏移量
        offset -= 0x7FFFFFFF;
    }

    # 不应该执行到这里
    # return DRWAV_TRUE;
}

# 读取 fmt 块函数
static drwav_bool32 drwav__read_fmt(drwav_read_proc onRead, drwav_seek_proc onSeek, void* pUserData, drwav_container container, drwav_uint64* pRunningBytesReadOut, drwav_fmt* fmtOut)
{
    # 定义块头和 fmt 数据
    drwav_chunk_header header;
    drwav_uint8 fmt[16];

    # 读取块头信息
    if (drwav__read_chunk_header(onRead, pUserData, container, pRunningBytesReadOut, &header) != DRWAV_SUCCESS) {
        return DRWAV_FALSE;
    }

    # 跳过非 fmt 块
    # 当容器为 RIFF 或 RF64 且不是 "fmt " 格式时，或者容器为 W64 且不是 W64_FMT 格式时，循环执行以下代码块
    while (((container == drwav_container_riff || container == drwav_container_rf64) && !drwav__fourcc_equal(header.id.fourcc, "fmt ")) || (container == drwav_container_w64 && !drwav__guid_equal(header.id.guid, drwavGUID_W64_FMT))) {
        # 如果无法向前跳过指定大小的数据块，返回 FALSE
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

    /* 验证数据块的格式 */
    if (container == drwav_container_riff || container == drwav_container_rf64) {
        # 如果容器为 RIFF 或 RF64，但不是 "fmt " 格式，返回 FALSE
        if (!drwav__fourcc_equal(header.id.fourcc, "fmt ")) {
            return DRWAV_FALSE;
        }
    } else {
        # 如果容器不是 RIFF 或 RF64，但不是 W64_FMT 格式，返回 FALSE
        if (!drwav__guid_equal(header.id.guid, drwavGUID_W64_FMT)) {
            return DRWAV_FALSE;
        }
    }

    # 读取 fmt 数据块内容
    if (onRead(pUserData, fmt, sizeof(fmt)) != sizeof(fmt)) {
        return DRWAV_FALSE;
    }
    # 更新已读取的字节数
    *pRunningBytesReadOut += sizeof(fmt);

    # 解析 fmt 数据块内容并填充到 fmtOut 结构体中
    fmtOut->formatTag      = drwav__bytes_to_u16(fmt + 0);
    fmtOut->channels       = drwav__bytes_to_u16(fmt + 2);
    fmtOut->sampleRate     = drwav__bytes_to_u32(fmt + 4);
    fmtOut->avgBytesPerSec = drwav__bytes_to_u32(fmt + 8);
    fmtOut->blockAlign     = drwav__bytes_to_u16(fmt + 12);
    fmtOut->bitsPerSample  = drwav__bytes_to_u16(fmt + 14);

    # 初始化 fmtOut 结构体的其他字段
    fmtOut->extendedSize       = 0;
    fmtOut->validBitsPerSample = 0;
    fmtOut->channelMask        = 0;
    memset(fmtOut->subFormat, 0, sizeof(fmtOut->subFormat));
    # 如果头部大小大于16个字节
    if (header.sizeInBytes > 16) {
        # 创建一个用于存储 fmt_cbSize 的数组
        drwav_uint8 fmt_cbSize[2];
        # 记录已读取的字节数
        int bytesReadSoFar = 0;

        # 从输入流中读取 fmt_cbSize 的数据
        if (onRead(pUserData, fmt_cbSize, sizeof(fmt_cbSize)) != sizeof(fmt_cbSize)) {
            return DRWAV_FALSE;    /* 期望更多数据 */
        }
        *pRunningBytesReadOut += sizeof(fmt_cbSize);

        # 更新已读取的字节数
        bytesReadSoFar = 18;

        # 将 fmtOut 的 extendedSize 设置为 fmt_cbSize 转换后的值
        fmtOut->extendedSize = drwav__bytes_to_u16(fmt_cbSize);
        # 如果 extendedSize 大于0
        if (fmtOut->extendedSize > 0) {
            /* 简单验证 */
            # 如果 formatTag 为 DR_WAVE_FORMAT_EXTENSIBLE
            if (fmtOut->formatTag == DR_WAVE_FORMAT_EXTENSIBLE) {
                # 如果 extendedSize 不等于22，则返回错误
                if (fmtOut->extendedSize != 22) {
                    return DRWAV_FALSE;
                }
            }

            # 如果 formatTag 为 DR_WAVE_FORMAT_EXTENSIBLE
            if (fmtOut->formatTag == DR_WAVE_FORMAT_EXTENSIBLE) {
                # 创建一个用于存储 fmtext 的数组
                drwav_uint8 fmtext[22];
                # 从输入流中读取 fmtext 的数据
                if (onRead(pUserData, fmtext, fmtOut->extendedSize) != fmtOut->extendedSize) {
                    return DRWAV_FALSE;    /* 期望更多数据 */
                }

                # 更新 fmtOut 的 validBitsPerSample、channelMask 和 subFormat
                fmtOut->validBitsPerSample = drwav__bytes_to_u16(fmtext + 0);
                fmtOut->channelMask        = drwav__bytes_to_u32(fmtext + 2);
                drwav__bytes_to_guid(fmtext + 6, fmtOut->subFormat);
            } else {
                # 如果 formatTag 不是 DR_WAVE_FORMAT_EXTENSIBLE，则跳过 extendedSize 字节
                if (!onSeek(pUserData, fmtOut->extendedSize, drwav_seek_origin_current)) {
                    return DRWAV_FALSE;
                }
            }
            *pRunningBytesReadOut += fmtOut->extendedSize;

            # 更新已读取的字节数
            bytesReadSoFar += fmtOut->extendedSize;
        }

        # 跳过剩余的字节。对于 w64，剩余的字节将根据块大小定义
        if (!onSeek(pUserData, (int)(header.sizeInBytes - bytesReadSoFar), drwav_seek_origin_current)) {
            return DRWAV_FALSE;
        }
        *pRunningBytesReadOut += (header.sizeInBytes - bytesReadSoFar);
    }
    # 如果头部有填充数据
    if (header.paddingSize > 0) {
        # 如果无法移动指针到填充数据的位置，返回假
        if (!onSeek(pUserData, header.paddingSize, drwav_seek_origin_current)) {
            return DRWAV_FALSE;
        }
        # 更新已读取的字节数
        *pRunningBytesReadOut += header.paddingSize;
    }

    # 返回真
    return DRWAV_TRUE;
}

// 读取回调函数，用于从数据源读取数据
static size_t drwav__on_read(drwav_read_proc onRead, void* pUserData, void* pBufferOut, size_t bytesToRead, drwav_uint64* pCursor)
{
    size_t bytesRead;

    // 确保读取回调函数和游标指针不为空
    DRWAV_ASSERT(onRead != NULL);
    DRWAV_ASSERT(pCursor != NULL);

    // 调用读取回调函数读取数据，并更新游标指针
    bytesRead = onRead(pUserData, pBufferOut, bytesToRead);
    *pCursor += bytesRead;
    return bytesRead;
}

#if 0
// 寻址回调函数，用于在数据源中定位到指定位置
static drwav_bool32 drwav__on_seek(drwav_seek_proc onSeek, void* pUserData, int offset, drwav_seek_origin origin, drwav_uint64* pCursor)
{
    DRWAV_ASSERT(onSeek != NULL);
    DRWAV_ASSERT(pCursor != NULL);

    // 调用寻址回调函数，根据返回值判断是否成功，并更新游标指针
    if (!onSeek(pUserData, offset, origin)) {
        return DRWAV_FALSE;
    }

    if (origin == drwav_seek_origin_start) {
        *pCursor = offset;
    } else {
        *pCursor += offset;
    }

    return DRWAV_TRUE;
}
#endif

// 获取每个 PCM 帧的字节数
static drwav_uint32 drwav_get_bytes_per_pcm_frame(drwav* pWav)
{
    /*
    The bytes per frame is a bit ambiguous. It can be either be based on the bits per sample, or the block align. The way I'm doing it here
    is that if the bits per sample is a multiple of 8, use floor(bitsPerSample*channels/8), otherwise fall back to the block align.
    */
    if ((pWav->bitsPerSample & 0x7) == 0) {
        /* Bits per sample is a multiple of 8. */
        return (pWav->bitsPerSample * pWav->fmt.channels) >> 3;
    } else {
        return pWav->fmt.blockAlign;
    }
}

// 获取格式标签
DRWAV_API drwav_uint16 drwav_fmt_get_format(const drwav_fmt* pFMT)
{
    if (pFMT == NULL) {
        return 0;
    }

    // 如果格式标签不是 DR_WAVE_FORMAT_EXTENSIBLE，则直接返回格式标签
    if (pFMT->formatTag != DR_WAVE_FORMAT_EXTENSIBLE) {
        return pFMT->formatTag;
    } else {
        return drwav__bytes_to_u16(pFMT->subFormat);    /* Only the first two bytes are required. */
    }
}

// 预初始化函数
static drwav_bool32 drwav_preinit(drwav* pWav, drwav_read_proc onRead, drwav_seek_proc onSeek, void* pReadSeekUserData, const drwav_allocation_callbacks* pAllocationCallbacks)
{
    // 确保 WAV 结构体和读取、寻址回调函数不为空
    if (pWav == NULL || onRead == NULL || onSeek == NULL) {
        return DRWAV_FALSE;
    }
    // 将 pWav 指向的内存清零，大小为 pWav 所指向对象的大小
    DRWAV_ZERO_MEMORY(pWav, sizeof(*pWav));
    // 设置 pWav 的 onRead 回调函数为 onRead
    pWav->onRead    = onRead;
    // 设置 pWav 的 onSeek 回调函数为 onSeek
    pWav->onSeek    = onSeek;
    // 设置 pWav 的 pUserData 为 pReadSeekUserData
    pWav->pUserData = pReadSeekUserData;
    // 复制 pAllocationCallbacks 的内存分配回调函数到 pWav 的 allocationCallbacks
    pWav->allocationCallbacks = drwav_copy_allocation_callbacks_or_defaults(pAllocationCallbacks);

    // 如果 pWav 的 allocationCallbacks 中的 onFree 为 NULL，或者 onMalloc 和 onRealloc 都为 NULL，则返回 DRWAV_FALSE
    if (pWav->allocationCallbacks.onFree == NULL || (pWav->allocationCallbacks.onMalloc == NULL && pWav->allocationCallbacks.onRealloc == NULL)) {
        return DRWAV_FALSE;    /* Invalid allocation callbacks. */
    }

    // 返回 DRWAV_TRUE
    return DRWAV_TRUE;
}

static drwav_bool32 drwav_init__internal(drwav* pWav, drwav_chunk_proc onChunk, void* pChunkUserData, drwav_uint32 flags)
{
    /* This function assumes drwav_preinit() has been called beforehand. */

    drwav_uint64 cursor;    /* <-- Keeps track of the byte position so we can seek to specific locations. */
    drwav_bool32 sequential;
    drwav_uint8 riff[4];
    drwav_fmt fmt;
    unsigned short translatedFormatTag;
    drwav_bool32 foundDataChunk;
    drwav_uint64 dataChunkSize = 0; /* <-- Important! Don't explicitly set this to 0 anywhere else. Calculation of the size of the data chunk is performed in different paths depending on the container. */
    drwav_uint64 sampleCountFromFactChunk = 0;  /* Same as dataChunkSize - make sure this is the only place this is initialized to 0. */
    drwav_uint64 chunkSize;

    cursor = 0;
    sequential = (flags & DRWAV_SEQUENTIAL) != 0;

    /* The first 4 bytes should be the RIFF identifier. */
    if (drwav__on_read(pWav->onRead, pWav->pUserData, riff, sizeof(riff), &cursor) != sizeof(riff)) {
        return DRWAV_FALSE;
    }

    /*
    The first 4 bytes can be used to identify the container. For RIFF files it will start with "RIFF" and for
    w64 it will start with "riff".
    */
    if (drwav__fourcc_equal(riff, "RIFF")) {
        pWav->container = drwav_container_riff;
    } else if (drwav__fourcc_equal(riff, "riff")) {
        int i;
        drwav_uint8 riff2[12];

        pWav->container = drwav_container_w64;

        /* Check the rest of the GUID for validity. */
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
    }
    } else {
        return DRWAV_FALSE;   /* 未知或不支持的容器类型。 */
    }


    if (pWav->container == drwav_container_riff || pWav->container == drwav_container_rf64) {
        drwav_uint8 chunkSizeBytes[4];
        drwav_uint8 wave[4];

        /* RIFF/WAVE */
        // 读取块大小信息
        if (drwav__on_read(pWav->onRead, pWav->pUserData, chunkSizeBytes, sizeof(chunkSizeBytes), &cursor) != sizeof(chunkSizeBytes)) {
            return DRWAV_FALSE;
        }

        if (pWav->container == drwav_container_riff) {
            if (drwav__bytes_to_u32(chunkSizeBytes) < 36) {
                return DRWAV_FALSE;    /* 块大小应至少为36字节。 */
            }
        } else {
            if (drwav__bytes_to_u32(chunkSizeBytes) != 0xFFFFFFFF) {
                return DRWAV_FALSE;    /* 对于RF64，块大小应设置为-1/0xFFFFFFFF。实际大小稍后获取。 */
            }
        }

        // 读取"WAVE"标识
        if (drwav__on_read(pWav->onRead, pWav->pUserData, wave, sizeof(wave), &cursor) != sizeof(wave)) {
            return DRWAV_FALSE;
        }

        if (!drwav__fourcc_equal(wave, "WAVE")) {
            return DRWAV_FALSE;    /* 期望"WAVE"标识。 */
        }
    } else {
        drwav_uint8 chunkSizeBytes[8];
        drwav_uint8 wave[16];

        /* W64 */
        // 读取块大小信息
        if (drwav__on_read(pWav->onRead, pWav->pUserData, chunkSizeBytes, sizeof(chunkSizeBytes), &cursor) != sizeof(chunkSizeBytes)) {
            return DRWAV_FALSE;
        }

        if (drwav__bytes_to_u64(chunkSizeBytes) < 80) {
            return DRWAV_FALSE;
        }

        // 读取"W64"标识
        if (drwav__on_read(pWav->onRead, pWav->pUserData, wave, sizeof(wave), &cursor) != sizeof(wave)) {
            return DRWAV_FALSE;
        }

        if (!drwav__guid_equal(wave, drwavGUID_W64_WAVE)) {
            return DRWAV_FALSE;
        }
    }


    /* 对于RF64，"ds64"块必须在"fmt "块之前出现。 */
    // 如果 WAV 文件的容器类型是 rf64
    if (pWav->container == drwav_container_rf64) {
        // 读取 8 字节大小的数据
        drwav_uint8 sizeBytes[8];
        // 记录当前 chunk 剩余的字节数
        drwav_uint64 bytesRemainingInChunk;
        // 定义 chunk 头部
        drwav_chunk_header header;
        // 读取 chunk 头部信息
        drwav_result result = drwav__read_chunk_header(pWav->onRead, pWav->pUserData, pWav->container, &cursor, &header);
        // 如果读取失败，返回 FALSE
        if (result != DRWAV_SUCCESS) {
            return DRWAV_FALSE;
        }

        // 检查 chunk 的 ID 是否为 "ds64"
        if (!drwav__fourcc_equal(header.id.fourcc, "ds64")) {
            return DRWAV_FALSE; /* Expecting "ds64". */
        }

        // 计算 chunk 剩余的字节数
        bytesRemainingInChunk = header.sizeInBytes + header.paddingSize;

        // 跳过 RIFF chunk 的大小
        if (!drwav__seek_forward(pWav->onSeek, 8, pWav->pUserData)) {
            return DRWAV_FALSE;
        }
        bytesRemainingInChunk -= 8;
        cursor += 8;

        // 读取下一个 8 字节作为 "data" chunk 的大小
        if (drwav__on_read(pWav->onRead, pWav->pUserData, sizeBytes, sizeof(sizeBytes), &cursor) != sizeof(sizeBytes)) {
            return DRWAV_FALSE;
        }
        bytesRemainingInChunk -= 8;
        dataChunkSize = drwav__bytes_to_u64(sizeBytes);

        // 读取下一个 8 字节作为从 FACT chunk 获取的采样数
        if (drwav__on_read(pWav->onRead, pWav->pUserData, sizeBytes, sizeof(sizeBytes), &cursor) != sizeof(sizeBytes)) {
            return DRWAV_FALSE;
        }
        bytesRemainingInChunk -= 8;
        sampleCountFromFactChunk = drwav__bytes_to_u64(sizeBytes);

        // 跳过剩余的 chunk 数据
        if (!drwav__seek_forward(pWav->onSeek, bytesRemainingInChunk, pWav->pUserData)) {
            return DRWAV_FALSE;
        }
        cursor += bytesRemainingInChunk;
    }

    // 读取下一个 chunk 作为 "fmt " chunk
    if (!drwav__read_fmt(pWav->onRead, pWav->onSeek, pWav->pUserData, pWav->container, &cursor, &fmt)) {
        return DRWAV_FALSE;    /* Failed to read the "fmt " chunk. */
    }
    /* 基本验证。*/
    如果采样率为0或大于最大采样率，通道数为0或大于最大通道数，每个样本的位数为0或大于最大位数，块对齐为0，则返回假，可能是无效的 WAV 文件。

    /* 转换内部格式。*/
    将内部格式标签翻译为 translatedFormatTag。
    如果翻译后的格式标签为 DR_WAVE_FORMAT_EXTENSIBLE，则从 fmt.subFormat 中读取两个字节转换为无符号16位整数。

    /*
    我们需要枚举每个块的原因有两个：
      1) "data" 块可能不是下一个块
      2) 我们可能需要将每个块报告给客户端
      
    为了正确将每个块报告给客户端，我们需要循环直到文件结束。
    */
    初始化 foundDataChunk 为假。

    /* 下一个我们关心的块是 "data" 块。这不一定是下一个块，所以我们需要循环。*/
    for (;;)
    }

    /* 如果没有找到数据块，则返回错误。*/
    如果没有找到数据块，则返回假。

    /* 可能已经移动到数据块之后。如果是这样，我们需要回退。如果在顺序模式下运行，可以假设已经位于数据块上。*/
    如果不是顺序模式：
        如果无法从文件开头偏移 pWav->dataChunkDataPos 个字节，则返回假。
        将游标设置为 pWav->dataChunkDataPos。

    /* 此时应该位于原始音频数据的第一个字节。*/

    将 pWav->fmt 设置为 fmt。
    将 pWav->sampleRate 设置为 fmt.sampleRate。
    将 pWav->channels 设置为 fmt.channels。
    将 pWav->bitsPerSample 设置为 fmt.bitsPerSample。
    将 pWav->bytesRemaining 设置为 dataChunkSize。
    将 pWav->translatedFormatTag 设置为 translatedFormatTag。
    # 设置 WAV 文件数据块大小
    pWav->dataChunkDataSize   = dataChunkSize;

    # 如果从 FACT 块中获取到采样数不为0，则使用该值作为总采样帧数
    if (sampleCountFromFactChunk != 0) {
        pWav->totalPCMFrameCount = sampleCountFromFactChunk;
    } else {
        # 否则根据数据块大小和每个 PCM 帧的字节数计算总采样帧数
        pWav->totalPCMFrameCount = dataChunkSize / drwav_get_bytes_per_pcm_frame(pWav);

        # 如果是 ADPCM 格式
        if (pWav->translatedFormatTag == DR_WAVE_FORMAT_ADPCM) {
            drwav_uint64 totalBlockHeaderSizeInBytes;
            drwav_uint64 blockCount = dataChunkSize / fmt.blockAlign;

            # 确保考虑到任何尾部不完整的块
            if ((blockCount * fmt.blockAlign) < dataChunkSize) {
                blockCount += 1;
            }

            # 每字节解码两个样本。数据块中会有 blockCount 个头部信息，足够计算总采样帧数
            totalBlockHeaderSizeInBytes = blockCount * (6*fmt.channels);
            pWav->totalPCMFrameCount = ((dataChunkSize - totalBlockHeaderSizeInBytes) * 2) / fmt.channels;
        }
        # 如果是 DVI ADPCM 格式
        if (pWav->translatedFormatTag == DR_WAVE_FORMAT_DVI_ADPCM) {
            drwav_uint64 totalBlockHeaderSizeInBytes;
            drwav_uint64 blockCount = dataChunkSize / fmt.blockAlign;

            # 确保考虑到任何尾部不完整的块
            if ((blockCount * fmt.blockAlign) < dataChunkSize) {
                blockCount += 1;
            }

            # 每字节解码两个样本。数据块中会有 blockCount 个头部信息，足够计算总采样帧数
            totalBlockHeaderSizeInBytes = blockCount * (4*fmt.channels);
            pWav->totalPCMFrameCount = ((dataChunkSize - totalBlockHeaderSizeInBytes) * 2) / fmt.channels;

            # 头部包括每个通道的解码样本，作为初始预测样本
            pWav->totalPCMFrameCount += blockCount;
        }
    }

    # 一些格式只支持特定数量的通道
    # 如果音频文件的格式为 ADPCM 或 DVI_ADPCM
    if (pWav->translatedFormatTag == DR_WAVE_FORMAT_ADPCM || pWav->translatedFormatTag == DR_WAVE_FORMAT_DVI_ADPCM) {
        # 如果音频文件的通道数大于2
        if (pWav->channels > 2) {
            # 返回假值
            return DRWAV_FALSE;
        }
    }
#ifdef DR_WAV_LIBSNDFILE_COMPAT
    /*
    如果定义了 DR_WAV_LIBSNDFILE_COMPAT 宏，则执行以下代码块，用于模拟 libsndfile 的逻辑以便正确运行测试。
    在我使用的 libsndfile 版本中（来自 libsndfile 网站的 Windows 安装程序），似乎 libsndfile 用于 MS-ADPCM 的总样本计数是不正确的。
    他们似乎是根据块数计算总样本计数，然而这会导致在最后一个块的末尾包含额外的静音样本。
    知道总样本计数的正确方法是检查“fact”块，对于压缩格式应该始终存在，并且应该始终包含样本计数。
    下面这段代码仅用于模拟 libsndfile 逻辑，以便我可以正确地运行我的正确性测试，且默认情况下是禁用的。
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

    // 初始化成功，返回 DRWAV_TRUE
    return DRWAV_TRUE;
}

// 初始化 WAV 文件读取器
DRWAV_API drwav_bool32 drwav_init(drwav* pWav, drwav_read_proc onRead, drwav_seek_proc onSeek, void* pUserData, const drwav_allocation_callbacks* pAllocationCallbacks)
{
    // 调用 drwav_init_ex 函数，传入参数并返回结果
    return drwav_init_ex(pWav, onRead, onSeek, NULL, pUserData, NULL, 0, pAllocationCallbacks);
}

// 初始化 WAV 文件读取器（扩展版）
DRWAV_API drwav_bool32 drwav_init_ex(drwav* pWav, drwav_read_proc onRead, drwav_seek_proc onSeek, drwav_chunk_proc onChunk, void* pReadSeekUserData, void* pChunkUserData, drwav_uint32 flags, const drwav_allocation_callbacks* pAllocationCallbacks)
{
    # 如果 drwav_preinit 函数返回 false，则返回 DRWAV_FALSE
    if (!drwav_preinit(pWav, onRead, onSeek, pReadSeekUserData, pAllocationCallbacks)) {
        return DRWAV_FALSE;
    }

    # 调用 drwav_init__internal 函数初始化 WAV 文件，返回初始化结果
    return drwav_init__internal(pWav, onChunk, pChunkUserData, flags);
}



static drwav_uint32 drwav__riff_chunk_size_riff(drwav_uint64 dataChunkSize)
{
    // 计算 RIFF 格式的 chunk 大小，包括 "WAVE" 头部、"fmt " chunk 大小、数据 chunk 大小和填充大小
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
    // 如果数据 chunk 大小不超过 0xFFFFFFFFUL，则直接返回数据 chunk 大小，否则返回 0xFFFFFFFFUL
    if (dataChunkSize <= 0xFFFFFFFFUL) {
        return (drwav_uint32)dataChunkSize;
    } else {
        return 0xFFFFFFFFUL;
    }
}

static drwav_uint64 drwav__riff_chunk_size_w64(drwav_uint64 dataChunkSize)
{
    // 计算 W64 格式的 chunk 大小，包括 "W64" 头部、"fmt " chunk 大小、数据 chunk 大小和填充大小
    drwav_uint64 dataSubchunkPaddingSize = drwav__chunk_padding_size_w64(dataChunkSize);

    // 返回 chunk 大小，包括 "W64" 头部、"fmt " chunk 大小、数据 chunk 大小和填充大小
    return 80 + 24 + dataChunkSize + dataSubchunkPaddingSize;   /* +24 because W64 includes the size of the GUID and size fields. */
}

static drwav_uint64 drwav__data_chunk_size_w64(drwav_uint64 dataChunkSize)
{
    // 返回数据 chunk 大小，包括 "W64" 头部、"fmt " chunk 大小和数据 chunk 大小
    return 24 + dataChunkSize;        /* +24 because W64 includes the size of the GUID and size fields. */
}

static drwav_uint64 drwav__riff_chunk_size_rf64(drwav_uint64 dataChunkSize)
{
    // 计算 RF64 格式的 chunk 大小，包括 "WAVE" 头部、"ds64" chunk 大小、"fmt " chunk 大小、数据 chunk 大小和填充大小
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
    // 返回数据 chunk 大小
    return dataChunkSize;
}

static size_t drwav__write(drwav* pWav, const void* pData, size_t dataSize)
{
    // 断言 WAV 结构体和写入回调函数不为空
    DRWAV_ASSERT(pWav          != NULL);
    DRWAV_ASSERT(pWav->onWrite != NULL);

    // 通用写入函数，假设不需要字节重新排序
    return pWav->onWrite(pWav->pUserData, pData, dataSize);
}

static size_t drwav__write_u16ne_to_le(drwav* pWav, drwav_uint16 value)
{
    // 断言 WAV 结构体和写入回调函数不为空
    DRWAV_ASSERT(pWav          != NULL);
    DRWAV_ASSERT(pWav->onWrite != NULL);
    # 如果系统不是小端字节序，则将数值进行字节交换
    if (!drwav__is_little_endian()) {
        value = drwav__bswap16(value);
    }

    # 将数值写入 WAV 文件，写入长度为 2 字节
    return drwav__write(pWav, &value, 2);
static size_t drwav__write_u32ne_to_le(drwav* pWav, drwav_uint32 value)
{
    // 确保 WAV 对象和写入回调函数不为空
    DRWAV_ASSERT(pWav          != NULL);
    DRWAV_ASSERT(pWav->onWrite != NULL);

    // 如果不是小端字节序，将数值转换为小端字节序
    if (!drwav__is_little_endian()) {
        value = drwav__bswap32(value);
    }

    // 调用写入函数，写入4字节数据
    return drwav__write(pWav, &value, 4);
}

static size_t drwav__write_u64ne_to_le(drwav* pWav, drwav_uint64 value)
{
    // 确保 WAV 对象和写入回调函数不为空
    DRWAV_ASSERT(pWav          != NULL);
    DRWAV_ASSERT(pWav->onWrite != NULL);

    // 如果不是小端字节序，将数值转换为小端字节序
    if (!drwav__is_little_endian()) {
        value = drwav__bswap64(value);
    }

    // 调用写入函数，写入8字节数据
    return drwav__write(pWav, &value, 8);
}

static drwav_bool32 drwav_preinit_write(drwav* pWav, const drwav_data_format* pFormat, drwav_bool32 isSequential, drwav_write_proc onWrite, drwav_seek_proc onSeek, void* pUserData, const drwav_allocation_callbacks* pAllocationCallbacks)
{
    // 如果 WAV 对象或写入回调函数为空，返回错误
    if (pWav == NULL || onWrite == NULL) {
        return DRWAV_FALSE;
    }

    // 如果不是顺序写入模式且寻址回调函数为空，返回错误
    if (!isSequential && onSeek == NULL) {
        return DRWAV_FALSE; /* <-- onSeek is required when in non-sequential mode. */
    }

    // 不支持压缩格式，需要在启用之前添加对 "fact" 块的支持
    if (pFormat->format == DR_WAVE_FORMAT_EXTENSIBLE) {
        return DRWAV_FALSE;
    }
    if (pFormat->format == DR_WAVE_FORMAT_ADPCM || pFormat->format == DR_WAVE_FORMAT_DVI_ADPCM) {
        return DRWAV_FALSE;
    }

    // 初始化 WAV 对象
    DRWAV_ZERO_MEMORY(pWav, sizeof(*pWav));
    pWav->onWrite   = onWrite;
    pWav->onSeek    = onSeek;
    pWav->pUserData = pUserData;
    pWav->allocationCallbacks = drwav_copy_allocation_callbacks_or_defaults(pAllocationCallbacks);

    // 检查分配回调函数是否有效
    if (pWav->allocationCallbacks.onFree == NULL || (pWav->allocationCallbacks.onMalloc == NULL && pWav->allocationCallbacks.onRealloc == NULL)) {
        return DRWAV_FALSE;    /* Invalid allocation callbacks. */
    }

    // 设置 WAV 格式信息
    pWav->fmt.formatTag = (drwav_uint16)pFormat->format;
    pWav->fmt.channels = (drwav_uint16)pFormat->channels;
    pWav->fmt.sampleRate = pFormat->sampleRate;
}
    # 计算每秒平均字节数，用于确定数据传输速率
    pWav->fmt.avgBytesPerSec = (drwav_uint32)((pFormat->bitsPerSample * pFormat->sampleRate * pFormat->channels) / 8);
    # 计算数据块对齐值，用于确定数据在文件中的对齐方式
    pWav->fmt.blockAlign = (drwav_uint16)((pFormat->channels * pFormat->bitsPerSample) / 8);
    # 设置每个样本的位数
    pWav->fmt.bitsPerSample = (drwav_uint16)pFormat->bitsPerSample;
    # 扩展大小设置为0
    pWav->fmt.extendedSize = 0;
    # 设置是否为顺序写入
    pWav->isSequentialWrite = isSequential;

    # 返回成功标志
    return DRWAV_TRUE;
}

static drwav_bool32 drwav_init_write__internal(drwav* pWav, const drwav_data_format* pFormat, drwav_uint64 totalSampleCount)
{
    /* 函数假设之前调用了 drwav_preinit_write()。 */

    size_t runningPos = 0;
    drwav_uint64 initialDataChunkSize = 0;
    drwav_uint64 chunkSizeFMT;

    /*
    "RIFF" 和 "data" 块的初始值取决于是否在顺序模式下初始化。在顺序模式下，我们直接设置为最终值，因为可以从总样本数计算出来。
    在非顺序模式下，我们将其全部初始化为零，并在 drwav_uninit() 中使用向后查找来填充。
    */
    if (pWav->isSequentialWrite) {
        initialDataChunkSize = (totalSampleCount * pWav->fmt.bitsPerSample) / 8;

        /*
        RIFF 容器对样本数有限制。drwav 不允许这种情况。对于 Wave64，没有实际的限制，为了简单起见，我没有对其进行任何验证。
        */
        if (pFormat->container == drwav_container_riff) {
            if (initialDataChunkSize > (0xFFFFFFFFUL - 36)) {
                return DRWAV_FALSE; /* 没有足够的空间存储每个样本。 */
            }
        }
    }

    pWav->dataChunkDataSizeTargetWrite = initialDataChunkSize;


    /* "RIFF" 块。 */
    if (pFormat->container == drwav_container_riff) {
        drwav_uint32 chunkSizeRIFF = 28 + (drwav_uint32)initialDataChunkSize;   /* +28 = "WAVE" + [sizeof "fmt " chunk] */
        runningPos += drwav__write(pWav, "RIFF", 4);
        runningPos += drwav__write_u32ne_to_le(pWav, chunkSizeRIFF);
        runningPos += drwav__write(pWav, "WAVE", 4);
    } else if (pFormat->container == drwav_container_w64) {
        // 计算 W64 格式的 chunkSizeRIFF，包括 GUID 和大小字段在内
        drwav_uint64 chunkSizeRIFF = 80 + 24 + initialDataChunkSize;            /* +24 because W64 includes the size of the GUID and size fields. */
        // 写入 RIFF 标识符
        runningPos += drwav__write(pWav, drwavGUID_W64_RIFF, 16);
        // 写入 chunkSizeRIFF 的大小
        runningPos += drwav__write_u64ne_to_le(pWav, chunkSizeRIFF);
        // 写入 WAVE 标识符
        runningPos += drwav__write(pWav, drwavGUID_W64_WAVE, 16);
    } else if (pFormat->container == drwav_container_rf64) {
        // 写入 RF64 标识符
        runningPos += drwav__write(pWav, "RF64", 4);
        // 写入 0xFFFFFFFF，RF64 中始终为此值，在 "ds64" chunk 中设置正确值
        runningPos += drwav__write_u32ne_to_le(pWav, 0xFFFFFFFF);               /* Always 0xFFFFFFFF for RF64. Set to a proper value in the "ds64" chunk. */
        // 写入 WAVE 标识符
        runningPos += drwav__write(pWav, "WAVE", 4);
    }

    
    /* "ds64" chunk (RF64 only). */
    // 如果是 RF64 格式
    if (pFormat->container == drwav_container_rf64) {
        // 初始的 ds64ChunkSize 大小
        drwav_uint32 initialds64ChunkSize = 28;                                 /* 28 = [Size of RIFF (8 bytes)] + [Size of DATA (8 bytes)] + [Sample Count (8 bytes)] + [Table Length (4 bytes)]. Table length always set to 0. */
        // 初始的 RIFF chunk 大小
        drwav_uint64 initialRiffChunkSize = 8 + initialds64ChunkSize + initialDataChunkSize;    /* +8 for the ds64 header. */

        // 写入 ds64 标识符
        runningPos += drwav__write(pWav, "ds64", 4);
        // 写入 ds64ChunkSize 的大小
        runningPos += drwav__write_u32ne_to_le(pWav, initialds64ChunkSize);     /* Size of ds64. */
        // 写入 initialRiffChunkSize 的大小
        runningPos += drwav__write_u64ne_to_le(pWav, initialRiffChunkSize);     /* Size of RIFF. Set to true value at the end. */
        // 写入 initialDataChunkSize 的大小
        runningPos += drwav__write_u64ne_to_le(pWav, initialDataChunkSize);     /* Size of DATA. Set to true value at the end. */
        // 写入 totalSampleCount 的大小
        runningPos += drwav__write_u64ne_to_le(pWav, totalSampleCount);         /* Sample count. */
        // 写入 0，表长度，在我们的情况下始终设置为零，因为我们除了 "DATA" 之外不做任何其他 chunk
        runningPos += drwav__write_u32ne_to_le(pWav, 0);                        /* Table length. Always set to zero in our case since we're not doing any other chunks than "DATA". */
    }


    /* "fmt " chunk. */
    // 如果容器类型是 RIFF 或 RF64
    if (pFormat->container == drwav_container_riff || pFormat->container == drwav_container_rf64) {
        // 设置 FMT 块大小为 16
        chunkSizeFMT = 16;
        // 写入 "fmt " 到 WAV 文件，返回写入的字节数
        runningPos += drwav__write(pWav, "fmt ", 4);
        // 将 FMT 块大小以小端格式写入到 WAV 文件，返回写入的字节数
        runningPos += drwav__write_u32ne_to_le(pWav, (drwav_uint32)chunkSizeFMT);
    } else if (pFormat->container == drwav_container_w64) {
        // 设置 FMT 块大小为 40
        chunkSizeFMT = 40;
        // 写入 W64 格式的 FMT GUID 到 WAV 文件，返回写入的字节数
        runningPos += drwav__write(pWav, drwavGUID_W64_FMT, 16);
        // 将 FMT 块大小以小端格式写入到 WAV 文件，返回写入的字节数
        runningPos += drwav__write_u64ne_to_le(pWav, chunkSizeFMT);
    }

    // 依次写入 WAV 文件的格式信息
    runningPos += drwav__write_u16ne_to_le(pWav, pWav->fmt.formatTag);
    runningPos += drwav__write_u16ne_to_le(pWav, pWav->fmt.channels);
    runningPos += drwav__write_u32ne_to_le(pWav, pWav->fmt.sampleRate);
    runningPos += drwav__write_u32ne_to_le(pWav, pWav->fmt.avgBytesPerSec);
    runningPos += drwav__write_u16ne_to_le(pWav, pWav->fmt.blockAlign);
    runningPos += drwav__write_u16ne_to_le(pWav, pWav->fmt.bitsPerSample);

    // 设置数据块的起始位置
    pWav->dataChunkDataPos = runningPos;

    /* "data" chunk. */
    // 根据不同的容器类型写入数据块信息
    if (pFormat->container == drwav_container_riff) {
        // 计算数据块大小
        drwav_uint32 chunkSizeDATA = (drwav_uint32)initialDataChunkSize;
        // 写入 "data" 到 WAV 文件，返回写入的字节数
        runningPos += drwav__write(pWav, "data", 4);
        // 将数据块大小以小端格式写入到 WAV 文件，返回写入的字节数
        runningPos += drwav__write_u32ne_to_le(pWav, chunkSizeDATA);
    } else if (pFormat->container == drwav_container_w64) {
        // 计算数据块大小
        drwav_uint64 chunkSizeDATA = 24 + initialDataChunkSize;     /* +24 because W64 includes the size of the GUID and size fields. */
        // 写入 W64 格式的 DATA GUID 到 WAV 文件，返回写入的字节数
        runningPos += drwav__write(pWav, drwavGUID_W64_DATA, 16);
        // 将数据块大小以小端格式写入到 WAV 文件，返回写入的字节数
        runningPos += drwav__write_u64ne_to_le(pWav, chunkSizeDATA);
    } else if (pFormat->container == drwav_container_rf64) {
        // 写入 "data" 到 WAV 文件，返回写入的字节数
        runningPos += drwav__write(pWav, "data", 4);
        // 对于 RF64，数据块大小始终设置为 0xFFFFFFFF，真实大小在 ds64 块中指定
        runningPos += drwav__write_u32ne_to_le(pWav, 0xFFFFFFFF);
    }

    /*
    /* 忽略未使用的 runningPos 变量，以避免静态分析工具检测到死代码 */
    (void)runningPos;

    /* 为客户端方便设置一些属性 */
    pWav->container = pFormat->container;
    pWav->channels = (drwav_uint16)pFormat->channels;
    pWav->sampleRate = pFormat->sampleRate;
    pWav->bitsPerSample = (drwav_uint16)pFormat->bitsPerSample;
    pWav->translatedFormatTag = (drwav_uint16)pFormat->format;

    /* 返回成功标志 */
    return DRWAV_TRUE;
}

/*
    初始化写入 WAV 文件，设置格式、回调函数等参数
    pWav: WAV 文件指针
    pFormat: 数据格式
    onWrite: 写入回调函数
    onSeek: 定位回调函数
    pUserData: 用户数据
    pAllocationCallbacks: 分配器回调函数
*/
DRWAV_API drwav_bool32 drwav_init_write(drwav* pWav, const drwav_data_format* pFormat, drwav_write_proc onWrite, drwav_seek_proc onSeek, void* pUserData, const drwav_allocation_callbacks* pAllocationCallbacks)
{
    // 如果初始化写入失败，则返回假
    if (!drwav_preinit_write(pWav, pFormat, DRWAV_FALSE, onWrite, onSeek, pUserData, pAllocationCallbacks)) {
        return DRWAV_FALSE;
    }

    // 调用内部初始化写入函数，返回结果
    return drwav_init_write__internal(pWav, pFormat, 0);               /* DRWAV_FALSE = Not Sequential */
}

/*
    初始化写入 WAV 文件，设置格式、回调函数等参数，支持顺序写入
    pWav: WAV 文件指针
    pFormat: 数据格式
    totalSampleCount: 总采样数
    onWrite: 写入回调函数
    pUserData: 用户数据
    pAllocationCallbacks: 分配器回调函数
*/
DRWAV_API drwav_bool32 drwav_init_write_sequential(drwav* pWav, const drwav_data_format* pFormat, drwav_uint64 totalSampleCount, drwav_write_proc onWrite, void* pUserData, const drwav_allocation_callbacks* pAllocationCallbacks)
{
    // 如果初始化写入失败，则返回假
    if (!drwav_preinit_write(pWav, pFormat, DRWAV_TRUE, onWrite, NULL, pUserData, pAllocationCallbacks)) {
        return DRWAV_FALSE;
    }

    // 调用内部初始化写入函数，返回结果
    return drwav_init_write__internal(pWav, pFormat, totalSampleCount); /* DRWAV_TRUE = Sequential */
}

/*
    初始化写入 WAV 文件，设置格式、回调函数等参数，支持顺序写入 PCM 帧
    pWav: WAV 文件指针
    pFormat: 数据格式
    totalPCMFrameCount: 总 PCM 帧数
    onWrite: 写入回调函数
    pUserData: 用户数据
    pAllocationCallbacks: 分配器回调函数
*/
DRWAV_API drwav_bool32 drwav_init_write_sequential_pcm_frames(drwav* pWav, const drwav_data_format* pFormat, drwav_uint64 totalPCMFrameCount, drwav_write_proc onWrite, void* pUserData, const drwav_allocation_callbacks* pAllocationCallbacks)
{
    // 如果数据格式为空，则返回假
    if (pFormat == NULL) {
        return DRWAV_FALSE;
    }

    // 调用顺序写入函数，返回结果
    return drwav_init_write_sequential(pWav, pFormat, totalPCMFrameCount*pFormat->channels, onWrite, pUserData, pAllocationCallbacks);
}

/*
    计算目标写入大小（字节数）
    pFormat: 数据格式
    totalSampleCount: 总采样数
*/
DRWAV_API drwav_uint64 drwav_target_write_size_bytes(const drwav_data_format* pFormat, drwav_uint64 totalSampleCount)
{
    /* Casting totalSampleCount to drwav_int64 for VC6 compatibility. No issues in practice because nobody is going to exhaust the whole 63 bits. */
    // 计算目标数据大小（字节数）
    drwav_uint64 targetDataSizeBytes = (drwav_uint64)((drwav_int64)totalSampleCount * pFormat->channels * pFormat->bitsPerSample/8.0);
    drwav_uint64 riffChunkSizeBytes;
    drwav_uint64 fileSizeBytes = 0;
    // 如果容器类型是 RIFF
    if (pFormat->container == drwav_container_riff) {
        // 计算 RIFF Chunk 的大小
        riffChunkSizeBytes = drwav__riff_chunk_size_riff(targetDataSizeBytes);
        // 计算文件总大小，加上8是因为 WAV 文件不包括 ChunkID 和 ChunkSize 字段的大小
        fileSizeBytes = (8 + riffChunkSizeBytes);
    } 
    // 如果容器类型是 W64
    else if (pFormat->container == drwav_container_w64) {
        // 计算 RIFF Chunk 的大小
        riffChunkSizeBytes = drwav__riff_chunk_size_w64(targetDataSizeBytes);
        // 文件总大小等于 RIFF Chunk 的大小
        fileSizeBytes = riffChunkSizeBytes;
    } 
    // 如果容器类型是 RF64
    else if (pFormat->container == drwav_container_rf64) {
        // 计算 RIFF Chunk 的大小
        riffChunkSizeBytes = drwav__riff_chunk_size_rf64(targetDataSizeBytes);
        // 计算文件总大小，加上8是因为 WAV 文件不包括 ChunkID 和 ChunkSize 字段的大小
        fileSizeBytes = (8 + riffChunkSizeBytes);
    }

    // 返回文件总大小
    return fileSizeBytes;
#ifndef DR_WAV_NO_STDIO

/* drwav_result_from_errno() is only used for fopen() and wfopen() so putting it inside DR_WAV_NO_STDIO for now. If something else needs this later we can move it out. */
#include <errno.h>
static drwav_result drwav_result_from_errno(int e)
{
    switch (e)
    {
        case 0: return DRWAV_SUCCESS;
    #ifdef EPERM
        case EPERM: return DRWAV_INVALID_OPERATION;
    #endif
    #ifdef ENOENT
        case ENOENT: return DRWAV_DOES_NOT_EXIST;
    #endif
    #ifdef ESRCH
        case ESRCH: return DRWAV_DOES_NOT_EXIST;
    #endif
    #ifdef EINTR
        case EINTR: return DRWAV_INTERRUPT;
    #endif
    #ifdef EIO
        case EIO: return DRWAV_IO_ERROR;
    #endif
    #ifdef ENXIO
        case ENXIO: return DRWAV_DOES_NOT_EXIST;
    #endif
    #ifdef E2BIG
        case E2BIG: return DRWAV_INVALID_ARGS;
    #endif
    #ifdef ENOEXEC
        case ENOEXEC: return DRWAV_INVALID_FILE;
    #endif
    #ifdef EBADF
        case EBADF: return DRWAV_INVALID_FILE;
    #endif
    #ifdef ECHILD
        case ECHILD: return DRWAV_ERROR;
    #endif
    #ifdef EAGAIN
        case EAGAIN: return DRWAV_UNAVAILABLE;
    #endif
    #ifdef ENOMEM
        case ENOMEM: return DRWAV_OUT_OF_MEMORY;
    #endif
    #ifdef EACCES
        case EACCES: return DRWAV_ACCESS_DENIED;
    #endif
    #ifdef EFAULT
        case EFAULT: return DRWAV_BAD_ADDRESS;
    #endif
    #ifdef ENOTBLK
        case ENOTBLK: return DRWAV_ERROR;
    #endif
    #ifdef EBUSY
        case EBUSY: return DRWAV_BUSY;
    #endif
    #ifdef EEXIST
        case EEXIST: return DRWAV_ALREADY_EXISTS;
    #endif
    #ifdef EXDEV
        case EXDEV: return DRWAV_ERROR;
    #endif
    #ifdef ENODEV
        case ENODEV: return DRWAV_DOES_NOT_EXIST;
    #endif
    #ifdef ENOTDIR
        case ENOTDIR: return DRWAV_NOT_DIRECTORY;
    #endif
    #ifdef EISDIR
        case EISDIR: return DRWAV_IS_DIRECTORY;
    #endif
    #ifdef EINVAL
        case EINVAL: return DRWAV_INVALID_ARGS;
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
        // 如果发生网络不可达错误，返回 DRWAV_NO_NETWORK
        case ENETUNREACH: return DRWAV_NO_NETWORK;
    #endif
    
    #ifdef ENETRESET
        // 如果发生网络重置错误，返回 DRWAV_NO_NETWORK
        case ENETRESET: return DRWAV_NO_NETWORK;
    #endif
    
    #ifdef ECONNABORTED
        // 如果发生连接中止错误，返回 DRWAV_NO_NETWORK
        case ECONNABORTED: return DRWAV_NO_NETWORK;
    #endif
    
    #ifdef ECONNRESET
        // 如果发生连接重置错误，返回 DRWAV_CONNECTION_RESET
        case ECONNRESET: return DRWAV_CONNECTION_RESET;
    #endif
    
    #ifdef ENOBUFS
        // 如果发生缓冲区不足错误，返回 DRWAV_NO_SPACE
        case ENOBUFS: return DRWAV_NO_SPACE;
    #endif
    
    #ifdef EISCONN
        // 如果已经连接，返回 DRWAV_ALREADY_CONNECTED
        case EISCONN: return DRWAV_ALREADY_CONNECTED;
    #endif
    
    #ifdef ENOTCONN
        // 如果未连接，返回 DRWAV_NOT_CONNECTED
        case ENOTCONN: return DRWAV_NOT_CONNECTED;
    #endif
    
    #ifdef ESHUTDOWN
        // 如果发生关闭错误，返回 DRWAV_ERROR
        case ESHUTDOWN: return DRWAV_ERROR;
    #endif
    
    #ifdef ETOOMANYREFS
        // 如果引用过多，返回 DRWAV_ERROR
        case ETOOMANYREFS: return DRWAV_ERROR;
    #endif
    
    #ifdef ETIMEDOUT
        // 如果发生超时错误，返回 DRWAV_TIMEOUT
        case ETIMEDOUT: return DRWAV_TIMEOUT;
    #endif
    
    #ifdef ECONNREFUSED
        // 如果连接被拒绝，返回 DRWAV_CONNECTION_REFUSED
        case ECONNREFUSED: return DRWAV_CONNECTION_REFUSED;
    #endif
    
    #ifdef EHOSTDOWN
        // 如果主机不可用，返回 DRWAV_NO_HOST
        case EHOSTDOWN: return DRWAV_NO_HOST;
    #endif
    
    #ifdef EHOSTUNREACH
        // 如果主机不可达，返回 DRWAV_NO_HOST
        case EHOSTUNREACH: return DRWAV_NO_HOST;
    #endif
    
    #ifdef EALREADY
        // 如果操作已经在进行中，返回 DRWAV_IN_PROGRESS
        case EALREADY: return DRWAV_IN_PROGRESS;
    #endif
    
    #ifdef EINPROGRESS
        // 如果操作正在进行中，返回 DRWAV_IN_PROGRESS
        case EINPROGRESS: return DRWAV_IN_PROGRESS;
    #endif
    
    #ifdef ESTALE
        // 如果文件状态不正确，返回 DRWAV_INVALID_FILE
        case ESTALE: return DRWAV_INVALID_FILE;
    #endif
    
    #ifdef EUCLEAN
        // 如果发生清理错误，返回 DRWAV_ERROR
        case EUCLEAN: return DRWAV_ERROR;
    #endif
    
    #ifdef ENOTNAM
        // 如果不是一个名字，返回 DRWAV_ERROR
        case ENOTNAM: return DRWAV_ERROR;
    #endif
    
    #ifdef ENAVAIL
        // 如果资源不可用，返回 DRWAV_ERROR
        case ENAVAIL: return DRWAV_ERROR;
    #endif
    
    #ifdef EISNAM
        // 如果是一个名字，返回 DRWAV_ERROR
        case EISNAM: return DRWAV_ERROR;
    #endif
    
    #ifdef EREMOTEIO
        // 如果远程 I/O 错误，返回 DRWAV_IO_ERROR
        case EREMOTEIO: return DRWAV_IO_ERROR;
    #endif
    
    #ifdef EDQUOT
        // 如果磁盘空间不足，返回 DRWAV_NO_SPACE
        case EDQUOT: return DRWAV_NO_SPACE;
    #endif
    
    #ifdef ENOMEDIUM
        // 如果介质不存在，返回 DRWAV_DOES_NOT_EXIST
        case ENOMEDIUM: return DRWAV_DOES_NOT_EXIST;
    #endif
    
    #ifdef EMEDIUMTYPE
        // 如果介质类型错误，返回 DRWAV_ERROR
        case EMEDIUMTYPE: return DRWAV_ERROR;
    #endif
    
    #ifdef ECANCELED
        // 如果操作被取消，返回 DRWAV_CANCELLED;
        case ECANCELED: return DRWAV_CANCELLED;
    #endif
    #ifdef ENOKEY
        case ENOKEY: return DRWAV_ERROR;
    #endif
    #ifdef EKEYEXPIRED
        case EKEYEXPIRED: return DRWAV_ERROR;
    #endif
    #ifdef EKEYREVOKED
        case EKEYREVOKED: return DRWAV_ERROR;
    #endif
    #ifdef EKEYREJECTED
        case EKEYREJECTED: return DRWAV_ERROR;
    #endif
    #ifdef EOWNERDEAD
        case EOWNERDEAD: return DRWAV_ERROR;
    #endif
    #ifdef ENOTRECOVERABLE
        case ENOTRECOVERABLE: return DRWAV_ERROR;
    #endif
    #ifdef ERFKILL
        case ERFKILL: return DRWAV_ERROR;
    #endif
    #ifdef EHWPOISON
        case EHWPOISON: return DRWAV_ERROR;
    #endif
        default: return DRWAV_ERROR;
    }
}

// 打开文件并返回结果
static drwav_result drwav_fopen(FILE** ppFile, const char* pFilePath, const char* pOpenMode)
{
#if _MSC_VER && _MSC_VER >= 1400
    errno_t err;
#endif

    // 检查文件指针是否为空
    if (ppFile != NULL) {
        *ppFile = NULL;  /* Safety. */
    }

    // 检查参数是否有效
    if (pFilePath == NULL || pOpenMode == NULL || ppFile == NULL) {
        return DRWAV_INVALID_ARGS;
    }

#if _MSC_VER && _MSC_VER >= 1400
    // 使用 fopen_s() 打开文件
    err = fopen_s(ppFile, pFilePath, pOpenMode);
    // 检查错误码并返回对应的结果
    if (err != 0) {
        return drwav_result_from_errno(err);
    }
#else
#if defined(_WIN32) || defined(__APPLE__)
    // 在 Windows 或者 macOS 下使用 fopen() 打开文件
    *ppFile = fopen(pFilePath, pOpenMode);
#else
    #if defined(_FILE_OFFSET_BITS) && _FILE_OFFSET_BITS == 64 && defined(_LARGEFILE64_SOURCE)
        // 在特定条件下使用 fopen64() 打开文件
        *ppFile = fopen64(pFilePath, pOpenMode);
    #else
        // 默认情况下使用 fopen() 打开文件
        *ppFile = fopen(pFilePath, pOpenMode);
    #endif
#endif
    // 检查文件指针是否为空，并返回对应的结果
    if (*ppFile == NULL) {
        drwav_result result = drwav_result_from_errno(errno);
        if (result == DRWAV_SUCCESS) {
            result = DRWAV_ERROR;   /* Just a safety check to make sure we never ever return success when pFile == NULL. */
        }

        return result;
    }
#endif

    // 返回成功结果
    return DRWAV_SUCCESS;
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
        #define DRWAV_HAS_WFOPEN
    #endif
#endif
static drwav_result drwav_wfopen(FILE** ppFile, const wchar_t* pFilePath, const wchar_t* pOpenMode, const drwav_allocation_callbacks* pAllocationCallbacks)
{
    // 如果传入的文件指针不为空，将其置为 NULL，确保安全
    if (ppFile != NULL) {
        *ppFile = NULL;  /* Safety. */
    }

    // 如果文件路径、打开模式或文件指针为空，返回参数无效错误
    if (pFilePath == NULL || pOpenMode == NULL || ppFile == NULL) {
        return DRWAV_INVALID_ARGS;
    }

#if defined(DRWAV_HAS_WFOPEN)
    {
        /* 在 Windows 上使用 _wfopen() */
    #if defined(_MSC_VER) && _MSC_VER >= 1400
        // 如果是 Visual Studio 2005 及以上版本，使用 _wfopen_s() 打开文件
        errno_t err = _wfopen_s(ppFile, pFilePath, pOpenMode);
        if (err != 0) {
            return drwav_result_from_errno(err);
        }
    #else
        // 否则使用 _wfopen() 打开文件
        *ppFile = _wfopen(pFilePath, pOpenMode);
        if (*ppFile == NULL) {
            return drwav_result_from_errno(errno);
        }
    #endif
        // 忽略分配回调
        (void)pAllocationCallbacks;
    }
#else
    /*
    在 Windows 以外的系统上使用 fopen() 打开文件，需要进行转换。这很麻烦，因为 fopen() 是与区域设置相关的。
    我能想到的唯一方法是使用 wcsrtombs()。注意，wcstombs() 显然不是线程安全的，因为它使用一个静态全局的 mbstate_t 对象来维护状态。
    我已经检查了这一点，使用 -std=c89 是可以的，但如果有人遇到编译错误，我会考虑改进兼容性。
    */
    {
        // 定义多字节状态结构体
        mbstate_t mbs;
        // 多字节长度
        size_t lenMB;
        // 临时存放宽字符路径
        const wchar_t* pFilePathTemp = pFilePath;
        // 存放多字节路径
        char* pFilePathMB = NULL;
        // 存放打开模式的多字节字符串
        char pOpenModeMB[32] = {0};
    
        /* 先获取长度。*/
        // 初始化多字节状态结构体
        DRWAV_ZERO_OBJECT(&mbs);
        // 获取多字节长度
        lenMB = wcsrtombs(NULL, &pFilePathTemp, 0, &mbs);
        // 如果获取长度失败，返回对应的错误码
        if (lenMB == (size_t)-1) {
            return drwav_result_from_errno(errno);
        }
    
        // 分配内存存放多字节路径
        pFilePathMB = (char*)drwav__malloc_from_callbacks(lenMB + 1, pAllocationCallbacks);
        // 如果内存分配失败，返回内存不足错误
        if (pFilePathMB == NULL) {
            return DRWAV_OUT_OF_MEMORY;
        }
    
        // 重新设置宽字符路径
        pFilePathTemp = pFilePath;
        // 重新初始化多字节状态结构体
        DRWAV_ZERO_OBJECT(&mbs);
        // 将宽字符路径转换为多字节路径
        wcsrtombs(pFilePathMB, &pFilePathTemp, lenMB + 1, &mbs);
    
        /* 打开模式应该始终由 ASCII 字符组成，因此我们应该能够进行简单的转换。*/
        {
            // 遍历打开模式字符串
            size_t i = 0;
            for (;;) {
                // 如果遇到字符串结束符，设置多字节打开模式字符串结束符并跳出循环
                if (pOpenMode[i] == 0) {
                    pOpenModeMB[i] = '\0';
                    break;
                }
    
                // 将宽字符打开模式转换为多字节字符
                pOpenModeMB[i] = (char)pOpenMode[i];
                i += 1;
            }
        }
    
        // 打开文件并将文件指针存入 ppFile
        *ppFile = fopen(pFilePathMB, pOpenModeMB);
    
        // 释放多字节路径内存
        drwav__free_from_callbacks(pFilePathMB, pAllocationCallbacks);
    }
    
    // 如果文件指针为空，返回错误
    if (*ppFile == NULL) {
        return DRWAV_ERROR;
    }
#endif

    return DRWAV_SUCCESS;
}


static size_t drwav__on_read_stdio(void* pUserData, void* pBufferOut, size_t bytesToRead)
{
    // 从标准输入流中读取数据到缓冲区
    return fread(pBufferOut, 1, bytesToRead, (FILE*)pUserData);
}

static size_t drwav__on_write_stdio(void* pUserData, const void* pData, size_t bytesToWrite)
{
    // 将数据写入标准输出流
    return fwrite(pData, 1, bytesToWrite, (FILE*)pUserData);
}

static drwav_bool32 drwav__on_seek_stdio(void* pUserData, int offset, drwav_seek_origin origin)
{
    // 在标准输入流中进行定位操作
    return fseek((FILE*)pUserData, offset, (origin == drwav_seek_origin_current) ? SEEK_CUR : SEEK_SET) == 0;
}

DRWAV_API drwav_bool32 drwav_init_file(drwav* pWav, const char* filename, const drwav_allocation_callbacks* pAllocationCallbacks)
{
    // 初始化文件读取，调用 drwav_init_file_ex 函数
    return drwav_init_file_ex(pWav, filename, NULL, NULL, 0, pAllocationCallbacks);
}


static drwav_bool32 drwav_init_file__internal_FILE(drwav* pWav, FILE* pFile, drwav_chunk_proc onChunk, void* pChunkUserData, drwav_uint32 flags, const drwav_allocation_callbacks* pAllocationCallbacks)
{
    drwav_bool32 result;

    // 预初始化 WAV 文件，设置读取和定位回调函数
    result = drwav_preinit(pWav, drwav__on_read_stdio, drwav__on_seek_stdio, (void*)pFile, pAllocationCallbacks);
    if (result != DRWAV_TRUE) {
        fclose(pFile);
        return result;
    }

    // 内部初始化 WAV 文件，设置块处理回调函数
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
    // 打开文件，以二进制只读方式
    if (drwav_fopen(&pFile, filename, "rb") != DRWAV_SUCCESS) {
        return DRWAV_FALSE;
    }

    /* This takes ownership of the FILE* object. */
    // 初始化文件，调用 drwav_init_file__internal_FILE 函数
    return drwav_init_file__internal_FILE(pWav, pFile, onChunk, pChunkUserData, flags, pAllocationCallbacks);
}
# 使用宽字符文件名初始化 WAV 文件，使用默认内存分配器
DRWAV_API drwav_bool32 drwav_init_file_w(drwav* pWav, const wchar_t* filename, const drwav_allocation_callbacks* pAllocationCallbacks)
{
    # 调用 drwav_init_file_ex_w 函数，传入默认参数
    return drwav_init_file_ex_w(pWav, filename, NULL, NULL, 0, pAllocationCallbacks);
}

# 使用宽字符文件名初始化 WAV 文件，可指定回调函数和标志
DRWAV_API drwav_bool32 drwav_init_file_ex_w(drwav* pWav, const wchar_t* filename, drwav_chunk_proc onChunk, void* pChunkUserData, drwav_uint32 flags, const drwav_allocation_callbacks* pAllocationCallbacks)
{
    # 声明 FILE 指针
    FILE* pFile;
    # 尝试打开文件，如果失败则返回 DRWAV_FALSE
    if (drwav_wfopen(&pFile, filename, L"rb", pAllocationCallbacks) != DRWAV_SUCCESS) {
        return DRWAV_FALSE;
    }

    /* This takes ownership of the FILE* object. */
    # 调用内部函数，传入 FILE 指针
    return drwav_init_file__internal_FILE(pWav, pFile, onChunk, pChunkUserData, flags, pAllocationCallbacks);
}

# 使用 FILE 指针初始化 WAV 文件写入，可指定数据格式、总样本数、是否顺序写入
static drwav_bool32 drwav_init_file_write__internal_FILE(drwav* pWav, FILE* pFile, const drwav_data_format* pFormat, drwav_uint64 totalSampleCount, drwav_bool32 isSequential, const drwav_allocation_callbacks* pAllocationCallbacks)
{
    drwav_bool32 result;

    # 预初始化写入操作
    result = drwav_preinit_write(pWav, pFormat, isSequential, drwav__on_write_stdio, drwav__on_seek_stdio, (void*)pFile, pAllocationCallbacks);
    # 如果预初始化失败，则关闭文件并返回结果
    if (result != DRWAV_TRUE) {
        fclose(pFile);
        return result;
    }

    # 初始化写入操作
    result = drwav_init_write__internal(pWav, pFormat, totalSampleCount);
    # 如果初始化失败，则关闭文件并返回结果
    if (result != DRWAV_TRUE) {
        fclose(pFile);
        return result;
    }

    return DRWAV_TRUE;
}

# 使用文件名初始化 WAV 文件写入，可指定数据格式、总样本数、是否顺序写入
static drwav_bool32 drwav_init_file_write__internal(drwav* pWav, const char* filename, const drwav_data_format* pFormat, drwav_uint64 totalSampleCount, drwav_bool32 isSequential, const drwav_allocation_callbacks* pAllocationCallbacks)
{
    FILE* pFile;
    # 尝试打开文件，如果失败则返回 DRWAV_FALSE
    if (drwav_fopen(&pFile, filename, "wb") != DRWAV_SUCCESS) {
        return DRWAV_FALSE;
    }

    /* This takes ownership of the FILE* object. */
    # 调用内部函数，传入 FILE 指针
    return drwav_init_file_write__internal_FILE(pWav, pFile, pFormat, totalSampleCount, isSequential, pAllocationCallbacks);
}
static drwav_bool32 drwav_init_file_write_w__internal(drwav* pWav, const wchar_t* filename, const drwav_data_format* pFormat, drwav_uint64 totalSampleCount, drwav_bool32 isSequential, const drwav_allocation_callbacks* pAllocationCallbacks)
{
    // 打开文件
    FILE* pFile;
    if (drwav_wfopen(&pFile, filename, L"wb", pAllocationCallbacks) != DRWAV_SUCCESS) {
        return DRWAV_FALSE;
    }

    /* This takes ownership of the FILE* object. */
    // 调用内部函数，传入文件指针和其他参数
    return drwav_init_file_write__internal_FILE(pWav, pFile, pFormat, totalSampleCount, isSequential, pAllocationCallbacks);
}

DRWAV_API drwav_bool32 drwav_init_file_write(drwav* pWav, const char* filename, const drwav_data_format* pFormat, const drwav_allocation_callbacks* pAllocationCallbacks)
{
    // 调用内部函数，传入文件名和其他参数
    return drwav_init_file_write__internal(pWav, filename, pFormat, 0, DRWAV_FALSE, pAllocationCallbacks);
}

DRWAV_API drwav_bool32 drwav_init_file_write_sequential(drwav* pWav, const char* filename, const drwav_data_format* pFormat, drwav_uint64 totalSampleCount, const drwav_allocation_callbacks* pAllocationCallbacks)
{
    // 调用内部函数，传入文件名和其他参数
    return drwav_init_file_write__internal(pWav, filename, pFormat, totalSampleCount, DRWAV_TRUE, pAllocationCallbacks);
}

DRWAV_API drwav_bool32 drwav_init_file_write_sequential_pcm_frames(drwav* pWav, const char* filename, const drwav_data_format* pFormat, drwav_uint64 totalPCMFrameCount, const drwav_allocation_callbacks* pAllocationCallbacks)
{
    // 检查格式是否为空
    if (pFormat == NULL) {
        return DRWAV_FALSE;
    }

    // 调用内部函数，传入文件名和其他参数
    return drwav_init_file_write_sequential(pWav, filename, pFormat, totalPCMFrameCount*pFormat->channels, pAllocationCallbacks);
}

DRWAV_API drwav_bool32 drwav_init_file_write_w(drwav* pWav, const wchar_t* filename, const drwav_data_format* pFormat, const drwav_allocation_callbacks* pAllocationCallbacks)
{
    // 调用内部函数，传入文件名和其他参数
    return drwav_init_file_write_w__internal(pWav, filename, pFormat, 0, DRWAV_FALSE, pAllocationCallbacks);
}
// 初始化一个用于按顺序写入 WAV 文件的函数，接受一个 wchar_t 类型的文件名，数据格式，总采样数和内存分配回调
DRWAV_API drwav_bool32 drwav_init_file_write_sequential_w(drwav* pWav, const wchar_t* filename, const drwav_data_format* pFormat, drwav_uint64 totalSampleCount, const drwav_allocation_callbacks* pAllocationCallbacks)
{
    // 调用内部函数 drwav_init_file_write_w__internal 来实现初始化
    return drwav_init_file_write_w__internal(pWav, filename, pFormat, totalSampleCount, DRWAV_TRUE, pAllocationCallbacks);
}

// 初始化一个用于按顺序写入 PCM 帧的 WAV 文件的函数，接受一个 wchar_t 类型的文件名，数据格式，总 PCM 帧数和内存分配回调
DRWAV_API drwav_bool32 drwav_init_file_write_sequential_pcm_frames_w(drwav* pWav, const wchar_t* filename, const drwav_data_format* pFormat, drwav_uint64 totalPCMFrameCount, const drwav_allocation_callbacks* pAllocationCallbacks)
{
    // 如果数据格式为空，则返回假
    if (pFormat == NULL) {
        return DRWAV_FALSE;
    }

    // 调用 drwav_init_file_write_sequential_w 函数来实现初始化，总 PCM 帧数乘以通道数作为参数
    return drwav_init_file_write_sequential_w(pWav, filename, pFormat, totalPCMFrameCount*pFormat->channels, pAllocationCallbacks);
}
#endif  /* DR_WAV_NO_STDIO */

// 用于从内存中读取数据的回调函数，接受用户数据，输出缓冲区和要读取的字节数
static size_t drwav__on_read_memory(void* pUserData, void* pBufferOut, size_t bytesToRead)
{
    // 将用户数据转换为 drwav 结构体
    drwav* pWav = (drwav*)pUserData;
    size_t bytesRemaining;

    // 断言 WAV 结构体不为空
    DRWAV_ASSERT(pWav != NULL);
    // 断言内存流数据大小大于等于当前读取位置
    DRWAV_ASSERT(pWav->memoryStream.dataSize >= pWav->memoryStream.currentReadPos);

    // 计算剩余字节数
    bytesRemaining = pWav->memoryStream.dataSize - pWav->memoryStream.currentReadPos;
    // 如果要读取的字节数大于剩余字节数，则将要读取的字节数设为剩余字节数
    if (bytesToRead > bytesRemaining) {
        bytesToRead = bytesRemaining;
    }

    // 如果要读取的字节数大于 0
    if (bytesToRead > 0) {
        // 将数据从内存流中复制到输出缓冲区
        DRWAV_COPY_MEMORY(pBufferOut, pWav->memoryStream.data + pWav->memoryStream.currentReadPos, bytesToRead);
        // 更新当前读取位置
        pWav->memoryStream.currentReadPos += bytesToRead;
    }

    // 返回实际读取的字节数
    return bytesToRead;
}

// 用于在内存中进行定位的回调函数，接受用户数据，偏移量和定位方式
static drwav_bool32 drwav__on_seek_memory(void* pUserData, int offset, drwav_seek_origin origin)
{
    // 将用户数据转换为 drwav 结构体
    drwav* pWav = (drwav*)pUserData;
    // 断言 WAV 结构体不为空
    DRWAV_ASSERT(pWav != NULL);
    # 如果是相对当前位置的偏移
    if (origin == drwav_seek_origin_current):
        # 如果偏移量大于0
        if (offset > 0):
            # 如果当前读取位置加上偏移量超过数据大小，返回假
            if (pWav->memoryStream.currentReadPos + offset > pWav->memoryStream.dataSize):
                return DRWAV_FALSE; /* Trying to seek too far forward. */
        else:
            # 如果当前读取位置小于负偏移量，返回假
            if (pWav->memoryStream.currentReadPos < (size_t)-offset):
                return DRWAV_FALSE; /* Trying to seek too far backwards. */

        # 由于上面的限制，这里不会出现下溢
        pWav->memoryStream.currentReadPos += offset;
    else:
        # 如果是绝对位置的偏移
        if ((drwav_uint32)offset <= pWav->memoryStream.dataSize):
            # 如果偏移量小于等于数据大小，设置当前读取位置为偏移量
            pWav->memoryStream.currentReadPos = offset;
        else:
            return DRWAV_FALSE; /* Trying to seek too far forward. */
    
    # 返回真
    return DRWAV_TRUE;
static size_t drwav__on_write_memory(void* pUserData, const void* pDataIn, size_t bytesToWrite)
{
    // 将用户数据转换为 drwav 结构体指针
    drwav* pWav = (drwav*)pUserData;
    size_t bytesRemaining;

    // 断言确保 pWav 不为空
    DRWAV_ASSERT(pWav != NULL);
    // 断言确保内存流写入数据容量大于等于当前写入位置
    DRWAV_ASSERT(pWav->memoryStreamWrite.dataCapacity >= pWav->memoryStreamWrite.currentWritePos);

    // 计算剩余可写入字节数
    bytesRemaining = pWav->memoryStreamWrite.dataCapacity - pWav->memoryStreamWrite.currentWritePos;
    if (bytesRemaining < bytesToWrite) {
        /* 需要重新分配内存。*/
        void* pNewData;
        size_t newDataCapacity = (pWav->memoryStreamWrite.dataCapacity == 0) ? 256 : pWav->memoryStreamWrite.dataCapacity * 2;

        /* 如果加倍仍不够，将其设置为写入数据所需的最小容量。*/
        if ((newDataCapacity - pWav->memoryStreamWrite.currentWritePos) < bytesToWrite) {
            newDataCapacity = pWav->memoryStreamWrite.currentWritePos + bytesToWrite;
        }

        // 重新分配内存
        pNewData = drwav__realloc_from_callbacks(*pWav->memoryStreamWrite.ppData, newDataCapacity, pWav->memoryStreamWrite.dataCapacity, &pWav->allocationCallbacks);
        if (pNewData == NULL) {
            return 0;
        }

        *pWav->memoryStreamWrite.ppData = pNewData;
        pWav->memoryStreamWrite.dataCapacity = newDataCapacity;
    }

    // 将数据写入内存流
    DRWAV_COPY_MEMORY(((drwav_uint8*)(*pWav->memoryStreamWrite.ppData)) + pWav->memoryStreamWrite.currentWritePos, pDataIn, bytesToWrite);

    // 更新当前写入位置
    pWav->memoryStreamWrite.currentWritePos += bytesToWrite;
    // 如果数据大小小于当前写入位置，更新数据大小
    if (pWav->memoryStreamWrite.dataSize < pWav->memoryStreamWrite.currentWritePos) {
        pWav->memoryStreamWrite.dataSize = pWav->memoryStreamWrite.currentWritePos;
    }

    *pWav->memoryStreamWrite.pDataSize = pWav->memoryStreamWrite.dataSize;

    return bytesToWrite;
}

static drwav_bool32 drwav__on_seek_memory_write(void* pUserData, int offset, drwav_seek_origin origin)
{
    // 将用户数据转换为 drwav 结构体指针
    drwav* pWav = (drwav*)pUserData;
    // 断言确保 pWav 不为空
    DRWAV_ASSERT(pWav != NULL);
    # 如果是相对当前位置的偏移
    if (origin == drwav_seek_origin_current):
        # 如果偏移量大于0
        if (offset > 0):
            # 如果当前写入位置加上偏移量大于数据大小
            if (pWav->memoryStreamWrite.currentWritePos + offset > pWav->memoryStreamWrite.dataSize):
                offset = (int)(pWav->memoryStreamWrite.dataSize - pWav->memoryStreamWrite.currentWritePos)  # 尝试向前寻址太远
        else:
            # 如果当前写入位置小于负偏移量
            if (pWav->memoryStreamWrite.currentWritePos < (size_t)-offset):
                offset = -(int)pWav->memoryStreamWrite.currentWritePos  # 尝试向后寻址太远

        # 由于上面的限制，这里不会出现下溢
        pWav->memoryStreamWrite.currentWritePos += offset
    else:
        # 如果偏移量小于等于数据大小
        if ((drwav_uint32)offset <= pWav->memoryStreamWrite.dataSize):
            pWav->memoryStreamWrite.currentWritePos = offset
        else:
            pWav->memoryStreamWrite.currentWritePos = pWav->memoryStreamWrite.dataSize  # 尝试向前寻址太远

    # 返回成功
    return DRWAV_TRUE;
// 初始化一个内存中的 WAV 文件，从给定的数据中读取
DRWAV_API drwav_bool32 drwav_init_memory(drwav* pWav, const void* data, size_t dataSize, const drwav_allocation_callbacks* pAllocationCallbacks)
{
    // 调用 drwav_init_memory_ex 函数，传入默认参数
    return drwav_init_memory_ex(pWav, data, dataSize, NULL, NULL, 0, pAllocationCallbacks);
}

// 初始化一个内存中的 WAV 文件，从给定的数据中读取，可以指定回调函数和标志
DRWAV_API drwav_bool32 drwav_init_memory_ex(drwav* pWav, const void* data, size_t dataSize, drwav_chunk_proc onChunk, void* pChunkUserData, drwav_uint32 flags, const drwav_allocation_callbacks* pAllocationCallbacks)
{
    // 检查数据是否为空或大小为0
    if (data == NULL || dataSize == 0) {
        return DRWAV_FALSE;
    }

    // 调用 drwav_preinit 函数，准备初始化 WAV 文件
    if (!drwav_preinit(pWav, drwav__on_read_memory, drwav__on_seek_memory, pWav, pAllocationCallbacks)) {
        return DRWAV_FALSE;
    }

    // 设置内存流的数据、大小和当前读取位置
    pWav->memoryStream.data = (const drwav_uint8*)data;
    pWav->memoryStream.dataSize = dataSize;
    pWav->memoryStream.currentReadPos = 0;

    // 调用内部初始化函数，返回结果
    return drwav_init__internal(pWav, onChunk, pChunkUserData, flags);
}

// 初始化一个内存中的 WAV 文件，用于写入数据
static drwav_bool32 drwav_init_memory_write__internal(drwav* pWav, void** ppData, size_t* pDataSize, const drwav_data_format* pFormat, drwav_uint64 totalSampleCount, drwav_bool32 isSequential, const drwav_allocation_callbacks* pAllocationCallbacks)
{
    // 检查数据指针是否为空
    if (ppData == NULL || pDataSize == NULL) {
        return DRWAV_FALSE;
    }

    // 重置数据指针和大小
    *ppData = NULL; /* Important because we're using realloc()! */
    *pDataSize = 0;

    // 准备初始化写入 WAV 文件
    if (!drwav_preinit_write(pWav, pFormat, isSequential, drwav__on_write_memory, drwav__on_seek_memory_write, pWav, pAllocationCallbacks)) {
        return DRWAV_FALSE;
    }

    // 设置内存流写入的数据指针、大小、容量和当前写入位置
    pWav->memoryStreamWrite.ppData = ppData;
    pWav->memoryStreamWrite.pDataSize = pDataSize;
    pWav->memoryStreamWrite.dataSize = 0;
    pWav->memoryStreamWrite.dataCapacity = 0;
    pWav->memoryStreamWrite.currentWritePos = 0;

    // 调用内部初始化写入函数，返回结果
    return drwav_init_write__internal(pWav, pFormat, totalSampleCount);
}

// 初始化一个内存中的 WAV 文件，用于写入数据
DRWAV_API drwav_bool32 drwav_init_memory_write(drwav* pWav, void** ppData, size_t* pDataSize, const drwav_data_format* pFormat, const drwav_allocation_callbacks* pAllocationCallbacks)
{
    # 调用内部函数 drwav_init_memory_write__internal，初始化内存写入操作
    return drwav_init_memory_write__internal(pWav, ppData, pDataSize, pFormat, 0, DRWAV_FALSE, pAllocationCallbacks);
}

// 初始化一个用于顺序写入内存的 WAV 文件对象
DRWAV_API drwav_bool32 drwav_init_memory_write_sequential(drwav* pWav, void** ppData, size_t* pDataSize, const drwav_data_format* pFormat, drwav_uint64 totalSampleCount, const drwav_allocation_callbacks* pAllocationCallbacks)
{
    // 调用内部函数初始化内存写入 WAV 文件对象
    return drwav_init_memory_write__internal(pWav, ppData, pDataSize, pFormat, totalSampleCount, DRWAV_TRUE, pAllocationCallbacks);
}

// 初始化一个用于顺序写入内存的 PCM 帧的 WAV 文件对象
DRWAV_API drwav_bool32 drwav_init_memory_write_sequential_pcm_frames(drwav* pWav, void** ppData, size_t* pDataSize, const drwav_data_format* pFormat, drwav_uint64 totalPCMFrameCount, const drwav_allocation_callbacks* pAllocationCallbacks)
{
    // 如果格式为空，则返回假
    if (pFormat == NULL) {
        return DRWAV_FALSE;
    }

    // 调用顺序写入内存的函数，传入 PCM 帧数目而非样本数目
    return drwav_init_memory_write_sequential(pWav, ppData, pDataSize, pFormat, totalPCMFrameCount*pFormat->channels, pAllocationCallbacks);
}

// 结束 WAV 文件对象
DRWAV_API drwav_result drwav_uninit(drwav* pWav)
{
    drwav_result result = DRWAV_SUCCESS;

    // 如果 WAV 对象为空，则返回无效参数
    if (pWav == NULL) {
        return DRWAV_INVALID_ARGS;
    }

    /*
    如果 WAV 对象是以写入模式打开的，我们需要完成一些事情：
      - 确保 "data" 块对于 RIFF 容器是 16 位对齐，对于 W64 容器是 64 位对齐。
      - 设置 "data" 块的大小。
    */
    }

#ifndef DR_WAV_NO_STDIO
    /*
    如果我们使用 drwav_open_file() 打开了文件，我们将关闭文件句柄。我们可以通过查看 onRead 和 onSeek 回调函数来判断是否使用了 drwav_open_file()。
    */
    if (pWav->onRead == drwav__on_read_stdio || pWav->onWrite == drwav__on_write_stdio) {
        fclose((FILE*)pWav->pUserData);
    }
#endif

    return result;
}

// 读取原始数据
DRWAV_API size_t drwav_read_raw(drwav* pWav, size_t bytesToRead, void* pBufferOut)
{
    size_t bytesRead;

    // 如果 WAV 对象为空或要读取的字节数为 0，则返回 0
    if (pWav == NULL || bytesToRead == 0) {
        return 0;
    }

    // 如果要读取的字节数大于剩余字节数，则将要读取的字节数设为剩余字节数
    if (bytesToRead > pWav->bytesRemaining) {
        bytesToRead = (size_t)pWav->bytesRemaining;
    }
    # 如果输出缓冲区不为空
    if (pBufferOut != NULL) {
        # 调用onRead函数读取数据到输出缓冲区，并返回读取的字节数
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

            # 调用onSeek函数进行定位，如果失败则退出循环
            if (pWav->onSeek(pWav->pUserData, (int)bytesToSeek, drwav_seek_origin_current) == DRWAV_FALSE) {
                break;
            }

            bytesRead += bytesToSeek;
        }

        # 当执行到这里时，可能需要读取并丢弃一些数据。
        while (bytesRead < bytesToRead) {
            # 创建一个缓冲区用于读取数据
            drwav_uint8 buffer[4096];
            size_t bytesSeeked;
            size_t bytesToSeek = (bytesToRead - bytesRead);
            if (bytesToSeek > sizeof(buffer)) {
                bytesToSeek = sizeof(buffer);
            }

            # 调用onRead函数读取数据到缓冲区，并返回读取的字节数
            bytesSeeked = pWav->onRead(pWav->pUserData, buffer, bytesToSeek);
            bytesRead += bytesSeeked;

            if (bytesSeeked < bytesToSeek) {
                break;  # 到达文件末尾
            }
        }
    }

    # 更新剩余字节数
    pWav->bytesRemaining -= bytesRead;
    # 返回读取的字节数
    return bytesRead;
}

/*
读取指定数量的 PCM 帧数据（小端序）
*/
DRWAV_API drwav_uint64 drwav_read_pcm_frames_le(drwav* pWav, drwav_uint64 framesToRead, void* pBufferOut)
{
    drwav_uint32 bytesPerFrame;  // 每帧的字节数
    drwav_uint64 bytesToRead;    /* 故意使用 uint64 而不是 size_t，以便在 32 位构建中检查我们没有读取过多。 */

    if (pWav == NULL || framesToRead == 0) {
        return 0;
    }

    /* 不能用于压缩格式。 */
    if (drwav__is_compressed_format_tag(pWav->translatedFormatTag)) {
        return 0;
    }

    bytesPerFrame = drwav_get_bytes_per_pcm_frame(pWav);  // 获取每帧的字节数
    if (bytesPerFrame == 0) {
        return 0;
    }

    /* 不要尝试读取超过输出缓冲区可能容纳的样本数。 */
    bytesToRead = framesToRead * bytesPerFrame;
    if (bytesToRead > DRWAV_SIZE_MAX) {
        bytesToRead = (DRWAV_SIZE_MAX / bytesPerFrame) * bytesPerFrame; /* 将要读取的字节数舍入到干净的帧边界。 */
    }

    /*
    在这里明确检查，以确保在没有字节可读时不要尝试读取任何内容。可能会因溢出而评估为 0。
    */
    if (bytesToRead == 0) {
        return 0;
    }

    return drwav_read_raw(pWav, (size_t)bytesToRead, pBufferOut) / bytesPerFrame;
}

/*
读取指定数量的 PCM 帧数据（大端序）
*/
DRWAV_API drwav_uint64 drwav_read_pcm_frames_be(drwav* pWav, drwav_uint64 framesToRead, void* pBufferOut)
{
    drwav_uint64 framesRead = drwav_read_pcm_frames_le(pWav, framesToRead, pBufferOut);

    if (pBufferOut != NULL) {
        drwav__bswap_samples(pBufferOut, framesRead*pWav->channels, drwav_get_bytes_per_pcm_frame(pWav)/pWav->channels, pWav->translatedFormatTag);
    }

    return framesRead;
}

/*
读取指定数量的 PCM 帧数据
*/
DRWAV_API drwav_uint64 drwav_read_pcm_frames(drwav* pWav, drwav_uint64 framesToRead, void* pBufferOut)
{
    if (drwav__is_little_endian()) {
        return drwav_read_pcm_frames_le(pWav, framesToRead, pBufferOut);
    } else {
        // 如果是大端字节序，则调用drwav_read_pcm_frames_be函数读取PCM帧数据
        return drwav_read_pcm_frames_be(pWav, framesToRead, pBufferOut);
    }
}

/* 定位到 WAV 文件的第一个 PCM 帧 */
DRWAV_API drwav_bool32 drwav_seek_to_first_pcm_frame(drwav* pWav)
{
    /* 如果是写入模式，则不支持定位 */
    if (pWav->onWrite != NULL) {
        return DRWAV_FALSE; /* No seeking in write mode. */
    }

    /* 使用 onSeek 函数将文件指针移动到数据块的起始位置 */
    if (!pWav->onSeek(pWav->pUserData, (int)pWav->dataChunkDataPos, drwav_seek_origin_start)) {
        return DRWAV_FALSE;
    }

    /* 对于压缩格式，需要清除缓存数据 */
    if (drwav__is_compressed_format_tag(pWav->translatedFormatTag)) {
        pWav->compressed.iCurrentPCMFrame = 0;

        /* 对于 ADPCM 格式，清除 msadpcm 结构体 */
        if (pWav->translatedFormatTag == DR_WAVE_FORMAT_ADPCM) {
            DRWAV_ZERO_OBJECT(&pWav->msadpcm);
        } 
        /* 对于 DVI ADPCM 格式，清除 ima 结构体 */
        else if (pWav->translatedFormatTag == DR_WAVE_FORMAT_DVI_ADPCM) {
            DRWAV_ZERO_OBJECT(&pWav->ima);
        } 
        /* 如果触发了此断言，则表示实现了新的压缩格式但忘记在此处添加分支 */
        else {
            DRWAV_ASSERT(DRWAV_FALSE);  
        }
    }
    
    /* 设置剩余字节数为数据块大小 */
    pWav->bytesRemaining = pWav->dataChunkDataSize;
    return DRWAV_TRUE;
}

/* 定位到指定 PCM 帧 */
DRWAV_API drwav_bool32 drwav_seek_to_pcm_frame(drwav* pWav, drwav_uint64 targetFrameIndex)
{
    /* 定位应与大于 2GB 的波形文件兼容 */

    if (pWav == NULL || pWav->onSeek == NULL) {
        return DRWAV_FALSE;
    }

    /* 如果是写入模式，则不支持定位 */
    if (pWav->onWrite != NULL) {
        return DRWAV_FALSE;
    }

    /* 如果没有样本，则直接返回成功 */
    if (pWav->totalPCMFrameCount == 0) {
        return DRWAV_TRUE;
    }

    /* 确保目标帧索引在有效范围内 */
    if (targetFrameIndex >= pWav->totalPCMFrameCount) {
        targetFrameIndex  = pWav->totalPCMFrameCount - 1;
    }

    /*
    对于压缩格式，使用慢速通用定位。如果向前定位，就向前定位。如果向后定位，就需要回到起始位置。
    */
    if (drwav__is_compressed_format_tag(pWav->translatedFormatTag)) {
        /* 如果是压缩格式的音频数据，需要进行特殊处理 */

        /*
        如果是向前查找，简单地读取样本直到找到请求的样本。如果是向后查找，
        首先需要回到开头，然后执行与向前查找相同的操作。
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
                    DRWAV_ASSERT(DRWAV_FALSE);  /* 如果触发此断言，表示我实现了新的压缩格式但忘记在这里添加相应的分支。 */
                }

                if (framesRead != framesToRead) {
                    return DRWAV_FALSE;
                }

                offsetInFrames -= framesRead;
            }
        }
    } else {
        // 定义变量，用于存储总字节数、当前字节位置、目标字节位置和偏移量
        drwav_uint64 totalSizeInBytes;
        drwav_uint64 currentBytePos;
        drwav_uint64 targetBytePos;
        drwav_uint64 offset;

        // 计算总字节数
        totalSizeInBytes = pWav->totalPCMFrameCount * drwav_get_bytes_per_pcm_frame(pWav);
        // 断言总字节数大于等于剩余字节数
        DRWAV_ASSERT(totalSizeInBytes >= pWav->bytesRemaining);

        // 计算当前字节位置和目标字节位置
        currentBytePos = totalSizeInBytes - pWav->bytesRemaining;
        targetBytePos  = targetFrameIndex * drwav_get_bytes_per_pcm_frame(pWav);

        // 根据当前字节位置和目标字节位置计算偏移量
        if (currentBytePos < targetBytePos) {
            /* Offset forwards. */
            offset = (targetBytePos - currentBytePos);
        } else {
            /* Offset backwards. */
            // 如果需要向后偏移，则将文件指针移动到第一个 PCM 帧
            if (!drwav_seek_to_first_pcm_frame(pWav)) {
                return DRWAV_FALSE;
            }
            offset = targetBytePos;
        }

        // 循环直到偏移量为0
        while (offset > 0) {
            // 将偏移量转换为32位整数
            int offset32 = ((offset > INT_MAX) ? INT_MAX : (int)offset);
            // 调用 onSeek 函数移动文件指针
            if (!pWav->onSeek(pWav->pUserData, offset32, drwav_seek_origin_current)) {
                return DRWAV_FALSE;
            }

            // 更新剩余字节数和偏移量
            pWav->bytesRemaining -= offset32;
            offset -= offset32;
        }
    }

    // 返回操作结果
    return DRWAV_TRUE;
}

# 写入原始数据到 WAV 文件
DRWAV_API size_t drwav_write_raw(drwav* pWav, size_t bytesToWrite, const void* pData)
{
    size_t bytesWritten;

    # 检查输入参数是否有效
    if (pWav == NULL || bytesToWrite == 0 || pData == NULL) {
        return 0;
    }

    # 调用回调函数写入数据，并更新数据块大小
    bytesWritten = pWav->onWrite(pWav->pUserData, pData, bytesToWrite);
    pWav->dataChunkDataSize += bytesWritten;

    return bytesWritten;
}

# 以小端格式写入 PCM 帧到 WAV 文件
DRWAV_API drwav_uint64 drwav_write_pcm_frames_le(drwav* pWav, drwav_uint64 framesToWrite, const void* pData)
{
    drwav_uint64 bytesToWrite;
    drwav_uint64 bytesWritten;
    const drwav_uint8* pRunningData;

    # 检查输入参数是否有效
    if (pWav == NULL || framesToWrite == 0 || pData == NULL) {
        return 0;
    }

    # 计算需要写入的字节数
    bytesToWrite = ((framesToWrite * pWav->channels * pWav->bitsPerSample) / 8);
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

        # 调用写入原始数据函数，更新写入字节数和数据指针
        bytesJustWritten = drwav_write_raw(pWav, (size_t)bytesToWriteThisIteration, pRunningData);
        if (bytesJustWritten == 0) {
            break;
        }

        bytesToWrite -= bytesJustWritten;
        bytesWritten += bytesJustWritten;
        pRunningData += bytesJustWritten;
    }

    return (bytesWritten * 8) / pWav->bitsPerSample / pWav->channels;
}

# 以大端格式写入 PCM 帧到 WAV 文件
DRWAV_API drwav_uint64 drwav_write_pcm_frames_be(drwav* pWav, drwav_uint64 framesToWrite, const void* pData)
{
    drwav_uint64 bytesToWrite;
    drwav_uint64 bytesWritten;
    drwav_uint32 bytesPerSample;
    const drwav_uint8* pRunningData;

    # 检查输入参数是否有效
    if (pWav == NULL || framesToWrite == 0 || pData == NULL) {
        return 0;
    }

    # 计算需要写入的字节数
    bytesToWrite = ((framesToWrite * pWav->channels * pWav->bitsPerSample) / 8);
    if (bytesToWrite > DRWAV_SIZE_MAX) {
        return 0;
    }
    # 初始化已写入字节数为0
    bytesWritten = 0;
    # 将pData强制转换为const drwav_uint8指针类型，并赋值给pRunningData
    pRunningData = (const drwav_uint8*)pData;

    # 计算每个采样帧的字节数
    bytesPerSample = drwav_get_bytes_per_pcm_frame(pWav) / pWav->channels;
    
    # 循环直到所有字节写入完成
    while (bytesToWrite > 0) {
        # 创建一个临时缓冲区temp用于存储数据
        drwav_uint8 temp[4096];
        # 用于存储采样帧数量
        drwav_uint32 sampleCount;
        # 用于存储实际写入的字节数
        size_t bytesJustWritten;
        # 用于存储本次迭代需要写入的字节数
        drwav_uint64 bytesToWriteThisIteration;

        # 确保本次迭代需要写入的字节数不超过最大限制
        bytesToWriteThisIteration = bytesToWrite;
        DRWAV_ASSERT(bytesToWriteThisIteration <= DRWAV_SIZE_MAX);  /* <-- This is checked above. */

        '''
        WAV文件总是小端字节序。在大端字节序的架构上，我们需要进行字节交换。由于我们的输入缓冲区是只读的，所以我们需要
        使用一个中间缓冲区进行转换。
        '''
        # 计算每次处理的采样帧数量
        sampleCount = sizeof(temp)/bytesPerSample;

        # 如果本次迭代需要写入的字节数超过了sampleCount*bytesPerSample，则将其限制为sampleCount*bytesPerSample
        if (bytesToWriteThisIteration > ((drwav_uint64)sampleCount)*bytesPerSample) {
            bytesToWriteThisIteration = ((drwav_uint64)sampleCount)*bytesPerSample;
        }

        # 将pRunningData指向的数据拷贝到temp中
        DRWAV_COPY_MEMORY(temp, pRunningData, (size_t)bytesToWriteThisIteration);
        # 对temp中的数据进行字节交换
        drwav__bswap_samples(temp, sampleCount, bytesPerSample, pWav->translatedFormatTag);

        # 将处理后的数据写入到WAV文件中
        bytesJustWritten = drwav_write_raw(pWav, (size_t)bytesToWriteThisIteration, temp);
        # 如果写入的字节数为0，则跳出循环
        if (bytesJustWritten == 0) {
            break;
        }

        # 更新剩余需要写入的字节数、已写入的字节数以及pRunningData指针的位置
        bytesToWrite -= bytesJustWritten;
        bytesWritten += bytesJustWritten;
        pRunningData += bytesJustWritten;
    }

    # 返回写入的总比特数
    return (bytesWritten * 8) / pWav->bitsPerSample / pWav->channels;
}

# 写入 PCM 帧数据
DRWAV_API drwav_uint64 drwav_write_pcm_frames(drwav* pWav, drwav_uint64 framesToWrite, const void* pData)
{
    # 如果是小端序系统
    if (drwav__is_little_endian()) {
        # 调用小端序写入函数
        return drwav_write_pcm_frames_le(pWav, framesToWrite, pData);
    } else {
        # 调用大端序写入函数
        return drwav_write_pcm_frames_be(pWav, framesToWrite, pData);
    }
}

# 读取 PCM 帧数据（MSADPCM 格式）
static drwav_uint64 drwav_read_pcm_frames_s16__msadpcm(drwav* pWav, drwav_uint64 framesToRead, drwav_int16* pBufferOut)
{
    # 初始化已读取帧数
    drwav_uint64 totalFramesRead = 0;

    # 断言 WAV 对象和读取帧数有效
    DRWAV_ASSERT(pWav != NULL);
    DRWAV_ASSERT(framesToRead > 0);

    # 待优化的部分

    }

    # 返回已读取帧数
    return totalFramesRead;
}

# 读取 PCM 帧数据（IMA ADPCM 格式）
static drwav_uint64 drwav_read_pcm_frames_s16__ima(drwav* pWav, drwav_uint64 framesToRead, drwav_int16* pBufferOut)
{
    # 初始化已读取帧数和通道数
    drwav_uint64 totalFramesRead = 0;
    drwav_uint32 iChannel;

    # 静态数组定义
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

    # 断言 WAV 对象和读取帧数有效
    DRWAV_ASSERT(pWav != NULL);
    DRWAV_ASSERT(framesToRead > 0);

    # 待优化的部分

    }

    # 返回已读取帧数
    return totalFramesRead;
}

# 防止重复定义
#ifndef DR_WAV_NO_CONVERSION_API
# 静态数组定义
static unsigned short g_drwavAlawTable[256] = {
    # 定义一系列十六进制数值
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
    # 定义十六进制数值列表
    0x02B0, 0x0290, 0x02F0, 0x02D0, 0x0230, 0x0210, 0x0270, 0x0250, 0x03B0, 0x0390, 0x03F0, 0x03D0, 0x0330, 0x0310, 0x0370, 0x0350
// 静态无符号短整型数组，存储了256个Mu-law编码值
static unsigned short g_drwavMulawTable[256] = {
    // Mu-law编码表，256个Mu-law编码值
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
    # 定义一组十六进制数值，用于后续操作
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

    /* Special case for 8-bit sample data because it's treated as unsigned. */
    if (bytesPerSample == 1) {
        // 将 8 位采样数据转换为有符号 16 位整数
        drwav_u8_to_s16(pOut, pIn, totalSampleCount);
        return;
    }

    /* Slightly more optimal implementation for common formats. */
    if (bytesPerSample == 2) {
        // 将 16 位采样数据转换为有符号 16 位整数
        for (i = 0; i < totalSampleCount; ++i) {
           *pOut++ = ((const drwav_int16*)pIn)[i];
        }
        return;
    }
    if (bytesPerSample == 3) {
        // 将 24 位采样数据转换为有符号 16 位整数
        drwav_s24_to_s16(pOut, pIn, totalSampleCount);
        return;
    }
    if (bytesPerSample == 4) {
        // 将 32 位采样数据转换为有符号 16 位整数
        drwav_s32_to_s16(pOut, (const drwav_int32*)pIn, totalSampleCount);
        return;
    }

    /* Anything more than 64 bits per sample is not supported. */
    if (bytesPerSample > 8) {
        // 不支持超过 64 位的采样数据
        DRWAV_ZERO_MEMORY(pOut, totalSampleCount * sizeof(*pOut));
        return;
    }

    /* Generic, slow converter. */
    // 通用的、较慢的转换器
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
        // 将 32 位浮点数采样数据转换为有符号 16 位整数
        drwav_f32_to_s16(pOut, (const float*)pIn, totalSampleCount);
        return;
    } else if (bytesPerSample == 8) {
        // 如果每个样本占用8个字节，则将输入的双精度浮点数转换为有符号16位整数，并存储到输出缓冲区中
        drwav_f64_to_s16(pOut, (const double*)pIn, totalSampleCount);
        // 返回
        return;
    } else {
        /* 只支持32位和64位浮点数。在所有其他情况下输出静音。欢迎为16位浮点数提供贡献。 */
        // 在所有其他情况下，将输出缓冲区中的数据清零
        DRWAV_ZERO_MEMORY(pOut, totalSampleCount * sizeof(*pOut));
        // 返回
        return;
    }
}

static drwav_uint64 drwav_read_pcm_frames_s16__pcm(drwav* pWav, drwav_uint64 framesToRead, drwav_int16* pBufferOut)
{
    drwav_uint32 bytesPerFrame; // 每帧的字节数
    drwav_uint64 totalFramesRead; // 总共读取的帧数
    drwav_uint8 sampleData[4096]; // 用于存储采样数据的缓冲区

    /* Fast path. */
    // 如果是 PCM 格式且每个采样为 16 位，或者输出缓冲区为空，则直接调用 drwav_read_pcm_frames 函数
    if ((pWav->translatedFormatTag == DR_WAVE_FORMAT_PCM && pWav->bitsPerSample == 16) || pBufferOut == NULL) {
        return drwav_read_pcm_frames(pWav, framesToRead, pBufferOut);
    }
    
    bytesPerFrame = drwav_get_bytes_per_pcm_frame(pWav); // 获取每帧的字节数
    if (bytesPerFrame == 0) {
        return 0;
    }

    totalFramesRead = 0; // 初始化总共读取的帧数为 0
    
    while (framesToRead > 0) {
        // 读取指定数量的 PCM 帧到 sampleData 缓冲区中
        drwav_uint64 framesRead = drwav_read_pcm_frames(pWav, drwav_min(framesToRead, sizeof(sampleData)/bytesPerFrame), sampleData);
        if (framesRead == 0) {
            break;
        }

        // 将 PCM 格式的采样数据转换为 16 位有符号整数格式
        drwav__pcm_to_s16(pBufferOut, sampleData, (size_t)(framesRead*pWav->channels), bytesPerFrame/pWav->channels);

        pBufferOut      += framesRead*pWav->channels; // 更新输出缓冲区指针位置
        framesToRead    -= framesRead; // 更新剩余需要读取的帧数
        totalFramesRead += framesRead; // 更新总共读取的帧数
    }

    return totalFramesRead; // 返回总共读取的帧数
}

static drwav_uint64 drwav_read_pcm_frames_s16__ieee(drwav* pWav, drwav_uint64 framesToRead, drwav_int16* pBufferOut)
{
    drwav_uint64 totalFramesRead; // 总共读取的帧数
    drwav_uint8 sampleData[4096]; // 用于存储采样数据的缓冲区
    drwav_uint32 bytesPerFrame; // 每帧的字节数

    if (pBufferOut == NULL) {
        return drwav_read_pcm_frames(pWav, framesToRead, NULL);
    }

    bytesPerFrame = drwav_get_bytes_per_pcm_frame(pWav); // 获取每帧的字节数
    if (bytesPerFrame == 0) {
        return 0;
    }

    totalFramesRead = 0; // 初始化总共读取的帧数为 0
    # 当还有需要读取的帧数大于0时，继续读取
    while (framesToRead > 0) {
        # 从 WAV 文件中读取指定数量的 PCM 帧到 sampleData 中
        drwav_uint64 framesRead = drwav_read_pcm_frames(pWav, drwav_min(framesToRead, sizeof(sampleData)/bytesPerFrame), sampleData);
        # 如果读取的帧数为0，跳出循环
        if (framesRead == 0) {
            break;
        }

        # 将 IEEE 格式的数据转换为有符号16位整数格式
        drwav__ieee_to_s16(pBufferOut, sampleData, (size_t)(framesRead*pWav->channels), bytesPerFrame/pWav->channels);

        # 更新输出缓冲区指针位置
        pBufferOut      += framesRead*pWav->channels;
        # 更新剩余需要读取的帧数
        framesToRead    -= framesRead;
        # 更新总共读取的帧数
        totalFramesRead += framesRead;
    }

    # 返回总共读取的帧数
    return totalFramesRead;
# 读取 ALAW 格式的 PCM 数据并转换为有符号 16 位整数格式
static drwav_uint64 drwav_read_pcm_frames_s16__alaw(drwav* pWav, drwav_uint64 framesToRead, drwav_int16* pBufferOut)
{
    # 总共读取的帧数
    drwav_uint64 totalFramesRead;
    # 存储样本数据的缓冲区
    drwav_uint8 sampleData[4096];
    # 每帧的字节数
    drwav_uint32 bytesPerFrame;

    # 如果输出缓冲区为空，则直接调用 drwav_read_pcm_frames 函数
    if (pBufferOut == NULL) {
        return drwav_read_pcm_frames(pWav, framesToRead, NULL);
    }

    # 获取每帧的字节数
    bytesPerFrame = drwav_get_bytes_per_pcm_frame(pWav);
    # 如果每帧字节数为 0，则返回 0
    if (bytesPerFrame == 0) {
        return 0;
    }

    # 初始化总帧数为 0
    totalFramesRead = 0;
    
    # 循环读取帧数据
    while (framesToRead > 0) {
        # 读取帧数据
        drwav_uint64 framesRead = drwav_read_pcm_frames(pWav, drwav_min(framesToRead, sizeof(sampleData)/bytesPerFrame), sampleData);
        # 如果读取的帧数为 0，则跳出循环
        if (framesRead == 0) {
            break;
        }

        # 将 ALAW 格式数据转换为有符号 16 位整数格式
        drwav_alaw_to_s16(pBufferOut, sampleData, (size_t)(framesRead*pWav->channels));

        # 更新输出缓冲区指针、剩余帧数和总帧数
        pBufferOut      += framesRead*pWav->channels;
        framesToRead    -= framesRead;
        totalFramesRead += framesRead;
    }

    # 返回总帧数
    return totalFramesRead;
}

# 读取 MULAW 格式的 PCM 数据并转换为有符号 16 位整数格式
static drwav_uint64 drwav_read_pcm_frames_s16__mulaw(drwav* pWav, drwav_uint64 framesToRead, drwav_int16* pBufferOut)
{
    # 总共读取的帧数
    drwav_uint64 totalFramesRead;
    # 存储样本数据的缓冲区
    drwav_uint8 sampleData[4096];
    # 每帧的字节数
    drwav_uint32 bytesPerFrame;

    # 如果输出缓冲区为空，则直接调用 drwav_read_pcm_frames 函数
    if (pBufferOut == NULL) {
        return drwav_read_pcm_frames(pWav, framesToRead, NULL);
    }

    # 获取每帧的字节数
    bytesPerFrame = drwav_get_bytes_per_pcm_frame(pWav);
    # 如果每帧字节数为 0，则返回 0
    if (bytesPerFrame == 0) {
        return 0;
    }

    # 初始化总帧数为 0
    totalFramesRead = 0;

    # 循环读取帧数据
    while (framesToRead > 0) {
        # 读取帧数据
        drwav_uint64 framesRead = drwav_read_pcm_frames(pWav, drwav_min(framesToRead, sizeof(sampleData)/bytesPerFrame), sampleData);
        # 如果读取的帧数为 0，则跳出循环
        if (framesRead == 0) {
            break;
        }

        # 将 MULAW 格式数据转换为有符号 16 位整数格式
        drwav_mulaw_to_s16(pBufferOut, sampleData, (size_t)(framesRead*pWav->channels));

        # 更新输出缓冲区指针、剩余帧数和总帧数
        pBufferOut      += framesRead*pWav->channels;
        framesToRead    -= framesRead;
        totalFramesRead += framesRead;
    }

    # 返回总帧数
    return totalFramesRead;
}
# 读取指定数量的帧数据，并将其转换为有符号16位整数格式
DRWAV_API drwav_uint64 drwav_read_pcm_frames_s16(drwav* pWav, drwav_uint64 framesToRead, drwav_int16* pBufferOut)
{
    # 如果输入的 WAV 文件指针为空或者要读取的帧数为0，则返回0
    if (pWav == NULL || framesToRead == 0) {
        return 0;
    }

    # 如果输出缓冲区为空，则调用 drwav_read_pcm_frames 函数
    if (pBufferOut == NULL) {
        return drwav_read_pcm_frames(pWav, framesToRead, NULL);
    }

    /* 不要尝试读取超过输出缓冲区可能容纳的样本数 */
    if (framesToRead * pWav->channels * sizeof(drwav_int16) > DRWAV_SIZE_MAX) {
        framesToRead = DRWAV_SIZE_MAX / sizeof(drwav_int16) / pWav->channels;
    }

    # 根据 WAV 文件的格式标签选择相应的读取函数
    if (pWav->translatedFormatTag == DR_WAVE_FORMAT_PCM) {
        return drwav_read_pcm_frames_s16__pcm(pWav, framesToRead, pBufferOut);
    }

    if (pWav->translatedFormatTag == DR_WAVE_FORMAT_IEEE_FLOAT) {
        return drwav_read_pcm_frames_s16__ieee(pWav, framesToRead, pBufferOut);
    }

    if (pWav->translatedFormatTag == DR_WAVE_FORMAT_ALAW) {
        return drwav_read_pcm_frames_s16__alaw(pWav, framesToRead, pBufferOut);
    }

    if (pWav->translatedFormatTag == DR_WAVE_FORMAT_MULAW) {
        return drwav_read_pcm_frames_s16__mulaw(pWav, framesToRead, pBufferOut);
    }

    if (pWav->translatedFormatTag == DR_WAVE_FORMAT_ADPCM) {
        return drwav_read_pcm_frames_s16__msadpcm(pWav, framesToRead, pBufferOut);
    }

    if (pWav->translatedFormatTag == DR_WAVE_FORMAT_DVI_ADPCM) {
        return drwav_read_pcm_frames_s16__ima(pWav, framesToRead, pBufferOut);
    }

    # 默认情况下返回0
    return 0;
}

# 以小端序读取有符号16位整数格式的帧数据
DRWAV_API drwav_uint64 drwav_read_pcm_frames_s16le(drwav* pWav, drwav_uint64 framesToRead, drwav_int16* pBufferOut)
{
    # 调用 drwav_read_pcm_frames_s16 函数读取帧数据
    drwav_uint64 framesRead = drwav_read_pcm_frames_s16(pWav, framesToRead, pBufferOut);
    # 如果输出缓冲区不为空且系统为小端序，则交换样本字节序
    if (pBufferOut != NULL && drwav__is_little_endian() == DRWAV_FALSE) {
        drwav__bswap_samples_s16(pBufferOut, framesRead*pWav->channels);
    }

    return framesRead;
}

# 以大端序读取有符号16位整数格式的帧数据
DRWAV_API drwav_uint64 drwav_read_pcm_frames_s16be(drwav* pWav, drwav_uint64 framesToRead, drwav_int16* pBufferOut)
{
    // 从 WAV 文件中读取指定数量的 PCM 帧到输出缓冲区中，并返回实际读取的帧数
    drwav_uint64 framesRead = drwav_read_pcm_frames_s16(pWav, framesToRead, pBufferOut);
    // 检查输出缓冲区是否存在且系统是否为小端序
    if (pBufferOut != NULL && drwav__is_little_endian() == DRWAV_TRUE) {
        // 如果是小端序，则交换输出缓冲区中的 16 位有符号整数样本字节序
        drwav__bswap_samples_s16(pBufferOut, framesRead*pWav->channels);
    }

    // 返回实际读取的帧数
    return framesRead;
}

// 将无符号8位整数数组转换为有符号16位整数数组
DRWAV_API void drwav_u8_to_s16(drwav_int16* pOut, const drwav_uint8* pIn, size_t sampleCount)
{
    int r; // 用于存储中间结果的变量
    size_t i; // 循环计数器
    for (i = 0; i < sampleCount; ++i) {
        int x = pIn[i]; // 获取输入数组中的值
        r = x << 8; // 左移8位
        r = r - 32768; // 减去32768
        pOut[i] = (short)r; // 将结果存入输出数组
    }
}

// 将有符号24位整数数组转换为有符号16位整数数组
DRWAV_API void drwav_s24_to_s16(drwav_int16* pOut, const drwav_uint8* pIn, size_t sampleCount)
{
    int r; // 用于存储中间结果的变量
    size_t i; // 循环计数器
    for (i = 0; i < sampleCount; ++i) {
        int x = ((int)(((unsigned int)(((const drwav_uint8*)pIn)[i*3+0]) << 8) | ((unsigned int)(((const drwav_uint8*)pIn)[i*3+1]) << 16) | ((unsigned int)(((const drwav_uint8*)pIn)[i*3+2])) << 24)) >> 8; // 复杂的位运算
        r = x >> 8; // 右移8位
        pOut[i] = (short)r; // 将结果存入输出数组
    }
}

// 将有符号32位整数数组转换为有符号16位整数数组
DRWAV_API void drwav_s32_to_s16(drwav_int16* pOut, const drwav_int32* pIn, size_t sampleCount)
{
    int r; // 用于存储中间结果的变量
    size_t i; // 循环计数器
    for (i = 0; i < sampleCount; ++i) {
        int x = pIn[i]; // 获取输入数组中的值
        r = x >> 16; // 右移16位
        pOut[i] = (short)r; // 将结果存入输出数组
    }
}

// 将单精度浮点数数组转换为有符号16位整数数组
DRWAV_API void drwav_f32_to_s16(drwav_int16* pOut, const float* pIn, size_t sampleCount)
{
    int r; // 用于存储中间结果的变量
    size_t i; // 循环计数器
    for (i = 0; i < sampleCount; ++i) {
        float x = pIn[i]; // 获取输入数组中的值
        float c; // 用于存储中间结果的变量
        c = ((x < -1) ? -1 : ((x > 1) ? 1 : x)); // 判断条件
        c = c + 1; // 加1
        r = (int)(c * 32767.5f); // 浮点数转整数
        r = r - 32768; // 减去32768
        pOut[i] = (short)r; // 将结果存入输出数组
    }
}

// 将双精度浮点数数组转换为有符号16位整数数组
DRWAV_API void drwav_f64_to_s16(drwav_int16* pOut, const double* pIn, size_t sampleCount)
{
    int r; // 用于存储中间结果的变量
    size_t i; // 循环计数器
    for (i = 0; i < sampleCount; ++i) {
        double x = pIn[i]; // 获取输入数组中的值
        double c; // 用于存储中间结果的变量
        c = ((x < -1) ? -1 : ((x > 1) ? 1 : x)); // 判断条件
        c = c + 1; // 加1
        r = (int)(c * 32767.5); // 浮点数转整数
        r = r - 32768; // 减去32768
        pOut[i] = (short)r; // 将结果存入输出数组
    }
}

// 将a-law编码数组转换为有符号16位整数数组
DRWAV_API void drwav_alaw_to_s16(drwav_int16* pOut, const drwav_uint8* pIn, size_t sampleCount)
{
    size_t i; // 循环计数器
    for (i = 0; i < sampleCount; ++i) {
        pOut[i] = drwav__alaw_to_s16(pIn[i]); // 调用alaw转换函数
    }
}

// 将u-law编码数组转换为有符号16位整数数组
DRWAV_API void drwav_mulaw_to_s16(drwav_int16* pOut, const drwav_uint8* pIn, size_t sampleCount)
{
    size_t i; // 循环计数器
    # 遍历从 0 到 sampleCount 的索引
    for (i = 0; i < sampleCount; ++i) {
        # 调用 drwav__mulaw_to_s16 函数将 pIn[i] 的 mu-law 编码转换为有符号 16 位整数，并将结果存储到 pOut[i] 中
        pOut[i] = drwav__mulaw_to_s16(pIn[i]);
    }
# 将 PCM 格式的音频数据转换为 32 位浮点数格式
static void drwav__pcm_to_f32(float* pOut, const drwav_uint8* pIn, size_t sampleCount, unsigned int bytesPerSample)
{
    unsigned int i;

    # 对于 8 位样本数据，因为它被视为无符号数，所以有特殊处理
    if (bytesPerSample == 1) {
        drwav_u8_to_f32(pOut, pIn, sampleCount);
        return;
    }

    # 对于常见格式，有稍微更优化的实现
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

    # 不支持超过 64 位每个样本的数据
    if (bytesPerSample > 8) {
        DRWAV_ZERO_MEMORY(pOut, sampleCount * sizeof(*pOut));
        return;
    }

    # 通用、较慢的转换器
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

# 将 IEEE 格式的音频数据转换为 32 位浮点数格式
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
        /* 只支持32位和64位浮点数。在其他情况下输出静音。欢迎为16位浮点数做出贡献。 */
        // 将输出缓冲区的内存清零，大小为样本数乘以每个样本的大小
        DRWAV_ZERO_MEMORY(pOut, sampleCount * sizeof(*pOut));
        // 返回
        return;
    }
static drwav_uint64 drwav_read_pcm_frames_f32__pcm(drwav* pWav, drwav_uint64 framesToRead, float* pBufferOut)
{
    // 定义变量用于存储总共读取的帧数
    drwav_uint64 totalFramesRead;
    // 定义一个用于存储采样数据的数组
    drwav_uint8 sampleData[4096];

    // 获取每帧的字节数
    drwav_uint32 bytesPerFrame = drwav_get_bytes_per_pcm_frame(pWav);
    // 如果每帧字节数为0，则返回0
    if (bytesPerFrame == 0) {
        return 0;
    }

    // 初始化总帧数为0
    totalFramesRead = 0;

    // 循环读取帧数据
    while (framesToRead > 0) {
        // 读取PCM帧数据
        drwav_uint64 framesRead = drwav_read_pcm_frames(pWav, drwav_min(framesToRead, sizeof(sampleData)/bytesPerFrame), sampleData);
        // 如果读取的帧数为0，则跳出循环
        if (framesRead == 0) {
            break;
        }

        // 将PCM数据转换为32位浮点数
        drwav__pcm_to_f32(pBufferOut, sampleData, (size_t)framesRead*pWav->channels, bytesPerFrame/pWav->channels);

        // 更新指针位置和剩余帧数
        pBufferOut      += framesRead*pWav->channels;
        framesToRead    -= framesRead;
        totalFramesRead += framesRead;
    }

    return totalFramesRead;
}

static drwav_uint64 drwav_read_pcm_frames_f32__msadpcm(drwav* pWav, drwav_uint64 framesToRead, float* pBufferOut)
{
    /*
    We're just going to borrow the implementation from the drwav_read_s16() since ADPCM is a little bit more complicated than other formats and I don't
    want to duplicate that code.
    */
    // 初始化总帧数为0
    drwav_uint64 totalFramesRead = 0;
    // 定义一个用于存储16位样本数据的数组
    drwav_int16 samples16[2048];
    // 循环读取ADPCM帧数据
    while (framesToRead > 0) {
        // 读取ADPCM帧数据
        drwav_uint64 framesRead = drwav_read_pcm_frames_s16(pWav, drwav_min(framesToRead, drwav_countof(samples16)/pWav->channels), samples16);
        // 如果读取的帧数为0，则跳出循环
        if (framesRead == 0) {
            break;
        }

        // 将16位样本数据转换为32位浮点数
        drwav_s16_to_f32(pBufferOut, samples16, (size_t)(framesRead*pWav->channels));   /* <-- Safe cast because we're clamping to 2048. */

        // 更新指针位置和剩余帧数
        pBufferOut      += framesRead*pWav->channels;
        framesToRead    -= framesRead;
        totalFramesRead += framesRead;
    }

    return totalFramesRead;
}

static drwav_uint64 drwav_read_pcm_frames_f32__ima(drwav* pWav, drwav_uint64 framesToRead, float* pBufferOut)
{
    /*
    /*
    我们将从 drwav_read_s16() 的实现中借用代码，因为IMA-ADPCM比其他格式复杂一些，我不想重复编写那些代码。
    */
    // 初始化已读取的总帧数
    drwav_uint64 totalFramesRead = 0;
    // 创建一个用于存储读取的 int16 样本的数组
    drwav_int16 samples16[2048];
    // 循环读取 PCM 帧数据
    while (framesToRead > 0) {
        // 读取 PCM 帧数据到 samples16 数组中
        drwav_uint64 framesRead = drwav_read_pcm_frames_s16(pWav, drwav_min(framesToRead, drwav_countof(samples16)/pWav->channels), samples16);
        // 如果没有读取到帧数据，则跳出循环
        if (framesRead == 0) {
            break;
        }

        // 将 int16 样本转换为 float32 样本
        drwav_s16_to_f32(pBufferOut, samples16, (size_t)(framesRead*pWav->channels));   /* <-- Safe cast because we're clamping to 2048. */

        // 更新输出缓冲区指针和剩余待读取帧数
        pBufferOut      += framesRead*pWav->channels;
        framesToRead    -= framesRead;
        // 更新已读取的总帧数
        totalFramesRead += framesRead;
    }

    // 返回已读取的总帧数
    return totalFramesRead;
}

static drwav_uint64 drwav_read_pcm_frames_f32__ieee(drwav* pWav, drwav_uint64 framesToRead, float* pBufferOut)
{
    drwav_uint64 totalFramesRead; // 用于记录总共读取的帧数
    drwav_uint8 sampleData[4096]; // 存储采样数据的缓冲区
    drwav_uint32 bytesPerFrame; // 每帧的字节数

    /* Fast path. */
    // 如果是 IEEE_FLOAT 格式且每个采样为 32 位，则直接调用 drwav_read_pcm_frames 函数
    if (pWav->translatedFormatTag == DR_WAVE_FORMAT_IEEE_FLOAT && pWav->bitsPerSample == 32) {
        return drwav_read_pcm_frames(pWav, framesToRead, pBufferOut);
    }
    
    bytesPerFrame = drwav_get_bytes_per_pcm_frame(pWav); // 获取每帧的字节数
    if (bytesPerFrame == 0) {
        return 0;
    }

    totalFramesRead = 0;

    while (framesToRead > 0) {
        // 读取指定数量的帧数到 sampleData 缓冲区中
        drwav_uint64 framesRead = drwav_read_pcm_frames(pWav, drwav_min(framesToRead, sizeof(sampleData)/bytesPerFrame), sampleData);
        if (framesRead == 0) {
            break;
        }

        // 将采样数据从 sampleData 转换为 IEEE_FLOAT 格式写入 pBufferOut 中
        drwav__ieee_to_f32(pBufferOut, sampleData, (size_t)(framesRead*pWav->channels), bytesPerFrame/pWav->channels);

        pBufferOut      += framesRead*pWav->channels; // 更新写入位置
        framesToRead    -= framesRead; // 更新剩余帧数
        totalFramesRead += framesRead; // 更新总共读取的帧数
    }

    return totalFramesRead;
}

static drwav_uint64 drwav_read_pcm_frames_f32__alaw(drwav* pWav, drwav_uint64 framesToRead, float* pBufferOut)
{
    drwav_uint64 totalFramesRead; // 用于记录总共读取的帧数
    drwav_uint8 sampleData[4096]; // 存储采样数据的缓冲区
    drwav_uint32 bytesPerFrame = drwav_get_bytes_per_pcm_frame(pWav); // 获取每帧的字节数
    if (bytesPerFrame == 0) {
        return 0;
    }

    totalFramesRead = 0;

    while (framesToRead > 0) {
        // 读取指定数量的帧数到 sampleData 缓冲区中
        drwav_uint64 framesRead = drwav_read_pcm_frames(pWav, drwav_min(framesToRead, sizeof(sampleData)/bytesPerFrame), sampleData);
        if (framesRead == 0) {
            break;
        }

        // 将采样数据从 sampleData 转换为 A-law 格式写入 pBufferOut 中
        drwav_alaw_to_f32(pBufferOut, sampleData, (size_t)(framesRead*pWav->channels));

        pBufferOut      += framesRead*pWav->channels; // 更新写入位置
        framesToRead    -= framesRead; // 更新剩余帧数
        totalFramesRead += framesRead; // 更新总共读取的帧数
    }

    return totalFramesRead;
}

static drwav_uint64 drwav_read_pcm_frames_f32__mulaw(drwav* pWav, drwav_uint64 framesToRead, float* pBufferOut)
{
    # 定义变量 totalFramesRead，用于记录总共读取的帧数
    drwav_uint64 totalFramesRead;
    # 定义数组 sampleData，用于存储采样数据
    drwav_uint8 sampleData[4096];

    # 获取每个 PCM 帧的字节数
    drwav_uint32 bytesPerFrame = drwav_get_bytes_per_pcm_frame(pWav);
    # 如果每个 PCM 帧的字节数为 0，则返回 0
    if (bytesPerFrame == 0) {
        return 0;
    }

    # 初始化 totalFramesRead 为 0
    totalFramesRead = 0;

    # 循环读取 PCM 帧数据，直到 framesToRead 为 0
    while (framesToRead > 0) {
        # 读取 PCM 帧数据，最多读取 sampleData 数组大小的数据
        drwav_uint64 framesRead = drwav_read_pcm_frames(pWav, drwav_min(framesToRead, sizeof(sampleData)/bytesPerFrame), sampleData);
        # 如果没有读取到任何帧，则跳出循环
        if (framesRead == 0) {
            break;
        }

        # 将 mu-law 编码转换为浮点数，并存储到 pBufferOut 中
        drwav_mulaw_to_f32(pBufferOut, sampleData, (size_t)(framesRead*pWav->channels));

        # 更新 pBufferOut 指针位置
        pBufferOut      += framesRead*pWav->channels;
        # 更新 framesToRead 剩余帧数
        framesToRead    -= framesRead;
        # 更新 totalFramesRead 总共读取的帧数
        totalFramesRead += framesRead;
    }

    # 返回总共读取的帧数
    return totalFramesRead;
}

// 读取指定数量的 32 位浮点 PCM 样本帧
DRWAV_API drwav_uint64 drwav_read_pcm_frames_f32(drwav* pWav, drwav_uint64 framesToRead, float* pBufferOut)
{
    // 如果 WAV 文件指针为空或者要读取的帧数为 0，则返回 0
    if (pWav == NULL || framesToRead == 0) {
        return 0;
    }

    // 如果输出缓冲区为空，则调用 drwav_read_pcm_frames 函数
    if (pBufferOut == NULL) {
        return drwav_read_pcm_frames(pWav, framesToRead, NULL);
    }

    /* 不要尝试读取超过输出缓冲区可能容纳的样本数 */
    if (framesToRead * pWav->channels * sizeof(float) > DRWAV_SIZE_MAX) {
        framesToRead = DRWAV_SIZE_MAX / sizeof(float) / pWav->channels;
    }

    // 根据 WAV 文件的格式标签调用相应的读取函数
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

// 读取指定数量的 32 位浮点 PCM 样本帧（小端序）
DRWAV_API drwav_uint64 drwav_read_pcm_frames_f32le(drwav* pWav, drwav_uint64 framesToRead, float* pBufferOut)
{
    // 调用 drwav_read_pcm_frames_f32 函数读取样本帧
    drwav_uint64 framesRead = drwav_read_pcm_frames_f32(pWav, framesToRead, pBufferOut);
    // 如果输出缓冲区不为空且系统不是小端序，则交换字节序
    if (pBufferOut != NULL && drwav__is_little_endian() == DRWAV_FALSE) {
        drwav__bswap_samples_f32(pBufferOut, framesRead*pWav->channels);
    }

    return framesRead;
}

DRWAV_API drwav_uint64 drwav_read_pcm_frames_f32be(drwav* pWav, drwav_uint64 framesToRead, float* pBufferOut)
{
    // 从 WAV 文件中读取指定数量的 PCM 帧到输出缓冲区中，并返回实际读取的帧数
    drwav_uint64 framesRead = drwav_read_pcm_frames_f32(pWav, framesToRead, pBufferOut);
    // 检查输出缓冲区是否不为空，并且系统是小端序
    if (pBufferOut != NULL && drwav__is_little_endian() == DRWAV_TRUE) {
        // 如果满足条件，交换输出缓冲区中的浮点数样本的字节序
        drwav__bswap_samples_f32(pBufferOut, framesRead*pWav->channels);
    }

    // 返回实际读取的帧数
    return framesRead;
}



DRWAV_API void drwav_u8_to_f32(float* pOut, const drwav_uint8* pIn, size_t sampleCount)
{
    size_t i;

    // 检查输入指针是否为空，如果为空则返回
    if (pOut == NULL || pIn == NULL) {
        return;
    }

#ifdef DR_WAV_LIBSNDFILE_COMPAT
    /*
    libsndfile 使用不同的逻辑进行 u8 -> f32 转换，与 dr_wav 稍有不同，我认为是不正确的。看起来
    libsndfile 执行的转换类似于 "f32 = (u8 / 256) * 2 - 1"，但我认为应该是 "f32 = (u8 / 255) * 2 - 1"（注意
    256 和 255 的除数不同）。我将 libsndfile 用作测试的基准，因此我将这个块留在这里仅用于自动化
    的正确性测试。默认情况下禁用此功能。
    */
    for (i = 0; i < sampleCount; ++i) {
        *pOut++ = (pIn[i] / 256.0f) * 2 - 1;
    }
#else
    // 使用 dr_wav 的逻辑进行 u8 -> f32 转换
    for (i = 0; i < sampleCount; ++i) {
        float x = pIn[i];
        x = x * 0.00784313725490196078f;    // 0..255 转换为 0..2
        x = x - 1;                          // 0..2 转换为 -1..1

        *pOut++ = x;
    }
#endif
}

DRWAV_API void drwav_s16_to_f32(float* pOut, const drwav_int16* pIn, size_t sampleCount)
{
    size_t i;

    // 检查输入指针是否为空，如果为空则返回
    if (pOut == NULL || pIn == NULL) {
        return;
    }

    // 将有符号 16 位整数转换为 32 位浮点数
    for (i = 0; i < sampleCount; ++i) {
        *pOut++ = pIn[i] * 0.000030517578125f;
    }
}

DRWAV_API void drwav_s24_to_f32(float* pOut, const drwav_uint8* pIn, size_t sampleCount)
{
    size_t i;

    // 检查输入指针是否为空，如果为空则返回
    if (pOut == NULL || pIn == NULL) {
        return;
    }

    // 将有符号 24 位整数转换为 32 位浮点数
    for (i = 0; i < sampleCount; ++i) {
        double x;
        drwav_uint32 a = ((drwav_uint32)(pIn[i*3+0]) <<  8);
        drwav_uint32 b = ((drwav_uint32)(pIn[i*3+1]) << 16);
        drwav_uint32 c = ((drwav_uint32)(pIn[i*3+2]) << 24);

        x = (double)((drwav_int32)(a | b | c) >> 8);
        *pOut++ = (float)(x * 0.00000011920928955078125);
    }
}

DRWAV_API void drwav_s32_to_f32(float* pOut, const drwav_int32* pIn, size_t sampleCount)
{
    size_t i;
    // 检查输入指针是否为空，如果为空则返回
    if (pOut == NULL || pIn == NULL) {
        return;
    }
}
    # 遍历从 0 到 sampleCount 的索引
    for (i = 0; i < sampleCount; ++i) {
        # 将 pIn[i] 的值除以 2147483648.0 转换为浮点数，存入 pOut 中，并移动指针到下一个位置
        *pOut++ = (float)(pIn[i] / 2147483648.0);
    }
}

// 将双精度浮点数组转换为单精度浮点数组
DRWAV_API void drwav_f64_to_f32(float* pOut, const double* pIn, size_t sampleCount)
{
    size_t i;

    // 检查输入指针是否为空
    if (pOut == NULL || pIn == NULL) {
        return;
    }

    // 遍历样本并将双精度浮点数转换为单精度浮点数
    for (i = 0; i < sampleCount; ++i) {
        *pOut++ = (float)pIn[i];
    }
}

// 将 A-law 格式数据转换为单精度浮点数组
DRWAV_API void drwav_alaw_to_f32(float* pOut, const drwav_uint8* pIn, size_t sampleCount)
{
    size_t i;

    // 检查输入指针是否为空
    if (pOut == NULL || pIn == NULL) {
        return;
    }

    // 遍历样本并将 A-law 格式数据转换为单精度浮点数
    for (i = 0; i < sampleCount; ++i) {
        *pOut++ = drwav__alaw_to_s16(pIn[i]) / 32768.0f;
    }
}

// 将 mu-law 格式数据转换为单精度浮点数组
DRWAV_API void drwav_mulaw_to_f32(float* pOut, const drwav_uint8* pIn, size_t sampleCount)
{
    size_t i;

    // 检查输入指针是否为空
    if (pOut == NULL || pIn == NULL) {
        return;
    }

    // 遍历样本并将 mu-law 格式数据转换为单精度浮点数
    for (i = 0; i < sampleCount; ++i) {
        *pOut++ = drwav__mulaw_to_s16(pIn[i]) / 32768.0f;
    }
}

// 将 PCM 格式数据转换为 32 位有符号整数数组
static void drwav__pcm_to_s32(drwav_int32* pOut, const drwav_uint8* pIn, size_t totalSampleCount, unsigned int bytesPerSample)
{
    unsigned int i;

    /* 对于 8 位样本数据，因为它被视为无符号数，所以有特殊处理。 */
    if (bytesPerSample == 1) {
        drwav_u8_to_s32(pOut, pIn, totalSampleCount);
        return;
    }

    /* 对于常见格式，有稍微更优化的实现。 */
    if (bytesPerSample == 2) {
        drwav_s16_to_s32(pOut, (const drwav_int16*)pIn, totalSampleCount);
        return;
    }
    if (bytesPerSample == 3) {
        drwav_s24_to_s32(pOut, pIn, totalSampleCount);
        return;
    }
    if (bytesPerSample == 4) {
        for (i = 0; i < totalSampleCount; ++i) {
           *pOut++ = ((const drwav_int32*)pIn)[i];
        }
        return;
    }

    /* 不支持超过 64 位每样本的数据。 */
    if (bytesPerSample > 8) {
        DRWAV_ZERO_MEMORY(pOut, totalSampleCount * sizeof(*pOut));
        return;
    }

    /* 通用、较慢的转换器。 */
    # 遍历每个采样点
    for (i = 0; i < totalSampleCount; ++i) {
        # 初始化采样值为0，shift为字节偏移量
        drwav_uint64 sample = 0;
        unsigned int shift  = (8 - bytesPerSample) * 8;

        # 遍历每个采样点的字节
        unsigned int j;
        for (j = 0; j < bytesPerSample; j += 1) {
            # 断言字节偏移量小于8
            DRWAV_ASSERT(j < 8);
            # 将字节数据转换为采样值
            sample |= (drwav_uint64)(pIn[j]) << shift;
            shift  += 8;
        }

        # 更新输入指针位置
        pIn += j;
        # 将采样值右移32位后转换为32位有符号整数，存入输出数组
        *pOut++ = (drwav_int32)((drwav_int64)sample >> 32);
    }
}

static void drwav__ieee_to_s32(drwav_int32* pOut, const drwav_uint8* pIn, size_t totalSampleCount, unsigned int bytesPerSample)
{
    // 如果字节大小为4，则将输入数据转换为32位浮点数
    if (bytesPerSample == 4) {
        drwav_f32_to_s32(pOut, (const float*)pIn, totalSampleCount);
        return;
    } 
    // 如果字节大小为8，则将输入数据转换为64位浮点数
    else if (bytesPerSample == 8) {
        drwav_f64_to_s32(pOut, (const double*)pIn, totalSampleCount);
        return;
    } 
    // 其他情况下输出静音
    else {
        /* Only supporting 32- and 64-bit float. Output silence in all other cases. Contributions welcome for 16-bit float. */
        DRWAV_ZERO_MEMORY(pOut, totalSampleCount * sizeof(*pOut));
        return;
    }
}


static drwav_uint64 drwav_read_pcm_frames_s32__pcm(drwav* pWav, drwav_uint64 framesToRead, drwav_int32* pBufferOut)
{
    drwav_uint64 totalFramesRead;
    drwav_uint8 sampleData[4096];
    drwav_uint32 bytesPerFrame;

    /* Fast path. */
    // 如果是PCM格式且每个样本为32位，则直接调用drwav_read_pcm_frames函数
    if (pWav->translatedFormatTag == DR_WAVE_FORMAT_PCM && pWav->bitsPerSample == 32) {
        return drwav_read_pcm_frames(pWav, framesToRead, pBufferOut);
    }
    
    bytesPerFrame = drwav_get_bytes_per_pcm_frame(pWav);
    if (bytesPerFrame == 0) {
        return 0;
    }

    totalFramesRead = 0;

    while (framesToRead > 0) {
        // 读取PCM帧数据
        drwav_uint64 framesRead = drwav_read_pcm_frames(pWav, drwav_min(framesToRead, sizeof(sampleData)/bytesPerFrame), sampleData);
        if (framesRead == 0) {
            break;
        }

        // 将PCM数据转换为32位有符号整数
        drwav__pcm_to_s32(pBufferOut, sampleData, (size_t)(framesRead*pWav->channels), bytesPerFrame/pWav->channels);

        pBufferOut      += framesRead*pWav->channels;
        framesToRead    -= framesRead;
        totalFramesRead += framesRead;
    }

    return totalFramesRead;
}

static drwav_uint64 drwav_read_pcm_frames_s32__msadpcm(drwav* pWav, drwav_uint64 framesToRead, drwav_int32* pBufferOut)
{
    /*
    We're just going to borrow the implementation from the drwav_read_s16() since ADPCM is a little bit more complicated than other formats and I don't
    want to duplicate that code.
    */
    // 初始化已读取的总帧数为0
    drwav_uint64 totalFramesRead = 0;
    // 创建一个长度为2048的int16数组用于存储采样数据
    drwav_int16 samples16[2048];
    // 当还有待读取的帧数时循环读取数据
    while (framesToRead > 0) {
        // 读取PCM帧数据到samples16数组中
        drwav_uint64 framesRead = drwav_read_pcm_frames_s16(pWav, drwav_min(framesToRead, drwav_countof(samples16)/pWav->channels), samples16);
        // 如果没有读取到数据则跳出循环
        if (framesRead == 0) {
            break;
        }

        // 将int16类型的采样数据转换为int32类型，安全转换因为我们限制了最大长度为2048
        drwav_s16_to_s32(pBufferOut, samples16, (size_t)(framesRead*pWav->channels));   /* <-- Safe cast because we're clamping to 2048. */

        // 更新输出缓冲区指针和待读取帧数
        pBufferOut      += framesRead*pWav->channels;
        framesToRead    -= framesRead;
        // 更新已读取的总帧数
        totalFramesRead += framesRead;
    }

    // 返回已读取的总帧数
    return totalFramesRead;
}

static drwav_uint64 drwav_read_pcm_frames_s32__ima(drwav* pWav, drwav_uint64 framesToRead, drwav_int32* pBufferOut)
{
    /*
    从 drwav_read_s16() 中借用实现，因为 IMA-ADPCM 比其他格式稍微复杂一些，我不想重复编写那段代码。
    */
    drwav_uint64 totalFramesRead = 0;
    drwav_int16 samples16[2048];
    while (framesToRead > 0) {
        drwav_uint64 framesRead = drwav_read_pcm_frames_s16(pWav, drwav_min(framesToRead, drwav_countof(samples16)/pWav->channels), samples16);
        if (framesRead == 0) {
            break;
        }

        drwav_s16_to_s32(pBufferOut, samples16, (size_t)(framesRead*pWav->channels));   /* <-- 安全的类型转换，因为我们限制在 2048 内。 */

        pBufferOut      += framesRead*pWav->channels;
        framesToRead    -= framesRead;
        totalFramesRead += framesRead;
    }

    return totalFramesRead;
}

static drwav_uint64 drwav_read_pcm_frames_s32__ieee(drwav* pWav, drwav_uint64 framesToRead, drwav_int32* pBufferOut)
{
    drwav_uint64 totalFramesRead;
    drwav_uint8 sampleData[4096];

    drwav_uint32 bytesPerFrame = drwav_get_bytes_per_pcm_frame(pWav);
    if (bytesPerFrame == 0) {
        return 0;
    }

    totalFramesRead = 0;

    while (framesToRead > 0) {
        drwav_uint64 framesRead = drwav_read_pcm_frames(pWav, drwav_min(framesToRead, sizeof(sampleData)/bytesPerFrame), sampleData);
        if (framesRead == 0) {
            break;
        }

        drwav__ieee_to_s32(pBufferOut, sampleData, (size_t)(framesRead*pWav->channels), bytesPerFrame/pWav->channels);

        pBufferOut      += framesRead*pWav->channels;
        framesToRead    -= framesRead;
        totalFramesRead += framesRead;
    }

    return totalFramesRead;
}

static drwav_uint64 drwav_read_pcm_frames_s32__alaw(drwav* pWav, drwav_uint64 framesToRead, drwav_int32* pBufferOut)
{
    drwav_uint64 totalFramesRead;
    drwav_uint8 sampleData[4096];
    # 获取每个 PCM 帧的字节数
    drwav_uint32 bytesPerFrame = drwav_get_bytes_per_pcm_frame(pWav);
    # 如果每个 PCM 帧的字节数为 0，则返回 0
    if (bytesPerFrame == 0) {
        return 0;
    }

    # 初始化已读取的总帧数为 0
    totalFramesRead = 0;

    # 当需要读取的帧数大于 0 时，进入循环
    while (framesToRead > 0) {
        # 读取 PCM 帧数据，最多读取 sampleData 数组大小的帧数
        drwav_uint64 framesRead = drwav_read_pcm_frames(pWav, drwav_min(framesToRead, sizeof(sampleData)/bytesPerFrame), sampleData);
        # 如果未读取到任何帧，则跳出循环
        if (framesRead == 0) {
            break;
        }

        # 将 ALAW 格式的数据转换为有符号 32 位整数，并存储到输出缓冲区
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
    drwav_uint64 totalFramesRead;
    drwav_uint8 sampleData[4096];

    drwav_uint32 bytesPerFrame = drwav_get_bytes_per_pcm_frame(pWav);
    // 检查每个 PCM 帧的字节数是否为 0，如果是则返回 0
    if (bytesPerFrame == 0) {
        return 0;
    }

    totalFramesRead = 0;

    // 循环读取 PCM 帧数据
    while (framesToRead > 0) {
        // 读取 PCM 帧数据到 sampleData 数组中
        drwav_uint64 framesRead = drwav_read_pcm_frames(pWav, drwav_min(framesToRead, sizeof(sampleData)/bytesPerFrame), sampleData);
        // 如果没有读取到 PCM 帧数据，则跳出循环
        if (framesRead == 0) {
            break;
        }

        // 将 mu-law 编码的数据转换为有符号 32 位整数
        drwav_mulaw_to_s32(pBufferOut, sampleData, (size_t)(framesRead*pWav->channels));

        // 更新输出缓冲区指针和剩余需要读取的 PCM 帧数
        pBufferOut      += framesRead*pWav->channels;
        framesToRead    -= framesRead;
        totalFramesRead += framesRead;
    }

    return totalFramesRead;
}

DRWAV_API drwav_uint64 drwav_read_pcm_frames_s32(drwav* pWav, drwav_uint64 framesToRead, drwav_int32* pBufferOut)
{
    // 检查输入参数是否有效
    if (pWav == NULL || framesToRead == 0) {
        return 0;
    }

    if (pBufferOut == NULL) {
        // 如果输出缓冲区为空，则直接读取 PCM 帧数据并返回
        return drwav_read_pcm_frames(pWav, framesToRead, NULL);
    }

    /* Don't try to read more samples than can potentially fit in the output buffer. */
    // 避免尝试读取超过输出缓冲区容量的样本数
    if (framesToRead * pWav->channels * sizeof(drwav_int32) > DRWAV_SIZE_MAX) {
        framesToRead = DRWAV_SIZE_MAX / sizeof(drwav_int32) / pWav->channels;
    }

    // 根据音频格式标签调用相应的读取函数
    if (pWav->translatedFormatTag == DR_WAVE_FORMAT_PCM) {
        return drwav_read_pcm_frames_s32__pcm(pWav, framesToRead, pBufferOut);
    }

    if (pWav->translatedFormatTag == DR_WAVE_FORMAT_ADPCM) {
        return drwav_read_pcm_frames_s32__msadpcm(pWav, framesToRead, pBufferOut);
    }

    if (pWav->translatedFormatTag == DR_WAVE_FORMAT_IEEE_FLOAT) {
        return drwav_read_pcm_frames_s32__ieee(pWav, framesToRead, pBufferOut);
    }

    if (pWav->translatedFormatTag == DR_WAVE_FORMAT_ALAW) {
        return drwav_read_pcm_frames_s32__alaw(pWav, framesToRead, pBufferOut);
    }
}
    # 如果 WAV 文件的格式为 mulaw，则调用相应的函数读取 PCM 帧数据
    if (pWav->translatedFormatTag == DR_WAVE_FORMAT_MULAW) {
        return drwav_read_pcm_frames_s32__mulaw(pWav, framesToRead, pBufferOut);
    }

    # 如果 WAV 文件的格式为 DVI ADPCM，则调用相应的函数读取 PCM 帧数据
    if (pWav->translatedFormatTag == DR_WAVE_FORMAT_DVI_ADPCM) {
        return drwav_read_pcm_frames_s32__ima(pWav, framesToRead, pBufferOut);
    }

    # 如果不是以上两种格式，则返回 0
    return 0;
}

# 读取带有 s32le 格式的 PCM 帧数据
DRWAV_API drwav_uint64 drwav_read_pcm_frames_s32le(drwav* pWav, drwav_uint64 framesToRead, drwav_int32* pBufferOut)
{
    # 调用 drwav_read_pcm_frames_s32 函数读取 s32 格式的 PCM 帧数据
    drwav_uint64 framesRead = drwav_read_pcm_frames_s32(pWav, framesToRead, pBufferOut);
    # 如果输出缓冲区不为空且系统是小端序，则交换 s32 格式的样本字节序
    if (pBufferOut != NULL && drwav__is_little_endian() == DRWAV_FALSE) {
        drwav__bswap_samples_s32(pBufferOut, framesRead*pWav->channels);
    }

    # 返回读取的帧数
    return framesRead;
}

# 读取带有 s32be 格式的 PCM 帧数据
DRWAV_API drwav_uint64 drwav_read_pcm_frames_s32be(drwav* pWav, drwav_uint64 framesToRead, drwav_int32* pBufferOut)
{
    # 调用 drwav_read_pcm_frames_s32 函数读取 s32 格式的 PCM 帧数据
    drwav_uint64 framesRead = drwav_read_pcm_frames_s32(pWav, framesToRead, pBufferOut);
    # 如果输出缓冲区不为空且系统是大端序，则交换 s32 格式的样本字节序
    if (pBufferOut != NULL && drwav__is_little_endian() == DRWAV_TRUE) {
        drwav__bswap_samples_s32(pBufferOut, framesRead*pWav->channels);
    }

    # 返回读取的帧数
    return framesRead;
}

# 将 u8 格式的样本数据转换为 s32 格式
DRWAV_API void drwav_u8_to_s32(drwav_int32* pOut, const drwav_uint8* pIn, size_t sampleCount)
{
    size_t i;

    # 如果输出缓冲区或输入数据为空，则直接返回
    if (pOut == NULL || pIn == NULL) {
        return;
    }

    # 遍历每个样本，将 u8 格式的样本数据转换为 s32 格式
    for (i = 0; i < sampleCount; ++i) {
        *pOut++ = ((int)pIn[i] - 128) << 24;
    }
}

# 将 s16 格式的样本数据转换为 s32 格式
DRWAV_API void drwav_s16_to_s32(drwav_int32* pOut, const drwav_int16* pIn, size_t sampleCount)
{
    size_t i;

    # 如果输出缓冲区或输入数据为空，则直接返回
    if (pOut == NULL || pIn == NULL) {
        return;
    }

    # 遍历每个样本，将 s16 格式的样本数据转换为 s32 格式
    for (i = 0; i < sampleCount; ++i) {
        *pOut++ = pIn[i] << 16;
    }
}

# 将 s24 格式的样本数据转换为 s32 格式
DRWAV_API void drwav_s24_to_s32(drwav_int32* pOut, const drwav_uint8* pIn, size_t sampleCount)
{
    size_t i;

    # 如果输出缓冲区或输入数据为空，则直接返回
    if (pOut == NULL || pIn == NULL) {
        return;
    }

    # 遍历每个样本，将 s24 格式的样本数据转换为 s32 格式
    for (i = 0; i < sampleCount; ++i) {
        unsigned int s0 = pIn[i*3 + 0];
        unsigned int s1 = pIn[i*3 + 1];
        unsigned int s2 = pIn[i*3 + 2];

        drwav_int32 sample32 = (drwav_int32)((s0 << 8) | (s1 << 16) | (s2 << 24));
        *pOut++ = sample32;
    }
}

# 将 f32 格式的样本数据转换为 s32 格式
DRWAV_API void drwav_f32_to_s32(drwav_int32* pOut, const float* pIn, size_t sampleCount)
{
    size_t i;

    # 如果输出缓冲区或输入数据为空，则直接返回
    if (pOut == NULL || pIn == NULL) {
        return;
    }
    # 遍历从 0 到 sampleCount 的索引
    for (i = 0; i < sampleCount; ++i) {
        # 将输入数组中的每个元素乘以 2147483648.0 转换为整数，并存入输出数组中
        *pOut++ = (drwav_int32)(2147483648.0 * pIn[i]);
    }
}

// 将双精度浮点数组转换为有符号32位整数数组
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

// 将A-law编码转换为有符号32位整数数组
DRWAV_API void drwav_alaw_to_s32(drwav_int32* pOut, const drwav_uint8* pIn, size_t sampleCount)
{
    size_t i;

    // 检查输入指针是否为空
    if (pOut == NULL || pIn == NULL) {
        return;
    }

    // 遍历输入数组，将A-law编码转换为有符号32位整数
    for (i = 0; i < sampleCount; ++i) {
        *pOut++ = ((drwav_int32)drwav__alaw_to_s16(pIn[i])) << 16;
    }
}

// 将μ-law编码转换为有符号32位整数数组
DRWAV_API void drwav_mulaw_to_s32(drwav_int32* pOut, const drwav_uint8* pIn, size_t sampleCount)
{
    size_t i;

    // 检查输入指针是否为空
    if (pOut == NULL || pIn == NULL) {
        return;
    }

    // 遍历输入数组，将μ-law编码转换为有符号32位整数
    for (i= 0; i < sampleCount; ++i) {
        *pOut++ = ((drwav_int32)drwav__mulaw_to_s16(pIn[i])) << 16;
    }
}

// 读取PCM帧数据并关闭WAV文件，返回有符号16位整数数组
static drwav_int16* drwav__read_pcm_frames_and_close_s16(drwav* pWav, unsigned int* channels, unsigned int* sampleRate, drwav_uint64* totalFrameCount)
{
    drwav_uint64 sampleDataSize;
    drwav_int16* pSampleData;
    drwav_uint64 framesRead;

    // 断言WAV文件指针不为空
    DRWAV_ASSERT(pWav != NULL);

    // 计算样本数据大小
    sampleDataSize = pWav->totalPCMFrameCount * pWav->channels * sizeof(drwav_int16);
    // 检查样本数据大小是否超出最大限制
    if (sampleDataSize > DRWAV_SIZE_MAX) {
        drwav_uninit(pWav);
        return NULL;    /* File's too big. */
    }

    // 分配内存存储样本数据
    pSampleData = (drwav_int16*)drwav__malloc_from_callbacks((size_t)sampleDataSize, &pWav->allocationCallbacks); /* <-- Safe cast due to the check above. */
    // 检查内存分配是否成功
    if (pSampleData == NULL) {
        drwav_uninit(pWav);
        return NULL;    /* Failed to allocate memory. */
    }

    // 读取PCM帧数据到有符号16位整数数组
    framesRead = drwav_read_pcm_frames_s16(pWav, (size_t)pWav->totalPCMFrameCount, pSampleData);
    // 检查读取帧数是否与总帧数相等
    if (framesRead != pWav->totalPCMFrameCount) {
        drwav__free_from_callbacks(pSampleData, &pWav->allocationCallbacks);
        drwav_uninit(pWav);
        return NULL;    /* There was an error reading the samples. */
    }

    // 关闭WAV文件
    drwav_uninit(pWav);
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

    # 返回 pSampleData 指针
    return pSampleData;
static float* drwav__read_pcm_frames_and_close_f32(drwav* pWav, unsigned int* channels, unsigned int* sampleRate, drwav_uint64* totalFrameCount)
{
    drwav_uint64 sampleDataSize; // 用于存储采样数据的大小
    float* pSampleData; // 存储读取的采样数据
    drwav_uint64 framesRead; // 存储读取的帧数

    DRWAV_ASSERT(pWav != NULL); // 断言 WAV 对象不为空

    sampleDataSize = pWav->totalPCMFrameCount * pWav->channels * sizeof(float); // 计算采样数据的大小
    if (sampleDataSize > DRWAV_SIZE_MAX) {
        drwav_uninit(pWav);
        return NULL;    /* File's too big. */ // 如果文件太大，则返回空指针
    }

    pSampleData = (float*)drwav__malloc_from_callbacks((size_t)sampleDataSize, &pWav->allocationCallbacks); /* <-- Safe cast due to the check above. */
    // 从回调函数中分配内存给 pSampleData，安全的类型转换，根据上面的检查
    if (pSampleData == NULL) {
        drwav_uninit(pWav);
        return NULL;    /* Failed to allocate memory. */ // 如果内存分配失败，则返回空指针
    }

    framesRead = drwav_read_pcm_frames_f32(pWav, (size_t)pWav->totalPCMFrameCount, pSampleData);
    // 读取浮点型的 PCM 帧到 pSampleData 中
    if (framesRead != pWav->totalPCMFrameCount) {
        drwav__free_from_callbacks(pSampleData, &pWav->allocationCallbacks);
        drwav_uninit(pWav);
        return NULL;    /* There was an error reading the samples. */ // 如果读取样本时出错，则返回空指针
    }

    drwav_uninit(pWav); // 关闭 WAV 对象

    if (sampleRate) {
        *sampleRate = pWav->sampleRate; // 如果 sampleRate 不为空，则将采样率赋值给它
    }
    if (channels) {
        *channels = pWav->channels; // 如果 channels 不为空，则将通道数赋值给它
    }
    if (totalFrameCount) {
        *totalFrameCount = pWav->totalPCMFrameCount; // 如果 totalFrameCount 不为空，则将总帧数赋值给它
    }

    return pSampleData; // 返回读取的采样数据
}

static drwav_int32* drwav__read_pcm_frames_and_close_s32(drwav* pWav, unsigned int* channels, unsigned int* sampleRate, drwav_uint64* totalFrameCount)
{
    drwav_uint64 sampleDataSize; // 用于存储采样数据的大小
    drwav_int32* pSampleData; // 存储读取的采样数据
    drwav_uint64 framesRead; // 存储读取的帧数

    DRWAV_ASSERT(pWav != NULL); // 断言 WAV 对象不为空

    sampleDataSize = pWav->totalPCMFrameCount * pWav->channels * sizeof(drwav_int32); // 计算采样数据的大小
    if (sampleDataSize > DRWAV_SIZE_MAX) {
        drwav_uninit(pWav);
        return NULL;    /* File's too big. */ // 如果文件太大，则返回空指针
    }

    pSampleData = (drwav_int32*)drwav__malloc_from_callbacks((size_t)sampleDataSize, &pWav->allocationCallbacks); /* <-- Safe cast due to the check above. */
    // 从回调函数中分配内存给 pSampleData，安全的类型转换，根据上面的检查
    // 如果样本数据为空，则释放 WAV 对象并返回空指针，表示内存分配失败
    if (pSampleData == NULL) {
        drwav_uninit(pWav);
        return NULL;    /* Failed to allocate memory. */
    }

    // 读取 WAV 文件中的 PCM 帧数据到 pSampleData 中
    framesRead = drwav_read_pcm_frames_s32(pWav, (size_t)pWav->totalPCMFrameCount, pSampleData);
    // 如果读取的帧数不等于总帧数，则释放 pSampleData 和 WAV 对象，并返回空指针，表示读取样本数据出错
    if (framesRead != pWav->totalPCMFrameCount) {
        drwav__free_from_callbacks(pSampleData, &pWav->allocationCallbacks);
        drwav_uninit(pWav);
        return NULL;    /* There was an error reading the samples. */
    }

    // 释放 WAV 对象
    drwav_uninit(pWav);

    // 如果 sampleRate 不为空，则将 pWav 的采样率赋值给 sampleRate
    if (sampleRate) {
        *sampleRate = pWav->sampleRate;
    }
    // 如果 channels 不为空，则将 pWav 的声道数赋值给 channels
    if (channels) {
        *channels = pWav->channels;
    }
    // 如果 totalFrameCount 不为空，则将 pWav 的总帧数赋值给 totalFrameCount
    if (totalFrameCount) {
        *totalFrameCount = pWav->totalPCMFrameCount;
    }

    // 返回样本数据指针
    return pSampleData;
# 打开并读取 PCM 帧数据，返回有符号 16 位整数数组
DRWAV_API drwav_int16* drwav_open_and_read_pcm_frames_s16(drwav_read_proc onRead, drwav_seek_proc onSeek, void* pUserData, unsigned int* channelsOut, unsigned int* sampleRateOut, drwav_uint64* totalFrameCountOut, const drwav_allocation_callbacks* pAllocationCallbacks)
{
    # 创建 drwav 结构体
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

    # 调用内部函数读取 PCM 帧数据并关闭文件，返回有符号 16 位整数数组
    return drwav__read_pcm_frames_and_close_s16(&wav, channelsOut, sampleRateOut, totalFrameCountOut);
}

# 打开并读取 PCM 帧数据，返回单精度浮点数数组
DRWAV_API float* drwav_open_and_read_pcm_frames_f32(drwav_read_proc onRead, drwav_seek_proc onSeek, void* pUserData, unsigned int* channelsOut, unsigned int* sampleRateOut, drwav_uint64* totalFrameCountOut, const drwav_allocation_callbacks* pAllocationCallbacks)
{
    # 创建 drwav 结构体
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

    # 调用内部函数读取 PCM 帧数据并关闭文件，返回单精度浮点数数组
    return drwav__read_pcm_frames_and_close_f32(&wav, channelsOut, sampleRateOut, totalFrameCountOut);
}

# 打开并读取 PCM 帧数据，返回有符号 32 位整数数组
DRWAV_API drwav_int32* drwav_open_and_read_pcm_frames_s32(drwav_read_proc onRead, drwav_seek_proc onSeek, void* pUserData, unsigned int* channelsOut, unsigned int* sampleRateOut, drwav_uint64* totalFrameCountOut, const drwav_allocation_callbacks* pAllocationCallbacks)
{
    # 创建 drwav 结构体
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
    # 调用函数 drwav__read_pcm_frames_and_close_s32() 读取 PCM 帧并关闭 WAV 文件，返回结果
    return drwav__read_pcm_frames_and_close_s32(&wav, channelsOut, sampleRateOut, totalFrameCountOut);
// 如果没有定义 DR_WAV_NO_STDIO，则定义 drwav_open_file_and_read_pcm_frames_s16 函数
DRWAV_API drwav_int16* drwav_open_file_and_read_pcm_frames_s16(const char* filename, unsigned int* channelsOut, unsigned int* sampleRateOut, drwav_uint64* totalFrameCountOut, const drwav_allocation_callbacks* pAllocationCallbacks)
{
    // 创建 drwav 结构体
    drwav wav;

    // 如果 channelsOut 不为空，则将其值设为 0
    if (channelsOut) {
        *channelsOut = 0;
    }
    // 如果 sampleRateOut 不为空，则将其值设为 0
    if (sampleRateOut) {
        *sampleRateOut = 0;
    }
    // 如果 totalFrameCountOut 不为空，则将其值设为 0
    if (totalFrameCountOut) {
        *totalFrameCountOut = 0;
    }

    // 初始化文件，如果失败则返回 NULL
    if (!drwav_init_file(&wav, filename, pAllocationCallbacks)) {
        return NULL;
    }

    // 调用内部函数读取 PCM 数据并关闭文件，返回 int16 类型指针
    return drwav__read_pcm_frames_and_close_s16(&wav, channelsOut, sampleRateOut, totalFrameCountOut);
}

// 定义 drwav_open_file_and_read_pcm_frames_f32 函数
DRWAV_API float* drwav_open_file_and_read_pcm_frames_f32(const char* filename, unsigned int* channelsOut, unsigned int* sampleRateOut, drwav_uint64* totalFrameCountOut, const drwav_allocation_callbacks* pAllocationCallbacks)
{
    // 创建 drwav 结构体
    drwav wav;

    // 如果 channelsOut 不为空，则将其值设为 0
    if (channelsOut) {
        *channelsOut = 0;
    }
    // 如果 sampleRateOut 不为空，则将其值设为 0
    if (sampleRateOut) {
        *sampleRateOut = 0;
    }
    // 如果 totalFrameCountOut 不为空，则将其值设为 0
    if (totalFrameCountOut) {
        *totalFrameCountOut = 0;
    }

    // 初始化文件，如果失败则返回 NULL
    if (!drwav_init_file(&wav, filename, pAllocationCallbacks)) {
        return NULL;
    }

    // 调用内部函数读取 PCM 数据并关闭文件，返回 float 类型指针
    return drwav__read_pcm_frames_and_close_f32(&wav, channelsOut, sampleRateOut, totalFrameCountOut);
}

// 定义 drwav_open_file_and_read_pcm_frames_s32 函数
DRWAV_API drwav_int32* drwav_open_file_and_read_pcm_frames_s32(const char* filename, unsigned int* channelsOut, unsigned int* sampleRateOut, drwav_uint64* totalFrameCountOut, const drwav_allocation_callbacks* pAllocationCallbacks)
{
    // 创建 drwav 结构体
    drwav wav;

    // 如果 channelsOut 不为空，则将其值设为 0
    if (channelsOut) {
        *channelsOut = 0;
    }
    // 如果 sampleRateOut 不为空，则将其值设为 0
    if (sampleRateOut) {
        *sampleRateOut = 0;
    }
    // 如果 totalFrameCountOut 不为空，则将其值设为 0
    if (totalFrameCountOut) {
        *totalFrameCountOut = 0;
    }

    // 初始化文件，如果失败则返回 NULL
    if (!drwav_init_file(&wav, filename, pAllocationCallbacks)) {
        return NULL;
    }

    // 调用内部函数读取 PCM 数据并关闭文件，返回 int32 类型指针
    return drwav__read_pcm_frames_and_close_s32(&wav, channelsOut, sampleRateOut, totalFrameCountOut);
}
# 打开一个 WAV 文件并读取 PCM 帧数据，返回有符号 16 位整数数组指针
DRWAV_API drwav_int16* drwav_open_file_and_read_pcm_frames_s16_w(const wchar_t* filename, unsigned int* channelsOut, unsigned int* sampleRateOut, drwav_uint64* totalFrameCountOut, const drwav_allocation_callbacks* pAllocationCallbacks)
{
    drwav wav;

    # 如果 sampleRateOut 不为空，则将其值设为 0
    if (sampleRateOut) {
        *sampleRateOut = 0;
    }
    # 如果 channelsOut 不为空，则将其值设为 0
    if (channelsOut) {
        *channelsOut = 0;
    }
    # 如果 totalFrameCountOut 不为空，则将其值设为 0
    if (totalFrameCountOut) {
        *totalFrameCountOut = 0;
    }

    # 初始化 WAV 结构体，打开文件
    if (!drwav_init_file_w(&wav, filename, pAllocationCallbacks)) {
        return NULL;
    }

    # 读取 PCM 帧数据并关闭 WAV 文件
    return drwav__read_pcm_frames_and_close_s16(&wav, channelsOut, sampleRateOut, totalFrameCountOut);
}

# 打开一个 WAV 文件并读取 PCM 帧数据，返回单精度浮点数数组指针
DRWAV_API float* drwav_open_file_and_read_pcm_frames_f32_w(const wchar_t* filename, unsigned int* channelsOut, unsigned int* sampleRateOut, drwav_uint64* totalFrameCountOut, const drwav_allocation_callbacks* pAllocationCallbacks)
{
    drwav wav;

    # 如果 sampleRateOut 不为空，则将其值设为 0
    if (sampleRateOut) {
        *sampleRateOut = 0;
    }
    # 如果 channelsOut 不为空，则将其值设为 0
    if (channelsOut) {
        *channelsOut = 0;
    }
    # 如果 totalFrameCountOut 不为空，则将其值设为 0
    if (totalFrameCountOut) {
        *totalFrameCountOut = 0;
    }

    # 初始化 WAV 结构体，打开文件
    if (!drwav_init_file_w(&wav, filename, pAllocationCallbacks)) {
        return NULL;
    }

    # 读取 PCM 帧数据并关闭 WAV 文件
    return drwav__read_pcm_frames_and_close_f32(&wav, channelsOut, sampleRateOut, totalFrameCountOut);
}

# 打开一个 WAV 文件并读取 PCM 帧数据，返回有符号 32 位整数数组指针
DRWAV_API drwav_int32* drwav_open_file_and_read_pcm_frames_s32_w(const wchar_t* filename, unsigned int* channelsOut, unsigned int* sampleRateOut, drwav_uint64* totalFrameCountOut, const drwav_allocation_callbacks* pAllocationCallbacks)
{
    drwav wav;

    # 如果 sampleRateOut 不为空，则将其值设为 0
    if (sampleRateOut) {
        *sampleRateOut = 0;
    }
    # 如果 channelsOut 不为空，则将其值设为 0
    if (channelsOut) {
        *channelsOut = 0;
    }
    # 如果 totalFrameCountOut 不为空，则将其值设为 0
    if (totalFrameCountOut) {
        *totalFrameCountOut = 0;
    }

    # 初始化 WAV 结构体，打开文件
    if (!drwav_init_file_w(&wav, filename, pAllocationCallbacks)) {
        return NULL;
    }

    # 读取 PCM 帧数据并关闭 WAV 文件
    return drwav__read_pcm_frames_and_close_s32(&wav, channelsOut, sampleRateOut, totalFrameCountOut);
}
#endif
# 从内存中打开并读取带有有符号16位整数格式的 PCM 帧
DRWAV_API drwav_int16* drwav_open_memory_and_read_pcm_frames_s16(const void* data, size_t dataSize, unsigned int* channelsOut, unsigned int* sampleRateOut, drwav_uint64* totalFrameCountOut, const drwav_allocation_callbacks* pAllocationCallbacks)
{
    # 创建一个 drwav 结构体
    drwav wav;

    # 如果 channelsOut 不为空，则将其值设为0
    if (channelsOut) {
        *channelsOut = 0;
    }
    # 如果 sampleRateOut 不为空，则将其值设为0
    if (sampleRateOut) {
        *sampleRateOut = 0;
    }
    # 如果 totalFrameCountOut 不为空，则将其值设为0
    if (totalFrameCountOut) {
        *totalFrameCountOut = 0;
    }

    # 初始化内存中的 WAV 文件，如果失败则返回空指针
    if (!drwav_init_memory(&wav, data, dataSize, pAllocationCallbacks)) {
        return NULL;
    }

    # 调用内部函数读取 PCM 帧并关闭 WAV 文件，返回结果
    return drwav__read_pcm_frames_and_close_s16(&wav, channelsOut, sampleRateOut, totalFrameCountOut);
}

# 从内存中打开并读取带有浮点32位格式的 PCM 帧
DRWAV_API float* drwav_open_memory_and_read_pcm_frames_f32(const void* data, size_t dataSize, unsigned int* channelsOut, unsigned int* sampleRateOut, drwav_uint64* totalFrameCountOut, const drwav_allocation_callbacks* pAllocationCallbacks)
{
    # 创建一个 drwav 结构体
    drwav wav;

    # 如果 channelsOut 不为空，则将其值设为0
    if (channelsOut) {
        *channelsOut = 0;
    }
    # 如果 sampleRateOut 不为空，则将其值设为0
    if (sampleRateOut) {
        *sampleRateOut = 0;
    }
    # 如果 totalFrameCountOut 不为空，则将其值设为0
    if (totalFrameCountOut) {
        *totalFrameCountOut = 0;
    }

    # 初始化内存中的 WAV 文件，如果失败则返回空指针
    if (!drwav_init_memory(&wav, data, dataSize, pAllocationCallbacks)) {
        return NULL;
    }

    # 调用内部函数读取 PCM 帧并关闭 WAV 文件，返回结果
    return drwav__read_pcm_frames_and_close_f32(&wav, channelsOut, sampleRateOut, totalFrameCountOut);
}

# 从内存中打开并读取带有有符号32位整数格式的 PCM 帧
DRWAV_API drwav_int32* drwav_open_memory_and_read_pcm_frames_s32(const void* data, size_t dataSize, unsigned int* channelsOut, unsigned int* sampleRateOut, drwav_uint64* totalFrameCountOut, const drwav_allocation_callbacks* pAllocationCallbacks)
{
    # 创建一个 drwav 结构体
    drwav wav;

    # 如果 channelsOut 不为空，则将其值设为0
    if (channelsOut) {
        *channelsOut = 0;
    }
    # 如果 sampleRateOut 不为空，则将其值设为0
    if (sampleRateOut) {
        *sampleRateOut = 0;
    }
    # 如果 totalFrameCountOut 不为空，则将其值设为0
    if (totalFrameCountOut) {
        *totalFrameCountOut = 0;
    }

    # 初始化内存中的 WAV 文件，如果失败则返回空指针
    if (!drwav_init_memory(&wav, data, dataSize, pAllocationCallbacks)) {
        return NULL;
    }

    # 调用内部函数读取 PCM 帧并关闭 WAV 文件，返回结果
    return drwav__read_pcm_frames_and_close_s32(&wav, channelsOut, sampleRateOut, totalFrameCountOut);
}
#ifdef DR_WAV_NO_CONVERSION_API
#endif  /* DR_WAV_NO_CONVERSION_API */

// 释放内存，根据传入的内存分配回调函数指针选择相应的释放函数
DRWAV_API void drwav_free(void* p, const drwav_allocation_callbacks* pAllocationCallbacks)
{
    // 如果存在内存分配回调函数指针，则使用该函数释放内存
    if (pAllocationCallbacks != NULL) {
        drwav__free_from_callbacks(p, pAllocationCallbacks);
    } else {
        // 否则使用默认的释放函数
        drwav__free_default(p, NULL);
    }
}

// 将字节数组转换为无符号16位整数
DRWAV_API drwav_uint16 drwav_bytes_to_u16(const drwav_uint8* data)
{
    return drwav__bytes_to_u16(data);
}

// 将字节数组转换为有符号16位整数
DRWAV_API drwav_int16 drwav_bytes_to_s16(const drwav_uint8* data)
{
    return drwav__bytes_to_s16(data);
}

// 将字节数组转换为无符号32位整数
DRWAV_API drwav_uint32 drwav_bytes_to_u32(const drwav_uint8* data)
{
    return drwav__bytes_to_u32(data);
}

// 将字节数组转换为有符号32位整数
DRWAV_API drwav_int32 drwav_bytes_to_s32(const drwav_uint8* data)
{
    return drwav__bytes_to_s32(data);
}

// 将字节数组转换为无符号64位整数
DRWAV_API drwav_uint64 drwav_bytes_to_u64(const drwav_uint8* data)
{
    return drwav__bytes_to_u64(data);
}

// 将字节数组转换为有符号64位整数
DRWAV_API drwav_int64 drwav_bytes_to_s64(const drwav_uint8* data)
{
    return drwav__bytes_to_s64(data);
}

// 比较两个GUID是否相等
DRWAV_API drwav_bool32 drwav_guid_equal(const drwav_uint8 a[16], const drwav_uint8 b[16])
{
    return drwav__guid_equal(a, b);
}

// 比较两个FourCC是否相等
DRWAV_API drwav_bool32 drwav_fourcc_equal(const drwav_uint8* a, const char* b)
{
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
    // 重新分配内存的函数，用于替换标准库中的 realloc 函数
    void* my_realloc(void* p, size_t sz, void* pUserData)
    {
        return realloc(p, sz);
    }
    
    // 释放内存的函数，用于替换标准库中的 free 函数
    void my_free(void* p, void* pUserData)
    {
        free(p);
    }
    
    // 创建一个结构体变量，用于存储自定义的内存分配回调函数
    drwav_allocation_callbacks allocationCallbacks;
    
    // 将自定义数据指针传递给 pUserData
    allocationCallbacks.pUserData = &myData;
    
    // 将自定义的内存分配函数指针传递给 onMalloc
    allocationCallbacks.onMalloc  = my_malloc;
    
    // 将自定义的内存重新分配函数指针传递给 onRealloc
    allocationCallbacks.onRealloc = my_realloc;
    
    // 将自定义的内存释放函数指针传递给 onFree
    allocationCallbacks.onFree    = my_free;
    
    // 初始化 WAV 文件对象，使用自定义的内存分配回调函数
    drwav_init_file(&wav, "my_file.wav", &allocationCallbacks);
# 这个新系统的优势在于它允许您指定用户数据，这些数据将传递给分配例程。
# 传递空值给分配回调对象将导致 dr_wav 使用默认值，与 DRWAV_MALLOC、DRWAV_REALLOC 和 DRWAV_FREE 相同，相当于在以前的版本中的工作方式。
# 现在每个打开 drwav 对象的 API 都需要这个额外的参数。这些包括以下内容：
# drwav_init()
# drwav_init_ex()
# drwav_init_file()
# drwav_init_file_ex()
# drwav_init_file_w()
# drwav_init_file_w_ex()
# drwav_init_memory()
# drwav_init_memory_ex()
# drwav_init_write()
# drwav_init_write_sequential()
# drwav_init_write_sequential_pcm_frames()
# drwav_init_file_write()
# drwav_init_file_write_sequential()
# drwav_init_file_write_sequential_pcm_frames()
# drwav_init_file_write_w()
# drwav_init_file_write_sequential_w()
# drwav_init_file_write_sequential_pcm_frames_w()
# drwav_init_memory_write()
# drwav_init_memory_write_sequential()
# drwav_init_memory_write_sequential_pcm_frames()
# drwav_open_and_read_pcm_frames_s16()
# drwav_open_and_read_pcm_frames_f32()
# drwav_open_and_read_pcm_frames_s32()
# drwav_open_file_and_read_pcm_frames_s16()
# drwav_open_file_and_read_pcm_frames_f32()
# drwav_open_file_and_read_pcm_frames_s32()
# drwav_open_file_and_read_pcm_frames_s16_w()
# drwav_open_file_and_read_pcm_frames_f32_w()
# drwav_open_file_and_read_pcm_frames_s32_w()
# drwav_open_memory_and_read_pcm_frames_s16()
# drwav_open_memory_and_read_pcm_frames_f32()
# drwav_open_memory_and_read_pcm_frames_s32()

# 端点改进
# 以前，以下 API 返回小端音频数据。现在这些 API 返回本机端数据。这提高了在大端架构上的兼容性。
# drwav_read_pcm_frames()
# drwav_read_pcm_frames_s16()
# drwav_read_pcm_frames_s32()
# drwav_read_pcm_frames_f32()
    # 调用函数 drwav_open_and_read_pcm_frames_s16()，打开并读取 PCM 帧数据，数据格式为有符号 16 位整数
    drwav_open_and_read_pcm_frames_s16()
    # 调用函数 drwav_open_and_read_pcm_frames_s32()，打开并读取 PCM 帧数据，数据格式为有符号 32 位整数
    drwav_open_and_read_pcm_frames_s32()
    # 调用函数 drwav_open_and_read_pcm_frames_f32()，打开并读取 PCM 帧数据，数据格式为 32 位浮点数
    drwav_open_and_read_pcm_frames_f32()
    # 调用函数 drwav_open_file_and_read_pcm_frames_s16()，打开文件并读取 PCM 帧数据，数据格式为有符号 16 位整数
    drwav_open_file_and_read_pcm_frames_s16()
    # 调用函数 drwav_open_file_and_read_pcm_frames_s32()，打开文件并读取 PCM 帧数据，数据格式为有符号 32 位整数
    drwav_open_file_and_read_pcm_frames_s32()
    # 调用函数 drwav_open_file_and_read_pcm_frames_f32()，打开文件并读取 PCM 帧数据，数据格式为 32 位浮点数
    drwav_open_file_and_read_pcm_frames_f32()
    # 调用函数 drwav_open_file_and_read_pcm_frames_s16_w()，打开文件并读取 PCM 帧数据，数据格式为有符号 16 位整数
    drwav_open_file_and_read_pcm_frames_s16_w()
    # 调用函数 drwav_open_file_and_read_pcm_frames_s32_w()，打开文件并读取 PCM 帧数据，数据格式为有符号 32 位整数
    drwav_open_file_and_read_pcm_frames_s32_w()
    # 调用函数 drwav_open_file_and_read_pcm_frames_f32_w()，打开文件并读取 PCM 帧数据，数据格式为 32 位浮点数
    drwav_open_file_and_read_pcm_frames_f32_w()
    # 调用函数 drwav_open_memory_and_read_pcm_frames_s16()，打开内存并读取 PCM 帧数据，数据格式为有符号 16 位整数
    drwav_open_memory_and_read_pcm_frames_s16()
    # 调用函数 drwav_open_memory_and_read_pcm_frames_s32()，打开内存并读取 PCM 帧数据，数据格式为有符号 32 位整数
    drwav_open_memory_and_read_pcm_frames_s32()
    # 调用函数 drwav_open_memory_and_read_pcm_frames_f32()，打开内存并读取 PCM 帧数据，数据格式为 32 位浮点数
    drwav_open_memory_and_read_pcm_frames_f32()
# 添加了一些 API，用于明确控制音频数据是以大端还是小端字节顺序读取或写入：
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

# 移除的 API
# 以下 API 在版本 0.10.0 中已被弃用并移除：
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

# 发行说明 - v0.10.0
# 版本 0.10.0 中有破坏性的 API 更改。在此版本中没有重大的错误修复，所以如果受到影响，则不需要升级。

# 移除的 API
# 以下 API 在版本 0.9.0 中已被弃用，并在版本 0.10.0 中完全移除：
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

# 请查看此文件底部的版本 0.9.0 的发行说明以获取替代 API。

# 弃用的 API
# 以下 API 已被弃用。drwav_init*() 和之间存在混乱和完全任意的区别
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

/*
These APIs will be removed completely in a future version. The rationale for this change is to remove confusion between the
two different ways to initialize a drwav object.
*/

/*
drwav_open*(), where drwav_init*() initializes a pre-allocated drwav object, whereas drwav_open*() will first allocated a
drwav object on the heap and then initialize it. drwav_open*() has been deprecated which means you must now use a pre-
allocated drwav object with drwav_init*(). If you need the previous functionality, you can just do a malloc() followed by
a called to one of the drwav_init*() APIs.
*/

// Deprecated API for opening and initializing a drwav object
drwav_open()

// Deprecated API for opening and initializing a drwav object with extended options
drwav_open_ex()

// Deprecated API for opening and initializing a drwav object for writing
drwav_open_write()

// Deprecated API for opening and initializing a drwav object for writing sequentially
drwav_open_write_sequential()

// Deprecated API for opening and initializing a drwav object from a file
drwav_open_file()

// Deprecated API for opening and initializing a drwav object from a file with extended options
drwav_open_file_ex()

// Deprecated API for opening and initializing a drwav object for writing to a file
drwav_open_file_write()

// Deprecated API for opening and initializing a drwav object for writing sequentially to a file
drwav_open_file_write_sequential()

// Deprecated API for opening and initializing a drwav object from memory
drwav_open_memory()

// Deprecated API for opening and initializing a drwav object from memory with extended options
drwav_open_memory_ex()

// Deprecated API for opening and initializing a drwav object for writing to memory
drwav_open_memory_write()

// Deprecated API for opening and initializing a drwav object for writing sequentially to memory
drwav_open_memory_write_sequential()

// Deprecated API for closing a drwav object
drwav_close()
# 版本 v0.12.6 - 2020-06-23
  - 修改 drwav_read_*()，允许将 NULL 作为输出缓冲区传入，相当于向前查找。
  - 修复尝试解码无效的 IMA-ADPCM 文件时的缓冲区溢出问题。
  - 为实现部分添加包含保护。

# 版本 v0.12.5 - 2020-05-27
  - 较小的文档修复。

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
  - 修复 VC6 编译错误。

# 版本 v0.12.2 - 2020-04-21
  - 修复一个 bug，即 drwav_init_file() 在尝试加载错误文件后未关闭文件句柄。

# 版本 v0.12.1 - 2020-04-13
  - 修复一些啰嗦的警告。

# 版本 v0.12.0 - 2020-04-04
  - API 更改：向 chunk 回调添加容器和格式参数。
  - 较小的文档更新。

# 版本 v0.11.5 - 2020-03-07
  - 修复与 Visual Studio .NET 2003 的编译错误。

# 版本 v0.11.4 - 2020-01-29
  - 修复一些静态分析警告。
  - 修复从 A-law 编码流中读取 f32 样本时的 bug。

# 版本 v0.11.3 - 2020-01-12
  - 对一些 f32 格式转换例程进行较小的更改。
  - 当到达文件结尾时，对 ADPCM 转换进行较小的 bug 修复。

# 版本 v0.11.2 - 2019-12-02
  - 修复在没有自定义 realloc() 实现的情况下使用自定义内存分配器可能导致崩溃的问题。
  - 修复整数溢出 bug。
  - 修复空指针解引用 bug。
  - 添加对采样率、通道数和每样本位数的限制，以加强一些验证。

# 版本 v0.11.1 - 2019-10-07
  - 内部代码清理。

# 版本 v0.11.0 - 2019-10-06
  - API 更改：添加对用户定义内存分配例程的支持。该系统允许程序指定自己的内存分配例程，带有一个用于客户特定上下文数据的用户数据指针。
    这在以下 API 的末尾添加了一个额外的参数：
    - drwav_init()
    - drwav_init_ex()
        # 初始化一个 drwav 对象，使用默认配置
    - drwav_init_file()
        # 初始化一个 drwav 对象，从文件中读取数据
    - drwav_init_file_ex()
        # 初始化一个 drwav 对象，从文件中读取数据，使用额外配置
    - drwav_init_file_w()
        # 初始化一个 drwav 对象，从文件中读取数据，支持宽字符路径
    - drwav_init_file_w_ex()
        # 初始化一个 drwav 对象，从文件中读取数据，支持宽字符路径，使用额外配置
    - drwav_init_memory()
        # 初始化一个 drwav 对象，从内存中读取数据
    - drwav_init_memory_ex()
        # 初始化一个 drwav 对象，从内存中读取数据，使用额外配置
    - drwav_init_write()
        # 初始化一个 drwav 对象，用于写入数据
    - drwav_init_write_sequential()
        # 初始化一个 drwav 对象，用于按顺序写入数据
    - drwav_init_write_sequential_pcm_frames()
        # 初始化一个 drwav 对象，用于按顺序写入 PCM 帧数据
    - drwav_init_file_write()
        # 初始化一个 drwav 对象，从文件中写入数据
    - drwav_init_file_write_sequential()
        # 初始化一个 drwav 对象，从文件中按顺序写入数据
    - drwav_init_file_write_sequential_pcm_frames()
        # 初始化一个 drwav 对象，从文件中按顺序写入 PCM 帧数据
    - drwav_init_file_write_w()
        # 初始化一个 drwav 对象，从文件中写入数据，支持宽字符路径
    - drwav_init_file_write_sequential_w()
        # 初始化一个 drwav 对象，从文件中按顺序写入数据，支持宽字符路径
    - drwav_init_file_write_sequential_pcm_frames_w()
        # 初始化一个 drwav 对象，从文件中按顺序写入 PCM 帧数据，支持宽字符路径
    - drwav_init_memory_write()
        # 初始化一个 drwav 对象，从内存中写入数据
    - drwav_init_memory_write_sequential()
        # 初始化一个 drwav 对象，从内存中按顺序写入数据
    - drwav_init_memory_write_sequential_pcm_frames()
        # 初始化一个 drwav 对象，从内存中按顺序写入 PCM 帧数据
    - drwav_open_and_read_pcm_frames_s16()
        # 打开并读取一个 drwav 对象中的 PCM 帧数据，返回有符号 16 位整数
    - drwav_open_and_read_pcm_frames_f32()
        # 打开并读取一个 drwav 对象中的 PCM 帧数据，返回单精度浮点数
    - drwav_open_and_read_pcm_frames_s32()
        # 打开并读取一个 drwav 对象中的 PCM 帧数据，返回有符号 32 位整数
    - drwav_open_file_and_read_pcm_frames_s16()
        # 打开文件并读取其中的 PCM 帧数据，返回有符号 16 位整数
    - drwav_open_file_and_read_pcm_frames_f32()
        # 打开文件并读取其中的 PCM 帧数据，返回单精度浮点数
    - drwav_open_file_and_read_pcm_frames_s32()
        # 打开文件并读取其中的 PCM 帧数据，返回有符号 32 位整数
    - drwav_open_file_and_read_pcm_frames_s16_w()
        # 打开文件并读取其中的 PCM 帧数据，返回有符号 16 位整数，支持宽字符路径
    - drwav_open_file_and_read_pcm_frames_f32_w()
        # 打开文件并读取其中的 PCM 帧数据，返回单精度浮点数，支持宽字符路径
    - drwav_open_file_and_read_pcm_frames_s32_w()
        # 打开文件并读取其中的 PCM 帧数据，返回有符号 32 位整数，支持宽字符路径
    - drwav_open_memory_and_read_pcm_frames_s16()
        # 打开内存并读取其中的 PCM 帧数据，返回有符号 16 位整数
    - drwav_open_memory_and_read_pcm_frames_f32()
        # 打开内存并读取其中的 PCM 帧数据，返回单精度浮点数
    - drwav_open_memory_and_read_pcm_frames_s32()
        # 打开内存并读取其中的 PCM 帧数据，返回有符号 32 位整数
    - Set this extra parameter to NULL to use defaults which is the same as the previous behaviour. Setting this NULL will use
        DRWAV_MALLOC, DRWAV_REALLOC and DRWAV_FREE.
        # 将此额外参数设置为 NULL 以使用默认值，与以前的行为相同。设置为 NULL 将使用 DRWAV_MALLOC、DRWAV_REALLOC 和 DRWAV_FREE。
    - Add support for reading and writing PCM frames in an explicit endianness. New APIs:
        # 添加对以显式字节序读写 PCM 帧的支持。新的 API：
    - drwav_read_pcm_frames_le()
        # 以小端字节序读取 PCM 帧
    - drwav_read_pcm_frames_be()
        # 以大端字节序读取 PCM 帧
    - drwav_read_pcm_frames_s16le()
        # 以小端字节序读取有符号 16 位整数 PCM 帧
    - drwav_read_pcm_frames_s16be()
        # 以大端字节序读取有符号 16 位整数 PCM 帧
    - drwav_read_pcm_frames_f32le()
        # 以小端字节序读取单精度浮点数 PCM 帧
    - drwav_read_pcm_frames_f32be()
        # 以大端字节序读取单精度浮点数 PCM 帧
    - drwav_read_pcm_frames_s32le()
        # 以小端字节序读取有符号 32 位整数 PCM 帧
    - drwav_read_pcm_frames_s32be()
        # 以大端字节序读取有符号 32 位整数 PCM 帧
    - drwav_write_pcm_frames_le()
        # 以小端字节序写入 PCM 帧
    - drwav_write_pcm_frames_be()
        # 以大端字节序写入 PCM 帧
    - Remove deprecated APIs.
        # 移除已弃用的 API。
    - API CHANGE: The following APIs now return native-endian data. Previously they returned little-endian data.
        # API 更改：以下 API 现在返回本机字节序数据。以前它们返回小端字节序数据。
    # 读取 WAV 文件中的 PCM 帧数据
    drwav_read_pcm_frames()
    
    # 读取 WAV 文件中的 PCM 帧数据，返回有符号 16 位整数格式
    drwav_read_pcm_frames_s16()
    
    # 读取 WAV 文件中的 PCM 帧数据，返回有符号 32 位整数格式
    drwav_read_pcm_frames_s32()
    
    # 读取 WAV 文件中的 PCM 帧数据，返回单精度浮点数格式
    drwav_read_pcm_frames_f32()
    
    # 打开 WAV 文件并读取 PCM 帧数据，返回有符号 16 位整数格式
    drwav_open_and_read_pcm_frames_s16()
    
    # 打开 WAV 文件并读取 PCM 帧数据，返回有符号 32 位整数格式
    drwav_open_and_read_pcm_frames_s32()
    
    # 打开 WAV 文件并读取 PCM 帧数据，返回单精度浮点数格式
    drwav_open_and_read_pcm_frames_f32()
    
    # 打开 WAV 文件并读取 PCM 帧数据，返回有符号 16 位整数格式
    drwav_open_file_and_read_pcm_frames_s16()
    
    # 打开 WAV 文件并读取 PCM 帧数据，返回有符号 32 位整数格式
    drwav_open_file_and_read_pcm_frames_s32()
    
    # 打开 WAV 文件并读取 PCM 帧数据，返回单精度浮点数格式
    drwav_open_file_and_read_pcm_frames_f32()
    
    # 打开 WAV 文件并读取 PCM 帧数据，返回有符号 16 位整数格式，支持写入
    drwav_open_file_and_read_pcm_frames_s16_w()
    
    # 打开 WAV 文件并读取 PCM 帧数据，返回有符号 32 位整数格式，支持写入
    drwav_open_file_and_read_pcm_frames_s32_w()
    
    # 打开 WAV 文件并读取 PCM 帧数据，返回单精度浮点数格式，支持写入
    drwav_open_file_and_read_pcm_frames_f32_w()
    
    # 打开内存中的 WAV 数据并读取 PCM 帧数据，返回有符号 16 位整数格式
    drwav_open_memory_and_read_pcm_frames_s16()
    
    # 打开内存中的 WAV 数据并读取 PCM 帧数据，返回有符号 32 位整数格式
    drwav_open_memory_and_read_pcm_frames_s32()
    
    # 打开内存中的 WAV 数据并读取 PCM 帧数据，返回单精度浮点数格式
    drwav_open_memory_and_read_pcm_frames_f32()
# 版本 v0.10.1 - 2019-08-31
  - 正确处理部分尾随的 ADPCM 块。

# 版本 v0.10.0 - 2019-08-04
  - 移除已弃用的 API。
  - 为文件加载 API 添加 wchar_t 变体：
      drwav_init_file_w()
      drwav_init_file_ex_w()
      drwav_init_file_write_w()
      drwav_init_file_write_sequential_w()
  - 添加 drwav_target_write_size_bytes()，根据格式和样本计数计算 WAV 文件的总字节大小。
  - 添加用于在顺序写模式下指定 PCM 帧计数而不是样本计数的 API：
      drwav_init_write_sequential_pcm_frames()
      drwav_init_file_write_sequential_pcm_frames()
      drwav_init_file_write_sequential_pcm_frames_w()
      drwav_init_memory_write_sequential_pcm_frames()
  - 弃用 drwav_open*() 和 drwav_close()：
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
  - 小幅更新文档。

# 版本 v0.9.2 - 2019-05-21
  - 修复警告。

# 版本 v0.9.1 - 2019-05-05
  - 添加对 C89 的支持。
  - 将许可证更改为公共领域或 MIT-0 的选择。

# 版本 v0.9.0 - 2018-12-16
  - API 更改：添加新的读取 API，用于按 PCM 帧而不是样本进行读取。旧的 API 已被弃用。
    # 在 v0.10.0 版本中将被移除。已弃用的 API 及其替代项：
      drwav_read()                     -> drwav_read_pcm_frames()
      drwav_read_s16()                 -> drwav_read_pcm_frames_s16()
      drwav_read_f32()                 -> drwav_read_pcm_frames_f32()
      drwav_read_s32()                 -> drwav_read_pcm_frames_s32()
      drwav_seek_to_sample()           -> drwav_seek_to_pcm_frame()
      drwav_write()                    -> drwav_write_pcm_frames()
      drwav_open_and_read_s16()        -> drwav_open_and_read_pcm_frames_s16()
      drwav_open_and_read_f32()        -> drwav_open_and_read_pcm_frames_f32()
      drwav_open_and_read_s32()        -> drwav_open_and_read_pcm_frames_s32()
      drwav_open_file_and_read_s16()   -> drwav_open_file_and_read_pcm_frames_s16()
      drwav_open_file_and_read_f32()   -> drwav_open_file_and_read_pcm_frames_f32()
      drwav_open_file_and_read_s32()   -> drwav_open_file_and_read_pcm_frames_s32()
      drwav_open_memory_and_read_s16() -> drwav_open_memory_and_read_pcm_frames_s16()
      drwav_open_memory_and_read_f32() -> drwav_open_memory_and_read_pcm_frames_f32()
      drwav_open_memory_and_read_s32() -> drwav_open_memory_and_read_pcm_frames_s32()
      drwav::totalSampleCount          -> drwav::totalPCMFrameCount
  - API CHANGE: 将 drwav_open_and_read_file_*() 重命名为 drwav_open_file_and_read_*()。
  - API CHANGE: 将 drwav_open_and_read_memory_*() 重命名为 drwav_open_memory_and_read_*()。
  - 添加对 smpl 块的内置支持。
  - 添加在初始化时为文件中的每个块触发回调的支持。
    - 通过 drwav_init_ex() 等一系列 API 来启用此功能。
  - 更加健壮地处理无效的 FMT 块。
v0.8.5 - 2018-09-11
  - Const correctness.
  - Fix a potential stack overflow.

v0.8.4 - 2018-08-07
  - Improve 64-bit detection.

v0.8.3 - 2018-08-05
  - Fix C++ build on older versions of GCC.

v0.8.2 - 2018-08-02
  - Fix some big-endian bugs.

v0.8.1 - 2018-06-29
  - Add support for sequential writing APIs.
  - Disable seeking in write mode.
  - Fix bugs with Wave64.
  - Fix typos.

v0.8 - 2018-04-27
  - Bug fix.
  - Start using major.minor.revision versioning.

v0.7f - 2018-02-05
  - Restrict ADPCM formats to a maximum of 2 channels.

v0.7e - 2018-02-02
  - Fix a crash.

v0.7d - 2018-02-01
  - Fix a crash.

v0.7c - 2018-02-01
  - Set drwav.bytesPerSample to 0 for all compressed formats.
  - Fix a crash when reading 16-bit floating point WAV files. In this case dr_wav will output silence for
    all format conversion reading APIs (*_s16, *_s32, *_f32 APIs).
  - Fix some divide-by-zero errors.

v0.7b - 2018-01-22
  - Fix errors with seeking of compressed formats.
  - Fix compilation error when DR_WAV_NO_CONVERSION_API

v0.7a - 2017-11-17
  - Fix some GCC warnings.

v0.7 - 2017-11-04
  - Add writing APIs.

v0.6 - 2017-08-16
  - API CHANGE: Rename dr_* types to drwav_*.
  - Add support for custom implementations of malloc(), realloc(), etc.
  - Add support for Microsoft ADPCM.
  - Add support for IMA ADPCM (DVI, format code 0x11).
  - Optimizations to drwav_read_s16().
  - Bug fixes.

v0.5g - 2017-07-16
  - Change underlying type for booleans to unsigned.

v0.5f - 2017-04-04
  - Fix a minor bug with drwav_open_and_read_s16() and family.

v0.5e - 2016-12-29
  - Added support for reading samples as signed 16-bit integers. Use the _s16() family of APIs for this.
  - Minor fixes to documentation.

v0.5d - 2016-12-28
  - Use drwav_int* and drwav_uint* sized types to improve compiler support.

v0.5c - 2016-11-11
  - Properly handle JUNK chunks that come before the FMT chunk.

v0.5b - 2016-10-23
  - A minor change to drwav_bool8 and drwav_bool32 types.
/*
版本历史记录，列出了每个版本的更改和更新内容
*/

/*
v0.5a - 2016-10-11
  - 修复了由于参数顺序不正确而导致的 drwav_open_and_read() 和相关函数的错误。
  - 提高了 A-law 和 mu-law 的效率。

v0.5 - 2016-09-29
  - API 更改。交换了 drwav_open_and_read*() 中 "channels" 和 "sampleRate" 参数的顺序。这样做的原因是为了
    与 dr_audio 和 dr_flac 保持一致。

v0.4b - 2016-09-18
  - 修正了文档中的拼写错误。

v0.4a - 2016-09-18
  - 修正了一个拼写错误。
  - 将日期格式更改为 ISO 8601（YYYY-MM-DD）

v0.4 - 2016-07-13
  - API 更改。使 onSeek 与 dr_flac 保持一致。
  - API 更改。将 drwav_seek() 重命名为 drwav_seek_to_sample()，以提高清晰度并与 dr_flac 保持一致。
  - 增加了对 Sony Wave64 的支持。

v0.3a - 2016-05-28
  - API 更改。在 onSeek 回调中返回 drwav_bool32 而不是 int。
  - 修复了一个内存泄漏。

v0.3 - 2016-05-22
  - 进行了许多 API 更改以保持一致性。

v0.2a - 2016-05-16
  - 修复了 Linux/GCC 构建问题。

v0.2 - 2016-05-11
  - 增加了作为有符号 32 位 PCM 读取数据的支持，以与 dr_flac 保持一致。

v0.1a - 2016-05-07
  - 修复了在 drwav_open_file() 中的一个错误，即如果加载器初始化失败，则文件句柄将不会关闭。

v0.1 - 2016-05-04
  - 初始版本发布。
*/

/*
此软件可选择以下许可证之一。选择您喜欢的。

===============================================================================
备选方案 1 - 公有领域（www.unlicense.org）
===============================================================================
这是释放到公共领域的自由且不受限制的软件。

任何人都可以自由复制、修改、发布、使用、编译、销售或分发此软件，无论是以源代码形式还是编译后的二进制形式，无论是出于任何目的，商业或非商业，以及通过任何方式。

在承认版权法的司法管辖区域，本软件的作者或作者将此软件的所有版权利益无偿捐赠给公众
*/
# 以下是一段软件许可协议的文本，包括两种不同类型的许可证：Unlicense 和 MIT No Attribution

# Unlicense 许可证部分
# 我们将此软件无偿奉献给公众，对我们的继承人和后继者造成损害。我们打算这种奉献是对此软件在版权法下所有现在和未来权利的永久放弃的公开行为。
# 此软件按原样提供，不附带任何形式的保证，包括但不限于适销性、特定用途适用性和非侵权性的保证。在任何情况下，作者均不对任何索赔、损害或其他责任承担责任，无论是合同诉讼、侵权行为还是其他情况，起因于、由此引起或与此软件或使用或其他交易有关。

# MIT No Attribution 许可证部分
# 版权所有 2020 年 David Reid
# 特此免费授予任何获得本软件及相关文档文件（以下简称“软件”）副本的人，无限制地处理本软件的权限，包括但不限于使用、复制、修改、合并、发布、分发、许可和/或出售本软件的副本，并允许获得本软件的人员这样做。
# 此软件按原样提供，不附带任何形式的保证，包括但不限于适销性、特定用途适用性和非侵权性的保证。在任何情况下，作者或版权持有人均不对任何索赔、损害或其他责任承担责任，无论是合同诉讼、侵权行为还是其他情况，起因于、由此引起或与此软件或使用或其他交易有关。
```