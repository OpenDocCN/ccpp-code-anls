# `xmrig\src\3rdparty\rapidjson\fwd.h`

```cpp
#ifndef RAPIDJSON_FWD_H_
#define RAPIDJSON_FWD_H_

#include "rapidjson.h"

RAPIDJSON_NAMESPACE_BEGIN

// 前向声明不同编码类型的结构体模板
template<typename CharType> struct UTF8;
template<typename CharType> struct UTF16;
template<typename CharType> struct UTF16BE;
template<typename CharType> struct UTF16LE;
template<typename CharType> struct UTF32;
template<typename CharType> struct UTF32BE;
template<typename CharType> struct UTF32LE;
template<typename CharType> struct ASCII;
template<typename CharType> struct AutoUTF;

// 前向声明编码转换器模板
template<typename SourceEncoding, typename TargetEncoding>
struct Transcoder;

// 前向声明内存分配器类
class CrtAllocator;

// 前向声明内存池分配器模板
template <typename BaseAllocator>
class MemoryPoolAllocator;

// 前向声明通用字符串流模板
template <typename Encoding>
struct GenericStringStream;

// 使用UTF8编码的通用字符串流类型
typedef GenericStringStream<UTF8<char> > StringStream;

// 前向声明通用原地字符串流模板
template <typename Encoding>
struct GenericInsituStringStream;

// 使用UTF8编码的通用原地字符串流类型
typedef GenericInsituStringStream<UTF8<char> > InsituStringStream;

// 前向声明通用字符串缓冲区模板
template <typename Encoding, typename Allocator>
class GenericStringBuffer;

// 使用UTF8编码和CrtAllocator分配器的通用字符串缓冲区类型
typedef GenericStringBuffer<UTF8<char>, CrtAllocator> StringBuffer;

// 前向声明文件读取流类
class FileReadStream;

// 前向声明文件写入流类
class FileWriteStream;

// 前向声明通用内存缓冲区模板
template <typename Allocator>
struct GenericMemoryBuffer;

#endif
// 定义使用 CrtAllocator 的通用内存缓冲区
typedef GenericMemoryBuffer<CrtAllocator> MemoryBuffer;

// 声明 MemoryStream 结构体
struct MemoryStream;

// 声明 BaseReaderHandler 模板，包括编码和派生类型
template<typename Encoding, typename Derived>
struct BaseReaderHandler;

// 声明 GenericReader 模板，包括源编码、目标编码、堆栈分配器
template <typename SourceEncoding, typename TargetEncoding, typename StackAllocator>
class GenericReader;

// 定义 Reader 类型为使用 UTF8 编码和 CrtAllocator 的 GenericReader
typedef GenericReader<UTF8<char>, UTF8<char>, CrtAllocator> Reader;

// 声明 Writer 模板，包括输出流、源编码、目标编码、堆栈分配器和写入标志
template<typename OutputStream, typename SourceEncoding, typename TargetEncoding, typename StackAllocator, unsigned writeFlags>
class Writer;

// 声明 PrettyWriter 模板，包括输出流、源编码、目标编码、堆栈分配器和写入标志
template<typename OutputStream, typename SourceEncoding, typename TargetEncoding, typename StackAllocator, unsigned writeFlags>
class PrettyWriter;

// 声明 GenericMember 类模板，包括编码和分配器
template <typename Encoding, typename Allocator>
class GenericMember;

// 声明 GenericMemberIterator 类模板，包括是否为常量、编码和分配器
template <bool Const, typename Encoding, typename Allocator>
class GenericMemberIterator;

// 声明 GenericStringRef 结构体模板，包括字符类型
template<typename CharType>
struct GenericStringRef;

// 声明 GenericValue 类模板，包括编码和分配器
template <typename Encoding, typename Allocator>
class GenericValue;

// 定义 Value 类型为使用 UTF8 编码和 MemoryPoolAllocator<CrtAllocator> 分配器的 GenericValue
typedef GenericValue<UTF8<char>, MemoryPoolAllocator<CrtAllocator> > Value;

// 声明 GenericDocument 类模板，包括编码、分配器和堆栈分配器
template <typename Encoding, typename Allocator, typename StackAllocator>
class GenericDocument;

// 定义 Document 类型为使用 UTF8 编码、MemoryPoolAllocator<CrtAllocator> 分配器和 CrtAllocator 堆栈分配器的 GenericDocument
typedef GenericDocument<UTF8<char>, MemoryPoolAllocator<CrtAllocator>, CrtAllocator> Document;

// 声明 GenericPointer 类模板，包括值类型和分配器
template <typename ValueType, typename Allocator>
class GenericPointer;

// 定义 Pointer 类型为使用 Value 值类型和 CrtAllocator 分配器的 GenericPointer
typedef GenericPointer<Value, CrtAllocator> Pointer;

// 声明 IGenericRemoteSchemaDocumentProvider 类模板，包括模式文档类型
template <typename SchemaDocumentType>
class IGenericRemoteSchemaDocumentProvider;

// 声明 GenericSchemaDocument 类模板，包括值类型和分配器
template <typename ValueT, typename Allocator>
class GenericSchemaDocument;

// 定义 SchemaDocument 类型为使用 Value 值类型和 CrtAllocator 分配器的 GenericSchemaDocument
typedef GenericSchemaDocument<Value, CrtAllocator> SchemaDocument;

// 定义 IRemoteSchemaDocumentProvider 类型为使用 SchemaDocument 模式文档类型的 IGenericRemoteSchemaDocumentProvider
typedef IGenericRemoteSchemaDocumentProvider<SchemaDocument> IRemoteSchemaDocumentProvider;

// 声明 GenericSchemaValidator 类模板，包括模式文档类型、输出处理程序、状态分配器
template <
    typename SchemaDocumentType,
    typename OutputHandler,
    typename StateAllocator>
class GenericSchemaValidator;

// 定义 SchemaValidator 类型为使用 SchemaDocument 模式文档类型、BaseReaderHandler<UTF8<char>, void> 输出处理程序和 CrtAllocator 状态分配器的 GenericSchemaValidator
typedef GenericSchemaValidator<SchemaDocument, BaseReaderHandler<UTF8<char>, void>, CrtAllocator> SchemaValidator;
// 结束 RapidJSON 命名空间
RAPIDJSON_NAMESPACE_END

// 结束 RapidJSONFWD.h 文件的条件编译
#endif // RAPIDJSON_RAPIDJSONFWD_H_
```