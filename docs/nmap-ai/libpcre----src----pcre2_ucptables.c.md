# `nmap\libpcre\src\pcre2_ucptables.c`

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
Instead, modify the maint/GenerateUcpTables.py script and run it to generate
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
#ifdef SUPPORT_UNICODE
/* 如果定义了 SUPPORT_UNICODE 宏，则编译以下代码块 */

/* 下面的 PRIV(utt)[] 表格将 Unicode 属性名称转换为类型和代码值。它通过二分查找进行搜索，因此必须按名称的排序顺序排列。
   最初，表格中的每个条目的第一个字段包含指向名称字符串的指针。然而，这会导致在动态加载共享库时出现大量的重定位。
   通过将所有名称放入单个大字符串中，并使用偏移量代替，可以显著减少这种情况。
   所有字母都转换为小写，并根据 Unicode 建议和 Perl 使用的“宽松匹配”规则，删除下划线。 */

/* 定义字符串常量 */
#define STRING_adlam0 STR_a STR_d STR_l STR_a STR_m "\0"
#define STRING_adlm0 STR_a STR_d STR_l STR_m "\0"
#define STRING_aghb0 STR_a STR_g STR_h STR_b "\0"
#define STRING_ahex0 STR_a STR_h STR_e STR_x "\0"
#define STRING_ahom0 STR_a STR_h STR_o STR_m "\0"
#define STRING_alpha0 STR_a STR_l STR_p STR_h STR_a "\0"
#define STRING_alphabetic0 STR_a STR_l STR_p STR_h STR_a STR_b STR_e STR_t STR_i STR_c "\0"
#define STRING_anatolianhieroglyphs0 STR_a STR_n STR_a STR_t STR_o STR_l STR_i STR_a STR_n STR_h STR_i STR_e STR_r STR_o STR_g STR_l STR_y STR_p STR_h STR_s "\0"
#define STRING_any0 STR_a STR_n STR_y "\0"
#define STRING_arab0 STR_a STR_r STR_a STR_b "\0"
#define STRING_arabic0 STR_a STR_r STR_a STR_b STR_i STR_c "\0"
#define STRING_armenian0 STR_a STR_r STR_m STR_e STR_n STR_i STR_a STR_n "\0"
#define STRING_armi0 STR_a STR_r STR_m STR_i "\0"
#define STRING_armn0 STR_a STR_r STR_m STR_n "\0"
#define STRING_ascii0 STR_a STR_s STR_c STR_i STR_i "\0"
#define STRING_asciihexdigit0 STR_a STR_s STR_c STR_i STR_i STR_h STR_e STR_x STR_d STR_i STR_g STR_i STR_t "\0"
#define STRING_avestan0 STR_a STR_v STR_e STR_s STR_t STR_a STR_n "\0"
#define STRING_avst0 STR_a STR_v STR_s STR_t "\0"
#define STRING_bali0 STR_b STR_a STR_l STR_i "\0"
#define STRING_balinese0 STR_b STR_a STR_l STR_i STR_n STR_e STR_s STR_e "\0"
#define STRING_bamu0 STR_b STR_a STR_m STR_u "\0"
#define STRING_bamum0 STR_b STR_a STR_m STR_u STR_m "\0"
#define STRING_bass0 STR_b STR_a STR_s STR_s "\0"
#define STRING_bassavah0 STR_b STR_a STR_s STR_s STR_a STR_v STR_a STR_h "\0"
#define STRING_batak0 STR_b STR_a STR_t STR_a STR_k "\0"
#define STRING_batk0 STR_b STR_a STR_t STR_k "\0"
#define STRING_beng0 STR_b STR_e STR_n STR_g "\0"
#define STRING_bengali0 STR_b STR_e STR_n STR_g STR_a STR_l STR_i "\0"
#define STRING_bhaiksuki0 STR_b STR_h STR_a STR_i STR_k STR_s STR_u STR_k STR_i "\0"
#define STRING_bhks0 STR_b STR_h STR_k STR_s "\0"
#define STRING_bidial0 STR_b STR_i STR_d STR_i STR_a STR_l "\0"
#define STRING_bidian0 STR_b STR_i STR_d STR_i STR_a STR_n "\0"
#define STRING_bidib0 STR_b STR_i STR_d STR_i STR_b "\0"
#define STRING_bidibn0 STR_b STR_i STR_d STR_i STR_b STR_n "\0"
#define STRING_bidic0 STR_b STR_i STR_d STR_i STR_c "\0"
#define STRING_bidicontrol0 STR_b STR_i STR_d STR_i STR_c STR_o STR_n STR_t STR_r STR_o STR_l "\0"
#define STRING_bidics0 STR_b STR_i STR_d STR_i STR_c STR_s "\0"
#define STRING_bidien0 STR_b STR_i STR_d STR_i STR_e STR_n "\0"
#define STRING_bidies0 STR_b STR_i STR_d STR_i STR_e STR_s "\0"
#define STRING_bidiet0 STR_b STR_i STR_d STR_i STR_e STR_t "\0"
#define STRING_bidifsi0 STR_b STR_i STR_d STR_i STR_f STR_s STR_i "\0"
#define STRING_bidil0 STR_b STR_i STR_d STR_i STR_l "\0"
#define STRING_bidilre0 STR_b STR_i STR_d STR_i STR_l STR_r STR_e "\0"
#define STRING_bidilri0 STR_b STR_i STR_d STR_i STR_l STR_r STR_i "\0"
#define STRING_bidilro0 STR_b STR_i STR_d STR_i STR_l STR_r STR_o "\0"
# 定义字符串常量 STRING_bidim0
#define STRING_bidim0 STR_b STR_i STR_d STR_i_m "\0"
# 定义字符串常量 STRING_bidimirrored0
#define STRING_bidimirrored0 STR_b STR_i_d STR_i_m STR_i_r_r_o_r_e_d "\0"
# 定义字符串常量 STRING_bidinsm0
#define STRING_bidinsm0 STR_b STR_i_d STR_i_n_s_m "\0"
# 定义字符串常量 STRING_bidion0
#define STRING_bidion0 STR_b STR_i_d STR_i_o_n "\0"
# 定义字符串常量 STRING_bidipdf0
#define STRING_bidipdf0 STR_b STR_i_d STR_i_p_d_f "\0"
# 定义字符串常量 STRING_bidipdi0
#define STRING_bidipdi0 STR_b STR_i_d STR_i_p_d_i "\0"
# 定义字符串常量 STRING_bidir0
#define STRING_bidir0 STR_b STR_i_d STR_i_r "\0"
# 定义字符串常量 STRING_bidirle0
#define STRING_bidirle0 STR_b STR_i_d STR_i_r_l_e "\0"
# 定义字符串常量 STRING_bidirli0
#define STRING_bidirli0 STR_b STR_i_d STR_i_r_l_i "\0"
# 定义字符串常量 STRING_bidirlo0
#define STRING_bidirlo0 STR_b STR_i_d STR_i_r_l_o "\0"
# 定义字符串常量 STRING_bidis0
#define STRING_bidis0 STR_b STR_i_d STR_i_s "\0"
# 定义字符串常量 STRING_bidiws0
#define STRING_bidiws0 STR_b STR_i_d STR_i_w_s "\0"
# 定义字符串常量 STRING_bopo0
#define STRING_bopo0 STR_b STR_o_p_o "\0"
# 定义字符串常量 STRING_bopomofo0
#define STRING_bopomofo0 STR_b STR_o_p_o_m_o_f_o "\0"
# 定义字符串常量 STRING_brah0
#define STRING_brah0 STR_b_r_a_h "\0"
# 定义字符串常量 STRING_brahmi0
#define STRING_brahmi0 STR_b_r_a_h_m_i "\0"
# 定义字符串常量 STRING_brai0
#define STRING_brai0 STR_b_r_a_i "\0"
# 定义字符串常量 STRING_braille0
#define STRING_braille0 STR_b_r_a_i_l_l_e "\0"
# 定义字符串常量 STRING_bugi0
#define STRING_bugi0 STR_b_u_g_i "\0"
# 定义字符串常量 STRING_buginese0
#define STRING_buginese0 STR_b_u_g_i_n_e_s_e "\0"
# 定义字符串常量 STRING_buhd0
#define STRING_buhd0 STR_b_u_h_d "\0"
# 定义字符串常量 STRING_buhid0
#define STRING_buhid0 STR_b_u_h_i_d "\0"
# 定义字符串常量 STRING_c0
#define STRING_c0 STR_c "\0"
# 定义字符串常量 STRING_cakm0
#define STRING_cakm0 STR_c_a_k_m "\0"
# 定义字符串常量 STRING_canadianaboriginal0
#define STRING_canadianaboriginal0 STR_c_a_n_a_d_i_a_n_a_b_o_r_i_g_i_n_a_l "\0"
# 定义字符串常量 STRING_cans0
#define STRING_cans0 STR_c_a_n_s "\0"
# 定义字符串常量 STRING_cari0
#define STRING_cari0 STR_c_a_r_i "\0"
# 定义字符串常量 STRING_carian0
#define STRING_carian0 STR_c_a_r_i_a_n "\0"
# 定义字符串常量 STRING_cased0
#define STRING_cased0 STR_c_a_s_e_d "\0"
# 定义字符串常量 STRING_caseignorable0
#define STRING_caseignorable0 STR_c_a_s_e_i_g_n_o_r_a_b_l_e "\0"
# 定义字符串常量，表示高加索阿尔巴尼亚语
#define STRING_caucasianalbanian0 STR_c STR_a STR_u STR_c STR_a STR_s STR_i STR_a STR_n STR_a STR_l STR_b STR_a STR_n STR_i STR_a STR_n "\0"
# 定义字符串常量，表示 CC
#define STRING_cc0 STR_c STR_c "\0"
# 定义字符串常量，表示 CF
#define STRING_cf0 STR_c STR_f "\0"
# 定义字符串常量，表示 Chakma 语
#define STRING_chakma0 STR_c STR_h STR_a STR_k STR_m STR_a "\0"
# 定义字符串常量，表示 Cham 语
#define STRING_cham0 STR_c STR_h STR_a STR_m "\0"
# 定义字符串常量，表示当进行大小写折叠时发生变化
#define STRING_changeswhencasefolded0 STR_c STR_h STR_a STR_n STR_g STR_e STR_s STR_w STR_h STR_e STR_n STR_c STR_a STR_s STR_e STR_f STR_o STR_l STR_d STR_e STR_d "\0"
# 定义字符串常量，表示当进行大小写映射时发生变化
#define STRING_changeswhencasemapped0 STR_c STR_h STR_a STR_n STR_g STR_e STR_s STR_w STR_h STR_e STR_n STR_c STR_a STR_s STR_e STR_m STR_a STR_p STR_p STR_e STR_d "\0"
# 定义字符串常量，表示当转换为小写时发生变化
#define STRING_changeswhenlowercased0 STR_c STR_h STR_a STR_n STR_g STR_e STR_s STR_w STR_h STR_e STR_n STR_l STR_o STR_w STR_e STR_r STR_c STR_a STR_s STR_e STR_d "\0"
# 定义字符串常量，表示当转换为首字母大写时发生变化
#define STRING_changeswhentitlecased0 STR_c STR_h STR_a STR_n STR_g STR_e STR_s STR_w STR_h STR_e STR_n STR_t STR_i STR_t STR_l STR_e STR_c STR_a STR_s STR_e STR_d "\0"
# 定义字符串常量，表示当转换为大写时发生变化
#define STRING_changeswhenuppercased0 STR_c STR_h STR_a STR_n STR_g STR_e STR_s STR_w STR_h STR_e STR_n STR_u STR_p STR_p STR_e STR_r STR_c STR_a STR_s STR_e STR_d "\0"
# 定义字符串常量，表示 Cher 语
#define STRING_cher0 STR_c STR_h STR_e STR_r "\0"
# 定义字符串常量，表示 Cherokee 语
#define STRING_cherokee0 STR_c STR_h STR_e STR_r STR_o STR_k STR_e STR_e "\0"
# 定义字符串常量，表示 Chorasmian 语
#define STRING_chorasmian0 STR_c STR_h STR_o STR_r STR_a STR_s STR_m STR_i STR_a STR_n "\0"
# 定义字符串常量，表示 Chrs
#define STRING_chrs0 STR_c STR_h STR_r STR_s "\0"
# 定义字符串常量，表示 Ci
#define STRING_ci0 STR_c STR_i "\0"
# 定义字符串常量，表示 Cn
#define STRING_cn0 STR_c STR_n "\0"
# 定义字符串常量，表示 Co
#define STRING_co0 STR_c STR_o "\0"
# 定义字符串常量，表示 Common
#define STRING_common0 STR_c STR_o STR_m STR_m STR_o STR_n "\0"
# 定义字符串常量，表示 Copt
#define STRING_copt0 STR_c STR_o STR_p STR_t "\0"
# 定义字符串常量，表示 Coptic
#define STRING_coptic0 STR_c STR_o STR_p STR_t STR_i STR_c "\0"
# 定义字符串常量，表示 Cpmn
#define STRING_cpmn0 STR_c STR_p STR_m STR_n "\0"
# 定义字符串常量，表示 Cprt
#define STRING_cprt0 STR_c STR_p STR_r STR_t "\0"
# 定义字符串常量，表示 Cs
#define STRING_cs0 STR_c STR_s "\0"
# 定义字符串常量，表示 Cuneiform
#define STRING_cuneiform0 STR_c STR_u STR_n STR_e STR_i STR_f STR_o STR_r STR_m "\0"
# 定义字符串常量，表示 Cwcf
#define STRING_cwcf0 STR_c STR_w STR_c STR_f "\0"
# 定义字符串常量 "cwcm0"
#define STRING_cwcm0 STR_c STR_w STR_c STR_m "\0"
# 定义字符串常量 "cwl0"
#define STRING_cwl0 STR_c STR_w STR_l "\0"
# 定义字符串常量 "cwt0"
#define STRING_cwt0 STR_c STR_w STR_t "\0"
# 定义字符串常量 "cwu0"
#define STRING_cwu0 STR_c STR_w STR_u "\0"
# 定义字符串常量 "cypriot0"
#define STRING_cypriot0 STR_c STR_y STR_p STR_r STR_i STR_o STR_t "\0"
# 定义字符串常量 "cyprominoan0"
#define STRING_cyprominoan0 STR_c STR_y STR_p STR_r STR_o STR_m STR_i STR_n STR_o STR_a STR_n "\0"
# 定义字符串常量 "cyrillic0"
#define STRING_cyrillic0 STR_c STR_y STR_r STR_i STR_l STR_l STR_i STR_c "\0"
# 定义字符串常量 "cyrl0"
#define STRING_cyrl0 STR_c STR_y STR_r STR_l "\0"
# 定义字符串常量 "dash0"
#define STRING_dash0 STR_d STR_a STR_s STR_h "\0"
# 定义字符串常量 "defaultignorablecodepoint0"
#define STRING_defaultignorablecodepoint0 STR_d STR_e STR_f STR_a STR_u STR_l STR_t STR_i STR_g STR_n STR_o STR_r STR_a STR_b STR_l STR_e STR_c STR_o STR_d STR_e STR_p STR_o STR_i STR_n STR_t "\0"
# 定义字符串常量 "dep0"
#define STRING_dep0 STR_d STR_e STR_p "\0"
# 定义字符串常量 "deprecated0"
#define STRING_deprecated0 STR_d STR_e STR_p STR_r STR_e STR_c STR_a STR_t STR_e STR_d "\0"
# 定义字符串常量 "deseret0"
#define STRING_deseret0 STR_d STR_e STR_s STR_e STR_r STR_e STR_t "\0"
# 定义字符串常量 "deva0"
#define STRING_deva0 STR_d STR_e STR_v STR_a "\0"
# 定义字符串常量 "devanagari0"
#define STRING_devanagari0 STR_d STR_e STR_v STR_a STR_n STR_a STR_g STR_a STR_r STR_i "\0"
# 定义字符串常量 "di0"
#define STRING_di0 STR_d STR_i "\0"
# 定义字符串常量 "dia0"
#define STRING_dia0 STR_d STR_i STR_a "\0"
# 定义字符串常量 "diacritic0"
#define STRING_diacritic0 STR_d STR_i STR_a STR_c STR_r STR_i STR_t STR_i STR_c "\0"
# 定义字符串常量 "diak0"
#define STRING_diak0 STR_d STR_i STR_a STR_k "\0"
# 定义字符串常量 "divesakuru0"
#define STRING_divesakuru0 STR_d STR_i STR_v STR_e STR_s STR_a STR_k STR_u STR_r STR_u "\0"
# 定义字符串常量 "dogr0"
#define STRING_dogr0 STR_d STR_o STR_g STR_r "\0"
# 定义字符串常量 "dogra0"
#define STRING_dogra0 STR_d STR_o STR_g STR_r STR_a "\0"
# 定义字符串常量 "dsrt0"
#define STRING_dsrt0 STR_d STR_s STR_r STR_t "\0"
# 定义字符串常量 "dupl0"
#define STRING_dupl0 STR_d STR_u STR_p STR_l "\0"
# 定义字符串常量 "duployan0"
#define STRING_duployan0 STR_d STR_u STR_p STR_l STR_o STR_y STR_a STR_n "\0"
# 定义字符串常量 "ebase0"
#define STRING_ebase0 STR_e STR_b STR_a STR_s STR_e "\0"
# 定义字符串常量 "ecomp0"
#define STRING_ecomp0 STR_e STR_c STR_o STR_m STR_p "\0"
# 定义字符串常量 "egyp0"
#define STRING_egyp0 STR_e STR_g STR_y STR_p "\0"
# 定义字符串常量 "egyptianhieroglyphs0"
#define STRING_egyptianhieroglyphs0 STR_e STR_g STR_y STR_p STR_t STR_i STR_a STR_n STR_h STR_i STR_e STR_r STR_o STR_g STR_l STR_y STR_p STR_h STR_s "\0"
# 定义字符串常量 "elba"，以空字符结尾
#define STRING_elba0 STR_e STR_l STR_b STR_a "\0"
# 定义字符串常量 "elbasan"，以空字符结尾
#define STRING_elbasan0 STR_e STR_l STR_b STR_a STR_s STR_a STR_n "\0"
# 定义字符串常量 "elym"，以空字符结尾
#define STRING_elym0 STR_e STR_l STR_y STR_m "\0"
# 定义字符串常量 "elymaic"，以空字符结尾
#define STRING_elymaic0 STR_e STR_l STR_y STR_m STR_a STR_i STR_c "\0"
# 定义字符串常量 "emod"，以空字符结尾
#define STRING_emod0 STR_e STR_m STR_o STR_d "\0"
# 定义字符串常量 "emoji"，以空字符结尾
#define STRING_emoji0 STR_e STR_m STR_o STR_j STR_i "\0"
# 定义字符串常量 "emojicomponent"，以空字符结尾
#define STRING_emojicomponent0 STR_e STR_m STR_o STR_j STR_i STR_c STR_o STR_m STR_p STR_o STR_n STR_e STR_n STR_t "\0"
# 定义字符串常量 "emojimodifier"，以空字符结尾
#define STRING_emojimodifier0 STR_e STR_m STR_o STR_j STR_i STR_m STR_o STR_d STR_i STR_f STR_i STR_e STR_r "\0"
# 定义字符串常量 "emojimodifierbase"，以空字符结尾
#define STRING_emojimodifierbase0 STR_e STR_m STR_o STR_j STR_i STR_m STR_o STR_d STR_i STR_f STR_i STR_e STR_r STR_b STR_a STR_s STR_e "\0"
# 定义字符串常量 "emojipresentation"，以空字符结尾
#define STRING_emojipresentation0 STR_e STR_m STR_o STR_j STR_i STR_p STR_r STR_e STR_s STR_e STR_n STR_t STR_a STR_t STR_i STR_o STR_n "\0"
# 定义字符串常量 "epres"，以空字符结尾
#define STRING_epres0 STR_e STR_p STR_r STR_e STR_s "\0"
# 定义字符串常量 "ethi"，以空字符结尾
#define STRING_ethi0 STR_e STR_t STR_h STR_i "\0"
# 定义字符串常量 "ethiopic"，以空字符结尾
#define STRING_ethiopic0 STR_e STR_t STR_h STR_i STR_o STR_p STR_i STR_c "\0"
# 定义字符串常量 "ext"，以空字符结尾
#define STRING_ext0 STR_e STR_x STR_t "\0"
# 定义字符串常量 "extendedpictographic"，以空字符结尾
#define STRING_extendedpictographic0 STR_e STR_x STR_t STR_e STR_n STR_d STR_e STR_d STR_p STR_i STR_c STR_t STR_o STR_g STR_r STR_a STR_p STR_h STR_i STR_c "\0"
# 定义字符串常量 "extender"，以空字符结尾
#define STRING_extender0 STR_e STR_x STR_t STR_e STR_n STR_d STR_e STR_r "\0"
# 定义字符串常量 "extpict"，以空字符结尾
#define STRING_extpict0 STR_e STR_x STR_t STR_p STR_i STR_c STR_t "\0"
# 定义字符串常量 "geor"，以空字符结尾
#define STRING_geor0 STR_g STR_e STR_o STR_r "\0"
# 定义字符串常量 "georgian"，以空字符结尾
#define STRING_georgian0 STR_g STR_e STR_o STR_r STR_g STR_i STR_a STR_n "\0"
# 定义字符串常量 "glag"，以空字符结尾
#define STRING_glag0 STR_g STR_l STR_a STR_g "\0"
# 定义字符串常量 "glagolitic"，以空字符结尾
#define STRING_glagolitic0 STR_g STR_l STR_a STR_g STR_o STR_l STR_i STR_t STR_i STR_c "\0"
# 定义字符串常量 "gong"，以空字符结尾
#define STRING_gong0 STR_g STR_o STR_n STR_g "\0"
# 定义字符串常量 "gonm"，以空字符结尾
#define STRING_gonm0 STR_g STR_o STR_n STR_m "\0"
# 定义字符串常量 "goth"，以空字符结尾
#define STRING_goth0 STR_g STR_o STR_t STR_h "\0"
# 定义字符串常量 "gothic"，以空字符结尾
#define STRING_gothic0 STR_g STR_o STR_t STR_h STR_i STR_c "\0"
# 定义字符串常量 "gran"，以空字符结尾
#define STRING_gran0 STR_g STR_r STR_a STR_n "\0"
#define STRING_grantha0 STR_g STR_r STR_a STR_n STR_t STR_h STR_a "\0"
# 定义名为 STRING_grantha0 的宏，其值为 "grantha\0"

