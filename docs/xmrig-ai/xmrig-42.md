# xmrig源码解析 42

# `src/3rdparty/libethash/endian.h`

这段代码是一个预处理指令，主要作用是在编译之前对系统环境进行检查，并为不同的操作系统定义不同的定义。

具体来说，这段代码定义了一些宏，包括：

1. `__MINGW32__` 和 `_WIN32` 定义了 "Microsoft Windows" 操作系统。
2. `__FreeBSD__`、`__DragonFly__` 和 `__NetBSD__` 定义了 FreeBSD、DragonFly 和 NetBSD 操作系统。
3. `__OpenBSD__` 和 `__SVR4__` 定义了 OpenBSD 和 SVR4 操作系统。
4. `__APPLE__` 定义了 macOS 操作系统。
5. `LITTLE_ENDIAN` 定义了 byte 序，值为 1234，表示大端字节小端字节。
6. `BYTE_ORDER` 定义了数据字节序，值为 LITTLE_ENDIAN，表示小端字节大端字节。

这些定义只有在定义了这些操作系统的特定头文件时才会生效，因此，如果定义了这个代码的程序不包含任何特定的操作系统头文件，那么它不会对任何操作系统做出任何贡献。


```cpp
#pragma once

#include <stdint.h>

#if defined(__MINGW32__) || defined(_WIN32)
  # define LITTLE_ENDIAN 1234
  # define BYTE_ORDER    LITTLE_ENDIAN
#elif defined(__FreeBSD__) || defined(__DragonFly__) || defined(__NetBSD__)
  # include <sys/endian.h>
#elif defined(__OpenBSD__) || defined(__SVR4)
  # include <sys/types.h>
#elif defined(__APPLE__)
# include <machine/endian.h>
#elif defined( BSD ) && (BSD >= 199103)
  # include <machine/endian.h>
```

这段代码是一个条件判断，用于根据定义的QNXNTO和LITTLEENDIAN环境变量来定义ENDIAN子系统。

如果不存在ENDIAN子系统定义，则输出特定的代码。否则，根据QNXNTO和LITTLEENDIAN选择适当的ENDIAN子系统定义。这些定义包括ENDIAN子系统的名称、定义的宏名称和可能的选项。

具体来说，如果QNXNTO和LITTLEENDIAN中至少有一个定义了ENDIAN子系统，则使用LITTLEENDIAN定义ENDIAN子系统，否则使用BIGENDIAN定义ENDIAN子系统。如果QNXNTO和LITTLEENDIAN中都没有定义ENDIAN子系统，则使用来自/endian.h头的ENDIAN子系统定义。

此外，如果定义了_WIN32或__APPLE__平台，则可能还需要包含特定的头文件。


```cpp
#elif defined( __QNXNTO__ ) && defined( __LITTLEENDIAN__ )
  # define LITTLE_ENDIAN 1234
  # define BYTE_ORDER    LITTLE_ENDIAN
#elif defined( __QNXNTO__ ) && defined( __BIGENDIAN__ )
  # define BIG_ENDIAN 1234
  # define BYTE_ORDER BIG_ENDIAN
#else
# include <endian.h>
#endif

#if defined(_WIN32)
#include <stdlib.h>
#define ethash_swap_u32(input_) _byteswap_ulong(input_)
#define ethash_swap_u64(input_) _byteswap_uint64(input_)
#elif defined(__APPLE__)
```

这段代码定义了一系列宏，用于对输入的u32和u64数据进行交换操作。

这些宏中，`ethash_swap_u32`和`ethash_swap_u64`用于在定义中使用，而其他宏则是在对应的操作系统中使用。这些宏根据系统是否支持`__FreeBSD__`、`__DragonFly__`、`__NetBSD__`、`__OpenBSD__`中的某些来选择不同的交换函数实现。

`ethash_swap_u32`和`ethash_swap_u64`使用的是`OSSwapInt32`和`OSSwapInt64`函数，属于Linux系统中的`libkern`库，用于在操作系统级别进行内存交换。

`ethash_swap_u32`和`ethash_swap_u64`的实现与Linux系统中的`swap32`和`swap64`函数类似，用于交换32位和64位数据。

`ethash_swap_u32`和`ethash_swap_u64`的实现还与操作系统有关。根据定义中的操作系统，会使用不同的函数实现进行交换。

操作系统函数的实现方式可能因操作系统而异，需要根据具体情况进行解释。


```cpp
#include <libkern/OSByteOrder.h>
#define ethash_swap_u32(input_) OSSwapInt32(input_)
#define ethash_swap_u64(input_) OSSwapInt64(input_)
#elif defined(__FreeBSD__) || defined(__DragonFly__) || defined(__NetBSD__)
#define ethash_swap_u32(input_) bswap32(input_)
#define ethash_swap_u64(input_) bswap64(input_)
#elif defined(__OpenBSD__)
#include <endian.h>
#define ethash_swap_u32(input_) swap32(input_)
#define ethash_swap_u64(input_) swap64(input_)
#else // posix
#include <byteswap.h>
#define ethash_swap_u32(input_) bswap_32(input_)
#define ethash_swap_u64(input_) bswap_64(input_)
#endif


```

这段代码定义了一系列的头文件，用于在目标系统字节序与本地字节序之间进行数据类型的转换和交换。

具体来说，这段代码定义了以下几个函数：

- fix_endian32：用于在目标系统是字节序小端(Little Endian)的情况下，将输入参数 src_ 的字节序列交换成目标系统字节序大端(BIG Endian)的值，即目标系统是字节序大端的情况下，将 src_ 的字节序列交换成目标系统字节序小端的值。
- fix_endian32_same：用于指定输入参数 val_ 的值，它与 fix_endian32 函数中的返回值相同，但不会输出到源代码中。
- fix_endian64：用于在目标系统是字节序小端(Little Endian)的情况下，将输入参数 src_ 的字节序列交换成目标系统字节序大端(BIG Endian)的值，即目标系统是字节序大端的情况下，将 src_ 的字节序列交换成目标系统字节序小端的值。
- fix_endian64_same：用于指定输入参数 val_ 的值，它与 fix_endian64 函数中的返回值相同，但不会输出到源代码中。
- fix_endian_arr32：用于在目标系统是字节序小端(Little Endian)的情况下，对输入参数 arr_ 的字节序列进行转换，将其交换成目标系统字节序大端(BIG Endian)的值。
- fix_endian_arr64：用于在目标系统是字节序小端(Little Endian)的情况下，对输入参数 arr_ 的字节序列进行转换，将其交换成目标系统字节序大端(BIG Endian)的值，并输出这个转换后的值。


```cpp
#if LITTLE_ENDIAN == BYTE_ORDER

#define fix_endian32(dst_ ,src_) dst_ = src_
#define fix_endian32_same(val_)
#define fix_endian64(dst_, src_) dst_ = src_
#define fix_endian64_same(val_)
#define fix_endian_arr32(arr_, size_)
#define fix_endian_arr64(arr_, size_)

#elif BIG_ENDIAN == BYTE_ORDER

#define fix_endian32(dst_, src_) dst_ = ethash_swap_u32(src_)
#define fix_endian32_same(val_) val_ = ethash_swap_u32(val_)
#define fix_endian64(dst_, src_) dst_ = ethash_swap_u64(src_)
#define fix_endian64_same(val_) val_ = ethash_swap_u64(val_)
```

这段代码定义了两个宏，分别固定了32位和64位数据类型的数组。

fix_endian_arr32的作用是固定32位数据类型的数组，具体实现方式是在循环中通过交换数组元素的位置，实现数据的重新排序。

fix_endian_arr64的作用是固定64位数据类型的数组，具体实现方式与fix_endian_arr32类似，只是在循环中使用了64位数据类型，并对元素进行交换。

fix_endian_arr64和fix_endian_arr32的定义中，都包含了一个if语句，如果没有这个if语句，则会输出“endian not supported”的错误信息。


```cpp
#define fix_endian_arr32(arr_, size_) \
  do { \
    for (unsigned i_ = 0; i_ < (size_); ++i_) { \
      arr_[i_] = ethash_swap_u32(arr_[i_]); \
    } \
  } while (0)
#define fix_endian_arr64(arr_, size_) \
  do { \
    for (unsigned i_ = 0; i_ < (size_); ++i_) { \
      arr_[i_] = ethash_swap_u64(arr_[i_]); \
    } \
  } while (0)
#else
# error "endian not supported"
#endif // BYTE_ORDER

```

# `src/3rdparty/libethash/ethash.h`

这段代码是一个文件，属于ethereum协议的的一部分。它声明了这个文件是开源的，允许用户自由地重新分发或修改它，只要遵守GNU通用公共许可证(GPL)的条款。这个文件是在ethash的源代码中的一部分，ethash是一个免费的开源项目，旨在为以太坊网络提供一种快速、安全、可靠的DAPP(智能合约应用程序)运行时状态的同步方案。


```cpp
/*
  This file is part of ethash.

  ethash is free software: you can redistribute it and/or modify
  it under the terms of the GNU General Public License as published by
  the Free Software Foundation, either version 3 of the License, or
  (at your option) any later version.

  ethash is distributed in the hope that it will be useful,
  but WITHOUT ANY WARRANTY; without even the implied warranty of
  MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
  GNU General Public License for more details.

  You should have received a copy of the GNU General Public License
  along with ethash.  If not, see <http://www.gnu.org/licenses/>.
```

这段代码是一个名为ethash.h的文件，其中定义了一些用于设计 ethash 哈希函数的常量和宏。

