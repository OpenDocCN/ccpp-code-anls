# `nmap\zenmap\zenmapCore\NmapParser.py`

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
# 导入所需的模块
import locale  # 用于处理特定地区的编码和格式设置
import time  # 用于处理时间相关的操作
import socket  # 用于网络通信
import copy  # 用于复制对象

from io import StringIO  # 从 io 模块中导入 StringIO 类

# 防止加载 PyXML
import xml  # 导入 xml 模块
xml.__path__ = [x for x in xml.__path__ if "_xmlplus" not in x]  # 如果路径中包含 "_xmlplus"，则移除该路径

from xml.sax import make_parser  # 导入 xml.sax 模块中的 make_parser 函数
from xml.sax import SAXException  # 导入 xml.sax 模块中的 SAXException 异常
from xml.sax.handler import ContentHandler, EntityResolver  # 导入 xml.sax.handler 模块中的 ContentHandler 和 EntityResolver
from xml.sax.saxutils import XMLGenerator  # 导入 xml.sax.saxutils 模块中的 XMLGenerator
from xml.sax.xmlreader import AttributesImpl as Attributes  # 导入 xml.sax.xmlreader 模块中的 AttributesImpl 类，并重命名为 Attributes

import zenmapCore.I18N  # 导入 zenmapCore.I18N 模块（未使用）
from zenmapCore.NmapOptions import NmapOptions, join_quoted  # 从 zenmapCore.NmapOptions 模块中导入 NmapOptions 和 join_quoted 函数
from zenmapCore.StringPool import unique  # 从 zenmapCore.StringPool 模块中导入 unique 函数

# Nmap DTD 文件的版本
XML_OUTPUT_VERSION = "1.04"

