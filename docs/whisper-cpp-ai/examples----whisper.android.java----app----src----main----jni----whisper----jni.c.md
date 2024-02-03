# `whisper.cpp\examples\whisper.android.java\app\src\main\jni\whisper\jni.c`

```cpp
#include <jni.h>
#include <android/asset_manager.h>
#include <android/asset_manager_jni.h>
#include <android/log.h>
#include <stdlib.h>
#include <sys/sysinfo.h>
#include <string.h>
#include "whisper.h"
#include "ggml.h"

#define UNUSED(x) (void)(x)
#define TAG "JNI"

#define LOGI(...) __android_log_print(ANDROID_LOG_INFO,     TAG, __VA_ARGS__)
#define LOGW(...) __android_log_print(ANDROID_LOG_WARN,     TAG, __VA_ARGS__)

// 定义一个内联函数，返回两个整数中的最小值
static inline int min(int a, int b) {
    return (a < b) ? a : b;
}

// 定义一个内联函数，返回两个整数中的最大值
static inline int max(int a, int b) {
    return (a > b) ? a : b;
}

// 定义一个结构体用于存储输入流的上下文信息
struct input_stream_context {
    size_t offset;
    JNIEnv * env;
    jobject thiz;
    jobject input_stream;

    jmethodID mid_available;
    jmethodID mid_read;
};

// 读取输入流中的数据到输出缓冲区
size_t inputStreamRead(void * ctx, void * output, size_t read_size) {
    struct input_stream_context* is = (struct input_stream_context*)ctx;

    // 调用 Java 方法获取输入流中可用的数据大小
    jint avail_size = (*is->env)->CallIntMethod(is->env, is->input_stream, is->mid_available);
    // 计算需要拷贝的数据大小
    jint size_to_copy = read_size < avail_size ? (jint)read_size : avail_size;

    // 创建一个新的字节数组
    jbyteArray byte_array = (*is->env)->NewByteArray(is->env, size_to_copy);

    // 调用 Java 方法从输入流中读取数据到字节数组
    jint n_read = (*is->env)->CallIntMethod(is->env, is->input_stream, is->mid_read, byte_array, 0, size_to_copy);

    // 如果拷贝的数据大小不等于请求的大小或者不等于实际读取的大小，则输出日志
    if (size_to_copy != read_size || size_to_copy != n_read) {
        LOGI("Insufficient Read: Req=%zu, ToCopy=%d, Available=%d", read_size, size_to_copy, n_read);
    }

    // 获取字节数组的元素指针并拷贝到输出缓冲区
    jbyte* byte_array_elements = (*is->env)->GetByteArrayElements(is->env, byte_array, NULL);
    memcpy(output, byte_array_elements, size_to_copy);
    // 释放字节数组的元素指针
    (*is->env)->ReleaseByteArrayElements(is->env, byte_array, byte_array_elements, JNI_ABORT);

    // 删除本地引用的字节数组
    (*is->env)->DeleteLocalRef(is->env, byte_array);

    // 更新偏移量
    is->offset += size_to_copy;

    return size_to_copy;
}

// 检查输入流是否已经结束
bool inputStreamEof(void * ctx) {
    struct input_stream_context* is = (struct input_stream_context*)ctx;

    // 调用 Java 方法获取输入流中可用的数据大小
    jint result = (*is->env)->CallIntMethod(is->env, is->input_stream, is->mid_available);
    // 判断是否已经到达流的末尾
    return result <= 0;
}
// 关闭输入流的函数
void inputStreamClose(void * ctx) {

}

// 从输入流初始化上下文的 JNI 函数
JNIEXPORT jlong JNICALL
Java_com_whispercpp_java_whisper_WhisperLib_initContextFromInputStream(
        JNIEnv *env, jobject thiz, jobject input_stream) {
    UNUSED(thiz);

    struct whisper_context *context = NULL;
    struct whisper_model_loader loader = {};
    struct input_stream_context inp_ctx = {};

    // 初始化输入流上下文
    inp_ctx.offset = 0;
    inp_ctx.env = env;
    inp_ctx.thiz = thiz;
    inp_ctx.input_stream = input_stream;

    // 获取输入流对象的类和方法 ID
    jclass cls = (*env)->GetObjectClass(env, input_stream);
    inp_ctx.mid_available = (*env)->GetMethodID(env, cls, "available", "()I");
    inp_ctx.mid_read = (*env)->GetMethodID(env, cls, "read", "([BII)I");

    // 设置模型加载器的上下文和方法
    loader.context = &inp_ctx;
    loader.read = inputStreamRead;
    loader.eof = inputStreamEof;
    loader.close = inputStreamClose;

    // 调用 eof 方法检查输入流是否结束
    loader.eof(loader.context);

    // 初始化 Whisper 上下文并返回
    context = whisper_init(&loader);
    return (jlong) context;
}

// 从资产文件初始化 Whisper 上下文
static struct whisper_context *whisper_init_from_asset(
        JNIEnv *env,
        jobject assetManager,
        const char *asset_path
) {
    LOGI("Loading model from asset '%s'\n", asset_path);
    // 从 Java AssetManager 获取 AAssetManager
    AAssetManager *asset_manager = AAssetManager_fromJava(env, assetManager);
    // 打开指定路径的资产文件
    AAsset *asset = AAssetManager_open(asset_manager, asset_path, AASSET_MODE_STREAMING);
    if (!asset) {
        LOGW("Failed to open '%s'\n", asset_path);
        return NULL;
    }

    // 设置资产文件读取器的方法
    whisper_model_loader loader = {
            .context = asset,
            .read = &asset_read,
            .eof = &asset_is_eof,
            .close = &asset_close
    };

    // 初始化 Whisper 上下文并返回
    return whisper_init(&loader);
}

// 资产文件读取函数
static size_t asset_read(void *ctx, void *output, size_t read_size) {
    return AAsset_read((AAsset *) ctx, output, read_size);
}

// 检查资产文件是否结束函数
static bool asset_is_eof(void *ctx) {
    return AAsset_getRemainingLength64((AAsset *) ctx) <= 0;
}

// 关闭资产文件函数
static void asset_close(void *ctx) {
    AAsset_close((AAsset *) ctx);
}
// 从 Android 资产管理器初始化上下文
Java_com_whispercpp_java_whisper_WhisperLib_initContextFromAsset(
        JNIEnv *env, jobject thiz, jobject assetManager, jstring asset_path_str) {
    // 忽略 thiz 参数
    UNUSED(thiz);
    // 初始化上下文指针
    struct whisper_context *context = NULL;
    // 获取 asset_path_str 的 UTF-8 字符串
    const char *asset_path_chars = (*env)->GetStringUTFChars(env, asset_path_str, NULL);
    // 从 Android 资产管理器和路径初始化上下文
    context = whisper_init_from_asset(env, assetManager, asset_path_chars);
    // 释放 UTF-8 字符串
    (*env)->ReleaseStringUTFChars(env, asset_path_str, asset_path_chars);
    // 返回上下文指针
    return (jlong) context;
}

// 初始化上下文
JNIEXPORT jlong JNICALL
Java_com_whispercpp_java_whisper_WhisperLib_initContext(
        JNIEnv *env, jobject thiz, jstring model_path_str) {
    // 忽略 thiz 参数
    UNUSED(thiz);
    // 初始化上下文指针
    struct whisper_context *context = NULL;
    // 获取 model_path_str 的 UTF-8 字符串
    const char *model_path_chars = (*env)->GetStringUTFChars(env, model_path_str, NULL);
    // 从文件路径初始化上下文
    context = whisper_init_from_file(model_path_chars);
    // 释放 UTF-8 字符串
    (*env)->ReleaseStringUTFChars(env, model_path_str, model_path_chars);
    // 返回上下文指针
    return (jlong) context;
}

// 释放上下文
JNIEXPORT void JNICALL
Java_com_whispercpp_java_whisper_WhisperLib_freeContext(
        JNIEnv *env, jobject thiz, jlong context_ptr) {
    // 忽略 env 和 thiz 参数
    UNUSED(env);
    UNUSED(thiz);
    // 将上下文指针转换为 whisper_context 类型
    struct whisper_context *context = (struct whisper_context *) context_ptr;
    // 释放上下文
    whisper_free(context);
}

// 完全转录
JNIEXPORT void JNICALL
Java_com_whispercpp_java_whisper_WhisperLib_fullTranscribe(
        JNIEnv *env, jobject thiz, jlong context_ptr, jint num_threads, jfloatArray audio_data) {
    // 忽略 thiz 参数
    UNUSED(thiz);
    // 将上下文指针转换为 whisper_context 类型
    struct whisper_context *context = (struct whisper_context *) context_ptr;
    // 获取 audio_data 的浮点数组元素
    jfloat *audio_data_arr = (*env)->GetFloatArrayElements(env, audio_data, NULL);
    // 获取 audio_data 的长度
    const jsize audio_data_length = (*env)->GetArrayLength(env, audio_data);

    // 从 Objective-C iOS 示例调整以下内容
    // 使用默认参数初始化 whisper_full_params 结构体
    struct whisper_full_params params = whisper_full_default_params(WHISPER_SAMPLING_GREEDY);
    // 设置实时打印
    params.print_realtime = true;
    // 设置不打印进度
    params.print_progress = false;
    // 设置打印时间戳
    params.print_timestamps = true;
    // 设置不打印特殊信息
    params.print_special = false;
    // 禁用翻译
    params.translate = false;
    // 设置语言为英语
    params.language = "en";
}
    # 设置参数中的线程数
    params.n_threads = num_threads;
    # 设置参数中的偏移时间为0
    params.offset_ms = 0;
    # 设置参数中的不使用上下文标志为true
    params.no_context = true;
    # 设置参数中的单段标志为false
    params.single_segment = false;

    # 重置计时器
    whisper_reset_timings(context);

    # 打印信息，即将运行whisper_full函数
    LOGI("About to run whisper_full");
    # 调用whisper_full函数运行模型，并检查返回值
    if (whisper_full(context, params, audio_data_arr, audio_data_length) != 0) {
        # 如果运行模型失败，打印错误信息
        LOGI("Failed to run the model");
    } else {
        # 如果运行模型成功，打印计时信息
        whisper_print_timings(context);
    }
    # 释放音频数据数组
    (*env)->ReleaseFloatArrayElements(env, audio_data, audio_data_arr, JNI_ABORT);
JNIEXPORT jint JNICALL
Java_com_whispercpp_java_whisper_WhisperLib_getTextSegmentCount(
        JNIEnv *env, jobject thiz, jlong context_ptr) {
    // 忽略参数 env 和 thiz
    UNUSED(env);
    UNUSED(thiz);
    // 将 context_ptr 转换为 whisper_context 结构体指针
    struct whisper_context *context = (struct whisper_context *) context_ptr;
    // 返回文本段的数量
    return whisper_full_n_segments(context);
}

JNIEXPORT jstring JNICALL
Java_com_whispercpp_java_whisper_WhisperLib_getTextSegment(
        JNIEnv *env, jobject thiz, jlong context_ptr, jint index) {
    // 忽略参数 thiz
    UNUSED(thiz);
    // 将 context_ptr 转换为 whisper_context 结构体指针
    struct whisper_context *context = (struct whisper_context *) context_ptr;
    // 获取指定索引的文本段文本内容
    const char *text = whisper_full_get_segment_text(context, index);
    // 将文本内容转换为 Java 字符串
    jstring string = (*env)->NewStringUTF(env, text);
    // 返回 Java 字符串
    return string;
}

JNIEXPORT jlong JNICALL
Java_com_whispercpp_java_whisper_WhisperLib_getTextSegmentT0(JNIEnv *env, jobject thiz,jlong context_ptr, jint index) {
    // 忽略参数 thiz
    UNUSED(thiz);
    // 将 context_ptr 转换为 whisper_context 结构体指针
    struct whisper_context *context = (struct whisper_context *) context_ptr;
    // 获取指定索引的文本段起始时间
    const int64_t t0 = whisper_full_get_segment_t0(context, index);
    // 返回起始时间
    return (jlong)t0;
}

JNIEXPORT jlong JNICALL
Java_com_whispercpp_java_whisper_WhisperLib_getTextSegmentT1(JNIEnv *env, jobject thiz,jlong context_ptr, jint index) {
    // 忽略参数 thiz
    UNUSED(thiz);
    // 将 context_ptr 转换为 whisper_context 结构体指针
    struct whisper_context *context = (struct whisper_context *) context_ptr;
    // 获取指定索引的文本段结束时间
    const int64_t t1 = whisper_full_get_segment_t1(context, index);
    // 返回结束时间
    return (jlong)t1;
}

JNIEXPORT jstring JNICALL
Java_com_whispercpp_java_whisper_WhisperLib_getSystemInfo(
        JNIEnv *env, jobject thiz
) {
    // 忽略参数 thiz
    UNUSED(thiz);
    // 获取系统信息
    const char *sysinfo = whisper_print_system_info();
    // 将系统信息转换为 Java 字符串
    jstring string = (*env)->NewStringUTF(env, sysinfo);
    // 返回 Java 字符串
    return string;
}

JNIEXPORT jstring JNICALL
Java_com_whispercpp_java_whisper_WhisperLib_benchMemcpy(JNIEnv *env, jobject thiz,
                                                                      jint n_threads) {
    // 忽略参数 thiz
    UNUSED(thiz);
    // 获取 memcpy 的性能测试结果
    const char *bench_ggml_memcpy = whisper_bench_memcpy_str(n_threads);
    // 使用 NewStringUTF 方法将 bench_ggml_memcpy 转换为 Java 字符串对象
    jstring string = (*env)->NewStringUTF(env, bench_ggml_memcpy);
}

JNIEXPORT jstring JNICALL
Java_com_whispercpp_java_whisper_WhisperLib_benchGgmlMulMat(JNIEnv *env, jobject thiz,
                                                                          jint n_threads) {
    // 忽略未使用的参数 thiz
    UNUSED(thiz);
    // 调用 whisper_bench_ggml_mul_mat_str 函数获取性能测试结果字符串
    const char *bench_ggml_mul_mat = whisper_bench_ggml_mul_mat_str(n_threads);
    // 将 C 字符串转换为 Java 字符串
    jstring string = (*env)->NewStringUTF(env, bench_ggml_mul_mat);
}
```