# GGML源码解析 9

# `examples/gpt-j/quantize.cpp`

这段代码是一个C++程序，旨在实现数据结构的数据结构。

进一步解释：

1. 它包含了多个头文件，包括ggml.h和common.h，这两个头文件定义了ggml和common库，ggml库是一个通用的数学库，common库包含一些通用的函数和变量。

2. 它导入了stdio.h,cmath.h,cassert，和cstring，这些标准库头文件提供了输入/输出函数和数学函数，以及断言和字符串操作的基本功能。

3. 它导入了fstream，这个标准库头文件提供了文件输入/输出的功能，并支持二进制文件、文本文件和二进制文件。

4. 它使用了名字作为函数参数的函数重载，这允许程序在不同的函数中使用同名的函数，而不会导致函数名称冲突。

5. 它使用了C++11中的constexpr类型，使得一些变量在编译期就可以确定值，从而提高程序的性能。

6. 它使用了正则表达式，通过它可以从文件中读取ID3编码格式的标签数据，并提取出相应的文本内容，以便进行搜索和分析。


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

This function is used to load a graphical model from an input file, represented by the GOML(Model-File Optimization Language) format. The function takes as input the path to the model file and outputs a graphical model object, represented by the QuantizedObject struct.

The function first reads the model file and checks that it is valid. If the file is invalid, the function prints an error message and returns false.

The function then reads the model file and quantity data. It reads the model file's metadata, such as the number of layers, the number of nodes per layer, the network type, and the word map.

The function also reads the model file's text data, such as the layers' descriptions and the layers' indices.

Finally, the function reads the model file's text data, such as the weights and the names of the weights.

The function then checks that the model file has been read valid, and then it attempts to quantize the graphical model using the QuantizeOptimizer function. If the quantization process fails, the function prints an error message and returns false.

If the function successfully quantizes the graphical model, it returns true.


