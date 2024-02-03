# `xmrig\src\crypto\common\Nonce.h`

```cpp
/* XMRig
 * 版权所有（c）2018-2021 SChernykh   <https://github.com/SChernykh>
 * 版权所有（c）2016-2021 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以重新分发或修改
 *   根据由自由软件基金会发布的GNU通用公共许可证的条款
 *   许可证的第3版或（根据您的选择）任何以后的版本。
 *
 *   本程序是希望它有用，
 *   但没有任何保证；甚至没有暗示的保证
 *   适销性或特定用途的保证。请参阅
 *   GNU通用公共许可证以获取更多详细信息。
 *
 *   您应该已经收到GNU通用公共许可证的副本
 *   与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#ifndef XMRIG_NONCE_H
#define XMRIG_NONCE_H


#include <atomic>


namespace xmrig {


class Nonce
{
public:
    enum Backend : uint32_t {
        CPU,
        OPENCL,
        CUDA,
        MAX
    };


    static inline bool isOutdated(Backend backend, uint64_t sequence)   { return m_sequence[backend].load(std::memory_order_relaxed) != sequence; }
    static inline bool isPaused()                                       { return m_paused.load(std::memory_order_relaxed); }
    static inline uint64_t sequence(Backend backend)                    { return m_sequence[backend].load(std::memory_order_relaxed); }
    static inline void pause(bool paused)                               { m_paused = paused; }
    static inline void reset(uint8_t index)                             { m_nonces[index] = 0; }
    static inline void stop(Backend backend)                            { m_sequence[backend] = 0; }
    static inline void touch(Backend backend)                           { m_sequence[backend]++; }

    static bool next(uint8_t index, uint32_t *nonce, uint32_t reserveCount, uint64_t mask);
    static void stop();
    static void touch();

private:
    # 创建一个静态的原子布尔变量，用于表示是否被暂停
    static std::atomic<bool> m_paused;
    # 创建一个静态的原子无符号长整型数组，用于存储最大长度为MAX的序列
    static std::atomic<uint64_t> m_sequence[MAX];
    # 创建一个静态的原子无符号长整型数组，用于存储两个非重复的值
    static std::atomic<uint64_t> m_nonces[2];
}; 
// 结束匿名命名空间

} // namespace xmrig
// 结束 xmrig 命名空间

#endif /* XMRIG_NONCE_H */
// 结束条件编译指令，检查是否定义了 XMRIG_NONCE_H
```