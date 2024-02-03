# `xmrig\scripts\js\opencl_minify.js`

```cpp
'use strict';

// 对输入的字符串进行压缩处理
function opencl_minify(input)
{
    // 去除输入字符串中的回车符
    let out = input.replace(/\r/g, '');
    // 去除注释
    out = out.replace(/\/\*[\s\S]*?\*\/|\/\/.*$/gm, ''); // comments
    // 去除宏定义中的多余空格
    out = out.replace(/^#\s+/gm, '#');        // macros with spaces
    // 去除多余的空行
    out = out.replace(/\n{2,}/g, '\n');       // empty lines
    // 去除每行开头的空白字符
    out = out.replace(/^\s+/gm, '');          // leading whitespace
    // 去除多余的空格
    out = out.replace(/ {2,}/g, ' ');         // extra whitespace

    // 将处理后的字符串按行分割成数组，并对每行进行处理
    let array = out.split('\n').map(line => {
        // 如果行以 # 开头，则直接返回
        if (line[0] === '#') {
            return line;
        }

        // 替换行中的逗号后的空格为逗号
        line = line.replace(/, /g, ',');
        // 替换行中的空格和问号为问号
        line = line.replace(/ \? /g, '?');
        // 替换行中的空格和冒号为冒号
        line = line.replace(/ : /g, ':');
        // 替换行中的空格和等号为等号
        line = line.replace(/ = /g, '=');
        // 替换行中的空格、感叹号和等号为不等号
        line = line.replace(/ != /g, '!=');
        // ... 其他替换规则

        return line;
    });

    // 将处理后的数组合并成字符串并返回
    return array.join('\n');
}

// 导出 opencl_minify 函数
module.exports.opencl_minify = opencl_minify;
```