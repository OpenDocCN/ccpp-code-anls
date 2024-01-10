# `ggml\examples\common.cpp`

```
#define _USE_MATH_DEFINES // for M_PI

#include "common.h"

// third-party utilities
// use your favorite implementations
#define DR_WAV_IMPLEMENTATION
#include "dr_wav.h"

#include <cmath> // for mathematical functions
#include <cstring> // for C-style string manipulation functions
#include <fstream> // for file input/output
#include <regex> // for regular expressions
#include <locale> // for localization
#include <codecvt> // for Unicode conversion
#include <sstream> // for string stream

#if defined(_MSC_VER)
#pragma warning(disable: 4244 4267) // possible loss of data
#endif

// Function to check if the next argument exists
std::string get_next_arg(int& i, int argc, char** argv, const std::string& flag, gpt_params& params) {
    if (i + 1 < argc && argv[i + 1][0] != '-') { // check if the next argument exists and is not a flag
        return argv[++i]; // return the next argument
    } else {
        fprintf(stderr, "error: %s requires one argument.\n", flag.c_str()); // print error message
        gpt_print_usage(argc, argv, params); // call the function to print usage
        exit(0); // exit the program
    }
}

bool gpt_params_parse(int argc, char ** argv, gpt_params & params) {
    // implementation of parsing function goes here
    return true; // return true after parsing
}

void gpt_print_usage(int /*argc*/, char ** argv, const gpt_params & params) {
    fprintf(stderr, "usage: %s [options]\n", argv[0]); // print program usage
    fprintf(stderr, "\n");
    fprintf(stderr, "options:\n");
    fprintf(stderr, "  -h, --help            show this help message and exit\n"); // print help option
    fprintf(stderr, "  -s SEED, --seed SEED  RNG seed (default: -1)\n"); // print seed option
    fprintf(stderr, "  -t N, --threads N     number of threads to use during computation (default: %d)\n", params.n_threads); // print threads option
    fprintf(stderr, "  -p PROMPT, --prompt PROMPT\n"); // print prompt option
    fprintf(stderr, "                        prompt to start generation with (default: random)\n");
    fprintf(stderr, "  -f FNAME, --file FNAME\n"); // print file option
    fprintf(stderr, "                        load prompt from a file\n");
    fprintf(stderr, "  -tt TOKEN_TEST, --token_test TOKEN_TEST\n"); // print token test option
    fprintf(stderr, "                        test tokenization\n");
    fprintf(stderr, "  -n N, --n_predict N   number of tokens to predict (default: %d)\n", params.n_predict); // print number of tokens to predict option
    fprintf(stderr, "  --top_k N             top-k sampling (default: %d)\n", params.top_k); // print top-k sampling option
}
    # 打印 top-p 抽样的默认值
    fprintf(stderr, "  --top_p N             top-p sampling (default: %.1f)\n", params.top_p);
    # 打印温度的默认值
    fprintf(stderr, "  --temp N              temperature (default: %.1f)\n", params.temp);
    # 打印考虑惩罚的最后 n 个标记的默认值
    fprintf(stderr, "  --repeat-last-n N     last n tokens to consider for penalize (default: %d, 0 = disabled)\n", params.repeat_last_n);
    # 打印重复标记序列的惩罚值的默认值
    fprintf(stderr, "  --repeat-penalty N    penalize repeat sequence of tokens (default: %.2f, 1.0 = disabled)\n", (double)params.repeat_penalty);
    # 打印提示处理的批处理大小的默认值
    fprintf(stderr, "  -b N, --batch_size N  batch size for prompt processing (default: %d)\n", params.n_batch);
    # 打印上下文/KV缓存大小的默认值
    fprintf(stderr, "  -c N, --context N     context / KV cache size (default: %d)\n", params.n_ctx);
    # 打印是否在生成过程中忽略 EOS 标记的信息
    fprintf(stderr, "  --ignore-eos          ignore EOS token during generation\n");
    # 打印在支持的模型上要卸载到 GPU 的层数的默认值
    fprintf(stderr, "  -ngl N, --gpu-layers N  number of layers to offload to GPU on supported models (default: %d)\n", params.n_gpu_layers);
    # 打印模型路径的默认值
    fprintf(stderr, "  -m FNAME, --model FNAME\n");
    fprintf(stderr, "                        model path (default: %s)\n", params.model.c_str());
    # 打印空行
    fprintf(stderr, "\n");
}

// 生成随机的提示语
std::string gpt_random_prompt(std::mt19937 & rng) {
    // 生成一个0到9的随机数
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

// 去除字符串两端的空白字符
std::string trim(const std::string & s) {
    std::regex e("^\\s+|\\s+$");
    return std::regex_replace(s, e, "");
}

// 替换字符串中的指定子串
std::string replace(const std::string & s, const std::string & from, const std::string & to) {
    std::string result = s;
    size_t pos = 0;
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

// 将字符串转换为宽字符
std::wstring convert_to_wstring(const std::string & input) {
    std::wstring_convert<std::codecvt_utf8<wchar_t>> converter;
    return converter.from_bytes(input);
}

// 将字符串拆分为单词并存储到向量中
void gpt_split_words(std::string str, std::vector<std::string>& words) {
    # 定义匹配模式，包括缩写、单词、数字、非空白字符等
    const std::string pattern = R"('s|'t|'re|'ve|'m|'ll|'d| ?[[:alpha:]]+| ?[[:digit:]]+| ?[^\s[:alpha:][:digit:]]+|\s+(?!\S)|\s+)";
    # 使用正则表达式模式创建正则表达式对象
    const std::regex re(pattern);
    # 定义一个匹配结果对象
    std::smatch m;

    # 在字符串中查找匹配正则表达式的内容
    while (std::regex_search(str, m, re)) {
        # 遍历匹配结果，将匹配到的内容添加到单词列表中
        for (auto x : m) {
            words.push_back(x);
        }
        # 更新字符串，去除已经匹配的部分
        str = m.suffix();
    }
}

std::vector<gpt_vocab::id> gpt_tokenize(const gpt_vocab & vocab, const std::string & text) {
    std::vector<std::string> words;

    // 首先将文本分割成单词
    {
        // 将文本保存到字符串变量中
        std::string str = text;

        // 如果 special_tokens 向量不为空，则从中生成子模式
        if (!vocab.special_tokens.empty()) {
            // 创建转义正则表达式
            const std::regex escape(R"([\[\\\^\$\.\|\?\*\+\(\)\{\}])");
            std::string special_tokens_subpattern;
            // 遍历 special_tokens 向量，生成特殊标记的子模式
            for (const auto & token : vocab.special_tokens) {
                if (!special_tokens_subpattern.empty()) {
                    special_tokens_subpattern += "|";
                }
                special_tokens_subpattern += std::regex_replace(token, escape, R"(\$&)");
            }

            // 创建正则表达式对象
            std::regex re(special_tokens_subpattern);
            std::smatch m;
            // 使用特殊标记分割文本
            while (std::regex_search(str, m, re)) {
                // 将特殊标记之间的子字符串分割成单词
                gpt_split_words(m.prefix(), words);
                // 将匹配的特殊标记作为单词添加到 words 中
                for (auto x : m) {
                    words.push_back(x);
                }
                str = m.suffix();
            }
            // 处理剩余的不包含特殊标记的文本
        }

        // 将剩余的文本分割成单词
        gpt_split_words(str, words);
    }

    // 找到每个单词形成的最长标记：
    std::vector<gpt_vocab::id> tokens;
    // 遍历输入的单词列表
    for (const auto & word : words) {
        // 遍历单词的每个字符
        for (int i = 0; i < (int) word.size(); ){
            // 从单词的当前位置开始，向后遍历字符
            for (int j = word.size() - 1; j >= i; j--){
                // 获取当前位置到 j 的子串
                auto cand = word.substr(i, j-i+1);
                // 在词汇表中查找子串
                auto it = vocab.token_to_id.find(cand);
                // 如果子串在词汇表中
                if (it != vocab.token_to_id.end()){ // word.substr(i, j-i+1) in vocab
                    // 将子串对应的 ID 添加到 tokens 中
                    tokens.push_back(it->second);
                    // 更新 i 的值为 j+1，准备查找下一个子串
                    i = j + 1;
                    // 跳出当前循环，继续下一个单词的处理
                    break;
                }
                // 如果子串不在词汇表中
                else if (j == i){ // word.substr(i, 1) has no matching
                    // 输出错误信息，指出未知的单词
                    fprintf(stderr, "%s: unknown token '%s'\n", __func__, word.substr(i, 1).data());
                    // 更新 i 的值，准备查找下一个子串
                    i++;
                }
            }
        }
    }

    // 返回 tokens 列表
    return tokens;
// 从输入字符串中解析出标记，并以向量形式返回
std::vector<gpt_vocab::id> parse_tokens_from_string(const std::string& input, char delimiter) {
    // 创建字符串流对象
    std::stringstream ss(input);
    // 用于存储解析出的标记
    std::vector<gpt_vocab::id> output;
    // 临时存储解析出的标记
    std::string token;

    // 从字符串流中逐行读取数据，以指定的分隔符分割，将解析出的标记存入output向量
    while (std::getline(ss, token, delimiter)) {
        output.push_back(std::stoi(token));
    }

    return output;
}

// 从文件中提取测试数据，返回以字符串为键，以标记向量为值的映射
std::map<std::string, std::vector<gpt_vocab::id>> extract_tests_from_file(const std::string & fpath_test){
    // 如果测试文件路径为空，则输出错误信息并返回空映射
    if (fpath_test.empty()){
        fprintf(stderr, "%s : No test file found.\n", __func__);
        return std::map<std::string, std::vector<gpt_vocab::id>>();
    }

    // 存储测试数据的映射
    std::map<std::string, std::vector<gpt_vocab::id>> tests;

    // 打开测试文件
    auto fin = std::ifstream(fpath_test, std::ios_base::in);
    // 分隔符字符串
    const char * delimeter = " => ";
    // 分隔符字符
    const char del_tok = ',';
    // 临时存储读取的一行数据
    std::string line;
    // 逐行读取文件内容
    while (std::getline(fin, line)) {
        // 查找分隔符的位置
        size_t delimiterPos = line.find(delimeter);
        // 如果找到分隔符
        if (delimiterPos != std::string::npos) {
            // 提取文本部分
            std::string text = line.substr(0, delimiterPos);
            // 提取标记部分，并调用parse_tokens_from_string函数解析
            std::string s_tokens = line.substr(delimiterPos + std::strlen(delimeter));
            tests[text] = parse_tokens_from_string(s_tokens, del_tok);
        }
    }
    return tests;
}

// 测试GPT标记器
void test_gpt_tokenizer(gpt_vocab & vocab, const std::string & fpath_test){
    // 从文件中提取测试数据
    std::map<std::string, std::vector<gpt_vocab::id>> tests = extract_tests_from_file(fpath_test);

    // 记录测试失败次数
    size_t n_fails = 0;
}
    // 遍历测试集中的每个测试用例
    for (const auto & test : tests) {
        // 对测试用例的输入进行分词，得到 token 列表
        std::vector<gpt_vocab::id> tokens = gpt_tokenize(vocab, test.first);

        // 检查分词结果是否与预期结果相同
        if (tokens != test.second){
            // 如果不同，记录失败的测试用例数量
            n_fails++;

            // 打印出失败的测试用例
            fprintf(stderr, "%s : failed test: '%s'\n", __func__, test.first.c_str());
            fprintf(stderr, "%s : tokens in hf:   ", __func__);
            // 打印预期结果中的 token
            for (const auto & t : test.second) {
                fprintf(stderr, "%s(%d), ", vocab.id_to_token[t].c_str(), t);
            }
            fprintf(stderr, "\n");
            fprintf(stderr, "%s : tokens in ggml: ", __func__);
            // 打印实际结果中的 token
            for (const auto & t : tokens) {
                fprintf(stderr, "%s(%d), ", vocab.id_to_token[t].c_str(), t);
            }
            fprintf(stderr, "\n");
        }
    }

    // 打印出测试结果中失败的测试用例数量
    fprintf(stderr, "%s : %zu tests failed out of %zu tests.\n", __func__, n_fails, tests.size());
// 初始化词汇表，从给定文件名加载词汇表
bool gpt_vocab_init(const std::string & fname, gpt_vocab & vocab) {
    // 打印加载词汇表的信息
    printf("%s: loading vocab from '%s'\n", __func__, fname.c_str());

    // 从文件中解析 JSON 数据，存储到词汇表的 token_to_id 中
    vocab.token_to_id = ::json_parse(fname);

    // 遍历 token_to_id，构建 id_to_token
    for (const auto & kv : vocab.token_to_id) {
        vocab.id_to_token[kv.second] = kv.first;
    }

    // 打印词汇表的大小
    printf("%s: vocab size = %d\n", __func__, (int) vocab.token_to_id.size());

    // 返回初始化成功
    return true;
}

// 从 logits 中采样出 top_k top_p 对应的词汇 id
gpt_vocab::id gpt_sample_top_k_top_p(
        const gpt_vocab & vocab,
        const float * logits,
        int    top_k,
        double top_p,
        double temp,
        std::mt19937 & rng) {
    // 获取词汇表大小
    int n_logits = vocab.id_to_token.size();

    // 创建存储 logits 和 id 的向量
    std::vector<std::pair<double, gpt_vocab::id>> logits_id;
    logits_id.reserve(n_logits);

    {
        // 计算缩放因子
        const double scale = 1.0/temp;
        // 将 logits 缩放后存储到 logits_id 中
        for (int i = 0; i < n_logits; ++i) {
            logits_id.push_back(std::make_pair(logits[i]*scale, i));
        }
    }

    // 找出 top_k 个最大的 token
    std::partial_sort(
            logits_id.begin(),
            logits_id.begin() + top_k, logits_id.end(),
            [](const std::pair<double, gpt_vocab::id> & a, const std::pair<double, gpt_vocab::id> & b) {
        return a.first > b.first;
    });

    // 调整 logits_id 的大小为 top_k
    logits_id.resize(top_k);

    // 计算最大的 logit
    double maxl = -INFINITY;
    for (const auto & kv : logits_id) {
        maxl = std::max(maxl, kv.first);
    }

    // 计算 top_k tokens 的概率
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
}
    // 如果给定的 top_p 值小于 1.0，则执行以下操作
    if (top_p < 1.0f) {
        // 初始化累积概率为 0.0
        double cumsum = 0.0f;
        // 遍历前 top_k 个概率值，计算累积概率
        for (int i = 0; i < top_k; i++) {
            cumsum += probs[i];
            // 如果累积概率大于等于 top_p，则更新 top_k 值，并调整概率和 logits_id 的大小
            if (cumsum >= top_p) {
                top_k = i + 1;
                probs.resize(top_k);
                logits_id.resize(top_k);
                break;
            }
        }

        // 计算概率的倒数，用于归一化
        cumsum = 1.0/cumsum;
        // 对概率进行归一化处理
        for (int i = 0; i < (int) probs.size(); i++) {
            probs[i] *= cumsum;
        }
    }

    // 下面的代码被注释掉，不会执行
    //printf("\n");
    //for (int i = 0; i < (int) probs.size(); i++) {
    //    printf("%d: '%s' %f\n", i, vocab.id_to_token.at(logits_id[i].second).c_str(), probs[i]);
    //}
    //exit(0);

    // 使用 probs 中的概率值创建离散分布对象
    std::discrete_distribution<> dist(probs.begin(), probs.end());
    // 从离散分布中采样一个索引值
    int idx = dist(rng);

    // 返回对应索引的 logits_id 中的值
    return logits_id[idx].second;
    // 选择概率最高的 top_k 个 token，考虑 top_p 和温度因子
    gpt_vocab::id gpt_sample_top_k_top_p_repeat(
        const gpt_vocab & vocab,  // 词汇表
        const float * logits,  // 模型输出的 logits
        const int32_t * last_n_tokens_data,  // 最近的 n 个 token
        size_t last_n_tokens_data_size,  // 最近的 n 个 token 的大小
        int    top_k,  // 选择概率最高的 top_k 个 token
        double top_p,  // top_p 参数
        double temp,  // 温度因子
        int repeat_last_n,  // 重复的最近 n 个 token
        float repeat_penalty,  // 重复惩罚
        std::mt19937 & rng) {  // 随机数生成器

    int n_logits = vocab.id_to_token.size();  // logits 的数量

    const auto * plogits = logits;  // 指向 logits 的指针

    const auto last_n_tokens = std::vector<int32_t>(last_n_tokens_data, last_n_tokens_data + last_n_tokens_data_size);  // 最近的 n 个 token

    if (temp <= 0) {
        // 如果温度因子小于等于 0，则直接选择概率最高的 token
        float max_logit = plogits[0];  // 最高 logit
        gpt_vocab::id max_id = 0;  // 最高 logit 对应的 token id

        for (int i = 1; i < n_logits; ++i) {
            if (plogits[i] > max_logit) {
                max_logit = plogits[i];
                max_id = i;
            }
        }
        return max_id;  // 返回最高 logit 对应的 token id
    }

    std::vector<std::pair<double, gpt_vocab::id>> logits_id;  // 存储 token 的 logit 和对应的 token id
    logits_id.reserve(n_logits);  // 预先分配空间

    {
        const float scale = 1.0f/temp;  // 缩放因子
        for (int i = 0; i < n_logits; ++i) {
            // 根据重复惩罚和重复的最近 n 个 token，调整 token 的 logit
            if (repeat_last_n > 0 && std::find(last_n_tokens.end()-repeat_last_n, last_n_tokens.end(), i) != last_n_tokens.end()) {
                // 如果分数 < 0，则需要乘以重复惩罚以减少先前 token 的概率
                if (plogits[i] < 0.0f) {
                    logits_id.push_back(std::make_pair(plogits[i]*scale*repeat_penalty, i));
                } else {
                    logits_id.push_back(std::make_pair(plogits[i]*scale/repeat_penalty, i));
                }
            } else {
                logits_id.push_back(std::make_pair(plogits[i]*scale, i));
            }
        }
    }

    // 找到概率最高的 top_k 个 token
    // 对logits_id进行部分排序，只保留前top_k个元素，按照元素的第一个值（double类型）从大到小排序
    std::partial_sort(
            logits_id.begin(),
            logits_id.begin() + top_k, logits_id.end(),
            [](const std::pair<double, gpt_vocab::id> & a, const std::pair<double, gpt_vocab::id> & b) {
        return a.first > b.first;
    });

    // 调整logits_id的大小为top_k
    logits_id.resize(top_k);

    // 找出logits_id中第一个值的最大值
    double maxl = -INFINITY;
    for (const auto & kv : logits_id) {
        maxl = std::max(maxl, kv.first);
    }

    // 为top K个token计算概率
    std::vector<double> probs;
    probs.reserve(logits_id.size());

    double sum = 0.0;
    for (const auto & kv : logits_id) {
        double p = exp(kv.first - maxl);
        probs.push_back(p);
        sum += p;
    }

    // 对概率进行归一化
    for (auto & p : probs) {
        p /= sum;
    }

    // 如果top_p小于1.0，根据top_p对概率进行调整
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
    // 使用离散分布初始化随机数生成器，概率由probs数组提供
    std::discrete_distribution<> dist(probs.begin(), probs.end());
    // 从离散分布中生成一个随机数作为索引
    int idx = dist(rng);
    // 返回根据索引找到的logits_id中的第二个元素
    return logits_id[idx].second;
}

// 读取WAV文件内容并存储为32位浮点数的向量，如果是立体声则存储为二维向量
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
                // 将读取的数据存储到wav_data中
                wav_data.insert(wav_data.end(), buf, buf + n);
            }
        }

        // 从内存中初始化WAV文件
        if (drwav_init_memory(&wav, wav_data.data(), wav_data.size(), nullptr) == false) {
            fprintf(stderr, "error: failed to open WAV file from stdin\n");
            return false;
        }

        fprintf(stderr, "%s: read %zu bytes from stdin\n", __func__, wav_data.size());
    }
    // 否则从文件中读取WAV文件
    else if (drwav_init_file(&wav, fname.c_str(), nullptr) == false) {
        fprintf(stderr, "error: failed to open '%s' as WAV file\n", fname.c_str());
        return false;
    }

    // 检查WAV文件的通道数是否为1或2
    if (wav.channels != 1 && wav.channels != 2) {
        fprintf(stderr, "%s: WAV file '%s' must be mono or stereo\n", __func__, fname.c_str());
        return false;
    }

    // 如果需要立体声，但WAV文件的通道数不是2，则报错
    if (stereo && wav.channels != 2) {
        fprintf(stderr, "%s: WAV file '%s' must be stereo for diarization\n", __func__, fname.c_str());
        return false;
    }

    // 检查WAV文件的采样率是否为指定的采样率
    if (wav.sampleRate != COMMON_SAMPLE_RATE) {
        fprintf(stderr, "%s: WAV file '%s' must be %i kHz\n", __func__, fname.c_str(), COMMON_SAMPLE_RATE/1000);
        return false;
    }
    # 如果 WAV 文件的每个样本不是16位，则输出错误信息并返回false
    if (wav.bitsPerSample != 16) {
        fprintf(stderr, "%s: WAV file '%s' must be 16-bit\n", __func__, fname.c_str());
        return false;
    }

    # 计算样本数，如果已经有WAV数据，则使用其样本数，否则使用WAV对象的总样本数
    const uint64_t n = wav_data.empty() ? wav.totalPCMFrameCount : wav_data.size()/(wav.channels*wav.bitsPerSample/8);

    # 创建一个存储16位PCM数据的向量，并设置其大小为样本数乘以通道数
    std::vector<int16_t> pcm16;
    pcm16.resize(n*wav.channels);
    # 从WAV对象中读取n个16位PCM帧到pcm16向量中
    drwav_read_pcm_frames_s16(&wav, n, pcm16.data());
    # 关闭WAV对象
    drwav_uninit(&wav);

    # 转换为单声道，浮点数
    pcmf32.resize(n);
    if (wav.channels == 1) {
        # 如果是单声道，则将每个16位PCM样本转换为浮点数并存储在pcmf32中
        for (uint64_t i = 0; i < n; i++) {
            pcmf32[i] = float(pcm16[i])/32768.0f;
        }
    } else {
        # 如果是双声道，则将每一对16位PCM样本转换为浮点数并存储在pcmf32中
        for (uint64_t i = 0; i < n; i++) {
            pcmf32[i] = float(pcm16[2*i] + pcm16[2*i + 1])/65536.0f;
        }
    }

    # 如果是立体声
    if (stereo) {
        # 转换为立体声，浮点数
        pcmf32s.resize(2);

        # 创建两个向量用于存储立体声数据
        pcmf32s[0].resize(n);
        pcmf32s[1].resize(n);
        # 将每一对16位PCM样本转换为浮点数并存储在pcmf32s中
        for (uint64_t i = 0; i < n; i++) {
            pcmf32s[0][i] = float(pcm16[2*i])/32768.0f;
            pcmf32s[1][i] = float(pcm16[2*i + 1])/32768.0f;
        }
    }

    # 返回true
    return true;
}

// 高通滤波器函数，用于对数据进行高通滤波处理
void high_pass_filter(std::vector<float> & data, float cutoff, float sample_rate) {
    // 计算 RC 值
    const float rc = 1.0f / (2.0f * M_PI * cutoff);
    // 计算采样间隔
    const float dt = 1.0f / sample_rate;
    // 计算 alpha 值
    const float alpha = dt / (rc + dt);

    // 初始化 y 值
    float y = data[0];

    // 遍历数据进行滤波处理
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

    // 如果最后一段样本数大于等于总样本数，则返回无语音
    if (n_samples_last >= n_samples) {
        return false;
    }

    // 如果频率阈值大于 0，则进行高通滤波处理
    if (freq_thold > 0.0f) {
        high_pass_filter(pcmf32, freq_thold, sample_rate);
    }

    // 计算总能量和最后一段能量
    float energy_all  = 0.0f;
    float energy_last = 0.0f;

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

    // 根据能量比较判断是否为语音
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

    // 初始化列向量
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
    # 定义一个常量dist，表示前一列的最后一个元素
    const float dist = prevCol[len1 - 1];
    
    # 返回两个字符串的相似度，使用1减去距离除以两个字符串长度的最大值
    return 1.0f - (dist / std::max(s0.size(), s1.size()));
// 解析命令行参数，将参数值存储到 sam_params 结构体中
bool sam_params_parse(int argc, char ** argv, sam_params & params) {
    // 遍历命令行参数
    for (int i = 1; i < argc; i++) {
        // 获取当前参数值
        std::string arg = argv[i];

        // 判断参数类型并解析对应的值
        if (arg == "-s" || arg == "--seed") {
            // 解析并存储种子值
            params.seed = std::stoi(argv[++i]);
        } else if (arg == "-t" || arg == "--threads") {
            // 解析并存储线程数
            params.n_threads = std::stoi(argv[++i]);
        } else if (arg == "-m" || arg == "--model") {
            // 存储模型路径
            params.model = argv[++i];
        } else if (arg == "-i" || arg == "--inp") {
            // 存储输入文件路径
            params.fname_inp = argv[++i];
        } else if (arg == "-o" || arg == "--out") {
            // 存储输出文件路径
            params.fname_out = argv[++i];
        } else if (arg == "-h" || arg == "--help") {
            // 打印帮助信息并退出程序
            sam_print_usage(argc, argv, params);
            exit(0);
        } else {
            // 打印错误信息并帮助信息，然后退出程序
            fprintf(stderr, "error: unknown argument: %s\n", arg.c_str());
            sam_print_usage(argc, argv, params);
            exit(0);
        }
    }

    return true;
}

// 打印程序的使用方法
void sam_print_usage(int /*argc*/, char ** argv, const sam_params & params) {
    // 打印程序使用方法
    fprintf(stderr, "usage: %s [options]\n", argv[0]);
    fprintf(stderr, "\n");
    fprintf(stderr, "options:\n");
    // 打印各个参数的使用方法和默认值
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