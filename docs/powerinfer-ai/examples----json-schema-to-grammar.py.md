# `PowerInfer\examples\json-schema-to-grammar.py`

```
#!/usr/bin/env python3
# 指定脚本的解释器为 Python 3

import argparse
# 导入用于解析命令行参数的模块
import json
# 导入用于处理 JSON 数据的模块
import re
# 导入用于处理正则表达式的模块
import sys
# 导入用于访问与 Python 解释器交互的功能的模块

# whitespace is constrained to a single space char to prevent model "running away" in
# whitespace. Also maybe improves generation quality?
SPACE_RULE = '" "?'
# 定义空白字符规则，限制为一个空格字符，以防止模型在空白字符中“跑偏”。也可能提高生成质量

PRIMITIVE_RULES = {
    'boolean': '("true" | "false") space',
    'number': '("-"? ([0-9] | [1-9] [0-9]*)) ("." [0-9]+)? ([eE] [-+]? [0-9]+)? space',
    'integer': '("-"? ([0-9] | [1-9] [0-9]*)) space',
    'string': r''' "\"" (
        [^"\\] |
        "\\" (["\\/bfnrt] | "u" [0-9a-fA-F] [0-9a-fA-F] [0-9a-fA-F] [0-9a-fA-F])
      )* "\"" space ''',
    'null': '"null" space',
}
# 定义基本规则，包括布尔值、数字、整数、字符串和空值的规则

INVALID_RULE_CHARS_RE = re.compile(r'[^a-zA-Z0-9-]+')
# 编译正则表达式，用于匹配无效的规则字符
GRAMMAR_LITERAL_ESCAPE_RE = re.compile(r'[\r\n"]')
# 编译正则表达式，用于匹配语法文字的转义字符
GRAMMAR_LITERAL_ESCAPES = {'\r': '\\r', '\n': '\\n', '"': '\\"'}
# 定义语法文字的转义字符

class SchemaConverter:
    def __init__(self, prop_order):
        self._prop_order = prop_order
        self._rules = {'space': SPACE_RULE}
        # 初始化 SchemaConverter 类的属性

    def _format_literal(self, literal):
        escaped = GRAMMAR_LITERAL_ESCAPE_RE.sub(
            lambda m: GRAMMAR_LITERAL_ESCAPES.get(m.group(0)), json.dumps(literal)
        )
        return f'"{escaped}"'
    # 定义方法，用于格式化语法文字

    def _add_rule(self, name, rule):
        esc_name = INVALID_RULE_CHARS_RE.sub('-', name)
        if esc_name not in self._rules or self._rules[esc_name] == rule:
            key = esc_name
        else:
            i = 0
            while f'{esc_name}{i}' in self._rules:
                i += 1
            key = f'{esc_name}{i}'
        self._rules[key] = rule
        return key
    # 定义方法，用于添加规则

    def format_grammar(self):
        return '\n'.join((f'{name} ::= {rule}' for name, rule in self._rules.items()))
    # 定义方法，用于格式化语法

def main(args_in = None):
    parser = argparse.ArgumentParser(
        description='''
            Generates a grammar (suitable for use in ./main) that produces JSON conforming to a
            given JSON schema. Only a subset of JSON schema features are supported; more may be
            added in the future.
        ''',
    )
    # 创建解析命令行参数的 ArgumentParser 对象
    # 添加一个命令行参数，用于指定属性的顺序
    parser.add_argument(
        '--prop-order',
        default=[],
        type=lambda s: s.split(','),
        help='''
            逗号分隔的属性名称，定义对象属性的优先顺序；
            没有在这里指定的属性比指定的属性优先级低，并且按字母顺序排序
        '''
    )
    # 添加一个命令行参数，用于指定包含 JSON schema 的文件
    parser.add_argument('schema', help='包含 JSON schema 的文件（“-”表示标准输入）')
    # 解析命令行参数
    args = parser.parse_args(args_in)

    # 从文件中加载 JSON 数据，如果文件名为“-”则从标准输入中加载
    schema = json.load(sys.stdin if args.schema == '-' else open(args.schema))
    # 创建属性顺序的字典，用于后续处理
    prop_order = {name: idx for idx, name in enumerate(args.prop_order)}
    # 创建 SchemaConverter 对象
    converter = SchemaConverter(prop_order)
    # 访问 JSON schema，并进行处理
    converter.visit(schema, '')
    # 打印格式化后的语法
    print(converter.format_grammar())
# 如果当前模块被直接执行，则调用 main() 函数
if __name__ == '__main__':
    main()
```