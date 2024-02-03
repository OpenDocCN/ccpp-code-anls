# `xmrig\src\net\JobResult.h`

```cpp
/*
 * XMRig
 * 版权所有 2010      Jeff Garzik <jgarzik@pobox.com>
 * 版权所有 2012-2014 pooler      <pooler@litecoinpool.org>
 * 版权所有 2014      Lucas Jones <https://github.com/lucasjones>
 * 版权所有 2014-2016 Wolf9466    <https://github.com/OhGodAPet>
 * 版权所有 2016      Jay D Dee   <jayddee246@gmail.com>
 * 版权所有 2017-2018 XMR-Stak    <https://github.com/fireice-uk>, <https://github.com/psychocrypt>
 * 版权所有 2018      Lee Clagett <https://github.com/vtnerd>
 * 版权所有 2018-2020 SChernykh   <https://github.com/SChernykh>
 * 版权所有 2016-2020 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 * 本程序是自由软件：您可以重新发布和/或修改
 * 根据GNU通用公共许可证的条款发布，由
 * 自由软件基金会发布的许可证的第3版，或者
 * （在您的选择）任何以后的版本。
 *
 * 本程序是希望它有用
 * 但没有任何保证；甚至没有暗示的保证
 * 适销性或特定用途的保证。请参阅
 * GNU通用公共许可证以获取更多详细信息。
 *
 * 您应该已经收到GNU通用公共许可证的副本
 * 与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#ifndef XMRIG_JOBRESULT_H
#define XMRIG_JOBRESULT_H


#include <memory.h>
#include <cstdint>


#include "base/tools/String.h"
#include "base/net/stratum/Job.h"


namespace xmrig {


class JobResult
{
public:
    // 禁用默认构造函数
    JobResult() = delete;

    // 构造函数，初始化作业结果对象
    inline JobResult(const Job &job, uint64_t nonce, const uint8_t *result, const uint8_t* header_hash = nullptr, const uint8_t *mix_hash = nullptr, const uint8_t* miner_signature = nullptr) :
        // 设置算法
        algorithm(job.algorithm()),
        // 设置索引
        index(job.index()),
        // 设置客户端ID
        clientId(job.clientId()),
        // 设置作业ID
        jobId(job.id()),
        // 设置后端
        backend(job.backend()),
        // 设置随机数
        nonce(nonce),
        // 设置难度
        diff(job.diff())
    {
        // 将 result 数组的内容复制到 m_result 数组中
        memcpy(m_result, result, sizeof(m_result));
    
        // 如果 header_hash 不为空，则将其内容复制到 m_headerHash 数组中
        if (header_hash) {
            memcpy(m_headerHash, header_hash, sizeof(m_headerHash));
        }
    
        // 如果 mix_hash 不为空，则将其内容复制到 m_mixHash 数组中
        if (mix_hash) {
            memcpy(m_mixHash, mix_hash, sizeof(m_mixHash));
        }
    
        // 如果 miner_signature 不为空，则将其内容复制到 m_minerSignature 数组中，并将 m_hasMinerSignature 设置为 true
        if (miner_signature) {
            m_hasMinerSignature = true;
            memcpy(m_minerSignature, miner_signature, sizeof(m_minerSignature));
        }
    }
    
    // 根据给定的 Job 对象初始化 JobResult 对象
    inline JobResult(const Job &job) :
        // 初始化 algorithm 属性为 job 的 algorithm 属性
        algorithm(job.algorithm()),
        // 初始化 index 属性为 job 的 index 属性
        index(job.index()),
        // 初始化 clientId 属性为 job 的 clientId 属性
        clientId(job.clientId()),
        // 初始化 jobId 属性为 job 的 id 属性
        jobId(job.id()),
        // 初始化 backend 属性为 job 的 backend 属性
        backend(job.backend()),
        // 初始化 nonce 属性为 0
        nonce(0),
        // 初始化 diff 属性为 0
        diff(0)
    {
    }
    
    // 返回 m_result 数组的指针，用于读取结果数据
    inline const uint8_t *result() const     { return m_result; }
    // 返回 m_result 数组的第 4 个元素解析后的实际难度值
    inline uint64_t actualDiff() const       { return Job::toDiff(reinterpret_cast<const uint64_t*>(m_result)[3]); }
    // 返回 m_result 数组的指针，用于修改结果数据
    inline uint8_t *result()                 { return m_result; }
    // 返回 m_headerHash 数组的指针，用于读取头哈希数据
    inline const uint8_t *headerHash() const { return m_headerHash; }
    // 返回 m_mixHash 数组的指针，用于读取混合哈希数据
    inline const uint8_t *mixHash() const    { return m_mixHash; }
    
    // 返回 m_minerSignature 数组的指针，如果 m_hasMinerSignature 为 true，则返回 m_minerSignature 数组的指针，否则返回 nullptr
    inline const uint8_t *minerSignature() const { return m_hasMinerSignature ? m_minerSignature : nullptr; }
    
    // algorithm 属性，表示算法
    const Algorithm algorithm;
    // index 属性，表示索引
    const uint8_t index;
    // clientId 属性，表示客户端 ID
    const String clientId;
    // jobId 属性，表示作业 ID
    const String jobId;
    // backend 属性，表示后端
    const uint32_t backend;
    // nonce 属性，表示随机数
    const uint64_t nonce;
    // diff 属性，表示难度
    const uint64_t diff;
private:
    // 用于存储结果的数组，长度为32，初始化为0
    uint8_t m_result[32]     = { 0 };
    // 用于存储头部哈希的数组，长度为32，初始化为0
    uint8_t m_headerHash[32] = { 0 };
    // 用于存储混合哈希的数组，长度为32，初始化为0
    uint8_t m_mixHash[32]    = { 0 };

    // 用于存储矿工签名的数组，长度为64，初始化为0
    uint8_t m_minerSignature[64] = { 0 };
    // 标识是否存在矿工签名，初始化为false
    bool m_hasMinerSignature = false;
};


} /* namespace xmrig */


#endif /* XMRIG_JOBRESULT_H */
```