# `nmap\zenmap\zenmapGUI\higwidgets\__init__.py`

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
"""
higwidgets/__init__.py

This module implements GTK Widgets that try their best to adhere to the
GNOME Human Interface Guidelines (aka HIG).

This is mostly implemented by subclassing from the GTK classes, and
providing defaults that better match the HIG specifications/recommendations.
"""

# 导入模块 gtkutils
from .gtkutils import *
# 导入模块 higboxes
from .higboxes import *
# 导入模块 higbuttons
from .higbuttons import *
# 导入模块 higdialogs
from .higdialogs import *
# 导入模块 higentries
from .higentries import *
# 导入模块 higexpanders
from .higexpanders import *
# 导入模块 higlabels
from .higlabels import *
# 导入模块 higlogindialogs
from .higlogindialogs import *
# 导入模块 higprogressbars
from .higprogressbars import *
# 导入模块 higscrollers
from .higscrollers import *
# 导入模块 higspinner
from .higspinner import *
# 导入模块 higtables
from .higtables import *
# 导入模块 higtextviewers
from .higtextviewers import *
# 导入模块 higwindows
from .higwindows import *
```