# `nmap\zenmap\zenmapCore\NmapOptions.py`

```
#!/usr/bin/env python3

# This is an Nmap command line parser. It has two main parts:
#
#   getopt_long_only_extras, which is like getopt_long_only with robust
#   handling of unknown options.
#
#   NmapOptions, a class representing a set of Nmap options.
#
# NmapOptions is the class for external use. NmapOptions.parse parses a list of
# a command followed by command-line arguments. NmapOptions.render returns a
# list of of a command followed by arguments. NmapOptions.parse_string and
# NmapOptions.render_string first split strings into lists, following certain
# quoting rules.
#
# >>> ops = NmapOptions()
# >>> ops.parse(["nmap", "-v", "--script", "safe", "localhost"])
# >>> ops.executable
# 'nmap'
# >>> ops.target_specs
# ['localhost']
# >>> ops["-v"]
# 1
# >>> ops["--script"]
# 'safe'
#
# The command line may be modified by accessing member variables:
#
# >>> ops.executable = "C:\Program Files\Nmap\nmap.exe"
# >>> ops["-v"] = 2
# >>> ops["-oX"] = "output.xml"
# >>> ops.render()
# ['C:\\Program Files\\Nmap\\nmap.exe', '-v', '-v', '-oX', 'output.xml',
#  '--script', 'safe', 'localhost']
# >>> ops.render_string()
# '"C:\\Program Files\\Nmap\\nmap.exe" -v -v -oX output.xml\
#  --script safe localhost'
#
# A primary design consideration was robust handling of unknown options. That
# gives this code a degree of independence from Nmap's own list of options. If
# an option is added to Nmap but not added here, that option is treated as an
# "extra," an uninterpreted string that is inserted verbatim into the option
# list. Because the unknown option may or may not take an argument, pains are
# taken to avoid interpreting any option ambiguously.
#
# Consider the following case, where -x is an unknown option:
#   nmap -x -e eth0 scanme.nmap.org
# If -x, whatever it is, does not take an argument, it is equivalent to
#   nmap -e eth0 scanme.nmap.org -x
# that is, a scan of scanme.nmap.org over interface eth0. But if it does take
# 导入 functools 模块中的 reduce 函数
from functools import reduce

# 定义一个名为 option 的类
class option:
    """A single option, part of a pool of potential options. It's just a name
    and a flag saying if the option takes no argument, if an argument is
    optional, or if an argument is required."""
    # 定义类属性 NO_ARGUMENT，表示选项不需要参数
    NO_ARGUMENT = 0
    # 定义常量，表示参数是否必需
    REQUIRED_ARGUMENT = 1
    OPTIONAL_ARGUMENT = 2
    
    # 初始化方法，接受参数名和参数是否必需作为参数
    def __init__(self, name, has_arg):
        # 将参数名赋给实例变量
        self.name = name
        # 将参数是否必需的标志赋给实例变量
        self.has_arg = has_arg
def split_quoted(s):
    """Like str.split, except that no splits occur inside quoted strings, and
    quoted strings are unquoted."""
    # 初始化结果列表
    r = []
    # 初始化索引
    i = 0
    # 跳过字符串开头的空白字符
    while i < len(s) and s[i].isspace():
        i += 1
    # 遍历字符串
    while i < len(s):
        # 初始化部分列表
        part = []
        # 遍历非空白字符
        while i < len(s) and not s[i].isspace():
            c = s[i]
            # 如果是引号，则处理引号内的内容
            if c == "\"" or c == "'":
                begin = c
                i += 1
                while i < len(s):
                    c = s[i]
                    # 如果是结束引号，则跳出循环
                    if c == begin:
                        i += 1
                        break
                    # 如果是转义字符，则处理转义字符
                    elif c == "\\":
                        i += 1
                        if i < len(s):
                            c = s[i]
                        # 否则，忽略错误并将反斜杠留在字符串末尾
                    part.append(c)
                    i += 1
            else:
                part.append(c)
                i += 1
        # 将部分列表转换为字符串并添加到结果列表中
        r.append("".join(part))
        # 跳过字符串末尾的空白字符
        while i < len(s) and s[i].isspace():
            i += 1
    # 返回结果列表
    return r


def maybe_quote(s):
    """Return s quoted if it needs to be, otherwise unchanged."""
    # 检查字符串是否需要引号
    for c in s:
        if c == "\"" or c == "\\" or c == "'" or c.isspace():
            break
    else:
        return s
    # 处理需要引号的情况
    r = []
    for c in s:
        if c == "\"":
            r.append("\\\"")
        elif c == "\\":
            r.append("\\\\")
        else:
            r.append(c)
    # 返回处理后的字符串
    return "\"" + "".join(r) + "\""


def join_quoted(l):
    # 将列表中的元素转换为带引号的字符串并用空格连接
    return " ".join([maybe_quote(x) for x in l])


def make_options(short_opts, long_opts):
    """Parse a short option specification string and long option tuples into a
    list of option objects."""
    # 初始化选项列表
    options = []
    # 遍历长选项
    for name, has_arg in long_opts:
        # 将长选项转换为选项对象并添加到选项列表中
        options.append(option(name, has_arg))
    # 当短选项列表长度大于0时，进入循环
    while len(short_opts) > 0:
        # 获取短选项列表的第一个选项
        name = short_opts[0]
        # 将获取的选项从短选项列表中移除
        short_opts = short_opts[1:]
        # 断言选项名称不为冒号
        assert name != ":"
        # 初始化冒号数量
        num_colons = 0
        # 当短选项列表长度大于0且第一个选项为冒号时，进入循环
        while len(short_opts) > 0 and short_opts[0] == ":":
            # 将冒号选项从短选项列表中移除
            short_opts = short_opts[1:]
            # 冒号数量加1
            num_colons += 1
        # 根据冒号数量确定选项是否需要参数
        if num_colons == 0:
            has_arg = option.NO_ARGUMENT
        elif num_colons == 1:
            has_arg = option.REQUIRED_ARGUMENT
        else:
            has_arg = option.OPTIONAL_ARGUMENT
        # 将选项和参数需求添加到选项列表中
        options.append(option(name, has_arg))

    # 返回选项列表
    return options
# 创建一个空的字典，用于缓存选项查找结果
lookup_option_cache = {}

# 查找具有给定（可能是缩写的）名称的选项。如果没有匹配的选项或名称模糊（有多个选项匹配但没有完全匹配），则返回None。
def lookup_option(name, options):
    # 这个函数是一个巨大的瓶颈。因此我们对其进行了记忆化处理。
    # 我们使用选项名称和选项列表的id进行哈希，因为列表是不可哈希的。这意味着在第一次调用此函数后，选项列表不能更改，否则会得到过时的结果。将列表转换为元组并对其进行哈希处理速度太慢。
    cache_code = (name, id(options))
    try:
        return lookup_option_cache[cache_code]
    except KeyError:
        pass

    # Nmap将'_'与长选项名称中的'-'视为相同。
    def canonicalize_name(name):
        return name.replace("_", "-")

    name = canonicalize_name(name)
    matches = [o for o in options
            if canonicalize_name(o.name).startswith(name)]
    if len(matches) == 0:
        # 没有匹配项。
        lookup_option_cache[cache_code] = None
    elif len(matches) == 1:
        # 只有一个匹配项--不是模糊的缩写。
        lookup_option_cache[cache_code] = matches[0]
    else:
        # 多于一个匹配项--只返回一个完全匹配项。
        for match in matches:
            if canonicalize_name(match.name) == name:
                lookup_option_cache[cache_code] = match
                break
        else:
            # 没有完全匹配项
            lookup_option_cache[cache_code] = None
    return lookup_option_cache[cache_code]

# 将选项拆分为名称、参数（如果有）和可能的剩余部分。
# 即使选项不包括参数但是需要参数，也不会报错；调用者必须从下一个命令行参数中获取参数。剩余部分是在剥离后剩下的部分。
def split_option(cmd_arg, options):
    ...
    # 如果命令参数以"--"开头，则提取出参数名和参数值（如果有的话），返回参数名、参数值和None
    if cmd_arg.startswith("--"):
        name = cmd_arg[2:]  # 提取参数名
        index = name.find('=')  # 寻找参数值的等号位置
        if index < 0:  # 如果没有等号，则参数值为None
            arg = None
        else:  # 如果有等号，则提取参数名和参数值
            name, arg = name[:index], name[index + 1:]
        return name, arg, None  # 返回参数名、参数值和None
    # 如果命令参数以"-"开头，则判断是单个短选项还是长选项
    elif cmd_arg.startswith("-"):
        name = cmd_arg[1:]  # 提取参数名
        # 检查是否只有一个短横线
        if name == "":  # 如果只有一个短横线，则返回参数名、None和None
            return name, None, None
        # 首先判断是否是长选项（或者单个短选项）
        index = name.find('=')  # 寻找参数值的等号位置
        if index < 0:  # 如果没有等号，则参数值为None
            arg = None
        else:  # 如果有等号，则提取参数名和参数值
            name, arg = name[:index], name[index + 1:]
        # 如果在选项列表中找到了参数名，则返回参数名、参数值和None
        if lookup_option(name, options) is not None:
            return name, arg, None
        # 没有找到，必须是一个短选项
        name = cmd_arg[1]  # 提取参数名
        option = lookup_option(name, options)  # 在选项列表中查找参数名对应的选项
        if option is None:  # 如果找不到对应的选项，则返回整个参数作为参数名、None和None
            return cmd_arg[1:], None, None
        rest = cmd_arg[2:]  # 提取参数值
        if rest == "":  # 如果参数值为空，则返回参数名、None和None
            return name, None, None
        if option.has_arg == option.NO_ARGUMENT:  # 如果选项不需要参数值，则返回参数名、None和剩余参数作为参数值
            return name, None, "-" + rest
        else:  # 如果选项需要参数值，则返回参数名、参数值和None
            return name, rest, None
    else:
        assert False, cmd_arg  # 如果不是以"--"或"-"开头，则断言错误
# 从命令行选项列表中查找并返回第一个选项（以及可能的选项参数）或位置参数
# 返回值有以下几种形式：
# * 一个字符串，表示位置参数；
# * 一个（选项，参数）对（参数可能为None）；
# * 一个（None，额外，...）元组，其中额外，...是一个未知选项及其后续参数的链，无法明确解释；
# * 在选项列表末尾返回None。
def get_option(cmd_args, options):
    # 如果命令行参数列表为空，返回None
    if len(cmd_args) == 0:
        return None
    # 弹出第一个命令行参数
    cmd_arg = cmd_args.pop(0)
    # 如果命令行参数为"--"
    if cmd_arg == "--":
        # 如果命令行参数列表为空，返回None
        if len(cmd_args) == 0:
            return None
        # 获取位置参数并替换"--"
        name = cmd_args[0]
        cmd_args[0] = "--"
        return name
    # 一个普通的位置参数
    if not cmd_arg.startswith("-"):
        return cmd_arg
    # 分割选项，获取选项名、参数和剩余部分
    name, arg, remainder = split_option(cmd_arg, options)
    # 如果有剩余部分，插入到命令行参数列表中
    if remainder is not None:
        cmd_args.insert(0, remainder)
    # 查找选项
    option = lookup_option(name, options)
    # 如果选项为空，则表示未识别的选项
    if option is None:
        # 如果参数不为空，则返回（None，cmd_arg）
        if arg is not None:
            return (None, cmd_arg)
        else:
            # 如果发现未知选项，但是有一个问题--我们不知道它是否需要参数。
            # 所以我们模拟如果选项需要参数和如果不需要参数的情况。
            # sync 函数通过在循环中调用这个函数来实现这一点。
            extras = [None, cmd_arg]
            rest = sync(cmd_args[1:], cmd_args[:], options)
            # rest 是参数列表的一部分，无论未知选项是否需要参数。将 rest 之前的所有内容放入 extras，然后将 cmd_args 设置为 rest。
            extras += cmd_args[0:len(cmd_args) - len(rest)]
            del cmd_args[0:len(cmd_args) - len(rest)]
            return tuple(extras)
    # 如果选项需要参数但是参数不为空，则将其视为额外参数
    elif option.has_arg == option.NO_ARGUMENT and arg is not None:
        return (None, cmd_arg)
    # 如果选项需要参数但是参数为空，则表示需要一个参数但是尚未读取
    elif option.has_arg == option.REQUIRED_ARGUMENT and arg is None:
        # 如果参数列表已经为空，则将其视为额外参数
        if len(cmd_args) == 0:
            return (None, cmd_arg)
        else:
            # 否则将参数列表的第一个参数弹出，并返回选项名和参数
            arg = cmd_args.pop(0)
            return (option.name, arg)
    else:
        # 其他情况直接返回选项名和参数
        return (option.name, arg)
# 给定两个命令行参数列表，逐步从较长的列表中获取选项，直到两个列表长度相等。返回结果列表。
def sync(a, b, options):
    while a != b:  # 当两个列表不相等时循环
        if len(a) > len(b):  # 如果列表 a 比列表 b 长
            get_option(a, options)  # 从列表 a 中获取选项
        else:
            get_option(b, options)  # 从列表 b 中获取选项
    return a  # 返回结果列表


# 这是 getopt_long_only 的生成器版本，另外还具有对未知选项的健壮处理。它产生的序列中的每个项将是以下之一：
# * 一个字符串，表示一个位置参数；
# * 一个 (option, argument) 对（argument 可能为 None）；
# * 一个 (None, extra, ...) 元组，其中 extra, ... 是一个未知选项及其后续参数的链，无法明确解释；
# * None，在选项列表的末尾。
def getopt_long_only_extras(cmd_args, short_opts, long_opts):
    options = make_options(short_opts, long_opts)  # 创建选项集合
    cmd_args_copy = cmd_args[:]  # 复制命令参数列表
    while True:  # 无限循环
        result = get_option(cmd_args_copy, options)  # 获取选项
        if result is None:  # 如果结果为 None
            break  # 跳出循环
        yield result  # 产生结果


class NmapOptions(object):
    SHORT_OPTIONS = "6Ab:D:d::e:Ffg:hi:M:m:nO::o:P:p:RrS:s:T:v::V"  # 短选项字符串

    # 应该从外部接口的角度视为等效的选项集合。例如，ops["--timing"] 意味着与 ops["-T"] 相同。
    EQUIVALENT_OPTIONS = (
        ("debug", "d"),
        ("help", "h"),
        ("iL", "i"),
        ("max-parallelism", "M"),
        ("osscan-guess", "fuzzy"),
        ("oG", "oM", "m"),
        ("oN", "o"),
        ("sP", "sn"),
        ("P", "PE", "PI"),
        ("PA", "PT"),
        ("P0", "PD", "PN", "Pn"),
        ("rH", "randomize-hosts"),
        ("source-port", "g"),
        ("timing", "T"),
        ("verbose", "v"),
        ("version", "V"),
    )
    # 创建一个空的等价映射字典
    EQUIVALENCE_MAP = {}
    # 遍历等价选项集合，将每个选项的别名映射到基本选项
    for set in EQUIVALENT_OPTIONS:
        base = set[0]
        aliases = set[1:]
        for alias in aliases:
            EQUIVALENCE_MAP[alias] = base
    
    # 定义一个时间性能名称到值的映射字典
    TIMING_PROFILE_NAMES = {
        "paranoid": 0, "sneaky": 1, "polite": 2,
        "normal": 3, "aggressive": 4, "insane": 5
    }
    
    # 初始化方法，创建选项对象并清空其内部状态
    def __init__(self):
        self.options = make_options(self.SHORT_OPTIONS, self.LONG_OPTIONS)
        self.clear()
    
    # 清空选项对象的内部状态
    def clear(self):
        self._executable = None
        self.target_specs = []
        self.extras = []
        # 内部映射选项名称到值的字典
        self.d = {}
    
    # 设置可执行文件路径
    def _set_executable(self, executable):
        self._executable = executable
    
    # 可执行文件路径的属性
    executable = property(lambda self: self._executable or "nmap",
            _set_executable)
    
    # 将选项名称转换为规范形式
    def canonicalize_name(self, name):
        opt, arg, remainder = split_option(name, self.options)
        assert remainder is None
        if arg is None:
            option = lookup_option(opt, self.options)
            if option:
                option = option.name
            else:
                option = opt
        else:
            option = name.lstrip("-")
        option = NmapOptions.EQUIVALENCE_MAP.get(option, option)
        return option
    
    # 获取选项值
    def __getitem__(self, key):
        return self.d.get(self.canonicalize_name(key))
    
    # 设置选项值
    def __setitem__(self, key, value):
        self.d[self.canonicalize_name(key)] = value
    
    # 获取选项值，如果不存在则设置默认值
    def setdefault(self, key, default):
        return self.d.setdefault(self.canonicalize_name(key), default)
    
    # 解析命令行选项
    def parse(self, opt_list):
        self.clear()
        if len(opt_list) > 0:
            self.executable = opt_list[0]
        for result in getopt_long_only_extras(
                opt_list[1:], self.SHORT_OPTIONS, self.LONG_OPTIONS):
            self.handle_result(result)
    
    # 解析字符串形式的选项
    def parse_string(self, opt_string):
        self.parse(split_quoted(opt_string))
    # 定义一个方法，用于渲染字符串
    def render_string(self):
        # 调用 render 方法渲染字符串，并将结果返回
        return join_quoted(self.render())
# 导入所需的模块
import doctest
import unittest

# 定义 NmapOptionsTest 类，继承自 unittest.TestCase
class NmapOptionsTest(unittest.TestCase):
    # 测试清除方法
    def test_clear(self):
        """Test that a new object starts without defining any options, that the
        clear method removes all options, and that parsing the empty string or
        an empty list removes all options."""
        # 定义测试用的 Nmap 命令
        TEST = "nmap -T4 -A -v localhost --webxml"
        # 创建 NmapOptions 对象
        ops = NmapOptions()
        # 断言新对象的选项数量为 1
        self.assertTrue(len(ops.render()) == 1)
        # 解析测试用的 Nmap 命令
        ops.parse_string(TEST)
        # 断言解析后的选项数量不为 1
        self.assertFalse(len(ops.render()) == 1)
        # 清除选项
        ops.clear()
        # 断言清除后的选项数量为 1
        self.assertTrue(len(ops.render()) == 1)
        # 重新解析测试用的 Nmap 命令
        ops.parse_string(TEST)
        # 解析空字符串
        ops.parse_string("")
        # 断言渲染后的字符串为 "nmap"
        self.assertEqual(ops.render_string(), "nmap")
        # 重新解析测试用的 Nmap 命令
        ops.parse_string(TEST)
        # 解析空列表
        ops.parse([])
        # 断言渲染后的字符串为 "nmap"

    # 测试默认可执行文件
    def test_default_executable(self):
        """Test that there is a default executable member set."""
        # 创建 NmapOptions 对象
        ops = NmapOptions()
        # 断言可执行文件不为空
        self.assertNotNull(ops.executable)

    # 测试设置可执行文件
    def test_default_executable(self):
        """Test that you can set the executable."""
        # 创建 NmapOptions 对象
        ops = NmapOptions()
        # 设置可执行文件为 "foo"
        ops.executable = "foo"
        # 断言可执行文件为 "foo"
        self.assertEqual(ops.executable, "foo")
        # 断言渲染后的列表为 ["foo"]
        self.assertEqual(ops.render(), ["foo"])

    # 测试渲染方法
    def test_render(self):
        """Test that the render method returns a list."""
        # 定义测试用的 Nmap 命令
        TEST = "nmap -T4 -A -v localhost --webxml"
        # 创建 NmapOptions 对象
        ops = NmapOptions()
        # 解析测试用的 Nmap 命令
        ops.parse_string(TEST)
        # 断言渲染后的类型为列表
        self.assertTrue(type(ops.render()) == list,
                "type == %s" % type(ops.render))
    # 测试需要引用的字符串是否被引用
    def test_render_quoted(self):
        """Test that strings that need to be quoted are quoted."""
        # 创建 NmapOptions 对象
        ops = NmapOptions()
        # 解析字符串，设置相应的选项
        ops.parse_string('"/path/ /nmap" --script "test one two three"')
        # 断言可执行文件路径是否正确
        self.assertEqual(ops.executable, "/path/ /nmap")
        # 断言脚本参数是否正确
        self.assertEqual(ops["--script"], "test one two three")
        # 断言目标规范是否为空
        self.assertEqual(ops.target_specs, [])
        # 重新渲染字符串
        s = ops.render_string()
        # 重新解析字符串
        ops.parse_string(s)
        # 断言可执行文件路径是否正确
        self.assertEqual(ops.executable, "/path/ /nmap")
        # 断言脚本参数是否正确
        self.assertEqual(ops["--script"], "test one two three")
        # 断言目标规范是否为空
        self.assertEqual(ops.target_specs, [])

    # 测试 -- 结束参数处理
    def test_end(self):
        """Test that -- ends argument processing."""
        # 创建 NmapOptions 对象
        ops = NmapOptions()
        # 解析字符串，设置相应的选项
        ops.parse_string("nmap -v -- -v")
        # 断言是否存在 -v 选项
        self.assertTrue(ops["-v"] == 1)
        # 断言目标规范是否包含 -v
        self.assertTrue(ops.target_specs == ["-v"])

    # 测试解析和重新渲染是否得到相同的结果
    def test_roundtrip(self):
        """Test that parsing and re-rendering a previous rendering gives the
        same thing as the previous rendering."""
        # 定义测试用例
        TESTS = (
            "nmap",
            "nmap -v",
            "nmap -vv",
            "nmap -d -v",
            "nmap -d -d",
            "nmap -d -v -d",
            "nmap localhost",
            "nmap -oX - 192.168.0.1 -PS10",
        )
        # 创建 NmapOptions 对象
        ops = NmapOptions()
        # 遍历测试用例
        for test in TESTS:
            # 解析字符串，设置相应的选项
            ops.parse_string(test)
            # 获取第一次渲染的字符串
            opt_string_1 = ops.render_string()
            # 重新解析第一次渲染的字符串
            ops.parse_string(opt_string_1)
            # 获取第二次渲染的字符串
            opt_string_2 = ops.render_string()
            # 断言两次渲染的字符串是否相同
            self.assertEqual(opt_string_1, opt_string_2)

    # 测试选项名称中的下划线是否被视为破折号，并且被规范为破折号
    def test_underscores(self):
        """Test that underscores in option names are treated the same as
        dashes (and are canonicalized to dashes)."""
        # 创建 NmapOptions 对象
        ops = NmapOptions()
        # 解析字符串，设置相应的选项
        ops.parse_string("nmap --osscan_guess")
        # 断言渲染后的字符串中是否包含规范化后的选项名称
        self.assertTrue("--osscan-guess" in ops.render_string())
    # 测试可能会出现复杂参数情况
    def test_args(self):
        """Test potentially tricky argument scenarios."""
        # 创建 NmapOptions 对象
        ops = NmapOptions()
        # 解析命令行参数字符串
        ops.parse_string("nmap -d9")
        # 断言目标规范列表长度为 0
        self.assertTrue(len(ops.target_specs) == 0)
        # 断言 ops 对象中 -d 参数的值为 9
        self.assertTrue(ops["-d"] == 9, ops["-d"])
        ops.parse_string("nmap -d 9")
        # 断言目标规范列表为 ["9"]
        self.assertTrue(ops.target_specs == ["9"])
        # 断言 ops 对象中 -d 参数的值为 1
        self.assertTrue(ops["-d"] == 1)

    # 测试可以重复使用以增加效果的选项
    def test_repetition(self):
        """Test options that can be repeated to increase their effect."""
        # 创建 NmapOptions 对象
        ops = NmapOptions()
        # 解析命令行参数字符串
        ops.parse_string("nmap -vv")
        # 断言 ops 对象中 -v 参数的值为 2
        self.assertTrue(ops["-v"] == 2)
        ops.parse_string("nmap -v -v")
        # 断言 ops 对象中 -v 参数的值为 2
        self.assertTrue(ops["-v"] == 2)
        ops.parse_string("nmap -ff")
        # 断言 ops 对象中 -f 参数的值为 2
        self.assertTrue(ops["-f"] == 2)
        ops.parse_string("nmap -f -f")
        # 断言 ops 对象中 -f 参数的值为 2
        self.assertTrue(ops["-f"] == 2)
        # 注意：与 -d 不同，-v 不接受可选的数值参数
        ops.parse_string("nmap -d2 -d")
        # 断言 ops 对象中 -d 参数的值为 3
        self.assertTrue(ops["-d"] == 3)

    # 测试给定给 -s 选项的多个扫描类型是否都被正确解释
    def test_scan_types(self):
        """Test that multiple scan types given to the -s option are all
        interpreted correctly."""
        # 创建 NmapOptions 对象
        ops = NmapOptions()
        # 解析命令行参数字符串
        ops.parse_string("nmap -s")
        # 断言额外参数列表为 ["-s"]
        self.assertTrue(ops.extras == ["-s"])
        ops.parse_string("nmap -sS")
        # 断言额外参数列表为空
        self.assertTrue(ops.extras == [])
        # 断言 ops 对象中 -sS 参数为 True
        self.assertTrue(ops["-sS"])
        # 断言 ops 对象中 -sU 参数为 False
        self.assertTrue(not ops["-sU"])
        ops.parse_string("nmap -sSU")
        # 断言 ops 对象中 -sS 参数为 True
        self.assertTrue(ops["-sS"])
        # 断言 ops 对象中 -sU 参数为 True
        self.assertTrue(ops["-sU"])
    # 测试处理在文档中未指定解释的构造，但应该与 GNU getopt 的解释匹配
    def test_quirks(self):
        ops = NmapOptions()
        # 长选项可以用一个破折号写成
        ops.parse_string("nmap -min-rate 100")
        self.assertTrue(ops["--min-rate"] == "100")
        ops.parse_string("nmap -min-rate=100")
        self.assertTrue(ops["--min-rate"] == "100")

        # 不需要参数的短选项可以后面跟一个长选项
        ops.parse_string("nmap -nFmin-rate 100")
        self.assertTrue(ops["-n"])
        self.assertTrue(ops["-F"])
        self.assertTrue(ops["--min-rate"] == "100")

        # 需要参数的短选项会消耗剩余的参数
        ops.parse_string("nmap -nFp1-100")
        self.assertTrue(ops["-n"])
        self.assertTrue(ops["-F"])
        self.assertTrue(ops["-p"] == "1-100")

    # 测试失败的整数转换会导致选项出现在额外参数中
    def test_conversion(self):
        ops = NmapOptions()
        ops.parse_string("nmap -d#")
        self.assertTrue(ops.extras == ["-d#"])
        ops.parse_string("nmap -T monkeys")
        self.assertTrue(ops["-T"] is None)
        self.assertTrue(ops.extras == ["-T", "monkeys"])
        ops.parse_string("nmap -iR monkeys")
        self.assertTrue(ops["-iR"] is None)
        self.assertTrue(ops.extras == ["-iR", "monkeys"])

    # 测试获取非选项的值返回 None
    def test_read_unknown(self):
        ops = NmapOptions()
        self.assertEqual(ops["-x"], None)
        self.assertEqual(ops["--nonoption"], None)
    # 定义一个测试函数，用于测试等效的选项名称是否被正确规范化
    def test_canonical_option_names(self):
        """Test that equivalent option names are properly canonicalized, so
        that ops["--timing"] and ops["-T"] mean the same thing, for example."""
        # 定义等效的选项名称的元组
        EQUIVS = (
            ("--debug", "-d"),
            ("--help", "-h"),
            ("-iL", "-i"),
            ("--max-parallelism", "-M"),
            ("--osscan-guess", "--fuzzy"),
            ("-oG", "-oM", "-m"),
            ("-oN", "-o"),
            ("-sP", "-sn"),
            ("-P", "-PE", "-PI"),
            ("-PA", "-PT"),
            ("-P0", "-PD", "-PN", "-Pn"),
            ("--source-port", "-g"),
            ("--timing", "-T"),
            ("--verbose", "-v"),
            ("--version", "-V"),
            ("--min-rate", "-min-rate", "--min_rate", "-min_rate")
        )
        # 创建一个 NmapOptions 对象
        ops = NmapOptions()
        # 遍历等效的选项名称元组
        for set in EQUIVS:
            # 遍历每个等效的选项名称
            for opt in set:
                # 清空 ops 对象
                ops.clear()
                # 设置当前选项名称对应的值为 "test"
                ops[opt] = "test"
                # 遍历当前等效的选项名称元组
                for other in set:
                    # 断言 ops 对象中当前选项名称对应的值都为 "test"
                    self.assertTrue(ops[other] == "test",
                        "%s and %s not the same" % (opt, other))
class SplitQuotedTest(unittest.TestCase):
    """A unittest class that tests the split_quoted function."""

    def test_split(self):
        # 测试空字符串的情况
        self.assertEqual(split_quoted(''), [])
        # 测试只有一个单词的情况
        self.assertEqual(split_quoted('a'), ['a'])
        # 测试有多个单词的情况
        self.assertEqual(split_quoted('a b c'), 'a b c'.split())

    def test_quotes(self):
        # 测试带双引号的情况
        self.assertEqual(split_quoted('a "b" c'), ['a', 'b', 'c'])
        # 测试带有空格的双引号的情况
        self.assertEqual(split_quoted('a "b c"'), ['a', 'b c'])
        # 测试带有多个双引号的情况
        self.assertEqual(split_quoted('a "b c""d e"'), ['a', 'b cd e'])
        # 测试带有双引号和其他字符的情况
        self.assertEqual(split_quoted('a "b c"z"d e"'), ['a', 'b czd e'])

    def test_backslash(self):
        # 测试带有转义字符的情况
        self.assertEqual(split_quoted('"\\""'), ['"'])
        self.assertEqual(split_quoted('\\"\\""'), ['\\"'])
        self.assertEqual(split_quoted('"\\"\\""'), ['""'])


if __name__ == "__main__":
    # 运行文档测试
    doctest.testmod()
    # 运行单元测试
    unittest.main()
```