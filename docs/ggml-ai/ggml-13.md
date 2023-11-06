# GGML源码解析 13

# `examples/python/ggml/__init__.py`

这段代码是一个Python库的函数，它通过Python与ggml库相结合，实现了ggml库的几个常用功能。通过调用这个函数，用户可以更方便地使用ggml库进行数据处理和绘图。现在让我们逐步分析这段代码：

1. 引入ggml库：
```cpp
 Python bindings for the ggml library.
```
这句话表明了这段代码引入了ggml库，以便用户可以在Python中使用该库。

1. 定义了使用ggml库的函数：
```cpp
 Usage example:

     from ggml import lib, ffi
     from ggml.utils import init, copy, numpy
     import numpy as np
```
这句话定义了一个使用ggml库的函数。它指导用户如何使用这段代码，以便在Python中进行数据处理和绘图。这段代码会指导用户创建一个ggml.CGraph对象，并使用lib.ggml\*函数进行各种ggml库操作。

1. 创建一个具有内存管理功能的上下文：
```cpp
 ctx = init(mem_size=10*1024*1024)
```
这个函数创建了一个上下文对象，具有10MB的内存管理功能。这个功能可以通过传递一个数字作为参数来实现，例如：`init(mem_size=20*1024*1024)`。

1. 创建两个ggml.Tensor对象：
```cpp
 a = lib.ggml_new_tensor_1d(ctx, lib.GGML_TYPE_Q5_K, n)
 b = lib.ggml_new_tensor_1d(ctx, lib.GGML_TYPE_F32, n)
```
这两个函数分别创建了两个ggml.Tensor对象，分别使用lib.ggml\*函数从内存中。第一个参数指定了要创建的 tensor 的类型（这里是 lib.GGML_TYPE_Q5_K，表示int 5 类型），第二个参数指定了要创建的 tensor 的维度，这里只有一个维度，表示从1开始。

1. 使用ggml.add函数执行ggml库的`+`操作：
```cpp
 sum = lib.ggml_add(ctx, a, b)
```
这个函数使用lib.ggml\*函数执行了ggml库的`+`操作，将两个tensor相加，并将结果存储在变量`sum`中。

1. 通过numpy将结果转换为NumPy数组：
```cpp
 gf = ffi.new('struct ggml_cgraph*')
 lib.ggml_
```


```cpp
"""
  Python bindings for the ggml library.

  Usage example:

      from ggml import lib, ffi
      from ggml.utils import init, copy, numpy
      import numpy as np

      ctx = init(mem_size=10*1024*1024)
      n = 1024
      n_threads = 4

      a = lib.ggml_new_tensor_1d(ctx, lib.GGML_TYPE_Q5_K, n)
      b = lib.ggml_new_tensor_1d(ctx, lib.GGML_TYPE_F32, n)
      sum = lib.ggml_add(ctx, a, b)

      gf = ffi.new('struct ggml_cgraph*')
      lib.ggml_build_forward_expand(gf, sum)

      copy(np.array([i for i in range(n)], np.float32), a)
      copy(np.array([i*100 for i in range(n)], np.float32), b)
      lib.ggml_graph_compute_with_ctx(ctx, gf, n_threads)

      print(numpy(sum, allow_copy=True))

  See https://cffi.readthedocs.io/en/latest/cdef.html for more on cffi.
```

该代码是一个Python脚本，用于从不同的库中查找ggml库。它使用try-except语句来捕获ImportError异常，如果没有找到相关的库，则会输出错误信息并请求用户重新运行脚本。

具体来说，该脚本首先尝试从环境中查找名为“ggml_library”的库。如果找到了该库，则从该库中查找名为“__candidates”的列表。如果环境中没有找到该库，则会尝试从系统询问中查找名为“ggml_shared.dll”和“llama.dll”的库。如果操作系统为Windows，则会尝试使用“libggml_shared.dll”和“libllama.dylib”库。

如果在任何地方找到了名为“__candidates”的列表，则会从该列表中查找名为“libggml_shared.so”和“libllama.so”的库。如果操作系统为“Darwin”，则会尝试使用“libggml_shared.dylib”和“libllama.dylib”的库。

如果任何地方也没有找到合适的库，则会抛出ImportError异常，并请求用户重新运行脚本。


```cpp
"""

try:
    from ggml.cffi import ffi as ffi
except ImportError as e:
    raise ImportError(f"Couldn't find ggml bindings ({e}). Run `python regenerate.py` or check your PYTHONPATH.")

import os, platform

__exact_library = os.environ.get("GGML_LIBRARY")
if __exact_library:
    __candidates = [__exact_library]
elif platform.system() == "Windows":
    __candidates = ["ggml_shared.dll", "llama.dll"]
else:
    __candidates = ["libggml_shared.so", "libllama.so"]
    if platform.system() == "Darwin":
        __candidates += ["libggml_shared.dylib", "libllama.dylib"]

```

这段代码是一个Python脚本，它定义了一个for循环，用于遍历一个名为__candidates的列表。在循环内部，它定义了一个函数lib，并尝试通过ffi库（ffi.dlopen）加载该函数库。如果该尝试失败（比如因为库不存在或访问权限不足），则引发一个OSError并跳过当前循环迭代。

该函数库可能是一个C++或Python库的共享库，该库被定义为ffi库的函数或类。在这里，根据给定的库名称，ffi库将尝试加载该库，并定义了几个常用的ffi帮助函数，如new、cast、string等。这个脚本的作用是使用这些ffi帮助函数，从而定义了一个新的库，这个库可以用来开发新的应用程序或者修改现有的应用程序。


```cpp
for i, name in enumerate(__candidates):
    try:
        # This is where all the functions, enums and constants are defined
        lib = ffi.dlopen(name)
    except OSError:
        if i < len(__candidates) - 1:
            continue
        raise OSError(f"Couldn't find ggml's shared library (tried names: {__candidates}). Add its directory to DYLD_LIBRARY_PATH (on Mac) or LD_LIBRARY_PATH, or define GGML_LIBRARY.")

# This contains the cffi helpers such as new, cast, string, etc.
# https://cffi.readthedocs.io/en/latest/ref.html#ffi-interface
ffi = ffi

```

# `examples/replit/convert-h5-to-ggml.py`

这段代码的作用是从给定的模型文件（h5）中读取文本数据，并将其转换为ggml格式的数据。它主要实现了以下几个功能：

1. 读取给定的模型文件（使用dir-model参数）。
2. 读取输入文件的文本数据（使用sys.argv[2]参数）。
3. 将文本数据转换为model.proto格式的数据。
4. 将数据存储为ggml格式的数据，以便使用sentencepiece库进行进一步的处理。


```cpp
from pathlib import Path
import sys
import struct
import json
import numpy as np
from transformers import AutoModelForCausalLM, AutoTokenizer
import sentencepiece.sentencepiece_model_pb2 as model

if len(sys.argv) < 3:
    print("Usage: convert-h5-to-ggml.py dir-model [use-f32]\n")
    print("  ftype == 0 -> float32")
    print("  ftype == 1 -> float16")
    sys.exit(1)


```

这段代码的作用是读取一个名为"config.json"的JSON文件，并将其中的配置参数解析为Python模型中的参数。然后，根据传入的文件名"output.txt"，将训练得到的模型保存为ggml格式，并将其保存到指定的目录中。

具体来说，代码首先读取传入目录下的"config.json"文件，并将其中的配置参数解析为模型参数的列表。接着，我们从传入文件中读取sp模型文件，并将其解码为sp协议，以便在训练其他模型时使用。然后，根据传入的文件名，将sp模型文件保存为ggml格式，并将其保存到指定的目录中。


```cpp
# output in the same directory as the model
dir_model = sys.argv[1]
fname_out = sys.argv[1] + "/ggml-model.bin"


with open(dir_model + "/config.json", "r", encoding="utf-8") as f:
    hparams = json.load(f)

sp_proto = model.ModelProto()
sp_proto.ParseFromString(open(Path(sys.argv[1]) / "spiece.model", "rb").read())


# possible data types
#   ftype == 0 -> float32
#   ftype == 1 -> float16
```

