# `chatglm.cpp\chatglm_test.cpp`

```
// 包含 chatglm.h 头文件
#include "chatglm.h"
// 包含文件系统和测试框架的头文件
#include <filesystem>
#include <gtest/gtest.h>

// 如果使用了 CUBLAS，则包含 CUDA 运行时和 ggml-cuda 头文件
#ifdef GGML_USE_CUBLAS
#include <cuda_runtime.h>
#include <ggml-cuda.h>
#endif

// chatglm 命名空间
namespace chatglm {

// 命名空间别名为 fs
namespace fs = std::filesystem;

// 获取线程数的静态内联函数
static inline int get_num_threads() {
    // 获取环境变量 CHATGLM_NUM_THREADS
    const char *chatglm_num_threads_env = getenv("CHATGLM_NUM_THREADS");
    // 如果环境变量存在，则将其转换为整数，否则使用默认线程数
    int num_threads = chatglm_num_threads_env ? std::stoi(chatglm_num_threads_env) : get_default_num_threads();
    return num_threads;
}

// 检查两个 ggml_tensor 是否全部接近的静态内联函数
static inline void expect_all_close(ggml_tensor *a, ggml_tensor *b, float atol = 1e-5f, float rtol = 0.f) {
    // 断言两个 tensor 的类型相同且为 GGML_TYPE_F32
    ASSERT_EQ(a->type, b->type);
    ASSERT_EQ(a->type, GGML_TYPE_F32);
    // 断言两个 tensor 的元素个数相同
    ASSERT_EQ(ggml_nelements(a), ggml_nelements(b));
    int64_t numel = ggml_nelements(a);
    // 遍历每个元素，检查其值是否接近
    for (int64_t i = 0; i < numel; i++) {
        float ai = ((float *)a->data)[i];
        float bi = ((float *)b->data)[i];
        EXPECT_LT(std::abs(ai - bi), atol + rtol * std::abs(bi)) << "diff " << ai << " vs " << bi;
    }
}

// 从指针中读取 tensor 数据的静态内联函数
static inline char *read_tensor_data(char *ptr, ggml_tensor *tensor) {
    // 将数据从指针复制到 tensor 中
    memcpy(tensor->data, ptr, ggml_nbytes(tensor));
    return ptr + ggml_nbytes(tensor);
}

// 生成随机数的静态内联函数
static inline float random() { return rand() / (float)RAND_MAX; }

// 生成指定范围内的随机数的静态内联函数
static inline float random(float lo, float hi) { return lo + random() * (hi - lo); }

// 随机填充 tensor 数据的静态内联函数
static inline void random_fill(ggml_tensor *tensor) {
    // 生成随机值并填充到 vector 中
    std::vector<float> values(ggml_nelements(tensor));
    for (float &v : values) {
        v = random();
    }
    int64_t hist[16]{};

    // 根据 tensor 类型进行不同的填充操作
    if (tensor->type == GGML_TYPE_F32) {
        // 如果是 F32 类型，则直接复制数据
        memcpy(tensor->data, values.data(), sizeof(float) * values.size());
    } else if (tensor->type == GGML_TYPE_F16) {
        // 如果是 F16 类型，则将数据转换为 F16 类型
        ggml_fp32_to_fp16_row(values.data(), (ggml_fp16_t *)tensor->data, values.size());
    } else if (tensor->type == GGML_TYPE_Q8_0) {
        // 如果是 Q8_0 类型，则进行 Q8_0 类型的量化
        ggml_quantize_q8_0(values.data(), tensor->data, ggml_nelements(tensor), tensor->ne[0], hist);
    // 如果张量的类型为 GGML_TYPE_Q4_0，则调用 ggml_quantize_q4_0 函数进行量化
    } else if (tensor->type == GGML_TYPE_Q4_0) {
        ggml_quantize_q4_0(values.data(), tensor->data, ggml_nelements(tensor), tensor->ne[0], hist);
    // 如果张量的类型为 GGML_TYPE_Q4_1，则调用 ggml_quantize_q4_1 函数进行量化
    } else if (tensor->type == GGML_TYPE_Q4_1) {
        ggml_quantize_q4_1(values.data(), tensor->data, ggml_nelements(tensor), tensor->ne[0], hist);
    // 如果张量的类型不是 Q4_0 或 Q4_1，则抛出异常，显示不支持的数据类型
    } else {
        CHATGLM_THROW << "unsupported dtype " << ggml_type_name(tensor->type);
    }
// 返回以毫秒为单位的经过的时间
static inline float timeit(std::function<void()> fn, int warmup, int active) {
    // 预热阶段，执行指定次数的函数调用
    for (int i = 0; i < warmup; i++) {
        fn();
    }

#ifdef GGML_USE_CUBLAS
    // 如果使用了 CUBLAS，等待 CUDA 设备同步
    CHATGLM_CHECK_CUDA(cudaDeviceSynchronize());
#endif
    // 记录开始时间
    int64_t start_us = ggml_time_us();
    // 执行指定次数的函数调用
    for (int i = 0; i < active; i++) {
        fn();
    }
#ifdef GGML_USE_CUBLAS
    // 如果使用了 CUBLAS，等待 CUDA 设备同步
    CHATGLM_CHECK_CUDA(cudaDeviceSynchronize());
#endif
    // 记录结束时间
    int64_t end_us = ggml_time_us();

    // 计算经过的时间并转换为毫秒
    float elapsed_ms = (end_us - start_us) / 1000.f;
    return elapsed_ms / active;
}

// 检查两个整数向量是否相等
static bool equal(const std::vector<int> &a, const std::vector<int> &b) {
    // 如果向量长度不相等，则返回 false
    if (a.size() != b.size()) {
        return false;
    }
    // 逐个比较向量元素，如果有不相等的则返回 false
    for (size_t i = 0; i < a.size(); i++) {
        if (a[i] != b[i]) {
            return false;
        }
    }
    // 否则返回 true
    return true;
}

// 从 TokenIdScore 向量中提取排序后的 id 向量
static std::vector<int> extract_sorted_ids(std::vector<TokenIdScore> &token_scores) {
    // 创建与 token_scores 大小相同的整数向量
    std::vector<int> token_ids(token_scores.size());
    // 将 TokenIdScore 向量中的 id 提取到 token_ids 中
    for (size_t i = 0; i < token_scores.size(); i++) {
        token_ids[i] = token_scores[i].id;
    }
    // 对 token_ids 进行排序
    std::sort(token_ids.begin(), token_ids.end());
    return token_ids;
}

// 重复惩罚采样测试
TEST(Sampling, RepetitionPenalty) {
    // 定义惩罚值
    constexpr float penalty = 1.2;
    // 初始化 logits、input_ids 和 target 向量
    std::vector<float> logits{0.96, 1.2, -2, -0.8, 0, 2.4, -1};
    std::vector<int> input_ids{0, 2, 5, 2};
    std::vector<float> target{0.8, 1.2, -2.4, -0.8, 0, 2, -1};
    // 执行采样重复惩罚函数
    BaseModelForCausalLM::sampling_repetition_penalty(logits.data(), logits.data() + logits.size(), input_ids, penalty);
    // 比较结果
    for (size_t i = 0; i < logits.size(); i++) {
        EXPECT_FLOAT_EQ(logits[i], target[i]);
    }
}

// 基准测试重复惩罚采样
TEST(DISABLED_Sampling, BenchmarkRepetitionPenalty) {
    // 定义惩罚值、词汇表大小和序列长度
    const float penalty = 1.2;
    constexpr size_t vocab_size = 128000;
    constexpr int seq_len = 32000;
    // 初始化 logits 向量
    std::vector<float> logits(vocab_size);
    // 为 logits 向量中的每个元素赋随机值
    for (auto &x : logits) {
        x = random(-1, 1);
    }
    // 初始化 input_ids 向量
    std::vector<int> input_ids(seq_len);
    // 遍历 input_ids 容器中的元素，将每个元素的值设置为其索引值
    for (size_t i = 0; i < input_ids.size(); i++) {
        input_ids[i] = i;
    }

    // 创建一个 lambda 函数 fn，捕获 logits、input_ids 和 penalty 变量
    auto fn = [&logits, &input_ids, penalty] {
        // 调用 BaseModelForCausalLM 类的 sampling_repetition_penalty 方法，传入参数 logits 的起始和结束位置、input_ids 和 penalty
        BaseModelForCausalLM::sampling_repetition_penalty(logits.data(), logits.data() + logits.size(), input_ids, penalty);
    };
    // 计算 fn 函数执行时间，重复执行 2 次，每次执行 100 次
    auto elapsed_ms = timeit(fn, 2, 100);
    // 输出测试名称、执行时间（毫秒）
    std::cout << "[" << ::testing::UnitTest::GetInstance()->current_test_info()->name() << "] " << elapsed_ms << " ms\n";
}

// 测试温度采样
TEST(Sampling, Temperature) {
    // 定义温度
    constexpr float temp = 0.7;
    // 创建包含64个元素的logits向量
    std::vector<float> logits(64);
    // 为logits向量中的每个元素生成随机值
    for (float &v : logits) {
        v = random();
    }
    // 创建参考向量
    std::vector<float> target = logits;
    // 将参考向量中的每个元素除以温度
    for (auto &v : target) {
        v /= temp;
    }
    // 进行测试，调用sampling_temperature函数
    BaseModelForCausalLM::sampling_temperature(logits.data(), logits.data() + logits.size(), temp);
    // 比较结果
    for (size_t i = 0; i < logits.size(); i++) {
        EXPECT_FLOAT_EQ(logits[i], target[i]);
    }
}

// 测试TopK采样
TEST(Sampling, TopK) {
    // 定义TopK值
    constexpr int top_k = 20;
    // 创建包含64个TokenIdScore结构的向量
    std::vector<TokenIdScore> token_scores(64);
    // 为每个TokenIdScore结构生成随机值
    for (size_t i = 0; i < token_scores.size(); i++) {
        token_scores[i] = TokenIdScore(i, random());
    }

    // 创建参考向量
    std::vector<TokenIdScore> target = token_scores;
    // 对参考向量进行排序，并保留前top_k个元素
    std::sort(target.begin(), target.end(), std::greater<TokenIdScore>());
    target.resize(top_k);

    // 进行测试，调用sampling_top_k函数
    BaseModelForCausalLM::sampling_top_k(token_scores.data(), token_scores.data() + top_k,
                                         token_scores.data() + token_scores.size());
    token_scores.resize(top_k);

    // 排序并比较结果
    EXPECT_TRUE(equal(extract_sorted_ids(token_scores), extract_sorted_ids(target)));
}

// 参考TopP函数
static void reference_top_p(std::vector<TokenIdScore> &token_scores, float top_p) {
    // 对token_scores向量进行排序
    std::sort(token_scores.begin(), token_scores.end(), std::greater<TokenIdScore>());
    // 对token_scores向量进行softmax操作
    BaseModelForCausalLM::sampling_softmax_inplace(token_scores.data(), token_scores.data() + token_scores.size());
    // 计算累积概率，根据top_p值调整向量大小
    float cumsum = 0.f;
    for (size_t i = 0; i < token_scores.size(); i++) {
        cumsum += token_scores[i].score;
        if (cumsum >= top_p) {
            token_scores.resize(i + 1);
            break;
        }
    }
}

// 测试TopP采样
TEST(Sampling, TopP) {
    // 定义TopP值
    constexpr float top_p = 0.7;
    // 循环10次，每次生成一个包含1024个TokenIdScore对象的vector
    for (int i = 0; i < 10; i++) {
        std::vector<TokenIdScore> token_scores(1024);
        // 遍历token_scores，为每个元素赋予一个随机的TokenIdScore对象
        for (size_t i = 0; i < token_scores.size(); i++) {
            token_scores[i] = TokenIdScore(i, random());
        }

        // 生成一个target vector，复制token_scores的内容
        std::vector<TokenIdScore> target = token_scores;
        // 调用reference_top_p函数对target进行处理
        reference_top_p(target, top_p);
        // 断言token_scores不为空
        EXPECT_TRUE(!token_scores.empty());

        // 调用BaseModelForCausalLM的sampling_top_p函数对token_scores进行处理
        TokenIdScore *pos = BaseModelForCausalLM::sampling_top_p(token_scores.data(), token_scores.data() + token_scores.size(), top_p);
        // 调整token_scores的大小，使其仅包含sampling_top_p处理后的部分
        token_scores.resize(pos - token_scores.data());

        // 提取并排序output_ids和target_ids，用于后续比较
        auto output_ids = extract_sorted_ids(token_scores);
        auto target_ids = extract_sorted_ids(target);
        // 断言output_ids和target_ids相等，否则输出错误信息
        EXPECT_TRUE(equal(output_ids, target_ids)) << "size " << output_ids.size() << " vs " << target_ids.size();
    }
// 定义一个名为 ChatGLMTest 的测试类，继承自 ::testing::Test
class ChatGLMTest : public ::testing::Test {
  protected:
    // 声明一个 ModelContext 对象 ctx
    ModelContext ctx;

    // 在每个测试用例执行前执行的 SetUp 函数
    void SetUp() override {
        // 设置 ctx 的数据类型为 GGML_TYPE_F32
        ctx.dtype = GGML_TYPE_F32;
        // 创建一个大小为 1024MB 的 ggml_context 对象 ctx_w
        ctx.ctx_w = make_unique_ggml_context(1024 * MB, nullptr, false);
        // 创建一个大小为 512MB 的 ggml_context 对象 ctx_kv
        ctx.ctx_kv = make_unique_ggml_context(512 * MB, nullptr, false);
        // 创建一个大小为 512MB 的 ggml_context 对象 ctx_b
        ctx.ctx_b = make_unique_ggml_context(512 * MB, nullptr, false);
        // 调整 ctx 的 scratch_buffer 大小为 1MB
        ctx.scratch_buffer.resize(1 * MB);
        // 初始化 ctx 的 scratch 属性
        ctx.scratch = {0, ctx.scratch_buffer.size(), ctx.scratch_buffer.data()};
#ifdef GGML_USE_CUBLAS
        // 设置 CUDA 的 scratch 大小
        ggml_cuda_set_scratch_size(ctx.scratch_buffer.size());
#endif
        // 初始化设备上下文
        ctx.init_device_context();

        // 重置计算图
        reset_cgraph();
    }

    // 在每个测试用例执行后执行的 TearDown 函数
    void TearDown() override {
#ifdef GGML_USE_CUBLAS
        // 释放 CUDA 的 scratch
        ggml_cuda_free_scratch();
#endif
    }

    // 重置计算图
    void reset_cgraph() { ctx.gf = {}; }

    // 在 CPU 上计算图
    void cpu_graph_compute(int n_threads) { ggml_graph_compute_helper(ctx.work_buffer, &ctx.gf, n_threads); }

    // 在设备上计算图
    void device_graph_compute(int n_threads) {
#ifdef GGML_USE_METAL
        // 设置 Metal 的回调数
        // ggml_metal_set_n_cb(ctx.ctx_metal.get(), n_threads);
        // 在 Metal 上计算图
        ggml_metal_graph_compute(ctx.ctx_metal.get(), &ctx.gf);
        // 获取 Metal 上的张量
        // ggml_metal_get_tensor(ctx.ctx_metal.get(), output);
#else
        // 在 CPU 上计算图
        cpu_graph_compute(n_threads);
#endif
    }

    // 性能计算图的实现
    template <bool FALLBACK_CPU>
    float _perf_graph_compute_impl() {
        // 获取线程数
        int num_threads = get_num_threads();
        auto fn = [this, num_threads] {
            if constexpr (FALLBACK_CPU) {
                // 如果需要回退到 CPU 计算，则在 CPU 上计算图
                cpu_graph_compute(num_threads);
            } else {
                // 否则在设备��计算图
                device_graph_compute(num_threads);
            }
        };
#if defined(GGML_USE_CUBLAS) || defined(GGML_USE_METAL)
        // 计算函数 fn 的执行时间
        return timeit(fn, 10, 100);
#else
        // 计算函数 fn 的执行时间
        return timeit(fn, 1, 3);
#endif
    }

    // 计算在 CPU 上计算图的性能
    float perf_cpu_graph_compute() { return _perf_graph_compute_impl<true>(); }
    // 计算在设备上计算图的性能
    float perf_device_graph_compute() { return _perf_graph_compute_impl<false>(); }

    // 模板函数
    template <typename Model>
    }
};

// ChatGLMTest 测试类的测试用例 Embedding
TEST_F(ChatGLMTest, Embedding) {
    # 定义包含权重数据的浮点数数组
    float w_data[]{1.5410, -0.2934, -2.1788, 0.5684,  -1.0845, -1.3986,
                   0.4033, 0.8380,  -0.7193, -0.4033, -0.5966, 0.1820};
    # 定义包含输入数据的整数数组
    int x_data[]{1, 3, 0, 2, 3};
    # 定义包含期望输出数据的浮点数数组
    float y_data[]{0.5684,  -1.0845, -1.3986, -0.4033, -0.5966, 0.1820,  1.5410, -0.2934,
                   -2.1788, 0.4033,  0.8380,  -0.7193, -0.4033, -0.5966, 0.1820};

    # 创建一个包含输入数据的一维张量
    ggml_tensor *x = ggml_new_tensor_1d(ctx.ctx_b.get(), GGML_TYPE_I32, 5);
    # 将输入数据复制到张量的数据中
    memcpy(x->data, x_data, sizeof(x_data));
    # 创建一个嵌入模型，指定上下文和输入输出维度
    Embedding model(&ctx, 4, 3);
    # 将权重数据复制到模型的权重张量中
    memcpy(model.weight->data, w_data, sizeof(w_data));
    # 创建一个包含期望输出数据的二维张量
    ggml_tensor *ref = ggml_new_tensor_2d(ctx.ctx_b.get(), GGML_TYPE_F32, 3, 5);
    # 将期望输出数据赋值给张量的数据
    ref->data = y_data;

    # 使用模型进行前向传播，得到输出张量
    ggml_tensor *out = model.forward(&ctx, x);

    # 构建前向传播的扩展
    ggml_build_forward_expand(&ctx.gf, out);
    # 在 CPU 上计算图
    cpu_graph_compute(1);

    # 检查输出张量是否与期望输出张量接近
    expect_all_close(ref, out);
// 定义测试用例 Linear
TEST_F(ChatGLMTest, Linear) {
    // 获取测试文件路径
    fs::path test_path = fs::path(__FILE__).parent_path() / "tests/data/linear.data";
    // 创建映射文件对象
    MappedFile mapped_file(test_path.string());
    // 获取映射文件数据指针
    char *ptr = mapped_file.data;

    // 创建 2D 张量 w
    ggml_tensor *w = ggml_new_tensor_2d(ctx.ctx_b.get(), GGML_TYPE_F32, 32, 16);
    // 读取张量数据
    ptr = read_tensor_data(ptr, w);
    // 创建 1D 张量 b
    ggml_tensor *b = ggml_new_tensor_1d(ctx.ctx_b.get(), GGML_TYPE_F32, 16);
    // 读取张量数据
    ptr = read_tensor_data(ptr, b);
    // 创建 2D 张量 x
    ggml_tensor *x = ggml_new_tensor_2d(ctx.ctx_b.get(), GGML_TYPE_F32, 32, 2);
    // 读取张量数据
    ptr = read_tensor_data(ptr, x);
    // 创建 2D 张量 ref
    ggml_tensor *ref = ggml_new_tensor_2d(ctx.ctx_b.get(), GGML_TYPE_F32, 16, 2);
    // 读取张量数据
    ptr = read_tensor_data(ptr, ref);
    // 断言指针位置是否正确
    ASSERT_EQ(ptr, mapped_file.data + mapped_file.size);

    // 创建 1D 张量 vx，复制数据
    ggml_tensor *vx = ggml_new_tensor_1d(ctx.ctx_b.get(), GGML_TYPE_F32, 32);
    memcpy(vx->data, x->data, 32 * sizeof(float));
    // 创建 1D 张量 vref，复制数据
    ggml_tensor *vref = ggml_new_tensor_1d(ctx.ctx_b.get(), GGML_TYPE_F32, 16);
    memcpy(vref->data, ref->data, 16 * sizeof(float));

    // 将张量数据传输到设备
    tensor_to_device(x);
    tensor_to_device(vx);

    // 定义测试用例结构
    struct TestCase {
        ggml_tensor *x;
        ggml_tensor *ref;
    };
    // 创建测试用例向量
    std::vector<TestCase> cases{{x, ref}, {vx, vref}};

    // 定义测试配置结构
    struct TestConfig {
        ggml_type dtype;
        float atol;
        float rtol;
    };
    // 创建测试配置向量
    std::vector<TestConfig> test_configs{
        {GGML_TYPE_F32, 1e-5, 0},
        {GGML_TYPE_F16, 1e-2, 5e-4},
        {GGML_TYPE_Q4_0, 1.0, 0.2},
    };
}
    // 遍历测试配置列表
    for (const auto &config : test_configs) {
        // 设置上下文的数据类型为配置中指定的数据类型
        ctx.dtype = config.dtype;
        // 创建一个线性模型对象，指定上下文、输入维度和输出维度
        Linear model(&ctx, 32, 16);

        // 根据数据类型不同，设置模型的权重数据
        if (config.dtype == GGML_TYPE_F32) {
            model.weight->data = w->data;
        } else if (config.dtype == GGML_TYPE_F16) {
            // 将权重数据从单精度浮点型转换为半精度浮点型
            ggml_fp32_to_fp16_row((float *)w->data, (ggml_fp16_t *)model.weight->data, ggml_nelements(model.weight));
        } else if (config.dtype == GGML_TYPE_Q4_0) {
            // 对权重数据进行 Q4.0 格式的量化
            int64_t hist[16]{};
            ggml_quantize_q4_0((float *)w->data, model.weight->data, ggml_nelements(w), w->ne[0], hist);
        } else {
            // 抛出异常，表示不支持的数据类型
            CHATGLM_THROW << "unsupported dtype " << config.dtype;
        }
        // 设置模型的偏置数据
        model.bias->data = b->data;
        // 将模型的权重数据传输到设备
        tensor_to_device(model.weight);
        // 将模型的偏置数据传输到设备
        tensor_to_device(model.bias);

        // 遍历测试用例列表
        for (const auto &c : cases) {
            // 重置计算图
            reset_cgraph();
            // 对模型进行前向传播，获取输出
            ggml_tensor *out = model.forward(&ctx, c.x);
            // 断言输出的后端与输入的后端相同
            EXPECT_EQ(out->backend, c.x->backend);
            // 将输出的后端设置为 CPU
            out->backend = GGML_BACKEND_CPU;

            // 构建前向传播的扩展计算图
            ggml_build_forward_expand(&ctx.gf, out);
            // 执行设备图计算
            device_graph_compute(get_num_threads());

            // 断言输出的数据类型为单精度浮点型
            EXPECT_EQ(out->type, GGML_TYPE_F32);
            // 检查输出与参考值的接近程度
            expect_all_close(c.ref, out, config.atol, config.rtol);
        }

        // 将模型的权重数据传输回 CPU
        tensor_to_cpu(model.weight);
        // 将模型的偏置数据传输回 CPU
        tensor_to_cpu(model.bias);
    }
    // 将输入数据传输回 CPU
    tensor_to_cpu(x);
    // 将输入数据的梯度传输回 CPU
    tensor_to_cpu(vx);
}

# 在 ChatGLMTest 类中定义一个 BenchmarkLinear 测试用例
TEST_F(ChatGLMTest, BenchmarkLinear) {
    # 定义常量 M、N、K 的值
    constexpr int M = 64, N = 1024, K = 1024 * 3;
    # 设置 ctx 的数据类型为 GGML_TYPE_F32
    ctx.dtype = GGML_TYPE_F32;
    # 创建一个 Linear 对象 m
    Linear m(&ctx, K, N);
    # 创建一个 2D 的 GGML 浮点数类型的张量 x
    ggml_tensor *x = ggml_new_tensor_2d(ctx.ctx_b.get(), GGML_TYPE_F32, K, M);

    # 创建包含 m.weight、m.bias、x 的张量指针向量 all_tensors
    std::vector<ggml_tensor *> all_tensors{m.weight, m.bias, x};
    # 遍历 all_tensors 中的每个张量，填充随机数据并将数据传输到设备
    for (auto tensor : all_tensors) {
        random_fill(tensor);
        tensor_to_device(tensor);
    }

    # 对输入张量 x 进行前向传播，得到输出张量 y
    ggml_tensor *y = m.forward(&ctx, x);
    # 构建前向传播的扩展
    ggml_build_forward_expand(&ctx.gf, y);
    # 输出性能信息
    std::cout << "[Benchmark] Linear " << ggml_type_name(ctx.dtype) << " time: " << perf_device_graph_compute()
              << " ms\n";

    # 将所有张量传输回 CPU
    for (auto tensor : all_tensors) {
        tensor_to_cpu(tensor);
    }
}

# 在 ChatGLMTest 类中定义一个 LayerNorm 测试用例
TEST_F(ChatGLMTest, LayerNorm) {
    # 获取测试文件路径
    fs::path test_path = fs::path(__FILE__).parent_path() / "tests/data/layer_norm.data";
    # 创建一个 MappedFile 对象 mapped_file
    MappedFile mapped_file(test_path.string());
    char *ptr = mapped_file.data;

    # 创建一个 LayerNorm 对象 model
    LayerNorm model(&ctx, 64);
    # 创建 2D 的 GGML 浮点数类型的张量 x 和 ref
    ggml_tensor *x = ggml_new_tensor_2d(ctx.ctx_b.get(), GGML_TYPE_F32, 64, 3);
    ggml_tensor *ref = ggml_new_tensor_2d(ctx.ctx_b.get(), GGML_TYPE_F32, 64, 3);

    # 创建包含 model.weight、model.bias、x、ref 的张量指针向量 all_tensors
    std::vector<ggml_tensor *> all_tensors{model.weight, model.bias, x, ref};
    # 遍历 all_tensors 中的每个张量，读取数据并将数据传输到设备
    for (auto tensor : all_tensors) {
        ptr = read_tensor_data(ptr, tensor);
        tensor_to_device(tensor);
    }
    # 断言指针位置
    ASSERT_EQ(ptr, mapped_file.data + mapped_file.size);

    # 对输入张量 x 进行前向传播，得到输出张量 out
    ggml_tensor *out = model.forward(&ctx, x);
    # 断言输出张量的后端与输入张量相同
    EXPECT_EQ(out->backend, x->backend);
    out->backend = GGML_BACKEND_CPU;

    # 构建前向传播的扩展
    ggml_build_forward_expand(&ctx.gf, out);
    # 计算设备图的性能
    device_graph_compute(get_num_threads());

    # 预期输出张量 ref 与 out 的接近程度
    expect_all_close(ref, out);

    # 将所有张量传输回 CPU
    for (auto tensor : all_tensors) {
        tensor_to_cpu(tensor);
    }
}

# 在 ChatGLMTest 类中定义一个 BenchmarkLayerNorm 测试用例
TEST_F(ChatGLMTest, BenchmarkLayerNorm) {
    # 定义序列长度和隐藏层大小
    constexpr int seq_len = 64;
    constexpr int hidden = 1024;

    # 创建一个 LayerNorm 对象 m
    LayerNorm m(&ctx, hidden);
    # 创建 2D 的 GGML 浮点数类型的张量 x
    ggml_tensor *x = ggml_new_tensor_2d(ctx.ctx_b.get(), GGML_TYPE_F32, hidden, seq_len);

    # 创建包含 m.weight、m.bias、x 的张量指针向量 all_tensors
    // 遍历所有张量，对每个张量进行随机填充
    for (auto tensor : all_tensors) {
        random_fill(tensor);
        // 将张量传输到设备上
        tensor_to_device(tensor);
    }

    // 使用输入张量 x 进行前向传播，得到输出张量 y
    ggml_tensor *y = m.forward(&ctx, x);
    // 对输出张量 y 进行前向扩展
    ggml_build_forward_expand(&ctx.gf, y);
    // 输出性能信息，包括 LayerNorm 的数据类型和计算时间
    std::cout << "[Benchmark] LayerNorm " << ggml_type_name(ctx.dtype) << " time: " << perf_device_graph_compute()
              << " ms\n";

    // 将所有张量传输回 CPU
    for (auto tensor : all_tensors) {
        tensor_to_cpu(tensor);
    }
}

# 测试 RMSNorm 函数
TEST_F(ChatGLMTest, RMSNorm) {
    # 获取测试数据文件路径
    fs::path test_path = fs::path(__FILE__).parent_path() / "tests/data/rms_norm.data";
    # 创建映射文件对象
    MappedFile mapped_file(test_path.string());
    # 获取映射文件数据指针
    char *ptr = mapped_file.data;

    # 创建 RMSNorm 模型对象
    RMSNorm model(&ctx, 64);
    # 创建输入和参考张量
    ggml_tensor *x = ggml_new_tensor_2d(ctx.ctx_b.get(), GGML_TYPE_F32, 64, 3);
    ggml_tensor *ref = ggml_new_tensor_2d(ctx.ctx_b.get(), GGML_TYPE_F32, 64, 3);

    # 将所有张量放入向量中
    std::vector<ggml_tensor *> all_tensors{model.weight, x, ref};
    # 遍历所有张量，读取数据并转移到设备
    for (auto tensor : all_tensors) {
        ptr = read_tensor_data(ptr, tensor);
        tensor_to_device(tensor);
    }
    # 断言指针位置是否正确
    ASSERT_EQ(ptr, mapped_file.data + mapped_file.size);

    # 进行前向传播
    ggml_tensor *out = model.forward(&ctx, x);
    # 断言输出张量的后端与输入张量相同
    EXPECT_EQ(out->backend, x->backend);
    out->backend = GGML_BACKEND_CPU;

    # 构建前向传播图并计算
    ggml_build_forward_expand(&ctx.gf, out);
    device_graph_compute(get_num_threads());

    # 预期输出与参考张量接近
    expect_all_close(ref, out);

    # 将所有张量转移到 CPU
    for (auto tensor : all_tensors) {
        tensor_to_cpu(tensor);
    }
}

# 测试 RMSNorm 函数的性能
TEST_F(ChatGLMTest, BenchmarkRMSNorm) {
    # 定义序列长度和隐藏层大小
    constexpr int seq_len = 64;
    constexpr int hidden = 1024;

    # 创建 RMSNorm 模型对象
    RMSNorm m(&ctx, hidden);
    # 创建输入张量
    ggml_tensor *x = ggml_new_tensor_2d(ctx.ctx_b.get(), GGML_TYPE_F32, hidden, seq_len);

    # 将所有张量放入向量中
    std::vector<ggml_tensor *> all_tensors{m.weight, x};
    # 遍历所有张量，填充随机数据并转移到设备
    for (auto tensor : all_tensors) {
        random_fill(tensor);
        tensor_to_device(tensor);
    }

    # 进行前向传播
    ggml_tensor *y = m.forward(&ctx, x);
    ggml_build_forward_expand(&ctx.gf, y);
    # 输出性能信息
    std::cout << "[Benchmark] RMSNorm " << ggml_type_name(ctx.dtype) << " time: " << perf_device_graph_compute()
              << " ms\n";

    # 将所有张量转移到 CPU
    for (auto tensor : all_tensors) {
        tensor_to_cpu(tensor);
    }
}

# 测试 GLMModel 函数
TEST_F(ChatGLMTest, GLMModel) {
    # 获取数据文件路径
    fs::path data_path = fs::path(__FILE__).parent_path() / "tests/data/glm_model.data";

    # 创建模型配置对��
    ModelConfig config;
    config.hidden_size = 32;
    config.num_attention_heads = 8;
    config.num_kv_heads = 2;
    config.intermediate_size = config.hidden_size * 4;
    config.num_hidden_layers = 1;
    // 设置配置参数：词汇表大小为5，最大长度为8，归一化参数为1e-5
    config.vocab_size = 5;
    config.max_length = 8;
    config.norm_eps = 1e-5;

    // 定义常量序列长度为3
    constexpr int seq_len = 3;

    // 创建 ChatGLMModel 模型对象，传入上下文和配置参数
    ChatGLMModel model(&ctx, config);

    // 创建存储所有权重的向量，包括词嵌入权重、各层的权重和偏置等
    std::vector<ggml_tensor *> all_weights{model.word_embeddings.weight,
                                           model.layers[0].input_layernorm.weight,
                                           model.layers[0].input_layernorm.bias,
                                           model.layers[0].attention.query_key_value.weight,
                                           model.layers[0].attention.query_key_value.bias,
                                           model.layers[0].attention.dense.weight,
                                           model.layers[0].attention.dense.bias,
                                           model.layers[0].post_attention_layernorm.weight,
                                           model.layers[0].post_attention_layernorm.bias,
                                           model.layers[0].mlp.dense_h_to_4h.weight,
                                           model.layers[0].mlp.dense_h_to_4h.bias,
                                           model.layers[0].mlp.dense_4h_to_h.weight,
                                           model.layers[0].mlp.dense_4h_to_h.bias,
                                           model.final_layernorm.weight,
                                           model.final_layernorm.bias};

    // 测试模型，传入模型对象、配置参数、数据路径、序列长度和所有权重
    test_model(model, config, data_path, seq_len, all_weights);
// 结束测试用例
}

// 在 ChatGLMTest 中进行 BenchmarkGLMBlock 测试
TEST_F(ChatGLMTest, BenchmarkGLMBlock) {
    // 定义隐藏层大小
    constexpr int hidden_size = 4096;
    // 定义注意力头数
    constexpr int num_attention_heads = 32;
    // 定义隐藏层层数
    constexpr int num_hidden_layers = 28;
    // 定义最大长度
    constexpr int max_length = 2048;
    // 定义序列长度
    constexpr int seq_len = 64;

    // 定义不同数据类型的数组
    ggml_type dtypes[]{GGML_TYPE_F32, GGML_TYPE_F16, GGML_TYPE_Q8_0, GGML_TYPE_Q4_0};
    // 遍历不同数据类型
    for (const auto dtype : dtypes) {
        // 设置测试环境
        SetUp();

        // 设置上下文数据类型
        ctx.dtype = dtype;
        // 创建 GLMBlock 模型
        GLMBlock model(&ctx, hidden_size, num_attention_heads, num_hidden_layers, max_length);

        // 创建自注意力张量
        ggml_tensor *self_attn_x = ggml_new_tensor_2d(ctx.ctx_b.get(), GGML_TYPE_F32, hidden_size, seq_len);
        // 创建交叉注意力张量
        ggml_tensor *cross_attn_x = ggml_new_tensor_1d(ctx.ctx_b.get(), GGML_TYPE_F32, hidden_size);

        // 创建所有张量的向量
        std::vector<ggml_tensor *> all_tensors{model.input_layernorm.weight,
                                               model.input_layernorm.bias,
                                               model.attention.query_key_value.weight,
                                               model.attention.query_key_value.bias,
                                               model.attention.dense.weight,
                                               model.attention.dense.bias,
                                               model.post_attention_layernorm.weight,
                                               model.post_attention_layernorm.bias,
                                               model.mlp.dense_h_to_4h.weight,
                                               model.mlp.dense_h_to_4h.bias,
                                               model.mlp.dense_4h_to_h.weight,
                                               model.mlp.dense_4h_to_h.bias,
                                               self_attn_x,
                                               cross_attn_x};

        // 遍历所有张量
        for (auto tensor : all_tensors) {
            // 填充张量数据
            random_fill(tensor);
            // 将张量数据传输到设备
            tensor_to_device(tensor);
//         }

//         // 重置计算图
//         reset_cgraph();
//         {
//             // 对自注意力进行计算
//             ggml_tensor *self_attn_y = model.forward(&ctx, self_attn_x, 0, seq_len);
//             // 构建前向传播计算图
//             ggml_build_forward_expand(&ctx.gf, self_attn_y);
//             // 输出自注意力计算时间
//             std::cout << "[Benchmark] GLMBlock " << ggml_type_name(dtype)
//                       << " self attn time: " << perf_cpu_graph_compute() << " ms\n";
//         }

//         // 交叉注意力
//         reset_cgraph();
//         {
//             // 对交叉注意力进行计算
//             ggml_tensor *cross_attn_y = model.forward(&ctx, cross_attn_x, seq_len, seq_len);
//             // 构建前向传播计算图
//             ggml_build_forward_expand(&ctx.gf, cross_attn_y);
//             // 输出交叉注意力计算时间
//             std::cout << "[Benchmark] GLMBlock " << ggml_type_name(dtype)
//                       << " cross attn time: " << perf_device_graph_compute() << " ms\n";
//         }

//         // 将所有张量转移到 CPU
//         for (auto tensor : all_tensors) {
//             tensor_to_cpu(tensor);
//         }
//     }
// }

// 测试 GLM2 模型
TEST_F(ChatGLMTest, GLM2Model) {
    // 设置数据路径
    fs::path data_path = fs::path(__FILE__).parent_path() / "tests/data/glm2_model.data";

    // 配置模型
    ModelConfig config;
    config.vocab_size = 5;
    config.hidden_size = 32;
    config.num_attention_heads = 8;
    config.num_kv_heads = 2;
    config.num_hidden_layers = 1;
    config.intermediate_size = 48;
    config.norm_eps = 1e-5;
    config.max_length = 8;

    // 序列长度
    constexpr int seq_len = 3;

    // 创建 ChatGLM2Model 模型
    ChatGLM2Model model(&ctx, config);

    // 将第一层注意力机制的 k_cache 和 v_cache 转移到设备
    tensor_to_device(model.layers[0].attention.k_cache);
    tensor_to_device(model.layers[0].attention.v_cache);
    // 创建一个包含所有权重指针的向量，用于传递给测试模型函数
    std::vector<ggml_tensor *> all_weights{model.word_embeddings.weight,  // 词嵌入层的权重
                                           model.layers[0].input_layernorm.weight,  // 第一个层的输入层归一化权重
                                           model.layers[0].attention.query_key_value.weight,  // 第一个层的注意力机制查询键值权重
                                           model.layers[0].attention.query_key_value.bias,  // 第一个层的注意力机制查询键值偏置
                                           model.layers[0].attention.dense.weight,  // 第一个层的注意力机制密集层权重
                                           model.layers[0].post_attention_layernorm.weight,  // 第一个层的注意力后归一化权重
                                           model.layers[0].mlp.gate_proj.weight,  // 第一个层的多层感知机门控投影权重
                                           model.layers[0].mlp.up_proj.weight,  // 第一个层的多层感知机上投影权重
                                           model.layers[0].mlp.down_proj.weight,  // 第一个层的多层感知机下投影权重
                                           model.final_layernorm.weight};  // 最终归一化层的权重

    // 调用测试模型函数，传入模型、配置、数据路径、序列长度和所有权重指针
    test_model(model, config, data_path, seq_len, all_weights);
// 结束测试用例
}

// 在 ChatGLMTest 测试套件中执行 BenchmarkGLM2Block 测试用例
TEST_F(ChatGLMTest, BenchmarkGLM2Block) {
    // 定义常量
    constexpr int seq_len = 64;
    constexpr int hidden_size = 4096;
    constexpr int num_attention_heads = 32;
    constexpr int num_kv_heads = 2;
    constexpr int ffn_hidden_size = 13696;
    constexpr int max_length = 2048;

    // 根据不同的编译选项选择不同的数据类型
#ifdef GGML_USE_METAL
    ggml_type dtypes[]{GGML_TYPE_F16, GGML_TYPE_Q8_0, GGML_TYPE_Q4_1, GGML_TYPE_Q4_0};
#else
    ggml_type dtypes[]{GGML_TYPE_F32, GGML_TYPE_F16, GGML_TYPE_Q8_0, GGML_TYPE_Q4_0};
#endif

    // 遍历不同的数据类型
    for (const auto dtype : dtypes) {
        // 设置测试环境
        SetUp();

        // 设置上下文的数据类型
        ctx.dtype = dtype;
        // 创建 GLM2Block 模型
        GLM2Block model(&ctx, hidden_size, num_attention_heads, num_kv_heads, ffn_hidden_size, max_length, 1e-5);
        // 将注意力机制的 k_cache 和 v_cache 数据传输到设备
        tensor_to_device(model.attention.k_cache);
        tensor_to_device(model.attention.v_cache);

        // 创建 self_attn_x 和 cross_attn_x 张量
        ggml_tensor *self_attn_x = ggml_new_tensor_2d(ctx.ctx_b.get(), GGML_TYPE_F32, hidden_size, seq_len);
        ggml_tensor *cross_attn_x = ggml_new_tensor_1d(ctx.ctx_b.get(), GGML_TYPE_F32, hidden_size);

        // 创建包含所有张量的向量
        std::vector<ggml_tensor *> all_tensors{model.input_layernorm.weight,
                                               model.attention.query_key_value.weight,
                                               model.attention.query_key_value.bias,
                                               model.attention.dense.weight,
                                               model.post_attention_layernorm.weight,
                                               model.mlp.gate_proj.weight,
                                               model.mlp.up_proj.weight,
                                               model.mlp.down_proj.weight,
                                               self_attn_x,
                                               cross_attn_x};

        // 遍历所有张量，填充随机数据并传输到设备
        for (auto tensor : all_tensors) {
            random_fill(tensor);
            tensor_to_device(tensor);
        }
    }
}
// self attention
// 重置计算图
reset_cgraph();
{
    // 使用模型进行自注意力计算，得到输出 self_attn_y
    ggml_tensor *self_attn_y = model.forward(&ctx, self_attn_x, 0, seq_len);
    // 构建前向传播计算图
    ggml_build_forward_expand(&ctx.gf, self_attn_y);
    // 输出自注意力计算时间
    std::cout << "[Benchmark] GLM2Block " << ggml_type_name(dtype)
              << " self attn time: " << perf_device_graph_compute() << " ms\n";
}

// cross attention
// 重置计算图
reset_cgraph();
{
    // 使用模型进行交叉注意力计算，得到输出 cross_attn_y
    ggml_tensor *cross_attn_y = model.forward(&ctx, cross_attn_x, seq_len, seq_len);
    // 构建前向传播计算图
    ggml_build_forward_expand(&ctx.gf, cross_attn_y);
    // 输出交叉注意力计算时间
    std::cout << "[Benchmark] GLM2Block " << ggml_type_name(dtype)
              << " cross attn time: " << perf_device_graph_compute() << " ms\n";
}

// 将所有张量转移到 CPU
for (auto tensor : all_tensors) {
    tensor_to_cpu(tensor);
}
// 将模型的注意力缓存张量转移到 CPU
tensor_to_cpu(model.attention.k_cache);
tensor_to_cpu(model.attention.v_cache);

``` 
TEST_F(ChatGLMTest, GLM3Model) {
    // 设置数据路径
    fs::path data_path = fs::path(__FILE__).parent_path() / "tests/data/glm3_model.data";

    // 配置模型参数
    ModelConfig config;
    config.vocab_size = 5;
    config.hidden_size = 32;
    config.num_attention_heads = 8;
    config.num_kv_heads = 2;
    config.num_hidden_layers = 1;
    config.intermediate_size = 48;
    config.norm_eps = 1e-5;
    config.max_length = 8;

    // 定义序列长度
    constexpr int seq_len = 3;

    // 创建 ChatGLM3Model 模型对象
    ChatGLM3Model model(&ctx, config);

    // 将第一个层的注意力缓存张量转移到设备
    tensor_to_device(model.layers[0].attention.k_cache);
    tensor_to_device(model.layers[0].attention.v_cache);
}
    // 创建一个包含所有权重指针的向量，用于传递给测试模型函数
    std::vector<ggml_tensor *> all_weights{model.word_embeddings.weight,  // 词嵌入层的权重
                                           model.layers[0].input_layernorm.weight,  // 第一个层的输入层归一化权重
                                           model.layers[0].attention.query_key_value.weight,  // 第一个层的注意力机制查询键值权重
                                           model.layers[0].attention.query_key_value.bias,  // 第一个层的注意力机制查询键值偏置
                                           model.layers[0].attention.dense.weight,  // 第一个层的注意力机制密集层权重
                                           model.layers[0].post_attention_layernorm.weight,  // 第一个层的注意力后归一化权重
                                           model.layers[0].mlp.gate_proj.weight,  // 第一个层的多层感知机门控投影权重
                                           model.layers[0].mlp.up_proj.weight,  // 第一个层的多层感知机上投影权重
                                           model.layers[0].mlp.down_proj.weight,  // 第一个层的多层感知机下投影权重
                                           model.final_layernorm.weight};  // 最终归一化层的权重

    // 调用测试模型函数，传入模型、配置、数据路径、序列长度和所有权重指针
    test_model(model, config, data_path, seq_len, all_weights);
