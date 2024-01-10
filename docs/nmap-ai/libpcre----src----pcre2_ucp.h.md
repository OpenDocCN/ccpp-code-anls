# `nmap\libpcre\src\pcre2_ucp.h`

```
/*************************************************
*      Perl-Compatible Regular Expressions       *
*************************************************/

/* PCRE is a library of functions to support regular expressions whose syntax
and semantics are as close as possible to those of the Perl 5 language.

                       Written by Philip Hazel
     Original API code Copyright (c) 1997-2012 University of Cambridge
          New API code Copyright (c) 2016-2022 University of Cambridge

This module is auto-generated from Unicode data files. DO NOT EDIT MANUALLY!
Instead, modify the maint/GenerateUcpHeader.py script and run it to generate
a new version of this code.

-----------------------------------------------------------------------------
Redistribution and use in source and binary forms, with or without
modification, are permitted provided that the following conditions are met:

    * Redistributions of source code must retain the above copyright notice,
      this list of conditions and the following disclaimer.

    * Redistributions in binary form must reproduce the above copyright
      notice, this list of conditions and the following disclaimer in the
      documentation and/or other materials provided with the distribution.

    * Neither the name of the University of Cambridge nor the names of its
      contributors may be used to endorse or promote products derived from
      this software without specific prior written permission.

THIS SOFTWARE IS PROVIDED BY THE COPYRIGHT HOLDERS AND CONTRIBUTORS "AS IS"
AND ANY EXPRESS OR IMPLIED WARRANTIES, INCLUDING, BUT NOT LIMITED TO, THE
IMPLIED WARRANTIES OF MERCHANTABILITY AND FITNESS FOR A PARTICULAR PURPOSE
ARE DISCLAIMED. IN NO EVENT SHALL THE COPYRIGHT OWNER OR CONTRIBUTORS BE
LIABLE FOR ANY DIRECT, INDIRECT, INCIDENTAL, SPECIAL, EXEMPLARY, OR
CONSEQUENTIAL DAMAGES (INCLUDING, BUT NOT LIMITED TO, PROCUREMENT OF
SUBSTITUTE GOODS OR SERVICES; LOSS OF USE, DATA, OR PROFITS; OR BUSINESS
# 定义了用于返回UCD访问宏的Unicode属性值的文件
#ifndef PCRE2_UCP_H_IDEMPOTENT_GUARD
#define PCRE2_UCP_H_IDEMPOTENT_GUARD

# 包含了通用字符类别的枚举值
enum {
  ucp_C,  # 控制字符
  ucp_L,  # 字母
  ucp_M,  # 标记
  ucp_N,  # 数字
  ucp_P,  # 标点
  ucp_S,  # 空白字符
  ucp_Z,  # 分隔符
};

# 这些是特定的字符类别
# 枚举类型，表示 Unicode 字符的属性
enum {
  ucp_Cc,    /* Control */                  # 控制字符
  ucp_Cf,    /* Format */                   # 格式字符
  ucp_Cn,    /* Unassigned */               # 未分配字符
  ucp_Co,    /* Private use */              # 私有使用字符
  ucp_Cs,    /* Surrogate */                # 代理字符
  ucp_Ll,    /* Lower case letter */        # 小写字母
  ucp_Lm,    /* Modifier letter */          # 修饰符字母
  ucp_Lo,    /* Other letter */             # 其他字母
  ucp_Lt,    /* Title case letter */        # 标题大小写字母
  ucp_Lu,    /* Upper case letter */        # 大写字母
  ucp_Mc,    /* Spacing mark */             # 空格标记
  ucp_Me,    /* Enclosing mark */           # 封闭标记
  ucp_Mn,    /* Non-spacing mark */         # 非间距标记
  ucp_Nd,    /* Decimal number */           # 十进制数字
  ucp_Nl,    /* Letter number */            # 字母数字
  ucp_No,    /* Other number */             # 其他数字
  ucp_Pc,    /* Connector punctuation */    # 连接标点符号
  ucp_Pd,    /* Dash punctuation */         # 破折号标点符号
  ucp_Pe,    /* Close punctuation */        # 结束标点符号
  ucp_Pf,    /* Final punctuation */        # 最终标点符号
  ucp_Pi,    /* Initial punctuation */      # 初始标点符号
  ucp_Po,    /* Other punctuation */        # 其他标点符号
  ucp_Ps,    /* Open punctuation */         # 开始标点符号
  ucp_Sc,    /* Currency symbol */          # 货币符号
  ucp_Sk,    /* Modifier symbol */          # 修饰符符号
  ucp_Sm,    /* Mathematical symbol */      # 数学符号
  ucp_So,    /* Other symbol */             # 其他符号
  ucp_Zl,    /* Line separator */           # 行分隔符
  ucp_Zp,    /* Paragraph separator */      # 段落分隔符
  ucp_Zs,    /* Space separator */          # 空格分隔符
};

/* 这些是布尔属性。 */
# 定义枚举类型，表示 Unicode 属性的各种取值
enum {
  ucp_ASCII,  # ASCII 字符
  ucp_ASCII_Hex_Digit,  # ASCII 十六进制数字
  ucp_Alphabetic,  # 字母
  ucp_Bidi_Control,  # 双向控制字符
  ucp_Bidi_Mirrored,  # 双向镜像字符
  ucp_Case_Ignorable,  # 大小写忽略字符
  ucp_Cased,  # 有大小写形式的字符
  ucp_Changes_When_Casefolded,  # 大小写折叠后改变的字符
  ucp_Changes_When_Casemapped,  # 大小写映射后改变的字符
  ucp_Changes_When_Lowercased,  # 转换为小写后改变的字符
  ucp_Changes_When_Titlecased,  # 转换为标题大小写后改变的字符
  ucp_Changes_When_Uppercased,  # 转换为大写后改变的字符
  ucp_Dash,  # 破折号
  ucp_Default_Ignorable_Code_Point,  # 默认可忽略的代码点
  ucp_Deprecated,  # 已废弃的字符
  ucp_Diacritic,  # 声调符号
  ucp_Emoji,  # 表情符号
  ucp_Emoji_Component,  # 表情符号组件
  ucp_Emoji_Modifier,  # 表情符号修饰符
  ucp_Emoji_Modifier_Base,  # 表情符号修饰符基字符
  ucp_Emoji_Presentation,  # 表情符号展示字符
  ucp_Extended_Pictographic,  # 扩展象形文字
  ucp_Extender,  # 扩展字符
  ucp_Grapheme_Base,  # 表意字符基字符
  ucp_Grapheme_Extend,  # 表意字符扩展字符
  ucp_Grapheme_Link,  # 表意字符连接字符
  ucp_Hex_Digit,  # 十六进制数字
  ucp_IDS_Binary_Operator,  # 标识符二元操作符
  ucp_IDS_Trinary_Operator,  # 标识符三元操作符
  ucp_ID_Continue,  # 标识符继续字符
  ucp_ID_Start,  # 标识符起始字符
  ucp_Ideographic,  # 表意文字
  ucp_Join_Control,  # 连接控制字符
  ucp_Logical_Order_Exception,  # 逻辑顺序异常字符
  ucp_Lowercase,  # 小写字符
  ucp_Math,  # 数学符号
  ucp_Noncharacter_Code_Point,  # 非字符代码点
  ucp_Pattern_Syntax,  # 模式语法字符
  ucp_Pattern_White_Space,  # 模式空白字符
  ucp_Prepended_Concatenation_Mark,  # 前置连接标记
  ucp_Quotation_Mark,  # 引号标点
  ucp_Radical,  # 部首
  ucp_Regional_Indicator,  # 区域指示符
  ucp_Sentence_Terminal,  # 句子结束符
  ucp_Soft_Dotted,  # 软点字符
  ucp_Terminal_Punctuation,  # 终结标点
  ucp_Unified_Ideograph,  # 统一表意文字
  ucp_Uppercase,  # 大写字符
  ucp_Variation_Selector,  # 变体选择器
  ucp_White_Space,  # 空白字符
  ucp_XID_Continue,  # XID 继续字符
  ucp_XID_Start,  # XID 起始字符
  /* This must be last */
  ucp_Bprop_Count  # 属性取值的数量
};

# 定义 ucd_boolprop_sets[] 数组中每个条目的大小
#define ucd_boolprop_sets_item_size 2

# 这些是双向类的值
# 定义枚举类型，表示双向文本的属性
enum {
  ucp_bidiAL,   /* Arabic letter */  # 阿拉伯字母
  ucp_bidiAN,   /* Arabic number */  # 阿拉伯数字
  ucp_bidiB,    /* Paragraph separator */  # 段落分隔符
  ucp_bidiBN,   /* Boundary neutral */  # 边界中性
  ucp_bidiCS,   /* Common separator */  # 通用分隔符
  ucp_bidiEN,   /* European number */  # 欧洲数字
  ucp_bidiES,   /* European separator */  # 欧洲分隔符
  ucp_bidiET,   /* European terminator */  # 欧洲终止符
  ucp_bidiFSI,  /* First strong isolate */  # 第一个强隔离符
  ucp_bidiL,    /* Left to right */  # 从左到右
  ucp_bidiLRE,  /* Left to right embedding */  # 左到右嵌入
  ucp_bidiLRI,  /* Left to right isolate */  # 左到右隔离
  ucp_bidiLRO,  /* Left to right override */  # 左到右覆盖
  ucp_bidiNSM,  /* Non-spacing mark */  # 非间距标记
  ucp_bidiON,   /* Other neutral */  # 其他中性
  ucp_bidiPDF,  /* Pop directional format */  # 弹出方向格式
  ucp_bidiPDI,  /* Pop directional isolate */  # 弹出方向隔离
  ucp_bidiR,    /* Right to left */  # 从右到左
  ucp_bidiRLE,  /* Right to left embedding */  # 右到左嵌入
  ucp_bidiRLI,  /* Right to left isolate */  # 右到左隔离
  ucp_bidiRLO,  /* Right to left override */  # 右到左覆盖
  ucp_bidiS,    /* Segment separator */  # 段分隔符
  ucp_bidiWS,   /* White space */  # 空白字符
};

# 定义枚举类型，表示分隔符和标记的属性
# 这些属性用于断字断句算法
enum {
  ucp_gbCR,                    /*  0 */  # 回车
  ucp_gbLF,                    /*  1 */  # 换行
  ucp_gbControl,               /*  2 */  # 控制字符
  ucp_gbExtend,                /*  3 */  # 扩展字符
  ucp_gbPrepend,               /*  4 */  # 前置字符
  ucp_gbSpacingMark,           /*  5 */  # 空格标记
  ucp_gbL,                     /*  6 Hangul syllable type L */  # 韩文音节类型 L
  ucp_gbV,                     /*  7 Hangul syllable type V */  # 韩文音节类型 V
  ucp_gbT,                     /*  8 Hangul syllable type T */  # 韩文音节类型 T
  ucp_gbLV,                    /*  9 Hangul syllable type LV */  # 韩文音节类型 LV
  ucp_gbLVT,                   /* 10 Hangul syllable type LVT */  # 韩文音节类型 LVT
  ucp_gbRegional_Indicator,    /* 11 */  # 区域指示符
  ucp_gbOther,                 /* 12 */  # 其他
  ucp_gbZWJ,                   /* 13 */  # 零宽连接符
  ucp_gbExtended_Pictographic, /* 14 */  # 扩展象形文字
};

# 定义枚举类型，表示脚本标识
# 用于指示字符所属的脚本
# 例如拉丁文、希伯来文、阿拉伯文等
# 这里的代码段似乎不完整，缺少了具体的脚本标识

# 定义 ucd_script_sets[] 数组中的条目大小
#define ucd_script_sets_item_size 3

# 结束 pcre2_ucp.h 文件
#endif  /* PCRE2_UCP_H_IDEMPOTENT_GUARD */

# pcre2_ucp.h 文件结束
```