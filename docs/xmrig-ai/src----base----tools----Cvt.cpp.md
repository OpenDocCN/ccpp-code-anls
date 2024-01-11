# `xmrig\src\base\tools\Cvt.cpp`

```
/* XMRig
 * 版权所有（c）2013-2020 Frank Denis <j at pureftpd dot org>
 * 版权所有（c）2018-2021 SChernykh   <https://github.com/SChernykh>
 * 版权所有（c）2016-2021 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 * 本程序是自由软件：您可以重新发布或修改它
 * 根据由自由软件基金会发布的GNU通用公共许可证的条款，
 * 无论是许可证的第3版还是（在您的选择下）任何以后的版本。
 *
 * 本程序是希望它有用的分发，
 * 但没有任何保证；甚至没有对特定目的的隐含保证。请参阅
 * GNU通用公共许可证以获取更多详细信息。
 *
 * 您应该已经收到了GNU通用公共许可证的副本
 * 与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#include "base/tools/Cvt.h"
#include "3rdparty/rapidjson/document.h"


#include <cassert>
#include <random>


#ifdef XMRIG_SODIUM
#   include <sodium.h>
#endif


namespace xmrig {


static char *cvt_bin2hex(char *const hex, const size_t hex_maxlen, const unsigned char *const bin, const size_t bin_len)
{
    size_t       i = 0U;
    unsigned int x = 0U;
    int          b = 0;
    int          c = 0;

    if (bin_len >= SIZE_MAX / 2 || hex_maxlen < bin_len * 2U) {
        return nullptr; /* LCOV_EXCL_LINE */
    }

    while (i < bin_len) {
        c = bin[i] & 0xf;
        b = bin[i] >> 4;
        x = (unsigned char) (87U + c + (((c - 10U) >> 8) & ~38U)) << 8 |
            (unsigned char) (87U + b + (((b - 10U) >> 8) & ~38U));
        hex[i * 2U] = (char) x;
        x >>= 8;
        hex[i * 2U + 1U] = (char) x;
        i++;
    }

    if (i * 2U < hex_maxlen) {
        hex[i * 2U] = 0U;
    }

    return hex;
}


#ifndef XMRIG_SODIUM
static std::random_device randomDevice;
static std::mt19937 randomEngine(randomDevice());
// 将十六进制字符串转换为二进制数据
static int cvt_hex2bin(unsigned char *const bin, const size_t bin_maxlen, const char *const hex, const size_t hex_len, const char *const ignore, size_t *const bin_len, const char **const hex_end)
{
    size_t        bin_pos   = 0U;  // 二进制数据位置
    size_t        hex_pos   = 0U;  // 十六进制字符串位置
    int           ret       = 0;    // 返回值
    unsigned char c         = 0U;   // 当前字符
    unsigned char c_acc     = 0U;   // 累积值
    unsigned char c_alpha0  = 0U;   // 字母值
    unsigned char c_alpha   = 0U;   // 字母值
    unsigned char c_num0    = 0U;   // 数字值
    unsigned char c_num     = 0U;   // 数字值
    unsigned char c_val     = 0U;   // 当前值
    unsigned char state     = 0U;   // 状态

    while (hex_pos < hex_len) {
        c        = (unsigned char) hex[hex_pos];  // 获取当前字符
        c_num    = c ^ 48U;  // 计算数字值
        c_num0   = (c_num - 10U) >> 8;  // 判断是否为数字
        c_alpha  = (c & ~32U) - 55U;  // 计算字母值
        c_alpha0 = ((c_alpha - 10U) ^ (c_alpha - 16U)) >> 8;  // 判断是否为字母

        if ((c_num0 | c_alpha0) == 0U) {  // 如果不是数字或字母
            if (ignore != nullptr && state == 0U && strchr(ignore, c) != nullptr) {  // 如果忽略字符不为空且当前状态为0且当前字符在忽略字符中
                hex_pos++;  // 继续下一个字符
                continue;
            }
            break;  // 否则跳出循环
        }

        c_val = (c_num0 & c_num) | (c_alpha0 & c_alpha);  // 计算当前值

        if (bin_pos >= bin_maxlen) {  // 如果二进制数据位置超出最大长度
            ret   = -1;  // 返回错误
            errno = ERANGE;  // 设置错误码
            break;  // 跳出循环
        }

        if (state == 0U) {  // 如果状态为0
            c_acc = c_val * 16U;  // 计算累积值
        } else {
            bin[bin_pos++] = c_acc | c_val;  // 将累积值和当前值组合成一个字节存入二进制数据
        }

        state = ~state;  // 切换状态
        hex_pos++;  // 下一个字符
    }

    if (state != 0U) {  // 如果状态不为0
        hex_pos--;  // 回退一个字符
        errno = EINVAL;  // 设置错误码
        ret = -1;  // 返回错误
    }

    if (ret != 0) {  // 如果返回值不为0
        bin_pos = 0U;  // 重置二进制数据位置
    }

    if (hex_end != nullptr) {  // 如果十六进制结束位置不为空
        *hex_end = &hex[hex_pos];  // 设置结束位置
    } else if (hex_pos != hex_len) {  // 否则如果结束位置不等于十六进制长度
        errno = EINVAL;  // 设置错误码
        ret = -1;  // 返回错误
    }

    if (bin_len != nullptr) {  // 如果二进制长度不为空
        *bin_len = bin_pos;  // 设置二进制长度
    }

    return ret;  // 返回结果
}

#define sodium_hex2bin cvt_hex2bin  // 定义宏
#endif


template<typename T>
inline bool fromHexImpl(T &buf, const char *in, size_t size)
{
    assert(in != nullptr && size > 0);  // 断言输入不为空且大小大于0
    # 如果输入指针为空或者大小为0，则返回false
    if (in == nullptr || size == 0) {
        return false;
    }

    # 调整缓冲区大小为输入大小的一半
    buf.resize(size / 2);

    # 将十六进制字符串转换为二进制数据，并存储到缓冲区中
    return sodium_hex2bin(reinterpret_cast<uint8_t *>(&buf.front()), buf.size(), in, size, nullptr, nullptr, nullptr) == 0;
} // namespace xmrig

// 从十六进制字符串转换为二进制数据
bool xmrig::Cvt::fromHex(Buffer &buf, const char *in, size_t size)
{
    return fromHexImpl(buf, in, size);
}

// 从十六进制字符串转换为二进制数据
bool xmrig::Cvt::fromHex(Buffer &buf, const rapidjson::Value &value)
{
    // 如果值不是字符串，则返回false
    if (!value.IsString()) {
        return false;
    }

    return fromHexImpl(buf, value.GetString(), value.GetStringLength());
}

// 从十六进制字符串转换为二进制数据
bool xmrig::Cvt::fromHex(std::string &buf, const char *in, size_t size)
{
    return fromHexImpl(buf, in, size);
}

// 从十六进制字符串转换为二进制数据
bool xmrig::Cvt::fromHex(uint8_t *bin, size_t bin_maxlen, const char *hex, size_t hex_len)
{
    // 断言十六进制字符串和长度不为空
    assert(hex != nullptr && hex_len > 0);
    if (hex == nullptr || hex_len == 0) {
        return false;
    }

    // 使用libsodium库将十六进制字符串转换为二进制数据
    return sodium_hex2bin(bin, bin_maxlen, hex, hex_len, nullptr, nullptr, nullptr) == 0;
}

// 从十六进制字符串转换为二进制数据
bool xmrig::Cvt::fromHex(uint8_t *bin, size_t bin_maxlen, const rapidjson::Value &value)
{
    // 如果值不是字符串，则返回false
    if (!value.IsString()) {
        return false;
    }

    return fromHex(bin, bin_maxlen, value.GetString(), value.GetStringLength());
}

// 从十六进制字符串转换为二进制数据
xmrig::Buffer xmrig::Cvt::fromHex(const char *in, size_t size)
{
    Buffer buf;
    // 如果转换失败，则返回空的Buffer对象
    if (!fromHex(buf, in, size)) {
        return {};
    }

    return buf;
}

// 从二进制数据转换为十六进制字符串
bool xmrig::Cvt::toHex(char *hex, size_t hex_maxlen, const uint8_t *bin, size_t bin_len)
{
    // 使用libsodium库将二进制数据转换为十六进制字符串
    return cvt_bin2hex(hex, hex_maxlen, bin, bin_len) != nullptr;
}

// 生成指定大小的随机二进制数据
xmrig::Buffer xmrig::Cvt::randomBytes(const size_t size)
{
    Buffer buf(size);

#   ifndef XMRIG_SODIUM
    // 如果未使用libsodium库，则使用C++的随机数生成器生成随机数据
    std::uniform_int_distribution<> dis(0, 255);

    for (size_t i = 0; i < size; ++i) {
        buf[i] = static_cast<char>(dis(randomEngine));
    }
#   else
    // 使用libsodium库生成随机数据
    randombytes_buf(buf.data(), size);
#   endif

    return buf;
}

// 将二进制数据转换为十六进制字符串
rapidjson::Value xmrig::Cvt::toHex(const Buffer &data, rapidjson::Document &doc)
{
    return toHex(data.data(), data.size(), doc);
}

// 将二进制数据转换为十六进制字符串
rapidjson::Value xmrig::Cvt::toHex(const Span &data, rapidjson::Document &doc)
{
    return toHex(data.data(), data.size(), doc);
}
// 将输入的字符串转换为十六进制表示的 JSON 值
rapidjson::Value xmrig::Cvt::toHex(const std::string &data, rapidjson::Document &doc)
{
    return toHex(reinterpret_cast<const uint8_t *>(data.data()), data.size(), doc);
}

// 将输入的字节数组转换为十六进制表示的 JSON 值
rapidjson::Value xmrig::Cvt::toHex(const uint8_t *in, size_t size, rapidjson::Document &doc)
{
    return toHex(in, size).toJSON(doc);
}

// 将输入的字节数组转换为十六进制表示的字符串
xmrig::String xmrig::Cvt::toHex(const uint8_t *in, size_t size)
{
    // 断言输入数组不为空且大小大于0
    assert(in != nullptr && size > 0);
    if (in == nullptr || size == 0) {
        return {};
    }

    // 计算十六进制表示的最大长度
    const size_t hex_maxlen = size * 2 + 1;

    // 创建存储结果的字符数组
    char *buf = new char[hex_maxlen];
    // 如果转换失败，则释放内存并返回空字符串
    if (!toHex(buf, hex_maxlen, in, size)) {
        delete [] buf;
        return {};
    }

    // 返回十六进制表示的字符串
    return buf;
}

// 生成指定大小的随机字节数组
void xmrig::Cvt::randomBytes(void *buf, size_t size)
{
    // 如果未使用 libsodium 库，则使用 C++ 标准库生成随机数
#   ifndef XMRIG_SODIUM
    std::uniform_int_distribution<> dis(0, 255);
    for (size_t i = 0; i < size; ++i) {
        static_cast<uint8_t *>(buf)[i] = static_cast<char>(dis(randomEngine));
    }
#   else
    // 使用 libsodium 库生成随机字节数组
    randombytes_buf(buf, size);
#   endif
}
```