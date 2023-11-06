# GGML源码解析 21

# `src/ggml-alloc.c`



这段代码是一个C语言程序，它包括了多个头文件和一些常量。

其中包括：

- ggml-alloc.h: 这是ggml内存分配器实现的头文件，定义了一些函数来分配内存和释放内存。
- ggml-backend-impl.h: 这是ggml后端实现的头文件，定义了一些函数来与操作系统交互，包括文件读写和进程上下文切换等。
- ggml.h: 这是ggml的主函数头文件，定义了一些全局变量和函数。
- ggml-impl.h: 这是ggml的接口头文件，定义了一些函数来实现ggml的各种操作。
- <assert.h>: 这是C语言标准库中的一个头文件，定义了一些常见的数学运算和判断条件。
- <limits.h>: 这是C语言标准库中的一个头文件，定义了一些常见的数学运算和限制条件。
- <stdarg.h>: 这是C语言标准库中的一个头文件，定义了一些函数来接受字符串参数。
- <stdio.h>: 这是C语言标准库中的一个头文件，定义了一些函数来读写标准输入和输出。
- <stdlib.h>: 这是C语言标准库中的一个头文件，定义了一些函数来使用操作系统提供的函数。
- <string.h>: 这是C语言标准库中的一个头文件，定义了一些函数来操作字符串。

另外，它还定义了一些常量，包括MAX_FREE_BLOCKS和MAX_FREE_BLOCKS，分别表示最大可用的free blocks和最大free blocks。MAX_FREE_BLOCKS是ggml内存分配器的一个参数，决定了它能够在进程内分配多少内存，而MAX_FREE_BLOCKS则是ggml后端实现的一个参数，表示最大的free blocks。


```cpp
#include "ggml-alloc.h"
#include "ggml-backend-impl.h"
#include "ggml.h"
#include "ggml-impl.h"
#include <assert.h>
#include <limits.h>
#include <stdarg.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

#define MAX(a, b) ((a) > (b) ? (a) : (b))
#define MAX_FREE_BLOCKS 256

//#define GGML_ALLOCATOR_DEBUG

```



这段代码定义了两个符号常量：AT_PRINTF和AT_PRINTF_X。AT_PRINTF定义了一个宏，用于在调试输出中打印给定的格式化字符串，并接受一个或多个参数。AT_PRINTF_X定义了一个宏，也用于在调试输出中打印给定的格式化字符串，但使用编译时-size_t类型参数。

AT_PRINTF宏的作用类似于printf函数，但是使用宏定义的方式。AT_PRINTF宏定义了一个名为fprintf的函数，接受三个参数：stderr、格式字符串和参数列表。其中，格式字符串是一个字符串，用于在调试输出中打印给定的信息，而参数列表则是一个包含多个可变参数的元组。

AT_PRINTF_X宏定义了一个名为fprintf的函数，但是该函数同时也使用了size_t类型参数。这个宏定义的函数与AT_PRINTF宏的作用类似，但是使用宏定义的方式。

接下来，该代码实现了一个名为aligned_offset的函数，该函数接受一个缓冲区、偏移量和铝片大小。函数首先计算偏移量与2的幂的差，然后将偏移量左移得到正确的偏移量，最后将偏移量添加到缓冲区的起始位置。

最后，该代码定义了一个名为free_block的结构体，该结构体包含一个指向内存的指针、一个大小和铝片大小。free_block用于在内存分配和释放时保证块状内存对齐。


```cpp
//#define AT_PRINTF(...) fprintf(stderr, __VA_ARGS__)
#define AT_PRINTF(...)

// TODO: GGML_PAD ?
static size_t aligned_offset(const void * buffer, size_t offset, size_t alignment) {
    assert(alignment && !(alignment & (alignment - 1))); // power of 2
    size_t align = (alignment - (((uintptr_t)buffer + offset) % alignment)) % alignment;
    return offset + align;
}

struct free_block {
    void * addr;
    size_t size;
};

```

这是一个用C++编写的GGML（Go-Backward Model）库的代码，该库提供了 tallocr 结构体。根据描述，可以得出以下结论：

1. 该代码定义了一个名为 ggml_tallocr 的结构体。
2. 这个结构体有多个成员变量：
   - buffer：指向一个后台缓冲区的指针，它是该结构体的一个成员。
   - buffer_owned：一个布尔值，表示缓冲区是否被内存的所有者（可能是 GPU）自动分配。
   - base：一个 void 类型指针，表示数据开始存储的位置。
   - alignment：一个 size_t 类型，表示与后跟的缓冲区大小对齐的次数。
   - n_free_blocks：一个 int 类型，表示自由的块（free blocks）的数量。
   - free_blocks：一个大小为 MAX_FREE_BLOCKS 的 array，定义了 free_blocks 数组。
   - max_size：一个 size_t 类型，表示能够存储的最大数据量。
   - measure：一个 bool 类型，表示是否测量可用内存大小。
3. 该结构体有一个名为 allocated_tensors 的数组，它的大小为 1024，但只初始化了其中的前 1024 个元素，因为后面还需要根据 measure 来分配内存。

结构体的定义并不包含实现，因此不能提供代码的完整副本。


```cpp
struct ggml_tallocr {
    struct ggml_backend_buffer * buffer;
    bool buffer_owned;
    void * base;
    size_t alignment;

    int n_free_blocks;
    struct free_block free_blocks[MAX_FREE_BLOCKS];

    size_t max_size;

    bool measure;

#ifdef GGML_ALLOCATOR_DEBUG
    struct ggml_tensor * allocated_tensors[1024];
```

这段代码定义了两个函数 add_allocated_tensor 和 remove_allocated_tensor，它们属于GGML的Tallocr分配器模块。

add_allocated_tensor函数用于在Tallocr分配器中增加一个已经存在的tensor的内存空间。它接受两个参数：一个是Tallocr分配器，另一个是tensor结构体。函数首先检查分配器中是否已经存在名为tensor的tensor，如果已经存在，则直接返回。否则，它将tensor复制到分配器中名为allocated_tensors的数组中，并返回。

remove_allocated_tensor函数用于在Tallocr分配器中删除一个已经存在的tensor的内存空间。它接受两个参数：一个是Tallocr分配器，另一个是tensor结构体。函数首先检查分配器中是否已经存在名为tensor的tensor，如果是，则尝试将其从allocated_tensors数组中删除。如果尝试删除失败，函数将输出一个错误消息，指出tensor tensor 无法被找到。


```cpp
#endif
};

#ifdef GGML_ALLOCATOR_DEBUG
static void add_allocated_tensor(ggml_tallocr_t alloc, struct ggml_tensor * tensor) {
    for (int i = 0; i < 1024; i++) {
        if (alloc->allocated_tensors[i] == NULL) {
            alloc->allocated_tensors[i] = tensor;
            return;
        }
    }
    GGML_ASSERT(!"out of allocated_tensors");
}
static void remove_allocated_tensor(ggml_tallocr_t alloc, struct ggml_tensor * tensor) {
    for (int i = 0; i < 1024; i++) {
        if (alloc->allocated_tensors[i] == tensor ||
            (alloc->allocated_tensors[i] != NULL && alloc->allocated_tensors[i]->data == tensor->data)) {
            alloc->allocated_tensors[i] = NULL;
            return;
        }
    }
    printf("tried to free tensor %s not found\n", tensor->name);
    GGML_ASSERT(!"tensor not found");
}
```

This is a function that calculates the size and best fit for a tensor tensor that is being added to a buffer, and also updates the buffer if the tensor is larger than the buffer. It takes in the tensor tensor, the buffer, and a number of blocks (a list of free blocks) that are available in the buffer. The function returns the number of blocks that can hold the tensor, or -1 if the buffer is full and the tensor is too large for the buffer.

The function first sets the buffer size to the maximum size, and then loops through the blocks to find the one that is the best fit for the tensor. The function finds the maximum size of the free block by taking the maximum of the available sizes. If the free block is larger than the maximum size and it is the best fit, the function updates the best fit block and the size to be the maximum of the available size and the size of the free block.

Finally, the function adds the tensor to the allocated buffer, and if the buffer is full, it removes the blocks that are not needed to hold the tensor.


```cpp
#endif

// check if a tensor is allocated by this buffer
static bool ggml_tallocr_is_own(ggml_tallocr_t alloc, const struct ggml_tensor * tensor) {
    return tensor->buffer == alloc->buffer;
}

static bool ggml_is_view(struct ggml_tensor * t) {
    return t->view_src != NULL;
}

void ggml_tallocr_alloc(ggml_tallocr_t alloc, struct ggml_tensor * tensor) {
    GGML_ASSERT(!ggml_is_view(tensor)); // views generally get data pointer from one of their sources
    GGML_ASSERT(tensor->data == NULL); // avoid allocating tensor which already has memory allocated

    size_t size = ggml_backend_buffer_get_alloc_size(alloc->buffer, tensor);
    size = aligned_offset(NULL, size, alloc->alignment);

    AT_PRINTF("%s: allocating %s (%zu bytes) - ", __func__, tensor->name, size);

    size_t max_avail = 0;

    // find the best fitting free block besides the last block
    int best_fit_block = -1;
    size_t best_fit_size = SIZE_MAX;
    for (int i = 0; i < alloc->n_free_blocks - 1; i++) {
        struct free_block * block = &alloc->free_blocks[i];
        max_avail = MAX(max_avail, block->size);
        if (block->size >= size && block->size <= best_fit_size) {
            best_fit_block = i;
            best_fit_size = block->size;
        }
    }

    AT_PRINTF("block %d\n", best_fit_block);

    if (best_fit_block == -1) {
        // the last block is our last resort
        struct free_block * block = &alloc->free_blocks[alloc->n_free_blocks - 1];
        max_avail = MAX(max_avail, block->size);
        if (block->size >= size) {
            best_fit_block = alloc->n_free_blocks - 1;
        } else {
            fprintf(stderr, "%s: not enough space in the buffer (needed %zu, largest block available %zu)\n",
                    __func__, size, max_avail);
            GGML_ASSERT(!"not enough space in the buffer");
            return;
        }
    }
    struct free_block * block = &alloc->free_blocks[best_fit_block];
    void * addr = block->addr;
    block->addr = (char*)block->addr + size;
    block->size -= size;
    if (block->size == 0) {
        // remove block if empty
        alloc->n_free_blocks--;
        for (int j = best_fit_block; j < alloc->n_free_blocks; j++) {
            alloc->free_blocks[j] = alloc->free_blocks[j+1];
        }
    }

    tensor->data = addr;
    tensor->buffer = alloc->buffer;
    if (!alloc->measure) {
        ggml_backend_buffer_init_tensor(alloc->buffer, tensor);
    }

```

这段代码是一个C语言中的函数，它有以下几个主要作用：

1. 检查GGML Allocator是否开启了调试模式。如果是，则执行以下操作：

  a. 添加已分配的张量（tensor）到分配器（alloc）中。

  b. 计算当前分配器中可用的最大张量大小。

  c. 如果当前分配器中的最大张量大小小于刚刚计算得到的较大张量，则输出最大张量的单位（MB）。

  如果不是开启了调试模式，则执行以下操作：

  a. 将分配器中最大张量的单位（MB）设置为刚刚计算得到的最大张量。

  b. 更新分配器中最大张量的单位（MB）。

  
这段代码的作用是监视一个名为GGML_ALLOCATOR_DEBUG的预定义标志，根据标志的值决定是否输出调试信息。如果开启了调试模式，那么函数会遍历分配器中已分配的张量，并输出每个张量的单位（MB）。


```cpp
#ifdef GGML_ALLOCATOR_DEBUG
    add_allocated_tensor(alloc, tensor);
    size_t cur_max = (char*)addr - (char*)alloc->data + size;
    if (cur_max > alloc->max_size) {
        printf("max_size = %.2f MB: tensors: ", cur_max / 1024.0 / 1024.0);
        for (int i = 0; i < 1024; i++) {
            if (alloc->allocated_tensors[i]) {
                printf("%s (%.2f MB) ", alloc->allocated_tensors[i]->name, ggml_nbytes(alloc->allocated_tensors[i]) / 1024.0 / 1024.0);
            }
        }
        printf("\n");
    }
#endif

    alloc->max_size = MAX(alloc->max_size, (char*)addr - (char*)alloc->base + size);
}

```

这段代码是一个非常naive的实现，但考虑到我们的案例中free blocks的数量可能非常少，因此需要对free blocks进行优化。

该函数的作用是检查传入的Tensor是否在内存中已分配，如果不是，则需要手动释放内存。如果Tensor已经在内存中分配过，则检查Tensor的data指针是否在分配的内存区域内，如果是，那么直接跳过这一行。如果不是，则需要计算出内存大小并使用aligned_offset函数计算出需要的内存大小，然后使用函数free_tensor释放内存。

函数的参数包括：

- alloc：存储Tensor的内存分配器。
- tensor：Tensor的struct结构体指针。

函数的实现忽略了一个重要的点：函数的实现没有考虑到free blocks，因此需要手动计算出free blocks的数量。这个数量可以根据具体应用场景进行调整。


```cpp
// this is a very naive implementation, but for our case the number of free blocks should be very small
static void ggml_tallocr_free_tensor(ggml_tallocr_t alloc, struct ggml_tensor * tensor) {
    if (ggml_tallocr_is_own(alloc, tensor) == false) {
        // the tensor was not allocated in this buffer
        // this can happen because the graph allocator will try to free weights and other tensors from different buffers
        // the easiest way to deal with this is just to ignore it
        // AT_PRINTF("ignoring %s (their buffer: %p, our buffer: %p)\n", tensor->name, (void *)tensor->buffer, (void *)alloc->buffer);
        return;
    }

    void * ptr = tensor->data;

    size_t size = ggml_backend_buffer_get_alloc_size(alloc->buffer, tensor);
    size = aligned_offset(NULL, size, alloc->alignment);
    AT_PRINTF("%s: freeing %s at %p (%zu bytes) - n_free_blocks = %d\n", __func__, tensor->name, ptr, size, alloc->n_free_blocks);

    if (!alloc->measure) {
        ggml_backend_buffer_free_tensor(alloc->buffer, tensor);
    }

```

This function appears to be a part of a larger software suite designed to manage memory allocation and deallocation. It functions as follows:

1. It checks if the given pointer is within the first block of free memory blocks and updates its value if it is.
2. If the pointer is not within the first block, it checks if the pointer points to the beginning of a block and updates the block's address and size accordingly.
3. If the pointer still does not correspond to a block, it checks if the function already exists in the free blocks array and updates the free blocks array accordingly.
4. If the function already exists in the free blocks array but it does not correspond to the given pointer, it merges the blocks and updates the free blocks array accordingly.
5. If the function does not exist in the free blocks array, it adds a new block to the free blocks array and updates the free blocks array accordingly.




```cpp
#ifdef GGML_ALLOCATOR_DEBUG
    remove_allocated_tensor(alloc, tensor);
#endif

    // see if we can merge with an existing block
    for (int i = 0; i < alloc->n_free_blocks; i++) {
        struct free_block * block = &alloc->free_blocks[i];
        // check if ptr is at the end of the block
        if ((char*)block->addr + block->size == ptr) {
            block->size += size;
            // check if we can merge with the next block
            if (i < alloc->n_free_blocks - 1 && (char*)block->addr + block->size == alloc->free_blocks[i+1].addr) {
                block->size += alloc->free_blocks[i+1].size;
                alloc->n_free_blocks--;
                for (int j = i+1; j < alloc->n_free_blocks; j++) {
                    alloc->free_blocks[j] = alloc->free_blocks[j+1];
                }
            }
            return;
        }
        // check if ptr is at the beginning of the block
        if ((char*)ptr + size == block->addr) {
            block->addr = ptr;
            block->size += size;
            // check if we can merge with the previous block
            if (i > 0 && (char*)alloc->free_blocks[i-1].addr + alloc->free_blocks[i-1].size == block->addr) {
                alloc->free_blocks[i-1].size += block->size;
                alloc->n_free_blocks--;
                for (int j = i; j < alloc->n_free_blocks; j++) {
                    alloc->free_blocks[j] = alloc->free_blocks[j+1];
                }
            }
            return;
        }
    }
    // otherwise, add a new block
    GGML_ASSERT(alloc->n_free_blocks < MAX_FREE_BLOCKS && "out of free blocks");
    // insert the new block in the correct position to keep the array sorted by address (to make merging blocks faster)
    int insert_pos = 0;
    while (insert_pos < alloc->n_free_blocks && alloc->free_blocks[insert_pos].addr < ptr) {
        insert_pos++;
    }
    // shift all blocks from insert_pos onward to make room for the new block
    for (int i = alloc->n_free_blocks; i > insert_pos; i--) {
        alloc->free_blocks[i] = alloc->free_blocks[i-1];
    }
    // insert the new block
    alloc->free_blocks[insert_pos].addr = ptr;
    alloc->free_blocks[insert_pos].size = size;
    alloc->n_free_blocks++;
}

```

这段代码定义了一个名为 `ggml_tallocr_reset` 的函数，用于重置 `ggml_tallocr_t` 类型的 alloc 对象。该函数在函数内部执行以下操作：

1. 将 alloc 对象中的 `n_free_blocks` 字段设置为 1，这意味着 now 这个 alloc 对象里面有且仅有一个 free 块。
2. 根据 `align_offset` 和 `alignment` 计算出 `align_offset` 变量，然后将其用于将 `free_blocks` 初始化为零的起始地址。
3. 如果 `measure` 参数为真，那么设置 free 块的最大大小为 `SIZE_MAX/2`，否则设置为 `ggml_backend_buffer_get_size(buffer) - align_offset`。
4. 调用 `ggml_backend_buffer_get_base(buffer)` 函数获取 free 块的起始地址，再将起始地址赋给 `free_blocks[0].addr`。