```cpp
// default hparams (GPT-J 6B)
struct gptj_hparams {
    int32_t n_vocab = 50400;
    int32_t n_ctx   = 2048;
    int32_t n_embd  = 4096;
    int32_t n_head  = 16;
    int32_t n_layer = 28;
    int32_t n_rot   = 64;
    int32_t ftype   = 1;
};

// quantize a model
bool gptj_model_quantize(const std::string & fname_inp, const std::string & fname_out, ggml_ftype ftype) {
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

    gptj_hparams hparams;

    // load hparams
    {
        finp.read((char *) &hparams.n_vocab, sizeof(hparams.n_vocab));
        finp.read((char *) &hparams.n_ctx,   sizeof(hparams.n_ctx));
        finp.read((char *) &hparams.n_embd,  sizeof(hparams.n_embd));
        finp.read((char *) &hparams.n_head,  sizeof(hparams.n_head));
        finp.read((char *) &hparams.n_layer, sizeof(hparams.n_layer));
        finp.read((char *) &hparams.n_rot,   sizeof(hparams.n_rot));
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
        fout.write((char *) &hparams.n_rot,   sizeof(hparams.n_rot));
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

This is a C++ program that uses the Model-f32 and Model-quant packages of the GPT-J model. It takes a model-f32.bin and model-quant.bin file as input and a type parameter to specify the output type.

The program first checks the input files and initializes the f16 tables using the Model-f32 and Model-quant packages. It then takes as input the quantized model file and the output file name.

The program quantizes the quantized model file using the Model-quant package and reports the timing information at the end of the quantization process.

It is worth noting that this program requires the GPT-J model to be installed on the system and that the Model-f32 and Model-quant packages must be specified separately.


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

        if (!gptj_model_quantize(fname_inp, fname_out, ggml_ftype(ftype))) {
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

# gpt-j

Local GPT-J inference on your computer using C/C++

No video card required. You just need to have 16 GB of RAM.

## Motivation

The GPT-J 6B model is the open-source alternative to OpenAI's GPT-3. It's basically a neural network that allows you to
generate coherent, human-like text given a certain context (prompt).

The GPT-J model is quite big - the compact version of the model uses 16-bit floating point representation of the weights
and is still 12 GB big. This means that in order to run inference on your computer, you would need to have a video card
with at least 12 GB of video RAM. Alternatively, you can try to run the python implementations on the CPU, but that
would probably not be very efficient as they are primarily optimized for running on a GPU (or at least this is my guess -
I don't have much experience with python).

I wanted to try and run the model on my MacBook, so I decided to implement the model inference from scratch using my own
custom build tensor library. The tensor library (called [ggml](https://github.com/ggerganov/ggml), written in C) is in
early development stage, but it already allows me to run the GPT-J model.

On my 32GB MacBook M1 Pro, I achieve an inference speed of about `125 ms/token` or about ~6 words per second (1 word
typically consists of 1 or 2 tokens).

Here is a sample run with prompt `int main(int argc, char ** argv) {`:

```cpp
$ time ./bin/gpt-j -p "int main(int argc, char ** argv) {"

gptj_model_load: loading model from 'models/gpt-j-6B/ggml-model.bin' - please wait ...
gptj_model_load: n_vocab = 50400
gptj_model_load: n_ctx   = 2048
gptj_model_load: n_embd  = 4096
gptj_model_load: n_head  = 16
gptj_model_load: n_layer = 28
gptj_model_load: n_rot   = 64
gptj_model_load: f16     = 1
gptj_model_load: ggml ctx size = 13334.86 MB
gptj_model_load: memory_size =  1792.00 MB, n_mem = 57344
gptj_model_load: ................................... done
gptj_model_load: model size = 11542.79 MB / num tensors = 285
main: number of tokens in prompt = 13

int main(int argc, char ** argv) {
    (void)argc;
    (void)argv;

    {
        struct sockaddr_in addr;
        int addrlen;
        char * ip = "192.168.1.4";
        int i;

        if ( (addrlen = sizeof(addr)) == -1 )
            return -1;

        for (i = 0; i < 10; ++i) {
            addr.sin_family = AF_INET;
            addr.sin_addr.s_addr = inet_addr(ip);

main: mem per token = 16430420 bytes
main:     load time =  6211.48 ms
main:   sample time =    13.74 ms
main:  predict time = 26420.34 ms / 124.62 ms per token
main:    total time = 33035.37 ms

real	0m33.171s
user	3m32.269s
sys      0m3.686s

$
```

It took ~6.2 seconds to load the model to memory. After that, it took ~26.4 seconds to generate 200 tokens of what
looks like to be the beginning of a networking program in C. Pretty cool!

Here is another run, just for fun:

```cpp
time ./bin/gpt-j -n 500 -t 8 -p "Ask HN: Inherited the worst code and tech team I have ever seen. How to fix it?
"

gptj_model_load: loading model from 'models/gpt-j-6B/ggml-model.bin' - please wait ...
gptj_model_load: n_vocab = 50400
gptj_model_load: n_ctx   = 2048
gptj_model_load: n_embd  = 4096
gptj_model_load: n_head  = 16
gptj_model_load: n_layer = 28
gptj_model_load: n_rot   = 64
gptj_model_load: f16     = 1
gptj_model_load: ggml ctx size = 13334.86 MB
gptj_model_load: memory_size =  1792.00 MB, n_mem = 57344
gptj_model_load: ................................... done
gptj_model_load: model size = 11542.79 MB / num tensors = 285
main: number of tokens in prompt = 24

Ask HN: Inherited the worst code and tech team I have ever seen. How to fix it?

I've inherited a team with some very strange and un-documented practices, one of them is that they use an old custom
application with a very slow tech stack written in Python that the team doesn't want to touch but also doesn't want to
throw away as it has some "legacy" code in it.

The problem is, the tech stack is very very slow.

They have a single web server on a VM that is slow.
The server is a little bit busy (not very busy though) and they have a lot of processes (30+ that are constantly being
spawned by the application)
They have an application that is single threaded and was written in Python and the team don't want to touch this, and
the application is very slow.

My task as a new member of the team is to fix this.

I'm a senior dev on the team (3 years on the project) and have been told that I will take the lead on this task. I know
next to nothing about Python. So here is what I have so far.

What I have done is I've been trying to debug the processes with the "ps" command. This way I can see what is running
and where. From what I see, the application spawns 10 processes a minute and some of them are used for nothing.

I have also started to look for the code. The application source is not in GitHub or any other repository, it is only on
our internal GitLab.

What I've found so far:

The application uses a custom SQLAlchemy implementation to interact with the data. I've looked at the source, it looks
like an object cache or something like that. But from what I've seen, the cache gets full every 20 minutes and then gets
cleared with a special command.

Another strange thing is that the application creates a file for every entry in the database (even if the entry already
exists). I've looked at the file to see if it contains something, but it seems to be a JSON file with lots of records.

The other strange thing is that I can only find the database tables in the GitLab repository and not the code. So I
can't really understand how the application is supposed to interact with the database.

I also found a "log" directory, but the code is encrypted with AES. From what I've found, it is in

main: mem per token = 16430420 bytes
main:     load time =  3900.10 ms
main:   sample time =    32.58 ms
main:  predict time = 68049.91 ms / 130.11 ms per token
main:    total time = 73020.05 ms

real	1m13.156s
user	9m1.328s
sys.    0m7.103s
```

## Implementation details

The high level implementation of the model is contained in the [main.cpp](main.cpp) file. The core computations are
performed by the [ggml](https://github.com/ggerganov/ggml/blob/master/include/ggml/ggml.h) library.


#### Matrix multiplication

The most performance critical part of the implementation is of course the matrix multiplication routine. 99% of the time
is spent here, so it was important to optimize this as much as possible.

On Arm64, I utilize the 128-bit NEON intrinsics for 16-bit floating point operations:

https://github.com/ggerganov/ggml/blob/fb558f78d905f85c54813602649ddd628ffe0f3a/src/ggml.c#L187-L243

These instructions allow each core to operate simultaneously on 64 16-bit floats. I'm no expert in SIMD, but after quite
some trials this was the most efficient code for dot product of a row and column that I could come up with. Combined
with the parallel computation on 8 CPU threads, I believe I'm close to the maximum performance that one could possibly
get on the M1 CPU. Still, I'm curious to know if there is a more efficient way to implement this.


#### Attempt to use the M1 GPU

One interesting property of the GPT-J transformer architecture is that it allows you to perform part of the inference in
parallel - i.e. the Feed-forward network can be computed in parallel to the Self-attention layer:

https://github.com/ggerganov/ggml/blob/fb558f78d905f85c54813602649ddd628ffe0f3a/examples/gpt-j/main.cpp#L507-L531

So I thought why not try and bring in the M1 GPU to compute half of the neural network in parallel to the CPU and
potentially gain some extra performance. Thanks to the M1's shared memory model, it was relatively easy to offload part
of the computation to the GPU using Apple's [Metal Performance
Shaders](https://developer.apple.com/documentation/metalperformanceshaders). The GPU shares the host memory, so there is
no need to copy the data back and forth as you would normally do with Cuda or OpenCL. The weight matrices are directly
available to be used by the GPU.

However, to my surprise, using MPS together with the CPU did not lead to any performance improvement at all. My
conclusion was that the 8-thread NEON CPU computation is already saturating the memory bandwidth of the M1 and since
the CPU and the GPU on the MacBook are sharing that bandwidth, it does not help to offload the computation to the GPU.
Another observation was that the MPS GPU matrix multiplication using 16-bit floats had the same performance as the
8-thread NEON CPU implementation. Again, I explain this with a saturated memory channel. But of course, my explanation
could be totally wrong and somehow the implementation wasn't utilizing the resources correctly.

In the end, I decided to not use MPS or the GPU all together.

### Zero memory allocations

Another property of my implementation is that it does not perform any memory allocations once the model is loaded into
memory. All required memory is allocated at the start of the program with a single `malloc` (technically 2 calls, but
that is not important).

## Usage

If you want to give this a try and you are on Linux or Mac OS, simply follow these instructions:

```cppbash
# Clone the ggml library and build the gpt-j example
git clone https://github.com/ggerganov/ggml
cd ggml
mkdir build && cd build
cmake ..
make -j4 gpt-j

# Download the ggml-compatible GPT-J 6B model (requires 12GB disk space)
../examples/gpt-j/download-ggml-model.sh 6B

# Run the inference (requires 16GB of CPU RAM)
./bin/gpt-j -m models/gpt-j-6B/ggml-model.bin -p "This is an example"

# Input prompt through pipe and run the inference.
echo "This is an example" > prompt.txt
cat prompt.txt | ./bin/gpt-j -m models/gpt-j-6B/ggml-model.bin
```

To run the `gpt-j` tool, you need the 12GB `ggml-model.bin` file which contains the GPT-J model in
[ggml](https://github.com/ggerganov/ggml) compatible format. In the instructions above, the binary file
is downloaded from my repository on Hugging Face using the [download-ggml-model.sh](download-ggml-model.sh) script.
You can also, download the file manually from this link:

https://huggingface.co/ggerganov/ggml/tree/main

---

Alternatively, if you don't want to download the 12GB ggml model file, you can perform the conversion yourself using
python.

First, you need to download the full GPT-J model from here: https://huggingface.co/EleutherAI/gpt-j-6B

Note that the full model is quite big - about 72 GB. After you download it, you need to convert it to ggml format using
the [convert-h5-to-ggml.py](convert-h5-to-ggml.py) script. This will generate the `ggml-model.bin` file, which you can
then use with the `gpt-j` program.


## GPT-2

I also implemented a tool for CPU inference using the smaller GPT-2 models. They have worse quality compared to GPT-J,
but are much faster to execute.

For example, the Small GPT-2 model is only 240 MB big and the inference speed on my MacBook is about 200 tokens/sec.

For more details, checkout the GPT-2 example here: [gpt-2](https://github.com/ggerganov/ggml/tree/master/examples/gpt-2)


# `examples/gpt-neox/convert-h5-to-ggml.py`

这段代码的作用是将HDF5格式的训练数据转换为GGML格式的文件，并保存到指定的目录中。它主要通过以下步骤实现：

1. 导入需要用到的库：包括Python标准库中的sys、struct、json、numpy，以及transformers库中的AutoModelForCausalLM和AutoTokenizer。

2. 检查命令行参数的数量，如果少于3个，则输出帮助信息并退出。

3. 如果传入了一个目录参数，则读取并保存训练数据。

4. 将训练数据从HDF5格式的文件中读取并转换为JSON格式的数据。

5. 将JSON格式的数据转换为NumPy数组。

6. 将NumPy数组转换为训练数据所需的format，即从float32到float16的格式。

7. 将训练数据从NumPy数组转换为模型所需的format，即AutoModelForCausalLM和AutoTokenizer所需的格式。

8. 将转换后的训练数据保存到指定的目录中。


```cpp
import sys
import struct
import json
import numpy as np

from transformers import AutoModelForCausalLM, AutoTokenizer

if len(sys.argv) < 3:
    print("Usage: convert-h5-to-ggml.py dir-model [use-f32]\n")
    print("  ftype == 0 -> float32")
    print("  ftype == 1 -> float16")
    sys.exit(1)

# output in the same directory as the model
dir_model = sys.argv[1]
```

这段代码的作用是读取一个名为"config.json"的JSON文件，并从中读取一个参数(通常是第二个参数)，然后根据读取的参数类型来确定数据类型类型，并将其存储在变量"ftype"中。接着，它使用"with open"语句打开一个名为"dir_model"的目录，目录下有一个名为"config.json"的文件，然后使用"json.load"方法从文件中读取JSON数据并存储在变量"hparams"中。

接下来，它定义了一个可能的输入数据类型列表"ftype_str"，其中包含两种数据类型(float32和float16)。然后，它根据从第二个参数中读取的值，判断该参数所属的数据类型，并将其存储在"ftype"变量中。最后，它根据ftype变量来生成输出文件名，并使用"sys.argv"列表来获取第一个参数的值作为输出文件名。


```cpp
fname_out = sys.argv[1] + "/ggml-model.bin"

with open(dir_model + "/config.json", "r", encoding="utf-8") as f:
    hparams = json.load(f)

# possible data types
#   ftype == 0 -> float32
#   ftype == 1 -> float16
#
# map from ftype to string
ftype_str = ["f32", "f16"]

ftype = 1
if len(sys.argv) > 2:
    ftype = int(sys.argv[2])
    if ftype < 0 or ftype > 1:
        print("Invalid ftype: " + str(ftype))
        sys.exit(1)
    fname_out = sys.argv[1] + "/ggml-model-" + ftype_str[ftype] + ".bin"


```

这段代码的作用是训练一个用于生成文本的模型。模型类型为 "AutoModelForCausalLM"，使用了导演模型（dir_model）预训练，并在训练时使用低CPU内存使用率。同时，还抓取了导演模型的参数列表和模型中的变量列表。

然后，将这些变量的列表序列化为字节流对象，并将其写入一个文件中。文件名为 hparams.txt，文件类型为二进制。在文件中，使用了 magic: ggml。

这个代码的主要目的是在训练模型之后，将模型的参数和变量序列化为字节流对象，并将其保存到 hparams.txt 文件中，以便在未来的使用中，可以重新加载模型并进行测试。


```cpp
tokenizer = AutoTokenizer.from_pretrained(dir_model)
model = AutoModelForCausalLM.from_pretrained(dir_model, low_cpu_mem_usage=True)

list_vars = model.state_dict()
for name in list_vars.keys():
    print(name, list_vars[name].shape, list_vars[name].dtype)

fout = open(fname_out, "wb")

print(hparams)

fout.write(struct.pack("i", 0x67676d6c)) # magic: ggml in hex
fout.write(struct.pack("i", hparams["vocab_size"]))
fout.write(struct.pack("i", hparams["max_position_embeddings"]))
fout.write(struct.pack("i", hparams["hidden_size"]))
```

This is a Python script that processes the data of a given variable by first tokenizing it using the pre-trained tokenizer and then storing the float data in a file. The script takes in a header dictionary (list of variable names and their corresponding data types), which is used to store the header information.

The script first imports the required libraries: `tokenizer`, `numpy`, `struct`, and `math`. The script then defines the header dictionary and the input data.

The script then loops through the header dictionary and extracts the variable name and its corresponding data type. The script then converts the data type to the appropriate ftype (float16 or float32) based on the header information.

The script then writes the header information and the data to the output file. The header information includes the dimensions of the input data, the variable name, and the data type.

The script then loops through the input data and writes it to the output file. The data is written in a format that is appropriate for the data type specified in the header information.

This script is useful for processing variable data and storing it in a file in a way that is compatible with the data type specified in the header information.


```cpp
fout.write(struct.pack("i", hparams["num_attention_heads"]))
fout.write(struct.pack("i", hparams["num_hidden_layers"]))
fout.write(struct.pack("i", int(hparams["rotary_pct"]*(hparams["hidden_size"]//hparams["num_attention_heads"]))))
fout.write(struct.pack("i", hparams["use_parallel_residual"] if "use_parallel_residual" in hparams else True))
fout.write(struct.pack("i", ftype))

# TODO: temporary hack to not deal with implementing the tokenizer
for i in range(hparams["vocab_size"]):
    text = tokenizer.decode([i]).encode('utf-8')
    fout.write(struct.pack("i", len(text)))
    fout.write(text)

for name in list_vars.keys():
    data = list_vars[name].squeeze().numpy()
    print("Processing variable: " + name + " with shape: ", data.shape)

    # we don't need these
    if name.endswith(".attention.masked_bias") or     \
       name.endswith(".attention.bias") or \
       name.endswith(".attention.rotary_emb.inv_freq"):
        print("  Skipping variable: " + name)
        continue

    n_dims = len(data.shape)

    # ftype == 0 -> float32, ftype == 1 -> float16
    ftype_cur = 0
    if ftype != 0:
        if name[-7:] == ".weight" and n_dims == 2:
            print("  Converting to float16")
            data = data.astype(np.float16)
            ftype_cur = 1
        else:
            print("  Converting to float32")
            data = data.astype(np.float32)
            ftype_cur = 0
    else:
        if data.dtype != np.float32:
            print("  Converting to float32")
            data = data.astype(np.float32)
            ftype_cur = 0

    # header
    str = name.encode('utf-8')
    fout.write(struct.pack("iii", n_dims, len(str), ftype_cur))
    for i in range(n_dims):
        fout.write(struct.pack("i", data.shape[n_dims - 1 - i]))
    fout.write(str)

    # data
    data.tofile(fout)

```

这段代码的作用是关闭名为 "fout" 的文件，并输出一个消息，指出文件已经完成输出，并提供了输出文件的文件名。

具体来说，第一行使用 `fout.close()` 方法关闭了文件输出流，这意味着文件输出操作已经完成，不能继续向文件中写入数据。

第二行使用 `print()` 函数输出了一个消息，这个消息由两个空格组成，第一个空格有一个字符串 "Done" 和一个空格，第二个空格包含一个文件名 "fname_out" 的值。这个字符串表示文件已经完成输出，可以进行其他操作了。

第三行也是一个空行，没有对文件或任何变量做出实际的操作。


```cpp
fout.close()

print("Done. Output file: " + fname_out)
print("")

```

# `examples/gpt-neox/main.cpp`

这段代码是一个C++程序，它包括了两个头文件和两个函数指针变量。

第一个头文件 "ggml/ggml.h" 很可能是ggml库的头文件，它定义了一些ggml库的标准函数和变量，ggml库可能是一个流行的C++机器学习库。

第二个头文件 "common/common.h" 可能是一个通用的头文件，定义了一些通用的数据结构和函数，可能是其他库或自己编写的头文件。

两个函数指针变量都指向同一个函数，可能是ggml库中的一个训练或预测函数。函数指针变量通常用于让用户传入函数地址，并在需要时调用函数。

在main函数中，首先通过包含头文件 "common/common.h" 来使该库中的通用水準函数和数据结构变得可用。然后，它定义了一个int类型的变量zzz，并按照ggml库中的训练函数训练模型，将数据集A转换为ggml模型，并预测数据点B的值。

虽然通用的头文件和函数指针变量提供了ggml库中的一些功能，但具体的行为和能力取决于ggml库的实现。


```cpp
#include "ggml/ggml.h"

#include "common.h"
#include "common-ggml.h"

#include <cassert>
#include <cmath>
#include <cstdio>
#include <cstring>
#include <cinttypes>
#include <fstream>
#include <map>
#include <string>
#include <vector>

```

这段代码是一个C语言代码，它包含了一些头文件和声明。

头文件：
```cpp
#include <cstdint>
#include <cassert>
```

声明：
```cpp
#if defined(_MSC_VER)
   #pragma warning(disable: 4244 4267) // possible loss of data
#endif
```

这段代码的作用是定义了一个名为`gpt_neox_hparams`的结构体，其中包含了用于训练语言模型的几个重要参数。

具体来说，这个结构体包含了以下参数：

* `n_vocab`：词汇表的大小，即模型能够识别的单词数量。这个值在代码中没有具体的实现，可能是从其他地方获取的常数。
* `n_ctx`：当前上下文的大小，即当前正在处理的部分文本长度。同样地，这个值在代码中没有具体的实现。
* `n_embd`：嵌入层的大小，即每个词的词向量大小。同样地，这个值在代码中没有具体的实现。
* `n_head`：头部的数量，即每个词的上下文数量。同样地，这个值在代码中没有具体的实现。
* `n_layer`：层数，即神经网络的层数。同样地，这个值在代码中没有具体的实现。
* `n_rot`：旋转百分比，即词汇表中词汇的数量与当前上下文处理部分的词汇数量之比。同样地，这个值在代码中没有具体的实现。
* `par_res`：是否使用残缺模型，如果是，则此值为1，否则为0。同样地，这个值在代码中没有具体的实现。
* `fty`：是否使用因子分解，如果是，则此值为1，否则为0。同样地，这个值在代码中没有具体的实现。
* `eps`：保留的小数位数，即对输入数据进行四舍五入时的小数位数。同样地，这个值在代码中没有具体的实现。


```cpp
#if defined(_MSC_VER)
#pragma warning(disable: 4244 4267) // possible loss of data
#endif

// default hparams (StableLM 3B)
struct gpt_neox_hparams {
    int32_t n_vocab = 50257;
    int32_t n_ctx   = 4096;
    int32_t n_embd  = 4096;
    int32_t n_head  = 32;
    int32_t n_layer = 16;
    int32_t n_rot   = 32; // rotary_pct * (n_embd / n_head)
    int32_t par_res = 1; // 1 = true, 0 = false
    int32_t ftype   = 1;
    float   eps     = 1e-5f;
};

```

这是一个名为`gpt_neox_layer`的结构体，它用于表示深度学习模型中的一个神经网络层。

这个结构体包含了以下信息：

* 前向网络中的`ln_1_g`和`ln_1_b`是两个`ggml_tensor`，它们表示输入的原始数据。
* 注意力机制中的`c_attn_attn_w`、`c_attn_attn_b`和`c_attn_proj_w`、`c_attn_proj_b`是四个`ggml_tensor`，它们表示注意力权重和注意力头。
* 前向网络中的`ln_2_g`和`ln_2_b`是两个`ggml_tensor`，它们表示输入的原始数据。
* 带宽函数（即`c_mlp_fc_w`和`c_mlp_fc_b`）是两个`ggml_tensor`，它们表示输入的原始数据。
* 前向网络中的`c_attn_proj_w`和`c_attn_proj_b`是两个`ggml_tensor`，它们表示注意力头。


```cpp
struct gpt_neox_layer {
    // pre normalization
    struct ggml_tensor * ln_1_g;
    struct ggml_tensor * ln_1_b;

    // attention
    struct ggml_tensor * c_attn_attn_w;
    struct ggml_tensor * c_attn_attn_b;

    struct ggml_tensor * c_attn_proj_w;
    struct ggml_tensor * c_attn_proj_b;

    // post normalization
    struct ggml_tensor * ln_2_g;
    struct ggml_tensor * ln_2_b;

    // ff
    struct ggml_tensor * c_mlp_fc_w;
    struct ggml_tensor * c_mlp_fc_b;

    struct ggml_tensor * c_mlp_proj_w;
    struct ggml_tensor * c_mlp_proj_b;
};

```

这段代码定义了一个名为 `gpt_neox_model` 的结构体，表示深度学习模型中的一个组件。

这个结构体包含以下几个部分：

1. `hparams`：表示模型超参数，包括 `num_layers`、`hidden_size`、`num_attention_heads` 等。
2. `ln_f_g` 和 `ln_f_b`：表示两个注意力头，可能用于计算上下文注意力。
3. `wte`：表示位置嵌入，用于在文本中插入位置标记。
4. `lmh_g` 和 `lmh_b`：表示语言模型头和 bias，分别用于预测文本下一词和预测文本标签。
5. `layers`：是一个 `std::vector<gpt_neox_layer>` 类型的层，表示整个模型的层数。
6. `memory_k` 和 `memory_v`：是一个 `std::vector<ggml_tensor>` 类型的键值对，用于存储模型参数的内存位置。
7. `ctx`：表示模型的上下文，用于在训练和推理时更新模型参数。
8. `tensors`：是一个 `std::map<std::string, struct ggml_tensor *>` 类型的键值对，用于存储模型参数的内存位置。

这个结构体可以用于定义一个 GPT-Neox 模型类，以及定义该模型的 forward 函数。


```cpp
struct gpt_neox_model {
    gpt_neox_hparams hparams;

    // normalization
    struct ggml_tensor * ln_f_g;
    struct ggml_tensor * ln_f_b;

    struct ggml_tensor * wte; // position embedding

    struct ggml_tensor * lmh_g; // language model head
    //struct ggml_tensor * lmh_b; // language model bias

    std::vector<gpt_neox_layer> layers;

    // key + value memory
    struct ggml_tensor * memory_k;
    struct ggml_tensor * memory_v;

    //
    struct ggml_context * ctx;
    std::map<std::string, struct ggml_tensor *> tensors;
};

```

This is a function that checks the shape and size of a tensor in a model file. It takes a single argument of type Tensor, which is a 2-dimensional tensor that represents a tensor in the model file.

The function first checks that the tensor has the correct shape by comparing it to the expected shape specified in the `Tensor` parameter. If the shapes do not match, the function prints an error message and returns `false`.

The function then calculates the size of the tensor and the number of elements in it. If the size or number of elements does not match, the function also prints an error message and returns `false`.

Finally, the function prints the total size of the tensor and the number of tensors in the model file.

It is important to note that the function assumes that the input `Tensor` is properly initialized and passed to the function.


```cpp
// load the model's weights from a file
bool gpt_neox_model_load(const std::string & fname, gpt_neox_model & model, gpt_vocab & vocab) {
    printf("%s: loading model from '%s' - please wait ...\n", __func__, fname.c_str());

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
        fin.read((char *) &hparams.n_rot,   sizeof(hparams.n_rot));
        fin.read((char *) &hparams.par_res, sizeof(hparams.par_res));
        fin.read((char *) &hparams.ftype,   sizeof(hparams.ftype));

        const int32_t qntvr = hparams.ftype / GGML_QNT_VERSION_FACTOR;

        printf("%s: n_vocab = %d\n", __func__, hparams.n_vocab);
        printf("%s: n_ctx   = %d\n", __func__, hparams.n_ctx);
        printf("%s: n_embd  = %d\n", __func__, hparams.n_embd);
        printf("%s: n_head  = %d\n", __func__, hparams.n_head);
        printf("%s: n_layer = %d\n", __func__, hparams.n_layer);
        printf("%s: n_rot   = %d\n", __func__, hparams.n_rot);
        printf("%s: par_res = %d\n", __func__, hparams.par_res);
        printf("%s: ftype   = %d\n", __func__, hparams.ftype);
        printf("%s: qntvr   = %d\n", __func__, qntvr);

        hparams.ftype %= GGML_QNT_VERSION_FACTOR;
    }

    // load vocab
    {
        const int32_t n_vocab = model.hparams.n_vocab;

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

    size_t ctx_size = 0;

    {
        const auto & hparams = model.hparams;

        const size_t n_embd  = hparams.n_embd;
        const size_t n_layer = hparams.n_layer;
        const size_t n_ctx   = hparams.n_ctx;
        const size_t n_vocab = hparams.n_vocab;

        ctx_size += n_embd*ggml_type_sizef(GGML_TYPE_F32); // ln_f_g
        ctx_size += n_embd*ggml_type_sizef(GGML_TYPE_F32); // ln_f_b

        ctx_size += n_embd*n_vocab*ggml_type_sizef(wtype); // wte

        ctx_size += n_embd*n_vocab*ggml_type_sizef(wtype);           // lmh_g
        //ctx_size +=        n_vocab*ggml_type_sizef(GGML_TYPE_F32); // lmh_b

        ctx_size += n_layer*(n_embd*ggml_type_sizef(GGML_TYPE_F32)); // ln_1_g
        ctx_size += n_layer*(n_embd*ggml_type_sizef(GGML_TYPE_F32)); // ln_1_b

        ctx_size += n_layer*(3*n_embd*n_embd*ggml_type_sizef(wtype));         // c_attn_attn_w
        ctx_size += n_layer*(       3*n_embd*ggml_type_sizef(GGML_TYPE_F32)); // c_attn_attn_b

        ctx_size += n_layer*(n_embd*n_embd*ggml_type_sizef(wtype));         // c_attn_proj_w
        ctx_size += n_layer*(n_embd*n_embd*ggml_type_sizef(GGML_TYPE_F32)); // c_attn_proj_b

        ctx_size += n_layer*(n_embd*ggml_type_sizef(GGML_TYPE_F32)); // ln_2_g
        ctx_size += n_layer*(n_embd*ggml_type_sizef(GGML_TYPE_F32)); // ln_2_b

        ctx_size += n_layer*(4*n_embd*n_embd*ggml_type_sizef(wtype));         // c_mlp_fc_w
        ctx_size += n_layer*(       4*n_embd*ggml_type_sizef(GGML_TYPE_F32)); // c_mlp_fc_b

        ctx_size += n_layer*(4*n_embd*n_embd*ggml_type_sizef(wtype));         // c_mlp_proj_w
        ctx_size += n_layer*(         n_embd*ggml_type_sizef(GGML_TYPE_F32)); // c_mlp_proj_b

        ctx_size += n_ctx*n_layer*n_embd*ggml_type_sizef(GGML_TYPE_F32); // memory_k
        ctx_size += n_ctx*n_layer*n_embd*ggml_type_sizef(GGML_TYPE_F32); // memory_v

        ctx_size += (6 + 16*n_layer)*1024; // object overhead

        printf("%s: ggml ctx size = %6.2f MB\n", __func__, ctx_size/(1024.0*1024.0));
    }

    // create the ggml context
    {
        struct ggml_init_params params = {
            /*.mem_size   =*/ ctx_size,
            /*.mem_buffer =*/ NULL,
            /*.no_alloc   =*/ false,
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
        const int n_vocab = hparams.n_vocab;

        model.layers.resize(n_layer);

        model.wte    = ggml_new_tensor_2d(ctx, wtype,         n_embd, n_vocab);

        model.ln_f_g = ggml_new_tensor_1d(ctx, GGML_TYPE_F32, n_embd);
        model.ln_f_b = ggml_new_tensor_1d(ctx, GGML_TYPE_F32, n_embd);

        model.lmh_g  = ggml_new_tensor_2d(ctx, wtype,         n_embd, n_vocab);
        //model.lmh_b  = ggml_new_tensor_1d(ctx, GGML_TYPE_F32, n_vocab);

        // map by name
        model.tensors["gpt_neox.embed_in.weight"] = model.wte;

        model.tensors["gpt_neox.final_layer_norm.weight"] = model.ln_f_g;
        model.tensors["gpt_neox.final_layer_norm.bias"]   = model.ln_f_b;

        model.tensors["embed_out.weight"] = model.lmh_g;
        //model.tensors["lm_head.bias"]   = model.lmh_b;

        for (int i = 0; i < n_layer; ++i) {
            auto & layer = model.layers[i];

            layer.ln_1_g          = ggml_new_tensor_1d(ctx, GGML_TYPE_F32,   n_embd);
            layer.ln_1_b          = ggml_new_tensor_1d(ctx, GGML_TYPE_F32,   n_embd);

            layer.c_attn_attn_w   = ggml_new_tensor_2d(ctx, wtype,           n_embd, 3*n_embd);
            layer.c_attn_attn_b   = ggml_new_tensor_1d(ctx, GGML_TYPE_F32, 3*n_embd);

            layer.c_attn_proj_w   = ggml_new_tensor_2d(ctx, wtype,           n_embd,   n_embd);
            layer.c_attn_proj_b   = ggml_new_tensor_1d(ctx, GGML_TYPE_F32,   n_embd);

            layer.ln_2_g          = ggml_new_tensor_1d(ctx, GGML_TYPE_F32,   n_embd);
            layer.ln_2_b          = ggml_new_tensor_1d(ctx, GGML_TYPE_F32,   n_embd);

            layer.c_mlp_fc_w      = ggml_new_tensor_2d(ctx, wtype,           n_embd, 4*n_embd);
            layer.c_mlp_fc_b      = ggml_new_tensor_1d(ctx, GGML_TYPE_F32, 4*n_embd);

            layer.c_mlp_proj_w    = ggml_new_tensor_2d(ctx, wtype,         4*n_embd,   n_embd);
            layer.c_mlp_proj_b    = ggml_new_tensor_1d(ctx, GGML_TYPE_F32,   n_embd);

            // map by name
            model.tensors["gpt_neox.layers." + std::to_string(i) + ".input_layernorm.weight"] = layer.ln_1_g;
            model.tensors["gpt_neox.layers." + std::to_string(i) + ".input_layernorm.bias"]   = layer.ln_1_b;

            model.tensors["gpt_neox.layers." + std::to_string(i) + ".attention.query_key_value.weight"] = layer.c_attn_attn_w;
            model.tensors["gpt_neox.layers." + std::to_string(i) + ".attention.query_key_value.bias"]   = layer.c_attn_attn_b;

            model.tensors["gpt_neox.layers." + std::to_string(i) + ".attention.dense.weight"] = layer.c_attn_proj_w;
            model.tensors["gpt_neox.layers." + std::to_string(i) + ".attention.dense.bias"]   = layer.c_attn_proj_b;

            model.tensors["gpt_neox.layers." + std::to_string(i) + ".post_attention_layernorm.weight"] = layer.ln_2_g;
            model.tensors["gpt_neox.layers." + std::to_string(i) + ".post_attention_layernorm.bias"]   = layer.ln_2_b;

            model.tensors["gpt_neox.layers." + std::to_string(i) + ".mlp.dense_h_to_4h.weight"] = layer.c_mlp_fc_w;
            model.tensors["gpt_neox.layers." + std::to_string(i) + ".mlp.dense_h_to_4h.bias"]   = layer.c_mlp_fc_b;

            model.tensors["gpt_neox.layers." + std::to_string(i) + ".mlp.dense_4h_to_h.weight"] = layer.c_mlp_proj_w;
            model.tensors["gpt_neox.layers." + std::to_string(i) + ".mlp.dense_4h_to_h.bias"]   = layer.c_mlp_proj_b;
        }
    }

    // key + value memory
    {
        const auto & hparams = model.hparams;

        const int n_embd  = hparams.n_embd;
        const int n_layer = hparams.n_layer;
        const int n_ctx   = hparams.n_ctx;

        const int64_t n_mem      = n_layer*n_ctx;
        const int64_t n_elements = n_embd*n_mem;

        model.memory_k = ggml_new_tensor_1d(ctx, GGML_TYPE_F16, n_elements);
        model.memory_v = ggml_new_tensor_1d(ctx, GGML_TYPE_F16, n_elements);

        const size_t memory_size = ggml_nbytes(model.memory_k) + ggml_nbytes(model.memory_v);

        printf("%s: memory_size = %8.2f MB, n_mem = %" PRId64 "\n", __func__, memory_size/1024.0/1024.0, n_mem);
    }

    // load weights
    {
        int n_tensors = 0;
        size_t total_size = 0;

        printf("%s: ", __func__);

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
            if (ggml_nelements(tensor) != nelements) {
                fprintf(stderr, "%s: tensor '%s' has wrong size in model file\n", __func__, name.c_str());
                return false;
            }

            if (tensor->ne[0] != ne[0] || tensor->ne[1] != ne[1]) {
                fprintf(stderr, "%s: tensor '%s' has wrong shape in model file: got [%5d, %5d], expected [%5d, %5d]\n",
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

            fin.read(reinterpret_cast<char *>(tensor->data), ggml_nbytes(tensor));

            total_size += ggml_nbytes(tensor);
            if (++n_tensors % 8 == 0) {
                printf(".");
                fflush(stdout);
            }
        }

        printf(" done\n");

        printf("%s: model size = %8.2f MB / num tensors = %d\n", __func__, total_size/1024.0/1024.0, n_tensors);
    }

    fin.close();

    return true;
}


```

这段代码是一个 feed-forward 网络的前向计算图。这个前向计算图包含了输入层（inp）、GELU 激活函数、投影层（proj_w 和 proj_b）以及输出层（c_mlp_fc_w 和 c_mlp_fc_b）。

在函数中，首先将输入 inp 传递给第一个隐藏层（layer.ln_2_g），然后将该隐藏层的输出（cur）传递给第一个激活函数（ggml_mul_mat）。接下来，将传递给第一个激活函数的输出（cur）传递给第二个隐藏层（layer.ln_2_b），然后将第二个激活函数（ggml_mul_mat）的输出（cur）传递给第三个激活函数（ggml_schedule_ element(1, layer.c_mlp_proj_w)）。

接下来，将通过调用 ggml_schedule_element(2, layer.c_mlp_proj_b) 的输出（cur）传递给 GELU 激活函数（ggml_ gelu）。最后，通过 ggml_schedule_element(3, layer.c_mlp_fc_w) 得到输出层的最后一个元素（cur），这个元素将作为激活后的输出，并返回。


```cpp
// feed-forward network
ggml_tensor * gpt_neox_ff(
        const gpt_neox_layer & layer,
        ggml_context         * ctx0,
        ggml_tensor          * inp,
        float                  eps) {
    ggml_tensor * cur = ggml_norm(ctx0, inp, eps);

    cur = ggml_add(ctx0,
        ggml_mul(ctx0,
            ggml_repeat(ctx0, layer.ln_2_g, cur),
            cur),
        ggml_repeat(ctx0, layer.ln_2_b, cur));

    cur = ggml_mul_mat(ctx0,
            layer.c_mlp_fc_w,
            cur);

    cur = ggml_add(ctx0,
            ggml_repeat(ctx0, layer.c_mlp_fc_b, cur),
            cur);

    // GELU activation
    cur = ggml_gelu(ctx0, cur);

    // projection
    // cur = proj_w*cur + proj_b
    cur = ggml_mul_mat(ctx0,
            layer.c_mlp_proj_w,
            cur);

    cur = ggml_add(ctx0,
            ggml_repeat(ctx0, layer.c_mlp_proj_b, cur),
            cur);
    return cur;
}

```

This is a function definition for the intersection of two represented data: a vector of logits and a vector of embeddings. The function takes a context object (`ctx0`) and an input tensor (`inpL`) as input and returns an exit code of 0 if the computation was successful, or an exit code of 1 if there was an error.

The function performs the following operations:

1. It multiplies the input tensor by the context object, which is a matrix of logits.
2. It applies the logits to the input tensor using the `linear_function` of the `model` object.
3. It performs an softmax operation on the input tensor to obtain a probability distribution over the output vocabulary.
4. It runs the computation using the `run_with_核准` method of the `ggml_graph_compute_with_ctx` function.
5. If the computation was successful, it returns 0.

Note that the input data should already be passed through the model's `linear_function` before passing it to this function. The function assumes that the input data is continuous and人造， but this could be modified according to the specific requirements of the application.


```cpp
// evaluate the transformer
//
//   - model:     the model
//   - n_threads: number of threads to use
//   - n_past:    the context size so far
//   - embd_inp:  the embeddings of the tokens in the context
//   - embd_w:    the predicted logits for the next token
//
bool gpt_neox_eval(
        const gpt_neox_model & model,
        const int n_threads,
        const int n_past,
        const std::vector<gpt_vocab::id> & embd_inp,
              std::vector<float>         & embd_w,
              size_t                     & mem_per_token) {
    const int N = embd_inp.size();

    const auto & hparams = model.hparams;

    const int n_embd  = hparams.n_embd;
    const int n_layer = hparams.n_layer;
    const int n_ctx   = hparams.n_ctx;
    const int n_head  = hparams.n_head;
    const int n_vocab = hparams.n_vocab;
    const int n_rot   = hparams.n_rot;

    static size_t buf_size = 256u*1024*1024;
    static void * buf = malloc(buf_size);

    // use 2 scratch buffers
    // TODO: very hacky solution - reimplement in a more elegant way
    static size_t scr0_size = 256u*1024*1024;
    static void * scr0 = malloc(scr0_size);

    static size_t scr1_size = 256u*1024*1024;
    static void * scr1 = malloc(scr1_size);

    if (mem_per_token > 0 && mem_per_token*N > buf_size) {
        const size_t buf_size_new = 1.1*(mem_per_token*N); // add 10% to account for ggml object overhead
        //printf("\n%s: reallocating buffer from %zu to %zu bytes\n", __func__, buf_size, buf_size_new);

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

    // KQ_pos - contains the positions
    struct ggml_tensor * KQ_pos = ggml_new_tensor_1d(ctx0, GGML_TYPE_I32, N);
    int * data = (int *) KQ_pos->data;
    for (int i = 0; i < N; ++i) {
        data[i] = n_past + i;
    }

    struct ggml_tensor * embd = ggml_new_tensor_1d(ctx0, GGML_TYPE_I32, N);
    memcpy(embd->data, embd_inp.data(), N*ggml_element_size(embd));

    // wte
    struct ggml_tensor * inpL = ggml_get_rows(ctx0, model.wte, embd);

    for (int il = 0; il < n_layer; ++il) {
        struct ggml_tensor * cur;

        ggml_set_scratch(ctx0, { 0, scr0_size, scr0, });

        // self-attention
        {
            {
                cur = ggml_norm(ctx0, inpL, hparams.eps);

                cur = ggml_add(ctx0,
                        ggml_mul(ctx0,
                            ggml_repeat(ctx0, model.layers[il].ln_1_g, cur),
                            cur),
                        ggml_repeat(ctx0, model.layers[il].ln_1_b, cur));
            }

            // compute QKV
            {
                cur = ggml_mul_mat(ctx0,
                        model.layers[il].c_attn_attn_w,
                        cur);

                cur = ggml_add(ctx0,
                        ggml_repeat(ctx0, model.layers[il].c_attn_attn_b, cur),
                        cur);
            }

            struct ggml_tensor * Qcur = ggml_cont(ctx0, ggml_view_3d(ctx0, cur, n_embd/n_head, n_head, N, cur->nb[1]/n_head, cur->nb[1], 0*sizeof(float)*n_embd/n_head));
            struct ggml_tensor * Kcur = ggml_cont(ctx0, ggml_view_3d(ctx0, cur, n_embd/n_head, n_head, N, cur->nb[1]/n_head, cur->nb[1], 1*sizeof(float)*n_embd/n_head));
            struct ggml_tensor * Vcur = ggml_cont(ctx0, ggml_view_3d(ctx0, cur, n_embd/n_head, n_head, N, cur->nb[1]/n_head, cur->nb[1], 2*sizeof(float)*n_embd/n_head));

            // using mode = 2 for GPT-NeoX mode
            Qcur = ggml_rope_inplace(ctx0, Qcur, KQ_pos, n_rot, 2, 0);
            Kcur = ggml_rope_inplace(ctx0, Kcur, KQ_pos, n_rot, 2, 0);

            // store key and value to memory
            {
                Vcur = ggml_transpose(ctx0, ggml_reshape_2d(ctx0, Vcur, n_embd, N));

                struct ggml_tensor * k = ggml_view_1d(ctx0, model.memory_k, N*n_embd, (ggml_element_size(model.memory_k)*n_embd)*(il*n_ctx + n_past));
                struct ggml_tensor * v = ggml_view_2d(ctx0, model.memory_v, N, n_embd,
                        (   n_ctx)*ggml_element_size(model.memory_v),
                        (il*n_ctx)*ggml_element_size(model.memory_v)*n_embd + n_past*ggml_element_size(model.memory_v));

                ggml_build_forward_expand(gf, ggml_cpy(ctx0, Kcur, k));
                ggml_build_forward_expand(gf, ggml_cpy(ctx0, Vcur, v));
            }

            // Q = Qcur.contiguous().view(n_embd/n_head, n_head, N).permute(0, 2, 1, 3)
            struct ggml_tensor * Q =
                ggml_permute(ctx0,
                        Qcur,
                        0, 2, 1, 3);

            // K = Kmem.view(n_embd/n_head, n_head, n_past + N).permute(0, 2, 1, 3)
            struct ggml_tensor * K =
                ggml_permute(ctx0,
                        ggml_reshape_3d(ctx0,
                            ggml_view_1d(ctx0, model.memory_k, (n_past + N)*n_embd, il*n_ctx*ggml_element_size(model.memory_k)*n_embd),
                            n_embd/n_head, n_head, n_past + N),
                        0, 2, 1, 3);

            // K * Q
            struct ggml_tensor * KQ = ggml_mul_mat(ctx0, K, Q);

            // KQ_scaled = KQ / sqrt(n_embd/n_head)
            struct ggml_tensor * KQ_scaled =
                ggml_scale_inplace(ctx0,
                        KQ,
                        ggml_new_f32(ctx0, 1.0f/sqrt(float(n_embd)/n_head))
                        );

            // KQ_masked = mask_past(KQ_scaled)
            struct ggml_tensor * KQ_masked = ggml_diag_mask_inf_inplace(ctx0, KQ_scaled, n_past);

            // KQ = soft_max(KQ_masked)
            struct ggml_tensor * KQ_soft_max = ggml_soft_max_inplace(ctx0, KQ_masked);

            // V_trans = Vmem.view(n_embd/n_head, n_head, n_past + N).permute(1, 2, 0, 3).contiguous()
            struct ggml_tensor * V =
                ggml_view_3d(ctx0, model.memory_v,
                        n_past + N, n_embd/n_head, n_head,
                        n_ctx*ggml_element_size(model.memory_v),
                        n_ctx*ggml_element_size(model.memory_v)*n_embd/n_head,
                        il*n_ctx*ggml_element_size(model.memory_v)*n_embd);

            // KQV = transpose(V) * KQ_soft_max
            struct ggml_tensor * KQV = ggml_mul_mat(ctx0, V, KQ_soft_max);

            // KQV_merged = KQV.permute(0, 2, 1, 3)
            struct ggml_tensor * KQV_merged = ggml_permute(ctx0, KQV, 0, 2, 1, 3);

            // cur = KQV_merged.contiguous().view(n_embd, N)
            cur = ggml_cpy(ctx0,
                    KQV_merged,
                    ggml_new_tensor_2d(ctx0, GGML_TYPE_F32, n_embd, N));

            // projection
            {
                cur = ggml_mul_mat(ctx0,
                        model.layers[il].c_attn_proj_w,
                        cur);

                cur = ggml_add(ctx0, ggml_repeat(ctx0, model.layers[il].c_attn_proj_b, cur), cur);
            }
        }

        ggml_set_scratch(ctx0, { 0, scr1_size, scr1, });

        if (hparams.par_res == 0) {
            struct ggml_tensor * inpFF = ggml_add(ctx0, cur, inpL);

            cur = gpt_neox_ff(model.layers[il], ctx0, inpFF, hparams.eps);

            // input for next layer
            inpL = ggml_add(ctx0, cur, inpFF);
        } else {
            struct ggml_tensor * inpFF = cur;

            // this is independent of the self-attention result, so it could be done in parallel to the self-attention
            // note here we pass inpL instead of cur
            cur = gpt_neox_ff(model.layers[il], ctx0, inpL, hparams.eps);

            // layer input + FF
            cur  = ggml_add(ctx0, cur, inpFF);

            // input for next layer
            inpL = ggml_add(ctx0, cur, inpL);
        }
    }

    ggml_set_scratch(ctx0, { 0, scr0_size, scr0, });

    // norm
    {
        inpL = ggml_norm(ctx0, inpL, hparams.eps);

        // inpL = ln_f_g*inpL + ln_f_b
        inpL = ggml_add(ctx0,
                ggml_mul(ctx0,
                    ggml_repeat(ctx0, model.ln_f_g, inpL),
                    inpL),
                ggml_repeat(ctx0, model.ln_f_b, inpL));
    }

    ggml_set_scratch(ctx0, { 0, 0, nullptr, });

    // lm_head
    {
        inpL = ggml_mul_mat(ctx0, model.lmh_g, inpL);

        //inpL = ggml_add(ctx0,
        //        ggml_repeat(ctx0, model.lmh_b, inpL),
        //        inpL);
    }

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

    // return result for just the last token
    embd_w.resize(n_vocab);
    memcpy(embd_w.data(), (float *) ggml_get_data(inpL) + (n_vocab*(N-1)), sizeof(float)*n_vocab);

    if (mem_per_token == 0) {
        mem_per_token = ggml_used_mem(ctx0)/N;
    }
    //printf("used_mem = %zu\n", ggml_used_mem(ctx0));

    ggml_free(ctx0);

    return true;
}

```

This is a C++ implementation of the text-to-sequence (T2S) model training and evaluation code. The T2S model is trained on an unsupervised corpus of text data, represented as a list of integers (tokens) and a vocabulary, represented as a list of IDs (terms). The input to the model is a text in the form of a list of integers, each representing a token, and a list of IDs (terms) representing the vocabulary.

The T2S model takes as input a text in the form of a list of integers, each representing a token, and a list of IDs (terms) representing the vocabulary. The output is a list of integer sequences, each representing a predefined text in the vocabulary order.

The T2S model uses a parallel and distributed implementation to process the input text and generate the output text. The input text is processed by the different layers of the model, while the output text is generated by the predict function. The predict function uses a parallel implementation to efficiently generate the output text, taking into account the number of processes and the size of the output.

The T2S model also includes timing information to track the elapsed time for each operation. This can be useful for debugging and understanding the performance of the model.

Overall, the T2S model is a tool for generating text in a specific vocabulary, based on a list of integers. It can be used for a variety of tasks, such as machine translation, summarization, and more.


```cpp
int main(int argc, char ** argv) {
    ggml_time_init();

    const int64_t t_main_start_us = ggml_time_us();

    gpt_params params;
    params.model = "models/stablelm-base-alpha-3b/ggml-model-f16.bin";

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
    gpt_neox_model model;

    // load the model
    {
        const int64_t t_start_us = ggml_time_us();

        if (!gpt_neox_model_load(params.model, model, vocab)) {
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

    printf("%s: number of tokens in prompt = %zu\n", __func__, embd_inp.size());
    for (size_t i = 0; i < embd_inp.size(); i++) {
        printf("%s: token[%zu] = %6d, %s\n", __func__, i, embd_inp[i], vocab.id_to_token.at(embd_inp[i]).c_str());
    }
    printf("\n");

    std::vector<gpt_vocab::id> embd;

    // determine the required inference memory per token:
    size_t mem_per_token = 0;
    gpt_neox_eval(model, params.n_threads, 0, { 0, 1, 2, 3 }, logits, mem_per_token);

    for (size_t i = embd.size(); i < embd_inp.size() + params.n_predict; i++) {
        // predict
        if (embd.size() > 0) {
            const int64_t t_start_us = ggml_time_us();

            if (!gpt_neox_eval(model, params.n_threads, n_past, embd, logits, mem_per_token)) {
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
                if (int32_t(embd.size()) > params.n_batch) {
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
        if (embd.back() == 0) {
            break;
        }
    }

    // report timing
    {
        const int64_t t_main_end_us = ggml_time_us();

        printf("\n\n");
        printf("%s: mem per token = %8zu bytes\n", __func__, mem_per_token);
        printf("%s:     load time = %8.2f ms\n", __func__, t_load_us/1000.0f);
        printf("%s:   sample time = %8.2f ms\n", __func__, t_sample_us/1000.0f);
        printf("%s:  predict time = %8.2f ms / %.2f ms per token\n", __func__, t_predict_us/1000.0f, t_predict_us/1000.0f/n_past);
        printf("%s:    total time = %8.2f ms\n", __func__, (t_main_end_us - t_main_start_us)/1000.0f);
    }

    ggml_free(model.ctx);

    return 0;
}

```