# `nmap\zenmap\zenmapCore\UmitConf.py`

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
# 导入正则表达式模块
import re

# 导入配置文件相关的异常类
from configparser import DuplicateSectionError, NoSectionError, NoOptionError
from configparser import Error as ConfigParser_Error

# 从 zenmapCore.Paths 模块中导入 Path 类
from zenmapCore.Paths import Path

# 从 zenmapCore.UmitLogging 模块中导入 log 函数
from zenmapCore.UmitLogging import log

# 从 zenmapCore.UmitConfigParser 模块中导入 UmitConfigParser 类
from zenmapCore.UmitConfigParser import UmitConfigParser

# 导入 zenmapCore.I18N 模块，但未使用
import zenmapCore.I18N  # lgtm[py/unused-import]

# 创建全局配置解析器对象，表示 zenmap.conf 的内容
# 应用程序应该初始化一次
config_parser = UmitConfigParser()

# 检查是否在 Maemo 平台上运行
MAEMO = False
try:
    # 尝试导入 hildon 模块，如果成功则表示在 Maemo 平台上运行
    import hildon
    MAEMO = True
except ImportError:
    pass

# 返回是否在 Maemo 平台上运行的布尔值
def is_maemo():
    return MAEMO
# 定义一个名为 SearchConfig 的类，继承自 UmitConfigParser 和 object
class SearchConfig(UmitConfigParser, object):
    # 设置类属性 section_name 为 "search"
    section_name = "search"

    # 初始化方法
    def __init__(self):
        # 如果配置解析器中没有名为 section_name 的部分，则创建该部分
        if not config_parser.has_section(self.section_name):
            self.create_section()

    # 保存更改的方法
    def save_changes(self):
        config_parser.save_changes()

    # 创建部分的方法
    def create_section(self):
        # 在配置解析器中添加名为 section_name 的部分
        config_parser.add_section(self.section_name)
        # 设置类属性 directory 为空字符串
        self.directory = ""
        # 设置类属性 file_extension 为 "xml"
        self.file_extension = "xml"
        # 设置类属性 save_time 为 "60;days"
        self.save_time = "60;days"
        # 设置类属性 store_results 为 True
        self.store_results = True
        # 设置类属性 search_db 为 True
        self.search_db = True

    # 获取属性值的方法
    def _get_it(self, p_name, default):
        return config_parser.get(self.section_name, p_name, fallback=default)

    # 设置属性值的方法
    def _set_it(self, p_name, value):
        config_parser.set(self.section_name, p_name, value)

    # 对布尔值进行规范化的方法
    def boolean_sanity(self, attr):
        # 如果属性为 True 或者 "True" 或者 "true" 或者 "1"，则返回 "True"，否则返回 "False"
        if attr is True or \
           attr == "True" or \
           attr == "true" or \
           attr == "1":
            return "True"
        return "False"

    # 获取 directory 属性值的方法
    def get_directory(self):
        return self._get_it("directory", "")

    # 设置 directory 属性值的方法
    def set_directory(self, directory):
        self._set_it("directory", directory)

    # 获取 file_extension 属性值的方法
    def get_file_extension(self):
        return self._get_it("file_extension", "xml").split(";")

    # 设置 file_extension 属性值的方法
    def set_file_extension(self, file_extension):
        if isinstance(file_extension, list):
            self._set_it("file_extension", ";".join(file_extension))
        elif isinstance(file_extension, str):
            self._set_it("file_extension", file_extension)

    # 获取 save_time 属性值的方法
    def get_save_time(self):
        return self._get_it("save_time", "60;days").split(";")

    # 设置 save_time 属性值的方法
    def set_save_time(self, save_time):
        if isinstance(save_time, list):
            self._set_it("save_time", ";".join(save_time))
        elif isinstance(save_time, str):
            self._set_it("save_time", save_time)

    # 获取 store_results 属性值的方法
    def get_store_results(self):
        return self.boolean_sanity(self._get_it("store_results", True))
    # 设置存储结果的属性，将其转换为布尔值后设置到实例属性中
    def set_store_results(self, store_results):
        self._set_it("store_results", self.boolean_sanity(store_results))
    
    # 获取搜索数据库的属性，将其转换为布尔值后返回
    def get_search_db(self):
        return self.boolean_sanity(self._get_it("search_db", True))
    
    # 设置搜索数据库的属性，将其转换为布尔值后设置到实例属性中
    def set_search_db(self, search_db):
        self._set_it("search_db", self.boolean_sanity(search_db))
    
    # 获取转换后的保存时间，如果出现异常则返回默认的保存时间（60天）
    def get_converted_save_time(self):
        try:
            return int(self.save_time[0]) * self.time_list[self.save_time[1]]
        except Exception:
            # 如果出现异常，返回默认的保存时间（60天）
            return 60 * 60 * 24 * 60
    
    # 获取时间单位和对应的秒数的字典
    def get_time_list(self):
        # 时间作为键，秒数作为值
        return {"hours": 60 * 60,
                "days": 60 * 60 * 24,
                "weeks": 60 * 60 * 24 * 7,
                "months": 60 * 60 * 24 * 7 * 30,
                "years": 60 * 60 * 24 * 7 * 30 * 12,
                "minutes": 60,
                "seconds": 1}
    
    # 设置属性为目录，使用property装饰器
    directory = property(get_directory, set_directory)
    # 设置属性为文件扩展名，使用property装饰器
    file_extension = property(get_file_extension, set_file_extension)
    # 设置属性为保存时间，使用property装饰器
    save_time = property(get_save_time, set_save_time)
    # 设置属性为存储结果，使用property装饰器
    store_results = property(get_store_results, set_store_results)
    # 设置属性为搜索数据库，使用property装饰器
    search_db = property(get_search_db, set_search_db)
    # 设置属性为转换后的保存时间，使用property装饰器
    converted_save_time = property(get_converted_save_time)
    # 设置属性为时间单位和对应的秒数的字典，使用property装饰器
    time_list = property(get_time_list)