# 在 ChatGLMTest 测试类中的 Baichuan7BModel 测试用例
TEST_F(ChatGLMTest, Baichuan7BModel) {
    # 获取当前文件的路径，并拼接测试数据文件的路径
    fs::path data_path = fs::path(__FILE__).parent_path() / "tests/data/baichuan7b_model.data";

    # 创建模型配置对象，并设置各项参数
    ModelConfig config;
    config.hidden_size = 32;
    config.num_attention_heads = 8;
    config.num_kv_heads = config.num_attention_heads;
    config.intermediate_size = config.hidden_size * 3;
    config.num_hidden_layers = 1;
    config.vocab_size = 5;
    config.max_length = 8;
    config.norm_eps = 1e-6;

    # 定义序列长度
    constexpr int seq_len = 3;

    # 创建 Baichuan7BModel 模型对象
    Baichuan7BModel model(&ctx, config);

    # 收集模型中所有权重张量的指针
    std::vector<ggml_tensor *> all_weights{model.word_embeddings.weight,
                                           model.layers[0].input_layernorm.weight,
                                           model.layers[0].attention.query_key_value.weight,
                                           model.layers[0].attention.dense.weight,
                                           model.layers[0].post_attention_layernorm.weight,
                                           model.layers[0].mlp.gate_proj.weight,
                                           model.layers[0].mlp.down_proj.weight,
                                           model.layers[0].mlp.up_proj.weight,
                                           model.final_layernorm.weight};

    # 测试模型
    test_model(model, config, data_path, seq_len, all_weights);
}

