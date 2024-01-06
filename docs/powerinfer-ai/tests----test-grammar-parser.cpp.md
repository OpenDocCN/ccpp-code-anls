# `PowerInfer\tests\test-grammar-parser.cpp`

```
#ifdef NDEBUG
// 如果 NDEBUG 宏已经定义，则取消定义它
#undef NDEBUG
#endif

#include "llama.h"
#include "grammar-parser.h"

#include <cassert>

int main()
{
    // 创建语法解析器的解析状态对象
    grammar_parser::parse_state parsed_grammar;

    // 定义包含语法规则的字符串
    const char *grammar_bytes = R"""(root  ::= (expr "=" term "\n")+
expr  ::= term ([-+*/] term)*
term  ::= [0-9]+)""";

    // 解析语法规则字符串，将结果赋值给 parsed_grammar
    parsed_grammar = grammar_parser::parse(grammar_bytes);

    // 创建包含预期结果的字符串-整数对的向量
    std::vector<std::pair<std::string, uint32_t>> expected = {
    // 创建一个包含预期符号和对应ID的数组
    std::pair<std::string, uint32_t> expected[] = {
        {"expr", 2},
        {"expr_5", 5},
        {"expr_6", 6},
        {"root", 0},
        {"root_1", 1},
        {"root_4", 4},
        {"term", 3},
        {"term_7", 7},
    };

    // 初始化索引值
    uint32_t index = 0;
    // 遍历解析后的语法符号和对应ID的映射
    for (auto it = parsed_grammar.symbol_ids.begin(); it != parsed_grammar.symbol_ids.end(); ++it)
    {
        // 获取当前符号和对应ID
        std::string key = it->first;
        uint32_t value = it->second;
        // 获取预期的符号和对应ID
        std::pair<std::string, uint32_t> expected_pair = expected[index];

        // 在断言之前漂亮地打印错误消息
        if (expected_pair.first != key || expected_pair.second != value)
        {
// 输出预期的键值对
fprintf(stderr, "expected_pair: %s, %d\n", expected_pair.first.c_str(), expected_pair.second);
// 输出实际的键值对
fprintf(stderr, "actual_pair: %s, %d\n", key.c_str(), value);
// 输出预期的键值对不等于实际的键值对
fprintf(stderr, "expected_pair != actual_pair\n");

// 使用断言检查预期的键和值是否等于实际的键和值
assert(expected_pair.first == key && expected_pair.second == value);

// 增加索引
index++;

// 创建期望的规则元素向量
std::vector<llama_grammar_element> expected_rules = {
    {LLAMA_GRETYPE_RULE_REF, 4},
    {LLAMA_GRETYPE_END, 0},
    {LLAMA_GRETYPE_RULE_REF, 2},
    {LLAMA_GRETYPE_CHAR, 61},
    {LLAMA_GRETYPE_RULE_REF, 3},
    {LLAMA_GRETYPE_CHAR, 10},
    {LLAMA_GRETYPE_END, 0},
    {LLAMA_GRETYPE_RULE_REF, 3},
    {LLAMA_GRETYPE_RULE_REF, 6},
    {LLAMA_GRETYPE_END, 0},
};
{LLAMA_GRETYPE_RULE_REF, 7},  // 使用规则引用7
{LLAMA_GRETYPE_END, 0},  // 结束
{LLAMA_GRETYPE_RULE_REF, 1},  // 使用规则引用1
{LLAMA_GRETYPE_RULE_REF, 4},  // 使用规则引用4
{LLAMA_GRETYPE_ALT, 0},  // 选择
{LLAMA_GRETYPE_RULE_REF, 1},  // 使用规则引用1
{LLAMA_GRETYPE_END, 0},  // 结束
{LLAMA_GRETYPE_CHAR, 45},  // 字符'-'
{LLAMA_GRETYPE_CHAR_ALT, 43},  // 字符'+'
{LLAMA_GRETYPE_CHAR_ALT, 42},  // 字符'*'
{LLAMA_GRETYPE_CHAR_ALT, 47},  // 字符'/'
{LLAMA_GRETYPE_RULE_REF, 3},  // 使用规则引用3
{LLAMA_GRETYPE_END, 0},  // 结束
{LLAMA_GRETYPE_RULE_REF, 5},  // 使用规则引用5
{LLAMA_GRETYPE_RULE_REF, 6},  // 使用规则引用6
{LLAMA_GRETYPE_ALT, 0},  // 选择
{LLAMA_GRETYPE_END, 0},  // 结束
{LLAMA_GRETYPE_CHAR, 48},  // 字符'0'
{LLAMA_GRETYPE_CHAR_RNG_UPPER, 57},  // 字符范围'0'到'9'
{LLAMA_GRETYPE_RULE_REF, 7},  // 使用规则引用7
// 定义期望的规则和对应的值
{LLAMA_GRETYPE_ALT, 0},
{LLAMA_GRETYPE_CHAR, 48},
{LLAMA_GRETYPE_CHAR_RNG_UPPER, 57},
{LLAMA_GRETYPE_END, 0},
};

// 初始化索引
index = 0;
// 遍历解析后的语法规则
for (auto rule : parsed_grammar.rules)
{
    // 比较规则和期望的规则
    for (uint32_t i = 0; i < rule.size(); i++)
    {
        // 获取当前规则元素和期望的规则元素
        llama_grammar_element element = rule[i];
        llama_grammar_element expected_element = expected_rules[index];

        // 在断言之前打印错误信息
        if (expected_element.type != element.type || expected_element.value != element.value)
        {
            fprintf(stderr, "index: %d\n", index);
            fprintf(stderr, "expected_element: %d, %d\n", expected_element.type, expected_element.value);
// 打印实际元素的类型和值
fprintf(stderr, "actual_element: %d, %d\n", element.type, element.value);
// 打印预期元素不等于实际元素的消息
fprintf(stderr, "expected_element != actual_element\n");

// 使用断言检查预期元素的类型和值是否与实际元素相匹配
assert(expected_element.type == element.type && expected_element.value == element.value);
// 增加索引以继续遍历元素

// 定义一个长字符串表示的语法规则
const char *longer_grammar_bytes = R"""(
root  ::= (expr "=" ws term "\n")+
expr  ::= term ([-+*/] term)*
term  ::= ident | num | "(" ws expr ")" ws
ident ::= [a-z] [a-z0-9_]* ws
num   ::= [0-9]+ ws
ws    ::= [ \t\n]*
)""";

// 使用语法解析器解析长字符串表示的语法规则
parsed_grammar = grammar_parser::parse(longer_grammar_bytes);
# 创建一个预期的结果字典，包含字符串键和整数值
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

# 初始化索引
index = 0;

# 遍历解析语法中的符号和对应的ID
for (auto it = parsed_grammar.symbol_ids.begin(); it != parsed_grammar.symbol_ids.end(); ++it)
{
    # 获取当前符号的键
    std::string key = it->first;
        // 从映射中获取值，并赋给变量value
        uint32_t value = it->second;
        // 从期望的pair中获取键值对，用于后续比较
        std::pair<std::string, uint32_t> expected_pair = expected[index];

        // 在断言之前，打印出错信息
        if (expected_pair.first != key || expected_pair.second != value)
        {
            // 打印期望的pair的键和值
            fprintf(stderr, "expected_pair: %s, %d\n", expected_pair.first.c_str(), expected_pair.second);
            // 打印实际的pair的键和值
            fprintf(stderr, "actual_pair: %s, %d\n", key.c_str(), value);
            // 打印出错信息
            fprintf(stderr, "expected_pair != actual_pair\n");
        }

        // 断言期望的pair的键和值与实际的pair的键和值相等
        assert(expected_pair.first == key && expected_pair.second == value);

        // 增加索引，用于下一次循环
        index++;
    }
    // 期望的规则
    expected_rules = {
        {LLAMA_GRETYPE_RULE_REF, 5},
        {LLAMA_GRETYPE_END, 0},
        {LLAMA_GRETYPE_RULE_REF, 2},
        {LLAMA_GRETYPE_CHAR, 61},
{LLAMA_GRETYPE_RULE_REF, 3},  // 使用规则引用3
{LLAMA_GRETYPE_RULE_REF, 4},  // 使用规则引用4
{LLAMA_GRETYPE_CHAR, 10},    // 使用字符10
{LLAMA_GRETYPE_END, 0},       // 结束
{LLAMA_GRETYPE_RULE_REF, 4},  // 使用规则引用4
{LLAMA_GRETYPE_RULE_REF, 7},  // 使用规则引用7
{LLAMA_GRETYPE_END, 0},       // 结束
{LLAMA_GRETYPE_RULE_REF, 12}, // 使用规则引用12
{LLAMA_GRETYPE_END, 0},       // 结束
{LLAMA_GRETYPE_RULE_REF, 8},  // 使用规则引用8
{LLAMA_GRETYPE_ALT, 0},       // 替代
{LLAMA_GRETYPE_RULE_REF, 9},  // 使用规则引用9
{LLAMA_GRETYPE_ALT, 0},       // 替代
{LLAMA_GRETYPE_CHAR, 40},     // 使用字符40
{LLAMA_GRETYPE_RULE_REF, 3},  // 使用规则引用3
{LLAMA_GRETYPE_RULE_REF, 2},  // 使用规则引用2
{LLAMA_GRETYPE_CHAR, 41},     // 使用字符41
{LLAMA_GRETYPE_RULE_REF, 3},  // 使用规则引用3
{LLAMA_GRETYPE_END, 0},       // 结束
{LLAMA_GRETYPE_RULE_REF, 1},  // 使用规则引用1
{LLAMA_GRETYPE_RULE_REF, 5},  // 使用规则引用5
{LLAMA_GRETYPE_ALT, 0},  // 使用备选项0
{LLAMA_GRETYPE_RULE_REF, 1},  // 使用规则引用1
{LLAMA_GRETYPE_END, 0},  // 结束0
{LLAMA_GRETYPE_CHAR, 45},  // 使用字符45
{LLAMA_GRETYPE_CHAR_ALT, 43},  // 使用备选字符43
{LLAMA_GRETYPE_CHAR_ALT, 42},  // 使用备选字符42
{LLAMA_GRETYPE_CHAR_ALT, 47},  // 使用备选字符47
{LLAMA_GRETYPE_RULE_REF, 4},  // 使用规则引用4
{LLAMA_GRETYPE_END, 0},  // 结束0
{LLAMA_GRETYPE_RULE_REF, 6},  // 使用规则引用6
{LLAMA_GRETYPE_RULE_REF, 7},  // 使用规则引用7
{LLAMA_GRETYPE_ALT, 0},  // 使用备选项0
{LLAMA_GRETYPE_END, 0},  // 结束0
{LLAMA_GRETYPE_CHAR, 97},  // 使用字符97
{LLAMA_GRETYPE_CHAR_RNG_UPPER, 122},  // 使用字符范围上限122
{LLAMA_GRETYPE_RULE_REF, 10},  // 使用规则引用10
{LLAMA_GRETYPE_RULE_REF, 3},  // 使用规则引用3
{LLAMA_GRETYPE_END, 0},  // 结束0
{LLAMA_GRETYPE_RULE_REF, 11},  // 使用规则引用11
# 定义 LLAMA_GRETYPE_RULE_REF 类型的规则引用，值为3
{LLAMA_GRETYPE_RULE_REF, 3},
# 定义 LLAMA_GRETYPE_END 类型的结束标记，值为0
{LLAMA_GRETYPE_END, 0},
# 定义 LLAMA_GRETYPE_CHAR 类型的字符，值为97
{LLAMA_GRETYPE_CHAR, 97},
# 定义 LLAMA_GRETYPE_CHAR_RNG_UPPER 类型的字符范围上限，值为122
{LLAMA_GRETYPE_CHAR_RNG_UPPER, 122},
# 定义 LLAMA_GRETYPE_CHAR_ALT 类型的备用字符，值为48
{LLAMA_GRETYPE_CHAR_ALT, 48},
# 定义 LLAMA_GRETYPE_CHAR_RNG_UPPER 类型的字符范围上限，值为57
{LLAMA_GRETYPE_CHAR_RNG_UPPER, 57},
# 定义 LLAMA_GRETYPE_CHAR_ALT 类型的备用字符，值为95
{LLAMA_GRETYPE_CHAR_ALT, 95},
# 定义 LLAMA_GRETYPE_RULE_REF 类型的规则引用，值为10
{LLAMA_GRETYPE_RULE_REF, 10},
# 定义 LLAMA_GRETYPE_ALT 类型的备选规则，值为0
{LLAMA_GRETYPE_ALT, 0},
# 定义 LLAMA_GRETYPE_END 类型的结束标记，值为0
{LLAMA_GRETYPE_END, 0},
# 定义 LLAMA_GRETYPE_CHAR 类型的字符，值为48
{LLAMA_GRETYPE_CHAR, 48},
# 定义 LLAMA_GRETYPE_CHAR_RNG_UPPER 类型的字符范围上限，值为57
{LLAMA_GRETYPE_CHAR_RNG_UPPER, 57},
# 定义 LLAMA_GRETYPE_RULE_REF 类型的规则引用，值为11
{LLAMA_GRETYPE_RULE_REF, 11},
# 定义 LLAMA_GRETYPE_ALT 类型的备选规则，值为0
{LLAMA_GRETYPE_ALT, 0},
# 定义 LLAMA_GRETYPE_CHAR 类型的字符，值为48
{LLAMA_GRETYPE_CHAR, 48},
# 定义 LLAMA_GRETYPE_CHAR_RNG_UPPER 类型的字符范围上限，值为57
{LLAMA_GRETYPE_CHAR_RNG_UPPER, 57},
# 定义 LLAMA_GRETYPE_END 类型的结束标记，值为0
{LLAMA_GRETYPE_END, 0},
# 定义 LLAMA_GRETYPE_CHAR 类型的字符，值为32
{LLAMA_GRETYPE_CHAR, 32},
# 定义 LLAMA_GRETYPE_CHAR_ALT 类型的备用字符，值为9
{LLAMA_GRETYPE_CHAR_ALT, 9},
# 定义 LLAMA_GRETYPE_CHAR_ALT 类型的备用字符，值为10
{LLAMA_GRETYPE_CHAR_ALT, 10},
// 定义一个包含 LLAMA_GRETYPE_RULE_REF 和 12 的数组
{LLAMA_GRETYPE_RULE_REF, 12},
// 定义一个包含 LLAMA_GRETYPE_ALT 和 0 的数组
{LLAMA_GRETYPE_ALT, 0},
// 定义一个包含 LLAMA_GRETYPE_END 和 0 的数组
{LLAMA_GRETYPE_END, 0},
};

// 初始化 index 为 0
index = 0;
// 遍历 parsed_grammar.rules 中的每个 rule
for (auto rule : parsed_grammar.rules)
{
    // 比较 rule 和期望的 rule
    for (uint32_t i = 0; i < rule.size(); i++)
    {
        // 获取 rule 中的元素
        llama_grammar_element element = rule[i];
        // 获取期望的 rule 中的元素
        llama_grammar_element expected_element = expected_rules[index];

        // 在断言之前打印错误消息
        if (expected_element.type != element.type || expected_element.value != element.value)
        {
            fprintf(stderr, "index: %d\n", index);
            fprintf(stderr, "expected_element: %d, %d\n", expected_element.type, expected_element.value);
            fprintf(stderr, "actual_element: %d, %d\n", element.type, element.value);
    // 如果预期元素与实际元素不相等，则输出错误信息到标准错误流
    fprintf(stderr, "expected_element != actual_element\n");
    
    // 断言预期元素的类型和值与实际元素的类型和值相等
    assert(expected_element.type == element.type && expected_element.value == element.value);
    
    // 索引自增1
    index++;
    
    // 返回0表示执行成功
    return 0;
}
```