# `PowerInfer\tests\test-grammar-parser.cpp`

```
#ifdef NDEBUG
#undef NDEBUG
#endif

#include "llama.h"  // 包含 llama.h 头文件
#include "grammar-parser.h"  // 包含 grammar-parser.h 头文件

#include <cassert>  // 包含 assert 断言库

int main()  // 主函数
{
    grammar_parser::parse_state parsed_grammar;  // 创建 grammar_parser::parse_state 对象 parsed_grammar

    const char *grammar_bytes = R"""(root  ::= (expr "=" term "\n")+  // 定义包含换行符的规则
expr  ::= term ([-+*/] term)*  // 定义表达式规则
term  ::= [0-9]+)""";  // 定义数字规则

    parsed_grammar = grammar_parser::parse(grammar_bytes);  // 解析 grammar_bytes 并赋值给 parsed_grammar

    std::vector<std::pair<std::string, uint32_t>> expected = {  // 创建期望的符号-标识对的向量
        {"expr", 2},
        {"expr_5", 5},
        {"expr_6", 6},
        {"root", 0},
        {"root_1", 1},
        {"root_4", 4},
        {"term", 3},
        {"term_7", 7},
    };

    uint32_t index = 0;  // 初始化索引为 0
    for (auto it = parsed_grammar.symbol_ids.begin(); it != parsed_grammar.symbol_ids.end(); ++it)  // 遍历 parsed_grammar.symbol_ids
    {
        std::string key = it->first;  // 获取符号
        uint32_t value = it->second;  // 获取标识
        std::pair<std::string, uint32_t> expected_pair = expected[index];  // 获取期望的符号-标识对

        // 在断言之前漂亮地打印错误消息
        if (expected_pair.first != key || expected_pair.second != value)  // 如果期望的符号或标识与实际不符
        {
            fprintf(stderr, "expected_pair: %s, %d\n", expected_pair.first.c_str(), expected_pair.second);  // 打印期望的符号-标识对
            fprintf(stderr, "actual_pair: %s, %d\n", key.c_str(), value);  // 打印实际的符号-标识对
            fprintf(stderr, "expected_pair != actual_pair\n");  // 打印错误消息
        }

        assert(expected_pair.first == key && expected_pair.second == value);  // 断言期望的符号-标识对与实际的符号-标识对相等

        index++;  // 索引加一
    }
}
    // 创建一个期望的语法规则向量，包含不同类型的语法元素和对应的值
    std::vector<llama_grammar_element> expected_rules = {
        {LLAMA_GRETYPE_RULE_REF, 4},  // 期望的规则引用类型和值
        {LLAMA_GRETYPE_END, 0},  // 结束标记
        {LLAMA_GRETYPE_RULE_REF, 2},  // 期望的规则引用类型和值
        {LLAMA_GRETYPE_CHAR, 61},  // 字符类型和值
        {LLAMA_GRETYPE_RULE_REF, 3},  // 期望的规则引用类型和值
        {LLAMA_GRETYPE_CHAR, 10},  // 字符类型和值
        {LLAMA_GRETYPE_END, 0},  // 结束标记
        {LLAMA_GRETYPE_RULE_REF, 3},  // 期望的规则引用类型和值
        {LLAMA_GRETYPE_RULE_REF, 6},  // 期望的规则引用类型和值
        {LLAMA_GRETYPE_END, 0},  // 结束标记
        {LLAMA_GRETYPE_RULE_REF, 7},  // 期望的规则引用类型和值
        {LLAMA_GRETYPE_END, 0},  // 结束标记
        {LLAMA_GRETYPE_RULE_REF, 1},  // 期望的规则引用类型和值
        {LLAMA_GRETYPE_RULE_REF, 4},  // 期望的规则引用类型和值
        {LLAMA_GRETYPE_ALT, 0},  // 备选规则标记
        {LLAMA_GRETYPE_RULE_REF, 1},  // 期望的规则引用类型和值
        {LLAMA_GRETYPE_END, 0},  // 结束标记
        {LLAMA_GRETYPE_CHAR, 45},  // 字符类型和值
        {LLAMA_GRETYPE_CHAR_ALT, 43},  // 字符类型和值
        {LLAMA_GRETYPE_CHAR_ALT, 42},  // 字符类型和值
        {LLAMA_GRETYPE_CHAR_ALT, 47},  // 字符类型和值
        {LLAMA_GRETYPE_RULE_REF, 3},  // 期望的规则引用类型和值
        {LLAMA_GRETYPE_END, 0},  // 结束标记
        {LLAMA_GRETYPE_RULE_REF, 5},  // 期望的规则引用类型和值
        {LLAMA_GRETYPE_RULE_REF, 6},  // 期望的规则引用类型和值
        {LLAMA_GRETYPE_ALT, 0},  // 备选规则标记
        {LLAMA_GRETYPE_END, 0},  // 结束标记
        {LLAMA_GRETYPE_CHAR, 48},  // 字符类型和值
        {LLAMA_GRETYPE_CHAR_RNG_UPPER, 57},  // 字符范围上限
        {LLAMA_GRETYPE_RULE_REF, 7},  // 期望的规则引用类型和值
        {LLAMA_GRETYPE_ALT, 0},  // 备选规则标记
        {LLAMA_GRETYPE_CHAR, 48},  // 字符类型和值
        {LLAMA_GRETYPE_CHAR_RNG_UPPER, 57},  // 字符范围上限
        {LLAMA_GRETYPE_END, 0},  // 结束标记
    };

    // 初始化索引
    index = 0;
    // 遍历解析后的语法规则
    for (auto rule : parsed_grammar.rules)
    {
        // 比较规则和期望规则
        for (uint32_t i = 0; i < rule.size(); i++)
        {
            // 获取当前规则元素和期望规则元素
            llama_grammar_element element = rule[i];
            llama_grammar_element expected_element = expected_rules[index];

            // 在断言之前漂亮地打印错误消息
            if (expected_element.type != element.type || expected_element.value != element.value)
            {
                fprintf(stderr, "index: %d\n", index);
                fprintf(stderr, "expected_element: %d, %d\n", expected_element.type, expected_element.value);
                fprintf(stderr, "actual_element: %d, %d\n", element.type, element.value);
                fprintf(stderr, "expected_element != actual_element\n");
            }

            // 断言当前元素和期望元素相等
            assert(expected_element.type == element.type && expected_element.value == element.value);
            index++;
        }
    }

    // 定义长字符串形式的语法规则
    const char *longer_grammar_bytes = R"""(
    root  ::= (expr "=" ws term "\n")+
    expr  ::= term ([-+*/] term)*
    term  ::= ident | num | "(" ws expr ")" ws
    ident ::= [a-z] [a-z0-9_]* ws
    num   ::= [0-9]+ ws
    ws    ::= [ \t\n]*
    )""";

    // 解析长字符串形式的语法规则
    parsed_grammar = grammar_parser::parse(longer_grammar_bytes);

    // 定义期望的语法规则
    expected = {
        {"expr", 2},
        {"expr_6", 6},
        {"expr_7", 7},
        {"ident", 8},
        {"ident_10", 10},
        {"num", 9},
        {"num_11", 11},
        {"root", 0},
        {"root_1", 1},
        {"root_5", 5},
        {"term", 4},
        {"ws", 3},
        {"ws_12", 12},
    };

    // 初始化索引
    index = 0;
    // 遍历解析后的语法规则的符号 ID
    for (auto it = parsed_grammar.symbol_ids.begin(); it != parsed_grammar.symbol_ids.end(); ++it)
    {
        // 从迭代器中获取键值对
        std::string key = it->first;
        uint32_t value = it->second;
        // 获取期望的键值对
        std::pair<std::string, uint32_t> expected_pair = expected[index];

        // 在断言之前打印错误消息
        if (expected_pair.first != key || expected_pair.second != value)
        {
            fprintf(stderr, "expected_pair: %s, %d\n", expected_pair.first.c_str(), expected_pair.second);
            fprintf(stderr, "actual_pair: %s, %d\n", key.c_str(), value);
            fprintf(stderr, "expected_pair != actual_pair\n");
        }

        // 断言期望的键值对与实际的键值对相等
        assert(expected_pair.first == key && expected_pair.second == value);

        index++;
    }
    };

    index = 0;
    for (auto rule : parsed_grammar.rules)
    {
        // 比较规则与期望的规则
        for (uint32_t i = 0; i < rule.size(); i++)
        {
            // 获取规则中的元素
            llama_grammar_element element = rule[i];
            // 获取期望的规则中的元素
            llama_grammar_element expected_element = expected_rules[index];

            // 在断言之前打印错误消息
            if (expected_element.type != element.type || expected_element.value != element.value)
            {
                fprintf(stderr, "index: %d\n", index);
                fprintf(stderr, "expected_element: %d, %d\n", expected_element.type, expected_element.value);
                fprintf(stderr, "actual_element: %d, %d\n", element.type, element.value);
                fprintf(stderr, "expected_element != actual_element\n");
            }

            // 断言期望的元素与实际的元素相等
            assert(expected_element.type == element.type && expected_element.value == element.value);
            index++;
        }
    }

    return 0;
# 闭合前面的函数定义
```