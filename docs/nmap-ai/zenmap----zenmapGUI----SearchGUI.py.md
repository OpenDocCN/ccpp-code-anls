# `nmap\zenmap\zenmapGUI\SearchGUI.py`

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
# 导入gi模块，要求版本为"Gtk"的"3.0"
import gi
# 从gi.repository中导入Gtk模块
gi.require_version("Gtk", "3.0")
from gi.repository import Gtk

# 导入re模块和copy模块
import re
import copy

# 从zenmapGUI.higwidgets.higbuttons中导入HIGButton和HIGToggleButton类
from zenmapGUI.higwidgets.higbuttons import HIGButton, HIGToggleButton
# 从zenmapGUI.higwidgets.higboxes中导入HIGHBox类
from zenmapGUI.higwidgets.higboxes import HIGHBox
# 从zenmapGUI.higwidgets.higlabels中导入HIGSectionLabel和HintWindow类
from zenmapGUI.higwidgets.higlabels import HIGSectionLabel, HintWindow
# 从zenmapGUI.higwidgets.higdialogs中导入HIGAlertDialog类
from zenmapGUI.higwidgets.higdialogs import HIGAlertDialog

# 导入datetime模块
import datetime

# 从zenmapCore.Name中导入APP_DISPLAY_NAME
from zenmapCore.Name import APP_DISPLAY_NAME
# 导入zenmapCore.I18N模块
import zenmapCore.I18N  # lgtm[py/unused-import]
# 从zenmapCore.NmapOptions中导入split_quoted函数
from zenmapCore.NmapOptions import split_quoted
# 从zenmapCore.SearchResult中导入SearchDir、SearchDB和SearchDummy类
from zenmapCore.SearchResult import SearchDir, SearchDB, SearchDummy
# 从zenmapCore.UmitConf中导入SearchConfig类
from zenmapCore.UmitConf import SearchConfig

# 创建SearchConfig类的实例对象
search_config = SearchConfig()

# 定义SearchParser类
class SearchParser(object):
    # 这个类负责解析搜索字符串，并更新搜索字典（这个字典会传递给执行实际搜索的类）。它持有对SearchGUI对象的引用，用于访问其search_dict字典，以便所有字典处理都在这里执行。它还负责通过'dir:'操作符向SearchGUI对象添加额外的目录。
    
    class SearchParser:
        def __init__(self, search_gui, search_keywords):
            # 将search_gui参数赋给self.search_gui属性
            self.search_gui = search_gui
            # 将search_gui对象的search_dict字典赋给self.search_dict属性
            self.search_dict = search_gui.search_dict
    
            # 我们需要创建一个操作符->搜索关键词的映射，因为搜索输入字段和搜索类具有不同的语法。
            #
            # 注意：如果要添加一个SearchResult类未处理的新搜索关键词，应该在SearchResult类中添加一个新的match_CRITERIANAME方法。例如，如果想要一个"noodles"条件，需要创建SearchResult.match_noodles(self, noodles_string)方法。要了解搜索是如何执行的，可以从SearchResult.search()方法开始阅读。
            self.ops2keys = copy.deepcopy(search_keywords)
    
            # 这实际上不是一个操作符（见下文）
            self.ops2keys["dir"] = "dir"
    def update(self, search):
        """Updates the search dictionary by parsing the input string."""

        # 清空搜索字典中的残留键，并重新解析输入字符串。虽然慢，但并不是真正的慢操作。
        self.search_dict.clear()

        # 遍历分割后的搜索词
        for word in split_quoted(search):
            # 如果搜索词中包含冒号
            if word.find(":") != -1:
                # 将冒号左边的部分作为键，右边的部分作为值
                op, arg = word.split(":", 1)
                # 如果操作符在 ops2keys 中
                if op in self.ops2keys:
                    # 将值添加到对应键的列表中，如果键已存在
                    key = self.ops2keys[op]
                    if key in self.search_dict:
                        self.search_dict[key].append(arg)
                    else:
                        self.search_dict[key] = [arg]
            else:
                # 只是一个简单的关键词
                if "keyword" in self.search_dict:
                    self.search_dict["keyword"].append(word)
                else:
                    self.search_dict["keyword"] = [word]

        # 检查搜索字典中是否有任何 dir: 操作符，如果有，将它们添加到 search_gui 对象中，并从搜索字典中移除。
        # dir: 操作符并不是一个真正的操作符，它不需要被 SearchResult.search() 函数处理。它只是用来创建一个新的 SearchDir 对象，
        # 然后用于执行实际的搜索。
        self.search_gui.init_search_dirs(self.search_dict.pop("dir", []))
