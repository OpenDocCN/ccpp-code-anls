# `PowerInfer\examples\json-schema-to-grammar.py`

```
#!/usr/bin/env python3
# 指定脚本解释器为 Python 3

import argparse
# 导入用于解析命令行参数的模块
import json
# 导入用于处理 JSON 数据的模块
import re
# 导入用于正则表达式操作的模块
import sys
# 导入用于访问与 Python 解释器交互的变量和函数的模块

# whitespace is constrained to a single space char to prevent model "running away" in
# whitespace. Also maybe improves generation quality?
# 定义空白字符规则，限制为单个空格字符，以防止模型在空白字符中“跑偏”。也可能提高生成质量？
SPACE_RULE = '" "?'

PRIMITIVE_RULES = {
    'boolean': '("true" | "false") space',
    # 定义布尔类型的规则
    'number': '("-"? ([0-9] | [1-9] [0-9]*)) ("." [0-9]+)? ([eE] [-+]? [0-9]+)? space',
    # 定义数字类型的规则
    'integer': '("-"? ([0-9] | [1-9] [0-9]*)) space',
    # 定义整数类型的规则
    'string': r''' "\"" (
        [^"\\] |
        "\\" (["\\/bfnrt] | "u" [0-9a-fA-F] [0-9a-fA-F] [0-9a-fA-F] [0-9a-fA-F])
      )* "\"" space ''',
    # 定义字符串类型的规则，使用原始字符串表示法
    'null': '"null" space',
    # 定义空值类型的规则
}
# 使用正则表达式匹配非字母、数字和破折号的字符
INVALID_RULE_CHARS_RE = re.compile(r'[^a-zA-Z0-9-]+')

# 使用正则表达式匹配换行符、回车符和双引号
GRAMMAR_LITERAL_ESCAPE_RE = re.compile(r'[\r\n"]')

# 定义需要转义的字符和对应的转义序列
GRAMMAR_LITERAL_ESCAPES = {'\r': '\\r', '\n': '\\n', '"': '\\"'}

# 定义 SchemaConverter 类
class SchemaConverter:
    def __init__(self, prop_order):
        # 初始化属性顺序
        self._prop_order = prop_order
        # 初始化规则字典，包含一个名为'space'的规则
        self._rules = {'space': SPACE_RULE}

    # 格式化文本字面量
    def _format_literal(self, literal):
        # 使用正则表达式替换特定字符，并转义文本字面量
        escaped = GRAMMAR_LITERAL_ESCAPE_RE.sub(
            lambda m: GRAMMAR_LITERAL_ESCAPES.get(m.group(0)), json.dumps(literal)
        )
        return f'"{escaped}"'

    # 添加规则到规则字典
    def _add_rule(self, name, rule):
        # 将规则名中的非法字符替换为破折号
        esc_name = INVALID_RULE_CHARS_RE.sub('-', name)
        # 如果规则名不在规则字典中，或者规则字典中的规则与当前规则相同
# 如果给定的名称未被转义，则直接使用该名称作为键
        if not self._is_escaped(esc_name):
            key = esc_name
        # 如果给定的名称已经被转义，则在名称后面添加数字直到找到一个未被使用的键
        else:
            i = 0
            while f'{esc_name}{i}' in self._rules:
                i += 1
            key = f'{esc_name}{i}'
        # 将规则添加到规则字典中，并返回键
        self._rules[key] = rule
        return key

    # 访问给定的模式，并为其生成规则
    def visit(self, schema, name):
        # 获取模式的类型
        schema_type = schema.get('type')
        # 获取规则的名称，如果没有指定名称，则使用'root'
        rule_name = name or 'root'

        # 如果模式中包含'oneOf'或'anyOf'关键字，则生成对应的规则
        if 'oneOf' in schema or 'anyOf' in schema:
            # 生成多个备选规则，并使用'|'连接起来
            rule = ' | '.join((
                self.visit(alt_schema, f'{name}{"-" if name else ""}{i}')
                for i, alt_schema in enumerate(schema.get('oneOf') or schema['anyOf'])
            ))
            # 将生成的规则添加到规则字典中，并返回规则名称
            return self._add_rule(rule_name, rule)
        # 如果 schema 中包含 'const' 关键字
        elif 'const' in schema:
            # 将 rule_name 和格式化后的 'const' 值添加到规则中
            return self._add_rule(rule_name, self._format_literal(schema['const']))

        # 如果 schema 中包含 'enum' 关键字
        elif 'enum' in schema:
            # 将 'enum' 中的值格式化后用 '|' 连接起来，添加到规则中
            rule = ' | '.join((self._format_literal(v) for v in schema['enum']))
            return self._add_rule(rule_name, rule)

        # 如果 schema 类型为 'object' 并且包含 'properties' 关键字
        elif schema_type == 'object' and 'properties' in schema:
            # TODO: `required` keyword
            # 获取属性的顺序
            prop_order = self._prop_order
            # 对属性进行排序，先按照 prop_order 中的位置排序，然后按照属性名排序
            prop_pairs = sorted(
                schema['properties'].items(),
                key=lambda kv: (prop_order.get(kv[0], len(prop_order)), kv[0]),
            )

            # 构建规则字符串
            rule = '"{" space'
            for i, (prop_name, prop_schema) in enumerate(prop_pairs):
                # 访问属性的 schema，生成属性的规则名
                prop_rule_name = self.visit(prop_schema, f'{name}{"-" if name else ""}{prop_name}')
                if i > 0:
# 如果 schema_type 是 object 并且 schema 中有 properties，则生成对象规则
if schema_type == 'object' and 'properties' in schema:
    # 遍历 properties 中的属性，生成属性规则
    for prop_name, prop_schema in schema['properties'].items():
        # 如果 rule 已经有内容，则添加逗号和空格
        if rule:
            rule += ' "," space'
        # 添加属性规则，格式为 "属性名" : 属性规则名
        rule += fr' {self._format_literal(prop_name)} space ":" space {prop_rule_name}'
    # 添加对象结束的规则
    rule += ' "}" space'
    # 将规则添加到语法中
    return self._add_rule(rule_name, rule)

# 如果 schema_type 是 array 并且 schema 中有 items，则生成数组规则
elif schema_type == 'array' and 'items' in schema:
    # TODO `prefixItems` keyword
    # 生成数组项的规则
    item_rule_name = self.visit(schema['items'], f'{name}{"-" if name else ""}item')
    # 生成数组规则，格式为 "[ 数组项 (, 数组项)* ]"
    rule = f'"[" space ({item_rule_name} ("," space {item_rule_name})*)? "]" space'
    # 将规则添加到语法中
    return self._add_rule(rule_name, rule)

# 如果以上条件都不满足，且 schema_type 是基本类型，则生成基本类型规则
else:
    # 断言 schema_type 在基本规则中，如果不在则抛出异常
    assert schema_type in PRIMITIVE_RULES, f'Unrecognized schema: {schema}'
    # 将基本类型规则添加到语法中
    return self._add_rule(
        'root' if rule_name == 'root' else schema_type,
        PRIMITIVE_RULES[schema_type]
    )

# 定义格式化语法的方法
def format_grammar(self):
# 将规则字典中的每个键值对格式化为字符串，并用换行符连接起来
return '\n'.join((f'{name} ::= {rule}' for name, rule in self._rules.items()))

# 主函数，用于解析命令行参数
def main(args_in = None):
    # 创建参数解析器对象
    parser = argparse.ArgumentParser(
        description='''
            生成一个符合给定 JSON 模式的 JSON 语法（适用于 ./main）。仅支持 JSON 模式的部分特性；将来可能会添加更多特性。
        ''',
    )
    # 添加命令行参数
    parser.add_argument(
        '--prop-order',
        default=[],
        type=lambda s: s.split(','),
        help='''
            逗号分隔的属性名称，定义对象属性的优先顺序；未在此处指定的属性优先级低于已指定的属性，并按字母顺序排序
        '''
    )
    # 添加一个参数，用于指定包含 JSON schema 的文件，如果为“-”则表示从标准输入读取
    parser.add_argument('schema', help='file containing JSON schema ("-" for stdin)')
    # 解析命令行参数
    args = parser.parse_args(args_in)

    # 从文件中加载 JSON 数据，如果参数为“-”则从标准输入读取
    schema = json.load(sys.stdin if args.schema == '-' else open(args.schema))
    # 创建属性顺序的字典，用于指定属性的顺序
    prop_order = {name: idx for idx, name in enumerate(args.prop_order)}
    # 创建 SchemaConverter 对象
    converter = SchemaConverter(prop_order)
    # 访问 JSON schema 并转换为特定格式的语法
    converter.visit(schema, '')
    # 打印转换后的语法
    print(converter.format_grammar())


if __name__ == '__main__':
    # 调用主函数
    main()
```