常量定义了两个哈希函数：一个用于输出修订版号，另一个用于计算数据集大小。宏定义了两个函数，一个是用于从给定的哈希数据集中读取数据，另一个是将数据集大小增加到当前版本的哈希函数中。

该文件的作用是定义和实现两个哈希函数，用于在将来的区块链数据中进行数据的哈希和比较。由于其与区块链的应用密切相关，因此它的实现可能会随着区块链技术的不断发展和改进而有所变化。


```cpp
*/

/** @file ethash.h
* @date 2015
*/
#pragma once

#include <stdint.h>
#include <stdbool.h>
#include <string.h>
#include <stddef.h>

#define ETHASH_REVISION 23
#define ETHASH_DATASET_BYTES_INIT 1073741824U // 2**30
#define ETHASH_DATASET_BYTES_GROWTH 8388608U  // 2**23
```

这段代码定义了一系列常量，用于支持以太坊智能合约的客户端。以下是对这些常量的解释：

1. ETHASH_CACHE_BYTES_INIT：初始化以太坊哈希高速缓存中的字节数。这个值是 2**24，也就是 512 个字节。
2. ETHASH_CACHE_BYTES_GROWTH：每次扩容哈希高速缓存时，需要增加的字节数。这个值是 2**17，也就是 852 个字节。
3. ETHASH_EPOCH_LENGTH：哈希 epoch（记时器）的长度，这个值是 30000 个字节。
4. ETHASH_MIX_BYTES：mix 哈希函数中使用的字节数。这个值是 128 个字节。
5. ETHASH_HASH_BYTES：哈希函数中使用的字节数。这个值是 64 个字节。
6. ETHASH_DATASET_PARENTS：用于表示整个分片网络的哈希数据集的父哈希函数。这个值是 256 个字节。
7. ETHASH_CACHE_ROUNDS：哈希高速缓存中的迭代的轮数。这个值是 3。
8. ETHASH_ACCESSES：哈希函数中单个客户端的访问量。这个值是 64 个字节。
9. ETHASH_DAG_MAGIC_NUM_SIZE：DAG（有向无环图）中的魔法数大小。这个值是 8 个字节。
10. ETHASH_DAG_MAGIC_NUM：用于计算散列值的魔法数。这个值是 0xFEE1DEADBADDCAFE。


```cpp
#define ETHASH_CACHE_BYTES_INIT 1073741824U // 2**24
#define ETHASH_CACHE_BYTES_GROWTH 131072U  // 2**17
#define ETHASH_EPOCH_LENGTH 30000U
#define ETHASH_MIX_BYTES 128
#define ETHASH_HASH_BYTES 64
#define ETHASH_DATASET_PARENTS 256
#define ETHASH_CACHE_ROUNDS 3
#define ETHASH_ACCESSES 64
#define ETHASH_DAG_MAGIC_NUM_SIZE 8
#define ETHASH_DAG_MAGIC_NUM 0xFEE1DEADBADDCAFE

#ifdef __cplusplus
extern "C" {
#endif

```

这段代码定义了一个名为ethash_h256的结构体，其中包含一个长度为32的uint8_t数组b。

接着定义了一个名为ethash_light和ethash_full的的结构体，分别包含一个uint8_t数组和两个ethash_h256类型的成员变量。

然后定义了一个名为ethash_h256_static_init的 macro，该 macro 会接受一个或多个形参，并将它们存储到一个ethash_h256类型的变量中。如果缺少所有形参，则会将数组长度设为32，并将其余成员设为0。

接着定义了一个名为ethash_light_t和ethash_full_t的宏，它们用于声明和声明指针变量。

最后，没有定义任何函数或其他代码结构，因此这些结构体和宏没有实际的功能。


```cpp
/// Type of a seedhash/blockhash e.t.c.
typedef struct ethash_h256 { uint8_t b[32]; } ethash_h256_t;

// convenience macro to statically initialize an h256_t
// usage:
// ethash_h256_t a = ethash_h256_static_init(1, 2, 3, ... )
// have to provide all 32 values. If you don't provide all the rest
// will simply be unitialized (not guranteed to be 0)
#define ethash_h256_static_init(...)			\
	{ {__VA_ARGS__} }

struct ethash_light;
typedef struct ethash_light* ethash_light_t;
struct ethash_full;
typedef struct ethash_full* ethash_full_t;
```

这段代码定义了一个名为`ethash_callback_t`的函数指针类型，它是一个指针变量，可以用来存储函数的实现。接下来定义了一个名为`ethash_return_value_t`的结构体，它包含一个`ethash_h256_t`类型的`result`成员和一个`ethash_h256_t`类型的`mix_hash`成员，还有一个`success`成员和一个`~ethash_compute_cache_nodes_t`类型的参数。

接下来，通过`ethash_compute_cache_nodes_t`类型的参数，指定了要使用的哈希函数的输出是整数类型的指针，并且是一个`unsigned`类型的整数，这个整数的范围应该在0到2^12-1之间。然后定义了一个名为`ethash_light`的函数，它接受一个`unsigned`类型的整数`block_number`作为参数，返回一个`ethash_return_value_t`类型的结构体，包含`result`、`mix_hash`和`success`成员。

最后，通过`ethash_callback_t`类型的指针，给`ethash_light`函数一个指针，这样`ethash_light`函数就可以使用上面定义的哈希函数和`ethash_callback_t`中定义的指针来处理输入参数`block_number`了。


```cpp
typedef int(*ethash_callback_t)(unsigned);

typedef struct ethash_return_value {
	ethash_h256_t result;
	ethash_h256_t mix_hash;
	bool success;
} ethash_return_value_t;

/**
 * Allocate and initialize a new ethash_light handler
 *
 * @param block_number   The block number for which to create the handler
 * @return               Newly allocated ethash_light handler or NULL in case of
 *                       ERRNOMEM or invalid parameters used for @ref ethash_compute_cache_nodes()
 */
```

这段代码定义了两个函数，一个是`ethash_light_new`，另一个是`ethash_light_delete`。这两个函数都与以太坊的智能合约相关。

`ethash_light_new`函数接受一个`uint64_t`类型的块编号作为参数。这个函数的作用是在智能合约中创建一个新的`ethash_light`实例，并将返回给用户。这个实例可以用来调用后面的`ethash_compute_cache_nodes`函数。

`ethash_light_delete`函数接收一个指向`ethash_light`类型的`light`参数。这个函数的作用是释放之前分配给`light`的内存，以便在需要时可以再次分配。

`ethash_compute_cache_nodes`函数的实现比较复杂，但大致可以概括为以下几个步骤：

1. 从`ethash_h256_t`类型的变量`seed`中计算出所需的负载，并将其缓存到`nodes`中。
2. 对于每个需要计算的块，创建一个新的`ethash_light`实例，并将其添加到`cache_nodes`中。
3. 将`cache_nodes`数组长度设置为`cache_size`。
4. 循环遍历所有的`ethash_light`实例，并为每个实例分配一个唯一的标识符`node_index`，以便在需要时查找实例。
5. 将计算出的负载作为参数返回。

总的来说，这两个函数的主要作用是在智能合约中创建和操作`ethash_light`实例，以支持快速查询轻量级数据。


```cpp
ethash_light_t ethash_light_new(uint64_t block_number);
/**
 */
bool ethash_compute_cache_nodes(
    void* nodes,
    uint64_t cache_size,
    ethash_h256_t const* seed
);
/**
 * Frees a previously allocated ethash_light handler
 * @param light        The light handler to free
 */
void ethash_light_delete(ethash_light_t light);
/**
 * Calculate the light client data
 *
 * @param light          The light client handler
 * @param header_hash    The header hash to pack into the mix
 * @param nonce          The nonce to pack into the mix
 * @return               an object of ethash_return_value_t holding the return values
 */
```

这段代码是一个名为`ethash_light_compute`的函数，它是`ethereum_blob_light_compute`的一个轻量级实现。

该函数的作用是返回一个`ethash_light_compute`结构体，它包含以下成员：

1. `light`：一个包含用于高速缓存的数据的`ethash_light_t`类型变量。高速缓存的数据将被散列到给定的`header_hash`，然后与给定的`nonce`一起用于生成新的DAG计算。
2. `callback`：一个带有`ethash_callback_t`类型的回调函数，它接受一个`unsigned<64>`类型的参数，用于表示DAG生成的进度。如果一切顺利，该函数将返回0，否则将继续生成DAG。
3. `errnok`：一个`null_ptr<>`类型的变量，用于在编译时检查是否传递了有效的参数。

该函数的实现类似于`ethereum_blob_light_compute`函数，但是使用了`ethash_light_t`代替了`ethereum_blob_t`，因为`ethereum_blob_light_compute`函数没有相应的`null_t`签名。


```cpp
ethash_return_value_t ethash_light_compute(
	ethash_light_t light,
	ethash_h256_t const header_hash,
	uint64_t nonce
);

/**
 * Allocate and initialize a new ethash_full handler
 *
 * @param light         The light handler containing the cache.
 * @param callback      A callback function with signature of @ref ethash_callback_t
 *                      It accepts an unsigned with which a progress of DAG calculation
 *                      can be displayed. If all goes well the callback should return 0.
 *                      If a non-zero value is returned then DAG generation will stop.
 *                      Be advised. A progress value of 100 means that DAG creation is
 *                      almost complete and that this function will soon return succesfully.
 *                      It does not mean that the function has already had a succesfull return.
 * @return              Newly allocated ethash_full handler or NULL in case of
 *                      ERRNOMEM or invalid parameters used for @ref ethash_compute_full_data()
 */
```

