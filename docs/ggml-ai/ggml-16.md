# GGML源码解析 16

# `examples/starcoder/quantize.cpp`

这段代码是一个C++程序，主要作用是读取一个定义好的结构体数组(ggml::Vector)，并将其中的元素打印出来。结构体数组的名字为"common.h"，这里我们假设这个结构体有多个成员变量。

程序首先引入了ggml::ggml.h和common.h两个头文件，ggml::ggml.h可能是一个类似于ggml库的头文件，common.h则是这个程序自己的头文件，很可能是定义了main函数。

接着，程序通过common.h头文件中的函数gcm_load_path(std::string& file_path)读取了一个定义好的ggml::Vector对象，并将其存储到了std::vector<ggml::Vector>容器中。然后，程序通过容器中的函数print_vectors(const std::vector<ggml::Vector>& vectors)打印出了这个ggml::Vector容器中的所有元素。

不过，程序在读取文件时需要提供一个文件路径，这里我们使用了ggml::ggml.h中的函数ggml_open_file(const std::string& file_path)来读取文件。regex是C++11中的一个标准库，其中包含了一个正则表达式，可以用于字符串的匹配。这里我们使用regex来判断一个字符串是否为"static_exports"。


```cpp
#include "ggml/ggml.h"

#include "common.h"
#include "common-ggml.h"

#include <cassert>
#include <cmath>
#include <cstdio>
#include <cstring>
#include <fstream>
#include <map>
#include <string>
#include <vector>
#include <regex>

```

This is a function that reads a model file andits attributes, such as the number of vocabulary elements and the size of the model. It reads the model file and its attributes from the input file, such as the name of the file and its size, and then reads the model file.

This function reads the text data of the model file and stores it in the input buffer. It then checks if the file size is equal to the size of the model file specified by the user. If the file size is not equal to the specified size, the function will print an error message and return false.

The function also reads the id of the word and its corresponding index in the vocabulary. After that, it prints the remaining words in the file if the user has not specified any specific options for quantization.

Finally, the function uses a common quantization method to quantize the model file and returns true if the quantization was successful. If the quantization fails, the function will print an error message and return false.


```cpp
// default hparams (GPT-2 117M)
struct starcoder_hparams {
    int32_t n_vocab = 49280;
    int32_t n_ctx   = 2048;
    int32_t n_embd  = 2048;
    int32_t n_head  = 16;
    int32_t n_layer = 24;
    int32_t ftype   = 1;
};

// quantize a model
bool starcoder_model_quantize(const std::string & fname_inp, const std::string & fname_out, ggml_ftype ftype) {
    gpt_vocab vocab;

    printf("%s: loading model from '%s'\n", __func__, fname_inp.c_str());

    auto finp = std::ifstream(fname_inp, std::ios::binary);
    if (!finp) {
        fprintf(stderr, "%s: failed to open '%s' for reading\n", __func__, fname_inp.c_str());
        return false;
    }

    auto fout = std::ofstream(fname_out, std::ios::binary);
    if (!fout) {
        fprintf(stderr, "%s: failed to open '%s' for writing\n", __func__, fname_out.c_str());
        return false;
    }

    // verify magic
    {
        uint32_t magic;
        finp.read((char *) &magic, sizeof(magic));
        if (magic != GGML_FILE_MAGIC) {
            fprintf(stderr, "%s: invalid model file '%s' (bad magic)\n", __func__, fname_inp.c_str());
            return false;
        }

        fout.write((char *) &magic, sizeof(magic));
    }

    starcoder_hparams hparams;

    // load hparams
    {
        finp.read((char *) &hparams.n_vocab, sizeof(hparams.n_vocab));
        finp.read((char *) &hparams.n_ctx,   sizeof(hparams.n_ctx));
        finp.read((char *) &hparams.n_embd,  sizeof(hparams.n_embd));
        finp.read((char *) &hparams.n_head,  sizeof(hparams.n_head));
        finp.read((char *) &hparams.n_layer, sizeof(hparams.n_layer));
        finp.read((char *) &hparams.ftype,   sizeof(hparams.ftype));

        const int32_t qntvr_src =    hparams.ftype / GGML_QNT_VERSION_FACTOR;
        const int32_t ftype_dst = GGML_QNT_VERSION * GGML_QNT_VERSION_FACTOR + ftype;

        printf("%s: n_vocab     = %d\n", __func__, hparams.n_vocab);
        printf("%s: n_ctx       = %d\n", __func__, hparams.n_ctx);
        printf("%s: n_embd      = %d\n", __func__, hparams.n_embd);
        printf("%s: n_head      = %d\n", __func__, hparams.n_head);
        printf("%s: n_layer     = %d\n", __func__, hparams.n_layer);
        printf("%s: ftype (src) = %d\n", __func__, hparams.ftype);
        printf("%s: qntvr (src) = %d\n", __func__, qntvr_src);
        printf("%s: ftype (dst) = %d\n", __func__, ftype_dst);
        printf("%s: qntvr (dst) = %d\n", __func__, GGML_QNT_VERSION);

        fout.write((char *) &hparams.n_vocab, sizeof(hparams.n_vocab));
        fout.write((char *) &hparams.n_ctx,   sizeof(hparams.n_ctx));
        fout.write((char *) &hparams.n_embd,  sizeof(hparams.n_embd));
        fout.write((char *) &hparams.n_head,  sizeof(hparams.n_head));
        fout.write((char *) &hparams.n_layer, sizeof(hparams.n_layer));
        fout.write((char *) &ftype_dst,       sizeof(ftype_dst));
    }

    // load vocab
    {
        int32_t n_vocab = 0;
        finp.read ((char *) &n_vocab, sizeof(n_vocab));
        fout.write((char *) &n_vocab, sizeof(n_vocab));

        if (n_vocab != hparams.n_vocab) {
            fprintf(stderr, "%s: invalid model file '%s' (bad vocab size %d != %d)\n",
                    __func__, fname_inp.c_str(), n_vocab, hparams.n_vocab);
            return false;
        }

        std::string word;
        for (int i = 0; i < n_vocab; i++) {
            uint32_t len;
            finp.read ((char *) &len, sizeof(len));
            fout.write((char *) &len, sizeof(len));

            word.resize(len);
            finp.read ((char *) word.data(), len);
            fout.write((char *) word.data(), len);

            vocab.token_to_id[word] = i;
            vocab.id_to_token[i] = word;
        }
    }

    // regexes of tensor names to be quantized
    const std::vector<std::string> to_quant = {
        "model/wte",
        "model/lm_head",
        "model/h.*/attn/c_attn/w",
        "model/h.*/attn/c_proj/w",
        "model/h.*/mlp/c_fc/w",
        "model/h.*/mlp/c_proj/w",
    };

    if (!ggml_common_quantize_0(finp, fout, ftype, to_quant, {})) {
        fprintf(stderr, "%s: failed to quantize model '%s'\n", __func__, fname_inp.c_str());
        return false;
    }

    finp.close();
    fout.close();

    return true;
}

```

This is a C++ program that appears to quantize a machine learning model represented in the F16 format. The program takes two command-line arguments: a file name for the input model and a file name for the output quantized model.

The program first checks the input file and reports any errors. It then initializes a feedback filter for the quantized model, which is used to filter the input and keep track of the quantized data.

The program then quantizes the input model by calling the `starcoder_model_quantize` function, which takes the name of the input file, the name of the output file, and the F16 type of the model. The function returns the quantized model in the output file.

