# `nmap\zenmap\radialnet\core\Interpolation.py`

```
# 设置文件编码为 UTF-8

# ***********************IMPORTANT NMAP LICENSE TERMS************************
# *
# * Nmap安全扫描器由Nmap软件有限责任公司（“Nmap项目”）（C）1996-2023年拥有。Nmap也是Nmap项目的注册商标。
# *
# * 本程序根据Nmap公共源代码许可证（NPSL）的条款分发。适用于特定Nmap版本或源代码控制修订版的确切许可证文本包含在该版本的Nmap或源代码控制修订版的LICENSE文件中。有关更多Nmap版权/法律信息，请访问https://nmap.org/book/man-legal.html，有关NPSL许可证本身的更多信息，请访问https://nmap.org/npsl/。此标题总结了Nmap许可证的一些关键点，但不能替代实际许可证文本。
# *
# * 一般情况下，用户可以免费下载和使用Nmap，包括商业用途。它可以从https://nmap.org获得。
# *
# * Nmap许可证一般禁止公司在商业产品中使用和重新分发Nmap，但我们销售具有更宽松许可证和特殊功能的特殊Nmap OEM版本。请参阅https://nmap.org/oem/
# *
# * 如果您收到了书面的Nmap许可协议或合同，其中规定了与这些条款（例如Nmap OEM许可证）不同的条款，您可以选择根据那些条款使用和重新分发Nmap。
# *
# * 官方的Nmap Windows版本包括Npcap软件（https://npcap.com）用于数据包捕获和传输。它受到禁止未经特殊许可重新分发的单独许可条款的约束。因此，官方的Nmap Windows版本可能不得在没有特殊许可（例如Nmap OEM许可证）的情况下重新分发。
# *
# * 我们提供此软件的源代码，因为我们相信用户有权根据自己的需求修改和重新分发它。但是，我们希望用户尊重我们的劳动成果，遵守我们的许可证条款。
# 定义一个类Linear2DInterpolator，实现二维线性插值
class Linear2DInterpolator:
    """
    实现一个二维线性插值器
    """

    def __init__(self):
        """
        Linear2DInterpolator类的构造方法
        """
        self.__start_point = (0, 0)
        """插值的起始点"""
        self.__final_point = (0, 0)
        """插值的结束点"""
        self.__interpolated_points = []
        """插值后的点的向量"""
    # 设置初始坐标点
    def set_start_point(self, a, b):
        # 将传入的参数组成元组，作为初始坐标点
        self.__start_point = (a, b)

    # 设置最终坐标点
    def set_final_point(self, a, b):
        # 将传入的参数组成元组，作为最终坐标点
        self.__final_point = (a, b)

    # 获取加权插值点
    def get_weighed_points(self, number_of_pass, pass_vector):
        # 获取初始坐标点和最终坐标点
        (ai, bi) = self.__start_point
        (af, bf) = self.__final_point

        # 计算a和b的转换因子
        a_conversion_factor = float(af - ai) / sum(pass_vector)
        b_conversion_factor = float(bf - bi) / sum(pass_vector)

        a_pass = 0
        b_pass = 0

        # 初始化插值点列表
        self.__interpolated_points = list(range(number_of_pass))

        # 遍历插值点
        for i in range(0, number_of_pass):
            # 计算每个插值点的坐标
            a_pass += pass_vector[i] * a_conversion_factor
            b_pass += pass_vector[i] * b_conversion_factor
            self.__interpolated_points[i] = (ai + a_pass, bi + b_pass)

        # 返回插值点列表
        return self.__interpolated_points
    # 定义一个方法，用于返回初始坐标和最终坐标之间指定大小的坐标向量
    def get_points(self, number_of_pass):
        """
        Return the vector of coordinates between the initial and final
        coordinates with the specified size
        @type  number_of_pass: number
        @param number_of_pass: The number of pass of interpolation
        @rtype: list
        @return: A list of tuples with interpolated points
        """
        # 获取初始坐标和最终坐标
        (ai, bi) = self.__start_point
        (af, bf) = self.__final_point

        # 计算每次插值的步长
        a_pass = float(af - ai) / number_of_pass
        b_pass = float(bf - bi) / number_of_pass

        # 初始化插值点列表
        self.__interpolated_points = list(range(number_of_pass))

        # 进行插值计算
        for i in range(1, number_of_pass + 1):
            self.__interpolated_points[i - 1] = (ai + a_pass * i,
                                                 bi + b_pass * i)

        # 返回插值点列表
        return self.__interpolated_points
# 如果当前模块被直接执行，则执行以下代码
if __name__ == "__main__":

    # 测试应用程序

    # 创建一个二维线性插值器对象
    i = Linear2DInterpolator()

    # 设置起始点坐标为 (0, 0)
    i.set_start_point(0, 0)
    # 设置终点坐标为 (1, 1)
    i.set_final_point(1, 1)

    # 打印插值器生成的10个点的数量和坐标
    print(len(i.get_points(10)), i.get_points(10))
```