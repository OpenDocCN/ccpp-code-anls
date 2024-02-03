# `nmap\zenmap\zenmapCore\NetworkInventory.py`

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
import os
import unittest
import zenmapCore
import zenmapCore.NmapParser
from zenmapGUI.SearchGUI import SearchParser
from .SearchResult import HostSearch

# 定义 NetworkInventory 类，用于存储聚合扫描结果，并负责将聚合结果持久化到存储中
class NetworkInventory(object):
    """This class acts as a container for aggregated scans. It is also
    responsible for opening/saving the aggregation from/to persistent
    storage."""
    def __init__(self, filename=None):
        # 存储所有构成该清单的扫描结果的列表
        self.scans = []

        # 将解析后的扫描结果与它们所加载的文件名进行映射的字典
        self.filenames = {}

        # 将 IP 地址映射到 HostInfo 对象的字典
        self.hosts = {}

        # 如果提供了文件名，则从文件中加载数据
        if filename is not None:
            self.open_from_file(filename)
    def remove_scan(self, scan):
        """Removes a scan and any host information it contained from the
        inventory."""
        # 从清单中移除指定的扫描及其包含的主机信息
        # 注意：如果传入的扫描不在清单中，这个方法会抛出 ValueError 异常并且不会完成
        # 从扫描列表中移除指定的扫描
        self.scans.remove(scan)

        # 清空主机字典
        self.hosts = {}

        # 记住扫描列表
        scans = self.scans

        # 清空扫描列表
        self.scans = []

        # 删除文件名条目（如果存在）
        if scan in self.filenames:
            del self.filenames[scan]

        # 对于记住的扫描列表中的每个扫描，将其添加到扫描列表中并相应地更新主机列表
        for scan in scans:
            self.add_scan(scan)

    def get_scans(self):
        return self.scans

    def get_hosts(self):
        return list(self.hosts.values())

    def get_hosts_up(self):
        return [h for h in list(self.hosts.values()) if h.get_state() == 'up']

    def get_hosts_down(self):
        return [h for h in list(self.hosts.values()) if h.get_state() == 'down']

    def open_from_file(self, path):
        """Loads a scan from the given file."""
        from zenmapCore.NmapParser import NmapParser

        parsed = NmapParser()
        parsed.parse_file(path)
        self.add_scan(parsed, path)

    def open_from_dir(self, path):
        """Loads all scans from the given directory into the network
        inventory."""
        from zenmapCore.NmapParser import NmapParser

        for filename in os.listdir(path):
            fullpath = os.path.join(path, filename)
            if os.path.isdir(fullpath):
                continue
            parsed = NmapParser()
            parsed.parse_file(fullpath)
            self.add_scan(parsed, filename=fullpath)
    # 将给定索引的扫描结果保存到指定路径的文件中，可以选择保存为 XML 格式或者普通文本格式
    def save_to_file(self, path, index, format="xml"):
        # 以写入模式打开文件
        f = open(path, 'w')
        # 如果选择保存为 XML 格式，则调用扫描结果对象的 write_xml 方法写入文件
        if format == "xml":
            self.get_scans()[index].write_xml(f)
            # 将文件名和文件对象的映射保存到字典中
            self.filenames[self.get_scans()[index]] = f
        # 如果选择保存为普通文本格式，则调用扫描结果对象的 write_text 方法写入文件
        else:
            self.get_scans()[index].write_text(f)
        # 关闭文件
        f.close()

    # 将所有扫描结果保存到指定目录，并返回保存的文件名列表
    def save_to_dir(self, path):
        # 生成文件名
        self._generate_filenames(path)

        # 遍历扫描结果和文件名的映射
        for scan, filename in self.filenames.items():
            # 以写入模式打开文件
            f = open(os.path.join(path, filename), "w")
            # 将扫描结果以 XML 格式写入文件
            scan.write_xml(f)
            # 关闭文件
            f.close()

        # 返回保存的文件名列表
        return self.filenames.values()

    # 从数据库中打开扫描结果
    def open_from_db(self, id):
        pass

    # 将扫描结果保存到数据库中
    def save_to_db(self):
        # 目前，这将分别将组成清单的每个扫描结果保存到数据库中
        from time import time
        from io import StringIO
        from zenmapCore.UmitDB import Scans

        # 遍历所有解析后的扫描结果
        for parsed in self.get_scans():
            # 创建一个字符串缓冲区
            f = StringIO()
            # 将扫描结果以 XML 格式写入字符串缓冲区
            parsed.write_xml(f)

            # 创建一个 Scans 对象，并将扫描名称、Nmap XML 输出和日期保存到数据库中
            scan = Scans(scan_name=parsed.scan_name,
                         nmap_xml_output=f.getvalue(),
                         date=time())
