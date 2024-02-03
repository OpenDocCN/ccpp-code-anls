# `whisper.cpp\examples\common-ggml.h`

```cpp
// 防止头文件重复包含
#pragma once

// 包含 ggml.h 头文件
#include "ggml.h"

// 包含文件输入输出流、向量和字符串头文件
#include <fstream>
#include <vector>
#include <string>

// 定义枚举类型 ggml_ftype，并声明 ggml_parse_ftype 函数
enum ggml_ftype ggml_parse_ftype(const char * str);

// 声明 ggml_print_ftypes 函数，打印文件类型到指定文件流，默认为标准错误流
void ggml_print_ftypes(FILE * fp = stderr);

// 声明 ggml_common_quantize_0 函数，对输入流进行通用量化处理
bool ggml_common_quantize_0(
        std::ifstream & finp, // 输入文件流
        std::ofstream & fout, // 输出文件流
        const ggml_ftype ftype, // 文件类型
        const std::vector<std::string> & to_quant, // 需要量化的内容
        const std::vector<std::string> & to_skip); // 需要跳过的内容
```