class SearchGUI(Gtk.Box, object):
    """This class is a VBox that holds the search entry field and buttons on
    top, and the results list on the bottom. The "Cancel" and "Open" buttons
    are a part of the SearchWindow class, not SearchGUI."""
    # 初始化搜索目录
    def init_search_dirs(self, dirs=[]):
        # 清空搜索目录
        self.search_dirs.clear()

        # 如果指定了目录，将其添加到搜索目录中
        conf_dir = self.options["directory"]
        if conf_dir:
            self.search_dirs[conf_dir] = SearchDir(
                    conf_dir, self.options["file_extension"])

        # 处理任何其他目录（通过dir:操作符添加）
        for dir in dirs:
            self.search_dirs[dir] = SearchDir(
                    dir, self.options["file_extension"])
    # 创建小部件
    def _create_widgets(self):
        # 搜索框和按钮
        self.search_top_hbox = HIGHBox()  # 创建水平布局
        self.search_label = HIGSectionLabel(_("Search:"))  # 创建搜索标签
        self.search_entry = Gtk.Entry()  # 创建搜索输入框
        self.expressions_btn = HIGToggleButton(  # 创建表达式按钮
                _("Expressions "), Gtk.STOCK_EDIT)

        # 快速参考工具提示按钮
        self.search_tooltip_btn = HIGButton(" ", Gtk.STOCK_INFO)  # 创建工具提示按钮

        # 表达式垂直框。仅在用户点击“表达式”后可见。表达式（如果有）应紧凑排列，以便不占用太多屏幕空间
        self.expr_vbox = Gtk.Box.new(Gtk.Orientation.VERTICAL, 0)  # 创建垂直布局

        # 结果部分
        self.result_list = Gtk.ListStore.new([str, str, int])  # 创建包含标题、日期、ID的列表存储
        self.result_view = Gtk.TreeView.new_with_model(self.result_list)  # 使用列表存储创建树视图
        self.result_scrolled = Gtk.ScrolledWindow()  # 创建滚动窗口
        self.result_title_column = Gtk.TreeViewColumn(title=_("Scan"))  # 创建标题列
        self.result_date_column = Gtk.TreeViewColumn(title=_("Date"))  # 创建日期列

        self.no_db_warning = Gtk.Label()  # 创建警告标签
        self.no_db_warning.set_line_wrap(True)  # 设置标签自动换行
        self.no_db_warning.set_no_show_all(True)  # 设置标签不在所有情况下都显示

        self.expr_window = None  # 初始化表达式窗口为None
    # 将标签、搜索框和按钮打包到水平框中
    self.search_top_hbox.set_spacing(4)
    self.search_top_hbox.pack_start(self.search_label, False, True, 0)
    self.search_top_hbox.pack_start(self.search_entry, True, True, 0)
    self.search_top_hbox.pack_start(self.expressions_btn, False, True, 0)
    self.search_top_hbox.pack_start(self.search_tooltip_btn, False, True, 0)

    # 将结果部分打包
    self.result_scrolled.add(self.result_view)
    self.result_scrolled.set_policy(
            Gtk.PolicyType.AUTOMATIC, Gtk.PolicyType.AUTOMATIC)

    # 将所有部分打包在一起
    self.set_spacing(4)
    self.pack_start(self.search_top_hbox, False, True, 0)
    self.pack_start(self.expr_vbox, False, True, 0)
    self.pack_start(self.result_scrolled, True, True, 0)
    self.pack_start(self.no_db_warning, False, True, 0)

def _connect_events(self):
    # 连接搜索框内容变化事件
    self.search_entry.connect("changed", self.update_search_entry)
    # 连接显示快速帮助按钮点击事件
    self.search_tooltip_btn.connect("clicked", self.show_quick_help)
    # 连接表达式按钮切换事件
    self.expressions_btn.connect("toggled", self.expressions_clicked)