# 创建一个名为 FilteredNetworkInventory 的类，继承自 NetworkInventory 类
class FilteredNetworkInventory(NetworkInventory):
    # 初始化方法，接受一个文件名参数，默认为 None
    def __init__(self, filename=None):
        # 调用父类 NetworkInventory 的初始化方法，传入文件名参数
        NetworkInventory.__init__(self, filename)

        # 一个字典，列出主机过滤条件
        self.search_dict = {}
        # 一个空列表，用于存储过滤后的主机
        self.filtered_hosts = []
        # 一个包含搜索关键词的字典
        search_keywords = dict()
        search_keywords["target"] = "target"
        search_keywords["t"] = "target"
        search_keywords["inroute"] = "in_route"
        search_keywords["ir"] = "in_route"
        search_keywords["hostname"] = "hostname"
        search_keywords["service"] = "service"
        search_keywords["s"] = "service"
        search_keywords["os"] = "os"
        search_keywords["open"] = "open"
        search_keywords["op"] = "open"
        search_keywords["closed"] = "closed"
        search_keywords["cp"] = "closed"
        search_keywords["filtered"] = "filtered"
        search_keywords["fp"] = "filtered"
        search_keywords["unfiltered"] = "unfiltered"
        search_keywords["ufp"] = "unfiltered"
        search_keywords["open|filtered"] = "open_filtered"
        search_keywords["ofp"] = "open_filtered"
        search_keywords["closed|filtered"] = "closed_filtered"
        search_keywords["cfp"] = "closed_filtered"
        # 创建一个 SearchParser 对象，传入当前对象和搜索关键词字典
        self.search_parser = SearchParser(self, search_keywords)

    # FIXME: 这个方法什么也不做。我们只需要支持 SearchParser 期望的接口类型，以便使用它。
    # 或许，我们最终会对 SearchParser 进行一些重构？
    # 初始化搜索目录的方法，接受一个参数 junk
    def init_search_dirs(self, junk):
        # 什么也不做，只是占位
        pass

    # 获取主机的方法
    def get_hosts(self):
        # 如果搜索条件字典的长度大于 0，则返回过滤后的主机列表
        if len(self.search_dict) > 0:
            return self.filtered_hosts
        # 否则调用父类 NetworkInventory 的 get_hosts 方法
        else:
            return NetworkInventory.get_hosts(self)

    # 获取处于运行状态的主机的方法
    def get_hosts_up(self):
        # 如果搜索条件字典的长度大于 0，则返回过滤后的主机列表中处于运行状态的主机
        if len(self.search_dict) > 0:
            return [h for h in self.filtered_hosts if h.get_state() == 'up']
        # 否则调用父类 NetworkInventory 的 get_hosts_up 方法
        else:
            return NetworkInventory.get_hosts_up(self)
    # 如果搜索字典不为空，则返回筛选后的主机列表中状态为'down'的主机
    def get_hosts_down(self):
        if len(self.search_dict) > 0:
            return [h for h in self.filtered_hosts if h.get_state() == 'down']
        else:
            return NetworkInventory.get_hosts_down(self)

    # 返回主机总数
    def get_total_host_count(self):
        return len(self.hosts)

    # 一个辅助函数，调用给定操作符和每个参数的匹配函数
    def _match_all_args(self, host, operator, args):
        """A helper function that calls the matching function for the given
        operator and each of its arguments."""
        for arg in args:
            positive = True
            if arg != "" and arg[0] == "!":
                arg = arg[1:]
                positive = False
            if positive != self.__getattribute__(
                    "match_%s" % operator)(host, arg):
                # No match for this operator
                return False
        else:
            # if the operator is not supported, pretend its true
            # All arguments for this operator produced a match
            return True

    # 返回主机总数
    def get_host_count(self):
        return len(self.network_inventory.hosts)

    # 匹配关键字
    def match_keyword(self, host, keyword):
        return (self.match_os(host, keyword) or
                self.match_target(host, keyword) or
                self.match_service(host, keyword))

    # 匹配目标
    def match_target(self, host, name):
        return HostSearch.match_target(host, name)

    # 匹配路由中的跳数
    def match_in_route(self, host, hop):
        hops = host.get_trace().get('hops', [])
        return hop in hops

    # 匹配主机名
    def match_hostname(self, host, hostname):
        return HostSearch.match_hostname(host, hostname)

    # 匹配服务
    def match_service(self, host, service):
        return HostSearch.match_service(host, service)

    # 匹配操作系统
    def match_os(self, host, os):
        return HostSearch.match_os(host, os)

    # 匹配开放端口
    def match_open(self, host, portno):
        host_ports = host.get_ports()
        return HostSearch.match_port(host_ports, portno, "open")
    # 匹配指定主机和端口号的端口状态是否为 closed
    def match_closed(self, host, portno):
        # 获取主机的端口列表
        host_ports = host.get_ports()
        # 调用 HostSearch 类的 match_port 方法，匹配端口状态是否为 closed
        return HostSearch.match_port(host_ports, portno, "closed")

    # 匹配指定主机和端口号的端口状态是否为 filtered
    def match_filtered(self, host, portno):
        # 获取主机的端口列表
        host_ports = host.get_ports()
        # 调用 HostSearch 类的 match_port 方法，匹配端口状态是否为 filtered
        return HostSearch.match_port(host_ports, portno, "filtered")

    # 匹配指定主机和端口号的端口状态是否为 unfiltered
    def match_unfiltered(self, host, portno):
        # 获取主机的端口列表
        host_ports = host.get_ports()
        # 调用 HostSearch 类的 match_port 方法，匹配端口状态是否为 unfiltered
        return HostSearch.match_port(host_ports, portno, "unfiltered")

    # 匹配指定主机和端口号的端口状态是否为 open|filtered
    def match_open_filtered(self, host, portno):
        # 获取主机的端口列表
        host_ports = host.get_ports()
        # 调用 HostSearch 类的 match_port 方法，匹配端口状态是否为 open|filtered
        return HostSearch.match_port(host_ports, portno, "open|filtered")

    # 匹配指定主机和端口号的端口状态是否为 closed|filtered
    def match_closed_filtered(self, host, portno):
        # 获取主机的端口列表
        host_ports = host.get_ports()
        # 调用 HostSearch 类的 match_port 方法，匹配端口状态是否为 closed|filtered
        return HostSearch.match_port(host_ports, portno, "closed|filtered")

    # 应用过滤器，根据给定的过滤文本对主机进行过滤
    def apply_filter(self, filter_text):
        # 将过滤文本转换为小写
        self.filter_text = filter_text.lower()
        # 更新搜索解析器
        self.search_parser.update(self.filter_text)
        # 初始化过滤后的主机列表
        self.filtered_hosts = []
        # 遍历所有主机
        for hostname, host in self.hosts.items():
            # 对于每个主机
            # 测试每个给定的操作符是否与当前主机匹配
            for operator, args in self.search_dict.items():
                if not self._match_all_args(host, operator, args):
                    # 没有匹配 => 我们丢弃这个扫描结果
                    break
            else:
                # 所有操作符匹配函数都返回 True，因此该主机满足所有条件
                self.filtered_hosts.append(host)
