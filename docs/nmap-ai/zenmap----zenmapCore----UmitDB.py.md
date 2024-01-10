# `nmap\zenmap\zenmapCore\UmitDB.py`

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
# 导入所需的模块
import sqlite3  # 导入sqlite3模块
import sys  # 导入sys模块
from hashlib import md5  # 从hashlib模块中导入md5函数
from time import time  # 从time模块中导入time函数
from zenmapCore.Paths import Path  # 从zenmapCore.Paths模块中导入Path类
from zenmapCore.UmitLogging import log  # 从zenmapCore.UmitLogging模块中导入log函数

# 初始化umitdb为空字符串
umitdb = ""

# 尝试获取umitdb的值
try:
    umitdb = Path.db  # 尝试获取Path.db的值
except Exception:
    import os.path  # 导入os.path模块
    from .BasePaths import base_paths  # 从当前目录下的BasePaths模块中导入base_paths变量

    # 如果获取失败，则设置umitdb的值为默认路径
    umitdb = os.path.join(Path.user_config_dir, base_paths["db"])  # 设置umitdb的值为默认路径
    Path.db = umitdb  # 设置Path.db的值为umitdb

# 导入exists、dirname、access、R_OK、W_OK函数
from os.path import exists, dirname
from os import access, R_OK, W_OK

# 初始化using_memory为False
using_memory = False

# 检查umitdb文件是否存在，以及是否有读写权限
if not exists(umitdb) or \
   not access(umitdb, R_OK and W_OK) or \
   not access(dirname(umitdb), R_OK and W_OK):
    # 如果文件不存在或者没有读写权限，则将umitdb设置为内存路径，并将using_memory设置为True
    umitdb = ":memory:"  # 将umitdb设置为内存路径
    using_memory = True  # 将using_memory设置为True

# 连接到umitdb数据库
connection = sqlite3.connect(umitdb)  # 连接到umitdb数据库

# 定义Table类
class Table(object):
    # 初始化方法，接受表名参数，并设置表名和表ID属性
    def __init__(self, table_name):
        self.table_name = table_name
        self.table_id = "%s_id" % table_name

        # 创建数据库连接游标
        self.cursor = connection.cursor()

    # 获取指定项的数值
    def get_item(self, item_name):
        # 如果已经存在该项的属性，则直接返回该属性的值
        if self.__getattribute__("_%s" % item_name):
            return self.__getattribute__("_%s" % item_name)

        # 构建 SQL 查询语句，查询指定项的值
        sql = "SELECT %s FROM %s WHERE %s_id = %s" % (
                item_name,
                self.table_name,
                self.table_name,
                self.__getattribute__(self.table_id))

        # 执行 SQL 查询
        self.cursor.execute(sql)

        # 将查询结果赋值给指定项的属性，并返回该属性的值
        self.__setattr__("_%s" % item_name, self.cursor.fetchall()[0][0])
        return self.__getattribute__("_%s" % item_name)

    # 设置指定项的数值
    def set_item(self, item_name, item_value):
        # 如果要设置的值与当前属性值相同，则直接返回
        if item_value == self.__getattribute__("_%s" % item_name):
            return None

        # 构建 SQL 更新语句，更新指定项的值
        sql = "UPDATE %s SET %s = ? WHERE %s_id = %s" % (
                self.table_name,
                item_name,
                self.table_name,
                self.__getattribute__(self.table_id))
        # 执行 SQL 更新
        self.cursor.execute(sql, (item_value,))
        # 提交更新
        connection.commit()
        # 更新属性值
        self.__setattr__("_%s" % item_name, item_value)

    # 插入新数据
    def insert(self, **kargs):
        # 构建 SQL 插入语句
        sql = "INSERT INTO %s ("
        for k in kargs.keys():
            sql += k
            sql += ", "

        sql = sql[:][:-2]
        sql += ") VALUES ("

        for v in range(len(kargs.values())):
            sql += "?, "

        sql = sql[:][:-2]
        sql += ")"

        sql %= self.table_name

        # 执行 SQL 插入
        self.cursor.execute(sql, tuple(kargs.values()))
        # 提交插入
        connection.commit()

        # 查询最大的表ID值，并返回
        sql = "SELECT MAX(%s_id) FROM %s;" % (self.table_name, self.table_name)
        self.cursor.execute(sql)
        return self.cursor.fetchall()[0][0]