def show_quick_help(self, widget=None, extra=None):
    # 显示快速帮助窗口
    hint_window = HintWindow(QUICK_HELP_TEXT)
    hint_window.show_all()

def close(self):
    # 如果表达式窗口不为空，则关闭表达式窗口
    if self.expr_window is not None:
        self.expr_window.close()

def add_criterion(self, caller):
    # 需要找到调用者（Criteria 对象）在所有行中的位置，以便我们可以在其后插入新行
    caller_index = self.expr_vbox.get_children().index(caller)

    # 创建一个新的 Criteria 行，并在调用行后插入它
    criteria = Criterion(self, "keyword")
    self.expr_vbox.pack_start(criteria, False, True, 0)
    self.expr_vbox.reorder_child(criteria, caller_index + 1)
    criteria.show_all()
    # 从对象中移除指定的条件
    def remove_criterion(self, c):
        # 如果表达式容器中的子元素数量大于1
        if len(self.expr_vbox.get_children()) > 1:
            # 销毁指定的条件对象
            c.destroy()
            # 调用条件改变的方法
            self.criterion_changed()
    
    # 当条件改变时调用的方法
    def criterion_changed(self):
        # 遍历所有条件行，生成新的搜索字符串
        search_string = ""
        for criterion in self.expr_vbox.get_children():
            # 如果条件的操作符不是"keyword"
            if criterion.operator != "keyword":
                search_string += criterion.operator + ":"
            search_string += criterion.argument.replace(" ", "") + " "
        
        # 设置搜索框的文本为生成的搜索字符串（去除首尾空格）
        self.search_entry.set_text(search_string.strip())
        
        # 更新搜索解析器的内容为搜索框的文本
        self.search_parser.update(self.search_entry.get_text())
        # 开始搜索
        self.start_search()
    
    # 添加搜索目录的方法
    def add_search_dir(self, dir):
        # 如果目录不在搜索目录中
        if dir not in self.search_dirs:
            # 将目录添加到搜索目录中
            self.search_dirs[dir] = SearchDir(dir, self.options["file_extension"])
    
    # 更新搜索框内容的方法
    def update_search_entry(self, widget, extra=None):
        """Called when the search entry field is modified."""
        # 更新搜索解析器的内容为搜索框的文本
        self.search_parser.update(widget.get_text())
        # 开始搜索
        self.start_search()
    # 启动搜索功能
    def start_search(self):
        # 如果没有选择搜索数据库和目录，则弹出警告对话框
        if not self.options["search_db"] and not self.options["directory"]:
            d = HIGAlertDialog(
                    message_format=_("No search method selected!"),
                    secondary_text=_(
                        "%s can search results on directories or inside its "
                        "own database. Please select a method by choosing a "
                        "directory or by checking the search data base option "
                        "in the 'Search options' tab before starting a search"
                        ) % APP_DISPLAY_NAME)
            d.run()
            d.destroy()
            return

        # 清空搜索结果列表
        self.clear_result_list()

        matched = 0
        total = 0
        # 如果选择了搜索数据库，则获取数据库扫描结果并进行搜索
        if self.options["search_db"]:
            total += len(self.search_db.get_scan_results())
            for result in self.search_db.search(**self.search_dict):
                self.append_result(result)
                matched += 1

        # 遍历搜索目录，获取扫描结果并进行搜索
        for search_dir in self.search_dirs.values():
            total += len(search_dir.get_scan_results())
            for result in search_dir.search(**self.search_dict):
                self.append_result(result)
                matched += 1

        # 更新搜索窗口的标签文本，显示匹配结果数量
        self.search_window.set_label_text(
                "Matched <b>%s</b> out of <b>%s</b> scans." % (
                    str(matched), str(total)))

    # 清空搜索结果列表
    def clear_result_list(self):
        # 遍历搜索结果列表，删除所有结果
        for i in range(len(self.result_list)):
            iter = self.result_list.get_iter_first()
            del(self.result_list[iter])
    # 将解析后的结果添加到结果列表中
    def append_result(self, parsed_result):
        # 获取扫描名称作为标题
        title = parsed_result.scan_name

        # 尝试将开始时间转换为日期时间格式，如果失败则设置为"Unknown"
        try:
            date = datetime.datetime.fromtimestamp(float(parsed_result.start))
            date_field = date.strftime("%Y-%m-%d %H:%M")
        except ValueError:
            date_field = _("Unknown")

        # 将解析结果添加到解析结果字典中
        self.parsed_results[self.id] = [title, parsed_result]
        # 将标题、日期和ID添加到结果列表中
        self.result_list.append([title, date_field, self.id])
        # 增加ID计数
        self.id += 1

    # 获取选定的结果
    def get_selected_results(self):
        # 获取结果视图的选择
        selection = self.result_view.get_selection()
        # 获取选定的行
        rows = selection.get_selected_rows()
        list_store = rows[0]

        results = {}
        # 遍历选定的行，将结果添加到结果字典中
        for row in rows[1]:
            r = row[0]
            results[list_store[r][2]] = self.parsed_results[list_store[r][2]]

        return results

    # 设置结果视图
    def _set_result_view(self):
        # 启用搜索功能
        self.result_view.set_enable_search(True)
        # 设置搜索列为第一列
        self.result_view.set_search_column(0)

        # 获取结果视图的选择
        selection = self.result_view.get_selection()
        # 设置选择模式为多选
        selection.set_mode(Gtk.SelectionMode.MULTIPLE)

        # 添加标题和日期列到结果视图
        self.result_view.append_column(self.result_title_column)
        self.result_view.append_column(self.result_date_column)

        # 设置标题列和日期列可调整大小
        self.result_title_column.set_resizable(True)
        self.result_title_column.set_min_width(200)
        self.result_date_column.set_resizable(True)

        # 设置标题列和日期列的排序列ID
        self.result_title_column.set_sort_column_id(0)
        self.result_date_column.set_sort_column_id(1)

        # 设置标题列和日期列可重新排序
        self.result_title_column.set_reorderable(True)
        self.result_date_column.set_reorderable(True)

        # 创建文本单元格渲染器
        cell = Gtk.CellRendererText()

        # 将文本单元格渲染器添加到标题列和日期列
        self.result_title_column.pack_start(cell, True)
        self.result_date_column.pack_start(cell, True)

        # 设置标题列和日期列的属性为文本
        self.result_title_column.set_attributes(cell, text=0)
        self.result_date_column.set_attributes(cell, text=1)

    # 获取选定的结果的属性
    selected_results = property(get_selected_results)