class Profile(UmitConfigParser, object):
    """This class represents not just one profile, but a whole collection of
    them found in a config file such as scan_profiles.usp. The methods
    therefore all take an argument that is the name of the profile to work
    on."""

    def __init__(self, user_profile=None, *args):
        # 调用父类的初始化方法
        UmitConfigParser.__init__(self, *args)

        try:
            if not user_profile:
                user_profile = Path.scan_profile

            # 读取用户配置文件
            self.read(user_profile)
        except ConfigParser_Error as e:
            # 如果配置文件不存在或损坏，则添加默认配置
            self.add_profile(_("Profiles not found"),
                    command="nmap",
                    description=_("The {} file is missing or corrupted"
                        ).format(user_profile))

        # 初始化属性字典
        self.attributes = {}

    def _get_it(self, profile, attribute):
        # 如果验证通过，则获取指定配置文件的指定属性值
        if self._verify_profile(profile):
            return self.get(profile, attribute)
        return ""

    def _set_it(self, profile, attribute, value=''):
        # 如果验证通过，则设置指定配置文件的指定属性值
        if self._verify_profile(profile):
            return self.set(profile, attribute, value)

    def add_profile(self, profile_name, **attributes):
        """Add a profile with the given name and attributes to the collection
        of profiles. If a profile with the same name exists, it is not
        overwritten, and the method returns immediately. The backing file for
        the profiles is automatically updated."""

        # 记录添加配置文件的日志
        log.debug(">>> Add Profile '%s': %s" % (profile_name, attributes))

        try:
            # 尝试添加新的配置文件
            self.add_section(profile_name)
        except DuplicateSectionError:
            return None

        # 设置每个属性（“command”，“description”）在ConfigParser中
        for attr in attributes:
            self._set_it(profile_name, attr, attributes[attr])

        # 保存更改到配置文件
        self.save_changes()
    # 从配置文件中移除指定的配置文件
    def remove_profile(self, profile_name):
        # 尝试从配置文件中移除指定的配置文件
        try:
            self.remove_section(profile_name)
        # 如果出现异常则忽略
        except Exception:
            pass
        # 保存对配置文件的修改
        self.save_changes()
    
    # 验证配置文件中是否存在指定的配置文件
    def _verify_profile(self, profile_name):
        # 如果指定的配置文件名不在配置文件的节中，则返回 False
        if profile_name not in self.sections():
            return False
        # 否则返回 True
        return True
