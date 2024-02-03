# `whisper.cpp\bindings\java\src\main\java\io\github\ggerganov\whispercpp\WhisperCpp.java`

```cpp
package io.github.ggerganov.whispercpp;

import com.sun.jna.Native;
import com.sun.jna.Pointer;
import io.github.ggerganov.whispercpp.bean.WhisperSegment;
import io.github.ggerganov.whispercpp.params.WhisperContextParams;
import io.github.ggerganov.whispercpp.params.WhisperFullParams;
import io.github.ggerganov.whispercpp.params.WhisperSamplingStrategy;
import java.io.File;
import java.io.FileNotFoundException;
import java.io.IOException;
import java.util.ArrayList;
import java.util.List;

/**
 * Before calling most methods, you must call `initContext(modelPath)` to initialise the `ctx` Pointer.
 */
public class WhisperCpp implements AutoCloseable {
    private WhisperCppJnaLibrary lib = WhisperCppJnaLibrary.instance;
    private Pointer ctx = null;
    private Pointer paramsPointer = null;
    private Pointer greedyParamsPointer = null;
    private Pointer beamParamsPointer = null;

    public File modelDir() {
        // 获取模型目录路径
        String modelDirPath = System.getenv("XDG_CACHE_HOME");
        // 如果环境变量中没有指定模型目录路径，则使用默认路径
        if (modelDirPath == null) {
            modelDirPath = System.getProperty("user.home") + "/.cache";
        }

        // 返回模型目录的 File 对象
        return new File(modelDirPath, "whisper");
    }

    /**
     * @param modelPath - absolute path, or just the name (eg: "base", "base-en" or "base.en")
     */
    public void initContext(String modelPath) throws FileNotFoundException {
        // 使用默认参数初始化上下文
        initContextImpl(modelPath, getContextDefaultParams());
    }

    /**
     * @param modelPath - absolute path, or just the name (eg: "base", "base-en" or "base.en")
     * @param params - params to use when initialising the context
     */
    public void initContext(String modelPath, WhisperContextParams params) throws FileNotFoundException {
        // 使用指定参数初始化上下文
        initContextImpl(modelPath, params);
    }
    // 初始化上下文实现方法，根据模型路径和参数初始化上下文
    private void initContextImpl(String modelPath, WhisperContextParams params) throws FileNotFoundException {
        // 如果上下文对象不为空，则释放上下文对象
        if (ctx != null) {
            lib.whisper_free(ctx);
        }

        // 如果模型路径不包含斜杠和反斜杠，则进行处理
        if (!modelPath.contains("/") && !modelPath.contains("\\")) {
            // 如果模型路径不以".bin"结尾，则修改模型路径
            if (!modelPath.endsWith(".bin")) {
                modelPath = "ggml-" + modelPath.replace("-", ".") + ".bin";
            }

            // 获取模型目录下的绝对路径
            modelPath = new File(modelDir(), modelPath).getAbsolutePath();
        }

        // 使用模型路径和参数初始化上下文对象
        ctx = lib.whisper_init_from_file_with_params(modelPath, params);

        // 如果上下文对象为空，则抛出文件未找到异常
        if (ctx == null) {
            throw new FileNotFoundException(modelPath);
        }
    }

    /**
     * 提供可用于`whisper_init_from_file_with_params()`等方法的默认参数。
     * 因为此函数为参数分配内存，调用者必须调用以下之一：
     * - 调用`whisper_free_context_params()`
     * - `Native.free(Pointer.nativeValue(pointer));`
     */
    public WhisperContextParams getContextDefaultParams() {
        // 通过引用获取默认参数指针
        paramsPointer = lib.whisper_context_default_params_by_ref();
        // 根据参数指针创建参数对象
        WhisperContextParams params = new WhisperContextParams(paramsPointer);
        // 读取参数内容
        params.read();
        // 返回参数对象
        return params;
    }
    
    /**
     * 提供可用于`whisper_full()`等方法的默认参数。
     * 因为此函数为参数分配内存，调用者必须调用以下之一：
     * - 调用`whisper_free_params()`
     * - `Native.free(Pointer.nativeValue(pointer));`
     *
     * @param strategy - GREEDY
     */
    // 获取完整的默认参数对象，根据给定的采样策略
    public WhisperFullParams getFullDefaultParams(WhisperSamplingStrategy strategy) {
        Pointer pointer;

        // whisper_full_default_params_by_ref 分配内存，需要删除，所以每种策略只创建一个指针
        if (strategy == WhisperSamplingStrategy.WHISPER_SAMPLING_GREEDY) {
            // 如果是贪婪采样策略
            if (greedyParamsPointer == null) {
                // 如果贪婪参数指针为空，创建贪婪参数指针
                greedyParamsPointer = lib.whisper_full_default_params_by_ref(strategy.ordinal());
            }
            pointer = greedyParamsPointer;
        } else {
            // 如果是波束采样策略
            if (beamParamsPointer == null) {
                // 如果波束参数指针为空，创建波束参数指针
                beamParamsPointer = lib.whisper_full_default_params_by_ref(strategy.ordinal());
            }
            pointer = beamParamsPointer;
        }

        // 根据指针创建 WhisperFullParams 对象
        WhisperFullParams params = new WhisperFullParams(pointer);
        // 读取参数
        params.read();
        // 返回参数对象
        return params;
    }

    // 关闭方法
    @Override
    public void close() {
        // 释放上下文
        freeContext();
        // 释放参数
        freeParams();
        // 打印 Whisper 关闭信息
        System.out.println("Whisper closed");
    }

    // 释放上下文方法
    private void freeContext() {
        if (ctx != null) {
            // 释放上下文
            lib.whisper_free(ctx);
        }
    }

    // 释放参数方法
    private void freeParams() {
        if (paramsPointer != null) {
            // 如果参数指针不为空，释放参数指针
            Native.free(Pointer.nativeValue(paramsPointer));
            paramsPointer = null;
        }
        if (greedyParamsPointer != null) {
            // 如果贪婪参数指针不为空，释放贪婪参数指针
            Native.free(Pointer.nativeValue(greedyParamsPointer));
            greedyParamsPointer = null;
        }
        if (beamParamsPointer != null) {
            // 如果波束参数指针不为空，释放波束参数指针
            Native.free(Pointer.nativeValue(beamParamsPointer));
            beamParamsPointer = null;
        }
    }

    /**
     * 运行整个模型：PCM -> 对数梅尔频谱 -> 编码器 -> 解码器 -> 文本。
     * 对于相同上下文不是线程安全的
     * 使用指定的解码策略获取文本。
     */
    // 对完整的 Whisper 参数和音频数据进行转录，返回完整的文本结果
    public String fullTranscribe(WhisperFullParams whisperParams, float[] audioData) throws IOException {
        // 检查模型是否已初始化
        if (ctx == null) {
            throw new IllegalStateException("Model not initialised");
        }

        // 调用底层库进行完整转录，处理音频数据
        if (lib.whisper_full(ctx, whisperParams, audioData, audioData.length) != 0) {
            throw new IOException("Failed to process audio");
        }

        // 获取转录结果的段落数量
        int nSegments = lib.whisper_full_n_segments(ctx);

        // 创建一个 StringBuilder 用于存储完整文本结果
        StringBuilder str = new StringBuilder();

        // 遍历每个段落，获取文本并打印
        for (int i = 0; i < nSegments; i++) {
            String text = lib.whisper_full_get_segment_text(ctx, i);
            System.out.println("Segment:" + text);
            str.append(text);
        }

        // 返回完整文本结果
        return str.toString().trim();
    }
    
    // 对完整的 Whisper 参数和音频数据进行转录，返回带时间信息的段落列表
    public List<WhisperSegment> fullTranscribeWithTime(WhisperFullParams whisperParams, float[] audioData) throws IOException {
        // 检查模型是否已初始化
        if (ctx == null) {
            throw new IllegalStateException("Model not initialised");
        }

        // 调用底层库进行完整转录，处理音频数据
        if (lib.whisper_full(ctx, whisperParams, audioData, audioData.length) != 0) {
            throw new IOException("Failed to process audio");
        }

        // 获取转录结果的段落数量
        int nSegments = lib.whisper_full_n_segments(ctx);
        // 创建一个列表用于存储带时间信息的段落
        List<WhisperSegment> segments= new ArrayList<>(nSegments);

        // 遍历每个段落，获取时间信息、文本并添加到列表中
        for (int i = 0; i < nSegments; i++) {
            long t0 = lib.whisper_full_get_segment_t0(ctx, i);
            String text = lib.whisper_full_get_segment_text(ctx, i);
            long t1 = lib.whisper_full_get_segment_t1(ctx, i);
            segments.add(new WhisperSegment(t0,t1,text));
        }

        // 返回带时间信息的段落列表
        return segments;
    }
// 获取指定上下文中文本段的数量
public int getTextSegmentCount(Pointer ctx) {
    return lib.whisper_full_n_segments(ctx);
}

// 获取指定上下文中指定索引的文本段
public String getTextSegment(Pointer ctx, int index) {
    return lib.whisper_full_get_segment_text(ctx, index);
}

// 获取系统信息
public String getSystemInfo() {
    return lib.whisper_print_system_info();
}

// 进行内存拷贝性能测试
public int benchMemcpy(int nthread) {
    return lib.whisper_bench_memcpy(nthread);
}

// 进行 GGML 矩阵乘法性能测试
public int benchGgmlMulMat(int nthread) {
    return lib.whisper_bench_ggml_mul_mat(nthread);
}
```