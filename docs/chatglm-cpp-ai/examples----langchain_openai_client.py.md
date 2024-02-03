# `chatglm.cpp\examples\langchain_openai_client.py`

```
# 从 langchain.chat_models 模块中导入 ChatOpenAI 类
from langchain.chat_models import ChatOpenAI

# 创建 ChatOpenAI 类的实例对象
chat_model = ChatOpenAI()
# 调用 ChatOpenAI 类的 predict 方法，传入文本 "你好" 和最大 token 数 2048，并打印返回结果
print(chat_model.predict(text="你好", max_tokens=2048))
```