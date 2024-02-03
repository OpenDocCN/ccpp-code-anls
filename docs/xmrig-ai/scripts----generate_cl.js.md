# `xmrig\scripts\generate_cl.js`

```cpp
#!/usr/bin/env node
'use strict';

const fs = require('fs');  // 导入文件系统模块
const path = require('path');  // 导入路径模块
const { text2h, text2h_bundle, addIncludes } = require('./js/opencl');  // 导入自定义的 opencl 模块中的函数
const { opencl_minify } = require('./js/opencl_minify');  // 导入自定义的 opencl_minify 函数
const cwd = process.cwd();  // 获取当前工作目录路径

function cn()
{
    const cn = opencl_minify(addIncludes('cryptonight.cl', [  // 调用 addIncludes 函数，将 'cryptonight.cl' 和其他文件名组成的数组作为参数
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

    // fs.writeFileSync('cryptonight_gen.cl', cn);  // 将 cn 内容写入文件 'cryptonight_gen.cl'
    fs.writeFileSync('cryptonight_cl.h', text2h(cn, 'xmrig', 'cryptonight_cl'));  // 将 cn 内容转换成头文件格式后写入文件 'cryptonight_cl.h'
}

function cn_r()
{
    const items = {};

    items.cryptonight_r_defines_cl = opencl_minify(addIncludes('cryptonight_r_defines.cl', [ 'wolf-aes.cl' ]));  // 调用 addIncludes 函数，将 'cryptonight_r_defines.cl' 和 'wolf-aes.cl' 组成的数组作为参数
    items.cryptonight_r_cl = opencl_minify(fs.readFileSync('cryptonight_r.cl', 'utf8'));  // 读取 'cryptonight_r.cl' 文件内容并进行压缩

    // for (let key in items) {
    //      fs.writeFileSync(key + '_gen.cl', items[key]);  // 将 items 对象中的内容写入文件
    // }

    fs.writeFileSync('cryptonight_r_cl.h', text2h_bundle('xmrig', items));  // 将 items 对象中的内容转换成头文件格式后写入文件 'cryptonight_r_cl.h'
}

function rx()
{
    let rx = addIncludes('randomx.cl', [  // 调用 addIncludes 函数，将 'randomx.cl' 和其他文件名组成的数组作为参数
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

    rx = rx.replace(/(\t| )*#include "fillAes1Rx4.cl"/g, fs.readFileSync('fillAes1Rx4.cl', 'utf8'));  // 用 'fillAes1Rx4.cl' 文件内容替换 rx 中的指定内容
    rx = rx.replace(/(\t| )*#include "blake2b_double_block.cl"/g, fs.readFileSync('blake2b_double_block.cl', 'utf8'));  // 用 'blake2b_double_block.cl' 文件内容替换 rx 中的指定内容
    rx = opencl_minify(rx);  // 对 rx 内容进行压缩

    //fs.writeFileSync('randomx_gen.cl', rx);  // 将 rx 内容写入文件 'randomx_gen.cl'
    fs.writeFileSync('randomx_cl.h', text2h(rx, 'xmrig', 'randomx_cl'));  // 将 rx 内容转换成头文件格式后写入文件 'randomx_cl.h'
}

function kawpow()
{
    const kawpow = opencl_minify(addIncludes('kawpow.cl', [ 'defs.h' ]));  // 调用 addIncludes 函数，将 'kawpow.cl' 和 'defs.h' 组成的数组作为参数
}
    // 读取并压缩 kawpow_dag.cl 文件，然后将其内容赋给 kawpow_dag 变量
    const kawpow_dag = opencl_minify(addIncludes('kawpow_dag.cl', [ 'defs.h' ]));

    // 将 kawpow 变量的内容写入到 kawpow_cl.h 文件中
    fs.writeFileSync('kawpow_cl.h', text2h(kawpow, 'xmrig', 'kawpow_cl'));
    // 将 kawpow_dag 变量的内容写入到 kawpow_dag_cl.h 文件中
    fs.writeFileSync('kawpow_dag_cl.h', text2h(kawpow_dag, 'xmrig', 'kawpow_dag_cl'));
# 切换当前工作目录到指定路径
process.chdir(path.resolve('src/backend/opencl/cl/cn'));
# 调用 cn 函数
cn();
# 调用 cn_r 函数
cn_r();
# 切换当前工作目录到之前保存的路径
process.chdir(cwd);
# 切换当前工作目录到指定路径
process.chdir(path.resolve('src/backend/opencl/cl/rx'));
# 调用 rx 函数
rx();
# 切换当前工作目录到之前保存的路径
process.chdir(cwd);
# 切换当前工作目录到指定路径
process.chdir(path.resolve('src/backend/opencl/cl/kawpow'));
# 调用 kawpow 函数
kawpow();
```