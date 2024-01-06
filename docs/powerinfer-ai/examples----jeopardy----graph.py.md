# `PowerInfer\examples\jeopardy\graph.py`

```
#!/usr/bin/env python3
# 指定使用 Python3 解释器

import matplotlib.pyplot as plt
# 导入 matplotlib 库中的 pyplot 模块，用于绘制图表
import os
# 导入 os 模块，用于与操作系统交互
import csv
# 导入 csv 模块，用于读写 CSV 文件

labels = []
# 初始化一个空列表，用于存储图表的标签
numbers = []
# 初始化一个空列表，用于存储图表的数据
numEntries = 1
# 初始化一个变量，用于记录条目数量

rows = []
# 初始化一个空列表，用于存储 CSV 文件的行数据

def bar_chart(numbers, labels, pos):
    # 定义一个函数，用于绘制柱状图
    plt.bar(pos, numbers, color='blue')
    # 绘制柱状图，设置颜色为蓝色
    plt.xticks(ticks=pos, labels=labels)
    # 设置 X 轴刻度和标签
    plt.title("Jeopardy Results by Model")
    # 设置图表标题
    plt.xlabel("Model")
    # 设置 X 轴标签
    plt.ylabel("Questions Correct")
    # 设置 Y 轴标签
    plt.show()
    # 显示图表
# 计算正确答案的数量
def calculatecorrect():
    # 获取指定目录下的文件夹
    directory = os.fsencode("./examples/jeopardy/results/")
    # 读取 CSV 文件
    csv_reader = csv.reader(open("./examples/jeopardy/qasheet.csv", 'rt'), delimiter=',')
    # 将 CSV 文件的每一行添加到全局变量 rows 中
    for row in csv_reader:
        global rows
        rows.append(row)
    # 遍历指定目录下的文件
    for listing in os.listdir(directory):
        filename = os.fsdecode(listing)
        # 如果文件是以 .txt 结尾
        if filename.endswith(".txt"):
            # 打开文件
            file = open("./examples/jeopardy/results/" + filename, "rt")
            global labels
            global numEntries
            global numbers
            # 将文件名添加到 labels 列表中
            labels.append(filename[:-4])
            # 增加 numEntries 计数
            numEntries += 1
            i = 1
            totalcorrect = 0
            # 逐行读取文件内容
            for line in file.readlines():
                # 如果行内容不是 "------"
# 打印当前行的内容
print(line)
# 如果当前行不是问题，则打印正确答案，并将索引加一
else:
    print("Correct answer: " + rows[i][2] + "\n")
    i += 1
    # 询问AI是否回答正确，如果是则总正确数加一
    print("Did the AI get the question right? (y/n)")
    if input() == "y":
        totalcorrect += 1
# 将总正确数加入列表中
numbers.append(totalcorrect)

# 如果是主程序入口
if __name__ == '__main__':
    # 调用calculatecorrect函数
    calculatecorrect()
    # 生成一个包含numEntries个元素的列表
    pos = list(range(numEntries))
    # 将"Human"加入标签列表中
    labels.append("Human")
    # 将48.11加入数字列表中
    numbers.append(48.11)
    # 调用bar_chart函数，生成条形图
    bar_chart(numbers, labels, pos)
    # 打印标签列表
    print(labels)
    # 打印数字列表
    print(numbers)
```