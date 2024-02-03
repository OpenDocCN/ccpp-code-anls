# `nmap\zenmap\zenmapGUI\ScanRunDetailsPage.py`

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
# 导入 gi 模块
import gi
# 要求使用 Gtk 3.0 版本
gi.require_version("Gtk", "3.0")
# 从 gi.repository 中导入 Gtk 模块
from gi.repository import Gtk
# 从 zenmapGUI.higwidgets.higboxes 模块中导入 HIGVBox, HIGHBox, hig_box_space_holder
from zenmapGUI.higwidgets.higboxes import HIGVBox, HIGHBox, hig_box_space_holder
# 从 zenmapGUI.higwidgets.higtables 模块中导入 HIGTable
from zenmapGUI.higwidgets.higtables import HIGTable
# 从 zenmapGUI.higwidgets.higlabels 模块中导入 HIGEntryLabel
from zenmapGUI.higwidgets.higlabels import HIGEntryLabel
# 导入 zenmapCore.I18N 模块
import zenmapCore.I18N  # lgtm[py/unused-import]

# 定义一个类 ScanRunDetailsPage，继承自 HIGVBox
class ScanRunDetailsPage(HIGVBox):
    # 从解析后的扫描结果初始化显示信息
    def _set_from_scan(self, scan):
        # 命令信息
        self.info_command_label.set_text(scan.get_nmap_command())  # 设置Nmap命令信息
        self.info_nmap_version_label.set_text(scan.get_scanner_version())  # 设置Nmap版本信息
        self.info_verbose_label.set_text(scan.get_verbose_level())  # 设置详细级别信息
        self.info_debug_label.set_text(scan.get_debugging_level())  # 设置调试级别信息

        # 一般信息
        self.info_start_label.set_text(scan.get_formatted_date())  # 设置扫描开始时间
        self.info_finished_label.set_text(scan.get_formatted_finish_date())  # 设置扫描结束时间
        self.info_hosts_up_label.set_text(str(scan.get_hosts_up()))  # 设置主机上线数量
        self.info_hosts_down_label.set_text(str(scan.get_hosts_down()))  # 设置主机下线数量
        self.info_hosts_scanned_label.set_text(str(scan.get_hosts_scanned()))  # 设置扫描主机数量
        self.info_open_label.set_text(str(scan.get_open_ports()))  # 设置开放端口数量
        self.info_filtered_label.set_text(str(scan.get_filtered_ports()))  # 设置过滤端口数量
        self.info_closed_label.set_text(str(scan.get_closed_ports()))  # 设置关闭端口数量

        # 遍历扫描信息列表
        for scaninfo in scan.get_scaninfo():
            # 创建可展开的标签
            exp = Gtk.Expander.new('<b>%s - %s</b>' % (
                _('Scan Info'), scaninfo['type'].capitalize()))
            exp.set_use_markup(True)

            # 创建扫描信息的显示
            display = self.make_scaninfo_display(scaninfo)

            # 将显示内容添加到可展开标签中
            exp.add(display)
            self._pack_noexpand_nofill(exp)  # 将可展开标签添加到布局中
    # 创建一个方法，用于生成显示扫描信息的小部件，包括扫描类型、协议、扫描端口数量和服务列表
    def make_scaninfo_display(self, scaninfo):
        """Return a widget displaying a scan's "scaninfo" information: type,
        protocol, number of scanned ports, and list of services."""
        # 创建一个水平布局容器
        hbox = HIGHBox()
        # 创建一个表格布局容器
        table = HIGTable()
        table.set_border_width(5)
        table.set_row_spacings(6)
        table.set_col_spacings(6)

        # 在表格中添加扫描类型的标签和值
        table.attach(HIGEntryLabel(_('Scan type:')), 0, 1, 0, 1)
        table.attach(HIGEntryLabel(scaninfo['type']), 1, 2, 0, 1)

        # 在表格中添加协议的标签和值
        table.attach(HIGEntryLabel(_('Protocol:')), 0, 1, 1, 2)
        table.attach(HIGEntryLabel(scaninfo['protocol']), 1, 2, 1, 2)

        # 在表格中添加扫描端口数量的标签和值
        table.attach(HIGEntryLabel(_('# scanned ports:')), 0, 1, 2, 3)
        table.attach(HIGEntryLabel(scaninfo['numservices']), 1, 2, 2, 3)

        # 在表格中添加服务列表的标签和值
        table.attach(HIGEntryLabel(_('Services:')), 0, 1, 3, 4)
        table.attach(
                self.make_services_display(scaninfo['services']), 1, 2, 3, 4)

        # 将表格添加到水平布局容器中
        hbox._pack_noexpand_nofill(hig_box_space_holder())
        hbox._pack_noexpand_nofill(table)

        # 返回水平布局容器
        return hbox

    # 创建一个方法，用于生成显示服务列表的小部件
    def make_services_display(self, services):
        """Return a widget displaying a list of services like
        1-1027,1029-1033,1040,1043,1050,1058-1059,1067-1068,1076,1080"""
        # 创建一个下拉框
        combo = Gtk.ComboBoxText()

        # 将服务列表拆分后逐个添加到下拉框中
        for i in services.split(","):
            combo.append_text(i)

        # 返回下拉框
        return combo
# 如果当前模块被直接执行，则执行以下代码
if __name__ == "__main__":
    # 导入 sys 模块
    import sys
    # 从 zenmapCore.NmapParser 模块中导入 NmapParser 类
    from zenmapCore.NmapParser import NmapParser

    # 从命令行参数中获取文件名
    filename = sys.argv[1]
    # 创建 NmapParser 实例
    parsed = NmapParser()
    # 解析给定文件
    parsed.parse_file(filename)
    # 创建 ScanRunDetailsPage 实例，传入解析结果
    run_details = ScanRunDetailsPage(parsed)
    # 创建 Gtk 窗口
    window = Gtk.Window()
    # 将 run_details 添加到窗口中
    window.add(run_details)
    # 关联窗口的关闭事件，当关闭窗口时退出 Gtk 主循环
    window.connect("delete-event", lambda *args: Gtk.main_quit())
    # 显示窗口及其所有子部件
    window.show_all()
    # 进入 Gtk 主循环
    Gtk.main()
```