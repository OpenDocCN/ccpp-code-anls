# `PowerInfer\examples\server\public\completion.js`

```cpp
const paramDefaults = {
  stream: true,  // 设置默认参数，流式生成
  n_predict: 500,  // 设置默认参数，预测数量
  temperature: 0.2,  // 设置默认参数，温度
  stop: ["</s>"]  // 设置默认参数，停止标志
};

let generation_settings = null;  // 初始化生成设置为null


// Completes the prompt as a generator. Recommended for most use cases.
//
// Example:
//
//    import { llama } from '/completion.js'
//
//    const request = llama("Tell me a joke", {n_predict: 800})
//    for await (const chunk of request) {
//      document.write(chunk.data.content)
//    }
//
export async function* llama(prompt, params = {}, config = {}) {
  let controller = config.controller;  // 从配置中获取控制器

  if (!controller) {
    controller = new AbortController();  // 如果没有控制器，则创建一个新的AbortController
  }

  const completionParams = { ...paramDefaults, ...params, prompt };  // 合并默认参数和传入参数，以及提示

  const response = await fetch("/completion", {  // 发送POST请求到/completion
    method: 'POST',
    body: JSON.stringify(completionParams),  // 将参数转换为JSON字符串作为请求体
    headers: {
      'Connection': 'keep-alive',  // 设置连接为持续连接
      'Content-Type': 'application/json',  // 设置请求体类型为JSON
      'Accept': 'text/event-stream'  // 设置接受类型为文本事件流
    },
    signal: controller.signal,  // 传入控制器的信号
  });

  const reader = response.body.getReader();  // 获取响应体的阅读器
  const decoder = new TextDecoder();  // 创建文本解码器

  let content = "";  // 初始化内容为空字符串
  let leftover = ""; // Buffer for partially read lines  // 初始化剩余内容为空字符串，用于部分读取的行的缓冲区

  try {
    let cont = true;  // 初始化cont为true，用于控制循环
    // 当 cont 为真时执行循环
    while (cont) {
      // 使用异步方式读取数据流
      const result = await reader.read();
      // 如果读取结束，则跳出循环
      if (result.done) {
        break;
      }

      // 将任何剩余数据添加到当前数据块中
      const text = leftover + decoder.decode(result.value);

      // 检查最后一个字符是否是换行符
      const endsWithLineBreak = text.endsWith('\n');

      // 将文本分割成行
      let lines = text.split('\n');

      // 如果文本不以换行符结尾，则最后一行是不完整的
      // 将其存储在剩余数据中，以便添加到下一个数据块中
      if (!endsWithLineBreak) {
        leftover = lines.pop();
      } else {
        leftover = ""; // 如果末尾有换行符，则重置剩余数据
      }

      // 解析所有 SSE 事件并将其添加到结果中
      const regex = /^(\S+):\s(.*)$/gm;
      for (const line of lines) {
        const match = regex.exec(line);
        if (match) {
          result[match[1]] = match[2]
          // 由于我们知道这是 llama.cpp，让我们只解码数据中的 JSON
          if (result.data) {
            result.data = JSON.parse(result.data);
            content += result.data.content;

            // 产生结果
            yield result;

            // 如果我们从服务器得到一个停止令牌，我们将在这里中断
            if (result.data.stop) {
              if (result.data.generation_settings) {
                generation_settings = result.data.generation_settings;
              }
              cont = false;
              break;
            }
          }
        }
      }
    }
  } catch (e) {
    // 如果捕获到异常且异常类型不是 AbortError，则打印错误信息
    if (e.name !== 'AbortError') {
      console.error("llama error: ", e);
    }
    // 抛出异常
    throw e;
  }
  finally {
    // 终止控制器
    controller.abort();
  }

  // 返回内容
  return content;
// Call llama, return an event target that you can subcribe to
//
// Example:
//
//    import { llamaEventTarget } from '/completion.js'
//
//    const conn = llamaEventTarget(prompt)
//    conn.addEventListener("message", (chunk) => {
//      document.write(chunk.detail.content)
//    })
//
// 从 llama 调用，返回一个可以订阅的事件目标
//
// 示例：
//
//    import { llamaEventTarget } from '/completion.js'
//
//    const conn = llamaEventTarget(prompt)
//    conn.addEventListener("message", (chunk) => {
//      document.write(chunk.detail.content)
//    })
//
export const llamaEventTarget = (prompt, params = {}, config = {}) => {
  const eventTarget = new EventTarget();
  (async () => {
    let content = "";
    for await (const chunk of llama(prompt, params, config)) {
      if (chunk.data) {
        content += chunk.data.content;
        eventTarget.dispatchEvent(new CustomEvent("message", { detail: chunk.data }));
      }
      if (chunk.data.generation_settings) {
        eventTarget.dispatchEvent(new CustomEvent("generation_settings", { detail: chunk.data.generation_settings }));
      }
      if (chunk.data.timings) {
        eventTarget.dispatchEvent(new CustomEvent("timings", { detail: chunk.data.timings }));
      }
    }
    eventTarget.dispatchEvent(new CustomEvent("done", { detail: { content } }));
  })();
  return eventTarget;
}

// Call llama, return a promise that resolves to the completed text. This does not support streaming
//
// Example:
//
//     llamaPromise(prompt).then((content) => {
//       document.write(content)
//     })
//
//     or
//
//     const content = await llamaPromise(prompt)
//     document.write(content)
//
// 从 llama 调用，返回一个解析为完成文本的 promise。这不支持流式传输
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
export const llamaPromise = (prompt, params = {}, config = {}) => {
  return new Promise(async (resolve, reject) => {
    let content = "";
    try {
      for await (const chunk of llama(prompt, params, config)) {
        content += chunk.data.content;
      }
      resolve(content);
    } catch (error) {
      reject(error);
    }
  });
};

/**
 * (deprecated)
 */
export const llamaComplete = async (params, controller, callback) => {
  for await (const chunk of llama(params.prompt, params, { controller })) {
    callback(chunk);
  }
}
// 从服务器获取模型信息。这对于获取上下文窗口等信息非常有用。
export const llamaModelInfo = async () => {
  // 如果生成设置不存在，则通过fetch方法获取"/model.json"的响应并将其转换为JSON格式
  if (!generation_settings) {
    generation_settings = await fetch("/model.json").then(r => r.json());
  }
  // 返回生成设置
  return generation_settings;
}
```