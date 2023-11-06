# GGML源码解析 6

# `examples/gpt-2/main-backend.cpp`

这段代码是一个通用的C++代码，用于在GGML中进行计算。GGML是一种C++的图形渲染引擎，可以用来创建高度优化的3D图形。

首先，这段代码引入了GGML的基本头文件、内存分配头文件和后端头文件。这些文件定义了GGML的基本概念和数据结构，为后续的代码实现提供了必要的支持。

接下来，代码中使用了三个条件编译指令#ifdef和#ifdef。这些指令允许在特定的编译环境下使用特定的一些C函数。如果预先定义了这些函数，则可以直接包含代码，否则需要安装对应的C库。这里，作者可能已经为GGML的CUDA和Metal库安装了C库，所以这些库可以被使用。

然后，代码中引入了两个头文件：common.h和common-ggml.h。这些文件中定义了一些通用的数据结构和函数，可以被所有GGML应用程序共享。

最后，代码中定义了一些函数，包括：ggmCreateContext、ggmCreateStyle、ggmCreateGeometry、ggmUpdateCamera、ggmAnimate等。这些函数是GGML的基本构建块，可以用来创建、修改和操作3D图形。


```cpp
#include "ggml/ggml.h"
#include "ggml/ggml-alloc.h"
#include "ggml/ggml-backend.h"

#ifdef GGML_USE_CUBLAS
#include "ggml-cuda.h"
#endif

#ifdef GGML_USE_METAL
#include "ggml-metal.h"
#endif

#include "common.h"
#include "common-ggml.h"

```

这段代码是一个 C++ 程序，它包括几个头文件，以及一个函数指针 ggml_log_callback_default。函数指针指向一个名为 "ggml_log_callback_default" 的函数，这个函数接受一个 ggml 日志级别、一个字符串文本和一个 void 类型的用户数据。

这个程序的作用是记录调试信息。它通过 fopen 函数打开一个名为 "dbg.txt" 的文件，将调试信息记录到文件中。通过 fflush 函数将文件中的所有内容刷新到屏幕上，以便在程序运行时可以看到调试信息。

程序中使用了一个名为 "ggml_log_callback_default" 的函数作为自定义的调试信息记录函数。这个函数接受一个 ggml 日志级别、一个字符串文本和一个 void 类型的用户数据。函数内部只是简单地将记录到的调试信息打印到屏幕上，并通过 fputs 函数将调试信息写入到 "dbg.txt" 文件中。

如果定义了 _MSC_VER，则可能会丢失一些调试信息。


```cpp
#include <cassert>
#include <cmath>
#include <cstdio>
#include <cstring>
#include <fstream>
#include <map>
#include <string>
#include <vector>

#if defined(_MSC_VER)
#pragma warning(disable: 4244 4267) // possible loss of data
#endif

static void ggml_log_callback_default(ggml_log_level level, const char * text, void * user_data) {
    (void) level;
    (void) user_data;
    fputs(text, stderr);
    fflush(stderr);
}

```

这段代码定义了一个名为 `gpt2_hparams` 的结构体，以及一个名为 `gpt2_layer` 的结构体。

`gpt2_hparams` 结构体定义了预训练语言模型的参数，包括：`n_vocab` 表示词汇表大小，`n_ctx` 表示当前序列长度和预训练语言模型的上下文连接数，`n_embd` 表示嵌入层大小，`n_head` 表示头数，`n_layer` 表示预训练语言模型的层数，`ftype` 表示是否使用 fastText 文本预处理。此外，还定义了一个浮点数变量 `eps`，用于表示精确度。

`gpt2_layer` 结构体定义了预训练语言模型的层信息，包括：

* `ln_1_g` 和 `ln_1_b` 表示第一个实体在序列中的起始位置
* `ln_2_g` 和 `ln_2_b` 表示第二个实体在序列中的起始位置
* `c_attn_attn_w` 和 `c_attn_attn_b` 表示注意力权重
* `c_attn_proj_w` 和 `c_attn_proj_b` 表示注意力 projection 权重
* `c_mlp_fc_w` 和 `c_mlp_fc_b` 表示多层感知器（MLP）的输出
* `c_mlp_proj_w` 和 `c_mlp_proj_b` 表示多层感知器（MLP）的权重

此外，还有一段注释，指出 `eps` 的作用是设置模型的最终输出结果的精确度。


```cpp
// default hparams (GPT-2 117M)
struct gpt2_hparams {
    int32_t n_vocab = 50257;
    int32_t n_ctx   = 1024;
    int32_t n_embd  = 768;
    int32_t n_head  = 12;
    int32_t n_layer = 12;
    int32_t ftype   = 1;
    float   eps     = 1e-5f;
};

struct gpt2_layer {
    // normalization
    struct ggml_tensor * ln_1_g;
    struct ggml_tensor * ln_1_b;

    struct ggml_tensor * ln_2_g;
    struct ggml_tensor * ln_2_b;

    // attention
    struct ggml_tensor * c_attn_attn_w;
    struct ggml_tensor * c_attn_attn_b;

    struct ggml_tensor * c_attn_proj_w;
    struct ggml_tensor * c_attn_proj_b;

    // mlp
    struct ggml_tensor * c_mlp_fc_w;
    struct ggml_tensor * c_mlp_fc_b;

    struct ggml_tensor * c_mlp_proj_w;
    struct ggml_tensor * c_mlp_proj_b;
};

```



这段代码定义了一个名为 `gpt2_model` 的结构体，用于表示基于 GPT-2 模型的语言模型。

这个结构体包含以下几个部分：

1. `hparams` 是一个 `gpt2_hparams` 类型的成员变量，用于设置模型参数，例如文本刻度，隐藏层大小等。

2. `ln_f_g` 和 `ln_f_b` 是两个 `ggml_tensor` 类型的成员变量，用于存储模型中的语言信息。这些信息在训练过程中使用，例如文本的嵌入向量。

3. `wte`、`wpe` 和 `lm_head` 是三个 `ggml_tensor` 类型的成员变量，用于存储文本的位置嵌入和词嵌入。

4. `layers` 是一个 `std::vector<gpt2_layer>` 类型的成员变量，用于存储整个语言模型的层。

5. `memory_k` 和 `memory_v` 是两个 `ggml_tensor` 类型的成员变量，用于存储模型中的键值对。

6. `ctx` 是这个结构体的成员变量，用于存储训练过程的上下文。

7. `backend` 是一个 `ggml_backend_t` 类型的成员变量，用于设置整个模型的后端。

8. `buffer_w` 和 `buffer_kv` 是两个 `ggml_backend_buffer_t` 类型的成员变量，用于存储输入数据和键值对。

9. `tensors` 是一个 `std::map<std::string, struct ggml_tensor *>` 类型的成员变量，用于存储模型中的各种数据。

这个结构体表示了一个完整的基于 GPT-2 模型的语言模型，它包含了模型和与之相关的训练过程中的参数和数据。


```cpp
struct gpt2_model {
    gpt2_hparams hparams;

    // normalization
    struct ggml_tensor * ln_f_g;
    struct ggml_tensor * ln_f_b;

    struct ggml_tensor * wte;     // position embedding
    struct ggml_tensor * wpe;     //    token embedding
    struct ggml_tensor * lm_head; // language model head

    std::vector<gpt2_layer> layers;

    // key + value memory
    struct ggml_tensor * memory_k;
    struct ggml_tensor * memory_v;

    //
    struct ggml_context * ctx;

    ggml_backend_t backend = NULL;

    ggml_backend_buffer_t buffer_w;
    ggml_backend_buffer_t buffer_kv;

    std::map<std::string, struct ggml_tensor *> tensors;
};

```

In the code you provided, the backend buffer size is being calculated as a sum of several tensor sizes, along with an alignment overhead. The buffer size is then printed along with the size of the ggml tensor.

Here's a breakdown of what's happening in the code:

1. Calculating the buffer size for each tensor size.
```cppmakefile
       buffer_size += n_layer*(       n_embd*ggml_type_sizef(GGML_TYPE_F32));   // c_attn_proj_b

       buffer_size += n_layer*(4*n_embd*n_embd*ggml_type_sizef(wtype));         // c_mlp_fc_w
       buffer_size += n_layer*(       4*n_embd*ggml_type_sizef(GGML_TYPE_F32)); // c_mlp_fc_b

       buffer_size += n_layer*(4*n_embd*n_embd*ggml_type_sizef(wtype));         // c_mlp_proj_w
       buffer_size += n_layer*(         n_embd*ggml_type_sizef(GGML_TYPE_F32)); // c_mlp_proj_b

       buffer_size += (6 + 12*n_layer)*128; // alignment overhead

       printf("%s: ggml tensor size    = %d bytes\n", __func__, (int) sizeof(ggml_tensor));
       printf("%s: backend buffer size = %6.2f MB\n", __func__, buffer_size/(1024.0*1024.0));
   }
```
This code calculates the buffer size for each tensor size by first getting the size of the tensor (which is the same as the `ggml_tensor_overhead()` function), and then adding a multiple of the number of layers (specified by `model.hparams.n_layer`) to get the total number of elements in the tensor. The resulting buffer size is then added to the `buffer_size` variable.

2. Creating the ggml context and initializing it with the appropriate parameters.
```cppphp
   {
       size_t n_tensors = 2 + 6 + 12*model.hparams.n_layer;
       struct ggml_init_params params = {
           /*.mem_size   =*/ ggml_tensor_overhead() * n_tensors,
           /*.mem_buffer =*/ NULL,
           /*.no_alloc   =*/ true,
       };

       model.ctx = ggml_init(params);
       if (!model.ctx) {
           fprintf(stderr, "%s: ggml_init() failed\n", __func__);
           return false;
       }
   }
```
This code creates the ggml context by calling `ggml_init()` with the appropriate parameters. The `ggml_tensor_overhead()` function is used to calculate the memory overhead for each tensor type, based on the number of elements in the tensor (which is specified by `model.hparams.n_layer`), the number of layers, and the size of the data type (which is also specified by `model.hparams.n_layer`). The resulting buffer size is then set to `params.mem_size`, and the `NULL` pointer is set to `params.mem_buffer`. Additionally, the `no_alloc` parameter is set to `params.no_alloc`, which means that the backend will automatically allocate the necessary memory even if the buffer is full.

3. Initializing the backend with the `ggml_init()` function.
```cppphp
   {
       size_t n_tensors = 2 + 6 + 12*model.hparams.n_layer;
       struct ggml_init_params params = {
           /*.mem_size   =*/ ggml_tensor_overhead() * n_tensors,
           /*.mem_buffer =*/ NULL,
           /*.no_alloc   =*/ true,
       };

       model.ctx = ggml_init(params);
       if (!model.ctx) {
           fprintf(stderr, "%s: ggml_init() failed\n", __func__);
           return false;
       }
   }
```
This code initializes the backend of the model with the `ggml_init()` function, by passing in the `params` structure with the same parameters as before.

Note that in the code you provided, the buffer size is calculated as a sum of several tensor sizes, along with an alignment overhead. The total buffer size is then printed along with the size of the ggml tensor.


```cpp
// load the model's weights from a file
bool gpt2_model_load(const std::string & fname, gpt2_model & model, gpt_vocab & vocab, int n_ctx, int n_gpu_layers) {
    printf("%s: loading model from '%s'\n", __func__, fname.c_str());

    auto fin = std::ifstream(fname, std::ios::binary);
    if (!fin) {
        fprintf(stderr, "%s: failed to open '%s'\n", __func__, fname.c_str());
        return false;
    }

    // verify magic
    {
        uint32_t magic;
        fin.read((char *) &magic, sizeof(magic));
        if (magic != GGML_FILE_MAGIC) {
            fprintf(stderr, "%s: invalid model file '%s' (bad magic)\n", __func__, fname.c_str());
            return false;
        }
    }

    // load hparams
    {
        auto & hparams = model.hparams;

        fin.read((char *) &hparams.n_vocab, sizeof(hparams.n_vocab));
        fin.read((char *) &hparams.n_ctx,   sizeof(hparams.n_ctx));
        fin.read((char *) &hparams.n_embd,  sizeof(hparams.n_embd));
        fin.read((char *) &hparams.n_head,  sizeof(hparams.n_head));
        fin.read((char *) &hparams.n_layer, sizeof(hparams.n_layer));
        fin.read((char *) &hparams.ftype,   sizeof(hparams.ftype));

        const int32_t qntvr = hparams.ftype / GGML_QNT_VERSION_FACTOR;

        printf("%s: n_vocab = %d\n", __func__, hparams.n_vocab);
        printf("%s: n_ctx   = %d\n", __func__, hparams.n_ctx);
        printf("%s: n_embd  = %d\n", __func__, hparams.n_embd);
        printf("%s: n_head  = %d\n", __func__, hparams.n_head);
        printf("%s: n_layer = %d\n", __func__, hparams.n_layer);
        printf("%s: ftype   = %d\n", __func__, hparams.ftype);
        printf("%s: qntvr   = %d\n", __func__, qntvr);

        hparams.ftype %= GGML_QNT_VERSION_FACTOR;
    }

    // load vocab
    {
        int32_t n_vocab = 0;
        fin.read((char *) &n_vocab, sizeof(n_vocab));

        if (n_vocab != model.hparams.n_vocab) {
            fprintf(stderr, "%s: invalid model file '%s' (bad vocab size %d != %d)\n",
                    __func__, fname.c_str(), n_vocab, model.hparams.n_vocab);
            return false;
        }

        std::string word;
        std::vector<char> buf(128);

        for (int i = 0; i < n_vocab; i++) {
            uint32_t len;
            fin.read((char *) &len, sizeof(len));

            buf.resize(len);
            fin.read((char *) buf.data(), len);
            word.assign(buf.data(), len);

            vocab.token_to_id[word] = i;
            vocab.id_to_token[i] = word;
        }
    }

    // for the big tensors, we have the option to store the data in 16-bit floats or quantized
    // in order to save memory and also to speed up the computation
    ggml_type wtype = ggml_ftype_to_ggml_type((ggml_ftype) (model.hparams.ftype));
    if (wtype == GGML_TYPE_COUNT) {
        fprintf(stderr, "%s: invalid model file '%s' (bad ftype value %d)\n",
                __func__, fname.c_str(), model.hparams.ftype);
        return false;
    }

    auto & ctx = model.ctx;

    size_t buffer_size = 0;

    {
        const auto & hparams = model.hparams;

        const int n_embd  = hparams.n_embd;
        const int n_layer = hparams.n_layer;
        const int n_ctx   = hparams.n_ctx;
        const int n_vocab = hparams.n_vocab;

        buffer_size += n_embd*ggml_type_sizef(GGML_TYPE_F32); // ln_f_g
        buffer_size += n_embd*ggml_type_sizef(GGML_TYPE_F32); // ln_f_b

        buffer_size += n_vocab*n_embd*ggml_type_sizef(wtype);         // wte
        buffer_size +=   n_ctx*n_embd*ggml_type_sizef(GGML_TYPE_F32); // wpe
        buffer_size += n_vocab*n_embd*ggml_type_sizef(wtype);         // lm_head

        buffer_size += n_layer*(n_embd*ggml_type_sizef(GGML_TYPE_F32)); // ln_1_g
        buffer_size += n_layer*(n_embd*ggml_type_sizef(GGML_TYPE_F32)); // ln_1_b

        buffer_size += n_layer*(n_embd*ggml_type_sizef(GGML_TYPE_F32)); // ln_2_g
        buffer_size += n_layer*(n_embd*ggml_type_sizef(GGML_TYPE_F32)); // ln_2_b

        buffer_size += n_layer*(3*n_embd*n_embd*ggml_type_sizef(wtype));         // c_attn_attn_w
        buffer_size += n_layer*(       3*n_embd*ggml_type_sizef(GGML_TYPE_F32)); // c_attn_attn_b

        buffer_size += n_layer*(n_embd*n_embd*ggml_type_sizef(wtype));           // c_attn_proj_w
        buffer_size += n_layer*(       n_embd*ggml_type_sizef(GGML_TYPE_F32));   // c_attn_proj_b

        buffer_size += n_layer*(4*n_embd*n_embd*ggml_type_sizef(wtype));         // c_mlp_fc_w
        buffer_size += n_layer*(       4*n_embd*ggml_type_sizef(GGML_TYPE_F32)); // c_mlp_fc_b

        buffer_size += n_layer*(4*n_embd*n_embd*ggml_type_sizef(wtype));         // c_mlp_proj_w
        buffer_size += n_layer*(         n_embd*ggml_type_sizef(GGML_TYPE_F32)); // c_mlp_proj_b

        buffer_size += (6 + 12*n_layer)*128; // alignment overhead

        printf("%s: ggml tensor size    = %d bytes\n", __func__, (int) sizeof(ggml_tensor));
        printf("%s: backend buffer size = %6.2f MB\n", __func__, buffer_size/(1024.0*1024.0));
    }

    // create the ggml context
    {
        size_t n_tensors = 2 + 6 + 12*model.hparams.n_layer;
        struct ggml_init_params params = {
            /*.mem_size   =*/ ggml_tensor_overhead() * n_tensors,
            /*.mem_buffer =*/ NULL,
            /*.no_alloc   =*/ true,
        };

        model.ctx = ggml_init(params);
        if (!model.ctx) {
            fprintf(stderr, "%s: ggml_init() failed\n", __func__);
            return false;
        }
    }

    // initialize the backend
```

