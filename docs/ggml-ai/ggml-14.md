# GGML源码解析 14

# `examples/sam/convert-pth-to-ggml.py`

这段代码的作用是将一个保存为SAM模型的checkpoint文件转换为ggml兼容的文件。它需要一个输入文件、一个输出文件夹和一个文件类型参数。如果输入参数不足3个，程序会输出一个使用说明，指出如何正确使用该程序。

程序的核心是转换函数，它将SAM模型的checkpoint文件中的数据转换为ggml兼容的格式。转换后的文件可以在指定的输出文件夹中保存为ggml兼容格式。

程序的输入参数包括输入文件、输出文件夹和文件类型参数。输入文件是一个SAM模型checkpoint文件，程序可以使用不同的文件类型参数指定输出文件中的数据类型。

程序的输出文件是一个ggml兼容的文件，它保存了输入文件的变换数据，以及一些元数据，如模型的架构、损失函数等。


```cpp
# Convert a SAM model checkpoint to a ggml compatible file
#

import sys
import torch
import struct
import numpy as np

if len(sys.argv) < 3:
    print("Usage: convert-pth-to-ggml.py file-model dir-output [ftype]\n")
    print("  ftype == 0 -> float32")
    print("  ftype == 1 -> float16")
    sys.exit(1)

# output in the same directory as the model
```

这段代码的作用是创建一个文件名模型（fname_model）和一个输出目录（dir_out），并将模型文件名（fname_out）设置为输出目录和模型文件名的组合。它还定义了一个可能的输入数据类型列表，并将输入数据类型类型（ftype_str）映射为字符串（ftype_str）。最后，它检查输入参数的数量是否大于3，如果是，则执行第三步并将输入参数的第三个值分配给输入数据类型（ftype）。


```cpp
fname_model = sys.argv[1]
dir_out     = sys.argv[2]
fname_out   = dir_out + "/ggml-model.bin"

# possible data types
#   ftype == 0 -> float32
#   ftype == 1 -> float16
#
# map from ftype to string
ftype_str = ["f32", "f16"]

ftype = 1
if len(sys.argv) > 3:
    ftype = int(sys.argv[3])

```

这段代码的主要作用是检查输入数据类型（ftype）是否合法，如果输入不合法，则输出并退出程序。然后，根据输入的ftype，将fname_out文件名中的".bin"替换为ftype_str[ftype] + ".bin"。接下来，设置编码器参数，根据输入的ftype来加载预训练的模型，并将模型的参数存储在模型变量中。最后，对输入的image_encoder.blocks.0.norm1.weight参数进行归一化预处理，以增加模型的稳定性。


```cpp
if ftype < 0 or ftype > 1:
    print("Invalid ftype: " + str(ftype))
    sys.exit(1)

fname_out = fname_out.replace(".bin", "-" + ftype_str[ftype] + ".bin")

# Default params are set to sam_vit_b checkpoint
n_enc_state = 768
n_enc_layers = 12
n_enc_heads = 12
n_enc_out_chans = 256
n_pt_embd = 4

model = torch.load(fname_model, map_location="cpu")
for k, v in model.items():
    print(k, v.shape)
    if k == "image_encoder.blocks.0.norm1.weight":
        n_enc_state = v.shape[0]

```

这段代码是使用PyTorch中的Sampling and Multi-Heading API（v1）实现的。它用于在给定n_enc_state的值时定义n_enc_layers和n_enc_heads层的参数。

具体来说，如果n_enc_state的值为1024，则表示要实现sam_vit_l层，该层具有24个encoder的heads，并在endpoints处进行池化。

而如果n_enc_state的值为1280，则表示要实现sam_vit_h层，该层具有32个encoder的heads，并在endpoints处进行池化。

在代码中，还定义了一个hparams变量，用于存储n_enc_state、n_enc_layers、n_enc_heads和n_enc_out_chans等参数的值。这些参数在Sampling and Multi-Heading API中用于控制层的行为。


```cpp
if n_enc_state == 1024: # sam_vit_l
    n_enc_layers = 24
    n_enc_heads  = 16
elif n_enc_state == 1280: # sam_vit_h
    n_enc_layers = 32
    n_enc_heads  = 16

hparams = {
    "n_enc_state":      n_enc_state,
    "n_enc_layers":     n_enc_layers,
    "n_enc_heads":      n_enc_heads,
    "n_enc_out_chans":  n_enc_out_chans,
    "n_pt_embd":        n_pt_embd,
}

```

这段代码的作用是输出一个名为 `hparams` 的字典中所有键值对。

首先，通过调用 `print(hparams)` 来输出字典 `hparams` 的内容，以便用户查看字典中的键和值。

接着，使用 `for` 循环遍历 `model` 对象中的所有键 `k` 和对应的值 `v`，并使用 `print` 函数输出每个键和对应的形状 `v`。

在循环体中，首先通过 `struct.pack("i", 0x67676d6c)` 将 `0x67676d6c` 转换为整数并打印出来，这是一个 magic 标记，告诉 `gensim` 库使用哪种格式来写入 `fname_out` 文件。

然后，通过循环引用 `hparams` 字典中的 `"n_enc_state"`、`"n_enc_layers"`、`"n_enc_heads"` 和 `"n_enc_out_chans"` 四个键，将它们打印出来，并按照格式 `struct.pack("i", hparams[key])` 对应的样式进行包装，以便 `gensim` 库能够正确地读取或写入这些键值对。


```cpp
print(hparams)

for k, v in model.items():
    print(k, v.shape)

#exit()
#code.interact(local=locals())

fout = open(fname_out, "wb")

fout.write(struct.pack("i", 0x67676d6c)) # magic: ggml in hex
fout.write(struct.pack("i", hparams["n_enc_state"]))
fout.write(struct.pack("i", hparams["n_enc_layers"]))
fout.write(struct.pack("i", hparams["n_enc_heads"]))
fout.write(struct.pack("i", hparams["n_enc_out_chans"]))
```

This code appears to be a Python script that performs various transformations on a dataset, such as converting it from a 1D tensor to a 4D tensor, reshaping it, and converting it from a float16 to a float32 data type. It is also modifying the header information in the file to specify the dimensions and data types of the data. The script is using the `ggml_encode` function to convert the data, which is a command line tool for encoding a variety of data formats including ISO-8859-1 and FPy16/F32. The script is then reading the header information and the data, and writing the header information and the data to a file.


```cpp
fout.write(struct.pack("i", hparams["n_pt_embd"]))
fout.write(struct.pack("i", ftype))

for k, v in model.items():
    name = k
    shape = v.shape

    if name[:19] == "prompt_encoder.mask":
        continue

    print("Processing variable: " + name + " with shape: ", shape, " and type: ", v.dtype)

    #data = tf.train.load_variable(dir_model, name).squeeze()
    #data = v.numpy().squeeze()
    data = v.numpy()
    n_dims = len(data.shape)

    # for efficiency - transpose some matrices
    # "model/h.*/attn/c_attn/w"
    # "model/h.*/attn/c_proj/w"
    # "model/h.*/mlp/c_fc/w"
    # "model/h.*/mlp/c_proj/w"
    #if name[-14:] == "/attn/c_attn/w" or \
    #   name[-14:] == "/attn/c_proj/w" or \
    #   name[-11:] == "/mlp/c_fc/w" or \
    #   name[-13:] == "/mlp/c_proj/w":
    #    print("  Transposing")
    #    data = data.transpose()

    dshape = data.shape

    # default type is fp16
    ftype_cur = 1
    if ftype == 0 or n_dims == 1 or \
            name == "image_encoder.pos_embed" or \
            name.startswith("prompt_encoder") or \
            name.startswith("mask_decoder.iou_token") or \
            name.startswith("mask_decoder.mask_tokens"):
        print("  Converting to float32")
        data = data.astype(np.float32)
        ftype_cur = 0
    else:
        print("  Converting to float16")
        data = data.astype(np.float16)

    # reshape the 1D bias into a 4D tensor so we can use ggml_repeat
    # keep it in F32 since the data is small
    if name == "image_encoder.patch_embed.proj.bias":
        data = data.reshape(1, data.shape[0], 1, 1)
        n_dims = len(data.shape)
        dshape = data.shape

    print("  New shape: ", dshape)

    # header
    str = name.encode('utf-8')
    fout.write(struct.pack("iii", n_dims, len(str), ftype_cur))
    for i in range(n_dims):
        fout.write(struct.pack("i", dshape[n_dims - 1 - i]))
    fout.write(str)

    # data
    data.tofile(fout)

```

这段代码的作用是关闭名为 "fout" 的文件，并输出文件名 "fout_done.txt"。

具体来说，第一行使用 `fout.close()` 方法关闭文件输出流(通常是文件对象 `fout`)，这样当程序运行完毕后，文件输出流就不再流露出数据。

第二行使用 `print()` 函数输出 "Done" 和文件名 "fout_done.txt"。其中，`print()` 函数是一个内置函数，可以将内容输出到命令行或控制台。在这里，它将输出一个字符串 "Done"，并跟上文件名 "fout_done.txt"。

第三行也是使用 `print()` 函数，但是它输出的是一个空字符串，用来在输出结果和之前的输出结果之间留一个空行。


```cpp
fout.close()

print("Done. Output file: " + fname_out)
print("")

```

# `examples/sam/main.cpp`

这段代码定义了一些预处理头文件，用于数学相关的定义和安全性提醒。

首先，定义了两个符号常量：

```cpp
#define _USE_MATH_DEFINES // for M_PI
#define _CRT_SECURE_NO_DEPRECATE // Disables ridiculous "unsafe" warnings on Windows
```

第一个符号常量`_USE_MATH_DEFINES`表示会包含`M_PI`的定义，使得程序可以更方便地使用`M_PI`。

第二个符号常量`_CRT_SECURE_NO_DEPRECATE`表示会在Windows上禁用一些警告，以提高代码的安全性。

接着，引入了两个头文件：

```cpp
#include "ggml.h"
#include "ggml-alloc.h"
```

这是Ggml库的头文件，可能是一个C++的图形渲染引擎。

```cpp
#define STB_IMAGE_IMPLEMENTATION // STB_IMAGE_IMPLEMENTATION
#define STB_IMAGE_WRITE_IMPLEMENTATION // STB_IMAGE_WRITE_IMPLEMENTATION
```

这是两个头文件，分别表示IMAGE和IMAGE_WRITE的实现。这两个头文件很可能包含一些通用的图像处理函数和类型。

```cpp
#include "stb_image.h" // STB_IMAGE_IMPLEMENTATION
#include "stb_image_write.h" // STB_IMAGE_WRITE_IMPLEMENTATION
```

这两个头文件应该包含了一些通用的图像处理函数和类型，以及IMAGE和IMAGE_WRITE的实现。

总结：

这段代码定义了一些预处理头文件，包括数学相关的定义和安全性提醒，以及图像处理方面的头文件。这些头文件很可能用于一个C++的图形渲染引擎，用于支持一些通用的图像处理函数和类型。


```cpp
#define _USE_MATH_DEFINES // for M_PI
#define _CRT_SECURE_NO_DEPRECATE // Disables ridiculous "unsafe" warnigns on Windows

#include "ggml.h"
#include "ggml-alloc.h"
#define STB_IMAGE_IMPLEMENTATION
#include "stb_image.h"
#define STB_IMAGE_WRITE_IMPLEMENTATION
#include "stb_image_write.h"

#include <cassert>
#include <cmath>
#include <cstddef>
#include <cstdio>
#include <cstring>
```

This is a CUDA code that uses a pre-trained model for image classification


```cpp
#include <fstream>
#include <map>
#include <string>
#include <vector>
#include <thread>
#include <cinttypes>

#if defined(_MSC_VER)
#pragma warning(disable: 4244 4267) // possible loss of data
#endif

// default hparams (ViT-B SAM)
struct sam_hparams {
    int32_t n_enc_state               = 768;
    int32_t n_enc_layer               = 12;
    int32_t n_enc_head                = 12;
    int32_t n_enc_out_chans           = 256;
    int32_t n_pt_embd                 = 4;
    int32_t n_dec_heads               = 8;
    int32_t ftype                     = 1;
    float   mask_threshold            = 0.f;
    float   iou_threshold             = 0.88f;
    float   stability_score_threshold = 0.95f;
    float   stability_score_offset    = 1.0f;
    float   eps                       = 1e-6f;
    float   eps_decoder_transformer   = 1e-5f;

    int32_t n_enc_head_dim() const { return n_enc_state / n_enc_head; }
    int32_t n_img_size()     const { return 1024; }
    int32_t n_window_size()  const { return 14; }
    int32_t n_patch_size()   const { return 16; }
    int32_t n_img_embd()     const { return n_img_size() / n_patch_size(); }

    std::vector<int32_t> global_attn_indices() const {
        switch (n_enc_state) {
            case  768: return {  2,  5,  8, 11 };
            case 1024: return {  5, 11, 17, 23 };
            case 1280: return {  7, 15, 23, 31 };
            default:
                {
                    fprintf(stderr, "%s: unsupported n_enc_state = %d\n", __func__, n_enc_state);
                } break;
        };

        return {};
    }

    bool is_global_attn(int32_t layer) const {
        const auto indices = global_attn_indices();

        for (const auto & idx : indices) {
            if (layer == idx) {
                return true;
            }
        }

        return false;
    }
};

```

这是一个结构体数组，名为 "sam_layer_enc"，它包含了以下元素：

- 1个 "norm1_w"，是一个结构体指针，指向一个二维数组，这个数组可能用于计算输入数据的均值和标准差。
- 1个 "norm1_b"，也是一个结构体指针，指向一个二维数组，这个数组可能用于计算输入数据的归一化变换。
- 1个 "rel_pos_w"，是一个结构体指针，指向一个二维数组，这个数组可能用于计算输入数据的平移量。
- 1个 "rel_pos_h"，也是一个结构体指针，指向一个二维数组，这个数组可能用于计算输入数据的对齐量。
- 1个 "qkv_w"，是一个结构体指针，指向一个二维数组，这个数组可能用于计算输入数据的查询和关键值。
- 1个 "qkv_b"，也是一个结构体指针，指向一个二维数组，这个数组可能用于计算输入数据的查询和关键值。
- 1个 "proj_w"，是一个结构体指针，指向一个二维数组，这个数组可能用于计算输入数据的投影。
- 1个 "proj_b"，也是一个结构体指针，指向一个二维数组，这个数组可能用于计算输入数据的投影。
- 1个 "norm2_w"，是一个结构体指针，指向一个二维数组，这个数组可能用于计算输入数据的归一化变换。
- 1个 "norm2_b"，也是一个结构体指针，指向一个二维数组，这个数组可能用于计算输入数据的归一化变换。
- 1个 "mlp_lin1_w"，是一个结构体指针，指向一个二维数组，这个数组可能用于计算输入数据的密集层1的加权连接。
- 1个 "mlp_lin1_b"，也是一个结构体指针，指向一个二维数组，这个数组可能用于计算输入数据的密集层1的加权连接。
- 1个 "mlp_lin2_w"，是一个结构体指针，指向一个二维数组，这个数组可能用于计算输入数据的密集层2的加权连接。
- 1个 "mlp_lin2_b"，也是一个结构体指针，指向一个二维数组，这个数组可能用于计算输入数据的密集层2的加权连接。


```cpp
struct sam_layer_enc {
    struct ggml_tensor * norm1_w;
    struct ggml_tensor * norm1_b;

    struct ggml_tensor * rel_pos_w;
    struct ggml_tensor * rel_pos_h;

    struct ggml_tensor * qkv_w;
    struct ggml_tensor * qkv_b;

    struct ggml_tensor * proj_w;
    struct ggml_tensor * proj_b;

    struct ggml_tensor * norm2_w;
    struct ggml_tensor * norm2_b;

    struct ggml_tensor * mlp_lin1_w;
    struct ggml_tensor * mlp_lin1_b;

    struct ggml_tensor * mlp_lin2_w;
    struct ggml_tensor * mlp_lin2_b;
};

```

这段代码定义了一个名为sam_encoder_image的结构体，它包含了一些输入和一些输出。输入是一个ggml张量，输出是多个ggml张量。现在我来逐步解释它的作用。

首先，这个结构体有一个名为pe的ggml张量，这是一个输入。

其次，这个结构体有一个名为proj_w的ggml张量和一个名为proj_b的ggml张量，这两个张量都是输出，可以被用来进行图像的透视操作。

然后，这个结构体有一个名为neck_conv_0的ggml张量和一个名为neck_norm_0_w的ggml张量和一个名为neck_norm_0_b的ggml张量，这三个张量都是输入，可以被用来进行图像的低层处理，如池化、裁剪等。

接着，这个结构体有一个名为neck_conv_1的ggml张量和一个名为neck_norm_1_w的ggml张量和一个名为neck_norm_1_b的ggml张量，这三个张量都是输入，可以被用来进行图像的高层处理，如上采样、全连接等。

最后，这个结构体有一个名为layers的std::vector<sam_layer_enc>，这个std::vector包含了一些sam层编码的实例。

综上所述，这段代码定义了一个可以进行图像处理和编码的结构体，它包含了一些输入和一个输出，用于实现图像的低层到高层的处理和编码。


```cpp
struct sam_encoder_image {
    struct ggml_tensor * pe;

    struct ggml_tensor * proj_w;
    struct ggml_tensor * proj_b;

    struct ggml_tensor * neck_conv_0;
    struct ggml_tensor * neck_norm_0_w;
    struct ggml_tensor * neck_norm_0_b;
    struct ggml_tensor * neck_conv_1;
    struct ggml_tensor * neck_norm_1_w;
    struct ggml_tensor * neck_norm_1_b;

    std::vector<sam_layer_enc> layers;
};

```

这是一个结构体数组，其中包含两个结构体。第一个结构体是sam_encoder_prompt，它包含一个ggml_tensor类型的pe变量和一个ggml_tensor类型的not_a_pt_embd_w。not_a_pt_embd_w是一个transformer中的编码器输入注意力分配所使用的ggml_tensor。

第二个结构体是sam_layer_dec_transformer_attn，它包含一个ggml_tensor类型的q_w和一个ggml_tensor类型的q_b，它还包含一个ggml_tensor类型的k_w和一个ggml_tensor类型的k_b，这两个变量是transformer中的编码器和中枢（no_decoder）注意力所使用的ggml_tensor。

另外，它还包含一个ggml_tensor类型的v_w和一个ggml_tensor类型的v_b，这两个变量是transformer中的解码器输入注意力分配所使用的ggml_tensor。

最后，它包含一个ggml_tensor类型的out_w和一个ggml_tensor类型的out_b，这两个变量是transformer中的输出所使用的ggml_tensor。


```cpp
struct sam_encoder_prompt {
    struct ggml_tensor * pe;

    struct ggml_tensor * not_a_pt_embd_w;
    std::vector<struct ggml_tensor *> pt_embd;

    struct ggml_tensor * no_mask_embd_w;
    //std::vector<struct ggml_tensor *> mask_down_w;
    //std::vector<struct ggml_tensor *> mask_down_b;
};

struct  sam_layer_dec_transformer_attn {
    // q_proj
    struct ggml_tensor * q_w;
    struct ggml_tensor * q_b;

    // k_proj
    struct ggml_tensor * k_w;
    struct ggml_tensor * k_b;

    // v_proj
    struct ggml_tensor * v_w;
    struct ggml_tensor * v_b;

    // out_proj
    struct ggml_tensor * out_w;
    struct ggml_tensor * out_b;
};

```

