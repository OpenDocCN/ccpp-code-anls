# `xmrig\src\3rdparty\rapidjson\encodings.h`

```
// 定义了 rapidjson 的编码概念
// 包含了字符类型 Ch，字符实际上是 Unicode 编码中的代码单元
// 定义了是否支持 Unicode 的枚举值
// 提供了将 Unicode 码点编码到输出流的方法
// 提供了从输入流解码 Unicode 码点的方法
    //! \return true if a valid codepoint can be decoded from the stream.
    // 从流中解码一个有效的码点，如果可以解码则返回true
    template <typename InputStream>
    static bool Decode(InputStream& is, unsigned* codepoint);
    
    //! \brief Validate one Unicode codepoint from an encoded stream.
    //! \param is Input stream to obtain codepoint.
    //! \param os Output for copying one codepoint.
    //! \return true if it is valid.
    //! \note This function just validating and copying the codepoint without actually decode it.
    // 从编码流中验证一个Unicode码点，将其复制到输出流中，如果有效则返回true
    template <typename InputStream, typename OutputStream>
    static bool Validate(InputStream& is, OutputStream& os);
    
    // The following functions are deal with byte streams.
    
    //! Take a character from input byte stream, skip BOM if exist.
    // 从输入字节流中获取一个字符，如果存在BOM则跳过
    template <typename InputByteStream>
    static CharType TakeBOM(InputByteStream& is);
    
    //! Take a character from input byte stream.
    // 从输入字节流中获取一个字符
    template <typename InputByteStream>
    static Ch Take(InputByteStream& is);
    
    //! Put BOM to output byte stream.
    // 将BOM放入输出字节流中
    template <typename OutputByteStream>
    static void PutBOM(OutputByteStream& os);
    
    //! Put a character to output byte stream.
    // 将一个字符放入输出字节流中
    template <typename OutputByteStream>
    static void Put(OutputByteStream& os, Ch c);
// 结构体模板，用于UTF-8编码
/*! http://en.wikipedia.org/wiki/UTF-8
    http://tools.ietf.org/html/rfc3629
    \tparam CharType 用于存储8位UTF-8数据的代码单元。默认为char。
    \note 实现编码概念
*/
template<typename CharType = char>
struct UTF8 {
    typedef CharType Ch;

    enum { supportUnicode = 1 };

    // 将Unicode代码点编码为UTF-8格式并写入输出流
    template<typename OutputStream>
    static void Encode(OutputStream& os, unsigned codepoint) {
        if (codepoint <= 0x7F)
            os.Put(static_cast<Ch>(codepoint & 0xFF));
        else if (codepoint <= 0x7FF) {
            os.Put(static_cast<Ch>(0xC0 | ((codepoint >> 6) & 0xFF)));
            os.Put(static_cast<Ch>(0x80 | ((codepoint & 0x3F))));
        }
        else if (codepoint <= 0xFFFF) {
            os.Put(static_cast<Ch>(0xE0 | ((codepoint >> 12) & 0xFF)));
            os.Put(static_cast<Ch>(0x80 | ((codepoint >> 6) & 0x3F)));
            os.Put(static_cast<Ch>(0x80 | (codepoint & 0x3F)));
        }
        else {
            RAPIDJSON_ASSERT(codepoint <= 0x10FFFF);
            os.Put(static_cast<Ch>(0xF0 | ((codepoint >> 18) & 0xFF)));
            os.Put(static_cast<Ch>(0x80 | ((codepoint >> 12) & 0x3F)));
            os.Put(static_cast<Ch>(0x80 | ((codepoint >> 6) & 0x3F)));
            os.Put(static_cast<Ch>(0x80 | (codepoint & 0x3F)));
        }
    }

    // 其他UTF-8编码相关函数
    template<typename OutputStream>
    // 将 Unicode 编码的 codepoint 转换成 UTF-8 编码并写入输出流 os
    static void EncodeUnsafe(OutputStream& os, unsigned codepoint) {
        // 如果 codepoint 在 0x7F 以内，则直接写入一个字节
        if (codepoint <= 0x7F)
            PutUnsafe(os, static_cast<Ch>(codepoint & 0xFF));
        // 如果 codepoint 在 0x7FF 以内，则写入两个字节
        else if (codepoint <= 0x7FF) {
            PutUnsafe(os, static_cast<Ch>(0xC0 | ((codepoint >> 6) & 0xFF)));
            PutUnsafe(os, static_cast<Ch>(0x80 | ((codepoint & 0x3F))));
        }
        // 如果 codepoint 在 0xFFFF 以内，则写入三个字节
        else if (codepoint <= 0xFFFF) {
            PutUnsafe(os, static_cast<Ch>(0xE0 | ((codepoint >> 12) & 0xFF)));
            PutUnsafe(os, static_cast<Ch>(0x80 | ((codepoint >> 6) & 0x3F)));
            PutUnsafe(os, static_cast<Ch>(0x80 | (codepoint & 0x3F)));
        }
        // 如果 codepoint 在 0x10FFFF 以内，则写入四个字节
        else {
            RAPIDJSON_ASSERT(codepoint <= 0x10FFFF);
            PutUnsafe(os, static_cast<Ch>(0xF0 | ((codepoint >> 18) & 0xFF)));
            PutUnsafe(os, static_cast<Ch>(0x80 | ((codepoint >> 12) & 0x3F)));
            PutUnsafe(os, static_cast<Ch>(0x80 | ((codepoint >> 6) & 0x3F)));
            PutUnsafe(os, static_cast<Ch>(0x80 | (codepoint & 0x3F)));
        }
    }

    // 从输入流 is 中解码 UTF-8 编码的字符，存储到 codepoint 中
    template <typename InputStream>
    static bool Decode(InputStream& is, unsigned* codepoint) {
// 定义宏，用于将输入流中的字符取出并赋值给变量 c，然后将其存入 codepoint 中
#define RAPIDJSON_COPY() c = is.Take(); *codepoint = (*codepoint << 6) | (static_cast<unsigned char>(c) & 0x3Fu)
// 定义宏，用于根据掩码 mask 对字符进行位运算，并将结果与 result 进行逻辑与操作
#define RAPIDJSON_TRANS(mask) result &= ((GetRange(static_cast<unsigned char>(c)) & mask) != 0)
// 定义宏，用于依次执行 RAPIDJSON_COPY 和 RAPIDJSON_TRANS(0x70) 操作
#define RAPIDJSON_TAIL() RAPIDJSON_COPY(); RAPIDJSON_TRANS(0x70)

// 声明变量 c，类型为输入流的字符类型
typename InputStream::Ch c = is.Take();
// 如果字符 c 的最高位为 0，则将其赋值给 codepoint，并返回 true
if (!(c & 0x80)) {
    *codepoint = static_cast<unsigned char>(c);
    return true;
}

// 获取字符 c 的范围
unsigned char type = GetRange(static_cast<unsigned char>(c));
// 根据字符 c 的范围进行不同的处理
if (type >= 32) {
    *codepoint = 0;
} else {
    *codepoint = (0xFFu >> type) & static_cast<unsigned char>(c);
}
// 初始化结果为 true
bool result = true;
// 根据字符 c 的范围进行不同的处理
switch (type) {
case 2: RAPIDJSON_TAIL(); return result;
case 3: RAPIDJSON_TAIL(); RAPIDJSON_TAIL(); return result;
case 4: RAPIDJSON_COPY(); RAPIDJSON_TRANS(0x50); RAPIDJSON_TAIL(); return result;
case 5: RAPIDJSON_COPY(); RAPIDJSON_TRANS(0x10); RAPIDJSON_TAIL(); RAPIDJSON_TAIL(); return result;
case 6: RAPIDJSON_TAIL(); RAPIDJSON_TAIL(); RAPIDJSON_TAIL(); return result;
case 10: RAPIDJSON_COPY(); RAPIDJSON_TRANS(0x20); RAPIDJSON_TAIL(); return result;
case 11: RAPIDJSON_COPY(); RAPIDJSON_TRANS(0x60); RAPIDJSON_TAIL(); RAPIDJSON_TAIL(); return result;
default: return false;
}
// 取消定义之前定义的宏
#undef RAPIDJSON_COPY
#undef RAPIDJSON_TRANS
#undef RAPIDJSON_TAIL
}

// 定义模板函数 Validate，用于验证输入流和输出流
template <typename InputStream, typename OutputStream>
static bool Validate(InputStream& is, OutputStream& os) {
// 定义宏，用于将输入流中的字符取出并存入输出流中
#define RAPIDJSON_COPY() os.Put(c = is.Take())
// 定义宏，用于根据掩码 mask 对字符进行位运算，并将结果与 result 进行逻辑与操作
#define RAPIDJSON_TRANS(mask) result &= ((GetRange(static_cast<unsigned char>(c)) & mask) != 0)
// 定义宏，用于在代码中执行一系列操作
#define RAPIDJSON_TAIL() RAPIDJSON_COPY(); RAPIDJSON_TRANS(0x70)
        // 声明字符变量c
        Ch c;
        // 执行宏定义的操作
        RAPIDJSON_COPY();
        // 如果字符c的最高位不为1，则返回true
        if (!(c & 0x80))
            return true;

        // 声明布尔变量result并赋值为true
        bool result = true;
        // 根据字符c的范围进行不同的操作
        switch (GetRange(static_cast<unsigned char>(c))) {
        case 2: RAPIDJSON_TAIL(); return result;
        case 3: RAPIDJSON_TAIL(); RAPIDJSON_TAIL(); return result;
        case 4: RAPIDJSON_COPY(); RAPIDJSON_TRANS(0x50); RAPIDJSON_TAIL(); return result;
        case 5: RAPIDJSON_COPY(); RAPIDJSON_TRANS(0x10); RAPIDJSON_TAIL(); RAPIDJSON_TAIL(); return result;
        case 6: RAPIDJSON_TAIL(); RAPIDJSON_TAIL(); RAPIDJSON_TAIL(); return result;
        case 10: RAPIDJSON_COPY(); RAPIDJSON_TRANS(0x20); RAPIDJSON_TAIL(); return result;
        case 11: RAPIDJSON_COPY(); RAPIDJSON_TRANS(0x60); RAPIDJSON_TAIL(); RAPIDJSON_TAIL(); return result;
        default: return false;
        }
// 取消宏定义
#undef RAPIDJSON_COPY
#undef RAPIDJSON_TRANS
#undef RAPIDJSON_TAIL
    }
    static unsigned char GetRange(unsigned char c) {
        // 定义一个静态的无符号字符型数组，用于存储字符类型
        // 根据 http://bjoern.hoehrmann.de/utf-8/decoder/dfa/ 中的DFA进行映射
        // 新的映射为 1 -> 0x10, 7 -> 0x20, 9 -> 0x40，这样进行与操作可以测试多种类型
        static const unsigned char type[] = {
            // ... (省略部分代码)
        };
        // 返回字符对应的类型
        return type[c];
    }

    template <typename InputByteStream>
    static CharType TakeBOM(InputByteStream& is) {
        // 断言输入字节流的字符类型大小为1字节
        RAPIDJSON_STATIC_ASSERT(sizeof(typename InputByteStream::Ch) == 1);
        // 从输入流中获取一个字符
        typename InputByteStream::Ch c = Take(is);
        // 如果获取的字符不是0xEFu，则返回该字符
        if (static_cast<unsigned char>(c) != 0xEFu) return c;
        // 获取下一个字符
        c = is.Take();
        // 如果获取的字符不是0xBBu，则返回该字符
        if (static_cast<unsigned char>(c) != 0xBBu) return c;
        // 获取下一个字符
        c = is.Take();
        // 如果获取的字符不是0xBFu，则返回该字符
        if (static_cast<unsigned char>(c) != 0xBFu) return c;
        // 获取下一个字符
        c = is.Take();
        // 返回获取的字符
        return c;
    }

    template <typename InputByteStream>
    static Ch Take(InputByteStream& is) {
        // 断言输入字节流的字符类型大小为1字节
        RAPIDJSON_STATIC_ASSERT(sizeof(typename InputByteStream::Ch) == 1);
        // 从输入流中获取一个字符，并转换为Ch类型
        return static_cast<Ch>(is.Take());
    }

    template <typename OutputByteStream>
    // 在输出流中添加 BOM（字节顺序标记）
    static void PutBOM(OutputByteStream& os) {
        // 确保输出流的字符类型大小为1字节
        RAPIDJSON_STATIC_ASSERT(sizeof(typename OutputByteStream::Ch) == 1);
        // 写入 BOM 的三个字节
        os.Put(static_cast<typename OutputByteStream::Ch>(0xEFu));
        os.Put(static_cast<typename OutputByteStream::Ch>(0xBBu));
        os.Put(static_cast<typename OutputByteStream::Ch>(0xBFu));
    }

    // 向输出流中写入单个字符
    template <typename OutputByteStream>
    static void Put(OutputByteStream& os, Ch c) {
        // 确保输出流的字符类型大小为1字节
        RAPIDJSON_STATIC_ASSERT(sizeof(typename OutputByteStream::Ch) == 1);
        // 将字符写入输出流
        os.Put(static_cast<typename OutputByteStream::Ch>(c));
    }
};

///////////////////////////////////////////////////////////////////////////////
// UTF16

//! UTF-16 encoding.
/*! http://en.wikipedia.org/wiki/UTF-16
    http://tools.ietf.org/html/rfc2781
    \tparam CharType Type for storing 16-bit UTF-16 data. Default is wchar_t. C++11 may use char16_t instead.
    \note implements Encoding concept

    \note For in-memory access, no need to concern endianness. The code units and code points are represented by CPU's endianness.
    For streaming, use UTF16LE and UTF16BE, which handle endianness.
*/
template<typename CharType = wchar_t>
struct UTF16 {
    typedef CharType Ch;
    RAPIDJSON_STATIC_ASSERT(sizeof(Ch) >= 2);

    enum { supportUnicode = 1 };

    template<typename OutputStream>
    static void Encode(OutputStream& os, unsigned codepoint) {
        RAPIDJSON_STATIC_ASSERT(sizeof(typename OutputStream::Ch) >= 2);
        if (codepoint <= 0xFFFF) {
            RAPIDJSON_ASSERT(codepoint < 0xD800 || codepoint > 0xDFFF); // Code point itself cannot be surrogate pair
            os.Put(static_cast<typename OutputStream::Ch>(codepoint));
        }
        else {
            RAPIDJSON_ASSERT(codepoint <= 0x10FFFF);
            unsigned v = codepoint - 0x10000;
            os.Put(static_cast<typename OutputStream::Ch>((v >> 10) | 0xD800));
            os.Put(static_cast<typename OutputStream::Ch>((v & 0x3FF) | 0xDC00));
        }
    }


    template<typename OutputStream>
    // 将 Unicode 编码的 codepoint 转换成 UTF-16 编码并写入输出流
    static void EncodeUnsafe(OutputStream& os, unsigned codepoint) {
        // 检查输出流的字符类型是否大于等于 2 字节
        RAPIDJSON_STATIC_ASSERT(sizeof(typename OutputStream::Ch) >= 2);
        // 如果 codepoint 小于等于 0xFFFF，则直接写入对应的 UTF-16 编码
        if (codepoint <= 0xFFFF) {
            // 检查 codepoint 是否为代理对的一部分
            RAPIDJSON_ASSERT(codepoint < 0xD800 || codepoint > 0xDFFF); // Code point itself cannot be surrogate pair
            // 将 codepoint 转换成 UTF-16 编码并写入输出流
            PutUnsafe(os, static_cast<typename OutputStream::Ch>(codepoint));
        }
        // 如果 codepoint 大于 0xFFFF，则需要进行代理对编码
        else {
            // 检查 codepoint 是否在有效的 Unicode 范围内
            RAPIDJSON_ASSERT(codepoint <= 0x10FFFF);
            // 计算 codepoint 的代理对编码，并写入输出流
            unsigned v = codepoint - 0x10000;
            PutUnsafe(os, static_cast<typename OutputStream::Ch>((v >> 10) | 0xD800));
            PutUnsafe(os, static_cast<typename OutputStream::Ch>((v & 0x3FF) | 0xDC00));
        }
    }

    // 从输入流中解码 UTF-16 编码的字符，并将结果存储在 codepoint 中
    template <typename InputStream>
    static bool Decode(InputStream& is, unsigned* codepoint) {
        // 检查输入流的字符类型是否大于等于 2 字节
        RAPIDJSON_STATIC_ASSERT(sizeof(typename InputStream::Ch) >= 2);
        // 从输入流中读取一个字符
        typename InputStream::Ch c = is.Take();
        // 如果字符不是代理对的一部分，则直接将其转换成 codepoint
        if (c < 0xD800 || c > 0xDFFF) {
            *codepoint = static_cast<unsigned>(c);
            return true;
        }
        // 如果字符是代理对的一部分，则进行代理对解码
        else if (c <= 0xDBFF) {
            *codepoint = (static_cast<unsigned>(c) & 0x3FF) << 10;
            c = is.Take();
            *codepoint |= (static_cast<unsigned>(c) & 0x3FF);
            *codepoint += 0x10000;
            // 检查第二个字符是否在代理对的有效范围内
            return c >= 0xDC00 && c <= 0xDFFF;
        }
        return false;
    }

    // 验证输入流中的 UTF-16 编码字符是否有效，并将结果写入输出流
    template <typename InputStream, typename OutputStream>
    static bool Validate(InputStream& is, OutputStream& os) {
        // 检查输入流和输出流的字符类型是否大于等于 2 字节
        RAPIDJSON_STATIC_ASSERT(sizeof(typename InputStream::Ch) >= 2);
        RAPIDJSON_STATIC_ASSERT(sizeof(typename OutputStream::Ch) >= 2);
        // 从输入流中读取一个字符，并将其写入输出流
        typename InputStream::Ch c;
        os.Put(static_cast<typename OutputStream::Ch>(c = is.Take()));
        // 如果字符不是代理对的一部分，则认为有效
        if (c < 0xD800 || c > 0xDFFF)
            return true;
        // 如果字符是代理对的一部分，则继续读取第二个字符，并将其写入输出流
        else if (c <= 0xDBFF) {
            os.Put(c = is.Take());
            // 检查第二个字符是否在代理对的有效范围内
            return c >= 0xDC00 && c <= 0xDFFF;
        }
        return false;
    }
};

//! UTF-16 little endian encoding.
// 定义 UTF-16 小端编码的结构体，继承自 UTF16
template<typename CharType = wchar_t>
struct UTF16LE : UTF16<CharType> {
    // 从输入字节流中获取 BOM（字节顺序标记）并返回对应的字符
    template <typename InputByteStream>
    static CharType TakeBOM(InputByteStream& is) {
        // 断言输入字节流的字符类型大小为 1 字节
        RAPIDJSON_STATIC_ASSERT(sizeof(typename InputByteStream::Ch) == 1);
        // 获取字符并判断是否为小端 BOM，是则返回下一个字符，否则返回当前字符
        CharType c = Take(is);
        return static_cast<uint16_t>(c) == 0xFEFFu ? Take(is) : c;
    }

    // 从输入字节流中获取字符并返回对应的字符
    template <typename InputByteStream>
    static CharType Take(InputByteStream& is) {
        // 断言输入字节流的字符类型大小为 1 字节
        RAPIDJSON_STATIC_ASSERT(sizeof(typename InputByteStream::Ch) == 1);
        // 获取两个字节并组合成字符返回
        unsigned c = static_cast<uint8_t>(is.Take());
        c |= static_cast<unsigned>(static_cast<uint8_t>(is.Take())) << 8;
        return static_cast<CharType>(c);
    }

    // 向输出字节流中写入小端 BOM
    template <typename OutputByteStream>
    static void PutBOM(OutputByteStream& os) {
        // 断言输出字节流的字符类型大小为 1 字节
        RAPIDJSON_STATIC_ASSERT(sizeof(typename OutputByteStream::Ch) == 1);
        // 写入小端 BOM
        os.Put(static_cast<typename OutputByteStream::Ch>(0xFFu));
        os.Put(static_cast<typename OutputByteStream::Ch>(0xFEu));
    }

    // 向输出字节流中写入字符
    template <typename OutputByteStream>
    static void Put(OutputByteStream& os, CharType c) {
        // 断言输出字节流的字符类型大小为 1 字节
        RAPIDJSON_STATIC_ASSERT(sizeof(typename OutputByteStream::Ch) == 1);
        // 将字符拆分成两个字节写入输出字节流
        os.Put(static_cast<typename OutputByteStream::Ch>(static_cast<unsigned>(c) & 0xFFu));
        os.Put(static_cast<typename OutputByteStream::Ch>((static_cast<unsigned>(c) >> 8) & 0xFFu));
    }
};

//! UTF-16 big endian encoding.
// 定义 UTF-16 大端编码的结构体，继承自 UTF16
template<typename CharType = wchar_t>
struct UTF16BE : UTF16<CharType> {
    // 从输入字节流中获取 BOM（字节顺序标记）并返回对应的字符
    template <typename InputByteStream>
    static CharType TakeBOM(InputByteStream& is) {
        // 断言输入字节流的字符类型大小为 1 字节
        RAPIDJSON_STATIC_ASSERT(sizeof(typename InputByteStream::Ch) == 1);
        // 获取字符并判断是否为大端 BOM，是则返回下一个字符，否则返回当前字符
        CharType c = Take(is);
        return static_cast<uint16_t>(c) == 0xFEFFu ? Take(is) : c;
    }

    // 从输入字节流中获取字符并返回对应的字符
    template <typename InputByteStream>
    // 从输入字节流中获取一个字符
    static CharType Take(InputByteStream& is) {
        // 确保输入字节流的字符大小为1字节
        RAPIDJSON_STATIC_ASSERT(sizeof(typename InputByteStream::Ch) == 1);
        // 读取两个字节并合并成一个字符
        unsigned c = static_cast<unsigned>(static_cast<uint8_t>(is.Take())) << 8;
        c |= static_cast<unsigned>(static_cast<uint8_t>(is.Take()));
        return static_cast<CharType>(c);
    }

    // 将字节顺序标记（BOM）写入输出字节流
    template <typename OutputByteStream>
    static void PutBOM(OutputByteStream& os) {
        // 确保输出字节流的字符大小为1字节
        RAPIDJSON_STATIC_ASSERT(sizeof(typename OutputByteStream::Ch) == 1);
        // 写入BOM标记
        os.Put(static_cast<typename OutputByteStream::Ch>(0xFEu));
        os.Put(static_cast<typename OutputByteStream::Ch>(0xFFu));
    }

    // 将字符写入输出字节流
    template <typename OutputByteStream>
    static void Put(OutputByteStream& os, CharType c) {
        // 确保输出字节流的字符大小为1字节
        RAPIDJSON_STATIC_ASSERT(sizeof(typename OutputByteStream::Ch) == 1);
        // 将字符拆分成两个字节并写入输出字节流
        os.Put(static_cast<typename OutputByteStream::Ch>((static_cast<unsigned>(c) >> 8) & 0xFFu));
        os.Put(static_cast<typename OutputByteStream::Ch>(static_cast<unsigned>(c) & 0xFFu));
    }
};

///////////////////////////////////////////////////////////////////////////////
// UTF32

//! UTF-32 encoding.
/*! http://en.wikipedia.org/wiki/UTF-32
    \tparam CharType Type for storing 32-bit UTF-32 data. Default is unsigned. C++11 may use char32_t instead.
    \note implements Encoding concept

    \note For in-memory access, no need to concern endianness. The code units and code points are represented by CPU's endianness.
    For streaming, use UTF32LE and UTF32BE, which handle endianness.
*/
template<typename CharType = unsigned>
struct UTF32 {
    typedef CharType Ch;
    RAPIDJSON_STATIC_ASSERT(sizeof(Ch) >= 4);

    enum { supportUnicode = 1 };

    template<typename OutputStream>
    static void Encode(OutputStream& os, unsigned codepoint) {
        RAPIDJSON_STATIC_ASSERT(sizeof(typename OutputStream::Ch) >= 4);
        RAPIDJSON_ASSERT(codepoint <= 0x10FFFF);
        os.Put(codepoint);
    }

    template<typename OutputStream>
    static void EncodeUnsafe(OutputStream& os, unsigned codepoint) {
        RAPIDJSON_STATIC_ASSERT(sizeof(typename OutputStream::Ch) >= 4);
        RAPIDJSON_ASSERT(codepoint <= 0x10FFFF);
        PutUnsafe(os, codepoint);
    }

    template <typename InputStream>
    static bool Decode(InputStream& is, unsigned* codepoint) {
        RAPIDJSON_STATIC_ASSERT(sizeof(typename InputStream::Ch) >= 4);
        Ch c = is.Take();
        *codepoint = c;
        return c <= 0x10FFFF;
    }

    template <typename InputStream, typename OutputStream>
    static bool Validate(InputStream& is, OutputStream& os) {
        RAPIDJSON_STATIC_ASSERT(sizeof(typename InputStream::Ch) >= 4);
        Ch c;
        os.Put(c = is.Take());
        return c <= 0x10FFFF;
    }
};

//! UTF-32 little endian enocoding.
template<typename CharType = unsigned>
struct UTF32LE : UTF32<CharType> {
    template <typename InputByteStream>
    // 从输入字节流中读取字符，处理 UTF-8 BOM（字节顺序标记），返回第一个非 BOM 字符
    static CharType TakeBOM(InputByteStream& is) {
        // 确保输入字节流的字符类型大小为 1 字节
        RAPIDJSON_STATIC_ASSERT(sizeof(typename InputByteStream::Ch) == 1);
        // 读取一个字符
        CharType c = Take(is);
        // 如果读取到的字符是 UTF-8 BOM，则再读取一个字符返回，否则直接返回读取到的字符
        return static_cast<uint32_t>(c) == 0x0000FEFFu ? Take(is) : c;
    }

    // 从输入字节流中读取字符，返回一个 Unicode 字符
    template <typename InputByteStream>
    static CharType Take(InputByteStream& is) {
        // 确保输入字节流的字符类型大小为 1 字节
        RAPIDJSON_STATIC_ASSERT(sizeof(typename InputByteStream::Ch) == 1);
        // 读取一个字节，组成一个 Unicode 字符返回
        unsigned c = static_cast<uint8_t>(is.Take());
        c |= static_cast<unsigned>(static_cast<uint8_t>(is.Take())) << 8;
        c |= static_cast<unsigned>(static_cast<uint8_t>(is.Take())) << 16;
        c |= static_cast<unsigned>(static_cast<uint8_t>(is.Take())) << 24;
        return static_cast<CharType>(c);
    }

    // 向输出字节流中写入 UTF-16 BOM（字节顺序标记）
    template <typename OutputByteStream>
    static void PutBOM(OutputByteStream& os) {
        // 确保输出字节流的字符类型大小为 1 字节
        RAPIDJSON_STATIC_ASSERT(sizeof(typename OutputByteStream::Ch) == 1);
        // 依次写入 UTF-16 BOM 的 4 个字节
        os.Put(static_cast<typename OutputByteStream::Ch>(0xFFu));
        os.Put(static_cast<typename OutputByteStream::Ch>(0xFEu));
        os.Put(static_cast<typename OutputByteStream::Ch>(0x00u));
        os.Put(static_cast<typename OutputByteStream::Ch>(0x00u));
    }

    // 向输出字节流中写入一个 Unicode 字符
    template <typename OutputByteStream>
    static void Put(OutputByteStream& os, CharType c) {
        // 确保输出字节流的字符类型大小为 1 字节
        RAPIDJSON_STATIC_ASSERT(sizeof(typename OutputByteStream::Ch) == 1);
        // 将 Unicode 字符按照 UTF-8 编码规则写入输出字节流
        os.Put(static_cast<typename OutputByteStream::Ch>(c & 0xFFu));
        os.Put(static_cast<typename OutputByteStream::Ch>((c >> 8) & 0xFFu));
        os.Put(static_cast<typename OutputByteStream::Ch>((c >> 16) & 0xFFu));
        os.Put(static_cast<typename OutputByteStream::Ch>((c >> 24) & 0xFFu));
    }
// 结构体模板，表示 UTF-32 大端编码
template<typename CharType = unsigned>
struct UTF32BE : UTF32<CharType> {
    // 从输入字节流中获取字节顺序标记（BOM）
    template <typename InputByteStream>
    static CharType TakeBOM(InputByteStream& is) {
        // 断言输入字节流的字符类型为 1 字节
        RAPIDJSON_STATIC_ASSERT(sizeof(typename InputByteStream::Ch) == 1);
        // 获取 BOM，并返回对应的字符
        CharType c = Take(is);
        return static_cast<uint32_t>(c) == 0x0000FEFFu ? Take(is) : c;
    }

    // 从输入字节流中获取一个字符
    template <typename InputByteStream>
    static CharType Take(InputByteStream& is) {
        // 断言输入字节流的字符类型为 1 字节
        RAPIDJSON_STATIC_ASSERT(sizeof(typename InputByteStream::Ch) == 1);
        // 从输入字节流中获取 4 个字节，组合成一个字符
        unsigned c = static_cast<unsigned>(static_cast<uint8_t>(is.Take())) << 24;
        c |= static_cast<unsigned>(static_cast<uint8_t>(is.Take())) << 16;
        c |= static_cast<unsigned>(static_cast<uint8_t>(is.Take())) << 8;
        c |= static_cast<unsigned>(static_cast<uint8_t>(is.Take()));
        return static_cast<CharType>(c);
    }

    // 向输出字节流中写入 UTF-32 大端编码的 BOM
    template <typename OutputByteStream>
    static void PutBOM(OutputByteStream& os) {
        // 断言输出字节流的字符类型为 1 字节
        RAPIDJSON_STATIC_ASSERT(sizeof(typename OutputByteStream::Ch) == 1);
        // 写入 BOM
        os.Put(static_cast<typename OutputByteStream::Ch>(0x00u));
        os.Put(static_cast<typename OutputByteStream::Ch>(0x00u));
        os.Put(static_cast<typename OutputByteStream::Ch>(0xFEu));
        os.Put(static_cast<typename OutputByteStream::Ch>(0xFFu));
    }

    // 向输出字节流中写入一个 UTF-32 大端编码的字符
    template <typename OutputByteStream>
    static void Put(OutputByteStream& os, CharType c) {
        // 断言输出字节流的字符类型为 1 字节
        RAPIDJSON_STATIC_ASSERT(sizeof(typename OutputByteStream::Ch) == 1);
        // 将字符按大端顺序写入输出字节流
        os.Put(static_cast<typename OutputByteStream::Ch>((c >> 24) & 0xFFu));
        os.Put(static_cast<typename OutputByteStream::Ch>((c >> 16) & 0xFFu));
        os.Put(static_cast<typename OutputByteStream::Ch>((c >> 8) & 0xFFu));
        os.Put(static_cast<typename OutputByteStream::Ch>(c & 0xFFu));
    }
};

// ASCII 编码
//! ASCII 编码
/*! http://en.wikipedia.org/wiki/ASCII
    # 参数说明：用于存储7位ASCII数据的代码单元。默认为char类型。
    # 实现编码概念的注释
template<typename CharType = char>
struct ASCII {
    typedef CharType Ch; // 定义字符类型 Ch

    enum { supportUnicode = 0 }; // 不支持 Unicode

    template<typename OutputStream>
    static void Encode(OutputStream& os, unsigned codepoint) {
        RAPIDJSON_ASSERT(codepoint <= 0x7F); // 断言 codepoint 小于等于 0x7F
        os.Put(static_cast<Ch>(codepoint & 0xFF)); // 将 codepoint 转换为 Ch 类型并写入输出流 os
    }

    template<typename OutputStream>
    static void EncodeUnsafe(OutputStream& os, unsigned codepoint) {
        RAPIDJSON_ASSERT(codepoint <= 0x7F); // 断言 codepoint 小于等于 0x7F
        PutUnsafe(os, static_cast<Ch>(codepoint & 0xFF)); // 调用 PutUnsafe 方法将 codepoint 转换为 Ch 类型并写入输出流 os
    }

    template <typename InputStream>
    static bool Decode(InputStream& is, unsigned* codepoint) {
        uint8_t c = static_cast<uint8_t>(is.Take()); // 从输入流中取出一个字节
        *codepoint = c; // 将取出的字节赋值给 codepoint
        return c <= 0X7F; // 返回是否小于等于 0x7F
    }

    template <typename InputStream, typename OutputStream>
    static bool Validate(InputStream& is, OutputStream& os) {
        uint8_t c = static_cast<uint8_t>(is.Take()); // 从输入流中取出一个字节
        os.Put(static_cast<typename OutputStream::Ch>(c)); // 将取出的字节写入输出流 os
        return c <= 0x7F; // 返回是否小于等于 0x7F
    }

    template <typename InputByteStream>
    static CharType TakeBOM(InputByteStream& is) {
        RAPIDJSON_STATIC_ASSERT(sizeof(typename InputByteStream::Ch) == 1); // 静态断言，输入流的字符类型大小为 1
        uint8_t c = static_cast<uint8_t>(Take(is)); // 从输入流中取出一个字节
        return static_cast<Ch>(c); // 将取出的字节转换为字符类型 Ch
    }

    template <typename InputByteStream>
    static Ch Take(InputByteStream& is) {
        RAPIDJSON_STATIC_ASSERT(sizeof(typename InputByteStream::Ch) == 1); // 静态断言，输入流的字符类型大小为 1
        return static_cast<Ch>(is.Take()); // 从输入流中取出一个字符类型 Ch
    }

    template <typename OutputByteStream>
    static void PutBOM(OutputByteStream& os) {
        RAPIDJSON_STATIC_ASSERT(sizeof(typename OutputByteStream::Ch) == 1); // 静态断言，输出流的字符类型大小为 1
        (void)os; // 忽略输出流 os
    }

    template <typename OutputByteStream>
    static void Put(OutputByteStream& os, Ch c) {
        RAPIDJSON_STATIC_ASSERT(sizeof(typename OutputByteStream::Ch) == 1); // 静态断言，输出流的字符类型大小为 1
        os.Put(static_cast<typename OutputByteStream::Ch>(c)); // 将字符类型 Ch 写入输出流 os
    }
};

///////////////////////////////////////////////////////////////////////////////
// AutoUTF
//! Runtime-specified UTF encoding type of a stream.
enum UTFType {
    kUTF8 = 0,      //!< UTF-8.
    kUTF16LE = 1,   //!< UTF-16 little endian.
    kUTF16BE = 2,   //!< UTF-16 big endian.
    kUTF32LE = 3,   //!< UTF-32 little endian.
    kUTF32BE = 4    //!< UTF-32 big endian.
};

//! Dynamically select encoding according to stream's runtime-specified UTF encoding type.
/*! \note This class can be used with AutoUTFInputtStream and AutoUTFOutputStream, which provides GetType().
*/
template<typename CharType>
struct AutoUTF {
    typedef CharType Ch;

    enum { supportUnicode = 1 };

#define RAPIDJSON_ENCODINGS_FUNC(x) UTF8<Ch>::x, UTF16LE<Ch>::x, UTF16BE<Ch>::x, UTF32LE<Ch>::x, UTF32BE<Ch>::x

    // 编码给定的 codepoint 到输出流中
    template<typename OutputStream>
    static RAPIDJSON_FORCEINLINE void Encode(OutputStream& os, unsigned codepoint) {
        typedef void (*EncodeFunc)(OutputStream&, unsigned);
        static const EncodeFunc f[] = { RAPIDJSON_ENCODINGS_FUNC(Encode) };
        (*f[os.GetType()])(os, codepoint);
    }

    // 不安全地编码给定的 codepoint 到输出流中
    template<typename OutputStream>
    static RAPIDJSON_FORCEINLINE void EncodeUnsafe(OutputStream& os, unsigned codepoint) {
        typedef void (*EncodeFunc)(OutputStream&, unsigned);
        static const EncodeFunc f[] = { RAPIDJSON_ENCODINGS_FUNC(EncodeUnsafe) };
        (*f[os.GetType()])(os, codepoint);
    }

    // 从输入流中解码出一个 codepoint
    template <typename InputStream>
    static RAPIDJSON_FORCEINLINE bool Decode(InputStream& is, unsigned* codepoint) {
        typedef bool (*DecodeFunc)(InputStream&, unsigned*);
        static const DecodeFunc f[] = { RAPIDJSON_ENCODINGS_FUNC(Decode) };
        return (*f[is.GetType()])(is, codepoint);
    }

    // 验证输入流和输出流的编码是否匹配
    template <typename InputStream, typename OutputStream>
    static RAPIDJSON_FORCEINLINE bool Validate(InputStream& is, OutputStream& os) {
        typedef bool (*ValidateFunc)(InputStream&, OutputStream&);
        static const ValidateFunc f[] = { RAPIDJSON_ENCODINGS_FUNC(Validate) };
        return (*f[is.GetType()])(is, os);
    }
}
// 定义了一个模板结构体 Transcoder，用于进行编码转换
template<typename SourceEncoding, typename TargetEncoding>
struct Transcoder {
    // 从源编码中取出一个 Unicode 代码点，将其转换为目标编码并放入输出流
    template<typename InputStream, typename OutputStream>
    static RAPIDJSON_FORCEINLINE bool Transcode(InputStream& is, OutputStream& os) {
        unsigned codepoint;
        // 如果无法从源编码中解码出代码点，则返回 false
        if (!SourceEncoding::Decode(is, &codepoint))
            return false;
        // 将代码点编码为目标编码并放入输出流
        TargetEncoding::Encode(os, codepoint);
        return true;
    }

    // 从源编码中取出一个 Unicode 代码点，将其转换为目标编码并放入输出流（不安全版本）
    template<typename InputStream, typename OutputStream>
    static RAPIDJSON_FORCEINLINE bool TranscodeUnsafe(InputStream& is, OutputStream& os) {
        unsigned codepoint;
        // 如果无法从源编码中解码出代码点，则返回 false
        if (!SourceEncoding::Decode(is, &codepoint))
            return false;
        // 将代码点编码为目标编码并放入输出流（不安全版本）
        TargetEncoding::EncodeUnsafe(os, codepoint);
        return true;
    }

    // 验证从编码流中取出一个 Unicode 代码点
    template<typename InputStream, typename OutputStream>
    static RAPIDJSON_FORCEINLINE bool Validate(InputStream& is, OutputStream& os) {
        return Transcode(is, os);   // 由于源/目标编码不同，必须进行编码转换
    }
};

// 前向声明
template<typename Stream>
inline void PutUnsafe(Stream& stream, typename Stream::Ch c);

// 用相同的源编码和目标编码进行特化的 Transcoder 结构体
template<typename Encoding>
struct Transcoder<Encoding, Encoding> {
    // 从源编码中取出一个 Unicode 代码点，将其直接放入目标编码的输出流
    template<typename InputStream, typename OutputStream>
    static RAPIDJSON_FORCEINLINE bool Transcode(InputStream& is, OutputStream& os) {
        os.Put(is.Take());  // 只是复制一个代码单元。这个语义与主模板类不同。
        return true;
    }

    template<typename InputStream, typename OutputStream>
    // 使用不安全的方式将输入流中的数据转码到输出流中
    static RAPIDJSON_FORCEINLINE bool TranscodeUnsafe(InputStream& is, OutputStream& os) {
        // 将输入流中的一个代码单元复制到输出流中，这种语义与主模板类不同
        PutUnsafe(os, is.Take());
        // 返回转码结果为真
        return true;
    }

    // 验证输入流和输出流中的数据是否有效
    template<typename InputStream, typename OutputStream>
    static RAPIDJSON_FORCEINLINE bool Validate(InputStream& is, OutputStream& os) {
        // 调用编码类的验证函数，验证源编码和目标编码是否相同
        return Encoding::Validate(is, os);
    }
};

RAPIDJSON_NAMESPACE_END

#if defined(__GNUC__) || (defined(_MSC_VER) && !defined(__clang__))
RAPIDJSON_DIAG_POP
#endif

#endif // RAPIDJSON_ENCODINGS_H_
```