该代码定义了两个函数，分别为ethash_full_delete和ethash_full_new。

ethash_full_delete函数用于释放之前分配的ethash_full处理器的内存空间。该函数接收一个ethash_full类型的参数full，然后执行操作。

ethash_full_new函数则创建一个新的ethash_full处理器实例，并将传递给该实例的ethash_light_t和ethash_callback_t类型的参数作为构造函数的实参。构造函数返回一个ethash_full类型的对象，该对象将作为ethash_full_new函数的返回值。

ethash_full_new函数的作用是创建一个新的ethash_full实例，该实例将调用传递给它的ethash_light_t和ethash_callback_t类型的构造函数，并返回该实例。

这两个函数一起工作， ethash_full_new函数将分配内存并调用构造函数，而ethash_full_delete函数将在构造函数返回后释放分配的内存。


```cpp
ethash_full_t ethash_full_new(ethash_light_t light, ethash_callback_t callback);

/**
 * Frees a previously allocated ethash_full handler
 * @param full    The light handler to free
 */
void ethash_full_delete(ethash_full_t full);
/**
 * Calculate the full client data
 *
 * @param full           The full client handler
 * @param header_hash    The header hash to pack into the mix
 * @param nonce          The nonce to pack into the mix
 * @return               An object of ethash_return_value to hold the return value
 */
```

这段代码是一个以太坊智能合约中的函数，它的作用是计算指定epoch的seedhash。

具体来说，这段代码实现了以下几个功能：

1. `ethash_full_compute`函数的参数是一个`ethash_full`类型和一个`ethash_h256`类型的变量和一个`uint64_t`类型的变量`nonce`。这个函数的作用是获取指定epoch的完整dag数据，并返回一个`const*`类型的指针。
2. `ethash_full_dag`函数的参数是一个`ethash_full`类型和一个`ethash_h256`类型的变量`header_hash`，这个函数的作用是获取指定epoch的dag data，并返回一个`const*`类型的指针。
3. `ethash_full_dag_size`函数的参数是一个`ethash_full`类型，这个函数的作用是计算指定epoch的dag data的大小，并返回一个`uint64_t`类型的值。
4. `ethash_compute_weak`函数的参数是一个`ethash_full`类型和一个`uint64_t`类型的变量`weak_hash`，这个函数的作用是生成指定epoch的 weak hash，并返回一个`uint64_t`类型的值。


```cpp
ethash_return_value_t ethash_full_compute(
	ethash_full_t full,
	ethash_h256_t const header_hash,
	uint64_t nonce
);
/**
 * Get a pointer to the full DAG data
 */
void const* ethash_full_dag(ethash_full_t full);
/**
 * Get the size of the DAG data
 */
uint64_t ethash_full_dag_size(ethash_full_t full);

/**
 * Calculate the seedhash for a given epoch
 */
```

这两行代码是使用 Ethereum 的 `ethash_h256_t` 和 `ethash_get_seedhash` 函数，用于在以太坊的提议权路上生成随机数种子。

`ethash_h256_t` 是 `ethereum_h256_t` 类型的别名，表示使用 Ethereum 2.0 的哈希函数。 `ethash_get_seedhash` 函数接受一个 `uint64_t` 类型的epoch，作为输入参数，返回一个 256 字节的种子哈希。

这两行代码一起作用于在 `keccakf800` 函数中生成随机数种子。 `keccakf800` 函数是一个自定义的哈希函数，接受一个 256 字节的输入参数 `state`。

这两行代码的作用是生成一个随机数种子，用于开始创建一个区块。


```cpp
ethash_h256_t ethash_get_seedhash(uint64_t epoch);

/**
 * KeccakF800 for ProgPoW
 */
void ethash_keccakf800(uint32_t state[25]);

#ifdef __cplusplus
}
#endif

```

# `src/3rdparty/libethash/ethash_internal.c`

这段代码是一个名为ethash.c的C文件，它是ethereum区块链上的一个智能合约。这段代码定义了一个名为gethash的函数，该函数会在智能合约中调用，并在需要时执行。

gethash的作用是帮助开发人员快速地计算目标值，通过使用一种高效的方式将输入参数（例如时间和内存）转换为智能合约中的合约地址。这个函数会根据输入的选项（包括要查找的地址的哈希值）自动调用，并在需要时动态地更新，从而使得合约中的计算更加高效。


```cpp
/*
  This file is part of ethash.

  ethash is free software: you can redistribute it and/or modify
  it under the terms of the GNU General Public License as published by
  the Free Software Foundation, either version 3 of the License, or
  (at your option) any later version.

  ethash is distributed in the hope that it will be useful,
  but WITHOUT ANY WARRANTY; without even the implied warranty of
  MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.	See the
  GNU General Public License for more details.

  You should have received a copy of the GNU General Public License
  along with cpp-ethereum.	If not, see <http://www.gnu.org/licenses/>.
```

这是一个C语言文件，包含多个头文件和函数。以下是文件的一些摘要：

这个文件是用来在服务器端计算RANDOM数的。

作者：Tim Hughes和Matthew Wampler-Doty。

日期：2015年。

头文件：

```cpp
#include <assert.h>
#include <inttypes.h>
#include <stddef.h>
#include <errno.h>
#include <math.h>
#include <stdlib.h>
```

函数：

```cpp
// 计算指定数值的RANDOM数
int random_int(int min, int max) {
   return (rand() % (max - min + 1)) + min;
}

// 计算指定随机数在不同时间戳上的RANDOM数
int random_int_file(const char *filename, int min, int max) {
   FILE *fp = fopen(filename, "r");
   if (!fp) {
       perror("random_int_file");
       return 0;
   }

   int hashes[2];
   for (int i = 0; i < 2; i++) {
       hashes[i] = i;
   }

   int seen = 0;
   int hash = 0;

   for (int i = 0; i < max; i++) {
       if (i == min || hashes[random_int(0, 4294967295) % 2] == hash) {
           continue;
       }

       int val = hashes[random_int(0, 4294967295) % 2];
       hash = val;
       if (i == min) {
           for (int j = 0; j < 2; j++) {
               hashes[j] = j;
           }
       } else {
           seen++;
       }
   }

   int index = 0;
   int i = 0;
   while (i < min && !hashes[i % 2] == hash) {
       int val = hashes[i % 2] % 2;
       hash = val;
       i++;
       if (i == min) {
           for (int j = 0; j < 2; j++) {
               hashes[j] = j;
           }
       } else {
           seen++;
       }
   }

   return i;
}
```

这个函数是 `random_int_file`，它会读取一个随机数文件，给定一个最小值和一个最大值，然后返回指定时间戳上的随机整数。

它包含一个 `random_int` 函数，用于计算指定数值的RANDOM数。这个函数使用了 `rand` 和 `%` 运算符，会在指定时间范围内随机生成一个数字。


```cpp
*/
/** @file internal.c
* @author Tim Hughes <tim@twistedfury.com>
* @author Matthew Wampler-Doty
* @date 2015
*/

#include <assert.h>
#include <inttypes.h>
#include <stddef.h>
#include <errno.h>
#include <math.h>
#include <stdlib.h>
#include "ethash.h"
#include "fnv.h"
```

这段代码包括以下几个部分：

1. 引入了一些头文件：endian.h，ethash_internal.h，data_sizes.h，base/crypto/sha3.h
2. 引入了一个定义：_M_X64，表示当前处理器架构是否支持x86_64架构。
3. 引入了一个宏：__GNUC__，表示当前的编译器是否为GNU编译器。
4. 在if语句中，通过_mm_prefetch函数，对大整数进行预取值，其中kp_prefetch是一个内部函数，它接受一个整数参数x，并返回一个整数类型的值。这个函数的作用是提高并行度，通过使用_MM_HINT_T0 hint来让CPU硬件进行预取值操作。
5. 在if语句中，定义了一个名为kp_prefetch的内部函数，这个函数是在_mm_prefetch函数的基础上进行了扩展，可以适应不同的输入类型。


```cpp
#include "endian.h"
#include "ethash_internal.h"
#include "data_sizes.h"
#include "base/crypto/sha3.h"

#if defined(_M_X64) || defined(__x86_64__) || defined(__SSE2__)
	#ifdef __GNUC__
		#include <x86intrin.h>
	#else
		#include <intrin.h>
	#endif

	#define kp_prefetch(x) _mm_prefetch((x), _MM_HINT_T0);
#else
	#define kp_prefetch(x)
```

这两行代码是用于计算基于SHA3-256和SHA3-512哈希算法的输出缓存大小。

具体来说，这两行代码会根据输入的块号，使用定义好的宏函数SHA3_256和SHA3_512，将输入的块号和硫 cash 生成物进行哈希运算，并将哈希结果存储到输出缓存中。其中，第一个宏函数使用256位哈希算法，第二个宏函数使用512位哈希算法。

输出缓存的大小是由ethash_get_datasize函数计算得出的。该函数会根据输入的块号，计算出该块需要的数据大小，并将该大小存储到输出缓存中。


```cpp
#endif

#define SHA3_256(a, b, c) sha3_HashBuffer(256, SHA3_FLAGS_KECCAK, b, c, a, 32)
#define SHA3_512(a, b, c) sha3_HashBuffer(512, SHA3_FLAGS_KECCAK, b, c, a, 64)

uint64_t ethash_get_datasize(uint64_t const block_number)
{
	assert(block_number / ETHASH_EPOCH_LENGTH < 2048);
	return dag_sizes[block_number / ETHASH_EPOCH_LENGTH];
}

uint64_t ethash_get_cachesize(uint64_t const block_number)
{
	assert(block_number / ETHASH_EPOCH_LENGTH < 2048);
	return cache_sizes[block_number / ETHASH_EPOCH_LENGTH];
}

```

