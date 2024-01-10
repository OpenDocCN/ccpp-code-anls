# `nmap\zenmap\radialnet\bestwidgets\boxes.py`

```
# 设置文件编码为 UTF-8
# 重要提示：NMAP 许可证条款
# Nmap 安全扫描器由 Nmap 软件有限责任公司（“Nmap 项目”）（C）1996-2023 年开发。Nmap 也是 Nmap 项目的注册商标。
# 本程序根据 Nmap 公共源代码许可证（NPSL）的条款分发。适用于特定 Nmap 版本或源代码控制修订版本的确切许可证文本包含在该版本的 Nmap 或源代码控制修订版本的 LICENSE 文件中。有关更多 Nmap 版权/法律信息，请访问 https://nmap.org/book/man-legal.html，有关 NPSL 许可证本身的更多信息，请访问 https://nmap.org/npsl/。此标题总结了 Nmap 许可证的一些关键点，但不能替代实际的许可证文本。
# 一般情况下，用户可以免费下载和使用 Nmap，包括商业用途。可从 https://nmap.org 获取。
# Nmap 许可证一般禁止公司在商业产品中使用和重新分发 Nmap，但我们销售一个特殊的 Nmap OEM 版本，具有更宽松的许可证和特殊功能，用于此目的。请参阅 https://nmap.org/oem/
# 如果您收到了书面的 Nmap 许可协议或合同，其中规定了与这些条款（例如 Nmap OEM 许可证）不同的条款，您可以选择根据这些条款使用和重新分发 Nmap。
# 官方的 Nmap Windows 构建包括 Npcap 软件（https://npcap.com）用于数据包捕获和传输。它受到禁止未经特殊许可证重新分发的单独许可证条款的约束。因此，官方的 Nmap Windows 构建可能不得未经特殊许可证（例如 Nmap OEM 许可证）重新分发。
# 我们提供此软件的源代码，因为我们相信用户有权根据自己的需求修改和重新分发软件。因此，我们希望用户能够了解并遵守 Nmap 许可证的条款。
# 导入 gi 模块
import gi

# 要求使用 Gtk 3.0 版本
gi.require_version("Gtk", "3.0")
# 从 gi.repository 中导入 Gtk 模块
from gi.repository import Gtk

# 定义一个包含以下类的列表
__all__ = ('BWBox', 'BWHBox', 'BWVBox',
        'BWStatusbar', 'BWTable', 'BWScrolledWindow')


# 创建一个继承自 Gtk.Box 的类 BWBox
class BWBox(Gtk.Box):
    """
    """

    # 在 BWBox 类中定义一个方法 bw_pack_start_expand_fill
    def bw_pack_start_expand_fill(self, widget, padding=0):
        """
        """
        # 调用 Gtk.Box 的 pack_start 方法，设置 expand 和 fill 为 True
        self.pack_start(widget, True, True, padding)

    # 在 BWBox 类中定义一个方法 bw_pack_start_expand_nofill
    def bw_pack_start_expand_nofill(self, widget, padding=0):
        """
        """
        # 调用 Gtk.Box 的 pack_start 方法，设置 expand 为 True，fill 为 False
        self.pack_start(widget, True, False, padding)

    # 在 BWBox 类中定义一个方法 bw_pack_start_noexpand_nofill
    def bw_pack_start_noexpand_nofill(self, widget, padding=0):
        """
        """
        # 调用 Gtk.Box 的 pack_start 方法，设置 expand 和 fill 为 False
        self.pack_start(widget, False, False, padding)
    # 在容器的末尾添加一个可扩展的部件，并填充指定的填充量
    def bw_pack_end_expand_fill(self, widget, padding=0):
        # 调用容器的pack_end方法，将部件添加到末尾，设置为可扩展，填充指定的填充量
        self.pack_end(widget, True, True, padding)
    
    # 在容器的末尾添加一个可扩展的部件，但不填充
    def bw_pack_end_expand_nofill(self, widget, padding=0):
        # 调用容器的pack_end方法，将部件添加到末尾，设置为可扩展，但不填充
        self.pack_end(widget, True, False, padding)
    
    # 在容器的末尾添加一个不可扩展的部件，且不填充
    def bw_pack_end_noexpand_nofill(self, widget, padding=0):
        # 调用容器的pack_end方法，将部件添加到末尾，设置为不可扩展，且不填充
        self.pack_end(widget, False, False, padding)
# 创建一个水平方向的框
class BWHBox(BWBox):
    """
    """
    # 初始化方法，设置是否均匀分布和间距
    def __init__(self, homogeneous=False, spacing=12):
        """
        """
        # 调用父类的初始化方法，设置方向为水平，均匀分布和间距
        Gtk.Box.__init__(self, orientation=Gtk.Orientation.HORIZONTAL,
                         homogeneous=homogeneous, spacing=spacing)


# 创建一个垂直方向的框
class BWVBox(BWBox):
    """
    """
    # 初始化方法，设置是否均匀分布和间距
    def __init__(self, homogeneous=False, spacing=12):
        """
        """
        # 调用父类的初始化方法，设置方向为垂直，均匀分布和间距
        Gtk.Box.__init__(self, orientation=Gtk.Orientation.VERTICAL,
                         homogeneous=homogeneous, spacing=spacing)


# 创建一个状态栏
class BWStatusbar(Gtk.Statusbar, BWBox):
    """
    """
    # 初始化方法，设置是否均匀分布和间距
    def __init__(self, homogeneous=False, spacing=12):
        """
        """
        # 调用父类的初始化方法，设置方向为水平，均匀分布和间距
        Gtk.Box.__init__(self, orientation=Gtk.Orientation.HORIZONTAL,
                         homogeneous=homogeneous, spacing=spacing)


# 创建一个表格
class BWTable(Gtk.Table, BWBox):
    """
    """
    # 初始化方法，设置行数、列数和是否均匀分布
    def __init__(self, rows=1, columns=1, homogeneous=False):
        """
        """
        # 调用父类的初始化方法，设置行数、列数和是否均匀分布
        Gtk.Table.__init__(self, n_rows=rows, n_columns=columns,
                           homogeneous=homogeneous)
        # 设置间距为12
        self.bw_set_spacing(12)

        # 设置私有属性行数和列数
        self.__rows = rows
        self.__columns = columns

        # 设置私有属性最后一个点的位置
        self.__last_point = (0, 0)

    # 设置间距的方法
    def bw_set_spacing(self, spacing):
        """
        """
        # 设置行间距和列间距
        self.set_row_spacings(spacing)
        self.set_col_spacings(spacing)

    # 调整表格大小的方法
    def bw_resize(self, rows, columns):
        """
        """
        # 设置私有属性行数和列数
        self.__rows = rows
        self.__columns = columns

        # 调整表格大小
        self.resize(rows, columns)
    # 在表格布局中添加下一个子部件，并指定其在表格中的布局选项和填充
    def bw_attach_next(self,
                       child,
                       xoptions=Gtk.AttachOptions.EXPAND | Gtk.AttachOptions.FILL,
                       yoptions=Gtk.AttachOptions.EXPAND | Gtk.AttachOptions.FILL,
                       xpadding=0,
                       ypadding=0):
        """
        """
        # 获取上一个子部件的位置
        row, column = self.__last_point

        # 如果当前行不是最后一行
        if row != self.__rows:

            # 将子部件添加到表格布局中，并指定其位置、布局选项和填充
            self.attach(child,
                        column,
                        column + 1,
                        row,
                        row + 1,
                        xoptions,
                        yoptions,
                        xpadding,
                        ypadding)

            # 如果当前列是最后一列
            if column + 1 == self.__columns:

                # 列归零，行加一
                column = 0
                row += 1

            else:
                # 列加一
                column += 1

            # 更新上一个子部件的位置
            self.__last_point = (row, column)
# 创建一个名为BWScrolledWindow的类，继承自Gtk.ScrolledWindow
class BWScrolledWindow(Gtk.ScrolledWindow):
    """
    """

    # 初始化方法，用于创建BWScrolledWindow对象
    def __init__(self):
        """
        """
        # 调用父类的初始化方法
        Gtk.ScrolledWindow.__init__(self)
        # 设置水平和垂直滚动条的显示策略为自动
        self.set_policy(Gtk.PolicyType.AUTOMATIC, Gtk.PolicyType.AUTOMATIC)
        # 设置阴影类型为无
        self.set_shadow_type(Gtk.ShadowType.NONE)
        # 设置边框宽度为6
        self.set_border_width(6)
```