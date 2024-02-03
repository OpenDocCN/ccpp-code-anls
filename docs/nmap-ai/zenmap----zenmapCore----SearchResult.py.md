# `nmap\zenmap\zenmapCore\SearchResult.py`

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
import os.path
import re
import io
import unittest
from glob import glob
# 从 zenmapCore.Name 模块中导入 APP_NAME
from zenmapCore.Name import APP_NAME
# 从 zenmapCore.NmapOptions 模块中导入 NmapOptions
from zenmapCore.NmapOptions import NmapOptions
# 从 zenmapCore.NmapParser 模块中导入 NmapParser
from zenmapCore.NmapParser import NmapParser
# 从 zenmapCore.UmitLogging 模块中导入 log
from zenmapCore.UmitLogging import log

# 定义 HostSearch 类
class HostSearch(object):
    # 静态方法
    @staticmethod
    # 匹配目标主机和名称是否相符
    def match_target(host, name):
        # 将名称转换为小写
        name = name.lower()
        # 获取主机的 MAC 地址
        mac = host.get_mac()
        # 获取主机的 IP 地址
        ip = host.get_ip()
        # 获取主机的 IPv6 地址
        ipv6 = host.get_ipv6()

        # 如果存在 MAC 地址并且包含 'addr' 字段
        if mac and 'addr' in mac:
            # 如果名称在 MAC 地址中
            if name in mac['addr'].lower():
                return True
        # 如果存在 IP 地址并且包含 'addr' 字段
        if ip and 'addr' in ip:
            # 如果名称在 IP 地址中
            if name in ip['addr'].lower():
                return True
        # 如果存在 IPv6 地址并且包含 'addr' 字段
        if ipv6 and 'addr' in ipv6:
            # 如果名称在 IPv6 地址中
            if name in ipv6['addr'].lower():
                return True

        # 调用 HostSearch 类的 match_hostname 方法，匹配主机名
        if HostSearch.match_hostname(host, name):
            return True
        # 如果都不匹配，则返回 False
        return False

    # 静态方法，用于匹配主机名
    @staticmethod
    def match_hostname(host, hostname):
        # 将主机名转换为小写
        hostname = hostname.lower()
        # 获取主机的所有主机名
        hostnames = host.get_hostnames()
        # 遍历所有主机名
        for hn in hostnames:
            # 如果主机名在主机的主机名中
            if hostname in hn['hostname'].lower():
                return True
        else:
            return False

    # 静态方法，用于匹配服务
    @staticmethod
    def match_service(host, service):
        # 遍历主机的所有端口
        for port in host.get_ports():
            # 如果端口状态不是 'open' 或 'open|filtered'，则跳过
            if port['port_state'] not in ['open', 'open|filtered']:
                continue
            # 将所有有用的字段连接起来，添加到列表中
            version = " ".join(
                    port.get(x, "") for x in (
                        "service_name",
                        "service_product",
                        "service_version",
                        "service_extrainfo"
                        )
                    )
            # 如果服务在版本信息中
            if service in version.lower():
                return True
        else:
            return False

    # 静态方法
    @staticmethod
    # 匹配操作系统类型
    def match_os(host, os):
        # 将操作系统类型转换为小写
        os = os.lower()

        # 获取主机的操作系统匹配信息
        osmatches = host.get_osmatches()

        # 遍历操作系统匹配信息
        for osmatch in osmatches:
            # 将操作系统名称转换为小写
            os_str = osmatch['name'].lower()
            # 遍历操作系统类别
            for osclass in osmatch['osclasses']:
                # 将操作系统类别的厂商、家族和类型转换为小写，并拼接到操作系统名称中
                os_str += " " + osclass['vendor'].lower() + " " +\
                          osclass['osfamily'].lower() + " " +\
                          osclass['type'].lower()
            # 检查给定的操作系统类型是否在操作系统名称中，如果是则返回 True
            if os in os_str:
                return True

        # 如果没有匹配的操作系统类型，则返回 False
        return False

    # 匹配端口
    @staticmethod
    def match_port(host_ports, port, port_state):
        # 检查端口是否可解析，如果不可解析则静默返回 False
        if re.match(r"^\d+$", port) is None:
            return False

        # 遍历主机的端口信息
        for hp in host_ports:
            # 如果端口号和端口状态匹配，则返回 True
            if hp['portid'] == port and hp['port_state'] == port_state:
                return True

        # 如果没有匹配的端口信息，则返回 False
        return False