这段代码是用来在ggml模型中根据使用的CUDA或Metal GPU层数量，选择使用CUDA或Metal进行模型的构建。

具体来说，如果使用的是CUDA GPU层，那么代码会输出一条消息并设置model.backend为ggml_backend_cuda_init的返回值，如果设置失败，则会输出第二条消息并设置为ggml_backend_cuda_init的返回值。如果使用的是Metal GPU层，那么代码会输出一条消息并设置ggml_log_callback_default为ggml_metal_log_set_callback的返回值，设置model.backend为ggml_backend_metal_init的返回值，如果设置失败，则会输出第二条消息并设置为ggml_backend_metal_init的返回值。ggml_metal_log_set_callback函数用于设置log输出函数，该函数会在模型构建成功后输出一个printf格式化字符串，其中n_gpu_layers为0时，输出为"using Metal backend"。


```cpp
#ifdef GGML_USE_CUBLAS
    if (n_gpu_layers > 0) {
        fprintf(stderr, "%s: using CUDA backend\n", __func__);
        model.backend = ggml_backend_cuda_init();
        if (!model.backend) {
            fprintf(stderr, "%s: ggml_backend_cuda_init() failed\n", __func__);
        }
    }
#endif

#ifdef GGML_USE_METAL
    if (n_gpu_layers > 0) {
        fprintf(stderr, "%s: using Metal backend\n", __func__);
        ggml_metal_log_set_callback(ggml_log_callback_default, nullptr);
        model.backend = ggml_backend_metal_init();
        if (!model.backend) {
            fprintf(stderr, "%s: ggml_backend_metal_init() failed\n", __func__);
        }
    }
```

and (ggml_features(tensor) & 1

```cpp


```
#endif

    if (!model.backend) {
        // fallback to CPU backend
        fprintf(stderr, "%s: using CPU backend\n", __func__);
        model.backend = ggml_backend_cpu_init();
    }

    if (!model.backend) {
        fprintf(stderr, "%s: ggml_backend_cpu_init() failed\n", __func__);
        return false;
    }

    // allocate weights buffer
    model.buffer_w = ggml_backend_alloc_buffer(model.backend, buffer_size);

    // prepare memory for the weights
    {
        const auto & hparams = model.hparams;

        const int n_embd  = hparams.n_embd;
        const int n_layer = hparams.n_layer;
        const int n_ctx   = hparams.n_ctx;
        const int n_vocab = hparams.n_vocab;

        model.layers.resize(n_layer);

        model.ln_f_g = ggml_new_tensor_1d(ctx, GGML_TYPE_F32, n_embd);
        model.ln_f_b = ggml_new_tensor_1d(ctx, GGML_TYPE_F32, n_embd);

        model.wte     = ggml_new_tensor_2d(ctx, wtype,         n_embd, n_vocab);
        model.wpe     = ggml_new_tensor_2d(ctx, GGML_TYPE_F32, n_embd, n_ctx);
        model.lm_head = ggml_new_tensor_2d(ctx, wtype,         n_embd, n_vocab);

        // map by name
        model.tensors["model/ln_f/g"] = model.ln_f_g;
        model.tensors["model/ln_f/b"] = model.ln_f_b;

        model.tensors["model/wte"]     = model.wte;
        model.tensors["model/wpe"]     = model.wpe;
        model.tensors["model/lm_head"] = model.lm_head;

        for (int i = 0; i < n_layer; ++i) {
            auto & layer = model.layers[i];

            layer.ln_1_g        = ggml_new_tensor_1d(ctx, GGML_TYPE_F32,   n_embd);
            layer.ln_1_b        = ggml_new_tensor_1d(ctx, GGML_TYPE_F32,   n_embd);

            layer.ln_2_g        = ggml_new_tensor_1d(ctx, GGML_TYPE_F32,   n_embd);
            layer.ln_2_b        = ggml_new_tensor_1d(ctx, GGML_TYPE_F32,   n_embd);

            layer.c_attn_attn_w = ggml_new_tensor_2d(ctx, wtype,           n_embd, 3*n_embd);
            layer.c_attn_attn_b = ggml_new_tensor_1d(ctx, GGML_TYPE_F32, 3*n_embd);

            layer.c_attn_proj_w = ggml_new_tensor_2d(ctx, wtype,           n_embd, n_embd);
            layer.c_attn_proj_b = ggml_new_tensor_1d(ctx, GGML_TYPE_F32,   n_embd);

            layer.c_mlp_fc_w    = ggml_new_tensor_2d(ctx, wtype,           n_embd, 4*n_embd);
            layer.c_mlp_fc_b    = ggml_new_tensor_1d(ctx, GGML_TYPE_F32, 4*n_embd);

            layer.c_mlp_proj_w  = ggml_new_tensor_2d(ctx, wtype,         4*n_embd, n_embd);
            layer.c_mlp_proj_b  = ggml_new_tensor_1d(ctx, GGML_TYPE_F32,   n_embd);

            // map by name
            model.tensors["model/h" + std::to_string(i) + "/ln_1/g"]        = layer.ln_1_g;
            model.tensors["model/h" + std::to_string(i) + "/ln_1/b"]        = layer.ln_1_b;

            model.tensors["model/h" + std::to_string(i) + "/ln_2/g"]        = layer.ln_2_g;
            model.tensors["model/h" + std::to_string(i) + "/ln_2/b"]        = layer.ln_2_b;

            model.tensors["model/h" + std::to_string(i) + "/attn/c_attn/w"] = layer.c_attn_attn_w;
            model.tensors["model/h" + std::to_string(i) + "/attn/c_attn/b"] = layer.c_attn_attn_b;

            model.tensors["model/h" + std::to_string(i) + "/attn/c_proj/w"] = layer.c_attn_proj_w;
            model.tensors["model/h" + std::to_string(i) + "/attn/c_proj/b"] = layer.c_attn_proj_b;

            model.tensors["model/h" + std::to_string(i) + "/mlp/c_fc/w"]    = layer.c_mlp_fc_w;
            model.tensors["model/h" + std::to_string(i) + "/mlp/c_fc/b"]    = layer.c_mlp_fc_b;

            model.tensors["model/h" + std::to_string(i) + "/mlp/c_proj/w"]  = layer.c_mlp_proj_w;
            model.tensors["model/h" + std::to_string(i) + "/mlp/c_proj/b"]  = layer.c_mlp_proj_b;
        }
    }

    // override the default training context with the user-provided
    model.hparams.n_ctx = n_ctx;

    // key + value memory
    {
        const auto & hparams = model.hparams;

        const int n_embd  = hparams.n_embd;
        const int n_layer = hparams.n_layer;
        const int n_ctx   = hparams.n_ctx;

        const int n_mem      = n_layer*n_ctx;
        const int n_elements = n_embd*n_mem;

        model.memory_k = ggml_new_tensor_1d(ctx, GGML_TYPE_F32, n_elements);
        model.memory_v = ggml_new_tensor_1d(ctx, GGML_TYPE_F32, n_elements);

        const size_t memory_size = ggml_nbytes(model.memory_k) + ggml_nbytes(model.memory_v);

        printf("%s: memory size = %8.2f MB, n_mem = %d\n", __func__, memory_size/1024.0/1024.0, n_mem);

        // create a backend buffer (can be in host or device memory)
        model.buffer_kv = ggml_backend_alloc_buffer(model.backend, memory_size + 256);

        // allocate the tensors into the backend buffer
        {
            ggml_allocr * alloc = ggml_allocr_new_from_buffer(model.buffer_kv);

            // this updates the pointers in the tensors to point to the correct location in the buffer
            // this is necessary since the ggml_context is .no_alloc == true
            // note that the buffer can actually be a device buffer, depending on the backend
            ggml_allocr_alloc(alloc, model.memory_k);
            ggml_allocr_alloc(alloc, model.memory_v);

            ggml_allocr_free(alloc);
        }
    }

    // load weights
    {
        ggml_allocr * alloc = ggml_allocr_new_from_buffer(model.buffer_w);

        size_t total_size = 0;

        bool has_lm_head = false;

        std::vector<char> read_buf;

        while (true) {
            int32_t n_dims;
            int32_t length;
            int32_t ttype;

            fin.read(reinterpret_cast<char *>(&n_dims), sizeof(n_dims));
            fin.read(reinterpret_cast<char *>(&length), sizeof(length));
            fin.read(reinterpret_cast<char *>(&ttype),  sizeof(ttype));

            if (fin.eof()) {
                break;
            }

            int32_t nelements = 1;
            int32_t ne[2] = { 1, 1 };
            for (int i = 0; i < n_dims; ++i) {
                fin.read(reinterpret_cast<char *>(&ne[i]), sizeof(ne[i]));
                nelements *= ne[i];
            }

            std::string name(length, 0);
            fin.read(&name[0], length);

            if (model.tensors.find(name) == model.tensors.end()) {
                fprintf(stderr, "%s: unknown tensor '%s' in model file\n", __func__, name.c_str());
                return false;
            }

            auto tensor = model.tensors[name];
            ggml_set_name(tensor, name.c_str());
            if (ggml_nelements(tensor) != nelements) {
                fprintf(stderr, "%s: tensor '%s' has wrong size in model file\n", __func__, name.c_str());
                return false;
            }

            if (tensor->ne[0] != ne[0] || tensor->ne[1] != ne[1]) {
                fprintf(stderr, "%s: tensor '%s' has wrong shape in model file: got [%d, %d], expected [%d, %d]\n",
                        __func__, name.c_str(), (int) tensor->ne[0], (int) tensor->ne[1], ne[0], ne[1]);
                return false;
            }

            // for debugging
            if (0) {
                printf("%24s - [%5d, %5d], type = %6s, %6.2f MB, %9zu bytes\n", name.c_str(), ne[0], ne[1], ggml_type_name(ggml_type(ttype)), ggml_nbytes(tensor)/1024.0/1024.0, ggml_nbytes(tensor));
            }

            const size_t bpe = ggml_type_size(ggml_type(ttype));

            if ((nelements*bpe)/ggml_blck_size(tensor->type) != ggml_nbytes(tensor)) {
                fprintf(stderr, "%s: tensor '%s' has wrong size in model file: got %zu, expected %zu\n",
                        __func__, name.c_str(), ggml_nbytes(tensor), nelements*bpe);
                return false;
            }

            ggml_allocr_alloc(alloc, tensor);

            if (ggml_backend_is_cpu  (model.backend)
```cpp

这段代码是一个 C++ 程序，定义了一个名为 "model_tensor_impl.h"。它主要负责读取模型 Tensor 中在 CPU 或 Metal（C++ GUI）backend下的数据。

主要作用是：

1. 如果模型使用了 Metal backend，那么就从模型后端直接读取数据。
2. 如果使用了 CPU backend，那么首先会从内存中读取数据，如果内存不足，就会创建一个临时缓冲区，然后从内存中逐个读取数据并将其复制到模型后端。
3. 判断模型是否支持 LM（语言模型）头部。如果支持，那么将 LM 头部存储到模型后端。
4. 更新模型大小以及 Tensor 类型。
5. 在函数结束时，关闭输入文件并输出模型大小。

具体实现包括：

1. 判断是否支持 Metal backend，如果支持，直接从模型后端读取数据。
2. 支持 CPU backend，但是需要从内存中读取数据，并创建一个临时缓冲区。
3. 支持 LM 头部，需要先从内存中读取数据，然后将其复制到模型后端。
4. 更新模型大小，需要记录 Tensor 类型并计算大小。
5. 在函数结束时，关闭输入文件并输出模型大小。


```
#ifdef GGML_USE_METAL
                || ggml_backend_is_metal(model.backend)
#endif
                ) {
                // for the CPU and Metal backend, we can read directly into the tensor
                fin.read(reinterpret_cast<char *>(tensor->data), ggml_nbytes(tensor));
            } else {
                // read into a temporary buffer first, then copy to device memory
                read_buf.resize(ggml_nbytes(tensor));
                fin.read(read_buf.data(), ggml_nbytes(tensor));
                ggml_backend_tensor_set(tensor, read_buf.data(), 0, ggml_nbytes(tensor));
            }

            // GPT-2 models share the WTE tensor as the LM head
            if (name == "model/wte" && has_lm_head == false) {
                //ggml_allocr_alloc(alloc, model.lm_head);
                //ggml_backend_tensor_copy(tensor, model.lm_head);
                model.lm_head = tensor;
            }

            if (name == "model/lm_head") {
                has_lm_head = true;
            }

            total_size += ggml_nbytes(tensor);
        }

        ggml_allocr_free(alloc);
        printf("%s: model size  = %8.2f MB\n", __func__, total_size/1024.0/1024.0);
    }

    fin.close();

    return true;
}

