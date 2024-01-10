# `nmap\libdnet-stripped\include\err.h`

```
/*
 * err.h
 *
 * Adapted from OpenBSD libc *err* *warn* code.
 * 从 OpenBSD libc *err* *warn* 代码改编而来。

 * Copyright (c) 2000 Dug Song <dugsong@monkey.org>
 * 版权所有 (c) 2000 Dug Song <dugsong@monkey.org>

 * Copyright (c) 1993
 *    The Regents of the University of California.  All rights reserved.
 * 版权所有 (c) 1993 加利福尼亚大学理事会。保留所有权利。

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
 * 源代码和二进制形式的再分发和使用，无论是否经过修改，均允许，前提是满足以下条件：
 * 1. 源代码的再分发必须保留上述版权声明、本条件列表和以下免责声明。
 * 2. 二进制形式的再分发必须在文档和/或其他提供的材料中复制上述版权声明、本条件列表和以下免责声明。
 * 3. 所有提及此软件特性或使用的广告材料必须显示以下确认：
 *    本产品包括由加利福尼亚大学伯克利分校及其贡献者开发的软件。
 * 4. 未经特定事先书面许可，不得使用大学的名称或其贡献者的名称来认可或推广从本软件衍生的产品。

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
 * 本软件由理事会和贡献者“按原样”提供，不提供任何明示或暗示的担保，包括但不限于对适销性和特定用途的暗示担保。在任何情况下，理事会或贡献者均不对任何直接、间接、附带、特殊、惩罚性或后果性的损害（包括但不限于替代商品或服务的采购、使用、数据或利润的损失或业务中断）承担责任，无论是在合同、严格责任或侵权行为（包括疏忽或其他方式）的任何责任理论下，即使已被告知可能发生此类损害。

 *    @(#)err.h    8.1 (Berkeley) 6/2/93
 */
# 如果 _ERR_H_ 未定义，则定义 _ERR_H_
#ifndef _ERR_H_
#define _ERR_H_

# 定义一个函数 err，接受一个整数参数 eval 和一个格式化字符串参数 fmt，以及可变数量的参数
void    err(int eval, const char *fmt, ...);
# 定义一个函数 warn，接受一个格式化字符串参数 fmt，以及可变数量的参数
void    warn(const char *fmt, ...);
# 定义一个函数 errx，接受一个整数参数 eval 和一个格式化字符串参数 fmt，以及可变数量的参数
void    errx(int eval, const char *fmt, ...);
# 定义一个函数 warnx，接受一个格式化字符串参数 fmt，以及可变数量的参数
void    warnx(const char *fmt, ...);

# 结束条件，如果 _ERR_H_ 已定义，则结束条件
#endif /* !_ERR_H_ */
```