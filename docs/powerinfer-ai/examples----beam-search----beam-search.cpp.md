# `PowerInfer\examples\beam-search\beam-search.cpp`

```
#include "common.h"
#include "llama.h"
// 引入自定义的头文件 common.h 和 llama.h

#include <cassert>
#include <cinttypes>
#include <cmath>
#include <cstdio>
#include <cstring>
#include <ctime>
#include <fstream>
#include <iostream>
#include <string>
#include <vector>
// 引入标准库头文件

#if defined (__unix__) || (defined (__APPLE__) && defined (__MACH__))
#include <signal.h>
#include <unistd.h>
// 如果是 UNIX 系统或者苹果系统，则引入信号处理和进程控制的头文件
#elif defined (_WIN32)
#define WIN32_LEAN_AND_MEAN
#ifndef NOMINMAX
// 如果是 Windows 系统，则定义 WIN32_LEAN_AND_MEAN，并且不定义 NOMINMAX
// 定义NOMINMAX，用于防止Windows.h中的宏定义干扰
#ifndef NOMINMAX
#include <windows.h>
#include <signal.h>
#endif

// 用于调试，打印出beam tokens的结构体
struct ostream_beam_view {
    llama_context * ctx; // 指向llama_context的指针
    llama_beam_view beam_view; // llama_beam_view类型的变量
};

// 重载操作符<<，用于打印ostream_beam_view对象
static std::ostream & operator<<(std::ostream & os, const ostream_beam_view & obv) {
    os << "p(" << obv.beam_view.p << ") eob(" << std::boolalpha << obv.beam_view.eob << ") tokens("; // 打印beam_view的p和eob属性
    for (size_t i = 0 ; i < obv.beam_view.n_tokens ; ++i) { // 遍历tokens数组
        os << llama_token_to_piece(obv.ctx, obv.beam_view.tokens[i]); // 调用llama_token_to_piece函数打印tokens数组中的元素
    }
    return os << ')'; // 返回打印结果
}
// 在beam_search_callback()函数中需要返回的任何内容都放在这里。
struct beam_search_callback_data {
    llama_context * ctx;  // 上下文对象指针
    std::vector<llama_token> response;  // 存储响应的标记向量
};

// 在这种情况下，结束标记（eob）等同于句子的结束标记（eos），但这并不总是相同的。
// 例如，eob可以由于最大标记长度、停用词等而被标记。
static bool is_at_eob(const beam_search_callback_data & callback_data, const llama_token * tokens, size_t n_tokens) {
    return n_tokens && tokens[n_tokens-1] == llama_token_eos(llama_get_model(callback_data.ctx));  // 检查是否到达了结束标记
}

// 匹配类型llama_beam_search_callback_fn_t的函数。
// 自定义回调示例在每次beam长度增加时被调用：
//  * 通过打印','后跟任何收敛的beam标记数量来显示进度。
//  * 当所有beam收敛到一个共同的前缀时，它们将在beams_state.beams[0]中可用。
//    当满足停止条件时，也会调用此函数。
//    将标记收集到由callback_data指向的std::vector<llama_token> response中。
static void beam_search_callback(void * callback_data_ptr, llama_beams_state beams_state) {
    auto& callback_data = *static_cast<beam_search_callback_data*>(callback_data_ptr);  // 获取回调数据对象的引用
    // 标记需要的情况下将光束标记为EOS（End of Sequence）。
    for (size_t i = 0 ; i < beams_state.n_beams ; ++i) {
        // 获取当前光束的视图
        llama_beam_view& beam_view = beams_state.beam_views[i];
        // 如果光束尚未结束，并且在结束位置，则将其标记为结束
        if (!beam_view.eob && is_at_eob(callback_data, beam_view.tokens, beam_view.n_tokens)) {
            beam_view.eob = true;
        }
    }
    printf(",");  // 显示进度
    // 如果存在公共前缀长度，则将其添加到回调数据的响应中
    if (const size_t n = beams_state.common_prefix_length) {
        callback_data.response.resize(callback_data.response.size() + n);
        assert(0u < beams_state.n_beams);
        const llama_token * tokens = beams_state.beam_views[0].tokens;
        std::copy(tokens, tokens + n, callback_data.response.end() - n);
        printf("%zu", n);
    }
    fflush(stdout);
#if 1 // DEBUG: 打印当前迭代的光束
    std::cout << "\n\nCurrent beams (last_call=" << beams_state.last_call << "):\n";
    for (size_t i = 0 ; i < beams_state.n_beams ; ++i) {
        std::cout << "beams["<<i<<"]: " << ostream_beam_view{callback_data.ctx,beams_state.beam_views[i]} << std::endl;
    }
    }
#endif
}

int main(int argc, char ** argv)
{
    gpt_params params;
    //params.n_gpu_layers = 200;  // 设置参数n_gpu_layers的值为200

    //---------------------------------
    // Print help :
    //---------------------------------

    if ( argc < 2 || argv[1][0] == '-' )  // 如果命令行参数小于2或者第一个参数的第一个字符是'-'，则执行以下代码
    {
        printf( "Usage: %s MODEL_PATH [BEAM_WIDTH=2] [PROMPT]\n" , argv[0] );  // 打印使用说明
        return 1 ;  // 返回1
    }

    //---------------------------------
```

    // 加载参数：
    //---------------------------------

    // 设置模型参数为命令行参数中的第一个参数
    params.model = argv[1];

    // 如果命令行参数数量大于2，则将第二个参数转换为整数并赋值给params.n_beams，否则默认为2
    params.n_beams = 2 < argc ? std::stoi(argv[2]) : 2;

    // 如果命令行参数数量大于3，则将第三个参数赋值给params.prompt
    if ( argc > 3 )
    {
        params.prompt = argv[3];
    }

    // 如果params.prompt为空，则设置默认的提示语句
    if ( params.prompt.empty() )
    {
        params.prompt = "### Request:\nHow many countries are there?\n\n### Response:\n";
    }

    //---------------------------------
    // 初始化LLM：
    //---------------------------------
