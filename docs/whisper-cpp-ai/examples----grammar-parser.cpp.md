# `whisper.cpp\examples\grammar-parser.cpp`

```cpp
namespace grammar_parser {
    // 假设输入的是有效的 UTF-8 编码（但检查是否越界）
    // 从 whisper.cpp 复制而来
    std::pair<uint32_t, const char *> decode_utf8(const char * src) {
        // 定义 UTF-8 编码长度表
        static const int lookup[] = { 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 2, 2, 3, 4 };
        // 获取第一个字节
        uint8_t  first_byte = static_cast<uint8_t>(*src);
        // 获取高位
        uint8_t  highbits   = first_byte >> 4;
        // 获取编码长度
        int      len        = lookup[highbits];
        // 获取掩码
        uint8_t  mask       = (1 << (8 - len)) - 1;
        // 获取第一个字节的值
        uint32_t value      = first_byte & mask;
        // 可能越界的结束位置
        const char * end    = src + len; // may overrun!
        // 当前位置
        const char * pos    = src + 1;
        // 解析 UTF-8 编码
        for ( ; pos < end && *pos; pos++) {
            value = (value << 6) + (static_cast<uint8_t>(*pos) & 0x3F);
        }
        return std::make_pair(value, pos);
    }

    // 获取符号 ID
    uint32_t get_symbol_id(parse_state & state, const char * src, size_t len) {
        // 下一个 ID
        uint32_t next_id = static_cast<uint32_t>(state.symbol_ids.size());
        // 插入符号 ID
        auto result = state.symbol_ids.insert(std::make_pair(std::string(src, len), next_id));
        return result.first->second;
    }

    // 生成符号 ID
    uint32_t generate_symbol_id(parse_state & state, const std::string & base_name) {
        // 下一个 ID
        uint32_t next_id = static_cast<uint32_t>(state.symbol_ids.size());
        // 插入符号 ID
        state.symbol_ids[base_name + '_' + std::to_string(next_id)] = next_id;
        return next_id;
    }

    // 添加规则
    void add_rule(
            parse_state & state,
            uint32_t      rule_id,
            const std::vector<whisper_grammar_element> & rule) {
        // 调整规则容器大小
        if (state.rules.size() <= rule_id) {
            state.rules.resize(rule_id + 1);
        }
        // 添加规则
        state.rules[rule_id] = rule;
    }

    // 判断字符是否为单词字符
    bool is_word_char(char c) {
        return ('a' <= c && c <= 'z') || ('A' <= c && c <= 'Z') || c == '-' || ('0' <= c && c <= '9');
    }
}
    // 解析十六进制字符串，返回一个无符号整数和指向解析结束位置的指针
    std::pair<uint32_t, const char *> parse_hex(const char * src, int size) {
        // 初始化指向字符串起始位置和结束位置的指针，以及数值变量
        const char * pos   = src;
        const char * end   = src + size;
        uint32_t     value = 0;
        // 遍历字符串，解析十六进制字符并计算数值
        for ( ; pos < end && *pos; pos++) {
            value <<= 4;
            char c = *pos;
            if ('a' <= c && c <= 'f') {
                value += c - 'a' + 10;
            } else if ('A' <= c && c <= 'F') {
                value += c - 'A' + 10;
            } else if ('0' <= c && c <= '9') {
                value += c - '0';
            } else {
                break;
            }
        }
        // 如果未解析完整个字符串，抛出异常
        if (pos != end) {
            throw std::runtime_error("expecting " + std::to_string(size) + " hex chars at " + src);
        }
        // 返回解析结果
        return std::make_pair(value, pos);
    }

    // 解析空格，返回指向下一个非空格字符的指针
    const char * parse_space(const char * src, bool newline_ok) {
        // 初始化指向字符串起始位置的指针
        const char * pos = src;
        // 遍历字符串，跳过空格、制表符、注释和可能的换行符
        while (*pos == ' ' || *pos == '\t' || *pos == '#' ||
                (newline_ok && (*pos == '\r' || *pos == '\n'))) {
            // 如果遇到注释，跳过整行
            if (*pos == '#') {
                while (*pos && *pos != '\r' && *pos != '\n') {
                    pos++;
                }
            } else {
                pos++;
            }
        }
        // 返回指向下一个非空格字符的指针
        return pos;
    }

    // 解析名称，返回指向名称结束位置的指针
    const char * parse_name(const char * src) {
        // 初始化指向字符串起始位置的指针
        const char * pos = src;
        // 遍历字符串，直到遇到非字母数字字符为止
        while (is_word_char(*pos)) {
            pos++;
        }
        // 如果未解析出名称，抛出异常
        if (pos == src) {
            throw std::runtime_error(std::string("expecting name at ") + src);
        }
        // 返回指向名称结束位置的指针
        return pos;
    }
    // 解析转义字符，返回转义后的字符和下一个位置的指针
    std::pair<uint32_t, const char *> parse_char(const char * src) {
        // 如果当前字符是反斜杠
        if (*src == '\\') {
            // 根据下一个字符的不同进行不同的处理
            switch (src[1]) {
                case 'x': return parse_hex(src + 2, 2); // 解析十六进制字符
                case 'u': return parse_hex(src + 2, 4); // 解析 Unicode 字符
                case 'U': return parse_hex(src + 2, 8); // 解析 Unicode 字符
                case 't': return std::make_pair('\t', src + 2); // 返回制表符
                case 'r': return std::make_pair('\r', src + 2); // 返回回车符
                case 'n': return std::make_pair('\n', src + 2); // 返回换行符
                case '\\':
                case '"':
                case '[':
                case ']':
                    return std::make_pair(src[1], src + 2); // 返回特殊字符
                default:
                    throw std::runtime_error(std::string("unknown escape at ") + src); // 抛出未知转义错误
            }
        } else if (*src) {
            return decode_utf8(src); // 解码 UTF-8 字符
        }
        throw std::runtime_error("unexpected end of input"); // 抛出意外的输入结束错误
    }

    // 解析备选项，返回解析后的位置指针
    const char * parse_alternates(
            parse_state       & state,
            const char        * src,
            const std::string & rule_name,
            uint32_t            rule_id,
            bool                is_nested);

    }

    // 解析备选项，返回解析后的位置指针
    const char * parse_alternates(
            parse_state       & state,
            const char        * src,
            const std::string & rule_name,
            uint32_t            rule_id,
            bool                is_nested) {
        // 创建规则元素向量
        std::vector<whisper_grammar_element> rule;
        // 解析序列，并更新位置指针
        const char * pos = parse_sequence(state, src, rule_name, rule, is_nested);
        // 循环解析备选项
        while (*pos == '|') {
            rule.push_back({WHISPER_GRETYPE_ALT, 0}); // 添加备选项标记
            pos = parse_space(pos + 1, true); // 解析空格
            pos = parse_sequence(state, pos, rule_name, rule, is_nested); // 解析序列
        }
        rule.push_back({WHISPER_GRETYPE_END, 0}); // 添加结束标记
        add_rule(state, rule_id, rule); // 添加规则
        return pos; // 返回更新后的位置指针
    }
    // 解析规则，更新解析状态，并返回下一个位置的指针
    const char * parse_rule(parse_state & state, const char * src) {
        // 解析规则名称的结束位置
        const char * name_end = parse_name(src);
        // 解析空格，返回下一个位置的指针
        const char * pos      = parse_space(name_end, false);
        // 计算规则名称的长度
        size_t       name_len = name_end - src;
        // 获取规则名称对应的符号 ID
        uint32_t     rule_id  = get_symbol_id(state, src, name_len);
        // 将规则名称转换为字符串
        const std::string name(src, name_len);

        // 检查规则定义符号是否为 "::="
        if (!(pos[0] == ':' && pos[1] == ':' && pos[2] == '=')) {
            throw std::runtime_error(std::string("expecting ::= at ") + pos);
        }
        // 跳过规则定义符号后的空格
        pos = parse_space(pos + 3, true);

        // 解析规则的备选项
        pos = parse_alternates(state, pos, name, rule_id, false);

        // 处理换行符
        if (*pos == '\r') {
            pos += pos[1] == '\n' ? 2 : 1;
        } else if (*pos == '\n') {
            pos++;
        } else if (*pos) {
            throw std::runtime_error(std::string("expecting newline or end at ") + pos);
        }
        // 返回下一个位置的指针
        return parse_space(pos, true);
    }

    // 解析整个语法
    parse_state parse(const char * src) {
        try {
            // 初始化解析状态
            parse_state state;
            // 解析空格，返回下一个位置的指针
            const char * pos = parse_space(src, true);
            // 循环解析规则
            while (*pos) {
                pos = parse_rule(state, pos);
            }
            // 返回解析状态
            return state;
        } catch (const std::exception & err) {
            // 捕获异常并输出错误信息
            fprintf(stderr, "%s: error parsing grammar: %s\n", __func__, err.what());
            // 返回空的解析状态
            return parse_state();
        }
    }

    // 打印字符到文件
    void print_grammar_char(FILE * file, uint32_t c) {
        // 判断字符是否可打印
        if (0x20 <= c && c <= 0x7f) {
            // 打印可打印字符
            fprintf(file, "%c", static_cast<char>(c));
        } else {
            // 无法打印的字符，输出其 Unicode 编码
            fprintf(file, "<U+%04X>", c);
        }
    }
    // 判断给定的语法元素是否为字符类型的元素
    bool is_char_element(whisper_grammar_element elem) {
        switch (elem.type) {
            case WHISPER_GRETYPE_CHAR:           return true;
            case WHISPER_GRETYPE_CHAR_NOT:       return true;
            case WHISPER_GRETYPE_CHAR_ALT:       return true;
            case WHISPER_GRETYPE_CHAR_RNG_UPPER: return true;
            default:                           return false;
        }
    }

    // 打印规则的二进制表示到文件中
    void print_rule_binary(FILE * file, const std::vector<whisper_grammar_element> & rule) {
        for (auto elem : rule) {
            switch (elem.type) {
                case WHISPER_GRETYPE_END:            fprintf(file, "END");            break;
                case WHISPER_GRETYPE_ALT:            fprintf(file, "ALT");            break;
                case WHISPER_GRETYPE_RULE_REF:       fprintf(file, "RULE_REF");       break;
                case WHISPER_GRETYPE_CHAR:           fprintf(file, "CHAR");           break;
                case WHISPER_GRETYPE_CHAR_NOT:       fprintf(file, "CHAR_NOT");       break;
                case WHISPER_GRETYPE_CHAR_RNG_UPPER: fprintf(file, "CHAR_RNG_UPPER"); break;
                case WHISPER_GRETYPE_CHAR_ALT:       fprintf(file, "CHAR_ALT");       break;
            }
            switch (elem.type) {
                case WHISPER_GRETYPE_END:
                case WHISPER_GRETYPE_ALT:
                case WHISPER_GRETYPE_RULE_REF:
                    fprintf(file, "(%u) ", elem.value);
                    break;
                case WHISPER_GRETYPE_CHAR:
                case WHISPER_GRETYPE_CHAR_NOT:
                case WHISPER_GRETYPE_CHAR_RNG_UPPER:
                case WHISPER_GRETYPE_CHAR_ALT:
                    fprintf(file, "(\"");
                    // 打印字符元素的值
                    print_grammar_char(file, elem.value);
                    fprintf(file, "\") ");
                    break;
            }
        }
        fprintf(file, "\n");
    }
    // 打印语法规则到文件中
    void print_grammar(FILE * file, const parse_state & state) {
        try {
            // 创建符号 ID 到名称的映射
            std::map<uint32_t, std::string> symbol_id_names;
            for (auto kv : state.symbol_ids) {
                symbol_id_names[kv.second] = kv.first;
            }
            // 遍历规则列表，打印每个规则
            for (size_t i = 0, end = state.rules.size(); i < end; i++) {
                // 调用打印规则的函数
                print_rule(file, uint32_t(i), state.rules[i], symbol_id_names);
            }
        } catch (const std::exception & err) {
            // 捕获异常并打印错误信息
            fprintf(stderr, "\n%s: error printing grammar: %s\n", __func__, err.what());
        }
    }

    // 返回规则列表的指针数组
    std::vector<const whisper_grammar_element *> parse_state::c_rules() const{
        std::vector<const whisper_grammar_element *> ret;
        // 遍历规则列表，将每个规则的指针添加到返回数组中
        for (const auto & rule : rules) {
            ret.push_back(rule.data());
        }
        return ret;
    }
# 闭合之前的代码块
```