```cpp

This is a function definition for an incoming neural network model that uses multi-layer logistic


```
// build the computation graph
struct ggml_cgraph * gpt2_graph(
        const gpt2_model & model,
        struct ggml_allocr * allocr,
        const int n_past,
        const std::vector<gpt_vocab::id> & embd_inp) {
    const int N = embd_inp.size();

    const auto & hparams = model.hparams;

    const int n_embd  = hparams.n_embd;
    const int n_layer = hparams.n_layer;
    const int n_ctx   = hparams.n_ctx;
    const int n_head  = hparams.n_head;

    // since we are using ggml-alloc, this buffer only needs enough space to hold the ggml_tensor and ggml_cgraph structs, but not the tensor data
    static size_t buf_size = ggml_tensor_overhead()*GGML_DEFAULT_GRAPH_SIZE + ggml_graph_overhead();
    static std::vector<uint8_t> buf(buf_size);

    struct ggml_init_params params = {
        /*.mem_size   =*/ buf_size,
        /*.mem_buffer =*/ buf.data(),
        /*.no_alloc   =*/ true, // the tensors will be allocated later by ggml_allocr_alloc_graph()
    };

    struct ggml_context * ctx0 = ggml_init(params);

    struct ggml_cgraph  * gf = ggml_new_graph(ctx0);

    struct ggml_tensor * embd = ggml_new_tensor_1d(ctx0, GGML_TYPE_I32, N);
    ggml_allocr_alloc(allocr, embd);

    // avoid writing to tensors if we are only measuring the memory usage
    if (!ggml_allocr_is_measure(allocr)) {
        ggml_backend_tensor_set(embd, embd_inp.data(), 0, N*ggml_element_size(embd));
    }

    struct ggml_tensor * position = ggml_new_tensor_1d(ctx0, GGML_TYPE_I32, N);
    ggml_allocr_alloc(allocr, position);
    if (!ggml_allocr_is_measure(allocr)) {
        for (int i = 0; i < N; ++i) {
            int32_t v = n_past + i;
            ggml_backend_tensor_set(position, &v, i*sizeof(int32_t), sizeof(v));
        }
    }

    struct ggml_tensor * KQ_scale = ggml_new_tensor_1d(ctx0, GGML_TYPE_F32, 1);
    ggml_allocr_alloc(allocr, KQ_scale);
    if (!ggml_allocr_is_measure(allocr)) {
        float s = 1.0f/sqrtf(float(n_embd)/n_head);
        ggml_backend_tensor_set(KQ_scale, &s, 0, sizeof(s));
    }

    // wte + wpe
    struct ggml_tensor * inpL =
        ggml_add(ctx0,
                ggml_get_rows(ctx0, model.wte, embd),
                ggml_get_rows(ctx0, model.wpe, position));

    for (int il = 0; il < n_layer; ++il) {
        struct ggml_tensor * cur;

        // norm
        {
            // [ 768, N]
            cur = ggml_norm(ctx0, inpL, hparams.eps);

            // cur = ln_1_g*cur + ln_1_b
            // [ 768, N]
            cur = ggml_add(ctx0,
                    ggml_mul(ctx0,
                        cur,
                        model.layers[il].ln_1_g),
                    model.layers[il].ln_1_b);
        }

        // attn
        // [2304, 768] - model.layers[il].c_attn_attn_w
        // [2304,   1] - model.layers[il].c_attn_attn_b
        // [ 768,   N] - cur (in)
        // [2304,   N] - cur (out)
        //
        // cur = attn_w*cur + attn_b
        // [2304, N]
        {
            cur = ggml_mul_mat(ctx0,
                    model.layers[il].c_attn_attn_w,
                    cur);

            cur = ggml_add(ctx0,
                    cur,
                    model.layers[il].c_attn_attn_b);
        }

        // self-attention
        {
            struct ggml_tensor * Qcur = ggml_view_2d(ctx0, cur, n_embd, N, cur->nb[1], 0*sizeof(float)*n_embd);
            struct ggml_tensor * Kcur = ggml_view_2d(ctx0, cur, n_embd, N, cur->nb[1], 1*sizeof(float)*n_embd);
            struct ggml_tensor * Vcur = ggml_view_2d(ctx0, cur, n_embd, N, cur->nb[1], 2*sizeof(float)*n_embd);

            // store key and value to memory
            if (N >= 1) {
                struct ggml_tensor * k = ggml_view_1d(ctx0, model.memory_k, N*n_embd, (ggml_element_size(model.memory_k)*n_embd)*(il*n_ctx + n_past));
                struct ggml_tensor * v = ggml_view_1d(ctx0, model.memory_v, N*n_embd, (ggml_element_size(model.memory_v)*n_embd)*(il*n_ctx + n_past));

                ggml_build_forward_expand(gf, ggml_cpy(ctx0, Kcur, k));
                ggml_build_forward_expand(gf, ggml_cpy(ctx0, Vcur, v));
            }

            // Q = Qcur.contiguous().view(n_embd/n_head, n_head, N).permute(0, 2, 1, 3)
            // [64, N, 12]
            struct ggml_tensor * Q =
                ggml_permute(ctx0,
                        ggml_cpy(ctx0,
                            Qcur,
                            ggml_new_tensor_3d(ctx0, GGML_TYPE_F32, n_embd/n_head, n_head, N)),
                        0, 2, 1, 3);

            // K = Kmem.view(n_embd/n_head, n_head, n_past + N).permute(0, 2, 1, 3)
            // [64, n_past + N, 12]
            struct ggml_tensor * K =
                ggml_permute(ctx0,
                        ggml_reshape_3d(ctx0,
                            ggml_view_1d(ctx0, model.memory_k, (n_past + N)*n_embd, il*n_ctx*ggml_element_size(model.memory_k)*n_embd),
                            n_embd/n_head, n_head, n_past + N),
                        0, 2, 1, 3);

            // GG: flash attention
            //struct ggml_tensor * V =
            //    ggml_cpy(ctx0,
            //            ggml_permute(ctx0,
            //                ggml_reshape_3d(ctx0,
            //                    ggml_view_1d(ctx0, model.memory_v, (n_past + N)*n_embd, il*n_ctx*ggml_element_size(model.memory_v)*n_embd),
            //                    n_embd/n_head, n_head, n_past + N),
            //                1, 2, 0, 3),
            //            ggml_new_tensor_3d(ctx0, GGML_TYPE_F32, n_past + N, n_embd/n_head, n_head));

            //struct ggml_tensor * KQV = ggml_flash_attn(ctx0, Q, K, V, true);

            // K * Q
            // [n_past + N, N, 12]
            struct ggml_tensor * KQ = ggml_mul_mat(ctx0, K, Q);

            // KQ_scaled = KQ / sqrt(n_embd/n_head)
            // [n_past + N, N, 12]
            struct ggml_tensor * KQ_scaled =
                ggml_scale(ctx0,
                        KQ,
                        KQ_scale);

            // KQ_masked = mask_past(KQ_scaled)
            // [n_past + N, N, 12]
            struct ggml_tensor * KQ_masked = ggml_diag_mask_inf(ctx0, KQ_scaled, n_past);

            // KQ = soft_max(KQ_masked)
            // [n_past + N, N, 12]
            struct ggml_tensor * KQ_soft_max = ggml_soft_max(ctx0, KQ_masked);

            // V_trans = Vmem.view(n_embd/n_head, n_head, n_past + N).permute(1, 2, 0, 3).contiguous()
            // [n_past + N, 64, 12]
            struct ggml_tensor * V_trans =
                ggml_cpy(ctx0,
                        ggml_permute(ctx0,
                            ggml_reshape_3d(ctx0,
                                ggml_view_1d(ctx0, model.memory_v, (n_past + N)*n_embd, il*n_ctx*ggml_element_size(model.memory_v)*n_embd),
                                n_embd/n_head, n_head, n_past + N),
                            1, 2, 0, 3),
                        ggml_new_tensor_3d(ctx0, model.memory_v->type, n_past + N, n_embd/n_head, n_head));

            // KQV = transpose(V) * KQ_soft_max
            // [64, N, 12]
            struct ggml_tensor * KQV = ggml_mul_mat(ctx0, V_trans, KQ_soft_max);

            // KQV_merged = KQV.permute(0, 2, 1, 3)
            // [64, 12, N]
            struct ggml_tensor * KQV_merged = ggml_permute(ctx0, KQV, 0, 2, 1, 3);

            // cur = KQV_merged.contiguous().view(n_embd, N)
            // [768, N]
            cur = ggml_cpy(ctx0,
                    KQV_merged,
                    ggml_new_tensor_2d(ctx0, GGML_TYPE_F32, n_embd, N));
        }

        // projection
        // [ 768, 768] - model.layers[il].c_attn_proj_w
        // [ 768,   1] - model.layers[il].c_attn_proj_b
        // [ 768,   N] - cur (in)
        // [ 768,   N] - cur (out)
        //
        // cur = proj_w*cur + proj_b
        // [768, N]
        {
            cur = ggml_mul_mat(ctx0,
                    model.layers[il].c_attn_proj_w,
                    cur);

            cur = ggml_add(ctx0,
                    cur,
                    model.layers[il].c_attn_proj_b);
        }

        // add the input
        cur = ggml_add(ctx0, cur, inpL);

        struct ggml_tensor * inpFF = cur;

        // feed-forward network
        {
            // norm
            {
                cur = ggml_norm(ctx0, inpFF, hparams.eps);

                // cur = ln_2_g*cur + ln_2_b
                // [ 768, N]
                cur = ggml_add(ctx0,
                        ggml_mul(ctx0,
                            cur,
                            model.layers[il].ln_2_g),
                        model.layers[il].ln_2_b);
            }

            // fully connected
            // [3072, 768] - model.layers[il].c_mlp_fc_w
            // [3072,   1] - model.layers[il].c_mlp_fc_b
            // [ 768,   N] - cur (in)
            // [3072,   N] - cur (out)
            //
            // cur = fc_w*cur + fc_b
            // [3072, N]
            cur = ggml_mul_mat(ctx0,
                    model.layers[il].c_mlp_fc_w,
                    cur);

            cur = ggml_add(ctx0,
                    cur,
                    model.layers[il].c_mlp_fc_b);

            // GELU activation
            // [3072, N]
            cur = ggml_gelu(ctx0, cur);

            // projection
            // [ 768, 3072] - model.layers[il].c_mlp_proj_w
            // [ 768,    1] - model.layers[il].c_mlp_proj_b
            // [3072,    N] - cur (in)
            // [ 768,    N] - cur (out)
            //
            // cur = proj_w*cur + proj_b
            // [768, N]
            cur = ggml_mul_mat(ctx0,
                    model.layers[il].c_mlp_proj_w,
                    cur);

            cur = ggml_add(ctx0,
                    cur,
                    model.layers[il].c_mlp_proj_b);
        }

        // input for next layer
        inpL = ggml_add(ctx0, cur, inpFF);
    }

    // norm
    {
        // [ 768, N]
        inpL = ggml_norm(ctx0, inpL, hparams.eps);

        // inpL = ln_f_g*inpL + ln_f_b
        // [ 768, N]
        inpL = ggml_add(ctx0,
                ggml_mul(ctx0,
                    inpL,
                    model.ln_f_g),
                model.ln_f_b);
    }

    // inpL = WTE * inpL
    // [ 768, 50257] - model.lm_head
    // [ 768, N]     - inpL
    inpL = ggml_mul_mat(ctx0, model.lm_head, inpL);

    // logits -> probs
    //inpL = ggml_soft_max(ctx0, inpL);

    ggml_build_forward_expand(gf, inpL);

    ggml_free(ctx0);

    return gf;
}

```cpp

这段代码是一个名为“evaluate_transformers”的函数，它接受一个GPT2模型、一个GGML Allocator和一个计算线程数，然后对模型进行评估。

具体来说，这段代码的功能如下：

1. 初始化输入参数：包括模型的参数、计算线程数、上下文大小和之前计算得到的嵌入向量。
2. 创建一个GGML Allocator对象。
3. 创建一个GGML Graph结构体，用于存储计算过程中的信息。
4. 使用GGML Allocator对象的属性，将输入参数分配给结构体。
5. 运行计算线程，设置线程数以确保模型在多个核心上运行。
6. 如果使用CPU计算，设置线程数。
7. 运行计算后，将结果存储在输出参数中。


```
// evaluate the transformer
//
//   - model:     the model
//   - allocr:    ggml_allocr to use to allocate the compute buffer
//   - n_threads: number of threads to use
//   - n_past:    the context size so far
//   - embd_inp:  the embeddings of the tokens in the context
//   - embd_w:    the predicted logits for the next token
//
bool gpt2_eval(
        const gpt2_model & model,
        struct ggml_allocr * allocr,
        const int n_threads,
        const int n_past,
        const std::vector<gpt_vocab::id> & embd_inp,
              std::vector<float>         & embd_w) {
    const int N = embd_inp.size();

    const auto & hparams = model.hparams;

    const int n_vocab = hparams.n_vocab;

    // reset the allocator to free all the memory allocated during the previous inference
    ggml_allocr_reset(allocr);

    struct ggml_cgraph * gf = gpt2_graph(model, allocr, n_past, embd_inp);

    // allocate tensors
    ggml_allocr_alloc_graph(allocr, gf);

    // run the computation
    if (ggml_backend_is_cpu(model.backend)) {
        ggml_backend_cpu_set_n_threads(model.backend, n_threads);
    }
```cpp

这段代码是一个C语言函数，它是用C语言编写的。函数名称为“ggml_graph_compute”，它用于计算输入图“gf”中的“gf”图的计算。该函数基于GGML的金属后端。

具体来说，该函数在计算图的节点时，根据输入图的后端类型，如果后端是金属，则设置计算图中的节点数（也就是N_THREADS）为计算图节点数的两倍。然后，该函数使用ggml_backend_graph_compute函数计算输入图的计算图。

在函数内部，首先检查输入图是否是已有图（也就是“N_PAST%100”是否为0），如果是，则使用ggml_graph_print和ggml_graph_dump_dot函数打印输出图。如果不是已有图，则创建一个大小为“n_vocab”行数，“N-1”列数的输入张量，并将输入图的最后一个节点（也就是“N_THREADS”-2）的输入值读入该张量中。

最后，函数返回一个布尔值，表示计算结果是否正确。如果正确，则返回TRUE，否则返回FALSE。


```
#ifdef GGML_USE_METAL
    if (ggml_backend_is_metal(model.backend)) {
        ggml_backend_metal_set_n_cb(model.backend, n_threads);
    }
#endif
    ggml_backend_graph_compute(model.backend, gf);

    //if (n_past%100 == 0) {
    //    ggml_graph_print   (&gf);
    //    ggml_graph_dump_dot(&gf, NULL, "gpt-2.dot");
    //}

    // in this case, the output tensor is the last one in the graph
    struct ggml_tensor * inpL = gf->nodes[gf->n_nodes - 1];

    //embd_w.resize(n_vocab*N);
    //ggml_backend_tensor_get(inpL, embd_w.data(), 0, sizeof(float)*n_vocab*N);

    // return result just for the last token
    embd_w.resize(n_vocab);
    ggml_backend_tensor_get(inpL, embd_w.data(), (n_vocab*(N-1))*sizeof(float), sizeof(float)*n_vocab);

    return true;
}

