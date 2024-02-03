# `nmap\zenmap\radialnet\util\drawing.py`

```cpp
# 设置文件编码为 UTF-8

# 重要提示：NMAP 许可证条款
# Nmap 安全扫描器由 Nmap 软件有限责任公司（“Nmap 项目”）（C）1996-2023 年开发。Nmap 也是 Nmap 项目的注册商标。
# 本程序根据 Nmap 公共源代码许可证（NPSL）的条款分发。适用于特定 Nmap 版本或源代码控制修订版的确切许可证文本包含在该版本的 Nmap 或源代码控制修订版的 LICENSE 文件中。有关更多 Nmap 版权/法律信息，请访问 https://nmap.org/book/man-legal.html，有关 NPSL 许可证本身的更多信息，请访问 https://nmap.org/npsl/。本标题总结了 Nmap 许可证的一些关键点，但不能替代实际的许可证文本。
# Nmap 通常允许最终用户自行下载和使用，包括商业用途。可从 https://nmap.org 获取。
# Nmap 许可证通常禁止公司在商业产品中使用和重新分发 Nmap，但我们销售一个特殊的 Nmap OEM 版本，具有更宽松的许可证和专门用于此目的的特殊功能。请参阅 https://nmap.org/oem/
# 如果您收到了书面的 Nmap 许可协议或合同，其中规定了与这些条款（如 Nmap OEM 许可证）不同的条款，您可以选择根据那些条款使用和重新分发 Nmap。
# 官方的 Nmap Windows 构建包括 Npcap 软件（https://npcap.com）用于数据包捕获和传输。它受到禁止未经特殊许可证重新分发的单独许可证条款的约束。因此，官方的 Nmap Windows 构建可能不得未经特殊许可证（如 Nmap OEM 许可证）重新分发。
# 我们提供此软件的源代码，因为我们相信用户有权根据其需求进行修改和定制。
# 定义一个函数，将 Cairo 颜色转换为 GDK 颜色
def cairo_to_gdk_color(color):
    """
    """
    # 创建一个与输入颜色长度相同的新颜色列表
    new_color = list(range(len(color)))

    # 遍历输入颜色，将每个颜色值乘以 65535 转换为整数，并存入新颜色列表
    for i in range(len(color)):
        new_color[i] = int(color[i] * 65535)

    # 返回转换后的新颜色列表
    return new_color
```