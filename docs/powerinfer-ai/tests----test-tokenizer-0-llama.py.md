# `PowerInfer\tests\test-tokenizer-0-llama.py`

```
# 导入所需的模块
import os
import sys
import argparse
# 从 sentencepiece 模块中导入 SentencePieceProcessor 类
from sentencepiece import SentencePieceProcessor

# 创建解析器对象
parser = argparse.ArgumentParser()
# 添加命令行参数，指定目录包含 'tokenizer.model' 文件
parser.add_argument("dir_tokenizer", help="directory containing 'tokenizer.model' file")
# 添加可选参数，指定要进行标记化的文本文件的路径
parser.add_argument("--fname-tok",   help="path to a text file to tokenize")
# 解析命令行参数
args = parser.parse_args()

# 获取目录参数的值
dir_tokenizer = args.dir_tokenizer

# 创建 SentencePieceProcessor 对象，加载指定目录下的 'tokenizer.model' 文件
tokenizer = SentencePieceProcessor(dir_tokenizer + '/tokenizer.model')

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
        # ... 其他测试用例 ...
    ]

# 遍历测试用例
for text in tests:
    # 打印当前测试文本
    print('text: ', text)
    print('\nwith bos:')
    # 对文本进行编码，添加 bos 标记
    print(tokenizer.encode(text, add_bos=True))
    # 对编码结果进行解码
    print(tokenizer.decode(tokenizer.encode(text, add_bos=True)))
    print('\nwithout bos:')
    # 对文本进行编码，不添加 bos 标记
    print(tokenizer.encode(text, add_bos=False))
    # 对编码结果进行解码
    print(tokenizer.decode(tokenizer.encode(text, add_bos=False))

# 打印指定 id 对应的 token
print("'" + tokenizer.id_to_piece(15043) + "'") # '_Hello'
print("'" + tokenizer.id_to_piece(29871) + "'") # '_'
print("'" + tokenizer.decode([15043]) + "'")        # 'Hello'
print("'" + tokenizer.decode([15043, 15043]) + "'") # 'Hello Hello'
print("'" + tokenizer.decode([29871, 15043]) + "'")               # ' Hello'
print("'" + tokenizer.decode([29871, 15043, 29871, 15043]) + "'") # ' Hello  Hello'

# 打印 C++ 测试用例
print("\n\ntests for C++:\n")
for text in tests:
    # 对文本进行编码，不添加 bos 标记
    res = tokenizer.encode(text, add_bos=False)

    # 替换换行符和制表符
    k = text.replace('\n', '\\n')
    k = k.replace('\t', '\\t')
    k = '"' + k + '"'
    # 打印格式化字符串，以左对齐方式填充空格，以大括号包围变量 k 的值
    print("{ %-24s, { " % k, end='')
    # 遍历列表 res 中的元素
    for x in res:
        # 打印格式化字符串，以右对齐方式填充空格，打印变量 x 的值
        print("%7d," % x, end='')
    # 打印右括号和逗号
    print(" }, },")
# 打印字符串 'hello' 的编码结果
print(tokenizer.encode('hello'))
# 打印字符串 'world' 的编码结果
print(tokenizer.encode('world'))
# 打印字符串 ' world' 的编码结果
print(tokenizer.encode(' world'))
# 打印字符串 'hello world' 的编码结果
print(tokenizer.encode('hello world'))

# 获取文件名参数
fname_tok = args.fname_tok
# 如果文件名参数存在
if fname_tok:
    # 打印正在进行标记化的文件名
    print('tokenizing file: ', fname_tok)
    # 设置输出文件名
    fname_out = fname_tok + '.tok'
    # 打开文件并读取内容
    with open(fname_tok, 'r', encoding='utf-8') as f:
        # 读取文件的所有行
        lines = f.readlines()
        # 将所有行连接成一个字符串
        s = ''.join(lines)
        # 使用分词器对字符串进行编码，同时添加起始标记
        res = tokenizer.encode(s, add_bos=True)
        # 写入文件
        with open(fname_out, 'w', encoding='utf-8') as f:
            # 遍历编码结果并写入文件
            for x in res:
                f.write(str(x) + ' \'' + tokenizer.decode(x) + '\'\n')
        # 打印编码结果的长度
        print('len(res): ', len(res))
        # 打印文件行数
        print('len(lines): ', len(lines))
    # 打印结果写入的文件名
    print('results written to: ', fname_out)
```