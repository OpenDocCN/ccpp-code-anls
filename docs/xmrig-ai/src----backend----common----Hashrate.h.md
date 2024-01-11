# `xmrig\src\backend\common\Hashrate.h`

```
#ifndef XMRIG_HASHRATE_H
#define XMRIG_HASHRATE_H

#include <cmath>  // 包含数学函数库
#include <cstddef>  // 包含标准库定义大小的头文件
#include <cstdint>  // 包含标准整数类型的头文件

#include "3rdparty/rapidjson/fwd.h"  // 包含第三方 JSON 库的前向声明
#include "base/tools/Object.h"  // 包含基础工具类的头文件

namespace xmrig {

class Hashrate
{
public:
    XMRIG_DISABLE_COPY_MOVE_DEFAULT(Hashrate)  // 禁用默认的拷贝和移动构造函数

    enum Intervals : size_t {  // 定义枚举类型 Intervals
        ShortInterval  = 10000,  // 短时间间隔为 10000
        MediumInterval = 60000,  // 中等时间间隔为 60000
        LargeInterval  = 900000  // 长时间间隔为 900000
    };

    Hashrate(size_t threads);  // 声明构造函数，传入线程数
    ~Hashrate();  // 声明析构函数

    inline double calc(size_t ms) const                                     { const double data = hashrate(0U, ms); return std::isnormal(data) ? data : 0.0; }  // 计算总的哈希率
    inline double calc(size_t threadId, size_t ms) const                    { return hashrate(threadId + 1, ms); }  // 计算特定线程的哈希率
    inline size_t threads() const                                           { return m_threads > 0U ? m_threads - 1U : 0U; }  // 返回线程数
    inline void add(size_t threadId, uint64_t count, uint64_t timestamp)    { addData(threadId + 1U, count, timestamp); }  // 添加线程的哈希计数和时间戳
    // 在数据中添加指定数量和时间戳的数据
    inline void add(uint64_t count, uint64_t timestamp)                     { addData(0U, count, timestamp); }

    // 计算数据的平均值
    double average() const;

    // 将 double 类型的数据格式化为字符串，并存储到指定的缓冲区中
    static const char *format(double h, char *buf, size_t size);

    // 将 double 类型的数据进行规范化处理，并返回 rapidjson::Value 类型的数据
    static rapidjson::Value normalize(double d);
// 如果定义了 XMRIG_FEATURE_API，则以下函数将可用
rapidjson::Value toJSON(rapidjson::Document &doc) const;
rapidjson::Value toJSON(size_t threadId, rapidjson::Document &doc) const;

// 计算指定索引和时间间隔下的哈希率
private:
double hashrate(size_t index, size_t ms) const;

// 向指定索引的数据中添加计数和时间戳
void addData(size_t index, uint64_t count, uint64_t timestamp);

// 定义静态常量 kBucketSize 和 kBucketMask
constexpr static size_t kBucketSize = 2 << 11;
constexpr static size_t kBucketMask = kBucketSize - 1;

// 线程数、顶部指针、计数和时间戳的指针数组
size_t m_threads;
uint32_t* m_top;
uint64_t** m_counts;
uint64_t** m_timestamps;

// 最早的时间戳和总计数
uint64_t m_earliestTimestamp;
uint64_t m_totalCount;
} // namespace xmrig

#endif /* XMRIG_HASHRATE_H */
```