class Criterion(Gtk.Box):
    """This class holds one criterion row, represented as an HBox.  It holds a
    ComboBox and a Subcriterion's subclass instance, depending on the selected
    entry in the ComboBox. For example, when the 'Target' option is selected, a
    SimpleSubcriterion widget is displayed, but when the 'Date' operator is
    selected, a DateSubcriterion widget is displayed."""

    def __init__(self, search_window, operator="keyword", argument=""):
        """A reference to the search window is passed so that we can call
        add_criterion and remove_criterion."""
        # 调用父类的初始化方法，设置布局为水平方向
        Gtk.Box.__init__(self, orientation=Gtk.Orientation.HORIZONTAL)

        # 保存对搜索窗口的引用，以便调用 add_criterion 和 remove_criterion 方法
        self.search_window = search_window
        # 设置默认的操作符和参数
        self.default_operator = operator
        self.default_argument = argument

        # 将操作符和对应的参数列表保存为字典，以便传递给 SimpleSubcriterion 实例
        self.combo_entries = {"Keyword": ["keyword"],
                              "Profile Name": ["profile"],
                              "Target": ["target"],
                              "Options": ["option"],
                              "Date": ["date", "after", "before"],
                              "Operating System": ["os"],
                              "Port": ["open", "scanned", "closed", "filtered",
                                  "unfiltered", "open_filtered",
                                  "closed_filtered"],
                              "Service": ["service"],
                              "Host In Route": ["inroute"],
                              "Include Directory": ["dir"]}

        # 创建小部件
        self._create_widgets()
        # 将小部件添加到布局中
        self._pack_widgets()
        # 连接事件
        self._connect_events()
    # 创建小部件
    def _create_widgets(self):
        # 包含操作符列表的下拉框
        self.operator_combo = Gtk.ComboBoxText()

        # 对 combo_entries 中的所有键进行排序，并为每个键创建一个条目
        sorted_entries = list(self.combo_entries.keys())
        sorted_entries.sort()
        for name in sorted_entries:
            self.operator_combo.append_text(name)

        # 选择默认操作符
        for entry, operators in self.combo_entries.items():
            for operator in operators:
                if operator == self.default_operator:
                    self.operator_combo.set_active(sorted_entries.index(entry))
                    break

        # 创建子条件
        self.subcriterion = self.new_subcriterion(
                self.default_operator, self.default_argument)

        # "添加" 和 "移除" 按钮
        self.add_btn = HIGButton(" ", Gtk.STOCK_ADD)
        self.remove_btn = HIGButton(" ", Gtk.STOCK_REMOVE)

    # 将小部件打包
    def _pack_widgets(self):
        self.pack_start(self.operator_combo, False, True, 0)
        self.pack_start(self.subcriterion, True, True, 0)
        self.pack_start(self.add_btn, False, True, 0)
        self.pack_start(self.remove_btn, False, True, 0)

    # 连接事件
    def _connect_events(self):
        self.operator_combo.connect("changed", self.operator_changed)
        self.add_btn.connect("clicked", self.add_clicked)
        self.remove_btn.connect("clicked", self.remove_clicked)

    # 获取操作符
    def get_operator(self):
        return self.subcriterion.operator

    # 获取参数
    def get_argument(self):
        return self.subcriterion.argument

    # 点击添加按钮的事件处理函数
    def add_clicked(self, widget=None, extra=None):
        self.search_window.add_criterion(self)

    # 点击移除按钮的事件处理函数
    def remove_clicked(self, widget=None, extra=None):
        self.search_window.remove_criterion(self)
    def value_changed(self, op, arg):
        """Subcriterion instances call this method when something changes
        inside of them."""
        # 当子条件实例内部发生变化时，调用此方法
        # 通知搜索窗口有变化发生
        self.search_window.criterion_changed()

    def new_subcriterion(self, operator="keyword", argument=""):
        # 创建新的子条件
        if operator in self.combo_entries["Date"]:
            return DateSubcriterion(operator, argument)
        elif operator in self.combo_entries["Port"]:
            return PortSubcriterion(operator, argument)
        elif operator == "dir":
            return DirSubcriterion(operator, argument)
        else:
            return SimpleSubcriterion(operator, argument)

    def operator_changed(self, widget=None, extra=None):
        """This function is called when the user selects a different entry in
        the Criterion's ComboBox."""
        # 当用户在条件的下拉框中选择不同的条目时，调用此函数
        # 销毁先前的子条件
        self.subcriterion.destroy()

        # 根据选择的操作符创建新的子条件
        selected = self.operator_combo.get_active_text()
        operator = self.combo_entries[selected][0]
        self.subcriterion = self.new_subcriterion(operator)

        # 将新的子条件放置在下拉框右侧
        self.pack_start(self.subcriterion, True, True, 0)
        self.reorder_child(self.subcriterion, 1)

        # 通知搜索窗口有变化发生
        self.search_window.criterion_changed()

        # 显示所有子条件
        self.subcriterion.show_all()

    operator = property(get_operator)
    argument = property(get_argument)
