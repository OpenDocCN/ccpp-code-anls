# `nmap\libpcap\varattrs.h`

```cpp
/*
 * 声明代码的版权信息和许可条件
 */
/* -*- Mode: c; tab-width: 8; indent-tabs-mode: 1; c-basic-offset: 8; -*- */
/*
 * 版权声明，版权归加利福尼亚大学理事会所有
 * 保留所有权利
 */
/*
 * 在源代码和二进制形式中重新分发和使用，需满足以下条件：
 * 1. 源代码的再分发必须保留上述版权声明、条件列表和以下免责声明
 * 2. 二进制形式的再分发必须在文档和/或其他提供的材料中重现上述版权声明、条件列表和以下免责声明
 * 3. 所有提及此软件特性或使用的广告材料必须显示以下声明：
 *    本产品包括由劳伦斯伯克利实验室的计算机系统工程组开发的软件
 * 4. 未经特定事先书面许可，不得使用大学或实验室的名称来认可或推广从本软件衍生的产品
 */
/*
 * 本软件由理事会和贡献者“按原样”提供，不提供任何明示或暗示的担保，
 * 包括但不限于对适销性和特定用途的暗示担保
 * 理事会或贡献者不对任何直接、间接、附带、特殊、惩罚性或后果性的损害负责
 * （包括但不限于替代商品或服务的采购、使用损失、数据或利润损失、业务中断）
 * 无论是合同责任、严格责任还是侵权行为（包括疏忽或其他方式）引起的
 * 任何理论上的责任，即使已被告知可能发生此类损害
 */
/*
 * 定义 varattrs_h 宏，防止头文件重复包含
 */
#ifndef varattrs_h
#define varattrs_h
/*
 * Attributes to apply to variables, using various compiler-specific
 * extensions.
 */

#if __has_attribute(unused) \
    || PCAP_IS_AT_LEAST_GNUC_VERSION(2,0)
  /*
   * 如果编译器支持__attribute__((unused))，或者是GCC 2.0及更高版本，则定义_U_为__attribute__((unused))
   */
  #define _U_ __attribute__((unused))
#else
  /*
   * 如果编译器不支持__attribute__((unused))，则定义_U_为空
   */
  #define _U_
#endif

#endif
```