这段代码的主要作用是定义了一个类型映射对象 `ftype_str`，其中包含 ftype 0 和 ftype 1 两种数据类型及其对应的字符串名称。接着读取两个命令行参数，第二个参数是 ftype 的值，如果 ftype 的值不小于 0 且不小于 1，则执行下一步操作。

具体来说，代码会根据 ftype 的值来选择对应的字符串，并将其输出。然后，代码会定义一个文件名 `fname_out`，将该文件名和 ftype 名称以及 `.bin` 后缀组成，用于输出训练好的模型文件。接着，代码会从预训练模型模型文件目录中读取模型文件，并使用预训练模型来执行推理。最后，代码会使用自定义的 `AutoTokenizer` 和 `AutoModelForCausalLM` 类，从预训练模型中获取数据，并根据 ftype 的值生成相应的输出文件。


```cpp
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


tokenizer = AutoTokenizer.from_pretrained(dir_model, trust_remote_code=True)
model = AutoModelForCausalLM.from_pretrained(
    dir_model, low_cpu_mem_usage=True, trust_remote_code=True
)
```

这段代码的作用是输出一个名为 "model" 的模型的所有变量的名称、形状和数据类型信息，然后将模型的参数保存到一个文件中。

具体来说，代码首先定义了一个名为 "print(model)" 的函数，这个函数的作用是打印模型的参数。接着定义了一个名为 "print(tokenizer.encode('I believe the meaning of life is'))" 的函数，这个函数的作用是打印一个字符串 "I believe the meaning of life is"，然后使用模型的 tokenizer 函数进行编码，得到一个 NumPy 数组。

接下来定义了一个名为 "list_vars" 的变量，这个变量记录了模型在 "state_dict()" 方法中所有的变量名、形状和数据类型信息。然后使用 for 循环遍历 list_vars 中的所有变量名，依次打印出每个变量的名称、形状和数据类型信息。

接着定义了一个名为 "fout" 的变量，这个变量是一个 file-like 对象，用于写入一个包含模型的参数的整数。然后使用 struct.pack 函数将一个整数类型的参数 "0x67676D6C" 写入到 fout 中。

最后，将模型的参数保存到 fout 对象中。


```cpp
# print (model)

# print(tokenizer.encode('I believe the meaning of life is'))

list_vars = model.state_dict()
for name in list_vars.keys():
    print(name, list_vars[name].shape, list_vars[name].dtype)

fout = open(fname_out, "wb")

print(hparams)

fout.write(struct.pack("i", 0x67676D6C))  # magic: ggml in hex
fout.write(struct.pack("i", hparams["d_model"]))
fout.write(struct.pack("i", hparams["max_seq_len"]))
```

这段代码的主要作用是读取一个名为 `sp_proto` 的结构体，将其中的三个整型字段 `n_heads`、`n_layers` 和 `vocab_size` 打包成字节序列并写入到文件 `fout` 中。

具体来说，代码中首先调用 `struct.pack` 函数将 `n_heads`、`n_layers` 和 `vocab_size` 字段中的值转换成字节序列，然后分别写入到文件中。其中，`fout.write(struct.pack("i", hparams["n_heads"]))` 将 `n_heads` 字段的值写入到文件中，`fout.write(struct.pack("i", hparams["n_layers"]))` 将 `n_layers` 字段的值写入到文件中，`fout.write(struct.pack("i", hparams["vocab_size"]))` 将 `vocab_size` 字段的值写入到文件中。

接着，代码中处理了一个特殊的情况，即 `vocab_size` 字段的长度大于了 `sp_proto.pieces` 结构体中所有子字段的数量。在这种情况下，代码会将 `sp_proto.pieces` 中的所有子字段的长度打包成一个字节序列并写入到文件中，同时将 `vocab_size` 的值设为 `0`，以避免在文件中出现多余的数据。


```cpp
fout.write(struct.pack("i", hparams["n_heads"]))
fout.write(struct.pack("i", hparams["n_layers"]))
fout.write(struct.pack("i", hparams["vocab_size"]))
fout.write(struct.pack("i", ftype))


# TODO: temporary hack to not deal with implementing the tokenizer
for piece in sp_proto.pieces:
    encoded_piece = piece.piece.encode("utf-8")
    fout.write(struct.pack("i", len(encoded_piece)))
    fout.write(encoded_piece)
    fout.write(struct.pack("f", piece.score))

if hparams["vocab_size"] > len(sp_proto.pieces):
    for i in range(hparams["vocab_size"] - len(sp_proto.pieces)):
        fout.write(struct.pack("i", 0))
        fout.write(struct.pack("f", 0))

```

这段代码的作用是处理一个列表 `list_vars` 中的所有变量，并输出每个变量的形状以及该变量的值。

具体来说，代码首先遍历 `list_vars` 中的每个键，并将每个键的值存储在变量 `data` 中。然后，代码根据变量 `data` 中的数据类型和形状，将数据转换成所需的类型(例如从 float32 转换为 float16，或从 float16 转换为 float32)。如果变量 `data` 的数据类型和形状不符合要求，则代码会将其转换为所需的类型。

接着，代码会将变量 `data` 写入文件 `fout`，并使用结构体 `struct` 将变量 `name`、变量形状 `n_dims` 和数据类型 `ftype` 打包到一个字符串中。在文件 `fout` 中，每行包含一个记录，其中第一个记录包含变量 `name`、第二个记录包含变量 `n_dims`、第三个记录包含变量 `ftype` 和一个空字符串。

最后，代码会将变量 `data` 写入文件 `fout` 中，其中每个值都被写成一个记录。


```cpp
for name in list_vars.keys():
    data = list_vars[name].squeeze().numpy()
    print("Processing variable: " + name + " with shape: ", data.shape)

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
    str = name.encode("utf-8")
    fout.write(struct.pack("iii", n_dims, len(str), ftype_cur))
    for i in range(n_dims):
        fout.write(struct.pack("i", data.shape[n_dims - 1 - i]))
    fout.write(str)

    # data
    data.tofile(fout)

```

这段代码的作用是关闭名为 "fout" 的文件，并输出文件名 "fname_out"。具体来说，第一行使用 `fout.close()` 方法关闭文件输出流(通常是标准输出流)，这样可以防止程序在运行时误读或输出错误信息。第二行使用 `print()` 函数输出一个空行，不会对文件产生任何影响。第三行再次使用 `print()` 函数输出一个空行，同样不会对文件产生任何影响。


```cpp
fout.close()

print("Done. Output file: " + fname_out)
print("")

```

# `examples/replit/main.cpp`

这段代码是一个通用的C++头文件，它包括了ggml和common-ggml两个头文件。ggml是一个C++的图形渲染引擎，而common-ggml则是ggml的一个通用类。通过包含这两个头文件，我们可以使用ggml的各种函数和类来创建和操作图形。

进一步分析，我们可以看到该代码定义了一些与标准C++库头文件包含的函数和类。这些函数和类包括：

```cpp
#include <cassert>
#include <cmath>
#include <cinttypes>
#include <cstddef>
#include <cstdio>
#include <cstring>
#include <fstream>
#include <iostream>
#include <map>
#include <stdint.h>
```

这些函数和类是C++标准库中的基本数据类型，如整型、浮点型、复数等，以及输入输出流头文件，如iostream、std::map、std::vector等。这些头文件提供了C++程序在运行时需要的基本类型和操作。

而通用的ggml和common-ggml头文件，则提供了更高级的ggml功能，如图形渲染引擎的函数和类。ggml头文件中定义了一系列ggml自己的数据结构和函数，而common-ggml头文件中则提供了一些通用的数据结构和函数，如平面图形的的基本操作。

通过包含这两个头文件，我们可以使用ggml的各种函数和类来创建和操作图形，同时又可以利用common-ggml提供的通用的数据结构和函数来简化ggml程序的编写。


```cpp
#include "ggml/ggml.h"

#include "common-ggml.h"
#include "common.h"

#include <cassert>
#include <cmath>
#include <cinttypes>
#include <cstddef>
#include <cstdio>
#include <cstring>
#include <fstream>
#include <iostream>
#include <map>
#include <stdint.h>
```

这段代码的作用是定义了一个名为“test_map”的哈希表，用于存储一些字符串，以供测试用。