# 定义一个测试类，用于测试网络清单
class NetworkInventoryTest(unittest.TestCase):
    # 定义一个测试方法，测试主机信息对象在聚合过程中是否被修改
    def test_no_external_modification(self):
        """Test that HostInfo objects passed into the inventory are not
        modified during aggregation."""
        # 创建一个 NmapParserBasics 对象
        scan_1 = zenmapCore.NmapParser.ParserBasics()
        # 创建一个 HostInfo 对象，并设置主机名和状态
        host_a = zenmapCore.NmapParser.HostInfo()
        host_a.hostnames = ["a"]
        host_a.set_state('up')
        # 设置扫描开始时间和扫描主机信息
        scan_1.start = "1000000000"
        scan_1.nmap["hosts"] = [host_a]

        # 创建另一个 NmapParserBasics 对象
        scan_2 = zenmapCore.NmapParser.ParserBasics()
        # 创建另一个 HostInfo 对象，并设置主机名和状态
        host_b = zenmapCore.NmapParser.HostInfo()
        host_b.hostnames = ["b"]
        host_b.set_state('up')
        # 设置扫描开始时间和扫描主机信息
        scan_2.start = "1000000001"
        scan_2.nmap["hosts"] = [host_b]

        # 创建一个网络清单对象
        inv = NetworkInventory()
        # 将两次扫描结果添加到网络清单中
        inv.add_scan(scan_1)
        inv.add_scan(scan_2)

        # 断言主机信息对象的主机名未被修改
        self.assertEqual(host_a.hostnames, ["a"])
        self.assertEqual(host_b.hostnames, ["b"])
        # 断言扫描结果中的主机信息未被修改
        self.assertEqual(scan_1.nmap["hosts"], [host_a])
        self.assertEqual(scan_2.nmap["hosts"], [host_b])
        # 断言网络清单中的主机信息未被修改
        self.assertEqual(inv.get_hosts_up()[0].hostnames, ["b"])
    # 定义一个测试函数，用于测试取消和移除扫描是否会影响清单主机
    def test_cancel_and_remove_scan(self):
        """Test that canceling and removing a scan does not blow away the
        inventory hosts"""
        # 添加的 IP 地址列表
        added_ips = ['10.0.0.1', '10.0.0.2']
        # 移除的 IP 地址列表
        removed_ips = ['10.0.0.3']
        # 创建一个扫描对象 scan_1
        scan_1 = zenmapCore.NmapParser.ParserBasics()
        # 创建一个主机对象 host_a，并设置主机名和 IP 地址
        host_a = zenmapCore.NmapParser.HostInfo()
        host_a.hostnames = ["a"]
        host_a.set_ip({'addr': added_ips[0]})
        # 设置扫描开始时间
        scan_1.start = "1000000000"
        # 设置扫描对象的主机信息
        scan_1.nmap["hosts"] = [host_a]

        # 创建一个扫描对象 scan_2
        scan_2 = zenmapCore.NmapParser.ParserBasics()
        # 创建一个主机对象 host_b，并设置主机名和 IP 地址
        host_b = zenmapCore.NmapParser.HostInfo()
        host_b.hostnames = ["b"]
        host_b.set_ip({'addr': added_ips[1]})
        # 设置扫描开始时间
        scan_2.start = "1000000001"
        # 设置扫描对象的主机信息
        scan_2.nmap["hosts"] = [host_b]

        # 创建一个扫描对象 scan_3
        scan_3 = zenmapCore.NmapParser.ParserBasics()
        # 创建一个主机对象 host_c，并设置主机名和 IP 地址
        host_c = zenmapCore.NmapParser.HostInfo()
        host_c.hostnames = ["b"]
        host_c.set_ip({'addr': removed_ips[0]})
        # 设置扫描开始时间
        scan_3.start = "1000000001"
        # 设置扫描对象的主机信息
        scan_3.nmap["hosts"] = [host_c]

        # 创建一个网络清单对象
        inv = NetworkInventory()
        # 向网络清单中添加扫描对象 scan_1
        inv.add_scan(scan_1)
        # 向网络清单中添加扫描对象 scan_2
        inv.add_scan(scan_2)
        # 尝试从网络清单中移除扫描对象 scan_3，如果出现异常则捕获并忽略
        try:
            inv.remove_scan(scan_3)
        except Exception:
            pass
        # 断言添加的 IP 地址列表与网络清单中的主机列表相同
        self.assertEqual(added_ips, list(inv.hosts.keys()))
        # 断言主机对象 host_a 的主机名为 ["a"]
        self.assertEqual(host_a.hostnames, ["a"])
        # 断言主机对象 host_b 的主机名为 ["b"]
        self.assertEqual(host_b.hostnames, ["b"])
