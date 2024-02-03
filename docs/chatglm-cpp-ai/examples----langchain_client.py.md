# `chatglm.cpp\examples\langchain_client.py`

```cpp
# 从 langchain.llms 模块中导入 ChatGLM 类
from langchain.llms import ChatGLM

# 创建 ChatGLM 类的实例，指定参数：endpoint_url为"http://127.0.0.1:8000"，max_token为2048，top_p为0.7，temperature为0.95，with_history为False
llm = ChatGLM(endpoint_url="http://127.0.0.1:8000", max_token=2048, top_p=0.7, temperature=0.95, with_history=False)

# 调用 llm 实例的 predict 方法，传入参数"你好"，并打印返回结果
print(llm.predict("你好"))
```