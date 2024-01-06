# `PowerInfer\common\grammar-parser.h`

```
// 实现了对扩展巴科斯-瑙尔范式（BNF）的解析器，生成 llama.h 指定的二进制无上下文语法格式。支持字符范围、分组和重复操作符。例如，算术的语法可能如下所示：

// 根 ::= 表达式
// 表达式 ::= 项 ([-+*/] 项)*
// 项 ::= 数字 | "(" 空格 表达式 ")" 空格
// 数字 ::= [0-9]+ 空格
// 空格 ::= [ \t\n]*

// 防止头文件重复包含
#pragma once
// 包含 llama.h 头文件
#include "llama.h"
// 包含必要的标准库头文件
#include <vector>
#include <map>
#include <cstdint>
#include <string>

// 命名空间 grammar_parser
namespace grammar_parser {
    // 定义 parse_state 结构体
    struct parse_state {
        // 结构体成员的定义将在下面的代码中继续
// 声明一个字符串到无符号整数的映射
std::map<std::string, uint32_t> symbol_ids;

// 声明一个包含 llama_grammar_element 元素的二维向量
std::vector<std::vector<llama_grammar_element>> rules;

// 声明一个返回指向 llama_grammar_element 元素的指针的向量
std::vector<const llama_grammar_element *> c_rules();

// 声明一个解析函数，接受一个指向字符的指针作为参数
parse_state parse(const char * src);

// 声明一个打印语法函数，接受一个文件指针和 parse_state 对象作为参数
void print_grammar(FILE * file, const parse_state & state);
```