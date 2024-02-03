# `nmap\zenmap\zenmapGUI\ScanInterface.py`

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
# 导入 gi 模块，确保使用的是 Gtk 3.0 版本
import gi

# 要求使用 Gtk 3.0 版本
gi.require_version("Gtk", "3.0")
# 从 gi.repository 中导入 Gtk 和 GLib 模块
from gi.repository import Gtk, GLib

# 导入 errno、os、time 模块
import errno
import os
import time

# 防止加载 PyXML 模块
# 将 xml.__path__ 中不包含 "_xmlplus" 的路径重新赋值给 xml.__path__
import xml
xml.__path__ = [x for x in xml.__path__ if "_xmlplus" not in x]

# 导入 xml.sax 模块
import xml.sax

# 从 zenmapGUI.higwidgets.hignotebooks 模块中导入 HIGNotebook
# 从 zenmapGUI.higwidgets.higboxes 模块中导入 HIGVBox
# 从 zenmapGUI.higwidgets.higdialogs 模块中导入 HIGAlertDialog
# 从 zenmapGUI.higwidgets.higscrollers 模块中导入 HIGScrolledWindow
from zenmapGUI.higwidgets.hignotebooks import HIGNotebook
from zenmapGUI.higwidgets.higboxes import HIGVBox
from zenmapGUI.higwidgets.higdialogs import HIGAlertDialog
from zenmapGUI.higwidgets.higscrollers import HIGScrolledWindow

# 从 zenmapGUI 模块中导入 FilterBar、ScanHostDetailsPage、ScanCommandToolbar、ScanToolbar、ScanHostsView、ScanOpenPortsPage、ScanNmapOutputPage
from zenmapGUI.FilterBar import FilterBar
from zenmapGUI.ScanHostDetailsPage import ScanHostDetailsPage
from zenmapGUI.ScanToolbar import ScanCommandToolbar, ScanToolbar
from zenmapGUI.ScanHostsView import ScanHostsView
from zenmapGUI.ScanOpenPortsPage import ScanOpenPortsPage
from zenmapGUI.ScanNmapOutputPage import ScanNmapOutputPage
# 从 zenmapGUI 中导入 ScanScanListPage、ScansListStore 和 TopologyPage 类
from zenmapGUI.ScanScanListPage import ScanScanListPage
from zenmapGUI.ScansListStore import ScansListStore
from zenmapGUI.TopologyPage import TopologyPage

# 从 zenmapCore 中导入 NetworkInventory、FilteredNetworkInventory、NmapCommand、CommandProfile、is_maemo、NmapParser、Path、get_extra_executable_search_paths、log、NmapOptions、split_quoted、join_quoted 和 I18N 类/函数
from zenmapCore.NetworkInventory import NetworkInventory, FilteredNetworkInventory
from zenmapCore.NmapCommand import NmapCommand
from zenmapCore.UmitConf import CommandProfile, is_maemo
from zenmapCore.NmapParser import NmapParser
from zenmapCore.Paths import Path, get_extra_executable_search_paths
from zenmapCore.UmitLogging import log
from zenmapCore.NmapOptions import NmapOptions, split_quoted, join_quoted
import zenmapCore.I18N  # lgtm[py/unused-import]

# 定义 NMAP_OUTPUT_REFRESH_INTERVAL 常量，表示 live output view 刷新的时间间隔，单位为毫秒
NMAP_OUTPUT_REFRESH_INTERVAL = 1000

