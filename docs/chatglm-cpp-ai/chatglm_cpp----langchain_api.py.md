# `chatglm.cpp\chatglm_cpp\langchain_api.py`

```
# 导入日志模块
import logging
# 导入日期时间模块
from datetime import datetime
# 导入类型提示模块
from typing import List, Tuple

# 导入 chatglm_cpp 模块
import chatglm_cpp
# 导入 FastAPI 框架
from fastapi import FastAPI, status
# 导入数据验证模块
from pydantic import BaseModel, Field
# 导入配置模块
from pydantic_settings import BaseSettings

# 配置日志输出格式
logging.basicConfig(level=logging.INFO, format=r"%(asctime)s - %(module)s - %(levelname)s - %(message)s")

# 定义配置类
class Settings(BaseSettings):
    model: str = "chatglm-ggml.bin"

# 定义聊天请求模型
class ChatRequest(BaseModel):
    prompt: str
    history: List[Tuple[str, str]] = []
    max_length: int = Field(default=2048, ge=0)
    top_p: float = Field(default=0.7, ge=0, le=1)
    temperature: float = Field(default=0.95, ge=0, le=2)

    # 模型配置
    model_config = {"json_schema_extra": {"examples": [{"prompt": "你好"}]}}

# 定义聊天响应模型
class ChatResponse(BaseModel):
    response: str
    history: List[Tuple[str, str]]
    status: int
    time: str

    # 模型配置
    model_config = {
        "json_schema_extra": {
            "examples": [
                {
                    "response": "你好👋！我是人工智能助手 ChatGLM-6B，很高兴见到你，欢迎问我任何问题。",
                    "history": [["你好", "你好👋！我是人工智能助手 ChatGLM2-6B，很高兴见到你，欢迎问我任何问题。"]],
                    "status": 200,
                    "time": "2023-08-05 23:01:35",
                }
            ]
        }
    }

# 创建 FastAPI 应用
app = FastAPI()
# 加载配置
settings = Settings()
# 记录日志
logging.info(settings)

# 创建聊天管道
pipeline = chatglm_cpp.Pipeline(settings.model)

# 定义聊天接口
@app.post("/")
async def chat(body: ChatRequest) -> ChatResponse:
    messages = []
    # 遍历历史消息
    for prompt, response in body.history:
        messages += [
            chatglm_cpp.ChatMessage(role="user", content=prompt),
            chatglm_cpp.ChatMessage(role="assistant", content=response),
        ]
    messages.append(chatglm_cpp.ChatMessage(role="user", content=body.prompt))

    # 进行聊天
    output = pipeline.chat(
        messages,
        max_length=body.max_length,
        do_sample=body.temperature > 0,
        top_p=body.top_p,
        temperature=body.temperature,
    )
    # 更新历史消息
    history = body.history + [(body.prompt, output.content)]
    # 创建一个包含聊天响应的对象，包括响应内容、历史记录、状态码和时间戳
    answer = ChatResponse(
        response=output.content,
        history=history,
        status=status.HTTP_200_OK,
        time=datetime.now().strftime("%Y-%m-%d %H:%M:%S"),
    )
    # 记录日志，包括用户提示和响应内容
    logging.info(f'prompt: "{body.prompt}", response: "{output.content}"')
    # 返回聊天响应对象
    return answer
```