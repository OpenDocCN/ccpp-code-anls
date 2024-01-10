# `nmap\ndiff\ndiff.py`

```
#!/usr/bin/env python3

# Ndiff
#
# This programs reads two Nmap XML files and displays a list of their
# differences.
#
# Copyright 2021 Nmap Software LLC
# Ndiff is distributed under the same license as Nmap. See the file
# LICENSE in the Nmap source distribution or
# https://nmap.org/book/man-legal.html for more details.
#
# Original author was David Fifield based on a design by Michael Pattrick

import datetime  # 导入 datetime 模块
import difflib  # 导入 difflib 模块
import getopt  # 导入 getopt 模块
import sys  # 导入 sys 模块
import time  # 导入 time 模块

# Prevent loading PyXML
import xml  # 导入 xml 模块
xml.__path__ = [x for x in xml.__path__ if "_xmlplus" not in x]  # 防止加载 PyXML

import xml.sax  # 导入 xml.sax 模块
import xml.sax.saxutils  # 导入 xml.sax.saxutils 模块
import xml.dom.minidom  # 导入 xml.dom.minidom 模块
from io import StringIO  # 从 io 模块中导入 StringIO 类

verbose = False  # 设置 verbose 变量为 False

NDIFF_XML_VERSION = "1"  # 设置 NDIFF_XML_VERSION 变量为 "1"


class OverrideEntityResolver(xml.sax.handler.EntityResolver):
    """This class overrides the default behavior of xml.sax to download
    remote DTDs, instead returning blank strings"""
    empty = StringIO()  # 创建一个空的 StringIO 对象

    def resolveEntity(self, publicId, systemId):
        return OverrideEntityResolver.empty  # 重写 resolveEntity 方法，返回空的 StringIO 对象


class Scan(object):
    """A single Nmap scan, corresponding to a single invocation of Nmap. It is
    a container for a list of hosts. It also has utility methods to load itself
    from an Nmap XML file."""
    def __init__(self):
        self.scanner = None  # 初始化 scanner 属性为 None
        self.version = None  # 初始化 version 属性为 None
        self.args = None  # 初始化 args 属性为 None
        self.start_date = None  # 初始化 start_date 属性为 None
        self.end_date = None  # 初始化 end_date 属性为 None
        self.hosts = []  # 初始化 hosts 属性为一个空列表
        self.pre_script_results = []  # 初始化 pre_script_results 属性为一个空列表
        self.post_script_results = []  # 初始化 post_script_results 属性为一个空列表

    def sort_hosts(self):
        self.hosts.sort(key=lambda h: h.get_id())  # 对 hosts 列表进行排序，排序规则为根据每个元素的 get_id 方法返回值

    def load(self, f):
        """Load a scan from the Nmap XML in the file-like object f."""
        parser = xml.sax.make_parser()  # 创建一个 SAX 解析器
        handler = NmapContentHandler(self)  # 创建一个 NmapContentHandler 对象，传入当前 Scan 对象
        parser.setEntityResolver(OverrideEntityResolver())  # 设置解析器的实体解析器为 OverrideEntityResolver 对象
        parser.setContentHandler(handler)  # 设置解析器的内容处理器为 handler 对象
        parser.parse(f)  # 解析传入的文件对象 f
    # 从给定的文件名加载 Nmap XML 文件扫描结果
    def load_from_file(self, filename):
        # 以只读方式打开文件
        with open(filename, "r") as f:
            # 调用 load 方法加载文件内容
            self.load(f)

    # 将 nmaprun 元素的开头写入到给定的 writer 中
    def write_nmaprun_open(self, writer):
        attrs = {}
        # 如果扫描器不为空，则添加到属性字典中
        if self.scanner is not None:
            attrs["scanner"] = self.scanner
        # 如果参数不为空，则添加到属性字典中
        if self.args is not None:
            attrs["args"] = self.args
        # 如果开始日期不为空，则添加到属性字典中
        if self.start_date is not None:
            attrs["start"] = "%d" % time.mktime(self.start_date.timetuple())
            attrs["startstr"] = self.start_date.strftime(
                    "%a %b %d %H:%M:%S %Y")
        # 如果版本不为空，则添加到属性字典中
        if self.version is not None:
            attrs["version"] = self.version
        # 使用属性字典开始 nmaprun 元素
        writer.startElement("nmaprun", attrs)

    # 将 nmaprun 元素的结尾写入到给定的 writer 中
    def write_nmaprun_close(self, writer):
        # 结束 nmaprun 元素
        writer.endElement("nmaprun")

    # 将 nmaprun 对象转换为 DOM 片段
    def nmaprun_to_dom_fragment(self, document):
        # 创建一个空的文档片段
        frag = document.createDocumentFragment()
        # 创建 nmaprun 元素
        elem = document.createElement("nmaprun")
        # 如果扫描器不为空，则设置属性
        if self.scanner is not None:
            elem.setAttribute("scanner", self.scanner)
        # 如果参数不为空，则设置属性
        if self.args is not None:
            elem.setAttribute("args", self.args)
        # 如果开始日期不为空，则设置属性
        if self.start_date is not None:
            elem.setAttribute(
                    "start", "%d" % time.mktime(self.start_date.timetuple()))
            elem.setAttribute(
                    "startstr",
                    self.start_date.strftime("%a %b %d %H:%M:%S %Y"))
        # 如果版本不为空，则设置属性
        if self.version is not None:
            elem.setAttribute("version", self.version)
        # 将创建的元素添加到文档片段中
        frag.appendChild(elem)
        # 返回文档片段
        return frag
class Host(object):
    """定义一个主机类，包括状态、地址、主机名、端口规格到端口的字典，以及 OS 匹配列表。主机状态为字符串，或者为“未知”的情况下为 None。"""
    def __init__(self):
        # 初始化主机状态为 None
        self.state = None
        # 初始化地址列表为空
        self.addresses = []
        # 初始化主机名列表为空
        self.hostnames = []
        # 初始化端口字典为空
        self.ports = {}
        # 初始化额外端口字典为空
        self.extraports = {}
        # 初始化 OS 匹配列表为空
        self.os = []
        # 初始化脚本结果列表为空
        self.script_results = []

    def get_id(self):
        """返回一个用于确定主机在扫描中是否“相同”的 ID。"""
        hid = None
        # 如果地址列表不为空，取地址列表中第一个地址作为 ID 的一部分
        if len(self.addresses) > 0:
            hid = "%-40s" % (str(sorted(self.addresses)[0]))
        # 如果主机名列表不为空，将主机名列表中第一个主机名作为 ID 的一部分
        if len(self.hostnames) > 0:
            return (hid or " " * 40) + str(sorted(self.hostnames)[0])
        # 如果地址和主机名列表都为空，返回对象的 ID
        return hid or id(self)

    def format_name(self):
        """返回一个可读的主机标识符。"""
        # 将地址列表中的地址按顺序连接成字符串
        address_s = ", ".join(a.s for a in sorted(self.addresses))
        # 将主机名列表中的主机名按顺序连接成字符串
        hostname_s = ", ".join(sorted(self.hostnames))
        # 如果主机名列表不为空
        if len(hostname_s) > 0:
            # 如果地址列表不为空，返回主机名和地址的组合
            if len(address_s) > 0:
                return "%s (%s)" % (hostname_s, address_s)
            # 如果地址列表为空，只返回主机名
            else:
                return hostname_s
        # 如果主机名列表为空
        elif len(address_s) > 0:
            # 返回地址列表中的地址
            return address_s
        # 如果地址和主机名列表都为空，返回无名称
        else:
            return "<no name>"

    def add_port(self, port):
        # 将端口添加到端口字典中
        self.ports[port.spec] = port

    def add_address(self, address):
        # 如果地址不在地址列表中，将地址添加到地址列表中
        if address not in self.addresses:
            self.addresses.append(address)

    def add_hostname(self, hostname):
        # 如果主机名不在主机名列表中，将主机名添加到主机名列表中
        if hostname not in self.hostnames:
            self.hostnames.append(hostname)

    def is_extraports(self, state):
        # 判断是否为额外端口
        return state is None or state in self.extraports

    def extraports_string(self):
        # 将额外端口字典转换为字符串
        locallist = [(count, state) for (state, count) in list(self.extraports.items())]
        # 按计数值逆序排序
        locallist.sort(reverse=True)
        return ", ".join(
                ["%d %s ports" % (count, state) for (count, state) in locallist])
    # 将状态转换为文档片段
    def state_to_dom_fragment(self, document):
        # 创建一个空的文档片段
        frag = document.createDocumentFragment()
        # 如果状态不为空
        if self.state is not None:
            # 创建一个名为 "status" 的元素
            elem = document.createElement("status")
            # 设置状态属性
            elem.setAttribute("state", self.state)
            # 将元素添加到文档片段中
            frag.appendChild(elem)
        # 返回文档片段
        return frag
    
    # 将主机名转换为文档片段
    def hostname_to_dom_fragment(self, document, hostname):
        # 创建一个空的文档片段
        frag = document.createDocumentFragment()
        # 创建一个名为 "hostname" 的元素
        elem = document.createElement("hostname")
        # 设置主机名属性
        elem.setAttribute("name", hostname)
        # 将元素添加到文档片段中
        frag.appendChild(elem)
        # 返回文档片段
        return frag
    
    # 将额外端口转换为文档片段
    def extraports_to_dom_fragment(self, document):
        # 创建一个空的文档片段
        frag = document.createDocumentFragment()
        # 遍历额外端口字典中的状态和计数
        for state, count in list(self.extraports.items()):
            # 创建一个名为 "extraports" 的元素
            elem = document.createElement("extraports")
            # 设置状态属性
            elem.setAttribute("state", state)
            # 设置计数属性
            elem.setAttribute("count", str(count))
            # 将元素添加到文档片段中
            frag.appendChild(elem)
        # 返回文档片段
        return frag
    
    # 将操作系统转换为文档片段
    def os_to_dom_fragment(self, document, os):
        # 创建一个空的文档片段
        frag = document.createDocumentFragment()
        # 创建一个名为 "osmatch" 的元素
        elem = document.createElement("osmatch")
        # 设置操作系统名称属性
        elem.setAttribute("name", os)
        # 将元素添加到文档片段中
        frag.appendChild(elem)
        # 返回文档片段
        return frag
    # 将对象转换为文档片段
    def to_dom_fragment(self, document):
        # 创建一个空的文档片段
        frag = document.createDocumentFragment()
        # 创建一个名为"host"的元素节点
        elem = document.createElement("host")

        # 如果状态不为空，则将状态转换为文档片段并添加到"host"元素节点中
        if self.state is not None:
            elem.appendChild(self.state_to_dom_fragment(document))

        # 遍历地址列表，将每个地址对象转换为文档片段并添加到"host"元素节点中
        for addr in self.addresses:
            elem.appendChild(addr.to_dom_fragment(document))

        # 如果主机名列表不为空，则创建一个名为"hostnames"的元素节点，并将每个主机名转换为文档片段添加到其中，最后将"hostnames"元素节点添加到"host"元素节点中
        if len(self.hostnames) > 0:
            hostnames_elem = document.createElement("hostnames")
            for hostname in self.hostnames:
                hostnames_elem.appendChild(
                        self.hostname_to_dom_fragment(document, hostname))
            elem.appendChild(hostnames_elem)

        # 创建一个名为"ports"的元素节点，并将额外端口转换为文档片段添加到其中，然后将每个端口对象转换为文档片段并添加到"ports"元素节点中，最后将"ports"元素节点添加到"host"元素节点中
        ports_elem = document.createElement("ports")
        ports_elem.appendChild(self.extraports_to_dom_fragment(document))
        for port in sorted(self.ports.values()):
            if not self.is_extraports(port.state):
                ports_elem.appendChild(port.to_dom_fragment(document))
        if ports_elem.hasChildNodes():
            elem.appendChild(ports_elem)

        # 如果操作系统列表不为空，则创建一个名为"os"的元素节点，并将每个操作系统对象转换为文档片段添加到其中，最后将"os"元素节点添加到"host"元素节点中
        if len(self.os) > 0:
            os_elem = document.createElement("os")
            for os in self.os:
                os_elem.appendChild(self.os_to_dom_fragment(document, os))
            elem.appendChild(os_elem)

        # 如果脚本结果列表不为空，则创建一个名为"hostscript"的元素节点，并将每个脚本结果对象转换为文档片段添加到其中，最后将"hostscript"元素节点添加到"host"元素节点中
        if len(self.script_results) > 0:
            hostscript_elem = document.createElement("hostscript")
            for sr in self.script_results:
                hostscript_elem.appendChild(sr.to_dom_fragment(document))
            elem.appendChild(hostscript_elem)

        # 将"host"元素节点添加到文档片段中
        frag.appendChild(elem)
        # 返回文档片段
        return frag
# 定义一个地址类
class Address(object):
    # 初始化方法，接受一个字符串参数
    def __init__(self, s):
        self.s = s

    # 判断两个地址对象是否相等
    def __eq__(self, other):
        return self.sort_key() == other.sort_key()

    # 判断两个地址对象是否不相等
    def __ne__(self, other):
        return not self.__eq__(other)

    # 返回地址对象的哈希值
    def __hash__(self):
        return hash(self.sort_key())

    # 判断地址对象的大小关系
    def __lt__(self, other):
        return self.sort_key() < other.sort_key()

    # 返回地址对象的字符串表示
    def __str__(self):
        return str(self.s)

    # 返回地址对象的 Unicode 表示
    def __unicode__(self):
        return self.s

    # 根据类型和字符串创建新的地址对象
    def new(type, s):
        if type == "ipv4":
            return IPv4Address(s)
        elif type == "ipv6":
            return IPv6Address(s)
        elif type == "mac":
            return MACAddress(s)
        else:
            raise ValueError("Unknown address type %s." % type)
    new = staticmethod(new)

    # 根据文档创建地址对象的 DOM 片段
    def to_dom_fragment(self, document):
        frag = document.createDocumentFragment()
        elem = document.createElement("address")
        elem.setAttribute("addr", self.s)
        elem.setAttribute("addrtype", self.type)
        frag.appendChild(elem)
        return frag

# IPv4Address 类，继承自 Address 类
class IPv4Address(Address):
    # 类型属性，返回地址类型
    type = property(lambda self: "ipv4")

    # 返回排序关键字
    def sort_key(self):
        return (0, self.s)

# IPv6Address 类，继承自 Address 类
class IPv6Address(Address):
    # 类型属性，返回地址类型
    type = property(lambda self: "ipv6")

    # 返回排序关键字
    def sort_key(self):
        return (1, self.s)

# MACAddress 类，继承自 Address 类
class MACAddress(Address):
    # 类型属性，返回地址类型
    type = property(lambda self: "mac")

    # 返回排序关键字
    def sort_key(self):
        return (2, self.s)

# 端口类
class Port(object):
    """A single port, consisting of a port specification, a state, and a
    service version. A specification, or "spec," is the 2-tuple (number,
    protocol). So (10, "tcp") corresponds to the port 10/tcp. Port states are
    strings, or None for "unknown"."""
    # 初始化方法，接受 spec 和 state 两个参数，初始化实例的 spec, state, service 和 script_results 属性
    def __init__(self, spec, state=None):
        self.spec = spec
        self.state = state
        self.service = Service()  # 初始化一个 Service 实例并赋值给 self.service
        self.script_results = []  # 初始化一个空列表赋值给 self.script_results
    
    # 返回状态的字符串表示，如果状态为 None，则返回 "unknown"，否则返回状态的字符串形式
    def state_string(self):
        if self.state is None:
            return "unknown"
        else:
            return str(self.state)
    
    # 返回规范的字符串表示，使用 spec 属性的值进行格式化
    def spec_string(self):
        return "%d/%s" % self.spec
    
    # 计算实例的哈希值，使用 spec 属性的哈希值作为返回值
    def __hash__(self):
        return hash(self.spec)
    
    # 比较方法，用于实例之间的比较，比较顺序为 spec, service, script_results
    def __lt__(self, other):
        return (self.spec, self.service, self.script_results) < (
            other.spec, other.service, other.script_results)
    
    # 将实例转换为 DOM 片段，使用 document 参数创建一个 DocumentFragment，然后创建一个 "port" 元素，设置其属性，根据状态创建 "state" 元素，添加到 "port" 元素中，然后添加 service 和 script_results 到 "port" 元素中，最后将 "port" 元素添加到 DocumentFragment 中并返回
    def to_dom_fragment(self, document):
        frag = document.createDocumentFragment()
        elem = document.createElement("port")
        elem.setAttribute("portid", str(self.spec[0]))
        elem.setAttribute("protocol", self.spec[1])
        if self.state is not None:
            state_elem = document.createElement("state")
            state_elem.setAttribute("state", self.state)
            elem.appendChild(state_elem)
        elem.appendChild(self.service.to_dom_fragment(document))
        for sr in self.script_results:
            elem.appendChild(sr.to_dom_fragment(document))
        frag.appendChild(elem)
        return frag
class Service(object):
    """A service version as determined by -sV scan. Also contains the looked-up
    port name if -sV wasn't used."""
    # 初始化 Service 类
    def __init__(self):
        # 初始化属性
        self.name = None
        self.product = None
        self.version = None
        self.extrainfo = None
        self.tunnel = None

        # self.hostname = None
        # self.ostype = None
        # self.devicetype = None

    # 定义 __hash__ 方法
    __hash__ = None

    # 定义 __eq__ 方法，用于判断两个对象是否相等
    def __eq__(self, other):
        return self.name == other.name \
            and self.product == other.product \
            and self.version == other.version \
            and self.extrainfo == other.extrainfo

    # 定义 __ne__ 方法，用于判断两个对象是否不相等
    def __ne__(self, other):
        return not self.__eq__(other)

    # 获取服务名称的字符串表示
    def name_string(self):
        parts = []
        if self.tunnel is not None:
            parts.append(self.tunnel)
        if self.name is not None:
            parts.append(self.name)

        if len(parts) == 0:
            return None
        else:
            return "/".join(parts)

    # 获取版本信息的字符串表示
    def version_string(self):
        """Get a string like in the VERSION column of Nmap output."""
        parts = []
        if self.product is not None:
            parts.append(self.product)
        if self.version is not None:
            parts.append(self.version)
        if self.extrainfo is not None:
            parts.append("(%s)" % self.extrainfo)

        if len(parts) == 0:
            return None
        else:
            return " ".join(parts)

    # 将对象转换为 DOM 片段
    def to_dom_fragment(self, document):
        frag = document.createDocumentFragment()
        elem = document.createElement("service")
        for attr in ("name", "product", "version", "extrainfo", "tunnel"):
            v = getattr(self, attr)
            if v is None:
                continue
            elem.setAttribute(attr, v)
        if len(elem.attributes) > 0:
            frag.appendChild(elem)
        return frag


class ScriptResult(object):
    # 初始化 ScriptResult 类
    def __init__(self):
        # 初始化属性
        self.id = None
        self.output = None
    # 设置__hash__属性为None
    __hash__ = None
    
    # 定义对象的相等性比较方法，比较id和output是否相等
    def __eq__(self, other):
        return self.id == other.id and self.output == other.output
    
    # 定义对象的不相等性比较方法，通过调用相等性比较方法的结果取反得到
    def __ne__(self, other):
        return not self.__eq__(other)
    
    # 获取输出内容的每一行，并对其进行处理
    def get_lines(self):
        result = []
        lines = self.output.splitlines()
        # 如果输出内容不为空，则在第一行添加id信息
        if len(lines) > 0:
            lines[0] = self.id + ": " + lines[0]
        # 将每一行添加到结果列表中，并添加前缀"|  "
        for line in lines[:-1]:
            result.append("|  " + line)
        # 如果输出内容不为空，则将最后一行添加到结果列表中，并添加前缀"|_ "
        if len(lines) > 0:
            result.append("|_ " + lines[-1])
        return result
    
    # 将对象转换为DOM片段
    def to_dom_fragment(self, document):
        frag = document.createDocumentFragment()
        elem = document.createElement("script")
        elem.setAttribute("id", self.id)
        elem.setAttribute("output", self.output)
        frag.appendChild(elem)
        return frag
# 格式化启动横幅，类似于 Nmap 的格式
def format_banner(scan):
    # 设置默认的扫描器为 "Nmap"
    scanner = "Nmap"
    # 如果扫描器不为空且不为 "nmap"，则使用扫描器的值
    if scan.scanner is not None and scan.scanner != "nmap":
        scanner = scan.scanner
    # 创建部分列表，包括扫描器、版本、扫描开始时间和参数
    parts = [scanner]
    if scan.version is not None:
        parts.append(scan.version)
    parts.append("scan")
    if scan.start_date is not None:
        parts.append("initiated %s" % scan.start_date.strftime(
            "%a %b %d %H:%M:%S %Y"))
    if scan.args is not None:
        parts.append("as: %s" % scan.args)
    # 将部分列表中的元素用空格连接成字符串
    return " ".join(parts)


def print_script_result_diffs_text(title, script_results_a, script_results_b,
        script_result_diffs, f=sys.stdout):
    # 创建一个表格对象
    table = Table("*")
    # 遍历脚本结果的差异列表，将每个差异添加到表格中
    for sr_diff in script_result_diffs:
        sr_diff.append_to_port_table(table)
    # 如果表格中有内容
    if len(table) > 0:
        # 打印空行
        print(file=f)
        # 根据脚本结果的情况打印标题
        if len(script_results_b) == 0:
            print("-%s:" % title, file=f)
        elif len(script_results_a) == 0:
            print("+%s:" % title, file=f)
        else:
            print(" %s:" % title, file=f)
        # 打印表格内容
        print(table, file=f)


def script_result_diffs_to_dom_fragment(elem, script_results_a,
        script_results_b, script_result_diffs, document):
    # 如果两个脚本结果列表都为空，则返回一个空的文档片段
    if len(script_results_a) == 0 and len(script_results_b) == 0:
        return document.createDocumentFragment()
    # 如果脚本结果列表 B 为空
    elif len(script_results_b) == 0:
        # 创建一个 <a> 元素，将脚本结果 A 转换为 DOM 片段并添加到 <a> 元素中，然后返回 <a> 元素
        a_elem = document.createElement("a")
        for sr in script_results_a:
            elem.appendChild(sr.to_dom_fragment(document))
        a_elem.appendChild(elem)
        return a_elem
    # 如果脚本结果列表 A 为空
    elif len(script_results_a) == 0:
        # 创建一个 <b> 元素，将脚本结果 B 转换为 DOM 片段并添加到 <b> 元素中，然后返回 <b> 元素
        b_elem = document.createElement("b")
        for sr in script_results_b:
            elem.appendChild(sr.to_dom_fragment(document))
        b_elem.appendChild(elem)
        return b_elem
    # 如果两个脚本结果列表都不为空
    else:
        # 遍历脚本结果的差异列表，将每个差异转换为 DOM 片段并添加到元素中，然后返回元素
        for sr_diff in script_result_diffs:
            elem.appendChild(sr_diff.to_dom_fragment(document))
        return elem


def host_pairs(a, b):
    # 定义一个函数，接受两个已按 id 排序的主机列表 a 和 b，并返回配对的主机
    # 当两个列表的头部具有相同的 id 时，它们一起返回
    # 否则，较小 id 的主机将与一个空主机作为配对返回，而较大 id 的主机将保留在列表中以供以后的迭代使用
    i = 0  # 初始化列表 a 的索引
    j = 0  # 初始化列表 b 的索引
    while i < len(a) and j < len(b):  # 当 a 和 b 的索引都未超出列表长度时执行循环
        if a[i].get_id() < b[j].get_id():  # 如果列表 a 的当前主机 id 小于列表 b 的当前主机 id
            yield a[i], Host()  # 返回列表 a 的当前主机和一个空主机作为配对
            i += 1  # 增加列表 a 的索引
        elif a[i].get_id() > b[j].get_id():  # 如果列表 a 的当前主机 id 大于列表 b 的当前主机 id
            yield Host(), b[j]  # 返回一个空主机和列表 b 的当前主机作为配对
            j += 1  # 增加列表 b 的索引
        else:  # 如果列表 a 和列表 b 的当前主机 id 相等
            yield a[i], b[j]  # 返回列表 a 和列表 b 的当前主机作为配对
            i += 1  # 增加列表 a 的索引
            j += 1  # 增加列表 b 的索引
    while i < len(a):  # 当列表 b 已经遍历完，但列表 a 还有剩余主机时执行循环
        yield a[i], Host()  # 返回列表 a 的剩余主机和一个空主机作为配对
        i += 1  # 增加列表 a 的索引
    while j < len(b):  # 当列表 a 已经遍历完，但列表 b 还有剩余主机时执行循环
        yield Host(), b[j]  # 返回一个空主机和列表 b 的剩余主机作为配对
        j += 1  # 增加列表 b 的索引
class ScanDiff(object):
    """An abstract class for different diff output types. Subclasses must
    define various output methods."""
    # 初始化方法，创建一个 ScanDiff 对象，接受两个扫描对象和一个文件对象作为参数
    def __init__(self, scan_a, scan_b, f=sys.stdout):
        """Create a ScanDiff from the "before" scan_a and the "after"
        scan_b."""
        # 将传入的扫描对象和文件对象保存到实例变量中
        self.scan_a = scan_a
        self.scan_b = scan_b
        self.f = f

    # 输出方法，对扫描对象进行排序，并输出不同之处
    def output(self):
        # 对扫描对象的主机列表进行排序
        self.scan_a.sort_hosts()
        self.scan_b.sort_hosts()

        # 输出开始部分
        self.output_beginning()

        # 对扫描对象的前置脚本结果进行比较，并输出差异
        pre_script_result_diffs = ScriptResultDiff.diff_lists(
                self.scan_a.pre_script_results, self.scan_b.pre_script_results)
        self.output_pre_scripts(pre_script_result_diffs)

        cost = 0
        # 目前我们不考虑具有不同 id（地址或主机名）的主机进行比较，这可能会导致更好的差异
        for host_a, host_b in host_pairs(self.scan_a.hosts, self.scan_b.hosts):
            h_diff = HostDiff(host_a, host_b)
            cost += h_diff.cost
            if h_diff.cost > 0 or verbose:
                self.output_host_diff(h_diff)

        # 对扫描对象的后置脚本结果进行比较，并输出差异
        post_script_result_diffs = ScriptResultDiff.diff_lists(
                self.scan_a.post_script_results,
                self.scan_b.post_script_results)
        self.output_post_scripts(post_script_result_diffs)

        # 输出结束部分
        self.output_ending()

        # 返回成本
        return cost


class ScanDiffText(ScanDiff):
    # 初始化方法，调用父类的初始化方法
    def __init__(self, scan_a, scan_b, f=sys.stdout):
        ScanDiff.__init__(self, scan_a, scan_b, f)

    # 输出开始部分的方法
    def output_beginning(self):
        banner_a = format_banner(self.scan_a)
        banner_b = format_banner(self.scan_b)
        if banner_a != banner_b:
            print("-%s" % banner_a, file=self.f)
            print("+%s" % banner_b, file=self.f)
        elif verbose:
            print(" %s" % banner_a, file=self.f)
    # 输出预扫描脚本结果的差异
    def output_pre_scripts(self, pre_script_result_diffs):
        # 打印预扫描脚本结果的文本
        print_script_result_diffs_text("Pre-scan script results",
            self.scan_a.pre_script_results, self.scan_b.pre_script_results,
            pre_script_result_diffs, self.f)
    
    # 输出后扫描脚本结果的差异
    def output_post_scripts(self, post_script_result_diffs):
        # 打印后扫描脚本结果的文本
        print_script_result_diffs_text("Post-scan script results",
            self.scan_a.post_script_results, self.scan_b.post_script_results,
            post_script_result_diffs, self.f)
    
    # 输出主机差异
    def output_host_diff(self, h_diff):
        # 打印到文件中
        print(file=self.f)
        # 打印主机差异的文本
        h_diff.print_text(self.f)
    
    # 输出结束
    def output_ending(self):
        # 什么也不做，只是占位
        pass
# 定义一个名为ScanDiffXML的类，继承自ScanDiff类
class ScanDiffXML(ScanDiff):
    # 初始化方法，接受scan_a、scan_b和f三个参数
    def __init__(self, scan_a, scan_b, f=sys.stdout):
        # 调用父类ScanDiff的初始化方法，传入scan_a、scan_b和f三个参数
        ScanDiff.__init__(self, scan_a, scan_b, f)

        # 获取XML文档实现对象
        impl = xml.dom.minidom.getDOMImplementation()
        # 创建一个空的XML文档对象
        self.document = impl.createDocument(None, None, None)

        # 创建XML写入器对象，传入f参数
        self.writer = XMLWriter(f)

    # 判断nmaprun是否有差异的方法
    def nmaprun_differs(self):
        # 遍历属性列表，判断scan_a和scan_b的对应属性值是否相等
        for attr in ("scanner", "version", "args", "start_date", "end_date"):
            if getattr(self.scan_a, attr, None) !=\
                    getattr(self.scan_b, attr, None):
                return True
        return False

    # 输出XML文档开始部分的方法
    def output_beginning(self):
        # 开始XML文档
        self.writer.startDocument()
        # 开始nmapdiff元素
        self.writer.startElement("nmapdiff", {"version": NDIFF_XML_VERSION})
        # 开始scandiff元素
        self.writer.startElement("scandiff", {})

        # 如果nmaprun有差异
        if self.nmaprun_differs():
            # 输出scan_a的nmaprun片段
            self.writer.frag_a(
                    self.scan_a.nmaprun_to_dom_fragment(self.document))
            # 输出scan_b的nmaprun片段
            self.writer.frag_b(
                    self.scan_b.nmaprun_to_dom_fragment(self.document))
        # 如果verbose为真
        elif verbose:
            # 输出scan_a的nmaprun片段
            self.writer.frag(
                    self.scan_a.nmaprun_to_dom_fragment(self.document))

    # 输出预脚本部分的方法
    def output_pre_scripts(self, pre_script_result_diffs):
        # 如果pre_script_result_diffs的长度大于0或者verbose为真
        if len(pre_script_result_diffs) > 0 or verbose:
            # 创建prescript元素
            prescript_elem = self.document.createElement("prescript")
            # 将预脚本结果差异转换为DOM片段
            frag = script_result_diffs_to_dom_fragment(
                prescript_elem, self.scan_a.pre_script_results,
                self.scan_b.pre_script_results, pre_script_result_diffs,
                self.document)
            # 输出DOM片段
            self.writer.frag(frag)
            # 释放DOM片段
            frag.unlink()
    # 输出后置脚本的差异结果
    def output_post_scripts(self, post_script_result_diffs):
        # 如果后置脚本的差异结果数量大于0或者verbose为True
        if len(post_script_result_diffs) > 0 or verbose:
            # 创建一个名为postscript的XML元素
            postscript_elem = self.document.createElement("postscript")
            # 将后置脚本的差异结果转换为DOM片段
            frag = script_result_diffs_to_dom_fragment(
                postscript_elem, self.scan_a.post_script_results,
                self.scan_b.post_script_results, post_script_result_diffs,
                self.document)
            # 将DOM片段写入输出
            self.writer.frag(frag)
            # 释放DOM片段的资源
            frag.unlink()

    # 输出主机差异
    def output_host_diff(self, h_diff):
        # 将主机差异转换为DOM片段
        frag = h_diff.to_dom_fragment(self.document)
        # 将DOM片段写入输出
        self.writer.frag(frag)
        # 释放DOM片段的资源
        frag.unlink()

    # 输出结束标记
    def output_ending(self):
        # 结束scandiff元素
        self.writer.endElement("scandiff")
        # 结束nmapdiff元素
        self.writer.endElement("nmapdiff")
        # 结束文档
        self.writer.endDocument()
# 定义一个名为 HostDiff 的类，用于比较两个主机的差异
class HostDiff(object):
    """A diff of two Hosts. It contains the two hosts, variables describing
    what changed, and a list of PortDiffs and OS differences."""
    # 初始化方法，接受两个主机对象作为参数
    def __init__(self, host_a, host_b):
        # 将传入的两个主机对象分别赋值给实例变量 host_a 和 host_b
        self.host_a = host_a
        self.host_b = host_b
        # 初始化一些描述变化的布尔变量
        self.state_changed = False
        self.id_changed = False
        self.os_changed = False
        self.extraports_changed = False
        # 初始化两个空列表，用于存储端口差异和操作系统差异
        self.ports = []
        self.port_diffs = {}
        self.os_diffs = []
        self.script_result_diffs = []
        # 初始化一个 cost 变量，用于存储成本
        self.cost = 0

        # 调用 diff 方法，比较两个主机的差异
        self.diff()
    # 定义一个方法用于比较两个主机对象的状态和属性差异
    def diff(self):
        # 如果两个主机的状态不同，则将状态改变标志设置为True，并增加成本计数
        if self.host_a.state != self.host_b.state:
            self.state_changed = True
            self.cost += 1

        # 如果两个主机的地址集合或主机名集合不同，则将ID改变标志设置为True，并增加成本计数
        if set(self.host_a.addresses) != set(self.host_b.addresses) \
           or set(self.host_a.hostnames) != set(self.host_b.hostnames):
            self.id_changed = True
            self.cost += 1

        # 获取所有端口规格的并集，并按规格排序
        all_specs = list(
                set(self.host_a.ports.keys()).union(
                    set(self.host_b.ports.keys())))
        all_specs.sort()
        # 遍历所有端口规格
        for spec in all_specs:
            # 创建端口差异对象，比较两个主机对应规格的端口
            port_a = self.host_a.ports.get(spec)
            port_b = self.host_b.ports.get(spec)
            diff = PortDiff(port_a or Port(spec), port_b or Port(spec))
            # 如果包含差异，则将端口添加到列表中，并更新成本计数
            if self.include_diff(diff):
                port = port_a or port_b
                self.ports.append(port)
                self.port_diffs[port] = diff
                self.cost += diff.cost

        # 使用difflib库比较两个主机的操作系统信息
        os_diffs = difflib.SequenceMatcher(
                None, self.host_a.os, self.host_b.os)
        self.os_diffs = os_diffs.get_opcodes()
        # 计算操作系统信息的差异成本，并更新总成本计数
        os_cost = len([x for x in self.os_diffs if x[0] != "equal"])
        if os_cost > 0:
            self.os_changed = True
        self.cost += os_cost

        # 比较两个主机的额外端口信息
        extraports_a = tuple((count, state)
                for (state, count) in list(self.host_a.extraports.items()))
        extraports_b = tuple((count, state)
                for (state, count) in list(self.host_b.extraports.items()))
        # 如果额外端口信息不同，则将额外端口改变标志设置为True，并增加成本计数
        if extraports_a != extraports_b:
            self.extraports_changed = True
            self.cost += 1

        # 比较两个主机的脚本执行结果信息
        self.script_result_diffs = ScriptResultDiff.diff_lists(
                self.host_a.script_results, self.host_b.script_results)
        # 更新总成本计数
        self.cost += len(self.script_result_diffs)
    # 如果两个端口的状态都是 extraports，且不是在详细模式下，则不包含差异
    if self.host_a.is_extraports(diff.port_a.state) and \
       self.host_b.is_extraports(diff.port_b.state):
        return False
    # 如果在详细模式下，则包含所有差异，即使 cost == 0
    elif verbose:
        return True
    # 否则，只包含 cost 大于 0 的差异
    return diff.cost > 0
# 定义一个名为 PortDiff 的类，用于比较两个端口的差异
class PortDiff(object):
    """A diff of two Ports. It contains the two ports and the cost of changing
    one into the other. If the cost is 0 then the two ports are the same."""
    # 初始化方法，接受两个端口对象作为参数
    def __init__(self, port_a, port_b):
        # 将传入的两个端口对象分别赋值给实例变量 port_a 和 port_b
        self.port_a = port_a
        self.port_b = port_b
        # 初始化脚本结果差异列表为空
        self.script_result_diffs = []
        # 初始化变更成本为 0
        self.cost = 0

        # 调用 diff 方法进行端口差异比较
        self.diff()

    # 定义 diff 方法，用于比较两个端口的差异
    def diff(self):
        # 如果端口 A 的规范不等于端口 B 的规范，则增加变更成本
        if self.port_a.spec != self.port_b.spec:
            self.cost += 1

        # 如果端口 A 的状态不等于端口 B 的状态，则增加变更成本
        if self.port_a.state != self.port_b.state:
            self.cost += 1

        # 如果端口 A 的服务不等于端口 B 的服务，则增加变更成本
        if self.port_a.service != self.port_b.service:
            self.cost += 1

        # 计算脚本结果的差异列表，并将差异的数量加到变更成本中
        self.script_result_diffs = ScriptResultDiff.diff_lists(
                self.port_a.script_results, self.port_b.script_results)
        self.cost += len(self.script_result_diffs)

    # PortDiffs are inserted into a Table and then printed, not printed out
    # directly. That's why this class has append_to_port_table instead of
    # print_text.
    # PortDiff 对象被插入到表中然后打印，而不是直接打印出来。这就是为什么这个类有 append_to_port_table 方法而不是 print_text 方法。
    # 将端口差异追加到包含五列的表中：
    # +- PORT STATE SERVICE VERSION
    # "+-" 代表差异指示列
    a_columns = [self.port_a.spec_string(),  # 获取端口A的规范字符串
        self.port_a.state_string(),  # 获取端口A的状态字符串
        self.port_a.service.name_string(),  # 获取端口A的服务名称字符串
        self.port_a.service.version_string()]  # 获取端口A的服务版本字符串
    b_columns = [self.port_b.spec_string(),  # 获取端口B的规范字符串
        self.port_b.state_string(),  # 获取端口B的状态字符串
        self.port_b.service.name_string(),  # 获取端口B的服务名称字符串
        self.port_b.service.version_string()]  # 获取端口B的服务版本字符串
    if a_columns == b_columns:  # 如果端口A和端口B的列相同
        if verbose or self.script_result_diffs > 0:  # 如果详细模式打开或脚本结果差异大于0
            table.append([" "] + a_columns)  # 将空格和端口A的列追加到表中
    else:  # 如果端口A和端口B的列不同
        if not host_a.is_extraports(self.port_a.state):  # 如果主机A不是额外端口
            table.append(["-"] + a_columns)  # 将减号和端口A的列追加到表中
        if not host_b.is_extraports(self.port_b.state):  # 如果主机B不是额外端口
            table.append(["+"] + b_columns)  # 将加号和端口B的列追加到表中

    for sr_diff in self.script_result_diffs:  # 遍历脚本结果差异
        sr_diff.append_to_port_table(table)  # 将脚本结果差异追加到表中
    # 将对象转换为文档片段并返回
    def to_dom_fragment(self, document):
        # 创建一个空的文档片段
        frag = document.createDocumentFragment()
        # 创建一个名为 "portdiff" 的元素节点
        portdiff_elem = document.createElement("portdiff")
        # 将 "portdiff" 元素节点添加到文档片段中
        frag.appendChild(portdiff_elem)
        # 检查端口A和端口B的规格和状态是否相同
        if (self.port_a.spec == self.port_b.spec and
                self.port_a.state == self.port_b.state):
            # 如果相同，创建一个名为 "port" 的元素节点
            port_elem = document.createElement("port")
            # 设置 "port" 元素节点的属性
            port_elem.setAttribute("portid", str(self.port_a.spec[0]))
            port_elem.setAttribute("protocol", self.port_a.spec[1])
            # 如果端口A的状态不为空，创建一个名为 "state" 的元素节点，并设置其属性
            if self.port_a.state is not None:
                state_elem = document.createElement("state")
                state_elem.setAttribute("state", self.port_a.state)
                # 将 "state" 元素节点添加到 "port" 元素节点中
                port_elem.appendChild(state_elem)
            # 如果端口A的服务与端口B的服务相同
            if self.port_a.service == self.port_b.service:
                # 将端口A的服务转换为文档片段，并添加到 "port" 元素节点中
                port_elem.appendChild(self.port_a.service.to_dom_fragment(document))
            else:
                # 如果不同，创建名为 "a" 的元素节点，并将端口A的服务转换为文档片段添加到其中
                a_elem = document.createElement("a")
                a_elem.appendChild(self.port_a.service.to_dom_fragment(document))
                # 将 "a" 元素节点添加到 "port" 元素节点中
                port_elem.appendChild(a_elem)
                # 创建名为 "b" 的元素节点，并将端口B的服务转换为文档片段添加到其中
                b_elem = document.createElement("b")
                b_elem.appendChild(self.port_b.service.to_dom_fragment(document))
                # 将 "b" 元素节点添加到 "port" 元素节点中
                port_elem.appendChild(b_elem)
            # 遍历端口脚本结果差异列表，将每个差异转换为文档片段并添加到 "port" 元素节点中
            for sr_diff in self.script_result_diffs:
                port_elem.appendChild(sr_diff.to_dom_fragment(document))
            # 将 "port" 元素节点添加到 "portdiff" 元素节点中
            portdiff_elem.appendChild(port_elem)
        else:
            # 如果端口A和端口B的规格和状态不相同，将端口A和端口B分别转换为文档片段并添加到 "portdiff" 元素节点中
            a_elem = document.createElement("a")
            a_elem.appendChild(self.port_a.to_dom_fragment(document))
            portdiff_elem.appendChild(a_elem)
            b_elem = document.createElement("b")
            b_elem.appendChild(self.port_b.to_dom_fragment(document))
            portdiff_elem.appendChild(b_elem)

        # 返回文档片段
        return frag
# 定义一个名为 ScriptResultDiff 的类
class ScriptResultDiff(object):
    # 初始化方法，接受两个参数 sr_a 和 sr_b
    def __init__(self, sr_a, sr_b):
        """One of sr_a and sr_b may be None."""
        # 将参数赋值给对象的属性
        self.sr_a = sr_a
        self.sr_b = sr_b

    # 定义一个静态方法 diff_lists，接受两个参数 a 和 b
    def diff_lists(a, b):
        """Return a list of ScriptResultDiffs from two sorted lists of
        ScriptResults."""
        # 初始化一个空列表 diffs
        diffs = []
        # 初始化变量 i 和 j
        i = 0
        j = 0
        # 这个算法类似于合并排序后的列表
        while i < len(a) and j < len(b):
            # 如果 a[i] 的 id 小于 b[j] 的 id
            if a[i].id < b[j].id:
                # 将 ScriptResultDiff(a[i], None) 添加到 diffs 列表中
                diffs.append(ScriptResultDiff(a[i], None))
                i += 1
            # 如果 a[i] 的 id 大于 b[j] 的 id
            elif a[i].id > b[j].id:
                # 将 ScriptResultDiff(None, b[j]) 添加到 diffs 列表中
                diffs.append(ScriptResultDiff(None, b[j]))
                j += 1
            # 如果 a[i] 的 id 等于 b[j] 的 id
            else:
                # 如果 a[i] 的 output 不等于 b[j] 的 output 或者 verbose 为真
                if a[i].output != b[j].output or verbose:
                    # 将 ScriptResultDiff(a[i], b[j]) 添加到 diffs 列表中
                    diffs.append(ScriptResultDiff(a[i], b[j]))
                i += 1
                j += 1
        # 将剩余的 a 中的元素添加到 diffs 列表中
        while i < len(a):
            diffs.append(ScriptResultDiff(a[i], None))
            i += 1
        # 将剩余的 b 中的元素添加到 diffs 列表中
        while j < len(b):
            diffs.append(ScriptResultDiff(None, b[j]))
            j += 1
        # 返回 diffs 列表
        return diffs
    # 将 diff_lists 方法设置为静态方法
    diff_lists = staticmethod(diff_lists)

    # Script result diffs are appended to a port table rather than being
    # printed directly, so append_to_port_table exists instead of print_text.
    # 将表格追加到端口表中
    def append_to_port_table(self, table):
        # 初始化两个空列表用于存储行数据
        a_lines = []
        b_lines = []
        # 如果 self.sr_a 不为空，则获取其行数据
        if self.sr_a is not None:
            a_lines = self.sr_a.get_lines()
        # 如果 self.sr_b 不为空，则获取其行数据
        if self.sr_b is not None:
            b_lines = self.sr_b.get_lines()
        # 如果 a_lines 不等于 b_lines 或者 verbose 为真
        if a_lines != b_lines or verbose:
            # 使用 difflib 库创建 SequenceMatcher 对象
            diffs = difflib.SequenceMatcher(None, a_lines, b_lines)
            # 遍历差异操作
            for op, i1, i2, j1, j2 in diffs.get_opcodes():
                # 如果操作为 "replace" 或 "delete"
                if op == "replace" or op == "delete":
                    # 将 a_lines 中的行数据添加到表格中，前面加上 "-"
                    for k in range(i1, i2):
                        table.append_raw("-" + a_lines[k])
                # 如果操作为 "replace" 或 "insert"
                if op == "replace" or op == "insert":
                    # 将 b_lines 中的行数据添加到表格中，前面加上 "+"
                    for k in range(j1, j2):
                        table.append_raw("+" + b_lines[k])
                # 如果操作为 "equal"
                if op == "equal":
                    # 将 a_lines 中的行数据添加到表格中，前面加上 " "
                    for k in range(i1, i2):
                        table.append_raw(" " + a_lines[k])

    # 将对象转换为文档片段
    def to_dom_fragment(self, document):
        # 创建一个空的文档片段
        frag = document.createDocumentFragment()
        # 如果 self.sr_a 和 self.sr_b 都不为空且相等
        if (self.sr_a is not None and
                self.sr_b is not None and
                self.sr_a == self.sr_b):
            # 将 self.sr_a 转换为文档片段并添加到 frag 中
            frag.appendChild(self.sr_a.to_dom_fragment(document))
        else:
            # 如果 self.sr_a 不为空
            if self.sr_a is not None:
                # 创建 <a> 元素，将 self.sr_a 转换为文档片段并添加到 <a> 元素中，然后将 <a> 元素添加到 frag 中
                a_elem = document.createElement("a")
                a_elem.appendChild(self.sr_a.to_dom_fragment(document))
                frag.appendChild(a_elem)
            # 如果 self.sr_b 不为空
            if self.sr_b is not None:
                # 创建 <b> 元素，将 self.sr_b 转换为文档片段并添加到 <b> 元素中，然后将 <b> 元素添加到 frag 中
                b_elem = document.createElement("b")
                b_elem.appendChild(self.sr_b.to_dom_fragment(document))
                frag.appendChild(b_elem)
        # 返回文档片段
        return frag
# 定义一个表格类，用于存储字符数据，类似于 NmapOutputTable
class Table(object):
    """A table of character data, like NmapOutputTable."""
    # 初始化方法，接受一个模板字符串作为参数
    def __init__(self, template):
        """template is a string consisting of "*" and other characters. Each
        "*" is a left-justified space-padded field. All other characters are
        copied to the output."""
        # 存储每列的宽度
        self.widths = []
        # 存储表格的行
        self.rows = []
        # 存储模板中第一个"*"之前的字符串
        self.prefix = ""
        # 存储模板中"*"之后的字符串，作为填充
        self.padding = []
        j = 0
        # 找到模板中第一个"*"之前的字符串
        while j < len(template) and template[j] != "*":
            j += 1
        self.prefix = template[:j]
        j += 1
        i = j
        # 处理模板中的"*"和其他字符
        while j < len(template):
            while j < len(template) and template[j] != "*":
                j += 1
            self.padding.append(template[i:j])
            j += 1
            i = j

    # 添加一行数据
    def append(self, row):
        strings = []
        # 将行转换为列表
        row = list(row)
        # 移除末尾的None值
        while len(row) > 0 and row[-1] is None:
            row.pop()
        # 格式化每个元素，并计算每列的宽度
        for i in range(len(row)):
            if row[i] is None:
                s = ""
            else:
                s = str(row[i])
            if i == len(self.widths):
                self.widths.append(len(s))
            elif len(s) > self.widths[i]:
                self.widths[i] = len(s)
            strings.append(s)
        self.rows.append(strings)

    # 添加一个未格式化为列的原始字符串
    def append_raw(self, s):
        """Append a raw string for a row that is not formatted into columns."""
        self.rows.append(s)

    # 返回表格的行数
    def __len__(self):
        return len(self.rows)
    # 定义对象的字符串表示形式
    def __str__(self):
        # 初始化行列表
        lines = []
        # 遍历对象的行
        for row in self.rows:
            # 初始化部分列表，包含前缀
            parts = [self.prefix]
            # 初始化索引
            i = 0
            # 如果行是字符串类型
            if isinstance(row, str):
                # 将该行添加到行列表中
                lines.append(row)
            else:
                # 如果行是列表类型
                while i < len(row):
                    # 将每个元素按照指定宽度添加到部分列表中
                    parts.append(row[i].ljust(self.widths[i]))
                    # 如果索引小于填充列表的长度，添加填充元素到部分列表中
                    if i < len(self.padding):
                        parts.append(self.padding[i])
                    # 索引自增
                    i += 1
                # 将部分列表拼接成字符串，并去除右侧空格后添加到行列表中
                lines.append("".join(parts).rstrip())
        # 将行列表中的内容用换行符连接成一个字符串返回
        return "\n".join(lines)
def warn(str):
    """Print a warning to stderr."""
    # 打印警告信息到标准错误输出
    print(str, file=sys.stderr)


class NmapContentHandler(xml.sax.handler.ContentHandler):
    """The xml.sax ContentHandler for the XML parser. It contains a Scan object
    that is filled in and can be read back again once the parse method is
    finished."""
    # 初始化方法，设置初始属性和状态
    def __init__(self, scan):
        xml.sax.handler.ContentHandler.__init__(self)
        # 设置扫描对象
        self.scan = scan

        # We keep a stack of the elements we've seen, pushing on start and
        # popping on end.
        # 保存已经遇到的元素的堆栈，开始时入栈，结束时出栈
        self.element_stack = []

        # 设置当前主机和端口为 None
        self.current_host = None
        self.current_port = None
        self.skip_over = False

        # 设置开始元素的处理函数
        self._start_elem_handlers = {
            "nmaprun": self._start_nmaprun,
            "host": self._start_host,
            "hosthint": self._start_hosthint,
            "status": self._start_status,
            "address": self._start_address,
            "hostname": self._start_hostname,
            "extraports": self._start_extraports,
            "port": self._start_port,
            "state": self._start_state,
            "service": self._start_service,
            "script": self._start_script,
            "osmatch": self._start_osmatch,
            "finished": self._start_finished,
        }
        # 设置结束元素的处理函数
        self._end_elem_handlers = {
            'host': self._end_host,
            'hosthint': self._end_hosthint,
            'port': self._end_port,
        }

    # 返回当前元素的父元素名称，如果是根元素则返回 None
    def parent_element(self):
        """Return the name of the element containing the current one, or None
        if this is the root element."""
        if len(self.element_stack) == 0:
            return None
        return self.element_stack[-1]
    # 定义一个方法，用于处理 XML 元素的开始标签，同时记录元素的堆栈
    def startElement(self, name, attrs):
        """This method keeps track of element_stack. The real parsing work is
        done in the _start_*() handlers. This is to make it easy for them to
        bail out on error."""
        # 获取处理该元素的处理器
        handler = self._start_elem_handlers.get(name)
        # 如果处理器存在且不需要跳过，则调用处理器处理该元素
        if handler is not None and not self.skip_over:
            handler(name, attrs)
        # 将元素名添加到元素堆栈中
        self.element_stack.append(name)
    
    # 定义一个方法，用于处理 XML 元素的结束标签，同时记录元素的堆栈
    def endElement(self, name):
        """This method keeps track of element_stack. The real parsing work is
        done in _end_*() handlers."""
        # 从元素堆栈中弹出当前元素名
        self.element_stack.pop()
        # 获取处理该元素的处理器
        handler = self._end_elem_handlers.get(name)
        # 如果处理器存在，则调用处理器处理该元素
        if handler is not None:
            handler(name)
    
    # 定义一个方法，用于处理 nmaprun 元素的开始标签
    def _start_nmaprun(self, name, attrs):
        # 确保当前元素没有父元素
        assert self.parent_element() is None
        # 如果属性中包含 "start"，则解析并设置扫描开始时间
        if "start" in attrs:
            start_timestamp = int(attrs.get("start"))
            self.scan.start_date = datetime.datetime.fromtimestamp(start_timestamp)
        # 设置扫描器、参数和版本信息
        self.scan.scanner = attrs.get("scanner")
        self.scan.args = attrs.get("args")
        self.scan.version = attrs.get("version")
    
    # 定义一个方法，用于处理 host 元素的开始标签
    def _start_host(self, name, attrs):
        # 确保当前元素的父元素是 nmaprun
        assert self.parent_element() == "nmaprun"
        # 创建一个新的 Host 对象，并将其添加到扫描对象的 hosts 列表中
        self.current_host = Host()
        self.scan.hosts.append(self.current_host)
    
    # 定义一个方法，用于处理 hosthint 元素的开始标签
    def _start_hosthint(self, name, attrs):
        # 确保当前元素的父元素是 nmaprun，并设置跳过标志为 True
        assert self.parent_element() == "nmaprun"
        self.skip_over = True
    
    # 定义一个方法，用于处理 status 元素的开始标签
    def _start_status(self, name, attrs):
        # 确保当前元素的父元素是 host，并且当前主机对象不为空
        assert self.parent_element() == "host"
        assert self.current_host is not None
        # 获取状态属性，并设置当前主机的状态
        state = attrs.get("state")
        if state is None:
            warn('%s element of host %s is missing the "state" attribute; '
                    'assuming "unknown".' % (
                        name, self.current_host.format_name()))
            return
        self.current_host.state = state
    # 定义一个方法，用于处理起始地址元素，需要确保当前父元素为"host"，当前主机对象不为空
    def _start_address(self, name, attrs):
        assert self.parent_element() == "host"
        assert self.current_host is not None
        # 获取地址属性
        addr = attrs.get("addr")
        # 如果地址为空，则发出警告并跳过
        if addr is None:
            warn('%s element of host %s is missing the "addr" '
                    'attribute; skipping.' % (
                        name, self.current_host.format_name()))
            return
        # 获取地址类型属性，默认为ipv4
        addrtype = attrs.get("addrtype", "ipv4")
        # 将地址添加到当前主机对象中
        self.current_host.add_address(Address.new(addrtype, addr))
    
    # 定义一个方法，用于处理起始主机名元素，需要确保当前父元素为"hostnames"，当前主机对象不为空
    def _start_hostname(self, name, attrs):
        assert self.parent_element() == "hostnames"
        assert self.current_host is not None
        # 获取主机名属性
        hostname = attrs.get("name")
        # 如果主机名为空，则发出警告并跳过
        if hostname is None:
            warn('%s element of host %s is missing the "name" '
                    'attribute; skipping.' % (
                        name, self.current_host.format_name()))
            return
        # 将主机名添加到当前主机对象中
        self.current_host.add_hostname(hostname)
    # 开始处理额外端口信息，接受参数为名称和属性
    def _start_extraports(self, name, attrs):
        # 断言当前父元素为 "ports"
        assert self.parent_element() == "ports"
        # 断言当前主机不为空
        assert self.current_host is not None
        # 获取状态属性
        state = attrs.get("state")
        # 如果状态属性为空，发出警告并假设状态为 "unknown"
        if state is None:
            warn('%s element of host %s is missing the "state" '
                    'attribute; assuming "unknown".' % (
                        name, self.current_host.format_name()))
            state = "unknown"
        # 如果状态在当前主机的额外端口中已存在，发出警告
        if state in self.current_host.extraports:
            warn('Duplicate extraports state "%s" in host %s.' % (
                state, self.current_host.format_name()))

        # 获取端口数量属性
        count = attrs.get("count")
        # 如果端口数量属性为空，发出警告并假设数量为 0
        if count is None:
            warn('%s element of host %s is missing the "count" '
                    'attribute; assuming 0.' % (
                        name, self.current_host.format_name()))
            count = 0
        else:
            # 尝试将端口数量转换为整数，如果失败则发出警告并假设数量为 0
            try:
                count = int(count)
            except ValueError:
                warn("Can't convert extraports count \"%s\" "
                        "to an integer in host %s; assuming 0." % (
                            attrs["count"], self.current_host.format_name()))
                count = 0
        # 将状态和数量添加到当前主机的额外端口字典中
        self.current_host.extraports[state] = count
    # 启动端口解析，检查当前元素是否在"ports"下，当前主机是否存在
    def _start_port(self, name, attrs):
        assert self.parent_element() == "ports"
        assert self.current_host is not None
        # 获取端口ID字符串
        portid_str = attrs.get("portid")
        # 如果端口ID字符串不存在，发出警告并跳过
        if portid_str is None:
            warn('%s element of host %s missing the "portid" '
                    'attribute; skipping.' % (
                        name, self.current_host.format_name()))
            return
        # 尝试将端口ID字符串转换为整数，如果失败则发出警告并跳过
        try:
            portid = int(portid_str)
        except ValueError:
            warn("Can't convert portid \"%s\" to an integer "
                    "in host %s; skipping port." % (
                        portid_str, self.current_host.format_name()))
            return
        # 获取协议类型
        protocol = attrs.get("protocol")
        # 如果协议类型不存在，发出警告并跳过
        if protocol is None:
            warn('%s element of host %s missing the "protocol" '
                    'attribute; skipping.' % (
                        name, self.current_host.format_name()))
            return
        # 设置当前端口为解析得到的端口ID和协议类型
        self.current_port = Port((portid, protocol))

    # 启动状态解析，检查当前元素是否在"port"下，当前主机和端口是否存在
    def _start_state(self, name, attrs):
        assert self.parent_element() == "port"
        assert self.current_host is not None
        # 如果当前端口不存在，则跳过
        if self.current_port is None:
            return
        # 如果状态属性不存在，假定为"unknown"状态并发出警告
        if "state" not in attrs:
            warn('%s element of port %s is missing the "state" '
                    'attribute; assuming "unknown".' % (
                        name, self.current_port.spec_string()))
            return
        # 设置当前端口的状态属性，并将端口添加到当前主机的端口列表中
        self.current_port.state = attrs["state"]
        self.current_host.add_port(self.current_port)
    # 启动服务的私有方法，设置当前端口的服务信息
    def _start_service(self, name, attrs):
        # 确保当前父元素是"port"
        assert self.parent_element() == "port"
        # 确保当前主机不为空
        assert self.current_host is not None
        # 如果当前端口为空，则返回
        if self.current_port is None:
            return
        # 设置当前端口的服务名称、产品、版本、额外信息和隧道信息
        self.current_port.service.name = attrs.get("name")
        self.current_port.service.product = attrs.get("product")
        self.current_port.service.version = attrs.get("version")
        self.current_port.service.extrainfo = attrs.get("extrainfo")
        self.current_port.service.tunnel = attrs.get("tunnel")
    
    # 启动脚本的私有方法，设置脚本执行结果
    def _start_script(self, name, attrs):
        # 创建脚本执行结果对象
        result = ScriptResult()
        # 设置脚本执行结果的ID
        result.id = attrs.get("id")
        # 如果ID为空，则发出警告并返回
        if result.id is None:
            warn('%s element missing the "id" attribute; skipping.' % name)
            return
        # 设置脚本执行结果的输出
        result.output = attrs.get("output")
        # 如果输出为空，则发出警告并返回
        if result.output is None:
            warn('%s element missing the "output" attribute; skipping.' % name)
            return
        # 根据父元素类型将脚本执行结果添加到相应的列表中
        if self.parent_element() == "prescript":
            self.scan.pre_script_results.append(result)
        elif self.parent_element() == "postscript":
            self.scan.post_script_results.append(result)
        elif self.parent_element() == "hostscript":
            self.current_host.script_results.append(result)
        elif self.parent_element() == "port":
            self.current_port.script_results.append(result)
        else:
            warn("%s element not inside prescript, postscript, hostscript, or port element; ignoring." % name)
            return
    
    # 启动操作系统匹配的私有方法，将操作系统信息添加到当前主机的操作系统列表中
    def _start_osmatch(self, name, attrs):
        # 确保当前父元素是"os"
        assert self.parent_element() == "os"
        # 确保当前主机不为空
        assert self.current_host is not None
        # 如果属性中缺少"name"，则发出警告并返回
        if "name" not in attrs:
            warn('%s element of host %s is missing the "name" attribute; skipping.' % (name, self.current_host.format_name()))
            return
        # 将操作系统信息添加到当前主机的操作系统列表中
        self.current_host.os.append(attrs["name"])
    # 定义一个方法，用于处理标签开始和结束时的操作，参数包括标签名和属性
    def _start_finished(self, name, attrs):
        # 断言当前父元素是"runstats"
        assert self.parent_element() == "runstats"
        # 如果属性中包含"time"，则获取时间戳并转换为日期时间格式，赋值给scan.end_date
        if "time" in attrs:
            end_timestamp = int(attrs.get("time"))
            self.scan.end_date = datetime.datetime.fromtimestamp(end_timestamp)
    
    # 定义一个方法，用于处理结束标签为"host"时的操作，不接受参数
    def _end_host(self, name):
        # 对当前主机的脚本结果进行排序
        self.current_host.script_results.sort()
        # 将当前主机置为None
        self.current_host = None
    
    # 定义一个方法，用于处理结束标签为"hosthint"时的操作，不接受参数
    def _end_hosthint(self, name):
        # 将skip_over置为False
        self.skip_over = False
    
    # 定义一个方法，用于处理结束标签为"port"时的操作，不接受参数
    def _end_port(self, name):
        # 对当前端口的脚本结果进行排序
        self.current_port.script_results.sort()
        # 将当前端口置为None
        self.current_port = None
# 定义一个 XMLWriter 类，继承自 xml.sax.saxutils.XMLGenerator
class XMLWriter (xml.sax.saxutils.XMLGenerator):
    # 初始化方法，接受一个文件对象作为参数
    def __init__(self, f):
        # 调用父类的初始化方法，设置编码为 utf-8
        xml.sax.saxutils.XMLGenerator.__init__(self, f, "utf-8")
        # 保存文件对象到实例属性中
        self.f = f

    # 定义一个方法用于处理 XML 片段
    def frag(self, frag):
        # 遍历片段中的节点，将节点的 XML 写入到文件对象中
        for node in frag.childNodes:
            node.writexml(self.f, newl="\n")

    # 定义一个方法用于处理 XML 片段，包裹在 <a> 标签中
    def frag_a(self, frag):
        # 开始 <a> 标签
        self.startElement("a", {})
        # 遍历片段中的节点，将节点的 XML 写入到文件对象中
        for node in frag.childNodes:
            node.writexml(self.f, newl="\n")
        # 结束 <a> 标签
        self.endElement("a")

    # 定义一个方法用于处理 XML 片段，包裹在 <b> 标签中
    def frag_b(self, frag):
        # 开始 <b> 标签
        self.startElement("b", {})
        # 遍历片段中的节点，将节点的 XML 写入到文件对象中
        for node in frag.childNodes:
            node.writexml(self.f, newl="\n")
        # 结束 <b> 标签
        self.endElement("b")

# 定义一个 usage 函数，用于显示程序的使用方法
def usage():
    print("""\
Usage: %s [option] FILE1 FILE2
Compare two Nmap XML files and display a list of their differences.
Differences include host state changes, port state changes, and changes to
service and OS detection.

  -h, --help     display this help
  -v, --verbose  also show hosts and ports that haven't changed.
  --text         display output in text format (default)
  --xml          display output in XML format\
""" % sys.argv[0])

# 定义三个退出状态常量
EXIT_EQUAL = 0
EXIT_DIFFERENT = 1
EXIT_ERROR = 2

# 定义一个 usage_error 函数，用于显示使用错误信息并退出程序
def usage_error(msg):
    print("%s: %s" % (sys.argv[0], msg), file=sys.stderr)
    print("Try '%s -h' for help." % sys.argv[0], file=sys.stderr)
    sys.exit(EXIT_ERROR)

# 定义一个 main 函数，用于程序的主要逻辑
def main():
    global verbose
    output_format = None

    try:
        # 解析命令行参数，获取选项和文件名
        opts, input_filenames = getopt.gnu_getopt(
                sys.argv[1:], "hv", ["help", "text", "verbose", "xml"])
    except getopt.GetoptError as e:
        # 如果解析参数出错，调用 usage_error 函数显示错误信息并退出程序
        usage_error(e.msg)
    # 遍历命令行参数中的选项和参数
    for o, a in opts:
        # 如果选项是 -h 或 --help，则调用 usage() 函数并退出程序
        if o == "-h" or o == "--help":
            usage()
            sys.exit(0)
        # 如果选项是 -v 或 --verbose，则设置 verbose 为 True
        elif o == "-v" or o == "--verbose":
            verbose = True
        # 如果选项是 --text，则检查输出格式是否已经设置为非文本格式，如果是则报错，否则设置输出格式为文本
        elif o == "--text":
            if output_format is not None and output_format != "text":
                usage_error("contradictory output format options.")
            output_format = "text"
        # 如果选项是 --xml，则检查输出格式是否已经设置为非 XML 格式，如果是则报错，否则设置输出格式为 XML
        elif o == "--xml":
            if output_format is not None and output_format != "xml":
                usage_error("contradictory output format options.")
            output_format = "xml"

    # 检查输入文件名的数量是否为 2，如果不是则报错
    if len(input_filenames) != 2:
        usage_error("need exactly two input filenames.")

    # 如果输出格式未设置，则默认设置为文本格式
    if output_format is None:
        output_format = "text"

    # 获取输入文件名的第一个和第二个文件名
    filename_a = input_filenames[0]
    filename_b = input_filenames[1]

    # 尝试加载文件并创建 Scan 对象，如果失败则打印错误信息并退出程序
    try:
        scan_a = Scan()
        scan_a.load_from_file(filename_a)
        scan_b = Scan()
        scan_b.load_from_file(filename_b)
    except IOError as e:
        print("Can't open file: %s" % str(e), file=sys.stderr)
        sys.exit(EXIT_ERROR)

    # 根据输出格式创建不同的 ScanDiff 对象
    if output_format == "text":
        diff = ScanDiffText(scan_a, scan_b)
    elif output_format == "xml":
        diff = ScanDiffXML(scan_a, scan_b)
    
    # 调用 diff 对象的 output() 方法，获取比较结果
    cost = diff.output()

    # 如果比较结果为 0，则返回相等的退出码
    if cost == 0:
        return EXIT_EQUAL
    # 否则返回不相等的退出码
    else:
        return EXIT_DIFFERENT
# 捕获未捕获的异常，以便它们可以产生退出代码2（EXIT_ERROR），而不是默认情况下的1。
def excepthook(type, value, tb):
    # 调用默认的异常处理函数，打印异常信息
    sys.__excepthook__(type, value, tb)
    # 退出程序，返回错误退出码
    sys.exit(EXIT_ERROR)

# 如果作为主程序执行
if __name__ == "__main__":
    # 设置异常处理钩子
    sys.excepthook = excepthook
    # 调用主函数，并返回其退出码
    sys.exit(main())
```