```cpp

This is a C++ implementation of the text-to-seq model training loop. This code is used to train the text-to-seq model based on an input text file and a pre-trained word embeddings.

The main function is responsible for reading the input text file, processing it, and then generating the output sequence. The function takes a while loop to iterate through the input text file.

The first while loop reads the input text file and processes it. The next while loop is responsible for generating the output sequence.

The while loop uses a while loop inside the second while loop to iterate through the input text file and the embeddings. The embeddings are generated by the input text using the variable `vocab.id_to_token[id].c_str()`.

The output sequence is generated by the input embeddings and stored in the variable `embd`. The output sequence is then stored in the variable `i`.

After the while loop, the function displays the output sequence.

Finally, the function has a结尾 that reports the timing information.

The function形如：
```
// function to train the text-to-seq model
int train_model(const std::string& input_text, const std::vector<std::string>& embd_inp, std::vector<std::string>& embd, int n_past, int params.n_batch) {
   int i = 0;
   while (i < input_text.size()) {
       int k = i;
       std::vector<std::string> embd;
       while (k < embd_inp.size() && embd.size() < params.n_batch) {
           embd.push_back(vocab.id_to_token[k].c_str());
           if (int32_t(embd.size()) >= params.n_batch) {
               break;
           }
           k++;
       }
       embd.push_back(embd_inp[i].c_str());
       i++;
   }
   // display text
   for (auto id : embd) {
       printf("%s", vocab.id_to_token[id].c_str());
   }
   fflush(stdout);

   // end of text token
   if (!params.ignore_eos && embd.back() == 50256) {
       break;
   }
   // report timing
   {
       const int64_t t_main_end_us = ggml_time_us();

       printf("\n\n");
       printf("%s:     load time = %8.2f ms\n", __func__, t_load_us/1000.0f);
       printf("%s:  sample time = %8.2f ms\n", __func__, t_sample_us/1000.0f);
       printf("%s:  predict time = %8.2f ms / %.2f ms per token\n", __func__, t_predict_us/1000.0f, t_predict_us/1000.0f/n_past);
       printf("%s:    total time = %8.2f ms\n", __func__, (t_main_end_us - t_main_start_us)/1000.0f);
   }

   ggml_free(model.ctx);

   ggml_backend_buffer_free(model.buffer_w);
   ggml_backend_buffer_free(model.buffer_kv);
   ggml_backend_buffer_free(buf_compute);
   ggml_backend_free(model.backend);

   return 0;
}
```cpp
The function also has a dimension parameter `params.n_batch`, which is the batch size for the predict, and a dimension parameter `params.ignore_eos`, which is a flag to indicate whether to ignore the end-of-sequence token.

This function is used to train the text-to-seq model based on a pre-trained word embeddings using the main function:
```
int main() {
   // read input text
   std::vector<std::string> input_text;
   std::vector<std::string>> embd_inp;
   std::vector<std::string> embd;
   int n_past = 0;
```cpp


```
int main(int argc, char ** argv) {
    ggml_time_init();

    const int64_t t_main_start_us = ggml_time_us();

    gpt_params params;
    params.model = "models/gpt-2-117M/ggml-model.bin";

    if (gpt_params_parse(argc, argv, params) == false) {
        return 1;
    }

    if (params.seed < 0) {
        params.seed = time(NULL);
    }

    printf("%s: seed = %d\n", __func__, params.seed);

    std::mt19937 rng(params.seed);
    if (params.prompt.empty()) {
        params.prompt = gpt_random_prompt(rng);
    }

    int64_t t_load_us = 0;

    gpt_vocab vocab;
    gpt2_model model;

    // load the model
    {
        const int64_t t_start_us = ggml_time_us();

        if (!gpt2_model_load(params.model, model, vocab, params.n_ctx, params.n_gpu_layers)) {
            fprintf(stderr, "%s: failed to load model from '%s'\n", __func__, params.model.c_str());
            return 1;
        }

        t_load_us = ggml_time_us() - t_start_us;

        test_gpt_tokenizer(vocab, params.token_test);
    }

    // keep this buffer alive while evaluating the model
    ggml_backend_buffer_t buf_compute;

    struct ggml_allocr * allocr = NULL;
    // allocate the compute buffer
    {
         // alignment required by the backend
        size_t align = ggml_backend_get_alignment(model.backend);
        allocr = ggml_allocr_new_measure(align);

        // create the worst case graph for memory usage estimation
        int n_tokens = std::min(model.hparams.n_ctx, params.n_batch);
        int n_past = model.hparams.n_ctx - n_tokens;
        struct ggml_cgraph * gf = gpt2_graph(model, allocr, n_past, std::vector<gpt_vocab::id>(n_tokens, 0));

        // compute the required memory
        size_t mem_size = ggml_allocr_alloc_graph(allocr, gf);

        // recreate the allocator with the required memory
        ggml_allocr_free(allocr);
        buf_compute = ggml_backend_alloc_buffer(model.backend, mem_size);
        allocr = ggml_allocr_new_from_buffer(buf_compute);

        fprintf(stderr, "%s: compute buffer size: %.2f MB\n", __func__, mem_size/1024.0/1024.0);
    }

    int n_past = 0;

    int64_t t_sample_us  = 0;
    int64_t t_predict_us = 0;

    std::vector<float> logits;

    // tokenize the prompt
    std::vector<gpt_vocab::id> embd_inp = ::gpt_tokenize(vocab, params.prompt);

    params.n_predict = std::min(params.n_predict, model.hparams.n_ctx - (int) embd_inp.size());

    printf("%s: prompt: '%s'\n", __func__, params.prompt.c_str());
    printf("%s: number of tokens in prompt = %zu, first 8 tokens: ", __func__, embd_inp.size());
    for (int i = 0; i < std::min(8, (int) embd_inp.size()); i++) {
        printf("%d ", embd_inp[i]);
    }
    printf("\n\n");

    // submit the input prompt token-by-token
    // this reduces the memory usage during inference, at the cost of a bit of speed at the beginning
    std::vector<gpt_vocab::id> embd;

    for (size_t i = embd.size(); i < embd_inp.size() + params.n_predict; i++) {
        // predict
        if (embd.size() > 0) {
            const int64_t t_start_us = ggml_time_us();

            if (!gpt2_eval(model, allocr, params.n_threads, n_past, embd, logits)) {
                printf("Failed to predict\n");
                return 1;
            }

            t_predict_us += ggml_time_us() - t_start_us;
        }

        n_past += embd.size();
        embd.clear();

        if (i >= embd_inp.size()) {
            // sample next token
            const int   top_k = params.top_k;
            const float top_p = params.top_p;
            const float temp  = params.temp;

            const int n_vocab = model.hparams.n_vocab;

            gpt_vocab::id id = 0;

            {
                const int64_t t_start_sample_us = ggml_time_us();

                id = gpt_sample_top_k_top_p(vocab, logits.data() + (logits.size() - n_vocab), top_k, top_p, temp, rng);

                t_sample_us += ggml_time_us() - t_start_sample_us;
            }

            // add it to the context
            embd.push_back(id);
        } else {
            // if here, it means we are still processing the input prompt
            for (size_t k = i; k < embd_inp.size(); k++) {
                embd.push_back(embd_inp[k]);
                if (int32_t(embd.size()) >= params.n_batch) {
                    break;
                }
            }
            i += embd.size() - 1;
        }

        // display text
        for (auto id : embd) {
            printf("%s", vocab.id_to_token[id].c_str());
        }
        fflush(stdout);

        // end of text token
        if (!params.ignore_eos && embd.back() == 50256) {
            break;
        }
    }

    // report timing
    {
        const int64_t t_main_end_us = ggml_time_us();

        printf("\n\n");
        printf("%s:     load time = %8.2f ms\n", __func__, t_load_us/1000.0f);
        printf("%s:   sample time = %8.2f ms\n", __func__, t_sample_us/1000.0f);
        printf("%s:  predict time = %8.2f ms / %.2f ms per token\n", __func__, t_predict_us/1000.0f, t_predict_us/1000.0f/n_past);
        printf("%s:    total time = %8.2f ms\n", __func__, (t_main_end_us - t_main_start_us)/1000.0f);
    }

    ggml_free(model.ctx);

    ggml_backend_buffer_free(model.buffer_w);
    ggml_backend_buffer_free(model.buffer_kv);
    ggml_backend_buffer_free(buf_compute);
    ggml_backend_free(model.backend);

    return 0;
}

```cpp

# `examples/gpt-2/main-batched.cpp`

这段代码是一个通用的数学库，名为"ggml"，它提供了C++和C一样的接口，用于与各种数学图形和算法进行交互。它包含了一些ggml头文件以及两个动态内存分配函数ggml_alloc和ggml_free，用于初始化和清理内存。

接下来，它通过嵌套包含了一些头文件，这些头文件包含了要使用ggml的数学库的具体功能。如果ggml_use_cuda和ggml_use_metal被定义为真，那么它将包含相应的cuda和metal库。

最后，它还包含一个通用的common.h头文件，它定义了一些通用的函数和数据结构，用于所有用户。

总的来说，这段代码是一个用于提供C++和C程序所需功能的通用数学库，可用于许多不同的计算机。


```
#include "ggml/ggml.h"
#include "ggml/ggml-alloc.h"
#include "ggml/ggml-backend.h"

#ifdef GGML_USE_CUBLAS
#include "ggml-cuda.h"
#endif

#ifdef GGML_USE_METAL
#include "ggml-metal.h"
#endif

#include "common.h"
#include "common-ggml.h"

```cpp

这段代码是一个C语言程序，它包括了一些标准库头文件（stdio.h, stdc++, cmath, cassert, etc.），以及一个自定义的ggml_log_callback_default函数。ggml_log_callback_default函数是一个I/O事件的回调函数，用于处理从标准输出（通常是键盘或鼠标输入）传递给程序的日志信息。

程序的主要作用是读取用户输入的日志信息并输出到标准输出。用户输入的日志信息被存储在std::string类型的一个map中，而程序会遍历map中的所有键值对，将每个键的值作为参数传递给ggml_log_callback_default函数，然后函数将返回的void类型的返回值忽略不计。

ggml_log_callback_default函数是一个宏定义，它包括两个函数：void类型的level和void类型的user_data。level函数是一个整数类型的函数，用于存储接收到的日志信息级别，而user_data是一个void类型的函数，用于存储接收到的日志信息用户数据。函数内部没有实现任何具体的逻辑，只是通过fflush函数将接收到的日志信息写入到标准输出流（通常是屏幕）中。

这段代码的作用是读取用户输入的日志信息并输出到屏幕，然后将其存储在地图中。


```
#include <cassert>
#include <cmath>
#include <cstdio>
#include <cstring>
#include <fstream>
#include <map>
#include <set>
#include <string>
#include <vector>

#if defined(_MSC_VER)
#pragma warning(disable: 4244 4267) // possible loss of data
#endif

static void ggml_log_callback_default(ggml_log_level level, const char * text, void * user_data) {
    (void) level;
    (void) user_data;
    fputs(text, stderr);
    fflush(stderr);
}

```cpp

这段代码定义了两个整型变量 gpt2_pos 和 gpt2_seq_id，它们都被定义为 int32_t 类型。

接着定义了一个名为 gpt2_hparams 的结构体，它包含了影响 GPT-2 模型的各种参数，包括词汇量 (n_vocab)、上下文长度 (n_ctx)、嵌入层维度 (n_embd)、注意力头数 (n_head)、层数 (n_layer) 和类型 (ftype)，以及一个浮点数 eps，用于防止除数为零。

接着定义了一个名为 gpt2_layer 的结构体，它包含了 GPT-2 模型中的各种组件，如注意力 (attention)、解码器 (decoder)、自注意力层 (self_attn_ layer)、多层感知器 (mlp) 等。这些组件的计算图中的参数都被用 Ggml 类型表示。

注意，在函数内部，还定义了一个名为 c_attn_attn_w 和 c_attn_attn_b 的浮点数变量，它们都被 Ggml 类型表示，并且在计算图中的参数都被指向这两个向量。


```
typedef int32_t gpt2_pos;
typedef int32_t gpt2_seq_id;

// default hparams (GPT-2 117M)
struct gpt2_hparams {
    int32_t n_vocab = 50257;
    int32_t n_ctx   = 1024;
    int32_t n_embd  = 768;
    int32_t n_head  = 12;
    int32_t n_layer = 12;
    int32_t ftype   = 1;
    float   eps     = 1e-5f;
};

struct gpt2_layer {
    // normalization
    struct ggml_tensor * ln_1_g;
    struct ggml_tensor * ln_1_b;

    struct ggml_tensor * ln_2_g;
    struct ggml_tensor * ln_2_b;

    // attention
    struct ggml_tensor * c_attn_attn_w;
    struct ggml_tensor * c_attn_attn_b;

    struct ggml_tensor * c_attn_proj_w;
    struct ggml_tensor * c_attn_proj_b;

    // mlp
    struct ggml_tensor * c_mlp_fc_w;
    struct ggml_tensor * c_mlp_fc_b;

    struct ggml_tensor * c_mlp_proj_w;
    struct ggml_tensor * c_mlp_proj_b;
};

```cpp

这段代码定义了两个结构体：`gpt2_kv_cell` 和 `gpt2_kv_cache`。它们的目的是在训练期间对GPT2模型的参数进行缓存，以避免在每次运行时都对模型进行重新初始化。

`gpt2_kv_cell` 结构体包含一个位置 `pos` 和一个大小为 `delta` 的变化量。它还包含一个 `seq_id` 集合，用于存储已经过时的序列ID。这个结构体有以下方法：

* `has_seq_id`：用于检查给定的 `seq_id` 是否存在于 `seq_id` 集合中。
* `pos`：获取 `gpt2_pos`。
* `delta`：设置 `gpt2_pos`。
* `seq_id`：将 `seq_id` 存储在 `seq_id` 集合中。

`gpt2_kv_cache` 结构体包含一个用于存储键值对数据的内存数组 `k` 和一个用于存储值数据的内存数组 `v`。它还包含一个指向当前缓存区大小 `size` 的指针。在构建每个图形模型之前，`gpt2_kv_cache` 会计算出要存储的参数数量 `n`。它还包含一个用于存储已经过时的参数的指针，以及一个用于存储用于缓存区内的数据变更通知的指针。这个结构体有以下方法：

* `add_seq_id`：将指定的 `seq_id` 添加到已过时的参数列表中。
* `remove_seq_id`：从已过时的参数列表中删除指定的 `seq_id`。
* `get_seq_ids`：返回一个 `std::set<gpt2_seq_id>` 用于存储已经过时的参数列表。
* `not_seq_ids`：返回一个 `std::vector<gpt2_seq_id>` 用于存储不属于 `seq_id` 集合的参数列表。
* `flush_cache`：将缓存区中的所有数据刷到磁盘。
* `load_cache`：从磁盘读取缓存区中的数据，并更新 `size` 和 `n`。然后，它将调用 `flush_cache` 来将所有数据刷到磁盘。


```
struct gpt2_kv_cell {
    gpt2_pos pos   = -1;
    gpt2_pos delta = 0;

    std::set<gpt2_seq_id> seq_id;

    bool has_seq_id(const gpt2_seq_id & id) const {
        return seq_id.find(id) != seq_id.end();
    }
};

struct gpt2_kv_cache {
    // key + value memory
    struct ggml_tensor * k;
    struct ggml_tensor * v;
    //

    uint32_t head = 0;
    uint32_t size = 0;

    // computed before each graph build
    uint32_t n = 0;

    std::vector<gpt2_kv_cell> cells;

    ggml_backend_buffer_t buffer;
};

```cpp

这是一个GPT-2模型结构体，它包含了模型所需的各种组件。

成员变量：

* `hparams`：GPT-2硬件参数结构体，包含了GPT-2模型的设置。
* `ln_f_g` 和 `ln_f_b`：两个NORMALIZATION层的中间隐藏状态。
* `wte`：文本定位嵌入向量。
* `wpe`：文本编码嵌入向量。
* `lm_head`：语言模型头。
* `layers`：模型层。
* `kv_cache`：缓存键值对。
* `ctx`：上下文。
* `backend`：当前正在使用的GPU或CPU后端。
* `buffer_w`：当前正在缓冲的数据缓冲区。
* `tensors`：存储各种不同类型的模型组件。

函数：

* `init()`：初始化函数，用于设置GPT-2模型的参数。
* `destroy()`：清理函数，用于释放GPT-2模型。
* `init_buffer()`：初始化缓冲区函数，用于分配内存并初始化数据缓冲区。
* `forward_ pass()`：前向传播函数，用于执行一次前向推理计算。
* `backward_ pass()`：反向传播函数，用于执行一次反向传播计算。


```
struct gpt2_model {
    gpt2_hparams hparams;

    // normalization
    struct ggml_tensor * ln_f_g;
    struct ggml_tensor * ln_f_b;

    struct ggml_tensor * wte;     // position embedding
    struct ggml_tensor * wpe;     //    token embedding
    struct ggml_tensor * lm_head; // language model head

    std::vector<gpt2_layer> layers;

    gpt2_kv_cache kv_cache;

    struct ggml_context * ctx;

    ggml_backend_t backend = NULL;

    ggml_backend_buffer_t buffer_w;

    std::map<std::string, struct ggml_tensor *> tensors;
};

```cpp

这段代码定义了一个名为 `gpt2_batch` 的结构体，用于表示一个 gpt2 批次的输入数据。该结构体包含一个 `n_tokens` 成员变量，用于表示输入数据中 token 的数量。同时，该结构体还包含其他成员变量，如 `token`、`embd`、`pos`、`seq_id` 和 `logits`，它们分别用于表示输入数据中的每个 token 所在的序列、token 自身的特征值、token 对应的序列 ID 和二进制逻辑值，等等。

这个结构体是一个输入数据结构，它在训练 gpt2 模型时使用，可以是一个批次的输入数据。它所代表的意义是，对于每一个批次的输入数据，结构体 `gpt2_batch` 包含了该批次的 token 数量、token 自身的特征值、token 所在的序列 ID 和二进制逻辑值等信息。


```
// Input data for gpt2_decode
// A gpt2_batch object can contain input about one or many sequences
// The provided arrays (i.e. token, embd, pos, etc.) must have size of n_tokens
//
// - token  : the token ids of the input (used when embd is NULL)
// - embd   : token embeddings (i.e. float vector of size n_embd) (used when token is NULL)
// - pos    : the positions of the respective token in the sequence
// - seq_id : the sequence to which the respective token belongs
// - logits : if zero, the logits for the respective token will not be output
//
struct gpt2_batch {
    int32_t n_tokens = -1;

    gpt_vocab::id  * token  = {};
    float          * embd   = {};
    gpt2_pos       * pos    = {};
    gpt2_seq_id    * seq_id = {};
    int8_t         * logits = {};
};

```cpp

以下是PyTorch实现的代码。

```python
#include <torch_geometry.h>
#include <geometry_mkfs.h>
#include <geometry_pics.h>
#include <max.h>

#define GGML_HEADER_SIZE 4096
#define GGML_CAPS_SIZE   1024

static_device::Module module;

namespace cuda {
using Geometry = torch::geometry::Geometry;
using营养价值 = torch::math::sin<float>();
using static:：申明<float> align;
using i32 = std::int32_t;

// 创建申明
static void create_buffer(std::vector<float> &buffer, size_t size, int n_layer) {
   static size_t buffer_size = 0;

   buffer_size += size;
   buffer.push_back((size_t) buffer_size / n_layer);
}

// 初始化ggml tensor
static std::vector<ggml_tensor> initialize_tensor(
   std::vector<float> &buffer, size_t size, int n_layer) {
   std::vector<ggml_tensor> tensors;
   static size_t buffer_size = 0;

   create_buffer(tensors, size, n_layer);
   buffer_size += size;

   int i = 0;
   for (int j = 0; j < n_layer; j++) {
       size_t pos = i * buffer_size + j * buffer_size / n_layer;
       float f = (size_t) buffer[pos];
       align<float>(f).clamp(0.0f, 1.0f);
       tensors.push_back({i, f});
       i++;
   }

   return tensors;
}

// 创建geometry
static Geometry create_geometry(
   std::vector<ggml_tensor> &tensors, size_t size, int n_layer) {
   Geometry geometry;
   geometry.ptr = new GeometryCaps<ggml_tensor_overhead<>>((int) size);
   geometry.make_host = true;
   geometry.host = (void) tensors.data;
   geometry.layout = GeometryCapsLayout<ggml_tensor_overhead<>>::kU;
   geometry.device = device;
   geometry.cuda_device = device;
   geometry.data_layout = GeometryAxisAlignment::kX;
   geometry.中铁丝数 = static_cast<int32_t>(std::min(std::max<int32_t>((size_t) tensors.size), 8));
   Geometry().add_functional(tensors, "create_geometry_tensor", "create_geometry");
   Geometry().add_gradient("create_geometry_gradient", tensors, "create_geometry");
   return geometry;
}

// 创建geometry模塊
static std::vector<Geometry> create_geometry_模塊(
   std::vector<ggml_tensor> &tensors, size_t size, int n_layer) {
   std::vector<Geometry> modules;
   Geometry geometry;
   geometry.ptr = new GeometryCaps<ggml_tensor_overhead<>>((int) size);
   geometry.make_host = true;
   geometry.host = (void) tensors.data;
   geometry.layout = GeometryCapsLayout<ggml_tensor_overhead<>>::kU;
   geometry.device = device;
   geometry.cuda_device = device;
   geometry.data_layout = GeometryAxisAlignment::kX;
   geometry.中铁丝数 = static_cast<int32_t>(std::min(std::max<int32_t>((size_t) tensors.size), 8));
   Geometry().add_functional(tensors, "create_geometry_tensor", "create_geometry");
   Geometry().add_gradient("create_geometry_gradient", tensors, "create_geometry");
   modules.push_back(geometry);
   static size_t buffer_size = 0;
   for (int layer = 0; layer < n_layer; layer++) {
       size_t pos = i * buffer_size + layer * buffer_size / n_layer;
       size_t offset = (layer + 1) * buffer_size / n_layer;
       Geometry module;
       module.ptr = new GeometryCaps<ggml_tensor_overhead<>>((int) size);
       module.make_host = true;
       module.host = (void) tensors.data + pos + offset;
       module.layout = GeometryCapsLayout<ggml_tensor_overhead<>>::kU;
       module.device = device;
       module.cuda_device = device;
       module.data_layout = GeometryAxisAlignment::kX;
       module.中铁丝数 = static_cast<int32_t>(std::min(std::max<int32_t>((size_t) tensors.size), 8));
       Geometry().add_functional(tensors, "create_geometry_tensor", "create_geometry");
       Geometry().add_gradient("create_geometry_gradient", tensors, "create_geometry");
       modules.push_back(module);
       buffer_size += offset + layer * buffer_size / n_layer;
   }
   return modules;
}

// 创建算法的geometry
static std::vector<Geometry> create_geometry_model(
   std::vector<ggml_tensor> &tensors, size_t size, int n_layer) {
   std::vector<Geometry> modules;
   Geometry geometry;
   geometry.ptr = new GeometryCaps<ggml_tensor_overhead<>>((int) size);
   geometry.make_host = true;
   geometry.host = (void) tensors.data;
   geometry.layout = GeometryCapsLayout<ggml_tensor_overhead<>>::kU;
   geometry.device = device;
   geometry.cuda_device =


```cpp
// load the model's weights from a file
bool gpt2_model_load(const std::string & fname, gpt2_model & model, gpt_vocab & vocab, int n_ctx, int n_gpu_layers) {
    printf("%s: loading model from '%s'\n", __func__, fname.c_str());

    auto fin = std::ifstream(fname, std::ios::binary);
    if (!fin) {
        fprintf(stderr, "%s: failed to open '%s'\n", __func__, fname.c_str());
        return false;
    }

    // verify magic
    {
        uint32_t magic;
        fin.read((char *) &magic, sizeof(magic));
        if (magic != GGML_FILE_MAGIC) {
            fprintf(stderr, "%s: invalid model file '%s' (bad magic)\n", __func__, fname.c_str());
            return false;
        }
    }

    // load hparams
    {
        auto & hparams = model.hparams;

        fin.read((char *) &hparams.n_vocab, sizeof(hparams.n_vocab));
        fin.read((char *) &hparams.n_ctx,   sizeof(hparams.n_ctx));
        fin.read((char *) &hparams.n_embd,  sizeof(hparams.n_embd));
        fin.read((char *) &hparams.n_head,  sizeof(hparams.n_head));
        fin.read((char *) &hparams.n_layer, sizeof(hparams.n_layer));
        fin.read((char *) &hparams.ftype,   sizeof(hparams.ftype));

        const int32_t qntvr = hparams.ftype / GGML_QNT_VERSION_FACTOR;

        printf("%s: n_vocab = %d\n", __func__, hparams.n_vocab);
        printf("%s: n_ctx   = %d\n", __func__, hparams.n_ctx);
        printf("%s: n_embd  = %d\n", __func__, hparams.n_embd);
        printf("%s: n_head  = %d\n", __func__, hparams.n_head);
        printf("%s: n_layer = %d\n", __func__, hparams.n_layer);
        printf("%s: ftype   = %d\n", __func__, hparams.ftype);
        printf("%s: qntvr   = %d\n", __func__, qntvr);

        hparams.ftype %= GGML_QNT_VERSION_FACTOR;
    }

    // load vocab
    {
        int32_t n_vocab = 0;
        fin.read((char *) &n_vocab, sizeof(n_vocab));

        if (n_vocab != model.hparams.n_vocab) {
            fprintf(stderr, "%s: invalid model file '%s' (bad vocab size %d != %d)\n",
                    __func__, fname.c_str(), n_vocab, model.hparams.n_vocab);
            return false;
        }

        std::string word;
        std::vector<char> buf(128);

        for (int i = 0; i < n_vocab; i++) {
            uint32_t len;
            fin.read((char *) &len, sizeof(len));

            buf.resize(len);
            fin.read((char *) buf.data(), len);
            word.assign(buf.data(), len);

            vocab.token_to_id[word] = i;
            vocab.id_to_token[i] = word;
        }
    }

    // for the big tensors, we have the option to store the data in 16-bit floats or quantized
    // in order to save memory and also to speed up the computation
    ggml_type wtype = ggml_ftype_to_ggml_type((ggml_ftype) (model.hparams.ftype));
    if (wtype == GGML_TYPE_COUNT) {
        fprintf(stderr, "%s: invalid model file '%s' (bad ftype value %d)\n",
                __func__, fname.c_str(), model.hparams.ftype);
        return false;
    }

    auto & ctx = model.ctx;

    size_t buffer_size = 0;

    {
        const auto & hparams = model.hparams;

        const int n_embd  = hparams.n_embd;
        const int n_layer = hparams.n_layer;
        const int n_ctx   = hparams.n_ctx;
        const int n_vocab = hparams.n_vocab;

        buffer_size += n_embd*ggml_type_sizef(GGML_TYPE_F32); // ln_f_g
        buffer_size += n_embd*ggml_type_sizef(GGML_TYPE_F32); // ln_f_b

        buffer_size += n_vocab*n_embd*ggml_type_sizef(wtype);         // wte
        buffer_size +=   n_ctx*n_embd*ggml_type_sizef(GGML_TYPE_F32); // wpe
        buffer_size += n_vocab*n_embd*ggml_type_sizef(wtype);         // lm_head

        buffer_size += n_layer*(n_embd*ggml_type_sizef(GGML_TYPE_F32)); // ln_1_g
        buffer_size += n_layer*(n_embd*ggml_type_sizef(GGML_TYPE_F32)); // ln_1_b

        buffer_size += n_layer*(n_embd*ggml_type_sizef(GGML_TYPE_F32)); // ln_2_g
        buffer_size += n_layer*(n_embd*ggml_type_sizef(GGML_TYPE_F32)); // ln_2_b

        buffer_size += n_layer*(3*n_embd*n_embd*ggml_type_sizef(wtype));         // c_attn_attn_w
        buffer_size += n_layer*(       3*n_embd*ggml_type_sizef(GGML_TYPE_F32)); // c_attn_attn_b

        buffer_size += n_layer*(n_embd*n_embd*ggml_type_sizef(wtype));           // c_attn_proj_w
        buffer_size += n_layer*(       n_embd*ggml_type_sizef(GGML_TYPE_F32));   // c_attn_proj_b

        buffer_size += n_layer*(4*n_embd*n_embd*ggml_type_sizef(wtype));         // c_mlp_fc_w
        buffer_size += n_layer*(       4*n_embd*ggml_type_sizef(GGML_TYPE_F32)); // c_mlp_fc_b

        buffer_size += n_layer*(4*n_embd*n_embd*ggml_type_sizef(wtype));         // c_mlp_proj_w
        buffer_size += n_layer*(         n_embd*ggml_type_sizef(GGML_TYPE_F32)); // c_mlp_proj_b

        buffer_size += (6 + 12*n_layer)*128; // alignment overhead

        printf("%s: ggml tensor size    = %d bytes\n", __func__, (int) sizeof(ggml_tensor));
        printf("%s: backend buffer size = %6.2f MB\n", __func__, buffer_size/(1024.0*1024.0));
    }

    // create the ggml context
    {
        size_t n_tensors = 2 + 6 + 12*model.hparams.n_layer;
        struct ggml_init_params params = {
            /*.mem_size   =*/ ggml_tensor_overhead() * n_tensors,
            /*.mem_buffer =*/ NULL,
            /*.no_alloc   =*/ true,
        };

        model.ctx = ggml_init(params);
        if (!model.ctx) {
            fprintf(stderr, "%s: ggml_init() failed\n", __func__);
            return false;
        }
    }

    // initialize the backend
```

这段代码是用来在GGML中根据使用CUDA或Metal计算引擎来选择使用哪种计算引擎的。

首先，在#ifdef GGML_USE_CUBLAS和#ifdef GGML_USE_METAL这两个条件语句中，判断当前是否支持使用CUDA或Metal计算引擎。如果当前支持使用CUDA计算引擎，那么会执行if语句，判断是否已经初始化好了。如果当前不支持使用CUDA计算引擎，那么就会执行if语句，判断是否初始化成功。

如果当前支持使用Metal计算引擎，那么会执行if语句，判断是否已经初始化好了。如果当前不支持使用Metal计算引擎，那么就会执行if语句，判断是否初始化成功。

最后，如果当前支持使用CUDA计算引擎，但是已经初始化失败，那么就会输出错误信息。


```cpp
#ifdef GGML_USE_CUBLAS
    if (n_gpu_layers > 0) {
        fprintf(stderr, "%s: using CUDA backend\n", __func__);
        model.backend = ggml_backend_cuda_init();
        if (!model.backend) {
            fprintf(stderr, "%s: ggml_backend_cuda_init() failed\n", __func__);
        }
    }
#endif

#ifdef GGML_USE_METAL
    if (n_gpu_layers > 0) {
        fprintf(stderr, "%s: using Metal backend\n", __func__);
        ggml_metal_log_set_callback(ggml_log_callback_default, nullptr);
        model.backend = ggml_backend_metal_init();
        if (!model.backend) {
            fprintf(stderr, "%s: ggml_backend_metal_init() failed\n", __func__);
        }
    }
```

) {
           // Compile the model
           if (!model.compile(model.filename, "CC", "")) {
               fprintf(stderr, "%s: failed to compile the model\n", __func__);
               return false;
           }
       }

       // Create an output tensor
       Tensor* out_tensor = new Tensor(model.tensors);
       if (!out_tensor) {
           fprintf(stderr, "%s: failed to allocate memory for output tensor\n", __func__);
           return false;
       }

       // Set the output tensor's data to the input tensor
       if (model.compile(out_tensor->filename, "F", out_tensor->data) != true) {
           fprintf(stderr, "%s: failed to compile the output tensor\n", __func__);
           return false;
       }

       // Save the output tensor
       if (!model.save(out_tensor->filename, out_tensor->data)) {
           fprintf(stderr, "%s: failed to save the output tensor\n", __func__);
           return false;
       }

       // Free memory
       model.free(out_tensor);

       return true;
   }
}


