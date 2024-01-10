# `nmap\zenmap\zenmapCore\ScriptMetadata.py`

```
# 指定脚本解释器为 Python 3
#!/usr/bin/env python3

# ***********************IMPORTANT NMAP LICENSE TERMS************************
# *
# * The Nmap Security Scanner is (C) 1996-2023 Nmap Software LLC ("The Nmap
# * Project"). Nmap is also a registered trademark of the Nmap Project.
# *
# * This program is distributed under the terms of the Nmap Public Source
# * License (NPSL). The exact license text applying to a particular Nmap
# * release or source code control revision is contained in the LICENSE
# * file distributed with that version of Nmap or source code control
# * revision. More Nmap copyright/legal information is available from
# * https://nmap.org/book/man-legal.html, and further information on the
# * NPSL license itself can be found at https://nmap.org/npsl/ . This
# * header summarizes some key points from the Nmap license, but is no
# * substitute for the actual license text.
# *
# * Nmap is generally free for end users to download and use themselves,
# * including commercial use. It is available from https://nmap.org.
# *
# * The Nmap license generally prohibits companies from using and
# * redistributing Nmap in commercial products, but we sell a special Nmap
# * OEM Edition with a more permissive license and special features for
# * this purpose. See https://nmap.org/oem/
# *
# * If you have received a written Nmap license agreement or contract
# * stating terms other than these (such as an Nmap OEM license), you may
# * choose to use and redistribute Nmap under those terms instead.
# *
# * The official Nmap Windows builds include the Npcap software
# * (https://npcap.com) for packet capture and transmission. It is under
# * separate license terms which forbid redistribution without special
# * permission. So the official Nmap Windows builds may not be redistributed
# * without special permission (such as an Nmap OEM license).
# *
# * Source is provided to this software because we believe users have a
# * right to know exactly what a program is going to do before they run it.
# 导入正则表达式、操作系统和系统模块
import re
import os
import sys

# 定义一个自定义异常类，用于在脚本中遇到语法错误时引发异常
class ScriptDBSyntaxError(SyntaxError):
    """Exception raised when encountering a syntax error in the script.db"""
    pass

# 定义一个类，用于解析script.db文件，获取脚本名称和类别
class ScriptDB (object):
    """Class responsible for parsing the script.db file, fetching script
    names and categories."""
    # Lua字符串转义字符映射表
    LUA_STRING_ESCAPES = {
        "a": "\a", "b": "\b", "f": "\f", "n": "\n", "r": "\r",
        "t": "\t", "v": "\v", "\\": "\\", "\"": "\"", "'": "'", "0": "\0"
    }
    # 初始化方法，设置初始值和读取脚本数据库路径
    def __init__(self, script_db_path=None):
        # 初始化未使用的缓冲区
        self.unget_buf = ""

        # 初始化行号和行内容
        self.lineno = 1
        self.line = ""
        # 打开脚本数据库文件，并使用上下文管理器
        with open(script_db_path, "r") as self.f:
            # 解析脚本数据库文件内容
            self.entries_list = self.parse()

    # 语法错误处理方法，返回一个脚本数据库语法错误对象
    def syntax_error(self, message):
        e = ScriptDBSyntaxError(message)
        e.filename = self.f.name
        e.lineno = self.lineno
        e.offset = len(self.line)
        e.text = self.line
        return e

    # 获取一个字符
    def getchar(self):
        c = None
        # 如果有未使用的缓冲区，则取出最后一个字符
        if self.unget_buf:
            c = self.unget_buf[-1]
            self.unget_buf = self.unget_buf[:-1]
        else:
            # 否则从文件中读取一个字符
            c = self.f.read(1)
        # 如果字符是换行符，则增加行号并重置行内容
        if c == "\n":
            self.lineno += 1
            self.line = ""
        else:
            # 否则将字符添加到行内容中
            self.line += c
        return c

    # 将数据放回未使用的缓冲区
    def unget(self, data):
        if data:
            # 从行内容中移除相应长度的数据
            self.line = self.line[:-len(data)]
            # 将数据添加到未使用的缓冲区中
            self.unget_buf += data

    # 解析脚本数据库文件内容，返回一个包含所有条目的列表
    def parse(self):
        """Parses a script.db entry and returns it as a dictionary. An entry
        looks like this:
        Entry { filename = "afp-brute.nse", categories = \
                { "auth", "intrusive", } }
        """
        entries = []
        while True:
            # 解析单个条目
            entry = self.parse_entry()
            # 如果没有条目了，则退出循环
            if not entry:
                break
            # 将解析的条目添加到列表中
            entries.append(entry)
        return entries
    # 定义一个方法，返回一个元组，元组的第一个元素是类型（"string"、"ident"或"delim"），第二个元素是标记文本
    def token(self):
        c = self.getchar()  # 获取下一个字符
        while c.isspace():  # 如果字符是空白字符，则继续获取下一个字符
            c = self.getchar()
        if not c:  # 如果字符为空，则返回空
            return None
        if c.isalpha() or c == "_":  # 如果字符是字母或下划线
            ident = []  # 创建一个空列表
            while c.isalpha() or c.isdigit() or c == "_":  # 当字符是字母、数字或下划线时
                ident.append(c)  # 将字符添加到列表中
                c = self.getchar()  # 获取下一个字符
            self.unget(c)  # 将字符放回输入流
            return ("ident", "".join(ident))  # 返回标识符类型和标识符文本
        elif c in "'\"":  # 如果字符是单引号或双引号
            string = []  # 创建一个空列表
            begin_quote = c  # 记录起始引号
            c = self.getchar()  # 获取下一个字符
            while c != begin_quote:  # 当字符不是起始引号时
                if c == "\\":  # 如果字符是反斜杠
                    # 处理转义字符
                    repl = None
                    c = self.getchar()  # 获取下一个字符
                    if not c:  # 如果字符为空
                        raise self.syntax_error("Unexpected EOF")  # 抛出语法错误异常
                    if c.isdigit():  # 如果字符是数字
                        d1 = c
                        d2 = self.getchar()
                        d3 = self.getchar()
                        if d1 and d2 and d3:
                            n = int(d1 + d2 + d3)
                            if n > 255:
                                raise self.syntax_error("Character code >255")
                            repl = chr(n)
                        else:
                            self.unget(d3)
                            self.unget(d2)
                    if not repl:  # 如果没有替换字符
                        repl = self.LUA_STRING_ESCAPES.get(c)  # 获取转义字符对应的替换字符
                    if not repl:  # 如果没有替换字符
                        raise self.syntax_error("Unhandled string escape")  # 抛出语法错误异常
                    c = repl  # 替换字符
                string.append(c)  # 将字符添加到列表中
                c = self.getchar()  # 获取下一个字符
            return ("string", "".join(string))  # 返回字符串类型和字符串文本
        elif c in "{},=":  # 如果字符是大括号、逗号或等号
            return ("delim", c)  # 返回分隔符类型和字符
        else:
            raise self.syntax_error("Unknown token")  # 抛出语法错误异常
    # 定义一个方法，用于验证当前的 token 是否符合预期的 tokens
    def expect(self, tokens):
        for token in tokens:
            t = self.token()
            if t != token:
                raise self.syntax_error(
                        "Unexpected token '%s', expected '%s'" % (
                            t[1], token[1]))

    # 解析一个条目
    def parse_entry(self):
        entry = {}
        token = self.token()
        if not token:
            return None
        # 验证 token 是否符合预期的格式
        self.expect((("delim", "{"), ("ident", "filename"), ("delim", "=")))
        token = self.token()
        if not token or token[0] != "string":
            raise self.syntax_error("Unexpected non-string token or EOF")
        entry["filename"] = token[1]
        # 验证 token 是否符合预期的格式
        self.expect((("delim", ","), ("ident", "categories"),
            ("delim", "="), ("delim", "{")))
        entry["categories"] = []
        token = self.token()
        if token and token[0] == "string":
            entry["categories"].append(token[1])
        token = self.token()
        while token == ("delim", ","):
            token = self.token()
            if token and token[0] == "string":
                entry["categories"].append(token[1])
            else:
                break
            token = self.token()
        if token != ("delim", "}"):
            raise self.syntax_error(
                    "Unexpected token '%s', expected '}'" % (token[1]))
        token = self.token()
        if token == ("delim", ","):
            token = self.token()
        if token != ("delim", "}"):
            raise self.syntax_error(
                    "Unexpected token '%s', expected '}'" % (token[1]))
        return entry

    # 返回条目列表
    def get_entries_list(self):
        return self.entries_list
# 定义一个函数，用于迭代处理 LuaDoc 标签
def nsedoc_tags_iter(f):
    # 初始化变量，用于标记是否在文档注释中，以及当前标签的名称和内容
    in_doc_comment = False
    tag_name = None
    tag_text = None
    # 遍历文件的每一行
    for line in f:
        # 判断是否是新的 LuaDoc 注释
        if re.match(r'^\s*---', line):
            in_doc_comment = True
        # 如果不在注释中，则继续下一行
        if not in_doc_comment:
            continue
        # 判断是否是新的 LuaDoc 标签
        m = re.match(r'^\s*--+\s*@(\w+)\s*(.*)', line, re.S)
        if m:
            # 如果已经有标签，则返回之前的标签和内容
            if tag_name:
                yield tag_name, tag_text
            tag_name = None
            tag_text = None
            # 获取新的标签名称和内容
            tag_name = m.group(1)
            tag_text = m.group(2)
        else:
            # 如果还在注释中
            m = re.match(r'^\s*--+\s*(.*)', line)
            if m:
                # 如果在标签中，则添加到标签内容中
                if tag_name:
                    tag_text += m.group(1) + "\n"
            else:
                # 如果不在注释中，则结束当前标签的处理
                in_doc_comment = False
                if tag_name:
                    yield tag_name, tag_text
                tag_name = None
                tag_text = None

# 定义一个类，用于解析所有脚本信息
class ScriptMetadata (object):
    """Class responsible for parsing all the script information."""

    # 定义一个内部类，用于存储特定脚本的所有信息
    class Entry (object):
        """An instance of this class is used to store all the information
        related to a particular script."""
        def __init__(self, filename):
            # 初始化脚本文件名、类别、参数、许可证、作者、描述、输出和用法
            self.filename = filename
            self.categories = []
            self.arguments = []  # Arguments including library arguments.
            self.license = ""
            self.author = []
            self.description = ""
            self.output = ""
            self.usage = ""

        # 定义一个属性，用于返回脚本的 URL
        url = property(lambda self: "https://nmap.org/nsedoc/scripts/"
                "%s.html" % (os.path.splitext(self.filename)[0]))

    # 初始化函数，用于设置脚本目录和库目录，并构造库参数
    def __init__(self, scripts_dir, nselib_dir):
        self.scripts_dir = scripts_dir
        self.nselib_dir = nselib_dir
        self.library_arguments = {}
        self.library_requires = {}
        self.construct_library_arguments()
    # 获取指定文件的元数据信息
    def get_metadata(self, filename):
        # 创建一个文件条目对象
        entry = self.Entry(filename)
        try:
            # 获取文件描述信息
            entry.description = self.get_string_variable(filename, "description")
            # 获取文件的参数信息
            entry.arguments = self.get_arguments(entry.filename)
            # 获取文件的许可证信息
            entry.license = self.get_string_variable(filename, "license")
            # 获取文件的作者信息
            entry.author = self.get_list_variable(filename, "author") or [
                    self.get_string_variable(filename, "author")]

            # 获取文件的完整路径
            filepath = os.path.join(self.scripts_dir, filename)
            # 打开文件并迭代处理每个标签
            with open(filepath, "r") as f:
                for tag_name, tag_text in nsedoc_tags_iter(f):
                    # 如果标签名为"output"且entry.output为空，则将标签文本赋给entry.output
                    if tag_name == "output" and not entry.output:
                        entry.output = tag_text
                    # 如果标签名为"usage"且entry.usage为空，则将标签文本赋给entry.usage
                    elif tag_name == "usage" and not entry.usage:
                        entry.usage = tag_text
        except IOError as e:
            # 如果发生IO错误，则将错误信息赋给entry.description
            entry.description = "Error getting metadata: {}".format(e)

        # 返回文件条目对象
        return entry

    # 获取指定文件的内容
    @staticmethod
    def get_file_contents(filename):
        # 打开文件并读取内容
        with open(filename, "r") as f:
            contents = f.read()
        # 返回文件内容
        return contents

    # 获取指定文件中指定变量的字符串值
    def get_string_variable(self, filename, varname):
        # 获取文件的内容
        contents = ScriptMetadata.get_file_contents(
            os.path.join(self.scripts_dir, filename))
        # 匹配短字符串
        m = re.search(
            re.escape(varname) + r'\s*=\s*(["\'])(.*?[^\\])\1', contents)
        if m:
            return m.group(2)
        # 匹配长字符串
        m = re.search(
            re.escape(varname) + r'\s*=\s*\[(=*)\[(.*?)\]\1\]', contents, re.S)
        if m:
            return m.group(2)
        # 如果没有匹配到，则返回None
        return None
    # 从文件中获取指定变量的列表值
    def get_list_variable(self, filename, varname):
        # 获取文件内容
        contents = ScriptMetadata.get_file_contents(
            os.path.join(self.scripts_dir, filename))
        # 使用正则表达式搜索变量名及其对应的列表值
        m = re.search(
            re.escape(varname) + r'\s*=\s*\{(.*?)}', contents)
        # 如果没有找到匹配的内容，则返回 None
        if not m:
            return None
        # 提取列表值中的字符串
        strings = m.group(1)
        out = []
        # 使用正则表达式遍历提取字符串，并添加到列表中
        for m in re.finditer(r'(["\'])(.*?[^\\])\1\s*,?', strings, re.S):
            out.append(m.group(2))
        return out

    # 从文件中获取脚本所需的模块列表
    @staticmethod
    def get_requires(filename):
        with open(filename, "r") as f:
            requires = ScriptMetadata.get_requires_from_file(f)
        return requires

    # 从文件对象中获取脚本所需的模块列表
    @staticmethod
    def get_requires_from_file(f):
        # 定义匹配 require 表达式的正则表达式
        require_expr = re.compile(r'.*\brequire\s*\(?([\'\"])([\w._-]+)\1\)?')
        requires = []
        # 遍历文件的每一行，匹配 require 表达式并添加到列表中
        for line in f.readlines():
            m = require_expr.match(line)
            if m:
                requires.append(m.group(2))
        return requires

    # 从文件中获取脚本的参数列表
    @staticmethod
    def get_script_args(filename):
        with open(filename, "r") as f:
            args = ScriptMetadata.get_script_args_from_file(f)
        return args

    # 从文件对象中获取脚本的参数列表
    @staticmethod
    def get_script_args_from_file(f):
        """从给定文件中提取脚本参数列表。结果以 (参数名, 描述) 元组的形式返回。"""
        args = []
        # 使用迭代器遍历文件中的标签和文本
        for tag_name, tag_text in nsedoc_tags_iter(f):
            # 使用正则表达式匹配参数名和描述，并添加到列表中
            m = re.match(r'(\S+)\s+(.*?)', tag_text, re.DOTALL)
            if (tag_name == "arg" or tag_name == "args") and m:
                args.append((m.group(1), m.group(2)))
        return args
    # 根据文件名获取参数列表，包括库参数
    def get_arguments(self, filename):
        """Returns list of arguments including library arguments on
        passing the file name."""
        # 拼接文件路径
        filepath = os.path.join(self.scripts_dir, filename)
        # 获取脚本参数
        script_args = self.get_script_args(filepath)

        # 递归遍历脚本所需的库（以及它们所需的库等），添加所有参数
        library_args = []
        seen = set()
        pool = set(self.get_requires(filepath))
        while pool:
            require = pool.pop()
            if require in seen:
                continue
            seen.add(require)
            sub_requires = self.library_requires.get(require)
            if sub_requires:
                pool.update(set(sub_requires))
            require_args = self.library_arguments.get(require)
            if require_args:
                library_args += require_args

        # 返回脚本参数和库参数的列表
        return script_args + library_args

    # 构建库参数的字典，使用库名称作为键，参数作为值。每个参数实际上是一个（名称，描述）元组。
    def construct_library_arguments(self):
        """Constructs a dictionary of library arguments using library
        names as keys and arguments as values. Each argument is really a
        (name, description) tuple."""
        # 遍历指定目录下的文件
        for filename in os.listdir(self.nselib_dir):
            filepath = os.path.join(self.nselib_dir, filename)
            # 如果不是文件则跳过
            if not os.path.isfile(filepath):
                continue

            base, ext = os.path.splitext(filename)
            if ext == ".lua" or ext == ".luadoc":
                libname = base
            else:
                libname = filename

            # 获取库的脚本参数
            self.library_arguments[libname] = self.get_script_args(filepath)
            # 获取库所需的库
            self.library_requires[libname] = self.get_requires(filepath)
def get_script_entries(scripts_dir, nselib_dir):
    """Merge the information obtained so far into one single entry for
    each script and return it."""
    # 创建脚本元数据对象
    metadata = ScriptMetadata(scripts_dir, nselib_dir)
    try:
        # 尝试从脚本目录中读取脚本数据库
        scriptdb = ScriptDB(os.path.join(scripts_dir, "script.db"))
    except IOError:
        # 如果读取失败，返回空列表
        return []
    # 创建空的条目列表
    entries = []
    # 遍历脚本数据库中的条目
    for dbentry in scriptdb.get_entries_list():
        # 获取条目的元数据
        entry = metadata.get_metadata(dbentry["filename"])
        # Categories 是 ScriptMetadata 没有处理的唯一内容
        entry.categories = dbentry["categories"]
        # 将条目添加到列表中
        entries.append(entry)
    # 返回条目列表
    return entries

if __name__ == '__main__':
    import sys
    # 遍历脚本目录和 nselib 目录中的条目
    for entry in get_script_entries(sys.argv[1], sys.argv[2]):
        # 打印分隔线
        print("*" * 75)
        # 打印文件名
        print("Filename:", entry.filename)
        # 打印分类
        print("Categories:", entry.categories)
        # 打印许可证
        print("License:", entry.license)
        # 打印作者
        print("Author:", entry.author)
        # 打印 URL
        print("URL:", entry.url)
        # 打印描述
        print("Description:", entry.description)
        # 打印参数
        print("Arguments:", [x[0] for x in entry.arguments])
        # 打印输出
        print("Output:")
        print(entry.output)
        # 打印用法
        print("Usage:")
        print(entry.usage)
        # 打印分隔线
        print("*" * 75)
```