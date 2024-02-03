# `nmap\zenmap\zenmapGUI\About.py`

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
# 导入gi模块，确保使用的是Gtk 3.0版本
import gi
gi.require_version("Gtk", "3.0")
from gi.repository import Gtk

# 导入webbrowser模块，用于在浏览器中打开网页
import webbrowser

# 导入zenmapGUI中的各种自定义控件
from zenmapGUI.higwidgets.higdialogs import HIGDialog
from zenmapGUI.higwidgets.higwindows import HIGWindow
from zenmapGUI.higwidgets.higboxes import HIGVBox, HIGHBox, hig_box_space_holder
from zenmapGUI.higwidgets.higbuttons import HIGButton
from zenmapGUI.higwidgets.hignotebooks import HIGNotebook
from zenmapGUI.higwidgets.higscrollers import HIGScrolledWindow
from zenmapGUI.higwidgets.higtextviewers import HIGTextView

# 导入zenmapCore中的各种信息和版本号
from zenmapCore.Name import APP_DISPLAY_NAME, APP_WEB_SITE, APP_COPYRIGHT, NMAP_DISPLAY_NAME, NMAP_WEB_SITE, UMIT_DISPLAY_NAME, UMIT_WEB_SITE
from zenmapCore.Version import VERSION
import zenmapCore.I18N  # lgtm[py/unused-import]

# 防止加载PyXML模块
import xml
# 从 xml 模块的路径列表中移除包含 "_xmlplus" 的路径
xml.__path__ = [x for x in xml.__path__ if "_xmlplus" not in x]

# 导入用于在标记标签中转义文本的模块
from xml.sax.saxutils import escape

# 定义一个名为 _program_entry 的类，继承自 Gtk.Box
class _program_entry(Gtk.Box):
    """A little box containing labels with a program's name and
    description and a clickable link to its web site."""

    # 用于设置程序名称和网站按钮之间的间距
    NAME_WEB_SITE_SPACING = 20

    # 初始化方法
    def __init__(self, name=None, web_site=None, description=None):
        # 调用父类的初始化方法
        Gtk.Box.__init__(self, orientation=Gtk.Orientation.VERTICAL)

        # 创建一个水平方向的盒子
        self.hbox = Gtk.Box.new(Gtk.Orientation.HORIZONTAL, self.NAME_WEB_SITE_SPACING)
        self.pack_start(self.hbox, True, True, 0)

        # 如果存在名称，则创建一个标签并添加到水平盒子中
        if name is not None:
            name_label = Gtk.Label()
            name_label.set_markup(
                    '<span size="large" weight="bold">%s</span>' % escape(
                        name))
            self.hbox.pack_start(name_label, False, True, 0)

        # 如果存在网站，则创建一个链接按钮并添加到水平盒子中
        if web_site is not None:
            web_site_button = Gtk.LinkButton.new(web_site)
            web_site_button.connect("clicked", self._link_button_open)
            self.hbox.pack_start(web_site_button, False, True, 0)

        # 如果存在描述，则创建一个标签并添加到垂直盒子中
        if description is not None:
            description_label = Gtk.Label()
            description_label.set_alignment(0.0, 0.0)
            description_label.set_line_wrap(True)
            description_label.set_text(description)
            self.pack_start(description_label, True, True, 0)

    # 打开链接按钮的回调方法
    def _link_button_open(self, widget):
        webbrowser.open(widget.get_uri())


# 定义一个名为 About 的类，继承自 HIGDialog
class About(HIGDialog):
    """An about dialog showing information about the program. It is meant to
    have roughly the same feel as Gtk.AboutDialog."""
    # 关闭方法
    def _close(self, widget, response):
        # 如果存在 _umit_credits_dialog，则销毁它
        if self._umit_credits_dialog is not None:
            self._umit_credits_dialog.destroy()
            self._umit_credits_dialog = None

        # 隐藏对话框
        self.hide()
    # 显示 Umit 的积分
    def _show_umit_credits(self, widget):
        # 如果 Umit 积分对话框已经存在，则显示它并返回
        if self._umit_credits_dialog is not None:
            self._umit_credits_dialog.present()
            return

        # 如果 Umit 积分对话框不存在，则创建一个新的 UmitCredits 对象
        self._umit_credits_dialog = UmitCredits()

        # 定义一个函数，在积分对话框销毁时执行，标记积分对话框已被销毁
        def credits_destroyed(widget):
            # 标记积分对话框已被销毁
            self._umit_credits_dialog = None

        # 连接积分对话框的 "destroy" 信号到销毁函数
        self._umit_credits_dialog.connect("destroy", credits_destroyed)
        # 显示积分对话框的所有部件
        self._umit_credits_dialog.show_all()