首先，引入了<string>头文件，它是C++标准库中的一个输入/输出字符串头文件，定义了一些字符串操作的基本模板。

然后，引入了<unordered_map>头文件，它也是C++标准库中的一个输入/输出字符串头文件，定义了一些哈希表操作的基本模板。

接着，引入了<utility>头文件，它也是C++标准库中的一个输入/输出字符串头文件，定义了一些辅助输出和输入的基本模板。

再然后，引入了<vector>头文件，它同样是C++标准库中的一个输入/输出字符串头文件，定义了一些vector操作的基本模板。

接下来，定义了一个名为“is_stdin_terminal”的函数，它使用了<unistd.h>头文件和< Windows.h>头文件，用于判断当前运行的环境是否为标准输入终端。

最后，对代码进行了注释，指出使用了哪些标准库头文件以及定义了一些什么函数。


```cpp
#include <string>
#include <unordered_map>
#include <utility>
#include <vector>

#if defined(_WIN32)
#define NOMINMAX
#include <Windows.h>
bool is_stdin_terminal() {
    auto in = GetStdHandle(STD_INPUT_HANDLE);
    return GetFileType(in) == FILE_TYPE_CHAR;
}
#else
#include <unistd.h>
bool is_stdin_terminal() {
    return isatty(STDIN_FILENO);
}
```

这段代码定义了一个名为`replit_tokenizer`的结构体，它包含一个`gpt_vocab`类型的词汇表`raw_vocab`，一个`piece_map_t`类型的词表`piece_map`，以及一个包含所有已知词汇的`std::vector<std::string>`类型的词汇表`vocab`。

首先，它通过`#ifdef _MSC_VER`的条件判断是否支持`__lambda_ Chebyshev近似可逆矩阵的逆`，如果是，它将忽略掉该条件的`#pragma`指令，即`#pragma warning(disable: 4244 4267)`，这会使得编译器在编译时给出警告，提示条件判断中的某些代码可能存在潜在的不可靠数据结构等问题，但不会影响程序的正确性。

接着，它定义了一个`replit_tokenizer`的结构体，其中的`raw_vocab`和`piece_map`成员变量直接使用了定义在文件头文件`包含_MSC_VER`的`std::pair`和`std::unordered_map`类型的成员。

最后，它定义了一个包含一些预定义词汇的`std::vector`类型的成员`vocab`，这个成员没有定义类型，但可以推断出它是一个`std::vector<std::string>`类型的成员。


```cpp
#endif

#if defined(_MSC_VER)
#pragma warning(disable: 4244 4267) // possible loss of data
#endif

using piece_t = std::pair<std::size_t, float>;
using piece_map_t = std::unordered_map<std::string, piece_t>;

struct replit_tokenizer {
    gpt_vocab raw_vocab;
    piece_map_t piece_map;
    std::vector<std::string> vocab;
};

```

This appears to be a function that finds the best segmentation of a given word based on a language model, such as the pre-training and fine-tuning of a neural network. The function takes a segmentation score array (`best_segmentations_scores`), which is passed through the function and includes the start index of each segmentation as well as the segmentation score. It also takes a word object (`word`) and an optional segmentation score at the start of the segmentation (`best_score_at_start`).

The function starts by initializing the segmentation score array with the first element, `best_segmentations_scores[0] = 0` and the start index of the first segmentation at `best_segmentations_starts[0] = 0`.

Then it loops through the segmentation score array and the word object, and uses the `model` object to look up the word in the dictionary, with the segmentation score as the lookup key. If the word is found and the segmentation score is not negative, it adds the word to the `tokens` vector and assigns the segmentation score to the `score` variable.

If the word is found but the segmentation score is negative or the segmentation score is negative and the lookup key is not found in the `model`, it sets the `best_segmentations_scores[i]` to `-std::numeric_limits<float>::infinity()` and sets the `best_segmentations_starts[i]` to `best_segmentations_starts[i-1]` (i.e., the previous segmentation start index).

The function then returns the segmentation tokens (i.e., the words in the `tokens` vector), the segmentation score, and the start index of the first segmentation if the segmentation was successful.

It is important to note that this function assumes that the segmentation scores are valid and are being passed through the `model` in a reliable manner, and that the `model` is well-suited to the task of segmentation. Additionally, the function does not handle the case where the segmentation score is negative or the segmentation score is negative and the lookup key is not found in the `model`.


```cpp
std::pair<std::vector<std::size_t>, float> encode_word(const std::string & word, const piece_map_t & model) {
    std::vector<int> best_segmentations_starts(word.length() + 1, -1);
    best_segmentations_starts[0] = 0;

    std::vector<float> best_segmentations_scores(word.length() + 1, -std::numeric_limits<float>::infinity());
    best_segmentations_scores[0] = 1.0;

    for (size_t start_idx = 0; start_idx < word.length(); ++start_idx) {
        float best_score_at_start = best_segmentations_scores[start_idx];
        for (size_t end_idx = start_idx + 1; end_idx <= word.length(); ++end_idx) {
            std::string token = word.substr(start_idx, end_idx - start_idx);
            if (model.count(token) && best_score_at_start != -std::numeric_limits<float>::infinity()) {
                float token_score = model.at(token).second;
                float score = token_score + best_score_at_start;
                if (best_segmentations_scores[end_idx] == -std::numeric_limits<float>::infinity() ||
                    best_segmentations_scores[end_idx] > score) {
                    best_segmentations_starts[end_idx] = start_idx;
                    best_segmentations_scores[end_idx] = score;
                }
            }
        }
    }

    if (best_segmentations_scores.back() == -std::numeric_limits<float>::infinity()) {
        return std::make_pair(std::vector<std::size_t>{0}, 0.0f);
    }

    float score = best_segmentations_scores.back();
    int start = best_segmentations_starts.back();
    int end = word.length();
    std::vector<std::size_t> tokens;
    while (start != 0) {
        const auto token_id = model.at(word.substr(start, end - start)).first;
        tokens.insert(tokens.begin(), token_id);
        int next_start = best_segmentations_starts[start];
        end = start;
        start = next_start;
    }
    const auto token_id = model.at(word.substr(start, end - start)).first;
    tokens.insert(tokens.begin(), token_id);
    return std::make_pair(tokens, score);
}

```

这段代码是一个名为 `replit_tokenizer_load` 的函数，它是 Replit Tokenizer 的一个加载函数。

该函数的作用是读取一个istream中的内容，并将其解析为replit_tokenizer类型的对象。函数需要读取的最大词汇量（max_vocab_size）是一个整数，表示replit_tokenizer最多可以支持多少个词汇。

具体实现中，首先定义了一个变量 `word`，用于存储当前正在解析的词汇。然后定义了一个大小为128的 `buf` 数组，用于存储replit_tokenizer读取到的内容。接着，使用 for 循环遍历replit_tokenizer需要的最大词汇量，每次读取一个长度为 `len` 的内容，并将其存储到 `buf` 中。然后读取一个浮点数 `score`，并将其存储到 `tokenizer.piece_map` 中的对应词汇的值中。最后，将 `word` 存储到 `replit_tokenizer.raw_vocab.id_to_token` 映射中。

如果replit_tokenizer可以正常工作，函数返回一个布尔值（true/false），表示成功加载并解析了replit_tokenizer。


```cpp
bool replit_tokenizer_load(replit_tokenizer & tokenizer, std::istream & fin, int max_vocab_size) {
    std::string word;
    std::vector<char> buf(128);

    for (int i = 0; i < max_vocab_size; i++) {
        uint32_t len;
        fin.read((char *)&len, sizeof(len));

        buf.resize(len);
        fin.read((char *)buf.data(), len);
        word.assign(buf.data(), len);

        float score;
        fin.read((char *)&score, sizeof(score));

        tokenizer.piece_map[word] = std::make_pair(i, -score);
        tokenizer.raw_vocab.id_to_token[i] = word;
    }

    return true;
}

```

这段代码定义了一个名为 `replace_all` 的函数，用于在给定的字符串 `str` 中查找并替换字符 `find` 和 `replace`。

