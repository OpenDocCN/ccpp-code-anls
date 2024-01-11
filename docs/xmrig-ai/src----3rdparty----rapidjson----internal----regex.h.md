# `xmrig\src\3rdparty\rapidjson\internal\regex.h`

```
#ifndef RAPIDJSON_INTERNAL_REGEX_H_
#define RAPIDJSON_INTERNAL_REGEX_H_

#include "../allocators.h"  // 包含分配器头文件
#include "../stream.h"  // 包含流处理头文件
#include "stack.h"  // 包含栈处理头文件

#ifdef __clang__  // 如果是 clang 编译器
RAPIDJSON_DIAG_PUSH  // 开启诊断信息
RAPIDJSON_DIAG_OFF(padded)  // 关闭指定类型的诊断信息
RAPIDJSON_DIAG_OFF(switch-enum)  // 关闭指定类型的诊断信息
#elif defined(_MSC_VER)  // 如果是 MSVC 编译器
RAPIDJSON_DIAG_PUSH  // 开启诊断信息
RAPIDJSON_DIAG_OFF(4512) // 关闭指定类型的诊断信息
#endif

#ifdef __GNUC__  // 如果是 GCC 编译器
RAPIDJSON_DIAG_PUSH  // 开启诊断信息
RAPIDJSON_DIAG_OFF(effc++)  // 关闭指定类型的诊断信息
#endif

#ifndef RAPIDJSON_REGEX_VERBOSE  // 如果未定义 RAPIDJSON_REGEX_VERBOSE
#define RAPIDJSON_REGEX_VERBOSE 0  // 设置 RAPIDJSON_REGEX_VERBOSE 为 0
#endif

RAPIDJSON_NAMESPACE_BEGIN  // 进入 rapidjson 命名空间
namespace internal {  // 进入 internal 命名空间

///////////////////////////////////////////////////////////////////////////////
// DecodedStream

template <typename SourceStream, typename Encoding>  // 定义模板类 DecodedStream
class DecodedStream {
public:
    DecodedStream(SourceStream& ss) : ss_(ss), codepoint_() { Decode(); }  // 构造函数，初始化成员变量并调用 Decode()
    unsigned Peek() { return codepoint_; }  // 返回当前字符的 Unicode 编码
    unsigned Take() {  // 取出当前字符的 Unicode 编码
        unsigned c = codepoint_;
        if (c) // 如果字符不为空
            Decode();  // 调用 Decode() 进行解码
        return c;  // 返回字符的 Unicode 编码
    }

private:
    void Decode() {  // 解码函数
        if (!Encoding::Decode(ss_, &codepoint_))  // 如果无法解码
            codepoint_ = 0;  // 将字符编码置为 0
    }

    SourceStream& ss_;  // 源流对象的引用
    unsigned codepoint_;  // 当前字符的 Unicode 编码
};

///////////////////////////////////////////////////////////////////////////////
// GenericRegex
static const SizeType kRegexInvalidState = ~SizeType(0);  //!< 代表 GenericRegex::State::out, out1 中的无效索引
static const SizeType kRegexInvalidRange = ~SizeType(0);

template <typename Encoding, typename Allocator>
class GenericRegexSearch;

//! 支持 ECMAscript 语法子集的正则表达式引擎
/*!
    支持的正则表达式语法:
    - \c ab     连接
    - \c a|b    选择
    - \c a?     零次或一次
    - \c a*     零次或多次
    - \c a+     一次或多次
    - \c a{3}   恰好 3 次
    - \c a{3,}  至少 3 次
    - \c a{3,5} 3 到 5 次
    - \c (ab)   分组
    - \c ^a     开头
    - \c a$     结尾
    - \c .      任意字符
    - \c [abc]  字符类
    - \c [a-c]  字符类范围
    - \c [a-z0-9_] 字符类组合
    - \c [^abc] 反向字符类
    - \c [^a-c] 反向字符类范围
    - \c [\b]   退格 (U+0008)
    - \c \\| \\\\ ...  转义字符
    - \c \\f 换页 (U+000C)
    - \c \\n 换行 (U+000A)
    - \c \\r 回车 (U+000D)
    - \c \\t 制表符 (U+0009)
    - \c \\v 垂直制表符 (U+000B)

    \note 这是一个 Thompson NFA 引擎，参考了 Cox, Russ 的实现
        "Regular Expression Matching Can Be Simple And Fast (but is slow in Java, Perl, PHP, Python, Ruby,...).",
        https://swtch.com/~rsc/regexp/regexp1.html
*/
template <typename Encoding, typename Allocator = CrtAllocator>
class GenericRegex {
public:
    typedef Encoding EncodingType;
    typedef typename Encoding::Ch Ch;
    template <typename, typename> friend class GenericRegexSearch;

    GenericRegex(const Ch* source, Allocator* allocator = 0) :
        ownAllocator_(allocator ? 0 : RAPIDJSON_NEW(Allocator)()), allocator_(allocator ? allocator : ownAllocator_),
        states_(allocator_, 256), ranges_(allocator_, 256), root_(kRegexInvalidState), stateCount_(), rangeCount_(),
        anchorBegin_(), anchorEnd_()
    {
        // 使用源字符串创建通用字符串流
        GenericStringStream<Encoding> ss(source);
        // 使用通用字符串流创建解码流
        DecodedStream<GenericStringStream<Encoding>, Encoding> ds(ss);
        // 解析解码流中的数据
        Parse(ds);
    }

    // 通用正则表达式的析构函数
    ~GenericRegex()
    {
        // 释放自己的分配器
        RAPIDJSON_DELETE(ownAllocator_);
    }

    // 检查正则表达式是否有效
    bool IsValid() const {
        // 返回根节点是否为有效状态
        return root_ != kRegexInvalidState;
    }
private:
    // 定义操作符枚举类型
    enum Operator {
        kZeroOrOne,
        kZeroOrMore,
        kOneOrMore,
        kConcatenation,
        kAlternation,
        kLeftParenthesis
    };

    // 定义特殊字符类别常量
    static const unsigned kAnyCharacterClass = 0xFFFFFFFF;   //!< For '.'
    static const unsigned kRangeCharacterClass = 0xFFFFFFFE;
    static const unsigned kRangeNegationFlag = 0x80000000;

    // 定义表示字符范围的结构体
    struct Range {
        unsigned start; // 范围起始值
        unsigned end;   // 范围结束值
        SizeType next;  // 下一个范围的索引
    };

    // 定义状态结构体
    struct State {
        SizeType out;     //!< Equals to kInvalid for matching state
        SizeType out1;    //!< Equals to non-kInvalid for split
        SizeType rangeStart;
        unsigned codepoint;
    };

    // 定义片段结构体
    struct Frag {
        Frag(SizeType s, SizeType o, SizeType m) : start(s), out(o), minIndex(m) {}
        SizeType start;
        SizeType out; //!< link-list of all output states
        SizeType minIndex;
    };

    // 获取指定索引的状态
    State& GetState(SizeType index) {
        RAPIDJSON_ASSERT(index < stateCount_);
        return states_.template Bottom<State>()[index];
    }

    // 获取指定索引的状态（常量版本）
    const State& GetState(SizeType index) const {
        RAPIDJSON_ASSERT(index < stateCount_);
        return states_.template Bottom<State>()[index];
    }

    // 获取指定索引的字符范围
    Range& GetRange(SizeType index) {
        RAPIDJSON_ASSERT(index < rangeCount_);
        return ranges_.template Bottom<Range>()[index];
    }

    // 获取指定索引的字符范围（常量版本）
    const Range& GetRange(SizeType index) const {
        RAPIDJSON_ASSERT(index < rangeCount_);
        return ranges_.template Bottom<Range>()[index];
    }

    // 从输入流中读取模板
    template <typename InputStream>
#if RAPIDJSON_REGEX_VERBOSE
            printf("root: %d\n", root_);
            for (SizeType i = 0; i < stateCount_ ; i++) {
                State& s = GetState(i);
                printf("[%2d] out: %2d out1: %2d c: '%c'\n", i, s.out, s.out1, (char)s.codepoint);
            }
            printf("\n");
#endif
        }
    }
    # 创建新的状态，并初始化其属性，返回新状态的索引
    SizeType NewState(SizeType out, SizeType out1, unsigned codepoint) {
        # 为新状态分配内存空间
        State* s = states_.template Push<State>();
        # 初始化新状态的属性
        s->out = out;
        s->out1 = out1;
        s->codepoint = codepoint;
        s->rangeStart = kRegexInvalidRange;
        # 返回新状态的索引，并递增状态计数
        return stateCount_++;
    }

    # 将操作数压入操作数栈中
    void PushOperand(Stack<Allocator>& operandStack, unsigned codepoint) {
        # 创建新状态，并将其作为操作数压入操作数栈中
        SizeType s = NewState(kRegexInvalidState, kRegexInvalidState, codepoint);
        *operandStack.template Push<Frag>() = Frag(s, s, s);
    }

    # 隐式连接操作，根据原子计数栈和操作符栈的情况进行隐式连接
    void ImplicitConcatenation(Stack<Allocator>& atomCountStack, Stack<Allocator>& operatorStack) {
        # 如果原子计数栈顶部的值不为0，则将连接操作符压入操作符栈中
        if (*atomCountStack.template Top<unsigned>())
            *operatorStack.template Push<Operator>() = kConcatenation;
        # 原子计数栈顶部的值加1
        (*atomCountStack.template Top<unsigned>())++;
    }

    # 追加操作，将状态 l1 的输出指向状态 l2
    SizeType Append(SizeType l1, SizeType l2) {
        # 保存原始状态 l1
        SizeType old = l1;
        # 循环直到状态 l1 的输出指向无效状态
        while (GetState(l1).out != kRegexInvalidState)
            l1 = GetState(l1).out;
        # 将状态 l1 的输出指向状态 l2
        GetState(l1).out = l2;
        # 返回原始状态 l1
        return old;
    }

    # 补丁操作，将状态 l 的输出指向状态 s
    void Patch(SizeType l, SizeType s) {
        # 循环直到状态 l 为无效状态
        for (SizeType next; l != kRegexInvalidState; l = next) {
            # 保存状态 l 的下一个状态
            next = GetState(l).out;
            # 将状态 l 的输出指向状态 s
            GetState(l).out = s;
        }
    }
    # 判断量词是否成立，根据给定的最小和最大次数
    bool EvalQuantifier(Stack<Allocator>& operandStack, unsigned n, unsigned m) {
        # 断言最小次数小于等于最大次数
        RAPIDJSON_ASSERT(n <= m);
        # 断言操作数栈的大小至少为一个 Frag 结构的大小
        RAPIDJSON_ASSERT(operandStack.GetSize() >= sizeof(Frag));

        if (n == 0) {
            if (m == 0)                             // a{0} not support
                return false;
            else if (m == kInfinityQuantifier)
                Eval(operandStack, kZeroOrMore);    // a{0,} -> a*
            else {
                Eval(operandStack, kZeroOrOne);         // a{0,5} -> a?
                for (unsigned i = 0; i < m - 1; i++)
                    CloneTopOperand(operandStack);      // a{0,5} -> a? a? a? a? a?
                for (unsigned i = 0; i < m - 1; i++)
                    Eval(operandStack, kConcatenation); // a{0,5} -> a?a?a?a?a?
            }
            return true;
        }

        for (unsigned i = 0; i < n - 1; i++)        // a{3} -> a a a
            CloneTopOperand(operandStack);

        if (m == kInfinityQuantifier)
            Eval(operandStack, kOneOrMore);         // a{3,} -> a a a+
        else if (m > n) {
            CloneTopOperand(operandStack);          // a{3,5} -> a a a a
            Eval(operandStack, kZeroOrOne);         // a{3,5} -> a a a a?
            for (unsigned i = n; i < m - 1; i++)
                CloneTopOperand(operandStack);      // a{3,5} -> a a a a? a?
            for (unsigned i = n; i < m; i++)
                Eval(operandStack, kConcatenation); // a{3,5} -> a a aa?a?
        }

        for (unsigned i = 0; i < n - 1; i++)
            Eval(operandStack, kConcatenation);     // a{3} -> aaa, a{3,} -> aaa+, a{3.5} -> aaaa?a?

        return true;
    }

    # 返回两个数中的最小值
    static SizeType Min(SizeType a, SizeType b) { return a < b ? a : b; }
    // 从操作数栈顶复制操作数
    void CloneTopOperand(Stack<Allocator>& operandStack) {
        // 复制操作数栈顶的 Frag 对象，使用复制构造函数以防止无效化
        const Frag src = *operandStack.template Top<Frag>();
        // 计算状态数量，假设顶部操作数包含状态在 [src->minIndex, stateCount_) 范围内
        SizeType count = stateCount_ - src.minIndex;
        // 在状态数组中分配 count 个状态的空间，并从 src->minIndex 处开始复制状态
        State* s = states_.template Push<State>(count);
        memcpy(s, &GetState(src.minIndex), count * sizeof(State));
        // 更新复制的状态中的 out 和 out1 值，如果不为 kRegexInvalidState 则加上 count
        for (SizeType j = 0; j < count; j++) {
            if (s[j].out != kRegexInvalidState)
                s[j].out += count;
            if (s[j].out1 != kRegexInvalidState)
                s[j].out1 += count;
        }
        // 更新操作数栈顶的 Frag 对象，将其 start、out 和 minIndex 分别加上 count
        *operandStack.template Push<Frag>() = Frag(src.start + count, src.out + count, src.minIndex + count);
        // 更新状态数量
        stateCount_ += count;
    }

    // 解析无符号整数
    template <typename InputStream>
    bool ParseUnsigned(DecodedStream<InputStream, Encoding>& ds, unsigned* u) {
        unsigned r = 0;
        // 如果下一个字符不是数字，则解析失败
        if (ds.Peek() < '0' || ds.Peek() > '9')
            return false;
        // 循环解析数字字符，计算无符号整数值
        while (ds.Peek() >= '0' && ds.Peek() <= '9') {
            // 检查是否会发生溢出
            if (r >= 429496729 && ds.Peek() > '5') // 2^32 - 1 = 4294967295
                return false; // 溢出
            r = r * 10 + (ds.Take() - '0');
        }
        // 将解析得到的无符号整数赋值给 u
        *u = r;
        return true;
    }

    // 创建新的范围
    SizeType NewRange(unsigned codepoint) {
        // 在范围数组中分配一个 Range 对象的空间
        Range* r = ranges_.template Push<Range>();
        // 设置范围的起始和结束码点为 codepoint，next 为 kRegexInvalidRange
        r->start = r->end = codepoint;
        r->next = kRegexInvalidRange;
        // 返回范围数量并递增
        return rangeCount_++;
    }
    # 判断是否需要对字符进行转义处理
    bool CharacterEscape(DecodedStream<InputStream, Encoding>& ds, unsigned* escapedCodepoint) {
        # 从输入流中获取下一个字符的 Unicode 码点
        unsigned codepoint;
        switch (codepoint = ds.Take()) {
            # 判断是否是需要转义的特殊字符
            case '^':
            case '$':
            case '|':
            case '(':
            case ')':
            case '?':
            case '*':
            case '+':
            case '.':
            case '[':
            case ']':
            case '{':
            case '}':
            case '\\':
                # 如果是特殊字符，则将其 Unicode 码点赋值给 escapedCodepoint，并返回 true
                *escapedCodepoint = codepoint; return true;
            # 处理常见的转义字符
            case 'f': *escapedCodepoint = 0x000C; return true;
            case 'n': *escapedCodepoint = 0x000A; return true;
            case 'r': *escapedCodepoint = 0x000D; return true;
            case 't': *escapedCodepoint = 0x0009; return true;
            case 'v': *escapedCodepoint = 0x000B; return true;
            # 如果不是特殊字符或常见转义字符，则返回 false，表示不支持的转义字符
            default:
                return false; // Unsupported escape character
        }
    }

    # 分配器指针
    Allocator* ownAllocator_;
    # 分配器指针
    Allocator* allocator_;
    # 分配器栈
    Stack<Allocator> states_;
    # 分配器栈
    Stack<Allocator> ranges_;
    # 根节点索引
    SizeType root_;
    # 状态数量
    SizeType stateCount_;
    # 范围数量
    SizeType rangeCount_;

    # 无限量词的标识
    static const unsigned kInfinityQuantifier = ~0u;

    # 用于 SearchWithAnchoring() 的锚定起始位置标识
    bool anchorBegin_;
    # 用于 SearchWithAnchoring() 的锚定结束位置标识
    bool anchorEnd_;
    // 结束类定义
};

// 定义通用的正则表达式搜索类模板
template <typename RegexType, typename Allocator = CrtAllocator>
class GenericRegexSearch {
public:
    // 定义类型别名
    typedef typename RegexType::EncodingType Encoding;
    typedef typename Encoding::Ch Ch;

    // 构造函数，初始化正则表达式和分配器
    GenericRegexSearch(const RegexType& regex, Allocator* allocator = 0) :
        regex_(regex), allocator_(allocator), ownAllocator_(0),
        state0_(allocator, 0), state1_(allocator, 0), stateSet_()
    {
        // 断言正则表达式有效
        RAPIDJSON_ASSERT(regex_.IsValid());
        // 如果分配器为空，创建一个新的分配器
        if (!allocator_)
            ownAllocator_ = allocator_ = RAPIDJSON_NEW(Allocator)();
        // 分配状态集内存
        stateSet_ = static_cast<unsigned*>(allocator_->Malloc(GetStateSetSize()));
        // 为状态0和状态1预留空间
        state0_.template Reserve<SizeType>(regex_.stateCount_);
        state1_.template Reserve<SizeType>(regex_.stateCount_);
    }

    // 析构函数，释放内存
    ~GenericRegexSearch() {
        Allocator::Free(stateSet_);
        RAPIDJSON_DELETE(ownAllocator_);
    }

    // 匹配输入流
    template <typename InputStream>
    bool Match(InputStream& is) {
        return SearchWithAnchoring(is, true, true);
    }

    // 匹配字符串
    bool Match(const Ch* s) {
        GenericStringStream<Encoding> is(s);
        return Match(is);
    }

    // 在输入流中搜索
    template <typename InputStream>
    bool Search(InputStream& is) {
        return SearchWithAnchoring(is, regex_.anchorBegin_, regex_.anchorEnd_);
    }

    // 在字符串中搜索
    bool Search(const Ch* s) {
        GenericStringStream<Encoding> is(s);
        return Search(is);
    }

private:
    // 定义类型别名
    typedef typename RegexType::State State;
    typedef typename RegexType::Range Range;

    // 定义模板函数
    template <typename InputStream>
    # 使用输入流和锚定信息进行搜索，返回是否匹配
    bool SearchWithAnchoring(InputStream& is, bool anchorBegin, bool anchorEnd) {
        # 使用解码流对输入流进行解码
        DecodedStream<InputStream, Encoding> ds(is);

        # 清空状态集合
        state0_.Clear();
        # 当前状态集合指针指向state0_，下一个状态集合指针指向state1_
        Stack<Allocator> *current = &state0_, *next = &state1_;
        # 获取状态集合大小
        const size_t stateSetSize = GetStateSetSize();
        # 将状态集合初始化为0
        std::memset(stateSet_, 0, stateSetSize);

        # 添加初始状态
        bool matched = AddState(*current, regex_.root_);
        unsigned codepoint;
        # 循环直到当前状态集合为空或者输入流中没有字符
        while (!current->Empty() && (codepoint = ds.Take()) != 0) {
            # 将状态集合初始化为0
            std::memset(stateSet_, 0, stateSetSize);
            # 清空下一个状态集合
            next->Clear();
            matched = false;
            # 遍历当前状态集合中的状态
            for (const SizeType* s = current->template Bottom<SizeType>(); s != current->template End<SizeType>(); ++s) {
                # 获取当前状态
                const State& sr = regex_.GetState(*s);
                # 如果当前状态的字符与输入字符匹配，或者是任意字符类，或者是范围字符类并且匹配范围
                if (sr.codepoint == codepoint ||
                    sr.codepoint == RegexType::kAnyCharacterClass ||
                    (sr.codepoint == RegexType::kRangeCharacterClass && MatchRange(sr.rangeStart, codepoint)))
                {
                    # 添加下一个状态并更新匹配状态
                    matched = AddState(*next, sr.out) || matched;
                    # 如果不需要锚定结束并且已经匹配，则返回true
                    if (!anchorEnd && matched)
                        return true;
                }
                # 如果不需要锚定开始，则添加初始状态
                if (!anchorBegin)
                    AddState(*next, regex_.root_);
            }
            # 交换当前状态集合和下一个状态集合
            internal::Swap(current, next);
        }

        # 返回是否匹配
        return matched;
    }

    # 获取状态集合大小
    size_t GetStateSetSize() const {
        return (regex_.stateCount_ + 31) / 32 * 4;
    }

    # 返回添加的状态是否是匹配状态
    # 向状态机中添加状态，参数为状态栈和状态索引
    bool AddState(Stack<Allocator>& l, SizeType index) {
        # 断言状态索引不为无效状态
        RAPIDJSON_ASSERT(index != kRegexInvalidState);

        # 获取状态机中的状态
        const State& s = regex_.GetState(index);
        # 如果状态有两个输出状态，即分裂状态
        if (s.out1 != kRegexInvalidState) { // Split
            # 递归调用AddState函数，将s.out1状态添加到状态栈中，并记录是否匹配
            bool matched = AddState(l, s.out);
            # 将s.out1状态添加到状态栈中，并返回是否匹配
            return AddState(l, s.out1) || matched;
        }
        # 如果状态集合中没有当前状态
        else if (!(stateSet_[index >> 5] & (1u << (index & 31)))) {
            # 将当前状态添加到状态集合中
            stateSet_[index >> 5] |= (1u << (index & 31));
            # 将当前状态索引添加到状态栈中
            *l.template PushUnsafe<SizeType>() = index;
        }
        # 返回当前状态是否为无效状态
        return s.out == kRegexInvalidState; // by using PushUnsafe() above, we can ensure s is not validated due to reallocation.
    }

    # 匹配字符范围
    bool MatchRange(SizeType rangeIndex, unsigned codepoint) const {
        # 判断是否为肯定范围
        bool yes = (regex_.GetRange(rangeIndex).start & RegexType::kRangeNegationFlag) == 0;
        # 遍历字符范围
        while (rangeIndex != kRegexInvalidRange) {
            # 获取当前范围
            const Range& r = regex_.GetRange(rangeIndex);
            # 如果字符在范围内，则返回yes
            if (codepoint >= (r.start & ~RegexType::kRangeNegationFlag) && codepoint <= r.end)
                return yes;
            # 获取下一个范围
            rangeIndex = r.next;
        }
        # 返回否定结果
        return !yes;
    }

    # 正则表达式类型的引用
    const RegexType& regex_;
    # 分配器指针
    Allocator* allocator_;
    # 拥有的分配器指针
    Allocator* ownAllocator_;
    # 状态栈0
    Stack<Allocator> state0_;
    # 状态栈1
    Stack<Allocator> state1_;
    # 状态集合
    uint32_t* stateSet_;
};

// 定义使用UTF-8编码的通用正则表达式类型
typedef GenericRegex<UTF8<> > Regex;
// 定义通用正则表达式搜索类型
typedef GenericRegexSearch<Regex> RegexSearch;

} // namespace internal
// 结束RAPIDJSON命名空间

RAPIDJSON_NAMESPACE_END

#ifdef __GNUC__
// 恢复之前的诊断状态
RAPIDJSON_DIAG_POP
#endif

#if defined(__clang__) || defined(_MSC_VER)
// 恢复之前的诊断状态
RAPIDJSON_DIAG_POP
#endif

#endif // RAPIDJSON_INTERNAL_REGEX_H_
```