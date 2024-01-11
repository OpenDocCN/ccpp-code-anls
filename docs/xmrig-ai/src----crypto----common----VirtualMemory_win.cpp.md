# `xmrig\src\crypto\common\VirtualMemory_win.cpp`

```
/* XMRig
 * 版权所有 (c) 2018-2020 tevador     <tevador@gmail.com>
 * 版权所有 (c) 2018-2021 SChernykh   <https://github.com/SChernykh>
 * 版权所有 (c) 2016-2021 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以根据 GNU 通用公共许可证的条款重新分发或修改它，
 *   该许可证由自由软件基金会发布，无论是许可证的第3版还是
 *   （在您的选择下）任何以后的版本。
 *
 *   本程序是希望它有用的分发，
 *   但没有任何保证；甚至没有适销性或特定用途的暗示保证。请参阅
 *   GNU 通用公共许可证以获取更多详细信息。
 *
 *   您应该已经收到了 GNU 通用公共许可证的副本
 *   与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */


#include <winsock2.h>
#include <windows.h>
#include <ntsecapi.h>
#include <tchar.h>


#include "crypto/common/VirtualMemory.h"
#include "base/io/log/Log.h"
#include "crypto/common/portable/mm_malloc.h"


#ifdef XMRIG_SECURE_JIT
#   define SECURE_PAGE_EXECUTE_READWRITE PAGE_READWRITE
#else
#   define SECURE_PAGE_EXECUTE_READWRITE PAGE_EXECUTE_READWRITE
#endif


namespace xmrig {


static bool hugepagesAvailable = false;


/*****************************************************************
SetLockPagesPrivilege: a function to obtain or
release the privilege of locking physical pages.

Inputs:

HANDLE hProcess: Handle for the process for which the
privilege is needed

BOOL bEnable: Enable (TRUE) or disable?

Return value: TRUE indicates success, FALSE failure.

*****************************************************************/
/**
 * AWE Example: https://msdn.microsoft.com/en-us/library/windows/desktop/aa366531(v=vs.85).aspx
 * Creating a File Mapping Using Large Pages: https://msdn.microsoft.com/en-us/library/aa366543(VS.85).aspx
 */
static BOOL SetLockPagesPrivilege() {
    # 声明一个句柄变量用于存储进程令牌
    HANDLE token;

    # 打开当前进程的令牌，获取对令牌的调整特权和查询特权的访问权限
    if (!OpenProcessToken(GetCurrentProcess(), TOKEN_ADJUST_PRIVILEGES | TOKEN_QUERY, &token)) {
        return FALSE;
    }

    # 声明一个特权结构体变量，并设置特权数量为1
    TOKEN_PRIVILEGES tp;
    tp.PrivilegeCount = 1;
    tp.Privileges[0].Attributes = SE_PRIVILEGE_ENABLED;

    # 查找特权名称对应的特权LUID，并存储到特权结构体中
    if (!LookupPrivilegeValue(nullptr, SE_LOCK_MEMORY_NAME, &(tp.Privileges[0].Luid))) {
        return FALSE;
    }

    # 调整令牌的特权，使其具有锁定内存的特权
    BOOL rc = AdjustTokenPrivileges(token, FALSE, (PTOKEN_PRIVILEGES) &tp, 0, nullptr, nullptr);
    if (!rc || GetLastError() != ERROR_SUCCESS) {
        return FALSE;
    }

    # 关闭令牌句柄
    CloseHandle(token);

    # 返回操作成功
    return TRUE;
// 将字符串转换为LSA_UNICODE_STRING类型
static LSA_UNICODE_STRING StringToLsaUnicodeString(LPCTSTR string) {
    LSA_UNICODE_STRING lsaString;

    // 计算字符串长度
    const auto dwLen = (DWORD) wcslen(string);
    // 设置LSA_UNICODE_STRING的Buffer为输入字符串
    lsaString.Buffer = (LPWSTR) string;
    // 设置LSA_UNICODE_STRING的Length为字符串长度乘以WCHAR的字节数
    lsaString.Length = (USHORT)((dwLen) * sizeof(WCHAR));
    // 设置LSA_UNICODE_STRING的MaximumLength为字符串长度加一乘以WCHAR的字节数
    lsaString.MaximumLength = (USHORT)((dwLen + 1) * sizeof(WCHAR));
    // 返回LSA_UNICODE_STRING对象
    return lsaString;
}

// 获取Lock Pages权限
static BOOL ObtainLockPagesPrivilege() {
    HANDLE token;
    PTOKEN_USER user = nullptr;

    // 打开当前进程的访问令牌
    if (OpenProcessToken(GetCurrentProcess(), TOKEN_QUERY, &token)) {
        DWORD size = 0;

        // 获取令牌信息的大小
        GetTokenInformation(token, TokenUser, nullptr, 0, &size);
        if (size) {
            user = (PTOKEN_USER) LocalAlloc(LPTR, size);
        }

        // 获取令牌信息
        GetTokenInformation(token, TokenUser, user, size, &size);
        // 关闭令牌
        CloseHandle(token);
    }

    // 如果未获取到用户信息，则返回FALSE
    if (!user) {
        return FALSE;
    }

    LSA_HANDLE handle;
    LSA_OBJECT_ATTRIBUTES attributes;
    ZeroMemory(&attributes, sizeof(attributes));

    BOOL result = FALSE;
    // 打开本地安全机构策略
    if (LsaOpenPolicy(nullptr, &attributes, POLICY_ALL_ACCESS, &handle) == 0) {
        // 将字符串转换为LSA_UNICODE_STRING类型
        LSA_UNICODE_STRING str = StringToLsaUnicodeString(_T(SE_LOCK_MEMORY_NAME));

        // 向用户的安全标识符添加权限
        if (LsaAddAccountRights(handle, user->User.Sid, &str, 1) == 0) {
            LOG_NOTICE("Huge pages support was successfully enabled, but reboot required to use it");
            result = TRUE;
        }

        // 关闭本地安全机构策略
        LsaClose(handle);
    }

    // 释放内存
    LocalFree(user);
    return result;
}

// 尝试设置Lock Pages权限
static BOOL TrySetLockPagesPrivilege() {
    // 如果成功设置Lock Pages权限，则返回TRUE
    if (SetLockPagesPrivilege()) {
        return TRUE;
    }

    // 获取Lock Pages权限并设置
    return ObtainLockPagesPrivilege() && SetLockPagesPrivilege();
}

// 检查是否支持Hugepages
bool xmrig::VirtualMemory::isHugepagesAvailable()
{
    return hugepagesAvailable;
}

// 检查是否支持1GB页面
bool xmrig::VirtualMemory::isOneGbPagesAvailable()
{
    return false;
}

// 保护内存区域为读写
bool xmrig::VirtualMemory::protectRW(void *p, size_t size)
{
    DWORD oldProtect;

    // 设置内存区域为可读写
    return VirtualProtect(p, size, PAGE_READWRITE, &oldProtect) != 0;
}
# 设置内存区域可读写执行
bool xmrig::VirtualMemory::protectRWX(void *p, size_t size)
{
    DWORD oldProtect;
    # 调用系统API设置内存区域的保护属性为可读写执行
    return VirtualProtect(p, size, PAGE_EXECUTE_READWRITE, &oldProtect) != 0;
}


# 设置内存区域可读执行
bool xmrig::VirtualMemory::protectRX(void *p, size_t size)
{
    DWORD oldProtect;
    # 调用系统API设置内存区域的保护属性为可读执行
    return VirtualProtect(p, size, PAGE_EXECUTE_READ, &oldProtect) != 0;
}


# 分配可执行内存
void *xmrig::VirtualMemory::allocateExecutableMemory(size_t size, bool hugePages)
{
    void* result = nullptr;
    # 如果需要使用大页内存
    if (hugePages) {
        # 调用系统API分配可执行内存，使用大页内存
        result = VirtualAlloc(nullptr, align(size), MEM_COMMIT | MEM_RESERVE | MEM_LARGE_PAGES, SECURE_PAGE_EXECUTE_READWRITE);
    }
    # 如果不使用大页内存
    if (!result) {
        # 调用系统API分配可执行内存
        result = VirtualAlloc(nullptr, size, MEM_COMMIT | MEM_RESERVE, SECURE_PAGE_EXECUTE_READWRITE);
    }
    # 返回分配的内存地址
    return result;
}


# 分配大页内存
void *xmrig::VirtualMemory::allocateLargePagesMemory(size_t size)
{
    const size_t min = GetLargePageMinimum();
    void *mem        = nullptr;
    # 如果系统支持大页内存
    if (min > 0) {
        # 调用系统API分配大页内存
        mem = VirtualAlloc(nullptr, align(size, min), MEM_COMMIT | MEM_RESERVE | MEM_LARGE_PAGES, PAGE_READWRITE);
    }
    # 返回分配的内存地址
    return mem;
}


# 分配1GB页内存
void *xmrig::VirtualMemory::allocateOneGbPagesMemory(size_t)
{
    return nullptr;
}


# 刷新指令缓存
void xmrig::VirtualMemory::flushInstructionCache(void *p, size_t size)
{
    # 调用系统API刷新指定内存区域的指令缓存
    ::FlushInstructionCache(GetCurrentProcess(), p, size);
}


# 释放大页内存
void xmrig::VirtualMemory::freeLargePagesMemory(void *p, size_t)
{
    # 调用系统API释放大页内存
    VirtualFree(p, 0, MEM_RELEASE);
}


# 初始化操作系统相关设置
void xmrig::VirtualMemory::osInit(size_t hugePageSize)
{
    # 如果需要使用大页内存
    if (hugePageSize) {
        # 尝试获取锁定页面的特权
        hugepagesAvailable = TrySetLockPagesPrivilege();
    }
}


# 分配大页内存
bool xmrig::VirtualMemory::allocateLargePagesMemory()
{
    # 分配大页内存，并将结果转换为uint8_t类型
    m_scratchpad = static_cast<uint8_t*>(allocateLargePagesMemory(m_size));
    # 如果成功分配大页内存
    if (m_scratchpad) {
        # 设置标志位为使用大页内存
        m_flags.set(FLAG_HUGEPAGES, true);
        # 返回成功
        return true;
    }
    # 返回失败
    return false;
}

# 分配1GB页内存
bool xmrig::VirtualMemory::allocateOneGbPagesMemory()
{
    # 未实现分配1GB页内存，直接返回失败
    m_scratchpad = nullptr;
    return false;
}


# 释放大页内存
void xmrig::VirtualMemory::freeLargePagesMemory()
{
    # 释放大页内存
    freeLargePagesMemory(m_scratchpad, m_size);
}
```