# 定义 UmitDB 类
class UmitDB(object):
    # 初始化方法，创建数据库游标
    def __init__(self):
        self.cursor = connection.cursor()

    # 创建数据库表
    def create_db(self):
        # SQL 删除表语句
        drop_string = "DROP TABLE scans;"
        try:
            # 执行删除表操作
            self.cursor.execute(drop_string)
        except Exception:
            # 出现异常时回滚操作
            connection.rollback()
        else:
            # 没有异常时提交操作
            connection.commit()

        # SQL 创建表语句
        creation_string = """CREATE TABLE scans (
            scans_id INTEGER PRIMARY KEY AUTOINCREMENT,
            scan_name TEXT,
            nmap_xml_output TEXT,
            digest TEXT,
            date INTEGER)"""
        # 执行创建表操作
        self.cursor.execute(creation_string)
        connection.commit()

    # 添加扫描记录
    def add_scan(self, **kargs):
        return Scans(**kargs)

    # 获取所有扫描记录的 ID
    def get_scans_ids(self):
        sql = "SELECT scans_id FROM scans;"
        self.cursor.execute(sql)
        # 返回所有扫描记录的 ID 列表
        return [sid[0] for sid in self.cursor.fetchall()]

    # 获取所有扫描记录
    def get_scans(self):
        # 获取所有扫描记录的 ID
        scans_ids = self.get_scans_ids()
        # 遍历所有扫描记录的 ID，返回对应的扫描记录对象
        for sid in scans_ids:
            yield Scans(scans_id=sid)

    # 清理数据库，删除指定时间之前的扫描记录
    def cleanup(self, save_time):
        log.debug(">>> Cleaning up data base.")
        log.debug(">>> Removing results older than %s seconds" % save_time)
        # 查询需要删除的扫描记录的 ID
        self.cursor.execute("SELECT scans_id FROM scans WHERE date < ?",
                (time() - save_time,))
        # 遍历需要删除的扫描记录的 ID，逐个删除
        for sid in [sid[0] for sid in self.cursor.fetchall()]:
            log.debug(">>> Removing results with scans_id %s" % sid)
            self.cursor.execute("DELETE FROM scans WHERE scans_id = ?",
                    (sid, ))
        # 提交删除操作
        connection.commit()
        log.debug(">>> Data base successfully cleaned up!")

