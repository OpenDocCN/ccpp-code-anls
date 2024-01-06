# `PowerInfer\examples\server\api_like_OAI.py`

```
#!/usr/bin/env python3
# 指定脚本解释器为 Python3

import argparse
# 导入命令行参数解析模块
from flask import Flask, jsonify, request, Response
# 从 Flask 框架中导入 Flask、jsonify、request、Response
import urllib.parse
# 导入用于解析 URL 的模块
import requests
# 导入用于发送 HTTP 请求的模块
import time
# 导入时间模块
import json
# 导入 JSON 模块

app = Flask(__name__)
# 创建一个 Flask 应用实例
slot_id = -1
# 初始化 slot_id 变量为 -1

parser = argparse.ArgumentParser(description="An example of using server.cpp with a similar API to OAI. It must be used together with server.cpp.")
# 创建一个参数解析器对象，设置描述信息
parser.add_argument("--user-name", type=str, help="USER name in chat completions(default: '\\nUSER: ')", default="\\nUSER: ")
# 添加一个命令行参数，指定用户名称，默认为 '\\nUSER: '
parser.add_argument("--ai-name", type=str, help="ASSISTANT name in chat completions(default: '\\nASSISTANT: ')", default="\\nASSISTANT: ")
# 添加一个命令行参数，指定助手名称，默认为 '\\nASSISTANT: '
parser.add_argument("--system-name", type=str, help="SYSTEM name in chat completions(default: '\\nASSISTANT's RULE: ')", default="\\nASSISTANT's RULE: ")
# 添加一个命令行参数，指定系统名称，默认为 '\\nASSISTANT's RULE: '
parser.add_argument("--stop", type=str, help="the end of response in chat completions(default: '</s>')", default="</s>")
# 添加一个命令行参数，指定对话结束标志，默认为 '</s>'
parser.add_argument("--llama-api", type=str, help="Set the address of server.cpp in llama.cpp(default: http://127.0.0.1:8080)", default='http://127.0.0.1:8080')
# 添加一个命令行参数，指定 llama.cpp 中 server.cpp 的地址，默认为 'http://127.0.0.1:8080'
parser.add_argument("--api-key", type=str, help="Set the api key to allow only few user(default: NULL)", default="")
# 添加一个命令行参数，指定 API 密钥，允许少数用户访问，默认为空
parser.add_argument("--host", type=str, help="Set the ip address to listen.(default: 127.0.0.1)", default='127.0.0.1')
# 添加一个命令行参数，指定监听的 IP 地址，默认为 '127.0.0.1'
# 添加一个命令行参数，用于设置监听的端口号，默认为8081
parser.add_argument("--port", type=int, help="Set the port to listen.(default: 8081)", default=8081)

# 解析命令行参数
args = parser.parse_args()

# 检查给定的 JSON 对象中是否存在指定的键
def is_present(json, key):
    try:
        buf = json[key]
    except KeyError:
        return False
    if json[key] == None:
        return False
    return True

# 将聊天消息转换为提示
def convert_chat(messages):
    # 从命令行参数中获取聊天提示，并替换其中的换行符
    prompt = "" + args.chat_prompt.replace("\\n", "\n")

    # 从命令行参数中获取系统名称，并替换其中的换行符
    system_n = args.system_name.replace("\\n", "\n")
    # 从命令行参数中获取用户名称，并替换其中的换行符
    user_n = args.user_name.replace("\\n", "\n")
    # 从命令行参数中获取 AI 名称，并替换其中的换行符
    ai_n = args.ai_name.replace("\\n", "\n")
# 将字符串中的"\n"替换为换行符"\n"
stop = args.stop.replace("\\n", "\n")

# 遍历消息列表
for line in messages:
    # 如果消息角色为系统，则将系统消息内容添加到提示中
    if (line["role"] == "system"):
        prompt += f"{system_n}{line['content']}"
    # 如果消息角色为用户，则将用户消息内容添加到提示中
    if (line["role"] == "user"):
        prompt += f"{user_n}{line['content']}"
    # 如果消息角色为助手，则将助手消息内容添加到提示中，并在末尾添加停止词
    if (line["role"] == "assistant"):
        prompt += f"{ai_n}{line['content']}{stop}"
# 去除提示末尾的助手标识
prompt += ai_n.rstrip()

# 返回提示
return prompt

# 根据参数生成postData字典
def make_postData(body, chat=False, stream=False):
    postData = {}
    # 如果是聊天模式，则将消息转换为聊天格式
    if (chat):
        postData["prompt"] = convert_chat(body["messages"])
    # 否则直接使用原始提示
    else:
        postData["prompt"] = body["prompt"]
# 如果请求体中包含温度信息，则将其添加到postData字典中
if(is_present(body, "temperature")): postData["temperature"] = body["temperature"]
# 如果请求体中包含top_k信息，则将其添加到postData字典中
if(is_present(body, "top_k")): postData["top_k"] = body["top_k"]
# 如果请求体中包含top_p信息，则将其添加到postData字典中
if(is_present(body, "top_p")): postData["top_p"] = body["top_p"]
# 如果请求体中包含max_tokens信息，则将其添加到postData字典中
if(is_present(body, "max_tokens")): postData["n_predict"] = body["max_tokens"]
# 如果请求体中包含presence_penalty信息，则将其添加到postData字典中
if(is_present(body, "presence_penalty")): postData["presence_penalty"] = body["presence_penalty"]
# 如果请求体中包含frequency_penalty信息，则将其添加到postData字典中
if(is_present(body, "frequency_penalty")): postData["frequency_penalty"] = body["frequency_penalty"]
# 如果请求体中包含repeat_penalty信息，则将其添加到postData字典中
if(is_present(body, "repeat_penalty")): postData["repeat_penalty"] = body["repeat_penalty"]
# 如果请求体中包含mirostat信息，则将其添加到postData字典中
if(is_present(body, "mirostat")): postData["mirostat"] = body["mirostat"]
# 如果请求体中包含mirostat_tau信息，则将其添加到postData字典中
if(is_present(body, "mirostat_tau")): postData["mirostat_tau"] = body["mirostat_tau"]
# 如果请求体中包含mirostat_eta信息，则将其添加到postData字典中
if(is_present(body, "mirostat_eta")): postData["mirostat_eta"] = body["mirostat_eta"]
# 如果请求体中包含seed信息，则将其添加到postData字典中
if(is_present(body, "seed")): postData["seed"] = body["seed"]
# 如果请求体中包含logit_bias信息，则将其添加到postData字典中
if(is_present(body, "logit_bias")): postData["logit_bias"] = [[int(token), body["logit_bias"][token]] for token in body["logit_bias"].keys()]
# 如果命令行参数中包含stop信息，则将其添加到postData字典中
if (args.stop != ""):
    postData["stop"] = [args.stop]
# 如果命令行参数中不包含stop信息，则将空列表添加到postData字典中
else:
    postData["stop"] = []
# 如果请求体中包含stop信息，则将其添加到postData字典中
if(is_present(body, "stop")): postData["stop"] += body["stop"]
# 将n_keep设置为-1
postData["n_keep"] = -1
# 将stream添加到postData字典中
postData["stream"] = stream
# 将cache_prompt设置为True
postData["cache_prompt"] = True
# 将slot_id赋值给postData字典的"slot_id"键
postData["slot_id"] = slot_id
# 返回postData字典
return postData

# 根据传入的数据和参数生成resData字典
def make_resData(data, chat=False, promptToken=[]):
    # 初始化resData字典
    resData = {
        "id": "chatcmpl" if (chat) else "cmpl",  # 根据chat参数确定id的取值
        "object": "chat.completion" if (chat) else "text_completion",  # 根据chat参数确定object的取值
        "created": int(time.time()),  # 设置created为当前时间的时间戳
        "truncated": data["truncated"],  # 设置truncated为传入data字典的"truncated"键的值
        "model": "LLaMA_CPP",  # 设置model为固定值"LLaMA_CPP"
        "usage": {
            "prompt_tokens": data["tokens_evaluated"],  # 设置prompt_tokens为传入data字典的"tokens_evaluated"键的值
            "completion_tokens": data["tokens_predicted"],  # 设置completion_tokens为传入data字典的"tokens_predicted"键的值
            "total_tokens": data["tokens_evaluated"] + data["tokens_predicted"]  # 设置total_tokens为tokens_evaluated和tokens_predicted之和
        }
    }
    # 如果promptToken列表不为空，则将其赋值给resData字典的"promptToken"键
    if (len(promptToken) != 0):
        resData["promptToken"] = promptToken
    # 如果chat为True，则添加注释
    if (chat):
        # 只支持一个选择
# 设置resData字典中的choices字段，包含一个字典列表
resData["choices"] = [{
    # 设置选项的索引
    "index": 0,
    # 设置消息内容，包括角色和内容
    "message": {
        "role": "assistant",
        "content": data["content"],
    },
    # 设置结束原因，如果遇到停止标记或者达到最大长度，则停止
    "finish_reason": "stop" if (data["stopped_eos"] or data["stopped_word"]) else "length"
}]
# 如果不是chat模式，只支持一个选项
else:
    # 设置resData字典中的choices字段，包含一个字典列表
    resData["choices"] = [{
        # 设置选项的文本内容
        "text": data["content"],
        # 设置选项的索引
        "index": 0,
        # 设置logprobs为None
        "logprobs": None,
        # 设置结束原因，如果遇到停止标记或者达到最大长度，则停止
        "finish_reason": "stop" if (data["stopped_eos"] or data["stopped_word"]) else "length"
    }]
# 返回结果字典
return resData

# 定义make_resData_stream函数，包含data、chat、time_now和start参数
def make_resData_stream(data, chat=False, time_now = 0, start=False):
    # 初始化resData字典
    resData = {
# 根据条件判断设置 id 字段，如果 chat 存在则为 "chatcmpl"，否则为 "cmpl"
"id": "chatcmpl" if (chat) else "cmpl",
# 根据条件判断设置 object 字段，如果 chat 存在则为 "chat.completion.chunk"，否则为 "text_completion.chunk"
"object": "chat.completion.chunk" if (chat) else "text_completion.chunk",
# 设置 created 字段为当前时间
"created": time_now,
# 设置 model 字段为 "LLaMA_CPP"
"model": "LLaMA_CPP",
# 设置 choices 字段为包含一个字典的列表
"choices": [
    {
        # 设置 finish_reason 字段为 None
        "finish_reason": None,
        # 设置 index 字段为 0
        "index": 0
    }
]
# 从 data 中获取 slot_id 字段的值
slot_id = data["slot_id"]
# 如果 chat 存在
if (chat):
    # 如果 start 存在
    if (start):
        # 设置 resData["choices"][0]["delta"] 字段为包含 role 字段的字典
        resData["choices"][0]["delta"] =  {
            "role": "assistant"
        }
    # 如果 start 不存在
    else:
        # 设置 resData["choices"][0]["delta"] 字段为包含 content 字段的字典
        resData["choices"][0]["delta"] =  {
            "content": data["content"]
# 定义 chat_completions 函数，处理 POST 请求
@app.route('/chat/completions', methods=['POST'])
@app.route('/v1/chat/completions', methods=['POST'])
def chat_completions():
    # 检查 API 密钥是否匹配，若不匹配则返回 403 错误
    if (args.api_key != "" and request.headers["Authorization"].split()[1] != args.api_key):
        return Response(status=403)
    # 获取请求的 JSON 数据
    body = request.get_json()
    # 初始化 stream 和 tokenize 变量
    stream = False
    tokenize = False
    # 检查请求中是否包含 stream 字段，若包含则将 stream 变量设置为对应值
    if(is_present(body, "stream")): stream = body["stream"]
# 如果请求体中包含"tokenize"字段，则将其赋值给tokenize变量
if(is_present(body, "tokenize")): tokenize = body["tokenize"]
# 根据请求体生成postData，chat参数为True，stream参数为stream的值
postData = make_postData(body, chat=True, stream=stream)

# 初始化promptToken为空列表
promptToken = []
# 如果tokenize为True，则发送请求获取tokenData，并将其tokens赋值给promptToken
if (tokenize):
    tokenData = requests.request("POST", urllib.parse.urljoin(args.llama_api, "/tokenize"), data=json.dumps({"content": postData["prompt"]})).json()
    promptToken = tokenData["tokens"]

# 如果stream为False，则发送请求获取数据，并打印返回的json数据，然后生成resData并返回
if (not stream):
    data = requests.request("POST", urllib.parse.urljoin(args.llama_api, "/completion"), data=json.dumps(postData))
    print(data.json())
    resData = make_resData(data.json(), chat=True, promptToken=promptToken)
    return jsonify(resData)
# 如果stream为True，则定义一个生成器函数generate
else:
    def generate():
        # 发送请求获取数据，并设置stream为True
        data = requests.request("POST", urllib.parse.urljoin(args.llama_api, "/completion"), data=json.dumps(postData), stream=True)
        # 获取当前时间戳
        time_now = int(time.time())
        # 生成初始的resData并返回
        resData = make_resData_stream({}, chat=True, time_now=time_now, start=True)
        yield 'data: {}\n'.format(json.dumps(resData))
        # 遍历返回的数据流
        for line in data.iter_lines():
# 如果读取到行数据
if line:
    # 将行数据解码为 UTF-8 格式
    decoded_line = line.decode('utf-8')
    # 根据解码后的数据生成响应数据流
    resData = make_resData_stream(json.loads(decoded_line[6:]), chat=True, time_now=time_now)
    # 生成数据流并返回
    yield 'data: {}\n'.format(json.dumps(resData))

# 返回响应对象，内容为生成的数据流，类型为 text/event-stream
return Response(generate(), mimetype='text/event-stream')

# 定义处理完成请求的函数
@app.route('/completions', methods=['POST'])
@app.route('/v1/completions', methods=['POST'])
def completion():
    # 如果 API 密钥不为空且请求头中的授权信息不等于 API 密钥，则返回状态码 403
    if (args.api_key != "" and request.headers["Authorization"].split()[1] != args.api_key):
        return Response(status=403)
    # 获取请求体的 JSON 数据
    body = request.get_json()
    stream = False
    tokenize = False
    # 如果请求体中包含 "stream" 字段，则将 stream 设置为对应值
    if(is_present(body, "stream")): stream = body["stream"]
    # 如果请求体中包含 "tokenize" 字段，则将 tokenize 设置为对应值
    if(is_present(body, "tokenize")): tokenize = body["tokenize"]
    # 根据请求体生成 POST 数据
    postData = make_postData(body, chat=False, stream=stream)

    # 初始化 promptToken 列表
    promptToken = []
    # 如果需要进行标记化处理
    if (tokenize):
        # 发送 POST 请求，将内容发送给 llama_api 的 /tokenize 接口，并获取返回的 token 数据
        tokenData = requests.request("POST", urllib.parse.urljoin(args.llama_api, "/tokenize"), data=json.dumps({"content": postData["prompt"]})).json()
        # 获取返回的 token 数据中的 tokens 字段
        promptToken = tokenData["tokens"]

    # 如果不需要流式处理
    if (not stream):
        # 发送 POST 请求，将 postData 发送给 llama_api 的 /completion 接口
        data = requests.request("POST", urllib.parse.urljoin(args.llama_api, "/completion"), data=json.dumps(postData))
        # 打印返回的数据
        print(data.json())
        # 将返回的数据转换成 resData，并设置 chat 为 False，传入 promptToken
        resData = make_resData(data.json(), chat=False, promptToken=promptToken)
        # 返回 JSON 格式的 resData
        return jsonify(resData)
    else:
        # 如果需要流式处理
        def generate():
            # 发送 POST 请求，将 postData 发送给 llama_api 的 /completion 接口，并设置 stream 为 True
            data = requests.request("POST", urllib.parse.urljoin(args.llama_api, "/completion"), data=json.dumps(postData), stream=True)
            # 获取当前时间戳
            time_now = int(time.time())
            # 遍历返回的数据流
            for line in data.iter_lines():
                if line:
                    # 将每行数据解码为 UTF-8 格式
                    decoded_line = line.decode('utf-8')
                    # 将解码后的数据转换成 resData，并设置 chat 为 False，传入 time_now
                    resData = make_resData_stream(json.loads(decoded_line[6:]), chat=False, time_now=time_now)
                    # 以事件流格式返回数据
                    yield 'data: {}\n'.format(json.dumps(resData))
        # 返回事件流响应
        return Response(generate(), mimetype='text/event-stream')
# 如果当前脚本被直接执行，则运行应用程序，使用给定的主机和端口参数。
```