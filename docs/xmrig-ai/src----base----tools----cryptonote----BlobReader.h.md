# `xmrig\src\base\tools\cryptonote\BlobReader.h`

```cpp
/* XMRig
 * 版权所有 (c) 2012-2013 The Cryptonote developers
 * 版权所有 (c) 2014-2021 The Monero Project
 * 版权所有 (c) 2018-2021 SChernykh   <https://github.com/SChernykh>
 * 版权所有 (c) 2016-2021 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 * 本程序是自由软件：您可以重新发布或修改它
 * 根据由自由软件基金会发布的 GNU 通用公共许可证的条款，
 * 无论是许可证的第 3 版还是（在您的选择下）任何以后的版本。
 *
 * 本程序是希望它有用的分发，
 * 但没有任何保证；甚至没有对特定目的的隐含保证。请参阅
 * GNU 通用公共许可证以获取更多详细信息。
 *
 * 您应该已经收到了 GNU 通用公共许可证的副本
 * 与本程序一起。如果没有，请参见 <http://www.gnu.org/licenses/>。
 */

#ifndef XMRIG_BLOBREADER_H
#define XMRIG_BLOBREADER_H


#include <cstdint>
#include <cstring>
#include <stdexcept>


namespace xmrig {


template<bool EXCEPTIONS>
class BlobReader
{
public:
    // 构造函数，初始化数据和大小
    inline BlobReader(const uint8_t *data, size_t size) :
        m_size(size),
        m_data(data)
    {}

    // 读取变长整数
    inline bool operator()(uint64_t &data)  { return getVarint(data); }
    // 读取字节
    inline bool operator()(uint8_t &data)   { return getByte(data); }
    // 返回当前索引
    inline size_t index() const             { return m_index; }
    // 返回剩余数据大小
    inline size_t remaining() const         { return m_size - m_index;  }

    // 跳过指定大小的数据
    inline bool skip(size_t n)
    {
        if (m_index + n > m_size) {
            return outOfRange();
        }

        m_index += n;

        return true;
    }

    // 读取固定大小的数据
    template<size_t N>
    inline bool operator()(uint8_t(&data)[N])
    {
        if (m_index + N > m_size) {
            return outOfRange();
        }

        memcpy(data, m_data + m_index, N);
        m_index += N;

        return true;
    }

    // 读取模板类型的数据
    template<typename T>
    # 定义一个重载的函数调用运算符，接受数据和大小作为参数
    inline bool operator()(T &data, size_t n)
    {
        # 如果索引加上大小超出了数据大小，则返回越界错误
        if (m_index + n > m_size) {
            return outOfRange();
        }

        # 将数据指针指向当前索引位置，并设置大小为n
        data = { m_data + m_index, n };
        # 更新索引位置
        m_index += n;

        # 返回true表示成功
        return true;
    }
private:
    // 从数据中获取一个字节，并存储到指定的变量中
    inline bool getByte(uint8_t &data)
    {
        // 如果索引超出数据大小，则返回越界错误
        if (m_index >= m_size) {
            return outOfRange();
        }

        // 从数据中获取一个字节，并将索引后移
        data = m_data[m_index++];

        return true;
    }

    // 从数据中获取一个变长整数，并存储到指定的变量中
    inline bool getVarint(uint64_t &data)
    {
        uint64_t result = 0;
        uint8_t t;
        int shift = 0;

        // 循环读取字节，直到遇到最后一个字节
        do {
            // 如果无法获取字节，则返回越界错误
            if (!getByte(t)) {
                return outOfRange();
            }

            // 将字节的低7位存储到结果中，并更新偏移量
            result |= static_cast<uint64_t>(t & 0x7F) << shift;
            shift += 7;
        } while (t & 0x80);

        // 将结果存储到指定的变量中
        data = result;

        return true;
    }

    // 如果开启异常处理，则抛出越界异常，否则返回 false
    inline bool outOfRange()
    {
        if (EXCEPTIONS) {
            throw std::out_of_range("Blob read out of range");
        }

        return false;
    }

    // 数据大小
    const size_t m_size;
    // 数据指针
    const uint8_t *m_data;
    // 当前索引
    size_t m_index  = 0;
};


} /* namespace xmrig */


#endif /* XMRIG_BLOBREADER_H */
```