# `nmap\zenmap\zenmapCore\NSEDocParser.py`

```cpp
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
# 导入正则表达式模块
import re

# 定义一个类 NSEDocEvent
class NSEDocEvent (object):
    # 初始化方法，接受类型和文本作为参数
    def __init__(self, type, text=None):
        self.type = type
        self.text = text

# 定义一个函数，用于解析段落级别的 NSEDoc 标记，用于段落和列表内部
# 返回解析结束后的位置和事件列表
def nsedoc_parse_sub(text, pos):
    # 初始化事件列表
    events = []
    # 使用正则表达式匹配文本中的 <code> 标记
    m = re.match(r'([^\n]*?)<code>(.*?)</code>', text[pos:], re.S)
    if m:
        # 如果匹配到了 <code> 标记
        if m.group(1):
            # 将 <code> 标记之前的文本作为文本事件添加到事件列表中
            events.append(NSEDocEvent("text", m.group(1).replace("\n", " ")))
        # 将 <code> 标记中的内容作为代码事件添加到事件列表中
        events.append(NSEDocEvent("code", m.group(2)))
        # 返回解析结束后的位置和事件列表
        return pos + m.end(), events
    # 使用正则表达式匹配文本中的换行符
    m = re.match(r'[^\n]*(\n|$)', text[pos:])
    # 如果匹配成功
    if m:
        # 如果匹配组成功
        if m.group():
            # 将匹配到的文本替换掉换行符，并添加到事件列表中
            events.append(NSEDocEvent("text", m.group().replace("\n", " ")))
        # 返回匹配结束的位置和事件列表
        return pos + m.end(), events
    # 如果没有匹配成功，则返回当前位置和事件列表
    return pos, events
def nsedoc_parse(text):
    """Parse text marked up for NSEDoc. This is a generator that returns a
    sequence of NSEDocEvents. The type of the event may be "paragraph_start",
    "paragraph_end", "list_start", "list_end", "list_item_start",
    "list_item_end", "text", or "code". The types "text" and "code" have a text
    member with the text that they contain."""
    # 初始化索引 i 为 0
    i = 0
    # 初始化 in_list 为 False，表示不在列表中
    in_list = False

    # 循环遍历文本内容
    while i < len(text):
        # 跳过空白字符
        while i < len(text) and text[i].isspace():
            i += 1
        # 如果已经遍历完文本，则跳出循环
        if i >= len(text):
            break
        # 生成段落开始的 NSEDocEvent
        yield NSEDocEvent("paragraph_start")
        # 继续遍历文本
        while i < len(text):
            # 如果匹配到换行符或者文本末尾，则跳出循环
            if re.match(r'\s*(\n|$)', text[i:]):
                break
            # 如果文本以 "*" 开头，则表示列表项
            if text.startswith("* ", i):
                # 如果不在列表中，则生成列表开始的 NSEDocEvent
                if not in_list:
                    yield NSEDocEvent("list_start")
                    in_list = True
                # 跳过列表标记 "* "
                i += 2
                # 生成列表项开始的 NSEDocEvent
                yield NSEDocEvent("list_item_start")
                # 调用 nsedoc_parse_sub 函数解析子内容
                i, events = nsedoc_parse_sub(text, i)
                # 生成子内容的 NSEDocEvent
                for event in events:
                    yield event
                # 生成列表项结束的 NSEDocEvent
                yield NSEDocEvent("list_item_end")
            else:
                # 如果在列表中，则生成列表结束的 NSEDocEvent
                if in_list:
                    yield NSEDocEvent("list_end")
                    in_list = False
                # 调用 nsedoc_parse_sub 函数解析子内容
                i, events = nsedoc_parse_sub(text, i)
                # 生成子内容的 NSEDocEvent
                for event in events:
                    yield event
        # 如果在列表中，则生成列表结束的 NSEDocEvent
        if in_list:
            yield NSEDocEvent("list_end")
            in_list = False
        # 生成段落结束的 NSEDocEvent
        yield NSEDocEvent("paragraph_end")
```