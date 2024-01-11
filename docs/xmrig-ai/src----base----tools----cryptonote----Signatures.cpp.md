# `xmrig\src\base\tools\cryptonote\Signatures.cpp`

```
/* XMRig
 * 版权所有 2012-2013 The Cryptonote developers
 * 版权所有 2014-2021 The Monero Project
 * 版权所有 2018-2021 SChernykh   <https://github.com/SChernykh>
 * 版权所有 2016-2021 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以根据 GNU 通用公共许可证的条款重新分发或修改它，
 *   该许可证由自由软件基金会发布，无论是许可证的第3版还是（在您的选择下）任何以后的版本。
 *
 *   本程序是基于希望它有用而分发的，但没有任何担保；甚至没有暗示的担保。
 *   适销性或特定用途的暗示担保。有关更多详细信息，请参见 GNU 通用公共许可证。
 *
 *   您应该已经收到了 GNU 通用公共许可证的副本。
 *   如果没有，请参阅 <http://www.gnu.org/licenses/>。
 */


#include "base/crypto/keccak.h"
#include "base/tools/cryptonote/Signatures.h"

extern "C" {

#include "base/tools/cryptonote/crypto-ops.h"

}

#include "base/tools/Cvt.h"

#ifdef XMRIG_PROXY_PROJECT
#define PROFILE_SCOPE(x)
#else
#include "crypto/rx/Profiler.h"
#endif


struct ec_scalar { char data[32]; };
struct hash { char data[32]; };
struct ec_point { char data[32]; };
struct signature { ec_scalar c, r; };
struct s_comm { hash h; ec_point key; ec_point comm; };


static inline void random_scalar(ec_scalar& res)
{
    // 不关心偏差或可能的减少后为0：概率约为10^-76，在这个宇宙中不会发生。
    // 性能更重要。毕竟这是一个矿工。
    xmrig::Cvt::randomBytes(res.data, sizeof(res.data));
    sc_reduce32((uint8_t*) res.data);
}


static void hash_to_scalar(const void* data, size_t length, ec_scalar& res)
{
    xmrig::keccak((const uint8_t*) data, length, (uint8_t*) &res, sizeof(res));
    sc_reduce32((uint8_t*) &res);
}
// 将派生值和输出索引转换为标量
static void derivation_to_scalar(const uint8_t* derivation, size_t output_index, ec_scalar& res)
{
    // 定义一个结构体，用于存储派生值和输出索引
    struct {
        uint8_t derivation[32];
        uint8_t output_index[(sizeof(size_t) * 8 + 6) / 7];
    } buf;

    // 指向输出索引的末尾
    uint8_t* end = buf.output_index;
    // 将派生值复制到缓冲区
    memcpy(buf.derivation, derivation, sizeof(buf.derivation));

    // 将输出索引转换为标量
    size_t k = output_index;
    while (k >= 0x80) {
        *(end++) = (static_cast<uint8_t>(k) & 0x7F) | 0x80;
        k >>= 7;
    }
    *(end++) = static_cast<uint8_t>(k);

    // 使用哈希函数将缓冲区转换为标量
    hash_to_scalar(&buf, end - reinterpret_cast<uint8_t*>(&buf), res);
}

// 生成签名
void generate_signature(const uint8_t* prefix_hash, const uint8_t* pub, const uint8_t* sec, uint8_t* sig_bytes)
{
    PROFILE_SCOPE(GenerateSignature);

    ge_p3 tmp3;
    ec_scalar k;
    s_comm buf;

    // 将前缀哈希和公钥复制到缓冲区
    memcpy(buf.h.data, prefix_hash, sizeof(buf.h.data));
    memcpy(buf.key.data, pub, sizeof(buf.key.data));

    // 将签名字节转换为签名结构
    signature& sig = *reinterpret_cast<signature*>(sig_bytes);

    // 生成签名
    do {
        random_scalar(k);
        ge_scalarmult_base(&tmp3, (unsigned char*)&k);
        ge_p3_tobytes((unsigned char*)&buf.comm, &tmp3);
        hash_to_scalar(&buf, sizeof(s_comm), sig.c);

        if (!sc_isnonzero((const unsigned char*)sig.c.data)) {
            continue;
        }

        sc_mulsub((unsigned char*)&sig.r, (unsigned char*)&sig.c, sec, (unsigned char*)&k);
    } while (!sc_isnonzero((const unsigned char*)sig.r.data));
}

// 检查签名
bool check_signature(const uint8_t* prefix_hash, const uint8_t* pub, const uint8_t* sig_bytes)
{
    ge_p2 tmp2;
    ge_p3 tmp3;
    ec_scalar c;
    s_comm buf;

    // 将前缀哈希和公钥复制到缓冲区
    memcpy(buf.h.data, prefix_hash, sizeof(buf.h.data));
    memcpy(buf.key.data, pub, sizeof(buf.key.data));

    // 如果公钥无效，则返回false
    if (ge_frombytes_vartime(&tmp3, pub) != 0) {
        return false;
    }

    // 将签名字节转换为签名结构
    const signature& sig = *reinterpret_cast<const signature*>(sig_bytes);
}
    # 检查签名的有效性，如果不满足条件则返回 false
    if (sc_check((const uint8_t*)&sig.c) != 0 || sc_check((const uint8_t*)&sig.r) != 0 || !sc_isnonzero((const uint8_t*)&sig.c)) {
        return false;
    }

    # 使用签名中的 c 和 r 值进行双倍标量乘法运算，结果存储在 tmp2 中
    ge_double_scalarmult_base_vartime(&tmp2, (const uint8_t*)&sig.c, &tmp3, (const uint8_t*)&sig.r);
    # 将 tmp2 转换为字节流，存储在 buf.comm 中
    ge_tobytes((uint8_t*)&buf.comm, &tmp2);

    # 创建一个表示无穷远点的静态常量 infinity
    static const ec_point infinity = { { 1, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0} };
    # 检查 buf.comm 是否等于无穷远点，如果是则返回 false
    if (memcmp(&buf.comm, &infinity, 32) == 0) {
        return false;
    }

    # 对 buf 中的数据进行哈希运算，结果存储在 c 中
    hash_to_scalar(&buf, sizeof(s_comm), c);
    # 计算 c - sig.c 的结果，存储在 c 中
    sc_sub((uint8_t*)&c, (uint8_t*)&c, (const uint8_t*)&sig.c);

    # 检查 c 是否为零，如果是则返回 true，否则返回 false
    return sc_isnonzero((uint8_t*)&c) == 0;
// 生成密钥派生函数，接受两个密钥和两个输出参数
bool generate_key_derivation(const uint8_t* key1, const uint8_t* key2, uint8_t* derivation, uint8_t* view_tag)
{
    // 定义椭圆曲线点
    ge_p3 point;
    ge_p2 point2;
    ge_p1p1 point3;

    // 如果将字节流转换为椭圆曲线点失败，则返回false
    if (ge_frombytes_vartime(&point, key1) != 0) {
        return false;
    }

    // 计算椭圆曲线点的倍数
    ge_scalarmult(&point2, key2, &point);
    // 将椭圆曲线点乘以8
    ge_mul8(&point3, &point2);
    // 将椭圆曲线点转换为另一种表示形式
    ge_p1p1_to_p2(&point2, &point3);
    // 将椭圆曲线点转换为字节流
    ge_tobytes(derivation, &point2);

    // 如果存在视图标签
    if (view_tag) {
        // 定义盐值和盐值大小
        constexpr uint8_t salt[] = "view_tag";
        constexpr size_t SALT_SIZE = sizeof(salt) - 1;

        // 定义缓冲区
        uint8_t buf[SALT_SIZE + 32 + 1];
        // 将盐值和派生值复制到缓冲区
        memcpy(buf, salt, SALT_SIZE);
        memcpy(buf + SALT_SIZE, derivation, 32);

        // 假设输出索引为0
        buf[SALT_SIZE + 32] = 0;

        // 定义视图标签的完整值
        uint8_t view_tag_full[32];
        // 使用Keccak哈希函数计算视图标签的完整值
        xmrig::keccak(buf, sizeof(buf), view_tag_full, sizeof(view_tag_full));
        *view_tag = view_tag_full[0];
    }

    return true;
}

// 导出秘密密钥
void derive_secret_key(const uint8_t* derivation, size_t output_index, const uint8_t* base, uint8_t* derived_key)
{
    // 定义椭圆曲线标量
    ec_scalar scalar;

    // 将派生值转换为标量
    derivation_to_scalar(derivation, output_index, scalar);
    // 将派生密钥添加到基础密钥
    sc_add(derived_key, base, (uint8_t*) &scalar);
}

// 导出公钥
bool derive_public_key(const uint8_t* derivation, size_t output_index, const uint8_t* base, uint8_t* derived_key)
{
    // 定义椭圆曲线标量和点
    ec_scalar scalar;
    ge_p3 point1;
    ge_p3 point2;
    ge_cached point3;
    ge_p1p1 point4;
    ge_p2 point5;

    // 如果将字节流转换为椭圆曲线点失败，则返回false
    if (ge_frombytes_vartime(&point1, base) != 0) {
        return false;
    }

    // 将派生值转换为标量
    derivation_to_scalar(derivation, output_index, scalar);
    // 将基础密钥乘以标量
    ge_scalarmult_base(&point2, (uint8_t*) &scalar);
    // 将椭圆曲线点转换为另一种表示形式
    ge_p3_to_cached(&point3, &point2);
    // 将两个椭圆曲线点相加
    ge_add(&point4, &point1, &point3);
    // 将椭圆曲线点转换为另一种表示形式
    ge_p1p1_to_p2(&point5, &point4);
    // 将椭圆曲线点转换为字节流
    ge_tobytes(derived_key, &point5);

    return true;
}

// 生成视图秘密密钥
void derive_view_secret_key(const uint8_t* spend_secret_key, uint8_t* view_secret_key)
{
    // 使用Keccak哈希函数计算视图秘密密钥
    keccak(spend_secret_key, 32, view_secret_key, 32);
    // 将视图秘密密钥进行模运算
    sc_reduce32(view_secret_key);
}

// 生成公钥和秘密密钥
void generate_keys(uint8_t* pub, uint8_t* sec)
{
    # 调用 random_scalar 函数，传入指向 sec 的指针，并将返回值赋给一个未命名的变量
    random_scalar(*((ec_scalar*)sec));

    # 创建一个名为 point 的 ge_p3 结构体变量
    ge_p3 point;
    # 调用 ge_scalarmult_base 函数，传入指向 point 和 sec 的指针，并将结果存储在 point 中
    ge_scalarmult_base(&point, sec);
    # 调用 ge_p3_tobytes 函数，传入指向 pub 和 point 的指针，将 point 转换为字节并存储在 pub 中
    ge_p3_tobytes(pub, &point);
# 结束当前命名空间的定义
}

# 将私钥转换为公钥的函数
bool secret_key_to_public_key(const uint8_t* sec, uint8_t* pub)
{
    # 检查私钥是否有效，如果无效则返回 false
    if (sc_check(sec) != 0) {
        return false;
    }

    # 定义椭圆曲线点对象
    ge_p3 point;
    # 使用基点和私钥进行标量乘法运算，得到椭圆曲线点
    ge_scalarmult_base(&point, sec);
    # 将椭圆曲线点转换为字节表示的公钥
    ge_p3_tobytes(pub, &point);

    # 返回 true 表示转换成功
    return true;
}

# 结束命名空间定义
} /* namespace xmrig */
```