函数的参数为三个字符串变量 `str`、`find` 和 `replace`，分别用于查找要替换的字符串，以及替换后的字符串。函数返回一个新的字符串变量 `result`，其中包含从 `find` 开始到 `replace` 在内的所有字符，以及 `replace` 之后的所有字符。

函数的实现采用以下步骤：

1. 定义一个名为 `result` 的字符串变量，用于存储替换后的字符串。

2. 定义一个名为 `find_len` 的整数变量，用于存储 `find` 字符串中从 `find` 开始到下一个查找位置之间的字符数量。

3. 定义一个名为 `pos` 的整数变量，用于存储 `str` 字符串中从 `find` 开始到当前查找位置之间的字符数量。

4. 定义一个名为 `from` 的整数变量，用于存储 `find` 字符串中从 `find` 开始的位置。

5. 进入循环结构，从 `pos` 开始遍历 `str` 字符串，直到字符 `find` 出现的位置。

6. 在循环内部，将 `result` 字符串中的从 `from` 到 `pos - from` 之间的字符复制到 `result`，并将 `from` 向后移动 `from`。

7. 将 `pos` 和 `find_len` 更新为从 `find` 开始的位置和字符数量。

8. 在循环结束后，将 `result` 字符串中的从 `find` 开始到 `string::npos` 字符串末尾的所有字符复制到 `result`，并将 `string::npos` 字符串从 `result` 的开始位置开始。

9. 返回 `result` 字符串变量。


```cpp
std::string replace_all(const std::string & str,    // where to work
                        const std::string & find,   // substitute 'find'
                        const std::string & replace //      by 'replace'
) {
    using namespace std;
    string result;
    size_t find_len = find.size();
    size_t pos, from = 0;
    while (string::npos != (pos = str.find(find, from))) {
        result.append(str, from, pos - from);
        result.append(replace);
        from = pos + find_len;
    }
    result.append(str, from, string::npos);
    return result;
}

```

这段代码定义了两个函数 `replit_tokenizer_tokenize` 和 `replit_tokenizer_detokenize`。这两个函数都在一个 `replit_tokenizer` 类中，用于处理文本中的词汇，将文本中的符号和词汇分离，并将符号存储在一个 `std::vector<std::size_t>` 中，将词汇存储在一个 `std::vector<std::size_t>` 中。

具体来说，这两个函数的实现步骤如下：

1. 在 `replit_tokenizer_tokenize` 函数中，首先将给定的文本中的所有符号用给定的符号替换，得到一个 normalized_text。然后，使用给定的 `encode_word` 函数，将 normalized_text 中的每个词汇编码成一个整数，并将编码结果存储在一个 `std::vector<std::size_t>` 中。最终，函数返回这个 `std::vector<std::size_t>`。

2. 在 `replit_tokenizer_detokenize` 函数中，首先将给定的文本中的所有符号用给定的符号替换，得到一个 denormalized_text。然后，从 denormalized_text 中删除符号，得到一个只包含符号和空格的字符串。最后，将这个字符串编码成一个整数，并将编码结果存储在给定的 `std::vector<std::size_t>` 中。

这两个函数都是 `replit_tokenizer` 类的非平凡成员函数，可以在 `replit_tokenizer` 对象上使用。


```cpp
std::string ws_symbol = "\342\226\201";
std::vector<std::size_t> replit_tokenizer_tokenize(replit_tokenizer & tokenizer, const std::string & text) {
    std::vector<std::size_t> tokens;
    auto normalized_text = replace_all(text, " ", ws_symbol);
    auto tokenized = encode_word(normalized_text, tokenizer.piece_map);

    return tokenized.first;
}

std::string replit_tokenizer_detokenize(replit_tokenizer & tokenizer, const std::vector<std::size_t> & tokens) {
    std::string text;
    for (auto token : tokens) {
        text += tokenizer.raw_vocab.id_to_token[token];
    }
    auto denormalized_text = replace_all(text, ws_symbol, " ");
    return denormalized_text;
}

```



这段代码定义了一个名为`replit_hparams`的结构体，用于表示RePLIT模型的参数设置。

它定义了以下参数：

- `d_model`：当前模型的大小，即模型的编码器和解码器通道数。
- `max_seq_len`：当前模型的最大输入序列长度。
- `n_heads`：当前模型的头数。
- `n_layers`：当前模型的层数。
- `n_vocab`：当前模型使用的词汇数。
- `fty`：当前模型的类型，可以是0表示原始文本，1表示序列到序列模型。

此外，还定义了一个名为`replit_layer`的结构体，用于表示每个RePLIT子模块的参数。

它定义了以下参数：

- `norm_1_weight`：对输入序列进行预处理的最外层权重。
- `c_attn_wqkv_weight`：注意力头对于每个位置的权重，用于计算注意力。
- `c_attn_out_proj_weight`：注意力头从注意力中获得的输出权重，用于计算注意力。
- `norm_2_weight`:RePLIT模型的第二个预处理层权重。
- `ffn_up_proj`:RePLIT模型的升维预处理权重，用于计算输入的范数。
- `ffn_down_proj`:RePLIT模型的降维预处理权重，用于计算输入的平方根范数。

此代码提供了一个可以用来定义RePLIT模型参数的结构体，用于在训练和推理时设置RePLIT模型的不同参数。


```cpp
// no defaults for now
struct replit_hparams {
    int32_t d_model = 0;
    int32_t max_seq_len = 0;
    int32_t n_heads = 0;
    int32_t n_layers = 0;
    int32_t n_vocab = 0;
    int32_t ftype = 0;
};

struct replit_layer {
    // pre normalization
    struct ggml_tensor * norm_1_weight;

    // attention
    struct ggml_tensor * c_attn_wqkv_weight;
    struct ggml_tensor * c_attn_out_proj_weight;

    // post normalization
    struct ggml_tensor * norm_2_weight;

    // ff
    struct ggml_tensor * ffn_up_proj;
    struct ggml_tensor * ffn_down_proj;
};

```

这是一个 C++ 的结构体，表示一个 Replit 模型。该模型包含以下部分：

1. `replit_hparams`：这是模型设置，包括一些参数，如词汇表大小、布林种子等。
2. `layers`：这是一个容器，用于存储模型层。每个层都是一个 `replit_layer` 结构体，包含层类型、参数等。
3. `memory_k` 和 `memory_v`：这两个变量存储的是一个 `ggml_tensor`，类型为 `struct ggml_tensor *`。这是一个指针，指向一个 `ggml_tensor`。
4. `ctx`：这是模型所使用 的 `ggml_context` 的指针。
5. `tensors`：这是一个 `std::map`，用于存储模型中所有的张量。

Replit 模型是一个基于 Replit 的自然语言处理模型，可以用于生成文本、回答问题等任务。该模型可以分为几个部分，如词嵌入、位置编码、语言模型等。通过 `layers` 容器中的 `replit_layer`，可以组合出不同的模型，如预训练的语言模型、微调的模型等。


```cpp
struct replit_model {
    replit_hparams hparams;

    struct ggml_tensor * wte_weight;    // position embedding
    struct ggml_tensor * norm_f_weight; // language model head

    std::vector<replit_layer> layers;

    // key + value memory
    struct ggml_tensor * memory_k;
    struct ggml_tensor * memory_v;

    struct ggml_context * ctx;
    std::map<std::string, struct ggml_tensor *> tensors;
};

```

This is a C function that parses a model file (ggml file) that defines a tensor. It reads the tensor data from the file and checks it against the model definition.

Here's the function signature:
```cppc
int ggml_parse_tensor(ggml_t *model, ggml_t *tensor, int n_tensors, const void *data, int len);
```
This function takes a pointer to a `ggml_t` object, one `ggml_t` tensor, and the number of tensors in the model. It reads the data from the tensor and checks it against the model definition. The tensor is passed in as a pointer to `data` and is read using the `ggml_read_tensor` function.

The function returns `true` if the tensor is valid and `false` if it is not.