# 创建一个名为 UmitCredits 的类，继承自 HIGWindow
class UmitCredits(HIGWindow):
    # 初始化方法
    def __init__(self):
        # 调用父类的初始化方法
        HIGWindow.__init__(self)
        # 设置窗口标题为 UMIT_DISPLAY_NAME 后面加上 " credits"
        self.set_title(_("%s credits") % UMIT_DISPLAY_NAME)
        # 设置窗口的大小请求为宽度不限制，高度为 250
        self.set_size_request(-1, 250)
        # 设置窗口的位置为居中
        self.set_position(Gtk.WindowPosition.CENTER)

        # 调用私有方法创建窗口部件
        self.__create_widgets()
        # 调用私有方法将窗口部件添加到布局中
        self.__packing()
        # 设置文本内容
        self.set_text()

    # 创建窗口部件的私有方法
    def __create_widgets(self):
        # 创建垂直布局容器
        self.vbox = HIGVBox()
        # 创建水平布局容器
        self.hbox = HIGHBox()
        # 创建笔记本容器
        self.notebook = HIGNotebook()
        # 创建一个带有关闭图标的按钮
        self.btn_close = HIGButton(stock=Gtk.STOCK_CLOSE)

        # 创建用于显示“written by”信息的滚动窗口
        self.written_by_scroll = HIGScrolledWindow()
        # 创建用于显示“written by”信息的文本视图
        self.written_by_text = HIGTextView()

        # 创建用于显示设计信息的滚动窗口
        self.design_scroll = HIGScrolledWindow()
        # 创建用于显示设计信息的文本视图
        self.design_text = HIGTextView()

        # 创建用于显示 soc2007 信息的滚动窗口
        self.soc2007_scroll = HIGScrolledWindow()
        # 创建用于显示 soc2007 信息的文本视图
        self.soc2007_text = HIGTextView()

        # 创建用于显示贡献者信息的滚动窗口
        self.contributors_scroll = HIGScrolledWindow()
        # 创建用于显示贡献者信息的文本视图
        self.contributors_text = HIGTextView()

        # 创建用于显示翻译信息的滚动窗口
        self.translation_scroll = HIGScrolledWindow()
        # 创建用于显示翻译信息的文本视图
        self.translation_text = HIGTextView()

        # 创建用于显示 nokia 信息的滚动窗口
        self.nokia_scroll = HIGScrolledWindow()
        # 创建用于显示 nokia 信息的文本视图
        self.nokia_text = HIGTextView()
    # 将 vbox 添加到当前对象中
    self.add(self.vbox)
    # 设置 vbox 中子部件的间距
    self.vbox.set_spacing(12)
    # 将 notebook 添加到 vbox 中并设置为可扩展和填充
    self.vbox._pack_expand_fill(self.notebook)
    # 将 hbox 添加到 vbox 中并设置为不可扩展和不填充
    self.vbox._pack_noexpand_nofill(self.hbox)

    # 将 hbox 中的占位符添加到 hbox 中并设置为可扩展和填充
    self.hbox._pack_expand_fill(hig_box_space_holder())
    # 将关闭按钮添加到 hbox 中并设置为不可扩展和不填充
    self.hbox._pack_noexpand_nofill(self.btn_close)

    # 在 notebook 中添加页面，并设置标签
    self.notebook.append_page(self.written_by_scroll, Gtk.Label.new(_("Written by")))
    self.notebook.append_page(self.design_scroll, Gtk.Label.new(_("Design")))
    self.notebook.append_page(self.soc2007_scroll, Gtk.Label.new("SoC 2007"))
    self.notebook.append_page(self.contributors_scroll, Gtk.Label.new(_("Contributors")))
    self.notebook.append_page(self.translation_scroll, Gtk.Label.new(_("Translation")))
    self.notebook.append_page(self.nokia_scroll, Gtk.Label.new("Maemo"))

    # 在 written_by_scroll 中添加 written_by_text，并设置文本换行模式
    self.written_by_scroll.add(self.written_by_text)
    self.written_by_text.set_wrap_mode(Gtk.WrapMode.NONE)

    # 在 design_scroll 中添加 design_text，并设置文本换行模式
    self.design_scroll.add(self.design_text)
    self.design_text.set_wrap_mode(Gtk.WrapMode.NONE)

    # 在 soc2007_scroll 中添加 soc2007_text，并设置文本换行模式
    self.soc2007_scroll.add(self.soc2007_text)
    self.soc2007_text.set_wrap_mode(Gtk.WrapMode.NONE)

    # 在 contributors_scroll 中添加 contributors_text，并设置文本换行模式
    self.contributors_scroll.add(self.contributors_text)
    self.contributors_text.set_wrap_mode(Gtk.WrapMode.NONE)

    # 在 translation_scroll 中添加 translation_text，并设置文本换行模式
    self.translation_scroll.add(self.translation_text)
    self.translation_text.set_wrap_mode(Gtk.WrapMode.NONE)

    # 在 nokia_scroll 中添加 nokia_text，并设置文本换行模式
    self.nokia_scroll.add(self.nokia_text)
    self.nokia_text.set_wrap_mode(Gtk.WrapMode.NONE)

    # 连接关闭按钮的点击事件，点击时销毁当前对象
    self.btn_close.connect('clicked', lambda x, y=None: self.destroy())

