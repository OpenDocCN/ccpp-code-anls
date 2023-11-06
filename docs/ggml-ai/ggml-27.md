# GGML源码解析 27

# `tests/test-opt.cpp`

这段代码是一个 C++ 程序，它包含了三个头文件：ggml.h、<cmath> 和 <cstdio>。ggml.h 是 GGML（GNU Graphical髓媒体语言）的头文件，它定义了一些与该库有关的函数和变量。<cmath> 是数学标准库头文件，它包含了各种数学函数和常量。<cstdio> 是标准输入输出库头文件，它包含了各种输入输出函数。

接下来的 <cstdlib> 和 <cassert> 是标准库头文件，它们包含了一些通用的函数和判断条件。

该程序的作用是定义了一个名为 "test_ggml" 的函数，但并未定义它的实现。由于该函数被定义为带参数的函数，因此它接受的最大参数数量目前被限制为 2。

该函数的实现大致如下：

1. 引入必要的头文件和库函数。
2. 定义了一个名为 "test_ggml" 的函数，但没有为其指定具体的实现。
3. 在函数体中添加了一系列的注释，其中包含了一些代码提示，这些提示有助于开发者在代码中编写更好的代码。


```cpp
#include "ggml.h"

#include <cmath>
#include <cstdio>
#include <cstdlib>
#include <cassert>

#define MAX_NARGS 2

#if defined(__GNUC__)
#pragma GCC diagnostic ignored "-Wdouble-promotion"
#endif

//
// logging
```

这段代码定义了一系列条件编译指令，用于在满足某个条件时输出调试信息。其中，GGML_DEBUG是一个定义，表示是否开启了调试模式。如果GGML_DEBUG的值为1，那么就会开启调试模式，否则不会输出调试信息。

接下来的三个if语句用于定义更多的if语句，这些if语句都使用了相似的结构，只是其中的条件不同。这些条件都是关于GGML_DEBUG的，表示要输出调试信息的级别。如果GGML_DEBUG的值大于等于这些条件，那么就会输出调试信息，否则不会输出。

GGML_PRINT_DEBUG是一个定义，表示要输出的信息中是否包含调试信息。如果GGML_DEBUG的值大于等于1并且不等于GGML_DEBUG，那么就会输出调试信息。如果GGML_DEBUG的值大于等于5并且不等于GGML_DEBUG，那么就会输出调试信息。以此类推，可以定义更多的if语句，来定义更多的调试信息。

GGML_PRINT_DEBUG_5是一个定义，表示要输出的信息中是否包含调试信息。与GGML_PRINT_DEBUG类似，只是输出信息中不包括调试信息的级别更深。

整段代码的作用是定义了一系列条件编译指令，以确定在何种情况下输出调试信息。在满足某些条件时，就会输出调试信息，输出信息中包含的调试信息的级别取决于GGML_DEBUG的值。


```cpp
//
#define GGML_DEBUG 0
#if (GGML_DEBUG >= 1)
#define GGML_PRINT_DEBUG(...) printf(__VA_ARGS__)
#else
#define GGML_PRINT_DEBUG(...)
#endif

#if (GGML_DEBUG >= 5)
#define GGML_PRINT_DEBUG_5(...) printf(__VA_ARGS__)
#else
#define GGML_PRINT_DEBUG_5(...)
#endif

#if (GGML_DEBUG >= 10)
```

This function appears to be a simple implementation of a neural network with 3 input features and 1 output layer. It uses the random function `rand()` to generate random float numbers within a given range (`fmin` and `fmax`), and returns a neural network output object.

The function has a maximum number of dimensions (`ndims`), which is the number of input and output features. It is not clear how this number is set or used in the code.

The function is using a for loop structure to iterate over the input and output features. For each input feature, it generates a random number within the range `fmin` to `fmax` and adds it to the output.

It is not clear how this function is intended to be used in practice, or how it relates to the rest of the code in the neural network implementation. It may be helpful to have some context in order to understand how this function fits into the larger program.


```cpp
#define GGML_PRINT_DEBUG_10(...) printf(__VA_ARGS__)
#else
#define GGML_PRINT_DEBUG_10(...)
#endif

#define GGML_PRINT(...) printf(__VA_ARGS__)


static float frand(void) {
    return (float)rand()/(float)RAND_MAX;
}

static struct ggml_tensor * get_random_tensor(
    struct ggml_context * ctx0, int ndims, int64_t ne[], float fmin, float fmax
) {
    struct ggml_tensor * result = ggml_new_tensor(ctx0, GGML_TYPE_F32, ndims, ne);

    switch (ndims) {
        case 1:
            for (int i0 = 0; i0 < ne[0]; i0++) {
                ((float *)result->data)[i0] = frand()*(fmax - fmin) + fmin;
            }
            break;
        case 2:
            for (int i1 = 0; i1 < ne[1]; i1++) {
                for (int i0 = 0; i0 < ne[0]; i0++) {
                    ((float *)result->data)[i1*ne[0] + i0] = frand()*(fmax - fmin) + fmin;
                }
            }
            break;
        case 3:
            for (int i2 = 0; i2 < ne[2]; i2++) {
                for (int i1 = 0; i1 < ne[1]; i1++) {
                    for (int i0 = 0; i0 < ne[0]; i0++) {
                        ((float *)result->data)[i2*ne[1]*ne[0] + i1*ne[0] + i0] = frand()*(fmax - fmin) + fmin;
                    }
                }
            }
            break;
        case 4:
            for (int i3 = 0; i3 < ne[3]; i3++) {
                for (int i2 = 0; i2 < ne[2]; i2++) {
                    for (int i1 = 0; i1 < ne[1]; i1++) {
                        for (int i0 = 0; i0 < ne[0]; i0++) {
                            ((float *)result->data)[i3*ne[2]*ne[1]*ne[0] + i2*ne[1]*ne[0] + i1*ne[0] + i0] = frand()*(fmax - fmin) + fmin;
                        }
                    }
                }
            }
            break;
        default:
            assert(false);
    }

    return result;
}

```

This is a C function that uses the GGML (Graph-based Generative Model Library) framework to perform a simple operation on a randomly initialized tensor. The function takes two arguments: a random tensor with a shape of NxN, and a scalar value. It returns the result of the operation, or -1 if there was an error.

The function first initializes a random tensor of size NxN with a random value, and then performs a multiplication operation on this tensor with a scalar value. Finally, it computes the result of the multiplication using the forward pass of the graph, and returns the result.

The function uses the `ggml_mul_mat` function to perform the multiplication, and the `ggml_sub` and `ggml_sum` functions to perform the addition and sum operations, respectively. The `ggml_graph_compute_with_ctx` function is used to execute the computation on the graph, and the `ggml_get_f32_1d` function is used to extract the scalar value from the final tensor.

The function also includes some additional functionality to handle optimization. The optimization is performed using the `ggml_opt` function, which takes as input a set of optimization parameters (in this case, the Adam optimization algorithm), and returns the result of the optimization if it succeeds. If the optimization fails, the function returns -1.

Note that the `ggml_graph_compute_with_ctx` function is used here to handle the computation of the final result. However, in practice, this function should be avoided as it can cause performance issues due to the overhead of the computation on the graph. A better approach in this case would be to perform the computation on the host (CPU) rather than the GPU.


```cpp
int main(void) {
    struct ggml_init_params params = {
        /* .mem_size   = */ 1024*1024*1024,
        /* .mem_buffer = */ NULL,
        /* .no_alloc   = */ false,
    };

    struct ggml_context * ctx = ggml_init(params);

    int64_t ne1[4] = {4, 128, 1, 1};
    int64_t ne2[4] = {4, 256, 1, 1};
    int64_t ne3[4] = {128, 256, 1, 1};

    struct ggml_tensor * a = get_random_tensor(ctx, 2, ne1, -1, +1);
    struct ggml_tensor * b = get_random_tensor(ctx, 2, ne2, -1, +1);
    ggml_set_param(ctx, a);
    ggml_set_param(ctx, b);

    struct ggml_tensor * c = get_random_tensor(ctx, 2, ne3, -1, +1);

    struct ggml_tensor * ab = ggml_mul_mat(ctx, a, b);
    struct ggml_tensor * d  = ggml_sub(ctx, c, ab);
    struct ggml_tensor * e  = ggml_sum(ctx, ggml_sqr(ctx, d));

    struct ggml_cgraph * ge = ggml_new_graph_custom(ctx, GGML_DEFAULT_GRAPH_SIZE, true);
    ggml_build_forward_expand(ge, e);
    ggml_graph_reset(ge);

    ggml_graph_compute_with_ctx(ctx, ge, /*n_threads*/ 1);

    const float fe = ggml_get_f32_1d(e, 0);
    printf("%s: e = %.4f\n", __func__, fe);

    struct ggml_opt_params opt_params = ggml_opt_default_params(GGML_OPT_ADAM);

    ggml_opt(ctx, opt_params, e);

    ggml_graph_reset(ge);

    ggml_graph_compute_with_ctx(ctx, ge, /*n_threads*/ 1);

    const float fe_opt = ggml_get_f32_1d(e, 0);
    printf("%s: original  e = %.4f\n", __func__, fe);
    printf("%s: optimized e = %.4f\n", __func__, fe_opt);

    const bool success = (fe_opt <= fe);
    assert(success);

    ggml_free(ctx);
    return success ? 0 : -1;
}
```