#define STRING_graphemebase0 STR_g STR_r STR_a STR_p STR_h STR_e STR_m STR_e STR_b STR_a STR_s STR_e "\0"
# 定义名为 STRING_graphemebase0 的宏，其值为 "graphemebase\0"

#define STRING_graphemeextend0 STR_g STR_r STR_a STR_p STR_h STR_e STR_m STR_e STR_e STR_x STR_t STR_e STR_n STR_d "\0"
# 定义名为 STRING_graphemeextend0 的宏，其值为 "graphemeextend\0"

#define STRING_graphemelink0 STR_g STR_r STR_a STR_p STR_h STR_e STR_m STR_e STR_l STR_i STR_n STR_k "\0"
# 定义名为 STRING_graphemelink0 的宏，其值为 "graphemelink\0"

#define STRING_grbase0 STR_g STR_r STR_b STR_a STR_s STR_e "\0"
# 定义名为 STRING_grbase0 的宏，其值为 "grbase\0"

#define STRING_greek0 STR_g STR_r STR_e STR_e STR_k "\0"
# 定义名为 STRING_greek0 的宏，其值为 "greek\0"

#define STRING_grek0 STR_g STR_r STR_e STR_k "\0"
# 定义名为 STRING_grek0 的宏，其值为 "grek\0"