Yes, you are correct. The function you provided is the "SeqMemoHash" function from the "memohash" library, which is a Memoization-based implementation of the SHA-256 hash function.

It appears that you are trying to compare the output of this function with the output of the "SHA256" function from the C++ standard library. However, the "SHA256" function from the standard library does not have a "SeqMemoHash" implementation, so it cannot be used to compare the output of the "SeqMemoHash" function with the output of the "SHA256" function.

If you want to compare the output of the "SeqMemoHash" function with the output of the "SHA256" function from the standard library, you can use the "sha256" function from the "shcrypt" library. This function is a C implementation of the SHA-256 hash function that should have the same behavior as the "SHA256" function from the standard library.


```cpp
// Follows Sergio's "STRICT MEMORY HARD HASHING FUNCTIONS" (2014)
// https://bitslog.files.wordpress.com/2013/12/memohash-v0-3.pdf
// SeqMemoHash(s, R, N)
bool ethash_compute_cache_nodes(
	void* nodes_ptr,
	uint64_t cache_size,
	ethash_h256_t const* seed
)
{
	if (cache_size % sizeof(node) != 0) {
		return false;
	}
	uint32_t const num_nodes = (uint32_t) (cache_size / sizeof(node));

	node* nodes = (node*)nodes_ptr;
	SHA3_512(nodes[0].bytes, (uint8_t*)seed, 32);

	for (uint32_t i = 1; i != num_nodes; ++i) {
		SHA3_512(nodes[i].bytes, nodes[i - 1].bytes, 64);
	}

	for (uint32_t j = 0; j != ETHASH_CACHE_ROUNDS; j++) {
		for (uint32_t i = 0; i != num_nodes; i++) {
			uint32_t const idx = nodes[i].words[0] % num_nodes;
			node data;
			data = nodes[(num_nodes - 1 + i) % num_nodes];
			for (uint32_t w = 0; w != NODE_WORDS; ++w) {
				data.words[w] ^= nodes[idx].words[w];
			}
			SHA3_512(nodes[i].bytes, data.bytes, sizeof(data));
		}
	}

	// now perform endian conversion
	fix_endian_arr32(nodes->words, num_nodes * NODE_WORDS);
	return true;
}

```

这段代码是一个以太坊DAG（有向无环图）算法中的函数，它的作用是计算指定节点及其父节点构成的有向无环图的哈希值。

具体来说，这段代码的功能如下：

1. 计算父节点数量：根据light的哈希值计算父节点数量，然后将父节点数量作为参数传递给函数。
2. 遍历父节点并计算哈希：初始化计算出的父节点，然后从父节点数量中选择一个，将该节点的字节序作为输入，将哈希算法计算得到的哈希值作为输出，同时将当前节点指针设置为输入哈希的节点。
3. 对节点进行SHA3-512哈希：使用SHA3-512哈希算法对节点进行哈希计算，并将哈希值存储到输出变量ret中。

此外，如果定义了_M_X64，并启用了SSE（安全子系统扩展），则在计算过程中可以使用_mm_set1_epi32函数来设置初始FNV_PRIME。


```cpp
void ethash_calculate_dag_item(
	node* const ret,
	uint32_t node_index,
	uint32_t num_parents,
	ethash_light_t const light
)
{
	uint32_t num_parent_nodes = (uint32_t) (light->cache_size / sizeof(node));
	node const* cache_nodes = (node const *) light->cache;
	node const* init = &cache_nodes[node_index % num_parent_nodes];
	memcpy(ret, init, sizeof(node));
	ret->words[0] ^= node_index;
	SHA3_512(ret->bytes, ret->bytes, sizeof(node));
#if defined(_M_X64) && ENABLE_SSE
	__m128i const fnv_prime = _mm_set1_epi32(FNV_PRIME);
	__m128i xmm0 = ret->xmm[0];
	__m128i xmm1 = ret->xmm[1];
	__m128i xmm2 = ret->xmm[2];
	__m128i xmm3 = ret->xmm[3];
```

这段代码是一个C语言中的预处理指令，其作用是在编译时检查特定条件是否成立，如果没有成立，则执行一些计算操作。

具体来说，这段代码的作用是：

1. 遍历一个数组ret，其中ret的长度为数组长度num_parents的整数倍，同时递增i。
2. 对于每个i，计算出parent_index，即从ret数组中节点编号为i的元素开始，到num_parent_nodes数组长度-1的元素的索引。
3. 计算出parent节点，即ret数组中节点编号为parent_index的元素。
4. 如果定义了_M_X64并且 enable_sse，则在计算SSE向量对大端序的计算结果，其他则直接计算结果。
5. 如果需要将计算结果存储到ret数组中，则需要先将结果存储到变量xmm0,xmm1,xmm2,xmm3中，然后将xmm0,xmm1,xmm2,xmm3复制回ret数组中。

如果编译时满足某些条件，则这段代码会执行第2步到第5步的计算操作。


```cpp
#endif

	for (uint32_t i = 0; i != num_parents; ++i) {
		uint32_t parent_index = fnv_hash(node_index ^ i, ret->words[i % NODE_WORDS]) % num_parent_nodes;
		node const *parent = &cache_nodes[parent_index];

#if defined(_M_X64) && ENABLE_SSE
		{
			xmm0 = _mm_mullo_epi32(xmm0, fnv_prime);
			xmm1 = _mm_mullo_epi32(xmm1, fnv_prime);
			xmm2 = _mm_mullo_epi32(xmm2, fnv_prime);
			xmm3 = _mm_mullo_epi32(xmm3, fnv_prime);
			xmm0 = _mm_xor_si128(xmm0, parent->xmm[0]);
			xmm1 = _mm_xor_si128(xmm1, parent->xmm[1]);
			xmm2 = _mm_xor_si128(xmm2, parent->xmm[2]);
			xmm3 = _mm_xor_si128(xmm3, parent->xmm[3]);

			// have to write to ret as values are used to compute index
			ret->xmm[0] = xmm0;
			ret->xmm[1] = xmm1;
			ret->xmm[2] = xmm2;
			ret->xmm[3] = xmm3;
		}
```

这段代码是一个 C 语言的函数，它定义了一个名为 "my_shader" 的函数。这个函数的作用是计算一个名为 "node" 的结构体中的 "bytes" 字段的后验位的哈希值。

函数中包含一个 else 分支，如果这个分支没有被定义，那么会执行函数体中的代码。在 if 分支中，首先会定义一个名为 "w" 的无符号整数变量，然后使用一个 for 循环来遍历 "parent->words" 数组，其中 "parent->words" 可能是一个指向其他结构体的指针，这个结构体中的 "words" 字段存储了哈希表。在循环中，"w" 变量会计算每个 "words" 数组中的元素的哈希值，并将结果存储在 "ret->words" 数组中对应 "w" 变量的位置。

接下来会调用一个名为 "SHA3_512" 的函数，这个函数的作用是生成一个固定长度的哈希值，并将其存储在 "ret->bytes" 数组中。最后，函数返回哈希值。


```cpp
#else
		{
			for (unsigned w = 0; w != NODE_WORDS; ++w) {
				ret->words[w] = fnv_hash(ret->words[w], parent->words[w]);
			}
		}
#endif
	}
	SHA3_512(ret->bytes, ret->bytes, sizeof(node));
}

static inline uint32_t fast_mod(uint64_t a, uint64_t d, uint64_t r, uint64_t i, uint64_t s)
{
	const uint32_t q = ((a + i) * r) >> s;
	return a - q * d;
}

```

该函数名为 `ethash_calculate_dag_item_opt`，其作用是计算给定 `num_parents` 和 `light` 属性的 `ethash` 哈希的 `dag_item` 选项。

具体来说，该函数接受一个指向 `node` 类型变量的 `ret` 参数，一个表示节点索引的 `node_index` 参数，一个表示哈希树中节点父数量的 `num_parents` 参数，和一个表示哈希树中 `light` 属性的 `light` 参数。

函数内部首先定义了一个名为 `cache_nodes` 的变量，该变量是一个指向哈希树中 `light` 属性所对应的节点 const 类型的指针。然后定义了一个名为 `init` 的变量，该变量指向 `cache_nodes[fast_mod(node_index, light->num_parent_nodes, light->reciprocal, light->increment, light->shift)]`，其中 `fast_mod` 函数的作用是将 `node_index` 和 `light` 属性的值进行一些计算，然后将结果作为参数传递给 `fast_mod` 函数，以获取一个与 `node_index` 灯武hash 相关的哈希值。

接下来，函数使用一个 for 循环遍历 `num_parents` 次幂，并且计算 `light` 哈希的 `dag_item` 选项，每次循环将计算出的 `parent_index` 存储到 `ret->words[i % NODE_WORDS]` 中，其中 `NODE_WORDS` 是哈希树中节点数量的二进制位数。

最后，函数再次使用一个 for 循环遍历 `num_parents` 次幂，并且计算 `light` 哈希的 `dag_item` 选项，并使用 `SHA3_512` 函数对结果进行哈希运算，并将结果存储到 `ret->bytes` 数组中。


