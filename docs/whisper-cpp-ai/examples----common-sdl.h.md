# `whisper.cpp\examples\common-sdl.h`

```cpp
#pragma once

#include <SDL.h>
#include <SDL_audio.h>

#include <atomic>
#include <cstdint>
#include <vector>
#include <mutex>

//
// SDL Audio capture
//

// 声明一个名为audio_async的类
class audio_async {
public:
    // 构造函数，接受一个长度参数len_ms
    audio_async(int len_ms);
    // 析构函数
    ~audio_async();

    // 初始化音频捕获，设置捕获设备ID和采样率
    bool init(int capture_id, int sample_rate);

    // 开始通过提供的SDL回调函数捕获音频，保留最后len_ms秒的音频在循环缓冲区中
    bool resume();
    // 暂停音频捕获
    bool pause();
    // 清空音频缓冲区
    bool clear();

    // 由SDL调用的回调函数
    void callback(uint8_t * stream, int len);

    // 从循环缓冲区中获取音频数据
    void get(int ms, std::vector<float> & audio);

private:
    // SDL音频设备ID
    SDL_AudioDeviceID m_dev_id_in = 0;

    // 长度和采样率
    int m_len_ms = 0;
    int m_sample_rate = 0;

    // 原子布尔变量和互斥锁
    std::atomic_bool m_running;
    std::mutex       m_mutex;

    // 音频数据和位置
    std::vector<float> m_audio;
    size_t             m_audio_pos = 0;
    size_t             m_audio_len = 0;
};

// 如果需要退出，则返回false
bool sdl_poll_events();
```