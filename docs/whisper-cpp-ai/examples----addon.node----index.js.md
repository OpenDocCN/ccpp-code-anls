# `whisper.cpp\examples\addon.node\index.js`

```cpp
// 引入 Node.js 的 path 模块
const path = require("path");
// 从指定路径引入 whisper 函数
const { whisper } = require(path.join(
  __dirname,
  "../../build/Release/whisper-addon"
));
// 引入 Node.js 的 promisify 函数
const { promisify } = require("util");

// 使用 promisify 将 whisper 函数转换为 Promise 风格的函数
const whisperAsync = promisify(whisper);

// 定义 whisper 函数的参数对象
const whisperParams = {
  language: "en",
  model: path.join(__dirname, "../../models/ggml-base.en.bin"),
  fname_inp: "../../samples/jfk.wav",
  use_gpu: true,
};

// 从命令行参数中提取参数
const arguments = process.argv.slice(2);
// 将参数转换为对象形式
const params = Object.fromEntries(
  arguments.reduce((pre, item) => {
    // 如果参数以 "--" 开头，则将其添加到对象中
    if (item.startsWith("--")) {
      return [...pre, item.slice(2).split("=")];
    }
    return pre;
  }, [])
);

// 遍历提取的参数，更新 whisperParams 对象
for (const key in params) {
  if (whisperParams.hasOwnProperty(key)) {
    whisperParams[key] = params[key];
  }
}

// 打印更新后的 whisperParams 对象
console.log("whisperParams =", whisperParams);

// 调用 whisperAsync 函数，并输出结果
whisperAsync(whisperParams).then((result) => {
  console.log(`Result from whisper: ${result}`);
});
```