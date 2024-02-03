# `nmap\zenmap\radialnet\core\XMLHandler.py`

```cpp
# 设置文件编码为 UTF-8

# 重要提示：NMAP 许可证条款
# Nmap 安全扫描器由 Nmap 软件有限责任公司（“Nmap 项目”）（C）1996-2023 年开发。Nmap 也是 Nmap 项目的注册商标。
# 本程序根据 Nmap 公共源代码许可证（NPSL）的条款分发。适用于特定 Nmap 版本或源代码控制修订版的确切许可证文本包含在该版本的 Nmap 或源代码控制修订版的 LICENSE 文件中。有关更多 Nmap 版权/法律信息，请访问 https://nmap.org/book/man-legal.html，有关 NPSL 许可证本身的更多信息，请访问 https://nmap.org/npsl/。本标题总结了 Nmap 许可证的一些关键点，但不能替代实际的许可证文本。
# Nmap 通常允许最终用户免费下载和使用，包括商业用途。可从 https://nmap.org 获取。
# Nmap 许可证通常禁止公司在商业产品中使用和重新分发 Nmap，但我们销售一个特殊的 Nmap OEM 版本，具有更宽松的许可证和专门用于此目的的特殊功能。请参阅 https://nmap.org/oem/
# 如果您收到了书面的 Nmap 许可协议或合同，其中规定了与这些条款（如 Nmap OEM 许可证）不同的条款，您可以选择根据那些条款使用和重新分发 Nmap。
# 官方的 Nmap Windows 构建包括 Npcap 软件（https://npcap.com）用于数据包捕获和传输。它受到禁止未经特殊许可证重新分发的单独许可证条款的约束。因此，官方的 Nmap Windows 构建可能不得未经特殊许可证（如 Nmap OEM 许可证）重新分发。
# 我们提供此软件的源代码，因为我们相信用户有权根据自己的需求修改和重新分发它。但是，我们希望用户尊重我们的劳动成果，遵守我们的许可证条款。
# 防止加载 PyXML
# 如果 xml.__path__ 中包含 "_xmlplus"，则将其从列表中移除
import xml
xml.__path__ = [x for x in xml.__path__ if "_xmlplus" not in x]

# 导入 xml.sax 模块
import xml.sax
# 导入 xml.sax.saxutils 模块
import xml.sax.saxutils
# 从 xml.sax.xmlreader 模块中导入 AttributesImpl 类
from xml.sax.xmlreader import AttributesImpl as Attributes

# 定义 XMLNode 类
class XMLNode:
    """
    """
    # 初始化方法，接受一个参数 name
    def __init__(self, name):
        """
        """
        # 将参数 name 赋值给私有属性 __name
        self.__name = name
        # 初始化私有属性 __text 为空字符串
        self.__text = ""
        # 初始化私有属性 __attrs 为空字典
        self.__attrs = dict()
        # 初始化私有属性 __children 为空列表
        self.__children = []

    # 设置文本内容的方法，接受一个参数 text
    def set_text(self, text):
        """
        """
        # 将参数 text 赋值给私有属性 __text
        self.__text = text

    # 获取文本内容的方法
    def get_text(self):
        """
        """
        # 返回私有属性 __text
        return self.__text

    # 设置节点名称的方法，接受一个参数 name
    def set_name(self, name):
        """
        """
        # 将参数 name 赋值给私有属性 __name
        self.__name = name

    # 获取节点名称的方法
    def get_name(self):
        """
        """
        # 返回私有属性 __name
        return self.__name
    # 为当前节点添加属性
    def add_attr(self, key, value):
        self.__attrs[key] = value

    # 为当前节点添加子节点
    def add_child(self, child):
        self.__children.append(child)

    # 获取当前节点属性的键值
    def get_keys(self):
        return self.__attrs.keys()

    # 获取当前节点指定属性的值
    def get_attr(self, attr):
        return self.__attrs.get(attr)

    # 获取当前节点所有属性
    def get_attrs(self):
        return self.__attrs

    # 获取当前节点所有子节点
    def get_children(self):
        return self.__children

    # 查询当前节点的子节点，可以指定节点名、属性名、属性值，是否只返回第一个匹配的节点，是否深度查询
    def query_children(self, name, attr, value, first=False, deep=False):
        result = []

        for child in self.__children:

            if child.get_name() == name:

                if attr in child.get_attrs():

                    c_value = child.get_attr(attr)

                    if c_value == value or c_value == str(value):
                        result.append(child)

            if deep:

                c_result = child.query_children(name, attr, value, first, deep)

                if c_result is not None:

                    if first:
                        return c_result

                    else:
                        result.extend(c_result)

        if first and len(result) > 0:
            return result[0]

        if first:
            return None

        return result
    # 在当前节点的子节点中搜索指定名称的节点
    def search_children(self, name, first=False, deep=False):
        # 初始化结果列表
        result = []

        # 遍历当前节点的子节点
        for child in self.__children:
            # 如果子节点的名称与指定名称相同
            if child.get_name() == name:
                # 将匹配的子节点添加到结果列表中
                result.append(child)
                # 如果指定只返回第一个匹配的子节点，则直接返回结果列表中的第一个节点
                if first:
                    return result[0]

            # 如果指定进行深度搜索
            if deep:
                # 递归调用子节点的搜索方法，获取子节点中匹配的结果
                c_result = child.search_children(name, first, deep)
                # 如果子节点中有匹配的结果且不为空
                if c_result is not None and c_result != []:
                    # 如果指定只返回第一个匹配的子节点，则直接返回子节点中的匹配结果
                    if first:
                        return c_result
                    # 否则将子节点中的匹配结果添加到当前结果列表中
                    else:
                        result.extend(c_result)

        # 如果指定只返回第一个匹配的子节点，则返回空
        if first:
            return None
        # 否则返回所有匹配的子节点
        return result
class XMLWriter(xml.sax.saxutils.XMLGenerator):
    """
    XMLWriter 类继承自 xml.sax.saxutils.XMLGenerator，用于生成 XML 文件
    """
    def __init__(self, file, root=None, encoding="utf-8"):
        """
        初始化 XMLWriter 对象
        """
        xml.sax.saxutils.XMLGenerator.__init__(self, file, encoding)

        self.__root = root

    def set_root(self, root):
        """
        设置 XMLWriter 对象的根节点
        """
        self.__root = root

    def write(self):
        """
        写入 XML 文件
        """
        self.startDocument()
        self.write_xml_node([self.__root])
        self.endDocument()

    def write_xml_node(self, root):
        """
        递归写入 XML 节点
        """
        for child in root:
            self.startElement(child.get_name(), Attributes(child.get_attrs()))

            if child.get_text() != "":
                self.characters(child.get_text())

            self.write_xml_node(child.get_children())

            self.endElement(child.get_name())


class XMLReader(xml.sax.ContentHandler):
    """
    XMLReader 类继承自 xml.sax.ContentHandler，用于解析 XML 文件
    """
    def __init__(self, file=None):
        """
        初始化 XMLReader 对象
        """
        xml.sax.ContentHandler.__init__(self)
        self.__text = ""
        self.__status = []

        self.__file = file
        self.__root = None

        self.__parser = xml.sax.make_parser()
        self.__parser.setContentHandler(self)

    def set_file(self, file, root):
        """
        设置 XMLReader 对象的文件和根节点
        """
        self.__file = file

    def get_file(self):
        """
        获取 XMLReader 对象的文件
        """
        return self.__file

    def get_root(self):
        """
        获取 XMLReader 对象的根节点
        """
        return self.__root

    def parse(self):
        """
        解析 XML 文件
        """
        if self.__file is not None:
            self.__parser.parse(self.__file)

    def startDocument(self):
        """
        开始解析 XML 文档
        """
        pass
    # 定义一个方法，用于处理 XML 元素的开始标签
    def startElement(self, name, attrs):
        # 创建一个新的 XML 节点
        node = XMLNode(name)
    
        # 将属性和值放入节点中
        for attr in attrs.getNames():
            node.add_attr(attr, attrs.get(attr).strip())
    
        # 判断当前节点的父节点是谁
        if len(self.__status) > 0:
            self.__status[-1].add_child(node)
    
        # 如果根节点为空，则将当前节点设置为根节点
        if self.__root is None:
            self.__root = node
    
        # 将当前节点添加到状态列表中
        self.__status.append(node)
    
    # 定义一个方法，用于处理 XML 元素的结束标签
    def endElement(self, name):
        # 设置当前节点的文本内容
        self.__status[-1].set_text(self.__text.strip())
    
        # 清空文本内容
        self.__text = ""
        
        # 移除当前节点
        self.__status.pop()
    
    # 定义一个方法，用于处理 XML 文档的结束
    def endDocument(self):
        # 什么也不做，保留方法以便后续扩展
        pass
    
    # 定义一个方法，用于处理 XML 元素的文本内容
    def characters(self, text):
        # 将文本内容添加到当前节点的文本中
        self.__text += text
# 如果当前模块被直接执行，则执行以下代码
if __name__ == "__main__":

    # 导入 sys 模块
    import sys

    # 创建 XMLReader 对象，传入命令行参数中的第一个参数作为 XML 文件名
    reader = XMLReader(sys.argv[1])
    # 解析 XML 文件
    reader.parse()

    # 获取 XML 文件的根节点
    root = reader.get_root()

    # 创建 XMLWriter 对象，打开 test.xml 文件并传入根节点
    writer = XMLWriter(open("test.xml", 'w'), root)
    # 将根节点写入到 test.xml 文件中
    writer.write()
```