这是一段使用深度学习模型结构的数据结构定义，其中包括一个名为sam_layer_dec_transformer的结构体。这个结构体包含多个输入输出注意力相关的成员变量。

sam_layer_dec_transformer包含一个名为self_attn的成员变量，它是一个sam_layer_dec_transformer_attn类型的结构体，负责保存输入序列中每个位置的注意力权重。

然后是一个名为norm1的成员变量，它是一个ggml_tensor类型的结构体，包含第一个自注意力层的权重和偏置。

接着是一个名为norm2的成员变量，它是一个ggml_tensor类型的结构体，包含第二个自注意力层的权重和偏置。

然后是一个名为mlp_lin1的成员变量，它是一个ggml_tensor类型的结构体，包含第一个多层感知层（MLP）的权重和偏置。

接着是一个名为mlp_lin2的成员变量，它是一个ggml_tensor类型的结构体，包含第二个多层感知层的权重和偏置。

然后是一个名为norm3的成员变量，它是一个ggml_tensor类型的结构体，包含第三个多层感知层的权重和偏置。

接着是一个名为norm4的成员变量，它是一个ggml_tensor类型的结构体，包含第四个多层感知层的权重和偏置。

最后，它还包含一个名为cross_attn_img_to_token的成员变量，它是一个sam_layer_dec_transformer_attn类型的结构体，用于处理输入序列中的图像到文本的注意力。


```cpp
struct sam_layer_dec_transformer {
    sam_layer_dec_transformer_attn self_attn;

    // norm1
    struct ggml_tensor * norm1_w;
    struct ggml_tensor * norm1_b;

    sam_layer_dec_transformer_attn cross_attn_token_to_img;

    // norm2
    struct ggml_tensor * norm2_w;
    struct ggml_tensor * norm2_b;

    // mlp.lin1
    struct ggml_tensor * mlp_lin1_w;
    struct ggml_tensor * mlp_lin1_b;

    // mlp.lin2
    struct ggml_tensor * mlp_lin2_w;
    struct ggml_tensor * mlp_lin2_b;

    // norm3
    struct ggml_tensor * norm3_w;
    struct ggml_tensor * norm3_b;

    // norm4
    struct ggml_tensor * norm4_w;
    struct ggml_tensor * norm4_b;

    sam_layer_dec_transformer_attn cross_attn_img_to_token;
};

```

This is a Rust implementation of the Transformer architecture for image classification tasks. It includes a Decoder component for each attention branch and an Image Output component. The Transformer architecture uses Split-comp Mel-Frequency Cepstral Coefficients (SPaT) embeddings and a multi-head self-attention mechanism.

It also includes an Image Decoder component, which performs a 2D down-sampling operation to increase the resolution of the input image.

The Transformer has multiple output channels, each of which is assigned a different downscaling factor to adjust the output resolution of the Image Output component. The output of the Transformer is passed through a series of SPaT embeddings and a final linear layer to generate the final image.

This implementation assumes that the input image has the same dimensions as the downscaled image and that the Downsampling layer has the same down-sampling factor for all channels.

You should be able to use this code as a starting point for building a Transformer architecture for image classification tasks and modify it according to your specific requirements.


```cpp
struct sam_layer_dec_output_hypernet_mlps {
    // mlps_*.layers.0
    struct ggml_tensor * w_0;
    struct ggml_tensor * b_0;

    // mlps_*.layers.1
    struct ggml_tensor * w_1;
    struct ggml_tensor * b_1;

    // mlps_*.layers.2
    struct ggml_tensor * w_2;
    struct ggml_tensor * b_2;
};

struct sam_decoder_mask {
    std::vector<sam_layer_dec_transformer> transformer_layers;

    // trasnformer.final_attn_token_to_image
    sam_layer_dec_transformer_attn transformer_final_attn_token_to_img;

    // transformer.norm_final
    struct ggml_tensor * transformer_norm_final_w;
    struct ggml_tensor * transformer_norm_final_b;

    // output_upscaling.0
    struct ggml_tensor * output_upscaling_0_w;
    struct ggml_tensor * output_upscaling_0_b;

    // output_upscaling.1
    struct ggml_tensor * output_upscaling_1_w;
    struct ggml_tensor * output_upscaling_1_b;

    // output_upscaling.3
    struct ggml_tensor * output_upscaling_3_w;
    struct ggml_tensor * output_upscaling_3_b;

    // output_hypernetworks_mlps
    std::vector<sam_layer_dec_output_hypernet_mlps> output_hypernet_mlps;

    // iou_prediction_head.0
    struct ggml_tensor * iou_prediction_head_0_w;
    struct ggml_tensor * iou_prediction_head_0_b;

    // iou_prediction_head.1
    struct ggml_tensor * iou_prediction_head_1_w;
    struct ggml_tensor * iou_prediction_head_1_b;

    // iou_prediction_head.2
    struct ggml_tensor * iou_prediction_head_2_w;
    struct ggml_tensor * iou_prediction_head_2_b;

    // iou_token.weight
    struct ggml_tensor * iou_token_w;

    // mask_tokens.weight
    struct ggml_tensor * mask_tokens_w;
};


```

这是一个结构体 `sam_state`，它用于表示一个序列标注模型的状态。

该结构体包含以下成员：

- `ggml_tensor * embd_img`：编码输入图像的内存指针。
- `ggml_tensor * low_res_masks`：低分辨率 mask 的内存指针。
- `ggml_tensor * iou_predictions`：IoU 预测的内存指针。
- `struct ggml_context * ctx`：当前 context 的指针。
- `std::vector<uint8_t> work_buffer`：用于保存计算过程中的临时数据。
- `std::vector<uint8_t> buf_alloc_img_enc`：用于记录编码输入图像所需的数据。
- `std::vector<uint8_t> buf_compute_img_enc`：用于记录计算 IoU 预测所需的 data。
- `std::vector<uint8_t> buf_alloc_fast`：用于记录编码输入图像所需的高效数据。
- `std::vector<uint8_t> buf_compute_fast`：用于记录计算 IoU 预测所需的高效数据。
- `struct ggml_allocr * allocr`：用于记录图像的 allocate_multi 函数的指针。

该结构体定义了一个 `sam_state` 类型的变量 `st`，该变量用于表示一个序列标注模型。`ggml_tensor * embd_img` 和 `ggml_tensor * low_res_masks` 和 `ggml_tensor * iou_predictions` 都用于表示输入图像的相关数据。


```cpp
struct sam_state {
    struct ggml_tensor * embd_img;

    struct ggml_tensor * low_res_masks;
    struct ggml_tensor * iou_predictions;

    //struct ggml_tensor * tmp_save = {};

    struct ggml_context * ctx;

    // buffer for `ggml_graph_plan.work_data`
    std::vector<uint8_t> work_buffer;
    // buffers to evaluate the model
    std::vector<uint8_t> buf_alloc_img_enc;
    std::vector<uint8_t> buf_compute_img_enc;

    std::vector<uint8_t> buf_alloc_fast;
    std::vector<uint8_t> buf_compute_fast;

    struct ggml_allocr  * allocr = {};
};

```

这段代码定义了一个名为`save_tensor`的函数，其作用是保存一个Tensor的状态，并将其保存到内存中。

具体来说，代码中首先检查`state.tmp_save`是否为`false`，如果是，就执行以下操作：

1. 从`state.ctx`中创建一个新的Tensor，其类型与`t`相同，尺寸与`t`相同，并且与`ne`大小对应的`dim`为`t`的大小减去1(因为减1后`ne`仍然是一个整数)。
2. 从`state.ctx`中复制`t`到新生成的Tensor中。
3. 从新生成的Tensor中构建前向推理张量。

如果`state.tmp_save`为`true`，则不做任何操作，直接返回。

接下来，定义了一个名为`sam_model`的结构体，其中包含一个`ctx`指针，用于与内存中的图形上下文交互。还包括一个`enc_img`变量，表示输入图像的内存缓冲区，一个`enc_prompt`变量，表示输入图像的提示信息，以及一个`dec`变量，表示输入图像的解码掩码。

最后，在`main`函数中，创建了一个`sam_model`实例，并调用`save_tensor`函数来保存输入图像到一个Tensor中，然后将其解码为图像，并显示输出图像。


```cpp
// void save_tensor(sam_state& state, struct ggml_tensor * t, struct ggml_cgraph * gf) {
//     if (!state.tmp_save) {
//         state.tmp_save = ggml_new_tensor(state.ctx, t->type, t->n_dims, t->ne);
//     }
//     struct ggml_tensor * tmp0 = ggml_cpy(state.ctx, t, state.tmp_save);
//     ggml_build_forward_expand(gf, tmp0);
// }

struct sam_model {
    sam_hparams hparams;

    sam_encoder_image  enc_img;
    sam_encoder_prompt enc_prompt;
    sam_decoder_mask   dec;

    //
    struct ggml_context * ctx;
    std::map<std::string, struct ggml_tensor *> tensors;
};

```

这段代码定义了两个结构体：sam_point和sam_image_u8。sam_point结构体有两个浮点型成员x和y。sam_image_u8结构体有两个整型成员：nx和ny，分别表示图像的宽度和高度。同时，它还有一个包含RGB图像数据的向量。

sam_image_u8结构体是内存布局为RGBRGBRGB...（循环）的8位图像的封装。这个结构体有一个成员函数，可以使用cuda_image_dependency纳浮点数类型（float32）的图像数据，但是，为了与CUDA的API兼容，该函数实际上使用的是int类型的nx和ny成员。这个函数可以像CUDA一样并行处理图像数据，从而提高性能。


```cpp
struct sam_point {
    float x;
    float y;
};

// RGB uint8 image
struct sam_image_u8 {
    int nx;
    int ny;

    std::vector<uint8_t> data;
};

// RGB float32 image
// Memory layout: RGBRGBRGB...
```

这段代码定义了一个名为sam_image_f32的结构体，表示法线图像数据。这个结构体有两个整型成员nx和ny，分别表示图像的宽度和高度。同时，它还有一个标量类型的成员data，是一个std::vector<float>类型的数据容器，用于存储图像数据。

接下来，定义了一个名为sam_params的结构体，用于存储参数。这个结构体有四个整型成员，分别是种子号，n_threads，模型，输入图像文件名和输出图像文件名，以及几个浮点型成员，分别是mask_threshold,iou_threshold,stability_score_threshold,stability_score_offset和eps。这些参数用于设置随机数生成器的种子，以及采样图像时的阈值和评分标准。

最后，定义了一个sam_point类型的成员pt，表示法线的坐标，是采样图像时的起点。


```cpp
struct sam_image_f32 {
    int nx;
    int ny;

    std::vector<float> data;
};

struct sam_params {
    int32_t seed      = -1; // RNG seed
    int32_t n_threads = std::min(4, (int32_t) std::thread::hardware_concurrency());

    std::string model     = "models/sam-vit-b/ggml-model-f16.bin"; // model path
    std::string fname_inp = "img.jpg";
    std::string fname_out = "img.out";
    float   mask_threshold            = 0.f;
    float   iou_threshold             = 0.88f;
    float   stability_score_threshold = 0.95f;
    float   stability_score_offset    = 1.0f;
    float   eps                       = 1e-6f;
    float   eps_decoder_transformer   = 1e-5f;
    sam_point pt = { 414.375f, 162.796875f, };
};

```

这段代码是一个函数，名为`print_t_f32`，它的作用是输出一个多维张量（struct ggml_tensor）中的数据。

具体来说，函数接受三个参数：

1. 一个字符串标题（在函数内部使用printf），作为函数名称的前缀。
2. 一个指向ggml_tensor类型的结构体指针（在函数内部使用*）。
3. 一个表示张量中元素数量的整数（在函数内部使用int）。

函数内部首先输出标题，然后输出张量维度信息，接着输出前10个元素。后续输出类似于在控制台上打印数据。最后输出张量的总和以及所有元素的累加和。


```cpp
void print_t_f32(const char* title, struct ggml_tensor * t, int n = 10) {
    printf("%s\n", title);
    float * data = (float *)t->data;
    printf("dims: % " PRId64 " % " PRId64 " % " PRId64 " % " PRId64 " f32\n", t->ne[0], t->ne[1], t->ne[2], t->ne[3]);
    printf("First & Last %d elements:\n", n);
    for (int i = 0; i < std::min((int) (t->ne[0]*t->ne[1]), n); i++) {
        printf("%.5f ", data[i]);
        if (i != 0 && i % t->ne[0] == 0) {
            printf("\n");
        }
    }
    printf("\n");
    for (int i = 0; i < std::min((int) (t->ne[0]*t->ne[1]), n); i++) {
        printf("%.5f ", data[ggml_nelements(t) - n + i]);
        if ((ggml_nelements(t) - n + i) % t->ne[0] == 0) {
            printf("\n");
        }
    }
    printf("\n");
    double sum = 0.0;
    for (int i = 0; i < ggml_nelements(t); i++) {
        sum += data[i];
    }
    printf("sum:  %f\n\n", sum);
}

```



这段代码定义了两个静态函数，分别是 `ggml_disconnect_node_from_graph()` 和 `ggml_graph_compute_helper()`。

`ggml_disconnect_node_from_graph()` 函数接收一个 `ggml_tensor` 类型的输入，并对其执行以下操作：

1. 将输入的 tensor 的操作类型更改为 `GGML_OP_NONE`。
2. 将输入 tensor 的所有输入源连接点设置为 `NULL`。

`ggml_graph_compute_helper()` 函数接收一个 `std::vector<uint8_t>` 类型的输入和一个 `ggml_cgraph` 类型的输入图形。它使用 `ggml_graph_plan()` 函数规划并创建一个计算图，然后使用 `ggml_graph_compute()` 函数执行计算图，将计算图的输出赋值给输入的 `std::vector<uint8_t>` 类型的缓冲区。

`ggml_graph_compute_helper()` 函数的主要作用是接收输入的图形，对其进行计算，并将计算图的输出赋值给一个 `std::vector<uint8_t>` 类型的缓冲区。这个缓冲区可以被后续的 `ggml_tensor` 类型的操作所使用，以执行对图形数据的操作。


```cpp
static void ggml_disconnect_node_from_graph(ggml_tensor * t) {
    t->op = GGML_OP_NONE;
    for (int i = 0; i < GGML_MAX_SRC; i++) {
        t->src[i] = NULL;
    }
}

static void ggml_graph_compute_helper(std::vector<uint8_t> & buf, ggml_cgraph * graph, int n_threads) {
    struct ggml_cplan plan = ggml_graph_plan(graph, n_threads);

    if (plan.work_size > 0) {
        buf.resize(plan.work_size);
        plan.work_data = buf.data();
    }

    ggml_graph_compute(graph, &plan);
}

```

这段代码是一个名为"gggml_sam_sin"的函数，属于GGML（General Graphical Modeling Library）库。这个函数的主要作用是计算一个低斯分布（连续型随机变量）的二维随机样本的二维插值。

具体来说，这段代码实现了一个低斯分布的插值算法，通过从高斯分布的二维随机样本中提取数据，然后对低斯分布的二维随机样本进行插值，使得低斯分布的二维随机变量具有相同的形状。插值结果存储在低斯分布的二维数据中。

该函数首先检查输入参数是否为空，然后检查输入参数的形状是否与目标形状相同。接着，函数根据输入的ith行和ne列索引，提取高斯分布中对应行和列的二维随机样本，然后对低斯分布的二维随机变量进行插值。插值完成后，函数将结果存储在低斯分布的二维数据中。

虽然该函数的名称中含有"sam"（sampling）一词，但实际上它并没有进行实际的采样操作，而是直接对低斯分布的二维随机变量进行插值计算。


```cpp
static void ggml_sam_sin(struct ggml_tensor * dst , const struct ggml_tensor * src, int ith, int nth, void * userdata) {
    GGML_ASSERT(userdata == NULL);
    GGML_ASSERT(ggml_are_same_shape(dst, src));
    GGML_ASSERT(ggml_is_contiguous(dst));
    GGML_ASSERT(ggml_is_contiguous(src));

    const float * src_data = ggml_get_data_f32(src);
    float * dst_data = ggml_get_data_f32(dst);

    const int ne = (int)ggml_nelements(dst);
    const int dr = (ne + nth - 1) / nth;
    const int ie0 = dr * ith;
    const int ie1 = std::min(ie0 + dr, ne);

    for (int i = ie0; i < ie1; ++i) {
        dst_data[i] = sinf(src_data[i]);
    }
}

```

这段代码定义了一个名为 "gggml_sam_cos" 的函数，属于 "gggml\_sam" 包。它的输入参数包括一个指向 "gggml\_tensor" 类型数据的指针 "dst"，一个指向 "gggml\_tensor" 类型数据的指针 "src"，以及两个整数 "ith" 和 "nth"，它们分别表示匹配索引和元素数量。函数的实现包括对输入数据进行相乘并取余数的操作，然后使用 cosine函数的 "f32" 函数计算结果，并将结果存储回目的数据指针上。

函数的参数包括一个指向 void 类型数据的指针 "userdata"，但userdata在函数体内部被声明为空，因此它的值在函数运行时并不会被使用。


```cpp
static void ggml_sam_cos(struct ggml_tensor * dst , const struct ggml_tensor * src, int ith, int nth, void * userdata) {
    GGML_ASSERT(userdata == NULL);
    GGML_ASSERT(ggml_are_same_shape(dst, src));
    GGML_ASSERT(ggml_is_contiguous(dst));
    GGML_ASSERT(ggml_is_contiguous(src));

    const float * src_data = ggml_get_data_f32(src);
    float * dst_data = ggml_get_data_f32(dst);

    const int ne = (int)ggml_nelements(dst);
    const int dr = (ne + nth - 1) / nth;
    const int ie0 = dr * ith;
    const int ie1 = std::min(ie0 + dr, ne);

    for (int i = ie0; i < ie1; ++i) {
        dst_data[i] = cosf(src_data[i]);
    }
}

```

这段代码是一个名为`sam_image_load_from_file`的函数，它从给定的文件名中加载一张RGB图像（红色、绿色和蓝色）并将其存储在名为`img`的变量中。

该函数首先通过调用`stbi_load`函数从文件中读取图像数据。`stbi_load`函数尝试从文件中读取数据，并返回一个指向内存的指针。如果读取失败，函数将打印错误消息并返回`false`。

如果`stbi_load`函数成功读取数据，函数将读取的图像的宽度和高度存储在`img`变量的`nx`和`ny`成员中，并将读取的图像数据存储在`img`变量的`data`成员中。

然后，函数通过调用`stbi_image_free`函数释放存储在`data`内存中的图像数据。

最后，函数返回`true`表示图像读取成功，否则返回`false`。


