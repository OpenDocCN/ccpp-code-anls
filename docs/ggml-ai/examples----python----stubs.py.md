# `ggml\examples\python\stubs.py`

```
"""
  This generates .pyi stubs for the cffi Python bindings generated by regenerate.py
"""
# 导入所需的模块
import sys, re, itertools
# 添加路径以便找到所需的模块
sys.path.extend(['.', '..']) # for pycparser

# 导入所需的模块
from pycparser import c_ast, parse_file, CParser
import pycparser.plyparser
from pycparser.c_ast import PtrDecl, TypeDecl, FuncDecl, EllipsisParam, IdentifierType, Struct, Enum, Typedef
from typing import Tuple

# 定义 C 类型到 Python 类型的映射关系
__c_type_to_python_type = {
    'void': 'None', '_Bool': 'bool',
    'char': 'int', 'short': 'int', 'int': 'int', 'long': 'int',
    'ptrdiff_t': 'int', 'size_t': 'int',
    'int8_t': 'int', 'uint8_t': 'int',
    'int16_t': 'int', 'uint16_t': 'int',
    'int32_t': 'int', 'uint32_t': 'int',
    'int64_t': 'int', 'uint64_t': 'int',
    'float': 'float', 'double': 'float',
    'ggml_fp16_t': 'np.float16',
}

# 格式化类型
def format_type(t: TypeDecl):
    if isinstance(t, PtrDecl) or isinstance(t, Struct):
        return 'ffi.CData'
    if isinstance(t, Enum):
        return 'int'
    if isinstance(t, TypeDecl):
        return format_type(t.type)
    if isinstance(t, IdentifierType):
        assert len(t.names) == 1, f'Expected a single name, got {t.names}'
        return __c_type_to_python_type.get(t.names[0]) or 'ffi.CData'
    return t.name

# 定义 PythonStubFuncDeclVisitor 类
class PythonStubFuncDeclVisitor(c_ast.NodeVisitor):
    def __init__(self):
        self.sigs = {}
        self.sources = {}
    # 获取源代码片段的行，以及注释行
    def get_source_snippet_lines(self, coord: pycparser.plyparser.Coord) -> Tuple[list[str], list[str]]:
        # 如果文件不在已缓存的源文件中，则读取文件内容并缓存
        if coord.file not in self.sources:
            with open(coord.file, 'rt') as f:
                self.sources[coord.file] = f.readlines()
        # 获取源文件的所有行
        source_lines = self.sources[coord.file]
        # 计算当前行之前的注释行数
        ncomment_lines = len(list(itertools.takewhile(lambda i: re.search(r'^\s*(//|/\*)', source_lines[i]), range(coord.line - 2, -1, -1))))
        # 获取注释行内容
        comment_lines = [l.strip() for l in source_lines[coord.line - 1 - ncomment_lines:coord.line - 1]]
        # 声明行列表
        decl_lines = []
        # 遍历当前行之后的行，直到遇到分号或左大括号为止
        for line in source_lines[coord.line - 1:]:
            decl_lines.append(line.rstrip())
            if (';' in line) or ('{' in line): break
        # 返回注释行和声明行
        return (comment_lines, decl_lines)

    # 访问枚举类型节点
    def visit_Enum(self, node: Enum):
        # 如果枚举类型有值，则为每个枚举值生成属性
        if node.values is not None:
          for e in node.values.enumerators:
              self.sigs[e.name] = f'  @property\n  def {e.name}(self) -> int: ...'

    # 访问类型定义节点
    def visit_Typedef(self, node: Typedef):
        # 不做任何处理
        pass
    # 定义一个方法，用于访问函数声明节点
    def visit_FuncDecl(self, node: FuncDecl):
        # 获取函数返回类型
        ret_type = node.type
        # 判断返回类型是否为指针类型
        is_ptr = False
        while isinstance(ret_type, PtrDecl):
            ret_type = ret_type.type
            is_ptr = True

        # 获取函数名
        fun_name = ret_type.declname
        # 如果函数名以'__'开头，则直接返回
        if fun_name.startswith('__'):
            return

        # 初始化参数列表和参数名列表
        args = []
        argnames = []
        # 定义一个生成参数名的函数
        def gen_name(stem):
            i = 1
            while True:
                new_name = stem if i == 1 else f'{stem}{i}'
                if new_name not in argnames: return new_name
                i += 1

        # 遍历函数参数列表
        for a in node.args.params:
            # 如果参数是省略号类型，则生成参数名并添加到参数列表中
            if isinstance(a, EllipsisParam):
                arg_name = gen_name('args')
                argnames.append(arg_name)
                args.append('*' + gen_name('args'))
            # 如果参数类型为'None'，则跳过
            elif format_type(a.type) == 'None':
                continue
            # 否则生成参数名并添加到参数列表中
            else:
                arg_name = a.name or gen_name('arg')
                argnames.append(arg_name)
                args.append(f'{arg_name}: {format_type(a.type)}')

        # 获取函数返回类型的格式化字符串
        ret = format_type(ret_type if not is_ptr else node.type)

        # 获取函数声明的注释和代码行
        comment_lines, decl_lines = self.get_source_snippet_lines(node.coord)

        # 构建函数声明的代码行
        lines = [f'  def {fun_name}({", ".join(args)}) -> {ret}:']
        # 如果注释行数为0且声明行数为1，则直接添加声明行作为注释
        if len(comment_lines) == 0 and len(decl_lines) == 1:
            lines += [f'    """{decl_lines[0]}"""']
        # 否则按照注释和声明行分别添加到代码行中
        else:
            lines += ['    """']
            lines += [f'    {c.lstrip("/* ")}' for c in comment_lines]
            if len(comment_lines) > 0:
                lines += ['']
            lines += [f'    {d}' for d in decl_lines]
            lines += ['    """']
        # 添加占位符'...'作为函数体
        lines += ['    ...']
        # 将函数声明代码行添加到函数签名字典中
        self.sigs[fun_name] = '\n'.join(lines)
# 生成一个 .pyi Python stub 文件，用于 GGML API，使用 C 头文件
def generate_stubs(header: str):
    # 创建一个 PythonStubFuncDeclVisitor 对象
    v = PythonStubFuncDeclVisitor()
    # 解析 C 头文件，访问其中的函数声明
    v.visit(CParser().parse(header, "<input>"))

    # 获取函数签名字典的键，并排序
    keys = list(v.sigs.keys())
    keys.sort()

    # 返回拼接后的字符串，包括注释、导入模块、类定义和函数签名
    return '\n'.join([
        '# auto-generated file',
        'import ggml.ffi as ffi',
        'import numpy as np',
        'class lib:',
        *[v.sigs[k] for k in keys]
    ])
```