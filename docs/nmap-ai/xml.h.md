# `nmap\xml.h`

```cpp
/* $Id: xml.h 15135 2009-08-19 21:05:21Z david $ */

#ifndef _XML_H
#define _XML_H

#include <stdarg.h>

// 声明函数 xml_write_raw，接受可变参数，格式化输出原始文本
int xml_write_raw(const char *fmt, ...) __attribute__ ((format (printf, 1, 2)));
// 声明函数 xml_write_escaped，接受可变参数，格式化输出转义后的文本
int xml_write_escaped(const char *fmt, ...) __attribute__ ((format (printf, 1, 2)));
// 声明函数 xml_write_escaped_v，接受可变参数列表，格式化输出转义后的文本
int xml_write_escaped_v(const char *fmt, va_list va) __attribute__ ((format (printf, 1, 0)));

// 声明函数 xml_start_document，开始一个 XML 文档
int xml_start_document(const char *rootnode);

// 声明函数 xml_start_comment，开始一个 XML 注释
int xml_start_comment();
// 声明函数 xml_end_comment，结束一个 XML 注释
int xml_end_comment();

// 声明函数 xml_open_pi，打开一个处理指令
int xml_open_pi(const char *name);
// 声明函数 xml_close_pi，关闭一个处理指令
int xml_close_pi();

// 声明函数 xml_open_start_tag，打开一个起始标签
int xml_open_start_tag(const char *name, const bool write = true);
// 声明函数 xml_close_start_tag，关闭一个起始标签
int xml_close_start_tag(const bool write = true);
// 声明函数 xml_close_empty_tag，关闭一个空标签
int xml_close_empty_tag();
// 声明函数 xml_start_tag，开始一个标签
int xml_start_tag(const char *name, const bool write = true);
// 声明函数 xml_end_tag，结束一个标签
int xml_end_tag();

// 声明函数 xml_attribute，为标签添加属性
int xml_attribute(const char *name, const char *fmt, ...) __attribute__ ((format (printf, 2, 3)));

// 声明函数 xml_newline，输出换行符
int xml_newline();

// 声明函数 xml_depth，返回当前 XML 标签的深度
int xml_depth();
// 声明函数 xml_tag_open，检查当前是否有标签处于打开状态
bool xml_tag_open();
// 声明函数 xml_root_written，检查 XML 根节点是否已经被写入
bool xml_root_written();

// 声明函数 xml_unescape，对 XML 转义字符进行反转义
char *xml_unescape(const char *str);

#endif
```