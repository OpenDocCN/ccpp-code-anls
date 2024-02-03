# `whisper.cpp\bindings\java\src\main\java\io\github\ggerganov\whispercpp\model\WhisperModel.java`

```cpp
public class WhisperModel {
//    EModel type = EModel.MODEL_UNKNOWN;
//    定义模型类型，默认为未知类型

//    WhisperHParams hparams;
//    WhisperFilters filters;
//    定义模型超参数和滤波器

//    // encoder.positional_embedding
//    GgmlTensor e_pe;
//    定义编码器的位置嵌入张量

//    // encoder.conv1
//    GgmlTensor e_conv_1_w;
//    GgmlTensor e_conv_1_b;
//    定义编码器的第一个卷积层权重和偏置张量

//    // encoder.conv2
//    GgmlTensor e_conv_2_w;
//    GgmlTensor e_conv_2_b;
//    定义编码器的第二个卷积层权重和偏置张量

//    // encoder.ln_post
//    GgmlTensor e_ln_w;
//    GgmlTensor e_ln_b;
//    定义编码器的 Layer Normalization 权重和偏置张量

//    // decoder.positional_embedding
//    GgmlTensor d_pe;
//    定义解码器的位置嵌入张量

//    // decoder.token_embedding
//    GgmlTensor d_te;
//    定义解码器的标记嵌入张量

//    // decoder.ln
//    GgmlTensor d_ln_w;
//    GgmlTensor d_ln_b;
//    定义解码器的 Layer Normalization 权重和偏置张量

//    std::vector<whisper_layer_encoder> layers_encoder;
//    std::vector<whisper_layer_decoder> layers_decoder;
//    定义编码器和解码器的层

//    // context
//    struct ggml_context * ctx;
//    定义上下文结构体指针

//    // the model memory buffer is read-only and can be shared between processors
//    std::vector<uint8_t> * buf;
//    定义模型内存缓冲区，只读且可在处理器之间共享

//    // tensors
//    int n_loaded;
//    Map<String, GgmlTensor> tensors;
//    定义加载的张量数量和张量映射
}
```