Here's the function implementation:
```cppc
int ggml_parse_tensor(ggml_t *model, ggml_t *tensor, int n_tensors, const void *data, int len) {
   int ret = ggml_parse_tensor_with_flexibility(model, tensor, data, len);
   if (!ret) return ret;

   if (ggml_type(tensor->type) == ggml_type_null) return 0;

   int total_size = 0;
   int bpe = ggml_type_size(ggml_type(tensor->type));
   int ne[4];
   int has_neg = 0;
   int i;

   for (i = 0; i < n_tensors; i++) {
       int start = i * bpe;
       int end = start + bpe;
       if (end < len) {
           int64_t value = (int64_t)data + i * end - i * bpe;
           if (has_neg) {
               value = -value;
           }
           ne[i] = value;
           total_size += bpe;
           has_neg = 1;
       }
   }

   if (bpe % ggml_blck_size(tensor->type) != 0) {
       fprintf(stderr,
               "%s: tensor '%s' has wrong size in model file: got %zu, expected %zu\n",
                   __func__, name.c_str(), ne[0], ne[1]
                               );
       return 1;
   }

   if (total_size / 1024.0 / 1024.0 != n_tensors) {
       fprintf(stderr,
               "%s: tensor '%s' has wrong size in model file: got %zu, expected %zu\n",
                   __func__, name.c_str(), total_size / 1024.0 / 1024.0, n_tensors);
       return 1;
   }

   if (!ggml_check_memory(tensor)) {
       fprintf(stderr,
               "%s: tensor '%s' has wrong memory shape in model file\n",
                   __func__, name.c_str(), tensor->data, tensor->data);
       return 1;
   }

   return 0;
}
```
This implementation reads the tensor data from the `data` pointer and checks it against the model definition. It uses the `ggml_read_tensor` function to read the tensor data from the file. The `ggml_type_null` check is added to ensure that the function can handle cases where the tensor data is not present.


```cpp
// load the model's weights from a file
bool replit_model_load(const std::string & fname, replit_model & model, replit_tokenizer & vocab) {
    printf("%s: loading model from '%s' - please wait ...\n", __func__, fname.c_str());

    auto fin = std::ifstream(fname, std::ios::binary);
    if (!fin) {
        fprintf(stderr, "%s: failed to open '%s'\n", __func__, fname.c_str());
        return false;
    }

    // verify magic
    {
        uint32_t magic;
        fin.read((char *)&magic, sizeof(magic));
        if (magic != GGML_FILE_MAGIC) {
            fprintf(stderr, "%s: invalid model file '%s' (bad magic)\n", __func__, fname.c_str());
            return false;
        }
    }

    // load hparams
    {
        auto & hparams = model.hparams;

        fin.read((char *)&hparams.d_model, sizeof(hparams.d_model));
        fin.read((char *)&hparams.max_seq_len, sizeof(hparams.max_seq_len));
        fin.read((char *)&hparams.n_heads, sizeof(hparams.n_heads));
        fin.read((char *)&hparams.n_layers, sizeof(hparams.n_layers));
        fin.read((char *)&hparams.n_vocab, sizeof(hparams.n_vocab));
        fin.read((char *)&hparams.ftype, sizeof(hparams.ftype));

        const int32_t qntvr = hparams.ftype / GGML_QNT_VERSION_FACTOR;

        printf("%s: d_model      = %d\n", __func__, hparams.d_model);
        printf("%s: max_seq_len  = %d\n", __func__, hparams.max_seq_len);
        printf("%s: n_heads      = %d\n", __func__, hparams.n_heads);
        printf("%s: n_layers     = %d\n", __func__, hparams.n_layers);
        printf("%s: n_vocab      = %d\n", __func__, hparams.n_vocab);
        printf("%s: ftype        = %d\n", __func__, hparams.ftype);
        printf("%s: qntvr        = %d\n", __func__, qntvr);

        hparams.ftype %= GGML_QNT_VERSION_FACTOR;
    }

    // load vocab
    replit_tokenizer_load(vocab, fin, model.hparams.n_vocab);

    // for the big tensors, we have the option to store the data in 16-bit
    // floats or quantized in order to save memory and also to speed up the
    // computation
    ggml_type wtype = ggml_ftype_to_ggml_type((ggml_ftype)(model.hparams.ftype));
    if (wtype == GGML_TYPE_COUNT) {
        fprintf(stderr, "%s: invalid model file '%s' (bad ftype value %d)\n", __func__, fname.c_str(),
                model.hparams.ftype);
        return false;
    }

    auto & ctx = model.ctx;

    size_t ctx_size = 0;

    {
        const auto & hparams = model.hparams;

        const int n_embd = hparams.d_model;
        const int n_layer = hparams.n_layers;
        const int n_ctx = hparams.max_seq_len;
        const int n_vocab = hparams.n_vocab;

        ctx_size += n_embd * n_vocab * ggml_type_sizef(wtype); // wte_weight
        ctx_size += n_embd * ggml_type_sizef(GGML_TYPE_F32);   // ln_f_weight

        ctx_size += n_layer * (n_embd * ggml_type_sizef(GGML_TYPE_F32));      // ln_1_weight
        ctx_size += n_layer * (3 * n_embd * n_embd * ggml_type_sizef(wtype)); // attn_Wqkv_weight
        ctx_size += n_layer * (n_embd * n_embd * ggml_type_sizef(wtype));     // attn_out_proj_weight
        ctx_size += n_layer * (n_embd * ggml_type_sizef(GGML_TYPE_F32));      // ln_2_weight
        ctx_size += n_layer * (4 * n_embd * n_embd * ggml_type_sizef(wtype)); // mlp_mlp_up_weight
        ctx_size += n_layer * (n_embd * n_embd * 4 * ggml_type_sizef(wtype)); // mlp_mlp_down_weight

        ctx_size += n_ctx * n_layer * n_embd * ggml_type_sizef(GGML_TYPE_F16); // memory_k
        ctx_size += n_ctx * n_layer * n_embd * ggml_type_sizef(GGML_TYPE_F16); // memory_v

        ctx_size += (1 + 6 * n_layer) * 512; // object overhead

        printf("%s: ggml ctx size = %6.2f MB\n", __func__, ctx_size / (1024.0 * 1024.0));
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

        const size_t n_embd = hparams.d_model;
        const size_t n_layer = hparams.n_layers;
        const size_t n_vocab = hparams.n_vocab;

        model.layers.resize(n_layer);

        model.wte_weight = ggml_new_tensor_2d(ctx, wtype, n_embd, n_vocab);
        model.norm_f_weight = ggml_new_tensor_1d(ctx, GGML_TYPE_F32, n_embd);

        // map by name
        model.tensors["transformer.wte.weight"] = model.wte_weight;
        model.tensors["transformer.norm_f.weight"] = model.norm_f_weight;

        for (int i = 0; i < (int)n_layer; ++i) {
            auto & layer = model.layers[i];

            layer.norm_1_weight = ggml_new_tensor_1d(ctx, GGML_TYPE_F32, n_embd);
            layer.c_attn_wqkv_weight = ggml_new_tensor_2d(ctx, wtype, n_embd, 3 * n_embd);
            layer.c_attn_out_proj_weight = ggml_new_tensor_2d(ctx, wtype, n_embd, n_embd);
            layer.norm_2_weight = ggml_new_tensor_1d(ctx, GGML_TYPE_F32, n_embd);
            layer.ffn_up_proj = ggml_new_tensor_2d(ctx, wtype, n_embd, 4 * n_embd);
            layer.ffn_down_proj = ggml_new_tensor_2d(ctx, wtype, 4 * n_embd, n_embd);

            // map by name
            model.tensors["transformer.blocks." + std::to_string(i) + ".norm_1.weight"] = layer.norm_1_weight;
            model.tensors["transformer.blocks." + std::to_string(i) + ".attn.Wqkv.weight"] = layer.c_attn_wqkv_weight;
            model.tensors["transformer.blocks." + std::to_string(i) + ".attn.out_proj.weight"] =
                layer.c_attn_out_proj_weight;
            model.tensors["transformer.blocks." + std::to_string(i) + ".norm_2.weight"] = layer.norm_2_weight;
            model.tensors["transformer.blocks." + std::to_string(i) + ".ffn.up_proj.weight"] = layer.ffn_up_proj;
            model.tensors["transformer.blocks." + std::to_string(i) + ".ffn.down_proj.weight"] = layer.ffn_down_proj;
        }
    }

    // key + value memory
    {
        const auto & hparams = model.hparams;

        const int n_embd = hparams.d_model;
        const int n_layer = hparams.n_layers;
        const int n_ctx = hparams.max_seq_len;

        const int64_t n_mem = n_layer * n_ctx;
        const int64_t n_elements = n_embd * n_mem;

        model.memory_k = ggml_new_tensor_1d(ctx, GGML_TYPE_F16, n_elements);
        model.memory_v = ggml_new_tensor_1d(ctx, GGML_TYPE_F16, n_elements);

        const size_t memory_size = ggml_nbytes(model.memory_k) + ggml_nbytes(model.memory_v);

        printf("%s: memory_size = %8.2f MB, n_mem = %" PRIu64 "\n", __func__, memory_size / 1024.0 / 1024.0, n_mem);
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
            fin.read(reinterpret_cast<char *>(&ttype), sizeof(ttype));

            if (fin.eof()) {
                break;
            }

            int32_t nelements = 1;
            int32_t ne[2] = {1, 1};
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
                fprintf(stderr,
                        "%s: tensor '%s' has wrong shape in model file: got [%5d, "
                        "%5d], expected [%5d, %5d]\n",
                        __func__, name.c_str(), (int)tensor->ne[0], (int)tensor->ne[1], ne[0], ne[1]);
                return false;
            }

            // for debugging
            if (0) {
                printf("%24s - [%5d, %5d], type = %6s, %6.2f MB, %9zu bytes\n", name.c_str(), ne[0], ne[1],
                       ggml_type_name(ggml_type(ttype)), ggml_nbytes(tensor) / 1024.0 / 1024.0, ggml_nbytes(tensor));
            }

            const size_t bpe = ggml_type_size(ggml_type(ttype));

            if ((nelements * bpe) / ggml_blck_size(tensor->type) != ggml_nbytes(tensor)) {
                fprintf(stderr,
                        "%s: tensor '%s' has wrong size in model file: got %zu, "
                        "expected %zu\n",
                        __func__, name.c_str(), ggml_nbytes(tensor), nelements * bpe);
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

        printf("%s: model size = %8.2f MB / num tensors = %d\n", __func__, total_size / 1024.0 / 1024.0, n_tensors);
    }

    fin.close();

    return true;
}

```

