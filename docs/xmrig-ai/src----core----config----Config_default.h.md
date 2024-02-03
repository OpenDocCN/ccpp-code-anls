# `xmrig\src\core\config\Config_default.h`

```cpp
/* XMRig
 * 版权所有 (c) 2018-2021 SChernykh   <https://github.com/SChernykh>
 * 版权所有 (c) 2016-2021 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以重新发布或修改它
 *   根据由自由软件基金会发布的 GNU 通用公共许可证的条款，
 *   无论是许可证的第 3 版还是（在您的选择下）任何以后的版本。
 *
 *   本程序是希望它有用的分发，
 *   但没有任何保证；甚至没有暗示的保证
 *   适销性或特定用途的保证。更多详情请参见
 *   GNU 通用公共许可证。
 *
 *   您应该已经收到了 GNU 通用公共许可证的副本
 *   与本程序一起。如果没有，请参见 <http://www.gnu.org/licenses/>。
 */


#ifndef XMRIG_CONFIG_DEFAULT_H
#define XMRIG_CONFIG_DEFAULT_H


namespace xmrig {


// 此功能需要 CMake 选项：-DWITH_EMBEDDED_CONFIG=ON
#ifdef XMRIG_FEATURE_EMBEDDED_CONFIG
const static char *default_config =
R"===(
{
    "api": {
        "id": null,
        "worker-id": null
    },
    "http": {
        "enabled": false,
        "host": "127.0.0.1",
        "port": 0,
        "access-token": null,
        "restricted": true
    },
    "autosave": true,
    "background": false,
    "colors": true,
    "title": true,
    "randomx": {
        "init": -1,
        "init-avx2": -1,
        "mode": "auto",
        "1gb-pages": false,
        "rdmsr": true,
        "wrmsr": true,
        "cache_qos": false,
        "numa": true,
        "scratchpad_prefetch_mode": 1
    },
    "cpu": {
        "enabled": true,
        "huge-pages": true,
        "huge-pages-jit": false,
        "hw-aes": null,
        "priority": null,
        "memory-pool": false,
        "yield": true,
        "max-threads-hint": 100,
        "asm": true,
        "argon2-impl": null,
        "cn/0": false,
        "cn-lite/0": false
    },
    # 设置 OpenCL 参数
    "opencl": {
        # 是否启用 OpenCL
        "enabled": false,
        # 是否缓存
        "cache": true,
        # 加载器
        "loader": null,
        # 平台
        "platform": "AMD",
        # ADL
        "adl": true,
        # cn/0
        "cn/0": false,
        # cn-lite/0
        "cn-lite/0": false
    },
    # 设置 CUDA 参数
    "cuda": {
        # 是否启用 CUDA
        "enabled": false,
        # 加载器
        "loader": null,
        # NVML
        "nvml": true,
        # cn/0
        "cn/0": false,
        # cn-lite/0
        "cn-lite/0": false
    },
    # 设置捐赠级别
    "donate-level": 1,
    # 通过代理进行捐赠
    "donate-over-proxy": 1,
    # 日志文件
    "log-file": null,
    # 设置矿池信息
    "pools": [
        {
            # 算法
            "algo": null,
            # 币种
            "coin": null,
            # 矿池地址
            "url": "donate.v2.xmrig.com:3333",
            # 矿工钱包地址
            "user": "YOUR_WALLET_ADDRESS",
            # 密码
            "pass": "x",
            # 矿机ID
            "rig-id": null,
            # 是否为 NiceHash
            "nicehash": false,
            # 保持连接
            "keepalive": false,
            # 是否启用
            "enabled": true,
            # 是否使用 TLS
            "tls": false,
            # TLS 指纹
            "tls-fingerprint": null,
            # 是否为守护进程
            "daemon": false,
            # SOCKS5 代理
            "socks5": null,
            # 自动选择
            "self-select": null,
            # 提交到原始地址
            "submit-to-origin": false
        }
    ],
    # 打印时间间隔
    "print-time": 60,
    # 健康状态打印时间间隔
    "health-print-time": 60,
    # DMI
    "dmi": true,
    # 重试次数
    "retries": 5,
    # 重试暂停时间
    "retry-pause": 5,
    # 系统日志
    "syslog": false,
    # TLS 设置
    "tls": {
        # 是否启用 TLS
        "enabled": false,
        # 协议
        "protocols": null,
        # 证书
        "cert": null,
        # 证书密钥
        "cert_key": null,
        # 加密算法
        "ciphers": null,
        # 密钥交换算法
        "ciphersuites": null,
        # DH 参数
        "dhparam": null
    },
    # 用户代理
    "user-agent": null,
    # 详细级别
    "verbose": 0,
    # 监控
    "watch": true,
    # 电池暂停
    "pause-on-battery": false,
    # 活动暂停
    "pause-on-active": false
// 结束 xmrig 命名空间
}
// 结束 ifdef 条件编译
)===";
#endif

// 结束 xmrig 命名空间
} // namespace xmrig

// 结束 ifndef 条件编译
#endif /* XMRIG_CONFIG_DEFAULT_H */
```