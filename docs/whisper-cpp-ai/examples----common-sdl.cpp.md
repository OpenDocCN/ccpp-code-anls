# `whisper.cpp\examples\common-sdl.cpp`

```cpp
// 包含 common-sdl.h 头文件

audio_async::audio_async(int len_ms) {
    // 初始化音频长度
    m_len_ms = len_ms;

    // 设置运行状态为 false
    m_running = false;
}

audio_async::~audio_async() {
    // 如果输入设备 ID 存在，则关闭音频设备
    if (m_dev_id_in) {
        SDL_CloseAudioDevice(m_dev_id_in);
    }
}

bool audio_async::init(int capture_id, int sample_rate) {
    // 设置日志输出级别为 INFO
    SDL_LogSetPriority(SDL_LOG_CATEGORY_APPLICATION, SDL_LOG_PRIORITY_INFO);

    // 初始化 SDL 音频模块
    if (SDL_Init(SDL_INIT_AUDIO) < 0) {
        // 输出错误信息并返回 false
        SDL_LogError(SDL_LOG_CATEGORY_APPLICATION, "Couldn't initialize SDL: %s\n", SDL_GetError());
        return false;
    }

    // 设置音频重采样模式为 medium
    SDL_SetHintWithPriority(SDL_HINT_AUDIO_RESAMPLING_MODE, "medium", SDL_HINT_OVERRIDE);

    {
        // 获取可用的音频设备数量
        int nDevices = SDL_GetNumAudioDevices(SDL_TRUE);
        fprintf(stderr, "%s: found %d capture devices:\n", __func__, nDevices);
        // 遍历输出每个音频设备的信息
        for (int i = 0; i < nDevices; i++) {
            fprintf(stderr, "%s:    - Capture device #%d: '%s'\n", __func__, i, SDL_GetAudioDeviceName(i, SDL_TRUE));
        }
    }

    // 初始化音频规格请求和获得的规格
    SDL_AudioSpec capture_spec_requested;
    SDL_AudioSpec capture_spec_obtained;

    // 将规格清零
    SDL_zero(capture_spec_requested);
    SDL_zero(capture_spec_obtained);

    // 设置音频规格请求参数
    capture_spec_requested.freq     = sample_rate;
    capture_spec_requested.format   = AUDIO_F32;
    capture_spec_requested.channels = 1;
    capture_spec_requested.samples  = 1024;
    // 设置回调函数
    capture_spec_requested.callback = [](void * userdata, uint8_t * stream, int len) {
        audio_async * audio = (audio_async *) userdata;
        audio->callback(stream, len);
    };
    capture_spec_requested.userdata = this;

    // 如果捕获设备 ID 大于等于 0
    if (capture_id >= 0) {
        // 尝试打开捕获设备
        fprintf(stderr, "%s: attempt to open capture device %d : '%s' ...\n", __func__, capture_id, SDL_GetAudioDeviceName(capture_id, SDL_TRUE));
        m_dev_id_in = SDL_OpenAudioDevice(SDL_GetAudioDeviceName(capture_id, SDL_TRUE), SDL_TRUE, &capture_spec_requested, &capture_spec_obtained, 0);
    } else {
        // 如果无法打开默认捕获设备，则输出错误信息
        fprintf(stderr, "%s: attempt to open default capture device ...\n", __func__);
        // 尝试打开音频捕获设备，获取捕获设备的规格
        m_dev_id_in = SDL_OpenAudioDevice(nullptr, SDL_TRUE, &capture_spec_requested, &capture_spec_obtained, 0);
    }

    // 如果无法打开音频捕获设备
    if (!m_dev_id_in) {
        // 输出错误信息
        fprintf(stderr, "%s: couldn't open an audio device for capture: %s!\n", __func__, SDL_GetError());
        // 将捕获设备 ID 置为 0
        m_dev_id_in = 0;
        // 返回 false
        return false;
    } else {
        // 输出成功获取输入设备规格的信息
        fprintf(stderr, "%s: obtained spec for input device (SDL Id = %d):\n", __func__, m_dev_id_in);
        // 输出捕获设备的采样率
        fprintf(stderr, "%s:     - sample rate:       %d\n",                   __func__, capture_spec_obtained.freq);
        // 输出捕获设备的格式和所需格式
        fprintf(stderr, "%s:     - format:            %d (required: %d)\n",    __func__, capture_spec_obtained.format,
                capture_spec_requested.format);
        // 输出捕获设备的声道数和所需声道数
        fprintf(stderr, "%s:     - channels:          %d (required: %d)\n",    __func__, capture_spec_obtained.channels,
                capture_spec_requested.channels);
        // 输出捕获设备的每帧样本数
        fprintf(stderr, "%s:     - samples per frame: %d\n",                   __func__, capture_spec_obtained.samples);
    }

    // 将采样率设置为捕获设备规格中的采样率
    m_sample_rate = capture_spec_obtained.freq;

    // 根据采样率和长度计算音频数据的大小，并调整音频数据的大小
    m_audio.resize((m_sample_rate*m_len_ms)/1000);

    // 返回 true
    return true;
// 恢复音频播放
bool audio_async::resume() {
    // 如果没有音频设备，则打印错误信息并返回 false
    if (!m_dev_id_in) {
        fprintf(stderr, "%s: no audio device to resume!\n", __func__);
        return false;
    }

    // 如果已经在运行，则打印错误信息并返回 false
    if (m_running) {
        fprintf(stderr, "%s: already running!\n", __func__);
        return false;
    }

    // 恢复音频设备的播放
    SDL_PauseAudioDevice(m_dev_id_in, 0);

    // 设置运行状态为 true
    m_running = true;

    return true;
}

// 暂停音频播放
bool audio_async::pause() {
    // 如果没有音频设备，则打印错误信息并返回 false
    if (!m_dev_id_in) {
        fprintf(stderr, "%s: no audio device to pause!\n", __func__);
        return false;
    }

    // 如果已经暂停，则打印错误信息并返回 false
    if (!m_running) {
        fprintf(stderr, "%s: already paused!\n", __func__);
        return false;
    }

    // 暂停音频设备的播放
    SDL_PauseAudioDevice(m_dev_id_in, 1);

    // 设置运行状态为 false
    m_running = false;

    return true;
}

// 清除音频数据
bool audio_async::clear() {
    // 如果没有音频设备，则打印错误信息并返回 false
    if (!m_dev_id_in) {
        fprintf(stderr, "%s: no audio device to clear!\n", __func__);
        return false;
    }

    // 如果没有在运行，则打印错误信息并返回 false
    if (!m_running) {
        fprintf(stderr, "%s: not running!\n", __func__);
        return false;
    }

    {
        // 使用互斥锁保护临界区
        std::lock_guard<std::mutex> lock(m_mutex);

        // 重置音频位置和长度
        m_audio_pos = 0;
        m_audio_len = 0;
    }

    return true;
}

// 由 SDL 调用的回调函数
void audio_async::callback(uint8_t * stream, int len) {
    // 如果没有在运行，则直接返回
    if (!m_running) {
        return;
    }

    // 计算样本数
    size_t n_samples = len / sizeof(float);

    // 如果样本数大于音频数据的大小，则调整样本数和流的位置
    if (n_samples > m_audio.size()) {
        n_samples = m_audio.size();

        stream += (len - (n_samples * sizeof(float)));
    }

    // 打印调试信息
    //fprintf(stderr, "%s: %zu samples, pos %zu, len %zu\n", __func__, n_samples, m_audio_pos, m_audio_len);
}
    {
        // 使用互斥锁保护临界区，确保线程安全
        std::lock_guard<std::mutex> lock(m_mutex);
    
        // 如果当前音频位置加上采样数大于音频大小
        if (m_audio_pos + n_samples > m_audio.size()) {
            // 计算需要拷贝的样本数
            const size_t n0 = m_audio.size() - m_audio_pos;
    
            // 将部分数据拷贝到音频缓冲区末尾
            memcpy(&m_audio[m_audio_pos], stream, n0 * sizeof(float));
            // 将剩余数据拷贝到音频缓冲区开头
            memcpy(&m_audio[0], stream + n0 * sizeof(float), (n_samples - n0) * sizeof(float));
    
            // 更新音频位置
            m_audio_pos = (m_audio_pos + n_samples) % m_audio.size();
            // 更新音频长度
            m_audio_len = m_audio.size();
        } else {
            // 将数据拷贝到音频缓冲区
            memcpy(&m_audio[m_audio_pos], stream, n_samples * sizeof(float));
    
            // 更新音频位置
            m_audio_pos = (m_audio_pos + n_samples) % m_audio.size();
            // 更新音频长度，取当前长度和新增采样数的最小值
            m_audio_len = std::min(m_audio_len + n_samples, m_audio.size());
        }
    }
// 获取指定毫秒数内的音频数据并存储在结果向量中
void audio_async::get(int ms, std::vector<float> & result) {
    // 如果音频输入设备未设置，则输出错误信息并返回
    if (!m_dev_id_in) {
        fprintf(stderr, "%s: no audio device to get audio from!\n", __func__);
        return;
    }

    // 如果音频未在运行状态，则输出错误信息并返回
    if (!m_running) {
        fprintf(stderr, "%s: not running!\n", __func__);
        return;
    }

    // 清空结果向量
    result.clear();

    {
        // 使用互斥锁保护临界区
        std::lock_guard<std::mutex> lock(m_mutex);

        // 如果指定毫秒数小于等于0，则使用默认长度
        if (ms <= 0) {
            ms = m_len_ms;
        }

        // 计算需要获取的样本数
        size_t n_samples = (m_sample_rate * ms) / 1000;
        if (n_samples > m_audio_len) {
            n_samples = m_audio_len;
        }

        // 调整结果向量大小
        result.resize(n_samples);

        // 计算起始位置
        int s0 = m_audio_pos - n_samples;
        if (s0 < 0) {
            s0 += m_audio.size();
        }

        // 复制音频数据到结果向量中
        if (s0 + n_samples > m_audio.size()) {
            const size_t n0 = m_audio.size() - s0;

            memcpy(result.data(), &m_audio[s0], n0 * sizeof(float));
            memcpy(&result[n0], &m_audio[0], (n_samples - n0) * sizeof(float));
        } else {
            memcpy(result.data(), &m_audio[s0], n_samples * sizeof(float));
        }
    }
}

// 检查并处理 SDL 事件，返回是否继续运行
bool sdl_poll_events() {
    SDL_Event event;
    while (SDL_PollEvent(&event)) {
        switch (event.type) {
            // 如果事件类型为退出事件，则返回 false
            case SDL_QUIT:
                {
                    return false;
                } break;
            default:
                break;
        }
    }

    // 如果没有退出事件，则返回 true
    return true;
}
```