# 定义一个测试类 FilteredNetworkInventoryTest，继承自 unittest.TestCase
class FilteredNetworkInventoryTest(unittest.TestCase):
    # 定义一个测试方法 test_filter
    def test_filter(self):
        """Test that the filter still works after moving code to the """
        """HostSearch class"""
        # 从 zenmapCore.NmapParser 模块导入 NmapParser 类
        from zenmapCore.NmapParser import NmapParser
        # 创建 FilteredNetworkInventory 实例
        inv = FilteredNetworkInventory()
        # 创建 NmapParser 实例
        scan = NmapParser()
        # 解析指定 XML 文件
        scan.parse_file("test/xml_test9.xml")
        # 设置过滤条件
        filter_text = "open:22 os:linux service:openssh"
        # 将扫描结果添加到网络清单中
        inv.add_scan(scan)
        # 应用过滤条件
        inv.apply_filter(filter_text)
        # 断言网络清单中的主机数量为 2
        assert(len(inv.get_hosts()) == 2)


# 定义一个测试类 PortChangeTest，继承自 unittest.TestCase
class PortChangeTest(unittest.TestCase):
    # 定义一个测试方法 test_port
    def test_port(self):
        """Verify that the port status (open/filtered/closed) is displayed
        correctly when the port status changes in newer scans"""
        # 从 zenmapCore.NmapParser 模块导入 NmapParser 类
        from zenmapCore.NmapParser import NmapParser
        # 创建 NetworkInventory 实例
        inv = NetworkInventory()
        # 创建 NmapParser 实例
        scan1 = NmapParser()
        # 解析指定 XML 文件
        scan1.parse_file("test/xml_test13.xml")
        # 将扫描结果添加到网络清单中
        inv.add_scan(scan1)
        # 创建 NmapParser 实例
        scan2 = NmapParser()
        # 解析指定 XML 文件
        scan2.parse_file("test/xml_test14.xml")
        # 将扫描结果添加到网络清单中
        inv.add_scan(scan2)
        # 断言网络清单中第一个主机的端口数量为 2
        assert(len(inv.get_hosts()[0].ports) == 2)
        # 创建 NmapParser 实例
        scan3 = NmapParser()
        # 解析指定 XML 文件
        scan3.parse_file("test/xml_test15.xml")
        # 将扫描结果添加到网络清单中
        inv.add_scan(scan3)
        # 断言网络清单中第一个主机的端口数量为 0

        # 重新创建 NetworkInventory 实例，用于额外的测试用例
        inv = NetworkInventory()
        # 创建 NmapParser 实例
        scan4 = NmapParser()
        # 解析指定 XML 文件
        scan4.parse_file("test/xml_test16.xml")
        # 将扫描结果添加到网络清单中
        inv.add_scan(scan4)
        # 断言网络清单中第一个主机的端口数量为 3
        assert(len(inv.get_hosts()[0].ports) == 3)
        # 创建 NmapParser 实例
        scan5 = NmapParser()
        # 解析指定 XML 文件
        scan5.parse_file("test/xml_test17.xml")
        # 将扫描结果添加到网络清单中
        inv.add_scan(scan5)
        # 断言网络清单中第一个主机的端口数量为 7