Finally, the program reports the timing information for the quantization process and returns 0 if the quantization was successful, or 1 if it failed.


```cpp
// usage:
//  ./gpt-2-quantize models/gpt-2-117M/ggml-model.bin models/gpt-2-117M/ggml-model-quant.bin type
//
int main(int argc, char ** argv) {
    if (argc != 4) {
        fprintf(stderr, "usage: %s model-f32.bin model-quant.bin type\n", argv[0]);
        ggml_print_ftypes(stderr);
        return 1;
    }

    // needed to initialize f16 tables
    {
        struct ggml_init_params params = { 0, NULL, false };
        struct ggml_context * ctx = ggml_init(params);
        ggml_free(ctx);
    }

    const std::string fname_inp = argv[1];
    const std::string fname_out = argv[2];

    const ggml_ftype ftype = ggml_parse_ftype(argv[3]);

    const int64_t t_main_start_us = ggml_time_us();

    int64_t t_quantize_us = 0;

    // load the model
    {
        const int64_t t_start_us = ggml_time_us();

        if (!starcoder_model_quantize(fname_inp, fname_out, ggml_ftype(ftype))) {
            fprintf(stderr, "%s: failed to quantize model from '%s'\n", __func__, fname_inp.c_str());
            return 1;
        }

        t_quantize_us = ggml_time_us() - t_start_us;
    }

    // report timing
    {
        const int64_t t_main_end_us = ggml_time_us();

        printf("\n");
        printf("%s: quantize time = %8.2f ms\n", __func__, t_quantize_us/1000.0f);
        printf("%s:    total time = %8.2f ms\n", __func__, (t_main_end_us - t_main_start_us)/1000.0f);
    }

    return 0;
}

```

# 💫 StarCoder

