# `PowerInfer\scripts\verify-checksum-models.py`

```
#!/usr/bin/env python3

# 导入操作系统和哈希库
import os
import hashlib

# 计算文件的 SHA256 校验和
def sha256sum(file):
    # 定义块大小为 16MB
    block_size = 16 * 1024 * 1024  
    # 创建一个指定大小的字节数组
    b = bytearray(block_size)
    # 创建一个 SHA256 对象
    file_hash = hashlib.sha256()
    # 创建一个内存视图
    mv = memoryview(b)
    # 以无缓冲模式打开文件
    with open(file, 'rb', buffering=0) as f:
        while True:
            # 读取数据到内存视图
            n = f.readinto(mv)
            if not n:
                break
            # 更新文件哈希值
            file_hash.update(mv[:n])

    return file_hash.hexdigest()

# 定义 llama 目录的路径（脚本目录的父目录）
llama_path = os.path.abspath(os.path.join(os.path.dirname(__file__), os.pardir))

# 定义包含哈希值和文件名的文件
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
for line in hash_list:
    # 将行拆分为哈希值和文件名
    hash_value, filename = line.split("  ")

    # 通过连接 llama 路径和文件名来获取文件的完整路径
    file_path = os.path.join(llama_path, filename)

    # 通知用户完整性检查的进度
    print(f"Verifying the checksum of {file_path}")

    # 检查文件是否存在
    if os.path.exists(file_path):
        # 使用 hashlib 计算文件的 SHA256 校验和
        file_hash = sha256sum(file_path)

        # 比较文件哈希与预期哈希
        if file_hash == hash_value:
            valid_checksum = "V"
            file_missing = ""
        else:
            valid_checksum = ""
            file_missing = ""
    else:
        valid_checksum = ""
        file_missing = "X"

    # 将结果添加到数组中
    # 将包含文件名、有效校验和文件缺失信息的字典添加到结果列表中
    results.append({
        "filename": filename,
        "valid checksum": valid_checksum,
        "file missing": file_missing
    })
# 打印结果表格的列标题
print("\n" + "filename".ljust(40) + "valid checksum".center(20) + "file missing".center(20)
# 打印分隔线
print("-" * 80)

# 以表格形式输出结果
for r in results:
    # 使用 f-string 格式化输出每一行的文件名、有效校验和、文件是否丢失的信息
    print(f"{r['filename']:40} {r['valid checksum']:^20} {r['file missing']:^20}")
```