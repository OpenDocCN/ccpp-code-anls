# `chatglm.cpp\examples\cli_demo.py`

```
# 导入必要的模块
import argparse
from pathlib import Path
from typing import List

# 导入 chatglm_cpp 模块
import chatglm_cpp

# 设置默认模型路径为当前文件的上级目录下的 chatglm-ggml.bin 文件
DEFAULT_MODEL_PATH = Path(__file__).resolve().parent.parent / "chatglm-ggml.bin"

# 定义程序的横幅信息
BANNER = """
    ________          __  ________    __  ___                 
   / ____/ /_  ____ _/ /_/ ____/ /   /  |/  /_________  ____  
  / /   / __ \/ __ `/ __/ / __/ /   / /|_/ // ___/ __ \/ __ \ 
 / /___/ / / / /_/ / /_/ /_/ / /___/ /  / // /__/ /_/ / /_/ / 
 \____/_/ /_/\__,_/\__/\____/_____/_/  /_(_)___/ .___/ .___/  
                                              /_/   /_/       
""".strip("\n")

# 设置欢迎信息
WELCOME_MESSAGE = "Welcome to ChatGLM.cpp! Ask whatever you want. Type 'clear' to clear context. Type 'stop' to exit."

# 主函数
def main() -> None:
    # 创建参数解析器
    parser = argparse.ArgumentParser()
    # 添加参数：模型路径
    parser.add_argument("-m", "--model", default=DEFAULT_MODEL_PATH, type=str, help="model path")
    # 添加参数：推理模式
    parser.add_argument("--mode", default="chat", type=str, choices=["chat", "generate"], help="inference mode")
    # 添加参数：起始提示
    parser.add_argument("-p", "--prompt", default="你好", type=str, help="prompt to start generation with")
    # 添加参数：提示路径
    parser.add_argument("--pp", "--prompt_path", default=None, type=Path, help="path to the plain text file that stores the prompt")
    # 添加参数：系统消息
    parser.add_argument("-s", "--system", default=None, type=str, help="system message to set the behavior of the assistant")
    # 添加参数：系统消息路径
    parser.add_argument("--sp", "--system_path", default=None, type=Path, help="path to the plain text file that stores the system message")
    # 添加参数：交互模式
    parser.add_argument("-i", "--interactive", action="store_true", help="run in interactive mode")
    # 添加参数：最大长度
    parser.add_argument("-l", "--max_length", default=2048, type=int, help="max total length including prompt and output")
    # 添加参数：最大新生成的标记数
    parser.add_argument("--max_new_tokens", default=-1, type=int, help="max number of tokens to generate, ignoring the number of prompt tokens")
    # 添加命令行参数，设置最大上下文长度，默认为512
    parser.add_argument("-c", "--max_context_length", default=512, type=int, help="max context length")
    # 添加命令行参数，设置top-k采样，默认为0
    parser.add_argument("--top_k", default=0, type=int, help="top-k sampling")
    # 添加命令行参数，设置top-p采样，默认为0.7
    parser.add_argument("--top_p", default=0.7, type=float, help="top-p sampling")
    # 添加命令行参数，设置温度，默认为0.95
    parser.add_argument("--temp", default=0.95, type=float, help="temperature")
    # 添加命令行参数，设置重复惩罚，默认为1.0
    parser.add_argument("--repeat_penalty", default=1.0, type=float, help="penalize repeat sequence of tokens")
    # 添加命令行参数，设置推理线程数，默认为0
    parser.add_argument("-t", "--threads", default=0, type=int, help="number of threads for inference")
    # 解析命令行参数
    args = parser.parse_args()

    # 如果命令行参数中包含prompt，则使用该参数作为提示
    prompt = args.prompt
    if args.pp:
        prompt = args.pp.read_text()

    # 如果命令行参数中包含system，则使用该参数作为系统消息
    system = args.system
    if args.sp:
        system = args.sp.read_text()

    # 创建聊天模型的管道
    pipeline = chatglm_cpp.Pipeline(args.model)

    # 如果模式不是"chat"且为交互模式，则提示不支持交互演示，回退到非交互模式
    if args.mode != "chat" and args.interactive:
        print("interactive demo is only supported for chat mode, falling back to non-interactive one")
        args.interactive = False

    # 设置生成参数
    generation_kwargs = dict(
        max_length=args.max_length,
        max_new_tokens=args.max_new_tokens,
        max_context_length=args.max_context_length,
        do_sample=args.temp > 0,
        top_k=args.top_k,
        top_p=args.top_p,
        temperature=args.temp,
        repetition_penalty=args.repeat_penalty,
        stream=True,
    )

    # 初始化系统消息列表
    system_messages: List[chatglm_cpp.ChatMessage] = []
    # 如果存在系统消息，则添加到系统消息列表中
    if system is not None:
        system_messages.append(chatglm_cpp.ChatMessage(role="system", content=system))

    # 复制系统消息到消息列表中
    messages = system_messages.copy()
    # 如果不是交互模式，则执行以下代码块
    if not args.interactive:
        # 如果是聊天模式
        if args.mode == "chat":
            # 将用户输入的内容添加到消息列表中
            messages.append(chatglm_cpp.ChatMessage(role="user", content=prompt))
            # 对消息列表进行处理，并输出生成的内容
            for chunk in pipeline.chat(messages, **generation_kwargs):
                print(chunk.content, sep="", end="", flush=True)
        else:
            # 生成内容并输出
            for chunk in pipeline.generate(prompt, **generation_kwargs):
                print(chunk, sep="", end="", flush=True)
        # 输出空行
        print()
        # 返回
        return
    
    # 输出横幅
    print(BANNER)
    print()
    # 输出欢迎消息
    print(WELCOME_MESSAGE)
    print()
    
    # 计算提示信息的宽度
    prompt_width = len(pipeline.model.config.model_type_name)
    
    # 如果有系统信息，则输出系统信息
    if system:
        print(f"{'System':{prompt_width}} > {system}")
    # 进入无限循环，直到条件不满足退出循环
    while True:
        # 检查是否存在消息且最后一条消息有工具调用
        if messages and messages[-1].tool_calls:
            # 获取最后一条消息的工具调用
            (tool_call,) = messages[-1].tool_calls
            # 根据工具调用类型输出不同提示信息
            if tool_call.type == "function":
                print(
                    f"Function Call > Please manually call function `{tool_call.function.name}` and provide the results below."
                )
                input_prompt = "Observation   > "
            elif tool_call.type == "code":
                print(f"Code Interpreter > Please manually run the code and provide the results below.")
                input_prompt = "Observation      > "
            else:
                raise ValueError(f"unexpected tool call type {tool_call.type}")
            role = "observation"
        else:
            # 设置输入提示信息和用户角色
            input_prompt = f"{'Prompt':{prompt_width}} > "
            role = "user"

        try:
            # 尝试获取用户输入
            prompt = input(input_prompt)
        except EOFError:
            break

        # 根据用户输入进行不同操作
        if not prompt:
            continue
        if prompt == "stop":
            break
        if prompt == "clear":
            messages = system_messages
            continue

        # 将用户输入添加到消息列表中
        messages.append(chatglm_cpp.ChatMessage(role=role, content=prompt))
        # 输出模型类型名称
        print(f"{pipeline.model.config.model_type_name} > ", sep="", end="")
        chunks = []
        # 遍历消息列表并生成响应
        for chunk in pipeline.chat(messages, **generation_kwargs):
            print(chunk.content, sep="", end="", flush=True)
            chunks.append(chunk)
        print()
        # 将生成的响应消息合并到消息列表中
        messages.append(pipeline.merge_streaming_messages(chunks))

    # 输出结束语
    print("Bye")
# 如果当前脚本被直接执行，则调用主函数
if __name__ == "__main__":
    main()
```