```cpp
#endif

    if (!model.backend) {
        // fallback to CPU backend
        fprintf(stderr, "%s: using CPU backend\n", __func__);
        model.backend = ggml_backend_cpu_init();
    }

    if (!model.backend) {
        fprintf(stderr, "%s: ggml_backend_cpu_init() failed\n", __func__);
        return false;
    }

    // allocate weights buffer
    model.buffer_w = ggml_backend_alloc_buffer(model.backend, buffer_size);

    // prepare memory for the weights
    {
        const auto & hparams = model.hparams;

        const int n_embd  = hparams.n_embd;
        const int n_layer = hparams.n_layer;
        const int n_ctx   = hparams.n_ctx;
        const int n_vocab = hparams.n_vocab;

        model.layers.resize(n_layer);

        model.ln_f_g = ggml_new_tensor_1d(ctx, GGML_TYPE_F32, n_embd);
        model.ln_f_b = ggml_new_tensor_1d(ctx, GGML_TYPE_F32, n_embd);

        model.wte     = ggml_new_tensor_2d(ctx, wtype,         n_embd, n_vocab);
        model.wpe     = ggml_new_tensor_2d(ctx, GGML_TYPE_F32, n_embd, n_ctx);
        model.lm_head = ggml_new_tensor_2d(ctx, wtype,         n_embd, n_vocab);

        // map by name
        model.tensors["model/ln_f/g"] = model.ln_f_g;
        model.tensors["model/ln_f/b"] = model.ln_f_b;

        model.tensors["model/wte"]     = model.wte;
        model.tensors["model/wpe"]     = model.wpe;
        model.tensors["model/lm_head"] = model.lm_head;

        for (int i = 0; i < n_layer; ++i) {
            auto & layer = model.layers[i];

            layer.ln_1_g        = ggml_new_tensor_1d(ctx, GGML_TYPE_F32,   n_embd);
            layer.ln_1_b        = ggml_new_tensor_1d(ctx, GGML_TYPE_F32,   n_embd);

            layer.ln_2_g        = ggml_new_tensor_1d(ctx, GGML_TYPE_F32,   n_embd);
            layer.ln_2_b        = ggml_new_tensor_1d(ctx, GGML_TYPE_F32,   n_embd);

            layer.c_attn_attn_w = ggml_new_tensor_2d(ctx, wtype,           n_embd, 3*n_embd);
            layer.c_attn_attn_b = ggml_new_tensor_1d(ctx, GGML_TYPE_F32, 3*n_embd);

            layer.c_attn_proj_w = ggml_new_tensor_2d(ctx, wtype,           n_embd, n_embd);
            layer.c_attn_proj_b = ggml_new_tensor_1d(ctx, GGML_TYPE_F32,   n_embd);

            layer.c_mlp_fc_w    = ggml_new_tensor_2d(ctx, wtype,           n_embd, 4*n_embd);
            layer.c_mlp_fc_b    = ggml_new_tensor_1d(ctx, GGML_TYPE_F32, 4*n_embd);

            layer.c_mlp_proj_w  = ggml_new_tensor_2d(ctx, wtype,         4*n_embd, n_embd);
            layer.c_mlp_proj_b  = ggml_new_tensor_1d(ctx, GGML_TYPE_F32,   n_embd);

            // map by name
            model.tensors["model/h" + std::to_string(i) + "/ln_1/g"]        = layer.ln_1_g;
            model.tensors["model/h" + std::to_string(i) + "/ln_1/b"]        = layer.ln_1_b;

            model.tensors["model/h" + std::to_string(i) + "/ln_2/g"]        = layer.ln_2_g;
            model.tensors["model/h" + std::to_string(i) + "/ln_2/b"]        = layer.ln_2_b;

            model.tensors["model/h" + std::to_string(i) + "/attn/c_attn/w"] = layer.c_attn_attn_w;
            model.tensors["model/h" + std::to_string(i) + "/attn/c_attn/b"] = layer.c_attn_attn_b;

            model.tensors["model/h" + std::to_string(i) + "/attn/c_proj/w"] = layer.c_attn_proj_w;
            model.tensors["model/h" + std::to_string(i) + "/attn/c_proj/b"] = layer.c_attn_proj_b;

            model.tensors["model/h" + std::to_string(i) + "/mlp/c_fc/w"]    = layer.c_mlp_fc_w;
            model.tensors["model/h" + std::to_string(i) + "/mlp/c_fc/b"]    = layer.c_mlp_fc_b;

            model.tensors["model/h" + std::to_string(i) + "/mlp/c_proj/w"]  = layer.c_mlp_proj_w;
            model.tensors["model/h" + std::to_string(i) + "/mlp/c_proj/b"]  = layer.c_mlp_proj_b;
        }
    }

    // override the default training context with the user-provided
    model.hparams.n_ctx = n_ctx;

    // key + value memory
    {
        const auto & hparams = model.hparams;

        const int n_embd  = hparams.n_embd;
        const int n_layer = hparams.n_layer;
        const int n_ctx   = hparams.n_ctx;

        const int n_mem      = n_layer*n_ctx;
        const int n_elements = n_embd*n_mem;

        model.kv_cache.k = ggml_new_tensor_1d(ctx, GGML_TYPE_F32, n_elements);
        model.kv_cache.v = ggml_new_tensor_1d(ctx, GGML_TYPE_F32, n_elements);

        model.kv_cache.head      = 0;
        model.kv_cache.size      = n_ctx;

        model.kv_cache.cells.resize(n_ctx);

        const size_t memory_size = ggml_nbytes(model.kv_cache.k) + ggml_nbytes(model.kv_cache.v);

        printf("%s: memory size = %8.2f MB, n_mem = %d\n", __func__, memory_size/1024.0/1024.0, n_mem);

        // create a backend buffer (can be in host or device memory)
        model.kv_cache.buffer = ggml_backend_alloc_buffer(model.backend, memory_size + 256);

        // allocate the tensors into the backend buffer
        {
            ggml_allocr * alloc = ggml_allocr_new_from_buffer(model.kv_cache.buffer);

            // this updates the pointers in the tensors to point to the correct location in the buffer
            // this is necessary since the ggml_context is .no_alloc == true
            // note that the buffer can actually be a device buffer, depending on the backend
            ggml_allocr_alloc(alloc, model.kv_cache.k);
            ggml_allocr_alloc(alloc, model.kv_cache.v);

            ggml_allocr_free(alloc);
        }
    }

    // load weights
    {
        ggml_allocr * alloc = ggml_allocr_new_from_buffer(model.buffer_w);

        size_t total_size = 0;

        bool has_lm_head = false;

        std::vector<char> read_buf;

        while (true) {
            int32_t n_dims;
            int32_t length;
            int32_t ttype;

            fin.read(reinterpret_cast<char *>(&n_dims), sizeof(n_dims));
            fin.read(reinterpret_cast<char *>(&length), sizeof(length));
            fin.read(reinterpret_cast<char *>(&ttype),  sizeof(ttype));

            if (fin.eof()) {
                break;
            }

            int32_t nelements = 1;
            int32_t ne[2] = { 1, 1 };
            for (int i = 0; i < n_dims; ++i) {
                fin.read(reinterpret_cast<char *>(&ne[i]), sizeof(ne[i]));
                nelements *= ne[i];
            }

            std::string name(length, 0);
            fin.read(&name[0], length);

            if (model.tensors.find(name) == model.tensors.end()) {
                fprintf(stderr, "%s: unknown tensor '%s' in model file\n", __func__, name.c_str());
                return false;
            }

            auto tensor = model.tensors[name];
            ggml_set_name(tensor, name.c_str());
            if (ggml_nelements(tensor) != nelements) {
                fprintf(stderr, "%s: tensor '%s' has wrong size in model file\n", __func__, name.c_str());
                return false;
            }

            if (tensor->ne[0] != ne[0] || tensor->ne[1] != ne[1]) {
                fprintf(stderr, "%s: tensor '%s' has wrong shape in model file: got [%d, %d], expected [%d, %d]\n",
                        __func__, name.c_str(), (int) tensor->ne[0], (int) tensor->ne[1], ne[0], ne[1]);
                return false;
            }

            // for debugging
            if (0) {
                printf("%24s - [%5d, %5d], type = %6s, %6.2f MB, %9zu bytes\n", name.c_str(), ne[0], ne[1], ggml_type_name(ggml_type(ttype)), ggml_nbytes(tensor)/1024.0/1024.0, ggml_nbytes(tensor));
            }

            const size_t bpe = ggml_type_size(ggml_type(ttype));

            if ((nelements*bpe)/ggml_blck_size(tensor->type) != ggml_nbytes(tensor)) {
                fprintf(stderr, "%s: tensor '%s' has wrong size in model file: got %zu, expected %zu\n",
                        __func__, name.c_str(), ggml_nbytes(tensor), nelements*bpe);
                return false;
            }

            ggml_allocr_alloc(alloc, tensor);

            if (ggml_backend_is_cpu  (model.backend)
```

