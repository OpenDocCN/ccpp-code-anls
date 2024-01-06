# `PowerInfer\examples\server\public\completion.js`

```
# 定义一个包含默认参数的常量对象
const paramDefaults = {
  stream: true,  # 是否以流的形式返回结果
  n_predict: 500,  # 预测的最大长度
  temperature: 0.2,  # 温度参数，控制生成文本的多样性
  stop: ["</s>"]  # 停止生成的标记
};

# 初始化生成设置为 null
let generation_settings = null;

# 完成提示作为生成器。推荐大多数使用情况。
#
# 示例：
#
#    import { llama } from '/completion.js'
#
#    const request = llama("Tell me a joke", {n_predict: 800})
#    for await (const chunk of request) {
#      document.write(chunk.data.content)
#    }
// 导出一个异步生成器函数 llama，接受 prompt、params 和 config 三个参数
export async function* llama(prompt, params = {}, config = {}) {
  // 从 config 参数中获取 controller，如果不存在则创建一个新的 AbortController
  let controller = config.controller;

  if (!controller) {
    controller = new AbortController();
  }

  // 将默认参数 paramDefaults 和传入的 params、prompt 合并成 completionParams 对象
  const completionParams = { ...paramDefaults, ...params, prompt };

  // 发起 POST 请求到 "/completion"，传入 completionParams 对象作为请求体
  const response = await fetch("/completion", {
    method: 'POST',
    body: JSON.stringify(completionParams),
    headers: {
      'Connection': 'keep-alive',
      'Content-Type': 'application/json',
      'Accept': 'text/event-stream'
    },
    // 传入 controller.signal 作为请求的信号
    signal: controller.signal,
  });
  // 获取响应体的读取器
  const reader = response.body.getReader();
  // 创建文本解码器
  const decoder = new TextDecoder();

  // 初始化内容为空字符串
  let content = "";
  // 初始化剩余数据为空字符串，用于存储部分读取的行
  let leftover = ""; 

  // 尝试执行以下代码块
  try {
    // 初始化循环条件为真
    let cont = true;

    // 当循环条件为真时执行以下代码块
    while (cont) {
      // 从读取器中读取数据
      const result = await reader.read();
      // 如果读取结束，则跳出循环
      if (result.done) {
        break;
      }

      // 将剩余数据与当前数据块合并
      const text = leftover + decoder.decode(result.value);

      // 检查最后一个字符是否是换行符
      // 检查文本是否以换行符结尾
      const endsWithLineBreak = text.endsWith('\n');

      // 将文本分割成行
      let lines = text.split('\n');

      // 如果文本不以换行符结尾，则最后一行是不完整的
      // 将其存储在 leftover 中，以便添加到下一块数据中
      if (!endsWithLineBreak) {
        leftover = lines.pop();
      } else {
        leftover = ""; // 如果文本以换行符结尾，则重置 leftover
      }

      // 解析所有的 SSE 事件并将它们添加到结果中
      const regex = /^(\S+):\s(.*)$/gm;
      for (const line of lines) {
        const match = regex.exec(line);
        if (match) {
          result[match[1]] = match[2]
          // 由于我们知道这是 llama.cpp，让我们只解码数据中的 JSON
          // 检查result对象中是否有data属性
          if (result.data) {
            // 将result.data转换为JSON格式
            result.data = JSON.parse(result.data);
            // 将result.data中的content属性添加到content变量中
            content += result.data.content;

            // 产生一个结果
            yield result;

            // 如果从服务器得到一个停止令牌，我们将在这里中断
            if (result.data.stop) {
              // 如果result.data中有generation_settings属性，将其赋值给generation_settings变量
              if (result.data.generation_settings) {
                generation_settings = result.data.generation_settings;
              }
              // 将cont变量设为false
              cont = false;
              // 跳出循环
              break;
            }
          }
        }
      }
    }
  } catch (e) {
// 如果捕获的错误不是 AbortError，则打印错误信息
if (e.name !== 'AbortError') {
  console.error("llama error: ", e);
}
// 抛出捕获的错误
throw e;
// 最终执行，取消控制器
finally {
  controller.abort();
}

// 返回内容
return content;
}

// 调用 llama，返回一个可以订阅的事件目标
//
// 示例:
//
//    import { llamaEventTarget } from '/completion.js'
//
//    const conn = llamaEventTarget(prompt)
//    conn.addEventListener("message", (chunk) => {
// 导出名为 llamaEventTarget 的函数，接受 prompt、params 和 config 三个参数
export const llamaEventTarget = (prompt, params = {}, config = {}) => {
  // 创建一个事件目标对象
  const eventTarget = new EventTarget();
  // 异步函数
  (async () => {
    // 初始化内容为空字符串
    let content = "";
    // 使用 for await 循环遍历 llama 函数返回的异步迭代器
    for await (const chunk of llama(prompt, params, config)) {
      // 如果 chunk 中有数据
      if (chunk.data) {
        // 将数据内容添加到 content 中
        content += chunk.data.content;
        // 分发自定义事件 "message"，并传递 chunk.data 作为 detail
        eventTarget.dispatchEvent(new CustomEvent("message", { detail: chunk.data }));
      }
      // 如果 chunk 中有 generation_settings
      if (chunk.data.generation_settings) {
        // 分发自定义事件 "generation_settings"，并传递 chunk.data.generation_settings 作为 detail
        eventTarget.dispatchEvent(new CustomEvent("generation_settings", { detail: chunk.data.generation_settings }));
      }
      // 如果 chunk 中有 timings
      if (chunk.data.timings) {
        // 分发自定义事件 "timings"，并传递 chunk.data.timings 作为 detail
        eventTarget.dispatchEvent(new CustomEvent("timings", { detail: chunk.data.timings }));
      }
    }
    // 分发自定义事件 "done"，并传递包含 content 的对象作为 detail
    eventTarget.dispatchEvent(new CustomEvent("done", { detail: { content } }));
  })();
}
// 调用 llama 函数，返回一个 Promise 对象，该对象在完成时解析为完成的文本。不支持流式处理
//
// 示例：
//
//     llamaPromise(prompt).then((content) => {
//       document.write(content)
//     })
//
//     或
//
//     const content = await llamaPromise(prompt)
//     document.write(content)
//
// 导出 llamaPromise 函数，该函数接受 prompt、params 和 config 作为参数
export const llamaPromise = (prompt, params = {}, config = {}) => {
  // 返回一个新的 Promise 对象，该对象在完成时解析为完成的文本
  return new Promise(async (resolve, reject) => {
    // 初始化 content 变量为空字符串
    let content = "";
# 尝试执行以下代码块，如果出现错误则捕获并拒绝
try {
  # 使用异步迭代器 llama 从 prompt、params 和 config 中获取数据块，将数据块的内容添加到 content 中
  for await (const chunk of llama(prompt, params, config)) {
    content += chunk.data.content;
  }
  # 解析并返回 content
  resolve(content);
} catch (error) {
  # 捕获错误并拒绝
  reject(error);
}
});

/**
 * (deprecated)  // 已废弃的函数
 */
# 异步函数 llamaComplete，接收 params、controller 和 callback 作为参数
export const llamaComplete = async (params, controller, callback) => {
  # 使用异步迭代器 llama 从 params.prompt、params 和 { controller } 中获取数据块，将数据块传递给回调函数 callback
  for await (const chunk of llama(params.prompt, params, { controller })) {
    callback(chunk);
  }
}
// 从服务器获取模型信息。这对于获取上下文窗口等信息非常有用。
export const llamaModelInfo = async () => {
  // 如果 generation_settings 不存在，则从 "/model.json" 获取并设置
  if (!generation_settings) {
    generation_settings = await fetch("/model.json").then(r => r.json());
  }
  // 返回 generation_settings
  return generation_settings;
}
```