该函数的作用是协调地将 free 块分配给主函数，并设置 free 块的大小以及是否可以测量。


```cpp
void ggml_tallocr_reset(ggml_tallocr_t alloc) {
    alloc->n_free_blocks = 1;
    size_t align_offset = aligned_offset(alloc->base, 0, alloc->alignment);
    alloc->free_blocks[0].addr = (char *)alloc->base + align_offset;

    if (alloc->measure) {
        alloc->free_blocks[0].size = SIZE_MAX/2; // restrict maximum size of a measure allocator to half size_t max to avoid overflows
    } else {
        alloc->free_blocks[0].size = ggml_backend_buffer_get_size(alloc->buffer) - align_offset;
    }
}

ggml_tallocr_t ggml_tallocr_new(void * data, size_t size, size_t alignment) {
    struct ggml_backend_buffer * buffer = ggml_backend_cpu_buffer_from_ptr(NULL, data, size);

    ggml_tallocr_t alloc = (ggml_tallocr_t)malloc(sizeof(struct ggml_tallocr));

    *alloc = (struct ggml_tallocr) {
        /*.buffer        = */ buffer,
        /*.buffer_owned  = */ true,
        /*.base          = */ ggml_backend_buffer_get_base(buffer),
        /*.alignment     = */ alignment,
        /*.n_free_blocks = */ 0,
        /*.free_blocks   = */ {{0}},
        /*.max_size      = */ 0,
        /*.measure       = */ false,
```

这段代码定义了一个名为ggml_tallocr_new_measure的函数，其作用是返回一个指向能够设置新测量度的tallocr分配器的指针。

函数的实现包括以下步骤：

1. 定义了一个名为#ifdef GGML_ALLOCATOR_DEBUG的判断框，表示如果DEBUG条件为真，那么下面代码块中的内容将被展开并执行。

2. 在函数体中，定义了一个名为allocated_tensors的tuple，其成员为0。

3. 调用ggml_tallocr_reset函数，该函数将传入一个alloc变量，用于存储新的分配器。

4. 返回调用函数返回的alloc变量，即新的分配器。

5. 在函数体中，定义了一个名为ggml_tallocr_new_measure的函数，用于创建一个新的测量度并设置为true。

6. 在函数体中，创建了一个名为alloc的tallocr分配器，并将其赋值为上面返回的分配器。

7. 返回alloc，即新的分配器。


```cpp
#ifdef GGML_ALLOCATOR_DEBUG
        /*.allocated_tensors = */ {0},
#endif
    };

    ggml_tallocr_reset(alloc);

    return alloc;
}

ggml_tallocr_t ggml_tallocr_new_measure(size_t alignment) {
    ggml_tallocr_t alloc = ggml_tallocr_new((void *)0x1000, SIZE_MAX/2, alignment);
    alloc->measure = true;

    return alloc;
}

```



这段代码定义了ggml_tallocr_t类型的变量，用于实现TallOcr算法。

ggml_tallocr_t:
- 构造函数ggml_tallocr_t ggml_tallocr_new_measure_from_backend(struct ggml_backend * backend)的实现，接收一个ggml_backend类型的参数backend，没有参数。
- 函数ggml_tallocr_new_from_backend(struct ggml_backend * backend, size_t size)的实现，接收一个ggml_backend类型的参数backend和一个size_t类型的参数size。

ggml_backend_buffer_t:
- 构造函数ggml_backend_alloc_buffer(struct ggml_backend * backend, size_t size)的实现，接收一个ggml_backend类型的参数backend和一个size_t类型的参数size，返回一个ggml_backend_buffer_t类型的缓冲区。

ggml_tallocr_t:
- 成员函数ggml_tallocr_new_from_buffer(struct ggml_backend * backend)的实现，接收一个ggml_backend类型的参数backend，返回一个ggml_tallocr_t类型的对象。
- 成员函数ggml_tallocr_reset(struct ggml_tallocr_t)的实现，接收一个ggml_tallocr_t类型的对象，重置其内部状态并返回。

这段代码的作用是实现了一个TallOcr算法，可以对从后端服务器获取的图像数据进行处理，将TallOcr算法的内存空间与后端服务器数据写入的内存空间对齐，并返回处理后的结果。


```cpp
ggml_tallocr_t ggml_tallocr_new_measure_from_backend(struct ggml_backend * backend) {
    // create a backend buffer to get the correct tensor allocation sizes
    ggml_backend_buffer_t buffer = ggml_backend_alloc_buffer(backend, 1);

    // TODO: move alloc initialization to a common ggml_tallocr_new_impl function
    ggml_tallocr_t alloc = ggml_tallocr_new_from_buffer(buffer);
    alloc->buffer_owned = true;
    alloc->measure = true;
    ggml_tallocr_reset(alloc);
    return alloc;
}

ggml_tallocr_t ggml_tallocr_new_from_backend(struct ggml_backend * backend, size_t size) {
    ggml_backend_buffer_t buffer = ggml_backend_alloc_buffer(backend, size);
    ggml_tallocr_t alloc = ggml_tallocr_new_from_buffer(buffer);
    alloc->buffer_owned = true;
    return alloc;
}

```

这段代码定义了一个名为 ggml_tallocr_t 的结构体，用于表示一个在缓冲区分配出来的 tallocr 分配器。该结构体包含一个指向缓冲区的指针 buffer，以及一些与 tallocr 分配器相关的成员变量。

其中，malloc() 函数用于在内存中分配足够大小的空间，并将其赋值为 struct ggml_tallocr 类型的 alloc。在 alloc 变量被赋值之后，我们可以对它进行一些修改，例如修改其成员变量的值。

ggml_tallocr_new_from_buffer() 函数接收一个指向缓冲区的指针 buffer，返回一个指向 alloc 的指针，即在 buffer 中分配出来的 tallocr 分配器。

该函数首先在内存中分配足够大小的空间，并将其赋值为 struct ggml_tallocr 类型的 alloc。然后，我们调用 ggml_tallocr_reset() 函数来初始化 alloc 变量，并将其设为 NULL。

最后，函数返回 alloc，即在 buffer 中分配出来的 tallocr 分配器。


```cpp
ggml_tallocr_t ggml_tallocr_new_from_buffer(struct ggml_backend_buffer * buffer) {
    ggml_tallocr_t alloc = (ggml_tallocr_t)malloc(sizeof(struct ggml_tallocr));

    *alloc = (struct ggml_tallocr) {
        /*.buffer        = */ buffer,
        /*.buffer_owned  = */ false,
        /*.base          = */ ggml_backend_buffer_get_base(buffer),
        /*.alignment     = */ ggml_backend_buffer_get_alignment(buffer),
        /*.n_free_blocks = */ 0,
        /*.free_blocks   = */ {{0}},
        /*.max_size      = */ 0,
        /*.measure       = */ false,
#ifdef GGML_ALLOCATOR_DEBUG
        /*.allocated_tensors = */ {0},
#endif
    };

    ggml_tallocr_reset(alloc);

    return alloc;
}

```

这段代码定义了两个结构体ggml_backend_buffer * ggml_tallocr_get_buffer和void ggml_tallocr_free，它们属于一个名为ggml_tallocr的类。

ggml_tallocr_get_buffer的函数实现了一个从名为ggml_tallocr的类的构造函数中获取缓冲区的操作。这个缓冲区是存储在ggml_tallocr的alloc成员变量中的，通过调用这个函数，可以在需要时获取到它所存储的缓冲区。

ggml_tallocr_free的函数实现了释放分配给ggml_tallocr的内存的逻辑。这个函数首先检查分配的内存是否为空，如果是，则直接返回。如果内存不为空，那么它将遍历该内存中的所有缓冲区，并使用gml_backend_buffer_free函数释放这些缓冲区。最后，使用free函数释放ggml_tallocr所分配的内存。


```cpp
struct ggml_backend_buffer * ggml_tallocr_get_buffer(ggml_tallocr_t alloc) {
    return alloc->buffer;
}

void ggml_tallocr_free(ggml_tallocr_t alloc) {
    if (alloc == NULL) {
        return;
    }

    if (alloc->buffer_owned) {
        ggml_backend_buffer_free(alloc->buffer);
    }
    free(alloc);
}

```

这是一个C++的代码，定义了两个函数，和一个结构体。现在我来解释一下这两个函数和一个结构体。

ggml_tallocr_is_measure函数的作用是判断一个ggml_tallocr_t类型的对象是否是一个测量值。函数返回一个布尔值，表示true或false。

ggml_tallocr_max_size函数的作用是获取一个ggml_tallocr_t类型的对象的最大的可见大小。函数返回一个size_t类型的对象。

struct hash_node {
   int n_children;
   int n_views;
};

这个结构体定义了一个哈希表节点。哈希表节点包含两个整数成员，n_children和n_views。n_children表示这个节点拥有的子节点数，n_views表示这个节点拥有的视图数。


```cpp
bool ggml_tallocr_is_measure(ggml_tallocr_t alloc) {
    return alloc->measure;
}

size_t ggml_tallocr_max_size(ggml_tallocr_t alloc) {
    return alloc->max_size;
}

// graph allocator

struct hash_node {
    int n_children;
    int n_views;
};

```

这是一个定义了 `ggml_gallocr` 结构体的函数，其作用是返回一个指向 `ggml_gallocr` 结构体的指针。该结构体定义了以下字段：

- `talloc`：指向一个 `ggml_tallocr_t` 的指针，用于管理内存分配和释放。
- `hash_set`：指向一个 `ggml_hash_set` 的指针，用于存储元数据页的键值。
- `hash_values`：指向一个链表的指针，用于存储数据页的键值。该链表将按照元数据页的键值存储。
- `hash_values_size`：存储 `hash_values` 指向链表的结点数量。
- `hash_allocs`：指向一个链表的指针，用于存储分配的内存块的指针。
- `parse_seq`：指向一个整数的指针，用于存储解析序列的起始和终止点。
- `parse_seq_len`：存储 `parse_seq` 指向的整数的长度。

该函数的实现类似于 `python.c` 文件中的 `ggml_gallocr_new` 函数，该函数接受两个参数，第一个参数是 `None` 或者是元数据页的键值，第二个参数是要分配的内存块的起始地址。函数返回一个指向 `ggml_gallocr` 结构体的指针，该结构体将按照指定的规则建立一个新的 `ggml_gallocr` 结构体。


```cpp
struct ggml_gallocr {
    ggml_tallocr_t talloc;
    struct ggml_hash_set hash_set;
    struct hash_node * hash_values;
    size_t hash_values_size;
    ggml_tallocr_t * hash_allocs;
    int * parse_seq;
    int parse_seq_len;
};

ggml_gallocr_t ggml_gallocr_new(void) {
    ggml_gallocr_t galloc = (ggml_gallocr_t)malloc(sizeof(struct ggml_gallocr));

    *galloc = (struct ggml_gallocr) {
        /*.talloc           = */ NULL,
        /*.hash_set         = */ {0},
        /*.hash_values      = */ NULL,
        /*.hash_values_size = */ 0,
        /*.hash_allocs      = */ NULL,
        /*.parse_seq        = */ NULL,
        /*.parse_seq_len    = */ 0,
    };

    return galloc;
}

```



这段代码是一个名为 `ggml_gallocr_free` 的函数，属于 `ggml_gallocr` 系列的函数。

它的作用是释放已经分配给 `ggalloc` 对象的内存。具体来说，它执行以下操作：

1. 如果 `galloc` 是一个 `ggaml_gallocr` 结构体或指针，那么直接返回，因为已经没有需要释放的内存。

2. 如果 `galloc` 指针中包含一个指向 `ggml_gallocr_hash_set` 的指针，那么释放该指针所指向的内存，因为 `ggml_gallocr_hash_set` 已经被释放。

3. 如果 `galloc` 指针中包含一个指向 `ggml_gallocr_hash_values` 的指针，那么释放该指针所指向的内存，因为 `ggml_gallocr_hash_values` 已经被释放。

4. 如果 `galloc` 指针中包含一个指向 `ggml_gallocr_hash_allocs` 的指针，那么释放该指针所指向的内存，因为 `ggml_gallocr_hash_allocs` 已经被释放。

5. 如果 `galloc` 指针中包含一个指向 `ggml_gallocr_parse_seq` 的指针，那么释放该指针所指向的内存，因为 `ggml_gallocr_parse_seq` 已经被释放。

6. 最后释放 `galloc` 对象本身，以便垃圾回收器可以回收使用过的内存。


```cpp
void ggml_gallocr_free(ggml_gallocr_t galloc) {
    if (galloc == NULL) {
        return;
    }

    if (galloc->hash_set.keys != NULL) {
        free(galloc->hash_set.keys);
    }
    if (galloc->hash_values != NULL) {
        free(galloc->hash_values);
    }
    if (galloc->hash_allocs != NULL) {
        free(galloc->hash_allocs);
    }
    if (galloc->parse_seq != NULL) {
        free(galloc->parse_seq);
    }
    free(galloc);
}

```

这段代码定义了两个函数，用于在ggml_gallocr_t类型的 galloc 对象中设置解析序列和生成哈希表。

1. `ggml_gallocr_set_parse_seq`函数接收一个ggml_gallocr_t类型的 galloc 对象，一个包含数字的数组 list 和数字的数量 n。函数的主要作用是分配内存并设置 galloc 对象中 parse_seq 成员的值。具体来说，函数首先释放了已分配的内存，然后为 galloc 对象分配了相同大小的内存，并将 list 中的每个元素存储到新分配的内存中。最后，函数还计算了 parse_seq 成员的位数。

2. `hash_get`函数接收一个ggml_gallocr_t类型的 galloc 对象和一个ggml_tensor类型的数据结构 t。函数的主要作用是在 galloc 对象中查找并返回具有键 t 的哈希表的第一个元素，如果哈希表中包含该键，则返回该元素的指针，否则返回 NULL。哈希表的元素类型被指定为 struct ggml_tensor_hash_node *。

这段代码的主要目的是在ggml_gallocr_t类型的 galloc 对象中管理数据结构哈希表，以便对数据结构进行查找、插入和删除操作。


```cpp
void ggml_gallocr_set_parse_seq(ggml_gallocr_t galloc, const int * list, int n) {
    free(galloc->parse_seq);
    galloc->parse_seq = malloc(sizeof(int) * n);

    for (int i = 0; i < n; i++) {
        galloc->parse_seq[i] = list[i];
    }
    galloc->parse_seq_len = n;
}

static struct hash_node * hash_get(ggml_gallocr_t galloc, struct ggml_tensor * t) {
    size_t i = ggml_hash_find_or_insert(galloc->hash_set, t);
    return &galloc->hash_values[i];
}

```

这段代码是一个判断两个ggml张量是否相同的函数，主要作用是检查两个张量的布局是否相同。通过检查张量类型、元素类型以及张量维度是否相同，来判断两个张量是否相同。如果张量类型、元素类型或者张量维度中的任何一个不匹配，函数就会返回false，表示两个张量不相同。如果所有元素类型和维度都匹配，函数就会返回true，表示两个张量相同。


```cpp
static bool ggml_are_same_layout(const struct ggml_tensor * a, const struct ggml_tensor * b) {
    if (a->type != b->type) {
        return false;
    }
    for (int i = 0; i < GGML_MAX_DIMS; i++) {
        if (a->ne[i] != b->ne[i]) {
            return false;
        }
        if (a->nb[i] != b->nb[i]) {
            return false;
        }
    }
    return true;
}

```

这段代码是一个静态判断语句，用于检查输入的ggml操作符是否可以在原地进行计算。这个判断在代码中进行了switch类型的分支判断，对于每个ggml操作符，如果它属于GGML_OP_SCALE、GGML_OP_DIAG_MASK_ZERO、GGML_OP_DIAG_MASK_INF、GGML_OP_ADD、GGML_OP_ADD1、GGML_OP_SUB、GGML_OP_MUL、GGML_OP_DIV、GGML_OP_SQR、GGML_OP_SQRT、GGML_OP_LOG、GGML_OP_UNARY、GGML_OP_ROPE、GGML_OP_RMS_NORM、GGML_OP_SOFT_MAX这17种情况，那么函数返回true，否则返回false。这个函数的作用是帮助开发者判断输入的ggml操作符是否可以进行原地计算，以避免在某些特殊情况下产生不必要的性能问题。


```cpp
static bool ggml_op_can_inplace(enum ggml_op op) {
    switch (op) {
        case GGML_OP_SCALE:
        case GGML_OP_DIAG_MASK_ZERO:
        case GGML_OP_DIAG_MASK_INF:
        case GGML_OP_ADD:
        case GGML_OP_ADD1:
        case GGML_OP_SUB:
        case GGML_OP_MUL:
        case GGML_OP_DIV:
        case GGML_OP_SQR:
        case GGML_OP_SQRT:
        case GGML_OP_LOG:
        case GGML_OP_UNARY:
        case GGML_OP_ROPE:
        case GGML_OP_RMS_NORM:
        case GGML_OP_SOFT_MAX:
            return true;

        default:
            return false;
    }
}

```



这段代码定义了两个函数，一个是`node_tallocr`，另一个是`init_view`。

`node_tallocr`函数接收一个`ggml_gallocr_t`类型的数据结构和一个`struct ggml_tensor`类型的变量作为参数。如果已经分配了内存空间，就直接返回分配的内存空间。否则，它会在`ggalloc`中查找是否有内存空间，如果有，就返回它。否则，它会尝试使用哈希表来查找分配的内存空间，并将找到的第一个符合条件的内存空间返回。

