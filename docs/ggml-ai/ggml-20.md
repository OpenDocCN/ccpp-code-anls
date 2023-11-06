# GGML源码解析 20

# `include/ggml/ggml.h`

这段代码是一个共享库头文件，它被声明为“Once”预处理指令，这意味着只有在程序被编译时才会生成最终的可执行文件。

该库的说明文件仍在编写中。但是，如果你想让某些特定的主题受到关注，你可以在注释中添加评论。

该库的目的是实现以下几个操作：

- 矩阵加法
- 矩阵乘法
- 矩阵减法
- 矩阵乘法
- 矩阵转置
- 获取最小值
- 获取最大值
- 获取平均值
- 获取标准差
- 获取协方差
- 获取期望
- 获取方差
- 生成正态分布随机数

由于该库仍在开发中，因此可能存在未经过测试或不完整的代码。在使用此库之前，请仔细阅读文档，并注意可能的警告和错误。


```cpp
#pragma once

//
// GGML Tensor Library
//
// This documentation is still a work in progress.
// If you wish some specific topics to be covered, feel free to drop a comment:
//
//   https://github.com/ggerganov/whisper.cpp/issues/40
//
// ## Overview
//
// This library implements:
//
//  - a set of tensor operations
```

这段代码是一个自动求导库，用于解决各种机器学习任务。它可以定义一个函数，该函数通过使用可用的张量操作来表示。每个张量操作在函数定义中对应于计算图中的一个节点。定义了函数后，用户可以选择计算其值并/或其梯度，还可以选择使用可用的优化算法来优化函数。


```cpp
//  - automatic differentiation
//  - basic optimization algorithms
//
// The aim of this library is to provide a minimalistic approach for various machine learning tasks. This includes,
// but is not limited to, the following:
//
//  - linear regression
//  - support vector machines
//  - neural networks
//
// The library allows the user to define a certain function using the available tensor operations. This function
// definition is represented internally via a computation graph. Each tensor operation in the function definition
// corresponds to a node in the graph. Having the computation graph defined, the user can choose to compute the
// function's value and/or its gradient with respect to the input variables. Optionally, the function can be optimized
// using one of the available optimization algorithms.
```

这段代码定义了一个名为 "ggml_init_params" 的结构体，其中包含两个成员变量，一个是内存大小（16 * 1024 * 1024），另一个是内存缓冲区。接着，代码使用 "ggml_init" 函数分配了足够的内存并返回了一个指向 "ggml_context" 类型的指针。然后，使用 "ggml_new_tensor_1d" 函数创建了一个名为 "x" 的 "ggml_tensor"，并将参数 "ctx" 和类型设为 "GGML_TYPE_F32" 和 "1"。最后，将 "x" 的值设为输入参数。


```cpp
//
// For example, here we define the function: f(x) = a*x^2 + b
//
//   {
//       struct ggml_init_params params = {
//           .mem_size   = 16*1024*1024,
//           .mem_buffer = NULL,
//       };
//
//       // memory allocation happens here
//       struct ggml_context * ctx = ggml_init(params);
//
//       struct ggml_tensor * x = ggml_new_tensor_1d(ctx, GGML_TYPE_F32, 1);
//
//       ggml_set_param(ctx, x); // x is an input variable
```

这段代码定义了一个名为 "ggml_tensor" 的结构体，它是一个 1 维的 double 型数组。这个数组在函数内部的作用是作为输入参数传递给一个名为 "ggml_tensor_mul" 的函数，这个函数将输入的两个 double 型数组相乘并存储到输出数组中。

进一步地，这段代码定义了一个名为 "ggml_tensor_add" 的函数，它将一个 double 型数组和一个 double 型数组相加并存储到输出数组中。

最后，没有进行实际的计算操作，因此，这段代码只是在定义一个结构体和函数，而不会进行实际的计算操作。


```cpp
//
//       struct ggml_tensor * a  = ggml_new_tensor_1d(ctx, GGML_TYPE_F32, 1);
//       struct ggml_tensor * b  = ggml_new_tensor_1d(ctx, GGML_TYPE_F32, 1);
//       struct ggml_tensor * x2 = ggml_mul(ctx, x, x);
//       struct ggml_tensor * f  = ggml_add(ctx, ggml_mul(ctx, a, x2), b);
//
//       ...
//   }
//
// Notice that the function definition above does not involve any actual computation. The computation is performed only
// when the user explicitly requests it. For example, to compute the function's value at x = 2.0:
//
//   {
//       ...
//
```

这段代码定义了一个名为 "ggml_cgraph" 的结构体，该结构体包含一个指向 "ggml_new_graph" 函数的指针。然后，使用 "ggml_build_forward_expand" 函数扩大了图的前向。

接下来，使用 "ggml_set_f32" 函数设置输入变量的初始值，然后使用 "ggml_graph_compute_with_ctx" 函数计算图的值并存储在变量 "f" 中。最后，输出 "f" 的值。

在这段代码中，使用了 C++ 的 forward-scaling 技术，通过随机游走产生一个 2008 个顶点的图，然后输出该图的边长。


```cpp
//       struct ggml_cgraph * gf = ggml_new_graph(ctx);
//       ggml_build_forward_expand(gf, f);
//
//       // set the input variable and parameter values
//       ggml_set_f32(x, 2.0f);
//       ggml_set_f32(a, 3.0f);
//       ggml_set_f32(b, 4.0f);
//
//       ggml_graph_compute_with_ctx(ctx, &gf, n_threads);
//
//       printf("f = %f\n", ggml_get_f32_1d(f, 0));
//
//       ...
//   }
//
```

这段代码定义了一个名为 "ggml_graph_compute" 的函数，它表示实际的计算操作是在这个函数中进行的。接下来定义了一系列名为 "ggml_new_tensor_..." 的函数，它们用于创建新的张量。这些函数会在内存缓冲区中分配新张量的内存，这个内存缓冲区是由 "ggml_init" 函数提供的。函数内部需要注意不要超过内存缓冲区的大小。

另外，函数 "ggml_set_param" 标定了一个张量作为一个输入变量，这个张量会被自动求导和优化算法使用。

整段代码定义了一个计算图功能，允许定义一个函数graph，然后使用这个函数graph计算函数graph的前向或后向计算。所有的计算都在 "ggml_graph_compute" 函数中进行，它使用和 "ggml_init" 函数提供的内存缓冲区中的内存进行计算。


```cpp
// The actual computation is performed in the ggml_graph_compute() function.
//
// The ggml_new_tensor_...() functions create new tensors. They are allocated in the memory buffer provided to the
// ggml_init() function. You have to be careful not to exceed the memory buffer size. Therefore, you have to know
// in advance how much memory you need for your computation. Alternatively, you can allocate a large enough memory
// and after defining the computation graph, call the ggml_used_mem() function to find out how much memory was
// actually needed.
//
// The ggml_set_param() function marks a tensor as an input variable. This is used by the automatic
// differentiation and optimization algorithms.
//
// The described approach allows to define the function graph once and then compute its forward or backward graphs
// multiple times. All computations will use the same memory buffer allocated in the ggml_init() function. This way
// the user can avoid the memory allocation overhead at runtime.
//
```

这段代码是一个用于支持多维张量的库，其中包括4维及以下。FP16和FP32数据类型是第一类公民，理论上还可以扩展支持FP8和整数数据类型。

每个张量操作都会生成一个新的张量。最初，库仅支持单例和二进制操作。随着时间推移，逐渐意识到库需要支持更复杂操作。目前尚不清楚如何支持这些复杂操作，但以下操作提供了一些示例：

* ggml_permute()
* ggml_conv_1d_1s()
* ggml_conv_1d_2s()

对于每个张量操作，库都实现了正向和反向计算函数。正向函数根据输入张量值计算输出张量值。反向函数根据输出张量值的伴随计算伴随输入张量值的计算。


```cpp
// The library supports multi-dimensional tensors - up to 4 dimensions. The FP16 and FP32 data types are first class
// citizens, but in theory the library can be extended to support FP8 and integer data types.
//
// Each tensor operation produces a new tensor. Initially the library was envisioned to support only the use of unary
// and binary operations. Most of the available operations fall into one of these two categories. With time, it became
// clear that the library needs to support more complex operations. The way to support these operations is not clear
// yet, but a few examples are demonstrated in the following operations:
//
//   - ggml_permute()
//   - ggml_conv_1d_1s()
//   - ggml_conv_1d_2s()
//
// For each tensor operator, the library implements a forward and backward computation function. The forward function
// computes the output tensor value given the input tensor values. The backward function computes the adjoint of the
// input tensors given the adjoint of the output tensor. For a detailed explanation of what this means, take a
```

这段代码定义了一个名为 "Automatic Differentiation" 的计算器类，该类可以用来计算多变量函数的梯度。

该类包含一个名为 "ggml_tensor" 的结构体，用于存储张量数据。该结构体包含张量的尺寸、数据类型以及内存缓冲区位置等信息，同时包含指向 "source" 张量数据的指针。

在 "Automatic Differentiation" 类的构造函数中，通过将两个张量 "a" 和 "b" 相加，并将结果存储在 "ggml_tensor" 结构体的 "c" 指针中，从而创建了一个新的张量。该张量可以用来计算多变量函数的梯度。


```cpp
// calculus class, or watch the following video:
//
//   What is Automatic Differentiation?
//   https://www.youtube.com/watch?v=wG_nF1awSSY
//
//
// ## Tensor data (struct ggml_tensor)
//
// The tensors are stored in memory via the ggml_tensor struct. The structure provides information about the size of
// the tensor, the data type, and the memory buffer where the tensor data is stored. Additionally, it contains
// pointers to the "source" tensors - i.e. the tensors that were used to compute the current tensor. For example:
//
//   {
//       struct ggml_tensor * c = ggml_add(ctx, a, b);
//
```

这段代码是一个C语言程序，它定义了一个名为"multi_tensor"的多维张量结构。这个结构包含一个二维数组的元素，以及一个表示该张量几维的整数。

首先，程序检查两个二维数组的元素是否与传入的二维数组a和b相等。如果两个二维数组与a和b相等，则程序验证张量是否在内存中连续，如果不是，则可以对张量进行操作。

程序的主要目的是定义一个多维数组结构，该结构可以存储非连续的数组元素。通过这个多维数组结构，用户可以轻松地执行各种任务，如 transpose（翻转）和 permutation（排列）。因为所有的张量操作都必须考虑stride（步长）变量，所以这个结构可以在不假设数组内存连续的情况下存储数组。


```cpp
//       assert(c->src[0] == a);
//       assert(c->src[1] == b);
//   }
//
// The multi-dimensional tensors are stored in row-major order. The ggml_tensor struct contains fields for the
// number of elements in each dimension ("ne") as well as the number of bytes ("nb", a.k.a. stride). This allows
// to store tensors that are not contiguous in memory, which is useful for operations such as transposition and
// permutation. All tensor operations have to take the stride into account and not assume that the tensor is
// contiguous in memory.
//
// The data of the tensor is accessed via the "data" pointer. For example:
//
//   {
//       const int nx = 2;
//       const int ny = 3;
```

这段代码定义了一个二维张量结构体变量 a，类型为GGML_TYPE_F32，具有 nx 和 ny 行，以及 nx 和 ny 列。

接下来，使用 for 循环遍历二维张量 a 的每个元素，将其第二个元素（float 类型）所包含的值乘以 x 和 y，并将结果存储到 a 的第二个元素。

该代码没有定义任何函数或使用 ggml_new_tensor_2d() 或其他辅助函数。它定义了一个二维张量并对其中的元素进行乘法运算，这可能是为了在后续的机器学习任务中进行数据预处理，例如将输入数据中的每个元素乘以一个权重矩阵。


```cpp
//
//       struct ggml_tensor * a = ggml_new_tensor_2d(ctx, GGML_TYPE_F32, nx, ny);
//
//       for (int y = 0; y < ny; y++) {
//           for (int x = 0; x < nx; x++) {
//               *(float *) ((char *) a->data + y*a->nb[1] + x*a->nb[0]) = x + y;
//           }
//       }
//
//       ...
//   }
//
// Alternatively, there are helper functions, such as ggml_get_f32_1d() and ggml_set_f32_1d() that can be used.
//
// ## The matrix multiplication operator (ggml_mul_mat)
```

这是一段使用Go（或Golang）编写的TODO（待办事项）提醒代码，用于在任务开始时提醒用户进行。该代码将以下内容输出到控制台：

```cpp
// TODO
//
//
// ## Multi-threading
//
// TODO
//
// ## Overview of ggml.c
//
// TODO
//
//
```

