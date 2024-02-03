# `xmrig\src\crypto\common\Nonce.cpp`

```cpp
/*
 * XMRig
 * 版权所有（c）2018-2021 SChernykh   <https://github.com/SChernykh>
 * 版权所有（c）2016-2021 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 * 本程序是自由软件：您可以重新分发或修改
 * 根据GNU通用公共许可证的条款发布
 * 自由软件基金会发布的许可证的第3版，或者
 * （在您的选择）任何以后的版本。
 *
 * 本程序是希望它有用，
 * 但没有任何保证；甚至没有暗示的保证
 * 适销性或特定用途的保证。请参阅
 * GNU通用公共许可证以获取更多详细信息。
 *
 * 您应该已经收到GNU通用公共许可证的副本
 * 与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#include "base/tools/Alignment.h"  // 包含对齐工具的头文件
#include "crypto/common/Nonce.h"    // 包含 Nonce 类的头文件

namespace xmrig {

std::atomic<bool> Nonce::m_paused = {true};  // 初始化静态成员变量 m_paused
std::atomic<uint64_t>  Nonce::m_sequence[Nonce::MAX] = { {1}, {1}, {1} };  // 初始化静态成员变量 m_sequence
std::atomic<uint64_t> Nonce::m_nonces[2] = { {0}, {0} };  // 初始化静态成员变量 m_nonces

} // namespace xmrig

bool xmrig::Nonce::next(uint8_t index, uint32_t *nonce, uint32_t reserveCount, uint64_t mask)  // Nonce 类的 next 方法
{
    mask &= 0x7FFFFFFFFFFFFFFFULL;  // 对 mask 进行位与运算
    if (reserveCount == 0 || mask < reserveCount - 1) {  // 如果 reserveCount 为0或者 mask 小于 reserveCount - 1
        return false;  // 返回 false
    }

    uint64_t counter = m_nonces[index].fetch_add(reserveCount, std::memory_order_relaxed);  // 使用原子操作对 m_nonces[index] 进行加法操作，并将结果赋值给 counter
    # 进入循环，条件为始终为真
    while (true) {
        # 如果掩码小于计数器，则返回假
        if (mask < counter) {
            return false;
        }

        # 如果掩码减去计数器小于等于保留计数减1，则暂停线程，如果掩码减去计数器小于保留计数减1，则返回假
        if (mask - counter <= reserveCount - 1) {
            pause(true);
            if (mask - counter < reserveCount - 1) {
                return false;
            }
        }
        # 如果无符号32位整数的最大值减去计数器小于保留计数减1，则更新计数器并继续循环
        else if (0xFFFFFFFFUL - (uint32_t)counter < reserveCount - 1) {
            counter = m_nonces[index].fetch_add(reserveCount, std::memory_order_relaxed);
            continue;
        }

        # 将nonce中的值与掩码取反后的值或上计数器的值写入
        writeUnaligned(nonce, static_cast<uint32_t>((readUnaligned(nonce) & ~mask) | counter));

        # 如果掩码大于无符号64位整数的最大值，则将nonce+1中的值与掩码右移32位取反后的值或上计数器右移32位的值写入
        if (mask > 0xFFFFFFFFULL) {
            writeUnaligned(nonce + 1, static_cast<uint32_t>((readUnaligned(nonce + 1) & (~mask >> 32)) | (counter >> 32)));
        }

        # 返回真
        return true;
    }
# 停止 Nonce 对象的操作
void xmrig::Nonce::stop()
{
    # 恢复 Nonce 对象的操作
    pause(false);

    # 遍历 m_sequence 数组，将每个元素设置为 0
    for (auto &i : m_sequence) {
        i = 0;
    }
}

# 更新 Nonce 对象的操作时间
void xmrig::Nonce::touch()
{
    # 遍历 m_sequence 数组，每个元素加一
    for (auto &i : m_sequence) {
        i++;
    }
}
```