```cpp
bool sam_image_load_from_file(const std::string & fname, sam_image_u8 & img) {
    int nx, ny, nc;
    auto data = stbi_load(fname.c_str(), &nx, &ny, &nc, 3);
    if (!data) {
        fprintf(stderr, "%s: failed to load '%s'\n", __func__, fname.c_str());
        return false;
    }

    img.nx = nx;
    img.ny = ny;
    img.data.resize(nx * ny * 3);
    memcpy(img.data.data(), data, nx * ny * 3);

    stbi_image_free(data);

    return true;
}

```

This appears to be a simple script for interpolating a 2D image. The script takes into account the position of an object (x and y coordinates), the scale of the image, and the position of the object in the image. It then calculates the dx and dy of the object, and uses these calculations to calculate the new position of the object in the image. The script also uses a background image (res), which is a 3D array with the same dimensions as the 2D image. The script then interpolates the object with the background image and returns the result.

Please note that this script is not optimized and runs the risk of generating artifacts, especially when the image has a non-linear composition.


```cpp
// ref: https://github.com/facebookresearch/segment-anything/blob/efeab7296ab579d4a261e554eca80faf6b33924a/segment_anything/modeling/sam.py#L164
// resize largest dimension to 1024
// normalize: x = (x - mean) / std
//     mean = [123.675, 116.28, 103.53]
//     std  = [58.395, 57.12, 57.375]
//     TODO: why are these hardcoded !?
// pad to 1024x1024
// TODO: for some reason, this is not numerically identical to pytorch's interpolation
bool sam_image_preprocess(const sam_image_u8 & img, sam_image_f32 & res) {
    const int nx = img.nx;
    const int ny = img.ny;

    const int nx2 = 1024;
    const int ny2 = 1024;

    res.nx = nx2;
    res.ny = ny2;
    res.data.resize(3*nx2*ny2);

    const float scale = std::max(nx, ny) / 1024.0f;

    fprintf(stderr, "%s: scale = %f\n", __func__, scale);

    const int nx3 = int(nx/scale + 0.5f);
    const int ny3 = int(ny/scale + 0.5f);

    const float m3[3] = { 123.675f, 116.280f, 103.530f };
    const float s3[3] = {  58.395f,  57.120f,  57.375f };

    for (int y = 0; y < ny3; y++) {
        for (int x = 0; x < nx3; x++) {
            for (int c = 0; c < 3; c++) {
                // linear interpolation
                const float sx = (x + 0.5f)*scale - 0.5f;
                const float sy = (y + 0.5f)*scale - 0.5f;

                const int x0 = std::max(0, (int) std::floor(sx));
                const int y0 = std::max(0, (int) std::floor(sy));

                const int x1 = std::min(x0 + 1, nx - 1);
                const int y1 = std::min(y0 + 1, ny - 1);

                const float dx = sx - x0;
                const float dy = sy - y0;

                const int j00 = 3*(y0*nx + x0) + c;
                const int j01 = 3*(y0*nx + x1) + c;
                const int j10 = 3*(y1*nx + x0) + c;
                const int j11 = 3*(y1*nx + x1) + c;

                const float v00 = img.data[j00];
                const float v01 = img.data[j01];
                const float v10 = img.data[j10];
                const float v11 = img.data[j11];

                const float v0 = v00*(1.0f - dx) + v01*dx;
                const float v1 = v10*(1.0f - dx) + v11*dx;

                const float v = v0*(1.0f - dy) + v1*dy;

                const uint8_t v2 = std::min(std::max(std::round(v), 0.0f), 255.0f);

                const int i = 3*(y*nx3 + x) + c;

                res.data[i] = (float(v2) - m3[c]) / s3[c];
            }
        }
    }

    return true;
}

```

This is a function that reads a tensor from a model file and checks its size and content. It uses the information in the model file to determine the correct data type and size of the tensor, and then prints any errors and calculates the total size of the model. Here's a summary of what the function does:

1. It reads the tensor from the input model file.
2. It reads the header information from the beginning of the tensor.
3. It reads the data from the end of the tensor.
4. It checks the size of the tensor and the total size of the model.
5. It prints the error message if the tensor's size is incorrect or the model file is empty.
6. It prints a final message.

The function takes in two arguments: the model file and a pointer to a tensor data structure. It returns a boolean indicating whether the tensor was read successfully.


