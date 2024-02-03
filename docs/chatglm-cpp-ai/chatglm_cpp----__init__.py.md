# `chatglm.cpp\chatglm_cpp\__init__.py`

```
# 导入必要的库
import tempfile
from dataclasses import dataclass
from pathlib import Path
from typing import Any, Dict, Iterator, List, Optional, Union

# 导入 C 扩展模块
import chatglm_cpp._C as _C
from chatglm_cpp._C import ChatMessage

# 定义模块版本号
__version__ = "0.3.1"

# 定义 DeltaMessage 数据类，包含角色、内容和令牌 ID 列表
@dataclass
class DeltaMessage:
    role: str
    content: str
    token_ids: List[int]

# 确保消息类型为 ChatMessage 或字典类型，返回 ChatMessage 对象
def _ensure_chat_message(message: Union[ChatMessage, Dict[str, Any]]) -> ChatMessage:
    if isinstance(message, ChatMessage):
        chat_message = message
    elif isinstance(message, dict):
        chat_message = ChatMessage(**message)
    else:
        raise TypeError(f"expect message type to be ChatMessage or dict, but got {type(message)}")
    return chat_message

# 定义 Pipeline 类，继承自 _C.Pipeline
class Pipeline(_C.Pipeline):
    def __init__(self, model_path: str, *, dtype: Optional[str] = None) -> None:
        # 如果模型路径是文件，则加载 ggml 模型
        if Path(model_path).is_file():
            super().__init__(str(model_path))
        else:
            # 否则将 hf 模型转换为 ggml 格式
            from chatglm_cpp.convert import convert

            # 如果未指定数据类型，则使用默认值 "q4_0"
            if dtype is None:
                dtype = "q4_0"

            # 使用临时文件转换模型，并初始化 Pipeline 对象
            with tempfile.NamedTemporaryFile("wb") as f:
                convert(f, model_path, dtype=dtype)
                super().__init__(f.name)

    # 聊天方法，接受消息列表和一些参数
    def chat(
        self,
        messages: List[ChatMessage],
        *,
        max_length: int = 2048,
        max_new_tokens: int = -1,
        max_context_length: int = 512,
        do_sample: bool = True,
        top_k: int = 0,
        top_p: float = 0.7,
        temperature: float = 0.95,
        repetition_penalty: float = 1.0,
        num_threads: int = 0,
        stream: bool = False,
    # 根据输入的消息列表，确保每个消息都是聊天消息类型
    messages = [_ensure_chat_message(msg) for msg in messages]
    # 使用分词器对消息列表进行编码，限制上下文长度
    input_ids = self.tokenizer.encode_messages(messages, max_context_length)
    # 创建生成配置对象，包括生成的最大长度、最大新标记数、最大上下文长度等参数
    gen_config = _C.GenerationConfig(
        max_length=max_length,
        max_new_tokens=max_new_tokens,
        max_context_length=max_context_length,
        do_sample=do_sample,
        top_k=top_k,
        top_p=top_p,
        temperature=temperature,
        repetition_penalty=repetition_penalty,
        num_threads=num_threads,
    )
    # 如果是流式生成，则调用流式聊天方法
    if stream:
        return self._stream_chat(input_ids=input_ids, gen_config=gen_config)
    # 否则调用同步聊天方法
    return self._sync_chat(input_ids=input_ids, gen_config=gen_config)

def generate(
    self,
    prompt: str,
    *,
    max_length: int = 2048,
    max_new_tokens: int = -1,
    max_context_length: int = 512,
    do_sample: bool = True,
    top_k: int = 0,
    top_p: float = 0.7,
    temperature: float = 0.95,
    repetition_penalty: float = 1.0,
    num_threads: int = 0,
    stream: bool = False,
) -> Union[Iterator[str], str]:
    # 使用分词器对提示进行编码，限制上下文长度
    input_ids = self.tokenizer.encode(prompt, max_context_length)
    # 创建生成配置对象，包括生成的最大长度、最大新标记数、最大上下文长度等参数
    gen_config = _C.GenerationConfig(
        max_length=max_length,
        max_new_tokens=max_new_tokens,
        max_context_length=max_context_length,
        do_sample=do_sample,
        top_k=top_k,
        top_p=top_p,
        temperature=temperature,
        repetition_penalty=repetition_penalty,
        num_threads=num_threads,
    )
    # 如果是流式生成，则调用流式生成方法
    if stream:
        return self._stream_generate(input_ids=input_ids, gen_config=gen_config)
    # 否则调用同步生成方法
    return self._sync_generate(input_ids=input_ids, gen_config=gen_config)
    # 生成器函数，用于生成下一个 token 的 id
    def _stream_generate_ids(self, input_ids: List[int], gen_config: _C.GenerationConfig) -> Iterator[int]:
        # 复制输入的 id 列表
        input_ids = input_ids.copy()
        # 初始化过去 token 数量为 0
        n_past = 0
        # 获取当前上下文 token 数量
        n_ctx = len(input_ids)
        # 计算最大新增 token 数量，如果配置中设置了最大新增 token 数量，则使用该值，否则使用最大长度
        max_new_tokens = gen_config.max_new_tokens if gen_config.max_new_tokens > 0 else gen_config.max_length

        # 循环生成下一个 token 的 id
        while len(input_ids) < min(gen_config.max_length, n_ctx + max_new_tokens):
            # 生成下一个 token 的 id
            next_token_id = self.model.generate_next_token(input_ids, gen_config, n_past, n_ctx)
            # 返回生成的 token 的 id
            yield next_token_id
            # 更新过去 token 数量
            n_past = len(input_ids)
            # 将生成的 token 的 id 添加到输入 id 列表中
            input_ids.append(next_token_id)

            # 如果生成的 token 的 id 是结束 token id 或者额外的结束 token id，则结束循环
            if next_token_id in [self.model.config.eos_token_id, *self.model.config.extra_eos_token_ids]:
                break
    # 生成对话流，根据输入的 ID 列表和生成配置，返回一个迭代器，每次生成一个 DeltaMessage 对象
    def _stream_chat(self, input_ids: List[int], gen_config: _C.GenerationConfig) -> Iterator[DeltaMessage]:
        # 用于缓存生成的 token
        token_cache = []
        # 记录已经打印的字符长度
        print_len = 0
        # 记录已经打印的 token 长度
        print_token_len = 0
        # 遍历生成的 token
        for next_token_id in self._stream_generate_ids(input_ids=input_ids, gen_config=gen_config):
            # 将下一个 token 添加到缓存中
            token_cache.append(next_token_id)
            # 将缓存的 token 转换为文本输出
            output = self.tokenizer.decode(token_cache)

            # 如果输出以换行符结尾
            if output.endswith("\n"):
                # 生成 DeltaMessage 对象，包含角色、内容和 token_ids
                yield DeltaMessage(
                    role=ChatMessage.ROLE_ASSISTANT, content=output[print_len:], token_ids=token_cache[print_token_len:]
                )
                # 重置缓存和打印长度
                token_cache = []
                print_len = 0
                print_token_len = 0
            # 如果输出以逗号、感叹号、冒号、分号、问号或特殊字符结尾
            elif output.endswith((",", "!", ":", ";", "?", "�")):
                # 不做任何操作
                pass
            else:
                # 生成 DeltaMessage 对象，包含角色、内容和 token_ids
                yield DeltaMessage(
                    role=ChatMessage.ROLE_ASSISTANT, content=output[print_len:], token_ids=token_cache[print_token_len:]
                )
                # 更新打印长度和 token 长度
                print_len = len(output)
                print_token_len = len(token_cache)

        # 将剩余的 token 转换为文本输出
        output = self.tokenizer.decode(token_cache)
        # 生成最后一个 DeltaMessage 对象
        yield DeltaMessage(
            role=ChatMessage.ROLE_ASSISTANT, content=output[print_len:], token_ids=token_cache[print_token_len:]
        )

    # 生成对话流，根据输入的 ID 列表和生成配置，返回一个迭代器，每次生成一个字符串
    def _stream_generate(self, input_ids: List[int], gen_config: _C.GenerationConfig) -> Iterator[str]:
        # 遍历生成的 DeltaMessage 对象
        for msg in self._stream_chat(input_ids=input_ids, gen_config=gen_config):
            # 返回 DeltaMessage 对象的内容部分
            yield msg.content

    # 同步生成 token ID 列表，根据输入的 ID 列表和生成配置，返回一个列表
    def _sync_generate_ids(self, input_ids: List[int], gen_config: _C.GenerationConfig) -> List[int]:
        # 调用 _stream_generate_ids 方法，将生成的 token ID 转换为列表返回
        return list(self._stream_generate_ids(input_ids=input_ids, gen_config=gen_config))

    # 同步生成字符串，根据输入的 ID 列表和生成配置，返回一个字符串
    def _sync_generate(self, input_ids: List[int], gen_config: _C.GenerationConfig) -> str:
        # 调用 _sync_generate_ids 方法，将生成的 token ID 转换为字符串返回
        output_ids = self._sync_generate_ids(input_ids=input_ids, gen_config=gen_config)
        return self.tokenizer.decode(output_ids)
    # 同步生成聊天消息，根据输入的ID列表和生成配置生成输出ID列表，然后解码为聊天消息
    def _sync_chat(self, input_ids: List[int], gen_config: _C.GenerationConfig) -> ChatMessage:
        # 调用内部方法生成输出ID列表
        output_ids = self._sync_generate_ids(input_ids=input_ids, gen_config=gen_config)
        # 使用分词器解码输出ID列表为聊天消息
        return self.tokenizer.decode_message(output_ids)
    
    # 合并流式消息，将多个消息块中的token ID列表合并为一个输出ID列表，然后解码为聊天消息
    def merge_streaming_messages(self, chunks: List[DeltaMessage]) -> ChatMessage:
        # 将所有消息块中的token ID列表合并为一个输出ID列表
        output_ids = [x for chunk in chunks for x in chunk.token_ids]
        # 使用分词器解码输出ID列表为聊天消息
        return self.tokenizer.decode_message(output_ids)
```