具体来说，这段代码的作用是提供一个简单的TODO列表，并在开始执行任务时提醒用户。它包含了一些使用Go内置的特性，如 multi-threading（多线程处理）和 SIMD（向量中断）优化。


```cpp
//
// TODO
//
//
// ## Multi-threading
//
// TODO
//
//
// ## Overview of ggml.c
//
// TODO
//
//
// ## SIMD optimizations
```

这段代码是一个C/C++语言的预处理指令，它会在编译时将一些定义或预处理指令编译到头文件中。它的目的是帮助开发人员在编写代码时更加方便和高效。

具体来说，它包括以下几个部分：

1. // TODO: 这个部分注释了一些 TODO 类型的提示，意味着未来某个时候需要实现的内容。

2. //: 这个部分定义了一些屏幕函数，包括 Debugging ggml。屏幕函数是 C++11 中引入的，用于在开发过程中调试程序。

3. #ifdef GGML_SHARED: 这个部分是一个条件编译语句，用于判断当前是否使用了共同的 GGML 定义。如果是，那么它会执行后面定义的代码。这里使用了 _WIN32 和 __MINGW32__ 这两个预处理指令，分别用于在 Windows 和 Visual C++ 环境下编译 64 位代码。

4. #define GGML_API __declspec(dllexport)

5. //: 这个部分定义了一个屏幕函数 Debugging ggml，但仅在 GGML_SHARED 条件成立时编译。

6. #ifdef GGML_BUILD: 这个部分是一个条件编译语句，用于判断当前是否使用了共同的 GGML 定义。如果是，那么它会执行后面定义的代码。

7. #define TODO TODO: 这个部分注释了一些 TODO 类型的提示，意味着未来某个时候需要实现的内容。

由于这段代码定义了屏幕函数 Debugging ggml，因此它将仅在当前使用了 GGML 定义的情况下编译。这意味着，如果其他源代码文件或库中定义了相同的屏幕函数，即使当前源代码文件中没有定义该函数，该函数也会在编译时成功定义并使用。


```cpp
//
// TODO
//
//
// ## Debugging ggml
//
// TODO
//
//

#ifdef GGML_SHARED
#    if defined(_WIN32) && !defined(__MINGW32__)
#        ifdef GGML_BUILD
#            define GGML_API __declspec(dllexport)
#        else
```

这段代码定义了一个名为 GGML_API 的头文件，并包含了一系列条件判断，用于将特定形式的函数声明与特定的库或头文件链接。

具体来说，代码首先定义了一个名为 GGML_API 的定义，使用的是 dllimport 修饰符。如果这个定义在源文件中没有被定义过，那么就会从库文件中进口它。

接着，代码定义了一系列使用 GGML_API 修饰符的函数定义。这些修饰符告诉编译器这些函数是来自 GGML 库的，需要使用特定的库函数实现。如果这些函数不在库中，那么编译器就会报错。

如果机器上下文中包含了 Clang，那么代码还会支持 Clang 编译器。在这种情况下，定义中的 `__attribute__((visibility ("default")))` 将不再需要，编译器会忽略它。

最后，如果机器上下文中既不包含 Clang，也不包含任何第三方库，那么 `#define GGML_API` 就是这段代码的最终定义，表示这段代码将不再需要链接任何库文件，也不需要使用任何特定的库函数实现。


```cpp
#            define GGML_API __declspec(dllimport)
#        endif
#    else
#        define GGML_API __attribute__ ((visibility ("default")))
#    endif
#else
#    define GGML_API
#endif

// TODO: support for clang
#ifdef __GNUC__
#    define GGML_DEPRECATED(func, hint) func __attribute__((deprecated(hint)))
#elif defined(_MSC_VER)
#    define GGML_DEPRECATED(func, hint) __declspec(deprecated(hint)) func
#else
```

这段代码定义了一个名为 GGML_DEPRECATED 的函数，它的参数为 func 和 hint，函数内部没有定义任何具体的实现。

接下来的两行代码定义了一些宏，用于在源文件被编译时检查不同的编译器是否支持 GGML 格式化。如果编译器支持，这些宏会定义为非空字符串，否则会定义为空字符串。

在 source 文件中，下面两行定义了一个名为 __attribute__((format(gnu_printf, __VA_ARGS__))) 的格式化字符串，它的作用是在编译时将 gnu_printf 函数的格式输出，它需要一个可变参数列表，其中第一个参数是一个格式化字符串，第二个参数是一个可变参数列表，格式化字符串的格式是通过 format 函数的第一个参数来确定的。这个可变参数列表包含两个元素，第一个元素是一个格式化字符串，第二个元素是一个指向整数类型的指针，这个指针存储格式化字符串中占字符数组位置的值。

接下来的两行定义了一个名为 __attribute__((format(printf, __VA_ARGS__))) 的格式化字符串，它的作用是在编译时将 printf 函数的格式输出，它需要一个可变参数列表，其中第一个参数是一个格式化字符串，第二个参数是一个可变参数列表，格式化字符串的格式是通过 printf 函数的第一个参数来确定的。这个可变参数列表包含两个元素，第一个元素是一个格式化字符串，第二个元素是一个指向整数类型的指针，这个指针存储格式化字符串中占字符数组位置的值。

最后两行代码定义了一个名为 GGML_ATTRIBUTE_FORMAT(...) 的函数，它的作用是在编译时根据不同的编译器来格式化输出源文件的内容，如果编译器支持 GGML 格式化，这个函数会被用于编译输出，否则会被用于编译输出空字符串。


```cpp
#    define GGML_DEPRECATED(func, hint) func
#endif

#ifndef __GNUC__
#    define GGML_ATTRIBUTE_FORMAT(...)
#elif defined(__MINGW32__)
#    define GGML_ATTRIBUTE_FORMAT(...) __attribute__((format(gnu_printf, __VA_ARGS__)))
#else
#    define GGML_ATTRIBUTE_FORMAT(...) __attribute__((format(printf, __VA_ARGS__)))
#endif

#include <stdint.h>
#include <stddef.h>
#include <stdbool.h>

```

这段代码定义了一些头文件，其中包括：GGML_FILE_MAGIC、GGML_FILE_VERSION、GGML_QNT_VERSION、GGML_QNT_VERSION_FACTOR、GGML_MAX_DIMS、GGML_MAX_PARAMS、GGML_MAX_CONTEXTS、GGML_MAX_SRC、GGML_MAX_NAME、GGML_MAX_OP_PARAMS、GGML_DEFAULT_N_THREADS、GGML_DEFAULT_GRAPH_SIZE 和GGML_MEM_ALIGN。GGML_FILE_MAGIC 和GGML_FILE_VERSION 是 defines，定义了文件头。GGML_QNT_VERSION、GGML_QNT_VERSION_FACTOR 和GGML_MAX_DIMS 是宏定义，定义了不同的 ggml 数据结构和最大参数。GGML_MAX_PARAMS 是 GGML 算法的最大参数数量。GGML_MAX_CONTEXTS 和GGML_MAX_SRC 是计算图的最大参数数量和源节点数量。GGML_MAX_NAME 是文本元素的最大文本长度。GGML_MAX_OP_PARAMS 是每个操作的最大参数数量。GGML_DEFAULT_N_THREADS 是默认线程数。GGML_DEFAULT_GRAPH_SIZE 是每个节点可以拥有的最大图形大小。GGML_MEM_ALIGN 是定义的内存对齐策略。


```cpp
#define GGML_FILE_MAGIC   0x67676d6c // "ggml"
#define GGML_FILE_VERSION 1

#define GGML_QNT_VERSION        2    // bump this on quantization format changes
#define GGML_QNT_VERSION_FACTOR 1000 // do not change this

#define GGML_MAX_DIMS           4
#define GGML_MAX_PARAMS         1024
#define GGML_MAX_CONTEXTS       64
#define GGML_MAX_SRC            6
#define GGML_MAX_NAME           64
#define GGML_MAX_OP_PARAMS      64
#define GGML_DEFAULT_N_THREADS  4
#define GGML_DEFAULT_GRAPH_SIZE 2048
#if UINTPTR_MAX == 0xFFFFFFFF
    #define GGML_MEM_ALIGN 4
```

这段代码定义了一些头文件和常量，其中一些常量定义了GGML和GGUF的标准，另外一些定义了GGML的出错处理方式。

以下是对每个部分的解释：

```cpp
#else
   #define GGML_MEM_ALIGN 16
#endif
```

这段代码定义了一个名为GGML_MEM_ALIGN的常量，其值为16。这个常量用于定义内存对齐策略，如果定义了GGML_MEM_ALIGN，那么在编译时会检查所有的内存对齐，如果对齐不正确，程序会崩溃。

```cpp
#define GGML_EXIT_SUCCESS 0
#define GGML_EXIT_ABORTED 1
```

这两行定义了GGML的出错处理方式。如果程序在编译时遇到了编译器错误或者定义的函数不符合要求，那么GGML将输出EXIT_SUCCESS或者EXIT_ABORTED，相应的值为0或者1。

```cpp
#define GGUF_MAGIC "GGUF"
```

这一行定义了一个名为GGUF_MAGIC的常量，其值为"GGUF"。GGUF是一个Fortran库，提供了一些方便的数学函数和数据结构，这个库可能需要定义函数或者使用这个常量。

```cpp
#define GGUF_VERSION 3
```

这一行定义了一个名为GGUF_VERSION的常量，其值为3。GGUF是一个Fortran库，这个库包含了很多数学函数和数据结构，这个常量用于定义GGUF库的版本。

```cpp
#define GGUF_DEFAULT_ALIGN 32
```

这一行定义了一个名为GGUF_DEFAULT_ALIGN的常量，其值为32。GGUF提供了一些数学函数和数据结构，默认的内存对齐策略是32字节，这个常量用于定义GGUF库默认的内存对齐策略。

```cpp
#define GGML_UNUSED(x) (void)(x)
```

这一行定义了一个名为GGML_UNUSED的函数，它的参数为单个变量x，它的作用是返回一个空值。这个函数定义了一种出错处理方式，如果定义了GGML_MEM_ALIGN，但是没有定义GGML_UNUSED，那么程序会在编译时捕捉到 unused variable的错误，并打印出错误信息。


```cpp
#else
    #define GGML_MEM_ALIGN 16
#endif

#define GGML_EXIT_SUCCESS 0
#define GGML_EXIT_ABORTED 1

#define GGUF_MAGIC "GGUF"

#define GGUF_VERSION 3

#define GGUF_DEFAULT_ALIGNMENT 32

#define GGML_UNUSED(x) (void)(x)

```

这段代码定义了三个头文件，其中第一个头文件是GGML_PAD，定义了一个带参数的函数，作用是对参数x进行类型转换，使得x可以被当做整型进行操作。第二个头文件是GGML_ASSERT，定义了一个伪命题，用于检查输入参数x是否为真，如果为真则执行后续语句，否则输出错误信息并返回程序外部的函数指针。第三个头文件是GGML_UNREACHABLE，定义了一个宏，表示程序无法继续执行，通常用于在程序中犯错时使用。

这里给出的是一个带参数的函数ggml_print_backtrace，并没有说明它的作用和使用场景，无法给出它的具体实现。


```cpp
#define GGML_PAD(x, n) (((x) + (n) - 1) & ~((n) - 1))

#define GGML_ASSERT(x) \
    do { \
        if (!(x)) { \
            fprintf(stderr, "GGML_ASSERT: %s:%d: %s\n", __FILE__, __LINE__, #x); \
            fflush(stderr); \
            fflush(stdout); \
            ggml_print_backtrace(); \
            exit(1); \
        } \
    } while (0)

#ifndef NDEBUG
#define GGML_UNREACHABLE() GGML_ASSERT(!"statement should not be reached")
```

这段代码定义了一个名为GGML_TENSOR_LOCALS的函数，它接受四个参数：类型（可以是int64、size_t等）、前缀（字符串，例如"int64_t"和"size_t"），指针和数组。它的作用是减少代码的重复，提高代码的可读性。

函数内部定义了一个名为GGML_UNREACHABLE的宏，它的值为1。如果函数定义时同时定义了__GNUC__，则将GGML_UNREACHABLE宏定义为产自GNU C库的宏，其值为1；否则，GGML_UNREACHABLE宏定义为内联函数，其值为0。


