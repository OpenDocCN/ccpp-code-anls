# `nmap\zenmap\zenmapCore\DelayedObject.py`

```
# 指定脚本解释器为 Python 3
#!/usr/bin/env python3

# 重要提示：NMAP 许可证条款
# Nmap 安全扫描器由 Nmap Software LLC（“Nmap 项目”）拥有版权，1996-2023 年。
# Nmap 也是 Nmap 项目的注册商标。
# 本程序根据 Nmap 公共源代码许可证（NPSL）的条款分发。适用于特定 Nmap 版本或源代码控制修订版本的确切许可证文本包含在该版本的 Nmap 或源代码控制修订版本中分发的 LICENSE 文件中。
# 更多关于 Nmap 版权/法律信息，请访问 https://nmap.org/book/man-legal.html，有关 NPSL 许可证本身的更多信息，请访问 https://nmap.org/npsl/。本标题总结了 Nmap 许可证的一些关键点，但不能替代实际的许可证文本。
# Nmap 通常允许最终用户免费下载和使用，包括商业用途。可从 https://nmap.org 获取。
# Nmap 许可证通常禁止公司在商业产品中使用和重新分发 Nmap，但我们销售一个特殊的 Nmap OEM 版本，具有更宽松的许可证和专门用于此目的的特殊功能。请参阅 https://nmap.org/oem/
# 如果您收到了书面的 Nmap 许可协议或合同，其中规定了与这些条款（如 Nmap OEM 许可证）不同的条款，您可以选择根据那些条款使用和重新分发 Nmap。
# 官方的 Nmap Windows 构建包括 Npcap 软件（https://npcap.com）用于数据包捕获和传输。它受到单独的许可证条款的约束，禁止未经特殊许可证的重新分发。因此，官方的 Nmap Windows 构建可能不得未经特殊许可证（如 Nmap OEM 许可证）重新分发。
# 我们提供此软件的源代码，因为我们认为用户在运行程序之前有权知道程序将要做什么。
# ***********************IMPORTANT NMAP LICENSE TERMS************************
# 定义一个延迟加载对象的类
class DelayedObject(object):
    # 初始化方法，接受类名、位置参数和关键字参数
    def __init__(self, klass, *args, **kwargs):
        # 设置对象的属性 "klass" 为传入的类名
        object.__setattr__(self, "klass", klass)
        # 设置对象的属性 "args" 为传入的位置参数
        object.__setattr__(self, "args", args)
        # 设置对象的属性 "kwargs" 为传入的关键字参数
        object.__setattr__(self, "kwargs", kwargs)

    # 设置对象属性的方法
    def __setattr__(self, name, value):
        # 创建一个新的类实例，传入之前的位置参数和关键字参数
        self = object.__getattribute__(self, "klass")(
                *object.__getattribute__(self, "args"),
                **object.__getattribute__(self, "kwargs")
                )
        # 设置新实例的属性为传入的值
        setattr(self, name, value)

    # 获取对象属性的方法
    def __getattribute__(self, name):
        # 创建一个新的类实例，传入之前的位置参数和关键字参数
        self = object.__getattribute__(self, "klass")(
                *object.__getattribute__(self, "args"),
                **object.__getattribute__(self, "kwargs")
                )
        # 返回新实例的属性值
        return getattr(self, name)
```