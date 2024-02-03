# `whisper.cpp\examples\talk\gpt-2.h`

```cpp
// 防止头文件被重复包含
#pragma once

// TODO: 更改为 C 风格的 API 并移动到 ./examples 目录以便于重用

// 包含通用头文件
#include "common.h"

// 包含必要的标准库头文件
#include <vector>
#include <map>
#include <string>

// 定义 GPT-2 上下文结构体
struct gpt2_context;

// 初始化 GPT-2 上下文，传入模型路径，返回上下文指针
struct gpt2_context * gpt2_init(const char * path_model);

// 释放 GPT-2 上下文，传入上下文指针
void gpt2_free(struct gpt2_context * ctx);

// 获取 GPT-2 上下文的提示文本，传入上下文指针，返回提示文本
const char * gpt2_get_prompt(struct gpt2_context * ctx);

// 设置 GPT-2 上下文的提示文本，传入上下文指针和提示文本
void gpt2_set_prompt(struct gpt2_context * ctx, const char * prompt);

// 对文本进行分词处理，传入上下文指针和文本，返回分词后的结果
std::vector<gpt_vocab::id> gpt2_tokenize(const gpt2_context * ctx, const char * text);

// 生成文本，传入上下文指针、文本和最大 token 数，返回生成的文本
std::string gpt2_gen_text(gpt2_context * ctx, const char * text, int max_tokens);
```