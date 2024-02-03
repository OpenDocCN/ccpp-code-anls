# `nmap\libz\contrib\iostream2\zstream_test.cpp`

```cpp
#include "zstream.h" // 包含自定义的 zstream 头文件
#include <math.h> // 包含数学函数库
#include <stdlib.h> // 包含标准库函数
#include <iomanip.h> // 包含输入输出格式控制库

void main() {
    char h[256] = "Hello"; // 定义并初始化字符数组 h
    char* g = "Goodbye"; // 定义指向字符串常量的指针 g
    ozstream out("temp.gz"); // 创建输出流对象 out，指定输出文件名为 temp.gz
    out < "This works well" < h < g; // 将字符串 "This works well"、字符数组 h 和指针 g 写入输出流
    out.close(); // 关闭输出流

    izstream in("temp.gz"); // 创建输入流对象 in，用于读取文件 temp.gz
    char *x = read_string(in), *y = new char[256], z[256]; // 从输入流中读取字符串并分配内存给指针 x，分配内存给指针 y 和字符数组 z
    in > y > z; // 从输入流中读取数据到指针 y 和字符数组 z
    in.close(); // 关闭输入流
    cout << x << endl << y << endl << z << endl; // 输出指针 x、指针 y 和字符数组 z 的值

    out.open("temp.gz"); // 打开输出流，用于尝试 ASCII 输出；使用 zcat temp.gz 命令查看结果
    out << setw(50) << setfill('#') << setprecision(20) << x << endl << y << endl << z << endl; // 格式化输出指针 x、指针 y 和字符数组 z 的值
    out << z << endl << y << endl << x << endl; // 输出字符数组 z、指针 y 和指针 x 的值
    out << 1.1234567890123456789 << endl; // 输出浮点数
    delete[] x; delete[] y; // 释放指针 x 和指针 y 所指向的内存
}
```