```cpp
#elif defined(__GNUC__)
#define GGML_UNREACHABLE() __builtin_unreachable()
#else
#define GGML_UNREACHABLE() ((void) 0)
#endif

// used to copy the number of elements and stride in bytes of tensors into local variables.
// main purpose is to reduce code duplication and improve readability.
//
// example:
//
//    GGML_TENSOR_LOCALS(int64_t, ne1, src1, ne);
//    GGML_TENSOR_LOCALS(size_t,  nb1, src1, nb);
//
#define GGML_TENSOR_LOCALS_1(type, prefix, pointer, array) \
    const type prefix##0 = (pointer)->array[0]; \
    GGML_UNUSED(prefix##0);
```

这段代码定义了一系列宏，用于在编译时检查特定类型的输入。它们看起来像是从某个通用的库或框架中继承下来的。

具体来说，这段代码定义了三个名为GGML_TENSOR_LOCALS_2、GGML_TENSOR_LOCALS_3和GGML_TENSOR_LOCALS的宏。这些宏的作用是覆盖掉一个名为GGML_TENSOR_LOCALS_1的定义，同时声明一个名为const type prefix##1的常量，和一个名为GGML_UNUSED(prefix##1)的未被定义的常量。

GGML_TENSOR_LOCALS_2进一步覆盖了GGML_TENSOR_LOCALS_1，声明了一个名为const type prefix##2的常量和一个未被定义的常量。GGML_TENSOR_LOCALS_3则进一步覆盖了GGML_TENSOR_LOCALS_2，声明了一个名为const type prefix##3的常量和一个未被定义的常量。

在整个段落中，有三个未被定义的常量。由于没有给这些常量赋值，它们的值将为27。同时，由于给定的代码不足以确定这些常量在哪些位置被定义，因此无法确定这些常量是否为const。


```cpp
#define GGML_TENSOR_LOCALS_2(type, prefix, pointer, array) \
    GGML_TENSOR_LOCALS_1    (type, prefix, pointer, array) \
    const type prefix##1 = (pointer)->array[1]; \
    GGML_UNUSED(prefix##1);
#define GGML_TENSOR_LOCALS_3(type, prefix, pointer, array) \
    GGML_TENSOR_LOCALS_2    (type, prefix, pointer, array) \
    const type prefix##2 = (pointer)->array[2]; \
    GGML_UNUSED(prefix##2);
#define GGML_TENSOR_LOCALS(type, prefix, pointer, array) \
    GGML_TENSOR_LOCALS_3  (type, prefix, pointer, array) \
    const type prefix##3 = (pointer)->array[3]; \
    GGML_UNUSED(prefix##3);

#ifdef  __cplusplus
extern "C" {
```

These are a list of加利福尼亚大学（UIUC）教授F♡A民族文化模型中，教授的编号。这些编号都表示 UIUC 教授在教授年金基金（GAFF）中的办公室或职位。

- 001: AI2683: AIAA Full Stack Professor
- 002: AI2683: AIAA Full Stack Professor
- 003: AI2683: AIAA Full Stack Professor
- 004: AI2683: AIAA Full Stack Professor
- 005: AI2683: AIAA Full Stack Professor
- 006: AI2683: AIAA Full Stack Professor
- 007: AI2683: AIAA Full Stack Professor
- 008: AI2683: AIAA Full Stack Professor
- 009: AI2683: AIAA Full Stack Professor
- 010: AI2683: AIAA Full Stack Professor
- 011: AI2683: AIAA Full Stack Professor
- 012: AI2683: AIAA Full Stack Professor
- 013: AI2683: AIAA Full Stack Professor
- 014: AI2683: AIAA Full Stack Professor
- 015: AI2683: AIAA Full Stack Professor
- 016: AI2683: AIAA Full Stack Professor
- 017: AI2683: AIAA Full Stack Professor
- 018: AI2683: AIAA Full Stack Professor
- 019: AI2683: AIAA Full Stack Professor
- 020: AI2683: AIAA Full Stack Professor
- 021: AI2683: AIAA Full Stack Professor
- 022: AI2683: AIAA Full Stack Professor
- 023: AI2683: AIAA Full Stack Professor
- 024: AI2683: AIAA Full Stack Professor
- 025: AI2683: AIAA Full Stack Professor
- 026: AI2683: AIAA Full Stack Professor
- 027: AI2683: AIAA Full Stack Professor
- 028: AI2683: AIAA Full Stack Professor
- 029: AI2683: AIAA Full Stack Professor
- 030: AI2683: AIAA Full Stack Professor
- 031: AI2683: AIAA Full Stack Professor
- 032: AI2683: AIAA Full Stack Professor
- 033: AI2683: AIAA Full Stack Professor
- 034: AI2683: AIAA Full Stack Professor
- 035: AI2683: AIAA Full Stack Professor
- 036: AI2683: AIAA Full Stack Professor
- 037: AI2683: AIAA Full Stack Professor
- 038: AI2683: AIAA Full Stack Professor
- 039: AI2683: AIAA Full Stack Professor
- 040: AI2683: AIAA Full Stack Professor
- 041: AI2683: AIAA Full Stack Professor
- 042: AI2683: AIAA Full Stack Professor
- 043: AI2683: AIAA Full Stack Professor
- 044: AI2683: AIAA Full Stack Professor
- 045: AI2683: AIAA Full Stack Professor
- 046: AI2683: AIAA Full Stack Professor
- 047: AI2683: AIAA Full Stack Professor
- 048: AI2683: AIAA Full Stack Professor
- 049: AI2683: AIAA Full Stack Professor
- 050: AI2683: AIAA Full Stack Professor
- 051: AI2683: AIAA Full Stack Professor
- 052: AI2683: AIAA Full Stack Professor
- 053: AI2683: AIAA Full Stack Professor
- 054: AI2683: AIAA Full Stack Professor
- 055: AI2683: AIAA Full Stack Professor
- 056: AI2683: AIAA Full Stack Professor
- 057: AI2683: AIAA Full Stack Professor
- 058: AI2683: AIAA Full Stack Professor
- 059: AI2683: AIAA Full Stack Professor
- 060: AI2683: AIAA Full Stack Professor
- 061: AI2683: AIAA Full Stack Professor
- 062: AI2683: AIAA Full Stack Professor
- 063: AI2683: AIAA Full Stack Professor
- 064: AI2683: AIAA Full Stack Professor
- 065: AI2683: AIAA Full Stack Professor
- 066: AI2683: AIAA Full Stack Professor
- 067: AI2683: AIAA Full Stack Professor
- 068: AI2683: AIAA Full Stack Professor
- 069: AI2683: AIAA Full Stack Professor
- 070: AI2683: AIAA Full Stack Professor
- 071: AI2683: AIAA Full Stack Professor
- 072: AI2683: AIAA Full Stack Professor
- 073: AI2683: AIAA Full Stack Professor
- 074: AI2683: AIAA Full Stack Professor
- 075: AI2683: AIAA Full Stack Professor
- 076: AI2683: AIAA Full Stack Professor
- 077: AI2683: AIAA Full Stack Professor
- 078: AI2683: AIAA Full Stack Professor
- 079: AI2683: AIAA Full Stack Professor
- 080: AI2683: AIAA Full Stack Professor
- 081: AI2683: AIAA Full Stack Professor
- 082: AI2683: AIAA Full Stack Professor
- 083: AI2683:


```cpp
#endif

#if defined(__ARM_NEON) && defined(__CUDACC__)
    typedef half ggml_fp16_t;
#elif defined(__ARM_NEON)
    typedef __fp16 ggml_fp16_t;
#else
    typedef uint16_t ggml_fp16_t;
#endif

    // convert FP16 <-> FP32
    GGML_API float       ggml_fp16_to_fp32(ggml_fp16_t x);
    GGML_API ggml_fp16_t ggml_fp32_to_fp16(float x);

    GGML_API void ggml_fp16_to_fp32_row(const ggml_fp16_t * x, float * y, int n);
    GGML_API void ggml_fp32_to_fp16_row(const float * x, ggml_fp16_t * y, int n);

    struct ggml_object;
    struct ggml_context;

    enum ggml_type {
        GGML_TYPE_F32  = 0,
        GGML_TYPE_F16  = 1,
        GGML_TYPE_Q4_0 = 2,
        GGML_TYPE_Q4_1 = 3,
        // GGML_TYPE_Q4_2 = 4, support has been removed
        // GGML_TYPE_Q4_3 (5) support has been removed
        GGML_TYPE_Q5_0 = 6,
        GGML_TYPE_Q5_1 = 7,
        GGML_TYPE_Q8_0 = 8,
        GGML_TYPE_Q8_1 = 9,
        // k-quantizations
        GGML_TYPE_Q2_K = 10,
        GGML_TYPE_Q3_K = 11,
        GGML_TYPE_Q4_K = 12,
        GGML_TYPE_Q5_K = 13,
        GGML_TYPE_Q6_K = 14,
        GGML_TYPE_Q8_K = 15,
        GGML_TYPE_I8,
        GGML_TYPE_I16,
        GGML_TYPE_I32,
        GGML_TYPE_COUNT,
    };

    enum ggml_backend_type {
        GGML_BACKEND_CPU = 0,
        GGML_BACKEND_GPU = 10,
        GGML_BACKEND_GPU_SPLIT = 20,
    };

    // model file types
    enum ggml_ftype {
        GGML_FTYPE_UNKNOWN     = -1,
        GGML_FTYPE_ALL_F32     = 0,
        GGML_FTYPE_MOSTLY_F16  = 1,  // except 1d tensors
        GGML_FTYPE_MOSTLY_Q4_0 = 2,  // except 1d tensors
        GGML_FTYPE_MOSTLY_Q4_1 = 3,  // except 1d tensors
        GGML_FTYPE_MOSTLY_Q4_1_SOME_F16 = 4, // tok_embeddings.weight and output.weight are F16
        GGML_FTYPE_MOSTLY_Q8_0 = 7,  // except 1d tensors
        GGML_FTYPE_MOSTLY_Q5_0 = 8,  // except 1d tensors
        GGML_FTYPE_MOSTLY_Q5_1 = 9,  // except 1d tensors
        GGML_FTYPE_MOSTLY_Q2_K = 10, // except 1d tensors
        GGML_FTYPE_MOSTLY_Q3_K = 11, // except 1d tensors
        GGML_FTYPE_MOSTLY_Q4_K = 12, // except 1d tensors
        GGML_FTYPE_MOSTLY_Q5_K = 13, // except 1d tensors
        GGML_FTYPE_MOSTLY_Q6_K = 14, // except 1d tensors
    };

    // available tensor operations:
    enum ggml_op {
        GGML_OP_NONE = 0,

        GGML_OP_DUP,
        GGML_OP_ADD,
        GGML_OP_ADD1,
        GGML_OP_ACC,
        GGML_OP_SUB,
        GGML_OP_MUL,
        GGML_OP_DIV,
        GGML_OP_SQR,
        GGML_OP_SQRT,
        GGML_OP_LOG,
        GGML_OP_SUM,
        GGML_OP_SUM_ROWS,
        GGML_OP_MEAN,
        GGML_OP_ARGMAX,
        GGML_OP_REPEAT,
        GGML_OP_REPEAT_BACK,
        GGML_OP_CONCAT,
        GGML_OP_SILU_BACK,
        GGML_OP_NORM, // normalize
        GGML_OP_RMS_NORM,
        GGML_OP_RMS_NORM_BACK,
        GGML_OP_GROUP_NORM,

        GGML_OP_MUL_MAT,
        GGML_OP_OUT_PROD,

        GGML_OP_SCALE,
        GGML_OP_SET,
        GGML_OP_CPY,
        GGML_OP_CONT,
        GGML_OP_RESHAPE,
        GGML_OP_VIEW,
        GGML_OP_PERMUTE,
        GGML_OP_TRANSPOSE,
        GGML_OP_GET_ROWS,
        GGML_OP_GET_ROWS_BACK,
        GGML_OP_DIAG,
        GGML_OP_DIAG_MASK_INF,
        GGML_OP_DIAG_MASK_ZERO,
        GGML_OP_SOFT_MAX,
        GGML_OP_SOFT_MAX_BACK,
        GGML_OP_ROPE,
        GGML_OP_ROPE_BACK,
        GGML_OP_ALIBI,
        GGML_OP_CLAMP,
        GGML_OP_CONV_1D,
        GGML_OP_CONV_1D_STAGE_0,  // internal
        GGML_OP_CONV_1D_STAGE_1,  // internal
        GGML_OP_CONV_TRANSPOSE_1D,
        GGML_OP_CONV_2D,
        GGML_OP_CONV_2D_STAGE_0, // internal
        GGML_OP_CONV_2D_STAGE_1, // internal
        GGML_OP_CONV_TRANSPOSE_2D,
        GGML_OP_POOL_1D,
        GGML_OP_POOL_2D,

        GGML_OP_UPSCALE, // nearest interpolate

        GGML_OP_FLASH_ATTN,
        GGML_OP_FLASH_FF,
        GGML_OP_FLASH_ATTN_BACK,
        GGML_OP_WIN_PART,
        GGML_OP_WIN_UNPART,
        GGML_OP_GET_REL_POS,
        GGML_OP_ADD_REL_POS,

        GGML_OP_UNARY,

        GGML_OP_MAP_UNARY,
        GGML_OP_MAP_BINARY,

        GGML_OP_MAP_CUSTOM1_F32,
        GGML_OP_MAP_CUSTOM2_F32,
        GGML_OP_MAP_CUSTOM3_F32,

        GGML_OP_MAP_CUSTOM1,
        GGML_OP_MAP_CUSTOM2,
        GGML_OP_MAP_CUSTOM3,

        GGML_OP_CROSS_ENTROPY_LOSS,
        GGML_OP_CROSS_ENTROPY_LOSS_BACK,

        GGML_OP_COUNT,
    };

    enum ggml_unary_op {
        GGML_UNARY_OP_ABS,
        GGML_UNARY_OP_SGN,
        GGML_UNARY_OP_NEG,
        GGML_UNARY_OP_STEP,
        GGML_UNARY_OP_TANH,
        GGML_UNARY_OP_ELU,
        GGML_UNARY_OP_RELU,
        GGML_UNARY_OP_GELU,
        GGML_UNARY_OP_GELU_QUICK,
        GGML_UNARY_OP_SILU,
        GGML_UNARY_OP_LEAKY
    };

    enum ggml_object_type {
        GGML_OBJECT_TENSOR,
        GGML_OBJECT_GRAPH,
        GGML_OBJECT_WORK_BUFFER
    };

    enum ggml_log_level {
        GGML_LOG_LEVEL_ERROR = 2,
        GGML_LOG_LEVEL_WARN = 3,
        GGML_LOG_LEVEL_INFO = 4
    };

    // ggml object
    struct ggml_object {
        size_t offs;
        size_t size;

        struct ggml_object * next;

        enum ggml_object_type type;

        char padding[4];
    };

    static const size_t GGML_OBJECT_SIZE = sizeof(struct ggml_object);

    // n-dimensional tensor
    struct ggml_tensor {
        enum ggml_type         type;
        enum ggml_backend_type backend;

        struct ggml_backend_buffer * buffer;

        int     n_dims;
        int64_t ne[GGML_MAX_DIMS]; // number of elements
        size_t  nb[GGML_MAX_DIMS]; // stride in bytes:
                                   // nb[0] = ggml_type_size(type)
                                   // nb[1] = nb[0]   * (ne[0] / ggml_blck_size(type)) + padding
                                   // nb[i] = nb[i-1] * ne[i-1]

        // compute data
        enum ggml_op op;

        // op params - allocated as int32_t for alignment
        int32_t op_params[GGML_MAX_OP_PARAMS / sizeof(int32_t)];

        bool is_param;

        struct ggml_tensor * grad;
        struct ggml_tensor * src[GGML_MAX_SRC];

        // performance
        int     perf_runs;
        int64_t perf_cycles;
        int64_t perf_time_us;

        struct ggml_tensor * view_src;
        size_t               view_offs;

        void * data;

        char name[GGML_MAX_NAME];

        void * extra; // extra things e.g. for ggml-cuda.cu

        char padding[12];
    };

    static const size_t GGML_TENSOR_SIZE = sizeof(struct ggml_tensor);

    // the compute plan that needs to be prepared for ggml_graph_compute()
    // since https://github.com/ggerganov/ggml/issues/287
    struct ggml_cplan {
        size_t    work_size; // size of work buffer, calculated by `ggml_graph_plan()`
        uint8_t * work_data; // work buffer, to be allocated by caller before calling to `ggml_graph_compute()`

        int n_threads;

        // abort ggml_graph_compute when true
        bool (*abort_callback)(void * data);
        void * abort_callback_data;
    };

    enum ggml_cgraph_eval_order {
        GGML_CGRAPH_EVAL_ORDER_LEFT_TO_RIGHT = 0,
        GGML_CGRAPH_EVAL_ORDER_RIGHT_TO_LEFT,
        GGML_CGRAPH_EVAL_ORDER_COUNT
    };

    struct ggml_hash_set {
        size_t size;
        struct ggml_tensor ** keys;
    };

    // computation graph
    struct ggml_cgraph {
        int size;
        int n_nodes;
        int n_leafs;

        struct ggml_tensor ** nodes;
        struct ggml_tensor ** grads;
        struct ggml_tensor ** leafs;

        struct ggml_hash_set visited_hash_table;

        enum ggml_cgraph_eval_order order;

        // performance
        int     perf_runs;
        int64_t perf_cycles;
        int64_t perf_time_us;
    };

    // scratch buffer
    struct ggml_scratch {
        size_t offs;
        size_t size;
        void * data;
    };

    struct ggml_init_params {
        // memory pool
        size_t mem_size;   // bytes
        void * mem_buffer; // if NULL, memory will be allocated internally
        bool   no_alloc;   // don't allocate memory for the tensor data
    };


    // compute types

    // NOTE: the INIT or FINALIZE pass is not scheduled unless explicitly enabled.
    // This behavior was changed since https://github.com/ggerganov/llama.cpp/pull/1995.
    enum ggml_task_type {
        GGML_TASK_INIT = 0,
        GGML_TASK_COMPUTE,
        GGML_TASK_FINALIZE,
    };

    struct ggml_compute_params {
        enum ggml_task_type type;

        // ith = thread index, nth = number of threads
        int ith, nth;

        // work buffer for all threads
        size_t wsize;
        void * wdata;
    };

    // misc

    GGML_API void    ggml_time_init(void); // call this once at the beginning of the program
    GGML_API int64_t ggml_time_ms(void);
    GGML_API int64_t ggml_time_us(void);
    GGML_API int64_t ggml_cycles(void);
    GGML_API int64_t ggml_cycles_per_ms(void);

    GGML_API void    ggml_print_backtrace(void);

    GGML_API void    ggml_numa_init(void); // call once for better performance on NUMA systems
    GGML_API bool    ggml_is_numa(void); // true if init detected that system has >1 NUMA node

    GGML_API void    ggml_print_object (const struct ggml_object * obj);
    GGML_API void    ggml_print_objects(const struct ggml_context * ctx);

    GGML_API int64_t ggml_nelements   (const struct ggml_tensor * tensor);
    GGML_API int64_t ggml_nrows       (const struct ggml_tensor * tensor);
    GGML_API size_t  ggml_nbytes      (const struct ggml_tensor * tensor);
    GGML_API size_t  ggml_nbytes_pad  (const struct ggml_tensor * tensor); // same as ggml_nbytes() but padded to GGML_MEM_ALIGN
    GGML_API size_t  ggml_nbytes_split(const struct ggml_tensor * tensor, int nrows_split);

    GGML_API int     ggml_blck_size (enum ggml_type type);
    GGML_API size_t  ggml_type_size (enum ggml_type type); // size in bytes for all elements in a block
    GGML_API float   ggml_type_sizef(enum ggml_type type); // ggml_type_size()/ggml_blck_size() as float

    GGML_API const char * ggml_type_name(enum ggml_type type);
    GGML_API const char * ggml_op_name  (enum ggml_op   op);
    GGML_API const char * ggml_op_symbol(enum ggml_op   op);

    GGML_API size_t  ggml_element_size(const struct ggml_tensor * tensor);

    GGML_API bool    ggml_is_quantized(enum ggml_type type);

    // TODO: temporary until model loading of ggml examples is refactored
    GGML_API enum ggml_type ggml_ftype_to_ggml_type(enum ggml_ftype ftype);

    GGML_API bool ggml_is_transposed(const struct ggml_tensor * tensor);
    GGML_API bool ggml_is_contiguous(const struct ggml_tensor * tensor);
    GGML_API bool ggml_is_permuted  (const struct ggml_tensor * tensor);

    GGML_API bool ggml_are_same_shape(const struct ggml_tensor * t0, const struct ggml_tensor * t1);

    // use this to compute the memory overhead of a tensor
    GGML_API size_t ggml_tensor_overhead(void);

    // main

    GGML_API struct ggml_context * ggml_init(struct ggml_init_params params);
    GGML_API void                  ggml_free(struct ggml_context * ctx);

    GGML_API size_t  ggml_used_mem(const struct ggml_context * ctx);

    GGML_API size_t  ggml_set_scratch (struct ggml_context * ctx, struct ggml_scratch scratch);
    GGML_API bool    ggml_get_no_alloc(struct ggml_context * ctx);
    GGML_API void    ggml_set_no_alloc(struct ggml_context * ctx, bool no_alloc);

    GGML_API void *  ggml_get_mem_buffer     (const struct ggml_context * ctx);
    GGML_API size_t  ggml_get_mem_size       (const struct ggml_context * ctx);
    GGML_API size_t  ggml_get_max_tensor_size(const struct ggml_context * ctx);

    GGML_API struct ggml_tensor * ggml_new_tensor(
            struct ggml_context * ctx,
            enum   ggml_type type,
            int    n_dims,
            const int64_t *ne);

    GGML_API struct ggml_tensor * ggml_new_tensor_1d(
            struct ggml_context * ctx,
            enum   ggml_type type,
            int64_t ne0);

    GGML_API struct ggml_tensor * ggml_new_tensor_2d(
            struct ggml_context * ctx,
            enum   ggml_type type,
            int64_t ne0,
            int64_t ne1);

    GGML_API struct ggml_tensor * ggml_new_tensor_3d(
            struct ggml_context * ctx,
            enum   ggml_type type,
            int64_t ne0,
            int64_t ne1,
            int64_t ne2);

    GGML_API struct ggml_tensor * ggml_new_tensor_4d(
            struct ggml_context * ctx,
            enum   ggml_type type,
            int64_t ne0,
            int64_t ne1,
            int64_t ne2,
            int64_t ne3);

    GGML_API struct ggml_tensor * ggml_new_i32(struct ggml_context * ctx, int32_t value);
    GGML_API struct ggml_tensor * ggml_new_f32(struct ggml_context * ctx, float value);

    GGML_API struct ggml_tensor * ggml_dup_tensor (struct ggml_context * ctx, const struct ggml_tensor * src);
    GGML_API struct ggml_tensor * ggml_view_tensor(struct ggml_context * ctx, struct ggml_tensor * src);

    // Context tensor enumeration and lookup
    GGML_API struct ggml_tensor * ggml_get_first_tensor(struct ggml_context * ctx);
    GGML_API struct ggml_tensor * ggml_get_next_tensor (struct ggml_context * ctx, struct ggml_tensor * tensor);
    GGML_API struct ggml_tensor * ggml_get_tensor(struct ggml_context * ctx, const char * name);

    GGML_API struct ggml_tensor * ggml_set_zero(struct ggml_tensor * tensor);
    GGML_API struct ggml_tensor * ggml_set_i32 (struct ggml_tensor * tensor, int32_t value);
    GGML_API struct ggml_tensor * ggml_set_f32 (struct ggml_tensor * tensor, float value);

    // Converts a flat index into coordinates
    GGML_API void    ggml_unravel_index(const struct ggml_tensor * tensor, int64_t i, int64_t * i0, int64_t * i1, int64_t * i2, int64_t * i3);

    GGML_API int32_t ggml_get_i32_1d(const struct ggml_tensor * tensor, int i);
    GGML_API void    ggml_set_i32_1d(const struct ggml_tensor * tensor, int i, int32_t value);

    GGML_API int32_t ggml_get_i32_nd(const struct ggml_tensor * tensor, int i0, int i1, int i2, int i3);
    GGML_API void    ggml_set_i32_nd(const struct ggml_tensor * tensor, int i0, int i1, int i2, int i3, int32_t value);

    GGML_API float   ggml_get_f32_1d(const struct ggml_tensor * tensor, int i);
    GGML_API void    ggml_set_f32_1d(const struct ggml_tensor * tensor, int i, float value);

    GGML_API float   ggml_get_f32_nd(const struct ggml_tensor * tensor, int i0, int i1, int i2, int i3);
    GGML_API void    ggml_set_f32_nd(const struct ggml_tensor * tensor, int i0, int i1, int i2, int i3, float value);

    GGML_API void *  ggml_get_data    (const struct ggml_tensor * tensor);
    GGML_API float * ggml_get_data_f32(const struct ggml_tensor * tensor);

    GGML_API enum ggml_unary_op ggml_get_unary_op(const struct ggml_tensor * tensor);

    GGML_API const char *         ggml_get_name   (const struct ggml_tensor * tensor);
    GGML_API struct ggml_tensor * ggml_set_name   (      struct ggml_tensor * tensor, const char * name);
    GGML_ATTRIBUTE_FORMAT(2, 3)
    GGML_API struct ggml_tensor * ggml_format_name(      struct ggml_tensor * tensor, const char * fmt, ...);

    //
    // operations on tensors with backpropagation
    //

    GGML_API struct ggml_tensor * ggml_dup(
            struct ggml_context * ctx,
            struct ggml_tensor  * a);

    // in-place, returns view(a)
    GGML_API struct ggml_tensor * ggml_dup_inplace(
            struct ggml_context * ctx,
            struct ggml_tensor  * a);

    GGML_API struct ggml_tensor * ggml_add(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            struct ggml_tensor  * b);

    GGML_API struct ggml_tensor * ggml_add_inplace(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            struct ggml_tensor  * b);

    GGML_API struct ggml_tensor * ggml_add_cast(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            struct ggml_tensor  * b,
            enum   ggml_type      type);

    GGML_API struct ggml_tensor * ggml_add1(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            struct ggml_tensor  * b);

    GGML_API struct ggml_tensor * ggml_add1_inplace(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            struct ggml_tensor  * b);

    GGML_API struct ggml_tensor * ggml_acc(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            struct ggml_tensor  * b,
            size_t                nb1,
            size_t                nb2,
            size_t                nb3,
            size_t                offset);

    GGML_API struct ggml_tensor * ggml_acc_inplace(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            struct ggml_tensor  * b,
            size_t                nb1,
            size_t                nb2,
            size_t                nb3,
            size_t                offset);

    GGML_API struct ggml_tensor * ggml_sub(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            struct ggml_tensor  * b);

    GGML_API struct ggml_tensor * ggml_sub_inplace(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            struct ggml_tensor  * b);

    GGML_API struct ggml_tensor * ggml_mul(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            struct ggml_tensor  * b);

    GGML_API struct ggml_tensor * ggml_mul_inplace(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            struct ggml_tensor  * b);

    GGML_API struct ggml_tensor * ggml_div(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            struct ggml_tensor  * b);

    GGML_API struct ggml_tensor * ggml_div_inplace(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            struct ggml_tensor  * b);

    GGML_API struct ggml_tensor * ggml_sqr(
            struct ggml_context * ctx,
            struct ggml_tensor  * a);

    GGML_API struct ggml_tensor * ggml_sqr_inplace(
            struct ggml_context * ctx,
            struct ggml_tensor  * a);

    GGML_API struct ggml_tensor * ggml_sqrt(
            struct ggml_context * ctx,
            struct ggml_tensor  * a);

    GGML_API struct ggml_tensor * ggml_sqrt_inplace(
            struct ggml_context * ctx,
            struct ggml_tensor  * a);

    GGML_API struct ggml_tensor * ggml_log(
            struct ggml_context * ctx,
            struct ggml_tensor  * a);

    GGML_API struct ggml_tensor * ggml_log_inplace(
            struct ggml_context * ctx,
            struct ggml_tensor  * a);

    // return scalar
    GGML_API struct ggml_tensor * ggml_sum(
            struct ggml_context * ctx,
            struct ggml_tensor  * a);

    // sums along rows, with input shape [a,b,c,d] return shape [1,b,c,d]
    GGML_API struct ggml_tensor * ggml_sum_rows(
            struct ggml_context * ctx,
            struct ggml_tensor  * a);

    // mean along rows
    GGML_API struct ggml_tensor * ggml_mean(
            struct ggml_context * ctx,
            struct ggml_tensor  * a);

    // argmax along rows
    GGML_API struct ggml_tensor * ggml_argmax(
            struct ggml_context * ctx,
            struct ggml_tensor  * a);

    // if a is the same shape as b, and a is not parameter, return a
    // otherwise, return a new tensor: repeat(a) to fit in b
    GGML_API struct ggml_tensor * ggml_repeat(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            struct ggml_tensor  * b);

    // sums repetitions in a into shape of b
    GGML_API struct ggml_tensor * ggml_repeat_back(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            struct ggml_tensor  * b);

    // concat a and b on dim 2
    // used in stable-diffusion
    GGML_API struct ggml_tensor * ggml_concat(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            struct ggml_tensor  * b);

    GGML_API struct ggml_tensor * ggml_abs(
            struct ggml_context * ctx,
            struct ggml_tensor  * a);

    GGML_API struct ggml_tensor * ggml_abs_inplace(
            struct ggml_context * ctx,
            struct ggml_tensor  * a);

    GGML_API struct ggml_tensor * ggml_sgn(
            struct ggml_context * ctx,
            struct ggml_tensor  * a);

    GGML_API struct ggml_tensor * ggml_sgn_inplace(
            struct ggml_context * ctx,
            struct ggml_tensor  * a);

    GGML_API struct ggml_tensor * ggml_neg(
            struct ggml_context * ctx,
            struct ggml_tensor  * a);

    GGML_API struct ggml_tensor * ggml_neg_inplace(
            struct ggml_context * ctx,
            struct ggml_tensor  * a);

    GGML_API struct ggml_tensor * ggml_step(
            struct ggml_context * ctx,
            struct ggml_tensor  * a);

    GGML_API struct ggml_tensor * ggml_step_inplace(
            struct ggml_context * ctx,
            struct ggml_tensor  * a);

    GGML_API struct ggml_tensor * ggml_tanh(
            struct ggml_context * ctx,
            struct ggml_tensor  * a);

    GGML_API struct ggml_tensor * ggml_tanh_inplace(
            struct ggml_context * ctx,
            struct ggml_tensor  * a);

    GGML_API struct ggml_tensor * ggml_elu(
            struct ggml_context * ctx,
            struct ggml_tensor  * a);

    GGML_API struct ggml_tensor * ggml_elu_inplace(
            struct ggml_context * ctx,
            struct ggml_tensor  * a);

    GGML_API struct ggml_tensor * ggml_relu(
            struct ggml_context * ctx,
            struct ggml_tensor  * a);

    GGML_API struct ggml_tensor * ggml_leaky(
            struct ggml_context * ctx,
            struct ggml_tensor  * a);

    GGML_API struct ggml_tensor * ggml_relu_inplace(
            struct ggml_context * ctx,
            struct ggml_tensor  * a);

    // TODO: double-check this computation is correct
    GGML_API struct ggml_tensor * ggml_gelu(
            struct ggml_context * ctx,
            struct ggml_tensor  * a);

    GGML_API struct ggml_tensor * ggml_gelu_inplace(
            struct ggml_context * ctx,
            struct ggml_tensor  * a);

    GGML_API struct ggml_tensor * ggml_gelu_quick(
            struct ggml_context * ctx,
            struct ggml_tensor  * a);

    GGML_API struct ggml_tensor * ggml_gelu_quick_inplace(
            struct ggml_context * ctx,
            struct ggml_tensor  * a);

    GGML_API struct ggml_tensor * ggml_silu(
            struct ggml_context * ctx,
            struct ggml_tensor  * a);

    GGML_API struct ggml_tensor * ggml_silu_inplace(
            struct ggml_context * ctx,
            struct ggml_tensor  * a);

    // a - x
    // b - dy
    GGML_API struct ggml_tensor * ggml_silu_back(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            struct ggml_tensor  * b);

    // normalize along rows
    GGML_API struct ggml_tensor * ggml_norm(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            float                 eps);

    GGML_API struct ggml_tensor * ggml_norm_inplace(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            float                 eps);

    GGML_API struct ggml_tensor * ggml_rms_norm(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            float                 eps);

    GGML_API struct ggml_tensor * ggml_rms_norm_inplace(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            float                 eps);

    // group normalize along ne0*ne1*n_groups
    // used in stable-diffusion
    // TODO: eps is hardcoded to 1e-6 for now
    GGML_API struct ggml_tensor * ggml_group_norm(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            int                   n_groups);

    GGML_API struct ggml_tensor * ggml_group_norm_inplace(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            int                   n_groups);

    // a - x
    // b - dy
    GGML_API struct ggml_tensor * ggml_rms_norm_back(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            struct ggml_tensor  * b,
            float                 eps);

    // A: k columns, n rows => [ne03, ne02, n, k]
    // B: k columns, m rows  (i.e. we transpose it internally) => [ne03 * x, ne02 * y, m, k]
    // result is n columns, m rows => [ne03 * x, ne02 * y, m, n]
    GGML_API struct ggml_tensor * ggml_mul_mat(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            struct ggml_tensor  * b);

    // A: m columns, n rows,
    // B: p columns, n rows,
    // result is m columns, p rows
    GGML_API struct ggml_tensor * ggml_out_prod(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            struct ggml_tensor  * b);

    //
    // operations on tensors without backpropagation
    //

    GGML_API struct ggml_tensor * ggml_scale(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            struct ggml_tensor  * b);

    // in-place, returns view(a)
    GGML_API struct ggml_tensor * ggml_scale_inplace(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            struct ggml_tensor  * b);

    // b -> view(a,offset,nb1,nb2,3), return modified a
    GGML_API struct ggml_tensor * ggml_set(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            struct ggml_tensor  * b,
            size_t                nb1,
            size_t                nb2,
            size_t                nb3,
            size_t                offset);

    // b -> view(a,offset,nb1,nb2,3), return view(a)
    GGML_API struct ggml_tensor * ggml_set_inplace(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            struct ggml_tensor  * b,
            size_t                nb1,
            size_t                nb2,
            size_t                nb3,
            size_t                offset);

    GGML_API struct ggml_tensor * ggml_set_1d(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            struct ggml_tensor  * b,
            size_t                offset);

    GGML_API struct ggml_tensor * ggml_set_1d_inplace(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            struct ggml_tensor  * b,
            size_t                offset);

    // b -> view(a,offset,nb1,nb2,3), return modified a
    GGML_API struct ggml_tensor * ggml_set_2d(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            struct ggml_tensor  * b,
            size_t                nb1,
            size_t                offset);

    // b -> view(a,offset,nb1,nb2,3), return view(a)
    GGML_API struct ggml_tensor * ggml_set_2d_inplace(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            struct ggml_tensor  * b,
            size_t                nb1,
            size_t                offset);

    // a -> b, return view(b)
    GGML_API struct ggml_tensor * ggml_cpy(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            struct ggml_tensor  * b);

    // a -> b, in-place, return view(b)
    GGML_API struct ggml_tensor * ggml_cpy_inplace(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            struct ggml_tensor  * b);

    // make contiguous
    GGML_API struct ggml_tensor * ggml_cont(
            struct ggml_context * ctx,
            struct ggml_tensor  * a);

    // make contiguous, in-place
    GGML_API struct ggml_tensor * ggml_cont_inplace(
            struct ggml_context * ctx,
            struct ggml_tensor  * a);

    // make contiguous, with new shape
    GGML_API struct ggml_tensor * ggml_cont_1d(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            int64_t               ne0);

    GGML_API struct ggml_tensor * ggml_cont_2d(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            int64_t               ne0,
            int64_t               ne1);

    GGML_API struct ggml_tensor * ggml_cont_3d(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            int64_t               ne0,
            int64_t               ne1,
            int64_t               ne2);

    GGML_API struct ggml_tensor * ggml_cont_4d(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            int64_t               ne0,
            int64_t               ne1,
            int64_t               ne2,
            int64_t               ne3);

    // return view(a), b specifies the new shape
    // TODO: when we start computing gradient, make a copy instead of view
    GGML_API struct ggml_tensor * ggml_reshape(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            struct ggml_tensor  * b);

    // return view(a)
    // TODO: when we start computing gradient, make a copy instead of view
    GGML_API struct ggml_tensor * ggml_reshape_1d(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            int64_t               ne0);

    GGML_API struct ggml_tensor * ggml_reshape_2d(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            int64_t               ne0,
            int64_t               ne1);

    // return view(a)
    // TODO: when we start computing gradient, make a copy instead of view
    GGML_API struct ggml_tensor * ggml_reshape_3d(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            int64_t               ne0,
            int64_t               ne1,
            int64_t               ne2);

    GGML_API struct ggml_tensor * ggml_reshape_4d(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            int64_t               ne0,
            int64_t               ne1,
            int64_t               ne2,
            int64_t               ne3);

    // offset in bytes
    GGML_API struct ggml_tensor * ggml_view_1d(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            int64_t               ne0,
            size_t                offset);

    GGML_API struct ggml_tensor * ggml_view_2d(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            int64_t               ne0,
            int64_t               ne1,
            size_t                nb1, // row stride in bytes
            size_t                offset);

    GGML_API struct ggml_tensor * ggml_view_3d(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            int64_t               ne0,
            int64_t               ne1,
            int64_t               ne2,
            size_t                nb1, // row   stride in bytes
            size_t                nb2, // slice stride in bytes
            size_t                offset);

    GGML_API struct ggml_tensor * ggml_view_4d(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            int64_t               ne0,
            int64_t               ne1,
            int64_t               ne2,
            int64_t               ne3,
            size_t                nb1, // row   stride in bytes
            size_t                nb2, // slice stride in bytes
            size_t                nb3,
            size_t                offset);

    GGML_API struct ggml_tensor * ggml_permute(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            int                   axis0,
            int                   axis1,
            int                   axis2,
            int                   axis3);

    // alias for ggml_permute(ctx, a, 1, 0, 2, 3)
    GGML_API struct ggml_tensor * ggml_transpose(
            struct ggml_context * ctx,
            struct ggml_tensor  * a);

    GGML_API struct ggml_tensor * ggml_get_rows(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            struct ggml_tensor  * b);

    GGML_API struct ggml_tensor * ggml_get_rows_back(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            struct ggml_tensor  * b,
            struct ggml_tensor  * c);

    GGML_API struct ggml_tensor * ggml_diag(
        struct ggml_context     * ctx,
        struct ggml_tensor      * a);

    // set elements above the diagonal to -INF
    GGML_API struct ggml_tensor * ggml_diag_mask_inf(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            int                   n_past);

    // in-place, returns view(a)
    GGML_API struct ggml_tensor * ggml_diag_mask_inf_inplace(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            int                   n_past);

    // set elements above the diagonal to 0
    GGML_API struct ggml_tensor * ggml_diag_mask_zero(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            int                   n_past);

    // in-place, returns view(a)
    GGML_API struct ggml_tensor * ggml_diag_mask_zero_inplace(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            int                   n_past);

    GGML_API struct ggml_tensor * ggml_soft_max(
            struct ggml_context * ctx,
            struct ggml_tensor  * a);

    // in-place, returns view(a)
    GGML_API struct ggml_tensor * ggml_soft_max_inplace(
            struct ggml_context * ctx,
            struct ggml_tensor  * a);

    GGML_API struct ggml_tensor * ggml_soft_max_back(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            struct ggml_tensor  * b);

    // in-place, returns view(a)
    GGML_API struct ggml_tensor * ggml_soft_max_back_inplace(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            struct ggml_tensor  * b);

    // rotary position embedding
    // if mode & 1 == 1, skip n_past elements (DEPRECATED)
    // if mode & 2 == 1, GPT-NeoX style
    // if mode & 4 == 1, ChatGLM style
    //
    // b is an int32 vector with size a->ne[2], it contains the positions
    GGML_API struct ggml_tensor * ggml_rope(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            struct ggml_tensor  * b,
            int                   n_dims,
            int                   mode,
            int                   n_ctx);

    // in-place, returns view(a)
    GGML_API struct ggml_tensor * ggml_rope_inplace(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            struct ggml_tensor  * b,
            int                   n_dims,
            int                   mode,
            int                   n_ctx);

    // custom RoPE
    GGML_API struct ggml_tensor * ggml_rope_custom(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            struct ggml_tensor  * b,
            int                   n_dims,
            int                   mode,
            int                   n_ctx,
            int                   n_orig_ctx,
            float                 freq_base,
            float                 freq_scale,
            float                 ext_factor,
            float                 attn_factor,
            float                 beta_fast,
            float                 beta_slow);

    // in-place, returns view(a)
    GGML_API struct ggml_tensor * ggml_rope_custom_inplace(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            struct ggml_tensor  * b,
            int                   n_dims,
            int                   mode,
            int                   n_ctx,
            int                   n_orig_ctx,
            float                 freq_base,
            float                 freq_scale,
            float                 ext_factor,
            float                 attn_factor,
            float                 beta_fast,
            float                 beta_slow);

    // compute correction dims for YaRN RoPE scaling
    void ggml_rope_yarn_corr_dims(
        int n_dims, int n_orig_ctx, float freq_base, float beta_fast, float beta_slow, float dims[2]);

    // xPos RoPE, in-place, returns view(a)
    GGML_API struct ggml_tensor * ggml_rope_xpos_inplace(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            struct ggml_tensor  * b,
            int                   n_dims,
            float                 base,
            bool                  down);

    // rotary position embedding backward, i.e compute dx from dy
    // a - dy
    GGML_API struct ggml_tensor * ggml_rope_back(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            struct ggml_tensor  * b,
            int                   n_dims,
            int                   mode,
            int                   n_ctx,
            float                 freq_base,
            float                 freq_scale,
            float                 xpos_base,
            bool                  xpos_down);

    // alibi position embedding
    // in-place, returns view(a)
    GGML_API struct ggml_tensor * ggml_alibi(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            int                   n_past,
            int                   n_head,
            float                 bias_max);

    // clamp
    // in-place, returns view(a)
    GGML_API struct ggml_tensor * ggml_clamp(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            float                 min,
            float                 max);

    GGML_API struct ggml_tensor * ggml_conv_1d(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            struct ggml_tensor  * b,
            int                   s0,  // stride
            int                   p0,  // padding
            int                   d0); // dilation

    // conv_1d with padding = half
    // alias for ggml_conv_1d(a, b, s, a->ne[0]/2, d)
    GGML_API struct ggml_tensor* ggml_conv_1d_ph(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            struct ggml_tensor  * b,
            int                   s,
            int                   d);

    GGML_API struct ggml_tensor * ggml_conv_transpose_1d(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            struct ggml_tensor  * b,
            int                   s0,
            int                   p0,
            int                   d0);

    GGML_API struct ggml_tensor * ggml_conv_2d(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            struct ggml_tensor  * b,
            int                   s0,
            int                   s1,
            int                   p0,
            int                   p1,
            int                   d0,
            int                   d1);


    // kernel size is a->ne[0] x a->ne[1]
    // stride is equal to kernel size
    // padding is zero
    // example:
    // a:     16   16    3  768
    // b:   1024 1024    3    1
    // res:   64   64  768    1
    // used in sam
    GGML_API struct ggml_tensor * ggml_conv_2d_sk_p0(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            struct ggml_tensor  * b);

    // kernel size is a->ne[0] x a->ne[1]
    // stride is 1
    // padding is half
    // example:
    // a:      3    3    256  256
    // b:     64   64    256    1
    // res:   64   64    256    1
    // used in sam
    GGML_API struct ggml_tensor * ggml_conv_2d_s1_ph(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            struct ggml_tensor  * b);

    GGML_API struct ggml_tensor * ggml_conv_transpose_2d_p0(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            struct ggml_tensor  * b,
            int                   stride);

    enum ggml_op_pool {
        GGML_OP_POOL_MAX,
        GGML_OP_POOL_AVG,
        GGML_OP_POOL_COUNT,
    };

    GGML_API struct ggml_tensor * ggml_pool_1d(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            enum ggml_op_pool     op,
            int                   k0, // kernel size
            int                   s0, // stride
            int                   p0); // padding

    // the result will have 2*p0 padding for the first dimension
    // and 2*p1 padding for the second dimension
    GGML_API struct ggml_tensor * ggml_pool_2d(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            enum ggml_op_pool     op,
            int                   k0,
            int                   k1,
            int                   s0,
            int                   s1,
            float                 p0,
            float                 p1);

    // nearest interpolate
    // used in stable-diffusion
    GGML_API struct ggml_tensor * ggml_upscale(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            int                   scale_factor);

    GGML_API struct ggml_tensor * ggml_flash_attn(
            struct ggml_context * ctx,
            struct ggml_tensor  * q,
            struct ggml_tensor  * k,
            struct ggml_tensor  * v,
            bool                  masked);

    GGML_API struct ggml_tensor * ggml_flash_attn_back(
           struct ggml_context * ctx,
           struct ggml_tensor  * q,
           struct ggml_tensor  * k,
           struct ggml_tensor  * v,
           struct ggml_tensor  * d,
           bool                  masked);

    GGML_API struct ggml_tensor * ggml_flash_ff(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            struct ggml_tensor  * b0,
            struct ggml_tensor  * b1,
            struct ggml_tensor  * c0,
            struct ggml_tensor  * c1);

    // partition into non-overlapping windows with padding if needed
    // example:
    // a:   768   64   64    1
    // w:    14
    // res: 768   14   14    25
    // used in sam
    GGML_API struct ggml_tensor * ggml_win_part(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            int                   w);

    // reverse of ggml_win_part
    // used in sam
    GGML_API struct ggml_tensor * ggml_win_unpart(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            int                   w0,
            int                   h0,
            int                   w);

    GGML_API struct ggml_tensor * ggml_unary(
            struct ggml_context * ctx,
             struct ggml_tensor * a,
             enum ggml_unary_op op);

    GGML_API struct ggml_tensor * ggml_unary_inplace(
        struct ggml_context * ctx,
        struct ggml_tensor  * a,
        enum ggml_unary_op op);

    // used in sam
    GGML_API struct ggml_tensor * ggml_get_rel_pos(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            int                   qh,
            int                   kh);

    // used in sam

    GGML_API struct ggml_tensor * ggml_add_rel_pos(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            struct ggml_tensor  * pw,
            struct ggml_tensor  * ph);

    GGML_API struct ggml_tensor * ggml_add_rel_pos_inplace(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            struct ggml_tensor  * pw,
            struct ggml_tensor  * ph);

    // custom operators

    typedef void (*ggml_unary_op_f32_t) (const int, float *, const float *);
    typedef void (*ggml_binary_op_f32_t)(const int, float *, const float *, const float *);

    typedef void (*ggml_custom1_op_f32_t)(struct ggml_tensor *, const struct ggml_tensor *);
    typedef void (*ggml_custom2_op_f32_t)(struct ggml_tensor *, const struct ggml_tensor *, const struct ggml_tensor *);
    typedef void (*ggml_custom3_op_f32_t)(struct ggml_tensor *, const struct ggml_tensor *, const struct ggml_tensor *, const struct ggml_tensor *);

    GGML_DEPRECATED(GGML_API struct ggml_tensor * ggml_map_unary_f32(
            struct ggml_context        * ctx,
            struct ggml_tensor         * a,
                   ggml_unary_op_f32_t   fun),
        "use ggml_map_custom1 instead");

    GGML_DEPRECATED(GGML_API struct ggml_tensor * ggml_map_unary_inplace_f32(
            struct ggml_context        * ctx,
            struct ggml_tensor         * a,
                   ggml_unary_op_f32_t   fun),
        "use ggml_map_custom1_inplace instead");

    GGML_DEPRECATED(GGML_API struct ggml_tensor * ggml_map_binary_f32(
            struct ggml_context         * ctx,
            struct ggml_tensor          * a,
            struct ggml_tensor          * b,
                   ggml_binary_op_f32_t   fun),
        "use ggml_map_custom2 instead");

    GGML_DEPRECATED(GGML_API struct ggml_tensor * ggml_map_binary_inplace_f32(
            struct ggml_context         * ctx,
            struct ggml_tensor          * a,
            struct ggml_tensor          * b,
                   ggml_binary_op_f32_t   fun),
        "use ggml_map_custom2_inplace instead");

    GGML_DEPRECATED(GGML_API struct ggml_tensor * ggml_map_custom1_f32(
            struct ggml_context          * ctx,
            struct ggml_tensor           * a,
                   ggml_custom1_op_f32_t   fun),
        "use ggml_map_custom1 instead");

    GGML_DEPRECATED(GGML_API struct ggml_tensor * ggml_map_custom1_inplace_f32(
            struct ggml_context          * ctx,
            struct ggml_tensor           * a,
                   ggml_custom1_op_f32_t   fun),
        "use ggml_map_custom1_inplace instead");

    GGML_DEPRECATED(GGML_API struct ggml_tensor * ggml_map_custom2_f32(
            struct ggml_context          * ctx,
            struct ggml_tensor           * a,
            struct ggml_tensor           * b,
                   ggml_custom2_op_f32_t   fun),
        "use ggml_map_custom2 instead");

    GGML_DEPRECATED(GGML_API struct ggml_tensor * ggml_map_custom2_inplace_f32(
            struct ggml_context          * ctx,
            struct ggml_tensor           * a,
            struct ggml_tensor           * b,
                   ggml_custom2_op_f32_t   fun),
        "use ggml_map_custom2_inplace instead");

    GGML_DEPRECATED(GGML_API struct ggml_tensor * ggml_map_custom3_f32(
            struct ggml_context          * ctx,
            struct ggml_tensor           * a,
            struct ggml_tensor           * b,
            struct ggml_tensor           * c,
                   ggml_custom3_op_f32_t   fun),
        "use ggml_map_custom3 instead");

    GGML_DEPRECATED(GGML_API struct ggml_tensor * ggml_map_custom3_inplace_f32(
            struct ggml_context          * ctx,
            struct ggml_tensor           * a,
            struct ggml_tensor           * b,
            struct ggml_tensor           * c,
                   ggml_custom3_op_f32_t   fun),
        "use ggml_map_custom3_inplace instead");

    // custom operators v2

    typedef void (*ggml_custom1_op_t)(struct ggml_tensor * dst , const struct ggml_tensor * a, int ith, int nth, void * userdata);
    typedef void (*ggml_custom2_op_t)(struct ggml_tensor * dst , const struct ggml_tensor * a, const struct ggml_tensor * b, int ith, int nth, void * userdata);
    typedef void (*ggml_custom3_op_t)(struct ggml_tensor * dst , const struct ggml_tensor * a, const struct ggml_tensor * b, const struct ggml_tensor * c, int ith, int nth, void * userdata);

    #define GGML_N_TASKS_MAX -1

    GGML_API struct ggml_tensor * ggml_map_custom1(
            struct ggml_context   * ctx,
            struct ggml_tensor    * a,
            ggml_custom1_op_t       fun,
            int                     n_tasks,
            void                  * userdata);

    GGML_API struct ggml_tensor * ggml_map_custom1_inplace(
            struct ggml_context   * ctx,
            struct ggml_tensor    * a,
            ggml_custom1_op_t       fun,
            int                     n_tasks,
            void                  * userdata);

    GGML_API struct ggml_tensor * ggml_map_custom2(
            struct ggml_context   * ctx,
            struct ggml_tensor    * a,
            struct ggml_tensor    * b,
            ggml_custom2_op_t       fun,
            int                     n_tasks,
            void                  * userdata);

    GGML_API struct ggml_tensor * ggml_map_custom2_inplace(
            struct ggml_context   * ctx,
            struct ggml_tensor    * a,
            struct ggml_tensor    * b,
            ggml_custom2_op_t       fun,
            int                     n_tasks,
            void                  * userdata);

    GGML_API struct ggml_tensor * ggml_map_custom3(
            struct ggml_context   * ctx,
            struct ggml_tensor    * a,
            struct ggml_tensor    * b,
            struct ggml_tensor    * c,
            ggml_custom3_op_t       fun,
            int                     n_tasks,
            void                  * userdata);

    GGML_API struct ggml_tensor * ggml_map_custom3_inplace(
            struct ggml_context   * ctx,
            struct ggml_tensor    * a,
            struct ggml_tensor    * b,
            struct ggml_tensor    * c,
            ggml_custom3_op_t       fun,
            int                     n_tasks,
            void                  * userdata);

    // loss function

    GGML_API struct ggml_tensor * ggml_cross_entropy_loss(
            struct ggml_context         * ctx,
            struct ggml_tensor          * a,
            struct ggml_tensor          * b);

    GGML_API struct ggml_tensor * ggml_cross_entropy_loss_back(
            struct ggml_context         * ctx,
            struct ggml_tensor          * a,
            struct ggml_tensor          * b,
            struct ggml_tensor          * c);

    //
    // automatic differentiation
    //

    GGML_API void ggml_set_param(
            struct ggml_context * ctx,
            struct ggml_tensor  * tensor);


    GGML_API void ggml_build_forward_expand (struct ggml_cgraph * cgraph, struct ggml_tensor * tensor);
    GGML_API void ggml_build_backward_expand(struct ggml_context * ctx, struct ggml_cgraph * gf, struct ggml_cgraph * gb, bool keep);

    // graph allocation in a context
    GGML_API struct ggml_cgraph * ggml_new_graph         (struct ggml_context * ctx); // size = GGML_DEFAULT_GRAPH_SIZE, grads = false
    GGML_API struct ggml_cgraph * ggml_new_graph_custom  (struct ggml_context * ctx, size_t size, bool grads);
    GGML_API struct ggml_cgraph * ggml_graph_dup         (struct ggml_context * ctx, struct ggml_cgraph * cgraph);
    GGML_API struct ggml_cgraph * ggml_graph_view        (struct ggml_context * ctx, struct ggml_cgraph * cgraph, int i0, int i1);
    GGML_API void                 ggml_graph_cpy         (struct ggml_cgraph * src, struct ggml_cgraph * dst);
    GGML_API void                 ggml_graph_reset       (struct ggml_cgraph * cgraph);  // zero grads
    GGML_API void                 ggml_graph_clear       (struct ggml_cgraph * cgraph);

    GGML_API size_t ggml_graph_overhead(void);
    GGML_API size_t ggml_graph_overhead_custom(size_t size, bool grads);

    // ggml_graph_plan() has to be called before ggml_graph_compute()
    // when plan.work_size > 0, caller must allocate memory for plan.work_data
    GGML_API struct ggml_cplan ggml_graph_plan   (struct ggml_cgraph * cgraph, int n_threads /*= GGML_DEFAULT_N_THREADS*/);
    GGML_API int               ggml_graph_compute(struct ggml_cgraph * cgraph, struct ggml_cplan * cplan);

    // same as ggml_graph_compute() but the work data is allocated as a part of the context
    // note: the drawback of this API is that you must have ensured that the context has enough memory for the work data
    GGML_API void ggml_graph_compute_with_ctx(struct ggml_context * ctx, struct ggml_cgraph * cgraph, int n_threads);

    GGML_API struct ggml_tensor * ggml_graph_get_tensor(struct ggml_cgraph * cgraph, const char * name);

    GGML_API void                 ggml_graph_export(const struct ggml_cgraph * cgraph, const char * fname);
    GGML_API struct ggml_cgraph * ggml_graph_import(const char * fname, struct ggml_context ** ctx_data, struct ggml_context ** ctx_eval);

    // print info and performance information for the graph
    GGML_API void ggml_graph_print(const struct ggml_cgraph * cgraph);

    // dump the graph into a file using the dot format
    GGML_API void ggml_graph_dump_dot(const struct ggml_cgraph * gb, const struct ggml_cgraph * gf, const char * filename);

    // build gradient checkpointing backward graph gb for gf using provided checkpoints
    // gb_tmp will contain original backward graph with rewritten backward process nodes,
    // but without the second forward pass nodes.
    GGML_API void ggml_build_backward_gradient_checkpointing(
            struct ggml_context   * ctx,
            struct ggml_cgraph    * gf,
            struct ggml_cgraph    * gb,
            struct ggml_cgraph    * gb_tmp,
            struct ggml_tensor  * * checkpoints,
            int                     n_checkpoints);
    //
    // optimization
    //

    // optimization methods
    enum ggml_opt_type {
        GGML_OPT_ADAM,
        GGML_OPT_LBFGS,
    };

    // linesearch methods
    enum ggml_linesearch {
        GGML_LINESEARCH_DEFAULT = 1,

        GGML_LINESEARCH_BACKTRACKING_ARMIJO       = 0,
        GGML_LINESEARCH_BACKTRACKING_WOLFE        = 1,
        GGML_LINESEARCH_BACKTRACKING_STRONG_WOLFE = 2,
    };

    // optimization return values
    enum ggml_opt_result {
        GGML_OPT_OK = 0,
        GGML_OPT_DID_NOT_CONVERGE,
        GGML_OPT_NO_CONTEXT,
        GGML_OPT_INVALID_WOLFE,
        GGML_OPT_FAIL,
        GGML_OPT_CANCEL,

        GGML_LINESEARCH_FAIL = -128,
        GGML_LINESEARCH_MINIMUM_STEP,
        GGML_LINESEARCH_MAXIMUM_STEP,
        GGML_LINESEARCH_MAXIMUM_ITERATIONS,
        GGML_LINESEARCH_INVALID_PARAMETERS,
    };

    typedef void (*ggml_opt_callback)(void * data, int accum_step, float * sched, bool * cancel);
    typedef void (*ggml_log_callback)(enum ggml_log_level level, const char * text, void * user_data);

    // optimization parameters
    //
    //   see ggml.c (ggml_opt_default_params) for default values
    //
    struct ggml_opt_params {
        enum ggml_opt_type type;

        size_t graph_size;

        int n_threads;

        // delta-based convergence test
        //
        //   if past == 0 - disabled
        //   if past > 0:
        //     stop if |f(x) - f(x_past)| < delta * max(1, |f(x)|)
        //
        int past;
        float delta;

        // maximum number of iterations without improvement
        //
        //   if 0 - disabled
        //   if > 0:
        //     assume convergence if no cost improvement in this number of iterations
        //
        int max_no_improvement;

        bool print_forward_graph;
        bool print_backward_graph;

        int n_gradient_accumulation;

        // ADAM parameters
        struct {
            int n_iter;

            float sched; // schedule multiplier (fixed, decay or warmup)
            float decay; // weight decay for AdamW, use 0.0f to disable
            int   decay_min_ndim; // minimum number of tensor dimension to apply weight decay
            float alpha; // learning rate
            float beta1;
            float beta2;
            float eps;   // epsilon for numerical stability
            float eps_f; // epsilon for convergence test
            float eps_g; // epsilon for convergence test
            float gclip; // gradient clipping
        } adam;

        // LBFGS parameters
        struct {
            int m; // number of corrections to approximate the inv. Hessian
            int n_iter;
            int max_linesearch;

            float eps;      // convergence tolerance
            float ftol;     // line search tolerance
            float wolfe;
            float min_step;
            float max_step;

            enum ggml_linesearch linesearch;
        } lbfgs;
    };

    struct ggml_opt_context {
        struct ggml_context * ctx;
        struct ggml_opt_params params;

        int iter;
        int64_t nx; // number of parameter elements

        bool just_initialized;

        float loss_before;
        float loss_after;

        struct {
            struct ggml_tensor * g;  // current gradient
            struct ggml_tensor * m;  // first moment
            struct ggml_tensor * v;  // second moment
            struct ggml_tensor * pf; // past function values
            float fx_best;
            float fx_prev;
            int n_no_improvement;
        } adam;

        struct {
            struct ggml_tensor * x;    // current parameters
            struct ggml_tensor * xp;   // previous parameters
            struct ggml_tensor * g;    // current gradient
            struct ggml_tensor * gp;   // previous gradient
            struct ggml_tensor * d;    // search direction
            struct ggml_tensor * pf;   // past function values
            struct ggml_tensor * lmal; // the L-BFGS memory alpha
            struct ggml_tensor * lmys; // the L-BFGS memory ys
            struct ggml_tensor * lms;  // the L-BFGS memory s
            struct ggml_tensor * lmy;  // the L-BFGS memory y
            float fx_best;
            float step;
            int j;
            int k;
            int end;
            int n_no_improvement;
        } lbfgs;
    };

    GGML_API struct ggml_opt_params ggml_opt_default_params(enum ggml_opt_type type);

    // optimize the function defined by the tensor f
    GGML_API enum ggml_opt_result ggml_opt(
            struct ggml_context * ctx,
            struct ggml_opt_params params,
            struct ggml_tensor * f);

    // initialize optimizer context
    GGML_API void ggml_opt_init(
            struct ggml_context     * ctx,
            struct ggml_opt_context * opt,
            struct ggml_opt_params    params,
            int64_t                   nx);

    // continue optimizing the function defined by the tensor f
    GGML_API enum ggml_opt_result ggml_opt_resume(
            struct ggml_context * ctx,
            struct ggml_opt_context * opt,
            struct ggml_tensor * f);

    // continue optimizing the function defined by the tensor f
    GGML_API enum ggml_opt_result ggml_opt_resume_g(
            struct ggml_context * ctx,
            struct ggml_opt_context * opt,
            struct ggml_tensor * f,
            struct ggml_cgraph * gf,
            struct ggml_cgraph * gb,
            ggml_opt_callback callback,
            void * callback_data);

    //
    // quantization
    //

    // TODO: these would probably get removed in favor of the more general ggml_quantize_chunk
    GGML_API size_t ggml_quantize_q4_0(const float * src, void * dst, int n, int k, int64_t * hist);
    GGML_API size_t ggml_quantize_q4_1(const float * src, void * dst, int n, int k, int64_t * hist);
    GGML_API size_t ggml_quantize_q5_0(const float * src, void * dst, int n, int k, int64_t * hist);
    GGML_API size_t ggml_quantize_q5_1(const float * src, void * dst, int n, int k, int64_t * hist);
    GGML_API size_t ggml_quantize_q8_0(const float * src, void * dst, int n, int k, int64_t * hist);

    GGML_API size_t ggml_quantize_q2_K(const float * src, void * dst, int n, int k, int64_t * hist);
    GGML_API size_t ggml_quantize_q3_K(const float * src, void * dst, int n, int k, int64_t * hist);
    GGML_API size_t ggml_quantize_q4_K(const float * src, void * dst, int n, int k, int64_t * hist);
    GGML_API size_t ggml_quantize_q5_K(const float * src, void * dst, int n, int k, int64_t * hist);
    GGML_API size_t ggml_quantize_q6_K(const float * src, void * dst, int n, int k, int64_t * hist);

    GGML_API size_t ggml_quantize_chunk(enum ggml_type type, const float * src, void * dst, int start, int n, int64_t * hist);

    //
    // gguf
    //

    enum gguf_type {
        GGUF_TYPE_UINT8   = 0,
        GGUF_TYPE_INT8    = 1,
        GGUF_TYPE_UINT16  = 2,
        GGUF_TYPE_INT16   = 3,
        GGUF_TYPE_UINT32  = 4,
        GGUF_TYPE_INT32   = 5,
        GGUF_TYPE_FLOAT32 = 6,
        GGUF_TYPE_BOOL    = 7,
        GGUF_TYPE_STRING  = 8,
        GGUF_TYPE_ARRAY   = 9,
        GGUF_TYPE_UINT64  = 10,
        GGUF_TYPE_INT64   = 11,
        GGUF_TYPE_FLOAT64 = 12,
        GGUF_TYPE_COUNT,       // marks the end of the enum
    };

    struct gguf_context;

    struct gguf_init_params {
        bool no_alloc;

        // if not NULL, create a ggml_context and allocate the tensor data in it
        struct ggml_context ** ctx;
    };

    GGML_API struct gguf_context * gguf_init_empty(void);
    GGML_API struct gguf_context * gguf_init_from_file(const char * fname, struct gguf_init_params params);
    //GGML_API struct gguf_context * gguf_init_from_buffer(..);

    GGML_API void gguf_free(struct gguf_context * ctx);

    GGML_API const char * gguf_type_name(enum gguf_type type);

    GGML_API int    gguf_get_version    (const struct gguf_context * ctx);
    GGML_API size_t gguf_get_alignment  (const struct gguf_context * ctx);
    GGML_API size_t gguf_get_data_offset(const struct gguf_context * ctx);
    GGML_API void * gguf_get_data       (const struct gguf_context * ctx);

    GGML_API int          gguf_get_n_kv(const struct gguf_context * ctx);
    GGML_API int          gguf_find_key(const struct gguf_context * ctx, const char * key);
    GGML_API const char * gguf_get_key (const struct gguf_context * ctx, int key_id);

    GGML_API enum gguf_type gguf_get_kv_type (const struct gguf_context * ctx, int key_id);
    GGML_API enum gguf_type gguf_get_arr_type(const struct gguf_context * ctx, int key_id);

    // will abort if the wrong type is used for the key
    GGML_API uint8_t      gguf_get_val_u8  (const struct gguf_context * ctx, int key_id);
    GGML_API int8_t       gguf_get_val_i8  (const struct gguf_context * ctx, int key_id);
    GGML_API uint16_t     gguf_get_val_u16 (const struct gguf_context * ctx, int key_id);
    GGML_API int16_t      gguf_get_val_i16 (const struct gguf_context * ctx, int key_id);
    GGML_API uint32_t     gguf_get_val_u32 (const struct gguf_context * ctx, int key_id);
    GGML_API int32_t      gguf_get_val_i32 (const struct gguf_context * ctx, int key_id);
    GGML_API float        gguf_get_val_f32 (const struct gguf_context * ctx, int key_id);
    GGML_API uint64_t     gguf_get_val_u64 (const struct gguf_context * ctx, int key_id);
    GGML_API int64_t      gguf_get_val_i64 (const struct gguf_context * ctx, int key_id);
    GGML_API double       gguf_get_val_f64 (const struct gguf_context * ctx, int key_id);
    GGML_API bool         gguf_get_val_bool(const struct gguf_context * ctx, int key_id);
    GGML_API const char * gguf_get_val_str (const struct gguf_context * ctx, int key_id);
    GGML_API int          gguf_get_arr_n   (const struct gguf_context * ctx, int key_id);
    GGML_API const void * gguf_get_arr_data(const struct gguf_context * ctx, int key_id);
    GGML_API const char * gguf_get_arr_str (const struct gguf_context * ctx, int key_id, int i);

    GGML_API int    gguf_get_n_tensors    (const struct gguf_context * ctx);
    GGML_API int    gguf_find_tensor      (const struct gguf_context * ctx, const char * name);
    GGML_API size_t gguf_get_tensor_offset(const struct gguf_context * ctx, int i);
    GGML_API char * gguf_get_tensor_name  (const struct gguf_context * ctx, int i);

    // overrides existing values or adds a new one
    GGML_API void gguf_set_val_u8  (struct gguf_context * ctx, const char * key, uint8_t  val);
    GGML_API void gguf_set_val_i8  (struct gguf_context * ctx, const char * key, int8_t   val);
    GGML_API void gguf_set_val_u16 (struct gguf_context * ctx, const char * key, uint16_t val);
    GGML_API void gguf_set_val_i16 (struct gguf_context * ctx, const char * key, int16_t  val);
    GGML_API void gguf_set_val_u32 (struct gguf_context * ctx, const char * key, uint32_t val);
    GGML_API void gguf_set_val_i32 (struct gguf_context * ctx, const char * key, int32_t  val);
    GGML_API void gguf_set_val_f32 (struct gguf_context * ctx, const char * key, float    val);
    GGML_API void gguf_set_val_u64 (struct gguf_context * ctx, const char * key, uint64_t val);
    GGML_API void gguf_set_val_i64 (struct gguf_context * ctx, const char * key, int64_t  val);
    GGML_API void gguf_set_val_f64 (struct gguf_context * ctx, const char * key, double   val);
    GGML_API void gguf_set_val_bool(struct gguf_context * ctx, const char * key, bool     val);
    GGML_API void gguf_set_val_str (struct gguf_context * ctx, const char * key, const char * val);
    GGML_API void gguf_set_arr_data(struct gguf_context * ctx, const char * key, enum gguf_type type, const void * data, int n);
    GGML_API void gguf_set_arr_str (struct gguf_context * ctx, const char * key, const char ** data, int n);

    // set or add KV pairs from another context
    GGML_API void gguf_set_kv(struct gguf_context * ctx, struct gguf_context * src);

    // manage tensor info
    GGML_API void gguf_add_tensor(struct gguf_context * ctx, const struct ggml_tensor * tensor);
    GGML_API void gguf_set_tensor_type(struct gguf_context * ctx, const char * name, enum ggml_type type);
    GGML_API void gguf_set_tensor_data(struct gguf_context * ctx, const char * name, const void * data, size_t size);

    // writing gguf files can be done in 2 ways:
    //
    // - write the entire gguf_context to a binary file in a single pass:
    //
    //   gguf_write_to_file(ctx, fname);
    //
    // - first prepare a file with a placeholder for the meta data, write the tensor data, then write the meta data:
    //
    //   FILE * f = fopen(fname, "wb");
    //   fseek(f, gguf_get_meta_size(ctx), SEEK_SET);
    //   fwrite(f, ...);
    //   void * data = gguf_meta_get_meta_data(ctx);
    //   fseek(f, 0, SEEK_SET);
    //   fwrite(f, data, gguf_get_meta_size(ctx));
    //   free(data);
    //   fclose(f);
    //

    // write the entire context to a binary file
    GGML_API void gguf_write_to_file(const struct gguf_context * ctx, const char * fname, bool only_meta);

    // get the size in bytes of the meta data (header, kv pairs, tensor info) including padding
    GGML_API size_t gguf_get_meta_size(const struct gguf_context * ctx);
    GGML_API void   gguf_get_meta_data(const struct gguf_context * ctx, void * data);

    //
    // system info
    //

    GGML_API int ggml_cpu_has_avx        (void);
    GGML_API int ggml_cpu_has_avx2       (void);
    GGML_API int ggml_cpu_has_avx512     (void);
    GGML_API int ggml_cpu_has_avx512_vbmi(void);
    GGML_API int ggml_cpu_has_avx512_vnni(void);
    GGML_API int ggml_cpu_has_fma        (void);
    GGML_API int ggml_cpu_has_neon       (void);
    GGML_API int ggml_cpu_has_arm_fma    (void);
    GGML_API int ggml_cpu_has_metal      (void);
    GGML_API int ggml_cpu_has_f16c       (void);
    GGML_API int ggml_cpu_has_fp16_va    (void);
    GGML_API int ggml_cpu_has_wasm_simd  (void);
    GGML_API int ggml_cpu_has_blas       (void);
    GGML_API int ggml_cpu_has_cublas     (void);
    GGML_API int ggml_cpu_has_clblast    (void);
    GGML_API int ggml_cpu_has_gpublas    (void);
    GGML_API int ggml_cpu_has_sse3       (void);
    GGML_API int ggml_cpu_has_ssse3      (void);
    GGML_API int ggml_cpu_has_vsx        (void);

    //
    // Internal types and functions exposed for tests and benchmarks
    //

```