这段代码定义了三个整型变量ne1、ne2和ne3，分别包含四个整型数值。这三个变量是同一个数组的副本，这个数组在内存中是连续的。

在主函数中，首先定义了一个整型变量e，它的初始值是25890.9375。然后，使用三个变量ne1、ne2和ne3来初始化e数组的四个元素，分别赋值为8、128、1和1。这样，e数组的值和ne1数组的值是相同的。

接下来，使用原始的ne1数组，而不是优化过的ne1数组。在主函数中，定义了一个整型变量optimized_e，它的初始值是优化过的ne1数组的值，即25890.9375。

最后，在主函数中，定义了一个整型变量original_e，它的初始值是未优化的ne1数组的值，即8、128、1和1。


```cpp
// int64_t ne1[4] = {4, 128, 1, 1};
// int64_t ne2[4] = {4, 256, 1, 1};;
// int64_t ne3[4] = {128, 256, 1, 1};
// main: original  e = 25890.9375
// main: optimized e = 10094.7031

// int64_t ne1[4] = {8, 128, 1, 1};
// int64_t ne2[4] = {8, 256, 1, 1};;
// int64_t ne3[4] = {128, 256, 1, 1};
// main: original  e = 39429.5078
// main: optimized e = 9275.8936

// int64_t ne1[4] = {16, 128, 1, 1};
// int64_t ne2[4] = {16, 256, 1, 1};;
// int64_t ne3[4] = {128, 256, 1, 1};
```



以上代码定义了两个整型变量ne1和ne2，分别包含四个整型元素，然后定义了两个整型变量ne3，也包含四个整型元素。接着在main函数中，分别给定了原始值e1和优化值e2，分别存储了原始数据类型下的值：68371.1328和5451.0166。 

然后，分别对ne1、ne2和ne3数组进行了多次赋值，将原始值存储在了变量中，使得变量可以被后续计算使用。


```cpp
// main: original  e = 68371.1328
// main: optimized e = 7854.4502


// int64_t ne1[4] = {32, 128, 1, 1};
// int64_t ne2[4] = {32, 256, 1, 1};;
// int64_t ne3[4] = {128, 256, 1, 1};
// main: original  e = 126061.1953
// main: optimized e = 5451.0166

// int64_t ne1[4] = {4, 1024, 1, 1};
// int64_t ne2[4] = {4, 2048, 1, 1};;
// int64_t ne3[4] = {1024, 2048, 1, 1};
// main: original  e = 1620817.8750
// main: optimized e = 698387.6875

```

这段代码定义了两个数组 `ne1` 和 `ne2`，每个数组包含四个整数 `int64_t` 类型的变量。这两个数组在程序中都被初始化为 {4, 1024, 1, 1} 和 {32, 2048, 1, 1}，然后定义了一个主函数 `main`。

在主函数 `main` 中，首先定义了两个变量 `e`，一个原始的 `int64_t` 类型的变量，另一个是经过优化的 `int64_t` 类型的变量。这两个变量的值分别是 `1629595.6250` 和 `651119.1250`。

然后，在主函数中创建了三个数组 `ne1`、`ne2` 和 `ne3`，它们的元素个数和类型都与原始的数组相同。这些数组在程序中也都被初始化为 `{32, 1024, 1, 1}` 和 `{32, 2048, 1, 1}`。


```cpp
// another run on M1
// int64_t ne1[4] = {4, 1024, 1, 1};
// int64_t ne2[4] = {4, 2048, 1, 1};;
// int64_t ne3[4] = {1024, 2048, 1, 1};
// main: original  e = 1629595.6250
// main: optimized e = 698169.1250

// int64_t ne1[4] = {32, 1024, 1, 1};
// int64_t ne2[4] = {32, 2048, 1, 1};;
// int64_t ne3[4] = {1024, 2048, 1, 1};
// main: original  e = 8146770.5000
// main: optimized e = 651119.1250

```

# `tests/test-pool.c`

This is a C function that uses the ML iris package to perform a operation on a 3D tensor represented as a matrix of 3浮点数. The tensor is represented as a pooled 2D matrix, with each element of the tensor stored in a 3D array. The 3D array is then expanded forward in the graph, and the resulting graph is used to compute the output of the tensor. The function returns 0 on success and error on failure.

Here is a high-level description of the function:

1. Create a new 3D tensor with the specified data and dimensions.
2. Create a new 2D tensor with the same data and dimensions, but with a single element in each dimension.
3. Create a new 2D tensor with the same data and dimensions, but with a single element in each dimension.
4. Create a new graph of the specified type using the `ggml_new_graph()` function.
5. Create a function pointer to the graph and a function handle.
6. Feed the input tensor through the graph and store the output in the output tensor.
7. Create a function pointer to the graph and a handle to the input tensor.
8. Feed the input tensor through the graph and store the output in the output tensor.
9. Create a function pointer to the graph and store the input tensor.
10. Create a function pointer to the graph and store the input tensor.
11. Create a function pointer to the graph and store the input tensor.
12. Create a function pointer to the graph and store the input tensor.
13. Create a function pointer to the graph and store the input tensor.
14. Create a function pointer to the graph and store the input tensor.
15. Create a function pointer to the graph and store the input tensor.
16. Create a function pointer to the graph and store the input tensor.
17. Create a function pointer to the graph and store the input tensor.
18. Create a function pointer to the graph and store the input tensor.
19. Create a function pointer to the graph and store the input tensor.
20. Create a function pointer to the graph and store the input tensor.
21. Create a function pointer to the graph and store the input tensor.
22. Create a function pointer to the graph and store the input tensor.
23. Create a function pointer to the graph and store the input tensor.
24. Create a function pointer to the graph and store the input tensor.
25. Create a function pointer to the graph and store the input tensor.


