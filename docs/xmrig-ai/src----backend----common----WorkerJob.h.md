# `xmrig\src\backend\common\WorkerJob.h`

```cpp
/* XMRig
 * 版权所有 2010      Jeff Garzik <jgarzik@pobox.com>
 * 版权所有 2012-2014 pooler      <pooler@litecoinpool.org>
 * 版权所有 2014      Lucas Jones <https://github.com/lucasjones>
 * 版权所有 2014-2016 Wolf9466    <https://github.com/OhGodAPet>
 * 版权所有 2016      Jay D Dee   <jayddee246@gmail.com>
 * 版权所有 2017-2018 XMR-Stak    <https://github.com/fireice-uk>, <https://github.com/psychocrypt>
 * 版权所有 2018-2020 SChernykh   <https://github.com/SChernykh>
 * 版权所有 2016-2020 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以重新发布和/或修改
 *   根据由自由软件基金会发布的 GNU 通用公共许可证的条款，
 *   要么是许可证的第3版，要么（在您的选择下）是任何更高版本。
 *
 *   本程序是希望它有用的分发，
 *   但没有任何保证；甚至没有暗示的保证适用于特定目的。请参阅
 *   GNU 通用公共许可证以获取更多详细信息。
 *
 *   您应该已经收到了 GNU 通用公共许可证的副本
 *   与本程序一起。如果没有，请参阅 <http://www.gnu.org/licenses/>。
 */

#ifndef XMRIG_WORKERJOB_H
#define XMRIG_WORKERJOB_H


#include <cstring>


#include "base/net/stratum/Job.h"
#include "base/tools/Alignment.h"
#include "crypto/common/Nonce.h"


namespace xmrig {


template<size_t N>
class WorkerJob
{
public:
    // 返回当前作业
    inline const Job &currentJob() const    { return m_jobs[index()]; }
    // 返回随机数指针
    inline uint32_t *nonce(size_t i = 0)    { return reinterpret_cast<uint32_t*>(blob() + (i * currentJob().size()) + nonceOffset()); }
    // 返回序列号
    inline uint64_t sequence() const        { return m_sequence; }
    // 返回数据块指针
    inline uint8_t *blob()                  { return m_blobs[index()]; }
    // 返回索引
    inline uint8_t index() const            { return m_index; }


    // 添加作业
    inline void add(const Job &job, uint32_t reserveCount, Nonce::Backend backend)
    {
        // 生成一个新的序列号，使用给定的后端
        m_sequence = Nonce::sequence(backend);

        // 如果当前作业等于给定的作业，则直接返回
        if (currentJob() == job) {
            return;
        }

        // 如果索引为1且给定作业的索引为0且作业等于m_jobs[0]，则将索引设置为0并返回
        if (index() == 1 && job.index() == 0 && job == m_jobs[0]) {
            m_index = 0;
            return;
        }

        // 保存给定的作业、保留计数和后端
        save(job, reserveCount, backend);
    }


    // 在下一轮中，增加当前轮次的计数，并根据条件更新nonce
    inline bool nextRound(uint32_t rounds, uint32_t roundSize)
    {
        // 增加当前索引位置的轮次计数
        m_rounds[index()]++;

        // 如果当前轮次计数与(rounds - 1)按位与的结果为0，则更新nonce
        if ((m_rounds[index()] & (rounds - 1)) == 0) {
            // 遍历N个nonce，根据条件更新nonce
            for (size_t i = 0; i < N; ++i) {
                if (!Nonce::next(index(), nonce(i), rounds * roundSize, nonceMask())) {
                    return false;
                }
            }
        }
        // 否则，根据条件更新nonce
        else {
            for (size_t i = 0; i < N; ++i) {
                writeUnaligned(nonce(i), readUnaligned(nonce(i)) + roundSize);
            }
        }

        return true;
    }


    // 返回当前作业的nonce偏移量
    inline int32_t nonceOffset() const { return currentJob().nonceOffset(); }
    // 返回当前作业的nonce大小
    inline size_t nonceSize() const { return currentJob().nonceSize(); }
private:
    // 返回当前索引位置的 nonce 掩码
    inline uint64_t nonceMask() const     { return m_nonce_mask[index()]; }

    // 保存作业信息和相关数据
    inline void save(const Job &job, uint32_t reserveCount, Nonce::Backend backend)
    {
        // 设置当前索引位置
        m_index           = job.index();
        // 获取作业大小
        const size_t size = job.size();
        // 保存作业信息到对应索引位置
        m_jobs[index()]   = job;
        // 初始化当前索引位置的轮数为 0
        m_rounds[index()] = 0;
        // 保存当前索引位置的 nonce 掩码
        m_nonce_mask[index()] = job.nonceMask();

        // 设置作业的后端类型
        m_jobs[index()].setBackend(backend);

        // 复制作业数据到对应索引位置的内存中
        for (size_t i = 0; i < N; ++i) {
            memcpy(m_blobs[index()] + (i * size), job.blob(), size);
            // 生成下一个 nonce
            Nonce::next(index(), nonce(i), reserveCount, nonceMask());
        }
    }

    // 对齐内存到 8 字节边界
    alignas(8) uint8_t m_blobs[2][Job::kMaxBlobSize * N]{};
    Job m_jobs[2];
    uint32_t m_rounds[2] = { 0, 0 };
    uint64_t m_nonce_mask[2] = { 0, 0 };
    uint64_t m_sequence  = 0;
    uint8_t m_index      = 0;
};

// 返回当前索引位置的 nonce 指针
template<>
inline uint32_t *xmrig::WorkerJob<1>::nonce(size_t)
{
    return reinterpret_cast<uint32_t*>(blob() + nonceOffset());
}

// 进行下一轮计算
template<>
inline bool xmrig::WorkerJob<1>::nextRound(uint32_t rounds, uint32_t roundSize)
{
    // 当前索引位置的轮数加一
    m_rounds[index()]++;

    // 获取当前索引位置的 nonce 指针
    uint32_t* n = nonce();

    // 如果当前轮数是指定轮数的倍数
    if ((m_rounds[index()] & (rounds - 1)) == 0) {
        // 生成下一个 nonce
        if (!Nonce::next(index(), n, rounds * roundSize, nonceMask())) {
            return false;
        }
        // 如果 nonce 大小为 8 字节，则进行特殊处理
        if (nonceSize() == sizeof(uint64_t)) {
            writeUnaligned(m_jobs[index()].nonce() + 1, readUnaligned(n + 1));
        }
    }
    else {
        // 对当前 nonce 进行加法运算
        writeUnaligned(n, readUnaligned(n) + roundSize);
    }

    return true;
}

// 保存作业信息和相关数据
template<>
inline void xmrig::WorkerJob<1>::save(const Job &job, uint32_t reserveCount, Nonce::Backend backend)
{
    // 设置当前索引位置
    m_index           = job.index();
    // 保存作业信息到对应索引位置
    m_jobs[index()]   = job;
    // 初始化当前索引位置的轮数为 0
    m_rounds[index()] = 0;
    // 保存当前索引位置的 nonce 掩码
    m_nonce_mask[index()] = job.nonceMask();

    // 设置作业的后端类型
    m_jobs[index()].setBackend(backend);

    // 复制作业数据到对应索引位置的内存中
    memcpy(blob(), job.blob(), job.size());
    // 生成下一个 nonce
    Nonce::next(index(), nonce(), reserveCount, nonceMask());
}

} // namespace xmrig

#endif /* XMRIG_WORKERJOB_H */
```