这段代码是一个C++语言的定义，用于定义一个名为"ggml_type_traits_t"的类型。这个类型定义了"ggml_to_float_t"函数和"ggml_from_float_t"函数以及"ggml_vec_dot_t"函数的签名。同时，通过#ifdef和#else进行条件编译。

具体来说，这段代码定义了一个"ggml_type_traits_t"类型，其中包含如下成员：

- "type_name"是一个字符串，用于指定该类型的名称。
- "blck_size"是一个整数，用于指定该类型每个成员的大小。
- "type_size"是一个整数，用于指定该类型每个成员所占用的字节数。
- "is_quantized"是一个布尔值，表示该类型是否已经量化。
- "to_float"是一个函数指针，指向可以将一个"ggml_vec_dot_t"类型的参数转换为浮点数的函数。
- "from_float"是一个函数指针，指向可以将一个浮点数转换为"ggml_vec_dot_t"类型的函数。
- "from_float_reference"是一个函数指针，指向可以转换为"ggml_vec_dot_t"类型并且该指针指向一个"ggml_type_traits_t"类型的函数。
- "vec_dot"是一个函数指针，指向可以对两个"ggml_vec_dot_t"类型的参数执行向量点积的函数。
- "vec_dot_type"是一个枚举类型，用于指定"vec_dot"函数使用的点积类型。