这段代码是一个 C 语言函数，名为“input_tensor”。它用于读取一个模型（Metal）中的张量（ two-dimensional data）并将其存储在内存中。以下是这段代码的作用：

1. 检查输入张量使用的硬件平台。如果是金属（Metal）平台，则可以直接从内存中读取；如果不是，则需要先将张量存储在临时缓冲区中，然后将其复制到设备内存中。
2. 对于 GPT-2 模型，如果发现没有 LM 头，则需要将 WTE 张量复制到 LM 头中。
3. 记录模型大小。

这段代码的具体实现如下：


```cpp
#ifdef GGML_USE_METAL
                || ggml_backend_is_metal(model.backend)
#endif
                ) {
                // for the CPU and Metal backend, we can read directly into the tensor
                fin.read(reinterpret_cast<char *>(tensor->data), ggml_nbytes(tensor));
            } else {
                // read into a temporary buffer first, then copy to device memory
                read_buf.resize(ggml_nbytes(tensor));
                fin.read(read_buf.data(), ggml_nbytes(tensor));
                ggml_backend_tensor_set(tensor, read_buf.data(), 0, ggml_nbytes(tensor));
            }

            // GPT-2 models share the WTE tensor as the LM head
            if (name == "model/wte" && has_lm_head == false) {
                //ggml_allocr_alloc(alloc, model.lm_head);
                //ggml_backend_tensor_copy(tensor, model.lm_head);
                model.lm_head = tensor;
            }

            if (name == "model/lm_head") {
                has_lm_head = true;
            }

            total_size += ggml_nbytes(tensor);
        }

        ggml_allocr_free(alloc);
        printf("%s: model size  = %8.2f MB\n", __func__, total_size/1024.0/1024.0);
    }

    fin.close();

    return true;
}

```

This is a function definition for the backward pass of a multi-layer perceptron (MLP) model. The input is an input tensor `inpL`, which has three dimensions: `(batch_size, input_size, num_classes)`. The function uses backpropagation through the process of culling, computing gradients, and applying the appropriate mask for the negative gradient.

The function first preloads the input tensor with the mask, then converts it to a floating-point number using the `nn.to_float` function. Then, it performs the following operations in the forward pass:

1.  Convolutional layers: The function applies a 3D convolutional neural network (CNN) to the input tensor, which consists of a 1D convolutional layer with 768 hidden units and a 1D pooling layer with a window size of 16. The function uses the `Keras.layers.Conv2D` class to define the convolutional layer and the `model.layers.Pooling1D` class to define the pooling layer.
2.  Max pooling layer: The function applies the `Keras.layers.MaxPooling1D` class to the output of the CNN to get a 1D max pooling operation.
3.  Dense layers: The function applies a 1D dense (linear) layer with `model.layers.Dense` class to create two output tensors, one for the input and the second for the output of the CNN.

The function then defines the backward pass using the `backpropagation` algorithm. It first computes the output of the forward pass as a float tensor. Then, it applies the backpropagation algorithm to compute the gradients of the model parameters with respect to the input tensor. Finally, it applies the masked operation for the negative gradient using the `Keras.layers.BatchNormalization` class.

The function returns the `Module` object for the MLP model.


