# `whisper.cpp\examples\python\whisper_processor.py`

```cpp
# 导入 subprocess 模块，用于执行外部命令
import subprocess
# 导入 sys 模块，用于处理命令行参数
import sys
# 导入 os 模块，用于操作文件系统
import os

# 处理音频文件，使用指定模型，返回处理后的字符串
def process_audio(wav_file, model_name="base.en"):
    """
    Processes an audio file using a specified model and returns the processed string.

    :param wav_file: Path to the WAV file
    :param model_name: Name of the model to use
    :return: Processed string output from the audio processing
    :raises: Exception if an error occurs during processing
    """

    # 拼接模型文件路径
    model = f"./models/ggml-{model_name}.bin"

    # 检查模型文件是否存在
    if not os.path.exists(model):
        raise FileNotFoundError(f"Model file not found: {model} \n\nDownload a model with this command:\n\n> bash ./models/download-ggml-model.sh {model_name}\n\n")

    # 检查音频文件是否存在
    if not os.path.exists(wav_file):
        raise FileNotFoundError(f"WAV file not found: {wav_file}")

    # 拼接完整的命令
    full_command = f"./main -m {model} -f {wav_file} -np -nt"

    # 执行命令
    process = subprocess.Popen(full_command, shell=True, stdout=subprocess.PIPE, stderr=subprocess.PIPE)

    # 获取输出和错误信息（如果有）
    output, error = process.communicate()

    # 如果有错误信息，则抛出异常
    if error:
        raise Exception(f"Error processing audio: {error.decode('utf-8')}")

    # 处理并返回输出字符串
    decoded_str = output.decode('utf-8').strip()
    processed_str = decoded_str.replace('[BLANK_AUDIO]', '').strip()

    return processed_str

# 主函数
def main():
    # 如果命令行参数大于等于2个
    if len(sys.argv) >= 2:
        # 获取音频文件路径
        wav_file = sys.argv[1]
        # 获取模型名称（如果有的话）
        model_name = sys.argv[2] if len(sys.argv) == 3 else "base.en"
        try:
            # 调用处理音频函数
            result = process_audio(wav_file, model_name)
            # 打印处理后的结果
            print(result)
        except Exception as e:
            # 捕获异常并打印错误信息
            print(f"Error: {e}")
    else:
        # 如果参数不足，则打印用法信息
        print("Usage: python whisper_processor.py <wav_file> [<model_name>]")

# 如果当前脚本作为主程序运行，则执行主函数
if __name__ == "__main__":
    main()
```