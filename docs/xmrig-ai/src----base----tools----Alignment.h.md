# `xmrig\src\base\tools\Alignment.h`

```cpp
/* XMRig
 * 版权所有（c）2018-2022 SChernykh   <https://github.com/SChernykh>
 * 版权所有（c）2016-2022 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以重新发布或修改它
 *   根据由自由软件基金会发布的GNU通用公共许可证的条款
 *   许可证的版本，或（在您的选择）任何以后的版本。
 *
 *   本程序是希望它有用的
 *   但没有任何保证；甚至没有暗示的保证
 *   适销性或特定用途的保证。有关更多详细信息
 *   请参阅GNU通用公共许可证。
 *
 *   您应该已经收到GNU通用公共许可证的副本
 *   与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#ifndef XMRIG_ALIGNMENT_H
#define XMRIG_ALIGNMENT_H


#include <type_traits>
#include <cstring>


namespace xmrig {


template<typename T>
inline T readUnaligned(const T* ptr)
{
    static_assert(std::is_trivially_copyable<T>::value, "T must be trivially copyable");

    T result;
    memcpy(&result, ptr, sizeof(T));
    return result;
}


template<typename T>
inline void writeUnaligned(T* ptr, T data)
{
    static_assert(std::is_trivially_copyable<T>::value, "T must be trivially copyable");

    memcpy(ptr, &data, sizeof(T));
}


} /* namespace xmrig */


#endif /* XMRIG_ALIGNMENT_H */
```