```cpp
#include "ggml/ggml.h"

#include <string.h>
#include <stdio.h>
#include <stdlib.h>

struct ggml_context* make_ctx(void) {
    struct ggml_init_params params = {
        .mem_size = 2 * 1024 * 1024,
    };

    return ggml_init(params);
}

int main(int argc, const char** argv) {

    float buf_f32[1024];
    for (int i = 0; i < 1024; ++i) {
        buf_f32[i] = (float)(i + 1);
    }

    // avg pool 1d
    {
        struct ggml_context * ctx = make_ctx();
        struct ggml_tensor * t = ggml_new_tensor_2d(ctx, GGML_TYPE_F32, 10, 2);
        memcpy(t->data, buf_f32, ggml_nbytes(t));

        struct ggml_tensor * t_pooled = ggml_pool_1d(ctx, t, GGML_OP_POOL_AVG, 3, 3, 0);
        GGML_ASSERT(t_pooled->ne[0] == 3);
        GGML_ASSERT(t_pooled->ne[1] == 2);
        GGML_ASSERT(t_pooled->ne[2] == 1);

        struct ggml_cgraph * graph = ggml_new_graph(ctx);
        ggml_build_forward_expand(graph, t_pooled);

        ggml_graph_compute_with_ctx(ctx, graph, 4);

        const float * output = ggml_get_data_f32(t_pooled);

        GGML_ASSERT(output[0] == 2);
        GGML_ASSERT(output[1] == 5);
        GGML_ASSERT(output[2] == 8);
        GGML_ASSERT(output[3] == 12);
        GGML_ASSERT(output[4] == 15);
        GGML_ASSERT(output[5] == 18);

        ggml_free(ctx);
    }

    // max pool 1d
    {
        struct ggml_context * ctx = make_ctx();
        struct ggml_tensor * t = ggml_new_tensor_2d(ctx, GGML_TYPE_F32, 10, 2);
        memcpy(t->data, buf_f32, ggml_nbytes(t));

        struct ggml_tensor * t_pooled = ggml_pool_1d(ctx, t, GGML_OP_POOL_MAX, 3, 3, 0);
        GGML_ASSERT(t_pooled->ne[0] == 3);
        GGML_ASSERT(t_pooled->ne[1] == 2);
        GGML_ASSERT(t_pooled->ne[2] == 1);

        struct ggml_cgraph * graph = ggml_new_graph(ctx);
        ggml_build_forward_expand(graph, t_pooled);

        ggml_graph_compute_with_ctx(ctx, graph, 4);

        const float * output = ggml_get_data_f32(t_pooled);
        GGML_ASSERT(output[0] == 3);
        GGML_ASSERT(output[1] == 6);
        GGML_ASSERT(output[2] == 9);
        GGML_ASSERT(output[3] == 13);
        GGML_ASSERT(output[4] == 16);
        GGML_ASSERT(output[5] == 19);

        ggml_free(ctx);
    }

    // avg pool 2d
    {
        struct ggml_context * ctx = make_ctx();
        struct ggml_tensor * t = ggml_new_tensor_3d(ctx, GGML_TYPE_F32, 10, 10, 2);
        memcpy(t->data, buf_f32, ggml_nbytes(t));

        struct ggml_tensor * t_pooled = ggml_pool_2d(ctx, t, GGML_OP_POOL_AVG, 3, 4, 3, 4, 0, 0);
        GGML_ASSERT(t_pooled->ne[0] == 3);
        GGML_ASSERT(t_pooled->ne[1] == 2);
        GGML_ASSERT(t_pooled->ne[2] == 2);
        GGML_ASSERT(t_pooled->ne[3] == 1);

        struct ggml_cgraph * graph = ggml_new_graph(ctx);
        ggml_build_forward_expand(graph, t_pooled);

        ggml_graph_compute_with_ctx(ctx, graph, 4);

        const float * output = ggml_get_data_f32(t_pooled);
        GGML_ASSERT(output[0] == 17);
        GGML_ASSERT(output[1] == 20);
        GGML_ASSERT(output[2] == 23);
        GGML_ASSERT(output[3] == 57);
        GGML_ASSERT(output[4] == 60);
        GGML_ASSERT(output[5] == 63);
        GGML_ASSERT(output[6] == 117);
        GGML_ASSERT(output[7] == 120);
        GGML_ASSERT(output[8] == 123);
        GGML_ASSERT(output[9] == 157);
        GGML_ASSERT(output[10] == 160);
        GGML_ASSERT(output[11] == 163);


        ggml_free(ctx);
    }

    // max pool 2d
    {
        struct ggml_context * ctx = make_ctx();
        struct ggml_tensor * t = ggml_new_tensor_3d(ctx, GGML_TYPE_F32, 10, 10, 2);
        memcpy(t->data, buf_f32, ggml_nbytes(t));

        struct ggml_tensor * t_pooled = ggml_pool_2d(ctx, t, GGML_OP_POOL_MAX, 3, 4, 3, 4, 0, 0);
        GGML_ASSERT(t_pooled->ne[0] == 3);
        GGML_ASSERT(t_pooled->ne[1] == 2);
        GGML_ASSERT(t_pooled->ne[2] == 2);
        GGML_ASSERT(t_pooled->ne[3] == 1);

        struct ggml_cgraph * graph = ggml_new_graph(ctx);
        ggml_build_forward_expand(graph, t_pooled);

        ggml_graph_compute_with_ctx(ctx, graph, 4);

        const float * output = ggml_get_data_f32(t_pooled);
        GGML_ASSERT(output[0] == 33);
        GGML_ASSERT(output[1] == 36);
        GGML_ASSERT(output[2] == 39);
        GGML_ASSERT(output[3] == 73);
        GGML_ASSERT(output[4] == 76);
        GGML_ASSERT(output[5] == 79);
        GGML_ASSERT(output[6] == 133);
        GGML_ASSERT(output[7] == 136);
        GGML_ASSERT(output[8] == 139);
        GGML_ASSERT(output[9] == 173);
        GGML_ASSERT(output[10] == 176);
        GGML_ASSERT(output[11] == 179);

        ggml_free(ctx);
    }

    return 0;
}

```

# `tests/test-quantize-fns.cpp`

这段代码是一个C语言编写的单元测试用例，用于测试某些与量化（quantization）相关的函数。具体来说，这段代码包括以下三个函数：

1. `quantize`：对一个大的浮点数进行量化，使其变成一个较小的浮点数。
2. `dequantize`：将一个较小的浮点数还原成一个大的浮点数。
3. `dot product`：计算两个浮点数之间的点积（dot product）。

函数的实现可能如下：
```cppc
#include "ggml.h"
#include <assert.h>
#include <math.h>
#include <stdio.h>
#include <string>
#include <vector>
```

由于缺乏具体的函数实现，我无法给出更具体的解释。但通常情况下，这段代码会包含在量化库的支持函数中，用于确保在需要时进行量化操作。


```cpp
// Unit tests for quantization specific functions - quantize, dequantize and dot product

#include "ggml.h"

#undef NDEBUG
#include <assert.h>
#include <math.h>
#include <stdio.h>
#include <string>
#include <vector>

#if defined(_MSC_VER)
#pragma warning(disable: 4244 4267) // possible loss of data
#endif

```

这段代码定义了4个常量float类型，分别为MAX_QUANTIZATION_REFERENCE_ERROR，MAX_QUANTIZATION_TOTAL_ERROR，MAX_QUANTIZATION_TOTAL_ERROR_2BITS，MAX_QUANTIZATION_TOTAL_ERROR_3BITS和MAX_DOT_PRODUCT_ERROR。

MAX_QUANTIZATION_REFERENCE_ERROR表示参考电压的最大允许误差，MAX_QUANTIZATION_TOTAL_ERROR表示合成过程的总误差允许的最大值，MAX_QUANTIZATION_TOTAL_ERROR_2BITS和MAX_QUANTIZATION_TOTAL_ERROR_3BITS分别表示2位和3位量化总的允许误差，MAX_DOT_PRODUCT_ERROR表示点生产生的误差允许的最大值。

此外，还定义了一个静态字符串数组RESULT_STR[]，用于存储合成结果成功或失败的描述。

最后，还有一段代码generate_data()，用于生成合成数据的临时变量。


```cpp
constexpr float MAX_QUANTIZATION_REFERENCE_ERROR = 0.0001f;
constexpr float MAX_QUANTIZATION_TOTAL_ERROR = 0.002f;
constexpr float MAX_QUANTIZATION_TOTAL_ERROR_2BITS = 0.0075f;
constexpr float MAX_QUANTIZATION_TOTAL_ERROR_3BITS = 0.0040f;
constexpr float MAX_DOT_PRODUCT_ERROR = 0.02f;

static const char* RESULT_STR[] = {"ok", "FAILED"};


// Generate synthetic data
static void generate_data(float offset, size_t n, float * dst) {
    for (size_t i = 0; i < n; i++) {
        dst[i] = 0.1 + 2*cosf(i + offset);
    }
}

```

这两段代码定义了一个名为“array_rmse”的函数和一个名为“total_quantization_error”的函数。

函数“array_rmse”接收两个float数组a1和a2以及数组长度n作为输入参数，然后计算这两个数组之间的相对误差（RMSE）。函数实现中，通过遍历数组a1和a2，计算每个元素之间的差异并累加，最后返回sqrtf（即sqrt）除以n。这个函数可以看做是一个将两个数组之间的差距计算成平方并求平方根的过程。

函数“total_quantization_error”接收一个ggml_type_traits_t类型的数据结构、测试数据大小（size_t类型的整数）以及两个float数组test_data和test_data的副本（相同大小但元素互为相反数）作为输入参数。函数实现中，首先将test_data和test_data的副本赋值给两个uint8_t类型的临时变量tmp_q和tmp_out，然后分别将qfns.from_float（test_data，tmp_q.data，test_size）和qfns.to_float（tmp_q.data，tmp_out.data，test_size）函数分别传入，最后将array_rmse函数的输入参数test_data和输出参数tmp_out.data作为参数传入，并返回array_rmse的结果。函数实现中，通过qfns.from_float和qfns.to_float函数，将test_data转换成float类型，然后通过array_rmse函数计算得到一个double类型的结果，最后将其除以test_size并获取结果，这样就可以得到总的量化误差。


```cpp
// Calculate RMSE between two float arrays
static float array_rmse(const float * a1, const float * a2, size_t n) {
    double sum = 0;
    for (size_t i = 0; i < n; i++) {
        double diff = a1[i] - a2[i];
        sum += diff * diff;
    }
    return sqrtf(sum) / n;
}

// Total quantization error on test data
static float total_quantization_error(ggml_type_traits_t & qfns, size_t test_size, const float * test_data) {
    std::vector<uint8_t> tmp_q(2*test_size);
    std::vector<float> tmp_out(test_size);

    qfns.from_float(test_data, tmp_q.data(), test_size);
    qfns.to_float(tmp_q.data(), tmp_out.data(), test_size);
    return array_rmse(test_data, tmp_out.data(), test_size);
}

```

这段代码定义了一个名为 "reference_quantization_error" 的函数，属于 "ggml_type_traits_t" 类型。它接受三个参数：

- "qfns"：一个 "ggml_type_traits_t" 类型的变量，用于存储输入数据。
- "test_size"：一个整数类型的参数，用于表示测试数据的大小。
- "test_data"：一个float类型的参数，用于存储输入数据的副本。

函数内部首先对输入数据 "test_data" 进行量化，然后使用 "qfns" 对象的 "from_float" 和 "to_float" 方法将量化后的数据转换为输出数据 "tmp_out"。

接着，对 "test_data" 再次进行量化，并使用 "qfns" 对象的 "from_float_reference" 方法将参考数据 "tmp_q_ref" 传入，将结果存储在 "tmp_out_ref" 变量中。

