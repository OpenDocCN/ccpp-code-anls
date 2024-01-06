# `PowerInfer\tests\test-tokenizer-0-llama.py`

```
# 导入所需的模块
import os
import sys
import argparse
from sentencepiece import SentencePieceProcessor

# 创建参数解析器
parser = argparse.ArgumentParser()
# 添加命令行参数
parser.add_argument("dir_tokenizer", help="directory containing 'tokenizer.model' file")
parser.add_argument("--fname-tok",   help="path to a text file to tokenize")
# 解析命令行参数
args = parser.parse_args()

# 获取参数值
dir_tokenizer = args.dir_tokenizer

# 使用指定的 tokenizer.model 文件创建 SentencePieceProcessor 对象
tokenizer = SentencePieceProcessor(dir_tokenizer + '/tokenizer.model')

# 待测试的文本
tests = [
        "",
        " ",
# 空字符串
"  ",
# 空格
"   ",
# 制表符
"\t",
# 换行符
"\n",
# 制表符和换行符组合
"\t\n",
# 字符串 "Hello world"
"Hello world",
# 字符串 " Hello world"
" Hello world",
# 字符串 "Hello World"
"Hello World",
# 字符串 " Hello World"
" Hello World",
# 字符串 " Hello World!"
" Hello World!",
# 字符串 "Hello, world!"
"Hello, world!",
# 字符串 " Hello, world!"
" Hello, world!",
# 字符串 " this is 🦙.cpp"
" this is 🦙.cpp",
# 字符串 "w048 7tuijk dsdfhu"
"w048 7tuijk dsdfhu",
# 字符串 "нещо на Български"
"нещо на Български",
# 字符串 "កាន់តែពិសេសអាចខលចេញ"
"កាន់តែពិសេសអាចខលចេញ",
# 字符串 "🚀 (normal) 😶‍🌫️ (multiple emojis concatenated) ✅ (only emoji that has its own token)"
"🚀 (normal) 😶‍🌫️ (multiple emojis concatenated) ✅ (only emoji that has its own token)",
# 字符串 "Hello"
"Hello",
# 字符串 " Hello"
" Hello",
# 字符串 "  Hello"
"  Hello",
# 定义一个包含不同文本的列表
tests = [
        "   Hello",
        "    Hello",
        "    Hello\n    Hello",
    ]

# 遍历测试文本列表
for text in tests:
    # 打印当前文本
    print('text: ', text)
    # 打印使用 bos 的编码结果
    print('\nwith bos:')
    print(tokenizer.encode(text, add_bos=True))
    # 打印使用 bos 的解码结果
    print(tokenizer.decode(tokenizer.encode(text, add_bos=True)))
    # 打印不使用 bos 的编码结果
    print('\nwithout bos:')
    print(tokenizer.encode(text, add_bos=False))
    # 打印不使用 bos 的解码结果
    print(tokenizer.decode(tokenizer.encode(text, add_bos=False))

# 打印特定 id 对应的 token
print("'" + tokenizer.id_to_piece(15043) + "'") # '_Hello'
print("'" + tokenizer.id_to_piece(29871) + "'") # '_'
# 打印特定 id 对应的解码结果
print("'" + tokenizer.decode([15043]) + "'")        # 'Hello'
print("'" + tokenizer.decode([15043, 15043]) + "'") # 'Hello Hello'
print("'" + tokenizer.decode([29871, 15043]) + "'")               # ' Hello'
# 打印解码后的编码结果，加上单引号
print("'" + tokenizer.decode([29871, 15043, 29871, 15043]) + "'") # ' Hello  Hello'

# 打印 C++ 测试的提示信息
print("\n\ntests for C++:\n")
# 遍历测试文本列表
for text in tests:
    # 对文本进行编码，不添加起始符号
    res = tokenizer.encode(text, add_bos=False)

    # 替换文本中的换行符和制表符，加上双引号
    k = text.replace('\n', '\\n')
    k = k.replace('\t', '\\t')
    k = '"' + k + '"'
    # 打印编码结果
    print("{ %-24s, { " % k, end='')
    for x in res:
        print("%7d," % x, end='')
    print(" }, },")

# 打印特定文本的编码结果
print(tokenizer.encode('hello'))
print(tokenizer.encode('world'))
print(tokenizer.encode(' world'))
print(tokenizer.encode('hello world'))

# 获取参数中的文件名
fname_tok = args.fname_tok
# 如果文件名不为空
if fname_tok:
    # 打印正在进行词汇标记的文件名
    print('tokenizing file: ', fname_tok)
    # 设置输出文件名
    fname_out = fname_tok + '.tok'
    # 打开文件并读取内容
    with open(fname_tok, 'r', encoding='utf-8') as f:
        # 读取文件的所有行
        lines = f.readlines()
        # 将所有行连接成一个字符串
        s = ''.join(lines)
        # 使用标记器对字符串进行编码
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