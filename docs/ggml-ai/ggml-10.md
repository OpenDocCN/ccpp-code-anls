# GGML源码解析 10

# `examples/gpt-neox/quantize.cpp`

这段代码是一个通用的C++代码，它涉及到机器学习中的数据预处理和特征工程方面。作用是帮助开发者处理和分析数据集，以及在机器学习中使用数据集。

具体来说，这段代码包含以下几个部分：

1. 引入必要的头文件，包括ggml和common的头的文件。

2. 引入了一个common.h文件，但是由于它没有定义任何函数，因此它的作用可能是定义了一些常量或变量，以便在main函数中使用。

3. 引入了一个common-ggml.h文件，同样由于它没有定义任何函数，因此它的作用也可能是定义了一些常量或变量，以便在main函数中使用。

4. 引入了一个数据预处理函数的定义，它的名字是"prepare_data"。

5. 引入了一个数据读取函数的定义，它的名字是"read_data"。

6. 在数据预处理函数中，使用<cassert>和<cmath>、<cstdio>和<cstring>、<fstream>和<map>、<string>和<vector>、<regex>对数据集进行处理。

7. 在数据读取函数中，使用<iostream>和<vector>对数据集进行读取。

8. 可能还包含其他代码，以便对数据集进行分析和应用机器学习算法。

总结起来，这段代码定义了一些通用的函数和变量，以便在机器学习开发中处理数据集。不过，由于它没有提供任何具体的函数或方法，因此具体的用途取决于开发者的需求。


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

This is a C++ function that reads a .pth file containing the ParaTechNet model parameters, including the weight and word amounts of each layer. The ParaTechNet is assumed to be a pre-trained neural network model with multiple layers, and the function expects the input file to be a ParaTechNet definition file (e.g., train.pth or test.pth) that describes the model architecture and parameters.

The function first reads the header information from the input file, such as the number of layers, the number ofctx blocks, and the embedding dimension. Then, it reads the embedding data from the file and stores it in a buffer. Next, it reads the layer information, such as the number of embeddings per layer and the rotation angles, and stores it in another buffer. Finally, it reads the type of the data (float or float32) and the float data type from the file and stores it in yet another buffer.

The function then loads the pre-trained word embeddings from the file. It assumes that the file contains a list of pre-defined words and their corresponding indexes.

The function uses the ParaTechNet function provided by the ParaTechNet library to read the model parameters from the .pth file, which allows for the conversion of the ParaTechNet definition file to a ParaTechNet model and back. The ParaTechNet function expects the input file to be a ParaTechNet definition file and uses the information in the file to create an object of the ParaTechNet class and then returns it.

The function uses the `std::vector` container to store the pre-defined words for each layer. It assumes that the input file contains multiple layers and uses the `std::regex` function to search for the defined regex patterns in the input text. If the .pth file is not successful in loading the pre-defined words, the function will print an error message and return `false`.


