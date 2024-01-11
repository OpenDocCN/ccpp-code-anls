# `xmrig\src\core\config\Config_platform.h`

```
/* XMRig
 * 版权所有（c）2018-2021 SChernykh   <https://github.com/SChernykh>
 * 版权所有（c）2016-2021 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以重新发布或修改它
 *   根据GNU通用公共许可证的条款发布，由
 *   自由软件基金会发布的版本3或
 *   （在您的选择下）任何以后的版本。
 *
 *   本程序是希望它有用，
 *   但没有任何保证；甚至没有暗示的保证
 *   适销性或特定用途的保证。请参阅
 *   GNU通用公共许可证以获取更多详细信息。
 *
 *   您应该已经收到GNU通用公共许可证的副本
 *   与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#ifndef XMRIG_CONFIG_PLATFORM_H
#define XMRIG_CONFIG_PLATFORM_H


#ifdef _MSC_VER
#   include "getopt/getopt.h"
#else
#   include <getopt.h>
#endif


#include "base/kernel/interfaces/IConfig.h"
#include "version.h"


namespace xmrig {


static const char short_options[] = "a:c:kBp:Px:r:R:s:t:T:o:u:O:v:l:Sx:";


static const option options[] = {
    { "algo",                  1, nullptr, IConfig::AlgorithmKey          },
    { "coin",                  1, nullptr, IConfig::CoinKey               },
#   ifdef XMRIG_FEATURE_HTTP
    { "api-worker-id",         1, nullptr, IConfig::ApiWorkerIdKey        },
    { "api-id",                1, nullptr, IConfig::ApiIdKey              },
    { "http-enabled",          0, nullptr, IConfig::HttpEnabledKey        },
    { "http-host",             1, nullptr, IConfig::HttpHostKey           },
    { "http-access-token",     1, nullptr, IConfig::HttpAccessTokenKey    },
    { "http-port",             1, nullptr, IConfig::HttpPort              },
    { "http-no-restricted",    0, nullptr, IConfig::HttpRestrictedKey     },
    { "daemon",                0, nullptr, IConfig::DaemonKey             },
    # 设置守护进程轮询间隔的配置项，参数为1，无默认值，对应的键为DaemonPollKey
    { "daemon-poll-interval",  1, nullptr, IConfig::DaemonPollKey         },
    # 设置守护进程作业超时的配置项，参数为1，无默认值，对应的键为DaemonJobTimeoutKey
    { "daemon-job-timeout",    1, nullptr, IConfig::DaemonJobTimeoutKey   },
    # 设置自我选择的配置项，参数为1，无默认值，对应的键为SelfSelectKey
    { "self-select",           1, nullptr, IConfig::SelfSelectKey         },
    # 设置提交到原始位置的配置项，参数为0，无默认值，对应的键为SubmitToOriginKey
    { "submit-to-origin",      0, nullptr, IConfig::SubmitToOriginKey     },
    # 设置守护进程 ZeroMQ 端口的配置项，参数为1，无默认值，对应的键为DaemonZMQPortKey
    { "daemon-zmq-port",       1, nullptr, IConfig::DaemonZMQPortKey      },
#   endif
    # 定义配置项的名称、是否需要参数、默认值、对应的键值
    { "av",                    1, nullptr, IConfig::AVKey                 },
    { "background",            0, nullptr, IConfig::BackgroundKey         },
    { "config",                1, nullptr, IConfig::ConfigKey             },
    { "cpu-affinity",          1, nullptr, IConfig::CPUAffinityKey        },
    { "cpu-priority",          1, nullptr, IConfig::CPUPriorityKey        },
    { "donate-level",          1, nullptr, IConfig::DonateLevelKey        },
    { "donate-over-proxy",     1, nullptr, IConfig::ProxyDonateKey        },
    { "dry-run",               0, nullptr, IConfig::DryRunKey             },
    { "keepalive",             0, nullptr, IConfig::KeepAliveKey          },
    { "log-file",              1, nullptr, IConfig::LogFileKey            },
    { "nicehash",              0, nullptr, IConfig::NicehashKey           },
    { "no-color",              0, nullptr, IConfig::ColorKey              },
    { "no-huge-pages",         0, nullptr, IConfig::HugePagesKey          },
    { "no-hugepages",          0, nullptr, IConfig::HugePagesKey          },
    { "hugepage-size",         1, nullptr, IConfig::HugePageSizeKey       },
    { "huge-pages-jit",        0, nullptr, IConfig::HugePagesJitKey       },
    { "hugepages-jit",         0, nullptr, IConfig::HugePagesJitKey       },
    { "rotation",              1, nullptr, IConfig::RotationKey           },
    { "pass",                  1, nullptr, IConfig::PasswordKey           },
    { "print-time",            1, nullptr, IConfig::PrintTimeKey          },
    { "retries",               1, nullptr, IConfig::RetriesKey            },
    { "retry-pause",           1, nullptr, IConfig::RetryPauseKey         },
    { "syslog",                0, nullptr, IConfig::SyslogKey             },
    { "threads",               1, nullptr, IConfig::ThreadsKey            },
    { "url",                   1, nullptr, IConfig::UrlKey                },
    { "user",                  1, nullptr, IConfig::UserKey               },  // 设置用户信息，对应配置键值为 UserKey
    { "user-agent",            1, nullptr, IConfig::UserAgentKey          },  // 设置用户代理信息，对应配置键值为 UserAgentKey
    { "userpass",              1, nullptr, IConfig::UserpassKey           },  // 设置用户密码，对应配置键值为 UserpassKey
    { "rig-id",                1, nullptr, IConfig::RigIdKey              },  // 设置矿机 ID，对应配置键值为 RigIdKey
    { "no-cpu",                0, nullptr, IConfig::CPUKey                },  // 禁用 CPU，对应配置键值为 CPUKey
    { "max-cpu-usage",         1, nullptr, IConfig::CPUMaxThreadsKey      },  // 设置 CPU 最大使用线程数，对应配置键值为 CPUMaxThreadsKey
    { "cpu-max-threads-hint",  1, nullptr, IConfig::CPUMaxThreadsKey      },  // 设置 CPU 最大使用线程数提示，对应配置键值为 CPUMaxThreadsKey
    { "cpu-memory-pool",       1, nullptr, IConfig::MemoryPoolKey         },  // 设置 CPU 内存池，对应配置键值为 MemoryPoolKey
    { "cpu-no-yield",          0, nullptr, IConfig::YieldKey              },  // 禁用 CPU 产出，对应配置键值为 YieldKey
    { "no-yield",              0, nullptr, IConfig::YieldKey              },  // 禁用产出，对应配置键值为 YieldKey
    { "cpu-argon2-impl",       1, nullptr, IConfig::Argon2ImplKey         },  // 设置 CPU Argon2 实现，对应配置键值为 Argon2ImplKey
    { "argon2-impl",           1, nullptr, IConfig::Argon2ImplKey         },  // 设置 Argon2 实现，对应配置键值为 Argon2ImplKey
    { "verbose",               0, nullptr, IConfig::VerboseKey            },  // 设置详细信息输出，对应配置键值为 VerboseKey
    { "proxy",                 1, nullptr, IConfig::ProxyKey              },  // 设置代理，对应配置键值为 ProxyKey
    { "data-dir",              1, nullptr, IConfig::DataDirKey            },  // 设置数据目录，对应配置键值为 DataDirKey
    { "title",                 1, nullptr, IConfig::TitleKey              },  // 设置标题，对应配置键值为 TitleKey
    { "no-title",              0, nullptr, IConfig::NoTitleKey            },  // 禁用标题，对应配置键值为 NoTitleKey
    { "pause-on-battery",      0, nullptr, IConfig::PauseOnBatteryKey     },  // 在电池模式下暂停，对应配置键值为 PauseOnBatteryKey
    { "pause-on-active",       1, nullptr, IConfig::PauseOnActiveKey      },  // 在活跃模式下暂停，对应配置键值为 PauseOnActiveKey
    { "dns-ipv6",              0, nullptr, IConfig::DnsIPv6Key            },  // 启用 DNS IPv6，对应配置键值为 DnsIPv6Key
    { "dns-ttl",               1, nullptr, IConfig::DnsTtlKey             },  // 设置 DNS TTL，对应配置键值为 DnsTtlKey
    { "spend-secret-key",      1, nullptr, IConfig::SpendSecretKey        },  // 设置花费密钥，对应配置键值为 SpendSecretKey
# 如果定义了 XMRIG_FEATURE_BENCHMARK，则以下是与基准测试相关的配置项
{ "stress",                0, nullptr, IConfig::StressKey             },  # stress 参数，用于启用压力测试
{ "bench",                 1, nullptr, IConfig::BenchKey              },  # bench 参数，用于指定基准测试的工作负载
{ "benchmark",             1, nullptr, IConfig::BenchKey              },  # benchmark 参数，用于指定基准测试的工作负载
# 如果定义了 XMRIG_FEATURE_HTTP，则以下是与 HTTP 相关的基准测试配置项
{ "submit",                0, nullptr, IConfig::BenchSubmitKey        },  # submit 参数，用于提交基准测试结果
{ "verify",                1, nullptr, IConfig::BenchVerifyKey        },  # verify 参数，用于验证基准测试结果
{ "token",                 1, nullptr, IConfig::BenchTokenKey         },  # token 参数，用于指定基准测试的令牌
{ "seed",                  1, nullptr, IConfig::BenchSeedKey          },  # seed 参数，用于指定基准测试的种子
{ "hash",                  1, nullptr, IConfig::BenchHashKey          },  # hash 参数，用于指定基准测试的哈希
# 如果定义了 XMRIG_FEATURE_TLS，则以下是与 TLS 相关的配置项
{ "tls",                   0, nullptr, IConfig::TlsKey                },  # tls 参数，用于启用 TLS
{ "tls-fingerprint",       1, nullptr, IConfig::FingerprintKey        },  # tls-fingerprint 参数，用于指定 TLS 的指纹
{ "tls-cert",              1, nullptr, IConfig::TlsCertKey            },  # tls-cert 参数，用于指定 TLS 的证书
{ "tls-cert-key",          1, nullptr, IConfig::TlsCertKeyKey         },  # tls-cert-key 参数，用于指定 TLS 的证书密钥
{ "tls-dhparam",           1, nullptr, IConfig::TlsDHparamKey         },  # tls-dhparam 参数，用于指定 TLS 的 DH 参数
{ "tls-protocols",         1, nullptr, IConfig::TlsProtocolsKey       },  # tls-protocols 参数，用于指定 TLS 的协议
{ "tls-ciphers",           1, nullptr, IConfig::TlsCiphersKey         },  # tls-ciphers 参数，用于指定 TLS 的加密算法
{ "tls-ciphersuites",      1, nullptr, IConfig::TlsCipherSuitesKey    },  # tls-ciphersuites 参数，用于指定 TLS 的密码套件
{ "tls-gen",               1, nullptr, IConfig::TlsGenKey             },  # tls-gen 参数，用于生成 TLS 相关的配置
# 如果定义了 XMRIG_FEATURE_ASM，则以下是与汇编优化相关的配置项
{ "asm",                   1, nullptr, IConfig::AssemblyKey           },  # asm 参数，用于启用汇编优化
# 如果定义了 XMRIG_ALGO_RANDOMX，则以下是与 RandomX 算法相关的配置项
{ "randomx-init",          1, nullptr, IConfig::RandomXInitKey        },  # randomx-init 参数，用于指定 RandomX 初始化参数
{ "randomx-no-numa",       0, nullptr, IConfig::RandomXNumaKey        },  # randomx-no-numa 参数，用于禁用 NUMA
{ "randomx-mode",          1, nullptr, IConfig::RandomXModeKey        },  # randomx-mode 参数，用于指定 RandomX 的模式
{ "randomx-1gb-pages",     0, nullptr, IConfig::RandomX1GbPagesKey    },  # randomx-1gb-pages 参数，用于启用 1GB 页
{ "1gb-pages",             0, nullptr, IConfig::RandomX1GbPagesKey    },  # 1gb-pages 参数，用于启用 1GB 页
    # 创建一个包含字符串、整数、空指针和枚举值的数组
    { "randomx-wrmsr",         2, nullptr, IConfig::RandomXWrmsrKey       },
    # 创建一个包含字符串、整数、空指针和枚举值的数组
    { "wrmsr",                 2, nullptr, IConfig::RandomXWrmsrKey       },
    # 创建一个包含字符串、整数、空指针和枚举值的数组
    { "randomx-no-rdmsr",      0, nullptr, IConfig::RandomXRdmsrKey       },
    # 创建一个包含字符串、整数、空指针和枚举值的数组
    { "no-rdmsr",              0, nullptr, IConfig::RandomXRdmsrKey       },
    # 创建一个包含字符串、整数、空指针和枚举值的数组
    { "randomx-cache-qos",     0, nullptr, IConfig::RandomXCacheQoSKey    },
    # 创建一个包含字符串、整数、空指针和枚举值的数组
    { "cache-qos",             0, nullptr, IConfig::RandomXCacheQoSKey    },
// 如果定义了 XMRIG_FEATURE_OPENCL，则添加以下配置项
    { "opencl",                0, nullptr, IConfig::OclKey                },  // 设置 opencl 配置项
    { "opencl-devices",        1, nullptr, IConfig::OclDevicesKey         },  // 设置 opencl-devices 配置项
    { "opencl-platform",       1, nullptr, IConfig::OclPlatformKey        },  // 设置 opencl-platform 配置项
    { "opencl-loader",         1, nullptr, IConfig::OclLoaderKey          },  // 设置 opencl-loader 配置项
    { "opencl-no-cache",       0, nullptr, IConfig::OclCacheKey           },  // 设置 opencl-no-cache 配置项
// 如果定义了 XMRIG_FEATURE_CUDA，则添加以下配置项
    { "cuda",                  0, nullptr, IConfig::CudaKey               },  // 设置 cuda 配置项
    { "cuda-loader",           1, nullptr, IConfig::CudaLoaderKey         },  // 设置 cuda-loader 配置项
    { "cuda-devices",          1, nullptr, IConfig::CudaDevicesKey        },  // 设置 cuda-devices 配置项
    { "cuda-bfactor-hint",     1, nullptr, IConfig::CudaBFactorKey        },  // 设置 cuda-bfactor-hint 配置项
    { "cuda-bsleep-hint",      1, nullptr, IConfig::CudaBSleepKey         },  // 设置 cuda-bsleep-hint 配置项
// 如果定义了 XMRIG_FEATURE_NVML，则添加以下配置项
    { "no-nvml",               0, nullptr, IConfig::NvmlKey               },  // 设置 no-nvml 配置项
// 如果定义了 XMRIG_FEATURE_NVML 或 XMRIG_FEATURE_ADL，则添加以下配置项
    { "health-print-time",     1, nullptr, IConfig::HealthPrintTimeKey    },  // 设置 health-print-time 配置项
// 如果定义了 XMRIG_FEATURE_DMI，则添加以下配置项
    { "no-dmi",                0, nullptr, IConfig::DmiKey                },  // 设置 no-dmi 配置项
    { nullptr,                 0, nullptr, 0 }  // 结束配置项列表
};


} // namespace xmrig  // 结束 xmrig 命名空间
#endif /* XMRIG_CONFIG_PLATFORM_H */  // 结束条件编译指令
```