#define STRING_grext0 STR_g STR_r STR_e STR_x STR_t "\0"
# 定义名为 STRING_grext0 的宏，其值为 "grext\0"

#define STRING_grlink0 STR_g STR_r STR_l STR_i STR_n STR_k "\0"
# 定义名为 STRING_grlink0 的宏，其值为 "grlink\0"

#define STRING_gujarati0 STR_g STR_u STR_j STR_a STR_r STR_a STR_t STR_i "\0"
# 定义名为 STRING_gujarati0 的宏，其值为 "gujarati\0"

#define STRING_gujr0 STR_g STR_u STR_j STR_r "\0"
# 定义名为 STRING_gujr0 的宏，其值为 "gujr\0"

#define STRING_gunjalagondi0 STR_g STR_u STR_n STR_j STR_a STR_l STR_a STR_g STR_o STR_n STR_d STR_i "\0"
# 定义名为 STRING_gunjalagondi0 的宏，其值为 "gunjalagondi\0"

#define STRING_gurmukhi0 STR_g STR_u STR_r STR_m STR_u STR_k STR_h STR_i "\0"
# 定义名为 STRING_gurmukhi0 的宏，其值为 "gurmukhi\0"

#define STRING_guru0 STR_g STR_u STR_r STR_u "\0"
# 定义名为 STRING_guru0 的宏，其值为 "guru\0"

#define STRING_han0 STR_h STR_a STR_n "\0"
# 定义名为 STRING_han0 的宏，其值为 "han\0"

#define STRING_hang0 STR_h STR_a STR_n STR_g "\0"
# 定义名为 STRING_hang0 的宏，其值为 "hang\0"

#define STRING_hangul0 STR_h STR_a STR_n STR_g STR_u STR_l "\0"
# 定义名为 STRING_hangul0 的宏，其值为 "hangul\0"

#define STRING_hani0 STR_h STR_a STR_n STR_i "\0"
# 定义名为 STRING_hani0 的宏，其值为 "hani\0"

#define STRING_hanifirohingya0 STR_h STR_a STR_n STR_i STR_f STR_i STR_r STR_o STR_h STR_i STR_n STR_g STR_y STR_a "\0"
# 定义名为 STRING_hanifirohingya0 的宏，其值为 "hanifirohingya\0"

#define STRING_hano0 STR_h STR_a STR_n STR_o "\0"
# 定义名为 STRING_hano0 的宏，其值为 "hano\0"

#define STRING_hanunoo0 STR_h STR_a STR_n STR_u STR_n STR_o STR_o "\0"
# 定义名为 STRING_hanunoo0 的宏，其值为 "hanunoo\0"

#define STRING_hatr0 STR_h STR_a STR_t STR_r "\0"
# 定义名为 STRING_hatr0 的宏，其值为 "hatr\0"

#define STRING_hatran0 STR_h STR_a STR_t STR_r STR_a STR_n "\0"
# 定义名为 STRING_hatran0 的宏，其值为 "hatran\0"

#define STRING_hebr0 STR_h STR_e STR_b STR_r "\0"
# 定义名为 STRING_hebr0 的宏，其值为 "hebr\0"

#define STRING_hebrew0 STR_h STR_e STR_b STR_r STR_e STR_w "\0"
# 定义名为 STRING_hebrew0 的宏，其值为 "hebrew\0"

#define STRING_hex0 STR_h STR_e STR_x "\0"
# 定义名为 STRING_hex0 的宏，其值为 "hex\0"

#define STRING_hexdigit0 STR_h STR_e STR_x STR_d STR_i G STR_i T "\0"
# 定义名为 STRING_hexdigit0 的宏，其值为 "hexdigit\0"

#define STRING_hira0 STR_h STR_i STR_r STR_a "\0"
# 定义名为 STRING_hira0 的宏，其值为 "hira\0"

