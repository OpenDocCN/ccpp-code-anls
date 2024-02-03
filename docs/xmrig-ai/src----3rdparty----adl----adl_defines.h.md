# `xmrig\src\3rdparty\adl\adl_defines.h`

```cpp
//
// Copyright (c) 2016 Advanced Micro Devices, Inc. All rights reserved.
//
// MIT LICENSE:
// Permission is hereby granted, free of charge, to any person obtaining a copy
// of this software and associated documentation files (the "Software"), to deal
// in the Software without restriction, including without limitation the rights
// to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
// copies of the Software, and to permit persons to whom the Software is
// furnished to do so, subject to the following conditions:
//
// The above copyright notice and this permission notice shall be included in
// all copies or substantial portions of the Software.
//
// THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
// IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
// FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
// AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
// LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
// OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
// SOFTWARE.

/// \file adl_defines.h
/// \brief Contains all definitions exposed by ADL for \ALL platforms.\n <b>Included in ADL SDK</b>
///
/// This file contains all definitions used by ADL.
/// The ADL definitions include the following:
/// \li ADL error codes
/// \li Enumerations for the ADLDisplayInfo structure
/// \li Maximum limits
///

#ifndef ADL_DEFINES_H_
#define ADL_DEFINES_H_

/// \defgroup DEFINES Constants and Definitions
// @{

/// \defgroup define_misc Miscellaneous Constant Definitions
// @{

/// \name General Definitions
// @{

/// Defines ADL_TRUE
#define ADL_TRUE    1  // 定义ADL_TRUE为1
/// Defines ADL_FALSE
#define ADL_FALSE        0  // 定义ADL_FALSE为0

/// Defines the maximum string length
#define ADL_MAX_CHAR                                    4096  // 定义最大字符串长度为4096
/// Defines the maximum string length
#define ADL_MAX_PATH                                    256  // 定义最大路径长度为256
/// 定义支持的最大适配器数量
#define ADL_MAX_ADAPTERS 250
/// 定义支持的最大显示器数量
#define ADL_MAX_DISPLAYS 150
/// 定义设备名称的最大字符串长度
#define ADL_MAX_DEVICENAME 32
/// 定义所有适配器
#define ADL_ADAPTER_INDEX_ALL -1
/// 定义具有 iOption 为 none 的 API
#define ADL_MAIN_API_OPTION_NONE 0
// @}

/// \name 由 ADL_Display_DDCBlockAccess_Get() 使用的 iOption 参数的定义
// @{

/// 在发送命令到显示器之前切换到 DDC 线路 2
#define ADL_DDC_OPTION_SWITCHDDC2 0x00000001
/// 将命令保存在注册表中，对应参数 \b iCommandIndex
#define ADL_DDC_OPTION_RESTORECOMMAND 0x00000002
/// 组合写-读 DDC 块访问命令
#define ADL_DDC_OPTION_COMBOWRITEREAD 0x00000010
/// 直接访问连接到显卡的设备的 DDC
/// 设置此选项的 MST：DDC 命令发送到第一个分支
/// 未设置此选项的 MST：DDC 命令发送到最终节点接收设备
#define ADL_DDC_OPTION_SENDTOIMMEDIATEDEVICE 0x00000020
// @}

/// \name 与 ADL_Display_WriteAndReadI2C() 一起使用的 ADLI2C.iAction 的值
// @{

#define ADL_DL_I2C_ACTIONREAD 0x00000001
#define ADL_DL_I2C_ACTIONWRITE 0x00000002
#define ADL_DL_I2C_ACTIONREAD_REPEATEDSTART 0x00000003
// @}


// @}        //Misc

/// \defgroup define_adl_results 结果代码
/// 这些定义组是所有 ADL 函数返回的各种结果 \n
// @{
/// 一切正常，但需要等待
#define ADL_OK_WAIT 4
/// 一切正常，但需要重新启动
#define ADL_OK_RESTART 3
/// 一切正常，但需要模式更改
#define ADL_OK_MODE_CHANGE 2
/// 一切正常，但有警告
// 定义ADL_OK_WARNING为1，表示ADL函数成功完成
#define ADL_OK_WARNING                1
/// ADL函数成功完成
#define ADL_OK                    0
/// 通用错误。很可能是对驱动程序的一个或多个Escape调用失败！
#define ADL_ERR                    -1
/// ADL未初始化
#define ADL_ERR_NOT_INIT            -2
/// 传递的参数之一无效
#define ADL_ERR_INVALID_PARAM            -3
/// 传递的参数大小无效
#define ADL_ERR_INVALID_PARAM_SIZE        -4
/// 传递的ADL索引无效
#define ADL_ERR_INVALID_ADL_IDX            -5
/// 传递的控制器索引无效
#define ADL_ERR_INVALID_CONTROLLER_IDX        -6
/// 传递的显示索引无效
#define ADL_ERR_INVALID_DIPLAY_IDX        -7
/// 驱动程序不支持的功能
#define ADL_ERR_NOT_SUPPORTED            -8
/// 空指针错误
#define ADL_ERR_NULL_POINTER            -9
/// 由于禁用的适配器而无法进行调用
#define ADL_ERR_DISABLED_ADAPTER        -10
/// 无效的回调
#define ADL_ERR_INVALID_CALLBACK            -11
/// 显示资源冲突
#define ADL_ERR_RESOURCE_CONFLICT                -12
//更新某些值失败。如果包含多个值的设置请求未成功提交所有值，则可以返回此错误。
#define ADL_ERR_SET_INCOMPLETE                 -20
/// 在Linux控制台环境中没有Linux XDisplay
#define ADL_ERR_NO_XDISPLAY                    -21

// @}
/// </A>

/// \defgroup define_display_type 显示类型
/// 定义监视器/CRT显示类型
// @{
/// 定义监视器显示类型
#define ADL_DT_MONITOR                  0
/// 定义电视显示类型
#define ADL_DT_TELEVISION                    1
/// 定义LCD显示类型
#define ADL_DT_LCD_PANEL                       2
/// 定义DFP显示类型
#define ADL_DT_DIGITAL_FLAT_PANEL        3
/// 定义组件视频显示类型
#define ADL_DT_COMPONENT_VIDEO               4
/// 定义投影仪显示类型
#define ADL_DT_PROJECTOR                       5
// @}
/// \defgroup define_display_connection_type Display Connection Type
// @{
/// 定义未知的显示输出类型
#define ADL_DOT_UNKNOWN                0
/// 定义复合显示输出类型
#define ADL_DOT_COMPOSITE            1
/// 定义SVideo显示输出类型
#define ADL_DOT_SVIDEO                2
/// 定义模拟显示输出类型
#define ADL_DOT_ANALOG                3
/// 定义数字显示输出类型
#define ADL_DOT_DIGITAL                4
// @}

/// \defgroup define_color_type Display Color Type and Source
/// 定义显示颜色类型和来源
// @{
#define ADL_DISPLAY_COLOR_BRIGHTNESS    (1 << 0)
#define ADL_DISPLAY_COLOR_CONTRAST    (1 << 1)
#define ADL_DISPLAY_COLOR_SATURATION    (1 << 2)
#define ADL_DISPLAY_COLOR_HUE        (1 << 3)
#define ADL_DISPLAY_COLOR_TEMPERATURE    (1 << 4)

/// 颜色温度来源是EDID
#define ADL_DISPLAY_COLOR_TEMPERATURE_SOURCE_EDID    (1 << 5)
/// 颜色温度来源是用户
#define ADL_DISPLAY_COLOR_TEMPERATURE_SOURCE_USER    (1 << 6)
// @}

/// \defgroup define_adjustment_capabilities Display Adjustment Capabilities
/// 显示调整能力值。由ADL_Display_AdjustCaps_Get返回
// @{
#define ADL_DISPLAY_ADJUST_OVERSCAN        (1 << 0)
#define ADL_DISPLAY_ADJUST_VERT_POS        (1 << 1)
#define ADL_DISPLAY_ADJUST_HOR_POS        (1 << 2)
#define ADL_DISPLAY_ADJUST_VERT_SIZE        (1 << 3)
#define ADL_DISPLAY_ADJUST_HOR_SIZE        (1 << 4)
#define ADL_DISPLAY_ADJUST_SIZEPOS        (ADL_DISPLAY_ADJUST_VERT_POS | ADL_DISPLAY_ADJUST_HOR_POS | ADL_DISPLAY_ADJUST_VERT_SIZE | ADL_DISPLAY_ADJUST_HOR_SIZE)
#define ADL_DISPLAY_CUSTOMMODES            (1<<5)
#define ADL_DISPLAY_ADJUST_UNDERSCAN        (1<<6)
// @}

///Down-scale support
#define ADL_DISPLAY_CAPS_DOWNSCALE        (1 << 0)

/// Sharpness support
#define ADL_DISPLAY_CAPS_SHARPNESS      (1 << 0)

/// \defgroup define_desktop_config Desktop Configuration Flags
/// 这些标志由ADL_DesktopConfig_xxx使用
/// \deprecated This API has been deprecated because it was only used for RandR 1.1 (Red Hat 5.x) distributions which is now not supported.
// @{
// 定义了一系列与桌面配置相关的常量，用于描述不同的桌面配置
#define ADL_DESKTOPCONFIG_UNKNOWN    0          /* UNKNOWN desktop config   */
#define ADL_DESKTOPCONFIG_SINGLE     (1 <<  0)    /* Single                   */
#define ADL_DESKTOPCONFIG_CLONE      (1 <<  2)    /* Clone                    */
#define ADL_DESKTOPCONFIG_BIGDESK_H  (1 <<  4)    /* Big Desktop Horizontal   */
#define ADL_DESKTOPCONFIG_BIGDESK_V  (1 <<  5)    /* Big Desktop Vertical     */
#define ADL_DESKTOPCONFIG_BIGDESK_HR (1 <<  6)    /* Big Desktop Reverse Horz */
#define ADL_DESKTOPCONFIG_BIGDESK_VR (1 <<  7)    /* Big Desktop Reverse Vert */
#define ADL_DESKTOPCONFIG_RANDR12    (1 <<  8)    /* RandR 1.2 Multi-display */
// @}

/// needed for ADLDDCInfo structure
// 定义了用于 ADLDDCInfo 结构的常量
#define ADL_MAX_DISPLAY_NAME                                256

/// \defgroup define_edid_flags Values for ulDDCInfoFlag
/// defines for ulDDCInfoFlag EDID flag
// @{
// 定义了一系列与 EDID 标志相关的常量
#define ADL_DISPLAYDDCINFOEX_FLAG_PROJECTORDEVICE       (1 << 0)
#define ADL_DISPLAYDDCINFOEX_FLAG_EDIDEXTENSION         (1 << 1)
#define ADL_DISPLAYDDCINFOEX_FLAG_DIGITALDEVICE         (1 << 2)
#define ADL_DISPLAYDDCINFOEX_FLAG_HDMIAUDIODEVICE       (1 << 3)
#define ADL_DISPLAYDDCINFOEX_FLAG_SUPPORTS_AI           (1 << 4)
#define ADL_DISPLAYDDCINFOEX_FLAG_SUPPORT_xvYCC601      (1 << 5)
#define ADL_DISPLAYDDCINFOEX_FLAG_SUPPORT_xvYCC709      (1 << 6)
// @}

/// \defgroup define_displayinfo_connector Display Connector Type
/// defines for ADLDisplayInfo.iDisplayConnector
// @{
// 定义了一系列与显示器连接器类型相关的常量
#define ADL_DISPLAY_CONTYPE_UNKNOWN                 0
#define ADL_DISPLAY_CONTYPE_VGA                     1
#define ADL_DISPLAY_CONTYPE_DVI_D                   2
#define ADL_DISPLAY_CONTYPE_DVI_I                   3
#define ADL_DISPLAY_CONTYPE_ATICVDONGLE_NTSC        4
#define ADL_DISPLAY_CONTYPE_ATICVDONGLE_JPN         5
#define ADL_DISPLAY_CONTYPE_ATICVDONGLE_NONI2C_JPN  6
// 定义 ATI/CVDongle 非 I2C NTSC 显示连接类型
#define ADL_DISPLAY_CONTYPE_ATICVDONGLE_NONI2C_NTSC 7
// 定义专有显示连接类型
#define ADL_DISPLAY_CONTYPE_PROPRIETARY 8
// 定义 HDMI 类型 A 显示连接类型
#define ADL_DISPLAY_CONTYPE_HDMI_TYPE_A 10
// 定义 HDMI 类型 B 显示连接类型
#define ADL_DISPLAY_CONTYPE_HDMI_TYPE_B 11
// 定义 S-Video 显示连接类型
#define ADL_DISPLAY_CONTYPE_SVIDEO 12
// 定义复合视频显示连接类型
#define ADL_DISPLAY_CONTYPE_COMPOSITE 13
// 定义 RCA 3 分量显示连接类型
#define ADL_DISPLAY_CONTYPE_RCA_3COMPONENT 14
// 定义 DisplayPort 显示连接类型
#define ADL_DISPLAY_CONTYPE_DISPLAYPORT 15
// 定义 EDP 显示连接类型
#define ADL_DISPLAY_CONTYPE_EDP 16
// 定义无线显示连接类型
#define ADL_DISPLAY_CONTYPE_WIRELESSDISPLAY 17

// TV Capabilities and Standards
// TV 显示能力和标准
// 不推荐使用 TV 显示
#define ADL_TV_STANDARDS (1 << 0)
#define ADL_TV_SCART (1 << 1)

// TV 标准定义
#define ADL_STANDARD_NTSC_M (1 << 0)
#define ADL_STANDARD_NTSC_JPN (1 << 1)
#define ADL_STANDARD_NTSC_N (1 << 2)
#define ADL_STANDARD_PAL_B (1 << 3)
#define ADL_STANDARD_PAL_COMB_N (1 << 4)
#define ADL_STANDARD_PAL_D (1 << 5)
#define ADL_STANDARD_PAL_G (1 << 6)
#define ADL_STANDARD_PAL_H (1 << 7)
#define ADL_STANDARD_PAL_I (1 << 8)
#define ADL_STANDARD_PAL_K (1 << 9)
#define ADL_STANDARD_PAL_K1 (1 << 10)
#define ADL_STANDARD_PAL_L (1 << 11)
#define ADL_STANDARD_PAL_M (1 << 12)
#define ADL_STANDARD_PAL_N (1 << 13)
#define ADL_STANDARD_PAL_SECAM_D (1 << 14)
#define ADL_STANDARD_PAL_SECAM_K (1 << 15)
#define ADL_STANDARD_PAL_SECAM_K1 (1 << 16)
#define ADL_STANDARD_PAL_SECAM_L (1 << 17)

// Video Custom Mode flags
// 视频自定义模式标志
// 组件视频自定义模式标志，由 ADLCustomMode 中的 iFlags 参数使用
#define ADL_CUSTOMIZEDMODEFLAG_MODESUPPORTED (1 << 0)
#define ADL_CUSTOMIZEDMODEFLAG_NOTDELETETABLE (1 << 1)
#define ADL_CUSTOMIZEDMODEFLAG_INSERTBYDRIVER (1 << 2)
// 定义自定义模式标志，用于指示是否交错
#define ADL_CUSTOMIZEDMODEFLAG_INTERLACED    (1 << 3)
// 定义自定义模式标志，用于指示是否基本模式
#define ADL_CUSTOMIZEDMODEFLAG_BASEMODE        (1 << 4)

/// \defgroup define_ddcinfoflag Values used for DDCInfoFlag
/// ulDDCInfoFlag field values used by the ADLDDCInfo structure
// @{
// 定义用于 DDCInfoFlag 的值，用于指示投影仪设备
#define ADL_DISPLAYDDCINFOEX_FLAG_PROJECTORDEVICE    (1 << 0)
// 定义用于 DDCInfoFlag 的值，用于指示 EDID 扩展
#define ADL_DISPLAYDDCINFOEX_FLAG_EDIDEXTENSION        (1 << 1)
// 定义用于 DDCInfoFlag 的值，用于指示数字设备
#define ADL_DISPLAYDDCINFOEX_FLAG_DIGITALDEVICE        (1 << 2)
// 定义用于 DDCInfoFlag 的值，用于指示 HDMI 音频设备
#define ADL_DISPLAYDDCINFOEX_FLAG_HDMIAUDIODEVICE    (1 << 3)
// 定义用于 DDCInfoFlag 的值，用于指示是否支持 AI
#define ADL_DISPLAYDDCINFOEX_FLAG_SUPPORTS_AI        (1 << 4)
// 定义用于 DDCInfoFlag 的值，用于指示是否支持 xvYCC601
#define ADL_DISPLAYDDCINFOEX_FLAG_SUPPORT_xvYCC601    (1 << 5)
// 定义用于 DDCInfoFlag 的值，用于指示是否支持 xvYCC709
#define ADL_DISPLAYDDCINFOEX_FLAG_SUPPORT_xvYCC709    (1 << 6)
// @}

/// \defgroup define_cv_dongle Values used by ADL_CV_DongleSettings_xxx
/// The following is applicable to ADL_DISPLAY_CONTYPE_ATICVDONGLE_JP and ADL_DISPLAY_CONTYPE_ATICVDONGLE_NONI2C_D only
/// \deprecated Dropping support for Component Video displays
// @{
// 定义用于 ADL_CV_DongleSettings_xxx 的值，适用于 ATICVDONGLE_JP 和 ATICVDONGLE_NONI2C_D
// @deprecated 不再支持分量视频显示
#define ADL_DISPLAY_CV_DONGLE_D1          (1 << 0)
#define ADL_DISPLAY_CV_DONGLE_D2          (1 << 1)
#define ADL_DISPLAY_CV_DONGLE_D3          (1 << 2)
#define ADL_DISPLAY_CV_DONGLE_D4          (1 << 3)
#define ADL_DISPLAY_CV_DONGLE_D5          (1 << 4)

/// The following is applicable to ADL_DISPLAY_CONTYPE_ATICVDONGLE_NA and ADL_DISPLAY_CONTYPE_ATICVDONGLE_NONI2C only

// 定义用于 ADL_CV_DongleSettings_xxx 的值，适用于 ATICVDONGLE_NA 和 ATICVDONGLE_NONI2C
#define ADL_DISPLAY_CV_DONGLE_480I        (1 << 0)
#define ADL_DISPLAY_CV_DONGLE_480P        (1 << 1)
#define ADL_DISPLAY_CV_DONGLE_540P        (1 << 2)
#define ADL_DISPLAY_CV_DONGLE_720P        (1 << 3)
#define ADL_DISPLAY_CV_DONGLE_1080I       (1 << 4)
#define ADL_DISPLAY_CV_DONGLE_1080P       (1 << 5)
#define ADL_DISPLAY_CV_DONGLE_16_9        (1 << 6)
#define ADL_DISPLAY_CV_DONGLE_720P50      (1 << 7)
#define ADL_DISPLAY_CV_DONGLE_1080I25     (1 << 8)
#define ADL_DISPLAY_CV_DONGLE_576I25      (1 << 9)
#define ADL_DISPLAY_CV_DONGLE_576P50      (1 << 10)
#define ADL_DISPLAY_CV_DONGLE_1080P24      (1 << 11)
#define ADL_DISPLAY_CV_DONGLE_1080P25      (1 << 12)
// 定义 CV 适配器支持的不同显示模式
#define ADL_DISPLAY_CV_DONGLE_1080P30      (1 << 13)
#define ADL_DISPLAY_CV_DONGLE_1080P50      (1 << 14)

// 定义格式覆盖设置中的显示强制模式标志
#define ADL_DISPLAY_FORMAT_FORCE_720P        0x00000001
#define ADL_DISPLAY_FORMAT_FORCE_1080I        0x00000002
#define ADL_DISPLAY_FORMAT_FORCE_1080P        0x00000004
#define ADL_DISPLAY_FORMAT_FORCE_720P50        0x00000008
#define ADL_DISPLAY_FORMAT_FORCE_1080I25    0x00000010
#define ADL_DISPLAY_FORMAT_FORCE_576I25        0x00000020
#define ADL_DISPLAY_FORMAT_FORCE_576P50        0x00000040
#define ADL_DISPLAY_FORMAT_FORCE_1080P24    0x00000080
#define ADL_DISPLAY_FORMAT_FORCE_1080P25    0x00000100
#define ADL_DISPLAY_FORMAT_FORCE_1080P30    0x00000200
#define ADL_DISPLAY_FORMAT_FORCE_1080P50    0x00000400

// 以下是扩展显示模式标志
#define ADL_DISPLAY_FORMAT_CVDONGLEOVERIDE  0x00000001
#define ADL_DISPLAY_FORMAT_CVMODEUNDERSCAN  0x00000002
#define ADL_DISPLAY_FORMAT_FORCECONNECT_SUPPORTED  0x00000004
#define ADL_DISPLAY_FORMAT_RESTRICT_FORMAT_SELECTION 0x00000008
#define ADL_DISPLAY_FORMAT_SETASPECRATIO 0x00000010
#define ADL_DISPLAY_FORMAT_FORCEMODES    0x00000020
#define ADL_DISPLAY_FORMAT_LCDRTCCOEFF   0x00000040

// 定义 OD5 使用的参数
#define ADL_PM_PARAM_DONT_CHANGE    0

// 定义总线类型
#define ADL_BUSTYPE_PCI           0       /* PCI 总线                          */
#define ADL_BUSTYPE_AGP           1       /* AGP 总线                          */
#define ADL_BUSTYPE_PCIE          2       /* PCI Express 总线                  */
#define ADL_BUSTYPE_PCIE_GEN2     3       /* PCI Express 第二代总线           */
#define ADL_BUSTYPE_PCIE_GEN3     4       /* PCI Express 第三代总线           */
#define ADL_BUSTYPE_PCIE_GEN4     5       /* PCI Express 第四代总线           */

// 定义工作站能力
/// 定义工作站卡支持通过立体声输出连接器进行主动立体声
#define ADL_STEREO_SUPPORTED        (1 << 2)
/// 定义工作站卡支持通过“蓝线”进行主动立体声
#define ADL_STEREO_BLUE_LINE        (1 << 3)
/// 用于关闭立体声模式
#define ADL_STEREO_OFF                0
/// 定义工作站卡支持主动立体声。也用于将立体声模式设置为通过立体声输出连接器进行主动
#define ADL_STEREO_ACTIVE             (1 << 1)
/// 定义工作站卡支持具有水平交错的自动立体声监视器。也用于将立体声模式设置为使用具有水平交错的自动立体声监视器
#define ADL_STEREO_AUTO_HORIZONTAL    (1 << 30)
/// 定义工作站卡支持具有垂直交错的自动立体声监视器。也用于将立体声模式设置为使用具有垂直交错的自动立体声监视器
#define ADL_STEREO_AUTO_VERTICAL    (1 << 31)
/// 定义工作站卡支持被动立体声，即非立体声同步
#define ADL_STEREO_PASSIVE              (1 << 6)
/// 定义工作站卡支持具有水平交错的自动立体声监视器。也用于将立体声模式设置为使用具有水平交错的自动立体声监视器
#define ADL_STEREO_PASSIVE_HORIZ        (1 << 7)
/// 定义工作站卡支持具有垂直交错的自动立体声监视器。也用于将立体声模式设置为使用具有垂直交错的自动立体声监视器
#define ADL_STEREO_PASSIVE_VERT         (1 << 8)
/// 定义工作站卡支持具有三星自动立体声监视器
#define ADL_STEREO_AUTO_SAMSUNG        (1 << 11)
/// 定义工作站卡支持具有Tridility自动立体声监视器
// 定义自动TSL立体声
#define ADL_STEREO_AUTO_TSL         (1 << 12)
// 表示工作站卡支持DeepBitDepth（10 bpp）
#define ADL_DEEPBITDEPTH_10BPP_SUPPORTED   (1 << 5)

// 表示工作站支持8位灰度
#define ADL_8BIT_GREYSCALE_SUPPORTED   (1 << 9)
// 表示工作站支持自定义定时
#define ADL_CUSTOM_TIMING_SUPPORTED   (1 << 10)

// 支持负载均衡
#define ADL_WORKSTATION_LOADBALANCING_SUPPORTED         0x00000001
// 可用负载均衡
#define ADL_WORKSTATION_LOADBALANCING_AVAILABLE         0x00000002

// 禁用负载均衡
#define ADL_WORKSTATION_LOADBALANCING_DISABLED          0x00000000
// 启用负载均衡
#define ADL_WORKSTATION_LOADBALANCING_ENABLED           0x00000001

// 适配器速度设置
#define ADL_CONTEXT_SPEED_UNFORCED        0        /* 默认ASIC运行速度 */
#define ADL_CONTEXT_SPEED_FORCEHIGH        1        /* ASIC运行速度强制为高 */
#define ADL_CONTEXT_SPEED_FORCELOW        2        /* ASIC运行速度强制为低 */

#define ADL_ADAPTER_SPEEDCAPS_SUPPORTED        (1 << 0)    /* 改变ASIC运行速度设置是支持的 */

// GL-Sync相关值
// GL-Sync端口类型（唯一值）
#define ADL_GLSYNC_PORT_UNKNOWN        0
#define ADL_GLSYNC_PORT_BNC            1
#define ADL_GLSYNC_PORT_RJ45PORT1    2
#define ADL_GLSYNC_PORT_RJ45PORT2    3

// GL-Sync Genlock设置掩码（位向量）
// ADLGLSyncGenlockConfig成员都无效
#define ADL_GLSYNC_CONFIGMASK_NONE                0
// ADLGLSyncGenlockConfig.lSignalSource成员有效
#define ADL_GLSYNC_CONFIGMASK_SIGNALSOURCE        (1 << 0)
// 定义ADLGLSyncGenlockConfig.iSyncField成员有效的掩码
#define ADL_GLSYNC_CONFIGMASK_SYNCFIELD            (1 << 1)
// 定义ADLGLSyncGenlockConfig.iSampleRate成员有效的掩码
#define ADL_GLSYNC_CONFIGMASK_SAMPLERATE        (1 << 2)
// 定义ADLGLSyncGenlockConfig.lSyncDelay成员有效的掩码
#define ADL_GLSYNC_CONFIGMASK_SYNCDELAY            (1 << 3)
// 定义ADLGLSyncGenlockConfig.iTriggerEdge成员有效的掩码
#define ADL_GLSYNC_CONFIGMASK_TRIGGEREDGE        (1 << 4)
// 定义ADLGLSyncGenlockConfig.iScanRateCoeff成员有效的掩码
#define ADL_GLSYNC_CONFIGMASK_SCANRATECOEFF        (1 << 5)
// 定义ADLGLSyncGenlockConfig.lFramelockCntlVector成员有效的掩码
#define ADL_GLSYNC_CONFIGMASK_FRAMELOCKCNTL        (1 << 6)

// GL-Sync帧锁控制掩码（位向量）

// 帧锁已禁用
#define ADL_GLSYNC_FRAMELOCKCNTL_NONE            0
// 帧锁已启用
#define ADL_GLSYNC_FRAMELOCKCNTL_ENABLE            ( 1 << 0)
// 帧锁已禁用
#define ADL_GLSYNC_FRAMELOCKCNTL_DISABLE        ( 1 << 1)
// 交换计数器复位
#define ADL_GLSYNC_FRAMELOCKCNTL_SWAP_COUNTER_RESET    ( 1 << 2)
// 交换计数器确认
#define ADL_GLSYNC_FRAMELOCKCNTL_SWAP_COUNTER_ACK    ( 1 << 3)
// KMD版本
#define ADL_GLSYNC_FRAMELOCKCNTL_VERSION_KMD    (1 << 4)
// 帧锁状态启用
#define ADL_GLSYNC_FRAMELOCKCNTL_STATE_ENABLE        ( 1 << 0)
// KMD状态
#define ADL_GLSYNC_FRAMELOCKCNTL_STATE_KMD        (1 << 4)

// GL-Sync帧锁计数器掩码（位向量）
#define ADL_GLSYNC_COUNTER_SWAP                ( 1 << 0 )

// GL-Sync信号源（唯一值）

// GL-Sync信号源未定义
#define ADL_GLSYNC_SIGNALSOURCE_UNDEFINED    0x00000100
// GL-Sync信号源为自由运行
#define ADL_GLSYNC_SIGNALSOURCE_FREERUN      0x00000101
// GL-Sync信号源为BNC GL-Sync端口
#define ADL_GLSYNC_SIGNALSOURCE_BNCPORT      0x00000102
// GL-Sync信号源为RJ45(1) GL-Sync端口
#define ADL_GLSYNC_SIGNALSOURCE_RJ45PORT1    0x00000103
// GL-Sync信号源为RJ45(2) GL-Sync端口
#define ADL_GLSYNC_SIGNALSOURCE_RJ45PORT2    0x00000104
// GL-Sync 信号类型（唯一值）

/// GL-Sync 信号类型未知
#define ADL_GLSYNC_SIGNALTYPE_UNDEFINED      0
/// GL-Sync 信号类型为 480I
#define ADL_GLSYNC_SIGNALTYPE_480I           1
/// GL-Sync 信号类型为 576I
#define ADL_GLSYNC_SIGNALTYPE_576I           2
/// GL-Sync 信号类型为 480P
#define ADL_GLSYNC_SIGNALTYPE_480P           3
/// GL-Sync 信号类型为 576P
#define ADL_GLSYNC_SIGNALTYPE_576P           4
/// GL-Sync 信号类型为 720P
#define ADL_GLSYNC_SIGNALTYPE_720P           5
/// GL-Sync 信号类型为 1080P
#define ADL_GLSYNC_SIGNALTYPE_1080P          6
/// GL-Sync 信号类型为 1080I
#define ADL_GLSYNC_SIGNALTYPE_1080I          7
/// GL-Sync 信号类型为 SDI
#define ADL_GLSYNC_SIGNALTYPE_SDI            8
/// GL-Sync 信号类型为 TTL
#define ADL_GLSYNC_SIGNALTYPE_TTL            9
/// GL_Sync 信号类型为 Analog
#define ADL_GLSYNC_SIGNALTYPE_ANALOG        10

// GL-Sync 同步字段选项（唯一值）

/// GL-Sync 同步字段选项未定义
#define ADL_GLSYNC_SYNCFIELD_UNDEFINED        0
/// GL-Sync 同步字段选项为同步到字段 1（用于交错信号类型）
#define ADL_GLSYNC_SYNCFIELD_BOTH            1
/// GL-Sync 同步字段选项为同步到两个字段（用于交错信号类型）
#define ADL_GLSYNC_SYNCFIELD_1                2


// GL-Sync 触发边缘选项（唯一值）

/// GL-Sync 触发边缘未定义
#define ADL_GLSYNC_TRIGGEREDGE_UNDEFINED     0
/// GL-Sync 触发边缘为上升沿
#define ADL_GLSYNC_TRIGGEREDGE_RISING        1
/// GL-Sync 触发边缘为下降沿
#define ADL_GLSYNC_TRIGGEREDGE_FALLING       2
/// GL-Sync 触发边缘为上升和下降沿
#define ADL_GLSYNC_TRIGGEREDGE_BOTH          3


// GL-Sync 扫描率系数/倍增器选项（唯一值）

/// GL-Sync 扫描率系数/倍增器未定义
#define ADL_GLSYNC_SCANRATECOEFF_UNDEFINED   0
/// GL-Sync 扫描率系数/倍增器为 5
// 定义 GL-Sync 扫描率系数/倍增器为 5
#define ADL_GLSYNC_SCANRATECOEFF_x5          1
// 定义 GL-Sync 扫描率系数/倍增器为 4
#define ADL_GLSYNC_SCANRATECOEFF_x4          2
// 定义 GL-Sync 扫描率系数/倍增器为 3
#define ADL_GLSYNC_SCANRATECOEFF_x3          3
// 定义 GL-Sync 扫描率系数/倍增器为 5:2 (SMPTE)
#define ADL_GLSYNC_SCANRATECOEFF_x5_DIV_2    4
// 定义 GL-Sync 扫描率系数/倍增器为 2
#define ADL_GLSYNC_SCANRATECOEFF_x2          5
// 定义 GL-Sync 扫描率系数/倍增器为 3 : 2
#define ADL_GLSYNC_SCANRATECOEFF_x3_DIV_2    6
// 定义 GL-Sync 扫描率系数/倍增器为 5 : 4
#define ADL_GLSYNC_SCANRATECOEFF_x5_DIV_4    7
// 定义 GL-Sync 扫描率系数/倍增器为 1 (默认)
#define ADL_GLSYNC_SCANRATECOEFF_x1          8
// 定义 GL-Sync 扫描率系数/倍增器为 4 : 5
#define ADL_GLSYNC_SCANRATECOEFF_x4_DIV_5    9
// 定义 GL-Sync 扫描率系数/倍增器为 2 : 3
#define ADL_GLSYNC_SCANRATECOEFF_x2_DIV_3    10
// 定义 GL-Sync 扫描率系数/倍增器为 1 : 2
#define ADL_GLSYNC_SCANRATECOEFF_x1_DIV_2    11
// 定义 GL-Sync 扫描率系数/倍增器为 2 : 5 (SMPTE)
#define ADL_GLSYNC_SCANRATECOEFF_x2_DIV_5    12
// 定义 GL-Sync 扫描率系数/倍增器为 1 : 3
#define ADL_GLSYNC_SCANRATECOEFF_x1_DIV_3    13
// 定义 GL-Sync 扫描率系数/倍增器为 1 : 4
#define ADL_GLSYNC_SCANRATECOEFF_x1_DIV_4    14
// 定义 GL-Sync 扫描率系数/倍增器为 1 : 5
#define ADL_GLSYNC_SCANRATECOEFF_x1_DIV_5    15

// GL-Sync 端口（信号存在）状态（唯一值）

// 定义 GL-Sync 端口状态为未定义
#define ADL_GLSYNC_PORTSTATE_UNDEFINED       0
// 定义 GL-Sync 端口未连接
#define ADL_GLSYNC_PORTSTATE_NOCABLE         1
// 定义 GL-Sync 端口空闲
#define ADL_GLSYNC_PORTSTATE_IDLE            2
// 定义 GL-Sync 端口有输入信号
#define ADL_GLSYNC_PORTSTATE_INPUT           3
// 定义 GL-Sync 端口为输出
#define ADL_GLSYNC_PORTSTATE_OUTPUT          4
// GL-Sync LED types (used index within ADL_Workstation_GLSyncPortState_Get returned ppGlSyncLEDs array) (unique values)

/// Index into the ADL_Workstation_GLSyncPortState_Get returned ppGlSyncLEDs array for the one LED of the BNC port
#define ADL_GLSYNC_LEDTYPE_BNC               0
/// Index into the ADL_Workstation_GLSyncPortState_Get returned ppGlSyncLEDs array for the Left LED of the RJ45(1) or RJ45(2) port
#define ADL_GLSYNC_LEDTYPE_RJ45_LEFT         0
/// Index into the ADL_Workstation_GLSyncPortState_Get returned ppGlSyncLEDs array for the Right LED of the RJ45(1) or RJ45(2) port
#define ADL_GLSYNC_LEDTYPE_RJ45_RIGHT        1


// GL-Sync LED colors (unique values)

/// GL-Sync LED undefined color
#define ADL_GLSYNC_LEDCOLOR_UNDEFINED        0
/// GL-Sync LED is unlit
#define ADL_GLSYNC_LEDCOLOR_NOLIGHT          1
/// GL-Sync LED is yellow
#define ADL_GLSYNC_LEDCOLOR_YELLOW           2
/// GL-Sync LED is red
#define ADL_GLSYNC_LEDCOLOR_RED              3
/// GL-Sync LED is green
#define ADL_GLSYNC_LEDCOLOR_GREEN            4
/// GL-Sync LED is flashing green
#define ADL_GLSYNC_LEDCOLOR_FLASH_GREEN      5


// GL-Sync Port Control (refers one GL-Sync Port) (unique values)

/// Used to configure the RJ54(1) or RJ42(2) port of GL-Sync is as Idle
#define ADL_GLSYNC_PORTCNTL_NONE             0x00000000
/// Used to configure the RJ54(1) or RJ42(2) port of GL-Sync is as Output
#define ADL_GLSYNC_PORTCNTL_OUTPUT           0x00000001


// GL-Sync Mode Control (refers one Display/Controller) (bitfields)

/// Used to configure the display to use internal timing (not genlocked)
#define ADL_GLSYNC_MODECNTL_NONE             0x00000000
/// Bitfield used to configure the display as genlocked (either as Timing Client or as Timing Server)
#define ADL_GLSYNC_MODECNTL_GENLOCK          0x00000001
/// Bitfield used to configure the display as Timing Server
#define ADL_GLSYNC_MODECNTL_TIMINGSERVER     0x00000002

// GL-Sync Mode Status
/// Display is currently not genlocked
// 定义 GLSync 模式控制状态常量
#define ADL_GLSYNC_MODECNTL_STATUS_NONE         0x00000000
/// 显示器当前处于 genlock 状态
#define ADL_GLSYNC_MODECNTL_STATUS_GENLOCK   0x00000001
/// 显示器需要进行模式切换
#define ADL_GLSYNC_MODECNTL_STATUS_SETMODE_REQUIRED 0x00000002
/// 显示器可以进行 genlock
#define ADL_GLSYNC_MODECNTL_STATUS_GENLOCK_ALLOWED 0x00000004

// 定义 GLSync 端口数量的最大值
#define ADL_MAX_GLSYNC_PORTS                            8
// 定义 GLSync 端口 LED 数量的最大值
#define ADL_MAX_GLSYNC_PORT_LEDS                        8

// 定义 CrossfireX 状态的常量
/// 适配器 CrossfireX 组合的状态
#define ADL_XFIREX_STATE_NOINTERCONNECT            ( 1 << 0 )    /* 适配器之间的连接线/线缆丢失 */
#define ADL_XFIREX_STATE_DOWNGRADEPIPES            ( 1 << 1 )    /* 如果管道被降级，则可以启用 CrossfireX */
#define ADL_XFIREX_STATE_DOWNGRADEMEM            ( 1 << 2 )    /* 除非内存被降级，否则无法启用 CrossfireX */
#define ADL_XFIREX_STATE_REVERSERECOMMENDED        ( 1 << 3 )    /* 推荐卡片反转，无法启用 CrossfireX */
#define ADL_XFIREX_STATE_3DACTIVE            ( 1 << 4 )    /* 3D 客户端正在运行 - 无法安全启用 CrossfireX */
#define ADL_XFIREX_STATE_MASTERONSLAVE            ( 1 << 5 )    /* 连接线正常，但主卡连接到从卡 */
#define ADL_XFIREX_STATE_NODISPLAYCONNECT        ( 1 << 6 )    /* 主卡未连接任何（有效的）显示器 */
#define ADL_XFIREX_STATE_NOPRIMARYVIEW            ( 1 << 7 )    /* CrossfireX 已启用，但主卡不是当前主设备 */
#define ADL_XFIREX_STATE_DOWNGRADEVISMEM        ( 1 << 8 )    /* 除非可见内存被降级，否则无法启用 CrossfireX */
#define ADL_XFIREX_STATE_LESSTHAN8LANE_MASTER        ( 1 << 9 )     /* 可以启用 CrossfireX，但由于小于 8 个通道，性能不佳 */
#define ADL_XFIREX_STATE_LESSTHAN8LANE_SLAVE        ( 1 << 10 )    /* 可以启用 CrossfireX，但由于小于 8 个通道，性能不佳 */
# 定义 CrossfireX 状态标志位，表示对等连接失败
#define ADL_XFIREX_STATE_PEERTOPEERFAILED        ( 1 << 11 )    /* CrossfireX cannot be enabled due to failed peer to peer test */
# 定义 CrossfireX 状态标志位，表示内存当前处于降级状态
#define ADL_XFIREX_STATE_MEMISDOWNGRADED        ( 1 << 16 )    /* Notification that memory is currently downgraded */
# 定义 CrossfireX 状态标志位，表示管道当前处于降级状态
#define ADL_XFIREX_STATE_PIPESDOWNGRADED        ( 1 << 17 )    /* Notification that pipes are currently downgraded */
# 定义 CrossfireX 状态标志位，表示当前设备上启用了 CrossfireX
#define ADL_XFIREX_STATE_XFIREXACTIVE            ( 1 << 18 )    /* CrossfireX is enabled on current device */
# 定义 CrossfireX 状态标志位，表示可见的 FB 内存当前处于降级状态
#define ADL_XFIREX_STATE_VISMEMISDOWNGRADED        ( 1 << 19 )    /* Notification that visible FB memory is currently downgraded */
# 定义 CrossfireX 状态标志位，表示不支持当前的互连配置
#define ADL_XFIREX_STATE_INVALIDINTERCONNECTION        ( 1 << 20 )    /* Cannot support current inter-connection configuration */
# 定义 CrossfireX 状态标志位，表示只有支持非 P2P 模式的客户端才能使用 CrossfireX
#define ADL_XFIREX_STATE_NONP2PMODE            ( 1 << 21 )    /* CrossfireX will only work with clients supporting non P2P mode */
# 定义 CrossfireX 状态标志位，表示除非内存降级，否则无法启用 CrossfireX
#define ADL_XFIREX_STATE_DOWNGRADEMEMBANKS        ( 1 << 22 )    /* CrossfireX cannot be enabled unless memory banks downgraded */
# 定义 CrossfireX 状态标志位，表示内存银行当前处于降级状态
#define ADL_XFIREX_STATE_MEMBANKSDOWNGRADED        ( 1 << 23 )    /* Notification that memory banks are currently downgraded */
# 定义 CrossfireX 状态标志位，表示允许扩展桌面或克隆模式
#define ADL_XFIREX_STATE_DUALDISPLAYSALLOWED        ( 1 << 24 )    /* Extended desktop or clone mode is allowed. */
# 定义 CrossfireX 状态标志位，表示 P2P 映射是通过对等孔径进行的
#define ADL_XFIREX_STATE_P2P_APERTURE_MAPPING        ( 1 << 25 )    /* P2P mapping was through peer aperture */
# 定义 CrossfireX 状态标志位，表示需要通过对等孔径进行 P2P 刷新
#define ADL_XFIREX_STATE_P2PFLUSH_REQUIRED        ADL_XFIREX_STATE_P2P_APERTURE_MAPPING    /* For back compatible */
# 定义 CrossfireX 状态标志位，表示 GPU 之间存在 CrossfireX 侧口连接
#define ADL_XFIREX_STATE_XSP_CONNECTED            ( 1 << 26 )    /* There is CrossfireX side port connection between GPUs */
# 定义 CrossfireX 状态标志位，表示启用 CrossfireX 需要系统重新启动
#define ADL_XFIREX_STATE_ENABLE_CF_REBOOT_REQUIRED    ( 1 << 27 )    /* System needs a reboot bofore enable CrossfireX */
# 定义 CrossfireX 状态标志位，表示禁用 CrossfireX 需要系统重新启动
#define ADL_XFIREX_STATE_DISABLE_CF_REBOOT_REQUIRED    ( 1 << 28 )    /* System needs a reboot after disable CrossfireX */
# 定义 CrossfireX 状态标志位，表示基础驱动程序处理降级密钥更新
#define ADL_XFIREX_STATE_DRV_HANDLE_DOWNGRADE_KEY    ( 1 << 29 )    /* Indicate base driver handles the downgrade key updating */
#define ADL_XFIREX_STATE_CF_RECONFIG_REQUIRED        ( 1 << 30 )    /* CrossfireX need to be reconfigured by CCC because of a LDA chain broken */
#define ADL_XFIREX_STATE_ERRORGETTINGSTATUS        ( 1 << 31 )    /* Could not obtain current status */
// @}

///////////////////////////////////////////////////////////////////////////
// ADL_DISPLAY_ADJUSTMENT_PIXELFORMAT adjustment values
// (bit-vector)
///////////////////////////////////////////////////////////////////////////
/// \defgroup define_pixel_formats Pixel Formats values
/// This group defines the various Pixel Formats that a particular digital display can support. \n
/// Since a display can support multiple formats, these values can be bit-or'ed to indicate the various formats \n
// @{
#define ADL_DISPLAY_PIXELFORMAT_UNKNOWN             0
#define ADL_DISPLAY_PIXELFORMAT_RGB                       (1 << 0)
#define ADL_DISPLAY_PIXELFORMAT_YCRCB444                  (1 << 1)    //Limited range
#define ADL_DISPLAY_PIXELFORMAT_YCRCB422                 (1 << 2)    //Limited range
#define ADL_DISPLAY_PIXELFORMAT_RGB_LIMITED_RANGE      (1 << 3)
#define ADL_DISPLAY_PIXELFORMAT_RGB_FULL_RANGE    ADL_DISPLAY_PIXELFORMAT_RGB  //Full range
#define ADL_DISPLAY_PIXELFORMAT_YCRCB420              (1 << 4)
// @}

/// \defgroup define_contype Connector Type Values
/// ADLDisplayConfig.ulConnectorType defines
// @{
#define ADL_DL_DISPLAYCONFIG_CONTYPE_UNKNOWN      0
#define ADL_DL_DISPLAYCONFIG_CONTYPE_CV_NONI2C_JP 1
#define ADL_DL_DISPLAYCONFIG_CONTYPE_CV_JPN       2
#define ADL_DL_DISPLAYCONFIG_CONTYPE_CV_NA        3
#define ADL_DL_DISPLAYCONFIG_CONTYPE_CV_NONI2C_NA 4
#define ADL_DL_DISPLAYCONFIG_CONTYPE_VGA          5
#define ADL_DL_DISPLAYCONFIG_CONTYPE_DVI_D        6
#define ADL_DL_DISPLAYCONFIG_CONTYPE_DVI_I        7
#define ADL_DL_DISPLAYCONFIG_CONTYPE_HDMI_TYPE_A  8
#define ADL_DL_DISPLAYCONFIG_CONTYPE_HDMI_TYPE_B  9
#define ADL_DL_DISPLAYCONFIG_CONTYPE_DISPLAYPORT  10
// @}
// 定义显示信息掩码的常量，用于 ADLDisplayInfo.iDisplayInfoMask 和 ADLDisplayInfo.iDisplayInfoValue
// (位向量)
///////////////////////////////////////////////////////////////////////////
/// \defgroup define_displayinfomask Display Info Mask Values
// @{
// 显示连接状态
#define ADL_DISPLAY_DISPLAYINFO_DISPLAYCONNECTED            0x00000001
// 显示映射状态
#define ADL_DISPLAY_DISPLAYINFO_DISPLAYMAPPED                0x00000002
// 非本地显示
#define ADL_DISPLAY_DISPLAYINFO_NONLOCAL                    0x00000004
// 强制支持
#define ADL_DISPLAY_DISPLAYINFO_FORCIBLESUPPORTED            0x00000008
// Genlock 支持
#define ADL_DISPLAY_DISPLAYINFO_GENLOCKSUPPORTED            0x00000010
// 多 GPU 支持
#define ADL_DISPLAY_DISPLAYINFO_MULTIVPU_SUPPORTED            0x00000020
// LDA 显示
#define ADL_DISPLAY_DISPLAYINFO_LDA_DISPLAY                    0x00000040
// 模式定时覆盖支持
#define ADL_DISPLAY_DISPLAYINFO_MODETIMING_OVERRIDESSUPPORTED            0x00000080

// 支持的显示方式：单显示器
#define ADL_DISPLAY_DISPLAYINFO_MANNER_SUPPORTED_SINGLE            0x00000100
// 支持的显示方式：克隆
#define ADL_DISPLAY_DISPLAYINFO_MANNER_SUPPORTED_CLONE            0x00000200

// XP 的传统支持
#define ADL_DISPLAY_DISPLAYINFO_MANNER_SUPPORTED_2VSTRETCH        0x00000400
#define ADL_DISPLAY_DISPLAYINFO_MANNER_SUPPORTED_2HSTRETCH        0x00000800
#define ADL_DISPLAY_DISPLAYINFO_MANNER_SUPPORTED_EXTENDED        0x00001000

// 更多支持方式
#define ADL_DISPLAY_DISPLAYINFO_MANNER_SUPPORTED_NSTRETCH1GPU    0x00010000
#define ADL_DISPLAY_DISPLAYINFO_MANNER_SUPPORTED_NSTRETCHNGPU    0x00020000
#define ADL_DISPLAY_DISPLAYINFO_MANNER_SUPPORTED_RESERVED2        0x00040000
#define ADL_DISPLAY_DISPLAYINFO_MANNER_SUPPORTED_RESERVED3        0x00080000

// 投影仪显示类型
#define ADL_DISPLAY_DISPLAYINFO_SHOWTYPE_PROJECTOR                0x00100000

// @}


///////////////////////////////////////////////////////////////////////////
// ADL_ADAPTER_DISPLAY_MANNER_SUPPORTED_ Definitions
// for ADLAdapterDisplayCap of ADL_Adapter_Display_Cap()
// 定义适配器方式支持数值的分组
///////////////////////////////////////////////////////////////////////////
/// \defgroup define_adaptermanner 适配器方式支持数值
// @{
// 适配器显示方式支持：未激活
#define ADL_ADAPTER_DISPLAYCAP_MANNER_SUPPORTED_NOTACTIVE        0x00000001
// 适配器显示方式支持：单显示器
#define ADL_ADAPTER_DISPLAYCAP_MANNER_SUPPORTED_SINGLE            0x00000002
// 适配器显示方式支持：克隆模式
#define ADL_ADAPTER_DISPLAYCAP_MANNER_SUPPORTED_CLONE            0x00000004
// 适配器显示方式支持：非跨 GPU 单显示器
#define ADL_ADAPTER_DISPLAYCAP_MANNER_SUPPORTED_NSTRETCH1GPU    0x00000008
// 适配器显示方式支持：跨 GPU 单显示器
#define ADL_ADAPTER_DISPLAYCAP_MANNER_SUPPORTED_NSTRETCHNGPU    0x00000010

/// 用于 XP 的旧版本支持
// 适配器显示方式支持：2 个垂直跨度
#define ADL_ADAPTER_DISPLAYCAP_MANNER_SUPPORTED_2VSTRETCH        0x00000020
// 适配器显示方式支持：2 个水平跨度
#define ADL_ADAPTER_DISPLAYCAP_MANNER_SUPPORTED_2HSTRETCH        0x00000040
// 适配器显示方式支持：扩展模式
#define ADL_ADAPTER_DISPLAYCAP_MANNER_SUPPORTED_EXTENDED        0x00000080

// 适配器显示方式支持：首选显示器支持
#define ADL_ADAPTER_DISPLAYCAP_PREFERDISPLAY_SUPPORTED            0x00000100
// 适配器显示方式支持：边框支持
#define ADL_ADAPTER_DISPLAYCAP_BEZEL_SUPPORTED                    0x00000200


///////////////////////////////////////////////////////////////////////////
// ADL_DISPLAY_DISPLAYMAP_MANNER_ 定义
// 用于 ADLDisplayMap.iDisplayMapMask 和 ADLDisplayMap.iDisplayMapValue
// (位向量)
///////////////////////////////////////////////////////////////////////////
// 显示器显示方式：保留
#define ADL_DISPLAY_DISPLAYMAP_MANNER_RESERVED            0x00000001
// 显示器显示方式：未激活
#define ADL_DISPLAY_DISPLAYMAP_MANNER_NOTACTIVE            0x00000002
// 显示器显示方式：单显示器
#define ADL_DISPLAY_DISPLAYMAP_MANNER_SINGLE            0x00000004
// 显示器显示方式：克隆模式
#define ADL_DISPLAY_DISPLAYMAP_MANNER_CLONE                0x00000008
// 显示器显示方式：保留1，已移除 NSTRETCH
#define ADL_DISPLAY_DISPLAYMAP_MANNER_RESERVED1            0x00000010  
// 显示器显示方式：水平跨度
#define ADL_DISPLAY_DISPLAYMAP_MANNER_HSTRETCH            0x00000020
// 显示器显示方式：垂直跨度
#define ADL_DISPLAY_DISPLAYMAP_MANNER_VSTRETCH            0x00000040
// 显示器显示方式：VLD
#define ADL_DISPLAY_DISPLAYMAP_MANNER_VLD                0x00000080

// @}

///////////////////////////////////////////////////////////////////////////
// ADL_DISPLAY_DISPLAYMAP_OPTION_ 定义
// 定义了函数 ADL_Display_DisplayMapConfig_Get 中的 iOption 参数的选项
// (位向量)
///////////////////////////////////////////////////////////////////////////
#define ADL_DISPLAY_DISPLAYMAP_OPTION_GPUINFO            0x00000001

///////////////////////////////////////////////////////////////////////////
// ADL_DISPLAY_DISPLAYTARGET_ 定义
// 用于 ADLDisplayTarget.iDisplayTargetMask 和 ADLDisplayTarget.iDisplayTargetValue
// (位向量)
///////////////////////////////////////////////////////////////////////////
#define ADL_DISPLAY_DISPLAYTARGET_PREFERRED            0x00000001

///////////////////////////////////////////////////////////////////////////
// ADL_DISPLAY_POSSIBLEMAPRESULT_VALID 定义
// 用于 ADLPossibleMapResult.iPossibleMapResultMask 和 ADLPossibleMapResult.iPossibleMapResultValue
// (位向量)
///////////////////////////////////////////////////////////////////////////
#define ADL_DISPLAY_POSSIBLEMAPRESULT_VALID                0x00000001
#define ADL_DISPLAY_POSSIBLEMAPRESULT_BEZELSUPPORTED    0x00000002
#define ADL_DISPLAY_POSSIBLEMAPRESULT_OVERLAPSUPPORTED    0x00000004

///////////////////////////////////////////////////////////////////////////
// ADL_DISPLAY_MODE_ 定义
// 用于 ADLMode.iModeMask, ADLMode.iModeValue, 和 ADLMode.iModeFlag
// (位向量)
///////////////////////////////////////////////////////////////////////////
/// \defgroup define_displaymode 显示模式数值
// @{
#define ADL_DISPLAY_MODE_COLOURFORMAT_565                0x00000001
#define ADL_DISPLAY_MODE_COLOURFORMAT_8888                0x00000002
#define ADL_DISPLAY_MODE_ORIENTATION_SUPPORTED_000        0x00000004
#define ADL_DISPLAY_MODE_ORIENTATION_SUPPORTED_090        0x00000008
#define ADL_DISPLAY_MODE_ORIENTATION_SUPPORTED_180        0x00000010
#define ADL_DISPLAY_MODE_ORIENTATION_SUPPORTED_270        0x00000020
#define ADL_DISPLAY_MODE_REFRESHRATE_ROUNDED            0x00000040
#define ADL_DISPLAY_MODE_REFRESHRATE_ONLY                0x00000080
// 定义了显示模式的标志位，分别表示渐进扫描和隔行扫描
#define ADL_DISPLAY_MODE_PROGRESSIVE_FLAG    0
#define ADL_DISPLAY_MODE_INTERLACED_FLAG    2
// @}

///////////////////////////////////////////////////////////////////////////
// ADL_OSMODEINFO Definitions
///////////////////////////////////////////////////////////////////////////
/// \defgroup define_osmode OS Mode Values
// @{
// 定义了操作系统模式的默认数值
#define ADL_OSMODEINFOXPOS_DEFAULT                -640
#define ADL_OSMODEINFOYPOS_DEFAULT                0
#define ADL_OSMODEINFOXRES_DEFAULT                640
#define ADL_OSMODEINFOYRES_DEFAULT                480
#define ADL_OSMODEINFOXRES_DEFAULT800            800
#define ADL_OSMODEINFOYRES_DEFAULT600            600
#define ADL_OSMODEINFOREFRESHRATE_DEFAULT        60
#define ADL_OSMODEINFOCOLOURDEPTH_DEFAULT        8
#define ADL_OSMODEINFOCOLOURDEPTH_DEFAULT16        16
#define ADL_OSMODEINFOCOLOURDEPTH_DEFAULT24        24
#define ADL_OSMODEINFOCOLOURDEPTH_DEFAULT32        32
#define ADL_OSMODEINFOORIENTATION_DEFAULT        0
#define ADL_OSMODEINFOORIENTATION_DEFAULT_WIN7    DISPLAYCONFIG_ROTATION_FORCE_UINT32
#define ADL_OSMODEFLAG_DEFAULT                    0
// @}

///////////////////////////////////////////////////////////////////////////
// ADLThreadingModel Enumeration
///////////////////////////////////////////////////////////////////////////
/// \defgroup thread_model
/// Used with \ref ADL_Main_ControlX2_Create and \ref ADL2_Main_ControlX2_Create to specify how ADL handles API calls when executed by multiple threads concurrently.
/// \brief Declares ADL threading behavior.
// @{
// 枚举了 ADL 的线程模型，用于指定 ADL 在多线程并发执行 API 调用时的处理方式
typedef enum ADLThreadingModel
{
    ADL_THREADING_UNLOCKED    = 0, /*!< 默认行为。ADL 不会强制对多线程执行的 ADL API 进行串行化。多个线程将被允许同时进入 ADL。注意，ADL 库不能保证是线程安全的。调用 ADL_Main_Control_Create 的客户端必须提供自己的机制来对 ADL 调用进行串行化。 */
    ADL_THREADING_LOCKED     /*!< ADL将在多线程调用时强制对ADL API进行串行化。一次只允许一个线程进入ADL API。此选项使ADL调用线程安全。如果ADL调用将在Linux上的x服务器渲染线程上执行，则不应使用此选项。它可能导致应用程序挂起。 */
// @}
// ADLThreadingModel 结构体定义结束

// ADLPurposeCode 枚举类型定义
enum ADLPurposeCode
{
    ADL_PURPOSECODE_NORMAL    = 0,  // 普通目的代码
    ADL_PURPOSECODE_HIDE_MODE_SWITCH,  // 隐藏模式切换目的代码
    ADL_PURPOSECODE_MODE_SWITCH,  // 模式切换目的代码
    ADL_PURPOSECODE_ATTATCH_DEVICE,  // 附加设备目的代码
    ADL_PURPOSECODE_DETACH_DEVICE,  // 分离设备目的代码
    ADL_PURPOSECODE_SETPRIMARY_DEVICE,  // 设置主设备目的代码
    ADL_PURPOSECODE_GDI_ROTATION,  // GDI 旋转目的代码
    ADL_PURPOSECODE_ATI_ROTATION  // ATI 旋转目的代码
};

// ADLAngle 枚举类型定义
enum ADLAngle
{
    ADL_ANGLE_LANDSCAPE = 0,  // 横向角度
    ADL_ANGLE_ROTATERIGHT = 90,  // 向右旋转角度
    ADL_ANGLE_ROTATE180 = 180,  // 180度旋转角度
    ADL_ANGLE_ROTATELEFT = 270,  // 向左旋转角度
};

// ADLOrientationDataType 枚举类型定义
enum ADLOrientationDataType
{
    ADL_ORIENTATIONTYPE_OSDATATYPE,  // OSD 数据类型
    ADL_ORIENTATIONTYPE_NONOSDATATYPE  // 非 OSD 数据类型
};

// ADLPanningMode 枚举类型定义
enum ADLPanningMode
{
    ADL_PANNINGMODE_NO_PANNING = 0,  // 无滚动模式
    ADL_PANNINGMODE_AT_LEAST_ONE_NO_PANNING = 1,  // 至少一个无滚动模式
    ADL_PANNINGMODE_ALLOW_PANNING = 2,  // 允许滚动模式
};

// ADLLARGEDESKTOPTYPE 枚举类型定义
enum ADLLARGEDESKTOPTYPE
{
    ADL_LARGEDESKTOPTYPE_NORMALDESKTOP = 0,  // 普通桌面类型
    ADL_LARGEDESKTOPTYPE_PSEUDOLARGEDESKTOP = 1,  // 伪大桌面类型
    ADL_LARGEDESKTOPTYPE_VERYLARGEDESKTOP = 2  // 非常大桌面类型
};

// ADLPlatform 枚举类型定义
// 定义枚举类型，表示图形平台为桌面或移动
enum ADLPlatForm
{
    GRAPHICS_PLATFORM_DESKTOP  = 0,
    GRAPHICS_PLATFORM_MOBILE   = 1
};

// 定义枚举类型，表示图形核心的世代
enum ADLGraphicCoreGeneration
{
    ADL_GRAPHIC_CORE_GENERATION_UNDEFINED                   = 0,
    ADL_GRAPHIC_CORE_GENERATION_PRE_GCN                     = 1,
    ADL_GRAPHIC_CORE_GENERATION_GCN                         = 2,
    ADL_GRAPHIC_CORE_GENERATION_RDNA                        = 3
};

// 内部使用的其他定义

// ADL_Display_WriteAndReadI2CRev_Get() 的返回值
#define ADL_I2C_MAJOR_API_REV           0x00000001
#define ADL_I2C_MINOR_DEFAULT_API_REV   0x00000000
#define ADL_I2C_MINOR_OEM_API_REV       0x00000001

// ADL_Display_WriteAndReadI2C() 的参数值
#define ADL_DL_I2C_LINE_OEM                0x00000001
#define ADL_DL_I2C_LINE_OD_CONTROL         0x00000002
#define ADL_DL_I2C_LINE_OEM2               0x00000003
#define ADL_DL_I2C_LINE_OEM3               0x00000004
#define ADL_DL_I2C_LINE_OEM4               0x00000005
#define ADL_DL_I2C_LINE_OEM5               0x00000006
#define ADL_DL_I2C_LINE_OEM6               0x00000007

// I2C 数据缓冲区的最大大小
#define ADL_DL_I2C_MAXDATASIZE             0x00000040
#define ADL_DL_I2C_MAXWRITEDATASIZE        0x0000000C
#define ADL_DL_I2C_MAXADDRESSLENGTH        0x00000006
#define ADL_DL_I2C_MAXOFFSETLENGTH         0x00000004

// ADLDisplayProperty.iPropertyType 的取值
#define ADL_DL_DISPLAYPROPERTY_TYPE_UNKNOWN              0
#define ADL_DL_DISPLAYPROPERTY_TYPE_EXPANSIONMODE        1
#define ADL_DL_DISPLAYPROPERTY_TYPE_USEUNDERSCANSCALING     2
#define ADL_DL_DISPLAYPROPERTY_TYPE_ITCFLAGENABLE        9
#define ADL_DL_DISPLAYPROPERTY_TYPE_DOWNSCALE            11
// 定义整数缩放类型常量
#define ADL_DL_DISPLAYPROPERTY_TYPE_INTEGER_SCALING      12


/// ADLDisplayContent.iContentType 的取值
/// 某些支持 ITC 的 HDMI 面板支持一种功能，即可以根据内容类型调整面板上显示的内容，以优化视图。
/// 取决于显示的内容类型，面板上的显示可以进行调整。
#define ADL_DL_DISPLAYCONTENT_TYPE_GRAPHICS        1
#define ADL_DL_DISPLAYCONTENT_TYPE_PHOTO        2
#define ADL_DL_DISPLAYCONTENT_TYPE_CINEMA        4
#define ADL_DL_DISPLAYCONTENT_TYPE_GAME            8



// ADLDisplayProperty.iExpansionMode 的取值
#define ADL_DL_DISPLAYPROPERTY_EXPANSIONMODE_CENTER        0
#define ADL_DL_DISPLAYPROPERTY_EXPANSIONMODE_FULLSCREEN    1
#define ADL_DL_DISPLAYPROPERTY_EXPANSIONMODE_ASPECTRATIO   2


///\defgroup define_dither_states 抖动选项
// @{
/// 禁用抖动
#define ADL_DL_DISPLAY_DITHER_DISABLED              0
/// 使用默认驱动程序设置进行抖动。请注意，默认设置可能是禁用抖动。
#define ADL_DL_DISPLAY_DITHER_DRIVER_DEFAULT        1
/// 临时抖动到 6 位/像素。请注意，如果输入是 12 位，最低有效位将被截断。
#define ADL_DL_DISPLAY_DITHER_FM6                   2
/// 临时抖动到 8 位/像素。
#define ADL_DL_DISPLAY_DITHER_FM8                   3
/// 临时抖动到 10 位/像素。
#define ADL_DL_DISPLAY_DITHER_FM10                  4
/// 空间抖动到 6 位/像素。请注意，如果输入是 12 位，最低有效位将被截断。
#define ADL_DL_DISPLAY_DITHER_DITH6                 5
/// 空间抖动到 8 位/像素。
#define ADL_DL_DISPLAY_DITHER_DITH8                 6
/// 空间抖动到 10 位/像素。
#define ADL_DL_DISPLAY_DITHER_DITH10                7
/// 将空间抖动到6位每像素。每帧重置一次随机数生成器，因此某个像素的相同输入值将始终被抖动到相同的输出值。注意，如果输入是12位，则两个最低有效位将被截断。
#define ADL_DL_DISPLAY_DITHER_DITH6_NO_FRAME_RAND   8
/// 将空间抖动到8位每像素。每帧重置一次随机数生成器，因此某个像素的相同输入值将始终被抖动到相同的输出值。
#define ADL_DL_DISPLAY_DITHER_DITH8_NO_FRAME_RAND   9
/// 将空间抖动到10位每像素。每帧重置一次随机数生成器，因此某个像素的相同输入值将始终被抖动到相同的输出值。
#define ADL_DL_DISPLAY_DITHER_DITH10_NO_FRAME_RAND  10
/// 截断到6位每像素。
#define ADL_DL_DISPLAY_DITHER_TRUN6                 11
/// 截断到8位每像素。
#define ADL_DL_DISPLAY_DITHER_TRUN8                 12
/// 截断到10位每像素。
#define ADL_DL_DISPLAY_DITHER_TRUN10                13
/// 截断到10位每像素，然后进行空间抖动到8位每像素。
#define ADL_DL_DISPLAY_DITHER_TRUN10_DITH8          14
/// 截断到10位每像素，然后进行空间抖动到6位每像素。
#define ADL_DL_DISPLAY_DITHER_TRUN10_DITH6          15
/// 截断到10位每像素，然后进行时间抖动到8位每像素。
#define ADL_DL_DISPLAY_DITHER_TRUN10_FM8            16
/// 截断到10位每像素，然后进行时间抖动到6位每像素。
#define ADL_DL_DISPLAY_DITHER_TRUN10_FM6            17
/// 截断到10位每像素，然后进行空间抖动到8位每像素，再进行时间抖动到6位每像素。
#define ADL_DL_DISPLAY_DITHER_TRUN10_DITH8_FM6      18
/// 将空间抖动到10位每像素，然后进行时间抖动到8位每像素。
#define ADL_DL_DISPLAY_DITHER_DITH10_FM8            19
/// 将空间抖动到10位每像素，然后进行时间抖动到6位每像素。
#define ADL_DL_DISPLAY_DITHER_DITH10_FM6            20
/// 截断到8位每像素，然后进行空间抖动到6位每像素。
// 定义显示抖动类型，将8位色彩截断为6位色彩，然后进行时间抖动
#define ADL_DL_DISPLAY_DITHER_TRUN8_DITH6           21
// 定义显示抖动类型，将8位色彩截断为6位色彩，然后进行时间抖动，使用空间抖动到8位色彩
#define ADL_DL_DISPLAY_DITHER_TRUN8_FM6             22
// 定义显示抖动类型，使用空间抖动到8位色彩，然后进行时间抖动到6位色彩
#define ADL_DL_DISPLAY_DITHER_DITH8_FM6             23
// 最后一个显示抖动类型的值
#define ADL_DL_DISPLAY_DITHER_LAST                  ADL_DL_DISPLAY_DITHER_DITH8_FM6
// @}


// 显示获取缓存的EDID标志
#define ADL_MAX_EDIDDATA_SIZE              256 // UCHAR的数量
#define ADL_MAX_OVERRIDEEDID_SIZE          512 // UCHAR的数量
#define ADL_MAX_EDID_EXTENSION_BLOCKS      3

// 控制器叠加透明度类型
#define ADL_DL_CONTROLLER_OVERLAY_ALPHA         0
#define ADL_DL_CONTROLLER_OVERLAY_ALPHAPERPIX   1

// 显示数据包信息重置、设置和扫描标志
#define ADL_DL_DISPLAY_DATA_PACKET__INFO_PACKET_RESET      0x00000000
#define ADL_DL_DISPLAY_DATA_PACKET__INFO_PACKET_SET        0x00000001
#define ADL_DL_DISPLAY_DATA_PACKET__INFO_PACKET_SCAN       0x00000002

///\defgroup define_display_packet 显示数据包类型
// @{
#define ADL_DL_DISPLAY_DATA_PACKET__TYPE__AVI              0x00000001
#define ADL_DL_DISPLAY_DATA_PACKET__TYPE__GAMMUT           0x00000002
#define ADL_DL_DISPLAY_DATA_PACKET__TYPE__VENDORINFO       0x00000004
#define ADL_DL_DISPLAY_DATA_PACKET__TYPE__HDR              0x00000008
#define ADL_DL_DISPLAY_DATA_PACKET__TYPE__SPD              0x00000010
// @}

// 矩阵类型
#define ADL_GAMUT_MATRIX_SD         1   // SD矩阵，即BT601
#define ADL_GAMUT_MATRIX_HD         2   // HD矩阵，即BT709

///\defgroup define_clockinfo_flags 时钟标志
/// 由ADLAdapterODClockInfo.iFlag使用
// @{
#define ADL_DL_CLOCKINFO_FLAG_FULLSCREEN3DONLY         0x00000001
#define ADL_DL_CLOCKINFO_FLAG_ALWAYSFULLSCREEN3D       0x00000002
#define ADL_DL_CLOCKINFO_FLAG_VPURECOVERYREDUCED       0x00000004
#define ADL_DL_CLOCKINFO_FLAG_THERMALPROTECTION        0x00000008
// @}

// 支持的GPU
// ADL_Display_PowerXpressActiveGPU_Get()
#define ADL_DL_POWERXPRESS_GPU_INTEGRATED        1
// 定义离散 GPU 的标识符
#define ADL_DL_POWERXPRESS_GPU_DISCRETE            2

// ADL_Display_PowerXpressActiveGPU_Get() 的 lpOperationResult 可能的取值
#define ADL_DL_POWERXPRESS_SWITCH_RESULT_STARTED         1 // 切换过程已启动 - 仅适用于 Windows 平台
#define ADL_DL_POWERXPRESS_SWITCH_RESULT_DECLINED        2 // 无法启动切换过程 - 所有平台
#define ADL_DL_POWERXPRESS_SWITCH_RESULT_ALREADY         3 // 系统已经具有所需状态 - 所有平台
#define ADL_DL_POWERXPRESS_SWITCH_RESULT_DEFERRED        5  // 切换被推迟，需要 X 重新启动 - 仅适用于 Linux 平台

// PowerXpress 支持版本
#define ADL_DL_POWERXPRESS_VERSION_MAJOR            2    // 当前 PowerXpress 支持版本 2.0
#define ADL_DL_POWERXPRESS_VERSION_MINOR            0

#define ADL_DL_POWERXPRESS_VERSION    (((ADL_DL_POWERXPRESS_VERSION_MAJOR) << 16) | ADL_DL_POWERXPRESS_VERSION_MINOR)

// ADLThermalControllerInfo.iThermalControllerDomain 的取值
#define ADL_DL_THERMAL_DOMAIN_OTHER      0
#define ADL_DL_THERMAL_DOMAIN_GPU        1

// ADLThermalControllerInfo.iFlags 的取值
#define ADL_DL_THERMAL_FLAG_INTERRUPT    1
#define ADL_DL_THERMAL_FLAG_FANCONTROL   2

///\defgroup define_fanctrl 风扇速度控制
/// ADLFanSpeedInfo.iFlags 的取值
// @{
#define ADL_DL_FANCTRL_SUPPORTS_PERCENT_READ     1
#define ADL_DL_FANCTRL_SUPPORTS_PERCENT_WRITE    2
#define ADL_DL_FANCTRL_SUPPORTS_RPM_READ         4
#define ADL_DL_FANCTRL_SUPPORTS_RPM_WRITE        8
// @}

// ADLFanSpeedValue.iSpeedType 的取值
#define ADL_DL_FANCTRL_SPEED_TYPE_PERCENT    1
#define ADL_DL_FANCTRL_SPEED_TYPE_RPM        2

// ADLFanSpeedValue.iFlags 的取值
#define ADL_DL_FANCTRL_FLAG_USER_DEFINED_SPEED   1

// MVPU 接口
#define ADL_DL_MAX_MVPU_ADAPTERS   4
#define MVPU_ADAPTER_0          0x00000001
#define MVPU_ADAPTER_1          0x00000002
#define MVPU_ADAPTER_2          0x00000004
#define MVPU_ADAPTER_3          0x00000008
// 定义注册表路径的最大长度为256
#define ADL_DL_MAX_REGISTRY_PATH   256

// ADLMVPUStatus.iStatus的取值
#define ADL_DL_MVPU_STATUS_OFF   0
#define ADL_DL_MVPU_STATUS_ON    1

// ASIC家族的取值
///\defgroup define_Asic_type 详细的ASIC类型
/// 适配器ASIC家族类型的定义
// @{
#define ADL_ASIC_UNDEFINED    0
#define ADL_ASIC_DISCRETE    (1 << 0)
#define ADL_ASIC_INTEGRATED    (1 << 1)
#define ADL_ASIC_FIREGL        (1 << 2)
#define ADL_ASIC_FIREMV        (1 << 3)
#define ADL_ASIC_XGP        (1 << 4)
#define ADL_ASIC_FUSION        (1 << 5)
#define ADL_ASIC_FIRESTREAM (1 << 6)
#define ADL_ASIC_EMBEDDED   (1 << 7)
// @}

///\defgroup define_detailed_timing_flags 详细的定时标志
/// ADLDetailedTiming.sTimingFlags字段的定义
// @{
#define ADL_DL_TIMINGFLAG_DOUBLE_SCAN              0x0001
// 当模式为INTERLACED时设置sTimingFlags，如果不是PROGRESSIVE
#define ADL_DL_TIMINGFLAG_INTERLACED               0x0002
// 当水平同步为POSITIVE时设置sTimingFlags，如果不是NEGATIVE
#define ADL_DL_TIMINGFLAG_H_SYNC_POLARITY          0x0004
// 当垂直同步为POSITIVE时设置sTimingFlags，如果不是NEGATIVE
#define ADL_DL_TIMINGFLAG_V_SYNC_POLARITY          0x0008
// @}

///\defgroup define_modetiming_standard 定时标准
/// ADLDisplayModeInfo.iTimingStandard字段的定义
// @{
#define ADL_DL_MODETIMING_STANDARD_CVT             0x00000001 // CVT标准
#define ADL_DL_MODETIMING_STANDARD_GTF             0x00000002 // GFT标准
#define ADL_DL_MODETIMING_STANDARD_DMT             0x00000004 // DMT标准
#define ADL_DL_MODETIMING_STANDARD_CUSTOM          0x00000008 // 用户定义的标准
#define ADL_DL_MODETIMING_STANDARD_DRIVER_DEFAULT  0x00000010 // 从覆盖列表中移除模式
#define ADL_DL_MODETIMING_STANDARD_CVT_RB           0x00000020 // CVT-RB标准
// @}

// \defgroup define_xserverinfo 驱动程序x服务器信息
/// 这些标志由ADL_XServerInfo_Get()使用
// @
// 定义 Xinerama 在 x 服务器中是否激活的标志位
#define ADL_XSERVERINFO_XINERAMAACTIVE            (1<<0)

// 定义驱动是否支持 RandR 1.2 的标志位
#define ADL_XSERVERINFO_RANDR12SUPPORTED          (1<<1)

// Eyefinity 定义常量的分组
///\defgroup define_eyefinity_constants Eyefinity Definitions
// @{

// 控制器向量 0 的定义
#define ADL_CONTROLLERVECTOR_0        1    // ADL_CONTROLLERINDEX_0 = 0, (1 << ADL_CONTROLLERINDEX_0)

// 控制器向量 1 的定义
#define ADL_CONTROLLERVECTOR_1        2    // ADL_CONTROLLERINDEX_1 = 1, (1 << ADL_CONTROLLERINDEX_1)

// SLS 网格方向的定义
#define ADL_DISPLAY_SLSGRID_ORIENTATION_000        0x00000001
#define ADL_DISPLAY_SLSGRID_ORIENTATION_090        0x00000002
#define ADL_DISPLAY_SLSGRID_ORIENTATION_180        0x00000004
#define ADL_DISPLAY_SLSGRID_ORIENTATION_270        0x00000008

// SLS 网格选项的定义
#define ADL_DISPLAY_SLSGRID_CAP_OPTION_RELATIVETO_LANDSCAPE     0x00000001
#define ADL_DISPLAY_SLSGRID_CAP_OPTION_RELATIVETO_CURRENTANGLE     0x00000002
#define ADL_DISPLAY_SLSGRID_PORTAIT_MODE                         0x00000004
#define ADL_DISPLAY_SLSGRID_KEEPTARGETROTATION                  0x00000080

// SLS 网格支持的定义
#define ADL_DISPLAY_SLSGRID_SAMEMODESLS_SUPPORT        0x00000010
#define ADL_DISPLAY_SLSGRID_MIXMODESLS_SUPPORT        0x00000020
#define ADL_DISPLAY_SLSGRID_DISPLAYROTATION_SUPPORT    0x00000040
#define ADL_DISPLAY_SLSGRID_DESKTOPROTATION_SUPPORT    0x00000080

// SLS 映射布局模式的定义
#define ADL_DISPLAY_SLSMAP_SLSLAYOUTMODE_FIT        0x0100
#define ADL_DISPLAY_SLSMAP_SLSLAYOUTMODE_FILL       0x0200
#define ADL_DISPLAY_SLSMAP_SLSLAYOUTMODE_EXPAND     0x0400

// SLS 映射类型的定义
#define ADL_DISPLAY_SLSMAP_IS_SLS        0x1000
#define ADL_DISPLAY_SLSMAP_IS_SLSBUILDER 0x2000
#define ADL_DISPLAY_SLSMAP_IS_CLONEVT     0x4000

// SLS 映射配置选项的定义
#define ADL_DISPLAY_SLSMAPCONFIG_GET_OPTION_RELATIVETO_LANDSCAPE         0x00000001
#define ADL_DISPLAY_SLSMAPCONFIG_GET_OPTION_RELATIVETO_CURRENTANGLE     0x00000002
# 定义创建 SLS 映射配置选项，相对于横向模式
#define ADL_DISPLAY_SLSMAPCONFIG_CREATE_OPTION_RELATIVETO_LANDSCAPE         0x00000001
# 定义创建 SLS 映射配置选项，相对于当前角度
#define ADL_DISPLAY_SLSMAPCONFIG_CREATE_OPTION_RELATIVETO_CURRENTANGLE     0x00000002

# 定义重新排列 SLS 映射配置选项，相对于横向模式
#define ADL_DISPLAY_SLSMAPCONFIG_REARRANGE_OPTION_RELATIVETO_LANDSCAPE     0x00000001
# 定义重新排列 SLS 映射配置选项，相对于当前角度
#define ADL_DISPLAY_SLSMAPCONFIG_REARRANGE_OPTION_RELATIVETO_CURRENTANGLE     0x00000002

# 定义 SLS 支持相同模式
#define ADL_SLS_SAMEMODESLS_SUPPORT         0x0001
# 定义 SLS 支持混合模式
#define ADL_SLS_MIXMODESLS_SUPPORT          0x0002
# 定义 SLS 支持显示旋转
#define ADL_SLS_DISPLAYROTATIONSLS_SUPPORT  0x0004
# 定义 SLS 支持桌面旋转
#define ADL_SLS_DESKTOPROTATIONSLS_SUPPORT  0x0008

# 定义 SLS 目标无效
#define ADL_SLS_TARGETS_INVALID     0x0001
# 定义 SLS 模式无效
#define ADL_SLS_MODES_INVALID       0x0002
# 定义 SLS 旋转无效
#define ADL_SLS_ROTATIONS_INVALID   0x0004
# 定义 SLS 位置无效
#define ADL_SLS_POSITIONS_INVALID   0x0008
# 定义 SLS 布局模式无效
#define ADL_SLS_LAYOUTMODE_INVALID  0x0010

# 定义 SLS 显示偏移有效
#define ADL_DISPLAY_SLSDISPLAYOFFSET_VALID        0x0002

# 定义 SLS 网格相对于横向模式
#define ADL_DISPLAY_SLSGRID_RELATIVETO_LANDSCAPE         0x00000010
# 定义 SLS 网格相对于当前角度
#define ADL_DISPLAY_SLSGRID_RELATIVETO_CURRENTANGLE     0x00000020

# 标识显示当前处于贝塞尔模式
#define ADL_DISPLAY_SLSMAP_BEZELMODE            0x00000010
# 标识显示从此映射中排列
#define ADL_DISPLAY_SLSMAP_DISPLAYARRANGED        0x00000002
# 标识此映射当前用于当前适配器
#define ADL_DISPLAY_SLSMAP_CURRENTCONFIG        0x00000004

# 仅用于激活 SLS 映射信息
#define ADL_DISPLAY_SLSMAPINDEXLIST_OPTION_ACTIVE        0x00000001

# 用于贝塞尔
#define ADL_DISPLAY_BEZELOFFSET_STEPBYSTEPSET            0x00000004
# 提交贝塞尔偏移
#define ADL_DISPLAY_BEZELOFFSET_COMMIT                    0x00000008

# 定义 SLS 图像裁剪类型
typedef enum _SLS_ImageCropType {
    Fit = 1,
    Fill = 2,
    Expand = 3
}SLS_ImageCropType;

# 定义 DCE 设置类型
typedef enum _DceSettingsType {
    DceSetting_HdmiLq,
    DceSetting_DpSettings,
    DceSetting_Protection
} DceSettingsType;

# 定义 DP 链路速率
typedef enum _DpLinkRate {
    DPLinkRate_Unknown,
    DPLinkRate_RBR,
    DPLinkRate_HBR,
    DPLinkRate_HBR2,
    DPLinkRate_HBR3
} DpLinkRate
///\defgroup define_powerxpress_constants PowerXpress Definitions
/// @{

/// 定义了 PX 功能的各种常量，用于 ADLPXConfigCaps.iPXConfigCapMask 和 ADLPXConfigCaps.iPXConfigCapValue 的位掩码
#define    ADL_PX_CONFIGCAPS_SPLASHSCREEN_SUPPORT        0x0001
#define    ADL_PX_CONFIGCAPS_CF_SUPPORT                0x0002
#define    ADL_PX_CONFIGCAPS_MUXLESS                    0x0004
#define    ADL_PX_CONFIGCAPS_PROFILE_COMPLIANT            0x0008
#define    ADL_PX_CONFIGCAPS_NON_AMD_DRIVEN_DISPLAYS    0x0010
#define ADL_PX_CONFIGCAPS_FIXED_SUPPORT             0x0020
#define ADL_PX_CONFIGCAPS_DYNAMIC_SUPPORT           0x0040
#define ADL_PX_CONFIGCAPS_HIDE_AUTO_SWITCH            0x0080

/// 用于 ADLPXSchemeRange 的位掩码，用于标识 PX 方案
#define ADL_PX_SCHEMEMASK_FIXED                        0x0001
#define ADL_PX_SCHEMEMASK_DYNAMIC                    0x0002

/// PX 方案
typedef enum _ADLPXScheme
{
    ADL_PX_SCHEME_INVALID   = 0, // 无效的 PX 方案
    ADL_PX_SCHEME_FIXED     = ADL_PX_SCHEMEMASK_FIXED, // 固定的 PX 方案
    ADL_PX_SCHEME_DYNAMIC   = ADL_PX_SCHEMEMASK_DYNAMIC // 动态的 PX 方案
}ADLPXScheme;

/// 为了兼容性而保留的旧定义，以后需要删除
typedef enum PXScheme
{
    PX_SCHEME_INVALID   = 0, // 无效的 PX 方案
    PX_SCHEME_FIXED     = 1, // 固定的 PX 方案
    PX_SCHEME_DYNAMIC   = 2  // 动态的 PX 方案
} PXScheme;


/// @}

///\defgroup define_appprofiles For Application Profiles
/// @{

#define ADL_APP_PROFILE_FILENAME_LENGTH        256 // 应用程序配置文件名的最大长度
#define ADL_APP_PROFILE_TIMESTAMP_LENGTH    32 // 应用程序配置文件时间戳的长度
#define ADL_APP_PROFILE_VERSION_LENGTH        32 // 应用程序配置文件版本号的长度
#define ADL_APP_PROFILE_PROPERTY_LENGTH        64 // 应用程序配置文件属性的最大长度

enum ApplicationListType
{
    ADL_PX40_MRU, // 最近使用的应用程序列表类型
    ADL_PX40_MISSED, // 未使用的应用程序列表类型
    ADL_PX40_DISCRETE, // 离散显卡应用程序列表类型
    ADL_PX40_INTEGRATED, // 集成显卡应用程序列表类型
    ADL_MMD_PROFILED, // MMD 配置文件应用程序列表类型
    ADL_PX40_TOTAL // 总应用程序列表类型
};

typedef enum _ADLProfilePropertyType
{
    ADL_PROFILEPROPERTY_TYPE_BINARY        = 0, // 二进制类型的配置文件属性
    ADL_PROFILEPROPERTY_TYPE_BOOLEAN, // 布尔类型的配置文件属性
    ADL_PROFILEPROPERTY_TYPE_DWORD, // 双字类型的配置文件属性
    ADL_PROFILEPROPERTY_TYPE_QWORD, // 四字类型的配置文件属性
    ADL_PROFILEPROPERTY_TYPE_ENUMERATED, // 枚举类型的配置文件属性
    ADL_PROFILEPROPERTY_TYPE_STRING // 字符串类型的配置文件属性
}ADLProfilePropertyType;
/// @}

///\defgroup define_dp12 For Display Port 1.2
/// @{

/// Maximum Relative Address Link
#define ADL_MAX_RAD_LINK_COUNT    15

/// @}

///\defgroup defines_gamutspace Driver Supported Gamut Space
/// @{

/// The flags desribes that gamut is related to source or to destination and to overlay or to graphics
#define ADL_GAMUT_REFERENCE_SOURCE       (1 << 0)  // 表示色域与源或目标相关，以及叠加层或图形相关的标志位
#define ADL_GAMUT_GAMUT_VIDEO_CONTENT    (1 << 1)  // 表示色域与视频内容相关的标志位

/// The flags are used to describe the source of gamut and how read information from struct ADLGamutData
#define ADL_CUSTOM_WHITE_POINT           (1 << 0)  // 描述色域来源和如何从结构ADLGamutData中读取信息的标志位
#define ADL_CUSTOM_GAMUT                 (1 << 1)  // 描述色域来源和如何从结构ADLGamutData中读取信息的标志位
#define ADL_GAMUT_REMAP_ONLY             (1 << 2)  // 仅重新映射色域的标志位

/// The define means the predefined gamut values  .
///Driver uses to find entry in the table and apply appropriate gamut space.
#define ADL_GAMUT_SPACE_CCIR_709     (1 << 0)  // 预定义色域值，驱动程序使用它在表中查找条目并应用适当的色域空间
#define ADL_GAMUT_SPACE_CCIR_601     (1 << 1)  // 预定义色域值，驱动程序使用它在表中查找条目并应用适当的色域空间
#define ADL_GAMUT_SPACE_ADOBE_RGB    (1 << 2)  // 预定义色域值，驱动程序使用它在表中查找条目并应用适当的色域空间
#define ADL_GAMUT_SPACE_CIE_RGB      (1 << 3)  // 预定义色域值，驱动程序使用它在表中查找条目并应用适当的色域空间
#define ADL_GAMUT_SPACE_CUSTOM       (1 << 4)  // 预定义色域值，驱动程序使用它在表中查找条目并应用适当的色域空间
#define ADL_GAMUT_SPACE_CCIR_2020    (1 << 5)  // 预定义色域值，驱动程序使用它在表中查找条目并应用适当的色域空间
#define ADL_GAMUT_SPACE_APPCTRL      (1 << 6)  // 预定义色域值，驱动程序使用它在表中查找条目并应用适当的色域空间

/// Predefine white point values are structed similar to gamut .
#define ADL_WHITE_POINT_5000K       (1 << 0)  // 预定义白点值，结构类似于色域
#define ADL_WHITE_POINT_6500K       (1 << 1)  // 预定义白点值，结构类似于色域
#define ADL_WHITE_POINT_7500K       (1 << 2)  // 预定义白点值，结构类似于色域
#define ADL_WHITE_POINT_9300K       (1 << 3)  // 预定义白点值，结构类似于色域
#define ADL_WHITE_POINT_CUSTOM      (1 << 4)  // 预定义白点值，结构类似于色域

///gamut and white point coordinates are from 0.0 -1.0 and divider is used to find the real value .
/// X float = X int /divider
#define ADL_GAMUT_WHITEPOINT_DIVIDER           10000  // 色域和白点坐标范围为0.0-1.0，使用除数来找到实际值

///gamma a0 coefficient uses the following divider:
#define ADL_REGAMMA_COEFFICIENT_A0_DIVIDER       10000000  // gamma a0系数使用以下除数

///gamma a1 ,a2,a3 coefficients use the following divider:
#define ADL_REGAMMA_COEFFICIENT_A1A2A3_DIVIDER   1000  // gamma a1、a2、a3系数使用以下除数

///describes whether the coefficients are from EDID or custom user values.
#define ADL_EDID_REGAMMA_COEFFICIENTS          (1 << 0)  // 描述系数是来自EDID还是自定义用户值的标志位
///Used for struct ADLRegamma. Feature if set use gamma ramp, if missing use regamma coefficents
#define ADL_USE_GAMMA_RAMP                     (1 << 4)
///Used for struct ADLRegamma. If the gamma ramp flag is used then the driver could apply de gamma corretion to the supplied curve and this depends on this flag
#define ADL_APPLY_DEGAMMA                      (1 << 5)
///specifies that standard SRGB gamma should be applied
#define ADL_EDID_REGAMMA_PREDEFINED_SRGB       (1 << 1)
///specifies that PQ gamma curve should be applied
#define ADL_EDID_REGAMMA_PREDEFINED_PQ         (1 << 2)
///specifies that PQ gamma curve should be applied, lower max nits
#define ADL_EDID_REGAMMA_PREDEFINED_PQ_2084_INTERIM (1 << 3)
///specifies that 3.6 gamma should be applied
#define ADL_EDID_REGAMMA_PREDEFINED_36         (1 << 6)
///specifies that BT709 gama should be applied
#define ADL_EDID_REGAMMA_PREDEFINED_BT709      (1 << 7)
///specifies that regamma should be disabled, and application controls regamma content (of the whole screen)
#define ADL_EDID_REGAMMA_PREDEFINED_APPCTRL    (1 << 8)

/// @}

/// \defgroup define_ddcinfo_pixelformats DDCInfo Pixel Formats
/// @{
/// defines for iPanelPixelFormat  in struct ADLDDCInfo2
#define ADL_DISPLAY_DDCINFO_PIXEL_FORMAT_RGB656                       0x00000001L
#define ADL_DISPLAY_DDCINFO_PIXEL_FORMAT_RGB666                       0x00000002L
#define ADL_DISPLAY_DDCINFO_PIXEL_FORMAT_RGB888                       0x00000004L
#define ADL_DISPLAY_DDCINFO_PIXEL_FORMAT_RGB101010                    0x00000008L
#define ADL_DISPLAY_DDCINFO_PIXEL_FORMAT_RGB161616                    0x00000010L
#define ADL_DISPLAY_DDCINFO_PIXEL_FORMAT_RGB_RESERVED1                0x00000020L
#define ADL_DISPLAY_DDCINFO_PIXEL_FORMAT_RGB_RESERVED2                0x00000040L
#define ADL_DISPLAY_DDCINFO_PIXEL_FORMAT_RGB_RESERVED3                0x00000080L
#define ADL_DISPLAY_DDCINFO_PIXEL_FORMAT_XRGB_BIAS101010              0x00000100L
# 定义不同的像素格式，用于表示不同的色彩空间和位深度
#define ADL_DISPLAY_DDCINFO_PIXEL_FORMAT_YCBCR444_8BPCC               0x00000200L
#define ADL_DISPLAY_DDCINFO_PIXEL_FORMAT_YCBCR444_10BPCC              0x00000400L
#define ADL_DISPLAY_DDCINFO_PIXEL_FORMAT_YCBCR444_12BPCC              0x00000800L
#define ADL_DISPLAY_DDCINFO_PIXEL_FORMAT_YCBCR422_8BPCC               0x00001000L
#define ADL_DISPLAY_DDCINFO_PIXEL_FORMAT_YCBCR422_10BPCC              0x00002000L
#define ADL_DISPLAY_DDCINFO_PIXEL_FORMAT_YCBCR422_12BPCC              0x00004000L
#define ADL_DISPLAY_DDCINFO_PIXEL_FORMAT_YCBCR420_8BPCC               0x00008000L
#define ADL_DISPLAY_DDCINFO_PIXEL_FORMAT_YCBCR420_10BPCC              0x00010000L
#define ADL_DISPLAY_DDCINFO_PIXEL_FORMAT_YCBCR420_12BPCC              0x00020000L
/// @}

/// \defgroup define_source_content_TF ADLSourceContentAttributes transfer functions (gamma)
/// @{
# 定义不同的传输函数，用于表示不同的色彩空间和伽马值
/// defines for iTransferFunction in ADLSourceContentAttributes
#define ADL_TF_sRGB                0x0001      ///< sRGB
#define ADL_TF_BT709            0x0002      ///< BT.709
#define ADL_TF_PQ2084            0x0004      ///< PQ2084
#define ADL_TF_PQ2084_INTERIM    0x0008        ///< PQ2084-Interim
#define ADL_TF_LINEAR_0_1        0x0010      ///< Linear 0 - 1
#define ADL_TF_LINEAR_0_125        0x0020      ///< Linear 0 - 125
#define ADL_TF_DOLBYVISION        0x0040      ///< DolbyVision
#define ADL_TF_GAMMA_22         0x0080      ///< Plain 2.2 gamma curve
/// @}

/// \defgroup define_source_content_CS ADLSourceContentAttributes color spaces
/// @{
# 定义不同的色彩空间，用于表示不同的色彩空间
/// defines for iColorSpace in ADLSourceContentAttributes
#define ADL_CS_sRGB                0x0001      ///< sRGB
#define ADL_CS_BT601             0x0002      ///< BT.601
#define ADL_CS_BT709            0x0004      ///< BT.709
#define ADL_CS_BT2020            0x0008      ///< BT.2020
#define ADL_CS_ADOBE            0x0010      ///< Adobe RGB
#define ADL_CS_P3                0x0020      ///< DCI-P3
#define ADL_CS_scRGB_MS_REF        0x0040      ///< scRGB (MS Reference)
#define ADL_CS_DISPLAY_NATIVE    0x0080      ///< Display Native
#define ADL_CS_APP_CONTROL         0x0100      ///< Application Controlled
#define ADL_CS_DOLBYVISION      0x0200      ///< DolbyVision
/// @}

/// \defgroup define_HDR_support ADLDDCInfo2 HDR support options
/// @{
/// defines for iSupportedHDR in ADLDDCInfo2
#define ADL_HDR_CEA861_3        0x0001      ///< HDR10/CEA861.3 HDR supported
#define ADL_HDR_DOLBYVISION        0x0002      ///< DolbyVision HDR supported
#define ADL_HDR_FREESYNC_HDR    0x0004      ///< FreeSync HDR supported
/// @}

/// \defgroup define_FreesyncFlags ADLDDCInfo2 Freesync HDR flags
/// @{
/// defines for iFreesyncFlags in ADLDDCInfo2
#define ADL_HDR_FREESYNC_BACKLIGHT_SUPPORT           0x0001      ///< Global backlight control supported
#define ADL_HDR_FREESYNC_LOCAL_DIMMING               0x0002      ///< Local dimming supported
/// @}

/// \defgroup define_source_content_flags ADLSourceContentAttributes flags
/// @{
/// defines for iFlags in ADLSourceContentAttributes
#define ADL_SCA_LOCAL_DIMMING_DISABLE    0x0001      ///< Disable local dimming
/// @}

/// \defgroup define_dbd_state Deep Bit Depth
/// @{

/// defines for ADL_Workstation_DeepBitDepth_Get and  ADL_Workstation_DeepBitDepth_Set functions
// This value indicates that the deep bit depth state is forced off
#define ADL_DEEPBITDEPTH_FORCEOFF     0
/// This value indicates that the deep bit depth state  is set to auto, the driver will automatically enable the
/// appropriate deep bit depth state depending on what connected display supports.
#define ADL_DEEPBITDEPTH_10BPP_AUTO     1
/// This value indicates that the deep bit depth state  is forced on to 10 bits per pixel, this is regardless if the display
/// supports 10 bpp.
#define ADL_DEEPBITDEPTH_10BPP_FORCEON     2

/// defines for ADLAdapterConfigMemory of ADL_Adapter_ConfigMemory_Get
/// If this bit is set, it indicates that the Deep Bit Depth pixel is set on the display
# 定义显示适配器配置内存的位掩码，表示显示是否旋转（90、180或270度）
#define ADL_ADAPTER_CONFIGMEMORY_DBD            (1 << 0)
# 如果设置了这个位，表示显示已经旋转
#define ADL_ADAPTER_CONFIGMEMORY_ROTATE            (1 << 1)
# 如果设置了这个位，表示显示上设置了被动立体视效
#define ADL_ADAPTER_CONFIGMEMORY_STEREO_PASSIVE    (1 << 2)
# 如果设置了这个位，表示显示上设置了主动立体视效
#define ADL_ADAPTER_CONFIGMEMORY_STEREO_ACTIVE    (1 << 3)
# 如果设置了这个位，表示显示上设置了无撕裂垂直同步
#define ADL_ADAPTER_CONFIGMEMORY_ENHANCEDVSYNC    (1 << 4)
#define ADL_ADAPTER_CONFIGMEMORY_TEARFREEVSYNC    (1 << 4)
# 定义内存类型的位掩码，用于ADLMemoryRequired结构
# 表示这是可见内存
#define ADL_MEMORYREQTYPE_VISIBLE                (1 << 0)
# 表示这是不可见内存
#define ADL_MEMORYREQTYPE_INVISIBLE                (1 << 1)
# 表示每个GPU应保留的用于所有其他分配的可见内存量
#define ADL_MEMORYREQTYPE_GPURESERVEDVISIBLE    (1 << 2)
# 定义适配器撕裂免费状态，用于指示撕裂免费桌面状态
# 撕裂免费桌面已启用
#define ADL_ADAPTER_TEAR_FREE_ON                1
# 由于缺乏图形适配器内存，无法启用撕裂免费桌面
#define ADL_ADAPTER_TEAR_FREE_NOTENOUGHMEM        -1
# 由于启用了四重缓冲立体视效，无法启用撕裂免费桌面
#define ADL_ADAPTER_TEAR_FREE_OFF_ERR_QUADBUFFERSTEREO    -2
# 由于启用了MGPU-SLS，无法启用撕裂免费桌面
#define ADL_ADAPTER_TEAR_FREE_OFF_ERR_MGPUSLD    -3
# 撕裂免费桌面已禁用
#define ADL_ADAPTER_TEAR_FREE_OFF                0
/// \defgroup define_adapter_crossdisplay_platforminfo
/// Used in ADL_Adapter_CrossDisplayPlatformInfo_Get function to indicate the Crossdisplay platform info.
/// @{
/// CROSSDISPLAY platform.
#define ADL_CROSSDISPLAY_PLATFORM                    (1 << 0)
/// CROSSDISPLAY platform for Lasso station.
#define ADL_CROSSDISPLAY_PLATFORM_LASSO                (1 << 1)
/// CROSSDISPLAY platform for docking station.
#define ADL_CROSSDISPLAY_PLATFORM_DOCKSTATION        (1 << 2)
/// @}

/// \defgroup define_adapter_crossdisplay_option
/// Used in ADL_Adapter_CrossdisplayInfoX2_Set function to indicate cross display options.
/// @{
/// Checking if 3D application is runnning. If yes, not to do switch, return ADL_OK_WAIT; otherwise do switch.
#define ADL_CROSSDISPLAY_OPTION_NONE            0
/// Force switching without checking for running 3D applications
#define ADL_CROSSDISPLAY_OPTION_FORCESWITCH        (1 << 0)
/// @}

/// \defgroup define_adapter_states Adapter Capabilities
/// These defines the capabilities supported by an adapter. It is used by \ref ADL_Adapter_ConfigureState_Get
/// @{
/// Indicates that the adapter is headless (i.e. no displays can be connected to it)
#define ADL_ADAPTERCONFIGSTATE_HEADLESS ( 1 << 2 )
/// Indicates that the adapter is configured to define the main rendering capabilities. For example, adapters
/// in Crossfire(TM) configuration, this bit would only be set on the adapter driving the display(s).
#define ADL_ADAPTERCONFIGSTATE_REQUISITE_RENDER ( 1 << 0 )
/// Indicates that the adapter is configured to be used to unload some of the rendering work for a particular
/// requisite rendering adapter. For eample, for adapters in a Crossfire configuration, this bit would be set
/// on all adapters that are currently not driving the display(s)
#define ADL_ADAPTERCONFIGSTATE_ANCILLARY_RENDER ( 1 << 1 )
/// Indicates that scatter gather feature enabled on the adapter
#define ADL_ADAPTERCONFIGSTATE_SCATTERGATHER ( 1 << 4 )
/// @}
/// \defgroup define_controllermode_ulModifiers
/// 这些定义了 set viewport 支持的详细操作。它被 \ref ADL_Display_ViewPort_Set 使用
/// @{
/// 表示视口设置将改变视图位置
#define ADL_CONTROLLERMODE_CM_MODIFIER_VIEW_POSITION       0x00000001
/// 表示视口设置将改变视图 PanLock
#define ADL_CONTROLLERMODE_CM_MODIFIER_VIEW_PANLOCK        0x00000002
/// 表示视口设置将改变视图大小
#define ADL_CONTROLLERMODE_CM_MODIFIER_VIEW_SIZE           0x00000008
/// @}

/// \defgroup defines for Mirabilis
/// 这些定义用于 Mirabilis 功能
/// @{
///
/// 表示音频采样率的最大数量
#define ADL_MAX_AUDIO_SAMPLE_RATE_COUNT                    16
/// @}

///////////////////////////////////////////////////////////////////////////
// ADLMultiChannelSplitStateFlag 枚举
///////////////////////////////////////////////////////////////////////////
enum ADLMultiChannelSplitStateFlag
{
    ADLMultiChannelSplit_Unitialized = 0,
    ADLMultiChannelSplit_Disabled    = 1,
    ADLMultiChannelSplit_Enabled     = 2,
    ADLMultiChannelSplit_SaveProfile = 3
};

///////////////////////////////////////////////////////////////////////////
// ADLSampleRate 枚举
///////////////////////////////////////////////////////////////////////////
enum ADLSampleRate
{
    ADLSampleRate_32KHz =0,
    ADLSampleRate_44P1KHz,
    ADLSampleRate_48KHz,
    ADLSampleRate_88P2KHz,
    ADLSampleRate_96KHz,
    ADLSampleRate_176P4KHz,
    ADLSampleRate_192KHz,
    ADLSampleRate_384KHz, //DP1.2
    ADLSampleRate_768KHz, //DP1.2
    ADLSampleRate_Undefined
};

/// \defgroup define_overdrive6_capabilities
/// 这些定义了 Overdrive 6 支持的功能。它被 \ref ADL_Overdrive6_Capabilities_Get 使用
// @{
/// 表示核心（引擎）时钟可以被改变
#define ADL_OD6_CAPABILITY_SCLK_CUSTOMIZATION               0x00000001
// 定义内存时钟可以被更改的标志
#define ADL_OD6_CAPABILITY_MCLK_CUSTOMIZATION               0x00000002
// 表示支持图形活动报告
#define ADL_OD6_CAPABILITY_GPU_ACTIVITY_MONITOR             0x00000004
// 表示可以自定义功率限制
#define ADL_OD6_CAPABILITY_POWER_CONTROL                    0x00000008
// 表示支持 SVI2 电压控制
#define ADL_OD6_CAPABILITY_VOLTAGE_CONTROL                  0x00000010
// 表示支持 OD6+ 百分比调整
#define ADL_OD6_CAPABILITY_PERCENT_ADJUSTMENT               0x00000020
// 表示支持热限解锁
#define ADL_OD6_CAPABILITY_THERMAL_LIMIT_UNLOCK             0x00000040
// 表示风扇速度需要以 RPM 显示
#define ADL_OD6_CAPABILITY_FANSPEED_IN_RPM                    0x00000080
// 定义 Overdrive 6 支持的电源状态
#define ADL_OD6_SUPPORTEDSTATE_PERFORMANCE                  0x00000001
// 保留字段，未来使用
#define ADL_OD6_SUPPORTEDSTATE_POWER_SAVING                 0x00000002
// 定义 Overdrive 6 获取状态信息的电源状态
#define ADL_OD6_GETSTATEINFO_DEFAULT_PERFORMANCE            0x00000001
// 保留字段，未来使用
#define ADL_OD6_GETSTATEINFO_DEFAULT_POWER_SAVING           0x00000002
// 获取当前状态的时钟。当前仅支持性能状态，因此与 ADL_OD6_GETSTATEINFO_CUSTOM_PERFORMANCE 相同
#define ADL_OD6_GETSTATEINFO_CURRENT                        0x00000003
/// Get the modified clocks (if any) for the performance state.  If clocks were not modified
/// through Overdrive 6, then this will return the same clocks as \ref ADL_OD6_GETSTATEINFO_DEFAULT_PERFORMANCE.
#define ADL_OD6_GETSTATEINFO_CUSTOM_PERFORMANCE             0x00000004
/// Do not use.  Reserved for future use.
#define ADL_OD6_GETSTATEINFO_CUSTOM_POWER_SAVING            0x00000005
// @}

/// \defgroup define_overdrive6_getstate and define_overdrive6_getmaxclockadjust
/// These defines the power states to get information about. It is used by \ref ADL_Overdrive6_StateEx_Get and \ref ADL_Overdrive6_MaxClockAdjust_Get
// @{
/// Get default clocks for the performance state.  Only performance state is currently supported.
#define ADL_OD6_STATE_PERFORMANCE            0x00000001
// @}

/// \defgroup define_overdrive6_setstate
/// These define which power state to set customized clocks on. It is used by \ref ADL_Overdrive6_State_Set
// @{
/// Set customized clocks for the performance state.
#define ADL_OD6_SETSTATE_PERFORMANCE                        0x00000001
/// Do not use.  Reserved for future use.
#define ADL_OD6_SETSTATE_POWER_SAVING                       0x00000002
// @}

/// \defgroup define_overdrive6_thermalcontroller_caps
/// These defines the capabilities of the GPU thermal controller. It is used by \ref ADL_Overdrive6_ThermalController_Caps
// @{
/// GPU thermal controller is supported.
#define ADL_OD6_TCCAPS_THERMAL_CONTROLLER                   0x00000001
/// GPU fan speed control is supported.
#define ADL_OD6_TCCAPS_FANSPEED_CONTROL                     0x00000002
/// Fan speed percentage can be read.
#define ADL_OD6_TCCAPS_FANSPEED_PERCENT_READ                0x00000100
/// Fan speed can be set by specifying a percentage value.
#define ADL_OD6_TCCAPS_FANSPEED_PERCENT_WRITE               0x00000200
/// Fan speed RPM (revolutions-per-minute) can be read.
#define ADL_OD6_TCCAPS_FANSPEED_RPM_READ                    0x00000400
/// Fan speed can be set by specifying an RPM value.
#define ADL_OD6_TCCAPS_FANSPEED_RPM_WRITE                   0x00000800
// @}

/// \defgroup define_overdrive6_fanspeed_type
/// These defines the fan speed type being reported. It is used by \ref ADL_Overdrive6_FanSpeed_Get
// @{
/// Fan speed reported in percentage.
#define ADL_OD6_FANSPEED_TYPE_PERCENT                       0x00000001
/// Fan speed reported in RPM.
#define ADL_OD6_FANSPEED_TYPE_RPM                           0x00000002
/// Fan speed has been customized by the user, and fan is not running in automatic mode.
#define ADL_OD6_FANSPEED_USER_DEFINED                       0x00000100
// @}

/// \defgroup define_overdrive_EventCounter_type
/// These defines the EventCounter type being reported. It is used by \ref ADL2_OverdriveN_CountOfEvents_Get ,can be used on older OD version supported ASICs also.
// @{
#define ADL_ODN_EVENTCOUNTER_THERMAL        0
#define ADL_ODN_EVENTCOUNTER_VPURECOVERY    1
// @}

///////////////////////////////////////////////////////////////////////////
// ADLODNControlType Enumeration
///////////////////////////////////////////////////////////////////////////
enum ADLODNControlType
{
    ODNControlType_Current = 0,  // 当前控制类型
    ODNControlType_Default,       // 默认控制类型
    ODNControlType_Auto,          // 自动控制类型
    ODNControlType_Manual          // 手动控制类型
};

enum ADLODNDPMMaskType
{
     ADL_ODN_DPM_CLOCK               = 1 << 0,  // DPM 时钟控制位
     ADL_ODN_DPM_VDDC                = 1 << 1,  // DPM 电压控制位
     ADL_ODN_DPM_MASK                = 1 << 2,  // DPM 控制位掩码
};

//ODN features Bits for ADLODNCapabilitiesX2
enum ADLODNFeatureControl
{
    // 定义各种特性的位掩码，用于表示各种功能的开启与关闭
    ADL_ODN_SCLK_DPM                = 1 << 0,
    ADL_ODN_MCLK_DPM                = 1 << 1,
    ADL_ODN_SCLK_VDD                = 1 << 2,
    ADL_ODN_MCLK_VDD                = 1 << 3,
    ADL_ODN_FAN_SPEED_MIN           = 1 << 4,
    ADL_ODN_FAN_SPEED_TARGET        = 1 << 5,
    ADL_ODN_ACOUSTIC_LIMIT_SCLK     = 1 << 6,
    ADL_ODN_TEMPERATURE_FAN_MAX     = 1 << 7,
    ADL_ODN_TEMPERATURE_SYSTEM      = 1 << 8,
    ADL_ODN_POWER_LIMIT             = 1 << 9,
    ADL_ODN_SCLK_AUTO_LIMIT         = 1 << 10,
    ADL_ODN_MCLK_AUTO_LIMIT         = 1 << 11,
    ADL_ODN_SCLK_DPM_MASK_ENABLE    = 1 << 12,
    ADL_ODN_MCLK_DPM_MASK_ENABLE    = 1 << 13,
    ADL_ODN_MCLK_UNDERCLOCK_ENABLE  = 1 << 14,
    ADL_ODN_SCLK_DPM_THROTTLE_NOTIFY= 1 << 15,
    ADL_ODN_POWER_UTILIZATION       = 1 << 16,
    ADL_ODN_PERF_TUNING_SLIDER      = 1 << 17,
    ADL_ODN_REMOVE_WATTMAN_PAGE     = 1 << 31 // Internal Only
};

// 如果添加了新功能，PPLIB 只需要添加扩展功能 ID 和项目 ID（设置 ID）。这些 ID 应与 CWDDEPM.h 中定义的驱动程序匹配
enum ADLODNExtFeatureControl
{
    ADL_ODN_EXT_FEATURE_MEMORY_TIMING_TUNE = 1 << 0,
    ADL_ODN_EXT_FEATURE_FAN_ZERO_RPM_CONTROL = 1 << 1,
    ADL_ODN_EXT_FEATURE_AUTO_UV_ENGINE = 1 << 2,   // 自动降压
    ADL_ODN_EXT_FEATURE_AUTO_OC_ENGINE = 1 << 3,   // 自动超频引擎
    ADL_ODN_EXT_FEATURE_AUTO_OC_MEMORY = 1 << 4,   // 自动超频内存
    ADL_ODN_EXT_FEATURE_FAN_CURVE = 1 << 5    // 风扇曲线
};

// 如果添加了新功能，PPLIB 只需要添加扩展功能 ID 和项目 ID（设置 ID）。这些 ID 应与 CWDDEPM.h 中定义的驱动程序匹配
enum ADLODNExtSettingId
{
    ADL_ODN_PARAMETER_AC_TIMING = 0,
    ADL_ODN_PARAMETER_FAN_ZERO_RPM_CONTROL,
    ADL_ODN_PARAMETER_AUTO_UV_ENGINE,
    ADL_ODN_PARAMETER_AUTO_OC_ENGINE,
    ADL_ODN_PARAMETER_AUTO_OC_MEMORY,
    ADL_ODN_PARAMETER_FAN_CURVE_TEMPERATURE_1,
    ADL_ODN_PARAMETER_FAN_CURVE_SPEED_1,
    # 定义 ADL_ODN_PARAMETER_FAN_CURVE_TEMPERATURE_2 常量
    ADL_ODN_PARAMETER_FAN_CURVE_TEMPERATURE_2,
    # 定义 ADL_ODN_PARAMETER_FAN_CURVE_SPEED_2 常量
    ADL_ODN_PARAMETER_FAN_CURVE_SPEED_2,
    # 定义 ADL_ODN_PARAMETER_FAN_CURVE_TEMPERATURE_3 常量
    ADL_ODN_PARAMETER_FAN_CURVE_TEMPERATURE_3,
    # 定义 ADL_ODN_PARAMETER_FAN_CURVE_SPEED_3 常量
    ADL_ODN_PARAMETER_FAN_CURVE_SPEED_3,
    # 定义 ADL_ODN_PARAMETER_FAN_CURVE_TEMPERATURE_4 常量
    ADL_ODN_PARAMETER_FAN_CURVE_TEMPERATURE_4,
    # 定义 ADL_ODN_PARAMETER_FAN_CURVE_SPEED_4 常量
    ADL_ODN_PARAMETER_FAN_CURVE_SPEED_4,
    # 定义 ADL_ODN_PARAMETER_FAN_CURVE_TEMPERATURE_5 常量
    ADL_ODN_PARAMETER_FAN_CURVE_TEMPERATURE_5,
    # 定义 ADL_ODN_PARAMETER_FAN_CURVE_SPEED_5 常量
    ADL_ODN_PARAMETER_FAN_CURVE_SPEED_5,
    # 定义 ADL_ODN_POWERGAUGE 常量
    ADL_ODN_POWERGAUGE,
    # 定义 ODN_COUNT 常量
    ODN_COUNT
// 定义了一个枚举类型，表示OD8功能的控制位
enum ADLOD8FeatureControl
{
    ADL_OD8_GFXCLK_LIMITS = 1 << 0,  // 图形时钟频率限制
    ADL_OD8_GFXCLK_CURVE = 1 << 1,  // 图形时钟曲线
    ADL_OD8_UCLK_MAX = 1 << 2,  // 内存时钟最大频率
    ADL_OD8_POWER_LIMIT = 1 << 3,  // 电源限制
    ADL_OD8_ACOUSTIC_LIMIT_SCLK = 1 << 4,   // 声学限制时钟
    ADL_OD8_FAN_SPEED_MIN = 1 << 5,   // 风扇最小转速
    ADL_OD8_TEMPERATURE_FAN = 1 << 6,   // 风扇目标温度
    ADL_OD8_TEMPERATURE_SYSTEM = 1 << 7,    // 系统最高温度
    ADL_OD8_MEMORY_TIMING_TUNE = 1 << 8,  // 内存时序调整
    ADL_OD8_FAN_ZERO_RPM_CONTROL = 1 << 9 ,  // 零转速控制
    ADL_OD8_AUTO_UV_ENGINE = 1 << 10,  // 自动欠压引擎
    ADL_OD8_AUTO_OC_ENGINE = 1 << 11,  // 自动超频引擎
    ADL_OD8_AUTO_OC_MEMORY = 1 << 12,  // 自动超频内存
    ADL_OD8_FAN_CURVE = 1 << 13,   // 风扇曲线
    ADL_OD8_WS_AUTO_FAN_ACOUSTIC_LIMIT = 1 << 14, // 工作站手动风扇控制
    ADL_OD8_POWER_GAUGE = 1 << 15 // 电源计量
};

// 定义了另一个枚举类型，表示OD8设置的ID
typedef enum _ADLOD8SettingId
{
    OD8_GFXCLK_FMAX = 0,  // 图形时钟最大频率
    OD8_GFXCLK_FMIN,  // 图形时钟最小频率
    OD8_GFXCLK_FREQ1,  // 图形时钟频率1
    OD8_GFXCLK_VOLTAGE1,  // 图形时钟电压1
    OD8_GFXCLK_FREQ2,  // 图形时钟频率2
    OD8_GFXCLK_VOLTAGE2,  // 图形时钟电压2
    OD8_GFXCLK_FREQ3,  // 图形时钟频率3
    OD8_GFXCLK_VOLTAGE3,  // 图形时钟电压3
    OD8_UCLK_FMAX,  // 内存时钟最大频率
    OD8_POWER_PERCENTAGE,  // 电源百分比
    OD8_FAN_MIN_SPEED,  // 风扇最小速度
    OD8_FAN_ACOUSTIC_LIMIT,  // 风扇声学限制
    OD8_FAN_TARGET_TEMP,  // 风扇目标温度
    OD8_OPERATING_TEMP_MAX,  // 最高操作温度
    OD8_AC_TIMING,  // AC时序
    OD8_FAN_ZERORPM_CONTROL,  // 零转速控制
    OD8_AUTO_UV_ENGINE_CONTROL,  // 自动欠压引擎控制
    OD8_AUTO_OC_ENGINE_CONTROL,  // 自动超频引擎控制
    OD8_AUTO_OC_MEMORY_CONTROL,  // 自动超频内存控制
    OD8_FAN_CURVE_TEMPERATURE_1,  // 风扇曲线温度1
    OD8_FAN_CURVE_SPEED_1,  // 风扇曲线速度1
    OD8_FAN_CURVE_TEMPERATURE_2,  // 风扇曲线温度2
    OD8_FAN_CURVE_SPEED_2,  // 风扇曲线速度2
    OD8_FAN_CURVE_TEMPERATURE_3,  // 风扇曲线温度3
    OD8_FAN_CURVE_SPEED_3,  // 风扇曲线速度3
    OD8_FAN_CURVE_TEMPERATURE_4,  // 风扇曲线温度4
    OD8_FAN_CURVE_SPEED_4,  // 风扇曲线速度4
    OD8_FAN_CURVE_TEMPERATURE_5,  // 风扇曲线温度5
    OD8_FAN_CURVE_SPEED_5,  // 风扇曲线速度5
    OD8_WS_FAN_AUTO_FAN_ACOUSTIC_LIMIT,  // 工作站自动风扇声学限制
    OD8_POWER_GAUGE, // 电源计量
    OD8_COUNT  // 设置ID数量
} ADLOD8SettingId;

// 定义了性能指标日志的最大传感器数量
#define ADL_PMLOG_MAX_SENSORS  256
# 定义枚举类型 ADLSensorType，包含各种传感器类型
typedef enum _ADLSensorType
{
    SENSOR_MAXTYPES = 0,  # 传感器类型的最大数量
    PMLOG_CLK_GFXCLK = 1,  # 图形时钟频率
    PMLOG_CLK_MEMCLK = 2,  # 内存时钟频率
    PMLOG_CLK_SOCCLK = 3,  # SOC 时钟频率
    ...
    PMLOG_SMART_POWERSHIFT_DGPU = 39,  # 智能动态功耗转移（dGPU）
    PMLOG_MAX_SENSORS_REAL  # 真实传感器类型的最大数量
} ADLSensorType;

# 定义枚举类型 ADL_THROTTLE_NOTIFICATION，包含各种节流状态通知
//Throttle Status
typedef enum _ADL_THROTTLE_NOTIFICATION
{
    ADL_PMLOG_THROTTLE_POWER = 1 << 0,  # 功率节流通知
    ADL_PMLOG_THROTTLE_THERMAL = 1 << 1,  # 热量节流通知
    ADL_PMLOG_THROTTLE_CURRENT = 1 << 2,  # 电流节流通知
} ADL_THROTTLE_NOTIFICATION;

# 定义枚举类型 ADL_PMLOG_SENSORS，包含各种 PMLOG 传感器类型
typedef enum _ADL_PMLOG_SENSORS
{
    ADL_SENSOR_MAXTYPES = 0,  # 传感器类型的最大数量
    ADL_PMLOG_CLK_GFXCLK = 1,  # 图形时钟频率
    ADL_PMLOG_CLK_MEMCLK = 2,  # 内存时钟频率
    ADL_PMLOG_CLK_SOCCLK = 3,  # SOC 时钟频率
    ADL_PMLOG_CLK_UVDCLK1 = 4,  # UVD 时钟频率1
    ADL_PMLOG_CLK_UVDCLK2 = 5,  # UVD 时钟频率2
    ADL_PMLOG_CLK_VCECLK = 6,  # VCE 时钟频率
    ADL_PMLOG_CLK_VCNCLK = 7,  # VCN 时钟频率
    ADL_PMLOG_TEMPERATURE_EDGE = 8,  # 边缘温度
    ADL_PMLOG_TEMPERATURE_MEM = 9,  # 内存温度
    ADL_PMLOG_TEMPERATURE_VRVDDC = 10,  # VRVDDC 温度
    ADL_PMLOG_TEMPERATURE_VRMVDD = 11,  # VRMVDD 温度
    ...
} ADL_PMLOG_SENSORS;
    # 定义了一系列的常量，用于记录不同类型的性能日志数据
    ADL_PMLOG_TEMPERATURE_LIQUID = 12,  # 液体温度
    ADL_PMLOG_TEMPERATURE_PLX = 13,  # PLX 温度
    ADL_PMLOG_FAN_RPM = 14,  # 风扇转速
    ADL_PMLOG_FAN_PERCENTAGE = 15,  # 风扇百分比
    ADL_PMLOG_SOC_VOLTAGE = 16,  # SOC 电压
    ADL_PMLOG_SOC_POWER = 17,  # SOC 功率
    ADL_PMLOG_SOC_CURRENT = 18,  # SOC 电流
    ADL_PMLOG_INFO_ACTIVITY_GFX = 19,  # 图形处理器活动信息
    ADL_PMLOG_INFO_ACTIVITY_MEM = 20,  # 内存活动信息
    ADL_PMLOG_GFX_VOLTAGE = 21,  # 图形处理器电压
    ADL_PMLOG_MEM_VOLTAGE = 22,  # 内存电压
    ADL_PMLOG_ASIC_POWER = 23,  # ASIC 功率
    ADL_PMLOG_TEMPERATURE_VRSOC = 24,  # VRSOC 温度
    ADL_PMLOG_TEMPERATURE_VRMVDD0 = 25,  # VRMVDD0 温度
    ADL_PMLOG_TEMPERATURE_VRMVDD1 = 26,  # VRMVDD1 温度
    ADL_PMLOG_TEMPERATURE_HOTSPOT = 27,  # 热点温度
    ADL_PMLOG_TEMPERATURE_GFX = 28,  # 图形处理器温度
    ADL_PMLOG_TEMPERATURE_SOC = 29,  # SOC 温度
    ADL_PMLOG_GFX_POWER = 30,  # 图形处理器功率
    ADL_PMLOG_GFX_CURRENT = 31,  # 图形处理器电流
    ADL_PMLOG_TEMPERATURE_CPU = 32,  # CPU 温度
    ADL_PMLOG_CPU_POWER = 33,  # CPU 功率
    ADL_PMLOG_CLK_CPUCLK = 34,  # CPU 时钟频率
    ADL_PMLOG_THROTTLER_STATUS = 35,  # 节流器状态
    ADL_PMLOG_CLK_VCN1CLK1 = 36,  # VCN1CLK1 时钟频率
    ADL_PMLOG_CLK_VCN1CLK2 = 37,  # VCN1CLK2 时钟频率
    ADL_PMLOG_SMART_POWERSHIFT_CPU = 38,  # 智能动态功耗转移 CPU
    ADL_PMLOG_SMART_POWERSHIFT_DGPU = 39  # 智能动态功耗转移离散图形处理器
// 定义了传感器的数据结构 ADL_PMLOG_SENSORS
} ADL_PMLOG_SENSORS;

/// \defgroup define_ecc_mode_states
/// 这些定义了 ECC（错误校正码）状态。它被 \ref ADL_Workstation_ECC_Get,ADL_Workstation_ECC_Set 使用
// @{
/// 错误校正关闭
#define ECC_MODE_OFF 0
/// 错误校正为 ECCV2
#define ECC_MODE_ON 2
/// 错误校正为 HBM
#define ECC_MODE_HBM 3
// @}

/// \defgroup define_board_layout_flags
/// 这些定义了板卡布局标志状态，指示了 \ref ADLBoardLayoutInfo 的有效属性。它被 \ref ADL_Adapter_BoardLayout_Get 使用
// @{
/// 表示槽位数量有效
#define ADL_BLAYOUT_VALID_NUMBER_OF_SLOTS 0x1
/// 表示槽位大小有效。槽位大小包括长度和宽度
#define ADL_BLAYOUT_VALID_SLOT_SIZES 0x2
/// 表示连接器偏移量有效
#define ADL_BLAYOUT_VALID_CONNECTOR_OFFSETS 0x4
/// 表示连接器长度有效
#define ADL_BLAYOUT_VALID_CONNECTOR_LENGTHS 0x8
// @}

/// \defgroup define_max_constants
/// 这些定义了最大值常量
// @{
/// 表示板卡上支持的最大槽位数
#define ADL_ADAPTER_MAX_SLOTS 4
/// 表示槽位上支持的最大连接器数
#define ADL_ADAPTER_MAX_CONNECTORS 10
/// 表示连接属性的最大支持数
#define ADL_MAX_CONNECTION_TYPES 32
/// 表示最大相对地址链接计数
#define ADL_MAX_RELATIVE_ADDRESS_LINK_COUNT 15
/// 表示 EDID 数据块的最大大小
#define ADL_MAX_DISPLAY_EDID_DATA_SIZE 1024
/// 表示错误记录的最大数量
#define ADL_MAX_ERROR_RECORDS_COUNT  256
/// 表示支持的最大电源状态数
#define ADL_MAX_POWER_POLICY    6
// @}

/// \defgroup define_connection_types
/// 这些定义了连接类型常量，指示给定连接器的有效连接类型。它被 \ref ADL_Adapter_SupportedConnections_Get 使用
// @{
/// 表示 VGA 连接类型有效。
#define ADL_CONNECTION_TYPE_VGA 0
/// 表示 DVI_I 连接类型有效。
#define ADL_CONNECTION_TYPE_DVI 1
/// 表示 DVI_SL 连接类型有效。
#define ADL_CONNECTION_TYPE_DVI_SL 2
/// 表示 HDMI 连接类型有效。
#define ADL_CONNECTION_TYPE_HDMI 3
/// 表示显示端口连接类型有效。
#define ADL_CONNECTION_TYPE_DISPLAY_PORT 4
/// 表示主动转接器 DP->DVI（单链接）连接类型有效。
#define ADL_CONNECTION_TYPE_ACTIVE_DONGLE_DP_DVI_SL 5
/// 表示主动转接器 DP->DVI（双链接）连接类型有效。
#define ADL_CONNECTION_TYPE_ACTIVE_DONGLE_DP_DVI_DL 6
/// 表示主动转接器 DP->HDMI 连接类型有效。
#define ADL_CONNECTION_TYPE_ACTIVE_DONGLE_DP_HDMI 7
/// 表示主动转接器 DP->VGA 连接类型有效。
#define ADL_CONNECTION_TYPE_ACTIVE_DONGLE_DP_VGA 8
/// 表示被动转接器 DP->HDMI 连接类型有效。
#define ADL_CONNECTION_TYPE_PASSIVE_DONGLE_DP_HDMI 9
/// 表示被动转接器 DP->VGA 连接类型有效。
#define ADL_CONNECTION_TYPE_PASSIVE_DONGLE_DP_DVI 10
/// 表示 MST 类型有效。
#define ADL_CONNECTION_TYPE_MST 11
/// 表示主动转接器，所有类型。
#define ADL_CONNECTION_TYPE_ACTIVE_DONGLE 12
/// 表示虚拟连接类型。
#define ADL_CONNECTION_TYPE_VIRTUAL 13
/// 用于从索引生成位掩码的宏。
#define ADL_CONNECTION_BITMAST_FROM_INDEX(index) (1 << index)
// @}

/// \defgroup define_connection_properties
/// 这些定义是连接属性，指示给定连接类型的有效属性。它被 \ref ADL_Adapter_SupportedConnections_Get 使用。
// @{
/// 表示属性比特率有效。
#define ADL_CONNECTION_PROPERTY_BITRATE 0x1
/// 表示属性通道数有效。
#define ADL_CONNECTION_PROPERTY_NUMBER_OF_LANES 0x2
/// Indicates the property 3D caps is valid.
#define ADL_CONNECTION_PROPERTY_3DCAPS  0x4
/// Indicates the property output bandwidth is valid.
#define ADL_CONNECTION_PROPERTY_OUTPUT_BANDWIDTH 0x8
/// Indicates the property colordepth is valid.
#define ADL_CONNECTION_PROPERTY_COLORDEPTH  0x10
// @}

/// \defgroup define_lanecount_constants
/// These defines are the Lane count constants which will be used in DP & etc.
// @{
/// Indicates if lane count is unknown
#define ADL_LANECOUNT_UNKNOWN 0
/// Indicates if lane count is 1
#define ADL_LANECOUNT_ONE 1
/// Indicates if lane count is 2
#define ADL_LANECOUNT_TWO 2
/// Indicates if lane count is 4
#define ADL_LANECOUNT_FOUR 4
/// Indicates if lane count is 8
#define ADL_LANECOUNT_EIGHT 8
/// Indicates default value of lane count
#define ADL_LANECOUNT_DEF ADL_LANECOUNT_FOUR
// @}

/// \defgroup define_linkrate_constants
/// These defines are the link rate constants which will be used in DP & etc.
// @{
/// Indicates if link rate is unknown
#define ADL_LINK_BITRATE_UNKNOWN 0
/// Indicates if link rate is 1.62Ghz
#define ADL_LINK_BITRATE_1_62_GHZ 0x06
/// Indicates if link rate is 2.7Ghz
#define ADL_LINK_BITRATE_2_7_GHZ 0x0A
/// Indicates if link rate is 5.4Ghz
#define ADL_LINK_BITRATE_5_4_GHZ 0x14

/// Indicates if link rate is 8.1Ghz
#define ADL_LINK_BITRATE_8_1_GHZ 0x1E
/// Indicates default value of link rate
#define ADL_LINK_BITRATE_DEF ADL_LINK_BITRATE_2_7_GHZ
// @}

/// \defgroup define_colordepth_constants
/// These defines are the color depth constants which will be used in DP & etc.
// @{
#define ADL_CONNPROP_S3D_ALTERNATE_TO_FRAME_PACK            0x00000001
// @}


/// \defgroup define_colordepth_constants
/// These defines are the color depth constants which will be used in DP & etc.
// @{
/// Indicates if color depth is unknown
#define ADL_COLORDEPTH_UNKNOWN 0
/// Indicates if color depth is 666
#define ADL_COLORDEPTH_666 1
/// Indicates if color depth is 888
#define ADL_COLORDEPTH_888 2
/// Indicates if color depth is 101010
#define ADL_COLORDEPTH_101010 3
/// Indicates if color depth is 121212
#define ADL_COLORDEPTH_121212 4
/// Indicates if color depth is 141414
#define ADL_COLORDEPTH_141414 5
/// Indicates if color depth is 161616
#define ADL_COLORDEPTH_161616 6
/// Indicates default value of color depth
#define ADL_COLOR_DEPTH_DEF ADL_COLORDEPTH_888
// @}


/// \defgroup define_emulation_status
/// These defines are the status of emulation
// @{
/// Indicates if real device is connected.
#define ADL_EMUL_STATUS_REAL_DEVICE_CONNECTED 0x1
/// Indicates if emulated device is presented.
#define ADL_EMUL_STATUS_EMULATED_DEVICE_PRESENT 0x2
/// Indicates if emulated device is used.
#define ADL_EMUL_STATUS_EMULATED_DEVICE_USED  0x4
/// In case when last active real/emulated device used (when persistence is enabled but no emulation enforced then persistence will use last connected/emulated device).
#define ADL_EMUL_STATUS_LAST_ACTIVE_DEVICE_USED 0x8
// @}

/// \defgroup define_emulation_mode
/// These defines are the modes of emulation
// @{
/// Indicates if no emulation is used
#define ADL_EMUL_MODE_OFF 0
/// Indicates if emulation is used when display connected
#define ADL_EMUL_MODE_ON_CONNECTED 1
/// Indicates if emulation is used when display dis connected
#define ADL_EMUL_MODE_ON_DISCONNECTED 2
/// Indicates if emulation is used always
#define ADL_EMUL_MODE_ALWAYS 3
// @}

/// \defgroup define_emulation_query
/// These defines are the modes of emulation
// @{
/// Indicates Data from real device
#define ADL_QUERY_REAL_DATA 0
/// Indicates Emulated data
#define ADL_QUERY_EMULATED_DATA 1
/// Indicates Data currently in use
#define ADL_QUERY_CURRENT_DATA 2
// @}

/// \defgroup define_persistence_state
/// These defines are the states of persistence
// @{
/// Indicates persistence is disabled
#define ADL_EDID_PERSISTANCE_DISABLED 0
/// Indicates persistence is enabled
#define ADL_EDID_PERSISTANCE_ENABLED 1
// @}
/// \defgroup define_connector_types Connector Type
/// defines for ADLConnectorInfo.iType
// @{
/// Indicates unknown Connector type
#define ADL_CONNECTOR_TYPE_UNKNOWN                 0
/// Indicates VGA Connector type
#define ADL_CONNECTOR_TYPE_VGA                     1
/// Indicates DVI-D Connector type
#define ADL_CONNECTOR_TYPE_DVI_D                   2
/// Indicates DVI-I Connector type
#define ADL_CONNECTOR_TYPE_DVI_I                   3
/// Indicates Active Dongle-NA Connector type
#define ADL_CONNECTOR_TYPE_ATICVDONGLE_NA          4
/// Indicates Active Dongle-JP Connector type
#define ADL_CONNECTOR_TYPE_ATICVDONGLE_JP          5
/// Indicates Active Dongle-NONI2C Connector type
#define ADL_CONNECTOR_TYPE_ATICVDONGLE_NONI2C      6
/// Indicates Active Dongle-NONI2C-D Connector type
#define ADL_CONNECTOR_TYPE_ATICVDONGLE_NONI2C_D    7
/// Indicates HDMI-Type A Connector type
#define ADL_CONNECTOR_TYPE_HDMI_TYPE_A             8
/// Indicates HDMI-Type B Connector type
#define ADL_CONNECTOR_TYPE_HDMI_TYPE_B             9
/// Indicates Display port Connector type
#define ADL_CONNECTOR_TYPE_DISPLAYPORT             10
/// Indicates EDP Connector type
#define ADL_CONNECTOR_TYPE_EDP                     11
/// Indicates MiniDP Connector type
#define ADL_CONNECTOR_TYPE_MINI_DISPLAYPORT        12
/// Indicates Virtual Connector type
#define ADL_CONNECTOR_TYPE_VIRTUAL                   13
// @}

/// \defgroup define_freesync_usecase
/// These defines are to specify use cases in which FreeSync should be enabled
/// They are a bit mask. To specify FreeSync for more than one use case, the input value
/// should be set to include multiple bits set
// @{
/// Indicates FreeSync is enabled for Static Screen case
#define ADL_FREESYNC_USECASE_STATIC                 0x1
/// Indicates FreeSync is enabled for Video use case
#define ADL_FREESYNC_USECASE_VIDEO                  0x2
/// Indicates FreeSync is enabled for Gaming use case
// 定义了ADL_FREESYNC_USECASE_GAMING为0x4

/// \defgroup define_freesync_caps
/// 用于检索FreeSync显示器功能的定义
/// GPU支持标志还指示显示器是否连接到实际支持FreeSync的GPU
// @{
#define ADL_FREESYNC_CAP_SUPPORTED                      (1 << 0)  // 表示FreeSync功能受支持
#define ADL_FREESYNC_CAP_GPUSUPPORTED                   (1 << 1)  // 表示GPU支持FreeSync
#define ADL_FREESYNC_CAP_DISPLAYSUPPORTED               (1 << 2)  // 表示显示器支持FreeSync
#define ADL_FREESYNC_CAP_CURRENTMODESUPPORTED           (1 << 3)  // 表示当前模式支持FreeSync
#define ADL_FREESYNC_CAP_NOCFXORCFXSUPPORTED            (1 << 4)  // 表示不支持CFX或CFX
#define ADL_FREESYNC_CAP_NOGENLOCKORGENLOCKSUPPORTED    (1 << 5)  // 表示不支持Genlock或Genlock
#define ADL_FREESYNC_CAP_BORDERLESSWINDOWSUPPORTED      (1 << 6)  // 表示支持无边界窗口
// @}


/// \defgroup define_MST_CommandLine_execute
// @{
/// 如果设置了该位，则表示MST命令行为分支消息，否则为显示消息
#define ADL_MST_COMMANDLINE_PATH_MSG                 0x1  // 表示MST命令行为分支消息
/// 如果设置了该位，则表示以广播方式发送消息的MST命令行
#define ADL_MST_COMMANDLINE_BROADCAST                  0x2  // 表示以广播方式发送消息的MST命令行
// @}


/// \defgroup define_Adapter_CloneTypes_Get
// @{
/// 表示存在与非AMD显示器的跨GPU克隆
#define ADL_CROSSGPUDISPLAYCLONE_AMD_WITH_NONAMD                 0x1  // 表示存在与非AMD显示器的跨GPU克隆
/// 表示存在跨GPU克隆
#define ADL_CROSSGPUDISPLAYCLONE                  0x2  // 表示存在跨GPU克隆
// @}

/// \defgroup define_D3DKMT_HANDLE
// @{
/// 当使用CreateDevice()时，句柄可用于创建设备句柄
typedef unsigned int ADL_D3DKMT_HANDLE;  // 句柄可用于创建设备句柄
// @}


// 常量和定义的结束括号。在此行之上添加新的组！

// @}


typedef enum _ADL_RAS_ERROR_INJECTION_MODE
{
    ADL_RAS_ERROR_INJECTION_MODE_SINGLE = 1,  // 单个错误注入模式
    ADL_RAS_ERROR_INJECTION_MODE_MULTIPLE = 2  // 多个错误注入模式
}ADL_RAS_ERROR_INJECTION_MODE;


typedef enum _ADL_RAS_BLOCK_ID
{
    ADL_RAS_BLOCK_ID_UMC = 0,  // UMC块ID为0
    ADL_RAS_BLOCK_ID_SDMA,  // SDMA块ID
    ADL_RAS_BLOCK_ID_GFX_HUB,  // GFX_HUB块ID
    ADL_RAS_BLOCK_ID_MMHUB,  // MMHUB块ID
    ADL_RAS_BLOCK_ID_ATHUB,  // ATHUB块ID
    # 定义了一系列的常量，代表不同的 RAS（Reliability, Availability, and Serviceability）块的 ID
    # 这些常量可能用于标识不同的硬件模块或功能模块
    ADL_RAS_BLOCK_ID_PCIE_BIF,
    ADL_RAS_BLOCK_ID_HDP,
    ADL_RAS_BLOCK_ID_XGMI_WAFL,
    ADL_RAS_BLOCK_ID_DF,
    ADL_RAS_BLOCK_ID_SMN,
    ADL_RAS_BLOCK_ID_SEM,
    ADL_RAS_BLOCK_ID_MP0,
    ADL_RAS_BLOCK_ID_MP1,
    ADL_RAS_BLOCK_ID_FUSE
// 定义了一个枚举类型 ADL_RAS_BLOCK_ID，用于表示 RAS（Reliability, Availability, and Serviceability）块的 ID
}ADL_RAS_BLOCK_ID;

// 定义了一个枚举类型 ADL_MEM_SUB_BLOCK_ID，用于表示内存子块的 ID
typedef enum _ADL_MEM_SUB_BLOCK_ID
{
    ADL_RAS__UMC_HBM = 0,  // UMC_HBM 内存子块
    ADL_RAS__UMC_SRAM = 1  // UMC_SRAM 内存子块
}ADL_MEM_SUB_BLOCK_ID;

// 定义了一个枚举类型 ADL_RAS_ERROR_TYPE，用于表示 RAS 错误类型
typedef enum  _ADL_RAS_ERROR_TYPE
{
    // 不同的 RAS 错误类型
    ADL_RAS_ERROR__NONE = 0,
    ADL_RAS_ERROR__PARITY = 1,
    ADL_RAS_ERROR__SINGLE_CORRECTABLE = 2,
    // ... 其他 RAS 错误类型 ...
    ADL_RAS_ERROR__PARITY_SINGLE_CORRECTABLE_MULTI_UNCORRECTABLE_POISON = 15
}ADL_RAS_ERROR_TYPE;

// 定义了一个枚举类型 ADL_RAS_INJECTION_METHOD，用于表示 RAS 错误注入方法
typedef enum _ADL_RAS_INJECTION_METHOD
{
    // 不同的 RAS 错误注入方法
    ADL_RAS_ERROR__UMC_METH_COHERENT = 0,
    ADL_RAS_ERROR__UMC_METH_SINGLE_SHOT = 1,
    // ... 其他 RAS 错误注入方法 ...
    ADL_RAS_ERROR__UMC_METH_PERSISTENT_DISABLE = 3
}ADL_RAS_INJECTION_METHOD;

// 定义了一个枚举类型 ADL_DRIVER_EVENT_TYPE，用于表示驱动事件类型
typedef enum _ADL_DRIVER_EVENT_TYPE
{
    ADL_EVENT_ID_AUTO_FEATURE_COMPLETED = 30,  // 自动功能完成事件
    ADL_EVENT_ID_FEATURE_AVAILABILITY = 31,     // 功能可用性事件
} ADL_DRIVER_EVENT_TYPE;

// 定义了一个枚举类型 ADL_UIFEATURES_GROUP，用于表示 UI 功能组
typedef enum _ADL_UIFEATURES_GROUP
{
    // 不同的 UI 功能组
    ADL_UIFEATURES_GROUP_DVR = 0,
    ADL_UIFEATURES_GROUP_TURBOSYNC = 1,
    // ... 其他 UI 功能组 ...
    ADL_UIFEATURES_GROUP_XGMI = 11
}ADL_UIFEATURES_GROUP;
#endif /* ADL_DEFINES_H_ */
```