# `PowerInfer\tests\test-llama-grammar.cpp`

```
#ifdef NDEBUG
#undef NDEBUG
#endif

#include "llama.cpp" // TODO: not great
#include "grammar-parser.h"

#include <cassert>

int main()
{
    // 创建解析后的语法对象
    grammar_parser::parse_state parsed_grammar;

    // 预期的符号和对应的 ID
    std::vector<std::pair<std::string, uint32_t>> expected = {
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

    // 将预期的符号和 ID 添加到解析后的语法对象中
    for (auto pair : expected)
    {
        parsed_grammar.symbol_ids[pair.first] = pair.second;
    }

    // 将预期的规则添加到解析后的语法对象中
    for (auto rule : expected_rules)
    {
        parsed_grammar.rules.push_back({});
        for (auto element : rule)
        {
            parsed_grammar.rules.back().push_back(element);
        }
    }

    // 初始化 llama_grammar 对象
    llama_grammar *grammar = NULL;
    std::vector<const llama_grammar_element *> grammar_rules(parsed_grammar.c_rules());
    grammar = llama_grammar_init(
        grammar_rules.data(), grammar_rules.size(), parsed_grammar.symbol_ids.at("root"));
}
    // 创建一个二维向量，存储期望的语法栈
    std::vector<std::vector<llama_grammar_element>> expected_stacks = {
        // 第一个期望的语法栈
        {
            {LLAMA_GRETYPE_RULE_REF, 5},  // 规则引用，规则编号为5
            {LLAMA_GRETYPE_CHAR, 61},     // 字符，ASCII码为61
            {LLAMA_GRETYPE_RULE_REF, 7},  // 规则引用，规则编号为7
            {LLAMA_GRETYPE_CHAR, 97},     // 字符，ASCII码为97
        },
        // 其他期望的语法栈...
    };

    // 初始化索引
    auto index = 0;
    // 遍历语法栈
    for (auto stack : grammar->stacks)
    {
        // 将栈与期望栈进行比较
        for (uint32_t i = 0; i < stack.size(); i++)
        {
            auto element = stack[i]; // 获取栈中元素
            auto expected_element = expected_stacks[index][i]; // 获取期望栈中对应位置的元素

            // 在断言之前打印漂亮的错误消息
            if (expected_element.type != element->type || expected_element.value != element->value)
            {
                fprintf(stderr, "index: %d\n", index); // 打印索引
                fprintf(stderr, "expected_element: %d, %d\n", expected_element.type, expected_element.value); // 打印期望元素
                fprintf(stderr, "actual_element: %d, %d\n", element->type, element->value); // 打印实际元素
                fprintf(stderr, "expected_element != actual_element\n"); // 打印错误消息
            }

            assert(expected_element.type == element->type && expected_element.value == element->value); // 断言期望元素与实际元素相等
        }
        index++; // 索引自增
    }

    std::vector<std::vector<const llama_grammar_element *>> next_stacks; // 创建下一个栈的二维向量
    std::vector<llama_grammar_candidate> next_candidates; // 创建下一个候选项的向量
    next_candidates.resize(24); // 调整下一个候选项的大小为24

    for (size_t i = 0; i < 24; ++i) // 遍历24次
    {
        uint32_t *cp = new uint32_t[2]; // 动态分配内存给 code_point
        cp[0] = 37 + i; // 设置第一个元素的值
        cp[1] = 0; // 设置第二个元素的值
        next_candidates[i] = {i, cp, {}}; // 将候选项添加到下一个候选项向量中
    }

    };

    std::vector<llama_grammar_candidate> rejects = llama_grammar_reject_candidates_for_stack(grammar->rules, grammar->stacks[0], next_candidates); // 获取拒绝的候选项

    std::vector<std::vector<llama_grammar_candidate>> all_rejects; // 创建所有拒绝的候选项的二维向量

    for (std::size_t count = 0; count < grammar->stacks.size(); ++count) // 遍历语法栈的大小
    {
        rejects = llama_grammar_reject_candidates_for_stack(grammar->rules, grammar->stacks[count], next_candidates); // 获取拒绝的候选项
        all_rejects.push_back(rejects); // 将拒绝的候选项添加到所有拒绝的候选项向量中
    }

    index = 0; // 重置索引为0
    for (auto rej : all_rejects) // 遍历所有拒绝的候选项
    {
        // 遍历拒绝集合中的元素
        for (uint32_t i = 0; i < rej.size(); i++)
        {
            // 获取当前元素
            auto element = rej[i];
            // 获取期望的拒绝元素
            auto expected_element = expected_reject[index][i];
            // 断言当前元素的索引和代码点与期望的元素相匹配
            assert(element.index == expected_element.first && *element.code_points == expected_element.second);
        }
        // 指向下一个期望的拒绝元素
        index++;
    }

    // 遍历下一个候选集合
    for (auto &candidate : next_candidates)
    {
        // 释放候选元素的代码点内存
        delete[] candidate.code_points;
        candidate.code_points = nullptr;
    }
    // 释放语法对象的内存
    delete grammar;
    // 返回 0 表示成功
    return 0;
}
# 闭合前面的函数定义
```