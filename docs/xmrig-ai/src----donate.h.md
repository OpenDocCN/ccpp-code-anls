# `xmrig\src\donate.h`

```cpp
/*
 * XMRig
 * 版权所有（c）2018-2022 SChernykh   <https://github.com/SChernykh>
 * 版权所有（c）2016-2022 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 * 本程序是自由软件：您可以重新发布或修改它，遵循由自由软件基金会发布的GNU通用公共许可证的条款，无论是许可证的第3版还是（在您的选择下）任何以后的版本。
 *
 * 本程序是基于希望它会有用的前提下分发的，但没有任何保证；甚至没有对适销性或特定用途适用性的暗示保证。更多详情请参阅GNU通用公共许可证。
 *
 * 您应该已经收到了GNU通用公共许可证的副本。如果没有，请参阅<http://www.gnu.org/licenses/>。
 */

#ifndef XMRIG_DONATE_H
#define XMRIG_DONATE_H

/*
 * 开发者捐赠。
 *
 * 您想要捐赠给开发者的哈希算力百分比可以是0%，但支持XMRig开发。
 *
 * 例如，设置为1%的工作原理示例：
 * 您的矿工将在您通常的矿池中进行随机时间的挖矿（范围从49.5到148.5分钟），
 * 然后切换到开发者的矿池1分钟，然后再切换到您的矿池99分钟，
 * 然后再切换到开发者的矿池1分钟；这些循环将持续，直到矿工停止。
 *
 * 仅在第一轮中随机化，以防止在捐赠矿池上产生波动。
 *
 * 切换是即时的，只会在成功连接后发生，因此您不会丢失任何哈希。
 *
 * 如果您计划将捐赠设置为0%，请考虑向我的钱包进行一次性捐赠：
 * XMR: 48edfHu7V9Z84YzzMa6fUueoELZ9ZRXq9VetWzYGzKt52XU5xvqgzYnDK9URnRoJMk1j8nLwEVsaSWJ4fhdUyZijBGUicoD
 */
constexpr const int kDefaultDonateLevel = 1;
constexpr const int kMinimumDonateLevel = 1;

#endif // XMRIG_DONATE_H
```