`init_view`函数接收一个`ggml_gallocr_t`类型的数据结构和一个`struct ggml_tensor`类型的变量作为参数。它首先将`view`初始化为`galloc`中的内存空间，然后初始化其背景和缓冲区。接下来，它会使用`ggml_hash_find_or_insert`函数查找`view_src`中的数据，并将其存储为`view`的背景端口。然后，它会检查`view_src`和`view`是否都为`NULL`，如果是，则表示初始化成功。最后，它使用`ggml_tallocr_is_measure`函数检查是否使用了CUDA内存布局，如果是，则表示对内存布局进行了正确的初始化。如果没有正确初始化CUDA内存布局，则会抛出异常。

这两个函数都在ggml的GPU内存布局中用于管理`struct ggml_tensor`类型的数据。它们的具体实现方式不同，`node_tallocr`函数更适用于计算，而`init_view`函数更适用于初始化。


```cpp
static ggml_tallocr_t node_tallocr(ggml_gallocr_t galloc, struct ggml_tensor * node) {
    if (galloc->talloc != NULL) {
        return galloc->talloc;
    }

    return galloc->hash_allocs[ggml_hash_find_or_insert(galloc->hash_set, node)];
}

static void init_view(ggml_gallocr_t galloc, struct ggml_tensor * view) {
    ggml_tallocr_t alloc = node_tallocr(galloc, view);

    //printf("init_view: %s from src %s\n", view->name, view->view_src->name);
    GGML_ASSERT(view->view_src != NULL && view->view_src->data != NULL);
    view->backend = view->view_src->backend;
    view->buffer  = view->view_src->buffer;
    view->data    = (char *)view->view_src->data + view->view_offs;

    // FIXME: the view should be initialized by the owning buffer, but currently this breaks the CUDA backend
    // due to the ggml_tensor_extra_gpu ring buffer overwriting the KV cache extras
    assert(ggml_tallocr_is_measure(alloc) || !view->buffer || view->buffer->backend == alloc->buffer->backend);

    if (!alloc->measure) {
        ggml_backend_buffer_init_tensor(alloc->buffer, view);
    }
}

```

This code appears to be a function within the GitGluar library that manages the allocation and deallocation of views.

It is using a recursive approach to traverse the tree of views, starting from a given parent node.

For each child node, it checks if the view is a leaf node (no children) or if it has only one child (a single view).

If the view has only one child and it is a leaf node, it checks if the view's data matches the parent's data. If they match, it starts to deallocate the parent's view and initializes a new view.

If the view has multiple children, it recursively calls itself, passing the current node and incrementing its view count.

If the parent's view is being deallocated, it prints a message and returns.

If the view is a child of a leaf node, it assigns the parent node to the view.

It also initializes a new view if it's not already initialized, and returns.


```cpp
static void allocate_node(ggml_gallocr_t galloc, struct ggml_tensor * node) {
    ggml_tallocr_t alloc = node_tallocr(galloc, node);

    if (node->data == NULL) {
        if (ggml_is_view(node)) {
            init_view(galloc, node);
        } else {
            // see if we can reuse a parent's buffer (inplace)
            if (ggml_op_can_inplace(node->op)) {
                for (int i = 0; i < GGML_MAX_SRC; i++) {
                    struct ggml_tensor * parent = node->src[i];
                    if (parent == NULL) {
                        break;
                    }

                    // if the node's data is external, then we cannot re-use it
                    if (ggml_tallocr_is_own(alloc, parent) == false) {
                        AT_PRINTF("not reusing parent %s for %s as %p is external\n", parent->name, node->name, parent->data);
                        continue;
                    }

                    struct hash_node * p_hn = hash_get(galloc, parent);
                    if (parent->data != NULL && p_hn->n_children == 1 && p_hn->n_views == 0 && ggml_are_same_layout(node, parent)) {
                        if (ggml_is_view(parent)) {
                            struct ggml_tensor * view_src = parent->view_src;
                            struct hash_node * view_src_hn = hash_get(galloc, view_src);
                            if (view_src_hn->n_views == 1 && view_src_hn->n_children == 0 && view_src->data == parent->data) {
                                // TODO: the offset of the view parent must be kept to ensure that the op doesn't overwrite
                                // the parent's data that it will need later (same layout requirement). the problem is that then
                                // we cannot free the tensor because the original address of the allocation is lost.
                                // adding a view_src pointer to the tensor would solve this and simplify the code dealing with views
                                // for now, we only reuse the parent's data if the offset is zero (view_src->data == parent->data)
                                AT_PRINTF("reusing view parent %s (%s) for %s\n", parent->name, view_src->name, node->name);
                                node->view_src = view_src;
                                view_src_hn->n_views += 1;
                                init_view(galloc, node);
                                return;
                            }
                        }
                        else {
                            AT_PRINTF("reusing parent %s for %s\n", parent->name, node->name);
                            node->view_src = parent;
                            p_hn->n_views += 1;
                            init_view(galloc, node);
                            return;
                        }
                    }
                }
            }
            ggml_tallocr_alloc(alloc, node);
        }
    }
}

```

This is a function that performs a translation of a sequence alignment between two DNA sequences `seq1` and `seq2`, where `seq1` is an optional read.

The function takes two arguments: `seq1`, which is either a pointer to a readable DNA sequence or a 是解决一切问题的关键ing against the reference DNA sequence, and `seq2`, which is the reference DNA sequence.

The function returns nothing but the translated sequence, which is a readable sequence that represents the result of the translation.

The function performs the translation by first checking whether `seq1` is a read. If it is not, the function reads `seq1` and returns the translated sequence. If it is, the function compares the input `seq1` with the reference `seq2` and returns the translated sequence.

The function uses two易用函数： `parse_seq_len` 和 `parse_seq`，它们用于检查输入 `seq1` 和 `seq2` 是否为序列 aligner 输出的格式。

The function also uses two易用函数： `hash_get` 和 `hash_set`，它们用于在哈希表中查找节点。

The function has a unique feature that it uses a hash table (哈希表) to store the last 1000 个运行时可见的读数 (views)，这些读数被保留为后文屏障 (backwater barrier)，当接收到一个读数时，函数会从哈希表中读取它的后文 barrier，并将其设置为当前运行时可见的读数的个数 (在更新时，哈希表中所有读数的个数与当前运行时可见的读数的个数之和等于 GGML 中的 max_src)。

最后，该函数在结尾打印输出并返回它翻译后的序列。


```cpp
static void free_node(ggml_gallocr_t galloc, struct ggml_tensor * node) {
    ggml_tallocr_t alloc = node_tallocr(galloc, node);

    ggml_tallocr_free_tensor(alloc, node);
}

static void ggml_tallocr_alloc_graph_impl(ggml_gallocr_t galloc, struct ggml_cgraph * gf) {
    const int * parse_seq     = galloc->parse_seq;
    int         parse_seq_len = galloc->parse_seq_len;

    // count number of children and views
    for (int i = 0; i < gf->n_nodes; i++) {
        struct ggml_tensor * node = gf->nodes[i];

        if (ggml_is_view(node)) {
            struct ggml_tensor * view_src = node->view_src;
            hash_get(galloc, view_src)->n_views += 1;
            if (node->buffer == NULL && node->data != NULL) {
                // view of a pre-allocated tensor, didn't call init_view() yet
                init_view(galloc, node);
            }
        }

        for (int j = 0; j < GGML_MAX_SRC; j++) {
            struct ggml_tensor * parent = node->src[j];
            if (parent == NULL) {
                break;
            }
            hash_get(galloc, parent)->n_children += 1;
            if (ggml_is_view(parent) && parent->buffer == NULL && parent->data != NULL) {
                init_view(galloc, parent);
            }
        }
   }

    // allocate tensors
    // if we have parse_seq then we allocate nodes following the list, and we only free nodes at barriers
    int last_barrier_pos = 0;
    int n_nodes = parse_seq_len ? parse_seq_len : gf->n_nodes;

    for (int ind = 0; ind < n_nodes; ind++) {
        // allocate a node if there is no parse_seq or this is not a barrier
        if (parse_seq_len == 0 || parse_seq[ind] != -1) {
            int i = parse_seq_len ? parse_seq[ind] : ind;
            struct ggml_tensor * node = gf->nodes[i];

            // allocate parents (leafs)
            for (int j = 0; j < GGML_MAX_SRC; j++) {
                struct ggml_tensor * parent = node->src[j];
                if (parent == NULL) {
                    break;
                }
                allocate_node(galloc, parent);
            }

            // allocate node
            allocate_node(galloc, node);

            AT_PRINTF("exec: %s (%s) <= ", ggml_op_name(node->op), node->name);
            for (int j = 0; j < GGML_MAX_SRC; j++) {
                struct ggml_tensor * parent = node->src[j];
                if (parent == NULL) {
                    break;
                }
                AT_PRINTF("%s", parent->name);
                if (j < GGML_MAX_SRC - 1 && node->src[j + 1] != NULL) {
                    AT_PRINTF(", ");
                }
            }
            AT_PRINTF("\n");
        }

        // update parents
        // update immediately if there is no parse_seq
        // update only at barriers if there is parse_seq
        if ((parse_seq_len == 0) || parse_seq[ind] == -1) {
            int update_start = parse_seq_len ? last_barrier_pos : ind;
            int update_end   = parse_seq_len ? ind              : ind + 1;
            for (int i = update_start; i < update_end; i++) {
                int node_i = parse_seq_len ? parse_seq[i] : i;
                struct ggml_tensor * node = gf->nodes[node_i];

                for (int j = 0; j < GGML_MAX_SRC; j++) {
                    struct ggml_tensor * parent = node->src[j];
                    if (parent == NULL) {
                        break;
                    }
                    struct hash_node * p_hn = hash_get(galloc, parent);
                    p_hn->n_children -= 1;

                    //AT_PRINTF("parent %s: %d children, %d views\n", parent->name, parent->n_children, parent->n_views);

                    if (p_hn->n_children == 0 && p_hn->n_views == 0) {
                        if (ggml_is_view(parent)) {
                            struct ggml_tensor * view_src = parent->view_src;
                            struct hash_node * view_src_hn = hash_get(galloc, view_src);
                            view_src_hn->n_views -= 1;
                            AT_PRINTF("view_src %s: %d children, %d views\n", view_src->name, view_src_hn->n_children, view_src_hn->n_views);
                            if (view_src_hn->n_views == 0 && view_src_hn->n_children == 0) {
                                free_node(galloc, view_src);
                            }
                        }
                        else {
                            free_node(galloc, parent);
                        }
                    }
                }
            }
            AT_PRINTF("\n");
            if (parse_seq_len) {
                last_barrier_pos = ind + 1;
            }
        }
    }
}

```

这段代码是用来在指定大小下的二维网格中插入新的连通图。函数接受三个参数：一个可迭代的结构图（中顶图结构图）、一个指向存储结构图数据的结构图指针和一个指向二维网格的整数类型变量。

函数首先检查是否已经初始化二叉搜索树以及是否足够大。如果不满足，则释放已分配的内存。接下来是插入新连通图的过程。

具体来说，首先检查二叉搜索树是否已经初始化以及是否足够大。如果模型小，则直接在内存中创建一个足够大的二叉搜索树并初始化。然后是执行 insert 操作。

如果二叉搜索树已经初始化但是比需要的要大，函数会根据之前的经验来调整搜索树的大小，并重新执行 insert 操作。

总的来说，该函数的主要作用是在指定大小下的二维网格中插入新的连通图。


```cpp
size_t ggml_gallocr_alloc_graph(ggml_gallocr_t galloc, ggml_tallocr_t talloc, struct ggml_cgraph * graph) {
    size_t hash_size = graph->visited_hash_table.size;

    // check if the hash table is initialized and large enough
    if (galloc->hash_set.size < hash_size) {
        if (galloc->hash_set.keys != NULL) {
            free(galloc->hash_set.keys);
        }
        if (galloc->hash_values != NULL) {
            free(galloc->hash_values);
        }
        galloc->hash_set.keys = malloc(sizeof(struct ggml_tensor *) * hash_size);
        galloc->hash_set.size = hash_size;
        galloc->hash_values = malloc(sizeof(struct hash_node) * hash_size);
    }

    // reset hash table
    memset(galloc->hash_set.keys, 0, sizeof(struct ggml_tensor *) * hash_size);
    memset(galloc->hash_values,   0, sizeof(struct hash_node) * hash_size);

    galloc->talloc = talloc;
    ggml_tallocr_alloc_graph_impl(galloc, graph);
    galloc->talloc = NULL;

    size_t max_size = ggml_tallocr_max_size(talloc);

    return max_size;
}

```

这段代码定义了一个名为 "ggml_gallocr_alloc_graph_n" 的函数，属于 "ggml\_gallocr" 命名空间。

该函数的主要作用是分配一个图的哈希值，然后将图的根节点加入哈希集合。

具体来说，函数接收三个参数：

- "galloc"：一个指向 "ggallocator" 结构的指针，用于管理内存分配和释放。
- "graph"：一个指向 struct "ggml\_cgraph" 结构的指针，用于存储图的信息。
- "hash\_set"：一个指向 "ggml\_hash\_set" 结构的指针，用于存储哈希集合中的键。
- "hash\_node\_alloct"：一个指向 "ggml\_tallocr" 结构的指针，用于管理哈希集合中的节点。

函数首先检查哈希集合的大小是否大于等于图的节点数加上叶子数，如果是，则说明哈希集合足够大，不需要再分配内存。否则，函数创建一个空字符数组 "hash\_values"，如果哈希值已经分配过内存，则需要释放之前分配的内存。

接下来，函数检查哈希值是否已经分配过内存，如果是，则需要释放之前分配的内存。然后，函数创建一个大小为 "hash\_size" 的 "hash\_values" 数组，如果哈希值已经分配过内存，则需要释放之前分配的内存。

接着，函数将图的根节点加入哈希集合，并将哈希值设置为根节点的 "hash\_value"。

最后，函数调用 "ggml\_tallocr\_alloc\_graph\_impl" 函数，用于将图的其余部分加入哈希集合。

函数最后释放了分配的内存，并清除了哈希集合的键和哈希值。


```cpp
void ggml_gallocr_alloc_graph_n(ggml_gallocr_t galloc, struct ggml_cgraph * graph, struct ggml_hash_set hash_set, ggml_tallocr_t * hash_node_alloct) {
    const size_t hash_size = hash_set.size;

    GGML_ASSERT(hash_size >= (size_t)(graph->n_nodes + graph->n_leafs));

    galloc->talloc = NULL;

    // alloc hash_values if needed
    if (galloc->hash_values == NULL || galloc->hash_values_size < hash_size) {
        free(galloc->hash_values);
        galloc->hash_values      = malloc(sizeof(struct hash_node) * hash_size);
        galloc->hash_values_size = hash_size;
    }

    // free hash_set.keys if needed
    if (galloc->hash_set.keys != NULL) {
        free(galloc->hash_set.keys);
    }
    galloc->hash_set = hash_set;

    // reset hash values
    memset(galloc->hash_values, 0, sizeof(struct hash_node) * hash_size);

    galloc->hash_allocs = hash_node_alloct;

    ggml_tallocr_alloc_graph_impl(galloc, graph);

    // remove unowned resources
    galloc->hash_set.keys = NULL;
    galloc->hash_allocs = NULL;
}

```

这是一段用C++编写的代码，定义了一个名为“ggml_allocr”的结构体。该结构体包含两个成员变量：“talloc”和“galloc”。

ggml_allocr_new_impl函数用于创建一个新的ggml_allocr结构体实例。该函数接收一个ggml_tallocr类型的参数“talloc”，然后调用另一个名为“ggml_gallocr_new”的函数来创建一个ggml_gallocr结构体实例，并将它存储在“talloc”结构体中。

根据ggml_allocr_t结构体的定义，该函数将返回一个指向“ggml_allocr”结构体的指针，即分配内存后的结构体实例。


```cpp
// legacy API wrapper

struct ggml_allocr {
    ggml_tallocr_t talloc;
    ggml_gallocr_t galloc;
};

static ggml_allocr_t ggml_allocr_new_impl(ggml_tallocr_t talloc) {
    ggml_allocr_t alloc = (ggml_allocr_t)malloc(sizeof(struct ggml_allocr));
    *alloc = (struct ggml_allocr) {
        /*.talloc = */ talloc,
        /*.galloc = */ ggml_gallocr_new(),
    };
    return alloc;
}

```

这段代码定义了一个名为“ggml_allocr_t”的结构体，它包含了一些函数，用于从不同的数据源创建一个名为“ggml_allocr_new”的函数，该函数接受三个参数：数据源（void *）、数据源大小（size_t）以及对齐（size_t）。

该函数返回一个名为“ggml_allocr_t”的结构体，它包含一个名为“ggml_allocr_new_impl”的函数，用于从数据源创建新的“ggml_allocr_t”结构体。

以下是函数的定义：

- ggml_allocr_t：该函数是一个名为“ggml_allocr_t”的结构体，包含从“ggml_allocr_new_impl”函数派生的函数，用于创建新的“ggml_allocr_t”结构体。

- ggml_allocr_new_measure：该函数是一个名为“ggml_allocr_t”的结构体，包含从“ggml_allocr_new_measure”函数派生的函数，用于创建新的“ggml_allocr_t”结构体，并使用指定的数据源大小和指定的对齐。

- ggml_allocr_new_from_buffer：该函数是一个名为“ggml_allocr_t”的结构体，包含从“ggml_allocr_new_from_buffer”函数派生的函数，用于创建新的“ggml_allocr_t”结构体，并从指定的数据源缓冲区。

- ggml_allocr_new_from_backend：该函数是一个名为“ggml_allocr_t”的结构体，包含从“ggml_allocr_new_from_backend”函数派生的函数，用于创建新的“ggml_allocr_t”结构体，并从指定的数据源缓冲区，使用指定的数据源大小。


