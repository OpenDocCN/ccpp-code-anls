# `PowerInfer\tests\test-llama-grammar.cpp`

```
#ifdef NDEBUG
#undef NDEBUG
#endif
// 如果定义了 NDEBUG 宏，则取消定义 NDEBUG 宏
#include "llama.cpp" // TODO: not great
// 包含 llama.cpp 文件，但是注释中提到这不是一个好的做法
#include "grammar-parser.h"
// 包含 grammar-parser.h 文件

#include <cassert>
// 包含 assert 头文件

int main()
{
    grammar_parser::parse_state parsed_grammar;
    // 创建 grammar_parser::parse_state 对象 parsed_grammar

    std::vector<std::pair<std::string, uint32_t>> expected = {
        // 创建一个包含字符串和无符号整数对的 vector
        {"expr", 2},
        {"expr_6", 6},
        {"expr_7", 7},
        {"ident", 8},
        {"ident_10", 10},
        {"num", 9},
        // 向 vector 中添加字符串和无符号整数对
// 创建一个包含字符串和对应整数的无序映射
{
    {"num_11", 11},
    {"root", 0},
    {"root_1", 1},
    {"root_5", 5},
    {"term", 4},
    {"ws", 3},
    {"ws_12", 12},
};

// 创建一个包含 llama_grammar_element 对象的二维向量
std::vector<std::vector<llama_grammar_element>> expected_rules = {
    // 第一个元素
    {{LLAMA_GRETYPE_RULE_REF, 5}, {LLAMA_GRETYPE_END, 0}},
    // 第二个元素
    {
        {LLAMA_GRETYPE_RULE_REF, 2},
        {LLAMA_GRETYPE_CHAR, 61},
        {LLAMA_GRETYPE_RULE_REF, 3},
        {LLAMA_GRETYPE_RULE_REF, 4},
        {LLAMA_GRETYPE_CHAR, 10},
        {LLAMA_GRETYPE_END, 0},
    },
    // 第三个元素
    {{LLAMA_GRETYPE_RULE_REF, 4}, {LLAMA_GRETYPE_RULE_REF, 7}, {LLAMA_GRETYPE_END, 0}},
};
// 定义一个规则，包含规则引用和结束标记
{{LLAMA_GRETYPE_RULE_REF, 12}, {LLAMA_GRETYPE_END, 0}},

// 定义一个规则，包含多个备选项，每个备选项包含规则引用或字符
{
    {LLAMA_GRETYPE_RULE_REF, 8}, // 规则引用
    {LLAMA_GRETYPE_ALT, 0}, // 备选项分隔符
    {LLAMA_GRETYPE_RULE_REF, 9}, // 规则引用
    {LLAMA_GRETYPE_ALT, 0}, // 备选项分隔符
    {LLAMA_GRETYPE_CHAR, 40}, // 字符 '('
    {LLAMA_GRETYPE_RULE_REF, 3}, // 规则引用
    {LLAMA_GRETYPE_RULE_REF, 2}, // 规则引用
    {LLAMA_GRETYPE_CHAR, 41}, // 字符 ')'
    {LLAMA_GRETYPE_RULE_REF, 3}, // 规则引用
    {LLAMA_GRETYPE_END, 0}, // 结束标记
},

// 定义一个规则，包含多个规则引用和备选项分隔符
{{LLAMA_GRETYPE_RULE_REF, 1}, {LLAMA_GRETYPE_RULE_REF, 5}, {LLAMA_GRETYPE_ALT, 0}, {LLAMA_GRETYPE_RULE_REF, 1}, {LLAMA_GRETYPE_END, 0}},

// 定义一个规则，包含多个字符和规则引用
{
    {LLAMA_GRETYPE_CHAR, 45}, // 字符 '-'
    {LLAMA_GRETYPE_CHAR_ALT, 43}, // 字符 '+'
    {LLAMA_GRETYPE_CHAR_ALT, 42}, // 字符 '*'
    {LLAMA_GRETYPE_CHAR_ALT, 47}, // 字符 '/'
    {LLAMA_GRETYPE_RULE_REF, 4}, // 规则引用
}
        {{LLAMA_GRETYPE_END, 0},  // 表示规则结束
        },
        {{LLAMA_GRETYPE_RULE_REF, 6}, {LLAMA_GRETYPE_RULE_REF, 7}, {LLAMA_GRETYPE_ALT, 0}, {LLAMA_GRETYPE_END, 0}},  // 表示规则引用和替代
        {
            {LLAMA_GRETYPE_CHAR, 97},  // 表示字符 'a'
            {LLAMA_GRETYPE_CHAR_RNG_UPPER, 122},  // 表示字符范围 'a' 到 'z'
            {LLAMA_GRETYPE_RULE_REF, 10},  // 表示规则引用
            {LLAMA_GRETYPE_RULE_REF, 3},  // 表示规则引用
            {LLAMA_GRETYPE_END, 0},  // 表示规则结束
        },
        {{LLAMA_GRETYPE_RULE_REF, 11}, {LLAMA_GRETYPE_RULE_REF, 3}, {LLAMA_GRETYPE_END, 0}},  // 表示规则引用和结束
        {
            {LLAMA_GRETYPE_CHAR, 97},  // 表示字符 'a'
            {LLAMA_GRETYPE_CHAR_RNG_UPPER, 122},  // 表示字符范围 'a' 到 'z'
            {LLAMA_GRETYPE_CHAR_ALT, 48},  // 表示字符 '0'
            {LLAMA_GRETYPE_CHAR_RNG_UPPER, 57},  // 表示字符范围 '0' 到 '9'
            {LLAMA_GRETYPE_CHAR_ALT, 95},  // 表示字符 '_'
            {LLAMA_GRETYPE_RULE_REF, 10},  // 表示规则引用
            {LLAMA_GRETYPE_ALT, 0},  // 表示替代
            {LLAMA_GRETYPE_END, 0},  // 表示规则结束
// 定义一个包含多个子数组的数组
{
    // 子数组1
    {LLAMA_GRETYPE_CHAR, 48}, // 表示字符类型为48
    {LLAMA_GRETYPE_CHAR_RNG_UPPER, 57}, // 表示字符范围为大写字母A-Z
    {LLAMA_GRETYPE_RULE_REF, 11}, // 表示引用规则11
    {LLAMA_GRETYPE_ALT, 0}, // 表示备选规则
    {LLAMA_GRETYPE_CHAR, 48}, // 表示字符类型为48
    {LLAMA_GRETYPE_CHAR_RNG_UPPER, 57}, // 表示字符范围为大写字母A-Z
    {LLAMA_GRETYPE_END, 0}, // 表示结束
},
{
    // 子数组2
    {LLAMA_GRETYPE_CHAR, 32}, // 表示字符类型为32
    {LLAMA_GRETYPE_CHAR_ALT, 9}, // 表示字符类型为9
    {LLAMA_GRETYPE_CHAR_ALT, 10}, // 表示字符类型为10
    {LLAMA_GRETYPE_RULE_REF, 12}, // 表示引用规则12
    {LLAMA_GRETYPE_ALT, 0}, // 表示备选规则
    {LLAMA_GRETYPE_END, 0}, // 表示结束
},
};
    // 遍历 expected 中的每对键值对
    for (auto pair : expected)
    {
        // 将键值对中的第一个元素作为键，第二个元素作为值，存入 parsed_grammar.symbol_ids 中
        parsed_grammar.symbol_ids[pair.first] = pair.second;
    }

    // 遍历 expected_rules 中的每个规则
    for (auto rule : expected_rules)
    {
        // 在 parsed_grammar.rules 中添加一个空的规则
        parsed_grammar.rules.push_back({});
        // 遍历当前规则中的每个元素
        for (auto element : rule)
        {
            // 将当前元素添加到最后一个规则中
            parsed_grammar.rules.back().push_back(element);
        }
    }

    // 初始化一个 llama_grammar 指针，并传入相应的参数
    llama_grammar *grammar = NULL;
    // 创建一个包含解析后规则的 vector
    std::vector<const llama_grammar_element *> grammar_rules(parsed_grammar.c_rules());
    // 初始化 grammar 指针，传入解析后的规则和根节点的 symbol_id
    grammar = llama_grammar_init(
        grammar_rules.data(), grammar_rules.size(), parsed_grammar.symbol_ids.at("root"));

    // 创建一个包含期望堆栈的 vector
    std::vector<std::vector<llama_grammar_element>> expected_stacks = {
// 创建一个包含规则引用和字符的数组
{
    {LLAMA_GRETYPE_RULE_REF, 5}, // 规则引用为5
    {LLAMA_GRETYPE_CHAR, 61}, // 字符为61
    {LLAMA_GRETYPE_RULE_REF, 7}, // 规则引用为7
    {LLAMA_GRETYPE_CHAR, 97}, // 字符为97
},
{
    {LLAMA_GRETYPE_RULE_REF, 5}, // 规则引用为5
    {LLAMA_GRETYPE_CHAR, 61}, // 字符为61
    {LLAMA_GRETYPE_RULE_REF, 7}, // 规则引用为7
    {LLAMA_GRETYPE_RULE_REF, 3}, // 规则引用为3
    {LLAMA_GRETYPE_CHAR, 48}, // 字符为48
},
{
    {LLAMA_GRETYPE_RULE_REF, 5}, // 规则引用为5
    {LLAMA_GRETYPE_CHAR, 61}, // 字符为61
    {LLAMA_GRETYPE_RULE_REF, 7}, // 规则引用为7
    {LLAMA_GRETYPE_RULE_REF, 3}, // 规则引用为3
    {LLAMA_GRETYPE_CHAR, 48}, // 字符为48
},
// 创建一个包含规则引用和字符的数组，每个元素包含规则引用和字符的类型和值
{
    {LLAMA_GRETYPE_RULE_REF, 5},  // 规则引用类型和值
    {LLAMA_GRETYPE_CHAR, 61},     // 字符类型和值
    {LLAMA_GRETYPE_RULE_REF, 7},  // 规则引用类型和值
    {LLAMA_GRETYPE_CHAR, 40},     // 字符类型和值
},
{
    {LLAMA_GRETYPE_CHAR, 61},     // 字符类型和值
    {LLAMA_GRETYPE_RULE_REF, 7},  // 规则引用类型和值
    {LLAMA_GRETYPE_CHAR, 97},     // 字符类型和值
},
{
    {LLAMA_GRETYPE_CHAR, 61},     // 字符类型和值
    {LLAMA_GRETYPE_RULE_REF, 7},  // 规则引用类型和值
    {LLAMA_GRETYPE_RULE_REF, 3},  // 规则引用类型和值
    {LLAMA_GRETYPE_CHAR, 48},     // 字符类型和值
},
{
    {LLAMA_GRETYPE_CHAR, 61},     // 字符类型和值
    {LLAMA_GRETYPE_RULE_REF, 7},  // 规则引用类型和值
}
// 定义一个二维数组，包含LLAMA_GRETYPE_RULE_REF和LLAMA_GRETYPE_CHAR类型的元素
{
    {LLAMA_GRETYPE_RULE_REF, 3},
    {LLAMA_GRETYPE_CHAR, 48},
},
{
    {LLAMA_GRETYPE_CHAR, 61},
    {LLAMA_GRETYPE_RULE_REF, 7},
    {LLAMA_GRETYPE_CHAR, 40},
}

// 初始化index变量为0
auto index = 0;
// 遍历语法的栈
for (auto stack : grammar->stacks)
{
    // 比较栈中的元素和期望的栈中的元素
    for (uint32_t i = 0; i < stack.size(); i++)
    {
        // 获取栈中的元素和期望的栈中的元素
        auto element = stack[i];
        auto expected_element = expected_stacks[index][i];

        // 在断言之前漂亮地打印错误消息
        if (expected_element.type != element->type || expected_element.value != element->value)
// 打印调试信息，输出索引值
fprintf(stderr, "index: %d\n", index);
// 打印调试信息，输出期望元素的类型和值
fprintf(stderr, "expected_element: %d, %d\n", expected_element.type, expected_element.value);
// 打印调试信息，输出实际元素的类型和值
fprintf(stderr, "actual_element: %d, %d\n", element->type, element->value);
// 打印调试信息，输出期望元素不等于实际元素的消息
fprintf(stderr, "expected_element != actual_element\n");

// 使用断言检查期望元素的类型和值是否等于实际元素的类型和值
assert(expected_element.type == element->type && expected_element.value == element->value);

// 增加索引值
index++;
}

// 创建一个二维向量，用于存储下一个状态的候选元素
std::vector<std::vector<const llama_grammar_element *>> next_stacks;
// 创建一个存储下一个候选元素的向量，并设置大小为24
std::vector<llama_grammar_candidate> next_candidates;
next_candidates.resize(24);

// 循环遍历24次
for (size_t i = 0; i < 24; ++i)
{
    // 动态分配内存给code_point
    uint32_t *cp = new uint32_t[2];
    // 给cp赋值，37 + i
    cp[0] = 37 + i;
        cp[1] = 0; // 将数组 cp 的第二个元素赋值为 0
        next_candidates[i] = {i, cp, {}}; // 将 i、cp 和空的字典组成一个元组，赋值给 next_candidates 的第 i 个元素
    }

    std::vector<std::vector<std::pair<uint32_t, uint16_t>>> expected_reject = { // 创建一个二维向量 expected_reject
        { // 第一个子向量
            {0, 37}, // 一个 pair，包含 0 和 37
            {1, 38}, // 一个 pair，包含 1 和 38
            {2, 39}, // 一个 pair，包含 2 和 39
            {3, 40}, // 一个 pair，包含 3 和 40
            {4, 41}, // 一个 pair，包含 4 和 41
            {5, 42}, // 一个 pair，包含 5 和 42
            {6, 43}, // 一个 pair，包含 6 和 43
            {7, 44}, // 一个 pair，包含 7 和 44
            {8, 45}, // 一个 pair，包含 8 和 45
            {9, 46}, // 一个 pair，包含 9 和 46
            {10, 47}, // 一个 pair，包含 10 和 47
            {11, 48}, // 一个 pair，包含 11 和 48
            {12, 49}, // 一个 pair，包含 12 和 49
            {13, 50}, // 一个 pair，包含 13 和 50
# 创建一个包含多个子集的二维数组
{
    # 第一个子集
    {
        # 子集中的元素
        {14, 51},
        {15, 52},
        {16, 53},
        {17, 54},
        {18, 55},
        {19, 56},
        {20, 57},
        {21, 58},
        {22, 59},
        {23, 60},
    },
    # 第二个子集
    {
        # 子集中的元素
        {0, 37},
        {1, 38},
        {2, 39},
        {3, 40},
        {4, 41},
        {5, 42},
        {6, 43},
        {7, 44},
    }
}
# 以下是一个二维数组，每个元素都是一个包含两个数字的元组，表示索引和值的对应关系
{
    # 第一个子数组
    {8, 45},  # 索引 8 对应值 45
    {9, 46},  # 索引 9 对应值 46
    {10, 47},  # 索引 10 对应值 47
    {21, 58},  # 索引 21 对应值 58
    {22, 59},  # 索引 22 对应值 59
    {23, 60},  # 索引 23 对应值 60
},
{
    # 第二个子数组
    {0, 37},  # 索引 0 对应值 37
    {1, 38},  # 索引 1 对应值 38
    {2, 39},  # 索引 2 对应值 39
    {3, 40},  # 索引 3 对应值 40
    {4, 41},  # 索引 4 对应值 41
    {5, 42},  # 索引 5 对应值 42
    {6, 43},  # 索引 6 对应值 43
    {7, 44},  # 索引 7 对应值 44
    {8, 45},  # 索引 8 对应值 45
    {9, 46},  # 索引 9 对应值 46
    {10, 47},  # 索引 10 对应值 47
    {21, 58},  # 索引 21 对应值 58
}
        {
            {22, 59},  # 第一个子集，包含元素22和59
            {23, 60},  # 第一个子集，包含元素23和60
        },
        {
            {0, 37},   # 第二个子集，包含元素0和37
            {1, 38},   # 第二个子集，包含元素1和38
            {2, 39},   # 第二个子集，包含元素2和39
            {4, 41},   # 第二个子集，包含元素4和41
            {5, 42},   # 第二个子集，包含元素5和42
            {6, 43},   # 第二个子集，包含元素6和43
            {7, 44},   # 第二个子集，包含元素7和44
            {8, 45},   # 第二个子集，包含元素8和45
            {9, 46},   # 第二个子集，包含元素9和46
            {10, 47},  # 第二个子集，包含元素10和47
            {11, 48},  # 第二个子集，包含元素11和48
            {12, 49},  # 第二个子集，包含元素12和49
            {13, 50},  # 第二个子集，包含元素13和50
            {14, 51},  # 第二个子集，包含元素14和51
            {15, 52},  # 第二个子集，包含元素15和52
            {16, 53},  # 第二个子集，包含元素16和53
# 以下是一个二维数组，每个元素都是一个包含两个数字的集合
# 第一个数字表示某种索引，第二个数字表示对应的数值
# 例如，{17, 54} 表示索引为17的位置对应的数值为54
# 以下是一个二维数组，每个元素都是一个包含两个数字的元组
# 第一个数字表示索引，第二个数字表示值
# 第一个大括号内的元组表示第一行的索引和值
# 第二个大括号内的元组表示第二行的索引和值
# 依此类推
# 创建一个包含多个集合的列表
{
    # 第一个集合
    {5, 42},  # 包含两个元素 5 和 42
    {6, 43},  # 包含两个元素 6 和 43
    {7, 44},  # 包含两个元素 7 和 44
    {8, 45},  # 包含两个元素 8 和 45
    {9, 46},  # 包含两个元素 9 和 46
    {10, 47},  # 包含两个元素 10 和 47
    {21, 58},  # 包含两个元素 21 和 58
    {22, 59},  # 包含两个元素 22 和 59
    {23, 60},  # 包含两个元素 23 和 60
},
{
    # 第二个集合
    {0, 37},  # 包含两个元素 0 和 37
    {1, 38},  # 包含两个元素 1 和 38
    {2, 39},  # 包含两个元素 2 和 39
    {3, 40},  # 包含两个元素 3 和 40
    {4, 41},  # 包含两个元素 4 和 41
    {5, 42},  # 包含两个元素 5 和 42
    {6, 43},  # 包含两个元素 6 和 43
    {7, 44},  # 包含两个元素 7 和 44
    {8, 45},  # 包含两个元素 8 和 45
}
# 以下是一个二维数组，每个元素都是一个包含两个数字的集合
{
    # 第一个集合
    {9, 46},
    {10, 47},
    {21, 58},
    {22, 59},
    {23, 60},
},
{
    # 第二个集合
    {0, 37},
    {1, 38},
    {2, 39},
    {4, 41},
    {5, 42},
    {6, 43},
    {7, 44},
    {8, 45},
    {9, 46},
    {10, 47},
    {11, 48},
    {12, 49},
    {13, 50},
}
    // 定义一个二维数组，包含一系列的数字对
    {
        {14, 51},
        {15, 52},
        {16, 53},
        {17, 54},
        {18, 55},
        {19, 56},
        {20, 57},
        {21, 58},
        {22, 59},
        {23, 60},
    },
};

// 调用 llama_grammar_reject_candidates_for_stack 函数，获取拒绝的候选项
std::vector<llama_grammar_candidate> rejects = llama_grammar_reject_candidates_for_stack(grammar->rules, grammar->stacks[0], next_candidates);

// 定义一个二维数组，用于存储所有拒绝的候选项
std::vector<std::vector<llama_grammar_candidate>> all_rejects;

// 遍历语法栈，获取每个栈的拒绝候选项
for (std::size_t count = 0; count < grammar->stacks.size(); ++count)
{
    rejects = llama_grammar_reject_candidates_for_stack(grammar->rules, grammar->stacks[count], next_candidates);
    // 将rejects添加到all_rejects的末尾
    all_rejects.push_back(rejects);
    }

    // 重置index为0，遍历all_rejects中的每个rejects
    for (auto rej : all_rejects)
    {
        // 遍历每个rejects中的元素
        for (uint32_t i = 0; i < rej.size(); i++)
        {
            // 获取当前元素和对应的期望元素
            auto element = rej[i];
            auto expected_element = expected_reject[index][i];
            // 断言当前元素的索引和数据与期望元素相符
            assert(element.index == expected_element.first && *element.code_points == expected_element.second);
        }
        // 增加index以匹配期望rejects的索引
        index++;
    }

    // 释放next_candidates中每个candidate的内存，并将指针置为空
    for (auto &candidate : next_candidates)
    {
        delete[] candidate.code_points;
        candidate.code_points = nullptr;
    }
    // 删除语法对象
    delete grammar;
    // 返回 0，表示程序正常结束
    return 0;
}
```