def set_text(self):
    # 获取 written_by_text 的缓冲区并设置文本
    b = self.written_by_text.get_buffer()
    b.set_text("""Adriano Monteiro Marques <py.adriano@gmail.com>""")

    # 获取 design_text 的缓冲区并设置文本
    b = self.design_text.get_buffer()
    b.set_text("""Operating System and Vulnerability Icons:
# 作者信息
Takeshi Alexandre Gondo <sinistrofumanchu@yahoo.com.br>

# Logo、应用程序图标和启动画面设计者信息
Virgílio Carlo de Menezes Vasconcelos <virgiliovasconcelos@gmail.com>

# Umit 项目网站设计者信息
Joao Paulo Pacheco <jp.pacheco@gmail.com>

# 设置 self.soc2007_text 的文本内容
b = self.soc2007_text.get_buffer()
b.set_text("""Independent Features:
Adriano Monteiro Marques <py.adriano@gmail.com>
Frederico Silva Ribeiro <fredegart@gmail.com>

Network Inventory:
Guilherme Henrique Polo Gonçalves <ggpolo@gmail.com>

Umit Radial Mapper:
João Paulo de Souza Medeiros <ignotus21@gmail.com>

Profile/Wizard interface editor:
Luis Antonio Bastião Silva <luis.kop@gmail.com>

NSE Facilitator:
Maxim I. Gavrilov <lovelymax@gmail.com>

Umit Web:
Rodolfo da Silva Carvalho <rodolfo.ueg@gmail.com>""")

# 设置 self.contributors_text 的文本内容
b = self.contributors_text.get_buffer()
b.set_text("""Sponsored by (SoC 2005, 2006 and 2007):
Google <code.summer@gmail.com>

Mentor of Umit for Google SoC 2005 and 2006:
Fyodor <fyodor@insecure.org>

Mentor of Umit for Google SoC 2007 Projects:
Adriano Monteiro Marques <py.adriano@gmail.com>

Initial development:
Adriano Monteiro Marques <py.adriano@gmail.com>
Cleber Rodrigues Rosa Junior <cleber.gnu@gmail.com>

Nmap students from Google SoC 2007 that helped Umit:
Eddie Bell <ejlbell@gmail.com>
David Fifield <david@bamsoftware.com>
Kris Katterjohn <katterjohn@gmail.com>

The Umit Project WebSite:
AbraoBarbosa dos Santos Neto <abraobsn@gmail.com>
Adriano Monteiro Marques <py.adriano@gmail.com>
Heitor de Lima Matos <heitordelima@hotmail.com>
Joao Paulo Pacheco <jp.pacheco@gmail.com>
João Paulo de Souza Medeiros <ignotus21@gmail.com>
Luis Antonio Bastião Silva <luis.kop@gmail.com>
Rodolfo da Silva Carvalho <rodolfo.ueg@gmail.com>

Beta testers for 0.9.5RC1:
Drew Miller <securitygeek@fribble.org>
Igor Feghali <ifeghali@php.net>
Joao Paulo Pacheco <jp.pacheco@gmail.com>
Luis Antonio Bastião Silva <luis.kop@gmail.com>
<ray-solomon@excite.com>
<jah@zadkiel.plus.com>
# 设置邮件地址
<epatterson@directapps.com>

# Maemo 端口的初始尝试
Adriano Monteiro Marques <py.adriano@gmail.com>
Osvaldo Santana Neto <osantana@gmail.com>

# 设置翻译文本的缓冲区内容
b = self.translation_text.get_buffer()
b.set_text("""Brazilian Portuguese:
Adriano Monteiro Marques <py.adriano@gmail.com>""")

# 设置诺基亚文本的缓冲区内容
b = self.nokia_text.get_buffer()
b.set_text("""Adriano Monteiro Marques <py.adriano@gmail.com>""")

# 如果作为独立程序运行，则显示关于窗口
if __name__ == '__main__':
    about = About()
    about.show()
    about.connect("response", lambda widget, response: Gtk.main_quit())

    # 运行主循环
    Gtk.main()
```