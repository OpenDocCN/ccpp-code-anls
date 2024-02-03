# `ggml\examples\common-ggml.h`

```cpp
// 防止头文件重复包含
#pragma once

// 包含自定义的头文件 ggml.h
#include "ggml.h"

// 包含文件输入输出流、向量和字符串的标准库头文件
#include <fstream>
#include <vector>
#include <string>

// 定义枚举类型 ggml_ftype，并声明一个函数用于将字符串转换为该枚举类型
enum ggml_ftype ggml_parse_ftype(const char * str);

// 声明一个函数用于将 ggml_ftype 枚举类型打印到指定文件流，默认为标准错误流
void ggml_print_ftypes(FILE * fp = stderr);

// 声明一个函数用于对输入文件进行量化处理，并将结果输出到输出文件
bool ggml_common_quantize_0(
        std::ifstream & finp,  // 输入文件流
        std::ofstream & fout,  // 输出文件流
        const ggml_ftype ftype,  // 文件类型
        const std::vector<std::string> & to_quant,  // 需要量化的内容
        const std::vector<std::string> & to_skip);  // 需要跳过的内容
```