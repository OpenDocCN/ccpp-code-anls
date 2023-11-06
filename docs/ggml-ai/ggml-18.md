# GGML源码解析 18

# `examples/whisper/quantize.cpp`

这段代码是一个C++程序，它包括一个头文件ggml.h和两个头文件common.h和common-ggml.h。ggml.h和common-ggml.h都继承自一个公共的头文件ggml.h。

这段代码的作用是定义了一个ggml类型的数据结构类ggml，可能用于存储一些数据。ggml类可能包含了一些成员变量和一个成员函数，不过由于没有给出完整的代码，所以无法确定这些成员函数的具体实现。

ggml类的一个可能的实现是：它是一个结构体，其中包含一个整数类型的成员变量表示图的ID，另一个整数类型的成员变量表示图的版本号。可能的成员函数包括添加节点、添加边、打印图等。不过由于没有给出完整的代码，所以无法确定这些函数的实现。


```cpp
#include "ggml.h"

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

这段代码定义了一个名为whisper_hparams的结构体，其中包含了一些与语音识别相关的参数。

default hparams是一个默认的whisper_hparams结构体，其中所有成员变量都未被初始化。

whisper_hparams结构体包含以下参数：

- n_vocab：词汇表大小，值为51864。
- n_audio_ctx：音频上下文数量，值为1500。
- n_audio_state：音频状态数量，值为384。
- n_audio_head：音频头宁数量，值为6。
- n_audio_layer：音频层数量，值为4。
- n_text_ctx：文本上下文数量，值为448。
- n_text_state：文本状态数量，值为384。
- n_text_head：文本头宁数量，值为6。
- n_text_layer：文本层数量，值为4。
- n_mels: Mel计算数量，值为80。
- ftype：输入的音频数据类型，值为1。

这些参数对于语音识别任务非常重要，它们可以在模型训练和预测时提供有用的信息。


```cpp
// default hparams (Whisper tiny)
struct whisper_hparams {
    int32_t n_vocab       = 51864;
    int32_t n_audio_ctx   = 1500;
    int32_t n_audio_state = 384;
    int32_t n_audio_head  = 6;
    int32_t n_audio_layer = 4;
    int32_t n_text_ctx    = 448;
    int32_t n_text_state  = 384;
    int32_t n_text_head   = 6;
    int32_t n_text_layer  = 4;
    int32_t n_mels        = 80;
    int32_t ftype         = 1;
};

```

This function seems to be loading a pre-trained language model from a file (usually an XML or TensorFlow发病文件) and converting the model's vocabulary, which is a mapping of unique words to their index, to a Hash table that maps each index to a corresponding word. It also skips certain tensor names to be quantized.


```cpp
struct whisper_filters {
    int32_t n_mel;
    int32_t n_fft;

    std::vector<float> data;
};

// quantize a model
bool whisper_model_quantize(const std::string & fname_inp, const std::string & fname_out, ggml_ftype ftype) {
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

    whisper_hparams hparams;

    // load hparams
    {
        finp.read((char *) &hparams.n_vocab,       sizeof(hparams.n_vocab));
        finp.read((char *) &hparams.n_audio_ctx,   sizeof(hparams.n_audio_ctx));
        finp.read((char *) &hparams.n_audio_state, sizeof(hparams.n_audio_state));
        finp.read((char *) &hparams.n_audio_head,  sizeof(hparams.n_audio_head));
        finp.read((char *) &hparams.n_audio_layer, sizeof(hparams.n_audio_layer));
        finp.read((char *) &hparams.n_text_ctx,    sizeof(hparams.n_text_ctx));
        finp.read((char *) &hparams.n_text_state,  sizeof(hparams.n_text_state));
        finp.read((char *) &hparams.n_text_head,   sizeof(hparams.n_text_head));
        finp.read((char *) &hparams.n_text_layer,  sizeof(hparams.n_text_layer));
        finp.read((char *) &hparams.n_mels,        sizeof(hparams.n_mels));
        finp.read((char *) &hparams.ftype,         sizeof(hparams.ftype));

        const int32_t qntvr_src =    hparams.ftype / GGML_QNT_VERSION_FACTOR;
        const int32_t ftype_dst = GGML_QNT_VERSION * GGML_QNT_VERSION_FACTOR + ftype;

        fprintf(stderr, "%s: n_vocab       = %d\n", __func__, hparams.n_vocab);
        fprintf(stderr, "%s: n_audio_ctx   = %d\n", __func__, hparams.n_audio_ctx);
        fprintf(stderr, "%s: n_audio_state = %d\n", __func__, hparams.n_audio_state);
        fprintf(stderr, "%s: n_audio_head  = %d\n", __func__, hparams.n_audio_head);
        fprintf(stderr, "%s: n_audio_layer = %d\n", __func__, hparams.n_audio_layer);
        fprintf(stderr, "%s: n_text_ctx    = %d\n", __func__, hparams.n_text_ctx);
        fprintf(stderr, "%s: n_text_state  = %d\n", __func__, hparams.n_text_state);
        fprintf(stderr, "%s: n_text_head   = %d\n", __func__, hparams.n_text_head);
        fprintf(stderr, "%s: n_text_layer  = %d\n", __func__, hparams.n_text_layer);
        fprintf(stderr, "%s: n_mels        = %d\n", __func__, hparams.n_mels);
        fprintf(stderr, "%s: ftype (src)   = %d\n", __func__, hparams.ftype);
        fprintf(stderr, "%s: qntvr (src)   = %d\n", __func__, qntvr_src);
        fprintf(stderr, "%s: ftype (dst)   = %d\n", __func__, ftype_dst);
        fprintf(stderr, "%s: qntvr (dst)   = %d\n", __func__, GGML_QNT_VERSION);

        fout.write((const char *) &hparams.n_vocab,       sizeof(hparams.n_vocab));
        fout.write((const char *) &hparams.n_audio_ctx,   sizeof(hparams.n_audio_ctx));
        fout.write((const char *) &hparams.n_audio_state, sizeof(hparams.n_audio_state));
        fout.write((const char *) &hparams.n_audio_head,  sizeof(hparams.n_audio_head));
        fout.write((const char *) &hparams.n_audio_layer, sizeof(hparams.n_audio_layer));
        fout.write((const char *) &hparams.n_text_ctx,    sizeof(hparams.n_text_ctx));
        fout.write((const char *) &hparams.n_text_state,  sizeof(hparams.n_text_state));
        fout.write((const char *) &hparams.n_text_head,   sizeof(hparams.n_text_head));
        fout.write((const char *) &hparams.n_text_layer,  sizeof(hparams.n_text_layer));
        fout.write((const char *) &hparams.n_mels,        sizeof(hparams.n_mels));
        fout.write((const char *) &ftype_dst,             sizeof(hparams.ftype));
    }

    // load mel filters
    {
        whisper_filters filters;

        finp.read ((char *) &filters.n_mel, sizeof(filters.n_mel));
        fout.write((char *) &filters.n_mel, sizeof(filters.n_mel));
        finp.read ((char *) &filters.n_fft, sizeof(filters.n_fft));
        fout.write((char *) &filters.n_fft, sizeof(filters.n_fft));

        filters.data.resize(filters.n_mel * filters.n_fft);
        finp.read ((char *) filters.data.data(), filters.data.size() * sizeof(float));
        fout.write((char *) filters.data.data(), filters.data.size() * sizeof(float));
    }

    // load vocab
    {
        int32_t n_vocab = 0;
        finp.read ((char *) &n_vocab, sizeof(n_vocab));
        fout.write((char *) &n_vocab, sizeof(n_vocab));

        //if (n_vocab != hparams.n_vocab) {
        //    fprintf(stderr, "%s: invalid model file '%s' (bad vocab size %d != %d)\n",
        //            __func__, fname_inp.c_str(), n_vocab, hparams.n_vocab);
        //    return false;
        //}

        char word[129];

        for (int i = 0; i < n_vocab; i++) {
            uint32_t len;
            finp.read ((char *) &len, sizeof(len));
            fout.write((char *) &len, sizeof(len));

            word[len] = '\0';

            finp.read ((char *) word, len);
            fout.write((char *) word, len);

            vocab.token_to_id[word] = i;
            vocab.id_to_token[i] = word;
        }
    }

    // regexes of tensor names to not be quantized
    const std::vector<std::string> to_skip = {
        //"encoder.*",
        "encoder.conv1.bias",
        "encoder.conv2.bias",
        "encoder.positional_embedding",
        "decoder.positional_embedding",
    };

    if (!ggml_common_quantize_0(finp, fout, ftype, { ".*" }, to_skip)) {
        fprintf(stderr, "%s: failed to quantize model '%s'\n", __func__, fname_inp.c_str());
        return false;
    }

    finp.close();
    fout.close();

    return true;
}

```

This is a C++ program that appears to quantize a neural network model represented in the FHVD (Facebook AI Brain) model-f32.bin and model-quant.bin files. The program takes two arguments: a string specifying the input file and a string specifying the output file.

The program first checks the input file and prints an error message if it cannot be found. It then initializes a Facebook AI Brain F16 table by calling the ggml_init() function with the f16 type.

The program then takes the input and output files, and attempts to quantize the model by calling the whisper_model_quantize() function. This function takes the input file and the output file as arguments and returns the quantization time in useconds.

Finally, the program reports the total time taken to quantize the model.

Note that the program may raise an error if the input or output files cannot be found, and it is not clear what the Facebook AI Brain F16 table is used for or how it is intended to be used.


```cpp
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

        if (!whisper_model_quantize(fname_inp, fname_out, ggml_ftype(ftype))) {
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

# whisper

Port of [OpenAI's Whisper](https://github.com/openai/whisper) ASR model in C/C++ using
[ggml](https://github.com/ggerganov/ggml)

## More info

Checkout https://github.com/ggerganov/whisper.cpp

## Memory usage

| Model  | Disk   | Mem     |
| ---    | ---    | ---     |
| tiny   |  75 MB | ~280 MB |
| base   | 142 MB | ~430 MB |
| small  | 466 MB | ~1.0 GB |
| medium | 1.5 GB | ~2.6 GB |
| large  | 2.9 GB | ~4.7 GB |

## ggml format

The original models are converted to a custom binary format. This allows to pack everything needed into a single file:

- model parameters
- mel filters
- vocabulary
- weights

For more details, see the conversion script [convert-pt-to-ggml.py](convert-pt-to-ggml.py)


# `examples/whisper/whisper.cpp`

这段代码的作用是实现了一个基于深度学习的语音转文本工具，名为 Whisper。它支持多种语音识别技术，包括依赖神经网络（WHISPER 使用 CoreML 来实现）和基于 OpenVIO 的神经网络（WHISPER 使用 OpenVIO 来实现）。同时，它还支持使用机器学习库 GGML 和深度学习库 Whisper。GGML 是一个用于机器学习和数据挖掘的开源库，而 Whisper 是一个支持多种语音识别技术的库。


```cpp
#include "whisper.h"
#ifdef WHISPER_USE_COREML
#include "coreml/whisper-encoder.h"
#endif

#ifdef GGML_USE_METAL
#  include "ggml-metal.h"
#endif

#ifdef WHISPER_USE_OPENVINO
#include "openvino/whisper-openvino-encoder.h"
#endif

#include "ggml.h"
#include "ggml-alloc.h"

```

这段代码是一个C++程序，它包括了多个头文件和定义，以及一些标准库函数和头文件。它的主要作用是提供一个数学库，以方便在程序中使用常见的数学函数和算法。

头文件中包含了一些C++标准库函数，例如std::algorithm和std::map。这些函数可用于对数据进行搜索、迭代、排序等操作。定义中包括了一些常量，定义了std::math库，以便在程序中使用MATLAB中的数学函数。

std::cassert是一个assert函数，用于在程序编译时检查输入是否符合预期。

std::cmath是一个C++数学库，它定义了一些常用的数学函数，例如sin、cos、sqrt等。

std::cstdio是一个C++输入输出库，它定义了一些常用的输入输出函数，例如std::cout和std::cin。

std::cstdarg是一个C++参数库，它定义了一些常用的参数函数，例如std::argc和std::argv。

std::cstring是一个C++字符串库，它定义了一些常用的字符串操作函数，例如std::string和std::sprintf。

std::fstream是一个C++文件输入输出库，它定义了一些常用的文件输入输出函数，例如std::ifstream和std::ofstream。

std::map是一个C++地图库，它定义了一些常用的地图操作函数，例如std::map和std::set。

std::set是一个C++集合库，它定义了一些常用的集合操作函数，例如std::set和std::pred_set。

std::string是一个C++字符串库，它定义了一些常用的字符串操作函数，例如std::string和std::sprintf。

std::regex是一个C++正则表达式库，它定义了一些常用的正则表达式操作函数，例如std::regex和std::smatch。

std::random是一个C++随机数生成库，它定义了一些常用的随机数生成函数，例如std::random和std::uniform_int_distribution。

std::thread是一个C++多线程库，它定义了一些常用的线程操作函数，例如std::thread和std::atomic。

std::vector是一个C++向量库，它定义了一些常用的向量操作函数，例如std::vector和std::list。

std::reverse是一个C++字符串库，它定义了一些常用的字符串翻转函数，例如std::reverse和std::iota。

std::aggregative_difference是一个C++算法库，它定义了一些常用的算法，例如std::aggregative_difference、std::iota和std::smpdf。


```cpp
#include <algorithm>
#include <cassert>
#define _USE_MATH_DEFINES
#include <cmath>
#include <cstdio>
#include <cstdarg>
#include <cstring>
#include <fstream>
#include <map>
#include <set>
#include <string>
#include <thread>
#include <vector>
#include <regex>
#include <random>
```

这段代码包含了两个头文件和两个模板函数。第一个头文件是`<functional>`头文件，它是一个C++11标准库中的函数式编程支持库，可以提供了一系列实用的函数和类型。第二个头文件是`<msclibver>`头文件，它是一个C++头文件，定义了`_MSC_VER`宏，用于判断当前编译器是否支持某些C++新特性。

第二个部分包含了一系列的条件语句和引入头文件。如果`_MSC_VER`被定义为`0`并且`GGML_BIG_ENDIAN`被定义为`true`，那么将引入`<bit>`头文件。这个头文件可能是在某些较老的MSC编译器中定义的，用于在`<bit>`头文件中声明一些位类型变量。

然后，定义了一个模板函数`byteswap`，它接受一个`T`类型的参数，并返回一个与传入值`value`字节顺序无关的`T`类型。这个函数实现了一个字节顺序交换的特性，可以在将一个`T`类型的值作为`byte`类型的参数传递给某些函数时，不会导致这些函数出现问题。

最后，在if语句中，通过`#pragma`指令通知编译器某些代码是可选的，并不会输出任何警告。`#include <functional>`和`#include <msclibver>`头文件中的函数和类型在此处被包含，可能会在某些条件下被使用。


```cpp
#include <functional>

#if defined(_MSC_VER)
#pragma warning(disable: 4244 4267) // possible loss of data
#endif

#if defined(GGML_BIG_ENDIAN)
#include <bit>

template<typename T>
static T byteswap(T value) {
    return std::byteswap(value);
}

template<>
```

这段代码定义了一个名为`byteswap`的函数，接受一个float类型的参数，并返回一个float类型的结果。

该函数首先接受一个uint32_t类型的输入参数，并将其字节化为float类型。然后，该函数使用`std::bit_cast`函数将uint32_t类型转换为float类型。最后，该函数将接受float类型参数的值作为输入，并将其字节化为float类型。

该函数还定义了一个名为`byteswap_tensor_data`的函数，接受一个ggml_tensor类型的参数，并将其datum指针作为输入。该函数使用一个for循环，遍历ggml_nelements函数获取的tensor的元素，并将每个元素的datum值调用byteswap函数。

该函数还定义了一个名为`byteswap_tensor`的函数，接受一个ggml_tensor类型的参数，并将其datum指针作为输入。该函数使用switch语句和GGML类型名称，判断输入tensor的类型，并调用相应的byteswap函数。


```cpp
float byteswap(float value) {
    return std::bit_cast<float>(byteswap(std::bit_cast<std::uint32_t>(value)));
}

template<typename T>
static void byteswap_tensor_data(ggml_tensor * tensor) {
    T * datum = reinterpret_cast<T *>(tensor->data);
    for (int i = 0; i < ggml_nelements(tensor); i++) {
        datum[i] = byteswap(datum[i]);
    }
}

static void byteswap_tensor(ggml_tensor * tensor) {
    switch (tensor->type) {
        case GGML_TYPE_I16: {
            byteswap_tensor_data<int16_t>(tensor);
            break;
        }
        case GGML_TYPE_F16: {
            byteswap_tensor_data<ggml_fp16_t>(tensor);
            break;
        }
        case GGML_TYPE_I32: {
            byteswap_tensor_data<int32_t>(tensor);
            break;
        }
        case GGML_TYPE_F32: {
            byteswap_tensor_data<float>(tensor);
            break;
        }
        default: { // GML_TYPE_I8
            break;
        }
    }
}

```



这些宏定义是在GNU C标准中定义的，作用是定义了不同类型的张量(tensor)，包括字节平行的张量。

具体来说，这些宏定义了以下操作：

- `#define BYTESWAP_VALUE(d)` 定义了一个名为`byteswap_value`的宏，它的参数是一个整数(d)，它的作用是将整数d的值进行字节交换并返回交换后的值。
- `#define BYTESWAP_FILTERS(f)` 定义了一个名为`byteswap_filters`的宏，它的作用是在定义了输入张量f之后，对张量中的每个元素执行一次byteswap函数并将结果保存回原来的位置。这个函数会在循环中遍历f中的所有元素，并将它们按照元素索引的值替换为byteswap函数的结果。
- `#define BYTESWAP_TENSOR(t)` 定义了一个名为`byteswap_tensor`的宏，它的作用是在定义了输入张量t之后，执行byteswap函数并将结果保存回原来的位置。这个函数会在执行过程中创建一个新张量t，并将byteswap函数的结果复制到张量中。
- `#define BYTESWAP_VALUE(d) do {} while (0)` 定义了一个名为`byteswap_value`的宏，它的作用是在定义了一个空空语句，但没有对其进行定义之前，执行一次byteswap函数并将结果存储为d。这个函数会在定义之后永远为空，并且在之后无法被调用。
- `#define BYTESWAP_FILTERS(f) do {} while (0)` 定义了一个名为`byteswap_filters`的宏，它的作用是在定义了一个空空语句，但没有对其进行定义之前，执行五次byteswap函数并将结果存储到f中的每个元素中。这个函数会在定义之后永远为空，并且在之后无法被调用。
- `#define BYTESWAP_TENSOR(t) do {} while (0)` 定义了一个名为`byteswap_tensor`的宏，它的作用是在定义了一个空空语句，但没有对其进行定义之前，执行五次byteswap函数并将结果存储到新张量t中的每个元素中。这个函数会在定义之后永远为空，并且在之后无法被调用。


```cpp
#define BYTESWAP_VALUE(d) d = byteswap(d)
#define BYTESWAP_FILTERS(f)            \
    do {                              \
        for (auto & datum : f.data) { \
            datum = byteswap(datum);  \
        }                             \
    } while (0)
#define BYTESWAP_TENSOR(t)       \
    do {                         \
        byteswap_tensor(t); \
    } while (0)
#else
#define BYTESWAP_VALUE(d) do {} while (0)
#define BYTESWAP_FILTERS(f) do {} while (0)
#define BYTESWAP_TENSOR(t) do {} while (0)
```

这段代码定义了一个名为WHISPER_DEBUG的宏，其含义是当定义了WHISPER_DEBUG时，以下所有使用WHISPER_DEBUG定义的函数都会启用 verbose 调试输出，输出调试信息。

进一步地，代码中定义了一个名为WHISPER_ASSERT的宏，其含义是在定义了一个整数变量x之后，使用do-while 循环 until 0，期间对x进行判断。如果x小于0，则输出 "WHISPER_ASSERT: %s:%d: %s\n" 信息并中止程序。否则，程序继续执行。

最后，代码中包含了一个未定义的宏定义，其含义是输出调试信息并中止程序。


```cpp
#endif

#define WHISPER_ASSERT(x) \
    do { \
        if (!(x)) { \
            log("WHISPER_ASSERT: %s:%d: %s\n", __FILE__, __LINE__, #x); \
            abort(); \
        } \
    } while (0)

// define this to enable verbose trace logging - useful for debugging purposes
//#define WHISPER_DEBUG

#if defined(WHISPER_DEBUG)
#define WHISPER_PRINT_DEBUG(...) \
    do { \
        fprintf(stderr, __VA_ARGS__); \
    } while (0)
```

这段代码定义了一系列头文件和常量，用于定义一个名为 WHISPER 的库，用于在議論控制器中压缩和解压缩 GHZ 格式的数据。

具体来说，这段代码实现了一个 WHISPER_PRINT_DEBUG() 函数，用于输出println调试信息，用于在编译时检查 WHISPER_USE_FLASH_ATTN 和 WHISPER_USE_FLASH_FF 这两个宏是否被定义。如果这两个宏被定义了，则这段代码会定义一个名为 ggml_graph_compute_helper() 的函数，该函数计算 graph 的 plan，并将其提交给 ggml_graph_compute() 函数进行计算。计算完成后，ggml_graph_compute() 函数会将计算结果存储到缓冲区中，并使用 ggml_abort_callback() 函数处理任何计划出现错误的情况。

此外，还定义了 WHISPER_MAX_DECODERS 和 WHISPER_MAX_NODES 这两个常量，用于设置最大解码器和节点数量。


```cpp
#else
#define WHISPER_PRINT_DEBUG(...)
#endif

//#define WHISPER_USE_FLASH_ATTN
//#define WHISPER_USE_FLASH_FF
#define WHISPER_MAX_DECODERS 16
#define WHISPER_MAX_NODES 4096

//
// ggml helpers
//

static void ggml_graph_compute_helper(
        std::vector<uint8_t> & buf,
                 ggml_cgraph * graph,
                         int   n_threads,
      whisper_abort_callback   abort_callback,
                        void * abort_callback_data) {
    struct ggml_cplan plan = ggml_graph_plan(graph, n_threads);

    plan.abort_callback = abort_callback;
    plan.abort_callback_data = abort_callback_data;

    if (plan.work_size > 0) {
        buf.resize(plan.work_size);
        plan.work_data = buf.data();
    }

    ggml_graph_compute(graph, &plan);
}

```

This code appears to be a function that takes two 3D tensors `x` and `y` and applies the multiplication operation between them. The function uses padding to prevent divide-by-zero issues and achieve better performance.

The function first checks whether dimension 0 of `x` is at least 8 times larger than the padding. If it is not, the function returns the original multiplication result. Otherwise, the function performs the multiplication using two calls to `ggml_mul_mat` with the `padding` argument set to 32 to handle the remaining padding.

The first call to `ggml_mul_mat` performs the multiplication between the 8x8 block of `x` and the 8x8 block of `y` using multiple multiplications. The second call to `ggml_mul_mat` is used to handle the remaining padding.

The function returns a pointer to the newly created 3D tensor that represents the result of the multiplication.


```cpp
// faster matrix multiplications for tensors that do not have dimension 0 divisible by "pad"
// the idea is to represent the original matrix multiplication:
//
//   Z = X @ Y
//
// with the sum of two matrix multiplications:
//
//   Z = (X_0 @ Y_0) + (X_1 @ Y_1)
//
// here X_0 and Y_0 are views of X and Y that have dimension 0 divisible by "pad"
// and X_1 and Y_1 are the remaining views. X_1 and Y_1 end up being small matrices that can be processed with more
// general-purpose kernels
//
static struct ggml_tensor * ggml_mul_mat_pad(struct ggml_context * ctx, struct ggml_tensor * x, struct ggml_tensor * y, int pad = 32) {
    // use padding only if dimension 0 is at least 8 times larger than the padding
    // else we won't get much benefit from the optimization
    const int n_pad_req = 8;

    if (x->ne[0] % pad == 0 || x->ne[0] / pad < n_pad_req) {
        return ggml_mul_mat(ctx, x, y);
    }

    struct ggml_tensor * x_0 = ggml_view_3d(ctx, x, (x->ne[0]/pad)*pad, x->ne[1], x->ne[2], x->nb[1], x->nb[2], 0);
    struct ggml_tensor * x_1 = ggml_view_3d(ctx, x,  x->ne[0]%pad,      x->ne[1], x->ne[2], x->nb[1], x->nb[2], x_0->ne[0]*x_0->nb[0]);

    struct ggml_tensor * y_0 = ggml_view_3d(ctx, y, (y->ne[0]/pad)*pad, y->ne[1], y->ne[2], y->nb[1], y->nb[2], 0);
    struct ggml_tensor * y_1 = ggml_view_3d(ctx, y,  y->ne[0]%pad,      y->ne[1], y->ne[2], y->nb[1], y->nb[2], y_0->ne[0]*y_0->nb[0]);

    return ggml_add(ctx,
            ggml_mul_mat(ctx, x_0, y_0),
            ggml_mul_mat(ctx, x_1, y_1));
}

```

这段代码是一个C语言代码片段，其中包含了一些条件判断和定义。

首先，它定义了一个名为“ggml_mul_mat”的宏，后面会对其进行解释。

接着，定义了一个名为“MODEL_UNKNOWN”的枚举类型，它对应着模型类型为“MODEL_UNKNOWN”。

然后，定义了一个名为“MODEL_TINY”的枚举类型，它对应着模型类型为“MODEL_TINY”。

接下来，定义了一个名为“MODEL_BASE”的枚举类型，它对应着模型类型为“MODEL_BASE”。

然后，定义了一个名为“MODEL_SMALL”的枚举类型，它对应着模型类型为“MODEL_SMALL”。

接着，定义了一个名为“MODEL_MEDIUM”的枚举类型，它对应着模型类型为“MODEL_MEDIUM”。

最后，定义了一个名为“MODEL_LARGE”的枚举类型，它对应着模型类型为“MODEL_LARGE”。

总的来说，这段代码定义了一些可变参数的模型类型，用于在代码中使用。这些模型类型可以用作宏名，通过调用“ggml_mul_mat”宏来使用相应的模型类型，从而实现代码的优化。


```cpp
// TODO: check if other platforms can benefit from this optimization
#if defined(GGML_USE_METAL)
#define ggml_mul_mat ggml_mul_mat_pad
#endif

// available whisper models
enum e_model {
    MODEL_UNKNOWN,
    MODEL_TINY,
    MODEL_BASE,
    MODEL_SMALL,
    MODEL_MEDIUM,
    MODEL_LARGE,
};

```

This appears to be a list of language code names for languages that are included in the Ethnologue project. The Ethnologue is a comprehensive list of the 7,800 languages spoken around the world, and it includes information on the language family, language, and script of each language.

The list of language codes is organized by language family, and within each family, the codes are listed in alphabetical order. For example, the code for the language family "Indo-European" is "96," the code for the language family "Asiatic" is "82," and the code for the language family "Nak tones" is "77."

Each language in the list is identified by a unique code, which is used to refer to the language in various ways, such as in writing, speaking, and in linguistic studies. For example, the code "76" could refer to the language "Yiddish," which is a language spoken by Jewish people around the world, and it uses the Yiddish language.

Overall, this list provides a comprehensive overview of the languages spoken around the world and the codes used to represent each language.


```cpp
static const std::map<std::string, std::pair<int, std::string>> g_lang = {
    { "en",  { 0,  "english",         } },
    { "zh",  { 1,  "chinese",         } },
    { "de",  { 2,  "german",          } },
    { "es",  { 3,  "spanish",         } },
    { "ru",  { 4,  "russian",         } },
    { "ko",  { 5,  "korean",          } },
    { "fr",  { 6,  "french",          } },
    { "ja",  { 7,  "japanese",        } },
    { "pt",  { 8,  "portuguese",      } },
    { "tr",  { 9,  "turkish",         } },
    { "pl",  { 10, "polish",          } },
    { "ca",  { 11,  "catalan",        } },
    { "nl",  { 12,  "dutch",          } },
    { "ar",  { 13,  "arabic",         } },
    { "sv",  { 14,  "swedish",        } },
    { "it",  { 15,  "italian",        } },
    { "id",  { 16,  "indonesian",     } },
    { "hi",  { 17,  "hindi",          } },
    { "fi",  { 18,  "finnish",        } },
    { "vi",  { 19,  "vietnamese",     } },
    { "he",  { 20,  "hebrew",         } },
    { "uk",  { 21,  "ukrainian",      } },
    { "el",  { 22,  "greek",          } },
    { "ms",  { 23,  "malay",          } },
    { "cs",  { 24,  "czech",          } },
    { "ro",  { 25,  "romanian",       } },
    { "da",  { 26,  "danish",         } },
    { "hu",  { 27,  "hungarian",      } },
    { "ta",  { 28,  "tamil",          } },
    { "no",  { 29,  "norwegian",      } },
    { "th",  { 30,  "thai",           } },
    { "ur",  { 31,  "urdu",           } },
    { "hr",  { 32,  "croatian",       } },
    { "bg",  { 33,  "bulgarian",      } },
    { "lt",  { 34,  "lithuanian",     } },
    { "la",  { 35,  "latin",          } },
    { "mi",  { 36,  "maori",          } },
    { "ml",  { 37,  "malayalam",      } },
    { "cy",  { 38,  "welsh",          } },
    { "sk",  { 39,  "slovak",         } },
    { "te",  { 40,  "telugu",         } },
    { "fa",  { 41,  "persian",        } },
    { "lv",  { 42,  "latvian",        } },
    { "bn",  { 43,  "bengali",        } },
    { "sr",  { 44,  "serbian",        } },
    { "az",  { 45,  "azerbaijani",    } },
    { "sl",  { 46,  "slovenian",      } },
    { "kn",  { 47,  "kannada",        } },
    { "et",  { 48,  "estonian",       } },
    { "mk",  { 49,  "macedonian",     } },
    { "br",  { 50,  "breton",         } },
    { "eu",  { 51,  "basque",         } },
    { "is",  { 52,  "icelandic",      } },
    { "hy",  { 53,  "armenian",       } },
    { "ne",  { 54,  "nepali",         } },
    { "mn",  { 55,  "mongolian",      } },
    { "bs",  { 56,  "bosnian",        } },
    { "kk",  { 57,  "kazakh",         } },
    { "sq",  { 58,  "albanian",       } },
    { "sw",  { 59,  "swahili",        } },
    { "gl",  { 60,  "galician",       } },
    { "mr",  { 61,  "marathi",        } },
    { "pa",  { 62,  "punjabi",        } },
    { "si",  { 63,  "sinhala",        } },
    { "km",  { 64,  "khmer",          } },
    { "sn",  { 65,  "shona",          } },
    { "yo",  { 66,  "yoruba",         } },
    { "so",  { 67,  "somali",         } },
    { "af",  { 68,  "afrikaans",      } },
    { "oc",  { 69,  "occitan",        } },
    { "ka",  { 70,  "georgian",       } },
    { "be",  { 71,  "belarusian",     } },
    { "tg",  { 72,  "tajik",          } },
    { "sd",  { 73,  "sindhi",         } },
    { "gu",  { 74,  "gujarati",       } },
    { "am",  { 75,  "amharic",        } },
    { "yi",  { 76,  "yiddish",        } },
    { "lo",  { 77,  "lao",            } },
    { "uz",  { 78,  "uzbek",          } },
    { "fo",  { 79,  "faroese",        } },
    { "ht",  { 80,  "haitian creole", } },
    { "ps",  { 81,  "pashto",         } },
    { "tk",  { 82,  "turkmen",        } },
    { "nn",  { 83,  "nynorsk",        } },
    { "mt",  { 84,  "maltese",        } },
    { "sa",  { 85,  "sanskrit",       } },
    { "lb",  { 86,  "luxembourgish",  } },
    { "my",  { 87,  "myanmar",        } },
    { "bo",  { 88,  "tibetan",        } },
    { "tl",  { 89,  "tagalog",        } },
    { "mg",  { 90,  "malagasy",       } },
    { "as",  { 91,  "assamese",       } },
    { "tt",  { 92,  "tatar",          } },
    { "haw", { 93,  "hawaiian",       } },
    { "ln",  { 94,  "lingala",        } },
    { "ha",  { 95,  "hausa",          } },
    { "ba",  { 96,  "bashkir",        } },
    { "jw",  { 97,  "javanese",       } },
    { "su",  { 98,  "sundanese",      } },
};

```



This appears to be a list of table names for a SQLite database. Each table name is separated by a comma and includes several fields separated by a space.

It looks like this list is being used to define the schema of a database and the data that will be stored in each table. Each table has its own set of fields, and some tables, like "MODEL_SMALL", "MODEL_MEDIUM", and "MODEL_LARGE", have multiple fields.

Please note that without more information, it's hard to know what this list exactly is used for and how it's intended to be used.


```cpp
static const size_t MB = 1ull*1024*1024;

// TODO: avoid using GGUF
static const std::map<ggml_type, std::map<e_model, size_t>> MEM_REQ_MODEL = {
    { GGML_TYPE_F32,
        {
            { MODEL_TINY,     74ull*MB },
            { MODEL_BASE,    142ull*MB },
            { MODEL_SMALL,   466ull*MB },
            { MODEL_MEDIUM, 1464ull*MB },
            { MODEL_LARGE,  2952ull*MB },
        },
    },
    { GGML_TYPE_F16,
        {
            { MODEL_TINY,     74ull*MB },
            { MODEL_BASE,    142ull*MB },
            { MODEL_SMALL,   466ull*MB },
            { MODEL_MEDIUM, 1464ull*MB },
            { MODEL_LARGE,  2952ull*MB },
        },
    },
    { GGML_TYPE_Q4_0,
        {
            { MODEL_TINY,     26ull*MB },
            { MODEL_BASE,     50ull*MB },
            { MODEL_SMALL,   154ull*MB },
            { MODEL_MEDIUM,  470ull*MB },
            { MODEL_LARGE,   940ull*MB },
        },
    },
    { GGML_TYPE_Q4_1,
        {
            { MODEL_TINY,     32ull*MB },
            { MODEL_BASE,     58ull*MB },
            { MODEL_SMALL,   182ull*MB },
            { MODEL_MEDIUM,  562ull*MB },
            { MODEL_LARGE,  1124ull*MB },
        },
    },
    { GGML_TYPE_Q5_0,
        {
            { MODEL_TINY,     30ull*MB },
            { MODEL_BASE,     54ull*MB },
            { MODEL_SMALL,   170ull*MB },
            { MODEL_MEDIUM,  516ull*MB },
            { MODEL_LARGE,  1034ull*MB },
        },
    },
    { GGML_TYPE_Q5_1,
        {
            { MODEL_TINY,     32ull*MB },
            { MODEL_BASE,     58ull*MB },
            { MODEL_SMALL,   182ull*MB },
            { MODEL_MEDIUM,  562ull*MB },
            { MODEL_LARGE,  1124ull*MB },
        },
    },
    { GGML_TYPE_Q8_0,
        {
            { MODEL_TINY,     45ull*MB },
            { MODEL_BASE,     84ull*MB },
            { MODEL_SMALL,   268ull*MB },
            { MODEL_MEDIUM,  834ull*MB },
            { MODEL_LARGE,  1674ull*MB },
        },
    },
};

```

这两段代码定义了两个结构体：whisper_mel和whisper_filters。

whisper_mel结构体包含了三个成员变量：
1. n_len：表示 Mel 中的音符长度，即一个音符持续的时间。
2. n_len_org：表示原始的音符长度，可能是处理过程中进行过修改的原始值。
3. n_mel：表示 Mel 中的音符数量，即一个完整的音符周期内包含的音符数量。

whisper_filters结构体包含了三个成员变量：
1. n_mel：表示 Mel 中的音符数量，即一个完整的音符周期内包含的音符数量。
2. n_fft：表示FFT尺寸，用于处理将Mel信号转换为FFT信号。
3. data：用于存储whisper_mel结构体中的数据。

这两段代码的作用是定义了whisper_mel和whisper_filters两个结构体，提供了存储Mel信号数据的功能。whisper_mel结构体用于存储原始的Mel信号，whisper_filters结构体用于存储经过FFT变换的Mel信号数据。


```cpp
struct whisper_mel {
    int n_len;
    int n_len_org;
    int n_mel;

    std::vector<float> data;
};

struct whisper_filters {
    int32_t n_mel;
    int32_t n_fft;

    std::vector<float> data;
};

```

这段代码定义了一个名为`whisper_vocab`的结构体，用于表示训练数据中的词汇。

该结构体包含以下成员：

- `id`：一个32位整数，用于标识词汇。
- `token`：一个字符串类型的成员，用于表示词汇。

此外，该结构体还包含以下成员：

- `n_vocab`：词汇总数，当前为51864个词汇。
- `token_to_id`：词典映射，将词汇映射到ID。
- `id_to_token`：ID映射，将ID映射到词汇。
- `token_eot`：词汇截止时间，当前为50256。
- `token_sot`：词汇截止时间，当前为50257。
- `token_translate`：用于多语言模型的特殊词汇，当前为50357。
- `token_transcribe`：用于多语言模型的特殊词汇，当前为50358。
- `token_solm`：用于TDRZ模型的特殊词汇，当前为50359。
- `token_prev`：上一个时间戳，用于计时功能。
- `token_nosp`：上一个非时间戳，用于计时功能。
- `token_beg`：开始时间戳，用于计时功能。

该结构体还声明了一个名为`is_multilingual`的函数，用于判断当前数据集是否为多语言数据集。


```cpp
struct whisper_vocab {
    using id    = int32_t;
    using token = std::string;

    int n_vocab = 51864;

    std::map<token, id> token_to_id;
    std::map<id, token> id_to_token;

    // reference: https://github.com/openai/whisper/blob/248b6cb124225dd263bb9bd32d060b6517e067f8/whisper/tokenizer.py#L334-L349
    id token_eot        = 50256;
    id token_sot        = 50257;
    // task tokens (used only for multilingual models)
    id token_translate  = 50357;
    id token_transcribe = 50358;
    // other special tokens
    id token_solm       = 50359; // [TDRZ] used by tinydiarize models to indicate speaker turn
    id token_prev       = 50360;
    id token_nosp       = 50361;
    id token_not        = 50362; // no timestamps
    id token_beg        = 50363; // begin timestamps

    bool is_multilingual() const {
        return n_vocab == 51865;
    }
};

```

这段代码定义了一个名为 "whisper_segment" 的结构体，其中包含以下成员：

1. "t0" 和 "t1"，它们都是整数64类型的变量，可能用于表示某个时间点的时值。
2. "text"，它是一个字符串类型的变量，可能用于保存说话者要表达的话语。
3. "tokens"，它是一个 "whisper_token_data" 类型的数组，可能用于存储经过处理的语言模型输出的词汇信息。
4. "speaker_turn_next"，它是一个布尔类型的变量，可能表示当前是否是说话者的回合。

这个结构体是一个 whisper_token_data 的容器，可能用于存储经过训练的模型输出的词汇信息，用于自然语言处理和生成。


```cpp
struct whisper_segment {
    int64_t t0;
    int64_t t1;

    std::string text;

    std::vector<whisper_token_data> tokens;

    bool speaker_turn_next;
};

// medium
// hparams: {
// 'n_mels': 80,
// 'n_vocab': 51864,
```

这段代码定义了一个名为 "whisper\_hparams" 的结构体，描述了在 TinyHMM 模型中进行语音识别时的设置。这个结构体包含多个整数类型变量和一个浮点数类型变量，分别表示语音上下文、音频状态、文本上下文、文本状态和 Mel 特征的数量。

其中，whisper\_hparams 中所有包含 "int32\_t" 的变量都是表示整数的类型，而所有包含 "float" 的变量则表示浮点数类型。整数类型的变量中，有些变量给出了具体的数值，比如 n\_vocab、n\_audio\_ctx、n\_audio\_state 和 n\_audio\_head，这些值是在进行预处理时需要用到的。另外，n\_text 和 n\_audio\_layer 的变量则表示要进行文本识别的文本层和音频层的大小，而 n\_mels 变量则表示 Mel 特征的数量。

另外，whisper\_hparams 中还有一个名为 "ftype" 的浮点数类型变量，表示用于评估模型性能的分数类型。还有一个名为 "eps" 的浮点数类型变量，表示模型输出的停用条件，其值应该足够大，以允许模型在较低的准确率下仍然正常运行。


```cpp
// 'n_audio_ctx': 1500,
// 'n_audio_state': 1024,
// 'n_audio_head': 16,
// 'n_audio_layer': 24,
// 'n_text_ctx': 448,
// 'n_text_state': 1024,
// 'n_text_head': 16,
// 'n_text_layer': 24
// }
//
// default hparams (Whisper tiny)
struct whisper_hparams {
    int32_t n_vocab       = 51864;
    int32_t n_audio_ctx   = 1500;
    int32_t n_audio_state = 384;
    int32_t n_audio_head  = 6;
    int32_t n_audio_layer = 4;
    int32_t n_text_ctx    = 448;
    int32_t n_text_state  = 384;
    int32_t n_text_head   = 6;
    int32_t n_text_layer  = 4;
    int32_t n_mels        = 80;
    int32_t ftype         = 1;
    float   eps           = 1e-5f;
};

```

这段代码定义了一个名为whisper_layer_encoder的结构体，表示音频编码中的一个层。该结构体包含了以下成员：

* attn_ln_0_w：编码器的第一个时钟输入，注意这个输入与attn_ln_0_b是两个不同的结构体，它们都是ggml_tensor类型的数据。
* attn_ln_1_w：编码器的第二个时钟输入，同样注意与attn_ln_1_b不同。
* attn_q_w：编码器的查询输入，与attn_q_b相同。
* attn_k_w：编码器的键输入，与attn_k_b相同。
* attn_v_w：编码器的值输入，与attn_v_b相同。
* mlp_ln_w：编码器的mlp层线性变换的权重，与mlp_ln_b相同。
* mlp_0_w：编码器的mlp层的第一个输入，与mlp_0_b相同。
* mlp_1_w：编码器的mlp层的第二个输入，与mlp_1_b相同。

因此，整个结构体表示了一个音频编码层的参数和输入。


```cpp
// audio encoding layer
struct whisper_layer_encoder {
    // encoder.blocks.*.attn_ln
    struct ggml_tensor * attn_ln_0_w;
    struct ggml_tensor * attn_ln_0_b;

    // encoder.blocks.*.attn.out
    struct ggml_tensor * attn_ln_1_w;
    struct ggml_tensor * attn_ln_1_b;

    // encoder.blocks.*.attn.query
    struct ggml_tensor * attn_q_w;
    struct ggml_tensor * attn_q_b;

    // encoder.blocks.*.attn.key
    struct ggml_tensor * attn_k_w;

    // encoder.blocks.*.attn.value
    struct ggml_tensor * attn_v_w;
    struct ggml_tensor * attn_v_b;

    // encoder.blocks.*.mlp_ln
    struct ggml_tensor * mlp_ln_w;
    struct ggml_tensor * mlp_ln_b;

    // encoder.blocks.*.mlp.0
    struct ggml_tensor * mlp_0_w;
    struct ggml_tensor * mlp_0_b;

    // encoder.blocks.*.mlp.2
    struct ggml_tensor * mlp_1_w;
    struct ggml_tensor * mlp_1_b;
};

```

This is a struct definition for a decoder neural network block. It appears to be based on the GGML library, which is a high-level library for building graphical models using梅利达语法。 

The block appears to include various attention mechanisms for specifying which parts of the input to focus on when making predictions. These attention mechanisms include queries (Q), keys (K), and values (V), as well as a method for computing cross-attention, which is the attention between the encoder and decoder.

The block also includes various tensor types for storing intermediate results, such as the input, output, and attention logs. These tensor types are likely passed to the decoder's forward pass through the network.


```cpp
// token decoding layer
struct whisper_layer_decoder {
    // decoder.blocks.*.attn_ln
    struct ggml_tensor * attn_ln_0_w;
    struct ggml_tensor * attn_ln_0_b;

    // decoder.blocks.*.attn.out
    struct ggml_tensor * attn_ln_1_w;
    struct ggml_tensor * attn_ln_1_b;

    // decoder.blocks.*.attn.query
    struct ggml_tensor * attn_q_w;
    struct ggml_tensor * attn_q_b;

    // decoder.blocks.*.attn.key
    struct ggml_tensor * attn_k_w;

    // decoder.blocks.*.attn.value
    struct ggml_tensor * attn_v_w;
    struct ggml_tensor * attn_v_b;

    // decoder.blocks.*.cross_attn_ln
    struct ggml_tensor * cross_attn_ln_0_w;
    struct ggml_tensor * cross_attn_ln_0_b;

    // decoder.blocks.*.cross_attn.out
    struct ggml_tensor * cross_attn_ln_1_w;
    struct ggml_tensor * cross_attn_ln_1_b;

    // decoder.blocks.*.cross_attn.query
    struct ggml_tensor * cross_attn_q_w;
    struct ggml_tensor * cross_attn_q_b;

    // decoder.blocks.*.cross_attn.key
    struct ggml_tensor * cross_attn_k_w;

    // decoder.blocks.*.cross_attn.value
    struct ggml_tensor * cross_attn_v_w;
    struct ggml_tensor * cross_attn_v_b;

    // decoder.blocks.*.mlp_ln
    struct ggml_tensor * mlp_ln_w;
    struct ggml_tensor * mlp_ln_b;

    // decoder.blocks.*.mlp.0
    struct ggml_tensor * mlp_0_w;
    struct ggml_tensor * mlp_0_b;

    // decoder.blocks.*.mlp.2
    struct ggml_tensor * mlp_1_w;
    struct ggml_tensor * mlp_1_b;
};

```

This is a C++ implementation of a simple decoder for the Whisper language model. The decoder uses a Context object to manage the inputs and outputs of the decoder.

It has a CORE\_ WhisperModel and a CORE\_ WhisperHparams, which are used to configure the decoder.

The CORE\_ WhisperModel has a type field that specifies the type of the model (e.g. Model\_TOKEN\_ORDER).

The CORE\_ WhisperHparams has a PositionalEmbedding field that is a pointer to the memory buffer that will be used to store the input tokens.

The CORE\_ WhisperModel also has a ConversionMatrix field that maps the output tokens to the input tokens.

The CORE\_ WhisperModel has a LN field that is a pointer to the memory buffer that will be used to store the learnig中期荷起步。

The CORE\_ WhisperModel also has a Decoder field that is a pointer to the buffer that will be used to store the output tokens.

The CORE\_ WhisperModel also has a Constructor field that is a constructor function for the WhisperModel class.

The CORE\_ WhisperModel also has a destructor field that is a destructor function for the WhisperModel class.


```cpp
struct whisper_kv_cache {
    struct ggml_tensor * k;
    struct ggml_tensor * v;

    struct ggml_context * ctx;

    // buf points to the memory allocated for both ggml_tensor 'k' and 'v' (see kv_cache_init)
    std::vector<uint8_t> buf;

    int n; // number of tokens currently in the cache
};

struct whisper_model {
    e_model type = MODEL_UNKNOWN;

    whisper_hparams hparams;
    whisper_filters filters;

    // encoder.positional_embedding
    struct ggml_tensor * e_pe;

    // encoder.conv1
    struct ggml_tensor * e_conv_1_w;
    struct ggml_tensor * e_conv_1_b;

    // encoder.conv2
    struct ggml_tensor * e_conv_2_w;
    struct ggml_tensor * e_conv_2_b;

    // encoder.ln_post
    struct ggml_tensor * e_ln_w;
    struct ggml_tensor * e_ln_b;

    // decoder.positional_embedding
    struct ggml_tensor * d_pe;

    // decoder.token_embedding
    struct ggml_tensor * d_te;

    // decoder.ln
    struct ggml_tensor * d_ln_w;
    struct ggml_tensor * d_ln_b;

    std::vector<whisper_layer_encoder> layers_encoder;
    std::vector<whisper_layer_decoder> layers_decoder;

    // context
    struct ggml_context * ctx;

    // the model memory buffer is read-only and can be shared between processors
    std::vector<uint8_t> * buf;

    // tensors
    int n_loaded;
    std::map<std::string, struct ggml_tensor *> tensors;
};

```

这是一个名为 WhisperDecoder 和 WhisperSequence 的结构体，它们用于实现一个循环生命的语音识别任务。

WhisperDecoder 负责处理语音信号中的文本，并生成对应的词汇表。而 WhisperSequence 则记录了每次循环生成的词汇，并包含了当前循环生成的词汇表。

具体来说，这个代码定义了一个 WhisperSequence 结构体，它包含了当前循环生成的词汇表，以及一个表示当前循环状态的变量 failed，一个表示当前循环是否完成变量 completed，一个表示是否已经采样的前一个时刻的timestamp变量 has\_ts，以及一个用于存储新词概率、逻辑和逻辑概率的 vector<float> 变量 probs，一个用于存储当前循环生成的词汇的 vector<float> 变量 logits，一个用于存储当前循环生成的词汇的 vector<float> 变量 logprobs。

同时，这个代码还定义了一个 WhisperDecoder 结构体，它包含一个用于存储当前循环生成的词汇表的 vector<WhisperDecoderNode> 变量，以及一个表示当前循环状态的变量 failed，一个表示当前循环是否完成变量 completed，一个表示是否已经采样的前一个时刻的timestamp变量 has\_ts，一个用于存储新词概率、逻辑和逻辑概率的 vector<float> 变量 probs，一个用于存储当前循环生成的词汇的 vector<float> 变量 logits，一个用于存储当前循环生成的词汇的 vector<float> 变量 logprobs。


```cpp
struct whisper_sequence {
    std::vector<whisper_token_data> tokens;

    // the accumulated transcription in the current iteration (used to truncate the tokens array)
    int result_len;

    double sum_logprobs_all; // the sum of the log probabilities of the tokens
    double sum_logprobs;     // the sum of the log probabilities of the tokens (first result_len tokens)
    double avg_logprobs;     // the average log probability of the tokens
    double entropy;          // the entropy of the tokens
    double score;            // likelihood rank score
};

// TAGS: WHISPER_DECODER_INIT
struct whisper_decoder {
    // each decoder keeps its own KV-cache
    whisper_kv_cache kv_self;

    // the currently generated sequence of tokens
    whisper_sequence sequence;

    int seek_delta; // the window shift found so far based on the decoded timestamp tokens

    bool failed;    // has the current segment failed to decode?
    bool completed; // has the decoder completed the current segment?
    bool has_ts;    // have we already sampled a non-beg timestamp token for the current segment?

    // new token probs, logits and logprobs after the last whisper_decode (1-dimensional array: [n_vocab])
    std::vector<float> probs;
    std::vector<float> logits;
    std::vector<float> logprobs;

    std::vector<whisper_token> tokens_tmp; // used for whisper_decode calls
};

```

这段代码定义了一个名为“whisper_pair”的结构体，用于替代标准库中的“std::pair”。这个结构体有两个成员变量：一个名为“first”的类型为A的成员变量，一个名为“second”的类型为B的成员变量。

whisper_pair的结构体定义了两个构造函数：一个接受两个A类型的参数的构造函数，和一个不接收任何参数的构造函数。这些构造函数在函数内部使用成员变量初始化默认值。

接下来，定义了一个名为“kv_buf”的结构体，用于存储一个基于键值对的缓冲区。这个结构体有两个成员变量：一个名为“k”的类型为std::vector<uint8_t>的成员变量，一个名为“v”的类型为std::vector<uint8_t>的成员变量。

最后，没有定义任何函数，导入了来自std::pair的成员函数，以便能够使用现有的实现。


```cpp
// replace std::pair by using customized pair struct (reason: std::pair is very slow)
template<typename A, typename B>
struct whisper_pair {
    A first;
    B second;

    // Define a constructor that takes two arguments.
    whisper_pair(const A& a, const B& b) : first(a), second(b) {}
    // Define a constructor that takes no argument.
    whisper_pair() : first(A()), second(B()) {}
};

// beam-search helpers
struct kv_buf {
    std::vector<uint8_t> k;
    std::vector<uint8_t> v;
};

```

这段代码定义了一个名为 "whisper_allocr" 的 struct，该 struct 包含一个指向 "ggml_allocr" 类型对象的指针和两个向量类型的成员变量："meta" 和 "data"。函数 "whisper_allocr_size" 计算给定 "whisper_allocr" 结构物的内存使用情况，函数 "whisper_allocr_graph_init" 在 "whisper_allocr" 结构物的初始化过程中，根据给定的 "get_graph" 函数返回的函数进行 graph 的初始化。在该 graph 初始化过程中，将计算得到的最小内存分配给 "meta" 和 "data" 成员变量。


```cpp
// ggml_allocr wrapper for whisper usage
struct whisper_allocr {
    ggml_allocr * alloc = nullptr;

    std::vector<uint8_t> meta;
    std::vector<uint8_t> data;
};

static size_t whisper_allocr_size(struct whisper_allocr & allocr) {
    return allocr.meta.size() + allocr.data.size();
}

// measure the memory usage of a graph and prepare the allocr's internal data buffer
static void whisper_allocr_graph_init(struct whisper_allocr & allocr, std::function<struct ggml_cgraph *()> && get_graph) {
    const int tensor_alignment = 32;

    auto & alloc = allocr.alloc;
    auto & meta  = allocr.meta;
    auto & data  = allocr.data;

    meta.resize(ggml_tensor_overhead()*WHISPER_MAX_NODES + ggml_graph_overhead());

    alloc = ggml_allocr_new_measure(tensor_alignment);

    const size_t alloc_size = ggml_allocr_alloc_graph(alloc, get_graph()) + tensor_alignment;

    ggml_allocr_free(alloc);

    data.resize(alloc_size);

    alloc = ggml_allocr_new(data.data(), data.size(), tensor_alignment);
}

```

0
========

123
=======


```cpp
static void whisper_allocr_free(struct whisper_allocr & allocr) {
    if (allocr.alloc) {
        ggml_allocr_free(allocr.alloc);
        allocr.alloc = nullptr;
    }
}

struct whisper_state {
    int64_t t_sample_us = 0;
    int64_t t_encode_us = 0;
    int64_t t_decode_us = 0;
    int64_t t_prompt_us = 0;
    int64_t t_mel_us = 0;

    int32_t n_sample = 0; // number of tokens sampled
    int32_t n_encode = 0; // number of encoder calls
    int32_t n_decode = 0; // number of decoder calls with n_tokens == 1 (text-generation)
    int32_t n_prompt = 0; // number of decoder calls with n_tokens >  1 (prompt encoding)
    int32_t n_fail_p = 0; // number of logprob threshold failures
    int32_t n_fail_h = 0; // number of entropy threshold failures

    // cross-attention KV cache for the decoders
    // shared between all decoders
    whisper_kv_cache kv_cross;
    whisper_mel mel;

    whisper_decoder decoders[WHISPER_MAX_DECODERS] = {};

    // buffer for swapping KV caches between decoders during beam-search
    std::vector<kv_buf> kv_swap_bufs;

    // reusable buffer for `struct ggml_graph_plan.work_data`
    std::vector<uint8_t> work_buffer;

    // ggml-alloc:
    // - stores meta info about the intermediate tensors into the `meta` buffers
    // - stores the actual tensor data into the `data` buffers
    whisper_allocr alloc_conv;
    whisper_allocr alloc_encode;
    whisper_allocr alloc_cross;
    whisper_allocr alloc_decode;

    // result of the encoder
    struct ggml_tensor * embd_conv = nullptr;
    struct ggml_tensor * embd_enc  = nullptr;

    // decode output (2-dimensional array: [n_tokens][n_vocab])
    std::vector<float> logits;

    std::vector<whisper_segment> result_all;
    std::vector<whisper_token>   prompt_past;

    // work container used to avoid memory allocations
    std::vector<whisper_pair<double, whisper_vocab::id>> logits_id;

    mutable std::mt19937 rng; // used for sampling at t > 0.0

    int lang_id = 0; // english by default

    std::string path_model; // populated by whisper_init_from_file()
```

这段代码是一个C++程序，主要作用是定义了三个条件分支，用于决定在不同的输入条件下是否使用不同的实现，以提高程序的性能。

具体来说，代码分为以下几个部分：

1. 定义了三个条件分支：

- `#ifdef WHISPER_USE_COREML`
- `#ifdef GGML_USE_METAL`
- `#ifdef WHISPER_USE_OPENVINO`

每个分支下面都没有代码块，所以这三个分支只有在需要的时候才会被调用。

2. 在每个分支下面，定义了一个指针变量，用于保存和该分支相关的上下文。这些上下文用于计算性能指标。

- `whisper_coreml_context * ctx_coreml = nullptr;`
- `ggml_metal_context * ctx_metal = nullptr;`
- `whisper_openvino_context * ctx_openvino = nullptr;`

3. 在主函数里面，初始化了每个条件分支的上下文，并将一些常量存储在全局变量中，以便在程序运行时进行初始化。

- `t_beg = 0;`
- `t_last = 0;`
- `whisper_token tid_last;`
- `std::vector<float> energy;`

4. 在主函数中，根据每个输入是否使用了其中一个实现，对程序中的实现进行了改动，以提高程序的性能。这些改动包括：

- 如果使用了`WHISPER_USE_COREML`，则执行以下代码：

 - 定义了一个`whisper_coreml_context * ctx_coreml`变量，用于保存与该实现相关的上下文。
- 如果使用了`GGML_USE_METAL`，则执行以下代码：

 - 定义了一个`ggml_metal_context * ctx_metal`变量，用于保存与该实现相关的上下文。
- 如果使用了`WHISPER_USE_OPENVINO`，则执行以下代码：

 - 定义了一个`whisper_openvino_context * ctx_openvino`变量，用于保存与该实现相关的上下文。

5. 在主函数中，使用条件分支来决定如何执行程序中的每个实现。如果使用了`WHISPER_USE_COREML`，则执行以下代码：

 - 计算和第一种实现对应的`t_beg`和`t_last`值。
- 如果使用了`GGML_USE_METAL`，则执行以下代码：

 - 计算和第二种实现对应的`t_beg`和`t_last`值。
- 如果使用了`WHISPER_USE_OPENVINO`，则执行以下代码：

 - 计算和第三种实现对应的`t_beg`和`t_last`值。


```cpp
#ifdef WHISPER_USE_COREML
    whisper_coreml_context * ctx_coreml = nullptr;
#endif

#ifdef GGML_USE_METAL
    ggml_metal_context * ctx_metal = nullptr;
#endif

#ifdef WHISPER_USE_OPENVINO
    whisper_openvino_context * ctx_openvino = nullptr;
#endif

    // [EXPERIMENTAL] token-level timestamps data
    int64_t t_beg = 0;
    int64_t t_last = 0;
    whisper_token tid_last;
    std::vector<float> energy; // PCM signal energy

    // [EXPERIMENTAL] speed-up techniques
    int32_t exp_n_audio_ctx = 0; // 0 - use default
};

```



这是一个用C++编写的Whisper库的定义。该库包含一个Whisper状态结构体，用于保存模型、词汇表和状态的指针。

Whisper是一个用于机器学习的开源库，主要用于点对点(点对点)分类问题的训练和部署。在本项目中，Whisper库被用于实现一个简单的分类问题，其中用户需要输入标签，并通过Whisper库训练模型，然后使用模型对新的数据进行预测。

该库包含一个默认的日志函数，用于将用户提供的标签打印到控制台。此外，还包含一个Whisper模型和一个Whisper词汇表。模型是一个用C++编写的内置类型，用于在训练和预测时保存模型参数。词汇表是一个由字符串键值对组成的容器，用于存储数据和标签。状态变量是一个指向Whisper状态的指针，用于在训练和预测时保存和更新模型参数。

该库的源代码没有提供，因此无法提供更多有关该库如何工作的详细信息。


```cpp
struct whisper_context {
    int64_t t_load_us  = 0;
    int64_t t_start_us = 0;

    ggml_type wtype = ggml_type::GGML_TYPE_F16; // weight type (FP32 / FP16 / QX)
    ggml_type itype = ggml_type::GGML_TYPE_F16; // intermediate type (FP32 or FP16)

    whisper_model model;
    whisper_vocab vocab;
    whisper_state * state = nullptr;

    std::string path_model; // populated by whisper_init_from_file()
};

static void whisper_default_log(const char * text) {
    fprintf(stderr, "%s", text);
}

```

这段代码定义了一个名为whisper_log的静态变量，它的值为whisper_default_log。然后，代码中使用if语句判断当前系统是否支持__GNUC__，如果是，则执行下面的代码块，如果不是，则执行下面的代码块。

if(__GNUC__) {
   if(__MINGW32__) {
       __attribute__((gnu_format(printf, 1, 2)))
   } else {
       __attribute__((format(printf, 1, 2)));
   }
}

代码else {
   printf("This code is not written for %s\n",__GNUC__?"__GNUC__"});
}

接下来，代码定义了一个名为log的函数，该函数接受一个const char *的格式化字符串以及多个参数。if语句检查是否已定义whisper_log变量，如果已定义，则执行以下操作：

if (whisper_log) {
   static void log_callback(const char *format, ...) {
       char buf[1024];
       va_list args;
       va_start(args, format);
       vsnprintf(buf, sizeof(buf), format, args);
       whisper_log(buf);
   }
   log_callback(fmt, args);
} else {
   printf("This code requires whisper_log to be defined.\n");
}

最后，代码中使用一个静态变量whisper_log，它的值为从whisper_default_log初始化的默认日志输出。


```cpp
static whisper_log_callback whisper_log = whisper_default_log;

#ifdef __GNUC__
#ifdef __MINGW32__
__attribute__((gnu_format(printf, 1, 2)))
#else
__attribute__((format(printf, 1, 2)))
#endif
#endif
static void log(const char * fmt, ...) {
    if (!whisper_log) return;
    char buf[1024];
    va_list args;
    va_start(args, fmt);
    vsnprintf(buf, sizeof(buf), fmt, args);
    whisper_log(buf);
}

```



这段代码定义了一个名为 `read_safe` 的函数，属于 `whisper_model_loader` 类的成员函数。

该函数的作用是读取一个 `T` 类型的数据，将其存储在 `dest` 变量中。首先，通过调用 `loader->read` 函数来读取数据，然后将其存储在 `dest` 变量中。然后通过 `BYTESWAP_VALUE` 函数，保证 `dest` 变量与读取的数据相同。

接下来，定义了一个名为 `kv_cache_init` 的函数，属于 `whisper_kv_cache` 类的成员函数。

该函数的作用是初始化一个 `whisper_hparams` 类型的 `kv_cache` 对象。首先根据 `whisper_hparams` 初始化 `kv_cache` 对象。然后根据 `kv_cache_init` 函数计算出 `kv_cache` 需要存储的字节数，即 `n_text_layer` 和 `n_ctx` 的值，并将计算出的字节数存储到 `cache.buf.resize` 函数中。接着，定义了一个 `ggml_init_params` 类型的初始化参数，用于初始化 `ggml_new_tensor_1d` 函数。然后，将 `ggml_init` 函数作为参数传递给 `kv_cache_init` 函数，并将读取数据、释放内存的上下文作为 `params` 的一部分。最后，如果初始化失败，将记录到日志中。

该代码的作用是初始化一个 `whisper_kv_cache` 对象，用于存储模型加载器读取的数据。


```cpp
template<typename T>
static void read_safe(whisper_model_loader * loader, T & dest) {
    loader->read(loader->context, &dest, sizeof(T));
    BYTESWAP_VALUE(dest);
}

static bool kv_cache_init(
        const struct whisper_hparams & hparams,
             struct whisper_kv_cache & cache,
                           ggml_type   wtype,
                                 int   n_ctx) {
    const int64_t n_text_state = hparams.n_text_state;
    const int64_t n_text_layer = hparams.n_text_layer;

    const int64_t n_mem      = n_text_layer*n_ctx;
    const int64_t n_elements = n_text_state*n_mem;

    const size_t mem_bytes = 2*(ggml_type_size(wtype)*n_elements + ggml_tensor_overhead());

    cache.buf.resize(mem_bytes);

    struct ggml_init_params params = {
        /*.mem_size   =*/ cache.buf.size(),
        /*.mem_buffer =*/ cache.buf.data(),
        /*.no_alloc   =*/ false,
    };

    cache.ctx = ggml_init(params);

    if (!cache.ctx) {
        log("%s: failed to allocate memory for kv cache\n", __func__);
        return false;
    }

    cache.k = ggml_new_tensor_1d(cache.ctx, wtype, n_elements);
    cache.v = ggml_new_tensor_1d(cache.ctx, wtype, n_elements);

    return true;
}

```

这段代码是一个静态函数，名为 `kv_cache_reinit`，属于 `whisper_kv_cache` 结构体的成员函数。它的作用是重新初始化KV缓存。

具体来说，代码首先检查缓存结构体中 `ctx` 和 `k`、`v` 成员是否已经被初始化，如果没有，就继续下一步。然后，判断KV类型是否和缓存类型相同，如果是，就说明缓存已经创建好，可以继续执行下面的操作。

接着，代码计算缓存区大小是否符合要求，如果不符合，就需要重新分配内存。然后，代码分配内存并初始化 `k` 和 `v` 成员。最后，代码返回一个布尔值，表示是否成功初始化缓存。

如果初始化失败，函数返回 `false`，否则返回 `true`。


```cpp
static bool kv_cache_reinit(struct whisper_kv_cache & cache) {
    WHISPER_ASSERT(cache.ctx);

    const int n_elements = ggml_nelements(cache.k);
    WHISPER_ASSERT(n_elements == ggml_nelements(cache.v));

    const ggml_type wtype = cache.k->type;
    WHISPER_ASSERT(wtype == cache.v->type);

    WHISPER_ASSERT(cache.buf.size() >= 2*n_elements*ggml_type_sizef(wtype));

    struct ggml_init_params params = {
        /*.mem_size   =*/ cache.buf.size(),
        /*.mem_buffer =*/ cache.buf.data(),
        /*.no_alloc   =*/ false,
    };

    cache.ctx = ggml_init(params);

    if (!cache.ctx) {
        log("%s: failed to allocate memory for kv cache\n", __func__);
        return false;
    }

    cache.k = ggml_new_tensor_1d(cache.ctx, wtype, n_elements);
    cache.v = ggml_new_tensor_1d(cache.ctx, wtype, n_elements);

    return true;
}

```

这段代码是一个静态函数，名为 `kv_cache_free`，属于 `whisper_kv_cache` 结构体。它的作用是释放 KV 缓存中的数据，前提条件是缓存对象 `cache` 还持有有效的上下文（`ctx`）。

具体来说，如果 `cache.ctx` 还持有有效的上下文，函数会尝试用 `ggml_free` 函数释放上下文，并将其赋值给 `cache.ctx`。如果释放成功，将缓存对象的上下文设置为 `null`，使得缓存对象不再引用任何内存。这样，当需要再次加载模型时，不再持有内存，从而避免了内存泄漏。


```cpp
static void kv_cache_free(struct whisper_kv_cache & cache) {
    if (cache.ctx) {
        ggml_free(cache.ctx);
        cache.ctx = nullptr;
    }
}

// load the model from a ggml file
//
// file format:
//
//   - hparams
//   - pre-computed mel filters
//   - vocab
//   - weights
```

This function appears to validate the model file for a deep learning model's tensors. It does this by reading the tensor data from the file and checking it against the model's expected data format.

It takes as input the loaded tensor data, which is passed to the function. The function then checks the data against the expected data format, which is defined by the size and type of each tensor in the model. If the data does not match the expected format, the function logs an error and returns false. If the data is valid, the function returns true.

The function also calculates the total size of the loaded data in the model and logs this in its log.

It is important to note that this function should be noted as a "model checker" or "tensors validator" rather than a "data converter" or "exporter".


```cpp
//
// see the convert-pt-to-ggml.py script for details
//
static bool whisper_model_load(struct whisper_model_loader * loader, whisper_context & wctx) {
    log("%s: loading model\n", __func__);

    const int64_t t_start_us = ggml_time_us();

    wctx.t_start_us = t_start_us;

    auto & model = wctx.model;
    auto & vocab = wctx.vocab;

    // verify magic
    {
        uint32_t magic;
        read_safe(loader, magic);
        if (magic != GGML_FILE_MAGIC) {
            log("%s: invalid model data (bad magic)\n", __func__);
            return false;
        }
    }

    //load hparams
    {
        auto & hparams = model.hparams;

        read_safe(loader, hparams.n_vocab);
        read_safe(loader, hparams.n_audio_ctx);
        read_safe(loader, hparams.n_audio_state);
        read_safe(loader, hparams.n_audio_head);
        read_safe(loader, hparams.n_audio_layer);
        read_safe(loader, hparams.n_text_ctx);
        read_safe(loader, hparams.n_text_state);
        read_safe(loader, hparams.n_text_head);
        read_safe(loader, hparams.n_text_layer);
        read_safe(loader, hparams.n_mels);
        read_safe(loader, hparams.ftype);

        assert(hparams.n_text_state == hparams.n_audio_state);

        if (hparams.n_audio_layer == 4) {
            model.type = e_model::MODEL_TINY;
        }

        if (hparams.n_audio_layer == 6) {
            model.type = e_model::MODEL_BASE;
        }

        if (hparams.n_audio_layer == 12) {
            model.type = e_model::MODEL_SMALL;
        }

        if (hparams.n_audio_layer == 24) {
            model.type = e_model::MODEL_MEDIUM;
        }

        if (hparams.n_audio_layer == 32) {
            model.type = e_model::MODEL_LARGE;
        }

        const int32_t qntvr = hparams.ftype / GGML_QNT_VERSION_FACTOR;

        hparams.ftype %= GGML_QNT_VERSION_FACTOR;

        // for the big tensors, we have the option to store the data in 16-bit floats or quantized
        // in order to save memory and also to speed up the computation
        wctx.wtype = ggml_ftype_to_ggml_type((ggml_ftype) (model.hparams.ftype));
        if (wctx.wtype == GGML_TYPE_COUNT) {
            log("%s: invalid model (bad ftype value %d)\n", __func__, model.hparams.ftype);
            return false;
        }

        const size_t scale = model.hparams.ftype ? 1 : 2;

        log("%s: n_vocab       = %d\n", __func__, hparams.n_vocab);
        log("%s: n_audio_ctx   = %d\n", __func__, hparams.n_audio_ctx);
        log("%s: n_audio_state = %d\n", __func__, hparams.n_audio_state);
        log("%s: n_audio_head  = %d\n", __func__, hparams.n_audio_head);
        log("%s: n_audio_layer = %d\n", __func__, hparams.n_audio_layer);
        log("%s: n_text_ctx    = %d\n", __func__, hparams.n_text_ctx);
        log("%s: n_text_state  = %d\n", __func__, hparams.n_text_state);
        log("%s: n_text_head   = %d\n", __func__, hparams.n_text_head);
        log("%s: n_text_layer  = %d\n", __func__, hparams.n_text_layer);
        log("%s: n_mels        = %d\n", __func__, hparams.n_mels);
        log("%s: ftype         = %d\n", __func__, model.hparams.ftype);
        log("%s: qntvr         = %d\n", __func__, qntvr);
        log("%s: type          = %d\n", __func__, model.type);

        // print memory requirements
        {
            // TODO
            //log("%s: mem required  = %7.2f MB (+ %7.2f MB per decoder)\n", __func__,
            //        mem_required / 1024.0 / 1024.0, mem_required_decoder / 1024.0 / 1024.0);
        }

        // initialize all memory buffers
        // always have at least one decoder

        wctx.model.buf = new std::vector<uint8_t>();
        wctx.model.buf->resize(scale*MEM_REQ_MODEL.at(wctx.wtype).at(model.type));

        // we skip initialization of the state until it is needed
        // because it might be that state will always be provided externally.
    }

    // load mel filters
    {
        auto & filters = wctx.model.filters;

        read_safe(loader, filters.n_mel);
        read_safe(loader, filters.n_fft);

        filters.data.resize(filters.n_mel * filters.n_fft);
        loader->read(loader->context, filters.data.data(), filters.data.size() * sizeof(float));
        BYTESWAP_FILTERS(filters);
    }

    // load vocab
    {
        int32_t n_vocab = 0;
        read_safe(loader, n_vocab);

        //if (n_vocab != model.hparams.n_vocab) {
        //    log("%s: invalid model file '%s' (bad vocab size %d != %d)\n",
        //            __func__, fname.c_str(), n_vocab, model.hparams.n_vocab);
        //    return false;
        //}

        std::string word;
        std::vector<char> tmp;

        tmp.reserve(128);

        for (int i = 0; i < n_vocab; i++) {
            uint32_t len;
            read_safe(loader, len);

            if (len > 0) {
                tmp.resize(len);
                loader->read(loader->context, &tmp[0], tmp.size()); // read to buffer
                word.assign(&tmp[0], tmp.size());
            } else {
                // seems like we have an empty-string token in multi-language models (i = 50256)
                //log("%s: warning: empty-string token in vocab, i = %d\n", __func__, i);
                word = "";
            }

            vocab.token_to_id[word] = i;
            vocab.id_to_token[i] = word;

            //printf("%s: vocab[%d] = '%s'\n", __func__, i, word.c_str());
        }

        vocab.n_vocab = model.hparams.n_vocab;
        if (vocab.is_multilingual()) {
            vocab.token_eot++;
            vocab.token_sot++;
            vocab.token_translate++;
            vocab.token_transcribe++;
            vocab.token_solm++;
            vocab.token_prev++;
            vocab.token_nosp++;
            vocab.token_not++;
            vocab.token_beg++;
        }

        if (n_vocab < model.hparams.n_vocab) {
            log("%s: adding %d extra tokens\n", __func__, model.hparams.n_vocab - n_vocab);
            for (int i = n_vocab; i < model.hparams.n_vocab; i++) {
                if (i > vocab.token_beg) {
                    word = "[_TT_" + std::to_string(i - vocab.token_beg) + "]";
                } else if (i == vocab.token_eot) {
                    word = "[_EOT_]";
                } else if (i == vocab.token_sot) {
                    word = "[_SOT_]";
                } else if (i == vocab.token_solm) {
                    word = "[_SOLM_]";
                } else if (i == vocab.token_prev) {
                    word = "[_PREV_]";
                } else if (i == vocab.token_nosp) {
                    word = "[_NOSP_]";
                } else if (i == vocab.token_not) {
                    word = "[_NOT_]";
                } else if (i == vocab.token_beg) {
                    word = "[_BEG_]";
                } else {
                    word = "[_extra_token_" + std::to_string(i) + "]";
                }
                vocab.token_to_id[word] = i;
                vocab.id_to_token[i] = word;
            }
        }
    }

    size_t ctx_size = 0;

    const ggml_type wtype = wctx.wtype;
    const ggml_type vtype = wctx.wtype == GGML_TYPE_F32 ? GGML_TYPE_F32 : GGML_TYPE_F16; // conv type

    {
        const auto & hparams = model.hparams;

        const int n_vocab = hparams.n_vocab;

        const int n_audio_ctx   = hparams.n_audio_ctx;
        const int n_audio_state = hparams.n_audio_state;
        const int n_audio_layer = hparams.n_audio_layer;

        const int n_text_ctx   = hparams.n_text_ctx;
        const int n_text_state = hparams.n_text_state;
        const int n_text_layer = hparams.n_text_layer;

        const int n_mels = hparams.n_mels;

        // encoder
        {
            ctx_size += n_audio_ctx*n_audio_state*ggml_type_sizef(GGML_TYPE_F32); // e_pe;

            ctx_size += 3*n_mels*n_audio_state*ggml_type_sizef(vtype);         // e_conv_1_w
            ctx_size +=          n_audio_state*ggml_type_sizef(GGML_TYPE_F32); // e_conv_1_b

            ctx_size += 3*n_audio_state*n_audio_state*ggml_type_sizef(vtype);         // e_conv_2_w
            ctx_size +=                 n_audio_state*ggml_type_sizef(GGML_TYPE_F32); // e_conv_2_b

            ctx_size += n_audio_state*ggml_type_sizef(GGML_TYPE_F32); // e_ln_w;
            ctx_size += n_audio_state*ggml_type_sizef(GGML_TYPE_F32); // e_ln_b;
        }

        // decoder
        {
            ctx_size += n_text_ctx*n_text_state*ggml_type_sizef(GGML_TYPE_F32); // d_pe;

            ctx_size += n_vocab*n_text_state*ggml_type_sizef(wtype); // d_te;

            ctx_size += n_text_state*ggml_type_sizef(GGML_TYPE_F32); // d_ln_w;
            ctx_size += n_text_state*ggml_type_sizef(GGML_TYPE_F32); // d_ln_b;
        }

        // encoder layers
        {
            ctx_size += n_audio_layer*(n_audio_state*ggml_type_sizef(GGML_TYPE_F32)); // mlp_ln_w
            ctx_size += n_audio_layer*(n_audio_state*ggml_type_sizef(GGML_TYPE_F32)); // mlp_ln_b

            ctx_size += n_audio_layer*(4*n_audio_state*n_audio_state*ggml_type_sizef(wtype));         // mlp_0_w
            ctx_size += n_audio_layer*(              4*n_audio_state*ggml_type_sizef(GGML_TYPE_F32)); // mlp_0_b

            ctx_size += n_audio_layer*(4*n_audio_state*n_audio_state*ggml_type_sizef(wtype));         // mlp_1_w
            ctx_size += n_audio_layer*(                n_audio_state*ggml_type_sizef(GGML_TYPE_F32)); // mlp_1_b

            ctx_size += n_audio_layer*(n_audio_state*ggml_type_sizef(GGML_TYPE_F32)); // attn_ln_0_w
            ctx_size += n_audio_layer*(n_audio_state*ggml_type_sizef(GGML_TYPE_F32)); // attn_ln_0_b

            ctx_size += n_audio_layer*(n_audio_state*n_audio_state*ggml_type_sizef(wtype));         // attn_q_w
            ctx_size += n_audio_layer*(              n_audio_state*ggml_type_sizef(GGML_TYPE_F32)); // attn_q_b

            ctx_size += n_audio_layer*(n_audio_state*n_audio_state*ggml_type_sizef(wtype)); // attn_k_w

            ctx_size += n_audio_layer*(n_audio_state*n_audio_state*ggml_type_sizef(wtype));         // attn_v_w
            ctx_size += n_audio_layer*(              n_audio_state*ggml_type_sizef(GGML_TYPE_F32)); // attn_v_b

            ctx_size += n_audio_layer*(n_audio_state*n_audio_state*ggml_type_sizef(wtype));         // attn_ln_1_w
            ctx_size += n_audio_layer*(              n_audio_state*ggml_type_sizef(GGML_TYPE_F32)); // attn_ln_1_b
        }

        // decoder layers
        {
            ctx_size += n_text_layer*(n_text_state*ggml_type_sizef(GGML_TYPE_F32)); // mlp_ln_w
            ctx_size += n_text_layer*(n_text_state*ggml_type_sizef(GGML_TYPE_F32)); // mlp_ln_b

            ctx_size += n_text_layer*(4*n_text_state*n_text_state*ggml_type_sizef(wtype));         // mlp_0_w
            ctx_size += n_text_layer*(             4*n_text_state*ggml_type_sizef(GGML_TYPE_F32)); // mlp_0_b

            ctx_size += n_text_layer*(4*n_text_state*n_text_state*ggml_type_sizef(wtype));         // mlp_1_w
            ctx_size += n_text_layer*(               n_text_state*ggml_type_sizef(GGML_TYPE_F32)); // mlp_1_b

            ctx_size += n_text_layer*(n_text_state*ggml_type_sizef(GGML_TYPE_F32)); // attn_ln_0_w
            ctx_size += n_text_layer*(n_text_state*ggml_type_sizef(GGML_TYPE_F32)); // attn_ln_0_b

            ctx_size += n_text_layer*(n_text_state*n_text_state*ggml_type_sizef(wtype));         // attn_q_w
            ctx_size += n_text_layer*(             n_text_state*ggml_type_sizef(GGML_TYPE_F32)); // attn_q_b

            ctx_size += n_text_layer*(n_text_state*n_text_state*ggml_type_sizef(wtype)); // attn_k_w

            ctx_size += n_text_layer*(n_text_state*n_text_state*ggml_type_sizef(wtype));         // attn_v_w
            ctx_size += n_text_layer*(             n_text_state*ggml_type_sizef(GGML_TYPE_F32)); // attn_v_b

            ctx_size += n_text_layer*(n_text_state*n_text_state*ggml_type_sizef(wtype));         // attn_ln_1_w
            ctx_size += n_text_layer*(             n_text_state*ggml_type_sizef(GGML_TYPE_F32)); // attn_ln_1_b
                                                                                                //
            ctx_size += n_text_layer*(n_text_state*ggml_type_sizef(GGML_TYPE_F32)); // cross_attn_ln_0_w
            ctx_size += n_text_layer*(n_text_state*ggml_type_sizef(GGML_TYPE_F32)); // cross_attn_ln_0_b

            ctx_size += n_text_layer*(n_text_state*n_text_state*ggml_type_sizef(wtype));         // cross_attn_q_w
            ctx_size += n_text_layer*(             n_text_state*ggml_type_sizef(GGML_TYPE_F32)); // cross_attn_q_b

            ctx_size += n_text_layer*(n_text_state*n_text_state*ggml_type_sizef(wtype)); // cross_attn_k_w

            ctx_size += n_text_layer*(n_text_state*n_text_state*ggml_type_sizef(wtype));         // cross_attn_v_w
            ctx_size += n_text_layer*(             n_text_state*ggml_type_sizef(GGML_TYPE_F32)); // cross_attn_v_b

            ctx_size += n_text_layer*(n_text_state*n_text_state*ggml_type_sizef(wtype));         // cross_attn_ln_1_w
            ctx_size += n_text_layer*(             n_text_state*ggml_type_sizef(GGML_TYPE_F32)); // cross_attn_ln_1_b
        }

        ctx_size += (15 + 15*n_audio_layer + 24*n_text_layer)*512; // object overhead

        log("%s: model ctx     = %7.2f MB\n", __func__, ctx_size/(1024.0*1024.0));
    }

    // create the ggml context
    {
        struct ggml_init_params params = {
            /*.mem_size   =*/ wctx.model.buf->size(),
            /*.mem_buffer =*/ wctx.model.buf->data(),
            /*.no_alloc   =*/ false,
        };

        model.ctx = ggml_init(params);
        if (!model.ctx) {
            log("%s: ggml_init() failed\n", __func__);
            return false;
        }
    }

    // prepare memory for the weights
    {
        auto & ctx = model.ctx;

        const auto & hparams = model.hparams;

        const int n_vocab = hparams.n_vocab;

        const int n_audio_ctx   = hparams.n_audio_ctx;
        const int n_audio_state = hparams.n_audio_state;
        const int n_audio_layer = hparams.n_audio_layer;

        const int n_text_ctx   = hparams.n_text_ctx;
        const int n_text_state = hparams.n_text_state;
        const int n_text_layer = hparams.n_text_layer;

        const int n_mels = hparams.n_mels;

        model.layers_encoder.resize(n_audio_layer);
        model.layers_decoder.resize(n_text_layer);

        // encoder
        {
            model.e_pe       = ggml_new_tensor_2d(ctx, GGML_TYPE_F32, n_audio_state, n_audio_ctx);

            model.e_conv_1_w = ggml_new_tensor_3d(ctx, vtype,         3, n_mels, n_audio_state);
            model.e_conv_1_b = ggml_new_tensor_2d(ctx, GGML_TYPE_F32, 1, n_audio_state);

            model.e_conv_2_w = ggml_new_tensor_3d(ctx, vtype,         3, n_audio_state, n_audio_state);
            model.e_conv_2_b = ggml_new_tensor_2d(ctx, GGML_TYPE_F32, 1, n_audio_state);

            model.e_ln_w     = ggml_new_tensor_1d(ctx, GGML_TYPE_F32, n_audio_state);
            model.e_ln_b     = ggml_new_tensor_1d(ctx, GGML_TYPE_F32, n_audio_state);

            // map by name
            model.tensors["encoder.positional_embedding"] = model.e_pe;

            model.tensors["encoder.conv1.weight"]         = model.e_conv_1_w;
            model.tensors["encoder.conv1.bias"]           = model.e_conv_1_b;

            model.tensors["encoder.conv2.weight"]         = model.e_conv_2_w;
            model.tensors["encoder.conv2.bias"]           = model.e_conv_2_b;

            model.tensors["encoder.ln_post.weight"]       = model.e_ln_w;
            model.tensors["encoder.ln_post.bias"]         = model.e_ln_b;

            for (int i = 0; i < n_audio_layer; ++i) {
                auto & layer = model.layers_encoder[i];

                layer.mlp_ln_w    = ggml_new_tensor_1d(ctx, GGML_TYPE_F32,   n_audio_state);
                layer.mlp_ln_b    = ggml_new_tensor_1d(ctx, GGML_TYPE_F32,   n_audio_state);

                layer.mlp_0_w     = ggml_new_tensor_2d(ctx, wtype,           n_audio_state, 4*n_audio_state);
                layer.mlp_0_b     = ggml_new_tensor_1d(ctx, GGML_TYPE_F32, 4*n_audio_state);

                layer.mlp_1_w     = ggml_new_tensor_2d(ctx, wtype,         4*n_audio_state, n_audio_state);
                layer.mlp_1_b     = ggml_new_tensor_1d(ctx, GGML_TYPE_F32,   n_audio_state);

                layer.attn_ln_0_w = ggml_new_tensor_1d(ctx, GGML_TYPE_F32,   n_audio_state);
                layer.attn_ln_0_b = ggml_new_tensor_1d(ctx, GGML_TYPE_F32,   n_audio_state);

                layer.attn_q_w    = ggml_new_tensor_2d(ctx, wtype,           n_audio_state, n_audio_state);
                layer.attn_q_b    = ggml_new_tensor_1d(ctx, GGML_TYPE_F32,   n_audio_state);

                layer.attn_k_w    = ggml_new_tensor_2d(ctx, wtype,           n_audio_state, n_audio_state);

                layer.attn_v_w    = ggml_new_tensor_2d(ctx, wtype,           n_audio_state, n_audio_state);
                layer.attn_v_b    = ggml_new_tensor_1d(ctx, GGML_TYPE_F32,   n_audio_state);

                layer.attn_ln_1_w = ggml_new_tensor_2d(ctx, wtype,           n_audio_state, n_audio_state);
                layer.attn_ln_1_b = ggml_new_tensor_1d(ctx, GGML_TYPE_F32,   n_audio_state);

                // map by name
                model.tensors["encoder.blocks." + std::to_string(i) + ".mlp_ln.weight"]     = layer.mlp_ln_w;
                model.tensors["encoder.blocks." + std::to_string(i) + ".mlp_ln.bias"]       = layer.mlp_ln_b;

                model.tensors["encoder.blocks." + std::to_string(i) + ".mlp.0.weight"]      = layer.mlp_0_w;
                model.tensors["encoder.blocks." + std::to_string(i) + ".mlp.0.bias"]        = layer.mlp_0_b;

                model.tensors["encoder.blocks." + std::to_string(i) + ".mlp.2.weight"]      = layer.mlp_1_w;
                model.tensors["encoder.blocks." + std::to_string(i) + ".mlp.2.bias"]        = layer.mlp_1_b;

                model.tensors["encoder.blocks." + std::to_string(i) + ".attn_ln.weight"]    = layer.attn_ln_0_w;
                model.tensors["encoder.blocks." + std::to_string(i) + ".attn_ln.bias"]      = layer.attn_ln_0_b;

                model.tensors["encoder.blocks." + std::to_string(i) + ".attn.query.weight"] = layer.attn_q_w;
                model.tensors["encoder.blocks." + std::to_string(i) + ".attn.query.bias"]   = layer.attn_q_b;

                model.tensors["encoder.blocks." + std::to_string(i) + ".attn.key.weight"]   = layer.attn_k_w;

                model.tensors["encoder.blocks." + std::to_string(i) + ".attn.value.weight"] = layer.attn_v_w;
                model.tensors["encoder.blocks." + std::to_string(i) + ".attn.value.bias"]   = layer.attn_v_b;

                model.tensors["encoder.blocks." + std::to_string(i) + ".attn.out.weight"]   = layer.attn_ln_1_w;
                model.tensors["encoder.blocks." + std::to_string(i) + ".attn.out.bias"]     = layer.attn_ln_1_b;
            }
        }

        // decoder
        {
            model.d_pe   = ggml_new_tensor_2d(ctx, GGML_TYPE_F32, n_text_state, n_text_ctx);

            model.d_te   = ggml_new_tensor_2d(ctx, wtype,         n_text_state, n_vocab);

            model.d_ln_w = ggml_new_tensor_1d(ctx, GGML_TYPE_F32, n_text_state);
            model.d_ln_b = ggml_new_tensor_1d(ctx, GGML_TYPE_F32, n_text_state);

            // map by name
            model.tensors["decoder.positional_embedding"]   = model.d_pe;

            model.tensors["decoder.token_embedding.weight"] = model.d_te;

            model.tensors["decoder.ln.weight"]              = model.d_ln_w;
            model.tensors["decoder.ln.bias"]                = model.d_ln_b;

            for (int i = 0; i < n_text_layer; ++i) {
                auto & layer = model.layers_decoder[i];

                layer.mlp_ln_w          = ggml_new_tensor_1d(ctx, GGML_TYPE_F32,   n_text_state);
                layer.mlp_ln_b          = ggml_new_tensor_1d(ctx, GGML_TYPE_F32,   n_text_state);

                layer.mlp_0_w           = ggml_new_tensor_2d(ctx, wtype,           n_text_state, 4*n_text_state);
                layer.mlp_0_b           = ggml_new_tensor_1d(ctx, GGML_TYPE_F32, 4*n_text_state);

                layer.mlp_1_w           = ggml_new_tensor_2d(ctx, wtype,         4*n_text_state, n_text_state);
                layer.mlp_1_b           = ggml_new_tensor_1d(ctx, GGML_TYPE_F32,   n_text_state);

                layer.attn_ln_0_w       = ggml_new_tensor_1d(ctx, GGML_TYPE_F32,   n_text_state);
                layer.attn_ln_0_b       = ggml_new_tensor_1d(ctx, GGML_TYPE_F32,   n_text_state);

                layer.attn_q_w          = ggml_new_tensor_2d(ctx, wtype,           n_text_state, n_text_state);
                layer.attn_q_b          = ggml_new_tensor_1d(ctx, GGML_TYPE_F32,   n_text_state);

                layer.attn_k_w          = ggml_new_tensor_2d(ctx, wtype,           n_text_state, n_text_state);

                layer.attn_v_w          = ggml_new_tensor_2d(ctx, wtype,           n_text_state, n_text_state);
                layer.attn_v_b          = ggml_new_tensor_1d(ctx, GGML_TYPE_F32,   n_text_state);

                layer.attn_ln_1_w       = ggml_new_tensor_2d(ctx, wtype,           n_text_state, n_text_state);
                layer.attn_ln_1_b       = ggml_new_tensor_1d(ctx, GGML_TYPE_F32,   n_text_state);

                layer.cross_attn_ln_0_w = ggml_new_tensor_1d(ctx, GGML_TYPE_F32,   n_text_state);
                layer.cross_attn_ln_0_b = ggml_new_tensor_1d(ctx, GGML_TYPE_F32,   n_text_state);

                layer.cross_attn_q_w    = ggml_new_tensor_2d(ctx, wtype,           n_text_state, n_text_state);
                layer.cross_attn_q_b    = ggml_new_tensor_1d(ctx, GGML_TYPE_F32,   n_text_state);

                layer.cross_attn_k_w    = ggml_new_tensor_2d(ctx, wtype,           n_text_state, n_text_state);

                layer.cross_attn_v_w    = ggml_new_tensor_2d(ctx, wtype,           n_text_state, n_text_state);
                layer.cross_attn_v_b    = ggml_new_tensor_1d(ctx, GGML_TYPE_F32,   n_text_state);

                layer.cross_attn_ln_1_w = ggml_new_tensor_2d(ctx, wtype,           n_text_state, n_text_state);
                layer.cross_attn_ln_1_b = ggml_new_tensor_1d(ctx, GGML_TYPE_F32,   n_text_state);

                // map by name
                model.tensors["decoder.blocks." + std::to_string(i) + ".mlp_ln.weight"]           = layer.mlp_ln_w;
                model.tensors["decoder.blocks." + std::to_string(i) + ".mlp_ln.bias"]             = layer.mlp_ln_b;

                model.tensors["decoder.blocks." + std::to_string(i) + ".mlp.0.weight"]            = layer.mlp_0_w;
                model.tensors["decoder.blocks." + std::to_string(i) + ".mlp.0.bias"]              = layer.mlp_0_b;

                model.tensors["decoder.blocks." + std::to_string(i) + ".mlp.2.weight"]            = layer.mlp_1_w;
                model.tensors["decoder.blocks." + std::to_string(i) + ".mlp.2.bias"]              = layer.mlp_1_b;

                model.tensors["decoder.blocks." + std::to_string(i) + ".attn_ln.weight"]          = layer.attn_ln_0_w;
                model.tensors["decoder.blocks." + std::to_string(i) + ".attn_ln.bias"]            = layer.attn_ln_0_b;

                model.tensors["decoder.blocks." + std::to_string(i) + ".attn.query.weight"]       = layer.attn_q_w;
                model.tensors["decoder.blocks." + std::to_string(i) + ".attn.query.bias"]         = layer.attn_q_b;

                model.tensors["decoder.blocks." + std::to_string(i) + ".attn.key.weight"]         = layer.attn_k_w;

                model.tensors["decoder.blocks." + std::to_string(i) + ".attn.value.weight"]       = layer.attn_v_w;
                model.tensors["decoder.blocks." + std::to_string(i) + ".attn.value.bias"]         = layer.attn_v_b;

                model.tensors["decoder.blocks." + std::to_string(i) + ".attn.out.weight"]         = layer.attn_ln_1_w;
                model.tensors["decoder.blocks." + std::to_string(i) + ".attn.out.bias"]           = layer.attn_ln_1_b;

                model.tensors["decoder.blocks." + std::to_string(i) + ".cross_attn_ln.weight"]    = layer.cross_attn_ln_0_w;
                model.tensors["decoder.blocks." + std::to_string(i) + ".cross_attn_ln.bias"]      = layer.cross_attn_ln_0_b;

                model.tensors["decoder.blocks." + std::to_string(i) + ".cross_attn.query.weight"] = layer.cross_attn_q_w;
                model.tensors["decoder.blocks." + std::to_string(i) + ".cross_attn.query.bias"]   = layer.cross_attn_q_b;

                model.tensors["decoder.blocks." + std::to_string(i) + ".cross_attn.key.weight"]   = layer.cross_attn_k_w;

                model.tensors["decoder.blocks." + std::to_string(i) + ".cross_attn.value.weight"] = layer.cross_attn_v_w;
                model.tensors["decoder.blocks." + std::to_string(i) + ".cross_attn.value.bias"]   = layer.cross_attn_v_b;

                model.tensors["decoder.blocks." + std::to_string(i) + ".cross_attn.out.weight"]   = layer.cross_attn_ln_1_w;
                model.tensors["decoder.blocks." + std::to_string(i) + ".cross_attn.out.bias"]     = layer.cross_attn_ln_1_b;
            }
        }
    }

    // load weights
    {
        size_t total_size = 0;

        model.n_loaded = 0;

        while (true) {
            int32_t n_dims;
            int32_t length;
            int32_t ttype;

            read_safe(loader, n_dims);
            read_safe(loader, length);
            read_safe(loader, ttype);

            if (loader->eof(loader->context)) {
                break;
            }

            int32_t nelements = 1;
            int32_t ne[4] = { 1, 1, 1, 1 };
            for (int i = 0; i < n_dims; ++i) {
                read_safe(loader, ne[i]);
                nelements *= ne[i];
            }

            std::string name;
            std::vector<char> tmp(length); // create a buffer
            loader->read(loader->context, &tmp[0], tmp.size()); // read to buffer
            name.assign(&tmp[0], tmp.size());

            if (model.tensors.find(name) == model.tensors.end()) {
                log("%s: unknown tensor '%s' in model file\n", __func__, name.data());
                return false;
            }

            auto tensor = model.tensors[name.data()];
            if (ggml_nelements(tensor) != nelements) {
                log("%s: tensor '%s' has wrong size in model file\n", __func__, name.data());
                log("%s: shape: [%d, %d, %d], expected: [%d, %d, %d]\n",
                        __func__, ne[0], ne[1], ne[2], (int) tensor->ne[0], (int) tensor->ne[1], (int) tensor->ne[2]);
                return false;
            }

            if (tensor->ne[0] != ne[0] || tensor->ne[1] != ne[1] || tensor->ne[2] != ne[2]) {
                log("%s: tensor '%s' has wrong shape in model file: got [%d, %d, %d], expected [%d, %d, %d]\n",
                        __func__, name.data(), (int) tensor->ne[0], (int) tensor->ne[1], (int) tensor->ne[2], ne[0], ne[1], ne[2]);
                return false;
            }

            const size_t bpe = ggml_type_size(ggml_type(ttype));

            if ((nelements*bpe)/ggml_blck_size(tensor->type) != ggml_nbytes(tensor)) {
                log("%s: tensor '%s' has wrong size in model file: got %zu, expected %zu\n",
                        __func__, name.data(), ggml_nbytes(tensor), nelements*bpe);
                return false;
            }

            loader->read(loader->context, tensor->data, ggml_nbytes(tensor));
            BYTESWAP_TENSOR(tensor);

            //printf("%48s - [%5d, %5d, %5d], type = %6s, %6.2f MB\n", name.data(), ne[0], ne[1], ne[2], ggml_type_name((ggml_type) ttype), ggml_nbytes(tensor)/1024.0/1024.0);
            total_size += ggml_nbytes(tensor);
            model.n_loaded++;
        }

        log("%s: model size    = %7.2f MB\n", __func__, total_size/1024.0/1024.0);

        if (model.n_loaded == 0) {
            log("%s: WARN no tensors loaded from model file - assuming empty model for testing\n", __func__);
        } else if (model.n_loaded != (int) model.tensors.size()) {
            log("%s: ERROR not all tensors loaded from model file - expected %zu, got %d\n", __func__, model.tensors.size(), model.n_loaded);
            return false;
        }
    }

    wctx.t_load_us = ggml_time_us() - t_start_us;

    return true;
}

```



该代码是一个静态函数 `whisper_encode_external()`，其作用是检查 `whisper_state` 是否符合某些条件，并返回相应的结果。

以下是代码的功能解释：

1. 该函数的参数 `wstate` 是一个 `whisper_state` 类型的常量，因此该函数可以随时接收不同的 `whisper_state` 作为参数。

2. 函数内部包含两个条件判断，根据不同的条件使用不同的功能模块。

3.第一个条件判断 `use_coreml` 是否为真，如果为真，则说明使用CoreML模型的训练。

4.第二个条件判断 `use_openvino` 是否为真，如果为真，则说明使用 OpenVIO 模型的训练。

5. 如果两个条件都不符合，则不使用任何特定的模型训练，返回 `false`。

6. 如果第一个条件符合，而第二个条件不符合，则说明使用CoreML模型，但不使用OpenVIO模型训练，返回 `false`。

7. 如果第一个条件不符合，而第二个条件符合，则说明使用OpenVIO模型，但不使用CoreML模型训练，返回 `false`。

8. 如果两个条件都符合，则说明同时使用CoreML模型和OpenVIO模型训练，返回 `true`。

9. 最后，该函数返回的结果可以用来做出某些决策，例如选择特定的模型训练计划。


```cpp
static bool whisper_encode_external(const whisper_state & wstate) {
    GGML_UNUSED(wstate);

#ifndef WHISPER_USE_COREML
    const bool use_coreml = false;
#else
    const bool use_coreml = wstate.ctx_coreml != nullptr;
#endif

#ifndef WHISPER_USE_OPENVINO
    const bool use_openvino = false;
#else
    const bool use_openvino = wstate.ctx_openvino != nullptr;
#endif

    return use_coreml || use_openvino;
}

```

首先，我们检查输入的 Mel 是否与预期类型 `GGML_TYPE_F32` 相等。接下来，我们检查是否成功对外部表示法中的 "GGML_ALLOC_IS_MEASURE" 选项进行验证。如果成功，我们继续在循环中，否则我们跳过外部表示法部分。

接下来，我们检查输入的 `MEL_INP` 是否与预期长度 `n_mels` 相等。然后，我们开始通过循环将外部表示法中的数据复制到输出向量 `DST` 中。最后，我们将 `DST` 中的数据转换为 float 类型，以便进行后续操作。

对于成功的结果，我们需要确保所有内部表示法都正确，并且在循环中使用正确的数据索引。所以，我们使用 `GGML_TYPE_F32` 来确保输入的数据类型与我们的预期相等。


```cpp
static struct ggml_cgraph * whisper_build_graph_conv(
        whisper_context & wctx,
          whisper_state & wstate,
              const int   mel_offset) {
    const auto & model   = wctx.model;
    const auto & mel_inp = wstate.mel;
    const auto & hparams = model.hparams;

    const int n_ctx   = wstate.exp_n_audio_ctx > 0 ? wstate.exp_n_audio_ctx : hparams.n_audio_ctx;
    const int n_state = hparams.n_audio_state; GGML_UNUSED(n_state);

    const int n_mels = hparams.n_mels;

    struct ggml_init_params params = {
        /*.mem_size   =*/ wstate.alloc_conv.meta.size(),
        /*.mem_buffer =*/ wstate.alloc_conv.meta.data(),
        /*.no_alloc   =*/ true,
    };

    struct ggml_context * ctx0 = ggml_init(params);

    ggml_cgraph * gf = ggml_new_graph(ctx0);

    ggml_allocr * alloc = wstate.alloc_conv.alloc;

    struct ggml_tensor * mel = ggml_new_tensor_2d(ctx0, GGML_TYPE_F32, 2*n_ctx, n_mels);
    ggml_allocr_alloc(alloc, mel);

    assert(mel->type == GGML_TYPE_F32);
    if (!ggml_allocr_is_measure(alloc)) {
        assert(mel_inp.n_mel == n_mels);

        float * dst = (float *) mel->data;
        memset(dst, 0, ggml_nbytes(mel));

        const int i0 = std::min(mel_offset, mel_inp.n_len);
        const int i1 = std::min(mel_offset + 2*n_ctx, mel_inp.n_len);

        for (int j = 0; j < mel_inp.n_mel; ++j) {
            for (int i = i0; i < i1; ++i) {
                dst[j*2*n_ctx + (i - i0)] = mel_inp.data[j*mel_inp.n_len + i];
            }
        }
    }

    struct ggml_tensor * cur = nullptr;

    if (!whisper_encode_external(wstate)) {
        // convolution + gelu
        {
            cur = ggml_conv_1d_ph(ctx0, model.e_conv_1_w, mel, 1, 1);
            cur = ggml_add(ctx0,
                    ggml_repeat(ctx0,
                        model.e_conv_1_b,
                        cur),
                    cur);

            cur = ggml_gelu(ctx0, cur);

            cur = ggml_conv_1d_ph(ctx0, model.e_conv_2_w, cur, 2, 1);
            cur = ggml_add(ctx0,
                    ggml_repeat(ctx0,
                        model.e_conv_2_b,
                        cur),
                    cur);

            cur = ggml_gelu(ctx0, cur);
        }

        wstate.embd_conv = cur;
    } else {
```

这段代码的作用是实现了一个分类任务中的数据预处理。它通过以下步骤实现了：

1. 如果使用COREML_USE_COREML，则创建一个具有n_state和n_ctx行和n_element的2D张量，并将其赋值为零。

2. 如果n_state大于n_ctx，则需要将数据复制到内存中。因此，使用ggml_allocr_is_measure判断是否需要进行复制。如果需要，则使用whisper_coreml_encode将数据从内存中复制到自有内存中。

3. 如果使用WHISPER_USE_OPENVINO，则创建一个具有n_state和n_ctx行和n_element的2D张量，并将其赋值为零。

4. 如果n_state大于n_ctx，则需要将数据复制到内存中。因此，使用whisper_openvino_encode将数据从内存中复制到自有内存中。

5. 如果n_state小于n_ctx，则需要将数据复制到内存中。因此，使用ggml_allocr_is_measure判断是否需要进行复制。如果需要，则使用ggml_copy_to_内存将数据从内存中复制到自有内存中。


```cpp
#ifdef WHISPER_USE_COREML
        cur = ggml_new_tensor_2d(ctx0, GGML_TYPE_F32, n_state, n_ctx);
        ggml_allocr_alloc(alloc, cur);

        if (!ggml_allocr_is_measure(alloc)) {
            whisper_coreml_encode(wstate.ctx_coreml, (float *) mel->data, (float *) cur->data);
        }
#endif
#ifdef WHISPER_USE_OPENVINO
        cur = ggml_new_tensor_2d(ctx0, GGML_TYPE_F32, n_state, n_ctx);
        ggml_allocr_alloc(alloc, cur);

        if (!ggml_allocr_is_measure(alloc)) {
            whisper_openvino_encode(wstate.ctx_openvino, mel, cur);
        }
```

This is a C++ implementation of the Graphic User Interface (GUI) for a deep learning model.

The function `ggml_add()` is used to perform matrix addition. The function `ggml_norm()` is used to perform normalization of the input tensor. The function `ggml_mul()` is used to perform matrix multiplication. The function `ggml_scale()` is used to perform scaling of the input tensor. The function `ggml_transpose()` is used to transpose the input tensor. The function `ggml_add()` is used to perform matrix addition. The function `ggml_mul_mat()` is used to perform matrix multiplication and normalization. The function `ggml_new_f32()` is used to create a new floating-point number.


```cpp
#endif

        wstate.embd_enc = cur;
    }

    ggml_build_forward_expand(gf, cur);

    ggml_free(ctx0);

    return gf;
}

static struct ggml_cgraph * whisper_build_graph_encoder(
        whisper_context & wctx,
          whisper_state & wstate) {
    const auto & model   = wctx.model;
    const auto & hparams = model.hparams;

    const int n_ctx   = wstate.exp_n_audio_ctx > 0 ? wstate.exp_n_audio_ctx : hparams.n_audio_ctx;
    const int n_state = hparams.n_audio_state;
    const int n_head  = hparams.n_audio_head;
    const int n_layer = hparams.n_audio_layer;

    struct ggml_init_params params = {
        /*.mem_size   =*/ wstate.alloc_encode.meta.size(),
        /*.mem_buffer =*/ wstate.alloc_encode.meta.data(),
        /*.no_alloc   =*/ true,
    };

    struct ggml_context * ctx0 = ggml_init(params);

    ggml_cgraph * gf = ggml_new_graph_custom(ctx0, WHISPER_MAX_NODES, false);

    ggml_allocr * alloc = wstate.alloc_encode.alloc;

    struct ggml_tensor * KQscale = ggml_new_tensor_1d(ctx0, GGML_TYPE_F32, 1);
    ggml_allocr_alloc(alloc, KQscale);

    if (!ggml_allocr_is_measure(alloc)) {
        ggml_set_f32(KQscale, 1.0f/sqrt(float(n_state)/n_head));
    }

    struct ggml_tensor * cur = ggml_view_tensor(ctx0, wstate.embd_conv);

    // ===================================================================
    // NOTE: experimenting with partial evaluation of the encoder (ignore)
    //static int iter = -1;
    //const int n_iter = 1500/n_ctx;

    //iter = (iter + 1) % n_iter;

    //if (iter == 0) {
    //    memset(model.memory_cross_k->data, 0, ggml_nbytes(model.memory_cross_k));
    //    memset(model.memory_cross_v->data, 0, ggml_nbytes(model.memory_cross_v));
    //}

    static int iter = 0;

    const size_t e_pe_stride = model.e_pe->ne[0]*ggml_element_size(model.e_pe);
    const size_t e_pe_offset = model.e_pe->ne[0]*ggml_element_size(model.e_pe)*n_ctx*iter;

    struct ggml_tensor * e_pe = ggml_view_2d(ctx0, model.e_pe, model.e_pe->ne[0], n_ctx, e_pe_stride, e_pe_offset);

    cur = ggml_add(ctx0, e_pe, ggml_cont(ctx0, ggml_transpose(ctx0, cur)));

    // ===================================================================

    // original:
    //cur = ggml_add(ctx0, model.e_pe, ggml_transpose(ctx0, cur));

    struct ggml_tensor * inpL = cur;

    for (int il = 0; il < n_layer; ++il) {
        const auto & layer = model.layers_encoder[il];

        // norm
        {
            cur = ggml_norm(ctx0, inpL, hparams.eps);

            // cur = ln_0_w*cur + ln_0_b
            cur = ggml_add(ctx0,
                    ggml_mul(ctx0, cur, layer.attn_ln_0_w),
                    layer.attn_ln_0_b);
        }

        // self-attention
        {
            struct ggml_tensor * Qcur = ggml_mul_mat(ctx0,
                    layer.attn_q_w,
                    cur);

            Qcur = ggml_add(ctx0, Qcur, layer.attn_q_b);

            //Qcur = ggml_scale(ctx0, Qcur, ggml_new_f32(ctx0, pow(float(n_state)/n_head, -0.25)));

            // note: no bias for Key
            struct ggml_tensor * Kcur = ggml_mul_mat(ctx0,
                    layer.attn_k_w,
                    cur);

            //Kcur = ggml_scale(ctx0, Kcur, ggml_new_f32(ctx0, pow(float(n_state)/n_head, -0.25)));

            struct ggml_tensor * Vcur = ggml_mul_mat(ctx0,
                    layer.attn_v_w,
                    cur);

            Vcur = ggml_add(ctx0, Vcur, layer.attn_v_b);

            // ------

```

这段代码使用了跨平台GPU加速的库ffi-ml-伦伦库，通过声明三个选项Q，K和V，定义了输入数据的形状，然后通过ffi-ml-伦伦库中的函数进行了一系列的3D变换，最后通过ffi-ml-伦伦库中的函数进行了一组Flash Attention操作。

通过ffi-ml-伦伦库中的函数，首先将输入数据Kcur和Vcur转换为3D张量，并进行了几个卷积神经网络的前向传播，获得了Q和K选项。接着，将K选项和V选项进行了一个新的3D张量Vcur，该张量的形状是(n_head, n_state/n_head, n_ctx)。

最后，通过ffi-ml-伦伦库中的flash_attn函数对输入的Q，K和V选项进行Flash Attention操作，将输入的信息进行增强，以提高模型的性能。


```cpp
#ifdef WHISPER_USE_FLASH_ATTN
            struct ggml_tensor * Q =
                ggml_permute(ctx0,
                        ggml_cpy(ctx0,
                            Qcur,
                            ggml_new_tensor_3d(ctx0, wctx.itype, n_state/n_head, n_head, n_ctx)),
                        0, 2, 1, 3);

            struct ggml_tensor * K =
                ggml_permute(ctx0,
                        ggml_cpy(ctx0,
                            Kcur,
                            ggml_new_tensor_3d(ctx0, wctx.itype, n_state/n_head, n_head, n_ctx)),
                        0, 2, 1, 3);

            struct ggml_tensor * V =
                ggml_cpy(ctx0,
                        ggml_permute(ctx0,
                            ggml_reshape_3d(ctx0,
                                Vcur,
                                n_state/n_head, n_head, n_ctx),
                            1, 2, 0, 3),
                        ggml_new_tensor_3d(ctx0, wctx.itype, n_ctx, n_state/n_head, n_head));

            struct ggml_tensor * KQV = ggml_flash_attn(ctx0, Q, K, V, false);
```

这段代码定义了一个名为 "Q" 的结构体，它是一个 3 层 GPU 标量的指针。接着定义了一个名为 "K" 的结构体，它也是一个 3 层 GPU 标量的指针。

在这之后，代码使用了一系列的 GPU 标量操作，包括 CPU 串行化。

最后，KQ 被计算出来并且进行了一些激活函数计算。


```cpp
#else
            struct ggml_tensor * Q =
                ggml_permute(ctx0,
                        ggml_cpy(ctx0,
                            Qcur,
                            ggml_new_tensor_3d(ctx0, GGML_TYPE_F32, n_state/n_head, n_head, n_ctx)),
                        0, 2, 1, 3);

            struct ggml_tensor * K =
                ggml_permute(ctx0,
                        ggml_cpy(ctx0,
                            Kcur,
                            ggml_new_tensor_3d(ctx0, wctx.itype, n_state/n_head, n_head, n_ctx)),
                        0, 2, 1, 3);

            // K * Q
            struct ggml_tensor * KQ = ggml_mul_mat(ctx0, K, Q);

            struct ggml_tensor * KQ_scaled = ggml_scale(ctx0, KQ, KQscale);

            struct ggml_tensor * KQ_soft_max = ggml_soft_max(ctx0, KQ_scaled);

            struct ggml_tensor * V =
                ggml_cpy(ctx0,
                        ggml_permute(ctx0,
                            ggml_reshape_3d(ctx0,
                                Vcur,
                                n_state/n_head, n_head, n_ctx),
                            1, 2, 0, 3),
                        ggml_new_tensor_3d(ctx0, wctx.itype, n_ctx, n_state/n_head, n_head)
                        );

            struct ggml_tensor * KQV = ggml_mul_mat(ctx0, V, KQ_soft_max);
```

这段代码是一个用深度学习框架GGML（Google Graphical Library）编写的C语言函数，主要作用是处理一个由多个神经网络层和一个输入数据组成的张量。通过以下步骤，该函数实现了一个将输入张量`inpL`与神经网络中的特征向量`cur`相加并输入到神经网络的第一个隐藏层中的操作：

1. 从KQV（一种存储张量的方式，Key值，值和数据大小）结构中选择一个名为`KQV_merged`的表示，并使用`ggml_permute`函数对其进行排列。

2. 使用`ggml_new_tensor_2d`函数创建一个大小为`(GGML_INDEX_COUNT, GGML_INDEX_COUNT)`的二维张量，并将其标以`GGML_TYPE_F32`类型。

3. 从`KQV_merged`中按列展开，将每列的`cur`值与对应的`layer.attn_ln_1_w`和`layer.attn_ln_1_b`值相加，得到一个新的张量`cur_ new`。

4. 使用`ggml_add`函数将输入张量`inpL`与`cur_ new`相加，得到一个新的张量`inpFF`。

5. 将`inpFF`作为输入传递给一个使用AddL异步训练函数的神经网络层。

6. 对输入张量`inpL`应用 feed-forward网络，并计算其欧氏范数以设置超参数`hparams.eps`。

7. 使用`ggml_norm`函数计算每个神经元的欧氏范数，然后将其与`cur_ new`相乘，得到一个新的张量`cur_ norm`。

8. 使用`ggml_add`函数将`cur_norm`和输入张量`inpL`相加，得到一个新的张量`cur_ final`。

9. 返回新创建的`cur_final`张量，用于输出神经网络的训练结果。


```cpp
#endif
            struct ggml_tensor * KQV_merged = ggml_permute(ctx0, KQV, 0, 2, 1, 3);

            cur = ggml_cpy(ctx0,
                    KQV_merged,
                    ggml_new_tensor_2d(ctx0, GGML_TYPE_F32, n_state, n_ctx));
        }

        // projection
        {
            cur = ggml_mul_mat(ctx0,
                    layer.attn_ln_1_w,
                    cur);

            cur = ggml_add(ctx0, cur, layer.attn_ln_1_b);
        }

        // add the input
        cur = ggml_add(ctx0, cur, inpL);

        struct ggml_tensor * inpFF = cur;

        // feed-forward network
        {
            // norm
            {
                cur = ggml_norm(ctx0, inpFF, hparams.eps);

                // cur = mlp_ln_w*cur + mlp_ln_b
                cur = ggml_add(ctx0,
                        ggml_mul(ctx0, cur, layer.mlp_ln_w),
                        layer.mlp_ln_b);
            }

```

这段代码是一个 conditional 代码，判断是否使用了 Flash 存储介质。如果没有使用 Flash，那么会执行以下操作：

1. 使用 Flash 存储介质时，首先进行以下操作：
```cppcss
   cur = ggml_flash_ff(ctx0,
                   ggml_cpy(ctx0, cur, ggml_new_tensor_2d(ctx0, wstate.itype, n_state, n_ctx)),
                   layer.mlp_0_w, layer.mlp_0_b, layer.mlp_1_w, layer.mlp_1_b);
```
2. 如果已经使用了 Flash，那么以下操作将不再执行。
```cppcss
   // fully connected
   cur = ggml_mul_mat(ctx0,
                   layer.mlp_0_w,
                   cur);

   cur = ggml_add(ctx0, cur, layer.mlp_0_b);

   // GELU activation
   cur = ggml_gelu(ctx0, cur);

   // projection
   cur = ggml_mul_mat(ctx0,
                   layer.mlp_1_w,
                   cur);

   cur = ggml_add(ctx0, cur, layer.mlp_1_b);
```
如果没有使用 Flash，那么上述代码将不会执行。


```cpp
#ifdef WHISPER_USE_FLASH_FF
            cur = ggml_flash_ff(ctx0,
                    ggml_cpy(ctx0, cur, ggml_new_tensor_2d(ctx0, wstate.itype, n_state, n_ctx)),
                    layer.mlp_0_w, layer.mlp_0_b, layer.mlp_1_w, layer.mlp_1_b);
#else
            // fully connected
            cur = ggml_mul_mat(ctx0,
                    layer.mlp_0_w,
                    cur);

            cur = ggml_add(ctx0, cur, layer.mlp_0_b);

            // GELU activation
            cur = ggml_gelu(ctx0, cur);

            // projection
            cur = ggml_mul_mat(ctx0,
                    layer.mlp_1_w,
                    cur);

            cur = ggml_add(ctx0, cur, layer.mlp_1_b);
```

这段代码是一个 C++ 函数，属于星际量子门（IQG）库的一部分。它的主要作用是执行一个矩阵的转置操作（transpose）。以下是代码的主要步骤：

1. 首先定义了一个名为 "ggml_add" 的函数，它接受两个参数：一个 ctx0 表示一个计算图（computation graph）和一个输入矩阵 inpFF，它包含输入的节点。函数返回一个计算图中的节点。

2. 然后定义了一个名为 "ggml_norm" 的函数，它接受一个参数 cur，一个参数 hparams，表示允许的误差范围。函数返回 cur 中的节点进行 L2 范数（Euclidean distance）后的结果。

3. 在 ggml_add 和 ggml_norm 函数后面，定义了一个名为 "cur" 的变量。这个变量将用作输入矩阵 inpFF 中节点的输入。

4. 在 ggml_add 和 ggml_norm 函数之外，还有一个 "norm" 函数，它的参数是 cur 和 hparams。这个函数计算输入矩阵中的节点进行 L2 范数（Euclidean distance）后的结果，并将其存储在 cur 中。

5. 在 "norm" 函数之后，定义了一个名为 "cur" 的变量。它是输入矩阵 inpFF 中节点的输入。

6. 在 "cur" 变量被分配给 "cur" 变量之后，定义了一个名为 "wstate" 的变量。这个变量包含用于构建计算图的 forward 链路的参数。

7. 在 "wstate" 变量被创建之后，定义了一个名为 "ggml_graph_print" 的函数。这个函数用于打印计算图。

8. 在所有函数之后，定义了一个名为 "main" 的函数。这个函数不知道从哪里获取，但用于打印输出。

9. 在 "main" 函数中，首先创建一个计算图 gf。然后，调用 ggml_add、ggml_norm 和 ggml_graph_print 函数，设置 cur 和 wstate。最后，创建并打印 gf。


```cpp
#endif
        }

        inpL = ggml_add(ctx0, cur, inpFF);
    }

    cur = inpL;

    // norm
    {
        cur = ggml_norm(ctx0, cur, hparams.eps);

        // cur = ln_f_g*cur + ln_f_b
        cur = ggml_add(ctx0,
                ggml_mul(ctx0, cur, model.e_ln_w),
                model.e_ln_b);
    }

    ggml_build_forward_expand(gf, cur);

    wstate.embd_enc = cur;

    //ggml_graph_print(gf);

    ////////////////////////////////////////////////////////////////////////////

    //printf("%s: used_mem = %f MB, %f MB, %f MB %f MB %f MB\n", __func__,
    //        ggml_used_mem(ctx0)/1024.0/1024.0,
    //        wstate.get_buf_max_mem(0)/1024.0/1024.0,
    //        wstate.get_buf_max_mem(1)/1024.0/1024.0,
    //        wstate.get_buf_max_mem(2)/1024.0/1024.0,
    //        wstate.get_buf_max_mem(3)/1024.0/1024.0);

    ggml_free(ctx0);

    return gf;
}

```

This function appears to be part of a neural network model that uses multi-head attention mechanisms to process natural language text data.

It first defines a forward pass through the input natural language text data, which is processed by the layers decoder.

The forward pass performs a combination of matrix multiplication, element-wise addition, and transposition of the output tensors of the attention mechanisms.

The `ggml_view_1d` function is used to perform a 1D view operation on the output tensor of the attention mechanism, where the user specifies the `k` and `v` variables.

The `ggml_view_2d` function is then used to perform a 2D view operation on the output tensor, where the user specifies the `v` variable.

Finally, the `ggml_graph_print` function is used to print the graph of the network.

Note that this code snippet is just a rough implementation and may not work as expected due to the absence of some crucial data and functions, such as the input data, the attention mechanism, and the loss function.


```cpp
// pre-compute cross-attention memory
static struct ggml_cgraph * whisper_build_graph_cross(
        whisper_context & wctx,
          whisper_state & wstate) {
    const auto & model   = wctx.model;
    const auto & hparams = model.hparams;

    const int n_ctx   = wstate.exp_n_audio_ctx > 0 ? wstate.exp_n_audio_ctx : hparams.n_audio_ctx;
    const int n_state = hparams.n_audio_state;
    const int n_head  = hparams.n_audio_head;

    struct ggml_init_params params = {
        /*.mem_size   =*/ wstate.alloc_cross.meta.size(),
        /*.mem_buffer =*/ wstate.alloc_cross.meta.data(),
        /*.no_alloc   =*/ true,
    };

    struct ggml_context * ctx0 = ggml_init(params);

    ggml_cgraph * gf = ggml_new_graph(ctx0);

    ggml_allocr * alloc = wstate.alloc_cross.alloc;

    struct ggml_tensor * cur = ggml_view_tensor(ctx0, wstate.embd_enc);

    struct ggml_tensor * Kscale = ggml_new_tensor_1d(ctx0, GGML_TYPE_F32, 1);
    ggml_allocr_alloc(alloc, Kscale);

    if (!ggml_allocr_is_measure(alloc)) {
        ggml_set_f32(Kscale, pow(float(n_state) / n_head, -0.25));
    }

    for (int il = 0; il < model.hparams.n_text_layer; ++il) {
        auto & layer = model.layers_decoder[il];

        struct ggml_tensor* Kcross = ggml_mul_mat(ctx0,
                layer.cross_attn_k_w,
                cur);

        Kcross = ggml_scale(ctx0, Kcross, Kscale);

        struct ggml_tensor* Vcross = ggml_mul_mat(ctx0,
                layer.cross_attn_v_w,
                cur);

        Vcross = ggml_add(ctx0,
                    Vcross,
                    layer.cross_attn_v_b);

        Vcross = ggml_transpose(ctx0, ggml_reshape_2d(ctx0, Vcross, n_state, n_ctx));

        struct ggml_tensor * k = ggml_view_1d(ctx0, wstate.kv_cross.k,
                n_state*n_ctx,
                (ggml_element_size(wstate.kv_cross.k)*n_state)*(il*n_ctx));

        struct ggml_tensor * v = ggml_view_2d(ctx0, wstate.kv_cross.v, n_ctx, n_state,
                (   n_ctx)*ggml_element_size(wstate.kv_cross.v),
                (il*n_ctx)*ggml_element_size(wstate.kv_cross.v)*n_state);

        ggml_build_forward_expand(gf, ggml_cpy(ctx0, Kcross, k));
        ggml_build_forward_expand(gf, ggml_cpy(ctx0, Vcross, v));
    }

    //ggml_graph_print(gf);

    ggml_free(ctx0);

    return gf;
}

```

这段代码的作用是使用所给的音频录音（即其对 mel 谱图）进行前馈操作，然后是编码器前馈。它将计算编码器所需的参数，并在内部执行此操作以获得编码器生成的编码。

具体来说，这段代码将执行以下操作：

1. 将输入 mel 谱图和参数（如 wctx、wstate 和 n_threads）存储在 wstate 中。
2. 如果已经编码过音频，将存储的编码器生成的编码作为结果。
3. 如果尚未编码过音频，它将构建一个图形编码器 graph，使用 n_threads 个线程并尝试执行编码器前馈。在执行编码器前馈时，它将使用给定的 wctx 和 wstate，并传递音频的 offsets（即 mel 谱图的音频延迟）。
4. 如果编码器前馈失败，它将调用给定的 abort_callback 函数。abort_callback 将收到一个 void 类型的数据，其中包含 abort 信号，用于通知编码器已准备好停止执行操作。
5. 最后，它将在完成编码器前馈后，如果成功，将使用 wstate.alloc_encode.alloc 中的内存分配，并使用 ggml_time_us() 函数获取执行操作的时钟中断值，从而确保在图形编码器计算之前获取正确的截止时间。


```cpp
// evaluate the encoder with the given state
//
// given audio recording (more specifically, its log mel spectrogram), runs forward pass of the encoder
// part of the transformer model and returns the encoded features
//
//   - wctx:      the model
//   - wstate:     the state of the encoder
//   - n_threads:  number of threads to use
//   - mel_offset: offset in the mel spectrogram (i.e. audio offset)
//
static bool whisper_encode_internal(
        whisper_context & wctx,
          whisper_state & wstate,
              const int   mel_offset,
              const int   n_threads,
 whisper_abort_callback   abort_callback,
                   void * abort_callback_data) {
    const int64_t t_start_us = ggml_time_us();

    // conv
    {
        auto & alloc = wstate.alloc_conv.alloc;

        ggml_allocr_reset(alloc);

        ggml_cgraph * gf = whisper_build_graph_conv(wctx, wstate, mel_offset);

        ggml_allocr_alloc_graph(alloc, gf);

        if (!whisper_encode_external(wstate)) {
            ggml_graph_compute_helper(wstate.work_buffer, gf, n_threads, abort_callback, abort_callback_data);
        }
    }

    // encoder
    if (!whisper_encode_external(wstate)) {
        auto & alloc = wstate.alloc_encode.alloc;

        ggml_allocr_reset(alloc);

        ggml_cgraph * gf = whisper_build_graph_encoder(wctx, wstate);

        ggml_allocr_alloc_graph(alloc, gf);

```

这段代码是用来在GGML和Metal之间进行交互的代码。它通过检查当前的工作状态（wstate）中是否定义了使用Metal的选项（GGML_USE_METAL），如果是，那么它将设置n_threads参数并调用ggml_metal_graph_compute函数来计算Metal图。否则，它将调用ggml_graph_compute_helper函数来计算Metal图。

代码中包含两个判断条件，判断是否使用Metal编译，如果使用则执行ggml_metal_set_n_cb函数设置n_threads，并调用ggml_metal_graph_compute函数计算Metal图。如果未定义使用Metal编译，则执行ggml_graph_compute_helper函数计算Metal图。

另外，在代码的最后部分，还包含一个输出语句，用于将计算结果输出。


```cpp
#ifdef GGML_USE_METAL
        if (wstate.ctx_metal) {
            ggml_metal_set_n_cb     (wstate.ctx_metal, n_threads);
            ggml_metal_graph_compute(wstate.ctx_metal, gf);
        } else {
            ggml_graph_compute_helper(wstate.work_buffer, gf, n_threads, abort_callback, abort_callback_data);
        }
#else
        ggml_graph_compute_helper(wstate.work_buffer, gf, n_threads, abort_callback, abort_callback_data);
#endif
    }

    // cross
    {
        auto & alloc = wstate.alloc_cross.alloc;

        ggml_allocr_reset(alloc);

        ggml_cgraph * gf = whisper_build_graph_cross(wctx, wstate);

        ggml_allocr_alloc_graph(alloc, gf);

```

这段代码是一个 C 语言函数，名为 `ggml_use_metal`。它用于在 ONNX 图形模型兼容的 GPU（Metal）设备上执行图形计算。

具体来说，这段代码的作用如下：

1. 如果开启了GGML对Metal的兼容，那么会执行以下操作：

  a. 设置 `n_threads` 为 `wstate.ctx_metal` 中的 `n_threads`。

  b. 使用 `ggml_metal_graph_compute` 函数对 `wstate.ctx_metal` 中的 `GF` 进行图形计算。

  c. 如果 `wstate.ctx_metal` 中没有开启对Metal的兼容，那么会执行以下操作：

     - 使用 `ggml_graph_compute_helper` 函数，同时传递 `wstate.work_buffer`、`GF` 和 `n_threads`，并设置 `abort_callback` 和 `abort_callback_data` 为 `null`。

2. 如果 `ggml_use_metal` 被定义为 `GGML_USE_METAL`，那么每次调用 `ggml_use_metal` 时，都会执行上述步骤 a 和 b。

3. 计算并返回图形渲染的时间（`wstate.t_encode_us` 和 `wstate.n_encode`）之差。


```cpp
#ifdef GGML_USE_METAL
        if (wstate.ctx_metal) {
            ggml_metal_set_n_cb     (wstate.ctx_metal, n_threads);
            ggml_metal_graph_compute(wstate.ctx_metal, gf);
        } else {
            ggml_graph_compute_helper(wstate.work_buffer, gf, n_threads, abort_callback, abort_callback_data);
        }
#else
        ggml_graph_compute_helper(wstate.work_buffer, gf, n_threads, abort_callback, abort_callback_data);
#endif
    }

    // ggml_graph_compute_with_ctx(ctx0, &gf, n_threads);

    wstate.t_encode_us += ggml_time_us() - t_start_us;
    wstate.n_encode++;

    return true;
}

```

This is a function definition for a neural network model that uses the GELU activation function. It appears to be a part of a larger neural network model that takes in input features and returns a logit output.

The function is first initializing the weights and biases for the different parts of the network. Then it performs a series of operations on the input features to compute the logits.

The first operation is a linear combination of the input features and the bias for the


```cpp
static struct ggml_cgraph * whisper_build_graph_decoder(
         whisper_context & wctx,
         whisper_state   & wstate,
         whisper_decoder & decoder,
     const whisper_token * tokens,
                   int   n_tokens,
                   int   n_past) {
    const auto & model   = wctx.model;
    const auto & hparams = model.hparams;

    auto & kv_self = decoder.kv_self;

    WHISPER_ASSERT(!!kv_self.ctx);

    const int n_ctx   = hparams.n_text_ctx;
    const int n_state = hparams.n_text_state;
    const int n_head  = hparams.n_text_head;
    const int n_layer = hparams.n_text_layer;

    const int N = n_tokens;
    const int M = wstate.exp_n_audio_ctx > 0 ? wstate.exp_n_audio_ctx : hparams.n_audio_ctx;

    //WHISPER_PRINT_DEBUG("%s: n_past = %d, N = %d, M = %d, n_ctx = %d\n", __func__, n_past, N, M, n_ctx);

    struct ggml_init_params params = {
        /*.mem_size   =*/ wstate.alloc_decode.meta.size(),
        /*.mem_buffer =*/ wstate.alloc_decode.meta.data(),
        /*.no_alloc   =*/ true,
    };

    struct ggml_context * ctx0 = ggml_init(params);

    ggml_cgraph * gf = ggml_new_graph_custom(ctx0, WHISPER_MAX_NODES, false);

    ggml_allocr * alloc = wstate.alloc_decode.alloc;

    struct ggml_tensor * embd = ggml_new_tensor_1d(ctx0, GGML_TYPE_I32, N);
    ggml_allocr_alloc(alloc, embd);

    if (!ggml_allocr_is_measure(alloc)) {
        memcpy(embd->data, tokens, N*ggml_element_size(embd));
    }

    struct ggml_tensor * position = ggml_new_tensor_1d(ctx0, GGML_TYPE_I32, N);
    ggml_allocr_alloc(alloc, position);

    if (!ggml_allocr_is_measure(alloc)) {
        for (int i = 0; i < N; ++i) {
            ((int32_t *) position->data)[i] = n_past + i;
        }
    }

    struct ggml_tensor * KQscale = ggml_new_tensor_1d(ctx0, GGML_TYPE_F32, 1);
    ggml_allocr_alloc(alloc, KQscale);

    if (!ggml_allocr_is_measure(alloc)) {
        ggml_set_f32(KQscale, pow(float(n_state)/n_head, -0.25));
    }

    // token encoding + position encoding
    struct ggml_tensor * cur =
        ggml_add(ctx0,
                ggml_get_rows(ctx0, model.d_te, embd),
                ggml_get_rows(ctx0, model.d_pe, position));

    struct ggml_tensor * inpL = cur;

    for (int il = 0; il < n_layer; ++il) {
        const auto & layer = model.layers_decoder[il];

        // norm
        {
            cur = ggml_norm(ctx0, inpL, hparams.eps);

            // cur = ln_0_w*cur + ln_0_b
            cur = ggml_add(ctx0,
                    ggml_mul(ctx0,
                        cur,
                        layer.attn_ln_0_w),
                    layer.attn_ln_0_b);
        }

        // self-attention
        {
            struct ggml_tensor * Qcur = ggml_mul_mat(ctx0,
                    layer.attn_q_w,
                    cur);

            Qcur = ggml_add(ctx0,
                        Qcur,
                        layer.attn_q_b);

            Qcur = ggml_scale(ctx0, Qcur, KQscale);

            // note: no bias for Key
            struct ggml_tensor * Kcur = ggml_mul_mat(ctx0,
                    layer.attn_k_w,
                    cur);

            Kcur = ggml_scale(ctx0, Kcur, KQscale);

            // store key and value to memory
            {
                struct ggml_tensor * Vcur = ggml_mul_mat(ctx0,
                        layer.attn_v_w,
                        cur);

                Vcur = ggml_add(ctx0,
                            Vcur,
                            layer.attn_v_b);

                Vcur = ggml_transpose(ctx0, ggml_reshape_2d(ctx0, Vcur, n_state, N));

                struct ggml_tensor * k = ggml_view_1d(ctx0, kv_self.k, N*n_state, (ggml_element_size(kv_self.k)*n_state)*(il*n_ctx + n_past));
                struct ggml_tensor * v = ggml_view_2d(ctx0, kv_self.v, N, n_state,
                        (   n_ctx)*ggml_element_size(kv_self.v),
                        (il*n_ctx)*ggml_element_size(kv_self.v)*n_state + n_past*ggml_element_size(kv_self.v));

                ggml_build_forward_expand(gf, ggml_cpy(ctx0, Kcur, k));
                ggml_build_forward_expand(gf, ggml_cpy(ctx0, Vcur, v));
            }

            // ------

            struct ggml_tensor * Q =
                ggml_permute(ctx0,
                        ggml_reshape_3d(ctx0, Qcur, n_state/n_head, n_head, N),
                        0, 2, 1, 3);

            struct ggml_tensor * K =
                ggml_view_3d(ctx0, kv_self.k,
                        n_state/n_head, n_past + N, n_head,
                        ggml_element_size(kv_self.k)*n_state,
                        ggml_element_size(kv_self.k)*n_state/n_head,
                        ggml_element_size(kv_self.k)*n_state*n_ctx*il);

            // K * Q
            struct ggml_tensor * KQ = ggml_mul_mat(ctx0, K, Q);

            //struct ggml_tensor * KQ_scaled = ggml_scale(ctx0, KQ, KQ_scale);

            struct ggml_tensor * KQ_masked = ggml_diag_mask_inf(ctx0, KQ, n_past);

            struct ggml_tensor * KQ_soft_max = ggml_soft_max(ctx0, KQ_masked);

            struct ggml_tensor * V =
                ggml_view_3d(ctx0, kv_self.v,
                        n_past + N, n_state/n_head, n_head,
                        n_ctx*ggml_element_size(kv_self.v),
                        n_ctx*ggml_element_size(kv_self.v)*n_state/n_head,
                        il*n_ctx*ggml_element_size(kv_self.v)*n_state);

            struct ggml_tensor * KQV = ggml_mul_mat(ctx0, V, KQ_soft_max);

            struct ggml_tensor * KQV_merged = ggml_permute(ctx0, KQV, 0, 2, 1, 3);

            cur = ggml_cpy(ctx0,
                    KQV_merged,
                    ggml_new_tensor_2d(ctx0, GGML_TYPE_F32, n_state, N));
        }

        // projection
        {
            cur = ggml_mul_mat(ctx0,
                    layer.attn_ln_1_w,
                    cur);

            cur = ggml_add(ctx0,
                    cur,
                    layer.attn_ln_1_b);
        }

        // add the input
        struct ggml_tensor * inpCA = ggml_add(ctx0, cur, inpL);

        // norm
        {
            cur = ggml_norm(ctx0, inpCA, hparams.eps); // note: we use inpCA here

            // cur = ln_0_w*cur + ln_0_b
            cur = ggml_add(ctx0,
                    ggml_mul(ctx0,
                        cur,
                        layer.cross_attn_ln_0_w),
                    layer.cross_attn_ln_0_b);
        }

        // cross-attention
        {
            struct ggml_tensor * Qcur = ggml_mul_mat(ctx0,
                    layer.cross_attn_q_w,
                    cur);

            Qcur = ggml_add(ctx0,
                        Qcur,
                        layer.cross_attn_q_b);

            Qcur = ggml_scale(ctx0, Qcur, KQscale);

            // Kcross is already scaled
            struct ggml_tensor * Kcross =
                ggml_view_3d(ctx0, wstate.kv_cross.k,
                        n_state/n_head, M, n_head,
                        ggml_element_size(wstate.kv_cross.k)*n_state,
                        ggml_element_size(wstate.kv_cross.k)*n_state/n_head,
                        ggml_element_size(wstate.kv_cross.k)*n_state*M*il);

            //struct ggml_tensor * Vcross =
            //    ggml_reshape_3d(ctx0,
            //            ggml_view_1d(ctx0, wstate.kv_cross.v, M*n_state, il*M*ggml_element_size(wstate.kv_cross.v)*n_state),
            //            n_state/n_head, n_head, M);

            //struct ggml_tensor * V_trans =
            //    ggml_cpy(ctx0,
            //            ggml_permute(ctx0, Vcross, 1, 2, 0, 3),
            //            ggml_new_tensor_3d(ctx0, Vcross->type, M, n_state/n_head, n_head));

            struct ggml_tensor * V =
                ggml_view_3d(ctx0, wstate.kv_cross.v,
                        M, n_state/n_head, n_head,
                        M*ggml_element_size(wstate.kv_cross.v),
                        M*ggml_element_size(wstate.kv_cross.v)*n_state/n_head,
                        il*M*ggml_element_size(wstate.kv_cross.v)*n_state);

            // ------

            struct ggml_tensor * Q =
                ggml_permute(ctx0,
                        ggml_reshape_3d(ctx0, Qcur, n_state/n_head, n_head, N),
                        0, 2, 1, 3);

            // K * Q
            struct ggml_tensor * KQ = ggml_mul_mat(ctx0, Kcross, Q);

            //struct ggml_tensor * KQ_scaled =
            //    ggml_scale(ctx0,
            //            KQ,
            //            ggml_new_f32(ctx0, 1.0f/sqrt(float(n_state)/n_head))
            //            );

            // no masking for cross-attention
            //struct ggml_tensor * KQ_masked = ggml_diag_mask_inf(ctx0, KQ_scaled, n_past);

            struct ggml_tensor * KQ_soft_max = ggml_soft_max(ctx0, KQ);

            struct ggml_tensor * KQV = ggml_mul_mat(ctx0, V, KQ_soft_max);

            struct ggml_tensor * KQV_merged = ggml_permute(ctx0, KQV, 0, 2, 1, 3);

            // cur = KQV_merged.contiguous().view(n_state, N)
            cur = ggml_cpy(ctx0,
                    KQV_merged,
                    ggml_new_tensor_2d(ctx0, GGML_TYPE_F32, n_state, N));
        }

        // projection
        {
            cur = ggml_mul_mat(ctx0,
                    layer.cross_attn_ln_1_w,
                    cur);

            cur = ggml_add(ctx0,
                    cur,
                    layer.cross_attn_ln_1_b);
        }

        // add the input
        cur = ggml_add(ctx0, cur, inpCA);

        struct ggml_tensor * inpFF = cur;

        // feed-forward network
        {
            // norm
            {
                cur = ggml_norm(ctx0, inpFF, hparams.eps);

                // cur = mlp_ln_w*cur + mlp_ln_b
                cur = ggml_add(ctx0,
                        ggml_mul(ctx0,
                            cur,
                            layer.mlp_ln_w),
                        layer.mlp_ln_b);
            }

            // fully connected
            cur = ggml_mul_mat(ctx0,
                    layer.mlp_0_w,
                    cur);

            cur = ggml_add(ctx0,
                    cur,
                    layer.mlp_0_b);

            // GELU activation
            cur = ggml_gelu(ctx0, cur);

            // projection
            cur = ggml_mul_mat(ctx0,
                    layer.mlp_1_w,
                    cur);

            cur = ggml_add(ctx0,
                    cur,
                    layer.mlp_1_b);
        }

        inpL = ggml_add(ctx0, cur, inpFF);
    }

    cur = inpL;

    // norm
    {
        cur = ggml_norm(ctx0, cur, hparams.eps);

        cur = ggml_add(ctx0,
                ggml_mul(ctx0,
                    cur,
                    model.d_ln_w),
                model.d_ln_b);
    }

    // compute logits only for the last token
    // comment this line to compute logits for all N tokens
    // might be useful in the future
    cur = ggml_view_2d(ctx0, cur, cur->ne[0], 1, cur->nb[1], (cur->ne[1] - 1)*cur->nb[1]);

    struct ggml_tensor * logits = ggml_mul_mat(ctx0, model.d_te, cur);

    ggml_build_forward_expand(gf, logits);

    ggml_free(ctx0);

    return gf;
}

```

这段代码是一个名为“evaluate_decoder.c”的 C 语言函数，它通过给定的文本提示（通过 "text" 参数）和音频特征，计算出下一个单词的概率。这段代码的作用是作为模型的解码部分，在给定文本提示和音频特征的情况下，输出模型的输出。

代码的主要组成部分包括：

1. 定义了一个名为“whisper_decode_internal”的函数，它接受一个 whisper 上下文（whisper_context & wctx）、一个 whisper 状态（whisper_state & wstate）、一个解码器（whisper_decoder & decoder）和一个通用的异常回调函数（whisper_abort_callback）。

2. 在函数内部，定义了一些整数变量，分别表示文本 prompt 的大小（n_tokens）、残缺的上下文块（n_past）、线程数（n_threads）和存储模型参数的内存指针。

3. 在函数内部，定义了一个名为“logits_out”的结构体变量，用于存储计算得到的下一行文本的概率分布。

4. 在函数内部，定义了一个名为“whisper_build_graph_decoder”的函数，它接受一个解码器（whisper_decoder & decoder）、一个上下文块（whisper_context & wctx）和一个残缺的上下文块（n_past）。这个函数的作用是构建一个用于解码的图形化上下文，并返回这个上下文。

5. 在函数内部，定义了一个名为“ggml_allocr_alloc_graph”的函数，它接受一个上下文对象（whisper_context & wctx）和一个上下文块（whisper_context & wctx）、一个解码器（whisper_decoder & decoder）和一个残缺的上下文块（n_past）。这个函数的作用是分配解码器的参数，并返回这个上下文的指针。

6. 在函数内部，定义了一个名为“ggml_allocr_reset”的函数，它接受一个上下文对象（whisper_context & wctx）。这个函数的作用是重置解码器的参数，并将已经计算过的部分清除。

7. 在函数内部，定义了一个名为“ggml_cgraph_forward”的函数，它接受一个上下文对象（whisper_context & wctx）、一个解码器（whisper_decoder & decoder）和一个残缺的上下文块（n_past）。这个函数的作用是计算解码器应该返回的值，并将计算结果存储在 logits 变量中。

8. 在函数内部，定义了一个名为“whisper_abort_callback”的函数，它接受一个上下文对象（whisper_context & wctx）和一个通用的异常回调函数（void *abort_callback_data）。这个函数的作用是在解码器无法继续处理数据时回调异常，并将异常信息传递给回调函数。

9. 在函数内部，定义了一个名为“evaluate_decoder”的函数，它接受一个解码器（whisper_decoder & decoder）、一个上下文对象（whisper_context & wctx）和一个残缺的上下文块（n_past）。这个函数的作用是解码数据并处理残缺的上下文，然后将解码结果存储在 logits 变量中。


```cpp
// evaluate the decoder
//
// given text prompt + audio features -> computes the logits for the next token
//
//   - model:      the model
//   - n_threads:  number of threads to use
//   - tokens:     text prompt
//   - n_tokens:   number of tokens in the prompt
//   - n_past:     number of past tokens to prefix the prompt with
//
static bool whisper_decode_internal(
        whisper_context & wctx,
          whisper_state & wstate,
        whisper_decoder & decoder,
    const whisper_token * tokens,
              const int   n_tokens,
              const int   n_past,
              const int   n_threads,
 whisper_abort_callback   abort_callback,
                   void * abort_callback_data) {
    const int64_t t_start_us = ggml_time_us();

    const auto & model   = wctx.model;
    const auto & hparams = model.hparams;

    const int n_vocab = hparams.n_vocab;

    auto & logits_out = wstate.logits;

    struct ggml_tensor * logits;

    // decoder
    {
        auto & alloc = wstate.alloc_decode.alloc;

        ggml_allocr_reset(alloc);

        ggml_cgraph * gf = whisper_build_graph_decoder(wctx, wstate, decoder, tokens, n_tokens, n_past);

        ggml_allocr_alloc_graph(alloc, gf);

        logits = gf->nodes[gf->n_nodes - 1];

```

This function appears to be used to compute the forward pass of a graph neural network and return the computed logits along with the number of weights and biases. The function takes as input a graph represented in either包容ator or partner format, as well as a number of threads to use for computation, an abort callback function, and an optional callback function data.
It appears to be using theggml\_graph\_compute\_helper function for the computation of the graph.
It also appears to be returning the logits of all N tokens, as well as the number of weights and biases for each token.
It also appears to be handling the case where the input is a single token, and using thet\_decode\_us and n\_decode variables to keep track of the number of tokens and the current position respectively.
It also appears to be using the wstate.t\_decode\_us and wstate.n\_decode variables to keep track of the time passed and the number of tokens that are decoded, respectively.


```cpp
#ifdef GGML_USE_METAL
        if (wstate.ctx_metal) {
            ggml_metal_set_n_cb     (wstate.ctx_metal, n_threads);
            ggml_metal_graph_compute(wstate.ctx_metal, gf);
        } else {
            ggml_graph_compute_helper(wstate.work_buffer, gf, n_threads, abort_callback, abort_callback_data);
        }
#else
        ggml_graph_compute_helper(wstate.work_buffer, gf, n_threads, abort_callback, abort_callback_data);
#endif
    }

    // extract logits for all N tokens
    //logits_out.resize(n_tokens*n_vocab);
    //memcpy(logits_out.data(), ggml_get_data(logits), sizeof(float)*n_tokens*n_vocab);

    // extract logits only for the last token
    logits_out.resize(n_vocab);
    memcpy(logits_out.data(), ggml_get_data(logits), sizeof(float)*n_vocab);

    if (n_tokens > 1) {
        //printf("%s: used_mem = %f MB, %f MB, %f MB %f MB %f MB\n", __func__,
        //        ggml_used_mem(ctx0)/1024.0/1024.0,
        //        wstate.get_buf_max_mem(0)/1024.0/1024.0,
        //        wstate.get_buf_max_mem(1)/1024.0/1024.0,
        //        wstate.get_buf_max_mem(2)/1024.0/1024.0,
        //        wstate.get_buf_max_mem(3)/1024.0/1024.0);
    }

    if (n_tokens == 1) {
        wstate.t_decode_us += ggml_time_us() - t_start_us;
        wstate.n_decode++;
    } else {
        wstate.t_prompt_us += ggml_time_us() - t_start_us;
        wstate.n_prompt++;
    }

    return true;
}


```

这段代码是一个C++语言中的函数，名为`to_timestamp`。它接受一个整数参数`t`，表示时间戳，以及一个布尔参数`commah`，表示是否以逗号分隔时间戳。函数返回一个字符串，表示从`t`时刻开始经过的毫秒数。

函数内部首先将`t`转换为毫秒数，然后分别计算出分钟、秒钟和微秒。接着将小时、分钟和微秒转换为字符，并加入'%s'格式化字符串中。最后，将格式化后的字符串返回。

这段代码主要用于将一个时间戳（如系统时间或预先定义的日期时间）转换为字符串格式，方便用户进行展示或处理。


```cpp
//  500 -> 00:05.000
// 6000 -> 01:00.000
static std::string to_timestamp(int64_t t, bool comma = false) {
    int64_t msec = t * 10;
    int64_t hr = msec / (1000 * 60 * 60);
    msec = msec - hr * (1000 * 60 * 60);
    int64_t min = msec / (1000 * 60);
    msec = msec - min * (1000 * 60);
    int64_t sec = msec / 1000;
    msec = msec - sec * 1000;

    char buf[32];
    snprintf(buf, sizeof(buf), "%02d:%02d:%02d%s%03d", (int) hr, (int) min, (int) sec, comma ? "," : ".", (int) msec);

    return std::string(buf);
}

```

这段代码定义了一个名为 SIN_COS_N_COUNT 的宏，表示一个包含 SIN 和 COS 函数值的数组长度。这个数组长度是固定的，并且使用 precomputed_values 函数来填充。

接下来是两个静态函数 fill_sin_cos_table 和 sin_vals_init_uint8。fill_sin_cos_table 函数用于初始化 sin 和 cos 函数值，以准备在 FFT 中使用。这个函数在内部分享，所以外部调用时不需要传递参数。

sin_vals_init_uint8 函数用于初始化一个包含 8 个整数的数组，这个数组将用于存储 sin 和 cos 函数的值。这个函数在内部分享，所以外部调用时不需要传递参数。

这两个函数的实现相对较为复杂，但它们在实现 FFT 的过程中非常重要，因为它们负责计算数组长度为 SIN_COS_N_COUNT 的函数值，这是 FFT 能够高效运行的关键。


```cpp
#define SIN_COS_N_COUNT WHISPER_N_FFT
static float sin_vals[SIN_COS_N_COUNT];
static float cos_vals[SIN_COS_N_COUNT];

// In FFT, we frequently use sine and cosine operations with the same values.
// We can use precalculated values to speed up the process.
static void fill_sin_cos_table() {
    static bool is_filled = false;
    if (is_filled) return;
    for (int i = 0; i < SIN_COS_N_COUNT; i++) {
        double theta = (2*M_PI*i)/SIN_COS_N_COUNT;
        sin_vals[i] = sinf(theta);
        cos_vals[i] = cosf(theta);
    }
    is_filled = true;
}

```

此代码是一个基于FFT算法的Discrete Fourier Transform（DFT）实现。它的输入是一个实数向量`in`，输出是一个相同大小的复数向量`out`。

具体来说，这个DFT实现包括了以下步骤：

1. 根据输入向量`in`的大小`N`，创建一个大小为`N*2`的复数向量`out`。
2. 对于输入向量`in`中的每个元素，执行以下操作：
a. 计算一个大小为`SIN_COS_N_COUNT`的整数倍数，并且该数值与输入向量中的索引`k`相对应的整数部分，记为`cos_vals[k]`。
b. 计算一个大小为`SIN_COS_N_COUNT`的整数倍数，并且该数值与输入向量中的索引`k`相对应的整数部分，记为`sin_vals[k]`。
c. 执行插值傅里叶变换，将输入向量中的每个点的实部和虚部乘以相应的振幅和相位，并且根据插值因子`cos_vals[k]`和`sin_vals[k]`对结果进行加权平均。
3. 将计算得到的`cos_vals[k]`和`sin_vals[k]`之和作为输出向量中的第`k`个元素。
4. 最终，输出向量中的元素为：`out[0]:= re, out[1]:= im`，其中`re`和`im`是输入向量`in`中的对应元素的实部和虚部。


```cpp
// naive Discrete Fourier Transform
// input is real-valued
// output is complex-valued
static void dft(const std::vector<float> & in, std::vector<float> & out) {
    int N = in.size();

    out.resize(N*2);
    const int sin_cos_step = SIN_COS_N_COUNT / N;

    for (int k = 0; k < N; k++) {
        float re = 0;
        float im = 0;

        for (int n = 0; n < N; n++) {
            int idx = (k * n * sin_cos_step) % (SIN_COS_N_COUNT); // t = 2*M_PI*k*n/N
            re += in[n]*cos_vals[idx]; // cos(t)
            im -= in[n]*sin_vals[idx]; // sin(t)
        }

        out[k*2 + 0] = re;
        out[k*2 + 1] = im;
    }
}

```

This function appears to be performing a Fast Fourier Transform (FFT) on a Discrete Fourier Transform (DFT) of a stream of floating-point numbers. It takes a stream of DFT values as input and returns a stream of重排后的DFT values as output. The function first checks whether the input data is odd or even. If it is odd, the function performs a simple reversible FFT on the even half of the input data and returns the result. If the input data is even, the function performs a more complex reversible FFT on the even half of the input data and returns the result. The function uses the FFT-recursive algorithm to compute the FFT of the even and odd parts of the input data in parallel.


```cpp
// Cooley-Tukey FFT
// poor man's implementation - use something better
// input is real-valued
// output is complex-valued
static void fft(const std::vector<float> & in, std::vector<float> & out) {
    out.resize(in.size()*2);

    int N = in.size();

    if (N == 1) {
        out[0] = in[0];
        out[1] = 0;
        return;
    }

    if (N%2 == 1) {
        dft(in, out);
        return;
    }

    std::vector<float> even;
    std::vector<float> odd;

    even.reserve(N/2);
    odd.reserve(N/2);

    for (int i = 0; i < N; i++) {
        if (i % 2 == 0) {
            even.push_back(in[i]);
        } else {
            odd.push_back(in[i]);
        }
    }

    std::vector<float> even_fft;
    std::vector<float> odd_fft;

    fft(even, even_fft);
    fft(odd, odd_fft);

    const int sin_cos_step = SIN_COS_N_COUNT / N;
    for (int k = 0; k < N/2; k++) {
        int idx = k * sin_cos_step; // t = 2*M_PI*k/N
        float re = cos_vals[idx]; // cos(t)
        float im = -sin_vals[idx]; // sin(t)

        float re_odd = odd_fft[2*k + 0];
        float im_odd = odd_fft[2*k + 1];

        out[2*k + 0] = even_fft[2*k + 0] + re*re_odd - im*im_odd;
        out[2*k + 1] = even_fft[2*k + 1] + re*im_odd + im*re_odd;

        out[2*(k + N/2) + 0] = even_fft[2*k + 0] - re*re_odd + im*im_odd;
        out[2*(k + N/2) + 1] = even_fft[2*k + 1] - re*im_odd - im*re_odd;
    }
}

```

该代码是一个Hann window simulation的实现。Hann window是一种在二维方向上展开的二维卓比克夫Window。它的窗控参数为 periodic = false，意味着 window会在每个时间步长结束时进行缩放，以避免出现震荡和饱和。

具体来说，该代码实现了一个名为 "hann_window" 的函数，它接受一个整数参数 length，表示窗体的高度，以及一个向量参数 output，表示输出结果。函数首先检查输出是否已经准备好，如果没有，则将输出向量长度扩展到与长度相同。

接着，函数通过 two_pi 函数计算 window 中心处的角度偏移量 offset。如果 periodic 参数为 false，则 offset 将为 0。

然后，函数使用一个循环来遍历 window 的所有位置。对于每个位置，函数使用 arcsin 函数计算一个时间步长为 2 的周期内 cosf(2 * M_PI * i / length) 的值，然后将其乘以 0.5，得到该位置的输出值。

最后，函数使用 return true; 语句来表示本函数是否成功，成功则返回 true，否则返回 false。


```cpp
static bool hann_window(int length, bool periodic, std::vector<float> & output) {
    if (output.size() < static_cast<size_t>(length)) {
        output.resize(length);
    }
    int offset = -1;
    if (periodic) {
        offset = 0;
    }
    for (int i = 0; i < length; i++) {
        output[i] = 0.5*(1.0 - cosf((2.0*M_PI*i)/(length + offset)));
    }

    return true;
}

```

This is a C++ implementation of a function called `run_fft_on_gpu`, which performs a Fast Fourier Transform (FFT) on a GPU and returns the results. The implementation uses the cuDNN library for performing the FFT calculations.

The function takes a two-dimensional array `fft_out` as input, which represents the intermediate results of the FFT calculation, and a two-dimensional array `mel` as input, which represents the input mel spectrogram. The function performs an inner-product processing on the `fft_out` array and returns the mel spectrogram.

The function uses a combination of unrolling and rolling loops to process the `fft_out` array efficiently. The unrolling loop is a suggestion from the GitHub user 'GHuser' and is used to improve the performance of the rolling loops.

The function also includes a handling of the remaining `n_fft` rollover issue. This is done by calculating the sum of the element-wise product of the `fft_out` array with the `filters.data` array, which has the same size as `fft_out`, but with a rollover factor of `n_fft`. The result is then set to the `mel.data` element at index `i` within the `mel` array.

If the `i`-th element in the `mel` array is NaN or it has a value of zero, the value of the corresponding element in the `fft_out` array is set to zero to avoid any issues with the resulting mel spectrogram.


```cpp
static void log_mel_spectrogram_worker_thread(int ith, const std::vector<float> & hann, const std::vector<float> & samples,
                                              int n_samples, int frame_size, int frame_step, int n_threads,
                                              const whisper_filters & filters, whisper_mel & mel) {
    std::vector<float> fft_in(frame_size, 0.0);
    std::vector<float> fft_out(2 * frame_step);
    // make sure n_fft == 1 + (WHISPER_N_FFT / 2), bin_0 to bin_nyquist
    int n_fft = 1 + (frame_size / 2);
    int i = ith;

    // calculate FFT only when fft_in are not all zero
    for (; i < std::min(n_samples / frame_step + 1, mel.n_len); i += n_threads) {
        const int offset = i * frame_step;

        // apply Hanning window (~10% faster)
        for (int j = 0; j < std::min(frame_size, n_samples - offset); j++) {
            fft_in[j] = hann[j] * samples[offset + j];
        }
        // fill the rest with zeros
        if (n_samples - offset < frame_size) {
            std::fill(fft_in.begin() + (n_samples - offset), fft_in.end(), 0.0);
        }

        // FFT
        fft(fft_in, fft_out);

        // Calculate modulus^2 of complex numbers
        // Use pow(fft_out[2 * j + 0], 2) + pow(fft_out[2 * j + 1], 2) causes inference quality problem? Interesting.
        for (int j = 0; j < frame_size; j++) {
            fft_out[j] = (fft_out[2 * j + 0] * fft_out[2 * j + 0] + fft_out[2 * j + 1] * fft_out[2 * j + 1]);
        }

        // mel spectrogram
        for (int j = 0; j < mel.n_mel; j++) {
            double sum = 0.0;

            // unroll loop (suggested by GH user @lunixbochs)
            int k = 0;
            for (k = 0; k < n_fft - 3; k += 4) {
                sum +=
                        fft_out[k + 0] * filters.data[j * n_fft + k + 0] +
                        fft_out[k + 1] * filters.data[j * n_fft + k + 1] +
                        fft_out[k + 2] * filters.data[j * n_fft + k + 2] +
                        fft_out[k + 3] * filters.data[j * n_fft + k + 3];
            }

            // handle n_fft remainder
            for (; k < n_fft; k++) {
                sum += fft_out[k] * filters.data[j * n_fft + k];
            }

            sum = log10(std::max(sum, 1e-10));

            mel.data[j * mel.n_len + i] = sum;
        }
    }

    // Otherwise fft_out are all zero
    double sum = log10(1e-10);
    for (; i < mel.n_len; i += n_threads) {
        for (int j = 0; j < mel.n_mel; j++) {
            mel.data[j * mel.n_len + i] = sum;
        }
    }
}

```

This is a C++ implementation of the log_mel_spectrogram() function, which is a worker function for the log_mel_spectrogram_worker_thread() function. The worker function takes in a hypothesized spectrogram (samples), samples the data according to the specified rate (e.g. 44100), and appends the sampled data to the hypothesized spectrogram. It also performs some additional processing, such as clipping and normalization of the data.

The worker function has an implementation of the main thread and a number of helper or worker threads. The main thread performs the following tasks:

1. Extracts the spectrogram data from the input file.
2. Clamps and normalizes the data.
3. Increments the timestamp of the extracted data.
4. Writes the normalized spectrogram data to a JSON file.
5. Allocates memory for the worker threads.
6. Creates the worker threads.
7. Starts the worker threads.
8. waits for the worker threads to finish.
9. Retrieves the processed spectrogram data from the main thread.

The helper or worker threads are responsible for performing the following tasks:

1. Sampling the data at the specified rate.
2. Appends the sampled data to the hypothesized spectrogram.
3. Performs some additional processing, such as clipping and normalization of the data.
4. Joins the worker threads.

Note that the code assumes that the input file is a binary file called "input_audio.wav" and that it contains the spectrogram data sampled at 44100Hz. Also, it assumes that the output JSON file is called "log_mel_spectrogram.json" and is written to the same directory as the executable.


```cpp
// ref: https://github.com/openai/whisper/blob/main/whisper/audio.py#L110-L157
static bool log_mel_spectrogram(
              whisper_state & wstate,
              const float * samples,
              const int   n_samples,
              const int   /*sample_rate*/,
              const int   frame_size,
              const int   frame_step,
              const int   n_mel,
              const int   n_threads,
              const whisper_filters & filters,
              const bool   debug,
              whisper_mel & mel) {
    const int64_t t_start_us = ggml_time_us();

    // Hanning window (Use cosf to eliminate difference)
    // ref: https://pytorch.org/docs/stable/generated/torch.hann_window.html
    // ref: https://github.com/openai/whisper/blob/main/whisper/audio.py#L147
    std::vector<float> hann;
    hann_window(frame_size, true, hann);


    // Calculate the length of padding
    int64_t stage_1_pad = WHISPER_SAMPLE_RATE * 30;
    int64_t stage_2_pad = frame_size / 2;

    // Initialize a vector and copy data from C array to it.
    std::vector<float> samples_padded;
    samples_padded.resize(n_samples + stage_1_pad + stage_2_pad * 2);
    std::copy(samples, samples + n_samples, samples_padded.begin() + stage_2_pad);

    // pad 30 seconds of zeros at the end of audio (480,000 samples) + reflective pad 200 samples at the end of audio
    std::fill(samples_padded.begin() + n_samples + stage_2_pad, samples_padded.begin() + n_samples + stage_1_pad + 2 * stage_2_pad, 0);

    // reflective pad 200 samples at the beginning of audio
    std::reverse_copy(samples + 1, samples + 1 + stage_2_pad, samples_padded.begin());

    mel.n_mel     = n_mel;
    // https://github.com/pytorch/pytorch/blob/main/aten/src/ATen/native/SpectralOps.cpp#L936
    // Calculate number of frames + remove the last frame
    mel.n_len     = (samples_padded.size() - frame_size) / frame_step;
    // Calculate semi-padded sample length to ensure compatibility
    mel.n_len_org = 1 + (n_samples + stage_2_pad - frame_size) / frame_step;
    mel.data.resize(mel.n_mel * mel.n_len);


    {
        std::vector<std::thread> workers(n_threads - 1);
        for (int iw = 0; iw < n_threads - 1; ++iw) {
            workers[iw] = std::thread(
                    log_mel_spectrogram_worker_thread, iw + 1, std::cref(hann), samples_padded,
                    n_samples + stage_2_pad, frame_size, frame_step, n_threads,
                    std::cref(filters), std::ref(mel));
        }

        // main thread
        log_mel_spectrogram_worker_thread(0, hann, samples_padded, n_samples + stage_2_pad, frame_size, frame_step, n_threads, filters, mel);

        for (int iw = 0; iw < n_threads - 1; ++iw) {
            workers[iw].join();
        }
    }

    // clamping and normalization
    double mmax = -1e20;
    for (int i = 0; i < mel.n_mel*mel.n_len; i++) {
        if (mel.data[i] > mmax) {
            mmax = mel.data[i];
        }
    }

    mmax -= 8.0;

    for (int i = 0; i < mel.n_mel*mel.n_len; i++) {
        if (mel.data[i] < mmax) {
            mel.data[i] = mmax;
        }

        mel.data[i] = (mel.data[i] + 4.0)/4.0;
    }

    wstate.t_mel_us += ggml_time_us() - t_start_us;

    // Dump log_mel_spectrogram
    if (debug) {
        std::ofstream outFile("log_mel_spectrogram.json");
        outFile << "[";
        for (uint64_t i = 0; i < mel.data.size() - 1; i++) {
            outFile << mel.data[i] << ", ";
        }
        outFile << mel.data[mel.data.size() - 1] << "]";
        outFile.close();
    }

    return true;
}

```

This is a C++ implementation of the tokenize() function for the Whisper Vocab language. The function takes a Whisper Vocab object and a string of text as input and returns a vector of token IDs.

The tokenize() function splits the input text into words by searching for matches in the regular expression `R"('s|'t|'re|'ve|'m|'ll|'d| ?[[:alpha:]]+| ?[[:digit:]]+| ?[^\s[:alpha:][:digit:]]+|\s+(?!\S)|\s+)` against the input string. The resulting words are then stored in a vector.

The function then searches for the longest tokens that form the words by keeping track of the current word and its index. When a complete word is found, the function adds it to the tokens vector and continues searching for the next word. If no complete word is found, the function logs an error and increments the current index.

Note that this implementation does not handle the case where the input text contains punctuation or special characters that are not part of the token.


```cpp
// split text into tokens
//
// ref: https://github.com/openai/gpt-2/blob/a74da5d99abaaba920de8131d64da2862a8f213b/src/encoder.py#L53
//
// Regex (Python):
// r"""'s|'t|'re|'ve|'m|'ll|'d| ?\p{L}+| ?\p{N}+| ?[^\s\p{L}\p{N}]+|\s+(?!\S)|\s+"""
//
// Regex (C++):
// R"('s|'t|'re|'ve|'m|'ll|'d| ?[[:alpha:]]+| ?[[:digit:]]+| ?[^\s[:alpha:][:digit:]]+|\s+(?!\S)|\s+)"
//
static std::vector<whisper_vocab::id> tokenize(const whisper_vocab & vocab, const std::string & text) {
    std::vector<std::string> words;

    // first split the text into words
    {
        std::string str = text;
        std::string pat = R"('s|'t|'re|'ve|'m|'ll|'d| ?[[:alpha:]]+| ?[[:digit:]]+| ?[^\s[:alpha:][:digit:]]+|\s+(?!\S)|\s+)";

        std::regex re(pat);
        std::smatch m;

        while (std::regex_search(str, m, re)) {
            for (auto x : m) {
                words.push_back(x);
            }
            str = m.suffix();
        }
    }

    // find the longest tokens that form the words:
    std::vector<whisper_vocab::id> tokens;
    for (const auto & word : words) {
        if (word.empty()) continue;

        int i = 0;
        int n = word.size();
        while (i < n) {
            int j = n;
            bool found = false;
            while (j > i) {
                auto sub = word.substr(i, j-i);
                auto it = vocab.token_to_id.find(sub);
                if (it != vocab.token_to_id.end()) {
                    tokens.push_back(it->second);
                    i = j;
                    found = true;
                    break;
                }
                --j;
            }
            if (!found) {
                log("unknown token\n");
                ++i;
            }
        }
    }

    return tokens;
}

```

这段代码是一个C++接口实现，其中包括一个函数`whisper_get_coreml_path_encoder`，用于获取CoreML编码器模型的路径。

函数的实现主要涉及以下几个步骤：

1. 通过`rfind`函数查找给定的二进制文件路径中最后一个`.`的位置，如果该位置不为空，则将该位置之前的字符串作为参数继续查找，直到找到文件路径的最后一个`.`的位置。

2. 在文件路径中查找与`-`字符有关的子串，如果找到该子串，则记录下其位置和子串长度。

3. 在文件路径中查找以`_`字符开头的与`-qx_x`匹配的字符串，如果找到该字符串，则将文件路径中的`-`子串替换为`_`子串，并将替换后的字符串与`encoder.mlmodelc`拼接。

4. 将生成的路径字符串与`encoder.mlmodelc`拼接，并返回该路径。

该函数的作用是获取一个CoreML编码器模型的二进制文件路径，根据一些规则对路径进行处理，最终返回处理后的路径。使用该函数，用户可以通过传递CoreML编码器模型的二进制文件路径，来获取该模型的CoreML编码器模型。


```cpp
//
// interface implementation
//

#ifdef WHISPER_USE_COREML
// replace .bin with -encoder.mlmodelc
static std::string whisper_get_coreml_path_encoder(std::string path_bin) {
    auto pos = path_bin.rfind('.');
    if (pos != std::string::npos) {
        path_bin = path_bin.substr(0, pos);
    }

    // match "-qx_x"
    pos = path_bin.rfind('-');
    if (pos != std::string::npos) {
        auto sub = path_bin.substr(pos);
        if (sub.size() == 5 && sub[1] == 'q' && sub[3] == '_') {
            path_bin = path_bin.substr(0, pos);
        }
    }

    path_bin += "-encoder.mlmodelc";

    return path_bin;
}
```

这段代码是一个C++代码中的一个预处理指令，它通过一个文件路径参数来定义了一个函数 `whisper_openvino_get_path_encoder`，用于获取一个名为 "openvino_encoder.bin" 的文件的实际路径。

具体来说，这段代码的作用是：

1. 如果已经定义了 "WHISPER_USE_OPENVINO"，那么将函数 `whisper_openvino_get_path_encoder` 的实现替换为以下内容：

```cppcpp
static std::string whisper_openvino_get_path_encoder(std::string path_bin) {
   auto pos = path_bin.rfind('.');
   if (pos != std::string::npos) {
       path_bin = path_bin.substr(0, pos);
   }

   path_bin += "-encoder-openvino.xml";

   return path_bin;
}
```

2. 如果还没有定义 "WHISPER_USE_OPENVINO"，那么将函数 `whisper_openvino_get_path_encoder` 的实现替换为以下内容：

```cppcpp
static std::string whisper_openvino_get_path_encoder(std::string path_bin) {
   static std::string encoder_path = "/path/to/openvino_encoder.xml";

   if (path_bin == "-encoder-openvino.xml") {
       return encoder_path;
   }

   path_bin += "-encoder-openvino.xml";

   return path_bin;
}
```

其中，`/path/to/openvino_encoder.xml` 是定义 `whisper_openvino_get_path_encoder` 时使用的文件路径。

总之，这段代码定义了一个函数 `whisper_openvino_get_path_encoder`，用于获取一个名为 "openvino_encoder.bin" 的文件的实际路径，即使 "WHISPER_USE_OPENVINO" 已经被定义。如果没有定义 "WHISPER_USE_OPENVINO"，则使用默认的文件路径。


```cpp
#endif

#ifdef WHISPER_USE_OPENVINO
// replace .bin with-encoder-openvino.xml
static std::string whisper_openvino_get_path_encoder(std::string path_bin) {
    auto pos = path_bin.rfind('.');
    if (pos != std::string::npos) {
        path_bin = path_bin.substr(0, pos);
    }

    path_bin += "-encoder-openvino.xml";

    return path_bin;
}

```

This code appears to be for training an image classification model using the Whisper algorithm. It includes a function for initializing the state of the Whisper algorithm and a fill function for initializing the sinus table and self-attention cache. Additionally, it includes a logging function for logging the size of the Whisper state.

The `whisper_init_state()` function takes a `whisper_context * ctx` pointer and initializes the state of the Whisper algorithm by calling the corresponding initialization functions for the self-attention cache and cross-attention cache. The logging function is also called to log the size of the Whisper state.

It is important to note that the `whisper_state` struct should have defined its member variables, such as the self-attention cache and cross-attention cache, before calling `whisper_init_state()` to ensure that the state is properly initialized.


```cpp
static std::string whisper_openvino_get_path_cache(std::string path_bin) {
    auto pos = path_bin.rfind('.');
    if (pos != std::string::npos) {
        path_bin = path_bin.substr(0, pos);
    }

    path_bin += "-encoder-openvino-cache";

    return path_bin;
}
#endif

struct whisper_state * whisper_init_state(whisper_context * ctx) {
    fill_sin_cos_table();
    whisper_state * state = new whisper_state;

    if (!kv_cache_init(ctx->model.hparams, state->decoders[0].kv_self, ctx->itype, ctx->model.hparams.n_text_ctx)) {
        log("%s: kv_cache_init() failed for self-attention cache\n", __func__);
        delete state;
        return nullptr;
    }

    {
        const size_t memory_size = ggml_nbytes(state->decoders[0].kv_self.k) + ggml_nbytes(state->decoders[0].kv_self.v);
        log("%s: kv self size  = %7.2f MB\n", __func__, memory_size / 1024.0 / 1024.0);
    }

    if (!kv_cache_init(ctx->model.hparams, state->kv_cross, ctx->itype, ctx->model.hparams.n_audio_ctx)) {
        log("%s: kv_cache_init() failed for cross-attention cache\n", __func__);
        delete state;
        return nullptr;
    }

    {
        const size_t memory_size = ggml_nbytes(state->kv_cross.k) + ggml_nbytes(state->kv_cross.v);
        log("%s: kv cross size = %7.2f MB\n", __func__, memory_size / 1024.0 / 1024.0);
    }

```

这段代码是一个C++程序，旨在加载一个Core ML模型，并检查其是否成功加载。它包含以下几部分：

1. 使用whisper_get_coreml_path_encoder函数获取Core ML路径编码器。
2. 加载Core ML模型，并输出成功加载的消息。
3. 如果加载失败，则输出错误消息，并尽可能地删除堆栈中的状态变量，以避免内存泄漏。
4. 如果加载成功，则输出成功消息。

这里使用了whisper_coreml_init函数来加载Core ML模型。如果该函数成功加载模型，则使用whisper_coreml_init函数返回逻辑成功，否则会输出错误消息并尽可能地删除堆栈中的状态变量。


```cpp
#ifdef WHISPER_USE_COREML
    const auto path_coreml = whisper_get_coreml_path_encoder(ctx->path_model);

    log("%s: loading Core ML model from '%s'\n", __func__, path_coreml.c_str());
    log("%s: first run on a device may take a while ...\n", __func__);

    state->ctx_coreml = whisper_coreml_init(path_coreml.c_str());
    if (!state->ctx_coreml) {
        log("%s: failed to load Core ML model from '%s'\n", __func__, path_coreml.c_str());
#ifndef WHISPER_COREML_ALLOW_FALLBACK
        delete state;
        return nullptr;
#endif
    } else {
        log("%s: Core ML model loaded\n", __func__);
    }
```

It looks like you are trying to compute the size of the buffer in the different components of the AllocInWhisper graph.

The `whisper_allocr_size` function appears to calculate the size of the entire graph, as a `printf` statement at the end of the function specifies this as the format.

The graph appears to have several components, including an encoder, a cross encoder, a decoder, and a default encoder. It is using the `whisper_allocr_graph_init` function to initialize these components in the graph, and the `whisper_build_graph_encoder`, `whisper_build_graph_cross`, and `whisper_build_graph_decoder` functions to build the graph.

It looks like the `%7.2f` format specifier is used to display the size in a human-readable format, with up to 7 decimal places.

It is important to note that the actual size of the graph may vary depending on the specific implementation and the size of the data being processed.


```cpp
#endif

    state->logits.reserve(ctx->vocab.n_vocab * ctx->model.hparams.n_text_ctx);

    state->logits_id.reserve(ctx->model.hparams.n_vocab);

    // TAGS: WHISPER_DECODER_INIT
    state->decoders[0].sequence.tokens.reserve(ctx->model.hparams.n_text_ctx);

    state->decoders[0].probs.reserve   (ctx->vocab.n_vocab);
    state->decoders[0].logits.reserve  (ctx->vocab.n_vocab);
    state->decoders[0].logprobs.reserve(ctx->vocab.n_vocab);

    // conv allocator
    {
        whisper_allocr_graph_init(state->alloc_conv,
                [&]() {
                    return whisper_build_graph_conv(*ctx, *state, 0);
                });

        log("%s: compute buffer (conv)   = %7.2f MB\n", __func__, whisper_allocr_size(state->alloc_conv) / 1024.0 / 1024.0);
    }

    // encoder allocator
    if (!whisper_encode_external(*state)) {
        whisper_allocr_graph_init(state->alloc_encode,
                [&]() {
                    return whisper_build_graph_encoder(*ctx, *state);
                });

        log("%s: compute buffer (encode) = %7.2f MB\n", __func__, whisper_allocr_size(state->alloc_encode) / 1024.0 / 1024.0);
    }

    // cross allocator
    {
        whisper_allocr_graph_init(state->alloc_cross,
                [&]() {
                    return whisper_build_graph_cross(*ctx, *state);
                });

        log("%s: compute buffer (cross)  = %7.2f MB\n", __func__, whisper_allocr_size(state->alloc_cross) / 1024.0 / 1024.0);
    }

    // decoder allocator
    {
        whisper_allocr_graph_init(state->alloc_decode,
                [&]() {
                    const auto & hparams = ctx->model.hparams;

                    // TODO: make sure this is the worst-case scenario
                    const int n_tokens = hparams.n_text_ctx;
                    const int n_past   = 0;

                    return whisper_build_graph_decoder(*ctx, *state, state->decoders[0], nullptr, n_tokens, n_past);
                });

        log("%s: compute buffer (decode) = %7.2f MB\n", __func__, whisper_allocr_size(state->alloc_decode) / 1024.0 / 1024.0);
    }

```

这段代码是一个C++程序，它使用了Google的GGML库。现在，让我们来逐步解释这段代码的作用。

```cppc
#ifdef GGML_USE_METAL
   state->ctx_metal = ggml_metal_init(1);
   if (!state->ctx_metal) {
       log("%s: ggml_metal_init() failed\n", __func__);
       delete state;
       return nullptr;
   }

   log("%s: Metal context initialized\n", __func__);
```

首先，定义了一个判断语句，判断该使用Metal库还是不需要使用Metal库。如果不需要使用Metal库，就执行以下操作并输出一条错误消息。如果使用Metal库，就执行以下操作并输出一条初始化消息。

```cppc
   void * data_ptr  = NULL;
   size_t data_size = 0;

   // TODO: add mmap support
   //if (params.use_mmap) {
   //    data_ptr  = ctx->model.mapping->addr;
   //    data_size = ctx->model.mapping->size;
   //} else {
   //    data_ptr  = ggml_get_mem_buffer(ctx->model.ctx);
   //    data_size = ggml_get_mem_size  (ctx->model.ctx);
   //}
```

接着，分配了一个整数类型的变量`data_ptr`，并将其初始化为`NULL`，同时定义了一个整数类型的变量`data_size`。它还包含一个if语句，用于判断是否使用Metal库。如果使用Metal库，就从`ctx->model.mapping`中获取Metal层的地址，并从`ctx->model.ctx`中获取Metal层的大小。否则，从`ggml_get_mem_buffer`函数中获取内存缓冲区，并从`ggml_get_mem_size`函数中获取要分配的内存大小。

```cppc
   const size_t max_size = ggml_get_max_tensor_size(ctx->model.ctx);
```

接下来，获取输入张量的最大大小，并将它存储在`max_size`变量中。

```cppc
   log("%s: max tensor size = %8.2f MB\n", __func__, max_size/1024.0/1024.0);
```

最后，输出一条消息，指出使用Metal库的最大tensor大小。


```cpp
#ifdef GGML_USE_METAL
    state->ctx_metal = ggml_metal_init(1);
    if (!state->ctx_metal) {
        log("%s: ggml_metal_init() failed\n", __func__);
        delete state;
        return nullptr;
    }

    log("%s: Metal context initialized\n", __func__);

    // this allocates all Metal resources and memory buffers

    void * data_ptr  = NULL;
    size_t data_size = 0;

    // TODO: add mmap support
    //if (params.use_mmap) {
    //    data_ptr  = ctx->model.mapping->addr;
    //    data_size = ctx->model.mapping->size;
    //} else {
    //    data_ptr  = ggml_get_mem_buffer(ctx->model.ctx);
    //    data_size = ggml_get_mem_size  (ctx->model.ctx);
    //}

    data_ptr  = ggml_get_mem_buffer(ctx->model.ctx);
    data_size = ggml_get_mem_size  (ctx->model.ctx);

    const size_t max_size = ggml_get_max_tensor_size(ctx->model.ctx);

    log("%s: max tensor size = %8.2f MB\n", __func__, max_size/1024.0/1024.0);

```

This code snippet appears to be a part of a machine learning pipeline, and it appears to be executing the following operations:

1. Allocating memory buffers for various data types (meta data, code data, and cross data) using the `ggml_metal_add_buffer` function.
2. Setting up a pipeline for data conversions, encoding, and decoding using the `ggml_metal_add_buffer` and `ggml_metal_check_buffer` functions.
3. Allocating a buffer for the千克键 (kv) cross-validation data and adding it to the buffer using the `ggml_metal_add_buffer` function.
4. Allocating a buffer for the self-encoded data and adding it to the buffer using the `ggml_metal_add_buffer` function.


```cpp
#define WHISPER_METAL_CHECK_BUF(result)              \
    if (!(result)) {                                 \
        log("%s: failed to add metal buffer\n", __func__); \
        delete state;                                \
        return nullptr;                              \
    }

    WHISPER_METAL_CHECK_BUF(ggml_metal_add_buffer(state->ctx_metal, "data", data_ptr, data_size, max_size));

    WHISPER_METAL_CHECK_BUF(ggml_metal_add_buffer(state->ctx_metal, "meta_conv",   state->alloc_conv.meta.data(),   state->alloc_conv.meta.size(),   0));
    WHISPER_METAL_CHECK_BUF(ggml_metal_add_buffer(state->ctx_metal, "meta_encode", state->alloc_encode.meta.data(), state->alloc_encode.meta.size(), 0));
    WHISPER_METAL_CHECK_BUF(ggml_metal_add_buffer(state->ctx_metal, "meta_cross",  state->alloc_cross.meta.data(),  state->alloc_cross.meta.size(),  0));
    WHISPER_METAL_CHECK_BUF(ggml_metal_add_buffer(state->ctx_metal, "meta_decode", state->alloc_decode.meta.data(), state->alloc_decode.meta.size(), 0));

    WHISPER_METAL_CHECK_BUF(ggml_metal_add_buffer(state->ctx_metal, "data_conv",   state->alloc_conv.data.data(),   state->alloc_conv.data.size(),   0));
    WHISPER_METAL_CHECK_BUF(ggml_metal_add_buffer(state->ctx_metal, "data_encode", state->alloc_encode.data.data(), state->alloc_encode.data.size(), 0));
    WHISPER_METAL_CHECK_BUF(ggml_metal_add_buffer(state->ctx_metal, "data_cross",  state->alloc_cross.data.data(),  state->alloc_cross.data.size(),  0));
    WHISPER_METAL_CHECK_BUF(ggml_metal_add_buffer(state->ctx_metal, "data_decode", state->alloc_decode.data.data(), state->alloc_decode.data.size(), 0));

    WHISPER_METAL_CHECK_BUF(ggml_metal_add_buffer(state->ctx_metal, "kv_cross",  state->kv_cross.buf.data(), state->kv_cross.buf.size(), 0));

    WHISPER_METAL_CHECK_BUF(ggml_metal_add_buffer(state->ctx_metal, "kv_self_0", state->decoders[0].kv_self.buf.data(), state->decoders[0].kv_self.buf.size(), 0));
```



这段代码是一个 C++ 语言中的预处理指令，其中定义了一个全局变量 whisper_金属检查缓冲区是否被定义过。这里的 whisper_金属检查缓冲区是一个宏定义，如果已经被定义过，则代码会直接返回 1，否则会创建一个新的缓冲区并返回其创建状态。

具体来说，这段代码的作用是定义了一个 whisper_金属检查缓冲区，如果这个缓冲区已经被定义过，则不做任何操作；否则创建一个新的缓冲区，并将它返回给调用者。这个缓冲区在代码中没有其他地方被使用，因此它的作用仅限于定义它的作用域。


```cpp
#undef WHISPER_METAL_CHECK_BUF
#endif

    state->rng = std::mt19937(0);

    return state;
}

int whisper_ctx_init_openvino_encoder(
        struct whisper_context * ctx,
                    const char * model_path,
                    const char * device,
                    const char * cache_dir) {
#ifndef WHISPER_USE_OPENVINO
    (void)(ctx);
    (void)(model_path);
    (void)(device);
    (void)(cache_dir);

    return 1;
```

这段代码是一个 C 语言函数，名为 `__openvino_init`，它用于初始化 OpenVINO 模型编码器。在 OpenVINO 中，模型编码器是将训练好的模型转换为模型可读的格式的过程。

具体来说，这段代码的作用如下：

1. 判断模型路径是否设置，如果设置，则可以加载模型。
2. 尝试从同一目录下查找模型文件，如果找不到，则继续尝试在当前目录下查找。
3. 如果模型文件已存在，则使用 OpenVINO 提供的路径编码器将模型文件编码为不同的格式。
4. 如果缓存目录已设置，则从该目录下加载模型。
5. 如果模型加载失败，则记录错误信息并返回 1，如果成功加载，则返回 0。


```cpp
#else
    if (!model_path && ctx->path_model.empty()) {
        log("%s: model_path is nullptr, and ctx has no model_path set.\n", __func__);
        return 1;
    }

    std::string path_encoder;
    if (!model_path) {
        //if model_path is not set, attempt to find it in the same directory as ggml-<model>.bin model
        path_encoder = whisper_openvino_get_path_encoder(ctx->path_model);
    } else {
        path_encoder = model_path;
    }

    std::string path_cache;
    if (!cache_dir) {
        //if cache_dir is not set, set it as a dir residing next to ggml-<model>.bin
        path_cache = whisper_openvino_get_path_cache(ctx->path_model);
    } else {
        path_cache = cache_dir;
    }

    log("%s: loading OpenVINO model from '%s'\n", __func__, path_encoder.c_str());
    log("%s: first run on a device may take a while ...\n", __func__);

    ctx->state->ctx_openvino = whisper_openvino_init(path_encoder.c_str(), device, path_cache.c_str());
    if (!ctx->state->ctx_openvino) {
        log("%s: failed to init OpenVINO encoder from '%s'\n", __func__, path_encoder.c_str());
        return 1;
    } else {
        log("%s: OpenVINO model loaded\n", __func__);
    }

    return 0;
```

这段代码是一个C++程序，它定义了一个名为whisper_context的结构体。这个结构体有一个指向std::ifstream对象的成员变量fin，这个ifstream对象被用来读取文件内容。

whisper_init_from_file_no_state函数的作用是加载一个模型文件，并将其存储在whisper_model_loader结构体中。它读取文件内容，并将其存储在ctx变量中。然后，它调用whisper_init函数，这个函数在函数内部初始化model和状态。

如果加载文件成功，whisper_init_from_file_no_state函数返回一个whisper_context类型的变量，这个变量将指向模型的实例。


```cpp
#endif
}

struct whisper_context * whisper_init_from_file_no_state(const char * path_model) {
    log("%s: loading model from '%s'\n", __func__, path_model);

    auto fin = std::ifstream(path_model, std::ios::binary);
    if (!fin) {
        log("%s: failed to open '%s'\n", __func__, path_model);
        return nullptr;
    }

    whisper_model_loader loader = {};

    loader.context = &fin;

    loader.read = [](void * ctx, void * output, size_t read_size) {
        std::ifstream * fin = (std::ifstream*)ctx;
        fin->read((char *)output, read_size);
        return read_size;
    };

    loader.eof = [](void * ctx) {
        std::ifstream * fin = (std::ifstream*)ctx;
        return fin->eof();
    };

    loader.close = [](void * ctx) {
        std::ifstream * fin = (std::ifstream*)ctx;
        fin->close();
    };

    auto ctx = whisper_init_no_state(&loader);

    if (ctx) {
        ctx->path_model = path_model;
    }

    return ctx;
}

```

这段代码定义了一个名为whisper_context的结构体，用于保存从缓冲区加载模型的上下文信息。

其中，whisper_init_from_buffer_no_state函数接收一个void类型的缓冲区指针和一个size_t类型的缓冲区大小，返回一个指向whisper_model_loader类型的变量，表示新的whisper_context实例。

whisper_model_loader是一个实现在缓冲区中加载模型的函数，它包括读取缓冲区、判断缓冲区末尾是否包含模型和关闭缓冲区等操作。

具体来说，whisper_init_from_buffer_no_state函数中，首先定义了一个buf_context结构体，用于保存缓冲区信息。然后，定义了一个加载器类，其中的read函数用于从缓冲区中读取数据，eof函数用于判断缓冲区是否末尾，close函数用于关闭缓冲区。

最后，调用whisper_init_from_buffer_no_state函数返回一个新的whisper_context实例，供后续使用。


```cpp
struct whisper_context * whisper_init_from_buffer_no_state(void * buffer, size_t buffer_size) {
    struct buf_context {
        uint8_t* buffer;
        size_t size;
        size_t current_offset;
    };

    buf_context ctx = { reinterpret_cast<uint8_t*>(buffer), buffer_size, 0 };

    log("%s: loading model from buffer\n", __func__);

    whisper_model_loader loader = {};

    loader.context = &ctx;

    loader.read = [](void * ctx, void * output, size_t read_size) {
        buf_context * buf = reinterpret_cast<buf_context *>(ctx);

        size_t size_to_copy = buf->current_offset + read_size < buf->size ? read_size : buf->size - buf->current_offset;

        memcpy(output, buf->buffer + buf->current_offset, size_to_copy);
        buf->current_offset += size_to_copy;

        return size_to_copy;
    };

    loader.eof = [](void * ctx) {
        buf_context * buf = reinterpret_cast<buf_context *>(ctx);

        return buf->current_offset >= buf->size;
    };

    loader.close = [](void * /*ctx*/) { };

    return whisper_init_no_state(&loader);
}

```

这段代码定义了一个名为whisper_context的结构体，用于保存模型加载器的实例。其作用是为了解决模型加载时的一些基本操作，包括初始化和关闭加载器实例。

具体来说，代码中定义了一个whisper_init_no_state函数，它接受一个whisper_model_loader类型的参数。函数首先调用ggml_time_init函数来初始化加载器使用的时间。然后，它创建了一个whisper_context实例，并尝试使用whisper_model_load函数加载模型。如果加载失败，函数将关闭加载器实例并输出一条错误消息，然后删除ctx实例并返回null。如果加载成功，函数将关闭加载器实例并返回ctx实例，以便进行后续操作。

可以看作是提供一个加载器初始化和使用的基本框架，其中通过whisper_model_load函数加载模型。如果加载失败将关闭加载器并输出错误消息，成功将返回一个whisper_context实例，以便进行后续操作。


```cpp
struct whisper_context * whisper_init_no_state(struct whisper_model_loader * loader) {
    ggml_time_init();

    whisper_context * ctx = new whisper_context;

    if (!whisper_model_load(loader, *ctx)) {
        loader->close(loader->context);
        log("%s: failed to load model\n", __func__);
        delete ctx;
        return nullptr;
    }

    loader->close(loader->context);

    return ctx;
}

```

这段代码定义了一个名为 whisper_context 的结构体，以及一个名为 whisper_init_from_file 的函数，该函数从给定的路径模型文件中初始化 whisper_context 结构体。

具体来说，whisper_init_from_file 函数的实现包括以下几个步骤：

1. 首先，函数从给定的路径模型文件中初始化一个 whisper_context 结构体，如果没有从文件中初始化，则返回 NULL。

2. 接着，函数调用 whisper_init_state 函数来初始化 whisper_context 结构体中的 state 成员。

3. 如果调用 whisper_init_state 函数时出现错误，函数将免费提醒并返回 NULL。

4. 最后，函数返回刚刚初始化的 whisper_context 结构体。


```cpp
struct whisper_context * whisper_init_from_file(const char * path_model) {
    whisper_context * ctx = whisper_init_from_file_no_state(path_model);
    if (!ctx) {
        return nullptr;
    }

    ctx->state = whisper_init_state(ctx);
    if (!ctx->state) {
        whisper_free(ctx);
        return nullptr;
    }

    return ctx;
}

```

这段代码定义了一个名为 whisper_context 的结构体，以及一个名为 whisper_init_from_buffer 的函数。

whisper_init_from_buffer 函数接收两个参数：一个指向内存缓冲区的指针，以及一个表示缓冲区大小的整数。这个函数的作用是在内存中创建一个 whisper_context 类型的变量，并将它初始化成为 whisper_init_state 的实现。

whisper_init_state 函数是一个中缀函数，它接收一个 whisper_context 类型的上下文，以及一些用于初始化状态的实参。这个函数的实现将 whisper_context 类型的初始化信息存储到 whisper_context 类型的上下文中。

最后，函数返回一个指向 whisper_context 类型的指针，这个指针将指向创建的 whisper_context 类型的上下文。


```cpp
struct whisper_context * whisper_init_from_buffer(void * buffer, size_t buffer_size) {
    whisper_context * ctx = whisper_init_from_buffer_no_state(buffer, buffer_size);
    if (!ctx) {
        return nullptr;
    }

    ctx->state = whisper_init_state(ctx);
    if (!ctx->state) {
        whisper_free(ctx);
        return nullptr;
    }

    return ctx;
}

```



这段代码定义了一个名为whisper_context的结构体，用于保存whisper协议初始化的上下文信息。其中，whisper_init函数用于初始化whisper协议加载器，并返回一个whisper_context类型的指针。

具体来说，whisper_init函数的实现如下：

1. 首先定义一个名为whisper_context的结构体，该结构体包含一个指向whisper_model_loader类型的指针loader，一个指向whisper_init_state函数的指针init_state，以及一个指向whisper_free函数的指针free;
2. 调用whisper_init_no_state函数对加载器进行初始化，如果初始化成功，则返回loader;
3. 如果初始化失败，则whisper_free函数被调用，并返回null;
4. 返回whisper_context类型的指针，该指针指向一个whisper_context结构体，其中包含初始化后的状态指针init_state，初始化后的加载器指针loader，以及一个指向whisper_free函数的指针。

因此，whisper_init函数的作用是初始化whisper协议加载器，并返回一个whisper_context结构体类型的指针，该指针包含whisper协议初始化的上下文信息。


```cpp
struct whisper_context * whisper_init(struct whisper_model_loader * loader) {
    whisper_context * ctx = whisper_init_no_state(loader);
    if (!ctx) {
        return nullptr;
    }

    ctx->state = whisper_init_state(ctx);
    if (!ctx->state) {
        whisper_free(ctx);
        return nullptr;
    }

    return ctx;
}

```

这段代码是一个名为“whisper_free_state”的函数，其作用是释放包括一个“whisper_state”结构体在内的内存。

首先，函数检查传入的“state”是否为空，如果是，则执行一些内存操作。然后，函数遍历一个名为“decoders”的数组，其中包含一个名为“whisper_max_decoders”的常量，代表最大的解码器数量。在遍历过程中，函数使用“kv_cache_free”函数释放解码器所使用的自旋锁（SELinux）数据存储器。

接下来，函数根据是否使用了“WHISPER_USE_COREML”设置，在创建或销毁 CoreML 上下文时使用“whisper_coreml_free”函数释放内存。最后，如果“state”中包含一个指向“ctx_coreml”的指针，函数使用“whisper_coreml_free”函数释放该指针。


```cpp
void whisper_free_state(struct whisper_state * state)
{
    if (state) {
        kv_cache_free(state->kv_cross);

        for (int i = 0; i < WHISPER_MAX_DECODERS; ++i) {
            kv_cache_free(state->decoders[i].kv_self);
        }

#ifdef WHISPER_USE_COREML
        if (state->ctx_coreml != nullptr) {
            whisper_coreml_free(state->ctx_coreml);
            state->ctx_coreml = nullptr;
        }
#endif

```

这段代码是一个 C 语言函数，名为 "whisper_free"，其作用是释放与 WHISPER_USE_OPENVINO 和 WHISPER_USE_METAL 相关的资源。

首先，判断当前是否使用了 WHISPER_USE_OPENVINO 和 WHISPER_USE_METAL，如果是，则执行相应的释放操作。然后，对于每个开放的 OpenVINO 和 Metal 资源，分别释放并设置为 null。接着，遍历释放的 Allocator 类型的对象，并对其进行释放操作。最后，将整个状态（state）及其所有已分配的内存对象（包括州长）都被删除。


```cpp
#ifdef GGML_USE_METAL
        if (state->ctx_metal) {
            ggml_metal_free(state->ctx_metal);
            state->ctx_metal = nullptr;
        }
#endif

#ifdef WHISPER_USE_OPENVINO
        if (state->ctx_openvino != nullptr) {
            whisper_openvino_free(state->ctx_openvino);
            state->ctx_openvino = nullptr;
        }
#endif

        whisper_allocr_free(state->alloc_conv);
        whisper_allocr_free(state->alloc_decode);
        whisper_allocr_free(state->alloc_cross);
        whisper_allocr_free(state->alloc_encode);

        delete state;
    }
}

```

这段代码定义了一个名为whisper_free的函数，它属于一个名为whisper_context的结构体。

这个函数的主要作用是释放whisper_context结构体中模型的指针和数据缓冲区，并释放该whisper_context结构体本身。

函数首先检查输入参数ctx是否为空，如果是，则执行以下操作：

1. 如果ctx指向模型的指针，则使用ggml_free函数释放该模型的指针。
2. 如果ctx指向数据缓冲区，则使用delete函数释放该数据缓冲区。
3. 然后，使用whisper_free_state函数释放该whisper_context结构体中的状态变量。
4. 最后，使用delete函数释放whisper_context结构体本身。

通过执行以上操作，函数可以确保whisper_context结构体中的所有成员在函数内部都被正确释放，从而避免内存泄漏。


```cpp
void whisper_free(struct whisper_context * ctx) {
    if (ctx) {
        if (ctx->model.ctx) {
            ggml_free(ctx->model.ctx);
        }
        if (ctx->model.buf) {
            delete ctx->model.buf;
        }

        whisper_free_state(ctx->state);

        delete ctx;
    }
}

```



这段代码定义了两个函数：

1. `whisper_free_params` 函数定义了一个结构体 `whisper_full_params` 和一个指向该结构体的指针 `params`。该函数的作用是检查 `params` 是否为空，如果是，则将其删除。

2. `whisper_pcm_to_mel_with_state` 函数定义了一个结构体 `whisper_context`、一个 `whisper_state` 和一个指向存储浮点数样本的数组 `samples`。函数的作用是计算 mel 分频信号，并将计算得到的 mel 分频信号存储在 `samples` 中。函数的参数包括一个 `whisper_context` 指针、一个 `whisper_state` 指针、一个包含浮点数样本的数组 `samples` 和计算 mel 分频信号所需的参数，如采样率、采样数、MEL 类型等。函数先调用 `log_mel_spectrogram` 函数计算 mel 分频信号，如果计算失败，则输出错误信息并返回 -1，否则返回 0。


```cpp
void whisper_free_params(struct whisper_full_params * params) {
    if (params) {
        delete params;
    }
}

int whisper_pcm_to_mel_with_state(struct whisper_context * ctx, struct whisper_state * state, const float * samples, int n_samples, int n_threads) {
    if (!log_mel_spectrogram(*state, samples, n_samples, WHISPER_SAMPLE_RATE, WHISPER_N_FFT, WHISPER_HOP_LENGTH, WHISPER_N_MEL, n_threads, ctx->model.filters, false, state->mel)) {
        log("%s: failed to compute mel spectrogram\n", __func__);
        return -1;
    }

    return 0;
}

```

这段代码实现了一个名为“whisper_pcm_to_mel”的函数，用于将PCM音频数据转换为Mel合成声音。在函数中，首先定义了一个名为“whisper_pcm_to_mel_phase_vocoder_with_state”的函数，这个函数在第一步将输入的PCM音频数据和采样率乘以一个系数，以提高计算速度。第二步通过调用一个名为“log_mel_spectrogram”的函数来获取初始状态的Mel光谱图，并将其与PCM音频数据中的采样率乘以系数，以便进一步计算。第三步，通过调用“log_mel_spectrogram”的函数，将计算得到的Mel光谱图与初始状态的Mel光谱图进行逻辑比较。如果返回值为0，则表示成功将PCM音频数据转换为Mel合成声音，否则会返回一个负数表示出现错误。


```cpp
int whisper_pcm_to_mel(struct whisper_context * ctx, const float * samples, int n_samples, int n_threads) {
    return whisper_pcm_to_mel_with_state(ctx, ctx->state, samples, n_samples, n_threads);
}

// same as whisper_pcm_to_mel, but applies a Phase Vocoder to speed up the audio x2 (PV without phase lock is not good)
int whisper_pcm_to_mel_phase_vocoder_with_state(struct whisper_context * ctx, struct whisper_state * state, const float * samples, int n_samples, int n_threads) {
    if (!log_mel_spectrogram(*state, samples, n_samples, WHISPER_SAMPLE_RATE, 2 * WHISPER_N_FFT, 2 * WHISPER_HOP_LENGTH, WHISPER_N_MEL, n_threads, ctx->model.filters, false, state->mel)) {
        log("%s: failed to compute mel spectrogram\n", __func__);
        return -1;
    }

    return 0;
}

// same as whisper_pcm_to_mel, but applies a Phase Vocoder to speed up the audio x2 (PV without phase lock is not good)
```

该代码定义了一个名为“whisper_pcm_to_mel_phase_vocoder”的函数，接受一个“whisper_context”结构的指针和一个包含浮点数样本数据的数组长度。函数实现将输入的PCM数据转换为Mel合成声音的函数，并使用了三个可选的加速技术：WSOLA、HPTSM和PV（带相位锁定的PV）。

具体来说，函数实现以下步骤：

1. 将输入的PCM数据中的数据存储在一个名为“state”的“whisper_state”结构中，该结构包含了一个包含N_MEL（即 mel的通道数）的整数和一个包含N_MEL*sizeof(float)大小的数组，用于存储Mel合成声音的数据。
2. 如果N_MEL不等于WHISPER_N_MEL，函数内部会输出一条错误消息并返回-1，因为函数需要正确处理输入的N_MEL值。
3. 如果N_MEL正确，函数将输入的PCM数据复制到一个名为“mel”的数组中，该数组长度为输入的PCM数据的长度乘以N_MEL，然后将数据存储在“state”结构中。
4. 函数使用可选的加速技术之一：WSOLA。如果该技术被启用，函数将使用高速的抽样率对输入的PCM数据进行抽样，以提高转换速度。
5. 函数使用可选的加速技术之一：HPTSM。如果该技术被启用，函数将使用高速的抽样率对输入的PCM数据进行抽样，以提高转换速度。
6. 函数使用可选的加速技术之一：PV。如果该技术被启用，函数将使用高速的抽样率对输入的PCM数据进行抽样，以提高转换速度。


```cpp
int whisper_pcm_to_mel_phase_vocoder(struct whisper_context * ctx, const float * samples, int n_samples, int n_threads) {
    return whisper_pcm_to_mel_phase_vocoder_with_state(ctx, ctx->state, samples, n_samples, n_threads);
}

// same as whisper_pcm_to_mel, but applies WSOLA to speed up the audio x2
// TODO

// same as whisper_pcm_to_mel, but applies HPTSM to speed up the audio x2
// TODO

// same as whisper_pcm_to_mel, but applies PV (with phase lock) to speed up the audio x2
// TODO

int whisper_set_mel_with_state(
        struct whisper_context * /*ctx*/,
          struct whisper_state * state,
                   const float * data,
                           int   n_len,
                           int   n_mel) {
    if (n_mel != WHISPER_N_MEL) {
        log("%s: invalid number of mel bands: %d (expected %d)\n", __func__, n_mel, WHISPER_N_MEL);
        return -1;
    }

    state->mel.n_len     = n_len;
    state->mel.n_len_org = n_len;
    state->mel.n_mel     = n_mel;

    state->mel.data.resize(n_len*n_mel);
    memcpy(state->mel.data.data(), data, n_len*n_mel*sizeof(float));

    return 0;
}

```



该代码是一个名为`whisper_set_mel`的函数，属于一个名为`whisper_context`的结构体。它将一个指向`whisper_context`结构的指针`ctx`和一个指向浮点数数据的指针`data`，以及数据的长度`n_len`和目标音高`n_mel`，作为参数。该函数的返回值表示是否成功执行了将输入数据转换为目标音高的操作。

具体来说，该函数执行以下操作：

1. 如果`whisper_context`中没有定义`whisper_state`结构体，函数将创建一个新的`whisper_state`结构体，包含一个指向目标音高的指针`target_freq`，和一个指向输入数据缓冲区的指针`in_data`。

2. 如果`whisper_encode_internal`函数执行失败，函数将抛出异常并返回-1，记录错误信息。

3. 函数首先调用`whisper_encode_internal`函数，如果没有成功，将记录错误信息并返回-1。

4. 如果`whisper_encode_internal`函数成功，函数将返回0。


```cpp
int whisper_set_mel(
        struct whisper_context * ctx,
        const float * data,
        int n_len,
        int n_mel) {
    return whisper_set_mel_with_state(ctx, ctx->state, data, n_len, n_mel);
}

int whisper_encode_with_state(struct whisper_context * ctx, struct whisper_state * state, int offset, int n_threads) {
    if (!whisper_encode_internal(*ctx, *state, offset, n_threads, nullptr, nullptr)) {
        log("%s: failed to eval\n", __func__);
        return -1;
    }

    return 0;
}

```

这两段代码是一个名为whisper_encode和whisper_decode_with_state的函数，它们是用来在whisper_token结构体中进行编码和解码的函数。

具体来说，whisper_encode函数接收一个whisper_context结构体，一个表示编码状态的整数offset，以及一个整数n_threads，然后返回一个整数，表示编码结果。如果调用whisper_encode_internal函数失败，函数将输出并返回-1。

whisper_decode_with_state函数接收一个whisper_context结构体，一个表示编码状态的整数offset，一个整数n_tokens，表示要解码的whisper token数，以及一个整数n_past，表示要回滚的步数，然后返回一个整数，表示解码结果。如果调用whisper_decode_internal函数失败，函数将输出并返回1。


```cpp
int whisper_encode(struct whisper_context * ctx, int offset, int n_threads) {
    if (!whisper_encode_internal(*ctx, *ctx->state, offset, n_threads, nullptr, nullptr)) {
        log("%s: failed to eval\n", __func__);
        return -1;
    }

    return 0;
}

int whisper_decode_with_state(struct whisper_context * ctx, struct whisper_state * state, const whisper_token * tokens, int n_tokens, int n_past, int n_threads) {
    const int selected_decoder_id = 0;

    if (!whisper_decode_internal(*ctx, *state, state->decoders[selected_decoder_id], tokens, n_tokens, n_past, n_threads, nullptr, nullptr)) {
        log("%s: failed to eval\n", __func__);
        return 1;
    }

    return 0;
}

```



这是一段用于语音识别任务的函数，名为 `whisper_decode` 。

该函数接收一个结构体 `whisper_context` 和一个整数数组 `whisper_tokens`，作为输入参数。函数的主要作用是执行选定的解码器的解密，并将结果返回。

以下是函数内部实现的主要步骤：

1. 初始化函数需要的变量和常量。
2. 如果已经加载了选定的解码器，则直接尝试调用 `whisper_decode_internal` 函数。
3. 如果未加载选定的解码器或者加载解码器失败，则记录错误并返回 1。
4. 返回 0，表示解密成功。

该函数的具体实现没有在题目中给出，但可以看出它是在处理语音识别任务中，对选定的编码器进行解密的过程。


```cpp
int whisper_decode(struct whisper_context * ctx, const whisper_token * tokens, int n_tokens, int n_past, int n_threads) {
    // TODO: add selected_decoder_id to state
    const int selected_decoder_id = 0;

    if (ctx->state == nullptr) {
        log("%s: ERROR state was not loaded.\n", __func__);
        return false;
    }

    if (!whisper_decode_internal(*ctx, *ctx->state, ctx->state->decoders[selected_decoder_id], tokens, n_tokens, n_past, n_threads, nullptr, nullptr)) {
        log("%s: failed to eval\n", __func__);
        return 1;
    }

    return 0;
}

```



这段代码是一个名为`whisper_tokenize`的函数，它对一个`whisper_context`结构体和一个字符串`text`进行 tokenize 操作，并将结果存储在`whisper_token`类型的数组中。

具体来说，代码首先通过一个名为`tokenize`的函数对`whisper_context`结构体中的`vocab`数组和给定的`text`字符串进行 tokenize 操作。然后，代码检查`n_max_tokens`参数是否小于等于生成的词语数组的大小。如果是，那么代码会输出一条日志信息，指出生成的词语数过多，然后返回-1，表示无法继续生成更多的词语。

如果`n_max_tokens`大于生成的词语数组的大小，那么代码会遍历生成的词语数组，将每个词语存储到`whisper_token`类型的数组中。最后，代码返回生成的词语数。


```cpp
int whisper_tokenize(struct whisper_context * ctx, const char * text, whisper_token * tokens, int n_max_tokens) {
    const auto res = tokenize(ctx->vocab, text);

    if (n_max_tokens < (int) res.size()) {
        log("%s: too many resulting tokens: %d (max %d)\n", __func__, (int) res.size(), n_max_tokens);
        return -1;
    }

    for (int i = 0; i < (int) res.size(); i++) {
        tokens[i] = res[i];
    }

    return res.size();
}

```

这两段代码是一个C++程序，是一个WhisperLanguage即兴编程器的入口函数。

这两段代码中定义了两个函数：

`whisper_lang_max_id()`函数的作用是获取给定语言中ID最大的词汇编号。它的实现方式是遍历语言模组(g_lang)中的所有词汇(kv)，并将它们的ID(也就是第一个单词)存储在变量`max_id`中。

`whisper_lang_id()`函数的作用是获取给定语言中ID最大的词汇编号。它的实现方式首先检查给定的语言是否存在于语言模组(g_lang)中，如果不存在，就遍历语言模组中的所有词汇，并将ID为给定语言的词汇编号存储在`kv`中。如果给定的语言存在于语言模组中，就直接返回该词汇的ID(也就是第一个单词)。

这两个函数都使用了`const`类型的变量`max_id`和`kv`来存储已经处理过的词汇ID，以便在后续的搜索中能够快速查找。


```cpp
int whisper_lang_max_id() {
    auto max_id = 0;
    for (const auto & kv : g_lang) {
        max_id = std::max(max_id, kv.second.first);
    }

    return max_id;
}

int whisper_lang_id(const char * lang) {
    if (!g_lang.count(lang)) {
        for (const auto & kv : g_lang) {
            if (kv.second.second == lang) {
                return kv.second.first;
            }
        }

        log("%s: unknown language '%s'\n", __func__, lang);
        return -1;
    }
    return g_lang.at(lang).first;
}

```

This is a C++ function that uses the Whisper language model to decode messages from a JSON document to user-level languages. It takes a context object and a vector of whisper tokens as input.

The function first initializes a vector of logits, which represent the decoded text for each whisper token. It then decodes the tokens one by one, using a helper function that takes a context object, a vector of whisper tokens, and the JSON document.

If the decoding fails, the function logs a message and returns -7. Otherwise, it returns the logits at the end of the decoding process.

The function also provides a `logits_id` property on the input object, which is a vector of logits for each language in the Whisper model. This property can be used for debugging purposes.

The function uses the `std::sort` function to sort the logits in descending order, which is useful for the softmax function that follows.

The `std::unique_ptr` is used to manage the logits id, as it is unique and unique\_ptr is not member variable of the struct.

The function uses the `WhisperTokenSOT<whisper_lang_t>` class to handle the decoding, which is a member function of the `WhisperToken<whisper_lang_t>` class that is a member of the `std::shared_ptr<WhisperModel<whisper_lang_t>>` class.

It is important to note that the function is not thread safe and should be called from a thread safe context.


```cpp
const char * whisper_lang_str(int id) {
    for (const auto & kv : g_lang) {
        if (kv.second.first == id) {
            return kv.first.c_str();
        }
    }

    log("%s: unknown language id %d\n", __func__, id);
    return nullptr;
}

int whisper_lang_auto_detect_with_state(
        struct whisper_context * ctx,
          struct whisper_state * state,
                           int   offset_ms,
                           int   n_threads,
                         float * lang_probs) {
    const int seek = offset_ms/10;

    if (seek < 0) {
        log("%s: offset %dms is before the start of the audio\n", __func__, offset_ms);
        return -1;
    }

    if (seek >= state->mel.n_len_org) {
        log("%s: offset %dms is past the end of the audio (%dms)\n", __func__, offset_ms, state->mel.n_len_org*10);
        return -2;
    }

    // run the encoder
    if (whisper_encode_with_state(ctx, state, seek, n_threads) != 0) {
        log("%s: failed to encode\n", __func__);
        return -6;
    }

    const std::vector<whisper_token> prompt = { whisper_token_sot(ctx) };

    if (whisper_decode_with_state(ctx, state, prompt.data(), prompt.size(), 0, n_threads) != 0) {
        log("%s: failed to decode\n", __func__);
        return -7;
    }

    auto & logits_id = state->logits_id;
    logits_id.clear();

    for (const auto & kv : g_lang) {
        const auto token_lang = whisper_token_lang(ctx, kv.second.first);
        logits_id.emplace_back(state->logits[token_lang], kv.second.first);
    }

    // sort descending
    {
        using pair_type = std::remove_reference<decltype(logits_id)>::type::value_type;
        std::sort(logits_id.begin(), logits_id.end(), [](const pair_type & a, const pair_type & b) {
            return a.first > b.first;
        });
    }

    // softmax
    {
        const auto max = logits_id[0].first;

        double sum = 0.0f;
        for (auto & kv : logits_id) {
            kv.first = exp(kv.first - max);
            sum += kv.first;
        }

        for (auto & kv : logits_id) {
            kv.first /= sum;
        }
    }

    {
        for (const auto & prob : logits_id) {
            if (lang_probs) {
                lang_probs[prob.second] = prob.first;
            }

            //printf("%s: lang %2d (%3s): %f\n", __func__, prob.second, whisper_lang_str(prob.second), prob.first);
        }
    }

    return logits_id[0].second;
}

```

这两组函数名下标的功能和作用如下：

1. `whisper_lang_auto_detect`函数：

这个函数的作用是接受一个结构体`whisper_context`，以及一个offsetMs的参数，和一个`n_threads`的参数，返回一个浮点数数组`lang_probs`，它表示在 whisper 语言检测过程中，每个模型的概率。函数的具体实现可能会根据上下文的不同而有所不同。

2. `whisper_model_n_vocab`函数：

这个函数的作用是返回一个结构体`whisper_context`中的`model.hparams`结构体中`n_vocab`成员的值，表示每个词汇在一个whisper模型中的个数。这个函数是用来在创建和初始化一个whisper模型时使用的。

3. `whisper_model_n_audio_ctx`函数：

这个函数的作用是返回一个结构体`whisper_context`中的`model.hparams`结构体中`n_audio_ctx`成员的值，表示每个音频上下文在一个whisper模型中的个数。这个函数也是用来在创建和初始化一个whisper模型时使用的。


```cpp
int whisper_lang_auto_detect(
        struct whisper_context * ctx,
                           int   offset_ms,
                           int   n_threads,
                         float * lang_probs) {
    return whisper_lang_auto_detect_with_state(ctx, ctx->state, offset_ms, n_threads, lang_probs);
}

int whisper_model_n_vocab(struct whisper_context * ctx) {
    return ctx->model.hparams.n_vocab;
}

int whisper_model_n_audio_ctx(struct whisper_context * ctx) {
    return ctx->model.hparams.n_audio_ctx;
}

```

这些函数是Whisper模型的音频设置。

* `whisper_model_n_audio_state`函数返回Whisper模型中的音频状态参数。
* `whisper_model_n_audio_head`函数返回Whisper模型中的音频头设置。
* `whisper_model_n_audio_layer`函数返回Whisper模型中的音频层设置。
* `whisper_model_n_text_ctx`函数返回Whisper模型中的文本上下文设置。

这些函数的主要作用是获取Whisper模型的音频设置。


```cpp
int whisper_model_n_audio_state(struct whisper_context * ctx) {
    return ctx->model.hparams.n_audio_state;
}

int whisper_model_n_audio_head(struct whisper_context * ctx) {
    return ctx->model.hparams.n_audio_head;
}

int whisper_model_n_audio_layer(struct whisper_context * ctx) {
    return ctx->model.hparams.n_audio_layer;
}

int whisper_model_n_text_ctx(struct whisper_context * ctx) {
    return ctx->model.hparams.n_text_ctx;
}

```



这些函数是Whisper模型的文本状态计算函数。

Whisper模型是一个用于自然语言处理任务的神经网络模型。这些函数计算了Whisper模型不同文本状态的个数。

具体来说，这些函数接受一个Whisper上下文结构体，并返回以下参数的值：

- `whisper_model_n_text_state`：这个函数计算了Whisper模型中所有文本状态的个数。
- `whisper_model_n_text_head`：这个函数计算了Whisper模型中所有文本头部分的个数。
- `whisper_model_n_text_layer`：这个函数计算了Whisper模型中所有文本层部分的个数。
- `whisper_model_n_mels`：这个函数计算了Whisper模型中所有mels部分的个数。

函数的实现没有提供具体的计算步骤，只是通过返回一个整数类型来表示计算结果。


```cpp
int whisper_model_n_text_state(struct whisper_context * ctx) {
    return ctx->model.hparams.n_text_state;
}

int whisper_model_n_text_head(struct whisper_context * ctx) {
    return ctx->model.hparams.n_text_head;
}

int whisper_model_n_text_layer(struct whisper_context * ctx) {
    return ctx->model.hparams.n_text_layer;
}

int whisper_model_n_mels(struct whisper_context * ctx) {
    return ctx->model.hparams.n_mels;
}

```



这两段代码是是一个C++语言的函数，属于Whisper模型的定义。

`whisper_model_ftype()`函数返回了`whisper_context`结构体中的`model.hparams.ftype`，即模型参数类型。

`whisper_model_type()`函数返回了`whisper_context`结构体中的`model.type`，即模型类型。

`whisper_model_type_readable()`函数返回了根据模型类型可读的名称，以使得使用者可以更容易地理解模型的类型。不同模型类型分别返回不同的名称，如下所示：

- `e_model::MODEL_TINY`：返回“微型”
- `e_model::MODEL_BASE`：返回“基础”
- `e_model::MODEL_SMALL`：返回“小型”
- `e_model::MODEL_MEDIUM`：返回“中号”
- `e_model::MODEL_LARGE`：返回“大号”

如果模型类型无法通过上述枚举结构的成员来确定，该函数将返回一个默认值“unknown”。


```cpp
int whisper_model_ftype(struct whisper_context * ctx) {
    return ctx->model.hparams.ftype;
}

int whisper_model_type(struct whisper_context * ctx) {
    return ctx->model.type;
}

const char *whisper_model_type_readable(struct whisper_context * ctx) {
    switch (ctx->model.type) {
    case e_model::MODEL_TINY:
        return "tiny";
    case e_model::MODEL_BASE:
        return "base";
    case e_model::MODEL_SMALL:
        return "small";
    case e_model::MODEL_MEDIUM:
        return "medium";
    case e_model::MODEL_LARGE:
        return "large";
    default:
        return "unknown";
    }
}

```

这些函数是Whisper库中的函数，它们用于从Whisper状态结构中获取不同维度的长度。

* `whisper_n_len_from_state`函数接收一个Whisper状态结构体，并返回其`mel.n_len_org`成员。`mel.n_len_org`是一个表示序列中音符的长度，它由多个时间步长组成。
* `whisper_n_len`函数同样接收一个Whisper上下文结构体，并返回其`state->mel.n_len_org`成员。这个函数没有参数，它直接从Whisper状态结构中获取音符长度。
* `whisper_n_vocab`函数接收一个Whisper上下文结构体，并返回其`vocab.n_vocab`成员。这个函数没有参数，它直接从Whisper状态结构中获取词汇表中的词汇数量。
* `whisper_n_text_ctx`函数接收一个Whisper上下文结构体，并返回其`model.hparams.n_text_ctx`成员。这个函数没有参数，它直接从Whisper状态结构中获取文本上下文中的位置。


```cpp
int whisper_n_len_from_state(struct whisper_state * state) {
    return state->mel.n_len_org;
}

int whisper_n_len(struct whisper_context * ctx) {
    return ctx->state->mel.n_len_org;
}

int whisper_n_vocab(struct whisper_context * ctx) {
    return ctx->vocab.n_vocab;
}

int whisper_n_text_ctx(struct whisper_context * ctx) {
    return ctx->model.hparams.n_text_ctx;
}

```

这两组函数定义在 whisper_core 中，它们的主要作用是控制和输出 Whisper 中音频上下文的数量以及模型是否支持多种语言。

具体来说，`whisper_n_audio_ctx()` 函数用于获取当前 Whisper 上下文中音频上下文的数量，并将其返回。这个函数主要用于在训练和推理过程中获取音频上下文的计数。

`whisper_is_multilingual()` 函数用于检查当前 Whisper 上下文的词汇是否支持多种语言。如果词汇支持多种语言，则返回 1，否则返回 0。这个函数主要用于在训练过程中确定是否需要为词汇表分离出多种语言的词汇。

`whisper_get_logits()` 函数用于获取 Whisper 当前状态中的模型输出，即 Whisper 的预测结果。这个函数主要用于在训练和推理过程中获取模型的输出结果。

`whisper_get_logits_from_state()` 函数用于获取 Whisper 状态中的模型输出，即使这些输出是在模型的输出状态中产生的。这个函数主要用于在训练过程中获取模型输出，即使模型还没有产生最终的输出结果。


```cpp
int whisper_n_audio_ctx(struct whisper_context * ctx) {
    return ctx->model.hparams.n_audio_ctx;
}

int whisper_is_multilingual(struct whisper_context * ctx) {
    return ctx->vocab.is_multilingual() ? 1 : 0;
}

float * whisper_get_logits(struct whisper_context * ctx) {
    return ctx->state->logits.data();
}

float * whisper_get_logits_from_state(struct whisper_state * state) {
    return state->logits.data();
}

```

该代码定义了四个whisper_token类型的函数，以及一个whisper_token类型的变量。函数的作用是将whisper_token类型的数据转换为字符串，并返回对应的花言巧语。变量则用于保存whisper_token类型的数据。

具体来说，这四个函数接受一个whisper_context类型的参数和一个whisper_token类型的数据，并返回相应的花言巧语。在函数内部，使用了结构体成员变量来访问whisper_context和whisper_token类型的数据，然后调用vocab数组中的一个元素来获取相应的花言巧语。

whisper_token类型的变量则用于保存输入的花言巧语，以便在需要时进行输出。


```cpp
const char * whisper_token_to_str(struct whisper_context * ctx, whisper_token token) {
    return ctx->vocab.id_to_token.at(token).c_str();
}

whisper_token whisper_token_eot(struct whisper_context * ctx) {
    return ctx->vocab.token_eot;
}

whisper_token whisper_token_sot(struct whisper_context * ctx) {
    return ctx->vocab.token_sot;
}

whisper_token whisper_token_solm(struct whisper_context * ctx) {
    return ctx->vocab.token_solm;
}

```

这是一组定义了在不同上下文情况下可以用作 "whisper_token" 的宏定义。这些宏定义了从 "struct whisper_context" 结构体中获取和设置 "whisper_token" 变量值的函数。

具体来说，这些宏定义了以下四个函数：

* "whisper_token_prev"：返回上一个 "whisper_token" 变量的值。
* "whisper_token_nosp"：返回一个没有 "Nosp" 属性的 "whisper_token"。
* "whisper_token_not"：返回一个没有 "Not" 属性的 "whisper_token"。
* "whisper_token_beg"：返回一个没有 "Beg" 属性的 "whisper_token"。

这些宏可以被用来在上下文中获取和设置 "whisper_token" 变量的值。


```cpp
whisper_token whisper_token_prev(struct whisper_context * ctx) {
    return ctx->vocab.token_prev;
}

whisper_token whisper_token_nosp(struct whisper_context * ctx) {
    return ctx->vocab.token_nosp;
}

whisper_token whisper_token_not(struct whisper_context * ctx) {
    return ctx->vocab.token_not;
}

whisper_token whisper_token_beg(struct whisper_context * ctx) {
    return ctx->vocab.token_beg;
}

```

This function appears to log information about the decoding process for a neural network. It is collecting information about the number of failed attempts, the number of prompts, the melting time, the sample time, the encoding time, the decoding time, and the prompt time. It is also calculating the total time elapsed and logging it.

The input for this function is the current state of the decoding process, which is stored in the `ctx->state` variable. The function is called with `std::max(1, ctx->state->n_decode)` which is the maximum of 1 and the number of failed attempts. The variable `ctx->state->n_prompt` is also calculated and stored within the function.

The function is using log information about the decoding process to improve the performance of the neural network. It is logging the number of failed attempts, the number of prompts, the melting time, the sample time, the encoding time, the decoding time, and the prompt time.

It is logging the total time elapsed after the neural network has been trained to run the decoding process.

Please note that the log information in the output is in milliseconds and the values are being passed in as arguements of the function.


```cpp
whisper_token whisper_token_lang(struct whisper_context * ctx, int lang_id) {
    return whisper_token_sot(ctx) + 1 + lang_id;
}

whisper_token whisper_token_translate(struct whisper_context * ctx) {
    return ctx->vocab.token_translate;
}

whisper_token whisper_token_transcribe(struct whisper_context * ctx) {
    return ctx->vocab.token_transcribe;
}

void whisper_print_timings(struct whisper_context * ctx) {
    const int64_t t_end_us = ggml_time_us();

    log("\n");
    log("%s:     load time = %8.2f ms\n", __func__, ctx->t_load_us / 1000.0f);
    if (ctx->state != nullptr) {

        const int32_t n_sample = std::max(1, ctx->state->n_sample);
        const int32_t n_encode = std::max(1, ctx->state->n_encode);
        const int32_t n_decode = std::max(1, ctx->state->n_decode);
        const int32_t n_prompt = std::max(1, ctx->state->n_prompt);

        log("%s:     fallbacks = %3d p / %3d h\n", __func__, ctx->state->n_fail_p, ctx->state->n_fail_h);
        log("%s:      mel time = %8.2f ms\n", __func__, ctx->state->t_mel_us / 1000.0f);
        log("%s:   sample time = %8.2f ms / %5d runs (%8.2f ms per run)\n", __func__, 1e-3f * ctx->state->t_sample_us, n_sample, 1e-3f * ctx->state->t_sample_us / n_sample);
        log("%s:   encode time = %8.2f ms / %5d runs (%8.2f ms per run)\n", __func__, 1e-3f * ctx->state->t_encode_us, n_encode, 1e-3f * ctx->state->t_encode_us / n_encode);
        log("%s:   decode time = %8.2f ms / %5d runs (%8.2f ms per run)\n", __func__, 1e-3f * ctx->state->t_decode_us, n_decode, 1e-3f * ctx->state->t_decode_us / n_decode);
        log("%s:   prompt time = %8.2f ms / %5d runs (%8.2f ms per run)\n", __func__, 1e-3f * ctx->state->t_prompt_us, n_prompt, 1e-3f * ctx->state->t_prompt_us / n_prompt);
    }
    log("%s:    total time = %8.2f ms\n", __func__, (t_end_us - ctx->t_start_us)/1000.0f);
}

```

这段代码是一个名为`whisper_reset_timings`的函数，属于一个名为`whisper_context`的结构体。函数的作用是重置Whisper的核心算法中的各个状态变量，使得在开始一个新的解码任务时，所有状态变量都为0。

具体来说，函数在以下步骤中实现了重置：

1. 如果Whisper上下文中存在`state`变量，则将`t_sample_us`、`t_encode_us`、`t_decode_us`、`t_prompt_us`、`n_sample`、`n_encode`、`n_decode`和`n_prompt`这些状态变量都重置为0。

2. 如果`state`变量不存在，则不做任何操作。

3. 调用`whisper_has_coreml`函数来检查是否使用了内部实现的核心算法。如果是，则返回`WHISPER_USE_COREML`的一个非零值，否则返回0。

由于`whisper_has_coreml`函数在函数头部未被定义，因此它的实现不会输出。


```cpp
void whisper_reset_timings(struct whisper_context * ctx) {
    if (ctx->state != nullptr) {
        ctx->state->t_sample_us = 0;
        ctx->state->t_encode_us = 0;
        ctx->state->t_decode_us = 0;
        ctx->state->t_prompt_us = 0;
        ctx->state->n_sample = 0;
        ctx->state->n_encode = 0;
        ctx->state->n_decode = 0;
        ctx->state->n_prompt = 0;
    }
}

static int whisper_has_coreml(void) {
#ifdef WHISPER_USE_COREML
    return 1;
```

This is a C++ function that takes in a boolean value and returns a string indicating if the input GBMM-CPU supports certain features. The features supported are:

* AVX-512 instructions
* FMA
* NEON
* ARM FMA
* Metal (OpenGL ES)
* F16C instructions
* FP16/VA instructions
* SIMD instructions
* Blas matrix multiplication
* SSE3 instructions
* SSSE3 instructions
* VSX-style instructions
* CoreML instructions
* OpenVINO instructions

The function first checks if the GBMM-CPU supports the AVX-512 instructions. If it does, it checks if it supports FMA. Then, it checks if it supports NEON. If it does not support NEON, it is assumed to not support the AVX-512 instructions.

If the GBMM-CPU supports the AVX-512 instructions and FMA, it checks if it supports NEON. If it does not support NEON, it is assumed to not support the AVX-512 instructions.

If the GBMM-CPU supports the ARM FMA, it checks if it supports the F16C instructions. If it does not support the F16C instructions, it is assumed to not support the ARM FMA.

If the GBMM-CPU supports any of the above features, it returns a string indicating that it supports those features. Otherwise, it returns an empty string.

Note: The above code is for illustration purposes only and may not reflect the actual behavior of the GBMM-CPU.


```cpp
#else
    return 0;
#endif
}

static int whisper_has_openvino(void) {
#ifdef WHISPER_USE_OPENVINO
    return 1;
#else
    return 0;
#endif
}

const char * whisper_print_system_info(void) {
    static std::string s;

    s  = "";
    s += "AVX = "       + std::to_string(ggml_cpu_has_avx())       + " | ";
    s += "AVX2 = "      + std::to_string(ggml_cpu_has_avx2())      + " | ";
    s += "AVX512 = "    + std::to_string(ggml_cpu_has_avx512())    + " | ";
    s += "FMA = "       + std::to_string(ggml_cpu_has_fma())       + " | ";
    s += "NEON = "      + std::to_string(ggml_cpu_has_neon())      + " | ";
    s += "ARM_FMA = "   + std::to_string(ggml_cpu_has_arm_fma())   + " | ";
    s += "METAL = "     + std::to_string(ggml_cpu_has_metal())     + " | ";
    s += "F16C = "      + std::to_string(ggml_cpu_has_f16c())      + " | ";
    s += "FP16_VA = "   + std::to_string(ggml_cpu_has_fp16_va())   + " | ";
    s += "WASM_SIMD = " + std::to_string(ggml_cpu_has_wasm_simd()) + " | ";
    s += "BLAS = "      + std::to_string(ggml_cpu_has_blas())      + " | ";
    s += "SSE3 = "      + std::to_string(ggml_cpu_has_sse3())      + " | ";
    s += "SSSE3 = "     + std::to_string(ggml_cpu_has_ssse3())     + " | ";
    s += "VSX = "       + std::to_string(ggml_cpu_has_vsx())       + " | ";
    s += "COREML = "    + std::to_string(whisper_has_coreml())     + " | ";
    s += "OPENVINO = "  + std::to_string(whisper_has_openvino())   + " | ";

    return s.c_str();
}

```

 This seems to be a function definition for a Kubernetes资源和御姐所说编译约束，用于控制语言模型的训练策略。具体来说，它指定了一系列可以影响最终模型的性能和行为的选项。

具体来说，这个函数指定了以下选项：

- `strategy`：采样策略，包括greedy和beam_search。greedy选项似乎指定了一些基于beam_search的策略，如best_of，最佳of的数量似乎是可变的，当前版本似乎为2。beam_search选项似乎指定了一些基于beam_search的策略，如beam_size和patience，其中beam_size指定beam的数量，patience指定一个指标，防止发出噪音。

- `whisper_sampling_strategy`：与strategy相同的whisper_sampling_strategy函数，用于给定不同的采样率，它似乎与strategy函数完全相同。

- `k的最佳适应度`：这个选项与strategy选项完全相同，似乎是一个可变值，用于度量最终模型的最佳性能。

- `最佳初始大小`：这个选项也与strategy选项完全相同，似乎是一个可变值，用于指定最终模型的初始大小。

- `输出文件`：这个选项也与strategy选项完全相同，似乎是一个可变值，用于指定保存输出数据文件的位置和文件名。

- `长期训练`：这个选项似乎指定了一个bool值，用于指定是否应该继续长期训练。

- `异步加载`：这个选项似乎指定了一个bool值，用于指定是否应该启用异步加载。

- `仅在有利噪声`：这个选项似乎指定了一个bool值，用于指定是否应该忽略有害噪声。

- `与whisper_sampling_strategy相同`：这个选项似乎是一个条件，用于指定是否与whisper_sampling_strategy选项相同。

这个函数的实现似乎基于一些默认设置，以及一些可以通过修改strategy选项来改变的行为。


```cpp
////////////////////////////////////////////////////////////////////////////

struct whisper_full_params * whisper_full_default_params_by_ref(enum whisper_sampling_strategy strategy) {
    struct whisper_full_params params = whisper_full_default_params(strategy);

    struct whisper_full_params* result = new whisper_full_params();
    *result = params;
    return result;
}

struct whisper_full_params whisper_full_default_params(enum whisper_sampling_strategy strategy) {
    struct whisper_full_params result = {
        /*.strategy          =*/ strategy,

        /*.n_threads         =*/ std::min(4, (int32_t) std::thread::hardware_concurrency()),
        /*.n_max_text_ctx    =*/ 16384,
        /*.offset_ms         =*/ 0,
        /*.duration_ms       =*/ 0,

        /*.translate         =*/ false,
        /*.no_context        =*/ true,
        /*.single_segment    =*/ false,
        /*.print_special     =*/ false,
        /*.print_progress    =*/ true,
        /*.print_realtime    =*/ false,
        /*.print_timestamps  =*/ true,

        /*.token_timestamps  =*/ false,
        /*.thold_pt          =*/ 0.01f,
        /*.thold_ptsum       =*/ 0.01f,
        /*.max_len           =*/ 0,
        /*.split_on_word     =*/ false,
        /*.max_tokens        =*/ 0,

        /*.speed_up          =*/ false,
        /*.debug_mode        =*/ false,
        /*.audio_ctx         =*/ 0,

        /*.tdrz_enable       =*/ false,

        /*.initial_prompt    =*/ nullptr,
        /*.prompt_tokens     =*/ nullptr,
        /*.prompt_n_tokens   =*/ 0,

        /*.language          =*/ "en",
        /*.detect_language   =*/ false,

        /*.suppress_blank    =*/ true,
        /*.suppress_non_speech_tokens =*/ false,

        /*.temperature       =*/  0.0f,
        /*.max_initial_ts    =*/  1.0f,
        /*.length_penalty    =*/ -1.0f,

        /*.temperature_inc   =*/  0.4f,
        /*.entropy_thold     =*/  2.4f,
        /*.logprob_thold     =*/ -1.0f,
        /*.no_speech_thold   =*/  0.6f,

        /*.greedy            =*/ {
            /*.best_of   =*/ -1,
        },

        /*.beam_search      =*/ {
            /*.beam_size =*/ -1,

            /*.patience  =*/ -1.0f,
        },

        /*.new_segment_callback           =*/ nullptr,
        /*.new_segment_callback_user_data =*/ nullptr,

        /*.progress_callback           =*/ nullptr,
        /*.progress_callback_user_data =*/ nullptr,

        /*.encoder_begin_callback           =*/ nullptr,
        /*.encoder_begin_callback_user_data =*/ nullptr,

        /*.logits_filter_callback           =*/ nullptr,
        /*.logits_filter_callback_user_data =*/ nullptr,
    };

    switch (strategy) {
        case WHISPER_SAMPLING_GREEDY:
            {
                result.greedy = {
                    /*.best_of   =*/ 2, // TODO: increase to 5 when we speed-up batch decoding
                };
            } break;
        case WHISPER_SAMPLING_BEAM_SEARCH:
            {
                result.beam_search = {
                    /*.beam_size =*/ 2, // TODO: increase to 5 when we speed-up batch decoding

                    /*.patience  =*/ -1.0f,
                };
            } break;
    }

    return result;
}

```

这段代码定义了两个静态函数：`get_signal_energy` 和 `whisper_exp_compute_token_level_timestamps`。函数名中包含了参数名称，但没有函数体。

函数 `get_signal_energy` 接收一个指向浮点数的指针 `signal`，以及 `n_samples` 和 `n_samples_per_half_window` 参数。函数返回一个 std::vector<float> 类型的变量，表示 half-words 中的信号能量。函数定义了一个 `should_split_on_word` 函数，用于判断是否应该在 half-words 中分割字符串。

函数 `whisper_exp_compute_token_level_timestamps` 接收一个 `whisper_context`、`whisper_state` 结构体、一个 `int` 类型的参数 `i_segment` 和两个浮点数的参数 `thold_pt` 和 `thold_ptsum`。函数计算了给定时刻的词级统计信息，并将统计信息存储到 `whisper_state` 中。函数定义了一个 `static` 类型的 inline 函数 `should_split_on_word`，用于判断是否应该在 half-words 中分割字符串。

函数 `get_signal_energy` 和 `whisper_exp_compute_token_level_timestamps` 可能用于计算词级统计信息，具体作用取决于代码的上下文。


```cpp
// forward declarations
static std::vector<float> get_signal_energy(const float * signal, int n_samples, int n_samples_per_half_window);
static void whisper_exp_compute_token_level_timestamps(
        struct whisper_context & ctx,
          struct whisper_state & state,
                           int   i_segment,
                         float   thold_pt,
                         float   thold_ptsum);

static inline bool should_split_on_word(const char * txt, bool split_on_word) {
    if (!split_on_word) return true;

    return txt[0] == ' ';
}

```

This is a C++ function that takes a `whisper_state` object and various parameters and returns the final result of the whispering process.

The function has a接收 input `state`, a `max_len` integer, a `split_on_word` flag, and an optional `ctx` pointer for theblic payer.

It starts by initializing a `res` variable to 1 and a `acc` variable to 0.

It then iterates through the `tokens` array of the `state.result_all` object, using the given `ctx` to check the token's id.

For each token, it gets the text by passing the token's `id` to the `whisper_token_to_str` function and adds it to the `text` variable.

Then, it checks whether the text has reached the maximum length and whether the function's `split_on_word` flag suggests that it should be split on a word. If it does, the function splits the text on the word and updates the `state.result_all` object accordingly.

If the loop has not reached the end of the `tokens` array, the function adds the tokens that have been processed to the `state.result_all` object and updates the `speaker_turn_next` flag.

Finally, the function returns the final result by returning the number of updates made to the `state.result_all` object.

Note that the function uses the `std::string` class to store the final result as a `std::string` object, and it uses the `segment.speaker_turn_next` flag to indicate the turn of the speaker.


```cpp
// wrap the last segment to max_len characters
// returns the number of new segments
static int whisper_wrap_segment(struct whisper_context & ctx, struct whisper_state & state, int max_len, bool split_on_word) {
    auto segment = state.result_all.back();

    int res = 1;
    int acc = 0;

    std::string text;

    for (int i = 0; i < (int) segment.tokens.size(); i++) {
        const auto & token = segment.tokens[i];
        if (token.id >= whisper_token_eot(&ctx)) {
            continue;
        }

        const auto txt = whisper_token_to_str(&ctx, token.id);
        const int cur = strlen(txt);

        if (acc + cur > max_len && i > 0 && should_split_on_word(txt, split_on_word)) {
            state.result_all.back().text = std::move(text);
            state.result_all.back().t1 = token.t0;
            state.result_all.back().tokens.resize(i);
            state.result_all.back().speaker_turn_next = false;

            state.result_all.push_back({});
            state.result_all.back().t0 = token.t0;
            state.result_all.back().t1 = segment.t1;

            // add tokens [i, end] to the new segment
            state.result_all.back().tokens.insert(
                state.result_all.back().tokens.end(),
                    segment.tokens.begin() + i,
                    segment.tokens.end());

            state.result_all.back().speaker_turn_next = segment.speaker_turn_next;

            acc = 0;
            text = "";

            segment = state.result_all.back();
            i = -1;

            res++;
        } else {
            acc += cur;
            text += txt;
        }
    }

    state.result_all.back().text = std::move(text);

    return res;
}

```

This is a C++ implementation of the雅典娜计划 attack algorithm for Python. The purpose of this algorithm is to convert the given text into a numerical representation that can be processed by the neural network. The input text is converted into a vector of logits, which are then used to compute the probability distribution over the output text.

The algorithm uses the information gain algorithm to compute the most informative words in the input text and their corresponding weights. These words are used to compute the log probabilities of the output text. The algorithm also uses the natural logarithm (using the `expf` function) to compute the log probabilities in a way that is compatible with the behavior of the neural network.

The output of the algorithm is a probability distribution over the output text, represented as a vector of log probabilities. This distribution can be used as input to a neural network for text classification or other machine learning tasks.


```cpp
static const std::vector<std::string> non_speech_tokens = {
    "\"", "#", "(", ")", "*", "+", "/", ":", ";", "<", "=", ">", "@", "[", "\\", "]", "^",
    "_", "`", "{", "|", "}", "~", "「", "」", "『", "』", "<<", ">>", "<<<", ">>>", "--",
    "---", "-(", "-[", "('", "(\"", "((", "))", "(((", ")))", "[[", "]]", "{{", "}}", "♪♪",
    "♪♪♪","♩", "♪", "♫", "♬", "♭", "♮", "♯"
};

// process the logits for the selected decoder
// - applies logit filters
// - computes logprobs and probs
static void whisper_process_logits(
              struct whisper_context & ctx,
               struct whisper_state  & state,
    const struct whisper_full_params   params,
              struct whisper_decoder & decoder,
                               float   temperature) {
    const auto & vocab      = ctx.vocab;
    const auto & tokens_cur = decoder.sequence.tokens;

    const bool is_initial = tokens_cur.size() == 0;
    const int  n_logits   = vocab.id_to_token.size();

    WHISPER_ASSERT(n_logits == ctx.vocab.n_vocab);

    // extract the logits for the last token
    // we will be mutating, and therefore we don't want to use the ctx.logits buffer directly
    auto & probs    = decoder.probs;
    auto & logits   = decoder.logits;
    auto & logprobs = decoder.logprobs;
    {
        logits.resize(n_logits);
        memcpy(logits.data(), state.logits.data() + (state.logits.size() - n_logits), n_logits*sizeof(float));

        if (temperature > 0.0f) {
            for (int i = 0; i < n_logits; i++) {
                logits[i] /= temperature;
            }
        }

        // will be populated a bit later
        probs.resize(n_logits);
        logprobs.resize(n_logits);
    }

    // apply logit filters here
    // ref: https://github.com/openai/whisper/blob/0b1ba3d46ebf7fe6f953acfd8cad62a4f851b49f/whisper/decoding.py#L480-L493
    {
        // suppress blank
        // https://github.com/openai/whisper/blob/0b1ba3d46ebf7fe6f953acfd8cad62a4f851b49f/whisper/decoding.py#L388-L390
        if (params.suppress_blank) {
            if (is_initial) {
                logits[vocab.token_eot]           = -INFINITY;
                logits[vocab.token_to_id.at(" ")] = -INFINITY;
            }
        }

        // suppress <|notimestamps|> token
        // ref: https://github.com/openai/whisper/blob/0b1ba3d46ebf7fe6f953acfd8cad62a4f851b49f/whisper/decoding.py#L410-L412
        logits[vocab.token_not] = -INFINITY;

        // suppress sot and nosp tokens
        logits[vocab.token_sot]  = -INFINITY;
        logits[vocab.token_nosp] = -INFINITY; // TODO: ignore this token for now

        // [TDRZ] when tinydiarize is disabled, suppress solm token
        if (params.tdrz_enable == false) {
            logits[vocab.token_solm] = -INFINITY;
        }

        // suppress task tokens
        logits[vocab.token_translate]  = -INFINITY;
        logits[vocab.token_transcribe] = -INFINITY;

        if (params.logits_filter_callback) {
            params.logits_filter_callback(&ctx, &state, tokens_cur.data(), tokens_cur.size(), logits.data(), params.logits_filter_callback_user_data);
        }

        // suppress non-speech tokens
        // ref: https://github.com/openai/whisper/blob/7858aa9c08d98f75575035ecd6481f462d66ca27/whisper/tokenizer.py#L224-L253
        if (params.suppress_non_speech_tokens) {
            for (const std::string & token : non_speech_tokens) {
                const std::string suppress_tokens[] = {token, " " + token};
                for (const std::string & suppress_token : suppress_tokens) {
                    if (vocab.token_to_id.find(suppress_token) != vocab.token_to_id.end()) {
                        logits[vocab.token_to_id.at(suppress_token)] = -INFINITY;
                    }
                }
            }

            // allow hyphens "-" and single quotes "'" between words, but not at the beginning of a word
            if (vocab.token_to_id.find(" -") != vocab.token_to_id.end()) {
                logits[vocab.token_to_id.at(" -")] = -INFINITY;
            }
            if (vocab.token_to_id.find(" '") != vocab.token_to_id.end()) {
                logits[vocab.token_to_id.at(" '")] = -INFINITY;
            }
        }

        // timestamps have to appear in pairs, except directly before EOT; mask logits accordingly
        // https://github.com/openai/whisper/blob/0b1ba3d46ebf7fe6f953acfd8cad62a4f851b49f/whisper/decoding.py#L414-L424
        {
            const bool last_was_timestamp        = tokens_cur.size() > 0 && tokens_cur.back().id >= vocab.token_beg;
            const bool penultimate_was_timestamp = tokens_cur.size() < 2 || tokens_cur[tokens_cur.size() - 2].id >= vocab.token_beg;

            //log("last_was_timestamp=%d penultimate_was_timestamp=%d\n", last_was_timestamp, penultimate_was_timestamp);

            if (last_was_timestamp) {
                if (penultimate_was_timestamp) {
                    for (int i = vocab.token_beg; i < n_logits; ++i) {
                        logits[i] = -INFINITY;
                    }
                } else {
                    for (int i = 0; i < vocab.token_eot; ++i) {
                        logits[i] = -INFINITY;
                    }
                }
            }
        }

        // the initial timestamp cannot be larger than max_initial_ts
        // ref: https://github.com/openai/whisper/blob/0b1ba3d46ebf7fe6f953acfd8cad62a4f851b49f/whisper/decoding.py#L426-L429
        if (is_initial && params.max_initial_ts > 0.0f) {
            const float precision = float(WHISPER_CHUNK_SIZE)/ctx.model.hparams.n_audio_ctx;
            const int   tid0      = std::round(params.max_initial_ts/precision);

            for (int i = vocab.token_beg + tid0 + 1; i < n_logits; ++i) {
                logits[i] = -INFINITY;
            }
        }

        // condition timestamp tokens to be increasing
        // ref: https://github.com/openai/whisper/pull/831#issuecomment-1385910556
        if (decoder.has_ts) {
            const int tid0 = decoder.seek_delta/2;

            for (int i = vocab.token_beg; i < vocab.token_beg + tid0; ++i) {
                logits[i] = -INFINITY;
            }
        }

        // populate the logprobs array (log_softmax)
        {
            const float logit_max = *std::max_element(logits.begin(), logits.end());
            float logsumexp = 0.0f;
            for (int i = 0; i < n_logits; ++i) {
                if (logits[i] > -INFINITY) {
                    logsumexp += expf(logits[i] - logit_max);
                }
            }
            logsumexp = logf(logsumexp) + logit_max;

            for (int i = 0; i < n_logits; ++i) {
                if (logits[i] > -INFINITY) {
                    logprobs[i] = logits[i] - logsumexp;
                } else {
                    logprobs[i] = -INFINITY;
                }
            }
        }

        // if sum of probability over timestamps is above any other token, sample timestamp
        // ref: https://github.com/openai/whisper/blob/0b1ba3d46ebf7fe6f953acfd8cad62a4f851b49f/whisper/decoding.py#L431-L437
        {
            // logsumexp over timestamps
            float timestamp_logprob = -INFINITY;
            {
                float logsumexp = 0.0f;
                const float logprob_max = *std::max_element(logprobs.begin() + vocab.token_beg, logprobs.end());
                for (int i = vocab.token_beg; i < n_logits; ++i) {
                    if (logprobs[i] > -INFINITY) {
                        logsumexp += expf(logprobs[i] - logprob_max);
                    }
                }
                if (logsumexp > 0.0f) {
                    timestamp_logprob = logf(logsumexp) + logprob_max;
                }
            }

            const float max_text_token_logprob = *std::max_element(logprobs.begin(), logprobs.begin() + vocab.token_beg);

            //log("timestamp_logprob=%f max_text_token_logprob=%f\n", timestamp_logprob, max_text_token_logprob);

            if (timestamp_logprob > max_text_token_logprob) {
                for (int i = 0; i < vocab.token_beg; ++i) {
                    logits[i]   = -INFINITY;
                    logprobs[i] = -INFINITY;
                }
            }
        }
    }

    // compute probs
    {
        for (int i = 0; i < n_logits; ++i) {
            if (logits[i] == -INFINITY) {
                probs[i] = 0.0f;
            } else {
                probs[i] = expf(logprobs[i]);
            }
        }
    }

```

It looks like you are getting a weighted sum of logits for each token that is in the vocabulary. The logits are likely obtained from a language model, such as BERT or RoBERTa. The probabilities for each token are also included and likely obtained from the same source as the logits.


```cpp
#if 0
    // print first 100 logits - token string : logit
    for (int i = 0; i < 100; i++) {
        const auto token   = vocab.id_to_token.at(i);
        const auto prob    = probs[i];
        const auto logit   = logits[i];
        const auto logprob = logprobs[i];
        printf("%s : prob=%9.5f logit=%9.5f logprob=%9.5f\n", token.c_str(), prob, logit, logprob);
    }

    // "And", "and", " And", " and"
    printf("logits[\"and\"]  = %f\n", logits[vocab.token_to_id.at("and")]);
    printf("logits[\"And\"]  = %f\n", logits[vocab.token_to_id.at("And")]);
    printf("logits[\" and\"] = %f\n", logits[vocab.token_to_id.at(" and")]);
    printf("logits[\" And\"] = %f\n", logits[vocab.token_to_id.at(" And")]);
    printf("logits[\" so\"]  = %f\n", logits[vocab.token_to_id.at(" so")]);

    printf("logprobs[\"and\"]  = %f\n", logprobs[vocab.token_to_id.at("and")]);
    printf("logprobs[\"And\"]  = %f\n", logprobs[vocab.token_to_id.at("And")]);
    printf("logprobs[\" and\"] = %f\n", logprobs[vocab.token_to_id.at(" and")]);
    printf("logprobs[\" And\"] = %f\n", logprobs[vocab.token_to_id.at(" And")]);
    printf("logprobs[\" so\"]  = %f\n", logprobs[vocab.token_to_id.at(" so")]);

    printf("probs[\"and\"]  = %f\n", probs[vocab.token_to_id.at("and")]);
    printf("probs[\"And\"]  = %f\n", probs[vocab.token_to_id.at("And")]);
    printf("probs[\" and\"] = %f\n", probs[vocab.token_to_id.at(" and")]);
    printf("probs[\" And\"] = %f\n", probs[vocab.token_to_id.at(" And")]);
    printf("probs[\" so\"]  = %f\n", probs[vocab.token_to_id.at(" so")]);
```

This is a C++ implementation of the Whisper-style tokenizers with attention mechanism. The `WhisperTokenizer` class takes a given vocabulary `ctx`, a decoder `decoder`, and an optional distribution `best`. It returns a sample of the tokenizer outputs.

The tokenizer has two main components: the logits and the probabilities. The logits are the predicted log probabilities for each token in the vocabulary. The probabilities are the predicted probability scores for each token.

The `decoder` should be an instance of a class that implements the `WhisperDecoder` interface. This interface should have a `forward()` method that generates the output sequence for each token in the vocabulary.

The `best` parameter is a boolean flag indicating whether to use the predicted probabilities as the token importance scores.

Here is the implementation for the `WhisperTokenizer` class:
```cppcpp
#include <vector>

namespace mk {

class WhisperTokenizer {
public:
   //! tokenize(vocab: Vocab, decoder: Decoder, best: bool = false)
   //! Prepare the tokenizer to compute predicted logits, probs, and return the best token.
   //!
   //! tokenize(vocab: Vocab, decoder: Decoder, best: bool = false)
   //! Prepare the tokenizer to compute predicted logits, probs, and return the best token.
   void tokenize(const std::vector<std::pair<int, int>> &vocab, std::vector<std::pair<double, double>> &probs, std::vector<int> &token_ids, bool &best = false);

private:
   //! This is a stream of更高 probability tokens.
   std::vector<std::pair<int, int>> _probs;

   //! This decoder to use for decoding the input tokens.
   std::shared_ptr<mpp::Decoder> _decoder;

   //! This store for the predicted token IDs.
   std::vector<int> _token_ids;

   //! This flag indicating whether to use the predicted probabilities as the token importance scores.
   bool _best;

   //! Compute the predicted log probabilities for each token in the vocabulary.
   void compute_logits(const std::vector<std::pair<int, int>> &vocab, std::vector<std::pair<double, double>> &probs, std::vector<int> &token_ids, bool &best);

   //! Compute the predicted probability score for each token in the vocabulary.
   void compute_probs(const std::vector<std::pair<int, int>> &vocab, std::vector<double> &probs, std::vector<int> &token_ids, bool &best);
};
```
The `tokenize()` method takes a vocabulary `ctx`, a decoder `decoder`, and an optional `best` flag. It computes the predicted logits, probs, and token IDs for each token in the vocabulary. If `best` is true, it uses the predicted probabilities as the token importance scores.

The `compute_logits()` method computes the predicted log probabilities for each token in the vocabulary. It uses the `decoder` to generate the output probabilities for each token.

The `compute_probs()` method computes the predicted probability score for each token in the vocabulary. It uses the `decoder` to generate the output probabilities for each token and the predicted token IDs to store the predicted probability scores.

Here is the implementation for the `WhisperDecoder` class:
```cppcpp
#include <vector>
#include <map>
#include <cmath>

namespace mk {

class WhisperDecoder {
public:
   //! forward()(input: Vector< pair<int, int>, int>, Vector< pair<double, double>, int> &output)
   //!
   //! forward()(input: Vector< pair<int, int>, int>, Vector< pair<double, double>, int> &output)
   //!
   //! forward()(input: Vector< pair<int, int>, int>, Vector< pair<double, double>, int> &output)
   void forward(const std::vector<std::pair<int, int>> &input, std::vector<std::pair<double, double>> &output);

private:
   //! This is a map from predicted tokens to their index in the vocabulary.
   std::map<std::pair<int, int>, int> _token_map;

   //! This store for the predicted output sequence.
   std::vector<int> _output_seq;

   //! This store for the predicted token IDs.
   std::vector<int> _token_ids;

   //! Compute the predicted index for the input token.
   int _get_token_index(const std::pair<int, int> &token) {
       return _token_map.find(token.first) ? token.second : -1;
   }

   //! Compute the predicted log probability for each token in the input sequence.
   double _compute_logits(const std::vector<std::pair<int, int>> &input, std::vector<std::pair<double, double>> &probs, std::vector<int> &token_ids, bool &best);

   //! Compute the predicted probability score for each token in the input sequence.
   void _compute_probs(const std::vector<std::pair<int, int>> &input, std::vector<double> &probs, std::vector<int> &token_ids, bool &best);
};
```
The `WhisperDecoder` class has a `forward()` method that takes an input sequence and generates the output sequence. It uses the `compute_logits()` and `compute_probs()` methods to generate the output sequence and compute the predicted log and probability scores, respectively.


```cpp
#endif
}

static whisper_token_data whisper_sample_token(
            whisper_context & ctx,
              whisper_state & state,
      const whisper_decoder & decoder,
                       bool   best) {
    whisper_token_data result = {
        0, 0, 0.0f, 0.0f, 0.0f, 0.0f, -1, -1, 0.0f,
    };

    const auto & vocab = ctx.vocab;

    const auto & probs    = decoder.probs;
    const auto & logprobs = decoder.logprobs;

    const int n_logits = vocab.n_vocab;

    {
        double sum_ts = 0.0;
        double max_ts = 0.0;

        for (int i = vocab.token_beg; i < n_logits; i++) {
            if (probs[i] == -INFINITY) {
                continue;
            }

            sum_ts += probs[i];
            if (max_ts < probs[i]) {
                max_ts = probs[i];
                result.tid = i;
            }
        }

        result.pt    = max_ts/(sum_ts + 1e-10);
        result.ptsum = sum_ts;
    }

    if (best) {
        for (int i = 0; i < n_logits; ++i) {
            if (result.p < probs[i]) {
                result.id   = i;
                result.p    = probs[i];
                result.plog = logprobs[i];
            }
        }
    } else {
        std::discrete_distribution<> dist(probs.begin(), probs.end());

        result.id   = dist(state.rng);
        result.p    = probs[result.id];
        result.plog = logprobs[result.id];
    }

    if (result.id >= vocab.token_beg) {
        result.tid = result.id;
        result.pt  = result.p;
    }

    state.n_sample++;

    return result;
}

```

This is a C++ implementation of the Whisper recommendation algorithm. It uses the logits of a language model to generate personalized recommendations for new users based on their interests.

The algorithm first generates a set of k recommenders, where k is the number of recommended users per user. Each recommender is represented by a vector of whisper tokens, which are words in the vocabulary.

The algorithm sorts the logits based on their probability and uses the highest probability logit to determine the recommender. If multiple logits have the same highest probability, the algorithm chooses the one that comes first.

The algorithm then generates k/2 recommenders for each user and merges their results to generate a total of k recommenders for each user.

The algorithm keeps track of the token IDs and uses these IDs to store information about which users have which recommenders.

The algorithm also generates a set of -1 probabilities for each logit, which represent stubs or trailing off points. These probabilities allow the algorithm to avoid stubs and trailing off points when generating recommendations.

The output of the algorithm is a vector of k/2 pairs of user IDs and recommender IDs, along with metadata such as the log probabilities and other information.

This implementation does not handle the validation or error handling of the input data, such as the possibility that some of the users may not have a logit or that the logits may be negative. It also does not handle the recommendation response phase, where the actual recommendations are generated for the user.


```cpp
static std::vector<whisper_token_data> whisper_sample_token_topk(
            whisper_context & ctx,
              whisper_state & state,
      const whisper_decoder & decoder,
                        int   k) {
    const auto & vocab = ctx.vocab;

    const auto & probs    = decoder.probs;
    const auto & logits   = decoder.logits;
    const auto & logprobs = decoder.logprobs;

    const int n_logits = vocab.n_vocab;

    auto & logits_id = state.logits_id;

    logits_id.resize(n_logits);
    for (int i = 0; i < n_logits; ++i) {
        logits_id[i].first = logits[i];
        logits_id[i].second = i;
    }

    {
        using pair_type = std::remove_reference<decltype(logits_id)>::type::value_type;
        std::partial_sort(
                logits_id.begin(),
                logits_id.begin() + k, logits_id.end(),
                [](const pair_type & a, const pair_type & b) {
            return a.first > b.first;
        });
    }

    std::vector<whisper_token_data> result;
    result.reserve(k);

    whisper_token tid = vocab.token_beg;

    float pt    = 0.0;
    float ptsum = 0.0;

    {
        double sum_ts = 0.0;
        double max_ts = 0.0;

        for (int i = vocab.token_beg; i < n_logits; i++) {
            if (probs[i] == -INFINITY) {
                continue;
            }

            sum_ts += probs[i];
            if (max_ts < probs[i]) {
                max_ts = probs[i];
                tid = i;
            }
        }

        pt    = max_ts/(sum_ts + 1e-10);
        ptsum = sum_ts;
    }

    for (int i = 0; i < k; ++i) {
        const auto id = logits_id[i].second;

        result.push_back({ id, tid, probs[id], logprobs[id], pt, ptsum, -1, -1, 0.0f, });

        if (result[i].id >= vocab.token_beg) {
            result[i].tid = result[i].id;
            result[i].pt  = result[i].p;
        }
    }

    state.n_sample++;

    return result;
}

```



This is a Python function that performs a simple optimization task called "whisper" that modifies the score of a sequence of shortest arithmetic programming (SAP) problems. The function takes a `whisper_full_params` struct as input and returns the updated sequence score.

The `whisper_sequence_score` function has the following signature:
```cpp
static void whisper_sequence_score(
   const struct whisper_full_params & params,
   whisper_sequence & sequence) {
       //L178-L192
   }
```
This function iterates over the result of the sequence and calculates the average log probability for each token in the result. It also calculates the entropy of the sequence by counting the number of occurrences of each token and estimating the log probability of each token based on its frequency. Finally, it computes the score by dividing the result by the penalty.

The function takes a `params` parameter of type `whisper_full_params` and an instance of `whisper_sequence` as input. The `whisper_full_params` struct has several fields, including `result_len`, `tokens`, and `length_penalty`, which are not defined in this function but are passed in as input.

The function returns an instance of `whisper_sequence` that has been modified by the scoring process.


```cpp
// ref: https://github.com/openai/whisper/blob/0b1ba3d46ebf7fe6f953acfd8cad62a4f851b49f/whisper/decoding.py#L178-L192
static void whisper_sequence_score(
        const struct whisper_full_params & params,
                        whisper_sequence & sequence) {
    if (sequence.result_len == 0) {
        return;
    }

    double result = 0.0f;

    for (int i = 0; i < sequence.result_len; ++i) {
        result += sequence.tokens[i].plog;
    }

    sequence.sum_logprobs = result;
    sequence.avg_logprobs = result/sequence.result_len;

    double penalty = sequence.result_len;

    if (params.length_penalty > 0.0f) {
        penalty = pow((5.0 + penalty)/6.0, params.length_penalty);
    }

    sequence.score = result/penalty;

    // compute the entropy of the sequence of the last 32 tokens
    {
        const int n = 32;

        int cnt = 0;
        double entropy = 0.0f;

        std::map<whisper_token, int> token_counts;
        for (int i = std::max(0, sequence.result_len - n); i < sequence.result_len; ++i) {
            token_counts[sequence.tokens[i].id]++;
            cnt++;
        }

        for (const auto & kv : token_counts) {
            const auto p = kv.second/(double)cnt;
            entropy -= p*log(p);

            //WHISPER_PRINT_DEBUG("entropy: %d %f %f, count %d\n", kv.first, p, log(p), kv.second);
        }

        sequence.entropy = entropy;
    }
}

```

This function appears to be part of a larger software build, and it appears to be a utility for swapping the key-value pairs (KV) of a decoder.

The function takes in several parameters, including a view of the original key-value pairs and a pointer to a buffer containing swap data. It appears to check whether the swap data has already been modified, and if it has, it copies the data to the correct KV cache using data from the swap buffer. If the swap data has not yet been modified, it appears to copy the data directly from the original key-value pairs to the corresponding KV cache.

It appears to be using a two-copy mechanism, where it prints debug information for one-copy and two-copy decoders.

The function also seems to be using a swap buffer, which is not defined in the code, but it appears to be a buffer of data used for swapping the KV pairs.

Overall, this function seems to be a utility for copying data between the KV cache and the decoder's local data, and it is used by the software to handle the swapping of key-value pairs.


```cpp
static bool whisper_kv_swap_fast(
                   std::vector<int> & view,
                    whisper_decoder   src[],
                std::vector<kv_buf> & kv_swap_bufs,
                          const int & n_decoders) {
    WHISPER_PRINT_DEBUG("%s: n_decoders %d\n", __func__, n_decoders);

    // (decoder->buffer->decoder or decoder->buffer + decoder->decoder)
    std::set<int> two_copy; // decoder indices require two copies to safely modify KV caches

    // (buffer->decoder or decoder->decoder)
    std::set<int> one_copy; // decoder indices require one copy to safely modify KV caches

    // (decoder<->decoder)
    std::set<int> p_swap_set; // decoder indices able to swap KV-cache pointers
    std::vector<whisper_pair<int, int>> p_swap_vec;
    p_swap_vec.reserve(n_decoders);

    // see https://github.com/ggerganov/whisper.cpp/wiki
    for (int i = 0; i < n_decoders; i++) {
        // zero-copy (no modification)
        if (i == view[i] || view[i] < 0) {
            continue;
        }

        bool is_one_copy = true;
        // since we modify data sequentially, we only consider decoder indices after current index
        for (int j = i + 1; j < n_decoders; j++) {
            if (i == view[j]) {
                // detect symmetric diagram
                if (j == view[i]) {
                    p_swap_set.insert(i);
                    p_swap_set.insert(j);
                    p_swap_vec.emplace_back(i, j);
                } else {
                    two_copy.insert(i);
                    is_one_copy = false;
                }
                break;
            }
        }
        if (is_one_copy) {
            one_copy.insert(i);
        }
    }

    kv_swap_bufs.resize(n_decoders);

    for (int i = 0; i < n_decoders; i++) {
        kv_swap_bufs[i].k.resize(ggml_nbytes(src[i].kv_self.k));
        kv_swap_bufs[i].v.resize(ggml_nbytes(src[i].kv_self.v));
    }

    for (auto & i : two_copy) {
        // make a copy of KV caches
        WHISPER_PRINT_DEBUG("%s: store KV cache into swap: idx %d\n", __func__, i);
        memcpy(kv_swap_bufs[i].k.data(), src[i].kv_self.k->data, kv_swap_bufs[i].k.size());
        memcpy(kv_swap_bufs[i].v.data(), src[i].kv_self.v->data, kv_swap_bufs[i].v.size());
    }

    // since two-copy decoder KV caches are protected by kv_swap_bufs, modify them first
    for (auto & i : two_copy) {
        // skip the decoder indices that require pointer swapping
        if (p_swap_set.find(i) != p_swap_set.end()) {
            continue;
        }

        if (two_copy.find(view[i]) != two_copy.end()) {
            // modify KV caches of decoder using data from kv_swap_bufs
            WHISPER_PRINT_DEBUG("%s: two-copy decoder using   swap buffers: swap[%d] -> %d\n", __func__, view[i], i);
            memcpy(src[i].kv_self.k->data, kv_swap_bufs[view[i]].k.data(), kv_swap_bufs[view[i]].k.size());
            memcpy(src[i].kv_self.v->data, kv_swap_bufs[view[i]].v.data(), kv_swap_bufs[view[i]].v.size());
        } else {
            // modify KV caches of decoder using data from correspond decoder KV caches directly
            WHISPER_PRINT_DEBUG("%s: two-copy decoder without swap buffers:      %d  -> %d\n", __func__, view[i], i);
            memcpy(src[i].kv_self.k->data, src[view[i]].kv_self.k->data, ggml_nbytes(src[view[i]].kv_self.k));
            memcpy(src[i].kv_self.v->data, src[view[i]].kv_self.v->data, ggml_nbytes(src[view[i]].kv_self.v));
        }
    }

    // then modify one-copy decoder KV caches
    for (auto & i : one_copy) {
        // skip the decoder indices that require pointer swapping
        if (p_swap_set.find(i) != p_swap_set.end()) {
            continue;
        }

        if (two_copy.find(view[i]) != two_copy.end()) {
            // modify KV caches of decoder using data from kv_swap_bufs
            WHISPER_PRINT_DEBUG("%s: one-copy decoder using   swap buffers: swap[%d] -> %d\n", __func__, view[i], i);
            memcpy(src[i].kv_self.k->data, kv_swap_bufs[view[i]].k.data(), kv_swap_bufs[view[i]].k.size());
            memcpy(src[i].kv_self.v->data, kv_swap_bufs[view[i]].v.data(), kv_swap_bufs[view[i]].v.size());
        } else {
            // modify KV caches of decoder using data from correspond decoder KV caches directly
            WHISPER_PRINT_DEBUG("%s: one-copy decoder without swap buffers:      %d  -> %d\n", __func__, view[i], i);
            memcpy(src[i].kv_self.k->data, src[view[i]].kv_self.k->data, ggml_nbytes(src[view[i]].kv_self.k));
            memcpy(src[i].kv_self.v->data, src[view[i]].kv_self.v->data, ggml_nbytes(src[view[i]].kv_self.v));
        }
    }

    // swap the pointers
    for (auto & i : p_swap_vec) {
        WHISPER_PRINT_DEBUG("%s: swap pointers: %d <-> %d\n", __func__, i.first, i.second);
        std::swap(src[i.first].kv_self, src[i.second].kv_self);
    }

    return true;
}

```

It looks like you are trying to initialize a TAG-based self-attention decoder. This involves initializing the decoder


```cpp
int whisper_full_with_state(
        struct whisper_context * ctx,
          struct whisper_state * state,
    struct whisper_full_params   params,
                   const float * samples,
                           int   n_samples) {
    // clear old results
    auto & result_all = state->result_all;

    result_all.clear();

    if (n_samples > 0) {
        // compute log mel spectrogram
        if (params.speed_up) {
            // TODO: Replace PV with more advanced algorithm
            log("%s: failed to compute log mel spectrogram\n", __func__);
            return -1;
        } else {
            if (whisper_pcm_to_mel_with_state(ctx, state, samples, n_samples, params.n_threads) != 0) {
                log("%s: failed to compute log mel spectrogram\n", __func__);
                return -2;
            }
        }
    }

    // auto-detect language if not specified
    if (params.language == nullptr || strlen(params.language) == 0 || strcmp(params.language, "auto") == 0 || params.detect_language) {
        std::vector<float> probs(whisper_lang_max_id() + 1, 0.0f);

        const auto lang_id = whisper_lang_auto_detect_with_state(ctx, state, 0, params.n_threads, probs.data());
        if (lang_id < 0) {
            log("%s: failed to auto-detect language\n", __func__);
            return -3;
        }
        state->lang_id = lang_id;
        params.language = whisper_lang_str(lang_id);

        log("%s: auto-detected language: %s (p = %f)\n", __func__, params.language, probs[whisper_lang_id(params.language)]);
        if (params.detect_language) {
            return 0;
        }
    }

    if (params.token_timestamps) {
        state->t_beg    = 0;
        state->t_last   = 0;
        state->tid_last = 0;
        if (n_samples > 0) {
            state->energy = get_signal_energy(samples, n_samples, 32);
        }
    }

    const int seek_start = params.offset_ms/10;
    const int seek_end = params.duration_ms == 0 ? whisper_n_len_from_state(state) : seek_start + params.duration_ms/10;

    // if length of spectrogram is less than 1.0s (100 frames), then return
    // basically don't process anything that is less than 1.0s
    // see issue #39: https://github.com/ggerganov/whisper.cpp/issues/39
    if (seek_end < seek_start + (params.speed_up ? 50 : 100)) {
        return 0;
    }

    // a set of temperatures to use
    // [ t0, t0 + delta, t0 + 2*delta, ..., < 1.0f + 1e-6f ]
    std::vector<float> temperatures;
    if (params.temperature_inc > 0.0f) {
        for (float t = params.temperature; t < 1.0f + 1e-6f; t += params.temperature_inc) {
            temperatures.push_back(t);
        }
    } else {
        temperatures.push_back(params.temperature);
    }

    // initialize the decoders
    int n_decoders = 1;

    switch (params.strategy) {
        case WHISPER_SAMPLING_GREEDY:
            {
                n_decoders = params.greedy.best_of;
            } break;
        case WHISPER_SAMPLING_BEAM_SEARCH:
            {
                n_decoders = std::max(params.greedy.best_of, params.beam_search.beam_size);
            } break;
    };

    n_decoders = std::max(1, n_decoders);

    // TAGS: WHISPER_DECODER_INIT
    for (int j = 1; j < n_decoders; j++) {
        auto & decoder = state->decoders[j];

        if (decoder.kv_self.ctx == nullptr) {
            decoder.kv_self = state->decoders[0].kv_self;
            if (!kv_cache_reinit(decoder.kv_self)) {
                log("%s: kv_cache_reinit() failed for self-attention, decoder %d\n", __func__, j);
                return -4;
            }

            WHISPER_PRINT_DEBUG("%s: initialized self-attention kv cache, decoder %d\n", __func__, j);

            decoder.sequence.tokens.reserve(state->decoders[0].sequence.tokens.capacity());

            decoder.probs.resize   (ctx->vocab.n_vocab);
            decoder.logits.resize  (ctx->vocab.n_vocab);
            decoder.logprobs.resize(ctx->vocab.n_vocab);

            // TODO: not very clean - look for a better way and potentially merging with the init of decoder 0
```

This code appears to be a part of a speech recognition system that uses multi-pass decoding. multi-pass decoding is a technique for training large language models by decoding停下来自动化仅自己。The code contains several nested if statements and dynamic function call, it should be able to handle different cases. But I'm unable to understand the fully context of this code. If you could provide more information about what it is supposed to do, I'd be happy to help you understand it better.


```cpp
#ifdef GGML_USE_METAL
#define WHISPER_METAL_CHECK_BUF(result)              \
            if (!(result)) {                                 \
                log("%s: failed to add metal buffer\n", __func__); \
                return 0;                              \
            }

            const std::string kv_name = "kv_self_" + std::to_string(j);
            auto & kv_self = decoder.kv_self;

            WHISPER_METAL_CHECK_BUF(ggml_metal_add_buffer(state->ctx_metal, kv_name.c_str(), kv_self.buf.data(), kv_self.buf.size(), 0));
#undef WHISPER_METAL_CHECK_BUF
#endif
        }
    }

    // the accumulated text context so far
    auto & prompt_past = state->prompt_past;
    if (params.no_context) {
        prompt_past.clear();
    }

    // prepare prompt
    {
        std::vector<whisper_token> prompt_tokens;

        // initial prompt
        if (!params.prompt_tokens && params.initial_prompt) {
            prompt_tokens.resize(1024);
            prompt_tokens.resize(whisper_tokenize(ctx, params.initial_prompt, prompt_tokens.data(), prompt_tokens.size()));
            params.prompt_tokens   = prompt_tokens.data();
            params.prompt_n_tokens = prompt_tokens.size();
        }

        // prepend the prompt tokens to the prompt_past
        if (params.prompt_tokens && params.prompt_n_tokens > 0) {
            // parse tokens from the pointer
            for (int i = 0; i < params.prompt_n_tokens; i++) {
                prompt_past.push_back(params.prompt_tokens[i]);
            }
            std::rotate(prompt_past.begin(), prompt_past.end() - params.prompt_n_tokens, prompt_past.end());
        }
    }

    // overwrite audio_ctx, max allowed is hparams.n_audio_ctx
    if (params.audio_ctx > whisper_n_audio_ctx(ctx)) {
        log("%s: audio_ctx is larger than the maximum allowed (%d > %d)\n", __func__, params.audio_ctx, whisper_n_audio_ctx(ctx));
        return -5;
    }
    state->exp_n_audio_ctx = params.audio_ctx;

    // these tokens determine the task that will be performed
    std::vector<whisper_token> prompt_init = { whisper_token_sot(ctx) };
    if (whisper_is_multilingual(ctx)) {
        const int lang_id = whisper_lang_id(params.language);
        state->lang_id = lang_id;
        prompt_init.push_back(whisper_token_lang(ctx, lang_id));
        if (params.translate) {
            prompt_init.push_back(whisper_token_translate(ctx));
        } else {
            prompt_init.push_back(whisper_token_transcribe(ctx));
        }
    }

    int seek = seek_start;

    std::vector<whisper_token> prompt;
    prompt.reserve(whisper_n_text_ctx(ctx));

    struct beam_candidate {
        int decoder_idx;
        int seek_delta;

        bool has_ts;

        whisper_sequence sequence;
    };

    std::vector<beam_candidate> beam_candidates;

    // main loop
    while (true) {
        if (params.progress_callback) {
            const int progress_cur = (100*(seek - seek_start))/(seek_end - seek_start);

            params.progress_callback(
                ctx, ctx->state, progress_cur, params.progress_callback_user_data);
        }

        // of only 1 second left, then stop
        if (seek + 100 >= seek_end) {
            break;
        }

        if (params.encoder_begin_callback) {
            if (params.encoder_begin_callback(ctx, state, params.encoder_begin_callback_user_data) == false) {
                log("%s: encoder_begin_callback returned false - aborting\n", __func__);
                break;
            }
        }

        // encode audio features starting at offset seek
        if (!whisper_encode_internal(*ctx, *state, seek, params.n_threads, params.abort_callback, params.abort_callback_user_data)) {
            log("%s: failed to encode\n", __func__);
            return -6;
        }

        // if there is a very short audio segment left to process, we remove any past prompt since it tends
        // to confuse the decoder and often make it repeat or hallucinate stuff
        if (seek > seek_start && seek + 500 >= seek_end) {
            prompt_past.clear();
        }

        int best_decoder_id = 0;

        for (int it = 0; it < (int) temperatures.size(); ++it) {
            const float t_cur = temperatures[it];

            int n_decoders_cur = 1;

            switch (params.strategy) {
                case whisper_sampling_strategy::WHISPER_SAMPLING_GREEDY:
                    {
                        if (t_cur > 0.0f) {
                            n_decoders_cur = params.greedy.best_of;
                        }
                    } break;
                case whisper_sampling_strategy::WHISPER_SAMPLING_BEAM_SEARCH:
                    {
                        if (t_cur > 0.0f) {
                            n_decoders_cur = params.greedy.best_of;
                        } else {
                            n_decoders_cur = params.beam_search.beam_size;
                        }
                    } break;
            };

            n_decoders_cur = std::max(1, n_decoders_cur);

            WHISPER_PRINT_DEBUG("\n%s: decoding with %d decoders, temperature = %.2f\n", __func__, n_decoders_cur, t_cur);

            // TAGS: WHISPER_DECODER_INIT
            for (int j = 0; j < n_decoders_cur; ++j) {
                auto & decoder = state->decoders[j];

                decoder.kv_self.n = 0;

                decoder.sequence.tokens.clear();
                decoder.sequence.result_len       = 0;
                decoder.sequence.sum_logprobs_all = 0.0;
                decoder.sequence.sum_logprobs     = -INFINITY;
                decoder.sequence.avg_logprobs     = -INFINITY;
                decoder.sequence.entropy          = 0.0;
                decoder.sequence.score            = -INFINITY;

                decoder.seek_delta = 100*WHISPER_CHUNK_SIZE;

                decoder.failed    = false;
                decoder.completed = false;
                decoder.has_ts    = false;
            }

            // init prompt and kv cache for the current iteration
            // run whisper_decoder() only for decoder 0 and copy the results for the other decoders
            {
                prompt.clear();

                // if we have already generated some text, use it as a prompt to condition the next generation
                if (!prompt_past.empty() && t_cur < 0.5f && params.n_max_text_ctx > 0) {
                    int n_take = std::min(std::min(params.n_max_text_ctx, whisper_n_text_ctx(ctx)/2), int(prompt_past.size()));

                    prompt = { whisper_token_prev(ctx) };
                    prompt.insert(prompt.begin() + 1, prompt_past.end() - n_take, prompt_past.end());
                }

                // init new transcription with sot, language (opt) and task tokens
                prompt.insert(prompt.end(), prompt_init.begin(), prompt_init.end());

                // print the prompt
                WHISPER_PRINT_DEBUG("\n\n");
                for (int i = 0; i < (int) prompt.size(); i++) {
                    WHISPER_PRINT_DEBUG("%s: prompt[%d] = %s\n", __func__, i, ctx->vocab.id_to_token.at(prompt[i]).c_str());
                }
                WHISPER_PRINT_DEBUG("\n\n");

                if (!whisper_decode_internal(*ctx, *state, state->decoders[0], prompt.data(), prompt.size(), 0, params.n_threads, params.abort_callback, params.abort_callback_user_data)) {
                    log("%s: failed to decode\n", __func__);
                    return -7;
                }

                {
                    const int64_t t_start_sample_us = ggml_time_us();

                    whisper_process_logits(*ctx, *state, params, state->decoders[0], t_cur);

                    state->decoders[0].kv_self.n += prompt.size();

                    for (int j = 1; j < n_decoders_cur; ++j) {
                        auto & decoder = state->decoders[j];

                        memcpy(decoder.kv_self.k->data, state->decoders[0].kv_self.k->data, ggml_nbytes(decoder.kv_self.k));
                        memcpy(decoder.kv_self.v->data, state->decoders[0].kv_self.v->data, ggml_nbytes(decoder.kv_self.v));

                        decoder.kv_self.n += prompt.size();

                        memcpy(decoder.probs.data(),    state->decoders[0].probs.data(),    decoder.probs.size()*sizeof(decoder.probs[0]));
                        memcpy(decoder.logits.data(),   state->decoders[0].logits.data(),   decoder.logits.size()*sizeof(decoder.logits[0]));
                        memcpy(decoder.logprobs.data(), state->decoders[0].logprobs.data(), decoder.logprobs.size()*sizeof(decoder.logprobs[0]));
                    }

                    state->t_sample_us += ggml_time_us() - t_start_sample_us;
                }
            }

            for (int i = 0, n_max = whisper_n_text_ctx(ctx)/2 - 4; i < n_max; ++i) {
                const int64_t t_start_sample_us = ggml_time_us();

                if (params.strategy == whisper_sampling_strategy::WHISPER_SAMPLING_BEAM_SEARCH) {
                    beam_candidates.clear();
                }

                // generate new sequence candidates for each decoder
                for (int j = 0; j < n_decoders_cur; ++j) {
                    auto & decoder = state->decoders[j];

                    if (decoder.completed || decoder.failed) {
                        continue;
                    }

                    switch (params.strategy) {
                        case whisper_sampling_strategy::WHISPER_SAMPLING_GREEDY:
                            {
                                if (t_cur < 1e-6f) {
                                    decoder.sequence.tokens.push_back(whisper_sample_token(*ctx, *state, decoder, true));
                                } else {
                                    decoder.sequence.tokens.push_back(whisper_sample_token(*ctx, *state, decoder, false));
                                }

                                decoder.sequence.sum_logprobs_all += decoder.sequence.tokens.back().plog;
                            } break;
                        case whisper_sampling_strategy::WHISPER_SAMPLING_BEAM_SEARCH:
                            {
                                const auto tokens_new = whisper_sample_token_topk(*ctx, *state, decoder, params.beam_search.beam_size);

                                for (const auto & token : tokens_new) {
                                    beam_candidates.push_back({ j, decoder.seek_delta, decoder.has_ts, decoder.sequence });
                                    beam_candidates.back().sequence.tokens.push_back(token);
                                    beam_candidates.back().sequence.sum_logprobs_all += token.plog;

                                    //WHISPER_PRINT_DEBUG("%s: beam candidate: %s (%f, %f)\n", __func__, ctx->vocab.id_to_token.at(token.id).c_str(), token.plog, beam_candidates.back().sequence.sum_logprobs_all);
                                }
                            } break;
                    };
                }

                // for beam-search, choose the top candidates and update the KV caches
                if (params.strategy == whisper_sampling_strategy::WHISPER_SAMPLING_BEAM_SEARCH) {
                    std::sort(
                            beam_candidates.begin(),
                            beam_candidates.end(),
                            [](const beam_candidate & a, const beam_candidate & b) {
                        return a.sequence.sum_logprobs_all > b.sequence.sum_logprobs_all;
                    });

                    uint32_t cur_c = 0;
                    std::vector<int> decoder_idx(n_decoders_cur, -1);

                    for (int j = 0; j < n_decoders_cur; ++j) {
                        auto & decoder = state->decoders[j];

                        if (decoder.completed || decoder.failed) {
                            continue;
                        }

                        auto & cur = beam_candidates[cur_c++];

                        while (beam_candidates.size() > cur_c && beam_candidates[cur_c].sequence.sum_logprobs_all == cur.sequence.sum_logprobs_all && i > 0) {
                            ++cur_c;
                        }

                        decoder.sequence   = cur.sequence;
                        decoder.seek_delta = cur.seek_delta;
                        decoder.has_ts     = cur.has_ts;

                        decoder_idx[j] = cur.decoder_idx;
                        WHISPER_PRINT_DEBUG("%s: beam search: decoder %d: from decoder %d: token = %10s, plog = %8.5f, sum_logprobs = %8.5f\n",
                                __func__, j, cur.decoder_idx, ctx->vocab.id_to_token.at(decoder.sequence.tokens.back().id).c_str(), decoder.sequence.tokens.back().plog, decoder.sequence.sum_logprobs_all);
                    }

                    // update KV caches
                    whisper_kv_swap_fast(decoder_idx, state->decoders, state->kv_swap_bufs, n_decoders_cur);
                }

                // update the decoder state
                // - check if the sequence is completed
                // - check if the sequence is failed
                // - update sliding window based on timestamp tokens
                for (int j = 0; j < n_decoders_cur; ++j) {
                    auto & decoder = state->decoders[j];

                    if (decoder.completed || decoder.failed) {
                        continue;
                    }

                    auto & has_ts     = decoder.has_ts;
                    auto & failed     = decoder.failed;
                    auto & completed  = decoder.completed;
                    auto & seek_delta = decoder.seek_delta;
                    auto & result_len = decoder.sequence.result_len;

                    {
                        const auto & token = decoder.sequence.tokens.back();

                        // timestamp token - update sliding window
                        if (token.id > whisper_token_beg(ctx)) {
                            const int seek_delta_new = 2*(token.id - whisper_token_beg(ctx));

                            // do not allow to go back in time
                            if (has_ts && seek_delta > seek_delta_new && result_len < i) {
                                failed = true; // TODO: maybe this is not a failure ?
                                continue;
                            }

                            seek_delta = seek_delta_new;
                            result_len = i + 1;
                            has_ts = true;
                        }

```

This is a C++ program that appears to implement a simple text-to-speech (TTS) system that uses a speech-synthesis engine to convert text into speech. The TTS system has a few different modes for converting text, such as a simple real-time (RTT) mode, a timestamps mode, and a new-segment-based mode.

The program takes in several parameters when it starts up, such as the speed-up factor for TTS, the number of seconds to keep the audio窗口 open, and a callback for when the TTS engine should start using the new-segment-based mode.

The TTS engine uses two different approaches to convert text into speech, depending on the mode:

* In the RTT mode, the TTS engine reads the text and displays it to the user. It also plays the audio in real-time using the fast-forward method.
* In the timestamps mode, the TTS engine reads the text and displays it to the user. It also records the timestamp for each segment of the speech.
* In the new-segment-based mode, the TTS engine reads the text and displays it to the user. It allows the user to split the text into smaller segments and then plays each segment to the user. The TTS engine also uses the new-segment-based mode to determine when to start using this approach.

The program also has a function that updates the audio window to reflect the changes in the TTS engine. This function is called every frame, and it updates the position of the audio waveform based on the seek value and the buffer size.

Overall, the program appears to be a simple and effective TTS system that can convert text into speech in different modes.


```cpp
#ifdef WHISPER_DEBUG
                        {
                            const auto tt = token.pt > 0.10 ? ctx->vocab.id_to_token.at(token.tid) : "[?]";
                            WHISPER_PRINT_DEBUG("%s: id = %3d, decoder = %d, token = %6d, p = %6.3f, ts = %10s, %6.3f, result_len = %4d '%s'\n",
                                    __func__, i, j, token.id, token.p, tt.c_str(), token.pt, result_len, ctx->vocab.id_to_token.at(token.id).c_str());
                        }
#endif

                        // end of segment
                        if (token.id == whisper_token_eot(ctx) ||               // end of text token
                           (params.max_tokens > 0 && i >= params.max_tokens) || // max tokens per segment reached
                           (has_ts && seek + seek_delta + 100 >= seek_end)      // end of audio reached
                           ) {
                            if (result_len == 0) {
                                if (seek + seek_delta + 100 >= seek_end) {
                                    result_len = i + 1;
                                } else {
                                    failed = true;
                                    continue;
                                }
                            }

                            if (params.single_segment) {
                                result_len = i + 1;
                                seek_delta = 100*WHISPER_CHUNK_SIZE;
                            }

                            completed = true;
                            continue;
                        }

                        // TESTS: if no tensors are loaded, it means we are running tests
                        if (ctx->model.n_loaded == 0) {
                            seek_delta = 100*WHISPER_CHUNK_SIZE;
                            completed = true;
                            continue;
                        }
                    }

                    // sometimes, the decoding can get stuck in a repetition loop
                    // this is an attempt to mitigate such cases - we flag the decoding as failed and use a fallback strategy
                    if (i == n_max - 1 && (result_len == 0 || seek_delta < 100*WHISPER_CHUNK_SIZE/2)) {
                        failed = true;
                        continue;
                    }
                }

                // check if all decoders have finished (i.e. completed or failed)
                {
                    bool completed_all = true;

                    for (int j = 0; j < n_decoders_cur; ++j) {
                        auto & decoder = state->decoders[j];

                        if (decoder.completed || decoder.failed) {
                            continue;
                        }

                        completed_all = false;
                    }

                    if (completed_all) {
                        break;
                    }
                }

                state->t_sample_us += ggml_time_us() - t_start_sample_us;

                // obtain logits for the next token
                for (int j = 0; j < n_decoders_cur; ++j) {
                    auto & decoder = state->decoders[j];

                    if (decoder.failed || decoder.completed) {
                        continue;
                    }

                    decoder.tokens_tmp.resize(1);
                    decoder.tokens_tmp[0] = decoder.sequence.tokens.back().id;

                    //WHISPER_PRINT_DEBUG("%s: decoder %d: token %d, kv_self.n %d, seek_delta %d\n", __func__, j, decoder.tokens_tmp[0], decoder.kv_self.n, decoder.seek_delta);

                    if (!whisper_decode_internal(*ctx, *state, decoder, decoder.tokens_tmp.data(), decoder.tokens_tmp.size(), decoder.kv_self.n, params.n_threads, params.abort_callback, params.abort_callback_user_data)) {
                        log("%s: failed to decode\n", __func__);
                        return -8;
                    }

                    {
                        const int64_t t_start_sample_us = ggml_time_us();

                        whisper_process_logits(*ctx, *state, params, decoder, t_cur);

                        ++decoder.kv_self.n;

                        state->t_sample_us += ggml_time_us() - t_start_sample_us;
                    }
                }
            }

            // rank the resulting sequences and select the best one
            {
                double best_score = -INFINITY;

                for (int j = 0; j < n_decoders_cur; ++j) {
                    auto & decoder = state->decoders[j];

                    if (decoder.failed) {
                        continue;
                    }

                    decoder.sequence.tokens.resize(decoder.sequence.result_len);
                    whisper_sequence_score(params, decoder.sequence);

                    WHISPER_PRINT_DEBUG("%s: decoder %2d: score = %8.5f, result_len = %3d, avg_logprobs = %8.5f, entropy = %8.5f\n",
                            __func__, j, decoder.sequence.score, decoder.sequence.result_len, decoder.sequence.avg_logprobs, decoder.sequence.entropy);

                    if (decoder.sequence.result_len > 32 && decoder.sequence.entropy < params.entropy_thold) {
                        WHISPER_PRINT_DEBUG("%s: decoder %2d: failed due to entropy %8.5f < %8.5f\n",
                                __func__, j, decoder.sequence.entropy, params.entropy_thold);

                        decoder.failed = true;
                        state->n_fail_h++;

                        continue;
                    }

                    if (best_score < decoder.sequence.score) {
                        best_score = decoder.sequence.score;
                        best_decoder_id = j;
                    }
                }

                WHISPER_PRINT_DEBUG("%s: best decoder = %d\n", __func__, best_decoder_id);
            }

            // was the decoding successful for the current temperature?
            // do fallback only if:
            // - we are not at the last temperature
            // - we are not at the end of the audio (3 sec)
            if (it != (int) temperatures.size() - 1 &&
                seek_end - seek > 10*WHISPER_CHUNK_SIZE) {
                bool success = true;

                const auto & decoder = state->decoders[best_decoder_id];

                if (decoder.failed || decoder.sequence.avg_logprobs < params.logprob_thold) {
                    success = false;
                    state->n_fail_p++;
                }

                if (success) {
                    //for (auto & token : ctx->decoders[best_decoder_id].sequence.tokens) {
                    //    WHISPER_PRINT_DEBUG("%s: token = %d, p = %6.3f, pt = %6.3f, ts = %s, str = %s\n", __func__, token.id, token.p, token.pt, ctx->vocab.id_to_token.at(token.tid).c_str(), ctx->vocab.id_to_token.at(token.id).c_str());
                    //}

                    break;
                }
            }

            WHISPER_PRINT_DEBUG("\n%s: failed to decode with temperature = %.2f\n", __func__, t_cur);
        }

        // output results through a user-provided callback
        {
            const auto & best_decoder = state->decoders[best_decoder_id];

            const auto seek_delta = best_decoder.seek_delta;
            const auto result_len = best_decoder.sequence.result_len;

            const auto & tokens_cur = best_decoder.sequence.tokens;

            //WHISPER_PRINT_DEBUG("prompt_init.size() = %d, prompt.size() = %d, result_len = %d, seek_delta = %d\n", prompt_init.size(), prompt.size(), result_len, seek_delta);

            // update prompt_past
            prompt_past.clear();
            if (prompt.front() == whisper_token_prev(ctx)) {
                prompt_past.insert(prompt_past.end(), prompt.begin() + 1, prompt.end() - prompt_init.size());
            }

            for (int i = 0; i < result_len; ++i) {
                prompt_past.push_back(tokens_cur[i].id);
            }

            if (!tokens_cur.empty() && ctx->model.n_loaded > 0) {
                int  i0 = 0;
                auto t0 = seek + 2*(tokens_cur.front().tid - whisper_token_beg(ctx));

                std::string text;
                bool speaker_turn_next = false;

                for (int i = 0; i < (int) tokens_cur.size(); i++) {
                    //printf("%s: %18s %6.3f %18s %6.3f\n", __func__,
                    //        ctx->vocab.id_to_token[tokens_cur[i].id].c_str(), tokens_cur[i].p,
                    //        ctx->vocab.id_to_token[tokens_cur[i].tid].c_str(), tokens_cur[i].pt);

                    if (params.print_special || tokens_cur[i].id < whisper_token_eot(ctx)) {
                        text += whisper_token_to_str(ctx, tokens_cur[i].id);
                    }

                    // [TDRZ] record if speaker turn was predicted after current segment
                    if (params.tdrz_enable && tokens_cur[i].id == whisper_token_solm(ctx)) {
                        speaker_turn_next = true;
                    }

                    if (tokens_cur[i].id > whisper_token_beg(ctx) && !params.single_segment) {
                        const auto t1 = seek + 2*(tokens_cur[i].tid - whisper_token_beg(ctx));

                        if (!text.empty()) {
                            const auto tt0 = params.speed_up ? 2*t0 : t0;
                            const auto tt1 = params.speed_up ? 2*t1 : t1;

                            if (params.print_realtime) {
                                if (params.print_timestamps) {
                                    printf("[%s --> %s]  %s\n", to_timestamp(tt0).c_str(), to_timestamp(tt1).c_str(), text.c_str());
                                } else {
                                    printf("%s", text.c_str());
                                    fflush(stdout);
                                }
                            }

                            //printf("tt0 = %d, tt1 = %d, text = %s, token = %s, token_id = %d, tid = %d\n", tt0, tt1, text.c_str(), ctx->vocab.id_to_token[tokens_cur[i].id].c_str(), tokens_cur[i].id, tokens_cur[i].tid);

                            result_all.push_back({ tt0, tt1, text, {}, speaker_turn_next });
                            for (int j = i0; j <= i; j++) {
                                result_all.back().tokens.push_back(tokens_cur[j]);
                            }

                            int n_new = 1;

                            if (params.token_timestamps) {
                                whisper_exp_compute_token_level_timestamps(
                                        *ctx, *state, result_all.size() - 1, params.thold_pt, params.thold_ptsum);

                                if (params.max_len > 0) {
                                    n_new = whisper_wrap_segment(*ctx, *state, params.max_len, params.split_on_word);
                                }
                            }
                            if (params.new_segment_callback) {
                                params.new_segment_callback(ctx, state, n_new, params.new_segment_callback_user_data);
                            }
                        }
                        text = "";
                        while (i < (int) tokens_cur.size() && tokens_cur[i].id > whisper_token_beg(ctx)) {
                            i++;
                        }
                        i--;
                        t0 = t1;
                        i0 = i + 1;
                        speaker_turn_next = false;
                    }
                }

                if (!text.empty()) {
                    const auto t1 = seek + seek_delta;

                    const auto tt0 = params.speed_up ? 2*t0 : t0;
                    const auto tt1 = params.speed_up ? 2*t1 : t1;

                    if (params.print_realtime) {
                        if (params.print_timestamps) {
                            printf("[%s --> %s]  %s\n", to_timestamp(tt0).c_str(), to_timestamp(tt1).c_str(), text.c_str());
                        } else {
                            printf("%s", text.c_str());
                            fflush(stdout);
                        }
                    }

                    result_all.push_back({ tt0, tt1, text, {} , speaker_turn_next });
                    for (int j = i0; j < (int) tokens_cur.size(); j++) {
                        result_all.back().tokens.push_back(tokens_cur[j]);
                    }

                    int n_new = 1;

                    if (params.token_timestamps) {
                        whisper_exp_compute_token_level_timestamps(
                                *ctx, *state, result_all.size() - 1, params.thold_pt, params.thold_ptsum);

                        if (params.max_len > 0) {
                            n_new = whisper_wrap_segment(*ctx, *state, params.max_len, params.split_on_word);
                        }
                    }
                    if (params.new_segment_callback) {
                        params.new_segment_callback(ctx, state, n_new, params.new_segment_callback_user_data);
                    }
                }
            }

            // update audio window
            seek += seek_delta;

            WHISPER_PRINT_DEBUG("seek = %d, seek_delta = %d\n", seek, seek_delta);
        }
    }

    return 0;
}

```

This function appears to be part of a larger audio processing pipeline. It takes in a SegmentedAudio object and multiple callback functions as input, and outputs a processed audio object.

The function first initializes the audio cluster and then loops through each state of the audio cluster, processing each state according to the corresponding callback function. The audio cluster is represented as an array of SegmentedAudio objects, each containing information about the audio for a particular segment.

The function uses several helper functions to calculate statistics about the audio, such as the total number of samples, the number of samples per processor, and the number of samples in each chunk. It also uses a helper function to convert timestamps to sample times.

The function also includes a loop to log information about the audio boundaries. This log may be useful for debugging purposes.

Overall, the function appears to be a key part of the audio processing pipeline, responsible for processing and preparing audio for further processing.


```cpp
int whisper_full(
        struct whisper_context * ctx,
    struct whisper_full_params   params,
                   const float * samples,
                           int   n_samples) {
    return whisper_full_with_state(ctx, ctx->state, params, samples, n_samples);
}

int whisper_full_parallel(
        struct whisper_context * ctx,
        struct whisper_full_params params,
        const float * samples,
        int n_samples,
        int n_processors) {
    if (n_processors == 1) {
        return whisper_full(ctx, params, samples, n_samples);
    }
    int ret = 0;

    // prepare separate states for each thread
    std::vector<whisper_state*> states;

    const int offset_samples = (WHISPER_SAMPLE_RATE*params.offset_ms)/1000;
    const int n_samples_per_processor = (n_samples - offset_samples)/n_processors;

    // the calling thread will process the first chunk
    // while the other threads will process the remaining chunks

    std::vector<std::thread> workers(n_processors - 1);
    for (int i = 0; i < n_processors - 1; ++i) {
        // create a new state for each thread
        states.push_back(whisper_init_state(ctx));

        const int start_samples = offset_samples + (i + 1)*n_samples_per_processor;
        const int n_samples_cur = (i == n_processors - 2) ? n_samples - start_samples : n_samples_per_processor;

        auto params_cur = params;

        params_cur.offset_ms = 0;
        params_cur.print_progress = false;
        params_cur.print_realtime = false;

        params_cur.new_segment_callback = nullptr;
        params_cur.new_segment_callback_user_data = nullptr;

        params_cur.progress_callback = nullptr;
        params_cur.progress_callback_user_data = nullptr;

        workers[i] = std::thread(whisper_full_with_state, ctx, states[i], std::move(params_cur), samples + start_samples, n_samples_cur);
    }

    {
        auto params_cur = params;

        // We need to disable the print real-time for this one as well, otherwise it will show only for the first chunk.
        params_cur.print_realtime = false;

        // Run the first transformation using default state but only for the first chunk.
        ret = whisper_full_with_state(ctx, ctx->state, std::move(params_cur), samples, offset_samples + n_samples_per_processor);
    }

    for (int i = 0; i < n_processors - 1; ++i) {
        workers[i].join();
    }

    const int64_t offset_t = (int64_t) params.offset_ms/10.0;

    // combine results into result_state->result_all from all other states
    for (int i = 0; i < n_processors - 1; ++i) {
        auto& results_i = states[i]->result_all;

        for (auto& result : results_i) {
            // correct the segment timestamp taking into account the offset
            result.t0 += 100 * ((i + 1) * n_samples_per_processor) / WHISPER_SAMPLE_RATE + offset_t;
            result.t1 += 100 * ((i + 1) * n_samples_per_processor) / WHISPER_SAMPLE_RATE + offset_t;

            // make sure that segments are not overlapping
            if (!ctx->state->result_all.empty()) {
                result.t0 = std::max(result.t0, ctx->state->result_all.back().t1);
            }

            ctx->state->result_all.push_back(std::move(result));

            // call the new_segment_callback for each segment
            if (params.new_segment_callback) {
                params.new_segment_callback(ctx, ctx->state, 1, params.new_segment_callback_user_data);
            }
        }

        ctx->state->t_mel_us += states[i]->t_mel_us;

        ctx->state->t_sample_us += states[i]->t_sample_us;
        ctx->state->t_encode_us += states[i]->t_encode_us;
        ctx->state->t_decode_us += states[i]->t_decode_us;
        ctx->state->t_prompt_us += states[i]->t_prompt_us;

        ctx->state->n_sample += states[i]->n_sample;
        ctx->state->n_encode += states[i]->n_encode;
        ctx->state->n_decode += states[i]->n_decode;
        ctx->state->n_prompt += states[i]->n_prompt;

        whisper_free_state(states[i]);
    }

    // average the timings
    ctx->state->t_mel_us    /= n_processors;
    ctx->state->t_sample_us /= n_processors;
    ctx->state->t_encode_us /= n_processors;
    ctx->state->t_decode_us /= n_processors;

    // print information about the audio boundaries
    log("\n");
    log("%s: the audio has been split into %d chunks at the following times:\n", __func__, n_processors);
    for (int i = 0; i < n_processors - 1; ++i) {
        log("%s: split %d - %s\n", __func__, (i + 1), to_timestamp(100*((i + 1)*n_samples_per_processor)/WHISPER_SAMPLE_RATE + offset_t).c_str());
    }
    log("%s: the transcription quality may be degraded near these boundaries\n", __func__);

    return ret;
}

```

以上代码定义了四个函数，分别接收一个whisper_state结构和whisper_context结构作为参数，并返回相应的值。

* whisper_full_n_segments_from_state函数接收一个whisper_state结构，返回该结构中result_all成员的大小。result_all成员表示whisper_state结构中与问题相关的结果数量，如问题类型数、生成的解数等。
* whisper_full_n_segments函数接收一个whisper_context结构，返回该结构中result_all成员的大小。result_all成员同上，但whisper_context结构中包含的是whisper_state结构，需要通过state->result_all获取。
* whisper_full_lang_id_from_state函数接收一个whisper_state结构，返回该结构中lang_id成员的值。lang_id成员表示问题类型，如是ENGLTRANSC issues，则是问题类型数，否则是另外的类型。
* whisper_full_lang_id函数接收一个whisper_context结构，返回该结构中lang_id成员的值。与whisper_full_lang_id_from_state函数类似，但需要通过state->lang_id获取。

函数的名称和参数如上，但源代码没有给出具体的实现，无法进一步分析每个函数的功能。


```cpp
int whisper_full_n_segments_from_state(struct whisper_state * state) {
    return state->result_all.size();
}

int whisper_full_n_segments(struct whisper_context * ctx) {
    return ctx->state->result_all.size();
}

int whisper_full_lang_id_from_state(struct whisper_state * state) {
    return state->lang_id;
}

int whisper_full_lang_id(struct whisper_context * ctx) {
    return ctx->state->lang_id;
}

```

这是一个C语言中定义的函数，定义了在不同情况下如何从状态结构中获取不同分量的值。这里，state结构体是一个whisper_state类型的变量，它存储了whisper_ctx结构体中的state成员。

函数whisper_full_get_segment_t0_from_state的作用是从state结构体中取出第i_segment个分量的值，并将其存储在return类型的变量中。

同样地，函数whisper_full_get_segment_t0的作用是从ctx结构体中取出第i_segment个分量的值，并将其存储在return类型的变量中。

第三个函数whisper_full_get_segment_t1_from_state的作用是从state结构体中取出第i_segment个分量的值，并将其存储在return类型的变量中。

最后一个函数whisper_full_get_segment_t1的作用是从ctx结构体中取出第i_segment个分量的值，并将其存储在return类型的变量中。


```cpp
int64_t whisper_full_get_segment_t0_from_state(struct whisper_state * state, int i_segment) {
    return state->result_all[i_segment].t0;
}

int64_t whisper_full_get_segment_t0(struct whisper_context * ctx, int i_segment) {
    return ctx->state->result_all[i_segment].t0;
}

int64_t whisper_full_get_segment_t1_from_state(struct whisper_state * state, int i_segment) {
    return state->result_all[i_segment].t1;
}

int64_t whisper_full_get_segment_t1(struct whisper_context * ctx, int i_segment) {
    return ctx->state->result_all[i_segment].t1;
}

```

以上代码是Whisper库中的函数函数指针。

Whisper是一个用于语音识别的库，提供了对语音信号进行处理的功能。这个库中定义了以下函数：

* `whisper_full_get_segment_speaker_turn_next_from_state`：从whisper_state结构中获取segment_speaker_turn_next的函数，用于获取说话者轮到的下一个状态。
* `whisper_full_get_segment_speaker_turn_next`：从whisper_context结构中获取segment_speaker_turn_next的函数，用于获取说话者轮到的下一个状态。
* `whisper_full_get_segment_text_from_state`：从whisper_state结构中获取segment_text的函数，用于获取说话者轮到的文本。
* `whisper_full_get_segment_text`：从whisper_context结构中获取segment_text的函数，用于获取说话者轮到的文本。

这些函数用于获取whisper库中状态（state）中的segment_speaker_turn_next和segment_text，并返回给用户。


```cpp
bool whisper_full_get_segment_speaker_turn_next_from_state(struct whisper_state * state, int i_segment) {
    return state->result_all[i_segment].speaker_turn_next;
}

bool whisper_full_get_segment_speaker_turn_next(struct whisper_context * ctx, int i_segment) {
    return ctx->state->result_all[i_segment].speaker_turn_next;
}

const char * whisper_full_get_segment_text_from_state(struct whisper_state * state, int i_segment) {
    return state->result_all[i_segment].text.c_str();
}

const char * whisper_full_get_segment_text(struct whisper_context * ctx, int i_segment) {
    return ctx->state->result_all[i_segment].text.c_str();
}

```

这两组函数定义了Whisper状态中的一个名为"result_all"的数组的元素，以及该数组中第i个元素的类型。

具体来说，第一个函数的作用是返回一个整数，表示从Whisper状态的"result_all"数组中包含第i个元素，该元素的值类型为整数类型。

第二个函数的作用是返回一个整数，表示从Whisper状态的"state"数组中包含第i个元素的值类型为整数类型的数组中的元素个数。

第三个函数的作用是返回一个字符串，表示从Whisper状态的"result_all"数组中包含第i个元素的值类型为整数类型的数组中该元素的文本内容。

第四个函数的作用是返回一个字符串，表示从Whisper状态的"state"数组中包含第i个元素的值类型为整数类型的数组中该元素的文本内容。


```cpp
int whisper_full_n_tokens_from_state(struct whisper_state * state, int i_segment) {
    return state->result_all[i_segment].tokens.size();
}

int whisper_full_n_tokens(struct whisper_context * ctx, int i_segment) {
    return ctx->state->result_all[i_segment].tokens.size();
}

const char * whisper_full_get_token_text_from_state(struct whisper_context * ctx, struct whisper_state * state, int i_segment, int i_token) {
    return ctx->vocab.id_to_token[state->result_all[i_segment].tokens[i_token].id].c_str();
}

const char* whisper_full_get_token_text(struct whisper_context * ctx, int i_segment, int i_token) {
    return ctx->vocab.id_to_token[ctx->state->result_all[i_segment].tokens[i_token].id].c_str();
}

```

这是一个C语言中的函数，它实现了从状态对象中获取语音识别token的ID。这个函数是在一个名为whisper_token的函数中定义的，它在不同的文件中都有定义。同时，这个函数还有两个从同一个文件中定义的函数whisper_full_get_token_id_from_state和whisper_full_get_token_data_from_state，它们都接受一个whisper_state类型的参数和一个int类型的i_segment和i_token参数。

具体来说，这个函数的作用是接收一个whisper_state类型的结构体，然后从中提取出i_segment位置的token对象，并返回该对象的id值。通过这个函数，我们可以使用统一的接口来获取whisper_token和whisper_full_get_token_data_from_state函数中的token对象。另外，这个函数还能够将whisper_token和whisper_full_get_token_data函数作为实参，以更灵活的方式获取token的ID。


```cpp
whisper_token whisper_full_get_token_id_from_state(struct whisper_state * state, int i_segment, int i_token) {
    return state->result_all[i_segment].tokens[i_token].id;
}

whisper_token whisper_full_get_token_id(struct whisper_context * ctx, int i_segment, int i_token) {
    return ctx->state->result_all[i_segment].tokens[i_token].id;
}

struct whisper_token_data whisper_full_get_token_data_from_state(struct whisper_state * state, int i_segment, int i_token) {
    return state->result_all[i_segment].tokens[i_token];
}

struct whisper_token_data whisper_full_get_token_data(struct whisper_context * ctx, int i_segment, int i_token) {
    return ctx->state->result_all[i_segment].tokens[i_token];
}

```

这是一个C语言代码，定义了两个函数，分别是 "whisper_full_get_token_p_from_state" 和 "whisper_full_get_token_p"。这两个函数的作用是获取 whisper 模型中特定位置的词元（token）的拼音值。

这两个函数都是通过输入一个 whisper 状态结构体，并返回一个浮点数，来实现的。具体来说，第一个函数接收一个 whisper 状态结构体和一个词元（token）的索引，返回该词元在状态结构体中的拼音值。第二个函数与第一个函数类似，但使用了一个通用的 "ctx" 变量，而不是一个特定的 whisper 状态结构体。

可以认为，这两个函数用于在 whisper 模型中获取特定位置的词元拼音值。在实际应用中，通过这两个函数，可以更方便地获取 whisper 模型中的数据。


```cpp
float whisper_full_get_token_p_from_state(struct whisper_state * state, int i_segment, int i_token) {
    return state->result_all[i_segment].tokens[i_token].p;
}

float whisper_full_get_token_p(struct whisper_context * ctx, int i_segment, int i_token) {
    return ctx->state->result_all[i_segment].tokens[i_token].p;
}

// =================================================================================================

//
// Temporary interface needed for exposing ggml interface
// Will be removed in the future when ggml becomes a separate library
//

```

This is a C function that calculates the memory usage of a game using the Linux "ggml" profiler. The profiler generates a profiling report that includes information about the time spent by different CPU activities in the game. The function takes as input the path to the profiler report and the number of threads used to collect the report.

The function first initializes the ggml time library and sets the profiler to collect the report for the given number of threads. It then allocates memory for a buffer of size 256 and creates a copy of the input report.

The function then reads the profiler report and calculates the total time spent by different CPU activities. It does this by creating a loop that reads the profile for each thread, calculate the elapsed time for each activity, and add the results to the total time.

Finally, the function constructs a string from the input report and returns it. Note that the memory usage information is shown in a human-readable format instead of a timestamp.


```cpp
WHISPER_API int whisper_bench_memcpy(int n_threads) {
    fputs(whisper_bench_memcpy_str(n_threads), stderr);
    return 0;
}

WHISPER_API const char * whisper_bench_memcpy_str(int n_threads) {
    static std::string s;
    s = "";
    char strbuf[256];

    ggml_time_init();

    size_t n    = 20;
    size_t arr  = n_threads > 0 ? 1024llu : n_threads; // trick to avoid compiler optimizations

    // 1GB MB array
    const size_t size = arr*1024llu*1024llu;

    // single-thread
    {
        char * src = (char *) malloc(size);
        char * dst = (char *) malloc(size);

        for (size_t i = 0; i < size; i++) src[i] = i;

        memcpy(dst, src, size); // heat-up

        double tsum = 0.0;
        double sum  = 0.0;

        for (size_t i = 0; i < n; i++) {
            const int64_t t0 = ggml_time_us();

            memcpy(dst, src, size);

            const int64_t t1 = ggml_time_us();

            tsum += (t1 - t0)*1e-6;

            src[rand() % size] = rand() % 256;
        }

        snprintf(strbuf, sizeof(strbuf), "memcpy: %.2f GB/s (1 thread)\n", (double) (n*size)/(tsum*1024llu*1024llu*1024llu));
        s += strbuf;

        // needed to prevent the compiler from optimizing the memcpy away
        {
            for (size_t i = 0; i < size; i++) sum += dst[i];

            snprintf(strbuf, sizeof(strbuf), "sum:    %f\n", sum);
            s += strbuf;
        }

        free(src);
        free(dst);
    }

    return s.c_str();
}

```

This is a C function that outputs the QoS (Queue-to-Queue) performance of a parallelism-based workflow in the AMPLify platform. It appears to calculate the QoS performance for a workflow with different data types (Queue A, Queue B, and Queue C) and for different execution durations (using a loop).

The function takes as input the number of processes (N), the number of data types (N), and the QoS metrics for each data type. It then calculates the QoS metrics for each run and the total number of runs, and outputs the results.

The function uses two helper function to calculate the QoS metrics: `ggml_time_us()` to get the timestamp in microseconds for each run, and `qiskit_portal_qclass_qos()` to get the QoS metrics for each data type.

The function also includes a loop to run the calculation for each execution duration. This loop is conditioned by whether the total number of runs is greater than 3, indicating that at least 3 runs are necessary to obtain an accurate QoS metric.

Overall, the function seems to be a useful tool for evaluating the QoS performance of a parallelism-based workflow in the AMPLify platform.


```cpp
WHISPER_API int whisper_bench_ggml_mul_mat(int n_threads) {
    fputs(whisper_bench_ggml_mul_mat_str(n_threads), stderr);
    return 0;
}

WHISPER_API const char * whisper_bench_ggml_mul_mat_str(int n_threads) {
    static std::string s;
    s = "";
    char strbuf[256];

    ggml_time_init();

    const int n_max = 128;

    const std::vector<size_t> sizes = {
        64, 128, 256, 512, 1024, 2048, 4096,
    };

    const size_t N_max = sizes.back();

    // a: N*N*sizeof(float)
    // b: N*N*sizeof(float)
    // c: N*N*sizeof(float)
    // when F16 is used, there is an extra work buffer of size N*N*sizeof(float)
    std::vector<uint8_t> buf(3llu*N_max*N_max*sizeof(float) + 3*ggml_tensor_overhead() + ggml_graph_overhead());
    std::vector<uint8_t> work;

    // put a bunch of random data in the buffer
    for (size_t i = 0; i < buf.size(); i++) buf[i] = i;

    for (int j = 0; j < (int) sizes.size(); j++) {
        int n_q4_0 = 0;
        int n_q4_1 = 0;
        int n_q5_0 = 0;
        int n_q5_1 = 0;
        int n_q8_0 = 0;
        int n_fp16 = 0;
        int n_fp32 = 0;

        // GFLOPS/s
        double s_q4_0 = 0.0;
        double s_q4_1 = 0.0;
        double s_q5_0 = 0.0;
        double s_q5_1 = 0.0;
        double s_q8_0 = 0.0;
        double s_fp16 = 0.0;
        double s_fp32 = 0.0;

        const size_t N = sizes[j];

        for (int k = 0; k < 7; ++k) {
            const ggml_type wtype =
                k == 0 ? GGML_TYPE_Q4_0 :
                k == 1 ? GGML_TYPE_Q4_1 :
                k == 2 ? GGML_TYPE_Q5_0 :
                k == 3 ? GGML_TYPE_Q5_1 :
                k == 4 ? GGML_TYPE_Q8_0 :
                k == 5 ? GGML_TYPE_F16  : GGML_TYPE_F32;

            double & s = k == 0 ? s_q4_0 : k == 1 ? s_q4_1 : k == 2 ? s_q5_0 : k == 3 ? s_q5_1 : k == 4 ? s_q8_0 : k == 5 ? s_fp16 : /*k == 6*/ s_fp32;
            int    & n = k == 0 ? n_q4_0 : k == 1 ? n_q4_1 : k == 2 ? n_q5_0 : k == 3 ? n_q5_1 : k == 4 ? n_q8_0 : k == 5 ? n_fp16 : /*k == 6*/ n_fp32;

            struct ggml_init_params gparams = {
                /*.mem_size   =*/ buf.size(),
                /*.mem_buffer =*/ buf.data(),
                /*.no_alloc   =*/ false,
            };

            struct ggml_context * ctx0 = ggml_init(gparams);

            struct ggml_tensor * a = ggml_new_tensor_2d(ctx0, wtype,         N, N);
            struct ggml_tensor * b = ggml_new_tensor_2d(ctx0, GGML_TYPE_F32, N, N);

            struct ggml_tensor * c = ggml_mul_mat(ctx0, a, b);

            struct ggml_cgraph * gf = ggml_new_graph(ctx0);

            ggml_build_forward_expand(gf, c);

            double tsum = 0.0;

            // heat-up
            ggml_graph_compute_helper(work, gf, n_threads, nullptr, nullptr);

            for (int i = 0; i < n_max; ++i) {
                const int64_t t0 = ggml_time_us();

                ggml_graph_compute_helper(work, gf, n_threads, nullptr, nullptr);

                const int64_t t1 = ggml_time_us();

                tsum += (t1 - t0)*1e-6;
                n++;

                if (tsum > 1.0 && n >= 3) {
                    break;
                }
            }

            ggml_free(ctx0);

            s = ((2.0*N*N*N*n)/tsum)*1e-9;
        }

        // Q4_0 | Q4_1
        snprintf(strbuf, sizeof(strbuf), "%4zu x %4zu: Q4_0 %7.1f GFLOPS (%3d runs) | Q4_1 %7.1f GFLOPS (%3d runs)\n",
                N, N, s_q4_0, n_q4_0, s_q4_1, n_q4_1);
        s += strbuf;

        // Q5_0 | Q5_1 | Q8_0
        snprintf(strbuf, sizeof(strbuf), "%4zu x %4zu: Q5_0 %7.1f GFLOPS (%3d runs) | Q5_1 %7.1f GFLOPS (%3d runs) | Q8_0 %7.1f GFLOPS (%3d runs)\n",
                N, N, s_q5_0, n_q5_0, s_q5_1, n_q5_1, s_q8_0, n_q8_0);
        s += strbuf;

        // F16 | F32
        snprintf(strbuf, sizeof(strbuf), "%4zu x %4zu: F16  %7.1f GFLOPS (%3d runs) | F32  %7.1f GFLOPS (%3d runs)\n",
                N, N, s_fp16, n_fp16, s_fp32, n_fp32);
        s += strbuf;
    }

    return s.c_str();
}

```

这段代码定义了一个名为“experimental stuff below”的标识，它表示下面的代码行可能是不稳定的，或者在将来的某个时间会被删除。然后，它定义了一个名为“token-level timestamps”的标识，它表示这段代码会添加一个名为“timestamps”的实体的定义。


```cpp
// =================================================================================================

// =================================================================================================

//
// Experimental stuff below
//
// Not sure if these should be part of the library at all, because the quality of the results is not
// guaranteed. Might get removed at some point unless a robust algorithm implementation is found
//

// =================================================================================================

//
// token-level timestamps
```

这段代码定义了三个函数，用于将不同的声音样本来评估其难度，并返回相应的难度值。

第一个函数 `timestamp_to_sample` 将一个时间戳（`int64_t` 类型）转换为采样率（`int` 类型）的采样数（`int64_t` 类型）。其实现方式是：将 `n_samples` 减 1（因为不能有负采样率），然后将 `t` 乘以一个系数 `WHISPER_SAMPLE_RATE`（未定义），最后的结果是 `0` 到 `n_samples-1` 之间的最大整数值。

第二个函数 `sample_to_timestamp` 将一个采样率（`int` 类型）转换为时间戳（`int64_t` 类型）的采样数（`int64_t` 类型）。其实现方式是：将 `i_sample`（未定义）除以 `WHISPER_SAMPLE_RATE`（未定义），然后的结果是整数部分，再将其乘以 100 并获取 `int64_t` 类型。

第三个函数 `voice_length` 定义了一个成本函数，用于评估一个字符串的难度。该函数接受一个字符串参数 `text`（未定义）。函数首先将字符串中的所有空格去掉，然后对于每个大写字母、小写字母、下划线、问号、感叹号，将分别计算其难度并累加。最后将所有难度值取最大值并获取，结果是 `float` 类型。


```cpp
//

static int timestamp_to_sample(int64_t t, int n_samples) {
    return std::max(0, std::min((int) n_samples - 1, (int) ((t*WHISPER_SAMPLE_RATE)/100)));
}

static int64_t sample_to_timestamp(int i_sample) {
    return (100ll*i_sample)/WHISPER_SAMPLE_RATE;
}

// a cost-function / heuristic that is high for text that takes longer to pronounce
// obviously, can be improved
static float voice_length(const std::string & text) {
    float res = 0.0f;

    for (char c : text) {
        if (c == ' ') {
            res += 0.01f;
        } else if (c == ',') {
            res += 2.00f;
        } else if (c == '.') {
            res += 3.00f;
        } else if (c == '!') {
            res += 3.00f;
        } else if (c == '?') {
            res += 3.00f;
        } else if (c >= '0' && c <= '9') {
            res += 3.00f;
        } else {
            res += 1.00f;
        }
    }

    return res;
}

```

该代码定义了一个名为`get_signal_energy`的静态函数，接受一个浮点数指针`signal`、一个整数`n_samples`以及一个整数`n_samples_per_half_window`作为参数。

函数内部首先定义了一个`std::vector`类型的变量`result`来存储平均信号能量的结果。接着，函数内部循环`n_samples`次，每次循环从`-hw`到`hw`的整数范围内遍历信号的每个 half-window。在内层循环中，函数使用`fabs`函数获取信号每个 half-window 的绝对值，并将其累加到`sum`变量中。最后，将`sum`除以半窗长度（`2*hw+1`）的平方并取倒数，得到每个 half-window 的平均信号能量，将其存储在`result`中的对应位置。

函数返回一个`std::vector`类型的变量`result`，其中包含信号中所有 half-window 的平均信号能量。


```cpp
// average the fabs of the signal
static std::vector<float> get_signal_energy(const float * signal, int n_samples, int n_samples_per_half_window) {
    const int hw = n_samples_per_half_window;

    std::vector<float> result(n_samples);

    for (int i = 0; i < n_samples; i++) {
        float sum = 0;
        for (int j = -hw; j <= hw; j++) {
            if (i + j >= 0 && i + j < n_samples) {
                sum += fabs(signal[i + j]);
            }
        }
        result[i] = sum/(2*hw + 1);
    }

    return result;
}

```

This is a C++ program that appears to be a simple implementation of a tokenizer for a natural language text. The program takes in a list of tokens, a threshold value `thold`, and a maximum expansion factor `t_expand`. It then expands the tokens by sampling from the closest neighbors and updating their distance and timestamp based on the context provided by the `whisper_token_to_str` function.

The program also has a `debug` function that prints out some information about the current token, such as its distance, timestamp, and ID, but only for debugging purposes and provides some comments about the expansion factor.

It is important to note that this program may not work correctly in all cases, and should be thoroughly tested and validated before being used in production.


```cpp
static void whisper_exp_compute_token_level_timestamps(
        struct whisper_context & ctx,
          struct whisper_state & state,
                           int   i_segment,
                         float   thold_pt,
                         float   thold_ptsum) {
    auto & segment = state.result_all[i_segment];
    auto & tokens  = segment.tokens;

    const int n_samples = state.energy.size();

    if (n_samples == 0) {
        log("%s: no signal data available\n", __func__);
        return;
    }

    const int64_t t0 = segment.t0;
    const int64_t t1 = segment.t1;

    const int n = tokens.size();

    if (n == 0) {
        return;
    }

    if (n == 1) {
        tokens[0].t0 = t0;
        tokens[0].t1 = t1;

        return;
    }

    auto & t_beg    = state.t_beg;
    auto & t_last   = state.t_last;
    auto & tid_last = state.tid_last;

    for (int j = 0; j < n; ++j) {
        auto & token = tokens[j];

        if (j == 0) {
            if (token.id == whisper_token_beg(&ctx)) {
                tokens[j    ].t0 = t0;
                tokens[j    ].t1 = t0;
                tokens[j + 1].t0 = t0;

                t_beg    = t0;
                t_last   = t0;
                tid_last = whisper_token_beg(&ctx);
            } else {
                tokens[j    ].t0 = t_last;
            }
        }

        const int64_t tt = t_beg + 2*(token.tid - whisper_token_beg(&ctx));

        tokens[j].id    = token.id;
        tokens[j].tid   = token.tid;
        tokens[j].p     = token.p;
        tokens[j].pt    = token.pt;
        tokens[j].ptsum = token.ptsum;

        tokens[j].vlen = voice_length(whisper_token_to_str(&ctx, token.id));

        if (token.pt > thold_pt && token.ptsum > thold_ptsum && token.tid > tid_last && tt <= t1) {
            if (j > 0) {
                tokens[j - 1].t1 = tt;
            }
            tokens[j].t0 = tt;
            tid_last = token.tid;
        }
    }

    tokens[n - 2].t1 = t1;
    tokens[n - 1].t0 = t1;
    tokens[n - 1].t1 = t1;

    t_last = t1;

    // find intervals of tokens with unknown timestamps
    // fill the timestamps by proportionally splitting the interval based on the token voice lengths
    {
        int p0 = 0;
        int p1 = 0;

        while (true) {
            while (p1 < n && tokens[p1].t1 < 0) {
                p1++;
            }

            if (p1 >= n) {
                p1--;
            }

            //printf("p0=%d p1=%d t0=%lld t1=%lld\n", p0, p1, tokens[p0].t0, tokens[p1].t1);

            if (p1 > p0) {
                double psum = 0.0;
                for (int j = p0; j <= p1; j++) {
                    psum += tokens[j].vlen;
                }

                //printf("analyzing %d - %d, psum = %f\n", p0, p1, psum);

                const double dt = tokens[p1].t1 - tokens[p0].t0;

                // split the time proportionally to the voice length
                for (int j = p0 + 1; j <= p1; j++) {
                    const double ct = tokens[j - 1].t0 + dt*tokens[j - 1].vlen/psum;

                    tokens[j - 1].t1 = ct;
                    tokens[j    ].t0 = ct;
                }
            }

            p1++;
            p0 = p1;
            if (p1 >= n) {
                break;
            }
        }
    }

    // fix up (just in case)
    for (int j = 0; j < n - 1; j++) {
        if (tokens[j].t1 < 0) {
            tokens[j + 1].t0 = tokens[j].t1;
        }

        if (j > 0) {
            if (tokens[j - 1].t1 > tokens[j].t0) {
                tokens[j].t0 = tokens[j - 1].t1;
                tokens[j].t1 = std::max(tokens[j].t0, tokens[j].t1);
            }
        }
    }

    // VAD
    // expand or contract tokens based on voice activity
    {
        const int hw = WHISPER_SAMPLE_RATE/8;

        for (int j = 0; j < n; j++) {
            if (tokens[j].id >= whisper_token_eot(&ctx)) {
                continue;
            }

            int s0 = timestamp_to_sample(tokens[j].t0, n_samples);
            int s1 = timestamp_to_sample(tokens[j].t1, n_samples);

            const int ss0 = std::max(s0 - hw, 0);
            const int ss1 = std::min(s1 + hw, n_samples);

            const int ns = ss1 - ss0;

            float sum = 0.0f;

            for (int k = ss0; k < ss1; k++) {
                sum += state.energy[k];
            }

            const float thold = 0.5*sum/ns;

            {
                int k = s0;
                if (state.energy[k] > thold && j > 0) {
                    while (k > 0 && state.energy[k] > thold) {
                        k--;
                    }
                    tokens[j].t0 = sample_to_timestamp(k);
                    if (tokens[j].t0 < tokens[j - 1].t1) {
                        tokens[j].t0 = tokens[j - 1].t1;
                    } else {
                        s0 = k;
                    }
                } else {
                    while (state.energy[k] < thold && k < s1) {
                        k++;
                    }
                    s0 = k;
                    tokens[j].t0 = sample_to_timestamp(k);
                }
            }

            {
                int k = s1;
                if (state.energy[k] > thold) {
                    while (k < n_samples - 1 && state.energy[k] > thold) {
                        k++;
                    }
                    tokens[j].t1 = sample_to_timestamp(k);
                    if (j < ns - 1 && tokens[j].t1 > tokens[j + 1].t0) {
                        tokens[j].t1 = tokens[j + 1].t0;
                    } else {
                        s1 = k;
                    }
                } else {
                    while (state.energy[k] < thold && k > s0) {
                        k--;
                    }
                    s1 = k;
                    tokens[j].t1 = sample_to_timestamp(k);
                }
            }
        }
    }

    // fixed token expand (optional)
    //{
    //    const int t_expand = 0;

    //    for (int j = 0; j < n; j++) {
    //        if (j > 0) {
    //            tokens[j].t0 = std::max(0, (int) (tokens[j].t0 - t_expand));
    //        }
    //        if (j < n - 1) {
    //            tokens[j].t1 = tokens[j].t1 + t_expand;
    //        }
    //    }
    //}

    // debug info
    //for (int j = 0; j < n; ++j) {
    //    const auto & token = tokens[j];
    //    const auto tt = token.pt > thold_pt && token.ptsum > 0.01 ? whisper_token_to_str(&ctx, token.tid) : "[?]";
    //    printf("%s: %10s %6.3f %6.3f %6.3f %6.3f %5d %5d '%s'\n", __func__,
    //            tt, token.p, token.pt, token.ptsum, token.vlen, (int) token.t0, (int) token.t1, whisper_token_to_str(&ctx, token.id));

    //    if (tokens[j].id >= whisper_token_eot(&ctx)) {
    //        continue;
    //    }
    //}
}

```

这段代码定义了一个名为`whisper_set_log_callback`的函数，它接受一个名为`callback`的参数。

这个函数的作用是设置一个名为`whisper_log`的实参，它是传递给`whisper_log_callback`函数的实参。通过设置这个实参，我们可以让`whisper_log_callback`函数访问到`whisper_log`，实现对日志的查看和输出。


```cpp
void whisper_set_log_callback(whisper_log_callback callback) {
    whisper_log = callback;
}

```