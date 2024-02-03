# `chatglm.cpp\tests\perplexity.cpp`

```cpp
// 包含头文件 chatglm.h
#include "chatglm.h"
// 包含标准库头文件
#include <algorithm>
#include <chrono>
#include <fstream>
#include <iostream>

// 定义结构体 Args，包含模型路径、语料路径、最大长度、步长和线程数等参数
struct Args {
    std::string model_path = "chatglm-ggml.bin";
    std::string corpus_path = "data/wikitext-2-raw/wiki.test.raw";
    int max_length = 1024;
    int stride = 512;
    int num_threads = 0;
};

// 显示程序使用方法
static void usage(const std::string &prog) {
    std::cout << "Usage: " << prog << R"( [options]

options:
  -h, --help            show this help message and exit
  -m, --model PATH      model path
  -f, --file            path to the corpus
  -l, --max_length N    max total length including prompt and output
  -s, --stride N        stride size of the sliding window
  -t, --threads N       number of threads for inference
)";
}

// 解析命令行参数
static Args parse_args(const std::vector<std::string> &argv) {
    Args args;

    for (size_t i = 1; i < argv.size(); i++) {
        const std::string &arg = argv.at(i);

        if (arg == "-h" || arg == "--help") {
            // 显示使用方法并退出程序
            usage(argv.at(0));
            exit(EXIT_SUCCESS);
        } else if (arg == "-m" || arg == "--model") {
            // 设置模型路径
            args.model_path = argv.at(++i);
        } else if (arg == "-f" || arg == "--file") {
            // 设置语料路径
            args.corpus_path = argv.at(++i);
        } else if (arg == "-l" || arg == "--max_length") {
            // 设置最大长度
            args.max_length = std::stoi(argv.at(++i));
        } else if (arg == "-s" || arg == "--stride") {
            // 设置步长
            args.stride = std::stoi(argv.at(++i));
        } else if (arg == "-t" || arg == "--threads") {
            // 设置线程数
            args.num_threads = std::stoi(argv.at(++i));
        } else {
            // 未知参数，显示错误信息并退出程序
            std::cerr << "Unknown argument: " << arg << std::endl;
            usage(argv.at(0));
            exit(EXIT_FAILURE);
        }
    }

    return args;
}

// 解析命令行参数
static Args parse_args(int argc, char **argv) {
    // 将命令行参数转换为 vector
    std::vector<std::string> argv_vec(argv, argv + argc);
    // 调用解析参数函数
    return parse_args(argv_vec);
}

// 读取文本文件内容
static std::string read_text(std::string path) {
    // 打开文件流
    std::ifstream fin(path);
    // 检查文件流是否打开成功，如果未成功则输出错误信息
    CHATGLM_CHECK(fin) << "cannot open file " << path;
    // 创建一个字符串流对象
    std::ostringstream oss;
    // 将文件流中的内容读取到字符串流中
    oss << fin.rdbuf();
    // 返回字符串流中的内容作为字符串
    return oss.str();
// 计算交叉熵损失函数
static float cross_entropy(const ggml_tensor *input, const ggml_tensor *target) {
    // 检查输入张量是否是二维且为浮点类型
    CHATGLM_CHECK(ggml_is_contiguous(input) && input->n_dims == 2 && input->type == GGML_TYPE_F32);
    // 检查目标张量是否是一维且为整型
    CHATGLM_CHECK(ggml_is_contiguous(target) && target->n_dims == 1 && target->type == GGML_TYPE_I32);
    // 检查输入张量的列数是否与目标张量的长度相等
    CHATGLM_CHECK(input->ne[1] == target->ne[0]);

    // 获取类别数和批量大小
    const int num_classes = input->ne[0];
    const int batch_size = input->ne[1];

    // 初始化损失值为0
    float loss = 0.f;
    // 使用 OpenMP 并行计算损失值
#pragma omp parallel for reduction(+ : loss)
    for (int i = 0; i < batch_size; i++) {
        // 获取当前样本的目标类别
        const int target_i = ((const int *)target->data)[i];
        // 获取当前样本的输入数据
        const float *row = (const float *)input->data + i * input->ne[0];
        // 计算当前样本中最大值
        const float max_val = *std::max_element(row, row + num_classes);
        // 初始化求和值为0
        float sum = 0.f;
        // 计算 softmax 函数的分母
        for (int j = 0; j < num_classes; j++) {
            sum += std::exp(row[j] - max_val);
        }
        // 计算交叉熵损失
        loss += -(row[target_i] - max_val - std::log(sum));
    }

    // 返回平均损失值
    return loss / batch_size;
}

// 参考链接：https://huggingface.co/docs/transformers/perplexity
// 计算困惑度
static void perplexity(Args &args) {
    // 输出加载模型的信息
    std::cout << "Loading model from " << args.model_path << " ...\n";
    // 加载模型
    chatglm::Pipeline pipeline(args.model_path);

    // 输出加载语料库的信息
    std::cout << "Loading corpus from " << args.corpus_path << " ...\n";
    // 读取文本语料库
    std::string corpus = read_text(args.corpus_path);

    // 输出标记化语料库的信息
    std::cout << "Tokenizing corpus of " << corpus.size() << " bytes ...\n";
    // 对语料库进行标记化
    std::vector<int> corpus_ids = pipeline.tokenizer->encode(corpus, std::numeric_limits<int>::max());
    // 移除前两个标记
    corpus_ids.erase(corpus_ids.begin(), corpus_ids.begin() + 2);

    // 输出计算困惑度的信息
    std::cout << "Computing perplexity against " << corpus_ids.size() << " tokens ...\n";

    // 初始化总损失值和样本数量
    float total_loss = 0.f;
    size_t num_samples = 0;

    // 初始化前一个结束位置
    size_t prev_end = 0;
}
    // 以步长为 args.stride 遍历语料库中的数据
    for (size_t begin = 0; begin < corpus_ids.size(); begin += args.stride) {
        // 记录当前时间
        const auto clk_start = std::chrono::system_clock::now();
        // 计算结束位置，确保不超过最大长度
        size_t end = std::min(begin + args.max_length, corpus_ids.size());
        // 计算目标长度，确保不超过最大长度减一
        size_t target_len = std::min(end - prev_end, size_t(args.max_length - 1));
        // 从语料库中提取输入数据
        std::vector<int> input_ids(corpus_ids.begin() + begin, corpus_ids.begin() + end);

        // 使用模型进行前向计算
        ggml_tensor *lm_logits = pipeline.model->forward_graph_compute(input_ids, 0, 0, args.num_threads, false);

        // 记录前向计算时间
        const auto clk_fwd = std::chrono::system_clock::now();

        // 创建上下文
        auto ctx = chatglm::make_unique_ggml_context(512 * chatglm::MB, nullptr, false);
        // 获取下一个 lm_logits
        ggml_tensor *next_lm_logits = ggml_view_2d(ctx.get(), lm_logits, lm_logits->ne[0], target_len, lm_logits->nb[1],
                                                   (input_ids.size() - target_len - 1) * lm_logits->nb[1]);
        // 创建下一个输入数据
        ggml_tensor *next_input_ids = ggml_new_tensor_1d(ctx.get(), GGML_TYPE_I32, target_len);
        // 复制数据到下一个输入数据
        memcpy(next_input_ids->data, input_ids.data() + input_ids.size() - target_len, target_len * sizeof(int));

        // 计算交叉熵损失
        const float loss = cross_entropy(next_lm_logits, next_input_ids);

        // 更新总损失和样本数量
        total_loss += loss * target_len;
        num_samples += target_len;

        // 记录结束时间
        const auto clk_end = std::chrono::system_clock::now();

        // 计算前向计算时间和交叉熵计算时间
        const auto elapsed_fwd = std::chrono::duration_cast<std::chrono::milliseconds>(clk_fwd - clk_start).count();
        const auto elapsed_ce = std::chrono::duration_cast<std::chrono::milliseconds>(clk_end - clk_fwd).count();

        // 计算进度百分比
        const int progress = end * 100 / corpus_ids.size();
        // 输出当前进度、困惑度、前向计算时间和交叉熵计算时间
        std::cout << "[" << progress << "%] chunk [" << end - target_len << ", " << end
                  << ") perplexity: " << std::fixed << std::setprecision(3) << std::exp(loss)
                  << ", forward time: " << elapsed_fwd << " ms, cross entropy time: " << elapsed_ce << " ms\n";

        // 更新上一个结束位置
        prev_end = end;
        // 如果到达语料库末尾，则跳出循环
        if (end == corpus_ids.size()) {
            break;
        }
    }
    // 计算困惑度（perplexity），使用指数函数计算平均损失除以样本数量的结果
    const float ppl = std::exp(total_loss / num_samples);
    // 输出最终困惑度，保留三位小数
    std::cout << "Final perplexity: " << std::fixed << std::setprecision(3) << ppl << "\n";
}

// 主函数，接收命令行参数，解析参数并计算困惑度
int main(int argc, char **argv) {
    try {
        // 解析命令行参数
        Args args = parse_args(argc, argv);
        // 计算困惑度
        perplexity(args);
    } catch (std::exception &e) {
        // 捕获异常并输出异常信息
        std::cerr << e.what() << std::endl;
        // 退出程序并返回失败状态
        exit(EXIT_FAILURE);
    }
    // 返回成功状态
    return 0;
}
```