# 定义 ScanInterface 类，继承自 HIGVBox 类
class ScanInterface(HIGVBox):
    """ScanInterface contains the scan toolbar and the scan results. Each
    ScanInterface represents a single NetworkInventory as well as a set of
    running scans."""

    # 定义 FILTER_DELAY 常量，表示在停止输入过滤字符串后，开始过滤的延迟时间，单位为毫秒
    FILTER_DELAY = 1000

    # 定义 toggle_filter_bar 方法，用于切换过滤栏的显示状态
    def toggle_filter_bar(self):
        self.scan_result.filter_toggle_button.clicked()

    # 定义 filter_toggle_toggled 方法，用于处理过滤栏切换按钮的状态变化
    def filter_toggle_toggled(self, widget):
        if self.scan_result.filter_toggle_button.get_active():
            # 显示过滤栏
            self.filter_bar.show()
            self.filter_bar.grab_focus()
            self.filter_hosts(self.filter_bar.get_filter_string())
        else:
            # 隐藏过滤栏
            self.filter_bar.hide()
            self.filter_hosts("")

        # 更新界面
        self.update_ui()

    # 定义 filter_changed 方法，用于处理过滤栏内容变化的事件
    def filter_changed(self, filter_bar):
        # 重新启动定时器以开始过滤
        if self.filter_timeout_id:
            GLib.source_remove(self.filter_timeout_id)
        self.filter_timeout_id = GLib.timeout_add(
                self.FILTER_DELAY, self.filter_hosts,
                filter_bar.get_filter_string())
    # 过滤主机列表，根据过滤条件字符串进行过滤
    def filter_hosts(self, filter_string):
        # 记录开始时间
        start = time.perf_counter()
        # 应用过滤条件到主机清单
        self.inventory.apply_filter(filter_string)
        # 计算过滤时间
        filter_time = time.perf_counter() - start
        # 更新 GUI
        start = time.perf_counter()
        self.update_ui()
        # 计算 GUI 更新时间
        gui_time = time.perf_counter() - start

        # 如果过滤时间和 GUI 更新时间大于 0.0
        if filter_time + gui_time > 0.0:
            # 记录调试信息
            log.debug("apply_filter %g ms  update_ui %g ms (%.0f%% filter)" %
                (filter_time * 1000.0, gui_time * 1000.0,
                100.0 * filter_time / (filter_time + gui_time)))

        # 重置过滤超时 ID
        self.filter_timeout_id = None
        # 返回 False
        return False

    # 检查窗口是否有未保存的更改，有则返回 True，否则返回 False
    def is_changed(self):
        for scan in self.inventory.get_scans():
            if scan.unsaved:
                return True
        return False
    # 创建 changed 属性，用于检查窗口是否有未保存的更改
    changed = property(is_changed)

    # 返回正在运行的扫描数
    def num_scans_running(self):
        return len(self.jobs)

    # 选择默认配置文件
    def select_default_profile(self):
        """Select a "default" profile. Currently this is defined to be the
        first profile."""
        # 如果工具栏中有配置文件
        if len(self.toolbar.profile_entry.get_model()) > 0:
            # 设置第一个配置文件为活动配置文件
            self.toolbar.profile_entry.set_active(0)

    # 滚动文本输出到指定主机的位置
    def go_to_host(self, hostname):
        """Scroll the text output to the appearance of the named host."""
        # 调用 scan_result_notebook.nmap_output.nmap_output 对象的 go_to_host 方法
        self.scan_result.scan_result_notebook.nmap_output.nmap_output.go_to_host(hostname)  # noqa

    # 创建工具栏
    def __create_toolbar(self):
        # 实例化 ScanToolbar 类，创建工具栏对象
        self.toolbar = ScanToolbar()

        # 连接目标输入框的 'changed' 信号到 _target_entry_changed 方法
        self.target_entry_changed_handler = self.toolbar.target_entry.connect(
                'changed', self._target_entry_changed)
        # 连接配置文件下拉框的 'changed' 信号到 _profile_entry_changed 方法
        self.profile_entry_changed_handler = \
            self.toolbar.profile_entry.connect(
                    'changed', self._profile_entry_changed)

        # 连接扫描按钮的 'clicked' 信号到 start_scan_cb 方法
        self.toolbar.scan_button.connect('clicked', self.start_scan_cb)
        # 连接取消按钮的 'clicked' 信号到 _cancel_scan_cb 方法
        self.toolbar.cancel_button.connect('clicked', self._cancel_scan_cb)
    # 创建命令工具栏
    def __create_command_toolbar(self):
        # 实例化ScanCommandToolbar类，创建命令工具栏
        self.command_toolbar = ScanCommandToolbar()
        # 将命令条目的'activate'信号连接到扫描按钮的点击事件
        self.command_toolbar.command_entry.connect(
                'activate', lambda x: self.toolbar.scan_button.clicked())
        # 连接命令条目的'changed'信号到_command_entry_changed方法
        self.command_entry_changed_handler = \
            self.command_toolbar.command_entry.connect(
                    'changed', self._command_entry_changed)

    # 命令条目改变时的处理方法
    def _command_entry_changed(self, editable):
        # 实例化NmapOptions类
        ops = NmapOptions()
        # 解析命令工具栏中的命令字符串
        ops.parse_string(self.command_toolbar.get_command())

        # 设置目标和配置文件，不触发"changed"信号
        self.set_target_quiet(join_quoted(ops.target_specs))
        self.set_profile_name_quiet("")

    # 目标条目改变时的处理方法
    def _target_entry_changed(self, editable):
        # 获取选定的目标字符串
        target_string = self.toolbar.get_selected_target()
        # 将目标字符串分割成列表
        targets = split_quoted(target_string)

        # 实例化NmapOptions类
        ops = NmapOptions()
        # 解析命令工具栏中的命令字符串
        ops.parse_string(self.command_toolbar.get_command())
        # 将目标列表赋值给NmapOptions对象的target_specs属性
        ops.target_specs = targets
        # 设置命令字符串，不触发"changed"信号
        self.set_command_quiet(ops.render_string())
    # 当目标和配置文件发生变化时更新命令
    def _profile_entry_changed(self, widget):
        """Update the command based on the contents of the target and profile
        entries. If the command corresponding to the current profile is not
        blank, use it. Otherwise use the current contents of the command
        entry."""
        # 获取当前选择的配置文件名和目标字符串
        profile_name = self.toolbar.get_selected_profile()
        target_string = self.toolbar.get_selected_target()

        # 创建命令配置文件对象，并获取当前配置文件对应的命令字符串
        cmd_profile = CommandProfile()
        command_string = cmd_profile.get_command(profile_name)
        del(cmd_profile)
        # 如果命令字符串为空，则使用命令栏中的命令
        if command_string == "":
            command_string = self.command_toolbar.get_command()

        # 创建 NmapOptions 对象，并解析命令字符串
        ops = NmapOptions()
        ops.parse_string(command_string)

        # 如果命令栏中有目标，则使用命令栏中的目标，否则使用配置文件中的目标
        targets = split_quoted(target_string)
        if len(targets) > 0:
            ops.target_specs = targets
        else:
            self.toolbar.set_selected_target(join_quoted(ops.target_specs))

        # 设置命令字符串
        self.set_command_quiet(ops.render_string())

    # 设置命令字符串，忽略任何进一步的“changed”信号
    def set_command_quiet(self, command_string):
        """Set the command used by this scan interface, ignoring any further
        "changed" signals."""
        self.command_toolbar.command_entry.handler_block(
                self.command_entry_changed_handler)
        self.command_toolbar.set_command(command_string)
        self.command_toolbar.command_entry.handler_unblock(
                self.command_entry_changed_handler)

    # 设置目标字符串，忽略任何进一步的“changed”信号
    def set_target_quiet(self, target_string):
        """Set the target string used by this scan interface, ignoring any
        further "changed" signals."""
        self.toolbar.target_entry.handler_block(
                self.target_entry_changed_handler)
        self.toolbar.set_selected_target(target_string)
        self.toolbar.target_entry.handler_unblock(
                self.target_entry_changed_handler)
    # 设置扫描接口使用的配置文件名，忽略任何后续的“changed”信号
    def set_profile_name_quiet(self, profile_name):
        # 阻塞 profile_entry_changed_handler 信号处理器
        self.toolbar.profile_entry.handler_block(
                self.profile_entry_changed_handler)
        # 设置选定的配置文件
        self.toolbar.set_selected_profile(profile_name)
        # 解除对 profile_entry_changed_handler 信号处理器的阻塞
        self.toolbar.profile_entry.handler_unblock(
                self.profile_entry_changed_handler)

    # 开始扫描回调函数
    def start_scan_cb(self, widget=None):
        # 获取选定的目标
        target = self.toolbar.selected_target
        # 获取命令工具栏的命令
        command = self.command_toolbar.command
        # 获取选定的配置文件
        profile = self.toolbar.selected_profile

        # 输出调试信息
        log.debug(">>> Start Scan:")
        log.debug(">>> Target: '%s'" % target)
        log.debug(">>> Profile: '%s'" % profile)
        log.debug(">>> Command: '%s'" % command)

        # 如果目标不为空
        if target != '':
            # 尝试添加新目标
            try:
                self.toolbar.add_new_target(target)
            # 捕获 IOError 异常
            except IOError as e:
                # 处理无法保存 target_list.txt 的情况，将其视为只读
                # 可能是由 root 拥有，而当前是普通用户
                log.debug(">>> Error saving %s: %s" % (
                    Path.target_list, str(e)))

        # 如果命令为空
        if command == '':
            # 弹出警告对话框
            warn_dialog = HIGAlertDialog(
                    message_format=_("Empty Nmap Command"),
                    secondary_text=_("There is no command to execute. "
                        "Maybe the selected/typed profile doesn't exist. "
                        "Please check the profile name or type the nmap "
                        "command you would like to execute."),
                    type=Gtk.MessageType.ERROR)
            warn_dialog.run()
            warn_dialog.destroy()
            return

        # 执行命令
        self.execute_command(command, target, profile)

    # 显示的扫描更改回调函数
    def _displayed_scan_change_cb(self, widget):
        # 更新取消按钮
        self.update_cancel_button()
    # 更新取消按钮的状态，取决于当前显示的扫描是否正在运行
    def update_cancel_button(self):
        # 获取当前活动的扫描结果
        entry = self.scan_result.scan_result_notebook.nmap_output.get_active_entry()  # noqa
        # 如果没有活动的扫描结果，则将取消按钮设置为不可用
        if entry is None:
            self.toolbar.cancel_button.set_sensitive(False)
        else:
            # 根据活动的扫描结果的运行状态设置取消按钮的可用性
            self.toolbar.cancel_button.set_sensitive(entry.running)

    # 取消显示的扫描结果
    def _cancel_scan_cb(self, widget):
        # 获取当前活动的扫描结果
        entry = self.scan_result.scan_result_notebook.nmap_output.get_active_entry()  # noqa
        # 如果有活动的扫描结果且正在运行，则取消该扫描
        if entry is not None and entry.running:
            self.cancel_scan(entry.command)

    # 取消当前选定的扫描
    def _cancel_scans_list_cb(self, widget):
        # 获取当前选定的扫描列表
        model, selection = self.scan_result.scan_result_notebook.scans_list.scans_list.get_selection().get_selected_rows()  # noqa
        # 遍历选定的扫描列表，取消正在运行的扫描
        for path in selection:
            entry = model.get_value(model.get_iter(path), 0)
            if entry.running:
                self.cancel_scan(entry.command)
    # 移除扫描回调函数，接受一个小部件作为参数
    def _remove_scan_cb(self, widget):
        # 获取扫描结果列表中的模型和选择
        model, selection = self.scan_result.scan_result_notebook.scans_list.scans_list.get_selection().get_selected_rows()  # noqa
        selected_refs = []
        # 遍历选择的路径
        for path in selection:
            # 获取模型中指定路径的条目
            entry = model.get_value(model.get_iter(path), 0)
            # 如果条目正在运行，则取消扫描并从库存中移除已完成的扫描
            if entry.running:
                self.cancel_scan(entry.command)
            try:
                # 如果存在，则从库存中移除扫描
                self.inventory.remove_scan(entry.parsed)
            except ValueError:
                pass
            # 创建 TreeRowReferences，因为这些在我们改变模型时会持续存在
            selected_refs.append(Gtk.TreeRowReference.new(model, path))
        # 从 ScansListStore 中删除条目
        for ref in selected_refs:
            model.remove(model.get_iter(ref.get_path()))
        # 更新用户界面
        self.update_ui()

    # 收集 UMIT 信息，接受命令和解析后的参数作为参数
    def collect_umit_info(self, command, parsed):
        parsed.profile_name = command.profile
        parsed.nmap_command = command.command

    # 终止所有运行中的扫描
    def kill_all_scans(self):
        """Kill all running scans."""
        for scan in self.jobs:
            try:
                scan.kill()
            except AttributeError:
                pass
        del self.jobs[:]

    # 取消运行中的扫描
    def cancel_scan(self, command):
        """Cancel a running scan."""
        # 取消运行中的扫描
        self.scans_store.cancel_running_scan(command)
        command.kill()
        self.jobs.remove(command)
        self.update_cancel_button()
    # 验证执行的回调函数，定期调用以刷新输出，检查是否有任何运行的扫描已经完成
    # 在 execute_command 中启动调度回调的定时器。当没有更多运行的扫描时，此函数返回 True，以便不再调度
    def verify_execution(self):
        # 刷新 nmap 输出
        self.scan_result.refresh_nmap_output()

        # 存储已完成的扫描任务
        finished_jobs = []
        # 遍历所有的扫描任务
        for scan in self.jobs:
            try:
                # 检查扫描的状态
                alive = scan.scan_state()
                # 如果扫描仍在进行，则继续下一个扫描
                if alive:
                    continue
            except Exception as e:
                # 如果扫描意外终止，则记录日志并标记为失败
                log.debug("Scan terminated unexpectedly: %s (%s)" % (scan.command, e))
                self.scans_store.fail_running_scan(scan)
            else:
                # 如果扫描完成，则记录日志，加载扫描结果，并关闭扫描
                log.debug("Scan finished: %s" % scan.command)
                self.load_from_command(scan)
                scan.close()
            # 更新取消按钮状态
            self.update_cancel_button()
            # 将已完成的扫描任务添加到已完成列表中
            finished_jobs.append(scan)

        # 从任务列表中移除已完成的任务
        for finished in finished_jobs:
            self.jobs.remove(finished)
        # 释放已完成任务列表的内存
        del(finished_jobs)

        # 返回是否还有未完成的任务
        return len(self.jobs) != 0

    # 从文件中加载扫描结果
    def load_from_file(self, filename):
        # 创建 NmapParser 实例
        parsed = NmapParser()
        # 解析指定文件
        parsed.parse(filename)
        # 将解析结果标记为已保存
        parsed.unsaved = False

        # 更新目标配置文件
        self.update_target_profile(parsed)
        # 将扫描结果添加到库存中
        self.inventory.add_scan(parsed, filename=filename)
        # 更新用户界面
        self.update_ui()
        # 将扫描结果添加到扫描存储中
        i = self.scans_store.add_scan(parsed)
        # 记录日志
        log.info("scans_store.add_scan")
        # 设置激活的迭代器为扫描结果的笔记本
        self.scan_result.scan_result_notebook.nmap_output.set_active_iter(i)
        # 切换到端口和主机标签页
        self.scan_result.change_to_ports_hosts_tab()
    # 从解析后的 NmapParser 对象中加载扫描结果
    def load_from_parsed_result(self, parsed_result):
        """Load scan results from a parsed NmapParser object."""
        # 将 parsed_result 赋值给 parsed 变量
        parsed = parsed_result
        # 将 unsaved 属性设置为 False
        parsed.unsaved = False
    
        # 更新目标配置信息
        self.update_target_profile(parsed)
        # 将扫描结果添加到清单中
        self.inventory.add_scan(parsed)
        # 更新用户界面
        self.update_ui()
        # 将扫描结果添加到扫描存储中
        i = self.scans_store.add_scan(parsed)
        # 设置激活的迭代器
        self.scan_result.scan_result_notebook.nmap_output.set_active_iter(i)
        # 切换到端口和主机标签页
        self.scan_result.change_to_ports_hosts_tab()
    
    # 根据解析后的扫描结果更新目标配置信息
    def update_target_profile(self, parsed):
        """Update the "Target" and "Profile" entries based on the contents of a
        parsed scan."""
        # 获取目标列表
        targets = parsed.get_targets()
        # 获取配置文件名称
        profile_name = parsed.get_profile_name()
    
        # 设置命令行参数
        self.set_command_quiet(parsed.get_nmap_command() or "")
        # 设置目标列表
        self.set_target_quiet(join_quoted(targets))
        # 设置配置文件名称
        self.set_profile_name_quiet(profile_name or "")
    # 更新界面的主机和端口列表，从解析的扫描结果中获取
    def update_ui(self):
        """Update the interface's lists of hosts and ports from a parsed
        scan."""
        # 标记界面不为空
        self.empty = False

        # 获取所有在线的主机
        up_hosts = self.inventory.get_hosts_up()

        # 更新主机视图
        self.scan_result.scan_host_view.mass_update(up_hosts)

        # 更新拓扑图
        self.scan_result.scan_result_notebook.topology.update_radialnet()

        # 初始化主机和服务的字典
        self.hosts = {}
        self.services = {}
        for host in up_hosts:
            hostname = host.get_hostname()

            # 遍历主机的服务
            for service in host.services:
                name = service["service_name"]

                # 如果服务名不在字典中，则添加
                if name not in self.services.keys():
                    self.services[name] = []

                # 构建主机和服务的信息
                hs = {"host": host, "hostname": hostname}
                hs.update(service)

                # 将服务信息添加到对应的服务列表中
                self.services[name].append(hs)

            # 将主机信息添加到主机字典中
            self.hosts[hostname] = host

        # 如果主机和服务选择为空或变为空，则选择第一个主机（如果有的话）
        if (len(self.service_view_selection.get_selected_rows()[1]) == 0 and
                len(self.host_view_selection.get_selected_rows()[1]) == 0 and
                len(self.scan_result.scan_host_view.host_list) > 0):
            self.host_view_selection.select_iter(
                self.scan_result.scan_host_view.host_list.get_iter_first())

        # 设置过滤栏的信息文本
        self.filter_bar.set_information_text(_("%d/%d hosts shown") %
            (len(self.inventory.get_hosts_up()),
             len(NetworkInventory.get_hosts_up(self.inventory))))

        # 根据主机视图的模式刷新端口或主机输出
        mode = self.scan_result.scan_host_view.mode
        if mode == ScanHostsView.HOST_MODE:
            self.refresh_port_output()
        elif mode == ScanHostsView.SERVICE_MODE:
            self.refresh_host_output()
    # 刷新“Ports / Hosts”选项卡的“Ports”输出，以反映当前的主机选择
    def refresh_port_output(self):
        """Refresh the "Ports" output of the "Ports / Hosts" tab to reflect the
        current host selection."""
        # 调用scan_result_notebook对象的port_mode方法
        self.scan_result.scan_result_notebook.port_mode()

        # 获取选中的主机列表和选择的行
        model_host_list, selection = \
                self.host_view_selection.get_selected_rows()
        host_objs = []
        # 遍历选择的行
        for i in selection:
            # 获取主机名
            hostname = model_host_list[i[0]][2]
            # 如果主机名在self.hosts中
            if hostname in self.hosts:
                # 将主机对象添加到host_objs列表中
                host_objs.append(self.hosts[hostname])

        # 如果host_objs列表长度为1
        if len(host_objs) == 1:
            # 调用set_single_host_port方法
            self.set_single_host_port(host_objs[0])
        else:
            # 调用set_multiple_host_port方法
            self.set_multiple_host_port(host_objs)
        # 调用switch_host_details方法，传入build_host_details方法返回的主机详情
        self.switch_host_details(self.build_host_details(host_objs))

    # 刷新“Ports / Hosts”选项卡的“Hosts”输出，以反映当前的服务选择
    def refresh_host_output(self):
        """Refresh the "Hosts" output of the "Ports / Hosts" tab to reflect the
        current service selection."""
        # 调用scan_result_notebook对象的host_mode方法
        self.scan_result.scan_result_notebook.host_mode()

        # 获取选中的服务列表和选择的行
        model_service_list, selection = \
                self.service_view_selection.get_selected_rows()
        serv_objs = []
        # 遍历选择的行
        for i in selection:
            # 获取键值
            key = model_service_list[i[0]][0]
            # 如果键值在self.services中
            if key in self.services:
                # 将服务对象添加到serv_objs列表中
                serv_objs.append(self.services[key])

        # 每个serv_objs元素都是一个端口字典的列表
        if len(serv_objs) == 1:
            # 调用set_single_service_host方法
            self.set_single_service_host(serv_objs[0])
        else:
            servs = []
            # 遍历serv_objs列表
            for s in serv_objs:
                # 将服务名和端口列表组成字典，添加到servs列表中
                servs.append({
                    "service_name": s[0]["service_name"],
                    "ports": s})
            # 调用set_multiple_service_host方法
            self.set_multiple_service_host(servs)

    # 主机选择发生变化时的处理方法
    def host_selection_changed(self, widget):
        # 调用refresh_port_output方法
        self.refresh_port_output()
        # 切换nmap输出，显示第一个主机的出现
        model, selection = self.host_view_selection.get_selected_rows()
        # 遍历选择的行
        for path in selection:
            # 调用go_to_host方法，传入主机名
            self.go_to_host(model[path][2])
            break
    # 当服务选择改变时的回调函数
    def service_selection_changed(self, widget):
        # 刷新主机输出
        self.refresh_host_output()
        # 将扫描选项卡更改为“端口/主机”
        self.scan_result.change_to_ports_hosts_tab()

    # 当服务主机选择改变时的回调函数
    def service_host_selection_changed(self, selection):
        """当视图处于“服务”模式并且用户在给定服务的众多主机中更改选择时调用此回调。"""
        model, selection = selection.get_selected_rows()
        host_objs = []
        for path in selection:
            host_objs.append(model.get_value(model.get_iter(path), 2))
        self.switch_host_details(self.build_host_details(host_objs))

    # 切换“主机详情”视图以显示给定列表中的ScanHostDetailsPages
    def switch_host_details(self, pages):
        """切换“主机详情”视图以显示给定列表中的ScanHostDetailsPages。"""
        vbox = self.scan_result.scan_result_notebook.host_details_vbox

        # 删除旧的子项
        for child in vbox.get_children():
            vbox.remove(child)

        for p in pages:
            p.set_expanded(False)
            vbox._pack_noexpand_nofill(p)
        if len(pages) == 1:
            pages[0].set_expanded(True)
        vbox.show_all()

    # 从评论文本输入框的内容设置主机的评论
    def _save_comment(self, widget, extra, host):
        """从评论文本输入框的内容设置主机的评论。"""
        buff = widget.get_buffer()
        comment = buff.get_text(
                buff.get_start_iter(), buff.get_end_iter())
        if host.comment == comment:
            # 没有改变，忽略
            return
        host.comment = comment
        for scan in self.inventory.get_scans():
            for h in scan.get_hosts():
                if (h.get_ip() == host.get_ip() and
                        h.get_ipv6() == host.get_ipv6()):
                    h.set_comment(host.comment)
                    scan.unsaved = True
                    break
    # 构建并返回与给定主机对应的“主机详情”页面列表
    def build_host_details(self, hosts):
        # 创建一个空列表用于存储页面
        pages = []
        # 遍历给定的主机列表
        for host in hosts:
            # 创建一个ScanHostDetailsPage对象
            page = ScanHostDetailsPage(host)
            # 将页面的comment_txt_vw信号连接到_save_comment方法，传入主机信息
            page.host_details.comment_txt_vw.connect(
                    "insert-at-cursor", self._save_comment, host)
            # 将页面的comment_txt_vw信号连接到_save_comment方法，传入主机信息
            page.host_details.comment_txt_vw.connect(
                    "focus-out-event", self._save_comment, host)
            # 将页面添加到页面列表中
            pages.append(page)
        # 返回页面列表
        return pages

    # 更改“端口/主机”选项卡，以显示单个给定主机的端口输出
    def set_single_host_port(self, host):
        # 获取主机页面对象
        host_page = self.scan_result.scan_result_notebook.open_ports.host
        # 切换端口列表显示方式为列表存储
        host_page.switch_port_to_list_store()

        # 冻结主机页面
        host_page.freeze()
        # 清空端口列表
        host_page.clear_port_list()
        # 遍历主机的端口列表，将端口添加到端口列表中
        for p in host.ports:
            host_page.add_to_port_list(p)
        # 解冻主机页面
        host_page.thaw()

    # 更改“端口/主机”选项卡，以显示与单个命名服务相关联的主机
    def set_single_service_host(self, service):
        # 获取主机页面对象
        host_page = self.scan_result.scan_result_notebook.open_ports.host
        # 切换主机列表显示方式为列表存储
        host_page.switch_host_to_list_store()

        # 冻结主机页面
        host_page.freeze()
        # 清空主机列表
        host_page.clear_host_list()
        # 遍历服务列表，将主机添加到主机列表中
        for p in service:
            host_page.add_to_host_list(p["host"], p)
        # 解冻主机页面
        host_page.thaw()

    # 更改“端口/主机”选项卡，以显示host_list中所有主机的端口输出
    def set_multiple_host_port(self, host_list):
        # 获取主机页面对象
        host_page = self.scan_result.scan_result_notebook.open_ports.host
        # 切换端口列表显示方式为树形存储
        host_page.switch_port_to_tree_store()

        # 冻结主机页面
        host_page.freeze()
        # 清空端口树
        host_page.clear_port_tree()
        # 遍历主机列表，将主机添加到端口树中
        for host in host_list:
            host_page.add_to_port_tree(host)
        # 解冻主机页面
        host_page.thaw()
    # 设置多个服务的主机
    def set_multiple_service_host(self, service_list):
        """Change the "Ports / Hosts" tab to show the hosts associated with
        each of the services in service_list. Each element of service_list must
        be a dict with the keys "service_name" and "ports". When multiple
        services are selected, the hosts for each are contained in an
        expander."""
        # 获取主机页面对象
        host_page = self.scan_result.scan_result_notebook.open_ports.host
        # 切换主机页面到树形存储
        host_page.switch_host_to_tree_store()

        # 冻结主机页面
        host_page.freeze()
        # 清空主机树
        host_page.clear_host_tree()
        # 遍历服务列表，将服务名和端口添加到主机树
        for service in service_list:
            host_page.add_to_host_tree(
                    service["service_name"], service["ports"])
        # 解冻主机页面
        host_page.thaw()