# 如果当前脚本为主程序，则执行测试
if __name__ == "__main__":
    unittest.main()
    # 如果条件为假，则执行以下代码块
    if False:
        # 从指定路径解析 Nmap XML 文件，创建 NmapParser 对象
        scan1 = NmapParser("/home/ndwi/scanz/neobee_1.xml")
        # 解析 NmapParser 对象
        scan1.parse()
        # 从指定路径解析 Nmap 用户定义的扫描文件，创建 NmapParser 对象
        scan2 = NmapParser("/home/ndwi/scanz/scanme_nmap_org.usr")
        # 解析 NmapParser 对象
        scan2.parse()

        # 创建 NetworkInventory 对象
        inventory1 = NetworkInventory()
        # 添加扫描结果到 inventory1
        inventory1.add_scan(scan1)
        inventory1.add_scan(scan2)

        # 遍历 inventory1 中的主机
        for host in inventory1.get_hosts():
            # 打印主机 IP 地址
            print("%s" % host.ip["addr"], end=' ')
            # 如果主机有主机名，则打印第一个主机名
            # 如果主机没有主机名，则打印冒号
            # 遍历主机的端口，打印端口号和端口状态
            # 打印主机的操作系统匹配信息
            # 打印主机使用的端口
            # 打印主机的跟踪信息
            # 如果主机的跟踪信息中有跳数信息，则打印跳数
        # 从 inventory1 中移除 scan2 的扫描结果
        inventory1.remove_scan(scan2)
        # 打印空行
        print
        # 再次遍历 inventory1 中的主机
        for host in inventory1.get_hosts():
            # 打印主机 IP 地址
            print("%s" % host.ip["addr"], end=' ')
        # 将 inventory1 中的数据保存到指定目录
        dir = "/home/ndwi/scanz/top01"
        inventory1.save_to_dir(dir)

        # 创建新的 NetworkInventory 对象
        inventory2 = NetworkInventory()
        # 从指定目录中加载数据到 inventory2
        inventory2.open_from_dir(dir)

        # 打印空行
        print()
        # 遍历 inventory2 中的主机
        for host in inventory2.get_hosts():
            # 打印主机 IP 地址
            print("%s" % host.ip["addr"], end=' ')
```