class Subcriterion(Gtk.Box):
    """This class is a base class for all subcriterion types. Depending on the
    criterion selected in the Criterion's ComboBox, a subclass of Subcriterion
    is created to display the appropriate GUI."""
    def __init__(self):
        # 调用父类的初始化方法，设置布局为水平方向
        Gtk.Box.__init__(self, orientation=Gtk.Orientation.HORIZONTAL)

        # 初始化操作符和参数
        self.operator = ""
        self.argument = ""

    def value_changed(self):
        """Propagates the operator and the argument up to the Criterion
        parent."""
        # 将操作符和参数传递给父类的value_changed方法
        self.get_parent().value_changed(self.operator, self.argument)


class SimpleSubcriterion(Subcriterion):
    """This class represents all 'simple' criterion types that need only an
    entry box in order to define the criterion."""
    def __init__(self, operator="keyword", argument=""):
        # 调用父类的初始化方法
        Subcriterion.__init__(self)

        # 设置操作符和参数
        self.operator = operator
        self.argument = argument

        # 创建小部件
        self._create_widgets()
        # 将小部件添加到布局中
        self._pack_widgets()
        # 连接小部件的信号
        self._connect_widgets()

    def _create_widgets(self):
        # 创建一个文本输入框
        self.entry = Gtk.Entry()
        # 如果参数不为空，设置文本输入框的文本为参数值
        if self.argument:
            self.entry.set_text(self.argument)

    def _pack_widgets(self):
        # 将文本输入框添加到布局中
        self.pack_start(self.entry, True, True, 0)

    def _connect_widgets(self):
        # 连接文本输入框的"changed"信号到entry_changed方法
        self.entry.connect("changed", self.entry_changed)

    def entry_changed(self, widget=None, extra=None):
        # 获取文本输入框的文本并更新参数值
        self.argument = widget.get_text()
        # 调用value_changed方法
        self.value_changed()


