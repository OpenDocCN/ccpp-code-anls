# `nmap\libpcap\pcap-new.c`

```cpp
/*
 * 版权声明，版权归属及使用条件
 */
/*
 * 版权声明，版权归属及使用条件
 * 版权声明，版权归属及使用条件
 * 版权声明，版权归属及使用条件
 */
/*
 * 版权声明，版权归属及使用条件
 * 1. 源代码的再发行必须保留上述版权声明、条件列表和以下免责声明。
 * 2. 二进制形式的再发行必须在文档和/或其他提供的材料中再现上述版权声明、条件列表和以下免责声明。
 * 3. 不得使用 Politecnico di Torino、CACE Technologies 或其贡献者的名称来认可或推广从本软件衍生的产品，除非事先书面许可。
 */
/*
 * 版权声明，版权归属及使用条件
 * 本软件由版权所有者和贡献者提供"按原样"，不提供任何明示或暗示的担保，包括但不限于适销性和特定用途适用性的暗示担保。在任何情况下，无论是合同责任、严格责任还是侵权（包括疏忽或其他方式）产生的任何损害，版权所有者或贡献者均不对任何直接、间接、附带、特殊、惩罚性或后果性损害（包括但不限于替代商品或服务的采购、使用损失、数据或利润损失、业务中断）负责，即使已被告知可能发生此类损害的可能性。
 */
#ifdef HAVE_CONFIG_H
#include <config.h>
#endif

#include "ftmacros.h"
#include "diag-control.h"
/*
 * sockutils.h可能在Windows上包含<crtdbg.h>，而pcap-int.h将包含portability.h，
 * 而portability.h在Windows上期望<crtdbg.h>已经被包含，因此首先包含sockutils.h。
 */
#include "sockutils.h"
#include "pcap-int.h"    // 用于pcap_t结构的详细信息
#include "pcap-rpcap.h"
#include "rpcap-protocol.h"
#include <errno.h>        // 用于errno变量
#include <stdlib.h>        // 用于malloc()，free()，...
#include <string.h>        // 用于strstr等

#ifndef _WIN32
#include <dirent.h>        // 用于readdir
#endif

/* 在pcap_findalldevs_ex()中使用的字符串标识符 */
#define PCAP_TEXT_SOURCE_FILE "File"
#define PCAP_TEXT_SOURCE_FILE_LEN (sizeof PCAP_TEXT_SOURCE_FILE - 1)
/* 在pcap_findalldevs_ex()中使用的字符串标识符 */
#define PCAP_TEXT_SOURCE_ADAPTER "Network adapter"
#define PCAP_TEXT_SOURCE_ADAPTER_LEN (sizeof "Network adapter" - 1)

/* 在pcap_findalldevs_ex()中使用的字符串标识符 */
#define PCAP_TEXT_SOURCE_ON_LOCAL_HOST "on local host"
#define PCAP_TEXT_SOURCE_ON_LOCAL_HOST_LEN (sizeof PCAP_TEXT_SOURCE_ON_LOCAL_HOST + 1)

/****************************************************
 *                                                  *
 * 函数体                                          *
 *                                                  *
 ****************************************************/

int pcap_findalldevs_ex(const char *source, struct pcap_rmtauth *auth, pcap_if_t **alldevs, char *errbuf)
{
    int type;
    char name[PCAP_BUF_SIZE], path[PCAP_BUF_SIZE], filename[PCAP_BUF_SIZE];
    size_t pathlen;
    size_t stringlen;
    pcap_t *fp;
    char tmpstring[PCAP_BUF_SIZE + 1];        /* 用于将名称和描述从“旧”语法转换为“新”语法 */
    pcap_if_t *lastdev;    /* pcap_if_t列表中的最后一个设备 */
    pcap_if_t *dev;        /* 我们要添加到pcap_if_t列表中的设备 */

    /* 列表开始为空。 */
    (*alldevs) = NULL;
    # 初始化一个变量lastdev，赋值为NULL
    lastdev = NULL;

    # 如果源字符串的长度大于PCAP_BUF_SIZE，则返回错误信息并退出函数
    if (strlen(source) > PCAP_BUF_SIZE)
    {
        snprintf(errbuf, PCAP_ERRBUF_SIZE, "The source string is too long. Cannot handle it correctly.");
        return -1;
    }

    '''
     * 确定源的类型（文件、本地、远程）
     * 如果调用pcap_findalldevs_ex()来列出文件和远程适配器，则会有一些差异。
     * 在第一种情况下，我们必须提供要查找的目录的名称（因此
     * pcap_parsesrcstr()的'name'参数是必需的）。
     * 在第二种情况下，不需要适配器的名称（只需要主机）。因此，我们需要
     * 第一次使用此函数来获取源类型，第二次获取适当的信息，
     * 这取决于源类型。
     '''
    # 解析源字符串，确定源的类型，并将结果存储在type变量中
    if (pcap_parsesrcstr(source, &type, NULL, NULL, NULL, errbuf) == -1)
        return -1;

    # 根据源类型进行不同的处理
    switch (type)
    {
    # 如果源类型为文件
    case PCAP_SRC_FILE:
    {
#ifdef _WIN32
        // 如果是在 Windows 平台下，定义 Windows 特有的文件数据结构和文件句柄
        WIN32_FIND_DATA filedata;
        HANDLE filehandle;
#else
        // 如果是在非 Windows 平台下，定义 UNIX 特有的文件数据结构和目录句柄
        struct dirent *filedata;
        DIR *unixdir;
#endif

        // 解析源字符串，获取源类型和名称
        if (pcap_parsesrcstr(source, &type, NULL, NULL, name, errbuf) == -1)
            return -1;

        /* 检查文件名是否正确 */
        stringlen = strlen(name);

        /* 目录在 Win32 下必须以 '\' 结尾，在 UNIX 下必须以 '/' 结尾 */
#ifdef _WIN32
#define ENDING_CHAR '\\'
#else
#define ENDING_CHAR '/'
#endif

        // 如果文件名最后一个字符不是指定的结尾字符
        if (name[stringlen - 1] != ENDING_CHAR)
        {
            // 在文件名末尾添加指定的结尾字符
            name[stringlen] = ENDING_CHAR;
            name[stringlen + 1] = 0;

            stringlen++;
        }

        /* 保存路径以备将来参考 */
        snprintf(path, sizeof(path), "%s", name);
        pathlen = strlen(path);

#ifdef _WIN32
        /* 要执行目录列表，Win32 必须以 'asterisk' 作为结尾字符 */
        if (name[stringlen - 1] != '*')
        {
            name[stringlen] = '*';
            name[stringlen + 1] = 0;
        }

        // 查找第一个文件
        filehandle = FindFirstFile(name, &filedata);

        // 如果查找失败
        if (filehandle == INVALID_HANDLE_VALUE)
        {
            // 设置错误消息并返回
            snprintf(errbuf, PCAP_ERRBUF_SIZE, "Error when listing files: does folder '%s' exist?", path);
            return -1;
        }

#else
        /* 打开目录 */
        unixdir= opendir(path);

        /* 获取其中的第一个文件 */
        filedata= readdir(unixdir);

        // 如果获取失败
        if (filedata == NULL)
        {
            DIAG_OFF_FORMAT_TRUNCATION
            // 设置错误消息并返回
            snprintf(errbuf, PCAP_ERRBUF_SIZE, "Error when listing files: does folder '%s' exist?", path);
            DIAG_ON_FORMAT_TRUNCATION
            closedir(unixdir);
            return -1;
        }
#endif

        /* 将找到的所有文件添加到列表中 */
        do
        {
#ifdef _WIN32
            /* 如果文件路径名超出缓冲区大小，则跳过该文件 */
            if (pathlen + strlen(filedata.cFileName) >= sizeof(filename))
                continue;
            // 将路径和文件名拼接成完整的文件路径
            snprintf(filename, sizeof(filename), "%s%s", path, filedata.cFileName);
#else
            if (pathlen + strlen(filedata->d_name) >= sizeof(filename))
                continue;
            // 关闭格式截断警告
            DIAG_OFF_FORMAT_TRUNCATION
            // 将路径和文件名拼接成完整的文件路径
            snprintf(filename, sizeof(filename), "%s%s", path, filedata->d_name);
            // 打开格式截断警告
            DIAG_ON_FORMAT_TRUNCATION
#endif

            // 打开离线捕获文件
            fp = pcap_open_offline(filename, errbuf);

            if (fp)
            {
                /* 分配主结构 */
                dev = (pcap_if_t *)malloc(sizeof(pcap_if_t));
                if (dev == NULL)
                {
                    // 如果分配内存失败，则设置错误消息并释放所有设备
                    pcap_fmt_errmsg_for_errno(errbuf,
                        PCAP_ERRBUF_SIZE, errno,
                        "malloc() failed");
                    pcap_freealldevs(*alldevs);
#ifdef _WIN32
                    FindClose(filehandle);
#else
                    closedir(unixdir);
#endif
                    return -1;
                }

                /* 初始化结构体为'零' */
                memset(dev, 0, sizeof(pcap_if_t));

                /* 将其附加到列表中。 */
                if (lastdev == NULL)
                {
                    /*
                     * 列表为空，因此它也是
                     * 第一个设备。
                     */
                    *alldevs = dev;
                }
                else
                {
                    /*
                     * 在最后一个设备之后附加。
                     */
                    lastdev->next = dev;
                }
                /* 现在它是最后一个设备。 */
                lastdev = dev;

                /* 创建新的源标识符 */
                if (pcap_createsrcstr(tmpstring, PCAP_SRC_FILE, NULL, NULL, filename, errbuf) == -1)
                {
                    pcap_freealldevs(*alldevs);
#ifdef _WIN32
                    FindClose(filehandle);
#else
                    closedir(unixdir);
#endif
                    return -1;
                }

                dev->name = strdup(tmpstring);
                if (dev->name == NULL)
                {
                    pcap_fmt_errmsg_for_errno(errbuf,
                        PCAP_ERRBUF_SIZE, errno,
                        "malloc() failed");
                    pcap_freealldevs(*alldevs);
#ifdef _WIN32
                    FindClose(filehandle);
#else
                    closedir(unixdir);
#endif
                    return -1;
                }

                /*
                 * 如果打开文件失败，返回-1
                 */
                if (pcap_asprintf(&dev->description,
                    "%s '%s' %s", PCAP_TEXT_SOURCE_FILE,
                    filename, PCAP_TEXT_SOURCE_ON_LOCAL_HOST) == -1)
                {
                    pcap_fmt_errmsg_for_errno(errbuf,
                        PCAP_ERRBUF_SIZE, errno,
                        "malloc() failed");
                    pcap_freealldevs(*alldevs);
#ifdef _WIN32
                    FindClose(filehandle);
#else
                    closedir(unixdir);
#endif
                    return -1;
                }

                pcap_close(fp);
            }
        }
#ifdef _WIN32
        while (FindNextFile(filehandle, &filedata) != 0);
#else
        while ( (filedata= readdir(unixdir)) != NULL);
#endif


        /* Close the search handle. */
#ifdef _WIN32
        FindClose(filehandle);
#else
        closedir(unixdir);
#endif

        return 0;
    }

    case PCAP_SRC_IFREMOTE:
        return pcap_findalldevs_ex_remote(source, auth, alldevs, errbuf);

    default:
        pcap_strlcpy(errbuf, "Source type not supported", PCAP_ERRBUF_SIZE);
        return -1;
    }
}

pcap_t *pcap_open(const char *source, int snaplen, int flags, int read_timeout, struct pcap_rmtauth *auth, char *errbuf)
{
    char name[PCAP_BUF_SIZE];
    int type;
    pcap_t *fp;
    int status;

    /*
     * 如果源为空，则将其设置为"any"
     */
    if (source == NULL)
        source = "any";

    /*
     * 如果源字符串过长，则返回错误
     */
    if (strlen(source) > PCAP_BUF_SIZE)
    {
        snprintf(errbuf, PCAP_ERRBUF_SIZE, "The source string is too long. Cannot handle it correctly.");
        return NULL;
    }
    /*
     * 确定源的类型（文件、本地、远程），
     * 如果是文件或本地，则确定文件名或捕获设备的名称。
     */
    if (pcap_parsesrcstr(source, &type, NULL, NULL, name, errbuf) == -1)
        return NULL;

    switch (type)
    {
    case PCAP_SRC_FILE:
        return pcap_open_offline(name, errbuf);

    case PCAP_SRC_IFLOCAL:
        fp = pcap_create(name, errbuf);
        break;

    case PCAP_SRC_IFREMOTE:
        /*
         * 虽然我们已经有主机、端口和接口，但我们更喜欢只传递 'source' 给 pcap_open_rpcap()，
         * 这样它就必须再次调用 pcap_parsesrcstr()。
         * 这样做不太优化，但更清晰。
         */
        return pcap_open_rpcap(source, snaplen, flags, read_timeout, auth, errbuf);

    default:
        pcap_strlcpy(errbuf, "Source type not supported", PCAP_ERRBUF_SIZE);
        return NULL;
    }

    if (fp == NULL)
        return (NULL);
    status = pcap_set_snaplen(fp, snaplen);
    if (status < 0)
        goto fail;
    if (flags & PCAP_OPENFLAG_PROMISCUOUS)
    {
        status = pcap_set_promisc(fp, 1);
        if (status < 0)
            goto fail;
    }
    if (flags & PCAP_OPENFLAG_MAX_RESPONSIVENESS)
    {
        status = pcap_set_immediate_mode(fp, 1);
        if (status < 0)
            goto fail;
    }
#ifdef _WIN32
    /*
     * This flag is supported on Windows only.
     * XXX - is there a way to support it with
     * the capture mechanisms on UN*X?  It's not
     * exactly a "set direction" operation; I
     * think it means "do not capture packets
     * injected with pcap_sendpacket() or
     * pcap_inject()".
     */
    /* disable loopback capture if requested */
    // 如果请求禁用本地回环捕获，则设置禁用本地回环捕获标志
    if (flags & PCAP_OPENFLAG_NOCAPTURE_LOCAL)
        fp->opt.nocapture_local = 1;
#endif /* _WIN32 */
    // 设置读取超时时间
    status = pcap_set_timeout(fp, read_timeout);
    // 如果设置失败，则跳转到失败处理
    if (status < 0)
        goto fail;
    // 激活捕获会话
    status = pcap_activate(fp);
    // 如果激活失败，则跳转到失败处理
    if (status < 0)
        goto fail;
    // 返回捕获会话指针
    return fp;

fail:
    DIAG_OFF_FORMAT_TRUNCATION
    // 如果捕获会话状态为错误，则将错误信息格式化到错误缓冲区
    if (status == PCAP_ERROR)
        snprintf(errbuf, PCAP_ERRBUF_SIZE, "%s: %s",
            name, fp->errbuf);
    // 如果捕获会话状态为设备不存在、权限被拒绝或者混杂模式权限被拒绝，则将错误信息格式化到错误缓冲区
    else if (status == PCAP_ERROR_NO_SUCH_DEVICE ||
        status == PCAP_ERROR_PERM_DENIED ||
        status == PCAP_ERROR_PROMISC_PERM_DENIED)
        snprintf(errbuf, PCAP_ERRBUF_SIZE, "%s: %s (%s)",
            name, pcap_statustostr(status), fp->errbuf);
    // 否则将错误信息格式化到错误缓冲区
    else
        snprintf(errbuf, PCAP_ERRBUF_SIZE, "%s: %s",
            name, pcap_statustostr(status));
    DIAG_ON_FORMAT_TRUNCATION
    // 关闭捕获会话
    pcap_close(fp);
    // 返回空指针
    return NULL;
}

// 设置采样参数
struct pcap_samp *pcap_setsampling(pcap_t *p)
{
    // 返回远程采样结构体指针
    return &p->rmt_samp;
}
```