This is a C++ example running 💫 StarCoder inference using the [ggml](https://github.com/ggerganov/ggml) library.

The program runs on the CPU - no video card is required.

The example supports the following 💫 StarCoder models:

- `bigcode/starcoder`
- `bigcode/gpt_bigcode-santacoder` aka the smol StarCoder

Sample performance on MacBook M1 Pro:

TODO


Sample output:

```cpp
$ ./bin/starcoder -h
usage: ./bin/starcoder [options]

options:
  -h, --help            show this help message and exit
  -s SEED, --seed SEED  RNG seed (default: -1)
  -t N, --threads N     number of threads to use during computation (default: 8)
  -p PROMPT, --prompt PROMPT
                        prompt to start generation with (default: random)
  -n N, --n_predict N   number of tokens to predict (default: 200)
  --top_k N             top-k sampling (default: 40)
  --top_p N             top-p sampling (default: 0.9)
  --temp N              temperature (default: 1.0)
  -b N, --batch_size N  batch size for prompt processing (default: 8)
  -m FNAME, --model FNAME
                        model path (default: models/starcoder-117M/ggml-model.bin)

$ ./bin/starcoder -m ../models/bigcode/gpt_bigcode-santacoder-ggml-q4_1.bin -p "def fibonnaci(" -t 4 --top_k 0 --top_p 0.95 --temp 0.2      
main: seed = 1683881276
starcoder_model_load: loading model from '../models/bigcode/gpt_bigcode-santacoder-ggml-q4_1.bin'
starcoder_model_load: n_vocab = 49280
starcoder_model_load: n_ctx   = 2048
starcoder_model_load: n_embd  = 2048
starcoder_model_load: n_head  = 16
starcoder_model_load: n_layer = 24
starcoder_model_load: ftype   = 3
starcoder_model_load: ggml ctx size = 1794.90 MB
starcoder_model_load: memory size =   768.00 MB, n_mem = 49152
starcoder_model_load: model size  =  1026.83 MB
main: prompt: 'def fibonnaci('
main: number of tokens in prompt = 7, first 8 tokens: 563 24240 78 2658 64 2819 7 

def fibonnaci(n):
    if n == 0:
        return 0
    elif n == 1:
        return 1
    else:
        return fibonacci(n-1) + fibonacci(n-2)

print(fibo(10))

main: mem per token =  9597928 bytes
main:     load time =   480.43 ms
main:   sample time =    26.21 ms
main:  predict time =  3987.95 ms / 19.36 ms per token
main:    total time =  4580.56 ms
```

## Quick start
```cppbash
git clone https://github.com/ggerganov/ggml
cd ggml

# Install Python dependencies
python3 -m pip install -r requirements.txt

# Convert HF model to ggml
python examples/starcoder/convert-hf-to-ggml.py bigcode/gpt_bigcode-santacoder

# Build ggml + examples
mkdir build && cd build
cmake .. && make -j4 starcoder starcoder-quantize

# quantize the model
./bin/starcoder-quantize ../models/bigcode/gpt_bigcode-santacoder-ggml.bin ../models/bigcode/gpt_bigcode-santacoder-ggml-q4_1.bin 3

# run inference
./bin/starcoder -m ../models/bigcode/gpt_bigcode-santacoder-ggml-q4_1.bin -p "def fibonnaci(" --top_k 0 --top_p 0.95 --temp 0.2
```


## Downloading and converting the original models (💫 StarCoder)

You can download the original model and convert it to `ggml` format using the script `convert-hf-to-ggml.py`:

```cpp
# Convert HF model to ggml
python examples/starcoder/convert-hf-to-ggml.py bigcode/gpt_bigcode-santacoder
```

This conversion requires that you have python and Transformers installed on your computer.

## Quantizing the models

You can also try to quantize the `ggml` models via 4-bit integer quantization.

```cpp
# quantize the model
./bin/starcoder-quantize ../models/bigcode/gpt_bigcode-santacoder-ggml.bin ../models/bigcode/gpt_bigcode-santacoder-ggml-q4_1.bin 3
```

| Model | Original size | Quantized size | Quantization type |
| --- | --- | --- | --- |
| `bigcode/gpt_bigcode-santacoder` | 5396.45 MB | 1026.83 MB | 4-bit integer (q4_1) |
| `bigcode/starcoder` | 71628.23 MB | 13596.23 MB | 4-bit integer (q4_1) |


# `examples/starcoder/starcoder-mmap.cpp`

这段代码是一个C++程序，它包括了两个头文件：ggml.h和common.h。ggml.h和common.h都包含了头文件ggml.h和common.h，这两个头文件没有定义函数，但它们的声明在函数的前面。

这段代码的作用是：定义了一个名为“common”的类，它包括了一个名为“ggml”的函数和一个名为“common_ggml”的函数。ggml函数的作用是“ggml”，common_ggml函数的作用是“common_ggml”。


```cpp
#include "ggml/ggml.h"

#include "common.h"
#include "common-ggml.h"

#include <cassert>
#include <cmath>
#include <cstdio>
#include <cstring>
#include <fstream>
#include <map>
#include <string>
#include <vector>

#if !defined(_WIN32)
```

这段代码定义了一个名为 "mmap" 的函数，它的作用是模拟操作系统中的 mmap 函数。这个函数可以用来动态地映射一个文件到内存中，然后通过指针 p 访问映射的内存区域。

具体来说，这段代码包含以下几行：

1. `#include <sys/types.h>`：引入了 `sys/types.h` 头文件，可能是用于定义 `typedef` 或者 `const` 类型的宏定义。
2. `#include <sys/mman.h>`：引入了 `sys/mman.h` 头文件，可能是用于模拟操作系统中的 mmap 函数。
3. `#include <unistd.h>`：引入了 `unistd.h` 头文件，可能是用于包含一个通用的文件操作函数。
4. `#include <fcntl.h>`：引入了 `fcntl.h` 头文件，可能是用于包含一个通用的文件操作函数。
5. `#ifdef GGML_USE_CUBLAS`：如果这个条件为真，那么会包含一个名为 "ggml-cuda.h" 的头文件，很可能是用于与 CUDA 库进行交互的。
6. `#endif`：用于将条件代码块和不需要的代码分离开来，起到增强代码可读性的作用。

另外，由于 `#ifdef` 和 `#define` 之间的代码块没有分号，因此它们应该被视为一个整体，作为一个简单的代码片段。


```cpp
// mmap
#include <sys/types.h>
#include <sys/mman.h>
#include <unistd.h>
#include <fcntl.h>
#else
#define NOMINMAX
#include <Windows.h>
#endif

#ifdef GGML_USE_CUBLAS
#include "ggml-cuda.h"
#endif

#ifdef GGML_USE_CLBLAST
```

这段代码包括两个部分：

1. 头文件包含：ggml-opencl.h，这是一个C++头文件，它引入了ggml-opencl库，用于在C++中使用GPT-2模型的OpenCL版本。

2. 定义结构体：starcoder_hparams，该结构体定义了参数化的哈希参数，包括词汇表大小（n_vocab）、上下文大小（n_ctx）、嵌入维度大小（n_embd）、头数（n_head）、层数（n_layer）和文件类型（ftype），以及浮点数类型（float）的邻域（eps）参数。这些参数用于配置模型架构和优化设置。


```cpp
#include "ggml-opencl.h"
#endif

// default hparams (GPT-2 117M)
// https://huggingface.co/bigcode/gpt_bigcode-santacoder/blob/main/config.json
struct starcoder_hparams {
    int32_t n_vocab = 49280;
    int32_t n_ctx   = 2048;
    int32_t n_embd  = 2048;
    int32_t n_head  = 16;
    int32_t n_layer = 24;
    int32_t ftype   = 1;
    float   eps     = 1e-5f;
};

```

这段代码定义了一个名为 `starcoder_layer` 的结构体，表示在一个神经网络的层中。这个结构体包含了一些用于层输出的参数。

首先，它定义了一个标量类型为 `struct ggml_tensor *` 的 `ln_1_g` 和 `ln_1_b` 成员变量，它们将用于对输入数据进行归一化操作。

接下来，它定义了一个标量类型为 `struct ggml_tensor *` 的 `ln_2_g` 和 `ln_2_b` 成员变量，它们将用于计算注意力权重。

然后，它定义了一个标量类型为 `struct ggml_tensor *` 的 `c_attn_attn_w` 和 `c_attn_attn_b` 成员变量，它们将用于计算注意力。

接着，它定义了一个标量类型为 `struct ggml_tensor *` 的 `c_attn_proj_w` 和 `c_attn_proj_b` 成员变量，它们将用于计算卷积层中的投影权重。

最后，它定义了一个标量类型为 `struct ggml_tensor *` 的 `c_mlp_fc_w` 和 `c_mlp_fc_b` 成员变量，它们将用于计算全连接层的输出参数。

综上所述，这个结构体表示一个神经网络的层，其中的参数用于对输入数据进行归一化、计算注意力权重、计算卷积层中的投影权重和全连接层的输出参数。


```cpp
struct starcoder_layer {
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

这是一个结构体变量 `llama_buffer`，它用来存储一个内存区域，可以动态调整这个区域的大小。

该结构体包含两个成员变量：

1. `addr`：一个 `uint8_t` 类型的指针，指向一个 `uint8_t` 类型的变量，初始化为 `NULL`。
2. `size`：一个 `size_t` 类型的整数，初始化为 0。

该结构体还有一个名为 `resize` 的函数，它的参数是一个表示要调整的大小的 `size_t` 类型的整数 `len`，函数内部先检查该内存区域是否已经分配好，如果没有，就释放已经分配好的内存，然后通过 `posix_memalign` 函数来分配一块足够大的内存区域，如果分配失败，就将 `addr` 变量设置为 `NULL`，并返回。最后，该函数将 `addr` 变量指向新分配的内存区域，以便使用新分配的内存区域。

该结构体的默认构造函数是定义在 `__attribute__((constructor))` 修饰下的，它会执行默认构造函数，即 `llama_buffer() = default;`，这个构造函数只是将 `addr` 变量初始化为 `NULL`，并不分配内存。

该结构体可以用来定义一个内存区域，并通过 `resize` 函数来动态调整这个区域的大小。注意，在使用该结构体之前，需要先定义好该内存区域，否则就会出现编译错误或者运行时错误。


```cpp
struct llama_buffer {
    uint8_t * addr = NULL;
    size_t size = 0;

    llama_buffer() = default;

    void resize(size_t len) {
#ifdef GGML_USE_METAL
        free(addr);
        int result = posix_memalign((void **) &addr, getpagesize(), len);
        if (result == 0) {
            memset(addr, 0, len);
        }
        else {
            addr = NULL;
        }
```

这段代码定义了一个名为`llama_buffer`的类，其作用是提供了一个`llama_buffer`类型的绑定，可以用来输出一个`llama_buffer`类型的变量，而不需要输出源代码。

具体来说，这段代码的功能如下：

1. 如果`GGML_USE_METAL`为真，则定义了一个以`addr`为参数的函数，用来在内存中删除一个`uint8_t`类型的指针，并将其复制到一个新的`uint8_t`类型的数组中。

2. 如果`GGML_USE_METAL`为假，则定义了一个以`addr`为参数的函数，用来在内存中删除一个`uint8_t`类型的指针，并将其复制到一个新的`uint8_t`类型的数组中。

3. 定义了一个`~llama_buffer`函数，用来在`llama_buffer`对象被删除时执行一些操作。

4. 定义了三个通用的成员函数重载函数，分别用来绑定`llama_buffer`类型变量、`llama_buffer`类型变量和`llama_buffer`类型变量的副本，避免不必要的拷贝和移动操作。


```cpp
#else
        delete[] addr;
        addr = new uint8_t[len];
#endif
        size = len;
    }

    ~llama_buffer() {
#ifdef GGML_USE_METAL
        free(addr);
#else
        delete[] addr;
#endif
        addr = NULL;
    }

    // disable copy and move
    llama_buffer(const llama_buffer&) = delete;
    llama_buffer(llama_buffer&&) = delete;
    llama_buffer& operator=(const llama_buffer&) = delete;
    llama_buffer& operator=(llama_buffer&&) = delete;
};


```

这段代码定义了一个名为 `kv_cache` 的结构体，用于存储键值对 `(key, value)`，同时该结构体还维护了一个指向内存的指针 `ctx` 和一个整数 `n`。

接下来定义了一个名为 `starcoder_model` 的结构体，该结构体包含了模型的参数和成员变量。

然后通过 `ggml_tensor` 和 `ggml_context` 类型的成员变量，实现了从输入到输出的一组操作。

接着通过 `llama_buffer` 类型的成员变量实现了从内存中读取数据到输入流中的操作。

最后通过 `struct starcoder_layer` 的容器类型存储了模型的各个部分，并实现了将部分信息通过 `std::vector<starcoder_layer>` 的形式进行存储和检索。

该代码的作用是：实现了一个简单的 StarCoder Model，支持从内存中读取数据到输入流中，并对模型参数和成员变量进行维护。


```cpp
struct kv_cache {
    struct ggml_tensor * k;
    struct ggml_tensor * v;

    struct ggml_context * ctx = NULL;

    //std::vector<uint8_t> buf;
    llama_buffer buf;

    int n;
};

struct starcoder_model {
    starcoder_hparams hparams;

    // normalization
    struct ggml_tensor * ln_f_g;
    struct ggml_tensor * ln_f_b;

    struct ggml_tensor * wte;     // position embedding
    struct ggml_tensor * wpe;     //    token embedding
    struct ggml_tensor * lm_head; // language model head

    std::vector<starcoder_layer> layers;

    // key + value memory
    //struct ggml_tensor * memory_k;
    //struct ggml_tensor * memory_v;
    struct kv_cache cache;

    // model memory mapped file
    void * mm_addr = NULL;
    uint64_t mm_length = 0;

    //
    struct ggml_context * ctx;
    std::map<std::string, struct ggml_tensor *> tensors;
};

```

这段代码的作用是实现了一个文件映射，它将一个指定的文件名映射到内存中的某个地址，并返回该地址。具体实现如下：

1. 首先定义了一个名为 `mmap_file` 的函数，它的参数包括一个文件名 `fname` 和一个指向内存中映射文件长度 `mm_length` 的指针。

2. 在函数定义之前，通过 `#if` 条件检查是否定义了 `_WIN32` 平台以及 `_POSIX_MAPPED_FILES` 是否为 `0`。如果是 `_WIN32`，则不执行后续代码，如果是 `_POSIX_MAPPED_FILES`，则可以执行后续代码。

3. 如果已经定义了 `_WIN32` 并且 `_POSIX_MAPPED_FILES` 不是 `0`，则创建一个文件 `fname`。

4. 使用 `CreateFileA` 函数尝试创建一个文件，并指定文件类型为 `GENERIC_READ`、`FILE_SHARE_READ` 和 `FILE_SHARE_WRITE`、`FILE_SHARE_DELETE`，同时指定文件 share 标志为 `FILE_SHARE_READ` 和 `FILE_SHARE_WRITE`，且不包含文件内容索引。

5. 如果创建成功，通过 `GetFileSizeEx` 函数获取文件长度 `fileSize`。

6. 创建一个名为 `hMapping` 的映射文件，并指定文件类型为 `PAGE_READONLY`，同时指定页大小为 `0`。

7. 关闭文件和映射文件。

8. 通过 `MapViewOfFile` 函数将文件映射到内存中的某个地址，并返回该地址。

9. 最后，如果映射失败，返回 `0`。


```cpp
// From PR #613 (https://github.com/ggerganov/llama.cpp/pull/613)
static void *mmap_file(const char *fname, uint64_t *mm_length) {
#if defined(_WIN32) && !defined(_POSIX_MAPPED_FILES)
    HANDLE hFile = CreateFileA(fname,
                               GENERIC_READ,
                               FILE_SHARE_READ | FILE_SHARE_WRITE | FILE_SHARE_DELETE,
                               NULL,
                               OPEN_EXISTING,
                               FILE_ATTRIBUTE_NORMAL | FILE_ATTRIBUTE_NOT_CONTENT_INDEXED,
                               NULL);
    if (hFile == INVALID_HANDLE_VALUE) return 0;
    LARGE_INTEGER fileSize;
    fileSize.QuadPart = -1;
    GetFileSizeEx(hFile, &fileSize);
    int64_t length = fileSize.QuadPart;
    HANDLE hMapping = CreateFileMappingA(hFile, NULL, PAGE_READONLY, 0, 0, NULL);
    CloseHandle(hFile);
    if (!hMapping) return 0;
    void *addr = MapViewOfFile(hMapping, FILE_MAP_READ, 0, 0, 0);
    CloseHandle(hMapping);
    if (!addr) return 0;
```

这段代码是一个C语言函数，名为`mmunmap_file`。它实现了一个内存映射函数，用于将一个文件中的内容映射到系统的内存中。

函数接收两个参数：一个void类型的指针`addr`，表示需要映射到内存中的起始地址，以及一个int64类型的整数`length`，表示文件的内容长度。函数内部使用这两个参数，根据需要在文件中找到起始地址和内容长度，并返回一个void类型的指针`addr`，表示在内存中的起始地址。

函数首先通过调用C标准库中的`open`函数，以只读方式打开一个文件`fname`，并返回其文件描述符`fd`。然后使用`lseek`函数在文件中查找起始地址，并返回它。接下来，使用`mmap`函数将文件内容从文件中映射到系统的内存中，并返回一个void类型的指针`addr`。最后，使用`close`函数关闭文件，并在内存映射过程中出错时返回0。

函数还有一个辅助函数`munmap_file`，它接收两个参数：`addr`和`length`。函数首先使用`UnmapViewOfFile`函数从文件中卸载`addr`所指的内存空间，然后使用`size_t`类型的变量`length`，表示文件的内容长度。这两个函数是在`munmap_file`函数中作用，用于确保函数使用正确的内存区域，并避免访问越界内存区域。


```cpp
#else
    int fd = open(fname, O_RDONLY);
    if (fd == -1) return 0;
    int64_t length = lseek(fd, 0, SEEK_END);
    void *addr = mmap(NULL, length, PROT_READ, MAP_SHARED, fd, 0);
    close(fd);
    if (addr == MAP_FAILED) return 0;
#endif
    *mm_length = length;
    return addr;
}

static void munmap_file(void * addr, size_t length) {
#if defined(_WIN32) && !defined(_POSIX_MAPPED_FILES)
    UnmapViewOfFile(addr);
```

基于对问题的分析，我得出的答案是B。


```cpp
#else
    munmap(addr, length);
#endif
}

// load the model's weights from a file
bool starcoder_model_load(const std::string & fname, starcoder_model & model, gpt_vocab & vocab, int32_t n_gpu_layers) {
    printf("%s: loading model from '%s'\n", __func__, fname.c_str());

    auto fin = std::ifstream(fname, std::ios::binary);
    if (!fin) {
        fprintf(stderr, "%s: failed to open '%s'\n", __func__, fname.c_str());
        return false;
    }

    std::vector<char> f_buf(1024*1024);
    fin.rdbuf()->pubsetbuf(f_buf.data(), f_buf.size());


    // verify magic
    {
        uint32_t magic;
        fin.read((char *) &magic, sizeof(magic));
        //if (magic != 0x67676a74) {
        if (magic != 0x67676d6c) {
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

            // if (i < 10) fprintf(stderr, "%.s: vocab[%d] = '%s'\n", __func__, i, word.c_str());
        }

        // Add StarChat special tokens.
        for (std::string token : {
                "<|system|>",
                "<|user|>",
                "<|assistant|>",
                "<|end|>",
                }) {
            if (vocab.token_to_id.find(token) != vocab.token_to_id.end()) {
                vocab.add_special_token(token);
            }
        }
    }

    char *mm_addr = NULL;
    model.mm_addr = mmap_file(fname.c_str(), &model.mm_length);
    if (model.mm_addr == NULL) {
        fprintf(stderr, "%s: failed to mmap '%s'\n", __func__, fname.c_str());
        return false;
    }
    mm_addr = (char *)model.mm_addr;
    fprintf(stderr, "%s: ggml map size = %6.2f MB\n", __func__, model.mm_length/(1024.0*1024.0));

    // for the big tensors, we have the option to store the data in 16-bit floats or quantized
    // in order to save memory and also to speed up the computation
    ggml_type wtype = ggml_ftype_to_ggml_type((ggml_ftype) (model.hparams.ftype));
    if (wtype == GGML_TYPE_COUNT) {
        fprintf(stderr, "%s: invalid model file '%s' (bad ftype value %d)\n",
                __func__, fname.c_str(), model.hparams.ftype);
        return false;
    }

    auto & ctx = model.ctx;

    size_t ctx_size = 0;

    {
        const auto & hparams = model.hparams;

        const int n_layer = hparams.n_layer;


        /*
        const int n_embd  = hparams.n_embd;
        const int n_layer = hparams.n_layer;
        const int n_ctx   = hparams.n_ctx;
        const int n_vocab = hparams.n_vocab;

        const int head_dim = n_embd / hparams.n_head;
        const int kv_heads = hparams.n_head; // 1 if MQA else hparams.n_head
        const int kv_dim   = kv_heads * head_dim;


        ctx_size += n_embd*ggml_type_sizef(GGML_TYPE_F32); // ln_f_g
        ctx_size += n_embd*ggml_type_sizef(GGML_TYPE_F32); // ln_f_b

        ctx_size += n_vocab*n_embd*ggml_type_sizef(wtype);         // wte
        ctx_size +=   n_ctx*n_embd*ggml_type_sizef(GGML_TYPE_F32); // wpe
        ctx_size += n_vocab*n_embd*ggml_type_sizef(wtype);         // lm_head

        ctx_size += n_layer*(n_embd*ggml_type_sizef(GGML_TYPE_F32)); // ln_1_g
        ctx_size += n_layer*(n_embd*ggml_type_sizef(GGML_TYPE_F32)); // ln_1_b

        ctx_size += n_layer*(n_embd*ggml_type_sizef(GGML_TYPE_F32)); // ln_2_g
        ctx_size += n_layer*(n_embd*ggml_type_sizef(GGML_TYPE_F32)); // ln_2_b

        ctx_size += n_layer*((n_embd + 2*kv_dim)*n_embd*ggml_type_sizef(wtype));         // c_attn_attn_w // TODO:
        ctx_size += n_layer*(       (n_embd + 2*kv_dim)*ggml_type_sizef(GGML_TYPE_F32)); // c_attn_attn_b

        ctx_size += n_layer*(n_embd*n_embd*ggml_type_sizef(wtype));           // c_attn_proj_w
        ctx_size += n_layer*(       n_embd*ggml_type_sizef(GGML_TYPE_F32));   // c_attn_proj_b

        ctx_size += n_layer*(4*n_embd*n_embd*ggml_type_sizef(wtype));         // c_mlp_fc_w
        ctx_size += n_layer*(       4*n_embd*ggml_type_sizef(GGML_TYPE_F32)); // c_mlp_fc_b

        ctx_size += n_layer*(4*n_embd*n_embd*ggml_type_sizef(wtype));         // c_mlp_proj_w
        ctx_size += n_layer*(         n_embd*ggml_type_sizef(GGML_TYPE_F32)); // c_mlp_proj_b

        ctx_size += n_ctx*n_layer*n_embd*ggml_type_sizef(GGML_TYPE_F32); // memory_k
        ctx_size += n_ctx*n_layer*n_embd*ggml_type_sizef(GGML_TYPE_F32); // memory_v
        */

        ctx_size += (6 + 12*n_layer)*512; // object overhead

        printf("%s: ggml ctx size = %6.2f MB\n", __func__, ctx_size/(1024.0*1024.0));
    }

    // create the ggml context
    {
        struct ggml_init_params params = {
            /*.mem_size   =*/ ctx_size,
            /*.mem_buffer =*/ NULL,
            /*.no_alloc   =*/ true,
        };

        model.ctx = ggml_init(params);
        if (!model.ctx) {
            fprintf(stderr, "%s: ggml_init() failed\n", __func__);
            return false;
        }
    }

    // prepare memory for the weights
    {
        const auto & hparams = model.hparams;

        const int n_embd  = hparams.n_embd;
        const int n_layer = hparams.n_layer;
        const int n_ctx   = hparams.n_ctx;
        const int n_vocab = hparams.n_vocab;

        const int head_dim = n_embd / hparams.n_head;
        const int kv_heads = hparams.n_head; // 1 if MQA else hparams.n_head
        const int kv_dim   = kv_heads * head_dim;

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

            layer.c_attn_attn_w = ggml_new_tensor_2d(ctx, wtype,           n_embd, n_embd + 2*kv_dim);
            layer.c_attn_attn_b = ggml_new_tensor_1d(ctx, GGML_TYPE_F32, n_embd + 2*kv_dim);

            layer.c_attn_proj_w = ggml_new_tensor_2d(ctx, wtype,           n_embd, n_embd);
            layer.c_attn_proj_b = ggml_new_tensor_1d(ctx, GGML_TYPE_F32,   n_embd);

            layer.c_mlp_fc_w    = ggml_new_tensor_2d(ctx, wtype,           n_embd, 4*n_embd); //TODO: 4*n_embd = config.n_inner
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

    // key + value memory
    {
        const auto & hparams = model.hparams;

        const int n_embd  = hparams.n_embd;
        const int n_layer = hparams.n_layer;
        const int n_ctx   = hparams.n_ctx;

        const int n_mem      = n_layer*n_ctx;
        const int n_elements = n_embd*n_mem;

        model.cache.buf.resize(2u*n_elements*ggml_type_size(GGML_TYPE_F16) + 2u*1024*1024);

        struct ggml_init_params c_params;
        c_params.mem_size   = model.cache.buf.size;
        c_params.mem_buffer = model.cache.buf.addr;
        c_params.no_alloc   = false;

        model.cache.ctx = ggml_init(c_params);

        if (!model.cache.ctx) {
            fprintf(stderr, "%s: failed to allocate memory for kv cache\n", __func__);
            return false;
        }

        model.cache.k = ggml_new_tensor_1d(model.cache.ctx, GGML_TYPE_F16, n_elements);
        model.cache.v = ggml_new_tensor_1d(model.cache.ctx, GGML_TYPE_F16, n_elements);

        const size_t memory_size = ggml_nbytes(model.cache.k) + ggml_nbytes(model.cache.v);

        printf("%s: kv_cache memory size = %8.2f MB, n_mem = %d\n", __func__, memory_size/1024.0/1024.0, n_mem);
    }

    // load weights
    {
        size_t total_size = 0;

        bool has_lm_head = false;

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

            if (model.tensors.find(name.data()) == model.tensors.end()) {
                fprintf(stderr, "%s: unknown tensor '%s' in model file\n", __func__, name.data());
                return false;
            }

            auto tensor = model.tensors[name.data()];

            if (tensor->ne[0] != ne[0] || tensor->ne[1] != ne[1]) {
                fprintf(stderr, "%s: tensor '%s' has wrong shape in model file: got [%d, %d], expected [%d, %d]\n",
                        __func__, name.data(), (int) tensor->ne[0], (int) tensor->ne[1], ne[0], ne[1]);
                return false;
            }
            if (ggml_nelements(tensor) != nelements) {
                fprintf(stderr, "%s: tensor '%s' has wrong size in model file. got %d, expected %d\n",
                        __func__, name.data(), (int) ggml_nelements(tensor), nelements);
                return false;
            }

            // for debugging
            if (0) {
                printf("%24s - [%5d, %5d], type = %6s, %6.2f MB, %9zu bytes\n", name.data(), ne[0], ne[1], ggml_type_name(ggml_type(ttype)), ggml_nbytes(tensor)/1024.0/1024.0, ggml_nbytes(tensor));
            }

            const size_t bpe = ggml_type_size(ggml_type(ttype));

            if ((nelements*bpe)/ggml_blck_size(tensor->type) != ggml_nbytes(tensor)) {
                fprintf(stderr, "%s: tensor '%s' has wrong size in model file: got %zu, expected %zu\n",
                        __func__, name.data(), ggml_nbytes(tensor), nelements*bpe);
                return false;
            }

            // mmap
            size_t offset = fin.tellg();
            size_t tensor_data_size = ggml_nbytes(tensor);
            //offset = (offset + 31) & -32;
            tensor->data = mm_addr + offset;
            fin.seekg(offset + tensor_data_size);
            total_size += tensor_data_size;

            // GPT-2 models share the WTE tensor as the LM head
            if (name == "model/wte" && has_lm_head == false) {
                // Dont know if this is required, test models have an lm_head
                model.lm_head->data = tensor->data;
            }

            if (name == "model/lm_head") {
                has_lm_head = true;
            }
        }

        printf("%s: model size  = %8.2f MB\n", __func__, total_size/1024.0/1024.0);
    }

    fin.close();

```

This code appears to be a function definition for the CUDA layer layer that offloads the computation of the last layer's activations to a GPU device. It appears to take in a layer configuration struct and a vector of weights in the output of the last layer and outputs the total amount of VRAM used by the function. The function also disables the scratch memory and sets the scratch index to 0. The scratch memory is used to store the activations of the last layer before offloading them to the GPU device.


```cpp
#ifdef GGML_USE_CUBLAS
    {
        const auto & hparams = model.hparams;
        const int n_gpu = std::min(n_gpu_layers, int(hparams.n_layer));

        fprintf(stderr, "%s: [cublas] offloading %d layers to GPU\n", __func__, n_gpu);

        size_t vram_total = 0;

        for (int i = 0; i < n_gpu; ++i) {
            const auto & layer = model.layers[i];

            layer.c_attn_attn_w->backend = GGML_BACKEND_GPU;
            ggml_cuda_transform_tensor((uint8_t *)layer.c_attn_attn_w->data, layer.c_attn_attn_w); vram_total += ggml_nbytes(layer.c_attn_attn_w);

            layer.c_attn_proj_w->backend = GGML_BACKEND_GPU;
            ggml_cuda_transform_tensor((uint8_t *)layer.c_attn_proj_w->data, layer.c_attn_proj_w); vram_total += ggml_nbytes(layer.c_attn_proj_w);

            layer.c_mlp_fc_w->backend = GGML_BACKEND_GPU;
            ggml_cuda_transform_tensor((uint8_t *)layer.c_mlp_fc_w->data, layer.c_mlp_fc_w); vram_total += ggml_nbytes(layer.c_mlp_fc_w);

            layer.c_mlp_proj_w->backend = GGML_BACKEND_GPU;
            ggml_cuda_transform_tensor((uint8_t *)layer.c_mlp_proj_w->data, layer.c_mlp_proj_w); vram_total += ggml_nbytes(layer.c_mlp_proj_w);
        }

        ggml_cuda_set_scratch_size(0); // disable scratch

        //if (n_gpu_layers > (int) hparams.n_layer) {
        //    fprintf(stderr, "%s: [cublas] offloading output layer to GPU\n", __func__);
        //    ggml_cuda_transform_tensor(model.output); vram_total += ggml_nbytes(model.output);
        //}

        fprintf(stderr, "%s: [cublas] total VRAM used: %zu MB\n", __func__, vram_total / 1024 / 1024);
    }
```

This is a open source software, it seems to be using NVLink to offload the computation to the GPU, as well as using CLI to perform the computation.

The function `opencl_nvlink_allocate_model` is used to allocate the model to the NVLink and CLI.

The function `opencl_minimize_vrams` is used to minimize the amount of VRAM usage by offloading the computation to the GPU. This function takes in two arguments, `n_gpu_layers` and `hparams.n_layer`.

It appears that the function first checks the number of GPU layers and uses the smaller number if the number of layers is less than the number of GPU layers.

Then it loops through the layers of the model and sets the backend of the layer to GGML_BACKEND_GPU if it's on the GPU, and sets the Transform of the Attention and Mlp to use the GPU.

It then uses ggml_cl_transform\_tensor to convert the data from the Attention and Mlp to the GPU, and updates the Vram\_total variable with the amount of memory used.

It finally returns true, indicating that the function has been executed correctly.


```cpp
#elif defined(GGML_USE_CLBLAST)
    //From koboldcpp
    {
        const auto & hparams = model.hparams;
        size_t vram_total = 0;
        const int n_gpu = std::min(n_gpu_layers, int(hparams.n_layer));
        fprintf(stderr, "%s: [opencl] offloading %d layers to GPU\n", __func__, n_gpu);
        for (int i = 0; i < n_gpu; ++i) {
            const auto & layer = model.layers[i];
            layer.c_attn_attn_w->backend = GGML_BACKEND_GPU;
            layer.c_attn_proj_w->backend = GGML_BACKEND_GPU;
            layer.c_mlp_fc_w->backend = GGML_BACKEND_GPU;
            layer.c_mlp_proj_w->backend = GGML_BACKEND_GPU;
            ggml_cl_transform_tensor(layer.c_attn_attn_w->data,layer.c_attn_attn_w); vram_total += ggml_nbytes(layer.c_attn_attn_w);
            ggml_cl_transform_tensor(layer.c_attn_proj_w->data,layer.c_attn_proj_w); vram_total += ggml_nbytes(layer.c_attn_proj_w);
            ggml_cl_transform_tensor(layer.c_mlp_fc_w->data,layer.c_mlp_fc_w); vram_total += ggml_nbytes(layer.c_mlp_fc_w);
            ggml_cl_transform_tensor(layer.c_mlp_proj_w->data,layer.c_mlp_proj_w); vram_total += ggml_nbytes(layer.c_mlp_proj_w);
        }
        fprintf(stderr, "%s: [opencl] total VRAM used: %zu MB\n", __func__, vram_total / 1024 / 1024);
    }
    #endif

    return true;
}

```

This function appears to perform a simple language model forward and calculate the logits for a given input sequence.

It first loops through the input sequence and performs a pointwise product with the first token to create a non-linear space for the input.

Then, it performs a softmax operation on this input to convert it to a probability distribution over the vocabulary.

Finally, it runs the computation for the input sequence and returns the result, including the logits for each token.

It is based on the model you provided, which is a simplified version of the language model architecture proposed by Girshick.


```cpp
// evaluate the transformer
//
//   - model:     the model
//   - n_threads: number of threads to use
//   - n_past:    the context size so far
//   - embd_inp:  the embeddings of the tokens in the context
//   - embd_w:    the predicted logits for the next token
//
bool starcoder_eval(
        const starcoder_model & model,
        const int n_threads,
        const int n_past,
        const std::vector<gpt_vocab::id> & embd_inp,
              std::vector<float>         & embd_w,
              size_t                     & mem_per_token) {

    const int N = int(embd_inp.size());

    const auto & hparams = model.hparams;

    auto & cache = model.cache;

    const int n_embd  = hparams.n_embd;
    const int n_layer = hparams.n_layer;
    const int n_ctx   = hparams.n_ctx;
    const int n_head  = hparams.n_head;
    const int n_vocab = hparams.n_vocab;

    // Scratch is too small for large n_batch (256)
    //static size_t buf_size = 256u*1024*1024;
    static size_t buf_size = 256u*1024*1024*2;
    static void * buf = malloc(buf_size);

    // use 2 scratch buffers
    // TODO: very hacky solution - reimplement in a more elegant way
    static size_t scratch0_size = 256u*1024*1024*2;
    static void * scratch0 = malloc(scratch0_size);

    static size_t scratch1_size = 256u*1024*1024*2;
    static void * scratch1 = malloc(scratch1_size);

    if (mem_per_token > 0 && mem_per_token*N > buf_size) {
        const size_t buf_size_new = size_t(1.1*(mem_per_token*N)); // add 10% to account for ggml object overhead
        printf("\n%s: reallocating buffer from %zu to %zu bytes\n", __func__, buf_size, buf_size_new);

        // reallocate
        buf_size = buf_size_new;
        buf = realloc(buf, buf_size);
        if (buf == nullptr) {
            fprintf(stderr, "%s: failed to allocate %zu bytes\n", __func__, buf_size);
            return false;
        }
    }

    struct ggml_init_params params = {
        /*.mem_size   =*/ buf_size,
        /*.mem_buffer =*/ buf,
        /*.no_alloc   =*/ false,
    };

    struct ggml_context * ctx0 = ggml_init(params);
    struct ggml_cgraph * gf = ggml_new_graph(ctx0);

    struct ggml_tensor * embd = ggml_new_tensor_1d(ctx0, GGML_TYPE_I32, N);


    memcpy(embd->data, embd_inp.data(), N*ggml_element_size(embd));

    struct ggml_tensor * position = ggml_new_tensor_1d(ctx0, GGML_TYPE_I32, N);
    for (int i = 0; i < N; ++i) {
        ((int32_t *) position->data)[i] = n_past + i;
    }

    // wte + wpe
    struct ggml_tensor * inpL =
        ggml_add(ctx0,
                ggml_get_rows(ctx0, model.wte, embd),
                ggml_get_rows(ctx0, model.wpe, position));

    for (int il = 0; il < n_layer; ++il) {
        struct ggml_tensor * cur;

        ggml_set_scratch(ctx0, { 0, scratch0_size, scratch0, });

        // norm
        {
            // [ 768, N]
            cur = ggml_norm(ctx0, inpL, hparams.eps);

            // cur = ln_1_g*cur + ln_1_b
            // [ 768, N]
            cur = ggml_add(ctx0,
                    ggml_mul(ctx0,
                        ggml_repeat(ctx0, model.layers[il].ln_1_g, cur),
                        cur),
                    ggml_repeat(ctx0, model.layers[il].ln_1_b, cur));
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
                    ggml_repeat(ctx0, model.layers[il].c_attn_attn_b, cur),
                    cur);
        }

        // self-attention
        {
            struct ggml_tensor * Qcur = ggml_view_2d(ctx0, cur, n_embd, N, cur->nb[1], 0*sizeof(float)*n_embd);
            struct ggml_tensor * Kcur = ggml_view_2d(ctx0, cur, n_embd, N, cur->nb[1], 1*sizeof(float)*n_embd);
            struct ggml_tensor * Vcur = ggml_view_2d(ctx0, cur, n_embd, N, cur->nb[1], 2*sizeof(float)*n_embd);

            // store key and value to memory
            if (N >= 1) {
                struct ggml_tensor * k = ggml_view_1d(ctx0, cache.k, N*n_embd, (ggml_element_size(cache.k)*n_embd)*(il*n_ctx + n_past));
                struct ggml_tensor * v = ggml_view_1d(ctx0, cache.v, N*n_embd, (ggml_element_size(cache.v)*n_embd)*(il*n_ctx + n_past));

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
                            ggml_view_1d(ctx0, cache.k, (n_past + N)*n_embd, il*n_ctx*ggml_element_size(cache.k)*n_embd),
                            n_embd/n_head, n_head, n_past + N),
                        0, 2, 1, 3); //TODO: need to be tiled

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
            struct ggml_tensor * KQ = ggml_mul_mat(ctx0, K, Q); //TODO: check if it broadcasts

            // KQ_scaled = KQ / sqrt(n_embd/n_head)
            // [n_past + N, N, 12]
            struct ggml_tensor * KQ_scaled =
                ggml_scale_inplace(ctx0,
                        KQ,
                        ggml_new_f32(ctx0, 1.0f/sqrt(float(n_embd)/n_head))
                        );

            // KQ_masked = mask_past(KQ_scaled)
            // [n_past + N, N, 12]
            struct ggml_tensor * KQ_masked = ggml_diag_mask_inf_inplace(ctx0, KQ_scaled, n_past);

            // KQ = soft_max(KQ_masked)
            // [n_past + N, N, 12]
            struct ggml_tensor * KQ_soft_max = ggml_soft_max_inplace(ctx0, KQ_masked);

            // V_trans = Vmem.view(n_embd/n_head, n_head, n_past + N).permute(1, 2, 0, 3).contiguous()
            // [n_past + N, 64, 12]
            struct ggml_tensor * V_trans =
                ggml_cpy(ctx0,
                        ggml_permute(ctx0,
                            ggml_reshape_3d(ctx0,
                                ggml_view_1d(ctx0, cache.v, (n_past + N)*n_embd, il*n_ctx*ggml_element_size(cache.v)*n_embd),
                                n_embd/n_head, n_head, n_past + N),
                            1, 2, 0, 3),
                        ggml_new_tensor_3d(ctx0, cache.v->type, n_past + N, n_embd/n_head, n_head));

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
                    ggml_repeat(ctx0, model.layers[il].c_attn_proj_b, cur),
                    cur);
        }

        // add the input
        cur = ggml_add(ctx0, cur, inpL);

        struct ggml_tensor * inpFF = cur;

        ggml_set_scratch(ctx0, { 0, scratch1_size, scratch1, });

        // feed-forward network
        {
            // norm
            {
                cur = ggml_norm(ctx0, inpFF, hparams.eps);

                // cur = ln_2_g*cur + ln_2_b
                // [ 768, N]
                cur = ggml_add(ctx0,
                        ggml_mul(ctx0,
                            ggml_repeat(ctx0, model.layers[il].ln_2_g, cur),
                            cur),
                        ggml_repeat(ctx0, model.layers[il].ln_2_b, cur));
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
                    ggml_repeat(ctx0, model.layers[il].c_mlp_fc_b, cur),
                    cur);

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
                    ggml_repeat(ctx0, model.layers[il].c_mlp_proj_b, cur),
                    cur);
        }

        // input for next layer
        inpL = ggml_add(ctx0, cur, inpFF);
    }

    ggml_set_scratch(ctx0, { 0, scratch0_size, scratch0, });

    // norm
    {
        // [ 768, N]
        inpL = ggml_norm(ctx0, inpL, hparams.eps);

        // inpL = ln_f_g*inpL + ln_f_b
        // [ 768, N]
        inpL = ggml_add(ctx0,
                ggml_mul(ctx0,
                    ggml_repeat(ctx0, model.ln_f_g, inpL),
                    inpL),
                ggml_repeat(ctx0, model.ln_f_b, inpL));
    }

    ggml_set_scratch(ctx0, { 0, 0, nullptr, });

    // inpL = WTE * inpL
    // [ 768, 50257] - model.lm_head
    // [ 768, N]     - inpL
    inpL = ggml_mul_mat(ctx0, model.lm_head, inpL);

    // logits -> probs
    //inpL = ggml_soft_max_inplace(ctx0, inpL);

    // run the computation
    ggml_build_forward_expand(gf, inpL);
    ggml_graph_compute_with_ctx(ctx0, gf, n_threads);

    //if (n_past%100 == 0) {
    //    ggml_graph_print   (&gf);
    //    ggml_graph_dump_dot(&gf, NULL, "gpt-2.dot");
    //}

    //embd_w.resize(n_vocab*N);
    //memcpy(embd_w.data(), ggml_get_data(inpL), sizeof(float)*n_vocab*N);

    // return result just for the last token
    embd_w.resize(n_vocab);
    memcpy(embd_w.data(), (float *) ggml_get_data(inpL) + (n_vocab*(N-1)), sizeof(float)*n_vocab);

    if (mem_per_token == 0) {
        mem_per_token = ggml_used_mem(ctx0)/N;
    }
    //printf("used_mem = %zu MB\n", ggml_used_mem(ctx0)/(1024*1024));

    ggml_free(ctx0);

    return true;
}


```

This is a C++ function that performs model training on a text-based data source. It takes in an embedded document (embd) and a training dataset (dataset), and trains a model for a specific task using the STarcoder algorithm. 

The function starts by iterating through the embedded document and each token. For each token, it checks if it is a model-specific token (i.e. a "<|end|>" token for STarcoder) or a StarChat-specific token (i.e. a "starcoder<|end|>" token). If the token is not a model-specific or StarChat-specific token, it skips to the next iteration. 

If the token is a model-specific token, it performs some additional checks. If the token is a STarcoder "<|end|>" token, it will train the model for the STarcoder algorithm. If the token is not a STarcoder "<|end|>" token but is a StarChat-specific token, it will handle the StarChat "<|end|>" token.

The function also includes some additional code to report the timing of the training process. It reports the mem used for each token, the load time taken to process the entire dataset, and the sample time taken to process each token. It also checks if the input prompt should beSubracted, and it also reports the total time taken to train the model.

The function also includes some internal functions that are not defined in the code provided. For example, it includes a function called ggml\_time\_us() that returns the current US time as a number of milliseconds. It also includes a function called ggml\_free() that frees the memory associated with the embedded document.


```cpp
int main(int argc, char ** argv) {
    ggml_time_init();

    const int64_t t_main_start_us = ggml_time_us();

    gpt_params params;
    params.model = "models/gpt-2-117M/ggml-model.bin";

    if (gpt_params_parse(argc, argv, params) == false) {
        return 1;
    }

    if (params.seed < 0) {
        params.seed = int(time(NULL));
    }

    printf("%s: seed = %d\n", __func__, params.seed);

    std::mt19937 rng(params.seed);
    if (params.prompt.empty()) {
        params.prompt = gpt_random_prompt(rng);
    }

    int64_t t_load_us = 0;

    gpt_vocab vocab;
    starcoder_model model;

    // load the model
    {
        const int64_t t_start_us = ggml_time_us();

        if (!starcoder_model_load(params.model, model, vocab, params.n_gpu_layers)) {
            fprintf(stderr, "%s: failed to load model from '%s'\n", __func__, params.model.c_str());
            return 1;
        }

        t_load_us = ggml_time_us() - t_start_us;

        test_gpt_tokenizer(vocab, params.token_test);
    }

    int n_past = 0;

    int64_t t_sample_us  = 0;
    int64_t t_predict_us = 0;

    std::vector<float> logits;

    // tokenize the prompt
    std::vector<gpt_vocab::id> embd_inp = ::gpt_tokenize(vocab, params.prompt);

    params.n_predict = std::min(params.n_predict, model.hparams.n_ctx - (int) embd_inp.size());

    printf("%s: prompt: '%s'\n", __func__, params.prompt.c_str());
    printf("%s: number of tokens in prompt = %zu\n", __func__, embd_inp.size());
    for (size_t i = 0; i < embd_inp.size(); i++) {
        printf("%s: token[%zu] = %6d, %s\n", __func__, i, embd_inp[i], vocab.id_to_token.at(embd_inp[i]).c_str());
    }
    printf("\n\n");

    // Handle StarChat "<|end|>" token.
    gpt_vocab::id starchat_end_token = -1;
    {
        const auto it = vocab.token_to_id.find("<|end|>");
        if (it != vocab.token_to_id.end()) {
            starchat_end_token = it->second;
        }
    }

    // submit the input prompt token-by-token
    // this reduces the memory usage during inference, at the cost of a bit of speed at the beginning
    std::vector<gpt_vocab::id> embd;

    // determine the required inference memory per token:
    size_t mem_per_token = 0;
    printf("Calling starcoder_eval\n");
    starcoder_eval(model, params.n_threads, 0, { 0, 1, 2, 3 }, logits, mem_per_token);

    for (size_t i = embd.size(); i < embd_inp.size() + params.n_predict; i++) {
        // predict
        if (embd.size() > 0) {
            const int64_t t_start_us = ggml_time_us();

            if (!starcoder_eval(model, params.n_threads, n_past, embd, logits, mem_per_token)) {
                printf("Failed to predict\n");
                return 1;
            }

            // Should input processing count towards t_predict?
            if (i > embd_inp.size()) {
                t_predict_us += ggml_time_us() - t_start_us;
            }
        }

        n_past += int(embd.size());
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
            i += int(embd.size()) - 1;
        }

        // display text
        for (auto id : embd) {
            printf("%s", vocab.id_to_token[id].c_str());
        }
        fflush(stdout);

        // check if model is santacoder
        if (model.hparams.n_layer <= 30 && embd.back() == 49152) {
            break;
        }
        // check if model is starcoder
        else if (embd.back() == 0) { //TODO: this is only for starcoder
            break;
        }
        // Handle StarChat "<|end|>" token.
        else if (embd.back() == starchat_end_token) {
            //break;
        }
    }

    // report timing
    {
        const int64_t t_main_end_us = ggml_time_us();

        printf("\n\n");
        printf("%s: mem per token = %8zu bytes\n", __func__, mem_per_token);
        printf("%s:     load time = %8.2f ms\n", __func__, t_load_us/1000.0f);
        printf("%s:   sample time = %8.2f ms\n", __func__, t_sample_us/1000.0f);
        //Shouldnt the input prompt be subracted?
        printf("%s:  predict time = %8.2f ms / %.2f ms per token\n", __func__, t_predict_us/1000.0f, t_predict_us/1000.0f/(n_past - embd_inp.size()));
        //printf("%s:  predict time = %8.2f ms / %.2f ms per token\n", __func__, t_predict_us/1000.0f, t_predict_us/1000.0f/n_past);

        printf("%s:    total time = %8.2f ms\n", __func__, (t_main_end_us - t_main_start_us)/1000.0f);
    }

    ggml_free(model.ctx);

    if (model.mm_addr) {
	    munmap_file(model.mm_addr, model.mm_length);
    }

    return 0;
}

```