# `xmrig\src\base\tools\Arguments.cpp`

```
/*
 * XMRig
 * 版权所有（c）2018-2020 SChernykh   <https://github.com/SChernykh>
 * 版权所有（c）2016-2020 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 * 本程序是自由软件：您可以重新发布或修改它
 * 根据由自由软件基金会发布的GNU通用公共许可证的条款，
 * 无论是许可证的第3版还是（在您的选择下）任何以后的版本。
 *
 * 本程序是希望它有用的分发，
 * 但没有任何保证；甚至没有对适销性或特定用途的隐含保证。请参阅
 * GNU通用公共许可证以获取更多详细信息。
 *
 * 您应该已经收到了GNU通用公共许可证的副本
 * 与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#include <algorithm>
#include <uv.h>

#include "base/tools/Arguments.h"

// 构造函数，初始化参数
xmrig::Arguments::Arguments(int argc, char **argv) :
    m_argv(argv),  // 保存命令行参数
    m_argc(argc)   // 保存命令行参数数量
{
    // 设置 libuv 的参数
    uv_setup_args(argc, argv);

    // 遍历命令行参数，将每个参数添加到数据中
    for (size_t i = 0; i < static_cast<size_t>(argc); ++i) {
        add(argv[i]);
    }
}

// 判断是否存在指定参数
bool xmrig::Arguments::hasArg(const char *name) const
{
    // 如果参数数量为1，表示没有参数
    if (m_argc == 1) {
        return false;
    }

    // 在数据中查找指定参数，返回是否存在
    return std::find(m_data.begin() + 1, m_data.end(), name) != m_data.end();
}

// 获取指定参数的值
const char *xmrig::Arguments::value(const char *key1, const char *key2) const
{
    const size_t size = m_data.size();
    // 如果参数数量小于3，返回空指针
    if (size < 3) {
        return nullptr;
    }

    // 遍历数据，查找指定参数的值
    for (size_t i = 1; i < size - 1; ++i) {
        if (m_data[i] == key1 || (key2 && m_data[i] == key2)) {
            return m_data[i + 1];
        }
    }

    return nullptr;
}

// 添加参数到数据中
void xmrig::Arguments::add(const char *arg)
{
    // 如果参数为空，直接返回
    if (arg == nullptr) {
        return;
    }

    // 获取参数的长度
    const size_t size = strlen(arg);
    # 检查参数长度是否大于4，并且第一个字符是'-'，第二个字符也是'-'
    if (size > 4 && arg[0] == '-' && arg[1] == '-') {
        # 在参数中查找'='字符的位置
        const char *p = strchr(arg, '=');

        # 如果找到了'='字符
        if (p) {
            # 计算键的长度
            const auto keySize = static_cast<size_t>(p - arg);

            # 将键和值分别添加到数据容器中
            m_data.emplace_back(arg, keySize);
            m_data.emplace_back(arg + keySize + 1);

            # 返回
            return;
        }
    }

    # 将参数添加到数据容器中
    m_data.emplace_back(arg);
# 闭合前面的函数定义
```