```cpp
// load the model's weights from a file
bool sam_model_load(const sam_params & params, sam_model & model) {
    fprintf(stderr, "%s: loading model from '%s' - please wait ...\n", __func__, params.model.c_str());

    auto fin = std::ifstream(params.model, std::ios::binary);
    if (!fin) {
        fprintf(stderr, "%s: failed to open '%s'\n", __func__, params.model.c_str());
        return false;
    }

    // verify magic
    {
        uint32_t magic;
        fin.read((char *) &magic, sizeof(magic));
        if (magic != 0x67676d6c) {
            fprintf(stderr, "%s: invalid model file '%s' (bad magic)\n", __func__, params.model.c_str());
            return false;
        }
    }

    // load hparams
    {
        // Override defaults with user choices
        model.hparams.mask_threshold            = params.mask_threshold;
        model.hparams.iou_threshold             = params.iou_threshold;
        model.hparams.stability_score_threshold = params.stability_score_threshold;
        model.hparams.stability_score_offset    = params.stability_score_offset;
        model.hparams.eps                       = params.eps;
        model.hparams.eps_decoder_transformer   = params.eps_decoder_transformer;

        auto & hparams = model.hparams;

        fin.read((char *) &hparams.n_enc_state,     sizeof(hparams.n_enc_state));
        fin.read((char *) &hparams.n_enc_layer,     sizeof(hparams.n_enc_layer));
        fin.read((char *) &hparams.n_enc_head,      sizeof(hparams.n_enc_head));
        fin.read((char *) &hparams.n_enc_out_chans, sizeof(hparams.n_enc_out_chans));
        fin.read((char *) &hparams.n_pt_embd,       sizeof(hparams.n_pt_embd));
        fin.read((char *) &hparams.ftype,           sizeof(hparams.ftype));

        const int32_t qntvr = hparams.ftype / GGML_QNT_VERSION_FACTOR;

        printf("%s: n_enc_state      = %d\n", __func__, hparams.n_enc_state);
        printf("%s: n_enc_layer      = %d\n", __func__, hparams.n_enc_layer);
        printf("%s: n_enc_head       = %d\n", __func__, hparams.n_enc_head);
        printf("%s: n_enc_out_chans  = %d\n", __func__, hparams.n_enc_out_chans);
        printf("%s: n_pt_embd        = %d\n", __func__, hparams.n_pt_embd);
        printf("%s: ftype            = %d\n", __func__, hparams.ftype);
        printf("%s: qntvr            = %d\n", __func__, qntvr);

        hparams.ftype %= GGML_QNT_VERSION_FACTOR;

    }

    // for the big tensors, we have the option to store the data in 16-bit floats or quantized
    // in order to save memory and also to speed up the computation
    ggml_type wtype = ggml_ftype_to_ggml_type((ggml_ftype) (model.hparams.ftype));
    if (wtype == GGML_TYPE_COUNT) {
        fprintf(stderr, "%s: invalid model file '%s' (bad ftype value %d)\n",
                __func__, params.model.c_str(), model.hparams.ftype);
        return false;
    }

    auto & ctx = model.ctx;

    const size_t ctx_size = [&]() {
        size_t ctx_size = 0;

        const auto & hparams = model.hparams;

        const int32_t n_enc_state     = hparams.n_enc_state;
        const int32_t n_enc_layer     = hparams.n_enc_layer;
        const int32_t n_enc_head_dim  = hparams.n_enc_head_dim();
        const int32_t n_enc_out_chans = hparams.n_enc_out_chans;
        const int32_t n_pt_embd       = hparams.n_pt_embd;

        const int32_t n_enc_layer_local  = hparams.global_attn_indices().size();
        const int32_t n_enc_layer_global = n_enc_layer - n_enc_layer_local;

        const int32_t n_img_embd    = hparams.n_img_embd();
        const int32_t n_window_size = hparams.n_window_size();
        const int32_t n_patch_size  = hparams.n_patch_size();

        // image encoder
        {
            ctx_size += n_enc_state*n_img_embd*n_img_embd*ggml_type_sizef(GGML_TYPE_F32);

            ctx_size += n_enc_state*3*n_patch_size*n_patch_size*ggml_type_sizef(GGML_TYPE_F16);
            ctx_size += n_enc_state*ggml_type_sizef(GGML_TYPE_F32);

            ctx_size +=     n_enc_state*n_enc_out_chans*1*1*ggml_type_sizef(GGML_TYPE_F16);
            ctx_size += n_enc_out_chans*n_enc_out_chans*3*3*ggml_type_sizef(GGML_TYPE_F16);

            ctx_size += n_enc_out_chans*ggml_type_sizef(GGML_TYPE_F32);
            ctx_size += n_enc_out_chans*ggml_type_sizef(GGML_TYPE_F32);

            ctx_size += n_enc_out_chans*ggml_type_sizef(GGML_TYPE_F32);
            ctx_size += n_enc_out_chans*ggml_type_sizef(GGML_TYPE_F32);
        }

        // image encoder layers
        {
            ctx_size += n_enc_layer*n_enc_state*ggml_type_sizef(GGML_TYPE_F32);
            ctx_size += n_enc_layer*n_enc_state*ggml_type_sizef(GGML_TYPE_F32);

            ctx_size += n_enc_layer_global*n_enc_head_dim*(2*n_img_embd - 1)*ggml_type_sizef(GGML_TYPE_F16);
            ctx_size += n_enc_layer_global*n_enc_head_dim*(2*n_img_embd - 1)*ggml_type_sizef(GGML_TYPE_F16);

            ctx_size += n_enc_layer_local*n_enc_head_dim*(2*n_window_size - 1)*ggml_type_sizef(GGML_TYPE_F16);
            ctx_size += n_enc_layer_local*n_enc_head_dim*(2*n_window_size - 1)*ggml_type_sizef(GGML_TYPE_F16);

            ctx_size += n_enc_layer*3*n_enc_state*n_enc_state*ggml_type_sizef(GGML_TYPE_F16);
            ctx_size += n_enc_layer*3*n_enc_state*            ggml_type_sizef(GGML_TYPE_F32);

            ctx_size += n_enc_layer*n_enc_state*n_enc_state*ggml_type_sizef(GGML_TYPE_F16);
            ctx_size += n_enc_layer*n_enc_state*            ggml_type_sizef(GGML_TYPE_F32);

            ctx_size += n_enc_layer*n_enc_state*ggml_type_sizef(GGML_TYPE_F32);
            ctx_size += n_enc_layer*n_enc_state*ggml_type_sizef(GGML_TYPE_F32);

            ctx_size += n_enc_layer*4*n_enc_state*n_enc_state*ggml_type_sizef(GGML_TYPE_F16);
            ctx_size += n_enc_layer*4*n_enc_state*            ggml_type_sizef(GGML_TYPE_F32);

            ctx_size += n_enc_layer*4*n_enc_state*n_enc_state*ggml_type_sizef(GGML_TYPE_F16);
            ctx_size += n_enc_layer*4*n_enc_state*            ggml_type_sizef(GGML_TYPE_F32);
        }

        ctx_size += (8 + 14*n_enc_layer)*ggml_tensor_overhead();

        // prompt encoder
        {
            ctx_size += n_enc_out_chans*ggml_type_sizef(GGML_TYPE_F16); // 2*(n_enc_out_chans/2)

            ctx_size += n_enc_out_chans*ggml_type_sizef(GGML_TYPE_F32);
            ctx_size += n_pt_embd*n_enc_out_chans*ggml_type_sizef(GGML_TYPE_F32);
        }

        ctx_size += (2 + n_pt_embd)*ggml_tensor_overhead();

        // mask decoder
        {
            //transformer
            {
                const int tfm_layers_count = 2;
                const int qkv_count = 3;
                const int norm_count = 4;
                const int n_hypernet_mpls_count = 4;

                // self_attn
                ctx_size += tfm_layers_count*qkv_count*n_enc_state*n_enc_state*ggml_type_sizef(GGML_TYPE_F16);
                ctx_size += tfm_layers_count*qkv_count*n_enc_state*            ggml_type_sizef(GGML_TYPE_F32);
                ctx_size += tfm_layers_count*n_enc_state*                      ggml_type_sizef(GGML_TYPE_F32);

                // all norms
                ctx_size += tfm_layers_count*norm_count*n_enc_state*ggml_type_sizef(GGML_TYPE_F32);
                ctx_size += tfm_layers_count*norm_count*n_enc_state*ggml_type_sizef(GGML_TYPE_F32);

                // cross_attn_token_to_img
                ctx_size += tfm_layers_count*qkv_count*n_enc_state*(n_enc_state/2)*ggml_type_sizef(GGML_TYPE_F16);
                ctx_size += tfm_layers_count*qkv_count*(n_enc_state/2)*            ggml_type_sizef(GGML_TYPE_F32);
                ctx_size += tfm_layers_count*n_enc_state*                          ggml_type_sizef(GGML_TYPE_F32);

                // mlp
                ctx_size += tfm_layers_count*8*n_enc_out_chans*n_enc_out_chans*ggml_type_sizef(GGML_TYPE_F16);
                ctx_size += tfm_layers_count*8*n_enc_out_chans*                ggml_type_sizef(GGML_TYPE_F32);
                ctx_size += tfm_layers_count*n_enc_out_chans*8*n_enc_out_chans*ggml_type_sizef(GGML_TYPE_F16);
                ctx_size += tfm_layers_count*n_enc_out_chans*                  ggml_type_sizef(GGML_TYPE_F32);

                // cross_attn_img_to_token
                ctx_size += tfm_layers_count*qkv_count*n_enc_state*(n_enc_state/2)*ggml_type_sizef(GGML_TYPE_F16);
                ctx_size += tfm_layers_count*qkv_count*(n_enc_state/2)*            ggml_type_sizef(GGML_TYPE_F32);
                ctx_size += tfm_layers_count*n_enc_state*                          ggml_type_sizef(GGML_TYPE_F32);

                // transformer_final_attn_token_to_img
                ctx_size += qkv_count*n_enc_state*(n_enc_state/2)*ggml_type_sizef(GGML_TYPE_F16);
                ctx_size += qkv_count*(n_enc_state/2)*            ggml_type_sizef(GGML_TYPE_F32);
                ctx_size += n_enc_state*                          ggml_type_sizef(GGML_TYPE_F32);

                // transformer_norm_final
                ctx_size += norm_count*n_enc_state*ggml_type_sizef(GGML_TYPE_F32);
                ctx_size += norm_count*n_enc_state*ggml_type_sizef(GGML_TYPE_F32);

                // output_upscaling
                ctx_size += n_enc_out_chans*n_img_embd*2*2*ggml_type_sizef(GGML_TYPE_F16);
                ctx_size += 3*n_img_embd*                  ggml_type_sizef(GGML_TYPE_F32);
                ctx_size += n_enc_out_chans*n_img_embd*(n_img_embd/2)*2*2*ggml_type_sizef(GGML_TYPE_F16);
                ctx_size += (n_img_embd/2)*                               ggml_type_sizef(GGML_TYPE_F32);

                // output_hypernetworks_mlps
                ctx_size += n_hypernet_mpls_count*2*n_enc_out_chans*n_enc_out_chans*ggml_type_sizef(GGML_TYPE_F16);
                ctx_size += n_hypernet_mpls_count*2*n_enc_out_chans*                ggml_type_sizef(GGML_TYPE_F32);
                ctx_size += n_hypernet_mpls_count*n_enc_out_chans*(n_img_embd/2)*ggml_type_sizef(GGML_TYPE_F16);
                ctx_size += n_hypernet_mpls_count*(n_img_embd/2)*                ggml_type_sizef(GGML_TYPE_F32);

                // iou_prediction_head
                ctx_size += 2*n_enc_out_chans*n_enc_out_chans*ggml_type_sizef(GGML_TYPE_F16);
                ctx_size += 2*n_enc_out_chans*                ggml_type_sizef(GGML_TYPE_F32);
                ctx_size += n_pt_embd*n_enc_out_chans*ggml_type_sizef(GGML_TYPE_F16);
                ctx_size += n_pt_embd*                ggml_type_sizef(GGML_TYPE_F32);

                // iou_token_w
                ctx_size += n_enc_out_chans*ggml_type_sizef(GGML_TYPE_F32);

                // mask_tokens_w
                ctx_size += n_pt_embd*n_enc_out_chans*ggml_type_sizef(GGML_TYPE_F32);
            }
        }
        fprintf(stderr, "%s: ggml ctx size = %6.2f MB\n", __func__, ctx_size/(1024.0*1024.0));

        return ctx_size;
    }();

    // create the ggml context
    {
        struct ggml_init_params params = {
            /*.mem_size   =*/ ctx_size,
            /*.mem_buffer =*/ NULL,
            /*.no_alloc   =*/ false,
        };

        ctx = ggml_init(params);
        if (!ctx) {
            fprintf(stderr, "%s: ggml_init() failed\n", __func__);
            return false;
        }
    }

    // prepare memory for the weights
    {
        const auto & hparams = model.hparams;

        const int32_t n_enc_state      = hparams.n_enc_state;
        const int32_t n_enc_layer      = hparams.n_enc_layer;
        const int32_t n_enc_head_dim   = hparams.n_enc_head_dim();
        const int32_t n_enc_out_chans  = hparams.n_enc_out_chans;
        const int32_t n_pt_embd        = hparams.n_pt_embd;

        const int32_t n_img_embd    = hparams.n_img_embd();
        const int32_t n_window_size = hparams.n_window_size();
        const int32_t n_patch_size  = hparams.n_patch_size();

        model.enc_img.layers.resize(n_enc_layer);

        // image encoder
        {
            auto & enc = model.enc_img;

            enc.pe = ggml_new_tensor_4d(ctx, GGML_TYPE_F32, n_enc_state, n_img_embd, n_img_embd, 1);

            enc.proj_w = ggml_new_tensor_4d(ctx, GGML_TYPE_F16, n_patch_size, n_patch_size,           3, n_enc_state);
            enc.proj_b = ggml_new_tensor_3d(ctx, GGML_TYPE_F32,            1,            1, n_enc_state);

            enc.neck_conv_0 = ggml_new_tensor_4d(ctx, GGML_TYPE_F16, 1, 1, n_enc_state,     n_enc_out_chans);
            enc.neck_conv_1 = ggml_new_tensor_4d(ctx, GGML_TYPE_F16, 3, 3, n_enc_out_chans, n_enc_out_chans);

            enc.neck_norm_0_w = ggml_new_tensor_1d(ctx, GGML_TYPE_F32, n_enc_out_chans);
            enc.neck_norm_0_b = ggml_new_tensor_1d(ctx, GGML_TYPE_F32, n_enc_out_chans);

            enc.neck_norm_1_w = ggml_new_tensor_1d(ctx, GGML_TYPE_F32, n_enc_out_chans);
            enc.neck_norm_1_b = ggml_new_tensor_1d(ctx, GGML_TYPE_F32, n_enc_out_chans);

            model.tensors["image_encoder.pos_embed"] = enc.pe;

            model.tensors["image_encoder.patch_embed.proj.weight"] = enc.proj_w;
            model.tensors["image_encoder.patch_embed.proj.bias"]   = enc.proj_b;

            model.tensors["image_encoder.neck.0.weight"] = enc.neck_conv_0;
            model.tensors["image_encoder.neck.2.weight"] = enc.neck_conv_1;

            model.tensors["image_encoder.neck.1.weight"] = enc.neck_norm_0_w;
            model.tensors["image_encoder.neck.1.bias"]   = enc.neck_norm_0_b;

            model.tensors["image_encoder.neck.3.weight"] = enc.neck_norm_1_w;
            model.tensors["image_encoder.neck.3.bias"]   = enc.neck_norm_1_b;

            for (int i = 0; i < n_enc_layer; ++i) {
                auto & layer = enc.layers[i];

                layer.norm1_w = ggml_new_tensor_1d(ctx, GGML_TYPE_F32, n_enc_state);
                layer.norm1_b = ggml_new_tensor_1d(ctx, GGML_TYPE_F32, n_enc_state);

                if (hparams.is_global_attn(i)) {
                    layer.rel_pos_w = ggml_new_tensor_2d(ctx, GGML_TYPE_F16, n_enc_head_dim, 2*n_img_embd - 1);
                    layer.rel_pos_h = ggml_new_tensor_2d(ctx, GGML_TYPE_F16, n_enc_head_dim, 2*n_img_embd - 1);
                } else {
                    layer.rel_pos_w = ggml_new_tensor_2d(ctx, GGML_TYPE_F16, n_enc_head_dim, 2*n_window_size - 1);
                    layer.rel_pos_h = ggml_new_tensor_2d(ctx, GGML_TYPE_F16, n_enc_head_dim, 2*n_window_size - 1);
                }

                layer.qkv_w = ggml_new_tensor_2d(ctx, GGML_TYPE_F16,   n_enc_state, 3*n_enc_state);
                layer.qkv_b = ggml_new_tensor_1d(ctx, GGML_TYPE_F32, 3*n_enc_state);

                layer.proj_w = ggml_new_tensor_2d(ctx, GGML_TYPE_F16,  n_enc_state,   n_enc_state);
                layer.proj_b = ggml_new_tensor_1d(ctx, GGML_TYPE_F32,  n_enc_state);

                layer.norm2_w = ggml_new_tensor_1d(ctx, GGML_TYPE_F32, n_enc_state);
                layer.norm2_b = ggml_new_tensor_1d(ctx, GGML_TYPE_F32, n_enc_state);

                layer.mlp_lin1_w = ggml_new_tensor_2d(ctx, GGML_TYPE_F16,   n_enc_state, 4*n_enc_state);
                layer.mlp_lin1_b = ggml_new_tensor_1d(ctx, GGML_TYPE_F32, 4*n_enc_state);

                layer.mlp_lin2_w = ggml_new_tensor_2d(ctx, GGML_TYPE_F16, 4*n_enc_state,   n_enc_state);
                layer.mlp_lin2_b = ggml_new_tensor_1d(ctx, GGML_TYPE_F32,   n_enc_state);

                model.tensors["image_encoder.blocks." + std::to_string(i) + ".norm1.weight"] = layer.norm1_w;
                model.tensors["image_encoder.blocks." + std::to_string(i) + ".norm1.bias"]   = layer.norm1_b;

                model.tensors["image_encoder.blocks." + std::to_string(i) + ".attn.rel_pos_w"] = layer.rel_pos_w;
                model.tensors["image_encoder.blocks." + std::to_string(i) + ".attn.rel_pos_h"] = layer.rel_pos_h;

                model.tensors["image_encoder.blocks." + std::to_string(i) + ".attn.qkv.weight"] = layer.qkv_w;
                model.tensors["image_encoder.blocks." + std::to_string(i) + ".attn.qkv.bias"]   = layer.qkv_b;

                model.tensors["image_encoder.blocks." + std::to_string(i) + ".attn.proj.weight"] = layer.proj_w;
                model.tensors["image_encoder.blocks." + std::to_string(i) + ".attn.proj.bias"]   = layer.proj_b;

                model.tensors["image_encoder.blocks." + std::to_string(i) + ".norm2.weight"] = layer.norm2_w;
                model.tensors["image_encoder.blocks." + std::to_string(i) + ".norm2.bias"]   = layer.norm2_b;

                model.tensors["image_encoder.blocks." + std::to_string(i) + ".mlp.lin1.weight"] = layer.mlp_lin1_w;
                model.tensors["image_encoder.blocks." + std::to_string(i) + ".mlp.lin1.bias"]   = layer.mlp_lin1_b;

                model.tensors["image_encoder.blocks." + std::to_string(i) + ".mlp.lin2.weight"] = layer.mlp_lin2_w;
                model.tensors["image_encoder.blocks." + std::to_string(i) + ".mlp.lin2.bias"]   = layer.mlp_lin2_b;
            }
        }

        // prompt encoder
        {
            auto & enc = model.enc_prompt;

            enc.pe = ggml_new_tensor_2d(ctx, GGML_TYPE_F32, n_enc_out_chans/2, 2);

            enc.not_a_pt_embd_w = ggml_new_tensor_1d(ctx, GGML_TYPE_F32, n_enc_out_chans);
            enc.no_mask_embd_w  = ggml_new_tensor_1d(ctx, GGML_TYPE_F32, n_enc_out_chans);

            model.tensors["prompt_encoder.pe_layer.positional_encoding_gaussian_matrix"] = enc.pe;
            model.tensors["prompt_encoder.not_a_point_embed.weight"] = enc.not_a_pt_embd_w;
            model.tensors["prompt_encoder.no_mask_embed.weight"]     = enc.no_mask_embd_w;

            enc.pt_embd.resize(n_pt_embd);
            for (int i = 0; i < n_pt_embd; i++) {
                enc.pt_embd[i] = ggml_new_tensor_1d(ctx, GGML_TYPE_F32, n_enc_out_chans);

                model.tensors["prompt_encoder.point_embeddings." + std::to_string(i) + ".weight"] = enc.pt_embd[i];
            }
        }

        // mask decoder
        {
            auto & dec = model.dec;
            auto & tfm_layers = dec.transformer_layers;

            const int tfm_layers_count = 2;
            tfm_layers.resize(tfm_layers_count);
            for (int i = 0; i < tfm_layers_count; ++i) {
                auto& l = tfm_layers[i];
                l.self_attn.q_w = ggml_new_tensor_2d(ctx, GGML_TYPE_F16, n_enc_out_chans, n_enc_out_chans);
                l.self_attn.q_b = ggml_new_tensor_1d(ctx, GGML_TYPE_F32, n_enc_out_chans);
                l.self_attn.k_w = ggml_new_tensor_2d(ctx, GGML_TYPE_F16, n_enc_out_chans, n_enc_out_chans);
                l.self_attn.k_b = ggml_new_tensor_1d(ctx, GGML_TYPE_F32, n_enc_out_chans);
                l.self_attn.v_w = ggml_new_tensor_2d(ctx, GGML_TYPE_F16, n_enc_out_chans, n_enc_out_chans);
                l.self_attn.v_b = ggml_new_tensor_1d(ctx, GGML_TYPE_F32, n_enc_out_chans);
                l.self_attn.out_w = ggml_new_tensor_2d(ctx, GGML_TYPE_F16, n_enc_out_chans, n_enc_out_chans);
                l.self_attn.out_b = ggml_new_tensor_1d(ctx, GGML_TYPE_F32, n_enc_out_chans);

                l.norm1_w = ggml_new_tensor_1d(ctx, GGML_TYPE_F32, n_enc_out_chans);
                l.norm1_b = ggml_new_tensor_1d(ctx, GGML_TYPE_F32, n_enc_out_chans);

                l.cross_attn_token_to_img.q_w = ggml_new_tensor_2d(ctx, GGML_TYPE_F16, n_enc_out_chans, n_enc_out_chans/2);
                l.cross_attn_token_to_img.q_b = ggml_new_tensor_1d(ctx, GGML_TYPE_F32, n_enc_out_chans/2);
                l.cross_attn_token_to_img.k_w = ggml_new_tensor_2d(ctx, GGML_TYPE_F16, n_enc_out_chans, n_enc_out_chans/2);
                l.cross_attn_token_to_img.k_b = ggml_new_tensor_1d(ctx, GGML_TYPE_F32, n_enc_out_chans/2);
                l.cross_attn_token_to_img.v_w = ggml_new_tensor_2d(ctx, GGML_TYPE_F16, n_enc_out_chans, n_enc_out_chans/2);
                l.cross_attn_token_to_img.v_b = ggml_new_tensor_1d(ctx, GGML_TYPE_F32, n_enc_out_chans/2);
                l.cross_attn_token_to_img.out_w = ggml_new_tensor_2d(ctx, GGML_TYPE_F16, n_enc_out_chans/2, n_enc_out_chans);
                l.cross_attn_token_to_img.out_b = ggml_new_tensor_1d(ctx, GGML_TYPE_F32, n_enc_out_chans);

                l.norm2_w = ggml_new_tensor_1d(ctx, GGML_TYPE_F32, n_enc_out_chans);
                l.norm2_b = ggml_new_tensor_1d(ctx, GGML_TYPE_F32, n_enc_out_chans);

                l.mlp_lin1_w = ggml_new_tensor_2d(ctx, GGML_TYPE_F16, n_enc_out_chans, 8*n_enc_out_chans);
                l.mlp_lin1_b = ggml_new_tensor_1d(ctx, GGML_TYPE_F32, 8*n_enc_out_chans);
                l.mlp_lin2_w = ggml_new_tensor_2d(ctx, GGML_TYPE_F16, 8*n_enc_out_chans, n_enc_out_chans);
                l.mlp_lin2_b = ggml_new_tensor_1d(ctx, GGML_TYPE_F32, n_enc_out_chans);

                l.norm3_w = ggml_new_tensor_1d(ctx, GGML_TYPE_F32, n_enc_out_chans);
                l.norm3_b = ggml_new_tensor_1d(ctx, GGML_TYPE_F32, n_enc_out_chans);

                l.norm4_w = ggml_new_tensor_1d(ctx, GGML_TYPE_F32, n_enc_out_chans);
                l.norm4_b = ggml_new_tensor_1d(ctx, GGML_TYPE_F32, n_enc_out_chans);

                l.cross_attn_img_to_token.q_w = ggml_new_tensor_2d(ctx, GGML_TYPE_F16, n_enc_out_chans, n_enc_out_chans/2);
                l.cross_attn_img_to_token.q_b = ggml_new_tensor_1d(ctx, GGML_TYPE_F32, n_enc_out_chans/2);
                l.cross_attn_img_to_token.k_w = ggml_new_tensor_2d(ctx, GGML_TYPE_F16, n_enc_out_chans, n_enc_out_chans/2);
                l.cross_attn_img_to_token.k_b = ggml_new_tensor_1d(ctx, GGML_TYPE_F32, n_enc_out_chans/2);
                l.cross_attn_img_to_token.v_w = ggml_new_tensor_2d(ctx, GGML_TYPE_F16, n_enc_out_chans, n_enc_out_chans/2);
                l.cross_attn_img_to_token.v_b = ggml_new_tensor_1d(ctx, GGML_TYPE_F32, n_enc_out_chans/2);
                l.cross_attn_img_to_token.out_w = ggml_new_tensor_2d(ctx, GGML_TYPE_F16, n_enc_out_chans/2, n_enc_out_chans);
                l.cross_attn_img_to_token.out_b = ggml_new_tensor_1d(ctx, GGML_TYPE_F32, n_enc_out_chans);

                const auto prefix = "mask_decoder.transformer.layers." + std::to_string(i) + ".";
                model.tensors[prefix + "self_attn.q_proj.weight"] = l.self_attn.q_w;
                model.tensors[prefix + "self_attn.q_proj.bias"]   = l.self_attn.q_b;
                model.tensors[prefix + "self_attn.k_proj.weight"] = l.self_attn.k_w;
                model.tensors[prefix + "self_attn.k_proj.bias"]   = l.self_attn.k_b;
                model.tensors[prefix + "self_attn.v_proj.weight"] = l.self_attn.v_w;
                model.tensors[prefix + "self_attn.v_proj.bias"]   = l.self_attn.v_b;
                model.tensors[prefix + "self_attn.out_proj.weight"] = l.self_attn.out_w;
                model.tensors[prefix + "self_attn.out_proj.bias"]   = l.self_attn.out_b;

                model.tensors[prefix + "norm1.weight"] = l.norm1_w;
                model.tensors[prefix + "norm1.bias"]   = l.norm1_b;

                model.tensors[prefix + "cross_attn_token_to_image.q_proj.weight"] = l.cross_attn_token_to_img.q_w;
                model.tensors[prefix + "cross_attn_token_to_image.q_proj.bias"]   = l.cross_attn_token_to_img.q_b;
                model.tensors[prefix + "cross_attn_token_to_image.k_proj.weight"] = l.cross_attn_token_to_img.k_w;
                model.tensors[prefix + "cross_attn_token_to_image.k_proj.bias"]   = l.cross_attn_token_to_img.k_b;
                model.tensors[prefix + "cross_attn_token_to_image.v_proj.weight"] = l.cross_attn_token_to_img.v_w;
                model.tensors[prefix + "cross_attn_token_to_image.v_proj.bias"]   = l.cross_attn_token_to_img.v_b;
                model.tensors[prefix + "cross_attn_token_to_image.out_proj.weight"] = l.cross_attn_token_to_img.out_w;
                model.tensors[prefix + "cross_attn_token_to_image.out_proj.bias"]   = l.cross_attn_token_to_img.out_b;

                model.tensors[prefix + "norm2.weight"] = l.norm2_w;
                model.tensors[prefix + "norm2.bias"]   = l.norm2_b;

                model.tensors[prefix + "mlp.lin1.weight"] = l.mlp_lin1_w;
                model.tensors[prefix + "mlp.lin1.bias"]   = l.mlp_lin1_b;
                model.tensors[prefix + "mlp.lin2.weight"] = l.mlp_lin2_w;
                model.tensors[prefix + "mlp.lin2.bias"]   = l.mlp_lin2_b;

                model.tensors[prefix + "norm3.weight"] = l.norm3_w;
                model.tensors[prefix + "norm3.bias"]   = l.norm3_b;
                model.tensors[prefix + "norm4.weight"] = l.norm4_w;
                model.tensors[prefix + "norm4.bias"]   = l.norm4_b;

                model.tensors[prefix + "cross_attn_image_to_token.q_proj.weight"] = l.cross_attn_img_to_token.q_w;
                model.tensors[prefix + "cross_attn_image_to_token.q_proj.bias"]   = l.cross_attn_img_to_token.q_b;
                model.tensors[prefix + "cross_attn_image_to_token.k_proj.weight"] = l.cross_attn_img_to_token.k_w;
                model.tensors[prefix + "cross_attn_image_to_token.k_proj.bias"]   = l.cross_attn_img_to_token.k_b;
                model.tensors[prefix + "cross_attn_image_to_token.v_proj.weight"] = l.cross_attn_img_to_token.v_w;
                model.tensors[prefix + "cross_attn_image_to_token.v_proj.bias"]   = l.cross_attn_img_to_token.v_b;
                model.tensors[prefix + "cross_attn_image_to_token.out_proj.weight"] = l.cross_attn_img_to_token.out_w;
                model.tensors[prefix + "cross_attn_image_to_token.out_proj.bias"]   = l.cross_attn_img_to_token.out_b;
            }

            dec.transformer_final_attn_token_to_img.q_w = ggml_new_tensor_2d(ctx, GGML_TYPE_F16, n_enc_out_chans, n_enc_out_chans/2);
            dec.transformer_final_attn_token_to_img.q_b = ggml_new_tensor_1d(ctx, GGML_TYPE_F32, n_enc_out_chans/2);
            dec.transformer_final_attn_token_to_img.k_w = ggml_new_tensor_2d(ctx, GGML_TYPE_F16, n_enc_out_chans, n_enc_out_chans/2);
            dec.transformer_final_attn_token_to_img.k_b = ggml_new_tensor_1d(ctx, GGML_TYPE_F32, n_enc_out_chans/2);
            dec.transformer_final_attn_token_to_img.v_w = ggml_new_tensor_2d(ctx, GGML_TYPE_F16, n_enc_out_chans, n_enc_out_chans/2);
            dec.transformer_final_attn_token_to_img.v_b = ggml_new_tensor_1d(ctx, GGML_TYPE_F32, n_enc_out_chans/2);
            dec.transformer_final_attn_token_to_img.out_w = ggml_new_tensor_2d(ctx, GGML_TYPE_F16, n_enc_out_chans/2, n_enc_out_chans);
            dec.transformer_final_attn_token_to_img.out_b = ggml_new_tensor_1d(ctx, GGML_TYPE_F32, n_enc_out_chans);

            model.tensors["mask_decoder.transformer.final_attn_token_to_image.q_proj.weight"] = dec.transformer_final_attn_token_to_img.q_w;
            model.tensors["mask_decoder.transformer.final_attn_token_to_image.q_proj.bias"]   = dec.transformer_final_attn_token_to_img.q_b;
            model.tensors["mask_decoder.transformer.final_attn_token_to_image.k_proj.weight"] = dec.transformer_final_attn_token_to_img.k_w;
            model.tensors["mask_decoder.transformer.final_attn_token_to_image.k_proj.bias"]   = dec.transformer_final_attn_token_to_img.k_b;
            model.tensors["mask_decoder.transformer.final_attn_token_to_image.v_proj.weight"] = dec.transformer_final_attn_token_to_img.v_w;
            model.tensors["mask_decoder.transformer.final_attn_token_to_image.v_proj.bias"]   = dec.transformer_final_attn_token_to_img.v_b;
            model.tensors["mask_decoder.transformer.final_attn_token_to_image.out_proj.weight"] = dec.transformer_final_attn_token_to_img.out_w;
            model.tensors["mask_decoder.transformer.final_attn_token_to_image.out_proj.bias"]   = dec.transformer_final_attn_token_to_img.out_b;

            dec.transformer_norm_final_w = ggml_new_tensor_1d(ctx, GGML_TYPE_F32, n_enc_out_chans);
            dec.transformer_norm_final_b = ggml_new_tensor_1d(ctx, GGML_TYPE_F32, n_enc_out_chans);

            model.tensors["mask_decoder.transformer.norm_final_attn.weight"] = dec.transformer_norm_final_w;
            model.tensors["mask_decoder.transformer.norm_final_attn.bias"]   = dec.transformer_norm_final_b;

            dec.output_upscaling_0_w = ggml_new_tensor_4d(ctx, GGML_TYPE_F16, 2, 2, n_img_embd, n_enc_out_chans);
            dec.output_upscaling_0_b = ggml_new_tensor_1d(ctx, GGML_TYPE_F32, n_img_embd);
            dec.output_upscaling_1_w = ggml_new_tensor_1d(ctx, GGML_TYPE_F32, n_img_embd);
            dec.output_upscaling_1_b = ggml_new_tensor_1d(ctx, GGML_TYPE_F32, n_img_embd);
            dec.output_upscaling_3_w = ggml_new_tensor_4d(ctx, GGML_TYPE_F16,  2, 2, n_img_embd/2, n_img_embd);
            dec.output_upscaling_3_b = ggml_new_tensor_1d(ctx, GGML_TYPE_F32, n_img_embd/2);

            model.tensors["mask_decoder.output_upscaling.0.weight"] = dec.output_upscaling_0_w;
            model.tensors["mask_decoder.output_upscaling.0.bias"]   = dec.output_upscaling_0_b;
            model.tensors["mask_decoder.output_upscaling.1.weight"] = dec.output_upscaling_1_w;
            model.tensors["mask_decoder.output_upscaling.1.bias"]   = dec.output_upscaling_1_b;
            model.tensors["mask_decoder.output_upscaling.3.weight"] = dec.output_upscaling_3_w;
            model.tensors["mask_decoder.output_upscaling.3.bias"]   = dec.output_upscaling_3_b;

            const int n_hypernet_mpls_count = 4;
            dec.output_hypernet_mlps.resize(n_hypernet_mpls_count);
            for (int i = 0; i < n_hypernet_mpls_count; ++i) {
                auto& mlp = dec.output_hypernet_mlps[i];

                mlp.w_0 = ggml_new_tensor_2d(ctx, GGML_TYPE_F16, n_enc_out_chans, n_enc_out_chans);
                mlp.b_0 = ggml_new_tensor_1d(ctx, GGML_TYPE_F32, n_enc_out_chans);
                mlp.w_1 = ggml_new_tensor_2d(ctx, GGML_TYPE_F16, n_enc_out_chans, n_enc_out_chans);
                mlp.b_1 = ggml_new_tensor_1d(ctx, GGML_TYPE_F32, n_enc_out_chans);
                mlp.w_2 = ggml_new_tensor_2d(ctx, GGML_TYPE_F16, n_enc_out_chans, n_img_embd/2);
                mlp.b_2 = ggml_new_tensor_1d(ctx, GGML_TYPE_F32, n_img_embd/2);

                const auto prefix = "mask_decoder.output_hypernetworks_mlps." + std::to_string(i) + ".";
                model.tensors[prefix + "layers.0.weight"] = mlp.w_0;
                model.tensors[prefix + "layers.0.bias"]   = mlp.b_0;
                model.tensors[prefix + "layers.1.weight"] = mlp.w_1;
                model.tensors[prefix + "layers.1.bias"]   = mlp.b_1;
                model.tensors[prefix + "layers.2.weight"] = mlp.w_2;
                model.tensors[prefix + "layers.2.bias"]   = mlp.b_2;
            }

            dec.iou_prediction_head_0_w = ggml_new_tensor_2d(ctx, GGML_TYPE_F16, n_enc_out_chans, n_enc_out_chans);
            dec.iou_prediction_head_0_b = ggml_new_tensor_1d(ctx, GGML_TYPE_F32, n_enc_out_chans);
            dec.iou_prediction_head_1_w = ggml_new_tensor_2d(ctx, GGML_TYPE_F16, n_enc_out_chans, n_enc_out_chans);
            dec.iou_prediction_head_1_b = ggml_new_tensor_1d(ctx, GGML_TYPE_F32, n_enc_out_chans);
            dec.iou_prediction_head_2_w = ggml_new_tensor_2d(ctx, GGML_TYPE_F16, n_enc_out_chans, n_pt_embd);
            dec.iou_prediction_head_2_b = ggml_new_tensor_1d(ctx, GGML_TYPE_F32, n_pt_embd);

            dec.iou_token_w = ggml_new_tensor_2d(ctx, GGML_TYPE_F32, n_enc_out_chans, 1);
            dec.mask_tokens_w = ggml_new_tensor_2d(ctx, GGML_TYPE_F32, n_enc_out_chans, n_pt_embd);

            model.tensors["mask_decoder.iou_prediction_head.layers.0.weight"] = dec.iou_prediction_head_0_w;
            model.tensors["mask_decoder.iou_prediction_head.layers.0.bias"]   = dec.iou_prediction_head_0_b;
            model.tensors["mask_decoder.iou_prediction_head.layers.1.weight"] = dec.iou_prediction_head_1_w;
            model.tensors["mask_decoder.iou_prediction_head.layers.1.bias"]   = dec.iou_prediction_head_1_b;
            model.tensors["mask_decoder.iou_prediction_head.layers.2.weight"] = dec.iou_prediction_head_2_w;
            model.tensors["mask_decoder.iou_prediction_head.layers.2.bias"]   = dec.iou_prediction_head_2_b;

            model.tensors["mask_decoder.iou_token.weight"] = dec.iou_token_w;
            model.tensors["mask_decoder.mask_tokens.weight"] = dec.mask_tokens_w;
        }
    }

    // load weights
    {
        int n_tensors = 0;
        size_t total_size = 0;

        fprintf(stderr, "%s: ", __func__);

        while (true) {
            int32_t n_dims;
            int32_t length;
            int32_t ftype;

            fin.read(reinterpret_cast<char *>(&n_dims), sizeof(n_dims));
            fin.read(reinterpret_cast<char *>(&length), sizeof(length));
            fin.read(reinterpret_cast<char *>(&ftype),  sizeof(ftype));

            if (fin.eof()) {
                break;
            }

            int64_t nelements = 1;
            int64_t ne[4] = { 1, 1, 1, 1 };
            for (int i = 0; i < n_dims; ++i) {
                int32_t ne_cur;
                fin.read(reinterpret_cast<char *>(&ne_cur), sizeof(ne_cur));
                ne[i] = ne_cur;
                nelements *= ne[i];
            }

            std::string name(length, 0);
            fin.read(&name[0], length);

            if (model.tensors.find(name.data()) == model.tensors.end()) {
                fprintf(stderr, "%s: unknown tensor '%s' in model file\n", __func__, name.data());
                return false;
            }

            auto tensor = model.tensors[name.data()];
            //printf("ne0 = %jd, ne1 = %jd, ne2 = %jd, ne3 = %jd\n", ne[0], ne[1], ne[2], ne[3]);

            if (ggml_nelements(tensor) != nelements) {
                fprintf(stderr, "%s: tensor '%s' has wrong size in model file: got %d, expected %d\n",
                        __func__, name.data(), (int) nelements, (int) ggml_nelements(tensor));
                return false;
            }

            if (tensor->ne[0] != ne[0] || tensor->ne[1] != ne[1] || tensor->ne[2] != ne[2] || tensor->ne[3] != ne[3]) {
                fprintf(stderr, "%s: tensor '%s' has wrong shape in model file: got [%d, %d, %d, %d], expected [%d, %d, %d, %d]\n",
                        __func__, name.data(),
                        (int) ne[0], (int) ne[1], (int) ne[2], (int) ne[3],
                        (int) tensor->ne[0], (int) tensor->ne[1], (int) tensor->ne[2], (int) tensor->ne[3]);
                return false;
            }

            size_t bpe = 0;

            switch (ftype) {
                case 0: bpe = ggml_type_size(GGML_TYPE_F32);  break;
                case 1: bpe = ggml_type_size(GGML_TYPE_F16);  break;
                case 2: bpe = ggml_type_size(GGML_TYPE_Q4_0); assert(ne[0] % 64 == 0); break;
                case 3: bpe = ggml_type_size(GGML_TYPE_Q4_1); assert(ne[0] % 64 == 0); break;
                default:
                        {
                            fprintf(stderr, "%s: unknown ftype %d in model file\n", __func__, ftype);
                            return false;
                        }
            };

            if ((nelements*bpe)/ggml_blck_size(tensor->type) != ggml_nbytes(tensor)) {
                fprintf(stderr, "%s: tensor '%s' has wrong size in model file: got %zu, expected %zu\n",
                        __func__, name.data(), ggml_nbytes(tensor), (size_t) nelements*bpe);
                return false;
            }

            fin.read(reinterpret_cast<char *>(tensor->data), ggml_nbytes(tensor));

            total_size += ggml_nbytes(tensor);
            if (++n_tensors % 8 == 0) {
                fprintf(stderr, ".");
                fflush(stdout);
            }
        }

        if (n_tensors != int(model.tensors.size())) {
            fprintf(stderr, "%s: model file has %d tensors, but %d tensors were expected\n", __func__, n_tensors, (int) model.tensors.size());
            return false;
        }

        fprintf(stderr, " done\n");

        fprintf(stderr, "%s: model size = %8.2f MB / num tensors = %d\n", __func__, total_size/1024.0/1024.0, n_tensors);
    }

    fin.close();

    return true;
}

```

