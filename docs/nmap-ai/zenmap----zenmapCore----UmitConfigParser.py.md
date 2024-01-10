# `nmap\zenmap\zenmapCore\UmitConfigParser.py`

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
# 导入所需的模块
from configparser import ConfigParser, DEFAULTSECT, NoOptionError, \
        NoSectionError
from zenmapCore.UmitLogging import log

# 创建一个自定义的配置解析器类，继承自ConfigParser
class UmitConfigParser(ConfigParser):

    # 初始化方法
    def __init__(self, *args):
        # 初始化文件名列表和失败标志
        self.filenames = None
        self.failed = False
        # 调用父类的初始化方法
        ConfigParser.__init__(self, *args)

    # 设置配置项的值
    def set(self, section, option, value):
        # 如果没有该section，则添加该section
        if not self.has_section(section):
            self.add_section(section)
        # 调用父类的设置方法设置配置项的值
        ConfigParser.set(self, section, option, str(value))
        # 保存修改
        self.save_changes()

    # 读取配置文件
    def read(self, filename):
        # 打印调试信息
        log.debug(">>> Trying to parse: %s" % filename)
        # 调用父类的读取方法读取配置文件
        if ConfigParser.read(self, filename):
            self.filenames = filename
        # 返回读取的文件名
        return self.filenames
    # 保存对配置文件的更改
    def save_changes(self):
        # 如果存在文件名
        if self.filenames:
            # 记录保存的文件名
            log.debug("saving to %s" % self.filenames)
            try:
                # 以写入模式打开文件
                with open(self.filenames, 'w') as fp:
                    # 调用 write 方法写入配置
                    self.write(fp)
            # 捕获异常
            except Exception as e:
                # 记录保存失败的异常
                self.failed = e
                log.error(">>> Can't save to %s: %s" % (self.filenames, e))
                return
            # 保存成功，重置保存失败标志
            self.failed = False
        # 如果不存在文件名
        else:
            # 记录无法保存更改的信息
            log.debug(">>> UmitConfigParser can't save changes: no filename")

    # 写入配置文件，按字母顺序排序
    def write(self, fp):
        if self._defaults:
            # 写入默认部分
            fp.write("[%s]\n" % DEFAULTSECT)

            # 对默认部分的键值对按字母顺序排序
            items = sorted(self._defaults.items())

            # 逐个写入默认部分的键值对
            for (key, value) in items:
                fp.write("%s = %s\n" % (key, str(value).replace('\n', '\n\t')))
            fp.write("\n")

        # 对各个部分按字母顺序排序
        sects = sorted(self._sections.keys())

        # 逐个写入各个部分的键值对
        for section in sects:
            fp.write("[%s]\n" % section)
            for (key, value) in self._sections[section].items():
                if key != "__name__":
                    fp.write("%s = %s\n" %
                             (key, str(value).replace('\n', '\n\t')))
            fp.write("\n")
# 测试读取指定文件的配置内容
def test_umit_conf_content(filename):
    # 创建配置解析器对象
    parser = ConfigParser()
    # 读取指定文件的配置内容
    parser.read(filename)

    # Paths section
    # 指定要读取的配置段落
    section = "paths"
    # 断言获取指定配置段落和选项的值，如果不存在则返回 False
    assert get_or_false(parser, section, "nmap_command_path")


# 获取指定配置解析器对象中指定段落和选项的值，如果不存在则返回 False
def get_or_false(parser, section, option):
    try:
        # 尝试获取指定段落和选项的值
        result = parser.get(section, option)
        return result
    except NoOptionError:
        # 如果选项不存在，则返回 False
        return False
    except NoSectionError:
        # 如果段落不存在，则返回 False
        return False
```