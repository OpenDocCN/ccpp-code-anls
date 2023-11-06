# xmrig源码解析 0

# v6.20.0
- Added new ARM CPU names.
- [#2394](https://github.com/xmrig/xmrig/pull/2394) Added new CMake options `ARM_V8` and `ARM_V7`.
- [#2830](https://github.com/xmrig/xmrig/pull/2830) Added API rebind polling.
- [#2927](https://github.com/xmrig/xmrig/pull/2927) Fixed compatibility with hwloc 1.11.x.
- [#3060](https://github.com/xmrig/xmrig/pull/3060) Added x86 to `README.md`.
- [#3236](https://github.com/xmrig/xmrig/pull/3236) Fixed: receive CUDA loader error on Linux too.
- [#3290](https://github.com/xmrig/xmrig/pull/3290) Added [Zephyr](https://www.zephyrprotocol.com/) coin support for solo mining.

# v6.19.3
- [#3245](https://github.com/xmrig/xmrig/issues/3245) Improved algorithm negotiation for donation rounds by sending extra information about current mining job.
- [#3254](https://github.com/xmrig/xmrig/pull/3254) Tweaked auto-tuning for Intel CPUs.
- [#3271](https://github.com/xmrig/xmrig/pull/3271) RandomX: optimized program generation.
- [#3273](https://github.com/xmrig/xmrig/pull/3273) RandomX: fixed undefined behavior.
- [#3275](https://github.com/xmrig/xmrig/pull/3275) RandomX: fixed `jccErratum` list.
- [#3280](https://github.com/xmrig/xmrig/pull/3280) Updated example scripts.

# v6.19.2
- [#3230](https://github.com/xmrig/xmrig/pull/3230) Fixed parsing of `TX_EXTRA_MERGE_MINING_TAG`.
- [#3232](https://github.com/xmrig/xmrig/pull/3232) Added new `X-Hash-Difficulty` HTTP header.
- [#3240](https://github.com/xmrig/xmrig/pull/3240) Improved .cmd files when run by shortcuts on another drive.
- [#3241](https://github.com/xmrig/xmrig/pull/3241) Added view tag calculation (fixes Wownero solo mining issue).

# v6.19.1
- Resolved deprecated methods warnings with OpenSSL 3.0.
- [#3213](https://github.com/xmrig/xmrig/pull/3213) Fixed build with 32-bit clang 15.
- [#3218](https://github.com/xmrig/xmrig/pull/3218) Fixed: `--randomx-wrmsr=-1` worked only on Intel.
- [#3228](https://github.com/xmrig/xmrig/pull/3228) Fixed build with gcc 13.

# v6.19.0
- [#3144](https://github.com/xmrig/xmrig/pull/3144) Update to latest `sse2neon.h`.
- [#3161](https://github.com/xmrig/xmrig/pull/3161) MSVC build: enabled parallel compilation.
- [#3163](https://github.com/xmrig/xmrig/pull/3163) Improved Zen 3 MSR mod.
- [#3176](https://github.com/xmrig/xmrig/pull/3176) Update cmake required version to 3.1.
- [#3182](https://github.com/xmrig/xmrig/pull/3182) DragonflyBSD compilation fixes.
- [#3196](https://github.com/xmrig/xmrig/pull/3196) Show IP address for failed connections.
- [#3185](https://github.com/xmrig/xmrig/issues/3185) Fixed macOS DMI reader.
- [#3198](https://github.com/xmrig/xmrig/pull/3198) Fixed broken RandomX light mode mining.
- [#3202](https://github.com/xmrig/xmrig/pull/3202) Solo mining: added job timeout (default is 15 seconds).

# v6.18.1
- [#3129](https://github.com/xmrig/xmrig/pull/3129) Fix: protectRX flushed CPU cache only on MacOS/iOS.
- [#3126](https://github.com/xmrig/xmrig/pull/3126) Don't reset when pool sends the same job blob.
- [#3120](https://github.com/xmrig/xmrig/pull/3120) RandomX: optimized `CFROUND` elimination.
- [#3109](https://github.com/xmrig/xmrig/pull/3109) RandomX: added Blake2 AVX2 version.
- [#3082](https://github.com/xmrig/xmrig/pull/3082) Fixed GCC 12 warnings.
- [#3075](https://github.com/xmrig/xmrig/pull/3075) Recognize `armv7ve` as valid ARMv7 target.
- [#3132](https://github.com/xmrig/xmrig/pull/3132) RandomX: added MSR mod for Zen 4.
- [#3134](https://github.com/xmrig/xmrig/pull/3134) Added Zen4 to `randomx_boost.sh`.

# v6.18.0
- [#3067](https://github.com/xmrig/xmrig/pull/3067) Monero v15 network upgrade support and more house keeping.
  - Removed deprecated AstroBWTv1 and v2.
  - Fixed debug GhostRider build.
  - Monero v15 network upgrade support.
  - Fixed ZMQ debug log.
  - Improved daemon ZMQ mining stability.
- [#3054](https://github.com/xmrig/xmrig/pull/3054) Fixes for 32-bit ARM.
- [#3042](https://github.com/xmrig/xmrig/pull/3042) Fixed being unable to resume from `pause-on-battery`.
- [#3031](https://github.com/xmrig/xmrig/pull/3031) Fixed `--cpu-priority` not working sometimes.
- [#3020](https://github.com/xmrig/xmrig/pull/3020) Removed old AstroBWT algorithm.

# v6.17.0
- [#2954](https://github.com/xmrig/xmrig/pull/2954) **Dero HE fork support (`astrobwt/v2` algorithm).**
  - [#2961](https://github.com/xmrig/xmrig/pull/2961) Dero HE (`astrobwt/v2`) CUDA config generator.
  - [#2969](https://github.com/xmrig/xmrig/pull/2969) Dero HE (`astrobwt/v2`) OpenCL support.
- Fixed displayed DMI memory information for empty slots.
- [#2932](https://github.com/xmrig/xmrig/pull/2932) Fixed GhostRider with hwloc disabled.

# v6.16.4
- [#2904](https://github.com/xmrig/xmrig/pull/2904) Fixed unaligned memory accesses.
- [#2908](https://github.com/xmrig/xmrig/pull/2908) Added MSVC/2022 to `version.h`.
- [#2910](https://github.com/xmrig/xmrig/issues/2910) Fixed donation for GhostRider/RTM.

# v6.16.3
- [#2778](https://github.com/xmrig/xmrig/pull/2778) Fixed `READY threads X/X` display after algorithm switching.
- [#2782](https://github.com/xmrig/xmrig/pull/2782) Updated GhostRider documentation.
- [#2815](https://github.com/xmrig/xmrig/pull/2815) Fixed `cn-heavy` in 32-bit builds.
- [#2827](https://github.com/xmrig/xmrig/pull/2827) GhostRider: set correct priority for helper threads.
- [#2837](https://github.com/xmrig/xmrig/pull/2837) RandomX: don't restart mining threads when the seed changes.
- [#2848](https://github.com/xmrig/xmrig/pull/2848) GhostRider: added support for `client.reconnect` method.
- [#2856](https://github.com/xmrig/xmrig/pull/2856) Fix for short responses from some Raptoreum pools.
- [#2873](https://github.com/xmrig/xmrig/pull/2873) Fixed GhostRider benchmark on single-core systems.
- [#2882](https://github.com/xmrig/xmrig/pull/2882) Fixed ARMv7 compilation.
- [#2893](https://github.com/xmrig/xmrig/pull/2893) KawPow OpenCL: use separate UV loop for building programs.

# v6.16.2
- [#2751](https://github.com/xmrig/xmrig/pull/2751) Fixed crash on CPUs supporting VAES and running GCC-compiled xmrig.
- [#2761](https://github.com/xmrig/xmrig/pull/2761) Fixed broken auto-tuning in GCC Windows build.
- [#2771](https://github.com/xmrig/xmrig/issues/2771) Fixed environment variables support for GhostRider and KawPow. 
- [#2769](https://github.com/xmrig/xmrig/pull/2769) Performance fixes:
  - Fixed several performance bottlenecks introduced in v6.16.1.
  - Fixed overall GCC-compiled build performance, it's the same speed as MSVC build now.
  - **Linux builds are up to 10% faster now compared to v6.16.0 GCC build.**
  - **Windows builds are up to 5% faster now compared to v6.16.0 MSVC build.**

# v6.16.1
- [#2729](https://github.com/xmrig/xmrig/pull/2729) GhostRider fixes:
  - Added average hashrate display.
  - Fixed the number of threads shown at startup.
  - Fixed `--threads` or `-t` command line option (but `--cpu-max-threads-hint` is recommended to use).
- [#2738](https://github.com/xmrig/xmrig/pull/2738) GhostRider fixes:
  - Fixed "difficulty is not a number" error when diff is high on some pools.
  - Fixed GhostRider compilation when `WITH_KAWPOW=OFF`.
- [#2740](https://github.com/xmrig/xmrig/pull/2740) Added VAES support for Cryptonight variants **+4% speedup on Zen3**.
  - VAES instructions are available on Intel Ice Lake/AMD Zen3 and newer CPUs.
  - +4% speedup on Ryzen 5 5600X.

# v6.16.0
- [#2712](https://github.com/xmrig/xmrig/pull/2712) **GhostRider algorithm (Raptoreum) support**: read the [RELEASE NOTES](src/crypto/ghostrider/README.md) for quick start guide and performance comparisons.
- [#2682](https://github.com/xmrig/xmrig/pull/2682) Fixed: use cn-heavy optimization only for Vermeer CPUs.
- [#2684](https://github.com/xmrig/xmrig/pull/2684) MSR mod: fix for error 183.

# v6.15.3
- [#2614](https://github.com/xmrig/xmrig/pull/2614) OpenCL fixes for non-AMD platforms.
- [#2623](https://github.com/xmrig/xmrig/pull/2623) Fixed compiling without kawpow.
- [#2636](https://github.com/xmrig/xmrig/pull/2636) [#2639](https://github.com/xmrig/xmrig/pull/2639) AstroBWT speedup (up to +35%).
- [#2646](https://github.com/xmrig/xmrig/pull/2646) Fixed MSVC compilation error.

# v6.15.2
- [#2606](https://github.com/xmrig/xmrig/pull/2606) Fixed: AstroBWT auto-config ignored `max-threads-hint`.
- Fixed possible crash on Windows (regression in v6.15.1).

# v6.15.1
- [#2586](https://github.com/xmrig/xmrig/pull/2586) Fixed Windows 7 compatibility.
- [#2594](https://github.com/xmrig/xmrig/pull/2594) Added Windows taskbar icon colors.

# v6.15.0
- [#2548](https://github.com/xmrig/xmrig/pull/2548) Added automatic coin detection for daemon mining.
- [#2563](https://github.com/xmrig/xmrig/pull/2563) Added new algorithm RandomX Graft (`rx/graft`).
- [#2565](https://github.com/xmrig/xmrig/pull/2565) AstroBWT: added AVX2 Salsa20 implementation.
- Added support for new CUDA plugin API (previous API still supported).

# v6.14.1
- [#2532](https://github.com/xmrig/xmrig/pull/2532) Refactoring: stable (persistent) algorithms IDs.
- [#2537](https://github.com/xmrig/xmrig/pull/2537) Fixed Termux build.

# v6.14.0
- [#2484](https://github.com/xmrig/xmrig/pull/2484) Added ZeroMQ support for solo mining.
- [#2476](https://github.com/xmrig/xmrig/issues/2476) Fixed crash in DMI memory reader.
- [#2492](https://github.com/xmrig/xmrig/issues/2492) Added missing `--huge-pages-jit` command line option.
- [#2512](https://github.com/xmrig/xmrig/pull/2512) Added show the number of transactions in pool job.

# v6.13.1
- [#2468](https://github.com/xmrig/xmrig/pull/2468) Fixed regression in previous version: don't send miner signature during regular mining.

# v6.13.0
- [#2445](https://github.com/xmrig/xmrig/pull/2445) Added support for solo mining with miner signatures for the upcoming Wownero fork.

# v6.12.2
- [#2280](https://github.com/xmrig/xmrig/issues/2280) GPU backends are now disabled in benchmark mode.
- [#2322](https://github.com/xmrig/xmrig/pull/2322) Improved MSR compatibility with recent Linux kernels and updated `randomx_boost.sh`.
- [#2340](https://github.com/xmrig/xmrig/pull/2340) Fixed AES detection on FreeBSD on ARM.
- [#2341](https://github.com/xmrig/xmrig/pull/2341) `sse2neon` updated to the latest version.
- [#2351](https://github.com/xmrig/xmrig/issues/2351) Fixed help output for `--cpu-priority` and `--cpu-affinity` option.
- [#2375](https://github.com/xmrig/xmrig/pull/2375) Fixed macOS CUDA backend default loader name.
- [#2378](https://github.com/xmrig/xmrig/pull/2378) Fixed broken light mode mining on x86.
- [#2379](https://github.com/xmrig/xmrig/pull/2379) Fixed CL code for KawPow where it assumes everything is AMD.
- [#2386](https://github.com/xmrig/xmrig/pull/2386) RandomX: enabled `IMUL_RCP` optimization for light mode mining.
- [#2393](https://github.com/xmrig/xmrig/pull/2393) RandomX: added BMI2 version for scratchpad prefetch.
- [#2395](https://github.com/xmrig/xmrig/pull/2395) RandomX: rewrote dataset read code.
- [#2398](https://github.com/xmrig/xmrig/pull/2398) RandomX: optimized ARMv8 dataset read.
- Added `argon2/ninja` alias for `argon2/wrkz` algorithm.

# v6.12.1
- [#2296](https://github.com/xmrig/xmrig/pull/2296) Fixed Zen3 assembly code for `cn/upx2` algorithm.

# v6.12.0
- [#2276](https://github.com/xmrig/xmrig/pull/2276) Added support for Uplexa (`cn/upx2` algorithm).
- [#2261](https://github.com/xmrig/xmrig/pull/2261) Show total hashrate if compiled without OpenCL.
- [#2289](https://github.com/xmrig/xmrig/pull/2289) RandomX: optimized `IMUL_RCP` instruction.
- Added support for `--user` command line option for online benchmark.

# v6.11.2
- [#2207](https://github.com/xmrig/xmrig/issues/2207) Fixed regression in HTTP parser and llhttp updated to v5.1.0.

# v6.11.1
- [#2239](https://github.com/xmrig/xmrig/pull/2239) Fixed broken `coin` setting functionality.

# v6.11.0
- [#2196](https://github.com/xmrig/xmrig/pull/2196) Improved DNS subsystem and added new DNS specific options.
- [#2172](https://github.com/xmrig/xmrig/pull/2172) Fixed build on Alpine 3.13.
- [#2177](https://github.com/xmrig/xmrig/pull/2177) Fixed ARM specific compilation error with GCC 10.2.
- [#2214](https://github.com/xmrig/xmrig/pull/2214) [#2216](https://github.com/xmrig/xmrig/pull/2216) [#2235](https://github.com/xmrig/xmrig/pull/2235) Optimized `cn-heavy` algorithm.
- [#2217](https://github.com/xmrig/xmrig/pull/2217) Fixed mining job creation sequence.
- [#2225](https://github.com/xmrig/xmrig/pull/2225) Fixed build without OpenCL support on some systems.
- [#2229](https://github.com/xmrig/xmrig/pull/2229) Don't use RandomX JIT if `WITH_ASM=OFF`.
- [#2228](https://github.com/xmrig/xmrig/pull/2228) Removed useless code for cryptonight algorithms.
- [#2234](https://github.com/xmrig/xmrig/pull/2234) Fixed build error on gcc 4.8.

# v6.10.0
- [#2122](https://github.com/xmrig/xmrig/pull/2122) Fixed pause logic when both pause on battery and user activity are enabled.
- [#2123](https://github.com/xmrig/xmrig/issues/2123) Fixed compatibility with gcc 4.8.
- [#2147](https://github.com/xmrig/xmrig/pull/2147) Fixed many `new job` messages when solo mining.
- [#2150](https://github.com/xmrig/xmrig/pull/2150) Updated `sse2neon.h` to the latest master, fixes build on ARMv7.
- [#2157](https://github.com/xmrig/xmrig/pull/2157) Fixed crash in `cn-heavy` on Zen3 with manual thread count.
- Fixed possible out of order write to log file.
- [http-parser](https://github.com/nodejs/http-parser) replaced to [llhttp](https://github.com/nodejs/llhttp).
- For official builds: libuv, hwloc and OpenSSL updated to latest versions.

# v6.9.0
- [#2104](https://github.com/xmrig/xmrig/pull/2104) Added [pause-on-active](https://xmrig.com/docs/miner/config/misc#pause-on-active) config option and `--pause-on-active=N` command line option.
- [#2112](https://github.com/xmrig/xmrig/pull/2112) Added support for [Tari merge mining](https://github.com/tari-project/tari/blob/development/README.md#tari-merge-mining).
- [#2117](https://github.com/xmrig/xmrig/pull/2117) Fixed crash when GPU mining `cn-heavy` on Zen3 system.

# v6.8.2
- [#2080](https://github.com/xmrig/xmrig/pull/2080) Fixed compile error in Termux.
- [#2089](https://github.com/xmrig/xmrig/pull/2089) Optimized CryptoNight-Heavy for Zen3, 7-8% speedup.

# v6.8.1
- [#2064](https://github.com/xmrig/xmrig/pull/2064) Added documentation for config.json CPU options.
- [#2066](https://github.com/xmrig/xmrig/issues/2066) Fixed AMD GPUs health data readings on Linux.
- [#2067](https://github.com/xmrig/xmrig/pull/2067) Fixed compilation error when RandomX and Argon2 are disabled.
- [#2076](https://github.com/xmrig/xmrig/pull/2076) Added support for flexible huge page sizes on Linux.
- [#2077](https://github.com/xmrig/xmrig/pull/2077) Fixed `illegal instruction` crash on ARM.

# v6.8.0
- [#2052](https://github.com/xmrig/xmrig/pull/2052) Added DMI/SMBIOS reader.
  - Added information about memory modules on the miner startup and for online benchmark.
  - Added new HTTP API endpoint: `GET /2/dmi`.
  - Added new command line option `--no-dmi` or config option `"dmi"`.
  - Added new CMake option `-DWITH_DMI=OFF`.
- [#2057](https://github.com/xmrig/xmrig/pull/2057) Improved MSR subsystem code quality.
- [#2058](https://github.com/xmrig/xmrig/pull/2058) RandomX JIT x86: removed unnecessary instructions.

# v6.7.2
- [#2039](https://github.com/xmrig/xmrig/pull/2039) Fixed solo mining.

# v6.7.1
- [#1995](https://github.com/xmrig/xmrig/issues/1995) Fixed log initialization.
- [#1998](https://github.com/xmrig/xmrig/pull/1998) Added hashrate in the benchmark finished message.
- [#2009](https://github.com/xmrig/xmrig/pull/2009) AstroBWT OpenCL fixes.
- [#2028](https://github.com/xmrig/xmrig/pull/2028) RandomX x86 JIT: removed redundant `CFROUND`.

# v6.7.0
- **[#1991](https://github.com/xmrig/xmrig/issues/1991) Added Apple M1 processor support.**
- **[#1986](https://github.com/xmrig/xmrig/pull/1986) Up to 20-30% faster RandomX dataset initialization with AVX2 on some CPUs.**
- [#1964](https://github.com/xmrig/xmrig/pull/1964) Cleanup and refactoring.
- [#1966](https://github.com/xmrig/xmrig/pull/1966) Removed libcpuid support.
- [#1968](https://github.com/xmrig/xmrig/pull/1968) Added virtual machine detection.
- [#1969](https://github.com/xmrig/xmrig/pull/1969) [#1970](https://github.com/xmrig/xmrig/pull/1970) Fixed errors found by static analysis.
- [#1977](https://github.com/xmrig/xmrig/pull/1977) Fixed: secure JIT and huge pages are incompatible on Windows.
- [#1979](https://github.com/xmrig/xmrig/pull/1979) Term `x64` replaced to `64-bit`.
- [#1980](https://github.com/xmrig/xmrig/pull/1980) Fixed build on gcc 11.
- [#1989](https://github.com/xmrig/xmrig/pull/1989) Fixed broken Dero solo mining.

# v6.6.2
- [#1958](https://github.com/xmrig/xmrig/pull/1958) Added example mining scripts to help new miners.
- [#1959](https://github.com/xmrig/xmrig/pull/1959) Optimized JIT compiler.
- [#1960](https://github.com/xmrig/xmrig/pull/1960) Fixed RandomX init when switching to other algo and back.

# v6.6.1
- Fixed, benchmark validation on NUMA hardware produced incorrect results in some conditions.

# v6.6.0
- Online benchmark protocol upgraded to v2, validation not compatible with previous versions.
  - Single thread benchmark now is cheat-resistant, not possible speedup it with multiple threads.
  - RandomX dataset is now always initialized with static seed, to prevent time cheat by report slow dataset initialization.
  - Zero delay online submission, to make time validation much more precise and strict.
  - DNS cache for online benchmark to prevent unexpected delays.

# v6.5.3
- [#1946](https://github.com/xmrig/xmrig/pull/1946) Fixed MSR mod names in JSON API (v6.5.2 affected).

# v6.5.2
- [#1935](https://github.com/xmrig/xmrig/pull/1935) Separate MSR mod for Zen/Zen2 and Zen3.
- [#1937](https://github.com/xmrig/xmrig/issues/1937) Print path to existing WinRing0 service without verbose option.
- [#1939](https://github.com/xmrig/xmrig/pull/1939) Fixed build with gcc 4.8.
- [#1941](https://github.com/xmrig/xmrig/pull/1941) Added CPUID info to JSON report.
- [#1941](https://github.com/xmrig/xmrig/pull/1942) Fixed alignment modification in memory pool.
- [#1944](https://github.com/xmrig/xmrig/pull/1944) Updated `randomx_boost.sh` with new MSR mod.
- Added `250K` and `500K` offline benchmarks.

# v6.5.1
- [#1932](https://github.com/xmrig/xmrig/pull/1932) New MSR mod for Ryzen, up to +3.5% on Zen2 and +1-2% on Zen3.
- [#1918](https://github.com/xmrig/xmrig/issues/1918) Fixed 1GB huge pages support on ARMv8.
- [#1926](https://github.com/xmrig/xmrig/pull/1926) Fixed compilation on ARMv8 with GCC 9.3.0.
- [#1929](https://github.com/xmrig/xmrig/issues/1929) Fixed build without HTTP.

# v6.5.0
- **Added [online benchmark](https://xmrig.com/benchmark) mode for sharing results.**
  - Added new command line options: `--submit`, `	--verify=ID`, `	--seed=SEED`, `--hash=HASH`.
- [#1912](https://github.com/xmrig/xmrig/pull/1912) Fixed MSR kernel module warning with new Linux kernels.
- [#1925](https://github.com/xmrig/xmrig/pull/1925) Add checking for config files in user home directory.
- Added vendor to ARM CPUs name and added `"arch"` field to API.
- Removed legacy CUDA plugin API.

# v6.4.0
- [#1862](https://github.com/xmrig/xmrig/pull/1862) **RandomX: removed `rx/loki` algorithm.**
- [#1890](https://github.com/xmrig/xmrig/pull/1890) **Added `argon2/chukwav2` algorithm.**
- [#1895](https://github.com/xmrig/xmrig/pull/1895) [#1897](https://github.com/xmrig/xmrig/pull/1897) **Added [benchmark and stress test](https://github.com/xmrig/xmrig/blob/dev/doc/BENCHMARK.md).**
- [#1864](https://github.com/xmrig/xmrig/pull/1864) RandomX: improved software AES performance.
- [#1870](https://github.com/xmrig/xmrig/pull/1870) RandomX: fixed unexpected resume due to disconnect during dataset init.
- [#1872](https://github.com/xmrig/xmrig/pull/1872) RandomX: fixed `randomx_create_vm` call.
- [#1875](https://github.com/xmrig/xmrig/pull/1875) RandomX: fixed crash on x86.
- [#1876](https://github.com/xmrig/xmrig/pull/1876) RandomX: added `huge-pages-jit` config parameter.
- [#1881](https://github.com/xmrig/xmrig/pull/1881) Fixed possible race condition in hashrate counting code.
- [#1882](https://github.com/xmrig/xmrig/pull/1882) [#1886](https://github.com/xmrig/xmrig/pull/1886) [#1887](https://github.com/xmrig/xmrig/pull/1887) [#1893](https://github.com/xmrig/xmrig/pull/1893) General code improvements.
- [#1885](https://github.com/xmrig/xmrig/pull/1885) Added more precise hashrate calculation.
- [#1889](https://github.com/xmrig/xmrig/pull/1889) Fixed libuv performance issue on Linux.

# v6.3.5
- [#1845](https://github.com/xmrig/xmrig/pull/1845) [#1861](https://github.com/xmrig/xmrig/pull/1861) Fixed ARM build and added CMake option `WITH_SSE4_1`.
- [#1846](https://github.com/xmrig/xmrig/pull/1846) KawPow: fixed OpenCL memory leak.
- [#1849](https://github.com/xmrig/xmrig/pull/1849) [#1859](https://github.com/xmrig/xmrig/pull/1859) RandomX: optimized soft AES code.
- [#1850](https://github.com/xmrig/xmrig/pull/1850) [#1852](https://github.com/xmrig/xmrig/pull/1852) General code improvements.
- [#1853](https://github.com/xmrig/xmrig/issues/1853) [#1856](https://github.com/xmrig/xmrig/pull/1856) [#1857](https://github.com/xmrig/xmrig/pull/1857) Fixed crash on old CPUs.

# v6.3.4
- [#1823](https://github.com/xmrig/xmrig/pull/1823) RandomX: added new option `scratchpad_prefetch_mode`.
- [#1827](https://github.com/xmrig/xmrig/pull/1827) [#1831](https://github.com/xmrig/xmrig/pull/1831) Improved nonce iteration performance.
- [#1828](https://github.com/xmrig/xmrig/pull/1828) RandomX: added SSE4.1-optimized Blake2b.
- [#1830](https://github.com/xmrig/xmrig/pull/1830) RandomX: added performance profiler (for developers).
- [#1835](https://github.com/xmrig/xmrig/pull/1835) RandomX: returned old soft AES implementation and added auto-select between the two.
- [#1840](https://github.com/xmrig/xmrig/pull/1840) RandomX: moved more stuff to compile time, small x86 JIT compiler speedup.
- [#1841](https://github.com/xmrig/xmrig/pull/1841) Fixed Cryptonight OpenCL for AMD 20.7.2 drivers.
- [#1842](https://github.com/xmrig/xmrig/pull/1842) RandomX: AES improvements, a bit faster hardware AES code when compiled with MSVC.
- [#1843](https://github.com/xmrig/xmrig/pull/1843) RandomX: improved performance of GCC compiled binaries.

# v6.3.3
- [#1817](https://github.com/xmrig/xmrig/pull/1817) Fixed self-select login sequence.
- Added brand new [build from source](https://xmrig.com/docs/miner/build) documentation.
- New binary downloads for macOS (`macos-x64`), FreeBSD (`freebsd-static-x64`), Linux (`linux-static-x64`), Ubuntu 18.04 (`bionic-x64`), Ubuntu 20.04 (`focal-x64`).
- Generic Linux download `xenial-x64` renamed to `linux-x64`.
- Builds without SSL/TLS support are no longer provided.
- Improved CUDA loader error reporting and fixed plugin load on Linux.
- Fixed build warnings with Clang compiler.
- Fixed colors on macOS.

# v6.3.2
- [#1794](https://github.com/xmrig/xmrig/pull/1794) More robust 1 GB pages handling.
  - Don't allocate 1 GB per thread if 1 GB is the default huge page size.
  - Try to allocate scratchpad from dataset's 1 GB huge pages, if normal huge pages are not available.
  - Correctly initialize RandomX cache if 1 GB pages fail to allocate on a first NUMA node.
- [#1806](https://github.com/xmrig/xmrig/pull/1806) Fixed macOS battery detection.
- [#1809](https://github.com/xmrig/xmrig/issues/1809) Improved auto configuration on ARM CPUs.
  - Added retrieving ARM CPU names, based on lscpu code and database.

# v6.3.1
- [#1786](https://github.com/xmrig/xmrig/pull/1786) Added `pause-on-battery` option, supported on Windows and Linux.
- Added command line options `--randomx-cache-qos` and `--argon2-impl`.

# v6.3.0
- [#1771](https://github.com/xmrig/xmrig/pull/1771) Adopted new SSE2NEON and reduced ARM-specific changes.
- [#1774](https://github.com/xmrig/xmrig/pull/1774) RandomX: Added new option `cache_qos` in `randomx` object for cache QoS support.
- [#1777](https://github.com/xmrig/xmrig/pull/1777) Added support for upcoming Haven offshore fork.
  - [#1780](https://github.com/xmrig/xmrig/pull/1780) CryptoNight OpenCL: fix for long input data.

# v6.2.3
- [#1745](https://github.com/xmrig/xmrig/pull/1745) AstroBWT: fixed OpenCL compilation on some systems.
- [#1749](https://github.com/xmrig/xmrig/pull/1749) KawPow: optimized CPU share verification.
- [#1752](https://github.com/xmrig/xmrig/pull/1752) RandomX: added error message when MSR mod fails.
- [#1754](https://github.com/xmrig/xmrig/issues/1754) Fixed GPU health readings for pre Vega GPUs on Linux.
- [#1756](https://github.com/xmrig/xmrig/issues/1756) Added results and connection reports.
- [#1759](https://github.com/xmrig/xmrig/pull/1759) KawPow: fixed DAG initialization on slower AMD GPUs.
- [#1763](https://github.com/xmrig/xmrig/pull/1763) KawPow: fixed rare duplicate share errors.
- [#1766](https://github.com/xmrig/xmrig/pull/1766) RandomX: small speedup on Ryzen CPUs.

# v6.2.2
- [#1742](https://github.com/xmrig/xmrig/issues/1742) Fixed crash when use HTTP API.

# v6.2.1
- [#1726](https://github.com/xmrig/xmrig/issues/1726) Fixed detection of AVX2/AVX512.
- [#1728](https://github.com/xmrig/xmrig/issues/1728) Fixed, 32 bit Windows builds was crash on start.
- [#1729](https://github.com/xmrig/xmrig/pull/1729) Fixed KawPow crash on old CPUs.
- [#1730](https://github.com/xmrig/xmrig/pull/1730) Improved displaying information for compute errors on GPUs.
- [#1732](https://github.com/xmrig/xmrig/pull/1732) Fixed NiceHash disconnects for KawPow.
- Fixed AMD GPU health (temperatures/power/clocks/fans) readings on Linux.

# v6.2.0-beta
- [#1717](https://github.com/xmrig/xmrig/pull/1717) Added new algorithm `cn/ccx` for Conceal.
- [#1718](https://github.com/xmrig/xmrig/pull/1718) Fixed, linker on Linux was marking entire executable as having an executable stack.
- [#1720](https://github.com/xmrig/xmrig/pull/1720) Fixed broken CryptoNight algorithms family with gcc 10.1.

# v6.0.1-beta
- [#1708](https://github.com/xmrig/xmrig/issues/1708) Added `title` option.
- [#1711](https://github.com/xmrig/xmrig/pull/1711) [cuda] Print errors from KawPow DAG initialization.
- [#1713](https://github.com/xmrig/xmrig/pull/1713) [cuda] Reduced memory usage for KawPow, minimum CUDA plugin version now is 6.1.0.

# v6.0.0-beta
- [#1694](https://github.com/xmrig/xmrig/pull/1694) Added support for KawPow algorithm (Ravencoin) on AMD/NVIDIA.
- Removed previously deprecated `cn/gpu` algorithm.
- Default donation level reduced to 1% but you still can increase it if you like.

# v5.11.3
- [#1718](https://github.com/xmrig/xmrig/pull/1718) Fixed, linker on Linux was marking entire executable as having an executable stack.
- [#1720](https://github.com/xmrig/xmrig/pull/1720) Fixed broken CryptoNight algorithms family with gcc 10.1.

# v5.11.2
- [#1664](https://github.com/xmrig/xmrig/pull/1664) Improved JSON config error reporting.
- [#1668](https://github.com/xmrig/xmrig/pull/1668) Optimized RandomX dataset initialization.
- [#1675](https://github.com/xmrig/xmrig/pull/1675) Fixed cross-compiling on Linux.
- Fixed memory leak in HTTP client.
- Build [dependencies](https://github.com/xmrig/xmrig-deps/releases/tag/v4.1) updated to recent versions.
- Compiler for Windows gcc builds updated to v10.1.

# v5.11.1
- [#1652](https://github.com/xmrig/xmrig/pull/1652) Up to 1% RandomX perfomance improvement on recent AMD CPUs.
- [#1306](https://github.com/xmrig/xmrig/issues/1306) Fixed possible double connection to a pool.
- [#1654](https://github.com/xmrig/xmrig/issues/1654) Fixed build with LibreSSL.

# v5.11.0
- **[#1632](https://github.com/xmrig/xmrig/pull/1632) Added AstroBWT CUDA support ([CUDA plugin](https://github.com/xmrig/xmrig-cuda) v3.0.0 or newer required).**
- [#1605](https://github.com/xmrig/xmrig/pull/1605) Fixed AstroBWT OpenCL for NVIDIA GPUs.
- [#1635](https://github.com/xmrig/xmrig/pull/1635) Added pooled memory allocation of RandomX VMs (+0.5% speedup on Zen2).
- [#1641](https://github.com/xmrig/xmrig/pull/1641) RandomX JIT refactoring, smaller memory footprint and a bit faster overall.
- [#1643](https://github.com/xmrig/xmrig/issues/1643) Fixed build on CentOS 7.

# v5.10.0
- [#1602](https://github.com/xmrig/xmrig/pull/1602) Added AMD GPUs support for AstroBWT algorithm.
- [#1590](https://github.com/xmrig/xmrig/pull/1590) MSR mod automatically deactivated after switching from RandomX algorithms.
- [#1592](https://github.com/xmrig/xmrig/pull/1592) Added AVX2 optimized code for AstroBWT algorithm.
  - Added new config option `astrobwt-avx2` in `cpu` object and command line option `--astrobwt-avx2`.
- [#1596](https://github.com/xmrig/xmrig/issues/1596) Major TLS (Transport Layer Security) subsystem update.
  - Added new TLS options, please check [xmrig-proxy documentation](https://xmrig.com/docs/proxy/tls) for details.
- `cn/gpu` algorithm now disabled by default and will be removed in next major (v6.x.x) release, no ETA for it right now.
- Added command line option `--data-dir`.

# v5.9.0
- [#1578](https://github.com/xmrig/xmrig/pull/1578) Added new RandomKEVA algorithm for upcoming Kevacoin fork, as `"algo": "rx/keva"` or `"coin": "keva"`.
- [#1584](https://github.com/xmrig/xmrig/pull/1584) Fixed invalid AstroBWT hashes after algorithm switching.
- [#1585](https://github.com/xmrig/xmrig/issues/1585) Fixed build without HTTP support.
- Added command line option `--astrobwt-max-size`.

# v5.8.2
- [#1580](https://github.com/xmrig/xmrig/pull/1580) AstroBWT algorithm 20-50% speedup.
  - Added new option `astrobwt-max-size`.
- [#1581](https://github.com/xmrig/xmrig/issues/1581) Fixed macOS build.

# v5.8.1
- [#1575](https://github.com/xmrig/xmrig/pull/1575) Fixed new block detection for DERO solo mining.

# v5.8.0
- [#1573](https://github.com/xmrig/xmrig/pull/1573) Added new AstroBWT algorithm for upcoming DERO fork, as `"algo": "astrobwt"` or `"coin": "dero"`.

# v5.7.0
- **Added SOCKS5 proxies support for Tor https://xmrig.com/docs/miner/tor.**
- [#377](https://github.com/xmrig/xmrig-proxy/issues/377) Fixed duplicate jobs in daemon (solo) mining client.
- [#1560](https://github.com/xmrig/xmrig/pull/1560) RandomX 0.3-0.4% speedup depending on CPU.
- Fixed possible crashes in HTTP client.

# v5.6.0
- [#1536](https://github.com/xmrig/xmrig/pull/1536) Added workaround for new AMD GPU drivers.
- [#1546](https://github.com/xmrig/xmrig/pull/1546) Fixed generic OpenCL code for AMD Navi GPUs.
- [#1551](https://github.com/xmrig/xmrig/pull/1551) Added RandomX JIT for AMD Navi  GPUs.
- Added health information for AMD GPUs (clocks/power/fan/temperature) via ADL (Windows) and sysfs (Linux).
- Fixed possible nicehash nonce overflow in some conditions.
- Fixed wrong OpenCL platform on macOS, option `platform` now ignored on this OS.

# v5.5.3
- [#1529](https://github.com/xmrig/xmrig/pull/1529) Fixed crash on Bulldozer CPUs.

# v5.5.2
- [#1500](https://github.com/xmrig/xmrig/pull/1500) Removed unnecessary code from RandomX JIT compiler.
- [#1502](https://github.com/xmrig/xmrig/pull/1502) Optimizations for AMD Bulldozer.
- [#1508](https://github.com/xmrig/xmrig/pull/1508) Added support for BMI2 instructions.
- [#1510](https://github.com/xmrig/xmrig/pull/1510) Optimized `CFROUND` instruction for RandomX.
- [#1520](https://github.com/xmrig/xmrig/pull/1520) Fixed thread affinity.

# v5.5.1
- [#1469](https://github.com/xmrig/xmrig/issues/1469) Fixed build with gcc 4.8.
- [#1473](https://github.com/xmrig/xmrig/pull/1473) Added RandomX auto-config for mobile Ryzen APUs.
- [#1477](https://github.com/xmrig/xmrig/pull/1477) Fixed build with Clang.
- [#1489](https://github.com/xmrig/xmrig/pull/1489) RandomX JIT compiler tweaks.
- [#1493](https://github.com/xmrig/xmrig/pull/1493) Default value for Intel MSR preset changed to `15`.
- Fixed unwanted resume after RandomX dataset change.

# v5.5.0
- [#179](https://github.com/xmrig/xmrig/issues/179) Added support for [environment variables](https://xmrig.com/docs/miner/environment-variables) in config file.
- [#1445](https://github.com/xmrig/xmrig/pull/1445) Removed `rx/v` algorithm.
- [#1453](https://github.com/xmrig/xmrig/issues/1453) Fixed crash on 32bit systems.
- [#1459](https://github.com/xmrig/xmrig/issues/1459) Fixed crash on very low memory systems.
- [#1465](https://github.com/xmrig/xmrig/pull/1465) Added fix for 1st-gen Ryzen crashes.
- [#1466](https://github.com/xmrig/xmrig/pull/1466) Added `cn-pico/tlo` algorithm.
- Added `--randomx-no-rdmsr` command line option.
- Added console title for Windows with miner name and version.
- On Windows `priority` option now also change base priority.

# v5.4.0
- [#1434](https://github.com/xmrig/xmrig/pull/1434) Added RandomSFX (`rx/sfx`) algorithm for Safex Cash.
- [#1445](https://github.com/xmrig/xmrig/pull/1445) Added RandomV (`rx/v`) algorithm for *new* MoneroV.
- [#1419](https://github.com/xmrig/xmrig/issues/1419) Added reverting MSR changes on miner exit, use `"rdmsr": false,` in `"randomx"` object to disable this feature.
- [#1423](https://github.com/xmrig/xmrig/issues/1423) Fixed conflicts with exists WinRing0 driver service.
- [#1425](https://github.com/xmrig/xmrig/issues/1425) Fixed crash on first generation Zen CPUs (MSR mod accidentally enable Opcache), additionally now you can disable Opcache and enable MSR mod via config `"wrmsr": ["0xc0011020:0x0", "0xc0011021:0x60", "0xc0011022:0x510000", "0xc001102b:0x1808cc16"],`.
- Added advanced usage for `wrmsr` option, for example: `"wrmsr": ["0x1a4:0x6"],` (Intel) and `"wrmsr": ["0xc0011020:0x0", "0xc0011021:0x40:0xffffffffffffffdf", "0xc0011022:0x510000", "0xc001102b:0x1808cc16"],` (Ryzen).
- Added new config option `"verbose"` and command line option `--verbose`.

# v5.3.0
- [#1414](https://github.com/xmrig/xmrig/pull/1414) Added native MSR support for Windows, by using signed **WinRing0 driver** (© 2007-2009 OpenLibSys.org).
- Added new [MSR documentation](https://xmrig.com/docs/miner/randomx-optimization-guide/msr).
- [#1418](https://github.com/xmrig/xmrig/pull/1418) Increased stratum send buffer size.

# v5.2.1
- [#1408](https://github.com/xmrig/xmrig/pull/1408) Added RandomX boost script for Linux (if you don't like run miner with root privileges).
- Added support for [AMD Ryzen MSR registers](https://www.reddit.com/r/MoneroMining/comments/e962fu/9526_hs_on_ryzen_7_3700x_xmrig_520_1gb_pages_msr/) (Linux only).
- Fixed command line option `--randomx-wrmsr` option without parameters.

# v5.2.0
- **[#1388](https://github.com/xmrig/xmrig/pull/1388) Added [1GB huge pages support](https://xmrig.com/docs/miner/hugepages#onegb-huge-pages) for Linux.**
  - Added new option `1gb-pages` in `randomx` object with command line equivalent `--randomx-1gb-pages`.
  - Added automatic huge pages configuration on Linux if use the miner with root privileges.
- **Added [automatic Intel prefetchers configuration](https://xmrig.com/docs/miner/randomx-optimization-guide#intel-specific-optimizations) on Linux.**
   - Added new option `wrmsr` in `randomx` object with command line equivalent `--randomx-wrmsr=6`.
- [#1396](https://github.com/xmrig/xmrig/pull/1396) [#1401](https://github.com/xmrig/xmrig/pull/1401) New performance optimizations for Ryzen CPUs. 
- [#1385](https://github.com/xmrig/xmrig/issues/1385) Added `max-threads-hint` option support for RandomX dataset initialization threads.  
- [#1386](https://github.com/xmrig/xmrig/issues/1386) Added `priority` option support for RandomX dataset initialization threads. 
- For official builds all dependencies (libuv, hwloc, openssl) updated to recent versions.
- Windows `msvc` builds now use Visual Studio 2019 instead of 2017.

# v5.1.1
- [#1365](https://github.com/xmrig/xmrig/issues/1365) Fixed various system response/stability issues.
  - Added new CPU option `yield` and command line equivalent `--cpu-no-yield`.
- [#1363](https://github.com/xmrig/xmrig/issues/1363) Fixed wrong priority of main miner thread.

# v5.1.0
- [#1351](https://github.com/xmrig/xmrig/pull/1351) RandomX optimizations and fixes.
  - Improved RandomX performance (up to +6-7% on Intel CPUs, +2-3% on Ryzen CPUs)
  - Added workaround for Intel JCC erratum bug see https://www.phoronix.com/scan.php?page=article&item=intel-jcc-microcode&num=1 for details.
  - Note! Always disable "Hardware prefetcher" and "Adjacent cacheline prefetch" in BIOS for Intel CPUs to get the optimal RandomX performance.
- [#1307](https://github.com/xmrig/xmrig/issues/1307) Fixed mining resume after donation round for pools with `self-select` feature.
- [#1318](https://github.com/xmrig/xmrig/issues/1318#issuecomment-559676080) Added option `"mode"` (or `--randomx-mode`) for RandomX.
  - Added memory information on miner startup.
  - Added `resources` field to summary API with memory information and load average.

# v5.0.1
- [#1234](https://github.com/xmrig/xmrig/issues/1234) Fixed compatibility with some AMD GPUs.
- [#1284](https://github.com/xmrig/xmrig/issues/1284) Fixed build without RandomX.
- [#1285](https://github.com/xmrig/xmrig/issues/1285) Added command line options `--cuda-bfactor-hint` and `--cuda-bsleep-hint`.
- [#1290](https://github.com/xmrig/xmrig/pull/1290) Fixed 32-bit ARM compilation.

# v5.0.0
This version is first stable unified 3 in 1 GPU+CPU release, OpenCL support built in in miner and not require additional external dependencies on compile time, NVIDIA CUDA available as external [CUDA plugin](https://github.com/xmrig/xmrig-cuda), for convenient, 3 in 1 downloads with recent CUDA version also provided.

This release based on 4.x.x series and include all features from v4.6.2-beta, changelog below include only the most important changes, [full changelog](doc/CHANGELOG_OLD.md) available separately.

- [#1272](https://github.com/xmrig/xmrig/pull/1272) Optimized hashrate calculation.
- [#1263](https://github.com/xmrig/xmrig/pull/1263) Added new option `dataset_host` for NVIDIA GPUs with less than 4 GB memory (RandomX only).
- [#1068](https://github.com/xmrig/xmrig/pull/1068) Added support for `self-select` stratum protocol extension.
- [#1227](https://github.com/xmrig/xmrig/pull/1227) Added new algorithm `rx/arq`, RandomX variant for upcoming ArQmA fork.
- [#808](https://github.com/xmrig/xmrig/issues/808#issuecomment-539297156) Added experimental support for persistent memory for CPU mining threads.
- [#1221](https://github.com/xmrig/xmrig/issues/1221) Improved RandomX dataset memory usage and initialization speed for NUMA machines.
- [#1175](https://github.com/xmrig/xmrig/issues/1175) Fixed support for systems where total count of NUMA nodes not equal usable nodes count.
- Added config option `cpu/max-threads-hint` and command line option `--cpu-max-threads-hint`.
- [#1185](https://github.com/xmrig/xmrig/pull/1185) Added JIT compiler for RandomX on ARMv8.
- Improved API endpoint `GET /2/backends` and added support for this endpoint to [workers.xmrig.info](http://workers.xmrig.info).
- Added command line option `--no-cpu` to disable CPU backend.
- Added OpenCL specific command line options: `--opencl`, `--opencl-devices`, `--opencl-platform`, `--opencl-loader` and `--opencl-no-cache`.
- Added CUDA specific command line options: `--cuda`, `--cuda-loader` and `--no-nvml`.
- Removed command line option `--http-enabled`, HTTP API enabled automatically if any other `--http-*` option provided.
- [#1172](https://github.com/xmrig/xmrig/issues/1172) **Added OpenCL mining backend.**
  - [#268](https://github.com/xmrig/xmrig-amd/pull/268) [#270](https://github.com/xmrig/xmrig-amd/pull/270) [#271](https://github.com/xmrig/xmrig-amd/pull/271) [#273](https://github.com/xmrig/xmrig-amd/pull/273) [#274](https://github.com/xmrig/xmrig-amd/pull/274) [#1171](https://github.com/xmrig/xmrig/pull/1171) Added RandomX support for OpenCL, thanks [@SChernykh](https://github.com/SChernykh).
- Algorithm `cn/wow` removed, as no longer alive. 

# Previous versions
[doc/CHANGELOG_OLD.md](doc/CHANGELOG_OLD.md)


# XMRig

[![Github All Releases](https://img.shields.io/github/downloads/xmrig/xmrig/total.svg)](https://github.com/xmrig/xmrig/releases)
[![GitHub release](https://img.shields.io/github/release/xmrig/xmrig/all.svg)](https://github.com/xmrig/xmrig/releases)
[![GitHub Release Date](https://img.shields.io/github/release-date/xmrig/xmrig.svg)](https://github.com/xmrig/xmrig/releases)
[![GitHub license](https://img.shields.io/github/license/xmrig/xmrig.svg)](https://github.com/xmrig/xmrig/blob/master/LICENSE)
[![GitHub stars](https://img.shields.io/github/stars/xmrig/xmrig.svg)](https://github.com/xmrig/xmrig/stargazers)
[![GitHub forks](https://img.shields.io/github/forks/xmrig/xmrig.svg)](https://github.com/xmrig/xmrig/network)

XMRig is a high performance, open source, cross platform RandomX, KawPow, CryptoNight and [GhostRider](https://github.com/xmrig/xmrig/tree/master/src/crypto/ghostrider#readme) unified CPU/GPU miner and [RandomX benchmark](https://xmrig.com/benchmark). Official binaries are available for Windows, Linux, macOS and FreeBSD.

## Mining backends
- **CPU** (x86/x64/ARMv7/ARMv8)
- **OpenCL** for AMD GPUs.
- **CUDA** for NVIDIA GPUs via external [CUDA plugin](https://github.com/xmrig/xmrig-cuda).

## Download
* **[Binary releases](https://github.com/xmrig/xmrig/releases)**
* **[Build from source](https://xmrig.com/docs/miner/build)**

## Usage
The preferred way to configure the miner is the [JSON config file](https://xmrig.com/docs/miner/config) as it is more flexible and human friendly. The [command line interface](https://xmrig.com/docs/miner/command-line-options) does not cover all features, such as mining profiles for different algorithms. Important options can be changed during runtime without miner restart by editing the config file or executing [API](https://xmrig.com/docs/miner/api) calls.

* **[Wizard](https://xmrig.com/wizard)** helps you create initial configuration for the miner.
* **[Workers](http://workers.xmrig.info)** helps manage your miners via HTTP API.

## Donations
* Default donation 1% (1 minute in 100 minutes) can be increased via option `donate-level` or disabled in source code.
* XMR: `48edfHu7V9Z84YzzMa6fUueoELZ9ZRXq9VetWzYGzKt52XU5xvqgzYnDK9URnRoJMk1j8nLwEVsaSWJ4fhdUyZijBGUicoD`

## Developers
* **[xmrig](https://github.com/xmrig)**
* **[sech1](https://github.com/SChernykh)**

## Contacts
* support@xmrig.com
* [reddit](https://www.reddit.com/user/XMRig/)
* [twitter](https://twitter.com/xmrig_dev)


---
name: Bug report
about: Create a report to help us improve
title: ''
labels: ''
assignees: ''

---

**Describe the bug**
A clear and concise description of what the bug is.

**To Reproduce**
Steps to reproduce the behavior.

**Expected behavior**
A clear and concise description of what you expected to happen.

**Required data**
 - Miner log as text or screenshot
 - Config file or command line (without wallets)
 - OS: [e.g. Windows]
 - For GPU related issues: information about GPUs and driver version.

**Additional context**
Add any other context about the problem here.


# Algorithms

Algorithm can be defined in 3 ways:

1. By pool, using algorithm negotiation, in this case no need specify algorithm on miner side.
2. Per pool `coin` option, currently only usable values for this option is `monero` and `arqma`.
3. Per pool `algo` option.

Option `coin` useful for pools without [algorithm negotiation](https://xmrig.com/docs/extensions/algorithm-negotiation) support or daemon to allow automatically switch algorithm in next hard fork. If you use xmrig-proxy don't need specify algorithm on miner side.

## Algorithm names

| Name | Memory | Version | Description | Notes |
|------|--------|---------|-------------|-------|
| `kawpow` | - | 6.0.0+ | KawPow (Ravencoin) | GPU only |
| `rx/keva` | 1 MB | 5.9.0+ | RandomKEVA (RandomX variant for Keva). |  |
| `astrobwt` | 20 MB | 5.8.0+ | AstroBWT (Dero). |  |
| `cn-pico/tlo` | 256 KB | 5.5.0+ | CryptoNight-Pico (Talleo). |  |
| `rx/sfx` | 2 MB | 5.4.0+ | RandomSFX (RandomX variant for Safex). |  |
| `rx/arq` | 256 KB | 4.3.0+ | RandomARQ (RandomX variant for ArQmA). |  |
| `rx/0` | 2 MB | 3.2.0+ | RandomX (Monero). |  |
| `argon2/chukwa` | 512 KB | 3.1.0+ | Argon2id (Chukwa). | CPU only |
| `argon2/wrkz` | 256 KB | 3.1.0+ | Argon2id (WRKZ) | CPU only |
| `rx/wow` | 1 MB | 3.0.0+ | RandomWOW (RandomX variant for Wownero). |  |
| `rx/loki` | 2 MB | 3.0.0+ | RandomXL (RandomX variant for Loki). |  |
| `cn/fast` | 2 MB | 3.0.0+ | CryptoNight variant 1 with half iterations. |  |
| `cn/rwz` | 2 MB | 2.14.0+ | CryptoNight variant 2 with 3/4 iterations and reversed shuffle operation. |  |
| `cn/zls` | 2 MB | 2.14.0+ | CryptoNight variant 2 with 3/4 iterations. |  |
| `cn/double` | 2 MB | 2.14.0+ | CryptoNight variant 2 with double iterations. |  |
| `cn/r` | 2 MB | 2.13.0+ | CryptoNightR (Monero's variant 4). |  |
| `cn-pico` | 256 KB | 2.10.0+ | CryptoNight-Pico. |  |
| `cn/half` | 2 MB | 2.9.0+ | CryptoNight variant 2 with half iterations. |  |
| `cn/2` | 2 MB | 2.8.0+ | CryptoNight variant 2. |  |
| `cn/xao` | 2 MB | 2.6.4+ | CryptoNight variant 0 (modified). |  |
| `cn/rto` | 2 MB | 2.6.4+ | CryptoNight variant 1 (modified). |  |
| `cn-heavy/tube` | 4 MB | 2.6.4+ | CryptoNight-Heavy (modified). |  |
| `cn-heavy/xhv` | 4 MB | 2.6.3+ | CryptoNight-Heavy (modified). |  |
| `cn-heavy/0` | 4 MB | 2.6.0+ | CryptoNight-Heavy. |  |
| `cn/1` | 2 MB | 2.5.0+ | CryptoNight variant 1. |  |
| `cn-lite/1` | 1 MB | 2.5.0+ | CryptoNight-Lite variant 1. |  |
| `cn-lite/0` | 1 MB | 0.8.0+ | CryptoNight-Lite variant 0. |  |
| `cn/0` | 2 MB | 0.5.0+ | CryptoNight (original). |  |

## Migration to v3
Since version 3 mining [algorithm](#algorithm-names) should specified for each pool separately (`algo` option), earlier versions was use one global `algo` option and per pool `variant` option (this option was removed in v3). If your pool support [mining algorithm negotiation](https://github.com/xmrig/xmrig-proxy/issues/168) you may not specify this option at all.
 
#### Example
```cppjson
{
  "pools": [
    {
      "url": "...",
      "algo": "cn/r",
      "coin": null
      ...
    }
 ],
 ...
}
```


# HTTP API

If you want use HTTP API you need enable it (`"enabled": true,`) then choice `port` and optionaly `host`. API not available if miner built without HTTP support (`-DWITH_HTTP=OFF`).

Offical HTTP client for API: http://workers.xmrig.info/

Example configuration:

```cppjson
"api": {
	"id": null,
	"worker-id": null,
},
"http": {
	"enabled": false,
	"host": "127.0.0.1",
	"port": 0,
	"access-token": null,
	"restricted": true
}
```

#### Global API options
* **id** Miner ID, if not set created automatically.
* **worker-id** Optional worker name, if not set will be detected automatically.

#### HTTP API options,
* **enabled** Enable (`true`) or disable (`false`) HTTP API.
* **host** Host for incoming connections `http://<host>:<port>`, to allow connections from all interfaces use `0.0.0.0` (IPv4) or `::` (IPv4+IPv6).
* **port** Port for incoming connections `http://<host>:<port>`, zero port is valid option and means random port.
* **access-token** [Bearer](https://gist.github.com/xmrig/c75fdd1f8e0f3bac05500be2ab718f8e#file-api-html-L54) access token to secure access to API. Miner support this token only via `Authorization` header.
* **restricted** Use `false` to allow remote configuration.

If you prefer use command line options instead of config file, you can use options: `--api-id`, `--api-worker-id`, `--http-enabled`, `--http-host`, `--http-access-token`, `--http-port`, `--http-no-restricted`.

Versions before 2.15 was use another options for API https://github.com/xmrig/xmrig/issues/1007

## Endpoints

### GET /1/summary

Get miner summary information. [Example](api/1/summary.json).

### GET /1/threads

Get detailed information about miner threads. [Example](api/1/threads.json).


## Restricted endpoints

All API endpoints below allow access to sensitive information and remote configure miner. You should set `access-token` and allow unrestricted access (`"restricted": false`).

### GET /1/config

Get current miner configuration. [Example](api/1/config.json).


### PUT /1/config

Update current miner configuration. Common use case, get current configuration, make changes, and upload it to miner.

Curl example:

```cpp
curl -v --data-binary @config.json -X PUT -H "Content-Type: application/json" -H "Authorization: Bearer SECRET" http://127.0.0.1:44444/1/config
```


# Embedded benchmark

You can run with XMRig with the following commands:
```cpp
xmrig --bench=1M
xmrig --bench=10M
xmrig --bench=1M -a rx/wow
xmrig --bench=10M -a rx/wow
```
This will run between 1 and 10 million RandomX hashes, depending on `bench` parameter, and print the time it took. First two commands use Monero variant (2 MB per thread, best for Zen2/Zen3 CPUs), second two commands use Wownero variant (1 MB per thread, useful for Intel and 1st gen Zen/Zen+ CPUs).

Checksum of all the hashes will be also printed to check stability of your hardware: if it's green then it's correct, if it's red then there was hardware error during computation. No Internet connection is required for the benchmark.

Double check that you see `Huge pages 100%` both for dataset and for all threads, and also check for `msr register values ... has been set successfully` - without this result will be far from the best. Running as administrator is required for MSR and huge pages to be set up properly.

![Benchmark example](https://i.imgur.com/PST3BYc.png)

### Benchmark with custom config

You can run benchmark with any configuration you want. Just start without command line parameteres, use regular config.json and add `"benchmark":"1M",` on the next line after pool url. 

# Stress test

You can also run continuous stress-test that is as close to the real RandomX mining as possible and doesn't require any configuration:
```cpp
xmrig --stress
xmrig --stress -a rx/wow
```
This will require Internet connection and will run indefinitely.

# v4.6.2-beta
- [#1274](https://github.com/xmrig/xmrig/issues/1274) Added `--cuda-devices` command line option.
- [#1277](https://github.com/xmrig/xmrig/pull/1277) Fixed function names for clang on Apple.

# v4.6.1-beta
- [#1272](https://github.com/xmrig/xmrig/pull/1272) Optimized hashrate calculation.
- [#1273](https://github.com/xmrig/xmrig/issues/1273) Fixed crash when use `GET /2/backends` API endpoint with disabled CUDA.

# v4.6.0-beta
- [#1263](https://github.com/xmrig/xmrig/pull/1263) Added new option `dataset_host` for NVIDIA GPUs with less than 4 GB memory (RandomX only).

# v4.5.0-beta
- Added NVIDIA CUDA support via external [CUDA plugun](https://github.com/xmrig/xmrig-cuda). XMRig now is unified 3 in 1 miner.

# v4.4.0-beta
- [#1068](https://github.com/xmrig/xmrig/pull/1068) Added support for `self-select` stratum protocol extension.
- [#1240](https://github.com/xmrig/xmrig/pull/1240) Sync with the latest RandomX code.
- [#1241](https://github.com/xmrig/xmrig/issues/1241) Fixed regression with colors on old Windows systems.
- [#1243](https://github.com/xmrig/xmrig/pull/1243) Fixed incorrect OpenCL memory size detection in some cases.
- [#1247](https://github.com/xmrig/xmrig/pull/1247) Fixed ARM64 RandomX code alignment.
- [#1248](https://github.com/xmrig/xmrig/pull/1248) Fixed RandomX code cache cleanup on iOS/Darwin.

# v4.3.1-beta
- Fixed regression in v4.3.0, miner didn't create `cn` mining profile with default config example.

# v4.3.0-beta
- [#1227](https://github.com/xmrig/xmrig/pull/1227) Added new algorithm `rx/arq`, RandomX variant for upcoming ArQmA fork.
- [#808](https://github.com/xmrig/xmrig/issues/808#issuecomment-539297156) Added experimental support for persistent memory for CPU mining threads.
- [#1221](https://github.com/xmrig/xmrig/issues/1221) Improved RandomX dataset memory usage and initialization speed for NUMA machines.

# v4.2.1-beta
- [#1150](https://github.com/xmrig/xmrig/issues/1150) Fixed build on FreeBSD.
- [#1175](https://github.com/xmrig/xmrig/issues/1175) Fixed support for systems where total count of NUMA nodes not equal usable nodes count.
- [#1199](https://github.com/xmrig/xmrig/issues/1199) Fixed excessive memory allocation for OpenCL threads with low intensity.
- [#1212](https://github.com/xmrig/xmrig/issues/1212) Fixed low RandomX performance after fast algorithm switching.

# v4.2.0-beta
- [#1202](https://github.com/xmrig/xmrig/issues/1202) Fixed algorithm verification in donate strategy.
- Added per pool option `coin` with single possible value `monero` for pools without algorithm negotiation, for upcoming Monero fork.
- Added config option `cpu/max-threads-hint` and command line option `--cpu-max-threads-hint`.

# v4.1.0-beta
- **OpenCL backend disabled by default.**.
- [#1183](https://github.com/xmrig/xmrig/issues/1183) Fixed compatibility with systemd.
- [#1185](https://github.com/xmrig/xmrig/pull/1185) Added JIT compiler for RandomX on ARMv8.
- Improved API endpoint `GET /2/backends` and added support for this endpoint to [workers.xmrig.info](http://workers.xmrig.info).
- Added command line option `--no-cpu` to disable CPU backend.
- Added OpenCL specific command line options: `--opencl`, `--opencl-devices`, `--opencl-platform`, `--opencl-loader` and `--opencl-no-cache`.
- Removed command line option `--http-enabled`, HTTP API enabled automatically if any other `--http-*` option provided.

# v4.0.1-beta
- [#1177](https://github.com/xmrig/xmrig/issues/1177) Fixed compatibility with old AMD drivers.
- [#1180](https://github.com/xmrig/xmrig/issues/1180) Fixed possible duplicated shares after algorithm switching.
- Added support for case if not all backend threads successfully started.
- Fixed wrong config file permissions after write (only gcc builds on recent Windows 10 affected).

# v4.0.0-beta
- [#1172](https://github.com/xmrig/xmrig/issues/1172) **Added OpenCL mining backend.**
  - [#268](https://github.com/xmrig/xmrig-amd/pull/268) [#270](https://github.com/xmrig/xmrig-amd/pull/270) [#271](https://github.com/xmrig/xmrig-amd/pull/271) [#273](https://github.com/xmrig/xmrig-amd/pull/273) [#274](https://github.com/xmrig/xmrig-amd/pull/274) [#1171](https://github.com/xmrig/xmrig/pull/1171) Added RandomX support for OpenCL, thanks [@SChernykh](https://github.com/SChernykh).
- Algorithm `cn/wow` removed, as no longer alive. 

# v3.2.0
- Added per pool option `coin` with single possible value `monero` for pools without algorithm negotiation, for upcoming Monero fork.
- [#1183](https://github.com/xmrig/xmrig/issues/1183) Fixed compatibility with systemd.

# v3.1.3
- [#1180](https://github.com/xmrig/xmrig/issues/1180) Fixed possible duplicated shares after algorithm switching.
- Fixed wrong config file permissions after write (only gcc builds on recent Windows 10 affected).

# v3.1.2
- Many RandomX optimizations and fixes.
  - [#1132](https://github.com/xmrig/xmrig/issues/1132) Fixed build on CentOS 7.
  - [#1163](https://github.com/xmrig/xmrig/pull/1163) Optimized soft AES code, up to +30% hashrate on CPU without AES support and other optimizations.
  - [#1166](https://github.com/xmrig/xmrig/pull/1166) Fixed crash when initialize dataset with big threads count (eg 272).
  - [#1168](https://github.com/xmrig/xmrig/pull/1168) Optimized loading from scratchpad.
- [#1128](https://github.com/xmrig/xmrig/issues/1128) Fixed CMake 2.8 compatibility.

# v3.1.1
- [#1133](https://github.com/xmrig/xmrig/issues/1133) Fixed syslog regression.
- [#1138](https://github.com/xmrig/xmrig/issues/1138) Fixed multiple network bugs.
- [#1141](https://github.com/xmrig/xmrig/issues/1141) Fixed log in background mode.
- [#1142](https://github.com/xmrig/xmrig/pull/1142) RandomX hashrate improved by 0.5-1.5% depending on variant and CPU.
- [#1146](https://github.com/xmrig/xmrig/pull/1146) Fixed race condition in RandomX thread init.
- [#1148](https://github.com/xmrig/xmrig/pull/1148) Fixed, on Linux linker marking entire executable as having an executable stack.
- Fixed, for Argon2 algorithms command line options like `--threads` was ignored.
- Fixed command line options for single pool, free order allowed again.

# v3.1.0
- [#1107](https://github.com/xmrig/xmrig/issues/1107#issuecomment-522235892) Added Argon2 algorithm family: `argon2/chukwa` and `argon2/wrkz`.

# v3.0.0
- **[#1111](https://github.com/xmrig/xmrig/pull/1111) Added RandomX (`rx/test`) algorithm for testing and benchmarking.**
- **[#1036](https://github.com/xmrig/xmrig/pull/1036) Added RandomWOW (`rx/wow`) algorithm for [Wownero](http://wownero.org/).**
- **[#1050](https://github.com/xmrig/xmrig/pull/1050) Added RandomXL (`rx/loki`) algorithm for [Loki](https://loki.network/).**
- **[#1077](https://github.com/xmrig/xmrig/issues/1077) Added NUMA support via hwloc**.
- **Added flexible [multi algorithm](doc/CPU.md) configuration.**
- **Added unlimited switching between incompatible algorithms, all mining options can be changed in runtime.**
- [#257](https://github.com/xmrig/xmrig-nvidia/pull/257) New logging subsystem, file and syslog now always without colors.
- [#314](https://github.com/xmrig/xmrig-proxy/issues/314) Added donate over proxy feature.
- [#1007](https://github.com/xmrig/xmrig/issues/1007) Old HTTP API backend based on libmicrohttpd, replaced to custom HTTP server (libuv + http_parser).
- [#1010](https://github.com/xmrig/xmrig/pull/1010#issuecomment-482632107) Added daemon support (solo mining).
- [#1066](https://github.com/xmrig/xmrig/issues/1066#issuecomment-518080529) Added error message if pool not ready for RandomX.
- [#1105](https://github.com/xmrig/xmrig/issues/1105) Improved auto configuration for `cn-pico` algorithm.
- Added commands `pause` and `resume` via JSON RPC 2.0 API (`POST /json_rpc`).
- Added command line option `--export-topology` for export hwloc topology to a XML file.
- Breaked backward compatibility with previous configs and command line, `variant` option replaced to `algo`, global option `algo` removed, all CPU related settings moved to `cpu` object.
- Options `av`, `safe` and `max-cpu-usage` removed.
- Algorithm `cn/msr` renamed to `cn/fast`.
- Algorithm `cn/xtl` removed.
- API endpoint `GET /1/threads` replaced to `GET /2/backends`.
- Added global uptime and extended connection information in API.
- API now return current algorithm.

# v2.99.6-beta
- Added commands `pause` and `resume` via JSON RPC 2.0 API (`POST /json_rpc`).
- Fixed autoconfig regression (since 2.99.5), mostly `rx/wow` was affected by this bug.
- Fixed user job recovery after donation round.
- Information about AVX2 CPU feature how hidden in miner summary.

# v2.99.5-beta
- [#1066](https://github.com/xmrig/xmrig/issues/1066#issuecomment-518080529) Fixed crash and added error message if pool not ready for RandomX.
- [#1092](https://github.com/xmrig/xmrig/issues/1092) Fixed crash if wrong CPU affinity used.
- [#1103](https://github.com/xmrig/xmrig/issues/1103) Improved auto configuration for RandomX for CPUs where L2 cache is limiting factor.
- [#1105](https://github.com/xmrig/xmrig/issues/1105) Improved auto configuration for `cn-pico` algorithm.
- [#1106](https://github.com/xmrig/xmrig/issues/1106) Fixed `hugepages` field in summary API. 
- Added alternative short format for CPU threads.
- Changed format for CPU threads with intensity above 1.
- Name for reference RandomX configuration changed to `rx/test` to avoid potential conflicts in future.

# v2.99.4-beta
- [#1062](https://github.com/xmrig/xmrig/issues/1062) Fixed 32 bit support. **32 bit is slow and deprecated**.
- [#1088](https://github.com/xmrig/xmrig/pull/1088) Fixed macOS compilation.
- [#1095](https://github.com/xmrig/xmrig/pull/1095) Fixed compatibility with hwloc 1.10.x.
- Optimized RandomX initialization and switching, fixed rare crash when re-initialize dataset.
- Fixed ARM build with hwloc.

# v2.99.3-beta
- [#1082](https://github.com/xmrig/xmrig/issues/1082) Fixed hwloc auto configuration on AMD FX CPUs.
- Added command line option `--export-topology` for export hwloc topology to a XML file.

# v2.99.2-beta
- [#1077](https://github.com/xmrig/xmrig/issues/1077) Added NUMA support via **hwloc**.
- Fixed miner freeze when switch between RandomX variants.
- Fixed dataset initialization speed on Linux if thread affinity was used.

# v2.99.1-beta
- [#1072](https://github.com/xmrig/xmrig/issues/1072) Fixed RandomX `seed_hash` re-initialization.

# v2.99.0-beta
- [#1050](https://github.com/xmrig/xmrig/pull/1050) Added RandomXL algorithm for [Loki](https://loki.network/), algorithm name used by miner is `randomx/loki` or `rx/loki`.
- Added [flexible](https://github.com/xmrig/xmrig/blob/evo/doc/CPU.md) multi algorithm configuration.
- Added unlimited switching between incompatible algorithms, all mining options can be changed in runtime.
- Breaked backward compatibility with previous configs and command line, `variant` option replaced to `algo`, global option `algo` removed, all CPU related settings moved to `cpu` object.
- Options `av`, `safe` and `max-cpu-usage` removed.
- Algorithm `cn/msr` renamed to `cn/fast`.
- Algorithm `cn/xtl` removed.
- API endpoint `GET /1/threads` replaced to `GET /2/backends`.

# v2.16.0-beta
- [#1036](https://github.com/xmrig/xmrig/pull/1036) Added RandomWOW (RandomX with different preferences) algorithm support for [Wownero](http://wownero.org/).
  - Algorithm name used by miner is `randomx/wow` or `rx/wow`.
  - Currently runtime algorithm switching NOT supported with other algorithms.

# v2.15.4-beta
- Added global uptime and extended connection information in API.
- API now return current algorithm instead of global algorithm specified in config.
- This version also include all changes from stable version v2.14.4.

# v2.15.3-beta
- [#1014](https://github.com/xmrig/xmrig/issues/1014) Fixed regression, default value for `algo` option was not applied.

# v2.15.2-beta
- [#1010](https://github.com/xmrig/xmrig/pull/1010#issuecomment-482632107) Added daemon support (solo mining).
- [#1012](https://github.com/xmrig/xmrig/pull/1012) Fixed compatibility with clang 9.
- Config subsystem was rewritten, internally JSON is primary format now.
- Fixed regression, big HTTP responses was truncated.

# v2.15.1-beta
- [#1007](https://github.com/xmrig/xmrig/issues/1007) Old HTTP API backend based on libmicrohttpd, replaced to custom HTTP server (libuv + http_parser).
- [#257](https://github.com/xmrig/xmrig-nvidia/pull/257) New logging subsystem, file and syslog now always without colors.

# v2.15.0-beta
- [#314](https://github.com/xmrig/xmrig-proxy/issues/314) Added donate over proxy feature.
  - Added new option `donate-over-proxy`.
  - Added real graceful exit.
  
# v2.14.4
- [#992](https://github.com/xmrig/xmrig/pull/992)  Fixed compilation with Clang 3.5.
- [#1012](https://github.com/xmrig/xmrig/pull/1012) Fixed compilation with Clang 9.0.
- In HTTP API for unknown hashrate now used `null` instead of `0.0`.
- Fixed MSVC 2019 version detection.
- Removed obsolete automatic variants.

# v2.14.1
* [#975](https://github.com/xmrig/xmrig/issues/975) Fixed crash on Linux if double thread mode used.

# v2.14.0
- **[#969](https://github.com/xmrig/xmrig/pull/969) Added new algorithm `cryptonight/rwz`, short alias `cn/rwz` (also known as CryptoNight ReverseWaltz), for upcoming [Graft](https://www.graft.network/) fork.**
- **[#931](https://github.com/xmrig/xmrig/issues/931) Added new algorithm `cryptonight/zls`, short alias `cn/zls` for [Zelerius Network](https://zelerius.org) fork.**
- **[#940](https://github.com/xmrig/xmrig/issues/940) Added new algorithm `cryptonight/double`, short alias `cn/double` (also known as CryptoNight HeavyX), for [X-CASH](https://x-cash.org/).**
- [#951](https://github.com/xmrig/xmrig/issues/951#issuecomment-469581529) Fixed crash if AVX was disabled on OS level.
- [#952](https://github.com/xmrig/xmrig/issues/952) Fixed compile error on some Linux.
- [#957](https://github.com/xmrig/xmrig/issues/957#issuecomment-468890667) Added support for embedded config.
- [#958](https://github.com/xmrig/xmrig/pull/958) Fixed incorrect user agent on ARM platforms.
- [#968](https://github.com/xmrig/xmrig/pull/968) Optimized `cn/r` algorithm performance.

# v2.13.1
- [#946](https://github.com/xmrig/xmrig/pull/946) Optimized software AES implementations for CPUs without hardware AES support. `cn/r`, `cn/wow` up to 2.6 times faster, 4-9% improvements for other algorithms.

# v2.13.0
- **[#938](https://github.com/xmrig/xmrig/issues/938) Added support for new algorithm `cryptonight/r`, short alias `cn/r` (also known as CryptoNightR or CryptoNight variant 4), for upcoming [Monero](https://www.getmonero.org/) fork on March 9, thanks [@SChernykh](https://github.com/SChernykh).**
- [#939](https://github.com/xmrig/xmrig/issues/939) Added support for dynamic (runtime) pools reload.
- [#932](https://github.com/xmrig/xmrig/issues/932) Fixed `cn-pico` hashrate drop, regression since v2.11.0.

# v2.12.0
- [#929](https://github.com/xmrig/xmrig/pull/929) Added support for new algorithm `cryptonight/wow`, short alias `cn/wow` (also known as CryptonightR), for upcoming [Wownero](http://wownero.org) fork on February 14.

# v2.11.0
- [#928](https://github.com/xmrig/xmrig/issues/928) Added support for new algorithm `cryptonight/gpu`, short alias `cn/gpu` (original name `cryptonight-gpu`), for upcoming [Ryo currency](https://ryo-currency.com) fork on February 14.
- [#749](https://github.com/xmrig/xmrig/issues/749) Added support for detect hardware AES in runtime on ARMv8 platforms.
- [#292](https://github.com/xmrig/xmrig/issues/292) Fixed build on ARMv8 platforms if compiler not support hardware AES.

# v2.10.0
- [#904](https://github.com/xmrig/xmrig/issues/904) Added new algorithm `cn-pico/trtl` (aliases `cryptonight-turtle`, `cn-trtl`) for upcoming TurtleCoin (TRTL) fork.
- Default value for option `max-cpu-usage` changed to `100` also this option now deprecated.

# v2.9.4
- [#913](https://github.com/xmrig/xmrig/issues/913) Fixed Masari (MSR) support (this update required for upcoming fork).
- [#915](https://github.com/xmrig/xmrig/pull/915) Improved security, JIT memory now read-only after patching.

# v2.9.3
- [#909](https://github.com/xmrig/xmrig/issues/909) Fixed compile errors on FreeBSD.
- [#912](https://github.com/xmrig/xmrig/pull/912) Fixed, C++ implementation of `cn/half` was produce up to 13% of invalid hashes.

# v2.9.2
- [#907](https://github.com/xmrig/xmrig/pull/907) Fixed crash on Linux.

# v2.9.1
- Restored compatibility with https://stellite.hashvault.pro.

# v2.9.0
- [#899](https://github.com/xmrig/xmrig/issues/899) Added support for new algorithm `cn/half` for Masari and Stellite forks.
- [#834](https://github.com/xmrig/xmrig/pull/834) Added ASM optimized code for AMD Bulldozer.
- [#839](https://github.com/xmrig/xmrig/issues/839) Fixed FreeBSD compile.
- [#857](https://github.com/xmrig/xmrig/pull/857) Fixed impossible to build for macOS without clang.

# v2.8.3
- [#813](https://github.com/xmrig/xmrig/issues/813) Fixed critical bug with Minergate pool and variant 2.

# v2.8.1
- [#768](https://github.com/xmrig/xmrig/issues/768) Fixed build with Visual Studio 2015.
- [#769](https://github.com/xmrig/xmrig/issues/769) Fixed regression, some ANSI escape sequences was in log with disabled colors.
- [#777](https://github.com/xmrig/xmrig/issues/777) Better report about pool connection issues. 
- Simplified checks for ASM auto detection, only AES support necessary.
- Added missing options to `--help` output.

# v2.8.0
- **[#753](https://github.com/xmrig/xmrig/issues/753) Added new algorithm [CryptoNight variant 2](https://github.com/xmrig/xmrig/issues/753) for Monero fork, thanks [@SChernykh](https://github.com/SChernykh).**
  - Added global and per thread option `"asm"` and and command line equivalent.
- **[#758](https://github.com/xmrig/xmrig/issues/758) Added SSL/TLS support for secure connections to pools.**
  - Added per pool options `"tls"` and `"tls-fingerprint"` and command line equivalents.
- [#767](https://github.com/xmrig/xmrig/issues/767) Added config autosave feature, same with GPU miners.  
- [#245](https://github.com/xmrig/xmrig-proxy/issues/245) Fixed API ID collision when run multiple miners on same machine.
- [#757](https://github.com/xmrig/xmrig/issues/757) Fixed send buffer overflow.

# v2.6.4
- [#700](https://github.com/xmrig/xmrig/issues/700) `cryptonight-lite/ipbc` replaced to `cryptonight-heavy/tube` for **Bittube (TUBE)**.
- Added `cryptonight/rto` (cryptonight variant 1 with IPBC/TUBE mod) variant for **Arto (RTO)** coin.
- Added `cryptonight/xao` (original cryptonight with bigger iteration count) variant for **Alloy (XAO)** coin.
- Better variant detection for **nicehash.com** and **minergate.com**.
- [#692](https://github.com/xmrig/xmrig/issues/692) Added support for specify both algorithm and variant via single `algo` option.

# v2.6.3
- **Added support for new cryptonight-heavy variant xhv** (`cn-heavy/xhv`) for upcoming Haven Protocol fork.
- **Added support for new cryptonight variant msr** (`cn/msr`) also known as `cryptonight-fast` for upcoming Masari fork.
- Added new detailed hashrate report.
- [#446](https://github.com/xmrig/xmrig/issues/446) Likely fixed SIGBUS error on 32 bit ARM CPUs.
- [#551](https://github.com/xmrig/xmrig/issues/551) Fixed `cn-heavy` algorithm on ARMv8.
- [#614](https://github.com/xmrig/xmrig/issues/614) Fixed display issue with huge pages percentage when colors disabled.
- [#615](https://github.com/xmrig/xmrig/issues/615) Fixed build without libcpuid.
- [#629](https://github.com/xmrig/xmrig/pull/629) Fixed file logging with non-seekable files.
- [#672](https://github.com/xmrig/xmrig/pull/672) Reverted back `cryptonight-light` and exit if no valid algorithm specified.

# v2.6.2
 - [#607](https://github.com/xmrig/xmrig/issues/607) Fixed donation bug.
 - [#610](https://github.com/xmrig/xmrig/issues/610) Fixed ARM build.

# v2.6.1
 - [#168](https://github.com/xmrig/xmrig-proxy/issues/168) Added support for [mining algorithm negotiation](https://github.com/xmrig/xmrig-proxy/blob/dev/doc/STRATUM_EXT.md#1-mining-algorithm-negotiation).
 - Added IPBC coin support, base algorithm `cn-lite` variant `ipbc`.
 - [#581](https://github.com/xmrig/xmrig/issues/581) Added support for upcoming Stellite (XTL) fork, base algorithm `cn` variant `xtl`, variant can set now, no need do it after fork.
 - Added support for **rig-id** stratum protocol extensions, compatible with xmr-stak.
 - Changed behavior for option `variant=-1` for `cryptonight`, now variant is `1` by default, if you mine old coins need change `variant` to `0`.
 - A lot of small fixes and better unification with proxy code.

# v2.6.0-beta3
- [#563](https://github.com/xmrig/xmrig/issues/563) **Added [advanced threads mode](https://github.com/xmrig/xmrig/issues/563), now possible configure each thread individually.**
- [#255](https://github.com/xmrig/xmrig/issues/563) Low power mode extended to **triple**, **quard** and **penta** modes.
- [#519](https://github.com/xmrig/xmrig/issues/519) Fixed high donation levels, improved donation start time randomization.
- [#554](https://github.com/xmrig/xmrig/issues/554) Fixed regression with `print-time` option.

# v2.6.0-beta2
- Improved performance for `cryptonight v7` especially in double hash mode.
- [#499](https://github.com/xmrig/xmrig/issues/499) IPv6 disabled for internal HTTP API by default, was causing issues on some systems.
- Added short aliases for algorithm names: `cn`, `cn-lite` and `cn-heavy`.
- Fixed regressions (v2.6.0-beta1 affected)
  - [#494](https://github.com/xmrig/xmrig/issues/494) Command line option `--donate-level` was broken.
  - [#502](https://github.com/xmrig/xmrig/issues/502) Build without libmicrohttpd was broken.
  - Fixed nonce calculation for `--av 4` (software AES, double hash) was causing reduction of effective hashrate and rejected shares on nicehash.

# v2.6.0-beta1
 - [#476](https://github.com/xmrig/xmrig/issues/476) **Added Cryptonight-Heavy support for Sumokoin ASIC resistance fork.**
 - HTTP server now runs in main loop, it make possible easy extend API without worry about thread synchronization.
 - Added initial graceful reload support, miner will reload configuration if config file changed, disabled by default until it will be fully implemented and tested.
 - Added API endpoint `PUT /1/config` to update current config.
 - Added API endpoint `GET /1/config` to get current active config.
 - Added API endpoint `GET /1/threads` to get current active threads configuration.
 - API endpoint `GET /` now deprecated, use `GET /1/summary` instead.
 - Added `--api-no-ipv6` and similar config option to disable IPv6 support for HTTP API.
 - Added `--api-no-restricted` to enable full access to api, this option has no effect if `--api-access-token` not specified.

# v2.5.3
- Fixed critical bug, in some cases miner was can't recovery connection and switch to failover pool, version 2.5.2 affected. If you use v2.6.0-beta3 this issue doesn't concern you.
- [#499](https://github.com/xmrig/xmrig/issues/499) IPv6 support disabled for internal HTTP API.
- Added workaround for nicehash.com if you use `cryptonightv7.<region>.nicehash.com` option `variant=1` will be set automatically.

# v2.5.2
- [#448](https://github.com/xmrig/xmrig/issues/478) Fixed broken reconnect.

# v2.5.1
- [#454](https://github.com/xmrig/xmrig/issues/454) Fixed build with libmicrohttpd version below v0.9.35.
- [#456](https://github.com/xmrig/xmrig/issues/459) Verbose errors related to donation pool was not fully silenced.
- [#459](https://github.com/xmrig/xmrig/issues/459) Fixed regression (version 2.5.0 affected) with connection to **xmr.f2pool.com**.

# v2.5.0
- [#434](https://github.com/xmrig/xmrig/issues/434) **Added support for Monero v7 PoW, scheduled on April 6.**
- Added full IPv6 support.
- Added protocol extension, when use the miner with xmrig-proxy 2.5+ no more need manually specify `nicehash` option.
- [#123](https://github.com/xmrig/xmrig-proxy/issues/123) Fixed regression (all versions since 2.4 affected) fragmented responses from pool/proxy was parsed incorrectly.
- [#428](https://github.com/xmrig/xmrig/issues/428) Fixed regression (version 2.4.5 affected) with CPU cache size detection.

# v2.4.5
- [#324](https://github.com/xmrig/xmrig/pull/324) Fixed build without libmicrohttpd (CMake cache issue).
- [#341](https://github.com/xmrig/xmrig/issues/341) Fixed wrong exit code and added command line option `--dry-run`.
- [#385](https://github.com/xmrig/xmrig/pull/385) Up to 20% performance increase for non-AES CPU and fixed Intel Core 2 cache detection.

# v2.4.4
 - Added libmicrohttpd version to --version output.
 - Fixed bug in singal handler, in some cases miner wasn't shutdown properly.
 - Fixed recent MSVC 2017 version detection.
 - [#279](https://github.com/xmrig/xmrig/pull/279) Fixed build on some macOS versions.

# v2.4.3
 - [#94](https://github.com/xmrig/xmrig/issues/94#issuecomment-342019257) [#216](https://github.com/xmrig/xmrig/issues/216) Added **ARMv8** and **ARMv7** support. Hardware AES supported, thanks [Imran Yusuff](https://github.com/imranyusuff).
 - [#157](https://github.com/xmrig/xmrig/issues/157) [#196](https://github.com/xmrig/xmrig/issues/196) Fixed Linux compile issues.
 - [#184](https://github.com/xmrig/xmrig/issues/184) Fixed cache size detection for CPUs with disabled Hyper-Threading.
 - [#200](https://github.com/xmrig/xmrig/issues/200) In some cases miner was doesn't write log to stdout.

# v2.4.2
 - [#60](https://github.com/xmrig/xmrig/issues/60) Added FreeBSD support, thanks [vcambur](https://github.com/vcambur).
 - [#153](https://github.com/xmrig/xmrig/issues/153) Fixed issues with dwarfpool.com.
 
# v2.4.1
  - [#147](https://github.com/xmrig/xmrig/issues/147) Fixed comparability with monero-stratum.

# v2.4.0
 - Added [HTTP API](https://github.com/xmrig/xmrig/wiki/API).
 - Added comments support in config file.
 - libjansson replaced to rapidjson.
 - [#98](https://github.com/xmrig/xmrig/issues/98) Ignore `keepalive` option with minergate.com and nicehash.com.
 - [#101](https://github.com/xmrig/xmrig/issues/101) Fixed MSVC 2017 (15.3) compile time version detection.
 - [#108](https://github.com/xmrig/xmrig/issues/108) Silently ignore invalid values for `donate-level` option.
 - [#111](https://github.com/xmrig/xmrig/issues/111) Fixed build without AEON support.
 
# v2.3.1
- [#68](https://github.com/xmrig/xmrig/issues/68) Fixed compatibility with Docker containers, was nothing print on console.

# v2.3.0
- Added `--cpu-priority` option (0 idle, 2 normal to 5 highest).
- Added `--user-agent` option, to set custom user-agent string for pool. For example `cpuminer-multi/0.1`.
- Added `--no-huge-pages` option, to disable huge pages support.
- [#62](https://github.com/xmrig/xmrig/issues/62) Don't send the login to the dev pool.
- Force reconnect if pool block miner IP address. helps switch to backup pool.
- Fixed: failed open default config file if path contains non English characters.
- Fixed: error occurred if try use unavailable stdin or stdout, regression since version 2.2.0.
- Fixed: message about huge pages support successfully enabled on Windows was not shown in release builds.

# v2.2.1
- Fixed [terminal issues](https://github.com/xmrig/xmrig-proxy/issues/2#issuecomment-319914085) after exit on Linux and OS X.

# v2.2.0
- [#46](https://github.com/xmrig/xmrig/issues/46) Restored config file support. Now possible use multiple config files and combine with command line options also added support for default config.
- Improved colors support on Windows, now used uv_tty, legacy code removed.
- QuickEdit Mode now disabled on Windows.
- Added interactive commands in console window:: **h**ashrate, **p**ause, **r**esume.
- Fixed autoconf mode for AMD FX CPUs.

# v2.1.0
- [#40](https://github.com/xmrig/xmrig/issues/40)
Improved miner shutdown, fixed crash on exit for Linux and OS X.
- Fixed, login request was contain malformed JSON if username or password has some special characters for example `\`. 
- [#220](https://github.com/fireice-uk/xmr-stak-cpu/pull/220) Better support for Round Robin DNS, IP address now always chosen randomly instead of stuck on first one.
- Changed donation address, new [xmrig-proxy](https://github.com/xmrig/xmrig-proxy) is coming soon.

# v2.0.2
- Better deal with possible duplicate jobs from pool, show warning and ignore duplicates.
- For Windows builds libuv updated to version 1.13.1 and gcc to 7.1.0.

# v2.0.1
 - [#27](https://github.com/xmrig/xmrig/issues/27) Fixed possibility crash on 32bit systems.

# v2.0.0
 - Option `--backup-url` removed, instead now possibility specify multiple pools for example: `-o example1.com:3333 -u user1 -p password1 -k -o example2.com:5555 -u user2 -o example3.com:4444 -u user3`
 - [#15](https://github.com/xmrig/xmrig/issues/15) Added option `-l, --log-file=FILE` to write log to file.
 - [#15](https://github.com/xmrig/xmrig/issues/15) Added option `-S, --syslog` to use syslog for logging, Linux only.
 - [#18](https://github.com/xmrig/xmrig/issues/18) Added nice messages for accepted/rejected shares with diff and network latency.
 - [#20](https://github.com/xmrig/xmrig/issues/20) Fixed `--cpu-affinity` for more than 32 threads.
 - Fixed Windows XP support.
 - Fixed regression, option `--no-color` was not fully disable colored output.
 - Show resolved pool IP address in miner output.
 
# v1.0.1
- Fix broken software AES implementation, app has crashed if CPU not support AES-NI, only version 1.0.0 affected.

# v1.0.0
- Miner complete rewritten in C++ with libuv.
- This version should be fully compatible (except config file) with previos versions, many new nice features will come in next versions.
- This is still beta. If you found regression, stability or perfomance issues or have an idea for new feature please fell free to open new [issue](https://github.com/xmrig/xmrig/issues/new).
- Added new option `--print-time=N`, print hashrate report every N seconds.
- New hashrate reports, by default every 60 secons.
- Added Microsoft Visual C++ 2015 and 2017 support.
- Removed dependency on libcurl.
- To compile this version from source please switch to [dev](https://github.com/xmrig/xmrig/tree/dev) branch.

# v0.8.2
- Fixed L2 cache size detection for AMD CPUs (Bulldozer/Piledriver/Steamroller/Excavator architecture).

# v0.8.2
- Fixed L2 cache size detection for AMD CPUs (Bulldozer/Piledriver/Steamroller/Excavator architecture).
- Fixed gcc 7.1 support.

# v0.8.1
- Added nicehash support, detects automaticaly by pool URL, for example `cryptonight.eu.nicehash.com:3355` or manually via option `--nicehash`.

# v0.8.0
- Added double hash mode, also known as lower power mode. `--av=2` and `--av=4`.
- Added smart automatic CPU configuration. Default threads count now depends on size of the L3 cache of CPU.
- Added CryptoNight-Lite support for AEON `-a cryptonight-lite`.
- Added `--max-cpu-usage` option for auto CPU configuration mode.
- Added `--safe` option for adjust threads and algorithm variations to current CPU.
- No more manual steps to enable huge pages on Windows. XMRig will do it automatically.
- Removed BMI2 algorithm variation.
- Removed default pool URL.

# v0.6.0
- Added automatic cryptonight self test.
- New software AES algorithm variation. Will be automatically selected if cpu not support AES-NI.
- Added 32 bit builds.
- Documented [algorithm variations](https://github.com/xmrig/xmrig#algorithm-variations).

# v0.5.0
- Initial public release.


**:warning: Recent version of this page https://xmrig.com/docs/miner/config/cpu.**

# CPU backend

All CPU related settings contains in one `cpu` object in config file, CPU backend allow specify multiple profiles and allow switch between them without restrictions by pool request or config change. Default auto-configuration create reasonable minimum of profiles which cover all supported algorithms.

### Example

Example below demonstrate all primary ideas of flexible profiles configuration:

* `"rx/wow"` Exact match to algorithm `rx/wow`, defined 4 threads without CPU affinity.
* `"cn"` Default failback profile for all `cn/*` algorithms, defined 2 threads with CPU affinity, another failback profiles is `cn-lite`, `cn-heavy` and `rx`.
* `"cn-lite"` Default failback profile for all `cn-lite/*` algorithms, defined 2 double threads with CPU affinity.
* `"cn-pico"` Alternative short object format.
* `"custom-profile"` Custom user defined profile.
* `"*"` Failback profile for all unhandled by other profiles algorithms.
* `"cn/r"` Exact match, alias to profile `custom-profile`.
* `"cn/0"` Exact match, disabled algorithm.

```cppjson
{
    "cpu": {
        "enabled": true,
        "huge-pages": true,
        "hw-aes": null,
        "priority": null,
        "asm": true,
        "rx/wow": [-1, -1, -1, -1],
        "cn": [
            [1, 0],
            [1, 2]
        ],
        "cn-lite": [
            [2, 0],
            [2, 2]
        ],
        "cn-pico": {
            "intensity": 2,
            "threads": 8,
            "affinity": -1
        },
        "custom-profile": [0, 2],
        "*": [-1],
        "cn/r": "custom-profile",
        "cn/0": false
    }
}
```

## Threads definition
Threads can be defined in 3 formats.

#### Array format
```cppjson
[
    [1, 0],
    [1, 2],
    [1, -1],
    [2, -1]
]
```
Each line represent one thread, first element is intensity, this option was known as `low_power_mode`, possible values is range from 1 to 5, second element is CPU affinity, special value `-1` means no affinity.

#### Short array format
```cppjson
[-1, -1, -1, -1]
```
Each number represent one thread and means CPU affinity, this is default format for algorithm with maximum intensity 1, currently it all RandomX variants and cryptonight-gpu.

#### Short object format
```cppjson
{
    "intensity": 2,
    "threads": 8,
    "affinity": -1
}
```
Internal format, but can be user defined.

## RandomX options

#### `init`
Thread count to initialize RandomX dataset. Auto-detect (`-1`) or any number greater than 0 to use that many threads.

#### `init-avx2`
Use AVX2 for dataset initialization. Faster on some CPUs. Auto-detect (`-1`), disabled (`0`), always enabled on CPUs that support AVX2 (`1`).

#### `mode`
RandomX mining mode: `auto`, `fast` (2 GB memory), `light` (256 MB memory).

#### `1gb-pages`
Use 1GB hugepages for RandomX dataset (Linux only). Enabled (`true`) or disabled (`false`). It gives 1-3% speedup.

#### `wrmsr`
[MSR mod](https://xmrig.com/docs/miner/randomx-optimization-guide/msr). Enabled (`true`) or disabled (`false`). It gives up to 15% speedup depending on your system. _(**Note**: Userspace MSR writes are no longer enabled by default; the flag `msr.allow_writes=on` must be set for Linux Kernels 5.9 and after.)_

#### `rdmsr`
Restore MSR register values to their original values on exit. Used together with `wrmsr`. Enabled (`true`) or disabled (`false`).

#### `cache_qos`
[Cache QoS](https://xmrig.com/docs/miner/randomx-optimization-guide/qos). Enabled (`true`) or disabled (`false`). It's useful when you can't or don't want to mine on all CPU cores to make mining hashrate more stable.

#### `numa`
NUMA support (better hashrate on multi-CPU servers and Ryzen Threadripper 1xxx/2xxx). Enabled (`true`) or disabled (`false`).

#### `scratchpad_prefetch_mode`
Which instruction to use in RandomX loop to prefetch data from scratchpad. `1` is default and fastest in most cases. Can be off (`0`), `prefetcht0` instruction (`1`), `prefetchnta` instruction (`2`, a bit faster on Coffee Lake and a few other CPUs), `mov` instruction (`3`).

## Shared options

#### `enabled`
Enable (`true`) or disable (`false`) CPU backend, by default `true`.

#### `huge-pages`
Enable (`true`) or disable (`false`) huge pages support, by default `true`.

#### `huge-pages-jit`
Enable (`true`) or disable (`false`) huge pages support for RandomX JIT code, by default `false`. It gives a very small boost on Ryzen CPUs, but hashrate is unstable between launches. Use with caution.

#### `hw-aes`
Force enable (`true`) or disable (`false`) hardware AES support. Default value `null` means miner autodetect this feature. Usually don't need change this option, this option useful for some rare cases when miner can't detect hardware AES, but it available. If you force enable this option, but your hardware not support it, miner will crash.

#### `priority`
Mining threads priority, value from `1` (lowest priority) to `5` (highest possible priority). Default value `null` means miner don't change threads priority at all. Setting priority higher than 2 can make your PC unresponsive.

#### `memory-pool` (since v4.3.0)
Use continuous, persistent memory block for mining threads, useful for preserve huge pages allocation while algorithm switching. Possible values `false` (feature disabled, by default) or `true` or specific count of 2 MB huge pages. It helps to avoid loosing huge pages for scratchpads when RandomX dataset is updated and mining threads restart after a 2-3 days of mining.

#### `yield` (since v5.1.1)
Prefer system better system response/stability `true` (default value) or maximum hashrate `false`.

#### `asm`
Enable/configure or disable ASM optimizations. Possible values: `true`, `false`, `"intel"`, `"ryzen"`, `"bulldozer"`.

#### `argon2-impl` (since v3.1.0)
Allow override automatically detected Argon2 implementation, this option added mostly for debug purposes, default value `null` means autodetect. This is used in RandomX dataset initialization and also in some other mining algorithms. Other possible values: `"x86_64"`, `"SSE2"`, `"SSSE3"`, `"XOP"`, `"AVX2"`, `"AVX-512F"`. Manual selection has no safe guards - if your CPU doesn't support required instuctions, miner will crash.

#### `astrobwt-max-size`
AstroBWT algorithm: skip hashes with large stage 2 size, default: `550`, min: `400`, max: `1200`. Optimal value depends on your CPU/GPU

#### `astrobwt-avx2`
AstroBWT algorithm: use AVX2 code. It's faster on some CPUs and slower on other

#### `max-threads-hint` (since v4.2.0)
Maximum CPU threads count (in percentage) hint for autoconfig. [CPU_MAX_USAGE.md](CPU_MAX_USAGE.md)


# Maximum CPU usage

Please read this document carefully, `max-threads-hint` (was known as `max-cpu-usage`) option is most confusing option in the miner with many myth and legends.
This option is just hint for automatic configuration and can't precise define CPU usage.

### Option definition
#### Config file:
```cppjson
{
    ...
    "cpu": {
        "max-threads-hint": 100,
        ...
    },
    ...
}
```

#### Command line
`--cpu-max-threads-hint 100`

### Known issues and usage

* This option has no effect if miner already generated CPU configuration, to prevent config generation use `"autosave":false,`.
* Only threads count can be changed, for 1 core CPU this option has no effect, for 2 core CPU only 2 values possible 50% and 100%, for 4 cores: 25%, 50%, 75%, 100%. etc. 
* You CPU may limited by other factors, eg cache.


# Persistent options

Options in list below can't changed in runtime by watching config file or via API.

* `background`
* `donate-level`
* `cpu/argon2-impl`
* `opencl/loader`
* `opencl/platform`


# CMake options
**Recent version of this document: https://xmrig.com/docs/miner/cmake-options**

## Algorithms

* **`-DWITH_CN_LITE=OFF`** disable all CryptoNight-Lite algorithms (`cn-lite/0`, `cn-lite/1`).
* **`-DWITH_CN_HEAVY=OFF`** disable all CryptoNight-Heavy algorithms (`cn-heavy/0`, `cn-heavy/xhv`, `cn-heavy/tube`).
* **`-DWITH_CN_PICO=OFF`** disable CryptoNight-Pico algorithm (`cn-pico`).
* **`-DWITH_RANDOMX=OFF`** disable RandomX algorithms (`rx/loki`, `rx/wow`).
* **`-DWITH_ARGON2=OFF`** disable Argon2 algorithms (`argon2/chukwa`, `argon2/wrkz`).

## Features

* **`-DWITH_HWLOC=OFF`**
disable [hwloc](https://github.com/xmrig/xmrig/issues/1077) support.
Disabling this feature is not recommended in most cases.
This feature add external dependency to libhwloc (1.10.0+) (except MSVC builds).
* **`-DWITH_LIBCPUID=OFF`** disable built in libcpuid support, this feature always disabled if hwloc enabled, if both hwloc and libcpuid disabled auto configuration for CPU will very limited.
* **`-DWITH_HTTP=OFF`** disable built in HTTP support, this feature used for HTTP API and daemon (solo mining) support.
* **`-DWITH_TLS=OFF`** disable SSL/TLS support (secure connections to pool). This feature add external dependency to OpenSSL.
* **`-DWITH_ASM=OFF`** disable assembly optimizations for modern CryptoNight algorithms.
* **`-DWITH_EMBEDDED_CONFIG=ON`** Enable [embedded](https://github.com/xmrig/xmrig/issues/957) config support.
* **`-DWITH_OPENCL=OFF`** Disable OpenCL backend.
* **`-DWITH_CUDA=OFF`** Disable CUDA backend.
* **`-DWITH_SSE4_1=OFF`** Disable SSE 4.1 for Blake2 (useful for arm builds).

## Debug options

* **`-DWITH_DEBUG_LOG=ON`** enable debug log (mostly network requests).
* **`-DHWLOC_DEBUG=ON`** enable some debug log for hwloc.
* **`-DCMAKE_BUILD_TYPE=Debug`** enable debug build, only useful for investigate crashes, this option slow down miner.

## Special build options

* **`-DXMRIG_DEPS=<path>`** path to precompiled dependencies https://github.com/xmrig/xmrig-deps
* **`-DARM_TARGET=<number>`** override ARM target, possible values `7` (ARMv7) and `8` (ARMv8).
* **`-DUV_INCLUDE_DIR=<path>`** custom path to libuv headers.
* **`-DUV_LIBRARY=<path>`** custom path to libuv library.
* **`-DHWLOC_INCLUDE_DIR=<path>`** custom path to hwloc headers.
* **`-DHWLOC_LIBRARY=<path>`** custom path to hwloc library.
* **`-DOPENSSL_ROOT_DIR=<path>`** custom path to OpenSSL.


# `scripts/generate_cl.js`

该代码是一个 Node.js 脚本，使用了 Node.js 的 `fs` 模块和 `path` 模块，以及从 `./js/opencl` 和 `./js/opencl_minify` 模块中引入的 `text2h` 和 `opencl_minify` 函数。

该脚本的主要目的是定义了一个名为 `cn` 的函数。函数内部使用 `opencl_minify` 函数将多个 OpenGL ES 算法的源代码打包成一个压缩后的二进制文件，并将其写入一个名为 `cryptonight_cl.h` 的文件中。同时，在函数内部使用了 `fs.writeFileSync` 函数将打包好的二进制文件写入一个名为 `cryptonight_gen.cl` 的文件中。

由于 `opencl_minify` 函数需要一个参数 `addIncludes`，该参数是一个数组，其中包含了需要包含在 `opencl_minify` 函数中的 OpenGL ES 算法。在该脚本中，该参数被设置为 `[
       'algorithm.cl',
       'wolf-aes.cl',
       'wolf-skein.cl',
       'jh.cl',
       'blake256.cl',
       'groestl256.cl',
       'fast_int_math_v2.cl',
       'fast_div_heavy.cl',
       'keccak.cl'
   ]`，表示将上述算法加入到 `opencl_minify` 函数的打包中。


```cpp
#!/usr/bin/env node

'use strict';

const fs = require('fs');
const path = require('path');
const { text2h, text2h_bundle, addIncludes } = require('./js/opencl');
const { opencl_minify } = require('./js/opencl_minify');
const cwd = process.cwd();


function cn()
{
    const cn = opencl_minify(addIncludes('cryptonight.cl', [
        'algorithm.cl',
        'wolf-aes.cl',
        'wolf-skein.cl',
        'jh.cl',
        'blake256.cl',
        'groestl256.cl',
        'fast_int_math_v2.cl',
        'fast_div_heavy.cl',
        'keccak.cl'
    ]));

    // fs.writeFileSync('cryptonight_gen.cl', cn);
    fs.writeFileSync('cryptonight_cl.h', text2h(cn, 'xmrig', 'cryptonight_cl'));
}


```

这是一个 JavaScript 函数，名为 `cn_r`。函数的作用是生成一个名为 `cryptonight_r_cl.h` 的头文件，同时包含一个名为 `cryptonight_r_defines.cl` 的 OpenCL 定义文件和一个名为 `cryptonight_r.cl` 的 OpenCL 安装程序。

函数内部创建了一个名为 `items` 的对象，用于存储生成器（generator）。

对于 `items.cryptonight_r_defines.cl`，函数使用 `opencl_minify` 函数将 `cryptonight_r_defines.cl` 中的内容进行打包压缩，并输出为一个名为 `cryptonight_r_defines.cl` 的 OpenCL 定义文件。函数使用 `opencl_minify` 函数将 `cryptonight_r.cl` 中的内容进行打包压缩，并输出为一个名为 `cryptonight_r.cl` 的 OpenCL 安装程序。

函数内部还有一个循环，用于将 `items` 对象中的每个键（key）的值存储为一个名为 `key_gen.cl` 的 OpenCL 定义文件。在循环内部，函数使用 `fs.writeFileSync` 函数将 `items[key]` 的值写入一个名为 `key_gen.cl` 的文件中。

最后，函数使用 `text2h_bundle` 函数将 `items` 对象打包成一个名为 `cryptonight_r_cl.h` 的头文件。函数返回头文件的内容。


```cpp
function cn_r()
{
    const items = {};

    items.cryptonight_r_defines_cl = opencl_minify(addIncludes('cryptonight_r_defines.cl', [ 'wolf-aes.cl' ]));
    items.cryptonight_r_cl         = opencl_minify(fs.readFileSync('cryptonight_r.cl', 'utf8'));

    // for (let key in items) {
    //      fs.writeFileSync(key + '_gen.cl', items[key]);
    // }

    fs.writeFileSync('cryptonight_r_cl.h', text2h_bundle('xmrig', items));
}


```

这是一段 TypeScript 代码，定义了一个名为 `rx` 的函数。函数内部创建了一个名为 `rx` 的异步流对象，通过调用 `addIncludes` 函数将多个 Clion 项目文件中的依赖项添加到 `rx` 的元数据中。

`addIncludes` 函数是一个 Clover 的插件，用于将其他项目中的头文件包含到当前项目中。在这个例子中，头文件包括：`randomx.cl`、`../cn/algorithm.cl`、`randomx_constants_monero.h`、`randomx_constants_wow.h`、`randomx_constants_arqma.h`、`randomx_constants_keva.h`、`randomx_constants_graft.h`、`aes.cl`、`blake2b.cl`、`randomx_vm.cl` 和 `randomx_jit.cl`。

接下来，函数内部通过 `replace` 函数替换了以下内容：

```cpp
//fs.readFileSync('fillAes1Rx4.cl', 'utf8');
//fs.readFileSync('blake2b_double_block.cl', 'utf8');
```

这两行代码读取了 `fillAes1Rx4.cl` 和 `blake2b_double_block.cl` 两个文件的内容，并将它们的内容存储到了变量 `rx` 的元数据中。

最后，函数还创建了一个名为 `randomx_gen.cl` 的文件，并将 `rx` 的内容写入其中。


```cpp
function rx()
{
    let rx = addIncludes('randomx.cl', [
        '../cn/algorithm.cl',
        'randomx_constants_monero.h',
        'randomx_constants_wow.h',
        'randomx_constants_arqma.h',
        'randomx_constants_keva.h',
        'randomx_constants_graft.h',
        'aes.cl',
        'blake2b.cl',
        'randomx_vm.cl',
        'randomx_jit.cl'
    ]);

    rx = rx.replace(/(\t| )*#include "fillAes1Rx4.cl"/g, fs.readFileSync('fillAes1Rx4.cl', 'utf8'));
    rx = rx.replace(/(\t| )*#include "blake2b_double_block.cl"/g, fs.readFileSync('blake2b_double_block.cl', 'utf8'));
    rx = opencl_minify(rx);

    //fs.writeFileSync('randomx_gen.cl', rx);
    fs.writeFileSync('randomx_cl.h', text2h(rx, 'xmrig', 'randomx_cl'));
}


```

该函数 `kawpow()` 的作用是定义了一个名为 `kawpow` 的函数。函数内部使用了多种 Cluser 语言特性，包括 OpenCL 指令、Clang 语义分析和类型推导等。

具体来说，函数定义了一个名为 `kawpow` 的函数，该函数会在定义时包含两个文件：`kawpow.cl` 和 `kawpow_dag.cl`。这两个文件分别定义了 OpenCL 指令函数体和定义函数体。函数还定义了一个文件：`kawpow_gen.cl`，该文件是这两个定义文件的哈希值。

此外，函数的实现还引入了 `opencl_minify`，这是一个 Cluser 工具，用于处理 OpenCL 指令中的语法错误和警告，并对其进行简化。

最后，函数将当前工作目录设置为 `src/backend/opencl/cl/cn`，并调用自身，即 `kawpow()` 函数本身。


```cpp
function kawpow()
{
    const kawpow = opencl_minify(addIncludes('kawpow.cl', [ 'defs.h' ]));
    const kawpow_dag = opencl_minify(addIncludes('kawpow_dag.cl', [ 'defs.h' ]));

    // fs.writeFileSync('kawpow_gen.cl', kawpow);
    fs.writeFileSync('kawpow_cl.h', text2h(kawpow, 'xmrig', 'kawpow_cl'));
    fs.writeFileSync('kawpow_dag_cl.h', text2h(kawpow_dag, 'xmrig', 'kawpow_dag_cl'));
}


process.chdir(path.resolve('src/backend/opencl/cl/cn'));

cn();
cn_r();

```

这段代码使用了 Node.js 的 `process.chdir` 函数来改变工作目录。

具体来说，第一行将当前工作目录（`cwd`）更改为给定的目录（`path.resolve('src/backend/opencl/cl/rx')`）。

接着，第二行再次使用 `process.chdir` 函数将工作目录更改为给定的目录（`path.resolve('src/backend/opencl/cl/kawpow')`）。

最后，第三行调用了一个名为 `rx` 的函数，它是一个接受管道流（Stream）的回调函数。由于没有提供具体的回调函数实现，我们无法知道这个函数处理了哪些数据或操作了哪些文件。


```cpp
process.chdir(cwd);
process.chdir(path.resolve('src/backend/opencl/cl/rx'));

rx();

process.chdir(cwd);
process.chdir(path.resolve('src/backend/opencl/cl/kawpow'));

kawpow();

```