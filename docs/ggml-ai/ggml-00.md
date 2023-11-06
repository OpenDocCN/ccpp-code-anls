# GGML源码解析 0

# ggml

[Roadmap](https://github.com/users/ggerganov/projects/7) / [Manifesto](https://github.com/ggerganov/llama.cpp/discussions/205)

Tensor library for machine learning

***Note that this project is under active development. \
Some of the development is currently happening in the [llama.cpp](https://github.com/ggerganov/llama.cpp) and [whisper.cpp](https://github.com/ggerganov/whisper.cpp) repos***

## Features

- Written in C
- 16-bit float support
- Integer quantization support (4-bit, 5-bit, 8-bit, etc.)
- Automatic differentiation
- ADAM and L-BFGS optimizers
- Optimized for Apple Silicon
- On x86 architectures utilizes AVX / AVX2 intrinsics
- On ppc64 architectures utilizes VSX intrinsics
- No third-party dependencies
- Zero memory allocations during runtime

## Updates

- [X] Example of GPT-2 inference [examples/gpt-2](https://github.com/ggerganov/ggml/tree/master/examples/gpt-2)
- [X] Example of GPT-J inference [examples/gpt-j](https://github.com/ggerganov/ggml/tree/master/examples/gpt-j)
- [X] Example of Whisper inference [examples/whisper](https://github.com/ggerganov/ggml/tree/master/examples/whisper)
- [X] Support 4-bit integer quantization https://github.com/ggerganov/ggml/pull/27
- [X] Example of Cerebras-GPT inference [examples/gpt-2](https://github.com/ggerganov/ggml/tree/master/examples/gpt-2)
- [ ] Example of FLAN-T5 inference https://github.com/ggerganov/ggml/pull/12
- [X] Example of LLaMA inference [ggerganov/llama.cpp](https://github.com/ggerganov/llama.cpp)
- [X] Example of LLaMA training [ggerganov/llama.cpp/examples/baby-llama](https://github.com/ggerganov/llama.cpp/tree/master/examples/baby-llama)
- [X] Example of Falcon inference [cmp-nct/ggllm.cpp](https://github.com/cmp-nct/ggllm.cpp)
- [X] Example of BLOOM inference [NouamaneTazi/bloomz.cpp](https://github.com/NouamaneTazi/bloomz.cpp)
- [X] Example of RWKV inference [saharNooby/rwkv.cpp](https://github.com/saharNooby/rwkv.cpp)
- [X] Example of SAM inference [examples/sam](https://github.com/ggerganov/ggml/tree/master/examples/sam)
- [X] Idea for GPU support: https://github.com/ggerganov/llama.cpp/discussions/915
- [X] Example of StableLM (GPT-NeoX) inference [examples/gpt-neox](https://github.com/ggerganov/ggml/tree/master/examples/gpt-neox)
- [X] Example of BERT inference [skeskinen/bert.cpp](https://github.com/skeskinen/bert.cpp)
- [X] Example of 💫 StarCoder inference [examples/starcoder](https://github.com/ggerganov/ggml/tree/master/examples/starcoder)
- [X] Example of MPT inference [examples/mpt](https://github.com/ggerganov/ggml/tree/master/examples/mpt)
- [X] Example of Replit inference [examples/replit](https://github.com/ggerganov/ggml/tree/master/examples/replit)
- [X] Example of BioGPT inference [PABannier/biogpt.cpp](https://github.com/PABannier/biogpt.cpp)
- [X] Example of Encodec inference [PABannier/encodec.cpp](https://github.com/PABannier/encodec.cpp)
- [X] Example of CLIP inference [monatis/clip.cpp](https://github.com/monatis/clip.cpp)
- [X] Example of MiniGPT4 inference [Maknee/minigpt4.cpp](https://github.com/Maknee/minigpt4.cpp)
- [X] Example of ChatGLM inference [li-plus/chatglm.cpp](https://github.com/li-plus/chatglm.cpp)
- [X] Example of Stable Diffusion inference [leejet/stable-diffusion.cpp](https://github.com/leejet/stable-diffusion.cpp)
- [X] Example of Qwen inference [QwenLM/qwen.cpp](https://github.com/QwenLM/qwen.cpp)
- [X] Example of YOLO inference [examples/yolo](https://github.com/ggerganov/ggml/tree/master/examples/yolo)

## Whisper inference (example)

With ggml you can efficiently run [Whisper](examples/whisper) inference on the CPU.

Memory requirements:

| Model  | Disk   | Mem     |
| ---    | ---    | ---     |
| tiny   |  75 MB | ~280 MB |
| base   | 142 MB | ~430 MB |
| small  | 466 MB | ~1.0 GB |
| medium | 1.5 GB | ~2.6 GB |
| large  | 2.9 GB | ~4.7 GB |

## GPT inference (example)

With ggml you can efficiently run [GPT-2](examples/gpt-2) and [GPT-J](examples/gpt-j) inference on the CPU.

Here is how to run the example programs:

```cppbash
# Build ggml + examples
git clone https://github.com/ggerganov/ggml
cd ggml
mkdir build && cd build
cmake ..
make -j4 gpt-2 gpt-j

# Run the GPT-2 small 117M model
../examples/gpt-2/download-ggml-model.sh 117M
./bin/gpt-2 -m models/gpt-2-117M/ggml-model.bin -p "This is an example"

# Run the GPT-J 6B model (requires 12GB disk space and 16GB CPU RAM)
../examples/gpt-j/download-ggml-model.sh 6B
./bin/gpt-j -m models/gpt-j-6B/ggml-model.bin -p "This is an example"

# Install Python dependencies
python3 -m pip install -r ../requirements.txt

# Run the Cerebras-GPT 111M model
# Download from: https://huggingface.co/cerebras
python3 ../examples/gpt-2/convert-cerebras-to-ggml.py /path/to/Cerebras-GPT-111M/
./bin/gpt-2 -m /path/to/Cerebras-GPT-111M/ggml-model-f16.bin -p "This is an example"
```

The inference speeds that I get for the different models on my 32GB MacBook M1 Pro are as follows:

| Model | Size  | Time / Token |
| ---   | ---   | ---    |
| GPT-2 |  117M |   5 ms |
| GPT-2 |  345M |  12 ms |
| GPT-2 |  774M |  23 ms |
| GPT-2 | 1558M |  42 ms |
| ---   | ---   | ---    |
| GPT-J |    6B | 125 ms |

For more information, checkout the corresponding programs in the [examples](examples) folder.

## Using Metal (only with GPT-2)

For GPT-2 models, offloading to GPU is possible. Note that it will not improve inference performances but will reduce power consumption and free up the CPU for other tasks.

To enable GPU offloading on MacOS:

```cppbash
cmake -DGGML_METAL=ON -DBUILD_SHARED_LIBS=Off ..

# add -ngl 1
./bin/gpt-2 -t 4 -ngl 100 -m models/gpt-2-117M/ggml-model.bin -p "This is an example"
```

## Using cuBLAS

```cppbash
# fix the path to point to your CUDA compiler
cmake -DGGML_CUBLAS=ON -DCMAKE_CUDA_COMPILER=/usr/local/cuda-12.1/bin/nvcc ..
```

## Using clBLAST

```cppbash
cmake -DGGML_CLBLAST=ON ..
```

## Resources

- [GGML - Large Language Models for Everyone](https://github.com/rustformers/llm/blob/main/crates/ggml/README.md): a description of the GGML format provided by the maintainers of the `llm` Rust crate, which provides Rust bindings for GGML
- [marella/ctransformers](https://github.com/marella/ctransformers): Python bindings for GGML models.
- [go-skynet/go-ggml-transformers.cpp](https://github.com/go-skynet/go-ggml-transformers.cpp): Golang bindings for GGML models
- [smspillaz/ggml-gobject](https://github.com/smspillaz/ggml-gobject): GObject-introspectable wrapper for use of GGML on the GNOME platform.


# GGUF

GGUF is a file format for storing models for inference with GGML and executors based on GGML. GGUF is a binary format that is designed for fast loading and saving of models, and for ease of reading. Models are traditionally developed using PyTorch or another framework, and then converted to GGUF for use in GGML.

It is a successor file format to GGML, GGMF and GGJT, and is designed to be unambiguous by containing all the information needed to load a model. It is also designed to be extensible, so that new information can be added to models without breaking compatibility.

For more information about the motivation behind GGUF, see [Historical State of Affairs](#historical-state-of-affairs).

## Specification

GGUF is a format based on the existing GGJT, but makes a few changes to the format to make it more extensible and easier to use. The following features are desired:

- Single-file deployment: they can be easily distributed and loaded, and do not require any external files for additional information.
- Extensible: new features can be added to GGML-based executors/new information can be added to GGUF models without breaking compatibility with existing models.
- `mmap` compatibility: models can be loaded using `mmap` for fast loading and saving.
- Easy to use: models can be easily loaded and saved using a small amount of code, with no need for external libraries, regardless of the language used.
- Full information: all information needed to load a model is contained in the model file, and no additional information needs to be provided by the user.

The key difference between GGJT and GGUF is the use of a key-value structure for the hyperparameters (now referred to as metadata), rather than a list of untyped values. This allows for new metadata to be added without breaking compatibility with existing models, and to annotate the model with additional information that may be useful for inference or for identifying the model.

### File Structure

GGUF files are structured as follows. They use a global alignment specified in the `general.alignment` metadata field, referred to as `ALIGNMENT` below. Where required, the file is padded with `0x00` bytes to the next multiple of `general.alignment`.

Fields, including arrays, are written sequentially without alignment unless otherwise specified.

Models are little-endian by default. They can also come in big-endian for use with big-endian computers; in this case, all values (including metadata values and tensors) will also be big-endian. At the time of writing, there is no way to determine if a model is big-endian; this may be rectified in future versions. If no additional information is provided, assume the model is little-endian.

```cppc
enum ggml_type: uint32_t {
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

enum gguf_metadata_value_type: uint32_t {
    // The value is a 8-bit unsigned integer.
    GGUF_METADATA_VALUE_TYPE_UINT8 = 0,
    // The value is a 8-bit signed integer.
    GGUF_METADATA_VALUE_TYPE_INT8 = 1,
    // The value is a 16-bit unsigned little-endian integer.
    GGUF_METADATA_VALUE_TYPE_UINT16 = 2,
    // The value is a 16-bit signed little-endian integer.
    GGUF_METADATA_VALUE_TYPE_INT16 = 3,
    // The value is a 32-bit unsigned little-endian integer.
    GGUF_METADATA_VALUE_TYPE_UINT32 = 4,
    // The value is a 32-bit signed little-endian integer.
    GGUF_METADATA_VALUE_TYPE_INT32 = 5,
    // The value is a 32-bit IEEE754 floating point number.
    GGUF_METADATA_VALUE_TYPE_FLOAT32 = 6,
    // The value is a boolean.
    // 1-byte value where 0 is false and 1 is true.
    // Anything else is invalid, and should be treated as either the model being invalid or the reader being buggy.
    GGUF_METADATA_VALUE_TYPE_BOOL = 7,
    // The value is a UTF-8 non-null-terminated string, with length prepended.
    GGUF_METADATA_VALUE_TYPE_STRING = 8,
    // The value is an array of other values, with the length and type prepended.
    ///
    // Arrays can be nested, and the length of the array is the number of elements in the array, not the number of bytes.
    GGUF_METADATA_VALUE_TYPE_ARRAY = 9,
    // The value is a 64-bit unsigned little-endian integer.
    GGUF_METADATA_VALUE_TYPE_UINT64 = 10,
    // The value is a 64-bit signed little-endian integer.
    GGUF_METADATA_VALUE_TYPE_INT64 = 11,
    // The value is a 64-bit IEEE754 floating point number.
    GGUF_METADATA_VALUE_TYPE_FLOAT64 = 12,
}

// A string in GGUF.
struct gguf_string_t {
    // The length of the string, in bytes.
    uint64_t len;
    // The string as a UTF-8 non-null-terminated string.
    char string[len];
}

union gguf_metadata_value_t {
    uint8_t uint8;
    int8_t int8;
    uint16_t uint16;
    int16_t int16;
    uint32_t uint32;
    int32_t int32;
    float float32;
    uint64_t uint64;
    int64_t int64;
    double float64;
    bool bool_;
    gguf_string_t string;
    struct {
        // Any value type is valid, including arrays.
        gguf_metadata_value_type type;
        // Number of elements, not bytes
        uint64_t len;
        // The array of values.
        gguf_metadata_value_t array[len];
    } array;
};

struct gguf_metadata_kv_t {
    // The key of the metadata. It is a standard GGUF string, with the following caveats:
    // - It must be a valid ASCII string.
    // - It must be a hierarchical key, where each segment is `lower_snake_case` and separated by a `.`.
    // - It must be at most 2^16-1/65535 bytes long.
    // Any keys that do not follow these rules are invalid.
    gguf_string_t key;

    // The type of the value.
    // Must be one of the `gguf_metadata_value_type` values.
    gguf_metadata_value_type value_type;
    // The value.
    gguf_metadata_value_t value;
};

struct gguf_header_t {
    // Magic number to announce that this is a GGUF file.
    // Must be `GGUF` at the byte level: `0x47` `0x47` `0x55` `0x46`.
    // Your executor might do little-endian byte order, so it might be
    // check for 0x46554747 and letting the endianness cancel out.
    // Consider being *very* explicit about the byte order here.
    uint32_t magic;
    // The version of the format implemented.
    // Must be `3` for version described in this spec, which introduces big-endian support.
    //
    // This version should only be increased for structural changes to the format.
    // Changes that do not affect the structure of the file should instead update the metadata
    // to signify the change.
    uint32_t version;
    // The number of tensors in the file.
    // This is explicit, instead of being included in the metadata, to ensure it is always present
    // for loading the tensors.
    uint64_t tensor_count;
    // The number of metadata key-value pairs.
    uint64_t metadata_kv_count;
    // The metadata key-value pairs.
    gguf_metadata_kv_t metadata_kv[metadata_kv_count];
};

uint64_t align_offset(uint64_t offset) {
    return offset + (ALIGNMENT - (offset % ALIGNMENT)) % ALIGNMENT;
}

struct gguf_tensor_info_t {
    // The name of the tensor. It is a standard GGUF string, with the caveat that
    // it must be at most 64 bytes long.
    gguf_string_t name;
    // The number of dimensions in the tensor.
    // Currently at most 4, but this may change in the future.
    uint32_t n_dimensions;
    // The dimensions of the tensor.
    uint64_t dimensions[n_dimensions];
    // The type of the tensor.
    ggml_type type;
    // The offset of the tensor's data in this file in bytes.
    //
    // This offset is relative to `tensor_data`, not to the start
    // of the file, to make it easier for writers to write the file.
    // Readers should consider exposing this offset relative to the
    // file to make it easier to read the data.
    //
    // Must be a multiple of `ALIGNMENT`. That is, `align_offset(offset) == offset`.
    uint64_t offset;
};

struct gguf_file_t {
    // The header of the file.
    gguf_header_t header;

    // Tensor infos, which can be used to locate the tensor data.
    gguf_tensor_info_t tensor_infos[header.tensor_count];

    // Padding to the nearest multiple of `ALIGNMENT`.
    //
    // That is, if `sizeof(header) + sizeof(tensor_infos)` is not a multiple of `ALIGNMENT`,
    // this padding is added to make it so.
    //
    // This can be calculated as `align_offset(position) - position`, where `position` is
    // the position of the end of `tensor_infos` (i.e. `sizeof(header) + sizeof(tensor_infos)`).
    uint8_t _padding[];

    // Tensor data.
    //
    // This is arbitrary binary data corresponding to the weights of the model. This data should be close
    // or identical to the data in the original model file, but may be different due to quantization or
    // other optimizations for inference. Any such deviations should be recorded in the metadata or as
    // part of the architecture definition.
    //
    // Each tensor's data must be stored within this array, and located through its `tensor_infos` entry.
    // The offset of each tensor's data must be a multiple of `ALIGNMENT`, and the space between tensors
    // should be padded to `ALIGNMENT` bytes.
    uint8_t tensor_data[];
};
```

## Standardized key-value pairs

The following key-value pairs are standardized. This list may grow in the future as more use cases are discovered. Where possible, names are shared with the original model definitions to make it easier to map between the two.

Not all of these are required, but they are all recommended. Keys that are required are bolded. For omitted pairs, the reader should assume that the value is unknown and either default or error as appropriate.

The community can develop their own key-value pairs to carry additional data. However, these should be namespaced with the relevant community name to avoid collisions. For example, the `rustformers` community might use `rustformers.` as a prefix for all of their keys.

If a particular community key is widely used, it may be promoted to a standardized key.

By convention, most counts/lengths/etc are `uint64` unless otherwise specified. This is to allow for larger models to be supported in the future. Some models may use `uint32` for their values; it is recommended that readers support both.

### General

#### Required

- **`general.architecture: string`**: describes what architecture this model implements. All lowercase ASCII, with only `[a-z0-9]+` characters allowed. Known values include:
  - `llama`
  - `mpt`
  - `gptneox`
  - `gptj`
  - `gpt2`
  - `bloom`
  - `falcon`
  - `rwkv`
- **`general.quantization_version: uint32`**: The version of the quantization format. Not required if the model is not quantized (i.e. no tensors are quantized). If any tensors are quantized, this _must_ be present. This is separate to the quantization scheme of the tensors itself; the quantization version may change without changing the scheme's name (e.g. the quantization scheme is Q5_K, and the quantization version is 4).
- **`general.alignment: uint32`**: the global alignment to use, as described above. This can vary to allow for different alignment schemes, but it must be a multiple of 8. Some writers may not write the alignment. If the alignment is **not** specified, assume it is `32`.

#### General metadata

- `general.name`: The name of the model. This should be a human-readable name that can be used to identify the model. It should be unique within the community that the model is defined in.
- `general.author`: The author of the model.
- `general.url`: URL to the model's homepage. This can be a GitHub repo, a paper, etc.
- `general.description: string`: free-form description of the model including anything that isn't covered by the other fields
- `general.license: string`: License of the model, expressed as a [SPDX license expression](https://spdx.github.io/spdx-spec/v2-draft/SPDX-license-expressions/) (e.g. `"MIT OR Apache-2.0`). Do not include any other information, such as the license text or the URL to the license.
- `general.file_type: uint32`: An enumerated value describing the type of the majority of the tensors in the file. Optional; can be inferred from the tensor types.
  - `ALL_F32 = 0`
  - `MOSTLY_F16 = 1`
  - `MOSTLY_Q4_0 = 2`
  - `MOSTLY_Q4_1 = 3`
  - `MOSTLY_Q4_1_SOME_F16 = 4`
  - `MOSTLY_Q4_2 = 5` (support removed)
  - `MOSTLY_Q4_3 = 6` (support removed)
  - `MOSTLY_Q8_0 = 7`
  - `MOSTLY_Q5_0 = 8`
  - `MOSTLY_Q5_1 = 9`
  - `MOSTLY_Q2_K = 10`
  - `MOSTLY_Q3_K_S = 11`
  - `MOSTLY_Q3_K_M = 12`
  - `MOSTLY_Q3_K_L = 13`
  - `MOSTLY_Q4_K_S = 14`
  - `MOSTLY_Q4_K_M = 15`
  - `MOSTLY_Q5_K_S = 16`
  - `MOSTLY_Q5_K_M = 17`
  - `MOSTLY_Q6_K = 18`

#### Source metadata

Information about where this model came from. This is useful for tracking the provenance of the model, and for finding the original source if the model is modified. For a model that was converted from GGML, for example, these keys would point to the model that was converted from.

- `general.source.url: string`: URL to the source of the model. Can be a GitHub repo, a paper, etc.
- `general.source.huggingface.repository: string`: Hugging Face model repository that this model is either hosted on or based on

### LLM

In the following, `[llm]` is used to fill in for the name of a specific LLM architecture. For example, `llama` for LLaMA, `mpt` for MPT, etc. If mentioned in an architecture's section, it is required for that architecture, but not all keys are required for all architectures. Consult the relevant section for more information.

- `[llm].context_length: uint64`: Also known as `n_ctx`. length of the context (in tokens) that the model was trained on. For most architectures, this is the hard limit on the length of the input. Architectures, like RWKV, that are not reliant on transformer-style attention may be able to handle larger inputs, but this is not guaranteed.
- `[llm].embedding_length: uint64`: Also known as `n_embd`. Embedding layer size.
- `[llm].block_count: uint64`: The number of blocks of attention+feed-forward layers (i.e. the bulk of the LLM). Does not include the input or embedding layers.
- `[llm].feed_forward_length: uint64`: Also known as `n_ff`. The length of the feed-forward layer.
- `[llm].use_parallel_residual: bool`: Whether or not the parallel residual logic should be used.
- `[llm].tensor_data_layout: string`: When a model is converted to GGUF, tensors may be rearranged to improve performance. This key describes the layout of the tensor data. This is not required; if not present, it is assumed to be `reference`.
  - `reference`: tensors are laid out in the same order as the original model
  - further options can be found for each architecture in their respective sections

#### Attention

- `[llm].attention.head_count: uint64`: Also known as `n_head`. Number of attention heads.
- `[llm].attention.head_count_kv: uint64`: The number of heads per group used in Grouped-Query-Attention. If not present or if present and equal to `[llm].attention.head_count`, the model does not use GQA.
- `[llm].attention.max_alibi_bias: float32`: The maximum bias to use for ALiBI.
- `[llm].attention.clamp_kqv: float32`: Value (`C`) to clamp the values of the `Q`, `K`, and `V` tensors between (`[-C, C]`).
- `[llm].attention.layer_norm_epsilon: float32`: Layer normalization epsilon.
- `[llm].attention.layer_norm_rms_epsilon: float32`: Layer RMS normalization epsilon.

#### RoPE

- `[llm].rope.dimension_count: uint64`: The number of rotary dimensions for RoPE.
- `[llm].rope.freq_base: float32`: The base frequency for RoPE.

##### Scaling

The following keys describe RoPE scaling parameters:

- `[llm].rope.scaling.type: string`: Can be `none`, `linear`, or `yarn`.
- `[llm].rope.scaling.factor: float32`: A scale factor for RoPE to adjust the context length.
- `[llm].rope.scaling.original_context_length: uint32_t`: The original context length of the base model.
- `[llm].rope.scaling.finetuned: bool`: True if model has been finetuned with RoPE scaling.

Note that older models may not have these keys, and may instead use the following key:

- `[llm].rope.scale_linear: float32`: A linear scale factor for RoPE to adjust the context length.

It is recommended that models use the newer keys if possible, as they are more flexible and allow for more complex scaling schemes. Executors will need to support both indefinitely.

#### Models

The following sections describe the metadata for each model architecture. Each key specified _must_ be present.

##### LLaMA

- `llama.context_length`
- `llama.embedding_length`
- `llama.block_count`
- `llama.feed_forward_length`
- `llama.rope.dimension_count`
- `llama.attention.head_count`
- `llama.attention.layer_norm_rms_epsilon`

###### Optional

- `llama.rope.scale`
- `llama.attention.head_count_kv`
- `llama.tensor_data_layout`:
  - `Meta AI original pth`:
    ```cpppython
    def permute(weights: NDArray, n_head: int) -> NDArray:
        return (weights.reshape(n_head, 2, weights.shape[0] // n_head // 2, *weights.shape[1:])
                    .swapaxes(1, 2)
                    .reshape(weights.shape))
    ```

##### MPT

- `mpt.context_length`
- `mpt.embedding_length`
- `mpt.block_count`
- `mpt.attention.head_count`
- `mpt.attention.alibi_bias_max`
- `mpt.attention.clip_kqv`
- `mpt.attention.layer_norm_epsilon`

##### GPT-NeoX

- `gptneox.context_length`
- `gptneox.embedding_length`
- `gptneox.block_count`
- `gptneox.use_parallel_residual`
- `gptneox.rope.dimension_count`
- `gptneox.attention.head_count`
- `gptneox.attention.layer_norm_epsilon`

###### Optional

- `gptneox.rope.scale`

##### GPT-J

- `gptj.context_length`
- `gptj.embedding_length`
- `gptj.block_count`
- `gptj.rope.dimension_count`
- `gptj.attention.head_count`
- `gptj.attention.layer_norm_epsilon`

###### Optional

- `gptj.rope.scale`

##### GPT-2

- `gpt2.context_length`
- `gpt2.embedding_length`
- `gpt2.block_count`
- `gpt2.attention.head_count`
- `gpt2.attention.layer_norm_epsilon`

##### BLOOM

- `bloom.context_length`
- `bloom.embedding_length`
- `bloom.block_count`
- `bloom.feed_forward_length`
- `bloom.attention.head_count`
- `bloom.attention.layer_norm_epsilon`

##### Falcon

- `falcon.context_length`
- `falcon.embedding_length`
- `falcon.block_count`
- `falcon.attention.head_count`
- `falcon.attention.head_count_kv`
- `falcon.attention.use_norm`
- `falcon.attention.layer_norm_epsilon`

###### Optional

- `falcon.tensor_data_layout`:

  - `jploski` (author of the original GGML implementation of Falcon):

    ```cpppython
    # The original query_key_value tensor contains n_head_kv "kv groups",
    # each consisting of n_head/n_head_kv query weights followed by one key
    # and one value weight (shared by all query heads in the kv group).
    # This layout makes it a big pain to work with in GGML.
    # So we rearrange them here,, so that we have n_head query weights
    # followed by n_head_kv key weights followed by n_head_kv value weights,
    # in contiguous fashion.

    if "query_key_value" in src:
        qkv = model[src].view(
            n_head_kv, n_head // n_head_kv + 2, head_dim, head_dim * n_head)

        q = qkv[:, :-2 ].reshape(n_head * head_dim, head_dim * n_head)
        k = qkv[:, [-2]].reshape(n_head_kv * head_dim, head_dim * n_head)
        v = qkv[:, [-1]].reshape(n_head_kv * head_dim, head_dim * n_head)

        model[src] = torch.cat((q,k,v)).reshape_as(model[src])
    ```

##### RWKV

The vocabulary size is the same as the number of rows in the `head` matrix.

- `rwkv.architecture_version: uint32`: The only allowed value currently is 4. Version 5 is expected to appear some time in the future.
- `rwkv.context_length: uint64`: Length of the context used during training or fine-tuning. RWKV is able to handle larger context than this limit, but the output quality may suffer.
- `rwkv.block_count: uint64`
- `rwkv.embedding_length: uint64`
- `rwkv.feed_forward_length: uint64`

##### Whisper

Keys that do not have types defined should be assumed to share definitions with `llm.` keys.
(For example, `whisper.context_length` is equivalent to `llm.context_length`.)
This is because they are both transformer models.

- `whisper.encoder.context_length`
- `whisper.encoder.embedding_length`
- `whisper.encoder.block_count`
- `whisper.encoder.mels_count: uint64`
- `whisper.encoder.attention.head_count`

- `whisper.decoder.context_length`
- `whisper.decoder.embedding_length`
- `whisper.decoder.block_count`
- `whisper.decoder.attention.head_count`

#### Prompting

**TODO**: Include prompt format, and/or metadata about how it should be used (instruction, conversation, autocomplete, etc).

### LoRA

**TODO**: Figure out what metadata is needed for LoRA. Probably desired features:

- match an existing model exactly, so that it can't be misapplied
- be marked as a LoRA so executors won't try to run it by itself

Should this be an architecture, or should it share the details of the original model with additional fields to mark it as a LoRA?

### Tokenizer

The following keys are used to describe the tokenizer of the model. It is recommended that model authors support as many of these as possible, as it will allow for better tokenization quality with supported executors.

#### GGML

GGML supports an embedded vocabulary that enables inference of the model, but implementations of tokenization using this vocabulary (i.e. `llama.cpp`'s tokenizer) may have lower accuracy than the original tokenizer used for the model. When a more accurate tokenizer is available and supported, it should be used instead.

It is not guaranteed to be standardized across models, and may change in the future. It is recommended that model authors use a more standardized tokenizer if possible.

- `tokenizer.ggml.model: string`: The name of the tokenizer model.
  - `llama`: Llama style SentencePiece (tokens and scores extracted from HF `tokenizer.model`)
  - `replit`: Replit style SentencePiece (tokens and scores extracted from HF `spiece.model`)
  - `gpt2`: GPT-2 / GPT-NeoX style BPE (tokens extracted from HF `tokenizer.json`)
  - `rwkv`: RWKV tokenizer
- `tokenizer.ggml.tokens: array[string]`: A list of tokens indexed by the token ID used by the model.
- `tokenizer.ggml.scores: array[float32]`: If present, the score/probability of each token. If not present, all tokens are assumed to have equal probability. If present, it must have the same length and index as `tokens`.
- `tokenizer.ggml.token_type: array[int32]`: The token type (1=normal, 2=unknown, 3=control, 4=user defined, 5=unused, 6=byte). If present, it must have the same length and index as `tokens`.
- `tokenizer.ggml.merges: array[string]`: If present, the merges of the tokenizer. If not present, the tokens are assumed to be atomic.
- `tokenizer.ggml.added_tokens: array[string]`: If present, tokens that were added after training.

##### Special tokens

- `tokenizer.ggml.bos_token_id: uint32`: Beginning of sequence marker
- `tokenizer.ggml.eos_token_id: uint32`: End of sequence marker
- `tokenizer.ggml.unknown_token_id: uint32`: Unknown token
- `tokenizer.ggml.separator_token_id: uint32`: Separator token
- `tokenizer.ggml.padding_token_id: uint32`: Padding token

#### Hugging Face

Hugging Face maintains their own `tokenizers` library that supports a wide variety of tokenizers. If your executor uses this library, it may be able to use the model's tokenizer directly.

- `tokenizer.huggingface.json: string`: the entirety of the HF `tokenizer.json` for a given model (e.g. <https://huggingface.co/mosaicml/mpt-7b-instruct/blob/main/tokenizer.json>). Included for compatibility with executors that support HF tokenizers directly.

#### Other

Other tokenizers may be used, but are not necessarily standardized. They may be executor-specific. They will be documented here as they are discovered/further developed.

- `tokenizer.rwkv.world: string`: a RWKV World tokenizer, like [this](https://github.com/BlinkDL/ChatRWKV/blob/main/tokenizer/rwkv_vocab_v20230424.txt). This text file should be included verbatim.

### Computation graph

This is a future extension and still needs to be discussed, and may necessitate a new GGUF version. At the time of writing, the primary blocker is the stabilization of the computation graph format.

A sample computation graph of GGML nodes could be included in the model itself, allowing an executor to run the model without providing its own implementation of the architecture. This would allow for a more consistent experience across executors, and would allow for more complex architectures to be supported without requiring the executor to implement them.

## Standardized tensor names

To minimize complexity and maximize compatibility, it is recommended that models using the transformer architecture use the following naming convention for their tensors:

### Base layers

`AA.weight` `AA.bias`

where `AA` can be:

- `token_embd`: Token embedding layer
- `pos_embd`: Position embedding layer
- `output_norm`: Output normalization layer
- `output`: Output layer

### Attention and feed-forward layer blocks

`blk.N.BB.weight` `blk.N.BB.bias`

where N signifies the block number a layer belongs to, and where `BB` could be:

- `attn_norm`: Attention normalization layer
- `attn_norm_2`: Attention normalization layer
- `attn_qkv`: Attention query-key-value layer
- `attn_q`: Attention query layer
- `attn_k`: Attention key layer
- `attn_v`: Attention value layer
- `attn_output`: Attention output layer

- `ffn_norm`: Feed-forward network normalization layer
- `ffn_up`: Feed-forward network "up" layer
- `ffn_gate`: Feed-forward network "gate" layer
- `ffn_down`: Feed-forward network "down" layer

## Version History

This document is actively updated to describe the current state of the metadata, and these changes are not tracked outside of the commits.

However, the format _itself_ has changed. The following sections describe the changes to the format itself.

### v3

Adds big-endian support.

### v2

Most countable values (lengths, etc) were changed from `uint32` to `uint64` to allow for larger models to be supported in the future.

### v1

Initial version.

## Historical State of Affairs

The following information is provided for context, but is not necessary to understand the rest of this document.

### Overview

At present, there are three GGML file formats floating around for LLMs:

- **GGML** (unversioned): baseline format, with no versioning or alignment.
- **GGMF** (versioned): the same as GGML, but with versioning. Only one version exists.
- **GGJT**: Aligns the tensors to allow for use with `mmap`, which requires alignment. v1, v2 and v3 are identical, but the latter versions use a different quantization scheme that is incompatible with previous versions.

GGML is primarily used by the examples in `ggml`, while GGJT is used by `llama.cpp` models. Other executors may use any of the three formats, but this is not 'officially' supported.

These formats share the same fundamental structure:

- a magic number with an optional version number
- model-specific hyperparameters, including
  - metadata about the model, such as the number of layers, the number of heads, etc.
  - a `ftype` that describes the type of the majority of the tensors,
    - for GGML files, the quantization version is encoded in the `ftype` divided by 1000
- an embedded vocabulary, which is a list of strings with length prepended. The GGMF/GGJT formats embed a float32 score next to the strings.
- finally, a list of tensors with their length-prepended name, type, and (aligned, in the case of GGJT) tensor data

Notably, this structure does not identify what model architecture the model belongs to, nor does it offer any flexibility for changing the structure of the hyperparameters. This means that the only way to add new hyperparameters is to add them to the end of the list, which is a breaking change for existing models.

### Drawbacks

Unfortunately, over the last few months, there are a few issues that have become apparent with the existing models:

- There's no way to identify which model architecture a given model is for, because that information isn't present
  - Similarly, existing programs cannot intelligently fail upon encountering new architectures
- Adding or removing any new hyperparameters is a breaking change, which is impossible for a reader to detect without using heuristics
- Each model architecture requires its own conversion script to their architecture's variant of GGML
- Maintaining backwards compatibility without breaking the structure of the format requires clever tricks, like packing the quantization version into the ftype, which are not guaranteed to be picked up by readers/writers, and are not consistent between the two formats

### Why not other formats?

There are a few other formats that could be used, but issues include:

- requiring additional dependencies to load or save the model, which is complicated in a C environment
- limited or no support for 4-bit quantization
- existing cultural expectations (e.g. whether or not the model is a directory or a file)
- lack of support for embedded vocabularies
- lack of control over direction of future development

Ultimately, it is likely that GGUF will remain necessary for the foreseeable future, and it is better to have a single format that is well-documented and supported by all executors than to contort an existing format to fit the needs of GGML.


# `examples/common-ggml.cpp`

这段代码包括两个部分，用于输出不同数据类型的最主要是的尺寸的记分单位(scale)。

首先，引入了两个头文件："common-ggml.h"和"regex"。然后，引入了一个头文件"map"。

接着，定义了一个类"ggml_ftype_map"，其中包含一个包含常用数据类型的记分单位的元组，类型键是字符串常量，记分单位是整数常量。

然后，定义了一个函数"ggml_print_ftypes"，这个函数接受一个文件指针作为参数，然后遍历存储在"ggml_ftype_map"中的键值对，将每个键和值打印到文件中。

在函数内部，使用for循环遍历所有的键值对，使用fprintf函数将每个键和值打印到文件中。

最后，包含头文件"map"，以便在需要使用这个类时进行头文件搜索。


```cpp
#include "common-ggml.h"

#include <regex>
#include <map>

static const std::map<std::string, enum ggml_ftype> GGML_FTYPE_MAP = {
    {"q4_0", GGML_FTYPE_MOSTLY_Q4_0},
    {"q4_1", GGML_FTYPE_MOSTLY_Q4_1},
    {"q5_0", GGML_FTYPE_MOSTLY_Q5_0},
    {"q5_1", GGML_FTYPE_MOSTLY_Q5_1},
    {"q8_0", GGML_FTYPE_MOSTLY_Q8_0},
};

void ggml_print_ftypes(FILE * fp) {
    for (auto it = GGML_FTYPE_MAP.begin(); it != GGML_FTYPE_MAP.end(); it++) {
        fprintf(fp, "  type = \"%s\" or %d\n", it->first.c_str(), it->second);
    }
}

```

这段代码定义了一个名为 `ggml_ftype` 的枚举类型 `ggml_ftype`，其中包含四个成员变量： 

1. `ggml_parse_ftype` 函数，它接受一个字符串参数 `str`。
2. 枚举类型变量 `ftype`，用于存储输入数据后的类型信息。
3. 函数参数 `const char * str` 是一个指向字符串的指针，用于存储输入数据。
4. 函数返回值 `ggml_ftype` 表示输入数据解析后的类型信息。

函数 `ggml_parse_ftype` 的实现方式如下：

1. 如果输入字符串以 'q' 开头，则说明要解析的类型是 Q 类型，直接使用映射表中的定义即可。
2. 如果输入字符串以其他字符开头，则尝试解析输入字符串为整数类型。如果解析成功，则存储解析得到的整数类型类型信息。

如果解析失败或者输入字符串无法解析，函数将输出错误信息并返回 `GGML_FTYPE_UNKNOWN`。


```cpp
enum ggml_ftype ggml_parse_ftype(const char * str) {
    enum ggml_ftype ftype;
    if (str[0] == 'q') {
        const auto it = GGML_FTYPE_MAP.find(str);
        if (it == GGML_FTYPE_MAP.end()) {
            fprintf(stderr, "%s: unknown ftype '%s'\n", __func__, str);
            return GGML_FTYPE_UNKNOWN;
        }
        ftype = it->second;
    } else {
        ftype = (enum ggml_ftype) atoi(str);
    }

    return ftype;
}

```

This is a C++ function that calculates the size of a model and quant, and writes the result to a file. The model size is calculated in bytes, while the quant size is calculated as the size of the quant content in bytes, divided by the number of elements in the quant. The function also includes information about the data type of the quant, and the name of the data type.

The function takes two arguments: one is a pointer to a user-defined data type object, and the other is a pointer to a file stream. The data type object is used to read the data from the input, and the file stream is used to write the result to the output.

The function first sets the total size of the model to the size of the entire data type object, and then sets the total size of the quant to the size of the quant content. The function then loops through the elements of the hist data type object and calculates the size of each element, adding the size of each element to the sum of all the elements in the hist data type object.

The function then writes the result to the file stream, including the total size of the model, the total size of the quant, and the data type of the quant. Finally, the function returns true to indicate that the calculation was successful.


```cpp
bool ggml_common_quantize_0(
        std::ifstream & finp,
        std::ofstream & fout,
        const ggml_ftype ftype,
        const std::vector<std::string> & to_quant,
        const std::vector<std::string> & to_skip) {

    ggml_type qtype = GGML_TYPE_F32;

    switch (ftype) {
        case GGML_FTYPE_MOSTLY_Q4_0: qtype = GGML_TYPE_Q4_0; break;
        case GGML_FTYPE_MOSTLY_Q4_1: qtype = GGML_TYPE_Q4_1; break;
        case GGML_FTYPE_MOSTLY_Q5_0: qtype = GGML_TYPE_Q5_0; break;
        case GGML_FTYPE_MOSTLY_Q5_1: qtype = GGML_TYPE_Q5_1; break;
        case GGML_FTYPE_MOSTLY_Q8_0: qtype = GGML_TYPE_Q8_0; break;
        case GGML_FTYPE_UNKNOWN:
        case GGML_FTYPE_ALL_F32:
        case GGML_FTYPE_MOSTLY_F16:
        case GGML_FTYPE_MOSTLY_Q4_1_SOME_F16:
        case GGML_FTYPE_MOSTLY_Q2_K:
        case GGML_FTYPE_MOSTLY_Q3_K:
        case GGML_FTYPE_MOSTLY_Q4_K:
        case GGML_FTYPE_MOSTLY_Q5_K:
        case GGML_FTYPE_MOSTLY_Q6_K:
                {
                    fprintf(stderr, "%s: invalid model type %d\n", __func__, ftype);
                    return false;
                }
    };

    if (!ggml_is_quantized(qtype)) {
        fprintf(stderr, "%s: invalid quantization type %d (%s)\n", __func__, qtype, ggml_type_name(qtype));
        return false;
    }

    size_t total_size_org = 0;
    size_t total_size_new = 0;

    std::vector<float> work;

    std::vector<uint8_t>     data_u8;
    std::vector<ggml_fp16_t> data_f16;
    std::vector<float>       data_f32;

    std::vector<int64_t> hist_all(1 << 4, 0);

    while (true) {
        int32_t n_dims;
        int32_t length;
        int32_t ttype;

        finp.read(reinterpret_cast<char *>(&n_dims), sizeof(n_dims));
        finp.read(reinterpret_cast<char *>(&length), sizeof(length));
        finp.read(reinterpret_cast<char *>(&ttype),  sizeof(ttype));

        if (finp.eof()) {
            break;
        }

        int32_t nelements = 1;
        int32_t ne[4] = { 1, 1, 1, 1 };
        for (int i = 0; i < n_dims; ++i) {
            finp.read (reinterpret_cast<char *>(&ne[i]), sizeof(ne[i]));
            nelements *= ne[i];
        }

        std::string name(length, 0);
        finp.read (&name[0], length);

        printf("%64s - [%5d, %5d, %5d], type = %6s ", name.data(), ne[0], ne[1], ne[2], ggml_type_name((ggml_type) ttype));

        bool quantize = false;

        // check if we should quantize this tensor
        for (const auto & s : to_quant) {
            if (std::regex_match(name, std::regex(s))) {
                quantize = true;
                break;
            }
        }

        // check if we should skip this tensor
        for (const auto & s : to_skip) {
            if (std::regex_match(name, std::regex(s))) {
                quantize = false;
                break;
            }
        }

        // quantize only 2D tensors
        quantize &= (n_dims == 2);

        if (quantize) {
            if (ttype != GGML_TYPE_F32 && ttype != GGML_TYPE_F16) {
                fprintf(stderr, "%s: unsupported ttype %d (%s) for integer quantization\n", __func__, ttype, ggml_type_name((ggml_type) ttype));
                return false;
            }

            if (ttype == GGML_TYPE_F16) {
                data_f16.resize(nelements);
                finp.read(reinterpret_cast<char *>(data_f16.data()), nelements * sizeof(ggml_fp16_t));
                data_f32.resize(nelements);
                for (int i = 0; i < nelements; ++i) {
                    data_f32[i] = ggml_fp16_to_fp32(data_f16[i]);
                }
            } else {
                data_f32.resize(nelements);
                finp.read(reinterpret_cast<char *>(data_f32.data()), nelements * sizeof(float));
            }

            ttype = qtype;
        } else {
            const int bpe = (ttype == 0) ? sizeof(float) : sizeof(uint16_t);

            data_u8.resize(nelements*bpe);
            finp.read(reinterpret_cast<char *>(data_u8.data()), nelements * bpe);
        }

        fout.write(reinterpret_cast<char *>(&n_dims), sizeof(n_dims));
        fout.write(reinterpret_cast<char *>(&length), sizeof(length));
        fout.write(reinterpret_cast<char *>(&ttype),  sizeof(ttype));
        for (int i = 0; i < n_dims; ++i) {
            fout.write(reinterpret_cast<char *>(&ne[i]), sizeof(ne[i]));
        }
        fout.write(&name[0], length);

        if (quantize) {
            work.resize(nelements); // for quantization

            size_t cur_size = 0;
            std::vector<int64_t> hist_cur(1 << 4, 0);

            switch ((ggml_type) ttype) {
                case GGML_TYPE_Q4_0:
                    {
                        cur_size = ggml_quantize_q4_0(data_f32.data(), work.data(), nelements, ne[0], hist_cur.data());
                    } break;
                case GGML_TYPE_Q4_1:
                    {
                        cur_size = ggml_quantize_q4_1(data_f32.data(), work.data(), nelements, ne[0], hist_cur.data());
                    } break;
                case GGML_TYPE_Q5_0:
                    {
                        cur_size = ggml_quantize_q5_0(data_f32.data(), work.data(), nelements, ne[0], hist_cur.data());
                    } break;
                case GGML_TYPE_Q5_1:
                    {
                        cur_size = ggml_quantize_q5_1(data_f32.data(), work.data(), nelements, ne[0], hist_cur.data());
                    } break;
                case GGML_TYPE_Q8_0:
                    {
                        cur_size = ggml_quantize_q8_0(data_f32.data(), work.data(), nelements, ne[0], hist_cur.data());
                    } break;
                case GGML_TYPE_F32:
                case GGML_TYPE_F16:
                case GGML_TYPE_I8:
                case GGML_TYPE_I16:
                case GGML_TYPE_I32:
                case GGML_TYPE_Q8_1:
                case GGML_TYPE_Q2_K:
                case GGML_TYPE_Q3_K:
                case GGML_TYPE_Q4_K:
                case GGML_TYPE_Q5_K:
                case GGML_TYPE_Q6_K:
                case GGML_TYPE_Q8_K:
                case GGML_TYPE_COUNT:
                    {
                        fprintf(stderr, "%s: unsupported quantization type %d (%s)\n", __func__, ttype, ggml_type_name((ggml_type) ttype));
                        return false;
                    }
            }

            fout.write(reinterpret_cast<char *>(work.data()), cur_size);
            total_size_new += cur_size;

            printf("size = %8.2f MB -> %8.2f MB | hist: ", nelements * sizeof(float)/1024.0/1024.0, cur_size/1024.0/1024.0);
            for (int i = 0; i < (int) hist_cur.size(); ++i) {
                hist_all[i] += hist_cur[i];
            }

            for (int i = 0; i < (int) hist_cur.size(); ++i) {
                printf("%5.3f ", hist_cur[i] / (float)nelements);
            }
            printf("\n");
        } else {
            printf("size = %8.3f MB\n", data_u8.size()/1024.0/1024.0);
            fout.write(reinterpret_cast<char *>(data_u8.data()), data_u8.size());
            total_size_new += data_u8.size();
        }

        total_size_org += nelements * sizeof(float);
    }

    printf("%s: model size  = %8.2f MB\n", __func__, total_size_org/1024.0/1024.0);
    printf("%s: quant size  = %8.2f MB | ftype = %d (%s)\n", __func__, total_size_new/1024.0/1024.0, ftype, ggml_type_name(qtype));

    {
        int64_t sum_all = 0;
        for (int i = 0; i < (int) hist_all.size(); ++i) {
            sum_all += hist_all[i];
        }

        printf("%s: hist: ", __func__);
        for (int i = 0; i < (int) hist_all.size(); ++i) {
            printf("%5.3f ", hist_all[i] / (float)sum_all);
        }
        printf("\n");
    }

    return true;
}

```

# `examples/common.cpp`

这段代码定义了一个宏，名为`_USE_MATH_DEFINES`，使用了`#define`预处理指令。它的作用是在编译时定义，而不是在使用时定义，这样就可以在需要时动态地链接使用。这个宏定义了一个代表`M_PI`的常量，这意味着在任何地方只要需要用到`M_PI`，就可以直接使用这个宏定义，而不必去查找`M_PI`的定义。

接下来的代码是一个头文件，名为`common.h`。可能是为了和其他源文件保持一致而定义的。这个头文件中定义了一些通用的函数和变量，可能是为了在整个程序中进行全局的使用而定义的。

然后是第三方库的头文件，名为`dr_wav.h`。可能是为了在程序中使用而定义的。这个头文件中定义了一些函数，可能是来自第三方库，用于对WAV格式的音频文件进行操作。

接下来定义了一个`DR_WAV_IMPLEMENTATION`宏，这个宏使用了一个未定义的实现，也就是一个空字符串，表示这个宏所代表的方法不存在。

接下来定义了一些数学函数，包括`cmath`库中的`sin`函数、`cos`函数、`sqrt`函数、`tan`函数，还有`regex`库中的`regex_match`函数。这些函数可能是在程序中需要使用时定义的。

接着定义了一些`<cmath>`、`<cstring>`、`<fstream>`和`<locale>`头文件，这些头文件中定义了一些常用的数学函数和字符串操作函数，可能是在程序中需要使用时定义的。

然后是两个`#include`语句，分别引入了`common.h`和`dr_wav.h`头文件。这样，`common.h`中定义的全局函数和变量就可以在程序中使用，而`dr_wav.h`中定义的函数则可以在这个特定的库中使用。

最后是定义了一个`printf`函数，这个函数可能是在程序中需要输出一些信息时定义的，例如`printf`函数可以将一个`const char*`类型的参数输出。


```cpp
#define _USE_MATH_DEFINES // for M_PI

#include "common.h"

// third-party utilities
// use your favorite implementations
#define DR_WAV_IMPLEMENTATION
#include "dr_wav.h"

#include <cmath>
#include <cstring>
#include <fstream>
#include <regex>
#include <locale>
#include <codecvt>
```

这段代码的作用是定义了一个名为`get_next_arg`的函数，用于获取下一个参数。函数接受四个参数：

1. 整型变量`i`，表示当前参数的位置。
2. 整型变量`argc`，表示命令行参数总数。
3. 指向字符数组的指针`argv`，保存了所有命令行参数的副本。
4. 常量引用`flag`，用于指示输入的参数是来自哪个程序。
5. 结构体`gpt_params`，该结构体包含了所有参数和选项的定义。

函数的作用是：

1. 如果`i+1`后的参数存在且不是`/`或`-`，则返回`argv[i+1]`。
2. 否则，在`std::string`类型的输入中，输出错误消息，并返回`gpt_print_usage`函数，该函数将打印出所有参数的列表，并传递参数总数和参数数组。
3. 如果`i+1`后的参数不存在，在`std::string`类型的输入中，输出错误消息，并返回`0`。


```cpp
#include <sstream>

#if defined(_MSC_VER)
#pragma warning(disable: 4244 4267) // possible loss of data
#endif

// Function to check if the next argument exists
std::string get_next_arg(int& i, int argc, char** argv, const std::string& flag, gpt_params& params) {
    if (i + 1 < argc && argv[i + 1][0] != '-') {
        return argv[++i];
    } else {
        fprintf(stderr, "error: %s requires one argument.\n", flag.c_str());
        gpt_print_usage(argc, argv, params);
        exit(0);
    }
}

```

This is a command-line tool that uses the GPT-2 API to generate human-like text. It allows you to specify various parameters to control the tool.

Here's a breakdown of the available parameters and their usage:

* `--gpu`: This flag enables GPU usage for training the model. It is recommended to have a NVIDIA GPU with at least 8GB of memory to use this flag.
* `--gpu-layers`: This flag is used to specify which parts of the model to train on the GPU. It is a lower-case怆士卜语言， and it should have the same effect as `--gpu`.
* `--model`: This flag specifies the name of the pre-trained model to use. It should be the name of a file in the `./models` directory, including the model architecture and the timestamp.
* `--interactive`: This flag allows you to interact with the model through an interactive terminal. It is required to have TensorFlow or PyTorch installed to work.
* `--ip`: This flag is similar to `--interactive`, but it specifies an IP address instead of a port number.
* `--file`: This flag specifies a file to read input from. It should be a text file that contains the input text.
* `--ignore-eos`: This flag is passed when running `--file` or `--ip`. It tells the tool to ignore the end-of-sequence token and continue reading input from the file.
* `--h`: This flag is used for the help message.

Here's an example usage:
```cppperl
gpt-2 generate --gpu --model my_model.pth --interactive
```
This command will train the last 4 layers of the model on the GPU, and display an interactive terminal that allows you to interact with the model.


```cpp
bool gpt_params_parse(int argc, char ** argv, gpt_params & params) {
    for (int i = 1; i < argc; i++) {
        std::string arg = argv[i];

        if (arg == "-s" || arg == "--seed") {
            params.seed = std::stoi(get_next_arg(i, argc, argv, arg, params));
        } else if (arg == "-t" || arg == "--threads") {
            params.n_threads = std::stoi(get_next_arg(i, argc, argv, arg, params));
        } else if (arg == "-p" || arg == "--prompt") {
            params.prompt = get_next_arg(i, argc, argv, arg, params);
        } else if (arg == "-n" || arg == "--n_predict") {
            params.n_predict = std::stoi(get_next_arg(i, argc, argv, arg, params));
        } else if (arg == "-np" || arg == "--n_parallel") {
            params.n_parallel = std::stoi(get_next_arg(i, argc, argv, arg, params));
        } else if (arg == "--top_k") {
            params.top_k = std::stoi(get_next_arg(i, argc, argv, arg, params));
        } else if (arg == "--top_p") {
            params.top_p = std::stof(get_next_arg(i, argc, argv, arg, params));
        } else if (arg == "--temp") {
            params.temp = std::stof(get_next_arg(i, argc, argv, arg, params));
        } else if (arg == "--repeat-last-n") {
            params.repeat_last_n = std::stoi(get_next_arg(i, argc, argv, arg, params));
        } else if (arg == "--repeat-penalty") {
            params.repeat_penalty = std::stof(get_next_arg(i, argc, argv, arg, params));
        } else if (arg == "-b" || arg == "--batch_size") {
            params.n_batch= std::stoi(get_next_arg(i, argc, argv, arg, params));
        } else if (arg == "-c" || arg == "--context") {
            params.n_ctx= std::stoi(get_next_arg(i, argc, argv, arg, params));
        } else if (arg == "-ngl" || arg == "--gpu-layers" || arg == "--n-gpu-layers") {
            params.n_gpu_layers = std::stoi(get_next_arg(i, argc, argv, arg, params));
        } else if (arg == "--ignore-eos") {
            params.ignore_eos = true;
        } else if (arg == "-m" || arg == "--model") {
            params.model = get_next_arg(i, argc, argv, arg, params);
        } else if (arg == "-i" || arg == "--interactive") {
            params.interactive = true;
        } else if (arg == "-ip" || arg == "--interactive-port") {
            params.interactive = true;
            params.interactive_port = std::stoi(get_next_arg(i, argc, argv, arg, params));
        } else if (arg == "-h" || arg == "--help") {
            gpt_print_usage(argc, argv, params);
            exit(0);
        } else if (arg == "-f" || arg == "--file") {
            get_next_arg(i, argc, argv, arg, params);
            std::ifstream file(argv[i]);
            if (!file) {
                fprintf(stderr, "error: failed to open file '%s'\n", argv[i]);
                break;
            }
            std::copy(std::istreambuf_iterator<char>(file), std::istreambuf_iterator<char>(), back_inserter(params.prompt));
            if (params.prompt.back() == '\n') {
                params.prompt.pop_back();
            }
        } else if (arg == "-tt" || arg == "--token_test") {
            params.token_test = get_next_arg(i, argc, argv, arg, params);
        }
        else {
            fprintf(stderr, "error: unknown argument: %s\n", arg.c_str());
            gpt_print_usage(argc, argv, params);
            exit(0);
        }
    }

    return true;
}

```

This is a command-line tool called `token_test` that performs tokenization on input files. The tool has several command-line options available:

-   `-tt`: This option specifies the token test algorithm.
-   `--token_test`: This option specifies the token test algorithm to use.
-   `-n N, --n_predict N`: This option specifies the number of tokens to predict. The default value is `params.n_predict` which is an unspecified number.
-   `--top_k N`: This option specifies the number of top-k predictions to make. The default value is `params.top_k` which is an unspecified number of samples.
-   `--top_p N, --top_p N`: These options both specify the number of top-p (predicted) samples to make. The first one is an unspecified number, and the second one is a specified number of samples. The default value is `params.top_p` which is an unspecified number of samples.
-   `--temp N`: This option specifies the temperature for the token test algorithm.
-   `--repeat-last-n N`: This option specifies the number of last n tokens to consider for penalties. The default value is `params.repeat_last_n` which is an unspecified number.
-   `--repeat-penalty N`: This option specifies the penalties for repeat sequences of tokens. The default value is `params.repeat_penalty`, which is an unspecified number.
-   `-b N, --batch-size N`: This option specifies the batch size for prompt processing. The default value is `params.n_batch`, which is an unspecified number.
-   `-c N, --context N`: This option specifies the context / KV cache size. The default value is `params.n_ctx`, which is an unspecified number.
-   `--ignore-eos`: This option specifies whether to ignore EOS tokens during generation.
-   `-ngl N`: This option specifies the number of layers to offload to the GPU on supported models.
-   `-m FNAME`: This option specifies the model to use.
-   `--model FNAME`: This option specifies the path to the model.
-   `-e EOF`: This option specifies the end-of-file (EOF) marker to use as the boundary between the input and output files.
-   `-x`, `-


```cpp
void gpt_print_usage(int /*argc*/, char ** argv, const gpt_params & params) {
    fprintf(stderr, "usage: %s [options]\n", argv[0]);
    fprintf(stderr, "\n");
    fprintf(stderr, "options:\n");
    fprintf(stderr, "  -h, --help            show this help message and exit\n");
    fprintf(stderr, "  -s SEED, --seed SEED  RNG seed (default: -1)\n");
    fprintf(stderr, "  -t N, --threads N     number of threads to use during computation (default: %d)\n", params.n_threads);
    fprintf(stderr, "  -p PROMPT, --prompt PROMPT\n");
    fprintf(stderr, "                        prompt to start generation with (default: random)\n");
    fprintf(stderr, "  -f FNAME, --file FNAME\n");
    fprintf(stderr, "                        load prompt from a file\n");
    fprintf(stderr, "  -tt TOKEN_TEST, --token_test TOKEN_TEST\n");
    fprintf(stderr, "                        test tokenization\n");
    fprintf(stderr, "  -n N, --n_predict N   number of tokens to predict (default: %d)\n", params.n_predict);
    fprintf(stderr, "  --top_k N             top-k sampling (default: %d)\n", params.top_k);
    fprintf(stderr, "  --top_p N             top-p sampling (default: %.1f)\n", params.top_p);
    fprintf(stderr, "  --temp N              temperature (default: %.1f)\n", params.temp);
    fprintf(stderr, "  --repeat-last-n N     last n tokens to consider for penalize (default: %d, 0 = disabled)\n", params.repeat_last_n);
    fprintf(stderr, "  --repeat-penalty N    penalize repeat sequence of tokens (default: %.2f, 1.0 = disabled)\n", (double)params.repeat_penalty);
    fprintf(stderr, "  -b N, --batch_size N  batch size for prompt processing (default: %d)\n", params.n_batch);
    fprintf(stderr, "  -c N, --context N     context / KV cache size (default: %d)\n", params.n_ctx);
    fprintf(stderr, "  --ignore-eos          ignore EOS token during generation\n");
    fprintf(stderr, "  -ngl N, --gpu-layers N  number of layers to offload to GPU on supported models (default: %d)\n", params.n_gpu_layers);
    fprintf(stderr, "  -m FNAME, --model FNAME\n");
    fprintf(stderr, "                        model path (default: %s)\n", params.model.c_str());
    fprintf(stderr, "\n");
}

```

这段代码定义了一个名为`gpt_random_prompt`的函数，它接受一个`std::mt19937`类型的随机整数`rng`作为参数。函数内部使用了一个`const int r = rng() % 10`语句来获取`rng`随机生成的整数，然后使用了一个`switch`语句来判断这个整数所对应的提示语句。

根据这个整数，函数会返回一个字符串。具体来说，如果整数是0，函数将返回"So"；如果整数是1，函数将返回"Once upon a time"；如果整数是2，函数将返回"When"；如果整数是3，函数将返回"The"；如果整数是4，函数将返回"After"；如果整数是5，函数将返回"If"；如果整数是6，函数将返回"import"；如果整数是7，函数将返回"He"；如果整数是8，函数将返回"She"；如果整数是9，函数将返回"They"；如果整数不是0到6或9，函数将返回"To"。

函数的作用是返回一个随机的提示语句，用于在程序中使用`std::mt19937`对象生成随机文本。


```cpp
std::string gpt_random_prompt(std::mt19937 & rng) {
    const int r = rng() % 10;
    switch (r) {
        case 0: return "So";
        case 1: return "Once upon a time";
        case 2: return "When";
        case 3: return "The";
        case 4: return "After";
        case 5: return "If";
        case 6: return "import";
        case 7: return "He";
        case 8: return "She";
        case 9: return "They";
        default: return "To";
    }

    return "The";
}

```

这两段代码主要实现了字符串的 trim 和 replace 操作。trim 操作的作用是移除字符串中的空格，确保输入的字符串是一个空字符串或只包含空格的字符串。replace 操作的作用是在指定的输入字符串中查找从 from 开始直到 to 结束的字符，并将匹配的字符替换为 to 指定的字符串。


```cpp
std::string trim(const std::string & s) {
    std::regex e("^\\s+|\\s+$");
    return std::regex_replace(s, e, "");
}

std::string replace(const std::string & s, const std::string & from, const std::string & to) {
    std::string result = s;
    size_t pos = 0;
    while ((pos = result.find(from, pos)) != std::string::npos) {
        result.replace(pos, from.length(), to);
        pos += to.length();
    }
    return result;
}

```

This is a C++ function that takes in a JSON string and returns a table of named values, where the names are keys and the values are numbers. The table is based on the keys-values format, where each key-value pair is represented by a named integer, where the key is a Unicode character, and the value is a Unicode string.

The function first reads the JSON string and loops through it. For each element in the JSON string, the function checks if it is a key-value pair or not. If it is a key-value pair, the function extracts the key and value from the JSON string and converts the key to a Unicode string. Then, it adds the key and value to the table.

If it is not a key-value pair, the function skips it.

The function also supports the "\" quotes, which indicate the start and end of the JSON string. When it finds a double quote, it sets the has\_key flag to be true and continues reading the JSON string. If it finds a closing double quote, it sets the has\_key flag to be false and skips the rest of the JSON string.

The function returns the table.


```cpp
void gpt_vocab::add_special_token(const std::string & token) {
    special_tokens.push_back(token);
}

std::map<std::string, int32_t> json_parse(const std::string & fname) {
    std::map<std::string, int32_t> result;

    // read file into string
    std::string json;
    {
        std::ifstream ifs(fname);
        if (!ifs) {
            fprintf(stderr, "Failed to open %s\n", fname.c_str());
            exit(1);
        }

        json = std::string((std::istreambuf_iterator<char>(ifs)),
                (std::istreambuf_iterator<char>()));
    }

    if (json[0] != '{') {
        return result;
    }

    // parse json
    {
        bool has_key  = false;
        bool in_token = false;

        std::string str_key = "";
        std::string str_val = "";

        int n = json.size();
        for (int i = 1; i < n; ++i) {
            if (!in_token) {
                if (json[i] == ' ') continue;
                if (json[i] == '"') {
                    in_token = true;
                    continue;
                }
            } else {
                if (json[i] == '\\' && i+1 < n) {
                    if (has_key == false) {
                        str_key += json[i];
                    } else {
                        str_val += json[i];
                    }
                    ++i;
                } else if (json[i] == '"') {
                    if (has_key == false) {
                        has_key = true;
                        ++i;
                        while (json[i] == ' ') ++i;
                        ++i; // :
                        while (json[i] == ' ') ++i;
                        if (json[i] != '\"') {
                            while (json[i] != ',' && json[i] != '}') {
                                str_val += json[i++];
                            }
                            has_key = false;
                        } else {
                            in_token = true;
                            continue;
                        }
                    } else {
                        has_key = false;
                    }

                    str_key = ::replace(str_key, "\\u0120", " " ); // \u0120 -> space
                    str_key = ::replace(str_key, "\\u010a", "\n"); // \u010a -> new line
                    str_key = ::replace(str_key, "\\\"",    "\""); // \\\"   -> "

                    try {
                        result[str_key] = std::stoi(str_val);
                    } catch (...) {
                        //fprintf(stderr, "%s: ignoring key '%s' with value '%s'\n", fname.c_str(), str_key.c_str(), str_val.c_str());

                    }
                    str_key = "";
                    str_val = "";
                    in_token = false;
                    continue;
                }
                if (has_key == false) {
                    str_key += json[i];
                } else {
                    str_val += json[i];
                }
            }
        }
    }

    return result;
}

```

这段代码主要实现了将输入的字符串（可能为 Unicode 字符）转换为 std::wstring 类型的功能。

具体来说，有两部分代码：

1. `convert_to_utf8()` 函数，接收一个 Unicode 字符串（`const std::wstring & input`）作为参数，然后通过 `std::codecvt_utf8<wchar_t>` 类型的 std::wstring_convert 对象，将输入的字符串转换为 bytes 类型的 std::wstring 对象，并返回。

2. `convert_to_wstring()` 函数，接收一个 Unicode 字符串（`const std::string & input`）作为参数，然后通过 `std::codecvt_utf8<wchar_t>` 类型的 std::wstring_convert 对象，将输入的字符串转换为 std::wstring 类型的对象，并返回。

3. `gpt_split_words()` 函数，接收一个 Unicode 字符串（`const std::string & str`）作为参数，然后通过在字符串中查找指定的模式（R"('s|'t|'re|'ve|'m|'ll|'d| ?[[:alpha:]]+| ?[[:digit:]]+| ?[^\s[:alpha:][:digit:]]+|\s+(?!\S)|\s+)")，并利用 std::regex 和 std::smatch 对象，将字符串中的每个匹配的子字符串（即一个或多个由 ' ' 和 non-'] 提取出来，将非 '' 的后面的所有字符串从 vector 中删除，并将匹配到的字符串添加到 vector 中。

这段代码的主要目的是为了在需要时将输入的 Unicode 字符串转换为 std::wstring 类型，以便在需要将输入字符串解析为 std::wstring 类型的函数中使用。


```cpp
std::string convert_to_utf8(const std::wstring & input) {
    std::wstring_convert<std::codecvt_utf8<wchar_t>> converter;
    return converter.to_bytes(input);
}


std::wstring convert_to_wstring(const std::string & input) {
    std::wstring_convert<std::codecvt_utf8<wchar_t>> converter;
    return converter.from_bytes(input);
}

void gpt_split_words(std::string str, std::vector<std::string>& words) {
    const std::string pattern = R"('s|'t|'re|'ve|'m|'ll|'d| ?[[:alpha:]]+| ?[[:digit:]]+| ?[^\s[:alpha:][:digit:]]+|\s+(?!\S)|\s+)";
    const std::regex re(pattern);
    std::smatch m;

    while (std::regex_search(str, m, re)) {
        for (auto x : m) {
            words.push_back(x);
        }
        str = m.suffix();
    }
}

```

This is a function that takes in a string and returns a vector of tokens (i.e. word segments) that were found in the original string using the Greek-geometric promotion (GPT) model. It first replaces certain special characters with their corresponding IDs and then splits the text into words. Then, it uses a regular expression to find all the occurrences of each word in the text and adds those occurrences as tokens to the output vector.

Finally, it returns the output vector.


```cpp
std::vector<gpt_vocab::id> gpt_tokenize(const gpt_vocab & vocab, const std::string & text) {
    std::vector<std::string> words;

    // first split the text into words
    {
        std::string str = text;

        // Generate the subpattern from the special_tokens vector if it's not empty
        if (!vocab.special_tokens.empty()) {
            const std::regex escape(R"([\[\\\^\$\.\|\?\*\+\(\)\{\}])");
            std::string special_tokens_subpattern;
            for (const auto & token : vocab.special_tokens) {
                if (!special_tokens_subpattern.empty()) {
                    special_tokens_subpattern += "|";
                }
                special_tokens_subpattern += std::regex_replace(token, escape, R"(\$&)");
            }

            std::regex re(special_tokens_subpattern);
            std::smatch m;
            // Split the text by special tokens.
            while (std::regex_search(str, m, re)) {
                // Split the substrings in-between special tokens into words.
                gpt_split_words(m.prefix(), words);
                // Add matched special tokens as words.
                for (auto x : m) {
                    words.push_back(x);
                }
                str = m.suffix();
            }
            // Remaining text without special tokens will be handled below.
        }

        gpt_split_words(str, words);
    }

    // find the longest token that forms each word in words:
    std::vector<gpt_vocab::id> tokens;
    for (const auto & word : words) {
        for (int i = 0; i < (int) word.size(); ){
            for (int j = word.size() - 1; j >= i; j--){
                auto cand = word.substr(i, j-i+1);
                auto it = vocab.token_to_id.find(cand);
                if (it != vocab.token_to_id.end()){ // word.substr(i, j-i+1) in vocab
                    tokens.push_back(it->second);
                    i = j + 1;
                    break;
                }
                else if (j == i){ // word.substr(i, 1) has no matching
                    fprintf(stderr, "%s: unknown token '%s'\n", __func__, word.substr(i, 1).data());
                    i++;
                }
            }
        }
    }

    return tokens;
}

```

这段代码的主要作用是读取一个文件中的测试用例并将其存储为另一个 map 中。这个 map 包含了每个测试用例对应的词汇表 id 集合。

首先，它定义了一个名为 parse_tokens_from_string 的函数，它接收一个字符串输入，以及一个字符作为分隔符。这个函数将接收输入字符串中的每一行，并将其转换为一个 token 字符串。它将使用 Boost.Property Tree ( Boost 库) 的 std::stoi 函数将 token 转换为整数，并将结果添加到输出 vector 中。

接下来，它定义了一个名为 extract_tests_from_file 的函数，它接收一个文件路径。如果文件路径为空，它将在地图中创建一个空集。否则，它将读取文件中的每一行，并将每一行转换为一个测试用例。对于每一个测试用例，它使用 parse_tokens_from_string 函数将输入字符串转换为 id 集合，并将结果添加到 map 中。

由于测试用例是在文件中以空格分隔的，因此 extract_tests_from_file 函数使用的是 std::ifstream 读取文件。而 parse_tokens_from_string 函数则没有使用任何文件输入或输出，它仅仅定义了一个内部函数来处理 token 字符串的解析。


```cpp
std::vector<gpt_vocab::id> parse_tokens_from_string(const std::string& input, char delimiter) {
    std::vector<gpt_vocab::id> output;
    std::stringstream ss(input);
    std::string token;

    while (std::getline(ss, token, delimiter)) {
        output.push_back(std::stoi(token));
    }

    return output;
}

std::map<std::string, std::vector<gpt_vocab::id>> extract_tests_from_file(const std::string & fpath_test){
    if (fpath_test.empty()){
        fprintf(stderr, "%s : No test file found.\n", __func__);
        return std::map<std::string, std::vector<gpt_vocab::id>>();
    }

    std::map<std::string, std::vector<gpt_vocab::id>> tests;

    auto fin = std::ifstream(fpath_test, std::ios_base::in);
    const char * delimeter = " => ";
    const char del_tok = ',';
    std::string line;
    while (std::getline(fin, line)) {
        size_t delimiterPos = line.find(delimeter);
        if (delimiterPos != std::string::npos) {
            std::string text = line.substr(0, delimiterPos);
            std::string s_tokens = line.substr(delimiterPos + std::strlen(delimeter));
            tests[text] = parse_tokens_from_string(s_tokens, del_tok);
        }
    }
    return tests;
}

```

这段代码是一个名为`test_gpt_tokenizer`的函数，它对一个给定的测试文件夹中的GPT模型进行测试。具体来说，它实现了以下功能：

1. 从给定的测试文件夹中提取测试用例。
2. 对提取出的测试用例进行处理，如果处理失败，则打印错误信息。
3. 对处理成功的测试用例进行处理。
4. 对每个测试用例，使用给定的GPT模型中的词汇表（vocab）进行 tokenize 操作，生成一个标准输出（std::vector<gpt_vocab::id>）。
5. 如果生成的标准输出与给定的测试用例不符，则记录失败测试的数量。
6. 打印失败测试的数量和详细信息，以便进行调试。

该函数的主要作用是检查给定的测试文件夹中的GPT模型是否能够正确地处理测试用例。它通过提取测试文件夹中的测试用例，并尝试使用GPT模型对测试用例进行处理，如果处理失败，则记录失败测试的数量，并输出详细信息。如果处理成功，则不对测试用例进行进一步处理，这样我们就可以在测试失败时进行调试。


```cpp
void test_gpt_tokenizer(gpt_vocab & vocab, const std::string & fpath_test){
    std::map<std::string, std::vector<gpt_vocab::id>> tests = extract_tests_from_file(fpath_test);

    size_t n_fails = 0;

    for (const auto & test : tests) {
        std::vector<gpt_vocab::id> tokens = gpt_tokenize(vocab, test.first);

        if (tokens != test.second){
            n_fails++;

            // print out failure cases
            fprintf(stderr, "%s : failed test: '%s'\n", __func__, test.first.c_str());
            fprintf(stderr, "%s : tokens in hf:   ", __func__);
            for (const auto & t : test.second) {
                fprintf(stderr, "%s(%d), ", vocab.id_to_token[t].c_str(), t);
            }
            fprintf(stderr, "\n");
            fprintf(stderr, "%s : tokens in ggml: ", __func__);
            for (const auto & t : tokens) {
                fprintf(stderr, "%s(%d), ", vocab.id_to_token[t].c_str(), t);
            }
            fprintf(stderr, "\n");
        }
    }

    fprintf(stderr, "%s : %zu tests failed out of %zu tests.\n", __func__, n_fails, tests.size());
}

```

这段代码是一个名为`gpt_vocab_init`的函数，它的作用是加载一个名为`vocab`的词汇表，并将其存储在内存中。

具体来说，这段代码做了以下几件事情：

1. 读取文件并解码 JSON 数据，将数据存储在`vocab.token_to_id`数组中；
2. 遍历`vocab.token_to_id`数组，将每个键值对（键和值）存储在`vocab.id_to_token`数组中；
3. 输出词汇表的大小；
4. 输出词汇表中的所有词汇。

输出结果将类似于这样：
```cpp
loading vocab from ' vocabulary.txt'
vocab size = 512
'clasiveness' -> 1
'extent' -> 2
'intent' -> 3
...（省略其他词汇）
```
这段代码的目的是让`gpt`模型能够使用一个包含预定义词汇的词汇表，从而在运行时更高效地预测语言模型。


```cpp
bool gpt_vocab_init(const std::string & fname, gpt_vocab & vocab) {
    printf("%s: loading vocab from '%s'\n", __func__, fname.c_str());

    vocab.token_to_id = ::json_parse(fname);

    for (const auto & kv : vocab.token_to_id) {
        vocab.id_to_token[kv.second] = kv.first;
    }

    printf("%s: vocab size = %d\n", __func__, (int) vocab.token_to_id.size());

    // print the vocabulary
    //for (auto kv : vocab.token_to_id) {
    //    printf("'%s' -> %d\n", kv.first.data(), kv.second);
    //}

    return true;
}

```

This code appears to be a function that generates next word suggestions based on a given set of top _k suggestions and a vocabulary of _vocab words. It uses a combination of exponential distributions and floating point probabilities to generate these suggestions.

The function takes in an array of top _k suggestions, represented by the `logits_id` array, and a vocabulary `vocab` mapping. It returns the corresponding word, or a word that is in the vocabulary if the top _k suggestions are not present.

The function first computes the probabilities of each of the top _k suggestions, assuming that the logits for each suggestion are normalized to be between 0 and 1. It then uses these probabilities to select the word that is most likely to be the next word, based on the硕士学位 of the probabilities. If the top _k suggestions have a certain probability, it will choose a word regardless of whether it is in the vocabulary or not.

Note that the word选择 algorithm assumes that the function has already computed the logits for each of the top _k suggestions. It also assumes that the input `vocab` is in a static，不定长， dense data structure.


```cpp
gpt_vocab::id gpt_sample_top_k_top_p(
        const gpt_vocab & vocab,
        const float * logits,
        int    top_k,
        double top_p,
        double temp,
        std::mt19937 & rng) {
    int n_logits = vocab.id_to_token.size();

    std::vector<std::pair<double, gpt_vocab::id>> logits_id;
    logits_id.reserve(n_logits);

    {
        const double scale = 1.0/temp;
        for (int i = 0; i < n_logits; ++i) {
            logits_id.push_back(std::make_pair(logits[i]*scale, i));
        }
    }

    // find the top K tokens
    std::partial_sort(
            logits_id.begin(),
            logits_id.begin() + top_k, logits_id.end(),
            [](const std::pair<double, gpt_vocab::id> & a, const std::pair<double, gpt_vocab::id> & b) {
        return a.first > b.first;
    });

    logits_id.resize(top_k);

    double maxl = -INFINITY;
    for (const auto & kv : logits_id) {
        maxl = std::max(maxl, kv.first);
    }

    // compute probs for the top K tokens
    std::vector<double> probs;
    probs.reserve(logits_id.size());

    double sum = 0.0;
    for (const auto & kv : logits_id) {
        double p = exp(kv.first - maxl);
        probs.push_back(p);
        sum += p;
    }

    // normalize the probs
    for (auto & p : probs) {
        p /= sum;
    }

    if (top_p < 1.0f) {
        double cumsum = 0.0f;
        for (int i = 0; i < top_k; i++) {
            cumsum += probs[i];
            if (cumsum >= top_p) {
                top_k = i + 1;
                probs.resize(top_k);
                logits_id.resize(top_k);
                break;
            }
        }

        cumsum = 1.0/cumsum;
        for (int i = 0; i < (int) probs.size(); i++) {
            probs[i] *= cumsum;
        }
    }

    //printf("\n");
    //for (int i = 0; i < (int) probs.size(); i++) {
    //    printf("%d: '%s' %f\n", i, vocab.id_to_token.at(logits_id[i].second).c_str(), probs[i]);
    //}
    //exit(0);

    std::discrete_distribution<> dist(probs.begin(), probs.end());
    int idx = dist(rng);

    return logits_id[idx].second;
}

```

This code appears to be a function that takes in a list of logits (with a scaling factor applied), and a list of probs for those logits. The function outputs a vector of probabilities for the top K tokens (with the same scaling factor applied), sorted in decreasing order of probability.

The input logits are first sorted in non-increasing order, then a max pooling operation is performed to combine all the logits into a single tensor. This tensor is then passed through a vector of probabilities, which are then normalized to sum to 1.0 and divide by the number of elements in the tensor.

The function then performs a partitioning operation, where the top K probs are selected and assigned to the output tensor. This is done by first calculating the cumulative sum of the probs, and then using this sum to determine the first K probabilities to output. The remaining probabilities are left to gather in the output tensor.

The output is a vector of probabilities for the top K tokens, sorted in decreasing order of probability.


```cpp
gpt_vocab::id gpt_sample_top_k_top_p_repeat(
        const gpt_vocab & vocab,
        const float * logits,
        const int32_t * last_n_tokens_data,
        size_t last_n_tokens_data_size,
        int    top_k,
        double top_p,
        double temp,
        int repeat_last_n,
        float repeat_penalty,
        std::mt19937 & rng) {

    int n_logits = vocab.id_to_token.size();

    const auto * plogits = logits;

    const auto last_n_tokens = std::vector<int32_t>(last_n_tokens_data, last_n_tokens_data + last_n_tokens_data_size);

    if (temp <= 0) {
        // select the token with the highest logit directly
        float max_logit = plogits[0];
        gpt_vocab::id max_id = 0;

        for (int i = 1; i < n_logits; ++i) {
            if (plogits[i] > max_logit) {
                max_logit = plogits[i];
                max_id = i;
            }
        }
        return max_id;
    }


    std::vector<std::pair<double, gpt_vocab::id>> logits_id;
    logits_id.reserve(n_logits);

    {
        const float scale = 1.0f/temp;
        for (int i = 0; i < n_logits; ++i) {
            // repetition penalty from ctrl paper (https://arxiv.org/abs/1909.05858)
            // credit https://github.com/facebookresearch/llama/compare/main...shawwn:llama:main
            if (repeat_last_n > 0 && std::find(last_n_tokens.end()-repeat_last_n, last_n_tokens.end(), i) != last_n_tokens.end()) {
                // if score < 0 then repetition penalty has to multiplied to reduce the previous token probability
                if (plogits[i] < 0.0f) {
                    logits_id.push_back(std::make_pair(plogits[i]*scale*repeat_penalty, i));
                } else {
                    logits_id.push_back(std::make_pair(plogits[i]*scale/repeat_penalty, i));
                }
            } else {
                logits_id.push_back(std::make_pair(plogits[i]*scale, i));
            }
        }
    }

    // find the top K tokens
    std::partial_sort(
            logits_id.begin(),
            logits_id.begin() + top_k, logits_id.end(),
            [](const std::pair<double, gpt_vocab::id> & a, const std::pair<double, gpt_vocab::id> & b) {
        return a.first > b.first;
    });

    logits_id.resize(top_k);

    double maxl = -INFINITY;
    for (const auto & kv : logits_id) {
        maxl = std::max(maxl, kv.first);
    }

    // compute probs for the top K tokens
    std::vector<double> probs;
    probs.reserve(logits_id.size());

    double sum = 0.0;
    for (const auto & kv : logits_id) {
        double p = exp(kv.first - maxl);
        probs.push_back(p);
        sum += p;
    }

    // normalize the probs
    for (auto & p : probs) {
        p /= sum;
    }

    if (top_p < 1.0f) {
        double cumsum = 0.0f;
        for (int i = 0; i < top_k; i++) {
            cumsum += probs[i];
            if (cumsum >= top_p) {
                top_k = i + 1;
                probs.resize(top_k);
                logits_id.resize(top_k);
                break;
            }
        }

        cumsum = 1.0/cumsum;
        for (int i = 0; i < (int) probs.size(); i++) {
            probs[i] *= cumsum;
        }
    }

```

根据问题，我们需要判断函数'%s'是否正确，它接受一个WAV文件的文件名作为参数，并返回一个布尔值。函数的实现如下：
```cppC
bool wav_read_pcm(const std::string& fileName, std::vector<int16_t>& pcm, std::vector<int16_t>& pcmf32, std::vector<int16_t>& pcmf32s);
```
我们需要根据WAV文件的文件名来读取WAV文件的内容，并将其转换为PCM数据，最后根据是否为立体声来转换为对应的浮点数数据。下面是一个具体的实现方式：
```cppC
bool wav_read_pcm(const std::string& fileName, std::vector<int16_t>& pcm, std::vector<int16_t>& pcmf32, std::vector<int16_t>& pcmf32s) {
   // 读取WAV文件
   if (!file_read_as_audio_file(fileName.c_str(), pcm, pcmf32, pcmf32s)) {
       return false;
   }

   // 判断WAV文件是否为16位
   const uint64_t n = pcmf32.empty() ? wav.totalPCMFrameCount : wav_data.size()/(wav.channels*wav.bitsPerSample/8);
   if (n != 16) {
       return false;
   }

   // 准备PCM数据
   std::vector<int16_t> pcm16;
   pcm16.resize(n*wav.channels);
   drwav_read_pcm_frames_s16(&wav, n, pcm16.data());
   drwav_uninit(&wav);

   // 转换为单声道，float
   pcmf32.resize(n);
   if (wav.channels == 1) {
       for (uint64_t i = 0; i < n; i++) {
           pcmf32[i] = float(pcm16[i])/32768.0f;
       }
   } else {
       for (uint64_t i = 0; i < n; i++) {
           pcmf32[i] = float(pcm16[2*i] + pcm16[2*i + 1])/65536.0f;
       }
   }

   // 判断是否为立体声
   if (stereo) {
       // 转换为立体声，float
       pcmf32s.resize(2);

       pcmf32s[0].resize(n);
       pcmf32s[1].resize(n);
       for (uint64_t i = 0; i < n; i++) {
           pcmf32s[0][i] = float(pcm16[2*i])/32768.0f;
           pcmf32s[1][i] = float(pcm16[2*i + 1])/32768.0f;
       }
   }

   return true;
}
```
下面是一个简单的测试，用来检查函数的正确性：
```cppC
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <fcntl.h>
#include <sys/stat.h>
#include <int16.h>
#include <int16_math.h>

static void usage_error(const std::string& fileName) {
   std::cerr << "Usage: %s fileName\n";
   return;
}

static void read_error(const std::string& fileName) {
   std::cerr << "Error: " << fileName << "\n";
   return;
}

static int16_t convert_16bit_float(const float& float_value) {
   return int16_t(float_value * 32767.0f);
}

static int16_t convert_32bit_float(const float& float_value) {
   return int16_t(float_value * 32767.0f / 16.0f);
}

static void print_help() {
   std::cout << "Usage: \"" << std::endl;
   std::cout << "wav_read_pcm" << std::endl;
   std::cout << " -i, --input-file <file name>" << std::endl;
   std::cout << "                      <std-定点数(float)>" << std::endl;
   std::cout << "                      <std-定点数(float)/32位" << std::endl;
   std::cout << "                      <单声道，float>" << std::endl;
   std::cout << "                      <立体声，float>" << std::endl;
   std::cout << "                      <立体声，float>" << std::endl;
   std::cout << "                      <单声道，float>" << std::endl;
   std::cout << "                      -f" << std::endl;
   std::cout << "                      -b <exponent>" << std::endl;
   std::cout << std::endl << std::endl << std::endl;
}

int main(int argc, char** argv) {
   std::string input_file = argv[0];
   if (argc < 1] || argv[1] != "fileName") {
       usage_error(std::string(argv[0]));
       return -1;
   }

   std::vector<int16_t> pcm;
   std::vector<int16_t> pcmf32;
   std::vector<int16_t> pcmf32s;

   if (!wav_read_pcm(input_file.c_str(), pcm, pcmf32, pcmf32s)) {
       return -1;
   }

   // usage example
   int16_t float_value = 0.5f;
   int16_t pcm


```
//    printf("\n");
//    for (int i = 0; i < (int) probs.size(); i++) {
//    for (int i = 0; i < 10; i++) {
//        printf("%d: '%s' %f\n", i, vocab.id_to_token.at(logits_id[i].second).c_str(), probs[i]);
//    }

    std::discrete_distribution<> dist(probs.begin(), probs.end());
    int idx = dist(rng);

    return logits_id[idx].second;

}

bool read_wav(const std::string & fname, std::vector<float>& pcmf32, std::vector<std::vector<float>>& pcmf32s, bool stereo) {
    drwav wav;
    std::vector<uint8_t> wav_data; // used for pipe input from stdin

    if (fname == "-") {
        {
            uint8_t buf[1024];
            while (true)
            {
                const size_t n = fread(buf, 1, sizeof(buf), stdin);
                if (n == 0) {
                    break;
                }
                wav_data.insert(wav_data.end(), buf, buf + n);
            }
        }

        if (drwav_init_memory(&wav, wav_data.data(), wav_data.size(), nullptr) == false) {
            fprintf(stderr, "error: failed to open WAV file from stdin\n");
            return false;
        }

        fprintf(stderr, "%s: read %zu bytes from stdin\n", __func__, wav_data.size());
    }
    else if (drwav_init_file(&wav, fname.c_str(), nullptr) == false) {
        fprintf(stderr, "error: failed to open '%s' as WAV file\n", fname.c_str());
        return false;
    }

    if (wav.channels != 1 && wav.channels != 2) {
        fprintf(stderr, "%s: WAV file '%s' must be mono or stereo\n", __func__, fname.c_str());
        return false;
    }

    if (stereo && wav.channels != 2) {
        fprintf(stderr, "%s: WAV file '%s' must be stereo for diarization\n", __func__, fname.c_str());
        return false;
    }

    if (wav.sampleRate != COMMON_SAMPLE_RATE) {
        fprintf(stderr, "%s: WAV file '%s' must be %i kHz\n", __func__, fname.c_str(), COMMON_SAMPLE_RATE/1000);
        return false;
    }

    if (wav.bitsPerSample != 16) {
        fprintf(stderr, "%s: WAV file '%s' must be 16-bit\n", __func__, fname.c_str());
        return false;
    }

    const uint64_t n = wav_data.empty() ? wav.totalPCMFrameCount : wav_data.size()/(wav.channels*wav.bitsPerSample/8);

    std::vector<int16_t> pcm16;
    pcm16.resize(n*wav.channels);
    drwav_read_pcm_frames_s16(&wav, n, pcm16.data());
    drwav_uninit(&wav);

    // convert to mono, float
    pcmf32.resize(n);
    if (wav.channels == 1) {
        for (uint64_t i = 0; i < n; i++) {
            pcmf32[i] = float(pcm16[i])/32768.0f;
        }
    } else {
        for (uint64_t i = 0; i < n; i++) {
            pcmf32[i] = float(pcm16[2*i] + pcm16[2*i + 1])/65536.0f;
        }
    }

    if (stereo) {
        // convert to stereo, float
        pcmf32s.resize(2);

        pcmf32s[0].resize(n);
        pcmf32s[1].resize(n);
        for (uint64_t i = 0; i < n; i++) {
            pcmf32s[0][i] = float(pcm16[2*i])/32768.0f;
            pcmf32s[1][i] = float(pcm16[2*i + 1])/32768.0f;
        }
    }

    return true;
}

```cpp

This is a C++ implementation of a simple variable adaptative (VAD) filter for speech processing. The VAD filter takes a PCMF (Pulse-Connected Multifunction) signal, a sample rate, a minimum allowed VAD noise threshold, a minimum allowed frequency adjustment threshold, and an optional display of energy information.

The VAD filter performs the following steps:

1. Compute the first difference of the input signal and store it in the `data` vector.
2. Compute a cumulative sum of the squared differences.
3. Compute the inverse-time-compressed data as the difference between the cumulative sum and the sample rate divided by 1000.
4. Apply a high-pass filter to the data to remove low-frequency noise.
5. Compute the energy statistics for the input signal and store them in the `energy_all` and `energy_last` variables.
6. Compare the cumulative sum with the sample rate-weighted sum to determine if the signal is speech.
7. Compare the energy of the input signal with the VAD noise threshold and update the filter accordingly.

The filter returns true if the signal is speech and the energy difference is below the threshold.

Note: This implementation assumes that the input signal is a one-dimensional PCMF vector and that the high-pass filter is a convolution-like operation. Depending on the actual implementation, the filter may be more complex or may be able to handle more channels.


```
void high_pass_filter(std::vector<float> & data, float cutoff, float sample_rate) {
    const float rc = 1.0f / (2.0f * M_PI * cutoff);
    const float dt = 1.0f / sample_rate;
    const float alpha = dt / (rc + dt);

    float y = data[0];

    for (size_t i = 1; i < data.size(); i++) {
        y = alpha * (y + data[i] - data[i - 1]);
        data[i] = y;
    }
}

bool vad_simple(std::vector<float> & pcmf32, int sample_rate, int last_ms, float vad_thold, float freq_thold, bool verbose) {
    const int n_samples      = pcmf32.size();
    const int n_samples_last = (sample_rate * last_ms) / 1000;

    if (n_samples_last >= n_samples) {
        // not enough samples - assume no speech
        return false;
    }

    if (freq_thold > 0.0f) {
        high_pass_filter(pcmf32, freq_thold, sample_rate);
    }

    float energy_all  = 0.0f;
    float energy_last = 0.0f;

    for (int i = 0; i < n_samples; i++) {
        energy_all += fabsf(pcmf32[i]);

        if (i >= n_samples - n_samples_last) {
            energy_last += fabsf(pcmf32[i]);
        }
    }

    energy_all  /= n_samples;
    energy_last /= n_samples_last;

    if (verbose) {
        fprintf(stderr, "%s: energy_all: %f, energy_last: %f, vad_thold: %f, freq_thold: %f\n", __func__, energy_all, energy_last, vad_thold, freq_thold);
    }

    if (energy_last > vad_thold*energy_all) {
        return false;
    }

    return true;
}

```cpp

该代码定义了一个名为 "similarity" 的函数，接受两个字符串参数 "s0" 和 "s1"。函数的目的是计算这两个字符串之间的相似度，并返回一个浮点数。

函数首先计算两个字符串的长度之和以及它们各自的单词数量。然后，定义了一个 "col" 向量，其长度为 "s1"，并且它的每个元素都被分配了一个独特的预处理值 "prevCol"。接下来，函数遍历 "col" 向量，并且对于每个元素，首先定义一个名为 "prevCol" 的变量，然后从 "col" 和 "prevCol" 中选择两个值，并通过 min() 函数来找到它们之间的最小值。最后，将选择的所有值替换回 "prevCol"，并且计算距离 "dist" 的值。

函数的返回值是 1.0f - (距离 "dist" 的值 / 最大长度之和)。


```
float similarity(const std::string & s0, const std::string & s1) {
    const size_t len0 = s0.size() + 1;
    const size_t len1 = s1.size() + 1;

    std::vector<int> col(len1, 0);
    std::vector<int> prevCol(len1, 0);

    for (size_t i = 0; i < len1; i++) {
        prevCol[i] = i;
    }

    for (size_t i = 0; i < len0; i++) {
        col[0] = i;
        for (size_t j = 1; j < len1; j++) {
            col[j] = std::min(std::min(1 + col[j - 1], 1 + prevCol[j]), prevCol[j - 1] + (i > 0 && s0[i - 1] == s1[j - 1] ? 0 : 1));
        }
        col.swap(prevCol);
    }

    const float dist = prevCol[len1 - 1];

    return 1.0f - (dist / std::max(s0.size(), s1.size()));
}

```cpp

这段代码是一个名为`sam_params_parse`的函数，它的作用是 parsing command line arguments，并返回一个`sam_params`结构体。

该函数接受一个整数参数`argc`，一个指向字符串的指针`argv`，以及一个`sam_params`结构体变量`params`。

该函数使用一个for循环来逐个检查给定的命令行参数。对于每个参数，函数首先检查参数是否为`-s`或`--seed`。如果是，函数将`args`转换为一个整数，并将其赋值给`params.seed`。

如果参数不是`-s`或`--seed`，则函数将逐个检查其他参数。如果找到了一个参数是`-t`或`--threads`，则函数将`args`转换为一个整数，并将其赋值给`params.n_threads`。

如果找到了一个参数是`-m`或`--model`，则函数将`args`转换为一个字符串，并将其赋值给`params.model`。

如果找到了一个参数是`-i`或`--inp`，则函数将`args`转换为一个字符串，并将其赋值给`params.fname_inp`。

如果找到了一个参数是`-o`或`--out`，则函数将`args`转换为一个字符串，并将其赋值给`params.fname_out`。

如果找到了一个参数是`-h`或`--help`，则函数将`sam_print_usage`函数作为参数传递给`stderr`输出，并使用`-e`选项来指示错误。如果函数在`for`循环中未找到任何有效的命令行参数，则它将从`stderr`中读取错误消息，并使用`sam_print_usage`函数打印错误消息，并跳转到`--help`选项，以帮助用户知道如何使用该函数。


```
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

```cpp

这段代码是一个函数，名为 `sam_print_usage`，其作用是打印出使用该函数的帮助信息并返回。

具体来说，该函数接受四个参数：

- `argc`：表示命令行参数的数量，但并不包含第一个参数。
- `argv`：包含两个指针，指向第一个和第二个参数。
- `params`：是一个 `sam_params` 类型的数据结构体，其中包含了程序需要的所有参数。

函数体中首先输出帮助信息，然后是所有可选项及其对应的缩写。缩写对应的字符串都是用`fprintf`函数打印到标准错误（通常是屏幕）上。

例如，函数输出的内容如下：

```
usage: %s [options]

options:
 -h, --help            show this help message and exit
 -s SEED, --seed SEED  RNG seed (default: -1)
 -t N, --threads N     number of threads to use during computation (default: %d)
 -m FNAME, --model FNAME
                       model path (default: %s)
 -i FNAME, --inp FNAME
                       input file (default: %s)
 -o FNAME, --out FNAME
                       output file (default: %s)
```cpp

然后是可选参数的详细说明，其中包括了 `-h` 和 `--help` 选项。


```
void sam_print_usage(int /*argc*/, char ** argv, const sam_params & params) {
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
    fprintf(stderr, "                        output file (default: %s)\n", params.fname_out.c_str());
    fprintf(stderr, "\n");
}

```