class ScanResult(Gtk.Paned):
    """This is the pane that has the "Host"/"Service" column (ScanHostsView) on
    the left and the "Nmap Output"/"Ports / Hosts"/etc. (ScanResultNotebook) on
    the right. It's the part of the interface below the toolbar."""
    # 定义一个名为ScanResult的类，继承自Gtk.Paned类，表示扫描结果的界面布局
    def __init__(self, inventory, scans_store, scan_interface=None):
        # 初始化方法，接受inventory、scans_store和scan_interface三个参数
        Gtk.Paned.__init__(self, orientation=Gtk.Orientation.HORIZONTAL)
        # 调用父类的初始化方法，设置布局为水平方向

        self.scan_host_view = ScanHostsView(scan_interface)
        # 创建ScanHostsView对象，表示扫描主机的视图
        self.scan_result_notebook = ScanResultNotebook(inventory, scans_store)
        # 创建ScanResultNotebook对象，表示扫描结果的笔记本视图
        self.filter_toggle_button = Gtk.ToggleButton.new_with_label(_("Filter Hosts"))
        # 创建一个带有标签"Filter Hosts"的ToggleButton对象

        vbox = Gtk.Box.new(Gtk.Orientation.VERTICAL, 0)
        # 创建一个垂直方向的Box对象
        vbox.pack_start(self.scan_host_view, True, True, 0)
        # 将scan_host_view添加到vbox中，设置为可扩展和填充
        vbox.pack_start(self.filter_toggle_button, False, True, 0)
        # 将filter_toggle_button添加到vbox中，设置为不可扩展和填充
        self.pack1(vbox)
        # 将vbox添加到布局的左侧
        self.pack2(self.scan_result_notebook, True, False)
        # 将scan_result_notebook添加到布局的右侧，设置为可扩展但不填充

    def set_nmap_output(self, msg):
        # 设置Nmap输出的方法，接受一个消息参数
        self.scan_result_notebook.nmap_output.nmap_output.text_view.get_buffer().set_text(msg)  # noqa
        # 获取Nmap输出的文本视图缓冲区，并设置文本为msg

    def clear_nmap_output(self):
        # 清空Nmap输出的方法
        self.scan_result_notebook.nmap_output.nmap_output.text_view.get_buffer().set_text("")  # noqa
        # 获取Nmap输出的文本视图缓冲区，并清空文本

    def get_host_selection(self):
        # 获取主机选择的方法
        return self.scan_host_view.host_view.get_selection()
        # 返回扫描主机视图中主机选择的结果

    def get_service_selection(self):
        # 获取服务选择的方法
        return self.scan_host_view.service_view.get_selection()
        # 返回扫描主机视图中服务选择的结果

    def get_nmap_output(self):
        # 获取Nmap输出的方法
        return self.scan_result_notebook.nmap_output.get_nmap_output()
        # 返回Nmap输出的结果

    def clear_port_list(self):
        # 清空端口列表的方法
        self.scan_result_notebook.open_ports.host.clear_port_list()
        # 清空扫描结果笔记本中打开端口的主机的端口列表

    def change_to_ports_hosts_tab(self):
        # 切换到端口/主机标签页的方法
        self.scan_result_notebook.set_current_page(1)
        # 设置扫描结果笔记本的当前页为端口/主机标签页

    def change_to_nmap_output_tab(self):
        # 切换到Nmap输出标签页的方法
        self.scan_result_notebook.set_current_page(0)
        # 设置扫描结果笔记本的当前页为Nmap输出标签页

    def refresh_nmap_output(self):
        """Refresh the Nmap output with the newest output of command_execution,
        if it is not None."""
        # 刷新Nmap输出的方法，如果command_execution的最新输出不为空，则刷新
        self.scan_result_notebook.nmap_output.nmap_output.refresh_output()
        # 刷新Nmap输出的文本视图
