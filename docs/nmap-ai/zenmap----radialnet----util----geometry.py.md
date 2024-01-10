# `nmap\zenmap\radialnet\util\geometry.py`

```
# 设置文件编码为 UTF-8

# ***********************IMPORTANT NMAP LICENSE TERMS************************
# *
# * Nmap安全扫描器由Nmap软件有限责任公司（“Nmap项目”）（C）1996-2023年拥有。Nmap也是Nmap项目的注册商标。
# *
# * 本程序根据Nmap公共源代码许可证（NPSL）的条款分发。适用于特定Nmap版本或源代码控制修订版的确切许可证文本包含在该版本的Nmap或源代码控制修订版的LICENSE文件中。有关更多Nmap版权/法律信息，请访问https://nmap.org/book/man-legal.html，有关NPSL许可证本身的更多信息，请访问https://nmap.org/npsl/。此标题总结了Nmap许可证的一些关键点，但不能替代实际许可证文本。
# *
# * 通常情况下，用户可以免费下载和使用Nmap，包括商业用途。它可以从https://nmap.org获得。
# *
# * Nmap许可证通常禁止公司在商业产品中使用和重新分发Nmap，但我们销售具有更宽松许可证和特殊功能的特殊Nmap OEM版本。请参阅https://nmap.org/oem/
# *
# * 如果您收到了书面的Nmap许可协议或合同，其中规定了与这些条款（例如Nmap OEM许可证）不同的条款，您可以选择根据那些条款使用和重新分发Nmap。
# *
# * 官方的Nmap Windows版本包括Npcap软件（https://npcap.com）用于数据包捕获和传输。它受到禁止未经特殊许可证重新分发的单独许可证条款的约束。因此，官方的Nmap Windows版本可能不得未经特殊许可证（例如Nmap OEM许可证）重新分发。
# *
# * 我们提供此软件的源代码，因为我们相信用户有权根据自己的需求修改和重新分发它。但是，我们希望用户尊重我们的劳动成果，遵守我们的许可证条款。
# 导入 math 模块
import math

# 判断点是否在正方形内
def is_in_square(point, half_side, center=(0, 0)):
    """
    判断给定点是否在以 center 为中心、边长为 half_side 的正方形内
    """
    x, y = point
    a, b = center

    if a + half_side >= x >= a - half_side:
        if b + half_side >= y >= b - half_side:
            return True

    return False

# 判断点是否在圆内
def is_in_circle(point, radius=1, center=(0, 0)):
    """
    判断给定点是否在以 center 为中心、半径为 radius 的圆内
    """
    x, y = point
    a, b = center

    if ((x - a) ** 2 + (y - b) ** 2) <= (radius ** 2):
        return True

    return False

# 对点进行反正切缩放
def atan_scale(point, scale_ceil):
    """
    对给定点进行反正切缩放，返回结果
    """
    new_point = float(10.0 * point / scale_ceil) - 5
    return math.atan(abs(new_point))

# 规范化角度
def normalize_angle(angle):
    """
    规范化给定角度，返回结果
    """
    new_angle = 360.0 * (float(angle / 360) - int(angle / 360))
    # 如果新角度小于0，则返回360加上新角度的值
    if new_angle < 0:
        return 360 + new_angle
    
    # 返回新角度的值
    return new_angle
# 判断角度 c 是否在角度 a 和 b 之间
def is_between_angles(a, b, c):
    # 规范化角度 a、b、c，使其在 0 到 360 度之间
    a = normalize_angle(a)
    b = normalize_angle(b)
    c = normalize_angle(c)

    # 如果 a 大于 b
    if a > b:
        # 如果 c 在 a 到 360 之间，或者在 0 到 b 之间，则返回 True
        if c >= a and c <= 360 or c <= b:
            return True
        # 否则返回 False
        return False
    # 如果 a 不大于 b
    else:
        # 如果 c 在 a 到 b 之间，则返回 True
        if c >= a and c <= b:
            return True
        # 否则返回 False
        return False


# 计算两个角度之间的距离
def angle_distance(a, b):
    # 计算两个角度的差的绝对值
    distance = abs(normalize_angle(a) - normalize_angle(b))
    # 如果差大于 180 度，则返回 360 减去差
    if distance > 180:
        return 360 - distance
    # 否则返回差
    return distance


# 计算两个角度之间的最短路径
def calculate_short_path(iangle, fangle):
    # 如果初始角度减去最终角度大于 180 度，则最终角度加上 360 度
    if iangle - fangle > 180:
        fangle += 360
    # 如果初始角度减去最终角度小于 -180 度，则最终角度减去 360 度
    if iangle - fangle < -180:
        fangle -= 360
    # 返回修正后的初始角度和最终角度
    return iangle, fangle


# 根据物体距离和大小计算角度
def angle_from_object(distance, size):
    # 使用反正切函数计算角度，并将弧度转换为角度
    return math.degrees(math.atan2(size / 2.0, distance))
```