class SearchResult(object):
    def __init__(self):
        """This constructor is always called by SearchResult subclasses."""
        # 空的构造函数，用于SearchResult的子类

    def search(self, **kargs):
        """Performs a search on each parsed scan. Since the 'and' operator is
        implicit, the search fails as soon as one of the tests fails. The
        kargs argument is a map having operators as keys and argument lists as
        values."""
        # 对每个解析的扫描进行搜索。由于'and'操作符是隐式的，一旦一个测试失败，搜索就会失败。
        # kargs参数是一个以操作符为键，参数列表为值的映射。

        for scan_result in self.get_scan_results():
            self.parsed_scan = scan_result

            # Test each given operator against the current parsed result
            for operator, args in kargs.items():
                if not self._match_all_args(operator, args):
                    # No match => we discard this scan_result
                    break
            else:
                # All operator-matching functions have returned True, so this
                # scan_result satisfies all conditions
                yield self.parsed_scan

    def _match_all_args(self, operator, args):
        """A helper function that calls the matching function for the given
        operator and each of its arguments."""
        # 一个辅助函数，调用给定操作符和每个参数的匹配函数。

        for arg in args:
            positive = True
            if arg != "" and arg[0] == "!":
                arg = arg[1:]
                positive = False
            if positive != self.__getattribute__("match_%s" % operator)(arg):
                # No match for this operator
                return False
        else:
            # All arguments for this operator produced a match
            return True

    def get_scan_results(self):
        # To be implemented by classes that are going to inherit this one
        pass
        # 由将要继承这个类的类来实现

    def basic_match(self, keyword, property):
        if keyword == "*" or keyword == "":
            return True

        return keyword.lower() in str(
                self.parsed_scan.__getattribute__(property)).lower()
    # 匹配给定关键字，记录匹配的关键字信息
    def match_keyword(self, keyword):
        log.debug("Match keyword: %s" % keyword)
        # 调用 basic_match 方法匹配关键字，如果匹配成功则返回 True
        return self.basic_match(keyword, "nmap_output") or \
               # 调用 match_profile 方法匹配关键字，如果匹配成功则返回 True
               self.match_profile(keyword) or \
               # 调用 match_target 方法匹配关键字，如果匹配成功则返回 True
               self.match_target(keyword)

    # 匹配给定配置文件，记录匹配的配置文件信息
    def match_profile(self, profile):
        log.debug("Match profile: %s" % profile)
        log.debug("Comparing: %s == %s ??" % (
            str(self.parsed_scan.profile_name).lower(),
            "*%s*" % profile.lower()))
        # 比较给定配置文件名和扫描结果的配置文件名，如果匹配成功则返回 True
        return (profile == "*" or profile == "" or
                profile.lower() in str(self.parsed_scan.profile_name).lower())

    # 匹配给定选项，记录匹配的选项信息
    def match_option(self, option):
        log.debug("Match option: %s" % option)

        # 如果选项为通配符或空，则返回 True
        if option == "*" or option == "":
            return True

        # 创建 NmapOptions 对象，解析扫描命令
        ops = NmapOptions()
        ops.parse_string(self.parsed_scan.get_nmap_command())

        # 如果选项包含括号，则解析选项名和值进行匹配
        if "(" in option and ")" in option:
            # 语法允许匹配选项参数，如 "opt:option_name(value)"，需要解析出选项名和值
            optname = option[:option.find("(")]
            optval = option[option.find("(") + 1:option.find(")")]

            val = ops["--" + optname]
            if val is None:
                val = ops["-" + optname]
            if val is None:
                return False
            return str(val) == optval or str(val) == optval
        else:
            # 如果选项存在于扫描命令中，则返回 True
            return (ops["--" + option] is not None or
                    ops["-" + option] is not None)
    # 匹配日期，根据给定的日期参数和操作符进行比较
    def match_date(self, date_arg, operator="date"):
        # 解析扫描的日期，返回一个 time.struct_time 对象，需要将其转换为日期对象
        from datetime import date, datetime
        scd = self.parsed_scan.get_date()
        scan_date = date(scd.tm_year, scd.tm_mon, scd.tm_mday)

        # 检查日期参数中是否包含模糊操作符（"~"）
        fuzz = 0
        if "~" in date_arg:
            # 统计模糊操作符的数量，并将其从日期参数中去除
            fuzz = date_arg.count("~")
            date_arg = date_arg.replace("~", "")

        # 如果日期参数匹配 "YYYY-MM-DD" 格式，则将其解析为日期对象
        if re.match(r"\d\d\d\d-\d\d-\d\d$", date_arg) is not None:
            year, month, day = date_arg.split("-")
            parsed_date = date(int(year), int(month), int(day))
        # 如果日期参数匹配 "-n" 格式（n 天前），则将其转换为日期对象
        elif re.match(r"[-|\+]\d+$", date_arg):
            parsed_date = date.fromordinal(
                    date.today().toordinal() + int(date_arg))
        else:
            # 静默失败，返回 False
            return False

        # 根据操作符（date, after, before）进行日期比较
        if operator == "date":
            return abs((scan_date - parsed_date).days) <= fuzz
        # 对于 after: 和 before: 操作符，忽略模糊度
        elif operator == "after":
            return (scan_date - parsed_date).days >= 0
        elif operator == "before":
            return (parsed_date - scan_date).days >= 0

    # 匹配给定日期之后的扫描日期
    def match_after(self, date_arg):
        return self.match_date(date_arg, operator="after")

    # 匹配给定日期之前的扫描日期
    def match_before(self, date_arg):
        return self.match_date(date_arg, operator="before")
    # 匹配目标，记录匹配的目标
    def match_target(self, target):
        log.debug("Match target: %s" % target)

        # 遍历已解析的扫描目标列表
        for spec in self.parsed_scan.get_targets():
            # 如果目标在列表中，返回 True
            if target in spec:
                return True
        else:
            # 在 (rDNS) 主机名列表中搜索
            for host in self.parsed_scan.get_hosts():
                # 如果匹配到目标主机，返回 True
                if HostSearch.match_target(host, target):
                    return True
        # 如果没有匹配到目标，返回 False
        return False

    # 匹配操作系统，记录匹配的操作系统
    def match_os(self, os):
        # 如果你的数据库中有大量的大型扫描（扫描了大量主机），最好使用关键字（自由文本）搜索。
        # 关键字搜索只是在 nmap 输出中进行搜索，而这个函数会遍历每个扫描中每个主机的所有解析的与操作系统相关的值！
        hosts = self.parsed_scan.get_hosts()
        # 遍历主机列表
        for host in hosts:
            # 如果匹配到操作系统，返回 True
            if HostSearch.match_os(host, os):
                return True
        # 如果没有匹配到操作系统，返回 False
        return False
    # 检查给定的端口是否与已扫描的端口匹配
    def match_scanned(self, ports):
        # 如果端口为空字符串，则返回 True
        if ports == "":
            return True

        # 将逗号分隔的端口字符串转换为列表
        ports = [not_empty for not_empty in ports.split(",") if not_empty]

        # 检查端口是否可解析，如果不能则静默返回 False
        for port in ports:
            if re.match(r"^\d+$", port) is None:
                return False

        # 创建一个包含所有已扫描端口的列表
        services = []
        for scaninfo in self.parsed_scan.get_scaninfo():
            services.append(scaninfo["services"].split(","))

        # 这两个循环分别遍历搜索端口和已扫描端口。一旦搜索在已扫描端口中找到给定端口，就会从服务循环中跳出，并继续处理端口列表中的下一个端口。如果在服务列表中找不到端口，则立即返回 False。
        for port in ports:
            for service in services:
                if "-" in service and \
                   int(port) >= int(service.split("-")[0]) and \
                   int(port) <= int(service.split("-")[1]):
                    # 端口范围，且我们的端口在范围内
                    break
                elif port == service:
                    break
            else:
                return False
        else:
            # 端口循环完成所有端口，这意味着搜索成功
            return True
    # 匹配端口，判断端口是否符合条件
    def match_port(self, ports, port_state):
        # 记录调试信息，输出匹配的端口
        log.debug("Match port:%s" % ports)

        # 将逗号分隔的端口字符串转换为列表
        ports = [not_empty for not_empty in ports.split(",") if not_empty]

        # 遍历扫描结果中的主机
        for host in self.parsed_scan.get_hosts():
            # 遍历端口列表
            for port in ports:
                # 如果端口不匹配指定状态，则跳出循环
                if not HostSearch.match_port(
                        host.get_ports(), port, port_state):
                    break
            else:
                return True
        else:
            return False

    # 匹配开放的端口
    def match_open(self, port):
        return self.match_port(port, "open")

    # 匹配被过滤的端口
    def match_filtered(self, port):
        return self.match_port(port, "filtered")

    # 匹配关闭的端口
    def match_closed(self, port):
        return self.match_port(port, "closed")

    # 匹配未被过滤的端口
    def match_unfiltered(self, port):
        return self.match_port(port, "unfiltered")

    # 匹配开放和被过滤的端口
    def match_open_filtered(self, port):
        return self.match_port(port, "open|filtered")

    # 匹配关闭和被过滤的端口
    def match_closed_filtered(self, port):
        return self.match_port(port, "closed|filtered")

    # 匹配服务版本
    def match_service(self, sversion):
        # 如果服务版本为空或为通配符，则返回True
        if sversion == "" or sversion == "*":
            return True

        # 遍历扫描结果中的主机，匹配服务版本
        for host in self.parsed_scan.get_hosts():
            if HostSearch.match_service(host, sversion):
                return True
        else:
            return False
    # 在路由中匹配主机名，如果主机名为空或为通配符，则返回True
    def match_in_route(self, host):
        if host == "" or host == "*":
            return True
        # 将主机名转换为小写
        host = host.lower()

        # 由于解析器不解析 traceroute 输出，我们需要在 Nmap 输出中查找主机，在扫描的 Traceroute 部分作弊
        nmap_out = self.parsed_scan.get_nmap_output()
        tr_pos = 0
        traceroutes = []        # 每个主机的扫描包含一个 traceroute 部分
        while tr_pos != -1:
            # 找到 traceroute 部分的开头和结尾，并将子字符串追加到 traceroutes 列表中
            tr_pos = nmap_out.find("TRACEROUTE", tr_pos + 1)
            tr_end_pos = nmap_out.find("\n\n", tr_pos)
            if tr_pos != -1:
                traceroutes.append(nmap_out[tr_pos:tr_end_pos])

        # 遍历 traceroutes 列表，如果主机名在 traceroute 中，则返回True
        for tr in traceroutes:
            if host in tr.lower():
                return True
        # 如果没有找到匹配的主机名，则返回False
        else:
            return False
