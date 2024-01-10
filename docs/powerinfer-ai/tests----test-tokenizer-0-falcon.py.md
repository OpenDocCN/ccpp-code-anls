# `PowerInfer\tests\test-tokenizer-0-falcon.py`

```
# 导入所需的库
import os
import sys
import argparse
from transformers import AutoTokenizer

# 创建命令行参数解析器
parser = argparse.ArgumentParser()
parser.add_argument("dir_tokenizer", help="directory containing 'tokenizer.model' file")
parser.add_argument("--fname-tok",   help="path to a text file to tokenize")
args = parser.parse_args()

# 获取命令行参数
dir_tokenizer = args.dir_tokenizer

# 从指定目录加载预训练的tokenizer
tokenizer = AutoTokenizer.from_pretrained(dir_tokenizer)

# 定义测试用例
tests = [
        "",
        " ",
        "  ",
        "   ",
        "\t",
        "\n",
        "\t\n",
        "Hello world",
        " Hello world",
        # ... 省略部分测试用例 ...
        "world"
    ]

# 对每个测试用例进行编码和解码
for text in tests:
    print('text: ', text)
    print(tokenizer.encode(text))
    print(tokenizer.decode(tokenizer.encode(text)))

# 打印 C++ 测试用例的编码结果
print("\n\ntests for C++:\n")
for text in tests:
    res = tokenizer.encode(text)

    k = text.replace('\n', '\\n')
    k = k.replace('\t', '\\t')
    k = '"' + k + '"'
    print("{ %-24s, { " % k, end='')
    for x in res:
        print("%7d," % x, end='')
    print(" }, },")

# 对指定的文件进行分词处理
fname_tok = args.fname_tok
if fname_tok:
    print('tokenizing file: ', fname_tok)
    fname_out = fname_tok + '.tok'
    # 使用指定的文件名以只读方式打开文件，并指定编码为utf-8
    with open(fname_tok, 'r', encoding='utf-8') as f:
        # 读取文件的所有行
        lines = f.readlines()
        # 将所有行连接成一个字符串
        s = ''.join(lines)
        # 使用tokenizer对字符串进行编码
        res = tokenizer.encode(s)
        # 以指定文件名以只写方式打开文件，并指定编码为utf-8
        with open(fname_out, 'w', encoding='utf-8') as f:
            # 遍历编码结果，并将结果写入文件
            for x in res:
                f.write(str(x) + ' \'' + tokenizer.decode(x) + '\'\n')
        # 打印编码结果的长度
        print('len(res): ', len(res))
        # 打印文件的行数
        print('len(lines): ', len(lines))
    # 打印结果文件的文件名
    print('results written to: ', fname_out)
```