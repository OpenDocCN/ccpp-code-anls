# `PowerInfer\common\grammar-parser.cpp`

```
namespace grammar_parser {
    // NOTE: assumes valid utf8 (but checks for overrun)
    // 从给定的 UTF-8 编码的字符串中解码出一个 Unicode 字符
    // 从 llama.cpp 复制而来
    static std::pair<uint32_t, const char *> decode_utf8(const char * src) {
        // UTF-8 编码长度查找表
        static const int lookup[] = { 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 2, 2, 3, 4 };
        // 获取第一个字节
        uint8_t  first_byte = static_cast<uint8_t>(*src);
        // 获取高位信息
        uint8_t  highbits   = first_byte >> 4;
        // 获取编码长度
        int      len        = lookup[highbits];
        // 获取掩码
        uint8_t  mask       = (1 << (8 - len)) - 1;
        // 获取 Unicode 字符值
        uint32_t value      = first_byte & mask;
        // 可能越界的结束位置
        const char * end    = src + len; // may overrun!
        // 当前位置
        const char * pos    = src + 1;
        // 循环解析后续字节
        for ( ; pos < end && *pos; pos++) {
            value = (value << 6) + (static_cast<uint8_t>(*pos) & 0x3F);
        }
        return std::make_pair(value, pos);
    }

    // 获取符号的 ID
    static uint32_t get_symbol_id(parse_state & state, const char * src, size_t len) {
        // 获取下一个符号的 ID
        uint32_t next_id = static_cast<uint32_t>(state.symbol_ids.size());
        // 将符号名和 ID 插入符号 ID 映射表中
        auto result = state.symbol_ids.insert(std::make_pair(std::string(src, len), next_id));
        return result.first->second;
    }

    // 生成符号的 ID
    static uint32_t generate_symbol_id(parse_state & state, const std::string & base_name) {
        // 获取下一个符号的 ID
        uint32_t next_id = static_cast<uint32_t>(state.symbol_ids.size());
        // 将符号名和 ID 插入符号 ID 映射表中
        state.symbol_ids[base_name + '_' + std::to_string(next_id)] = next_id;
        return next_id;
    }

    // 添加规则
    static void add_rule(
            parse_state & state,
            uint32_t      rule_id,
            const std::vector<llama_grammar_element> & rule) {
        // 如果规则列表长度不足，则扩展规则列表
        if (state.rules.size() <= rule_id) {
            state.rules.resize(rule_id + 1);
        }
        // 将规则添加到规则列表中
        state.rules[rule_id] = rule;
    }

    // 判断字符是否为单词字符
    static bool is_word_char(char c) {
        return ('a' <= c && c <= 'z') || ('A' <= c && c <= 'Z') || c == '-' || ('0' <= c && c <= '9');
    }
}
    // 解析十六进制字符串，返回解析后的数值和指向解析结束位置的指针
    static std::pair<uint32_t, const char *> parse_hex(const char * src, int size) {
        const char * pos   = src;  // 初始化指向字符串起始位置的指针
        const char * end   = src + size;  // 计算字符串结束位置的指针
        uint32_t     value = 0;  // 初始化解析后的数值
        for ( ; pos < end && *pos; pos++) {  // 遍历字符串
            value <<= 4;  // 左移4位
            char c = *pos;  // 获取当前字符
            if ('a' <= c && c <= 'f') {  // 如果是小写字母a-f
                value += c - 'a' + 10;  // 计算对应的数值并加到结果中
            } else if ('A' <= c && c <= 'F') {  // 如果是大写字母A-F
                value += c - 'A' + 10;  // 计算对应的数值并加到结果中
            } else if ('0' <= c && c <= '9') {  // 如果是数字0-9
                value += c - '0';  // 计算对应的数值并加到结果中
            } else {  // 其他情况
                break;  // 结束循环
            }
        }
        if (pos != end) {  // 如果解析未到达字符串末尾
            throw std::runtime_error("expecting " + std::to_string(size) + " hex chars at " + src);  // 抛出异常
        }
        return std::make_pair(value, pos);  // 返回解析后的数值和指向解析结束位置的指针
    }

    // 解析空格，返回指向非空格字符的指针，允许换行符
    static const char * parse_space(const char * src, bool newline_ok) {
        const char * pos = src;  // 初始化指向字符串起始位置的指针
        while (*pos == ' ' || *pos == '\t' || *pos == '#' ||  // 循环直到遇到非空格、制表符或注释符号
                (newline_ok && (*pos == '\r' || *pos == '\n'))) {
            if (*pos == '#') {  // 如果是注释符号
                while (*pos && *pos != '\r' && *pos != '\n') {  // 跳过注释内容
                    pos++;
                }
            } else {  // 其他情况
                pos++;  // 指针后移
            }
        }
        return pos;  // 返回指向非空格字符的指针
    }

    // 解析名称，返回指向名称结束位置的指针
    static const char * parse_name(const char * src) {
        const char * pos = src;  // 初始化指向字符串起始位置的指针
        while (is_word_char(*pos)) {  // 循环直到遇到非名称字符
            pos++;  // 指针后移
        }
        if (pos == src) {  // 如果没有解析到名称
            throw std::runtime_error(std::string("expecting name at ") + src);  // 抛出异常
        }
        return pos;  // 返回指向名称结束位置的指针
    }
    // 解析转义字符
    static std::pair<uint32_t, const char *> parse_char(const char * src) {
        // 如果字符以反斜杠开头
        if (*src == '\\') {
            // 根据转义字符类型进行不同的处理
            switch (src[1]) {
                case 'x': return parse_hex(src + 2, 2);  // 解析十六进制字符
                case 'u': return parse_hex(src + 2, 4);  // 解析 Unicode 字符
                case 'U': return parse_hex(src + 2, 8);  // 解析 Unicode 字符
                case 't': return std::make_pair('\t', src + 2);  // 返回制表符
                case 'r': return std::make_pair('\r', src + 2);  // 返回回车符
                case 'n': return std::make_pair('\n', src + 2);  // 返回换行符
                case '\\':
                case '"':
                case '[':
                case ']':
                    return std::make_pair(src[1], src + 2);  // 返回反斜杠、双引号、左方括号或右方括号
                default:
                    throw std::runtime_error(std::string("unknown escape at ") + src);  // 抛出未知转义字符异常
            }
        } else if (*src) {
            return decode_utf8(src);  // 解码 UTF-8 字符
        }
        throw std::runtime_error("unexpected end of input");  // 抛出意外的输入结束异常
    }

    // 解析备选规则
    const char * parse_alternates(
            parse_state       & state,
            const char        * src,
            const std::string & rule_name,
            uint32_t            rule_id,
            bool                is_nested);

    }

    // 解析备选规则
    const char * parse_alternates(
            parse_state       & state,
            const char        * src,
            const std::string & rule_name,
            uint32_t            rule_id,
            bool                is_nested) {
        std::vector<llama_grammar_element> rule;  // 创建规则向量
        const char * pos = parse_sequence(state, src, rule_name, rule, is_nested);  // 解析序列
        // 循环解析备选规则
        while (*pos == '|') {
            rule.push_back({LLAMA_GRETYPE_ALT, 0});  // 添加备选规则类型
            pos = parse_space(pos + 1, true);  // 解析空格
            pos = parse_sequence(state, pos, rule_name, rule, is_nested);  // 解析序列
        }
        rule.push_back({LLAMA_GRETYPE_END, 0});  // 添加规则结束标记
        add_rule(state, rule_id, rule);  // 添加规则到状态中
        return pos;  // 返回解析位置
    }
    // 解析规则，更新解析状态，并返回下一个位置的指针
    static const char * parse_rule(parse_state & state, const char * src) {
        // 解析规则名称的结束位置
        const char * name_end = parse_name(src);
        // 解析规则名称后的空格位置
        const char * pos      = parse_space(name_end, false);
        // 计算规则名称的长度
        size_t       name_len = name_end - src;
        // 获取规则名称对应的符号 ID
        uint32_t     rule_id  = get_symbol_id(state, src, name_len);
        // 将规则名称转换为字符串
        const std::string name(src, name_len);

        // 检查规则定义符号 "::="
        if (!(pos[0] == ':' && pos[1] == ':' && pos[2] == '=')) {
            throw std::runtime_error(std::string("expecting ::= at ") + pos);
        }
        // 跳过规则定义符号后的空格
        pos = parse_space(pos + 3, true);

        // 解析规则的备选项，并更新解析状态
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

    // 解析输入的语法规则，并返回解析状态
    parse_state parse(const char * src) {
        try {
            // 初始化解析状态
            parse_state state;
            // 跳过输入的空格，并循环解析规则
            const char * pos = parse_space(src, true);
            while (*pos) {
                pos = parse_rule(state, pos);
            }
            // 返回解析状态
            return state;
        } catch (const std::exception & err) {
            // 捕获异常并打印错误信息
            fprintf(stderr, "%s: error parsing grammar: %s\n", __func__, err.what());
            // 返回空的解析状态
            return parse_state();
        }
    }

    // 打印语法字符到文件
    static void print_grammar_char(FILE * file, uint32_t c) {
        // 如果是可打印字符，则直接打印
        if (0x20 <= c && c <= 0x7f) {
            fprintf(file, "%c", static_cast<char>(c));
        } else {
            // 否则打印 Unicode 编码
            // 这里简化处理，直接输出 Unicode 编码
            fprintf(file, "<U+%04X>", c);
        }
    }
    // 检查给定的 llama_grammar_element 是否是字符元素，返回布尔值
    static bool is_char_element(llama_grammar_element elem) {
        switch (elem.type) {
            case LLAMA_GRETYPE_CHAR:           return true;
            case LLAMA_GRETYPE_CHAR_NOT:       return true;
            case LLAMA_GRETYPE_CHAR_ALT:       return true;
            case LLAMA_GRETYPE_CHAR_RNG_UPPER: return true;
            default:                           return false;
        }
    }
    
    // 打印规则的二进制表示到文件中
    static void print_rule_binary(FILE * file, const std::vector<llama_grammar_element> & rule) {
        for (auto elem : rule) {
            switch (elem.type) {
                case LLAMA_GRETYPE_END:            fprintf(file, "END");            break;
                case LLAMA_GRETYPE_ALT:            fprintf(file, "ALT");            break;
                case LLAMA_GRETYPE_RULE_REF:       fprintf(file, "RULE_REF");       break;
                case LLAMA_GRETYPE_CHAR:           fprintf(file, "CHAR");           break;
                case LLAMA_GRETYPE_CHAR_NOT:       fprintf(file, "CHAR_NOT");       break;
                case LLAMA_GRETYPE_CHAR_RNG_UPPER: fprintf(file, "CHAR_RNG_UPPER"); break;
                case LLAMA_GRETYPE_CHAR_ALT:       fprintf(file, "CHAR_ALT");       break;
            }
            switch (elem.type) {
                case LLAMA_GRETYPE_END:
                case LLAMA_GRETYPE_ALT:
                case LLAMA_GRETYPE_RULE_REF:
                    fprintf(file, "(%u) ", elem.value);
                    break;
                case LLAMA_GRETYPE_CHAR:
                case LLAMA_GRETYPE_CHAR_NOT:
                case LLAMA_GRETYPE_CHAR_RNG_UPPER:
                case LLAMA_GRETYPE_CHAR_ALT:
                    fprintf(file, "(\"");
                    print_grammar_char(file, elem.value);
                    fprintf(file, "\") ");
                    break;
            }
        }
        fprintf(file, "\n");
    }
    // 打印语法信息到文件
    void print_grammar(FILE * file, const parse_state & state) {
        try {
            // 创建符号 ID 到符号名的映射
            std::map<uint32_t, std::string> symbol_id_names;
            // 遍历状态中的符号 ID，将其与符号名对应起来
            for (const auto & kv : state.symbol_ids) {
                symbol_id_names[kv.second] = kv.first;
            }
            // 遍历规则集合
            for (size_t i = 0, end = state.rules.size(); i < end; i++) {
                // 调用打印规则的函数，将规则信息输出到文件
                print_rule(file, uint32_t(i), state.rules[i], symbol_id_names);
            }
        } catch (const std::exception & err) {
            // 捕获异常，打印错误信息到标准错误流
            fprintf(stderr, "\n%s: error printing grammar: %s\n", __func__, err.what());
        }
    }

    // 返回 C 风格的规则集合
    std::vector<const llama_grammar_element *> parse_state::c_rules() {
        // 创建返回的规则指针集合
        std::vector<const llama_grammar_element *> ret;
        // 预留足够的空间
        ret.reserve(rules.size());
        // 遍历规则集合，将规则指针添加到返回集合中
        for (const auto & rule : rules) {
            ret.push_back(rule.data());
        }
        // 返回规则指针集合
        return ret;
    }
# 闭合前面的函数定义
```