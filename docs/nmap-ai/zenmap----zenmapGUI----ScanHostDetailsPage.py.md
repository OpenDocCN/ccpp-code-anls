# `nmap\zenmap\zenmapGUI\ScanHostDetailsPage.py`

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
# 从 zenmapGUI.higwidgets.higexpanders 模块中导入 HIGExpander 类
from zenmapGUI.higwidgets.higexpanders import HIGExpander
# 从 zenmapGUI.higwidgets.higboxes 模块中导入 HIGVBox, HIGHBox, hig_box_space_holder 类
from zenmapGUI.higwidgets.higboxes import HIGVBox, HIGHBox, hig_box_space_holder
# 从 zenmapGUI.higwidgets.higlabels 模块中导入 HIGEntryLabel 类
from zenmapGUI.higwidgets.higlabels import HIGEntryLabel
# 从 zenmapGUI.higwidgets.higtables 模块中导入 HIGTable 类
from zenmapGUI.higwidgets.higtables import HIGTable
# 从 zenmapGUI.Icons 模块中导入 get_os_logo, get_vulnerability_logo 函数
from zenmapGUI.Icons import get_os_logo, get_vulnerability_logo
# 导入 zenmapCore.I18N 模块
import zenmapCore.I18N  # lgtm[py/unused-import]

# 定义变量 na 并赋值为 'Not available'
na = _('Not available')

# 定义类 ScanHostDetailsPage，继承自 HIGExpander 类
class ScanHostDetailsPage(HIGExpander):
    # 初始化方法，接受参数 host
    def __init__(self, host):
        # 调用父类 HIGExpander 的初始化方法，传入参数 host.get_hostname()
        HIGExpander.__init__(self, host.get_hostname())
        # 创建 HostDetails 实例并赋值给 self.host_details
        self.host_details = HostDetails(host)
        # 调用 self.hbox 的 _pack_expand_fill 方法，传入参数 self.host_details
        self.hbox._pack_expand_fill(self.host_details)