class PortSubcriterion(Subcriterion):
    """This class shows the port criterion GUI."""
    def __init__(self, operator="open", argument=""):
        # 调用父类的初始化方法
        Subcriterion.__init__(self)

        # 设置操作符和参数
        self.operator = operator
        self.argument = argument

        # 创建小部件
        self._create_widgets()
        # 将小部件添加到布局中
        self._pack_widgets()
        # 连接小部件的信号
        self._connect_widgets()
    # 创建小部件
    def _create_widgets(self):
        # 创建一个文本输入框
        self.entry = Gtk.Entry()
        # 如果有参数，则设置文本输入框的文本为参数值
        if self.argument:
            self.entry.set_text(self.argument)
    
        # 创建一个标签
        self.label = Gtk.Label.new("  is  ")
    
        # 创建一个下拉框
        self.port_state_combo = Gtk.ComboBoxText()
        # 定义下拉框的选项
        states = ["open", "scanned", "closed", "filtered", "unfiltered",
                "open|filtered", "closed|filtered"]
        # 将选项添加到下拉框中
        for state in states:
            self.port_state_combo.append_text(state)
        # 设置下拉框的默认选项
        self.port_state_combo.set_active(
                states.index(self.operator.replace("_", "|")))
    
    # 将小部件添加到布局中
    def _pack_widgets(self):
        self.pack_start(self.entry, True, True, 0)
        self.pack_start(self.label, False, True, 0)
        self.pack_start(self.port_state_combo, False, True, 0)
    
    # 连接小部件的信号和回调函数
    def _connect_widgets(self):
        self.entry.connect("changed", self.entry_changed)
        self.port_state_combo.connect("changed", self.port_criterion_changed)
    
    # 文本输入框内容改变时的回调函数
    def entry_changed(self, widget=None, extra=None):
        # 获取文本输入框的内容，并赋值给参数
        self.argument = widget.get_text()
        # 调用值改变的函数
        self.value_changed()
    
    # 下拉框选项改变时的回调函数
    def port_criterion_changed(self, widget=None, extra=None):
        # 获取下拉框的选中项，并赋值给操作符
        self.operator = widget.get_active_text()
        # 调用值改变的函数
        self.value_changed()
