# `xmrig\src\3rdparty\rapidjson\prettywriter.h`

```
#ifndef RAPIDJSON_PRETTYWRITER_H_
#define RAPIDJSON_PRETTYWRITER_H_

#include "writer.h"  // 包含 writer.h 头文件

#ifdef __GNUC__  // 如果是 GNU 编译器
RAPIDJSON_DIAG_PUSH  // 开启诊断信息
RAPIDJSON_DIAG_OFF(effc++)  // 关闭指定的诊断信息
#endif

#if defined(__clang__)  // 如果是 Clang 编译器
RAPIDJSON_DIAG_PUSH  // 开启诊断信息
RAPIDJSON_DIAG_OFF(c++98-compat)  // 关闭指定的诊断信息
#endif

RAPIDJSON_NAMESPACE_BEGIN  // 开始 RapidJSON 命名空间

//! Combination of PrettyWriter format flags.
/*! \see PrettyWriter::SetFormatOptions
 */
enum PrettyFormatOptions {
    kFormatDefault = 0,         //!< Default pretty formatting.
    kFormatSingleLineArray = 1  //!< Format arrays on a single line.
};

//! Writer with indentation and spacing.
/*!
    \tparam OutputStream Type of output os.
    \tparam SourceEncoding Encoding of source string.
    \tparam TargetEncoding Encoding of output stream.
    \tparam StackAllocator Type of allocator for allocating memory of stack.
*/
template<typename OutputStream, typename SourceEncoding = UTF8<>, typename TargetEncoding = UTF8<>, typename StackAllocator = CrtAllocator, unsigned writeFlags = kWriteDefaultFlags>
class PrettyWriter : public Writer<OutputStream, SourceEncoding, TargetEncoding, StackAllocator, writeFlags> {
public:
    typedef Writer<OutputStream, SourceEncoding, TargetEncoding, StackAllocator, writeFlags> Base;  // 定义 Writer 类型的别名 Base
    typedef typename Base::Ch Ch;  // 定义字符类型的别名 Ch

    //! Constructor
    # 用于输出流的构造函数，可以指定用户提供的分配器，如果为null，则创建一个私有的分配器
    explicit PrettyWriter(OutputStream& os, StackAllocator* allocator = 0, size_t levelDepth = Base::kDefaultLevelDepth) :
        Base(os, allocator, levelDepth), indentChar_(' '), indentCharCount_(4), formatOptions_(kFormatDefault) {}
    
    # 用于私有分配器的构造函数，可以指定用户提供的分配器，如果为null，则创建一个私有的分配器
    explicit PrettyWriter(StackAllocator* allocator = 0, size_t levelDepth = Base::kDefaultLevelDepth) :
        Base(allocator, levelDepth), indentChar_(' '), indentCharCount_(4), formatOptions_(kFormatDefault) {}
#if RAPIDJSON_HAS_CXX11_RVALUE_REFS
    // 如果支持 C++11 的右值引用，则实现移动构造函数
    PrettyWriter(PrettyWriter&& rhs) :
        Base(std::forward<PrettyWriter>(rhs)), indentChar_(rhs.indentChar_), indentCharCount_(rhs.indentCharCount_), formatOptions_(rhs.formatOptions_) {}
#endif

    //! 设置自定义缩进
    /*! \param indentChar       缩进的字符。必须是空白字符（' ', '\\t', '\\n', '\\r'）。
        \param indentCharCount  每个缩进级别的缩进字符数。
        \note 默认缩进是 4 个空格。
    */
    PrettyWriter& SetIndent(Ch indentChar, unsigned indentCharCount) {
        RAPIDJSON_ASSERT(indentChar == ' ' || indentChar == '\t' || indentChar == '\n' || indentChar == '\r');
        indentChar_ = indentChar;
        indentCharCount_ = indentCharCount;
        return *this;
    }

    //! 设置漂亮的写入格式选项
    /*! \param options 格式选项。
    */
    PrettyWriter& SetFormatOptions(PrettyFormatOptions options) {
        formatOptions_ = options;
        return *this;
    }

    /*! @name Implementation of Handler
        \see Handler
    */
    //@{

    bool Null()                 { PrettyPrefix(kNullType);   return Base::EndValue(Base::WriteNull()); }
    bool Bool(bool b)           { PrettyPrefix(b ? kTrueType : kFalseType); return Base::EndValue(Base::WriteBool(b)); }
    bool Int(int i)             { PrettyPrefix(kNumberType); return Base::EndValue(Base::WriteInt(i)); }
    bool Uint(unsigned u)       { PrettyPrefix(kNumberType); return Base::EndValue(Base::WriteUint(u)); }
    bool Int64(int64_t i64)     { PrettyPrefix(kNumberType); return Base::EndValue(Base::WriteInt64(i64)); }
    bool Uint64(uint64_t u64)   { PrettyPrefix(kNumberType); return Base::EndValue(Base::WriteUint64(u64));  }
    bool Double(double d)       { PrettyPrefix(kNumberType); return Base::EndValue(Base::WriteDouble(d)); }
    # 检查输入的字符串是否为原始数字，可选择是否复制字符串
    bool RawNumber(const Ch* str, SizeType length, bool copy = false) {
        # 断言输入的字符串不为空
        RAPIDJSON_ASSERT(str != 0);
        # 忽略复制参数
        (void)copy;
        # 添加数字类型的前缀
        PrettyPrefix(kNumberType);
        # 结束值的写入，并返回结果
        return Base::EndValue(Base::WriteString(str, length));
    }
    
    # 将输入的字符串作为 JSON 字符串处理，可选择是否复制字符串
    bool String(const Ch* str, SizeType length, bool copy = false) {
        # 断言输入的字符串不为空
        RAPIDJSON_ASSERT(str != 0);
        # 忽略复制参数
        (void)copy;
        # 添加字符串类型的前缀
        PrettyPrefix(kStringType);
        # 结束值的写入，并返回结果
        return Base::EndValue(Base::WriteString(str, length));
    }
#if RAPIDJSON_HAS_STDSTRING
    // 如果编译器支持 std::string 类型，使用 std::string 类型的字符串来生成 JSON 字符串
    bool String(const std::basic_string<Ch>& str) {
        return String(str.data(), SizeType(str.size()));
    }
#endif

    // 开始一个 JSON 对象
    bool StartObject() {
        // 添加 JSON 对象的前缀
        PrettyPrefix(kObjectType);
        // 在堆栈中创建一个新的 JSON 对象
        new (Base::level_stack_.template Push<typename Base::Level>()) typename Base::Level(false);
        // 写入 JSON 对象的开始标记
        return Base::WriteStartObject();
    }

    // 添加 JSON 对象的键
    bool Key(const Ch* str, SizeType length, bool copy = false) { return String(str, length, copy); }

#if RAPIDJSON_HAS_STDSTRING
    // 如果编译器支持 std::string 类型，使用 std::string 类型的字符串作为键
    bool Key(const std::basic_string<Ch>& str) {
        return Key(str.data(), SizeType(str.size()));
    }
#endif

    // 结束一个 JSON 对象
    bool EndObject(SizeType memberCount = 0) {
        (void)memberCount;
        // 断言当前堆栈中的元素数量大于等于 JSON 对象的大小
        RAPIDJSON_ASSERT(Base::level_stack_.GetSize() >= sizeof(typename Base::Level)); // not inside an Object
        // 断言当前不在数组内，而是在对象内
        RAPIDJSON_ASSERT(!Base::level_stack_.template Top<typename Base::Level>()->inArray); // currently inside an Array, not Object
        // 断言对象的键值对数量为偶数
        RAPIDJSON_ASSERT(0 == Base::level_stack_.template Top<typename Base::Level>()->valueCount % 2); // Object has a Key without a Value

        // 检查对象是否为空
        bool empty = Base::level_stack_.template Pop<typename Base::Level>(1)->valueCount == 0;

        // 如果对象不为空，添加换行和缩进
        if (!empty) {
            Base::os_->Put('\n');
            WriteIndent();
        }
        // 结束对象的值，并写入结束标记
        bool ret = Base::EndValue(Base::WriteEndObject());
        (void)ret;
        // 断言结束操作成功
        RAPIDJSON_ASSERT(ret == true);
        // 如果堆栈为空，表示 JSON 文本结束，刷新输出
        if (Base::level_stack_.Empty()) // end of json text
            Base::Flush();
        return true;
    }

    // 开始一个 JSON 数组
    bool StartArray() {
        // 添加 JSON 数组的前缀
        PrettyPrefix(kArrayType);
        // 在堆栈中创建一个新的 JSON 数组
        new (Base::level_stack_.template Push<typename Base::Level>()) typename Base::Level(true);
        // 写入 JSON 数组的开始标记
        return Base::WriteStartArray();
    }
    # 结束 JSON 数组的解析
    bool EndArray(SizeType memberCount = 0) {
        # 忽略成员数量
        (void)memberCount;
        # 确保堆栈中有足够的级别
        RAPIDJSON_ASSERT(Base::level_stack_.GetSize() >= sizeof(typename Base::Level));
        # 确保当前在数组中
        RAPIDJSON_ASSERT(Base::level_stack_.template Top<typename Base::Level>()->inArray);
        # 弹出当前级别
        auto level = Base::level_stack_.template Pop<typename Base::Level>(1);
        # 检查数组是否为空
        bool empty = level->valueCount == 0;

        # 如果数组不为空且不在同一行，则换行并缩进
        if (!empty && !level->inLine) {
            Base::os_->Put('\n');
            WriteIndent();
        }
        # 结束值的写入
        bool ret = Base::EndValue(Base::WriteEndArray());
        # 忽略返回值
        (void)ret;
        # 确保返回值为 true
        RAPIDJSON_ASSERT(ret == true);
        # 如果堆栈为空，则结束 JSON 文本
        if (Base::level_stack_.Empty()) 
            Base::Flush();
        return true;
    }

    //@}

    /*! @name Convenience extensions */
    //@{

    # 写入字符串值的简化版本
    bool String(const Ch* str) { return String(str, internal::StrLen(str)); }
    # 写入键的简化版本
    bool Key(const Ch* str) { return Key(str, internal::StrLen(str)); }

    //@}

    # 写入原始 JSON 值
    /*!
        用户可以将字符串化的 JSON 写入为值。

        \param json 一个格式良好的 JSON 值。它不应该在 [0, length - 1] 范围内包含空字符。
        \param length JSON 的长度
        \param type JSON 根节点的类型
        \note 使用 PrettyWriter::RawValue() 时，结果 JSON 可能无法正确缩进。
    */
    bool RawValue(const Ch* json, size_t length, Type type) {
        # 确保 json 不为空
        RAPIDJSON_ASSERT(json != 0);
        # 写入 JSON 值的前缀
        PrettyPrefix(type);
        # 结束值的写入
        return Base::EndValue(Base::WriteRawValue(json, length));
    }
protected:
    void PrettyPrefix(Type type) {
        (void)type;  // 忽略未使用的参数
        if (Base::level_stack_.GetSize() != 0) { // 如果当前值不是根节点
            typename Base::Level* level = Base::level_stack_.template Top<typename Base::Level>();  // 获取当前层级

            if (level->inArray) {  // 如果当前在数组中
                level->inLine = (formatOptions_ & kFormatSingleLineArray) && type != kObjectType && type != kArrayType;  // 根据格式选项和类型设置是否在同一行

                if (level->valueCount > 0) {
                    Base::os_->Put(',');  // 如果不是数组中的第一个元素，添加逗号
                    if (level->inLine) {
                        Base::os_->Put(' ');  // 如果在同一行，添加空格
                    }
                }

                if (!level->inLine) {
                    Base::os_->Put('\n');  // 如果不在同一行，换行
                    WriteIndent();  // 写入缩进
                }
            }
            else {  // 如果在对象中
                if (level->valueCount > 0) {
                    if (level->valueCount % 2 == 0) {
                        Base::os_->Put(',');  // 如果值的数量为偶数，添加逗号
                        Base::os_->Put('\n');  // 换行
                    }
                    else {
                        Base::os_->Put(':');  // 如果值的数量为奇数，添加冒号
                        Base::os_->Put(' ');  // 添加空格
                    }
                }
                else
                    Base::os_->Put('\n');  // 如果值的数量为0，换行

                if (level->valueCount % 2 == 0)
                    WriteIndent();  // 如果值的数量为偶数，写入缩进
            }
            if (!level->inArray && level->valueCount % 2 == 0)
                RAPIDJSON_ASSERT(type == kStringType);  // 如果不在数组中，且值的数量为偶数，断言类型为字符串
            level->valueCount++;  // 值的数量加一
        }
        else {
            RAPIDJSON_ASSERT(!Base::hasRoot_);  // 断言没有根节点
            Base::hasRoot_ = true;  // 设置有根节点
        }
    }

    void WriteIndent()  {
        size_t count = (Base::level_stack_.GetSize() / sizeof(typename Base::Level)) * indentCharCount_;  // 计算缩进的字符数量
        PutN(*Base::os_, static_cast<typename OutputStream::Ch>(indentChar_), count);  // 写入指定数量的缩进字符
    }

    Ch indentChar_;  // 缩进字符
    # 用于存储缩进字符的数量
    unsigned indentCharCount_;
    # 用于存储格式化选项的对象
    PrettyFormatOptions formatOptions_;
private:
    // 禁止复制构造函数和赋值运算符。
    PrettyWriter(const PrettyWriter&);
    PrettyWriter& operator=(const PrettyWriter&);
};

RAPIDJSON_NAMESPACE_END

#if defined(__clang__)
RAPIDJSON_DIAG_POP
#endif

#ifdef __GNUC__
RAPIDJSON_DIAG_POP
#endif

#endif // RAPIDJSON_RAPIDJSON_H_
```