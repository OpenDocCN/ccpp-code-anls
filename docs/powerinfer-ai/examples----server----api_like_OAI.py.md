# `PowerInfer\examples\server\api_like_OAI.py`

```
#!/usr/bin/env python3
# 指定脚本的解释器为 Python3

import argparse
# 导入 argparse 模块，用于解析命令行参数
from flask import Flask, jsonify, request, Response
# 从 flask 框架中导入 Flask、jsonify、request、Response 类
import urllib.parse
# 导入 urllib.parse 模块，用于解析 URL
import requests
# 导入 requests 模块，用于发送 HTTP 请求
import time
# 导入 time 模块，用于时间相关操作
import json
# 导入 json 模块，用于处理 JSON 数据

app = Flask(__name__)
# 创建一个 Flask 应用实例
slot_id = -1
# 初始化变量 slot_id 为 -1

parser = argparse.ArgumentParser(description="An example of using server.cpp with a similar API to OAI. It must be used together with server.cpp.")
# 创建一个参数解析器对象，设置描述信息
parser.add_argument("--chat-prompt", type=str, help="the top prompt in chat completions(default: 'A chat between a curious user and an artificial intelligence assistant. The assistant follows the given rules no matter what.\\n')", default='A chat between a curious user and an artificial intelligence assistant. The assistant follows the given rules no matter what.\\n')
# 添加一个命令行参数，设置聊天提示的默认值
parser.add_argument("--user-name", type=str, help="USER name in chat completions(default: '\\nUSER: ')", default="\\nUSER: ")
# 添加一个命令行参数，设置用户名称的默认值
parser.add_argument("--ai-name", type=str, help="ASSISTANT name in chat completions(default: '\\nASSISTANT: ')", default="\\nASSISTANT: ")
# 添加一个命令行参数，设置助手名称的默认值
parser.add_argument("--system-name", type=str, help="SYSTEM name in chat completions(default: '\\nASSISTANT's RULE: ')", default="\\nASSISTANT's RULE: ")
# 添加一个命令行参数，设置系统名称的默认值
parser.add_argument("--stop", type=str, help="the end of response in chat completions(default: '</s>')", default="</s>")
# 添加一个命令行参数，设置聊天结束标志的默认值
parser.add_argument("--llama-api", type=str, help="Set the address of server.cpp in llama.cpp(default: http://127.0.0.1:8080)", default='http://127.0.0.1:8080')
# 添加一个命令行参数，设置 llama.cpp 中 server.cpp 的地址的默认值
parser.add_argument("--api-key", type=str, help="Set the api key to allow only few user(default: NULL)", default="")
# 添加一个命令行参数，设置 API 密钥的默认值
parser.add_argument("--host", type=str, help="Set the ip address to listen.(default: 127.0.0.1)", default='127.0.0.1')
# 添加一个命令行参数，设置监听的 IP 地址的默认值
parser.add_argument("--port", type=int, help="Set the port to listen.(default: 8081)", default=8081)
# 添加一个命令行参数，设置监听的端口号的默认值

args = parser.parse_args()
# 解析命令行参数，并将结果存储在 args 变量中

def is_present(json, key):
    # 定义一个函数，用于判断 JSON 对象中是否存在指定的键
    try:
        buf = json[key]
        # 尝试获取指定键对应的值
    except KeyError:
        return False
    # 如果出现 KeyError 异常，说明指定键不存在，返回 False
    if json[key] == None:
        return False
    # 如果指定键对应的值为 None，返回 False
    return True
    # 否则返回 True

#convert chat to prompt
def convert_chat(messages):
    # 定义一个函数，用于将聊天消息转换为提示
    # 将输入的聊天提示中的"\n"替换为换行符，并赋值给prompt变量
    prompt = "" + args.chat_prompt.replace("\\n", "\n")

    # 将输入的系统名称中的"\n"替换为换行符，并赋值给system_n变量
    system_n = args.system_name.replace("\\n", "\n")
    # 将输入的用户名称中的"\n"替换为换行符，并赋值给user_n变量
    user_n = args.user_name.replace("\\n", "\n")
    # 将输入的AI名称中的"\n"替换为换行符，并赋值给ai_n变量
    ai_n = args.ai_name.replace("\\n", "\n")
    # 将输入的停止标志中的"\n"替换为换行符，并赋值给stop变量
    stop = args.stop.replace("\\n", "\n")

    # 遍历消息列表中的每条消息
    for line in messages:
        # 如果消息的角色是系统，则将系统名称和内容添加到prompt中
        if (line["role"] == "system"):
            prompt += f"{system_n}{line['content']}"
        # 如果消息的角色是用户，则将用户名称和内容添加到prompt中
        if (line["role"] == "user"):
            prompt += f"{user_n}{line['content']}"
        # 如果消息的角色是助手，则将AI名称、内容和停止标志添加到prompt中
        if (line["role"] == "assistant"):
            prompt += f"{ai_n}{line['content']}{stop}"
    # 将AI名称的右侧空格去除，并添加到prompt中
    prompt += ai_n.rstrip()

    # 返回拼接后的prompt
    return prompt
# 创建 postData 字典，用于存储请求数据
def make_postData(body, chat=False, stream=False):
    postData = {}
    # 如果 chat 参数为 True，则将 body["messages"] 转换为聊天格式的 prompt 存入 postData
    if (chat):
        postData["prompt"] = convert_chat(body["messages"])
    else:
        # 否则将 body["prompt"] 存入 postData
        postData["prompt"] = body["prompt"]
    # 如果 body 中存在 temperature，则将其存入 postData
    if(is_present(body, "temperature")): postData["temperature"] = body["temperature"]
    # 如果 body 中存在 top_k，则将其存入 postData
    if(is_present(body, "top_k")): postData["top_k"] = body["top_k"]
    # 如果 body 中存在 top_p，则将其存入 postData
    if(is_present(body, "top_p")): postData["top_p"] = body["top_p"]
    # 如果 body 中存在 max_tokens，则将其存入 postData
    if(is_present(body, "max_tokens")): postData["n_predict"] = body["max_tokens"]
    # 如果 body 中存在 presence_penalty，则将其存入 postData
    if(is_present(body, "presence_penalty")): postData["presence_penalty"] = body["presence_penalty"]
    # 如果 body 中存在 frequency_penalty，则将其存入 postData
    if(is_present(body, "frequency_penalty")): postData["frequency_penalty"] = body["frequency_penalty"]
    # 如果 body 中存在 repeat_penalty，则将其存入 postData
    if(is_present(body, "repeat_penalty")): postData["repeat_penalty"] = body["repeat_penalty"]
    # 如果 body 中存在 mirostat，则将其存入 postData
    if(is_present(body, "mirostat")): postData["mirostat"] = body["mirostat"]
    # 如果 body 中存在 mirostat_tau，则将其存入 postData
    if(is_present(body, "mirostat_tau")): postData["mirostat_tau"] = body["mirostat_tau"]
    # 如果 body 中存在 mirostat_eta，则将其存入 postData
    if(is_present(body, "mirostat_eta")): postData["mirostat_eta"] = body["mirostat_eta"]
    # 如果 body 中存在 seed，则将其存入 postData
    if(is_present(body, "seed")): postData["seed"] = body["seed"]
    # 如果 body 中存在 logit_bias，则将其存入 postData
    if(is_present(body, "logit_bias")): postData["logit_bias"] = [[int(token), body["logit_bias"][token]] for token in body["logit_bias"].keys()]
    # 如果 args.stop 不为空，则将其存入 postData["stop"]，否则将 postData["stop"] 置为空列表
    if (args.stop != ""):
        postData["stop"] = [args.stop]
    else:
        postData["stop"] = []
    # 如果 body 中存在 stop，则将其添加到 postData["stop"] 中
    if(is_present(body, "stop")): postData["stop"] += body["stop"]
    # 设置 n_keep 为 -1
    postData["n_keep"] = -1
    # 设置 stream 参数
    postData["stream"] = stream
    # 设置 cache_prompt 为 True
    postData["cache_prompt"] = True
    # 设置 slot_id
    postData["slot_id"] = slot_id
    # 返回 postData
    return postData

# 创建 resData 字典，用于存储响应数据
def make_resData(data, chat=False, promptToken=[]):
    # 创建一个包含响应数据的字典
    resData = {
        "id": "chatcmpl" if (chat) else "cmpl",  # 根据条件设置 id 字段的值
        "object": "chat.completion" if (chat) else "text_completion",  # 根据条件设置 object 字段的值
        "created": int(time.time()),  # 设置 created 字段为当前时间的时间戳
        "truncated": data["truncated"],  # 设置 truncated 字段为给定数据中的 truncated 值
        "model": "LLaMA_CPP",  # 设置 model 字段的值
        "usage": {
            "prompt_tokens": data["tokens_evaluated"],  # 设置 usage 字段中的 prompt_tokens 值
            "completion_tokens": data["tokens_predicted"],  # 设置 usage 字段中的 completion_tokens 值
            "total_tokens": data["tokens_evaluated"] + data["tokens_predicted"]  # 设置 usage 字段中的 total_tokens 值
        }
    }
    # 如果 promptToken 非空，则将其添加到 resData 字典中
    if (len(promptToken) != 0):
        resData["promptToken"] = promptToken
    # 如果 chat 为真，则添加特定格式的 choices 到 resData 字典中
    if (chat):
        # 只支持一个选项
        resData["choices"] = [{
            "index": 0,
            "message": {
                "role": "assistant",
                "content": data["content"],
            },
            "finish_reason": "stop" if (data["stopped_eos"] or data["stopped_word"]) else "length"
        }]
    # 如果 chat 为假，则添加特定格式的 choices 到 resData 字典中
    else:
        # 只支持一个选项
        resData["choices"] = [{
            "text": data["content"],
            "index": 0,
            "logprobs": None,
            "finish_reason": "stop" if (data["stopped_eos"] or data["stopped_word"]) else "length"
        }]
    # 返回包含响应数据的字典
    return resData
# 创建一个返回聊天或文本完成数据的函数，可以根据参数决定返回的数据类型
def make_resData_stream(data, chat=False, time_now = 0, start=False):
    # 初始化返回数据字典
    resData = {
        "id": "chatcmpl" if (chat) else "cmpl",  # 根据 chat 参数确定返回数据的 id
        "object": "chat.completion.chunk" if (chat) else "text_completion.chunk",  # 根据 chat 参数确定返回数据的 object
        "created": time_now,  # 设置返回数据的创建时间
        "model": "LLaMA_CPP",  # 设置返回数据的模型
        "choices": [  # 初始化返回数据的选择列表
            {
                "finish_reason": None,  # 初始化完成原因为 None
                "index": 0  # 初始化索引为 0
            }
        ]
    }
    # 从输入的数据中获取 slot_id
    slot_id = data["slot_id"]
    if (chat):  # 如果是聊天完成数据
        if (start):  # 如果是开始聊天
            resData["choices"][0]["delta"] =  {  # 设置返回数据的 delta 字段
                "role": "assistant"  # 设置角色为 assistant
            }
        else:  # 如果不是开始聊天
            resData["choices"][0]["delta"] =  {  # 设置返回数据的 delta 字段
                "content": data["content"]  # 设置内容为输入数据的内容
            }
            if (data["stop"]):  # 如果输入数据中包含停止标志
                resData["choices"][0]["finish_reason"] = "stop" if (data["stopped_eos"] or data["stopped_word"]) else "length"  # 根据停止原因设置完成原因
    else:  # 如果是文本完成数据
        resData["choices"][0]["text"] = data["content"]  # 设置返回数据的文本内容
        if (data["stop"]):  # 如果输入数据中包含停止标志
            resData["choices"][0]["finish_reason"] = "stop" if (data["stopped_eos"] or data["stopped_word"]) else "length"  # 根据停止原因设置完成原因

    return resData  # 返回构建好的返回数据字典


# 处理聊天完成请求的路由处理函数
@app.route('/chat/completions', methods=['POST'])
@app.route('/v1/chat/completions', methods=['POST'])
def chat_completions():
    if (args.api_key != "" and request.headers["Authorization"].split()[1] != args.api_key):  # 检查 API 密钥是否匹配
        return Response(status=403)  # 如果不匹配，返回 403 错误
    body = request.get_json()  # 获取请求的 JSON 数据
    stream = False  # 初始化 stream 参数为 False
    tokenize = False  # 初始化 tokenize 参数为 False
    if(is_present(body, "stream")): stream = body["stream"]  # 如果请求中包含 stream 参数，则设置为对应的值
    if(is_present(body, "tokenize")): tokenize = body["tokenize"]  # 如果请求中包含 tokenize 参数，则设置为对应的值
    postData = make_postData(body, chat=True, stream=stream)  # 根据请求数据构建 postData

    promptToken = []  # 初始化 promptToken 为空列表
    if (tokenize):  # 如果需要进行 tokenize
        tokenData = requests.request("POST", urllib.parse.urljoin(args.llama_api, "/tokenize"), data=json.dumps({"content": postData["prompt"]})).json()  # 发送 tokenize 请求并获取返回数据
        promptToken = tokenData["tokens"]  # 获取 token 数据中的 tokens 字段
    # 如果没有流，则发送 POST 请求到指定的 URL，获取数据并打印
    if (not stream):
        data = requests.request("POST", urllib.parse.urljoin(args.llama_api, "/completion"), data=json.dumps(postData))
        print(data.json())
        # 根据返回的数据生成响应数据，并返回 JSON 格式的响应
        resData = make_resData(data.json(), chat=True, promptToken=promptToken)
        return jsonify(resData)
    # 如果有流，则定义一个生成器函数
    else:
        def generate():
            # 发送带有流的 POST 请求到指定的 URL，获取数据
            data = requests.request("POST", urllib.parse.urljoin(args.llama_api, "/completion"), data=json.dumps(postData), stream=True)
            # 获取当前时间戳
            time_now = int(time.time())
            # 根据返回的数据生成初始响应数据，并以事件流的形式返回
            resData = make_resData_stream({}, chat=True, time_now=time_now, start=True)
            yield 'data: {}\n'.format(json.dumps(resData))
            # 遍历返回的数据流，解码并生成响应数据，以事件流的形式返回
            for line in data.iter_lines():
                if line:
                    decoded_line = line.decode('utf-8')
                    resData = make_resData_stream(json.loads(decoded_line[6:]), chat=True, time_now=time_now)
                    yield 'data: {}\n'.format(json.dumps(resData))
        # 返回生成的事件流响应
        return Response(generate(), mimetype='text/event-stream')
# 定义处理 POST 请求的路由，处理自动补全功能
@app.route('/completions', methods=['POST'])
@app.route('/v1/completions', methods=['POST'])
def completion():
    # 检查 API 密钥是否匹配，如果不匹配则返回 403 错误
    if (args.api_key != "" and request.headers["Authorization"].split()[1] != args.api_key):
        return Response(status=403)
    # 获取请求的 JSON 数据
    body = request.get_json()
    stream = False
    tokenize = False
    # 检查请求中是否包含 stream 和 tokenize 参数，并设置对应的变量
    if(is_present(body, "stream")): stream = body["stream"]
    if(is_present(body, "tokenize")): tokenize = body["tokenize"]
    # 根据请求数据生成 postData
    postData = make_postData(body, chat=False, stream=stream)

    promptToken = []
    # 如果需要进行 tokenize，则发送请求获取 tokenData，并提取 tokens
    if (tokenize):
        tokenData = requests.request("POST", urllib.parse.urljoin(args.llama_api, "/tokenize"), data=json.dumps({"content": postData["prompt"]})).json()
        promptToken = tokenData["tokens"]

    # 如果不需要进行 stream，则发送请求获取数据，并返回处理后的结果
    if (not stream):
        data = requests.request("POST", urllib.parse.urljoin(args.llama_api, "/completion"), data=json.dumps(postData))
        print(data.json())
        resData = make_resData(data.json(), chat=False, promptToken=promptToken)
        return jsonify(resData)
    # 如果需要进行 stream，则定义生成器函数，发送请求获取数据，并以事件流的形式返回结果
    else:
        def generate():
            data = requests.request("POST", urllib.parse.urljoin(args.llama_api, "/completion"), data=json.dumps(postData), stream=True)
            time_now = int(time.time())
            for line in data.iter_lines():
                if line:
                    decoded_line = line.decode('utf-8')
                    resData = make_resData_stream(json.loads(decoded_line[6:]), chat=False, time_now=time_now)
                    yield 'data: {}\n'.format(json.dumps(resData))
        return Response(generate(), mimetype='text/event-stream')

# 如果是主程序入口，则运行应用
if __name__ == '__main__':
    app.run(args.host, port=args.port)
```