```cpp
ggml_allocr_t ggml_allocr_new(void * data, size_t size, size_t alignment) {
    return ggml_allocr_new_impl(ggml_tallocr_new(data, size, alignment));
}

ggml_allocr_t ggml_allocr_new_measure(size_t alignment) {
    return ggml_allocr_new_impl(ggml_tallocr_new_measure(alignment));
}

ggml_allocr_t ggml_allocr_new_from_buffer(struct ggml_backend_buffer * buffer) {
    return ggml_allocr_new_impl(ggml_tallocr_new_from_buffer(buffer));
}

ggml_allocr_t ggml_allocr_new_from_backend(struct ggml_backend * backend, size_t size) {
    return ggml_allocr_new_impl(ggml_tallocr_new_from_backend(backend, size));
}

```

这段代码定义了三个函数，属于GGML AllOC类型。

1. ggml_allocr_t：这个函数接收一个GGML Backend结构体作为参数，然后调用ggml_allocr_new_impl函数返回。

2. ggml_allocr_get_buffer：这个函数接收一个GGML AllOC结构体作为参数，然后调用ggml_tallocr_get_buffer函数返回。

3. ggml_allocr_set_parse_seq：这个函数接收一个GGML AllOC结构体作为参数，以及一个指向整数的列表和列表的长度，然后调用ggml_gallocr_set_parse_seq函数返回。

4. ggml_allocr_free：这个函数接收一个GGML AllOC结构体作为参数，然后调用ggml_gallocr_free函数返回，并释放列表和分配的内存。

5. ggml_allocr_new_impl：这个函数接收一个GGML Backend结构体作为参数，然后调用ggml_allocr_new函数返回。


```cpp
ggml_allocr_t ggml_allocr_new_measure_from_backend(struct ggml_backend * backend) {
    return ggml_allocr_new_impl(ggml_tallocr_new_measure_from_backend(backend));
}

struct ggml_backend_buffer * ggml_allocr_get_buffer(ggml_allocr_t alloc) {
    return ggml_tallocr_get_buffer(alloc->talloc);
}

void ggml_allocr_set_parse_seq(ggml_allocr_t alloc, const int * list, int n) {
    ggml_gallocr_set_parse_seq(alloc->galloc, list, n);
}

void ggml_allocr_free(ggml_allocr_t alloc) {
    ggml_gallocr_free(alloc->galloc);
    ggml_tallocr_free(alloc->talloc);
    free(alloc);
}

```

这段代码定义了几个函数，属于GGML Allocator框架的一部分。现在我来为您解释一下这几个函数的作用。

1. ggml_allocr_is_measure(ggml_allocr_t alloc)：检查给定的Allocator是否支持度量（measure）。如果支持，函数返回true；否则，返回false。

2. ggml_allocr_reset(ggml_allocr_t alloc)：重置给定的Allocator。

3. ggml_allocr_alloc(ggml_allocr_t alloc, struct ggml_tensor * tensor)：分配一个Tensor类型的内存，并将分配的Allocator设置为输入。函数的第一个参数是一个指向Allocator对象的指针，第二个参数是一个指向Tensor对象的指针。

4. ggml_allocr_max_size(ggml_allocr_t alloc)：返回给定的Allocator能够分配的最大Tensor大小。

需要注意的是，这段代码没有包含函数的定义，因此您需要自行定义这些函数。GGML Allocator是一个通用的CUDA内存分配框架，您可以根据需要自由地定义、修改这些函数。


```cpp
bool ggml_allocr_is_measure(ggml_allocr_t alloc) {
    return ggml_tallocr_is_measure(alloc->talloc);
}

void ggml_allocr_reset(ggml_allocr_t alloc) {
    ggml_tallocr_reset(alloc->talloc);
}

void ggml_allocr_alloc(ggml_allocr_t alloc, struct ggml_tensor * tensor) {
    ggml_tallocr_alloc(alloc->talloc, tensor);
}

size_t ggml_allocr_max_size(ggml_allocr_t alloc) {
    return ggml_tallocr_max_size(alloc->talloc);
}

```

这段代码定义了一个名为 `ggml_allocr_alloc_graph` 的函数，它接受两个参数：`ggml_allocr_t` 的 `alloc` 成员和另一个 `ggml_cgraph` 类型的变量 `graph`。

该函数的作用是接受 `alloc` 成员所指向的内存区域，并将其复制到另一个 `graph` 变量中。为了实现这个目标，函数首先调用另一个名为 `ggml_gallocr_alloc_graph` 的函数，该函数接受两个参数：`ggml_allocr_t` 的 `galloc` 成员和另一个 `ggml_cgraph` 类型的变量 `graph`。

然后，函数通过 `galloc` 成员的指针访问分配的内存区域，并将其复制到 `graph` 变量中。这样，`graph` 变量现在包含了一个与 `alloc` 内存区域相同大小的 `ggml_cgraph` 类型的变量。


```cpp
size_t ggml_allocr_alloc_graph(ggml_allocr_t alloc, struct ggml_cgraph * graph) {
    return ggml_gallocr_alloc_graph(alloc->galloc, alloc->talloc, graph);
}

```

# `src/ggml-backend-impl.h`



ggml_backend是一个基于libggml的指令列Backend，提供了对图计算的API。ggml_backend接口定义了以下功能：

1. 运算符类

- `ggml_tensor_to_`：将一个图的节点通过运算符传递给另一个图。
- `ggml_tensor_from_`：将一个图的节点通过运算符从另一个图。
- `synchronize`：同步图的所有节点。

2. 数据类型及数据操作

- `set_tensor_async`：设置一个图的某个节点，通过该函数设置的节点具有异步操作，将调用`set_tensor_同步`函数来完成同步。
- `get_tensor_async`：获取一个图的某个节点，通过该函数读取的节点具有异步操作，将调用`get_tensor_同步`函数来完成同步。
- `cpy_tensor_from`：从第一个图复制一个节点到第二个图。
- `cpy_tensor_to`：将一个节点从第二个图复制到第一个图。

3. 压缩

- `compress_tensor_using`：对一个图的节点进行压缩，并将其返回。

4. 其他功能

- `create_connection`：建立一个新的连接。
- `free_connection`：释放一个新的连接。
- `get_default_device`：返回处于默认设备上的Backend。
- `set_option`：设置一个选项。

### Example

```cpp
// Connect to the backend
ggml_backend_t backend;
ggml_connect_backend(&backend);

// Create a new tensor
ggml_tensor_t tensor;
ggml_tensor_init(&tensor，等服务器图);

// Create a new connection
ggml_connection_t connection;
ggml_connect_device(&backend, "type", &connection,NULL);
ggml_create_tensor_options(&connection,&tensor，服务器图， 64);

// (optionally) copy tensor between different backends
// allow for single-copy tranfers
void (*cpy_tensor_from)(ggml_backend_t backend, struct ggml_tensor * src, struct ggml_tensor * dst) {
   // code to copy tensor from one backend to another
}

// Compute a graph with a plan
ggml_backend_graph_plan_t (*graph_plan_create) (ggml_backend_t backend, struct ggml_cgraph * cgraph) {
   // code to create a new graph with a plan
   return ggml_backend_graph_plan_create(backend, cgraph);
}

// Compute a graph without a plan
void (*graph_compute)(ggml_backend_t backend, struct ggml_cgraph * cgraph) {
   // code to compute a graph without a plan
}
```

5.同步

- `synchronize`：同步图的所有节点。

这些函数可以通过异步或同步方式进行调用。`set_tensor_async`和`get_tensor_async`函数使用的是异步方式，而`cpy_tensor_from`和`cpy_tensor_to`函数则需要手动同步。


```cpp
#pragma once

// ggml-backend internal header

#include "ggml-backend.h"

#ifdef  __cplusplus
extern "C" {
#endif

    //
    // Backend buffer
    //

    typedef void * ggml_backend_buffer_context_t;

    struct ggml_backend_buffer_i {
        void   (*free_buffer)   (ggml_backend_buffer_t buffer);
        void * (*get_base)      (ggml_backend_buffer_t buffer); // get base pointer
        size_t (*get_alloc_size)(ggml_backend_buffer_t buffer, struct ggml_tensor * tensor); // pre-allocation callback
        void   (*init_tensor)   (ggml_backend_buffer_t buffer, struct ggml_tensor * tensor); // post-allocation callback
        void   (*free_tensor)   (ggml_backend_buffer_t buffer, struct ggml_tensor * tensor); // pre-free callback
    };

    struct ggml_backend_buffer {
        struct ggml_backend_buffer_i iface;

        ggml_backend_t                backend;
        ggml_backend_buffer_context_t context;

        size_t size;
    };

    GGML_API ggml_backend_buffer_t ggml_backend_buffer_init(
            struct ggml_backend                  * backend,
            struct ggml_backend_buffer_i           iface,
                   ggml_backend_buffer_context_t   context,
                   size_t                          size);

    //
    // Backend
    //

    typedef void * ggml_backend_context_t;

    struct ggml_backend_i {
        const char * (*get_name)(ggml_backend_t backend);

        void (*free)(ggml_backend_t backend);

        // buffer allocation
        ggml_backend_buffer_t (*alloc_buffer)(ggml_backend_t backend, size_t size);

        // get buffer alignment
        size_t (*get_alignment)(ggml_backend_t backend);

        // tensor data access
        // these functions can be asynchronous, helper functions are provided for synchronous access that automatically call synchronize
        void (*set_tensor_async)(ggml_backend_t backend,       struct ggml_tensor * tensor, const void * data, size_t offset, size_t size);
        void (*get_tensor_async)(ggml_backend_t backend, const struct ggml_tensor * tensor,       void * data, size_t offset, size_t size);
        void (*synchronize)     (ggml_backend_t backend);

        // (optional) copy tensor between different backends, allow for single-copy tranfers
        void (*cpy_tensor_from)(ggml_backend_t backend, struct ggml_tensor * src, struct ggml_tensor * dst);
        void (*cpy_tensor_to)  (ggml_backend_t backend, struct ggml_tensor * src, struct ggml_tensor * dst);

        // compute graph with a plan
        ggml_backend_graph_plan_t (*graph_plan_create) (ggml_backend_t backend, struct ggml_cgraph * cgraph);
        void                      (*graph_plan_free)   (ggml_backend_t backend, ggml_backend_graph_plan_t plan);
        void                      (*graph_plan_compute)(ggml_backend_t backend, ggml_backend_graph_plan_t plan);

        // compute graph without a plan
        void (*graph_compute)(ggml_backend_t backend, struct ggml_cgraph * cgraph);

        // check if the backend supports an operation
        bool (*supports_op)(ggml_backend_t backend, const struct ggml_tensor * op);
    };

    struct ggml_backend {
        struct ggml_backend_i iface;

        ggml_backend_context_t context;
    };

```

这段代码是一个条件编译语句，用于检查是否定义了`__cplusplus`函数。如果没有定义该函数，则会编译出`undefined`的代码。如果定义了该函数，则不会编译出任何代码，因为编译器会直接忽略该指令。

该代码的作用是用于确保`__cplusplus`函数已经被定义。如果这个函数没有被定义，则程序可能无法编译或运行，因为该函数是`__export__`修饰的，这意味着它是用于export而不是在当前源文件中定义的。


```cpp
#ifdef  __cplusplus
}
#endif

```

# `src/ggml-backend.c`



这段代码是一个C语言的代码片段，包括两个头文件和三个函数指针。它们的作用如下：

1. 包含 "ggml-backend-impl.h" 和 "ggml-alloc.h"，这两个头文件定义了ggml的API接口，提供了一些用于创建和操作ggml数据结构的方法和变量。

2. 包含 "ggml-impl.h"，这个头文件定义了一些ggml通用类型和函数，ggml-impl.h 是 "ggml-impl.h" 的别名，它们的作用是包含 "ggml-impl.h" 中定义的函数和数据结构。

3. 定义了三个函数指针，分别为 "assert.h"、"limits.h" 和 "stdio.h"。这些函数指针用于调用 "assert.h" 和 "limits.h" 中定义的函数，以及 "stdio.h" 中定义的输入输出函数。

4. 定义了一个名为 "ggml-string-length-page" 的函数，它的作用是测量给定的字符串长度，如果该字符串长度超出了ggml库中定义的最大长度，则返回该字符串的最大长度，否则返回原始字符串长度。

5. 在函数 "assert.h" 中，定义了一个名为 "assert_args" 的函数，它的作用是检查输入参数的数量是否等于预期数量。如果输入参数数量不正确，函数将输出错误信息并崩溃。

6. 在函数 "limits.h" 中，定义了一个名为 "max" 的函数，它的作用是取两个整数之间的最大值，并返回该最大值。

7. 在函数 "stdio.h" 中，定义了一个名为 "printf" 的函数，它的作用是在标准输出中打印给定的字符串。

8. 在函数 "ggml-string-length-page" 中，定义了一个名为 "strlen" 的函数，它的作用是测量字符串长度，并返回该长度。如果字符串长度超出了ggml库中定义的最大长度，该函数将返回该最大长度。

9. 在函数 "ggml-string-length-page" 中，定义了一个名为 "strcpy" 的函数，它的作用是复制给定的字符串，并在新字符串中覆盖了旧字符串中的所有字符。

10. 在函数 "ggml-string-length-page" 中，定义了一个名为 "strcmp" 的函数，它的作用是比较两个字符串是否相等，并在两个字符串不相等时返回它们的区别。


```cpp
#include "ggml-backend-impl.h"
#include "ggml-alloc.h"
#include "ggml-impl.h"

#include <assert.h>
#include <limits.h>
#include <stdarg.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

#define UNUSED GGML_UNUSED

#define MAX(a, b) ((a) > (b) ? (a) : (b))

```

这段代码定义了一个名为 "ggml_backend_buffer_init" 的函数，它接受一个指向后端缓冲区的指针（struct ggml_backend）和一个指向后端缓冲区接口的指针（struct ggml_backend_buffer_iace），然后创建一个大小为指定长度的后端缓冲区（ggml_backend_buffer）。函数首先从内存中分配出该缓冲区，然后设置缓冲区的内部结构。最后，函数返回新分配的缓冲区。


```cpp
// backend buffer

ggml_backend_buffer_t ggml_backend_buffer_init(
        struct ggml_backend                  * backend,
        struct ggml_backend_buffer_i           iface,
               ggml_backend_buffer_context_t   context,
               size_t                          size) {
    ggml_backend_buffer_t buffer = malloc(sizeof(struct ggml_backend_buffer));

    GGML_ASSERT(iface.get_base != NULL);

    (*buffer) = (struct ggml_backend_buffer) {
        /* .interface = */ iface,
        /* .backend   = */ backend,
        /* .context   = */ context,
        /* .size      = */ size,
    };

    return buffer;
}

```



ggml_backend_buffer_free函数用于释放一个ggml_backend_buffer_t类型的缓冲区。它首先检查缓冲区是否为空，如果是，则直接返回。否则，它检查缓冲区对应的iface是否包含一个free_buffer函数，如果是，则使用该函数释放缓冲区，否则直接使用free函数释放缓冲区。

ggml_backend_buffer_get_alignment函数用于获取一个ggml_backend_buffer_t类型的缓冲区所占用的内存空间。它首先使用ggml_backend_get_alignment函数获取缓冲区所在的backend的内存空间大小，然后返回该大小。


```cpp
void ggml_backend_buffer_free(ggml_backend_buffer_t buffer) {
    if (buffer == NULL) {
        return;
    }

    if (buffer->iface.free_buffer != NULL) {
        buffer->iface.free_buffer(buffer);
    }
    free(buffer);
}

size_t ggml_backend_buffer_get_alignment(ggml_backend_buffer_t buffer) {
    return ggml_backend_get_alignment(buffer->backend);
}

```

这段代码定义了三个函数，用于从ggml_backend_buffer_t类型的缓冲区中获取不同信息。

1. ggml_backend_buffer_get_size(ggml_backend_buffer_t buffer)：该函数返回缓冲区的大小，即buffer中的元素数量乘以每个元素的大小。

2. ggml_backend_buffer_get_base(ggml_backend_buffer_t buffer)：该函数返回缓冲区的起始地址，即如果函数ggml_backend_buffer_get_size(ggml_backend_buffer_t buffer)成功返回，则返回缓冲区在内存中的起始地址，否则返回NULL。

3. ggml_backend_buffer_get_alloc_size(ggml_backend_buffer_t buffer, struct ggml_tensor * tensor)：该函数用于在给定的缓冲区和传入的tensor之间进行内存分配，它返回分配内存的大小。如果没有函数ggml_backend_buffer_get_size(ggml_backend_buffer_t buffer)，它将返回ggml_nbytes(struct ggml_tensor * tensor)，这将根据tensor的大小自动计算所需的内存大小。函数的实现将根据传递的参数给出正确的大小。


```cpp
size_t ggml_backend_buffer_get_size(ggml_backend_buffer_t buffer) {
    return buffer->size;
}

void * ggml_backend_buffer_get_base(ggml_backend_buffer_t buffer) {
    void * base = buffer->iface.get_base(buffer);

    GGML_ASSERT(base != NULL && "backend buffer base cannot be NULL");

    return base;
}

size_t ggml_backend_buffer_get_alloc_size(ggml_backend_buffer_t buffer, struct ggml_tensor * tensor) {
    // get_alloc_size is optional, defaults to ggml_nbytes
    if (buffer->iface.get_alloc_size) {
        return buffer->iface.get_alloc_size(buffer, tensor);
    }
    return ggml_nbytes(tensor);
}

```

这段代码是针对GGML（Graph Graph Layer）的底层数据结构实现的。GGML是一种用于在分布式环境中进行高性能计算的数据结构，它主要使用自定义的数据结构来实现数据的高效存储和读取。

