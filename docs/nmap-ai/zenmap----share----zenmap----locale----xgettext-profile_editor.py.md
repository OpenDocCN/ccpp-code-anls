# `nmap\zenmap\share\zenmap\locale\xgettext-profile_editor.py`

```cpp
#!/usr/bin/env python3

# This program acts like xgettext, specialized to extract strings from Zenmap's
# profile_editor.xml file.

# 导入 getopt 模块，用于解析命令行参数
import getopt
# 导入 os 模块，用于提供与操作系统交互的功能
import os
# 导入 sys 模块，用于提供对 Python 解释器的访问
import sys

# 防止加载 PyXML
# 从 xml 模块中移除包含 "_xmlplus" 的路径
import xml
xml.__path__ = [x for x in xml.__path__ if "_xmlplus" not in x]

# 导入 xml.sax 模块，用于解析 XML 文件
import xml.sax

# 初始化 directory 变量
directory = None

# 定义字符串转义函数
def escape(s):
    return '"' + s.replace('"', '\\"') + '"'

# 输出 msgid 和其定位信息
def output_msgid(msgid, locator):
    print()
    print("#: %s:%d" % (locator.getSystemId(), locator.getLineNumber()))
    print("msgid", escape(msgid))
    print("msgstr", escape(""))

# 定义 XML 内容处理器类
class Handler (xml.sax.handler.ContentHandler):
    # 设置文档定位器
    def setDocumentLocator(self, locator):
        self.locator = locator

    # 处理 XML 元素的开始标签
    def startElement(self, name, attrs):
        if name == "group":
            output_msgid(attrs["name"], self.locator)
        if attrs.get("short_desc"):
            output_msgid(attrs["short_desc"], self.locator)
        if attrs.get("label"):
            output_msgid(attrs["label"], self.locator)

# 解析命令行参数
opts, filenames = getopt.gnu_getopt(sys.argv[1:], "D:", ["directory="])
for o, a in opts:
    if o == "-D" or o == "--directory":
        directory = a

# 如果指定了目录，则切换到该目录
if directory is not None:
    os.chdir(directory)

# 遍历文件名列表
for fn in filenames:
    # 打开文件
    with open(fn, "r") as f:
        # 创建 XML 解析器
        parser = xml.sax.make_parser()
        # 设置内容处理器
        parser.setContentHandler(Handler())
        # 解析 XML 文件
        parser.parse(f)

# 如果文件名列表长度小于 2
if len(filenames) < 2:
    # 创建 XML 解析器
    parser = xml.sax.make_parser()
    # 设置内容处理器
    parser.setContentHandler(Handler())
    # 解析 XML 文件
    parser.parse
```