最后，函数返回 "tmp_out.data()" 和 "tmp_out_ref.data()" 中的数据的方差，作为测试数据与参考数据之间的量化误差。


```cpp
// Total quantization error on test data
static float reference_quantization_error(ggml_type_traits_t & qfns, size_t test_size, const float * test_data) {
    std::vector<uint8_t> tmp_q(2*test_size);
    std::vector<float> tmp_out(test_size);
    std::vector<float> tmp_out_ref(test_size);

    qfns.from_float(test_data, tmp_q.data(), test_size);
    qfns.to_float(tmp_q.data(), tmp_out.data(), test_size);

    qfns.from_float_reference(test_data, tmp_q.data(), test_size);
    qfns.to_float(tmp_q.data(), tmp_out_ref.data(), test_size);

    return array_rmse(tmp_out.data(), tmp_out_ref.data(), test_size);
}

```

这两段代码定义了一个名为 dot_product 的函数和一个名为 dot_product_error 的函数。它们都是接受两个标量的指针，大小为 dot_product 函数的参数个数。

dot_product 函数接收两个浮点数数组 a1 和 a2，并通过循环将它们的内容相乘并累加到一个名为 sum 的变量中。它然后返回 sum。

dot_product_error 函数接收两个浮点数数组 test_data1 和 test_data2，以及 dot_product 函数的输出，它使用 dot_product 函数计算两个数组的点积，然后计算这个点积与 dot_product 函数输出的差值。它将这个差值除以测试大小，并将结果存储在名为 result 的变量中。它返回这个差值。


```cpp
static float dot_product(const float * a1, const float * a2, size_t test_size) {
    double sum = 0;
    for (size_t i = 0; i < test_size; i++) {
        sum += a1[i] * a2[i];
    }
    return sum;
}

// Total dot product error
static float dot_product_error(
    ggml_type_traits_t & qfns, size_t test_size, const float * test_data1, const float *test_data2
) {
    std::vector<uint8_t> tmp_q1(2*test_size);
    std::vector<uint8_t> tmp_q2(2*test_size);

    auto vdot = ggml_internal_get_type_traits(qfns.vec_dot_type);

    qfns.from_float(test_data1, tmp_q1.data(), test_size);
    vdot.from_float(test_data2, tmp_q2.data(), test_size);

    float result = INFINITY;
    qfns.vec_dot(test_size, &result, tmp_q1.data(), tmp_q2.data());

    const float dot_ref = dot_product(test_data1, test_data2, test_size);

    return fabsf(result - dot_ref) / test_size;
}

```

This is a function that reads test data and measures the total error of a quantized implementation. The function takes a pointer to a QFN object, a test size, and a pointer to the test data.

The function first initializes the reference implementation error and the reference data error to zero. It then reads the test data and calculates the total error based on the type of the QFN and the quality level of the data.

If the QFN is of the Q2 or Q3 type, the function uses the appropriate formula to calculate the total error based on the number of bits used for quantization.

The function checks whether the total error is less than the maximum error allowed and prints an error message if it is not. It also prints the number of tests that failed.

If the QFN is of the vector-dot-product type, the function reads the test data and calculates the dot product error based on the QFN.

The function returns true if more than one test failed and false otherwise. The function also prints a failure message if the QFN is of the vector-dot-product type and the dot product error is not less than the maximum error allowed.


```cpp
int main(int argc, char * argv[]) {
    bool verbose = false;
    const size_t test_size = 32 * 128;

    std::string arg;
    for (int i = 1; i < argc; i++) {
        arg = argv[i];

        if (arg == "-v") {
            verbose = true;
        } else {
            fprintf(stderr, "error: unknown argument: %s\n", arg.c_str());
            return 1;
        }
    }

    std::vector<float> test_data(test_size);
    std::vector<float> test_data2(test_size);

    generate_data(0.0, test_data.size(), test_data.data());
    generate_data(1.0, test_data2.size(), test_data2.data());

    // Initialize GGML, ensures float conversion tables are initialized
    struct ggml_init_params ggml_params = {
        /* .mem_size   = */ 1*1024,
        /* .mem_buffer = */ NULL,
        /* .no_alloc   = */ true,
    };
    struct ggml_context * ctx = ggml_init(ggml_params);

    int num_failed = 0;
    bool failed = false;

    for (int i = 0; i < GGML_TYPE_COUNT; i++) {
        ggml_type type = (ggml_type) i;
        ggml_type_traits_t qfns = ggml_internal_get_type_traits(type);

        // deprecated - skip
        if (qfns.blck_size == 0) {
            continue;
        }

        printf("Testing %s\n", ggml_type_name((ggml_type) i));

        if (qfns.from_float && qfns.to_float) {
            const float total_error = total_quantization_error(qfns, test_size, test_data.data());
            const float max_quantization_error =
                type == GGML_TYPE_Q2_K ? MAX_QUANTIZATION_TOTAL_ERROR_2BITS :
                type == GGML_TYPE_Q3_K ? MAX_QUANTIZATION_TOTAL_ERROR_3BITS : MAX_QUANTIZATION_TOTAL_ERROR;
            failed = !(total_error < max_quantization_error);
            num_failed += failed;
            if (failed || verbose) {
                printf("%5s absolute quantization error:    %s (%f)\n", ggml_type_name(type), RESULT_STR[failed], total_error);
            }

            const float reference_error = reference_quantization_error(qfns, test_size, test_data.data());
            failed = !(reference_error < MAX_QUANTIZATION_REFERENCE_ERROR);
            num_failed += failed;
            if (failed || verbose) {
                printf("%5s reference implementation error: %s (%f)\n", ggml_type_name(type), RESULT_STR[failed], reference_error);
            }

            const float vec_dot_error = dot_product_error(qfns, test_size, test_data.data(), test_data2.data());
            failed = !(vec_dot_error < MAX_DOT_PRODUCT_ERROR);
            num_failed += failed;
            if (failed || verbose) {
                printf("%5s dot product error:              %s (%f)\n", ggml_type_name(type), RESULT_STR[failed], vec_dot_error);
            }
        }
    }

    if (num_failed || verbose) {
        printf("%d tests failed\n", num_failed);
    }

    ggml_free(ctx);

    return num_failed > 0;
}

```

# `tests/test-quantize-perf.cpp`

这段代码是一个用于基准量化的代码，主要作用是实现对 synthetic（模拟）数据的量化。Benchmark quantization specific functions on synthetic data 是它的名字，意味着这段代码主要针对模拟数据进行量化特定功能的测试。

在这段注释中，开发人员说明了他们的工作旨在基准化在模拟数据上实现特定功能的函数。因此，可以认为这个代码库是一个基准库，用于测试和评估其他代码库或算法在模拟数据上的效果。

这段代码的具体作用可能还涉及其他内容，但它是为了解决特定问题的基准库。


```cpp
// Benchmark quantization specific functions on synthetic data

#include "ggml.h"

#undef NDEBUG
#include <algorithm>
#include <assert.h>
#include <functional>
#include <inttypes.h>
#include <math.h>
#include <memory>
#include <stdio.h>
#include <string>
#include <vector>

```

这段代码定义了一系列头文件和常量，用于定义和控制向量操作中的参数和值。

首先，它检查定义的大气模型是否为 _MSC_VER，如果是，就使用 pragma 声明一个名为 "disable" 的 warning，通知编译器该特定警告可能存在但不会输出详细信息。

接下来，定义了一些常量，包括 MAX_ALIGNMENT、QK、WARMUP、ITERATIONS 和 MAX_ITERATIONS，这些参数会在程序中用于控制向量操作中的迭代次数、最大向量长度和向量的对齐。

然后，定义了一系列头文件，包括 stdint.h、math.h 和 cmplex.h，这些头文件提供了向量操作中所需的函数和宏定义。

接下来，定义了一些变量和常量，包括 L1_SIZE、L2_SIZE、L3_SIZE 和 MEM_SIZE，这些变量将用于存储向量的实际大小。

整个函数的作用是定义了一些参数和常量，以及一些头文件和函数，用于在程序中执行向量操作。


```cpp
#if defined(_MSC_VER)
#pragma warning(disable: 4244 4267) // possible loss of data
#endif

#define MAX_ALIGNMENT 64
#define QK 32
#define WARMUP 5
#define ITERATIONS 10
#define MAX_ITERATIONS 100000000

#define L1_SIZE      32*128
#define L2_SIZE     32*2048
#define L3_SIZE    32*20480
#define MEM_SIZE 32*2048000

```

这是一个结构体，名为“quantize_perf_params”，它的成员包括：

