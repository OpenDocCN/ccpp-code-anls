# `xmrig\src\base\tools\cryptonote\Signatures.h`

```
/* XMRig
 * 版权所有 2012-2013 The Cryptonote developers
 * 版权所有 2014-2021 The Monero Project
 * 版权所有 2018-2021 SChernykh   <https://github.com/SChernykh>
 * 版权所有 2016-2021 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以重新发布或修改它
 *   根据由自由软件基金会发布的 GNU 通用公共许可证的条款，
 *   无论是许可证的第 3 版还是（在您的选择下）任何以后的版本。
 *
 *   本程序是希望它有用，
 *   但没有任何保证；甚至没有暗示的保证适用于特定用途。请参阅
 *   GNU 通用公共许可证以获取更多详细信息。
 *
 *   您应该已经收到了 GNU 通用公共许可证的副本
 *   与本程序一起。如果没有，请参阅 <http://www.gnu.org/licenses/>。
 */

#ifndef XMRIG_SIGNATURES_H
#define XMRIG_SIGNATURES_H


#include <cstdint>


namespace xmrig {


void generate_signature(const uint8_t* prefix_hash, const uint8_t* pub, const uint8_t* sec, uint8_t* sig);
bool check_signature(const uint8_t* prefix_hash, const uint8_t* pub, const uint8_t* sig);

bool generate_key_derivation(const uint8_t* key1, const uint8_t* key2, uint8_t* derivation, uint8_t* view_tag);
void derive_secret_key(const uint8_t* derivation, size_t output_index, const uint8_t* base, uint8_t* derived_key);
bool derive_public_key(const uint8_t* derivation, size_t output_index, const uint8_t* base, uint8_t* derived_key);

void derive_view_secret_key(const uint8_t* spend_secret_key, uint8_t* view_secret_key);

void generate_keys(uint8_t* pub, uint8_t* sec);
bool secret_key_to_public_key(const uint8_t* sec, uint8_t* pub);

} /* namespace xmrig */


#endif /* XMRIG_SIGNATURES_H */
```