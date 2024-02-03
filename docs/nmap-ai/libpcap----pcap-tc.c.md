# `nmap\libpcap\pcap-tc.c`

```cpp
/*
 * 版权声明，版权所有
 *
 * 在源代码和二进制形式下重新分发和使用，无论是否经过修改，都需要满足以下条件：
 * 1. 源代码的再分发必须保留上述版权声明、条件列表和以下免责声明。
 * 2. 二进制形式的再分发必须在文档和/或其他提供的材料中重现上述版权声明、条件列表和以下免责声明。
 * 3. 未经特定书面许可，不得使用 CACE Technologies 的名称或其贡献者的名称来认可或推广从本软件衍生的产品。
 *
 * 本软件由版权所有者和贡献者 "按原样" 提供，不提供任何明示或暗示的担保，包括但不限于对适销性和特定用途的暗示担保。在任何情况下，无论是在合同、严格责任还是侵权行为（包括疏忽或其他情况）下，版权所有者或贡献者均不对任何直接、间接、附带、特殊、惩罚性或后果性损害（包括但不限于替代商品或服务的采购、使用损失、数据或利润损失、业务中断）承担责任。
 *
 */

#ifdef HAVE_CONFIG_H
#include <config.h>
#endif

#include <pcap.h>
#include <pcap-int.h>

#include "pcap-tc.h"

#include <malloc.h>
#include <memory.h>
#include <string.h>
#include <errno.h>

#ifdef _WIN32
#include <tchar.h>
#endif

// 定义了一个名为 TcFcnQueryPortList 的函数指针类型，该函数接受两个参数，返回 TC_STATUS 类型的值
typedef TC_STATUS    (TC_CALLCONV *TcFcnQueryPortList)            (PTC_PORT *ppPorts, PULONG pLength);
# 定义了一系列函数指针类型，用于指向不同的函数
typedef TC_STATUS    (TC_CALLCONV *TcFcnFreePortList)            (TC_PORT *pPorts);  # 定义了一个函数指针类型 TcFcnFreePortList，该函数接受一个 TC_PORT 指针参数，并返回 TC_STATUS 类型的值

typedef PCHAR        (TC_CALLCONV *TcFcnStatusGetString)            (TC_STATUS status);  # 定义了一个函数指针类型 TcFcnStatusGetString，该函数接受一个 TC_STATUS 类型的参数，并返回 PCHAR 类型的值

typedef PCHAR        (TC_CALLCONV *TcFcnPortGetName)                (TC_PORT port);  # 定义了一个函数指针类型 TcFcnPortGetName，该函数接受一个 TC_PORT 类型的参数，并返回 PCHAR 类型的值
typedef PCHAR        (TC_CALLCONV *TcFcnPortGetDescription)        (TC_PORT port);  # 定义了一个函数指针类型 TcFcnPortGetDescription，该函数接受一个 TC_PORT 类型的参数，并返回 PCHAR 类型的值

typedef TC_STATUS    (TC_CALLCONV *TcFcnInstanceOpenByName)        (PCHAR name, PTC_INSTANCE pInstance);  # 定义了一个函数指针类型 TcFcnInstanceOpenByName，该函数接受一个 PCHAR 类型和一个 PTC_INSTANCE 指针参数，并返回 TC_STATUS 类型的值
typedef TC_STATUS    (TC_CALLCONV *TcFcnInstanceClose)            (TC_INSTANCE instance);  # 定义了一个函数指针类型 TcFcnInstanceClose，该函数接受一个 TC_INSTANCE 类型的参数，并返回 TC_STATUS 类型的值
typedef TC_STATUS    (TC_CALLCONV *TcFcnInstanceSetFeature)        (TC_INSTANCE instance, ULONG feature, ULONG value);  # 定义了一个函数指针类型 TcFcnInstanceSetFeature，该函数接受一个 TC_INSTANCE 类型和两个 ULONG 类型的参数，并返回 TC_STATUS 类型的值
typedef TC_STATUS    (TC_CALLCONV *TcFcnInstanceQueryFeature)    (TC_INSTANCE instance, ULONG feature, PULONG pValue);  # 定义了一个函数指针类型 TcFcnInstanceQueryFeature，该函数接受一个 TC_INSTANCE 类型和两个 ULONG 类型的参数，并返回 TC_STATUS 类型的值
typedef TC_STATUS    (TC_CALLCONV *TcFcnInstanceReceivePackets)    (TC_INSTANCE instance, PTC_PACKETS_BUFFER pBuffer);  # 定义了一个函数指针类型 TcFcnInstanceReceivePackets，该函数接受一个 TC_INSTANCE 类型和一个 PTC_PACKETS_BUFFER 指针参数，并返回 TC_STATUS 类型的值
typedef HANDLE        (TC_CALLCONV *TcFcnInstanceGetReceiveWaitHandle) (TC_INSTANCE instance);  # 定义了一个函数指针类型 TcFcnInstanceGetReceiveWaitHandle，该函数接受一个 TC_INSTANCE 类型的参数，并返回 HANDLE 类型的值
typedef TC_STATUS    (TC_CALLCONV *TcFcnInstanceTransmitPackets)    (TC_INSTANCE instance, TC_PACKETS_BUFFER pBuffer);  # 定义了一个函数指针类型 TcFcnInstanceTransmitPackets，该函数接受一个 TC_INSTANCE 类型和一个 TC_PACKETS_BUFFER 类型的参数，并返回 TC_STATUS 类型的值
typedef TC_STATUS    (TC_CALLCONV *TcFcnInstanceQueryStatistics)    (TC_INSTANCE instance, PTC_STATISTICS pStatistics);  # 定义了一个函数指针类型 TcFcnInstanceQueryStatistics，该函数接受一个 TC_INSTANCE 类型和一个 PTC_STATISTICS 指针参数，并返回 TC_STATUS 类型的值

typedef TC_STATUS    (TC_CALLCONV *TcFcnPacketsBufferCreate)        (ULONG size, PTC_PACKETS_BUFFER pBuffer);  # 定义了一个函数指针类型 TcFcnPacketsBufferCreate，该函数接受一个 ULONG 类型和一个 PTC_PACKETS_BUFFER 指针参数，并返回 TC_STATUS 类型的值
typedef VOID        (TC_CALLCONV *TcFcnPacketsBufferDestroy)    (TC_PACKETS_BUFFER buffer);  # 定义了一个函数指针类型 TcFcnPacketsBufferDestroy，该函数接受一个 TC_PACKETS_BUFFER 类型的参数，并返回 VOID 类型的值
typedef TC_STATUS    (TC_CALLCONV *TcFcnPacketsBufferQueryNextPacket)(TC_PACKETS_BUFFER buffer, PTC_PACKET_HEADER pHeader, PVOID *ppData);  # 定义了一个函数指针类型 TcFcnPacketsBufferQueryNextPacket，该函数接受一个 TC_PACKETS_BUFFER 类型和两个指针参数，并返回 TC_STATUS 类型的值
typedef TC_STATUS    (TC_CALLCONV *TcFcnPacketsBufferCommitNextPacket)(TC_PACKETS_BUFFER buffer, PTC_PACKET_HEADER pHeader, PVOID pData);  # 定义了一个函数指针类型 TcFcnPacketsBufferCommitNextPacket，该函数接受一个 TC_PACKETS_BUFFER 类型和两个指针参数，并返回 TC_STATUS 类型的值

typedef VOID        (TC_CALLCONV *TcFcnStatisticsDestroy)        (TC_STATISTICS statistics);  # 定义了一个函数指针类型 TcFcnStatisticsDestroy，该函数接受一个 TC_STATISTICS 类型的参数，并返回 VOID 类型的值
typedef TC_STATUS    (TC_CALLCONV *TcFcnStatisticsUpdate)        (TC_STATISTICS statistics);  # 定义了一个函数指针类型 TcFcnStatisticsUpdate，该函数接受一个 TC_STATISTICS 类型的参数，并返回 TC_STATUS 类型的值
# 定义一个函数指针类型，用于查询统计数据的数值
typedef TC_STATUS    (TC_CALLCONV *TcFcnStatisticsQueryValue)    (TC_STATISTICS statistics, ULONG counterId, PULONGLONG pValue);

# 定义一个枚举类型，表示 API 的加载状态
typedef enum LONG
{
    TC_API_UNLOADED = 0,  # API 未加载
    TC_API_LOADED,        # API 已加载
    TC_API_CANNOT_LOAD,   # API 无法加载
    TC_API_LOADING        # API 加载中
}
    TC_API_LOAD_STATUS;

# 定义一个结构体，包含了一系列函数指针和加载状态
typedef struct _TC_FUNCTIONS
{
    TC_API_LOAD_STATUS            LoadStatus;  # API 加载状态
#ifdef _WIN32
    HMODULE                        hTcApiDllHandle;  # Windows 平台下的动态链接库句柄
#endif
    TcFcnQueryPortList            QueryPortList;  # 查询端口列表的函数指针
    TcFcnFreePortList            FreePortList;  # 释放端口列表的函数指针
    TcFcnStatusGetString        StatusGetString;  # 获取状态字符串的函数指针
    # ... 其他函数指针

# 声明一系列静态函数，用于操作 pcap_t 结构体
static pcap_if_t* TcCreatePcapIfFromPort(TC_PORT port);  # 根据端口创建 pcap_if_t 结构体
static int TcSetDatalink(pcap_t *p, int dlt);  # 设置数据链路类型
static int TcGetNonBlock(pcap_t *p);  # 获取非阻塞模式
static int TcSetNonBlock(pcap_t *p, int nonblock);  # 设置非阻塞模式
static void TcCleanup(pcap_t *p);  # 清理资源
static int TcInject(pcap_t *p, const void *buf, int size);  # 发送数据包
static int TcRead(pcap_t *p, int cnt, pcap_handler callback, u_char *user);  # 读取数据包
static int TcStats(pcap_t *p, struct pcap_stat *ps);  # 获取统计信息
#ifdef _WIN32
static struct pcap_stat *TcStatsEx(pcap_t *p, int *pcap_stat_size);
// 返回扩展的统计信息

static int TcSetBuff(pcap_t *p, int dim);
// 设置接收缓冲区大小

static int TcSetMode(pcap_t *p, int mode);
// 设置模式

static int TcSetMinToCopy(pcap_t *p, int size);
// 设置最小拷贝大小

static HANDLE TcGetReceiveWaitHandle(pcap_t *p);
// 获取接收等待句柄

static int TcOidGetRequest(pcap_t *p, bpf_u_int32 oid, void *data, size_t *lenp);
// 发送 OID 获取请求

static int TcOidSetRequest(pcap_t *p, bpf_u_int32 oid, const void *data, size_t *lenp);
// 发送 OID 设置请求

static u_int TcSendqueueTransmit(pcap_t *p, pcap_send_queue *queue, int sync);
// 发送队列传输

static int TcSetUserBuffer(pcap_t *p, int size);
// 设置用户缓冲区大小

static int TcLiveDump(pcap_t *p, char *filename, int maxsize, int maxpacks);
// 实时转储

static int TcLiveDumpEnded(pcap_t *p, int sync);
// 实时转储结束

static PAirpcapHandle TcGetAirPcapHandle(pcap_t *p);
// 获取 AirPcap 句柄

#ifdef _WIN32
TC_FUNCTIONS g_TcFunctions =
{
    TC_API_UNLOADED, /* LoadStatus */
    NULL,  /* hTcApiDllHandle */
    NULL,  /* QueryPortList */
    NULL,  /* FreePortList */
    NULL,  /* StatusGetString */
    NULL,  /* PortGetName */
    NULL,  /* PortGetDescription */
    NULL,  /* InstanceOpenByName */
    NULL,  /* InstanceClose */
    NULL,  /* InstanceSetFeature */
    NULL,  /* InstanceQueryFeature */
    NULL,  /* InstanceReceivePackets */
    NULL,  /* InstanceGetReceiveWaitHandle */
    NULL,  /* InstanceTransmitPackets */
    NULL,  /* InstanceQueryStatistics */
    NULL,  /* PacketsBufferCreate */
    NULL,  /* PacketsBufferDestroy */
    NULL,  /* PacketsBufferQueryNextPacket */
    NULL,  /* PacketsBufferCommitNextPacket */
    NULL,  /* StatisticsDestroy */
    NULL,  /* StatisticsUpdate */
    NULL  /* StatisticsQueryValue */
};
#else
TC_FUNCTIONS g_TcFunctions =
{
    TC_API_LOADED, /* LoadStatus */
    TcQueryPortList,
    TcFreePortList,
    TcStatusGetString,
    TcPortGetName,
    TcPortGetDescription,
    TcInstanceOpenByName,
    TcInstanceClose,
    TcInstanceSetFeature,
    TcInstanceQueryFeature,
    TcInstanceReceivePackets,
#ifdef _WIN32
    TcInstanceGetReceiveWaitHandle,
#endif
    TcInstanceTransmitPackets,
    # 定义了一系列与 TcInstance 相关的查询和统计函数
    TcInstanceQueryStatistics,
    TcPacketsBufferCreate,
    TcPacketsBufferDestroy,
    TcPacketsBufferQueryNextPacket,
    TcPacketsBufferCommitNextPacket,
    TcStatisticsDestroy,
    TcStatisticsUpdate,
    TcStatisticsQueryValue,
};  // 结束 #endif 块

#define MAX_TC_PACKET_SIZE    9500  // 定义最大 TC 数据包大小

#pragma pack(push, 1)  // 设置结构体按 1 字节对齐

#define PPH_PH_FLAG_PADDING    ((UCHAR)0x01)  // 定义 PPH 标志位填充
#define PPH_PH_VERSION        ((UCHAR)0x00)  // 定义 PPH 版本号

typedef struct _PPI_PACKET_HEADER  // 定义 PPI 数据包头结构体
{
    UCHAR    PphVersion;  // PPH 版本号
    UCHAR    PphFlags;  // PPH 标志位
    USHORT    PphLength;  // PPH 长度
    ULONG    PphDlt;  // PPH 数据链路类型
}
    PPI_PACKET_HEADER, *PPPI_PACKET_HEADER;  // 定义 PPI_PACKET_HEADER 结构体及指针类型

typedef struct _PPI_FIELD_HEADER  // 定义 PPI 字段头结构体
{
    USHORT PfhType;  // 字段类型
    USHORT PfhLength;  // 字段长度
}
    PPI_FIELD_HEADER, *PPPI_FIELD_HEADER;  // 定义 PPI_FIELD_HEADER 结构体及指针类型

#define        PPI_FIELD_TYPE_AGGREGATION_EXTENSION    ((UCHAR)0x08)  // 定义聚合扩展字段类型

typedef struct _PPI_FIELD_AGGREGATION_EXTENSION  // 定义聚合扩展字段结构体
{
    ULONG        InterfaceId;  // 接口 ID
}
    PPI_FIELD_AGGREGATION_EXTENSION, *PPPI_FIELD_AGGREGATION_EXTENSION;  // 定义 PPI_FIELD_AGGREGATION_EXTENSION 结构体及指针类型

#define        PPI_FIELD_TYPE_802_3_EXTENSION            ((UCHAR)0x09)  // 定义 802.3 扩展字段类型
#define PPI_FLD_802_3_EXT_FLAG_FCS_PRESENT            ((ULONG)0x00000001)  // 定义 802.3 扩展字段 FCS 存在标志

typedef struct _PPI_FIELD_802_3_EXTENSION  // 定义 802.3 扩展字段结构体
{
    ULONG        Flags;  // 标志
    ULONG        Errors;  // 错误
}
    PPI_FIELD_802_3_EXTENSION, *PPPI_FIELD_802_3_EXTENSION;  // 定义 PPI_FIELD_802_3_EXTENSION 结构体及指针类型

typedef struct _PPI_HEADER  // 定义 PPI 头结构体
{
    PPI_PACKET_HEADER PacketHeader;  // 数据包头
    PPI_FIELD_HEADER  AggregationFieldHeader;  // 聚合字段头
    PPI_FIELD_AGGREGATION_EXTENSION AggregationField;  // 聚合字段
    PPI_FIELD_HEADER  Dot3FieldHeader;  // 802.3 字段头
    PPI_FIELD_802_3_EXTENSION Dot3Field;  // 802.3 字段
}
    PPI_HEADER, *PPPI_HEADER;  // 定义 PPI_HEADER 结构体及指针类型
#pragma pack(pop)  // 恢复默认的结构体对齐方式

#ifdef _WIN32
/*
 * 注意：这个函数应该被理论上可以处理 Tc 库的 pcap 函数调用，比如列出适配器和打开适配器的函数所调用。
 * 其他函数（关闭、读取、写入、设置参数）都是在 TC 的实例上操作的，所以我们不需要调用这个函数
 */
TC_API_LOAD_STATUS LoadTcFunctions(void)  // 加载 Tc 函数
{
    TC_API_LOAD_STATUS currentStatus;  // 当前状态

    do
    }while(FALSE);  // 循环直到条件为假

    if (currentStatus != TC_API_LOADED)  // 如果当前状态不是 TC_API_LOADED
    {
        if (g_TcFunctions.hTcApiDllHandle != NULL)  // 如果 TC 函数句柄不为空
        {
            FreeLibrary(g_TcFunctions.hTcApiDllHandle);  // 释放 TC 函数句柄
            g_TcFunctions.hTcApiDllHandle = NULL;  // 将 TC 函数句柄置为空
        }
    }
    # 使用原子操作将 g_TcFunctions.LoadStatus 的值替换为 currentStatus
    InterlockedExchange((LONG*)&g_TcFunctions.LoadStatus, currentStatus);
    # 返回 currentStatus 的值
    return currentStatus;
}
#else
// 如果没有使用动态链接库，则使用静态链接库
TC_API_LOAD_STATUS LoadTcFunctions(void)
{
    // 返回 TC_API_LOADED 表示成功加载 TurboCap 函数
    return TC_API_LOADED;
}
#endif

/*
 * 用于捕获 TurboCap 设备的私有数据。
 */
struct pcap_tc {
    // TurboCap 实例
    TC_INSTANCE TcInstance;
    // TurboCap 数据包缓冲区
    TC_PACKETS_BUFFER TcPacketsBuffer;
    // 已接受的数据包数量
    ULONG TcAcceptedCount;
    // PPI 数据包
    u_char *PpiPacket;
};

// 查找所有 TurboCap 设备
int
TcFindAllDevs(pcap_if_list_t *devlist, char *errbuf)
{
    // 加载 TurboCap 函数的状态
    TC_API_LOAD_STATUS loadStatus;
    // 端口数量
    ULONG numPorts;
    // 端口列表
    PTC_PORT pPorts = NULL;
    // TurboCap 状态
    TC_STATUS status;
    // 结果
    int result = 0;
    // 设备
    pcap_if_t *dev;
    // 索引
    ULONG i;

    do
    {
        // 加载 TurboCap 函数
        loadStatus = LoadTcFunctions();

        // 如果加载失败，则返回 0
        if (loadStatus != TC_API_LOADED)
        {
            result = 0;
            break;
        }

        /*
         * 枚举端口，并将它们添加到列表中
         */
        status = g_TcFunctions.QueryPortList(&pPorts, &numPorts);

        // 如果查询端口列表失败，则返回 0
        if (status != TC_SUCCESS)
        {
            result = 0;
            break;
        }

        for (i = 0; i < numPorts; i++)
        {
            /*
             * 将端口转换为列表中的条目
             */
            dev = TcCreatePcapIfFromPort(pPorts[i]);

            // 如果创建成功，则将设备添加到列表中
            if (dev != NULL)
                add_dev(devlist, dev->name, dev->flags, dev->description, errbuf);
        }

        if (numPorts > 0)
        {
            /*
             * 忽略此处的结果
             */
            status = g_TcFunctions.FreePortList(pPorts);
        }

    }while(FALSE);

    return result;
}

// 从端口创建 pcap_if_t 结构
static pcap_if_t* TcCreatePcapIfFromPort(TC_PORT port)
{
    CHAR *name;
    CHAR *description;
    pcap_if_t *newIf = NULL;

    newIf = (pcap_if_t*)malloc(sizeof(*newIf));
    if (newIf == NULL)
    {
        return NULL;
    }

    memset(newIf, 0, sizeof(*newIf));

    name = g_TcFunctions.PortGetName(port);
    description = g_TcFunctions.PortGetDescription(port);

    newIf->name = (char*)malloc(strlen(name) + 1);
    if (newIf->name == NULL)
    {
        free(newIf);
        return NULL;
    }
    // 为描述分配内存空间，长度为描述字符串长度加1，用于存储字符串结尾的空字符
    newIf->description = (char*)malloc(strlen(description) + 1);
    // 检查描述内存分配是否成功
    if (newIf->description == NULL)
    {
        // 如果描述内存分配失败，释放名称和新接口结构体的内存空间，并返回空指针
        free(newIf->name);
        free(newIf);
        return NULL;
    }

    // 将名称复制到新接口结构体的名称字段
    strcpy(newIf->name, name);
    // 将描述复制到新接口结构体的描述字段
    strcpy(newIf->description, description);

    // 初始化新接口结构体的地址、下一个接口、标志字段
    newIf->addresses = NULL;
    newIf->next = NULL;
    newIf->flags = 0;

    // 返回新接口结构体指针
    return newIf;
    # 激活 TurboCap 适配器
    static int
    TcActivate(pcap_t *p)
    {
        # 获取 pcap_t 结构体中的私有数据
        struct pcap_tc *pt = p->priv;
        # 定义 TC_STATUS 变量和 ULONG 变量
        TC_STATUS status;
        ULONG timeout;
        PPPI_HEADER pPpiHeader;

        # 如果设置了 rfmon（monitor mode），则返回不支持监控模式的错误
        if (p->opt.rfmon)
        {
            /*
             * Tc 卡不支持监控模式；它们是以太网
             * 抓包适配器。
             */
            return PCAP_ERROR_RFMON_NOTSUP;
        }

        # 分配 PPI 包的内存空间
        pt->PpiPacket = malloc(sizeof(PPI_HEADER) + MAX_TC_PACKET_SIZE);

        # 如果分配内存失败，则返回错误
        if (pt->PpiPacket == NULL)
        {
            snprintf(p->errbuf, PCAP_ERRBUF_SIZE, "Error allocating memory");
            return PCAP_ERROR;
        }

        /*
         * 将负的快照值（无效）、快照值为 0（未指定）或大于正常最大值的值
         * 转换为最大允许的值。
         *
         * 如果某些应用程序确实 *需要* 更大的快照长度，我们应该增加 MAXIMUM_SNAPLEN。
         */
        if (p->snapshot <= 0 || p->snapshot > MAXIMUM_SNAPLEN)
            p->snapshot = MAXIMUM_SNAPLEN;

        /*
         * 初始化 PPI 固定字段
         */
        pPpiHeader = (PPPI_HEADER)pt->PpiPacket;
        pPpiHeader->PacketHeader.PphDlt = DLT_EN10MB;
        pPpiHeader->PacketHeader.PphLength = sizeof(PPI_HEADER);
        pPpiHeader->PacketHeader.PphFlags = 0;
        pPpiHeader->PacketHeader.PphVersion = 0;

        pPpiHeader->AggregationFieldHeader.PfhLength = sizeof(PPI_FIELD_AGGREGATION_EXTENSION);
        pPpiHeader->AggregationFieldHeader.PfhType = PPI_FIELD_TYPE_AGGREGATION_EXTENSION;

        pPpiHeader->Dot3FieldHeader.PfhLength = sizeof(PPI_FIELD_802_3_EXTENSION);
        pPpiHeader->Dot3FieldHeader.PfhType = PPI_FIELD_TYPE_802_3_EXTENSION;

        # 通过设备名称打开 TurboCap 适配器
        status = g_TcFunctions.InstanceOpenByName(p->opt.device, &pt->TcInstance);

        # 如果打开失败，则返回错误
        if (status != TC_SUCCESS)
        {
            /* 检测到适配器，但我们无法打开它。返回失败。 */
            snprintf(p->errbuf, PCAP_ERRBUF_SIZE, "Error opening TurboCap adapter: %s", g_TcFunctions.StatusGetString(status));
            return PCAP_ERROR;
        }

        # 设置链路类型为 DLT_EN10MB
        p->linktype = DLT_EN10MB;
    }
    # 为指针 p 分配两个 u_int 类型的内存空间
    p->dlt_list = (u_int *) malloc(sizeof(u_int) * 2);
    # 如果分配内存失败，则将列表保持为空
    if (p->dlt_list != NULL) {
        # 将以太网和PPI数据链路类型分别赋值给列表的第一个和第二个元素
        p->dlt_list[0] = DLT_EN10MB;
        p->dlt_list[1] = DLT_PPI;
        # 设置数据链路类型的数量为2
        p->dlt_count = 2;
    }

    # 忽略混杂模式
    # p->opt.promisc

    # 忽略所有缓冲区大小

    # 启用接收
    status = g_TcFunctions.InstanceSetFeature(pt->TcInstance, TC_INST_FT_RX_STATUS, 1);
    # 如果启用接收失败，则输出错误信息并跳转到 bad 标签
    if (status != TC_SUCCESS)
    {
        snprintf(p->errbuf, PCAP_ERRBUF_SIZE,"Error enabling reception on a TurboCap instance: %s", g_TcFunctions.StatusGetString(status));
        goto bad;
    }

    # 启用传输
    status = g_TcFunctions.InstanceSetFeature(pt->TcInstance, TC_INST_FT_TX_STATUS, 1);
    # 忽略此处的错误

    # 设置数据包注入操作为 TcInject

    # 如果超时时间为 -1，则表示立即返回，没有超时
    # 如果超时时间为 0，则表示无限超时
    if (p->opt.timeout == 0)
    {
        timeout = 0xFFFFFFFF;
    }
    else if (p->opt.timeout < 0)
    {
        # 在这里插入一个最小超时时间
        timeout = 10;
    }
    else
    {
        timeout = p->opt.timeout;
    }

    # 设置读取超时时间
    status = g_TcFunctions.InstanceSetFeature(pt->TcInstance, TC_INST_FT_READ_TIMEOUT, timeout);
    # 如果设置读取超时时间失败，则输出错误信息并跳转到 bad 标签
    if (status != TC_SUCCESS)
    {
        snprintf(p->errbuf, PCAP_ERRBUF_SIZE,"Error setting the read timeout a TurboCap instance: %s", g_TcFunctions.StatusGetString(status));
        goto bad;
    }

    # 设置读取操作为 TcRead
    # 设置过滤器操作为 install_bpf_program
    # 设置方向操作为 NULL（未实现）
    # 设置数据链路类型操作为 TcSetDatalink
    # 获取非阻塞操作为 TcGetNonBlock
    # 设置非阻塞操作为 TcSetNonBlock
    # 统计操作为 TcStats
#ifdef _WIN32
    // 如果是 Windows 系统
    p->stats_ex_op = TcStatsEx;
    // 设置 stats_ex_op 操作为 TcStatsEx
    p->setbuff_op = TcSetBuff;
    // 设置 setbuff_op 操作为 TcSetBuff
    p->setmode_op = TcSetMode;
    // 设置 setmode_op 操作为 TcSetMode
    p->setmintocopy_op = TcSetMinToCopy;
    // 设置 setmintocopy_op 操作为 TcSetMinToCopy
    p->getevent_op = TcGetReceiveWaitHandle;
    // 设置 getevent_op 操作为 TcGetReceiveWaitHandle
    p->oid_get_request_op = TcOidGetRequest;
    // 设置 oid_get_request_op 操作为 TcOidGetRequest
    p->oid_set_request_op = TcOidSetRequest;
    // 设置 oid_set_request_op 操作为 TcOidSetRequest
    p->sendqueue_transmit_op = TcSendqueueTransmit;
    // 设置 sendqueue_transmit_op 操作为 TcSendqueueTransmit
    p->setuserbuffer_op = TcSetUserBuffer;
    // 设置 setuserbuffer_op 操作为 TcSetUserBuffer
    p->live_dump_op = TcLiveDump;
    // 设置 live_dump_op 操作为 TcLiveDump
    p->live_dump_ended_op = TcLiveDumpEnded;
    // 设置 live_dump_ended_op 操作为 TcLiveDumpEnded
    p->get_airpcap_handle_op = TcGetAirPcapHandle;
    // 设置 get_airpcap_handle_op 操作为 TcGetAirPcapHandle
#else
    // 如果不是 Windows 系统
    p->selectable_fd = -1;
    // 设置 selectable_fd 为 -1
#endif

    p->cleanup_op = TcCleanup;
    // 设置 cleanup_op 操作为 TcCleanup

    return 0;
bad:
    // 如果出现错误
    TcCleanup(p);
    // 调用 TcCleanup 函数清理资源
    return PCAP_ERROR;
}

pcap_t *
TcCreate(const char *device, char *ebuf, int *is_ours)
{
    ULONG numPorts;
    PTC_PORT pPorts = NULL;
    TC_STATUS status;
    int is_tc;
    ULONG i;
    pcap_t *p;

    if (LoadTcFunctions() != TC_API_LOADED)
    {
        // 如果加载 TcFunctions 失败
        *is_ours = 0;
        // 设置 is_ours 为 0
        return NULL;
    }

    /*
     * 枚举端口，并将它们添加到列表中
     */
    status = g_TcFunctions.QueryPortList(&pPorts, &numPorts);

    if (status != TC_SUCCESS)
    {
        // 如果查询端口列表失败
        *is_ours = 0;
        // 设置 is_ours 为 0
        return NULL;
    }

    is_tc = FALSE;
    for (i = 0; i < numPorts; i++)
    {
        if (strcmp(g_TcFunctions.PortGetName(pPorts[i]), device) == 0)
        {
            is_tc = TRUE;
            break;
        }
    }

    if (numPorts > 0)
    {
        /*
         * 忽略此处的结果
         */
        (void)g_TcFunctions.FreePortList(pPorts);
    }

    if (!is_tc)
    {
        *is_ours = 0;
        // 设置 is_ours 为 0
        return NULL;
    }

    /* OK, it's probably ours. */
    // 好的，它可能是我们的
    *is_ours = 1;
    // 设置 is_ours 为 1

    p = PCAP_CREATE_COMMON(ebuf, struct pcap_tc);
    // 创建一个 pcap_t 结构
    if (p == NULL)
        return NULL;

    p->activate_op = TcActivate;
    // 设置 activate_op 操作为 TcActivate
    /*
     * 在前面设置这些，这样，即使我们的客户端在我们激活之前尝试设置非阻塞模式，
     * 或者查询非阻塞模式的状态，他们会得到一个错误，而不是在以后使用非阻塞模式选项。
     */
    p->getnonblock_op = TcGetNonBlock;
    p->setnonblock_op = TcSetNonBlock;
    返回 p;
# 设置数据链路类型，不需要做任何工作；pcap_set_datalink() 检查数值是否在我们提供的 DLT_ 值列表中，如果有效，则将 p->linktype 设置为新值；我们不需要在硬件上做任何操作，只需使用 p->linktype 中的值。
static int TcSetDatalink(pcap_t *p, int dlt)
{
    return 0;
}

# 获取非阻塞模式，不支持 TurboCap 端口的非阻塞模式
static int TcGetNonBlock(pcap_t *p)
{
    snprintf(p->errbuf, PCAP_ERRBUF_SIZE, "Non-blocking mode isn't supported for TurboCap ports");
    return -1;
}

# 设置非阻塞模式，不支持 TurboCap 端口的非阻塞模式
static int TcSetNonBlock(pcap_t *p, int nonblock)
{
    snprintf(p->errbuf, PCAP_ERRBUF_SIZE, "Non-blocking mode isn't supported for TurboCap ports");
    return -1;
}

# 清理资源，释放内存和关闭 TurboCap 实例
static void TcCleanup(pcap_t *p)
{
    struct pcap_tc *pt = p->priv;

    if (pt->TcPacketsBuffer != NULL)
    {
        g_TcFunctions.PacketsBufferDestroy(pt->TcPacketsBuffer);
        pt->TcPacketsBuffer = NULL;
    }
    if (pt->TcInstance != NULL)
    {
        g_TcFunctions.InstanceClose(pt->TcInstance);
        pt->TcInstance = NULL;
    }

    if (pt->PpiPacket != NULL)
    {
        free(pt->PpiPacket);
        pt->PpiPacket = NULL;
    }

    pcap_cleanup_live_common(p);
}

# 发送数据包到网络
static int TcInject(pcap_t *p, const void *buf, int size)
{
    struct pcap_tc *pt = p->priv;
    TC_STATUS status;
    TC_PACKETS_BUFFER buffer;
    TC_PACKET_HEADER header;

    # 如果数据包大小大于等于 0xFFFF，则返回错误
    if (size >= 0xFFFF)
    {
        snprintf(p->errbuf, PCAP_ERRBUF_SIZE, "send error: the TurboCap API does not support packets larger than 64k");
        return -1;
    }

    # 使用 TurboCap API 创建数据包缓冲区
    status = g_TcFunctions.PacketsBufferCreate(sizeof(TC_PACKET_HEADER) + TC_ALIGN_USHORT_TO_64BIT((USHORT)size), &buffer);
    # 如果发送状态不是成功，则记录错误信息并返回-1
    if (status != TC_SUCCESS)
    {
        snprintf(p->errbuf, PCAP_ERRBUF_SIZE, "send error: TcPacketsBufferCreate failure: %s (%08x)", g_TcFunctions.StatusGetString(status), status);
        return -1;
    }

    # 假设数据包没有校验和，通常在 WinPcap 中是这样
    memset(&header, 0, sizeof(header));

    # 设置数据包长度和捕获长度
    header.Length = (USHORT)size;
    header.CapturedLength = header.Length;

    # 提交下一个数据包到缓冲区
    status = g_TcFunctions.PacketsBufferCommitNextPacket(buffer, &header, (PVOID)buf);

    # 如果提交成功，则发送数据包
    if (status == TC_SUCCESS)
    {
        status = g_TcFunctions.InstanceTransmitPackets(pt->TcInstance, buffer);

        # 如果发送失败，则记录错误信息
        if (status != TC_SUCCESS)
        {
            snprintf(p->errbuf, PCAP_ERRBUF_SIZE, "send error: TcInstanceTransmitPackets failure: %s (%08x)", g_TcFunctions.StatusGetString(status), status);
        }
    }
    # 如果提交失败，则记录错误信息
    else
    {
        snprintf(p->errbuf, PCAP_ERRBUF_SIZE, "send error: TcPacketsBufferCommitNextPacket failure: %s (%08x)", g_TcFunctions.StatusGetString(status), status);
    }

    # 销毁缓冲区
    g_TcFunctions.PacketsBufferDestroy(buffer);

    # 如果发送状态不是成功，则返回-1，否则返回0
    if (status != TC_SUCCESS)
    {
        return -1;
    }
    else
    {
        return 0;
    }
}

static int TcRead(pcap_t *p, int cnt, pcap_handler callback, u_char *user)
{
    struct pcap_tc *pt = p->priv;
    TC_STATUS status;
    int n = 0;

    /*
     * Has "pcap_breakloop()" been called?
     */
    if (p->break_loop)
    {
        /*
         * Yes - clear the flag that indicates that it
         * has, and return -2 to indicate that we were
         * told to break out of the loop.
         */
        p->break_loop = 0;
        return -2;
    }

    if (pt->TcPacketsBuffer == NULL)
    {
        // 获取 TurboCap 实例接收的数据包
        status = g_TcFunctions.InstanceReceivePackets(pt->TcInstance, &pt->TcPacketsBuffer);
        if (status != TC_SUCCESS)
        {
            // 如果获取失败，设置错误信息并返回 -1
            snprintf(p->errbuf, PCAP_ERRBUF_SIZE, "read error, TcInstanceReceivePackets failure: %s (%08x)", g_TcFunctions.StatusGetString(status), status);
            return -1;
        }
    }

    while (TRUE)
    }

    return n;
}

static int
TcStats(pcap_t *p, struct pcap_stat *ps)
{
    struct pcap_tc *pt = p->priv;
    TC_STATISTICS statistics;
    TC_STATUS status;
    ULONGLONG counter;
    struct pcap_stat s;

    // 查询 TurboCap 实例的统计信息
    status = g_TcFunctions.InstanceQueryStatistics(pt->TcInstance, &statistics);

    if (status != TC_SUCCESS)
    {
        // 如果查询失败，设置错误信息并返回 -1
        snprintf(p->errbuf, PCAP_ERRBUF_SIZE, "TurboCap error in TcInstanceQueryStatistics: %s (%08x)", g_TcFunctions.StatusGetString(status), status);
        return -1;
    }

    // 初始化统计信息结构体
    memset(&s, 0, sizeof(s));

    // 查询接收数据包总数
    status = g_TcFunctions.StatisticsQueryValue(statistics, TC_COUNTER_INSTANCE_TOTAL_RX_PACKETS, &counter);
    if (status != TC_SUCCESS)
    {
        // 如果查询失败，设置错误信息并返回 -1
        snprintf(p->errbuf, PCAP_ERRBUF_SIZE, "TurboCap error in TcStatisticsQueryValue: %s (%08x)", g_TcFunctions.StatusGetString(status), status);
        return -1;
    }
    if (counter <= (ULONGLONG)0xFFFFFFFF)
    {
        s.ps_recv = (ULONG)counter;
    }
    else
    {
        s.ps_recv = 0xFFFFFFFF;
    }

    // 查询丢弃数据包总数
    status = g_TcFunctions.StatisticsQueryValue(statistics, TC_COUNTER_INSTANCE_RX_DROPPED_PACKETS, &counter);
    # 如果状态不等于成功
    if (status != TC_SUCCESS)
    {
        # 格式化错误信息到 p->errbuf
        snprintf(p->errbuf, PCAP_ERRBUF_SIZE, "TurboCap error in TcStatisticsQueryValue: %s (%08x)", g_TcFunctions.StatusGetString(status), status);
        # 返回 -1
        return -1;
    }
    # 如果 counter 小于等于 0xFFFFFFFF
    if (counter <= (ULONGLONG)0xFFFFFFFF)
    {
        # 设置 s.ps_ifdrop 和 s.ps_drop 为 counter 的值
        s.ps_ifdrop = (ULONG)counter;
        s.ps_drop = (ULONG)counter;
    }
    # 否则
    else
    {
        # 设置 s.ps_ifdrop 和 s.ps_drop 为 0xFFFFFFFF
        s.ps_ifdrop = 0xFFFFFFFF;
        s.ps_drop = 0xFFFFFFFF;
    }
#if defined(_WIN32) && defined(ENABLE_REMOTE)
    // 如果定义了_WIN32并且启用了远程功能，则将接受的数据包数量赋值给ps_capt
    s.ps_capt = pt->TcAcceptedCount;
#endif
    // 将s的值赋给指针ps
    *ps = s;

    // 返回0表示成功
    return 0;
}


#ifdef _WIN32
// 获取TurboCap的统计信息
static struct pcap_stat *
TcStatsEx(pcap_t *p, int *pcap_stat_size)
{
    // 获取pcap_t结构体中的私有数据
    struct pcap_tc *pt = p->priv;
    // 定义TC_STATISTICS和TC_STATUS结构体以及ULONGLONG类型的counter
    TC_STATISTICS statistics;
    TC_STATUS status;
    ULONGLONG counter;

    // 将pcap_stat_size的值设置为p->stat的大小
    *pcap_stat_size = sizeof (p->stat);

    // 调用g_TcFunctions的InstanceQueryStatistics方法获取统计信息
    status = g_TcFunctions.InstanceQueryStatistics(pt->TcInstance, &statistics);

    // 如果获取失败，则返回NULL
    if (status != TC_SUCCESS)
    {
        snprintf(p->errbuf, PCAP_ERRBUF_SIZE, "TurboCap error in TcInstanceQueryStatistics: %s (%08x)", g_TcFunctions.StatusGetString(status), status);
        return NULL;
    }

    // 将p->stat的值初始化为0
    memset(&p->stat, 0, sizeof(p->stat));

    // 获取接收数据包数量并赋值给counter
    status = g_TcFunctions.StatisticsQueryValue(statistics, TC_COUNTER_INSTANCE_TOTAL_RX_PACKETS, &counter);
    if (status != TC_SUCCESS)
    {
        snprintf(p->errbuf, PCAP_ERRBUF_SIZE, "TurboCap error in TcStatisticsQueryValue: %s (%08x)", g_TcFunctions.StatusGetString(status), status);
        return NULL;
    }
    // 根据counter的值设置ps_recv的值
    if (counter <= (ULONGLONG)0xFFFFFFFF)
    {
        p->stat.ps_recv = (ULONG)counter;
    }
    else
    {
        p->stat.ps_recv = 0xFFFFFFFF;
    }

    // 获取丢弃数据包数量并赋值给counter
    status = g_TcFunctions.StatisticsQueryValue(statistics, TC_COUNTER_INSTANCE_RX_DROPPED_PACKETS, &counter);
    if (status != TC_SUCCESS)
    {
        snprintf(p->errbuf, PCAP_ERRBUF_SIZE, "TurboCap error in TcStatisticsQueryValue: %s (%08x)", g_TcFunctions.StatusGetString(status), status);
        return NULL;
    }
    // 根据counter的值设置ps_ifdrop和ps_drop的值
    if (counter <= (ULONGLONG)0xFFFFFFFF)
    {
        p->stat.ps_ifdrop = (ULONG)counter;
        p->stat.ps_drop = (ULONG)counter;
    }
    else
    {
        p->stat.ps_ifdrop = 0xFFFFFFFF;
        p->stat.ps_drop = 0xFFFFFFFF;
    }

#if defined(_WIN32) && defined(ENABLE_REMOTE)
    // 如果定义了_WIN32并且启用了远程功能，则将接受的数据包数量赋值给ps_capt
    p->stat.ps_capt = pt->TcAcceptedCount;
#endif

    // 返回p->stat的地址
    return &p->stat;
}

/* 设置内核级捕获缓冲区的维度 */
static int
TcSetBuff(pcap_t *p, int dim)
{
    /*
     * XXX turbocap has an internal way of managing buffers.
     * And at the moment it's not configurable, so we just
     * silently ignore the request to set the buffer.
     */
    // 返回 0，表示忽略设置缓冲区的请求
    return 0;
# 设置 TurboCap 设备的模式，如果不是捕获模式则报错
static int
TcSetMode(pcap_t *p, int mode)
{
    if (mode != MODE_CAPT)
    {
        # 如果模式不是捕获模式，则设置错误信息并返回-1
        snprintf(p->errbuf, PCAP_ERRBUF_SIZE, "Mode %d not supported by TurboCap devices. TurboCap only supports capture.", mode);
        return -1;
    }

    # 如果是捕获模式则返回0
    return 0;
}

# 设置 TurboCap 设备的最小拷贝大小
static int
TcSetMinToCopy(pcap_t *p, int size)
{
    # 获取 TurboCap 设备的私有数据结构
    struct pcap_tc *pt = p->priv;
    TC_STATUS status;

    # 如果最小拷贝大小小于0，则设置错误信息并返回-1
    if (size < 0)
    {
        snprintf(p->errbuf, PCAP_ERRBUF_SIZE, "Mintocopy cannot be less than 0.");
        return -1;
    }

    # 调用 TurboCap 函数设置最小拷贝大小，并根据返回状态设置错误信息
    status = g_TcFunctions.InstanceSetFeature(pt->TcInstance, TC_INST_FT_MINTOCOPY, (ULONG)size);

    if (status != TC_SUCCESS)
    {
        snprintf(p->errbuf, PCAP_ERRBUF_SIZE, "TurboCap error setting the mintocopy: %s (%08x)", g_TcFunctions.StatusGetString(status), status);
    }

    return 0;
}

# 获取 TurboCap 设备的接收等待句柄
static HANDLE
TcGetReceiveWaitHandle(pcap_t *p)
{
    # 获取 TurboCap 设备的私有数据结构
    struct pcap_tc *pt = p->priv;

    # 返回 TurboCap 函数获取的接收等待句柄
    return g_TcFunctions.InstanceGetReceiveWaitHandle(pt->TcInstance);
}

# 执行 OID get 请求，但 TurboCap 设备不支持此操作
static int
TcOidGetRequest(pcap_t *p, bpf_u_int32 oid _U_, void *data _U_, size_t *lenp _U_)
{
    # 设置错误信息并返回错误码
    snprintf(p->errbuf, PCAP_ERRBUF_SIZE,
        "An OID get request cannot be performed on a TurboCap device");
    return PCAP_ERROR;
}

# 执行 OID set 请求，但 TurboCap 设备不支持此操作
static int
TcOidSetRequest(pcap_t *p, bpf_u_int32 oid _U_, const void *data _U_,
    size_t *lenp _U_)
{
    # 设置错误信息并返回错误码
    snprintf(p->errbuf, PCAP_ERRBUF_SIZE,
        "An OID set request cannot be performed on a TurboCap device");
    return PCAP_ERROR;
}

# 在 TurboCap 设备上无法进行批量传输数据包
static u_int
TcSendqueueTransmit(pcap_t *p, pcap_send_queue *queue _U_, int sync _U_)
{
    # 设置错误信息并返回0
    snprintf(p->errbuf, PCAP_ERRBUF_SIZE,
        "Packets cannot be bulk transmitted on a TurboCap device");
    return 0;
}

# 在 TurboCap 设备上无法设置用户缓冲区
static int
TcSetUserBuffer(pcap_t *p, int size _U_)
{
    # 设置错误信息并返回-1
    snprintf(p->errbuf, PCAP_ERRBUF_SIZE,
        "The user buffer cannot be set on a TurboCap device");
    return -1;
}

# 在 TurboCap 设备上无法进行实时转储
static int
TcLiveDump(pcap_t *p, char *filename _U_, int maxsize _U_, int maxpacks _U_)
{
    # 使用snprintf函数将错误信息写入p->errbuf，限制错误信息长度为PCAP_ERRBUF_SIZE
    snprintf(p->errbuf, PCAP_ERRBUF_SIZE,
        "Live packet dumping cannot be performed on a TurboCap device");
    # 返回-1，表示Live packet dumping不能在TurboCap设备上执行
    return -1;
# 结束TcLiveDumpEnded函数，返回Live packet dumping不能在TurboCap设备上执行的错误信息
static int
TcLiveDumpEnded(pcap_t *p, int sync _U_)
{
    # 使用snprintf函数将错误信息写入p->errbuf
    snprintf(p->errbuf, PCAP_ERRBUF_SIZE,
        "Live packet dumping cannot be performed on a TurboCap device");
    # 返回-1表示出错
    return -1;
}

# 返回空的PAirpcapHandle对象
static PAirpcapHandle
TcGetAirPcapHandle(pcap_t *p _U_)
{
    return NULL;
}
#endif
```