1. “include_types”是一个只读的向量，包含了多个字符串类型的变量，这些变量是“包括”在当前结构体中的类型。
2. “test_sizes”是一个只读的向量，包含了多个整数类型的变量，这些变量是用于测试的负载大小。
3. “alignment_offset”是一个只读的整数类型变量，它存储了向量“alignment_offset”在内存中的位置，如果没有定义，则默认值为0。
4. “op_quantize_row_q_reference”是一个只读的布尔类型变量，表示是否引用行级量化（row-level quantization）的参考值。
5. “op_quantize_row_q”是一个只读的布尔类型变量，表示是否量化行级（row-level）的Q值。
6. “op_dequantize_row_q”是一个只读的布尔类型变量，表示是否反量化行级（row-level）的Q值。
7. “op_quantize_row_q_dot”是一个只读的布尔类型变量，表示是否对每个行级（row-level）的Q值进行点积（dot product）运算。
8. “op_vec_dot_q”是一个只读的布尔类型变量，表示是否对每个列级（column-level）的Q值进行点积（dot product）运算。
9. “iterations”是一个只读的整数类型变量，表示对多少个行级（row-level）和列级（column-level）进行量化处理。

此外，还包括一些成员函数，如“initialize”用于初始化结构体，“deinitialize”用于反初始化结构体，“quantize_row”用于量化行级（row-level）的Q值，“dequantize_row”用于反量化行级（row-level）的Q值，以及“quantize_perf”用于对整个数据集进行量化处理。


```cpp
struct quantize_perf_params {
    std::vector<std::string> include_types;
    std::vector<size_t> test_sizes;
    size_t alignment_offset = 0;
    bool op_quantize_row_q_reference = false;
    bool op_quantize_row_q = false;
    bool op_dequantize_row_q = false;
    bool op_quantize_row_q_dot = false;
    bool op_vec_dot_q = false;
    int64_t iterations = ITERATIONS;
};

#if defined(__x86_64__) || defined(__i386__)

#include <x86intrin.h>
```

这段代码是一个内联函数 `cpu_cycles()`，用于检测计算机的CPU  cycles 计数器是否支持 64 位计数。如果没有支持，函数将返回 0。

函数体中包含两行代码，第一行是在某些编译器中检测 CPU 是否支持 64 位计数。第二行是在大多数编译器中定义的常量，表示支持 64 位计数。

具体来说，如果计算机的 CPU 支持 64 位计数，那么第一行代码将返回 CPU 当前的计数器值，也就是 64 位计数器的值。否则，第二行代码将返回 0。

这个函数可以被用来检测一台计算机是否支持 64 位计数，然后在代码中根据这个结果做出相应的调整。


```cpp
inline int64_t cpu_cycles() {
// Rough way to detect new-ish CPUs
#ifdef __POPCNT__
    unsigned int dummy;
    return __rdtscp(&dummy);
#else
    return __rdtsc();
#endif
}

#else

#define cpu_cycles() 0

#endif


```

这段代码的主要作用是生成一些 synthetic（模拟）数据。它包含三个函数，分别用于生成指定数量级的随机数、计算每秒字节数所需的时间，以及将生成的随机数对准指定的偏移量。

1. `generate_data`函数：这个函数的主要作用是生成指定数量级的随机数。它的参数包括一个offset值，一个数据大小（n）和一个数据指针（dst）。函数内部使用`cosf`函数生成一个在指定偏移量范围内（0到2π）的随机数，然后将这个随机数累加到dst数组中。n参数表示数据的大小，dst参数指向的是数据指针，指向的内存区域需要被分配。

2. `gigabytes_per_second`函数：这个函数的作用是计算每秒字节数（GB）。它接收两个参数：一个字节数（bytes）和一个持续时间（usecs）。函数内部将bytes字节数除以usecs乘以1000000再除以（1024*1024*1024），这样就可以得到每秒的字节数。

3. `align_with_offset`函数：这个函数的主要作用是将一个给定的内存指针（ptr）对齐到指定位置（offset），然后返回对齐后的内存地址。它接收两个参数：一个原始内存指针（ptr）和一个偏移量（offset）。函数内部首先计算出要分配的 alignment 值，然后将 ptr 和 alignment 值相加，最后将结果存储一个新的对齐后的内存地址。alignment 值是一个max_alignment（设置为4）的整数值，用于确保内存对齐。

总结：这段代码的主要目的是生成一些模拟数据，包括随机数、计算每秒字节数以及将随机数对准指定偏移量。


```cpp
// Generate synthetic data
static void generate_data(float offset, size_t n, float * dst) {
    for (size_t i = 0; i < n; i++) {
        dst[i] = 0.1 + 2*cosf(i + offset);
    }
}

static float gigabytes_per_second(size_t bytes, int64_t usecs) {
    return bytes / (float) usecs * 1000000 / (1024*1024*1024);
}

static void * align_with_offset(void * ptr, int offset) {
    size_t dummy_size = MAX_ALIGNMENT * 4;
    return (char *) std::align(MAX_ALIGNMENT, MAX_ALIGNMENT, ptr, dummy_size) + offset;
}

```

这是一个用GNU Scientific程序（GMP）编写的 benchmark 函数。该函数名为“基准函数”，用于在给定的输入尺寸和质量数据的情况下对给定的函数进行性能测试。

基准函数接受四个参数：输入数据的大小（Size）、查询数据的大小（QSize）、测试迭代数（Iterations）和函数本身。函数内部使用三个整型变量min_time_us、total_time_us和min_time_cycles来记录最小时间us、总时间us和最小时间 cycles，以及一个浮点型变量QK来表示输入数据的数量。

函数的主要部分在一个循环中进行，该循环使用给定的函数对输入数据进行调用，并记录每个调用的实际执行时间（包括函数调用之间的延迟）。在循环结束后，函数计算出平均时间us和总执行时间us，以及最小和平均时间 cycles。

函数的最终输出包括四个输出：平均每个输入单位的查询数据大小、每个输入单位的平均查询数据大小（包括AVG_QUERY_SIZE_GB对秒的换算）、量化通过性和量化总通过率。此外，函数还输出一个额外的信息：通过量（QK）在每秒钟处理了多少字节的数据。

该基准函数可以作为一种性能测试工具，用于比较不同实现之间的性能差异。您可以根据需要对其进行修改以适应您的具体应用场景。


```cpp
static void benchmark_function(size_t size, size_t q_size, int64_t iterations, const std::function<float(void)> & func) {
    int64_t min_time_us = INT64_MAX;
    int64_t total_time_us = 0;
    int64_t min_time_cycles = INT64_MAX;
    int64_t total_time_cycles = 0;

    for (int i = 0; i < WARMUP; i++) {
        func();
    }

    for (int i = 0; i < iterations; i++) {
        const int64_t start_time = ggml_time_us();
        const int64_t start_cycles = cpu_cycles();

        func();

        const int64_t end_cycles = cpu_cycles();
        const int64_t end_time = ggml_time_us();

        total_time_cycles += end_cycles - start_cycles;
        min_time_cycles = std::min(min_time_cycles, end_cycles - start_cycles);
        total_time_us += end_time - start_time;
        min_time_us = std::min(min_time_us, end_time - start_time);
    }

    printf("      min cycles/%d vals   : %9.2f\n",  QK, QK * min_time_cycles / (float) size);
    printf("      avg cycles/%d vals   : %9.2f\n",  QK, QK * total_time_cycles / (float) (size * iterations));
    printf("      float32 throughput   : %9.2f GB/s\n",  gigabytes_per_second(4 * size * iterations, total_time_us));
    printf("      quantized throughput : %9.2f GB/s\n",  gigabytes_per_second(q_size * iterations, total_time_us));
}

```

This is a C program that uses the `ggml` toolkit to perform a range of quantitative本是分析， also known as L1, L2, and L3 sizeof (SIZE). The program is typically used with the `-h` or `--help` option to see a usage example and then exits.

The program usage is as follows:

`./script.sh [options]`

The available options are:

* `-h, --help`
* `--size SIZE`
* `-3`
* `-4`
* `--op OP`
* `--type TYPE`
* `--alignment-offset OFFSET`
* `-i NUM, --iterations NUM`

The `-h` or `--help` option displays the usage information and then exits.

The `--size` option allows the user to specify the size of the test,divisible by 32 (e.g. L1_SIZE=64).

The `-3` or `--size` option use the specified size as L1, L2, L3, and MEM (e.g. L1_SIZE=64, L2_SIZE=128, L3_SIZE=192, MEM_SIZE=64).

The `--op` option allows the user to specify the test operation as `quantize_row_q_reference`, `quantize_row_q`, `dequantize_row_q` or `vec_dot_q` (all).

The `--type` option allows the user to specify the test type as `GGML_TYPE_COUNT` (e.g. GGML_TYPE_COUNT=1) or specific types like `GGML_TYPE_EXAMPLE`, `GGML_TYPE_SPARSE`, `GGML_TYPE_CSV`, etc.

The `--alignment-offset` option allows the user to specify the alignment offset as an integer offset (0).

The `-i` or `--iterations` option allows the user to specify the test iteration number (e.g. ITERATIONS=100).


