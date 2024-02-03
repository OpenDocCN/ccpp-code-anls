# `nmap\zenmap\radialnet\core\Coordinate.py`

```cpp
# 设置文件编码为 UTF-8

# ***********************IMPORTANT NMAP LICENSE TERMS************************
# *
# * Nmap安全扫描器由Nmap软件有限责任公司（“Nmap项目”）（C）1996-2023年拥有。Nmap也是Nmap项目的注册商标。
# *
# * 本程序根据Nmap公共源代码许可证（NPSL）的条款分发。适用于特定Nmap版本或源代码控制修订版的确切许可证文本包含在该版本的Nmap或源代码控制修订版的LICENSE文件中。有关更多Nmap版权/法律信息，请访问https://nmap.org/book/man-legal.html，有关NPSL许可证本身的更多信息，请访问https://nmap.org/npsl/。此标题总结了Nmap许可证的一些关键点，但不能替代实际许可证文本。
# *
# * 一般情况下，用户可以免费下载和使用Nmap，包括商业用途。可从https://nmap.org获取。
# *
# * Nmap许可证一般禁止公司在商业产品中使用和重新分发Nmap，但我们销售具有更宽松许可证和特殊功能的特殊Nmap OEM版本。请参阅https://nmap.org/oem/
# *
# * 如果您收到了书面的Nmap许可协议或合同，其中规定了与这些条款（例如Nmap OEM许可证）不同的条款，您可以选择根据那些条款使用和重新分发Nmap。
# *
# * 官方Nmap Windows版本包括Npcap软件（https://npcap.com）用于数据包捕获和传输。它受到禁止未经特别许可就重新分发的单独许可条款的约束。因此，官方Nmap Windows版本可能不得未经特别许可（例如Nmap OEM许可证）重新分发。
# *
# * 我们提供此软件的源代码，因为我们相信用户有权根据自己的需求修改和重新分发它。但是，我们希望用户尊重我们的劳动成果，遵守我们的许可证条款。
# 导入数学库
import math

# 定义极坐标类
class PolarCoordinate:
    """
    Class to implement a polar coordinate object
    """

    # 构造方法，初始化极坐标的半径和角度
    def __init__(self, r=0, t=0):
        """
        Constructor method of PolarCoordinate class
        @type  r: number
        @param r: The radius of coordinate
        @type  t: number
        @param t: The angle (theta) of coordinate in radians
        """

        self.__r = r
        """Radius of polar coordinate"""
        self.__t = t
        """Angle (theta) of polar coordinate in radians"""

    # 获取极坐标的角度（弧度转换为角度）
    def get_theta(self):
        """
        """
        return math.degrees(self.__t)

    # 获取极坐标的半径
    def get_radius(self):
        """
        """
        return self.__r
    # 设置极坐标的角度，将角度转换为弧度
    def set_theta(self, t):
        self.__t = math.radians(t)

    # 设置极坐标的半径
    def set_radius(self, r):
        self.__r = r

    # 获取极坐标
    def get_coordinate(self):
        """
        返回极坐标
        @rtype: tuple
        @return: 极坐标 (r, t)
        """
        return (self.__r, math.degrees(self.__t))

    # 设置极坐标
    def set_coordinate(self, r, t):
        """
        设置极坐标
        @type  r: number
        @param r: 极坐标的半径
        @type  t: number
        @param t: 极坐标的角度
        """
        self.__r = r
        self.__t = math.radians(t)

    # 将极坐标转换为直角坐标
    def to_cartesian(self):
        """
        将极坐标转换为直角坐标
        @rtype: tuple
        @return: 直角坐标 (x, y)
        """
        x = self.__r * math.cos(self.__t)
        y = self.__r * math.sin(self.__t)

        return (x, y)
class CartesianCoordinate:
    """
    Class to implement a cartesian coordinate object
    """
    def __init__(self, x=0, y=0):
        """
        Constructor method of CartesianCoordinate class
        @type  x: number
        @param x: The x component of coordinate
        @type  y: number
        @param y: The y component of coordinate
        """
        self.__x = x
        """X component of cartesian coordinate"""
        self.__y = y
        """Y component of cartesian coordinate"""

    def get_coordinate(self):
        """
        Get cartesian coordinate
        @rtype: tuple
        @return: Cartesian coordinates (x, y)
        """
        return (self.__x, self.__y)

    def set_coordinate(self, x, y):
        """
        Set cartesian coordinate
        @type  x: number
        @param x: The x component of coordinate
        @type  y: number
        @param y: The y component of coordinate
        """
        self.__x = x
        self.__y = y

    def to_polar(self):
        """
        Convert cartesian in polar coordinate
        @rtype: tuple
        @return: polar coordinates (r, t)
        """
        r = math.sqrt(self.__x ** 2 + self.__y ** 2)

        if self.__x > 0:
            if self.__y >= 0:
                t = math.atan(self.__y / self.__x)
            else:
                t = math.atan(self.__y / self.__x) + 2 * math.pi
        elif self.__x < 0:
            t = math.atan(self.__y / self.__x) + math.pi
        elif self.__x == 0:
            if self.__y == 0:
                t = 0
            if self.__y > 0:
                t = math.pi / 2
            else:
                t = -math.pi / 2
        return (r, t)


if __name__ == "__main__":

    # Testing application

    polar = PolarCoordinate(1, math.pi)
    cartesian = CartesianCoordinate(-1, 0)

    print(polar.to_cartesian())
    print(cartesian.to_polar())
```