# `xmrig\src\Summary.h`

```
/* XMRig
 * 版权所有 (c) 2018-2022 SChernykh   <https://github.com/SChernykh>
 * 版权所有 (c) 2016-2022 XMRig       <support@xmrig.com>
 *
 *   本程序是自由软件：您可以重新发布或修改
 *   根据GNU通用公共许可证的条款发布，由
 *   自由软件基金会发布的许可证的第3版，或者
 *   （在您的选择下）任何以后的版本。
 *
 *   本程序是希望它有用，
 *   但没有任何保证；甚至没有暗示的保证
 *   适销性或特定用途的保证。请参阅
 *   GNU通用公共许可证以获取更多详细信息。
 *
 *   您应该已经收到GNU通用公共许可证的副本
 *   与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#ifndef XMRIG_SUMMARY_H
#define XMRIG_SUMMARY_H


namespace xmrig {


class Controller;


class Summary
{
public:
    static void print(Controller *controller);
};


} // namespace xmrig


#endif /* XMRIG_SUMMARY_H */
```