```cpp
static void usage(char * argv[]) {
    printf("Benchmark quantization specific functions on synthetic data\n");
    printf("\n");
    printf("usage: %s [options]\n", argv[0]);
    printf("\n");
    printf("options: (default)\n");
    printf("  -h, --help            show this help message and exit\n");
    printf("  --size SIZE           set test size, divisible by 32 (L1_SIZE:%d)\n", L1_SIZE);
    printf("  -3                    use size as L1, L2, L3 sizes (L1:%d L2:%d L3:%d)\n", L1_SIZE, L2_SIZE, L3_SIZE);
    printf("  -4                    use size as L1, L2, L3, MEM sizes (L1:%d L2:%d L3:%d MEM:%d)\n", L1_SIZE, L2_SIZE, L3_SIZE, MEM_SIZE);
    printf("  --op OP               set test opration as quantize_row_q_reference, quantize_row_q, dequantize_row_q,\n");
    printf("                        quantize_row_q_dot, vec_dot_q (all)\n");
    printf("  --type TYPE           set test type as");
    for (int i = 0; i < GGML_TYPE_COUNT; i++) {
        ggml_type type = (ggml_type) i;
        ggml_type_traits_t qfns = ggml_internal_get_type_traits(type);
        if (ggml_type_name(type) != NULL) {
            if (qfns.from_float && qfns.to_float) {
                printf(" %s", ggml_type_name(type));
            }
        }
    }
    printf(" (all)\n");
    printf("  --alignment-offset OFFSET\n");
    printf("                        set alignment offset as OFFSET (0)\n");
    printf("  -i NUM, --iterations NUM\n");
    printf("                        set test iteration number (%d)\n", ITERATIONS);
}

```

This is a C++ program that uses the Quantile量化器对矩阵的dot产品进行量化。它使用了自动量化库TFLib，并使用了CUDA进行加速。程序的主要功能是对于给定的测试数据对矩阵的dot产品进行量化，并输出量化后的结果。

程序的输入是一个整型参数，包括要测量的两个矩阵，以及量化等级。程序将这些参数存储在一个结构体中，并使用这两个矩阵来计算出每个等级的量化数。

程序的主要步骤包括：

1. 对于每个等级的量化数，程序会使用TFLib自动量化器将其量化为实数类型。
2. 使用CUDA进行加速，将每个等级的量化数存储在一个包含多个元素的double数组中。
3. 将所有的量化数打印出来，以便进行比较和分析。

程序的输出结果包括：

1. 对于每个等级的量化数，程序会输出数量值和每输出一个等级所占用的内存空间（以MB为单位）。
2. 最后，程序会输出一个包含所有等级量化数的总体积。


```cpp
int main(int argc, char * argv[]) {
    quantize_perf_params params {};

    // read command line

    bool invalid_param = false;
    std::string arg;
    for (int i = 1; i < argc; i++) {
        arg = argv[i];

        if (arg == "--size") {
            if (++i >= argc) {
                invalid_param = true;
                break;
            }
            size_t size = std::stoi(argv[i]);
            if (size % 32 != 0) {
                fprintf(stderr, "error: size %zu not divisible by 32\n", size);
                invalid_param = true;
                break;
            }
            params.test_sizes.push_back(size);
        } else if (arg == "-3") {
            // quick select sizes that probably fit in CPU caches
            params.test_sizes.push_back(L1_SIZE);
            params.test_sizes.push_back(L2_SIZE);
            params.test_sizes.push_back(L3_SIZE);
        } else if (arg == "-4") {
            // quick select cache sizes + memory
            params.test_sizes.push_back(L1_SIZE);
            params.test_sizes.push_back(L2_SIZE);
            params.test_sizes.push_back(L3_SIZE);
            params.test_sizes.push_back(MEM_SIZE);
        } else if (arg == "--op") {
            if (++i >= argc) {
                invalid_param = true;
                break;
            }
            std::string op {argv[i]};
            if (op == "quantize_row_q_reference") {
                params.op_quantize_row_q_reference = true;
            } else if (op == "quantize_row_q") {
                params.op_quantize_row_q = true;
            } else if (op == "dequantize_row_q") {
                params.op_dequantize_row_q = true;
            } else if (op == "quantize_row_q_dot") {
                params.op_quantize_row_q_dot = true;
            } else if (op == "vec_dot_q") {
                params.op_vec_dot_q = true;
            } else {
                invalid_param = true;
                break;
            }
        } else if (arg == "--type") {
            if (++i >= argc) {
                invalid_param = true;
                break;
            }
            params.include_types.push_back(argv[i]);
        } else if (arg == "--alignment-offset") {
            if (++i >= argc) {
                invalid_param = true;
                break;
            }
            int alignment = std::stoi(argv[i]);
            if (alignment < 0 || alignment > MAX_ALIGNMENT) {
            fprintf(stderr, "error: aligment-offset must be less than %d\n", MAX_ALIGNMENT);
                invalid_param = true;
                break;
            }
            params.alignment_offset = alignment;
        } else if ((arg == "-i") || (arg == "--iterations")) {
            if (++i >= argc) {
                invalid_param = true;
                break;
            }
            int number = std::stoi(argv[i]);
            if (number < 0 || number > MAX_ITERATIONS) {
            fprintf(stderr, "error: iterations must be less than %d\n", MAX_ITERATIONS);
                invalid_param = true;
                break;
            }
            params.iterations = number;
        } else if ((arg == "-h") || (arg == "--help")) {
            usage(argv);
            return 1;
        } else {
            fprintf(stderr, "error: unknown argument: %s\n", arg.c_str());
            return 1;
        }
    }
    if (invalid_param) {
        fprintf(stderr, "error: invalid parameter for argument: %s\n", arg.c_str());
        return 1;
    }

    if (params.test_sizes.empty()) {
        params.test_sizes.push_back(L1_SIZE);
    }
    if (!(params.op_quantize_row_q_reference || params.op_quantize_row_q || params.op_dequantize_row_q || params.op_quantize_row_q_dot || params.op_vec_dot_q)) {
        params.op_quantize_row_q_reference = params.op_quantize_row_q = params.op_dequantize_row_q = params.op_quantize_row_q_dot = params.op_vec_dot_q = true;
    }

    std::sort(params.test_sizes.begin(), params.test_sizes.end());
    size_t largest = params.test_sizes.back();

    std::vector<uint8_t> test_data1_v(largest*4 + MAX_ALIGNMENT*2);
    std::vector<uint8_t> test_data2_v(largest*4 + MAX_ALIGNMENT*2);
    std::vector<uint8_t> test_q1_v   (largest*4 + MAX_ALIGNMENT*2);
    std::vector<uint8_t> test_q2_v   (largest*4 + MAX_ALIGNMENT*2);
    std::vector<uint8_t> test_out_v  (largest*4 + MAX_ALIGNMENT*2);

    float * test_data1 = (float *) align_with_offset(test_data1_v.data(), params.alignment_offset);
    float * test_data2 = (float *) align_with_offset(test_data2_v.data(), params.alignment_offset);
    float * test_q1    = (float *) align_with_offset(test_q1_v.data(),    params.alignment_offset);
    float * test_q2    = (float *) align_with_offset(test_q2_v.data(),    params.alignment_offset);
    float * test_out   = (float *) align_with_offset(test_out_v.data(),   params.alignment_offset);

    generate_data(0, largest, test_data1);
    generate_data(1, largest, test_data2);

    int64_t iterations = params.iterations;


    // Initialize GGML, ensures float conversion tables are initialized
    struct ggml_init_params ggml_params = {
        /* .mem_size   = */ 1*1024,
        /* .mem_buffer = */ NULL,
        /* .no_alloc   = */ true,
    };
    struct ggml_context * ctx = ggml_init(ggml_params);

    for (int i = 0; i < GGML_TYPE_COUNT; i++) {
        ggml_type type = (ggml_type) i;
        ggml_type_traits_t qfns = ggml_internal_get_type_traits(type);
        if (!params.include_types.empty() && ggml_type_name(type) && std::find(params.include_types.begin(), params.include_types.end(), ggml_type_name(type)) == params.include_types.end()) {
            continue;
        }

        if (qfns.from_float && qfns.to_float) {
            printf("%s\n", ggml_type_name(type));

            if (params.op_quantize_row_q_reference) {
                printf("  quantize_row_q_reference\n");
                for (size_t size : params.test_sizes) {
                    printf("    %zu values (%.2f MB)\n", size, 4*size/(float)(1024*1024));
                    auto quantize_fn = [&](void) -> float {
                        qfns.from_float_reference(test_data1, test_q1, size);
                        return test_q1[0];
                    };
                    size_t quantized_size = size / ggml_blck_size(type) * ggml_type_size(type);
                    benchmark_function(size, quantized_size, iterations, quantize_fn);
                }
                printf("\n");
            }

            if (params.op_quantize_row_q) {
                printf("  quantize_row_q\n");
                for (size_t size : params.test_sizes) {
                    printf("    %zu values (%.2f MB)\n", size, 4*size/(float)(1024*1024));
                    auto quantize_fn = [&](void) -> float {
                        qfns.from_float(test_data1, test_q1, size);
                        return test_q1[0];
                    };
                    size_t quantized_size = size / ggml_blck_size(type) * ggml_type_size(type);
                    benchmark_function(size, quantized_size, iterations, quantize_fn);
                }
                printf("\n");
            }

            if (params.op_dequantize_row_q) {
                printf("  dequantize_row_q\n");
                qfns.from_float(test_data1, test_q1, largest);
                for (size_t size : params.test_sizes) {
                    printf("    %zu values (%.2f MB)\n", size, 4*size/(float)(1024*1024));
                    auto quantize_fn = [&](void) -> float {
                        qfns.to_float(test_q1, test_out, size);
                        return test_out[0];
                    };
                    size_t quantized_size = size / ggml_blck_size(type) * ggml_type_size(type);
                    benchmark_function(size, quantized_size, iterations, quantize_fn);
                }
                printf("\n");
            }

            if (params.op_quantize_row_q_dot) {
                printf("  quantize_row_q_dot\n");
                for (size_t size : params.test_sizes) {
                    printf("    %zu values (%.2f MB)\n", size, 4*size/(float)(1024*1024));
                    auto quantize_fn = [&](void) -> float {
                        auto vdot = ggml_internal_get_type_traits(qfns.vec_dot_type);
                        vdot.from_float(test_data1, test_q1, size);
                        return test_q1[0];
                    };
                    size_t quantized_size = size / ggml_blck_size(type) * ggml_type_size(type);
                    benchmark_function(size, quantized_size, iterations, quantize_fn);
                }
                printf("\n");
            }

            if (params.op_vec_dot_q) {
                printf("  vec_dot_q\n");
                qfns.from_float(test_data1, test_q1, largest);
                qfns.from_float(test_data2, test_q2, largest);
                for (size_t size : params.test_sizes) {
                    printf("    %zu values (%.2f MB)\n", size, 4*size/(float)(1024*1024));
                    auto quantize_fn = [&](void) -> float {
                        float result;
                        qfns.vec_dot(size, &result, test_q1, test_q2);
                        return result;
                    };
                    size_t quantized_size = size / ggml_blck_size(type) * ggml_type_size(type);
                    benchmark_function(size, quantized_size, iterations, quantize_fn);
                }
                printf("\n");
            }
        }
    }

    ggml_free(ctx);

    return 0;
}

```

