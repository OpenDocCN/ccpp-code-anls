# `whisper.cpp\tests\test-whisper.js`

```cpp
// 引入whisper.js模块并创建工厂对象
var factory = require('../bindings/javascript/whisper.js')

// 使用工厂对象创建whisper实例
factory().then(function(whisper) {
    var fs = require('fs');

    // 为了避免读取WAV文件并依赖第三方包，直接读取32位浮点PCM数据。生成方法如下：
    //
    //   $ ffmpeg -i samples/jfk.wav -f f32le -acodec pcm_f32le samples/jfk.pcmf32
    //
    let fname_wav   = "../samples/jfk.pcmf32";
    let fname_model = "../models/ggml-base.en.bin";

    // 初始化whisper
    {
        // 从文件中读取二进制数据
        var model_data = fs.readFileSync(fname_model);
        if (model_data == null) {
            console.log("whisper: failed to read model file");
            process.exit(1);
        }

        // 将二进制数据写入WASM内存
        whisper.FS_createDataFile("/", "whisper.bin", model_data, true, true);

        // 初始化模型
        var ret = whisper.init("whisper.bin");
        if (ret == false) {
            console.log('whisper: failed to init');
            process.exit(1);
        }
    }

    // 转录wav文件
    {
        // 读取原始二进制数据
        var pcm_data = fs.readFileSync(fname_wav);
        if (pcm_data == null) {
            console.log("whisper: failed to read wav file");
            process.exit(1);
        }

        // 转换为32位浮点数组
        var pcm = new Float32Array(pcm_data.buffer);

        // 转录
        var ret = whisper.full_default(pcm, "en", false);
        if (ret != 0) {
            console.log("whisper: failed to transcribe");
            process.exit(1);
        }
    }

    // 释放内存
    {
        whisper.free();
    }
});
```