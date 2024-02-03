# `ggml\tests\test-backend-ops.cpp`

```cpp
// 初始化张量的数值为均匀分布的随机数，范围默认为 -1.0 到 1.0
static void init_tensor_uniform(ggml_tensor * tensor, float min = -1.0f, float max = 1.0f) {
    // 获取张量的元素数量
    size_t size = ggml_nelements(tensor);
    // 创建一个大小为 size 的浮点数向量
    std::vector<float> data(size);

    // 如果条件为假，则执行以下代码块
#if 0
    // 创建一个固定种子的默认随机数生成器
    static std::default_random_engine generator(1234);
    // 创建一个指定范围的均匀分布
    std::uniform_real_distribution<float> distribution(min, max);

    // 遍历张量的每个元素，赋值为随机数
    for (size_t i = 0; i < size; i++) {
        data[i] = distribution(generator);
    }
// 如果条件为真，则执行以下代码块
#else
    // 创建一个 lambda 函数，用于在指定范围内生成均匀分布的随机数
    auto init_thread = [&](size_t start, size_t end) {
        // 创建一个随机设备
        std::random_device rd;
        // 创建一个随机数生成器，使用随机设备生成种子
        std::default_random_engine generator(rd());
        // 创建一个指定范围的均匀分布
        std::uniform_real_distribution<float> distribution(min, max);

        // 遍历指定范围内的元素，赋值为随机数
        for (size_t i = start; i < end; i++) {
            data[i] = distribution(generator);
        }
    };

    // 获取系统支持的线程数
    size_t n_threads = std::thread::hardware_concurrency();
    // 创建一个线程向量
    std::vector<std::thread> threads;
    threads.reserve(n_threads);
    // 根据线程数将元素范围分配给不同的线程，并启动线程
    for (size_t i = 0; i < n_threads; i++) {
        size_t start =     i*size/n_threads;
        size_t end   = (i+1)*size/n_threads;
        threads.emplace_back(init_thread, start, end);
    }
    // 等待所有线程执行完毕
    for (auto & t : threads) {
        t.join();
    }
#endif

    // 如果张量的类型为 GGML_TYPE_F32 或 GGML_TYPE_I32，则将数据设置到张量中
    if (tensor->type == GGML_TYPE_F32 || tensor->type == GGML_TYPE_I32) {
        ggml_backend_tensor_set(tensor, data.data(), 0, size * sizeof(float));
    } 
    // 如果张量的类型为量化类型或 GGML_TYPE_F16，则进行相应的处理
    else if (ggml_is_quantized(tensor->type) || tensor->type == GGML_TYPE_F16) {
        // 断言张量的大小能够被块大小整除
        GGML_ASSERT(size % ggml_blck_size(tensor->type) == 0);
        // 创建一个存储量化数据的字节向量
        std::vector<uint8_t> dataq(ggml_row_size(tensor->type, size));
        // 创建一个长度为 16 的整型数组
        int64_t hist[16];
        // 对数据进行量化处理，并设置到张量中
        ggml_quantize_chunk(tensor->type, data.data(), dataq.data(), 0, size, hist);
        ggml_backend_tensor_set(tensor, dataq.data(), 0, dataq.size());
    }
}
    } else if (tensor->type == GGML_TYPE_I8 || tensor->type == GGML_TYPE_I16 || tensor->type == GGML_TYPE_I32) {
        // 如果张量的类型是8位整数、16位整数或32位整数，则执行以下操作
        // 这将创建一些奇怪的整数
        ggml_backend_tensor_set(tensor, data.data(), 0, ggml_nbytes(tensor));
    } else {
        // 如果张量的类型不是上述类型，则触发断言错误
        GGML_ASSERT(false);
    }
// 将 ggml_tensor 转换为 float 类型的向量
static std::vector<float> tensor_to_float(const ggml_tensor * t) {
    // 创建一个空的 float 向量 tv，并预留 ggml_tensor 中元素的空间
    std::vector<float> tv;
    tv.reserve(ggml_nelements(t));

    // 创建一个 uint8_t 类型的缓冲区 buf，大小为 ggml_tensor 的字节数
    std::vector<uint8_t> buf(ggml_nbytes(t));
    // 从 ggml_tensor 中获取数据，存储到 buf 中
    ggml_backend_tensor_get(t, buf.data(), 0, ggml_nbytes(t));

    // 获取 ggml_tensor 的类型特性
    ggml_type_traits_t tt = ggml_internal_get_type_traits(t->type);
    // 获取 ggml_tensor 的块大小
    size_t bs = ggml_blck_size(t->type);
    // 创建一个 float 类型的向量 vq，大小为 ggml_tensor 的块大小
    std::vector<float> vq(ggml_blck_size(t->type));
    // 检查 ggml_tensor 是否是量化的
    bool quantized = ggml_is_quantized(t->type);

    // 通过索引访问元素，避免视图中的间隙
    for (int64_t i3 = 0; i3 < t->ne[3]; i3++) {
        for (int64_t i2 = 0; i2 < t->ne[2]; i2++) {
            for (int64_t i1 = 0; i1 < t->ne[1]; i1++) {
                for (int64_t i0 = 0; i0 < t->ne[0]; i0 += bs) {
                    // 计算元素在缓冲区中的索引
                    size_t i = i3*t->nb[3] + i2*t->nb[2] + i1*t->nb[1] + i0/bs*t->nb[0];
                    // 根据 ggml_tensor 的类型，将数据转换为 float 类型，并存储到 tv 中
                    if (t->type == GGML_TYPE_F16) {
                        tv.push_back(ggml_fp16_to_fp32(*(ggml_fp16_t*)&buf[i]));
                    } else if (t->type == GGML_TYPE_F32) {
                        tv.push_back(*(float *) &buf[i]);
                    } else if (t->type == GGML_TYPE_I32) {
                        tv.push_back((float)*(int32_t *) &buf[i]);
                    } else if (t->type == GGML_TYPE_I16) {
                        tv.push_back((float)*(int16_t *) &buf[i]);
                    } else if (t->type == GGML_TYPE_I8) {
                        tv.push_back((float)*(int8_t *) &buf[i]);
                    } else if (quantized) {
                        // 如果是量化的，将数据转换为 float 类型，并插入到 tv 中
                        std::vector<float> vq(ggml_blck_size(t->type));
                        tt.to_float(&buf[i], vq.data(), ggml_blck_size(t->type));
                        tv.insert(tv.end(), vq.begin(), vq.end());
                    } else {
                        // 如果类型不匹配，抛出异常
                        GGML_ASSERT(false);
                    }
                }
            }
        }
    }

    // 返回转换后的 float 向量
    return tv;
}
    // 初始化变量 mag2 为 0.0
    double mag2 = 0.0;

    // 遍历数组 v1 和 v2，计算它们的点积和模长
    for (size_t i = 0; i < n; i++) {
        // 如果 v1 或 v2 中有任何一个元素为 NaN，则返回 -1.0f
        if (std::isnan(v1[i]) || std::isnan(v2[i])) {
            return -1.0f;
        }
        // 如果 v1 和 v2 中有任何一个元素为无穷大，则继续下一次循环
        if (std::isinf(v1[i]) && std::isinf(v2[i])) {
            continue;
        }
        // 计算点积
        dot  += v1[i]*v2[i];
        // 计算 v1 的模长的平方
        mag1 += v1[i]*v1[i];
        // 计算 v2 的模长的平方
        mag2 += v2[i]*v2[i];
    }

    // 返回点积除以两个向量模长的乘积的平方根
    return dot/sqrt(mag1*mag2);
// 计算两个向量之间的欧几里德距离
static float distance(const float * v1, const float * v2, size_t n) {
    double d = 0.0;

    for (size_t i = 0; i < n; i++) {
        // 如果向量中存在 NaN，则返回无穷大
        if (std::isnan(v1[i]) || std::isnan(v2[i])) {
            return INFINITY;
        }
        // 如果向量中存在正负无穷大，则继续下一次循环
        if (std::isinf(v1[i]) && std::isinf(v2[i])) {
            continue;
        }
        // 计算欧几里德距离的平方
        d += (v1[i] - v2[i])*(v1[i] - v2[i]);
    }

    return sqrt(d); // 返回欧几里德距离
}

// 计算向量的长度
static float vec_len(const float * v, size_t n) {
    double d = 0.0;

    for (size_t i = 0; i < n; i++) {
        // 如果向量中存在 NaN，则返回无穷大
        if (std::isnan(v[i])) {
            return INFINITY;
        }
        // 如果向量中存在正负无穷大，则继续下一次循环
        if (std::isinf(v[i])) {
            continue;
        }
        // 计算向量长度的平方
        d += v[i]*v[i];
    }

    return sqrt(d); // 返回向量长度
}

// 计算标准化均方误差 = mse(a, b) / mse(a, 0)
static double nmse(const float * a, const float * b, size_t n) {
    double mse_a_b = 0.0;
    double mse_a_0 = 0.0;

    for (size_t i = 0; i < n; i++) {
        float a_i = a[i];
        float b_i = b[i];

        mse_a_b += (a_i - b_i) * (a_i - b_i);
        mse_a_0 += a_i * a_i;
    }

    return mse_a_b / mse_a_0; // 返回标准化均方误差
}

// 用于打印测试用例的变量的工具函数
#define VAR_TO_STR(x) (#x "=" + var_to_str(x))

// 将变量转换为字符串
template<typename T>
static std::string var_to_str(const T & x) {
    return std::to_string(x);
}

// 将数组转换为字符串
template<typename T, size_t N>
static std::string var_to_str(const T (&x)[N]) {
    std::string s = "[";
    for (size_t i = 0; i < N; i++) {
        if (i > 0) {
            s += ",";
        }
        s += var_to_str(x[i]);
    }
    s += "]";
    return s;
}

// 将 std::array 转换为字符串
template<typename T, size_t N>
static std::string var_to_str(const std::array<T, N> & x) {
    std::string s = "[";
    for (size_t i = 0; i < N; i++) {
        if (i > 0) {
            s += ",";
        }
        s += var_to_str(x[i]);
    }
    s += "]";
    return s;
}

// 将 ggml_type 转换为字符串
static std::string var_to_str(ggml_type type) {
    return ggml_type_name(type);
}
// 将单个变量转换为字符串
#define VARS_TO_STR1(a) VAR_TO_STR(a)
// 将两个变量转换为字符串并用逗号分隔
#define VARS_TO_STR2(a, b) VAR_TO_STR(a) + "," + VAR_TO_STR(b)
// 将三个变量转换为字符串并用逗号分隔
#define VARS_TO_STR3(a, b, c) VAR_TO_STR(a) + "," + VARS_TO_STR2(b, c)
// 将四个变量转换为字符串并用逗号分隔
#define VARS_TO_STR4(a, b, c, d) VAR_TO_STR(a) + "," + VARS_TO_STR3(b, c, d)
// 将五个变量转换为字符串并用逗号分隔
#define VARS_TO_STR5(a, b, c, d, e) VAR_TO_STR(a) + "," + VARS_TO_STR4(b, c, d, e)
// 将六个变量转换为字符串并用逗号分隔
#define VARS_TO_STR6(a, b, c, d, e, f) VAR_TO_STR(a) + "," + VARS_TO_STR5(b, c, d, e, f)
// 将七个变量转换为字符串并用逗号分隔
#define VARS_TO_STR7(a, b, c, d, e, f, g) VAR_TO_STR(a) + "," + VARS_TO_STR6(b, c, d, e, f, g)
// 将八个变量转换为字符串并用逗号分隔
#define VARS_TO_STR8(a, b, c, d, e, f, g, h) VAR_TO_STR(a) + "," + VARS_TO_STR7(b, c, d, e, f, g, h)
// 将九个变量转换为字符串并用逗号分隔
#define VARS_TO_STR9(a, b, c, d, e, f, g, h, i) VAR_TO_STR(a) + "," + VARS_TO_STR8(b, c, d, e, f, g, h, i)
// 将十个变量转换为字符串并用逗号分隔
#define VARS_TO_STR10(a, b, c, d, e, f, g, h, i, j) VAR_TO_STR(a) + "," + VARS_TO_STR9(b, c, d, e, f, g, h, i, j)
// 将十一个变量转换为字符串并用逗号分隔
#define VARS_TO_STR11(a, b, c, d, e, f, g, h, i, j, k) VAR_TO_STR(a) + "," + VARS_TO_STR10(b, c, d, e, f, g, h, i, j, k)

// 判断浮点数是否为无穷大或最大值
static bool isinf_or_max(float f) {
    return std::isinf(f) || f == FLT_MAX || f == -FLT_MAX;
}

// 判断操作是否为视图操作
static bool ggml_is_view_op(enum ggml_op op) {
    return op == GGML_OP_VIEW || op == GGML_OP_RESHAPE || op == GGML_OP_PERMUTE || op == GGML_OP_TRANSPOSE;
}

// 测试模式枚举
enum test_mode {
    MODE_TEST,
    MODE_PERF,
};

// 测试用例结构体
struct test_case {
    virtual ~test_case() {}

    // 获取操作描述
    virtual std::string op_desc(ggml_tensor * t) {
        return ggml_op_desc(t);
    }

    // 获取变量
    virtual std::string vars() {
        return "";
    }

    // 构建图形
    virtual ggml_tensor * build_graph(ggml_context * ctx) = 0;

    // 获取最大 NMSE 误差
    virtual double max_nmse_err() {
        return 1e-7;
    }

    // 初始化张量
    virtual void initialize_tensors(ggml_context * ctx) {
        for (ggml_tensor * t = ggml_get_first_tensor(ctx); t != nullptr; t = ggml_get_next_tensor(ctx, t)) {
            init_tensor_uniform(t);
        }
    }
    // 计算张量及其源张量的总大小
    virtual size_t op_size(ggml_tensor * t) {
        size_t size = ggml_nbytes(t);
        // 添加源张量的大小
        for (int i = 0; i < GGML_MAX_SRC; i++) {
            if (t->src[i] != NULL) {
                size += ggml_nbytes(t->src[i]);
            }
        }
        return size;
    }

    // 初始化指针 gf 为 nullptr
    ggml_cgraph * gf = nullptr;

    // 定义常量 sentinel_size 为 1024
    static const int sentinel_size = 1024;

    // 定义枚举类型 test_mode
    test_mode mode;

    // 定义存储 ggml_tensor 指针的向量 sentinels
    std::vector<ggml_tensor *> sentinels;

    // 向 sentinels 向量中添加一个新的 sentinel
    void add_sentinel(ggml_context * ctx) {
        // 如果 mode 为 MODE_PERF，则直接返回
        if (mode == MODE_PERF) {
            return;
        }
        // 创建一个大小为 sentinel_size 的新的 sentinel，并添加到 sentinels 向量中
        ggml_tensor * sentinel = ::ggml_new_tensor_1d(ctx, GGML_TYPE_F32, sentinel_size);
        ggml_format_name(sentinel, "sent_%zu", sentinels.size());
        sentinels.push_back(sentinel);
    }

    // 重载 ggml_new_tensor 函数，在创建新张量后添加一个 sentinel
    ggml_tensor * ggml_new_tensor(ggml_context * ctx, ggml_type type, int n_dims, const int64_t * ne) {
        ggml_tensor * t = ::ggml_new_tensor(ctx, type, n_dims, ne);
        add_sentinel(ctx);
        return t;
    }

    // 重载 ggml_new_tensor_1d 函数，在创建新一维张量后添加一个 sentinel
    ggml_tensor * ggml_new_tensor_1d(ggml_context * ctx, ggml_type type, int64_t ne0) {
        ggml_tensor * t = ::ggml_new_tensor_1d(ctx, type, ne0);
        add_sentinel(ctx);
        return t;
    }

    // 重载 ggml_new_tensor_2d 函数，在创建新二维张量后添加一个 sentinel
    ggml_tensor * ggml_new_tensor_2d(ggml_context * ctx, ggml_type type, int64_t ne0, int64_t ne1) {
        ggml_tensor * t = ::ggml_new_tensor_2d(ctx, type, ne0, ne1);
        add_sentinel(ctx);
        return t;
    }

    // 重载 ggml_new_tensor_3d 函数，在创建新三维张量后添加一个 sentinel
    ggml_tensor * ggml_new_tensor_3d(ggml_context * ctx, ggml_type type, int64_t ne0, int64_t ne1, int64_t ne2) {
        ggml_tensor * t = ::ggml_new_tensor_3d(ctx, type, ne0, ne1, ne2);
        add_sentinel(ctx);
        return t;
    }

    // 重载 ggml_new_tensor_4d 函数，在创建新四维张量后添加一个 sentinel
    ggml_tensor * ggml_new_tensor_4d(ggml_context * ctx, ggml_type type, int64_t ne0, int64_t ne1, int64_t ne2, int64_t ne3) {
        ggml_tensor * t = ::ggml_new_tensor_4d(ctx, type, ne0, ne1, ne2, ne3);
        add_sentinel(ctx);
        return t;
    }
    # 结束第三个代码块
    }
    # 结束第二个代码块
    }
    # 结束第一个代码块
// 结构体 test_unary 继承自 test_case，用于测试一元操作
struct test_unary : public test_case {
    // 一元操作符
    const ggml_unary_op op;
    // 数据类型
    const ggml_type type;
    // 数组维度
    const std::array<int64_t, 4> ne;

    // 重写父类方法，返回变量的字符串表示
    std::string vars() override {
        return VARS_TO_STR2(type, ne);
    }

    // 构造函数，初始化一元操作符、数据类型和数组维度
    test_unary(ggml_unary_op op,
            ggml_type type = GGML_TYPE_F32,
            std::array<int64_t, 4> ne = {128, 10, 10, 10})
        : op(op), type(type), ne(ne) {}

    // 构建计算图
    ggml_tensor * build_graph(ggml_context * ctx) override {
        // 创建输入张量
        ggml_tensor * in = ggml_new_tensor(ctx, type, 4, ne.data());
        // 执行一元操作
        ggml_tensor * out = ggml_unary(ctx, in, op);
        // 返回输出张量
        return out;
    }
};

// 结构体 test_get_rows 继承自 test_case，用于测试获取行操作
struct test_get_rows : public test_case {
    // 数据类型
    const ggml_type type;
    // 列数
    const int n;
    // 行数
    const int m;
    // 要获取的行数
    const int r;
    // 批处理大小
    const int b;
    // 是否视图（非连续 src1）
    const bool v;

    // 重写父类方法，返回变量的字符串表示
    std::string vars() override {
        return VARS_TO_STR6(type, n, m, r, b, v);
    }

    // 构造函数，初始化数据类型、列数、行数、要获取的行数、批处理大小和是否视图
    test_get_rows(ggml_type type = GGML_TYPE_F32, int n = 10, int m = 5, int r = 3, int b = 1, bool v = false)
        : type(type), n(n), m(m), r(r), b(b), v(v) {}

    // 构建计算图
    ggml_tensor * build_graph(ggml_context * ctx) override {
        // 创建输入张量
        ggml_tensor * in = ggml_new_tensor_3d(ctx, type, n, m, b);
        // 创建包含要获取的行数的张量
        ggml_tensor * rows = ggml_new_tensor_2d(ctx, GGML_TYPE_I32, r, b);
        // 如果是视图，则调用视图函数
        if (v) {
            rows = ggml_view_2d(ctx, rows, r/2, b, rows->nb[1], 0);
        }
        // 获取指定行数的张量
        ggml_tensor * out = ggml_get_rows(ctx, in, rows);
        // 返回输出张量
        return out;
    }
    // 重写初始化张量的函数
    void initialize_tensors(ggml_context * ctx) override {
        // 遍历上下文中的张量
        for (ggml_tensor * t = ggml_get_first_tensor(ctx); t != NULL; t = ggml_get_next_tensor(ctx, t)) {
            // 如果张量类型为 GGML_TYPE_I32
            if (t->type == GGML_TYPE_I32) {
                // 如果张量是视图操作，跳过当前循环
                if (ggml_is_view_op(t->op)) { continue; }
                // 为整型张量分配内存空间
                // rows * b 表示数据的长度
                std::vector<int> data(r*b);
                // 为数据填充随机值
                for (int i = 0; i < r*b; i++) {
                    data[i] = rand() % m;
                }
                // 将数据设置到张量中
                ggml_backend_tensor_set(t, data.data(), 0, r * b * sizeof(int));
            } else {
                // 对于其他类型的张量，使用 init_tensor_uniform 函数进行初始化
                init_tensor_uniform(t);
            }
        }
    }
// 结构体 test_repeat，继承自 test_case
struct test_repeat : public test_case {
    // 常量成员变量 type，表示 ggml_type 类型
    const ggml_type type;
    // 常量成员变量 ne，表示包含4个int64_t类型元素的数组
    const std::array<int64_t, 4> ne;
    // 常量成员变量 nr，表示包含4个int类型元素的数组
    const std::array<int, 4> nr;

    // 重写父类的虚函数 vars，返回类型为字符串
    std::string vars() override {
        return VARS_TO_STR3(type, ne, nr);
    }

    // 重写父类的虚函数 op_size，返回类型为 size_t
    size_t op_size(ggml_tensor * t) override {
        return ggml_nbytes(t) * 2;
    }

    // 构造函数，初始化 type、ne、nr
    test_repeat(ggml_type type = GGML_TYPE_F32,
            std::array<int64_t, 4> ne = {10, 10, 10, 10},
            std::array<int, 4> nr = {2, 2, 2, 2})
        : type(type), ne(ne), nr(nr) {}

    // 重写父类的虚函数 build_graph，返回类型为 ggml_tensor 指针
    ggml_tensor * build_graph(ggml_context * ctx) override {
        // 创建一个4维的新张量 target
        ggml_tensor * target = ggml_new_tensor_4d(ctx, type, ne[0]*nr[0], ne[1]*nr[1], ne[2]*nr[2], ne[3]*nr[3]);
        // 创建一个新张量 src
        ggml_tensor * src = ggml_new_tensor(ctx, type, 4, ne.data());
        // 重复 src 到 target，返回结果张量 out
        ggml_tensor * out = ggml_repeat(ctx, src, target);
        return out;
    }
};

// 结构体 test_dup，继承自 test_case
struct test_dup : public test_case {
    // 常量成员变量 type，表示 ggml_type 类型
    const ggml_type type;
    // 常量成员变量 ne，表示包含4个int64_t类型元素的数组
    const std::array<int64_t, 4> ne;
    // 常量成员变量 permute，表示包含4个int64_t类型元素的数组
    const std::array<int64_t, 4> permute;
    // 布尔成员变量 _use_permute
    bool _use_permute;

    // 重写父类的虚函数 vars，返回类型为字符串
    std::string vars() override {
        std::string v = VARS_TO_STR2(type, ne);
        if (_use_permute) v += "," + VAR_TO_STR(permute);
        return v;
    }

    // 构造函数，初始化 type、ne、permute、_use_permute
    test_dup(ggml_type type = GGML_TYPE_F32,
            std::array<int64_t, 4> ne = {10, 10, 10, 1},
            std::array<int64_t, 4> permute = {0, 0, 0, 0})
        : type(type), ne(ne), permute(permute),
            _use_permute(permute[0] + permute[1] + permute[2] + permute[3] > 0) {}

    // 重写父类的虚函数 build_graph，返回类型为 ggml_tensor 指针
    ggml_tensor * build_graph(ggml_context * ctx) override {
        // 创建一个新张量 src
        ggml_tensor * src = ggml_new_tensor(ctx, type, 4, ne.data());
        // 如果 _use_permute 为真，则对 src 进行排列操作
        if (_use_permute) {
            src = ggml_permute(ctx, src, permute[0], permute[1], permute[2], permute[3]);
        }
        // 复制 src，返回结果张量 out
        ggml_tensor * out = ggml_dup(ctx, src);
        return out;
    }
};

// 结构体 test_cpy，继承自 test_case
struct test_cpy : public test_case {
    // 常量成员变量 type_src，表示 ggml_type 类型
    const ggml_type type_src;
    // 常量成员变量 type_dst，表示 ggml_type 类型
    const ggml_type type_dst;
    // 常量成员变量 ne，表示包含4个int64_t类型元素的数组
    const std::array<int64_t, 4> ne;
    # 重写父类的 vars 方法，返回包含 type_src、type_dst 和 ne 的字符串
    std::string vars() override {
        return VARS_TO_STR3(type_src, type_dst, ne);
    }

    # 重写父类的 op_size 方法，返回输入张量 t 和其源张量的总字节数
    size_t op_size(ggml_tensor * t) override {
        return ggml_nbytes(t) + ggml_nbytes(t->src[0]);
    }

    # 定义 test_cpy 构造函数，初始化 type_src、type_dst 和 ne
    test_cpy(ggml_type type_src = GGML_TYPE_F32, ggml_type type_dst = GGML_TYPE_F32,
            std::array<int64_t, 4> ne = {10, 10, 10, 1})
        : type_src(type_src), type_dst(type_dst), ne(ne) {}

    # 重写父类的 build_graph 方法，构建图并返回输出张量
    ggml_tensor * build_graph(ggml_context * ctx) override {
        # 创建输入张量 src 和目标张量 dst
        ggml_tensor * src = ggml_new_tensor(ctx, type_src, 4, ne.data());
        ggml_tensor * dst = ggml_new_tensor(ctx, type_dst, 4, ne.data());
        # 复制 src 到 dst，并返回输出张量
        ggml_tensor * out = ggml_cpy(ctx, src, dst);
        return out;
    }
// 结构体 test_cont 继承自 test_case，表示一个连续操作的测试用例
struct test_cont : public test_case {
    // 常量成员变量，表示 ggml_type 类型和包含四个 int64_t 元素的数组
    const ggml_type type;
    const std::array<int64_t, 4> ne;

    // 重写父类的虚函数，返回 type 和 ne 的字符串表示
    std::string vars() override {
        return VARS_TO_STR2(type, ne);
    }

    // 构造函数，初始化 type 和 ne
    test_cont(ggml_type type = GGML_TYPE_F32, std::array<int64_t, 4> ne = {10, 10, 10, 1})
        : type(type), ne(ne) {}

    // 重写父类的虚函数，构建图并返回输出的 ggml_tensor 指针
    ggml_tensor * build_graph(ggml_context * ctx) override {
        // 创建一个新的 tensor 对象 src
        ggml_tensor * src = ggml_new_tensor(ctx, type, 4, ne.data());
        // 对 src 进行转置操作
        src = ggml_transpose(ctx, src);
        // 对转置后的 src 进行连续操作
        ggml_tensor * out = ggml_cont(ctx, src);

        return out;
    }
};

// 结构体 test_bin_bcast 继承自 test_case，表示一个二元广播操作的测试用例
struct test_bin_bcast : public test_case {
    // 定义 op_t 类型的别名 op_t，表示一个函数指针类型
    using op_t = ggml_tensor * (*) (ggml_context *, ggml_tensor *, ggml_tensor *);
    // op 是函数指针类型 op_t 的变量
    op_t op;
    // 常量成员变量，表示 ggml_type 类型和包含四个 int64_t 元素的数组，以及包含四个 int 元素的数组
    const ggml_type type;
    const std::array<int64_t, 4> ne;
    const std::array<int, 4> nr;

    // 重写父类的虚函数，返回 type、ne 和 nr 的字符串表示
    std::string vars() override {
        return VARS_TO_STR3(type, ne, nr);
    }

    // 重写父类的虚函数，返回 tensor 对象 t 的大小
    size_t op_size(ggml_tensor * t) override {
        return ggml_nbytes(t) * 3;
    }

    // 构造函数，初始化 op、type、ne 和 nr
    test_bin_bcast(op_t op, ggml_type type = GGML_TYPE_F32, std::array<int64_t, 4> ne = {10, 10, 1, 1}, std::array<int, 4> nr = {1, 2, 1, 1})
        : op(op), type(type), ne(ne), nr(nr) {}

    // 重写父类的虚函数，构建图并返回输出的 ggml_tensor 指针
    ggml_tensor * build_graph(ggml_context * ctx) override {
        // 创建一个新的四维 tensor 对象 a
        ggml_tensor * a = ggml_new_tensor_4d(ctx, type, ne[0]*nr[0], ne[1]*nr[1], ne[2]*nr[2], ne[3]*nr[3]);
        // 创建一个新的 tensor 对象 b
        ggml_tensor * b = ggml_new_tensor(ctx, type, 4, ne.data());
        // 对 a 和 b 进行 op 操作
        ggml_tensor * out = op(ctx, a, b);
        return out;
    }

    // 重写父类的虚函数，初始化 tensor 对象
    void initialize_tensors(ggml_context * ctx) override {
        for (ggml_tensor * t = ggml_get_first_tensor(ctx); t != NULL; t = ggml_get_next_tensor(ctx, t)) {
            if (op == ggml_div) {
                // 避免除以零
                init_tensor_uniform(t, 1.0f, 2.0f);
            } else {
                init_tensor_uniform(t);
            }
        }
    }
};
// 定义一个结构体 test_scale，继承自 test_case
struct test_scale : public test_case {
    // 声明常量成员变量 type，表示 ggml_type 类型
    const ggml_type type;
    // 声明常量成员变量 ne，表示包含4个 int64_t 元素的数组
    const std::array<int64_t, 4> ne;
    // 声明浮点型成员变量 scale
    float scale;

    // 重写父类的虚函数 vars，返回包含 type、ne、scale 的字符串
    std::string vars() override {
        return VARS_TO_STR3(type, ne, scale);
    }

    // 构造函数，初始化 type、ne、scale
    test_scale(ggml_type type = GGML_TYPE_F32,
            std::array<int64_t, 4> ne = {10, 10, 10, 10},
            float scale = 2.0f)
        : type(type), ne(ne), scale(scale) {}

    // 重写父类的虚函数 build_graph，构建图并返回 ggml_tensor 指针
    ggml_tensor * build_graph(ggml_context * ctx) override {
        // 创建一个新的 tensor 对象 a
        ggml_tensor * a = ggml_new_tensor(ctx, type, 4, ne.data());
        // 对 tensor 对象 a 进行缩放操作，返回结果 out
        ggml_tensor * out = ggml_scale(ctx, a, scale);
        // 返回结果 out
        return out;
    }
};

// 定义结构体 test_norm，继承自 test_case
// ... (后续代码类似，按照相同的格式添加注释)
    // 常量 k，表示矩阵乘法中的中间维度
    const int64_t k;
    // 常量 bs，表示矩阵 A 和 B 的维度 3 和 4
    const std::array<int64_t, 2> bs; // dims 3 and 4
    // 常量 nr，表示在维度 3 和 4 中的重复次数
    const std::array<int64_t, 2> nr; // repeat in dims 3 and 4

    // 返回变量的字符串表示，包括 type_a, type_b, m, n, k, bs, nr
    std::string vars() override {
        return VARS_TO_STR7(type_a, type_b, m, n, k, bs, nr);
    }

    // 返回最大的均方误差，用于测试
    double max_nmse_err() override {
        return 5e-4;
    }

    // 计算操作的大小，包括输入张量的字节数和重复次数
    size_t op_size(ggml_tensor * t) override {
        size_t a = ggml_nbytes(t->src[0]) * n * nr[0] * nr[1];
        size_t b = ggml_nbytes(t->src[1]) * m;
        size_t c  = ggml_nbytes(t);
        return a + b + c;

        GGML_UNUSED(t);
    }

    // 构造函数，初始化矩阵乘法的参数
    test_mul_mat(ggml_type type_a = GGML_TYPE_F32, ggml_type type_b = GGML_TYPE_F32,
            int64_t m = 32, int64_t n = 32, int64_t k = 32,
            std::array<int64_t, 2> bs = {10, 10},
            std::array<int64_t, 2> nr = {2, 2})
        : type_a(type_a), type_b(type_b), m(m), n(n), k(k), bs(bs), nr(nr) {}

    // 构建计算图，包括创建输入张量和进行矩阵乘法操作
    ggml_tensor * build_graph(ggml_context * ctx) override {
        // C^T = A * B^T: (k, m) * (k, n) => (m, n)
        ggml_tensor * a = ggml_new_tensor_4d(ctx, type_a, k, m, bs[0]      , bs[1]);
        ggml_tensor * b = ggml_new_tensor_4d(ctx, type_b, k, n, bs[0]*nr[0], bs[1]*nr[1]);
        ggml_tensor * out = ggml_mul_mat(ctx, a, b);
        return out;
    }
// 结构体 test_mul_mat_id 继承自 test_case，用于测试矩阵乘法
struct test_mul_mat_id : public test_case {
    // 定义成员变量
    const ggml_type type_a;  // 第一个矩阵的数据类型
    const ggml_type type_b;  // 第二个矩阵的数据类型
    const int n_mats;  // 矩阵数量
    const int id;  // 矩阵 id
    const int64_t m;  // 矩阵行数
    const int64_t n;  // 矩阵列数
    const int64_t k;  // 矩阵维度
    const bool v;  // 是否为视图（非连续的 id）

    // 重写父类方法，返回成员变量的字符串表示
    std::string vars() override {
        return VARS_TO_STR8(type_a, type_b, n_mats, id, m, n, k, v);
    }

    // 重写父类方法，返回最大均方误差
    double max_nmse_err() override {
        return 5e-4;
    }

    // 重写父类方法，计算操作所需的内存大小
    size_t op_size(ggml_tensor * t) override {
        size_t a = ggml_nbytes(t->src[2]) * n;  // 计算第一个矩阵的内存大小
        size_t b = ggml_nbytes(t->src[1]) * m;  // 计算第二个矩阵的内存大小
        size_t c  = ggml_nbytes(t);  // 计算输出矩阵的内存大小
        return a + b + c;  // 返回总内存大小

        GGML_UNUSED(t);  // 标记 t 未使用
    }

    // 构造函数，初始化成员变量
    test_mul_mat_id(ggml_type type_a = GGML_TYPE_F32, ggml_type type_b = GGML_TYPE_F32,
            int n_mats = 2, int id = 0,
            int64_t m = 32, int64_t n = 32, int64_t k = 32, bool v = false)
        : type_a(type_a), type_b(type_b), n_mats(n_mats), id(id),
            m(m), n(n), k(k), v(v) {}

    // 重写父类方法，构建计算图
    ggml_tensor * build_graph(ggml_context * ctx) override {
        // 创建矩阵数组
        std::vector<ggml_tensor *> mats;
        for (int i = 0; i < n_mats; i++) {
            ggml_tensor * a = ggml_new_tensor_2d(ctx, type_a, k, m);  // 创建第一个矩阵
            mats.push_back(a);  // 将矩阵添加到数组中
        }
        ggml_tensor * ids = ggml_new_tensor_2d(ctx, GGML_TYPE_I32, n_mats, n);  // 创建 id 矩阵
        if (v) {
            ids = ggml_view_2d(ctx, ids, n_mats/2, ids->ne[1], ids->nb[1], 0);  // 如果是视图，则进行视图操作
        }
        ggml_tensor * b = ggml_new_tensor_2d(ctx, type_b, k, n);  // 创建第二个矩阵
        ggml_tensor * out = ggml_mul_mat_id(ctx, mats.data(), n_mats, ids, v ? id/2 : id, b);  // 执行矩阵乘法操作
        return out;  // 返回输出矩阵
    }
    // 重写父类的初始化张量方法
    void initialize_tensors(ggml_context * ctx) override {
        // 生成随机设备
        std::random_device rd;
        // 创建默认的随机数生成引擎
        std::default_random_engine rng(rd());
        // 遍历上下文中的张量
        for (ggml_tensor * t = ggml_get_first_tensor(ctx); t != NULL; t = ggml_get_next_tensor(ctx, t)) {
            // 如果张量类型为 GGML_TYPE_I32
            if (t->type == GGML_TYPE_I32) {
                // 如果是视图操作，跳过当前张量
                if (ggml_is_view_op(t->op)) { continue; }
                // 对于每一行
                for (int64_t r = 0; r < ggml_nrows(t); r++) {
                    // 创建一个包含 t->ne[0] 个元素的 int32_t 类型的向量
                    std::vector<int32_t> data(t->ne[0]);
                    // 对向量中的每个元素进行初始化
                    for (int i = 0; i < t->ne[0]; i++) {
                        data[i] = i % n_mats;
                    }
                    // 对向量中的元素进行随机重排
                    std::shuffle(data.begin(), data.end(), rng);
                    // 设置张量的数据
                    ggml_backend_tensor_set(t, data.data(), r * t->nb[1], t->ne[0] * sizeof(int32_t));
                }
            } else {
                // 对于其他类型的张量，使用均匀分布初始化
                init_tensor_uniform(t);
            }
        }
    }
};

// 定义一个结构体 test_sqr，继承自 test_case
struct test_sqr : public test_case {
    // 声明常量成员变量 type 和 ne
    const ggml_type type;
    const std::array<int64_t, 4> ne;

    // 重写父类的虚函数 vars，返回 type 和 ne 的字符串表示
    std::string vars() override {
        return VARS_TO_STR2(type, ne);
    }

    // 构造函数，初始化 type 和 ne
    test_sqr(ggml_type type = GGML_TYPE_F32,
            std::array<int64_t, 4> ne = {10, 10, 10, 10})
        : type(type), ne(ne) {}

    // 重写父类的虚函数 build_graph，构建并返回一个 ggml_tensor 对象
    ggml_tensor * build_graph(ggml_context * ctx) override {
        ggml_tensor * a = ggml_new_tensor(ctx, type, 4, ne.data());
        ggml_tensor * out = ggml_sqr(ctx, a);
        return out;
    }
};

// 定义一个结构体 test_clamp，继承自 test_case
struct test_clamp : public test_case {
    // 声明常量成员变量 type、ne、min 和 max
    const ggml_type type;
    const std::array<int64_t, 4> ne;
    float min;
    float max;

    // 重写父类的虚函数 vars，返回 type、ne、min 和 max 的字符串表示
    std::string vars() override {
        return VARS_TO_STR4(type, ne, min, max);
    }

    // 构造函数，初始化 type、ne、min 和 max
    test_clamp(ggml_type type = GGML_TYPE_F32,
            std::array<int64_t, 4> ne = {10, 10, 10, 10},
            float min = -0.5f, float max = 0.5f)
        : type(type), ne(ne), min(min), max(max) {}

    // 重写父类的虚函数 build_graph，构建并返回一个 ggml_tensor 对象
    ggml_tensor * build_graph(ggml_context * ctx) override {
        ggml_tensor * a = ggml_new_tensor(ctx, type, 4, ne.data());
        ggml_tensor * out = ggml_clamp(ctx, a, min, max);
        return out;
    }
};

// 定义一个结构体 test_diag_mask_inf，继承自 test_case
struct test_diag_mask_inf : public test_case {
    // 声明常量成员变量 type、ne 和 n_past
    const ggml_type type;
    const std::array<int64_t, 4> ne;
    const int n_past;

    // 重写父类的虚函数 vars，返回 type、ne 和 n_past 的字符串表示
    std::string vars() override {
        return VARS_TO_STR3(type, ne, n_past);
    }

    // 构造函数，初始化 type、ne 和 n_past
    test_diag_mask_inf(ggml_type type = GGML_TYPE_F32,
            std::array<int64_t, 4> ne = {10, 10, 10, 10},
            int n_past = 5)
        : type(type), ne(ne), n_past(n_past) {}

    // 重写父类的虚函数 build_graph，构建并返回一个 ggml_tensor 对象
    ggml_tensor * build_graph(ggml_context * ctx) override {
        ggml_tensor * a = ggml_new_tensor(ctx, type, 4, ne.data());
        ggml_tensor * out = ggml_diag_mask_inf(ctx, a, n_past);
        return out;
    }
};

// 定义一个结构体 test_soft_max，继承自 test_case
struct test_soft_max : public test_case {
    // 声明常量成员变量 type 和 ne
    const ggml_type type;
    const std::array<int64_t, 4> ne;
    // 重写父类的 vars 方法，返回 type 和 ne 的字符串表示
    std::string vars() override {
        return VARS_TO_STR2(type, ne);
    }

    // 构造函数，初始化 type 和 ne，默认值为 GGML_TYPE_F32 和 {10, 10, 10, 10}
    test_soft_max(ggml_type type = GGML_TYPE_F32,
            std::array<int64_t, 4> ne = {10, 10, 10, 10})
        : type(type), ne(ne) {}

    // 重写父类的 build_graph 方法，构建计算图
    ggml_tensor * build_graph(ggml_context * ctx) override {
        // 创建一个新的张量 a，类型为 type，维度为 4，大小为 ne 中的数据
        ggml_tensor * a = ggml_new_tensor(ctx, type, 4, ne.data());
        // 对张量 a 进行 soft max 操作，得到输出张量 out
        ggml_tensor * out = ggml_soft_max(ctx, a);
        // 返回输出张量 out
        return out;
    }
// 结构体 test_rope 继承自 test_case，用于测试 GGML_OP_ROPE 操作
struct test_rope : public test_case {
    // 声明常量成员变量 type 和 ne
    const ggml_type type;
    const std::array<int64_t, 4> ne;
    // 声明变量 n_dims, mode, n_ctx
    int n_dims;
    int mode;
    int n_ctx;

    // 重写父类方法，返回变量的字符串表示
    std::string vars() override {
        return VARS_TO_STR5(type, ne, n_dims, mode, n_ctx);
    }

    // 构造函数，初始化成员变量
    test_rope(ggml_type type = GGML_TYPE_F32,
            std::array<int64_t, 4> ne = {10, 10, 10, 1},
            int n_dims = 10, int mode = 0, int n_ctx = 512)
        : type(type), ne(ne), n_dims(n_dims), mode(mode), n_ctx(n_ctx) {}

    // 构建图形，返回 ggml_tensor 指针
    ggml_tensor * build_graph(ggml_context * ctx) override {
        ggml_tensor * a = ggml_new_tensor(ctx, type, 4, ne.data());
        ggml_tensor * pos = ggml_new_tensor_1d(ctx, GGML_TYPE_I32, ne[2]);
        ggml_tensor * out = ggml_rope(ctx, a, pos, n_dims, mode, n_ctx);
        return out;
    }

    // 初始化张量
    void initialize_tensors(ggml_context * ctx) override {
        for (ggml_tensor * t = ggml_get_first_tensor(ctx); t != NULL; t = ggml_get_next_tensor(ctx, t)) {
            if (t->type == GGML_TYPE_I32) {
                // 初始化 pos 张量的数据
                std::vector<int> data(ne[2]);
                for (int i = 0; i < ne[2]; i++) {
                    data[i] = rand() % n_ctx;
                }
                ggml_backend_tensor_set(t, data.data(), 0, ne[2] * sizeof(int));
            } else {
                // 初始化其他类型的张量
                init_tensor_uniform(t);
            }
        }
    }
};

// 结构体 test_alibi 继承自 test_case，用于测试 GGML_OP_ALIBI 操作
struct test_alibi : public test_case {
    // 声明常量成员变量 type 和 ne
    const ggml_type type;
    const std::array<int64_t, 4> ne;
    // 声明变量 n_past, n_head, bias_max
    int n_past;
    int n_head;
    float bias_max;

    // 重写父类方法，返回变量的字符串表示
    std::string vars() override {
        return VARS_TO_STR5(type, ne, n_past, n_head, bias_max);
    }

    // 构造函数，初始化成员变量
    test_alibi(ggml_type type = GGML_TYPE_F32,
            std::array<int64_t, 4> ne = {10, 10, 10, 10},
            int n_past = 512, int n_head = 10, float bias_max = 0.5f)
        : type(type), ne(ne), n_past(n_past), n_head(n_head), bias_max(bias_max) {}
    # 重写 build_graph 方法，构建图形
    ggml_tensor * build_graph(ggml_context * ctx) override {
        # 创建一个新的张量 a，使用给定的类型和数据维度
        ggml_tensor * a = ggml_new_tensor(ctx, type, 4, ne.data());
        # 使用 ggml_alibi 函数对张量 a 进行处理，得到输出张量 out
        ggml_tensor * out = ggml_alibi(ctx, a, n_past, n_head, bias_max);
        # 返回输出张量
        return out;
    }
// 结构体 test_im2col，继承自 test_case
struct test_im2col : public test_case {
    // 输入数据类型
    const ggml_type type_input;
    // 卷积核数据类型
    const ggml_type type_kernel;
    // 输入数据的维度
    const std::array<int64_t, 4> ne_input;
    // 卷积核的维度
    const std::array<int64_t, 4> ne_kernel;
    // 步长
    const int s0;
    const int s1;
    // 填充
    const int p0;
    const int p1;
    // 膨胀
    const int d0;
    const int d1;
    // 模式
    const bool is_2D;

    // 返回变量的字符串表示
    std::string vars() override {
        return VARS_TO_STR11(type_input, type_kernel, ne_input, ne_kernel, s0, s1, p0, p1, d0, d1, is_2D);
    }

    // 构造函数
    test_im2col(ggml_type type_input = GGML_TYPE_F32, ggml_type type_kernel = GGML_TYPE_F16,
            std::array<int64_t, 4> ne_input = {10, 10, 3, 1}, // [input_width, input_height, input_channels, 1]
            std::array<int64_t, 4> ne_kernel = {3, 3, 3, 1}, // [kernel_width, kernel_height, input_channels, 1]
            int s0 = 1, int s1 = 1,
            int p0 = 1, int p1 = 1,
            int d0 = 1, int d1 = 1,
            bool is_2D = true)
        : type_input(type_input), type_kernel(type_kernel), ne_input(ne_input), ne_kernel(ne_kernel), s0(s0), s1(s1), p0(p0), p1(p1), d0(d0), d1(d1), is_2D(is_2D) {}

    // 构建图形
    ggml_tensor * build_graph(ggml_context * ctx) override {
        // 创建输入张量
        ggml_tensor * input = ggml_new_tensor(ctx, type_input, 4, ne_input.data());
        // 创建卷积核张量
        ggml_tensor * kernel = ggml_new_tensor(ctx, type_kernel, 4, ne_kernel.data());
        // 执行 im2col 操作
        ggml_tensor * out = ggml_im2col(ctx, kernel, input, s0, s1, p0, p1, d0, d1, is_2D);
        // 返回输出张量
        return out;
    }
};

// 结构体 test_concat，继承自 test_case
struct test_concat : public test_case {
    // 数据类型
    const ggml_type type;
    // 维度
    const std::array<int64_t, 4> ne;
    // 第二个维度
    const int64_t b_ne2;

    // 返回变量的字符串表示
    std::string vars() override {
        return VARS_TO_STR3(type, ne, b_ne2);
    }

    // 构造函数
    test_concat(ggml_type type = GGML_TYPE_F32,
            std::array<int64_t, 4> ne = {10, 10, 10, 10},
            int64_t b_ne2 = 10)
        : type(type), ne(ne), b_ne2(b_ne2) {}
    # 重写 build_graph 方法，构建计算图
    ggml_tensor * build_graph(ggml_context * ctx) override {
        # 创建一个四维张量 a，类型为 type，形状为 ne.data()
        ggml_tensor * a = ggml_new_tensor(ctx, type, 4, ne.data());
        # 创建一个四维张量 b，类型为 type，形状为 ne[0], ne[1], b_ne2, ne[3]
        ggml_tensor * b = ggml_new_tensor_4d(ctx, type, ne[0], ne[1], b_ne2, ne[3]);
        # 将张量 a 和张量 b 进行拼接，得到输出张量 out
        ggml_tensor * out = ggml_concat(ctx, a, b);
        # 返回输出张量 out
        return out;
    }
// 结构体 test_argsort 继承自 test_case，用于测试 ggml_argsort 函数
struct test_argsort : public test_case {
    // 声明成员变量，分别表示 ggml_type 类型、包含4个int64_t元素的数组、ggml_sort_order 类型
    const ggml_type type;
    const std::array<int64_t, 4> ne;
    ggml_sort_order order;

    // 重写父类的虚函数，返回成员变量的字符串表示
    std::string vars() override {
        return VARS_TO_STR3(type, ne, order);
    }

    // 构造函数，初始化成员变量
    test_argsort(ggml_type type = GGML_TYPE_F32,
            std::array<int64_t, 4> ne = {16, 10, 10, 10},
            ggml_sort_order order = GGML_SORT_ASC)
        : type(type), ne(ne), order(order) {}

    // 重写父类的虚函数，构建计算图并返回输出的 ggml_tensor 指针
    ggml_tensor * build_graph(ggml_context * ctx) override {
        ggml_tensor * a = ggml_new_tensor(ctx, type, 4, ne.data());
        ggml_tensor * out = ggml_argsort(ctx, a, order);
        return out;
    }

    // 重写父类的虚函数，初始化张量
    void initialize_tensors(ggml_context * ctx) override {
        // 使用随机数初始化张量
        std::random_device rd;
        std::default_random_engine rng(rd());
        for (ggml_tensor * t = ggml_get_first_tensor(ctx); t != NULL; t = ggml_get_next_tensor(ctx, t)) {
            if (t->type == GGML_TYPE_I32) {
                // 初始化索引张量
                std::vector<int> data(ggml_nelements(t));
                for (int i = 0; i < ggml_nelements(t); i++) {
                    data[i] = rand();
                }
                std::shuffle(data.begin(), data.end(), rng);
                ggml_backend_tensor_set(t, data.data(), 0, ne[0]*ne[1]*ne[2]*ne[3] * sizeof(int));
            } else if (t->type == GGML_TYPE_F32) {
                // 使用唯一值初始化张量，避免相同值
                for (int64_t r = 0; r < ggml_nrows(t); r++) {
                    std::vector<float> data(t->ne[0]);
                    for (int i = 0; i < t->ne[0]; i++) {
                        data[i] = i;
                    }
                    std::shuffle(data.begin(), data.end(), rng);
                    ggml_backend_tensor_set(t, data.data(), r * t->nb[1], t->ne[0] * sizeof(float));
                }
            } else {
                GGML_ASSERT(false);
            }
        }
    }
};

// 结构体 test_sum_rows 继承自 test_case
struct test_sum_rows : public test_case {
    // 定义一个常量 ggml_type 类型的变量 type
    const ggml_type type;
    // 定义一个常量包含4个int64_t类型元素的数组 ne
    const std::array<int64_t, 4> ne;

    // 重写父类的 vars 方法，返回 type 和 ne 的字符串表示
    std::string vars() override {
        return VARS_TO_STR2(type, ne);
    }

    // 构造函数 test_sum_rows，初始化 type 和 ne
    test_sum_rows(ggml_type type = GGML_TYPE_F32,
            std::array<int64_t, 4> ne = {10, 10, 10, 10})
        : type(type), ne(ne) {}

    // 重写父类的 build_graph 方法，构建图并返回结果
    ggml_tensor * build_graph(ggml_context * ctx) override {
        // 创建一个新的 tensor 对象 a
        ggml_tensor * a = ggml_new_tensor(ctx, type, 4, ne.data());
        // 对 tensor a 进行行求和操作，返回结果
        ggml_tensor * out = ggml_sum_rows(ctx, a);
        // 返回求和结果
        return out;
    }
// 定义结构体 test_upscale，继承自 test_case
struct test_upscale : public test_case {
    // 声明成员变量 type，ne，scale_factor
    const ggml_type type;
    const std::array<int64_t, 4> ne;
    const int32_t scale_factor;

    // 重写父类方法，返回成员变量的字符串表示
    std::string vars() override {
        return VARS_TO_STR3(type, ne, scale_factor);
    }

    // 构造函数，初始化成员变量
    test_upscale(ggml_type type = GGML_TYPE_F32,
            std::array<int64_t, 4> ne = {512, 512, 3, 1},
            int32_t scale_factor = 2)
        : type(type), ne(ne), scale_factor(scale_factor) {}

    // 重写父类方法，构建图并返回输出张量
    ggml_tensor * build_graph(ggml_context * ctx) override {
        ggml_tensor * a = ggml_new_tensor(ctx, type, 4, ne.data());
        ggml_tensor * out = ggml_upscale(ctx, a, scale_factor);
        return out;
    }
};

// 定义结构体 test_group_norm，继承自 test_case
struct test_group_norm : public test_case {
    // 声明成员变量 type，ne，num_groups
    const ggml_type type;
    const std::array<int64_t, 4> ne;
    const int32_t num_groups;

    // 重写父类方法，返回成员变量的字符串表示
    std::string vars() override {
        return VARS_TO_STR3(type, ne, num_groups);
    }

    // 构造函数，初始化成员变量
    test_group_norm(ggml_type type = GGML_TYPE_F32,
            std::array<int64_t, 4> ne = {64, 64, 320, 1},
            int32_t num_groups = 32)
        : type(type), ne(ne), num_groups(num_groups) {}

    // 重写父类方法，构建图并返回输出张量
    ggml_tensor * build_graph(ggml_context * ctx) override {
        ggml_tensor * a = ggml_new_tensor(ctx, type, 4, ne.data());
        ggml_tensor * out = ggml_group_norm(ctx, a, num_groups);
        return out;
    }
};

// 定义结构体 test_acc，继承自 test_case
struct test_acc : public test_case {
    // 声明成员变量 type，ne_a，ne_b
    const ggml_type type;
    const std::array<int64_t, 4> ne_a;
    const std::array<int64_t, 4> ne_b;

    // 重写父类方法，返回成员变量的字符串表示
    std::string vars() override {
        return VARS_TO_STR3(type, ne_a, ne_b);
    }

    // 构造函数，初始化成员变量
    test_acc(ggml_type type = GGML_TYPE_F32,
            std::array<int64_t, 4> ne_a = {1024, 577, 1, 1},
            std::array<int64_t, 4> ne_b = {1024, 576, 1, 1})
        : type(type), ne_a(ne_a), ne_b(ne_b) {}
    # 重写父类方法，构建图形
    ggml_tensor * build_graph(ggml_context * ctx) override {
        # 创建一个新的张量a，使用给定的数据和类型
        ggml_tensor * a = ggml_new_tensor(ctx, type, 4, ne_a.data());
        # 创建一个新的张量b，使用给定的数据和类型
        ggml_tensor * b = ggml_new_tensor(ctx, type, 4, ne_b.data());
        # 使用上下文和两个张量a、b，以及它们的特定维度，计算得到一个新的张量out
        ggml_tensor * out = ggml_acc(ctx, a, b, a->nb[1], a->nb[2], a->nb[3], b->nb[1]);
        # 返回计算结果张量out
        return out;
    }
// 结构体 test_pad 继承自 test_case，用于测试 GGML_OP_PAD 操作
struct test_pad : public test_case {
    // 定义成员变量，分别表示类型、数组、两个填充值
    const ggml_type type;
    const std::array<int64_t, 4> ne_a;
    const int pad_0;
    const int pad_1;

    // 重写父类方法，返回成员变量的字符串表示
    std::string vars() override {
        return VARS_TO_STR4(type, ne_a, pad_0, pad_1);
    }

    // 构造函数，初始化成员变量
    test_pad(ggml_type type = GGML_TYPE_F32,
            std::array<int64_t, 4> ne_a = {512, 512, 1, 1},
            int pad_0 = 1, int pad_1 = 1)
        : type(type), ne_a(ne_a), pad_0(pad_0), pad_1(pad_1)  {}

    // 重写父类方法，构建图并返回输出张量
    ggml_tensor * build_graph(ggml_context * ctx) override {
        ggml_tensor * a = ggml_new_tensor(ctx, type, 4, ne_a.data());
        ggml_tensor * out = ggml_pad(ctx, a, pad_0, pad_1, 0, 0);
        return out;
    }
};

// 结构体 test_leaky_relu 继承自 test_case，用于测试 GGML_OP_LEAKY_RELU 操作
struct test_leaky_relu : public test_case {
    // 定义成员变量，分别表示类型、数组、负斜率
    const ggml_type type;
    const std::array<int64_t, 4> ne_a;
    const float negative_slope;

    // 重写父类方法，返回成员变量的字符串表示
    std::string vars() override {
        return VARS_TO_STR3(type, ne_a, negative_slope);
    }

    // 构造函数，初始化成员变量
    test_leaky_relu(ggml_type type = GGML_TYPE_F32,
            std::array<int64_t, 4> ne_a = {10, 10, 10, 10},
            float negative_slope = 0.1f)
        : type(type), ne_a(ne_a), negative_slope(negative_slope)  {}

    // 重写父类方法，构建图并返回输出张量
    ggml_tensor * build_graph(ggml_context * ctx) override {
        ggml_tensor * a = ggml_new_tensor(ctx, type, 4, ne_a.data());
        ggml_tensor * out = ggml_leaky_relu(ctx, a, negative_slope, true);
        return out;
    }
};

// 结构体 test_moe 继承自 test_case，用于测试 Mixtral MOE 操作
struct test_moe : public test_case {
    // 定义成员变量，表示专家数量、每个令牌的专家数量、令牌数量、嵌入维度、前馈网络维度
    const int n_experts;
    const int n_experts_per_tok;
    const int n_tokens;
    const int n_embd;
    const int n_ff;

    // 重写父类方法，返回操作描述
    std::string op_desc(ggml_tensor * t) override {
        return "MOE";

        GGML_UNUSED(t);
    }

    // 重写父类方法，返回成员变量的字符串表示
    std::string vars() override {
        return VARS_TO_STR5(n_experts, n_experts_per_tok, n_tokens, n_embd, n_ff);
    }
    # 定义一个名为 test_moe 的函数，参数包括专家数量、每个标记的专家数量、标记数量、嵌入维度和前馈网络维度
    test_moe(int n_experts = 8, int n_experts_per_tok = 2, int n_tokens = 1, int n_embd = 4096, int n_ff = 14336)
        # 初始化函数的成员变量，包括专家数量、每个标记的专家数量、标记数量、嵌入维度和前馈网络维度
        : n_experts(n_experts), n_experts_per_tok(n_experts_per_tok), n_tokens(n_tokens), n_embd(n_embd), n_ff(n_ff) {
    }
    # 结束函数定义
// 测试后端的函数，接受后端类型、测试模式和操作名作为参数
static bool test_backend(ggml_backend_t backend, test_mode mode, const char * op_name) {
    // 创建测试用例的动态数组
    std::vector<std::unique_ptr<test_case>> test_cases;

    // 所有类型的数组
    const ggml_type all_types[] = {
        GGML_TYPE_F32, GGML_TYPE_F16,
        GGML_TYPE_Q4_0, GGML_TYPE_Q4_1,
        GGML_TYPE_Q5_0, GGML_TYPE_Q5_1,
        GGML_TYPE_Q8_0,
        GGML_TYPE_Q2_K, GGML_TYPE_Q3_K,
        GGML_TYPE_Q4_K, GGML_TYPE_Q5_K,
        GGML_TYPE_Q6_K
    };

    // 一元操作
    for (int op = 0; op < GGML_UNARY_OP_COUNT; op++) {
        test_cases.emplace_back(new test_unary((ggml_unary_op) op));
    }

    // 添加测试用例
    test_cases.emplace_back(new test_get_rows(GGML_TYPE_F32, 1, 8, 2, 1, false));
    for (ggml_type type : all_types) {
        for (int b : {1, 7}) {
            for (bool v : {false, true}) {
                test_cases.emplace_back(new test_get_rows(type, 256, 5, 4, b, v));
            }
        }
    }
    for (int b : {1, 7}) {
        for (bool v : {false, true}) {
            test_cases.emplace_back(new test_get_rows(GGML_TYPE_I32, 256, 5, 4, b, v));
        }
    }

    // 添加测试用例
    test_cases.emplace_back(new test_repeat(GGML_TYPE_F32, {10, 10, 10, 10}, {1, 1, 1, 1}));
    test_cases.emplace_back(new test_repeat(GGML_TYPE_F32, {10, 10, 10, 10}, {2, 1, 1, 1}));
    test_cases.emplace_back(new test_repeat(GGML_TYPE_F32, {10, 10, 10, 10}, {1, 2, 1, 1}));
    test_cases.emplace_back(new test_repeat(GGML_TYPE_F32, {10, 10, 10, 10}, {1, 1, 2, 1}));
    test_cases.emplace_back(new test_repeat(GGML_TYPE_F32, {10, 10, 10, 10}, {1, 1, 1, 2}));
    test_cases.emplace_back(new test_repeat(GGML_TYPE_I32, {10, 10, 10, 10}, {2, 1, 1, 1}));
    test_cases.emplace_back(new test_repeat(GGML_TYPE_I16, {10, 10, 10, 10}, {1, 1, 1, 2}));

    // 添加测试用例
    test_cases.emplace_back(new test_dup(GGML_TYPE_F32));
    test_cases.emplace_back(new test_dup(GGML_TYPE_F16));
    test_cases.emplace_back(new test_dup(GGML_TYPE_I32));
    test_cases.emplace_back(new test_dup(GGML_TYPE_I16));
    // 将包含指定值和索引的对象添加到测试用例列表中
    test_cases.emplace_back(new test_dup(GGML_TYPE_I16, {10, 8, 3, 1}, {0, 2, 1, 3}));
    test_cases.emplace_back(new test_dup(GGML_TYPE_I16, {10, 8, 3, 1}, {1, 2, 0, 3});

    // 遍历所有类型，将包含指定值和索引的对象添加到测试用例列表中
    for (ggml_type type : all_types) {
       test_cases.emplace_back(new test_cpy(GGML_TYPE_F32, type, {256, 10, 10, 1}));
    }

    // 将包含默认值的对象添加到测试用例列表中
    test_cases.emplace_back(new test_cont());

    // 定义并添加二进制广播测试用例
    auto add_test_bin_bcast = [&](ggml_type type, std::array<int64_t, 4> ne, std::array<int, 4> nr) {
        for (auto op : {ggml_add, ggml_mul, ggml_div}) {
            test_cases.emplace_back(new test_bin_bcast(op, type, ne, nr));
        }
    };

    // 添加不同参数的二进制广播测试用例
    add_test_bin_bcast(GGML_TYPE_F32, {1, 1, 8, 1}, {1, 1, 1, 1});
    add_test_bin_bcast(GGML_TYPE_F32, {1, 1, 1, 1}, {32, 1, 1, 1});
    // ... 其他测试用例

    // 添加稳定扩散的二进制广播测试用例
    add_test_bin_bcast(GGML_TYPE_F32, {1280, 1, 1, 1}, {1, 1, 1, 1});
    add_test_bin_bcast(GGML_TYPE_F32, {1280, 1, 1, 1}, {1, 16, 16, 1});
    // ... 其他测试用例
    # 添加测试用例，对类型为 GGML_TYPE_F32 的数据进行广播操作，传入输入形状和输出形状
    add_test_bin_bcast(GGML_TYPE_F32, {1, 1, 1920, 1}, {16, 16, 1, 1});
    add_test_bin_bcast(GGML_TYPE_F32, {1, 1, 2560, 1}, {16, 16, 1, 1});
    add_test_bin_bcast(GGML_TYPE_F32, {1, 1, 1280, 1}, {32, 32, 1, 1});
    add_test_bin_bcast(GGML_TYPE_F32, {1, 1, 1920, 1}, {32, 32, 1, 1});
    add_test_bin_bcast(GGML_TYPE_F32, {1, 1, 640, 1}, {32, 32, 1, 1});
    add_test_bin_bcast(GGML_TYPE_F32, {5120, 1, 1, 1}, {1, 256, 1, 1});
    add_test_bin_bcast(GGML_TYPE_F32, {640, 1, 1, 1}, {1, 1, 1, 1});
    # 添加测试用例对象到测试用例列表
    test_cases.emplace_back(new test_scale());
    # 遍历 eps 取值为 1e-6f, 1e-5f, 1e-3f, 1e-1f，分别创建 test_norm 和 test_rms_norm 测试用例对象，加入测试用例列表
    for (float eps : {1e-6f, 1e-5f, 1e-3f, 1e-1f}) {
        test_cases.emplace_back(new test_norm(GGML_TYPE_F32, {64, 10, 10, 10}, eps));
        test_cases.emplace_back(new test_rms_norm(GGML_TYPE_F32, {64, 10, 10, 10}, eps));
    }
    # 遍历所有类型的组合
    for (ggml_type type_a : all_types) {
        # 遍历指定类型的组合
        for (ggml_type type_b : {GGML_TYPE_F32, GGML_TYPE_F16}) {
            # 添加测试用例，创建新的矩阵乘法测试对象，参数为类型A、类型B、矩阵大小、输入通道数、输出通道数、输入尺寸、输出尺寸
            test_cases.emplace_back(new test_mul_mat(type_a, type_b, 16, 1, 256, { 1,  1}, {1, 1}));
            test_cases.emplace_back(new test_mul_mat(type_a, type_b, 16, 1, 256, {10,  1}, {1, 1}));
            test_cases.emplace_back(new test_mul_mat(type_a, type_b, 16, 1, 256, {10,  1}, {2, 1}));
            test_cases.emplace_back(new test_mul_mat(type_a, type_b, 16, 1, 256, {10, 10}, {1, 1}));
            test_cases.emplace_back(new test_mul_mat(type_a, type_b, 16, 1, 256, {10, 10}, {2, 1}));
            test_cases.emplace_back(new test_mul_mat(type_a, type_b, 16, 1, 256, {10, 10}, {1, 2}));
            test_cases.emplace_back(new test_mul_mat(type_a, type_b, 16, 1, 256, {10, 10}, {2, 2}));

            test_cases.emplace_back(new test_mul_mat(type_a, type_b, 16, 16, 256, { 1,  1}, {1, 1}));
            test_cases.emplace_back(new test_mul_mat(type_a, type_b, 16, 16, 256, {10,  1}, {1, 1}));
            test_cases.emplace_back(new test_mul_mat(type_a, type_b, 16, 16, 256, {10,  1}, {2, 1}));
            test_cases.emplace_back(new test_mul_mat(type_a, type_b, 16, 16, 256, {10, 10}, {1, 1}));
            test_cases.emplace_back(new test_mul_mat(type_a, type_b, 16, 16, 256, {10, 10}, {2, 1}));
            test_cases.emplace_back(new test_mul_mat(type_a, type_b, 16, 16, 256, {10, 10}, {1, 2}));
            test_cases.emplace_back(new test_mul_mat(type_a, type_b, 16, 16, 256, {10, 10}, {2, 2}));
        }
    }

    # 遍历所有类型的组合
    for (ggml_type type_a : all_types) {
        # 遍历指定类型的组合
        for (ggml_type type_b : {GGML_TYPE_F32 /*, GGML_TYPE_F16 */}) {
            # 遍历不同矩阵数量
            for (int n_mats : {2, 4, 8}) {
                # 遍历每个矩阵的ID
                for (int id = 0; id < n_mats; id++) {
                    # 遍历布尔值
                    for (bool v : {false, true}) {
                        # 添加测试用例，创建新的矩阵乘法测试对象，参数为类型A、类型B、矩阵数量、矩阵ID、输入通道数、输出通道数、输入尺寸、输出尺寸、布尔值
                        test_cases.emplace_back(new test_mul_mat_id(type_a, type_b, n_mats, id, 16, 16, 256, v));
                    }
                }
            }
        }
    }
    # 将 test_sqr 对象指针添加到 test_cases 中
    test_cases.emplace_back(new test_sqr());
    # 将 test_clamp 对象指针添加到 test_cases 中
    test_cases.emplace_back(new test_clamp());

    # 将 test_diag_mask_inf 对象指针添加到 test_cases 中，传入参数为 GGML_TYPE_F32, {10, 10,  1,  1}, 5
    test_cases.emplace_back(new test_diag_mask_inf(GGML_TYPE_F32, {10, 10,  1,  1}, 5));
    # 将 test_diag_mask_inf 对象指针添加到 test_cases 中，传入参数为 GGML_TYPE_F32, {10, 10, 10,  1}, 5
    test_cases.emplace_back(new test_diag_mask_inf(GGML_TYPE_F32, {10, 10, 10,  1}, 5));
    # 将 test_diag_mask_inf 对象指针添加到 test_cases 中，传入参数为 GGML_TYPE_F32, {10, 10, 10, 10}, 5
    test_cases.emplace_back(new test_diag_mask_inf(GGML_TYPE_F32, {10, 10, 10, 10}, 5));

    # 将 test_soft_max 对象指针添加到 test_cases 中
    test_cases.emplace_back(new test_soft_max());

    # 遍历 GGML_TYPE_F32 和 GGML_TYPE_F16
    for (ggml_type type : {GGML_TYPE_F32, GGML_TYPE_F16}) {
        # 将 test_rope 对象指针添加到 test_cases 中，传入参数为 type, {128,  32, 10, 1}, 128, 0, 512
        test_cases.emplace_back(new test_rope(type, {128,  32, 10, 1}, 128, 0, 512)); // llama 7B
        # 将 test_rope 对象指针添加到 test_cases 中，传入参数为 type, {128,  40, 10, 1}, 128, 0, 512
        test_cases.emplace_back(new test_rope(type, {128,  40, 10, 1}, 128, 0, 512)); // llama 13B
        # 将 test_rope 对象指针添加到 test_cases 中，传入参数为 type, {128,  52, 10, 1}, 128, 0, 512
        test_cases.emplace_back(new test_rope(type, {128,  52, 10, 1}, 128, 0, 512)); // llama 30B
        # 将 test_rope 对象指针添加到 test_cases 中，传入参数为 type, {128,  64, 10, 1}, 128, 0, 512
        test_cases.emplace_back(new test_rope(type, {128,  64, 10, 1}, 128, 0, 512)); // llama 65B
        # 将 test_rope 对象指针添加到 test_cases 中，传入参数为 type, { 64,   1, 10, 1},  64, 2, 512
        test_cases.emplace_back(new test_rope(type, { 64,   1, 10, 1},  64, 2, 512)); // neox (falcon 7B)
        # 将 test_rope 对象指针添加到 test_cases 中，传入参数为 type, { 64,  71, 10, 1},  64, 2, 512
        test_cases.emplace_back(new test_rope(type, { 64,  71, 10, 1},  64, 2, 512)); // neox (falcon 7B)
        # 将 test_rope 对象指针添加到 test_cases 中，传入参数为 type, { 64,   8, 10, 1},  64, 2, 512
        test_cases.emplace_back(new test_rope(type, { 64,   8, 10, 1},  64, 2, 512)); // neox (falcon 40B)
        # 将 test_rope 对象指针添加到 test_cases 中，传入参数为 type, { 64, 128, 10, 1},  64, 2, 512
        test_cases.emplace_back(new test_rope(type, { 64, 128, 10, 1},  64, 2, 512)); // neox (falcon 40B)
        # 将 test_rope 对象指针添加到 test_cases 中，传入参数为 type, { 80,  32, 10, 1},  20, 2, 512
        test_cases.emplace_back(new test_rope(type, { 80,  32, 10, 1},  20, 2, 512)); // neox (stablelm)
        # 将 test_rope 对象指针添加到 test_cases 中，传入参数为 type, { 80,  32, 10, 1},  32, 2, 512
        test_cases.emplace_back(new test_rope(type, { 80,  32, 10, 1},  32, 2, 512)); // neox (phi-2)
    }

    # 将 test_alibi 对象指针添加到 test_cases 中
    test_cases.emplace_back(new test_alibi());
    # 将 test_im2col 对象指针添加到 test_cases 中
    test_cases.emplace_back(new test_im2col());
    # 将 test_concat 对象指针添加到 test_cases 中，传入参数为 GGML_TYPE_F32
    test_cases.emplace_back(new test_concat(GGML_TYPE_F32));
    # 将 test_concat 对象指针添加到 test_cases 中，传入参数为 GGML_TYPE_I32
    test_cases.emplace_back(new test_concat(GGML_TYPE_I32));

    # 遍历 GGML_SORT_ASC 和 GGML_SORT_DESC
    for (ggml_sort_order order : {GGML_SORT_ASC, GGML_SORT_DESC}) {
        # 将 test_argsort 对象指针添加到 test_cases 中，传入参数为 GGML_TYPE_F32, {8, 1, 1, 1}, order
        test_cases.emplace_back(new test_argsort(GGML_TYPE_F32, {8, 1, 1, 1}, order));
        # 将 test_argsort 对象指针添加到 test_cases 中，传入参数为 GGML_TYPE_F32, {16, 10, 10, 10}, order
        test_cases.emplace_back(new test_argsort(GGML_TYPE_F32, {16, 10, 10, 10}, order));
    }
    # 将 test_sum_rows() 对象添加到 test_cases 列表中
    test_cases.emplace_back(new test_sum_rows());
    # 将 test_upscale() 对象添加到 test_cases 列表中
    test_cases.emplace_back(new test_upscale());
    # 将 test_group_norm() 对象添加到 test_cases 列表中
    test_cases.emplace_back(new test_group_norm());
    # 将 test_acc() 对象添加到 test_cases 列表中
    test_cases.emplace_back(new test_acc());
    # 将 test_pad() 对象添加到 test_cases 列表中
    test_cases.emplace_back(new test_pad());
    # 将 test_leaky_relu() 对象添加到 test_cases 列表中
    test_cases.emplace_back(new test_leaky_relu());
#if !defined(__SANITIZE_THREAD__)
    // 如果未定义__SANITIZE_THREAD__，则执行以下代码块
    // FIXME: these tests use too much memory with thread sanitizer
    // FIXME: 这些测试在使用线程检测器时会占用太多内存
    // 创建一个 test_moe 对象并添加到 test_cases 中
    test_cases.emplace_back(new test_moe(8, 2, 1, 4096, 8*1024));
    //test_cases.emplace_back(new test_moe(8, 2, 8, 4096, 14336));
#endif

    // 运行测试
    if (mode == MODE_TEST) {
        // 初始化 CPU 后端
        ggml_backend_t backend_cpu = ggml_backend_cpu_init();

        // 通过循环遍历 test_cases，对每个测试进行评估
        size_t n_ok = 0;
        for (auto & test : test_cases) {
            if (test->eval(backend, backend_cpu, op_name)) {
                n_ok++;
            }
        }
        // 打印通过的测试数量
        printf("  %zu/%zu tests passed\n", n_ok, test_cases.size());

        // 释放 CPU 后端资源
        ggml_backend_free(backend_cpu);

        // 如果所有测试都通过，则返回 true
        return n_ok == test_cases.size();
    }

    // 如果运行模式为性能评估
    if (mode == MODE_PERF) {
        // 对每个测试进行性能评估
        for (auto & test : test_cases) {
            test->eval_perf(backend, op_name);
        }
        // 返回 true
        return true;
    }

    // 如果运行模式不是测试也不是性能评估，则断言失败
    GGML_ASSERT(false);
    // 返回 false
    return false;
}

// 显示程序的使用方法
static void usage(char ** argv) {
    printf("Usage: %s [mode] [-o op] [-b backend]\n", argv[0]);
    printf("  valid modes are: test (compare with CPU backend for correctness) or perf (performance evaluation)\n");
    printf("  op names are as given by ggml_op_desc()\n");
}

// 主函数
int main(int argc, char ** argv) {
    // 初始化默认模式、操作名称和后端名称
    test_mode mode = MODE_TEST;
    const char * op_name = NULL;
    const char * backend = NULL;

    // 遍历命令行参数
    for (int i = 1; i < argc; i++) {
        // 根据参数设置运行模式、操作名称和后端名称
        if (strcmp(argv[i], "test") == 0) {
            mode = MODE_TEST;
        } else if (strcmp(argv[i], "perf") == 0) {
            mode = MODE_PERF;
        } else if (strcmp(argv[i], "-o") == 0) {
            if (i + 1 < argc) {
                op_name = argv[++i];
            } else {
                // 显示使用方法并返回错误码
                usage(argv);
                return 1;
            }
        } else if (strcmp(argv[i], "-b") == 0) {
            if (i + 1 < argc) {
                backend = argv[++i];
            } else {
                // 显示使用方法并返回错误码
                usage(argv);
                return 1;
            }
        } else {
            // 显示使用方法并返回错误码
            usage(argv);
            return 1;
        }
    }
}
    // 枚举后端
    printf("Testing %zu backends\n\n", ggml_backend_reg_get_count());

    size_t n_ok = 0;

    for (size_t i = 0; i < ggml_backend_reg_get_count(); i++) {
        printf("Backend %zu/%zu (%s)\n", i + 1, ggml_backend_reg_get_count(), ggml_backend_reg_get_name(i));

        // 如果指定了后端并且当前后端不匹配，则跳过
        if (backend != NULL && strcmp(backend, ggml_backend_reg_get_name(i)) != 0) {
            printf("  Skipping\n");
            n_ok++;
            continue;
        }

        // 初始化后端
        ggml_backend_t backend = ggml_backend_reg_init_backend(i, NULL);
        GGML_ASSERT(backend != NULL);
        printf("  Backend name: %s\n", ggml_backend_name(backend));

        // 测试后端
        bool ok = test_backend(backend, mode, op_name);

        printf("  Backend %s: ", ggml_backend_name(backend));
        if (ok) {
            printf("\033[1;32mOK\033[0m\n");
            n_ok++;
        } else {
            printf("\033[1;31mFAIL\033[0m\n");
        }

        printf("\n");

        // 释放后端资源
        ggml_backend_free(backend);
    }

    // 输出通过测试的后端数量
    printf("%zu/%zu backends passed\n", n_ok, ggml_backend_reg_get_count());

    // 如果通过测试的后端数量不等于总后端数量，则返回失败
    if (n_ok != ggml_backend_reg_get_count()) {
        printf("\033[1;31mFAIL\033[0m\n");
        return 1;
    }

    // 返回成功
    printf("\033[1;32mOK\033[0m\n");
    return 0;
# 闭合前面的函数定义
```