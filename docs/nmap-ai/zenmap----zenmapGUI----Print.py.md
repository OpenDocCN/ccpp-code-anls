# `nmap\zenmap\zenmapGUI\Print.py`

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
# 导入gi库，确保使用的是Gtk 3.0版本和PangoCairo 1.0版本
import gi
gi.require_version("Gtk", "3.0")
gi.require_version("PangoCairo", "1.0")
from gi.repository import Gtk, GLib, Pango, PangoCairo

# 设置等宽字体的描述信息
MONOSPACE_FONT_DESC = Pango.FontDescription("Monospace 12")

# 定义PrintState类，作为传递给Gtk.PrintOperation回调函数的用户数据
class PrintState(object):
    """This is the userdatum passed to Gtk.PrintOperation callbacks."""
    # 初始化方法，接受一个库存和一个条目作为参数
    def __init__(self, inventory, entry):
        """entry is a ScansListStoreEntry."""
        # 如果条目已解析，则使用解析后的 nmap 输出
        if entry.parsed:
            output = entry.parsed.nmap_output
        else:
            # 如果条目还在运行、失败或取消，则获取命令的输出
            output = entry.command.get_output()
        # 如果输出为空，则设置为换行符
        if not output:
            output = "\n"
        # 将输出按行分割，存储在实例变量 lines 中
        self.lines = output.splitlines()

    # 开始打印方法，计算打印的页数
    def begin_print(self, op, context):
        """Calculates the number of printed pages."""
        # 创建 Pango 布局对象
        layout = context.create_pango_layout()
        # 设置字体描述
        layout.set_font_description(MONOSPACE_FONT_DESC)
        # 设置文本内容为 "dummy"
        layout.set_text("dummy", -1)
        # 获取第一行的高度
        line = layout.get_line(0)
        # 获取逻辑矩形的高度
        line_height = line.get_extents()[1].height / Pango.SCALE

        # 获取页面的高度
        page_height = context.get_height()
        # 计算每页的行数
        self.lines_per_page = int(page_height / line_height)
        # 设置打印操作的页数
        op.set_n_pages((len(self.lines) - 1) // self.lines_per_page + 1)

    # 绘制页面方法
    def draw_page(self, op, context, page_nr):
        # 获取当前页的行
        this_page_lines = self.lines[
                page_nr * self.lines_per_page:
                (page_nr + 1) * self.lines_per_page]
        # 创建 Pango 布局对象
        layout = context.create_pango_layout()
        # 不进行换行
        layout.set_width(-1)
        # 设置字体描述
        layout.set_font_description(MONOSPACE_FONT_DESC)
        # 将当前页的行连接成文本
        text = "\n".join(this_page_lines)
        layout.set_text(text, -1)

        # 获取 Cairo 上下文
        cr = context.get_cairo_context()
        # 显示布局
        PangoCairo.show_layout(cr, layout)
# 定义一个运行打印操作的函数，接受存货和条目作为参数
def run_print_operation(inventory, entry):
    # 创建一个打印操作对象
    op = Gtk.PrintOperation()
    # 创建一个打印状态对象，传入存货和条目
    state = PrintState(inventory, entry)
    # 连接打印操作的"begin-print"信号到状态对象的begin_print方法
    op.connect("begin-print", state.begin_print)
    # 连接打印操作的"draw-page"信号到状态对象的draw_page方法
    op.connect("draw-page", state.draw_page)
    # 尝试运行打印操作，显示打印对话框，不传入打印设置
    try:
        op.run(Gtk.PrintOperationAction.PRINT_DIALOG, None)
    # 捕获 GLib.GError 异常
    except GLib.GError:
        # 取消打印操作可能会导致错误
        #   GError: Error from StartDoc
        # 参考链接：http://seclists.org/nmap-dev/2012/q4/161
        pass
```