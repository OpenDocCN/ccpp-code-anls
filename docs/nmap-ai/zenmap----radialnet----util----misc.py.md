# `nmap\zenmap\radialnet\util\misc.py`

```cpp
# 设置文件编码为 UTF-8

# 重要提示：NMAP 许可证条款
# Nmap 安全扫描器由 Nmap 软件有限责任公司（“Nmap 项目”）（C）1996-2023 年开发。Nmap 也是 Nmap 项目的注册商标。
# 本程序根据 Nmap 公共源代码许可证（NPSL）的条款分发。适用于特定 Nmap 版本或源代码控制修订版的确切许可证文本包含在该版本的 Nmap 或源代码控制修订版的 LICENSE 文件中。有关更多 Nmap 版权/法律信息，请访问 https://nmap.org/book/man-legal.html，有关 NPSL 许可证本身的更多信息，请访问 https://nmap.org/npsl/。本标题总结了 Nmap 许可证的一些关键点，但不能替代实际的许可证文本。
# Nmap 通常允许最终用户自由下载和使用，包括商业用途。可从 https://nmap.org 获取。
# Nmap 许可证通常禁止公司在商业产品中使用和重新分发 Nmap，但我们销售具有更宽松许可证和特殊功能的特殊 Nmap OEM 版本。请参阅 https://nmap.org/oem/
# 如果您收到了书面的 Nmap 许可协议或合同，其中规定了与这些条款（例如 Nmap OEM 许可证）不同的条款，您可以选择根据那些条款使用和重新分发 Nmap。
# 官方的 Nmap Windows 构建包括 Npcap 软件（https://npcap.com）用于数据包捕获和传输。它受到禁止未经特殊许可证重新分发的单独许可证条款的约束。因此，官方的 Nmap Windows 构建可能不得未经特殊许可证（例如 Nmap OEM 许可证）重新分发。
# 我们提供此软件的源代码，因为我们相信用户有权根据自己的需求修改和重新分发它。但是，我们希望用户尊重我们的劳动成果，并遵守我们的许可证条款。
# 导入所需的模块
from radialnet.core.Coordinate import CartesianCoordinate
from radialnet.util.geometry import normalize_angle
import math

# 定义一个函数，用于比较两个 IPv4 地址的大小
def ipv4_compare(ip1, ip2):
    # 将 IPv4 地址按照点号分隔，并转换成整数列表
    ip1 = [int(i) for i in ip1.split('.')]
    ip2 = [int(i) for i in ip2.split('.')]

    # 逐个比较 IPv4 地址的每一部分
    for i in range(4):
        # 如果对应部分不相等
        if ip1[i] != ip2[i]:
            # 返回-1或1，表示ip1小于或大于ip2
            if ip1[i] < ip2[i]:
                return -1
            else:
                return 1
    # 如果所有部分都相等，则返回0
    return 0

# 定义一个函数，用于交换列表中的两个元素
def swap(list, a, b):
    # 交换列表中索引为a和b的元素
    list[a], list[b] = list[b], list[a]

# 定义一个函数，用于对子节点进行排序
def sort_children(children, father):
    # 如果子节点数量小于2，则无需排序，直接返回
    if len(children) < 2:
        return children

    # 创建角度参考
    f_x, f_y = father.get_cartesian_coordinate()
    # 遍历子节点列表
    for child in children:
        # 获取子节点的笛卡尔坐标
        c_x, c_y = child.get_cartesian_coordinate()
        # 计算子节点相对于父节点的极坐标角度
        _, angle = CartesianCoordinate(c_x - f_x, c_y - f_y).to_polar()
        # 设置子节点的绘制信息，包括相对于父节点的角度
        child.set_draw_info({'angle_from_father': math.degrees(angle)})
    # 返回按角度排序后的子节点列表
    return sort_children_by_angle(children)
# 根据角度对子元素进行排序
def sort_children_by_angle(children):
    # 将子元素列表转换为可变列表
    vector = list(children)
    # 使用 lambda 函数对子元素列表进行排序，排序依据是子元素相对于父元素的角度
    vector.sort(
            key=lambda c: normalize_angle(
                c.get_draw_info('angle_from_father')))
    # 返回排序后的子元素列表
    return vector
```