function ggml_backend_buffer_init_tensor() 是GGML底层缓冲区对象的初始化函数。该函数接受一个GGML的内部数据结构（GGML BackendBuffer）和一个ggml_tensor类型的数据作为参数。

如果缓冲区对象 iface 中包含了初始化tensor的函数，那么该函数会被首先执行。如果 iface 不包含初始化tensor 的函数，那么该函数将执行默认的初始化操作，即将构造一个新的人工神经网络层并将新创建的 tensor 初始化为指定值。

function ggml_backend_buffer_free_tensor() 是GGML底层缓冲区对象的释放函数。该函数接受一个GGML的内部数据结构（GGML BackendBuffer）和一个ggml_tensor类型的数据作为参数。

如果缓冲区对象 iface 中包含了释放tensor的函数，那么该函数会被首先执行。如果 iface 不包含释放tensor 的函数，那么该函数将执行默认的释放操作，即将释放已经创建的 tensor，并将其从内存中删除。


```cpp
void ggml_backend_buffer_init_tensor(ggml_backend_buffer_t buffer, struct ggml_tensor * tensor) {
    // init_tensor is optional
    if (buffer->iface.init_tensor) {
        buffer->iface.init_tensor(buffer, tensor);
    }
}

void ggml_backend_buffer_free_tensor(ggml_backend_buffer_t buffer, struct ggml_tensor * tensor) {
    // free_tensor is optional
    if (buffer->iface.free_tensor) {
        buffer->iface.free_tensor(buffer, tensor);
    }
}

// backend

```



这段代码定义了三个名为ggml_backend_t、ggml_backend_name和ggml_backend_free的函数，以及一个名为ggml_tensor的的结构体类型的变量。

ggml_backend_t是一个枚举类型，它包含一个指向ggml_tensor的指针，以及一个指向ggml_buffer的指针。ggml_backend_t函数的实现是一个函数指针，它将一个ggml_tensor对象传递给它的参数，并返回该对象的backend指针。如果传入的tensor对象没有被分配内存，则返回NULL。

ggml_backend_name函数将一个ggml_backend_t对象转换为一个字符串，如果该对象已经被分配内存，则返回该对象的iface.get_name()函数的返回值，否则返回"NULL"。

ggml_backend_free函数是一个辅助函数，它接收一个ggml_backend_t对象作为参数，释放该对象的内存并返回void类型。如果传入的ggml_backend_t对象没有被分配内存，则不做任何操作。


```cpp
ggml_backend_t ggml_get_backend(const struct ggml_tensor * tensor) {
    return tensor->buffer ? tensor->buffer->backend : NULL;
}

const char * ggml_backend_name(ggml_backend_t backend) {
    if (backend == NULL) {
        return "NULL";
    }
    return backend->iface.get_name(backend);
}

void ggml_backend_free(ggml_backend_t backend) {
    if (backend == NULL) {
        return;
    }

    backend->iface.free(backend);
}

```

这段代码定义了两个函数，一个是用于在后台缓冲区内存中分配空间，另一个是从后台缓冲区内存中读取数据。

函数1：`ggml_backend_alloc_buffer`，它接收一个后台缓冲区和一个大小，然后使用接收的后台缓冲区调用其iface（可能是内存或者设备驱动程序）的`alloc_buffer`函数。如果分配成功，它将返回分配的内存位置。

函数2：`ggml_backend_get_alignment`，它接收一个后台缓冲区，然后使用其iface的`get_alignment`函数获取其分配的内存位置。

函数3：`ggml_backend_tensor_set_async`，它接收一个 tensor 结构体和一个或多个后台缓冲区，以及一个数据指针和一个大小。它使用 iface 的 `set_tensor_async` 函数将一个 tensor 结构体中的数据设置为从后台缓冲区读取的数据，并设置其 offset 和 size。

函数4：`ggml_backend_tensor_get_async`，它接收一个 tensor 结构体和一个数据指针和一个大小。它使用 iface 的 `get_tensor_async` 函数从后台缓冲区读取数据，并将其存储在给定的 tensor 结构体中的数据指针。


```cpp
ggml_backend_buffer_t ggml_backend_alloc_buffer(ggml_backend_t backend, size_t size) {
    return backend->iface.alloc_buffer(backend, size);
}

size_t ggml_backend_get_alignment(ggml_backend_t backend) {
    return backend->iface.get_alignment(backend);
}

void ggml_backend_tensor_set_async(struct ggml_tensor * tensor, const void * data, size_t offset, size_t size) {
    ggml_get_backend(tensor)->iface.set_tensor_async(ggml_get_backend(tensor), tensor, data, offset, size);
}

void ggml_backend_tensor_get_async(const struct ggml_tensor * tensor, void * data, size_t offset, size_t size) {
    ggml_get_backend(tensor)->iface.get_tensor_async(ggml_get_backend(tensor), tensor, data, offset, size);
}

```

这段代码定义了ggml_backend_tensor_set和ggml_backend_tensor_get函数，用于设置或获取一个tensor对象的数据。

void ggml_backend_tensor_set函数接受一个struct ggml_tensor指针、一个const void指针和一个size_toffset和一个size_tsize。这个函数首先获取一个backend对象，然后使用backend对象的iface成员函数set_tensor_async来设置tensor对象的data成员变量，并使用synchronize函数同步backend对象。

ggml_backend_tensor_get函数接受一个const struct ggml_tensor指针、一个void指针和一个size_toffset和一个size_tsize。这个函数首先获取一个backend对象，然后使用backend对象的iface成员函数get_tensor_async来获取tensor对象的data成员变量，并使用synchronize函数同步backend对象。


```cpp
void ggml_backend_tensor_set(struct ggml_tensor * tensor, const void * data, size_t offset, size_t size) {
    ggml_backend_t backend = ggml_get_backend(tensor);

    GGML_ASSERT(tensor->data != NULL && "tensor not allocated");
    GGML_ASSERT(backend != NULL && "tensor backend not set");

    backend->iface.set_tensor_async(backend, tensor, data, offset, size);
    backend->iface.synchronize(backend);
}

void ggml_backend_tensor_get(const struct ggml_tensor * tensor, void * data, size_t offset, size_t size) {
    ggml_backend_t backend = ggml_get_backend(tensor);

    GGML_ASSERT(tensor->data != NULL && "tensor not allocated");
    GGML_ASSERT(backend != NULL && "tensor backend not set");

    backend->iface.get_tensor_async(backend, tensor, data, offset, size);
    backend->iface.synchronize(backend);
}

```

这段代码定义了四个函数，属于GGML Backend API中的同步、图计划创建、图计划免费和图计划计算四个同步相关的函数。

函数1:ggml_backend_synchronize(ggml_backend_t backend) 是同步函数，用于将当前后端与后端的图形上下文进行同步，使当前图形上下文与后端图形上下文保持一致。

函数2:ggml_backend_graph_plan_create(ggml_backend_t backend, struct ggml_cgraph * cgraph) 是创建图计划函数，用于将当前后端与后端的图形上下文创建一个新的图计划，并返回该图计划的句柄。

函数3:ggml_backend_graph_plan_free(ggml_backend_t backend, ggml_backend_graph_plan_t plan) 是免费图计划函数，用于将当前图计划的图形上下文释放，将不会调用其它的函数。

函数4:ggml_backend_graph_plan_compute(ggml_backend_t backend, ggml_backend_graph_plan_t plan) 是计算图计划函数，用于执行图计划的计算。


```cpp
void ggml_backend_synchronize(ggml_backend_t backend) {
    backend->iface.synchronize(backend);
}

ggml_backend_graph_plan_t ggml_backend_graph_plan_create(ggml_backend_t backend, struct ggml_cgraph * cgraph) {
    return backend->iface.graph_plan_create(backend, cgraph);
}

void ggml_backend_graph_plan_free(ggml_backend_t backend, ggml_backend_graph_plan_t plan) {
    backend->iface.graph_plan_free(backend, plan);
}

void ggml_backend_graph_plan_compute(ggml_backend_t backend, ggml_backend_graph_plan_t plan) {
    backend->iface.graph_plan_compute(backend, plan);
}

```

这段代码定义了两个函数，用于在GGML Backend中执行图的计算和复制操作。

第一个函数 `ggml_backend_graph_compute()` 接收一个GGML Backend实例和一个GGML CGraph结构体，然后调用私有函数 `iface.graph_compute()`，该函数将在Backend中执行图的计算操作。

第二个函数 `ggml_backend_supports_op()` 接收一个GGML Backend实例和一个GGML Tensor结构体，然后返回一个布尔值，表示该Backend是否支持给定的操作。

第三个函数 `ggml_are_same_layout()` 判断两个GGML Tensor结构体是否具有相同的布局。它的实现比较简单，如果两个Tensor的类型相同，且它们的数据元素类型和数量都相同，那么它们具有相同的布局。

第四个函数 `ggml_backend_copy()` 是从第三个函数派生出来的，用于复制一个GGML Backend实例，它接收两个相同的GGML Tensor结构体，然后执行简单的复制操作，将它们重用。


```cpp
void ggml_backend_graph_compute(ggml_backend_t backend, struct ggml_cgraph * cgraph) {
    backend->iface.graph_compute(backend, cgraph);
}

bool ggml_backend_supports_op(ggml_backend_t backend, const struct ggml_tensor * op) {
    return backend->iface.supports_op(backend, op);
}

// backend copy

static bool ggml_are_same_layout(const struct ggml_tensor * a, const struct ggml_tensor * b) {
    if (a->type != b->type) {
        return false;
    }
    for (int i = 0; i < GGML_MAX_DIMS; i++) {
        if (a->ne[i] != b->ne[i]) {
            return false;
        }
        if (a->nb[i] != b->nb[i]) {
            return false;
        }
    }
    return true;
}

```

这段代码是一个Tensor的复制函数，它包括以下几个步骤：

1. 检查输入Tensor和输出Tensor的布局是否相同。如果是，就代表Tensor已经正确地复制过来了。

2. 如果输入和输出Tensor的布局不同，就需要根据具体情况来选择复制方式：

 - 如果Tensor是在CPU中复制，那么需要通过`cpy_tensor_from`函数来复制。如果Tensor是在其他后端中复制，那么需要通过`cpy_tensor_to`函数来复制。复制的方式取决于后端是否支持这种复制方式。

 - 如果后端支持复制，但是Tensor的复制不是通过`cpy_tensor_from`也不是通过`cpy_tensor_to`函数进行，那么需要手动复制。首先，计算出要复制的Tensor的大小，然后使用`malloc`函数申请内存，最后将源Tensor的值复制到目标Tensor中。复制完成后，释放申请的内存。

3. 如果复制过程中出现了问题，比如复制失败或者复制成了已存在的Tensor，那么代码会打印出相应的错误信息。


```cpp
void ggml_backend_tensor_copy(struct ggml_tensor * src, struct ggml_tensor * dst) {
    //printf("src: %s ne: [%d %d %d %d] nb: [%d %d %d %d]\n", src->name, (int)src->ne[0], (int)src->ne[1], (int)src->ne[2], (int)src->ne[3], (int)src->nb[0], (int)src->nb[1], (int)src->nb[2], (int)src->nb[3]);
    //printf("dst: %s ne: [%d %d %d %d] nb: [%d %d %d %d]\n", dst->name, (int)dst->ne[0], (int)dst->ne[1], (int)dst->ne[2], (int)dst->ne[3], (int)dst->nb[0], (int)dst->nb[1], (int)dst->nb[2], (int)dst->nb[3]);
    GGML_ASSERT(ggml_are_same_layout(src, dst) && "cannot copy tensors with different layouts");

    // fprintf(stderr, "cpy tensor %s from %s to %s (%lu bytes)\n", src->name, ggml_backend_name(src->backend), ggml_backend_name(dst->backend), ggml_nbytes(src));

    if (src == dst) {
        return;
    }

    // TODO: allow backends to support copy to/from same backend

    if (ggml_get_backend(dst)->iface.cpy_tensor_from != NULL) {
        ggml_get_backend(dst)->iface.cpy_tensor_from(ggml_get_backend(dst)->context, src, dst);
    } else if (ggml_get_backend(src)->iface.cpy_tensor_to != NULL) {
        ggml_get_backend(src)->iface.cpy_tensor_to(ggml_get_backend(src)->context, src, dst);
    } else {
        // shouldn't be hit when copying from/to CPU
        #ifndef NDEBUG
        fprintf(stderr, "ggml_backend_tensor_copy: neither cpy_tensor_from nor cpy_tensor_to are implemented for backends %s and %s, falling back to get/set\n", ggml_backend_name(src->buffer->backend), ggml_backend_name(dst->buffer->backend));
        #endif
        size_t nbytes = ggml_nbytes(src);
        void * data = malloc(nbytes);
        ggml_backend_tensor_get(src, data, 0, nbytes);
        ggml_backend_tensor_set(dst, data, 0, nbytes);
        free(data);
    }
}

```

这段代码定义了一个名为 "ggml_backend_cpu_context" 的结构体，其中包含一个整型变量 "n_threads"，一个指向 void 类型指针的 "work_data" 成员，以及一个指定大小的 "work_size"。

然后，定义了一个名为 "ggml_backend_cpu_name" 的函数，用于根据给定的 "backend" 参数返回一个字符串表示 "CPU" 或 "GPU" 类型。

接着，定义了一个名为 "ggml_backend_cpu_free" 的函数，用于释放 "ggml_backend_cpu_context" 类型的数据，该数据在 "ggml_backend_cpu_context" 结构体中被引用。函数的实现包括获取当前 "backend" 对象的安全指针(如果有)，然后释放 "work_data" 和 "cpu_ctx" 成员的内存，并最终释放整个 "ggml_backend_cpu_context"。


```cpp
// backend CPU

struct ggml_backend_cpu_context {
    int n_threads;
    void * work_data;
    size_t work_size;
};

static const char * ggml_backend_cpu_name(ggml_backend_t backend) {
    return "CPU";

    UNUSED(backend);
}

static void ggml_backend_cpu_free(ggml_backend_t backend) {
    struct ggml_backend_cpu_context * cpu_ctx = (struct ggml_backend_cpu_context *)backend->context;
    free(cpu_ctx->work_data);
    free(cpu_ctx);
    free(backend);
}

```

这段代码定义了两个静态函数，名为：

1. `ggml_backend_cpu_buffer_get_base`，返回一个指向内存缓冲区内容的指针。
2. `ggml_backend_cpu_buffer_free_buffer`，释放一个指向内存缓冲区数据的指针。

同时，还定义了一个名为 `cpu_backend_buffer_i` 的结构体，其包含了上述两个函数需要使用的成员变量。

从代码的作用来看，这两个函数与内存缓冲区相关，可以在 `ggml_backend_buffer_t` 这个结构体中进行一些基本的读取、写入操作。


```cpp
static void * ggml_backend_cpu_buffer_get_base(ggml_backend_buffer_t buffer) {
    return (void *)buffer->context;
}

static void ggml_backend_cpu_buffer_free_buffer(ggml_backend_buffer_t buffer) {
    free(buffer->context);
    UNUSED(buffer);
}

static struct ggml_backend_buffer_i cpu_backend_buffer_i = {
    /* .free_buffer    = */ ggml_backend_cpu_buffer_free_buffer,
    /* .get_base       = */ ggml_backend_cpu_buffer_get_base,
    /* .get_alloc_size = */ NULL, // defaults to ggml_nbytes
    /* .init_tensor    = */ NULL, // no initialization required
    /* .free_tensor    = */ NULL, // no cleanup required
};

```

这段代码定义了一个名为 ggml_backend_buffer_i 的结构体，它表示 CPU 类别的缓冲区。该结构体定义了缓冲区的几个关键成员，包括 .free_buffer 是否被设置为 NULL，.get_base 是否设置为函数返回地址，.get_alloc_size 和 .init_tensor 是否被设置等。

接着，该结构体定义了一个名为 ggml_backend_cpu_alloc_buffer 的函数，它接受一个 .backend 参数和一个 size 参数。该函数首先执行必要的内存分配，然后返回新分配的缓冲区的 ggml_backend_buffer 实例。

该函数的实现比较简单，只是通过传入不同的 .backend 参数来设置其它的成员，这些成员定义了缓冲区的一些基本属性。然而，由于该函数没有进行充分的检查，因此可能会导致缓冲区使用不当或者在使用时出现问题。


```cpp
// for buffers from ptr, free is not called
static struct ggml_backend_buffer_i cpu_backend_buffer_i_from_ptr = {
    /* .free_buffer    = */ NULL, // ptr is not owned by the buffer, so it does not need to be freed
    /* .get_base       = */ ggml_backend_cpu_buffer_get_base,
    /* .get_alloc_size = */ NULL, // defaults to ggml_nbytes
    /* .init_tensor    = */ NULL,
    /* .free_tensor    = */ NULL,
};

static const size_t TENSOR_ALIGNMENT = 64; // should be enough for AVX 512

static ggml_backend_buffer_t ggml_backend_cpu_alloc_buffer(ggml_backend_t backend, size_t size) {
    size += TENSOR_ALIGNMENT;   // malloc may return an address that is not aligned
    void * data = malloc(size); // TODO: maybe use GGML_ALIGNED_MALLOC?

    GGML_ASSERT(data != NULL && "failed to allocate buffer");

    return ggml_backend_buffer_init(backend, cpu_backend_buffer_i, data, size);
}

```

这段代码定义了两个名为“ggml_backend_cpu_get_alignment”和“ggml_backend_cpu_set_tensor_async”的函数。它们都是ggml_backend_cpu_XXX函数。

