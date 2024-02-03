# `whisper.cpp\extra\bench.py`

```cpp
# 导入必要的模块
import os
import subprocess
import re
import csv
import wave
import contextlib
import argparse

# 自定义操作以处理逗号分隔的列表
class ListAction(argparse.Action):
    def __call__(self, parser, namespace, values, option_string=None):
        setattr(namespace, self.dest, [int(val) for val in values.split(",")])

# 创建参数解析器对象，描述为“Benchmark the speech recognition model”
parser = argparse.ArgumentParser(description="Benchmark the speech recognition model")

# 定义接受列表的参数
parser.add_argument(
    "-t",
    "--threads",
    dest="threads",
    action=ListAction,
    default=[4],
    help="List of thread counts to benchmark (comma-separated, default: 4)",
)

parser.add_argument(
    "-p",
    "--processors",
    dest="processors",
    action=ListAction,
    default=[1],
    help="List of processor counts to benchmark (comma-separated, default: 1)",
)

parser.add_argument(
    "-f",
    "--filename",
    type=str,
    default="./samples/jfk.wav",
    help="Relative path of the file to transcribe (default: ./samples/jfk.wav)",
)

# 解析命令行参数
args = parser.parse_args()

# 获取样本文件路径
sample_file = args.filename

# 获取线程和处理器数量
threads = args.threads
processors = args.processors

# 定义要进行基准测试的模型、线程和处理器数量
models = [
    "ggml-tiny.en.bin",
    "ggml-tiny.bin",
    "ggml-base.en.bin",
    "ggml-base.bin",
    "ggml-small.en.bin",
    "ggml-small.bin",
    "ggml-medium.en.bin",
    "ggml-medium.bin",
    "ggml-large-v1.bin",
    "ggml-large-v2.bin",
    "ggml-large-v3.bin",
]

# 初始化一个字典来保存结果
results = {}

# 定义表头
gitHashHeader = "Commit"
modelHeader = "Model"
hardwareHeader = "Hardware"
recordingLengthHeader = "Recording Length (seconds)"
threadHeader = "Thread"
processorCountHeader = "Processor Count"
loadTimeHeader = "Load Time (ms)"
sampleTimeHeader = "Sample Time (ms)"
encodeTimeHeader = "Encode Time (ms)"
decodeTimeHeader = "Decode Time (ms)"
sampleTimePerRunHeader = "Sample Time per Run (ms)"
# 定义编码运行时间的表头
encodeTimePerRunHeader = "Encode Time per Run (ms)"
# 定义解码运行时间的表头
decodeTimePerRunHeader = "Decode Time per Run (ms)"
# 定义总运行时间的表头
totalTimeHeader = "Total Time (ms)"

# 检查文件是否存在
def check_file_exists(file: str) -> bool:
    return os.path.isfile(file)

# 获取 Git 仓库的短哈希值
def get_git_short_hash() -> str:
    try:
        return (
            subprocess.check_output(["git", "rev-parse", "--short", "HEAD"])
            .decode()
            .strip()
        )
    except subprocess.CalledProcessError as e:
        return ""

# 获取 WAV 文件的长度
def wav_file_length(file: str = sample_file) -> float:
    with contextlib.closing(wave.open(file, "r")) as f:
        frames = f.getnframes()
        rate = f.getframerate()
        duration = frames / float(rate)
        return duration

# 从输出中提取指标
def extract_metrics(output: str, label: str) -> tuple[float, float]:
    match = re.search(rf"{label} \s*=\s*(\d+\.\d+)\s*ms\s*/\s*(\d+)\s*runs", output)
    time = float(match.group(1)) if match else None
    runs = float(match.group(2)) if match else None
    return time, runs

# 从输出中提取设备信息
def extract_device(output: str) -> str:
    match = re.search(r"picking default device: (.*)", output)
    device = match.group(1) if match else "Not found"
    return device

# 检查示例文件是否存在
if not check_file_exists(sample_file):
    raise FileNotFoundError(f"Sample file {sample_file} not found")

# 获取录音文件的长度
recording_length = wav_file_length()

# 检查所有模型是否存在
# 从列表中过滤出已下载的模型
filtered_models = []
for model in models:
    if check_file_exists(f"models/{model}"):
        filtered_models.append(model)
    else:
        print(f"Model {model} not found, removing from list")

models = filtered_models

# 遍历每个参数组合
for model in filtered_models:
    # 遍历线程数和处理器数量的组合
    for thread in threads:
        for processor_count in processors:
            # 构造要运行的命令
            cmd = f"./main -m models/{model} -t {thread} -p {processor_count} -f {sample_file}"
            # 运行命令并获取输出
            process = subprocess.Popen(
                cmd, shell=True, stdout=subprocess.PIPE, stderr=subprocess.STDOUT
            )

            output = ""
            while process.poll() is None:
                output += process.stdout.read().decode()

            # 解析输出内容
            load_time_match = re.search(r"load time\s*=\s*(\d+\.\d+)\s*ms", output)
            load_time = float(load_time_match.group(1)) if load_time_match else None

            # 提取设备信息
            metal_device = extract_device(output)
            sample_time, sample_runs = extract_metrics(output, "sample time")
            encode_time, encode_runs = extract_metrics(output, "encode time")
            decode_time, decode_runs = extract_metrics(output, "decode time")

            total_time_match = re.search(r"total time\s*=\s*(\d+\.\d+)\s*ms", output)
            total_time = float(total_time_match.group(1)) if total_time_match else None

            model_name = model.replace("ggml-", "").replace(".bin", "")

            # 打印运行结果信息
            print(
                f"Ran model={model_name} threads={thread} processor_count={processor_count}, took {total_time}ms"
            )
            # 将时间信息存储在结果字典中
            results[(model_name, thread, processor_count)] = {
                loadTimeHeader: load_time,
                sampleTimeHeader: sample_time,
                encodeTimeHeader: encode_time,
                decodeTimeHeader: decode_time,
                sampleTimePerRunHeader: round(sample_time / sample_runs, 2),
                encodeTimePerRunHeader: round(encode_time / encode_runs, 2),
                decodeTimePerRunHeader: round(decode_time / decode_runs, 2),
                totalTimeHeader: total_time,
            }
# 将结果写入 CSV 文件
with open("benchmark_results.csv", "w", newline="") as csvfile:
    # 定义 CSV 文件的列名
    fieldnames = [
        gitHashHeader,
        modelHeader,
        hardwareHeader,
        recordingLengthHeader,
        threadHeader,
        processorCountHeader,
        loadTimeHeader,
        sampleTimeHeader,
        encodeTimeHeader,
        decodeTimeHeader,
        sampleTimePerRunHeader,
        encodeTimePerRunHeader,
        decodeTimePerRunHeader,
        totalTimeHeader,
    ]
    # 创建 CSV 写入对象
    writer = csv.DictWriter(csvfile, fieldnames=fieldnames)

    # 写入 CSV 文件的列名
    writer.writeheader()

    # 获取 Git 仓库的短哈希值
    shortHash = get_git_short_hash()
    # 按照总时间升序对结果进行排序
    sorted_results = sorted(results.items(), key=lambda x: x[1].get(totalTimeHeader, 0))
    # 遍历排序后的结果
    for params, times in sorted_results:
        # 构建每一行的数据
        row = {
            gitHashHeader: shortHash,
            modelHeader: params[0],
            hardwareHeader: metal_device,
            recordingLengthHeader: recording_length,
            threadHeader: params[1],
            processorCountHeader: params[2],
        }
        # 更新行数据
        row.update(times)
        # 写入一行数据到 CSV 文件
        writer.writerow(row)
```