# `xmrig\src\3rdparty\adl\adl_structures.h`

```
// 版权声明，版权所有
// MIT 许可证：允许任何人免费获取并使用此软件及相关文档文件，包括但不限于使用、复制、修改、合并、发布、分发、再许可和出售
// 在使用软件时，需要包含上述版权声明和许可声明
// 本软件按原样提供，不提供任何形式的明示或暗示的担保，包括但不限于适销性、特定用途的适用性和非侵权性的担保
// 作者或版权持有人不对任何索赔、损害或其他责任负责，无论是合同行为、侵权行为还是其他行为，由此产生的、与之相关的或与之有关的
// 软件或使用或其他交易中的其他行为

/// \file adl_structures.h
///\brief This file contains the structure declarations that are used by the public ADL interfaces for \ALL platforms.\n <b>Included in ADL SDK</b>
/// 该文件包含了在所有平台上公共 ADL 接口中使用的结构声明。

#ifndef ADL_STRUCTURES_H_
#define ADL_STRUCTURES_H_

#include "adl_defines.h"

/////////////////////////////////////////////////////////////////////////////////////////////
///\brief Structure containing information about the graphics adapter.
/// 用于存储有关图形适配器的各种信息的结构。这些信息可以返回给用户，也可以用于访问各种驱动程序调用，以根据用户的请求设置或获取各种设置。
/// \nosubgrouping
// 定义一个结构体，包含有关适配器信息的结构
typedef struct AdapterInfo
{
    /// \ALL_STRUCT_MEM

    /// 结构体的大小
    int iSize;
    /// 与此适配器关联的 ADL 索引句柄。一个 GPU 可能与一个或两个索引句柄关联
    int iAdapterIndex;
    /// 与此适配器关联的唯一设备 ID
    char strUDID[ADL_MAX_PATH];
    /// 与此适配器关联的 BUS 号
    int iBusNumber;
    /// 与此适配器关联的驱动程序号
    int iDeviceNumber;
    /// 功能号
    int iFunctionNumber;
    /// 与此适配器关联的供应商 ID
    int iVendorID;
    /// 适配器名称
    char strAdapterName[ADL_MAX_PATH];
    /// 显示名称。例如，Windows 下的 "\\\\Display0" 或 Linux 下的 ":0:0"
    char strDisplayName[ADL_MAX_PATH];
    /// 是否存在；如果逻辑适配器存在，则可以从操作系统中找到显示名称，例如 \\\\.\\Display1
    int iPresent;

    #if defined (_WIN32) || defined (_WIN64)
    /// \WIN_STRUCT_MEM

    /// 是否存在；1 表示存在，0 表示不存在
    int iExist;
    /// 驱动程序注册表路径
    char strDriverPath[ADL_MAX_PATH];
    /// 驱动程序注册表路径扩展
    char strDriverPathExt[ADL_MAX_PATH];
    /// 来自 Windows 的 PNP 字符串
    char strPNPString[ADL_MAX_PATH];
    /// 从 EnumDisplayDevices 生成
    int iOSDisplayIndex;
    #endif /* (_WIN32) || (_WIN64) */

    #if defined (LINUX)
    /// \LNX_STRUCT_MEM

    /// 来自 GPUMapInfo 的内部 X 屏幕编号（已弃用，请使用 XScreenInfo）
    int iXScreenNum;
    /// 来自 GPUMapInfo 的内部驱动程序索引
    int iDrvIndex;
    /// \deprecated 内部 x 配置文件屏幕标识符名称。请改用 XScreenInfo
    char strXScreenConfigName[ADL_MAX_PATH];

    #endif /* (LINUX) */
} AdapterInfo, *LPAdapterInfo;

/////////////////////////////////////////////////////////////////////////////////////////////
///\brief 包含有关 Linux X 屏幕信息的结构。
///
/// This structure is used to store the current screen number and xorg.conf ID name assoicated with an adapter index.
/// This structure is updated during ADL_Main_Control_Refresh or ADL_ScreenInfo_Update.
/// Note:  This structure should be used in place of iXScreenNum and strXScreenConfigName in AdapterInfo as they will be
/// deprecated.
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
#if defined (LINUX)
typedef struct XScreenInfo
{
    /// Internal X screen number from GPUMapInfo.
    int iXScreenNum;
    /// Internal x config file screen identifier name.
    char strXScreenConfigName[ADL_MAX_PATH];
} XScreenInfo, *LPXScreenInfo;
#endif /* (LINUX) */


/////////////////////////////////////////////////////////////////////////////////////////////
///\brief Structure containing information about the ASIC memory.
///
/// This structure is used to store various information about the ASIC memory.  This
/// information can be returned to the user.
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
typedef struct ADLMemoryInfo
{
    /// Memory size in bytes.
    long long iMemorySize;
    /// Memory type in string.
    char strMemoryType[ADL_MAX_PATH];
    /// Memory bandwidth in Mbytes/s.
    long long iMemoryBandwidth;
} ADLMemoryInfo, *LPADLMemoryInfo;

/////////////////////////////////////////////////////////////////////////////////////////////
///\brief Structure containing information about memory required by type
///
/// This structure is returned by ADL_Adapter_ConfigMemory_Get, which given a desktop and display configuration
/// will return the Memory used.
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
typedef struct ADLMemoryRequired
{
    long long iMemoryReq;        /// Memory in bytes required
    int iType;                    /// Type of Memory \ref define_adl_validmemoryrequiredfields
    # 定义一个整型变量 iDisplayFeatureValue，用于存储使用该类型内存的显示特性值
    int iDisplayFeatureValue;   /// Display features \ref define_adl_visiblememoryfeatures that are using this type of memory
// 结构体，包含与显示相关特性信息
typedef struct ADLMemoryDisplayFeatures
{
    int iDisplayIndex;            /// ADL 显示索引
    int iDisplayFeatureValue;    /// 显示器正在使用的特性 \ref define_adl_visiblememoryfeatures
} ADLMemoryDisplayFeatures, *LPADLMemoryDisplayFeatures;

// 结构体，包含DDC信息
typedef struct ADLDDCInfo
{
    int  ulSize;  /// 结构体大小
    int  ulSupportsDDC;  /// 表示连接的显示器是否支持DDC。如果此字段在返回时为零，则不会使用其他DDC信息字段。
    int  ulManufacturerID;  /// 返回显示设备的制造商ID。如果此信息不可用，则应将其清零。
    int  ulProductID;  /// 返回显示设备的产品ID。如果此信息不可用，则应将其零化。
    char cDisplayName[ADL_MAX_DISPLAY_NAME];  /// 返回显示设备的名称。如果此信息不可用，则应将其零化。
    int  ulMaxHResolution;  /// 返回支持的最大水平分辨率。如果此信息不可用，则应将其零化。
/// Returns the maximum Vertical supported resolution. Should be zeroed if this information is not available.
    int  ulMaxVResolution;
/// Returns the maximum supported refresh rate. Should be zeroed if this information is not available.
    int  ulMaxRefresh;
/// Returns the display device preferred timing mode's horizontal resolution.
    int  ulPTMCx;
/// Returns the display device preferred timing mode's vertical resolution.
    int  ulPTMCy;
/// Returns the display device preferred timing mode's refresh rate.
    int  ulPTMRefreshRate;
/// Return EDID flags.
    int  ulDDCInfoFlag;
} ADLDDCInfo, *LPADLDDCInfo;

/////////////////////////////////////////////////////////////////////////////////////////////
///\brief Structure containing DDC information.
///
/// This structure is used to store various DDC information that can be returned to the user.
/// Note that all fields of type int are actually defined as unsigned int types within the driver.
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
typedef struct ADLDDCInfo2
{
/// Size of the structure
    int  ulSize;
/// Indicates whether the attached display supports DDC. If this field is zero on return, no other DDC
/// information fields will be used.
    int  ulSupportsDDC;
/// Returns the manufacturer ID of the display device. Should be zeroed if this information is not available.
    int  ulManufacturerID;
/// Returns the product ID of the display device. Should be zeroed if this information is not available.
    int  ulProductID;
/// Returns the name of the display device. Should be zeroed if this information is not available.
    char cDisplayName[ADL_MAX_DISPLAY_NAME];
/// Returns the maximum Horizontal supported resolution. Should be zeroed if this information is not available.
    int  ulMaxHResolution;
/// Returns the maximum Vertical supported resolution. Should be zeroed if this information is not available.
    int  ulMaxVResolution;
/// 返回支持的最大刷新率。如果此信息不可用，则应将其清零。
    int  ulMaxRefresh;
/// 返回显示设备首选定时模式的水平分辨率。
    int  ulPTMCx;
/// 返回显示设备首选定时模式的垂直分辨率。
    int  ulPTMCy;
/// 返回显示设备首选定时模式的刷新率。
    int  ulPTMRefreshRate;
/// 返回 EDID 标志。
    int  ulDDCInfoFlag;
/// 如果显示支持打包像素，则返回 1，否则返回 0
    int bPackedPixelSupported;
/// 返回显示支持的像素格式 \ref define_ddcinfo_pixelformats
    int iPanelPixelFormat;
/// 返回 EDID 序列号。
    int  ulSerialID;
/// 返回最小监视器亮度数据
    int ulMinLuminanceData;
/// 返回平均监视器亮度数据
    int ulAvgLuminanceData;
/// 返回最大监视器亮度数据
    int ulMaxLuminanceData;

/// 支持的传输函数的位向量 \ref define_source_content_TF
    int iSupportedTransferFunction;

/// 支持的色彩空间的位向量 \ref define_source_content_CS
    int iSupportedColorSpace;

/// 显示红色色度 X 坐标乘以 10000
    int iNativeDisplayChromaticityRedX;
/// 显示红色色度 Y 坐标乘以 10000
    int iNativeDisplayChromaticityRedY;
/// 显示绿色色度 X 坐标乘以 10000
    int iNativeDisplayChromaticityGreenX;
/// 显示绿色色度 Y 坐标乘以 10000
    int iNativeDisplayChromaticityGreenY;
/// 显示蓝色色度 X 坐标乘以 10000
    int iNativeDisplayChromaticityBlueX;
/// 显示蓝色色度 Y 坐标乘以 10000
    int iNativeDisplayChromaticityBlueY;
/// 显示白点 X 坐标乘以 10000
    int iNativeDisplayChromaticityWhitePointX;
/// 显示白点 Y 坐标乘以 10000
    int iNativeDisplayChromaticityWhitePointY;
/// Display diffuse screen reflectance 0-1 (100%) in units of 0.01
    int iDiffuseScreenReflectance;  // 显示漫反射屏幕反射率0-1（100%），单位为0.01
/// Display specular screen reflectance 0-1 (100%) in units of 0.01
    int iSpecularScreenReflectance;  // 显示镜面屏幕反射率0-1（100%），单位为0.01
/// Bit vector of supported color spaces \ref define_HDR_support
    int iSupportedHDR;  // 支持的颜色空间的位向量 \ref define_HDR_support
/// Bit vector for freesync flags
    int iFreesyncFlags;  // 用于 freesync 标志的位向量

/// Return minimum monitor luminance without dimming data
    int ulMinLuminanceNoDimmingData;  // 返回没有调光数据的最小监视器亮度
    int ulMaxBacklightMaxLuminanceData;  // 最大背光最大亮度数据
    int ulMinBacklightMaxLuminanceData;  // 最小背光最大亮度数据
    int ulMaxBacklightMinLuminanceData;  // 最大背光最小亮度数据
    int ulMinBacklightMinLuminanceData;  // 最小背光最小亮度数据

    // Reserved for future use
    int iReserved[4];  // 保留以供将来使用
} ADLDDCInfo2, *LPADLDDCInfo2;


/////////////////////////////////////////////////////////////////////////////////////////////
///\brief Structure containing information controller Gamma settings.
///
/// This structure is used to store the red, green and blue color channel information for the.
/// controller gamma setting. This information is returned by ADL, and it can also be used to
/// set the controller gamma setting.
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
typedef struct ADLGamma
{
/// Red color channel gamma value.
    float fRed;  // 红色通道的伽马值
/// Green color channel gamma value.
    float fGreen;  // 绿色通道的伽马值
/// Blue color channel gamma value.
    float fBlue;  // 蓝色通道的伽马值
} ADLGamma, *LPADLGamma;

/////////////////////////////////////////////////////////////////////////////////////////////
///\brief Structure containing information about component video custom modes.
///
/// This structure is used to store the component video custom mode.
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
typedef struct ADLCustomMode
{
/// Custom mode flags.  They are returned by the ADL driver.
    int iFlags;  // 自定义模式标志。它们由ADL驱动程序返回。
/// Custom mode width.
    int iModeWidth;  // 自定义模式宽度
/// Custom mode height.
    int iModeHeight;  // 自定义模式高度
/// Custom mode base width.
    # 定义一个整型变量 iBaseModeWidth
/// Custom mode base height.
// 自定义模式基本高度

    int iBaseModeHeight;
    // 自定义模式刷新率
    int iRefreshRate;
} ADLCustomMode, *LPADLCustomMode;

/////////////////////////////////////////////////////////////////////////////////////////////
///\brief Structure containing Clock information for OD5 calls.
///
/// This structure is used to retrieve clock information for OD5 calls.
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
// 包含 OD5 调用的时钟信息的结构体

typedef struct ADLGetClocksOUT
{
    // 高核心时钟
    long ulHighCoreClock;
    // 高内存时钟
    long ulHighMemoryClock;
    // 高 Vddc
    long ulHighVddc;
    // 核心最小值
    long ulCoreMin;
    // 核心最大值
    long ulCoreMax;
    // 内存最小值
    long ulMemoryMin;
    // 内存最大值
    long ulMemoryMax;
    // 活动百分比
    long ulActivityPercent;
    // 当前核心时钟
    long ulCurrentCoreClock;
    // 当前内存时钟
    long ulCurrentMemoryClock;
    // 保留字段
    long ulReserved;
} ADLGetClocksOUT;

/////////////////////////////////////////////////////////////////////////////////////////////
///\brief Structure containing HDTV information for display calls.
///
/// This structure is used to retrieve HDTV information information for display calls.
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
// 包含显示调用的 HDTV 信息的结构体

typedef struct ADLDisplayConfig
{
    /// 结构体的大小
  long ulSize;
  // HDTV 连接器类型
  long ulConnectorType;
  // HDTV 能力
  long ulDeviceData;
  // 覆盖的 HDTV 能力
  long ulOverridedDeviceData;
  // 保留字段
  long ulReserved;
} ADLDisplayConfig;


/////////////////////////////////////////////////////////////////////////////////////////////
///\brief Structure containing information about the display device.
///
/// This structure is used to store display device information
/// such as display index, type, name, connection status, mapped adapter and controller indexes,
/// whether or not multiple VPUs are supported, local display connections or not (through Lasso), etc.
// 包含有关显示设备信息的结构体
/// This information can be returned to the user. Alternatively, it can be used to access various driver calls to set
/// or fetch various display device related settings upon the user's request.
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
/// 结构体，包含有关显示设备的信息
///
/// 该结构用于存储有关显示设备的各种信息。这些信息可以返回给用户，也可以根据用户的请求用于访问各种驱动程序调用，以设置或获取与显示设备相关的各种设置
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
typedef struct ADLDisplayID
{
/// 该适配器所属的逻辑显示索引
    int iDisplayLogicalIndex;

///\brief 物理显示索引。
/// 例如，来自适配器 2 的显示索引 2 可以被当前适配器 1 使用。\n
/// 因此，当前适配器可能将此适配器枚举为逻辑显示 7，但物理显示索引仍为 2。
    int iDisplayPhysicalIndex;

/// 显示的持久逻辑适配器索引
    int iDisplayLogicalAdapterIndex;

///\brief 显示的持久物理适配器索引。
/// 它可以是当前适配器或非本地适配器。 \n
/// 如果此适配器索引与当前适配器不同，则在 DisplayInfoValue 中设置 Display Non Local 标志。
    int iDisplayPhysicalAdapterIndex;
} ADLDisplayID, *LPADLDisplayID;

/////////////////////////////////////////////////////////////////////////////////////////////
///\brief Structure containing information about the display device.
///
/// This structure is used to store various information about the display device.  This
/// information can be returned to the user, or used to access various driver calls to set
/// or fetch various display-device-related settings upon the user's request
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
/// 结构体，包含有关显示设备的信息
///
/// 该结构用于存储有关显示设备的各种信息。这些信息可以返回给用户，也可以根据用户的请求用于访问各种驱动程序调用，以设置或获取与显示设备相关的各种设置
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
typedef struct ADLDisplayInfo
{
/// 显示ID结构
    ADLDisplayID displayID;

///\deprecated 映射显示的控制器索引。\n 将来不会使用\n
    int  iDisplayControllerIndex;

/// 显示的EDID名称
    char strDisplayName[ADL_MAX_PATH];

/// 显示的制造商名称。
    # 定义一个字符数组，用于存储显示器制造商的名称，数组长度为ADL_MAX_PATH
    char strDisplayManufacturerName[ADL_MAX_PATH];
/// The Display type. For example: CRT, TV, CV, DFP.
    // 定义显示类型，例如：CRT，TV，CV，DFP
    int  iDisplayType;

/// The display output type. For example: HDMI, SVIDEO, COMPONMNET VIDEO.
    // 定义显示输出类型，例如：HDMI，SVIDEO，COMPONMNET VIDEO
    int  iDisplayOutputType;

/// The connector type for the device.
    // 设备的连接器类型
    int  iDisplayConnector;

///\brief The bit mask identifies the number of bits ADLDisplayInfo is currently using. \n
/// It will be the sum all the bit definitions in ADL_DISPLAY_DISPLAYINFO_xxx.
    // 位掩码标识 ADLDisplayInfo 当前使用的位数
    int  iDisplayInfoMask;

/// The bit mask identifies the display status. \ref define_displayinfomask
    // 位掩码标识显示状态
    int  iDisplayInfoValue;
} ADLDisplayInfo, *LPADLDisplayInfo;

/////////////////////////////////////////////////////////////////////////////////////////////
///\brief Structure containing information about the display port MST device.
///
/// This structure is used to store various MST information about the display port device.  This
/// information can be returned to the user, or used to access various driver calls to
/// fetch various display-device-related settings upon the user's request
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
typedef struct ADLDisplayDPMSTInfo
{
    /// The ADLDisplayID structure
    // ADLDisplayID 结构
    ADLDisplayID displayID;

    /// total bandwidth available on the DP connector
    // DP 连接器上可用的总带宽
    int    iTotalAvailableBandwidthInMpbs;
    /// bandwidth allocated to this display
    // 分配给此显示器的带宽
    int    iAllocatedBandwidthInMbps;

    // info from DAL DpMstSinkInfo
    /// string identifier for the display
    // 显示的字符串标识符
    char    strGlobalUniqueIdentifier[ADL_MAX_PATH];

    /// The link count of relative address, rad[0] upto rad[linkCount] are valid
    // 相对地址的链接计数，rad[0] 到 rad[linkCount] 是有效的
    int        radLinkCount;
    /// The physical connector ID, used to identify the physical DP port
    // 物理连接器 ID，用于标识物理 DP 端口
    int        iPhysicalConnectorID;

    /// Relative address, address scheme starts from source side
    // 相对地址，地址方案从源端开始
    char    rad[ADL_MAX_RAD_LINK_COUNT];
} ADLDisplayDPMSTInfo, *LPADLDisplayDPMSTInfo;
///\brief Structure containing the display mode definition used per controller.
///
/// This structure is used to store the display mode definition used per controller.
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
typedef struct ADLDisplayMode
{
/// Vertical resolution (in pixels).
   int  iPelsHeight;
/// Horizontal resolution (in pixels).
   int  iPelsWidth;
/// Color depth.
   int  iBitsPerPel;
/// Refresh rate.
   int  iDisplayFrequency;
} ADLDisplayMode;

/////////////////////////////////////////////////////////////////////////////////////////////
///\brief Structure containing detailed timing parameters.
///
/// This structure is used to store the detailed timing parameters.
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
typedef struct ADLDetailedTiming
{
/// Size of the structure.
     int   iSize;
/// Timing flags. \ref define_detailed_timing_flags
     short sTimingFlags;
/// Total width (columns).
     short sHTotal;
/// Displayed width.
     short sHDisplay;
/// Horizontal sync signal offset.
     short sHSyncStart;
/// Horizontal sync signal width.
     short sHSyncWidth;
/// Total height (rows).
     short sVTotal;
/// Displayed height.
     short sVDisplay;
/// Vertical sync signal offset.
     short sVSyncStart;
/// Vertical sync signal width.
     short sVSyncWidth;
/// Pixel clock value.
     short sPixelClock;
/// Overscan right.
     short sHOverscanRight;
/// Overscan left.
     short sHOverscanLeft;
/// Overscan bottom.
     short sVOverscanBottom;
/// Overscan top.
     short sVOverscanTop;
     short sOverscan8B;
     short sOverscanGR;
} ADLDetailedTiming;

/////////////////////////////////////////////////////////////////////////////////////////////
///\brief Structure containing display mode information.
///
/// This structure is used to store the display mode information.
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
typedef struct ADLDisplayModeInfo
{
/// Timing standard of the current mode. \ref define_modetiming_standard
  int  iTimingStandard;  // 当前模式的定时标准
/// Applicable timing standards for the current mode.
  int  iPossibleStandard;  // 当前模式的适用定时标准
/// Refresh rate factor.
  int  iRefreshRate;  // 刷新率因子
/// Num of pixels in a row.
  int  iPelsWidth;  // 一行中的像素数
/// Num of pixels in a column.
  int  iPelsHeight;  // 一列中的像素数
/// Detailed timing parameters.
  ADLDetailedTiming  sDetailedTiming;  // 详细的定时参数
} ADLDisplayModeInfo;

/////////////////////////////////////////////////////////////////////////////////////////////
/// \brief Structure containing information about display property.
///
/// This structure is used to store the display property for the current adapter.
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
typedef struct ADLDisplayProperty
{
/// Must be set to sizeof the structure
  int iSize;  // 必须设置为结构体的大小
/// Must be set to \ref ADL_DL_DISPLAYPROPERTY_TYPE_EXPANSIONMODE or \ref ADL_DL_DISPLAYPROPERTY_TYPE_USEUNDERSCANSCALING
  int iPropertyType;  // 必须设置为 \ref ADL_DL_DISPLAYPROPERTY_TYPE_EXPANSIONMODE 或 \ref ADL_DL_DISPLAYPROPERTY_TYPE_USEUNDERSCANSCALING
/// Get or Set \ref ADL_DL_DISPLAYPROPERTY_EXPANSIONMODE_CENTER or \ref ADL_DL_DISPLAYPROPERTY_EXPANSIONMODE_FULLSCREEN or \ref ADL_DL_DISPLAYPROPERTY_EXPANSIONMODE_ASPECTRATIO or \ref ADL_DL_DISPLAYPROPERTY_TYPE_ITCFLAGENABLE
  int iExpansionMode;  // 获取或设置 \ref ADL_DL_DISPLAYPROPERTY_EXPANSIONMODE_CENTER 或 \ref ADL_DL_DISPLAYPROPERTY_EXPANSIONMODE_FULLSCREEN 或 \ref ADL_DL_DISPLAYPROPERTY_EXPANSIONMODE_ASPECTRATIO 或 \ref ADL_DL_DISPLAYPROPERTY_TYPE_ITCFLAGENABLE
/// Display Property supported? 1: Supported, 0: Not supported
  int iSupport;  // 显示属性是否支持？1：支持，0：不支持
/// Display Property current value
  int iCurrent;  // 显示属性的当前值
/// Display Property Default value
  int iDefault;  // 显示属性的默认值
} ADLDisplayProperty;

/////////////////////////////////////////////////////////////////////////////////////////////
///\brief Structure containing information about Clock.
///
/// This structure is used to store the clock information for the current adapter
/// such as core clock and memory clock info.
///\nosubgrouping
// 定义了一个结构体 ADLClockInfo，用于存储核心时钟和内存时钟的信息
typedef struct ADLClockInfo
{
    // 核心时钟，单位为10KHz
    int iCoreClock;
    // 内存时钟，单位为10KHz
    int iMemoryClock;
} ADLClockInfo, *LPADLClockInfo;

// 定义了一个结构体 ADLI2C，用于存储 I2C 信息
typedef struct ADLI2C
{
    // 结构体的大小
    int iSize;
    // 表示硬件 I2C 的数值
    int iLine;
    // 7 位 I2C 从设备地址，左移一位
    int iAddress;
    // 数据偏移量
    int iOffset;
    // 读取或写入从设备的动作
    int iAction;
    // I2C 时钟速度，单位为KHz
    int iSpeed;
    // 在 I2C 总线上发送或接收的字节数
    int iDataSize;
    // 要在 I2C 总线上发送或接收的字符的地址
    char *pcData;
} ADLI2C;

// 定义了一个结构体 ADLDisplayEDIDData，用于存储 EDID 数据的信息
typedef struct ADLDisplayEDIDData
{
    // 结构体的大小
    int iSize;
/// Set to 0
  int iFlag;
  /// Size of cEDIDData. Set by ADL_Display_EdidData_Get() upon return
  int iEDIDSize;
/// 0, 1 or 2. If set to 3 or above an error ADL_ERR_INVALID_PARAM is generated
  int iBlockIndex;
/// EDID data
  char cEDIDData[ADL_MAX_EDIDDATA_SIZE];
/// Reserved
  int iReserved[4];
}ADLDisplayEDIDData;

/////////////////////////////////////////////////////////////////////////////////////////////
///\brief Structure containing information about input of controller overlay adjustment.
///
/// This structure is used to store the information about input of controller overlay adjustment for the adapter.
/// This structure is used by the ADL_Display_ControllerOverlayAdjustmentCaps_Get, ADL_Display_ControllerOverlayAdjustmentData_Get, and
/// ADL_Display_ControllerOverlayAdjustmentData_Set() functions.
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
typedef struct ADLControllerOverlayInput
{
/// Should be set to the sizeof the structure
  int  iSize;
///\ref ADL_DL_CONTROLLER_OVERLAY_ALPHA or \ref ADL_DL_CONTROLLER_OVERLAY_ALPHAPERPIX
  int  iOverlayAdjust;
/// Data.
  int  iValue;
/// Should be 0.
  int  iReserved;
} ADLControllerOverlayInput;

/////////////////////////////////////////////////////////////////////////////////////////////
///\brief Structure containing information about overlay adjustment.
///
/// This structure is used to store the information about overlay adjustment for the adapter.
/// This structure is used by the ADLControllerOverlayInfo() function.
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
typedef struct ADLAdjustmentinfo
{
/// Default value
  int iDefault;
/// Minimum value
  int iMin;
/// Maximum Value
  int iMax;
/// Step value
  int iStep;
} ADLAdjustmentinfo;

/////////////////////////////////////////////////////////////////////////////////////////////
///\brief 用于存储有关控制器叠加信息的结构。
///
/// 该结构用于存储有关适配器的控制器叠加信息。该结构由ADL_Display_ControllerOverlayAdjustmentCaps_Get()函数使用。
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
typedef struct ADLControllerOverlayInfo
{
/// 应设置为结构的大小
  int                    iSize;
/// 数据
  ADLAdjustmentinfo        sOverlayInfo;
/// 应为0
  int                    iReserved[3];
} ADLControllerOverlayInfo;

/////////////////////////////////////////////////////////////////////////////////////////////
///\brief 包含GL-Sync模块信息的结构。
///
/// 该结构用于检索工作站帧锁定/Genlock的GL-Sync模块信息。
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
typedef struct ADLGLSyncModuleID
{
/// 唯一的GL-Sync模块ID
    int        iModuleID;
/// GL-Sync GPU端口索引（传递给ADLGLSyncGenlockConfig.lSignalSource和ADLGlSyncPortControl.lSignalSource）
    int        iGlSyncGPUPort;
/// GL-Sync模块引导扇区的固件版本
    int        iFWBootSectorVersion;
/// GL-Sync模块用户扇区的固件版本
    int        iFWUserSectorVersion;
} ADLGLSyncModuleID , *LPADLGLSyncModuleID;

/////////////////////////////////////////////////////////////////////////////////////////////
///\brief 包含GL-Sync端口能力的结构。
///
/// 该结构用于检索GL-Sync模块端口的硬件能力
/// 用于工作站帧锁定/Genlock（例如端口类型和关联LED的数量）。
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
typedef struct ADLGLSyncPortCaps
{
/// Port type. Bitfield of ADL_GLSYNC_PORTTYPE_*  \ref define_glsync
    int        iPortType;
/// Number of LEDs associated for this port.
    int        iNumOfLEDs;
}ADLGLSyncPortCaps, *LPADLGLSyncPortCaps;

/////////////////////////////////////////////////////////////////////////////////////////////
///\brief Structure containing GL-Sync Genlock settings.
///
/// This structure is used to get and set genlock settings for the GPU ports of the GL-Sync module
/// for Workstation Framelock/Genlock.\n
/// \see define_glsync
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
typedef struct ADLGLSyncGenlockConfig
{
/// Specifies what fields in this structure are valid \ref define_glsync
    int        iValidMask;
/// Delay (ms) generating a sync signal.
    int        iSyncDelay;
/// Vector of framelock control bits. Bitfield of ADL_GLSYNC_FRAMELOCKCNTL_* \ref define_glsync
    int        iFramelockCntlVector;
/// Source of the sync signal. Either GL_Sync GPU Port index or ADL_GLSYNC_SIGNALSOURCE_* \ref define_glsync
    int        iSignalSource;
/// Use sampled sync signal. A value of 0 specifies no sampling.
    int        iSampleRate;
/// For interlaced sync signals, the value can be ADL_GLSYNC_SYNCFIELD_1 or *_BOTH \ref define_glsync
    int        iSyncField;
/// The signal edge that should trigger synchronization. ADL_GLSYNC_TRIGGEREDGE_* \ref define_glsync
    int        iTriggerEdge;
/// Scan rate multiplier applied to the sync signal. ADL_GLSYNC_SCANRATECOEFF_* \ref define_glsync
    int        iScanRateCoeff;
}ADLGLSyncGenlockConfig, *LPADLGLSyncGenlockConfig;

/////////////////////////////////////////////////////////////////////////////////////////////
///\brief Structure containing GL-Sync port information.
///
/// This structure is used to get status of the GL-Sync ports (BNC or RJ45s)
/// for Workstation Framelock/Genlock.
/// \see define_glsync
/// \nosubgrouping
// 定义了一个结构体 ADLGlSyncPortInfo，用于存储 GL-Sync 端口信息
typedef struct ADLGlSyncPortInfo
{
    /// GL-Sync 端口的类型 (ADL_GLSYNC_PORT_*)
    int        iPortType;
    /// 该端口的 LED 数量，也在 ADLGLSyncPortCaps 中填充
    int        iNumOfLEDs;
    /// 端口状态 ADL_GLSYNC_PORTSTATE_*  \ref define_glsync
    int        iPortState;
    /// 该端口的扫描频率 (以毫赫兹为单位的垂直刷新率；60000 表示 60 Hz)
    int        iFrequency;
    /// 用于 ADL_GLSYNC_PORT_BNC。它是 ADL_GLSYNC_SIGNALTYPE_*   \ref define_glsync
    int        iSignalType;
    /// 用于 ADL_GLSYNC_PORT_RJ45PORT*。它是 GL_Sync GPU 端口索引或 ADL_GLSYNC_SIGNALSOURCE_*。  \ref define_glsync
    int        iSignalSource;

} ADLGlSyncPortInfo, *LPADLGlSyncPortInfo;

/////////////////////////////////////////////////////////////////////////////////////////////
///\brief 包含 GL-Sync 端口控制设置的结构体。
///
/// 该结构用于配置 GL-Sync 端口 (仅限 RJ45) 用于工作站帧锁定/同步。
/// \see define_glsync
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
typedef struct ADLGlSyncPortControl
{
    /// 要控制的端口 ADL_GLSYNC_PORT_RJ45PORT1 或 ADL_GLSYNC_PORT_RJ45PORT2   \ref define_glsync
    int        iPortType;
    /// 端口控制数据 ADL_GLSYNC_PORTCNTL_*   \ref define_glsync
    int        iControlVector;
    /// 同步信号的来源。可以是 GL_Sync GPU 端口索引或 ADL_GLSYNC_SIGNALSOURCE_*   \ref define_glsync
    int        iSignalSource;
} ADLGlSyncPortControl;

/////////////////////////////////////////////////////////////////////////////////////////////
///\brief 包含显示器的 GL-Sync 模式的结构体。
///
/// 该结构用于获取和设置连接到 GL-Sync 模块所附适配器的显示器的 GL-Sync 模式设置，用于工作站帧锁定/同步。
/// \see define_glsync
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
/// 定义了ADLGlSyncMode结构体，用于存储GL-Sync模式的相关信息
typedef struct ADLGlSyncMode
{
    /// 模式控制向量。ADL_GLSYNC_MODECNTL_*的位域   \ref define_glsync
    int        iControlVector;
    /// 模式状态向量。ADL_GLSYNC_MODECNTL_STATUS_*的位域   \ref define_glsync
    int        iStatusVector;
    /// 用于同步显示/控制器的GL-Sync连接器的索引
    int        iGLSyncConnectorIndex;
} ADLGlSyncMode, *LPADLGlSyncMode;

/////////////////////////////////////////////////////////////////////////////////////////////
///\brief 包含显示器的GL-Sync模式的结构体。
///
/// 该结构体用于获取和设置连接到GL-Sync模块的适配器上的显示器的GL-Sync模式设置，用于工作站帧锁定/同步。
/// \see define_glsync
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
typedef struct ADLGlSyncMode2
{
    /// 模式控制向量。ADL_GLSYNC_MODECNTL_*的位域   \ref define_glsync
    int        iControlVector;
    /// 模式状态向量。ADL_GLSYNC_MODECNTL_STATUS_*的位域   \ref define_glsync
    int        iStatusVector;
    /// 用于同步显示/控制器的GL-Sync连接器的索引
    int        iGLSyncConnectorIndex;
    /// 适用于此GLSync的显示器的索引
    int        iDisplayIndex;
} ADLGlSyncMode2, *LPADLGlSyncMode2;


/////////////////////////////////////////////////////////////////////////////////////////////
///\brief 包含显示器的数据包信息的结构体。
///
/// 该结构体用于获取和设置显示器的数据包信息。
/// 该结构体被ADLDisplayDataPacket使用。
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
typedef struct  ADLInfoPacket
{
    char hb0;
    char hb1;
    char hb2;
    /// sb0~sb27
    char sb[28];
}ADLInfoPacket;
// 定义了一个结构体，用于存储显示器的 AVI 包信息
// 该结构体用于获取和设置显示器的 AVI 包信息
// 该结构体被 ADLDisplayDataPacket 使用
// 不包含子组
typedef struct ADLAVIInfoPacket  //Valid user defined data/
{
    // 字节 3，第 7 位
    char bPB3_ITC;
    // 字节 5，位 [7:4]
    char bPB5;
}ADLAVIInfoPacket;

// Overdrive 时钟设置结构体定义

// 定义了一个结构体，用于存储 Overdrive 时钟设置
// 该结构体用于获取 Overdrive 时钟设置
// 该结构体被 ADLAdapterODClockInfo 使用
// 不包含子组
typedef struct ADLODClockSetting
{
    // 默认时钟
    int iDefaultClock;
    // 当前时钟
    int iCurrentClock;
    // 最大时钟
    int iMaxClock;
    // 最小时钟
    int iMinClock;
    // 请求的时钟
    int iRequestedClock;
    // 步长
    int iStepClock;
} ADLODClockSetting;

// 定义了一个结构体，用于存储 Overdrive 时钟信息
// 该结构体用于获取 Overdrive 时钟信息
// 该结构体被 ADL_Display_ODClockInfo_Get() 函数使用
// 不包含子组
typedef struct ADLAdapterODClockInfo
{
    // 结构体的大小
    int iSize;
    // 标志 \ref define_clockinfo_flags
    int iFlags;
    // 内存时钟
    ADLODClockSetting sMemoryClock;
    // 引擎时钟
    ADLODClockSetting sEngineClock;
} ADLAdapterODClockInfo;
///\brief 结构体，包含超频时钟配置信息。
///
/// 该结构体用于设置超频时钟配置。
/// 该结构体由 ADL_Display_ODClockConfig_Set() 函数使用。
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
typedef struct ADLAdapterODClockConfig
{
/// 结构体的大小
  int iSize;
/// 标志 \ref define_clockinfo_flags
  int iFlags;
/// 内存时钟
  int iMemoryClock;
/// 核心时钟
  int iEngineClock;
} ADLAdapterODClockConfig;

/////////////////////////////////////////////////////////////////////////////////////////////
///\brief 结构体，包含当前电源管理相关活动的信息。
///
/// 该结构体用于存储当前电源管理相关活动的信息。
/// 该结构体（Overdrive 5 接口）由 ADL_PM_CurrentActivity_Get() 函数使用。
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
typedef struct ADLPMActivity
{
/// 必须设置为结构体的大小
    int iSize;
/// 当前核心时钟。
    int iEngineClock;
/// 当前内存时钟。
    int iMemoryClock;
/// 当前核心电压。
    int iVddc;
/// GPU 利用率。
    int iActivityPercent;
/// 性能级别索引。
    int iCurrentPerformanceLevel;
/// 当前 PCIE 总线速度。
    int iCurrentBusSpeed;
/// PCIE 总线的数量。
    int iCurrentBusLanes;
/// 最大 PCIE 总线的数量。
    int iMaximumBusLanes;
/// 保留字段，用于将来使用。
    int iReserved;
} ADLPMActivity;

/////////////////////////////////////////////////////////////////////////////////////////////
///\brief 结构体，包含关于热控制器的信息。
///
/// 该结构体用于存储关于热控制器的信息。
/// This structure is used by ADL_PM_ThermalDevices_Enum.
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
// 定义了用于 ADL_PM_ThermalDevices_Enum 的结构体
typedef struct ADLThermalControllerInfo
{
/// Must be set to the size of the structure
  // 必须设置为结构体的大小
  int iSize;
/// Possible valies: \ref ADL_DL_THERMAL_DOMAIN_OTHER or \ref ADL_DL_THERMAL_DOMAIN_GPU.
  // 可能的取值：\ref ADL_DL_THERMAL_DOMAIN_OTHER 或 \ref ADL_DL_THERMAL_DOMAIN_GPU
  int iThermalDomain;
///    GPU 0, 1, etc.
  // GPU 0、1 等
  int iDomainIndex;
/// Possible valies: \ref ADL_DL_THERMAL_FLAG_INTERRUPT or \ref ADL_DL_THERMAL_FLAG_FANCONTROL
  // 可能的取值：\ref ADL_DL_THERMAL_FLAG_INTERRUPT 或 \ref ADL_DL_THERMAL_FLAG_FANCONTROL
  int iFlags;
} ADLThermalControllerInfo;

/////////////////////////////////////////////////////////////////////////////////////////////
///\brief Structure containing information about thermal controller temperature.
///
/// This structure is used to store information about thermal controller temperature.
/// This structure is used by the ADL_PM_Temperature_Get() function.
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
// 包含有关热控制器温度信息的结构
typedef struct ADLTemperature
{
/// Must be set to the size of the structure
  // 必须设置为结构体的大小
  int iSize;
/// Temperature in millidegrees Celsius.
  // 温度以毫摄氏度为单位
  int iTemperature;
} ADLTemperature;

/////////////////////////////////////////////////////////////////////////////////////////////
///\brief Structure containing information about thermal controller fan speed.
///
/// This structure is used to store information about thermal controller fan speed.
/// This structure is used by the ADL_PM_FanSpeedInfo_Get() function.
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
// 包含有关热控制器风扇速度信息的结构
typedef struct ADLFanSpeedInfo
{
/// Must be set to the size of the structure
  // 必须设置为结构体的大小
  int iSize;
/// \ref define_fanctrl
  // \ref define_fanctrl
  int iFlags;
/// Minimum possible fan speed value in percents.
  // 最小可能的风扇速度值（百分比）
  int iMinPercent;
/// Maximum possible fan speed value in percents.
  // 最大可能的风扇速度值（百分比）
  int iMaxPercent;
/// Minimum possible fan speed value in RPM.
  // 最小可能的风扇速度值（RPM）
  int iMinRPM;
/// Maximum possible fan speed value in RPM.
  int iMaxRPM;
} ADLFanSpeedInfo;

/////////////////////////////////////////////////////////////////////////////////////////////
///\brief Structure containing information about fan speed reported by thermal controller.
///
/// This structure is used to store information about fan speed reported by thermal controller.
/// This structure is used by the ADL_Overdrive5_FanSpeed_Get() and ADL_Overdrive5_FanSpeed_Set() functions.
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
typedef struct ADLFanSpeedValue
{
/// Must be set to the size of the structure
  int iSize;
/// Possible valies: \ref ADL_DL_FANCTRL_SPEED_TYPE_PERCENT or \ref ADL_DL_FANCTRL_SPEED_TYPE_RPM
  int iSpeedType;
/// Fan speed value
  int iFanSpeed;
/// The only flag for now is: \ref ADL_DL_FANCTRL_FLAG_USER_DEFINED_SPEED
  int iFlags;
} ADLFanSpeedValue;

////////////////////////////////////////////////////////////////////////////////////////////
///\brief Structure containing the range of Overdrive parameter.
///
/// This structure is used to store information about the range of Overdrive parameter.
/// This structure is used by ADLODParameters.
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
typedef struct ADLODParameterRange
{
/// Minimum parameter value.
  int iMin;
/// Maximum parameter value.
  int iMax;
/// Parameter step value.
  int iStep;
} ADLODParameterRange;

/////////////////////////////////////////////////////////////////////////////////////////////
///\brief Structure containing information about Overdrive parameters.
///
/// This structure is used to store information about Overdrive parameters.
/// This structure is used by the ADL_Overdrive5_ODParameters_Get() function.
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
typedef struct ADLODParameters
/// 必须设置为结构体的大小
  int iSize;
/// 标准性能状态的数量
  int iNumberOfPerformanceLevels;
/// 指示 GPU 是否能够测量其活动
  int iActivityReportingSupported;
/// 指示 GPU 是否支持离散性能级别或性能范围
  int iDiscretePerformanceLevels;
/// 保留以供将来使用
  int iReserved;
/// 引擎时钟范围
  ADLODParameterRange sEngineClock;
/// 内存时钟范围
  ADLODParameterRange sMemoryClock;
/// 核心电压范围
  ADLODParameterRange sVddc;
} ADLODParameters;

/////////////////////////////////////////////////////////////////////////////////////////////
///\brief 包含有关超频级别的信息的结构体。
///
/// 该结构用于存储有关超频级别的信息。
/// 该结构由 ADLODPerformanceLevels 使用。
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
typedef struct ADLODPerformanceLevel
{
/// 引擎时钟
  int iEngineClock;
/// 内存时钟
  int iMemoryClock;
/// 核心电压
  int iVddc;
} ADLODPerformanceLevel;

/////////////////////////////////////////////////////////////////////////////////////////////
///\brief 包含有关超频性能级别的信息的结构体。
///
/// 该结构用于存储有关超频性能级别的信息。
/// 该结构由 ADL_Overdrive5_ODPerformanceLevels_Get() 和 ADL_Overdrive5_ODPerformanceLevels_Set() 函数使用。
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
typedef struct ADLODPerformanceLevels
{
/// 必须设置为 sizeof( \ref ADLODPerformanceLevels ) + sizeof( \ref ADLODPerformanceLevel ) * (ADLODParameters.iNumberOfPerformanceLevels - 1)
  int iSize;
  int iReserved;
/// Array of performance state descriptors. Must have ADLODParameters.iNumberOfPerformanceLevels elements.
  ADLODPerformanceLevel aLevels [1];
} ADLODPerformanceLevels;

/////////////////////////////////////////////////////////////////////////////////////////////
///\brief Structure containing information about the proper CrossfireX chains combinations.
///
/// This structure is used to store information about the CrossfireX chains combination for a particular adapter.
/// This structure is used by the ADL_Adapter_Crossfire_Caps(), ADL_Adapter_Crossfire_Get(), and ADL_Adapter_Crossfire_Set() functions.
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
typedef struct ADLCrossfireComb
{
/// Number of adapters in this combination.
  int iNumLinkAdapter;
/// A list of ADL indexes of the linked adapters in this combination.
  int iAdaptLink[3];
} ADLCrossfireComb;

/////////////////////////////////////////////////////////////////////////////////////////////
///\brief Structure containing CrossfireX state and error information.
///
/// This structure is used to store state and error information about a particular adapter CrossfireX combination.
/// This structure is used by the ADL_Adapter_Crossfire_Get() function.
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
typedef struct ADLCrossfireInfo
{
/// Current error code of this CrossfireX combination.
  int iErrorCode;
/// Current \ref define_crossfirestate
  int iState;
/// If CrossfireX is supported by this combination. The value is either \ref ADL_TRUE or \ref ADL_FALSE.
  int iSupported;
} ADLCrossfireInfo;

/////////////////////////////////////////////////////////////////////////////////////////////
/// \brief Structure containing information about the BIOS.
///
/// This structure is used to store various information about the Chipset.  This
/// information can be returned to the user.
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
// 定义了包含 BIOS 信息的结构体
typedef struct ADLBiosInfo
{
    char strPartNumber[ADL_MAX_PATH];    ///< Part number.  // 部件号
    char strVersion[ADL_MAX_PATH];        ///< Version number.  // 版本号
    char strDate[ADL_MAX_PATH];        ///< BIOS date in yyyy/mm/dd hh:mm format.  // BIOS 日期，格式为年/月/日 时:分
} ADLBiosInfo, *LPADLBiosInfo;


/////////////////////////////////////////////////////////////////////////////////////////////
/// \brief Structure containing information about adapter location.
///
/// This structure is used to store information about adapter location.
/// This structure is used by ADLMVPUStatus.
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
// 定义了包含适配器位置信息的结构体
typedef struct ADLAdapterLocation
{
/// PCI Bus number : 8 bits
    int iBus;  // PCI 总线号：8 位
/// Device number : 5 bits
    int iDevice;  // 设备号：5 位
/// Function number : 3 bits
    int iFunction;  // 功能号：3 位
} ADLAdapterLocation,ADLBdf;

/////////////////////////////////////////////////////////////////////////////////////////////
/// \brief Structure containing version information
///
/// This structure is used to store software version information, description of the display device and a web link to the latest installed Catalyst drivers.
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
// 定义了包含版本信息的结构体
typedef struct ADLVersionsInfo
{
    /// Driver Release (Packaging) Version (e.g. 8.71-100128n-094835E-ATI)
    char strDriverVer[ADL_MAX_PATH];  // 驱动程序版本（打包版本）（例如 8.71-100128n-094835E-ATI）
    /// Catalyst Version(e.g. "10.1").
    char strCatalystVersion[ADL_MAX_PATH];  // Catalyst 版本（例如 "10.1"）
    /// Web link to an XML file with information about the latest AMD drivers and locations (e.g. "http://www.amd.com/us/driverxml" )
    char strCatalystWebLink[ADL_MAX_PATH];  // 包含有关最新 AMD 驱动程序和位置信息的 XML 文件的 Web 链接（例如 "http://www.amd.com/us/driverxml"）
} ADLVersionsInfo, *LPADLVersionsInfo;

/////////////////////////////////////////////////////////////////////////////////////////////
/// \brief Structure containing version information
///
/// This structure is used to store software version information, description of the display device and a web link to the latest installed Catalyst drivers.
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
typedef struct ADLVersionsInfoX2
{
    /// Driver Release (Packaging) Version (e.g. "16.20.1035-160621a-303814C")
    char strDriverVer[ADL_MAX_PATH];
    /// Catalyst Version(e.g. "15.8").
    char strCatalystVersion[ADL_MAX_PATH];
    /// Crimson Version(e.g. "16.6.2").
    char strCrimsonVersion[ADL_MAX_PATH];
    /// Web link to an XML file with information about the latest AMD drivers and locations (e.g. "http://support.amd.com/drivers/xml/driver_09_us.xml" )
    char strCatalystWebLink[ADL_MAX_PATH];

} ADLVersionsInfoX2, *LPADLVersionsInfoX2;

/////////////////////////////////////////////////////////////////////////////////////////////
/// \brief Structure containing information about MultiVPU capabilities.
///
/// This structure is used to store information about MultiVPU capabilities.
/// This structure is used by the ADL_Display_MVPUCaps_Get() function.
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
typedef struct ADLMVPUCaps
{
/// Must be set to sizeof( ADLMVPUCaps ).
  int iSize;
/// Number of adapters.
  int iAdapterCount;
/// Bits set for all possible MVPU masters. \ref MVPU_ADAPTER_0 .. \ref MVPU_ADAPTER_3
  int iPossibleMVPUMasters;
/// Bits set for all possible MVPU slaves. \ref MVPU_ADAPTER_0 .. \ref MVPU_ADAPTER_3
  int iPossibleMVPUSlaves;
/// Registry path for each adapter.
  char cAdapterPath[ADL_DL_MAX_MVPU_ADAPTERS][ADL_DL_MAX_REGISTRY_PATH];
} ADLMVPUCaps;

/////////////////////////////////////////////////////////////////////////////////////////////
/// \brief Structure containing information about MultiVPU status.
///
/// This structure is used to store information about MultiVPU status.
/// Ths structure is used by the ADL_Display_MVPUStatus_Get() function.
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
typedef struct ADLMVPUStatus
{
/// Must be set to sizeof( ADLMVPUStatus ).
  int iSize;
/// Number of active adapters.
  int iActiveAdapterCount;
/// MVPU status.
  int iStatus;
/// PCI Bus/Device/Function for each active adapter participating in MVPU.
  ADLAdapterLocation aAdapterLocation[ADL_DL_MAX_MVPU_ADAPTERS];
} ADLMVPUStatus;

// Displays Manager structures

///////////////////////////////////////////////////////////////////////////
/// \brief Structure containing information about the activatable source.
///
/// This structure is used to store activatable source information
/// This information can be returned to the user.
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
typedef struct ADLActivatableSource
{
    /// The Persistent logical Adapter Index.
    int iAdapterIndex;
    /// The number of Activatable Sources.
    int iNumActivatableSources;
    /// The bit mask identifies the number of bits ActivatableSourceValue is using. (Not currnetly used)
    int iActivatableSourceMask;
    /// The bit mask identifies the status.  (Not currnetly used)
    int iActivatableSourceValue;
} ADLActivatableSource, *LPADLActivatableSource;

/////////////////////////////////////////////////////////////////////////////////////////////
/// \brief Structure containing information about display mode.
///
/// This structure is used to store the display mode for the current adapter
/// such as X, Y positions, screen resolutions, orientation,
/// color depth, refresh rate, progressive or interlace mode, etc.
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////

typedef struct ADLMode
{
/// Adapter index.
    int iAdapterIndex;
/// Display IDs.
    ADLDisplayID displayID;
/// Screen position X coordinate.
    // 屏幕位置的X坐标
    int iXPos;
/// Screen position Y coordinate.
    // 屏幕位置的Y坐标
    int iYPos;
/// Screen resolution Width.
    // 屏幕分辨率宽度
    int iXRes;
/// Screen resolution Height.
    // 屏幕分辨率高度
    int iYRes;
/// Screen Color Depth. E.g., 16, 32.
    // 屏幕颜色深度。例如，16位，32位
    int iColourDepth;
/// Screen refresh rate. Could be fractional E.g. 59.97
    // 屏幕刷新率。可以是分数，例如59.97
    float fRefreshRate;
/// Screen orientation. E.g., 0, 90, 180, 270.
    // 屏幕方向。例如，0，90，180，270
    int iOrientation;
/// Vista mode flag indicating Progressive or Interlaced mode.
    // Vista模式标志，指示渐进模式或隔行模式
    int iModeFlag;
/// The bit mask identifying the number of bits this Mode is currently using. It is the sum of all the bit definitions defined in \ref define_displaymode
    // 位掩码，标识此模式当前使用的位数。它是在\ref define_displaymode中定义的所有位定义的总和
    int iModeMask;
/// The bit mask identifying the display status. The detailed definition is in  \ref define_displaymode
    // 位掩码，标识显示状态。详细定义在\ref define_displaymode中
    int iModeValue;
} ADLMode, *LPADLMode;


/////////////////////////////////////////////////////////////////////////////////////////////
/// \brief Structure containing information about display target information.
///
/// This structure is used to store the display target information.
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
typedef struct ADLDisplayTarget
{
    /// The Display ID.
    // 显示ID
    ADLDisplayID displayID;

    /// The display map index identify this manner and the desktop surface.
    // 显示映射索引标识此方式和桌面表面
    int iDisplayMapIndex;

    /// The bit mask identifies the number of bits DisplayTarget is currently using. It is the sum of all the bit definitions defined in \ref ADL_DISPLAY_DISPLAYTARGET_PREFERRED.
    // 位掩码，标识DisplayTarget当前使用的位数。它是在\ref ADL_DISPLAY_DISPLAYTARGET_PREFERRED中定义的所有位定义的总和
    int  iDisplayTargetMask;

    /// The bit mask identifies the display status. The detailed definition is in \ref ADL_DISPLAY_DISPLAYTARGET_PREFERRED.
    // 位掩码，标识显示状态。详细定义在\ref ADL_DISPLAY_DISPLAYTARGET_PREFERRED中
    int  iDisplayTargetValue;

} ADLDisplayTarget, *LPADLDisplayTarget;


/////////////////////////////////////////////////////////////////////////////////////////////
///\brief Structure containing information about the display SLS bezel Mode information.
/// 
/// This structure is used to store the display SLS bezel Mode information.
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
typedef struct tagADLBezelTransientMode
{
    /// Adapter Index
    int iAdapterIndex;

    /// SLS Map Index
    int iSLSMapIndex;

    /// The mode index
    int iSLSModeIndex;

    /// The mode
    ADLMode displayMode;

    /// The number of bezel offsets belongs to this map
    int  iNumBezelOffset;

    /// The first bezel offset array index in the native mode array
    int  iFirstBezelOffsetArrayIndex;

    /// The bit mask identifies the bits this structure is currently using. It will be the total OR of all the bit definitions.
    int  iSLSBezelTransientModeMask;

    /// The bit mask identifies the display status. The detail definition is defined below.
    int  iSLSBezelTransientModeValue;

} ADLBezelTransientMode, *LPADLBezelTransientMode;


/////////////////////////////////////////////////////////////////////////////////////////////
/// \brief Structure containing information about the adapter display manner.
///
/// This structure is used to store adapter display manner information
/// This information can be returned to the user. Alternatively, it can be used to access various driver calls to
/// fetch various display device related display manner settings upon the user's request.
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
typedef struct ADLAdapterDisplayCap
{
    /// The Persistent logical Adapter Index.
    int iAdapterIndex;
    /// The bit mask identifies the number of bits AdapterDisplayCap is currently using. Sum all the bits defined in ADL_ADAPTER_DISPLAYCAP_XXX
    int  iAdapterDisplayCapMask;
    /// The bit mask identifies the status. Refer to ADL_ADAPTER_DISPLAYCAP_XXX
    int  iAdapterDisplayCapValue;
} ADLAdapterDisplayCap, *LPADLAdapterDisplayCap;
///\brief Structure containing information about display mapping.
///
/// This structure is used to store the display mapping data such as display manner.
/// For displays with horizontal or vertical stretch manner,
/// this structure also stores the display order, display row, and column data.
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
typedef struct ADLDisplayMap
{
    /// The current display map index. It is the OS desktop index. For example, if the OS index 1 is showing clone mode, the display map will be 1.
    int iDisplayMapIndex;

    /// The Display Mode for the current map
    ADLMode displayMode;

    /// The number of display targets belongs to this map\n
    int iNumDisplayTarget;

    /// The first target array index in the Target array\n
    int iFirstDisplayTargetArrayIndex;

    /// The bit mask identifies the number of bits DisplayMap is currently using. It is the sum of all the bit definitions defined in ADL_DISPLAY_DISPLAYMAP_MANNER_xxx.
    int  iDisplayMapMask;

    ///The bit mask identifies the display status. The detailed definition is in ADL_DISPLAY_DISPLAYMAP_MANNER_xxx.
    int  iDisplayMapValue;

} ADLDisplayMap, *LPADLDisplayMap;


/////////////////////////////////////////////////////////////////////////////////////////////
/// \brief Structure containing information about the display device possible map for one GPU
///
/// This structure is used to store the display device possible map
/// This information can be returned to the user.
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
typedef struct ADLPossibleMap
{
    /// The current PossibleMap index. Each PossibleMap is assigned an index
    int iIndex;
    /// The adapter index identifying the GPU for which to validate these Maps & Targets
    int iAdapterIndex;
    // 定义要验证的 GPU 的显示地图数量
    int iNumDisplayMap;
    // 要验证的显示地图列表
    ADLDisplayMap* displayMap;
    // 这些显示地图的显示目标数量
    int iNumDisplayTarget;
    // 要验证的这些显示地图的显示目标列表
    ADLDisplayTarget* displayTarget;
// 定义了一个结构体 ADLPossibleMap，用于存储显示可能映射的信息
typedef struct ADLPossibleMap
{
    int iDisplayIndex;                ///< 显示索引。每个显示器都被分配一个索引。
    int iDisplayControllerIndex;    ///< 显示器映射的控制器索引。
    int iDisplayMannerSupported;    ///< 支持的显示方式。
} ADLPossibleMap, *LPADLPossibleMap;

// 定义了一个结构体 ADLPossibleMapping，用于存储显示可能映射的信息
typedef struct ADLPossibleMapping
{
    int iDisplayIndex;                ///< 显示索引。每个显示器都被分配一个索引。
    int iDisplayControllerIndex;    ///< 显示器映射的控制器索引。
    int iDisplayMannerSupported;    ///< 支持的显示方式。
} ADLPossibleMapping, *LPADLPossibleMapping;

// 定义了一个结构体 ADLPossibleMapResult，用于存储验证后的显示设备可能映射的结果
typedef struct ADLPossibleMapResult
{
    /// 当前显示映射索引。这是操作系统桌面的索引。例如，操作系统索引 1 显示克隆模式。显示映射将是 1。
    int iIndex;
    // 位掩码标识当前 PossibleMapResult 使用的位数。它将是 ADL_DISPLAY_POSSIBLEMAPRESULT_VALID 中所有位定义的总和。
    int iPossibleMapResultMask;
    /// 位掩码标识可能的映射结果。详细定义在 ADL_DISPLAY_POSSIBLEMAPRESULT_XXX 中定义。
    int iPossibleMapResultValue;
} ADLPossibleMapResult, *LPADLPossibleMapResult;
///\brief Structure containing information about the display SLS Grid information.
///
/// This structure is used to store the display SLS Grid information.
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
typedef struct ADLSLSGrid
{
/// The Adapter index.
    int iAdapterIndex;

/// The grid index.
    int  iSLSGridIndex;

/// The grid row.
    int  iSLSGridRow;

/// The grid column.
    int  iSLSGridColumn;

/// The grid bit mask identifies the number of bits DisplayMap is currently using. Sum of all bits defined in ADL_DISPLAY_SLSGRID_ORIENTATION_XXX
    int  iSLSGridMask;

/// The grid bit value identifies the display status. Refer to ADL_DISPLAY_SLSGRID_ORIENTATION_XXX
    int  iSLSGridValue;

} ADLSLSGrid, *LPADLSLSGrid;

/////////////////////////////////////////////////////////////////////////////////////////////
///\brief Structure containing information about the display SLS Map information.
///
/// This structure is used to store the display SLS Map information.
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
typedef struct    ADLSLSMap
{
    /// The Adapter Index
    int iAdapterIndex;

    /// The current display map index. It is the OS Desktop index. For example, OS Index 1 showing clone mode. The Display Map will be 1.
    int iSLSMapIndex;

    /// Indicate the current grid
    ADLSLSGrid grid;

    /// OS surface index
    int  iSurfaceMapIndex;

     ///  Screen orientation. E.g., 0, 90, 180, 270
     int iOrientation;

    /// The number of display targets belongs to this map
    int  iNumSLSTarget;

    /// The first target array index in the Target array
    int  iFirstSLSTargetArrayIndex;

    /// The number of native modes belongs to this map
    int  iNumNativeMode;

    /// The first native mode array index in the native mode array
    int  iFirstNativeModeArrayIndex;
    // 拥有的边框模式数量
    int  iNumBezelMode;
    
    // 原生模式数组中的第一个边框模式数组索引
    int  iFirstBezelModeArrayIndex;
    
    // 拥有的边框偏移数量
    int  iNumBezelOffset;
    
    // 原生模式数组中的第一个边框偏移数组索引
    int  iFirstBezelOffsetArrayIndex;
    
    // 位掩码标识 DisplayMap 当前正在使用的位数。将所有在 ADL_DISPLAY_SLSMAP_XXX 中定义的位定义求和。
    int  iSLSMapMask;
    
    // 位掩码标识显示映射状态。参考 ADL_DISPLAY_SLSMAP_XXX
    int  iSLSMapValue;
// 结构体，包含显示 SLS 偏移信息
typedef struct ADLSLSOffset
{
    // 适配器索引
    int iAdapterIndex;

    // 当前显示映射索引。这是操作系统桌面索引。例如，OS 索引 1 显示克隆模式。显示映射将为 1。
    int iSLSMapIndex;

    // 显示 ID
    ADLDisplayID displayID;

    // SLS 贝塞尔模式索引
    int iBezelModeIndex;

    // SLS 贝塞尔偏移 X
    int iBezelOffsetX;

    // SLS 贝塞尔偏移 Y
    int iBezelOffsetY;

    // SLS 显示宽度
    int iDisplayWidth;

    // SLS 显示高度
    int iDisplayHeight;

    // 位掩码标识当前偏移正在使用的位数
    int iBezelOffsetMask;

    // 位掩码标识显示状态
    int iBezelffsetValue;
} ADLSLSOffset, *LPADLSLSOffset;

// 结构体，包含显示 SLS 模式信息
typedef struct ADLSLSMode
{
    // 适配器索引
    int iAdapterIndex;

    // 当前显示映射索引。这是操作系统桌面索引。例如，OS 索引 1 显示克隆模式。显示映射将为 1。
    int iSLSMapIndex;

    // 模式索引
    int iSLSModeIndex;

    // 该映射的模式
    ADLMode displayMode;

    // 位掩码标识当前模式正在使用的位数
    int iSLSNativeModeMask;
    // 位掩码用于标识显示状态
    int iSLSNativeModeValue;
// 定义了一个枚举类型 ADLSLSMode 和一个指向 ADLSLSMode 的指针 LPADLSLSMode
} ADLSLSMode, *LPADLSLSMode;

// 定义了一个结构体 ADLPossibleSLSMap，用于存储显示可能的 SLS 地图信息
typedef struct ADLPossibleSLSMap
{
    // 当前显示地图索引，即操作系统桌面索引
    // 例如，操作系统索引 1 显示克隆模式。显示地图将为 1。
    int iSLSMapIndex;

    // 要验证的显示地图数量
    int iNumSLSMap;

    // 用于验证的显示地图列表
    ADLSLSMap* lpSLSMap;

    // 要验证的显示地图配置数量
    int iNumSLSTarget;

    // 用于验证的显示目标列表
    ADLDisplayTarget* lpDisplayTarget;
} ADLPossibleSLSMap, *LPADLPossibleSLSMap;

// 定义了一个结构体 ADLSLSTarget，用于存储 SLS 目标的信息
typedef struct ADLSLSTarget
{
    // 逻辑适配器索引
    int iAdapterIndex;

    // SLS 地图索引
    int iSLSMapIndex;

    // 目标 ID
    ADLDisplayTarget displayTarget;

    // SLS 网格中的目标位置 X
    int iSLSGridPositionX;

    // SLS 网格中的目标位置 Y
    int iSLSGridPositionY;

    // 每个 SLS 目标的视图大小宽度、高度和旋转角度
    ADLMode viewSize;

    // 位掩码，用于标识 iSLSTargetValue 中当前使用的位
    int iSLSTargetMask;

    // 位掩码，用于标识状态信息。用于功能扩展目的
    int iSLSTargetValue;
} ADLSLSTarget, *LPADLSLSTarget;
# 结构体，包含有关适配器偏移步进大小的信息
# 用于存储适配器偏移步进大小信息
# 不包含子组
typedef struct ADLBezelOffsetSteppingSize
{
    # 逻辑适配器索引
    int iAdapterIndex;

    # SLS映射索引
    int iSLSMapIndex;

    # Bezel X步进大小偏移
    int iBezelOffsetSteppingSizeX;

    # Bezel Y步进大小偏移
    int iBezelOffsetSteppingSizeY;

    # 标识当前结构体正在使用的位。它将是所有位定义的总和。
    int iBezelOffsetSteppingSizeMask;

    # 位掩码标识显示状态。
    int iBezelOffsetSteppingSizeValue;

} ADLBezelOffsetSteppingSize, *LPADLBezelOffsetSteppingSize;

# 结构体，包含有关每个SLS模式的所有显示的重叠偏移信息
# 用于存储每个SLS模式的重叠模式数量，一旦用户从重叠小部件完成配置
# 不包含子组
typedef struct ADLSLSOverlappedMode
{
    # 配置了重叠的SLS模式
    ADLMode SLSMode;
    # SLS中的目标显示数量
    int iNumSLSTarget;
    # 目标数组中的第一个目标数组索引
    int iFirstTargetArrayIndex;
}ADLSLSTargetOverlap, *LPADLSLSTargetOverlap;

# 结构体，包含有关驱动程序支持的PowerExpress配置Caps的信息
/// This structure is used to store the driver supported PowerExpress Config Caps
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
typedef struct ADLPXConfigCaps
{
    /// The Persistent logical Adapter Index.
    int iAdapterIndex;

    /// The bit mask identifies the number of bits PowerExpress Config Caps is currently using. It is the sum of all the bit definitions defined in ADL_PX_CONFIGCAPS_XXXX /ref define_powerxpress_constants.
    int  iPXConfigCapMask;

    /// The bit mask identifies the PowerExpress Config Caps value. The detailed definition is in ADL_PX_CONFIGCAPS_XXXX /ref define_powerxpress_constants.
    int  iPXConfigCapValue;

} ADLPXConfigCaps, *LPADLPXConfigCaps;

/////////////////////////////////////////////////////////////////////////////////////////
///\brief Enum containing PX or HG type
///
/// This enum is used to get PX or hG type
///
/// \nosubgrouping
//////////////////////////////////////////////////////////////////////////////////////////
enum ADLPxType
{
    //Not AMD related PX/HG or not PX or HG at all
    ADL_PX_NONE = 0,
    //A+A PX
    ADL_SWITCHABLE_AMDAMD = 1,
    // A+A HG
    ADL_HG_AMDAMD = 2,
    //A+I PX
    ADL_SWITCHABLE_AMDOTHER = 3,
    //A+I HG
    ADL_HG_AMDOTHER = 4,
};


/////////////////////////////////////////////////////////////////////////////////////////////
///\brief Structure containing information about an application
///
/// This structure is used to store basic information of an application
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
typedef struct _ADLApplicationData
{
    /// Path Name
    char strPathName[ADL_MAX_PATH];
    /// File Name
    char strFileName[ADL_APP_PROFILE_FILENAME_LENGTH];
    /// Creation timestamp
    char strTimeStamp[ADL_APP_PROFILE_TIMESTAMP_LENGTH];
    /// Version
    char strVersion[ADL_APP_PROFILE_VERSION_LENGTH];
}ADLApplicationData;
// 定义结构体_ADLApplicationDataX2，用于存储应用程序的基本信息
typedef struct _ADLApplicationDataX2
{
    // 路径名
    wchar_t strPathName[ADL_MAX_PATH];
    // 文件名
    wchar_t strFileName[ADL_APP_PROFILE_FILENAME_LENGTH];
    // 创建时间戳
    wchar_t strTimeStamp[ADL_APP_PROFILE_TIMESTAMP_LENGTH];
    // 版本
    wchar_t strVersion[ADL_APP_PROFILE_VERSION_LENGTH];
}ADLApplicationDataX2;

// 定义结构体_ADLApplicationDataX3，用于存储带有进程ID的应用程序的基本信息
typedef struct _ADLApplicationDataX3
{
    // 路径名
    wchar_t strPathName[ADL_MAX_PATH];
    // 文件名
    wchar_t strFileName[ADL_APP_PROFILE_FILENAME_LENGTH];
    // 创建时间戳
    wchar_t strTimeStamp[ADL_APP_PROFILE_TIMESTAMP_LENGTH];
    // 版本
    wchar_t strVersion[ADL_APP_PROFILE_VERSION_LENGTH];
    // 应用程序进程ID
    unsigned int iProcessId;
}ADLApplicationDataX3;

// 定义结构体_PropertyRecord，用于存储应用程序配置属性的信息
typedef struct _PropertyRecord
{
    // 属性名称
    char strName [ADL_APP_PROFILE_PROPERTY_LENGTH];
    // 属性类型
    # 定义枚举类型变量 eType，用于表示属性类型
    ADLProfilePropertyType eType;
    # 定义整型变量 iDataSize，用于表示数据大小（单位：字节）
    int iDataSize;
    # 定义无符号字符型数组 uData，用于存储属性值，数组大小为1
    # 注意：这里使用数组大小为1的技巧，实际上是为了在运行时动态分配内存
    unsigned char uData[1];
// 定义了一个结构体，用于存储属性记录的数量和属性记录的缓冲区
}PropertyRecord;

/////////////////////////////////////////////////////////////////////////////////////////////
///\brief 存储应用程序配置文件信息的结构体
///
/// 该结构体用于存储应用程序配置文件的信息
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
typedef struct _ADLApplicationProfile
{
    /// 属性数量
    int iCount;
    /// 用于存储所有属性记录的缓冲区
    PropertyRecord record[1];
}ADLApplicationProfile;


/////////////////////////////////////////////////////////////////////////////////////////////
///\brief 存储OD5功率控制特性信息的结构体
///
/// 该结构体用于存储功率控制特性的信息
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
typedef struct ADLPowerControlInfo
{
    /// 最小值
    int iMinValue;
    /// 最大值
    int iMaxValue;
    /// 最小变化值
    int iStepValue;
 } ADLPowerControlInfo;

/////////////////////////////////////////////////////////////////////////////////////////////
///\brief 存储控制器模式信息的结构体
///
/// 该结构体用于存储控制器模式的信息
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
typedef struct _ADLControllerMode
{
    /// 该标志指示将由设置视口应用的操作
    /// 该值可以是ADL_CONTROLLERMODE_CM_MODIFIER_VIEW_POSITION、ADL_CONTROLLERMODE_CM_MODIFIER_VIEW_PANLOCK和ADL_CONTROLLERMODE_CM_MODIFIER_VIEW_SIZE的组合
    int iModifiers;

    /// 水平视图起始位置
    int iViewPositionCx;

    /// 垂直视图起始位置
    int iViewPositionCy;

    /// 水平左侧panlock位置
    int iViewPanLockLeft;
    # 水平右侧固定位置
    int iViewPanLockRight;
    
    # 垂直顶部固定位置
    int iViewPanLockTop;
    
    # 垂直底部固定位置
    int iViewPanLockBottom;
    
    # 视图分辨率（宽度）的像素值
    int iViewResolutionCx;
    
    # 视图分辨率（高度）的像素值
    int iViewResolutionCy;
// 定义枚举类型 ADLControllerMode，用于表示控制器模式
}ADLControllerMode;

/////////////////////////////////////////////////////////////////////////////////////////////
///\brief Structure containing information about a display
///
/// This structure is used to store information about a display
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
// 定义结构体 ADLDisplayIdentifier，用于存储显示器信息
typedef struct ADLDisplayIdentifier
{
    /// ADL display index
    long ulDisplayIndex;

    /// manufacturer ID of the display
    long ulManufacturerId;

    /// product ID of the display
    long ulProductId;

    /// serial number of the display
    long ulSerialNo;

} ADLDisplayIdentifier;

/////////////////////////////////////////////////////////////////////////////////////////////
///\brief Structure containing information about Overdrive 6 clock range
///
/// This structure is used to store information about Overdrive 6 clock range
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
// 定义结构体 ADLOD6ParameterRange，用于存储 Overdrive 6 时钟范围信息
typedef struct _ADLOD6ParameterRange
{
    /// The starting value of the clock range
    int     iMin;
    /// The ending value of the clock range
    int     iMax;
    /// The minimum increment between clock values
    int     iStep;

} ADLOD6ParameterRange;

/////////////////////////////////////////////////////////////////////////////////////////////
///\brief Structure containing information about Overdrive 6 capabilities
///
/// This structure is used to store information about Overdrive 6 capabilities
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
// 定义结构体 ADLOD6Capabilities，用于存储 Overdrive 6 的能力信息
typedef struct _ADLOD6Capabilities
{
    /// Contains a bitmap of the OD6 capability flags.  Possible values: \ref ADL_OD6_CAPABILITY_SCLK_CUSTOMIZATION,
    /// \ref ADL_OD6_CAPABILITY_MCLK_CUSTOMIZATION, \ref ADL_OD6_CAPABILITY_GPU_ACTIVITY_MONITOR
    int     iCapabilities;
    /// Contains a bitmap indicating the power states
    /// OD6支持的状态。目前只支持性能状态。可能的值：\ref ADL_OD6_SUPPORTEDSTATE_PERFORMANCE
    int iSupportedStates;
    /// 级别数量。OD6将始终使用2个级别，描述最小到最大时钟范围。
    /// 第1级表示最小时钟，第2级表示最大时钟。
    int iNumberOfPerformanceLevels;
    /// 包含sclk范围的硬限制。超频时钟不能设置在此范围之外。
    ADLOD6ParameterRange sEngineClockRange;
    /// 包含mclk范围的硬限制。超频时钟不能设置在此范围之外。
    ADLOD6ParameterRange sMemoryClockRange;
    
    /// 未来扩展的值
    int iExtValue;
    /// 未来扩展的掩码
    int iExtMask;
// 结构体，包含关于 Overdrive 6 的能力信息
} ADLOD6Capabilities;

/////////////////////////////////////////////////////////////////////////////////////////////
///\brief 结构体，包含关于 Overdrive 6 时钟数值的信息
///
/// 该结构体用于存储关于 Overdrive 6 时钟数值的信息。
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
typedef struct _ADLOD6PerformanceLevel
{
    /// 引擎（核心）时钟
    int iEngineClock;
    /// 内存时钟
    int iMemoryClock;

} ADLOD6PerformanceLevel;

/////////////////////////////////////////////////////////////////////////////////////////////
///\brief 结构体，包含关于 Overdrive 6 时钟的信息
///
/// 该结构体用于存储关于 Overdrive 6 时钟的信息。这是一个可变大小的结构体。iNumberOfPerformanceLevels 指示了数组 aLevels 包含了多少个元素。
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
typedef struct _ADLOD6StateInfo
{
    /// 等级数量。OD6 使用时钟范围而不是离散的性能等级。iNumberOfPerformanceLevels 总是 2。第一个等级表示范围内的最小时钟，第二个等级表示范围内的最大时钟。
    int     iNumberOfPerformanceLevels;

    /// 未来扩展的数值
    int     iExtValue;
    /// 未来扩展的掩码
    int     iExtMask;

    /// 可变大小的等级数组。数组中的元素数量由 iNumberofPerformanceLevels 指定。
    ADLOD6PerformanceLevel aLevels [1];

} ADLOD6StateInfo;

/////////////////////////////////////////////////////////////////////////////////////////////
///\brief 结构体，包含关于当前 Overdrive 6 性能状态的信息
///
/// 该结构体用于存储关于当前 Overdrive 6 性能状态的信息。
/// \nosubgrouping
// 定义了一个结构体，用于存储当前的显卡性能状态信息
typedef struct _ADLOD6CurrentStatus
{
    /// 当前的引擎时钟，以10 KHz为单位
    int     iEngineClock;
    /// 当前的内存时钟，以10 KHz为单位
    int     iMemoryClock;
    /// 当前 GPU 活动百分比，表示 GPU 的繁忙程度
    int     iActivityPercent;
    /// 未使用，保留以供将来使用
    int     iCurrentPerformanceLevel;
    /// 当前 PCI-E 总线速度
    int     iCurrentBusSpeed;
    /// 当前 PCI-E 总线的通道数
    int     iCurrentBusLanes;
    /// 最大可能的 PCI-E 总线通道数
    int     iMaximumBusLanes;

    /// 用于将来扩展的数值
    int     iExtValue;
    /// 用于将来扩展的掩码
    int     iExtMask;

} ADLOD6CurrentStatus;

/////////////////////////////////////////////////////////////////////////////////////////////
///\brief 结构体，包含关于 Overdrive 6 温控器能力的信息
///
/// 该结构体用于存储关于 Overdrive 6 温控器能力的信息
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
typedef struct _ADLOD6ThermalControllerCaps
{
    /// 包含温控器能力标志的位图。可能的值：\ref ADL_OD6_TCCAPS_THERMAL_CONTROLLER, \ref ADL_OD6_TCCAPS_FANSPEED_CONTROL,
    /// \ref ADL_OD6_TCCAPS_FANSPEED_PERCENT_READ, \ref ADL_OD6_TCCAPS_FANSPEED_PERCENT_WRITE, \ref ADL_OD6_TCCAPS_FANSPEED_RPM_READ, \ref ADL_OD6_TCCAPS_FANSPEED_RPM_WRITE
    int     iCapabilities;
    /// 最小风扇速度，以百分比表示
    int     iFanMinPercent;
    /// 最大风扇速度，以百分比表示
    int     iFanMaxPercent;
    /// 最小风扇速度，以每分钟转数表示
    int     iFanMinRPM;
    /// 最大风扇速度，以每分钟转数表示
    int     iFanMaxRPM;

    /// 用于将来扩展的数值
    # 定义整型变量 iExtValue，用于存储值
    int     iExtValue;
    # 定义整型变量 iExtMask，用于存储掩码值，用于未来的扩展
    int     iExtMask;
// 结构体，包含关于Overdrive 6热控制器能力的信息
typedef struct _ADLOD6ThermalControllerCaps;

/////////////////////////////////////////////////////////////////////////////////////////////
///\brief 结构体，包含关于Overdrive 6风扇速度信息的信息
///
/// 用于存储关于Overdrive 6风扇速度信息的结构体
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
typedef struct _ADLOD6FanSpeedInfo
{
    /// 包含有效风扇速度类型标志的位图。可能的值：\ref ADL_OD6_FANSPEED_TYPE_PERCENT，\ref ADL_OD6_FANSPEED_TYPE_RPM，\ref ADL_OD6_FANSPEED_USER_DEFINED
    int     iSpeedType;
    /// 当前风扇速度百分比（如果iSpeedType中存在有效标志）
    int     iFanSpeedPercent;
    /// 当前风扇速度RPM（如果iSpeedType中存在有效标志）
    int     iFanSpeedRPM;

    /// 未来扩展值
    int     iExtValue;
    /// 未来扩展掩码
    int     iExtMask;

} ADLOD6FanSpeedInfo;

/////////////////////////////////////////////////////////////////////////////////////////////
///\brief 结构体，包含关于Overdrive 6风扇速度值的信息
///
/// 用于存储关于Overdrive 6风扇速度值的结构体
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
typedef struct _ADLOD6FanSpeedValue
{
    /// 指示风扇速度的单位。可能的值：\ref ADL_OD6_FANSPEED_TYPE_PERCENT，\ref ADL_OD6_FANSPEED_TYPE_RPM
    int     iSpeedType;
    /// 风扇速度值（单位如上所示）
    int     iFanSpeed;

    /// 未来扩展值
    int     iExtValue;
    /// 未来扩展掩码
    int     iExtMask;

} ADLOD6FanSpeedValue;

/////////////////////////////////////////////////////////////////////////////////////////////
///\brief 结构体，包含关于Overdrive 6功耗控制设置的信息
///
/// This structure is used to store information about Overdrive 6 PowerControl settings.
/// PowerControl is the feature which allows the performance characteristics of the GPU
/// to be adjusted by changing the PowerTune power limits.
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
typedef struct _ADLOD6PowerControlInfo
{
    /// The minimum PowerControl adjustment value
    int     iMinValue;
    /// The maximum PowerControl adjustment value
    int     iMaxValue;
    /// The minimum difference between PowerControl adjustment values
    int     iStepValue;

    /// Value for future extension
    int     iExtValue;
    /// Mask for future extension
    int     iExtMask;

} ADLOD6PowerControlInfo;

/////////////////////////////////////////////////////////////////////////////////////////////
///\brief Structure containing information about Overdrive 6 PowerControl settings.
///
/// This structure is used to store information about Overdrive 6 PowerControl settings.
/// PowerControl is the feature which allows the performance characteristics of the GPU
/// to be adjusted by changing the PowerTune power limits.
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
typedef struct _ADLOD6VoltageControlInfo
{
    /// The minimum VoltageControl adjustment value
    int     iMinValue;
    /// The maximum VoltageControl adjustment value
    int     iMaxValue;
    /// The minimum difference between VoltageControl adjustment values
    int     iStepValue;

    /// Value for future extension
    int     iExtValue;
    /// Mask for future extension
    int     iExtMask;

} ADLOD6VoltageControlInfo;

////////////////////////////////////////////////////////////////////////////////////////////
///\brief Structure containing ECC statistics namely SEC counts and DED counts
/// Single error count - count of errors that can be corrected
/// Doubt Error Detect -  count of errors that cannot be corrected
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
// 定义了一个结构体 _ADLECCData，用于存储单错误计数和双错误检测的数据
typedef struct _ADLECCData
{
    // Single error count - count of errors that can be corrected
    int iSec;  // 存储可以被纠正的错误数量
    // Double error detect - count of errors that cannot be corrected
    int iDed;  // 存储无法被纠正的错误数量

} ADLECCData;




/// \brief Handle to ADL client context.
///
///  ADL clients obtain context handle from initial call to \ref ADL2_Main_Control_Create.
///  Clients have to pass the handle to each subsequent ADL call and finally destroy
///  the context with call to \ref ADL2_Main_Control_Destroy
/// \nosubgrouping
// 定义了一个指向 ADL 客户端上下文的句柄类型 ADL_CONTEXT_HANDLE
typedef void *ADL_CONTEXT_HANDLE;

/////////////////////////////////////////////////////////////////////////////////////////////
///\brief Structure containing the display mode definition used per controller.
///
/// This structure is used to store the display mode definition used per controller.
/// \nosubgrouping
// 定义了一个结构体 ADLDisplayModeX2，用于存储每个控制器使用的显示模式定义
typedef struct ADLDisplayModeX2
{
/// Horizontal resolution (in pixels).
   int  iWidth;  // 水平分辨率（以像素为单位）
/// Vertical resolution (in lines).
   int  iHeight;  // 垂直分辨率（以行为单位）
/// Interlaced/Progressive. The value will be set for Interlaced as ADL_DL_TIMINGFLAG_INTERLACED. If not set it is progressive. Refer define_detailed_timing_flags.
   int  iScanType;  // 插行/逐行扫描。对于插行扫描，值将设置为 ADL_DL_TIMINGFLAG_INTERLACED。如果未设置，则为逐行扫描。参考 define_detailed_timing_flags。
/// Refresh rate.
   int  iRefreshRate;  // 刷新率
/// Timing Standard. Refer define_modetiming_standard.
   int  iTimingStandard;  // 时序标准
} ADLDisplayModeX2;

/////////////////////////////////////////////////////////////////////////////////////////////
///\brief Structure containing information about Overdrive 6 extension capabilities
///
/// This structure is used to store information about Overdrive 6 extension capabilities
/// \nosubgrouping
// 定义了一个结构体 _ADLOD6CapabilitiesEx，用于存储关于 Overdrive 6 扩展功能的信息
typedef struct _ADLOD6CapabilitiesEx
{
    /// 包含 OD6 扩展功能标志的位图。可能的值：\ref ADL_OD6_CAPABILITY_SCLK_CUSTOMIZATION,
    /// \ref ADL_OD6_CAPABILITY_MCLK_CUSTOMIZATION, \ref ADL_OD6_CAPABILITY_GPU_ACTIVITY_MONITOR,
    /// \ref ADL_OD6_CAPABILITY_POWER_CONTROL, \ref ADL_OD6_CAPABILITY_VOLTAGE_CONTROL, \ref ADL_OD6_CAPABILITY_PERCENT_ADJUSTMENT,
    /// \ref ADL_OD6_CAPABILITY_THERMAL_LIMIT_UNLOCK
    int iCapabilities;
    /// 支持时钟和功率定制的电源状态。目前仅支持性能状态。
    /// 可能的值：\ref ADL_OD6_SUPPORTEDSTATE_PERFORMANCE
    int iSupportedStates;
    /// 返回 SCLK 超频调整范围的硬限制。超频时钟不应调整超出此范围。值以+/-百分比指定。
    ADLOD6ParameterRange sEngineClockPercent;
    /// 返回 MCLK 超频调整范围的硬限制。超频时钟不应调整超出此范围。值以+/-百分比指定。
    ADLOD6ParameterRange sMemoryClockPercent;
    /// 返回功率限制调整范围的硬限制。功率限制不应调整超出此范围。值以+/-百分比指定。
    ADLOD6ParameterRange sPowerControlPercent;
    /// 保留用于结构的未来扩展。
    int iExtValue;
    /// 保留用于结构的未来扩展。
    int iExtMask;
// 结构体，包含有关Overdrive 6扩展状态信息的信息
typedef struct _ADLOD6StateEx
{
    // 当前引擎时钟调整值，以+/-百分比表示
    int iEngineClockPercent;
    // 当前内存时钟调整值，以+/-百分比表示
    int iMemoryClockPercent;
    // 当前功率控制调整值，以+/-百分比表示
    int iPowerControlPercent;
    // 保留用于结构的未来扩展
    int iExtValue;
    // 保留用于结构的未来扩展
    int iExtMask;
} ADLOD6StateEx;

// 结构体，包含有关Overdrive 6扩展推荐的最大时钟调整值的信息
typedef struct _ADLOD6MaxClockAdjust
{
    // 推荐的最大引擎时钟调整百分比，针对指定的功率限制值
    int iEngineClockMax;
    // 推荐的最大内存时钟调整百分比，针对指定的功率限制值
    // 目前内存独立于功率限制设置，因此iMemoryClockMax将始终返回最大可能的调整值。此字段用于未来增强，以防我们在内存时钟调整和功率限制设置之间添加依赖关系。
    int iMemoryClockMax;
    # 保留结构的未来扩展
    int iExtValue;
    # 保留结构的未来扩展
    int iExtMask;
// 结构体，包含连接器信息
typedef struct ADLConnectorInfo
{
    // 连接器的索引（从0开始）
    int iConnectorIndex;
    // 用于显示标识/排序的连接器ID
    int iConnectorId;
    // 插槽的索引，从0开始
    int iSlotIndex;
    // 连接器类型，参考连接器类型定义
    int iType;
    // 连接器的位置（以毫米为单位），从插槽的右侧开始计算
    int iOffset;
    // 连接器的长度（以毫米为单位）
    int iLength;
} ADLConnectorInfo;

// 结构体，包含插槽信息
typedef struct ADLBracketSlotInfo
{
    // 插槽的索引，从0开始
    int iSlotIndex;
    // 插槽的长度（以毫米为单位）
    int iLength;
    // 插槽的宽度（以毫米为单位）
    int iWidth;
} ADLBracketSlotInfo;

// 结构体，包含MST分支信息
typedef struct ADLMSTRad
{
    // 链路的深度
    int iLinkNumber;
    // 相对地址，地址方案从源端开始
    char rad[ADL_MAX_RAD_LINK_COUNT];
} ADLMSTRad;
// 结构体，包含显示器或 MST 分支信息
typedef struct ADLDevicePort
{
    // 连接器的索引
    int iConnectorIndex;
    // 相对 MST 地址。如果 MST RAD 包含 0，则表示 DP 或 MST 拓扑的根。对于非 DP 连接器，MST RAD 被忽略。
    ADLMSTRad aMSTRad;
} ADLDevicePort;

// 结构体，包含支持的连接类型和属性
typedef struct ADLSupportedConnections
{
    // 支持的连接类型的位向量。位掩码在常量部分定义。 \ref define_connection_types
    int iSupportedConnections;
    // 位向量数组。每个位向量表示一个连接类型的支持属性。该数组的索引是连接类型（掩码中的位数）。
    int iSupportedProperties[ADL_MAX_CONNECTION_TYPES];
} ADLSupportedConnections;

// 结构体，包含连接器的连接状态
typedef struct ADLConnectionState
{
    // 该值是位向量。每个位表示状态。有关详细信息，请参阅掩码常量。 \ref define_emulation_status
    int iEmulationStatus;
    // 当前仿真模式的信息，参考常量以获取详细信息。参见 define_emulation_mode
    int iEmulationMode;
    // 如果连接处于活动状态，则包含显示ID，否则为 CWDDEDI_INVALID_DISPLAY_INDEX
    int iDisplayIndex;
// 枚举类型，表示连接状态
} ADLConnectionState;

////////////////////////////////////////////////////////////////////////////////////////////
///\brief 包含连接属性信息的结构
///
/// 该结构用于检索连接类型的属性
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
typedef struct ADLConnectionProperties
{
    // 位向量。表示实际属性。特定连接类型支持的属性。 \ref define_connection_properties
    int iValidProperties;
    // 比特率（以 MHz 为单位）。可用于 MST 分支、DP 或 DP 主动转接器。 \ref define_linkrate_constants
    int iBitrate;
    // DP 连接中的通道数。 \ref define_lanecount_constants
    int iNumberOfLanes;
    // 颜色深度（以位为单位）。 \ref define_colordepth_constants
    int iColorDepth;
    // 3D 功能。可用于某些转接器。例如：交替帧包。该属性的值是位向量。
    int iStereo3DCaps;
    /// 输出带宽。可用于 MST 分支、DP 或 DP 主动转接器。 \ref define_linkrate_constants
    int iOutputBandwidth;
} ADLConnectionProperties;

////////////////////////////////////////////////////////////////////////////////////////////
///\brief 包含连接信息的结构
///
/// 该结构用于从驱动程序中检索数据，其中包括
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
typedef struct ADLConnectionData
{
    /// 连接类型。根据连接类型，iNumberofPorts 或 IDataSize、EDIDdata 有效。 \ref define_connection_types
    int iConnectionType;
    /// 指定连接属性。
    ADLConnectionProperties aConnectionProperties;
    /// 端口数
    int iNumberofPorts;
    /// 活动连接数
    int iActiveConnections;
    /// EDID 数据块的实际大小。
    int iDataSize;
    // 定义一个字符数组用于存储EDID数据，数组大小为ADL_MAX_DISPLAY_EDID_DATA_SIZE
    char EdidData[ADL_MAX_DISPLAY_EDID_DATA_SIZE];
// 结构体，包含有关控制器模式的信息，包括连接器数量
// 用于存储控制器模式的信息
// 不包含子组
typedef struct ADLAdapterCapsX2
{
    // 适配器的适配器ID
    int iAdapterID;
    // 此适配器的控制器数量
    int iNumControllers;
    // 此适配器的显示器数量
    int iNumDisplays;
    // 此适配器的叠加层数量
    int iNumOverlays;
    // GLSync连接器的数量
    int iNumOfGLSyncConnectors;
    // 位掩码标识适配器功能
    int iCapsMask;
    // 位标识适配器功能
    int iCapsValue;
    // 此适配器的连接器数量
    int iNumConnectors;
}ADLAdapterCapsX2;

// 错误记录的严重性枚举
typedef enum _ADL_ERROR_RECORD_SEVERITY
{
    ADL_GLOBALLY_UNCORRECTED  = 1,  // 全局未校正
    ADL_LOCALLY_UNCORRECTED   = 2,  // 本地未校正
    ADL_DEFFERRED             = 3,  // 延迟
    ADL_CORRECTED             = 4   // 已校正
}ADL_ERROR_RECORD_SEVERITY;

// ECC_EDC标志的联合体
typedef union _ADL_ECC_EDC_FLAG
{
    struct
    {
        unsigned int isEccAccessing        : 1;  // ECC访问
        unsigned int reserved              : 31;  // 保留
    }bits;
    unsigned int u32All;
}ADL_ECC_EDC_FLAG;

// 结构体，包含有关EDC错误记录的信息
// 用于存储EDC错误记录
// 不包含子组
typedef struct ADLErrorRecord
{
    // 错误的严重性
    ADL_ERROR_RECORD_SEVERITY Severity;

    // 计数器是否有效？
    int  countValid;

    // 计数器值，如果有效
    unsigned int count;

    // 位置信息是否有效？
    # 定义一个整型变量，用于表示位置是否有效
    int locationValid;

    # 错误发生的物理位置
    unsigned int CU; // 如果已知，表示发生错误的 CU 编号
    char StructureName[32]; // 例如 LDS、TCC 等

    # 错误记录创建的时间（例如查询时间或中断时间）
    char tiestamp[32];

    # 用于填充的无符号整型数组，长度为3
    unsigned int padding[3];
// 定义了一个结构体 ADLErrorRecord，用于记录 ADL 错误信息
}ADLErrorRecord;

// 定义了一个枚举类型 ADL_EDC_BLOCK_ID，用于表示 EDC 块的 ID
typedef enum _ADL_EDC_BLOCK_ID
{
    ADL_EDC_BLOCK_ID_SQCIS = 1,
    ADL_EDC_BLOCK_ID_SQCDS = 2,
    ADL_EDC_BLOCK_ID_SGPR  = 3,
    ADL_EDC_BLOCK_ID_VGPR  = 4,
    ADL_EDC_BLOCK_ID_LDS   = 5,
    ADL_EDC_BLOCK_ID_GDS   = 6,
    ADL_EDC_BLOCK_ID_TCL1  = 7,
    ADL_EDC_BLOCK_ID_TCL2  = 8
}ADL_EDC_BLOCK_ID;

// 定义了一个枚举类型 ADL_ERROR_INJECTION_MODE，用于表示错误注入模式
typedef enum _ADL_ERROR_INJECTION_MODE
{
    ADL_ERROR_INJECTION_MODE_SINGLE      = 1,
    ADL_ERROR_INJECTION_MODE_MULTIPLE    = 2,
    ADL_ERROR_INJECTION_MODE_ADDRESS     = 3
}ADL_ERROR_INJECTION_MODE;

// 定义了一个联合类型 ADL_ERROR_PATTERN，用于表示错误模式
typedef union _ADL_ERROR_PATTERN
{
    // 定义了一个结构体 bits，包含了不同的错误模式的位域
    struct
    {
        unsigned long  EccInjVector         :  16;
        unsigned long  EccInjEn             :  9;
        unsigned long  EccBeatEn            :  4;
        unsigned long  EccChEn              :  4;
        unsigned long  reserved             :  31;
    } bits;
    // 定义了一个 64 位的整数类型 u64Value，用于存储错误模式的值
    unsigned long long u64Value;
} ADL_ERROR_PATTERN;

// 定义了一个结构体 ADL_ERROR_INJECTION_DATA，用于存储错误注入的数据
typedef struct _ADL_ERROR_INJECTION_DATA
{
    // 存储错误地址
    unsigned long long errorAddress;
    // 存储错误模式
    ADL_ERROR_PATTERN errorPattern;
}ADL_ERROR_INJECTION_DATA;

// 定义了一个结构体 ADLErrorInjection，用于存储 EDC 错误注入的信息
typedef struct ADLErrorInjection
{
    // 存储 EDC 块的 ID
    ADL_EDC_BLOCK_ID blockId;
    // 存储错误注入模式
    ADL_ERROR_INJECTION_MODE errorInjectionMode;
}ADLErrorInjection;

// 定义了一个结构体 ADLErrorInjectionX2，用于存储 EDC 错误注入的详细信息
typedef struct ADLErrorInjectionX2
{
    // 存储 EDC 块的 ID
    ADL_EDC_BLOCK_ID blockId;
    // 存储错误注入模式
    ADL_ERROR_INJECTION_MODE errorInjectionMode;
    // 存储错误注入的数据
    ADL_ERROR_INJECTION_DATA errorInjectionData;
}ADLErrorInjectionX2;

// 结构体 ADLFreeSyncCap，用于存储显示器的 FreeSync 能力信息
/// the GPU the display is connected to.
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
// 定义了一个结构体 ADLFreeSyncCap，用于存储 FreeSync 的能力信息
typedef struct ADLFreeSyncCap
{
    /// FreeSync capability flags. \ref define_freesync_caps
    int iCaps;  // FreeSync 能力标志
    /// Reports minimum FreeSync refresh rate supported by the display in micro hertz
    int iMinRefreshRateInMicroHz;  // 报告显示器支持的最小 FreeSync 刷新率（单位：微赫兹）
    /// Reports maximum FreeSync refresh rate supported by the display in micro hertz
    int iMaxRefreshRateInMicroHz;  // 报告显示器支持的最大 FreeSync 刷新率（单位：微赫兹）
    /// Reserved
    int iReserved[5];  // 保留字段
} ADLFreeSyncCap;

/////////////////////////////////////////////////////////////////////////////////////////////
///\brief Structure containing per display Display Connectivty Experience Settings
///
/// This structure is used to store the Display Connectivity Experience settings of a
/// display
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
// 定义了一个结构体 _ADLDceSettings，用于存储每个显示器的显示连接体验设置
typedef struct _ADLDceSettings
{
    DceSettingsType type;  // 定义了联合体中的结构类型
    union
    {
        struct
        {
            bool qualityDetectionEnabled;  // HDMI 质量检测是否启用
        } HdmiLq;
        struct
        {
            DpLinkRate linkRate;  // 只读
            unsigned int numberOfActiveLanes;  // 只读
            unsigned int numberofTotalLanes;  // 只读
            int relativePreEmphasis;  // 允许的值为 -2 到 +2
            int relativeVoltageSwing;  // 允许的值为 -2 到 +2
            int persistFlag;
        } DpLink;
        struct
        {
            bool linkProtectionEnabled;  // 只读
        } Protection;
    } Settings;
    int iReserved[15];  // 保留字段
} ADLDceSettings;

/////////////////////////////////////////////////////////////////////////////////////////////
///\brief Structure containing information about Graphic Core
///
/// This structure is used to get Graphic Core Info
/// \nosubgrouping
// 定义了一个结构体 ADLGraphicCoreInfo，用于存储图形核心信息
typedef struct ADLGraphicCoreInfo
{
    /// 表示图形核心的代数
    int iGCGen;

    union
    {
        /// CUs 的总数。对于 GCN 架构有效 (iGCGen == GCN)
        int iNumCUs;
        /// WGPs 的总数。对于 RDNA 架构有效 (iGCGen == RDNA)
        int iNumWGPs;
    };

    union
    {
        /// 每个 CU 的处理单元数。对于 GCN 架构有效 (iGCGen == GCN)
        int iNumPEsPerCU;
        /// 每个 WGP 的处理单元数。对于 RDNA 架构有效 (iGCGen == RDNA)
        int iNumPEsPerWGP;
    };

    /// SIMD 的总数。对于 Pre GCN 架构有效 (iGCGen == Pre-GCN)
    int iNumSIMDs;

    /// ROP 的总数。对于 GCN 和 Pre GCN 架构都有效
    int iNumROPs;

    /// 保留字段，用于将来使用
    int iReserved[11];
}ADLGraphicCoreInfo;

// 定义了一个结构体 ADLODNParameterRange，用于存储 Overdrive N 时钟范围的信息
typedef struct _ADLODNParameterRange
{
    /// 时钟范围的起始值
    int     iMode;
    /// 时钟范围的最小值
    int     iMin;
    /// 时钟范围的最大值
    int     iMax;
    /// 时钟值之间的最小增量
    int     iStep;
    /// 默认的时钟值
    int     iDefault;

} ADLODNParameterRange;

// 定义了一个结构体，用于存储 Overdrive N 的能力信息
typedef struct _ADLODNCapabilities
{
    // 省略部分结构体成员...
} ADLODNCapabilities;
# 定义了一个结构体，用于存储 Overdrive N 的能力信息
typedef struct _ADLODNCapabilities
{
    /// 描述最小到最大时钟范围的级别数量
    /// 第一级表示最小时钟，第二级表示最大时钟
    int     iMaximumNumberOfPerformanceLevels;
    /// 包含 sclk 范围的硬限制。超频时钟不能设置在此范围之外。
    ADLODNParameterRange     sEngineClockRange;
    /// 包含 mclk 范围的硬限制。超频时钟不能设置在此范围之外。
    ADLODNParameterRange     sMemoryClockRange;
    /// 包含 vddc 范围的硬限制。超频时钟不能设置在此范围之外。
    ADLODNParameterRange     svddcRange;
    /// 包含功率范围的硬限制。超频时钟不能设置在此范围之外。
    ADLODNParameterRange     power;
    /// 包含功率范围的硬限制。超频时钟不能设置在此范围之外。
    ADLODNParameterRange     powerTuneTemperature;
    /// 包含温度范围的硬限制。超频时钟不能设置在此范围之外。
    ADLODNParameterRange     fanTemperature;
    /// 包含风扇范围的硬限制。超频时钟不能设置在此范围之外。
    ADLODNParameterRange     fanSpeed;
    /// 包含风扇范围的硬限制。超频时钟不能设置在此范围之外。
    ADLODNParameterRange     minimumPerformanceClock;
} ADLODNCapabilities;

/////////////////////////////////////////////////////////////////////////////////////////////
///\brief 包含关于 Overdrive N 能力的信息的结构
///
/// 该结构用于存储关于 Overdrive N 能力的信息
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
typedef struct _ADLODNCapabilitiesX2
    // 描述最小到最大时钟范围的级别数量
    // 第一级表示最小时钟，第二级表示最大时钟
    int     iMaximumNumberOfPerformanceLevels;
    // 位向量，指示支持的特性
    // 参考：ADLODNFEATURECONTROL
    int iFlags;
    // 包含 sclk 范围的硬限制。超频时钟不能设置在此范围之外
    ADLODNParameterRange     sEngineClockRange;
    // 包含 mclk 范围的硬限制。超频时钟不能设置在此范围之外
    ADLODNParameterRange     sMemoryClockRange;
    // 包含 vddc 范围的硬限制。超频时钟不能设置在此范围之外
    ADLODNParameterRange     svddcRange;
    // 包含功率范围的硬限制。超频时钟不能设置在此范围之外
    ADLODNParameterRange     power;
    // 包含功率范围的硬限制。超频时钟不能设置在此范围之外
    ADLODNParameterRange     powerTuneTemperature;
    // 包含温度范围的硬限制。超频时钟不能设置在此范围之外
    ADLODNParameterRange     fanTemperature;
    // 包含风扇范围的硬限制。超频时钟不能设置在此范围之外
    ADLODNParameterRange     fanSpeed;
    // 包含最小性能时钟的硬限制
    ADLODNParameterRange     minimumPerformanceClock;
    // 包含节流通知的硬限制
    ADLODNParameterRange throttleNotificaion;
    // 包含自动系统时钟的硬限制
    ADLODNParameterRange autoSystemClock;
// 结构体，包含有关 Overdrive 级别的信息
typedef struct ADLODNCapabilitiesX2;

/////////////////////////////////////////////////////////////////////////////////////////////
///\brief 结构体，包含有关 Overdrive 级别的信息。
///
/// 该结构用于存储有关 Overdrive 级别的信息。
/// 该结构由 ADLODPerformanceLevels 使用。
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
typedef struct ADLODNPerformanceLevel
{
    /// 时钟频率
    int iClock;
    /// 电压
    int iVddc;
    /// 是否启用
    int iEnabled;
} ADLODNPerformanceLevel;

/////////////////////////////////////////////////////////////////////////////////////////////
///\brief 结构体，包含有关 Overdrive N 性能级别的信息。
///
/// 该结构用于存储有关 Overdrive 性能级别的信息。
/// 该结构由 ADL_OverdriveN_ODPerformanceLevels_Get() 和 ADL_OverdriveN_ODPerformanceLevels_Set() 函数使用。
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
typedef struct ADLODNPerformanceLevels
{
    int iSize;
    // 自动/手动模式
    int iMode;
    /// 必须设置为 sizeof( \ref ADLODPerformanceLevels ) + sizeof( \ref ADLODPerformanceLevel ) * (ADLODParameters.iNumberOfPerformanceLevels - 1)
    int iNumberOfPerformanceLevels;
    /// 性能状态描述符数组。必须有 ADLODParameters.iNumberOfPerformanceLevels 个元素。
    ADLODNPerformanceLevel aLevels[1];
} ADLODNPerformanceLevels;

/////////////////////////////////////////////////////////////////////////////////////////////
///\brief 结构体，包含有关 Overdrive N 风扇速度的信息。
///
/// 该结构用于存储有关 Overdrive 风扇控制的信息。
/// 该结构由 ADL_OverdriveN_ODPerformanceLevels_Get() 和 ADL_OverdriveN_ODPerformanceLevels_Set() 函数使用。
/// \nosubgrouping
// 定义了一个结构体 ADLODNFanControl，用于存储风扇控制相关信息
typedef struct ADLODNFanControl
{
    int iMode;  // 模式
    int iFanControlMode;  // 风扇控制模式
    int iCurrentFanSpeedMode;  // 当前风扇速度模式
    int iCurrentFanSpeed;  // 当前风扇速度
    int iTargetFanSpeed;  // 目标风扇速度
    int iTargetTemperature;  // 目标温度
    int iMinPerformanceClock;  // 最小性能时钟
    int iMinFanLimit;  // 最小风扇限制
} ADLODNFanControl;

// 定义了一个结构体 ADLODNPowerLimitSetting，用于存储功率限制相关信息
typedef struct ADLODNPowerLimitSetting
{
    int iMode;  // 模式
    int iTDPLimit;  // TDP 限制
    int iMaxOperatingTemperature;  // 最大操作温度
} ADLODNPowerLimitSetting;

// 定义了一个结构体 ADLODNPerformanceStatus，用于存储性能状态相关信息
typedef struct ADLODNPerformanceStatus
{
    int iCoreClock;  // 核心时钟
    int iMemoryClock;  // 内存时钟
    int iDCEFClock;  // DCEF 时钟
    int iGFXClock;  // GFX 时钟
    int iUVDClock;  // UVD 时钟
    int iVCEClock;  // VCE 时钟
    int iGPUActivityPercent;  // GPU 活动百分比
    int iCurrentCorePerformanceLevel;  // 当前核心性能级别
    int iCurrentMemoryPerformanceLevel;  // 当前内存性能级别
    int iCurrentDCEFPerformanceLevel;  // 当前 DCEF 性能级别
    int iCurrentGFXPerformanceLevel;  // 当前 GFX 性能级别
    int iUVDPerformanceLevel;  // UVD 性能级别
    int iVCEPerformanceLevel;  // VCE 性能级别
    int iCurrentBusSpeed;  // 当前总线速度
    int iCurrentBusLanes;  // 当前总线通道数
    int iMaximumBusLanes;  // 最大总线通道数
    int iVDDC;  // VDDC
    int iVDDCI;  // VDDCI
} ADLODNPerformanceStatus;

// 定义了一个结构体 ADLODNPerformanceLevelX2，用于存储性能级别相关信息
typedef struct ADLODNPerformanceLevelX2
{
    int iClock;  // 时钟
    int iVddc;  // VDCC
    int iEnabled;  // 是否启用
    int iControl;  // 控制
// 结构体，包含有关 Overdrive N 性能级别的信息
typedef struct ADLODNPerformanceLevelsX2
{
    int iSize; // 结构体大小
    int iMode; // 自动/手动模式
    int iNumberOfPerformanceLevels; // 性能级别数量
    ADLODNPerformanceLevelX2 aLevels[1]; // 性能状态描述数组
} ADLODNPerformanceLevelsX2;

// Overdrive N 当前功率类型枚举
typedef enum _ADLODNCurrentPowerType
{
    ODN_GPU_TOTAL_POWER = 0, // GPU 总功率
    ODN_GPU_PPT_POWER, // GPU PPT 功率
    ODN_GPU_SOCKET_POWER, // GPU 插座功率
    ODN_GPU_CHIP_POWER // GPU 芯片功率
} ADLODNCurrentPowerType;

// 输入/输出：CWDDEPM_CURRENTPOWERPARAMETERS
// 结构体，包含有关 Overdrive N 当前功率参数的信息
typedef struct _ADLODNCurrentPowerParameters
{
    int size; // 结构体大小
    ADLODNCurrentPowerType powerType; // 当前功率类型
    int currentPower; // 当前功率值
} ADLODNCurrentPowerParameters;

// Overdrive N 扩展范围数据结构
typedef struct _ADLODNExtSingleInitSetting
{
    int mode; // 模式
    int minValue; // 最小值
    int maxValue; // 最大值
    int step; // 步长
    int defaultValue; // 默认值
} ADLODNExtSingleInitSetting;

// Overdrive 8 扩展范围数据结构
typedef struct _ADLOD8SingleInitSetting
{
    int featureID; // 特征 ID
    int minValue; // 最小值
    int maxValue; // 最大值
    int defaultValue; // 默认值
} ADLOD8SingleInitSetting;

// 结构体，包含有关 Overdrive8 初始设置的信息
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
// 定义了一个结构体 ADLOD8InitSetting，包含了初始化设置的数量和 Overdrive8 的能力，以及 OD8_COUNT 个 OD8 单个初始化设置
typedef struct _ADLOD8InitSetting
{
    int count;
    int overdrive8Capabilities;
    ADLOD8SingleInitSetting  od8SettingTable[OD8_COUNT];
} ADLOD8InitSetting;

/////////////////////////////////////////////////////////////////////////////////////////////
///\brief Structure containing information about Overdrive8 current setting
///
/// 该结构体用于存储有关 Overdrive8 当前设置的信息
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
// 定义了一个结构体 ADLOD8CurrentSetting，包含了当前设置的数量和 OD8_COUNT 个 Od8SettingTable
typedef struct _ADLOD8CurrentSetting
{
    int count;
    int Od8SettingTable[OD8_COUNT];
} ADLOD8CurrentSetting;


/////////////////////////////////////////////////////////////////////////////////////////////
///\brief Structure containing information about Overdrive8 set setting
///
/// 该结构体用于存储有关 Overdrive8 设置设置的信息
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
// 定义了一个结构体 ADLOD8SingleSetSetting，包含了一个值、一个请求标志和一个重置标志
typedef struct _ADLOD8SingleSetSetting
{
    int value;
    int requested;      // 0 - default , 1 - requested
    int reset;          // 0 - do not reset , 1 - reset setting back to default
} ADLOD8SingleSetSetting;

// 定义了一个结构体 ADLOD8SetSetting，包含了设置的数量和 OD8_COUNT 个 OD8 单个设置
typedef struct _ADLOD8SetSetting
{
    int count;
    ADLOD8SingleSetSetting  od8SettingTable[OD8_COUNT];
} ADLOD8SetSetting;

/////////////////////////////////////////////////////////////////////////////////////////////
///\brief Structure containing information about Performance Metrics data
///
/// 该结构体用于存储有关性能指标数据的信息
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
// 定义了一个结构体 ADLSingleSensorData，包含了一个支持标志和一个值
typedef struct _ADLSingleSensorData
{
    int supported;
    int  value;
} ADLSingleSensorData;

// 定义了一个结构体 ADLPMLogDataOutput，包含了一个大小
typedef struct _ADLPMLogDataOutput
{
    int size;
    # 定义一个名为sensors的ADLSingleSensorData类型的数组，数组大小为ADL_PMLOG_MAX_SENSORS
    ADLSingleSensorData sensors[ADL_PMLOG_MAX_SENSORS];
// 结构体，用于存储 ADLPPLogSettings 结构的信息
typedef struct ADLPPLogSettings
{
    // 断言时是否中断
    int BreakOnAssert;
    // 警告时是否中断
    int BreakOnWarn;
    // 日志是否启用
    int LogEnabled;
    // 日志字段掩码
    int LogFieldMask;
    // 日志目的地
    int LogDestinations;
    // 日志严重性是否启用
    int LogSeverityEnabled;
    // 日志来源掩码
    int LogSourceMask;
    // 电源分析是否启用
    int PowerProfilingEnabled;
    // 电源分析时间间隔
    int PowerProfilingTimeInterval;
}ADLPPLogSettings;

// 结构体，用于存储与交流和直流每秒帧数相关的信息
typedef struct _ADLFPSSettingsOutput
{
    // 结构体大小
    int ulSize;
    // 如果为1，则交流状态下启用 FPS 监视器
    int bACFPSEnabled;
    // 如果为1，则直流状态下启用 FPS 监视器
    int bDCFPSEnabled;
    // 交流状态下 FPS 监视器的当前值
    int ulACFPSCurrent;
    // 直流状态下 FPS 监视器的当前值
    int ulDCFPSCurrent;
    // 交流状态下 PPLib 允许的最大 FPS 阈值
    int ulACFPSMaximum;
    // 交流状态下 PPLib 允许的最小 FPS 阈值
    int ulACFPSMinimum;
    // 直流状态下 PPLib 允许的最大 FPS 阈值
    int ulDCFPSMaximum;
    // 直流状态下 PPLib 允许的最小 FPS 阈值
    int ulDCFPSMinimum;
} ADLFPSSettingsOutput;
///\brief Structure containing information related Frames Per Second for AC and DC.
///
/// This structure is used to store information related AC and DC Frames Per Second settings
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
typedef struct _ADLFPSSettingsInput
{
    /// size
    int ulSize;  ///< 用于存储结构体的大小
    /// Settings are for Global FPS (used by CCC)
    int bGlobalSettings;  ///< 用于存储全局 FPS 设置
    /// Current Value of FPS Monitor in AC state
    int ulACFPSCurrent;  ///< 用于存储交流状态下 FPS 监视器的当前值
    /// Current Value of FPS Monitor in DC state
    int ulDCFPSCurrent;  ///< 用于存储直流状态下 FPS 监视器的当前值
    /// Reserved
    int ulReserved[6];  ///< 保留字段，用于填充结构体

} ADLFPSSettingsInput;

/////////////////////////////////////////////////////////////////////////////////////////////
///\brief Structure containing information related power management logging.
///
/// This structure is used to store support information for power management logging.
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
enum { ADL_PMLOG_MAX_SUPPORTED_SENSORS = 256 };

typedef struct _ADLPMLogSupportInfo
{
    /// list of sensors defined by ADL_PMLOG_SENSORS
    unsigned short usSensors[ADL_PMLOG_MAX_SUPPORTED_SENSORS];  ///< 由 ADL_PMLOG_SENSORS 定义的传感器列表
    /// Reserved
    int ulReserved[16];  ///< 保留字段，用于填充结构体

} ADLPMLogSupportInfo;


/////////////////////////////////////////////////////////////////////////////////////////////
///\brief Structure containing information to start power management logging.
///
/// This structure is used as input to ADL2_Adapter_PMLog_Start
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
typedef struct _ADLPMLogStartInput
{
    /// list of sensors defined by ADL_PMLOG_SENSORS
    unsigned short usSensors[ADL_PMLOG_MAX_SUPPORTED_SENSORS];  ///< 由 ADL_PMLOG_SENSORS 定义的传感器列表
    /// Sample rate in milliseconds
    unsigned long ulSampleRate;  ///< 以毫秒为单位的采样率
    /// Reserved
    int ulReserved[15];  ///< 保留字段，用于填充结构体

} ADLPMLogStartInput;

typedef struct _ADLPMLogData
{
    /// Structure version
    # 无符号整数类型的变量，用于存储版本号
    unsigned int ulVersion;
    # 当前驱动程序的采样率
    unsigned int ulActiveSampleRate;
    # 上次更新的时间戳
    unsigned long long ulLastUpdated;
    # 存储传感器和数值的二维数组
    unsigned int ulValues[ADL_PMLOG_MAX_SUPPORTED_SENSORS][2];
    # 保留字段，用于存储预留的数据
    unsigned int ulReserved[256];
// 结构体，包含用于开始电源管理日志记录的信息
typedef struct _ADLPMLogData {
    // 指向包含日志数据的内存地址的指针
    union {
        void* pLoggingAddress;
        unsigned long long ptr_LoggingAddress;
    };
    // 保留字段
    int ulReserved[14];
} ADLPMLogData;

// 结构体，包含用于开始电源管理日志记录的输出信息
typedef struct _ADLPMLogStartOutput {
    // 指向包含日志数据的内存地址的指针
    union {
        void* pLoggingAddress;
        unsigned long long ptr_LoggingAddress;
    };
    // 保留字段
    int ulReserved[14];
} ADLPMLogStartOutput;

// 结构体，包含与 RAS 获取错误计数信息相关的信息
typedef struct _ADLRASGetErrorCountsInput {
    // 保留字段
    unsigned int Reserved[16];
} ADLRASGetErrorCountsInput;

// 结构体，包含与 RAS 获取错误计数信息相关的输出信息
typedef struct _ADLRASGetErrorCountsOutput {
    // 纠正的错误计数，包括 DRAM 和 SRAM ECC
    unsigned int CorrectedErrors;
    // 未纠正的错误计数，包括 DRAM 和 SRAM ECC
    unsigned int UnCorrectedErrors;
    // 保留字段
    unsigned int Reserved[14];
} ADLRASGetErrorCountsOutput;
/// This structure is used to store RAS Error Counts Get Information
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
// 定义用于存储 RAS 错误计数获取信息的结构体

typedef struct _ADLRASGetErrorCounts
{
    unsigned int                InputSize;
    ADLRASGetErrorCountsInput   Input;
    unsigned int                OutputSize;
    ADLRASGetErrorCountsOutput  Output;
} ADLRASGetErrorCounts;


/////////////////////////////////////////////////////////////////////////////////////////////
///\brief Structure containing information related RAS Error Counts Reset Information
///
/// This structure is used to store RAS Error Counts Reset Input Information
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
// 定义包含与 RAS 错误计数重置信息相关的信息的结构体
// 用于存储 RAS 错误计数重置输入信息的结构体

typedef struct _ADLRASResetErrorCountsInput
{
    unsigned int                Reserved[8];
} ADLRASResetErrorCountsInput;

/////////////////////////////////////////////////////////////////////////////////////////////
///\brief Structure containing information related RAS Error Counts Reset Information
///
/// This structure is used to store RAS Error Counts Reset Output Information
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
// 定义包含与 RAS 错误计数重置信息相关的信息的结构体
// 用于存储 RAS 错误计数重置输出信息的结构体

typedef struct _ADLRASResetErrorCountsOutput
{
    unsigned int                Reserved[8];
} ADLRASResetErrorCountsOutput;

/////////////////////////////////////////////////////////////////////////////////////////////
///\brief Structure containing information related RAS Error Counts Reset Information
///
/// This structure is used to store RAS Error Counts Reset Information
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
// 定义包含与 RAS 错误计数重置信息相关的信息的结构体
// 用于存储 RAS 错误计数重置信息的结构体

typedef struct _ADLRASResetErrorCounts
{
    unsigned int                    InputSize;
    ADLRASResetErrorCountsInput     Input;
    unsigned int                    OutputSize;
    # 声明一个名为ADLRASResetErrorCountsOutput的变量，类型为Output
// 结构体，用于存储 RAS 错误注入输入信息
typedef struct _ADLRASErrorInjectonInput
{
    unsigned long long Address; // 存储地址信息
    ADL_RAS_INJECTION_METHOD Value; // 存储注入方法
    ADL_RAS_BLOCK_ID BlockId; // 存储块 ID
    ADL_RAS_ERROR_TYPE InjectErrorType; // 存储错误类型
    ADL_MEM_SUB_BLOCK_ID SubBlockIndex; // 存储子块索引
    unsigned int padding[9]; // 填充数组
} ADLRASErrorInjectonInput;

// 结构体，用于存储 RAS 错误注入输出信息
typedef struct _ADLRASErrorInjectionOutput
{
    unsigned int ErrorInjectionStatus; // 存储错误注入状态
    unsigned int padding[15]; // 填充数组
} ADLRASErrorInjectionOutput;

// 结构体，用于存储 RAS 错误注入信息
typedef struct _ADLRASErrorInjection
{
    unsigned int InputSize; // 存储输入大小
    ADLRASErrorInjectonInput Input; // 存储输入信息
    unsigned int OutputSize; // 存储输出大小
    ADLRASErrorInjectionOutput Output; // 存储输出信息
} ADLRASErrorInjection;
/// This structure is used to store basic information of a recently ran or currently running application
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
// 定义存储最近运行或当前运行应用程序的基本信息的结构体
typedef struct _ADLSGApplicationInfo
{
    /// Application file name
    wchar_t strFileName[ADL_MAX_PATH];  // 应用程序文件名
    /// Application file path
    wchar_t strFilePath[ADL_MAX_PATH];  // 应用程序文件路径
    /// Application version
    wchar_t strVersion[ADL_MAX_PATH];  // 应用程序版本
    /// Timestamp at which application has run
    long long int timeStamp;  // 应用程序运行的时间戳
    /// Holds whether the applicaition profile exists or not
    unsigned int iProfileExists;  // 指示应用程序配置文件是否存在
    /// The GPU on which application runs
    unsigned int iGPUAffinity;  // 应用程序运行的 GPU
    /// The BDF of the GPU on which application runs
    ADLBdf GPUBdf;  // 应用程序运行的 GPU 的 BDF
} ADLSGApplicationInfo;

/////////////////////////////////////////////////////////////////////////////////////////////
///\brief Structure containing information related Frames Per Second for AC and DC.
///
/// This structure is used to store information related AC and DC Frames Per Second settings
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
// 枚举值，表示无效的 LUT 索引
enum { ADLPreFlipPostProcessingInfoInvalidLUTIndex = 0xFFFFFFFF };

// 枚举类型，表示预翻转后处理的 LUT 算法
enum ADLPreFlipPostProcessingLUTAlgorithm
{
    ADLPreFlipPostProcessingLUTAlgorithm_Default = 0,  // 默认算法
    ADLPreFlipPostProcessingLUTAlgorithm_Full,  // 完整算法
    ADLPreFlipPostProcessingLUTAlgorithm_Approximation  // 近似算法
};

// 结构体，存储与 AC 和 DC 的每秒帧数相关的信息
typedef struct _ADLPreFlipPostProcessingInfo
{
    /// size
    int ulSize;  // 大小
    /// Current active state
    int bEnabled;  // 当前激活状态
    /// Current selected LUT index.  0xFFFFFFF returned if nothing selected.
    int ulSelectedLUTIndex;  // 当前选择的 LUT 索引。如果未选择任何内容，则返回 0xFFFFFFF。
    /// Current selected LUT Algorithm
    int ulSelectedLUTAlgorithm;  // 当前选择的 LUT 算法
    /// Reserved
    int ulReserved[12];  // 保留字段
} ADLPreFlipPostProcessingInfo;

// 结构体，存储错误原因
typedef struct _ADL_ERROR_REASON
{
    int boost; //ON, when boost is Enabled  // 当启用增强时为 ON
    int delag; //ON, when delag is Enabled  // 当启用 delag 时为 ON
    int chill; //ON, when chill is Enabled  // 当启用 chill 时为 ON
// 定义枚举类型，包含 ADL 错误原因
}ADL_ERROR_REASON;

/////////////////////////////////////////////////////////////////////////////////////////////
///\brief 包含有关 DELAG 设置更改原因的信息的结构
///
///  DELAG 设置更改原因的元素。
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
typedef struct _ADL_DELAG_NOTFICATION_REASON
{
    int HotkeyChanged; // 当热键值更改时设置
    int GlobalEnableChanged; // 当全局启用值更改时设置
    int GlobalLimitFPSChanged; // 当全局启用值更改时设置
}ADL_DELAG_NOTFICATION_REASON;

/////////////////////////////////////////////////////////////////////////////////////////////
///\brief 包含有关 DELAG 设置的信息的结构
///
///  DELAG 设置的元素。
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
typedef struct _ADL_DELAG_SETTINGS
{
    int Hotkey; // 热键值
    int GlobalEnable; // 全局启用值
    int GlobalLimitFPS; // 全局限制 FPS
    int GlobalLimitFPS_MinLimit; // 全局限制 FPS 滑块最小限制值
    int GlobalLimitFPS_MaxLimit; // 全局限制 FPS 滑块最大限制值
    int GlobalLimitFPS_Step; // 全局限制 FPS 步长值
}ADL_DELAG_SETTINGS;

/////////////////////////////////////////////////////////////////////////////////////////////
///\brief 包含有关 BOOST 设置更改原因的信息的结构
///
///  BOOST 设置更改原因的元素。
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
typedef struct _ADL_BOOST_NOTFICATION_REASON
{
    int HotkeyChanged; // 当热键值更改时设置
    int GlobalEnableChanged; // 当全局启用值更改时设置
    int GlobalMinResChanged; // 当全局最小分辨率值更改时设置
}ADL_BOOST_NOTFICATION_REASON;
// 结构体，包含有关 BOOST 设置的信息
typedef struct _ADL_BOOST_SETTINGS
{
    int Hotkey; // 热键数值
    int GlobalEnable; // 全局启用数值
    int GlobalMinRes; // 全局最小分辨率数值
    int GlobalMinRes_MinLimit; // 全局最小分辨率滑块最小限制数值
    int GlobalMinRes_MaxLimit; // 全局最小分辨率滑块最大限制数值
    int GlobalMinRes_Step; // 全局最小分辨率步长数值
}ADL_BOOST_SETTINGS;

// 结构体，包含有关 RIS 设置更改原因的信息
typedef struct _ADL_RIS_NOTFICATION_REASON
{
    unsigned int GlobalEnableChanged; // 当全局启用数值更改时设置
    unsigned int GlobalSharpeningDegreeChanged; // 当全局锐化度数值更改时设置
}ADL_RIS_NOTFICATION_REASON;

// 结构体，包含有关 RIS 设置的信息
typedef struct _ADL_RIS_SETTINGS
{
    int GlobalEnable; // 全局启用数值
    int GlobalSharpeningDegree; // 全局锐化度数值
    int GlobalSharpeningDegree_MinLimit; // 全局锐化度滑块最小限制数值
    int GlobalSharpeningDegree_MaxLimit; // 全局锐化度滑块最大限制数值
    int GlobalSharpeningDegree_Step; // 全局锐化度步长数值
}ADL_RIS_SETTINGS;
// 定义了一个结构体，包含了关于 CHILL 设置更改原因的信息
// Chiil 设置更改原因的元素
typedef struct _ADL_CHILL_NOTFICATION_REASON
{
    int HotkeyChanged; // 当热键值改变时设置
    int GlobalEnableChanged; // 当全局启用值改变时设置
    int GlobalMinFPSChanged; // 当全局最小 FPS 值改变时设置
    int GlobalMaxFPSChanged; // 当全局最大 FPS 值改变时设置
}ADL_CHILL_NOTFICATION_REASON;

// 定义了一个结构体，包含了关于 CHILL 设置的信息
// Chiil 设置的元素
typedef struct _ADL_CHILL_SETTINGS
{
    int Hotkey; // 热键值
    int GlobalEnable; // 全局启用值
    int GlobalMinFPS; // 全局最小 FPS 值
    int GlobalMaxFPS; // 全局最大 FPS 值
    int GlobalFPS_MinLimit; // 全局 FPS 滑块最小限制值
    int GlobalFPS_MaxLimit; // 全局 FPS 滑块最大限制值
    int GlobalFPS_Step; // 全局 FPS 滑块步长值
}ADL_CHILL_SETTINGS;
#endif /* ADL_STRUCTURES_H_ */
```