# 定义 WindowConfig 类，继承自 UmitConfigParser 类和 object 类
class WindowConfig(UmitConfigParser, object):
    # 设置 section_name 属性为 "window"
    section_name = "window"

    # 设置默认的窗口位置和大小
    default_x = 0
    default_y = 0
    default_width = -1
    default_height = 650

    # 初始化方法
    def __init__(self):
        # 如果配置解析器中没有 "window" 这个 section，则创建该 section
        if not config_parser.has_section(self.section_name):
            self.create_section()

    # 保存更改的方法
    def save_changes(self):
        config_parser.save_changes()

    # 创建 section 的方法
    def create_section(self):
        config_parser.add_section(self.section_name)
        self.x = self.default_x
        self.y = self.default_y
        self.width = self.default_width
        self.height = self.default_height

    # 获取配置项的方法
    def _get_it(self, p_name, default):
        return config_parser.get(self.section_name, p_name, fallback=default)

    # 设置配置项的方法
    def _set_it(self, p_name, value):
        config_parser.set(self.section_name, p_name, value)

    # 获取窗口 x 坐标的方法
    def get_x(self):
        try:
            value = int(self._get_it("x", self.default_x))
        except (ValueError, NoOptionError):
            value = self.default_x
        except TypeError as e:
            v = self._get_it("x", self.default_x)
            log.exception("Trouble parsing x value as int: %s",
                    repr(v), exc_info=e)
            value = self.default_x
        return value

    # 设置窗口 x 坐标的方法
    def set_x(self, x):
        self._set_it("x", "%d" % x)

    # 获取窗口 y 坐标的方法
    def get_y(self):
        try:
            value = int(self._get_it("y", self.default_y))
        except (ValueError, NoOptionError):
            value = self.default_y
        except TypeError as e:
            v = self._get_it("y", self.default_y)
            log.exception("Trouble parsing y value as int: %s",
                    repr(v), exc_info=e)
            value = self.default_y
        return value

    # 设置窗口 y 坐标的方法
    def set_y(self, y):
        self._set_it("y", "%d" % y)
    # 获取宽度属性的值
    def get_width(self):
        try:
            # 尝试将获取到的宽度属性值转换为整数
            value = int(self._get_it("width", self.default_width))
        except (ValueError, NoOptionError):
            # 如果获取值时出现错误，则使用默认宽度值
            value = self.default_width
        except TypeError as e:
            # 如果获取值时出现类型错误，则记录异常信息并使用默认宽度值
            v = self._get_it("width", self.default_width)
            log.exception("Trouble parsing width value as int: %s",
                    repr(v), exc_info=e)
            value = self.default_width

        # 如果值小于-1，则使用默认宽度值
        if not (value >= -1):
            value = self.default_width

        # 返回宽度属性的值
        return value

    # 设置宽度属性的值
    def set_width(self, width):
        # 将宽度属性的值设置为指定的宽度值
        self._set_it("width", "%d" % width)

    # 获取高度属性的值
    def get_height(self):
        try:
            # 尝试将获取到的高度属性值转换为整数
            value = int(self._get_it("height", self.default_height))
        except (ValueError, NoOptionError):
            # 如果获取值时出现错误，则使用默认高度值
            value = self.default_height
        except TypeError as e:
            # 如果获取值时出现类型错误，则记录异常信息并使用默认高度值
            v = self._get_it("height", self.default_height)
            log.exception("Trouble parsing y value as int: %s",
                    repr(v), exc_info=e)
            value = self.default_height

        # 如果值小于-1，则使用默认高度值
        if not (value >= -1):
            value = self.default_height

        # 返回高度属性的值
        return value

    # 设置高度属性的值
    def set_height(self, height):
        # 将高度属性的值设置为指定的高度值
        self._set_it("height", "%d" % height)

    # 使用 property 方法将 get_x 和 set_x 方法绑定到 x 属性
    x = property(get_x, set_x)
    # 使用 property 方法将 get_y 和 set_y 方法绑定到 y 属性
    y = property(get_y, set_y)
    # 使用 property 方法将 get_width 和 set_width 方法绑定到 width 属性
    width = property(get_width, set_width)
    # 使用 property 方法将 get_height 和 set_height 方法绑定到 height 属性
    height = property(get_height, set_height)
# 定义一个名为 CommandProfile 的类，继承自 Profile 类和 object 类
class CommandProfile (Profile, object):
    """This class is a wrapper around Profile that provides accessors for the
    attributes of a profile: command and description"""
    # 初始化方法，接受一个 user_profile 参数，默认为 None
    def __init__(self, user_profile=None):
        # 调用父类 Profile 的初始化方法
        Profile.__init__(self, user_profile)

    # 获取 profile 的 command 属性
    def get_command(self, profile):
        # 从 profile 中获取 command 属性的值
        command_string = self._get_it(profile, 'command')
        # 如果配置文件损坏，可能包含多个命令，取第一个命令
        if isinstance(command_string, list):
            command_string = command_string[0]
        # 如果 command_string 没有 endswith 方法，返回 "nmap"
        if not hasattr(command_string, "endswith"):
            return "nmap"
        # 旧版本的 Zenmap 会在命令末尾添加 "%s" 用于替换目标，如果存在则忽略
        if command_string.endswith("%s"):
            command_string = command_string[:-len("%s")]
        # 返回处理后的 command_string
        return command_string

    # 获取 profile 的 description 属性
    def get_description(self, profile):
        # 从 profile 中获取 description 属性的值
        desc = self._get_it(profile, 'description')
        # 如果 desc 是列表，将其转换为字符串
        if isinstance(desc, list):
            desc = " ".join(desc)
        # 返回处理后的 desc
        return desc

    # 设置 profile 的 command 属性
    def set_command(self, profile, command=''):
        # 设置 profile 的 command 属性为指定的 command
        self._set_it(profile, 'command', command)

    # 设置 profile 的 description 属性
    def set_description(self, profile, description=''):
        # 设置 profile 的 description 属性为指定的 description
        self._set_it(profile, 'description', description)

    # 获取指定 profile 的信息，包括 profile 名称、command 和 description
    def get_profile(self, profile_name):
        return {'profile': profile_name,
                'command': self.get_command(profile_name),
                'description': self.get_description(profile_name)}