“ggml_backend_cpu_get_alignment”函数返回一个整数类型的变量ggml_backend_cpu_alignment，它代表输入参数backend的CPU对齐策略。它返回的值在后面的说明中会给出。

“ggml_backend_cpu_set_tensor_async”函数是一个用户定义的函数，它接受一个整数类型的输入参数backend和一个结构体类型的参数tensor，以及一个指针类型的参数data和一个表示偏移量和尺寸的整数类型的参数offset和size。它的功能是读取一个Tensor并将其数据从内存中拷贝到backend的地址中指定的位置，并返回backend。

“ggml_backend_cpu_get_tensor_async”函数与“ggml_backend_cpu_set_tensor_async”函数功能类似，但是它的参数中包括一个const结构体类型的tensor，而不是普通的结构体类型。它的功能是读取一个Tensor并将其数据从内存中拷贝到backend的地址中指定的位置，并返回backend。


```cpp
static size_t ggml_backend_cpu_get_alignment(ggml_backend_t backend) {
    return TENSOR_ALIGNMENT;
    UNUSED(backend);
}

static void ggml_backend_cpu_set_tensor_async(ggml_backend_t backend, struct ggml_tensor * tensor, const void * data, size_t offset, size_t size) {
    GGML_ASSERT(offset + size <= ggml_nbytes(tensor) && "tensor write out of bounds");
    GGML_ASSERT(tensor->data != NULL && "tensor not allocated");

    memcpy((char *)tensor->data + offset, data, size);

    UNUSED(backend);
}

static void ggml_backend_cpu_get_tensor_async(ggml_backend_t backend, const struct ggml_tensor * tensor, void * data, size_t offset, size_t size) {
    GGML_ASSERT(offset + size <= ggml_nbytes(tensor) && "tensor read out of bounds");
    GGML_ASSERT(tensor->data != NULL && "tensor not allocated");

    memcpy(data, (const char *)tensor->data + offset, size);

    UNUSED(backend);
}

```

这段代码定义了两个名为"ggml_backend_cpu_synchronize"和"ggml_backend_cpu_cpy_tensor_from"的函数，以及两个名为"ggml_backend_cpu_cpy_tensor_to"的函数。它们都属于一个名为"ggml_backend_cpu"的函数家族。

"ggml_backend_cpu_synchronize"函数的作用是提供一个无参函数，但可以在函数内部使用内部状态。

"ggml_backend_cpu_cpy_tensor_from"函数的作用是将源 tensor 中的数据复制到目标 tensor 中，将源 tensor 包装为 void 类型，因此这个函数不会对输入数据进行修改。

"ggml_backend_cpu_cpy_tensor_to"函数的作用是将目标 tensor 中的数据复制到源 tensor 中，将源 tensor 包装为 void 类型，因此这个函数也不会对输入数据进行修改。

"ggml_backend_cpu"函数的作用是定义了这两个函数和它们的参数类型，但没有进行实现。


```cpp
static void ggml_backend_cpu_synchronize(ggml_backend_t backend) {
    UNUSED(backend);
}

static void ggml_backend_cpu_cpy_tensor_from(ggml_backend_t backend, struct ggml_tensor * src, struct ggml_tensor * dst) {
    ggml_backend_tensor_get(src, dst->data, 0, ggml_nbytes(src));

    UNUSED(backend);
}

static void ggml_backend_cpu_cpy_tensor_to(ggml_backend_t backend, struct ggml_tensor * src, struct ggml_tensor * dst) {
    ggml_backend_tensor_set(dst, src->data, 0, ggml_nbytes(src));

    UNUSED(backend);
}

```

这段代码定义了一个名为 "ggml_backend_plan_cpu" 的结构体，它表示一个计算图的后端计划。这个结构体包含两个成员：一个名为 "ggml_cplan" 的结构体，它包含了计算图的详细信息，包括线的编号、每个线所使用的CPU、每个线所需的内存等等；另一个名为 "ggml_cgraph" 的结构体，它包含了计算图的详细信息，包括图中的节点、边、每个节点的出度和每个边所需的带宽等等。

接着，定义了一个名为 "ggml_backend_graph_plan_create" 的函数，它接收两个参数：一个 "ggml_backend_t" 的后端实例和一个 "ggml_cgraph" 的 pointer。这个函数将返回一个后端实例，这个后端实例将作为 "ggml_backend_graph_plan_create" 函数的参数。

在函数内部，首先创建一个名为 "ggml_backend_plan_cpu" 的后端实例，这个实例包含了计算图的计划和图形表示。接着，定义了一个名为 "ggml_cplan" 的成员变量，这个成员变量使用了 "ggml_graph_plan" 函数来创建计算图的计划。然后，定义了一个名为 "ggml_cgraph" 的成员变量，这个成员变量使用了 "*cgraph" 解引用了一个 "ggml_cgraph" 变量。

接着，定义了一个名为 "ggml_backend_graph_plan_create" 的函数，它接收两个参数：一个 "ggml_backend_t" 的后端实例和一个 "ggml_cgraph" 的 pointer。这个函数将返回一个后端实例，这个后端实例将作为 "ggml_backend_graph_plan_create" 函数的参数。在函数内部，首先创建一个名为 "ggml_backend_plan_cpu" 的后端实例，这个实例包含了计算图的计划和图形表示。接着，定义了一个名为 "ggml_cplan" 的成员变量，这个成员变量使用了 "ggml_graph_plan" 函数来创建计算图的计划。然后，定义了一个名为 "ggml_cgraph" 的成员变量，这个成员变量使用了 "*cgraph" 解引用了一个 "ggml_cgraph" 变量。


```cpp
struct ggml_backend_plan_cpu {
    struct ggml_cplan cplan;
    struct ggml_cgraph cgraph;
};

static ggml_backend_graph_plan_t ggml_backend_cpu_graph_plan_create(ggml_backend_t backend, struct ggml_cgraph * cgraph) {
    struct ggml_backend_cpu_context * cpu_ctx = (struct ggml_backend_cpu_context *)backend->context;

    struct ggml_backend_plan_cpu * cpu_plan = malloc(sizeof(struct ggml_backend_plan_cpu));

    cpu_plan->cplan = ggml_graph_plan(cgraph, cpu_ctx->n_threads);
    cpu_plan->cgraph = *cgraph;

    if (cpu_plan->cplan.work_size > 0) {
        cpu_plan->cplan.work_data = malloc(cpu_plan->cplan.work_size);
    }

    return cpu_plan;
}

```



这两个函数定义在ggml_backend_cpu_graph_plan_free和ggml_backend_cpu_graph_plan_compute中，用于在ggml_backend_cpu_graph_plan_free函数中管理与graph plan相关的资源。

在ggml_backend_cpu_graph_plan_free函数中，首先通过从传入的plan结构中获取cpu plan，然后释放它所指向的内存。接着，通过解引用整型变量backend，将其与ggml_backend_cpu_graph_plan_free函数中的this指针进行比较，确保不会修改this引用的指针。

ggml_backend_cpu_graph_plan_compute函数与ggml_backend_cpu_graph_plan_free函数类似，但它的作用是执行计算，而不是释放资源。它与ggml_backend_cpu_graph_plan_free函数的主要区别是，前者返回计算结果，后者返回void类型的空指针。

这两个函数的具体实现没有提供任何用例，因此无法确定它们如何被使用。


```cpp
static void ggml_backend_cpu_graph_plan_free(ggml_backend_t backend, ggml_backend_graph_plan_t plan) {
    struct ggml_backend_plan_cpu * cpu_plan = (struct ggml_backend_plan_cpu *)plan;

    free(cpu_plan->cplan.work_data);
    free(cpu_plan);

    UNUSED(backend);
}

static void ggml_backend_cpu_graph_plan_compute(ggml_backend_t backend, ggml_backend_graph_plan_t plan) {
    struct ggml_backend_plan_cpu * cpu_plan = (struct ggml_backend_plan_cpu *)plan;

    ggml_graph_compute(&cpu_plan->cgraph, &cpu_plan->cplan);

    UNUSED(backend);
}

```

这段代码定义了一个名为 `ggml_backend_cpu_graph_compute` 的函数，属于 `ggml_backend_t` 类的成员函数。

该函数接收两个参数，一个是 `ggml_backend_t` 类型的后端实例(`backend`)，另一个是一个指向 `ggml_cgraph` 类型的结构体(`cgraph`)。

函数内部首先获取后端实例中的 `context` 成员，然后使用这个 `context` 来创建一个 `ggml_backend_cpu_context` 类型的结构体，存储在 `cpu_ctx` 中。

接着，函数使用 `ggml_graph_plan` 函数来计算要执行的图形计划，这个计划包括将 `cgraph` 中的顶点数乘以 `cpu_ctx` 中的 `n_threads` 参数，以便在 `cpu` 线程上执行。

如果计算得到的计划大小小于 `cpu_ctx` 中的 `work_size` 参数，那么函数将在不使用额外的内存复制的情况下，将 `cpu_ctx` 中的 `work_data` 成员复制到 `cgraph` 中的对应顶点数组中。此时，由于 `cpu_ctx` 中的 `work_size` 参数比 `cgraph` 中的顶点数要大，因此函数实际上会从 `cgraph` 中的顶点数中减去 `cpu_ctx` 中的 `work_size` 参数，以确保正确执行图形计划。

最后，函数调用 `ggml_graph_compute` 函数，将 `cgraph` 中的顶点数带入计算。


```cpp
static void ggml_backend_cpu_graph_compute(ggml_backend_t backend, struct ggml_cgraph * cgraph) {
    struct ggml_backend_cpu_context * cpu_ctx = (struct ggml_backend_cpu_context *)backend->context;

    struct ggml_cplan cplan = ggml_graph_plan(cgraph, cpu_ctx->n_threads);

    if (cpu_ctx->work_size < cplan.work_size) {
        // TODO: may be faster to free and use malloc to avoid the copy
        cpu_ctx->work_data = realloc(cpu_ctx->work_data, cplan.work_size);
        cpu_ctx->work_size = cplan.work_size;
    }

    cplan.work_data = cpu_ctx->work_data;

    ggml_graph_compute(cgraph, &cplan);
}

```

这段代码定义了一个名为ggml_backend_cpu_supports_op的函数，属于ggml_backend_t类型的函数。函数接收一个ggml_backend_t类型的后端和一个ggml_tensor类型的参数op，并返回一个布尔值表示当前CPU是否支持op。

函数体中包含了一些ggml_backend_cpu相关的成员函数和结构体，如ggml_backend_cpu_name、ggml_backend_cpu_free、ggml_backend_cpu_alloc_buffer、ggml_backend_cpu_get_alignment、ggml_backend_cpu_set_tensor_async、ggml_backend_cpu_get_tensor_async、ggml_backend_cpu_synchronize、ggml_backend_cpu_cpy_tensor_from和ggml_backend_cpu_cpy_tensor_to等。

ggml_backend_cpu_supports_op函数的作用是判断当前CPU是否支持传入的op，如果支持则返回true，否则返回false。这个函数在ggml_backend_cpu_client和ggml_backend_cpu_server等继承自ggml_backend_cpu_supports_op的函数中会被调用，从而影响ggml应用程序的性能。


```cpp
static bool ggml_backend_cpu_supports_op(ggml_backend_t backend, const struct ggml_tensor * op) {
    return true;
    UNUSED(backend);
    UNUSED(op);
}

static struct ggml_backend_i cpu_backend_i = {
    /* .get_name            = */ ggml_backend_cpu_name,
    /* .free                = */ ggml_backend_cpu_free,
    /* .alloc_buffer        = */ ggml_backend_cpu_alloc_buffer,
    /* .get_alignment       = */ ggml_backend_cpu_get_alignment,
    /* .set_tensor_async    = */ ggml_backend_cpu_set_tensor_async,
    /* .get_tensor_async    = */ ggml_backend_cpu_get_tensor_async,
    /* .synchronize         = */ ggml_backend_cpu_synchronize,
    /* .cpy_tensor_from     = */ ggml_backend_cpu_cpy_tensor_from,
    /* .cpy_tensor_to       = */ ggml_backend_cpu_cpy_tensor_to,
    /* .graph_plan_create   = */ ggml_backend_cpu_graph_plan_create,
    /* .graph_plan_free     = */ ggml_backend_cpu_graph_plan_free,
    /* .graph_plan_compute  = */ ggml_backend_cpu_graph_plan_compute,
    /* .graph_compute       = */ ggml_backend_cpu_graph_compute,
    /* .supports_op         = */ ggml_backend_cpu_supports_op,
};

```

这段代码定义了一个名为 ggml_backend_cpu_init 的函数，它用于初始化 CPU 平台的底层执行单元。以下是该函数的主要作用：

1. 创建一个指向结构体ggml_backend_cpu_context的指针变量ctx，该结构体定义了CPU执行单元的属性和状态。
2. 设置ctx中的n_threads成员为GGML_DEFAULT_N_THREADS，这是一个默认值，用于在不同CPU上发挥最大的并行度。
3. 设置ctx中的work_data成员为一个空指针，用于存储计算过程中的数据。
4. 设置ctx中的work_size成员为0，用于存储计算完成后的结果。
5. 创建一个指向结构体ggml_backend的指针变量cpu_backend，该结构体定义了CPU执行单元的底层实现。
6. 将ctx指针变量和cpu_backend指针变量进行解引用，将CPU执行单元的实现赋值给ctx。
7. 返回创建的cpu_backend，以便后续使用。

总体来说，该函数的目的是初始化CPU执行单元，以便在ggml_backend_cpu_init函数中正确使用。


```cpp
ggml_backend_t ggml_backend_cpu_init(void) {
    struct ggml_backend_cpu_context * ctx = malloc(sizeof(struct ggml_backend_cpu_context));

    ctx->n_threads = GGML_DEFAULT_N_THREADS;
    ctx->work_data = NULL;
    ctx->work_size = 0;

    ggml_backend_t cpu_backend = malloc(sizeof(struct ggml_backend));

    *cpu_backend = (struct ggml_backend) {
        /* .interface = */ cpu_backend_i,
        /* .context   = */ ctx
    };
    return cpu_backend;
}

```

这段代码定义了两个名为“ggml_backend_is_cpu()”和“ggml_backend_cpu_set_n_threads()”的函数，以及三个函数类型。

“ggml_backend_is_cpu()”函数用于判断给定的backend是否为CPU类型的backend。如果backend为CPU类型的backend，函数返回true，否则返回false。

“ggml_backend_cpu_set_n_threads()”函数用于设置CPU类型的backend的n_threads成员变量。函数需要一个backend参数和一个n_threads参数，其中backend参数必须是已经声明为“ggml_backend_t”的backend对象，而n_threads参数必须是整数类型的变量。

“ggml_backend_buffer_t”定义了一个名为“ggml_backend_cpu_buffer_from_ptr()”的函数，用于从ptr指向的内存区域中分配大小为size个字节的高性能缓冲区。函数需要两个backend参数和一个ptr参数和一个size参数，其中backend参数必须是已经声明为“ggml_backend_t”的backend对象，而ptr参数必须是void指针，Size参数必须是size_t类型。函数返回一个ggml_backend_buffer_t类型的缓冲区对象。

这段代码的目的是定义了一些函数类型，以及一些函数，用于实现高性能缓冲区的操作，以支持更多的并发操作。


```cpp
bool ggml_backend_is_cpu(ggml_backend_t backend) {
    return backend->iface.get_name == ggml_backend_cpu_name;
}

void ggml_backend_cpu_set_n_threads(ggml_backend_t backend_cpu, int n_threads) {
    GGML_ASSERT(ggml_backend_is_cpu(backend_cpu));

    struct ggml_backend_cpu_context * ctx = (struct ggml_backend_cpu_context *)backend_cpu->context;
    ctx->n_threads = n_threads;
}

ggml_backend_buffer_t ggml_backend_cpu_buffer_from_ptr(ggml_backend_t backend_cpu, void * ptr, size_t size) {
    return ggml_backend_buffer_init(backend_cpu, cpu_backend_buffer_i_from_ptr, ptr, size);
}

```

这段代码定义了一个名为 "ggml_backend_sched_split" 的结构体，表示在机器学习任务中执行分割操作时的内部数据结构。这个结构体包含一个指向 "ggml_tallocr_t" 类型的变量的 "tallocr"，用于指示当前正在执行的分割操作类型。

它还包含两个整型变量 "i_start" 和 "i_end"，用于指示所要分割的数据的起始和结束索引。接着，它定义了一个包含 "struct ggml_tensor * inputs[GGML_MAX_SPLIT_INPUTS]" 的数组，用于存储输入数据，每个输入数据都是一个包含 GGML 张量数据的 "struct ggml_tensor" 类型的指针。

它还定义了一个指向 "struct ggml_cgraph * graph" 的指针 "graph"，用于在分割操作期间在图上执行高效的计算操作。最后，它还包含一个名为 "n_inputs" 的整型变量，用于记录当前正在使用的输入数据的数量。


```cpp
// scheduler

#define GGML_MAX_BACKENDS 4
#define GGML_MAX_SPLITS 256
#define GGML_MAX_SPLIT_INPUTS 16

struct ggml_backend_sched_split {
    ggml_tallocr_t tallocr;
    int i_start;
    int i_end;
    struct ggml_tensor * inputs[GGML_MAX_SPLIT_INPUTS];
    int n_inputs;
    struct ggml_cgraph * graph;
};

```

这是一个定义了GGML驱动程序的struct类型的变量。

struct ggml_backend_sched定义了GGML驱动程序的大致结构和成员变量。它包括以下成员：