This is a function definition for a piece of code that appears to be a scale-shifted version of a 3D protein structure in the PDB format, which is a file format for protein structures. The function is part of a model called "SEGMENT_ANYTHING," which is developed by Facebook Research.

The function takes a context (`ctx0`) and a Current status (`cur`), which is the 3D structure of the protein, and returns a dense 3D tensor representing the image. The input to the function is a顺铂值，这个值是训练得到的。函数使用ggml_scale函数实现对输入的顺铂值的缩放。

函数的主要步骤如下：

1. 将输入的当前顺铂值转换为包含两个0到顺铂值的单层顺铂值的顺铂值。
2. 对单层顺铂值应用cur的顺铂值，并使用ggml_new_f32函数创建一个新的f32类型的3D结构。
3. 对生成的3D结构应用forward_expand函数，并使用ggml_permute函数对数据进行排序。
4. 返回生成的3D结构。

函数的输入参数为顺铂值和Current结构，输出


```cpp
struct ggml_tensor * sam_fill_dense_pe(
            const sam_model   & model,
          struct ggml_context * ctx0,
          struct ggml_cgraph  * gf,
                  sam_state   & state) {
    const auto & hparams = model.hparams;
    const auto & enc     = model.enc_prompt;

    const int32_t n_img_embd = hparams.n_img_embd();
    const float n_img_embd_inv = 1.0f / n_img_embd;

    struct ggml_tensor * xy_embed_stacked = ggml_new_tensor_3d(ctx0, GGML_TYPE_F32, 2, n_img_embd, n_img_embd);
    ggml_allocr_alloc(state.allocr, xy_embed_stacked);

    if (!ggml_allocr_is_measure(state.allocr)) {
        float * data = (float *) ggml_get_data(xy_embed_stacked);
        for (int i = 0; i < n_img_embd; ++i) {
            const int row = 2*i*n_img_embd;
            const float y_val = 2 * (i + 0.5f) * n_img_embd_inv - 1;
            for (int j = 0; j < n_img_embd; ++j) {
                const float x_val = 2 * (j + 0.5f) * n_img_embd_inv - 1;
                data[row + 2*j + 0] = x_val;
                data[row + 2*j + 1] = y_val;
            }
        }
    }

    struct ggml_tensor * cur = ggml_mul_mat(ctx0, ggml_cont(ctx0, ggml_transpose(ctx0, enc.pe)), xy_embed_stacked);

    cur = ggml_scale(ctx0, cur, ggml_new_f32(ctx0, float(2.0*M_PI)));

    // concat
    // ref: https://github.com/facebookresearch/segment-anything/blob/main/segment_anything/modeling/prompt_encoder.py#L192
    {
        struct ggml_tensor * t_sin = ggml_map_custom1(ctx0, cur, ggml_sam_sin, GGML_N_TASKS_MAX, NULL);
        struct ggml_tensor * t_cos = ggml_map_custom1(ctx0, cur, ggml_sam_cos, GGML_N_TASKS_MAX, NULL);

        cur = ggml_new_tensor_3d(ctx0, GGML_TYPE_F32, t_sin->ne[0] + t_cos->ne[0], cur->ne[1], cur->ne[2]);

        ggml_build_forward_expand(gf, ggml_cpy(ctx0, t_sin, ggml_view_3d(ctx0, cur, t_sin->ne[0], t_sin->ne[1], t_sin->ne[2], cur->nb[1], cur->nb[2], 0)));
        ggml_build_forward_expand(gf, ggml_cpy(ctx0, t_cos, ggml_view_3d(ctx0, cur, t_sin->ne[0], t_sin->ne[1], t_sin->ne[2], cur->nb[1], cur->nb[2], t_sin->nb[1])));
    }

    struct ggml_tensor * pe_img_dense = ggml_cont(ctx0, ggml_permute(ctx0, cur, 2, 0, 1, 3));
    ggml_build_forward_expand(gf, pe_img_dense);

    return pe_img_dense;
}

```

这段代码定义了一个名为 `sam_layer_norm_2d` 的函数，它接受一个 `ggml_tensor` 的输入参数 layer，该参数是一个二维张量，通道数为 `n_channels`。函数的主要作用是对 layer 进行 LayerNorm2d 操作，并返回对 layer 的引用。

函数实现中，首先定义了一个名为 `layer_norm_2d` 的函数，该函数接受一个 `ggml_tensor` 类型的 layer，并对其进行 LayerNorm2d 操作。该函数的核心实现是先将 layer 中的值沿着 channel 维度展开，然后对每个 channel 执行 LayerNorm2d 操作。这个操作可以在 GPU 上执行，因此性能比 CPU 上的 LayerNorm2d 操作要快得多。

接着，在 `sam_layer_norm_2d` 函数中，对输入参数 layer 执行了两次 LayerNorm2d 操作。第一次将 layer 中的值沿着 channel 维度展开，然后对每个 channel 执行 LayerNorm2d 操作。第二次将 layer 中的值和乘以一个体重载因子 `w` 和偏移量 `b`，然后再次对每个 channel 执行 LayerNorm2d 操作。

最后，函数返回了 input layer。


```cpp
struct ggml_tensor* sam_layer_norm_2d(
                    struct ggml_context * ctx0,
                    struct ggml_tensor  * layer,
                    int                   n_channels,
                    struct ggml_tensor  * w,
                    struct ggml_tensor  * b,
                    float                 eps) {
    // LayerNorm2d
    // normalize along channel dimmension
    // TODO: better implementation
    layer = ggml_permute(ctx0,
                ggml_norm(ctx0, ggml_cont(ctx0, ggml_permute(ctx0, layer, 1, 2, 0, 3)), eps),
                2, 0, 1, 3);

    layer = ggml_add(ctx0,
              ggml_mul(ctx0,
                  ggml_repeat(ctx0, ggml_reshape_3d(ctx0, w, 1, 1, n_channels), layer),
                  layer),
              ggml_repeat(ctx0, ggml_reshape_3d(ctx0, b, 1, 1, n_channels), layer));

    return layer;
}

```

This is a function definition for an operation called "sam_layer_norm_2d" which performs batch normalization for a given layer. The operation takes a Context object, a layer object, and an image tensor as input and returns the normalized layer tensor.

The operation performs a fully connected and a projection operation before applying the batch normalization. The batch normalization is performed using the GELU activation function. The layer normalization is applied after the fully connected operation using the state.embd_img tensor.

The function should be added to the graph that the model is built upon.