```cpp
void ethash_calculate_dag_item_opt(
	node* const ret,
	uint32_t node_index,
	uint32_t num_parents,
	ethash_light_t const light
)
{
	node const* cache_nodes = (node const*)light->cache;
	node const* init = &cache_nodes[fast_mod(node_index, light->num_parent_nodes, light->reciprocal, light->increment, light->shift)];
	memcpy(ret, init, sizeof(node));
	ret->words[0] ^= node_index;
	SHA3_512(ret->bytes, ret->bytes, sizeof(node));

	for (uint32_t i = 0; i != num_parents; ++i) {
		uint32_t parent_index = fast_mod(fnv_hash(node_index ^ i, ret->words[i % NODE_WORDS]), light->num_parent_nodes, light->reciprocal, light->increment, light->shift);
		node const* parent = &cache_nodes[parent_index];
		for (unsigned w = 0; w != NODE_WORDS; ++w) {
			ret->words[w] = fnv_hash(ret->words[w], parent->words[w]);
		}
	}
	SHA3_512(ret->bytes, ret->bytes, sizeof(node));
}

```

以上是一个用 Rust 实现的memcpy函数，接受两个参数：一个是指针 light，它存储了一个哈希表；另一个是一个整数 num_parents，表示哈希表的深度。哈希表存储了一个节点结构，每个节点结构包含一个指向其父节点的指针、一个布尔类型的数据、以及一个固定长度的字节数组。

这个函数的主要思想是复制哈希表 light，并将其作为子哈希表的根节点，然后将其余节点的数据复制到哈希表中。在复制的过程中，对于每个子哈希表节点，首先将其父节点存储在子哈希表的根节点中，然后将其数据复制到哈希表中。最后，对于每个节点，将其数据和叶子节点进行哈希，并将其结果存储在哈希表中。


```cpp
void ethash_calculate_dag_item4_opt(
	node* ret,
	uint32_t node_index,
	uint32_t num_parents,
	ethash_light_t const light
)
{
	node const* cache_nodes = (node const*)light->cache;

	for (size_t i = 0; i < 4; ++i) {
		node const* init = &cache_nodes[fast_mod(node_index + i, light->num_parent_nodes, light->reciprocal, light->increment, light->shift)];
		memcpy(ret + i, init, sizeof(node));
		ret[i].words[0] ^= node_index + i;
		SHA3_512(ret[i].bytes, ret[i].bytes, sizeof(node));
	}

	for (uint32_t i = 0; i != num_parents; ++i) {
		node const* parent[4];

		for (uint32_t j = 0; j < 4; ++j) {
			const uint32_t parent_index = fast_mod(fnv_hash((node_index + j) ^ i, ret[j].words[i % NODE_WORDS]), light->num_parent_nodes, light->reciprocal, light->increment, light->shift);
			parent[j] = &cache_nodes[parent_index];
			kp_prefetch(parent[j]);
		}

		for (unsigned w = 0; w != NODE_WORDS; ++w) ret[0].words[w] = fnv_hash(ret[0].words[w], parent[0]->words[w]);
		for (unsigned w = 0; w != NODE_WORDS; ++w) ret[1].words[w] = fnv_hash(ret[1].words[w], parent[1]->words[w]);
		for (unsigned w = 0; w != NODE_WORDS; ++w) ret[2].words[w] = fnv_hash(ret[2].words[w], parent[2]->words[w]);
		for (unsigned w = 0; w != NODE_WORDS; ++w) ret[3].words[w] = fnv_hash(ret[3].words[w], parent[3]->words[w]);
	}

	for (size_t i = 0; i < 4; ++i) {
		SHA3_512(ret[i].bytes, ret[i].bytes, sizeof(node));
	}
}

```

该函数是一个以太坊智能合约中的一个名为 `ethash_compute_full_data` 的函数。它的作用是计算给定数据的散列值，并将其返回。

函数的参数包括四个：

- `mem`：要计算散列值的内存区域，指针类型，用于保存计算过程中的数据。
- `full_size`：要计算的数据长度，以字节为单位。该参数用于保证函数使用正确数组长度。
- `light`：用于指示输出数据的类型，可以是 `ETHASH_LIGHT_IDX`、`ETHASH_LIGHT_PARTIAL_DATA` 或 `ETHASH_LIGHT_ENCRYPTED_DATA`。
- `callback`：一个 `ethash_callback_t` 类型的指针，用于指示函数在计算过程中执行的回调函数。该参数必须是一个 `void*` 类型的指针，用于存储计算结果的内存空间。

函数的主要步骤如下：

1. 检查输入参数中给出的 `full_size` 是否是 `sizeof(uint32_t)` 的倍数。如果不是，函数返回 `false`。
2. 将 `max_n` 设置为输入参数 `full_size` 除以 `sizeof(node)` 的商，用于将散列值数据分配到适当的节点上。
3. 将 `progress_change` 设置为计算过程中剩余数据的百分比，有助于确保计算过程具有可预测性。
4. 使用给定的 `callback` 函数在计算过程中执行所需的回调操作。
5. 调用 `ethash_calculate_dag_item` 函数计算数据集中的所有节点。
6. 如果所有的回调函数都返回 `true`，函数返回 `true`。

该函数可以作为智能合约中的一个函数，用于将数据集中的所有节点计算为散列值，并将其存储到给定的内存区域中。


```cpp
bool ethash_compute_full_data(
	void* mem,
	uint64_t full_size,
	ethash_light_t const light,
	ethash_callback_t callback
)
{
	if (full_size % (sizeof(uint32_t) * MIX_WORDS) != 0 ||
		(full_size % sizeof(node)) != 0) {
		return false;
	}
	uint32_t const max_n = (uint32_t)(full_size / sizeof(node));
	node* full_nodes = (node*) mem;
	double const progress_change = 1.0f / max_n;
	double progress = 0.0f;
	// now compute full nodes
	for (uint32_t n = 0; n != max_n; ++n) {
		if (callback &&
			n % (max_n / 100) == 0 &&
			callback((unsigned int)(ceil(progress * 100.0f))) != 0) {

			return false;
		}
		progress += progress_change;
		ethash_calculate_dag_item(&(full_nodes[n]), n, ETHASH_DATASET_PARENTS, light);
	}
	return true;
}

```

I'm sorry, I am not sure what you are asking. Could you please provide more context or clarify your question?


```cpp
static bool ethash_hash(
	ethash_return_value_t* ret,
	node const* full_nodes,
	ethash_light_t const light,
	uint64_t full_size,
	ethash_h256_t const header_hash,
	uint64_t const nonce
)
{
	if (full_size % MIX_WORDS != 0) {
		return false;
	}

	// pack hash and nonce together into first 40 bytes of s_mix
	assert(sizeof(node) * 8 == 512);
	node s_mix[MIX_NODES + 1];
	memcpy(s_mix[0].bytes, &header_hash, 32);
	fix_endian64(s_mix[0].double_words[4], nonce);

	// compute sha3-512 hash and replicate across mix
	SHA3_512(s_mix->bytes, s_mix->bytes, 40);
	fix_endian_arr32(s_mix[0].words, 16);

	node* const mix = s_mix + 1;
	for (uint32_t w = 0; w != MIX_WORDS; ++w) {
		mix->words[w] = s_mix[0].words[w % NODE_WORDS];
	}

	unsigned const page_size = sizeof(uint32_t) * MIX_WORDS;
	unsigned const num_full_pages = (unsigned) (full_size / page_size);

	for (unsigned i = 0; i != ETHASH_ACCESSES; ++i) {
		uint32_t const index = fnv_hash(s_mix->words[0] ^ i, mix->words[i % MIX_WORDS]) % num_full_pages;

		for (unsigned n = 0; n != MIX_NODES; ++n) {
			node const* dag_node;
			node tmp_node;
			if (full_nodes) {
				dag_node = &full_nodes[MIX_NODES * index + n];
			} else {
				ethash_calculate_dag_item(&tmp_node, index * MIX_NODES + n, ETHASH_DATASET_PARENTS, light);
				dag_node = &tmp_node;
			}

```

这段代码 checks if the system supports SSE (Stream S 向量 Extensions) and defines a function that takes advantage of it if it does.

The code defines a macro function called `__notnull(mix[n].words[w])` which returns a memory location of type `m128i` (a 128-bit integer).

The macro is defined with two conditions that must be met for it to be true:

1. `defined(_M_X64)`: This checks if the `_M_X64` preprocessor symbol is defined. If it is, the code inside the macro can access the `m64` type member in `m128i`.
2. `ENABLE_SSE`: This checks if the SSE extension is enabled in the system. If it is, the macro defines the `__notnull` macro with the `m128i` type member.

If the conditions are met, the `__notnull(mix[n].words[w])` macro will return a memory location of type `m128i` for the `words` array element of the `mix` array at index `w`.


```cpp
#if defined(_M_X64) && ENABLE_SSE
			{
				__m128i fnv_prime = _mm_set1_epi32(FNV_PRIME);
				__m128i xmm0 = _mm_mullo_epi32(fnv_prime, mix[n].xmm[0]);
				__m128i xmm1 = _mm_mullo_epi32(fnv_prime, mix[n].xmm[1]);
				__m128i xmm2 = _mm_mullo_epi32(fnv_prime, mix[n].xmm[2]);
				__m128i xmm3 = _mm_mullo_epi32(fnv_prime, mix[n].xmm[3]);
				mix[n].xmm[0] = _mm_xor_si128(xmm0, dag_node->xmm[0]);
				mix[n].xmm[1] = _mm_xor_si128(xmm1, dag_node->xmm[1]);
				mix[n].xmm[2] = _mm_xor_si128(xmm2, dag_node->xmm[2]);
				mix[n].xmm[3] = _mm_xor_si128(xmm3, dag_node->xmm[3]);
			}
			#else
			{
				for (unsigned w = 0; w != NODE_WORDS; ++w) {
					mix[n].words[w] = fnv_hash(mix[n].words[w], dag_node->words[w]);
				}
			}
```

