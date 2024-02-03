# `nmap\libdnet-stripped\src\err.c`

```cpp
/*
 * err.c
 *
 * Adapted from OpenBSD libc *err* *warn* code.
 *
 * Copyright (c) 2000 Dug Song <dugsong@monkey.org>
 *
 * Copyright (c) 1993
 *    The Regents of the University of California.  All rights reserved.
 *
 * Redistribution and use in source and binary forms, with or without
 * modification, are permitted provided that the following conditions
 * are met:
 * 1. Redistributions of source code must retain the above copyright
 *    notice, this list of conditions and the following disclaimer.
 * 2. Redistributions in binary form must reproduce the above copyright
 *    notice, this list of conditions and the following disclaimer in the
 *    documentation and/or other materials provided with the distribution.
 * 3. All advertising materials mentioning features or use of this software
 *    must display the following acknowledgement:
 *    This product includes software developed by the University of
 *    California, Berkeley and its contributors.
 * 4. Neither the name of the University nor the names of its contributors
 *    may be used to endorse or promote products derived from this software
 *    without specific prior written permission.
 *
 * THIS SOFTWARE IS PROVIDED BY THE REGENTS AND CONTRIBUTORS ``AS IS'' AND
 * ANY EXPRESS OR IMPLIED WARRANTIES, INCLUDING, BUT NOT LIMITED TO, THE
 * IMPLIED WARRANTIES OF MERCHANTABILITY AND FITNESS FOR A PARTICULAR PURPOSE
 * ARE DISCLAIMED.  IN NO EVENT SHALL THE REGENTS OR CONTRIBUTORS BE LIABLE
 * FOR ANY DIRECT, INDIRECT, INCIDENTAL, SPECIAL, EXEMPLARY, OR CONSEQUENTIAL
 * DAMAGES (INCLUDING, BUT NOT LIMITED TO, PROCUREMENT OF SUBSTITUTE GOODS
 * OR SERVICES; LOSS OF USE, DATA, OR PROFITS; OR BUSINESS INTERRUPTION)
 * HOWEVER CAUSED AND ON ANY THEORY OF LIABILITY, WHETHER IN CONTRACT, STRICT
 * LIABILITY, OR TORT (INCLUDING NEGLIGENCE OR OTHERWISE) ARISING IN ANY WAY
 * OUT OF THE USE OF THIS SOFTWARE, EVEN IF ADVISED OF THE POSSIBILITY OF
 * SUCH DAMAGE.
 */

#ifdef _WIN32
#include <windows.h>
#endif
#include <stdio.h>  // 包含标准输入输出库
#include <stdlib.h>  // 包含标准库
#include <stdarg.h>  // 包含可变参数列表
#include <string.h>  // 包含字符串处理函数库
#include <errno.h>   // 包含错误处理库

void
err(int eval, const char *fmt, ...)  // 定义错误处理函数err，接受格式化字符串和可变参数
{
    va_list ap;  // 声明可变参数列表
    
    va_start(ap, fmt);  // 初始化可变参数列表
    if (fmt != NULL) {  // 如果格式化字符串不为空
        (void)vfprintf(stderr, fmt, ap);  // 将格式化字符串和可变参数列表输出到标准错误流
        (void)fprintf(stderr, ": ");  // 输出冒号和空格到标准错误流
    }
    va_end(ap);  // 结束可变参数列表
#ifdef _WIN32
    (void)fprintf(stderr, "error %lu\n", GetLastError());  // 如果是Windows系统，输出错误码到标准错误流
#else
    (void)fprintf(stderr, "%s\n", strerror(errno));  // 如果不是Windows系统，输出错误信息到标准错误流
#endif
    exit(eval);  // 退出程序并返回eval作为退出状态
}

void
warn(const char *fmt, ...)  // 定义警告处理函数warn，接受格式化字符串和可变参数
{
    va_list ap;  // 声明可变参数列表
    
    va_start(ap, fmt);  // 初始化可变参数列表
    if (fmt != NULL) {  // 如果格式化字符串不为空
        (void)vfprintf(stderr, fmt, ap);  // 将格式化字符串和可变参数列表输出到标准错误流
        (void)fprintf(stderr, ": ");  // 输出冒号和空格到标准错误流
    }
    va_end(ap);  // 结束可变参数列表
#ifdef _WIN32
    (void)fprintf(stderr, "error %lu\n", GetLastError());  // 如果是Windows系统，输出错误码到标准错误流
#else
    (void)fprintf(stderr, "%s\n", strerror(errno));  // 如果不是Windows系统，输出错误信息到标准错误流
#endif
}

void
errx(int eval, const char *fmt, ...)  // 定义无错误信息的错误处理函数errx，接受格式化字符串和可变参数
{
    va_list ap;  // 声明可变参数列表
    
    va_start(ap, fmt);  // 初始化可变参数列表
    if (fmt != NULL)
        (void)vfprintf(stderr, fmt, ap);  // 如果格式化字符串不为空，将格式化字符串和可变参数列表输出到标准错误流
    (void)fprintf(stderr, "\n");  // 输出换行符到标准错误流
    va_end(ap);  // 结束可变参数列表
    exit(eval);  // 退出程序并返回eval作为退出状态
}

void
warnx(const char *fmt, ...)  // 定义无错误信息的警告处理函数warnx，接受格式化字符串和可变参数
{
    va_list ap;  // 声明可变参数列表
    
    va_start(ap, fmt);  // 初始化可变参数列表
    if (fmt != NULL)
        (void)vfprintf(stderr, fmt, ap);  // 如果格式化字符串不为空，将格式化字符串和可变参数列表输出到标准错误流
        (void)fprintf(stderr, "\n");  // 输出换行符到标准错误流
    va_end(ap);  // 结束可变参数列表
}
```