# `chatglm.cpp\tests\test_chatglm_cpp.py`

```
# 导入必要的模块
from pathlib import Path

# 导入 chatglm_cpp 模块和 pytest 模块
import chatglm_cpp
import pytest

# 获取项目根目录
PROJECT_ROOT = Path(__file__).resolve().parent.parent

# 定义各个模型文件的路径
CHATGLM_MODEL_PATH = PROJECT_ROOT / "chatglm-ggml.bin"
CHATGLM2_MODEL_PATH = PROJECT_ROOT / "chatglm2-ggml.bin"
CHATGLM3_MODEL_PATH = PROJECT_ROOT / "chatglm3-ggml.bin"
CODEGEEX2_MODEL_PATH = PROJECT_ROOT / "codegeex2-ggml.bin"
BAICHUAN13B_MODEL_PATH = PROJECT_ROOT / "baichuan-13b-chat-ggml.bin"
BAICHUAN2_7B_MODEL_PATH = PROJECT_ROOT / "baichuan2-7b-chat-ggml.bin"
BAICHUAN2_13B_MODEL_PATH = PROJECT_ROOT / "baichuan2-13b-chat-ggml.bin"
INTERNLM7B_MODEL_PATH = PROJECT_ROOT / "internlm-chat-7b-ggml.bin"
INTERNLM20B_MODEL_PATH = PROJECT_ROOT / "internlm-chat-20b-ggml.bin"

# 测试 chatglm_cpp 模块的版本
def test_chatglm_version():
    print(chatglm_cpp.__version__)

# 定义检查管道的函数
def check_pipeline(model_path, prompt, target, gen_kwargs={}):
    # 创建用户消息列表
    messages = [chatglm_cpp.ChatMessage(role="user", content=prompt)]

    # 创建管道对象
    pipeline = chatglm_cpp.Pipeline(model_path)
    # 聊天并获取输出内容
    output = pipeline.chat(messages, do_sample=False, **gen_kwargs).content
    # 断言输出内容与目标内容相同
    assert output == target

    # 如果是 CHATGLM3_MODEL_PATH 模型，则进行特殊处理
    stream_output = pipeline.chat(messages, do_sample=False, stream=True, **gen_kwargs)
    stream_output = "".join([msg.content for msg in stream_output])
    if model_path == CHATGLM3_MODEL_PATH:
        # 为 ChatGLM3 进行特殊处理
        stream_output = stream_output.strip()
    assert stream_output == target

# 测试 ChatGLM 模型的管道
@pytest.mark.skipif(not CHATGLM_MODEL_PATH.exists(), reason="model file not found")
def test_chatglm_pipeline():
    check_pipeline(model_path=CHATGLM_MODEL_PATH, prompt="你好", target="你好👋！我是人工智能助手 ChatGLM-6B，很高兴见到你，欢迎问我任何问题。")

# 测试 ChatGLM2 模型的管道
@pytest.mark.skipif(not CHATGLM2_MODEL_PATH.exists(), reason="model file not found")
def test_chatglm2_pipeline():
    check_pipeline(model_path=CHATGLM2_MODEL_PATH, prompt="你好", target="你好👋！我是人工智能助手 ChatGLM2-6B，很高兴见到你，欢迎问我任何问题。")

# 测试 ChatGLM3 模型的管道
@pytest.mark.skipif(not CHATGLM3_MODEL_PATH.exists(), reason="model file not found")
def test_chatglm3_pipeline():
    # 调用 check_pipeline 函数，传入模型路径、提示语和目标文本作为参数
    check_pipeline(model_path=CHATGLM3_MODEL_PATH, prompt="你好", target="你好👋！我是人工智能助手 ChatGLM3-6B，很高兴见到你，欢迎问我任何问题。")
# 标记为跳过测试，如果 CODEGEEX2_MODEL_PATH 不存在则跳过
@pytest.mark.skipif(not CODEGEEX2_MODEL_PATH.exists(), reason="model file not found")
def test_codegeex2_pipeline():
    # 设置输入的提示和目标输出
    prompt = "# language: Python\n# write a bubble sort function\n"
    target = """

def bubble_sort(list):
    for i in range(len(list) - 1):
        for j in range(len(list) - 1):
            if list[j] > list[j + 1]:
                list[j], list[j + 1] = list[j + 1], list[j]
    return list


print(bubble_sort([5, 4, 3, 2, 1]))"""
    
    # 创建 chatglm_cpp.Pipeline 对象
    pipeline = chatglm_cpp.Pipeline(CODEGEEX2_MODEL_PATH)
    # 生成输出结果，禁用采样
    output = pipeline.generate(prompt, do_sample=False)
    # 断言输出结果与目标相同
    assert output == target

    # 生成输出结果，禁用采样，使用流式输出
    stream_output = pipeline.generate(prompt, do_sample=False, stream=True)
    stream_output = "".join(stream_output)
    # 断言流式输出结果与目标相同
    assert stream_output == target


# 标记为跳过测试，如果 BAICHUAN13B_MODEL_PATH 不存在则跳过
@pytest.mark.skipif(not BAICHUAN13B_MODEL_PATH.exists(), reason="model file not found")
def test_baichuan13b_pipeline():
    # 检查指定模型的输出结果是否符合预期
    check_pipeline(
        model_path=BAICHUAN13B_MODEL_PATH,
        prompt="你好呀",
        target="你好！很高兴见到你。请问有什么我可以帮助你的吗？",
        gen_kwargs=dict(repetition_penalty=1.1),
    )


# 标记为跳过测试，如果 BAICHUAN2_7B_MODEL_PATH 不存在则跳过
@pytest.mark.skipif(not BAICHUAN2_7B_MODEL_PATH.exists(), reason="model file not found")
def test_baichuan2_7b_pipeline():
    # 检查指定模型的输出结果是否符合预期
    check_pipeline(
        model_path=BAICHUAN2_7B_MODEL_PATH,
        prompt="你好呀",
        target="你好！很高兴为你服务。请问有什么问题我可以帮助你解决？",
        gen_kwargs=dict(repetition_penalty=1.05),
    )


# 标记为跳过测试，如果 BAICHUAN2_13B_MODEL_PATH 不存在则跳过
@pytest.mark.skipif(not BAICHUAN2_13B_MODEL_PATH.exists(), reason="model file not found")
def test_baichuan2_13b_pipeline():
    # 检查指定模型的输出结果是否符合预期
    check_pipeline(
        model_path=BAICHUAN2_13B_MODEL_PATH,
        prompt="你好呀",
        target="你好！很高兴见到你。请问有什么我可以帮助你的吗？",
        gen_kwargs=dict(repetition_penalty=1.05),
    )


# 标记为跳过测试，如果 INTERNLM7B_MODEL_PATH 不存在则跳过
@pytest.mark.skipif(not INTERNLM7B_MODEL_PATH.exists(), reason="model file not found")
def test_internlm7b_pipeline():
    # 检查指定模型的输出结果是否符合预期
    check_pipeline(model_path=INTERNLM7B_MODEL_PATH, prompt="你好", target="你好，有什么我可以帮助你的吗？")


# 标记为跳过测试，如果 INTERNLM20B_MODEL_PATH 不存在则跳过
@pytest.mark.skipif(not INTERNLM20B_MODEL_PATH.exists(), reason="model file not found")
# 定义一个测试函数，用于测试INTERNLM20B模型的pipeline
def test_internlm20b_pipeline():
    # 调用check_pipeline函数，传入INTERNLM20B_MODEL_PATH作为模型路径，"你好"作为输入提示，"你好！有什么我可以帮助你的吗？"作为目标输出
    check_pipeline(model_path=INTERNLM20B_MODEL_PATH, prompt="你好", target="你好！有什么我可以帮助你的吗？")
```