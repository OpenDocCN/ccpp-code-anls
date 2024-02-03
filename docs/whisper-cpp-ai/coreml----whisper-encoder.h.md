# `whisper.cpp\coreml\whisper-encoder.h`

```cpp
// 封装了 Core ML Whisper Encoder 模型的结构体
//
// 代码来源于 Github 用户 @wangchou 的工作
// 参考链接: https://github.com/wangchou/callCoreMLFromCpp

#include <stdint.h>

#if __cplusplus
extern "C" {
#endif

// 定义了一个结构体 whisper_coreml_context，用于存储 Core ML 模型的上下文信息
struct whisper_coreml_context;

// 初始化 Core ML 模型的上下文信息，传入模型文件路径，返回一个指向 whisper_coreml_context 结构体的指针
struct whisper_coreml_context * whisper_coreml_init(const char * path_model);

// 释放 Core ML 模型的上下文信息，传入指向 whisper_coreml_context 结构体的指针
void whisper_coreml_free(struct whisper_coreml_context * ctx);

// 使用 Core ML 模型对输入数据进行编码，传入模型上下文信息、n_ctx、n_mel、mel 数组和输出数组 out
void whisper_coreml_encode(
        const whisper_coreml_context * ctx,
                             int64_t   n_ctx,
                             int64_t   n_mel,
                               float * mel,
                               float * out);

#if __cplusplus
}
#endif
```