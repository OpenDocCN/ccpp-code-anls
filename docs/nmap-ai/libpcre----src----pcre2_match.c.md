# `nmap\libpcre\src\pcre2_match.c`

```
/*************************************************
*      Perl-Compatible Regular Expressions       *
*************************************************/

/* PCRE is a library of functions to support regular expressions whose syntax
and semantics are as close as possible to those of the Perl 5 language.

                       Written by Philip Hazel
     Original API code Copyright (c) 1997-2012 University of Cambridge
          New API code Copyright (c) 2015-2022 University of Cambridge

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
INTERRUPTION) HOWEVER CAUSED AND ON ANY THEORY OF LIABILITY, WHETHER IN
CONTRACT, STRICT LIABILITY, OR TORT (INCLUDING NEGLIGENCE OR OTHERWISE)
#ifdef HAVE_CONFIG_H
#include "config.h"
#endif

// 如果有配置文件，则包含配置文件


/* These defines enable debugging code */
/* #define DEBUG_FRAMES_DISPLAY */
/* #define DEBUG_SHOW_OPS */
/* #define DEBUG_SHOW_RMATCH */

// 这些定义启用调试代码


#ifdef DEBUG_FRAMES_DISPLAY
#include <stdarg.h>
#endif

// 如果启用了帧显示调试，则包含可变参数的标准头文件


#define NLBLOCK mb              /* Block containing newline information */
#define PSSTART start_subject   /* Field containing processed string start */
#define PSEND   end_subject     /* Field containing processed string end */

// 定义包含换行信息的块，以及包含处理字符串起始和结束的字段


#include "pcre2_internal.h"

// 包含内部 PCRE2 头文件


#define RECURSE_UNSET 0xffffffffu  /* Bigger than max group number */

// 未设置递归的值，大于最大组号


#define PUBLIC_MATCH_OPTIONS \
  (PCRE2_ANCHORED|PCRE2_ENDANCHORED|PCRE2_NOTBOL|PCRE2_NOTEOL|PCRE2_NOTEMPTY| \
   PCRE2_NOTEMPTY_ATSTART|PCRE2_NO_UTF_CHECK|PCRE2_PARTIAL_HARD| \
   PCRE2_PARTIAL_SOFT|PCRE2_NO_JIT|PCRE2_COPY_MATCHED_SUBJECT)

// 允许在匹配时使用的公共选项的掩码


#define PUBLIC_JIT_MATCH_OPTIONS \
   (PCRE2_NO_UTF_CHECK|PCRE2_NOTBOL|PCRE2_NOTEOL|PCRE2_NOTEMPTY|\
    PCRE2_NOTEMPTY_ATSTART|PCRE2_PARTIAL_SOFT|PCRE2_PARTIAL_HARD|\
    PCRE2_COPY_MATCHED_SUBJECT)

// 允许在 JIT 匹配时使用的公共选项的掩码


#define MATCH_MATCH        1
#define MATCH_NOMATCH      0

// 匹配成功和不匹配的返回值


#define MATCH_ACCEPT       (-999)
#define MATCH_KETRPOS      (-998)
#define MATCH_COMMIT       (-997)
#define MATCH_PRUNE        (-996)

// 匹配函数中使用的特殊内部返回值，保持足够负以避免外部错误代码
# 定义匹配跳过的返回值
#define MATCH_SKIP         (-995)
# 定义匹配跳过参数的返回值
#define MATCH_SKIP_ARG     (-994)
# 定义匹配成功的返回值
#define MATCH_THEN         (-993)
# 定义回溯的最大返回值
#define MATCH_BACKTRACK_MAX MATCH_THEN
# 定义回溯的最小返回值
#define MATCH_BACKTRACK_MIN MATCH_COMMIT

# 组帧类型数值。零表示该帧不是组帧。低16位用于数据（例如捕获编号）。组帧用于大多数组，以便在结束时轻松获取有关开始的信息，而无需通过中间帧（回溯点）进行扫描。
#define GF_CAPTURE     0x00010000u
#define GF_NOCAPTURE   0x00020000u
#define GF_CONDASSERT  0x00030000u
#define GF_RECURSE     0x00040000u

# 组帧类型的标识和数据部分的掩码
#define GF_IDMASK(a)   ((a) & 0xffff0000u)
#define GF_DATAMASK(a) ((a) & 0x0000ffffu)

# 重复类型
enum { REPTYPE_MIN, REPTYPE_MAX, REPTYPE_POS };

# 常见重复的最小和最大值；最大值为UINT32_MAX => 无穷大。
static const uint32_t rep_min[] = {
  0, 0,       /* * and *? */
  1, 1,       /* + and +? */
  0, 0,       /* ? and ?? */
  0, 0,       /* dummy placefillers for OP_CR[MIN]RANGE */
  0, 1, 0 };  /* OP_CRPOS{STAR, PLUS, QUERY} */

static const uint32_t rep_max[] = {
  UINT32_MAX, UINT32_MAX,      /* * and *? */
  UINT32_MAX, UINT32_MAX,      /* + and +? */
  1, 1,                        /* ? and ?? */
  0, 0,                        /* dummy placefillers for OP_CR[MIN]RANGE */
  UINT32_MAX, UINT32_MAX, 1 }; /* OP_CRPOS{STAR, PLUS, QUERY} */

# 重复类型 - 必须包括OP_CRPOSRANGE（上面不需要）
static const uint32_t rep_typ[] = {
  REPTYPE_MAX, REPTYPE_MIN,    /* * and *? */
  REPTYPE_MAX, REPTYPE_MIN,    /* + and +? */
  REPTYPE_MAX, REPTYPE_MIN,    /* ? and ?? */
  REPTYPE_MAX, REPTYPE_MIN,    /* OP_CRRANGE and OP_CRMINRANGE */
  REPTYPE_POS, REPTYPE_POS,    /* OP_CRPOSSTAR, OP_CRPOSPLUS */
  REPTYPE_POS, REPTYPE_POS };  /* OP_CRPOSQUERY, OP_CRPOSRANGE */
/* Numbers for RMATCH calls at backtracking points. When these lists are
changed, the code at RETURN_SWITCH below must be updated in sync.  */
// 定义在回溯点的 RMATCH 调用的数字。当这些列表被改变时，下面的 RETURN_SWITCH 代码必须同步更新。

enum { RM1=1, RM2,  RM3,  RM4,  RM5,  RM6,  RM7,  RM8,  RM9,  RM10,
       RM11,  RM12, RM13, RM14, RM15, RM16, RM17, RM18, RM19, RM20,
       RM21,  RM22, RM23, RM24, RM25, RM26, RM27, RM28, RM29, RM30,
       RM31,  RM32, RM33, RM34, RM35, RM36 };

#ifdef SUPPORT_WIDE_CHARS
enum { RM100=100, RM101 };
#endif

#ifdef SUPPORT_UNICODE
enum { RM200=200, RM201, RM202, RM203, RM204, RM205, RM206, RM207,
       RM208,     RM209, RM210, RM211, RM212, RM213, RM214, RM215,
       RM216,     RM217, RM218, RM219, RM220, RM221, RM222, RM223,
       RM224,     RM225 };
#endif

/* Define short names for general fields in the current backtrack frame, which
is always pointed to by the F variable. Occasional references to fields in
other frames are written out explicitly. There are also some fields in the
current frame whose names start with "temp" that are used for short-term,
localised backtracking memory. These are #defined with Lxxx names at the point
of use and undefined afterwards. */
// 为当前回溯帧中的一般字段定义简短的名称，该帧始终由变量 F 指向。偶尔对其他帧中的字段的引用会被明确写出。当前帧中还有一些字段，它们的名称以 "temp" 开头，用于短期的、局部的回溯记忆。这些在使用时用 Lxxx 名称进行 #define，然后在使用后取消定义。

#define Fback_frame        F->back_frame
#define Fcapture_last      F->capture_last
#define Fcurrent_recurse   F->current_recurse
#define Fecode             F->ecode
#define Feptr              F->eptr
#define Fgroup_frame_type  F->group_frame_type
#define Flast_group_offset F->last_group_offset
#define Flength            F->length
#define Fmark              F->mark
#define Frdepth            F->rdepth
#define Fstart_match       F->start_match
#define Foffset_top        F->offset_top
#define Foccu              F->occu
#define Fop                F->op
#define Fovector           F->ovector
#define Freturn_id         F->return_id


#ifdef DEBUG_FRAMES_DISPLAY
/*************************************************
*      Display current frames and contents       *
*************************************************/
// 显示当前帧和内容
/* This debugging function displays the current set of frames and their
contents. It is not called automatically from anywhere, the intention being
that calls can be inserted where necessary when debugging frame-related
problems.

Arguments:
  f           the file to write to
  F           the current top frame
  P           a previous frame of interest
  frame_size  the frame size
  mb          points to the match block
  match_data  points to the match data block
  s           identification text

Returns:    nothing
*/
static void
display_frames(FILE *f, heapframe *F, heapframe *P, PCRE2_SIZE frame_size,
  match_block *mb, pcre2_match_data *match_data, const char *s, ...)
{
    uint32_t i;
    heapframe *Q;
    va_list ap;
    va_start(ap, s);

    fprintf(f, "FRAMES ");
    vfprintf(f, s, ap);
    va_end(ap);

    if (P != NULL) fprintf(f, " P=%lu",
      ((char *)P - (char *)(match_data->heapframes))/frame_size);
    fprintf(f, "\n");

    for (i = 0, Q = match_data->heapframes;
         Q <= F;
         i++, Q = (heapframe *)((char *)Q + frame_size))
      {
      fprintf(f, "Frame %d type=%x subj=%lu code=%d back=%lu id=%d",
        i, Q->group_frame_type, Q->eptr - mb->start_subject, *(Q->ecode),
        Q->back_frame, Q->return_id);

      if (Q->last_group_offset == PCRE2_UNSET)
        fprintf(f, " lgoffset=unset\n");
      else
        fprintf(f, " lgoffset=%lu\n",  Q->last_group_offset/frame_size);
      }
}
#endif

/*************************************************
*                Process a callout               *
*************************************************/

/* This function is called for all callouts, whether "standalone" or at the
start of a conditional group. Feptr will be pointing to either OP_CALLOUT or
OP_CALLOUT_STR. A callout block is allocated in pcre2_match() and initialized
with fixed values.

Arguments:
  F          points to the current backtracking frame
  mb         points to the match block
  lengthptr  where to return the length of the callout item
*/
# 执行调用函数，传入堆栈帧、匹配块和长度指针
static int
do_callout(heapframe *F, match_block *mb, PCRE2_SIZE *lengthptr)
{
int rc;
PCRE2_SIZE save0, save1;
PCRE2_SIZE *callout_ovector;
pcre2_callout_block *cb;

# 设置长度指针的值，根据条件选择不同的值
*lengthptr = (*Fecode == OP_CALLOUT)?
  PRIV(OP_lengths)[OP_CALLOUT] : GET(Fecode, 1 + 2*LINK_SIZE);

# 如果没有提供调用函数，则返回0
if (mb->callout == NULL) return 0;

# 为了向后兼容性，传递捕获顶部和偏移向量给调用函数
# 并确保前两个插槽未设置
callout_ovector = (PCRE2_SIZE *)(Fovector) - 2;

# 设置调用块的各个字段的值
cb = mb->cb;
cb->capture_top      = (uint32_t)Foffset_top/2 + 1;
cb->capture_last     = Fcapture_last;
cb->offset_vector    = callout_ovector;
cb->mark             = mb->nomatch_mark;
cb->current_position = (PCRE2_SIZE)(Feptr - mb->start_subject);
cb->pattern_position = GET(Fecode, 1);
cb->next_item_length = GET(Fecode, 1 + LINK_SIZE);

# 如果是数字调用，则设置调用块的相关字段
if (*Fecode == OP_CALLOUT)
  {
  cb->callout_number = Fecode[1 + 2*LINK_SIZE];
  cb->callout_string_offset = 0;
  cb->callout_string = NULL;
  cb->callout_string_length = 0;
  }
else  /* String callout */
  {
  // 如果不是数字调用，设置回调号为0
  cb->callout_number = 0;
  // 获取回调字符串的偏移量
  cb->callout_string_offset = GET(Fecode, 1 + 3*LINK_SIZE);
  // 获取回调字符串的地址
  cb->callout_string = Fecode + (1 + 4*LINK_SIZE) + 1;
  // 获取回调字符串的长度
  cb->callout_string_length =
    *lengthptr - (1 + 4*LINK_SIZE) - 2;
  }

// 保存回调向量的值
save0 = callout_ovector[0];
save1 = callout_ovector[1];
// 设置回调向量的值为未设置状态
callout_ovector[0] = callout_ovector[1] = PCRE2_UNSET;
// 调用回调函数
rc = mb->callout(cb, mb->callout_data);
// 恢复回调向量的值
callout_ovector[0] = save0;
callout_ovector[1] = save1;
// 清除回调标志
cb->callout_flags = 0;
// 返回回调结果
return rc;
}



/*************************************************
*          Match a back-reference                *
*************************************************/

/* This function is called only when it is known that the offset lies within
the offsets that have so far been used in the match. Note that in caseless
UTF-8 mode, the number of subject bytes matched may be different to the number
of reference bytes. (In theory this could also happen in UTF-16 mode, but it
seems unlikely.)

Arguments:
  offset      index into the offset vector
  caseless    TRUE if caseless
  F           the current backtracking frame pointer
  mb          points to match block
  lengthptr   pointer for returning the length matched

Returns:      = 0 sucessful match; number of code units matched is set
              < 0 no match
              > 0 partial match
*/

static int
match_ref(PCRE2_SIZE offset, BOOL caseless, heapframe *F, match_block *mb,
  PCRE2_SIZE *lengthptr)
{
// 定义指针变量
PCRE2_SPTR p;
// 定义长度变量
PCRE2_SIZE length;
// 定义指针变量
PCRE2_SPTR eptr;
// 定义指针变量
PCRE2_SPTR eptr_start;

/* Deal with an unset group. The default is no match, but there is an option to
match an empty string. */

// 处理未设置的组
if (offset >= Foffset_top || Fovector[offset] == PCRE2_UNSET)
  {
  // 如果设置了匹配未设置的回溯引用选项，返回匹配成功并设置长度为0
  if ((mb->poptions & PCRE2_MATCH_UNSET_BACKREF) != 0)
    {
    *lengthptr = 0;
    return 0;      /* Match */
    }
  else return -1;  /* No match */
  }

/* Separate the caseless and UTF cases for speed. */

// 分别处理大小写不敏感和UTF模式的情况
eptr = eptr_start = Feptr;
p = mb->start_subject + Fovector[offset];
# 计算长度，Fovector[offset+1]为结束位置，Fovector[offset]为起始位置
length = Fovector[offset+1] - Fovector[offset];

# 如果需要忽略大小写
if (caseless)
  {
#if defined SUPPORT_UNICODE
  # 检查是否启用了 UTF 或 UCP 模式
  BOOL utf = (mb->poptions & PCRE2_UTF) != 0;

  # 如果启用了 UTF 或 UCP 模式
  if (utf || (mb->poptions & PCRE2_UCP) != 0)
    {
    # 设置指向结束位置的指针
    PCRE2_SPTR endptr = p + length;

    /* 匹配直到引用的末尾。注意：匹配的代码单元数量可能不同，因为在 UTF-8 中有一些字符
    其大写和小写代码的字节数不同。例如，U+023A（UTF-8中的2个字节）是U+2C65（UTF-8中的3个字节）的大写版本；
    前者的3个序列使用6个字节，后者的2个序列也使用6个字节。因此，重要的是检查引用的长度，而不是主题的长度（之前的代码做错了）。
    UCP 没有使用 UTF 编码的 Unicode 属性。 */

    while (p < endptr)
      {
      uint32_t c, d;
      const ucd_record *ur;
      if (eptr >= mb->end_subject) return 1;   /* 部分匹配 */

      if (utf)
        {
        GETCHARINC(c, eptr);
        GETCHARINC(d, p);
        }
      else
        {
        c = *eptr++;
        d = *p++;
        }

      ur = GET_UCD(d);
      if (c != d && c != (uint32_t)((int)d + ur->other_case))
        {
        const uint32_t *pp = PRIV(ucd_caseless_sets) + ur->caseset;
        for (;;)
          {
          if (c < *pp) return -1;  /* 无匹配 */
          if (c == *pp++) break;
          }
        }
      }
    }
  else
#endif

  /* 不在 UTF 或 UCP 模式下 */
    {
    for (; length > 0; length--)
      {
      uint32_t cc, cp;
      if (eptr >= mb->end_subject) return 1;   /* 部分匹配 */
      cc = UCHAR21TEST(eptr);
      cp = UCHAR21TEST(p);
      if (TABLE_GET(cp, mb->lcc, cp) != TABLE_GET(cc, mb->lcc, cc))
        return -1;  /* 无匹配 */
      p++;
      eptr++;
      }
    }
  }

# 在区分大小写的情况下，我们可以直接比较代码单元，无论我们是否
else
  {
  // 如果不是 UTF 或 UCP 模式，进行部分匹配
  if (mb->partial != 0)
    {
    // 遍历每个单元，进行部分匹配
    for (; length > 0; length--)
      {
      // 如果已经匹配到字符串末尾，返回 1，表示部分匹配
      if (eptr >= mb->end_subject) return 1;   /* Partial match */
      // 如果当前单元不匹配，返回 -1，表示不匹配
      if (UCHAR21INCTEST(p) != UCHAR21INCTEST(eptr)) return -1;  /* No match */
      }
    }

  // 不是部分匹配
  else
    {
    // 如果剩余长度小于要匹配的长度，返回 1，表示部分匹配
    if ((PCRE2_SIZE)(mb->end_subject - eptr) < length) return 1; /* Partial */
    // 如果当前单元不匹配，返回 -1，表示不匹配
    if (memcmp(p, eptr, CU2BYTES(length)) != 0) return -1;  /* No match */
    // 移动指针到下一个位置
    eptr += length;
    }
  }

*lengthptr = eptr - eptr_start;
// 返回 0，表示匹配
return 0;  /* Match */
}
/* 
如果相同的 match_data 块用于多个匹配，则不会分配新的内存（除非需要扩展 frames 向量）。
*******************************************************************************
******************************************************************************/
*/

/*************************************************
*       用于 match() 函数的宏                      *
*************************************************/

/* 
这些宏封装了在代码中多次使用的部分匹配测试。第二个宏用于当我们已经知道我们已经超过了主题的末尾时。如果指针位于主题的末尾，并且（a）指针超过了最早检查的字符（即，已经匹配了一些内容，即使不是实际匹配的字符串的一部分），或者（b）模式包含一个回顾后发断言。这些是添加更多字符可能允许当前匹配继续的条件。

对于硬部分匹配，我们立即返回部分匹配。否则，继续意味着将寻找当前主题上的完全匹配。仅当找不到完全匹配时才返回部分匹配。
*/
#define CHECK_PARTIAL()\
  if (Feptr >= mb->end_subject) \
    { \
    SCHECK_PARTIAL(); \
    }

#define SCHECK_PARTIAL()\
  if (mb->partial != 0 && \
      (Feptr > mb->start_used_ptr || mb->allowemptypartial)) \
    { \
    mb->hitend = TRUE; \
    if (mb->partial > 1) return PCRE2_ERROR_PARTIAL; \
    }

/* 
这些宏用于实现回溯。它们通过本地帧向量记住回溯点，模拟了对 match() 函数的递归调用。
*/
#define RMATCH(ra,rb)\
  {\
  start_ecode = ra;\
  Freturn_id = rb;\
  goto MATCH_RECURSE;\
  L_##rb:;\
  }

#define RRETURN(ra)\
  {\
  rrc = ra;\
  goto RETURN_SWITCH;\
  }

/*************************************************
*         从当前位置开始匹配                        *
*************************************************/
/*************************************************/

/* 这个函数用于在主题中的单个起始点运行一次匹配尝试。

性能注意：可能会有诱惑将 mb 结构中常用的字段（例如 end_subject）提取为单独的变量以提高性能。在 SPARC 上使用 gcc 进行的测试证明这样做会使性能变差。

参数：
   start_eptr   主题中的起始字符
   start_ecode  编译代码中的起始位置
   top_bracket  模式中捕获括号的数量
   frame_size   回溯帧的大小
   match_data   指向 match_data 块的指针
   mb           指向“静态”变量块的指针

返回值：        MATCH_MATCH 如果匹配成功            )  这些值都是 >= 0
                MATCH_NOMATCH 如果匹配失败         )
                负的 MATCH_xxx 值表示 PRUNE、SKIP 等
                负的 PCRE2_ERROR_xxx 值表示被错误条件中止（例如被重复调用或深度限制所停止）
*/

static int
match(PCRE2_SPTR start_eptr, PCRE2_SPTR start_ecode, uint16_t top_bracket,
  PCRE2_SIZE frame_size, pcre2_match_data *match_data, match_block *mb)
{
/* 帧处理变量 */

heapframe *F;           /* 当前帧指针 */
heapframe *N = NULL;    /* 临时帧指针 */
heapframe *P = NULL;

heapframe *frames_top;  /* 帧向量的末尾 */
heapframe *assert_accept_frame = NULL;  /* 用于传递带有捕获的帧 */
PCRE2_SIZE heapframes_size;   /* 帧向量的可用大小 */
PCRE2_SIZE frame_copy_size;   /* 创建新帧时要复制的量 */

/* 不需要在对 RRMATCH() 的调用之间保留的本地变量 */

PCRE2_SPTR bracode;     /* 组的起始指针 */
PCRE2_SIZE offset;      /* 用于组偏移 */
PCRE2_SIZE length;      /* 用于各种长度计算 */

int rrc;                /* 函数返回值和回溯“递归” */
#ifdef SUPPORT_UNICODE
int proptype;           /* Type of character property */
#endif

uint32_t i;             /* Used for local loops */
uint32_t fc;            /* Character values */
uint32_t number;        /* Used for group and other numbers */
uint32_t reptype = 0;   /* Type of repetition (0 to avoid compiler warning) */
uint32_t group_frame_type;  /* Specifies type for new group frames */

BOOL condition;         /* Used in conditional groups */
BOOL cur_is_word;       /* Used in "word" tests */
BOOL prev_is_word;      /* Used in "word" tests */

/* UTF and UCP flags */

#ifdef SUPPORT_UNICODE
BOOL utf = (mb->poptions & PCRE2_UTF) != 0;  /* Check if UTF mode is enabled */
BOOL ucp = (mb->poptions & PCRE2_UCP) != 0;  /* Check if UCP mode is enabled */
#else
BOOL utf = FALSE;  /* Required for convenience even when no Unicode support */
#endif

/* This is the length of the last part of a backtracking frame that must be
copied when a new frame is created. */

frame_copy_size = frame_size - offsetof(heapframe, eptr);  /* Calculate the size of the last part of a backtracking frame */

/* Set up the first frame and the end of the frames vector. We set the local
heapframes_size to the usuable amount of the vector, that is, a whole number of
frames. */

F = match_data->heapframes;  /* Set F to point to the heapframes */
heapframes_size = (match_data->heapframes_size / frame_size) * frame_size;  /* Calculate the usable amount of the frames vector */
frames_top = (heapframe *)((char *)F + heapframes_size);  /* Set frames_top to point to the end of the frames vector */

Frdepth = 0;                        /* Initialize "Recursion" depth */
Fcapture_last = 0;                  /* Initialize Number of most recent capture */
Fcurrent_recurse = RECURSE_UNSET;   /* Initialize pattern recursing status */
Fstart_match = Feptr = start_eptr;  /* Initialize current data pointer and start match */
Fmark = NULL;                       /* Initialize most recent mark */
Foffset_top = 0;                    /* Initialize end of captures within the frame */
Flast_group_offset = PCRE2_UNSET;   /* Initialize saved frame of most recent group */
group_frame_type = 0;               /* Initialize start of group frame type */
goto NEW_FRAME;                     /* Start processing with this frame */

/* Come back here when we want to create a new frame for remembering a
/* 回溯点 */

MATCH_RECURSE:

/* 设置一个新的回溯帧。如果向量已满，则获取一个新的向量，将大小加倍，但受堆限制（以 KiB 为单位） */

N = (heapframe *)((char *)F + frame_size);
if (N >= frames_top)
  {
  heapframe *new;
  PCRE2_SIZE newsize = match_data->heapframes_size * 2;

  if (newsize > mb->heap_limit)
    {
    PCRE2_SIZE maxsize = (mb->heap_limit/frame_size) * frame_size;
    if (match_data->heapframes_size >= maxsize) return PCRE2_ERROR_HEAPLIMIT;
    newsize = maxsize;
    }

  new = match_data->memctl.malloc(newsize, match_data->memctl.memory_data);
  if (new == NULL) return PCRE2_ERROR_NOMEMORY;
  memcpy(new, match_data->heapframes, heapframes_size);

  F = (heapframe *)((char *)new + ((char *)F - (char *)match_data->heapframes));
  N = (heapframe *)((char *)F + frame_size);

  match_data->memctl.free(match_data->heapframes, match_data->memctl.memory_data);
  match_data->heapframes = new;
  match_data->heapframes_size = newsize;

  heapframes_size = (newsize / frame_size) * frame_size;
  frames_top = (heapframe *)((char *)new + heapframes_size);
  }

#ifdef DEBUG_SHOW_RMATCH
fprintf(stderr, "++ RMATCH %2d frame=%d", Freturn_id, Frdepth + 1);
if (group_frame_type != 0)
  {
  fprintf(stderr, " type=%x ", group_frame_type);
  switch (GF_IDMASK(group_frame_type))
    {
    case GF_CAPTURE:
    fprintf(stderr, "capture=%d", GF_DATAMASK(group_frame_type));
    break;

    case GF_NOCAPTURE:
    fprintf(stderr, "nocapture op=%d", GF_DATAMASK(group_frame_type));
    break;

    case GF_CONDASSERT:
    fprintf(stderr, "condassert op=%d", GF_DATAMASK(group_frame_type));
    break;

    case GF_RECURSE:
    fprintf(stderr, "recurse=%d", GF_DATAMASK(group_frame_type));
    break;

    default:
    fprintf(stderr, "*** unknown ***");
    break;
    }
  }
fprintf(stderr, "\n");
#endif

/* 复制必须复制到新帧中的字段，增加 */
/* 递归深度（即新帧的索引）并使新帧成为当前帧。 */

// 将当前帧的 eptr 数据拷贝到新帧的 eptr 数据
memcpy((char *)N + offsetof(heapframe, eptr),
       (char *)F + offsetof(heapframe, eptr),
       frame_copy_size);

// 设置新帧的递归深度为当前帧的递归深度加一
N->rdepth = Frdepth + 1;
// 将当前帧指针指向新帧
F = N;

/* 继续使用新帧进行处理。 */

NEW_FRAME:
// 设置 Fgroup_frame_type 为 group_frame_type
Fgroup_frame_type = group_frame_type;
// 设置 Fecode 为 start_ecode，即起始代码指针
Fecode = start_ecode;      /* Starting code pointer */
// 设置 Fback_frame 为 frame_size，即默认回退一帧
Fback_frame = frame_size;  /* Default is go back one frame */

/* 如果这是一种特殊类型的组帧，记住其偏移量以便在组结束时快速访问。如果这是一个递归，设置一个新的当前递归值。 */

if (group_frame_type != 0)
  {
  // 记录当前组帧的偏移量
  Flast_group_offset = (char *)F - (char *)match_data->heapframes;
  // 如果 group_frame_type 是递归类型，设置 Fcurrent_recurse 为 group_frame_type 的数据部分
  if (GF_IDMASK(group_frame_type) == GF_RECURSE)
    Fcurrent_recurse = GF_DATAMASK(group_frame_type);
  // 重置 group_frame_type
  group_frame_type = 0;
  }


/* ========================================================================= */
/* 这是主处理循环。首先检查我们是否记录了太多的回溯（搜索树太大），或者是否超过了递归深度限制（使用了太多的回溯帧）。如果没有，处理操作码。 */

// 如果匹配调用计数大于等于匹配限制，返回匹配限制错误
if (mb->match_call_count++ >= mb->match_limit) return PCRE2_ERROR_MATCHLIMIT;
// 如果当前递归深度大于等于匹配深度限制，返回深度限制错误
if (Frdepth >= mb->match_limit_depth) return PCRE2_ERROR_DEPTHLIMIT;

// 进入无限循环
for (;;)
  {
#ifdef DEBUG_SHOW_OPS
fprintf(stderr, "++ op=%d\n", *Fecode);
#endif

  // 设置 Fop 为 Fecode 指向的值
  Fop = (uint8_t)(*Fecode);  /* Cast needed for 16-bit and 32-bit modes */
  // 根据 Fop 的值进行不同的处理
  switch(Fop)
    {
    /* ===================================================================== */
    /* 在 OP_ACCEPT 之前可能有任意数量的 OP_CLOSE 操作码，用于关闭当前打开的捕获括号。与到达组的结尾不同，我们知道起始帧在链接帧的顶部，但在这种情况下，我们必须搜索相关帧以防其他类型的组使用了链接帧。多个 OP_CLOSE 操作码总是
    # 递归中最内层的情况首先出现，与链式顺序匹配。在递归中可以忽略这一点，因为捕获不会传递出递归。
    case OP_CLOSE:
    if (Fcurrent_recurse == RECURSE_UNSET)
      {
      number = GET2(Fecode, 1);
      offset = Flast_group_offset;
      for(;;)
        {
        if (offset == PCRE2_UNSET) return PCRE2_ERROR_INTERNAL;
        N = (heapframe *)((char *)match_data->heapframes + offset);
        P = (heapframe *)((char *)N - frame_size);
        if (N->group_frame_type == (GF_CAPTURE | number)) break;
        offset = P->last_group_offset;
        }
      offset = (number << 1) - 2;
      Fcapture_last = number;
      Fovector[offset] = P->eptr - mb->start_subject;
      Fovector[offset+1] = Feptr - mb->start_subject;
      if (offset >= Foffset_top) Foffset_top = offset + 2;
      }
    Fecode += PRIV(OP_lengths)[*Fecode];
    break;

    /* ===================================================================== */
    /* 模式的真实或强制结束，断言或递归。在断言ACCEPT中，更新最后使用的指针，并记住当前帧，以便从中获取捕获和标记。 */
    case OP_ASSERT_ACCEPT:
    if (Feptr > mb->last_used_ptr) mb->last_used_ptr = Feptr;
    assert_accept_frame = F;
    RRETURN(MATCH_ACCEPT);

    /* 如果递归，我们必须找到最近的递归。 */
    case OP_ACCEPT:
    case OP_END:
    /* 处理递归的结束。 */
    # 如果当前递归不是未设置状态
    if (Fcurrent_recurse != RECURSE_UNSET)
      {
      # 设置偏移量为上一次组的偏移量
      offset = Flast_group_offset;
      # 无限循环
      for(;;)
        {
        # 如果偏移量未设置，则返回内部错误
        if (offset == PCRE2_UNSET) return PCRE2_ERROR_INTERNAL;
        # N 是递归的帧；前一个帧在 OP_RECURSE 位置。回到那里，复制当前主题位置和标记，以及 start_match 位置（\K 可能已经改变了它），然后继续前进到 OP_RECURSE 之后。
        N = (heapframe *)((char *)match_data->heapframes + offset);
        P = (heapframe *)((char *)N - frame_size);
        # 如果 N 的组帧类型是 GF_RECURSE，则跳出循环
        if (GF_IDMASK(N->group_frame_type) == GF_RECURSE) break;
        # 设置偏移量为上一个组的偏移量
        offset = P->last_group_offset;
        }

      # 现在 N 是递归的帧；前一个帧在 OP_RECURSE 位置。回到那里，复制当前主题位置和标记，以及 start_match 位置（\K 可能已经改变了它），然后继续前进到 OP_RECURSE 之后。
      P->eptr = Feptr;
      P->mark = Fmark;
      P->start_match = Fstart_match;
      F = P;
      Fecode += 1 + LINK_SIZE;
      continue;
      }

    # 如果不是递归。如果设置了 PCRE2_NOTEMPTY 或者设置了 PCRE2_NOTEMPTY_ATSTART 并且我们在主题的开头匹配，则对空字符串匹配失败。在这两种情况下，回溯将尝试其他替代方案（如果有的话）。
    if (Feptr == Fstart_match &&
         ((mb->moptions & PCRE2_NOTEMPTY) != 0 ||
           ((mb->moptions & PCRE2_NOTEMPTY_ATSTART) != 0 &&
             Fstart_match == mb->start_subject + mb->start_offset)))
      RRETURN(MATCH_NOMATCH);

    # 如果设置了 PCRE2_ENDANCHORED 并且匹配的结束不是主题的结束。在 (*ACCEPT) 之后，整个匹配失败（在这个位置），但在达到模式的结束时回溯。
    if (Feptr < mb->end_subject &&
        ((mb->moptions | mb->poptions) & PCRE2_ENDANCHORED) != 0)
      {
      if (Fop == OP_END) RRETURN(MATCH_NOMATCH);
      return MATCH_NOMATCH;
      }

    # 我们成功匹配整个模式。记录结果，然后直接从函数返回。如果偏移向量中有空间，设置任何跟随最高编号捕获字符串的对
    /* 设置匹配结束指针为当前位置 */
    mb->end_match_ptr = Feptr;           
    /* 记录已经取得的提取组数量 */
    mb->end_offset_top = Foffset_top;    
    /* 记录最后一次成功匹配的标记 */
    mb->mark = Fmark;                    
    /* 如果当前位置大于上次使用的指针位置，则更新上次使用的指针位置 */
    if (Feptr > mb->last_used_ptr) mb->last_used_ptr = Feptr;

    /* 设置匹配数据的起始位置和结束位置 */
    match_data->ovector[0] = Fstart_match - mb->start_subject;
    match_data->ovector[1] = Feptr - mb->start_subject;

    /* 设置 i 为外部和框架 ovector 大小的较小值 */
    i = 2 * ((top_bracket + 1 > match_data->oveccount)?
      match_data->oveccount : top_bracket + 1);
    /* 复制 Fovector 中的数据到 match_data->ovector 中 */
    memcpy(match_data->ovector + 2, Fovector, (i - 2) * sizeof(PCRE2_SIZE));
    /* 将外部和框架 ovector 中多余的部分设置为 PCRE2_UNSET */
    while (--i >= Foffset_top + 2) match_data->ovector[i] = PCRE2_UNSET;
    /* 返回匹配成功的标志 */
    return MATCH_MATCH;  /* 注意：不是返回 RRETURN */


    /*===================================================================== */
    /* 匹配除换行符之外的任意单个字符类型；需要注意 CRLF 换行符和部分匹配。 */

    case OP_ANY:
    /* 如果当前位置是换行符，则返回不匹配 */
    if (IS_NEWLINE(Feptr)) RRETURN(MATCH_NOMATCH);
    /* 如果启用了部分匹配，并且当前位置是结束位置的前一个字符，并且换行符类型是固定长度的CRLF，并且当前字符是CRLF的第一个字符 */
    if (mb->partial != 0 &&
        Feptr == mb->end_subject - 1 &&
        NLBLOCK->nltype == NLTYPE_FIXED &&
        NLBLOCK->nllen == 2 &&
        UCHAR21TEST(Feptr) == NLBLOCK->nl[0])
      {
      /* 设置匹配到了结束位置 */
      mb->hitend = TRUE;
      /* 如果部分匹配的标志大于1，则返回部分匹配错误 */
      if (mb->partial > 1) return PCRE2_ERROR_PARTIAL;
      }
    /* 继续执行下面的操作 */

    /* 匹配任意单个字符 */
    case OP_ALLANY:
    /* 如果当前位置超出了结束位置，则进行部分匹配检查，并返回不匹配 */
    if (Feptr >= mb->end_subject)  
      {                            
      SCHECK_PARTIAL();
      RRETURN(MATCH_NOMATCH);
      }
    /* 移动当前位置到下一个字符 */
    Feptr++;
#ifdef SUPPORT_UNICODE
    # 如果支持 Unicode，则在 UTF 模式下移动 Feptr 指针
    if (utf) ACROSSCHAR(Feptr < mb->end_subject, Feptr, Feptr++);
#endif
    # 增加 Fecode 指针的值
    Fecode++;
    # 跳出当前的 switch 语句块
    break;


    /* ===================================================================== */
    /* 匹配单个代码单元，即使在 UTF 模式下也是如此。这个操作码实际上匹配任何代码单元，甚至是换行符。(当然，它实际上应该被称为 ANYCODEUNIT，字节名称来自于 16 位之前的时代。) */

    case OP_ANYBYTE:
    # 如果 Feptr 指针超出了 mb->end_subject，则执行以下操作
    if (Feptr >= mb->end_subject)   /* DO NOT merge the Feptr++ here; it must */
      {                             /* not be updated before SCHECK_PARTIAL. */
      # 检查是否部分匹配
      SCHECK_PARTIAL();
      # 返回不匹配
      RRETURN(MATCH_NOMATCH);
      }
    # 移动 Feptr 指针
    Feptr++;
    # 增加 Fecode 指针的值
    Fecode++;
    # 跳出当前的 switch 语句块
    break;


    /* ===================================================================== */
    /* 匹配单个字符，区分大小写 */

    case OP_CHAR:
#ifdef SUPPORT_UNICODE
    # 如果支持 Unicode，则执行以下操作
    if (utf)
      {
      # 设置 Flength 为 1
      Flength = 1;
      # 增加 Fecode 指针的值
      Fecode++;
      # 获取字符长度
      GETCHARLEN(fc, Fecode, Flength);
      # 如果字符长度大于 Feptr 到 mb->end_subject 的长度
      if (Flength > (PCRE2_SIZE)(mb->end_subject - Feptr))
        {
        # 检查是否部分匹配
        CHECK_PARTIAL();             /* Not SCHECK_PARTIAL() */
        # 返回不匹配
        RRETURN(MATCH_NOMATCH);
        }
      # 遍历字符长度
      for (; Flength > 0; Flength--)
        {
        # 如果 Fecode 指针指向的字符不等于 Feptr 指针指向的字符，则返回不匹配
        if (*Fecode++ != UCHAR21INC(Feptr)) RRETURN(MATCH_NOMATCH);
        }
      }
    else
#endif

    /* 非 UTF 模式 */
      {
      # 如果 mb->end_subject 减去 Feptr 的长度小于 1
      if (mb->end_subject - Feptr < 1)
        {
        # 检查是否部分匹配
        SCHECK_PARTIAL();            /* This one can use SCHECK_PARTIAL() */
        # 返回不匹配
        RRETURN(MATCH_NOMATCH);
        }
      # 如果 Fecode 指针指向的下一个字符不等于 Feptr 指针指向的字符，则返回不匹配
      if (Fecode[1] != *Feptr++) RRETURN(MATCH_NOMATCH);
      # 增加 Fecode 指针的值
      Fecode += 2;
      }
    # 跳出当前的 switch 语句块
    break;


    /* ===================================================================== */
    /* 匹配单个字符，不区分大小写。如果我们在主题的末尾，立即放弃。我们只有在模式字符最多有一个其他大小写时才会到达这里。具有两个以上大小写的字符编码为具有伪属性 PT_CLIST 的 OP_PROP。 */

    case OP_CHARI:
    # 如果指针 Feptr 大于等于 mb->end_subject，则表示已经匹配到了字符串结尾
    if (Feptr >= mb->end_subject)
      {
      # 检查是否需要部分匹配
      SCHECK_PARTIAL();
      # 返回不匹配的结果
      RRETURN(MATCH_NOMATCH);
      }
#ifdef SUPPORT_UNICODE
    # 如果支持 Unicode
    if (utf)
      {
      # 如果是 UTF 编码
      Flength = 1;
      Fecode++;
      GETCHARLEN(fc, Fecode, Flength);

      /* 如果模式字符的值 < 128，我们知道它的其他情况（如果有的话）也 < 128
      （因此在所有代码单元宽度上只有一个代码单元长），所以我们可以使用快速查找表。
      我们在上面检查了主题中至少还有一个字符。 */

      if (fc < 128)
        {
        uint32_t cc = UCHAR21(Feptr);
        if (mb->lcc[fc] != TABLE_GET(cc, mb->lcc, cc)) RRETURN(MATCH_NOMATCH);
        Fecode++;
        Feptr++;
        }

      /* 否则，我们必须拾取主题字符并使用 Unicode 属性支持来测试其其他情况。
      请注意，我们不能使用 "Flength" 的值来检查是否还有足够的字节，因为字符的其他情况可能有更多或更少的代码单元。 */

      else
        {
        uint32_t dc;
        GETCHARINC(dc, Feptr);
        Fecode += Flength;
        if (dc != fc && dc != UCD_OTHERCASE(fc)) RRETURN(MATCH_NOMATCH);
        }
      }

    /* 如果设置了 UCP 但没有设置 UTF，我们必须像上面一样做，但每个代码单元一个字符。 */

    else if (ucp)
      {
      uint32_t cc = UCHAR21(Feptr);
      fc = Fecode[1];
      if (fc < 128)
        {
        if (mb->lcc[fc] != TABLE_GET(cc, mb->lcc, cc)) RRETURN(MATCH_NOMATCH);
        }
      else
        {
        if (cc != fc && cc != UCD_OTHERCASE(fc)) RRETURN(MATCH_NOMATCH);
        }
      Feptr++;
      Fecode += 2;
      }

    else
#endif   /* SUPPORT_UNICODE */

    /* 如果不是 UTF 或 UCP 模式；对于字符 < 256 使用表进行匹配。 */
      {
      if (TABLE_GET(Fecode[1], mb->lcc, Fecode[1])
          != TABLE_GET(*Feptr, mb->lcc, *Feptr)) RRETURN(MATCH_NOMATCH);
      Feptr++;
      Fecode += 2;
      }
    break;


    /* ===================================================================== */
    /* 匹配不是单个字符。 */

    case OP_NOT:
    # 如果操作码是OP_NOTI
    case OP_NOTI:
    # 如果Feptr大于等于主题的结束位置
    if (Feptr >= mb->end_subject)
      {
      # 检查是否部分匹配
      SCHECK_PARTIAL();
      # 返回不匹配
      RRETURN(MATCH_NOMATCH);
      }
#ifdef SUPPORT_UNICODE
    # 如果支持 Unicode
    if (utf)
      {
      # 增加指针 Fecode 的值
      Fecode++;
      # 读取字符并增加指针 Fecode 的值
      GETCHARINC(ch, Fecode);
      # 读取字符并增加指针 Feptr 的值
      GETCHARINC(fc, Feptr);
      # 如果 ch 等于 fc，则返回不匹配
      if (ch == fc)
        {
        RRETURN(MATCH_NOMATCH);  /* Caseful match */
        }
      # 如果 Fop 等于 OP_NOTI，则表示不区分大小写
      else if (Fop == OP_NOTI)   /* If caseless */
        {
        # 如果 ch 大于 127，则获取其其他大小写形式
        if (ch > 127)
          ch = UCD_OTHERCASE(ch);
        else
          # 否则获取其对应的大小写形式
          ch = (mb->fcc)[ch];
        # 如果 ch 等于 fc，则返回不匹配
        if (ch == fc) RRETURN(MATCH_NOMATCH);
        }
      }

    /* UCP without UTF is as above, but with one character per code unit. */

    else if (ucp)
      {
      # 读取字符并增加指针 Feptr 的值
      fc = UCHAR21INC(Feptr);
      # 获取 Fecode[1] 的值
      ch = Fecode[1];
      # 增加指针 Fecode 的值
      Fecode += 2;

      # 如果 ch 等于 fc，则返回不匹配
      if (ch == fc)
        {
        RRETURN(MATCH_NOMATCH);  /* Caseful match */
        }
      # 如果 Fop 等于 OP_NOTI，则表示不区分大小写
      else if (Fop == OP_NOTI)   /* If caseless */
        {
        # 如果 ch 大于 127，则获取其其他大小写形式
        if (ch > 127)
          ch = UCD_OTHERCASE(ch);
        else
          # 否则获取其对应的大小写形式
          ch = (mb->fcc)[ch];
        # 如果 ch 等于 fc，则返回不匹配
        if (ch == fc) RRETURN(MATCH_NOMATCH);
        }
      }

    else
#endif  /* SUPPORT_UNICODE */

    /* 如果既不支持 UTF 也不支持 UCP */

      {
      # 获取 Fecode[1] 的值
      uint32_t ch = Fecode[1];
      # 读取字符并增加指针 Feptr 的值
      fc = UCHAR21INC(Feptr);
      # 如果 ch 等于 fc 或者（Fop 等于 OP_NOTI 并且 mb->fcc 中 ch 对应的值等于 fc），则返回不匹配
      if (ch == fc || (Fop == OP_NOTI && TABLE_GET(ch, mb->fcc, ch) == fc))
        RRETURN(MATCH_NOMATCH);
      # 增加指针 Fecode 的值
      Fecode += 2;
      }
    break;


    /* ===================================================================== */
    /* Match a single character repeatedly. */

#define Loclength    F->temp_size
#define Lstart_eptr  F->temp_sptr[0]
#define Lcharptr     F->temp_sptr[1]
#define Lmin         F->temp_32[0]
#define Lmax         F->temp_32[1]
#define Lc           F->temp_32[2]
#define Loc          F->temp_32[3]

    # 匹配一个字符并重复
    case OP_EXACT:
    case OP_EXACTI:
    # 获取两个字节的值，赋给 Lmin 和 Lmax
    Lmin = Lmax = GET2(Fecode, 1);
    # 增加指针 Fecode 的值
    Fecode += 1 + IMM2_SIZE;
    # 跳转到 REPEATCHAR 标签处
    goto REPEATCHAR;

    case OP_POSUPTO:
    case OP_POSUPTOI:
    # 设置重复类型为 REPTYPE_POS
    reptype = REPTYPE_POS;
    # Lmin 为 0，Lmax 为获取的两个字节的值
    Lmin = 0;
    Lmax = GET2(Fecode, 1);
    # 增加指针 Fecode 的值
    Fecode += 1 + IMM2_SIZE;
    # 跳转到 REPEATCHAR 标签处
    goto REPEATCHAR;

    case OP_UPTO:
    case OP_UPTOI:
    # 设置重复类型为 REPTYPE_MAX
    reptype = REPTYPE_MAX;
    # 设置最小重复次数为0
    Lmin = 0;
    # 从Fecode中获取2个字节的数据，作为最大重复次数
    Lmax = GET2(Fecode, 1);
    # Fecode指针移动到下一个位置
    Fecode += 1 + IMM2_SIZE;
    # 跳转到REPEATCHAR标签处执行代码

    case OP_MINUPTO:
    case OP_MINUPTOI:
    # 设置重复类型为最小重复
    reptype = REPTYPE_MIN;
    # 设置最小重复次数为0
    Lmin = 0;
    # 从Fecode中获取2个字节的数据，作为最大重复次数
    Lmax = GET2(Fecode, 1);
    # Fecode指针移动到下一个位置
    Fecode += 1 + IMM2_SIZE;
    # 跳转到REPEATCHAR标签处执行代码

    case OP_POSSTAR:
    case OP_POSSTARI:
    # 设置重复类型为正向最小重复
    reptype = REPTYPE_POS;
    # 设置最小重复次数为0
    Lmin = 0;
    # 设置最大重复次数为无限大
    Lmax = UINT32_MAX;
    # Fecode指针移动到下一个位置
    Fecode++;
    # 跳转到REPEATCHAR标签处执行代码

    case OP_POSPLUS:
    case OP_POSPLUSI:
    # 设置重复类型为正向最小重复
    reptype = REPTYPE_POS;
    # 设置最小重复次数为1
    Lmin = 1;
    # 设置最大重复次数为无限大
    Lmax = UINT32_MAX;
    # Fecode指针移动到下一个位置
    Fecode++;
    # 跳转到REPEATCHAR标签处执行代码

    case OP_POSQUERY:
    case OP_POSQUERYI:
    # 设置重复类型为正向最小重复
    reptype = REPTYPE_POS;
    # 设置最小重复次数为0
    Lmin = 0;
    # 设置最大重复次数为1
    Lmax = 1;
    # Fecode指针移动到下一个位置
    Fecode++;
    # 跳转到REPEATCHAR标签处执行代码

    case OP_STAR:
    case OP_STARI:
    case OP_MINSTAR:
    case OP_MINSTARI:
    case OP_PLUS:
    case OP_PLUSI:
    case OP_MINPLUS:
    case OP_MINPLUSI:
    case OP_QUERY:
    case OP_QUERYI:
    case OP_MINQUERY:
    case OP_MINQUERYI:
    # 从Fecode中读取一个字符，根据不同的操作码计算出重复次数的最小值、最大值和类型
    fc = *Fecode++ - ((Fop < OP_STARI)? OP_STAR : OP_STARI);
    Lmin = rep_min[fc];
    Lmax = rep_max[fc];
    reptype = rep_typ[fc];

    # 以下是所有重复单字符匹配的通用代码。首先检查最小字符数是否满足，如果最小值等于最大值，则匹配完成。
    # 否则，如果是最小化匹配，检查剩余的模式是否匹配；如果不匹配，逐个字符增加直到达到最大值。
    # 如果是最大化匹配，增加直到达到最大匹配字符数，直到Feptr超出最大运行的末尾。如果是占有性匹配，那么匹配完成（不回溯）。
    # 否则，在这个位置匹配；任何不匹配的情况都会立即返回。对于不匹配，回溯一个字符，除非我们匹配\R并且最后匹配的是\r\n，
    # 在这种情况下，回溯两个代码单元，直到达到第一个可选字符位置。
    # 不同的UTF/非UTF和区分大小写/不区分大小写的情况会分别处理，以提高速度。
    REPEATCHAR:
#ifdef SUPPORT_UNICODE
    else
#endif  /* SUPPORT_UNICODE */
    /* 如果支持 Unicode，则执行以下代码块，否则跳过 */

    /* 当不处于 UTF 模式时，加载单代码单元字符。然后按照上面的方式继续，如果设置了 UTF 或 UCP，则使用 Unicode 大小写。 */
    Lc = *Fecode++;
    /* 读取下一个字符，并将其赋值给 Lc */

    /* 无大小写区分比较 */
    if (Fop >= OP_STARI)
    {
#if PCRE2_CODE_UNIT_WIDTH == 8
#ifdef SUPPORT_UNICODE
        if (ucp && !utf && Lc > 127) Loc = UCD_OTHERCASE(Lc);
        else
#endif  /* SUPPORT_UNICODE */
        /* 如果支持 Unicode 属性，且不是 UTF 模式，并且 Lc 大于 127，则使用 UCD_OTHERCASE 获取其大小写对应字符 */
        /* 否则，Lc 在 UTF-8 模式下将小于 128 */
        Loc = mb->fcc[Lc];
#else /* 16-bit & 32-bit */
#ifdef SUPPORT_UNICODE
        if ((utf || ucp) && Lc > 127) Loc = UCD_OTHERCASE(Lc);
        else
#endif  /* SUPPORT_UNICODE */
        /* 如果是 16 位或 32 位，并且设置了 UTF 或 UCP，并且 Lc 大于 127，则使用 UCD_OTHERCASE 获取其大小写对应字符 */
        /* 否则，使用 TABLE_GET 获取其大小写对应字符 */
        Loc = TABLE_GET(Lc, mb->fcc, Lc);
#endif  /* PCRE2_CODE_UNIT_WIDTH == 8 */

      for (i = 1; i <= Lmin; i++)
        {
        uint32_t cc;                 /* Faster than PCRE2_UCHAR */
        if (Feptr >= mb->end_subject)
          {
          SCHECK_PARTIAL();          /* 检查是否需要进行部分匹配 */
          RRETURN(MATCH_NOMATCH);     /* 返回不匹配 */
          }
        cc = UCHAR21TEST(Feptr);      /* 获取当前字符的值 */
        if (Lc != cc && Loc != cc) RRETURN(MATCH_NOMATCH);  /* 如果当前字符不等于Lc并且不等于Loc，则返回不匹配 */
        Feptr++;                      /* 移动到下一个字符 */
        }
      if (Lmin == Lmax) continue;     /* 如果Lmin等于Lmax，则继续下一次循环 */

      if (reptype == REPTYPE_MIN)      /* 如果重复类型是最小匹配 */
        {
        for (;;)
          {
          uint32_t cc;               /* Faster than PCRE2_UCHAR */
          RMATCH(Fecode, RM25);      /* 调用RMATCH函数进行匹配 */
          if (rrc != MATCH_NOMATCH) RRETURN(rrc);  /* 如果匹配结果不是不匹配，则返回匹配结果 */
          if (Lmin++ >= Lmax) RRETURN(MATCH_NOMATCH);  /* 如果Lmin大于等于Lmax，则返回不匹配 */
          if (Feptr >= mb->end_subject)
            {
            SCHECK_PARTIAL();        /* 检查是否需要进行部分匹配 */
            RRETURN(MATCH_NOMATCH);   /* 返回不匹配 */
            }
          cc = UCHAR21TEST(Feptr);   /* 获取当前字符的值 */
          if (Lc != cc && Loc != cc) RRETURN(MATCH_NOMATCH);  /* 如果当前字符不等于Lc并且不等于Loc，则返回不匹配 */
          Feptr++;                   /* 移动到下一个字符 */
          }
        /* Control never gets here */
        }

      else  /* Maximize */           /* 否则，最大化匹配 */
        {
        Lstart_eptr = Feptr;         /* 记录起始位置的指针 */
        for (i = Lmin; i < Lmax; i++)
          {
          uint32_t cc;               /* Faster than PCRE2_UCHAR */
          if (Feptr >= mb->end_subject)
            {
            SCHECK_PARTIAL();        /* 检查是否需要进行部分匹配 */
            break;                   /* 跳出循环 */
            }
          cc = UCHAR21TEST(Feptr);   /* 获取当前字符的值 */
          if (Lc != cc && Loc != cc) break;  /* 如果当前字符不等于Lc并且不等于Loc，则跳出循环 */
          Feptr++;                   /* 移动到下一个字符 */
          }
        if (reptype != REPTYPE_POS) for (;;)
          {
          if (Feptr == Lstart_eptr) break;  /* 如果当前指针等于起始指针，则跳出循环 */
          RMATCH(Fecode, RM26);      /* 调用RMATCH函数进行匹配 */
          Feptr--;                   /* 指针回退 */
          if (rrc != MATCH_NOMATCH) RRETURN(rrc);  /* 如果匹配结果不是不匹配，则返回匹配结果 */
          }
        }
      }

    /* Caseful comparisons (includes all multi-byte characters) */
    else
      {
      // 如果不是贪婪匹配，对最小重复次数进行匹配
      for (i = 1; i <= Lmin; i++)
        {
        // 如果当前位置超出了主字符串的末尾，进行部分匹配检查
        if (Feptr >= mb->end_subject)
          {
          SCHECK_PARTIAL();
          RRETURN(MATCH_NOMATCH);
          }
        // 如果当前字符不匹配，返回不匹配
        if (Lc != UCHAR21INCTEST(Feptr)) RRETURN(MATCH_NOMATCH);
        }

      // 如果最小重复次数等于最大重复次数，继续下一轮匹配
      if (Lmin == Lmax) continue;

      // 如果是最小匹配，进行最小匹配的处理
      if (reptype == REPTYPE_MIN)
        {
        for (;;)
          {
          // 递归匹配下一个字符
          RMATCH(Fecode, RM27);
          // 如果匹配成功，返回结果
          if (rrc != MATCH_NOMATCH) RRETURN(rrc);
          // 如果达到最大重复次数，返回不匹配
          if (Lmin++ >= Lmax) RRETURN(MATCH_NOMATCH);
          // 如果当前位置超出了主字符串的末尾，进行部分匹配检查
          if (Feptr >= mb->end_subject)
            {
            SCHECK_PARTIAL();
            RRETURN(MATCH_NOMATCH);
            }
          // 如果当前字符不匹配，返回不匹配
          if (Lc != UCHAR21INCTEST(Feptr)) RRETURN(MATCH_NOMATCH);
          }
        /* Control never gets here */
        }
      else  /* Maximize */
        {
        // 记录当前位置
        Lstart_eptr = Feptr;
        // 对最大重复次数进行匹配
        for (i = Lmin; i < Lmax; i++)
          {
          // 如果当前位置超出了主字符串的末尾，进行部分匹配检查
          if (Feptr >= mb->end_subject)
            {
            SCHECK_PARTIAL();
            break;
            }
          // 如果当前字符不匹配，跳出循环
          if (Lc != UCHAR21TEST(Feptr)) break;
          // 移动到下一个字符
          Feptr++;
          }

        // 如果不是正向匹配，进行回溯匹配
        if (reptype != REPTYPE_POS) for (;;)
          {
          // 如果当前位置小于等于起始位置，跳出循环
          if (Feptr <= Lstart_eptr) break;
          // 递归匹配上一个字符
          RMATCH(Fecode, RM28);
          // 移动到上一个字符
          Feptr--;
          // 如果匹配成功，返回结果
          if (rrc != MATCH_NOMATCH) RRETURN(rrc);
          }
        }
      }
    break;
    # 取消定义之前的宏
    # 取消定义之前的宏
    # 取消定义之前的宏
    # 取消定义之前的宏
    # 取消定义之前的宏
    # 取消定义之前的宏
    # 取消定义之前的宏

    # 匹配一个重复出现的非单字节字符。这几乎是重复单个字符的代码，但我还没有找到一个很好的方法来将它们合并起来，而不需要对每个字符匹配的正负选项进行测试。也许这不会增加太多时间，但字符匹配*就是这一切的关键...
#define Lstart_eptr  F->temp_sptr[0]
#define Lmin         F->temp_32[0]
#define Lmax         F->temp_32[1]
#define Lc           F->temp_32[2]
#define Loc          F->temp_32[3]

    case OP_NOTEXACT:
    case OP_NOTEXACTI:
    # 设置最小和最大重复次数
    Lmin = Lmax = GET2(Fecode, 1)
    # 移动指针到下一个操作码
    Fecode += 1 + IMM2_SIZE
    # 跳转到 REPEATNOTCHAR 标签处
    goto REPEATNOTCHAR

    case OP_NOTUPTO:
    case OP_NOTUPTOI:
    # 设置最小和最大重复次数
    Lmin = 0
    Lmax = GET2(Fecode, 1)
    # 设置重复类型为最大
    reptype = REPTYPE_MAX
    # 移动指针到下一个操作码
    Fecode += 1 + IMM2_SIZE
    # 跳转到 REPEATNOTCHAR 标签处
    goto REPEATNOTCHAR

    case OP_NOTMINUPTO:
    case OP_NOTMINUPTOI:
    # 设置最小和最大重复次数
    Lmin = 0
    Lmax = GET2(Fecode, 1)
    # 设置重复类型为最小
    reptype = REPTYPE_MIN
    # 移动指针到下一个操作码
    Fecode += 1 + IMM2_SIZE
    # 跳转到 REPEATNOTCHAR 标签处
    goto REPEATNOTCHAR

    case OP_NOTPOSSTAR:
    case OP_NOTPOSSTARI:
    # 设置重复类型为正向
    reptype = REPTYPE_POS
    # 设置最小和最大重复次数
    Lmin = 0
    Lmax = UINT32_MAX
    # 移动指针到下一个操作码
    Fecode++
    # 跳转到 REPEATNOTCHAR 标签处
    goto REPEATNOTCHAR

    case OP_NOTPOSPLUS:
    case OP_NOTPOSPLUSI:
    # 设置重复类型为正向
    reptype = REPTYPE_POS
    # 设置最小和最大重复次数
    Lmin = 1
    Lmax = UINT32_MAX
    # 移动指针到下一个操作码
    Fecode++
    # 跳转到 REPEATNOTCHAR 标签处
    goto REPEATNOTCHAR

    case OP_NOTPOSQUERY:
    case OP_NOTPOSQUERYI:
    # 设置重复类型为正向
    reptype = REPTYPE_POS
    # 设置最小和最大重复次数
    Lmin = 0
    Lmax = 1
    # 移动指针到下一个操作码
    Fecode++
    # 跳转到 REPEATNOTCHAR 标签处
    goto REPEATNOTCHAR

    case OP_NOTPOSUPTO:
    case OP_NOTPOSUPTOI:
    # 设置重复类型为正向
    reptype = REPTYPE_POS
    # 设置最小和最大重复次数
    Lmin = 0
    Lmax = GET2(Fecode, 1)
    # 移动指针到下一个操作码
    Fecode += 1 + IMM2_SIZE
    # 跳转到 REPEATNOTCHAR 标签处
    goto REPEATNOTCHAR

    case OP_NOTSTAR:
    case OP_NOTSTARI:
    case OP_NOTMINSTAR:
    case OP_NOTMINSTARI:
    case OP_NOTPLUS:
    case OP_NOTPLUSI:
    # 其他操作，暂时省略
    // 如果操作码是OP_NOTMINPLUS、OP_NOTMINPLUSI、OP_NOTQUERY、OP_NOTQUERYI、OP_NOTMINQUERY、OP_NOTMINQUERYI中的一个，执行以下代码
    case OP_NOTMINPLUS:
    case OP_NOTMINPLUSI:
    case OP_NOTQUERY:
    case OP_NOTQUERYI:
    case OP_NOTMINQUERY:
    case OP_NOTMINQUERYI:
    // 从Fecode指针指向的位置读取一个字符，减去OP_NOTSTARI或OP_NOTSTAR，得到fc
    fc = *Fecode++ - ((Fop >= OP_NOTSTARI)? OP_NOTSTARI: OP_NOTSTAR);
    // 从rep_min数组中获取fc对应的最小重复次数
    Lmin = rep_min[fc];
    // 从rep_max数组中获取fc对应的最大重复次数
    Lmax = rep_max[fc];
    // 从rep_typ数组中获取fc对应的重复类型
    reptype = rep_typ[fc];

    /* Common code for all repeated single-character non-matches. */
    // 重复单字符非匹配的通用代码

    REPEATNOTCHAR:
    // 从Fecode指针指向的位置读取一个字符，存入Lc中，并检查是否到达字符串结尾
    GETCHARINCTEST(Lc, Fecode);

    /* The code is duplicated for the caseless and caseful cases, for speed,
    since matching characters is likely to be quite common. First, ensure the
    minimum number of matches are present. If Lmin = Lmax, we are done.
    Otherwise, if minimizing, keep trying the rest of the expression and
    advancing one matching character if failing, up to the maximum.
    Alternatively, if maximizing, find the maximum number of characters and
    work backwards. */
    // 为了提高速度，代码在大小写敏感和不敏感的情况下都有重复，因为匹配字符很可能是非常常见的。首先，确保存在最小数量的匹配。如果Lmin = Lmax，则完成。否则，如果最小化，继续尝试表达式的其余部分，并在失败时前进一个匹配字符，直到达到最大值。或者，如果最大化，找到最大数量的字符并向后工作。

    // 如果操作码大于等于OP_NOTSTARI，表示不区分大小写
    if (Fop >= OP_NOTSTARI)     /* Caseless */
      {
#ifdef SUPPORT_UNICODE
      # 如果支持 Unicode，并且当前字符是大于 127 的字符
      if ((utf || ucp) && Lc > 127)
        # 获取当前字符的其他大小写形式
        Loc = UCD_OTHERCASE(Lc);
      else
#endif /* SUPPORT_UNICODE */

      # 从表中获取当前字符的其他大小写形式
      Loc = TABLE_GET(Lc, mb->fcc, Lc);  /* Other case from table */

#ifdef SUPPORT_UNICODE
      # 如果是 UTF 模式
      if (utf)
        {
        uint32_t d;
        # 遍历最小重复次数
        for (i = 1; i <= Lmin; i++)
          {
          # 检查是否已经到达字符串末尾
          if (Feptr >= mb->end_subject)
            {
            SCHECK_PARTIAL();
            RRETURN(MATCH_NOMATCH);
            }
          # 逐个获取字符并比较
          GETCHARINC(d, Feptr);
          if (Lc == d || Loc == d) RRETURN(MATCH_NOMATCH);
          }
        }
      else
#endif  /* SUPPORT_UNICODE */

      /* 非 UTF 模式 */
        {
        # 遍历最小重复次数
        for (i = 1; i <= Lmin; i++)
          {
          # 检查是否已经到达字符串末尾
          if (Feptr >= mb->end_subject)
            {
            SCHECK_PARTIAL();
            RRETURN(MATCH_NOMATCH);
            }
          # 逐个比较字符
          if (Lc == *Feptr || Loc == *Feptr) RRETURN(MATCH_NOMATCH);
          Feptr++;
          }
        }

      # 如果最小重复次数等于最大重复次数，则继续循环
      if (Lmin == Lmax) continue;  /* Finished for exact count */

      # 如果是最小重复次数
      if (reptype == REPTYPE_MIN)
        {
#ifdef SUPPORT_UNICODE
        # 如果是 UTF 模式
        if (utf)
          {
          uint32_t d;
          # 无限循环
          for (;;)
            {
            RMATCH(Fecode, RM204);
            if (rrc != MATCH_NOMATCH) RRETURN(rrc);
            if (Lmin++ >= Lmax) RRETURN(MATCH_NOMATCH);
            if (Feptr >= mb->end_subject)
              {
              SCHECK_PARTIAL();
              RRETURN(MATCH_NOMATCH);
              }
            GETCHARINC(d, Feptr);
            if (Lc == d || Loc == d) RRETURN(MATCH_NOMATCH);
            }
          }
        else
        # 如果不支持 Unicode
        # 进入循环，尝试匹配最小到最大次数的重复
        {
        RMATCH(Fecode, RM29);
        # 如果匹配成功，返回匹配结果
        if (rrc != MATCH_NOMATCH) RRETURN(rrc);
        # 如果达到最大重复次数，返回不匹配
        if (Lmin++ >= Lmax) RRETURN(MATCH_NOMATCH);
        # 如果已经匹配到字符串末尾，检查是否需要部分匹配，然后返回不匹配
        if (Feptr >= mb->end_subject)
          {
          SCHECK_PARTIAL();
          RRETURN(MATCH_NOMATCH);
          }
        # 如果当前字符不等于 Lc 或 Loc，则返回不匹配
        if (Lc == *Feptr || Loc == *Feptr) RRETURN(MATCH_NOMATCH);
        # 继续向后移动一个字符
        Feptr++;
        }
        # 不会执行到这里
        }

      # 最大化匹配情况
      else
        {
        # 记录当前位置为起始位置
        Lstart_eptr = Feptr;
        
        # 如果支持 Unicode
        if (utf)
          {
          # 定义变量和循环，尝试匹配最小到最大次数的重复
          uint32_t d;
          for (i = Lmin; i < Lmax; i++)
            {
            int len = 1;
            # 如果已经匹配到字符串末尾，检查是否需要部分匹配，然后跳出循环
            if (Feptr >= mb->end_subject)
              {
              SCHECK_PARTIAL();
              break;
              }
            # 获取当前字符的 Unicode 编码和长度
            GETCHARLEN(d, Feptr, len);
            # 如果当前字符等于 Lc 或 Loc，则跳出循环
            if (Lc == d || Loc == d) break;
            # 继续向后移动一个字符
            Feptr += len;
            }

          # 在 UTF 模式下，\C 后面可能是一个 Unicode 字符的中间位置，使用 <= Lstart_eptr 确保回溯不会过多
          if (reptype != REPTYPE_POS) for(;;)
            {
            # 如果当前位置小于等于起始位置，跳出循环
            if (Feptr <= Lstart_eptr) break;
            # 尝试匹配最小到最大次数的重复
            RMATCH(Fecode, RM205);
            # 如果匹配成功，返回匹配结果
            if (rrc != MATCH_NOMATCH) RRETURN(rrc);
            # 向前移动一个字符
            Feptr--;
            # 回退一个字符
            BACKCHAR(Feptr);
            }
          }
        else
#endif  /* SUPPORT_UNICODE */

        /* Not UTF mode */
          {
          for (i = Lmin; i < Lmax; i++)
            {
            // 如果当前指针超出了主题的末尾，进行部分匹配检查并跳出循环
            if (Feptr >= mb->end_subject)
              {
              SCHECK_PARTIAL();
              break;
              }
            // 如果当前字符与 Lc 或 Loc 相等，跳出循环
            if (Lc == *Feptr || Loc == *Feptr) break;
            // 指针向后移动一位
            Feptr++;
            }
          // 如果不是 POS 类型的重复，执行以下循环
          if (reptype != REPTYPE_POS) for (;;)
            {
            // 如果指针等于起始指针，跳出循环
            if (Feptr == Lstart_eptr) break;
            // 调用 RMATCH 函数，如果匹配成功则返回匹配结果
            RMATCH(Fecode, RM30);
            if (rrc != MATCH_NOMATCH) RRETURN(rrc);
            // 指针向前移动一位
            Feptr--;
            }
          }
        }
      }

    /* Caseful comparisons */

    else
      {
#ifdef SUPPORT_UNICODE
      // 如果支持 UTF 模式
      if (utf)
        {
        uint32_t d;
        for (i = 1; i <= Lmin; i++)
          {
          // 如果当前指针超出了主题的末尾，进行部分匹配检查并返回不匹配
          if (Feptr >= mb->end_subject)
            {
            SCHECK_PARTIAL();
            RRETURN(MATCH_NOMATCH);
            }
          // 读取下一个字符
          GETCHARINC(d, Feptr);
          // 如果 Lc 与当前字符相等，返回不匹配
          if (Lc == d) RRETURN(MATCH_NOMATCH);
          }
        }
      else
#endif
      /* Not UTF mode */
        {
        for (i = 1; i <= Lmin; i++)
          {
          // 如果当前指针超出了主题的末尾，进行部分匹配检查并返回不匹配
          if (Feptr >= mb->end_subject)
            {
            SCHECK_PARTIAL();
            RRETURN(MATCH_NOMATCH);
            }
          // 如果 Lc 与当前字符相等，返回不匹配
          if (Lc == *Feptr++) RRETURN(MATCH_NOMATCH);
          }
        }

      // 如果 Lmin 等于 Lmax，则继续循环
      if (Lmin == Lmax) continue;

      // 如果重复类型为最小重复
      if (reptype == REPTYPE_MIN)
        {
#ifdef SUPPORT_UNICODE
        // 如果支持 UTF 模式
        if (utf)
          {
          uint32_t d;
          for (;;)
            {
            // 调用 RMATCH 函数，如果匹配成功则返回匹配结果
            RMATCH(Fecode, RM206);
            if (rrc != MATCH_NOMATCH) RRETURN(rrc);
            // 如果 Lmin 大于等于 Lmax，返回不匹配
            if (Lmin++ >= Lmax) RRETURN(MATCH_NOMATCH);
            // 如果当前指针超出了主题的末尾，进行部分匹配检查并返回不匹配
            if (Feptr >= mb->end_subject)
              {
              SCHECK_PARTIAL();
              RRETURN(MATCH_NOMATCH);
              }
            // 读取下一个字符
            GETCHARINC(d, Feptr);
            // 如果 Lc 与当前字符相等，返回不匹配
            if (Lc == d) RRETURN(MATCH_NOMATCH);
            }
          }
        else
#endif
        /* 如果不是 UTF 模式 */
          {
          for (;;)
            {
            // 使用 RM31 模式匹配
            RMATCH(Fecode, RM31);
            // 如果匹配成功，返回匹配结果
            if (rrc != MATCH_NOMATCH) RRETURN(rrc);
            // 如果 Lmin 大于等于 Lmax，返回不匹配
            if (Lmin++ >= Lmax) RRETURN(MATCH_NOMATCH);
            // 如果 Feptr 超出了结束位置，检查是否为部分匹配，然后返回不匹配
            if (Feptr >= mb->end_subject)
              {
              SCHECK_PARTIAL();
              RRETURN(MATCH_NOMATCH);
              }
            // 如果 Lc 等于 *Feptr，返回不匹配
            if (Lc == *Feptr++) RRETURN(MATCH_NOMATCH);
            }
          }
        /* 控制永远不会到达这里 */
        }

      /* 最大化情况 */

      else
        {
        // 设置 Lstart_eptr 为 Feptr
        Lstart_eptr = Feptr;

#ifdef SUPPORT_UNICODE
        // 如果是 UTF 模式
        if (utf)
          {
          uint32_t d;
          for (i = Lmin; i < Lmax; i++)
            {
            int len = 1;
            // 如果 Feptr 超出了结束位置，检查是否为部分匹配，然后跳出循环
            if (Feptr >= mb->end_subject)
              {
              SCHECK_PARTIAL();
              break;
              }
            // 获取字符长度并赋值给 d 和 len
            GETCHARLEN(d, Feptr, len);
            // 如果 Lc 等于 d，跳出循环
            if (Lc == d) break;
            // Feptr 增加 len
            Feptr += len;
            }

          /* 在 UTF 模式下，\C 后面可能是一个 Unicode 字符的中间。使用 <= Lstart_eptr 确保回溯不会太远。 */

          // 如果不是贪婪匹配，进行循环
          if (reptype != REPTYPE_POS) for(;;)
            {
            // 如果 Feptr 小于等于 Lstart_eptr，跳出循环
            if (Feptr <= Lstart_eptr) break;
            // 使用 RM207 模式匹配
            RMATCH(Fecode, RM207);
            // 如果匹配成功，返回匹配结果
            if (rrc != MATCH_NOMATCH) RRETURN(rrc);
            // Feptr 减一
            Feptr--;
            // 回退一个字符
            BACKCHAR(Feptr);
            }
          }
        else
#endif
        /* 不是 UTF 模式 */
          {
          for (i = Lmin; i < Lmax; i++)
            {
            // 如果 Feptr 超出了结束位置，检查是否为部分匹配，然后跳出循环
            if (Feptr >= mb->end_subject)
              {
              SCHECK_PARTIAL();
              break;
              }
            // 如果 Lc 等于 *Feptr，跳出循环
            if (Lc == *Feptr) break;
            // Feptr 增加一
            Feptr++;
            }
          // 如果不是贪婪匹配，进行循环
          if (reptype != REPTYPE_POS) for (;;)
            {
            // 如果 Feptr 等于 Lstart_eptr，跳出循环
            if (Feptr == Lstart_eptr) break;
            // 使用 RM32 模式匹配
            RMATCH(Fecode, RM32);
            // 如果匹配成功，返回匹配结果
            if (rrc != MATCH_NOMATCH) RRETURN(rrc);
            // Feptr 减一
            Feptr--;
            }
          }
        }
      }
    break;
/* 重置宏定义，避免重复定义 */
#undef Lstart_eptr
#undef Lmin
#undef Lmax
#undef Lc
#undef Loc

/* ===================================================================== */
/* 匹配一个位图字符类，可能重复。这些操作码在以下情况下使用：
   - 当类中的所有字符的值在0-255范围内，并且匹配是大小写敏感的，或者当启用UTF处理时，字符在0-127范围内。
   当数据字符超出范围时，OP_CLASS和OP_NCLASS之间唯一的区别在于。 */

/* 定义宏，用于临时存储最小值和最大值 */
#define Lmin               F->temp_32[0]
#define Lmax               F->temp_32[1]
/* 定义宏，用于临时存储指针位置 */
#define Lstart_eptr        F->temp_sptr[0]
/* 定义宏，用于临时存储字节映射表的地址 */
#define Lbyte_map_address  F->temp_sptr[1]
/* 定义宏，用于访问字节映射表 */
#define Lbyte_map          ((unsigned char *)Lbyte_map_address)

/* 处理OP_NCLASS操作码的情况 */
case OP_NCLASS:
    # 如果操作码为OP_CLASS
    case OP_CLASS:
      {
      # 保存Fecode + 1的地址，用于匹配
      Lbyte_map_address = Fecode + 1;           /* Save for matching */
      # Fecode指针向前移动，跳过这个项目
      Fecode += 1 + (32 / sizeof(PCRE2_UCHAR)); /* Advance past the item */

      /* 在项目的末尾查看是否有重复信息，然后遵循类似字符类型重复的代码。 */

      # 根据Fecode指针指向的内容进行判断
      switch (*Fecode)
        {
        # 如果是以下情况之一，则执行相似的字符类型重复的代码
        case OP_CRSTAR:
        case OP_CRMINSTAR:
        case OP_CRPLUS:
        case OP_CRMINPLUS:
        case OP_CRQUERY:
        case OP_CRMINQUERY:
        case OP_CRPOSSTAR:
        case OP_CRPOSPLUS:
        case OP_CRPOSQUERY:
        # 计算重复的最小值、最大值和类型
        fc = *Fecode++ - OP_CRSTAR;
        Lmin = rep_min[fc];
        Lmax = rep_max[fc];
        reptype = rep_typ[fc];
        break;

        # 如果是以下情况之一，则执行相似的字符类型重复的代码
        case OP_CRRANGE:
        case OP_CRMINRANGE:
        case OP_CRPOSRANGE:
        # 获取重复的最小值和最大值
        Lmin = GET2(Fecode, 1);
        Lmax = GET2(Fecode, 1 + IMM2_SIZE);
        # 如果最大值为0，则表示无限大
        if (Lmax == 0) Lmax = UINT32_MAX;       /* Max 0 => infinity */
        # 获取重复类型
        reptype = rep_typ[*Fecode - OP_CRSTAR];
        # Fecode指针向前移动，跳过重复信息
        Fecode += 1 + 2 * IMM2_SIZE;
        break;

        # 如果没有重复信息，则最小值和最大值都为1
        default:               /* No repeat follows */
        Lmin = Lmax = 1;
        break;
        }

      /* 首先，确保存在最小数量的匹配。 */
#ifdef SUPPORT_UNICODE
      # 如果支持 Unicode，则执行以下代码块
      if (utf)
        {
        # 如果使用 UTF 模式
        for (i = 1; i <= Lmin; i++)
          {
          # 循环 Lmin 次
          if (Feptr >= mb->end_subject)
            {
            # 如果指针超出了主字符串的末尾
            SCHECK_PARTIAL();
            # 检查是否为部分匹配
            RRETURN(MATCH_NOMATCH);
            # 返回不匹配
            }
          GETCHARINC(fc, Feptr);
          # 获取并增加指针位置的字符
          if (fc > 255)
            {
            # 如果字符大于 255
            if (Fop == OP_CLASS) RRETURN(MATCH_NOMATCH);
            # 如果操作为类操作，则返回不匹配
            }
          else
            if ((Lbyte_map[fc/8] & (1u << (fc&7))) == 0) RRETURN(MATCH_NOMATCH);
          # 否则，如果字符不在字节映射中，则返回不匹配
          }
        }
      else
#endif
      /* Not UTF mode */
        {
        # 如果不是 UTF 模式
        for (i = 1; i <= Lmin; i++)
          {
          # 循环 Lmin 次
          if (Feptr >= mb->end_subject)
            {
            # 如果指针超出了主字符串的末尾
            SCHECK_PARTIAL();
            # 检查是否为部分匹配
            RRETURN(MATCH_NOMATCH);
            # 返回不匹配
            }
          fc = *Feptr++;
          # 获取并增加指针位置的字符
#if PCRE2_CODE_UNIT_WIDTH != 8
          if (fc > 255)
            {
            # 如果字符大于 255
            if (Fop == OP_CLASS) RRETURN(MATCH_NOMATCH);
            # 如果操作为类操作，则返回不匹配
            }
          else
#endif
          if ((Lbyte_map[fc/8] & (1u << (fc&7))) == 0) RRETURN(MATCH_NOMATCH);
          # 否则，如果字符不在字节映射中，则返回不匹配
          }
        }

      /* If Lmax == Lmin we are done. Continue with main loop. */

      # 如果 Lmax == Lmin，则完成。继续主循环。

      if (Lmin == Lmax) continue;

      # 如果 Lmin 等于 Lmax，则继续循环

      /* If minimizing, keep testing the rest of the expression and advancing
      the pointer while it matches the class. */

      # 如果是最小匹配模式，则在匹配类的情况下继续测试表达式的其余部分并推进指针。

      if (reptype == REPTYPE_MIN)
        {
#ifdef SUPPORT_UNICODE
        if (utf)
          {
          # 如果支持 Unicode 并且使用 UTF 模式
          for (;;)
            {
            # 无限循环
            RMATCH(Fecode, RM200);
            # 调用 RMATCH 函数
            if (rrc != MATCH_NOMATCH) RRETURN(rrc);
            # 如果不是不匹配，则返回 rrc
            if (Lmin++ >= Lmax) RRETURN(MATCH_NOMATCH);
            # 如果 Lmin 大于等于 Lmax，则返回不匹配
            if (Feptr >= mb->end_subject)
              {
              # 如果指针超出了主字符串的末尾
              SCHECK_PARTIAL();
              # 检查是否为部分匹配
              RRETURN(MATCH_NOMATCH);
              # 返回不匹配
              }
            GETCHARINC(fc, Feptr);
            # 获取并增加指针位置的字符
            if (fc > 255)
              {
              # 如果字符大于 255
              if (Fop == OP_CLASS) RRETURN(MATCH_NOMATCH);
              # 如果操作为类操作，则返回不匹配
              }
            else
              if ((Lbyte_map[fc/8] & (1u << (fc&7))) == 0) RRETURN(MATCH_NOMATCH);
            # 否则，如果字符不在字节映射中，则返回不匹配
            }
          }
        else
#else
        /* 如果不是 UTF 模式 */
          {
          // 无限循环，直到匹配成功或者达到最大长度
          for (;;)
            {
            // 尝试匹配下一个字符
            RMATCH(Fecode, RM23);
            // 如果匹配成功，返回匹配结果
            if (rrc != MATCH_NOMATCH) RRETURN(rrc);
            // 如果达到最大长度，返回匹配失败
            if (Lmin++ >= Lmax) RRETURN(MATCH_NOMATCH);
            // 如果已经到达输入的末尾，检查是否需要部分匹配，然后返回匹配失败
            if (Feptr >= mb->end_subject)
              {
              SCHECK_PARTIAL();
              RRETURN(MATCH_NOMATCH);
              }
            // 读取下一个字符
            fc = *Feptr++;
            // 如果不是 8 位编码，检查字符是否大于 255，如果是则返回匹配失败
#if PCRE2_CODE_UNIT_WIDTH != 8
            if (fc > 255)
              {
              if (Fop == OP_CLASS) RRETURN(MATCH_NOMATCH);
              }
            else
#endif
            // 如果字符不在字节映射中，返回匹配失败
            if ((Lbyte_map[fc/8] & (1u << (fc&7))) == 0) RRETURN(MATCH_NOMATCH);
            }
          }
        /* 控制永远不会到达这里 */
        }

      /* 如果是最大化匹配，找到最长可能的匹配，然后向后回溯。 */

      else
        {
        // 记录当前位置
        Lstart_eptr = Feptr;

#ifdef SUPPORT_UNICODE
        // 如果是 UTF 模式
        if (utf)
          {
          // 循环直到达到最大长度
          for (i = Lmin; i < Lmax; i++)
            {
            // 获取字符的长度
            int len = 1;
            // 如果已经到达输入的末尾，检查是否需要部分匹配，然后跳出循环
            if (Feptr >= mb->end_subject)
              {
              SCHECK_PARTIAL();
              break;
              }
            // 获取字符并计算长度
            GETCHARLEN(fc, Feptr, len);
            // 如果字符大于 255，根据操作符类型决定是否跳出循环
            if (fc > 255)
              {
              if (Fop == OP_CLASS) break;
              }
            else
              // 如果字符不在字节映射中，跳出循环
              if ((Lbyte_map[fc/8] & (1u << (fc&7))) == 0) break;
            // 移动指针到下一个字符
            Feptr += len;
            }

          // 如果是贪婪匹配，继续下一轮匹配
          if (reptype == REPTYPE_POS) continue;    /* No backtracking */

          /* 在 UTF 模式下，\C 后可能会处于 Unicode 字符的中间。使用 <= Lstart_eptr 确保回溯不会太远。 */
          // 无限循环，直到匹配成功或者达到最大长度
          for (;;)
            {
            // 尝试匹配下一个字符
            RMATCH(Fecode, RM201);
            // 如果匹配成功，返回匹配结果
            if (rrc != MATCH_NOMATCH) RRETURN(rrc);
            // 如果回溯到原始位置，跳出循环
            if (Feptr-- <= Lstart_eptr) break;  /* Tried at original position */
            // 回退一个字符
            BACKCHAR(Feptr);
            }
          }
        else
#endif
          /* 结束条件判断，如果不是 UTF 模式则执行以下代码块 */
          {
          for (i = Lmin; i < Lmax; i++)
            {
            /* 检查是否已经到达字符串末尾 */
            if (Feptr >= mb->end_subject)
              {
              SCHECK_PARTIAL();  /* 检查是否需要部分匹配 */
              break;
              }
            fc = *Feptr;  /* 读取当前字符 */
#if PCRE2_CODE_UNIT_WIDTH != 8
            /* 如果不是 8 位编码，判断字符是否大于 255 */
            if (fc > 255)
              {
              if (Fop == OP_CLASS) break;  /* 如果是字符类操作，则跳出循环 */
              }
            else
#endif
            /* 如果是 8 位编码，判断字符是否在字节映射中 */
            if ((Lbyte_map[fc/8] & (1u << (fc&7))) == 0) break;
            Feptr++;  /* 移动指针到下一个字符 */
            }

          if (reptype == REPTYPE_POS) continue;    /* 如果是 POS 类型的重复操作，则继续下一次匹配 */

          while (Feptr >= Lstart_eptr)
            {
            RMATCH(Fecode, RM24);  /* 递归调用匹配函数 */
            if (rrc != MATCH_NOMATCH) RRETURN(rrc);  /* 如果匹配成功，则返回结果 */
            Feptr--;  /* 向前移动指针 */
            }
          }

        RRETURN(MATCH_NOMATCH);  /* 返回匹配失败 */
        }
      }
    /* 控制永远不会到达这里 */

#undef Lbyte_map_address
#undef Lbyte_map
#undef Lstart_eptr
#undef Lmin
#undef Lmax


    /* ===================================================================== */
    /* 匹配扩展字符类。在 8 位库中，只有在支持 UTF-8 模式时才会遇到此操作码。在 16 位和 32 位库中，即使不支持 UTF，也可能遇到大于 255 的码点。 */

#define Lstart_eptr  F->temp_sptr[0]  /* 设置起始指针 */
#define Lxclass_data F->temp_sptr[1]  /* 设置扩展字符类数据 */
#define Lmin         F->temp_32[0]     /* 设置最小值 */
#define Lmax         F->temp_32[1]     /* 设置最大值 */

#ifdef SUPPORT_WIDE_CHARS
#ifdef SUPPORT_UNICODE
          GETCHARLENTEST(fc, Feptr, len);  /* 获取字符长度测试 */
#else
          fc = *Feptr;  /* 读取当前字符 */
#endif
          # 如果当前字符不是类别字符，跳出循环
          if (!PRIV(xclass)(fc, Lxclass_data, utf)) break;
          # 移动指针到下一个位置
          Feptr += len;
          }

        # 如果是正向匹配，则继续循环
        if (reptype == REPTYPE_POS) continue;    /* No backtracking */

        /* 在 UTF 模式下，\C 后面可能是一个 Unicode 字符的中间位置，使用 <= Lstart_eptr 确保回溯不会走得太远 */
        for(;;)
          {
          RMATCH(Fecode, RM101);
          # 如果匹配成功，返回匹配结果
          if (rrc != MATCH_NOMATCH) RRETURN(rrc);
          # 如果回溯到原始位置，跳出循环
          if (Feptr-- <= Lstart_eptr) break;  /* Tried at original position */
          # 如果是 UTF 模式，回退一个字符
#ifdef SUPPORT_UNICODE
          if (utf) BACKCHAR(Feptr);
#endif
          }
        # 返回不匹配结果
        RRETURN(MATCH_NOMATCH);
        }

      /* 控制永远不会到达这里 */
      }
#endif  /* SUPPORT_WIDE_CHARS: end of XCLASS */

#undef Lstart_eptr
#undef Lxclass_data
#undef Lmin
#undef Lmax


    /* ===================================================================== */
    /* 当 PCRE2_UCP 未设置时，匹配各种字符类型。当 PCRE2_UCP 设置时，不会生成这些操作码 - 而是编译适当的属性测试。 */

    case OP_NOT_DIGIT:
    # 如果指针超出了主串的末尾，检查是否为部分匹配，然后返回不匹配结果
    if (Feptr >= mb->end_subject)
      {
      SCHECK_PARTIAL();
      RRETURN(MATCH_NOMATCH);
      }
    # 获取下一个字符并测试是否为数字字符，如果是则返回不匹配结果
    GETCHARINCTEST(fc, Feptr);
    if (CHMAX_255(fc) && (mb->ctypes[fc] & ctype_digit) != 0)
      RRETURN(MATCH_NOMATCH);
    Fecode++;
    break;

    case OP_DIGIT:
    # 如果指针超出了主串的末尾，检查是否为部分匹配，然后返回不匹配结果
    if (Feptr >= mb->end_subject)
      {
      SCHECK_PARTIAL();
      RRETURN(MATCH_NOMATCH);
      }
    # 获取下一个字符并测试是否为数字字符，如果不是则返回不匹配结果
    GETCHARINCTEST(fc, Feptr);
    if (!CHMAX_255(fc) || (mb->ctypes[fc] & ctype_digit) == 0)
      RRETURN(MATCH_NOMATCH);
    Fecode++;
    break;

    case OP_NOT_WHITESPACE:
    # 如果指针超出了主串的末尾，检查是否为部分匹配，然后返回不匹配结果
    if (Feptr >= mb->end_subject)
      {
      SCHECK_PARTIAL();
      RRETURN(MATCH_NOMATCH);
      }
    # 获取下一个字符并测试是否为空白字符，如果是则返回不匹配结果
    GETCHARINCTEST(fc, Feptr);
    if (CHMAX_255(fc) && (mb->ctypes[fc] & ctype_space) != 0)
      RRETURN(MATCH_NOMATCH);
    Fecode++;
    break;

    case OP_WHITESPACE:
    # 如果指针超出了匹配范围，则进行部分匹配检查，然后返回不匹配
    if (Feptr >= mb->end_subject)
      {
      SCHECK_PARTIAL();
      RRETURN(MATCH_NOMATCH);
      }
    # 获取下一个字符并进行测试
    GETCHARINCTEST(fc, Feptr);
    # 如果字符不是空格或者不是255以内的字符，则返回不匹配
    if (!CHMAX_255(fc) || (mb->ctypes[fc] & ctype_space) == 0)
      RRETURN(MATCH_NOMATCH);
    # 移动指令指针到下一个指令
    Fecode++;
    break;

    case OP_NOT_WORDCHAR:
    # 如果指针超出了匹配范围，则进行部分匹配检查，然后返回不匹配
    if (Feptr >= mb->end_subject)
      {
      SCHECK_PARTIAL();
      RRETURN(MATCH_NOMATCH);
      }
    # 获取下一个字符并进行测试
    GETCHARINCTEST(fc, Feptr);
    # 如果字符是255以内的字符并且不是单词字符，则返回不匹配
    if (CHMAX_255(fc) && (mb->ctypes[fc] & ctype_word) != 0)
      RRETURN(MATCH_NOMATCH);
    # 移动指令指针到下一个指令
    Fecode++;
    break;

    case OP_WORDCHAR:
    # 如果指针超出了匹配范围，则进行部分匹配检查，然后返回不匹配
    if (Feptr >= mb->end_subject)
      {
      SCHECK_PARTIAL();
      RRETURN(MATCH_NOMATCH);
      }
    # 获取下一个字符并进行测试
    GETCHARINCTEST(fc, Feptr);
    # 如果字符不是255以内的字符或者不是单词字符，则返回不匹配
    if (!CHMAX_255(fc) || (mb->ctypes[fc] & ctype_word) == 0)
      RRETURN(MATCH_NOMATCH);
    # 移动指令指针到下一个指令
    Fecode++;
    break;

    case OP_ANYNL:
    # 如果指针超出了匹配范围，则进行部分匹配检查，然后返回不匹配
    if (Feptr >= mb->end_subject)
      {
      SCHECK_PARTIAL();
      RRETURN(MATCH_NOMATCH);
      }
    # 获取下一个字符并进行测试
    GETCHARINCTEST(fc, Feptr);
    # 根据字符的不同情况进行处理
    switch(fc)
      {
      default: RRETURN(MATCH_NOMATCH);

      case CHAR_CR:
      # 如果指针超出了匹配范围，则进行部分匹配检查
      if (Feptr >= mb->end_subject)
        {
        SCHECK_PARTIAL();
        }
      # 如果下一个字符是LF，则移动指针到下一个字符
      else if (UCHAR21TEST(Feptr) == CHAR_LF) Feptr++;
      break;

      case CHAR_LF:
      break;

      case CHAR_VT:
      case CHAR_FF:
      case CHAR_NEL:
#ifndef EBCDIC
      // 如果不是 EBCDIC 编码，则执行以下代码
      case 0x2028:
      case 0x2029:
#endif  /* Not EBCDIC */
      // 如果换行符不是 EBCDIC 编码，且换行符不匹配当前的行结束符约定，则返回不匹配
      if (mb->bsr_convention == PCRE2_BSR_ANYCRLF) RRETURN(MATCH_NOMATCH);
      // 结束 switch 语句
      break;
      }
    // Fecode 自增
    Fecode++;
    // 结束 switch 语句
    break;

    case OP_NOT_HSPACE:
    // 如果 Feptr 指向的位置超出了主字符串的末尾，则检查是否部分匹配，然后返回不匹配
    if (Feptr >= mb->end_subject)
      {
      SCHECK_PARTIAL();
      RRETURN(MATCH_NOMATCH);
      }
    // 获取下一个字符并测试是否是水平空白字符
    GETCHARINCTEST(fc, Feptr);
    // 根据获取的字符进行判断
    switch(fc)
      {
      HSPACE_CASES: RRETURN(MATCH_NOMATCH);  /* Byte and multibyte cases */
      default: break;
      }
    // Fecode 自增
    Fecode++;
    // 结束 switch 语句
    break;

    case OP_HSPACE:
    // 如果 Feptr 指向的位置超出了主字符串的末尾，则检查是否部分匹配，然后返回不匹配
    if (Feptr >= mb->end_subject)
      {
      SCHECK_PARTIAL();
      RRETURN(MATCH_NOMATCH);
      }
    // 获取下一个字符并测试是否是水平空白字符
    GETCHARINCTEST(fc, Feptr);
    // 根据获取的字符进行判断
    switch(fc)
      {
      HSPACE_CASES: break;  /* Byte and multibyte cases */
      default: RRETURN(MATCH_NOMATCH);
      }
    // Fecode 自增
    // 结束 switch 语句
    break;

    case OP_NOT_VSPACE:
    // 如果 Feptr 指向的位置超出了主字符串的末尾，则检查是否部分匹配，然后返回不匹配
    if (Feptr >= mb->end_subject)
      {
      SCHECK_PARTIAL();
      RRETURN(MATCH_NOMATCH);
      }
    // 获取下一个字符并测试是否是垂直空白字符
    GETCHARINCTEST(fc, Feptr);
    // 根据获取的字符进行判断
    switch(fc)
      {
      VSPACE_CASES: RRETURN(MATCH_NOMATCH);
      default: break;
      }
    // Fecode 自增
    // 结束 switch 语句
    break;

    case OP_VSPACE:
    // 如果 Feptr 指向的位置超出了主字符串的末尾，则检查是否部分匹配，然后返回不匹配
    if (Feptr >= mb->end_subject)
      {
      SCHECK_PARTIAL();
      RRETURN(MATCH_NOMATCH);
      }
    // 获取下一个字符并测试是否是垂直空白字符
    GETCHARINCTEST(fc, Feptr);
    // 根据获取的字符进行判断
    switch(fc)
      {
      VSPACE_CASES: break;
      default: RRETURN(MATCH_NOMATCH);
      }
    // Fecode 自增
    // 结束 switch 语句
    break;


#ifdef SUPPORT_UNICODE

    /* ===================================================================== */
    /* Check the next character by Unicode property. We will get here only
    if the support is in the binary; otherwise a compile-time error occurs. */

    case OP_PROP:
    case OP_NOTPROP:
    // 如果 Feptr 指向的位置超出了主字符串的末尾，则检查是否部分匹配，然后返回不匹配
    if (Feptr >= mb->end_subject)
      {
      SCHECK_PARTIAL();
      RRETURN(MATCH_NOMATCH);
      }
    // 结束 switch 语句
    break;


    /* ===================================================================== */
    /* Match an extended Unicode sequence. We will get here only if the support
    /* 
    如果 Feptr 大于等于 mb->end_subject，则进行部分匹配检查，如果不满足部分匹配条件则返回不匹配
    */
    case OP_EXTUNI:
    if (Feptr >= mb->end_subject)
      {
      SCHECK_PARTIAL();
      RRETURN(MATCH_NOMATCH);
      }
    else
      {
      /*
      从 Feptr 指向的位置读取一个字符，并将 Feptr 移动到下一个字符的位置
      */
      GETCHARINCTEST(fc, Feptr);
      /*
      调用 extuni 函数处理 Unicode 字符，更新 Feptr 的位置
      */
      Feptr = PRIV(extuni)(fc, Feptr, mb->start_subject, mb->end_subject, utf,
        NULL);
      }
    /*
    进行部分匹配检查
    */
    CHECK_PARTIAL();
    /*
    更新 Fecode 的值
    */
    Fecode++;
    /*
    结束当前 case
    */
    break;
#endif  /* SUPPORT_UNICODE */


    /* ===================================================================== */
    /* Match a single character type repeatedly. Note that the property type
    does not need to be in a stack frame as it is not used within an RMATCH()
    loop. */

#define Lstart_eptr  F->temp_sptr[0]  // 定义起始指针
#define Lmin         F->temp_32[0]     // 定义最小匹配次数
#define Lmax         F->temp_32[1]     // 定义最大匹配次数
#define Lctype       F->temp_32[2]     // 定义字符类型
#define Lpropvalue   F->temp_32[3]     // 定义属性值

    case OP_TYPEEXACT:  // 匹配精确次数
    Lmin = Lmax = GET2(Fecode, 1);  // 获取精确次数
    Fecode += 1 + IMM2_SIZE;  // 移动指针到下一个操作码
    goto REPEATTYPE;  // 跳转到 REPEATTYPE 标签

    case OP_TYPEUPTO:  // 匹配最多次数
    case OP_TYPEMINUPTO:  // 匹配最少到最多次数
    Lmin = 0;  // 最小匹配次数为 0
    Lmax = GET2(Fecode, 1);  // 获取最大匹配次数
    reptype = (*Fecode == OP_TYPEMINUPTO)? REPTYPE_MIN : REPTYPE_MAX;  // 根据操作码设置重复类型
    Fecode += 1 + IMM2_SIZE;  // 移动指针到下一个操作码
    goto REPEATTYPE;  // 跳转到 REPEATTYPE 标签

    case OP_TYPEPOSSTAR:  // 匹配最少 0 次
    reptype = REPTYPE_POS;  // 设置重复类型为 POS
    Lmin = 0;  // 最小匹配次数为 0
    Lmax = UINT32_MAX;  // 最大匹配次数为无穷大
    Fecode++;  // 移动指针到下一个操作码
    goto REPEATTYPE;  // 跳转到 REPEATTYPE 标签

    case OP_TYPEPOSPLUS:  // 匹配最少 1 次
    reptype = REPTYPE_POS;  // 设置重复类型为 POS
    Lmin = 1;  // 最小匹配次数为 1
    Lmax = UINT32_MAX;  // 最大匹配次数为无穷大
    Fecode++;  // 移动指针到下一个操作码
    goto REPEATTYPE;  // 跳转到 REPEATTYPE 标签

    case OP_TYPEPOSQUERY:  // 匹配最少 0 次，最多 1 次
    reptype = REPTYPE_POS;  // 设置重复类型为 POS
    Lmin = 0;  // 最小匹配次数为 0
    Lmax = 1;  // 最大匹配次数为 1
    Fecode++;  // 移动指针到下一个操作码
    goto REPEATTYPE;  // 跳转到 REPEATTYPE 标签

    case OP_TYPEPOSUPTO:  // 匹配最少 0 次，最多指定次数
    reptype = REPTYPE_POS;  // 设置重复类型为 POS
    Lmin = 0;  // 最小匹配次数为 0
    Lmax = GET2(Fecode, 1);  // 获取最大匹配次数
    Fecode += 1 + IMM2_SIZE;  // 移动指针到下一个操作码
    goto REPEATTYPE;  // 跳转到 REPEATTYPE 标签

    case OP_TYPESTAR:  // 匹配 0 次或多次
    case OP_TYPEMINSTAR:  // 匹配 0 次或多次（最少匹配）
    case OP_TYPEPLUS:  // 匹配 1 次或多次
    case OP_TYPEMINPLUS:  // 匹配 1 次或多次（最少匹配）
    case OP_TYPEQUERY:  // 匹配 0 次或 1 次
    case OP_TYPEMINQUERY:  // 匹配 0 次或 1 次（最少匹配）
    fc = *Fecode++ - OP_TYPESTAR;  // 计算重复类型
    Lmin = rep_min[fc];  // 获取最小匹配次数
    Lmax = rep_max[fc];  // 获取最大匹配次数
    reptype = rep_typ[fc];  // 获取重复类型

    /* Common code for all repeated character type matches. */

    REPEATTYPE:  // 重复类型标签
    Lctype = *Fecode++;      /* Code for the character type */  // 获取字符类型的操作码

#ifdef SUPPORT_UNICODE
    if (Lctype == OP_PROP || Lctype == OP_NOTPROP)  // 如果字符类型为属性或非属性
      {
      proptype = *Fecode++;  // 获取属性类型
      Lpropvalue = *Fecode++;  // 获取属性值
      }
    else proptype = -1;  // 否则属性类型为 -1
#endif

    /* First, ensure the minimum number of matches are present. Use inline  // 确保最小匹配次数存在。使用内联
    # 用于最大化速度的代码，并在开始时进行类型测试（即将其放在循环之外）。由于循环中没有对RMATCH的调用，我们可以使用普通变量来表示"notmatch"。UTF模式的代码被分离出来以保持整洁，除了Unicode属性测试。
    # 如果Lmin大于0
    if (Lmin > 0)
      {
#endif     /* SUPPORT_UNICODE */

/* 在 UTF 模式下处理所有其他情况 */

#ifdef SUPPORT_UNICODE
      if (utf) switch(Lctype)
        {
        case OP_ANY:
        for (i = 1; i <= Lmin; i++)
          {
          if (Feptr >= mb->end_subject)
            {
            SCHECK_PARTIAL();
            RRETURN(MATCH_NOMATCH);
            }
          if (IS_NEWLINE(Feptr)) RRETURN(MATCH_NOMATCH);
          if (mb->partial != 0 &&
              Feptr + 1 >= mb->end_subject &&
              NLBLOCK->nltype == NLTYPE_FIXED &&
              NLBLOCK->nllen == 2 &&
              UCHAR21(Feptr) == NLBLOCK->nl[0])
            {
            mb->hitend = TRUE;
            if (mb->partial > 1) return PCRE2_ERROR_PARTIAL;
            }
          Feptr++;
          ACROSSCHAR(Feptr < mb->end_subject, Feptr, Feptr++);
          }
        break;

        case OP_ALLANY:
        for (i = 1; i <= Lmin; i++)
          {
          if (Feptr >= mb->end_subject)
            {
            SCHECK_PARTIAL();
            RRETURN(MATCH_NOMATCH);
            }
          Feptr++;
          ACROSSCHAR(Feptr < mb->end_subject, Feptr, Feptr++);
          }
        break;

        case OP_ANYBYTE:
        if (Feptr > mb->end_subject - Lmin) RRETURN(MATCH_NOMATCH);
        Feptr += Lmin;
        break;

        case OP_ANYNL:
        for (i = 1; i <= Lmin; i++)
          {
          if (Feptr >= mb->end_subject)
            {
            SCHECK_PARTIAL();
            RRETURN(MATCH_NOMATCH);
            }
          GETCHARINC(fc, Feptr);
          switch(fc)
            {
            default: RRETURN(MATCH_NOMATCH);

            case CHAR_CR:
            if (Feptr < mb->end_subject && UCHAR21(Feptr) == CHAR_LF) Feptr++;
            break;

            case CHAR_LF:
            break;

            case CHAR_VT:
            case CHAR_FF:
            case CHAR_NEL:
#ifndef EBCDIC
            case 0x2028:
            case 0x2029:
#endif     /* SUPPORT_UNICODE */

      /* 对于不支持 UTF 的情况下，用于最小匹配操作符 OP_PROP 和 OP_NOTPROP 之外的操作符的代码 */

      switch(Lctype)
        {
        case OP_ANY:
        for (i = 1; i <= Lmin; i++)
          {
          // 如果指针超出了主题的末尾，检查是否部分匹配，然后返回不匹配
          if (Feptr >= mb->end_subject)
            {
            SCHECK_PARTIAL();
            RRETURN(MATCH_NOMATCH);
            }
          // 如果是换行符，返回不匹配
          if (IS_NEWLINE(Feptr)) RRETURN(MATCH_NOMATCH);
          // 如果启用了部分匹配，并且指针接近末尾，且换行符类型为固定长度为2的情况下，且当前字符为换行符的第一个字符
          // 标记已经到达末尾，如果部分匹配大于1，返回部分匹配错误
          if (mb->partial != 0 &&
              Feptr + 1 >= mb->end_subject &&
              NLBLOCK->nltype == NLTYPE_FIXED &&
              NLBLOCK->nllen == 2 &&
              *Feptr == NLBLOCK->nl[0])
            {
            mb->hitend = TRUE;
            if (mb->partial > 1) return PCRE2_ERROR_PARTIAL;
            }
          Feptr++;
          }
        break;

        case OP_ALLANY:
        // 如果指针超出了主题的末尾，检查是否部分匹配，然后返回不匹配
        if (Feptr > mb->end_subject - Lmin)
          {
          SCHECK_PARTIAL();
          RRETURN(MATCH_NOMATCH);
          }
        // 指针向前移动 Lmin 个位置
        Feptr += Lmin;
        break;

        /* 这种 OP_ANYBYTE 情况永远不会被触发，因为在非 UTF 模式下 \C 会被转换为 OP_ALLANY。
        剪切掉这段代码，以免覆盖报告抱怨它从未被使用。 */

/*        case OP_ANYBYTE:
*        if (Feptr > mb->end_subject - Lmin)
*          {
*          SCHECK_PARTIAL();
*          RRETURN(MATCH_NOMATCH);
*          }
*        Feptr += Lmin;
*        break;
*/
        case OP_ANYNL:
        for (i = 1; i <= Lmin; i++)
          {
          // 如果指针超出了主题的末尾，检查是否部分匹配，然后返回不匹配
          if (Feptr >= mb->end_subject)
            {
            SCHECK_PARTIAL();
            RRETURN(MATCH_NOMATCH);
            }
          switch(*Feptr++)
            {
            default: RRETURN(MATCH_NOMATCH);

            case CHAR_CR:
            // 如果当前字符为回车符，且下一个字符为换行符，指针向前移动一个位置
            if (Feptr < mb->end_subject && *Feptr == CHAR_LF) Feptr++;
            break;

            case CHAR_LF:
            break;

            case CHAR_VT:
            case CHAR_FF:
            case CHAR_NEL:
#if PCRE2_CODE_UNIT_WIDTH != 8
            // 如果 PCRE2_CODE_UNIT_WIDTH 不等于 8，则执行以下代码
            case 0x2028:
            // 匹配 Unicode 行分隔符
            case 0x2029:
            // 匹配 Unicode 段分隔符
#endif
            // 如果换行符约定为 PCRE2_BSR_ANYCRLF，则返回不匹配
            if (mb->bsr_convention == PCRE2_BSR_ANYCRLF) RRETURN(MATCH_NOMATCH);
            // 结束 switch 语句
            break;
            }
          }
        break;

        case OP_NOT_HSPACE:
        // 匹配非水平空白字符
        for (i = 1; i <= Lmin; i++)
          {
          if (Feptr >= mb->end_subject)
            {
            SCHECK_PARTIAL();
            RRETURN(MATCH_NOMATCH);
            }
          // 开始 switch 语句，根据字符类型进行匹配
          switch(*Feptr++)
            {
            default: break;
            // 匹配单字节水平空白字符
            HSPACE_BYTE_CASES:
#if PCRE2_CODE_UNIT_WIDTH != 8
            // 如果 PCRE2_CODE_UNIT_WIDTH 不等于 8，则执行以下代码
            HSPACE_MULTIBYTE_CASES:
#endif
            // 返回不匹配
            RRETURN(MATCH_NOMATCH);
            }
          }
        break;

        case OP_HSPACE:
        // 匹配水平空白字符
        for (i = 1; i <= Lmin; i++)
          {
          if (Feptr >= mb->end_subject)
            {
            SCHECK_PARTIAL();
            RRETURN(MATCH_NOMATCH);
            }
          // 开始 switch 语句，根据字符类型进行匹配
          switch(*Feptr++)
            {
            default: RRETURN(MATCH_NOMATCH);
            // 匹配单字节水平空白字符
            HSPACE_BYTE_CASES:
#if PCRE2_CODE_UNIT_WIDTH != 8
            // 如果 PCRE2_CODE_UNIT_WIDTH 不等于 8，则执行以下代码
            HSPACE_MULTIBYTE_CASES:
#endif
            // 结束 switch 语句
            break;
            }
          }
        break;

        case OP_NOT_VSPACE:
        // 匹配非垂直空白字符
        for (i = 1; i <= Lmin; i++)
          {
          if (Feptr >= mb->end_subject)
            {
            SCHECK_PARTIAL();
            RRETURN(MATCH_NOMATCH);
            }
          // 开始 switch 语句，根据字符类型进行匹配
          switch(*Feptr++)
            {
            // 匹配单字节垂直空白字符
            VSPACE_BYTE_CASES:
#if PCRE2_CODE_UNIT_WIDTH != 8
            // 如果 PCRE2_CODE_UNIT_WIDTH 不等于 8，则执行以下代码
            VSPACE_MULTIBYTE_CASES:
#endif
            // 返回不匹配
            RRETURN(MATCH_NOMATCH);
            default: break;
            }
          }
        break;

        case OP_VSPACE:
        // 匹配垂直空白字符
        for (i = 1; i <= Lmin; i++)
          {
          if (Feptr >= mb->end_subject)
            {
            SCHECK_PARTIAL();
            RRETURN(MATCH_NOMATCH);
            }
          // 开始 switch 语句，根据字符类型进行匹配
          switch(*Feptr++)
            {
            default: RRETURN(MATCH_NOMATCH);
            // 匹配单字节垂直空白字符
            VSPACE_BYTE_CASES:
#if PCRE2_CODE_UNIT_WIDTH != 8
            VSPACE_MULTIBYTE_CASES:
    # 如果 PCRE2_CODE_UNIT_WIDTH 不等于 8，则执行以下操作
    /* If Lmin = Lmax we are done. Continue with the main loop. */
    # 如果 Lmin 等于 Lmax，则执行主循环
    if (Lmin == Lmax) continue;
    # 如果 Lmin 不等于 Lmax，则继续执行
    /* If minimizing, we have to test the rest of the pattern before each
    subsequent match. This means we cannot use a local "notmatch" variable as
    in the other cases. As all 4 temporary 32-bit values in the frame are
    already in use, just test the type each time. */
    # 如果是最小化操作，需要在每次匹配之前测试模式的剩余部分。这意味着我们不能像其他情况那样使用本地的 "notmatch" 变量。由于帧中的所有 4 个临时 32 位值已经在使用中，因此每次只需测试类型。
    if (reptype == REPTYPE_MIN)
      {
#endif     /* SUPPORT_UNICODE */
    # 如果是最小化操作，则执行以下操作
      /* UTF mode for non-property testing character types. */
#ifdef SUPPORT_UNICODE
      # 如果支持 Unicode
      if (utf)
        {
        # 如果是 UTF 编码
        for (;;)
          {
          # 循环匹配
          RMATCH(Fecode, RM219);
          # 调用 RMATCH 函数进行匹配
          if (rrc != MATCH_NOMATCH) RRETURN(rrc);
          # 如果匹配结果不是 NOMATCH，则返回匹配结果
          if (Lmin++ >= Lmax) RRETURN(MATCH_NOMATCH);
          # 如果最小长度大于等于最大长度，则返回 NOMATCH
          if (Feptr >= mb->end_subject)
            {
            SCHECK_PARTIAL();
            RRETURN(MATCH_NOMATCH);
            }
          # 如果当前位置超出主题字符串范围，则检查是否为部分匹配，然后返回 NOMATCH
          if (Lctype == OP_ANY && IS_NEWLINE(Feptr)) RRETURN(MATCH_NOMATCH);
          # 如果当前字符类型为 OP_ANY 且是换行符，则返回 NOMATCH
          GETCHARINC(fc, Feptr);
          # 获取并增加字符指针位置
          switch(Lctype)
            {
            # 根据字符类型进行不同的处理
            case OP_ANY:               /* This is the non-NL case */
            # 如果字符类型为 OP_ANY（非换行符）
            if (mb->partial != 0 &&    /* Take care with CRLF partial */
                Feptr >= mb->end_subject &&
                NLBLOCK->nltype == NLTYPE_FIXED &&
                NLBLOCK->nllen == 2 &&
                fc == NLBLOCK->nl[0])
              {
              mb->hitend = TRUE;
              if (mb->partial > 1) return PCRE2_ERROR_PARTIAL;
              }
            break;
            # 如果是部分匹配，则设置 hitend 为 TRUE，如果 partial 大于 1，则返回部分匹配错误

            case OP_ALLANY:
            case OP_ANYBYTE:
            break;
            # 如果字符类型为 OP_ALLANY 或 OP_ANYBYTE，则不做处理

            case OP_ANYNL:
            # 如果字符类型为 OP_ANYNL
            switch(fc)
              {
              # 根据当前字符进行不同的处理
              default: RRETURN(MATCH_NOMATCH);
              # 默认情况下返回 NOMATCH

              case CHAR_CR:
              if (Feptr < mb->end_subject && UCHAR21(Feptr) == CHAR_LF) Feptr++;
              break;
              # 如果是回车符，则如果下一个字符是换行符，则字符指针向后移动一位

              case CHAR_LF:
              break;
              # 如果是换行符，则不做处理

              case CHAR_VT:
              case CHAR_FF:
              case CHAR_NEL:
#ifndef EBCDIC
              case 0x2028:
              case 0x2029:
              # 如果是垂直制表符、换页符、下一行、U+2028、U+2029
#else  /* Not EBCDIC */
              # 如果不是 EBCDIC 编码，则执行以下代码
              if (mb->bsr_convention == PCRE2_BSR_ANYCRLF)
                # 如果换行符是任意的 CRLF，则返回不匹配
                RRETURN(MATCH_NOMATCH);
              # 结束当前 case
              break;
              }
            # 结束当前 switch
            break;

            case OP_NOT_HSPACE:
            # 对于非水平空白字符的情况
            switch(fc)
              {
              # 对于水平空白字符的情况，返回不匹配
              HSPACE_CASES: RRETURN(MATCH_NOMATCH);
              # 对于其他情况，继续执行
              default: break;
              }
            # 结束当前 case
            break;

            case OP_HSPACE:
            # 对于水平空白字符的情况
            switch(fc)
              {
              # 对于水平空白字符的情况，继续执行
              HSPACE_CASES: break;
              # 对于其他情况，返回不匹配
              default: RRETURN(MATCH_NOMATCH);
              }
            # 结束当前 case
            break;

            case OP_NOT_VSPACE:
            # 对于非垂直空白字符的情况
            switch(fc)
              {
              # 对于垂直空白字符的情况，返回不匹配
              VSPACE_CASES: RRETURN(MATCH_NOMATCH);
              # 对于其他情况，继续执行
              default: break;
              }
            # 结束当前 case
            break;

            case OP_VSPACE:
            # 对于垂直空白字符的情况
            switch(fc)
              {
              # 对于垂直空白字符的情况，继续执行
              VSPACE_CASES: break;
              # 对于其他情况，返回不匹配
              default: RRETURN(MATCH_NOMATCH);
              }
            # 结束当前 case
            break;

            case OP_NOT_DIGIT:
            # 对于非数字字符的情况
            if (fc < 256 && (mb->ctypes[fc] & ctype_digit) != 0)
              # 如果字符小于 256 并且是数字字符，则返回不匹配
              RRETURN(MATCH_NOMATCH);
            # 结束当前 case
            break;

            case OP_DIGIT:
            # 对于数字字符的情况
            if (fc >= 256 || (mb->ctypes[fc] & ctype_digit) == 0)
              # 如果字符大于等于 256 或者不是数字字符，则返回不匹配
              RRETURN(MATCH_NOMATCH);
            # 结束当前 case
            break;

            case OP_NOT_WHITESPACE:
            # 对于非空白字符的情况
            if (fc < 256 && (mb->ctypes[fc] & ctype_space) != 0)
              # 如果字符小于 256 并且是空白字符，则返回不匹配
              RRETURN(MATCH_NOMATCH);
            # 结束当前 case
            break;

            case OP_WHITESPACE:
            # 对于空白字符的情况
            if (fc >= 256 || (mb->ctypes[fc] & ctype_space) == 0)
              # 如果字符大于等于 256 或者不是空白字符，则返回不匹配
              RRETURN(MATCH_NOMATCH);
            # 结束当前 case
            break;

            case OP_NOT_WORDCHAR:
            # 对于非单词字符的情况
            if (fc < 256 && (mb->ctypes[fc] & ctype_word) != 0)
              # 如果字符小于 256 并且是单词字符，则返回不匹配
              RRETURN(MATCH_NOMATCH);
            # 结束当前 case
            break;

            case OP_WORDCHAR:
            # 对于单词字符的情况
            if (fc >= 256 || (mb->ctypes[fc] & ctype_word) == 0)
              # 如果字符大于等于 256 或者不是单词字符，则返回不匹配
              RRETURN(MATCH_NOMATCH);
            # 结束当前 case
            break;

            default:
            # 默认情况，返回内部错误
            return PCRE2_ERROR_INTERNAL;
            }
          }
        }
      else
#endif  /* SUPPORT_UNICODE */

      /* Not UTF mode */
        {
        // 无限循环，直到条件满足
        for (;;)
          {
          // 使用 RM33 模式匹配
          RMATCH(Fecode, RM33);
          // 如果匹配成功，返回匹配结果
          if (rrc != MATCH_NOMATCH) RRETURN(rrc);
          // 如果 Lmin 大于等于 Lmax，返回不匹配
          if (Lmin++ >= Lmax) RRETURN(MATCH_NOMATCH);
          // 如果 Feptr 超出了主字符串的末尾，检查是否为部分匹配，然后返回不匹配
          if (Feptr >= mb->end_subject)
            {
            SCHECK_PARTIAL();
            RRETURN(MATCH_NOMATCH);
            }
          // 如果 Lctype 为 OP_ANY 且 Feptr 指向换行符，返回不匹配
          if (Lctype == OP_ANY && IS_NEWLINE(Feptr))
            RRETURN(MATCH_NOMATCH);
          // 读取 Feptr 指向的字符
          fc = *Feptr++;
          // 根据 Lctype 的不同进行不同的处理
          switch(Lctype)
            {
            case OP_ANY:               /* This is the non-NL case */
            // 如果为部分匹配且 Feptr 指向主字符串末尾，且换行符类型为固定长度且为 CRLF，且 fc 为 NLBLOCK 中的换行符，设置 hitend 为 TRUE，如果部分匹配大于 1，返回部分匹配错误
            if (mb->partial != 0 &&    
                Feptr >= mb->end_subject &&
                NLBLOCK->nltype == NLTYPE_FIXED &&
                NLBLOCK->nllen == 2 &&
                fc == NLBLOCK->nl[0])
              {
              mb->hitend = TRUE;
              if (mb->partial > 1) return PCRE2_ERROR_PARTIAL;
              }
            break;

            case OP_ALLANY:
            case OP_ANYBYTE:
            break;

            case OP_ANYNL:
            // 根据不同的换行符类型进行处理
            switch(fc)
              {
              default: RRETURN(MATCH_NOMATCH);

              case CHAR_CR:
              // 如果 Feptr 指向主字符串末尾且下一个字符为 LF，移动 Feptr 指向下一个字符
              if (Feptr < mb->end_subject && *Feptr == CHAR_LF) Feptr++;
              break;

              case CHAR_LF:
              break;

              case CHAR_VT:
              case CHAR_FF:
              case CHAR_NEL:
#if PCRE2_CODE_UNIT_WIDTH != 8
              case 0x2028:
              case 0x2029:
#endif
              // 如果换行符类型为 ANYCRLF，返回不匹配
              if (mb->bsr_convention == PCRE2_BSR_ANYCRLF)
                RRETURN(MATCH_NOMATCH);
              break;
              }
            break;

            case OP_NOT_HSPACE:
            // 根据不同的字符类型进行处理
            switch(fc)
              {
              default: break;
              HSPACE_BYTE_CASES:
#if PCRE2_CODE_UNIT_WIDTH != 8
              HSPACE_MULTIBYTE_CASES:
#endif
              // 如果条件不满足，返回不匹配
              RRETURN(MATCH_NOMATCH);
              }
            break;

            case OP_HSPACE:
            // 检查是否是水平空白字符
            switch(fc)
              {
              default: RRETURN(MATCH_NOMATCH);
              // 根据不同的字符类型进行匹配
              HSPACE_BYTE_CASES:
#if PCRE2_CODE_UNIT_WIDTH != 8
              HSPACE_MULTIBYTE_CASES:
#endif
              break;
              }
            break;

            case OP_NOT_VSPACE:
            // 检查是否不是垂直空白字符
            switch(fc)
              {
              default: break;
              // 根据不同的字符类型进行匹配
              VSPACE_BYTE_CASES:
#if PCRE2_CODE_UNIT_WIDTH != 8
              VSPACE_MULTIBYTE_CASES:
#endif
              // 如果条件不满足，返回不匹配
              RRETURN(MATCH_NOMATCH);
              }
            break;

            case OP_VSPACE:
            // 检查是否是垂直空白字符
            switch(fc)
              {
              default: RRETURN(MATCH_NOMATCH);
              // 根据不同的字符类型进行匹配
              VSPACE_BYTE_CASES:
#if PCRE2_CODE_UNIT_WIDTH != 8
              VSPACE_MULTIBYTE_CASES:
#else
              // 如果条件不成立，跳出当前循环
              break;
              }
            // 跳出 switch 语句
            break;

            case OP_NOT_DIGIT:
            // 如果字符不是数字，并且字符编码小于 255
            if (MAX_255(fc) && (mb->ctypes[fc] & ctype_digit) != 0)
              // 返回不匹配
              RRETURN(MATCH_NOMATCH);
            // 跳出 case
            break;

            case OP_DIGIT:
            // 如果字符是数字或者字符编码大于 255
            if (!MAX_255(fc) || (mb->ctypes[fc] & ctype_digit) == 0)
              // 返回不匹配
              RRETURN(MATCH_NOMATCH);
            // 跳出 case
            break;

            case OP_NOT_WHITESPACE:
            // 如果字符不是空白字符，并且字符编码小于 255
            if (MAX_255(fc) && (mb->ctypes[fc] & ctype_space) != 0)
              // 返回不匹配
              RRETURN(MATCH_NOMATCH);
            // 跳出 case
            break;

            case OP_WHITESPACE:
            // 如果字符是空白字符或者字符编码大于 255
            if (!MAX_255(fc) || (mb->ctypes[fc] & ctype_space) == 0)
              // 返回不匹配
              RRETURN(MATCH_NOMATCH);
            // 跳出 case
            break;

            case OP_NOT_WORDCHAR:
            // 如果字符不是单词字符，并且字符编码小于 255
            if (MAX_255(fc) && (mb->ctypes[fc] & ctype_word) != 0)
              // 返回不匹配
              RRETURN(MATCH_NOMATCH);
            // 跳出 case
            break;

            case OP_WORDCHAR:
            // 如果字符是单词字符或者字符编码大于 255
            if (!MAX_255(fc) || (mb->ctypes[fc] & ctype_word) == 0)
              // 返回不匹配
              RRETURN(MATCH_NOMATCH);
            // 跳出 case
            break;

            default:
            // 返回内部错误
            return PCRE2_ERROR_INTERNAL;
            }
          }
        }
      // 控制永远不会到达这里
      }

    // 如果最大化，值得使用内联代码来提高速度，在开始时进行类型测试（即将其排除在循环之外）。再次，"notmatch" 可以是普通的局部变量，因为循环不调用 RMATCH。
    else
      {
      Lstart_eptr = Feptr;  /* 记住我们开始的位置 */

#endif   /* SUPPORT_UNICODE */

#ifndef EBCDIC
                    && fc != 0x2028 && fc != 0x2029
#endif  /* SUPPORT_UNICODE */  // 如果不支持 Unicode，则结束条件编译

      /* Not UTF mode */  // 不是 UTF 模式
        {
        switch(Lctype)  // 根据字符类型进行判断
          {
          case OP_ANY:  // 任意字符
          for (i = Lmin; i < Lmax; i++)  // 循环匹配最小到最大次数
            {
            if (Feptr >= mb->end_subject)  // 如果指针超出了主题字符串的末尾
              {
              SCHECK_PARTIAL();  // 检查是否为部分匹配
              break;  // 跳出循环
              }
            if (IS_NEWLINE(Feptr)) break;  // 如果是换行符，则跳出循环
            if (mb->partial != 0 &&    /* Take care with CRLF partial */  // 如果是部分匹配并且是 CRLF
                Feptr + 1 >= mb->end_subject &&  // 如果指针加一后超出了主题字符串的末尾
                NLBLOCK->nltype == NLTYPE_FIXED &&  // 换行符类型为固定长度
                NLBLOCK->nllen == 2 &&  // 换行符长度为 2
                *Feptr == NLBLOCK->nl[0])  // 当前字符是换行符的第一个字符
              {
              mb->hitend = TRUE;  // 设置匹配到末尾标志为真
              if (mb->partial > 1) return PCRE2_ERROR_PARTIAL;  // 如果部分匹配大于 1，则返回部分匹配错误
              }
            Feptr++;  // 指针后移
            }
          break;  // 结束循环

          case OP_ALLANY:  // 匹配任意字符
          case OP_ANYBYTE:  // 匹配任意字节
          fc = Lmax - Lmin;  // 计算匹配次数
          if (fc > (uint32_t)(mb->end_subject - Feptr))  // 如果匹配次数大于剩余主题字符串长度
            {
            Feptr = mb->end_subject;  // 指针指向主题字符串末尾
            SCHECK_PARTIAL();  // 检查是否为部分匹配
            }
          else Feptr += fc;  // 否则指针后移匹配次数
          break;  // 结束循环

          case OP_ANYNL:  // 匹配任意换行符
          for (i = Lmin; i < Lmax; i++)  // 循环匹配最小到最大次数
            {
            if (Feptr >= mb->end_subject)  // 如果指针超出了主题字符串的末尾
              {
              SCHECK_PARTIAL();  // 检查是否为部分匹配
              break;  // 跳出循环
              }
            fc = *Feptr;  // 获取当前字符
            if (fc == CHAR_CR)  // 如果是回车符
              {
              if (++Feptr >= mb->end_subject) break;  // 指针后移并判断是否超出主题字符串末尾
              if (*Feptr == CHAR_LF) Feptr++;  // 如果下一个字符是换行符，则指针再后移一位
              }
            else  // 否则
              {
              if (fc != CHAR_LF && (mb->bsr_convention == PCRE2_BSR_ANYCRLF ||  // 如果不是换行符并且 BSR 规则为任意 CRLF
                 (fc != CHAR_VT && fc != CHAR_FF && fc != CHAR_NEL  // 且不是垂直制表符、换页符、下一行
#if PCRE2_CODE_UNIT_WIDTH != 8
                 && fc != 0x2028 && fc != 0x2029  // 且不是行分隔符、段分隔符
#endif
                 ))) break;  // 如果条件成立，跳出循环
              Feptr++;  // 指针向后移动一个位置
              }
            }
          break;

          case OP_NOT_HSPACE:  // 如果不是水平空白字符
          for (i = Lmin; i < Lmax; i++)  // 循环遍历指定范围
            {
            if (Feptr >= mb->end_subject)  // 如果指针超出了主题的末尾
              {
              SCHECK_PARTIAL();  // 检查是否为部分匹配
              break;  // 跳出循环
              }
            switch(*Feptr)  // 开始对当前字符进行判断
              {
              default: Feptr++; break;  // 默认情况下，指针向后移动一个位置
              HSPACE_BYTE_CASES:  // 如果是水平空白字符的情况
#if PCRE2_CODE_UNIT_WIDTH != 8
              HSPACE_MULTIBYTE_CASES:  // 如果是多字节字符的情况
#endif
              goto ENDLOOP00;  // 跳转到标签ENDLOOP00处
              }
            }
          ENDLOOP00:  // 标签ENDLOOP00
          break;

          case OP_HSPACE:  // 如果是水平空白字符
          for (i = Lmin; i < Lmax; i++)  // 循环遍历指定范围
            {
            if (Feptr >= mb->end_subject)  // 如果指针超出了主题的末尾
              {
              SCHECK_PARTIAL();  // 检查是否为部分匹配
              break;  // 跳出循环
              }
            switch(*Feptr)  // 开始对当前字符进行判断
              {
              default: goto ENDLOOP01;  // 默认情况下，跳转到标签ENDLOOP01处
              HSPACE_BYTE_CASES:  // 如果是水平空白字符的情况
#if PCRE2_CODE_UNIT_WIDTH != 8
              HSPACE_MULTIBYTE_CASES:  // 如果是多字节字符的情况
#endif
              Feptr++; break;  // 指针向后移动一个位置
              }
            }
          ENDLOOP01:  // 标签ENDLOOP01
          break;

          case OP_NOT_VSPACE:  // 如果不是垂直空白字符
          for (i = Lmin; i < Lmax; i++)  // 循环遍历指定范围
            {
            if (Feptr >= mb->end_subject)  // 如果指针超出了主题的末尾
              {
              SCHECK_PARTIAL();  // 检查是否为部分匹配
              break;  // 跳出循环
              }
            switch(*Feptr)  // 开始对当前字符进行判断
              {
              default: Feptr++; break;  // 默认情况下，指针向后移动一个位置
              VSPACE_BYTE_CASES:  // 如果是垂直空白字符的情况
#if PCRE2_CODE_UNIT_WIDTH != 8
              VSPACE_MULTIBYTE_CASES:  // 如果是多字节字符的情况
#endif
              goto ENDLOOP02;  // 跳转到标签ENDLOOP02处
              }
            }
          ENDLOOP02:  // 标签ENDLOOP02
          break;

          case OP_VSPACE:  // 如果是垂直空白字符
          for (i = Lmin; i < Lmax; i++)  // 循环遍历指定范围
            {
            if (Feptr >= mb->end_subject)  // 如果指针超出了主题的末尾
              {
              SCHECK_PARTIAL();  // 检查是否为部分匹配
              break;  // 跳出循环
              }
            switch(*Feptr)  // 开始对当前字符进行判断
              {
              default: goto ENDLOOP03;  // 默认情况下，跳转到标签ENDLOOP03处
              VSPACE_BYTE_CASES:  // 如果是垂直空白字符的情况
#if PCRE2_CODE_UNIT_WIDTH != 8
              VSPACE_MULTIBYTE_CASES:  // 如果是多字节字符的情况
    # 当前语句用于结束重复字符类型的处理
# 取消定义 Lstart_eptr
#undef Lstart_eptr
# 取消定义 Lmin
#undef Lmin
# 取消定义 Lmax
#undef Lmax
# 取消定义 Lctype
#undef Lctype
# 取消定义 Lpropvalue
#undef Lpropvalue

    /* ===================================================================== */
    /* 匹配反向引用，可能重复。查看项目的末尾，看看后面是否有重复信息。OP_REF 和
    OP_REFI 操作码用于对编号组或非重复命名组的引用。对于重复的命名组，使用OP_DNREF和
    OP_DNREFI。在这种情况下，我们必须扫描名称引用的组列表，并使用设置的第一个组。 */

#define Lmin      F->temp_32[0]
#define Lmax      F->temp_32[1]
#define Lcaseless F->temp_32[2]
#define Lstart    F->temp_sptr[0]
#define Loffset   F->temp_size

    case OP_DNREF:
    case OP_DNREFI:
    Lcaseless = (Fop == OP_DNREFI);
      {
      int count = GET2(Fecode, 1+IMM2_SIZE);
      PCRE2_SPTR slot = mb->name_table + GET2(Fecode, 1) * mb->name_entry_size;
      Fecode += 1 + 2*IMM2_SIZE;

      while (count-- > 0)
        {
        Loffset = (GET2(slot, 0) << 1) - 2;
        if (Loffset < Foffset_top && Fovector[Loffset] != PCRE2_UNSET) break;
        slot += mb->name_entry_size;
        }
      }
    跳转到 REF_REPEAT;

    case OP_REF:
    case OP_REFI:
    Lcaseless = (Fop == OP_REFI);
    Loffset = (GET2(Fecode, 1) << 1) - 2;
    Fecode += 1 + IMM2_SIZE;

    /* 设置重复，或处理非重复情况。最大值和最小值必须在堆栈帧中，但由于它们是短期值，我们
    使用临时字段。 */

    REF_REPEAT:
    # 根据 Fecode 指向的操作码进行不同的处理
    switch (*Fecode)
      {
      # 如果操作码是 OP_CRSTAR, OP_CRMINSTAR, OP_CRPLUS, OP_CRMINPLUS, OP_CRQUERY, OP_CRMINQUERY 中的一个，则执行以下操作
      case OP_CRSTAR:
      case OP_CRMINSTAR:
      case OP_CRPLUS:
      case OP_CRMINPLUS:
      case OP_CRQUERY:
      case OP_CRMINQUERY:
      # 计算重复次数的最小值和最大值
      fc = *Fecode++ - OP_CRSTAR;
      Lmin = rep_min[fc];
      Lmax = rep_max[fc];
      reptype = rep_typ[fc];
      break;

      # 如果操作码是 OP_CRRANGE 或 OP_CRMINRANGE，则执行以下操作
      case OP_CRRANGE:
      case OP_CRMINRANGE:
      # 获取重复次数的最小值和最大值
      Lmin = GET2(Fecode, 1);
      Lmax = GET2(Fecode, 1 + IMM2_SIZE);
      reptype = rep_typ[*Fecode - OP_CRSTAR];
      # 如果最大值为0，则将其设置为无穷大
      if (Lmax == 0) Lmax = UINT32_MAX;  /* Max 0 => infinity */
      Fecode += 1 + 2 * IMM2_SIZE;
      break;

      # 如果操作码不是以上任何一个，则执行以下操作
      default:                  /* No repeat follows */
        {
        # 调用 match_ref 函数进行匹配
        rrc = match_ref(Loffset, Lcaseless, F, mb, &length);
        # 如果匹配失败，则根据情况进行处理
        if (rrc != 0)
          {
          if (rrc > 0) Feptr = mb->end_subject;   /* Partial match */
          CHECK_PARTIAL();
          RRETURN(MATCH_NOMATCH);
          }
        }
      # 将 Feptr 增加 length，然后继续主循环
      Feptr += length;
      continue;              /* With the main loop */
      }

    # 处理重复的反向引用
    if (Loffset < Foffset_top && Fovector[Loffset] != PCRE2_UNSET)
      {
      # 如果组已设置且两个相邻的组匹配，则继续主循环
      if (Fovector[Loffset] == Fovector[Loffset + 1]) continue;
      }
    else  /* Group is not set */
      {
      # 如果最小值为0或者 PCRE2_MATCH_UNSET_BACKREF 标志被设置，则继续主循环
      if (Lmin == 0 || (mb->poptions & PCRE2_MATCH_UNSET_BACKREF) != 0)
        continue;
      }

    # 确保至少有最小次数的匹配
    # 循环匹配最小次数
    for (i = 1; i <= Lmin; i++)
      {
      # 定义匹配长度
      PCRE2_SIZE slength;
      # 调用 match_ref 函数进行匹配
      rrc = match_ref(Loffset, Lcaseless, F, mb, &slength);
      # 如果匹配不成功
      if (rrc != 0)
        {
        # 如果是部分匹配，更新 Feptr 指针位置
        if (rrc > 0) Feptr = mb->end_subject;   /* Partial match */
        # 检查是否允许部分匹配
        CHECK_PARTIAL();
        # 返回不匹配结果
        RRETURN(MATCH_NOMATCH);
        }
      # 更新 Feptr 指针位置
      Feptr += slength;
      }

    # 如果最小次数等于最大次数，继续下一次循环
    if (Lmin == Lmax) continue;

    # 如果是最小匹配，继续尝试并移动指针
    if (reptype == REPTYPE_MIN)
      {
      # 无限循环
      for (;;)
        {
        # 定义匹配长度
        PCRE2_SIZE slength;
        # 调用 RMATCH 函数进行匹配
        RMATCH(Fecode, RM20);
        # 如果匹配成功，返回匹配结果
        if (rrc != MATCH_NOMATCH) RRETURN(rrc);
        # 如果最小次数大于等于最大次数，返回不匹配结果
        if (Lmin++ >= Lmax) RRETURN(MATCH_NOMATCH);
        # 调用 match_ref 函数进行匹配
        rrc = match_ref(Loffset, Lcaseless, F, mb, &slength);
        # 如果匹配不成功
        if (rrc != 0)
          {
          # 如果是部分匹配，更新 Feptr 指针位置
          if (rrc > 0) Feptr = mb->end_subject;   /* Partial match */
          # 检查是否允许部分匹配
          CHECK_PARTIAL();
          # 返回不匹配结果
          RRETURN(MATCH_NOMATCH);
          }
        # 更新 Feptr 指针位置
        Feptr += slength;
        }
      # 控制永远不会到达这里
      }

    # 控制永远不会到达这里
/* 重置 Lcaseless、Lmin、Lmax、Lstart、Loffset 变量 */
#undef Lcaseless
#undef Lmin
#undef Lmax
#undef Lstart
#undef Loffset

/* ========================================================================= */
/*           Opcodes for the start of various parenthesized items            */
/* ========================================================================= */

    /* 在所有情况下，如果 RMATCH() 的结果是 MATCH_THEN，则检查 (*THEN) 是否在当前分支内，方法是将传回的 OP_THEN 的地址与分支的末尾进行比较。如果 (*THEN) 在当前分支内，并且该分支是两个或更多个备选项之一（它要么以 OP_ALT 开头，要么以 OP_ALT 结尾），则已经达到 THEN 动作的限制，因此将返回码转换为 NOMATCH，这将导致从现在开始发生正常的回溯。否则，THEN 会传回到外部备选项。这实现了 Perl 对括号分组的处理，其中不包含 | 的分组不会影响当前备选项，也就是说，(X) 不同于 (X|(*F))。 */

    /* ===================================================================== */
    /* BRAZERO、BRAMINZERO 和 SKIPZERO 出现在非占有式括号组的前面，表示它可能出现零次。它可能无限重复，也可能根本不出现 - 即它可以是()*或()?或甚至(){0}在模式中。具有固定上限重复限制的括号会被编译为若干个副本，可选的括号之前会有 BRAZERO 或 BRAMINZERO。具有可能零次重复的占有式组会在 BRAPOSZERO 前面。 */

#define Lnext_ecode F->temp_sptr[0]

    case OP_BRAZERO:
    Lnext_ecode = Fecode + 1;  // 设置 Lnext_ecode 为 Fecode + 1
    RMATCH(Lnext_ecode, RM9);  // 调用 RMATCH() 函数
    if (rrc != MATCH_NOMATCH) RRETURN(rrc);  // 如果 rrc 不等于 MATCH_NOMATCH，则返回 rrc
    do Lnext_ecode += GET(Lnext_ecode, 1); while (*Lnext_ecode == OP_ALT);  // 当 *Lnext_ecode 等于 OP_ALT 时，Lnext_ecode 加上 GET(Lnext_ecode, 1)
    Fecode = Lnext_ecode + 1 + LINK_SIZE;  // 设置 Fecode 为 Lnext_ecode + 1 + LINK_SIZE
    break;

    case OP_BRAMINZERO:
    Lnext_ecode = Fecode + 1;  // 设置 Lnext_ecode 为 Fecode + 1
    # 使用do-while循环，将Lnext_ecode增加1，直到*Lnext_ecode不等于OP_ALT
    do Lnext_ecode += GET(Lnext_ecode, 1); while (*Lnext_ecode == OP_ALT);
    # 调用RMATCH函数，传入Lnext_ecode + 1 + LINK_SIZE和RM10作为参数
    RMATCH(Lnext_ecode + 1 + LINK_SIZE, RM10);
    # 如果rrc不等于MATCH_NOMATCH，则返回rrc
    if (rrc != MATCH_NOMATCH) RRETURN(rrc);
    # Fecode增加1
    Fecode++;
    # 跳出当前循环
    break;
# 取消定义 Lnext_ecode

    # 处理 OP_SKIPZERO 操作码
    case OP_SKIPZERO:
    Fecode++;  # Fecode 指针向后移动一个位置
    do Fecode += GET(Fecode,1); while (*Fecode == OP_ALT);  # 循环直到遇到 OP_ALT 操作码
    Fecode += 1 + LINK_SIZE;  # Fecode 指针向后移动 1 + LINK_SIZE 个位置
    break;

    # 处理 OP_BRAPOSZERO 操作码
    /* ===================================================================== */
    /* 处理具有无限重复的占有式括号。这些括号的结尾将始终是 OP_KETRPOS，它返回 MATCH_KETRPOS 而不会在模式中继续。 */
#define Lframe_type    F->temp_32[0]
#define Lmatched_once  F->temp_32[1]
#define Lzero_allowed  F->temp_32[2]
#define Lstart_eptr    F->temp_sptr[0]
#define Lstart_group   F->temp_sptr[1]

    case OP_BRAPOSZERO:
    Lzero_allowed = TRUE;  # 允许零重复
    Fecode += 1;  # Fecode 指针向后移动一个位置
    if (*Fecode == OP_CBRAPOS || *Fecode == OP_SCBRAPOS)
      goto POSSESSIVE_CAPTURE;  # 跳转到 POSSESSIVE_CAPTURE 标签
    goto POSSESSIVE_NON_CAPTURE;  # 跳转到 POSSESSIVE_NON_CAPTURE 标签

    case OP_BRAPOS:
    case OP_SBRAPOS:
    Lzero_allowed = FALSE;  # 不允许零重复

    POSSESSIVE_NON_CAPTURE:
    Lframe_type = GF_NOCAPTURE;  # 记住帧类型为 GF_NOCAPTURE
    goto POSSESSIVE_GROUP;  # 跳转到 POSSESSIVE_GROUP 标签

    case OP_CBRAPOS:
    case OP_SCBRAPOS:
    Lzero_allowed = FALSE;  # 不允许零重复

    POSSESSIVE_CAPTURE:
    number = GET2(Fecode, 1+LINK_SIZE);  # 获取操作码后面的两个字节的值
    Lframe_type = GF_CAPTURE | number;  # 记住帧类型为 GF_CAPTURE 加上 number
    Lmatched_once = FALSE;  # 从未匹配过
    Lstart_group = Fecode;  # 记住这个组的起始位置
    for (;;)
      {
      Lstart_eptr = Feptr;               /* 保存当前位置作为组的起始位置 */
      group_frame_type = Lframe_type;    /* 保存当前帧类型 */
      RMATCH(Fecode + PRIV(OP_lengths)[*Fecode], RM8);  /* 调用 RMATCH 函数进行匹配 */
      if (rrc == MATCH_KETRPOS)          /* 如果匹配到了反向引用 */
        {
        Lmatched_once = TRUE;            /* 标记至少匹配了一次 */
        if (Feptr == Lstart_eptr)        /* 如果匹配为空，跳到结尾 */
          {
          do Fecode += GET(Fecode, 1); while (*Fecode == OP_ALT);  /* 跳过空匹配的分支 */
          break;
          }

        Fecode = Lstart_group;           /* 重新设置 Fecode 为组的起始位置 */
        continue;                        /* 继续下一轮匹配 */
        }

      /* 处理 THEN 操作符的注释见上方。*/

      if (rrc == MATCH_THEN)             /* 如果匹配到了 THEN 操作符 */
        {
        PCRE2_SPTR next_ecode = Fecode + GET(Fecode,1);  /* 获取下一个操作符的位置 */
        if (mb->verb_ecode_ptr < next_ecode &&
            (*Fecode == OP_ALT || *next_ecode == OP_ALT))  /* 如果下一个操作符是 ALT 或者当前操作符是 ALT */
          rrc = MATCH_NOMATCH;            /* 标记为不匹配 */
        }

      if (rrc != MATCH_NOMATCH) RRETURN(rrc);  /* 如果匹配成功或者出错，返回匹配结果 */
      Fecode += GET(Fecode, 1);          /* 移动 Fecode 到下一个操作符 */
      if (*Fecode != OP_ALT) break;      /* 如果下一个操作符不是 ALT，跳出循环 */
      }

    /* 如果匹配到了内容或者允许零次重复，则匹配成功 */
    if (Lmatched_once || Lzero_allowed)
      {
      Fecode += 1 + LINK_SIZE;           /* 移动 Fecode 到下一个操作符 */
      break;                             /* 跳出循环 */
      }

    RRETURN(MATCH_NOMATCH);              /* 返回不匹配结果 */
# 取消定义 Lmatched_once
#undef Lmatched_once
# 取消定义 Lzero_allowed
#undef Lzero_allowed
# 取消定义 Lframe_type
#undef Lframe_type
# 取消定义 Lstart_eptr
#undef Lstart_eptr
# 取消定义 Lstart_group
#undef Lstart_group

    /* ===================================================================== */
    /* 处理不能匹配空字符串的非捕获括号。当我们到达括号内的最后一个选择时，只要模式中没有 THEN，我们就可以通过不记录新的回溯点来进行优化。（理想情况下，我们应该在这个组内测试是否有 THEN，但我们没有这个信息。）但是，如果我们在最顶层，就不要这样做，因为这会使处理断言和一次性括号变得更加混乱，当没有东西可以回到时。 */

#define Lframe_type F->temp_32[0]     /* 为所有使用 GROUPLOOP 的设置 */
#define Lnext_branch F->temp_sptr[0]  /* 仅在处理 OP_BRA 时使用 */

    case OP_BRA:
    if (mb->hasthen || Frdepth == 0)
      {
      Lframe_type = 0;
      goto GROUPLOOP;
      }

    for (;;)
      {
      Lnext_branch = Fecode + GET(Fecode, 1);
      if (*Lnext_branch != OP_ALT) break;

      /* 这永远不是最终选择。我们不需要在这里测试 MATCH_THEN，因为当模式中有 THEN 时，不使用这段代码。 */

      RMATCH(Fecode + PRIV(OP_lengths)[*Fecode], RM1);
      if (rrc != MATCH_NOMATCH) RRETURN(rrc);
      Fecode = Lnext_branch;
      }

    /* 到达最终选择的开始。在这个级别继续。 */

    Fecode += PRIV(OP_lengths)[*Fecode];
    break;

#undef Lnext_branch

    /* ===================================================================== */
    /* 处理捕获括号，除了那些具有无限重复的占有性捕获括号。 */

    case OP_CBRA:
    case OP_SCBRA:
    Lframe_type = GF_CAPTURE | GET2(Fecode, 1+LINK_SIZE);
    goto GROUPLOOP;

    /* ===================================================================== */
    /* 原子组和可以匹配空字符串的非捕获括号
    /* 必须记录回溯点并设置一个链接的帧。*/

    case OP_ONCE:
    case OP_SCRIPT_RUN:
    case OP_SBRA:
    Lframe_type = GF_NOCAPTURE | Fop;

    GROUPLOOP:
    for (;;)
      {
      group_frame_type = Lframe_type;
      RMATCH(Fecode + PRIV(OP_lengths)[*Fecode], RM2);
      if (rrc == MATCH_THEN)
        {
        PCRE2_SPTR next_ecode = Fecode + GET(Fecode,1);
        if (mb->verb_ecode_ptr < next_ecode &&
            (*Fecode == OP_ALT || *next_ecode == OP_ALT))
          rrc = MATCH_NOMATCH;
        }
      if (rrc != MATCH_NOMATCH) RRETURN(rrc);
      Fecode += GET(Fecode, 1);
      if (*Fecode != OP_ALT) RRETURN(MATCH_NOMATCH);
      }
    /* 控制永远不会到达这里。*/
#undef Lframe_type

/* ===================================================================== */
/* 递归要么匹配当前的正则表达式，要么匹配某个子表达式。偏移数据是从整个模式的起始括号到开始的偏移量。（这样可以处理重复的子模式。） */
#define Lframe_type F->temp_32[0]
#define Lstart_branch F->temp_sptr[0]

case OP_RECURSE:
    // bracode 是指向起始代码的偏移量
    bracode = mb->start_code + GET(Fecode, 1);
    // 如果 bracode 等于起始代码，number 为 0，否则从 bracode 的偏移量处获取 number
    number = (bracode == mb->start_code)? 0 : GET2(bracode, 1 + LINK_SIZE);

    /* 如果我们已经在一个递归中，检查是否重复相同的递归而不推进主题指针。这应该捕获复杂的相互递归。（一些简单的情况在编译时就被捕获了。） */
    if (Fcurrent_recurse != RECURSE_UNSET)
    {
        offset = Flast_group_offset;
        while (offset != PCRE2_UNSET)
        {
            N = (heapframe *)((char *)match_data->heapframes + offset);
            P = (heapframe *)((char *)N - frame_size);
            if (N->group_frame_type == (GF_RECURSE | number))
            {
                if (Feptr == P->eptr) return PCRE2_ERROR_RECURSELOOP;
                break;
            }
            offset = P->last_group_offset;
        }
    }

    /* 现在逐个分支运行递归。 */
    Lstart_branch = bracode;
    Lframe_type = GF_RECURSE | number;
    # 无限循环，直到条件不满足
    for (;;)
      {
      # 定义指向下一个执行码的指针
      PCRE2_SPTR next_ecode;

      # 将当前组帧类型赋值给 group_frame_type
      group_frame_type = Lframe_type;
      # 调用 RMATCH 函数，传入起始分支位置和对应的执行码，执行匹配操作
      RMATCH(Lstart_branch + PRIV(OP_lengths)[*Lstart_branch], RM11);
      # 获取下一个执行码的位置
      next_ecode = Lstart_branch + GET(Lstart_branch,1);

      /* 处理回溯动词，这些动词定义在一个范围内，可以轻松测试。
      PCRE 不允许 THEN、SKIP、PRUNE 或 COMMIT 超出递归范围；它们会导致整个递归的 NOMATCH。

      当其中一个动词触发时，记录当前递归组号。
      如果它与我们正在处理的递归匹配，则该动词发生在递归内部，我们必须处理它。
      否则它必须发生在递归完成后，因此必须被传递回去。参见上面关于处理 THEN 的注释。 */
      if (rrc >= MATCH_BACKTRACK_MIN && rrc <= MATCH_BACKTRACK_MAX &&
          mb->verb_current_recurse == (Lframe_type ^ GF_RECURSE))
        {
        if (rrc == MATCH_THEN && mb->verb_ecode_ptr < next_ecode &&
            (*Lstart_branch == OP_ALT || *next_ecode == OP_ALT))
          rrc = MATCH_NOMATCH;
        else RRETURN(MATCH_NOMATCH);
        }

      /* 注意，在递归中继续执行 (*ACCEPT) 在 OP_ACCEPT 代码中处理。这里不需要做任何操作。 */
      if (rrc != MATCH_NOMATCH) RRETURN(rrc);
      # 更新起始分支位置为下一个执行码的位置
      Lstart_branch = next_ecode;
      # 如果下一个执行码不是 OP_ALT，则返回 NOMATCH
      if (*Lstart_branch != OP_ALT) RRETURN(MATCH_NOMATCH);
      }
    /* 永远不会执行到这里。 */
# 取消之前定义的 Lframe_type 和 Lstart_branch
#undef Lframe_type
#undef Lstart_branch

    /* ===================================================================== */
    /* Positive assertions are like other groups except that PCRE doesn't allow
    the effect of (*THEN) to escape beyond an assertion; it is therefore
    treated as NOMATCH. (*ACCEPT) is treated as successful assertion, with its
    captures and mark retained. Any other return is an error. */

# 重新定义 Lframe_type 为 F->temp_32[0]
#define Lframe_type  F->temp_32[0]

    case OP_ASSERT:
    case OP_ASSERTBACK:
    case OP_ASSERT_NA:
    case OP_ASSERTBACK_NA:
    # 设置 Lframe_type 为 GF_NOCAPTURE | Fop
    Lframe_type = GF_NOCAPTURE | Fop;
    # 无限循环
    for (;;)
      {
      # 将 Lframe_type 赋值给 group_frame_type
      group_frame_type = Lframe_type;
      # 调用 RMATCH 函数
      RMATCH(Fecode + PRIV(OP_lengths)[*Fecode], RM3);
      # 如果返回值为 MATCH_ACCEPT
      if (rrc == MATCH_ACCEPT)
        {
        # 复制 assert_accept_frame 中的数据到 Fovector
        memcpy(Fovector,
              (char *)assert_accept_frame + offsetof(heapframe, ovector),
              assert_accept_frame->offset_top * sizeof(PCRE2_SIZE));
        # 设置 Foffset_top 为 assert_accept_frame 中的 offset_top
        Foffset_top = assert_accept_frame->offset_top;
        # 设置 Fmark 为 assert_accept_frame 中的 mark
        Fmark = assert_accept_frame->mark;
        # 跳出循环
        break;
        }
      # 如果返回值不是 MATCH_NOMATCH 且不是 MATCH_THEN，则返回 rrc
      if (rrc != MATCH_NOMATCH && rrc != MATCH_THEN) RRETURN(rrc);
      # Fecode 增加 GET(Fecode, 1) 的值
      Fecode += GET(Fecode, 1);
      # 如果 *Fecode 不是 OP_ALT，则返回 MATCH_NOMATCH
      if (*Fecode != OP_ALT) RRETURN(MATCH_NOMATCH);
      }

    # 循环，直到 *Fecode 不是 OP_ALT
    do Fecode += GET(Fecode, 1); while (*Fecode == OP_ALT);
    # Fecode 增加 1 + LINK_SIZE 的值
    Fecode += 1 + LINK_SIZE;
    # 跳出循环
    break;

# 取消之前定义的 Lframe_type

#undef Lframe_type

    /* ===================================================================== */
    /* Handle negative assertions. Loop for each non-matching branch as for
    positive assertions. */

# 重新定义 Lframe_type 为 F->temp_32[0]
#define Lframe_type  F->temp_32[0]

    case OP_ASSERT_NOT:
    case OP_ASSERTBACK_NOT:
    # 设置 Lframe_type 为 GF_NOCAPTURE | Fop
    Lframe_type  = GF_NOCAPTURE | Fop
    # 无限循环，用于匹配正则表达式
    for (;;)
      {
      # 将当前帧类型赋值给组帧类型
      group_frame_type = Lframe_type;
      # 使用 RM4 匹配 Fecode + PRIV(OP_lengths)[*Fecode]，并将结果赋值给 rrc
      RMATCH(Fecode + PRIV(OP_lengths)[*Fecode], RM4);
      # 根据 rrc 的值进行不同的操作
      switch(rrc)
        {
        case MATCH_ACCEPT:   /* 断言匹配，因此失败 */
        case MATCH_MATCH:
        # 返回不匹配
        RRETURN (MATCH_NOMATCH);

        case MATCH_NOMATCH:  /* 分支失败，尝试下一个分支 */
        case MATCH_THEN:
        # 将 Fecode 移动到下一个分支的位置
        Fecode += GET(Fecode, 1);
        # 如果下一个位置不是 OP_ALT，则跳转到 ASSERT_NOT_FAILED
        if (*Fecode != OP_ALT) goto ASSERT_NOT_FAILED;
        break;

        case MATCH_COMMIT:   /* 断言被强制失败，因此继续 */
        case MATCH_SKIP:
        case MATCH_PRUNE:
        # 当前分支失败，跳过到下一个分支
        do Fecode += GET(Fecode, 1); while (*Fecode == OP_ALT);
        # 跳转到 ASSERT_NOT_FAILED
        goto ASSERT_NOT_FAILED;

        default:             /* 返回其他任何值 */
        # 返回 rrc
        RRETURN(rrc);
        }
      }

    # 所有分支都不匹配，或者在最后一个分支有回溯到 (*COMMIT), (*SKIP), (*PRUNE), 或 (*THEN)。对于负断言来说，这是成功的，因此继续。
    ASSERT_NOT_FAILED:
    # 将 Fecode 移动到下一个位置
    Fecode += 1 + LINK_SIZE;
    # 跳出循环
    break;
# 取消定义 Lframe_type

    /* ===================================================================== */
    /* 调用项调用外部函数（如果提供了），传递到目前为止的匹配详情。这主要用于调试，尽管该函数能够强制失败。 */

    case OP_CALLOUT:
    case OP_CALLOUT_STR:
    rrc = do_callout(F, mb, &length);
    if (rrc > 0) RRETURN(MATCH_NOMATCH);
    if (rrc < 0) RRETURN(rrc);
    Fecode += length;
    break;

    /* ===================================================================== */
    /* 条件组：编译检查不超过两个分支。如果条件为假，跳过第一个分支将使我们超过项目的末尾，如果只有一个分支，但这正是我们想要的。 */

    case OP_COND:
    case OP_SCOND:

    /* 当条件为假时，变量 Flength 将被添加到 Fecode，以到达第二个分支。将其设置为 ALT 或 KET 的偏移量，然后递增 Fecode 可实现此效果。但是，如果第二个分支不存在，我们必须指向 KET，以便正确处理组的末尾。现在，Fecode 指向条件或调用。 */

    Flength = GET(Fecode, 1);    /* 第二个分支的偏移量 */
    if (Fecode[Flength] != OP_ALT) Flength -= 1 + LINK_SIZE;
    Fecode += 1 + LINK_SIZE;     /* 从这个操作码开始 */

    /* 由于编译期间自动调用的方式，调用项被插入在 OP_COND 和断言条件之间。这样的调用也可以手动插入。 */
    # 如果当前指令是OP_CALLOUT或者OP_CALLOUT_STR
    if (*Fecode == OP_CALLOUT || *Fecode == OP_CALLOUT_STR)
      {
      # 调用外部函数do_callout，获取返回值rrc和长度length
      rrc = do_callout(F, mb, &length);
      # 如果返回值大于0，表示不匹配，返回MATCH_NOMATCH
      if (rrc > 0) RRETURN(MATCH_NOMATCH);
      # 如果返回值小于0，直接返回rrc
      if (rrc < 0) RRETURN(rrc);

      /* Advance Fecode past the callout, so it now points to the condition. We
      must adjust Flength so that the value of Fecode+Flength is unchanged. */

      # 将Fecode指针向前移动length个位置，指向条件部分
      Fecode += length;
      # 调整Flength的值，保持Fecode+Flength的值不变
      Flength -= length;
      }

    # 设置条件为FALSE
    condition = FALSE;
    # 根据 Fecode 中的操作码进行不同的操作
    switch(*Fecode)
      {
      # 如果操作码为 OP_RREF，进行组递归测试
      case OP_RREF:                  /* Group recursion test */
      # 如果当前递归值不是未设置状态
      if (Fcurrent_recurse != RECURSE_UNSET)
        {
        # 从 Fecode 中获取一个 2 字节的数字
        number = GET2(Fecode, 1);
        # 判断条件是否为真，条件为数字等于 RREF_ANY 或者数字等于当前递归值
        condition = (number == RREF_ANY || number == Fcurrent_recurse);
        }
      break;

      # 如果操作码为 OP_DNRREF，进行重复命名组递归测试
      case OP_DNRREF:       /* Duplicate named group recursion test */
      # 如果当前递归值不是未设置状态
      if (Fcurrent_recurse != RECURSE_UNSET)
        {
        # 从 Fecode 中获取一个 2 字节的数字，以及一个立即数大小的值
        int count = GET2(Fecode, 1 + IMM2_SIZE);
        # 从 mb->name_table 中获取一个指针，指向 Fecode 中的一个值乘以 mb->name_entry_size
        PCRE2_SPTR slot = mb->name_table + GET2(Fecode, 1) * mb->name_entry_size;
        # 循环 count 次
        while (count-- > 0)
          {
          # 从 slot 中获取一个 2 字节的数字
          number = GET2(slot, 0);
          # 判断条件是否为真，条件为数字等于当前递归值
          condition = number == Fcurrent_recurse;
          # 如果条件为真，跳出循环
          if (condition) break;
          # slot 指针向后移动 mb->name_entry_size
          slot += mb->name_entry_size;
          }
        }
      break;

      # 如果操作码为 OP_CREF，进行编号组使用测试
      case OP_CREF:                         /* Numbered group used test */
      # 计算偏移量，从 Fecode 中获取一个 2 字节的数字，左移 1 位，再减去 2
      offset = (GET2(Fecode, 1) << 1) - 2;  /* Doubled ref number */
      # 判断条件是否为真，条件为偏移量小于 Foffset_top 并且 Fovector 中对应位置不等于 PCRE2_UNSET
      condition = offset < Foffset_top && Fovector[offset] != PCRE2_UNSET;
      break;

      # 如果操作码为 OP_DNCREF，进行重复命名组使用测试
      case OP_DNCREF:      /* Duplicate named group used test */
        {
        # 从 Fecode 中获取一个 2 字节的数字，以及一个立即数大小的值
        int count = GET2(Fecode, 1 + IMM2_SIZE);
        # 从 mb->name_table 中获取一个指针，指向 Fecode 中的一个值乘以 mb->name_entry_size
        PCRE2_SPTR slot = mb->name_table + GET2(Fecode, 1) * mb->name_entry_size;
        # 循环 count 次
        while (count-- > 0)
          {
          # 从 slot 中获取一个 2 字节的数字
          offset = (GET2(slot, 0) << 1) - 2;
          # 判断条件是否为真，条件为偏移量小于 Foffset_top 并且 Fovector 中对应位置不等于 PCRE2_UNSET
          condition = offset < Foffset_top && Fovector[offset] != PCRE2_UNSET;
          # 如果条件为真，跳出循环
          if (condition) break;
          # slot 指针向后移动 mb->name_entry_size
          slot += mb->name_entry_size;
          }
        }
      break;

      # 如果操作码为 OP_FALSE 或者 OP_FAIL，不进行任何操作
      case OP_FALSE:
      case OP_FAIL:   /* The assertion (?!) becomes OP_FAIL */
      break;

      # 如果操作码为 OP_TRUE，将条件设置为真
      case OP_TRUE:
      condition = TRUE;
      break;

      # 如果条件为断言，则运行类似断言代码的代码
      /* The condition is an assertion. Run code similar to the assertion code
      above. */
#define Lpositive      F->temp_32[0]  // 定义宏 Lpositive，表示正向断言
#define Lstart_branch  F->temp_sptr[0]  // 定义宏 Lstart_branch，表示断言的起始位置

      default:  // 默认情况
      Lpositive = (*Fecode == OP_ASSERT || *Fecode == OP_ASSERTBACK);  // 判断当前指令是否为断言，更新 Lpositive 的值
      Lstart_branch = Fecode;  // 将当前指令位置赋值给 Lstart_branch

      for (;;)  // 无限循环
        {
        group_frame_type = GF_CONDASSERT | *Fecode;  // 设置组帧类型为条件断言
        RMATCH(Lstart_branch + PRIV(OP_lengths)[*Lstart_branch], RM5);  // 调用 RMATCH 函数进行匹配

        switch(rrc)  // 根据 rrc 的值进行不同的操作
          {
          case MATCH_ACCEPT:  // 匹配成功，保存捕获内容
          memcpy(Fovector,
                (char *)assert_accept_frame + offsetof(heapframe, ovector),
                assert_accept_frame->offset_top * sizeof(PCRE2_SIZE));
          Foffset_top = assert_accept_frame->offset_top;

          /* Fall through */
          /* 在匹配成功的情况下，捕获内容已经放入当前帧中 */

          case MATCH_MATCH:  // 匹配成功
          condition = Lpositive;   // 正向断言条件为真
          break;

          /* PCRE 不允许 (*THEN) 的效果逃逸出断言；因此它总是被视为不匹配。 */

          case MATCH_NOMATCH:  // 不匹配
          case MATCH_THEN:  // (*THEN) 指令
          Lstart_branch += GET(Lstart_branch, 1);  // 更新起始位置到下一个分支
          if (*Lstart_branch == OP_ALT) continue;  // 尝试下一个分支
          condition = !Lpositive;  // 负向断言条件为真
          break;

          /* 这些指令强制不匹配，不需要检查其他分支。 */

          case MATCH_COMMIT:  // 提交指令
          case MATCH_SKIP:  // 跳过指令
          case MATCH_PRUNE:  // 放弃指令
          condition = !Lpositive;  // 负向断言条件为真
          break;

          default:
          RRETURN(rrc);  // 返回 rrc 的值
          }
        break;  // 跳出分支循环
        }

      /* 如果条件为真，找到断言的结束位置，以便在向前移动时到达第一个分支的起始位置。 */

      if (condition)  // 如果条件为真
        {
        do Fecode += GET(Fecode, 1); while (*Fecode == OP_ALT);  // 找到断言的结束位置
        }
      break;  // 断言条件结束
      }

#undef Lpositive  // 取消定义宏 Lpositive
#undef Lstart_branch  // 取消定义宏 Lstart_branch
    /* 根据条件选择分支。 */

    Fecode += condition? PRIV(OP_lengths)[*Fecode] : Flength;

    /* 如果操作码是 OP_SCOND，意味着我们处于一个可能匹配空字符串的重复条件组中。因此，我们必须下降一个级别，以便记住开始位置进行检查。对于 OP_COND，我们可以在当前级别继续。 */

    if (Fop == OP_SCOND)
      {
      group_frame_type  = GF_NOCAPTURE | Fop;
      RMATCH(Fecode, RM35);
      RRETURN(rrc);
      }
    break;
/* ========================================================================= */
/*                  End of start of parenthesis opcodes                      */
/* ========================================================================= */


    /* ===================================================================== */
    /* Move the subject pointer back. This occurs only at the start of each
    branch of a lookbehind assertion. If we are too close to the start to move
    back, fail. When working with UTF-8 we move back a number of characters,
    not bytes. */

    case OP_REVERSE:
    // 获取操作数
    number = GET(Fecode, 1);
#ifdef SUPPORT_UNICODE
    // 如果是 UTF-8 模式，向前移动指针一定数量的字符
    if (utf)
      {
      while (number-- > 0)
        {
        // 如果指针已经到达起始位置，匹配失败
        if (Feptr <= mb->check_subject) RRETURN(MATCH_NOMATCH);
        // 向前移动指针
        Feptr--;
        // 向前移动一个字符
        BACKCHAR(Feptr);
        }
      }
    else
#endif

    /* No UTF-8 support, or not in UTF-8 mode: count is code unit count */

      {
      // 如果不是 UTF-8 模式，或者不在 UTF-8 模式下：向前移动指定数量的代码单元
      if ((ptrdiff_t)number > Feptr - mb->start_subject) RRETURN(MATCH_NOMATCH);
      Feptr -= number;
      }

    /* Save the earliest consulted character, then skip to next opcode */

    // 保存最早被检查的字符，然后跳到下一个操作码
    if (Feptr < mb->start_used_ptr) mb->start_used_ptr = Feptr;
    Fecode += 1 + LINK_SIZE;
    break;


    /* ===================================================================== */
    /* An alternation is the end of a branch; scan along to find the end of the
    bracketed group. */

    case OP_ALT:
    // 交替操作是分支的结束；扫描以找到括号组的结束
    do Fecode += GET(Fecode,1); while (*Fecode == OP_ALT);
    break;


    /* ===================================================================== */
    /* The end of a parenthesized group. For all but OP_BRA and OP_COND, the
    starting frame was added to the chained frames in order to remember the
    starting subject position for the group. */

    case OP_KET:
    case OP_KETRMIN:
    case OP_KETRMAX:
    case OP_KETRPOS:

    // 获取括号组的起始位置
    bracode = Fecode - GET(Fecode, 1);

    /* Point N to the frame at the start of the most recent group.
    # 记住组的起始指针
    Remember the subject pointer at the start of the group. */

    # 如果当前指令不是OP_BRA和OP_COND
    if (*bracode != OP_BRA && *bracode != OP_COND)
      {
      # 设置N为当前组的指针
      N = (heapframe *)((char *)match_data->heapframes + Flast_group_offset);
      # 设置P为上一个组的指针
      P = (heapframe *)((char *)N - frame_size);
      # 更新上一个组的偏移量
      Flast_group_offset = P->last_group_offset;
#ifdef DEBUG_SHOW_RMATCH
      // 如果定义了 DEBUG_SHOW_RMATCH 宏，则输出调试信息
      fprintf(stderr, "++ KET for frame=%d type=%x prev char offset=%lu\n",
        N->rdepth, N->group_frame_type,
        (char *)P->eptr - (char *)mb->start_subject);
#endif

      /* 如果我们处于一个条件断言的结尾，返回一个匹配，丢弃任何中间的回溯点。
      将标记设置和捕获拷贝回 N 之前的帧中，这样它们在返回时就被设置了。
      对所有断言都这样做，包括正向和负向断言，似乎与 Perl 的行为相匹配。 */
      if (GF_IDMASK(N->group_frame_type) == GF_CONDASSERT)
        {
        // 拷贝标记设置和捕获到帧中
        memcpy((char *)P + offsetof(heapframe, ovector), Fovector,
          Foffset_top * sizeof(PCRE2_SIZE));
        P->offset_top = Foffset_top;
        P->mark = Fmark;
        // 计算 Fback_frame 的值
        Fback_frame = (char *)F - (char *)P;
        // 返回匹配
        RRETURN(MATCH_MATCH);
        }
      }
    else P = NULL;   /* 表示未记录起始帧 */

    /* OP_KETRPOS 是一个占有型重复的闭合括号。记住当前位置，并返回 MATCH_KETRPOS。
    这样可以从外层逐个重复。这必须在空字符串测试之前进行 - 在这种情况下，该测试在外层进行。 */
    if (*Fecode == OP_KETRPOS)
      {
      // 拷贝当前位置
      memcpy((char *)P + offsetof(heapframe, eptr),
             (char *)F + offsetof(heapframe, eptr),
             frame_copy_size);
      // 返回 MATCH_KETRPOS
      RRETURN(MATCH_KETRPOS);
      }

    /* 处理不同类型的闭合括号。非重复的闭合括号不需要特殊操作，只需在当前级别继续。
    对于重复的闭合括号，如果组没有匹配任何字符，则需要强制中断无限循环。
    否则，重复的闭合括号尝试模式的其余部分，或者从前面的括号重新开始，顺序适当。 */
    # 如果当前操作码不是 OP_KET 且（P 为空或 Feptr 不等于 P 的 eptr）
    if (Fop != OP_KET && (P == NULL || Feptr != P->eptr))
      {
      # 如果当前操作码是 OP_KETRMIN
      if (Fop == OP_KETRMIN)
        {
        # 调用 RMATCH 函数，传入参数 Fecode + 1 + LINK_SIZE, RM6
        RMATCH(Fecode + 1 + LINK_SIZE, RM6);
        # 如果 rrc 不等于 MATCH_NOMATCH，则返回 rrc
        if (rrc != MATCH_NOMATCH) RRETURN(rrc);
        # Fecode 减去 GET(Fecode, 1)
        Fecode -= GET(Fecode, 1);
        # 跳出循环，结束 ket 处理
        break;   /* End of ket processing */
        }

      # 重复最大次数（KETRMAX）
      # 调用 RMATCH 函数，传入参数 bracode, RM7
      RMATCH(bracode, RM7);
      # 如果 rrc 不等于 MATCH_NOMATCH，则返回 rrc
      if (rrc != MATCH_NOMATCH) RRETURN(rrc);
      }

    # 在非重复的 ket，或匹配空字符串后，或重复最大次数后，继续在当前级别进行
    Fecode += 1 + LINK_SIZE;
    break;


    /* ===================================================================== */
    /* Start and end of line assertions, not multiline mode. */

    # 当操作码为 OP_CIRC 时，即行的开始，除非设置了 PCRE2_NOTBOL
    case OP_CIRC:   /* Start of line, unless PCRE2_NOTBOL is set. */
    # 如果 Feptr 不等于 mb 的 start_subject 或者（mb 的 moptions 与 PCRE2_NOTBOL 的按位与结果不等于 0）
    if (Feptr != mb->start_subject || (mb->moptions & PCRE2_NOTBOL) != 0)
      # 返回 MATCH_NOMATCH
      RRETURN(MATCH_NOMATCH);
    # Fecode 加一
    Fecode++;
    break;

    # 当操作码为 OP_SOD 时，即无条件的主题开始
    case OP_SOD:    /* Unconditional start of subject */
    # 如果 Feptr 不等于 mb 的 start_subject
    if (Feptr != mb->start_subject) RRETURN(MATCH_NOMATCH);
    # Fecode 加一
    Fecode++;
    break;

    # 当 PCRE2_NOTEOL 未设置时，在主题结束前断言，或终止的换行符，除非设置了 PCRE2_DOLLAR_ENDONLY
    case OP_DOLL:
    # 如果（mb 的 moptions 与 PCRE2_NOTEOL 的按位与结果不等于 0）
    if ((mb->moptions & PCRE2_NOTEOL) != 0) RRETURN(MATCH_NOMATCH);
    # 如果（mb 的 poptions 与 PCRE2_DOLLAR_ENDONLY 的按位与结果等于 0），则跳转到 ASSERT_NL_OR_EOS
    if ((mb->poptions & PCRE2_DOLLAR_ENDONLY) == 0) goto ASSERT_NL_OR_EOS;

    # 无条件的主题结束断言（\z）
    case OP_EOD:
    # 如果 Feptr 小于 mb 的 end_subject，则返回 MATCH_NOMATCH
    if (Feptr < mb->end_subject) RRETURN(MATCH_NOMATCH);
    # 如果 mb 的 partial 不为 0
    if (mb->partial != 0)
      {
      # 设置 mb 的 hitend 为 TRUE
      mb->hitend = TRUE;
      # 如果 mb 的 partial 大于 1，则返回 PCRE2_ERROR_PARTIAL
      if (mb->partial > 1) return PCRE2_ERROR_PARTIAL;
      }
    # Fecode 加一
    Fecode++;
    break;

    # 主题结束或结束的 \n 断言（\Z）
    case OP_EODN:
    ASSERT_NL_OR_EOS:
    # 如果当前位置不是字符串的结尾，并且不是换行符，或者不是倒数第二个位置是换行符
    if (Feptr < mb->end_subject &&
        (!IS_NEWLINE(Feptr) || Feptr != mb->end_subject - mb->nllen))
      {
      # 如果设置了部分匹配，并且当前位置加1等于字符串的结尾，并且换行符类型是固定的，并且换行符长度是2，并且当前位置的字符等于换行符的第一个字符
      if (mb->partial != 0 &&
          Feptr + 1 >= mb->end_subject &&
          NLBLOCK->nltype == NLTYPE_FIXED &&
          NLBLOCK->nllen == 2 &&
          UCHAR21TEST(Feptr) == NLBLOCK->nl[0])
        {
        # 设置匹配到结尾标志，并返回部分匹配错误
        mb->hitend = TRUE;
        if (mb->partial > 1) return PCRE2_ERROR_PARTIAL;
        }
      # 返回不匹配
      RRETURN(MATCH_NOMATCH);
      }

    /* Either at end of string or \n before end. */

    # 如果设置了部分匹配
    if (mb->partial != 0)
      {
      # 设置匹配到结尾标志，并返回部分匹配错误
      mb->hitend = TRUE;
      if (mb->partial > 1) return PCRE2_ERROR_PARTIAL;
      }
    # 移动到下一个字符
    Fecode++;
    # 跳出循环
    break;


    /* ===================================================================== */
    /* Start and end of line assertions, multiline mode. */

    /* Start of subject unless notbol, or after any newline except for one at
    the very end, unless PCRE2_ALT_CIRCUMFLEX is set. */

    # 如果设置了多行模式
    case OP_CIRCM:
    # 如果设置了不是行首匹配，并且当前位置是字符串的开头
    if ((mb->moptions & PCRE2_NOTBOL) != 0 && Feptr == mb->start_subject)
      RRETURN(MATCH_NOMATCH);
    # 如果当前位置不是字符串的开头，并且（当前位置是字符串的结尾，并且没有设置PCRE2_ALT_CIRCUMFLEX，或者当前位置不是换行符）
    if (Feptr != mb->start_subject &&
        ((Feptr == mb->end_subject &&
           (mb->poptions & PCRE2_ALT_CIRCUMFLEX) == 0) ||
         !WAS_NEWLINE(Feptr)))
      RRETURN(MATCH_NOMATCH);
    # 移动到下一个字符
    Fecode++;
    # 跳出循环
    break;

    /* Assert before any newline, or before end of subject unless noteol is
    set. */

    # 如果设置了多行模式
    case OP_DOLLM:
    # 如果当前位置不是字符串的结尾
    if (Feptr < mb->end_subject)
      {
      # 如果当前位置不是换行符
      if (!IS_NEWLINE(Feptr))
        {
        # 如果设置了部分匹配，并且当前位置加1等于字符串的结尾，并且换行符类型是固定的，并且换行符长度是2，并且当前位置的字符等于换行符的第一个字符
        if (mb->partial != 0 &&
            Feptr + 1 >= mb->end_subject &&
            NLBLOCK->nltype == NLTYPE_FIXED &&
            NLBLOCK->nllen == 2 &&
            UCHAR21TEST(Feptr) == NLBLOCK->nl[0])
          {
          # 设置匹配到结尾标志，并返回部分匹配错误
          mb->hitend = TRUE;
          if (mb->partial > 1) return PCRE2_ERROR_PARTIAL;
          }
        # 返回不匹配
        RRETURN(MATCH_NOMATCH);
        }
      }
    else
      {
      # 如果设置了PCRE2_NOTEOL，则返回不匹配
      if ((mb->moptions & PCRE2_NOTEOL) != 0) RRETURN(MATCH_NOMATCH);
      # 检查部分匹配
      SCHECK_PARTIAL();
      }
    # 移动到下一个字符
    Fecode++;
    # 跳出循环
    break;
    # 开始匹配断言部分

    # 如果当前位置不是匹配起始位置，则返回不匹配
    case OP_SOM:
    if (Feptr != mb->start_subject + mb->start_offset) RRETURN(MATCH_NOMATCH)
    Fecode++
    break


    # 重置匹配起始位置

    case OP_SET_SOM:
    Fstart_match = Feptr
    Fecode++
    break


    # 单词边界断言。查找前一个和当前字符是否为“单词”字符。在 UTF 模式下需要更多的工作。
    # 当 PCRE2_UCP 未设置时，假定大于 255 的字符为“非单词”字符。
    # 当设置了 PCRE2_UCP 时，即使不在 UTF 模式下，也使用 Unicode 属性（如果可用）。
    # 记住最早和最晚被查询的字符。

    case OP_NOT_WORD_BOUNDARY:
    case OP_WORD_BOUNDARY:
    if (Feptr == mb->check_subject) prev_is_word = FALSE; else
      {
      PCRE2_SPTR lastptr = Feptr - 1;
#ifdef SUPPORT_UNICODE
      # 如果支持 Unicode，则进行以下操作
      if (utf)
        {
        # 如果使用 UTF 编码，则回退一个字符
        BACKCHAR(lastptr);
        # 获取回退后的字符
        GETCHAR(fc, lastptr);
        }
      else
#endif  /* SUPPORT_UNICODE */
      # 如果不支持 Unicode，则直接获取当前字符
      fc = *lastptr;
      # 如果当前指针小于已使用的起始指针，则更新已使用的起始指针
      if (lastptr < mb->start_used_ptr) mb->start_used_ptr = lastptr;
#ifdef SUPPORT_UNICODE
      # 如果支持 Unicode 属性，则进行以下操作
      if ((mb->poptions & PCRE2_UCP) != 0)
        {
        # 如果当前字符是下划线，则设置前一个字符为单词字符；否则根据 Unicode 类别判断前一个字符是否为单词字符
        if (fc == '_') prev_is_word = TRUE; else
          {
          int cat = UCD_CATEGORY(fc);
          prev_is_word = (cat == ucp_L || cat == ucp_N);
          }
        }
      else
#endif  /* SUPPORT_UNICODE */
      # 如果不支持 Unicode 属性，则根据字符属性判断前一个字符是否为单词字符
      prev_is_word = CHMAX_255(fc) && (mb->ctypes[fc] & ctype_word) != 0;
      }

    /* Get status of next character */

    # 获取下一个字符的状态
    if (Feptr >= mb->end_subject)
      {
      # 如果已经到达主题的末尾，则进行部分匹配检查，当前字符不是单词字符
      SCHECK_PARTIAL();
      cur_is_word = FALSE;
      }
    else
      {
      # 获取下一个字符的指针
      PCRE2_SPTR nextptr = Feptr + 1;
#ifdef SUPPORT_UNICODE
      # 如果支持 Unicode，则进行以下操作
      if (utf)
        {
        # 如果使用 UTF 编码，则向前测试一个字符，并获取当前字符
        FORWARDCHARTEST(nextptr, mb->end_subject);
        GETCHAR(fc, Feptr);
        }
      else
#endif  /* SUPPORT_UNICODE */
      # 如果不支持 Unicode，则直接获取当前字符
      fc = *Feptr;
      # 如果下一个指针大于已使用的最后指针，则更新已使用的最后指针
      if (nextptr > mb->last_used_ptr) mb->last_used_ptr = nextptr;
#ifdef SUPPORT_UNICODE
      # 如果支持 Unicode 属性，则进行以下操作
      if ((mb->poptions & PCRE2_UCP) != 0)
        {
        # 如果当前字符是下划线，则设置当前字符为单词字符；否则根据 Unicode 类别判断当前字符是否为单词字符
        if (fc == '_') cur_is_word = TRUE; else
          {
          int cat = UCD_CATEGORY(fc);
          cur_is_word = (cat == ucp_L || cat == ucp_N);
          }
        }
      else
#endif  /* SUPPORT_UNICODE */
      # 如果不支持 Unicode 属性，则根据字符属性判断当前字符是否为单词字符
      cur_is_word = CHMAX_255(fc) && (mb->ctypes[fc] & ctype_word) != 0;
      }

    /* Now see if the situation is what we want */

    # 现在看看情况是否符合我们的要求
    if ((*Fecode++ == OP_WORD_BOUNDARY)?
         cur_is_word == prev_is_word : cur_is_word != prev_is_word)
      RRETURN(MATCH_NOMATCH);
    break;


    /* ===================================================================== */
    /* Backtracking (*VERB)s, with and without arguments. Note that if the
    pattern is successfully matched, we do not come back from RMATCH. */

    # 回溯(*VERB)指令，带有和不带有参数。请注意，如果成功匹配模式，则不会从 RMATCH 返回。

    case OP_MARK:
    # 设置标记
    Fmark = mb->nomatch_mark = Fecode + 2;
    # 调用 RMATCH 函数，传入参数 Fecode + PRIV(OP_lengths)[*Fecode] + Fecode[1] 和 RM12
    RMATCH(Fecode + PRIV(OP_lengths)[*Fecode] + Fecode[1], RM12);

    /* A return of MATCH_SKIP_ARG means that matching failed at SKIP with an
    argument, and we must check whether that argument matches this MARK's
    argument. It is passed back in mb->verb_skip_ptr. If it does match, we
    return MATCH_SKIP with mb->verb_skip_ptr now pointing to the subject
    position that corresponds to this mark. Otherwise, pass back the return
    code unaltered. */
    # 如果返回 MATCH_SKIP_ARG，表示在带有参数的 SKIP 失败，必须检查该参数是否与此 MARK 的参数匹配。
    # 它在 mb->verb_skip_ptr 中传回。如果匹配，则返回 MATCH_SKIP，mb->verb_skip_ptr 现在指向与此标记对应的主题位置。否则，不改变返回代码。
    if (rrc == MATCH_SKIP_ARG &&
             PRIV(strcmp)(Fecode + 2, mb->verb_skip_ptr) == 0)
      {
      mb->verb_skip_ptr = Feptr;   /* Pass back current position */
      RRETURN(MATCH_SKIP);
      }
    RRETURN(rrc);

    case OP_FAIL:
    RRETURN(MATCH_NOMATCH);

    /* Record the current recursing group number in mb->verb_current_recurse
    when a backtracking return such as MATCH_COMMIT is given. This enables the
    recurse processing to catch verbs from within the recursion. */
    # 当出现回溯返回，如 MATCH_COMMIT 时，在 mb->verb_current_recurse 中记录当前递归组号。这使得递归处理能够捕捉递归内部的动词。
    case OP_COMMIT:
    RMATCH(Fecode + PRIV(OP_lengths)[*Fecode], RM13);
    if (rrc != MATCH_NOMATCH) RRETURN(rrc);
    mb->verb_current_recurse = Fcurrent_recurse;
    RRETURN(MATCH_COMMIT);

    case OP_COMMIT_ARG:
    Fmark = mb->nomatch_mark = Fecode + 2;
    RMATCH(Fecode + PRIV(OP_lengths)[*Fecode] + Fecode[1], RM36);
    if (rrc != MATCH_NOMATCH) RRETURN(rrc);
    mb->verb_current_recurse = Fcurrent_recurse;
    RRETURN(MATCH_COMMIT);

    case OP_PRUNE:
    RMATCH(Fecode + PRIV(OP_lengths)[*Fecode], RM14);
    if (rrc != MATCH_NOMATCH) RRETURN(rrc);
    mb->verb_current_recurse = Fcurrent_recurse;
    RRETURN(MATCH_PRUNE);

    case OP_PRUNE_ARG:
    Fmark = mb->nomatch_mark = Fecode + 2;
    RMATCH(Fecode + PRIV(OP_lengths)[*Fecode] + Fecode[1], RM15);
    if (rrc != MATCH_NOMATCH) RRETURN(rrc);
    mb->verb_current_recurse = Fcurrent_recurse;
    RRETURN(MATCH_PRUNE);

    case OP_SKIP:
    RMATCH(Fecode + PRIV(OP_lengths)[*Fecode], RM16);
    if (rrc != MATCH_NOMATCH) RRETURN(rrc);
    # 将当前位置传递回去
    mb->verb_skip_ptr = Feptr;
    # 传递当前递归深度
    mb->verb_current_recurse = Fcurrent_recurse;
    # 返回匹配跳过
    RRETURN(MATCH_SKIP);

    # 注意，为了与 Perl 兼容，带参数的 SKIP 不会设置 nomatch_mark。当模式匹配以未匹配的标记结束时，我们必须重新运行匹配，忽略失败的 SKIP_ARG 和任何在它之前的 SKIP_ARG（它们要么也失败了，要么没有被触发）。为此，我们维护一个执行的 SKIP_ARG 计数。如果一个 SKIP_ARG 到达顶层，匹配将重新运行，mb->ignore_skip_arg 设置为失败的计数。 
    case OP_SKIP_ARG:
    mb->skip_arg_count++;
    if (mb->skip_arg_count <= mb->ignore_skip_arg)
      {
      Fecode += PRIV(OP_lengths)[*Fecode] + Fecode[1];
      break;
      }
    RMATCH(Fecode + PRIV(OP_lengths)[*Fecode] + Fecode[1], RM17);
    if (rrc != MATCH_NOMATCH) RRETURN(rrc);

    # 传递当前跳过名称并返回特殊的 MATCH_SKIP_ARG 返回码。这要么被匹配的 MARK 捕获，要么到达顶层，导致重新匹配，mb->ignore_skip_arg 设置为 mb->skip_arg_count 的值。
    mb->verb_skip_ptr = Fecode + 2;
    mb->verb_current_recurse = Fcurrent_recurse;
    RRETURN(MATCH_SKIP_ARG);

    # 对于 THEN（和 THEN_ARG），我们传递操作码的地址，以便确定它出现在哪个分支中。
    case OP_THEN:
    RMATCH(Fecode + PRIV(OP_lengths)[*Fecode], RM18);
    if (rrc != MATCH_NOMATCH) RRETURN(rrc);
    mb->verb_ecode_ptr = Fecode;
    mb->verb_current_recurse = Fcurrent_recurse;
    RRETURN(MATCH_THEN);

    case OP_THEN_ARG:
    Fmark = mb->nomatch_mark = Fecode + 2;
    RMATCH(Fecode + PRIV(OP_lengths)[*Fecode] + Fecode[1], RM19);
    if (rrc != MATCH_NOMATCH) RRETURN(rrc);
    mb->verb_ecode_ptr = Fecode;
    mb->verb_current_recurse = Fcurrent_recurse;
    RRETURN(MATCH_THEN);
    /* ===================================================================== */
    /* 这里发生了一些可怕的灾难。到达这里只能意味着上面的代码或 OP_xxx 定义中出现了严重的问题。 */

    // 默认情况下，返回内部错误代码
    default:
    return PCRE2_ERROR_INTERNAL;
    }

  /* 不要在这里插入任何代码，除非经过深思熟虑；假定上面的代码中的 "continue" 跳出到这里来重复主循环。 */

  }  /* 主循环结束 */
/* Control never reaches here */
/* 控制永远不会到达这里 */

/* ========================================================================= */
/* The RRETURN() macro jumps here. The number that is saved in Freturn_id
indicates which label we actually want to return to. The value in Frdepth is
the index number of the frame in the vector. The return value has been placed
in rrc. */
/* RRETURN()宏跳转到这里。保存在Freturn_id中的数字表示我们实际想要返回的标签。Frdepth中的值是向量中帧的索引号。返回值已经放在rrc中。 */

#define LBL(val) case val: goto L_RM##val;

RETURN_SWITCH:
if (Feptr > mb->last_used_ptr) mb->last_used_ptr = Feptr;
if (Frdepth == 0) return rrc;                     /* Exit from the top level */
F = (heapframe *)((char *)F - Fback_frame);       /* Backtrack */
mb->cb->callout_flags |= PCRE2_CALLOUT_BACKTRACK; /* Note for callouts */
#ifdef DEBUG_SHOW_RMATCH
fprintf(stderr, "++ RETURN %d to %d\n", rrc, Freturn_id);
#endif

switch (Freturn_id)
  {
  LBL( 1) LBL( 2) LBL( 3) LBL( 4) LBL( 5) LBL( 6) LBL( 7) LBL( 8)
  LBL( 9) LBL(10) LBL(11) LBL(12) LBL(13) LBL(14) LBL(15) LBL(16)
  LBL(17) LBL(18) LBL(19) LBL(20) LBL(21) LBL(22) LBL(23) LBL(24)
  LBL(25) LBL(26) LBL(27) LBL(28) LBL(29) LBL(30) LBL(31) LBL(32)
  LBL(33) LBL(34) LBL(35) LBL(36)

#ifdef SUPPORT_WIDE_CHARS
  LBL(100) LBL(101)
#endif

#ifdef SUPPORT_UNICODE
  LBL(200) LBL(201) LBL(202) LBL(203) LBL(204) LBL(205) LBL(206)
  LBL(207) LBL(208) LBL(209) LBL(210) LBL(211) LBL(212) LBL(213)
  LBL(214) LBL(215) LBL(216) LBL(217) LBL(218) LBL(219) LBL(220)
  LBL(221) LBL(222) LBL(223) LBL(224) LBL(225)
#endif

  default:
  return PCRE2_ERROR_INTERNAL;
  }
#undef LBL
}

/*************************************************
*           Match a Regular Expression           *
*************************************************/
/* 匹配正则表达式 */

/* This function applies a compiled pattern to a subject string and picks out
portions of the string if it matches. Two elements in the vector are set for
each substring: the offsets to the start and end of the substring. */
/* 此函数将编译后的模式应用于主题字符串，并在匹配时提取字符串的部分。向量中为每个子字符串设置了两个元素：子字符串的起始和结束偏移量。 */
# 定义函数 pcre2_match，用于执行正则表达式的匹配
PCRE2_EXP_DEFN int PCRE2_CALL_CONVENTION
pcre2_match(const pcre2_code *code, PCRE2_SPTR subject, PCRE2_SIZE length,
  PCRE2_SIZE start_offset, uint32_t options, pcre2_match_data *match_data,
  pcre2_match_context *mcontext)
{
# 定义变量 rc，用于存储匹配结果
int rc;
# 定义变量 was_zero_terminated，用于标记主题字符串是否以零结尾
int was_zero_terminated = 0;
# 定义变量 start_bits，用于存储起始位
const uint8_t *start_bits = NULL;
# 将编译后的表达式转换为实际的 pcre2_real_code 对象
const pcre2_real_code *re = (const pcre2_real_code *)code;

# 定义布尔型变量，用于标记匹配过程中的一些状态
BOOL anchored;
BOOL firstline;
BOOL has_first_cu = FALSE;
BOOL has_req_cu = FALSE;
BOOL startline;

# 如果 PCRE2_CODE_UNIT_WIDTH 等于 8，则定义额外的变量
#if PCRE2_CODE_UNIT_WIDTH == 8
PCRE2_SPTR memchr_found_first_cu;
PCRE2_SPTR memchr_found_first_cu2;
#endif

# 定义用于存储 Unicode 字符的变量
PCRE2_UCHAR first_cu = 0;
PCRE2_UCHAR first_cu2 = 0;
PCRE2_UCHAR req_cu = 0;
PCRE2_UCHAR req_cu2 = 0;

# 定义用于存储匹配过程中的位置指针
PCRE2_SPTR bumpalong_limit;
PCRE2_SPTR end_subject;
PCRE2_SPTR true_end_subject;
PCRE2_SPTR start_match;
PCRE2_SPTR req_cu_ptr;
PCRE2_SPTR start_partial;
PCRE2_SPTR match_partial;

# 如果支持 JIT 编译，则定义用于标记 JIT 编译状态的变量
#ifdef SUPPORT_JIT
BOOL use_jit;
#endif

# 定义用于标记 Unicode 和 UCP 支持状态的变量
BOOL utf = FALSE;

#ifdef SUPPORT_UNICODE
BOOL ucp = FALSE;
BOOL allow_invalid;
uint32_t fragment_options = 0;
#ifdef SUPPORT_JIT
BOOL jit_checked_utf = FALSE;
#endif
#endif  /* SUPPORT_UNICODE */

# 定义用于存储堆栈大小和堆大小的变量
PCRE2_SIZE frame_size;
PCRE2_SIZE heapframes_size;
/* We need to have mb as a pointer to a match block, because the IS_NEWLINE
macro is used below, and it expects NLBLOCK to be defined as a pointer. */
// 定义一个指向匹配块的指针mb，因为下面使用了IS_NEWLINE宏，并且它期望NLBLOCK被定义为一个指针。

pcre2_callout_block cb;
match_block actual_match_block;
match_block *mb = &actual_match_block;

/* Recognize NULL, length 0 as an empty string. */
// 将NULL且长度为0的情况识别为空字符串。

if (subject == NULL && length == 0) subject = (PCRE2_SPTR)"";

/* Plausibility checks */
// 可信性检查

if ((options & ~PUBLIC_MATCH_OPTIONS) != 0) return PCRE2_ERROR_BADOPTION;
if (code == NULL || subject == NULL || match_data == NULL)
  return PCRE2_ERROR_NULL;

start_match = subject + start_offset;
req_cu_ptr = start_match - 1;
if (length == PCRE2_ZERO_TERMINATED)
  {
  length = PRIV(strlen)(subject);
  was_zero_terminated = 1;
  }
true_end_subject = end_subject = subject + length;

if (start_offset > length) return PCRE2_ERROR_BADOFFSET;

/* Check that the first field in the block is the magic number. */
// 检查块中的第一个字段是否是魔术数字。

if (re->magic_number != MAGIC_NUMBER) return PCRE2_ERROR_BADMAGIC;

/* Check the code unit width. */
// 检查代码单元宽度。

if ((re->flags & PCRE2_MODE_MASK) != PCRE2_CODE_UNIT_WIDTH/8)
  return PCRE2_ERROR_BADMODE;

/* PCRE2_NOTEMPTY and PCRE2_NOTEMPTY_ATSTART are match-time flags in the
options variable for this function. Users of PCRE2 who are not calling the
function directly would like to have a way of setting these flags, in the same
way that they can set pcre2_compile() flags like PCRE2_NO_AUTOPOSSESS with
constructions like (*NO_AUTOPOSSESS). To enable this, (*NOTEMPTY) and
(*NOTEMPTY_ATSTART) set bits in the pattern's "flag" function which we now
transfer to the options for this function. The bits are guaranteed to be
adjacent, but do not have the same values. This bit of Boolean trickery assumes
that the match-time bits are not more significant than the flag bits. If by
accident this is not the case, a compile-time division by zero error will
occur. */
// PCRE2_NOTEMPTY和PCRE2_NOTEMPTY_ATSTART是此函数的选项变量中的匹配时间标志。不直接调用PCRE2函数的用户希望以与设置pcre2_compile()标志相同的方式设置这些标志，例如使用(*NO_AUTOPOSSESS)来设置PCRE2_NO_AUTOPOSSESS标志。为了实现这一点，(*NOTEMPTY)和(*NOTEMPTY_ATSTART)在模式的“flag”函数中设置位，现在将这些位传输到此函数的选项中。这些位保证是相邻的，但值不相同。这一点布尔技巧假设匹配时间位不比标志位更重要。如果不小心不是这种情况，将会发生编译时的除零错误。

#define FF (PCRE2_NOTEMPTY_SET|PCRE2_NE_ATST_SET)
#define OO (PCRE2_NOTEMPTY|PCRE2_NOTEMPTY_ATSTART)
# 对选项进行按位或操作，用于设置正则表达式的标志位
options |= (re->flags & FF) / ((FF & (~FF+1)) / (OO & (~OO+1)));
# 清除 FF 和 OO 的定义
#undef FF
#undef OO

# 如果成功使用 JIT 支持对模式进行研究，将运行 JIT 可执行文件而不是函数的其余部分。大多数选项必须在编译时设置，以便 JIT 代码可用。
#ifdef SUPPORT_JIT
use_jit = (re->executable_jit != NULL &&
          (options & ~PUBLIC_JIT_MATCH_OPTIONS) == 0);
#endif

# 初始化 UTF/UCP 参数。
#ifdef SUPPORT_UNICODE
utf = (re->overall_options & PCRE2_UTF) != 0;
allow_invalid = (re->overall_options & PCRE2_MATCH_INVALID_UTF) != 0;
ucp = (re->overall_options & PCRE2_UCP) != 0;
#endif  /* SUPPORT_UNICODE */

# 将部分匹配标志转换为整数。
mb->partial = ((options & PCRE2_PARTIAL_HARD) != 0)? 2 :
              ((options & PCRE2_PARTIAL_SOFT) != 0)? 1 : 0;

# 部分匹配和 PCRE2_ENDANCHORED 目前不允许同时出现。
if (mb->partial != 0 &&
   ((re->overall_options | options) & PCRE2_ENDANCHORED) != 0)
  return PCRE2_ERROR_BADOPTION;

# 在不设置编译时标志的情况下设置偏移限制是错误的。
if (mcontext != NULL && mcontext->offset_limit != PCRE2_UNSET &&
     (re->overall_options & PCRE2_USE_OFFSET_LIMIT) == 0)
  return PCRE2_ERROR_BADOFFSETLIMIT;

# 如果匹配数据块先前使用了 PCRE2_COPY_MATCHED_SUBJECT，释放获得的内存。对于无匹配的情况，将字段设置为 NULL。
if ((match_data->flags & PCRE2_MD_COPIED_SUBJECT) != 0)
  {
  match_data->memctl.free((void *)match_data->subject,
    match_data->memctl.memory_data);
  match_data->flags &= ~PCRE2_MD_COPIED_SUBJECT;
  }
match_data->subject = NULL;

# 将错误偏移量清零，以防第一个代码单元无效 UTF。
match_data->startchar = 0;

# 准备进行 JIT 匹配。检查 UTF 字符串的有效性，除非不进行检查。
/* 
   处理请求或无效的 UTF 可以被处理。我们只检查可能在匹配过程中被检查的主题部分 - 从偏移减去最大回溯到给定长度。当通过起始偏移匹配大主题的一小部分时，这样可以节省时间。请注意，最大回溯是字符数，而不是代码单元。 
*/

#ifdef SUPPORT_JIT
if (use_jit)
  {
#ifdef SUPPORT_UNICODE
  if (utf && (options & PCRE2_NO_UTF_CHECK) == 0 && !allow_invalid)
    {
#if PCRE2_CODE_UNIT_WIDTH != 32
    unsigned int i;
#endif

    /* 对于 8 位和 16 位 UTF，请检查第一个代码单元是否是有效的字符起始。 */

#if PCRE2_CODE_UNIT_WIDTH != 32
    if (start_match < end_subject && NOT_FIRSTCU(*start_match))
      {
      if (start_offset > 0) return PCRE2_ERROR_BADUTFOFFSET;
#if PCRE2_CODE_UNIT_WIDTH == 8
      return PCRE2_ERROR_UTF8_ERR20;  /* 孤立的 0x80 字节 */
#else
      return PCRE2_ERROR_UTF16_ERR3;  /* 孤立的低代理项 */
#endif
      }
#endif  /* WIDTH != 32 */

    /* 向后移动最大回溯，以防它发生在匹配的开始处。 */

#if PCRE2_CODE_UNIT_WIDTH != 32
    for (i = re->max_lookbehind; i > 0 && start_match > subject; i--)
      {
      start_match--;
      while (start_match > subject &&
#if PCRE2_CODE_UNIT_WIDTH == 8
      (*start_match & 0xc0) == 0x80)
#else  /* 16 位 */
      (*start_match & 0xfc00) == 0xdc00)
#endif
        start_match--;
      }
#else  /* PCRE2_CODE_UNIT_WIDTH != 32 */

    /* 在 32 位库中，一个代码单元等于一个字符。但是，我们不能只减去回溯然后比较指针，因为非常大的回溯可能会创建一个无效的指针。 */

    if (start_offset >= re->max_lookbehind)
      start_match -= re->max_lookbehind;
    else
      start_match = subject;
#endif  /* PCRE2_CODE_UNIT_WIDTH != 32 */

    /* 验证主题的相关部分。调整无效 UTF 的偏移
    /* 
    无效的代码点作为整个字符串的绝对偏移量。*/
    
    match_data->rc = PRIV(valid_utf)(start_match,
      length - (start_match - subject), &(match_data->startchar));
    // 调用 valid_utf 函数检查从 start_match 开始的字符串是否是有效的 UTF-8 编码，并将结果赋给 match_data->rc
    
    if (match_data->rc != 0)
      {
      match_data->startchar += start_match - subject;
      // 如果 valid_utf 返回的结果不为 0，则将 match_data->startchar 增加 start_match 和 subject 之间的偏移量
      return match_data->rc;
      // 返回 valid_utf 的结果
      }
    jit_checked_utf = TRUE;
    // 设置 jit_checked_utf 为 TRUE
    }
#endif  /* SUPPORT_UNICODE */

  /* 如果 JIT 返回 BADOPTION，表示选择的完全或部分匹配模式未编译，则转到解释器。 */
  rc = pcre2_jit_match(code, subject, length, start_offset, options,
    match_data, mcontext);
  if (rc != PCRE2_ERROR_JIT_BADOPTION)
    {
    if (rc >= 0 && (options & PCRE2_COPY_MATCHED_SUBJECT) != 0)
      {
      length = CU2BYTES(length + was_zero_terminated);
      match_data->subject = match_data->memctl.malloc(length,
        match_data->memctl.memory_data);
      if (match_data->subject == NULL) return PCRE2_ERROR_NOMEMORY;
      memcpy((void *)match_data->subject, subject, length);
      match_data->flags |= PCRE2_MD_COPIED_SUBJECT;
      }
    return rc;
    }
  }
#endif  /* SUPPORT_JIT */

/* ========================= JIT 匹配结束 ========================== */

/* 继续非 JIT 匹配。默认情况下允许回溯到主题的开头。当偏移量为非零时，UTF 检查可能会改变这一点。 */
mb->check_subject = subject;

/* 如果 JIT 代码中未检查 UTF 主题字符串的有效性，在这里检查，并处理对无效 UTF 字符串的支持。上面的检查仅在不支持无效 UTF 且未设置 PCRE2_NO_CHECK_UTF 时发生。如果在这些情况下到达这里，这意味着主题字符串是有效的，但由于某种原因 JIT 匹配不成功。此时无需再次检查主题。

我们仅检查可能在匹配过程中被检查的主题部分 - 从偏移量减去最大回溯到给定长度。这在使用起始偏移量匹配大主题的一小部分时可以节省时间。请注意，最大回溯是字符数，而不是代码单元。

还要注意，对无效 UTF 的支持会强制进行检查，覆盖 PCRE2_NO_CHECK_UTF 的设置。 */
#ifdef SUPPORT_UNICODE
if (utf &&
#ifdef SUPPORT_JIT
    !jit_checked_utf &&
#endif
    # 如果选项中包含 PCRE2_NO_UTF_CHECK 并且 allow_invalid 为真，则执行以下代码块
    ((options & PCRE2_NO_UTF_CHECK) == 0 || allow_invalid))
  {
#if PCRE2_CODE_UNIT_WIDTH != 32
  # 如果 PCRE2_CODE_UNIT_WIDTH 不等于 32，则定义一个布尔类型的变量 skipped_bad_start，并初始化为 FALSE
  BOOL skipped_bad_start = FALSE;
#endif

  /* 对于8位和16位UTF，检查第一个代码单元是否是有效的字符起始。如果我们处理无效的UTF，只需跳过这样的代码单元。否则，给出适当的错误。 */

#if PCRE2_CODE_UNIT_WIDTH != 32
  # 如果允许无效字符，则循环检查起始匹配位置是否为有效字符起始，如果不是则跳过，并将 skipped_bad_start 设置为 TRUE
  if (allow_invalid)
    {
    while (start_match < end_subject && NOT_FIRSTCU(*start_match))
      {
      start_match++;
      skipped_bad_start = TRUE;
      }
    }
  # 如果不允许无效字符，并且起始匹配位置不是有效字符起始，则根据不同的宽度返回相应的错误码
  else if (start_match < end_subject && NOT_FIRSTCU(*start_match))
    {
    if (start_offset > 0) return PCRE2_ERROR_BADUTFOFFSET;
    # 如果宽度为8位，则返回错误码 PCRE2_ERROR_UTF8_ERR20，表示孤立的0x80字节
    # 如果宽度为16位，则返回错误码 PCRE2_ERROR_UTF16_ERR3，表示孤立的低代理项
    # 如果宽度为32位，则不执行任何操作
    #endif  /* WIDTH != 32 */
    }
  # 如果宽度为32位，则不执行任何操作
#endif

  /* mb->check_subject 字段指向 UTF 检查的起始位置；回溯不能回溯到比这更早的位置。 */

  mb->check_subject = start_match;

  /* 向后移动最大回溯量，以防它发生在匹配的开始处，但如果我们在上面跳过了坏的8位或16位代码单元，则不执行此操作。 */

#if PCRE2_CODE_UNIT_WIDTH != 32
  # 如果没有跳过坏的8位或16位代码单元，则向后移动最大回溯量
  if (!skipped_bad_start)
    {
    unsigned int i;
    for (i = re->max_lookbehind; i > 0 && mb->check_subject > subject; i--)
      {
      mb->check_subject--;
      while (mb->check_subject > subject &&
#if PCRE2_CODE_UNIT_WIDTH == 8
      (*mb->check_subject & 0xc0) == 0x80)
#else  /* 16-bit */
      (*mb->check_subject & 0xfc00) == 0xdc00)
#endif
        mb->check_subject--;
      }
    }
#else  /* PCRE2_CODE_UNIT_WIDTH != 32 */

  /* 在32位库中，一个代码单元等于一个字符。然而，我们不能简单地减去回溯量然后比较指针，因为非常大的回溯量可能会创建一个无效的指针。 */

  # 如果起始偏移量大于等于最大回溯量，则将 mb->check_subject 减去最大回溯量；否则将 mb->check_subject 设置为 subject
  if (start_offset >= re->max_lookbehind)
    mb->check_subject -= re->max_lookbehind;
  else
    mb->check_subject = subject;
#endif  /* PCRE2_CODE_UNIT_WIDTH != 32 */

  /* 验证主题的相关部分。如果我们遇到了在 start_match 之前的字符中出现的坏的 UTF 字符，就会进入循环。*/

  for (;;)
    {
    // 使用私有函数 valid_utf 验证主题的一部分，检查是否是有效的 UTF 字符串，并返回起始字符的位置
    match_data->rc = PRIV(valid_utf)(mb->check_subject,
      length - (mb->check_subject - subject), &(match_data->startchar));

    // 如果是有效的 UTF 字符串，则跳出循环
    if (match_data->rc == 0) break;   /* 有效的 UTF 字符串 */

    /* 如果是无效的 UTF 字符串，调整偏移量为整个字符串中的绝对偏移量。如果我们处理无效的 UTF 字符串，将 end_subject 设置为在坏的代码单元之前停止，并将选项设置为“不是行尾”。否则返回错误。 */
    match_data->startchar += mb->check_subject - subject;
    if (!allow_invalid || match_data->rc > 0) return match_data->rc;
    end_subject = subject + match_data->startchar;

    /* 如果结束位置在 start_match 之前，意味着在我们因为回顾而反转的额外代码单元中有无效的 UTF。前进到第一个坏的代码单元，然后在 8 位和 16 位模式下跳过无效字符开始的代码单元，并使用原始结束点再次尝试。 */
    if (end_subject < start_match)
      {
      mb->check_subject = end_subject + 1;
#if PCRE2_CODE_UNIT_WIDTH != 32
      while (mb->check_subject < start_match && NOT_FIRSTCU(*mb->check_subject))
        mb->check_subject++;
#endif
      end_subject = true_end_subject;
      }

    /* 否则，设置非行尾选项，并进行匹配。 */
    else
      {
      fragment_options = PCRE2_NOTEOL;
      break;
      }
    }
  }
#endif  /* SUPPORT_UNICODE */

/* 如果匹配上下文为空，则使用默认上下文，但是我们从模式中获取内存控制函数。 */
if (mcontext == NULL)
  {
  mcontext = (pcre2_match_context *)(&PRIV(default_match_context));
  mb->memctl = re->memctl;
  }
else mb->memctl = mcontext->memctl;
# 检查正则表达式是否以锚定模式匹配
anchored = ((re->overall_options | options) & PCRE2_ANCHORED) != 0;
# 检查正则表达式是否使用了 PCRE2_FIRSTLINE 选项
firstline = (re->overall_options & PCRE2_FIRSTLINE) != 0;
# 检查正则表达式是否使用了 PCRE2_STARTLINE 选项
startline = (re->flags & PCRE2_STARTLINE) != 0;
# 设置匹配上限
bumpalong_limit = (mcontext->offset_limit == PCRE2_UNSET)? true_end_subject : subject + mcontext->offset_limit;

# 初始化并设置调用块中的固定字段，并在匹配块中设置指针
mb->cb = &cb;
cb.version = 2;
cb.subject = subject;
cb.subject_length = (PCRE2_SIZE)(end_subject - subject);
cb.callout_flags = 0;

# 填充匹配块中的其余字段，除了 moptions，该字段稍后设置
mb->callout = mcontext->callout;
mb->callout_data = mcontext->callout_data;
mb->start_subject = subject;
mb->start_offset = start_offset;
mb->end_subject = end_subject;
mb->hasthen = (re->flags & PCRE2_HASTHEN) != 0;
mb->allowemptypartial = (re->max_lookbehind > 0) || (re->flags & PCRE2_MATCH_EMPTY) != 0;
mb->poptions = re->overall_options;          # 模式选项
mb->ignore_skip_arg = 0;
mb->mark = mb->nomatch_mark = NULL;          # 如果从未设置，则为空

# 名称表用于查找与给定名称关联的所有数字，用于条件测试。代码跟随名称表。
mb->name_table = (PCRE2_UCHAR *)((uint8_t *)re + sizeof(pcre2_real_code));
mb->name_count = re->name_count;
mb->name_entry_size = re->name_entry_size;
mb->start_code = mb->name_table + re->name_count * re->name_entry_size;

# 处理 \R 和换行符设置
mb->bsr_convention = re->bsr_convention;
mb->nltype = NLTYPE_FIXED;
# 根据正则表达式的换行约定设置换行符的长度和值
switch(re->newline_convention)
  {
  case PCRE2_NEWLINE_CR:
  mb->nllen = 1;
  mb->nl[0] = CHAR_CR;
  break;

  case PCRE2_NEWLINE_LF:
  mb->nllen = 1;
  mb->nl[0] = CHAR_NL;
  break;

  case PCRE2_NEWLINE_NUL:
  mb->nllen = 1;
  mb->nl[0] = CHAR_NUL;
  break;

  case PCRE2_NEWLINE_CRLF:
  mb->nllen = 2;
  mb->nl[0] = CHAR_CR;
  mb->nl[1] = CHAR_NL;
  break;

  case PCRE2_NEWLINE_ANY:
  mb->nltype = NLTYPE_ANY;
  break;

  case PCRE2_NEWLINE_ANYCRLF:
  mb->nltype = NLTYPE_ANYCRLF;
  break;

  default: return PCRE2_ERROR_INTERNAL;
  }

# 计算回溯帧的大小，确保对齐
frame_size = (offsetof(heapframe, ovector) +
  re->top_bracket * 2 * sizeof(PCRE2_SIZE) + HEAPFRAME_ALIGNMENT - 1) &
  ~(HEAPFRAME_ALIGNMENT - 1);

# 根据模式设置限制，如果模式设置的限制小于匹配上下文的限制，则使用模式设置的限制
mb->heap_limit = ((mcontext->heap_limit < re->limit_heap)?
  mcontext->heap_limit : re->limit_heap) * 1024;

mb->match_limit = (mcontext->match_limit < re->limit_match)?
  mcontext->match_limit : re->limit_match;

mb->match_limit_depth = (mcontext->depth_limit < re->limit_depth)?
  mcontext->depth_limit : re->limit_depth;

# 如果模式有很多捕获括号，帧大小可能会很大。设置初始帧向量大小，以确保至少有10
/* 计算堆栈帧的大小，至少为 START_FRAMES_SIZE。如果大于堆限制，则尽可能获取尽可能大的向量。始终将大小舍入为帧大小的倍数。 */

heapframes_size = frame_size * 10;
if (heapframes_size < START_FRAMES_SIZE) heapframes_size = START_FRAMES_SIZE;
if (heapframes_size > mb->heap_limit)
  {
  if (frame_size > mb->heap_limit ) return PCRE2_ERROR_HEAPLIMIT;
  heapframes_size = mb->heap_limit;
  }

/* 如果匹配数据块中的现有帧向量足够大，我们可以使用它。否则，释放任何预先存在的向量并获取一个新的向量。 */

if (match_data->heapframes_size < heapframes_size)
  {
  match_data->memctl.free(match_data->heapframes,
    match_data->memctl.memory_data);
  match_data->heapframes = match_data->memctl.malloc(heapframes_size,
    match_data->memctl.memory_data);
  if (match_data->heapframes == NULL)
    {
    match_data->heapframes_size = 0;
    return PCRE2_ERROR_NOMEMORY;
    }
  match_data->heapframes_size = heapframes_size;
  }

/* 写入第一个帧内的 ovector，标记每个捕获未设置，并在复制到新帧时避免未初始化的内存读取错误。 */

memset((char *)(match_data->heapframes) + offsetof(heapframe, ovector), 0xff,
  frame_size - offsetof(heapframe, ovector));

/* 指向各个字符表的指针 */

mb->lcc = re->tables + lcc_offset;
mb->fcc = re->tables + fcc_offset;
mb->ctypes = re->tables + ctypes_offset;

/* 设置第一个匹配的代码单元，如果可用。如果没有第一个代码单元，则可能有可能的第一个字符的位图。 */

if ((re->flags & PCRE2_FIRSTSET) != 0)
  {
  has_first_cu = TRUE;
  first_cu = first_cu2 = (PCRE2_UCHAR)(re->first_codeunit);
  if ((re->flags & PCRE2_FIRSTCASELESS) != 0)
    {
    first_cu2 = TABLE_GET(first_cu, mb->fcc, first_cu);
#ifdef SUPPORT_UNICODE
#if PCRE2_CODE_UNIT_WIDTH == 8
    if (first_cu > 127 && ucp && !utf) first_cu2 = UCD_OTHERCASE(first_cu);
#else
    # 如果第一个字符的 Unicode 值大于 127，并且 utf 或 ucp 为真，则将第一个字符的其他大小写形式赋给 first_cu2
    if (first_cu > 127 && (utf || ucp)) first_cu2 = UCD_OTHERCASE(first_cu);
#else
#endif  /* SUPPORT_UNICODE */
    }
  }
else
  if (!startline && (re->flags & PCRE2_FIRSTMAPSET) != 0)
    start_bits = re->start_bitmap;

# 如果设置了 PCRE2_LASTSET 标志位
if ((re->flags & PCRE2_LASTSET) != 0)
  {
  # 标记是否存在最后已知必需字符
  has_req_cu = TRUE
  # 获取最后已知必需字符
  req_cu = req_cu2 = (PCRE2_UCHAR)(re->last_codeunit)
  # 如果设置了 PCRE2_LASTCASELESS 标志位
  if ((re->flags & PCRE2_LASTCASELESS) != 0)
    {
    # 获取字符的其他大小写形式
    req_cu2 = TABLE_GET(req_cu, mb->fcc, req_cu)
    # 如果支持 UNICODE
    # 如果字符大于 127 并且不是 UTF-8 编码，则获取字符的其他大小写形式
    # 否则，获取字符的其他大小写形式
    if (req_cu > 127 && ucp && !utf) req_cu2 = UCD_OTHERCASE(req_cu)
    }
  }

# 循环处理未锚定的重复匹配尝试；对于锚定的正则表达式，循环只运行一次
# 如果支持 UNICODE，则跳转到 FRAGMENT_RESTART 标签
# 初始化部分匹配的起始位置和完全匹配的起始位置
start_partial = match_partial = NULL
# 标记是否匹配到字符串末尾
mb->hitend = FALSE

# 如果字符宽度为 8 位
# 初始化 memchr_found_first_cu 和 memchr_found_first_cu2
memchr_found_first_cu = NULL
memchr_found_first_cu2 = NULL

# 无限循环
for(;;)
  {
  # 新的匹配起始位置
  PCRE2_SPTR new_start_match

  # ----------------- 开始匹配优化 ----------------

  # 如果未设置 PCRE2_NO_START_OPTIMIZE 标志位
  # 如果 firstline 为 TRUE，则匹配的起始位置受多行字符串的第一行限制
  # 临时调整 end_subject，以便在换行符处停止第一个字符的扫描
  # 如果匹配失败，则后续代码会中断循环
  if ((re->overall_options & PCRE2_NO_START_OPTIMIZE) == 0)
    {
    if (firstline)
      {
      # 临时变量 t 用于保存匹配的起始位置
      PCRE2_SPTR t = start_match
#ifdef SUPPORT_UNICODE
      # 如果支持 Unicode，则在不是换行符的情况下移动 t 指针
      if (utf)
        {
        while (t < end_subject && !IS_NEWLINE(t))
          {
          t++;
          ACROSSCHAR(t < end_subject, t, t++);
          }
        }
      else
#endif
      # 如果不支持 Unicode，则在不是换行符的情况下移动 t 指针
      while (t < end_subject && !IS_NEWLINE(t)) t++;
      # 将 end_subject 指针移动到 t 指针的位置
      end_subject = t;
      }

    /* Anchored: check the first code unit if one is recorded. This may seem
    pointless but it can help in detecting a no match case without scanning for
    the required code unit. */

    # 如果是锚定的，则检查是否记录了第一个代码单元
    if (anchored)
      {
      # 如果记录了第一个代码单元或者 start_bits 不为空
      if (has_first_cu || start_bits != NULL)
        {
        # 检查是否 start_match 小于 end_subject
        BOOL ok = start_match < end_subject;
        if (ok)
          {
          # 获取 start_match 对应的字符
          PCRE2_UCHAR c = UCHAR21TEST(start_match);
          # 如果记录了第一个代码单元，并且 c 等于 first_cu 或者 c 等于 first_cu2
          ok = has_first_cu && (c == first_cu || c == first_cu2);
          # 如果不满足上述条件，并且 start_bits 不为空
          if (!ok && start_bits != NULL)
            {
            # 如果 PCRE2_CODE_UNIT_WIDTH 不等于 8，则将 c 限制在 255 以内
#if PCRE2_CODE_UNIT_WIDTH != 8
            if (c > 255) c = 255;
#endif
            # 检查 start_bits 中对应位置是否为 1
            ok = (start_bits[c/8] & (1u << (c&7))) != 0;
            }
          }
        # 如果不满足上述条件
        if (!ok)
          {
          # 设置 rc 为 MATCH_NOMATCH，并且跳出循环
          rc = MATCH_NOMATCH;
          break;
          }
        }
      }

    /* Not anchored. Advance to a unique first code unit if there is one. */

    # 如果没有锚定，则移动到一个唯一的第一个代码单元
    else
      {
      # 如果记录了第一个代码单元
      if (has_first_cu)
        {
        # 如果 first_cu 不等于 first_cu2（不区分大小写）
        if (first_cu != first_cu2)
          {
          # 在 16 位和 32 位模式下，自行搜索，可以同时查找两种情况
#if PCRE2_CODE_UNIT_WIDTH != 8
          PCRE2_UCHAR smc;
          while (start_match < end_subject &&
                (smc = UCHAR21TEST(start_match)) != first_cu &&
                 smc != first_cu2)
            start_match++;
#endif  /* 8-bit handling */
          }

        # 如果区分大小写
        else
          {
#if PCRE2_CODE_UNIT_WIDTH != 8
          while (start_match < end_subject && UCHAR21TEST(start_match) !=
                 first_cu)
            start_match++;
#else
          // 如果没有找到所需的第一个代码单元，并且已经到达主题的末尾，则中断循环，强制匹配失败，除非进行部分匹配，此时让下一个循环在主题末尾运行。例如，考虑模式 /(?<=abc)def/，即使字符串不包含起始字符"d"，也部分匹配"abc"。如果尚未到达主题的真正末尾（PCRE2_FIRSTLINE导致end_subject被临时修改），我们也让循环运行，因为匹配字符串合法地允许以换行符的第一个代码单元开头。

        start_match = memchr(start_match, first_cu, end_subject - start_match);
        if (start_match == NULL) start_match = end_subject;
#endif
        }

        /* 如果找不到第一个代码单元，并且已经到达主题的真正末尾，中断bumpalong循环，强制匹配失败，除非进行部分匹配，此时让下一个循环在主题末尾运行。要了解原因，请考虑模式 /(?<=abc)def/，即使字符串不包含起始字符"d"，也部分匹配"abc"。如果尚未到达主题的真正末尾（PCRE2_FIRSTLINE导致end_subject被临时修改），我们也让循环运行，因为匹配字符串合法地允许以换行符的第一个代码单元开头。 */

        if (mb->partial == 0 && start_match >= mb->end_subject)
          {
          rc = MATCH_NOMATCH;
          break;
          }
        }

      /* 如果没有第一个代码单元，根据需要，对于多行匹配，前进到换行符的后面。 */

      else if (startline)
        {
        if (start_match > mb->start_subject + start_offset)
          {
#ifdef SUPPORT_UNICODE
          if (utf)
            {
            while (start_match < end_subject && !WAS_NEWLINE(start_match))
              {
              start_match++;
              ACROSSCHAR(start_match < end_subject, start_match, start_match++);
              }
            }
          else
#endif
          // 当匹配起始位置小于结束位置并且不是换行符时，向前移动匹配起始位置
          while (start_match < end_subject && !WAS_NEWLINE(start_match))
            start_match++;

          /* 如果刚刚经过一个 CR 并且换行符选项是 ANY 或 ANYCRLF，并且现在是 LF，再向前移动一个代码单元的匹配位置 */
          if (start_match[-1] == CHAR_CR &&
               (mb->nltype == NLTYPE_ANY || mb->nltype == NLTYPE_ANYCRLF) &&
               start_match < end_subject &&
               UCHAR21TEST(start_match) == CHAR_NL)
            start_match++;
          }
        }

      /* 如果没有第一个代码单元或需要多行起始的要求，如果已经识别出非唯一的第一个代码单元，就向前移动到非唯一的第一个代码单元。位图只包含 256 位。当代码单元宽度为 16 或 32 位时，所有大于 254 的代码单元设置第 255 位。 */
      else if (start_bits != NULL)
        {
        while (start_match < end_subject)
          {
          uint32_t c = UCHAR21TEST(start_match);
#if PCRE2_CODE_UNIT_WIDTH != 8
          if (c > 255) c = 255;
#endif
          if ((start_bits[c/8] & (1u << (c&7))) != 0) break;
          start_match++;
          }

        /* 参见上面关于第一个代码单元检查的注释。 */
        if (mb->partial == 0 && start_match >= mb->end_subject)
          {
          rc = MATCH_NOMATCH;
          break;
          }
        }
      }   /* 结束第一个代码单元处理 */

    /* 恢复修改后的结束位置 */
    end_subject = mb->end_subject;

    /* 下面两个优化必须在部分匹配时禁用。 */
#if PCRE2_CODE_UNIT_WIDTH != 8
            while (p < end_subject)
              {
              uint32_t pp = UCHAR21INCTEST(p);
              if (pp == req_cu || pp == req_cu2) { p--; break; }
              }
#else  /* 8-bit code units */
            // 如果是8位代码单元，定义指针pp指向p的位置
            PCRE2_SPTR pp = p;
            // 在pp指向的位置到end_subject之间查找req_cu，返回指向该位置的指针
            p = memchr(pp, req_cu, end_subject - pp);
            // 如果找不到req_cu
            if (p == NULL)
              {
              // 在pp指向的位置到end_subject之间查找req_cu2，返回指向该位置的指针
              p = memchr(pp, req_cu2, end_subject - pp);
              // 如果找不到req_cu2，将p指向end_subject
              if (p == NULL) p = end_subject;
              }
#endif /* PCRE2_CODE_UNIT_WIDTH != 8 */
            }

          /* The caseful case */

          else
            {
#if PCRE2_CODE_UNIT_WIDTH != 8
            // 如果不是8位代码单元，循环直到p指向end_subject
            while (p < end_subject)
              {
              // 如果p指向的位置的字符等于req_cu，将p向前移动一位并跳出循环
              if (UCHAR21INCTEST(p) == req_cu) { p--; break; }
              }

#else  /* 8-bit code units */
            // 如果是8位代码单元，查找p指向的位置到end_subject之间的req_cu，返回指向该位置的指针
            p = memchr(p, req_cu, end_subject - p);
            // 如果找不到req_cu，将p指向end_subject
            if (p == NULL) p = end_subject;
#endif
            }

          /* If we can't find the required code unit, break the bumpalong loop,
          forcing a match failure. */

          // 如果找不到所需的代码单元，跳出bumpalong循环，强制匹配失败
          if (p >= end_subject)
            {
            rc = MATCH_NOMATCH;
            break;
            }

          /* If we have found the required code unit, save the point where we
          found it, so that we don't search again next time round the bumpalong
          loop if the start hasn't yet passed this code unit. */

          // 如果找到了所需的代码单元，保存找到的位置，以便下次bumpalong循环时不再搜索
          // 如果起始位置还没有通过这个代码单元
          req_cu_ptr = p;
          }
        }
      }
    }

  /* ------------ End of start of match optimizations ------------ */

  /* Give no match if we have passed the bumpalong limit. */

  // 如果已经超过bumpalong限制，不进行匹配
  if (start_match > bumpalong_limit)
    {
    rc = MATCH_NOMATCH;
    break;
    }

  /* OK, we can now run the match. If "hitend" is set afterwards, remember the
  first starting point for which a partial match was found. */

  // 现在可以运行匹配。如果之后设置了"hitend"，记住找到部分匹配的第一个起始点
  cb.start_match = (PCRE2_SIZE)(start_match - subject);
  cb.callout_flags |= PCRE2_CALLOUT_STARTMATCH;

  mb->start_used_ptr = start_match;
  mb->last_used_ptr = start_match;
#ifdef SUPPORT_UNICODE
  mb->moptions = options | fragment_options;
#else
  mb->moptions = options;
#endif
  # 重置匹配调用计数
  mb->match_call_count = 0;
  # 重置结束偏移量
  mb->end_offset_top = 0;
  # 重置跳过参数计数
  mb->skip_arg_count = 0;

  # 进行匹配操作
  rc = match(start_match, mb->start_code, re->top_bracket, frame_size,
    match_data, mb);

  # 如果匹配到结尾并且部分匹配为空
  if (mb->hitend && start_partial == NULL)
    {
    start_partial = mb->start_used_ptr;
    match_partial = start_match;
    }

  # 根据匹配结果进行不同的处理
  switch(rc)
    {
    # 如果 MATCH_SKIP_ARG 到达这个级别，意味着未找到匹配 SKIP 参数的 MARK。在这种情况下，Perl 忽略 SKIP。我们唯一能做的就是在相同的位置重新进行匹配，使用一个标志来强制忽略带参数的 SKIP。仅将此情况视为 NOMATCH 是行不通的，因为它不会检查主题为 AC 时的其他模式的替代项，比如 A(*SKIP:A)B|AC。
    case MATCH_SKIP_ARG:
    new_start_match = start_match;
    mb->ignore_skip_arg = mb->skip_arg_count;
    break;

    # SKIP 明确传回下一个起始点，但如果它不大于我们刚刚完成的匹配，则将其视为 NOMATCH。
    case MATCH_SKIP:
    if (mb->verb_skip_ptr > start_match)
      {
      new_start_match = mb->verb_skip_ptr;
      break;
      }
    # 继续执行下面的操作

    # NOMATCH 和 PRUNE 向前移动一个字符。在这个级别，THEN 的行为与 PRUNE 完全相同。取消忽略带参数的 SKIP。 
    case MATCH_NOMATCH:
    case MATCH_PRUNE:
    case MATCH_THEN:
    mb->ignore_skip_arg = 0;
    new_start_match = start_match + 1;
#ifdef SUPPORT_UNICODE
    if (utf)
      ACROSSCHAR(new_start_match < end_subject, new_start_match,
        new_start_match++);
#endif
    break;

    # COMMIT 禁用了 bumpalong，但在其他方面的行为与 NOMATCH 相同。
    case MATCH_COMMIT:
    rc = MATCH_NOMATCH;
    goto ENDLOOP;

    # 任何其他返回值要么是匹配，要么是某种错误。
    default:
    goto ENDLOOP;
    }

  /* Control reaches here for the various types of "no match at this point"
  result. Reset the code to MATCH_NOMATCH for subsequent checking. */
  // 重置代码为MATCH_NOMATCH，用于后续检查

  rc = MATCH_NOMATCH;

  /* If PCRE2_FIRSTLINE is set, the match must happen before or at the first
  newline in the subject (though it may continue over the newline). Therefore,
  if we have just failed to match, starting at a newline, do not continue. */
  // 如果设置了PCRE2_FIRSTLINE，则匹配必须发生在主题中的第一个换行符之前或之后（尽管它可能会继续跨越换行符）。因此，如果我们刚刚在换行符处失败了匹配，就不要继续了。

  if (firstline && IS_NEWLINE(start_match)) break;

  /* Advance to new matching position */
  // 前进到新的匹配位置
  start_match = new_start_match;

  /* Break the loop if the pattern is anchored or if we have passed the end of
  the subject. */
  // 如果模式是锚定的，或者我们已经超过了主题的末尾，则中断循环。
  if (anchored || start_match > end_subject) break;

  /* If we have just passed a CR and we are now at a LF, and the pattern does
  not contain any explicit matches for \r or \n, and the newline option is CRLF
  or ANY or ANYCRLF, advance the match position by one more code unit. In
  normal matching start_match will aways be greater than the first position at
  this stage, but a failed *SKIP can cause a return at the same point, which is
  why the first test exists. */
  // 如果我们刚刚通过了一个CR，现在又到了一个LF，并且模式不包含任何对\r或\n的显式匹配，并且换行选项是CRLF或ANY或ANYCRLF，则将匹配位置向前移动一个代码单元。在正常匹配中，start_match始终大于此阶段的第一个位置，但是失败的*SKIP可能会导致在相同的位置返回，这就是为什么存在第一个测试的原因。

  if (start_match > subject + start_offset &&
      start_match[-1] == CHAR_CR &&
      start_match < end_subject &&
      *start_match == CHAR_NL &&
      (re->flags & PCRE2_HASCRORLF) == 0 &&
        (mb->nltype == NLTYPE_ANY ||
         mb->nltype == NLTYPE_ANYCRLF ||
         mb->nllen == 2))
    start_match++;

  mb->mark = NULL;   /* Reset for start of next match attempt */
  // 重置为下一次匹配尝试的开始
  }                  /* End of for(;;) "bumpalong" loop */
/* ==========================================================================*/

/* 当我们到达这里时，以下停止条件之一为真：

(1) 匹配成功，完全或部分；

(2) 模式已锚定，或者在 (*COMMIT) 后匹配失败；

(3) 我们已经超出了主题的末尾，或者超出了 bumpalong 限制；

(4) 设置了 PCRE2_FIRSTLINE，并且在主题中的换行符处匹配失败，因为此选项要求匹配发生在主题中的第一个换行符之前。

(5) 发生了某种错误。

*/

ENDLOOP:

/* 如果 end_subject != true_end_subject，则意味着我们正在处理无效的 UTF，并且刚刚处理了一个非终端片段。如果这导致没有匹配或部分匹配，我们必须继续到下一个片段（部分匹配仅在主题的最末尾返回给调用者）。使用循环是为了避免尝试匹配空片段；如果模式可以匹配空字符串，它已经这样做了。 */

#ifdef SUPPORT_UNICODE
if (utf && end_subject != true_end_subject &&
    (rc == MATCH_NOMATCH || rc == PCRE2_ERROR_PARTIAL))
  {
  for (;;)
    {
    /* 跳过第一个坏代码单元，然后在 8 位和 16 位模式下跳过无效字符起始代码单元。 */

    start_match = end_subject + 1;

#if PCRE2_CODE_UNIT_WIDTH != 32
    while (start_match < true_end_subject && NOT_FIRSTCU(*start_match))
      start_match++;
#endif

    /* 如果我们已经到达主题的末尾，那么没有另一个非空片段，所以放弃。 */

    if (start_match >= true_end_subject)
      {
      rc = MATCH_NOMATCH;  /* 以防它是部分匹配 */
      break;
      }

    /* 检查主题的其余部分 */

    mb->check_subject = start_match;
    rc = PRIV(valid_utf)(start_match, length - (start_match - subject),
      &(match_data->startchar));

    /* 主题的其余部分是有效的 UTF。 */
    # 如果 rc 等于 0，则表示成功匹配到了一个片段
    if (rc == 0)
      {
      # 设置匹配结束位置为真正的结束位置
      mb->end_subject = end_subject = true_end_subject;
      # 设置片段选项为 PCRE2_NOTBOL，并跳转到 FRAGMENT_RESTART 标签处重新开始
      fragment_options = PCRE2_NOTBOL;
      goto FRAGMENT_RESTART;
      }

    # 如果 rc 小于 0，则表示出现了 UTF 错误
    else if (rc < 0)
      {
      # 设置匹配结束位置为开始匹配位置加上匹配数据的起始字符位置
      mb->end_subject = end_subject = start_match + match_data->startchar;
      # 如果结束位置大于开始位置，则设置片段选项为 PCRE2_NOTBOL|PCRE2_NOTEOL，并跳转到 FRAGMENT_RESTART 标签处重新开始
      if (end_subject > start_match)
        {
        fragment_options = PCRE2_NOTBOL|PCRE2_NOTEOL;
        goto FRAGMENT_RESTART;
        }
      }
    }
  }
#endif  /* SUPPORT_UNICODE */

/* 填充始终返回在匹配数据中的字段 */

match_data->code = re;  // 将正则表达式对象的地址赋给匹配数据的code字段
match_data->mark = mb->mark;  // 将mark字段的值赋给匹配数据的mark字段
match_data->matchedby = PCRE2_MATCHEDBY_INTERPRETER;  // 将匹配方式标记为解释器匹配

/* 处理完全成功的匹配。将返回码设置为捕获字符串的数量，如果捕获字符串太多无法适应ovector，则返回0，然后设置剩余的返回值并返回。如果请求了主题字符串的副本，则进行复制。 */

if (rc == MATCH_MATCH)  // 如果匹配成功
  {
  match_data->rc = ((int)mb->end_offset_top >= 2 * match_data->oveccount)?  // 如果结束偏移大于等于oveccount的两倍
    0 : (int)mb->end_offset_top/2 + 1;  // 则返回0，否则返回(end_offset_top/2 + 1)
  match_data->startchar = start_match - subject;  // 设置匹配数据的startchar字段
  match_data->leftchar = mb->start_used_ptr - subject;  // 设置匹配数据的leftchar字段
  match_data->rightchar = ((mb->last_used_ptr > mb->end_match_ptr)?  // 设置匹配数据的rightchar字段
    mb->last_used_ptr : mb->end_match_ptr) - subject;
  if ((options & PCRE2_COPY_MATCHED_SUBJECT) != 0)  // 如果请求了复制主题字符串
    {
    length = CU2BYTES(length + was_zero_terminated);  // 计算主题字符串的长度
    match_data->subject = match_data->memctl.malloc(length,  // 分配内存
      match_data->memctl.memory_data);
    if (match_data->subject == NULL) return PCRE2_ERROR_NOMEMORY;  // 如果分配内存失败，则返回内存不足错误
    memcpy((void *)match_data->subject, subject, length);  // 复制主题字符串
    match_data->flags |= PCRE2_MD_COPIED_SUBJECT;  // 设置匹配数据的标志
    }
  else match_data->subject = subject;  // 否则直接将主题字符串赋给匹配数据的subject字段
  return match_data->rc;  // 返回返回码
  }

/* 如果出现部分匹配、错误，或者整体匹配尝试在所有允许的起始位置都失败，则控制流会到这里。任何标记数据都在nomatch_mark字段中。 */

match_data->mark = mb->nomatch_mark;  // 将nomatch_mark字段的值赋给匹配数据的mark字段

/* 对于除了nomatch或部分匹配之外的任何情况，只需返回返回码。 */

if (rc != MATCH_NOMATCH && rc != PCRE2_ERROR_PARTIAL) match_data->rc = rc;  // 如果返回码不是nomatch或部分匹配，则将返回码设置为rc

/* 处理部分匹配。如果请求了“软”部分匹配，搜索完整匹配将继续进行，此时rc的值将是MATCH_NOMATCH。对于“硬”部分匹配，它将已经是PCRE2_ERROR_PARTIAL。 */
# 如果部分匹配不为空，则执行以下操作
else if (match_partial != NULL)
  {
  # 将匹配数据的主题设置为给定的主题
  match_data->subject = subject;
  # 设置匹配数据的偏移量，表示匹配部分在主题中的起始位置和结束位置
  match_data->ovector[0] = match_partial - subject;
  match_data->ovector[1] = end_subject - subject;
  # 设置匹配数据的起始字符位置、左侧字符位置和右侧字符位置
  match_data->startchar = match_partial - subject;
  match_data->leftchar = start_partial - subject;
  match_data->rightchar = end_subject - subject;
  # 设置匹配数据的返回代码为部分匹配错误
  match_data->rc = PCRE2_ERROR_PARTIAL;
  }

# 否则，这是经典的不匹配情况
else match_data->rc = PCRE2_ERROR_NOMATCH;

# 返回匹配数据的返回代码
return match_data->rc;
}

# 这些 #undef 是为了使 CMake 能够进行统一构建
# 取消定义 NLBLOCK、PSSTART 和 PSEND
#undef NLBLOCK /* 包含换行符信息的块 */
#undef PSSTART /* 包含处理后字符串起始位置的字段 */
#undef PSEND   /* 包含处理后字符串结束位置的字段 */

# pcre2_match.c 结束
```