# `tests/test-rel-pos.c`

这段代码包括两个部分，其中第一个部分定义了一个名为 `make_ctx` 的函数，其参数为 `void`。该函数调用了 `ggml_init` 函数，并返回了一个指向 `ggml_context` 结构体的指针。

第二个部分定义了一个名为 `check_tensor` 的函数，该函数接收一个 `struct ggml_tensor` 类型的输入 `t`，以及一个指向 `float` 类型的预期输出 `expected_t_d` 的指针 `t_d` 和三个整数 `ne0` 和 `ne1`。该函数使用一系列的循环来检查输入 `t` 中的数据是否与预期输出 `expected_t_d` 一致。具体来说，该函数首先检查输入 `t` 的类型是否为 `GGML_TYPE_F32`，然后检查输入 `t` 中的 `ne` 是否与输入 `ne0` 和 `ne1` 一致。最后，该函数使用循环来逐个比较输入 `t` 中的每个数据元素与预期输出 `expected_t_d` 中的对应元素是否一致，如果一致，则返回 `true`，否则返回 `false`。

该代码的作用是定义了一个函数 `make_ctx`，该函数返回一个指向 `ggml_context` 结构体的指针。该函数使用 `ggml_init` 函数初始化了一个 `ggml_init_params` 结构体，该结构体包括 `.mem_size` 成员，用于指定 `ggml_init` 函数要分配的内存大小。

该代码还定义了一个函数 `check_tensor`，该函数用于检查一个 `struct ggml_tensor` 输入 `t` 中的数据是否与一个指向 `float` 类型的预期输出 `expected_t_d` 一致。该函数接收输入 `t`、预期输出 `expected_t_d` 和三个整数 `ne0` 和 `ne1` 作为参数。该函数使用一系列的循环来逐个比较输入 `t` 中的每个数据元素与预期输出 `expected_t_d` 中的对应元素是否一致，如果一致，则返回 `true`，否则返回 `false`。


```cpp
#include "ggml/ggml.h"

#include <string.h>
#include <stdio.h>
#include <stdlib.h>

struct ggml_context* make_ctx(void) {
    struct ggml_init_params params = {
        .mem_size = 2 * 1024 * 1024,
    };

    return ggml_init(params);
}

void check_tensor(struct ggml_tensor * t, float * expected_t_d, int ne0, int ne1, int ne2) {
    GGML_ASSERT(t->type == GGML_TYPE_F32);
    GGML_ASSERT(t->ne[0] == ne0);
    GGML_ASSERT(t->ne[1] == ne1);
    GGML_ASSERT(t->ne[2] == ne2);
    for (int i2 = 0; i2 < ne2; ++i2) {
        for (int i1 = 0; i1 < ne1; ++i1) {
            for (int i0 = 0; i0 < ne0; ++i0) {
                float expected = *(expected_t_d + i2 * ne1 * ne0 + i1 * ne0 + i0);
                float actual = ggml_get_data_f32(t)[i2 * ne1 * ne0 + i1 * ne0 + i0];
                GGML_ASSERT(expected == actual);
            }
        }
    }
}

```

This is a C function that performs a simple linear regression on a 3D tensor. The function takes in a 3D tensor represented by the input parameter `ggml_tensor_3d(ctx, GGML_TYPE_F32, 3, 2, 2)`, and returns the linear regression result.

The function first initializes the output tensor as a copy of the input tensor with a static value of 0. Then it creates a loop that iterates through each element of the input tensor. Inside the loop, it creates a new tensor that represents the current value of the input tensor at the current iteration, and sets the static value of the output tensor to 1.

Next, it creates a new tensor that represents the static value of the output tensor, and sets the elements of the new tensor to 1. Finally, it performs a linear regression by building a new tensor that represents the linear combination of the input tensor and the static value of the output tensor, and returns the result.

Note that the function does not handle the computation of the output value outside of the input range, and does not handle the derivative of the linear relationship.


```cpp
int main(int argc, const char** argv) {
    ggml_fp16_t buf_f16[1024];
    for (int i = 0; i < 1024; ++i) {
        buf_f16[i] = ggml_fp32_to_fp16((float)i);
    }

    float expected_out[4][9] = {
        { 8.0, 9.0, 10.0, 9.0, 10.0, 11.0, 10.0, 11.0, 12.0 },
        { 2.0, 3.0, 4.0, 3.0, 4.0, 5.0, 4.0, 5.0, 6.0 },
        { 14.0, 15.0, 16.0, 15.0, 16.0, 17.0, 16.0, 17.0, 18.0 },
        { 8.0, 9.0, 10.0, 9.0, 10.0, 11.0, 10.0, 11.0, 12.0 },
    };

    {
        struct ggml_context * ctx = make_ctx();


        struct ggml_tensor * t = ggml_new_tensor_2d(ctx, GGML_TYPE_F16, 3, 3);
        ggml_fp16_t* t_d = (ggml_fp16_t*)t->data;
        memcpy(t_d, buf_f16, ggml_nbytes(t));

        struct ggml_tensor * t_2 = ggml_new_tensor_2d(ctx, GGML_TYPE_F16, 3, 3);
        ggml_fp16_t* t_d_2 = (ggml_fp16_t*)t_2->data;
        memcpy(t_d_2, buf_f16 + 1, ggml_nbytes(t_2));

        struct ggml_tensor * rw = ggml_get_rel_pos(ctx, t, 2, 2);
        struct ggml_tensor * rh = ggml_get_rel_pos(ctx, t_2, 2, 2);

        struct ggml_tensor * rw_f32 = ggml_cpy(ctx, rw, ggml_new_tensor_3d(ctx, GGML_TYPE_F32, 3, 2, 2));
        struct ggml_tensor * rh_f32 = ggml_cpy(ctx, rh, ggml_new_tensor_3d(ctx, GGML_TYPE_F32, 3, 2, 2));

        struct ggml_tensor * in = ggml_new_tensor_2d(ctx, GGML_TYPE_F32, 9, 4);
        struct ggml_tensor * out_inplace = ggml_new_tensor_2d(ctx, GGML_TYPE_F32, 9, 4);
        float * in_d          = (float*)in->data;
        float * out_inplace_d = (float*)out_inplace->data;
        for (int i = 0; i < ggml_nelements(in); ++i) {
            in_d[i]          = 1.f;
            out_inplace_d[i] = 1.f;
        }

        struct ggml_tensor * out = ggml_add_rel_pos(ctx, in, rw_f32, rh_f32);
        struct ggml_cgraph * gf = ggml_new_graph(ctx);
        ggml_build_forward_expand(gf, out);
        ggml_graph_compute_with_ctx(ctx, gf, 1);

        out_inplace = ggml_add_rel_pos_inplace(ctx, out_inplace, rw_f32, rh_f32);
        struct ggml_cgraph * gf_2 = ggml_new_graph(ctx);
        ggml_build_forward_expand(gf_2, out_inplace);
        ggml_graph_compute_with_ctx(ctx, gf_2, 1);

        check_tensor(out, (float*)expected_out, 9, 4, 1);
        check_tensor(out_inplace, (float*)expected_out, 9, 4, 1);
    }

    return 0;
}

```

