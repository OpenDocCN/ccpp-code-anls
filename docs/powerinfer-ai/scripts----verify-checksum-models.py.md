# `PowerInfer\scripts\verify-checksum-models.py`

```
#!/usr/bin/env python3
# 指定脚本解释器为 Python3

import os
import hashlib
# 导入所需的模块

def sha256sum(file):
    # 定义块大小为 16 MB
    block_size = 16 * 1024 * 1024  # 16 MB block size
    # 创建一个指定大小的字节数组
    b = bytearray(block_size)
    # 创建一个 SHA-256 哈希对象
    file_hash = hashlib.sha256()
    # 创建一个内存视图
    mv = memoryview(b)
    # 以二进制只读模式打开文件
    with open(file, 'rb', buffering=0) as f:
        # 循环读取文件内容并更新哈希值
        while True:
            n = f.readinto(mv)
            if not n:
                break
            file_hash.update(mv[:n])

    # 返回文件的 SHA-256 哈希值
    return file_hash.hexdigest()
# 定义llama目录的路径（脚本目录的父文件夹）
llama_path = os.path.abspath(os.path.join(os.path.dirname(__file__), os.pardir))

# 定义包含哈希值和文件名列表的文件
hash_list_file = os.path.join(llama_path, "SHA256SUMS")

# 检查哈希列表文件是否存在
if not os.path.exists(hash_list_file):
    print(f"Hash list file not found: {hash_list_file}")
    exit(1)

# 读取哈希文件内容并将其拆分为行数组
with open(hash_list_file, "r") as f:
    hash_list = f.read().splitlines()

# 创建一个数组来存储结果
results = []

# 遍历哈希列表中的每一行
# 遍历哈希列表中的每一行
for line in hash_list:
    # 将每行分割成哈希值和文件名
    hash_value, filename = line.split("  ")

    # 通过连接llama路径和文件名获取文件的完整路径
    file_path = os.path.join(llama_path, filename)

    # 通知用户完整性检查的进度
    print(f"Verifying the checksum of {file_path}")

    # 检查文件是否存在
    if os.path.exists(file_path):
        # 使用hashlib计算文件的SHA256哈希值
        file_hash = sha256sum(file_path)

        # 比较文件哈希值与预期哈希值
        if file_hash == hash_value:
            valid_checksum = "V"
            file_missing = ""
        else:
# 初始化有效校验和和文件缺失的变量
valid_checksum = ""
file_missing = ""

# 如果条件成立，重新赋值有效校验和和文件缺失的变量
else:
    valid_checksum = ""
    file_missing = "X"

# 将结果添加到数组中
results.append({
    "filename": filename,
    "valid checksum": valid_checksum,
    "file missing": file_missing
})

# 打印结果表的列标题
print("\n" + "filename".ljust(40) + "valid checksum".center(20) + "file missing".center(20)
print("-" * 80)

# 以表格形式输出结果
for r in results:
# 使用 f-string 格式化输出，输出文件名、有效校验和、文件是否丢失的信息
```