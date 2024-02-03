# `whisper.cpp\bindings\java\src\test\java\io\github\ggerganov\whispercpp\WhisperCppTest.java`

```cpp
package io.github.ggerganov.whispercpp;

import static org.junit.jupiter.api.Assertions.*;

import io.github.ggerganov.whispercpp.bean.WhisperSegment;
import io.github.ggerganov.whispercpp.params.CBool;
import io.github.ggerganov.whispercpp.params.WhisperFullParams;
import io.github.ggerganov.whispercpp.params.WhisperSamplingStrategy;
import org.junit.jupiter.api.BeforeAll;
import org.junit.jupiter.api.Test;
import javax.sound.sampled.AudioInputStream;
import javax.sound.sampled.AudioSystem;
import java.io.File;
import java.io.FileNotFoundException;
import java.util.List;

class WhisperCppTest {
    private static WhisperCpp whisper = new WhisperCpp();
    private static boolean modelInitialised = false;

    @BeforeAll
    static void init() throws FileNotFoundException {
        // 默认情况下，模型从 ~/.cache/whisper/ 加载，通常命名为 "ggml-${name}.bin"
        // 或者您可以提供模型文件的绝对路径。
        //String modelName = "../../models/ggml-tiny.bin";
        String modelName = "../../models/ggml-tiny.en.bin";
        try {
            // 初始化 WhisperCpp 实例的上下文，加载模型
            whisper.initContext(modelName);
            // 获取默认参数
            //whisper.getFullDefaultParams(WhisperSamplingStrategy.WHISPER_SAMPLING_GREEDY);
            //whisper.getJavaDefaultParams(WhisperSamplingStrategy.WHISPER_SAMPLING_BEAM_SEARCH);
            modelInitialised = true;
        } catch (FileNotFoundException ex) {
            // 捕获文件未找到异常
            System.out.println("Model " + modelName + " not found");
        }
    }

    @Test
    // 测试获取使用 Beam Search 策略的默认参数
    void testGetDefaultFullParams_BeamSearch() {
        // 调用方法获取参数
        WhisperFullParams params = whisper.getFullDefaultParams(WhisperSamplingStrategy.WHISPER_SAMPLING_BEAM_SEARCH);

        // 断言参数值符合预期
        assertEquals(WhisperSamplingStrategy.WHISPER_SAMPLING_BEAM_SEARCH.ordinal(), params.strategy);
        assertNotEquals(0, params.n_threads);
        assertEquals(16384, params.n_max_text_ctx);
        assertFalse(params.translate);
        assertEquals(0.01f, params.thold_pt);
        assertEquals(5, params.beam_search.beam_size);
        assertEquals(-1.0f, params.beam_search.patience);
    }

    // 测试获取使用 Greedy 策略的默认参数
    @Test
    void testGetDefaultFullParams_Greedy() {
        // 调用方法获取参数
        WhisperFullParams params = whisper.getFullDefaultParams(WhisperSamplingStrategy.WHISPER_SAMPLING_GREEDY);

        // 断言参数值符合预期
        assertEquals(WhisperSamplingStrategy.WHISPER_SAMPLING_GREEDY.ordinal(), params.strategy);
        assertNotEquals(0, params.n_threads);
        assertEquals(16384, params.n_max_text_ctx);
        assertEquals(5, params.greedy.best_of);
    }

    @Test
    // 定义一个测试方法，用于测试完整转录功能
    void testFullTranscribe() throws Exception {
        // 如果模型未初始化，则打印消息并跳过测试
        if (!modelInitialised) {
            System.out.println("Model not initialised, skipping test");
            return;
        }

        // 给定音频文件路径
        File file = new File(System.getProperty("user.dir"), "../../samples/jfk.wav");
        // 获取音频输入流
        AudioInputStream audioInputStream = AudioSystem.getAudioInputStream(file);

        // 创建一个字节数组，大小为音频输入流可用字节数
        byte[] b = new byte[audioInputStream.available()];
        // 创建一个浮点数数组，大小为字节数组长度的一半
        float[] floats = new float[b.length / 2];

        // 获取默认的 WhisperFullParams 参数对象，使用 WHISPER_SAMPLING_BEAM_SEARCH 策略
        WhisperFullParams params = whisper.getFullDefaultParams(WhisperSamplingStrategy.WHISPER_SAMPLING_BEAM_SEARCH);
        // 设置进度回调函数
        params.setProgressCallback((ctx, state, progress, user_data) -> System.out.println("progress: " + progress));
        // 设置打印进度为假
        params.print_progress = CBool.FALSE;
        // 设置初始提示语句
        //params.initial_prompt = "and so my fellow Americans um, like";

        try {
            // 读取音频输入流到字节数组
            audioInputStream.read(b);

            // 将字节数组中的每两个字节转换为一个整数样本，并存储到浮点数数组中
            for (int i = 0, j = 0; i < b.length; i += 2, j++) {
                int intSample = (int) (b[i + 1]) << 8 | (int) (b[i]) & 0xFF;
                floats[j] = intSample / 32767.0f;
            }

            // 进行完整转录，获取结果字符串
            String result = whisper.fullTranscribe(params, floats);

            // 打印结果字符串
            System.err.println(result);
            // 断言结果字符串与预期结果相同（去除逗号）
            assertEquals("And so my fellow Americans ask not what your country can do for you " +
                    "ask what you can do for your country.",
                    result.replace(",", ""));
        } finally {
            // 关闭音频输入流
            audioInputStream.close();
        }
    }

    @Test
    // 测试带时间的完整转录
    void testFullTranscribeWithTime() throws Exception {
        // 如果模型未初始化，则跳过测试
        if (!modelInitialised) {
            System.out.println("Model not initialised, skipping test");
            return;
        }

        // 给定音频文件
        File file = new File(System.getProperty("user.dir"), "../../samples/jfk.wav");
        // 获取音频输入流
        AudioInputStream audioInputStream = AudioSystem.getAudioInputStream(file);

        // 创建字节数组并将音频流的可用字节数读入数组
        byte[] b = new byte[audioInputStream.available()];
        // 创建浮点数数组，长度为字节数组长度的一半
        float[] floats = new float[b.length / 2];

        // 获取默认参数并设置进度回调
        WhisperFullParams params = whisper.getFullDefaultParams(WhisperSamplingStrategy.WHISPER_SAMPLING_BEAM_SEARCH);
        params.setProgressCallback((ctx, state, progress, user_data) -> System.out.println("progress: " + progress));
        params.print_progress = CBool.FALSE;
        //params.initial_prompt = "and so my fellow Americans um, like";

        try {
            // 读取音频流到字节数组
            audioInputStream.read(b);

            // 将字节数组转换为浮点数数组
            for (int i = 0, j = 0; i < b.length; i += 2, j++) {
                int intSample = (int) (b[i + 1]) << 8 | (int) (b[i]) & 0xFF;
                floats[j] = intSample / 32767.0f;
            }

            // 进行完整转录并获取结果
            List<WhisperSegment> segments = whisper.fullTranscribeWithTime(params, floats);
            // 断言结果列表的大小大于0
            assertTrue(segments.size() > 0, "The size of segments should be greater than 0");
            // 打印每个转录片段
            for (WhisperSegment segment : segments) {
                System.out.println(segment);
            }
        } finally {
            // 关闭音频输入流
            audioInputStream.close();
        }
    }
# 闭合之前的代码块
```