# `nmap\zenmap\zenmapCore\UmitOptionParser.py`

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
# 导入 OptionParser 类
from optparse import OptionParser
# 从 zenmapCore.Name 模块导入 NMAP_DISPLAY_NAME 变量
from zenmapCore.Name import NMAP_DISPLAY_NAME
# 从 zenmapCore.Version 模块导入 VERSION 变量
from zenmapCore.Version import VERSION
# 导入 zenmapCore.I18N 模块，但未使用
import zenmapCore.I18N
# 从 zenmapCore.BasePaths 模块导入 base_paths 变量
from zenmapCore.BasePaths import base_paths
# 从 zenmapCore.DelayedObject 模块导入 DelayedObject 类
from zenmapCore.DelayedObject import DelayedObject

# 定义 UmitOptionParser 类，继承自 OptionParser 类
class UmitOptionParser(OptionParser):
    # 初始化方法
    def __init__(self, args=False):
        # 调用父类 OptionParser 的初始化方法，设置版本号
        OptionParser.__init__(self, version="%%prog %s" % VERSION)

        # 设置命令行使用说明
        self.set_usage("%prog [options] [result files]")

        # 添加 --confdir 选项，设置默认值为 base_paths["user_config_dir"]，并提供帮助信息
        self.add_option("--confdir",
            default=base_paths["user_config_dir"],
            dest="confdir",
            metavar="DIR",
            help=_("\
# 将 DIR 作为用户配置目录。默认值为 %default
Use DIR as the user configuration directory. Default: %default"))

## 打开扫描结果（GUI）
### 运行，打开指定的扫描结果文件，应该是一个 nmap XML 输出文件。
### 如果没有选项，用户指定了一些位置参数，应该验证此选项，并将其视为扫描结果文件。
Open Scan Results (GUI)
self.add_option("-f", "--file",
                default=[],
                action="append",
                type="string",
                dest="result_files",
                help=_("Specify a scan result file in Nmap XML output \
format. Can be used more than once to specify several \
scan result files."))

## 使用参数运行 nmap（GUI）
### 打开并使用指定的参数运行 nmap。位置参数应该用于提供 nmap 命令
Run nmap with args (GUI)
self.add_option("-n", "--nmap",
                default=[],
                action="callback",
                callback=self.__nmap_callback,
                help=_("Run %s with the specified args."
                    ) % NMAP_DISPLAY_NAME)

## 对目标执行配置文件（GUI）
### 位置参数应该被视为要提供给此扫描的目标
Execute a profile against a target (GUI)
self.add_option("-p", "--profile",
                default="",
                action="store",
                help=_("Begin with the specified profile \
selected. If combined with the -t (--target) option, \
# 添加一个选项，用于指定要针对的目标，并自动运行配置文件
self.add_option("-p", "--profile",
                default=False,
                action="store",
                help=_("Specify a profile to be used along with other \
options, or simply open with the profile field filled with the specified profile"))

## 目标（GUI）
### 指定一个目标，可与其他命令行选项一起使用，或者仅打开第一个选项卡的目标字段填充为指定的目标
self.add_option("-t", "--target",
                default=False,
                action="store",
                help=_("Specify a target to be used along with other \
options. If specified alone, open with the target field filled with the \
specified target"))

## 详细程度
self.add_option("-v", "--verbose",
                default=0,
                action="count",
                help=_("Increase verbosity of the output. May be \
used more than once to get even more verbosity"))

# 解析选项和参数
if args:
    self.options, self.args = self.parse_args(args)
else:
    self.options, self.args = self.parse_args()

def __nmap_callback(self, option, opt_str, value, parser):
    nmap_args = []
    # 遍历尚未解析的命令行中传递的下一个参数
    while parser.rargs:
        # 将下一个参数存储在特定列表中
        nmap_args.append(parser.rargs[0])

        # 从rargs中删除添加的参数，以避免后续由optparse解析
        del parser.rargs[0]

    # 在parser.values中设置变量nmap，因此您可以调用option.nmap，并将nmap_args作为结果
    setattr(parser.values, "nmap", nmap_args)

def get_confdir(self):
    return self.options.confdir
    # 返回一个 nmap 参数列表，如果用户没有调用这个选项，则返回 False
    def get_nmap(self):
        try:
            # 获取用户输入的 nmap 参数
            nmap = self.options.nmap
            # 如果 nmap 参数存在，则返回它
            if nmap:
                return nmap
        except AttributeError:
            # 如果没有找到 nmap 参数，则返回 False
            return False
    
    # 返回一个字符串，其中包含配置文件的名称，如果用户没有指定配置文件选项，则返回 False
    def get_profile(self):
        if self.options.profile != "":
            # 如果用户指定了配置文件选项，则返回配置文件名称
            return self.options.profile
        # 如果没有指定配置文件选项，则返回 False
        return False
    
    # 返回一个字符串，其中包含用户指定的目标，如果用户没有调用这个选项，则返回 False
    def get_target(self):
        return self.options.target
    
    # 返回一个字符串列表，其中包含使用 -f (--file) 选项指定的文件名和每个位置参数
    def get_open_results(self):
        files = []
        # 添加使用 -f 指定的参数
        if self.options.result_files:
            files = self.options.result_files[:]
        # 添加任何其他参数
        files += self.args
        return files
    
    # 返回一个表示应用程序详细程度的整数。详细程度从 40 开始，这意味着只有高于 ERROR 级别的消息才会在输出中报告。随着这个值的降低，详细程度会增加。
    def get_verbose(self):
        return 40 - (self.options.verbose * 10)
# 创建一个延迟对象，用于延迟执行 UmitOptionParser
option_parser = DelayedObject(UmitOptionParser)

# 如果当前脚本作为主程序执行
if __name__ == "__main__":
    # 创建 UmitOptionParser 对象
    opt = UmitOptionParser()
    # 解析命令行参数，返回 options 和 args
    options, args = opt.parse_args()
```