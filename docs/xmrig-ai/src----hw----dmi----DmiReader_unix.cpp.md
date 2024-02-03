# `xmrig\src\hw\dmi\DmiReader_unix.cpp`

```cpp
/*
 * XMRig
 * 版权所有（c）2000-2002 Alan Cox     <alan@redhat.com>
 * 版权所有（c）2005-2020 Jean Delvare <jdelvare@suse.de>
 * 版权所有（c）2018-2021 SChernykh    <https://github.com/SChernykh>
 * 版权所有（c）2016-2021 XMRig        <https://github.com/xmrig>, <support@xmrig.com>
 *
 * 本程序是自由软件：您可以根据GNU通用公共许可证的条款重新分发或修改它，无论是许可证的第3版还是（在您的选择下）任何以后的版本。
 *
 * 本程序是基于希望它有用的情况下分发的，但没有任何保证；甚至没有适销性或特定用途的暗示保证。有关更多详细信息，请参见GNU通用公共许可证。
 *
 * 您应该已经收到了GNU通用公共许可证的副本。如果没有，请参见<http://www.gnu.org/licenses/>。
 */


#include "hw/dmi/DmiReader.h"
#include "hw/dmi/DmiTools.h"


#include <cerrno>
#include <fcntl.h>
#include <sys/mman.h>
#include <sys/stat.h>
#include <sys/types.h>
#include <unistd.h>
#include <cstdio>

#ifdef XMRIG_OS_FREEBSD
#   include <kenv.h>
#endif


#define FLAG_NO_FILE_OFFSET     (1 << 0)


namespace xmrig {


static const char *kMemDevice       = "/dev/mem";
static const char *kSysEntryFile    = "/sys/firmware/dmi/tables/smbios_entry_point";
static const char *kSysTableFile    = "/sys/firmware/dmi/tables/DMI";


static inline void safe_memcpy(void *dest, const void *src, size_t n)
{
#   ifdef XMRIG_ARM
    for (size_t i = 0; i < n; i++) {
        *((uint8_t *)dest + i) = *((const uint8_t *)src + i);
    }
#   else
    memcpy(dest, src, n);
#   endif
}

// 从文件描述符中读取数据到缓冲区中
static int myread(int fd, uint8_t *buf, size_t count, const char *prefix)
{
    ssize_t r = 1;  // 读取的字节数
    size_t r2 = 0;  // 已经读取的字节数
    # 当读取的字节数不等于指定的数量并且读取的字节数不为0时执行循环
    while (r2 != count && r != 0) {
        # 从文件描述符中读取数据到缓冲区中，返回实际读取的字节数
        r = read(fd, buf + r2, count - r2);
        # 如果读取返回-1，表示出错
        if (r == -1) {
            # 如果出错原因不是被中断，则返回-1
            if (errno != EINTR) {
                return -1;
            }
        }
        # 如果读取成功，更新已读取的字节数
        else {
            r2 += r;
        }
    }

    # 如果实际读取的字节数不等于指定的数量，返回-1
    if (r2 != count) {
        return -1;
    }

    # 读取成功，返回0
    return 0;
}

/*
 * 从给定的偏移量开始读取文件的全部内容，最多读取 max_len 字节。
 * 此函数会分配最多 max_len 字节的缓冲区，需要由调用者释放。
 * 这提供了类似于 mem_chunk() 的用法模型。
 *
 * 返回指向分配的缓冲区的指针，如果出错则返回 NULL，并将 max_len 设置为实际读取的长度。
 */
static uint8_t *read_file(off_t base, size_t *max_len, const char *filename)
{
    // 打开文件，以只读方式
    const int fd = open(filename, O_RDONLY);
    // 指向缓冲区的指针
    uint8_t *p   = nullptr;

    // 如果打开文件失败，则返回 NULL
    if (fd == -1) {
        return nullptr;
    }

    // 文件状态结构体
    struct stat statbuf{};
    // 如果获取文件状态成功
    if (fstat(fd, &statbuf) == 0) {
        // 如果基地址大于等于文件大小，则跳转到 out 标签
        if (base >= statbuf.st_size) {
            goto out;
        }

        // 如果 max_len 大于文件大小减去基地址，则将 max_len 设置为文件大小减去基地址
        if (*max_len > static_cast<size_t>(statbuf.st_size) - base) {
            *max_len = statbuf.st_size - base;
        }
    }

    // 如果分配缓冲区失败，则跳转到 out 标签
    if ((p = reinterpret_cast<uint8_t *>(malloc(*max_len))) == nullptr) {
        goto out;
    }

    // 如果设置文件偏移失败，则跳转到 err_free 标签
    if (lseek(fd, base, SEEK_SET) == -1) {
        goto err_free;
    }

    // 如果读取文件失败，则跳转到 out 标签
    if (myread(fd, p, *max_len, filename) == 0) {
        goto out;
    }

err_free:
    // 释放缓冲区
    free(p);
    p = nullptr;

out:
    // 关闭文件
    close(fd);

    // 返回缓冲区指针
    return p;
}


/*
 * 将物理内存块复制到内存缓冲区中。
 * 此函数会分配内存。
 */
static uint8_t *mem_chunk(off_t base, size_t len, const char *devmem)
{
    // 打开设备文件，以只读方式
    const int fd = open(devmem, O_RDONLY);
    // 指向缓冲区的指针
    uint8_t *p   = nullptr;
    // 指向内存映射的指针
    uint8_t *mmp = nullptr;
    // 文件状态结构体
    struct stat statbuf{};

#   ifdef _SC_PAGESIZE
    // 如果定义了 _SC_PAGESIZE，则使用 sysconf 函数获取页面大小
    const off_t mmoffset = base % sysconf(_SC_PAGESIZE);
#   else
    // 否则使用 getpagesize 函数获取页面大小
    const off_t mmoffset = base % getpagesize();
#   endif

    // 如果打开设备文件失败，则返回 NULL
    if (fd == -1) {
        return nullptr;
    }

    // 如果分配缓冲区失败，则跳转到 out 标签
    if ((p = reinterpret_cast<uint8_t *>(malloc(len))) == nullptr) {
        goto out;
    }

    // 如果获取文件状态失败，则跳转到 err_free 标签
    if (fstat(fd, &statbuf) == -1) {
        goto err_free;
    }

    // 如果是普通文件且基地址加上长度大于文件大小，则跳转到 err_free 标签
    if (S_ISREG(statbuf.st_mode) && base + (off_t)len > statbuf.st_size) {
        goto err_free;
    }
    # 使用 mmap 将文件映射到内存中，并返回指向映射区域的指针
    mmp = reinterpret_cast<uint8_t *>(mmap(nullptr, mmoffset + len, PROT_READ, MAP_SHARED, fd, base - mmoffset));
    # 如果映射失败，则跳转到 try_read 标签处
    if (mmp == MAP_FAILED) {
        goto try_read;
    }

    # 从映射区域中安全地复制数据到指定位置
    safe_memcpy(p, mmp + mmoffset, len);
    # 取消内存映射
    munmap(mmp, mmoffset + len);

    # 跳转到 out 标签处
    goto out;
# 尝试读取数据
try_read:
    # 将文件指针移动到指定位置
    if (lseek(fd, base, SEEK_SET) == -1) {
        # 如果移动失败，跳转到错误处理标签
        goto err_free;
    }

    # 从文件中读取指定长度的数据到缓冲区
    if (myread(fd, p, len, devmem) == 0) {
        # 如果读取失败，跳转到结束标签
        goto out;
    }

# 错误处理标签，释放缓冲区内存并将指针置为空
err_free:
    free(p);
    p = nullptr;

# 结束标签，关闭文件
out:
    close(fd);

    # 返回缓冲区指针
    return p;
}


# 计算校验和
static int checksum(const uint8_t *buf, size_t len)
{
    # 初始化校验和为0
    uint8_t sum = 0;

    # 遍历缓冲区中的每个字节，将其加到校验和上
    for (size_t a = 0; a < len; a++) {
        sum += buf[a];
    }

    # 返回校验和是否为0的结果
    return (sum == 0);
}


# 解析 DMI 表
static uint8_t *dmi_table(off_t base, uint32_t &len, const char *devmem, uint32_t flags)
{
    # 如果标志中包含 FLAG_NO_FILE_OFFSET
    if (flags & FLAG_NO_FILE_OFFSET) {
        # 读取文件中的数据到缓冲区，并更新长度
        size_t size = len;
        auto buf    = read_file(0, &size, devmem);
        len         = size;

        # 返回缓冲区指针
        return buf;
    }

    # 否则，根据基地址和长度从内存中读取数据到缓冲区
    return mem_chunk(base, len, devmem);
}


# 解析 SMBIOS3 数据
static uint8_t *smbios3_decode(uint8_t *buf, const char *devmem, uint32_t &size, uint32_t &version, uint32_t flags)
{
    # 如果缓冲区中的第6个字节大于0x20，或者校验和不正确
    if (buf[0x06] > 0x20 || !checksum(buf, buf[0x06])) {
        # 返回空指针
        return nullptr;
    }

    # 计算版本号
    version          = (buf[0x07] << 16) + (buf[0x08] << 8) + buf[0x09];
    # 获取数据大小
    size             = dmi_get<uint32_t>(buf + 0x0C);
    # 获取偏移量
    const u64 offset = dmi_get<u64>(buf + 0x10);

    # 根据偏移量和标志从内存中读取数据到缓冲区
    return dmi_table(((off_t)offset.h << 32) | offset.l, size, devmem, flags);
}


# 解析 SMBIOS 数据
static uint8_t *smbios_decode(uint8_t *buf, const char *devmem, uint32_t &size, uint32_t &version, uint32_t flags)
{
    # 如果缓冲区中的第5个字节大于0x20，或者校验和不正确，或者缓冲区中的一部分不是"_DMI_"，或者"_DMI_"后面的校验和不正确
    if (buf[0x05] > 0x20 || !checksum(buf, buf[0x05]) || memcmp(buf + 0x10, "_DMI_", 5) != 0 || !checksum(buf + 0x10, 0x0F))  {
        # 返回空指针
        return nullptr;
    }

    # 计算版本号
    version = (buf[0x06] << 8) + buf[0x07];

    # 根据版本号进行转换
    switch (version) {
    case 0x021F:
    case 0x0221:
        version = 0x0203;
        break;

    case 0x0233:
        version = 0x0206;
        break;
    }

    # 左移8位，将版本号放在高位
    version = version << 8;
    # 获取数据大小
    size    = dmi_get<uint16_t>(buf + 0x16);

    # 根据偏移量和标志从内存中读取数据到缓冲区
    return dmi_table(dmi_get<uint32_t>(buf + 0x18), size, devmem, flags);
}


# 解析传统 DMI 数据
static uint8_t *legacy_decode(uint8_t *buf, const char *devmem, uint32_t &size, uint32_t &version, uint32_t flags)
{
    # 如果校验和不正确
    if (!checksum(buf, 0x0F)) {
        # 返回空指针
        return nullptr;
    }
    # 从缓冲区中获取版本号的高4位和低4位，并将其左移12位和8位，然后相加，得到版本号
    version = ((buf[0x0E] & 0xF0) << 12) + ((buf[0x0E] & 0x0F) << 8);
    # 从缓冲区中获取大小，使用dmi_get函数将其转换为uint16_t类型
    size    = dmi_get<uint16_t>(buf + 0x06);
    # 使用dmi_get函数从缓冲区中获取偏移量和大小，并将其作为参数创建dmi_table对象
    return dmi_table(dmi_get<uint32_t>(buf + 0x08), size, devmem, flags);
} // namespace xmrig

// 从 EFI 获取地址
static off_t address_from_efi()
{
    #if defined(__linux__)
    // 定义 EFI systab 文件指针和文件名
    FILE *efi_systab;
    const char *filename;
    // 定义行缓冲区和地址变量
    char linebuf[64];
    off_t address = 0;
    #elif defined(XMRIG_OS_FREEBSD)
    // 定义地址字符串
    char addrstr[KENV_MVALLEN + 1];
    #endif

    #if defined(__linux__)
    // 如果无法打开 EFI systab 文件，则返回 EFI_NOT_FOUND
    if ((efi_systab = fopen(filename = "/sys/firmware/efi/systab", "r")) == nullptr && (efi_systab = fopen(filename = "/proc/efi/systab", "r")) == nullptr) {
        return EFI_NOT_FOUND;
    }

    // 初始化地址为 EFI_NO_SMBIOS
    address = EFI_NO_SMBIOS;
    // 逐行读取 EFI systab 文件
    while ((fgets(linebuf, sizeof(linebuf) - 1, efi_systab)) != nullptr) {
        // 在行缓冲区中查找等号，并将其替换为字符串结束符
        char *addrp = strchr(linebuf, '=');
        *(addrp++) = '\0';
        // 如果行缓冲区中的字符串是 "SMBIOS3" 或 "SMBIOS"，则将地址设置为 addrp 所指向的地址
        if (strcmp(linebuf, "SMBIOS3") == 0 || strcmp(linebuf, "SMBIOS") == 0) {
            address = strtoull(addrp, nullptr, 0);
            break;
        }
    }

    // 关闭 EFI systab 文件
    fclose(efi_systab);

    // 返回地址
    return address;
    #elif defined(XMRIG_OS_FREEBSD)
    // 如果无法获取 hint.smbios.0.mem 的值，则返回 EFI_NOT_FOUND
    if (kenv(KENV_GET, "hint.smbios.0.mem", addrstr, sizeof(addrstr)) == -1) {
        return EFI_NOT_FOUND;
    }

    // 将地址字符串转换为地址
    return strtoull(addrstr, nullptr, 0);
    #endif

    // 返回 EFI_NOT_FOUND
    return EFI_NOT_FOUND;
}

// 读取 DMI
bool xmrig::DmiReader::read()
{
    // 初始化大小和缓冲区
    size_t size  = 0x20;
    uint8_t *buf = read_file(0, &size, kSysEntryFile);
    uint8_t *smb = nullptr;

    // 如果缓冲区不为空
    if (buf) {
        smb = nullptr;

        // 如果缓冲区大小大于等于 24，并且前五个字节是 "_SM3_"，则使用 smbios3_decode 解码
        if (size >= 24 && memcmp(buf, "_SM3_", 5) == 0) {
            smb = smbios3_decode(buf, kSysTableFile, m_size, m_version, FLAG_NO_FILE_OFFSET);
        }
        // 如果缓冲区大小大于等于 31，并且前四个字节是 "_SM_"，则使用 smbios_decode 解码
        else if (size >= 31 && memcmp(buf, "_SM_", 4) == 0) {
            smb = smbios_decode(buf, kSysTableFile, m_size, m_version, FLAG_NO_FILE_OFFSET);
        }
        // 如果缓冲区大小大于等于 15，并且前五个字节是 "_DMI_"，则使用 legacy_decode 解码
        else if (size >= 15 && memcmp(buf, "_DMI_", 5) == 0) {
            smb = legacy_decode(buf, kSysTableFile, m_size, m_version, FLAG_NO_FILE_OFFSET);
        }

        // 如果成功解码，则返回解码结果
        if (smb) {
            return decode(smb, [smb, buf]() { free(smb); free(buf); });
        }

        // 释放缓冲区
        free(buf);
    }
}
    # 从 EFI 获取地址
    const auto efi = address_from_efi();
    # 如果 EFI 中没有 SMBIOS 信息，则返回 false
    if (efi == EFI_NO_SMBIOS) {
        return false;
    }

    # 如果 EFI 中存在 SMBIOS 信息
    if (efi != EFI_NOT_FOUND) {
        # 从 EFI 中读取 0x20 字节的内存块
        if ((buf = mem_chunk(efi, 0x20, kMemDevice)) == nullptr) {
            return false;
        }

        # 初始化 SMB 指针
        smb = nullptr;

        # 如果内存块的前 5 个字节是 "_SM3_"，则使用 smbios3_decode 解码
        if (memcmp(buf, "_SM3_", 5) == 0) {
            smb = smbios3_decode(buf, kMemDevice, m_size, m_version, 0);
        }
        # 如果内存块的前 4 个字节是 "_SM_"，则使用 smbios_decode 解码
        else if (memcmp(buf, "_SM_", 4) == 0) {
            smb = smbios_decode(buf, kSysTableFile, m_size, m_version, 0);
        }

        # 如果成功解码 SMBIOS 信息，则调用 decode 函数，并在结束时释放内存
        if (smb) {
            return decode(smb, [smb, buf]() { free(smb); free(buf); });
        }

        # 释放内存块
        free(buf);
    }
# 如果是 x86_64 或者 _M_AMD64 架构
if ((buf = mem_chunk(0xF0000, 0x10000, kMemDevice)) == nullptr) {
    return false;
}

smb = nullptr;

# 遍历内存块，查找 _SM3_ 字符串，如果找到则解析为 SMBIOS3 结构
for (off_t fp = 0; fp <= 0xFFE0; fp += 16) {
    if (memcmp(buf + fp, "_SM3_", 5) == 0) {
        smb = smbios3_decode(buf + fp, kMemDevice, m_size, m_version, 0);
    }

    # 如果 smb 不为空，则调用 decode 函数并释放内存
    if (smb) {
        return decode(smb, [smb, buf]() { free(smb); free(buf); });
    }
}

# 如果未找到 _SM3_ 字符串，则继续查找 _SM_ 和 _DMI_ 字符串
for (off_t fp = 0; fp <= 0xFFF0; fp += 16) {
    if (memcmp(buf + fp, "_SM_", 4) == 0 && fp <= 0xFFE0) {
        smb = smbios3_decode(buf + fp, kMemDevice, m_size, m_version, 0);
    }
    else if (!smb && memcmp(buf + fp, "_DMI_", 5) == 0) {
        smb = legacy_decode(buf + fp, kMemDevice, m_size, m_version, 0);
    }

    # 如果 smb 不为空，则调用 decode 函数并释放内存
    if (smb) {
        return decode(smb, [smb, buf]() { free(smb); free(buf); });
    }
}

# 释放内存
free(buf);
# 返回 false
return false;
```