class SearchDummy(SearchResult):
    """A dummy search class that returns no results. It is used as a
    placeholder when SearchDB can't be used."""
    # 重写父类的方法，返回空列表作为搜索结果
    def get_scan_results(self):
        return []


class SearchDB(SearchResult, object):
    # 初始化方法，继承父类的初始化方法
    def __init__(self):
        SearchResult.__init__(self)
        # 记录调试信息
        log.debug(">>> Getting scan results stored in data base")
        # 初始化扫描结果列表
        self.scan_results = []
        # 导入 UmitDB 类
        from zenmapCore.UmitDB import UmitDB
        u = UmitDB()

        # 遍历数据库中的扫描结果
        for scan in u.get_scans():
            # 记录调试信息
            log.debug(">>> Retrieving result of scans_id %s" % scan.scans_id)
            log.debug(">>> Nmap xml output: %s" % scan.nmap_xml_output)

            try:
                # 将扫描结果的 XML 输出转换为字符串缓冲区
                buffer = io.StringIO(scan.nmap_xml_output)
                # 创建 NmapParser 实例
                parsed = NmapParser()
                # 解析 XML 输出
                parsed.parse(buffer)
                buffer.close()
            except Exception as e:
                # 记录警告信息，输出错误信息
                log.warning(">>> Error loading scan with ID %u from database: "
                        "%s" % (scan.scans_id, str(e)))
            else:
                # 将解析后的结果添加到扫描结果列表中
                self.scan_results.append(parsed)

    # 获取扫描结果的方法
    def get_scan_results(self):
        return self.scan_results


