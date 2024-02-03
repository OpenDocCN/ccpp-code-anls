# `xmrig\src\base\net\tools\LineReader.cpp`

```cpp
/*
 * XMRig
 * 版权所有（c）2020      cohcho      <https://github.com/cohcho>
 * 版权所有（c）2018-2020 SChernykh   <https://github.com/SChernykh>
 * 版权所有（c）2016-2020 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 * 本程序是自由软件：您可以重新发布或修改
 * 根据由自由软件基金会发布的GNU通用公共许可证的条款，
 * 许可证的版本为3，或者
 * （在您的选择下）任何以后的版本。
 *
 * 本程序是希望它有用，
 * 但没有任何保证；甚至没有暗示的保证
 * 适销性或特定用途的保证。有关更多详细信息，请参见
 * GNU通用公共许可证。
 *
 * 您应该已经收到GNU通用公共许可证的副本
 * 与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#include "base/net/tools/LineReader.h"
#include "base/kernel/constants.h"
#include "base/kernel/interfaces/ILineListener.h"
#include "base/net/tools/NetBuffer.h"

#include <cassert>
#include <cstring>


// 析构函数，释放网络缓冲区
xmrig::LineReader::~LineReader()
{
    NetBuffer::release(m_buf);
}


// 解析数据，确保监听器不为空且数据大小大于0，然后调用getline函数
void xmrig::LineReader::parse(char *data, size_t size)
{
    assert(m_listener != nullptr && size > 0);
    if (!m_listener || size == 0) {
        return;
    }

    getline(data, size);
}


// 重置LineReader对象，释放网络缓冲区
void xmrig::LineReader::reset()
{
    if (m_buf) {
        NetBuffer::release(m_buf);
        m_buf = nullptr;
        m_pos = 0;
    }
}


// 添加数据到网络缓冲区，如果数据大小加上当前位置大于网络缓冲区的最大大小，则返回
// 否则，分配网络缓冲区并将数据复制到缓冲区中
void xmrig::LineReader::add(const char *data, size_t size)
{
    if (size + m_pos > XMRIG_NET_BUFFER_CHUNK_SIZE) {
        // it breaks correctness silently for long lines
        return;
    }

    if (!m_buf) {
        m_buf = NetBuffer::allocate();
        m_pos = 0;
    }

    memcpy(m_buf + m_pos, data, size);
    m_pos += size;
}


// 从网络缓冲区中获取一行数据
void xmrig::LineReader::getline(char *data, size_t size)
{
    char *end        = nullptr;
    char *start      = data;
    # 声明一个变量 remaining，表示剩余未处理的数据大小
    size_t remaining = size;

    # 在剩余数据中查找换行符 \n
    while ((end = static_cast<char*>(memchr(start, '\n', remaining))) != nullptr) {
        # 将找到的换行符替换为字符串结束符 \0
        *end = '\0';

        # 移动到下一行的起始位置
        end++;

        # 计算当前行的长度
        const auto len = static_cast<size_t>(end - start);
        # 如果当前位置不为0，则将当前行数据添加到缓冲区中，并触发 onLine 事件
        if (m_pos) {
            add(start, len);
            m_listener->onLine(m_buf, m_pos - 1);
            m_pos = 0;
        }
        # 如果当前位置为0且行长度大于1，则直接触发 onLine 事件
        else if (len > 1) {
            m_listener->onLine(start, len - 1);
        }

        # 更新剩余数据大小
        remaining -= len;
        # 更新起始位置到下一行的起始位置
        start = end;
    }

    # 如果剩余数据大小为0，则重置处理状态
    if (remaining == 0) {
        return reset();
    }

    # 将剩余数据添加到缓冲区中
    add(start, remaining);
# 闭合前面的函数定义
```