这段代码是一个 C 语言的函数，它实现了一个压缩混合（mix）哈希的过程。混合哈希算法通常将输入数据压缩成一个哈希值，以便在存储和传输时进行保护。

具体来说，这段代码执行以下操作：

1. 如果已经定义了一个哈希表（可能已经定义好了），那么先将输入数据（32 字节的输入数据和 64 字节的哈希值）复制到哈希表中。
2. 如果哈希表还没有被定义，那么定义一个哈希表，类型为 struct HashTable { 32Uc words : 32Uc }，然后使用循环将输入数据中的每个 32 字节的哈希值计算出来，并将计算结果存储到哈希表中。
3. 接下来，使用循环将哈希表中的每个哈希值进行压缩，压缩方法是将哈希值的第一部分（也就是 32 字节的输入数据）与一个固定值（在这里是 FNV_PRIME）进行异或运算，并将结果的哈希值再次与 FNV_PRIME 异或运算，得到一个新的哈希值。然后将新的哈希值存储回哈希表中。
4. 最后，使用 FixEndian 函数对输入数据和哈希值进行字节对齐，然后将哈希值和输入数据一起发送给客户端。

整个过程旨在实现一个具有高效率、低碰撞的压缩混合哈希。


```cpp
#endif
		}

	}

	// compress mix
	for (uint32_t w = 0; w != MIX_WORDS; w += 4) {
		uint32_t reduction = mix->words[w + 0];
		reduction = reduction * FNV_PRIME ^ mix->words[w + 1];
		reduction = reduction * FNV_PRIME ^ mix->words[w + 2];
		reduction = reduction * FNV_PRIME ^ mix->words[w + 3];
		mix->words[w / 4] = reduction;
	}

	fix_endian_arr32(mix->words, MIX_WORDS / 4);
	memcpy(&ret->mix_hash, mix->bytes, 32);
	// final Keccak hash
	SHA3_256(&ret->result, s_mix->bytes, 64 + 32); // Keccak-256(s + compressed_mix)
	return true;
}

```

该函数的作用是创建一个带有用户定义输出哈希值的ethash_h256实例，并使用输入的header_hash、nonce和mix_hash来生成新的输出哈希值。

具体来说，函数的实现分为以下几步：

1. 将输入的header_hash和nonce转换为字节数组，并将其存储在buf数组中。
2. 将输入的nonce转换为8字节整数，并将其存储在buf数组中。
3. 使用SHA3-512算法对妞妞哈希值和nonce进行哈希，并将结果存储在buf数组中。
4. 使用SHA3-256算法对妞妞哈希值和上面生成的哈希值进行哈希，并将结果存储在return_hash变量中。

ethash_quick_hash函数的输出结果是一个ethash_h256实例，它包含了用户定义的输出哈希值，可以用于任何需要输出哈希值的函数中。


```cpp
void ethash_quick_hash(
	ethash_h256_t* return_hash,
	ethash_h256_t const* header_hash,
	uint64_t nonce,
	ethash_h256_t const* mix_hash
)
{
	uint8_t buf[64 + 32];
	memcpy(buf, header_hash, 32);
	fix_endian64_same(nonce);
	memcpy(&(buf[32]), &nonce, 8);
	SHA3_512(buf, buf, 40);
	memcpy(&(buf[64]), mix_hash, 32);
	SHA3_256(return_hash, buf, 64 + 32);
}

```

以下是以上两个函数的作用及其它周围的代码。

第一个函数 `ethash_h256_t ethash_get_seedhash(uint64_t epoch)` 用于从给定的 epoch 值中计算 seed hash。函数的核心部分是一个 for 循环，该循环使用 SHA3-256 函数对 epoch 值进行哈希，并将其存储在 ret 变量中。函数返回 ret 变量，其中包含计算得到的 seed hash。

第二个函数 `bool ethash_quick_check_difficulty(ethash_h256_t const* header_hash, uint64_t const nonce, ethash_h256_t const* mix_hash, ethash_h256_t const* boundary)` 用于检查给定的 header hash、nonce 和 mix-hash 是否能够产生正确的 difficulty 值。函数的核心部分是一个 if 语句，该语句首先调用第一个 `ethash_h256_t ethash_get_seedhash` 函数来计算 seed hash，然后使用 ethash_check_difficulty 函数检查 the seed hash 是否符合给定的 boundary 值。函数返回 true，如果种子哈希符合要求，否则返回 false。

以下是 surrounding 代码，显示了如何使用这两个函数：
```cppc
#include <stdint.h>
#include <ethash.h>

// ethash_h256_t 和 uint64_t 是从 Ethash 库中定义的别名。
// SHA3_256 是从 Ethash 库中定义的函数，用于执行输入哈希。
// ethash_h256_t ethash_get_seedhash(uint64_t epoch) 的实现与上述说明相同。
// ethash_quick_check_difficulty 的实现与上述说明相同。

int main(int argc, char *argv[])
{
   uint64_t epoch = 330001000; // 以太坊网络的 epoch 值
   uint64_t nonce; // 非交易便士
   uint64_t mix_hash[4]; // 从 0 到 3 的四个哈希值
   uint64_t boundary; // 边界值，这里是 0xFFFFFFFFFFF
   ethash_h256_t header_hash; // 用于區分高耗能交易和普通交易的 header hash
   bool result;

   result = ethash_quick_check_difficulty(
       header_hash, nonce, mix_hash, boundary);
   if (result == true)
       printf("The difficulty is %" PRIu64 "difficulty\n", result);
   else
       printf("The difficulty is %" PRIu64 "easy\n", result);

   return 0;
}
```
此代码示例演示了如何使用 `ethash_h256_t` 和 `uint64_t` 别来计算以太坊网络的 epoch 值。


```cpp
ethash_h256_t ethash_get_seedhash(uint64_t epoch)
{
	ethash_h256_t ret;
	ethash_h256_reset(&ret);
	for (uint32_t i = 0; i < epoch; ++i)
		SHA3_256(&ret, (uint8_t*)&ret, 32);
	return ret;
}

bool ethash_quick_check_difficulty(
	ethash_h256_t const* header_hash,
	uint64_t const nonce,
	ethash_h256_t const* mix_hash,
	ethash_h256_t const* boundary
)
{
	ethash_h256_t return_hash;
	ethash_quick_hash(&return_hash, header_hash, nonce, mix_hash);
	return ethash_check_difficulty(&return_hash, boundary);
}

```

这段代码定义了一个名为 `ethash_light_new_internal` 的函数，它接受两个参数：`cache_size` 和 `seed`。函数内部执行了一系列的操作来创建一个 `struct ethash_light` 类型的变量 `ret`。

首先，函数尝试从内存中分配空间，如果内存不足，函数将返回 `NULL`。然后，函数使用 `calloc` 函数创建了一个大小为 `sizeof(*ret)` 的内存空间，并使用 `malloc` 函数为该内存空间分配了空间。

接下来，函数使用 `ethash_compute_cache_nodes` 函数尝试从给定的 `seed` 开始，计算出 `cache_size` 大小的链表节点。如果函数失败，将执行一些错误处理，然后返回 `NULL`。

最后，函数将创建的 `struct ethash_light` 类型的变量 `ret` 的 `cache_size` 字段设置为分配的内存空间大小，并返回该变量。


```cpp
ethash_light_t ethash_light_new_internal(uint64_t cache_size, ethash_h256_t const* seed)
{
	struct ethash_light *ret;
	ret = (struct ethash_light*)calloc(sizeof(*ret), 1);
	if (!ret) {
		return NULL;
	}
	ret->cache = malloc((size_t)cache_size);
	if (!ret->cache) {
		goto fail_free_light;
	}
	node* nodes = (node*)ret->cache;
	if (!ethash_compute_cache_nodes(nodes, cache_size, seed)) {
		goto fail_free_cache_mem;
	}
	ret->cache_size = cache_size;
	return ret;

```

这两段代码定义了一个名为 `ethash_light_t` 的结构体，用于表示在以太坊网络中执行的身份验证操作的结果。

`ethash_light_new` 函数接收一个 `uint64_t` 类型的 `block_number` 参数。函数首先调用 `ethash_get_seedhash` 函数来获取与传入 `block_number` 相关的种子哈希。然后，调用 `ethash_get_cachesize` 函数获取与种子哈希相关的缓存大小。接着，调用 `ethash_light_new_internal` 函数，传递给 `种子哈希` 和 `缓存大小` 两个参数，并将 `种子哈希` 和 `缓存大小` 存储到结构体变量 `ret` 中。最后，将 `ret` 指向的结构体变量被赋值为调用 `ethash_light_new` 的返回值，并且函数返回该返回值。

`fail_free_cache_mem` 函数在 `ethash_light_new` 函数中调用，负责释放与 `ethash_light_new` 相关的缓存内容。首先，释放 `ret->cache`，然后释放 `ret` 本身，最后返回 `NULL`。

`fail_free_light` 函数在 `ethash_light_new` 函数中调用，负责在 `ethash_light_new` 函数失败时释放与 `ethash_light_new` 相关的资源。首先，释放 `ret`，然后返回 `NULL`，使得调用者可以继续调用其他辅助函数。