This is a Rust implementation of the FastGSLM model for natural language processing task. FastGSLM is an open-source library developed by Facebook AI Research, which is based on gradient-based optimization of neural networks for machine translation tasks.

The Rust implementation includes the forward and backward passes of the model, as well as the computation of the softmax probabilities for the output tokens. The input tokens are processed through the embedding weights, which are then passed through the model to compute the output logits.

Note that the output probabilities are computed using the `ggml_soft_max` function, which computes the probability distribution over the output tokens. The resulting output probabilities are then used to compute the embedding weights, which are used to store the input embeddings for the next layer of the model.


```cpp
// evaluate the transformer
//
//   - model:     the model
//   - n_threads: number of threads to use
//   - n_past:    the context size so far
//   - embd_inp:  the embeddings of the tokens in the context
//   - embd_w:    the predicted logits for the next token
//
bool replit_eval(const replit_model & model, const int n_threads, const int n_past,
                 const std::vector<gpt_vocab::id> & embd_inp, std::vector<float> & embd_w, bool logits_all,
                 size_t & mem_per_token) {
    const int N = embd_inp.size();

    const auto & hparams = model.hparams;

    const int n_embd = hparams.d_model;
    const int n_layer = hparams.n_layers;
    const int n_head = hparams.n_heads;
    const int n_vocab = hparams.n_vocab;
    const int n_ctx = hparams.max_seq_len;
    const float eps = 1e-5f;

    static size_t buf_size = 256u * 1024 * 1024;
    static void * buf = malloc(buf_size);

    if (mem_per_token > 0 && mem_per_token * N > buf_size) {
        const size_t buf_size_new = 1.1 * (mem_per_token * N); // add 10% to account for ggml object overhead
        // printf("\n%s: reallocating buffer from %zu to %zu bytes\n", __func__,
        // buf_size, buf_size_new);

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
    memcpy(embd->data, embd_inp.data(), N * ggml_element_size(embd));

    struct ggml_tensor * inpL = ggml_get_rows(ctx0, model.wte_weight, embd);

    for (int il = 0; il < n_layer; ++il) {

        struct ggml_tensor * cur;

        // a = self.ln_1(x)
        {
            cur = ggml_norm(ctx0, inpL, eps);

            cur = ggml_mul(ctx0, ggml_repeat(ctx0, model.layers[il].norm_1_weight, cur), cur);
        }

        // self-attention
        //  b, _, past_key_value = self.attn(a, past_key_value=past_key_value,
        //  attn_bias=attn_bias, attention_mask=attention_mask,
        //  is_causal=is_causal)
        {
            // compute QKV
            cur = ggml_mul_mat(ctx0, model.layers[il].c_attn_wqkv_weight, cur);

            struct ggml_tensor * Qcur = ggml_view_2d(ctx0, cur, n_embd, N, cur->nb[1], 0 * sizeof(float) * n_embd);
            struct ggml_tensor * Kcur = ggml_view_2d(ctx0, cur, n_embd, N, cur->nb[1], 1 * sizeof(float) * n_embd);
            struct ggml_tensor * Vcur = ggml_view_2d(ctx0, cur, n_embd, N, cur->nb[1], 2 * sizeof(float) * n_embd);

            // store key and value to memory
            {
                struct ggml_tensor * k =
                    ggml_view_1d(ctx0, model.memory_k, N * n_embd,
                                 (ggml_element_size(model.memory_k) * n_embd) * (il * n_ctx + n_past));
                struct ggml_tensor * v =
                    ggml_view_1d(ctx0, model.memory_v, N * n_embd,
                                 (ggml_element_size(model.memory_v) * n_embd) * (il * n_ctx + n_past));

                ggml_build_forward_expand(gf, ggml_cpy(ctx0, Kcur, k));
                ggml_build_forward_expand(gf, ggml_cpy(ctx0, Vcur, v));
            }

            // Q = Qcur.contiguous().view(n_embd/n_head, n_head, N).permute(0,
            // 2, 1, 3) [64, N, 12]
            struct ggml_tensor * Q = ggml_permute(
                ctx0, ggml_cpy(ctx0, Qcur, ggml_new_tensor_3d(ctx0, GGML_TYPE_F32, n_embd / n_head, n_head, N)), 0, 2,
                1, 3);

            // K = Kmem.view(n_embd/n_head, n_head, n_past + N).permute(0, 2, 1,
            // 3) [64, n_past + N, 12]
            struct ggml_tensor * K =
                ggml_permute(ctx0,
                             ggml_reshape_3d(ctx0,
                                             ggml_view_1d(ctx0, model.memory_k, (n_past + N) * n_embd,
                                                          il * n_ctx * ggml_element_size(model.memory_k) * n_embd),
                                             n_embd / n_head, n_head, n_past + N),
                             0, 2, 1, 3);
            // K * Q
            struct ggml_tensor * KQ = ggml_mul_mat(ctx0, K, Q);

            // KQ_scaled = KQ / sqrt(n_embd/n_head)
            struct ggml_tensor * KQ_scaled =
                ggml_scale(ctx0, KQ, ggml_new_f32(ctx0, 1.0f / sqrt(float(n_embd) / n_head)));

            struct ggml_tensor * KQ_scaled_alibi = ggml_alibi(ctx0, KQ_scaled, n_past, n_head, 8.0f);

            // KQ_masked = mask_past(KQ_scaled)
            struct ggml_tensor * KQ_masked = ggml_diag_mask_inf(ctx0, KQ_scaled_alibi, n_past);

            // KQ = soft_max(KQ_masked)
            struct ggml_tensor * KQ_soft_max = ggml_soft_max(ctx0, KQ_masked);

            // V_trans = Vmem.view(n_embd/n_head, n_head, n_past + N).permute(1,
            // 2, 0, 3).contiguous() [n_past + N, 64, 12]
            struct ggml_tensor * V_trans = ggml_cpy(
                ctx0,
                ggml_permute(ctx0,
                             ggml_reshape_3d(ctx0,
                                             ggml_view_1d(ctx0, model.memory_v, (n_past + N) * n_embd,
                                                          il * n_ctx * ggml_element_size(model.memory_v) * n_embd),
                                             n_embd / n_head, n_head, n_past + N),
                             1, 2, 0, 3),
                ggml_new_tensor_3d(ctx0, model.memory_v->type, n_past + N, n_embd / n_head, n_head));

            // KQV = transpose(V) * KQ_soft_max
            struct ggml_tensor * KQV = ggml_mul_mat(ctx0, V_trans, KQ_soft_max);

            // KQV_merged = KQV.permute(0, 2, 1, 3)
            struct ggml_tensor * KQV_merged = ggml_permute(ctx0, KQV, 0, 2, 1, 3);

            // cur = KQV_merged.contiguous().view(n_embd, N)
            cur = ggml_cpy(ctx0, KQV_merged, ggml_new_tensor_2d(ctx0, GGML_TYPE_F32, n_embd, N));

            // projection
            { cur = ggml_mul_mat(ctx0, model.layers[il].c_attn_out_proj_weight, cur); }
        }

        inpL = ggml_add(ctx0, inpL, cur);

        // m = self.ln_2(x)
        {
            cur = ggml_norm(ctx0, inpL, eps);

            cur = ggml_mul(ctx0, ggml_repeat(ctx0, model.layers[il].norm_2_weight, cur), cur);
        }

        // n = self.mlp(m)
        {

            cur = ggml_mul_mat(ctx0, model.layers[il].ffn_up_proj, cur);

            // GELU activation
            cur = ggml_gelu(ctx0, cur);

            // projection
            // cur = proj_w*cur + proj_b
            cur = ggml_mul_mat(ctx0, model.layers[il].ffn_down_proj, cur);
        }

        // x = x + n
        inpL = ggml_add(ctx0, inpL, cur);
    }

    // norm
    {
        inpL = ggml_norm(ctx0, inpL, eps);
        // inpL = ln_f_g*inpL
        inpL = ggml_mul(ctx0, ggml_repeat(ctx0, model.norm_f_weight, inpL), inpL);
    }

    // output embedding weight tied to input embedding
    inpL = ggml_mul_mat(ctx0, model.wte_weight, inpL);

    // logits -> probs
    // inpL = ggml_soft_max(ctx0, inpL);

    // run the computation
    ggml_build_forward_expand(gf, inpL);
    ggml_graph_compute_with_ctx(ctx0, gf, n_threads);

    // std::cout << "Qcur" << std::endl;
    // print_tensor(Qcur);

    // if (n_past%100 == 0) {
    // ggml_graph_print(&gf);
    // ggml_graph_dump_dot(&gf, NULL, "mpt-model.dot");
    // }

    if (logits_all) {
        // return result for all tokens
        embd_w.resize(n_vocab * N);
        memcpy(embd_w.data(), (float *)ggml_get_data(inpL), sizeof(float) * n_vocab * N);
    } else {
        // return result for just the last token
        embd_w.resize(n_vocab);
        memcpy(embd_w.data(), (float *)ggml_get_data(inpL) + (n_vocab * (N - 1)), sizeof(float) * n_vocab);
    }

    if (mem_per_token == 0) {
        mem_per_token = ggml_used_mem(ctx0) / N;
    }
    // printf("used_mem = %zu\n", ggml_used_mem(ctx0));

    ggml_free(ctx0);

    return true;
}

```