```cpp
struct ggml_cgraph  * sam_encode_image(
            const sam_model & model,
                  sam_state & state,
        const sam_image_f32 & img) {

    const auto & hparams = model.hparams;
    const auto & enc     = model.enc_img;

    const int32_t n_enc_state     = hparams.n_enc_state;
    const int32_t n_enc_layer     = hparams.n_enc_layer;
    const int32_t n_enc_head      = hparams.n_enc_head;
    const int32_t n_enc_head_dim  = hparams.n_enc_head_dim();
    const int32_t n_enc_out_chans = hparams.n_enc_out_chans;
    const int32_t n_img_size    = hparams.n_img_size();
    const int32_t n_window_size = hparams.n_window_size();

    struct ggml_init_params ggml_params = {
        /*.mem_size   =*/ state.buf_compute_img_enc.size(),
        /*.mem_buffer =*/ state.buf_compute_img_enc.data(),
        /*.no_alloc   =*/ true, // skip allocating as we use ggml_alloc to allocate exact memory requirements
    };

    struct ggml_context * ctx0   = ggml_init(ggml_params);
    struct ggml_cgraph  * gf     = ggml_new_graph(ctx0);

    struct ggml_tensor * inp = ggml_new_tensor_4d(ctx0, GGML_TYPE_F32, n_img_size, n_img_size, 3, 1);
    ggml_allocr_alloc(state.allocr, inp);
    if (!ggml_allocr_is_measure(state.allocr)) {
        float * data = (float *) ggml_get_data(inp);

        const int nx = img.nx;
        const int ny = img.ny;
        const int n  = nx*ny;

        GGML_ASSERT(nx == n_img_size && ny == n_img_size);

        for (int k = 0; k < 3; k++) {
            for (int y = 0; y < ny; y++) {
                for (int x = 0; x < nx; x++) {
                    data[k*n + y*nx + x] = img.data[3*(y*nx + x) + k];
                }
            }
        }
    }

    // ref: https://github.com/facebookresearch/segment-anything/blob/main/segment_anything/modeling/image_encoder.py#L392
    struct ggml_tensor * cur = ggml_conv_2d_sk_p0(ctx0, enc.proj_w, inp);
    cur = ggml_add_inplace(ctx0,
            cur,
            ggml_repeat(ctx0, enc.proj_b, cur));

    // ref: https://github.com/facebookresearch/segment-anything/blob/main/segment_anything/modeling/image_encoder.py#L394
    // keep in F32
    cur = ggml_cont(ctx0,
            ggml_permute(ctx0, cur, 1, 2, 0, 3));

    // convert to F16
    //cur = ggml_cpy(ctx0,
    //        ggml_permute(ctx0, cur, 1, 2, 0, 3),
    //        ggml_new_tensor_3d(ctx0, GGML_TYPE_F16, n_enc_state, n_img_embd, n_img_embd));

    // ref: https://github.com/facebookresearch/segment-anything/blob/main/segment_anything/modeling/image_encoder.py#L108-L109
    cur = ggml_add_inplace(ctx0, cur, enc.pe);

    struct ggml_tensor * inpL = cur;

    for (int il = 0; il < n_enc_layer; ++il) {
        const auto & layer = enc.layers[il];

        // norm
        // ref: https://github.com/facebookresearch/segment-anything/blob/main/segment_anything/modeling/image_encoder.py#L168
        {
            cur = ggml_norm(ctx0, inpL, hparams.eps);

            // cur = ln_0_w*cur + ln_0_b
            cur = ggml_mul(ctx0, cur, layer.norm1_w);
            cur = ggml_add_inplace(ctx0, cur, layer.norm1_b);
        }

        const int64_t w0 = cur->ne[1];
        const int64_t h0 = cur->ne[2];

        if (hparams.is_global_attn(il) == false) {
            // local attention layer - apply window partition
            // ref: https://github.com/facebookresearch/segment-anything/blob/main/segment_anything/modeling/image_encoder.py#L169-L172
            cur = ggml_win_part(ctx0, cur, n_window_size);
        }

        const int64_t W = cur->ne[1];
        const int64_t H = cur->ne[2];

        // self-attention
        {
            cur = ggml_mul_mat(ctx0, layer.qkv_w, cur);
            cur = ggml_add_inplace(ctx0, cur, layer.qkv_b);

            // split qkv into separate tensors
            // ref: https://github.com/facebookresearch/segment-anything/blob/main/segment_anything/modeling/image_encoder.py#L225-L229
            const int B = cur->ne[3];

            cur = ggml_reshape_4d(ctx0, cur, n_enc_state, 3, W*H, B);
            cur = ggml_cont(ctx0, ggml_permute(ctx0, cur, 0, 3, 1, 2));

            struct ggml_tensor * Q;
            struct ggml_tensor * K;
            struct ggml_tensor * V;

            Q = ggml_view_3d   (ctx0, cur, n_enc_state, W*H, B, cur->nb[1], cur->nb[2], 0*cur->nb[3]);
            Q = ggml_reshape_4d(ctx0, Q,   n_enc_head_dim, n_enc_head, W*H, B);
            Q = ggml_cont      (ctx0, ggml_permute(ctx0, Q, 0, 2, 1, 3));
            Q = ggml_reshape_3d(ctx0, Q,   n_enc_head_dim, W*H, B*n_enc_head);

            K = ggml_view_3d   (ctx0, cur, n_enc_state, W*H, B, cur->nb[1], cur->nb[2], 1*cur->nb[3]);
            K = ggml_reshape_4d(ctx0, K,   n_enc_head_dim, n_enc_head, W*H, B);
            K = ggml_cont      (ctx0, ggml_permute(ctx0, K, 0, 2, 1, 3));
            K = ggml_reshape_3d(ctx0, K,   n_enc_head_dim, W*H, B*n_enc_head);

            V = ggml_view_3d   (ctx0, cur, n_enc_state, W*H, B, cur->nb[1], cur->nb[2], 2*cur->nb[3]);
            V = ggml_reshape_4d(ctx0, V,   n_enc_head_dim, n_enc_head, W*H, B);
            V = ggml_cont      (ctx0, ggml_permute(ctx0, V, 1, 2, 0, 3)); // transposed
            V = ggml_reshape_3d(ctx0, V,   W*H, n_enc_head_dim, B*n_enc_head);

            struct ggml_tensor * KQ = ggml_mul_mat(ctx0, K, Q);

            struct ggml_tensor * KQ_scaled =
                ggml_scale_inplace(ctx0,
                        KQ,
                        ggml_new_f32(ctx0, 1.0f/sqrtf(n_enc_head_dim))
                        );

            struct ggml_tensor * rw = ggml_get_rel_pos(ctx0, layer.rel_pos_w, W, W);
            struct ggml_tensor * rh = ggml_get_rel_pos(ctx0, layer.rel_pos_h, H, H);

            struct ggml_tensor * q_r = ggml_reshape_4d(ctx0, Q, n_enc_head_dim, W, H, B*n_enc_head);

            struct ggml_tensor * rel_w = ggml_cont(ctx0, ggml_permute(ctx0,
                        ggml_mul_mat(ctx0,
                            rw,
                            ggml_cont(ctx0, ggml_permute(ctx0, q_r, 0, 2, 1, 3))),
                        0, 2, 1, 3));
            struct ggml_tensor * rel_h = ggml_mul_mat(ctx0, rh, q_r);

            struct ggml_tensor * attn = ggml_add_rel_pos_inplace(ctx0, KQ_scaled, rel_w, rel_h);

            struct ggml_tensor * KQ_soft_max = ggml_soft_max_inplace(ctx0, attn);

            struct ggml_tensor * KQV = ggml_mul_mat(ctx0, V, KQ_soft_max);

            cur =
                ggml_reshape_4d(ctx0,
                        ggml_cont(ctx0,
                            ggml_permute(ctx0,
                                ggml_reshape_4d(ctx0, KQV, n_enc_head_dim, W*H, n_enc_head, B),
                                0, 2, 1, 3)),
                        n_enc_state, W, H, B);

            cur = ggml_mul_mat(ctx0, layer.proj_w, cur);
            cur = ggml_add_inplace(ctx0, cur, layer.proj_b);
        }

        if (hparams.is_global_attn(il) == false) {
            // local attention layer - reverse window partition
            cur = ggml_win_unpart(ctx0, cur, w0, h0, n_window_size);
        }

        cur = ggml_add_inplace(ctx0, cur, inpL);

        struct ggml_tensor * inpFF = cur;

        // feed-forward network
        {
            // norm
            {
                cur = ggml_norm(ctx0, inpFF, hparams.eps);

                // cur = mlp_ln_w*cur + mlp_ln_b
                cur = ggml_mul(ctx0, cur, layer.norm2_w);
                cur = ggml_add_inplace(ctx0, cur, layer.norm2_b);
            }

            // fully connected
            cur = ggml_mul_mat(ctx0, layer.mlp_lin1_w, cur);
            cur = ggml_add_inplace(ctx0, cur, layer.mlp_lin1_b);

            // GELU activation
            cur = ggml_gelu(ctx0, cur);

            // projection
            cur = ggml_mul_mat(ctx0, layer.mlp_lin2_w, cur);
            cur = ggml_add_inplace(ctx0, cur, layer.mlp_lin2_b);
        }

        inpL = ggml_add(ctx0, cur, inpFF);
    }

    cur = ggml_cont(ctx0, ggml_permute(ctx0, inpL, 2, 0, 1, 3));

    cur = ggml_conv_2d_sk_p0(ctx0, enc.neck_conv_0, cur);

    cur = sam_layer_norm_2d(ctx0, cur, n_enc_out_chans, enc.neck_norm_0_w, enc.neck_norm_0_b, hparams.eps);

    cur = ggml_conv_2d_s1_ph(ctx0, enc.neck_conv_1, cur);

    cur = sam_layer_norm_2d(ctx0, cur, n_enc_out_chans, enc.neck_norm_1_w, enc.neck_norm_1_b, hparams.eps);

    cur = ggml_cpy(ctx0, cur, state.embd_img);

    ggml_build_forward_expand(gf, cur);
    ggml_disconnect_node_from_graph(state.embd_img);

    //ggml_graph_print(&gf);

    ggml_free(ctx0);

    return gf;
}


```

This is a C++ implementation of the Prompt Encoder model from the Segment Anything pre-trained framework. The Prompt Encoder model takes an input sequence of fixed length `seq_len` and an integer classification label `label`, and outputs a sparse vector of embeddings in the feature space of the input text.

The input sequence is first expanded to a two-dimensional tensor of size `(seq_len, label)` by adding a special "classification" embedding, which has a dimension of size `(1, label)`. This "classification" embedding is then expanded along the second dimension to create a tensor of size `(seq_len, label)`.

The attention mechanism is then applied to this tensor by taking a bbox attention that computes the attention weights based on the spatial relationships between the input embeddings. The attention weights are then used to weight the input embeddings to compute the output of the model.

Finally, the output of the model is returned as a sparse vector of embeddings in the feature space of the input text.


```cpp
struct prompt_encoder_result {
    struct ggml_tensor * embd_prompt_sparse = {};
    struct ggml_tensor * embd_prompt_dense = {};
};

// encode a prompt
//
// - points
// - boxes
// - masks
//
// TODO: currently just encode a single point for simplicity
//
prompt_encoder_result sam_encode_prompt(
        const sam_model     & model,
        struct ggml_context * ctx0,
        struct ggml_cgraph  * gf,
                  sam_state & state,
                        int   nx,
                        int   ny,
                  sam_point   point) {

    const auto & hparams = model.hparams;
    const auto & enc = model.enc_prompt;

    // transform points
    // ref: https://github.com/facebookresearch/segment-anything/blob/main/segment_anything/automatic_mask_generator.py#L276
    {
        const int nmax = std::max(nx, ny);

        const float scale = hparams.n_img_size() / (float) nmax;

        const int nx_new = int(nx*scale + 0.5f);
        const int ny_new = int(ny*scale + 0.5f);

        point.x = point.x*(float(nx_new)/nx) + 0.5f;
        point.y = point.y*(float(ny_new)/ny) + 0.5f;
    }

    struct ggml_tensor * inp = ggml_new_tensor_2d(ctx0, GGML_TYPE_F32, 2, 2);

    ggml_allocr_alloc(state.allocr, inp);
    if (!ggml_allocr_is_measure(state.allocr)) {
        // set the input by converting the [0, 1] coordinates to [-1, 1]
        float * data = (float *) inp->data;

        data[0] = 2.0f*(point.x / hparams.n_img_size()) - 1.0f;
        data[1] = 2.0f*(point.y / hparams.n_img_size()) - 1.0f;

        // padding
        // ref: https://github.com/facebookresearch/segment-anything/blob/main/segment_anything/modeling/prompt_encoder.py#L81-L85
        data[2] = 2.0f*(0.0f) - 1.0f;
        data[3] = 2.0f*(0.0f) - 1.0f;
    }

    struct ggml_tensor * cur = ggml_mul_mat(ctx0, ggml_cont(ctx0, ggml_transpose(ctx0, enc.pe)), inp);

    cur = ggml_scale(ctx0, cur, ggml_new_f32(ctx0, float(2.0*M_PI)));

    // concat
    // ref: https://github.com/facebookresearch/segment-anything/blob/main/segment_anything/modeling/prompt_encoder.py#L192
    {
        struct ggml_tensor * t_sin = ggml_map_custom1(ctx0, cur, ggml_sam_sin, GGML_N_TASKS_MAX, NULL);
        struct ggml_tensor * t_cos = ggml_map_custom1(ctx0, cur, ggml_sam_cos, GGML_N_TASKS_MAX, NULL);

        cur = ggml_new_tensor_2d(ctx0, GGML_TYPE_F32, t_sin->ne[0] + t_cos->ne[0], cur->ne[1]);

        ggml_build_forward_expand(gf, ggml_cpy(ctx0, t_sin, ggml_view_2d(ctx0, cur, t_sin->ne[0], t_sin->ne[1], cur->nb[1], 0)));
        ggml_build_forward_expand(gf, ggml_cpy(ctx0, t_cos, ggml_view_2d(ctx0, cur, t_sin->ne[0], t_sin->ne[1], cur->nb[1], t_sin->nb[1])));

        // overwrite label == -1 with not_a_point_embed.weight
        // ref: https://github.com/facebookresearch/segment-anything/blob/main/segment_anything/modeling/prompt_encoder.py#L86
        // TODO: extend for multiple points
        ggml_build_forward_expand(gf, ggml_cpy(ctx0, enc.not_a_pt_embd_w, ggml_view_2d(ctx0, cur, cur->ne[0], 1, cur->nb[1], cur->nb[1])));
    }

    // add point_embeddings[1] to label == 1
    // ref: https://github.com/facebookresearch/segment-anything/blob/main/segment_anything/modeling/prompt_encoder.py#L90
    struct ggml_tensor * v = ggml_view_2d(ctx0, cur, cur->ne[0], 1, cur->nb[1], 0);
    ggml_build_forward_expand(gf, ggml_cpy(ctx0, ggml_add_inplace(ctx0, v, enc.pt_embd[1]), v));

    struct ggml_tensor * embd_prompt_sparse = cur;
    ggml_build_forward_expand(gf, embd_prompt_sparse);

    struct ggml_tensor * embd_prompt_dense = ggml_repeat(ctx0,
            ggml_cont(ctx0,
                ggml_view_3d(ctx0, enc.no_mask_embd_w,
                    1, 1, enc.no_mask_embd_w->ne[0], enc.no_mask_embd_w->nb[0], enc.no_mask_embd_w->nb[0], 0)),
            ggml_new_tensor_3d(ctx0, GGML_TYPE_F32, hparams.n_img_embd(), hparams.n_img_embd(), hparams.n_enc_out_chans));

    ggml_build_forward_expand(gf, embd_prompt_dense);

    //printf("used_mem = %zu\n", ggml_used_mem(ctx0));

    prompt_encoder_result res;
    res.embd_prompt_sparse = embd_prompt_sparse;
    res.embd_prompt_dense  = embd_prompt_dense;
    return res;
}

```

This function appears to calculate the cross-entropy loss for a given input tensor `Q`, which has shape `(batch_size, sequence_length, num_classes)`. It does this by first normalizing the input tensor `Q` using the function `ggml_norm_mean_var` in the `ggml` package, which computes the mean and variance of each element in the tensor and normalizes the result to be between 0 and 1.

Next, the input tensor `Q` is converted to a format that can be used by the neural network, such as a tensor of floating point numbers with shape `(seq_length, batch_size, num_classes)`. This is done by splitting the tensor `Q` along the sequence dimension and casting the resulting slice to be a tensor.

The function then computes the soft logarithm of the input tensor `Q`, which is a commonly used activation function in deep learning. The soft logarithm is computed using the `ggml_soft_log` function in the `ggml` package.

The function then computes the element-wise product of the input tensor `Q` with the tensor `K` (which has shape `(num_classes, num_classes)`), and the resulting tensor is then passed through a function `ggml_permute` to obtain a tensor of shape `(num_classes,`.

Finally, the function computes the cross-entropy loss for the input tensor `Q` using the `ggml_cont` function in the `ggml` package, which computes the element-wise product of a tensor with a vector and returns the result. The cross-entropy loss is calculated by summing the elements of the negative log probability distribution of `Q` with respect to the true labels of the input tensor `Q`.

The function returns the resulting tensor `KQV`, which has shape `(batch_size, sequence_length, num_classes)`.


```cpp
struct ggml_tensor* sam_decode_mask_transformer_attn(
    const sam_layer_dec_transformer_attn & attn,
                      struct ggml_tensor * queries,
                      struct ggml_tensor * keys,
                      struct ggml_tensor * values,
                     struct ggml_context * ctx0,
                         const sam_model & model) {
    const auto & hparams = model.hparams;
    const int n_head = hparams.n_dec_heads;

    struct ggml_tensor * Qcur = {};
    struct ggml_tensor * Kcur = {};
    struct ggml_tensor * Vcur = {};

    Qcur = ggml_mul_mat(ctx0, attn.q_w, queries);
    Qcur = ggml_add_inplace(ctx0, Qcur, attn.q_b);

    Kcur = ggml_mul_mat(ctx0, attn.k_w, keys);
    Kcur = ggml_add_inplace(ctx0, Kcur, attn.k_b);

    Vcur = ggml_mul_mat(ctx0, attn.v_w, values);
    Vcur = ggml_add_inplace(ctx0, Vcur, attn.v_b);

    struct ggml_tensor * Q = {};
    struct ggml_tensor * K = {};
    struct ggml_tensor * V = {};

    Q = ggml_reshape_4d(ctx0, Qcur, Qcur->ne[0]/n_head, n_head, Qcur->ne[1], Qcur->ne[2]);
    Q = ggml_cont(ctx0, ggml_permute(ctx0, Q, 0, 2, 1, 3));

    K = ggml_reshape_4d(ctx0, Kcur, Kcur->ne[0]/n_head, n_head, Kcur->ne[1], Kcur->ne[2]);
    K = ggml_cont(ctx0, ggml_permute(ctx0, K, 0, 2, 1, 3));

    V = ggml_reshape_4d(ctx0, Vcur, Vcur->ne[0]/n_head, n_head, Vcur->ne[1], Vcur->ne[2]);
    V = ggml_cont(ctx0, ggml_permute(ctx0, V, 0, 2, 1, 3));

    // Q * K
    struct ggml_tensor * KQ = ggml_mul_mat(ctx0, K, Q);

    struct ggml_tensor * KQ_scaled =
        ggml_scale_inplace(ctx0,
                KQ,
                ggml_new_f32(ctx0, 1.0f/sqrt(float(Q->ne[0]))));

    struct ggml_tensor * KQ_soft_max = ggml_soft_max_inplace(ctx0, KQ_scaled);

    struct ggml_tensor * KQV = ggml_mul_mat(ctx0, KQ_soft_max, ggml_cont(ctx0, ggml_transpose(ctx0, V)));

    struct ggml_tensor * KQV_merged = ggml_cont(ctx0, ggml_transpose(ctx0, KQV));
    KQV_merged = ggml_cont(ctx0, ggml_permute(ctx0, KQV_merged, 0, 2, 1, 3));
    KQV_merged = ggml_reshape_3d(ctx0, KQV_merged, KQV_merged->ne[0]*KQV_merged->ne[1], KQV_merged->ne[2], KQV_merged->ne[3]);
    KQV_merged = ggml_mul_mat(ctx0, attn.out_w, KQV_merged);
    KQV_merged = ggml_add_inplace(ctx0, KQV_merged, attn.out_b);

    return KQV_merged;
}

```

这是一段C语言代码，定义了一个名为“sam_decode_mask_mlp_relu_3”的结构体函数，它的输入参数为：

- 第一个输入张量“in”，它的类型为“ggml_tensor”结构体，大小未知，未定义；
- 第二个输入张量“w_0”，它的类型为“ggml_tensor”结构体，大小未知，未定义；
- 第三个输入张量“b_0”，它的类型为“ggml_tensor”结构体，大小未知，未定义；
- 第四个输入张量“w_1”，它的类型为“ggml_tensor”结构体，大小未知，未定义；
- 第五个输入张量“b_1”，它的类型为“ggml_tensor”结构体，大小未知，未定义；
- 第六个输入张量“w_2”，它的类型为“ggml_tensor”结构体，大小未知，未定义；
- 第七个输入张量“b_2”，它的类型为“ggml_tensor”结构体，大小未知，未定义；
- 最后，还有一个输出张量“out”，它的类型为“ggml_tensor”结构体，输出于函数内部，未定义。

函数实现中，首先定义了一个名为“cur”的结构体变量，它的类型为“ggml_tensor”结构体，未定义；

然后，在函数实现中，对输入张量“in”进行了“ggml_mul”和“ggml_add_inplace”操作，得到了一个中间结果张量“cur”；

接着，对中间结果张量“cur”应用了“ggml_relu”运算，得到了一个最终结果张量“out”。


```cpp
struct ggml_tensor * sam_decode_mask_mlp_relu_3(
     struct ggml_tensor * in,
     struct ggml_tensor * w_0,
     struct ggml_tensor * b_0,
     struct ggml_tensor * w_1,
     struct ggml_tensor * b_1,
     struct ggml_tensor * w_2,
     struct ggml_tensor * b_2,
    struct ggml_context * ctx0) {

    struct ggml_tensor * cur = {};
    cur = ggml_mul_mat(ctx0, w_0, in);
    cur = ggml_add_inplace(ctx0, cur, b_0);

    cur = ggml_relu_inplace(ctx0, cur);

    cur = ggml_mul_mat(ctx0, w_1, cur);
    cur = ggml_add_inplace(ctx0, cur, b_1);

    cur = ggml_relu_inplace(ctx0, cur);

    cur = ggml_mul_mat(ctx0, w_2, cur);
    cur = ggml_add_inplace(ctx0, cur, b_2);

    return cur;
}

```

This is a function definition in the `mask_decoder.py` file of the `segment_anything` model that uses a mask localization and prediction module (MLP) with a Multi-Level Prediction (MLP) and a mask decoder. The MLP is defined using the `sam_decode_mask_mlp_relu_3` function, which takes the input `iou_pred` and outputs a binary mask indicating which object is the foreground. The mask decoder is then applied to produce the final segmentation mask.