class SearchDir(SearchResult, object):
    # 初始化搜索目录和文件扩展名，默认为"usr"
    def __init__(self, search_directory, file_extensions=["usr"]):
        # 调用父类的初始化方法
        SearchResult.__init__(self)
        # 输出调试信息
        log.debug(">>> SearchDir initialized")
        # 设置搜索目录
        self.search_directory = search_directory

        # 判断文件扩展名的类型，如果是字符串则按分号分割，如果是列表则直接使用，否则抛出异常
        if isinstance(file_extensions, str):
            self.file_extensions = file_extensions.split(";")
        elif isinstance(file_extensions, list):
            self.file_extensions = file_extensions
        else:
            raise Exception(
                    "Wrong file extension format! '%s'" % file_extensions)

        # 输出调试信息
        log.debug(">>> Getting directory's scan results")
        # 初始化扫描结果列表
        self.scan_results = []
        # 初始化文件列表
        files = []
        # 遍历文件扩展名列表，获取对应目录下的文件列表
        for ext in self.file_extensions:
            files += glob(os.path.join(self.search_directory, "*.%s" % ext))

        # 输出调试信息，显示扫描结果的文件列表
        log.debug(">>> Scan results at selected directory: %s" % files)
        # 遍历文件列表，逐个处理扫描结果
        for scan_file in files:
            log.debug(">>> Retrieving scan result %s" % scan_file)
            # 判断文件是否可读且为普通文件
            if os.access(scan_file, os.R_OK) and os.path.isfile(scan_file):

                try:
                    # 解析扫描结果文件
                    parsed = NmapParser()
                    parsed.parse_file(scan_file)
                except Exception:
                    # 如果解析出现异常，则跳过该文件
                    pass
                else:
                    # 如果解析成功，则将结果添加到扫描结果列表中
                    self.scan_results.append(parsed)

    # 获取扫描结果列表
    def get_scan_results(self):
        return self.scan_results