# 在 ChatGLMTest 测试类中的 Baichuan13BModel 测试用例
TEST_F(ChatGLMTest, Baichuan13BModel) {
    # 获取当前文件的路径，并拼接测试数据文件的路径
    fs::path data_path = fs::path(__FILE__).parent_path() / "tests/data/baichuan13b_model.data";

    # 创建模型配置对象，并设置各项参数
    ModelConfig config;
    config.hidden_size = 32;
    config.num_attention_heads = 8;
    config.num_kv_heads = config.num_attention_heads;
    config.intermediate_size = config.hidden_size * 3;
    config.num_hidden_layers = 1;
    config.vocab_size = 5;
    config.max_length = 8;
    config.norm_eps = 1e-6;

    # 定义序列长度
    constexpr int seq_len = 3;

    # 创建 Baichuan13BModel 模型对象
    Baichuan13BModel model(&ctx, config);
    # 创建一个包含所有权重指针的向量，用于传递给测试模型函数
    std::vector<ggml_tensor *> all_weights{model.word_embeddings.weight,  # 将词嵌入层的权重指针添加到向量中
                                           model.layers[0].input_layernorm.weight,  # 将第一个层的输入层归一化层的权重指针添加到向量中
                                           model.layers[0].attention.query_key_value.weight,  # 将第一个层的注意力机制的查询、键、值权重指针添加到向量中
                                           model.layers[0].attention.dense.weight,  # 将第一个层的注意力机制的密集层权重指针添加到向量中
                                           model.layers[0].post_attention_layernorm.weight,  # 将第一个层的注意力后归一化层的权重指针添加到向量中
                                           model.layers[0].mlp.gate_proj.weight,  # 将第一个层的多层感知机的门控投影权重指针添加到向量中
                                           model.layers[0].mlp.down_proj.weight,  # 将第一个层的多层感知机的下投影权重指针添加到向量中
                                           model.layers[0].mlp.up_proj.weight,  # 将第一个层的多层感知机的上投影权重指针添加到向量中
                                           model.final_layernorm.weight};  # 将最终归一化层的权重指针添加到向量中
    
    # 调用测试模型函数，传入模型、配置、数据路径、序列长度和所有权重指针向量
    test_model(model, config, data_path, seq_len, all_weights);
// 在 ChatGLMTest 测试套件中的 InternLMModel 测试用例
TEST_F(ChatGLMTest, InternLMModel) {
    // 设置数据路径为当前文件所在目录下的 internlm_model.data 文件
    fs::path data_path = fs::path(__FILE__).parent_path() / "tests/data/internlm_model.data";

    // 配置模型参数
    ModelConfig config;
    config.hidden_size = 32;
    config.num_attention_heads = 8;
    config.num_kv_heads = config.num_attention_heads;
    config.intermediate_size = config.hidden_size * 3;
    config.num_hidden_layers = 1;
    config.vocab_size = 5;
    config.max_length = 8;
    config.norm_eps = 1e-6;

    // 定义序列长度为 3
    constexpr int seq_len = 3;

    // 创建 InternLM7BModel 模型对象
    InternLM7BModel model(&ctx, config);

    // 收集所有权重张量
    std::vector<ggml_tensor *> all_weights{model.word_embeddings.weight,
                                           model.layers[0].input_layernorm.weight,
                                           model.layers[0].attention.query_key_value.weight,
                                           model.layers[0].attention.query_key_value.bias,
                                           model.layers[0].attention.dense.weight,
                                           model.layers[0].attention.dense.bias,
                                           model.layers[0].post_attention_layernorm.weight,
                                           model.layers[0].mlp.gate_proj.weight,
                                           model.layers[0].mlp.up_proj.weight,
                                           model.layers[0].mlp.down_proj.weight,
                                           model.final_layernorm.weight};

    // 测试模型
    test_model(model, config, data_path, seq_len, all_weights);
}

// 在 ChatGLMTest 测试套件中的 quantize 测试用例
TEST_F(ChatGLMTest, quantize) {
    // 跳过量化数据生成
    GTEST_SKIP() << "Skipping quantization data generation";
    // 创建一个 128x2 的浮点型张量
    ggml_tensor *src = ggml_new_tensor_2d(ctx.ctx_b.get(), GGML_TYPE_F32, 128, 2);
    // 将数据拷贝到张量中
    memcpy(src->data, src_data, sizeof(src_data));

    // q8_0
}
    {
        // 创建一个新的 2D 张量，数据类型为 GGML_TYPE_Q8_0，大小为 128x2
        ggml_tensor *q8_dst = ggml_new_tensor_2d(ctx.ctx_b.get(), GGML_TYPE_Q8_0, 128, 2);
        // 创建一个长度为 16 的数组 hist，初始化为 0
        int64_t hist[16]{};
        // 对输入数据进行 Q8_0 类型的量化，并将结果存储在 q8_dst 中，同时更新 hist
        ggml_quantize_q8_0((float *)src->data, q8_dst->data, ggml_nelements(src), src->ne[0], hist);

        // 输出 Q8_0 类型的量化结果
        std::cout << "Q8: [";
        // 遍历 q8_dst 中的数据并输出
        for (size_t i = 0; i < ggml_nbytes(q8_dst); i++) {
            std::cout << (i > 0 ? ", " : "") << (int)((char *)q8_dst->data)[i];
        }
        std::cout << "]\n";
    }
    // q4_0
    {
        // 创建一个新的 2D 张量，数据类型为 GGML_TYPE_Q4_0，大小为 128x2
        ggml_tensor *q4_dst = ggml_new_tensor_2d(ctx.ctx_b.get(), GGML_TYPE_Q4_0, 128, 2);
        // 创建一个长度为 16 的数组 hist，初始化为 0
        int64_t hist[16]{};
        // 对输入数据进行 Q4_0 类型的量化，并将结果存储在 q4_dst 中，同时更新 hist
        ggml_quantize_q4_0((float *)src->data, q4_dst->data, ggml_nelements(src), src->ne[0], hist);

        // 输出 Q4_0 类型的量化结果
        std::cout << "Q4_0: [";
        // 遍历 q4_dst 中的数据并输出
        for (size_t i = 0; i < ggml_nbytes(q4_dst); i++) {
            std::cout << (i > 0 ? ", " : "") << (int)((char *)q4_dst->data)[i];
        }
        std::cout << "]\n";
    }
    // q4_1
    {
        // 创建一个新的 2D 张量，数据类型为 GGML_TYPE_Q4_1，大小为 128x2
        ggml_tensor *q4_dst = ggml_new_tensor_2d(ctx.ctx_b.get(), GGML_TYPE_Q4_1, 128, 2);
        // 创建一个长度为 16 的数组 hist，初始化为 0
        int64_t hist[16]{};
        // 对输入数据进行 Q4_1 类型的量化，并将结果存储在 q4_dst 中，同时更新 hist
        ggml_quantize_q4_1((float *)src->data, q4_dst->data, ggml_nelements(src), src->ne[0], hist);

        // 输出 Q4_1 类型的量化结果
        std::cout << "Q4_1: [";
        // 遍历 q4_dst 中的数据并输出
        for (size_t i = 0; i < ggml_nbytes(q4_dst); i++) {
            std::cout << (i > 0 ? ", " : "") << (int)((char *)q4_dst->data)[i];
        }
        std::cout << "]\n";
    }
    // q5_0
    {
        // 创建一个新的 2D 张量，数据类型为 GGML_TYPE_Q5_0，大小为 128x2
        ggml_tensor *q5_dst = ggml_new_tensor_2d(ctx.ctx_b.get(), GGML_TYPE_Q5_0, 128, 2);
        // 创建一个长度为 16 的数组 hist，初始化为 0
        int64_t hist[16]{};
        // 对输入数据进行 Q5_0 类型的量化，并将结果存储在 q5_dst 中，同时更新 hist
        ggml_quantize_q5_0((float *)src->data, q5_dst->data, ggml_nelements(src), src->ne[0], hist);

        // 输出 Q5_0 类型的量化结果
        std::cout << "Q5_0: [";
        // 遍历 q5_dst 中的数据并输出
        for (size_t i = 0; i < ggml_nbytes(q5_dst); i++) {
            std::cout << (i > 0 ? ", " : "") << (int)((char *)q5_dst->data)[i];
        }
        std::cout << "]\n";
    }
    // q5_1
    {
        // 创建一个新的 2 维 Q5 张量，大小为 128x2
        ggml_tensor *q5_dst = ggml_new_tensor_2d(ctx.ctx_b.get(), GGML_TYPE_Q5_1, 128, 2);
        // 初始化一个长度为 16 的整型数组 hist，所有元素初始化为 0
        int64_t hist[16]{};
        // 对输入数据进行 Q5_1 格式的量化，将结果存储到 q5_dst 中，并更新直方图 hist
        ggml_quantize_q5_1((float *)src->data, q5_dst->data, ggml_nelements(src), src->ne[0], hist);
    
        // 输出 Q5_1 格式的数据到控制台
        std::cout << "Q5_1: [";
        // 遍历 Q5 张量的数据，输出每个元素的值
        for (size_t i = 0; i < ggml_nbytes(q5_dst); i++) {
            std::cout << (i > 0 ? ", " : "") << (int)((char *)q5_dst->data)[i];
        }
        std::cout << "]\n";
    }
// 定义一个结构体，用于存储测试用例的提示、输入和是否跳过解码的信息
struct TokenizerTestCase {
    std::string prompt;
    std::vector<int> input_ids;
    bool skip_decode = false;
};

// 检查分词器的函数，接受一个分词器和测试用例的向量作为参数
static void check_tokenizer(const BaseTokenizer *tokenizer, const std::vector<TokenizerTestCase> &cases) {
    // 遍历测试用例
    for (const auto &c : cases) {
        // 编码
        std::vector<int> input_ids = tokenizer->encode(c.prompt, 2048);
        // 断言输入的编码结果与预期的输入编码一致
        EXPECT_TRUE(equal(input_ids, c.input_ids));
        if (!c.skip_decode) {
            // 解码
            std::string output = tokenizer->decode(c.input_ids);
            // 断言解码结果与提示信息一致
            EXPECT_EQ(output, c.prompt);
        }
    }
}

// 测试 Pipeline 中的 ChatGLM 函数
TEST(Pipeline, ChatGLM) {
    // 获取模型路径
    fs::path model_path = fs::path(__FILE__).parent_path() / "chatglm-ggml.bin";
    // 如果模型路径不存在，则跳过测试
    if (!fs::exists(model_path)) {
        GTEST_SKIP() << "Skipping ChatGLM e2e test (ggml model not found)";
    }
    // 创建 Pipeline 对象
    Pipeline pipeline(model_path.string());
    // 断言 Pipeline 中的模型是 ChatGLMForCausalLM 类型
    EXPECT_TRUE(dynamic_cast<ChatGLMForCausalLM *>(pipeline.model.get()));

    // 分词器
    {
        // 定义分词器测试用例
        std::vector<TokenizerTestCase> cases{
            {"你好", {5, 74874, 130001, 130004}},
            {"[Round 0]\n问：你好\n答：你好👋！我是人工智能助手 ChatGLM-6B，很高兴见到你，欢迎问我任何问题。\n[Round "
             "1]\n问：晚上睡不着应该怎么办\n答：",
             {53,     6945,   5,      8,     42,    4,     64286,  12,    74874, 4,   67342,  12,    74874, 130328,
              130247, 130233, 130227, 35,    65806, 68241, 75890,  14132, 5388,  340, 11,     21,    222,   6,
              76693,  66877,  63852,  6,     66430, 68747, 102501, 63823, 4,     52,  6945,   5,     9,     42,
              4,      64286,  12,     65450, 83400, 64213, 66846,  4,     67342, 12,  130001, 130004}},
            {"def main():\n    print('hello world')\t# greeting",
             {1616, 594, 125936, 4, 130011, 2274, 89, 7283, 398, 125686, 130008, 61, 25672, 130001, 130004}}};
        // 检查分词器
        check_tokenizer(pipeline.tokenizer.get(), cases);
    }

    // prompter
}
    {
        // 测试 ChatGLMTokenizer 类的 build_prompt 方法，输入为用户角色和消息内容的映射，期望输出为消息内容
        EXPECT_EQ(ChatGLMTokenizer::build_prompt({{ChatMessage::ROLE_USER, "你好"}}), "你好");
        // 测试 ChatGLMTokenizer 类的 build_prompt 方法，输入为多个用户角色和消息内容的映射，期望输出为特定格式的字符串
        EXPECT_EQ(
            ChatGLMTokenizer::build_prompt({
                {ChatMessage::ROLE_USER, "你好"},
                {ChatMessage::ROLE_ASSISTANT, "你好👋！我是人工智能助手 ChatGLM-6B，很高兴见到你，欢迎问我任何问题。"},
                {ChatMessage::ROLE_USER, "晚上睡不着应该怎么办"},
            }),
            "[Round 0]\n问：你好\n答：你好👋！我是人工智能助手 "
            "ChatGLM-6B，很高兴见到你，欢迎问我任何问题。\n[Round 1]\n问：晚上睡不着应该怎么办\n答：");
    }

    // 内存测试
    {
        // 创建 GenerationConfig 对象
        GenerationConfig gen_config;
        gen_config.max_length = 2048;
        gen_config.max_context_length = gen_config.max_length - 1;
        gen_config.do_sample = false;

        // 创建字符串流对象 oss，循环添加 "你好" 到字符串流中
        std::ostringstream oss;
        for (int i = 0; i < gen_config.max_context_length; i++) {
            oss << "你好";
        }
        // 创建包含用户角色和消息内容的 ChatMessage 对象的向量
        std::vector<ChatMessage> messages{{ChatMessage::ROLE_USER, oss.str()}};
        // 调用 pipeline 的 chat 方法
        pipeline.chat(messages, gen_config);
    }

    // 聊天
    {
        // 创建 GenerationConfig 对象
        GenerationConfig gen_config;
        gen_config.do_sample = false;
        // 创建包含用户角色和消息内容的 ChatMessage 对象的向量
        std::vector<ChatMessage> messages{{ChatMessage::ROLE_USER, "你好"}};
        // 调用 pipeline 的 chat 方法，获取输出的 ChatMessage 对象
        ChatMessage output = pipeline.chat(messages, gen_config);
        // 验证输出的消息内容是否符合预期
        EXPECT_EQ(output.content, "你好👋！我是人工智能助手 ChatGLM-6B，很高兴见到你，欢迎问我任何问题。");
    }

}
// 测试 Pipeline 类中的 ChatGLM2 方法
TEST(Pipeline, ChatGLM2) {
    // 获取当前文件的路径，并拼接 ggml 模型文件的路径
    fs::path model_path = fs::path(__FILE__).parent_path() / "chatglm2-ggml.bin";
    // 如果 ggml 模型文件不存在，则跳过测试
    if (!fs::exists(model_path)) {
        GTEST_SKIP() << "Skipping ChatGLM2 e2e test (ggml model not found)";
    }
    // 创建 Pipeline 对象，加载模型
    Pipeline pipeline(model_path.string());
    // 断言模型类型为 ChatGLM2ForCausalLM
    EXPECT_TRUE(dynamic_cast<ChatGLM2ForCausalLM *>(pipeline.model.get()));

    // tokenizer
    {
        // 定义 Tokenizer 测试用例
        std::vector<TokenizerTestCase> cases{
            {"你好", {64790, 64792, 36474, 54591}},
            {"[Round 1]\n\n问：你好\n\n答：",
             {64790, 64792, 790, 30951, 517, 30910, 30939, 30996, 13, 13, 54761, 31211, 39701, 13, 13, 55437, 31211}},
            {"[Round 1]\n\n问：你好\n\n答：你好👋！我是人工智能助手 "
             "ChatGLM2-6B，很高兴见到你，欢迎问我任何问题。\n\n[Round 2]\n\n问：晚上睡不着应该怎么办\n\n答：",
             {64790, 64792, 790,   30951, 517,   30910, 30939, 30996, 13,    13,    54761, 31211, 39701,
              13,    13,    55437, 31211, 39701, 243,   162,   148,   142,   31404, 33030, 34797, 42481,
              22011, 10461, 30944, 30943, 30941, 30978, 30949, 31123, 48895, 35214, 54622, 31123, 32616,
              39905, 31901, 31639, 31155, 13,    13,    30995, 30951, 517,   30910, 30943, 30996, 13,
              13,    54761, 31211, 32820, 54266, 31876, 35153, 13,    13,    55437, 31211}},
            {"def main():\n    print('hello world')\t# greeting",
             {64790, 64792, 884, 1301, 9427, 13, 296, 4466, 2029, 15616, 30914, 993, 3387, 12, 31010, 30174}}};
        // 检查 Tokenizer 的输出是否符合预期
        check_tokenizer(pipeline.tokenizer.get(), cases);
    }

    // prompter
}
    {
        // 测试 ChatGLM2Tokenizer 类的 build_prompt 方法，生成包含用户消息的提示文本
        EXPECT_EQ(ChatGLM2Tokenizer::build_prompt({{ChatMessage::ROLE_USER, "你好"}}), "[Round 1]\n\n问：你好\n\n答：");
        // 测试 ChatGLM2Tokenizer 类的 build_prompt 方法，生成包含多条消息的提示文本
        EXPECT_EQ(
            ChatGLM2Tokenizer::build_prompt({
                {ChatMessage::ROLE_USER, "你好"},
                {ChatMessage::ROLE_ASSISTANT, "你好👋！我是人工智能助手 ChatGLM2-6B，很高兴见到你，欢迎问我任何问题。"},
                {ChatMessage::ROLE_USER, "晚上睡不着应该怎么办"},
            }),
            "[Round 1]\n\n问：你好\n\n答：你好👋！我是人工智能助手 "
            "ChatGLM2-6B，很高兴见到你，欢迎问我任何问题。\n\n[Round 2]\n\n问：晚上睡不着应该怎么办\n\n答：");
    }

    // 内存测试
    {
        // 创建 GenerationConfig 对象，设置生成配置参数
        GenerationConfig gen_config;
        gen_config.max_length = 2048;
        gen_config.max_context_length = gen_config.max_length - 1;
        gen_config.do_sample = false;

        // 创建字符串流对象 oss，生成指定长度的字符串
        std::ostringstream oss;
        for (int i = 0; i < gen_config.max_context_length; i++) {
            oss << "你好";
        }
        // 创建包含用户消息的 ChatMessage 对象列表
        std::vector<ChatMessage> messages{{ChatMessage::ROLE_USER, oss.str()}};
        // 调用 pipeline 的 chat 方法进行对话生成
        pipeline.chat(messages, gen_config);
    }

    // 对话测试
    {
        // 创建 GenerationConfig 对象，设置生成配置参数
        GenerationConfig gen_config;
        gen_config.do_sample = false;
        // 创建包含用户消息的 ChatMessage 对象列表
        std::vector<ChatMessage> messages{{ChatMessage::ROLE_USER, "你好"}};
        // 调用 pipeline 的 chat 方法进行对话生成
        ChatMessage output = pipeline.chat(messages, gen_config);
        // 验证生成的对话输出是否符合预期
        EXPECT_EQ(output.content, "你好👋！我是人工智能助手 ChatGLM2-6B，很高兴见到你，欢迎问我任何问题。");
    }

}
// 读取指定路径的文本文件内容并返回字符串
static inline std::string read_text(const fs::path &path) {
    // 使用MappedFile类读取文件内容
    MappedFile mapped_file(path.string());
    // 将文件内容转换为字符串并返回
    return std::string(mapped_file.data, mapped_file.size);
}