# 初始化 LLAMA 后端
llama_backend_init(params.numa);

# 声明 LLAMA 模型和上下文
llama_model * model;
llama_context * ctx;

# 从 GPT 参数初始化 LLAMA 模型和上下文
std::tie(model, ctx) = llama_init_from_gpt_params( params );

# 如果模型为空，则打印错误信息并返回
if ( model == NULL )
{
    fprintf( stderr , "%s: error: unable to load model\n" , __func__ );
    return 1;
}

# 分词提示信息
std::vector<llama_token> tokens_list = llama_tokenize(ctx, params.prompt, true);
// 获取上下文的最大大小
const size_t max_context_size = llama_n_ctx(ctx);
// 计算最大的 tokens_list 大小
const size_t max_tokens_list_size = max_context_size - 4;

// 如果 tokens_list 的大小超过最大 tokens_list 大小
if (tokens_list.size() > max_tokens_list_size)
{
    // 打印错误信息并返回 1
    fprintf(stderr, "%s: error: prompt too long (%zu tokens, max %zu)\n",
        __func__, tokens_list.size(), max_tokens_list_size);
    return 1;
}

// 打印换行符
fprintf(stderr, "\n\n");

// 打印提示中的 tokens
for (auto id : tokens_list)
{
    std::cout << llama_token_to_piece(ctx, id);
}
// 刷新输出缓冲区
std::cout << std::flush;
    # 初始化过去的标记数量
    int n_past = 0;

    # 调用 llama_decode 函数，解码 tokens_list 中的数据，并将结果存储到 ctx 中
    if (llama_decode(ctx, llama_batch_get_one(tokens_list.data(), tokens_list.size(), n_past, 0)))
    {
        # 如果解码失败，打印错误信息并返回 1
        fprintf(stderr, "%s : failed to eval prompt.\n" , __func__ );
        return 1;
    }
    # 更新过去的标记数量
    n_past += tokens_list.size();

    # 初始化 beam_search_callback_data 结构体
    beam_search_callback_data callback_data{ctx, {}};
    # 设置 beam 宽度
    size_t const beam_width = static_cast<size_t>(params.n_beams);
    # 设置预测数量
    int const n_predict = 256;
    # 调用 llama_beam_search 函数，进行 beam search
    llama_beam_search(ctx, beam_search_callback, &callback_data, beam_width, n_past, n_predict);

    # 打印换行符
    std::cout << "\n\n";
    # 遍历 callback_data.response 中的 token_id，并打印对应的文本
    for (llama_token const token_id : callback_data.response) {
        std::cout << llama_token_to_piece(ctx,token_id);
    }
    # 打印换行符
    std::cout << std::endl;
    释放上下文资源
    llama_free( ctx );
    释放模型资源
    llama_free_model( model );
    释放后端资源
    llama_backend_free();
    返回成功状态码
    return 0;
}
```