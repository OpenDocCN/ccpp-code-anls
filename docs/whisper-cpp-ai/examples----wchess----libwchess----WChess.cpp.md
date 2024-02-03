# `whisper.cpp\examples\wchess\libwchess\WChess.cpp`

```cpp
// 包含头文件 "WChess.h"
// 包含头文件 "Chessboard.h"
// 包含头文件 "grammar-parser.h"
// 包含头文件 "common.h"
// 包含头文件 <thread>
WChess::WChess(whisper_context * ctx,
        const whisper_full_params & wparams,
        callbacks cb,
        settings s)
        : m_ctx(ctx)
        , m_wparams(wparams)
        , m_cb(cb)
        , m_settings(s)
        , m_board(new Chessboard())
{}

// 默认析构函数
WChess::~WChess() = default;

// 设置移动
void WChess::set_move(const std::string& moves, float prob) const {
    // 如果回调函数 set_move 存在，则调用该函数
    if (m_cb.set_move) (*m_cb.set_move)(moves, prob);
}

// 设置语法
void WChess::set_grammar(const std::string& grammar) const {
    // 如果回调函数 set_grammar 存在，则调用该函数
    if (m_cb.set_grammar) (*m_cb.set_grammar)(grammar);
}

// 获取音频数据
bool WChess::get_audio(std::vector<float>& pcmf32) const {
    // 如果回调函数 get_audio 存在，则调用该函数并返回结果
    if (m_cb.get_audio) return (*m_cb.get_audio)(pcmf32);
    return false;
}

// 返回棋盘的字符串表示
std::string WChess::stringify_board() const {
    return m_board->stringifyBoard();
}

// 获取语法
std::string WChess::get_grammar() const {
    return m_board->grammar();
}

// 运行函数
void WChess::run() {
    bool have_prompt  = true;
    bool ask_prompt   = !have_prompt;

    float logprob_min  = 0.0f;
    float logprob_sum  = 0.0f;
    int n_tokens  = 0;
    std::vector<float> pcmf32_cur;
    std::vector<float> pcmf32_prompt;
    const std::string k_prompt = have_prompt ? "" : "rook to d4, f3";
    int64_t t_ms = 0;

    // 如果需要提示，则输出提示信息
    if (ask_prompt) {
        fprintf(stdout, "\n");
        fprintf(stdout, "%s: Say the following phrase: '%s%s%s'\n", __func__, "\033[1m", k_prompt.c_str(), "\033[0m");
        fprintf(stdout, "\n");
        ask_prompt = false;
    }
}

// 转录函数
std::string WChess::transcribe(
                const std::vector<float> & pcmf32,
                float & logprob_min,
                float & logprob_sum,
                int & n_tokens,
                int64_t & t_ms) {
    // 获取当前时间
    const auto t_start = std::chrono::high_resolution_clock::now();

    // 初始化变量
    logprob_min = 0.0f;
    logprob_sum = 0.0f;
    n_tokens    = 0;
    t_ms = 0;
}
    # 如果调用whisper_full函数返回非零值，则返回空字典
    if (whisper_full(m_ctx, m_wparams, pcmf32.data(), pcmf32.size()) != 0) {
        return {};
    }

    # 初始化一个空字符串用于存储结果
    std::string result;

    # 获取whisper_full返回的段数
    const int n_segments = whisper_full_n_segments(m_ctx);
    # 遍历每个段
    for (int i = 0; i < n_segments; ++i) {
        # 获取当前段的文本内容
        const char * text = whisper_full_get_segment_text(m_ctx, i);

        # 将文本内容添加到结果字符串中
        result += text;

        # 获取当前段的标记数
        const int n = whisper_full_n_tokens(m_ctx, i);
        # 遍历每个标记
        for (int j = 0; j < n; ++j) {
            # 获取当前标记的数据
            const auto token = whisper_full_get_token_data(m_ctx, i, j);

            # 如果标记的概率大于0，则返回空字典
            if(token.plog > 0.0f) return {};
            # 更新最小概率值
            logprob_min = std::min(logprob_min, token.plog);
            # 累加概率值
            logprob_sum += token.plog;
            # 增加标记数
            ++n_tokens;
        }
    }

    # 计算程序运行时间
    const auto t_end = std::chrono::high_resolution_clock::now();
    t_ms = std::chrono::duration_cast<std::chrono::milliseconds>(t_end - t_start).count();

    # 返回结果字符串
    return result;
# 闭合之前的代码块
```