```cpp
// default hparams (StableLM 3B)
struct gpt_neox_hparams {
    int32_t n_vocab = 50257;
    int32_t n_ctx   = 4096;
    int32_t n_embd  = 4096;
    int32_t n_head  = 32;
    int32_t n_layer = 16;
    int32_t n_rot   = 32; // 0.25 * (n_embd / n_head)
    int32_t par_res = 1; // 1 = true, 0 = false
    int32_t ftype   = 1;
};

// quantize a model
bool gpt_neox_model_quantize(const std::string & fname_inp, const std::string & fname_out, ggml_ftype ftype) {
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

    gpt_neox_hparams hparams;

    // load hparams
    {
        finp.read((char *) &hparams.n_vocab, sizeof(hparams.n_vocab));
        finp.read((char *) &hparams.n_ctx,   sizeof(hparams.n_ctx));
        finp.read((char *) &hparams.n_embd,  sizeof(hparams.n_embd));
        finp.read((char *) &hparams.n_head,  sizeof(hparams.n_head));
        finp.read((char *) &hparams.n_layer, sizeof(hparams.n_layer));
        finp.read((char *) &hparams.n_rot,   sizeof(hparams.n_rot));
        finp.read((char *) &hparams.par_res, sizeof(hparams.par_res));
        finp.read((char *) &hparams.ftype,   sizeof(hparams.ftype));

        const int32_t qntvr_src =    hparams.ftype / GGML_QNT_VERSION_FACTOR;
        const int32_t ftype_dst = GGML_QNT_VERSION * GGML_QNT_VERSION_FACTOR + ftype;

        printf("%s: n_vocab     = %d\n", __func__, hparams.n_vocab);
        printf("%s: n_ctx       = %d\n", __func__, hparams.n_ctx);
        printf("%s: n_embd      = %d\n", __func__, hparams.n_embd);
        printf("%s: n_head      = %d\n", __func__, hparams.n_head);
        printf("%s: n_layer     = %d\n", __func__, hparams.n_layer);
        printf("%s: par_res     = %d\n", __func__, hparams.par_res);
        printf("%s: ftype (src) = %d\n", __func__, hparams.ftype);
        printf("%s: qntvr (src) = %d\n", __func__, qntvr_src);
        printf("%s: ftype (dst) = %d\n", __func__, ftype_dst);
        printf("%s: qntvr (dst) = %d\n", __func__, GGML_QNT_VERSION);

        fout.write((char *) &hparams.n_vocab, sizeof(hparams.n_vocab));
        fout.write((char *) &hparams.n_ctx,   sizeof(hparams.n_ctx));
        fout.write((char *) &hparams.n_embd,  sizeof(hparams.n_embd));
        fout.write((char *) &hparams.n_head,  sizeof(hparams.n_head));
        fout.write((char *) &hparams.n_layer, sizeof(hparams.n_layer));
        fout.write((char *) &hparams.n_rot,   sizeof(hparams.n_rot));
        fout.write((char *) &hparams.par_res, sizeof(hparams.par_res));
        fout.write((char *) &ftype_dst,       sizeof(ftype_dst));
    }

    // load vocab
    {
        const int32_t n_vocab = hparams.n_vocab;

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
        ".*weight",
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

This is a C++ program that uses the Deep Model Optimization (DMO) tool to perform a model optimization internship. The program takes two command-line arguments:

* The first argument is the name of the input model file (in the format "YYYY-MM-DD HH:MM:SS.MMM").
* The second argument is the name of the output model file (in the format "YYYY-MM-DD HH:MM:SS.MMM"). The input and output model files must already exist.

The program performs the following tasks:

* Reads the input model file and unququeuesues the f16 data in the queue.
* Reads the input model file and performs model optimization by quantizing the f16 data in the queue.
* Report the timing information for the quantized model optimization.

To use this program, you can save it to a file named "model-f32.bin" and "model-quant.bin" in the same directory and run the following command:
```cppphp
./model_opt.py input_model.bin output_model.bin
```
This will perform the optimization and report the timing information for the optimization.


```cpp
// usage:
//  ./gpt-neox-quantize models/stalellm2-117M/ggml-model.bin models/stablelm2-117M/ggml-model-quant.bin type
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

        if (!gpt_neox_model_quantize(fname_inp, fname_out, ggml_ftype(ftype))) {
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

# GPT-NeoX

Transformer architecture: GPT-NeoX

Ref: https://github.com/stability-AI/stableLM/#stablelm-alpha

## Usage

```cppbash
# get the repo and build it
git clone https://github.com/ggerganov/ggml
cd ggml
mkdir build && cd build
cmake ..
make -j

# get the StableLM 3B Alpha model
git clone https://huggingface.co/stabilityai/gpt_neox-base-alpha-3b

# install Python dependencies
python3 -m pip install -r ../requirements.txt

# convert model to FP16
python3 ../examples/gpt-neox/convert-h5-to-ggml.py ./stablelm-base-alpha-3b/ 1

# run inference using FP16 precision
make -j && ./bin/gpt-neox -m ./stablelm-base-alpha-3b/ggml-model-f16.bin -p "I believe the meaning of life is" -t 8 -n 64

main: seed = 1681940611
gpt_neox_model_load: loading model from 'models/stablelm-base-alpha-3b/ggml-model-f16.bin' - please wait ...
gpt_neox_model_load: n_vocab = 50688
gpt_neox_model_load: n_ctx   = 4096
gpt_neox_model_load: n_embd  = 4096
gpt_neox_model_load: n_head  = 32
gpt_neox_model_load: n_layer = 16
gpt_neox_model_load: n_rot   = 32
gpt_neox_model_load: ftype   = 1
gpt_neox_model_load: ggml ctx size = 10011.10 MB
gpt_neox_model_load: memory_size =  2048.00 MB, n_mem = 65536
gpt_neox_model_load: ................................ done
gpt_neox_model_load: model size =  6939.28 MB / num tensors = 260
main: number of tokens in prompt = 7
main: token[0] =     42, I
main: token[1] =   2868,  believe
main: token[2] =    253,  the
main: token[3] =   4495,  meaning
main: token[4] =    273,  of
main: token[5] =   1495,  life
main: token[6] =    310,  is

I believe the meaning of life is to grow, to find a way, to love, to find an appreciation for life, and to live it with all of its beauty.

For I am the child of God. I am the offspring of God's love. I am the offspring of the light of the world. I am the offspring of the

main: mem per token = 12186760 bytes
main:     load time =  2118.55 ms
main:   sample time =     9.59 ms
main:  predict time =  4474.07 ms / 63.92 ms per token
main:    total time =  6911.26 ms
```

## 5-bit integer quantization mode

```cppbash
# quantize the model to 5-bits using Q5_0 quantization
./bin/gpt-neox-quantize ./stablelm-base-alpha-3b/ggml-model-f16.bin ./stablelm-base-alpha-3b/ggml-model-q5_0.bin q5_0

# run the quantized model
./bin/gpt-neox -m ./stablelm-base-alpha-3b/ggml-model-q5_0.bin -p "I believe the meaning of life is" -t 8 -n 64

main: seed = 1682021489
gpt_neox_model_load: loading model from 'models/stablelm-base-alpha-3b/ggml-model-q5_0.bin' - please wait ...
gpt_neox_model_load: n_vocab = 50688
gpt_neox_model_load: n_ctx   = 4096
gpt_neox_model_load: n_embd  = 4096
gpt_neox_model_load: n_head  = 32
gpt_neox_model_load: n_layer = 16
gpt_neox_model_load: n_rot   = 32
gpt_neox_model_load: ftype   = 6
gpt_neox_model_load: ggml ctx size = 5676.10 MB
gpt_neox_model_load: memory_size =  1024.00 MB, n_mem = 65536
gpt_neox_model_load: ........................ done
gpt_neox_model_load: model size =  2604.28 MB / num tensors = 196
main: number of tokens in prompt = 7
main: token[0] =     42, I
main: token[1] =   2868,  believe
main: token[2] =    253,  the
main: token[3] =   4495,  meaning
main: token[4] =    273,  of
main: token[5] =   1495,  life
main: token[6] =    310,  is

I believe the meaning of life is to love and be loved. The last three verses were enough to tie us all together. If you love someone you love them all. There are some things in this world that are just not equal in Heaven. - Be here in this moment.

This world is not what is outside of us. It is what

main: mem per token = 12958024 bytes
main:     load time =   850.51 ms
main:   sample time =     9.95 ms
main:  predict time =  3103.81 ms / 44.34 ms per token
main:    total time =  4177.68 ms

```

## Notes

- No guarantees for correctness
- The tokenizer is currently hacked - probably works only for English
- Non-parallel residual is not supported
- Contributions and improvements are welcome


# `examples/mnist/convert-h5-to-ggml.py`

这段代码的作用是将其MNIS h5 transformer模型转换为ggml格式。它主要实现了以下几个步骤：

1. 加载（state_dict）保存的模型，这个模型是使用PyTorch实现的。
2. 遍历模型的所有变量，并将它们输出到一个 binary 文件中。
3. 给每个变量写入一个二进制数据，包括变量的数量维度、变量名称长度、变量的维度和变量名称。
4. 在每个二进制数据中，写入模型的参数。
5. 在ggml文件的开始位置，写入了模型的参数。

这段代码主要实现了将MNIS h5 transformer模型转换为ggml格式的功能。


```cpp
# Convert MNIS h5 transformer model to ggml format
#
# Load the (state_dict) saved model using PyTorch
# Iterate over all variables and write them to a binary file.
#
# For each variable, write the following:
#   - Number of dimensions (int)
#   - Name length (int)
#   - Dimensions (int[n_dims])
#   - Name (char[name_length])
#   - Data (float[n_dims])
#
# At the start of the ggml file we write the model parameters

import sys
```

这段代码的作用是从HDF5格式数据中读取一个模型，并将其转换为GraphML格式数据，然后将其保存为JSON格式。转换后的数据可以被输入到神经网络中进行训练。

具体步骤如下：

1. 导入需要用到的库：包括struct、json、numpy、re、torch、torchvision.datasets、torchvision.transforms和Variable。

2. 从HDF5文件中读取数据，并将其转换为Python中的结构体。

3. 将结构体转换为JSON格式。

4. 从HDF5文件中读取数据中的模型，并将其保存为变量。

5. 将模型转换为PyTorch中的神经网络，并将其保存为Variable。

6. 准备数据和模型，以便进行训练。


```cpp
import struct
import json
import numpy as np
import re


import torch
import torch.nn as nn
import torchvision.datasets as dsets
import torchvision.transforms as transforms
from torch.autograd import Variable

if len(sys.argv) != 2:
    print("Usage: convert-h5-to-ggml.py model\n")
    sys.exit(1)

```

这段代码的主要作用是读取一个名为"state_dict_file"的文件，该文件存储了训练数据中的参数，包括模型参数和损失函数等。该文件使用的是PyTorch中的`torch.load()`函数来加载这些参数，并将其存回一个PyTorch张量。然后，代码将这些张量打印出来，并接着读取文件中的数据，每个数据都是张量中的一个数值序列。通过循环读取数据，代码将每个数据序列的形状打印出来，然后将其写入到名为"fname_out"的文件中。写入的文件格式是先输出一个固定长度的二进制数据(即0x67676d6c)，然后是数据类型，接着是数据形状，然后是数据序列中的元素个数。在循环中，代码使用了`struct.pack()`函数将数据序列的元素类型和形状转换为二进制数据，并将其写入文件中。最后，代码将数据序列中的元素打印出来，以便用户查看数据。


```cpp
state_dict_file = sys.argv[1]
fname_out = "models/mnist/ggml-model-f32.bin"

state_dict = torch.load(state_dict_file, map_location=torch.device('cpu'))
#print (model)

list_vars = state_dict
print (list_vars)

fout = open(fname_out, "wb")

fout.write(struct.pack("i", 0x67676d6c)) # magic: ggml in hex


for name in list_vars.keys():
    data = list_vars[name].squeeze().numpy()
    print("Processing variable: " + name + " with shape: ", data.shape) 
    n_dims = len(data.shape);
   
    fout.write(struct.pack("i", n_dims))
    
    data = data.astype(np.float32)
    for i in range(n_dims):
        fout.write(struct.pack("i", data.shape[n_dims - 1 - i]))

    # data
    data.tofile(fout)

```

这段代码的主要作用是关闭文件fout并输出文件名为fname_out的文件内容。

1. `fout.close()` 是一个文件操作类的函数，作用是关闭名为fout的文件。通常情况下，当有一个文件需要操作时，我们需要使用一个文件操作类来打开或关闭文件，而这个函数就是用来关闭文件的。

2. `print("Done. Output file: " + fname_out)` 是一个用于输出文本数据的函数，作用是在关闭文件之后输出一个字符串，其中字符串由两部分组成：一个是“Done.”(表示文件操作成功)，另一个是`fname_out`(源文件名)，即将fout文件的内容输出到字符串中。

3. `print("")` 是一个空输出函数，作用是在前两个输出函数之后输出一个空字符串，以便在之后的输出中有一些空间。


```cpp
fout.close()

print("Done. Output file: " + fname_out)
print("")

```

# `examples/mnist/main-cnn.cpp`

这段代码是一个C++程序，它包括两个头文件：ggml.h和common.h。ggml.h可能是一个第三方库或框架，common.h是必不可少的。

该程序的主要作用是计算一个二维矩阵的逆矩阵。具体来说，程序会执行以下操作：

1. 读取一个整数矩阵ggml.h中的数据，以及一个名为ggml_map的二维数组ggml_map，ggml_map可能用于存储数据和一些元数据。

2. 读取一个文件（可能是一个稀疏矩阵或边界矩阵），名为ggml_map.txt，ggml_map.txt可能用于存储数据和一些元数据。

3. 计算ggml.h中的数据矩阵与ggml_map中的数据矩阵的逆矩阵。

4. 将计算得到的逆矩阵存储在ggml.h中，以便后续使用。

该程序的输出是一个二维矩阵，每个元素是一个用ggml.h中数据表示的整数。


```cpp
#include "ggml/ggml.h"

#include "common.h"

#include <cmath>
#include <cstdio>
#include <cstring>
#include <ctime>
#include <fstream>
#include <string>
#include <vector>
#include <algorithm>

#if defined(_MSC_VER)
#pragma warning(disable: 4244 4267) // possible loss of data
```

这段代码是一个 C++ 程序，定义了一个名为 "mnist_model" 的结构体，其中包括一些指向不同张量的指针以及一些在内存中分配的内存空间。

具体来说，这个结构体存储了一个在第一层卷积层中使用的两个卷积层和两个密集层中使用的权重和偏移量，以及一个用于计算输出数据的总和的内存变量。

此外，这个结构体还包含一个指向其输入数据的输入张量，以及一个指向其输出数据的输出张量。

函数 `mnist_model_load()` 用于加载一个 MNIST 数据集，并将其转换为模型结构体，该结构体包含在训练时使用的卷积层和数据预处理层的参数。该函数首先尝试从传入的文件中加载参数化的数据，如果失败，则输出错误信息并返回 false。如果成功加载数据，则将加载的卷积层和数据预处理层的参数存储在模型结构体中，并返回 true。


```cpp
#endif

struct mnist_model {
    struct ggml_tensor * conv2d_1_kernel;
    struct ggml_tensor * conv2d_1_bias;
    struct ggml_tensor * conv2d_2_kernel;
    struct ggml_tensor * conv2d_2_bias;
    struct ggml_tensor * dense_weight;
    struct ggml_tensor * dense_bias;
    struct ggml_context * ctx;
};

bool mnist_model_load(const std::string & fname, mnist_model & model) {
    struct gguf_init_params params = {
        /*.no_alloc   =*/ false,
        /*.ctx        =*/ &model.ctx,
    };
    gguf_context * ctx = gguf_init_from_file(fname.c_str(), params);
    if (!ctx) {
        fprintf(stderr, "%s: gguf_init_from_file() failed\n", __func__);
        return false;
    }
    model.conv2d_1_kernel = ggml_get_tensor(model.ctx, "kernel1");
    model.conv2d_1_bias = ggml_get_tensor(model.ctx, "bias1");
    model.conv2d_2_kernel = ggml_get_tensor(model.ctx, "kernel2");
    model.conv2d_2_bias = ggml_get_tensor(model.ctx, "bias2");
    model.dense_weight = ggml_get_tensor(model.ctx, "dense_w");
    model.dense_bias = ggml_get_tensor(model.ctx, "dense_b");
    return true;
}

```

This is a Python implementation of a simple CNN model for the MNIST dataset using GraphGDB. The model uses a Conv2D layer with a single pooling operation, a MaxPooling2D layer, and a Dense layer with fully-connected output. The `ggml_pool_2d`, `ggml_cont`, and `ggml_reshape_2d` functions are defined in the GraphGDB C library and are used to perform the pooling and reshaping operations. The `ggml_add` and `ggml_mul_mat` functions are also defined in the GraphGDB C library and are used to perform the dense layer operations. The `ggml_permute`, `ggml_soft_max`, and `ggml_graph_print` functions are also defined in the GraphGDB C library and are used to perform the permutation and softmax operations.

The `mnist_cnn` function takes as input a GraphGDB context and an optional file name for the computed graph. It first creates the input tensor, performs theMaxPooling2D operation, applies the Dense layer, and returns the output. The `fname_cgraph` option is used to specify the output file name for the computed graph. If this option is specified, the `ggml_graph_print` function is used to export the graph to a file.

The `mnist_cpu` example can be used to evaluate the performance of the model on a CPU rather than a GPU. The output of the `mnist_cnn` function is the predicted class for the given input image, which can be compared to the ground truth labels to evaluate the accuracy of the model.


```cpp
int mnist_eval(
        const mnist_model & model,
        const int n_threads,
        std::vector<float> digit,
        const char * fname_cgraph
        )
{
    static size_t buf_size = 100000 * sizeof(float) * 4;
    static void * buf = malloc(buf_size);

    struct ggml_init_params params = {
        /*.mem_size   =*/ buf_size,
        /*.mem_buffer =*/ buf,
        /*.no_alloc   =*/ false,
    };

    struct ggml_context * ctx0 = ggml_init(params);
    struct ggml_cgraph * gf = ggml_new_graph(ctx0);

    struct ggml_tensor * input = ggml_new_tensor_4d(ctx0, GGML_TYPE_F32, 28, 28, 1, 1);
    memcpy(input->data, digit.data(), ggml_nbytes(input));
    ggml_set_name(input, "input");
    ggml_tensor * cur = ggml_conv_2d(ctx0, model.conv2d_1_kernel, input, 1, 1, 0, 0, 1, 1);
    cur = ggml_add(ctx0, cur, model.conv2d_1_bias);
    cur = ggml_relu(ctx0, cur);
    // Output shape after Conv2D: (26 26 32 1)
    cur = ggml_pool_2d(ctx0, cur, GGML_OP_POOL_MAX, 2, 2, 2, 2, 0, 0);
    // Output shape after MaxPooling2D: (13 13 32 1)
    cur = ggml_conv_2d(ctx0, model.conv2d_2_kernel, cur, 1, 1, 0, 0, 1, 1);
    cur = ggml_add(ctx0, cur, model.conv2d_2_bias);
    cur = ggml_relu(ctx0, cur);
    // Output shape after Conv2D: (11 11 64 1)
    cur = ggml_pool_2d(ctx0, cur, GGML_OP_POOL_MAX, 2, 2, 2, 2, 0, 0);
    // Output shape after MaxPooling2D: (5 5 64 1)
    cur = ggml_cont(ctx0, ggml_permute(ctx0, cur, 1, 2, 0, 3));
    // Output shape after permute: (64 5 5 1)
    cur = ggml_reshape_2d(ctx0, cur, 1600, 1);
    // Final Dense layer
    cur = ggml_add(ctx0, ggml_mul_mat(ctx0, model.dense_weight, cur), model.dense_bias);
    ggml_tensor * probs = ggml_soft_max(ctx0, cur);
    ggml_set_name(probs, "probs");

    ggml_build_forward_expand(gf, probs);
    ggml_graph_compute_with_ctx(ctx0, gf, n_threads);

    //ggml_graph_print(&gf);
    ggml_graph_dump_dot(gf, NULL, "mnist-cnn.dot");

    if (fname_cgraph) {
        // export the compute graph for later use
        // see the "mnist-cpu" example
        ggml_graph_export(gf, fname_cgraph);

        fprintf(stderr, "%s: exported compute graph to '%s'\n", __func__, fname_cgraph);
    }

    const float * probs_data = ggml_get_data_f32(probs);
    const int prediction = std::max_element(probs_data, probs_data + 10) - probs_data;
    ggml_free(ctx0);
    return prediction;
}

```

This is a C++ program that uses the GGML library to evaluate the performance of a machine learning model on a randomly generated digit. The program takes a filename of a pre-trained model and a digit as input arguments, and outputs the predicted digit, as well as the time it takes to load the model from the file.

Here's a breakdown of how the program works:

1. The program first checks if the input file is a valid model file by checking the file extension. If it's not a valid file, it prints an error message and returns 1.
2. The program reads the pre-trained model file and loads it into memory. This is done using the ggml\_load\_model() function. The function takes three arguments: the file name, the model format, and the location of the model.
3. The program reads a random 28-byte integer from the test set and uses it to render the digit as an ASCII character. This is done using the ggml\_evaluate\_prediction() function. The function takes two arguments: the model, and the sample to evaluate.
4. The program outputs the predicted digit and the time it takes to load the model from the file.
5. The program reads a random digit from the test set and uses it to evaluate the performance of the model. This is done using the mnist\_evaluate() function. The function takes three arguments: the model, the sample to evaluate, and the digit. The digit is converted to a floating-point number and passed to the function.
6. The program outputs the predicted digit, as well as the time it takes to load the model from the file.

It's worth noting that the program assumes that the input file is a valid pre-trained model file and that it has been properly read and loaded into memory. It also assumes that the mnist\_evaluate() function is a valid function for evaluating the performance of a machine learning model on a randomly generated digit.


```cpp
int main(int argc, char ** argv) {
    srand(time(NULL));
    ggml_time_init();

    if (argc != 3) {
        fprintf(stderr, "Usage: %s models/mnist/mnist-cnn.gguf models/mnist/t10k-images.idx3-ubyte\n", argv[0]);
        exit(0);
    }

    uint8_t buf[784];
    mnist_model model;
    std::vector<float> digit;

    // load the model
    {
        const int64_t t_start_us = ggml_time_us();

        if (!mnist_model_load(argv[1], model)) {
            fprintf(stderr, "%s: failed to load model from '%s'\n", __func__, argv[1]);
            return 1;
        }

        const int64_t t_load_us = ggml_time_us() - t_start_us;

        fprintf(stdout, "%s: loaded model in %8.2f ms\n", __func__, t_load_us / 1000.0f);
    }

    // read a random digit from the test set
    {
        std::ifstream fin(argv[2], std::ios::binary);
        if (!fin) {
            fprintf(stderr, "%s: failed to open '%s'\n", __func__, argv[2]);
            return 1;
        }

        // seek to a random digit: 16-byte header + 28*28 * (random 0 - 10000)
        fin.seekg(16 + 784 * (rand() % 10000));
        fin.read((char *) &buf, sizeof(buf));
    }

    // render the digit in ASCII
    {
        digit.resize(sizeof(buf));

        for (int row = 0; row < 28; row++) {
            for (int col = 0; col < 28; col++) {
                fprintf(stderr, "%c ", (float)buf[row*28 + col] > 230 ? '*' : '_');
                digit[row*28 + col] = ((float)buf[row*28 + col] / 255.0f);
            }

            fprintf(stderr, "\n");
        }

        fprintf(stderr, "\n");
    }

    const int prediction = mnist_eval(model, 1, digit, nullptr);
    fprintf(stdout, "%s: predicted digit is %d\n", __func__, prediction);
    ggml_free(model.ctx);
    return 0;
}

```

# `examples/mnist/main-cpu.cpp`

这段代码的作用是使用一个预定义的MNIST计算图在CPU上进行推理。它首先从命令行中运行 "mnist" 工具来生成一个计算图文件 "mnist.ggml"。接着，它运行 "mnist-cpu" 工具来使用生成的计算图文件。

具体来说，代码中包含以下两个主要函数：

1. "create_mnist_model_ffi"：这是一个用于创建MNIST模型的函数，这个函数使用了一个预定义的模型文件(名为 "mnist/models/mnist/ffi.dat")，并从命令行中运行 "mnist" 工具来生成计算图文件 "mnist.ggml"。

2. "run_mnist_inference"：这是一个用于在CPU上运行MNIST模型的函数，它使用生成的计算图文件 "mnist.ggml" 作为输入，并返回模型的输出结果。这个函数调用了 "create_mnist_model_ffi" 函数来创建MNIST模型。

虽然这段代码的实现比较简单，但它仍然涉及到MNIST数据集的预处理和模型文件的生成。


```cpp
// Use a pre-generated MNIST compute graph for inference on the CPU
//
// You can generate a compute graph using the "mnist" tool:
//
// $ ./bin/mnist ./models/mnist/ggml-model-f32.bin ../examples/mnist/models/mnist/t10k-images.idx3-ubyte
//
// This command creates the "mnist.ggml" file, which contains the generated compute graph.
// Now, you can re-use the compute graph with the "mnist-cpu" tool:
//
// $ ./bin/mnist-cpu ./models/mnist/mnist.ggml ../examples/mnist/models/mnist/t10k-images.idx3-ubyte
//

#include "ggml/ggml.h"

#include <algorithm>
```

这段代码是一个用于评估MNIST数据集 compute graph 的函数。它使用了C++标准库中的数学库、标准输入输出库、字符串库和时间库。

具体来说，它包括以下几个部分：

1. 引入了必要的C++数学库，如cmath、cstdio、cstring和ctime。
2. 引入了用于从文件中读取数据的开源工具，如fopen。
3. 引入了用于创建向量计算机的库，如vector。
4. 通过`#if defined(_MSC_VER)`这一条注释，告诉编译器在支持多线程的情况下运行这个代码。
5. 通过`#pragma warning(disable: 4244 4267)`这一条注释，屏蔽掉除4244和4267号类型的警告信息。

该函数的作用是读取MNIST数据集中的每一条边，并将每条边的权值存储在一个vector中，权值为-1表示无效边，权值为1表示有效边。


```cpp
#include <cmath>
#include <cstdio>
#include <cstring>
#include <ctime>
#include <fstream>
#include <vector>

#if defined(_MSC_VER)
#pragma warning(disable: 4244 4267) // possible loss of data
#endif

// evaluate the MNIST compute graph
//
//   - fname_cgraph: path to the compute graph
//   - n_threads:    number of threads to use
```

This is a function that uses the MNIST dataset from the TensorFlow library to evaluate the performance of a machine learning model. The function takes in the file name of the compute graph and the number of threads to use for computation. It also takes in a vector of floating-point numbers representing the digits in the MNIST dataset.

The function first loads the compute graph and then gets a reference to the graph's root node. It then creates a structure to hold the computation graph and a structure to hold the input data.

The function then sets the memory size and assigns the input data to the input structure. It then initializes the computation graph with the computation parameters and computes the forward iteration of the graph using the `ggml_graph_compute_with_ctx()` function.

Finally, it extracts the prediction from the output tensor and returns it.

Note that the function assumes that the input data has already been passed to the `memcpy()` function before passing it to the `ggml_graph_compute_with_ctx()` function.


```cpp
//   - digit:        784 pixel values
//
// returns 0 - 9 prediction
int mnist_eval(
        const char * fname_cgraph,
        const int n_threads,
        std::vector<float> digit) {
    // load the compute graph
    struct ggml_context * ctx_data = NULL;
    struct ggml_context * ctx_eval = NULL;

    struct ggml_cgraph * gfi = ggml_graph_import(fname_cgraph, &ctx_data, &ctx_eval);

    // param export/import test
    GGML_ASSERT(ggml_graph_get_tensor(gfi, "fc1_bias")->op_params[0] == int(0xdeadbeef));

    // allocate work context
    // needed during ggml_graph_compute() to allocate a work tensor
    static size_t buf_size = 128ull*1024*1024; // TODO
    static void * buf = malloc(buf_size);

    struct ggml_init_params params = {
        /*.mem_size   =*/ buf_size,
        /*.mem_buffer =*/ buf,
        /*.no_alloc   =*/ false,
    };

    struct ggml_context * ctx_work = ggml_init(params);

    struct ggml_tensor * input = ggml_graph_get_tensor(gfi, "input");
    memcpy(input->data, digit.data(), ggml_nbytes(input));

    ggml_graph_compute_with_ctx(ctx_work, gfi, n_threads);

    const float * probs_data = ggml_get_data_f32(ggml_graph_get_tensor(gfi, "probs"));

    const int prediction = std::max_element(probs_data, probs_data + 10) - probs_data;

    ggml_free(ctx_work);
    ggml_free(ctx_data);
    ggml_free(ctx_eval);

    return prediction;
}

```

This code uses the TensorFlow C++ library to perform binary classification on the MNIST data set. It takes two command-line arguments:

* The first argument is the path to the MNIST data set, in the format "models/mnist/mnist.ggml".
* The second argument is the path to a pre-trained binary classification model, in the format "models/mnist/t10k-images.idx3-ubyte".

The code first sets up the environment and initializes the necessary libraries. Then it reads a random digit from the test set (explicitly defined as "Usage: %s models/mnist/mnist.ggml models/mnist/t10k-images.idx3-ubyte") and renders it in ASCII.

The code uses a helper function `mnist_eval` to perform the binary classification. This function takes the path to the binary classification model and the path to the MNIST data set as arguments. It then returns the predicted digit and class label.

The main function initializes the necessary variables and reads the MNIST data set and the binary classification model from the file systems. It then performs the binary classification using the `mnist_eval` function and outputs the predicted digit and class label.


```cpp
int main(int argc, char ** argv) {
    srand(time(NULL));
    ggml_time_init();

    if (argc != 3) {
        fprintf(stderr, "Usage: %s models/mnist/mnist.ggml models/mnist/t10k-images.idx3-ubyte\n", argv[0]);
        exit(0);
    }

    uint8_t buf[784];
    std::vector<float> digit;

    // read a random digit from the test set
    {
        std::ifstream fin(argv[2], std::ios::binary);
        if (!fin) {
            fprintf(stderr, "%s: failed to open '%s'\n", __func__, argv[2]);
            return 1;
        }

        // seek to a random digit: 16-byte header + 28*28 * (random 0 - 10000)
        fin.seekg(16 + 784 * (rand() % 10000));
        fin.read((char *) &buf, sizeof(buf));
    }

    // render the digit in ASCII
    {
        digit.resize(sizeof(buf));

        for (int row = 0; row < 28; row++) {
            for (int col = 0; col < 28; col++) {
                fprintf(stderr, "%c ", (float)buf[row*28 + col] > 230 ? '*' : '_');
                digit[row*28 + col] = ((float)buf[row*28 + col]);
            }

            fprintf(stderr, "\n");
        }

        fprintf(stderr, "\n");
    }

    const int prediction = mnist_eval(argv[1], 1, digit);

    fprintf(stdout, "%s: predicted digit is %d\n", __func__, prediction);

    return 0;
}

```

# `examples/mnist/main-mtl.cpp`

这段代码的作用是使用 MNIST 计算图在 M1 GPU 上进行推理。它首先通过 `mnist` 工具生成一个计算图，然后使用 `mnist-mtl` 工具将计算图保存为 `mnist.ggml` 文件。这个文件包含了经过 MPS（MPS 是 M1 GPU 的一个硬件加速器）加速的计算图，可以在以后需要时通过 `mnist-mtl` 工具重新使用。


```cpp
// Use a pre-generated MNIST compute graph for inference on the M1 GPU via MPS
//
// You can generate a compute graph using the "mnist" tool:
//
// $ ./bin/mnist ./models/mnist/ggml-model-f32.bin ../examples/mnist/models/mnist/t10k-images.idx3-ubyte
//
// This command creates the "mnist.ggml" file, which contains the generated compute graph.
// Now, you can re-use the compute graph on the GPU with the "mnist-mtl" tool:
//
// $ ./bin/mnist-mtl ./models/mnist/mnist.ggml ../examples/mnist/models/mnist/t10k-images.idx3-ubyte
//

#include "ggml/ggml.h"

#include "main-mtl.h"

```

This is a function to evaluate the predictions of a trained model on new images. It takes a file name of a pre-trained model and a vector of floating-point numbers representing the digits in an image. The function returns the predicted class label.

The function starts by loading the pre-trained model and creating a context for the evaluation. It then loops through the input images and uses the model to predict the class of each image. The


```cpp
#include <cmath>
#include <cstdio>
#include <cstring>
#include <ctime>
#include <fstream>
#include <vector>

// evaluate the MNIST compute graph
//
//   - fname_cgraph: path to the compute graph
//   - digit:        784 pixel values
//
// returns 0 - 9 prediction
int mnist_eval(
        const char * fname_cgraph,
        std::vector<float> digit
        ) {
    // load the compute graph
    struct ggml_context * ctx_data = NULL;
    struct ggml_context * ctx_eval = NULL;

    struct ggml_cgraph * gf = ggml_graph_import(fname_cgraph, &ctx_data, &ctx_eval);

    // allocate work context
    static size_t buf_size = 128ull*1024*1024; // TODO
    static void * buf = malloc(buf_size);

    struct ggml_init_params params = {
        /*.mem_size   =*/ buf_size,
        /*.mem_buffer =*/ buf,
        /*.no_alloc   =*/ false,
    };

    struct ggml_context * ctx_work = ggml_init(params);

    // this allocates all Metal resources and memory buffers
    auto ctx_mtl = mnist_mtl_init(ctx_data, ctx_eval, ctx_work, gf);

    int prediction = -1;

    for (int i = 0; i < 1; ++i) {
        struct ggml_tensor * input = ggml_graph_get_tensor(gf, "input");

        if (i % 2 == 0) {
            memcpy(input->data, digit.data(), ggml_nbytes(input));
        } else {
            memset(input->data, 0, ggml_nbytes(input));
        }

        // the actual inference happens here
        prediction = mnist_mtl_eval(ctx_mtl, gf);
    }

    mnist_mtl_free(ctx_mtl);

    ggml_free(ctx_work);
    ggml_free(ctx_data);
    ggml_free(ctx_eval);

    return prediction;
}

```

这段代码的作用是读取一个MNIST测试集中的图像，对它进行预处理并预测这个图像属于哪个类别，然后输出预测的类别。

具体来说，代码的功能如下：

1. 使用srand函数初始化随机数种子。
2. 调用ggml_time_init函数，开始记录当前时间。
3. 检查命令行参数是否小于3个。如果不是，则输出 Usage: 说明了如何使用这个程序，并退出程序。
4. 读取一个随机的0到10000范围内的数字，并将其存入buf数组中。
5. 对buf数组进行读取，并从28个28x28的范围内随机选择一个数字，并将其存储到digit数组中。
6. 对digit数组进行遍历，将其打印出来。
7. 使用mnist_eval函数，将digit数组中的所有数字输入到模型中，并输出预测的类别。
8. 对结果进行打印，并输出预测的类别。

代码中还定义了一个用来记录开始时间的变量，以及一个用来存储预测类别的变量。


```cpp
int main(int argc, char ** argv) {
    srand(time(NULL));
    ggml_time_init();

    if (argc != 3) {
        fprintf(stderr, "Usage: %s models/mnist/mnist.ggml models/mnist/t10k-images.idx3-ubyte\n", argv[0]);
        exit(0);
    }

    uint8_t buf[784];
    std::vector<float> digit;

    // read a random digit from the test set
    {
        std::ifstream fin(argv[2], std::ios::binary);
        if (!fin) {
            fprintf(stderr, "%s: failed to open '%s'\n", __func__, argv[2]);
            return 1;
        }

        // seek to a random digit: 16-byte header + 28*28 * (random 0 - 10000)
        fin.seekg(16 + 784 * (rand() % 10000));
        fin.read((char *) &buf, sizeof(buf));
    }

    // render the digit in ASCII
    {
        digit.resize(sizeof(buf));

        for (int row = 0; row < 28; row++) {
            for (int col = 0; col < 28; col++) {
                fprintf(stderr, "%c ", (float)buf[row*28 + col] > 230 ? '*' : '_');
                digit[row*28 + col] = ((float)buf[row*28 + col]);
            }

            fprintf(stderr, "\n");
        }

        fprintf(stderr, "\n");
    }

    const int prediction = mnist_eval(argv[1], digit);

    fprintf(stdout, "%s: predicted digit is %d\n", __func__, prediction);

    return 0;
}

```

# `examples/mnist/main.cpp`

这段代码的作用是定义了一个名为 "ggml" 的头文件，以及一些通用的函数和变量。

"ggml.h" 是一个头文件，可能是定义了一些通用类、函数或者常量的头文件。

"common.h" 是另一个头文件，可能定义了一些通用的函数或者变量。

<cmath> 和 <cstdio> 是标准库头文件，可能定义了一些数学函数和输入输出函数。

<cstring> 是 C 语言标准库中的一个头文件，定义了一些字符串操作的函数。

<ctime> 是 C 语言标准库中的一个头文件，定义了一些时间操作的函数。

<fstream> 是 C 语言标准库中的一个头文件，定义了一些文件读取和写入的函数。

<string> 是 C 语言标准库中的一个头文件，定义了一些字符串操作的函数。

<vector> 是 C 语言标准库中的一个头文件，定义了一些向量操作的函数。

<algorithm> 是 C 语言标准库中的一个头文件，定义了一些算法实现的函数。

这些函数和头文件的作用和用途需要根据上下文来确定。


```cpp
#include "ggml/ggml.h"

#include "common.h"

#include <cmath>
#include <cstdio>
#include <cstring>
#include <ctime>
#include <fstream>
#include <string>
#include <vector>
#include <algorithm>

#if defined(_MSC_VER)
#pragma warning(disable: 4244 4267) // possible loss of data
```

这段代码定义了一个用于训练和评估深度学习模型的结构和函数。

这是一个#endif预处理指令，用于检查该代码是否在已经定义的结构中定义了所有的变量。如果没有定义所有变量，该代码将导致编译错误。

该代码定义了一个结构和函数：

结构和函数：mnist_hparams 和 mnist_model

mnist_hparams定义了在训练和评估模型时需要的一些参数。

mnist_model定义了用于在训练和评估中使用的函数和数据结构。

mnist_model中的fc1_weight和fc1_bias存储了输入数据(在fc2_weight和fc2_bias中)和模型的参数。这些参数在模型训练和评估期间进行更新。

mnist_model中的fc2_weight和fc2_bias存储了输入数据(在fc1_weight和fc1_bias中)和模型的参数。这些参数在模型训练和评估期间进行更新。

mnist_model中的ctx存储了一个指向模型的上下文的指针。在训练和评估期间，可以使用该上下文来获取输出数据并执行其他操作。


```cpp
#endif

// default hparams
struct mnist_hparams {
    int32_t n_input   = 784;
    int32_t n_hidden  = 500;
    int32_t n_classes = 10;
};

struct mnist_model {
    mnist_hparams hparams;

    struct ggml_tensor * fc1_weight;
    struct ggml_tensor * fc1_bias;

    struct ggml_tensor * fc2_weight;
    struct ggml_tensor * fc2_bias;

    struct ggml_context * ctx;
};

```

This is a Rust implementation of a simple neural network model that takes in a binary classification task with a budgeted budget.

The model has 2FC1 layers with a fully connected layer and a fc2 layer. The fc2 layer has 2 output neurons corresponding to the 2 classes.

The input data is read in from a file and stored in the input tensor buffer. The input data is then passed through the first fc1 layer.

The first fc1 layer has a getting a vector of dimensions 10 and 500.

The next dimension is 2 which means the input data has 2 parts.

The rest of the code is related to the custom made data.

The custom data is passed through the first fc1 layer and stored in the output tensor buffer.

Then the rest of the code is related to the custom made data, nothing else.


```cpp
// load the model's weights from a file
bool mnist_model_load(const std::string & fname, mnist_model & model) {
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

    auto & ctx = model.ctx;

    size_t ctx_size = 0;

    {
        const auto & hparams = model.hparams;

        const int n_input   = hparams.n_input;
        const int n_hidden  = hparams.n_hidden;
        const int n_classes = hparams.n_classes;

        ctx_size += n_input * n_hidden * ggml_type_sizef(GGML_TYPE_F32); // fc1 weight
        ctx_size +=           n_hidden * ggml_type_sizef(GGML_TYPE_F32); // fc1 bias

        ctx_size += n_hidden * n_classes * ggml_type_sizef(GGML_TYPE_F32); // fc2 weight
        ctx_size +=            n_classes * ggml_type_sizef(GGML_TYPE_F32); // fc2 bias

        printf("%s: ggml ctx size = %6.2f MB\n", __func__, ctx_size/(1024.0*1024.0));
    }

    // create the ggml context
    {
        struct ggml_init_params params = {
            /*.mem_size   =*/ ctx_size + 1024*1024,
            /*.mem_buffer =*/ NULL,
            /*.no_alloc   =*/ false,
        };

        model.ctx = ggml_init(params);
        if (!model.ctx) {
            fprintf(stderr, "%s: ggml_init() failed\n", __func__);
            return false;
        }
    }

    // Read FC1 layer 1
    {
        // Read dimensions
        int32_t n_dims;
        fin.read(reinterpret_cast<char *>(&n_dims), sizeof(n_dims));

        {
            int32_t ne_weight[2] = { 1, 1 };
            for (int i = 0; i < n_dims; ++i) {
                fin.read(reinterpret_cast<char *>(&ne_weight[i]), sizeof(ne_weight[i]));
            }

            // FC1 dimensions taken from file, eg. 768x500
            model.hparams.n_input  = ne_weight[0];
            model.hparams.n_hidden = ne_weight[1];

            model.fc1_weight = ggml_new_tensor_2d(ctx, GGML_TYPE_F32, model.hparams.n_input, model.hparams.n_hidden);
            fin.read(reinterpret_cast<char *>(model.fc1_weight->data), ggml_nbytes(model.fc1_weight));
            ggml_set_name(model.fc1_weight, "fc1_weight");
        }

        {
            int32_t ne_bias[2] = { 1, 1 };
            for (int i = 0; i < n_dims; ++i) {
                fin.read(reinterpret_cast<char *>(&ne_bias[i]), sizeof(ne_bias[i]));
            }

            model.fc1_bias = ggml_new_tensor_1d(ctx, GGML_TYPE_F32, model.hparams.n_hidden);
            fin.read(reinterpret_cast<char *>(model.fc1_bias->data), ggml_nbytes(model.fc1_bias));
            ggml_set_name(model.fc1_bias, "fc1_bias");

            // just for testing purposes, set some parameters to non-zero
            model.fc1_bias->op_params[0] = 0xdeadbeef;
        }
    }

    // Read FC2 layer 2
    {
        // Read dimensions
        int32_t n_dims;
        fin.read(reinterpret_cast<char *>(&n_dims), sizeof(n_dims));

        {
            int32_t ne_weight[2] = { 1, 1 };
            for (int i = 0; i < n_dims; ++i) {
                fin.read(reinterpret_cast<char *>(&ne_weight[i]), sizeof(ne_weight[i]));
            }

            // FC1 dimensions taken from file, eg. 10x500
            model.hparams.n_classes = ne_weight[1];

            model.fc2_weight = ggml_new_tensor_2d(ctx, GGML_TYPE_F32, model.hparams.n_hidden, model.hparams.n_classes);
            fin.read(reinterpret_cast<char *>(model.fc2_weight->data), ggml_nbytes(model.fc2_weight));
            ggml_set_name(model.fc2_weight, "fc2_weight");
        }

        {
            int32_t ne_bias[2] = { 1, 1 };
            for (int i = 0; i < n_dims; ++i) {
                fin.read(reinterpret_cast<char *>(&ne_bias[i]), sizeof(ne_bias[i]));
            }

            model.fc2_bias = ggml_new_tensor_1d(ctx, GGML_TYPE_F32, model.hparams.n_classes);
            fin.read(reinterpret_cast<char *>(model.fc2_bias->data), ggml_nbytes(model.fc2_bias));
            ggml_set_name(model.fc2_bias, "fc2_bias");
        }
    }

    fin.close();

    return true;
}

```

This is a function definition for a neural network model that uses the MLP-based backpropagation algorithm to train the neural network on a binary classification problem. The function is part of a Graph neural network that uses an external "model" with two input features (i.e., "Ax" and "b") and two output features (i.e., "input").

The function first initializes the input tensor and then performs the following operations on the input tensor:

1. Compute the fully-connected (FC1) output of the model using the weight matrix and bias values.
2. Compute the output of the second fully-connected layer using the ReLU activation function.
3. Create a probability tensor by applying the ReLU function to the output of the second fully-connected layer.

The function then performs the soft-max operation to apply the output of the second fully-connected layer to the input tensor, which is expected to be the probability distribution of the input data.

Finally, the function builds the computational graph, computes the forward pass using the ReLU-based graph, and runs the computation graph.

If the user has specified a file name for the computation graph, the function prints the graph to the specified file.


```cpp
// evaluate the model
//
//   - model:     the model
//   - n_threads: number of threads to use
//   - digit:     784 pixel values
//
// returns 0 - 9 prediction
int mnist_eval(
        const mnist_model & model,
        const int n_threads,
        std::vector<float> digit,
        const char * fname_cgraph
        ) {

    const auto & hparams = model.hparams;

    static size_t buf_size = hparams.n_input * sizeof(float) * 32;
    static void * buf = malloc(buf_size);

    struct ggml_init_params params = {
        /*.mem_size   =*/ buf_size,
        /*.mem_buffer =*/ buf,
        /*.no_alloc   =*/ false,
    };

    struct ggml_context * ctx0 = ggml_init(params);
    struct ggml_cgraph * gf = ggml_new_graph(ctx0);

    struct ggml_tensor * input = ggml_new_tensor_1d(ctx0, GGML_TYPE_F32, hparams.n_input);
    memcpy(input->data, digit.data(), ggml_nbytes(input));
    ggml_set_name(input, "input");

    // fc1 MLP = Ax + b
    ggml_tensor * fc1 = ggml_add(ctx0, ggml_mul_mat(ctx0, model.fc1_weight, input),                model.fc1_bias);
    ggml_tensor * fc2 = ggml_add(ctx0, ggml_mul_mat(ctx0, model.fc2_weight, ggml_relu(ctx0, fc1)), model.fc2_bias);

    // soft max
    ggml_tensor * probs = ggml_soft_max(ctx0, fc2);
    ggml_set_name(probs, "probs");

    // build / export / run the computation graph
    ggml_build_forward_expand(gf, probs);
    ggml_graph_compute_with_ctx(ctx0, gf, n_threads);

    //ggml_graph_print   (&gf);
    ggml_graph_dump_dot(gf, NULL, "mnist.dot");

    if (fname_cgraph) {
        // export the compute graph for later use
        // see the "mnist-cpu" example
        ggml_graph_export(gf, "mnist.ggml");

        fprintf(stderr, "%s: exported compute graph to '%s'\n", __func__, fname_cgraph);
    }

    const float * probs_data = ggml_get_data_f32(probs);

    const int prediction = std::max_element(probs_data, probs_data + 10) - probs_data;

    ggml_free(ctx0);

    return prediction;
}

```

这段代码是一个C语言编译器编译器，通过使用C语言特性，它可以将C语言代码编译成具有C语言语法和语义的特性的字节码。以下是这段代码的作用：

1. 在if语句下，定义了一个名为wasm_eval的函数，其参数为传递给它的一个uint8_t指针和一个uint8_t指针。
2. 通过函数指针，调用了一个名为mnist_eval的函数，并接收一个uint8_t指针和一个uint8_t指针作为其参数。
3. 在mnist_eval函数中，使用输入的model加载一个MNIST数据集的模型，并从模型中获取一个1x28x28的数字数据向量。
4. 通过MNIST数据集的模型，输出数字数据向量的正确答案。
5. 在函数指针返回后，将使用者释放输入的model。


```cpp
#ifdef __cplusplus
extern "C" {
#endif

int wasm_eval(uint8_t * digitPtr) {
    mnist_model model;
    if (!mnist_model_load("models/mnist/ggml-model-f32.bin", model)) {
        fprintf(stderr, "error loading model\n");
        return -1;
    }
    std::vector<float> digit(digitPtr, digitPtr + 784);
    int result = mnist_eval(model, 1, digit, nullptr);
    ggml_free(model.ctx);

    return result;
}

```

该代码的作用是：从名为 "models/mnist/t10k-images.idx3-ubyte" 的文件中读取字节，并从文件中随机读取一个 0 到 10000 之间的数字，然后将读取的数字存储在名为 "digitPtr" 的字符指针中，并返回 1，如果期间出现错误则返回 0。

具体来说，代码首先使用 `std::ifstream` 类从文件中读取数据，如果期间文件不存在或读取过程中出现错误，则会输出错误信息并返回 0。接着，使用 `srand` 函数和 `time` 函数（或其他库函数）获取当前时间，并将其作为参数传递给 `rnd` 函数，用于在文件中随机生成一个 0 到 10000 之间的随机整数。

然后，使用 `fin.seekg` 和 `fin.read` 方法从文件中读取读取的数字，并将其存储在 `digitPtr` 指向的连续字节区域。最后，如果整个过程成功，则返回 1，否则返回 0。


```cpp
int wasm_random_digit(char * digitPtr) {
    auto fin = std::ifstream("models/mnist/t10k-images.idx3-ubyte", std::ios::binary);
    if (!fin) {
        fprintf(stderr, "failed to open digits file\n");
        return 0;
    }
    srand(time(NULL));

    // Seek to a random digit: 16-byte header + 28*28 * (random 0 - 10000)
    fin.seekg(16 + 784 * (rand() % 10000));
    fin.read(digitPtr, 784);

    return 1;
}

```

This is a C++ program that uses the GGML (General Graphical Model Library) to perform various operations on a loaded machine learning model, such as reloading it, truncating it, and predicting the digits in the MNIST test set.

The program first checks if the model is loaded from a file called '%s' and if it fails to load, it will print an error message and return 1. Then it reads a random digit from the test set and performs some transformations on it, before rendering it in ASCII.

The program also has a function that loads a random 16-byte header from the MNIST test set and performs some transformations on it. This function is used to read a random digit from the test set and then render it.

Finally, the program has a function that performs some transformations on a loaded model and then predicts the digits in the MNIST test set. This function uses the model to predict the digits and returns the predicted digit as an integer.

Overall, the program is designed to be a useful tool for testing and manipulating machine learning models in the MNIST test set using the GGML library.


```cpp
#ifdef __cplusplus
}
#endif

int main(int argc, char ** argv) {
    srand(time(NULL));
    ggml_time_init();

    if (argc != 3) {
        fprintf(stderr, "Usage: %s models/mnist/ggml-model-f32.bin models/mnist/t10k-images.idx3-ubyte\n", argv[0]);
        exit(0);
    }

    uint8_t buf[784];
    mnist_model model;
    std::vector<float> digit;

    // load the model
    {
        const int64_t t_start_us = ggml_time_us();

        if (!mnist_model_load(argv[1], model)) {
            fprintf(stderr, "%s: failed to load model from '%s'\n", __func__, "models/ggml-model-f32.bin");
            return 1;
        }

        const int64_t t_load_us = ggml_time_us() - t_start_us;

        fprintf(stdout, "%s: loaded model in %8.2f ms\n", __func__, t_load_us / 1000.0f);
    }

    // read a random digit from the test set
    {
        std::ifstream fin(argv[2], std::ios::binary);
        if (!fin) {
            fprintf(stderr, "%s: failed to open '%s'\n", __func__, argv[2]);
            return 1;
        }

        // seek to a random digit: 16-byte header + 28*28 * (random 0 - 10000)
        fin.seekg(16 + 784 * (rand() % 10000));
        fin.read((char *) &buf, sizeof(buf));
    }

    // render the digit in ASCII
    {
        digit.resize(sizeof(buf));

        for (int row = 0; row < 28; row++) {
            for (int col = 0; col < 28; col++) {
                fprintf(stderr, "%c ", (float)buf[row*28 + col] > 230 ? '*' : '_');
                digit[row*28 + col] = ((float)buf[row*28 + col]);
            }

            fprintf(stderr, "\n");
        }

        fprintf(stderr, "\n");
    }

    const int prediction = mnist_eval(model, 1, digit, "mnist.ggml");

    fprintf(stdout, "%s: predicted digit is %d\n", __func__, prediction);

    ggml_free(model.ctx);

    return 0;
}

```

# `examples/mnist/mnist-cnn.py`

This code appears to be an example of how to train a convolutional neural network (CNN) for a handwritten digit classification task. The code uses the TensorFlow and Keras libraries to create a Keras model and then trains it using the Adam optimizer. The model is trained on the input images and their labels, which are one-hot encoded. The final result is the predicted class of each image.

The code starts by defining the input shape and the number of classes, which is 10 in this case (since there are 10 possible digits). The input images are then expanded to a 2D shape of (28, 28, 1) to match the shape of the images in the dataset. This allows the model to take in a full image and extract only the relevant information.

The next step is to normalize the input images and apply the necessary pre-processing steps, such as convolutional and pooling layers. These layers are applied to the input images and then the output of the convolutional layer is flattened and passed through a dropout layer to prevent overfitting.

The final step is to create a model with a single hidden layer that outputs a 10-class prediction. This model is then summarized and saved to disk.

The code assumes that the data is already prepared and that the labels have already been assigned a one-hot encoded value. It is also assumes that the data is divided into training and test sets, and that the test set is not used for training.

Please note that this is a basic example, you should adapt it to your specific use case and data.


```cpp
#!/usr/bin/env python3
import sys
import gguf
import numpy as np
from tensorflow import keras
from tensorflow.keras import layers

def train(model_name):
    # Model / data parameters
    num_classes = 10
    input_shape = (28, 28, 1)

    # Load the data and split it between train and test sets
    (x_train, y_train), (x_test, y_test) = keras.datasets.mnist.load_data()

    # Scale images to the [0, 1] range
    x_train = x_train.astype("float32") / 255
    x_test = x_test.astype("float32") / 255
    # Make sure images have shape (28, 28, 1)
    x_train = np.expand_dims(x_train, -1)
    x_test = np.expand_dims(x_test, -1)
    print("x_train shape:", x_train.shape)
    print(x_train.shape[0], "train samples")
    print(x_test.shape[0], "test samples")

    # convert class vectors to binary class matrices
    y_train = keras.utils.to_categorical(y_train, num_classes)
    y_test = keras.utils.to_categorical(y_test, num_classes)

    model = keras.Sequential(
        [
            keras.Input(shape=input_shape),
            layers.Conv2D(32, kernel_size=(3, 3), activation="relu"),
            layers.MaxPooling2D(pool_size=(2, 2)),
            layers.Conv2D(64, kernel_size=(3, 3), activation="relu"),
            layers.MaxPooling2D(pool_size=(2, 2)),
            layers.Flatten(),
            layers.Dropout(0.5),
            layers.Dense(num_classes, activation="softmax"),
        ]
    )

    model.summary()
    batch_size = 128
    epochs = 15
    model.compile(loss="categorical_crossentropy", optimizer="adam", metrics=["accuracy"])
    model.fit(x_train, y_train, batch_size=batch_size, epochs=epochs, validation_split=0.1)

    score = model.evaluate(x_test, y_test, verbose=0)
    print("Test loss:", score[0])
    print("Test accuracy:", score[1])
    model.save(model_name)
    print("Keras model saved to '" + model_name + "'")

```

It looks like you have provided a code snippet for converting a TensorFlow Keras model to ONNX format. Here's a brief explanation of what's happening in the code:

1. The model weights are being converted from TensorFlow to ONNX format using the `astype()` method.
2. The model is being renamed as specified by the `model.layers[-1].weights[0].numpy()`.
3. The bias for the first layer of the model is being converted from TensorFlow to ONNX format and added to the model using the `add_tensor()` method.
4. The `add_tensor()` method is then writing the header, data, and closing the file.

It's important to note that this code snippet assumes that the TensorFlow Keras model has already been defined and that the ONNX format is being used to save the model. If this is not the case, you may need to modify the code accordingly.


```cpp
def convert(model_name):
    model = keras.models.load_model(model_name)
    gguf_model_name = model_name + ".gguf"
    gguf_writer = gguf.GGUFWriter(gguf_model_name, "mnist-cnn")

    kernel1 = model.layers[0].weights[0].numpy()
    kernel1 = np.moveaxis(kernel1, [2,3], [0,1])
    kernel1 = kernel1.astype(np.float16)
    gguf_writer.add_tensor("kernel1", kernel1, raw_shape=(32, 1, 3, 3))

    bias1 = model.layers[0].weights[1].numpy()
    bias1 = np.repeat(bias1, 26*26)
    gguf_writer.add_tensor("bias1", bias1, raw_shape=(1, 32, 26, 26))

    kernel2 = model.layers[2].weights[0].numpy()
    kernel2 = np.moveaxis(kernel2, [0,1,2,3], [2,3,1,0])
    kernel2 = kernel2.astype(np.float16)
    gguf_writer.add_tensor("kernel2", kernel2, raw_shape=(64, 32, 3, 3))

    bias2 = model.layers[2].weights[1].numpy()
    bias2 = np.repeat(bias2, 11*11)
    gguf_writer.add_tensor("bias2", bias2, raw_shape=(1, 64, 11, 11))

    dense_w = model.layers[-1].weights[0].numpy()
    dense_w = dense_w.transpose()
    gguf_writer.add_tensor("dense_w", dense_w, raw_shape=(10, 1600))

    dense_b = model.layers[-1].weights[1].numpy()
    gguf_writer.add_tensor("dense_b", dense_b)

    gguf_writer.write_header_to_file()
    gguf_writer.write_kv_data_to_file()
    gguf_writer.write_tensors_to_file()
    gguf_writer.close()
    print("Model converted and saved to '{}'".format(gguf_model_name))

```

这段代码是一个条件判断语句，它首先检查是否运行了程序本身（即 __name__ 是否等于 '__main__'）。如果是，就执行程序体内的代码。程序体内有两个条件判断，第一个是检查命令行参数的数量，如果数量小于3个，就输出一条使用说明，然后程序就退出。第二个条件判断是在第一个条件判断失败的情况下，一个是训练（train）一个是转换（convert），根据用户输入的参数来决定执行哪个操作。如果用户输入的第一个参数是 'train"，则训练所选的模型，输入第二个参数；如果第一个参数是 'convert'，则转换所选的模型，输入第二个参数。如果用户输入的第一个参数有误，或者是任何其它情况，程序就会退出并给出一个错误信息。


```cpp
if __name__ == '__main__':
    if len(sys.argv) < 3:
        print("Usage: %s <train|convert> <model_name>".format(sys.argv[0]))
        sys.exit(1)
    if sys.argv[1] == 'train':
        train(sys.argv[2])
    elif sys.argv[1] == 'convert':
        convert(sys.argv[2])
    else:
        print("Usage: %s <train|convert> <model_name>".format(sys.argv[0]))
        sys.exit(1)

```