# 定义一个名为 NmapOutputHighlight 的类
class NmapOutputHighlight(object):
    # 类属性 setts，包含一些设置项
    setts = ["bold", "italic", "underline", "text", "highlight", "regex"]

    # 保存更改的方法
    def save_changes(self):
        # 调用 config_parser 的 save_changes 方法
        config_parser.save_changes()
    # 从私有方法中获取属性值
    def __get_it(self, p_name):
        # 根据属性名生成属性名加上_highlight的字符串
        property_name = "%s_highlight" % p_name
    
        # 尝试获取属性值，如果失败则返回默认高亮设置
        try:
            return self.sanity_settings([
                # 获取配置文件中的属性值
                config_parser.get(
                    property_name, prop, raw=True) for prop in self.setts])
        except Exception:
            # 如果获取失败，则返回默认高亮设置
            settings = []
            prop_settings = self.default_highlights[p_name]
            settings.append(prop_settings["bold"])
            settings.append(prop_settings["italic"])
            settings.append(prop_settings["underline"])
            settings.append(prop_settings["text"])
            settings.append(prop_settings["highlight"])
            settings.append(prop_settings["regex"])
    
            # 调用私有方法设置属性值
            self.__set_it(p_name, settings)
    
            # 返回默认高亮设置
            return settings
    
    # 设置属性值的私有方法
    def __set_it(self, property_name, settings):
        # 根据属性名生成属性名加上_highlight的字符串
        property_name = "%s_highlight" % property_name
        # 对设置进行检查和处理
        settings = self.sanity_settings(list(settings))
    
        # 遍历设置列表，将设置值写入配置文件
        for pos in range(len(settings)):
            config_parser.set(property_name, self.setts[pos], settings[pos])
    # 对设置进行检查，将不合理的设置转换为合理的设置
    def sanity_settings(self, settings):
        """This method tries to convert insane settings to sanity ones ;-)
        If user send a True, "True" or "true" value, for example, it tries to
        convert then to the integer 1.
        Same to False, "False", etc.

        Sequence: [bold, italic, underline, text, highlight, regex]
        """
        # log.debug(">>> Sanitize %s" % str(settings))

        # 将第一个设置转换为布尔值
        settings[0] = self.boolean_sanity(settings[0])
        # 将第二个设置转换为布尔值
        settings[1] = self.boolean_sanity(settings[1])
        # 将第三个设置转换为布尔值
        settings[2] = self.boolean_sanity(settings[2])

        # 定义元组的正则表达式
        tuple_regex = r"[\(\[]\s?(\d+)\s?,\s?(\d+)\s?,\s?(\d+)\s?[\)\]]"
        # 如果第四个设置是字符串，则根据正则表达式转换为整数列表
        if isinstance(settings[3], str):
            settings[3] = [
                    int(t) for t in re.findall(tuple_regex, settings[3])[0]
                    ]

        # 如果第五个设置是字符串，则根据正则表达式转换为整数列表
        if isinstance(settings[4], str):
            settings[4] = [
                    int(h) for h in re.findall(tuple_regex, settings[4])[0]
                    ]

        # 返回转换后的设置
        return settings

    # 将布尔值转换为整数
    def boolean_sanity(self, attr):
        if attr is True or attr == "True" or attr == "true" or attr == "1":
            return 1
        return 0

    # 获取日期设置
    def get_date(self):
        return self.__get_it("date")

    # 设置日期
    def set_date(self, settings):
        self.__set_it("date", settings)

    # 获取主机名设置
    def get_hostname(self):
        return self.__get_it("hostname")

    # 设置主机名
    def set_hostname(self, settings):
        self.__set_it("hostname", settings)

    # 获取IP设置
    def get_ip(self):
        return self.__get_it("ip")

    # 设置IP
    def set_ip(self, settings):
        self.__set_it("ip", settings)

    # 获取端口列表设置
    def get_port_list(self):
        return self.__get_it("port_list")

    # 设置端口列表
    def set_port_list(self, settings):
        self.__set_it("port_list", settings)

    # 获取开放端口设置
    def get_open_port(self):
        return self.__get_it("open_port")

    # 设置开放端口
    def set_open_port(self, settings):
        self.__set_it("open_port", settings)

    # 获取关闭端口设置
    def get_closed_port(self):
        return self.__get_it("closed_port")
    # 设置关闭端口的属性
    def set_closed_port(self, settings):
        self.__set_it("closed_port", settings)
    
    # 获取过滤端口的属性
    def get_filtered_port(self):
        return self.__get_it("filtered_port")
    
    # 设置过滤端口的属性
    def set_filtered_port(self, settings):
        self.__set_it("filtered_port", settings)
    
    # 获取详细信息的属性
    def get_details(self):
        return self.__get_it("details")
    
    # 设置详细信息的属性
    def set_details(self, settings):
        self.__set_it("details", settings)
    
    # 获取是否启用属性
    def get_enable(self):
        enable = True
        try:
            enable = config_parser.get("output_highlight", "enable_highlight")
        except NoSectionError:
            config_parser.set(
                    "output_highlight", "enable_highlight", str(True))
    
        if enable == "False" or enable == "0" or enable == "":
            return False
        return True
    
    # 设置是否启用属性
    def set_enable(self, enable):
        if enable is False or enable == "0" or enable is None or enable == "":
            config_parser.set(
                    "output_highlight", "enable_highlight", str(False))
        else:
            config_parser.set(
                    "output_highlight", "enable_highlight", str(True))
    
    # 设置属性为日期
    date = property(get_date, set_date)
    # 设置属性为主机名
    hostname = property(get_hostname, set_hostname)
    # 设置属性为IP地址
    ip = property(get_ip, set_ip)
    # 设置属性为端口列表
    port_list = property(get_port_list, set_port_list)
    # 设置属性为开放端口
    open_port = property(get_open_port, set_open_port)
    # 设置属性为关闭端口
    closed_port = property(get_closed_port, set_closed_port)
    # 设置属性为过滤端口
    filtered_port = property(get_filtered_port, set_filtered_port)
    # 设置属性为详细信息
    details = property(get_details, set_details)
    # 设置属性为是否启用
    enable = property(get_enable, set_enable)
    
    # 当尚未设置任何内容时，进行这些设置。它们将设置高亮颜色的“工厂”默认值
