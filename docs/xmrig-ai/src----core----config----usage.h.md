# `xmrig\src\core\config\usage.h`

```cpp
/* XMRig
 * 版权声明
 * 该程序是自由软件：您可以重新发布或修改它，遵循 GNU 通用公共许可证的条款，可以选择遵循许可证的第三版或者之后的版本。
 * 该程序是基于希望它能有用而分发的，但没有任何担保；甚至没有暗示的担保，包括适销性或特定用途的适用性。详细信息请参见 GNU 通用公共许可证。
 * 您应该已经收到了 GNU 通用公共许可证的副本。如果没有，请访问 http://www.gnu.org/licenses/。
 */

#ifndef XMRIG_USAGE_H
#define XMRIG_USAGE_H

#include "version.h"

#include <string>

namespace xmrig {

// 定义静态内联函数，返回用法说明
static inline const std::string &usage()
{
    // 静态字符串对象，用于存储用法说明
    static std::string u;

    // 如果用法说明不为空，则直接返回
    if (!u.empty()) {
        return u;
    }

    // 拼接用法说明
    u += "Usage: " APP_ID " [OPTIONS]\n\nNetwork:\n";
    u += "  -o, --url=URL                 URL of mining server\n";
    u += "  -a, --algo=ALGO               mining algorithm https://xmrig.com/docs/algorithms\n";
    u += "      --coin=COIN               specify coin instead of algorithm\n";
    u += "  -u, --user=USERNAME           username for mining server\n";
    u += "  -p, --pass=PASSWORD           password for mining server\n";
    u += "  -O, --userpass=U:P            username:password pair for mining server\n";
    u += "  -x, --proxy=HOST:PORT         connect through a SOCKS5 proxy\n";
    # 将字符串添加到变量 u 中，用于描述命令行参数 -k, --keepalive 的作用
    u += "  -k, --keepalive               send keepalived packet for prevent timeout (needs pool support)\n";
    # 将字符串添加到变量 u 中，用于描述命令行参数 --nicehash 的作用
    u += "      --nicehash                enable nicehash.com support\n";
    # 将字符串添加到变量 u 中，用于描述命令行参数 --rig-id=ID 的作用
    u += "      --rig-id=ID               rig identifier for pool-side statistics (needs pool support)\n";
#   ifdef XMRIG_FEATURE_TLS
    # 如果定义了 XMRIG_FEATURE_TLS，则添加启用 SSL/TLS 支持的说明
    u += "      --tls                     enable SSL/TLS support (needs pool support)\n";
    # 如果定义了 XMRIG_FEATURE_TLS，则添加设置严格证书固定的说明
    u += "      --tls-fingerprint=HEX     pool TLS certificate fingerprint for strict certificate pinning\n";
#   endif

    # 添加优先使用 IPv6 地址的说明
    u += "      --dns-ipv6                prefer IPv6 records from DNS responses\n";
    # 添加设置内部 DNS 缓存的 TTL 时间的说明
    u += "      --dns-ttl=N               N seconds (default: 30) TTL for internal DNS cache\n";

#   ifdef XMRIG_FEATURE_HTTP
    # 如果定义了 XMRIG_FEATURE_HTTP，则添加使用守护进程 RPC 进行独立挖矿的说明
    u += "      --daemon                  use daemon RPC instead of pool for solo mining\n";
    # 如果定义了 XMRIG_FEATURE_HTTP，则添加设置守护进程 zmq-pub 端口号的说明
    u += "      --daemon-zmq-port         daemon's zmq-pub port number (only use it if daemon has it enabled)\n";
    # 如果定义了 XMRIG_FEATURE_HTTP，则添加设置守护进程轮询间隔的说明
    u += "      --daemon-poll-interval=N  daemon poll interval in milliseconds (default: 1000)\n";
    # 如果定义了 XMRIG_FEATURE_HTTP，则添加设置守护进程作业超时时间的说明
    u += "      --daemon-job-timeout=N    daemon job timeout in milliseconds (default: 15000)\n";
    # 如果定义了 XMRIG_FEATURE_HTTP，则添加从指定 URL 自动选择区块模板的说明
    u += "      --self-select=URL         self-select block templates from URL\n";
    # 如果定义了 XMRIG_FEATURE_HTTP，则添加将解决方案提交回自选 URL 的说明
    u += "      --submit-to-origin        also submit solution back to self-select URL\n";
#   endif

    # 添加重试次数的说明
    u += "  -r, --retries=N               number of times to retry before switch to backup server (default: 5)\n";
    # 添加重试间隔时间的说明
    u += "  -R, --retry-pause=N           time to pause between retries (default: 5)\n";
    # 添加设置自定义用户代理字符串的说明
    u += "      --user-agent              set custom user-agent string for pool\n";
    # 添加设置捐赠级别的说明
    u += "      --donate-level=N          donate level, default 1%% (1 minute in 100 minutes)\n";
    # 添加控制通过 xmrig-proxy 功能进行捐赠的说明
    u += "      --donate-over-proxy=N     control donate over xmrig-proxy feature\n";

    # 添加 CPU 后端的说明
    u += "\nCPU backend:\n";

    # 添加禁用 CPU 挖矿后端的说明
    u += "      --no-cpu                  disable CPU mining backend\n";
    # 添加设置 CPU 线程数的说明
    u += "  -t, --threads=N               number of CPU threads, proper CPU affinity required for some optimizations.\n";
    # 添加设置进程亲和性到 CPU 核心的说明
    u += "      --cpu-affinity=N          set process affinity to CPU core(s), mask 0x3 for cores 0 and 1\n";
    # 添加算法变化的说明
    u += "  -v, --av=N                    algorithm variation, 0 auto select\n";
    # 设置进程优先级（0为闲置，2为正常，5为最高）
    u += "      --cpu-priority=N          set process priority (0 idle, 2 normal to 5 highest)\n";
    # 设置自动配置的最大CPU线程数（以百分比表示）
    u += "      --cpu-max-threads-hint=N  maximum CPU threads count (in percentage) hint for autoconfig\n";
    # 设置持久内存池的2MB页面数，-1为自动，0为禁用
    u += "      --cpu-memory-pool=N       number of 2 MB pages for persistent memory pool, -1 (auto), 0 (disable)\n";
    # 优先获取最大的哈希率，而不是系统响应/稳定性
    u += "      --cpu-no-yield            prefer maximum hashrate rather than system response/stability\n";
    # 禁用大页面支持
    u += "      --no-huge-pages           disable huge pages support\n";
#   ifdef XMRIG_OS_LINUX
    # 如果定义了 XMRIG_OS_LINUX，则添加自定义巨页大小的说明
    u += "      --hugepage-size=N         custom hugepage size in kB\n";
#   endif
#   ifdef XMRIG_ALGO_RANDOMX
    # 如果定义了 XMRIG_ALGO_RANDOMX，则添加启用 RandomX JIT 代码的巨页支持说明
    u += "      --huge-pages-jit          enable huge pages support for RandomX JIT code\n";
#   endif
    # 添加 ASM 优化的说明，包括可能的取值
    u += "      --asm=ASM                 ASM optimizations, possible values: auto, none, intel, ryzen, bulldozer\n";

#   if defined(__x86_64__) || defined(_M_AMD64)
    # 如果定义了 __x86_64__ 或 _M_AMD64，则添加 argon2 实现的说明，包括可能的取值
    u += "      --argon2-impl=IMPL        argon2 implementation: x86_64, SSE2, SSSE3, XOP, AVX2, AVX-512F\n";
#   endif

#   ifdef XMRIG_ALGO_RANDOMX
    # 如果定义了 XMRIG_ALGO_RANDOMX，则添加 RandomX 初始化数据集线程数的说明
    u += "      --randomx-init=N          threads count to initialize RandomX dataset\n";
    # 添加禁用 RandomX 的 NUMA 支持的说明
    u += "      --randomx-no-numa         disable NUMA support for RandomX\n";
    # 添加 RandomX 模式的说明，包括可能的取值
    u += "      --randomx-mode=MODE       RandomX mode: auto, fast, light\n";
    # 添加使用 1GB 巨页的 RandomX 数据集的说明（仅限 Linux）
    u += "      --randomx-1gb-pages       use 1GB hugepages for RandomX dataset (Linux only)\n";
    # 添加写入自定义值到 MSR 寄存器或禁用 MSR 修改的说明
    u += "      --randomx-wrmsr=N         write custom value(s) to MSR registers or disable MSR mod (-1)\n";
    # 添加禁用退出时恢复初始 MSR 值的说明
    u += "      --randomx-no-rdmsr        disable reverting initial MSR values on exit\n";
    # 添加启用缓存 QoS 的说明
    u += "      --randomx-cache-qos       enable Cache QoS\n";
#   endif

#   ifdef XMRIG_FEATURE_OPENCL
    # 如果定义了 XMRIG_FEATURE_OPENCL，则添加 OpenCL 后端的说明
    u += "\nOpenCL backend:\n";
    # 添加启用 OpenCL 挖矿后端的说明
    u += "      --opencl                  enable OpenCL mining backend\n";
    # 添加要使用的 OpenCL 设备的逗号分隔列表的说明
    u += "      --opencl-devices=N        comma separated list of OpenCL devices to use\n";
    # 添加 OpenCL 平台索引或名称的说明
    u += "      --opencl-platform=N       OpenCL platform index or name\n";
    # 添加 OpenCL-ICD-Loader 的路径的说明
    u += "      --opencl-loader=PATH      path to OpenCL-ICD-Loader (OpenCL.dll or libOpenCL.so)\n";
    # 添加禁用 OpenCL 缓存的说明
    u += "      --opencl-no-cache         disable OpenCL cache\n";
    # 添加打印可用的 OpenCL 平台并退出的说明
    u += "      --print-platforms         print available OpenCL platforms and exit\n";
#   endif

#   ifdef XMRIG_FEATURE_CUDA
    # 如果定义了 XMRIG_FEATURE_CUDA，则添加 CUDA 后端的说明
    u += "\nCUDA backend:\n";
    # 添加启用 CUDA 挖矿后端的说明
    u += "      --cuda                    enable CUDA mining backend\n";
    # 添加 CUDA 插件路径的说明
    u += "      --cuda-loader=PATH        path to CUDA plugin (xmrig-cuda.dll or libxmrig-cuda.so)\n";
    # 将字符串添加到变量 u 中，用于描述使用 CUDA 设备的参数
    u += "      --cuda-devices=N          comma separated list of CUDA devices to use\n";
    # 将字符串添加到变量 u 中，用于描述自动配置的 bfactor hint 参数
    u += "      --cuda-bfactor-hint=N     bfactor hint for autoconfig (0-12)\n";
    # 将字符串添加到变量 u 中，用于描述自动配置的 bsleep hint 参数
    u += "      --cuda-bsleep-hint=N      bsleep hint for autoconfig\n";
#   endif
#   ifdef XMRIG_FEATURE_NVML
    # 如果定义了 XMRIG_FEATURE_NVML，则添加禁用 NVML 支持的选项说明
    u += "      --no-nvml                 disable NVML (NVIDIA Management Library) support\n";
#   endif

#   ifdef XMRIG_FEATURE_HTTP
    # 如果定义了 XMRIG_FEATURE_HTTP，则添加 API 选项说明
    u += "\nAPI:\n";
    u += "      --api-worker-id=ID        custom worker-id for API\n";
    u += "      --api-id=ID               custom instance ID for API\n";
    u += "      --http-host=HOST          bind host for HTTP API (default: 127.0.0.1)\n";
    u += "      --http-port=N             bind port for HTTP API\n";
    u += "      --http-access-token=T     access token for HTTP API\n";
    u += "      --http-no-restricted      enable full remote access to HTTP API (only if access token set)\n";
#   endif

#   ifdef XMRIG_FEATURE_TLS
    # 如果定义了 XMRIG_FEATURE_TLS，则添加 TLS 选项说明
    u += "\nTLS:\n";
    u += "      --tls-gen=HOSTNAME        generate TLS certificate for specific hostname\n";
    u += "      --tls-cert=FILE           load TLS certificate chain from a file in the PEM format\n";
    u += "      --tls-cert-key=FILE       load TLS certificate private key from a file in the PEM format\n";
    u += "      --tls-dhparam=FILE        load DH parameters for DHE ciphers from a file in the PEM format\n";
    u += "      --tls-protocols=N         enable specified TLS protocols, example: \"TLSv1 TLSv1.1 TLSv1.2 TLSv1.3\"\n";
    u += "      --tls-ciphers=S           set list of available ciphers (TLSv1.2 and below)\n";
    u += "      --tls-ciphersuites=S      set list of available TLSv1.3 ciphersuites\n";
#   endif

    # 添加日志选项说明
    u += "\nLogging:\n";

#   ifdef HAVE_SYSLOG_H
    # 如果定义了 HAVE_SYSLOG_H，则添加使用系统日志输出消息的选项说明
    u += "  -S, --syslog                  use system log for output messages\n";
#   endif

    u += "  -l, --log-file=FILE           log all output to a file\n";
    u += "      --print-time=N            print hashrate report every N seconds\n";
#   if defined(XMRIG_FEATURE_NVML) || defined(XMRIG_FEATURE_ADL)
    # 如果定义了 XMRIG_FEATURE_NVML 或 XMRIG_FEATURE_ADL，则添加打印健康报告的选项说明
    u += "      --health-print-time=N     print health report every N seconds\n";
#   endif
    u += "      --no-color                disable colored output\n";
    # 添加 --verbose 选项的说明到帮助信息字符串
    u += "      --verbose                 verbose output\n";

    # 添加 Misc 标题到帮助信息字符串
    u += "\nMisc:\n";

    # 添加 --config 选项的说明到帮助信息字符串
    u += "  -c, --config=FILE             load a JSON-format configuration file\n";
    # 添加 --background 选项的说明到帮助信息字符串
    u += "  -B, --background              run the miner in the background\n";
    # 添加 --version 选项的说明到帮助信息字符串
    u += "  -V, --version                 output version information and exit\n";
    # 添加 --help 选项的说明到帮助信息字符串
    u += "  -h, --help                    display this help and exit\n";
    # 添加 --dry-run 选项的说明到帮助信息字符串
    u += "      --dry-run                 test configuration and exit\n";
#ifdef XMRIG_FEATURE_HWLOC
    #ifdef 指令，如果定义了 XMRIG_FEATURE_HWLOC 宏，则执行以下代码
    u += "      --export-topology         export hwloc topology to a XML file and exit\n";
#endif

#ifdef XMRIG_OS_WIN
    #ifdef 指令，如果定义了 XMRIG_OS_WIN 宏，则执行以下代码
    u += "      --title                   set custom console window title\n";
    u += "      --no-title                disable setting console window title\n";
#endif
    u += "      --pause-on-battery        pause mine on battery power\n";
    u += "      --pause-on-active=N       pause mine when the user is active (resume after N seconds of last activity)\n";

#ifdef XMRIG_FEATURE_BENCHMARK
    #ifdef 指令，如果定义了 XMRIG_FEATURE_BENCHMARK 宏，则执行以下代码
    u += "      --stress                  run continuous stress test to check system stability\n";
    u += "      --bench=N                 run benchmark, N can be between 1M and 10M\n";
    #ifdef XMRIG_FEATURE_HTTP
        #ifdef 指令，如果定义了 XMRIG_FEATURE_HTTP 宏，则执行以下代码
        u += "      --submit                  perform an online benchmark and submit result for sharing\n";
        u += "      --verify=ID               verify submitted benchmark by ID\n";
    #endif
    u += "      --seed=SEED               custom RandomX seed for benchmark\n";
    u += "      --hash=HASH               compare benchmark result with specified hash\n";
#endif

#ifdef XMRIG_FEATURE_DMI
    #ifdef 指令，如果定义了 XMRIG_FEATURE_DMI 宏，则执行以下代码
    u += "      --no-dmi                  disable DMI/SMBIOS reader\n";
#endif

return u;
}

} /* namespace xmrig */

#endif /* XMRIG_USAGE_H */
```