# `tests/test-svd0.c`

这段代码是一个用于进行主成分分析（SVD）的C代码程序。它的主要目的是在三维点对中找出最主要的主成分，以便对数据进行降维。以下是这段代码的功能和用途的简要说明：

1. 包含必要的头文件：包括GNU浮点数库（float.h）、GNU整数库（stdint.h）和GNU调试库（assert.h）。

2. 定义宏：定义了一个名为“__ __RCSID(0,4365)”的宏，表示这段代码是由ID为0，版本为4365的软件贡献者开发的。

3. 包含输入文件：从用户输入中读取数据，通常是一个三元组（即三个数值）的主成分。

4. 包含输出文件：将降维后的数据保存为用户指定的文件名。

5. 实现SVD算法：为了解决SVD问题，程序首先需要计算矩阵的协方差矩阵、H矩阵和欧拉矩阵。然后，利用G气象提法（RMS擅长加速）来求解这个问题，如果加速失败，就直接使用慢的方式求解。

6. 包含时间测量：通过调用系统计时器（而不是标准输入或用户输入）来记录执行时间，以便在以后回顾。

7. 定义文件头缓冲区：为了在文件读取时可以方便地检测并修复读取错误。

8. 通过输入文件数据计算主成分：程序从用户输入的三元组中提取数据，并计算降维的主成分。

9. 保存数据：将计算得到的二级主成分按照降序存储，保存为用户指定的文件名。

总之，这段代码的主要目的是实现一个基于SVD的主成分分析算法，以便对三维点对数据进行降维处理。


```cpp
// SVD dimensionality reduction

#include <float.h>
#include <stdint.h>
#include <stdio.h>
#include <assert.h>
#include <stdlib.h>
#include <string.h>
#include <time.h>
#include <math.h>

#include <sys/time.h>

#ifdef GGML_USE_ACCELERATE
#include <Accelerate/Accelerate.h>
```

This is a C-language program that performs an image registration task, where it combines a set of non-maximum contrast images (U), and corresponding non-maximum contrast objects (A0), into a single binary image (A). The program reads input data from two separate files, U and A0, and outputs the注册后的 binary image A.

The program follows these steps:

1. reads U and A0 from two separate files.
2. normalizes U by normalizing each pixel to a sum of its corresponding pixels in A0.
3. normalizes A0 by normalizing each pixel to a sum of its corresponding pixels in U.
4. normalizes both U and A0 by dividing by asqrt(number of pixels in the image).
5. prints the normalized U.
6. prints the normalized A0.
7. prints the final binary image A.

The program assumes that the input files U and A0 contain non-maximum contrast images, and that the images have the same dimensions. The program also assumes that the normalization step normalizing U by normalizing each pixel to a sum of its corresponding pixels in A0 is done correctly.


```cpp
#endif

float frand(void) {
    return (float) rand() / (float) RAND_MAX;
}

//int sgesvd_(char *__jobu, char *__jobvt, __CLPK_integer *__m,
//        __CLPK_integer *__n, __CLPK_real *__a, __CLPK_integer *__lda,
//        __CLPK_real *__s, __CLPK_real *__u, __CLPK_integer *__ldu,
//        __CLPK_real *__vt, __CLPK_integer *__ldvt, __CLPK_real *__work,
//        __CLPK_integer *__lwork,
//        __CLPK_integer *__info)

int main(int argc, const char ** argv) {
    int m = 10;
    int n = 5;

    float * A  = malloc(n * m * sizeof(float));
    float * A0 = malloc(n * m * sizeof(float));

    for (int i = 0; i < n; ++i) {
        for (int j = 0; j < m; ++j) {
            A[i * m + j] = (float) (10.0f*(i + 1) + 1.0f * frand());
            //A[i * m + j] = (float) (10.0f*(i%2 + 1) + 0.1f * frand());
            //if (i == 2) {
            //    A[i * m + j] += 20*frand();
            //}
            if ((i == 1 || i == 3) && j > m/2) {
                A[i * m + j] = -A[i * m + j];
            }
        }
    }

    // average vector
    //float * M = malloc(m * sizeof(float));

    //{
    //    for (int j = 0; j < m; ++j) {
    //        M[j] = 0.0f;
    //    }
    //    for (int i = 0; i < n; ++i) {
    //        for (int j = 0; j < m; ++j) {
    //            M[j] += A[i * m + j];
    //        }
    //    }
    //    for (int j = 0; j < m; ++j) {
    //        M[j] /= (float) n;
    //    }
    //}

    //// subtract average vector
    //for (int i = 0; i < n; ++i) {
    //    for (int j = 0; j < m; ++j) {
    //        A[i * m + j] -= M[j];
    //    }
    //}

    memcpy(A0, A, n * m * sizeof(float));

    // print A
    printf("A:\n");
    for (int i = 0; i < n; ++i) {
        printf("col %d : ", i);
        for (int j = 0; j < m; ++j) {
            printf("%9.5f ", A[i * m + j]);
        }
        printf("\n");
    }
    printf("\n");

    // SVD
    // A = U * S * V^T

    float * U = malloc(n * m * sizeof(float));
    float * S = malloc(n * sizeof(float));
    float * V = malloc(n * n * sizeof(float));

    int lda = m;
    int ldu = m;
    int ldvt = n;

    float work_size;
    int lwork = -1;
    int info = 0;

    sgesvd_("S", "S", &m, &n, A, &lda, S, U, &ldu, V, &ldvt, &work_size, &lwork, &info);

    lwork = (int) work_size;

    printf("work_size = %f, info = %d, lwork = %d\n", work_size, info, lwork);

    float * work = malloc(lwork * sizeof(float));

    sgesvd_("S", "S", &m, &n, A, &lda, S, U, &ldu, V, &ldvt, work, &lwork, &info);

    // print U
    printf("U:\n");
    for (int i = 0; i < n; ++i) {
        printf("col %d : ", i);
        for (int j = 0; j < m; ++j) {
            printf("%9.5f ", U[i * m + j]);
        }
        printf("\n");
    }
    printf("\n");

    // normalize S
    {
        double sum = 0.0;
        for (int i = 0; i < n; ++i) {
            sum += S[i];
        }
        sum *= sqrt((double) m);
        for (int i = 0; i < n; ++i) {
            S[i] /= sum;
        }
    }

    // print S
    printf("S:\n");
    for (int i = 0; i < n; ++i) {
        printf("- %d = %9.5f\n", i, S[i]);
    }
    printf("\n");

    // print V
    printf("V:\n");
    for (int i = 0; i < n; ++i) {
        printf("col %d : ", i);
        for (int j = 0; j < n; ++j) {
            printf("%9.5f ", V[i * n + j]);
        }
        printf("\n");
    }
    printf("\n");

    // print A
    printf("A:\n");
    for (int i = 0; i < n; ++i) {
        printf("col %d : ", i);
        for (int j = 0; j < m; ++j) {
            printf("%9.5f ", A[i * m + j]);
        }
        printf("\n");
    }
    printf("\n");

    // compute singular vectors in U
    for (int i = 0; i < n; ++i) {
        for (int j = 0; j < m; ++j) {
            U[i * m + j] *= S[i];
        }
    }

    // normalize U
    for (int i = 0; i < n; ++i) {
        double sum = 0.0;
        for (int j = 0; j < m; ++j) {
            sum += U[i * m + j] * U[i * m + j];
        }
        sum = sqrt(sum);
        for (int j = 0; j < m; ++j) {
            U[i * m + j] /= sum*sqrt((double) m);
        }
    }

    // print U
    printf("U:\n");
    for (int i = 0; i < n; ++i) {
        printf("col %d : ", i);
        for (int j = 0; j < m; ++j) {
            printf("%9.5f ", U[i * m + j]);
        }
        printf("\n");
    }
    printf("\n");


    // project A0 onto U
    float * A1 = malloc(n * n * sizeof(float));

    for (int i = 0; i < n; ++i) {
        for (int j = 0; j < n; ++j) {
            A1[i * n + j] = 0.0f;
            for (int k = 0; k < m; ++k) {
                A1[i * n + j] += A0[i * m + k] * U[j * m + k];
            }
        }
    }

    // print A1
    printf("A1:\n");
    for (int i = 0; i < n; ++i) {
        printf("col %d : ", i);
        for (int j = 0; j < n; ++j) {
            printf("%9.5f ", A1[i * n + j]);
        }
        printf("\n");
    }
    printf("\n");

    return 0;
}

```