- 整型成员ggml_backends，表示当前正在使用的GGML后端的数量。
- 整型成员backends，是一个整型数组，用于存储GGML后端的具体实现。
- 整型成员ggml_tallocrs，表示当前正在使用的TTallOCR的实例数量。
- 整型成员ggml_max_backends，表示定义的最大GGML后端数量。
- 整型成员hash_set，表示一个HHashSet类型的数据结构，用于存储所有已分配的内存区域。
- 函数指针类型成员node_talloc，用于指定TTallOCR的实例，以便能够根据HHashSet中存储的内存区域分配TTallOCR实例。
- 函数指针类型成员node_copies，用于指定TTallOCR的实例，以便能够根据HHashSet中存储的内存区域复制TTallOCR实例。
- 结构体类型成员graph，用于存储整个GGML图的根节点。
- 结构体类型成员splits，用于存储所有的分裂。
- 整型成员n_splits，表示定义的分裂数量。
- 结构体类型成员ctx，用于存储当前的上下文。

此外，还定义了一些用于管理其他数据结构的成员变量，如内存对齐的指针和一些与HHashSet相关的成员函数。


```cpp
struct ggml_backend_sched {
    int n_backends;
    ggml_backend_t backends[GGML_MAX_BACKENDS];
    ggml_tallocr_t  tallocs[GGML_MAX_BACKENDS];

    ggml_gallocr_t galloc;

    struct ggml_hash_set    hash_set;
    ggml_tallocr_t *        node_talloc;                     // [hash_set.size]
    struct ggml_tensor * (* node_copies)[GGML_MAX_BACKENDS]; // [hash_set.size][GGML_MAX_BACKENDS]

    struct ggml_cgraph * graph;
    struct ggml_backend_sched_split splits[GGML_MAX_SPLITS];
    int n_splits;

    struct ggml_context * ctx;

    // align context_buffer to GGML_MEM_ALIGN
    #ifdef _MSC_VER
    __declspec(align(GGML_MEM_ALIGN))
    #else
    __attribute__((aligned(GGML_MEM_ALIGN)))
    #endif
    char context_buffer[GGML_MAX_SPLITS*GGML_MAX_SPLIT_INPUTS*sizeof(struct ggml_tensor) + GGML_MAX_SPLITS*sizeof(struct ggml_cgraph)];
};

```



This code defines two macros with two different preprocess directives.

The first macro defines a macro called `hash_id(node)`. It uses the `gggml_hash_find_or_insert` function to hash the `node` and returns the hash code. The hash code is then used by the `sched->node_talloc` function to allocate a node in the backend with the specified hash code.

The second macro defines a macro called `node_allocr`. It takes a `node` as an argument and returns the index of the allocated node in the backend.

The code also defines a function called `gggml_is_view_op(enum ggml_op op)`. It returns `true` if the `op` is any of the following: `GGML_OP_VIEW`, `GGML_OP_RESHAPE`, `GGML_OP_PERMUTE`, or `GGML_OP_TRANSPOSE`.

Finally, the code defines an integer function called `sched_backend_prio(ggml_backend_sched_t sched, ggml_backend_t backend)`. It returns the priority of the backend. If the `backend` is the same as the current backend, the function returns the index of that backend. Otherwise, the function returns the highest priority available.


```cpp
#define hash_id(node) ggml_hash_find_or_insert(sched->hash_set, node)
#define node_allocr(node) sched->node_talloc[hash_id(node)]

static bool ggml_is_view_op(enum ggml_op op) {
    return op == GGML_OP_VIEW || op == GGML_OP_RESHAPE || op == GGML_OP_PERMUTE || op == GGML_OP_TRANSPOSE;
}

// returns the priority of the backend, lower is better
static int sched_backend_prio(ggml_backend_sched_t sched, ggml_backend_t backend) {
    for (int i = 0; i < sched->n_backends; i++) {
        if (sched->backends[i] == backend) {
            return i;
        }
    }
    return INT_MAX;
}

```

This function appears to be part of a scheduler in the context of the GGML (Graphics Gospel Middleware Library). The function takes a node in the form of a `ggml_tensor` and a list of `ggml_tensor_view` and returns the backend of the tensor.

The function first checks if the tensor has already been allocated in a buffer and, if so, it assumes that it is critical to keep it there. Then it checks if the tensor has a specified backend and, if it does, it uses that backend. If not, it checks if the tensor has a valid backend, such as `NULL`, and returns it if it does.

The function then checks if the tensor has a view source, and, if it does, it checks the backend of that source. If the tensor has a view source and has a valid backend, the function returns that backend.

If the tensor has a view source and has no valid backend, the function iterates over all possible backends and returns the one with the highest priority. If there are multiple backends with the same priority, the function based the return on the `sched_backend_prio` function, which appears to be a function that determines the priority of backends based on the current scheduler.

If the function can't find a valid backend for the tensor, it will return `NULL`, indicating that the tensor needs to be allocated in memory.


```cpp
static int sched_allocr_prio(ggml_backend_sched_t sched, ggml_tallocr_t allocr) {
    for (int i = 0; i < sched->n_backends; i++) {
        if (sched->tallocs[i] == allocr) {
            return i;
        }
    }
    return INT_MAX;
}

// returns the backend that should be used for the node based on the current locations
char causes[GGML_DEFAULT_GRAPH_SIZE*4 + GGML_MAX_SPLITS*GGML_MAX_SPLIT_INPUTS][128]; // debug, remove
static ggml_backend_t sched_backend_from_cur(ggml_backend_sched_t sched, struct ggml_tensor * node) {
    // if the dst tensor is already allocated in a buffer, we must assume that it is critical to keep it there
    // ie. kv cache updates
    // note that this doesn't allow fallback to CPU. need to add output tensors to the splits to copy the data back to the original backend.
    // dst
    ggml_backend_t cur_backend = ggml_get_backend(node);
    if (cur_backend != NULL) {
        sprintf(causes[hash_id(node)], "1.dst");
        return cur_backend;
    }

    // view_src
    if (node->view_src != NULL && ggml_get_backend(node->view_src) != NULL) {
        sprintf(causes[hash_id(node)], "1.vsrc");
        return ggml_get_backend(node->view_src);
    }

    // src
    int cur_prio = INT_MAX;
    size_t cur_size = 0;

    for (int i = 0; i < GGML_MAX_SRC; i++) {
        const struct ggml_tensor * src = node->src[i];
        if (src == NULL) {
            break;
        }
        ggml_backend_t src_backend = ggml_get_backend(src);
        if (src_backend != NULL) {
            int src_prio = sched_backend_prio(sched, src_backend);
            size_t src_size = ggml_nbytes(src);
            if (src_prio < cur_prio && src_size >= cur_size) {
                cur_prio = src_prio;
                cur_size = src_size;
                cur_backend = src_backend;
                sprintf(causes[hash_id(node)], "1.src%d", i);
            }
        }
    }
    return cur_backend;
}

```

This is a C function that performs a splittable operation on a piece of a graph, represented by the splits vector. It splits the graph along a specified edge, and creates a new graph node for each split edge. The new nodes are each associated with an existing backend edge in the original graph, which provides the necessary context for the new node to be used. 

The function takes in an array of nodes representing the original graph, and an array of edge identifiers indicating which edge to split along. It then performs the split operation by遍历 the edge identifiers, creating new nodes for each edge, and setting the backend edge of each new node to the original backend edge associated with that edge. 

The function also supports the addition of new nodes at the specified edge, which are processed in the same way as new nodes created during the split operation.


```cpp
static char * fmt_size(size_t size) {
    static char buffer[128];
    if (size >= 1024*1024) {
        sprintf(buffer, "%zuM", size/1024/1024);
    } else {
        sprintf(buffer, "%zuK", size/1024);
    }
    return buffer;
}

static void sched_print_assignments(ggml_backend_sched_t sched, struct ggml_cgraph * graph) {
    int cur_split = 0;
    for (int i = 0; i < graph->n_nodes; i++) {
        if (cur_split < sched->n_splits && i == sched->splits[cur_split].i_start) {
            ggml_backend_t split_backend = ggml_tallocr_get_buffer(sched->splits[cur_split].tallocr)->backend;
            fprintf(stderr, "\n## SPLIT #%d: %s # %d inputs: ", cur_split, ggml_backend_name(split_backend), sched->splits[cur_split].n_inputs);
            for (int j = 0; j < sched->splits[cur_split].n_inputs; j++) {
                fprintf(stderr, "[%s (%5.5s)] ", sched->splits[cur_split].inputs[j]->name, fmt_size(ggml_nbytes(sched->splits[cur_split].inputs[j])));
            }
            fprintf(stderr, "\n");
            cur_split++;
        }
        struct ggml_tensor * node = graph->nodes[i];
        if (ggml_is_view_op(node->op)) {
            continue;
        }
        ggml_tallocr_t node_allocr = node_allocr(node);
        ggml_backend_t node_backend = node_allocr ? ggml_tallocr_get_buffer(node_allocr)->backend : NULL;
        fprintf(stderr, "node #%3d (%10.10s): %20.20s (%4.4s) [%4.4s %8.8s]:", i, ggml_op_name(node->op), node->name, fmt_size(ggml_nbytes(node)), node_allocr ? ggml_backend_name(node_backend) : "NULL", causes[hash_id(node)]);
        for (int j = 0; j < GGML_MAX_SRC; j++) {
            struct ggml_tensor * src = node->src[j];
            if (src == NULL) {
                break;
            }
            ggml_tallocr_t src_allocr = node_allocr(src);
            ggml_backend_t src_backend = src_allocr ? ggml_tallocr_get_buffer(src_allocr)->backend : NULL;
            fprintf(stderr, " %20.20s (%4.4s) [%4.4s %8.8s]", src->name, fmt_size(ggml_nbytes(src)), src_backend ? ggml_backend_name(src_backend) : "NULL", causes[hash_id(src)]);
        }
        fprintf(stderr, "\n");
    }
}

```

In the fourth assignment, you're modifying the priorities of the splitting edges to ensure that each source block has a unique priority. You're doing this by first finding all the sources that are not on the same backend, and then modifying the priorities of those sources to be higher than the priorities of their src blocks.

Here's the high-level logic for this assignment:

1. Find all sources that are not on the same backend.
2. For each source, find its corresponding priority in the original priority array.
3. Modify the priority array to have higher priorities for sources that are not on the same backend.
4. Update the scheduling table for each source to have a unique priority.
5. Repeat this process for all sources that are not on the same backend.
6. Calculate the output of the last modification and store it in the output array.

Note that this is a simplified description, and the actual implementation may have more complex logic and more helper functions.


```cpp
// creates a copy of the tensor with the same memory layout
static struct ggml_tensor * ggml_dup_tensor_layout(struct ggml_context * ctx, const struct ggml_tensor * tensor) {
    struct ggml_tensor * dup = ggml_dup_tensor(ctx, tensor);
    for (int i = 0; i < GGML_MAX_DIMS; i++) {
        dup->nb[i] = tensor->nb[i];
    }
    return dup;
}

// assigns backends to ops and splits the graph into subgraphs that can be computed on the same backend
// TODO: merge passes
static void sched_split_graph(ggml_backend_sched_t sched, struct ggml_cgraph * graph) {
    // reset state
    size_t hash_size = sched->hash_set.size;
    memset(sched->hash_set.keys, 0, sizeof(sched->hash_set.keys[0]) * hash_size);
    memset(sched->node_talloc,   0, sizeof(sched->node_talloc[0])   * hash_size);
    memset(sched->node_copies,   0, sizeof(sched->node_copies[0])   * hash_size);
    sched->n_splits = 0;

    struct ggml_init_params params = {
        /*.mem_size =   */ sizeof(sched->context_buffer),
        /*.mem_buffer = */ sched->context_buffer,
        /*.no_alloc =   */ true
    };

    if (sched->ctx != NULL) {
        ggml_free(sched->ctx);
    }

    sched->ctx = ggml_init(params);

    // pass 1: assign backends to ops with allocated inputs
    for (int i = 0; i < graph->n_leafs; i++) {
        struct ggml_tensor * leaf = graph->leafs[i];
        if (node_allocr(leaf) != NULL) {
            // do not overwrite user assignments
            continue;
        }
        ggml_backend_t leaf_backend = ggml_get_backend(leaf);
        if (leaf_backend == NULL && leaf->view_src != NULL) {
            leaf_backend = ggml_get_backend(leaf->view_src);
        }
        if (leaf_backend != NULL) {
            node_allocr(leaf) = ggml_backend_sched_get_tallocr(sched, leaf_backend);
        }
    }

    for (int i = 0; i < graph->n_nodes; i++) {
        struct ggml_tensor * node = graph->nodes[i];
        if (node_allocr(node) != NULL) {
            // do not overwrite user assignments
            continue;
        }
        ggml_backend_t node_backend = sched_backend_from_cur(sched, node);
        if (node_backend != NULL) {
            node_allocr(node) = ggml_backend_sched_get_tallocr(sched, node_backend);
        }
    }
    //printf("PASS 1 ASSIGNMENTS\n"); sched_print_assignments(sched, graph);

    // pass 2: assign backends to ops from current assignments
    // TODO:
    //  - reuse sched_backend_from_cur
    for (int i = 0; i < graph->n_nodes; i++) {
        struct ggml_tensor * node = graph->nodes[i];
        ggml_tallocr_t node_allocr = node_allocr(node);
        if (node_allocr == NULL) {
            int    cur_prio = INT_MAX;
            size_t cur_size = 0;
            for (int j = 0; j < GGML_MAX_SRC; j++) {
                struct ggml_tensor * src = node->src[j];
                if (src == NULL) {
                    break;
                }
                ggml_tallocr_t src_allocr = node_allocr(src);
                if (src_allocr != NULL) {
                    int    src_prio = sched_allocr_prio(sched, src_allocr);
                    size_t src_size = ggml_nbytes(src);
                    if (src_prio < cur_prio && src_size >= cur_size) {
                        cur_prio = src_prio;
                        cur_size = src_size;
                        node_allocr = src_allocr;
                        sprintf(causes[hash_id(node)], "2.src%d", j);
                    }
                }
            }
            if (node_allocr != NULL) {
                node_allocr(node) = node_allocr;
            }
        }
    }
    //printf("PASS 2 ASSIGNMENTS\n"); sched_print_assignments(sched, graph);

    // pass 3: assign backends to remaining src from dst (should only be leafs)
    for (int i = 0; i < graph->n_nodes; i++) {
        struct ggml_tensor * node = graph->nodes[i];
        ggml_tallocr_t node_allocr = node_allocr(node);
        for (int j = 0; j < GGML_MAX_SRC; j++) {
            struct ggml_tensor * src = node->src[j];
            if (src == NULL) {
                break;
            }
            ggml_tallocr_t src_allocr = node_allocr(src);
            if (src_allocr == NULL) {
                node_allocr(src) = node_allocr;
            }
        }
    }
    //printf("PASS 3 ASSIGNMENTS\n"); sched_print_assignments(sched, graph);

    // pass 4: split graph, find tensors that need to be copied
    // TODO:
    //  - when switching from a less preferred backend to a more preferred backend, check if it is possible to move the switch to an earlier point for the same cost
    // find first backend
    int cur_split = 0;
    for (int i = 0; i < graph->n_nodes; i++) {
        struct ggml_tensor * node = graph->nodes[i];
        if (node->view_src == NULL) {
            sched->splits[0].tallocr = node_allocr(node);
            break;
        }
    }
    sched->splits[0].i_start = 0;
    sched->splits[0].n_inputs = 0;
    memset(sched->splits[0].inputs, 0, sizeof(sched->splits[0].inputs)); //HACK
    ggml_tallocr_t cur_allocr = sched->splits[0].tallocr;
    size_t cur_backend_id = sched_allocr_prio(sched, cur_allocr);
    for (int i = 0; i < graph->n_nodes; i++) {
        struct ggml_tensor * node = graph->nodes[i];

        if (ggml_is_view_op(node->op)) {
            continue;
        }

        ggml_tallocr_t node_allocr = node_allocr(node);

        if (node_allocr != cur_allocr) {
            sched->splits[cur_split].i_end = i;
            cur_split++;
            GGML_ASSERT(cur_split < GGML_MAX_SPLITS);
            sched->splits[cur_split].tallocr = node_allocr;
            sched->splits[cur_split].i_start = i;
            sched->splits[cur_split].n_inputs = 0;
            memset(sched->splits[cur_split].inputs, 0, sizeof(sched->splits[cur_split].inputs)); //HACK
            cur_allocr = node_allocr;
            cur_backend_id = sched_allocr_prio(sched, cur_allocr);
        }

        // find inputs that are not on the same backend
        for (int j = 0; j < GGML_MAX_SRC; j++) {
            struct ggml_tensor * src = node->src[j];
            if (src == NULL) {
                break;
            }
            ggml_tallocr_t src_allocr = node_allocr(src);
            if (src_allocr != node_allocr) {
                int n_inputs = sched->splits[cur_split].n_inputs++;
                GGML_ASSERT(n_inputs < GGML_MAX_SPLIT_INPUTS);
                sched->splits[cur_split].inputs[n_inputs] = (struct ggml_tensor *)src;

                // create copies
                size_t id = hash_id(src);
                if (sched->node_copies[id][cur_backend_id] == NULL) {
                    struct ggml_tensor * tensor_copy = ggml_dup_tensor_layout(sched->ctx, src);
                    sched->node_copies[id][cur_backend_id] = tensor_copy;
                    node_allocr(tensor_copy) = cur_allocr;
                    ggml_backend_t backend = ggml_tallocr_get_buffer(cur_allocr)->backend;
                    ggml_format_name(tensor_copy, "%s#%s", ggml_backend_name(backend), src->name);
                }
                node->src[j] = sched->node_copies[id][cur_backend_id];
            }
        }
    }
    sched->splits[cur_split].i_end = graph->n_nodes;
    sched->n_splits = cur_split + 1;

    //fprintf(stderr, "PASS 4 ASSIGNMENTS\n"); sched_print_assignments(sched, graph); fflush(stdout);

```