# 定义类 HostDetails，继承自 HIGVBox 类
class HostDetails(HIGVBox):
    # 初始化函数，传入主机名参数
    def __init__(self, host):
        # 调用父类的初始化函数
        HIGVBox.__init__(self)
    
        # 创建窗口部件
        self.__create_widgets()
    
        # 设置操作系统图标
        self.set_os_image(get_os_logo(host))
    
        # 设置漏洞图标
        self.set_vulnerability_image(get_vulnerability_logo(host.get_open_ports()))
    
        # 设置主机状态信息
        self.set_host_status({'state': host.get_state(),
            'open': str(host.get_open_ports()),
            'filtered': str(host.get_filtered_ports()),
            'closed': str(host.get_closed_ports()),
            'scanned': str(host.get_scanned_ports()),
            'uptime': host.get_uptime()['seconds'],
            'lastboot': host.get_uptime()['lastboot']})
    
        # 初始化地址字典
        addresses = {}
        # 如果主机有 IPv4 地址，则添加到地址字典中
        if host.ip is not None:
            addresses['ipv4'] = host.ip['addr']
        # 如果主机有 IPv6 地址，则添加到地址字典中
        if host.ipv6 is not None:
            addresses['ipv6'] = host.ipv6['addr']
        # 如果主机有 MAC 地址，则添加到地址字典中
        if host.mac is not None:
            addresses['mac'] = host.mac['addr']
        # 设置地址信息
        self.set_addresses(addresses)
    
        # 设置主机名
        self.set_hostnames(host.get_hostnames())
    
        # 获取最佳匹配的操作系统信息
        os = host.get_best_osmatch()
        # 如果存在操作系统信息，则添加端口使用信息
        if os:
            os['portsused'] = host.get_ports_used()
    
        # 设置操作系统信息
        self.set_os(os)
        # 设置TCP序列
        self.set_tcpseq(host.get_tcpsequence())
        # 设置IP序列
        self.set_ipseq(host.get_ipidsequence())
        # 设置TCP时间戳序列
        self.set_tcptsseq(host.get_tcptssequence())
        # 设置主机备注信息
        self.set_comment(host.comment)
    
    # 创建表格水平盒子
    def create_table_hbox(self):
        # 创建表格
        table = HIGTable()
        # 创建水平盒子
        hbox = HIGHBox()
    
        # 将空白填充添加到水平盒子中
        hbox._pack_noexpand_nofill(hig_box_space_holder())
        # 将表格添加到水平盒子中
        hbox._pack_noexpand_nofill(table)
    
        # 返回表格和水平盒子
        return table, hbox
    
    # 设置操作系统图标
    def set_os_image(self, image):
        self.os_image.set_from_stock(image, Gtk.IconSize.DIALOG)
    
    # 设置漏洞图标
    def set_vulnerability_image(self, image):
        self.vulnerability_image.set_from_stock(image, Gtk.IconSize.DIALOG)
    # 设置地址信息，将地址信息展开
    def set_addresses(self, address):
        # 设置地址展开器使用标记语言
        self.address_expander.set_use_markup(True)
        # 创建表格和水平框
        table, hbox = self.create_table_hbox()
        # 展开地址展开器
        self.address_expander.set_expanded(True)

        # 如果存在 IPv4 地址并且不等于1
        if ('ipv4' in address and
                address['ipv4'] != 1):
            # 设置 IPv4 地址信息
            self.info_ipv4_label.set_text(address['ipv4'])

        # 如果存在 IPv6 地址并且不等于1
        if ('ipv6' in address and
                address['ipv6'] != 1):
            # 设置 IPv6 地址信息
            self.info_ipv6_label.set_text(address['ipv6'])

        # 如果存在 MAC 地址并且不等于1
        if ('mac' in address and
                address['mac'] != 1):
            # 设置 MAC 地址信息
            self.info_mac_label.set_text(address['mac'])

        # 将 IPv4 标签和信息添加到表格中
        table.attach(self.ipv4_label, 0, 1, 0, 1)
        table.attach(self.info_ipv4_label, 1, 2, 0, 1)

        # 将 IPv6 标签和信息添加到表格中
        table.attach(self.ipv6_label, 0, 1, 1, 2)
        table.attach(self.info_ipv6_label, 1, 2, 1, 2)

        # 将 MAC 标签和信息添加到表格中
        table.attach(self.mac_label, 0, 1, 2, 3)
        table.attach(self.info_mac_label, 1, 2, 2, 3)

        # 将水平框添加到地址展开器中
        self.address_expander.add(hbox)
        # 将地址展开器添加到界面中
        self._pack_noexpand_nofill(self.address_expander)

    # 设置主机名信息
    def set_hostnames(self, hostname):
        # 如果存在主机名
        if hostname:
            # 设置主机名展开器使用标记语言
            self.hostnames_expander.set_use_markup(True)
            # 展开主机名展开器
            self.hostnames_expander.set_expanded(True)
            # 创建表格和水平框
            table, hbox = self.create_table_hbox()

            # 初始化行数
            y1 = 1
            y2 = 2

            # 遍历主机名列表
            for h in hostname:
                # 获取主机名和类型
                name = h.get('hostname', na)
                type = h.get('hostname_type', na)

                # 将主机名和类型添加到表格中
                table.attach(HIGEntryLabel(_('Name - Type:')), 0, 1, y1, y2)
                table.attach(HIGEntryLabel(name + ' - ' + type), 1, 2, y1, y2)
                y1 += 1
                y2 += 1

            # 将水平框添加到主机名展开器中
            self.hostnames_expander.add(hbox)
            # 将主机名展开器添加到界面中
            self._pack_noexpand_nofill(self.hostnames_expander)
    # 设置操作系统信息，包括名称、准确度和端口使用情况
    def set_os(self, os):
        # 如果有操作系统信息
        if os:
            # 设置操作系统展开框使用标记为真
            self.os_expander.set_use_markup(True)
            # 设置操作系统展开框展开
            self.os_expander.set_expanded(True)
            # 创建表格和水平框
            table, hbox = self.create_table_hbox()
            # 创建进度条
            progress = Gtk.ProgressBar()

            # 如果操作系统信息中包含准确度
            if 'accuracy' in os:
                # 设置进度条的百分比
                progress.set_fraction(float(os['accuracy']) / 100.0)
                # 设置进度条的文本为准确度百分比
                progress.set_text(os['accuracy'] + '%')
            else:
                # 如果没有准确度信息，设置进度条文本为“Not Available”
                progress.set_text(_('Not Available'))

            # 将名称标签和操作系统名称添加到表格中
            table.attach(HIGEntryLabel(_('Name:')), 0, 1, 0, 1)
            table.attach(HIGEntryLabel(os['name']), 1, 2, 0, 1)

            # 将准确度标签和进度条添加到表格中
            table.attach(HIGEntryLabel(_('Accuracy:')), 0, 1, 1, 2)
            table.attach(progress, 1, 2, 1, 2)

            # 设置y1和y2的初始值
            y1 = 2
            y2 = 3

            # 如果操作系统信息中包含端口使用情况
            if 'portsused' in os:
                # 设置端口使用情况
                self.set_ports_used(os['portsused'])
                # 将端口使用情况展开框添加到表格中
                table.attach(self.portsused_expander, 0, 2, y1, y2)
                y1 += 1
                y2 += 1

            # 如果操作系统信息中包含操作系统类别
            if 'osclasses' in os:
                # 设置操作系统类别
                self.set_osclass(os['osclasses'])
                # 设置操作系统类别展开框使用标记为真
                self.osclass_expander.set_use_markup(True)
                # 将操作系统类别展开框添加到表格中
                table.attach(self.osclass_expander, 0, 2, y1, y2)

            # 将水平框添加到操作系统展开框中
            self.os_expander.add(hbox)
            # 将操作系统展开框添加到界面中
            self._pack_noexpand_nofill(self.os_expander)

    # 设置端口使用情况
    def set_ports_used(self, ports):
        # 设置端口使用情况展开框使用标记为真
        self.portsused_expander.set_use_markup(True)
        # 创建表格和水平框
        table, hbox = self.create_table_hbox()

        # 设置y1和y2的初始值
        y1 = 0
        y2 = 1

        # 遍历端口列表
        for p in ports:
            # 将端口-协议-状态标签和端口信息添加到表格中
            table.attach(HIGEntryLabel(
                _('Port-Protocol-State:')), 0, 1, y1, y2)
            table.attach(HIGEntryLabel(
                p['portid'] + ' - ' + p['proto'] + ' - ' + p['state']
                ), 1, 2, y1, y2)
            y1 += 1
            y2 += 1

        # 将水平框添加到端口使用情况展开框中
        self.portsused_expander.add(hbox)
    # 设置操作系统类型信息，如果存在操作系统类型信息
    def set_osclass(self, osclass):
        # 如果存在操作系统类型信息
        if osclass:
            # 设置操作系统类型扩展器使用标记
            self.osclass_expander.set_use_markup(True)
            # 创建表格和水平框
            table, hbox = self.create_table_hbox()

            # 添加操作系统类型信息的标签到表格中
            table.attach(HIGEntryLabel(_('Type')), 0, 1, 0, 1)
            table.attach(HIGEntryLabel(_('Vendor')), 1, 2, 0, 1)
            table.attach(HIGEntryLabel(_('OS Family')), 2, 3, 0, 1)
            table.attach(HIGEntryLabel(_('OS Generation')), 3, 4, 0, 1)
            table.attach(HIGEntryLabel(_('Accuracy')), 4, 5, 0, 1)

            # 初始化行数
            y1 = 1
            y2 = 2

            # 遍历操作系统类型信息列表
            for o in osclass:
                # 将操作系统类型信息添加到表格中
                table.attach(HIGEntryLabel(o['type']), 0, 1, y1, y2)
                table.attach(HIGEntryLabel(o['vendor']), 1, 2, y1, y2)
                table.attach(HIGEntryLabel(o['osfamily']), 2, 3, y1, y2)
                table.attach(HIGEntryLabel(o['osgen']), 3, 4, y1, y2)

                # 创建进度条并设置文本和进度
                progress = Gtk.ProgressBar()
                progress.set_text(o['accuracy'] + '%')
                progress.set_fraction(float(o['accuracy']) / 100.0)
                table.attach(progress, 4, 5, y1, y2)
                y1 += 1
                y2 += 1

            # 将操作系统类型扩展器添加到水平框中
            self.osclass_expander.add(hbox)

    # 设置TCP序列信息，如果存在TCP序列信息
    def set_tcpseq(self, tcpseq):
        # 如果存在TCP序列信息
        if tcpseq:
            # 设置TCP扩展器使用标记
            self.tcp_expander.set_use_markup(True)
            # 创建表格和水平框
            table, hbox = self.create_table_hbox()

            # 创建下拉框并添加值
            combo = Gtk.ComboBoxText()
            for v in tcpseq['values'].split(','):
                combo.append_text(v)

            # 添加TCP序列信息的标签和下拉框到表格中
            table.attach(HIGEntryLabel(_('Difficulty:')), 0, 1, 1, 2)
            table.attach(HIGEntryLabel(tcpseq['difficulty']), 1, 2, 1, 2)

            table.attach(HIGEntryLabel(_('Index:')), 0, 1, 2, 3)
            table.attach(HIGEntryLabel(tcpseq['index']), 1, 2, 2, 3)

            table.attach(HIGEntryLabel(_('Values:')), 0, 1, 3, 4)
            table.attach(combo, 1, 2, 3, 4)

            # 将TCP扩展器添加到水平框中
            self.tcp_expander.add(hbox)
            # 将TCP扩展器添加到布局中
            self._pack_noexpand_nofill(self.tcp_expander)
    # 设置 IP 序列
    def set_ipseq(self, ipseq):
        # 如果 IP 序列存在
        if ipseq:
            # 设置 IP 扩展器使用标记语言
            self.ip_expander.set_use_markup(True)
            # 创建表格和水平框
            table, hbox = self.create_table_hbox()

            # 创建下拉框
            combo = Gtk.ComboBoxText()

            # 将 IP 序列的值拆分并添加到下拉框中
            for i in ipseq['values'].split(','):
                combo.append_text(i)

            # 将标签和 IP 序列的类别添加到表格中
            table.attach(HIGEntryLabel(_('Class:')), 0, 1, 0, 1)
            table.attach(HIGEntryLabel(ipseq['class']), 1, 2, 0, 1)

            # 将标签和下拉框添加到表格中
            table.attach(HIGEntryLabel(_('Values:')), 0, 1, 1, 2)
            table.attach(combo, 1, 2, 1, 2)

            # 将水平框添加到 IP 扩展器中
            self.ip_expander.add(hbox)
            # 将 IP 扩展器添加到界面中
            self._pack_noexpand_nofill(self.ip_expander)

    # 设置 TCP/TS 序列
    def set_tcptsseq(self, tcptsseq):
        # 如果 TCP/TS 序列存在
        if tcptsseq:
            # 设置 TCP/TS 扩展器使用标记语言
            self.tcpts_expander.set_use_markup(True)
            # 创建表格和水平框
            table, hbox = self.create_table_hbox()

            # 创建下拉框
            combo = Gtk.ComboBoxText()

            # 将 TCP/TS 序列的值拆分并添加到下拉框中
            for i in tcptsseq['values'].split(','):
                combo.append_text(i)

            # 将标签和 TCP/TS 序列的类别添加到表格中
            table.attach(HIGEntryLabel(_('Class:')), 0, 1, 0, 1)
            table.attach(HIGEntryLabel(tcptsseq['class']), 1, 2, 0, 1)

            # 将标签和下拉框添加到表格中
            table.attach(HIGEntryLabel(_('Values:')), 0, 1, 1, 2)
            table.attach(combo, 1, 2, 1, 2)

            # 将水平框添加到 TCP/TS 扩展器中
            self.tcpts_expander.add(hbox)
            # 将 TCP/TS 扩展器添加到界面中
            self._pack_noexpand_nofill(self.tcpts_expander)
    # 设置注释内容，如果有注释则展开注释框
    def set_comment(self, comment=''):
        # 设置注释框使用标记语言
        self.comment_expander.set_use_markup(True)
        # 如果有注释，则展开注释框
        if comment:
            self.comment_expander.set_expanded(True)
    
        # 创建水平布局
        hbox = HIGHBox()
    
        # 创建可滚动的注释框
        self.comment_scrolled = Gtk.ScrolledWindow()
        self.comment_scrolled.set_border_width(5)
        self.comment_scrolled.set_policy(
                Gtk.PolicyType.AUTOMATIC, Gtk.PolicyType.AUTOMATIC)
    
        # 创建文本视图
        self.comment_txt_vw = Gtk.TextView()
        self.comment_txt_vw.set_wrap_mode(Gtk.WrapMode.WORD)
        self.comment_txt_vw.get_buffer().set_text(comment)
    
        # 将文本视图添加到可滚动的注释框中
        self.comment_scrolled.add(self.comment_txt_vw)
        # 将可滚动的注释框添加到水平布局中
        hbox._pack_expand_fill(self.comment_scrolled)
    
        # 将水平布局添加到注释展开器中
        self.comment_expander.add(hbox)
        # 将注释展开器添加到布局中
        self._pack_noexpand_nofill(self.comment_expander)
    
    # 获取注释内容
    def get_comment(self):
        # 获取文本视图的缓冲区
        buffer = self.comment_txt_vw.get_buffer()
        # 返回缓冲区中的文本内容
        return buffer.get_text(buffer.get_start_iter(), buffer.get_end_iter())
```