```cpp
fail_free_cache_mem:
	free(ret->cache);
fail_free_light:
	free(ret);
	return NULL;
}

ethash_light_t ethash_light_new(uint64_t block_number)
{
	ethash_h256_t seedhash = ethash_get_seedhash(block_number / ETHASH_EPOCH_LENGTH);
	ethash_light_t ret;
	ret = ethash_light_new_internal(ethash_get_cachesize(block_number), &seedhash);
	ret->block_number = block_number;
	return ret;
}

```



这段代码定义了两个函数，分别是`ethash_light_delete()`和`ethash_light_compute_internal()`。

`ethash_light_delete()`函数的作用是删除传入的`ethash_light_t`类型的数据，并将其对应的缓存释放。

`ethash_light_compute_internal()`函数的作用是计算输入的`ethash_light_t`类型的数据的哈希值，其中输入参数包括`light`、`full_size`、`header_hash`和`nonce`。函数首先检查`light`是否有一个缓存，如果是，则使用`free()`函数释放缓存，否则继续计算哈希值。

具体来说，`ethash_light_compute_internal()`函数的实现可以分为以下几个步骤：

1. 使用`ethash_hash()`函数计算输入的`header_hash`和`nonce`的哈希值，并将结果存储在`ret`变量中。

2. 如果哈希值计算成功，则将`success`值设置为`true`，否则将`success`值设置为`false`。

3. 调用`ret`变量的值，将其存储到`success`变量中，并返回。

4. 在`ethash_light_delete()`函数中，如果`light`参数有一个缓存，则使用`free()`函数释放缓存，否则执行哈希计算。


```cpp
void ethash_light_delete(ethash_light_t light)
{
	if (light->cache) {
		free(light->cache);
	}
	free(light);
}

ethash_return_value_t ethash_light_compute_internal(
	ethash_light_t light,
	uint64_t full_size,
	ethash_h256_t const header_hash,
	uint64_t nonce
)
{
	ethash_return_value_t ret;
	ret.success = true;
	if (!ethash_hash(&ret, NULL, light, full_size, header_hash, nonce)) {
		ret.success = false;
	}
	return ret;
}

```

这两函数函数是 `ethereum_bls128_producer_v11` 中实现的内容。

`ethash_light_compute()` 函数的目的是计算一个轻量级哈希的输出值。输入参数包括一个 `ethereum_bls128_view_t` 类型的 `light` 表示要计算的区块的哈希，一个 `ethereum_bls128_h256_view_t` 类型的 `header_hash` 表示哈希头信息，一个 `uint64_t` 类型的 `nonce` 表示哈希输入中的非ce。函数实现中计算并返回一个 `ethash_light_compute_internal()` 返回值的哈希值。

`ethash_full_compute()` 函数的目的是计算一个完整哈希的输出值。输入参数包括一个 `ethereum_bls128_view_t` 类型的 `full` 表示要计算的完整哈希的哈希，一个 `ethereum_bls128_h256_view_t` 类型的 `header_hash` 表示哈希头信息，一个 `uint64_t` 类型的 `nonce` 表示哈希输入中的非ce。函数实现中计算并返回一个 `ethash_full_compute_internal()` 返回值的 `ethash_return_value_t` 类型的变量，其中包含一个 `true` 表示计算成功，以及一个从 `ethereum_bls128_producer_v11_external_dtor_create()` 函数返回的 `void` 类型的参数。


```cpp
ethash_return_value_t ethash_light_compute(
	ethash_light_t light,
	ethash_h256_t const header_hash,
	uint64_t nonce
)
{
	uint64_t full_size = ethash_get_datasize(light->block_number);
	return ethash_light_compute_internal(light, full_size, header_hash, nonce);
}

ethash_return_value_t ethash_full_compute(
	ethash_full_t full,
	ethash_h256_t const header_hash,
	uint64_t nonce
)
{
	ethash_return_value_t ret;
	ret.success = true;
	if (!ethash_hash(
		&ret,
		(node const*)full->data,
		NULL,
		full->file_size,
		header_hash,
		nonce)) {
		ret.success = false;
	}
	return ret;
}

```

这两个函数是用来计算基于以太坊DAG(有向无环图)的散列函数的大小。`ethash_full_dag`函数返回输入的`ethash_full_t`结构中包含的`data`成员，而`ethash_full_dag_size`函数返回`full`参数的`file_size`成员的值。

以太坊DAG是一个包含了一系列数据和函数的指针数组。每个函数都执行散列操作以计算输入的哈希值。 `ethash_full_dag`函数的作用是返回输入的哈希值，而 `ethash_full_dag_size`函数的作用是返回整个DAG文件的大小。

这两个函数是通过对DAG文件进行扫描来计算的。扫描包括从DAG文件的起始地址到结束地址的连续字符流。在扫描过程中，每个字符流都会执行一次`ethash_full_dag`函数来计算哈希值，并将结果存储到`data`成员中。因此，每个哈希值都会被计算一次，最终导致`file_size`成员的值不断累积。


```cpp
void const* ethash_full_dag(ethash_full_t full)
{
	return full->data;
}

uint64_t ethash_full_dag_size(ethash_full_t full)
{
	return full->file_size;
}

```

# `src/3rdparty/libethash/fnv.h`

这段代码是一个C++ Ethereum库文件，它定义了一些C++ Ethereum库函数和变量。这个库文件是由ethereum库的开发者之一编写的。它告诉用户这个库文件是开源的，允许用户自由地重新分发或修改它，前提是在开源协议下进行。这个库文件也明确告知用户这个库文件不保证其可用性，也不保证它的伏尔坦保障。

作者在文件中强调了库文件是“部分归cpp-ethereum项目所有”，这个项目可能由其他人维护，并且作者欢迎贡献。如果你想贡献自己的代码或修复库中的错误，可以访问项目的GitHub仓库：<https://github.com/ethereum/cpp-ethereum>


```cpp
/*
  This file is part of cpp-ethereum.

  cpp-ethereum is free software: you can redistribute it and/or modify
  it under the terms of the GNU General Public License as published by
  the Free Software Foundation, either version 3 of the License, or
  (at your option) any later version.

  cpp-ethereum is distributed in the hope that it will be useful,
  but WITHOUT ANY WARRANTY; without even the implied warranty of
  MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
  GNU General Public License for more details.

  You should have received a copy of the GNU General Public License
  along with cpp-ethereum.  If not, see <http://www.gnu.org/licenses/>.
```

这段代码是一个 C 语言的预处理指令，名为 "#pragma once"。它告诉编译器在编译之前将这段代码保存到一个名为 "fnv.h" 的文件中。

具体来说，这段代码定义了一个名为 "FNV_PRIME" 的常量，其值为 0x01000193。这个值是一个固定的汇编代码，用于实现 CPU 向量回避（CPU 缓存）机制。通过将这个值作为参数传递给汇编函数，可以实现缓存回避，从而提高程序的运行效率。


```cpp
*/
/** @file fnv.h
* @author Matthew Wampler-Doty <negacthulhu@gmail.com>
* @date 2015
*/

#pragma once
#include <stdint.h>

#ifdef __cplusplus
extern "C" {
#endif

#define FNV_PRIME 0x01000193

```

这段代码定义了一个名为fnv_hash的函数，它的作用是计算两个32位无符号整数x和y的哈希值。

该函数采用了一种称为FNV-1的哈希算法，其标准库函数的实现方式与题目描述不同，具体来说，它将输入的两个字节字节数级相乘，然后再将结果乘以FNV_PRIME，其中FNV_PRIME是一个32位无符号整数，这个值在哈希算法中被用来执行异或操作，最后得到一个32位无符号整数的结果，即哈希值。

该函数的实现中，我们也可以看到在计算哈希值的过程中，输入的两个参数x和y都是无符号整数，并且对输入值执行异或操作，这个异或操作的具体实现方式与题目描述一致，只不过在函数内部给出了变量名称，如FNV_PRIME，以便更好地理解。


```cpp
/* The FNV-1 spec multiplies the prime with the input one byte (octet) in turn.
   We instead multiply it with the full 32-bit input.
   This gives a different result compared to a canonical FNV-1 implementation.
*/
static inline uint32_t fnv_hash(uint32_t const x, uint32_t const y)
{
	return x * FNV_PRIME ^ y;
}

#ifdef __cplusplus
}
#endif

```

# `src/3rdparty/libethash/keccakf800.c`

该代码是一个名为Ethash的C/C++实现，可以用来实现以太坊的Proof of Work算法。Proof of Work算法是用于验证加密货币网络中的交易并添加新的区块的算法。

该代码的作用是定义了一个名为rol的函数，用于计算一个32位整数x对一个32位整数s的轮带结果。该函数会将x向右移动s位并将结果向上取整，然后将x和s进行异或操作，得到一个新的32位整数。

该代码还定义了一个包含22个轮带结果的数组，称为round_constants。这些轮带结果用于计算网络中的Proof of Work。

最后，该代码没有输出任何内容，但定义了一个roll函数，可以被用于计算任何给定的32位整数x和轮带结果32。


```cpp
/* ethash: C/C++ implementation of Ethash, the Ethereum Proof of Work algorithm.
 * Copyright 2018-2019 Pawel Bylica.
 * Licensed under the Apache License, Version 2.0.
 */

#include <stdint.h>

static uint32_t rol(uint32_t x, unsigned s)
{
    return (x << s) | (x >> (32 - s));
}

static const uint32_t round_constants[22] = {
    0x00000001,
    0x00008082,
    0x0000808A,
    0x80008000,
    0x0000808B,
    0x80000001,
    0x80008081,
    0x00008009,
    0x0000008A,
    0x00000088,
    0x80008009,
    0x8000000A,
    0x8000808B,
    0x0000008B,
    0x00008089,
    0x00008003,
    0x00008002,
    0x00000080,
    0x0000800A,
    0x8000000A,
    0x80008081,
    0x00008080,
};

```