另外，该代码定义了一个名为"ggml_internal_get_type_traits"的函数，用于根据指定的ggml类型返回相应的类型签名。


```cpp
#ifdef  __cplusplus
// restrict not standard in C++
#define GGML_RESTRICT
#else
#define GGML_RESTRICT restrict
#endif
    typedef void (*ggml_to_float_t)  (const void  * GGML_RESTRICT x, float * GGML_RESTRICT y, int k);
    typedef void (*ggml_from_float_t)(const float * GGML_RESTRICT x, void  * GGML_RESTRICT y, int k);
    typedef void (*ggml_vec_dot_t)   (const int n, float * GGML_RESTRICT s, const void * GGML_RESTRICT x, const void * GGML_RESTRICT y);

    typedef struct {
        const char      * type_name;
        int               blck_size;
        size_t            type_size;
        bool              is_quantized;
        ggml_to_float_t   to_float;
        ggml_from_float_t from_float;
        ggml_from_float_t from_float_reference;
        ggml_vec_dot_t    vec_dot;
        enum ggml_type    vec_dot_type;
    } ggml_type_traits_t;

    GGML_API ggml_type_traits_t ggml_internal_get_type_traits(enum ggml_type type);

```

这段代码是一个条件编译语句，它检查是否定义了`__cplusplus`函数。如果没有定义该函数，则会编译为空字符串；如果已经定义，则会编译为`__cplusplus`返回类型。

简单来说，该代码会检查`__cplusplus`函数是否已经被定义，如果还没有定义，则会进行定义；如果已经定义，则编译时会使用已经定义的函数。


```cpp
#ifdef  __cplusplus
}
#endif

```