// 测试ChatGLM3管道
TEST(Pipeline, ChatGLM3) {
    // 获取模型文件路径
    fs::path model_path = fs::path(__FILE__).parent_path() / "chatglm3-ggml.bin";
    // 如果模型文件不存在，则跳过测试
    if (!fs::exists(model_path)) {
        GTEST_SKIP() << "Skipping ChatGLM3 e2e test (ggml model not found)";
    }
    // 创建Pipeline对象并加载模型
    Pipeline pipeline(model_path.string());
    // 断言模型类型为ChatGLM3ForCausalLM
    EXPECT_TRUE(dynamic_cast<ChatGLM3ForCausalLM *>(pipeline.model.get()));

    // 读取系统工具调用文本内容
    const std::string system_tool_call =
        read_text(fs::path(__FILE__).parent_path() / "examples/system/function_call.txt");
    // 读取系统代码解释器文本内容
    const std::string system_ci = read_text(fs::path(__FILE__).parent_path() / "examples/system/code_interpreter.txt");

    // 分词器测试
    {
        // 目标输入ID数组
        std::vector<int> target_ids{64790, 64792, 36474, 54591};
        // 使用分词器对文本"你好"进行编码
        std::vector<int> input_ids = pipeline.tokenizer->encode("你好", 2048);
        // 断言编码结果与目标ID数组相等
        EXPECT_EQ(input_ids, target_ids);
    }
    {
        // 聊天消息数组
        std::vector<ChatMessage> messages{{ChatMessage::ROLE_USER, "你好"}};
        // 使用分词器对聊天消息数组进行编码
        std::vector<int> input_ids = pipeline.tokenizer->encode_messages(messages, 2048);
        // 目标输入ID数组
        std::vector<int> target_ids{64790, 64792, 64795, 30910, 13, 36474, 54591, 64796};
        // 断言编码结果与目标ID数组相等
        EXPECT_EQ(input_ids, target_ids);
    }
}
    {
        // 创建包含聊天消息的向量
        std::vector<ChatMessage> messages{
            {ChatMessage::ROLE_USER, "你好"},
            {ChatMessage::ROLE_ASSISTANT, "你好👋！我是人工智能助手 ChatGLM3-6B，很高兴见到你，欢迎问我任何问题。"},
            {ChatMessage::ROLE_USER, "晚上睡不着应该怎么办"},
        };
        // 使用pipeline的tokenizer对消息进行编码，限制最大长度为2048
        std::vector<int> input_ids = pipeline.tokenizer->encode_messages(messages, 2048);
        // 创建目标ID向量
        std::vector<int> target_ids{64790, 64792, 64795, 30910, 13,    36474, 54591, 64796, 30910, 13,    36474, 54591,
                                    243,   162,   148,   142,   31404, 33030, 34797, 42481, 22011, 10461, 30944, 30966,
                                    30941, 30978, 30949, 31123, 48895, 35214, 54622, 31123, 32616, 39905, 31901, 31639,
                                    31155, 64795, 30910, 13,    30910, 32820, 54266, 31876, 35153, 64796};
        // 断言输入ID与目标ID相等
        EXPECT_EQ(input_ids, target_ids);
    }
    }

    // memory test
    {
        // 创建生成配置对象
        GenerationConfig gen_config;
        gen_config.max_length = 2048;
        gen_config.max_context_length = gen_config.max_length - 1;
        gen_config.do_sample = false;

        // 创建包含大量"你好"字符串的消息向量
        std::ostringstream oss;
        for (int i = 0; i < gen_config.max_context_length; i++) {
            oss << "你好";
        }
        std::vector<ChatMessage> messages{{ChatMessage::ROLE_USER, oss.str()}};
        // 使用pipeline进行聊天
        pipeline.chat(messages, gen_config);
    }

    // chat
    {
        // 创建生成配置对象
        GenerationConfig gen_config;
        gen_config.do_sample = false;
        // 创建包含"你好"字符串的消息向量
        std::vector<ChatMessage> messages{{ChatMessage::ROLE_USER, "你好"}};
        // 进行聊天并获取输出消息
        ChatMessage output = pipeline.chat(messages, gen_config);
        // 断言输出消息内容与预期相等
        EXPECT_EQ(output.content, "你好👋！我是人工智能助手 ChatGLM3-6B，很高兴见到你，欢迎问我任何问题。");
    }

    // tool call
    {
        // 创建生成配置对象
        GenerationConfig gen_config;
        // 设置生成配置对象的 do_sample 属性为 false
        gen_config.do_sample = false;
        // 创建包含系统工具调用和用户消息的消息向量
        std::vector<ChatMessage> messages{
            {ChatMessage::ROLE_SYSTEM, system_tool_call},
            {ChatMessage::ROLE_USER, "生成一个随机数"},
        };
        {
            // 调用 pipeline 的 chat 方法，生成回复消息
            ChatMessage output = pipeline.chat(messages, gen_config);
            // 检查回复消息的角色和内容是否符合预期
            EXPECT_EQ(output.role, ChatMessage::ROLE_ASSISTANT);
            EXPECT_EQ(output.content, "```python\n"
                                      "tool_call(seed=42, range=(0, 100))\n"
                                      "```");
            // 将回复消息添加到消息向量中
            messages.emplace_back(std::move(output));
        }
        // 向消息向量中添加观察者消息
        messages.emplace_back(ChatMessage::ROLE_OBSERVATION, "22");
        {
            // 再次调用 pipeline 的 chat 方法，生成回复消息
            ChatMessage output = pipeline.chat(messages, gen_config);
            // 检查回复消息的角色和内容是否符合预期
            EXPECT_EQ(output.role, ChatMessage::ROLE_ASSISTANT);
            EXPECT_EQ(output.content, "根据您的要求，我使用随机数生成器API生成了一个在0和100之间的随机数，结果为22。");
        }
    }

    // 代码解释器
    {
        // 创建生成配置对象
        GenerationConfig gen_config;
        // 设置生成配置对象的 do_sample 属性为 false
        gen_config.do_sample = false;
        // 创建包含系统代码解释器和用户消息的消息向量
        std::vector<ChatMessage> messages{
            {ChatMessage::ROLE_SYSTEM, system_ci},
            {ChatMessage::ROLE_USER, "列出100以内的所有质数"},
        };
        {
            // 调用 pipeline 的 chat 方法，生成回复消息
            ChatMessage output = pipeline.chat(messages, gen_config);
            // 检查回复消息的角色和内容是否符合预期
            EXPECT_EQ(output.role, ChatMessage::ROLE_ASSISTANT);
            EXPECT_EQ(output.content, "好的，我会为您列出100以内的所有质数。\n\n质数是指只能被1和它本身整除的大于1"
                                      "的整数。例如，2、3、5、7等都是质数。\n\n让我们开始吧！");
            // 检查回复消息的工具调用代码输入是否符合预期
            EXPECT_EQ(output.tool_calls.front().code.input, R"(```python
# 定义一个冒泡排序函数，接受一个列表作为参数
def bubble_sort(list):
    # 遍历列表中的元素，从第一个元素到倒数第二个元素
    for i in range(len(list) - 1):
        # 再次遍历列表中的元素，从第一个元素到倒数第二个元素
        for j in range(len(list) - 1):
            # 如果当前元素大于下一个元素
            if list[j] > list[j + 1]:
                # 交换当前元素和下一个元素的位置
                list[j], list[j + 1] = list[j + 1], list[j]
    # 返回排序后的列表
    return list
// 打印冒泡排序后的结果
print(bubble_sort([5, 4, 3, 2, 1]));

// 生成输出结果
std::string output = pipeline.generate(prompt, gen_config);
// 断言输出结果与目标结果相等
EXPECT_EQ(output, target);
}

// 测试Baichuan13B模型
TEST(Pipeline, Baichuan13B) {
    // 获取模型路径
    fs::path model_path = fs::path(__FILE__).parent_path() / "baichuan-13b-chat-ggml.bin";
    // 如果模型路径不存在，则跳过测试
    if (!fs::exists(model_path)) {
        GTEST_SKIP() << "Skipping Baichuan13B e2e test (ggml model not found)";
    }
    // 创建Pipeline对象
    Pipeline pipeline(model_path.string());
    // 断言模型类型为Baichuan13BForCausalLM
    EXPECT_TRUE(dynamic_cast<Baichuan13BForCausalLM *>(pipeline.model.get()));

    // tokenizer
    {
        // 定义测试用例
        std::vector<TokenizerTestCase> cases{
            {"你是谁", {9875, 21797}},
            {"我是百川大模型，是由百川智能的工程师们创造的大语言模型，我可以和人类进行自然交流、解答问题、协助创作，帮"
             "助大众轻松、普惠的获得世界知识和专业服务。如果你有任何问题，可以随时向我提问",
             {6323,  31161, 31745, 32213, 31175, 14830, 72,    16347, 31745, 32213, 6358,  31135, 14823, 31212, 8823,
              5114,  7234,  14830, 72,    31182, 1231,  31188, 8627,  1696,  3823,  5536,  76,    17133, 1766,  76,
              16345, 11769, 72,    4090,  13169, 8385,  76,    31840, 32195, 31135, 4137,  2781,  3317,  31188, 2285,
              1910,  73,    6011,  31169, 4315,  1766,  72,    1231,  11533, 31490, 31182, 21934}}};
        // 检查tokenizer的输出是否符合预期
        check_tokenizer(pipeline.tokenizer.get(), cases);

        // 定义聊天消息
        std::vector<ChatMessage> messages{
            {ChatMessage::ROLE_USER, "你好呀"},
            {ChatMessage::ROLE_ASSISTANT, "你好！很高兴和你交流。请问有什么我可以帮助你的吗？"},
            {ChatMessage::ROLE_USER, "你叫什么名字？"},
        };
        // 使用tokenizer编码消息
        std::vector<int> input_ids = pipeline.tokenizer->encode_messages(messages, 2048);
        // 定义目标输入id
        std::vector<int> target_input_ids{195,   9875, 31213, 32889, 196,  9875,  31213, 74,   17318, 31906,
                                          14822, 5536, 73,    20389, 7713, 31182, 1231,  4090, 2689,  31763,
                                          75,    195,  9875,  32177, 1534, 10240, 75,    196};
        // 断言编码结果与目标结果相等
        EXPECT_TRUE(equal(input_ids, target_input_ids));
    }

    // memory test
    {
        // 创建一个 GenerationConfig 对象，用于配置生成器的参数
        GenerationConfig gen_config;
        // 设置生成的文本最大长度为 512
        gen_config.max_length = 512;
        // 设置上下文的最大长度为生成文本最大长度减去1
        gen_config.max_context_length = gen_config.max_length - 1;
        // 设置不进行采样
        gen_config.do_sample = false;

        // 创建一个包含128的整数向量，长度为上下文的最大长度
        std::vector<int> input_ids(gen_config.max_context_length, 128);
        // 使用生成器生成文本，传入输入向量和生成配置
        pipeline.generate(input_ids, gen_config);
    }

    // chat
    {
        // 创建一个 GenerationConfig 对象，用于配置对话生成器的参数
        GenerationConfig gen_config;
        // 设置不进行采样
        gen_config.do_sample = false;
        // 设置重复惩罚系数为1.1
        gen_config.repetition_penalty = 1.1;
        // 创建一个包含用户角色和消息内容的对话消息向量
        std::vector<ChatMessage> messages{{ChatMessage::ROLE_USER, "你好呀"}};
        // 使用对话生成器生成对话消息，传入消息向量和生成配置
        ChatMessage output = pipeline.chat(messages, gen_config);
        // 断言生成的对话消息内容与期望的内容相等
        EXPECT_EQ(output.content, "你好！很高兴见到你。请问有什么我可以帮助你的吗？");
    }
// 测试 Pipeline 类中的 Baichuan2_7B 方法
TEST(Pipeline, Baichuan2_7B) {
    // 获取模型文件路径
    fs::path model_path = fs::path(__FILE__).parent_path() / "baichuan2-7b-chat-ggml.bin";
    // 如果模型文件不存在，则跳过测试
    if (!fs::exists(model_path)) {
        GTEST_SKIP() << "Skipping Baichuan2-7B e2e test (ggml model not found)";
    }
    // 创建 Pipeline 对象
    Pipeline pipeline(model_path.string());
    // 断言 Pipeline 模型是 Baichuan7BForCausalLM 类型的对象
    EXPECT_TRUE(dynamic_cast<Baichuan7BForCausalLM *>(pipeline.model.get()));

    // tokenizer
    {
        // 定义 TokenizerTestCase 测试用例
        std::vector<TokenizerTestCase> cases{
            {"你是谁", {92067}},
            {"我是百川大模型，是由百川智能的工程师们创造的大语言模型，我可以和人类进行自然交流、解答问题、协助创作，帮"
             "助大众轻松、普惠的获得世界知识和专业服务。如果你有任何问题，可以随时向我提问",
             {6461, 70335, 92366, 9528, 65,    10879, 70335, 3932, 92333, 8832,  92414, 5034,
              3133, 5002,  9528,  65,   28756, 92385, 5243,  1697, 2559,  3341,  69,    10474,
              1754, 69,    9036,  7356, 65,    2716,  7499,  4892, 69,    24816, 92333, 2693,
              2089, 23672, 1940,  1760, 66,    4173,  23181, 1754, 65,    65351, 39975, 14590}}};
        // 检查 Tokenizer 测试用例
        check_tokenizer(pipeline.tokenizer.get(), cases);

        // 定义 ChatMessage 对话消息
        std::vector<ChatMessage> messages{
            {ChatMessage::ROLE_USER, "你好呀"},
            {ChatMessage::ROLE_ASSISTANT, "你好！很高兴和你交流。请问有什么问题我可以帮助你解决吗？"},
            {ChatMessage::ROLE_USER, "你叫什么名字？"},
        };
        // 使用 Tokenizer 对话消息编码
        std::vector<int> input_ids = pipeline.tokenizer->encode_messages(messages, 2048);
        // 目标输入 ID
        std::vector<int> target_input_ids{195, 16829, 94278, 196,   16829, 67,    52160, 10329, 3341,
                                          66,  23216, 5817,  1754,  92392, 21777, 92430, 2740,  93122,
                                          68,  195,   92430, 93410, 1747,  6642,  68,    196};
        // 断言输入 ID 与目标输入 ID 相等
        EXPECT_TRUE(equal(input_ids, target_input_ids));
    }

    // memory test
}
    {
        // 创建一个 GenerationConfig 对象，用于配置生成器的参数
        GenerationConfig gen_config;
        // 设置生成的最大长度为 2048
        gen_config.max_length = 2048;
        // 设置上下文的最大长度为生成的最大长度减去1
        gen_config.max_context_length = gen_config.max_length - 1;
        // 设置是否进行采样为 false
        gen_config.do_sample = false;

        // 创建一个包含128个元素的整数向量 input_ids
        std::vector<int> input_ids(gen_config.max_context_length, 128);
        // 使用生成器生成文本，传入 input_ids 和 gen_config
        pipeline.generate(input_ids, gen_config);
    }

    // chat
    {
        // 创建一个 GenerationConfig 对象，用于配置对话生成器的参数
        GenerationConfig gen_config;
        // 设置是否进行采样为 false
        gen_config.do_sample = false;
        // 设置重复惩罚系数为1.05
        gen_config.repetition_penalty = 1.05;
        // 创建一个包含一个 ChatMessage 对象的向量 messages，包含用户角色和消息内容
        std::vector<ChatMessage> messages{{ChatMessage::ROLE_USER, "你好呀"}};
        // 使用对话生成器生成回复，传入消息向量和 gen_config
        ChatMessage output = pipeline.chat(messages, gen_config);
        // 断言生成的回复内容与期望的内容相等
        EXPECT_EQ(output.content, "你好！很高兴为你服务。请问有什么问题我可以帮助你解决？");
    }
// 测试 Baichuan2-13B 模型
TEST(Pipeline, Baichuan2_13B) {
    // 获取模型文件路径
    fs::path model_path = fs::path(__FILE__).parent_path() / "baichuan2-13b-chat-ggml.bin";
    // 如果模型文件不存在，则跳过测试
    if (!fs::exists(model_path)) {
        GTEST_SKIP() << "Skipping Baichuan2-13B e2e test (ggml model not found)";
    }
    // 创建 Pipeline 对象
    Pipeline pipeline(model_path.string());
    // 断言模型类型为 Baichuan13BForCausalLM
    EXPECT_TRUE(dynamic_cast<Baichuan13BForCausalLM *>(pipeline.model.get()));

    // tokenizer
    {
        // 定义 Tokenizer 测试用例
        std::vector<TokenizerTestCase> cases{
            {"你是谁", {92067}},
            {"我是百川大模型，是由百川智能的工程师们创造的大语言模型，我可以和人类进行自然交流、解答问题、协助创作，帮"
             "助大众轻松、普惠的获得世界知识和专业服务。如果你有任何问题，可以随时向我提问",
             {6461, 70335, 92366, 9528, 65,    10879, 70335, 3932, 92333, 8832,  92414, 5034,
              3133, 5002,  9528,  65,   28756, 92385, 5243,  1697, 2559,  3341,  69,    10474,
              1754, 69,    9036,  7356, 65,    2716,  7499,  4892, 69,    24816, 92333, 2693,
              2089, 23672, 1940,  1760, 66,    4173,  23181, 1754, 65,    65351, 39975, 14590}}};
        // 检查 Tokenizer 测试用例
        check_tokenizer(pipeline.tokenizer.get(), cases);

        // 定义聊天消息
        std::vector<ChatMessage> messages{
            {ChatMessage::ROLE_USER, "你好呀"},
            {ChatMessage::ROLE_ASSISTANT, "你好！很高兴和你交流。请问有什么我可以帮助你的吗？"},
            {ChatMessage::ROLE_USER, "你叫什么名字？"},
        };
        // 编码聊天消息
        std::vector<int> input_ids = pipeline.tokenizer->encode_messages(messages, 2048);
        // 目标输入 ID
        std::vector<int> target_input_ids{195,   16829, 94278, 196,   16829, 67,  52160, 10329, 3341, 66,   23216, 5817,
                                          92392, 21777, 2193,  93122, 68,    195, 92430, 93410, 1747, 6642, 68,    196};
        // 断言输入 ID 与目标输入 ID 相等
        EXPECT_TRUE(equal(input_ids, target_input_ids));
    }

    // chat
    {
        // 生成配置
        GenerationConfig gen_config;
        gen_config.do_sample = false;
        gen_config.repetition_penalty = 1.05;
        // 定义聊天消息
        std::vector<ChatMessage> messages{{ChatMessage::ROLE_USER, "你好呀"}};
        // 进行聊天
        ChatMessage output = pipeline.chat(messages, gen_config);
        // 断言输出内容
        EXPECT_EQ(output.content, "你好！很高兴见到你。请问有什么我可以帮助你的吗？");
    }

这是一个代码块的结束标记，表示一个函数或者一个循环的结束。
// 测试 Pipeline 类中的 InternLM 方法
TEST(Pipeline, InternLM) {
    // 获取模型路径
    fs::path model_path = fs::path(__FILE__).parent_path() / "internlm-chat-7b-ggml.bin";
    // 如果模型路径不存在，则跳过测试
    if (!fs::exists(model_path)) {
        GTEST_SKIP() << "Skipping InternLM e2e test (ggml model not found)";
    }
    // 创建 Pipeline 对象
    Pipeline pipeline(model_path.string());
    // 断言 Pipeline 模型是 InternLM7BForCausalLM 类型的指针
    EXPECT_TRUE(dynamic_cast<InternLM7BForCausalLM *>(pipeline.model.get()));

    // tokenizer
    {
        // 定义 Tokenizer 测试用例
        std::vector<TokenizerTestCase> cases{
            {"你好", {1, 76379}},
            {"<|User|>:你好<eoh>\n<|Bot|>:你好，有什么我可以帮助你的吗？<eoa>\n<|User|>:晚上睡不着应该怎么办<eoh>\n<|"
             "Bot|>:",
             {1,     333,   352,   1621,  352,   27232,  76379, 103027, 364,    333,   352, 23845, 352,  27232,
              76379, 98899, 68408, 73159, 67566, 67513,  61056, 99050,  103028, 364,   333, 352,   1621, 352,
              27232, 67891, 76046, 67551, 68573, 103027, 364,   333,    352,    23845, 352, 27232},
             true}};
        // 检查 Tokenizer 测试用例
        check_tokenizer(pipeline.tokenizer.get(), cases);
    }

    // prompter
    {
        // 断言生成的提示符符合预期
        EXPECT_EQ(InternLMTokenizer::build_prompt({{ChatMessage::ROLE_USER, "你好"}}), "<|User|>:你好<eoh>\n<|Bot|>:");
        EXPECT_EQ(InternLMTokenizer::build_prompt({
                      {ChatMessage::ROLE_USER, "你好"},
                      {ChatMessage::ROLE_ASSISTANT, "你好，有什么我可以帮助你的吗？"},
                      {ChatMessage::ROLE_USER, "晚上睡不着应该怎么办"},
                  }),
                  "<|User|>:你好<eoh>\n<|Bot|>:你好，有什么我可以帮助你的吗？<eoa>\n<|User|>:晚上睡不着应该怎么办<eoh>"
                  "\n<|Bot|>:");
    }

    // memory test
    {
        // 生成配置
        GenerationConfig gen_config;
        gen_config.max_length = 2048;
        gen_config.max_context_length = gen_config.max_length - 1;
        gen_config.do_sample = false;

        // 创建输入 ID 列表
        std::vector<int> input_ids(gen_config.max_context_length, 128);
        // 生成文本
        pipeline.generate(input_ids, gen_config);
    }

    // chat
}
    {
        // 创建 GenerationConfig 对象
        GenerationConfig gen_config;
        // 设置生成配置中的 do_sample 属性为 false
        gen_config.do_sample = false;
        // 创建包含一个 ChatMessage 对象的向量，其中包含角色为 ROLE_USER，内容为 "你好" 的消息
        std::vector<ChatMessage> messages{{ChatMessage::ROLE_USER, "你好"}};
        // 使用 pipeline 对象处理消息向量，并返回生成的 ChatMessage 对象
        ChatMessage output = pipeline.chat(messages, gen_config);
        // 检查生成的消息内容是否为 "你好，有什么我可以帮助你的吗？"
        EXPECT_EQ(output.content, "你好，有什么我可以帮助你的吗？");
    }
// 运行基准测试的函数，接受模型路径作为参数
static void run_benchmark(const fs::path &model_path) {
    // 检查模型路径是否存在，如果不存在则跳过基准测试
    if (!fs::exists(model_path)) {
        GTEST_SKIP() << "Skipping benchmark test (model " << model_path << " not found)";
    }

    // 初始化时间
    ggml_time_init();
    // 记录开始时间
    int64_t start_ms = ggml_time_ms();
    // 创建 Pipeline 对象，加载模型
    Pipeline pipeline(model_path.string());
    // 计算加载模型所需时间
    int64_t load_model_ms = ggml_time_ms() - start_ms;

    // 准备聊天消息
    start_ms = ggml_time_ms();
    std::vector<ChatMessage> messages{
        {ChatMessage::ROLE_USER, "你好"},
        {ChatMessage::ROLE_ASSISTANT, "你好👋！我是人工智能助手 ChatGLM-6B，很高兴见到你，欢迎问我任何问题。"},
        {ChatMessage::ROLE_USER, "晚上睡不着应该怎么办"},
    };

    // 配置生成器
    GenerationConfig gen_config;
    gen_config.do_sample = false;
    gen_config.num_threads = get_num_threads();

    // 创建性能流对象
    PerfStreamer streamer;
    // 记录开始时间
    start_ms = ggml_time_ms();
    // 进行聊天生成
    pipeline.chat(messages, gen_config, &streamer);
    // 计算生成所需时间
    int64_t gen_s = (ggml_time_ms() - start_ms) / 1000.f;

    // 输出基准测试结果
    std::cout << "======== benchmark results for " << model_path.filename() << " ========\n"
              << "using #threads: " << gen_config.num_threads << "\n"
              << "model loaded within: " << load_model_ms << " ms\n"
              << "generation finished within: " << gen_s << " s\n"
              << streamer.to_string() << "\n"
              << "===========================================================\n";
}

// ChatGLM 基准测试
TEST(Benchmark, ChatGLM) {
    // 设置模型路径
    fs::path model_path = fs::path(__FILE__).parent_path() / "chatglm-ggml.bin";
    // 运行基准测试
    run_benchmark(model_path);
}

// ChatGLM2 基准测试
TEST(Benchmark, ChatGLM2) {
    // 设置模型路径
    fs::path model_path = fs::path(__FILE__).parent_path() / "chatglm2-ggml.bin";
    // 运行基准测试
    run_benchmark(model_path);
}

// Baichuan2_7B 基准测试
TEST(Benchmark, Baichuan2_7B) {
    // 设置模型路径
    fs::path model_path = fs::path(__FILE__).parent_path() / "baichuan2-7b-chat-ggml.bin";
    // 运行基准测试
    run_benchmark(model_path);
}

// Baichuan2_13B 基准测试
TEST(Benchmark, Baichuan2_13B) {
    // 设置模型路径
    fs::path model_path = fs::path(__FILE__).parent_path() / "baichuan2-13b-chat-ggml.bin";
    // 运行基准测试
    run_benchmark(model_path);
}

// InternLM7B 基准测试
    // 定义一个文件系统路径对象，指向当前文件的父路径下的指定文件
    fs::path model_path = fs::path(__FILE__).parent_path() / "internlm-chat-7b-ggml.bin";
    // 运行基准测试，传入模型路径作为参数
    run_benchmark(model_path);
}

# 定义一个测试用例 Benchmark，测试 InternLM20B 模型
TEST(Benchmark, InternLM20B) {
    # 获取当前文件的路径，并拼接上 InternLM20B 模型的文件名
    fs::path model_path = fs::path(__FILE__).parent_path() / "internlm-chat-20b-ggml.bin";
    # 运行基准测试，传入 InternLM20B 模型的路径
    run_benchmark(model_path);
}

# 结束 chatglm 命名空间
} // namespace chatglm
```