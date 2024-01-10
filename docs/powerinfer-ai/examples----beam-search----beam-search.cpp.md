# `PowerInfer\examples\beam-search\beam-search.cpp`

```
// 包含自定义头文件 common.h 和 llama.h
#include "common.h"
#include "llama.h"

// 包含 C++ 标准库头文件
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

// 根据操作系统定义不同的头文件
#if defined (__unix__) || (defined (__APPLE__) && defined (__MACH__))
#include <signal.h>
#include <unistd.h>
#elif defined (_WIN32)
#define WIN32_LEAN_AND_MEAN
#ifndef NOMINMAX
#   define NOMINMAX
#endif
#include <windows.h>
#include <signal.h>
#endif

// 用于调试，打印出 beam tokens
struct ostream_beam_view {
    llama_context * ctx;
    llama_beam_view beam_view;
};

// 重载输出运算符，用于打印 beam tokens
static std::ostream & operator<<(std::ostream & os, const ostream_beam_view & obv) {
    os << "p(" << obv.beam_view.p << ") eob(" << std::boolalpha << obv.beam_view.eob << ") tokens(";
    for (size_t i = 0 ; i < obv.beam_view.n_tokens ; ++i) {
        os << llama_token_to_piece(obv.ctx, obv.beam_view.tokens[i]);
    }
    return os << ')';
}

// 存储在 beam_search_callback() 中需要返回的数据
struct beam_search_callback_data {
    llama_context * ctx;
    std::vector<llama_token> response;
};

// 判断是否到达 beam 结尾
static bool is_at_eob(const beam_search_callback_data & callback_data, const llama_token * tokens, size_t n_tokens) {
    return n_tokens && tokens[n_tokens-1] == llama_token_eos(llama_get_model(callback_data.ctx));
}

// 匹配类型 llama_beam_search_callback_fn_t 的函数
// 自定义回调函数示例，在每次 beam 长度增加时调用：
//  * 通过打印','和收敛的 beam tokens 数量来显示进度。
//  * 当所有 beam 收敛到一个共同的前缀时，它们将在 beams_state.beams[0] 中可用。
//    当满足停止条件时，也会调用此函数。
// Collect tokens into std::vector<llama_token> response which is pointed to by callback_data.
static void beam_search_callback(void * callback_data_ptr, llama_beams_state beams_state) {
    // 将回调数据指针转换为beam_search_callback_data类型的引用
    auto& callback_data = *static_cast<beam_search_callback_data*>(callback_data_ptr);
    // 标记需要的beam为EOS
    for (size_t i = 0 ; i < beams_state.n_beams ; ++i) {
        llama_beam_view& beam_view = beams_state.beam_views[i];
        if (!beam_view.eob && is_at_eob(callback_data, beam_view.tokens, beam_view.n_tokens)) {
            beam_view.eob = true;
        }
    }
    printf(",");  // 显示进度
    if (const size_t n = beams_state.common_prefix_length) {
        // 调整response的大小以容纳新的tokens
        callback_data.response.resize(callback_data.response.size() + n);
        assert(0u < beams_state.n_beams);
        const llama_token * tokens = beams_state.beam_views[0].tokens;
        // 将新的tokens复制到response中
        std::copy(tokens, tokens + n, callback_data.response.end() - n);
        printf("%zu", n);
    }
    fflush(stdout);
#if 1 // DEBUG: print current beams for this iteration
    // 调试信息：打印当前迭代的beam
    std::cout << "\n\nCurrent beams (last_call=" << beams_state.last_call << "):\n";
    for (size_t i = 0 ; i < beams_state.n_beams ; ++i) {
        std::cout << "beams["<<i<<"]: " << ostream_beam_view{callback_data.ctx,beams_state.beam_views[i]} << std::endl;
    }
#endif
}

int main(int argc, char ** argv)
{
    gpt_params params;
    //params.n_gpu_layers = 200;

    //---------------------------------
    // Print help :
    //---------------------------------

    // 如果参数不足或第一个参数为'-'，则打印使用说明并返回
    if ( argc < 2 || argv[1][0] == '-' )
    {
        printf( "Usage: %s MODEL_PATH [BEAM_WIDTH=2] [PROMPT]\n" , argv[0] );
        return 1 ;
    }

    //---------------------------------
    // Load parameters :
    //---------------------------------

    // 加载模型路径
    params.model = argv[1];

    // 设置beam的宽度
    params.n_beams = 2 < argc ? std::stoi(argv[2]) : 2;

    // 如果有第三个参数，则设置prompt
    if ( argc > 3 )
    {
        params.prompt = argv[3];
    }

    // 如果prompt为空
    {
        // 设置提示信息，包括请求和响应
        params.prompt = "### Request:\nHow many countries are there?\n\n### Response:\n";
    }

    //---------------------------------
    // 初始化LLM：
    //---------------------------------

    // 初始化LLM后端
    llama_backend_init(params.numa);

    // 初始化LLM模型和上下文
    llama_model * model;
    llama_context * ctx;

    std::tie(model, ctx) = llama_init_from_gpt_params( params );

    // 如果无法加载模型，则输出错误信息并返回
    if ( model == NULL )
    {
        fprintf( stderr , "%s: error: unable to load model\n" , __func__ );
        return 1;
    }

    //---------------------------------
    // 对提示进行标记化：
    //---------------------------------

    // 对提示进行标记化，返回标记列表
    std::vector<llama_token> tokens_list = llama_tokenize(ctx, params.prompt, true);

    // 设置最大上下文大小和最大标记列表大小
    const size_t max_context_size     = llama_n_ctx( ctx );
    const size_t max_tokens_list_size = max_context_size - 4 ;

    // 如果标记列表大小超过最大限制，则输出错误信息并返回
    if (tokens_list.size() > max_tokens_list_size)
    {
        fprintf( stderr , "%s: error: prompt too long (%zu tokens, max %zu)\n" ,
             __func__ , tokens_list.size() , max_tokens_list_size );
        return 1;
    }

    // 输出换行符
    fprintf( stderr, "\n\n" );

    // 打印提示中的标记
    for( auto id : tokens_list )
    {
        std::cout << llama_token_to_piece(ctx, id);
    }
    std::cout << std::flush;

    // 初始化过去标记数量
    int n_past = 0;

    // 解码提示并获取响应
    if (llama_decode(ctx, llama_batch_get_one(tokens_list.data(), tokens_list.size(), n_past, 0)))
    {
        fprintf(stderr, "%s : failed to eval prompt.\n" , __func__ );
        return 1;
    }
    n_past += tokens_list.size();

    // 设置beam search回调数据和参数
    beam_search_callback_data callback_data{ctx, {}};
    size_t const beam_width = static_cast<size_t>(params.n_beams);
    int const n_predict = 256;
    // 进行beam search
    llama_beam_search(ctx, beam_search_callback, &callback_data, beam_width, n_past, n_predict);

    // 输出换行符
    std::cout << "\n\n";
    // 打印响应中的标记
    for (llama_token const token_id : callback_data.response) {
        std::cout << llama_token_to_piece(ctx,token_id);
    }
    std::cout << std::endl;

    // 释放上下文和模型
    llama_free( ctx );
    llama_free_model( model );
    # 释放后端资源，清理内存
    llama_backend_free();
    # 返回 0，表示程序正常结束
    return 0;
# 闭合前面的函数定义
```