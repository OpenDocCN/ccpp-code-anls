# `xmrig\scripts\js\opencl.js`

```
'use strict';

const fs = require('fs');


function bin2h(buf, namespace, name)
{
    // 计算缓冲区的字节长度
    const size = buf.byteLength;
    // 初始化输出字符串
    let out    = `#pragma once\n\nnamespace ${namespace} {\n\nstatic const unsigned char ${name}[${size}] = {\n    `;

    // 每32个字节输出一行
    let b = 32;
    // 遍历缓冲区的每个字节
    for (let i = 0; i < size; i++) {
        // 将字节转换为十六进制，并添加到输出字符串中
        out += `0x${buf.readUInt8(i).toString(16).padStart(2, '0')}${size - i > 1 ? ',' : ''}`;

        // 每输出32个字节换行
        if (--b === 0) {
            b = 32;
            out += '\n    ';
        }
    }

    // 结尾添加分号和右大括号
    out += `\n};\n\n} // namespace ${namespace}\n`;

    return out;
}


function text2h_internal(text, name)
{
    // 将文本转换为缓冲区
    const buf  = Buffer.from(text);
    // 计算缓冲区的字节长度
    const size = buf.byteLength;
    // 初始化输出字符串
    let out    = `\nstatic const char ${name}[${size + 1}] = {\n    `;

    // 每32个字节输出一行
    let b = 32;
    // 遍历缓冲区的每个字节
    for (let i = 0; i < size; i++) {
        // 将字节转换为十六进制，并添加到输出字符串中
        out += `0x${buf.readUInt8(i).toString(16).padStart(2, '0')},`;

        // 每输出32个字节换行
        if (--b === 0) {
            b = 32;
            out += '\n    ';
        }
    }

    // 添加字符串结束符
    out += '0x00';

    // 结尾添加分号和右大括号
    out += '\n};\n';

    return out;
}


function text2h(text, namespace, name)
{
    // 返回包含文本转换为C++代码的字符串
    return `#pragma once\n\nnamespace ${namespace} {\n` + text2h_internal(text, name) + `\n} // namespace ${namespace}\n`;
}


function text2h_bundle(namespace, items)
{
    // 初始化输出字符串
    let out = `#pragma once\n\nnamespace ${namespace} {\n`;

    // 遍历文本项，将每个文本转换为C++代码并添加到输出字符串中
    for (let key in items) {
        out += text2h_internal(items[key], key);
    }

    // 结尾添加右大括号
    return out + `\n} // namespace ${namespace}\n`;
}


function addInclude(input, name)
{
    // 读取指定文件的内容，并替换输入字符串中的#include语句
    return input.replace(`#include "${name}"`, fs.readFileSync(name, 'utf8'));
}


function addIncludes(inputFileName, names)
{
    // 读取输入文件的内容
    let data = fs.readFileSync(inputFileName, 'utf8');

    // 遍历要包含的文件名，依次替换输入文件中的#include语句
    for (let name of names) {
        data = addInclude(data, name);
    }

    return data;
}


module.exports.bin2h         = bin2h;
module.exports.text2h        = text2h;
module.exports.text2h_bundle = text2h_bundle;
module.exports.addInclude    = addInclude;
module.exports.addIncludes   = addIncludes;
```