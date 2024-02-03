# `chatglm.cpp\chatglm_cpp\openai_api.py`

```
# 导入必要的模块
import asyncio
import json
import logging
import time
from typing import Dict, List, Literal, Optional, Union

import chatglm_cpp
import uvicorn
from fastapi import FastAPI, HTTPException, status
from fastapi.middleware.cors import CORSMiddleware
from pydantic import BaseModel, Field, computed_field
from pydantic_settings import BaseSettings
from sse_starlette.sse import EventSourceResponse

# 配置日志记录器，设置日志级别和格式
logging.basicConfig(level=logging.INFO, format=r"%(asctime)s - %(module)s - %(levelname)s - %(message)s")

# 定义配置类，继承自BaseSettings
class Settings(BaseSettings):
    model: str = "chatglm3-ggml.bin"
    num_threads: int = 0

# 定义工具调用函数模型
class ToolCallFunction(BaseModel):
    arguments: str
    name: str

# 定义工具调用模型
class ToolCall(BaseModel):
    function: Optional[ToolCallFunction] = None
    type: Literal["function"]

# 定义聊天消息模型
class ChatMessage(BaseModel):
    role: Literal["system", "user", "assistant"]
    content: str
    tool_calls: Optional[List[ToolCall]] = None

# 定义Delta消息模型
class DeltaMessage(BaseModel):
    role: Optional[Literal["system", "user", "assistant"]] = None
    content: Optional[str] = None
    tool_calls: Optional[List[ToolCall]] = None

# 定义聊天完成工具函数模型
class ChatCompletionToolFunction(BaseModel):
    description: Optional[str] = None
    name: str
    parameters: Dict

# 定义聊天完成工具模型
class ChatCompletionTool(BaseModel):
    type: Literal["function"] = "function"
    function: ChatCompletionToolFunction

# 定义聊天完成请求模型
class ChatCompletionRequest(BaseModel):
    model: str = "default-model"
    messages: List[ChatMessage]
    temperature: float = Field(default=0.95, ge=0.0, le=2.0)
    top_p: float = Field(default=0.7, ge=0.0, le=1.0)
    stream: bool = False
    max_tokens: int = Field(default=2048, ge=0)
    tools: Optional[List[ChatCompletionTool]] = None

    # 定义模型配置
    model_config = {
        "json_schema_extra": {"examples": [{"model": "default-model", "messages": [{"role": "user", "content": "你好"}]}]}
    }

# 定义聊天完成响应选择模型
class ChatCompletionResponseChoice(BaseModel):
    index: int = 0
    message: ChatMessage
    finish_reason: Literal["stop", "length", "function_call"]
# 定义 ChatCompletionResponseStreamChoice 类，包含 index、delta 和 finish_reason 属性
class ChatCompletionResponseStreamChoice(BaseModel):
    index: int = 0
    delta: DeltaMessage
    finish_reason: Optional[Literal["stop", "length"]] = None

# 定义 ChatCompletionUsage 类，包含 prompt_tokens、completion_tokens 和 total_tokens 属性
class ChatCompletionUsage(BaseModel):
    prompt_tokens: int
    completion_tokens: int

    # 计算属性，返回 prompt_tokens 和 completion_tokens 之和
    @computed_field
    @property
    def total_tokens(self) -> int:
        return self.prompt_tokens + self.completion_tokens

# 定义 ChatCompletionResponse 类，包含 id、model、object、created、choices 和 usage 属性
class ChatCompletionResponse(BaseModel):
    id: str = "chatcmpl"
    model: str = "default-model"
    object: Literal["chat.completion", "chat.completion.chunk"]
    created: int = Field(default_factory=lambda: int(time.time()))
    choices: Union[List[ChatCompletionResponseChoice], List[ChatCompletionResponseStreamChoice]]
    usage: Optional[ChatCompletionUsage] = None

    # 定义 model_config 字典
    model_config = {
        "json_schema_extra": {
            "examples": [
                {
                    "id": "chatcmpl",
                    "model": "default-model",
                    "object": "chat.completion",
                    "created": 1691166146,
                    "choices": [
                        {
                            "index": 0,
                            "message": {"role": "assistant", "content": "你好👋！我是人工智能助手 ChatGLM2-6B，很高兴见到你，欢迎问我任何问题。"},
                            "finish_reason": "stop",
                        }
                    ],
                    "usage": {"prompt_tokens": 17, "completion_tokens": 29, "total_tokens": 46},
                }
            ]
        }
    }

# 创建 Settings 实例
settings = Settings()
# 创建 FastAPI 应用实例
app = FastAPI()
# 添加 CORS 中间件，允许跨域请求
app.add_middleware(
    CORSMiddleware, allow_origins=["*"], allow_credentials=True, allow_methods=["*"], allow_headers=["*"]
)
# 创建 chatglm_cpp.Pipeline 实例
pipeline = chatglm_cpp.Pipeline(settings.model)
# 创建 asyncio 锁实例
lock = asyncio.Lock()

# 定义 stream_chat 函数，生成 ChatCompletionResponse 对象
def stream_chat(messages, body):
    yield ChatCompletionResponse(
        object="chat.completion.chunk",
        choices=[ChatCompletionResponseStreamChoice(delta=DeltaMessage(role="assistant"))],
    )
    # 遍历聊天管道返回的内容块
    for chunk in pipeline.chat(
        messages=messages,  # 传入消息列表
        max_length=body.max_tokens,  # 最大生成长度
        do_sample=body.temperature > 0,  # 是否使用采样
        top_p=body.top_p,  # 顶部 p 值
        temperature=body.temperature,  # 温度值
        num_threads=settings.num_threads,  # 线程数
        stream=True,  # 是否流式处理
    ):
        # 返回聊天完成响应对象，包含内容块
        yield ChatCompletionResponse(
            object="chat.completion.chunk",  # 对象类型
            choices=[ChatCompletionResponseStreamChoice(delta=DeltaMessage(content=chunk.content))],  # 选择项列表
        )

    # 返回聊天完成响应对象，包含空内容块和停止原因
    yield ChatCompletionResponse(
        object="chat.completion.chunk",  # 对象类型
        choices=[ChatCompletionResponseStreamChoice(delta=DeltaMessage(), finish_reason="stop")],  # 选择项列表
    )
# 异步函数，用于发布聊天事件流
async def stream_chat_event_publisher(history, body):
    # 初始化输出字符串
    output = ""
    try:
        # 使用锁来确保线程安全
        async with lock:
            # 遍历聊天事件流的每个块
            for chunk in stream_chat(history, body):
                # 暂停当前协程，将控制权交还给事件循环以进行取消检查
                await asyncio.sleep(0)
                # 将当前块的内容添加到输出中
                output += chunk.choices[0].delta.content or ""
                # 返回当前块的 JSON 格式化字符串
                yield chunk.model_dump_json(exclude_unset=True)
        # 记录日志，包含最后一个历史记录和输出内容
        logging.info(f'prompt: "{history[-1]}", stream response: "{output}"')
    except asyncio.CancelledError as e:
        # 如果发生取消错误，记录日志，包含最后一个历史记录和部分输出内容
        logging.info(f'prompt: "{history[-1]}", stream response (partial): "{output}"')
        raise e

# 创建聊天完成请求的路由处理函数
@app.post("/v1/chat/completions")
async def create_chat_completion(body: ChatCompletionRequest) -> ChatCompletionResponse:
    # 将字符串转换为 JSON 格式的参数
    def to_json_arguments(arguments):
        def tool_call(**kwargs):
            return kwargs

        try:
            return json.dumps(eval(arguments, dict(tool_call=tool_call)))
        except Exception:
            return arguments

    # 如果消息为空，则抛出 HTTP 异常
    if not body.messages:
        raise HTTPException(status.HTTP_400_BAD_REQUEST, "empty messages")

    # 将请求中的消息转换为 ChatMessage 对象列表
    messages = [chatglm_cpp.ChatMessage(role=msg.role, content=msg.content) for msg in body.messages]
    
    # 如果存在工具列表，则创建系统消息内容
    if body.tools:
        system_content = (
            "Answer the following questions as best as you can. You have access to the following tools:\n"
            + json.dumps([tool.model_dump() for tool in body.tools], indent=4)
        )
        # 将系统消息插入到消息列表的开头
        messages.insert(0, chatglm_cpp.ChatMessage(role="system", content=system_content))

    # 如果需要流式处理，则创建事件流发布器
    if body.stream:
        generator = stream_chat_event_publisher(messages, body)
        return EventSourceResponse(generator)

    # 设置最大上下文长度
    max_context_length = 512
    # 调用聊天模型，生成聊天完成的输出
    output = pipeline.chat(
        messages=messages,
        max_length=body.max_tokens,
        max_context_length=max_context_length,
        do_sample=body.temperature > 0,
        top_p=body.top_p,
        temperature=body.temperature,
    )
    # 记录日志，包含最后一条消息的内容和同步响应的内容
    logging.info(f'prompt: "{messages[-1].content}", sync response: "{output.content}"')
    # 计算提示消息的 token 数量
    prompt_tokens = len(pipeline.tokenizer.encode_messages(messages, max_context_length))
    # 计算完成消息的 token 数量
    completion_tokens = len(pipeline.tokenizer.encode(output.content, body.max_tokens))

    # 完成原因默认为 "stop"
    finish_reason = "stop"
    tool_calls = None
    # 如果存在工具调用
    if output.tool_calls:
        # 将工具调用转换为 ToolCall 对象列表
        tool_calls = [
            ToolCall(
                type=tool_call.type,
                function=ToolCallFunction(
                    name=tool_call.function.name, arguments=to_json_arguments(tool_call.function.arguments)
                ),
            )
            for tool_call in output.tool_calls
        ]
        # 完成原因更新为 "function_call"
        finish_reason = "function_call"

    # 返回 ChatCompletionResponse 对象
    return ChatCompletionResponse(
        object="chat.completion",
        choices=[
            ChatCompletionResponseChoice(
                message=ChatMessage(role="assistant", content=output.content, tool_calls=tool_calls),
                finish_reason=finish_reason,
            )
        ],
        usage=ChatCompletionUsage(prompt_tokens=prompt_tokens, completion_tokens=completion_tokens),
    )
# 定义一个模型卡片类，继承自BaseModel
class ModelCard(BaseModel):
    # 模型卡片的id属性，类型为字符串
    id: str
    # 模型卡片的object属性，值为"model"，默认为"model"
    object: Literal["model"] = "model"
    # 模型卡片的owned_by属性，值为"owner"，默认为"owner"
    owned_by: str = "owner"
    # 模型卡片的permission属性，值为列表，为空列表
    permission: List = []

# 定义一个模型列表类，继承自BaseModel
class ModelList(BaseModel):
    # 模型列表的object属性，值为"list"，默认为"list"
    object: Literal["list"] = "list"
    # 模型列表的data属性，值为模型卡片列表，初始为空列表
    data: List[ModelCard] = []

    # 模型列表的model_config属性，值为字典，包含json_schema_extra字段
    model_config = {
        "json_schema_extra": {
            "examples": [
                {
                    "object": "list",
                    "data": [{"id": "gpt-3.5-turbo", "object": "model", "owned_by": "owner", "permission": []}],
                }
            ]
        }
    }

# 定义一个异步函数，用于处理GET请求，返回一个模型列表对象
@app.get("/v1/models")
async def list_models() -> ModelList:
    # 返回一个包含一个模型卡片对象的模型列表对象
    return ModelList(data=[ModelCard(id="gpt-3.5-turbo")])

# 如果当前脚本作为主程序运行，则使用uvicorn运行应用，监听0.0.0.0:8000地址，使用1个worker
if __name__ == "__main__":
    uvicorn.run(app, host="0.0.0.0", port=8000, workers=1)
```