This appears to be a implementation of a simple program that performs a cyclic降幂排列 on a given list of integers. The program takes two inputs: a list and a distance. The list is passed through the program multiple times, and each time a different number is added to the beginning of the list. The distance is passed through the program once, and is used to determine the number of times the list should be transformed. The program then performs a cyclic降幂排列 on the list, starting with the list and the distance, and ends with the list that will be used for the next iteration of the program.



```cpp
void ethash_keccakf800(uint32_t state[25])
{
    /* The implementation directly translated from ethash_keccakf1600. */

    int round;

    uint32_t Aba, Abe, Abi, Abo, Abu;
    uint32_t Aga, Age, Agi, Ago, Agu;
    uint32_t Aka, Ake, Aki, Ako, Aku;
    uint32_t Ama, Ame, Ami, Amo, Amu;
    uint32_t Asa, Ase, Asi, Aso, Asu;

    uint32_t Eba, Ebe, Ebi, Ebo, Ebu;
    uint32_t Ega, Ege, Egi, Ego, Egu;
    uint32_t Eka, Eke, Eki, Eko, Eku;
    uint32_t Ema, Eme, Emi, Emo, Emu;
    uint32_t Esa, Ese, Esi, Eso, Esu;

    uint32_t Ba, Be, Bi, Bo, Bu;

    uint32_t Da, De, Di, Do, Du;

    Aba = state[0];
    Abe = state[1];
    Abi = state[2];
    Abo = state[3];
    Abu = state[4];
    Aga = state[5];
    Age = state[6];
    Agi = state[7];
    Ago = state[8];
    Agu = state[9];
    Aka = state[10];
    Ake = state[11];
    Aki = state[12];
    Ako = state[13];
    Aku = state[14];
    Ama = state[15];
    Ame = state[16];
    Ami = state[17];
    Amo = state[18];
    Amu = state[19];
    Asa = state[20];
    Ase = state[21];
    Asi = state[22];
    Aso = state[23];
    Asu = state[24];

    for (round = 0; round < 22; round += 2)
    {
        /* Round (round + 0): Axx -> Exx */

        Ba = Aba ^ Aga ^ Aka ^ Ama ^ Asa;
        Be = Abe ^ Age ^ Ake ^ Ame ^ Ase;
        Bi = Abi ^ Agi ^ Aki ^ Ami ^ Asi;
        Bo = Abo ^ Ago ^ Ako ^ Amo ^ Aso;
        Bu = Abu ^ Agu ^ Aku ^ Amu ^ Asu;

        Da = Bu ^ rol(Be, 1);
        De = Ba ^ rol(Bi, 1);
        Di = Be ^ rol(Bo, 1);
        Do = Bi ^ rol(Bu, 1);
        Du = Bo ^ rol(Ba, 1);

        Ba = Aba ^ Da;
        Be = rol(Age ^ De, 12);
        Bi = rol(Aki ^ Di, 11);
        Bo = rol(Amo ^ Do, 21);
        Bu = rol(Asu ^ Du, 14);
        Eba = Ba ^ (~Be & Bi) ^ round_constants[round];
        Ebe = Be ^ (~Bi & Bo);
        Ebi = Bi ^ (~Bo & Bu);
        Ebo = Bo ^ (~Bu & Ba);
        Ebu = Bu ^ (~Ba & Be);

        Ba = rol(Abo ^ Do, 28);
        Be = rol(Agu ^ Du, 20);
        Bi = rol(Aka ^ Da, 3);
        Bo = rol(Ame ^ De, 13);
        Bu = rol(Asi ^ Di, 29);
        Ega = Ba ^ (~Be & Bi);
        Ege = Be ^ (~Bi & Bo);
        Egi = Bi ^ (~Bo & Bu);
        Ego = Bo ^ (~Bu & Ba);
        Egu = Bu ^ (~Ba & Be);

        Ba = rol(Abe ^ De, 1);
        Be = rol(Agi ^ Di, 6);
        Bi = rol(Ako ^ Do, 25);
        Bo = rol(Amu ^ Du, 8);
        Bu = rol(Asa ^ Da, 18);
        Eka = Ba ^ (~Be & Bi);
        Eke = Be ^ (~Bi & Bo);
        Eki = Bi ^ (~Bo & Bu);
        Eko = Bo ^ (~Bu & Ba);
        Eku = Bu ^ (~Ba & Be);

        Ba = rol(Abu ^ Du, 27);
        Be = rol(Aga ^ Da, 4);
        Bi = rol(Ake ^ De, 10);
        Bo = rol(Ami ^ Di, 15);
        Bu = rol(Aso ^ Do, 24);
        Ema = Ba ^ (~Be & Bi);
        Eme = Be ^ (~Bi & Bo);
        Emi = Bi ^ (~Bo & Bu);
        Emo = Bo ^ (~Bu & Ba);
        Emu = Bu ^ (~Ba & Be);

        Ba = rol(Abi ^ Di, 30);
        Be = rol(Ago ^ Do, 23);
        Bi = rol(Aku ^ Du, 7);
        Bo = rol(Ama ^ Da, 9);
        Bu = rol(Ase ^ De, 2);
        Esa = Ba ^ (~Be & Bi);
        Ese = Be ^ (~Bi & Bo);
        Esi = Bi ^ (~Bo & Bu);
        Eso = Bo ^ (~Bu & Ba);
        Esu = Bu ^ (~Ba & Be);


        /* Round (round + 1): Exx -> Axx */

        Ba = Eba ^ Ega ^ Eka ^ Ema ^ Esa;
        Be = Ebe ^ Ege ^ Eke ^ Eme ^ Ese;
        Bi = Ebi ^ Egi ^ Eki ^ Emi ^ Esi;
        Bo = Ebo ^ Ego ^ Eko ^ Emo ^ Eso;
        Bu = Ebu ^ Egu ^ Eku ^ Emu ^ Esu;

        Da = Bu ^ rol(Be, 1);
        De = Ba ^ rol(Bi, 1);
        Di = Be ^ rol(Bo, 1);
        Do = Bi ^ rol(Bu, 1);
        Du = Bo ^ rol(Ba, 1);

        Ba = Eba ^ Da;
        Be = rol(Ege ^ De, 12);
        Bi = rol(Eki ^ Di, 11);
        Bo = rol(Emo ^ Do, 21);
        Bu = rol(Esu ^ Du, 14);
        Aba = Ba ^ (~Be & Bi) ^ round_constants[round + 1];
        Abe = Be ^ (~Bi & Bo);
        Abi = Bi ^ (~Bo & Bu);
        Abo = Bo ^ (~Bu & Ba);
        Abu = Bu ^ (~Ba & Be);

        Ba = rol(Ebo ^ Do, 28);
        Be = rol(Egu ^ Du, 20);
        Bi = rol(Eka ^ Da, 3);
        Bo = rol(Eme ^ De, 13);
        Bu = rol(Esi ^ Di, 29);
        Aga = Ba ^ (~Be & Bi);
        Age = Be ^ (~Bi & Bo);
        Agi = Bi ^ (~Bo & Bu);
        Ago = Bo ^ (~Bu & Ba);
        Agu = Bu ^ (~Ba & Be);

        Ba = rol(Ebe ^ De, 1);
        Be = rol(Egi ^ Di, 6);
        Bi = rol(Eko ^ Do, 25);
        Bo = rol(Emu ^ Du, 8);
        Bu = rol(Esa ^ Da, 18);
        Aka = Ba ^ (~Be & Bi);
        Ake = Be ^ (~Bi & Bo);
        Aki = Bi ^ (~Bo & Bu);
        Ako = Bo ^ (~Bu & Ba);
        Aku = Bu ^ (~Ba & Be);

        Ba = rol(Ebu ^ Du, 27);
        Be = rol(Ega ^ Da, 4);
        Bi = rol(Eke ^ De, 10);
        Bo = rol(Emi ^ Di, 15);
        Bu = rol(Eso ^ Do, 24);
        Ama = Ba ^ (~Be & Bi);
        Ame = Be ^ (~Bi & Bo);
        Ami = Bi ^ (~Bo & Bu);
        Amo = Bo ^ (~Bu & Ba);
        Amu = Bu ^ (~Ba & Be);

        Ba = rol(Ebi ^ Di, 30);
        Be = rol(Ego ^ Do, 23);
        Bi = rol(Eku ^ Du, 7);
        Bo = rol(Ema ^ Da, 9);
        Bu = rol(Ese ^ De, 2);
        Asa = Ba ^ (~Be & Bi);
        Ase = Be ^ (~Bi & Bo);
        Asi = Bi ^ (~Bo & Bu);
        Aso = Bo ^ (~Bu & Ba);
        Asu = Bu ^ (~Ba & Be);
    }

    state[0] = Aba;
    state[1] = Abe;
    state[2] = Abi;
    state[3] = Abo;
    state[4] = Abu;
    state[5] = Aga;
    state[6] = Age;
    state[7] = Agi;
    state[8] = Ago;
    state[9] = Agu;
    state[10] = Aka;
    state[11] = Ake;
    state[12] = Aki;
    state[13] = Ako;
    state[14] = Aku;
    state[15] = Ama;
    state[16] = Ame;
    state[17] = Ami;
    state[18] = Amo;
    state[19] = Amu;
    state[20] = Asa;
    state[21] = Ase;
    state[22] = Asi;
    state[23] = Aso;
    state[24] = Asu;
}

```