class HostInfo(object):
    # 这里是类的定义，包括类的属性和方法
    # 初始化函数，初始化各个属性为默认值
    def __init__(self):
        self.comment = None  # 注释
        self._tcpsequence = {}  # TCP 序列
        self._osmatches = []  # 操作系统匹配结果
        self._ports = []  # 端口列表
        self._ports_used = []  # 使用的端口列表
        self._extraports = []  # 额外端口列表
        self._uptime = {}  # 运行时间
        self._hostnames = []  # 主机名列表
        self._tcptssequence = {}  # TCP 时间戳序列
        self._ipidsequence = {}  # IP ID 序列
        self._ip = None  # IP 地址
        self._ipv6 = None  # IPv6 地址
        self._mac = None  # MAC 地址
        self._state = ''  # 状态
        self._comment = ''  # 注释
        self._trace = {}  # 跟踪信息字典
    
    # 创建克隆对象
    def make_clone(self):
        clone = HostInfo()
        clone.comment = self.comment  # 复制注释
        clone._tcpsequence = copy.deepcopy(self._tcpsequence)  # 复制 TCP 序列
        clone._osmatches = copy.deepcopy(self._osmatches)  # 复制操作系统匹配结果
        clone._ports = copy.deepcopy(self._ports)  # 复制端口列表
        clone._ports_used = self._ports_used  # 复制使用的端口列表
        clone._extraports = self._extraports  # 复制额外端口列表
        clone._uptime = copy.deepcopy(self._uptime)  # 复制运行时间
        clone._hostnames = copy.deepcopy(self._hostnames)  # 复制主机名列表
        clone._tcptssequence = copy.deepcopy(self._tcptssequence)  # 复制 TCP 时间戳序列
        clone._ipidsequence = copy.deepcopy(self._ipidsequence)  # 复制 IP ID 序列
        clone._ip = copy.deepcopy(self._ip)  # 复制 IP 地址
        clone._ipv6 = copy.deepcopy(self._ipv6)  # 复制 IPv6 地址
        clone._mac = copy.deepcopy(self._mac)  # 复制 MAC 地址
        clone._state = self._state  # 复制状态
        clone._comment = self._comment  # 复制注释
        clone._trace = copy.deepcopy(self._trace)  # 复制跟踪信息字典
        return clone  # 返回克隆对象
    
    # 设置 TCP 序列
    def set_tcpsequence(self, sequence):
        self._tcpsequence = sequence  # 设置 TCP 序列
    
    # 获取 TCP 序列
    def get_tcpsequence(self):
        if self._tcpsequence:  # 如果 TCP 序列存在
            return self._tcpsequence  # 返回 TCP 序列
        return {}  # 否则返回空字典
    
    # 设置 TCP 时间戳序列
    def set_tcptssequence(self, sequence):
        self._tcptssequence = sequence  # 设置 TCP 时间戳序列
    # 获取TCP连接的序列号
    def get_tcptssequence(self):
        # 如果已经存在TCP连接的序列号，则返回该序列号
        if self._tcptssequence:
            return self._tcptssequence
        # 否则返回空字典
        return {}

    # 设置IP数据包的序列号
    def set_ipidsequence(self, sequence):
        # 将传入的序列号赋值给_ipidsequence属性
        self._ipidsequence = sequence

    # 获取IP数据包的序列号
    def get_ipidsequence(self):
        # 如果已经存在IP数据包的序列号，则返回该序列号
        if self._ipidsequence:
            return self._ipidsequence
        # 否则返回空字典
        return {}

    # 设置操作系统匹配结果
    def set_osmatches(self, matches):
        # 将传入的匹配结果赋值给_osmatches属性
        self._osmatches = matches

    # 获取操作系统匹配结果
    def get_osmatches(self):
        return self._osmatches

    # 获取最佳的操作系统匹配结果
    def get_best_osmatch(self):
        """Return the OS match with the highest accuracy."""
        # 如果没有操作系统匹配结果，则返回None
        if not self._osmatches:
            return None

        # 定义一个函数用于排序操作系统匹配结果
        def osmatch_key(osmatch):
            try:
                return -float(osmatch["accuracy"])
            except ValueError:
                return 0

        # 返回排序后的操作系统匹配结果中的第一个元素
        return sorted(self._osmatches, key=osmatch_key)[0]

    # 设置已使用的端口
    def set_ports_used(self, ports):
        # 将传入的端口列表赋值给_ports_used属性
        self._ports_used = ports

    # 获取已使用的端口
    def get_ports_used(self):
        return self._ports_used

    # 设置系统运行时间
    def set_uptime(self, uptime):
        # 将传入的系统运行时间赋值给_uptime属性
        self._uptime = uptime

    # 获取系统运行时间
    def get_uptime(self):
        # 如果已经存在系统运行时间，则返回该系统运行时间
        if self._uptime:
            return self._uptime

        # 避免返回空字典
        return {"seconds": "", "lastboot": ""}
    # ports 是一个包含字典的数组，每个字典的形式如下
    # {'port_state': u'open', 'portid': u'22', 'protocol': u'tcp',
    #  'service_conf': u'10', 'service_extrainfo': u'protocol 2.0',
    #  'service_method': u'probed', 'service_name': u'ssh',
    #  'service_product': u'OpenSSH', 'service_version': u'4.3'}
    def set_ports(self, ports):
        # 设置对象的端口属性为给定的 ports
        self._ports = ports
    
    def get_ports(self):
        # 返回对象的端口属性
        return self._ports
    
    # extraports 是一个包含字典的数组，每个字典的形式如下
    # {'count': u'1709', 'state': u'filtered'}
    def set_extraports(self, port_list):
        # 设置对象的额外端口属性为给定的 port_list
        self._extraports = port_list
    
    def get_extraports(self):
        # 返回对象的额外端口属性
        return self._extraports
    
    # hostnames 是一个包含字典的列表，每个字典的形式如下
    # [{'hostname': u'scanme.nmap.org', 'hostname_type': u'PTR'}]
    def set_hostnames(self, hostname_list):
        # 设置对象的主机名属性为给定的 hostname_list
        self._hostnames = hostname_list
    
    def get_hostnames(self):
        # 返回对象的主机名属性
        return self._hostnames
    
    # ip, ipv6, 和 mac 要么是 None，要么是字典，形式如下
    # {'vendor': u'', 'type': u'ipv4', 'addr': u'64.13.134.52'}
    def set_ip(self, addr):
        # 设置对象的 IP 地址属性为给定的 addr
        self._ip = addr
    
    def get_ip(self):
        # 返回对象的 IP 地址属性
        return self._ip
    
    def set_mac(self, addr):
        # 设置对象的 MAC 地址属性为给定的 addr
        self._mac = addr
    
    def get_mac(self):
        # 返回对象的 MAC 地址属性
        return self._mac
    
    def set_ipv6(self, addr):
        # 设置对象的 IPv6 地址属性为给定的 addr
        self._ipv6 = addr
    
    def get_ipv6(self):
        # 返回对象的 IPv6 地址属性
        return self._ipv6
    # 返回一个按照以下规则排序的地址列表：
    # 1) IPv4 在 IPv6 之前，在 MAC 之前
    # 2) 地址按照它们的二进制值排序，而不是它们的字符串表示
    # 在对主机按照地址排序时，使用此函数作为比较键。

    # 创建一个空列表
    l = []
    # 如果存在 IPv4 地址，则将其添加到列表中，使用 inet_aton 将地址转换为二进制
    if self.ip:
        l.append((1, socket.inet_aton(self.ip["addr"])))
    # 如果存在 IPv6 地址
    if self.ipv6:
        try:
            # 尝试使用 inet_pton 将地址转换为二进制，并添加到列表中
            l.append((1, socket.inet_pton(socket.AF_INET6, self.ipv6["addr"]))
        except AttributeError:
            # Windows 没有 socket.inet_pton，因此按照字母顺序排序
            # 将地址编码为字节字符串，以便与二进制地址字符串进行比较
            l.append((1, self.ipv6["addr"].encode("utf-8")))
    # 如果存在 MAC 地址，则将其添加到列表中，将 MAC 地址转换为二进制
    if self.mac:
        l.append((3, "".join(
            chr(int(x, 16)) for x in self.mac["addr"].split(":"))))
    # 对列表进行排序
    l.sort()
    # 返回排序后的列表
    return l

    # 返回注释字符串
    def get_comment(self):
        return self._comment

    # 设置注释字符串
    def set_comment(self, comment):
        self._comment = comment

    # 设置状态字符串，如 u'up' 或 u'down'
    def set_state(self, status):
        self._state = status

    # 返回状态字符串
    def get_state(self):
        return self._state

    # 返回主机名
    def get_hostname(self):
        hostname = None
        # 如果主机名列表不为空，则获取第一个主机名
        if len(self._hostnames) > 0:
            hostname = self._hostnames[0]["hostname"]

        # 获取 IP、IPv6 或 MAC 地址
        address = self.ip or self.ipv6 or self.mac
        if address is not None:
            address = address["addr"]

        # 如果存在主机名和地址，则返回格式化后的字符串
        if hostname is not None:
            if address is not None:
                return "%s (%s)" % (hostname, address)
            else:
                return hostname
        else:
            # 如果不存在主机名，则返回地址
            if address is not None:
                return address
            else:
                # 如果主机名和地址都不存在，则返回"Unknown Host"
                return _("Unknown Host")
    # 根据给定的状态列表统计端口数量
    def get_port_count_by_states(self, states):
        # 初始化计数器
        count = 0

        # 遍历端口列表
        for p in self.ports:
            # 获取端口状态
            state = p.get('port_state')
            # 如果状态在给定的状态列表中，则计数加一
            if state in states:
                count += 1

        # 遍历额外端口列表
        for extra in self.get_extraports():
            # 如果额外端口的状态在给定的状态列表中，则将计数加上额外端口的数量
            if extra['state'] in states:
                count += int(extra['count'])

        # 返回计数结果
        return count

    # 获取开放状态的端口数量
    def get_open_ports(self):
        return self.get_port_count_by_states(('open', 'open|filtered'))

    # 获取过滤状态的端口数量
    def get_filtered_ports(self):
        return self.get_port_count_by_states(
                ('filtered', 'open|filtered', 'closed|filtered'))

    # 获取关闭状态的端口数量
    def get_closed_ports(self):
        return self.get_port_count_by_states(('closed', 'closed|filtered'))

    # 获取扫描过的端口数量
    def get_scanned_ports(self):
        scanned = 0

        # 遍历端口列表，计数加一
        for p in self.ports:
            scanned += 1

        # 遍历额外端口列表，将计数加上额外端口的数量
        for extra in self.get_extraports():
            scanned += int(extra["count"])

        # 返回扫描过的端口数量
        return scanned

    # 获取服务信息列表
    def get_services(self):
        services = []
        # 遍历端口列表，将每个端口的服务信息添加到服务列表中
        for p in self.ports:
            services.append({
                "service_name": p.get("service_name", _("unknown")),
                "portid": p.get("portid", ""),
                "service_version": p.get("service_version",
                    _("Unknown version")),
                "service_product": p.get("service_product", ""),
                "service_extrainfo": p.get("service_extrainfo", ""),
                "port_state": p.get("port_state", _("unknown")),
                "protocol": p.get("protocol", "")
                })
        # 返回服务信息列表
        return services

    # 获取跟踪信息
    def get_trace(self):
        return self._trace

    # 设置跟踪信息
    def set_trace(self, trace):
        self._trace = trace

    # 添加跟踪信息的跳数
    def append_trace_hop(self, hop):
        # 如果跟踪信息中存在"hops"键，则将跳数添加到"hops"列表中，否则创建"hops"列表并添加跳数
        if "hops" in self._trace:
            self._trace["hops"].append(hop)
        else:
            self._trace["hops"] = [hop]

    # 设置跟踪信息的错误信息
    def set_trace_error(self, errorstr):
        # 设置跟踪信息的错误信息
        self._trace["error"] = errorstr

    # 属性
    tcpsequence = property(get_tcpsequence, set_tcpsequence)
    # 定义属性 osmatches，用于获取和设置操作系统匹配信息
    osmatches = property(get_osmatches, set_osmatches)
    # 定义属性 ports，用于获取和设置端口信息
    ports = property(get_ports, set_ports)
    # 定义属性 ports_used，用于获取和设置已使用的端口信息
    ports_used = property(get_ports_used, set_ports_used)
    # 定义属性 extraports，用于获取和设置额外端口信息
    extraports = property(get_extraports, set_extraports)
    # 定义属性 uptime，用于获取和设置主机的正常运行时间
    uptime = property(get_uptime, set_uptime)
    # 定义属性 hostnames，用于获取和设置主机名信息
    hostnames = property(get_hostnames, set_hostnames)
    # 定义属性 tcptssequence，用于获取和设置 TCP 时间戳序列信息
    tcptssequence = property(get_tcptssequence, set_tcptssequence)
    # 定义属性 ipidsequence，用于获取和设置 IP ID 序列信息
    ipidsequence = property(get_ipidsequence, set_ipidsequence)
    # 定义属性 ip，用于获取和设置 IP 地址信息
    ip = property(get_ip, set_ip)
    # 定义属性 ipv6，用于获取和设置 IPv6 地址信息
    ipv6 = property(get_ipv6, set_ipv6)
    # 定义属性 mac，用于获取和设置 MAC 地址信息
    mac = property(get_mac, set_mac)
    # 定义属性 state，用于获取和设置主机状态信息
    state = property(get_state, set_state)
    # 定义属性 comment，用于获取和设置注释信息
    comment = property(get_comment, set_comment)
    # 定义属性 services，用于获取服务信息
    services = property(get_services)
    # 定义属性 trace，用于获取和设置跟踪信息
    trace = property(get_trace, set_trace)
class ParserBasics(object):
    def __init__(self):
        # 初始化 XML 输出文件是否为临时文件的标志，True 表示临时文件，False 表示用户指定的文件
        # 如果有任何一个是用户指定的文件，则在 set_nmap_command 中不会从命令字符串中删除它
        self.xml_is_temp = True

        # 初始化 nmap 字典，包括 nmaprun、scaninfo、verbose、debugging、hosts、runstats
        self.nmap = {
                'nmaprun': {},
                'scaninfo': [],
                'verbose': '',
                'debugging': '',
                'hosts': [],
                'runstats': {}
                }

        # 初始化 NmapOptions 对象
        self.ops = NmapOptions()
        # 初始化 _nmap_output 为 StringIO 对象
        self._nmap_output = StringIO()

    def set_xml_is_temp(self, xml_is_temp):
        # 如果用户指定了自己的 -oX 选项，则该标志为 False
        # 这意味着我们不应该从命令字符串中删除 -oX 选项
        # 值为 True 表示我们使用的是临时文件，应该从命令字符串中删除它（参见 set_nmap_command）
        self.xml_is_temp = xml_is_temp

    def get_profile_name(self):
        # 获取 nmaprun 中的 profile_name，如果不存在则返回空字符串
        return self.nmap['nmaprun'].get('profile_name', '')

    def set_profile_name(self, name):
        # 设置 nmaprun 中的 profile_name
        self.nmap['nmaprun']['profile_name'] = name

    def get_targets(self):
        # 获取目标列表
        return self.ops.target_specs

    def set_targets(self, targets):
        # 设置目标列表
        self.ops.target_specs = targets

    def get_nmap_output(self):
        # 获取 _nmap_output 中的值
        return self._nmap_output.getvalue()

    def set_nmap_output(self, nmap_output):
        # 关闭原有的 _nmap_output，创建新的 StringIO 对象，并写入 nmap_output
        self._nmap_output.close()
        del self._nmap_output
        self._nmap_output = StringIO()
        self._nmap_output.write(nmap_output)

    def del_nmap_output(self):
        # 关闭 _nmap_output 并删除它
        self._nmap_output.close()
        del self._nmap_output

    def get_debugging_level(self):
        # 获取 debugging 等级
        return self.nmap.get('debugging', '')

    def set_debugging_level(self, level):
        # 设置 debugging 等级
        self.nmap['debugging'] = level

    def get_verbose_level(self):
        # 获取 verbose 等级
        return self.nmap.get('verbose', '')
    # 设置详细级别，将级别值存储到 nmap 字典中的 verbose 键
    def set_verbose_level(self, level):
        self.nmap['verbose'] = level
    
    # 获取扫描信息，如果不存在则返回空字符串
    def get_scaninfo(self):
        return self.nmap.get('scaninfo', '')
    
    # 设置扫描信息，将信息存储到 nmap 字典中的 scaninfo 键
    def set_scaninfo(self, info):
        self.nmap['scaninfo'] = info
    
    # 获取已扫描的服务，如果未扫描则返回 None，否则返回已扫描的服务列表
    def get_services_scanned(self):
        if self._services_scanned is None:
            return self._services_scanned
    
        services = []
        for scan in self.nmap.get('scaninfo', []):
            services.append(scan['services'])
    
        self._services_scanned = ','.join(services)
        return self._services_scanned
    
    # 设置已扫描的服务，将服务列表存储到 _services_scanned 属性中
    def set_services_scanned(self, services_scanned):
        self._services_scanned = services_scanned
    
    # 获取 nmap 命令，通过 ops 对象渲染字符串并返回
    def get_nmap_command(self):
        return self.ops.render_string()
    
    # 设置 nmap 命令，解析命令字符串并存储到 nmap 字典中的 nmaprun 键
    def set_nmap_command(self, command):
        self.ops.parse_string(command)
        if self.xml_is_temp:
            self.ops["-oX"] = None
        self.nmap['nmaprun']['args'] = self.ops.render_string()
    
    # 获取扫描类型列表
    def get_scan_type(self):
        types = []
        for t in self.nmap.get('scaninfo', []):
            types.append(t['type'])
        return types
    
    # 获取协议列表
    def get_protocol(self):
        protocols = []
        for proto in self.nmap.get('scaninfo', []):
            protocols.append(proto['protocol'])
        return protocols
    
    # 获取服务数量，如果未获取则返回 None，否则返回服务数量
    def get_num_services(self):
        if self._num_services is None:
            return self._num_services
    
        num = 0
        for n in self.nmap.get('scaninfo', []):
            num += int(n['numservices'])
    
        self._num_services = num
        return self._num_services
    
    # 设置服务数量，将服务数量存储到 _num_services 属性中
    def set_num_services(self, num_services):
        self._num_services = num_services
    
    # 获取日期，将 nmaprun 中的 start 值转换为时间格式并返回
    def get_date(self):
        epoch = int(self.nmap['nmaprun'].get('start', '0'))
        return time.localtime(epoch)
    
    # 获取扫描开始时间，如果不存在则返回 '0'
    def get_start(self):
        return self.nmap['nmaprun'].get('start', '0')
    
    # 设置扫描开始时间，将开始时间存储到 nmap 字典中的 nmaprun 键
    def set_start(self, start):
        self.nmap['nmaprun']['start'] = start
    # 定义一个方法，用于设置日期
    def set_date(self, date):
        # 检查传入的日期类型是否为整数
        if type(date) == type(int):
            # 如果是整数类型，则将日期赋值给nmaprun字典中的start键
            self.nmap['nmaprun']['start'] = date
        else:
            # 如果不是整数类型，则抛出异常，提示日期格式错误
            raise Exception("Wrong date format. Date should be saved \
    # 获取已开放的端口数量
    def get_open_ports(self):
        ports = 0

        # 遍历主机列表，获取每个主机的已开放端口数量并累加
        for h in self.nmap.get('hosts', []):
            ports += h.get_open_ports()

        return ports

    # 获取被过滤的端口数量
    def get_filtered_ports(self):
        ports = 0

        # 遍历主机列表，获取每个主机的被过滤端口数量并累加
        for h in self.nmap.get('hosts', []):
            ports += h.get_filtered_ports()

        return ports

    # 获取已关闭的端口数量
    def get_closed_ports(self):
        ports = 0

        # 遍历主机列表，获取每个主机的已关闭端口数量并累加
        for h in self.nmap['hosts']:
            ports += h.get_closed_ports()

        return ports

    # 获取格式化的日期
    def get_formatted_date(self):
        return time.strftime("%B %d, %Y - %H:%M", self.get_date())

    # 获取扫描器信息
    def get_scanner(self):
        return self.nmap['nmaprun'].get('scanner', '')

    # 设置扫描器信息
    def set_scanner(self, scanner):
        self.nmap['nmaprun']['scanner'] = scanner

    # 获取扫描器版本信息
    def get_scanner_version(self):
        return self.nmap['nmaprun'].get('version', '')

    # 设置扫描器版本信息
    def set_scanner_version(self, version):
        self.nmap['nmaprun']['version'] = version

    # 获取 IPv4 地址列表
    def get_ipv4(self):
        hosts = self.nmap.get('hosts')
        if hosts is None:
            return []
        return [host.ip for host in hosts if host.ip is not None]

    # 获取 MAC 地址列表
    def get_mac(self):
        hosts = self.nmap.get('hosts')
        if hosts is None:
            return []
        return [host.mac for host in hosts if host.mac is not None]

    # 获取 IPv6 地址列表
    def get_ipv6(self):
        hosts = self.nmap.get('hosts')
        if hosts is None:
            return []
        return [host.ipv6 for host in hosts if host.ipv6 is not None]

    # 获取主机名列表
    def get_hostnames(self):
        hostnames = []
        for host in self.nmap.get('hosts', []):
            hostnames += host.get_hostnames()
        return hostnames

    # 获取主机信息
    def get_hosts(self):
        return self.nmap.get('hosts', None)

    # 获取扫描统计信息
    def get_runstats(self):
        return self.nmap.get('runstats', None)

    # 设置扫描统计信息
    def set_runstats(self, stats):
        self.nmap['runstats'] = stats
    # 获取主机数量，如果没有则返回 0
    def get_hosts_down(self):
        return int(self.nmap['runstats'].get('hosts_down', '0'))

    # 设置主机数量为 down
    def set_hosts_down(self, down):
        self.nmap['runstats']['hosts_down'] = int(down)

    # 获取主机数量，如果没有则返回 0
    def get_hosts_up(self):
        return int(self.nmap['runstats'].get('hosts_up', '0'))

    # 设置主机数量为 up
    def set_hosts_up(self, up):
        self.nmap['runstats']['hosts_up'] = int(up)

    # 获取已扫描主机数量，如果没有则返回 0
    def get_hosts_scanned(self):
        return int(self.nmap['runstats'].get('hosts_scanned', '0'))

    # 设置已扫描主机数量
    def set_hosts_scanned(self, scanned):
        self.nmap['runstats']['hosts_scanned'] = int(scanned)

    # 获取扫描完成时间
    def get_finish_time(self):
        return time.localtime(int(self.nmap['runstats'].get('finished_time', '0')))

    # 设置扫描完成时间
    def set_finish_time(self, finish):
        self.nmap['runstats']['finished_time'] = int(finish)

    # 获取扫描完成时间的时间戳
    def get_finish_epoc_time(self):
        return int(self.nmap['runstats'].get('finished_time', '0'))

    # 设置扫描完成时间的时间戳
    def set_finish_epoc_time(self, time):
        self.nmap['runstats']['finished_time'] = time

    # 获取扫描名称的人类可读字符串
    def get_scan_name(self):
        scan_name = self.nmap.get("scan_name")
        if scan_name:
            return scan_name
        if self.profile_name and self.get_targets():
            return _("%s on %s") % (self.profile_name, join_quoted(self.get_targets()))
        return self.get_nmap_command()

    # 设置扫描名称
    def set_scan_name(self, scan_name):
        self.nmap["scan_name"] = scan_name

    # 获取格式化的扫描完成日期
    def get_formatted_finish_date(self):
        return time.strftime("%B %d, %Y - %H:%M", self.get_finish_time())
    # 创建一个端口到协议的字典，包含所有扫描到的端口
    def get_port_protocol_dict(self):
        # 初始化一个空字典用于存储端口和协议的对应关系
        ports = {}
        # 遍历扫描信息列表
        for scaninfo in self.scaninfo:
            # 获取服务字符串并去除首尾空格
            services_string = scaninfo['services'].strip()
            # 如果服务字符串为空，则初始化为空数组
            if services_string == "":
                services_array = []
            else:
                # 否则，将服务字符串按逗号分割成数组
                services_array = services_string.split(',')
            # 遍历服务数组
            for item in services_array:
                # 如果服务项中不包含连字符
                if item.find('-') == -1:
                    # 如果端口不在字典中，则初始化为空数组
                    if int(item) not in ports:
                        ports[int(item)] = []
                    # 将协议添加到对应端口的协议数组中
                    ports[int(item)].append(scaninfo['protocol'])
                else:
                    # 否则，将连字符分割的起始和结束端口
                    begin, end = item.split('-')
                    # 遍历起始到结束端口范围
                    for port in range(int(begin), int(end) + 1):
                        # 如果端口不在字典中，则初始化为空数组
                        if int(port) not in ports:
                            ports[int(port)] = []
                        # 将协议添加到对应端口的协议数组中
                        ports[int(port)].append(scaninfo['protocol'])
        # 返回端口到协议的字典
        return ports

    # 以下是一系列属性的定义，用于获取和设置对象的各种属性
    profile_name = property(get_profile_name, set_profile_name)
    nmap_output = property(get_nmap_output, set_nmap_output, del_nmap_output)
    debugging_level = property(get_debugging_level, set_debugging_level)
    verbose_level = property(get_verbose_level, set_verbose_level)
    scaninfo = property(get_scaninfo, set_scaninfo)
    services_scanned = property(get_services_scanned, set_services_scanned)
    nmap_command = property(get_nmap_command, set_nmap_command)
    scan_type = property(get_scan_type)
    protocol = property(get_protocol)
    num_services = property(get_num_services, set_num_services)
    date = property(get_date, set_date)
    open_ports = property(get_open_ports)
    filtered_ports = property(get_filtered_ports)
    closed_ports = property(get_closed_ports)
    formatted_date = property(get_formatted_date)
    scanner = property(get_scanner, set_scanner)
    scanner_version = property(get_scanner_version, set_scanner_version)
    ipv4 = property(get_ipv4)
    mac = property(get_mac)
    ipv6 = property(get_ipv6)
    # 定义属性 hostnames，使用 get_hostnames 方法获取值
    hostnames = property(get_hostnames)
    # 定义属性 hosts，使用 get_hosts 方法获取值
    hosts = property(get_hosts)
    # 定义属性 runstats，使用 get_runstats 方法获取值，使用 set_runstats 方法设置值
    runstats = property(get_runstats, set_runstats)
    # 定义属性 hosts_down，使用 get_hosts_down 方法获取值，使用 set_hosts_down 方法设置值
    hosts_down = property(get_hosts_down, set_hosts_down)
    # 定义属性 hosts_up，使用 get_hosts_up 方法获取值，使用 set_hosts_up 方法设置值
    hosts_up = property(get_hosts_up, set_hosts_up)
    # 定义属性 hosts_scanned，使用 get_hosts_scanned 方法获取值，使用 set_hosts_scanned 方法设置值
    hosts_scanned = property(get_hosts_scanned, set_hosts_scanned)
    # 定义属性 finish_time，使用 get_finish_time 方法获取值，使用 set_finish_time 方法设置值
    finish_time = property(get_finish_time, set_finish_time)
    # 定义属性 finish_epoc_time，使用 get_finish_epoc_time 方法获取值，使用 set_finish_epoc_time 方法设置值
    finish_epoc_time = property(get_finish_epoc_time, set_finish_epoc_time)
    # 定义属性 formatted_finish_date，使用 get_formatted_finish_date 方法获取值
    formatted_finish_date = property(get_formatted_finish_date)
    # 定义属性 start，使用 get_start 方法获取值，使用 set_start 方法设置值
    start = property(get_start, set_start)
    # 定义属性 scan_name，使用 get_scan_name 方法获取值，使用 set_scan_name 方法设置值
    scan_name = property(get_scan_name, set_scan_name)
    
    # 初始化 _num_services 变量
    _num_services = None
    # 初始化 _services_scanned 变量
    _services_scanned = None
# 创建一个 NmapParserSAX 类，继承自 ParserBasics 和 ContentHandler
class NmapParserSAX(ParserBasics, ContentHandler):
    # 初始化方法
    def __init__(self):
        # 调用 ParserBasics 类的初始化方法
        ParserBasics.__init__(self)
        # 调用 ContentHandler 类的初始化方法
        ContentHandler.__init__(self)

        # 用于存储 xml-stylesheet 处理指令中的文本，例如 'href="file:///usr/share/nmap/nmap.xsl" type="text/xsl"'
        self.xml_stylesheet_data = None

        # 以下变量用于标记当前解析的位置
        self.in_interactive_output = False
        self.in_run_stats = False
        self.in_host = False
        self.in_hostnames = False
        self.in_ports = False
        self.in_port = False
        self.in_os = False
        self.in_trace = False
        self.list_extraports = []

        # 用于存储文件名
        self.filename = None

        # 用于标记是否有未保存的内容
        self.unsaved = False

    # 设置解析器
    def set_parser(self, parser):
        self.parser = parser

    # 从文件对象 f 中解析 Nmap XML 文件
    def parse(self, f):
        self.parser.parse(f)

    # 从指定文件名的文件中解析 Nmap XML 文件
    def parse_file(self, filename):
        # 打开文件
        with open(filename, "r") as f:
            # 调用 parse 方法解析文件
            self.parse(f)
            # 存储文件名
            self.filename = filename

    # 解析 nmaprun 标签
    def _parse_nmaprun(self, attrs):
        run_tag = "nmaprun"

        # 如果 nmap_output 为空并且 attrs 中包含 nmap_output，则将其赋值给 nmap_output
        if self.nmap_output == "" and "nmap_output" in attrs:
            self.nmap_output = attrs["nmap_output"]
        # 将 attrs 中的一些属性值存储到 nmap 字典中
        self.nmap[run_tag]["profile_name"] = attrs.get("profile_name", "")
        self.nmap[run_tag]["start"] = attrs.get("start", "")
        self.nmap[run_tag]["args"] = attrs.get("args", "")
        self.nmap[run_tag]["scanner"] = attrs.get("scanner", "")
        self.nmap[run_tag]["version"] = attrs.get("version", "")
        self.nmap[run_tag]["xmloutputversion"] = attrs.get("xmloutputversion", "")

        # 将 nmap 字典中的 args 值赋给 nmap_command
        self.nmap_command = self.nmap[run_tag]["args"]
    # 解析输出属性，如果类型不是交互式，则返回
    def _parse_output(self, attrs):
        if attrs.get("type") != "interactive":
            return
        # 如果已经在交互式输出中，则抛出异常
        if self.in_interactive_output:
            raise SAXException("Unexpected nested \"output\" element.")
        # 设置标志表示当前在交互式输出中
        self.in_interactive_output = True
        # 初始化 nmap_output 为空字符串
        self.nmap_output = ""

    # 解析扫描信息属性，将属性存储到字典中，并添加到 nmap["scaninfo"] 列表中
    def _parse_scaninfo(self, attrs):
        dic = {}
        dic["type"] = unique(attrs.get("type", ""))
        dic["protocol"] = unique(attrs.get("protocol", ""))
        dic["numservices"] = attrs.get("numservices", "")
        dic["services"] = attrs.get("services", "")
        self.nmap["scaninfo"].append(dic)

    # 解析详细信息属性，将详细级别存储到 nmap["verbose"] 中
    def _parse_verbose(self, attrs):
        self.nmap["verbose"] = attrs.get("level", "")

    # 解析调试信息属性，将调试级别存储到 nmap["debugging"] 中
    def _parse_debugging(self, attrs):
        self.nmap["debugging"] = attrs.get("level", "")

    # 解析运行统计完成信息属性，将完成时间存储到 nmap["runstats"]["finished_time"] 中
    def _parse_runstats_finished(self, attrs):
        self.nmap["runstats"]["finished_time"] = attrs.get("time", "")

    # 解析运行统计主机信息属性，将主机状态信息存储到 nmap["runstats"] 中
    def _parse_runstats_hosts(self, attrs):
        self.nmap["runstats"]["hosts_up"] = attrs.get("up", "")
        self.nmap["runstats"]["hosts_down"] = attrs.get("down", "")
        self.nmap["runstats"]["hosts_scanned"] = attrs.get("total", "")

    # 解析主机信息属性，初始化 HostInfo 对象，并设置注释信息
    def _parse_host(self, attrs):
        self.host_info = HostInfo()
        self.host_info.comment = attrs.get("comment", "")

    # 解析主机状态属性，设置主机状态信息到 HostInfo 对象中
    def _parse_host_status(self, attrs):
        self.host_info.set_state(unique(attrs.get("state", "")))

    # 解析主机地址属性，根据地址类型设置 IP、IPv6 或 MAC 地址信息到 HostInfo 对象中
    def _parse_host_address(self, attrs):
        address_attributes = {"type": unique(attrs.get("addrtype", "")),
                              "vendor": attrs.get("vendor", ""),
                              "addr": attrs.get("addr", "")}
        if address_attributes["type"] == "ipv4":
            self.host_info.set_ip(address_attributes)
        elif address_attributes["type"] == "ipv6":
            self.host_info.set_ipv6(address_attributes)
        elif address_attributes["type"] == "mac":
            self.host_info.set_mac(address_attributes)
    # 解析主机的主机名和主机类型，并添加到主机名列表中
    def _parse_host_hostname(self, attrs):
        self.list_hostnames.append({"hostname": attrs.get("name", ""),
                                    "hostname_type": attrs.get("type", "")})
    
    # 解析主机的额外端口信息，并添加到额外端口列表中
    def _parse_host_extraports(self, attrs):
        self.list_extraports.append({"state": unique(attrs.get("state", "")),
                                     "count": attrs.get("count", "")})
    
    # 解析主机的端口信息，并存储在端口字典中
    def _parse_host_port(self, attrs):
        self.dic_port = {"protocol": unique(attrs.get("protocol", "")),
                         "portid": unique(attrs.get("portid", ""))}
    
    # 解析主机的端口状态信息，并添加到端口字典中
    def _parse_host_port_state(self, attrs):
        self.dic_port["port_state"] = unique(attrs.get("state", ""))
        self.dic_port["reason"] = unique(attrs.get("reason", ""))
        self.dic_port["reason_ttl"] = unique(attrs.get("reason_ttl", ""))
    
    # 解析主机的端口服务信息，并添加到端口字典中
    def _parse_host_port_service(self, attrs):
        self.dic_port["service_name"] = attrs.get("name", "")
        self.dic_port["service_method"] = unique(attrs.get("method", ""))
        self.dic_port["service_conf"] = attrs.get("conf", "")
        self.dic_port["service_product"] = attrs.get("product", "")
        self.dic_port["service_version"] = attrs.get("version", "")
        self.dic_port["service_extrainfo"] = attrs.get("extrainfo", "")
    
    # 解析主机的操作系统匹配信息，并添加到操作系统匹配列表中
    def _parse_host_osmatch(self, attrs):
        osmatch = self._parsing(attrs, [], ['name', 'accuracy', 'line'])
        osmatch['osclasses'] = []
        self.list_osmatch.append(osmatch)
    
    # 解析主机的使用端口信息，并添加到使用端口列表中
    def _parse_host_portused(self, attrs):
        self.list_portused.append(self._parsing(
            attrs, ['state', 'proto', 'portid'], []))
    
    # 解析主机的操作系统类别信息，并添加到操作系统类别列表中
    def _parse_host_osclass(self, attrs):
        self.list_osclass.append(self._parsing(
            attrs, ['type', 'vendor', 'osfamily', 'osgen'], ['accuracy']))
    # 返回一个字典，包含给定标签的属性，属性名作为键，对应的值作为值
    def _parsing(self, attrs, unique_names, other_names):
        dic = {}
        # 遍历唯一属性名列表，将属性名和对应值添加到字典中
        for at in unique_names:
            dic[at] = unique(attrs.get(at, ""))
        # 遍历其他属性名列表，将属性名和对应值添加到字典中
        for at in other_names:
            dic[at] = attrs.get(at, "")
        # 返回结果字典
        return dic

    # 解析主机的运行时间信息
    def _parse_host_uptime(self, attrs):
        self.host_info.set_uptime(self._parsing(
            attrs, [], ["seconds", "lastboot"]))

    # 解析主机的 TCP 序列信息
    def _parse_host_tcpsequence(self, attrs):
        self.host_info.set_tcpsequence(self._parsing(
            attrs, ['difficulty'], ['index', 'values']))

    # 解析主机的 TCP 时间戳序列信息
    def _parse_host_tcptssequence(self, attrs):
        self.host_info.set_tcptssequence(self._parsing(
            attrs, ['class'], ['values']))

    # 解析主机的 IP ID 序列信息
    def _parse_host_ipidsequence(self, attrs):
        self.host_info.set_ipidsequence(self._parsing(
            attrs, ['class'], ['values']))

    # 解析主机的跟踪信息
    def _parse_host_trace(self, attrs):
        trace = {}
        # 遍历属性列表，将属性名和对应值添加到跟踪信息字典中
        for attr in ["proto", "port"]:
            trace[attr] = unique(attrs.get(attr, ""))
        self.host_info.set_trace(trace)

    # 解析主机的跟踪跳数信息
    def _parse_host_trace_hop(self, attrs):
        # 解析跳数信息，将属性名和对应值添加到字典中
        hop = self._parsing(attrs, [], ["ttl", "rtt", "ipaddr", "host"])
        self.host_info.append_trace_hop(hop)

    # 解析主机的跟踪错误信息
    def _parse_host_trace_error(self, attrs):
        self.host_info.set_trace_error(unique(attrs.get("errorstr", "")))

    # 处理处理指令
    def processingInstruction(self, target, data):
        # 如果处理指令为 "xml-stylesheet"，则将数据保存到 xml_stylesheet_data 中
        if target == "xml-stylesheet":
            self.xml_stylesheet_data = data
    # 当 XML 结束标签为 "output" 时，将 in_interactive_output 置为 False
    def endElement(self, name):
        if name == "output":
            self.in_interactive_output = False
        # 当 XML 结束标签为 "runstats" 时，将 in_run_stats 置为 False
        elif name == "runstats":
            self.in_run_stats = False
        # 当 XML 结束标签为 "host" 时
        elif name == "host":
            self.in_host = False
            # 设置 host_info 的额外端口信息
            self.host_info.set_extraports(self.list_extraports)
            # 设置 host_info 的端口信息
            self.host_info.set_ports(self.list_ports)
            # 将 host_info 添加到 nmap 字典的 "hosts" 键对应的值中
            self.nmap["hosts"].append(self.host_info)
        # 当处于 host 标签内，并且 XML 结束标签为 "hostnames" 时
        elif self.in_host and name == "hostnames":
            self.in_hostnames = False
            # 设置 host_info 的主机名信息
            self.host_info.set_hostnames(self.list_hostnames)
        # 当处于 host 标签内，并且 XML 结束标签为 "ports" 时
        elif self.in_host and name == "ports":
            self.in_ports = False
        # 当处于 host 标签内，并且处于 ports 标签内，并且 XML 结束标签为 "port" 时
        elif self.in_host and self.in_ports and name == "port":
            self.in_port = False
            # 将端口信息添加到 list_ports 中，并删除 dic_port
            self.list_ports.append(self.dic_port)
            del(self.dic_port)
        # 当处于 host 标签内，并且处于 os 标签内，并且 XML 结束标签为 "osmatch" 时
        elif self.in_host and self.in_os and name == "osmatch":
            # 将 list_osclass 添加到 list_osmatch 的最后一个元素的 "osclasses" 键对应的值中，并清空 list_osclass
            self.list_osmatch[-1]['osclasses'].extend(self.list_osclass)
            self.list_osclass = []
        # 当处于 host 标签内，并且处于 os 标签内，并且 XML 结束标签为 "os" 时
        elif self.in_host and self.in_os and name == "os":
            self.in_os = False
            # 设置 host_info 的使用端口信息
            self.host_info.set_ports_used(self.list_portused)
            # 设置 host_info 的操作系统匹配信息
            self.host_info.set_osmatches(self.list_osmatch)
            # 删除 list_portused 和 list_osmatch
            del(self.list_portused)
            del(self.list_osmatch)
        # 当处于 host 标签内，并且处于 trace 标签内，并且 XML 结束标签为 "trace" 时
        elif self.in_host and self.in_trace and name == "trace":
            self.in_trace = False

    # 如果处于 in_interactive_output 状态，将 content 写入 _nmap_output
    def characters(self, content):
        if self.in_interactive_output:
            self._nmap_output.write(content)

    # 将该对象的 Nmap 文本输出写入文件对象 f
    def write_text(self, f):
        """Write the Nmap text output of this object to the file-like object
        f."""
        # 如果 nmap_output 为空，则返回
        if self.nmap_output == "":
            return
        # 将 nmap_output 写入文件对象 f
        f.write(self.nmap_output)
    # 将对象的 XML 表示写入文件对象 f 中
    def write_xml(self, f):
        # 创建 XML 生成器对象
        writer = XMLGenerator(f)
        # 开始 XML 文档
        writer.startDocument()
        # 如果存在 XML 样式表数据，则添加处理指令
        if self.xml_stylesheet_data is not None:
            writer.processingInstruction(
                    "xml-stylesheet", self.xml_stylesheet_data)
        # 写入 nmaprun 元素
        self._write_nmaprun(writer)
        # 写入 scaninfo 元素
        self._write_scaninfo(writer)
        # 写入 verbose 元素
        self._write_verbose(writer)
        # 写入 debugging 元素
        self._write_debugging(writer)
        # 写入 output 元素
        self._write_output(writer)
        # 写入 hosts 元素
        self._write_hosts(writer)
        # 写入 runstats 元素
        self._write_runstats(writer)
        # 结束 nmaprun 元素
        writer.endElement("nmaprun")
        # 结束 XML 文档
        writer.endDocument()

    # 返回包含此扫描的 XML 表示的字符串
    def get_xml(self):
        # 创建字符串缓冲区
        buffer = StringIO()
        # 将 XML 写入缓冲区
        self.write_xml(buffer)
        # 获取缓冲区中的字符串
        string = buffer.getvalue()
        # 关闭缓冲区
        buffer.close()
        # 返回字符串
        return string

    # 将此扫描的 XML 表示写入给定文件名的文件中
    def write_xml_to_file(self, filename):
        # 打开文件
        fd = open(filename, "w")
        # 将 XML 写入文件
        self.write_xml(fd)
        # 关闭文件
        fd.close()

    # 写入 output 元素
    def _write_output(self, writer):
        # 如果 nmap_output 为空，则返回
        if self.nmap_output == "":
            return
        # 开始 output 元素
        writer.startElement("output", Attributes({"type": "interactive"}))
        # 写入 nmap_output 数据
        writer.characters(self.nmap_output)
        # 结束 output 元素
        writer.endElement("output")
    # 写入运行统计信息到 XML 写入器中
    def _write_runstats(self, writer):
        # 开始 runstats 元素
        writer.startElement("runstats", Attributes(dict()))
    
        # 开始 finished 元素
        writer.startElement("finished",
                        Attributes(dict(time=str(self.finish_epoc_time),
                                        timestr=time.ctime(time.mktime(
                                            self.get_finish_time())))))
        # 结束 finished 元素
        writer.endElement("finished")
    
        # 开始 hosts 元素
        writer.startElement("hosts",
                            Attributes(dict(up=str(self.hosts_up),
                                            down=str(self.hosts_down),
                                            total=str(self.hosts_scanned))))
        # 结束 hosts 元素
        writer.endElement("hosts")
    
        # 结束 runstats 元素
        writer.endElement("runstats")
        # 结束 runstats 元素的注释
        #########################
    
    # 写入调试信息到 XML 写入器中
    def _write_debugging(self, writer):
        # 开始 debugging 元素
        writer.startElement("debugging", Attributes(dict(
                                            level=str(self.debugging_level))))
        # 结束 debugging 元素
        writer.endElement("debugging")
    
    # 写入详细信息到 XML 写入器中
    def _write_verbose(self, writer):
        # 开始 verbose 元素
        writer.startElement("verbose", Attributes(dict(
                                            level=str(self.verbose_level))))
        # 结束 verbose 元素
        writer.endElement("verbose")
    
    # 写入扫描信息到 XML 写入器中
    def _write_scaninfo(self, writer):
        for scan in self.scaninfo:
            # 开始 scaninfo 元素
            writer.startElement("scaninfo",
                Attributes(dict(type=scan.get("type", ""),
                                protocol=scan.get("protocol", ""),
                                numservices=scan.get("numservices", ""),
                                services=scan.get("services", ""))))
            # 结束 scaninfo 元素
            writer.endElement("scaninfo")
    # 写入 nmaprun 元素到 XML 文件中
    def _write_nmaprun(self, writer):
        # 开始 nmaprun 元素
        writer.startElement("nmaprun",
                # 设置 nmaprun 元素的属性
                Attributes(dict(args=str(self.nmap_command),
                                profile_name=str(self.profile_name),
                                scanner=str(self.scanner),
                                start=str(self.start),
                                startstr=time.ctime(
                                    time.mktime(self.get_date())),
                                version=str(self.scanner_version),
                                xmloutputversion=str(XML_OUTPUT_VERSION))))
    
    # 设置对象为未保存状态
    def set_unsaved(self):
        self.unsaved = True
    
    # 检查对象是否为未保存状态
    def is_unsaved(self):
        return self.unsaved
class OverrideEntityResolver(EntityResolver):
    """This class overrides the default behavior of xml.sax to download
    remote DTDs, instead returning blank strings"""
    # 创建一个空的字符串IO对象
    empty = StringIO()

    # 重写父类的resolveEntity方法，返回空字符串
    def resolveEntity(self, publicId, systemId):
        return OverrideEntityResolver.empty


def nmap_parser_sax():
    # 创建一个解析器对象
    parser = make_parser()
    # 创建一个NmapParserSAX对象
    nmap_parser = NmapParserSAX()

    # 设置解析器对象的内容处理器为nmap_parser
    parser.setContentHandler(nmap_parser)
    # 设置解析器对象的实体解析器为OverrideEntityResolver对象
    parser.setEntityResolver(OverrideEntityResolver())
    # 设置nmap_parser对象的解析器为parser
    nmap_parser.set_parser(parser)

    # 返回nmap_parser对象
    return nmap_parser

# 将nmap_parser_sax函数赋值给NmapParser变量
NmapParser = nmap_parser_sax


if __name__ == '__main__':
    import sys

    # 从命令行参数中获取要解析的文件名
    file_to_parse = sys.argv[1]

    # 创建一个NmapParser对象
    np = NmapParser()
    # 解析指定文件
    np.parse_file(file_to_parse)

    # 遍历解析结果中的主机信息
    for host in np.hosts:
        print("%s:" % host.ip["addr"])
        print("  Comment:", repr(host.comment))
        print("  TCP sequence:", repr(host.tcpsequence))
        print("  TCP TS sequence:", repr(host.tcptssequence))
        print("  IP ID sequence:", repr(host.ipidsequence))
        print("  Uptime:", repr(host.uptime))
        print("  OS Match:", repr(host.osmatches))
        print("  Ports:")
        for p in host.ports:
            print("\t%s" % repr(p))
        print("  Ports used:", repr(host.ports_used))
        print("  OS Matches:", repr(host.osmatches))
        print("  Hostnames:", repr(host.hostnames))
        print("  IP:", repr(host.ip))
        print("  IPv6:", repr(host.ipv6))
        print("  MAC:", repr(host.mac))
        print("  State:", repr(host.state))
        if "hops" in host.trace:
            print("  Trace:")
            for hop in host.trace["hops"]:
                print("    ", repr(hop))
            print()
```