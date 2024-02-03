# `whisper.cpp\openvino\whisper-openvino-encoder.h`

```cpp
// OpenVINO Whisper Encoder 模型的包装器

#if __cplusplus
extern "C" {
#endif

// 定义 OpenVINO 上下文结构体
struct whisper_openvino_context;

// 初始化 OpenVINO 编码器，传入模型 XML 路径、设备类型（"CPU"、"GPU" 等）和缓存目录路径。失败时返回 null。
struct whisper_openvino_context * whisper_openvino_init(const char * path_model,
                                                        const char * device,
                                                        const char * cache_dir);

// 释放之前从 whisper_openvino_init() 返回的上下文 ctx
void whisper_openvino_free(struct whisper_openvino_context * ctx);

// 定义 GGML 张量结构体
struct ggml_tensor;

// 使用 OpenVINO 进行编码
// 成功时返回 1
// 失败时返回 0
int whisper_openvino_encode(
    whisper_openvino_context* ctx,
    ggml_tensor* mel,
    ggml_tensor* out);

#if __cplusplus
}
#endif
```