The function takes two arguments: `dec` and `ctx0`. `dec` is a decoder object that has been initialized with the `sam_decode_mask_mlp_relu_3` function, which takes the input `iou_pred` and outputs a binary mask indicating which object is the foreground. `ctx0` is an integer that is passed to the `ggml_view_1d`, `ggml_view_4d`, and `ggml_cpy` functions to access the input and output data structures of the decoder.

The function returns `True` if the mask decoder produces a valid segmentation mask, and `False` otherwise.


```cpp
bool sam_decode_mask(
                    const sam_model & model,
        const prompt_encoder_result & prompt,
                 struct ggml_tensor * pe_img,
                struct ggml_context * ctx0,
                struct ggml_cgraph  * gf,
                          sam_state & state) {

    const auto & hparams = model.hparams;
    const auto & dec = model.dec;
    const int n_img_embd = hparams.n_img_embd();

    struct ggml_tensor * tokens = {};
    {
        // Concatenate output tokens
        // ref: https://github.com/facebookresearch/segment-anything/blob/6fdee8f2727f4506cfbbe553e23b895e27956588/segment_anything/modeling/mask_decoder.py#L120
        const auto& sparse = prompt.embd_prompt_sparse;

        tokens = ggml_new_tensor_3d(ctx0, GGML_TYPE_F32, dec.iou_token_w->ne[0], dec.iou_token_w->ne[1] + dec.mask_tokens_w->ne[1] + sparse->ne[1], sparse->ne[2]);

        const size_t offsets[3] = { 0, dec.iou_token_w->ne[1]*tokens->nb[1], dec.iou_token_w->ne[1]*tokens->nb[1] + dec.mask_tokens_w->ne[1]*tokens->nb[1] };
        ggml_build_forward_expand(gf, ggml_cpy(ctx0, dec.iou_token_w,   ggml_view_2d(ctx0, tokens, tokens->ne[0], dec.iou_token_w->ne[1],   tokens->nb[1], offsets[0])));
        ggml_build_forward_expand(gf, ggml_cpy(ctx0, dec.mask_tokens_w, ggml_view_2d(ctx0, tokens, tokens->ne[0], dec.mask_tokens_w->ne[1], tokens->nb[1], offsets[1])));
        ggml_build_forward_expand(gf, ggml_cpy(ctx0, sparse,            ggml_view_2d(ctx0, tokens, tokens->ne[0], sparse->ne[1],            tokens->nb[1], offsets[2])));
        // TODO: Sparse prompt embeddings can have more than one point
    }


    struct ggml_tensor * src = {};
    struct ggml_tensor * pos_src = {};
    int srcNE[4] = { 0, 0, 0, 0 };
    {
        // Expand per-image data in the batch direction to be per-mask
        // ref: https://github.com/facebookresearch/segment-anything/blob/6fdee8f2727f4506cfbbe553e23b895e27956588/segment_anything/modeling/mask_decoder.py#L125
        src = ggml_new_tensor_4d(ctx0, GGML_TYPE_F32, state.embd_img->ne[0], state.embd_img->ne[1], state.embd_img->ne[2], tokens->ne[2]);

        src = ggml_add(ctx0,
            ggml_repeat(ctx0,
                state.embd_img,
                src),
            prompt.embd_prompt_dense);

        srcNE[0] = src->ne[0];
        srcNE[1] = src->ne[1];
        srcNE[2] = src->ne[2];
        srcNE[3] = src->ne[3];

        // flatten & permute
        // ref: https://github.com/facebookresearch/segment-anything/blob/6fdee8f2727f4506cfbbe553e23b895e27956588/segment_anything/modeling/transformer.py#L83
        src = ggml_cont(ctx0, ggml_permute(ctx0,
            ggml_view_3d(ctx0,
                src,
                src->ne[0]*src->ne[1],
                src->ne[2],
                src->ne[3],
                src->nb[2],
                src->nb[3],
                0),
            1, 0, 2, 3));

        pos_src = ggml_new_tensor_4d(ctx0, GGML_TYPE_F32, pe_img->ne[0], pe_img->ne[1], pe_img->ne[2], tokens->ne[2]);
        pos_src = ggml_repeat(ctx0,
            pe_img,
            pos_src);

        // flatten & permute
        // ref: https://github.com/facebookresearch/segment-anything/blob/6fdee8f2727f4506cfbbe553e23b895e27956588/segment_anything/modeling/transformer.py#L83
        pos_src = ggml_cont(ctx0, ggml_permute(ctx0,
            ggml_view_3d(ctx0,
                pos_src,
                pos_src->ne[0]*pos_src->ne[1],
                pos_src->ne[2],
                pos_src->ne[3],
                pos_src->nb[2],
                pos_src->nb[3],
                0),
            1, 0, 2, 3));
    }

    struct ggml_tensor * queries = tokens;
    struct ggml_tensor * keys = src;
    {
        // Run the transformer
        // ref: https://github.com/facebookresearch/segment-anything/blob/6fdee8f2727f4506cfbbe553e23b895e27956588/segment_anything/modeling/transformer.py#L62
        for (int i = 0; i < int(model.dec.transformer_layers.size()); ++i) {
            const auto& tfm_layer = model.dec.transformer_layers[i];

            // Self attention block
            // ref: https://github.com/facebookresearch/segment-anything/blob/6fdee8f2727f4506cfbbe553e23b895e27956588/segment_anything/modeling/transformer.py#L154
            const bool skip_first_layer_pe = i == 0;
            if (skip_first_layer_pe) {
                queries = sam_decode_mask_transformer_attn(tfm_layer.self_attn, queries, queries, queries, ctx0, model);
            }
            else {
                struct ggml_tensor * q_0 = ggml_add(ctx0, queries, tokens);

                struct ggml_tensor * self_attn = sam_decode_mask_transformer_attn(tfm_layer.self_attn, q_0, q_0, queries, ctx0, model);
                queries = ggml_add(ctx0, queries, self_attn);
            }

            queries = ggml_norm(ctx0, queries, hparams.eps_decoder_transformer);
            queries = ggml_add_inplace(ctx0,
                    ggml_mul(ctx0, queries, tfm_layer.norm1_w),
                    tfm_layer.norm1_b);

            // Cross attention block, tokens attending to image embedding
            // ref: https://github.com/facebookresearch/segment-anything/blob/6fdee8f2727f4506cfbbe553e23b895e27956588/segment_anything/modeling/transformer.py#L163
            struct ggml_tensor * q_1 = ggml_add(ctx0, queries, tokens);
            struct ggml_tensor * k_1 = ggml_add(ctx0, keys, pos_src);

            struct ggml_tensor * cross_attn_token_to_img = sam_decode_mask_transformer_attn(tfm_layer.cross_attn_token_to_img, q_1, k_1, keys, ctx0, model);

            queries = ggml_add_inplace(ctx0, queries, cross_attn_token_to_img);
            queries = ggml_norm_inplace(ctx0, queries, hparams.eps_decoder_transformer);
            queries = ggml_add_inplace(ctx0,
                    ggml_mul(ctx0, queries, tfm_layer.norm2_w),
                    tfm_layer.norm2_b);

            // MLP block
            // ref: https://github.com/facebookresearch/segment-anything/blob/6fdee8f2727f4506cfbbe553e23b895e27956588/segment_anything/modeling/transformer.py#L170
            struct ggml_tensor * mlp_out = ggml_mul_mat(ctx0,
                tfm_layer.mlp_lin1_w,
                queries);

            mlp_out = ggml_add_inplace(ctx0, mlp_out, tfm_layer.mlp_lin1_b);

            // RELU activation
            mlp_out = ggml_relu_inplace(ctx0, mlp_out);
            mlp_out = ggml_mul_mat(ctx0, tfm_layer.mlp_lin2_w, mlp_out);

            mlp_out = ggml_add_inplace(ctx0, mlp_out, tfm_layer.mlp_lin2_b);

            queries = ggml_add_inplace(ctx0, queries, mlp_out);
            queries = ggml_norm_inplace(ctx0, queries, hparams.eps_decoder_transformer);
            queries = ggml_add_inplace(ctx0,
                    ggml_mul(ctx0, queries, tfm_layer.norm3_w),
                    tfm_layer.norm3_b);

            // Cross attention block, image embedding attending to tokens
            // ref: https://github.com/facebookresearch/segment-anything/blob/6fdee8f2727f4506cfbbe553e23b895e27956588/segment_anything/modeling/transformer.py#L175
            struct ggml_tensor * q_2 = ggml_add(ctx0, queries, tokens);
            struct ggml_tensor * k_2 = ggml_add(ctx0, keys, pos_src);

            struct ggml_tensor * cross_attn_img_to_token = sam_decode_mask_transformer_attn(tfm_layer.cross_attn_img_to_token, k_2, q_2, queries, ctx0, model);
            keys = ggml_add_inplace(ctx0, keys, cross_attn_img_to_token);
            keys = ggml_norm_inplace(ctx0, keys, hparams.eps_decoder_transformer);
            keys = ggml_add_inplace(ctx0,
                    ggml_mul(ctx0, keys, tfm_layer.norm4_w),
                    tfm_layer.norm4_b);
        }

        // Apply the final attention layer from the points to the image
        // ref: https://github.com/facebookresearch/segment-anything/blob/6fdee8f2727f4506cfbbe553e23b895e27956588/segment_anything/modeling/transformer.py#L99
        struct ggml_tensor * q = ggml_add(ctx0, queries, tokens);
        struct ggml_tensor * k = ggml_add(ctx0, keys, pos_src);

        struct ggml_tensor * final_attn_token_to_img = sam_decode_mask_transformer_attn(dec.transformer_final_attn_token_to_img, q, k, keys, ctx0, model);

        queries = ggml_add_inplace(ctx0, queries, final_attn_token_to_img);
        queries = ggml_norm_inplace(ctx0, queries, hparams.eps_decoder_transformer);
        queries = ggml_add_inplace(ctx0,
                ggml_mul(ctx0, queries, dec.transformer_norm_final_w),
                dec.transformer_norm_final_b);
    }


    struct ggml_tensor * iou_pred = ggml_view_2d(ctx0, queries, queries->ne[0], queries->ne[2], queries->nb[2], 0);
    const int num_mask_tokens = 4; // num_multimask_outputs + 1
    struct ggml_tensor * mask_tokens_out = ggml_view_3d(ctx0, queries, queries->ne[0], num_mask_tokens, queries->ne[2], queries->nb[1], num_mask_tokens*queries->nb[1], queries->nb[1]);

    // Upscale mask embeddings and predict masks using the mask tokens
    // ref: https://github.com/facebookresearch/segment-anything/blob/6fdee8f2727f4506cfbbe553e23b895e27956588/segment_anything/modeling/mask_decoder.py#L136
    keys = ggml_cont(ctx0, ggml_transpose(ctx0, keys));
    keys = ggml_view_4d(ctx0, keys, srcNE[0], srcNE[1], srcNE[2], srcNE[3], srcNE[0]*keys->nb[0], keys->nb[1], keys->nb[2], 0);
    // ggml_build_forward_expand(gf, keys);
    struct ggml_tensor * upscaled_embedding = {};
    {
        // ConvTranspose2d
        keys = ggml_conv_transpose_2d_p0(ctx0, dec.output_upscaling_0_w, keys, 2);
        ggml_allocr_alloc(state.allocr, keys); // TODO: This alloc shouldn't be needed
        keys = ggml_add_inplace(ctx0, keys, ggml_repeat(ctx0,
                                     ggml_reshape_3d(ctx0, dec.output_upscaling_0_b, 1, 1, dec.output_upscaling_0_b->ne[0]),
                                     keys));

        keys = sam_layer_norm_2d(ctx0, keys, n_img_embd, dec.output_upscaling_1_w, dec.output_upscaling_1_b, hparams.eps);

        // GELU activation
        keys = ggml_gelu_inplace(ctx0, keys);

        // ConvTranspose2d
        keys = ggml_conv_transpose_2d_p0(ctx0, dec.output_upscaling_3_w, keys, 2);
        ggml_allocr_alloc(state.allocr, keys); // TODO: This alloc shouldn't be needed
        keys = ggml_add_inplace(ctx0, ggml_repeat(ctx0,
                                ggml_reshape_3d(ctx0, dec.output_upscaling_3_b, 1, 1, dec.output_upscaling_3_b->ne[0]),
                                keys), keys);
        // GELU activation
        keys = ggml_gelu_inplace(ctx0, keys);
        upscaled_embedding = ggml_reshape_3d(ctx0, keys, keys->ne[0]*keys->ne[1], keys->ne[2], keys->ne[3]);
        upscaled_embedding = ggml_cont(ctx0, ggml_transpose(ctx0, upscaled_embedding)); // TODO: Shouldn't be needed
    }

    struct ggml_tensor * hyper_in = ggml_new_tensor_3d(ctx0, GGML_TYPE_F32, n_img_embd/2, num_mask_tokens, mask_tokens_out->ne[2]);

    for (int i = 0; i < num_mask_tokens; ++i) {
        const auto& mlp = dec.output_hypernet_mlps[i];
        struct ggml_tensor * in = ggml_view_2d(ctx0, mask_tokens_out, mask_tokens_out->ne[0], mask_tokens_out->ne[2], mask_tokens_out->nb[1], i*mask_tokens_out->nb[1]);
        struct ggml_tensor * out = sam_decode_mask_mlp_relu_3(in, mlp.w_0, mlp.b_0, mlp.w_1, mlp.b_1, mlp.w_2, mlp.b_2, ctx0);
        ggml_build_forward_expand(gf, ggml_cpy(ctx0, out, ggml_view_2d(ctx0, hyper_in, hyper_in->ne[0], hyper_in->ne[2], hyper_in->nb[1], i*hyper_in->nb[1])));
    }

    struct ggml_tensor * masks = ggml_mul_mat(ctx0, hyper_in, upscaled_embedding);
    masks = ggml_cont(ctx0, ggml_transpose(ctx0, masks)); // TODO: Shouldn't be needed
    masks = ggml_reshape_4d(ctx0, masks, keys->ne[0], keys->ne[1], masks->ne[1], keys->ne[3]);

    // Generate mask quality predictions
    // ref: https://github.com/facebookresearch/segment-anything/blob/6fdee8f2727f4506cfbbe553e23b895e27956588/segment_anything/modeling/mask_decoder.py#L146
    iou_pred = sam_decode_mask_mlp_relu_3(iou_pred, dec.iou_prediction_head_0_w, dec.iou_prediction_head_0_b, dec.iou_prediction_head_1_w, dec.iou_prediction_head_1_b, dec.iou_prediction_head_2_w, dec.iou_prediction_head_2_b, ctx0);

    // Select the correct mask or masks for output
    // ref: https://github.com/facebookresearch/segment-anything/blob/6fdee8f2727f4506cfbbe553e23b895e27956588/segment_anything/modeling/mask_decoder.py#L101
    iou_pred = ggml_cpy(state.ctx, ggml_view_1d(ctx0, iou_pred, iou_pred->ne[0] - 1, iou_pred->nb[0]), state.iou_predictions);
    masks = ggml_view_4d(ctx0, masks, masks->ne[0], masks->ne[1], masks->ne[2] - 1, masks->ne[3],
                                      masks->nb[1], masks->nb[2], masks->nb[3], masks->nb[2] /* offset*/);
    masks = ggml_cpy(state.ctx, masks, state.low_res_masks);

    ggml_build_forward_expand(gf, masks);
    ggml_build_forward_expand(gf, iou_pred);

    ggml_disconnect_node_from_graph(state.low_res_masks);
    ggml_disconnect_node_from_graph(state.iou_predictions);

    return true;
}

```




```cpp
bool sam_write_masks(const sam_hparams& hparams, int nx, int ny, const sam_state & state, const std::string & fname) {
    if (state.low_res_masks->ne[2] == 0) return true;
    if (state.low_res_masks->ne[2] != state.iou_predictions->ne[0]) {
        printf("Error: number of masks (%d) does not match number of iou predictions (%d)\n", (int)state.low_res_masks->ne[2], (int)state.iou_predictions->ne[0]);
        return false;
    }

    const int n_img_size = hparams.n_img_size();
    const float mask_threshold = hparams.mask_threshold;
    const float iou_threshold = hparams.iou_threshold;
    const float stability_score_threshold = hparams.stability_score_threshold;
    const float intersection_threshold = mask_threshold + hparams.stability_score_offset;
    const float union_threshold = mask_threshold - hparams.stability_score_offset;

    const int ne0 = state.low_res_masks->ne[0];
    const int ne1 = state.low_res_masks->ne[1];
    const int ne2 = state.low_res_masks->ne[2];

    // Remove padding and upscale masks to the original image size.
    // ref: https://github.com/facebookresearch/segment-anything/blob/efeab7296ab579d4a261e554eca80faf6b33924a/segment_anything/modeling/sam.py#L140

    const float preprocess_scale = std::max(nx, ny) / float(n_img_size);
    const int cropped_nx = int(nx / preprocess_scale + 0.5f);
    const int cropped_ny = int(ny / preprocess_scale + 0.5f);

    const float scale_x_1 = (float)ne0 / (float)n_img_size;
    const float scale_y_1 = (float)ne1 / (float)n_img_size;

    const float scale_x_2 = float(cropped_nx) / float(nx);
    const float scale_y_2 = float(cropped_ny) / float(ny);

    const auto iou_data = (float*)state.iou_predictions->data;

    for (int i = 0; i < ne2; ++i) {
        if (iou_threshold > 0.f && iou_data[i] < iou_threshold) {
            printf("Skipping mask %d with iou %f below threshold %f\n", i, iou_data[i], iou_threshold);
            continue; // Filtering masks with iou below the threshold
        }

        std::vector<float> mask_data(n_img_size*n_img_size);
        {
            const float* data = (float *) state.low_res_masks->data + i*ne0*ne1;

            for (int iy = 0; iy < n_img_size; ++iy) {
                for (int ix = 0; ix < n_img_size; ++ix) {
                    const float sx = std::max(scale_x_1*(ix + 0.5f) - 0.5f, 0.0f);
                    const float sy = std::max(scale_y_1*(iy + 0.5f) - 0.5f, 0.0f);

                    const int x0 = std::max(0, (int)sx);
                    const int y0 = std::max(0, (int)sy);

                    const int x1 = std::min(x0 + 1, ne0 - 1);
                    const int y1 = std::min(y0 + 1, ne1 - 1);

                    const float dx = sx - x0;
                    const float dy = sy - y0;

                    const int j00 = y0*ne0 + x0;
                    const int j01 = y0*ne0 + x1;
                    const int j10 = y1*ne0 + x0;
                    const int j11 = y1*ne0 + x1;

                    const float v00 = data[j00];
                    const float v01 = data[j01];
                    const float v10 = data[j10];
                    const float v11 = data[j11];

                    const float v0 = (1-dx)*v00 + dx*v01;
                    const float v1 = (1-dx)*v10 + dx*v11;

                    const float v = (1-dy)*v0 + dy*v1;

                    mask_data[iy*n_img_size + ix] = v;
                }
            }
        }

        int intersections = 0;
        int unions = 0;
        sam_image_u8 res;
        int min_iy = ny;
        int max_iy = 0;
        int min_ix = nx;
        int max_ix = 0;
        {
            const float* data = mask_data.data();

            res.nx = nx;
            res.ny = ny;
            res.data.resize(nx*ny);

            for (int iy = 0; iy < ny; ++iy) {
                for (int ix = 0; ix < nx; ++ix) {
                    const float sx = std::max(scale_x_2*(ix + 0.5f) - 0.5f, 0.0f);
                    const float sy = std::max(scale_y_2*(iy + 0.5f) - 0.5f, 0.0f);

                    const int x0 = std::max(0, (int)sx);
                    const int y0 = std::max(0, (int)sy);

                    const int x1 = std::min(x0 + 1, cropped_nx - 1);
                    const int y1 = std::min(y0 + 1, cropped_ny - 1);

                    const float dx = sx - x0;
                    const float dy = sy - y0;

                    const int j00 = y0*n_img_size + x0;
                    const int j01 = y0*n_img_size + x1;
                    const int j10 = y1*n_img_size + x0;
                    const int j11 = y1*n_img_size + x1;

                    const float v00 = data[j00];
                    const float v01 = data[j01];
                    const float v10 = data[j10];
                    const float v11 = data[j11];

                    const float v0 = (1-dx)*v00 + dx*v01;
                    const float v1 = (1-dx)*v10 + dx*v11;

                    const float v = (1-dy)*v0 + dy*v1;

                    if (v > intersection_threshold) {
                        intersections++;
                    }
                    if (v > union_threshold) {
                        unions++;
                    }
                    if (v > mask_threshold) {
                        min_iy = std::min(min_iy, iy);
                        max_iy = std::max(max_iy, iy);
                        min_ix = std::min(min_ix, ix);
                        max_ix = std::max(max_ix, ix);

                        res.data[iy*nx + ix] = 255;
                    }
                }
            }
        }

        const float stability_score = float(intersections) / float(unions);
        if (stability_score_threshold > 0.f && stability_score < stability_score_threshold) {
            printf("Skipping mask %d with stability score %f below threshold %f\n", i, stability_score, stability_score_threshold);
            continue; // Filtering masks with stability score below the threshold
        }

        printf("Mask %d: iou = %f, stability_score = %f, bbox (%d, %d), (%d, %d)\n",
                i, iou_data[i], stability_score, min_ix, max_ix, min_iy, max_iy);

        std::string filename = fname + std::to_string(i) + ".png";
        if (!stbi_write_png(filename.c_str(), res.nx, res.ny, 1, res.data.data(), res.nx)) {
            printf("%s: failed to write mask %s\n", __func__, filename.c_str());
            return false;
        }
    }


    return true;
}

```