```cpp
// build the computation graph
struct ggml_cgraph * gpt2_graph(
        const  gpt2_model  & model,
        struct ggml_allocr * allocr,
        const  gpt2_batch  & batch) {
    const auto & hparams = model.hparams;

    const int n_embd  = hparams.n_embd;
    const int n_layer = hparams.n_layer;
    const int n_ctx   = hparams.n_ctx;
    const int n_head  = hparams.n_head;

    const auto & kv_cache = model.kv_cache;

    const int32_t n_tokens = batch.n_tokens;
    const int32_t n_kv     = ggml_allocr_is_measure(allocr) ? n_ctx            : kv_cache.n;
    const int32_t kv_head  = ggml_allocr_is_measure(allocr) ? n_ctx - n_tokens : kv_cache.head;

    // since we are using ggml-alloc, this buffer only needs enough space to hold the ggml_tensor and ggml_cgraph structs, but not the tensor data
    static size_t buf_size = ggml_tensor_overhead()*GGML_DEFAULT_GRAPH_SIZE + ggml_graph_overhead();
    static std::vector<uint8_t> buf(buf_size);

    struct ggml_init_params params = {
        /*.mem_size   =*/ buf_size,
        /*.mem_buffer =*/ buf.data(),
        /*.no_alloc   =*/ true, // the tensors will be allocated later by ggml_allocr_alloc_graph()
    };

    struct ggml_context * ctx0 = ggml_init(params);

    struct ggml_cgraph  * gf = ggml_new_graph(ctx0);

    struct ggml_tensor * inpL;
    if (batch.token) {
        struct ggml_tensor * inp_tokens = ggml_new_tensor_1d(ctx0, GGML_TYPE_I32, n_tokens);
        ggml_allocr_alloc(allocr, inp_tokens);
        if (!ggml_allocr_is_measure(allocr)) {
            ggml_backend_tensor_set(inp_tokens, batch.token, 0, n_tokens*ggml_element_size(inp_tokens));
        }

        struct ggml_tensor * position = ggml_new_tensor_1d(ctx0, GGML_TYPE_I32, n_tokens);
        ggml_allocr_alloc(allocr, position);
        if (!ggml_allocr_is_measure(allocr)) {
            for (int i = 0; i < n_tokens; ++i) {
                int32_t v = batch.pos[i];
                ggml_backend_tensor_set(position, &v, i*sizeof(int32_t), sizeof(v));
            }
        }

        // wte + wpe
        inpL =
            ggml_add(ctx0,
                    ggml_get_rows(ctx0, model.wte, inp_tokens),
                    ggml_get_rows(ctx0, model.wpe, position));
    } else {
        GGML_ASSERT(batch.embd);

        inpL = ggml_new_tensor_2d(ctx0, GGML_TYPE_F32, n_embd, n_tokens);

        ggml_allocr_alloc(allocr, inpL);
        if (!ggml_allocr_is_measure(allocr)) {
            ggml_backend_tensor_set(inpL, batch.embd, 0, n_tokens * n_embd * ggml_element_size(inpL));
        }
    }

    struct ggml_tensor * KQ_scale = ggml_new_tensor_1d(ctx0, GGML_TYPE_F32, 1);
    ggml_allocr_alloc(allocr, KQ_scale);
    if (!ggml_allocr_is_measure(allocr)) {
        float s = 1.0f/sqrtf(float(n_embd)/n_head);
        ggml_backend_tensor_set(KQ_scale, &s, 0, sizeof(s));
    }

    // KQ_mask (mask for 1 head, it will be broadcasted to all heads)
    struct ggml_tensor * KQ_mask = ggml_new_tensor_3d(ctx0, GGML_TYPE_F32, n_kv, n_tokens, 1);
    ggml_set_name(KQ_mask, "KQ_mask");
    ggml_allocr_alloc(allocr, KQ_mask);
    if (!ggml_allocr_is_measure(allocr)) {
        std::vector<float> data_buf(n_kv*n_tokens);
        const float neg_inf_v = -INFINITY;

        for (int h = 0; h < 1; ++h) {
            int h_offset = h*(n_kv*n_tokens);
            for (int j = 0; j < n_tokens; ++j) {
                const gpt2_pos    pos    = batch.pos[j];
                const gpt2_seq_id seq_id = batch.seq_id[j];

                for (int i = 0; i < n_kv; ++i) {
                    if (!kv_cache.cells[i].has_seq_id(seq_id) || kv_cache.cells[i].pos > pos) {
                        data_buf[h_offset + j*n_kv + i] = neg_inf_v;
                    }
                }
            }
        }

        ggml_backend_tensor_set(KQ_mask, data_buf.data(), 0, data_buf.size() * sizeof(float));
    }

    for (int il = 0; il < n_layer; ++il) {
        struct ggml_tensor * cur;

        // norm
        {
            // [ 768, N]
            cur = ggml_norm(ctx0, inpL, hparams.eps);

            // cur = ln_1_g*cur + ln_1_b
            // [ 768, N]
            cur = ggml_add(ctx0,
                    ggml_mul(ctx0,
                        cur,
                        model.layers[il].ln_1_g),
                    model.layers[il].ln_1_b);
        }

        // attn
        // [2304,        768] - model.layers[il].c_attn_attn_w
        // [2304,          1] - model.layers[il].c_attn_attn_b
        // [ 768,   n_tokens] - cur (in)
        // [2304,   n_tokens] - cur (out)
        //
        // cur = attn_w*cur + attn_b
        // [2304, n_tokens]
        {
            cur = ggml_mul_mat(ctx0,
                    model.layers[il].c_attn_attn_w,
                    cur);

            cur = ggml_add(ctx0,
                    cur,
                    model.layers[il].c_attn_attn_b);
        }

        // self-attention
        {
            struct ggml_tensor * Qcur = ggml_view_2d(ctx0, cur, n_embd, n_tokens, cur->nb[1], 0*sizeof(float)*n_embd);
            struct ggml_tensor * Kcur = ggml_view_2d(ctx0, cur, n_embd, n_tokens, cur->nb[1], 1*sizeof(float)*n_embd);
            struct ggml_tensor * Vcur = ggml_view_2d(ctx0, cur, n_embd, n_tokens, cur->nb[1], 2*sizeof(float)*n_embd);

            // store key and value to memory
            if (n_tokens >= 1) {
                struct ggml_tensor * k = ggml_view_1d(ctx0, model.kv_cache.k, n_tokens*n_embd, (ggml_element_size(model.kv_cache.k)*n_embd)*(il*n_ctx + kv_head));
                struct ggml_tensor * v = ggml_view_1d(ctx0, model.kv_cache.v, n_tokens*n_embd, (ggml_element_size(model.kv_cache.v)*n_embd)*(il*n_ctx + kv_head));

                ggml_build_forward_expand(gf, ggml_cpy(ctx0, Kcur, k));
                ggml_build_forward_expand(gf, ggml_cpy(ctx0, Vcur, v));
            }

            // Q = Qcur.contiguous().view(n_embd/n_head, n_head, N).permute(0, 2, 1, 3)
            // [64, N, 12]
            struct ggml_tensor * Q =
                ggml_permute(ctx0,
                        ggml_cpy(ctx0,
                            Qcur,
                            ggml_new_tensor_3d(ctx0, GGML_TYPE_F32, n_embd/n_head, n_head, n_tokens)),
                        0, 2, 1, 3);

            // K = Kmem.view(n_embd/n_head, n_head, n_kv).permute(0, 2, 1, 3)
            // [64, n_kv, 12]
            struct ggml_tensor * K =
                ggml_permute(ctx0,
                        ggml_reshape_3d(ctx0,
                            ggml_view_1d(ctx0, model.kv_cache.k, n_kv*n_embd, il*n_ctx*ggml_element_size(model.kv_cache.k)*n_embd),
                            n_embd/n_head, n_head, n_kv),
                        0, 2, 1, 3);

            // GG: flash attention
            //struct ggml_tensor * V =
            //    ggml_cpy(ctx0,
            //            ggml_permute(ctx0,
            //                ggml_reshape_3d(ctx0,
            //                    ggml_view_1d(ctx0, model.kv_cache.v, n_kv*n_embd, il*n_ctx*ggml_element_size(model.kv_cache.v)*n_embd),
            //                    n_embd/n_head, n_head, n_kv),
            //                1, 2, 0, 3),
            //            ggml_new_tensor_3d(ctx0, GGML_TYPE_F32, n_kv, n_embd/n_head, n_head));

            //struct ggml_tensor * KQV = ggml_flash_attn(ctx0, Q, K, V, true);

            // K * Q
            // [n_kv, n_tokens, 12]
            struct ggml_tensor * KQ = ggml_mul_mat(ctx0, K, Q);

            // KQ_scaled = KQ / sqrt(n_embd/n_head)
            // [n_kv, n_tokens, 12]
            struct ggml_tensor * KQ_scaled =
                ggml_scale(ctx0,
                        KQ,
                        KQ_scale);

            // KQ_masked = mask_past(KQ_scaled)
            // [n_kv, n_tokens, 12]
            struct ggml_tensor * KQ_masked = ggml_add(ctx0, KQ_scaled, KQ_mask);

            // KQ = soft_max(KQ_masked)
            // [n_kv, N, 12]
            struct ggml_tensor * KQ_soft_max = ggml_soft_max(ctx0, KQ_masked);

            // V_trans = Vmem.view(n_embd/n_head, n_head, n_kv).permute(1, 2, 0, 3).contiguous()
            // [n_kv, 64, 12]
            struct ggml_tensor * V_trans =
                ggml_cpy(ctx0,
                        ggml_permute(ctx0,
                            ggml_reshape_3d(ctx0,
                                ggml_view_1d(ctx0, model.kv_cache.v, n_kv*n_embd, il*n_ctx*ggml_element_size(model.kv_cache.v)*n_embd),
                                n_embd/n_head, n_head, n_kv),
                            1, 2, 0, 3),
                        ggml_new_tensor_3d(ctx0, model.kv_cache.v->type, n_kv, n_embd/n_head, n_head));

            // KQV = transpose(V) * KQ_soft_max
            // [64, n_tokens, 12]
            struct ggml_tensor * KQV = ggml_mul_mat(ctx0, V_trans, KQ_soft_max);

            // KQV_merged = KQV.permute(0, 2, 1, 3)
            // [64, 12, n_tokens]
            struct ggml_tensor * KQV_merged = ggml_permute(ctx0, KQV, 0, 2, 1, 3);

            // cur = KQV_merged.contiguous().view(n_embd, N)
            // [768, n_tokens]
            cur = ggml_cpy(ctx0,
                    KQV_merged,
                    ggml_new_tensor_2d(ctx0, GGML_TYPE_F32, n_embd, n_tokens));
        }

        // projection
        // [ 768, 768] - model.layers[il].c_attn_proj_w
        // [ 768,   1] - model.layers[il].c_attn_proj_b
        // [ 768,   N] - cur (in)
        // [ 768,   N] - cur (out)
        //
        // cur = proj_w*cur + proj_b
        // [768, N]
        {
            cur = ggml_mul_mat(ctx0,
                    model.layers[il].c_attn_proj_w,
                    cur);

            cur = ggml_add(ctx0,
                    cur,
                    model.layers[il].c_attn_proj_b);
        }

        // add the input
        cur = ggml_add(ctx0, cur, inpL);

        struct ggml_tensor * inpFF = cur;

        // feed-forward network
        {
            // norm
            {
                cur = ggml_norm(ctx0, inpFF, hparams.eps);

                // cur = ln_2_g*cur + ln_2_b
                // [ 768, N]
                cur = ggml_add(ctx0,
                        ggml_mul(ctx0,
                            cur,
                            model.layers[il].ln_2_g),
                        model.layers[il].ln_2_b);
            }

            // fully connected
            // [3072, 768] - model.layers[il].c_mlp_fc_w
            // [3072,   1] - model.layers[il].c_mlp_fc_b
            // [ 768,   N] - cur (in)
            // [3072,   N] - cur (out)
            //
            // cur = fc_w*cur + fc_b
            // [3072, N]
            cur = ggml_mul_mat(ctx0,
                    model.layers[il].c_mlp_fc_w,
                    cur);

            cur = ggml_add(ctx0,
                    cur,
                    model.layers[il].c_mlp_fc_b);

            // GELU activation
            // [3072, N]
            cur = ggml_gelu(ctx0, cur);

            // projection
            // [ 768, 3072] - model.layers[il].c_mlp_proj_w
            // [ 768,    1] - model.layers[il].c_mlp_proj_b
            // [3072,    N] - cur (in)
            // [ 768,    N] - cur (out)
            //
            // cur = proj_w*cur + proj_b
            // [768, N]
            cur = ggml_mul_mat(ctx0,
                    model.layers[il].c_mlp_proj_w,
                    cur);

            cur = ggml_add(ctx0,
                    cur,
                    model.layers[il].c_mlp_proj_b);
        }

        // input for next layer
        inpL = ggml_add(ctx0, cur, inpFF);
    }

    // norm
    {
        // [ 768, N]
        inpL = ggml_norm(ctx0, inpL, hparams.eps);

        // inpL = ln_f_g*inpL + ln_f_b
        // [ 768, N]
        inpL = ggml_add(ctx0,
                ggml_mul(ctx0,
                    inpL,
                    model.ln_f_g),
                model.ln_f_b);
    }

    // inpL = WTE * inpL
    // [ 768, 50257] - model.lm_head
    // [ 768, N]     - inpL
    inpL = ggml_mul_mat(ctx0, model.lm_head, inpL);

    // logits -> probs
    //inpL = ggml_soft_max(ctx0, inpL);

    ggml_build_forward_expand(gf, inpL);

    ggml_free(ctx0);

    return gf;
}

```

这段代码定义了一个名为 `gpt2_kv_cache_seq_cp` 的函数，属于 `gpt2_kv_cache` 结构的成员函数。

函数接收两个参数：`cache` 和 `seq_id_src`，分别代表上下文缓存和输入序列的起始位置。同时，函数还接收两个参数：`seq_id_dst` 和 `p0`，分别代表目标序列的起始位置和输入序列的起始位置。

函数的主要作用是检查输入序列是否在上下文缓存中，如果存在且起始位置在正确的范围内，将目标序列的起始位置从输入序列中获取并插入到上下文缓存中。

具体实现可以分为以下几个步骤：

1. 检查输入序列的起始位置 `seq_id_src` 是否在上下文缓存中，如果不是，将起始位置设为 0。
2. 检查目标序列的起始位置 `seq_id_dst` 是否在上下文缓存中，如果不是，设置为 `std::numeric_limits<gpt2_pos>::max()`，确保不会超出上下文缓存的范围。
3. 遍历上下文缓存中的每个单元格，检查当前单元格的起始位置 `seq_id_src` 是否在输入序列中，如果是，将起始位置 `seq_id_dst` 更新为当前单元格的起始位置。
4. 最后，将目标序列的起始位置更新为输入序列中的起始位置，即 `seq_id_src`。


```cpp
static void gpt2_kv_cache_seq_cp(
        struct gpt2_kv_cache & cache,
                 gpt2_seq_id   seq_id_src,
                 gpt2_seq_id   seq_id_dst,
                    gpt2_pos   p0,
                    gpt2_pos   p1) {
    if (p0 < 0) p0 = 0;
    if (p1 < 0) p1 = std::numeric_limits<gpt2_pos>::max();

    for (uint32_t i = 0; i < cache.size; ++i) {
        if (cache.cells[i].has_seq_id(seq_id_src) && cache.cells[i].pos >= p0 && cache.cells[i].pos < p1) {
            cache.cells[i].seq_id.insert(seq_id_dst);
        }
    }
}

```

这段代码定义了一个名为 `gpt2_batch` 的结构体，用于表示一个 GPT2 模型的批次。

