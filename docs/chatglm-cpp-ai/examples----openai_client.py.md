# `chatglm.cpp\examples\openai_client.py`

```cpp
# 导入 argparse 模块，用于解析命令行参数
import argparse

# 从 openai 模块中导入 OpenAI 类
from openai import OpenAI

# 创建 ArgumentParser 对象
parser = argparse.ArgumentParser()
# 添加命令行参数 --stream，如果存在则设置为 True
parser.add_argument("--stream", action="store_true")
# 添加命令行参数 --prompt，默认值为"你好"，类型为字符串
parser.add_argument("--prompt", default="你好", type=str)
# 添加命令行参数 --tool_call，如果存在则设置为 True
parser.add_argument("--tool_call", action="store_true")
# 解析命令行参数
args = parser.parse_args()

# 创建 OpenAI 客户端对象
client = OpenAI()

# 初始化工具列表为 None
tools = None
# 如果命令行参数中存在 --tool_call
if args.tool_call:
    # 设置工具列表为包含一个字典的列表
    tools = [
        {
            "type": "function",
            "function": {
                "name": "get_current_weather",
                "description": "Get the current weather in a given location",
                "parameters": {
                    "type": "object",
                    "properties": {
                        "location": {
                            "type": "string",
                            "description": "The city and state, e.g. San Francisco, CA",
                        },
                        "unit": {"type": "string", "enum": ["celsius", "fahrenheit"]},
                    },
                    "required": ["location"],
                },
            },
        }
    ]

# 创建消息列表，包含用户角色和命令行参数中的提示内容
messages = [{"role": "user", "content": args.prompt}]
# 如果命令行参数中存在 --stream
if args.stream:
    # 发送消息并获取响应，设置 stream 为 True，传入工具列表
    response = client.chat.completions.create(model="default-model", messages=messages, stream=True, tools=tools)
    # 遍历响应的每个部分
    for chunk in response:
        # 获取响应内容
        content = chunk.choices[0].delta.content
        # 如果内容不为空
        if content is not None:
            # 打印内容，不换行，刷新缓冲区
            print(content, end="", flush=True)
    # 打印换行符
    print()
else:
    # 发送消息并获取响应，传入工具列表
    response = client.chat.completions.create(model="default-model", messages=messages, tools=tools)
    # 打印响应的第一个选择的消息内容
    print(response.choices[0].message.content)
```