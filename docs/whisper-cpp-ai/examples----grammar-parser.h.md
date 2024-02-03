# `whisper.cpp\examples\grammar-parser.h`

```cpp
// 实现了对扩展巴科斯-瑙尔范式（BNF）的解析器，生成了whisper.h指定的二进制无上下文语法格式。支持字符范围、分组和重复操作符。例如，一个用于算术的语法可能如下所示：
//
// root  ::= expr
// expr  ::= term ([-+*/] term)*
// term  ::= num | "(" space expr ")" space
// num   ::= [0-9]+ space
// space ::= [ \t\n]*
//
// 一旦解析完成，将生成的语法规则存储在parse_state结构中
#pragma once
#include "whisper.h"
#include <vector>
#include <map>
#include <cstdint>
#include <string>

namespace grammar_parser {
    // 定义解析器的状态结构
    struct parse_state {
        // 存储符号名称到ID的映射
        std::map<std::string, uint32_t> symbol_ids;
        // 存储解析后的语法规则
        std::vector<std::vector<whisper_grammar_element>> rules;

        // 返回C风格的语法规则
        std::vector<const whisper_grammar_element *> c_rules() const;
    };

    // 解析给定源代码，返回解析后的状态
    parse_state parse(const char * src);
    // 打印解析后的语法规则到文件
    void print_grammar(FILE * file, const parse_state & state);
}
```