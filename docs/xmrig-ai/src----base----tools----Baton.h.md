# `xmrig\src\base\tools\Baton.h`

```cpp
/*
 * XMRig
 * 版权所有 (c) 2018-2020 SChernykh   <https://github.com/SChernykh>
 * 版权所有 (c) 2016-2020 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 * 本程序是自由软件：您可以重新发布和/或修改
 * 根据由自由软件基金会发布的GNU通用公共许可证的条款，
 * 许可证的版本为3，或者
 * （在您的选择下）任何更高版本。
 *
 * 本程序是希望它有用，
 * 但没有任何保证；甚至没有暗示的保证
 * 适销性或特定用途的保证。请参阅
 * GNU通用公共许可证以获取更多详细信息。
 *
 * 您应该已经收到了GNU通用公共许可证的副本
 * 与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#ifndef XMRIG_BATON_H
#define XMRIG_BATON_H

namespace xmrig {

template<typename REQ>
class Baton
{
public:
    // 默认构造函数，将请求的数据指针指向当前对象
    inline Baton() { req.data = this; }

    // 请求对象
    REQ req;
};

} /* namespace xmrig */

#endif /* XMRIG_BATON_H */
```