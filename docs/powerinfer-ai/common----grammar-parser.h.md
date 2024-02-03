# `PowerInfer\common\grammar-parser.h`

```cpp
// 实现了对扩展的Backus-Naur形式（BNF）的解析器，生成llama.h指定的二进制无上下文语法格式。支持字符范围、分组和重复操作符。例如，算术的语法可能如下所示：
//
// root  ::= expr
// expr  ::= term ([-+*/] term)*
// term  ::= num | "(" space expr ")" space
// num   ::= [0-9]+ space
// space ::= [ \t\n]*
#pragma once
#include "llama.h"
#include <vector>
#include <map>
#include <cstdint>
#include <string>

namespace grammar_parser {
    // 定义解析状态结构体
    struct parse_state {
        // 符号名称到ID的映射
        std::map<std::string, uint32_t> symbol_ids;
        // 规则集合
        std::vector<std::vector<llama_grammar_element>> rules;

        // 返回C风格的规则集合
        std::vector<const llama_grammar_element *> c_rules();
    };

    // 解析给定源代码，返回解析状态
    parse_state parse(const char * src);
    // 将解析状态打印到文件
    void print_grammar(FILE * file, const parse_state & state);
}
```