# 从 zenmap.conf 中检索关于路径子部分的详细信息（例如 nmap_command_path）- jurand
class PathsConfig(object):
    section_name = "paths"

    # 处理配置文件中缺少的条目
    # 如果出现这些错误，默认为 "nmap"
    # NoOptionError, NoSectionError
    def __get_it(self, p_name, default):
        try:
            return config_parser.get(self.section_name, p_name)
        except (NoOptionError, NoSectionError):
            log.debug(
                    ">>> Using default \"%s\" for \"%s\"." % (default, p_name))
            return default

    def __set_it(self, property_name, settings):
        config_parser.set(self.section_name, property_name, settings)

    # 获取 nmap_command_path 的值
    def get_nmap_command_path(self):
        return self.__get_it("nmap_command_path", "nmap")

    # 设置 nmap_command_path 的值
    def set_nmap_command_path(self, settings):
        self.__set_it("nmap_command_path", settings)

    # 获取 ndiff_command_path 的值
    def get_ndiff_command_path(self):
        return self.__get_it("ndiff_command_path", "ndiff")

    # 设置 ndiff_command_path 的值
    def set_ndiff_command_path(self, settings):
        self.__set_it("ndiff_command_path", settings)

    # 使用 property 装饰器定义 nmap_command_path 属性
    nmap_command_path = property(get_nmap_command_path, set_nmap_command_path)
    # 使用 property 装饰器定义 ndiff_command_path 属性
    ndiff_command_path = property(
            get_ndiff_command_path, set_ndiff_command_path)


# 异常类
class ProfileNotFound:
    def __init__(self, profile):
        self.profile = profile

    def __str__(self):
        return "No profile named '" + self.profile + "' found!"


class ProfileCouldNotBeSaved:
    def __init__(self, profile):
        self.profile = profile

    def __str__(self):
        return "Profile named '" + self.profile + "' could not be saved!"
```