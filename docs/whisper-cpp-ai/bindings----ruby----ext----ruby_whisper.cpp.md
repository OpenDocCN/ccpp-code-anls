# `whisper.cpp\bindings\ruby\ext\ruby_whisper.cpp`

```cpp
#ifdef __cplusplus
extern "C" {
#endif



// 如果是 C++ 环境，则使用 extern "C" 包裹代码



#define BOOL_PARAMS_SETTER(self, prop, value) \
  ruby_whisper_params *rwp; \
  Data_Get_Struct(self, ruby_whisper_params, rwp); \
  if (value == Qfalse || value == Qnil) { \
    rwp->params.prop = false; \
  } else { \
    rwp->params.prop = true; \
  } \
  return value; \



// 定义一个宏，用于设置布尔类型参数的值



#define BOOL_PARAMS_GETTER(self,  prop) \
  ruby_whisper_params *rwp; \
  Data_Get_Struct(self, ruby_whisper_params, rwp); \
  if (rwp->params.prop) { \
    return Qtrue; \
  } else { \
    return Qfalse; \
  }



// 定义一个宏，用于获取布尔类型参数的值



static void ruby_whisper_free(ruby_whisper *rw) {
  if (rw->context) {
    whisper_free(rw->context);
    rw->context = NULL;
  }
}



// 释放 ruby_whisper 结构体中的资源



static void ruby_whisper_params_free(ruby_whisper_params *rwp) {
}



// 释放 ruby_whisper_params 结构体中的资源



void rb_whisper_mark(ruby_whisper *rw) {
  // call rb_gc_mark on any ruby references in rw
}



// 在 ruby_whisper 结构体中标记任何 Ruby 引用



void rb_whisper_free(ruby_whisper *rw) {
  ruby_whisper_free(rw);
  free(rw);
}



// 释放 ruby_whisper 结构体



void rb_whisper_params_mark(ruby_whisper_params *rwp) {
}



// 在 ruby_whisper_params 结构体中标记任何 Ruby 引用



void rb_whisper_params_free(ruby_whisper_params *rwp) {
  ruby_whisper_params_free(rwp);
  free(rwp);
}



// 释放 ruby_whisper_params 结构体



static VALUE ruby_whisper_allocate(VALUE klass) {
  ruby_whisper *rw;
  rw = ALLOC(ruby_whisper);
  rw->context = NULL;
  return Data_Wrap_Struct(klass, rb_whisper_mark, rb_whisper_free, rw);
}



// 分配内存并初始化 ruby_whisper 结构体



static VALUE ruby_whisper_params_allocate(VALUE klass) {
  ruby_whisper_params *rwp;
  rwp = ALLOC(ruby_whisper_params);
  rwp->params = whisper_full_default_params(WHISPER_SAMPLING_GREEDY);
  return Data_Wrap_Struct(klass, rb_whisper_params_mark, rb_whisper_params_free, rwp);
}



// 分配内存并初始化 ruby_whisper_params 结构体
// 初始化 Ruby Whisper 对象，接受参数并返回 self
static VALUE ruby_whisper_initialize(int argc, VALUE *argv, VALUE self) {
  ruby_whisper *rw;
  VALUE whisper_model_file_path;

  // TODO: we can support init from buffer here too maybe another ruby object to expose
  // 解析参数，获取 whisper_model_file_path
  rb_scan_args(argc, argv, "01", &whisper_model_file_path);
  // 获取 self 对应的 ruby_whisper 结构体指针
  Data_Get_Struct(self, ruby_whisper, rw);

  // 检查 whisper_model_file_path 是否响应 to_s 方法
  if (!rb_respond_to(whisper_model_file_path, rb_intern("to_s"))) {
    rb_raise(rb_eRuntimeError, "Expected file path to model to initialize Whisper::Context");
  }
  // 使用 whisper_model_file_path 和默认参数初始化 Whisper::Context，存储在 rw->context 中
  rw->context = whisper_init_from_file_with_params(StringValueCStr(whisper_model_file_path), whisper_context_default_params());
  // 如果初始化失败，抛出异常
  if (rw->context == nullptr) {
    rb_raise(rb_eRuntimeError, "error: failed to initialize whisper context");
  }
  // 返回 self
  return self;
}

/*
 * transcribe a single file
 * can emit to a block results
 *
 **/
// 转录单个文件，可以将结果传递给块
static VALUE ruby_whisper_transcribe(int argc, VALUE *argv, VALUE self) {
  ruby_whisper *rw;
  ruby_whisper_params *rwp;
  VALUE wave_file_path, blk, params;

  // 解析参数，获取 wave_file_path, params, blk
  rb_scan_args(argc, argv, "02&", &wave_file_path, &params, &blk);
  // 获取 self 对应的 ruby_whisper 结构体指针
  Data_Get_Struct(self, ruby_whisper, rw);
  // 获取 params 对应的 ruby_whisper_params 结构体指针
  Data_Get_Struct(params, ruby_whisper_params, rwp);

  // 检查 wave_file_path 是否响应 to_s 方法
  if (!rb_respond_to(wave_file_path, rb_intern("to_s"))) {
    rb_raise(rb_eRuntimeError, "Expected file path to wave file");
  }

  // 将 wave_file_path 转换为 std::string
  std::string fname_inp = StringValueCStr(wave_file_path);

  // 定义存储单声道 F32 PCM 和立体声 F32 PCM 的向量
  std::vector<float> pcmf32; // mono-channel F32 PCM
  std::vector<std::vector<float>> pcmf32s; // stereo-channel F32 PCM

  // WAV 输入 - 这直接来自 main.cpp 示例
  {
    drwav wav;
    std::vector<uint8_t> wav_data; // 用于从 stdin 读取输入
  }
}
    // 如果文件名为 "-"，表示从标准输入读取数据
    if (fname_inp == "-") {
      {
        // 创建一个大小为 1024 的字节缓冲区
        uint8_t buf[1024];
        // 循环读取标准输入的数据，直到结束
        while (true) {
          // 读取数据到缓冲区中
          const size_t n = fread(buf, 1, sizeof(buf), stdin);
          // 如果没有读取到数据，跳出循环
          if (n == 0) {
            break;
          }
          // 将读取的数据添加到 wav_data 中
          wav_data.insert(wav_data.end(), buf, buf + n);
        }
      }

      // 从内存中初始化 WAV 文件
      if (drwav_init_memory(&wav, wav_data.data(), wav_data.size(), nullptr) == false) {
        fprintf(stderr, "error: failed to open WAV file from stdin\n");
        return self;
      }

      // 打印从标准输入读取的字节数
      fprintf(stderr, "%s: read %zu bytes from stdin\n", __func__, wav_data.size());
    } else if (drwav_init_file(&wav, fname_inp.c_str(), nullptr) == false) {
      // 从文件中初始化 WAV 文件
      fprintf(stderr, "error: failed to open '%s' as WAV file\n", fname_inp.c_str());
      return self;
    }

    // 检查 WAV 文件的声道数是否为 1 或 2
    if (wav.channels != 1 && wav.channels != 2) {
      fprintf(stderr, "WAV file '%s' must be mono or stereo\n", fname_inp.c_str());
      return self;
    }

    // 检查是否需要进行说话人分离，并且声道数不为 2 或时间戳未启用
    if (rwp->diarize && wav.channels != 2 && rwp->params.print_timestamps == false) {
      fprintf(stderr, "WAV file '%s' must be stereo for diarization and timestamps have to be enabled\n", fname_inp.c_str());
      return self;
    }

    // 检查 WAV 文件的采样率是否为指定值
    if (wav.sampleRate != WHISPER_SAMPLE_RATE) {
      fprintf(stderr, "WAV file '%s' must be %i kHz\n", fname_inp.c_str(), WHISPER_SAMPLE_RATE/1000);
      return self;
    }

    // 检查 WAV 文件的位深度是否为 16 位
    if (wav.bitsPerSample != 16) {
      fprintf(stderr, "WAV file '%s' must be 16-bit\n", fname_inp.c_str());
      return self;
    }

    // 计算 PCM 帧数
    const uint64_t n = wav_data.empty() ? wav.totalPCMFrameCount : wav_data.size()/(wav.channels*wav.bitsPerSample/8);

    // 创建一个存储 int16_t 类型数据的向量
    std::vector<int16_t> pcm16;
    pcm16.resize(n*wav.channels);
    // 读取 PCM 帧数据到 pcm16 中
    drwav_read_pcm_frames_s16(&wav, n, pcm16.data());
    // 释放 WAV 文件资源
    drwav_uninit(&wav);

    // 转换为单声道、浮点数
    pcmf32.resize(n);
    if (wav.channels == 1) {
      for (uint64_t i = 0; i < n; i++) {
        // 将 int16_t 类型数据转换为浮点数
        pcmf32[i] = float(pcm16[i])/32768.0f;
      }
    } else {
      // 如果不需要分离声音，将16位PCM数据转换为32位浮点数
      for (uint64_t i = 0; i < n; i++) {
        pcmf32[i] = float(pcm16[2*i] + pcm16[2*i + 1])/65536.0f;
      }
    }

    if (rwp->diarize) {
      // 如果需要分离声音，将16位PCM数据转换为32位浮点数并转换为立体声
      pcmf32s.resize(2);

      pcmf32s[0].resize(n);
      pcmf32s[1].resize(n);
      for (uint64_t i = 0; i < n; i++) {
        pcmf32s[0][i] = float(pcm16[2*i])/32768.0f;
        pcmf32s[1][i] = float(pcm16[2*i + 1])/32768.0f;
      }
    }
  }
  {
    // 定义静态变量is_aborted，应该是原子的以避免数据竞争
    static bool is_aborted = false; // NOTE: this should be atomic to avoid data race

    // 设置编码器开始回调函数，用于检查是否中止
    rwp->params.encoder_begin_callback = [](struct whisper_context * /*ctx*/, struct whisper_state * /*state*/, void * user_data) {
      bool is_aborted = *(bool*)user_data;
      return !is_aborted;
    };
    rwp->params.encoder_begin_callback_user_data = &is_aborted;
  }

  // 处理音频数据
  if (whisper_full_parallel(rw->context, rwp->params, pcmf32.data(), pcmf32.size(), 1) != 0) {
    fprintf(stderr, "failed to process audio\n");
    return self;
  }
  // 获取分段文本并拼接到输出字符串
  const int n_segments = whisper_full_n_segments(rw->context);
  VALUE output = rb_str_new2("");
  for (int i = 0; i < n_segments; ++i) {
    const char * text = whisper_full_get_segment_text(rw->context, i);
    output = rb_str_concat(output, rb_str_new2(text));
  }
  VALUE idCall = rb_intern("call");
  // 如果有块传入，则调用块函数
  if (blk != Qnil) {
    rb_funcall(blk, idCall, 1, output);
  }
  return self;
/*
 * 设置参数中的语言选项，可以是"auto"或其他语言代码
 */
static VALUE ruby_whisper_params_set_language(VALUE self, VALUE value) {
  // 获取 self 对应的 ruby_whisper_params 结构体指针
  ruby_whisper_params *rwp;
  Data_Get_Struct(self, ruby_whisper_params, rwp);
  // 如果 value 为 Qfalse 或 Qnil，则设置语言为"auto"
  if (value == Qfalse || value == Qnil) {
    rwp->params.language = "auto";
  } else {
    // 否则将 value 转换为 C 字符串并设置为语言选项
    rwp->params.language = StringValueCStr(value);
  }
  return value;
}
/*
 * 获取参数中的语言选项
 */
static VALUE ruby_whisper_params_get_language(VALUE self) {
  // 获取 self 对应的 ruby_whisper_params 结构体指针
  ruby_whisper_params *rwp;
  Data_Get_Struct(self, ruby_whisper_params, rwp);
  // 如果语言选项存在，则返回对应的 Ruby 字符串对象，否则返回"auto"
  if (rwp->params.language) {
    return rb_str_new2(rwp->params.language);
  } else {
    return rb_str_new2("auto");
  }
}
/*
 * 设置参数中的翻译选项
 */
static VALUE ruby_whisper_params_set_translate(VALUE self, VALUE value) {
  BOOL_PARAMS_SETTER(self, translate, value)
}
/*
 * 获取参数中的翻译选项
 */
static VALUE ruby_whisper_params_get_translate(VALUE self) {
  BOOL_PARAMS_GETTER(self, translate)
}
/*
 * 设置参数中的无上下文选项
 */
static VALUE ruby_whisper_params_set_no_context(VALUE self, VALUE value) {
  BOOL_PARAMS_SETTER(self, no_context, value)
}
/*
 * 获取参数中的无上下文选项
 */
static VALUE ruby_whisper_params_get_no_context(VALUE self) {
  BOOL_PARAMS_GETTER(self, no_context)
}
/*
 * 设置参数中的单段翻译选项
 */
static VALUE ruby_whisper_params_set_single_segment(VALUE self, VALUE value) {
  BOOL_PARAMS_SETTER(self, single_segment, value)
}
/*
 * 获取参数中的单段翻译选项
 */
static VALUE ruby_whisper_params_get_single_segment(VALUE self) {
  BOOL_PARAMS_GETTER(self, single_segment)
}
/*
 * 设置参数中的打印特殊字符选项
 */
static VALUE ruby_whisper_params_set_print_special(VALUE self, VALUE value) {
  BOOL_PARAMS_SETTER(self, print_special, value)
}
/*
 * 获取参数中的打印特殊字符选项
 */
static VALUE ruby_whisper_params_get_print_special(VALUE self) {
  BOOL_PARAMS_GETTER(self, print_special)
}
/*
 * 设置参数中的打印进度选项
 */
static VALUE ruby_whisper_params_set_print_progress(VALUE self, VALUE value) {
  BOOL_PARAMS_SETTER(self, print_progress, value)
}
/*
 * 获取参数中的打印进度选项
 */
static VALUE ruby_whisper_params_get_print_progress(VALUE self) {
  BOOL_PARAMS_GETTER(self, print_progress)
}
/*
 * 设置参数中的实时打印选项
 */
static VALUE ruby_whisper_params_set_print_realtime(VALUE self, VALUE value) {
  BOOL_PARAMS_SETTER(self, print_realtime, value)
}
# 获取实时打印参数的值
static VALUE ruby_whisper_params_get_print_realtime(VALUE self) {
  BOOL_PARAMS_GETTER(self, print_realtime)
}

# 设置打印时间戳参数的值
static VALUE ruby_whisper_params_set_print_timestamps(VALUE self, VALUE value) {
  BOOL_PARAMS_SETTER(self, print_timestamps, value)
}

# 获取打印时间戳参数的值
static VALUE ruby_whisper_params_get_print_timestamps(VALUE self) {
  BOOL_PARAMS_GETTER(self, print_timestamps)
}

# 设置忽略空白参数的值
static VALUE ruby_whisper_params_set_suppress_blank(VALUE self, VALUE value) {
  BOOL_PARAMS_SETTER(self, suppress_blank, value)
}

# 获取忽略空白参数的值
static VALUE ruby_whisper_params_get_suppress_blank(VALUE self) {
  BOOL_PARAMS_GETTER(self, suppress_blank)
}

# 设置忽略非语音标记参数的值
static VALUE ruby_whisper_params_set_suppress_non_speech_tokens(VALUE self, VALUE value) {
  BOOL_PARAMS_SETTER(self, suppress_non_speech_tokens, value)
}

# 获取忽略非语音标记参数的值
static VALUE ruby_whisper_params_get_suppress_non_speech_tokens(VALUE self) {
  BOOL_PARAMS_GETTER(self, suppress_non_speech_tokens)
}

# 获取标记时间戳参数的值
static VALUE ruby_whisper_params_get_token_timestamps(VALUE self) {
  BOOL_PARAMS_GETTER(self, token_timestamps)
}

# 设置标记时间戳参数的值
static VALUE ruby_whisper_params_set_token_timestamps(VALUE self, VALUE value) {
  BOOL_PARAMS_SETTER(self, token_timestamps, value)
}

# 获取按单词拆分参数的值
static VALUE ruby_whisper_params_get_split_on_word(VALUE self) {
  BOOL_PARAMS_GETTER(self, split_on_word)
}

# 设置按单词拆分参数的值
static VALUE ruby_whisper_params_set_split_on_word(VALUE self, VALUE value) {
  BOOL_PARAMS_SETTER(self, split_on_word, value)
}

# 获取加速参数的值
static VALUE ruby_whisper_params_get_speed_up(VALUE self) {
  BOOL_PARAMS_GETTER(self, speed_up)
}

# 设置加速参数的值
static VALUE ruby_whisper_params_set_speed_up(VALUE self, VALUE value) {
  BOOL_PARAMS_SETTER(self, speed_up, value)
}

# 获取分析参数的值
static VALUE ruby_whisper_params_get_diarize(VALUE self) {
  ruby_whisper_params *rwp;
  Data_Get_Struct(self, ruby_whisper_params, rwp);
  如果分析参数为真，则返回 Qtrue，否则返回 Qfalse
  if (rwp->diarize) {
    return Qtrue;
  } else {
    return Qfalse;
  }
}
static VALUE ruby_whisper_params_set_diarize(VALUE self, VALUE value) {
  // 获取 self 对应的 ruby_whisper_params 结构体指针
  ruby_whisper_params *rwp;
  Data_Get_Struct(self, ruby_whisper_params, rwp);
  // 如果 value 是 Qfalse 或者 Qnil，则将 diarize 设置为 false，否则设置为 true
  if (value == Qfalse || value == Qnil) {
    rwp->diarize = false;
  } else {
    rwp->diarize = true;
  } \
  // 返回 value
  return value;
}

static VALUE ruby_whisper_params_get_offset(VALUE self) {
  // 获取 self 对应的 ruby_whisper_params 结构体指针
  ruby_whisper_params *rwp;
  Data_Get_Struct(self, ruby_whisper_params, rwp);
  // 返回 params 结构体中的 offset_ms 字段值
  return INT2NUM(rwp->params.offset_ms);
}
static VALUE ruby_whisper_params_set_offset(VALUE self, VALUE value) {
  // 获取 self 对应的 ruby_whisper_params 结构体指针
  ruby_whisper_params *rwp;
  Data_Get_Struct(self, ruby_whisper_params, rwp);
  // 将 value 转换为整数并设置为 params 结构体中的 offset_ms 字段值
  rwp->params.offset_ms = NUM2INT(value);
  // 返回 value
  return value;
}
static VALUE ruby_whisper_params_get_duration(VALUE self) {
  // 获取 self 对应的 ruby_whisper_params 结构体指针
  ruby_whisper_params *rwp;
  Data_Get_Struct(self, ruby_whisper_params, rwp);
  // 返回 params 结构体中的 duration_ms 字段值
  return INT2NUM(rwp->params.duration_ms);
}
static VALUE ruby_whisper_params_set_duration(VALUE self, VALUE value) {
  // 获取 self 对应的 ruby_whisper_params 结构体指针
  ruby_whisper_params *rwp;
  Data_Get_Struct(self, ruby_whisper_params, rwp);
  // 将 value 转换为整数并设置为 params 结构体中的 duration_ms 字段值
  rwp->params.duration_ms = NUM2INT(value);
  // 返回 value
  return value;
}

static VALUE ruby_whisper_params_get_max_text_tokens(VALUE self) {
  // 获取 self 对应的 ruby_whisper_params 结构体指针
  ruby_whisper_params *rwp;
  Data_Get_Struct(self, ruby_whisper_params, rwp);
  // 返回 params 结构体中的 n_max_text_ctx 字段值
  return INT2NUM(rwp->params.n_max_text_ctx);
}
static VALUE ruby_whisper_params_set_max_text_tokens(VALUE self, VALUE value) {
  // 获取 self 对应的 ruby_whisper_params 结构体指针
  ruby_whisper_params *rwp;
  Data_Get_Struct(self, ruby_whisper_params, rwp);
  // 将 value 转换为整数并设置为 params 结构体中的 n_max_text_ctx 字段值
  rwp->params.n_max_text_ctx = NUM2INT(value);
  // 返回 value
  return value;
}

}
#ifdef __cplusplus
}
#endif
```