这段代码的作用是检查一个名为"graph"的二维图结构中所有的节点，确保它们的后端相同。具体来说，代码首先检查每个节点的后端是否为空，如果是，则输出一条错误消息。然后，对于每个节点，代码会遍历它的源节点，如果当前节点是一个源节点，则代码会继续遍历其源节点。对于每个源节点，代码会再次遍历其所有后端，并检查它是否与当前节点的前端拥有相同的后端。如果当前节点的前端和后端都与当前节点的后端不匹配，则代码会输出一条错误消息。如果当前节点的前端和后端至少有一个匹配的，则代码不会输出错误消息，而是继续遍历当前节点的源节点。


```cpp
#if 1
    // sanity check: all sources should have the same backend as the node
    for (int i = 0; i < graph->n_nodes; i++) {
        struct ggml_tensor * node = graph->nodes[i];
        ggml_tallocr_t node_allocr = node_allocr(node);
        if (node_allocr == NULL) {
            fprintf(stderr, "!!!!!!! %s has no backend\n", node->name);
        }
        for (int j = 0; j < GGML_MAX_SRC; j++) {
            struct ggml_tensor * src = node->src[j];
            if (src == NULL) {
                break;
            }
            ggml_tallocr_t src_allocr = node_allocr(src);
            if (src_allocr != node_allocr /* && src_backend != NULL */) { // ignore nulls for now
                fprintf(stderr, "!!!! %s has backend %s, src %d (%s) has backend %s\n",
                    node->name, node_allocr ? ggml_backend_name(ggml_tallocr_get_buffer(node_allocr)->backend) : "NULL",
                    j, src->name, src_allocr ? ggml_backend_name(ggml_tallocr_get_buffer(src_allocr)->backend) : "NULL");
            }
        }
    }
```

这段代码的作用是创建每个分裂输出的图的副本，避免对原始图的多次复制，而是将分裂输出的信息传递给 `ggml_gallocr_alloc_graph_n` 函数。

具体来说，代码首先定义了一个名为 `graph_copy` 的结构体，其中包含原始图和每个分裂输出的图。然后，使用一个循环来遍历每个分裂输出，并为每个输出创建一个副本。

对于每个输出，首先调用 `ggml_graph_view` 函数从原始图获取分裂输出的信息，然后添加每个分裂输出的输入到新创建的图的输入中。为了保证每个输入只被复制一次，在新图中与旧图中的对应节点指针被复制，而不是将整个节点复制。

最后，将新创建的图设置为 `sched` 对象的 `graph` 成员，以便使用 `ggml_gallocr_alloc_graph_n` 函数。


```cpp
#endif

    // create copies of the graph for each split
    // FIXME: avoid this copy, pass split inputs to ggml_gallocr_alloc_graph_n in some other way
    struct ggml_cgraph * graph_copy = ggml_new_graph_custom(sched->ctx, graph->n_nodes + sched->n_splits*GGML_MAX_SPLIT_INPUTS, false);
    for (int i = 0; i < sched->n_splits; i++) {
        struct ggml_backend_sched_split * split = &sched->splits[i];
        split->graph = ggml_graph_view(sched->ctx, graph, split->i_start, split->i_end);

        // add inputs to the graph copy so that they are allocated by ggml-alloc at the start of the split
        for (int j = 0; j < split->n_inputs; j++) {
            struct ggml_tensor * input = split->inputs[j];
            struct ggml_tensor * input_cpy = sched->node_copies[hash_id(input)][sched_allocr_prio(sched, split->tallocr)];
            input_cpy->src[0] = input;
            graph_copy->nodes[graph_copy->n_nodes++] = input_cpy;
        }

        for (int j = split->i_start; j < split->i_end; j++) {
            graph_copy->nodes[graph_copy->n_nodes++] = graph->nodes[j];
        }
    }
    sched->graph = graph_copy;
}

```

It looks like this code is a C++ program that performs a backend promotion operation for a split node in a PyTorch scheduler. The code contains several functions, including `split_backend_id()`, `split_backend()`, `backend_prio()`, `sched_backend_prio()`, `copy_backend_tensor()`, `backend_tensor_view()`, `ggml_backend_buffer_init_tensor()`, `ggml_backend_tensor_copy()`, and `ggml_backend_synchronize()`.

The `split_backend_id()` function returns a unique identifier for the current split node. The `split_backend()` function returns a pointer to the current backend of the split node. The `backend_prio()` function returns a priority value for the current backend, with higher values indicating higher priority.

The `sched_backend_prio()` function is apriori, it should return the priority of the backend that the current scheduler is using. The `copy_backend_tensor()` function copies a tensor from the input backend to the output backend. It takes as input the tensor to copy and the backend to copy it to. It also takes as input the buffer that should be used for the tensor.

The `backend_tensor_view()` function creates a view of a tensor, it takes as input the buffer and the backend of the tensor.

The `ggml_backend_buffer_init_tensor()` function initializes a buffer for a tensor, it takes as input the buffer, the backend of the tensor, and the current scheduler.

The `ggml_backend_tensor_copy()` function copies a tensor from the input backend to the output backend, it takes as input the tensor to copy, the buffer, and the backend of the tensor.

The `ggml_backend_synchronize()` function synchronizes the backends, it takes as input the backend, and it should be called after all the tensors have been copied.


```cpp
static void sched_alloc_splits(ggml_backend_sched_t sched) {
    ggml_gallocr_alloc_graph_n(
        sched->galloc,
        sched->graph,
        sched->hash_set,
        sched->node_talloc);
}

static void sched_compute_splits(ggml_backend_sched_t sched) {
    uint64_t copy_us[GGML_MAX_BACKENDS] = {0};
    uint64_t compute_us[GGML_MAX_BACKENDS] = {0};

    struct ggml_backend_sched_split * splits = sched->splits;

    for (int i = 0; i < sched->n_splits; i++) {
        struct ggml_backend_sched_split * split = &splits[i];
        ggml_backend_t split_backend = ggml_tallocr_get_buffer(split->tallocr)->backend;
        int split_backend_id = sched_backend_prio(sched, split_backend);

        // copy the input tensors to the split backend
        uint64_t copy_start_us = ggml_time_us();
        for (int j = 0; j < split->n_inputs; j++) {
            struct ggml_tensor * input_cpy = sched->node_copies[hash_id(split->inputs[j])][sched_backend_prio(sched, split_backend)];
            if (split->inputs[j]->buffer == NULL) {
                if (split->inputs[j]->view_src == NULL) {
                    fprintf(stderr, "input %s has no buffer and no view_src\n", split->inputs[j]->name);
                    exit(1);
                }
                struct ggml_tensor * view = split->inputs[j];
                view->backend = view->view_src->backend;
                view->buffer  = view->view_src->buffer;
                view->data    = (char *)view->view_src->data + view->view_offs;
                ggml_backend_buffer_init_tensor(ggml_backend_sched_get_buffer(sched, view->buffer->backend), view);
            }
            if (input_cpy->buffer == NULL) {
                fprintf(stderr, "input_cpy %s has no buffer\n", input_cpy->name);
                exit(1);
            }
            GGML_ASSERT(split->inputs[j]->buffer->backend != input_cpy->buffer->backend);
            GGML_ASSERT(input_cpy->buffer->backend == split_backend);
            ggml_backend_tensor_copy(split->inputs[j], input_cpy);
        }
        // ggml_backend_synchronize(split_backend);
        int64_t copy_end_us = ggml_time_us();
        copy_us[split_backend_id] += copy_end_us - copy_start_us;

```

这段代码的作用是用于计算 splitting 算法中各个 backend的调度时间。该算法旨在将大文件分割成多个小文件，从而在单个节点的机器上运行计算任务。

具体来说，这段代码实现了以下功能：

1. 读取大文件中的注释信息，包括文件名和后端名称。
2. 如果后端为分布式，则调用指定后端实现的 graph_dump_dot 函数，将整个文件导出为 DOT 格式，并保存到指定的分片中。
3. 对于每个节点，记录其计算开始和结束时间，以及计算所需的时间。
4. 如果后端为分布式，则遍历所有的 backend，计算出每个后端需要的复制时间和计算时间（包括异步计算的时间）。
5. 将计算时间信息存储到指定后端的信息中，以便后续统计和分析。

代码中包含的一些宏：

* GGML_MAX_NAME：定义了允许的最大文件名长度，值为 32。
* "split_%i_%s.dot"：定义了生成split文件的格式，根据输入参数i和ggml_backend_name决定要生成的文件名。
* ggml_time_us：定义了计算时间的单位为微秒。
* sched：结构体，存储了整个计算任务的计划和调度信息。
* copy_us：整数数组，用于存储每个文件的复制时间。
* compute_us：整数数组，用于存储每个文件的计算时间。
* sched->n_splits：存储了要分割的文本文档数。


```cpp
#if 0
        char split_filename[GGML_MAX_NAME];
        snprintf(split_filename, GGML_MAX_NAME, "split_%i_%s.dot", i, ggml_backend_name(split_backend));
        ggml_graph_dump_dot(split->graph, NULL, split_filename);
#endif

        uint64_t compute_start_us = ggml_time_us();
        ggml_backend_graph_compute(split_backend, split->graph);
        // ggml_backend_synchronize(split_backend);
        uint64_t compute_end_us = ggml_time_us();
        compute_us[split_backend_id] += compute_end_us - compute_start_us;
    }

#if 0
    // per-backend timings
    fprintf(stderr, "sched_compute_splits times (%d splits):\n", sched->n_splits);
    for (int i = 0; i < sched->n_backends; i++) {
        if (copy_us[i] > 0 || compute_us[i] > 0) {
            fprintf(stderr, "\t%5.5s: %lu us copy, %lu us compute\n", ggml_backend_name(sched->backends[i]), copy_us[i], compute_us[i]);
        }
    }
```

这段代码定义了一个名为“sched”的结构体，其中包含一个名为“n_backends”的整型成员和一个名为“backends”的数组，以及一个名为“tallocs”的数组，用于存储每个backend的内存空间。

此外，还定义了一个名为“sched_reset”的函数，该函数重置sched中所有backend的内存空间。

另外，还定义了一个名为“ggml_backend_sched_new”的函数，该函数接收一个backends数组和backends数组长度作为参数，然后创建一个新的ggml_backend_sched结构体，并将sched中所有backend的内存空间初始化，最后将该结构体返回。

最后，还定义了一个函数“fprintf”，用于将字符串打印到标准错误中，并使用了“GGML_MAX_BACKENDS”作为死限制。


```cpp
#endif
}

static void sched_reset(ggml_backend_sched_t sched) {
    for (int i = 0; i < sched->n_backends; i++) {
        ggml_tallocr_reset(sched->tallocs[i]);
    }
}

ggml_backend_sched_t ggml_backend_sched_new(ggml_backend_t * backends, int n_backends) {
    GGML_ASSERT(n_backends <= GGML_MAX_BACKENDS);

    struct ggml_backend_sched * sched = malloc(sizeof(struct ggml_backend_sched));
    memset(sched, 0, sizeof(struct ggml_backend_sched));

    fprintf(stderr, "ggml_backend_sched size: %lu KB\n", sizeof(struct ggml_backend_sched)/1024);

    sched->n_backends = n_backends;
    for (int i = 0; i < n_backends; i++) {
        sched->backends[i] = backends[i];
    }

    sched->galloc = ggml_gallocr_new();

    // init measure allocs for each backend
    for (int i = 0; i < n_backends; i++) {
        sched->tallocs[i] = ggml_tallocr_new_measure_from_backend(backends[i]);
    }

    return sched;
}

```

This code appears to be a Java implementation of a backend scheduler for the GGML (Graph-Generative-Measurement-Based-Levenshtein-Distance) algorithm. It creates a scheduler for splitting the dataset of measures, allocating space for each measure, and re-splitting the data when it runs out of space.

The scheduler has a number of variables that keep track of the current state of the scheduler, such as the hash table for storing the keys of the hash table, the memory分配 for each measure, and the number of backends that are currently running. It also has a number of functions that perform the actual work of the scheduler, such as splitting the data, allocating space for measures, and re-splitting the data.

It is worth noting that this code is just one possible implementation and may not be optimized or work correctly without additional modifications.


```cpp
void ggml_backend_sched_free(ggml_backend_sched_t sched) {
    if (sched == NULL) {
        return;
    }
    for (int i = 0; i < sched->n_backends; i++) {
        ggml_tallocr_free(sched->tallocs[i]);
    }
    ggml_gallocr_free(sched->galloc);
    free(sched->hash_set.keys);
    free(sched->node_talloc);
    free(sched->node_copies);
    free(sched);
}

void ggml_backend_sched_init_measure(ggml_backend_sched_t sched, struct ggml_cgraph * measure_graph) {
    // initialize hash tables
    size_t hash_size = measure_graph->visited_hash_table.size + GGML_MAX_SPLITS*GGML_MAX_SPLIT_INPUTS;
    sched->hash_set.size = hash_size;
    sched->hash_set.keys = malloc(sizeof(sched->hash_set.keys[0]) * hash_size);
    sched->node_talloc   = malloc(sizeof(sched->node_talloc[0])   * hash_size);
    sched->node_copies   = malloc(sizeof(sched->node_copies[0])   * hash_size);

    sched_split_graph(sched, measure_graph);
    sched_alloc_splits(sched);

    // allocate buffers and reset allocators
    for (int i = 0; i < sched->n_backends; i++) {
        size_t size = ggml_tallocr_max_size(sched->tallocs[i]);
        ggml_tallocr_free(sched->tallocs[i]);
        sched->tallocs[i] = ggml_tallocr_new_from_backend(sched->backends[i], size);
    }

    sched_reset(sched);
}

```



这段代码是一个C++函数，属于GGML（GraphicsMagick）库的后台服务器扩展（backend）函数。它的作用是计算和调度一个图的分割。以下是函数的详细解释：

1. `ggml_backend_sched_graph_compute()`：

该函数接收两个参数，一个是`ggml_backend_sched_t`类型的调度对象，另一个是一个`struct ggml_cgraph`类型的图结构体。函数首先检查调度对象和图结构体中的哈希表大小是否符合一定的条件，然后执行以下操作：

- `sched_split_graph(sched, graph)`：将图结构体分割成多个小图，有助于在计算过程中减少内存开销。
- `sched_alloc_splits(sched)`：为图结构体分配适当的 Split 数，有助于在计算过程中优化性能。
- `sched_compute_splits(sched)`：对图结构体中的 Split 进行计算，有助于优化性能。
- `sched_reset(sched)`：重置调度对象，使它能够重新开始计算。

2. `ggml_tallocr_t ggml_backend_sched_get_tallocr(ggml_backend_sched_t sched, ggml_backend_t backend)`：

该函数接收两个参数，一个是`ggml_backend_sched_t`类型的调度对象，另一个是`ggml_backend_t`类型的返回类型。函数首先从调度对象中获取指定的后端，然后返回其对应的 TALLOC 数组首地址。

3. `ggml_backend_buffer_t ggml_backend_sched_get_buffer(ggml_backend_sched_t sched, ggml_backend_t backend)`：

该函数与上面同名的第二个函数，但返回类型为`ggml_backend_buffer_t`类型。它接收两个参数，一个是`ggml_backend_sched_t`类型的调度对象，另一个是`ggml_backend_t`类型的返回类型。函数从调度对象中获取指定的后端，然后返回对应的 TALLOC 数组首地址。


```cpp
void ggml_backend_sched_graph_compute(ggml_backend_sched_t sched, struct ggml_cgraph * graph) {
    GGML_ASSERT(sched->hash_set.size >= graph->visited_hash_table.size + GGML_MAX_SPLITS*GGML_MAX_SPLIT_INPUTS);

    sched_split_graph(sched, graph);
    sched_alloc_splits(sched);
    sched_compute_splits(sched);
    sched_reset(sched);
}

ggml_tallocr_t ggml_backend_sched_get_tallocr(ggml_backend_sched_t sched, ggml_backend_t backend) {
    int backend_index = sched_backend_prio(sched, backend);
    return sched->tallocs[backend_index];
}

ggml_backend_buffer_t ggml_backend_sched_get_buffer(ggml_backend_sched_t sched, ggml_backend_t backend) {
    int backend_index = sched_backend_prio(sched, backend);
    return ggml_tallocr_get_buffer(sched->tallocs[backend_index]);
}

```

这段代码定义了一个名为 `ggml_backend_sched_set_node_backend` 的函数，属于GGML Backend Scheduler 的内部函数。

该函数的主要作用是设置一个后端为给定 `ggml_tensor` 类型数据结构的 `node` 数据的后端。具体实现步骤如下：

1. 从 `sched` 参数中获取当前调度器实例，以及要设置的后端 `backend` 的索引。
2. 通过索引查找 `sched` 实例中与给定 `backend` 相关的后端索引，如果找不到，则返回-1。
3. 将查找到的索引值存储在 `sched->tallocs` 数组中，以便在分配内存时使用。
4. 将 `node` 数据的后端设置为给定的后端。

设置好后端后，后续在 `sched` 调度器中调用 `ggml_backend_sched_set_node_backend` 函数时，就可以直接使用传入的 `sched` 参数和 `node` 数据，而不需要关心数据的后端是什么。


```cpp
void ggml_backend_sched_set_node_backend(ggml_backend_sched_t sched, struct ggml_tensor * node, ggml_backend_t backend) {
    int backend_index = sched_backend_prio(sched, backend);
    GGML_ASSERT(backend_index >= 0 && backend_index < sched->n_backends);
    node_allocr(node) = sched->tallocs[backend_index];
}

```