# `chatglm.cpp\examples\chatglm3_demo.py`

```
# 导入必要的库
from __future__ import annotations

import base64
import functools
import io
import json
import queue
import re
import traceback
from enum import Enum
from pathlib import Path
from typing import Callable

# 导入第三方库
import chatglm_cpp
import jupyter_client
import streamlit as st
from PIL import Image

# 定义常量
IPYKERNEL = "chatglm3-demo"
MODEL_PATH = Path(__file__).resolve().parent.parent / "chatglm3-ggml.bin"

# 定义聊天系统提示语
CHAT_SYSTEM_PROMPT = "You are ChatGLM3, a large language model trained by Zhipu.AI. Follow the user's instructions carefully. Respond using markdown."

# 定义工具列表
TOOLS = [
    {
        "name": "random_number_generator",
        "description": "Generates a random number x, s.t. range[0] <= x < range[1]",
        "parameters": {
            "type": "object",
            "properties": {
                "seed": {"description": "The random seed used by the generator", "type": "integer"},
                "range": {
                    "description": "The range of the generated numbers",
                    "type": "array",
                    "items": [{"type": "integer"}, {"type": "integer"}],
                },
            },
            "required": ["seed", "range"],
        },
    },
    {
        "name": "get_weather",
        "description": "Get the current weather for `city_name`",
        "parameters": {
            "type": "object",
            "properties": {"city_name": {"description": "The name of the city to be queried", "type": "string"}},
            "required": ["city_name"],
        },
    },
]

# 定义工具系统提示语
TOOL_SYSTEM_PROMPT = (
    "Answer the following questions as best as you can. You have access to the following tools:\n"
    + json.dumps(TOOLS, indent=4)
)

# 定义CI系统提示语
CI_SYSTEM_PROMPT = "你是一位智能AI助手，你叫ChatGLM，你连接着一台电脑，但请注意不能联网。在使用Python解决任务时，你可以运行代码并得到结果，如果运行结果有错误，你需要尽可能对代码进行改进。你可以处理用户上传到电脑上的文件，文件默认存储路径是/mnt/data/。"

# 定义模式枚举类
class Mode(str, Enum):
    CHAT = "💬 Chat"
    TOOL = "🛠️ Tool"
    CI = "🧑‍💻 Code Interpreter"

# 使用Streamlit缓存资源的装饰器，获取模型
@st.cache_resource
def get_model(model_path: str) -> chatglm_cpp.Pipeline:
    # 返回一个 chatglm_cpp.Pipeline 对象，该对象使用指定的模型路径
    return chatglm_cpp.Pipeline(model_path)
# 定义一个名为 Message 的类，继承自 chatglm_cpp.ChatMessage
class Message(chatglm_cpp.ChatMessage):
    # 初始化方法，接受角色、内容、工具调用列表和图片作为参数
    def __init__(
        self, role: str, content: str, tool_calls: list | None = None, image: Image.Image | None = None
    ) -> None:
        # 如果工具调用列表为空，则将其设为一个空列表
        if tool_calls is None:
            tool_calls = []
        # 调用父类的初始化方法，传入角色、内容和工具调用列表
        super().__init__(role, content, tool_calls)
        # 将图片赋值给对象的 image 属性
        self.image = image

    # 静态方法，从 chatglm_cpp.ChatMessage 对象转换为 Message 对象
    @staticmethod
    def from_cpp(cpp_message: chatglm_cpp.ChatMessage) -> Message:
        # 创建一个 Message 对象，传入角色、内容、工具调用列表和空的图片
        return Message(
            role=cpp_message.role, content=cpp_message.content, tool_calls=cpp_message.tool_calls, image=None
        )

# 显示消息的函数，接受一个 Message 对象作为参数
def show_message(message: Message) -> None:
    # 角色头像映射关系
    role_avatars = {"user": "user", "observation": "user", "assistant": "assistant"}
    # 根据消息的角色获取对应的头像
    avatar = role_avatars.get(message.role)
    # 如果头像为空，输出错误信息并返回
    if avatar is None:
        st.error(f"Unexpected message role {message.role}")
        return

    # 显示内容为消息的内容
    display_content = message.content
    # 如果消息有工具调用
    if message.tool_calls:
        # 获取第一个工具调用
        (tool_call,) = message.tool_calls
        # 如果工具调用类型为函数，将函数名添加到显示内容前面
        if tool_call.type == "function":
            display_content = f"{tool_call.function.name}\n{display_content}"
        # 如果工具调用类型为代码，将代码输入添加到显示内容后面
        elif tool_call.type == "code":
            display_content += "\n" + tool_call.code.input

    # 如果消息角色为 observation，将显示内容用代码块包裹
    if message.role == "observation":
        display_content = f"```\n{display_content.strip()}\n```"

    # 使用 st.chat_message 显示消息，传入角色和头像
    with st.chat_message(name=message.role, avatar=avatar):
        # 如果消息有图片，显示图片
        if message.image:
            st.image(message.image)
        # 否则，使用 st.markdown 显示内容
        else:
            st.markdown(display_content)

# ----- begin function call -----

# 函数注册表
_FUNCTION_REGISTRY = {}

# 注册函数的装饰器，接受一个函数作为参数
def register_function(func: Callable) -> Callable:
    # 将函数名和函数添加到函数注册表中
    _FUNCTION_REGISTRY[func.__name__] = func

    # 包装函数，保留原函数的元信息
    @functools.wraps(func)
    def wrap(*args, **kwargs):
        return func(*args, **kwargs)

    return wrap

# 注册一个名为 random_number_generator 的函数，接受种子和范围作为参数，返回一个随机数
@register_function
def random_number_generator(seed: int, range: tuple[int, int]) -> int:
    # 导入 random 模块
    import random
    # 使用给定种子创建随机数生成器，返回指定范围内的随机数
    return random.Random(seed).randint(*range)

# 注册一个名为 get_weather 的函数，接受城市名作为参数，返回天气信息
@register_function
def get_weather(city_name: str) -> str:
    # 导入 requests 模块
    import requests
    # 定义一个字典，包含了需要从API响应中提取的关键信息
    key_selection = {
        "current_condition": ["temp_C", "FeelsLikeC", "humidity", "weatherDesc", "observation_time"],
    }
    # 发送GET请求获取API响应
    resp = requests.get(f"https://wttr.in/{city_name}?format=j1")
    # 如果响应状态码不是200，抛出异常
    resp.raise_for_status()
    # 将API响应转换为JSON格式
    resp = resp.json()

    # 从API响应中提取指定的关键信息，组成新的字典
    ret = {k: {_v: resp[k][0][_v] for _v in v} for k, v in key_selection.items()}
    # 将提取的信息转换为JSON格式并返回
    return json.dumps(ret)
# 运行指定函数，并返回结果字符串
def run_function(name: str, arguments: str) -> str:
    # 定义一个工具函数，接受关键字参数并返回
    def tool_call(**kwargs):
        return kwargs

    # 获取函数对象
    func = _FUNCTION_REGISTRY.get(name)
    # 如果函数不存在，则返回错误信息
    if func is None:
        return f"Function `{name}` is not defined"

    try:
        # 尝试解析参数字符串并执行
        kwargs = eval(arguments, dict(tool_call=tool_call))
    except Exception:
        return f"Invalid arguments {arguments}"

    try:
        # 尝试执行函数并返回结果字符串
        return str(func(**kwargs))
    except Exception:
        return traceback.format_exc()


# ----- end function call -----

# ----- begin code interpreter -----

# 从缓存中获取指定内核客户端对象
@st.cache_resource
def get_kernel_client(kernel_name) -> jupyter_client.BlockingKernelClient:
    # 创建内核管理器对象并启动内核
    km = jupyter_client.KernelManager(kernel_name=kernel_name)
    km.start_kernel()

    # 获取阻塞式内核客户端对象
    kc: jupyter_client.BlockingKernelClient = km.blocking_client()
    kc.start_channels()

    return kc


# 清除文本中的 ANSI 转义码
def clean_ansi_codes(text: str) -> str:
    ansi_escape = re.compile(r"(\x9B|\x1B\[|\u001b\[)[0-?]*[ -/]*[@-~]")
    return ansi_escape.sub("", text)


# 从文本中提取代码块
def extract_code(text: str) -> str:
    return re.search(r"```.*?\n(.*?)```", text, re.DOTALL)[1]


# 运行代码块并返回结果
def run_code(kc: jupyter_client.BlockingKernelClient, code: str) -> str | Image.Image:
    kc.execute(code)
    # 尝试从内核客户端获取 shell 消息，设置超时时间为 30 秒
    try:
        shell_msg = kc.get_shell_msg(timeout=30)
        io_msg_content = None
        # 循环获取 iopub 消息
        while True:
            try:
                next_io_msg_content = kc.get_iopub_msg(timeout=30)["content"]
            except queue.Empty:
                break
            # 如果下一个 iopub 消息的执行状态为 idle，则跳出循环
            if next_io_msg_content.get("execution_state") == "idle":
                break
            io_msg_content = next_io_msg_content

        # 如果 shell 消息的状态为 timeout，则返回 "Execution Timeout Expired"
        if shell_msg["metadata"]["status"] == "timeout":
            return "Execution Timeout Expired"

        # 如果 shell 消息的状态为 error，则处理异常信息
        if shell_msg["metadata"]["status"] == "error":
            try:
                # 清理 ANSI 控制码并获取 traceback 内容
                traceback_content = clean_ansi_codes(io_msg_content["traceback"][-1])
            except Exception:
                traceback_content = "Traceback Error"
            return traceback_content

        # 如果 iopub 消息中包含 "text" 字段，则返回该字段内容
        if "text" in io_msg_content:
            return io_msg_content["text"]

        # 获取 iopub 消息中的数据内容
        data_content = io_msg_content.get("data")
        if data_content is not None:
            # 获取数据内容中的 image/png 类型内容，并返回图像对象
            image_content = data_content.get("image/png")
            if image_content is not None:
                return Image.open(io.BytesIO(base64.b64decode(image_content)))

            # 获取数据内容中的 text/plain 类型内容，并返回文本内容
            text_content = data_content.get("text/plain")
            if text_content is not None:
                return text_content

        # 如果没有符合条件的内容，则返回空字符串
        return ""

    # 捕获所有异常，返回异常信息
    except Exception:
        return traceback.format_exc()
# ----- end code interpreter -----

# 主函数，设置页面配置，加载模型，处理用户输入
def main():
    # 设置页面标题、图标、布局和侧边栏状态
    st.set_page_config(page_title="ChatGLM3 Demo", page_icon="🚀", layout="centered", initial_sidebar_state="auto")

    # 获取模型路径
    pipeline = get_model(MODEL_PATH)

    # 设置会话状态中的消息列表
    st.session_state.setdefault("messages", [])

    # 显示页面标题
    st.title("ChatGLM3 Demo")

    # 获取用户输入的聊天信息
    prompt = st.chat_input("Chat with ChatGLM3!", key="chat_input")

    # 选择模式：聊天、工具、CI
    mode = st.radio("Mode", [x.value for x in Mode], horizontal=True, label_visibility="hidden")

    # 默认系统提示映射
    DEFAULT_SYSTEM_PROMPT_MAP = {
        Mode.CHAT: CHAT_SYSTEM_PROMPT,
        Mode.TOOL: TOOL_SYSTEM_PROMPT,
        Mode.CI: CI_SYSTEM_PROMPT,
    }
    default_system_prompt = DEFAULT_SYSTEM_PROMPT_MAP.get(mode)
    if default_system_prompt is None:
        st.error(f"Unexpected mode {mode}")

    # 在侧边栏显示设置选项
    with st.sidebar:
        top_p = st.slider(label="Top P", min_value=0.0, max_value=1.0, value=0.8, step=0.01)
        temperature = st.slider(label="Temperature", min_value=0.0, max_value=1.5, value=0.8, step=0.01)
        max_length = st.slider(label="Max Length", min_value=128, max_value=2048, value=2048, step=16)
        max_context_length = st.slider(label="Max Context Length", min_value=128, max_value=2048, value=1536, step=16)
        system_prompt = st.text_area(label="System Prompt", value=default_system_prompt, height=300)
        if st.button(label="Clear Context", type="primary"):
            st.session_state.messages = []

    # 获取会话状态中的消息列表
    messages: list[Message] = st.session_state.messages

    # 显示消息列表中的消息
    for msg in messages:
        show_message(msg)

    # 如果没有用户输入，则返回
    if not prompt:
        return

    # 去除用户输入的空格
    prompt = prompt.strip()
    # 将用户输入的消息添加到消息列表中
    messages.append(Message(role="user", content=prompt))
    show_message(messages[-1])

    # 工具调用最大重试次数
    TOOL_CALL_MAX_RETRY = 5

if __name__ == "__main__":
    main()
```