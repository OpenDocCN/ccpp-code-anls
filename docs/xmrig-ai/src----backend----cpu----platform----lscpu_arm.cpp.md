# `xmrig\src\backend\cpu\platform\lscpu_arm.cpp`

```
/* XMRig
 * 版权所有（C）2018      Riku Voipio <riku.voipio@iki.fi>
 * 版权所有（C）2018-2023 SChernykh   <https://github.com/SChernykh>
 * 版权所有（C）2016-2023 XMRig       <support@xmrig.com>
 *
 *   本程序是自由软件：您可以重新发布或修改它
 *   根据GNU通用公共许可证的条款发布，由
 *   自由软件基金会发布的许可证的第3版，或者
 *   （在您的选择）任何以后的版本。
 *
 *   本程序是希望它有用，
 *   但没有任何保证；甚至没有暗示的保证
 *   适销性或特定用途的保证。请参阅
 *   GNU通用公共许可证以获取更多详细信息。
 *
 *   您应该已经收到了GNU通用公共许可证的副本
 *   与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#include "base/tools/String.h"
#include "3rdparty/fmt/core.h"


#include <cstdio>
#include <cctype>


namespace xmrig {


struct lscpu_desc
{
    String vendor;
    String model;

    inline bool isReady() const { return !vendor.isNull() && !model.isNull(); }
};


struct id_part {
    const int id;
    const char *name;
};


struct hw_impl {
   const int id;
   const id_part *parts;
   const char *name;
};


static const id_part arm_part[] = {
    { 0x810, "ARM810" },
    { 0x920, "ARM920" },
    { 0x922, "ARM922" },
    { 0x926, "ARM926" },
    { 0x940, "ARM940" },
    { 0x946, "ARM946" },
    { 0x966, "ARM966" },
    { 0xa20, "ARM1020" },
    { 0xa22, "ARM1022" },
    { 0xa26, "ARM1026" },
    { 0xb02, "ARM11 MPCore" },
    { 0xb36, "ARM1136" },
    { 0xb56, "ARM1156" },
    { 0xb76, "ARM1176" },
    { 0xc05, "Cortex-A5" },
    { 0xc07, "Cortex-A7" },
    { 0xc08, "Cortex-A8" },
    { 0xc09, "Cortex-A9" },
    { 0xc0d, "Cortex-A17" }, /* 最初是A12 */
    { 0xc0f, "Cortex-A15" },
    { 0xc0e, "Cortex-A17" },
    { 0xc14, "Cortex-R4" },
    { 0xc15, "Cortex-R5" },
    { 0xc17, "Cortex-R7" },
    { 0xc18, "Cortex-R8" },  // 将十六进制数 0xc18 映射为字符串 "Cortex-R8"
    { 0xc20, "Cortex-M0" },  // 将十六进制数 0xc20 映射为字符串 "Cortex-M0"
    { 0xc21, "Cortex-M1" },  // 将十六进制数 0xc21 映射为字符串 "Cortex-M1"
    { 0xc23, "Cortex-M3" },  // 将十六进制数 0xc23 映射为字符串 "Cortex-M3"
    { 0xc24, "Cortex-M4" },  // 将十六进制数 0xc24 映射为字符串 "Cortex-M4"
    { 0xc27, "Cortex-M7" },  // 将十六进制数 0xc27 映射为字符串 "Cortex-M7"
    { 0xc60, "Cortex-M0+" },  // 将十六进制数 0xc60 映射为字符串 "Cortex-M0+"
    { 0xd01, "Cortex-A32" },  // 将十六进制数 0xd01 映射为字符串 "Cortex-A32"
    { 0xd02, "Cortex-A34" },  // 将十六进制数 0xd02 映射为字符串 "Cortex-A34"
    { 0xd03, "Cortex-A53" },  // 将十六进制数 0xd03 映射为字符串 "Cortex-A53"
    { 0xd04, "Cortex-A35" },  // 将十六进制数 0xd04 映射为字符串 "Cortex-A35"
    { 0xd05, "Cortex-A55" },  // 将十六进制数 0xd05 映射为字符串 "Cortex-A55"
    { 0xd06, "Cortex-A65" },  // 将十六进制数 0xd06 映射为字符串 "Cortex-A65"
    { 0xd07, "Cortex-A57" },  // 将十六进制数 0xd07 映射为字符串 "Cortex-A57"
    { 0xd08, "Cortex-A72" },  // 将十六进制数 0xd08 映射为字符串 "Cortex-A72"
    { 0xd09, "Cortex-A73" },  // 将十六进制数 0xd09 映射为字符串 "Cortex-A73"
    { 0xd0a, "Cortex-A75" },  // 将十六进制数 0xd0a 映射为字符串 "Cortex-A75"
    { 0xd0b, "Cortex-A76" },  // 将十六进制数 0xd0b 映射为字符串 "Cortex-A76"
    { 0xd0c, "Neoverse-N1" },  // 将十六进制数 0xd0c 映射为字符串 "Neoverse-N1"
    { 0xd0d, "Cortex-A77" },  // 将十六进制数 0xd0d 映射为字符串 "Cortex-A77"
    { 0xd0e, "Cortex-A76AE" },  // 将十六进制数 0xd0e 映射为字符串 "Cortex-A76AE"
    { 0xd13, "Cortex-R52" },  // 将十六进制数 0xd13 映射为字符串 "Cortex-R52"
    { 0xd15, "Cortex-R82" },  // 将十六进制数 0xd15 映射为字符串 "Cortex-R82"
    { 0xd20, "Cortex-M23" },  // 将十六进制数 0xd20 映射为字符串 "Cortex-M23"
    { 0xd21, "Cortex-M33" },  // 将十六进制数 0xd21 映射为字符串 "Cortex-M33"
    { 0xd40, "Neoverse-V1" },  // 将十六进制数 0xd40 映射为字符串 "Neoverse-V1"
    { 0xd41, "Cortex-A78" },  // 将十六进制数 0xd41 映射为字符串 "Cortex-A78"
    { 0xd42, "Cortex-A78AE" },  // 将十六进制数 0xd42 映射为字符串 "Cortex-A78AE"
    { 0xd43, "Cortex-A65AE" },  // 将十六进制数 0xd43 映射为字符串 "Cortex-A65AE"
    { 0xd44, "Cortex-X1" },  // 将十六进制数 0xd44 映射为字符串 "Cortex-X1"
    { 0xd46, "Cortex-A510" },  // 将十六进制数 0xd46 映射为字符串 "Cortex-A510"
    { 0xd47, "Cortex-A710" },  // 将十六进制数 0xd47 映射为字符串 "Cortex-A710"
    { 0xd48, "Cortex-X2" },  // 将十六进制数 0xd48 映射为字符串 "Cortex-X2"
    { 0xd49, "Neoverse-N2" },  // 将十六进制数 0xd49 映射为字符串 "Neoverse-N2"
    { 0xd4a, "Neoverse-E1" },  // 将十六进制数 0xd4a 映射为字符串 "Neoverse-E1"
    { 0xd4b, "Cortex-A78C" },  // 将十六进制数 0xd4b 映射为字符串 "Cortex-A78C"
    { 0xd4c, "Cortex-X1C" },  // 将十六进制数 0xd4c 映射为字符串 "Cortex-X1C"
    { 0xd4d, "Cortex-A715" },  // 将十六进制数 0xd4d 映射为字符串 "Cortex-A715"
    { 0xd4e, "Cortex-X3" },  // 将十六进制数 0xd4e 映射为字符串 "Cortex-X3"
    { 0xd4f, "Neoverse-V2" },  // 将十六进制数 0xd4f 映射为字符串 "Neoverse-V2"
    { -1, nullptr }  // 将 -1 映射为空指针
// 定义 Broadcom 芯片的 ID 和对应的名称
static const id_part brcm_part[] = {
    { 0x0f, "Brahma-B15" },
    { 0x100, "Brahma-B53" },
    { 0x516, "ThunderX2" },
    { -1, nullptr }
};

// 定义 DEC 芯片的 ID 和对应的名称
static const id_part dec_part[] = {
    { 0xa10, "SA110" },
    { 0xa11, "SA1100" },
    { -1, nullptr }
};

// 定义 Cavium 芯片的 ID 和对应的名称
static const id_part cavium_part[] = {
    { 0x0a0, "ThunderX" },
    { 0x0a1, "ThunderX-88XX" },
    { 0x0a2, "ThunderX-81XX" },
    { 0x0a3, "ThunderX-83XX" },
    { 0x0af, "ThunderX2-99xx" },
    { 0x0b0, "OcteonTX2" },
    { 0x0b1, "OcteonTX2-98XX" },
    { 0x0b2, "OcteonTX2-96XX" },
    { 0x0b3, "OcteonTX2-95XX" },
    { 0x0b4, "OcteonTX2-95XXN" },
    { 0x0b5, "OcteonTX2-95XXMM" },
    { 0x0b6, "OcteonTX2-95XXO" },
    { 0x0b8, "ThunderX3-T110" },
    { -1, nullptr }
};

// 定义 APM 芯片的 ID 和对应的名称
static const id_part apm_part[] = {
    { 0x000, "X-Gene" },
    { -1, nullptr }
};

// 定义 Qualcomm 芯片的 ID 和对应的名称
static const id_part qcom_part[] = {
    { 0x00f, "Scorpion" },
    { 0x02d, "Scorpion" },
    { 0x04d, "Krait" },
    { 0x06f, "Krait" },
    { 0x201, "Kryo" },
    { 0x205, "Kryo" },
    { 0x211, "Kryo" },
    { 0x800, "Falkor-V1/Kryo" },
    { 0x801, "Kryo-V2" },
    { 0x802, "Kryo-3XX-Gold" },
    { 0x803, "Kryo-3XX-Silver" },
    { 0x804, "Kryo-4XX-Gold" },
    { 0x805, "Kryo-4XX-Silver" },
    { 0xc00, "Falkor" },
    { 0xc01, "Saphira" },
    { -1, nullptr }
};

// 定义 Samsung 芯片的 ID 和对应的名称
static const id_part samsung_part[] = {
    { 0x001, "exynos-m1" },
    { 0x002, "exynos-m3" },
    { 0x003, "exynos-m4" },
    { 0x004, "exynos-m5" },
    { -1, nullptr }
};

// 定义 Nvidia 芯片的 ID 和对应的名称
static const id_part nvidia_part[] = {
    { 0x000, "Denver" },
    { 0x003, "Denver 2" },
    { 0x004, "Carmel" },
    { -1, nullptr }
};

// 定义 Marvell 芯片的 ID 和对应的名称
static const id_part marvell_part[] = {
    { 0x131, "Feroceon-88FR131" },
    { 0x581, "PJ4/PJ4b" },
    { 0x584, "PJ4B-MP" },
    { -1, nullptr }
};

// 定义 Faraday 芯片的 ID 和对应的名称
static const id_part faraday_part[] = {
    { 0x526, "FA526" },
    { 0x626, "FA626" },
    { -1, nullptr }
};

// 定义 Intel 芯片的 ID 和对应的名称
static const id_part intel_part[] = {
    { 0x200, "i80200" },
    { 0x210, "PXA250A" },
    { 0x212, "PXA210A" },
    # 创建一个十六进制值到字符串的映射关系
    { 0x242, "i80321-400" },
    { 0x243, "i80321-600" },
    { 0x290, "PXA250B/PXA26x" },
    { 0x292, "PXA210B" },
    { 0x2c2, "i80321-400-B0" },
    { 0x2c3, "i80321-600-B0" },
    { 0x2d0, "PXA250C/PXA255/PXA26x" },
    { 0x2d2, "PXA210C" },
    { 0x411, "PXA27x" },
    { 0x41c, "IPX425-533" },
    { 0x41d, "IPX425-400" },
    { 0x41f, "IPX425-266" },
    { 0x682, "PXA32x" },
    { 0x683, "PXA930/PXA935" },
    { 0x688, "PXA30x" },
    { 0x689, "PXA31x" },
    { 0xb11, "SA1110" },
    { 0xc12, "IPX1200" },
    { -1, nullptr }
    # 结束映射关系，nullptr 表示空指针
// 定义富士通处理器型号和对应的 ID
static const struct id_part fujitsu_part[] = {
    { 0x001, "A64FX" },
    { -1, nullptr }
};

// 定义海思处理器型号和对应的 ID
static const id_part hisi_part[] = {
    { 0xd01, "Kunpeng-920" },    /* aka tsv110 */
    { 0xd40, "Cortex-A76" },    /* HiSilicon uses this ID though advertises A76 */
    { -1, nullptr }
};

// 定义苹果处理器型号和对应的 ID
static const id_part apple_part[] = {
    { 0x022, "M1" },
    { 0x023, "M1" },
    { 0x024, "M1-Pro" },
    { 0x025, "M1-Pro" },
    { 0x028, "M1-Max" },
    { 0x029, "M1-Max" },
    { 0x032, "M2" },
    { 0x033, "M2" },
    { 0x034, "M2-Pro" },
    { 0x035, "M2-Pro" },
    { 0x038, "M2-Max" },
    { 0x039, "M2-Max" },
    { -1, nullptr }
};

// 定义 FT 处理器型号和对应的 ID
static const struct id_part ft_part[] = {
    { 0x660, "FTC660" },
    { 0x661, "FTC661" },
    { 0x662, "FTC662" },
    { 0x663, "FTC663" },
    { -1, nullptr }
};

// 定义安晟处理器型号和对应的 ID
static const struct id_part ampere_part[] = {
    { 0xac3, "Ampere-1" },
    { 0xac4, "Ampere-1a" },
    { -1, nullptr }
};

// 定义处理器实现者和对应的处理器型号和 ID
static const hw_impl hw_implementer[] = {
    { 0x41, arm_part,     "ARM" },
    { 0x42, brcm_part,    "Broadcom" },
    { 0x43, cavium_part,  "Cavium" },
    { 0x44, dec_part,     "DEC" },
    { 0x46, fujitsu_part, "FUJITSU" },
    { 0x48, hisi_part,    "HiSilicon" },
    { 0x4e, nvidia_part,  "Nvidia" },
    { 0x50, apm_part,     "APM" },
    { 0x51, qcom_part,    "Qualcomm" },
    { 0x53, samsung_part, "Samsung" },
    { 0x56, marvell_part, "Marvell" },
    { 0x61, apple_part,   "Apple" },
    { 0x66, faraday_part, "Faraday" },
    { 0x69, intel_part,   "Intel" },
    { 0x70, ft_part,      "Phytium" },
    { 0xc0, ampere_part,  "Ampere" }
};

// 查找函数，用于查找特定模式的字符串并返回对应的值
static bool lookup(char *line, const char *pattern, String &value)
{
    // 如果行为空或者值不为空，则返回 false
    if (!*line || !value.isNull()) {
        return false;
    }

    char *p;
    int len = strlen(pattern);

    // 如果行的前缀与模式不匹配，则返回 false
    if (strncmp(line, pattern, len) != 0) {
        return false;
    }

    // 跳过空白字符
    for (p = line + len; isspace(*p); p++);

    // 如果不是冒号，则返回 false
    if (*p != ':') {
        return false;
    }

    // 跳过冒号后的空白字符
    for (++p; isspace(*p); p++);
    # 如果指针 p 指向的内容为空，则返回 false
    if (!*p) {
        return false;
    }

    # 将指针 v 指向指针 p 的位置
    const char *v = p;

    # 计算字符串 line 的长度，并去除末尾的空格
    len = strlen(line) - 1;
    for (p = line + len; isspace(*(p-1)); p--);
    *p = '\0';

    # 将 value 指向指针 v 的位置
    value = v;

    # 返回 true
    return true;
// 读取基本信息的函数，参数为 lscpu_desc 结构体指针，返回布尔值
static bool read_basicinfo(lscpu_desc *desc)
{
    // 打开 /proc/cpuinfo 文件，只读方式
    auto fp = fopen("/proc/cpuinfo", "r");
    // 如果文件打开失败，返回 false
    if (!fp) {
        return false;
    }

    // 定义一个缓冲区
    char buf[BUFSIZ];
    // 逐行读取文件内容，存入缓冲区
    while (fgets(buf, sizeof(buf), fp) != nullptr) {
        // 如果缓冲区内容包含 "CPU implementer"，则调用 lookup 函数，将结果存入 desc->vendor
        if (!lookup(buf, "CPU implementer", desc->vendor)) {
            // 如果缓冲区内容包含 "CPU part"，则调用 lookup 函数，将结果存入 desc->model
            lookup(buf, "CPU part", desc->model);
        }

        // 如果 desc 已经包含所需信息，跳出循环
        if (desc->isReady()) {
            break;
        }
    }

    // 关闭文件
    fclose(fp);

    // 返回 desc 是否包含所需信息
    return desc->isReady();
}

// ARM CPU 解码函数，参数为 lscpu_desc 结构体指针，返回布尔值
static bool arm_cpu_decode(lscpu_desc *desc)
{
    // 如果 vendor 或 model 不是以 "0x" 开头，返回 false
    if ((strncmp(desc->vendor, "0x", 2) != 0 || strncmp(desc->model, "0x", 2) != 0)) {
        return false;
    }

    // 将 vendor 和 model 转换为整数
    const int vendor = strtol(desc->vendor, nullptr, 0);
    const int model  = strtol(desc->model, nullptr, 0);

    // 遍历 hw_implementer 数组
    for (const auto &impl : hw_implementer) {
        // 如果 id 不匹配，继续下一次循环
        if (impl.id != vendor) {
            continue;
        }

        // 遍历 parts 数组
        for (size_t i = 0; impl.parts[i].id != -1; ++i) {
            // 如果 id 匹配，更新 desc 的 vendor 和 model，并返回 true
            if (impl.parts[i].id == model) {
                desc->vendor = impl.name;
                desc->model  = impl.parts[i].name;

                return true;
            }
        }
    }

    // 没有匹配的信息，返回 false
    return false;
}

// 返回 ARM CPU 名称的函数
String cpu_name_arm()
{
    // 创建 lscpu_desc 结构体
    lscpu_desc desc;
    // 调用 read_basicinfo 和 arm_cpu_decode 函数，如果都成功，返回格式化后的 CPU 名称
    if (read_basicinfo(&desc) && arm_cpu_decode(&desc)) {
        return fmt::format("{} {}", desc.vendor, desc.model).c_str();
    }

    // 如果没有匹配的信息，返回空字符串
    return {};
}
```