# 定义一个名为 DirSubcriterion 的类，继承自 Subcriterion 类
class DirSubcriterion(Subcriterion):
    # 初始化方法，设置 operator 和 argument 属性
    def __init__(self, operator="dir", argument=""):
        # 调用父类的初始化方法
        Subcriterion.__init__(self)

        # 设置 operator 和 argument 属性
        self.operator = operator
        self.argument = argument

        # 创建小部件
        self._create_widgets()
        # 将小部件添加到布局中
        self._pack_widgets()
        # 连接小部件的信号和槽
        self._connect_widgets()

    # 创建小部件的方法
    def _create_widgets(self):
        # 创建一个文本输入框
        self.dir_entry = Gtk.Entry()
        # 如果 argument 存在，则设置文本输入框的文本为 argument
        if self.argument:
            self.dir_entry.set_text(self.argument)
        # 创建一个按钮
        self.chooser_btn = HIGButton("Choose...", Gtk.STOCK_OPEN)

    # 将小部件添加到布局中的方法
    def _pack_widgets(self):
        # 将文本输入框添加到布局中
        self.pack_start(self.dir_entry, True, True, 0)
        # 将按钮添加到布局中
        self.pack_start(self.chooser_btn, False, True, 0)

    # 连接小部件的方法
    def _connect_widgets(self):
        # 连接按钮的点击信号和 choose_clicked 方法
        self.chooser_btn.connect("clicked", self.choose_clicked)
        # 连接文本输入框的文本改变信号和 dir_entry_changed 方法
        self.dir_entry.connect("changed", self.dir_entry_changed)

    # 当按钮被点击时调用的方法
    def choose_clicked(self, widget=None, extra=None):
        # 显示一个目录选择对话框
        chooser_dlg = DirectoryChooserDialog("Include folder in search")

        # 如果用户点击了确定按钮
        if chooser_dlg.run() == Gtk.ResponseType.OK:
            # 设置文本输入框的文本为选择的目录
            self.dir_entry.set_text(chooser_dlg.get_filename())

        # 关闭对话框
        chooser_dlg.destroy()

    # 当文本输入框的文本改变时调用的方法
    def dir_entry_changed(self, widget=None, extra=None):
        # 更新 argument 属性为文本输入框的文本
        self.argument = widget.get_text()
        # 调用 value_changed 方法
        self.value_changed()
    # 初始化函数，设置默认操作符为日期，参数为空字符串
    def __init__(self, operator="date", argument=""):
        # 调用父类的初始化函数
        Subcriterion.__init__(self)

        # 定义文本操作符映射关系
        self.text2op = {"is": "date",
                        "after": "after",
                        "before": "before"}

        # 设置操作符为传入的操作符
        self.operator = operator

        # 创建小部件
        self._create_widgets()
        # 将小部件打包
        self._pack_widgets()
        # 连接小部件
        self._connect_widgets()

        # 计算模糊操作符的数量，以便稍后将它们附加到参数中
        self.fuzzies = argument.count("~")
        # 移除参数中的模糊操作符
        argument = argument.replace("~", "")
        # 初始化减号标记为 False
        self.minus_notation = False
        # 如果参数匹配日期格式，则解析日期
        if re.match(r"\d\d\d\d-\d\d-\d\d$", argument) is not None:
            year, month, day = argument.split("-")
            self.date = datetime.date(int(year), int(month), int(day))
            self.argument = argument
        # 如果参数匹配"-n"格式，则将其转换为YYYY-MM-DD格式
        elif re.match(r"[-|\+]\d+$", argument) is not None:
            parsed_date = datetime.date.fromordinal(
                    datetime.date.today().toordinal() + int(argument))
            self.argument = argument
            self.date = datetime.date(
                    parsed_date.year, parsed_date.month, parsed_date.day)

            self.minus_notation = True
        # 如果参数不匹配任何格式，则使用当前日期作为参数
        else:
            self.date = datetime.date.today()
            self.argument = self.date.isoformat()

        # 如果有模糊操作符，则将其附加到参数中
        self.argument += "~" * self.fuzzies

    # 创建小部件的函数
    def _create_widgets(self):
        # 创建日期条件下拉框
        self.date_criterion_combo = Gtk.ComboBoxText()
        self.date_criterion_combo.append_text("is")
        self.date_criterion_combo.append_text("after")
        self.date_criterion_combo.append_text("before")
        # 根据操作符设置默认选中的条件
        if self.operator == "date":
            self.date_criterion_combo.set_active(0)
        elif self.operator == "after":
            self.date_criterion_combo.set_active(1)
        else:
            self.date_criterion_combo.set_active(2)
        # 创建日期按钮
        self.date_button = HIGButton()
    # 将日期选择框和日期按钮添加到父容器中
    def _pack_widgets(self):
        self.pack_start(self.date_criterion_combo, False, True, 0)
        self.pack_start(self.date_button, True, True, 0)

    # 连接日期选择框和日期按钮的信号与回调函数
    def _connect_widgets(self):
        self.date_criterion_combo.connect(
                "changed", self.date_criterion_changed)
        self.date_button.connect("clicked", self.show_calendar)

    # 当日期选择框的数值改变时触发的回调函数
    def date_criterion_changed(self, widget=None, extra=None):
        self.operator = self.text2op[widget.get_active_text()]

        # 通知父类操作符已经改变
        self.value_changed()

    # 点击日期按钮时显示日历的回调函数
    def show_calendar(self, widget):
        calendar = DateCalendar()
        calendar.connect_calendar(self.update_button)
        calendar.show_all()

    # 更新日期按钮的回调函数
    def update_button(self, widget):
        cal_date = widget.get_date()
        # 将月份加1，因为 Gtk.Calendar 的日期是从0开始的
        self.date = datetime.date(cal_date[0], cal_date[1] + 1, cal_date[2])

        # 根据选择的日期设置参数，使用搜索格式
        if self.minus_notation:
            # 需要计算日期相对于今天的偏移量，以便表示为“-n”表示法
            today = datetime.date.today()
            offset = self.date.toordinal() - today.toordinal()
            if offset > 0:
                self.argument = "+" + str(offset)
            else:
                self.argument = str(offset)
        else:
            self.argument = self.date.isoformat()
        self.argument += "~" * self.fuzzies

        # 通知父类有关变化
        self.value_changed()

    # 设置日期并更新日期按钮的标签
    def set_date(self, date):
        self.date_button.set_label(date.strftime("%d %b %Y"))
        self._date = date

    # 获取日期
    def get_date(self):
        return self._date

    # 使用 property 方法将日期设置和获取方法绑定到 date 属性上
    date = property(get_date, set_date)
    # 初始化 _date 属性为今天的日期
    _date = datetime.date.today()
