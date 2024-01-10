# `nmap\zenmap\zenmapCore\ScriptArgsParser.py`

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
# 导入 re 模块，用于正则表达式的匹配和处理
import re
# 定义正则表达式，用于匹配未引用的字符串
unquoted_re = re.compile(r'\s*([^\'"\s{},=][^{},=]*?)\s*([},=]|$)')
# 定义正则表达式，用于匹配引用的字符串
quoted_re = re.compile(r'\s*(([\'"])(.*?[^\\])\2)\s*([},=]|$)')
# 定义正则表达式，用于匹配空字符串
empty_re = re.compile(r'\s*(([\'"])\2)\s*([},=]|$)')

# 定义函数，用于解析单个字符串，可以是引用的、未引用的或空的。返回找到的字符串以及下一个起始位置
def parse_string(s, start):
    """Parses a single string that is quoted, unquoted or empty. It returns the
    found string along with the next starting position """
    # 遍历 unquoted_re, quoted_re, empty_re 这三个正则表达式模式
    for pattern in unquoted_re, quoted_re, empty_re:
        # 尝试使用当前模式匹配字符串 s，从指定位置 start 开始
        m = pattern.match(s, start) or quoted_re.match(s, start)
        # 如果匹配成功，则返回匹配的字符串和结束位置
        if m:
            return m.group(1), m.end(1)
        # 如果没有匹配成功，则抛出 ValueError 异常
        raise ValueError("No string found at %s." % repr(s[start:]))
# 返回字符串中下一个非空字符及其位置
def next_char(s, start):
    while start < len(s) and s[start].isspace():
        start += 1
    if start < len(s):
        return s[start], start
    else:
        return None, start


# 如果从 start 开始的字符串是名称-值对，则返回名称-值元组，否则返回普通字符串
def parse_value(s, start):
    nc, j = next_char(s, start)
    if nc == "{":
        j = parse_table(s, j)
        return s[start:j], j
    else:
        tmp, j = parse_string(s, j)
        nc, j = next_char(s, j)
        if nc == "=":
            # 键/值对？
            j += 1
            begin = j
            nc, j = next_char(s, j)
            if nc == "{":
                j = parse_table(s, j)
            else:
                dummy, j = parse_string(s, j)
            return (tmp, s[begin:j]), j
        else:
            return s[start:j], j


# 负责解析表格的函数；即以 '{' 开头的字符串。返回平衡括号的位置
def parse_table(s, start):
    nc, j = next_char(s, start)
    if not nc or nc != "{":
        raise ValueError("No '{' found at %s." % repr(s[start:]))
    j += 1
    while True:
        nc, j = next_char(s, j)
        if nc == "}":
            # 表格结束
            return j + 1
        else:
            # 用 parse_value 的调用替换这里
            v, j = parse_value(s, j)
            nc, j = next_char(s, j)
            if nc == ",":
                j += 1


# 负责解析脚本参数并将名称-值对存储在列表中的主要函数。如果存在无效参数，则将值存储为 None
def parse_script_args(s):
    args = []
    nc, j = next_char(s, 0)
    # 尝试执行以下代码块，如果出现异常则执行 except 语句块
    try:
        # 当 nc 不为 None 时执行循环
        while nc is not None:
            # 调用 parse_value 函数解析字符串 s 中的值，并返回值和下一个位置
            val, j = parse_value(s, j)
            # 如果值的类型为字符串，抛出数值错误异常
            if type(val) == str:
                raise ValueError(
                        "Only name-value pairs expected in parse_script_args.")
            else:
                # 将值添加到参数列表中
                args.append(val)
            # 获取下一个字符和位置
            nc, j = next_char(s, j)
            # 如果下一个字符为逗号，则增加位置并获取下一个字符
            if nc == ",":
                j += 1
                nc, j = next_char(s, j)
    # 捕获数值错误异常
    except ValueError:
        # 返回空值
        return None
    # 返回参数列表
    return args
def parse_script_args_dict(raw_argument):
    """Wrapper function that copies the name-value pairs from a list into a
    dictionary."""
    # 创建一个空字典
    args_dict = {}
    # 调用parse_script_args函数，将返回的结果存储在args中
    args = parse_script_args(raw_argument)
    # 如果args为None，则直接返回None
    if args is None:
        return None
    # 遍历args列表
    for item in args:
        # 只有长度为2的元素才会被存储到args_dict中
        if(len(item) == 2):  # only key/value pairs are stored
            args_dict[item[0]] = item[1]
    # 返回存储了name-value对的字典
    return args_dict

if __name__ == '__main__':
    # 定义测试用例
    TESTS = (
            ('', []),
            ('a=b,c=d', [('a', 'b'), ('c', 'd')]),
            ('a="b=c"', [('a', '"b=c"')]),
            ('a="def\\"ghi"', [('a', '"def\\"ghi"')]),
            ('a={one,{two,{three}}}', [('a', '{one,{two,{three}}}')]),
            ('a={"quoted}quoted"}', [('a', '{"quoted}quoted"}']),
            ('a="unterminated', None),
            ('user=foo,pass=",{}=bar",whois={whodb=nofollow+ripe},'
                'userdb=C:\\Some\\Path\\To\\File',
                [('user', 'foo'), ('pass', '",{}=bar"'),
                    ('whois', '{whodb=nofollow+ripe}'),
                    ('userdb', 'C:\\Some\\Path\\To\\File')]),
                )

    # 遍历测试用例
    for test, expected in TESTS:
        # 调用parse_script_args_dict函数，将返回的结果存储在args_dict中
        args_dict = parse_script_args_dict(test)
        # 打印args_dict
        print(args_dict)
        # 调用parse_script_args函数，将返回的结果存储在args中
        args = parse_script_args(test)
        # 如果args与期望结果相同，则打印"PASS"和测试用例
        if args == expected:
            print("PASS", test)
            continue
        # 如果args为None，则打印"Parsing error"
        print("FAIL", test)
        if args is None:
            print("Parsing error")
        else:
            # 打印args的长度和每个元素的值
            print("%d args" % len(args))
            for a, v in args:
                print(a, "=", v)
        if expected is None:
            print("Expected parsing error")
        else:
            # 打印期望结果的长度和每个元素的值
            print("Expected %d args" % len(expected))
            for a, v in expected:
                print(a, "=", v)
```