This is a C++ implementation of the text-to-sequence (T2S) preprocessing step in the life cycle of a pre-trained language model. The T2S preprocessing step involves converting the input text to a sequential tensor, where each element in the tensor corresponds to a word in the vocabulary.

The function takes an input text tensor and a list of integers representing the vocabulary of words in the language model. It returns a processed tensor that contains the word indices for each word in the input text.

The function uses two main components:

1. A simple loop that goes through the input text tensor and embeddings the lowercase word indices to the embedding vectors in the model.
2. A second loop that goes through the embeddings and displays the processed text.

The function uses a global max pooling operation to combine multiple embeddings for each input sequence. It also uses the window\_size parameter to control the number of embeddings that are processed at once.

The function uses the deprecated function cjson\_write() to write the output text to a JSON file.


```cpp
int main(int argc, char ** argv) {
    const int64_t t_main_start_us = ggml_time_us();

    gpt_params params;
    params.model = "";

    if (gpt_params_parse(argc, argv, params) == false) {
        return 1;
    }

    if (params.seed < 0) {
        params.seed = time(NULL);
    }

    printf("%s: seed = %d\n", __func__, params.seed);

    std::mt19937 rng(params.seed);
    if (params.prompt.empty()) {
        if (!is_stdin_terminal()) {
            std::string line;
            while (std::getline(std::cin, line)) {
                params.prompt = params.prompt + "\n" + line;
            }
        } else {
            params.prompt = gpt_random_prompt(rng);
        }
    }

    int64_t t_load_us = 0;

    replit_tokenizer vocab;
    replit_model model;

    // load the model
    {
        const int64_t t_start_us = ggml_time_us();

        if (!replit_model_load(params.model, model, vocab)) {
            fprintf(stderr, "%s: failed to load model from '%s'\n", __func__, params.model.c_str());
            return 1;
        }

        t_load_us = ggml_time_us() - t_start_us;
    }

    int n_past = 0;

    int64_t t_sample_us = 0;
    int64_t t_predict_us = 0;

    std::vector<float> logits;

    // tokenize the prompt
    std::vector<std::size_t> embd_inp = replit_tokenizer_tokenize(vocab, params.prompt);

    printf("%s: number of tokens in prompt = %zu\n", __func__, embd_inp.size());

    for (size_t i = 0; i < embd_inp.size(); i++) {
        printf("%s: token[%zu] = %6zu\n", __func__, i, embd_inp[i]);
        // vocab.id_to_token.at(embd_inp[i]).c_str()
    }
    printf("\n");

    params.n_predict = std::min(params.n_predict, model.hparams.max_seq_len - (int)embd_inp.size());

    std::vector<gpt_vocab::id> embd;

    // determine the required inference memory per token:
    size_t mem_per_token = 0;
    replit_eval(model, params.n_threads, 0, {0, 1, 2, 3}, logits, false, mem_per_token);

    for (size_t i = embd.size(); i < embd_inp.size() + params.n_predict; i++) {
        // predict
        if (embd.size() > 0) {
            const int64_t t_start_us = ggml_time_us();

            if (!replit_eval(model, params.n_threads, n_past, embd, logits, false, mem_per_token)) {
                printf("Failed to predict\n");
                return 1;
            }

            t_predict_us += ggml_time_us() - t_start_us;
        }

        n_past += embd.size();
        embd.clear();

        if (i >= embd_inp.size()) {
            // sample next token
            const int top_k = params.top_k;
            const float top_p = params.top_p;
            const float temp = params.temp;

            const int n_vocab = model.hparams.n_vocab;

            gpt_vocab::id id = 0;

            {
                const int64_t t_start_sample_us = ggml_time_us();

                id = gpt_sample_top_k_top_p(vocab.raw_vocab, logits.data() + (logits.size() - n_vocab), top_k, top_p,
                                            temp, rng);

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
            printf("%s", replit_tokenizer_detokenize(vocab, {static_cast<std::size_t>(id)}).c_str());
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
        printf("%s:     load time = %8.2f ms\n", __func__, t_load_us / 1000.0f);
        printf("%s:   sample time = %8.2f ms\n", __func__, t_sample_us / 1000.0f);
        printf("%s:  predict time = %8.2f ms / %.2f ms per token\n", __func__, t_predict_us / 1000.0f,
               t_predict_us / 1000.0f / n_past);
        printf("%s:    total time = %8.2f ms\n", __func__, (t_main_end_us - t_main_start_us) / 1000.0f);
    }

    ggml_free(model.ctx);

    return 0;
}

```

# `examples/replit/quantize.cpp`

这段代码是一个通用的C++代码，旨在定义了一个名为"ggml"的机器学习格式，该格式允许用户将多个数据集(通常为图像或音频)和相关元数据存储在同一个文件中。该代码包含以下几个部分：

