# `nmap\zenmap\zenmapCore\StringPool.py`

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
# 定义一个继承自字典的类UniqueStringMap
class UniqueStringMap(dict):
    # 当字典中缺少指定的键时，将该键添加到字典中并返回该键
    def __missing__(self, key):
        self[key] = key
        return key

# 创建一个UniqueStringMap的实例，并赋值给UNIQUE_STRING_MAP变量
UNIQUE_STRING_MAP = UniqueStringMap()

# 返回一个字符串的唯一表示（根据id），并允许字符串被垃圾回收
unique = UNIQUE_STRING_MAP.__getitem__

# 导入unittest模块
import unittest

# 定义一个StringPoolTest类，继承自unittest.TestCase
class StringPoolTest(unittest.TestCase):

    # 定义一个测试方法test_pool
    def test_pool(self):
        # 定义一个源字符串source
        source = "Test string. Zenmap. Test string."
        # 断言source的前12个字符和后12个字符经过unique函数处理后是相同的
        self.assertIs(unique(source[:12]), unique(source[-12:]))

# 如果该脚本被直接执行，则执行测试
if __name__ == '__main__':
    unittest.main()
```