class SearchResultTest(unittest.TestCase):
    # 定义一个用于单元测试的搜索类
    class SearchClass(SearchResult):
        """This class is for use by the unit testing code"""
        # 初始化方法，接受文件名列表作为参数
        def __init__(self, filenames):
            # 调用 SearchResult 类的初始化方法
            SearchResult.__init__(self)
            # 初始化扫描结果列表
            self.scan_results = []
            # 遍历文件名列表，解析每个文件并将结果添加到扫描结果列表中
            for filename in filenames:
                scan = NmapParser()
                scan.parse_file(filename)
                self.scan_results.append(scan)

        # 获取扫描结果的方法
        def get_scan_results(self):
            return self.scan_results

    # 设置测试环境
    def setUp(self):
        # 创建文件名列表
        files = ["test/xml_test%d.xml" % no for no in range(1, 13)]
        # 使用文件名列表创建 SearchClass 实例
        self.search_result = self.SearchClass(files)

    # 测试骨架方法，接受键和值作为参数
    def _test_skeleton(self, key, val):
        results = []
        # 构建搜索条件字典
        search = {key: [val]}
        # 遍历搜索结果并添加到结果列表中
        for scan in self.search_result.search(**search):
            results.append(scan)
        return len(results)

    # 测试匹配操作系统的方法
    def test_match_os(self):
        """Test that checks if the match_os predicate works"""
        # 断言匹配操作系统为 linux 的结果数量为 2
        assert(self._test_skeleton('os', 'linux') == 2)

    # 测试匹配目标的方法
    def test_match_target(self):
        """Test that checks if the match_target predicate works"""
        # 断言匹配目标为 localhost 的结果数量为 4
        assert(self._test_skeleton('target', 'localhost') == 4)

    # 测试匹配开放端口的方法
    def test_match_port_open(self):
        """Test that checks if the match_open predicate works"""
        # 断言匹配开放端口为 22 的结果数量为 7
        assert(self._test_skeleton('open', '22') == 7)

    # 测试匹配关闭端口的方法
    def test_match_port_closed(self):
        """Test that checks if the match_closed predicate works"""
        # 断言匹配开放端口为 22 的结果数量为 7
        assert(self._test_skeleton('open', '22') == 7)
        # 断言匹配关闭端口为 22 的结果数量为 9
        assert(self._test_skeleton('closed', '22') == 9)

    # 测试匹配服务的方法
    def test_match_service(self):
        """Test that checks if the match_service predicate works"""
        # 断言匹配服务为 apache 的结果数量为 9
        assert(self._test_skeleton('service', 'apache') == 9)
        # 断言匹配服务为 openssh 的结果数量为 7
        assert(self._test_skeleton('service', 'openssh') == 7)

    # 测试匹配服务版本的方法
    def test_match_service_version(self):
        """Test that checks if the match_service predicate works when """
        """checking version"""
        # 断言匹配服务为 2.0.52 的结果数量为 7
        assert(self._test_skeleton('service', '2.0.52') == 7)
# 如果当前模块是主程序，则执行测试
if __name__ == "__main__":
    # 调用 unittest 模块的主函数执行测试
    unittest.main()
```