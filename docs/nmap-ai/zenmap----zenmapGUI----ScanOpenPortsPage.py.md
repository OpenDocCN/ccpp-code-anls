# `nmap\zenmap\zenmapGUI\ScanOpenPortsPage.py`

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
# 要求使用 Gtk 版本 3.0
gi.require_version("Gtk", "3.0")
# 从 gi.repository 中导入 Gtk 模块
from gi.repository import Gtk
# 从 zenmapGUI.higwidgets.higboxes 模块中导入 HIGVBox 类
from zenmapGUI.higwidgets.higboxes import HIGVBox
# 从 zenmapCore.UmitLogging 模块中导入 log 函数
from zenmapCore.UmitLogging import log
# 导入 zenmapCore.I18N 模块，但未使用
import zenmapCore.I18N

# 定义一个函数，根据端口信息返回相应的图标
def findout_service_icon(port_info):
    # 如果端口状态为 "open" 或 "open|filtered"，返回 Gtk.STOCK_YES 图标
    if port_info["port_state"] in ["open", "open|filtered"]:
        return Gtk.STOCK_YES
    # 否则返回 Gtk.STOCK_NO 图标
    else:
        return Gtk.STOCK_NO

# 定义一个函数，从字典 d 中获取人类可读的版本字符串
def get_version_string(d):
    """Get a human-readable version string from the dict d. The keys used in d
    are "service_product", "service_version", and "service_extrainfo" (all are
    optional). This produces a string like "OpenSSH 4.3p2 Debian 9etch2
    (protocol 2.0)"."""
    # 创建一个空列表用于存储结果
    result = []
    # 如果字典 d 中存在 "service_product" 键
    if d.get("service_product"):
        # 将 "service_product" 对应的值添加到结果列表中
        result.append(d["service_product"])
    # 如果字典 d 中包含键 "service_version"，则将其值添加到结果列表中
    if d.get("service_version"):
        result.append(d["service_version"])
    # 如果字典 d 中包含键 "service_extrainfo"，则将其值添加到结果列表中，并在值两侧添加括号
    if d.get("service_extrainfo"):
        result.append("(" + d["service_extrainfo"] + ")")
    # 将结果列表中的元素用空格连接成字符串，并返回
    return " ".join(result)
# 根据主机名获取地址列表，如果主机名为空则返回空列表
def get_addrs(host):
    if host is None:
        return []
    return host.get_addrs_for_sort()


# 比较两个主机的地址列表
def cmp_addrs(host_a, host_b):
    # 定义比较函数
    def cmp(a, b):
        return (a > b) - (a < b)
    # 返回两个主机地址列表的比较结果
    return cmp(get_addrs(host_a), get_addrs(host_b))


# 比较端口列表中的地址
def cmp_port_list_addr(model, iter_a, iter_b, *_):
    host_a = model.get_value(iter_a, 0)
    host_b = model.get_value(iter_b, 0)
    return cmp_addrs(host_a, host_b)


# 比较端口树中的地址
def cmp_port_tree_addr(model, iter_a, iter_b, *_):
    host_a = model.get_value(iter_a, 0)
    host_b = model.get_value(iter_b, 0)
    return cmp_addrs(host_a, host_b)


# 比较主机列表中的地址
def cmp_host_list_addr(model, iter_a, iter_b, *_):
    host_a = model.get_value(iter_a, 2)
    host_b = model.get_value(iter_b, 2)
    return cmp_addrs(host_a, host_b)


# 比较主机树中的地址
def cmp_host_tree_addr(model, iter_a, iter_b, *_):
    host_a = model.get_value(iter_a, 2)
    host_b = model.get_value(iter_b, 2)
    return cmp_addrs(host_a, host_b)


# 扫描开放端口页面的类
class ScanOpenPortsPage(Gtk.ScrolledWindow):
    def __init__(self):
        Gtk.ScrolledWindow.__init__(self)
        self.set_policy(Gtk.PolicyType.AUTOMATIC, Gtk.PolicyType.AUTOMATIC)

        self.__create_widgets()

        self.add_with_viewport(self.host)

    # 创建页面的小部件
    def __create_widgets(self):
        self.host = HostOpenPorts()