这段代码是一个名为 `sam_build_fast_graph` 的函数，它的作用是构建一个快速的图形表示（ggml）图。它接受一个 `sam_model` 对象，一个 `sam_state` 对象，一个 `int` 类型的图的边数 `nx` 和一个 `int` 类型的点的坐标 `point`。

函数首先定义了一个名为 `ggml_params` 的结构体，它指定了需要分配的内存大小和缓冲区大小。然后，它创建了一个名为 `ctx0` 的 `ggml_context` 对象和一个新的图 `gf`。

接下来，它调用了一个名为 `ggml_init` 的函数，该函数初始化 `ggml_params` 并返回一个 `ggml_context` 对象。然后，它使用 `ggml_new_graph` 函数创建了一个新的图。

在图形表示的编码过程中，函数调用了一个名为 `sam_encode_prompt` 的函数，该函数的参数包括 `model`、`ctx0` 和 `gf`。如果没有编码成功，函数将打印一个错误消息并返回。

然后，函数使用 `sam_fill_dense_pe` 函数获取了 `model` 中的点的密度的行数，并创建了一个密度映射的 `ggml_tensor` 对象。如果该函数失败，函数将打印一个错误消息并返回。

最后，函数使用 `sam_decode_mask` 函数解码了密度映射，如果解码成功，函数将打印一个错误消息并返回。然后，函数 free 了分配的内存并返回图 `gf`。


```cpp
struct ggml_cgraph  * sam_build_fast_graph(
        const sam_model     & model,
                  sam_state & state,
                        int   nx,
                        int   ny,
                  sam_point   point) {

    struct ggml_init_params ggml_params = {
        /*.mem_size   =*/ state.buf_compute_fast.size(),
        /*.mem_buffer =*/ state.buf_compute_fast.data(),
        /*.no_alloc   =*/ true, // skip allocating as we use ggml_alloc to allocate exact memory requirements
    };

    struct ggml_context * ctx0   = ggml_init(ggml_params);
    struct ggml_cgraph  * gf     = ggml_new_graph(ctx0);

    prompt_encoder_result enc_res = sam_encode_prompt(model, ctx0, gf, state, nx, ny, point);
    if (!enc_res.embd_prompt_sparse || !enc_res.embd_prompt_dense) {
        fprintf(stderr, "%s: failed to encode prompt (%f, %f)\n", __func__, point.x, point.y);
        return {};
    }

    struct ggml_tensor * pe_img_dense = sam_fill_dense_pe(model, ctx0, gf, state);
    if (!pe_img_dense) {
        fprintf(stderr, "%s: failed to get dense positional encoding\n", __func__);
        return {};
    }

    if (!sam_decode_mask(model, enc_res, pe_img_dense, ctx0, gf, state)) {
         fprintf(stderr, "%s: failed to decode mask\n", __func__);
         return {};
    }

    ggml_free(ctx0);

    return gf;
}

```

This is a Sam utility that generates a usage message and some placeholders for the input parameters. The usage message explains how to use the tool and gives examples of how to use it.

The placeholders are parameters that the user needs to specify when running the tool. For example, the user needs to specify the output file name and the threshold values for masking and iou.

The message also explains the hyperparameters that can be used to control the behavior of the tool. For example, the user can specify the score threshold, the iou threshold, the score offset, the epsilon value, and the epsilon decoder transformer.

Overall, this tool is designed to help users understand what parameters are required to use the Sam tool for scoring big endian aligned samples.



```cpp
void sam_print_usage(int argc, char ** argv, const sam_params & params) {
    fprintf(stderr, "usage: %s [options]\n", argv[0]);
    fprintf(stderr, "\n");
    fprintf(stderr, "options:\n");
    fprintf(stderr, "  -h, --help            show this help message and exit\n");
    fprintf(stderr, "  -s SEED, --seed SEED  RNG seed (default: -1)\n");
    fprintf(stderr, "  -t N, --threads N     number of threads to use during computation (default: %d)\n", params.n_threads);
    fprintf(stderr, "  -m FNAME, --model FNAME\n");
    fprintf(stderr, "                        model path (default: %s)\n", params.model.c_str());
    fprintf(stderr, "  -i FNAME, --inp FNAME\n");
    fprintf(stderr, "                        input file (default: %s)\n", params.fname_inp.c_str());
    fprintf(stderr, "  -o FNAME, --out FNAME\n");
    fprintf(stderr, "                        mask file name prefix (default: %s)\n", params.fname_out.c_str());
    fprintf(stderr, "SAM hyperparameters:\n");
    fprintf(stderr, "  -mt FLOAT, --mask-threshold\n");
    fprintf(stderr, "                        mask threshold (default: %f)\n", params.mask_threshold);
    fprintf(stderr, "  -it FLOAT, --iou-threshold\n");
    fprintf(stderr, "                        iou threshold (default: %f)\n", params.iou_threshold);
    fprintf(stderr, "  -st FLOAT, --score-threshold\n");
    fprintf(stderr, "                        score threshold (default: %f)\n", params.stability_score_threshold);
    fprintf(stderr, "  -so FLOAT, --score-offset\n");
    fprintf(stderr, "                        score offset (default: %f)\n", params.stability_score_offset);
    fprintf(stderr, "  -e FLOAT, --epsilon\n");
    fprintf(stderr, "                        epsilon (default: %f)\n", params.eps);
    fprintf(stderr, "  -ed FLOAT, --epsilon-decoder-transformer\n");
    fprintf(stderr, "                        epsilon decoder transformer (default: %f)\n", params.eps_decoder_transformer);
    fprintf(stderr, "SAM prompt:\n");
    fprintf(stderr, "  -p TUPLE, --point-prompt\n");
    fprintf(stderr, "                        point to be used as prompt for SAM (default: %f,%f). Must be in a format FLOAT,FLOAT \n", params.pt.x, params.pt.y);
    fprintf(stderr, "\n");
}

```

This is a C++ program that uses the Samy拨粉训练和评估脚本。Samy是一个基于深度学习的点对测试引擎，可以对模型的每个点进行评估。

该程序通过处理输入文件中的参数，来读取用户提供的命令行参数。如果用户提供的参数不正确，程序将会打印错误信息并退出。

具体来说，程序支持以下命令行参数：

-a，--algorithm-level：表示输入算法的最低级别。默认为0。
-h，--help：打印帮助信息，帮助用户了解程序的使用方法。
-i，--input：输入文件的路径，可以是文件名或文件路径。
-o，--output：输出文件的路径，可以是文件名或文件路径。
-e，--epsilon：表示点对测试的误差允许值。
-f，--freq：表示点对测试的频率。
-m，--model：表示模型文件的路径，可以是文件名或文件路径。
-n，--name：表示模型文件的命名，可以是用户自定义的名称。
-p，--point-prompt：提示用户输入一个点对。
-p，--point-prompt-mode：表示点对提示信息中，点之间的距离是否可以忽略。
-st，--score-threshold：表示点对分数的阈值。
-threshold：表示点对阈值，即当两个点之间的距离小于该阈值时，该点对通过该阈值。
-s，--strategy：表示评估策略，目前支持基于距离的IoU分数和基于策略的分数。
-t，--tocolor：表示是否使用颜色标记得分。
-x，--ximport：表示是否从指定文件中读取模型。
-X，--MAX_X：表示设置X的最大值。
-X，--MAX_Y：表示设置Y的最大值。
-X，--MUZ：表示指示该参数用于哪个模型。
-X，--NORM：表示指示是否使用归一化的点对。
-X，--SPACE：表示指示是否启用空间分割。
-X，--WIN：表示指定窗口。

如果用户提供的参数不正确，程序将会打印错误信息并退出。


```cpp
bool sam_params_parse(int argc, char ** argv, sam_params & params) {
    for (int i = 1; i < argc; i++) {
        std::string arg = argv[i];

        if (arg == "-s" || arg == "--seed") {
            params.seed = std::stoi(argv[++i]);
        } else if (arg == "-t" || arg == "--threads") {
            params.n_threads = std::stoi(argv[++i]);
        } else if (arg == "-m" || arg == "--model") {
            params.model = argv[++i];
        } else if (arg == "-i" || arg == "--inp") {
            params.fname_inp = argv[++i];
        } else if (arg == "-o" || arg == "--out") {
            params.fname_out = argv[++i];
        } else if (arg == "-mt" || arg == "--mask-threshold") {
            params.mask_threshold = std::stof(argv[++i]);
        } else if (arg == "-it" || arg == "--iou-threshold") {
            params.iou_threshold = std::stof(argv[++i]);
        } else if (arg == "-st" || arg == "--score-threshold") {
            params.stability_score_threshold = std::stof(argv[++i]);
        } else if (arg == "-so" || arg == "--score-offset") {
            params.stability_score_offset = std::stof(argv[++i]);
        } else if (arg == "-e" || arg == "--epsilon") {
            params.eps = std::stof(argv[++i]);
        } else if (arg == "-ed" || arg == "--epsilon-decoder-transformer") {
            params.eps_decoder_transformer = std::stof(argv[++i]);
        } else if (arg == "-p" || arg == "--point-prompt") {
            // TODO multiple points per model invocation
            char* point = argv[++i];

            char* coord = strtok(point, ",");
            if (!coord){
                fprintf(stderr, "Error while parsing prompt!\n");
                exit(1);
            }
            params.pt.x = std::stof(coord);
            coord = strtok(NULL, ",");
            if (!coord){
                fprintf(stderr, "Error while parsing prompt!\n");
                exit(1);
            }
            params.pt.y = std::stof(coord);
        } else if (arg == "-h" || arg == "--help") {
            sam_print_usage(argc, argv, params);
            exit(0);
        } else {
            fprintf(stderr, "error: unknown argument: %s\n", arg.c_str());
            sam_print_usage(argc, argv, params);
            exit(0);
        }
    }

    return true;
}

```

This is a function definition for `__main__` that builds a FastGlow model and executes it.

The function takes a `state` object that contains information about the input data and the graph. It initializes the input data and sets up the input handling infrastructure.

Then it computes the exact memory requirements of the input data and builds a FastGlow graph with the measured memory requirements.

Next, it computes the graph with the input data and executes it. Finally, it prints some timing information and saves the output data.

Note that the `__main__` function is not thread safe and should not be called directly from thread-based code.


```cpp
int main(int argc, char ** argv) {
    const int64_t t_main_start_us = ggml_time_us();

    sam_params params;
    params.model = "models/sam-vit-b/ggml-model-f16.bin";

    sam_model model;
    sam_state state;
    int64_t t_load_us = 0;

    if (sam_params_parse(argc, argv, params) == false) {
        return 1;
    }

    if (params.seed < 0) {
        params.seed = time(NULL);
    }
    fprintf(stderr, "%s: seed = %d\n", __func__, params.seed);

    // load the image
    sam_image_u8 img0;
    if (!sam_image_load_from_file(params.fname_inp, img0)) {
        fprintf(stderr, "%s: failed to load image from '%s'\n", __func__, params.fname_inp.c_str());
        return 1;
    }
    fprintf(stderr, "%s: loaded image '%s' (%d x %d)\n", __func__, params.fname_inp.c_str(), img0.nx, img0.ny);

    // preprocess to f32
    sam_image_f32 img1;
    if (!sam_image_preprocess(img0, img1)) {
        fprintf(stderr, "%s: failed to preprocess image\n", __func__);
        return 1;
    }
    fprintf(stderr, "%s: preprocessed image (%d x %d)\n", __func__, img1.nx, img1.ny);


    // load the model
    {
        const int64_t t_start_us = ggml_time_us();

        if (!sam_model_load(params, model)) {
            fprintf(stderr, "%s: failed to load model from '%s'\n", __func__, params.model.c_str());
            return 1;
        }

        t_load_us = ggml_time_us() - t_start_us;
    }

    {
        static size_t buf_size = 256u*1024*1024;

        struct ggml_init_params ggml_params = {
            /*.mem_size   =*/ buf_size,
            /*.mem_buffer =*/ NULL,
            /*.no_alloc   =*/ false,
        };

        state.ctx = ggml_init(ggml_params);

        state.embd_img = ggml_new_tensor_3d(state.ctx, GGML_TYPE_F32,
                model.hparams.n_img_embd(), model.hparams.n_img_embd(), model.hparams.n_enc_out_chans);

        state.low_res_masks = ggml_new_tensor_3d(state.ctx, GGML_TYPE_F32,
                model.hparams.n_enc_out_chans, model.hparams.n_enc_out_chans, 3);

        state.iou_predictions = ggml_new_tensor_1d(state.ctx, GGML_TYPE_F32, 3);
    }


    static const size_t tensor_alignment = 32;
    {
        state.buf_compute_img_enc.resize(ggml_tensor_overhead()*GGML_DEFAULT_GRAPH_SIZE + ggml_graph_overhead());
        state.allocr = ggml_allocr_new_measure(tensor_alignment);
        struct ggml_cgraph * gf_measure = sam_encode_image(model, state, img1);
        if (!gf_measure) {
            fprintf(stderr, "%s: failed to encode image\n", __func__);
            return 1;
        }

        size_t alloc_size = ggml_allocr_alloc_graph(state.allocr, gf_measure) + tensor_alignment;
        ggml_allocr_free(state.allocr);

        // recreate allocator with exact memory requirements
        state.buf_alloc_img_enc.resize(alloc_size);
        state.allocr = ggml_allocr_new(state.buf_alloc_img_enc.data(), state.buf_alloc_img_enc.size(), tensor_alignment);

        // compute the graph with the measured exact memory requirements from above
        ggml_allocr_reset(state.allocr);

        struct ggml_cgraph  * gf = sam_encode_image(model, state, img1);
        if (!gf) {
            fprintf(stderr, "%s: failed to encode image\n", __func__);
            return 1;
        }

        ggml_allocr_alloc_graph(state.allocr, gf);

        ggml_graph_compute_helper(state.work_buffer, gf, params.n_threads);

        print_t_f32("embd_img", state.embd_img);

        ggml_allocr_free(state.allocr);
        state.allocr = NULL;
        state.work_buffer.clear();
    }
    {
        state.buf_compute_fast.resize(ggml_tensor_overhead()*GGML_DEFAULT_GRAPH_SIZE + ggml_graph_overhead());
        state.allocr = ggml_allocr_new_measure(tensor_alignment);

        // TODO: more varied prompts
        fprintf(stderr, "prompt: (%f, %f)\n", params.pt.x, params.pt.y);

        // measure memory requirements for the graph
        struct ggml_cgraph  * gf_measure = sam_build_fast_graph(model, state, img0.nx, img0.ny, params.pt);
        if (!gf_measure) {
            fprintf(stderr, "%s: failed to build fast graph to measure\n", __func__);
            return 1;
        }

        size_t alloc_size = ggml_allocr_alloc_graph(state.allocr, gf_measure) + tensor_alignment;
        ggml_allocr_free(state.allocr);

        // recreate allocator with exact memory requirements
        state.buf_alloc_fast.resize(alloc_size);
        state.allocr = ggml_allocr_new(state.buf_alloc_fast.data(), state.buf_alloc_fast.size(), tensor_alignment);

        // compute the graph with the measured exact memory requirements from above
        ggml_allocr_reset(state.allocr);

        struct ggml_cgraph  * gf = sam_build_fast_graph(model, state, img0.nx, img0.ny, params.pt);
        if (!gf) {
            fprintf(stderr, "%s: failed to build fast graph\n", __func__);
            return 1;
        }

        ggml_allocr_alloc_graph(state.allocr, gf);

        ggml_graph_compute_helper(state.work_buffer, gf, params.n_threads);

        //print_t_f32("iou_predictions", state.iou_predictions);
        //print_t_f32("low_res_masks", state.low_res_masks);
        ggml_allocr_free(state.allocr);
        state.allocr = NULL;
    }

    if (!sam_write_masks(model.hparams, img0.nx, img0.ny, state, params.fname_out)) {
        fprintf(stderr, "%s: failed to write masks\n", __func__);
        return 1;
    }

    // report timing
    {
        const int64_t t_main_end_us = ggml_time_us();

        fprintf(stderr, "\n\n");
        fprintf(stderr, "%s:     load time = %8.2f ms\n", __func__, t_load_us/1000.0f);
        fprintf(stderr, "%s:    total time = %8.2f ms\n", __func__, (t_main_end_us - t_main_start_us)/1000.0f);
    }

    ggml_free(model.ctx);

    return 0;
}

```