在结构体定义之前，定义了一个名为 `gpt2_batch_init` 的函数，它的输入参数包括 `n_tokens` 和 `embd` 两个整型变量，用于初始化 GPT2 模型中的参数。

函数首先检查 `embd` 是否为真，如果是，就创建一个大小为 `n_tokens` 行 `embd` 大小的二维数组，并将其存储在 `batch.embd` 中；如果不是，就创建一个大小为 `n_tokens` 的指针数组，并将其存储在 `batch.token` 中。

接着，函数创建一个大小为 `n_tokens` 的位置指针数组，并将其存储在 `batch.pos` 中；还创建一个大小为 `n_tokens` 的序列ID指针数组，并将其存储在 `batch.seq_id` 中。

最后，函数创建一个大小为 `n_tokens` 的逻辑值数组，并将其存储在 `batch.logits` 中。

函数返回一个名为 `batch` 的结构体，用于表示初始化后的 GPT2 模型批次。


```cpp
struct gpt2_batch gpt2_batch_init(int32_t n_tokens, int32_t embd) {
    gpt2_batch batch;

    if (embd) {
        batch.embd = (float *) malloc(sizeof(float) * n_tokens * embd);
    } else {
        batch.token = (gpt_vocab::id *) malloc(sizeof(gpt_vocab::id) * n_tokens);
    }

    batch.pos    = (gpt2_pos *)    malloc(sizeof(gpt2_pos)    * n_tokens);
    batch.seq_id = (gpt2_seq_id *) malloc(sizeof(gpt2_seq_id) * n_tokens);
    batch.logits = (int8_t *)      malloc(sizeof(int8_t)      * n_tokens);

    return batch;
}

```

This function appears to be used by the TensorFlow library to perform inference on a pre-trained language model using the GPT-2 model. It takes in a batch of logits as input and returns an integer indicating whether the inference was successful, error or warning.

If the input logits are not provided, the function returns an error. If the input logits are provided, the function first verifies that the batch.seq_id is not provided and that the batch.n_tokens is greater than zero. The function then proceeds to allocate the necessary memory and run the inference using the provided logits.

If the inference is successful, the function returns 0. If the inference is not successful, the function returns a non-zero error code. If the input logits are not valid or the input logits are not provided, the function returns a non-zero error code.


```cpp
void gpt2_batch_free(struct gpt2_batch batch) {
    if (batch.token)  free(batch.token);
    if (batch.embd)   free(batch.embd);
    if (batch.pos)    free(batch.pos);
    if (batch.seq_id) free(batch.seq_id);
    if (batch.logits) free(batch.logits);
}

// Positive return values does not mean a fatal error, but rather a warning.
//   0 - success
// < 0 - error
int gpt2_decode(
        struct gpt2_model  & model,
        struct ggml_allocr * allocr,
        struct gpt2_batch    batch,
        int                  n_threads,
        std::vector<float> & logits) {
    const int32_t n_tokens = batch.n_tokens;
    const auto &  hparams  = model.hparams;
    const int     n_vocab  = hparams.n_vocab;

    if (n_tokens == 0) {
        printf("%s: n_tokens == 0", __func__);
        return -1;
    }

    GGML_ASSERT((!batch.token && batch.embd) || (batch.token && !batch.embd));

    auto & cache = model.kv_cache;

    for (int i = 0; i < n_tokens; i++) {
        cache.cells[cache.head + i].pos = batch.pos[i];
        cache.cells[cache.head + i].seq_id.insert(batch.seq_id[i]);
    }

    cache.n = cache.head + n_tokens;

    // reset the allocator to free all the memory allocated during the previous inference
    ggml_allocr_reset(allocr);

    struct ggml_cgraph * gf = gpt2_graph(model, allocr, batch);

    // allocate tensors
    ggml_allocr_alloc_graph(allocr, gf);

    // run the computation
    if (ggml_backend_is_cpu(model.backend)) {
        ggml_backend_cpu_set_n_threads(model.backend, n_threads);
    }
```

这段代码是一个用C++编写的演示如何使用ggml库中的MetalAPI来进行模型的推理。

首先，它检查ggml库的后台是否为Metal，如果是，则设置n_threads参数。然后，它执行ggml库的推理，并将结果保存到logits数组中。接着，如果输入数据中存在批处理信息，它将返回批处理结果。最后，如果输入数据中包含推理结果，它将被存储到logits数组中，并更新KV ring缓冲区。

下面是示例代码：

```cppc
#include <iostream>
#include <cstring>
#include <fstream>
#include <ggml/ggml.h>
#include <ggml/api.h>
#include <ggml/util/api.h>
#include <ggml/example/example.h>
#include <ggml/methods/method_handle.h>
#include <ggml/methods/method_arg.h>
#include <ggml/methods/output_tensor.h>
#include <ggml/execute/execute.h>
#include <ggml/executor/executor.h>
#include <ggml/utils/events.h>
#include <ggml/utils/parser.h>
#include <ggml/utils/url_resolver.h>
```


```cpp
#ifdef GGML_USE_METAL
    if (ggml_backend_is_metal(model.backend)) {
        ggml_backend_metal_set_n_cb(model.backend, n_threads);
    }
#endif
    ggml_backend_graph_compute(model.backend, gf);

    //if (n_past%100 == 0) {
    //    ggml_graph_print   (&gf);
    //    ggml_graph_dump_dot(&gf, NULL, "gpt-2.dot");
    //}

    // in this case, the output tensor is the last one in the graph
    struct ggml_tensor * inpL = gf->nodes[gf->n_nodes - 1];

    if (batch.logits) {
        // return logits for all tokens
        logits.resize(n_vocab*n_tokens);
        for (int32_t i = 0; i < n_tokens; i++) {
            if (batch.logits[i] == 0) {
                continue;
            }
            ggml_backend_tensor_get(inpL, logits.data() + n_vocab*i, n_vocab*i*sizeof(float), sizeof(float)*n_vocab);
        }
    } else {
        // return result just for the last token
        logits.resize(n_vocab);
        ggml_backend_tensor_get(inpL, logits.data(), (n_vocab*(n_tokens-1))*sizeof(float), sizeof(float)*n_vocab);
    }

    // update the kv ring buffer
    cache.head += n_tokens;

    // ensure kv cache head points to a valid index.
    if (cache.head >= cache.size) {
        printf("%s: cache.head >= cache.size\n", __func__);
        return -2;
    }

    return 0;
}

```

This is a function definition for `__create__()` of the GPT model. It initializes the model with a specific configuration and timing settings, and then performs a prediction using the configured samples.

Here is a brief description of the arguments and their usage:

- `batch`: The batch of samples to predict. This argument is not used in this function.
- `model.ctx`: The context for the model. This argument is not used in this function.
- `model.buffer_w`: The write buffer for the model. This argument is not used in this function.
- `model.kv_cache.buffer`: The key-value cache buffer for the model. This argument is not used in this function.
- `buf_compute`: The buffer for the compute function. This argument is not used in this function.
- `model.backend`: The backend for the model. This argument is not used in this function.

The function starts by creating the model with the specified configuration and then initializes the buffer for the compute function. The function then loops over the samples in the batch and performs a prediction for each sample using the `compute` function.

After the prediction, the function checks if the computation took longer than the specified timeout. If it took longer, it prints an error message and returns 1. If the computation took less time than the timeout, the function prints a message indicating that the computation was successful.

Finally, the function reports the timing information, including the number of decoded and predicted elements per sample, the load time and sample time for each sample, and the total time for the computation.


```cpp
int main(int argc, char ** argv) {
    ggml_time_init();

    const int64_t t_main_start_us = ggml_time_us();

    gpt_params params;

    if (gpt_params_parse(argc, argv, params) == false) {
        return 1;
    }

    if (params.seed < 0) {
        params.seed = time(NULL);
    }

    printf("%s: seed = %d\n", __func__, params.seed);

    std::mt19937 rng(params.seed);
    if (params.prompt.empty()) {
        params.prompt = gpt_random_prompt(rng);
    }

    int64_t t_load_us = 0;

    gpt_vocab vocab;
    gpt2_model model;

    // load the model
    {
        const int64_t t_start_us = ggml_time_us();

        if (!gpt2_model_load(params.model, model, vocab, params.n_ctx, params.n_gpu_layers)) {
            fprintf(stderr, "%s: failed to load model from '%s'\n", __func__, params.model.c_str());
            return 1;
        }

        t_load_us = ggml_time_us() - t_start_us;

        test_gpt_tokenizer(vocab, params.token_test);
    }

    // tokenize the prompt
    std::vector<gpt_vocab::id> embd_inp = ::gpt_tokenize(vocab, params.prompt);

    // keep this buffer alive while evaluating the model
    ggml_backend_buffer_t buf_compute;

    const int n_parallel = params.n_parallel;
    const int n_batch_max = std::max(embd_inp.size(), (size_t)n_parallel);

    // create a gpt2_batch
    // we use this object to submit token data for decoding
    gpt2_batch batch = gpt2_batch_init(n_batch_max, 0);

    // prepare required memory and allocate the compute buffer
    struct ggml_allocr * allocr = NULL;
    {
        // alignment required by the backend
        size_t align = ggml_backend_get_alignment(model.backend);
        allocr = ggml_allocr_new_measure(align);

        batch.n_tokens = n_batch_max;

        // create the worst case graph for memory usage estimation
        struct ggml_cgraph * gf = gpt2_graph(model, allocr, batch);

        // compute the required memory
        size_t mem_size = ggml_allocr_alloc_graph(allocr, gf);

        // recreate the allocator with the required memory
        ggml_allocr_free(allocr);
        buf_compute = ggml_backend_alloc_buffer(model.backend, mem_size);
        allocr = ggml_allocr_new_from_buffer(buf_compute);

        fprintf(stderr, "%s: compute buffer size: %.2f MB\n", __func__, mem_size/1024.0/1024.0);
    }

    int64_t t_sample_us  = 0;
    int64_t t_predict_us = 0;

    std::vector<float> logits;

    // evaluate the initial prompt
    batch.n_tokens = embd_inp.size();

    for (int32_t i = 0; i < batch.n_tokens; i++) {
        batch.token[i]  = embd_inp[i];
        batch.pos[i]    = i;
        batch.seq_id[i] = 0;
        batch.logits[i] = false;
    }

    // gpt2_decode will output logits only for the last token of the prompt
    batch.logits[batch.n_tokens - 1] = true;

    if (gpt2_decode(model, allocr, batch, params.n_threads, logits) != 0) {
        printf("%s: gpt2_decode() failed\n", __func__);
        return 1;
    }

    // assign the system KV cache to all parallel sequences
    // this way, the parallel sequences will "reuse" the prompt tokens without having to copy them
    for (int32_t i = 1; i < n_parallel; ++i) {
        gpt2_kv_cache_seq_cp(model.kv_cache, 0, i, 0, batch.n_tokens);
    }

    if (n_parallel > 1) {
        printf("\n\n%s: generating %d sequences ...\n", __func__, n_parallel);
    }

    params.n_predict = std::min(params.n_predict, model.hparams.n_ctx - (int) embd_inp.size());

    printf("%s: prompt: '%s'\n", __func__, params.prompt.c_str());
    printf("%s: number of tokens in prompt = %zu, first 8 tokens: ", __func__, embd_inp.size());
    for (int i = 0; i < std::min(8, (int) embd_inp.size()); i++) {
        printf("%d ", embd_inp[i]);
    }
    printf("\n\n");

    std::vector<gpt_vocab::token> streams(n_parallel);

    // remember the batch index of the last token for each parallel sequence
    // we need this to determine which logits to sample from
    std::vector<int32_t> i_batch(n_parallel, batch.n_tokens - 1);

    int n_cur     = batch.n_tokens;
    int n_len     = batch.n_tokens + params.n_predict;
    int n_decoded = 0;

    const int   n_vocab = model.hparams.n_vocab;
    const int   top_k = params.top_k;
    const float top_p = params.top_p;
    const float temp  = params.temp;

    while (n_cur < n_len) {
        batch.n_tokens = 0;

        for (int32_t i = 0; i < n_parallel; ++i) {
            if (i_batch[i] < 0) {
                // the stream has already finished
                continue;
            }

            auto * logits_i = logits.data() + i_batch[i]*n_vocab;

            gpt_vocab::id id = 0;
            {
                const int64_t t_start_sample_us = ggml_time_us();

                id = gpt_sample_top_k_top_p(vocab, logits_i, top_k, top_p, temp, rng);

                t_sample_us += ggml_time_us() - t_start_sample_us;
            }

            // is it an end of stream? -> mark the stream as finished
            if ((!params.ignore_eos && id == 50256) || n_cur == n_len - 1) {
                i_batch[i] = -1;
                printf("\n");
                if (n_parallel > 1) {
                    printf("%s: stream %d finished at n_cur = %d", __func__, i, n_cur);
                }

                continue;
            }

            auto& token = vocab.id_to_token[id];
            if (n_parallel == 1) {
                printf("%s", token.c_str());
                fflush(stdout);
            }

            streams[i] += token;

            // push this new token for next evaluation
            batch.token [batch.n_tokens] = id;
            batch.pos   [batch.n_tokens] = n_cur;
            batch.seq_id[batch.n_tokens] = i;
            batch.logits[batch.n_tokens] = true;

            i_batch[i] = batch.n_tokens;

            batch.n_tokens += 1;

            n_decoded += 1;
        }

        // all streams are finished
        if (batch.n_tokens == 0) {
            break;
        }

        n_cur += 1;

        {
            const int64_t t_start_us = ggml_time_us();

            // evaluate the current batch with the transformer model
            int ret_code = gpt2_decode(model, allocr, batch, params.n_threads, logits);
            if (ret_code != 0) {
                fprintf(stderr, "%s : failed to eval, return code %d\n", __func__, ret_code);
                return 1;
            }

            t_predict_us += ggml_time_us() - t_start_us;
        }
    }

    if (n_parallel > 1) {
        printf("\n");

        for (int32_t i = 0; i < n_parallel; ++i) {
            printf("sequence %d:\n\n%s%s\n\n", i, params.prompt.c_str(), streams[i].c_str());
        }
    }

    // report timing
    {
        const int64_t t_main_end_us = ggml_time_us();

        printf("\n\n");
        printf("%s:     n_decoded = %8d\n",      __func__, n_decoded);
        printf("%s:     load time = %8.2f ms\n", __func__, t_load_us/1000.0f);
        printf("%s:   sample time = %8.2f ms\n", __func__, t_sample_us/1000.0f);
        printf("%s:  predict time = %8.2f ms\n", __func__, t_predict_us/1000.0f);
        printf("%s:    total time = %8.2f ms\n", __func__, (t_main_end_us - t_main_start_us)/1000.0f);
    }

    gpt2_batch_free(batch);
    ggml_free(model.ctx);

    ggml_backend_buffer_free(model.buffer_w);
    ggml_backend_buffer_free(model.kv_cache.buffer);
    ggml_backend_buffer_free(buf_compute);
    ggml_backend_free(model.backend);

    return 0;
}

```