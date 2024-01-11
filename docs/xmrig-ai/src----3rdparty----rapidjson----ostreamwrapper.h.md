# `xmrig\src\3rdparty\rapidjson\ostreamwrapper.h`

```
#ifndef RAPIDJSON_OSTREAMWRAPPER_H_
#define RAPIDJSON_OSTREAMWRAPPER_H_

#include "stream.h"
#include <iosfwd>

#ifdef __clang__
RAPIDJSON_DIAG_PUSH
RAPIDJSON_DIAG_OFF(padded)
#endif

RAPIDJSON_NAMESPACE_BEGIN

//! Wrapper of \c std::basic_ostream into RapidJSON's Stream concept.
/*!
    The classes can be wrapped including but not limited to:

    - \c std::ostringstream
    - \c std::stringstream
    - \c std::wpstringstream
    - \c std::wstringstream
    - \c std::ifstream
    - \c std::fstream
    - \c std::wofstream
    - \c std::wfstream

    \tparam StreamType Class derived from \c std::basic_ostream.
*/
template <typename StreamType>
class BasicOStreamWrapper {
public:
    // 定义字符类型为流的字符类型
    typedef typename StreamType::char_type Ch;
    // 构造函数，接受一个流对象的引用
    BasicOStreamWrapper(StreamType& stream) : stream_(stream) {}

    // 向流中写入一个字符
    void Put(Ch c) {
        stream_.put(c);
    }

    // 刷新流
    void Flush() {
        stream_.flush();
    }

    // 以下函数未实现
    char Peek() const { RAPIDJSON_ASSERT(false); return 0; }
    char Take() { RAPIDJSON_ASSERT(false); return 0; }
    size_t Tell() const { RAPIDJSON_ASSERT(false); return 0; }
    char* PutBegin() { RAPIDJSON_ASSERT(false); return 0; }
    size_t PutEnd(char*) { RAPIDJSON_ASSERT(false); return 0; }

private:
    // 禁止拷贝构造函数
    BasicOStreamWrapper(const BasicOStreamWrapper&);
    # 重载赋值运算符，用于将一个 BasicOStreamWrapper 对象的值赋给另一个对象
    BasicOStreamWrapper& operator=(const BasicOStreamWrapper&);

    # 引用类型的成员变量，用于保存流对象
    StreamType& stream_;
};

// 定义基本的输出流包装器，使用 std::ostream
typedef BasicOStreamWrapper<std::ostream> OStreamWrapper;
// 定义基本的输出流包装器，使用 std::wostream
typedef BasicOStreamWrapper<std::wostream> WOStreamWrapper;

// 如果是 Clang 编译器，则恢复之前的诊断设置
#ifdef __clang__
RAPIDJSON_DIAG_POP
#endif

// 结束 RapidJSON 命名空间
RAPIDJSON_NAMESPACE_END

// 结束条件编译，关闭 OStreamWrapper.h 文件
#endif // RAPIDJSON_OSTREAMWRAPPER_H_
```