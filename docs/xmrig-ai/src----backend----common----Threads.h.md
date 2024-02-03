# `xmrig\src\backend\common\Threads.h`

```cpp
/* XMRig
 * 版权所有 (c) 2018-2021 SChernykh   <https://github.com/SChernykh>
 * 版权所有 (c) 2016-2021 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以重新发布或修改它
 *   根据由自由软件基金会发布的 GNU 通用公共许可证的条款，
 *   无论是许可证的第 3 版还是（在您的选择下）任何以后的版本。
 *
 *   本程序是希望它有用的，
 *   但没有任何保证；甚至没有适销性或特定用途的暗示保证。请参阅
 *   GNU 通用公共许可证以获取更多详细信息。
 *
 *   您应该已经收到了 GNU 通用公共许可证的副本
 *   与本程序一起。如果没有，请参阅 <http://www.gnu.org/licenses/>。
 */


#ifndef XMRIG_THREADS_H
#define XMRIG_THREADS_H


#include <map>
#include <set>


#include "3rdparty/rapidjson/fwd.h"
#include "base/crypto/Algorithm.h"
#include "base/tools/String.h"


namespace xmrig {


template <class T>
class Threads
{
public:
    // 检查是否存在指定的配置文件
    inline bool has(const char *profile) const                                         { return m_profiles.count(profile) > 0; }
    // 检查指定算法是否被禁用
    inline bool isDisabled(const Algorithm &algo) const                                { return m_disabled.count(algo) > 0; }
    // 检查是否为空
    inline bool isEmpty() const                                                        { return m_profiles.empty(); }
    // 检查指定算法是否存在
    inline bool isExist(const Algorithm &algo) const                                   { return isDisabled(algo) || m_aliases.count(algo) > 0 || has(algo.name()); }
    // 获取指定算法的配置文件
    inline const T &get(const Algorithm &algo, bool strict = false) const              { return get(profileName(algo, strict)); }
    // 禁用指定算法
    inline void disable(const Algorithm &algo)                                         { m_disabled.insert(algo); }
    // 设置指定算法的别名
    inline void setAlias(const Algorithm &algo, const char *profile)                   { m_aliases[algo] = profile; }
    // 将线程移动到指定的配置文件中
    inline size_t move(const char *profile, T &&threads)
    {
        // 如果配置文件已存在，则返回 0
        if (has(profile)) {
            return 0;
        }

        // 获取线程数量
        const size_t count = threads.count();

        // 如果线程不为空，则将线程移动到指定配置文件中
        if (!threads.isEmpty()) {
            m_profiles.insert({ profile, std::move(threads) });
        }

        // 返回移动的线程数量
        return count;
    }

    // 获取指定配置文件中的线程
    const T &get(const String &profileName) const;

    // 从 JSON 值中读取配置文件
    size_t read(const rapidjson::Value &value);

    // 获取算法对应的配置文件名
    String profileName(const Algorithm &algorithm, bool strict = false) const;

    // 将配置文件转换为 JSON 格式
    void toJSON(rapidjson::Value &out, rapidjson::Document &doc) const;
# 声明私有成员变量，用于存储算法的别名
std::map<Algorithm, String> m_aliases;
# 声明私有成员变量，用于存储算法配置文件
std::map<String, T> m_profiles;
# 声明私有成员变量，用于存储被禁用的算法集合
std::set<Algorithm> m_disabled;
```