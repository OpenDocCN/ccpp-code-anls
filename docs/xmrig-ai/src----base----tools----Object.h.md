# `xmrig\src\base\tools\Object.h`

```
/* XMRig
 * 版权所有 (c) 2018-2023 SChernykh   <https://github.com/SChernykh>
 * 版权所有 (c) 2016-2023 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以重新发布或修改它
 *   根据由自由软件基金会发布的 GNU 通用公共许可证的条款
 *   其版本为 3 或 (在您的选择下) 任何以后的版本。
 *
 *   本程序是希望它有用的
 *   但没有任何保证；甚至没有暗示的保证
 *   适销性或特定用途的保证。更多细节请参见
 *   GNU 通用公共许可证。
 *
 *   您应该已经收到了 GNU 通用公共许可证的副本
 *   与本程序一起。如果没有，请参见 <http://www.gnu.org/licenses/>。
 */

#ifndef XMRIG_OBJECT_H
#define XMRIG_OBJECT_H


#include <cstddef>
#include <cstdint>
#include <memory>


namespace xmrig {


#define XMRIG_DISABLE_COPY_MOVE(X) \
    X(const X &other)            = delete; \   // 禁用拷贝构造函数
    X(X &&other)                 = delete; \   // 禁用移动构造函数
    X &operator=(const X &other) = delete; \   // 禁用拷贝赋值运算符
    X &operator=(X &&other)      = delete;     // 禁用移动赋值运算符


#define XMRIG_DISABLE_COPY_MOVE_DEFAULT(X) \
    X()                          = delete; \   // 禁用默认构造函数
    X(const X &other)            = delete; \   // 禁用拷贝构造函数
    X(X &&other)                 = delete; \   // 禁用移动构造函数
    X &operator=(const X &other) = delete; \   // 禁用拷贝赋值运算符
    X &operator=(X &&other)      = delete;     // 禁用移动赋值运算符


} // namespace xmrig


#endif // XMRIG_OBJECT_H
```