1. 引入"ggml/ggml.h"头文件，定义了ggml的数据结构、函数和变量类型。
2. 引入"common-ggml.h"和"common.h"头文件，定义了一些通用的函数和数据结构。
3. 引入从"cassert"、"cmath"、"cstdio"、"cstring"、"fstream"、"map"、"regex"、"string"和"vector"的C头文件。
4. 在通用的"common.h"中，定义了多个函数，包括"ggml_read_image"、"ggml_write_image"、"ggml_read_audio"和"ggml_write_audio"。
5. 在main函数中，读取用户输入的文件并读取其内容，然后根据需要将其保存为ggml格式。

该代码的作用是定义了一个通用的机器学习格式，以存储多个数据集和相关元数据。该格式支持图像和音频数据，并允许用户将数据集存储在同一个文件中。用户需要使用该格式提供的函数来读取和保存数据集，并自行定义数据集的格式和结构。


```cpp
#include "ggml/ggml.h"

#include "common-ggml.h"
#include "common.h"

#include <cassert>
#include <cmath>
#include <cstdio>
#include <cstring>
#include <fstream>
#include <map>
#include <regex>
#include <string>
#include <vector>

```

This is a C++ function that takes in a High-根据细分数型（model parameters） and outputs the quantized version of the model.

It first reads in the name of the input file and the size of the input file from the user, and then opens the input file.

Then it reads in the number of input sequences and the maximum sequence length from the input file and opens the input file.

It reads in the number of heads, the number of layers, and the number of vocabulary from the input file and opens the input file.

It then writes the size of the input file, the maximum sequence length, the number of heads, and the number of layers to the output file.

It then writes the data type of the output querying the user and the data type of the queried data.

Then it reads in the input file's content and writes it to the output file.

Finally, it checks if the quantization process failed and prints out the error message.

It should be noted that this function assumes that the input file is a valid high-根据细分数型 and that it has the correct format for quantization.


```cpp
struct mpt_hparams {
    int32_t d_model     = 0;
    int32_t max_seq_len = 0;
    int32_t n_heads     = 0;
    int32_t n_layers    = 0;
    int32_t n_vocab     = 0;
    int32_t ftype       = 0;
};

// quantize a model
bool mpt_model_quantize(const std::string & fname_inp,
                        const std::string & fname_out, ggml_ftype ftype) {

    printf("%s: loading model from '%s'\n", __func__, fname_inp.c_str());

    auto finp = std::ifstream(fname_inp, std::ios::binary);
    if (!finp) {
        fprintf(stderr, "%s: failed to open '%s' for reading\n", __func__,
                fname_inp.c_str());
        return false;
    }

    auto fout = std::ofstream(fname_out, std::ios::binary);
    if (!fout) {
        fprintf(stderr, "%s: failed to open '%s' for writing\n", __func__,
                fname_out.c_str());
        return false;
    }

    // verify magic
    {
        uint32_t magic;
        finp.read((char *)&magic, sizeof(magic));
        if (magic != GGML_FILE_MAGIC) {
            fprintf(stderr, "%s: invalid model file '%s' (bad magic)\n",
                    __func__, fname_inp.c_str());
            return false;
        }

        fout.write((char *)&magic, sizeof(magic));
    }

    mpt_hparams hparams;

    // load hparams
    {
        finp.read((char *) &hparams.d_model,     sizeof(hparams.d_model));
        finp.read((char *) &hparams.max_seq_len, sizeof(hparams.max_seq_len));
        finp.read((char *) &hparams.n_heads,     sizeof(hparams.n_heads));
        finp.read((char *) &hparams.n_layers,    sizeof(hparams.n_layers));
        finp.read((char *) &hparams.n_vocab,     sizeof(hparams.n_vocab));
        finp.read((char *) &hparams.ftype,       sizeof(hparams.ftype));

        const int32_t qntvr_src =    hparams.ftype / GGML_QNT_VERSION_FACTOR;
        const int32_t ftype_dst = GGML_QNT_VERSION * GGML_QNT_VERSION_FACTOR + ftype;

        printf("%s: d_model     = %d\n", __func__, hparams.d_model);
        printf("%s: max_seq_len = %d\n", __func__, hparams.max_seq_len);
        printf("%s: n_heads     = %d\n", __func__, hparams.n_heads);
        printf("%s: n_layers    = %d\n", __func__, hparams.n_layers);
        printf("%s: n_vocab     = %d\n", __func__, hparams.n_vocab);
        printf("%s: ftype (src) = %d\n", __func__, hparams.ftype);
        printf("%s: qntvr (src) = %d\n", __func__, qntvr_src);
        printf("%s: ftype (dst) = %d\n", __func__, ftype_dst);
        printf("%s: qntvr (dst) = %d\n", __func__, GGML_QNT_VERSION);

        fout.write((char *) &hparams.d_model,     sizeof(hparams.d_model));
        fout.write((char *) &hparams.max_seq_len, sizeof(hparams.max_seq_len));
        fout.write((char *) &hparams.n_heads,     sizeof(hparams.n_heads));
        fout.write((char *) &hparams.n_layers,    sizeof(hparams.n_layers));
        fout.write((char *) &hparams.n_vocab,     sizeof(hparams.n_vocab));
        fout.write((char *) &ftype_dst,           sizeof(ftype_dst));
    }

    // load vocab
    {
        const int32_t n_vocab = hparams.n_vocab;

        std::string word;
        for (int i = 0; i < n_vocab; i++) {
            uint32_t len;
            finp.read((char *)&len, sizeof(len));
            fout.write((char *)&len, sizeof(len));

            word.resize(len);
            finp.read((char *)word.data(), len);
            fout.write((char *)word.data(), len);

            float prob;
            finp.read((char *)&prob, sizeof(prob));
            fout.write((char *)&prob, sizeof(prob));
        }
    }

    printf("%s: quantizing tensors\n", __func__);

    // regexes of tensor names to be quantized
    const std::vector<std::string> to_quant = {
        ".*weight",
    };

    if (!ggml_common_quantize_0(finp, fout, ftype, to_quant, {})) {
        fprintf(stderr, "%s: failed to quantize model '%s'\n", __func__,
                fname_inp.c_str());
        return false;
    }

    finp.close();
    fout.close();

    return true;
}

```

This is a C++ program that appears to quantize a machine learning model represented in a binary file (model-f32.bin and model-quant.bin) to use a model-f16 data type. The program takes two command-line arguments:

* The first argument is the input model file (model-f32.bin or model-quant.bin).
* The second argument is the output quantized model file (model-quant.bin).

The program first checks that the input file is valid and then reads in the contents of the file. If the model file is not valid, the program prints an error message and returns 1.

Next, the program reads in the input model file and quantizes it to use a model-f16 data type. The quantization process takes place in a separate function (q_model<br>), which takes as input the original model file and the output quantized model file. The function returns the quantized model file.

Finally, the program reports the timing information for the quantization process by measuring the time it takes to read in the input and output files and calculate the total time taken.

Note that the program assumes that the input and output files are valid models and that the model-f16 data type is properly defined.


```cpp
// usage:
//  ./replit-quantize models/replit/ggml-model.bin
//  models/replit/ggml-model-quant.bin type
//
int main(int argc, char ** argv) {
    if (argc != 4) {
        fprintf(stderr, "usage: %s model-f32.bin model-quant.bin type\n",
                argv[0]);
        ggml_print_ftypes(stderr);
        return 1;
    }

    // needed to initialize f16 tables
    {
        struct ggml_init_params params = {0, NULL, false};
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

        if (!mpt_model_quantize(fname_inp, fname_out, ggml_ftype(ftype))) {
            fprintf(stderr, "%s: failed to quantize model from '%s'\n",
                    __func__, fname_inp.c_str());
            return 1;
        }

        t_quantize_us = ggml_time_us() - t_start_us;
    }

    // report timing
    {
        const int64_t t_main_end_us = ggml_time_us();

        printf("\n");
        printf("%s: quantize time = %8.2f ms\n", __func__,
               t_quantize_us / 1000.0f);
        printf("%s:    total time = %8.2f ms\n", __func__,
               (t_main_end_us - t_main_start_us) / 1000.0f);
    }

    return 0;
}

```