# 主机开放端口类
class HostOpenPorts(HIGVBox):
    def __init__(self):
        HIGVBox.__init__(self)

        self._create_widgets()
        self._set_port_list()
        self._set_host_list()
        self._pack_widgets()

    # 设置端口模式
    def port_mode(self):
        child = self.scroll_ports_hosts.get_child()
        if id(child) != id(self.port_view):
            if child is not None:
                self.scroll_ports_hosts.remove(child)
            self.scroll_ports_hosts.add(self.port_view)
            self.port_view.show_all()
            self.host_view.hide()
    # 进入主机模式，显示主机视图，隐藏端口视图
    def host_mode(self):
        # 获取当前子部件
        child = self.scroll_ports_hosts.get_child()
        # 如果当前子部件不是主机视图
        if id(child) != id(self.host_view):
            # 如果当前子部件不为空，则移除
            if child is not None:
                self.scroll_ports_hosts.remove(child)
            # 添加主机视图到滚动窗口
            self.scroll_ports_hosts.add(self.host_view)
            # 显示主机视图
            self.host_view.show_all()
            # 隐藏端口视图
            self.port_view.hide()

    # 冻结通知和排序，以加快向模型中添加大量元素的速度
    def freeze(self):
        """Freeze notifications and sorting to make adding lots of elements to
        the model faster."""
        # 保存当前主机列表的排序列 ID
        self.frozen_host_list_sort_column_id = \
                self.host_list.get_sort_column_id()
        # 保存当前主机树的排序列 ID
        self.frozen_host_tree_sort_column_id = \
                self.host_tree.get_sort_column_id()
        # 保存当前端口列表的排序列 ID
        self.frozen_port_list_sort_column_id = \
                self.port_list.get_sort_column_id()
        # 保存当前端口树的排序列 ID
        self.frozen_port_tree_sort_column_id = \
                self.port_tree.get_sort_column_id()
        # 设置主机列表的默认排序函数
        self.host_list.set_default_sort_func(lambda *args: -1)
        # 设置主机树的默认排序函数
        self.host_tree.set_default_sort_func(lambda *args: -1)
        # 设置端口列表的默认排序函数
        self.port_list.set_default_sort_func(lambda *args: -1)
        # 设置端口树的默认排序函数
        self.port_tree.set_default_sort_func(lambda *args: -1)
        # 保存当前主机视图的模型
        self.frozen_host_view_model = self.host_view.get_model()
        # 保存当前端口视图的模型
        self.frozen_port_view_model = self.port_view.get_model()
        # 冻结主机视图的子部件通知
        self.host_view.freeze_child_notify()
        # 冻结端口视图的子部件通知
        self.port_view.freeze_child_notify()
        # 设置主机视图的模型为 None
        self.host_view.set_model(None)
        # 设置端口视图的模型为 None
        self.port_view.set_model(None)
    # 恢复通知和排序（在对模型进行更改后）
    def thaw(self):
        # 如果主机列表的排序列不为空
        if self.frozen_host_list_sort_column_id != (None, None):
            # 设置主机列表的排序列
            self.host_list.set_sort_column_id(
                    *self.frozen_host_list_sort_column_id)
        # 如果主机树的排序列不为空
        if self.frozen_host_tree_sort_column_id != (None, None):
            # 设置主机树的排序列
            self.host_tree.set_sort_column_id(
                    *self.frozen_host_tree_sort_column_id)
        # 如果端口列表的排序列不为空
        if self.frozen_port_list_sort_column_id != (None, None):
            # 设置端口列表的排序列
            self.port_list.set_sort_column_id(
                    *self.frozen_port_list_sort_column_id)
        # 如果端口树的排序列不为空
        if self.frozen_port_tree_sort_column_id != (None, None):
            # 设置端口树的排序列
            self.port_tree.set_sort_column_id(
                    *self.frozen_port_tree_sort_column_id)
        # 设置主机视图的模型为冻结的主机视图模型
        self.host_view.set_model(self.frozen_host_view_model)
        # 设置端口视图的模型为冻结的端口视图模型
        self.port_view.set_model(self.frozen_port_view_model)
        # 恢复主机视图的子通知
        self.host_view.thaw_child_notify()
        # 恢复端口视图的子通知
        self.port_view.thaw_child_notify()

    # 向端口列表中添加条目
    def add_to_port_list(self, p):
        # 创建端口条目
        entry = [None, "", findout_service_icon(p), int(p.get('portid', '0')),
            p.get('protocol', ''), p.get('port_state', ''),
            p.get('service_name', ''), get_version_string(p)]
        # 记录调试信息
        log.debug(">>> Add Port: %s" % entry)
        # 将条目添加到端口列表
        self.port_list.append(entry)

    # 向主机列表中添加条目
    def add_to_host_list(self, host, p):
        # 创建主机条目
        entry = ["", findout_service_icon(p), host, host.get_hostname(),
            int(p.get('portid', '0')), p.get('protocol', ''),
            p.get('port_state', ''), get_version_string(p)]
        # 记录调试信息
        log.debug(">>> Add Host: %s" % entry)
        # 将条目添加到主机列表
        self.host_list.append(entry)
    # 将主机信息添加到端口树中
    def add_to_port_tree(self, host):
        # 在端口树中添加父节点，包含主机名和其他信息
        parent = self.port_tree.append(
                None, [host, host.get_hostname(), None, 0, '', '', '', ''])
        # 遍历主机的端口信息，将每个端口添加到端口树中
        for p in host.get_ports():
            self.port_tree.append(parent,
                [None, '', findout_service_icon(p), int(p.get('portid', "0")),
                p.get('protocol', ''), p.get('port_state', ""),
                p.get('service_name', _("Unknown")), get_version_string(p)])

    # 将服务名和端口信息添加到主机树中
    def add_to_host_tree(self, service_name, ports):
        # 在主机树中添加父节点，包含服务名和其他信息
        parent = self.host_tree.append(
                None, [service_name, '', None, '', 0, '', '', ''])
        # 遍历端口信息，将每个端口添加到主机树中
        for p in ports:
            self.host_tree.append(parent,
                    [
                        '',
                        findout_service_icon(p),
                        p["host"],
                        p["host"].get_hostname(),
                        int(p.get('portid', "0")),
                        p.get('protocol', ""),
                        p.get('port_state', _("unknown")),
                        get_version_string(p)
                    ]
                )

    # 切换端口视图为列表模式
    def switch_port_to_list_store(self):
        # 如果端口视图的模型不是端口列表模型，则切换为端口列表模型
        if self.port_view.get_model() != self.port_list:
            self.port_view.set_model(self.port_list)
            self.port_columns['hostname'].set_visible(False)

    # 切换端口视图为树形模式
    def switch_port_to_tree_store(self):
        # 如果端口视图的模型不是端口树模型，则切换为端口树模型
        if self.port_view.get_model() != self.port_tree:
            self.port_view.set_model(self.port_tree)
            self.port_columns['hostname'].set_visible(True)

    # 切换主机视图为列表模式
    def switch_host_to_list_store(self):
        # 如果主机视图的模型不是主机列表模型，则切换为主机列表模型
        if self.host_view.get_model() != self.host_list:
            self.host_view.set_model(self.host_list)
            self.host_columns['service'].set_visible(False)

    # 切换主机视图为树形模式
    def switch_host_to_tree_store(self):
        # 如果主机视图的模型不是主机树模型，则切换为主机树模型
        if self.host_view.get_model() != self.host_tree:
            self.host_view.set_model(self.host_tree)
            self.host_columns['service'].set_visible(True)
    # 将端口视图添加到滚动窗口中
    def _pack_widgets(self):
        self.scroll_ports_hosts.add(self.port_view)
        self._pack_expand_fill(self.scroll_ports_hosts)
    
    # 清空端口列表
    def clear_port_list(self):
        for i in range(len(self.port_list)):
            iter = self.port_list.get_iter_first()
            del(self.port_list[iter])
    
    # 清空主机列表
    def clear_host_list(self):
        for i in range(len(self.host_list)):
            iter = self.host_list.get_iter_first()
            del(self.host_list[iter])
    
    # 清空端口树
    def clear_port_tree(self):
        for i in range(len(self.port_tree)):
            iter = self.port_tree.get_iter_first()
            del(self.port_tree[iter])
    
    # 清空主机树
    def clear_host_tree(self):
        for i in range(len(self.host_tree)):
            iter = self.host_tree.get_iter_first()
            del(self.host_tree[iter])
# 如果当前模块被直接执行，则执行以下代码
if __name__ == "__main__":
    # 创建一个 GTK 窗口对象
    w = Gtk.Window()
    # 创建一个 HostOpenPorts 对象
    h = HostOpenPorts()
    # 将 HostOpenPorts 对象添加到窗口中
    w.add(h)

    # 连接窗口的 "delete-event" 信号，当窗口关闭时退出 GTK 主循环
    w.connect("delete-event", lambda x, y: Gtk.main_quit())
    # 显示窗口及其所有子部件
    w.show_all()
    # 进入 GTK 主循环，等待事件处理
    Gtk.main()
```