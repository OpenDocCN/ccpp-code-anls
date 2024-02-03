# `PowerInfer\examples\jeopardy\graph.py`

```cpp
#!/usr/bin/env python3
# 指定脚本解释器为 Python 3

import matplotlib.pyplot as plt
# 导入 matplotlib 库中的 pyplot 模块，并重命名为 plt
import os
# 导入 os 模块

import csv
# 导入 csv 模块

labels = []
# 初始化空列表 labels，用于存储标签数据
numbers = []
# 初始化空列表 numbers，用于存储数字数据
numEntries = 1
# 初始化变量 numEntries 为 1，用于记录条目数量

rows = []
# 初始化空列表 rows，用于存储 CSV 文件中的行数据

def bar_chart(numbers, labels, pos):
    # 定义函数 bar_chart，用于生成条形图
    plt.bar(pos, numbers, color='blue')
    # 生成条形图，设置位置、高度和颜色
    plt.xticks(ticks=pos, labels=labels)
    # 设置 x 轴刻度的位置和标签
    plt.title("Jeopardy Results by Model")
    # 设置图表标题
    plt.xlabel("Model")
    # 设置 x 轴标签
    plt.ylabel("Questions Correct")
    # 设置 y 轴标签
    plt.show()
    # 显示图表

def calculatecorrect():
    # 定义函数 calculatecorrect，用于计算正确答案数量
    directory = os.fsencode("./examples/jeopardy/results/")
    # 获取指定目录的字节编码形式
    csv_reader = csv.reader(open("./examples/jeopardy/qasheet.csv", 'rt'), delimiter=',')
    # 读取 CSV 文件
    for row in csv_reader:
        # 遍历 CSV 文件的每一行
        global rows
        # 声明使用全局变量 rows
        rows.append(row)
        # 将每行数据添加到列表 rows 中
    for listing in os.listdir(directory):
        # 遍历指定目录中的文件列表
        filename = os.fsdecode(listing)
        # 将字节编码的文件名解码为字符串
        if filename.endswith(".txt"):
            # 如果文件名以 .txt 结尾
            file = open("./examples/jeopardy/results/" + filename, "rt")
            # 打开文件
            global labels
            # 声明使用全局变量 labels
            global numEntries
            # 声明使用全局变量 numEntries
            global numbers
            # 声明使用全局变量 numbers
            labels.append(filename[:-4])
            # 将文件名添加到 labels 列表中
            numEntries += 1
            # 条目数量加一
            i = 1
            # 初始化变量 i 为 1
            totalcorrect = 0
            # 初始化变量 totalcorrect 为 0
            for line in file.readlines():
                # 遍历文件的每一行
                if line.strip() != "------":
                    # 如果行内容不是 "------"
                    print(line)
                    # 打印行内容
                else:
                    print("Correct answer: " + rows[i][2] + "\n")
                    # 打印正确答案
                    i += 1
                    # i 加一
                    print("Did the AI get the question right? (y/n)")
                    # 提示用户是否 AI 回答正确
                    if input() == "y":
                        # 如果用户输入为 "y"
                        totalcorrect += 1
                        # totalcorrect 加一
            numbers.append(totalcorrect)
            # 将 totalcorrect 添加到 numbers 列表中

if __name__ == '__main__':
    # 如果脚本作为主程序执行
    calculatecorrect()
    # 调用 calculatecorrect 函数
    pos = list(range(numEntries))
    # 生成位置列表
    labels.append("Human")
    # 将 "Human" 添加到 labels 列表中
    numbers.append(48.11)
    # 将 48.11 添加到 numbers 列表中
    bar_chart(numbers, labels, pos)
    # 调用 bar_chart 函数，生成条形图
    print(labels)
    # 打印 labels 列表
    print(numbers)
    # 打印 numbers 列表
```