# 创建一个名为 DateCalendar 的类，继承自 Gtk.Window 和 object
class DateCalendar(Gtk.Window, object):
    # 初始化方法
    def __init__(self):
        # 调用父类的初始化方法，设置窗口类型为弹出窗口
        Gtk.Window.__init__(self, type=Gtk.WindowType.POPUP)
        # 设置窗口位置为鼠标位置
        self.set_position(Gtk.WindowPosition.MOUSE)

        # 创建一个日历对象
        self.calendar = Gtk.Calendar()
        # 将日历对象添加到窗口中
        self.add(self.calendar)

    # 连接日历对象的信号，当双击选择日期时执行指定的回调函数
    def connect_calendar(self, update_button_cb):
        self.calendar.connect("day-selected-double-click",
                              self.kill_calendar, update_button_cb)

    # 关闭日历窗口的方法，执行指定的回调函数后关闭窗口
    def kill_calendar(self, widget, method):
        method(widget)
        self.destroy()

# 快速帮助文本，包含关于搜索和操作符的说明
QUICK_HELP_TEXT = _("""\
Entering the text into the search performs a <b>keyword search</b> - the \
search string is matched against the entire output of each scan.

To refine the search, you can use <b>operators</b> to search only within \
a specific part of a scan. Operators can be added to the search \
interactively if you click on the <b>Expressions</b> button, or you can \
enter them manually into the search field. Most operators have a short \
form, listed.

<b>profile: (pr:)</b> - Profile used.
<b>target: (t:)</b> - User-supplied target, or a rDNS result.
<b>option: (o:)</b> - Scan options.
<b>date: (d:)</b> - The date when scan was performed. Fuzzy matching is \
possible using the "~" suffix. Each "~" broadens the search by one day \
on "each side" of the date. In addition, it is possible to use the \
\"date:-n\" notation which means "n days ago".
<b>after: (a:)</b> - Matches scans made after the supplied date \
(<i>YYYY-MM-DD</i> or <i>-n</i>).
<b>before (b:)</b> - Matches scans made before the supplied \
date(<i>YYYY-MM-DD</i> or <i>-n</i>).
<b>os:</b> - All OS-related fields.
<b>scanned: (sp:)</b> - Matches a port if it was among those scanned.
<b>open: (op:)</b> - Open ports discovered in a scan.
<b>closed: (cp:)</b> - Closed ports discovered in a scan.
<b>filtered: (fp:)</b> - Filtered ports discovered in scan.
<b>unfiltered: (ufp:)</b> - Unfiltered ports found in a scan (using, for \
example, an ACK scan).
# 定义变量 open|filtered 用于存储处于 "open|filtered" 状态的端口
open|filtered: (ofp:)
# 定义变量 closed|filtered 用于存储处于 "closed|filtered" 状态的端口
closed|filtered: (cfp:)
# 定义变量 service 用于存储所有与服务相关的字段
service: (s:)
# 定义变量 inroute 用于匹配扫描的跟踪路由输出中的路由器
inroute: (ir:)
```