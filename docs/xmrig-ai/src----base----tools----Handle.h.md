# `xmrig\src\base\tools\Handle.h`

```cpp
/* XMRig
 * 版权所有（c）2018-2020 SChernykh   <https://github.com/SChernykh>
 * 版权所有（c）2016-2020 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以重新发布或修改它
 *   根据GNU通用公共许可证的条款发布，由
 *   自由软件基金会发布的许可证的第3版，或者
 *   （在您的选择下）任何以后的版本。
 *
 *   本程序是希望它有用，
 *   但没有任何保证；甚至没有暗示的保证
 *   适销性或特定用途的保证。请参阅
 *   GNU通用公共许可证以获取更多详细信息。
 *
 *   您应该已经收到了GNU通用公共许可证的副本
 *   与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#ifndef XMRIG_HANDLE_H
#define XMRIG_HANDLE_H


#include <uv.h>


namespace xmrig {


class Handle
{
public:
    template<typename T>
    static inline void close(T handle)
    {
        if (handle) {
            deleteLater(handle);
        }
    }


    template<typename T>
    static inline void deleteLater(T handle)
    {
        if (uv_is_closing(reinterpret_cast<uv_handle_t *>(handle))) {
            return;
        }

        uv_close(reinterpret_cast<uv_handle_t *>(handle), [](uv_handle_t *handle) { delete reinterpret_cast<T>(handle); });
    }
};


template<>
inline void Handle::close(uv_timer_t *handle)
{
    if (handle) {
        uv_timer_stop(handle);
        deleteLater(handle);
    }
}


template<>
inline void Handle::close(uv_signal_t *handle)
{
    if (handle) {
        uv_signal_stop(handle);
        deleteLater(handle);
    }
}


template<>
inline void Handle::close(uv_fs_event_t *handle)
{
    if (handle) {
        uv_fs_event_stop(handle);
        deleteLater(handle);
    }
}


} /* namespace xmrig */


#endif /* XMRIG_HANDLE_H */
```