# 定义一个名为ScanResultNotebook的类，继承自HIGNotebook
class ScanResultNotebook(HIGNotebook):
    """This is the right side of a ScanResult, the notebook with the tabs such
    as "Nmap Output"."""
    # 初始化方法，接受inventory和scans_store两个参数
    def __init__(self, inventory, scans_store):
        # 调用父类的初始化方法
        HIGNotebook.__init__(self)
        # 设置边框宽度为5
        self.set_border_width(5)

        # 调用私有方法__create_widgets，传入inventory和scans_store参数
        self.__create_widgets(inventory, scans_store)

        # 连接scans_list的row-activated信号到_scan_row_activated方法
        self.scans_list.scans_list.connect(
                "row-activated", self._scan_row_activated)

        # 在页面末尾添加nmap_output_page，并设置标签为'Nmap Output'
        self.append_page(self.nmap_output_page, Gtk.Label.new(_('Nmap Output')))
        # 在页面末尾添加open_ports_page，并设置标签为'Ports / Hosts'
        self.append_page(self.open_ports_page, Gtk.Label.new(_('Ports / Hosts')))
        # 在页面末尾添加topology_page，并设置标签为'Topology'
        self.append_page(self.topology_page, Gtk.Label.new(_('Topology')))
        # 在页面末尾添加host_details_page，并设置标签为'Host Details'
        self.append_page(self.host_details_page, Gtk.Label.new(_('Host Details')))
        # 在页面末尾添加scans_list_page，并设置标签为'Scans'
        self.append_page(self.scans_list_page, Gtk.Label.new(_('Scans'))

    # 定义host_mode方法
    def host_mode(self):
        # 调用open_ports的host_mode方法
        self.open_ports.host.host_mode()

    # 定义port_mode方法
    def port_mode(self):
        # 调用open_ports的port_mode方法
        self.open_ports.host.port_mode()

    # 定义私有方法__create_widgets，接受inventory和scans_store两个参数
    def __create_widgets(self, inventory, scans_store):
        # 创建open_ports_page为HIGVBox
        self.open_ports_page = HIGVBox()
        # 创建nmap_output_page为HIGVBox
        self.nmap_output_page = HIGVBox()
        # 创建topology_page为HIGVBox
        self.topology_page = HIGVBox()
        # 创建host_details_page为HIGScrolledWindow
        self.host_details_page = HIGScrolledWindow()
        # 创建host_details_vbox为HIGVBox
        self.host_details_vbox = HIGVBox()
        # 创建scans_list_page为HIGVBox
        self.scans_list_page = HIGVBox()

        # 创建open_ports为ScanOpenPortsPage
        self.open_ports = ScanOpenPortsPage()
        # 创建nmap_output为ScanNmapOutputPage，传入scans_store参数
        self.nmap_output = ScanNmapOutputPage(scans_store)
        # 创建topology为TopologyPage，传入inventory参数
        self.topology = TopologyPage(inventory)
        # 创建scans_list为ScanScanListPage，传入scans_store参数
        self.scans_list = ScanScanListPage(scans_store)

        # 创建no_selected为标签，内容为'No host selected.'
        self.no_selected = Gtk.Label.new(_('No host selected.'))
        # 将host_details设置为no_selected
        self.host_details = self.no_selected

        # 在open_ports_page中添加open_ports
        self.open_ports_page.add(self.open_ports)
        # 在nmap_output_page中添加nmap_output
        self.nmap_output_page.add(self.nmap_output)
        # 在topology_page中添加topology
        self.topology_page.add(self.topology)
        # 在scans_list_page中添加scans_list
        self.scans_list_page.add(self.scans_list)

        # 在host_details_page中添加host_details_vbox
        self.host_details_page.add_with_viewport(self.host_details_vbox)
        # 将host_details_vbox设置为填充和扩展
        self.host_details_vbox._pack_expand_fill(self.host_details)
    # 当在扫描列表上激活（双击）一个扫描时，切换回Nmap输出视图
    def _scan_row_activated(self, treeview, path, view_column):
        # 设置Nmap输出视图的活动迭代器为指定路径上的迭代器
        self.nmap_output.set_active_iter(treeview.get_model().get_iter(path))
        # 设置当前页为第一个页面（索引为0）
        self.set_current_page(0)
```