# 定义 Scans 类，继承自 Table 类和 object 类
class Scans(Table, object):
    # 初始化方法，接受关键字参数
    def __init__(self, **kargs):
        # 调用父类 Table 的初始化方法，设置表名为 "scans"
        Table.__init__(self, "scans")
        # 如果关键字参数中包含 "scans_id"，则将其赋值给 self.scans_id
        if "scans_id" in kargs.keys():
            self.scans_id = kargs["scans_id"]
        else:
            # 如果没有 "scans_id"，则记录调试信息
            log.debug(">>> Creating new scan result entry at data base")
            # 定义字段列表
            fields = ["scan_name", "nmap_xml_output", "date"]
    
            # 遍历关键字参数的键
            for k in kargs.keys():
                # 如果键不在字段列表中，则抛出异常
                if k not in fields:
                    raise Exception(
                            "Wrong table field passed to creation method. "
                            "'%s'" % k)
    
            # 如果关键字参数中没有 "nmap_xml_output" 或者其值为空，则抛出异常
            if ("nmap_xml_output" not in kargs.keys() or
                    not kargs["nmap_xml_output"]):
                raise Exception("Can't save result without xml output")
    
            # 如果校验摘要不通过，则抛出异常
            if not self.verify_digest(
                    md5(kargs["nmap_xml_output"].encode("UTF-8")).hexdigest()):
                raise Exception("XML output registered already!")
    
            # 调用 insert 方法，将关键字参数插入数据库，并将返回的 ID 赋值给 self.scans_id
            self.scans_id = self.insert(**kargs)
    
    # 校验摘要方法，接受摘要作为参数
    def verify_digest(self, digest):
        # 执行 SQL 查询，查找具有指定摘要的记录
        self.cursor.execute(
                "SELECT scans_id FROM scans WHERE digest = ?", (digest, ))
        # 获取查询结果
        result = self.cursor.fetchall()
        # 如果有结果，则返回 False，否则返回 True
        if result:
            return False
        return True
    
    # 添加主机方法，接受关键字参数
    def add_host(self, **kargs):
        # 将 self.table_id 和 self.scans_id 添加到关键字参数中，然后调用 Hosts 类的初始化方法
        kargs.update({self.table_id: self.scans_id})
        return Hosts(**kargs)
    
    # 获取主机方法
    def get_hosts(self):
        # 构建 SQL 查询语句
        sql = "SELECT hosts_id FROM hosts WHERE scans_id= %s" % self.scans_id
        # 执行 SQL 查询
        self.cursor.execute(sql)
        # 获取查询结果
        result = self.cursor.fetchall()
        # 遍历结果，使用生成器返回 Hosts 对象
        for h in result:
            yield Hosts(hosts_id=h[0])
    
    # 获取扫描 ID 方法
    def get_scans_id(self):
        return self._scans_id
    
    # 设置扫描 ID 方法
    def set_scans_id(self, scans_id):
        # 如果传入的扫描 ID 与当前扫描 ID 不同，则更新当前扫描 ID
        if scans_id != self._scans_id:
            self._scans_id = scans_id
    
    # 获取扫描名称方法
    def get_scan_name(self):
        return self.get_item("scan_name")
    
    # 设置扫描名称方法
    def set_scan_name(self, scan_name):
        self.set_item("scan_name", scan_name)
    
    # 获取 nmap_xml_output 方法
    def get_nmap_xml_output(self):
        return self.get_item("nmap_xml_output")
    # 设置 nmap_xml_output 属性的数值，并计算其 MD5 哈希值
    def set_nmap_xml_output(self, nmap_xml_output):
        # 调用 set_item 方法设置 nmap_xml_output 属性
        self.set_item("nmap_xml_output", nmap_xml_output)
        # 计算 nmap_xml_output 的 MD5 哈希值，并调用 set_item 方法设置 digest 属性
        self.set_item("digest", md5(nmap_xml_output.encode("UTF-8")).hexdigest())
    
    # 获取 date 属性的数值
    def get_date(self):
        return self.get_item("date")
    
    # 设置 date 属性的数值
    def set_date(self, date):
        # 调用 set_item 方法设置 date 属性
        self.set_item("date", date)
    
    # 定义 scans_id 属性，并使用 property 方法将其与 get_scans_id 和 set_scans_id 方法关联
    scans_id = property(get_scans_id, set_scans_id)
    # 定义 scan_name 属性，并使用 property 方法将其与 get_scan_name 和 set_scan_name 方法关联
    scan_name = property(get_scan_name, set_scan_name)
    # 定义 nmap_xml_output 属性，并使用 property 方法将其与 get_nmap_xml_output 和 set_nmap_xml_output 方法关联
    nmap_xml_output = property(get_nmap_xml_output, set_nmap_xml_output)
    # 定义 date 属性，并使用 property 方法将其与 get_date 和 set_date 方法关联
    date = property(get_date, set_date)
    
    # 初始化 _scans_id、_scan_name、_nmap_xml_output、_date 四个属性
    _scans_id = None
    _scan_name = None
    _nmap_xml_output = None
    _date = None
# 验证数据库是否存在以及是否具有所需的表，如果有问题，则重新创建表
def verify_db():
    # 创建游标对象
    cursor = connection.cursor()
    try:
        # 执行 SQL 查询，检查是否存在符合条件的数据
        cursor.execute("SELECT scans_id FROM scans WHERE date = 0")
    except sqlite3.OperationalError:
        # 如果出现异常，创建新的数据库对象，并重新创建表
        u = UmitDB()
        u.create_db()
# 调用验证数据库的函数
verify_db()

if __name__ == "__main__":
    from pprint import pprint

    # 创建数据库对象
    u = UmitDB()

    # 打印信息，创建数据表
    #u.create_db()

    # 打印信息，创建新的扫描
    #s = u.add_scan(scan_name="Fake scan", nmap_xml_output="", date="007")

    # 创建扫描对象
    #s = Scans(scans_id=2)
    # 打印扫描对象的属性
    #print s.scans_id
    #print s.scan_name
    #print s.nmap_xml_output
    #print s.date

    # 执行 SQL 查询，获取所有扫描数据
    sql = "SELECT * FROM scans;"
    u.cursor.execute(sql)
    # 打印扫描数据
    print("Scans:", end=' ')
    pprint(u.cursor.fetchall())
```