# `chatglm.cpp\examples\web_demo.py`

```
# 导入必要的库
import argparse
from pathlib import Path

# 导入自定义的 chatglm_cpp 模块和 gradio 模块
import chatglm_cpp
import gradio as gr

# 默认模型路径为当前文件的上级目录下的 chatglm-ggml.bin 文件
DEFAULT_MODEL_PATH = Path(__file__).resolve().parent.parent / "chatglm-ggml.bin"

# 创建参数解析器
parser = argparse.ArgumentParser()
# 添加参数选项
parser.add_argument("-m", "--model", default=DEFAULT_MODEL_PATH, type=Path, help="model path")
parser.add_argument("--mode", default="chat", type=str, choices=["chat", "generate"], help="inference mode")
parser.add_argument("-l", "--max_length", default=2048, type=int, help="max total length including prompt and output")
parser.add_argument("-c", "--max_context_length", default=512, type=int, help="max context length")
parser.add_argument("--top_k", default=0, type=int, help="top-k sampling")
parser.add_argument("--top_p", default=0.7, type=float, help="top-p sampling")
parser.add_argument("--temp", default=0.95, type=float, help="temperature")
parser.add_argument("--repeat_penalty", default=1.0, type=float, help="penalize repeat sequence of tokens")
parser.add_argument("-t", "--threads", default=0, type=int, help="number of threads for inference")
parser.add_argument("--plain", action="store_true", help="display in plain text without markdown support")
# 解析命令行参数
args = parser.parse_args()

# 创建 chatglm_cpp.Pipeline 对象，使用指定的模型路径
pipeline = chatglm_cpp.Pipeline(args.model)

# 定义用于后处理文本的函数
def postprocess(text):
    if args.plain:
        return f"<pre>{text}</pre>"
    return text

# 定义预测函数，用于生成聊天回复
def predict(input, chatbot, max_length, top_p, temperature, messages):
    # 将用户输入添加到聊天记录中
    chatbot.append((postprocess(input), ""))
    messages.append(chatglm_cpp.ChatMessage(role="user", content=input)

    # 设置生成参数
    generation_kwargs = dict(
        max_length=max_length,
        max_context_length=args.max_context_length,
        do_sample=temperature > 0,
        top_k=args.top_k,
        top_p=top_p,
        temperature=temperature,
        repetition_penalty=args.repeat_penalty,
        num_threads=args.threads,
        stream=True,
    )

    response = ""
    # 如果模式是"chat"
    if args.mode == "chat":
        # 初始化一个空列表用于存储生成的消息块
        chunks = []
        # 遍历pipeline.chat方法生成的消息块
        for chunk in pipeline.chat(messages, **generation_kwargs):
            # 将当前消息块的内容添加到响应中
            response += chunk.content
            # 将当前消息块添加到chunks列表中
            chunks.append(chunk)
            # 更新chatbot列表中最后一个元素，将响应进行后处理
            chatbot[-1] = (chatbot[-1][0], postprocess(response))
            # 生成chatbot和messages的元组并返回
            yield chatbot, messages
        # 将生成的消息块合并并添加到messages列表中
        messages.append(pipeline.merge_streaming_messages(chunks))
    # 如果模式不是"chat"
    else:
        # 遍历pipeline.generate方法生成的消息块
        for chunk in pipeline.generate(input, **generation_kwargs):
            # 将当前消息块的内容添加到响应中
            response += chunk
            # 更新chatbot列表中最后一个元素，将响应进行后处理
            chatbot[-1] = (chatbot[-1][0], postprocess(response))
            # 生成chatbot和messages的元组并返回
            yield chatbot, messages

    # 生成chatbot和messages的元组并返回
    yield chatbot, messages
# 重置用户输入框的内容为空字符串
def reset_user_input():
    return gr.update(value="")

# 重置状态，清空消息列表和聊天记录
def reset_state():
    return [], []

# 创建一个 Blocks 对象 demo
with gr.Blocks() as demo:
    # 在页面中央显示标题为 ChatGLM.cpp 的 HTML 元素
    gr.HTML("""<h1 align="center">ChatGLM.cpp</h1>""")

    # 创建一个 Chatbot 对象
    chatbot = gr.Chatbot()
    # 创建一个行元素
    with gr.Row():
        # 创建一个列元素，比例为 4
        with gr.Column(scale=4):
            # 创建一个文本框用于用户输入，不显示标签，占位符为 "Input..."，行数为 8
            user_input = gr.Textbox(show_label=False, placeholder="Input...", lines=8)
            # 创建一个提交按钮，文本为 "Submit"，样式为 primary
            submitBtn = gr.Button("Submit", variant="primary")
        # 创建一个列元素，比例为 1
        with gr.Column(scale=1):
            # 创建一个滑块，范围为 0 到 2048，默认值为 args.max_length，步长为 1，标签为 "Maximum Length"，可交互
            max_length = gr.Slider(0, 2048, value=args.max_length, step=1.0, label="Maximum Length", interactive=True)
            # 创建一个滑块，范围为 0 到 1，默认值为 args.top_p，步长为 0.01，标签为 "Top P"，可交互
            top_p = gr.Slider(0, 1, value=args.top_p, step=0.01, label="Top P", interactive=True)
            # 创建一个滑块，范围为 0 到 1，默认值为 args.temp，步长为 0.01，标签为 "Temperature"，可交互
            temperature = gr.Slider(0, 1, value=args.temp, step=0.01, label="Temperature", interactive=True)
            # 创建一个按钮，文本为 "Clear History"
            emptyBtn = gr.Button("Clear History")

    # 创建一个状态对象 messages，初始为空列表
    messages = gr.State([])

    # 点击提交按钮时执行 predict 函数，传入参数为用户输入、Chatbot 对象、最大长度、Top P、Temperature 和消息列表
    # 更新 Chatbot 对象和消息列表，显示进度条
    submitBtn.click(
        predict,
        [user_input, chatbot, max_length, top_p, temperature, messages],
        [chatbot, messages],
        show_progress=True,
    )
    # 点击提交按钮时执行 reset_user_input 函数，清空用户输入框
    submitBtn.click(reset_user_input, [], [user_input])

    # 点击清空历史按钮时执行 reset_state 函数，清空 Chatbot 对象和消息列表，显示进度条
    emptyBtn.click(reset_state, outputs=[chatbot, messages], show_progress=True)

# 启动 demo 对象，不共享，使用浏览器打开
demo.queue().launch(share=False, inbrowser=True)
```