# `PowerInfer\common\grammar-parser.cpp`

```
// 包含头文件 grammar-parser.h
#include "grammar-parser.h"
// 包含必要的标准库头文件
#include <cstdint>
#include <cwchar>
#include <string>
#include <utility>
#include <stdexcept>
#include <exception>

// 命名空间 grammar_parser
namespace grammar_parser {
    // 注意：假设输入是有效的 utf8 编码（但检查是否越界）
    // 从 llama.cpp 复制而来
    // 解码 utf8 编码的字符，返回解码后的 unicode 码点和下一个字符的指针
    static std::pair<uint32_t, const char *> decode_utf8(const char * src) {
        // utf8 编码长度查找表
        static const int lookup[] = { 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 2, 2, 3, 4 };
        // 获取第一个字节
        uint8_t  first_byte = static_cast<uint8_t>(*src);
        // 获取高位信息
        uint8_t  highbits   = first_byte >> 4;
        // 获取 utf8 编码的长度
        int      len        = lookup[highbits];
        // 用于获取后续字节的掩码
        uint8_t  mask       = (1 << (8 - len)) - 1;
        // 存储解码后的 unicode 码点
        uint32_t value      = first_byte & mask;
        // 可能越界的结束位置
        const char * end    = src + len; // may overrun!
        // 下一个字符的指针
        const char * pos    = src + 1;
    // 从当前位置开始，循环直到到达字符串末尾或者遇到空字符
    for ( ; pos < end && *pos; pos++) {
        // 将当前字符转换为 uint8_t 类型，并且取其低 6 位，然后左移 6 位后与 value 相加
        value = (value << 6) + (static_cast<uint8_t>(*pos) & 0x3F);
    }
    // 返回 value 和当前位置的 pair 对象
    return std::make_pair(value, pos);
}

// 获取符号的 ID
static uint32_t get_symbol_id(parse_state & state, const char * src, size_t len) {
    // 获取下一个符号的 ID
    uint32_t next_id = static_cast<uint32_t>(state.symbol_ids.size());
    // 将新的符号和其 ID 插入符号 ID 的 map 中
    auto result = state.symbol_ids.insert(std::make_pair(std::string(src, len), next_id));
    // 返回插入结果中的符号 ID
    return result.first->second;
}

// 生成符号的 ID
static uint32_t generate_symbol_id(parse_state & state, const std::string & base_name) {
    // 获取下一个符号的 ID
    uint32_t next_id = static_cast<uint32_t>(state.symbol_ids.size());
    // 将新的符号和其 ID 插入符号 ID 的 map 中
    state.symbol_ids[base_name + '_' + std::to_string(next_id)] = next_id;
    // 返回新符号的 ID
    return next_id;
}

// 添加规则
static void add_rule(
        parse_state & state,
// 将规则ID和规则元素添加到状态对象中
void add_rule(uint32_t rule_id, const std::vector<llama_grammar_element> & rule) {
    // 如果状态对象中规则的数量小于等于规则ID，则扩展规则数组
    if (state.rules.size() <= rule_id) {
        state.rules.resize(rule_id + 1);
    }
    // 将规则添加到状态对象的规则数组中
    state.rules[rule_id] = rule;
}

// 检查字符是否为单词字符
static bool is_word_char(char c) {
    return ('a' <= c && c <= 'z') || ('A' <= c && c <= 'Z') || c == '-' || ('0' <= c && c <= '9');
}

// 解析十六进制数并返回其值和指向源字符串的指针
static std::pair<uint32_t, const char *> parse_hex(const char * src, int size) {
    const char * pos   = src;
    const char * end   = src + size;
    uint32_t     value = 0;
    // 遍历源字符串，将十六进制字符转换为对应的数值
    for ( ; pos < end && *pos; pos++) {
        value <<= 4;
        char c = *pos;
        if ('a' <= c && c <= 'f') {
            // 如果字符为小写字母a-f，则将其转换为对应的数值
        // 如果字符是小写字母，则将其转换为对应的十六进制值并加上10
        value += c - 'a' + 10;
        // 如果字符是大写字母，则将其转换为对应的十六进制值并加上10
    } else if ('A' <= c && c <= 'F') {
        value += c - 'A' + 10;
        // 如果字符是数字，则将其转换为对应的十进制值
    } else if ('0' <= c && c <= '9') {
        value += c - '0';
        // 如果字符不是合法的十六进制字符，则跳出循环
    } else {
        break;
    }
}
// 如果循环结束后仍有剩余字符，则抛出异常
if (pos != end) {
    throw std::runtime_error("expecting " + std::to_string(size) + " hex chars at " + src);
}
// 返回解析出的十六进制值和结束位置
return std::make_pair(value, pos);
}

// 解析空白字符，newline_ok 表示是否允许换行符
static const char * parse_space(const char * src, bool newline_ok) {
    const char * pos = src;
    // 循环跳过空格、制表符、#注释以及换行符（如果允许）
    while (*pos == ' ' || *pos == '\t' || *pos == '#' ||
            (newline_ok && (*pos == '\r' || *pos == '\n'))) {
        // 如果遇到 # 注释，则跳过整行
        if (*pos == '#') {
    // 当指针位置不为空且不是回车或换行符时，移动指针位置
    while (*pos && *pos != '\r' && *pos != '\n') {
        pos++;
    }
    // 如果指针位置为空，则继续移动指针位置
    } else {
        pos++;
    }
    // 返回指针位置
    return pos;
}

// 解析名称函数，输入参数为源字符串
static const char * parse_name(const char * src) {
    // 初始化指针位置
    const char * pos = src;
    // 当指针位置为单词字符时，移动指针位置
    while (is_word_char(*pos)) {
        pos++;
    }
    // 如果指针位置等于源字符串位置，则抛出运行时错误
    if (pos == src) {
        throw std::runtime_error(std::string("expecting name at ") + src);
    }
    // 返回指针位置
    return pos;
}
// 解析转义字符
static std::pair<uint32_t, const char *> parse_char(const char * src) {
    // 如果是反斜杠开头的转义字符
    if (*src == '\\') {
        // 根据转义字符的第二个字符进行不同的处理
        switch (src[1]) {
            // 如果是\x开头的转义字符，解析16进制数
            case 'x': return parse_hex(src + 2, 2);
            // 如果是\u开头的转义字符，解析16进制数
            case 'u': return parse_hex(src + 2, 4);
            // 如果是\U开头的转义字符，解析16进制数
            case 'U': return parse_hex(src + 2, 8);
            // 如果是\t，返回制表符
            case 't': return std::make_pair('\t', src + 2);
            // 如果是\r，返回回车符
            case 'r': return std::make_pair('\r', src + 2);
            // 如果是\n，返回换行符
            case 'n': return std::make_pair('\n', src + 2);
            // 如果是\\，\"，[，]，返回对应的字符
            case '\\':
            case '"':
            case '[':
            case ']':
                return std::make_pair(src[1], src + 2);
            // 其他情况抛出异常
            default:
                throw std::runtime_error(std::string("unknown escape at ") + src);
        }
    } else if (*src) {
        // 解码UTF-8字符
        return decode_utf8(src);
    }
}
    }
    // 如果解析到输入的意外结束，抛出运行时错误
    throw std::runtime_error("unexpected end of input");
}

// 解析替代项
const char * parse_alternates(
        parse_state       & state,
        const char        * src,
        const std::string & rule_name,
        uint32_t            rule_id,
        bool                is_nested);

// 解析序列
static const char * parse_sequence(
        parse_state                        & state,
        const char                         * src,
        const std::string                  & rule_name,
        std::vector<llama_grammar_element> & out_elements,
        bool                                 is_nested) {
    // 记录当前序列的起始位置
    size_t last_sym_start = out_elements.size();
    // 获取当前位置的指针
    const char * pos = src;
    // 循环解析序列中的元素
    while (*pos) {
            // 如果当前位置是双引号，则表示是一个字面字符串
            if (*pos == '"') { 
                pos++; // 移动到下一个位置
                last_sym_start = out_elements.size(); // 记录当前位置
                // 循环直到遇到双引号，解析字符并添加到输出元素中
                while (*pos != '"') {
                    auto char_pair = parse_char(pos); // 解析字符
                         pos       = char_pair.second; // 更新位置
                    out_elements.push_back({LLAMA_GRETYPE_CHAR, char_pair.first}); // 将解析的字符添加到输出元素中
                }
                pos = parse_space(pos + 1, is_nested); // 解析空格
            } else if (*pos == '[') { // 如果当前位置是左方括号，则表示是字符范围
                pos++; // 移动到下一个位置
                enum llama_gretype start_type = LLAMA_GRETYPE_CHAR; // 设置起始类型为字符
                if (*pos == '^') { // 如果下一个位置是脱字符，则表示是排除字符范围
                    pos++; // 移动到下一个位置
                    start_type = LLAMA_GRETYPE_CHAR_NOT; // 更新起始类型
                }
                last_sym_start = out_elements.size(); // 记录当前位置
                // 循环直到遇到右方括号，解析字符并添加到输出元素中
                while (*pos != ']') {
                    auto char_pair = parse_char(pos); // 解析字符
                         pos       = char_pair.second; // 更新位置
// 定义枚举类型 llama_gretype，并根据条件赋值给 type
enum llama_gretype type = last_sym_start < out_elements.size()
    ? LLAMA_GRETYPE_CHAR_ALT
    : start_type;

// 将 type 和 char_pair.first 添加到 out_elements 中
out_elements.push_back({type, char_pair.first});

// 如果当前位置是 '-' 且下一个位置不是 ']'，则解析字符范围并添加到 out_elements 中
if (pos[0] == '-' && pos[1] != ']') {
    auto endchar_pair = parse_char(pos + 1);
    pos = endchar_pair.second;
    out_elements.push_back({LLAMA_GRETYPE_CHAR_RNG_UPPER, endchar_pair.first});
}

// 解析空格并更新 pos
pos = parse_space(pos + 1, is_nested);
} else if (is_word_char(*pos)) { // 如果当前位置是单词字符，则表示是规则引用
    // 解析规则引用的名称，并获取其对应的规则 ID
    const char * name_end    = parse_name(pos);
    uint32_t     ref_rule_id = get_symbol_id(state, pos, name_end - pos);
    pos = parse_space(name_end, is_nested);
    last_sym_start = out_elements.size();
    // 将规则引用的类型和 ID 添加到 out_elements 中
    out_elements.push_back({LLAMA_GRETYPE_RULE_REF, ref_rule_id});
} else if (*pos == '(') { // 如果当前位置是 '('，表示分组
    // 解析嵌套的备选项并合成规则
                // 从当前位置开始解析空格，并返回下一个非空格字符的位置
                pos = parse_space(pos + 1, true);
                // 生成符号 ID，用于表示当前规则的符号
                uint32_t sub_rule_id = generate_symbol_id(state, rule_name);
                // 解析当前位置的备选规则
                pos = parse_alternates(state, pos, rule_name, sub_rule_id, true);
                // 记录上一个符号的起始位置
                last_sym_start = out_elements.size();
                // 输出对合成规则的引用
                out_elements.push_back({LLAMA_GRETYPE_RULE_REF, sub_rule_id});
                // 如果当前位置不是 ')'，则抛出运行时错误
                if (*pos != ')') {
                    throw std::runtime_error(std::string("expecting ')' at ") + pos);
                }
                // 从当前位置开始解析空格，并返回下一个非空格字符的位置
                pos = parse_space(pos + 1, is_nested);
            } else if (*pos == '*' || *pos == '+' || *pos == '?') { // 重复操作符
                // 如果上一个符号的起始位置等于当前输出元素的大小，则抛出运行时错误
                if (last_sym_start == out_elements.size()) {
                    throw std::runtime_error(std::string("expecting preceeding item to */+/? at ") + pos);
                }

                // 根据重写规则对前一个符号（从 last_sym_start 到末尾）应用转换：
                // S* --> S' ::= S S' |
                // S+ --> S' ::= S S' | S
                // S? --> S' ::= S |
// 生成符号 ID
uint32_t sub_rule_id = generate_symbol_id(state, rule_name);
// 创建子规则的 vector
std::vector<llama_grammar_element> sub_rule;
// 将前面的符号添加到生成的规则中
sub_rule.insert(
    sub_rule.end(), out_elements.begin() + last_sym_start, out_elements.end());
// 如果当前字符为 '*' 或 '+'，则使生成的规则递归
if (*pos == '*' || *pos == '+') {
    sub_rule.push_back({LLAMA_GRETYPE_RULE_REF, sub_rule_id});
}
// 标记交替定义的开始
sub_rule.push_back({LLAMA_GRETYPE_ALT, 0});
// 如果当前字符为 '+'，则将前面的符号作为交替规则的一部分添加（否则为空）
if (*pos == '+') {
    sub_rule.insert(
        sub_rule.end(), out_elements.begin() + last_sym_start, out_elements.end());
}
// 标记规则结束
sub_rule.push_back({LLAMA_GRETYPE_END, 0});
// 添加规则到状态中
add_rule(state, sub_rule_id, sub_rule);

// 在原始规则中，用生成的规则替换先前的符号引用
        // 调整 out_elements 的大小为 last_sym_start
        out_elements.resize(last_sym_start);
        // 将 {LLAMA_GRETYPE_RULE_REF, sub_rule_id} 添加到 out_elements 中
        out_elements.push_back({LLAMA_GRETYPE_RULE_REF, sub_rule_id});

        // 调用 parse_space 函数，解析空格并更新 pos 的位置
        pos = parse_space(pos + 1, is_nested);
    } else {
        // 如果不满足条件，跳出循环
        break;
    }
}
// 返回 pos 的位置
return pos;
}

// 解析替代项
const char * parse_alternates(
        parse_state       & state,
        const char        * src,
        const std::string & rule_name,
        uint32_t            rule_id,
        bool                is_nested) {
    // 创建 llama_grammar_element 类型的 rule 向量
    std::vector<llama_grammar_element> rule;
    // 调用 parse_sequence 函数，解析序列并更新 pos 的位置
    const char * pos = parse_sequence(state, src, rule_name, rule, is_nested);
    // 当 pos 指向的字符为 '|' 时，执行循环
    while (*pos == '|') {
// 将LLAMA_GRETYPE_ALT和0添加到rule向量的末尾
rule.push_back({LLAMA_GRETYPE_ALT, 0});
// 从当前位置向后解析空格，并返回新的位置
pos = parse_space(pos + 1, true);
// 解析规则的序列部分
pos = parse_sequence(state, pos, rule_name, rule, is_nested);
// 将LLAMA_GRETYPE_END和0添加到rule向量的末尾
rule.push_back({LLAMA_GRETYPE_END, 0});
// 将规则添加到状态中
add_rule(state, rule_id, rule);
// 返回新的位置
return pos;
}

// 解析规则的函数
static const char * parse_rule(parse_state & state, const char * src) {
    // 解析规则名称的末尾
    const char * name_end = parse_name(src);
    // 解析空格，返回新的位置
    const char * pos      = parse_space(name_end, false);
    // 计算规则名称的长度
    size_t       name_len = name_end - src;
    // 获取规则的符号ID
    uint32_t     rule_id  = get_symbol_id(state, src, name_len);
    // 创建规则名称的字符串
    const std::string name(src, name_len);

    // 检查是否遇到"::="，否则抛出异常
    if (!(pos[0] == ':' && pos[1] == ':' && pos[2] == '=')) {
        throw std::runtime_error(std::string("expecting ::= at ") + pos);
    }
    // 解析空格，返回新的位置
    pos = parse_space(pos + 3, true);
        // 调用 parse_alternates 函数，解析当前状态下的备选项
        pos = parse_alternates(state, pos, name, rule_id, false);

        // 如果当前位置是回车符，则根据下一个字符是换行符还是其他字符来移动位置
        if (*pos == '\r') {
            pos += pos[1] == '\n' ? 2 : 1;
        } 
        // 如果当前位置是换行符，则移动位置到下一个字符
        else if (*pos == '\n') {
            pos++;
        } 
        // 如果当前位置不是回车符也不是换行符，则抛出异常
        else if (*pos) {
            throw std::runtime_error(std::string("expecting newline or end at ") + pos);
        }
        // 调用 parse_space 函数，解析空格
        return parse_space(pos, true);
    }

    // 解析输入字符串
    parse_state parse(const char * src) {
        try {
            // 创建解析状态对象
            parse_state state;
            // 调用 parse_space 函数，解析空格，并返回解析后的位置
            const char * pos = parse_space(src, true);
            // 循环解析规则
            while (*pos) {
                pos = parse_rule(state, pos);
            }
    // 返回当前状态
    return state;
} catch (const std::exception & err) {
    // 捕获异常并打印错误信息
    fprintf(stderr, "%s: error parsing grammar: %s\n", __func__, err.what());
    // 返回解析状态
    return parse_state();
}

// 打印语法字符到文件
static void print_grammar_char(FILE * file, uint32_t c) {
    // 如果是可打印字符，则直接打印
    if (0x20 <= c && c <= 0x7f) {
        fprintf(file, "%c", static_cast<char>(c));
    } else {
        // 否则，打印 Unicode 编码
        fprintf(file, "<U+%04X>", c);
    }
}

// 判断是否为字符元素
static bool is_char_element(llama_grammar_element elem) {
    switch (elem.type) {
        // 如果是字符或非字符类型，则返回 true
        case LLAMA_GRETYPE_CHAR:           return true;
        case LLAMA_GRETYPE_CHAR_NOT:       return true;
// 根据元素类型返回 true 或 false
bool is_valid_element(llama_grammar_element elem) {
    switch (elem.type) {
        case LLAMA_GRETYPE_CHAR_ALT:       return true;  // 如果是字符替代类型，返回 true
        case LLAMA_GRETYPE_CHAR_RNG_UPPER: return true;  // 如果是字符范围上限类型，返回 true
        default:                           return false; // 其他情况返回 false
    }
}

// 将规则以二进制形式打印到文件
static void print_rule_binary(FILE * file, const std::vector<llama_grammar_element> & rule) {
    for (auto elem : rule) {
        switch (elem.type) {
            case LLAMA_GRETYPE_END:            fprintf(file, "END");            break;  // 如果是规则结束类型，打印"END"
            case LLAMA_GRETYPE_ALT:            fprintf(file, "ALT");            break;  // 如果是规则分支类型，打印"ALT"
            case LLAMA_GRETYPE_RULE_REF:       fprintf(file, "RULE_REF");       break;  // 如果是规则引用类型，打印"RULE_REF"
            case LLAMA_GRETYPE_CHAR:           fprintf(file, "CHAR");           break;  // 如果是字符类型，打印"CHAR"
            case LLAMA_GRETYPE_CHAR_NOT:       fprintf(file, "CHAR_NOT");       break;  // 如果是非字符类型，打印"CHAR_NOT"
            case LLAMA_GRETYPE_CHAR_RNG_UPPER: fprintf(file, "CHAR_RNG_UPPER"); break;  // 如果是字符范围上限类型，打印"CHAR_RNG_UPPER"
            case LLAMA_GRETYPE_CHAR_ALT:       fprintf(file, "CHAR_ALT");       break;  // 如果是字符替代类型，打印"CHAR_ALT"
        }
        switch (elem.type) {
            case LLAMA_GRETYPE_END:  // 如果是规则结束类型
            case LLAMA_GRETYPE_ALT:  // 或者是规则分支类型
// 根据元素类型进行不同的处理
switch (elem.type) {
    // 如果是引用规则类型，则在文件中打印规则的值
    case LLAMA_GRETYPE_RULE_REF:
        fprintf(file, "(%u) ", elem.value);
        break;
    // 如果是字符类型或者字符范围类型，则在文件中打印字符值
    case LLAMA_GRETYPE_CHAR:
    case LLAMA_GRETYPE_CHAR_NOT:
    case LLAMA_GRETYPE_CHAR_RNG_UPPER:
    case LLAMA_GRETYPE_CHAR_ALT:
        fprintf(file, "(\"");
        print_grammar_char(file, elem.value);
        fprintf(file, "\") ");
        break;
}
// 在文件中打印换行符
fprintf(file, "\n");
// 打印规则
static void print_rule(
        FILE     * file,
        uint32_t   rule_id,
        const std::vector<llama_grammar_element> & rule,
// 根据规则、符号ID名称和文件指针输出语法规则
void output_rule(FILE *file, const std::vector<llama_grammar_element> & rule, uint32_t rule_id,
                 const std::map<uint32_t, std::string> & symbol_id_names) {
    // 如果规则为空或者最后一个元素不是LLAMA_GRETYPE_END，则抛出异常
    if (rule.empty() || rule.back().type != LLAMA_GRETYPE_END) {
        throw std::runtime_error(
            "malformed rule, does not end with LLAMA_GRETYPE_END: " + std::to_string(rule_id));
    }
    // 输出规则的开始部分
    fprintf(file, "%s ::= ", symbol_id_names.at(rule_id).c_str());
    // 遍历规则中的元素
    for (size_t i = 0, end = rule.size() - 1; i < end; i++) {
        llama_grammar_element elem = rule[i];
        // 根据元素的类型进行不同的处理
        switch (elem.type) {
            // 如果是LLAMA_GRETYPE_END类型，则抛出异常
            case LLAMA_GRETYPE_END:
                throw std::runtime_error(
                    "unexpected end of rule: " + std::to_string(rule_id) + "," +
                    std::to_string(i));
            // 如果是LLAMA_GRETYPE_ALT类型，则输出"| "
            case LLAMA_GRETYPE_ALT:
                fprintf(file, "| ");
                break;
            // 如果是LLAMA_GRETYPE_RULE_REF类型，则输出对应的符号名称
            case LLAMA_GRETYPE_RULE_REF:
                fprintf(file, "%s ", symbol_id_names.at(elem.value).c_str());
                break;
            // 如果是LLAMA_GRETYPE_CHAR类型，则...
// 在文件中打印左方括号，并打印语法元素的值
fprintf(file, "[");
print_grammar_char(file, elem.value);
break;

// 在文件中打印左方括号和脱字符，并打印语法元素的值
case LLAMA_GRETYPE_CHAR_NOT:
    fprintf(file, "[^");
    print_grammar_char(file, elem.value);
    break;

// 如果当前字符元素之前没有字符元素，抛出运行时错误
case LLAMA_GRETYPE_CHAR_RNG_UPPER:
    if (i == 0 || !is_char_element(rule[i - 1])) {
        throw std::runtime_error(
            "LLAMA_GRETYPE_CHAR_RNG_UPPER without preceding char: " +
            std::to_string(rule_id) + "," + std::to_string(i));
    }
    // 在文件中打印连字符和语法元素的值
    fprintf(file, "-");
    print_grammar_char(file, elem.value);
    break;

// 如果当前字符元素之前没有字符元素，抛出运行时错误
case LLAMA_GRETYPE_CHAR_ALT:
    if (i == 0 || !is_char_element(rule[i - 1])) {
        throw std::runtime_error(
            "LLAMA_GRETYPE_CHAR_ALT without preceding char: " +
// 将 rule_id 和 i 转换为字符串，并拼接在一起
std::to_string(rule_id) + "," + std::to_string(i));
// 打印语法元素的字符值
print_grammar_char(file, elem.value);
// 终止 switch 语句
break;
// 判断当前元素是否为字符元素
if (is_char_element(elem)) {
    // 根据 rule[i + 1].type 的不同情况进行处理
    switch (rule[i + 1].type) {
        // 如果 rule[i + 1].type 为 LLAMA_GRETYPE_CHAR_ALT 或 LLAMA_GRETYPE_CHAR_RNG_UPPER，则终止 switch 语句
        case LLAMA_GRETYPE_CHAR_ALT:
        case LLAMA_GRETYPE_CHAR_RNG_UPPER:
            break;
        // 如果 rule[i + 1].type 不是上述两种情况，则在文件中打印 "] "
        default:
            fprintf(file, "] ");
    }
}
// 在文件中打印换行符
fprintf(file, "\n");
// 结束函数 print_grammar
}

// 打印语法
void print_grammar(FILE * file, const parse_state & state) {
    // 尝试执行以下代码块
    try {
// 创建一个从uint32_t到string的映射，用于存储符号ID和对应的名称
std::map<uint32_t, std::string> symbol_id_names;
// 遍历state.symbol_ids，将符号ID和对应的名称存储到symbol_id_names中
for (const auto & kv : state.symbol_ids) {
    symbol_id_names[kv.second] = kv.first;
}
// 遍历state.rules，打印每个规则的内容
for (size_t i = 0, end = state.rules.size(); i < end; i++) {
    // 调用print_rule函数，打印规则的内容
    print_rule(file, uint32_t(i), state.rules[i], symbol_id_names);
}
// 捕获可能发生的异常，并打印错误信息
} catch (const std::exception & err) {
    fprintf(stderr, "\n%s: error printing grammar: %s\n", __func__, err.what());
}

// 返回一个包含llama_grammar_element指针的vector
std::vector<const llama_grammar_element *> parse_state::c_rules() {
    // 创建一个vector，用于存储规则的指针
    std::vector<const llama_grammar_element *> ret;
    // 预留足够的空间
    ret.reserve(rules.size());
    // 遍历rules，将规则的指针存储到ret中
    for (const auto & rule : rules) {
        ret.push_back(rule.data());
    }
这部分代码缺少上下文，无法确定每个语句的作用。
```