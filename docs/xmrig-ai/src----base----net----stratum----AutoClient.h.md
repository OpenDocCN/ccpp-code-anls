# `xmrig\src\base\net\stratum\AutoClient.h`

```
/* XMRig
 * 版权所有（c）2018-2021 SChernykh   <https://github.com/SChernykh>
 * 版权所有（c）2016-2021 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以重新发布或修改它
 *   根据由自由软件基金会发布的GNU通用公共许可证的条款，
 *   许可证的版本为3，或者（根据您的选择）任何以后的版本。
 *
 *   本程序是希望它有用的，
 *   但没有任何保证；甚至没有暗示的保证
 *   适销性或特定用途的保证。有关更多详细信息，请参见
 *   GNU通用公共许可证。
 *
 *   您应该已经收到了GNU通用公共许可证的副本
 *   与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#ifndef XMRIG_AUTOCLIENT_H
#define XMRIG_AUTOCLIENT_H


#include "base/net/stratum/EthStratumClient.h"


#include <utility>


namespace xmrig {


class AutoClient : public EthStratumClient
{
public:
    XMRIG_DISABLE_COPY_MOVE_DEFAULT(AutoClient)

    AutoClient(int id, const char *agent, IClientListener *listener);
    ~AutoClient() override = default;

protected:
    inline void login() override    { Client::login(); }

    bool handleResponse(int64_t id, const rapidjson::Value &result, const rapidjson::Value &error) override;
    bool parseLogin(const rapidjson::Value &result, int *code) override;
    int64_t submit(const JobResult &result) override;
    void parseNotification(const char *method, const rapidjson::Value &params, const rapidjson::Value &error) override;

private:
    enum Mode {
        DEFAULT_MODE,
        ETH_MODE
    };

    Mode m_mode = DEFAULT_MODE;
};


} /* namespace xmrig */


#endif /* XMRIG_AUTOCLIENT_H */
```