#define STRING_hiragana0 STR_h STR_i STR_r STR_a STR_g STR_a STR_n STR_a "\0"
# 定义名为 STRING_hiragana0 的宏，其值为 "hiragana\0"
# 定义字符串常量 "hluw0"，由 "h", "l", "u", "w" 组成，以空字符结尾
#define STRING_hluw0 STR_h STR_l STR_u STR_w "\0"
# 定义字符串常量 "hmng0"，由 "h", "m", "n", "g" 组成，以空字符结尾
#define STRING_hmng0 STR_h STR_m STR_n STR_g "\0"
# 定义字符串常量 "hmnp0"，由 "h", "m", "n", "p" 组成，以空字符结尾
#define STRING_hmnp0 STR_h STR_m STR_n STR_p "\0"
# 定义字符串常量 "hung0"，由 "h", "u", "n", "g" 组成，以空字符结尾
#define STRING_hung0 STR_h STR_u STR_n STR_g "\0"
# 定义字符串常量 "idc0"，由 "i", "d", "c" 组成，以空字符结尾
#define STRING_idc0 STR_i STR_d STR_c "\0"
# 定义字符串常量 "idcontinue0"，由 "i", "d", "c", "o", "n", "t", "i", "n", "u", "e" 组成，以空字符结尾
#define STRING_idcontinue0 STR_i STR_d STR_c STR_o STR_n STR_t STR_i STR_n STR_u STR_e "\0"
# 定义字符串常量 "ideo0"，由 "i", "d", "e", "o" 组成，以空字符结尾
#define STRING_ideo0 STR_i STR_d STR_e STR_o "\0"
# 定义字符串常量 "ideographic0"，由 "i", "d", "e", "o", "g", "r", "a", "p", "h", "i", "c" 组成，以空字符结尾
#define STRING_ideographic0 STR_i STR_d STR_e STR_o STR_g STR_r STR_a STR_p STR_h STR_i STR_c "\0"
# 定义字符串常量 "ids0"，由 "i", "d", "s" 组成，以空字符结尾
#define STRING_ids0 STR_i STR_d STR_s "\0"
# 定义字符串常量 "idsb0"，由 "i", "d", "s", "b" 组成，以空字符结尾
#define STRING_idsb0 STR_i STR_d STR_s STR_b "\0"
# 定义字符串常量 "idsbinaryoperator0"，由 "i", "d", "s", "b", "i", "n", "a", "r", "y", "o", "p", "e", "r", "a", "t", "o", "r" 组成，以空字符结尾
#define STRING_idsbinaryoperator0 STR_i STR_d STR_s STR_b STR_i STR_n STR_a STR_r STR_y STR_o STR_p STR_e STR_r STR_a STR_t STR_o STR_r "\0"
# 定义字符串常量 "idst0"，由 "i", "d", "s", "t" 组成，以空字符结尾
#define STRING_idst0 STR_i STR_d STR_s STR_t "\0"
# 定义字符串常量 "idstart0"，由 "i", "d", "s", "t", "a", "r", "t" 组成，以空字符结尾
#define STRING_idstart0 STR_i STR_d STR_s STR_t STR_a STR_r STR_t "\0"
# 定义字符串常量 "idstrinaryoperator0"，由 "i", "d", "s", "t", "r", "i", "n", "a", "r", "y", "o", "p", "e", "r", "a", "t", "o", "r" 组成，以空字符结尾
#define STRING_idstrinaryoperator0 STR_i STR_d STR_s STR_t STR_r STR_i STR_n STR_a STR_r STR_y STR_o STR_p STR_e STR_r STR_a STR_t STR_o STR_r "\0"
# 定义字符串常量 "imperialaramaic0"，由 "i", "m", "p", "e", "r", "i", "a", "l", "a", "r", "a", "m", "a", "i", "c" 组成，以空字符结尾
#define STRING_imperialaramaic0 STR_i STR_m STR_p STR_e STR_r STR_i STR_a STR_l STR_a STR_r STR_a STR_m STR_a STR_i STR_c "\0"
# 定义字符串常量 "inherited0"，由 "i", "n", "h", "e", "r", "i", "t", "e", "d" 组成，以空字符结尾
#define STRING_inherited0 STR_i STR_n STR_h STR_e STR_r STR_i STR_t STR_e STR_d "\0"
# 定义字符串常量 "inscriptionalpahlavi0"，由 "i", "n", "s", "c", "r", "i", "p", "t", "i", "o", "n", "a", "l", "p", "a", "h", "l", "a", "v", "i" 组成，以空字符结尾
#define STRING_inscriptionalpahlavi0 STR_i STR_n STR_s STR_c STR_r STR_i STR_p STR_t STR_i STR_o STR_n STR_a STR_l STR_p STR_a STR_h STR_l STR_a STR_v STR_i "\0"
# 定义字符串常量 "inscriptionalparthian0"，由 "i", "n", "s", "c", "r", "i", "p", "t", "i", "o", "n", "a", "l", "p", "a", "r", "t", "h", "i", "a", "n" 组成，以空字符结尾
#define STRING_inscriptionalparthian0 STR_i STR_n STR_s STR_c STR_r STR_i STR_p STR_t STR_i STR_o STR_n STR_a STR_l STR_p STR_a STR_r STR_t STR_h STR_i STR_a STR_n "\0"
# 定义字符串常量 "ital0"，由 "i", "t", "a", "l" 组成，以空字符结尾
#define STRING_ital0 STR_i STR_t STR_a STR_l "\0"
# 定义字符串常量 "java0"，由 "j", "a", "v", "a" 组成，以空字符结尾
#define STRING_java0 STR_j STR_a STR_v STR_a "\0"
# 定义字符串常量 "javanese0"，由 "j", "a", "v", "a", "n", "e", "s", "e" 组成，以空字符结尾
#define STRING_javanese0 STR_j STR_a STR_v STR_a STR_n STR_e STR_s STR_e "\0"
# 定义字符串常量 "joinc0"，由 "j", "o", "i", "n", "c" 组成，以空字符结尾
#define STRING_joinc0 STR_j STR_o STR_i STR_n STR_c "\0"
# 定义字符串常量 "joincontrol0"，由 "j", "o", "i", "n", "c", "o", "n", "t", "r", "o", "l" 组成，以空字符结尾
#define STRING_joincontrol0 STR_j STR_o STR_i STR_n STR_c STR_o STR_n STR_t STR_r STR_o STR_l "\0"
# 定义字符串常量 "kaithi0"，由 "k", "a", "i", "t", "h", "i" 组成，以空字符结尾
#define STRING_kaithi0 STR_k STR_a STR_i STR_t STR_h STR_i "\0"
# 定义字符串常量 "kali0"，由 "k", "a", "l", "i" 组成，以空字符结尾
#define STRING_kali0 STR_k STR_a STR_l STR_i "\0"
# 定义字符串常量 STRING_kana0，值为 "kana"，以空字符结尾
#define STRING_kana0 STR_k STR_a STR_n STR_a "\0"
# 定义字符串常量 STRING_kannada0，值为 "kannada"，以空字符结尾
#define STRING_kannada0 STR_k STR_a STR_n STR_n STR_a STR_d STR_a "\0"
# 定义字符串常量 STRING_katakana0，值为 "katakana"，以空字符结尾
#define STRING_katakana0 STR_k STR_a STR_t STR_a STR_k STR_a STR_n STR_a "\0"
# 定义字符串常量 STRING_kayahli0，值为 "kayahli"，以空字符结尾
#define STRING_kayahli0 STR_k STR_a STR_y STR_a STR_h STR_l STR_i "\0"
# 定义字符串常量 STRING_khar0，值为 "khar"，以空字符结尾
#define STRING_khar0 STR_k STR_h STR_a STR_r "\0"
# 定义字符串常量 STRING_kharoshthi0，值为 "kharoshthi"，以空字符结尾
#define STRING_kharoshthi0 STR_k STR_h STR_a STR_r STR_o STR_s STR_h STR_t STR_h STR_i "\0"
# 定义字符串常量 STRING_khitansmallscript0，值为 "khitansmallscript"，以空字符结尾
#define STRING_khitansmallscript0 STR_k STR_h STR_i STR_t STR_a STR_n STR_s STR_m STR_a STR_l STR_l STR_s STR_c STR_r STR_i STR_p STR_t "\0"
# 定义字符串常量 STRING_khmer0，值为 "khmer"，以空字符结尾
#define STRING_khmer0 STR_k STR_h STR_m STR_e STR_r "\0"
# 定义字符串常量 STRING_khmr0，值为 "khmr"，以空字符结尾
#define STRING_khmr0 STR_k STR_h STR_m STR_r "\0"
# 定义字符串常量 STRING_khoj0，值为 "khoj"，以空字符结尾
#define STRING_khoj0 STR_k STR_h STR_o STR_j "\0"
# 定义字符串常量 STRING_khojki0，值为 "khojki"，以空字符结尾
#define STRING_khojki0 STR_k STR_h STR_o STR_j STR_k STR_i "\0"
# 定义字符串常量 STRING_khudawadi0，值为 "khudawadi"，以空字符结尾
#define STRING_khudawadi0 STR_k STR_h STR_u STR_d STR_a STR_w STR_a STR_d STR_i "\0"
# 定义字符串常量 STRING_kits0，值为 "kits"，以空字符结尾
#define STRING_kits0 STR_k STR_i STR_t STR_s "\0"
# 定义字符串常量 STRING_knda0，值为 "knda"，以空字符结尾
#define STRING_knda0 STR_k STR_n STR_d STR_a "\0"
# 定义字符串常量 STRING_kthi0，值为 "kthi"，以空字符结尾
#define STRING_kthi0 STR_k STR_t STR_h STR_i "\0"
# 定义字符串常量 STRING_l0，值为 "l"，以空字符结尾
#define STRING_l0 STR_l "\0"
# 定义字符串常量 STRING_l_AMPERSAND0，值为 "l_AMPERSAND"，以空字符结尾
#define STRING_l_AMPERSAND0 STR_l STR_AMPERSAND "\0"
# 定义字符串常量 STRING_lana0，值为 "lana"，以空字符结尾
#define STRING_lana0 STR_l STR_a STR_n STR_a "\0"
# 定义字符串常量 STRING_lao0，值为 "lao"，以空字符结尾
#define STRING_lao0 STR_l STR_a STR_o "\0"
# 定义字符串常量 STRING_laoo0，值为 "laoo"，以空字符结尾
#define STRING_laoo0 STR_l STR_a STR_o STR_o "\0"
# 定义字符串常量 STRING_latin0，值为 "latin"，以空字符结尾
#define STRING_latin0 STR_l STR_a STR_t STR_i STR_n "\0"
# 定义字符串常量 STRING_latn0，值为 "latn"，以空字符结尾
#define STRING_latn0 STR_l STR_a STR_t STR_n "\0"
# 定义字符串常量 STRING_lc0，值为 "lc"，以空字符结尾
#define STRING_lc0 STR_l STR_c "\0"
# 定义字符串常量 STRING_lepc0，值为 "lepc"，以空字符结尾
#define STRING_lepc0 STR_l STR_e STR_p STR_c "\0"
# 定义字符串常量 STRING_lepcha0，值为 "lepcha"，以空字符结尾
#define STRING_lepcha0 STR_l STR_e STR_p STR_c STR_h STR_a "\0"
# 定义字符串常量 STRING_limb0，值为 "limb"，以空字符结尾
#define STRING_limb0 STR_l STR_i STR_m STR_b "\0"
# 定义字符串常量 STRING_limbu0，值为 "limbu"，以空字符结尾
#define STRING_limbu0 STR_l STR_i STR_m STR_b STR_u "\0"
# 定义字符串常量 STRING_lina0，值为 "lina"，以空字符结尾
#define STRING_lina0 STR_l STR_i STR_n STR_a "\0"
# 定义字符串常量 STRING_linb0，值为 "linb"，以空字符结尾
#define STRING_linb0 STR_l STR_i STR_n STR_b "\0"
# 定义字符串常量 STRING_lineara0，值为 "lineara"，以空字符结尾
#define STRING_lineara0 STR_l STR_i STR_n STR_e STR_a STR_r STR_a "\0"
# 定义字符串常量 STRING_linearb0，值为 "linearb"，以空字符结尾
#define STRING_linearb0 STR_l STR_i STR_n STR_e STR_a STR_r STR_b "\0"
# 定义字符串常量 STRING_lisu0，值为 "lisu"，以空字符结尾
#define STRING_lisu0 STR_l STR_i STR_s STR_u "\0"
# 定义字符串常量 STRING_ll0，值为 "ll"，以空字符结尾
#define STRING_ll0 STR_l STR_l "\0"
# 定义字符串常量 STRING_lm0，值为 "lm"，以空字符结尾
#define STRING_lm0 STR_l STR_m "\0"
# 定义字符串常量 STRING_lo0，值为 "lo"，以空字符结尾
#define STRING_lo0 STR_l STR_o "\0"
# 定义字符串 "loe0" 为 "loe" 后加上空字符
#define STRING_loe0 STR_l STR_o STR_e "\0"
# 定义字符串 "logicalorderexception0" 为 "logicalorderexception" 后加上空字符
#define STRING_logicalorderexception0 STR_l STR_o STR_g STR_i STR_c STR_a STR_l STR_o STR_r STR_d STR_e STR_r STR_e STR_x STR_c STR_e STR_p STR_t STR_i STR_o STR_n "\0"
# 定义字符串 "lower0" 为 "lower" 后加上空字符
#define STRING_lower0 STR_l STR_o STR_w STR_e STR_r "\0"
# 定义字符串 "lowercase0" 为 "lowercase" 后加上空字符
#define STRING_lowercase0 STR_l STR_o STR_w STR_e STR_r STR_c STR_a STR_s STR_e "\0"
# 定义字符串 "lt0" 为 "lt" 后加上空字符
#define STRING_lt0 STR_l STR_t "\0"
# 定义字符串 "lu0" 为 "lu" 后加上空字符
#define STRING_lu0 STR_l STR_u "\0"
# 定义字符串 "lyci0" 为 "lyci" 后加上空字符
#define STRING_lyci0 STR_l STR_y STR_c STR_i "\0"
# 定义字符串 "lycian0" 为 "lycian" 后加上空字符
#define STRING_lycian0 STR_l STR_y STR_c STR_i STR_a STR_n "\0"
# 定义字符串 "lydi0" 为 "lydi" 后加上空字符
#define STRING_lydi0 STR_l STR_y STR_d STR_i "\0"
# 定义字符串 "lydian0" 为 "lydian" 后加上空字符
#define STRING_lydian0 STR_l STR_y STR_d STR_i STR_a STR_n "\0"
# 定义字符串 "m0" 为 "m" 后加上空字符
#define STRING_m0 STR_m "\0"
# 定义字符串 "mahajani0" 为 "mahajani" 后加上空字符
#define STRING_mahajani0 STR_m STR_a STR_h STR_a STR_j STR_a STR_n STR_i "\0"
# 定义字符串 "mahj0" 为 "mahj" 后加上空字符
#define STRING_mahj0 STR_m STR_a STR_h STR_j "\0"
# 定义字符串 "maka0" 为 "maka" 后加上空字符
#define STRING_maka0 STR_m STR_a STR_k STR_a "\0"
# 定义字符串 "makasar0" 为 "makasar" 后加上空字符
#define STRING_makasar0 STR_m STR_a STR_k STR_a STR_s STR_a STR_r "\0"
# 定义字符串 "malayalam0" 为 "malayalam" 后加上空字符
#define STRING_malayalam0 STR_m STR_a STR_l STR_a STR_y STR_a STR_l STR_a STR_m "\0"
# 定义字符串 "mand0" 为 "mand" 后加上空字符
#define STRING_mand0 STR_m STR_a STR_n STR_d "\0"
# 定义字符串 "mandaic0" 为 "mandaic" 后加上空字符
#define STRING_mandaic0 STR_m STR_a STR_n STR_d STR_a STR_i STR_c "\0"
# 定义字符串 "mani0" 为 "mani" 后加上空字符
#define STRING_mani0 STR_m STR_a STR_n STR_i "\0"
# 定义字符串 "manichaean0" 为 "manichaean" 后加上空字符
#define STRING_manichaean0 STR_m STR_a STR_n STR_i STR_c STR_h STR_a STR_e STR_a STR_n "\0"
# 定义字符串 "marc0" 为 "marc" 后加上空字符
#define STRING_marc0 STR_m STR_a STR_r STR_c "\0"
# 定义字符串 "marchen0" 为 "marchen" 后加上空字符
#define STRING_marchen0 STR_m STR_a STR_r STR_c STR_h STR_e STR_n "\0"
# 定义字符串 "masaramgondi0" 为 "masaramgondi" 后加上空字符
#define STRING_masaramgondi0 STR_m STR_a STR_s STR_a STR_r STR_a STR_m STR_g STR_o STR_n STR_d STR_i "\0"
# 定义字符串 "math0" 为 "math" 后加上空字符
#define STRING_math0 STR_m STR_a STR_t STR_h "\0"
# 定义字符串 "mc0" 为 "mc" 后加上空字符
#define STRING_mc0 STR_m STR_c "\0"
# 定义字符串 "me0" 为 "me" 后加上空字符
#define STRING_me0 STR_m STR_e "\0"
# 定义字符串 "medefaidrin0" 为 "medefaidrin" 后加上空字符
#define STRING_medefaidrin0 STR_m STR_e STR_d STR_e STR_f STR_a STR_i STR_d STR_r STR_i STR_n "\0"
# 定义字符串 "medf0" 为 "medf" 后加上空字符
#define STRING_medf0 STR_m STR_e STR_d STR_f "\0"
# 定义字符串 "meeteimayek0" 为 "meeteimayek" 后加上空字符
#define STRING_meeteimayek0 STR_m STR_e STR_e STR_t STR_e STR_i STR_m STR_a STR_y STR_e STR_k "\0"
# 定义字符串 "mend0" 为 "mend" 后加上空字符
#define STRING_mend0 STR_m STR_e STR_n STR_d "\0"
# 定义字符串常量 STRING_mendekikakui0
#define STRING_mendekikakui0 STR_m STR_e STR_n STR_d STR_e STR_k STR_i STR_k STR_a STR_k STR_u STR_i "\0"
# 定义字符串常量 STRING_merc0
#define STRING_merc0 STR_m STR_e STR_r STR_c "\0"
# 定义字符串常量 STRING_mero0
#define STRING_mero0 STR_m STR_e STR_r STR_o "\0"
# 定义字符串常量 STRING_meroiticcursive0
#define STRING_meroiticcursive0 STR_m STR_e STR_r STR_o STR_i STR_t STR_i STR_c STR_c STR_u STR_r STR_s STR_i STR_v STR_e "\0"
# 定义字符串常量 STRING_meroitichieroglyphs0
#define STRING_meroitichieroglyphs0 STR_m STR_e STR_r STR_o STR_i STR_t STR_i STR_c STR_h STR_i STR_e STR_r STR_o STR_g STR_l STR_y STR_p STR_h STR_s "\0"
# 定义字符串常量 STRING_miao0
#define STRING_miao0 STR_m STR_i STR_a STR_o "\0"
# 定义字符串常量 STRING_mlym0
#define STRING_mlym0 STR_m STR_l STR_y STR_m "\0"
# 定义字符串常量 STRING_mn0
#define STRING_mn0 STR_m STR_n "\0"
# 定义字符串常量 STRING_modi0
#define STRING_modi0 STR_m STR_o STR_d STR_i "\0"
# 定义字符串常量 STRING_mong0
#define STRING_mong0 STR_m STR_o STR_n STR_g "\0"
# 定义字符串常量 STRING_mongolian0
#define STRING_mongolian0 STR_m STR_o STR_n STR_g STR_o STR_l STR_i STR_a STR_n "\0"
# 定义字符串常量 STRING_mro0
#define STRING_mro0 STR_m STR_r STR_o "\0"
# 定义字符串常量 STRING_mroo0
#define STRING_mroo0 STR_m STR_r STR_o STR_o "\0"
# 定义字符串常量 STRING_mtei0
#define STRING_mtei0 STR_m STR_t STR_e STR_i "\0"
# 定义字符串常量 STRING_mult0
#define STRING_mult0 STR_m STR_u STR_l STR_t "\0"
# 定义字符串常量 STRING_multani0
#define STRING_multani0 STR_m STR_u STR_l STR_t STR_a STR_n STR_i "\0"
# 定义字符串常量 STRING_myanmar0
#define STRING_myanmar0 STR_m STR_y STR_a STR_n STR_m STR_a STR_r "\0"
# 定义字符串常量 STRING_mymr0
#define STRING_mymr0 STR_m STR_y STR_m STR_r "\0"
# 定义字符串常量 STRING_n0
#define STRING_n0 STR_n "\0"
# 定义字符串常量 STRING_nabataean0
#define STRING_nabataean0 STR_n STR_a STR_b STR_a STR_t STR_a STR_e STR_a STR_n "\0"
# 定义字符串常量 STRING_nand0
#define STRING_nand0 STR_n STR_a STR_n STR_d "\0"
# 定义字符串常量 STRING_nandinagari0
#define STRING_nandinagari0 STR_n STR_a STR_n STR_d STR_i STR_n STR_a STR_g STR_a STR_r STR_i "\0"
# 定义字符串常量 STRING_narb0
#define STRING_narb0 STR_n STR_a STR_r STR_b "\0"
# 定义字符串常量 STRING_nbat0
#define STRING_nbat0 STR_n STR_b STR_a STR_t "\0"
# 定义字符串常量 STRING_nchar0
#define STRING_nchar0 STR_n STR_c STR_h STR_a STR_r "\0"
# 定义字符串常量 STRING_nd0
#define STRING_nd0 STR_n STR_d "\0"
# 定义字符串常量 STRING_newa0
#define STRING_newa0 STR_n STR_e STR_w STR_a "\0"
# 定义字符串常量 STRING_newtailue0
#define STRING_newtailue0 STR_n STR_e STR_w STR_t STR_a STR_i STR_l STR_u STR_e "\0"
# 定义字符串常量 STRING_nko0
#define STRING_nko0 STR_n STR_k STR_o "\0"
# 定义字符串常量 STRING_nkoo0
#define STRING_nkoo0 STR_n STR_k STR_o STR_o "\0"
# 定义字符串常量 STRING_nl0
#define STRING_nl0 STR_n STR_l "\0"
# 定义字符串常量 STRING_no0
#define STRING_no0 STR_n STR_o "\0"
# 定义字符串常量 STRING_noncharactercodepoint0
#define STRING_noncharactercodepoint0 STR_n STR_o STR_n STR_c STR_h STR_a STR_r STR_a STR_c STR_t STR_e STR_r STR_c STR_o STR_d STR_e STR_p STR_o STR_i STR_n STR_t "\0"
# 定义字符串常量 STRING_nshu0
#define STRING_nshu0 STR_n STR_s STR_h STR_u "\0"
# 定义字符串常量 STRING_nushu0
#define STRING_nushu0 STR_n STR_u STR_s STR_h STR_u "\0"
# 定义字符串常量 STRING_nyiakengpuachuehmong0
#define STRING_nyiakengpuachuehmong0 STR_n STR_y STR_i STR_a STR_k STR_e STR_n STR_g STR_p STR_u STR_a STR_c STR_h STR_u STR_e STR_h STR_m STR_o STR_n STR_g "\0"
# 定义字符串常量 STRING_ogam0
#define STRING_ogam0 STR_o STR_g STR_a STR_m "\0"
# 定义字符串常量 STRING_ogham0
#define STRING_ogham0 STR_o STR_g STR_h STR_a STR_m "\0"
# 定义字符串常量 STRING_olchiki0
#define STRING_olchiki0 STR_o STR_l STR_c STR_h STR_i STR_k STR_i "\0"
# 定义字符串常量 STRING_olck0
#define STRING_olck0 STR_o STR_l STR_c STR_k "\0"
# 定义字符串常量 STRING_oldhungarian0
#define STRING_oldhungarian0 STR_o STR_l STR_d STR_h STR_u STR_n STR_g STR_a STR_r STR_i STR_a STR_n "\0"
# 定义字符串常量 STRING_olditalic0
#define STRING_olditalic0 STR_o STR_l STR_d STR_i STR_t STR_a STR_l STR_i STR_c "\0"
# 定义字符串常量 STRING_oldnortharabian0
#define STRING_oldnortharabian0 STR_o STR_l STR_d STR_n STR_o STR_r STR_t STR_h STR_a STR_r STR_a STR_b STR_i STR_a STR_n "\0"
# 定义字符串常量 STRING_oldpermic0
#define STRING_oldpermic0 STR_o STR_l STR_d STR_p STR_e STR_r STR_m STR_i STR_c "\0"
# 定义字符串常量 STRING_oldpersian0
#define STRING_oldpersian0 STR_o STR_l STR_d STR_p STR_e STR_r STR_s STR_i STR_a STR_n "\0"
# 定义字符串常量 STRING_oldsogdian0
#define STRING_oldsogdian0 STR_o STR_l STR_d STR_s STR_o STR_g STR_d STR_i STR_a STR_n "\0"
# 定义字符串常量 STRING_oldsoutharabian0
#define STRING_oldsoutharabian0 STR_o STR_l STR_d STR_s STR_o STR_u STR_t STR_h STR_a STR_r STR_a STR_b STR_i STR_a STR_n "\0"
# 定义字符串常量 STRING_oldturkic0
#define STRING_oldturkic0 STR_o STR_l STR_d STR_t STR_u STR_r STR_k STR_i STR_c "\0"
# 定义字符串常量 STRING_olduyghur0
#define STRING_olduyghur0 STR_o STR_l STR_d STR_u STR_y STR_g STR_h STR_u STR_r "\0"
# 定义字符串常量 STRING_oriya0
#define STRING_oriya0 STR_o STR_r STR_i STR_y STR_a "\0"
# 定义字符串常量 STRING_orkh0
#define STRING_orkh0 STR_o STR_r STR_k STR_h "\0"
# 定义字符串常量 STRING_orya0
#define STRING_orya0 STR_o STR_r STR_y STR_a "\0"
# 定义字符串常量 STRING_osage0
#define STRING_osage0 STR_o STR_s STR_a STR_g STR_e "\0"
# 定义字符串常量 STRING_osge0
#define STRING_osge0 STR_o STR_s STR_g STR_e "\0"
# 定义字符串常量 STRING_osma0
#define STRING_osma0 STR_o STR_s STR_m STR_a "\0"
# 定义字符串常量 STRING_osmanya0
#define STRING_osmanya0 STR_o STR_s STR_m STR_a STR_n STR_y STR_a "\0"
# 定义字符串常量 STRING_ougr0
#define STRING_ougr0 STR_o STR_u STR_g STR_r "\0"
# 定义字符串宏，将 STR_p 和空字符连接起来
#define STRING_p0 STR_p "\0"
# 定义字符串宏，将 STR_p、STR_a、STR_h、STR_w、STR_h、STR_m、STR_o、STR_n、STR_g 和空字符连接起来
#define STRING_pahawhhmong0 STR_p STR_a STR_h STR_a STR_w STR_h STR_h STR_m STR_o STR_n STR_g "\0"
# 定义字符串宏，将 STR_p、STR_a、STR_l、STR_m 和空字符连接起来
#define STRING_palm0 STR_p STR_a STR_l STR_m "\0"
# 定义字符串宏，将 STR_p、STR_a、STR_l、STR_m、STR_y、STR_r、STR_e、STR_n、STR_e 和空字符连接起来
#define STRING_palmyrene0 STR_p STR_a STR_l STR_m STR_y STR_r STR_e STR_n STR_e "\0"
# 定义字符串宏，将 STR_p、STR_a、STR_t、STR_s、STR_y、STR_n 和空字符连接起来
#define STRING_patsyn0 STR_p STR_a STR_t STR_s STR_y STR_n "\0"
# 定义字符串宏，将 STR_p、STR_a、STR_t、STR_t、STR_e、STR_r、STR_n、STR_s、STR_y、STR_n、STR_t、STR_a、STR_x 和空字符连接起来
#define STRING_patternsyntax0 STR_p STR_a STR_t STR_t STR_e STR_r STR_n STR_s STR_y STR_n STR_t STR_a STR_x "\0"
# 定义字符串宏，将 STR_p、STR_a、STR_t、STR_t、STR_e、STR_r、STR_n、STR_w、STR_h、STR_i、STR_t、STR_e、STR_s、STR_p、STR_a、STR_c、STR_e 和空字符连接起来
#define STRING_patternwhitespace0 STR_p STR_a STR_t STR_t STR_e STR_r STR_n STR_w STR_h STR_i STR_t STR_e STR_s STR_p STR_a STR_c STR_e "\0"
# 定义字符串宏，将 STR_p、STR_a、STR_t、STR_w、STR_s 和空字符连接起来
#define STRING_patws0 STR_p STR_a STR_t STR_w STR_s "\0"
# 定义字符串宏，将 STR_p、STR_a、STR_u、STR_c 和空字符连接起来
#define STRING_pauc0 STR_p STR_a STR_u STR_c "\0"
# 定义字符串宏，将 STR_p、STR_a、STR_u、STR_c、STR_i、STR_n、STR_h、STR_a、STR_u 和空字符连接起来
#define STRING_paucinhau0 STR_p STR_a STR_u STR_c STR_i STR_n STR_h STR_a STR_u "\0"
# 定义字符串宏，将 STR_p、STR_c 和空字符连接起来
#define STRING_pc0 STR_p STR_c "\0"
# 定义字符串宏，将 STR_p、STR_c、STR_m 和空字符连接起来
#define STRING_pcm0 STR_p STR_c STR_m "\0"
# 定义字符串宏，将 STR_p、STR_d 和空字符连接起来
#define STRING_pd0 STR_p STR_d "\0"
# 定义字符串宏，将 STR_p、STR_e 和空字符连接起来
#define STRING_pe0 STR_p STR_e "\0"
# 定义字符串宏，将 STR_p、STR_e、STR_r、STR_m 和空字符连接起来
#define STRING_perm0 STR_p STR_e STR_r STR_m "\0"
# 定义字符串宏，将 STR_p、STR_f 和空字符连接起来
#define STRING_pf0 STR_p STR_f "\0"
# 定义字符串宏，将 STR_p、STR_h、STR_a、STR_g 和空字符连接起来
#define STRING_phag0 STR_p STR_h STR_a STR_g "\0"
# 定义字符串宏，将 STR_p、STR_h、STR_a、STR_g、STR_s、STR_p、STR_a 和空字符连接起来
#define STRING_phagspa0 STR_p STR_h STR_a STR_g STR_s STR_p STR_a "\0"
# 定义字符串宏，将 STR_p、STR_h、STR_l、STR_i 和空字符连接起来
#define STRING_phli0 STR_p STR_h STR_l STR_i "\0"
# 定义字符串宏，将 STR_p、STR_h、STR_l、STR_p 和空字符连接起来
#define STRING_phlp0 STR_p STR_h STR_l STR_p "\0"
# 定义字符串宏，将 STR_p、STR_h、STR_n、STR_x 和空字符连接起来
#define STRING_phnx0 STR_p STR_h STR_n STR_x "\0"
# 定义字符串宏，将 STR_p、STR_h、STR_o、STR_e、STR_n、STR_i、STR_c、STR_i、STR_a、STR_n 和空字符连接起来
#define STRING_phoenician0 STR_p STR_h STR_o STR_e STR_n STR_i STR_c STR_i STR_a STR_n "\0"
# 定义字符串宏，将 STR_p、STR_i 和空字符连接起来
#define STRING_pi0 STR_p STR_i "\0"
# 定义字符串宏，将 STR_p、STR_l、STR_r、STR_d 和空字符连接起来
#define STRING_plrd0 STR_p STR_l STR_r STR_d "\0"
# 定义字符串宏，将 STR_p、STR_o 和空字符连接起来
#define STRING_po0 STR_p STR_o "\0"
# 定义字符串宏，将 STR_p、STR_r、STR_e、STR_p、STR_e、STR_n、STR_d、STR_e、STR_d、STR_c、STR_o、STR_n、STR_c、STR_a、STR_t、STR_i、STR_o、STR_n、STR_m、STR_a、STR_r、STR_k 和空字符连接起来
#define STRING_prependedconcatenationmark0 STR_p STR_r STR_e STR_p STR_e STR_n STR_d STR_e STR_d STR_c STR_o STR_n STR_c STR_a STR_t STR_i STR_o STR_n STR_m STR_a STR_r STR_k "\0"
# 定义字符串宏，将 STR_p、STR_r、STR_t、STR_i 和空字符连接起来
#define STRING_prti0 STR_p STR_r STR_t STR_i "\0"
# 定义字符串宏，将 STR_p、STR_s 和空字符连接起来
#define STRING_ps0 STR_p STR_s "\0"
# 定义字符串宏，将 STR_p、STR_s、STR_a、STR_l、STR_t、STR_e、STR_r、STR_p、STR_a、STR_h、STR_l、STR_a、STR_v、STR_i 和空字符连接起来
#define STRING_psalterpahlavi0 STR_p STR_s STR_a STR_l STR_t STR_e STR_r STR_p STR_a STR_h STR_l STR_a STR_v STR_i "\0"
# 定义字符串宏，将 STR_q、STR_a、STR_a、STR_c 和空字符连接起来
#define STRING_qaac0 STR_q STR_a STR_a STR_c "\0"
# 定义字符串常量 STRING_qaai0，值为 "qaai\0"
#define STRING_qaai0 STR_q STR_a STR_a STR_i "\0"
# 定义字符串常量 STRING_qmark0，值为 "qmark\0"
#define STRING_qmark0 STR_q STR_m STR_a STR_r STR_k "\0"
# 定义字符串常量 STRING_quotationmark0，值为 "quotationmark\0"
#define STRING_quotationmark0 STR_q STR_u STR_o STR_t STR_a STR_t STR_i STR_o STR_n STR_m STR_a STR_r STR_k "\0"
# 定义字符串常量 STRING_radical0，值为 "radical\0"
#define STRING_radical0 STR_r STR_a STR_d STR_i STR_c STR_a STR_l "\0"
# 定义字符串常量 STRING_regionalindicator0，值为 "regionalindicator\0"
#define STRING_regionalindicator0 STR_r STR_e STR_g STR_i STR_o STR_n STR_a STR_l STR_i STR_n STR_d STR_i STR_c STR_a STR_t STR_o STR_r "\0"
# 定义字符串常量 STRING_rejang0，值为 "rejang\0"
#define STRING_rejang0 STR_r STR_e STR_j STR_a STR_n STR_g "\0"
# 定义字符串常量 STRING_ri0，值为 "ri\0"
#define STRING_ri0 STR_r STR_i "\0"
# 定义字符串常量 STRING_rjng0，值为 "rjng\0"
#define STRING_rjng0 STR_r STR_j STR_n STR_g "\0"
# 定义字符串常量 STRING_rohg0，值为 "rohg\0"
#define STRING_rohg0 STR_r STR_o STR_h STR_g "\0"
# 定义字符串常量 STRING_runic0，值为 "runic\0"
#define STRING_runic0 STR_r STR_u STR_n STR_i STR_c "\0"
# 定义字符串常量 STRING_runr0，值为 "runr\0"
#define STRING_runr0 STR_r STR_u STR_n STR_r "\0"
# 定义字符串常量 STRING_s0，值为 "s\0"
#define STRING_s0 STR_s "\0"
# 定义字符串常量 STRING_samaritan0，值为 "samaritan\0"
#define STRING_samaritan0 STR_s STR_a STR_m STR_a STR_r STR_i STR_t STR_a STR_n "\0"
# 定义字符串常量 STRING_samr0，值为 "samr\0"
#define STRING_samr0 STR_s STR_a STR_m STR_r "\0"
# 定义字符串常量 STRING_sarb0，值为 "sarb\0"
#define STRING_sarb0 STR_s STR_a STR_r STR_b "\0"
# 定义字符串常量 STRING_saur0，值为 "saur\0"
#define STRING_saur0 STR_s STR_a STR_u STR_r "\0"
# 定义字符串常量 STRING_saurashtra0，值为 "saurashtra\0"
#define STRING_saurashtra0 STR_s STR_a STR_u STR_r STR_a STR_s STR_h STR_t STR_r STR_a "\0"
# 定义字符串常量 STRING_sc0，值为 "sc\0"
#define STRING_sc0 STR_s STR_c "\0"
# 定义字符串常量 STRING_sd0，值为 "sd\0"
#define STRING_sd0 STR_s STR_d "\0"
# 定义字符串常量 STRING_sentenceterminal0，值为 "sentenceterminal\0"
#define STRING_sentenceterminal0 STR_s STR_e STR_n STR_t STR_e STR_n STR_c STR_e STR_t STR_e STR_r STR_m STR_i STR_n STR_a STR_l "\0"
# 定义字符串常量 STRING_sgnw0，值为 "sgnw\0"
#define STRING_sgnw0 STR_s STR_g STR_n STR_w "\0"
# 定义字符串常量 STRING_sharada0，值为 "sharada\0"
#define STRING_sharada0 STR_s STR_h STR_a STR_r STR_a STR_d STR_a "\0"
# 定义字符串常量 STRING_shavian0，值为 "shavian\0"
#define STRING_shavian0 STR_s STR_h STR_a STR_v STR_i STR_a STR_n "\0"
# 定义字符串常量 STRING_shaw0，值为 "shaw\0"
#define STRING_shaw0 STR_s STR_h STR_a STR_w "\0"
# 定义字符串常量 STRING_shrd0，值为 "shrd\0"
#define STRING_shrd0 STR_s STR_h STR_r STR_d "\0"
# 定义字符串常量 STRING_sidd0，值为 "sidd\0"
#define STRING_sidd0 STR_s STR_i STR_d STR_d "\0"
# 定义字符串常量 STRING_siddham0，值为 "siddham\0"
#define STRING_siddham0 STR_s STR_i STR_d STR_d STR_h STR_a STR_m "\0"
# 定义字符串常量 STRING_signwriting0，值为 "signwriting\0"
#define STRING_signwriting0 STR_s STR_i STR_g STR_n STR_w STR_r STR_i STR_t STR_i STR_n STR_g "\0"
# 定义字符串常量 STRING_sind0，值为 "sind\0"
#define STRING_sind0 STR_s STR_i STR_n STR_d "\0"
# 定义字符串常量 STRING_sinh0，值为 "sinh\0"
#define STRING_sinh0 STR_s STR_i STR_n STR_h "\0"
# 定义字符串常量 STRING_sinhala0，值为 "sinhala\0"
#define STRING_sinhala0 STR_s STR_i STR_n STR_h STR_a STR_l STR_a "\0"
# 定义字符串常量 STRING_sk0，值为 "s" + "k" + "\0"
#define STRING_sk0 STR_s STR_k "\0"
# 定义字符串常量 STRING_sm0，值为 "s" + "m" + "\0"
#define STRING_sm0 STR_s STR_m "\0"
# 定义字符串常量 STRING_so0，值为 "s" + "o" + "\0"
#define STRING_so0 STR_s STR_o "\0"
# 定义字符串常量 STRING_softdotted0，值为 "s" + "o" + "f" + "t" + "d" + "o" + "t" + "t" + "e" + "d" + "\0"
#define STRING_softdotted0 STR_s STR_o STR_f STR_t STR_d STR_o STR_t STR_t STR_e STR_d "\0"
# 定义字符串常量 STRING_sogd0，值为 "s" + "o" + "g" + "d" + "\0"
#define STRING_sogd0 STR_s STR_o STR_g STR_d "\0"
# 定义字符串常量 STRING_sogdian0，值为 "s" + "o" + "g" + "d" + "i" + "a" + "n" + "\0"
#define STRING_sogdian0 STR_s STR_o STR_g STR_d STR_i STR_a STR_n "\0"
# 定义字符串常量 STRING_sogo0，值为 "s" + "o" + "g" + "o" + "\0"
#define STRING_sogo0 STR_s STR_o STR_g STR_o "\0"
# 定义字符串常量 STRING_sora0，值为 "s" + "o" + "r" + "a" + "\0"
#define STRING_sora0 STR_s STR_o STR_r STR_a "\0"
# 定义字符串常量 STRING_sorasompeng0，值为 "s" + "o" + "r" + "a" + "s" + "o" + "m" + "p" + "e" + "n" + "g" + "\0"
#define STRING_sorasompeng0 STR_s STR_o STR_r STR_a STR_s STR_o STR_m STR_p STR_e STR_n STR_g "\0"
# 定义字符串常量 STRING_soyo0，值为 "s" + "o" + "y" + "o" + "\0"
#define STRING_soyo0 STR_s STR_o STR_y STR_o "\0"
# 定义字符串常量 STRING_soyombo0，值为 "s" + "o" + "y" + "o" + "m" + "b" + "o" + "\0"
#define STRING_soyombo0 STR_s STR_o STR_y STR_o STR_m STR_b STR_o "\0"
# 定义字符串常量 STRING_space0，值为 "s" + "p" + "a" + "c" + "e" + "\0"
#define STRING_space0 STR_s STR_p STR_a STR_c STR_e "\0"
# 定义字符串常量 STRING_sterm0，值为 "s" + "t" + "e" + "r" + "m" + "\0"
#define STRING_sterm0 STR_s STR_t STR_e STR_r STR_m "\0"
# 定义字符串常量 STRING_sund0，值为 "s" + "u" + "n" + "d" + "\0"
#define STRING_sund0 STR_s STR_u STR_n STR_d "\0"
# 定义字符串常量 STRING_sundanese0，值为 "s" + "u" + "n" + "d" + "a" + "n" + "e" + "s" + "e" + "\0"
#define STRING_sundanese0 STR_s STR_u STR_n STR_d STR_a STR_n STR_e STR_s STR_e "\0"
# 定义字符串常量 STRING_sylo0，值为 "s" + "y" + "l" + "o" + "\0"
#define STRING_sylo0 STR_s STR_y STR_l STR_o "\0"
# 定义字符串常量 STRING_sylotinagri0，值为 "s" + "y" + "l" + "o" + "t" + "i" + "n" + "a" + "g" + "r" + "i" + "\0"
#define STRING_sylotinagri0 STR_s STR_y STR_l STR_o STR_t STR_i STR_n STR_a STR_g STR_r STR_i "\0"
# 定义字符串常量 STRING_syrc0，值为 "s" + "y" + "r" + "c" + "\0"
#define STRING_syrc0 STR_s STR_y STR_r STR_c "\0"
# 定义字符串常量 STRING_syriac0，值为 "s" + "y" + "r" + "i" + "a" + "c" + "\0"
#define STRING_syriac0 STR_s STR_y STR_r STR_i STR_a STR_c "\0"
# 定义字符串常量 STRING_tagalog0，值为 "t" + "a" + "g" + "a" + "l" + "o" + "g" + "\0"
#define STRING_tagalog0 STR_t STR_a STR_g STR_a STR_l STR_o STR_g "\0"
# 定义字符串常量 STRING_tagb0，值为 "t" + "a" + "g" + "b" + "\0"
#define STRING_tagb0 STR_t STR_a STR_g STR_b "\0"
# 定义字符串常量 STRING_tagbanwa0，值为 "t" + "a" + "g" + "b" + "a" + "n" + "w" + "a" + "\0"
#define STRING_tagbanwa0 STR_t STR_a STR_g STR_b STR_a STR_n STR_w STR_a "\0"
# 定义字符串常量 STRING_taile0，值为 "t" + "a" + "i" + "l" + "e" + "\0"
#define STRING_taile0 STR_t STR_a STR_i STR_l STR_e "\0"
# 定义字符串常量 STRING_taitham0，值为 "t" + "a" + "i" + "t" + "h" + "a" + "m" + "\0"
#define STRING_taitham0 STR_t STR_a STR_i STR_t STR_h STR_a STR_m "\0"
# 定义字符串常量 STRING_taiviet0，值为 "t" + "a" + "i" + "v" + "i" + "e" + "t" + "\0"
#define STRING_taiviet0 STR_t STR_a STR_i STR_v STR_i STR_e STR_t "\0"
# 定义字符串常量 STRING_takr0，值为 "t" + "a" + "k" + "r" + "\0"
#define STRING_takr0 STR_t STR_a STR_k STR_r "\0"
# 定义字符串常量 STRING_takri0，值为 "t" + "a" + "k" + "r" + "i" + "\0"
#define STRING_takri0 STR_t STR_a STR_k STR_r STR_i "\0"
# 定义字符串常量 STRING_tale0，值为 "t" + "a" + "l" + "e" + "\0"
#define STRING_tale0 STR_t STR_a STR_l STR_e "\0"
# 定义字符串常量 STRING_talu0，值为 "t" + "a" + "l" + "u" + "\0"
#define STRING_talu0 STR_t STR_a STR_l STR_u "\0"
# 定义字符串常量 STRING_tamil0，值为 "t" + "a" + "m" + "i" + "l" + "\0"
#define STRING_tamil0 STR_t STR_a STR_m STR_i STR_l "\0"
# 定义字符串常量 STRING_taml0，值为 "t" + "a" + "m" + "l" + "\0"
#define STRING_taml0 STR_t STR_a STR_m STR_l "\0"
# 定义字符串常量 STRING_tang0，值为 "t" + "a" + "n" + "g" + "\0"
#define STRING_tang0 STR_t STR_a STR_n STR_g "\0"
# 定义字符串常量 STRING_tangsa0，值为 "t" + "a" + "n" + "g" + "s" + "a" + "\0"
#define STRING_tangsa0 STR_t STR_a STR_n STR_g STR_s STR_a "\0"
# 定义字符串常量 STRING_tangut0，值为 "tangut"，以空字符结尾
#define STRING_tangut0 STR_t STR_a STR_n STR_g STR_u STR_t "\0"
# 定义字符串常量 STRING_tavt0，值为 "tavt"，以空字符结尾
#define STRING_tavt0 STR_t STR_a STR_v STR_t "\0"
# 定义字符串常量 STRING_telu0，值为 "telu"，以空字符结尾
#define STRING_telu0 STR_t STR_e STR_l STR_u "\0"
# 定义字符串常量 STRING_telugu0，值为 "telugu"，以空字符结尾
#define STRING_telugu0 STR_t STR_e STR_l STR_u STR_g STR_u "\0"
# 定义字符串常量 STRING_term0，值为 "term"，以空字符结尾
#define STRING_term0 STR_t STR_e STR_r STR_m "\0"
# 定义字符串常量 STRING_terminalpunctuation0，值为 "terminalpunctuation"，以空字符结尾
#define STRING_terminalpunctuation0 STR_t STR_e STR_r STR_m STR_i STR_n STR_a STR_l STR_p STR_u STR_n STR_c STR_t STR_u STR_a STR_t STR_i STR_o STR_n "\0"
# 定义字符串常量 STRING_tfng0，值为 "tfng"，以空字符结尾
#define STRING_tfng0 STR_t STR_f STR_n STR_g "\0"
# 定义字符串常量 STRING_tglg0，值为 "tglg"，以空字符结尾
#define STRING_tglg0 STR_t STR_g STR_l STR_g "\0"
# 定义字符串常量 STRING_thaa0，值为 "thaa"，以空字符结尾
#define STRING_thaa0 STR_t STR_h STR_a STR_a "\0"
# 定义字符串常量 STRING_thaana0，值为 "thaana"，以空字符结尾
#define STRING_thaana0 STR_t STR_h STR_a STR_a STR_n STR_a "\0"
# 定义字符串常量 STRING_thai0，值为 "thai"，以空字符结尾
#define STRING_thai0 STR_t STR_h STR_a STR_i "\0"
# 定义字符串常量 STRING_tibetan0，值为 "tibetan"，以空字符结尾
#define STRING_tibetan0 STR_t STR_i STR_b STR_e STR_t STR_a STR_n "\0"
# 定义字符串常量 STRING_tibt0，值为 "tibt"，以空字符结尾
#define STRING_tibt0 STR_t STR_i STR_b STR_t "\0"
# 定义字符串常量 STRING_tifinagh0，值为 "tifinagh"，以空字符结尾
#define STRING_tifinagh0 STR_t STR_i STR_f STR_i STR_n STR_a STR_g STR_h "\0"
# 定义字符串常量 STRING_tirh0，值为 "tirh"，以空字符结尾
#define STRING_tirh0 STR_t STR_i STR_r STR_h "\0"
# 定义字符串常量 STRING_tirhuta0，值为 "tirhuta"，以空字符结尾
#define STRING_tirhuta0 STR_t STR_i STR_r STR_h STR_u STR_t STR_a "\0"
# 定义字符串常量 STRING_tnsa0，值为 "tnsa"，以空字符结尾
#define STRING_tnsa0 STR_t STR_n STR_s STR_a "\0"
# 定义字符串常量 STRING_toto0，值为 "toto"，以空字符结尾
#define STRING_toto0 STR_t STR_o STR_t STR_o "\0"
# 定义字符串常量 STRING_ugar0，值为 "ugar"，以空字符结尾
#define STRING_ugar0 STR_u STR_g STR_a STR_r "\0"
# 定义字符串常量 STRING_ugaritic0，值为 "ugaritic"，以空字符结尾
#define STRING_ugaritic0 STR_u STR_g STR_a STR_r STR_i STR_t STR_i STR_c "\0"
# 定义字符串常量 STRING_uideo0，值为 "uideo"，以空字符结尾
#define STRING_uideo0 STR_u STR_i STR_d STR_e STR_o "\0"
# 定义字符串常量 STRING_unifiedideograph0，值为 "unifiedideograph"，以空字符结尾
#define STRING_unifiedideograph0 STR_u STR_n STR_i STR_f STR_i STR_e STR_d STR_i STR_d STR_e STR_o STR_g STR_r STR_a STR_p STR_h "\0"
# 定义字符串常量 STRING_unknown0，值为 "unknown"，以空字符结尾
#define STRING_unknown0 STR_u STR_n STR_k STR_n STR_o STR_w STR_n "\0"
# 定义字符串常量 STRING_upper0，值为 "upper"，以空字符结尾
#define STRING_upper0 STR_u STR_p STR_p STR_e STR_r "\0"
# 定义字符串常量 STRING_uppercase0，值为 "uppercase"，以空字符结尾
#define STRING_uppercase0 STR_u STR_p STR_p STR_e STR_r STR_c STR_a STR_s STR_e "\0"
# 定义字符串常量 STRING_vai0，值为 "vai"，以空字符结尾
#define STRING_vai0 STR_v STR_a STR_i "\0"
# 定义字符串常量 STRING_vaii0，值为 "vaii"，以空字符结尾
#define STRING_vaii0 STR_v STR_a STR_i STR_i "\0"
# 定义字符串常量 STRING_variationselector0，值为 "variationselector"，以空字符结尾
#define STRING_variationselector0 STR_v STR_a STR_r STR_i STR_a STR_t STR_i STR_o STR_n STR_s STR_e STR_l STR_e STR_c STR_t STR_o STR_r "\0"
# 定义字符串常量 STRING_vith0，值为 "vith"，以空字符结尾
#define STRING_vith0 STR_v STR_i STR_t STR_h "\0"
#define STRING_vithkuqi0 STR_v STR_i STR_t STR_h STR_k STR_u STR_q STR_i "\0"
#define STRING_vs0 STR_v STR_s "\0"
#define STRING_wancho0 STR_w STR_a STR_n STR_c STR_h STR_o "\0"
#define STRING_wara0 STR_w STR_a STR_r STR_a "\0"
#define STRING_warangciti0 STR_w STR_a STR_r STR_a STR_n STR_g STR_c STR_i STR_t STR_i "\0"
#define STRING_wcho0 STR_w STR_c STR_h STR_o "\0"
#define STRING_whitespace0 STR_w STR_h STR_i STR_t STR_e STR_s STR_p STR_a STR_c STR_e "\0"
#define STRING_wspace0 STR_w STR_s STR_p STR_a STR_c STR_e "\0"
#define STRING_xan0 STR_x STR_a STR_n "\0"
#define STRING_xidc0 STR_x STR_i STR_d STR_c "\0"
#define STRING_xidcontinue0 STR_x STR_i STR_d STR_c STR_o STR_n STR_t STR_i STR_n STR_u STR_e "\0"
#define STRING_xids0 STR_x STR_i STR_d STR_s "\0"
#define STRING_xidstart0 STR_x STR_i STR_d STR_s STR_t STR_a STR_r STR_t "\0"
#define STRING_xpeo0 STR_x STR_p STR_e STR_o "\0"
#define STRING_xps0 STR_x STR_p STR_s "\0"
#define STRING_xsp0 STR_x STR_s STR_p "\0"
#define STRING_xsux0 STR_x STR_s STR_u STR_x "\0"
#define STRING_xuc0 STR_x STR_u STR_c "\0"
#define STRING_xwd0 STR_x STR_w STR_d "\0"
#define STRING_yezi0 STR_y STR_e STR_z STR_i "\0"
#define STRING_yezidi0 STR_y STR_e STR_z STR_i STR_d STR_i "\0"
#define STRING_yi0 STR_y STR_i "\0"
#define STRING_yiii0 STR_y STR_i STR_i STR_i "\0"
#define STRING_z0 STR_z "\0"
#define STRING_zanabazarsquare0 STR_z STR_a STR_n STR_a STR_b STR_a STR_z STR_a STR_r STR_s STR_q STR_u STR_a STR_r STR_e "\0"
#define STRING_zanb0 STR_z STR_a STR_n STR_b "\0"
#define STRING_zinh0 STR_z STR_i STR_n STR_h "\0"
#define STRING_zl0 STR_z STR_l "\0"
#define STRING_zp0 STR_z STR_p "\0"
#define STRING_zs0 STR_z STR_s "\0"
#define STRING_zyyy0 STR_z STR_y STR_y STR_y "\0"
#define STRING_zzzz0 STR_z STR_z STR_z STR_z "\0"

};

const size_t PRIV(utt_size) = sizeof(PRIV(utt)) / sizeof(ucp_type_table);

#endif /* SUPPORT_UNICODE */

/* End of pcre2_ucptables.c */
```