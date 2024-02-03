# `whisper.cpp\examples\common.cpp`

```cpp
// 定义 _USE_MATH_DEFINES 以使用 M_PI

#include "common.h"

// 引入第三方工具库
// 使用你喜欢的实现方式
#define DR_WAV_IMPLEMENTATION
#include "dr_wav.h"

#include <cmath> // 数学函数库
#include <cstring> // 字符串操作库
#include <fstream> // 文件操作库
#include <regex> // 正则表达式库
#include <locale> // 本地化库
#include <codecvt> // 编码转换库
#include <sstream> // 字符串流库

#if defined(_MSC_VER)
#pragma warning(disable: 4244 4267) // 禁止警告 4244 和 4267，可能会丢失数据
#endif

// 检查下一个参数是否存在的函数
std::string get_next_arg(int& i, int argc, char** argv, const std::string& flag, gpt_params& params) {
    if (i + 1 < argc && argv[i + 1][0] != '-') {
        return argv[++i];
    } else {
        fprintf(stderr, "error: %s requires one argument.\n", flag.c_str());
        gpt_print_usage(argc, argv, params);
        exit(0);
    }
}

// 解析参数的函数
bool gpt_params_parse(int argc, char ** argv, gpt_params & params) {
    // 实现参数解析的逻辑
    // ...
    return true;
}

// 打印程序用法的函数
void gpt_print_usage(int /*argc*/, char ** argv, const gpt_params & params) {
    fprintf(stderr, "usage: %s [options]\n", argv[0]);
    fprintf(stderr, "\n");
    fprintf(stderr, "options:\n");
    fprintf(stderr, "  -h, --help            show this help message and exit\n");
    fprintf(stderr, "  -s SEED, --seed SEED  RNG seed (default: -1)\n");
    fprintf(stderr, "  -t N, --threads N     number of threads to use during computation (default: %d)\n", params.n_threads);
    fprintf(stderr, "  -p PROMPT, --prompt PROMPT\n");
    fprintf(stderr, "                        prompt to start generation with (default: random)\n");
    fprintf(stderr, "  -f FNAME, --file FNAME\n");
    fprintf(stderr, "                        load prompt from a file\n");
    fprintf(stderr, "  -tt TOKEN_TEST, --token_test TOKEN_TEST\n");
    fprintf(stderr, "                        test tokenization\n");
    fprintf(stderr, "  -n N, --n_predict N   number of tokens to predict (default: %d)\n", params.n_predict);
    fprintf(stderr, "  --top_k N             top-k sampling (default: %d)\n", params.top_k);
}
    # 打印提示信息，显示 top-p 采样的默认值
    fprintf(stderr, "  --top_p N             top-p sampling (default: %.1f)\n", params.top_p);
    # 打印提示信息，显示温度的默认值
    fprintf(stderr, "  --temp N              temperature (default: %.1f)\n", params.temp);
    # 打印提示信息，显示考虑惩罚的最后 n 个标记的默认值
    fprintf(stderr, "  --repeat-last-n N     last n tokens to consider for penalize (default: %d, 0 = disabled)\n", params.repeat_last_n);
    # 打印提示信息，显示重复标记序列的惩罚值的默认值
    fprintf(stderr, "  --repeat-penalty N    penalize repeat sequence of tokens (default: %.2f, 1.0 = disabled)\n", (double)params.repeat_penalty);
    # 打印提示信息，显示提示处理的批量大小的默认值
    fprintf(stderr, "  -b N, --batch_size N  batch size for prompt processing (default: %d)\n", params.n_batch);
    # 打印提示信息，显示上下文/键值缓存大小的默认值
    fprintf(stderr, "  -c N, --context N     context / KV cache size (default: %d)\n", params.n_ctx);
    # 打印提示信息，指示在生成过程中忽略 EOS 标记
    fprintf(stderr, "  --ignore-eos          ignore EOS token during generation\n");
    # 打印提示信息，显示在支持的模型上要卸载到 GPU 的层数的默认值
    fprintf(stderr, "  -ngl N, --gpu-layers N  number of layers to offload to GPU on supported models (default: %d)\n", params.n_gpu_layers);
    # 打印提示信息，显示模型路径的默认值
    fprintf(stderr, "  -m FNAME, --model FNAME\n");
    fprintf(stderr, "                        model path (default: %s)\n", params.model.c_str());
    # 打印空行
    fprintf(stderr, "\n");
}

// 生成随机的 GPT 提示语
std::string gpt_random_prompt(std::mt19937 & rng) {
    // 生成一个 0 到 9 之间的随机数
    const int r = rng() % 10;
    // 根据随机数返回不同的提示语
    switch (r) {
        case 0: return "So";
        case 1: return "Once upon a time";
        case 2: return "When";
        case 3: return "The";
        case 4: return "After";
        case 5: return "If";
        case 6: return "import";
        case 7: return "He";
        case 8: return "She";
        case 9: return "They";
        default: return "To";
    }

    return "The";
}

// 去除字符串两端的空格
std::string trim(const std::string & s) {
    // 创建正则表达式模式
    std::regex e("^\\s+|\\s+$");
    // 使用正则表达式替换字符串中的空格
    return std::regex_replace(s, e, "");
}

// 替换字符串中的指定子串
std::string replace(const std::string & s, const std::string & from, const std::string & to) {
    std::string result = s;
    size_t pos = 0;
    // 在字符串中查找并替换指定子串
    while ((pos = result.find(from, pos)) != std::string::npos) {
        result.replace(pos, from.length(), to);
        pos += to.length();
    }
    return result;
}

// 添加特殊标记到特殊标记列表
void gpt_vocab::add_special_token(const std::string & token) {
    special_tokens.push_back(token);
}

// 解析 JSON 文件并返回键值对
std::map<std::string, int32_t> json_parse(const std::string & fname) {
    std::map<std::string, int32_t> result;

    // 读取文件内容到字符串
    std::string json;
    {
        std::ifstream ifs(fname);
        if (!ifs) {
            fprintf(stderr, "Failed to open %s\n", fname.c_str());
            exit(1);
        }

        json = std::string((std::istreambuf_iterator<char>(ifs)),
                (std::istreambuf_iterator<char>()));
    }

    if (json[0] != '{') {
        return result;
    }

    // 解析 JSON
    }

    return result;
}

// 将宽字符转换为 UTF-8 字符串
std::string convert_to_utf8(const std::wstring & input) {
    std::wstring_convert<std::codecvt_utf8<wchar_t>> converter;
    return converter.to_bytes(input);
}

// 将 UTF-8 字符串转换为宽字符
std::wstring convert_to_wstring(const std::string & input) {
    std::wstring_convert<std::codecvt_utf8<wchar_t>> converter;
    return converter.from_bytes(input);
}

// 将字符串拆分为单词并存储在向量中
void gpt_split_words(std::string str, std::vector<std::string>& words) {
    // 定义正则表达式模式，用于匹配单词缩写、单词、数字、非空白字符等
    const std::string pattern = R"('s|'t|'re|'ve|'m|'ll|'d| ?[[:alpha:]]+| ?[[:digit:]]+| ?[^\s[:alpha:][:digit:]]+|\s+(?!\S)|\s+)";
    // 使用正则表达式模式创建正则表达式对象
    const std::regex re(pattern);
    // 定义 std::smatch 对象，用于存储匹配结果
    std::smatch m;

    // 在字符串 str 中查找匹配正则表达式 re 的内容
    while (std::regex_search(str, m, re)) {
        // 遍历匹配结果，将每个匹配的内容添加到 words 容器中
        for (auto x : m) {
            words.push_back(x);
        }
        // 更新字符串 str，去除已匹配的部分
        str = m.suffix();
    }
}

std::vector<gpt_vocab::id> gpt_tokenize(const gpt_vocab & vocab, const std::string & text) {
    // 创建一个字符串向量用于存储单词
    std::vector<std::string> words;

    // 首先将文本拆分为单词
    {
        // 复制文本内容到新的字符串
        std::string str = text;

        // 如果特殊标记向量不为空，则生成子模式
        if (!vocab.special_tokens.empty()) {
            // 创建用于转义的正则表达式
            const std::regex escape(R"([\[\\\^\$\.\|\?\*\+\(\)\{\}])");
            std::string special_tokens_subpattern;
            // 遍历特殊标记向量，生成子模式
            for (const auto & token : vocab.special_tokens) {
                if (!special_tokens_subpattern.empty()) {
                    special_tokens_subpattern += "|";
                }
                special_tokens_subpattern += std::regex_replace(token, escape, R"(\$&)");
            }

            // 创建正则表达式对象
            std::regex re(special_tokens_subpattern);
            std::smatch m;
            // 通过特殊标记拆分文本
            while (std::regex_search(str, m, re)) {
                // 将特殊标记之间的子字符串拆分为单词
                gpt_split_words(m.prefix(), words);
                // 将匹配的特殊标记作为单词添加到向量中
                for (auto x : m) {
                    words.push_back(x);
                }
                str = m.suffix();
            }
            // 处理剩余没有特殊标记的文本
        }

        // 将剩余文本拆分为单词
        gpt_split_words(str, words);
    }

    // 查找形成每个单词的最长标记：
    std::vector<gpt_vocab::id> tokens;
    // 遍历输入的单词列表
    for (const auto & word : words) {
        // 遍历单词的每个可能的子串
        for (int i = 0; i < (int) word.size(); ){
            // 从单词的末尾向前遍历子串
            for (int j = word.size() - 1; j >= i; j--){
                // 提取当前子串
                auto cand = word.substr(i, j-i+1);
                // 在词汇表中查找当前子串
                auto it = vocab.token_to_id.find(cand);
                // 如果在词汇表中找到了匹配的子串
                if (it != vocab.token_to_id.end()){ // word.substr(i, j-i+1) in vocab
                    // 将匹配的子串对应的 ID 添加到 tokens 中
                    tokens.push_back(it->second);
                    // 更新 i 的值，跳过已匹配的子串
                    i = j + 1;
                    // 跳出内层循环
                    break;
                }
                // 如果当前子串在词汇表中找不到匹配
                else if (j == i){ // word.substr(i, 1) has no matching
                    // 输出错误信息，表示未知的标记
                    fprintf(stderr, "%s: unknown token '%s'\n", __func__, word.substr(i, 1).data());
                    // 更新 i 的值，继续查找下一个子串
                    i++;
                }
            }
        }
    }

    // 返回 tokens 列表
    return tokens;
}

// 从输入字符串中解析出标记并返回一个标记ID的向量
std::vector<gpt_vocab::id> parse_tokens_from_string(const std::string& input, char delimiter) {
    std::vector<gpt_vocab::id> output;
    // 使用输入字符串创建一个字符串流
    std::stringstream ss(input);
    std::string token;

    // 从字符串流中逐行读取标记，并将其转换为整数后添加到输出向量中
    while (std::getline(ss, token, delimiter)) {
        output.push_back(std::stoi(token));
    }

    return output;
}

// 从文件中提取测试数据并返回一个映射，其中键是文本字符串，值是标记ID的向量
std::map<std::string, std::vector<gpt_vocab::id>> extract_tests_from_file(const std::string & fpath_test){
    // 如果测试文件路径为空，则输出错误信息并返回空映射
    if (fpath_test.empty()){
        fprintf(stderr, "%s : No test file found.\n", __func__);
        return std::map<std::string, std::vector<gpt_vocab::id>>();
    }

    std::map<std::string, std::vector<gpt_vocab::id>> tests;

    // 打开测试文件并读取内容
    auto fin = std::ifstream(fpath_test, std::ios_base::in);
    const char * delimeter = " => ";
    const char del_tok = ',';
    std::string line;
    // 逐行读取文件内容
    while (std::getline(fin, line)) {
        // 查找分隔符位置
        size_t delimiterPos = line.find(delimeter);
        if (delimiterPos != std::string::npos) {
            // 提取文本和标记字符串
            std::string text = line.substr(0, delimiterPos);
            std::string s_tokens = line.substr(delimiterPos + std::strlen(delimeter));
            // 将文本和标记字符串转换为标记ID的向量，并存储到映射中
            tests[text] = parse_tokens_from_string(s_tokens, del_tok);
        }
    }
    return tests;
}

// 测试 GPT 分词器
void test_gpt_tokenizer(gpt_vocab & vocab, const std::string & fpath_test){
    // 从文件中提取测试数据
    std::map<std::string, std::vector<gpt_vocab::id>> tests = extract_tests_from_file(fpath_test);

    size_t n_fails = 0;
    // 遍历测试集中的每个测试用例
    for (const auto & test : tests) {
        // 使用给定词汇表对测试用例进行分词
        std::vector<gpt_vocab::id> tokens = gpt_tokenize(vocab, test.first);

        // 检查分词结果是否与预期结果不同
        if (tokens != test.second){
            n_fails++;

            // 打印失败的测试用例信息
            fprintf(stderr, "%s : failed test: '%s'\n", __func__, test.first.c_str());
            fprintf(stderr, "%s : tokens in hf:   ", __func__);
            // 打印预期结果中的词汇
            for (const auto & t : test.second) {
                fprintf(stderr, "%s(%d), ", vocab.id_to_token[t].c_str(), t);
            }
            fprintf(stderr, "\n");
            fprintf(stderr, "%s : tokens in ggml: ", __func__);
            // 打印实际结果中的词汇
            for (const auto & t : tokens) {
                fprintf(stderr, "%s(%d), ", vocab.id_to_token[t].c_str(), t);
            }
            fprintf(stderr, "\n");
        }
    }

    // 打印测试结果中失败的测试用例数量
    fprintf(stderr, "%s : %zu tests failed out of %zu tests.\n", __func__, n_fails, tests.size());
}

// 初始化 GPT 词汇表，从给定文件名加载词汇表数据
bool gpt_vocab_init(const std::string & fname, gpt_vocab & vocab) {
    // 打印加载词汇表的信息
    printf("%s: loading vocab from '%s'\n", __func__, fname.c_str());

    // 解析 JSON 文件，将结果存储到词汇表的 token_to_id 字典中
    vocab.token_to_id = ::json_parse(fname);

    // 遍历 token_to_id 字典，构建 id_to_token 字典
    for (const auto & kv : vocab.token_to_id) {
        vocab.id_to_token[kv.second] = kv.first;
    }

    // 打印词汇表的大小
    printf("%s: vocab size = %d\n", __func__, (int) vocab.token_to_id.size());

    // 打印词汇表内容
    //for (auto kv : vocab.token_to_id) {
    //    printf("'%s' -> %d\n", kv.first.data(), kv.second);
    //}

    return true;
}

// 从 logits 中采样出 top_k 个概率最高的 token
gpt_vocab::id gpt_sample_top_k_top_p(
        const gpt_vocab & vocab,
        const float * logits,
        int    top_k,
        double top_p,
        double temp,
        std::mt19937 & rng) {
    // 获取词汇表大小
    int n_logits = vocab.id_to_token.size();

    // 创建存储 logits 和 id 的 pair 的向量
    std::vector<std::pair<double, gpt_vocab::id>> logits_id;
    logits_id.reserve(n_logits);

    {
        // 计算 logits 的缩放比例
        const double scale = 1.0/temp;
        // 将 logits 缩放后存入 logits_id 中
        for (int i = 0; i < n_logits; ++i) {
            logits_id.push_back(std::make_pair(logits[i]*scale, i));
        }
    }

    // 找出概率最高的 top_k 个 token
    std::partial_sort(
            logits_id.begin(),
            logits_id.begin() + top_k, logits_id.end(),
            [](const std::pair<double, gpt_vocab::id> & a, const std::pair<double, gpt_vocab::id> & b) {
        return a.first > b.first;
    });

    // 裁剪 logits_id 到 top_k 大小
    logits_id.resize(top_k);

    // 计算最大的 logits 值
    double maxl = -INFINITY;
    for (const auto & kv : logits_id) {
        maxl = std::max(maxl, kv.first);
    }

    // 计算 top_k token 的概率
    std::vector<double> probs;
    probs.reserve(logits_id.size());

    double sum = 0.0;
    for (const auto & kv : logits_id) {
        double p = exp(kv.first - maxl);
        probs.push_back(p);
        sum += p;
    }

    // 归一化概率
    for (auto & p : probs) {
        p /= sum;
    }
    // 如果 top_p 小于 1.0，则执行以下操作
    if (top_p < 1.0f) {
        // 初始化累积概率为 0.0
        double cumsum = 0.0f;
        // 遍历前 top_k 个概率值
        for (int i = 0; i < top_k; i++) {
            // 累积概率值
            cumsum += probs[i];
            // 如果累积概率值大于等于 top_p，则更新 top_k，并调整概率和标签数组大小
            if (cumsum >= top_p) {
                top_k = i + 1;
                probs.resize(top_k);
                logits_id.resize(top_k);
                break;
            }
        }

        // 计算概率值的归一化系数
        cumsum = 1.0 / cumsum;
        // 对概率值进行归一化
        for (int i = 0; i < (int) probs.size(); i++) {
            probs[i] *= cumsum;
        }
    }

    // 打印概率值和对应标签（注释掉的代码）
    //printf("\n");
    //for (int i = 0; i < (int) probs.size(); i++) {
    //    printf("%d: '%s' %f\n", i, vocab.id_to_token.at(logits_id[i].second).c_str(), probs[i]);
    //}
    //exit(0);

    // 创建离散分布对象，以概率数组为参数
    std::discrete_distribution<> dist(probs.begin(), probs.end());
    // 从分布中采样一个索引
    int idx = dist(rng);

    // 返回采样得到的标签
    return logits_id[idx].second;
}
// 闭合之前的函数定义

gpt_vocab::id gpt_sample_top_k_top_p_repeat(
        const gpt_vocab & vocab,
        const float * logits,
        const int32_t * last_n_tokens_data,
        size_t last_n_tokens_data_size,
        int    top_k,
        double top_p,
        double temp,
        int repeat_last_n,
        float repeat_penalty,
        std::mt19937 & rng) {
// 定义一个函数，用于从 logits 中采样生成一个 token

    int n_logits = vocab.id_to_token.size();
    // 获取 logits 的长度

    const auto * plogits = logits;
    // 指向 logits 的指针

    const auto last_n_tokens = std::vector<int32_t>(last_n_tokens_data, last_n_tokens_data + last_n_tokens_data_size);
    // 将 last_n_tokens_data 转换为 vector 存储

    if (temp <= 0) {
        // 如果温度小于等于0，则直接选择具有最高 logit 的 token
        float max_logit = plogits[0];
        gpt_vocab::id max_id = 0;

        for (int i = 1; i < n_logits; ++i) {
            if (plogits[i] > max_logit) {
                max_logit = plogits[i];
                max_id = i;
            }
        }
        return max_id;
    }

    std::vector<std::pair<double, gpt_vocab::id>> logits_id;
    logits_id.reserve(n_logits);
    // 创建一个 vector 用于存储 logit 和对应的 token id

    {
        const float scale = 1.0f/temp;
        // 计算缩放比例
        for (int i = 0; i < n_logits; ++i) {
            // 遍历 logits
            if (repeat_last_n > 0 && std::find(last_n_tokens.end()-repeat_last_n, last_n_tokens.end(), i) != last_n_tokens.end()) {
                // 如果重复的 token 数大于0且当前 token 在最近的 last_n_tokens 中
                if (plogits[i] < 0.0f) {
                    // 如果 logit 小于0，则乘以重复惩罚以减少前一个 token 的概率
                    logits_id.push_back(std::make_pair(plogits[i]*scale*repeat_penalty, i));
                } else {
                    logits_id.push_back(std::make_pair(plogits[i]*scale/repeat_penalty, i));
                }
            } else {
                logits_id.push_back(std::make_pair(plogits[i]*scale, i));
            }
        }
    }
    // 根据温度和重复惩罚计算 logit，并存储到 logits_id 中

    // find the top K tokens
    // 找到前 K 个 token
    // 对 logits_id 中的元素进行部分排序，保留前 top_k 个元素，按照第一个元素（double类型）从大到小排序
    std::partial_sort(
            logits_id.begin(),
            logits_id.begin() + top_k, logits_id.end(),
            [](const std::pair<double, gpt_vocab::id> & a, const std::pair<double, gpt_vocab::id> & b) {
        return a.first > b.first;
    });

    // 调整 logits_id 的大小为 top_k
    logits_id.resize(top_k);

    // 找出 logits_id 中第一个元素的最大值
    double maxl = -INFINITY;
    for (const auto & kv : logits_id) {
        maxl = std::max(maxl, kv.first);
    }

    // 计算 top K 个 token 的概率
    std::vector<double> probs;
    probs.reserve(logits_id.size());

    double sum = 0.0;
    for (const auto & kv : logits_id) {
        double p = exp(kv.first - maxl);
        probs.push_back(p);
        sum += p;
    }

    // 归一化概率
    for (auto & p : probs) {
        p /= sum;
    }

    // 如果 top_p 小于 1.0f，则根据 top_p 裁剪概率分布
    if (top_p < 1.0f) {
        double cumsum = 0.0f;
        for (int i = 0; i < top_k; i++) {
            cumsum += probs[i];
            if (cumsum >= top_p) {
                top_k = i + 1;
                probs.resize(top_k);
                logits_id.resize(top_k);
                break;
            }
        }

        cumsum = 1.0/cumsum;
        for (int i = 0; i < (int) probs.size(); i++) {
            probs[i] *= cumsum;
        }
    }
// 使用离散分布初始化一个随机数生成器，概率分布由probs数组提供
std::discrete_distribution<> dist(probs.begin(), probs.end());
// 从离散分布中生成一个随机索引
int idx = dist(rng);
// 返回根据生成的索引从logits_id中获取的字符串
return logits_id[idx].second;
}

// 检查给定的字符串是否符合WAV文件格式
bool is_wav_buffer(const std::string buf) {
    // 检查字符串长度是否满足最小要求，以及是否以"RIFF"和"WAVE"开头
    if (buf.size() < 12 || buf.substr(0, 4) != "RIFF" || buf.substr(8, 4) != "WAVE") {
        return false;
    }

    // 读取WAV文件的数据块大小，并检查是否与字符串长度匹配
    uint32_t chunk_size = *reinterpret_cast<const uint32_t*>(buf.data() + 4);
    if (chunk_size + 8 != buf.size()) {
        return false;
    }

    return true;
}

// 从文件或标准输入中读取WAV文件数据
bool read_wav(const std::string & fname, std::vector<float>& pcmf32, std::vector<std::vector<float>>& pcmf32s, bool stereo) {
    drwav wav;
    std::vector<uint8_t> wav_data; // 用于从标准输入中读取数据

    // 如果文件名为"-"，则从标准输入中读取数据
    if (fname == "-") {
        {
            uint8_t buf[1024];
            while (true)
            {
                const size_t n = fread(buf, 1, sizeof(buf), stdin);
                if (n == 0) {
                    break;
                }
                wav_data.insert(wav_data.end(), buf, buf + n);
            }
        }

        // 初始化WAV文件结构体，从标准输入中读取的数据
        if (drwav_init_memory(&wav, wav_data.data(), wav_data.size(), nullptr) == false) {
            fprintf(stderr, "error: failed to open WAV file from stdin\n");
            return false;
        }

        fprintf(stderr, "%s: read %zu bytes from stdin\n", __func__, wav_data.size());
    }
    // 如果文件内容符合WAV格式，则从内存中初始化WAV文件结构体
    else if (is_wav_buffer(fname)) {
        if (drwav_init_memory(&wav, fname.c_str(), fname.size(), nullptr) == false) {
            fprintf(stderr, "error: failed to open WAV file from fname buffer\n");
            return false;
        }
    }
    // 如果无法以 WAV 文件格式打开文件，则输出错误信息并返回 false
    else if (drwav_init_file(&wav, fname.c_str(), nullptr) == false) {
        fprintf(stderr, "error: failed to open '%s' as WAV file\n", fname.c_str());
        return false;
    }

    // 检查 WAV 文件的声道数是否为1或2，如果不是则输出错误信息并返回 false
    if (wav.channels != 1 && wav.channels != 2) {
        fprintf(stderr, "%s: WAV file '%s' must be mono or stereo\n", __func__, fname.c_str());
        return false;
    }

    // 如果需要立体声而文件声道数不为2，则输出错误信息并返回 false
    if (stereo && wav.channels != 2) {
        fprintf(stderr, "%s: WAV file '%s' must be stereo for diarization\n", __func__, fname.c_str());
        return false;
    }

    // 检查 WAV 文件的采样率是否为指定值，如果不是则输出错误信息并返回 false
    if (wav.sampleRate != COMMON_SAMPLE_RATE) {
        fprintf(stderr, "%s: WAV file '%s' must be %i kHz\n", __func__, fname.c_str(), COMMON_SAMPLE_RATE/1000);
        return false;
    }

    // 检查 WAV 文件的采样位数是否为16位，如果不是则输出错误信息并返回 false
    if (wav.bitsPerSample != 16) {
        fprintf(stderr, "%s: WAV file '%s' must be 16-bit\n", __func__, fname.c_str());
        return false;
    }

    // 计算需要读取的 PCM 帧数
    const uint64_t n = wav_data.empty() ? wav.totalPCMFrameCount : wav_data.size()/(wav.channels*wav.bitsPerSample/8);

    // 读取 PCM 数据到 int16_t 类型的向量
    std::vector<int16_t> pcm16;
    pcm16.resize(n*wav.channels);
    drwav_read_pcm_frames_s16(&wav, n, pcm16.data());
    drwav_uninit(&wav);

    // 将 PCM 数据转换为单声道、浮点数类型
    pcmf32.resize(n);
    if (wav.channels == 1) {
        for (uint64_t i = 0; i < n; i++) {
            pcmf32[i] = float(pcm16[i])/32768.0f;
        }
    } else {
        for (uint64_t i = 0; i < n; i++) {
            pcmf32[i] = float(pcm16[2*i] + pcm16[2*i + 1])/65536.0f;
        }
    }

    // 如果需要立体声，则将 PCM 数据转换为立体声、浮点数类型
    if (stereo) {
        pcmf32s.resize(2);

        pcmf32s[0].resize(n);
        pcmf32s[1].resize(n);
        for (uint64_t i = 0; i < n; i++) {
            pcmf32s[0][i] = float(pcm16[2*i])/32768.0f;
            pcmf32s[1][i] = float(pcm16[2*i + 1])/32768.0f;
        }
    }

    // 返回 true 表示成功读取并转换 PCM 数据
    return true;
}

// 高通滤波器函数，用于对数据进行高通滤波
void high_pass_filter(std::vector<float> & data, float cutoff, float sample_rate) {
    // 计算 RC 值
    const float rc = 1.0f / (2.0f * M_PI * cutoff);
    // 计算采样间隔
    const float dt = 1.0f / sample_rate;
    // 计算 alpha 值
    const float alpha = dt / (rc + dt);

    // 初始化 y 值
    float y = data[0];

    // 遍历数据进行高通滤波
    for (size_t i = 1; i < data.size(); i++) {
        y = alpha * (y + data[i] - data[i - 1]);
        data[i] = y;
    }
}

// 简单的语音活动检测函数
bool vad_simple(std::vector<float> & pcmf32, int sample_rate, int last_ms, float vad_thold, float freq_thold, bool verbose) {
    // 计算总样本数和最后一段样本数
    const int n_samples      = pcmf32.size();
    const int n_samples_last = (sample_rate * last_ms) / 1000;

    // 如果最后一段样本数大于等于总样本数，则返回 false
    if (n_samples_last >= n_samples) {
        // not enough samples - assume no speech
        return false;
    }

    // 如果频率阈值大于 0，则进行高通滤波
    if (freq_thold > 0.0f) {
        high_pass_filter(pcmf32, freq_thold, sample_rate);
    }

    // 初始化总能量和最后一段能量
    float energy_all  = 0.0f;
    float energy_last = 0.0f;

    // 计算总能量和最后一段能量
    for (int i = 0; i < n_samples; i++) {
        energy_all += fabsf(pcmf32[i]);

        if (i >= n_samples - n_samples_last) {
            energy_last += fabsf(pcmf32[i]);
        }
    }

    energy_all  /= n_samples;
    energy_last /= n_samples_last;

    // 如果 verbose 为 true，则输出能量信息
    if (verbose) {
        fprintf(stderr, "%s: energy_all: %f, energy_last: %f, vad_thold: %f, freq_thold: %f\n", __func__, energy_all, energy_last, vad_thold, freq_thold);
    }

    // 根据能量比较判断是否为语音活动
    if (energy_last > vad_thold*energy_all) {
        return false;
    }

    return true;
}

// 计算两个字符串的相似度
float similarity(const std::string & s0, const std::string & s1) {
    // 计算字符串长度
    const size_t len0 = s0.size() + 1;
    const size_t len1 = s1.size() + 1;

    // 初始化列向量和前一列向量
    std::vector<int> col(len1, 0);
    std::vector<int> prevCol(len1, 0);

    // 初始化第一列
    for (size_t i = 0; i < len1; i++) {
        prevCol[i] = i;
    }

    // 计算编辑距离
    for (size_t i = 0; i < len0; i++) {
        col[0] = i;
        for (size_t j = 1; j < len1; j++) {
            col[j] = std::min(std::min(1 + col[j - 1], 1 + prevCol[j]), prevCol[j - 1] + (i > 0 && s0[i - 1] == s1[j - 1] ? 0 : 1));
        }
        col.swap(prevCol);
    }
    # 定义一个常量 dist，表示前一列最后一个元素的值
    const float dist = prevCol[len1 - 1];

    # 返回两个字符串的相似度，使用编辑距离计算，值范围在 0 到 1 之间
    return 1.0f - (dist / std::max(s0.size(), s1.size()));
// 解析命令行参数，填充 sam_params 结构体
bool sam_params_parse(int argc, char ** argv, sam_params & params) {
    // 遍历命令行参数
    for (int i = 1; i < argc; i++) {
        // 获取当前参数
        std::string arg = argv[i];

        // 根据参数类型进行处理
        if (arg == "-s" || arg == "--seed") {
            // 设置随机数种子
            params.seed = std::stoi(argv[++i]);
        } else if (arg == "-t" || arg == "--threads") {
            // 设置线程数
            params.n_threads = std::stoi(argv[++i]);
        } else if (arg == "-m" || arg == "--model") {
            // 设置模型路径
            params.model = argv[++i];
        } else if (arg == "-i" || arg == "--inp") {
            // 设置输入文件路径
            params.fname_inp = argv[++i];
        } else if (arg == "-o" || arg == "--out") {
            // 设置输出文件路径
            params.fname_out = argv[++i];
        } else if (arg == "-h" || arg == "--help") {
            // 打印帮助信息并退出程序
            sam_print_usage(argc, argv, params);
            exit(0);
        } else {
            // 打印错误信息并退出程序
            fprintf(stderr, "error: unknown argument: %s\n", arg.c_str());
            sam_print_usage(argc, argv, params);
            exit(0);
        }
    }

    return true;
}

// 打印程序的使用方法
void sam_print_usage(int /*argc*/, char ** argv, const sam_params & params) {
    // 打印程序名称和选项
    fprintf(stderr, "usage: %s [options]\n", argv[0]);
    fprintf(stderr, "\n");
    fprintf(stderr, "options:\n");
    // 打印各个选项的说明
    fprintf(stderr, "  -h, --help            show this help message and exit\n");
    fprintf(stderr, "  -s SEED, --seed SEED  RNG seed (default: -1)\n");
    fprintf(stderr, "  -t N, --threads N     number of threads to use during computation (default: %d)\n", params.n_threads);
    fprintf(stderr, "  -m FNAME, --model FNAME\n");
    fprintf(stderr, "                        model path (default: %s)\n", params.model.c_str());
    fprintf(stderr, "  -i FNAME, --inp FNAME\n");
    fprintf(stderr, "                        input file (default: %s)\n", params.fname_inp.c_str());
    fprintf(stderr, "  -o FNAME, --out FNAME\n");
    fprintf(stderr, "                        output file (default: %s)\n", params.fname_out.c_str());
    fprintf(stderr, "\n");
}
```