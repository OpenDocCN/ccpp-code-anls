# `whisper.cpp\bindings\ruby\ext\ruby_whisper.h`

```cpp
#ifndef __RUBY_WHISPER_H
#define __RUBY_WHISPER_H

// 包含 whisper.h 头文件
#include "whisper.h"

// 定义 ruby_whisper 结构体，包含 whisper_context 指针
typedef struct {
  struct whisper_context *context;
} ruby_whisper;

// 定义 ruby_whisper_params 结构体，包含 whisper_full_params 结构体和 diarize 布尔值
typedef struct {
  struct whisper_full_params params;
  bool diarize;
} ruby_whisper_params;

#endif
```