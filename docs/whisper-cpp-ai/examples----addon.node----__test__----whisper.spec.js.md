# `whisper.cpp\examples\addon.node\__test__\whisper.spec.js`

```cpp
// 引入 Node.js 内置的 path 模块
const path = require("path");
// 引入 whisper-addon 模块，使用绝对路径
const { whisper } = require(path.join(
  __dirname,
  "../../../build/Release/whisper-addon"
));
// 引入 Node.js 内置的 util 模块中的 promisify 方法
const { promisify } = require("util");

// 使用 promisify 方法将 whisper 函数转换为 Promise 风格的函数
const whisperAsync = promisify(whisper);

// 定义 whisper 函数的参数对象
const whisperParamsMock = {
  language: "en",
  // 指定模型文件的路径
  model: path.join(__dirname, "../../../models/ggml-base.en.bin"),
  // 指定输入音频文件的路径
  fname_inp: path.join(__dirname, "../../../samples/jfk.wav"),
  // 使用 GPU 运行
  use_gpu: true,
};

// 定义测试套件描述
describe("Run whisper.node", () => {
    // 定义测试用例，异步执行
    test("it should receive a non-empty value", async () => {
        // 调用 whisperAsync 函数，传入参数对象，获取结果
        let result = await whisperAsync(whisperParamsMock);

        // 断言结果的长度大于 0
        expect(result.length).toBeGreaterThan(0);
    }, 10000); // 设置超时时间为 10000 毫秒
});
```