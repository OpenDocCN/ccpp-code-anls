# `nmap\libpcre\src\pcre2_dfa_match.c`

```
/*************************************************
*      Perl-Compatible Regular Expressions       *
*************************************************/

/* PCRE is a library of functions to support regular expressions whose syntax
and semantics are as close as possible to those of the Perl 5 language.

                       Written by Philip Hazel
     Original API code Copyright (c) 1997-2012 University of Cambridge
          New API code Copyright (c) 2016-2022 University of Cambridge

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
/* 
此模块包含外部函数 pcre2_dfa_match()，它是一种使用 DFA 算法（不是真正的 FSM）的替代匹配函数。这不是 Perl 兼容的，但在某些应用中具有优势。
*/

/* 
关于性能的注意事项：这个函数的用户发送了一些改进他的模式性能的代码。我不能直接使用它，因为它不是线程安全的，并且对模式大小做出了假设。此外，它导致测试 7 循环，测试 9 崩溃并出现段错误。

问题在于检查重复状态，它是通过对状态列表进行简单的线性搜索来完成的。（在下面的代码中搜索 "duplicate" 以找到相关代码。）对于许多模式，一次可能不会有太多活动状态，所以简单的线性搜索是可以接受的。但对于有许多活动状态的模式来说，这可能会成为瓶颈。建议的代码在添加状态到状态列表时使用了一个索引方案来记住哪些状态先前已经被用于每个字符，并在知道不会有重复的情况下避免了线性搜索。这是在检查重复状态时实现的。

我在检查重复状态时写了一些线程安全的、不受限制的代码，尝试了类似的方法（而不是在添加状态时）。它对某些特定构造的模式和特定主题字符串确实提供了 13% 的改进，但对其他字符串和测试套件中的许多简单模式则效果更差。我认为主要问题是初始化索引所需的额外时间。这必须为每次调用 internal_dfa_match() 完成。（提供的补丁使用了一个静态向量，只初始化一次 - 我怀疑这是测试中出现问题的原因。）

总的来说，我得出结论，某些情况下的收益并不足以弥补损失。
*/
/* 在其他地方找到了更好的代码，所以我放弃了这段代码。 */

#ifdef HAVE_CONFIG_H
#include "config.h"
#endif

#define NLBLOCK mb             /* 包含换行信息的块 */
#define PSSTART start_subject  /* 包含处理过的字符串起始位置的字段 */
#define PSEND   end_subject    /* 包含处理过的字符串结束位置的字段 */

#include "pcre2_internal.h"

#define PUBLIC_DFA_MATCH_OPTIONS \
  (PCRE2_ANCHORED|PCRE2_ENDANCHORED|PCRE2_NOTBOL|PCRE2_NOTEOL|PCRE2_NOTEMPTY| \
   PCRE2_NOTEMPTY_ATSTART|PCRE2_NO_UTF_CHECK|PCRE2_PARTIAL_HARD| \
   PCRE2_PARTIAL_SOFT|PCRE2_DFA_SHORTEST|PCRE2_DFA_RESTART| \
   PCRE2_COPY_MATCHED_SUBJECT)


/*************************************************
*      Code parameters and static tables         *
*************************************************/

/* 这些是用于将 OP_TYPESTAR 和其他操作码转换为其他操作码的偏移量，在特殊条件下使用。
   两个块之间的间隔应该足够大。生成的操作码不必小于 256，因为它们从不被存储，
   所以我们将它们推到正常操作码之外。 */
#define OP_PROP_EXTRA       300
#define OP_EXTUNI_EXTRA     320
#define OP_ANYNL_EXTRA      340
#define OP_HSPACE_EXTRA     360
#define OP_VSPACE_EXTRA     380


/* 这个表标识了那些紧跟着要测试某种方式的字符的操作码。
   这使得可以集中加载这些字符。对于 Type * 等情况，"字符" 是 \D、\d、\S、\s、\W 或 \w 的操作码，
   这些操作码总是一个小的值。表中非零值是从操作码开始的偏移量，用于找到字符。 */
};

/* 这个表标识了那些检查字符的操作码。它用于记住当到达主题的末尾时可能已经检查了字符。
   ***注意*** 如果修改了这个表的开始部分，那么后面的三个表也必须修改。
/* 这两个表格也必须被修改。*/

};

/* 这两个表格允许紧凑的代码来测试 \D、\d、\S、\s、\W 和 \w */

static const uint8_t toptable1[] = {
  0, 0, 0, 0, 0, 0,
  ctype_digit, ctype_digit,
  ctype_space, ctype_space,
  ctype_word,  ctype_word,
  0, 0                            /* OP_ANY, OP_ALLANY */
};

static const uint8_t toptable2[] = {
  0, 0, 0, 0, 0, 0,
  ctype_digit, 0,
  ctype_space, 0,
  ctype_word,  0,
  1, 1                            /* OP_ANY, OP_ALLANY */
};


/* 用于保存有关特定状态的数据的结构，实际上是通过匹配树的活动路径的当前数据。它必须完全由 int 组成，因为我们传递的工作向量是一个 int 向量。 */

typedef struct stateblock {
  int offset;                     /* 操作码的偏移量（负数有意义） */
  int count;                      /* 重复计数 */
  int data;                       /* 一些使用额外数据 */
} stateblock;

#define INTS_PER_STATEBLOCK  (int)(sizeof(stateblock)/sizeof(int))


/* 在版本 10.32 之前，internal_dfa_match() 的递归调用传递了在堆栈上创建的本地工作空间和输出向量。这对于一些模式造成了问题，特别是在小堆栈环境（如 Windows）中。现在使用了一种新的方案，它在堆栈上设置一个向量，但如果这个向量太小，就会使用堆内存，直到堆限制。主要参数都是 int 的数量，因为工作空间是一个 int 向量。

起始堆栈向量的大小，DFA_START_RWS_SIZE，以字节为单位，在 pcre2_internal.h 中定义，以便在 pcre2test 找到匹配的最小堆需求时可用。*/

#define OVEC_UNIT  (sizeof(PCRE2_SIZE)/sizeof(int))

#define RWS_BASE_SIZE   (DFA_START_RWS_SIZE/sizeof(int))  /* 堆栈向量 */
#define RWS_RSIZE       1000                    /* 递归的工作大小 */
#define RWS_OVEC_RSIZE  (1000*OVEC_UNIT)        /* 定义递归时的Ovector大小 */
#define RWS_OVEC_OSIZE  (2*OVEC_UNIT)           /* 定义其他情况下的Ovector大小 */

/* 这个结构体位于每个工作空间块的开头。*/
typedef struct RWS_anchor {
  struct RWS_anchor *next;  /* 指向下一个工作空间块 */
  uint32_t size;  /* 整数的数量 */
  uint32_t free;  /* 可用的整数数量 */
} RWS_anchor;

#define RWS_ANCHOR_SIZE (sizeof(RWS_anchor)/sizeof(int))  /* 定义RWS_anchor结构体的大小 */

/*************************************************
*               处理调用                    *
*************************************************/

/* 这个函数用于执行调用。

参数:
  code              当前代码指针
  offsets           指向当前捕获偏移量
  current_subject   当前主题匹配的起始位置
  ptr               主题中的当前位置
  mb                匹配块
  extracode         从条件中调用时的额外代码偏移量
  lengthptr         用于返回调用长度的指针

返回:            调用的返回值
*/

static int
do_callout_dfa(PCRE2_SPTR code, PCRE2_SIZE *offsets, PCRE2_SPTR current_subject,
  PCRE2_SPTR ptr, dfa_match_block *mb, PCRE2_SIZE extracode,
  PCRE2_SIZE *lengthptr)
{
pcre2_callout_block *cb = mb->cb;  /* 获取匹配块的调用块 */

*lengthptr = (code[extracode] == OP_CALLOUT)?  /* 设置调用长度 */
  (PCRE2_SIZE)PRIV(OP_lengths)[OP_CALLOUT] :
  (PCRE2_SIZE)GET(code, 1 + 2*LINK_SIZE + extracode);

if (mb->callout == NULL) return 0;    /* 如果没有提供调用，则返回0 */

/* 调用块中的固定字段在匹配开始时设置一次。*/

cb->offset_vector    = offsets;  /* 设置偏移量向量 */
cb->start_match      = (PCRE2_SIZE)(current_subject - mb->start_subject);  /* 设置匹配开始位置 */
cb->current_position = (PCRE2_SIZE)(ptr - mb->start_subject);  /* 设置当前位置 */
cb->pattern_position = GET(code, 1 + extracode);  /* 设置模式位置 */
cb->next_item_length = GET(code, 1 + LINK_SIZE + extracode);  /* 设置下一个项目的长度 */
if (code[extracode] == OP_CALLOUT)
  {
  # 如果当前指令是调用外部函数
  cb->callout_number = code[1 + 2*LINK_SIZE + extracode];
  cb->callout_string_offset = 0;
  cb->callout_string = NULL;
  cb->callout_string_length = 0;
  }
else
  {
  # 如果当前指令不是调用外部函数
  cb->callout_number = 0;
  cb->callout_string_offset = GET(code, 1 + 3*LINK_SIZE + extracode);
  cb->callout_string = code + (1 + 4*LINK_SIZE + extracode) + 1;
  cb->callout_string_length = *lengthptr - (1 + 4*LINK_SIZE) - 2;
  }

# 调用外部函数
return (mb->callout)(cb, mb->callout_data);
}



/*************************************************
*         Expand local workspace memory          *
*************************************************/

/* This function is called when internal_dfa_match() is about to be called
recursively and there is insufficient working space left in the current
workspace block. If there's an existing next block, use it; otherwise get a new
block unless the heap limit is reached.

Arguments:
  rwsptr     pointer to block pointer (updated)
  ovecsize   space needed for an ovector
  mb         the match block

Returns:     0 rwsptr has been updated
            !0 an error code
*/

static int
more_workspace(RWS_anchor **rwsptr, unsigned int ovecsize, dfa_match_block *mb)
{
RWS_anchor *rws = *rwsptr;
RWS_anchor *new;

if (rws->next != NULL)
  {
  # 如果存在下一个工作空间块，则使用它
  new = rws->next;
  }

/* Sizes in the RWS_anchor blocks are in units of sizeof(int), but
mb->heap_limit and mb->heap_used are in kibibytes. Play carefully, to avoid
overflow. */

else
  {
  # 如果不存在下一个工作空间块
  uint32_t newsize = (rws->size >= UINT32_MAX/2)? UINT32_MAX/2 : rws->size * 2;
  uint32_t newsizeK = newsize/(1024/sizeof(int));

  if (newsizeK + mb->heap_used > mb->heap_limit)
    # 如果新的大小加上已使用的堆空间超过了堆限制
    newsizeK = (uint32_t)(mb->heap_limit - mb->heap_used);
  newsize = newsizeK*(1024/sizeof(int));

  if (newsize < RWS_RSIZE + ovecsize + RWS_ANCHOR_SIZE)
  # 如果新的大小小于指定的大小
    # 返回堆限制错误代码
    return PCRE2_ERROR_HEAPLIMIT;
  # 使用内存控制器分配新的内存空间
  new = mb->memctl.malloc(newsize*sizeof(int), mb->memctl.memory_data);
  # 如果分配失败，则返回内存不足错误代码
  if (new == NULL) return PCRE2_ERROR_NOMEMORY;
  # 更新已使用的堆内存大小
  mb->heap_used += newsizeK;
  # 设置新分配的内存块的下一个指针为 NULL
  new->next = NULL;
  # 设置新分配的内存块的大小
  new->size = newsize;
  # 将新分配的内存块添加到链表中
  rws->next = new;
  }
/* 设置新的空闲空间大小 */
new->free = new->size - RWS_ANCHOR_SIZE;
/* 将指针指向新的空间 */
*rwsptr = new;
/* 返回 0 表示成功 */
return 0;
}

/*************************************************
*     匹配正则表达式 - DFA 引擎                    *
*************************************************/

/* 这个内部函数使用 DFA 引擎将编译好的模式应用到一个主题字符串上，
从给定点开始。这个函数从外部函数调用，如果模式没有锚定，可能会被多次调用。
对于某些类型的子模式，该函数会递归调用自身。

参数：
  mb                包含固定信息的匹配数据块
  this_start_code   这个子表达式代码的开头括号
  current_subject   主题字符串中的当前位置
  start_offset      主题字符串中的起始偏移量
  offsets           用于存储匹配字符串偏移量的向量
  offsetcount       向量的大小
  workspace         工作空间的向量
  wscount           向量的大小
  rlevel            函数调用的递归级别

返回值：            > 0 => 将匹配偏移量对放入 offsets 中
                    = 0 => offsets 溢出；最长的匹配已经存在
                     -1 => 匹配失败
                   < -1 => 一些意外问题

以下的宏用于向两个状态向量（一个用于当前字符，一个用于下一个字符）中添加状态。 */

#define ADD_ACTIVE(x,y) \
  if (active_count++ < wscount) \
    { \
    next_active_state->offset = (x); \
    next_active_state->count  = (y); \
    next_active_state++; \
    } \
  else return PCRE2_ERROR_DFA_WSSIZE

#define ADD_ACTIVE_DATA(x,y,z) \
  if (active_count++ < wscount) \
    { \
    next_active_state->offset = (x); \
    next_active_state->count  = (y); \
    next_active_state->data   = (z); \
    next_active_state++; \
    } \
  else return PCRE2_ERROR_DFA_WSSIZE

#define ADD_NEW(x,y) \
  if (new_count++ < wscount) \
    { \
    next_new_state->offset = (x); \
    # 设置下一个新状态的计数值为 y
    next_new_state->count  = (y); \
    # 指针移动到下一个新状态
    next_new_state++; \
    # 如果条件不满足，则返回 DFA 的空间不足错误
  else return PCRE2_ERROR_DFA_WSSIZE
#define ADD_NEW_DATA(x,y,z) \  // 定义宏，用于向状态块中添加新数据
  if (new_count++ < wscount) \  // 如果新状态计数小于工作空间计数
    { \  // 进入代码块
    next_new_state->offset = (x); \  // 设置下一个新状态的偏移量为 x
    next_new_state->count  = (y); \  // 设置下一个新状态的计数为 y
    next_new_state->data   = (z); \  // 设置下一个新状态的数据为 z
    next_new_state++; \  // 指向下一个新状态
    } \  // 退出代码块
  else return PCRE2_ERROR_DFA_WSSIZE  // 如果新状态计数大于等于工作空间计数，返回错误

/* And now, here is the code */

static int
internal_dfa_match(
  dfa_match_block *mb,
  PCRE2_SPTR this_start_code,
  PCRE2_SPTR current_subject,
  PCRE2_SIZE start_offset,
  PCRE2_SIZE *offsets,
  uint32_t offsetcount,
  int *workspace,
  int wscount,
  uint32_t rlevel,
  int *RWS)
{
stateblock *active_states, *new_states, *temp_states;
stateblock *next_active_state, *next_new_state;
const uint8_t *ctypes, *lcc, *fcc;
PCRE2_SPTR ptr;
PCRE2_SPTR end_code;
dfa_recursion_info new_recursive;
int active_count, new_count, match_count;

/* Some fields in the mb block are frequently referenced, so we load them into
independent variables in the hope that this will perform better. */

PCRE2_SPTR start_subject = mb->start_subject;  // 将 mb 块中的 start_subject 字段加载到独立变量中
PCRE2_SPTR end_subject = mb->end_subject;  // 将 mb 块中的 end_subject 字段加载到独立变量中
PCRE2_SPTR start_code = mb->start_code;  // 将 mb 块中的 start_code 字段加载到独立变量中

#ifdef SUPPORT_UNICODE
BOOL utf = (mb->poptions & PCRE2_UTF) != 0;  // 检查是否支持 Unicode
BOOL utf_or_ucp = utf || (mb->poptions & PCRE2_UCP) != 0;  // 检查是否支持 Unicode 或 UCP
#else
BOOL utf = FALSE;  // 如果不支持 Unicode，则设置为 FALSE
#endif

BOOL reset_could_continue = FALSE;  // 初始化 reset_could_continue 为 FALSE

if (mb->match_call_count++ >= mb->match_limit) return PCRE2_ERROR_MATCHLIMIT;  // 如果匹配调用计数大于等于匹配限制，返回匹配限制错误
if (rlevel++ > mb->match_limit_depth) return PCRE2_ERROR_DEPTHLIMIT;  // 如果递归层级大于匹配深度限制，返回深度限制错误
offsetcount &= (uint32_t)(-2);  /* Round down */  // 对 offsetcount 进行位与操作，将其向下舍入为偶数

wscount -= 2;  // 减去 2
wscount = (wscount - (wscount % (INTS_PER_STATEBLOCK * 2))) /  // 计算工作空间计数
          (2 * INTS_PER_STATEBLOCK);

ctypes = mb->tables + ctypes_offset;  // 设置 ctypes 指向 mb 块中的 tables 字段加上 ctypes_offset 的位置
lcc = mb->tables + lcc_offset;  // 设置 lcc 指向 mb 块中的 tables 字段加上 lcc_offset 的位置
fcc = mb->tables + fcc_offset;  // 设置 fcc 指向 mb 块中的 tables 字段加上 fcc_offset 的位置

match_count = PCRE2_ERROR_NOMATCH;   /* A negative number */  // 初始化 match_count 为 PCRE2_ERROR_NOMATCH，表示无匹配

active_states = (stateblock *)(workspace + 2);  // 设置活动状态指针指向工作空间加 2 的位置
next_new_state = new_states = active_states + wscount;  // 设置下一个新状态指针和新状态指针都指向活动状态加上工作空间计数的位置
new_count = 0;  // 初始化新状态计数为 0

/* The first thing in any (sub) pattern is a bracket of some sort. Push all
the alternative states onto the list, and find out where the end is. This
# 如果我们要在匹配内部键而不是在结尾处停止时，使递归使用此函数成为可能。
if (*this_start_code == OP_ASSERTBACK || *this_start_code == OP_ASSERTBACK_NOT)
  {
  size_t max_back = 0;  # 初始化最大回退量
  size_t gone_back;  # 初始化已回退量

  end_code = this_start_code  # 将当前代码指针指向起始代码
  do
    {
    size_t back = (size_t)GET(end_code, 2+LINK_SIZE)  # 获取回退量
    if (back > max_back) max_back = back  # 更新最大回退量
    end_code += GET(end_code, 1)  # 移动代码指针到下一个位置
    }
  while (*end_code == OP_ALT)  # 当下一个代码是分支时继续循环

  # 如果我们无法回退到最长回溯模式所需的量，尽可能回退；某些分支可能仍然可行。
  # 在字符模式下，我们必须逐个字符地后退
  # 在字节模式下，我们可以快速完成这个过程。
  # 保存最早被查询的字符
  # 现在我们可以处理各个分支。每个分支的开头都会有一个OP_REVERSE，除非分支的长度为零。
  end_code = this_start_code  # 将代码指针重新指向起始代码
  do
    {
    uint32_t revlen = (end_code[1+LINK_SIZE] == OP_REVERSE)? 1 + LINK_SIZE : 0  # 获取反转长度
    size_t back = (revlen == 0)? 0 : (size_t)GET(end_code, 2+LINK_SIZE)  # 获取回退量
    # 如果回退位置小于等于已回退的位置
    if (back <= gone_back)
      {
      # 计算状态值
      int bstate = (int)(end_code - start_code + 1 + LINK_SIZE + revlen);
      # 添加新数据到数据结构中
      ADD_NEW_DATA(-bstate, 0, (int)(gone_back - back));
      }
    # 更新结束代码位置
    end_code += GET(end_code, 1);
    # 循环直到结束代码位置指向的内容不是 OP_ALT
    }
  while (*end_code == OP_ALT);
 }
/* 这是一个“正常”子模式的代码（不是反向断言）。整个模式的起始点总是其中一个。如果我们处于顶层，可能会被要求从先前部分匹配的相同点重新开始匹配。我们仍然必须扫描顶层分支，找到结束状态。 */
else
  {
  end_code = this_start_code;

  /* 重新开始 */

  if (rlevel == 1 && (mb->moptions & PCRE2_DFA_RESTART) != 0)
    {
    do { end_code += GET(end_code, 1); } while (*end_code == OP_ALT);
    new_count = workspace[1];
    if (!workspace[0])
      memcpy(new_states, active_states, (size_t)new_count * sizeof(stateblock));
    }

  /* 不重新开始 */

  else
    {
    int length = 1 + LINK_SIZE +
      ((*this_start_code == OP_CBRA || *this_start_code == OP_SCBRA ||
        *this_start_code == OP_CBRAPOS || *this_start_code == OP_SCBRAPOS)
        ? IMM2_SIZE:0);
    do
      {
      ADD_NEW((int)(end_code - start_code + length), 0);
      end_code += GET(end_code, 1);
      length = 1 + LINK_SIZE;
      }
    while (*end_code == OP_ALT);
    }
  }

workspace[0] = 0;    /* 用于指示当前向量的位 */

/* 循环扫描主题 */

ptr = current_subject;
# 无限循环，需要在循环内部使用 break 或者 return 来跳出循环
for (;;)
  {
  int i, j;  # 定义整型变量 i 和 j
  int clen, dlen;  # 定义整型变量 clen 和 dlen
  uint32_t c, d;  # 定义无符号整型变量 c 和 d
  int forced_fail = 0;  # 定义整型变量 forced_fail 并初始化为 0
  BOOL partial_newline = FALSE;  # 定义布尔变量 partial_newline 并初始化为 FALSE
  BOOL could_continue = reset_could_continue;  # 定义布尔变量 could_continue 并初始化为 reset_could_continue
  reset_could_continue = FALSE;  # 将 reset_could_continue 初始化为 FALSE

  if (ptr > mb->last_used_ptr) mb->last_used_ptr = ptr;  # 如果 ptr 大于 mb->last_used_ptr，则将 mb->last_used_ptr 设置为 ptr

  /* 将新状态列表变为活动状态列表并清空新状态列表。 */
  temp_states = active_states;  # 临时保存活动状态列表
  active_states = new_states;  # 将新状态列表变为活动状态列表
  new_states = temp_states;  # 将临时保存的活动状态列表赋值给新状态列表
  active_count = new_count;  # 将新状态数量赋值给活动状态数量
  new_count = 0;  # 将新状态数量清零

  workspace[0] ^= 1;  # 对 workspace[0] 进行异或操作
  workspace[1] = active_count;  # 将活动状态数量赋值给 workspace[1]

  /* 设置添加新状态的指针 */
  next_active_state = active_states + active_count;  # 设置下一个活动状态的指针
  next_new_state = new_states;  # 设置下一个新状态的指针

  /* 在循环外部加载主题中的当前字符，因为许多不同的状态可能希望查看它，我们假设至少有一个会。 */
  if (ptr < end_subject)  # 如果指针小于主题的结束位置
    {
    clen = 1;  # 字符中的数据项数
#ifdef SUPPORT_UNICODE
    GETCHARLENTEST(c, ptr, clen);  # 获取字符长度测试
#else
    c = *ptr;  # 将指针指向的值赋给 c
#endif  /* SUPPORT_UNICODE */
    }
  else
    {
    clen = 0;  # 这表示主题的结束
    c = NOTACHAR;  # 这个值实际上不应该被使用
    }

  /* 扫描活动状态并对每个状态进行操作。操作的结果可能是向当前活动列表添加更多状态（例如，击中括号），也可能是将状态放在新列表中，以便在移动字符指针时考虑。 */
  for (i = 0; i < active_count; i++)  # 遍历活动状态列表
    {
    stateblock *current_state = active_states + i;  # 获取当前状态
    BOOL caseless = FALSE;  # 定义布尔变量 caseless 并初始化为 FALSE
    PCRE2_SPTR code;  # 定义 PCRE2_SPTR 类型的变量 code
    uint32_t codevalue;  # 定义无符号整型变量 codevalue
    int state_offset = current_state->offset;  # 获取当前状态的偏移量
    int rrc;  # 定义整型变量 rrc
    int count;  # 定义整型变量 count
    /* 负偏移是一个特殊情况，意思是“等到数据字段中的字符数被跳过后再去这个（否定的）状态”。如果 could_continue 标志从先前传递过来 */
    /* 如果状态偏移小于0，表示需要传递状态 */
    if (state_offset < 0)
      {
      /* 如果当前状态的数据大于0，添加新数据到状态偏移，当前状态的计数和数据减1 */
        ADD_NEW_DATA(state_offset, current_state->count,
          current_state->data - 1);
        /* 如果可以继续，重置继续标志 */
        if (could_continue) reset_could_continue = TRUE;
        /* 继续下一轮循环 */
        continue;
      }
      else
      {
        /* 否则，设置当前状态的偏移为状态偏移的相反数 */
        current_state->offset = state_offset = -state_offset;
      }
      }

    /* 检查是否存在具有相同计数的重复状态，如果找到则跳过。
    请参阅本模块开头的注释，了解在这里可能提高性能的可能性。 */
    for (j = 0; j < i; j++)
      {
      if (active_states[j].offset == state_offset &&
          active_states[j].count == current_state->count)
        goto NEXT_ACTIVE_STATE;
      }

    /* 状态偏移是到操作码的偏移 */
    code = start_code + state_offset;
    codevalue = *code;

    /* 如果此操作码检查一个字符，但我们已经到达主题的末尾，记住这个事实，以便在测试部分匹配时使用。 */
    if (clen == 0 && poptable[codevalue] != 0)
      could_continue = TRUE;

    /* 如果此操作码后面跟着一个内联字符，加载它。在这里测试主题字符的存在是诱人的，但是错误的，因为有时允许主题的零重复。
    我们还使用这种机制来处理像OP_TYPEPLUS这样需要一个不是数据字符的参数的操作码 - 但是总是一个字节长，因为值很小。我们必须采取特殊措施来处理\P、\p、\H、\h、\V、\v和\X。为了保持其他情况的速度，将这些转换为新的操作码。 */
    if (coptable[codevalue] > 0)
      {
      dlen = 1;
#ifdef SUPPORT_UNICODE
      // 如果支持 Unicode，则根据 utf 标志获取字符长度
      if (utf) { GETCHARLEN(d, (code + coptable[codevalue]), dlen); } else
#endif  /* SUPPORT_UNICODE */
      // 否则直接获取字符
      d = code[coptable[codevalue]];
      // 如果操作码值大于等于 OP_TYPESTAR
      if (codevalue >= OP_TYPESTAR)
        {
        // 根据不同的操作码值进行处理
        switch(d)
          {
          case OP_ANYBYTE: return PCRE2_ERROR_DFA_UITEM;
          case OP_NOTPROP:
          case OP_PROP: codevalue += OP_PROP_EXTRA; break;
          case OP_ANYNL: codevalue += OP_ANYNL_EXTRA; break;
          case OP_EXTUNI: codevalue += OP_EXTUNI_EXTRA; break;
          case OP_NOT_HSPACE:
          case OP_HSPACE: codevalue += OP_HSPACE_EXTRA; break;
          case OP_NOT_VSPACE:
          case OP_VSPACE: codevalue += OP_VSPACE_EXTRA; break;
          default: break;
          }
        }
      }
    else
      {
      // 设置变量初始值
      dlen = 0;         /* Not strictly necessary, but compilers moan */
      d = NOTACHAR;     /* if these variables are not set. */
      }


    /* Now process the individual opcodes */

    // 根据操作码值进行处理
    switch (codevalue)
      {
/* ========================================================================== */
      // 这些情况永远不会被执行，用于在编译时检查向量的长度是否正确
      case OP_TABLE_LENGTH:
      case OP_TABLE_LENGTH +
        ((sizeof(coptable) == OP_TABLE_LENGTH) &&
         (sizeof(poptable) == OP_TABLE_LENGTH)):
      return 0;
/* ========================================================================== */
/* 到达一个闭合括号。如果不在模式的末尾，继续处理下一个操作码。对于重复操作码，还需要添加重复状态。注意，KETRPOS总是在子模式的末尾遇到，因为占有子模式重复总是使用递归调用处理。因此，它从不添加任何新状态。

在（子）模式的末尾，除非我们有一个空字符串并且设置了PCRE2_NOTEMPTY，或者设置了PCRE2_NOTEMPTY_ATSTART并且我们在主题的开头，保存匹配数据，将所有先前的匹配向上移动，以便始终具有最长的匹配。*/

case OP_KET:
case OP_KETRMIN:
case OP_KETRMAX:
case OP_KETRPOS:
if (code != end_code)
{
ADD_ACTIVE(state_offset + 1 + LINK_SIZE, 0);
if (codevalue != OP_KET)
{
ADD_ACTIVE(state_offset - (int)GET(code, 1), 0);
}
}
else
{
if (ptr > current_subject ||
((mb->moptions & PCRE2_NOTEMPTY) == 0 &&
((mb->moptions & PCRE2_NOTEMPTY_ATSTART) == 0 ||
current_subject > start_subject + mb->start_offset)))
{
if (match_count < 0) match_count = (offsetcount >= 2)? 1 : 0;
else if (match_count > 0 && ++match_count * 2 > (int)offsetcount)
match_count = 0;
count = ((match_count == 0)? (int)offsetcount : match_count * 2) - 2;
if (count > 0) (void)memmove(offsets + 2, offsets,
(size_t)count * sizeof(PCRE2_SIZE));
if (offsetcount >= 2)
{
offsets[0] = (PCRE2_SIZE)(current_subject - start_subject);
offsets[1] = (PCRE2_SIZE)(ptr - start_subject);
}
if ((mb->moptions & PCRE2_DFA_SHORTEST) != 0) return match_count;
}
}
break;
#if defined SUPPORT_UNICODE && PCRE2_CODE_UNIT_WIDTH != 32
          // 如果支持 Unicode 并且 PCRE2_CODE_UNIT_WIDTH 不等于 32
          if (utf) { BACKCHAR(temp); }
          // 如果 utf 为真，则回退一个字符
#endif
          GETCHARTEST(d, temp);
          // 获取字符测试
#ifdef SUPPORT_UNICODE
          // 如果支持 Unicode
          if ((mb->poptions & PCRE2_UCP) != 0)
            {
            // 如果选项中包含 PCRE2_UCP
            if (d == '_') left_word = TRUE; else
              {
              uint32_t cat = UCD_CATEGORY(d);
              left_word = (cat == ucp_L || cat == ucp_N);
              }
            }
          else
#endif
          left_word = d < 256 && (ctypes[d] & ctype_word) != 0;
          // 如果不支持 Unicode，则判断字符是否为字母或数字
          }
        else left_word = FALSE;

        if (clen > 0)
          {
          // 如果 clength 大于 0
          if (ptr >= mb->last_used_ptr)
            {
            // 如果指针大于等于上次使用的指针
            PCRE2_SPTR temp = ptr + 1;
#if defined SUPPORT_UNICODE && PCRE2_CODE_UNIT_WIDTH != 32
            // 如果支持 Unicode 并且 PCRE2_CODE_UNIT_WIDTH 不等于 32
            if (utf) { FORWARDCHARTEST(temp, mb->end_subject); }
            // 如果 utf 为真，则前进一个字符测试
#endif
            mb->last_used_ptr = temp;
            // 更新上次使用的指针
            }
#ifdef SUPPORT_UNICODE
          // 如果支持 Unicode
          if ((mb->poptions & PCRE2_UCP) != 0)
            {
            // 如果选项中包含 PCRE2_UCP
            if (c == '_') right_word = TRUE; else
              {
              uint32_t cat = UCD_CATEGORY(c);
              right_word = (cat == ucp_L || cat == ucp_N);
              }
            }
          else
#endif
          right_word = c < 256 && (ctypes[c] & ctype_word) != 0;
          // 如果不支持 Unicode，则判断字符是否为字母或数字
          }
        else right_word = FALSE;

        if ((left_word == right_word) == (codevalue == OP_NOT_WORD_BOUNDARY))
          { ADD_ACTIVE(state_offset + 1, 0); }
          // 如果左边字符和右边字符的字边界状态与 codevalue 的非字边界状态相同，则添加活动状态
        }
      break;


      /*-----------------------------------------------------------------*/
      /* Check the next character by Unicode property. We will get here only
      if the support is in the binary; otherwise a compile-time error occurs.
      */
/* ========================================================================== */
      /* 这些是虚拟操作码，用于当类似 OP_TYPEPLUS 的操作码的参数为 OP_PROP、OP_NOTPROP、OP_ANYNL 或 OP_EXTUNI 时使用。
      它使得上面的代码对于其他情况保持快速。参数存储在变量 d 中。 */

#endif

      /*-----------------------------------------------------------------*/
      case OP_ANYNL_EXTRA + OP_TYPEPLUS:
      case OP_ANYNL_EXTRA + OP_TYPEMINPLUS:
      case OP_ANYNL_EXTRA + OP_TYPEPOSPLUS:
      count = current_state->count;  /* 已经匹配 */
      if (count > 0) { ADD_ACTIVE(state_offset + 2, 0); }
      if (clen > 0)
        {
        int ncount = 0;
        switch (c)
          {
          case CHAR_VT:
          case CHAR_FF:
          case CHAR_NEL:
#ifndef EBCDIC
          case 0x2028:
          case 0x2029:
#endif

      /*-----------------------------------------------------------------*/
      case OP_ANYNL_EXTRA + OP_TYPEQUERY:
      case OP_ANYNL_EXTRA + OP_TYPEMINQUERY:
      case OP_ANYNL_EXTRA + OP_TYPEPOSQUERY:
      count = 2;
      goto QS3;

      case OP_ANYNL_EXTRA + OP_TYPESTAR:
      case OP_ANYNL_EXTRA + OP_TYPEMINSTAR:
      case OP_ANYNL_EXTRA + OP_TYPEPOSSTAR:
      count = 0;

      QS3:
      ADD_ACTIVE(state_offset + 2, 0);
      if (clen > 0)
        {
        int ncount = 0;
        switch (c)
          {
          case CHAR_VT:
          case CHAR_FF:
          case CHAR_NEL:
#ifndef EBCDIC
          case 0x2028:
          case 0x2029:
#endif
      /*-----------------------------------------------------------------*/
      # 如果当前操作码是 OP_ANYNL_EXTRA + OP_TYPEEXACT 或者 OP_ANYNL_EXTRA + OP_TYPEUPTO 或者 OP_ANYNL_EXTRA + OP_TYPEMINUPTO 或者 OP_ANYNL_EXTRA + OP_TYPEPOSUPTO
      case OP_ANYNL_EXTRA + OP_TYPEEXACT:
      case OP_ANYNL_EXTRA + OP_TYPEUPTO:
      case OP_ANYNL_EXTRA + OP_TYPEMINUPTO:
      case OP_ANYNL_EXTRA + OP_TYPEPOSUPTO:
      # 如果当前操作码不等于 OP_ANYNL_EXTRA + OP_TYPEEXACT
      if (codevalue != OP_ANYNL_EXTRA + OP_TYPEEXACT)
        { ADD_ACTIVE(state_offset + 2 + IMM2_SIZE, 0); }
      # 已经匹配的次数
      count = current_state->count;  /* Number already matched */
      # 如果字符长度大于0
      if (clen > 0)
        {
        int ncount = 0;
        # 根据字符类型进行不同的处理
        switch (c)
          {
          # 如果字符是垂直制表符、换页符、下一行
          case CHAR_VT:
          case CHAR_FF:
          case CHAR_NEL:
          #ifndef EBCDIC
          # 如果字符是 0x2028 或者 0x2029
          case 0x2028:
          case 0x2029:
/* ========================================================================== */
      /* 这些操作码后面跟着的是通常与当前主题字符进行比较的字符；它被加载到 d 中。即使没有主题字符，我们仍然会到达这里，因为在某些情况下允许零重复。 */

      /*-----------------------------------------------------------------*/
      # 如果当前操作码是 OP_CHAR
      case OP_CHAR:
      # 如果字符长度大于0且字符等于 d，则添加新的活动状态
      if (clen > 0 && c == d) { ADD_NEW(state_offset + dlen + 1, 0); }
      break;

      /*-----------------------------------------------------------------*/
      # 如果当前操作码是 OP_CHARI
      case OP_CHARI:
      # 如果字符长度为0，则跳出循环
      if (clen == 0) break;
      # 如果支持 Unicode
      # 如果字符等于 d，则添加新的活动状态；否则，根据不同情况进行处理
      # 如果不是 UTF 或 UCP 模式
      # 如果字符的小写形式等于 d 的小写形式，则添加新的活动状态
      # 如果支持 Unicode
      # 如果字符等于 d，则添加新的活动状态；否则，根据不同情况进行处理
      # 如果不是 UTF 或 UCP 模式
      # 如果字符的小写形式等于 d 的小写形式，则添加新的活动状态
      # 如果字符的其他形式等于 d 的其他形式，则添加新的活动状态
      # 如果字符的小写形式等于 d 的小写形式，则添加新的活动状态
      # 如果字符的其他形式等于 d 的其他形式，则添加新的活动状态
      # 如果字符的小写形式等于 d 的小写形式，则添加新的活动状态
      # 如果字符的其他形式等于 d 的其他形式，则添加新的活动状态
      # 如果字符的小写形式等于 d 的小写形式，则添加新的活动状态
      # 如果字符的其他形式等于 d 的其他形式，则添加新的活动状态
      # 如果字符的小写形式等于 d 的小写形式，则添加新的活动状态
      # 如果字符的其他形式等于 d 的其他形式，则添加新的活动状态
      # 如果字符的小写形式等于 d 的小写形式，则添加新的活动状态
      # 如果字符的其他形式等于 d 的其他形式，则添加新的活动状态
      # 如果字符的小写形式等于 d 的小写形式，则添加新的活动状态
      # 如果字符的其他形式等于 d 的其他形式，则添加新的活动状态
      # 如果字符的小写形式等于 d 的小写形式，则添加新的活动状态
      # 如果字符的其他形式等于 d 的其他形式，则添加新的活动状态
      # 如果字符的小写形式等于 d 的小写形式，则添加新的活动状态
      # 如果字符的其他形式等于 d 的其他形式，则添加新的活动状态
      # 如果字符的小写形式等于 d 的小写形式，则添加新的活动状态
      # 如果字符的其他形式等于 d 的其他形式，则添加新的活动状态
      # 如果字符的小写形式等于 d 的小写形式，则添加新的活动状态
      # 如果字符的其他形式等于 d 的其他形式，则添加新的活动状态
      # 如果字符的小写形式等于 d 的小写形式，则添加新的活动状态
      # 如果字符的其他形式等于 d 的其他形式，则添加新的活动状态
      # 如果字符的小写形式等于 d 的小写形式，则添加新的活动状态
      # 如果字符的其他形式等于 d 的其他形式，则添加新的活动状态
      # 如果字符的小写形式等于 d 的小写形式，则添加新的活动状态
      # 如果字符的其他形式等于 d 的其他形式，则添加新的活动状态
      # 如果字符的小写形式等于 d 的小写形式，则添加新的活动状态
      # 如果字符的其他形式等于 d 的其他形式，则添加新的活动状态
      # 如果字符的小写形式等于 d 的小写形式，则添加新的活动状态
      # 如果字符的其他形式等于 d 的其他形式，则添加新的活动状态
      # 如果字符的小写形式等于 d 的小写形式，则添加新的活动状态
      # 如果字符的其他形式等于 d 的其他形式，则添加新的活动状态
      # 如果字符的小写形式等于 d 的小写形式，则添加新的活动状态
      # 如果字符的其他形式等于 d 的其他形式，则添加新的活动状态
      # 如果字符的小写形式等于 d 的小写形式，则添加新的活动状态
      # 如果字符的其他形式等于 d 的其他形式，则添加新的活动状态
      # 如果字符的小写形式等于 d 的小写形式，则添加新的活动状态
      # 如果字符的其他形式等于 d 的其他形式，则添加新的活动状态
      # 如果字符的小写形式等于 d 的小写形式，则添加新的活动状态
      # 如果字符的其他形式等于 d 的其他形式，则添加新的活动状态
      # 如果字符的小写形式等于 d 的小写形式，则添加新的活动状态
      # 如果字符的其他形式等于 d 的其他形式，则添加新的活动状态
      # 如果字符的小写形式等于 d 的小写形式，则添加新的活动状态
      # 如果字符的其他形式等于 d 的其他形式，则添加新的活动状态
      # 如果字符的小写形式等于 d 的小写形式，则添加新的活动状态
      # 如果字符的其他形式等于 d 的其他形式，则添加新的活动状态
      # 如果字符的小写形式等于 d 的小写形式，则添加新的活动状态
      # 如果字符的其他形式等于 d 的其他形式，则添加新的活动状态
      # 如果字符的小写形式等于 d 的小写形式，则添加新的活动状态
      # 如果字符的其他形式等于 d 的其他形式，则添加新的活动状态
      # 如果字符的小写形式等于 d 的小写形式，则添加新的活动状态
      # 如果字符的其他形式等于 d 的其他形式，则添加新的活动状态
      # 如果字符的小写形式等于 d 的小写形式，则添加新的活动状态
      # 如果字符的其他形式等于 d 的其他形式，则添加新的活动状态
      # 如果字符的小写形式等于 d 的小写形式，则添加新的活动状态
      # 如果字符的其他形式等于 d 的其他形式，则添加新的活动状态
      # 如果字符的小写形式等于 d 的小写形式，则添加新的活动状态
      # 如果字符的其他形式等于 d 的其他形式，则添加新的活动状态
      # 如果字符的小写形式等于 d 的小写形式，则添加新的活动状态
      # 如果字符的其他形式等于 d 的其他形式，则添加新的活动状态
      # 如果字符的小写形式等于 d 的小写形式，则添加新的活动状态
      # 如果字符的其他形式等于 d 的其他形式，则添加新的活动状态
      # 如果字符的小写形式等于 d 的小写形式，则添加新的活动状态
      # 如果字符的其他形式等于 d 的其他形式，则添加新的活动状态
      # 如果字符的小写形式等于 d 的小写形式，则添加新的活动状态
      # 如果字符的其他形式等于 d 的其他形式，则添加新的活动状态
      # 如果字符的小写形式等于 d 的小写形式，则添加新的活动状态
      # 如果字符的其他形式等于 d 的其他形式，则添加新的活动状态
      # 如果字符的小写形式等于 d 的小写形式，则添加新的活动状态
      # 如果字符的其他形式等于 d 的其他形式，则添加新的活动状态
      # 如果字符的小写形式等于 d 的小写形式，则添加新的活动状态
      # 如果字符的其他形式等于 d 的其他形式，则添加新的活动状态
      # 如果字符的小写形式等于 d 的小写形式，则添加新的活动状态
      # 如果字符的其他形式等于 d 的其他形式，则添加新的活动状态
      # 如果字符的小写形式等于 d 的小写形式，则添加新的活动状态
      # 如果字符的其他形式等于 d 的其他形式，则添加新的活动状态
      # 如果字符的小写形式等于 d 的小写形式，则添加新的活动状态
      # 如果字符的其他形式等于 d 的其他形式，则添加新的活动状态
      # 如果字符的小写形式等于 d 的小写形式，则添加新的活动状态
      # 如果字符的其他形式等于 d 的其他形式，则添加新的活动状态
      # 如果字符的小写形式等于 d 的小写形式，则添加新的活动状态
      # 如果字符的其他形式等于 d 的其他形式，则添加新的活动状态
      # 如果字符的小写形式等于 d 的小写形式，则添加新的活动状态
      # 如果字符的其他形式等于 d 的其他形式，则添加新的活动状态
      # 如果字符的小写形式等于 d 的小写形式，则添加新的活动状态
      # 如果字符的其他形式等于 d 的其他形式，则添加新的活动状态
      # 如果字符的小写形式等于 d 的小写形式，则添加新的活动状态
      # 如果字符的其他形式等于 d 的其他形式，则添加新的活动状态
      # 如果字符的小写形式等于 d 的小写形式，则添加新的活动状态
      # 如果字符的其他形式等于 d 的其他形式，则添加新的活动状态
      # 如果字符的小写形式等于 d 的小写形式，则添加新的活动状态
      # 如果字符的其他形式等于 d 的其他形式，则添加新的活动状态
      # 如果字符的小写形式等于 d 的小写形式，则添加新的活动状态
      # 如果字符的其他形式等于 d 的其他形式，则添加新的活动状态
      # 如果字符的小写形式等于 d 的小写形式，则添加新的活动状态
      # 如果字符的其他形式等于 d 的其他形式，则添加新的活动状态
      # 如果字符的小写形式等于 d 的小写形式，则添加新的活动状态
      # 如果字符的其他形式等于 d 的其他形式，则添加新的活动状态
      # 如果字符的小写形式等于 d 的小写形式，则添加新的活动状态
      # 如果字符的其他形式等于 d 的其他形式，则添加新的活动状态
      # 如果字符的小写形式等于 d 的小写形式，则添加新的活动状态
      # 如果字符的其他形式等于 d 的其他形式，则添加新的活动状态
      # 如果字符的小写形式等于 d 的小写形式，则添加新的活动状态
      # 如果字符的其他形式等于 d 的其他形式，则添加新的活动状态
      # 如果字符的小写形式等于 d 的小写形式，则添加新的活动状态
      # 如果字符的其他形式等于 d 的其他形式，则添加新的活动状态
      # 如果字符的小写形式等于 d 的小写形式，则添加新的活动状态
      # 如果字符的其他形式等于 d 的其他形式，则添加新的活动状态
      # 如果字符的小写形式等于 d 的小写形式，则添加新的活动状态
      # 如果字符的其他形式等于 d 的其他形式，则添加新的活动状态
      # 如果字符的小写形式等于 d 的小写形式，则添加新的活动状态
      # 如果字符的其他形式等于 d 的其他形式，则添加新的活动状态
      # 如果字符的小写形式等于 d 的小写形式，则添加新的活动状态
      # 如果字符的其他形式等于 d 的其他形式，则添加新的活动状态
      # 如果字符的小写形式等于 d 的小写形式，则添加新的活动状态
      # 如果字符的其他形式等于 d 的其他形式，则添加新的活动状态
      # 如果字符的小写形式等于 d 的小写形式，则添加新的活动状态
      # 如果字符的其他形式等于 d 的其他形式，则添加新的活动状态
      # 如果字符的小写形式等于 d 的小写形式，则添加新的活动状态
      # 如果字符的其他形式等于 d 的其他形式，则添加新的活动状态
      # 如果字符的小写形式等于 d 的小写形式，则添加新的活动状态
      # 如果字符的其他形式等于 d 的其他形式，则添加新的活动状态
      # 如果字符的小写形式等于 d 的小写形式，则添加新的活动状态
      # 如果字符的其他形式等于 d 的其他形式，则添加新的活动状态
      # 如果字符的小写形式等于 d 的小写形式，则添加新的活动状态
      # 如果字符的其他形式等于 d 的其他形式，则添加新的活动状态
      # 如果字符的小写形式等于 d 的小写形式，则添加新的活动状态
      # 如果字符的其他形式等于 d 的其他形式，则添加新的活动状态
      # 如果字符的小写形式等于 d 的小写形式，则添加新的活动状态
      # 如果字符的其他形式等于 d 的其他形式，则添加新的活动状态
      # 如果字符的小写形式等于 d 的小写形式，则添加新的活动状态
      # 如果字符的其他形式等于 d 的其他形式，则添加新的活动状态
      # 如果字符的小写形式等于 d 的小写形式，则添加新的活动状态
      # 如果字符的其他形式等于 d 的其他形式，则添加新的活动状态
      # 如果字符的小写形式等于 d 的小写形式，则添加新的活动状态
      # 如果字符的其他形式等于 d 的其他形式，则添加新的活动状态
      # 如果字符的小写形式等于 d 的小写形式，则添加新的活动状态
      # 如果字符的其他形式等于 d 的其他形式，则添加新的活动状态
      # 如果字符的小写形式等于 d 的小写形式，则添加新的活动状态
      # 如果字符的其他形式等于 d 的其他形式，则添加新的活动状态
      # 如果字符的小写形式等于 d 的小写形式，则添加新的活动状态
      # 如果字符的其他形式等于 d 的其他形式，则添加新
#ifdef SUPPORT_UNICODE
      /*-----------------------------------------------------------------*/
      /* 如果支持 Unicode，则执行以下操作 */
      /* 这是一个棘手的情况，因为它可以匹配多个字符。
      找出要跳过多少个字符，然后设置一个负状态来等待它们通过后继续。 */

      case OP_EXTUNI:
      if (clen > 0)
        {
        int ncount = 0;
        PCRE2_SPTR nptr = PRIV(extuni)(c, ptr + clen, mb->start_subject,
          end_subject, utf, &ncount);
        if (nptr >= end_subject && (mb->moptions & PCRE2_PARTIAL_HARD) != 0)
            reset_could_continue = TRUE;
        ADD_NEW_DATA(-(state_offset + 1), 0, ncount);
        }
      break;
#endif

      /*-----------------------------------------------------------------*/
      /* 这是一个棘手的情况，类似于 EXTUNI，因为它也可以匹配多个字符（当 CR 后面跟着 LF 时）。
      在这种情况下，设置一个负状态来等待一个字符通过后继续。 */

      case OP_ANYNL:
      if (clen > 0) switch(c)
        {
        case CHAR_VT:
        case CHAR_FF:
        case CHAR_NEL:
#ifndef EBCDIC
        case 0x2028:
        case 0x2029:
#ifdef SUPPORT_UNICODE
        if (utf_or_ucp && d >= 128)
          otherd = UCD_OTHERCASE(d);
        else
#endif  /* SUPPORT_UNICODE */
        // 获取字符 d 对应的其他字符
        otherd = TABLE_GET(d, fcc, d);
        // 如果字符 c 不等于字符 d 且不等于其他字符 otherd
        if (c != d && c != otherd)
          { ADD_NEW(state_offset + dlen + 1, 0); }
        }
      break;

      /*-----------------------------------------------------------------*/
      // 不区分大小写的情况下
      case OP_PLUSI:
      case OP_MINPLUSI:
      case OP_POSPLUSI:
      case OP_NOTPLUSI:
      case OP_NOTMINPLUSI:
      case OP_NOTPOSPLUSI:
      caseless = TRUE;
      // 减去 OP_STARI - OP_STAR 的值
      codevalue -= OP_STARI - OP_STAR;

      /* Fall through */
      // 下面的情况都是不区分大小写的情况
      case OP_PLUS:
      case OP_MINPLUS:
      case OP_POSPLUS:
      case OP_NOTPLUS:
      case OP_NOTMINPLUS:
      case OP_NOTPOSPLUS:
      // 获取当前状态的匹配次数
      count = current_state->count;  /* Already matched */
      // 如果匹配次数大于 0，则添加活动状态
      if (count > 0) { ADD_ACTIVE(state_offset + dlen + 1, 0); }
      // 如果 clength 大于 0
      if (clen > 0)
        {
        // 初始化 otherd 为 NOTACHAR
        uint32_t otherd = NOTACHAR;
        // 如果是不区分大小写的情况
        if (caseless)
          {
#ifdef SUPPORT_UNICODE
          // 如果是 UTF 或 UCP 并且 d 大于等于 128
          if (utf_or_ucp && d >= 128)
            // 获取 d 的其他大小写形式
            otherd = UCD_OTHERCASE(d);
          else
#endif  /* SUPPORT_UNICODE */
          // 如果不支持 Unicode，则获取其他字符的值
          otherd = TABLE_GET(d, fcc, d);
          }
        // 如果当前字符等于 d 或者等于 otherd，并且 codevalue 小于 OP_NOTSTAR，则执行以下操作
        if ((c == d || c == otherd) == (codevalue < OP_NOTSTAR))
          {
          // 如果 count 大于 0 并且 codevalue 等于 OP_POSPLUS 或者 OP_NOTPOSPLUS，则执行以下操作
          if (count > 0 &&
              (codevalue == OP_POSPLUS || codevalue == OP_NOTPOSPLUS))
            {
            // 移除非匹配的可能性
            active_count--;
            next_active_state--;
            }
          count++;
          // 添加新的状态偏移和计数
          ADD_NEW(state_offset, count);
          }
        }
      break;

      /*-----------------------------------------------------------------*/
      // 根据不同的操作码执行不同的操作
      case OP_QUERYI:
      case OP_MINQUERYI:
      case OP_POSQUERYI:
      case OP_NOTQUERYI:
      case OP_NOTMINQUERYI:
      case OP_NOTPOSQUERYI:
      // 忽略大小写
      caseless = TRUE;
      // 从操作码中减去 OP_STARI - OP_STAR
      codevalue -= OP_STARI - OP_STAR;
      /* Fall through */
      case OP_QUERY:
      case OP_MINQUERY:
      case OP_POSQUERY:
      case OP_NOTQUERY:
      case OP_NOTMINQUERY:
      case OP_NOTPOSQUERY:
      // 添加活动状态
      ADD_ACTIVE(state_offset + dlen + 1, 0);
      // 如果 clen 大于 0，则执行以下操作
      if (clen > 0)
        {
        // 其他字符的值初始化为 NOTACHAR
        uint32_t otherd = NOTACHAR;
        // 如果忽略大小写
        if (caseless)
          {
#ifdef SUPPORT_UNICODE
          // 如果支持 Unicode 并且 d 大于等于 128，则获取其他大小写形式的字符值
          if (utf_or_ucp && d >= 128)
            otherd = UCD_OTHERCASE(d);
          else
#else  /* SUPPORT_UNICODE */
          // 如果不支持 Unicode，则将 otherd 设置为表 d 中 fcc 对应的值
          otherd = TABLE_GET(d, fcc, d);
          }
        // 如果 c 等于 d 或者 c 等于 otherd，且 codevalue 小于 OP_NOTSTAR，则执行以下代码块
        if ((c == d || c == otherd) == (codevalue < OP_NOTSTAR))
          {
          // 如果 codevalue 等于 OP_POSQUERY 或者 OP_NOTPOSQUERY，则执行以下代码块
          if (codevalue == OP_POSQUERY || codevalue == OP_NOTPOSQUERY)
            {
            // 移除非匹配的可能性
            active_count--;
            next_active_state--;
            }
          // 添加新的状态
          ADD_NEW(state_offset + dlen + 1, 0);
          }
        }
      break;

      /*-----------------------------------------------------------------*/
      // 根据不同的操作码执行不同的操作
      case OP_STARI:
      case OP_MINSTARI:
      case OP_POSSTARI:
      case OP_NOTSTARI:
      case OP_NOTMINSTARI:
      case OP_NOTPOSSTARI:
      // 设置 caseless 为真
      caseless = TRUE;
      // 将 codevalue 减去 OP_STARI - OP_STAR
      codevalue -= OP_STARI - OP_STAR;
      /* Fall through */
      // 执行以下操作
      case OP_STAR:
      case OP_MINSTAR:
      case OP_POSSTAR:
      case OP_NOTSTAR:
      case OP_NOTMINSTAR:
      case OP_NOTPOSSTAR:
      // 添加新的活动状态
      ADD_ACTIVE(state_offset + dlen + 1, 0);
      // 如果 clen 大于 0，则执行以下操作
      if (clen > 0)
        {
        // 设置 otherd 为 NOTACHAR
        uint32_t otherd = NOTACHAR;
        // 如果 caseless 为真，则执行以下操作
        if (caseless)
          {
#ifdef SUPPORT_UNICODE
          // 如果支持 Unicode 并且 d 大于等于 128，则将 otherd 设置为 UCD_OTHERCASE(d)
          if (utf_or_ucp && d >= 128)
            otherd = UCD_OTHERCASE(d);
          else
#endif  /* SUPPORT_UNICODE */
          // 如果不支持 Unicode，则使用默认的表格获取字符对应的值
          otherd = TABLE_GET(d, fcc, d);
          }
        // 如果当前字符等于 d 或者等于 otherd，则判断 codevalue 是否小于 OP_NOTSTAR
        if ((c == d || c == otherd) == (codevalue < OP_NOTSTAR))
          {
          // 如果 codevalue 是 OP_POSSTAR 或者 OP_NOTPOSSTAR，则减少活跃状态计数
          if (codevalue == OP_POSSTAR || codevalue == OP_NOTPOSSTAR)
            {
            active_count--;            /* 移除非匹配可能性 */
            next_active_state--;
            }
          // 添加新的状态偏移和计数
          ADD_NEW(state_offset, 0);
          }
        }
      break;

      /*-----------------------------------------------------------------*/
      // 匹配大小写不敏感的精确字符串
      case OP_EXACTI:
      case OP_NOTEXACTI:
      caseless = TRUE;
      // 将 codevalue 转换为对应的大小写不敏感的操作码
      codevalue -= OP_STARI - OP_STAR;
      /* Fall through */
      // 匹配精确字符串
      case OP_EXACT:
      case OP_NOTEXACT:
      // 获取当前状态已匹配的数量
      count = current_state->count;  /* 已匹配的数量 */
      // 如果 clen 大于 0
      if (clen > 0)
        {
        uint32_t otherd = NOTACHAR;
        // 如果大小写不敏感
        if (caseless)
          {
#ifdef SUPPORT_UNICODE
          // 如果支持 Unicode 并且 d 大于等于 128
          if (utf_or_ucp && d >= 128)
            // 获取 d 的其他大小写形式
            otherd = UCD_OTHERCASE(d);
          else
#else  /* SUPPORT_UNICODE */
          otherd = TABLE_GET(d, fcc, d);
          }
        if ((c == d || c == otherd) == (codevalue < OP_NOTSTAR))
          {
          if (++count >= (int)GET2(code, 1))
            { ADD_NEW(state_offset + dlen + 1 + IMM2_SIZE, 0); }
          else
            { ADD_NEW(state_offset, count); }
          }
        }
      break;

      /*-----------------------------------------------------------------*/
      case OP_UPTOI:
      case OP_MINUPTOI:
      case OP_POSUPTOI:
      case OP_NOTUPTOI:
      case OP_NOTMINUPTOI:
      case OP_NOTPOSUPTOI:
      caseless = TRUE;
      codevalue -= OP_STARI - OP_STAR;
      /* Fall through */
      case OP_UPTO:
      case OP_MINUPTO:
      case OP_POSUPTO:
      case OP_NOTUPTO:
      case OP_NOTMINUPTO:
      case OP_NOTPOSUPTO:
      ADD_ACTIVE(state_offset + dlen + 1 + IMM2_SIZE, 0);
      count = current_state->count;  /* Number already matched */
      if (clen > 0)
        {
        uint32_t otherd = NOTACHAR;
        if (caseless)
          {
#ifdef SUPPORT_UNICODE
          if (utf_or_ucp && d >= 128)
            otherd = UCD_OTHERCASE(d);
          else
#endif  /* SUPPORT_UNICODE */
          otherd = TABLE_GET(d, fcc, d);
          }
        if ((c == d || c == otherd) == (codevalue < OP_NOTSTAR))
          {
          if (codevalue == OP_POSUPTO || codevalue == OP_NOTPOSUPTO)
            {
            active_count--;             /* Remove non-match possibility */
            next_active_state--;
            }
          if (++count >= (int)GET2(code, 1))
            { ADD_NEW(state_offset + dlen + 1 + IMM2_SIZE, 0); }
          else
            { ADD_NEW(state_offset, count); }
          }
        }
      break;
#if defined SUPPORT_UNICODE && PCRE2_CODE_UNIT_WIDTH != 32
    // 如果支持 Unicode 并且 PCRE2_CODE_UNIT_WIDTH 不等于 32
    if (utf)
      {
      // 如果是 UTF 编码，则对指定范围内的字符进行处理
      PCRE2_SPTR p = start_subject + local_offsets[rc];
      PCRE2_SPTR pp = start_subject + local_offsets[rc+1];
      while (p < pp) if (NOT_FIRSTCU(*p++)) charcount--;
      }
#if defined SUPPORT_UNICODE && PCRE2_CODE_UNIT_WIDTH != 32
    // 如果支持 Unicode 并且 PCRE2_CODE_UNIT_WIDTH 不等于 32，对指定范围内的字符进行处理
    if (utf) while (p < pp) if (NOT_FIRSTCU(*p++)) charcount--;
#if defined SUPPORT_UNICODE && PCRE2_CODE_UNIT_WIDTH != 32
    // 如果支持 Unicode 并且 PCRE2_CODE_UNIT_WIDTH 不等于 32
    if (utf)
      {
      // 如果是 UTF 编码，则对指定范围内的字符进行处理
      PCRE2_SPTR p = start_subject + local_offsets[0];
      PCRE2_SPTR pp = start_subject + local_offsets[1];
      while (p < pp) if (NOT_FIRSTCU(*p++)) charcount--;
      }
#endif
    // 添加新的数据到状态
    ADD_NEW_DATA(-next_state_offset, 0, (int)(charcount - 1));
    // 如果重复状态偏移大于等于 0，则添加新的数据到状态
    if (repeat_state_offset >= 0)
      { ADD_NEW_DATA(-repeat_state_offset, 0, (int)(charcount - 1)); }
    }
  }
else if (rc != PCRE2_ERROR_NOMATCH) return rc;
}
break;

/* ========================================================================== */
  /* 处理调用 */

  case OP_CALLOUT:
  case OP_CALLOUT_STR:
    {
    // 定义调用长度
    PCRE2_SIZE callout_length;
    // 调用 do_callout_dfa 函数处理调用
    rrc = do_callout_dfa(code, offsets, current_subject, ptr, mb, 0,
      &callout_length);
    // 如果返回值小于 0，则放弃
    if (rrc < 0) return rrc;   /* Abandon */
    // 如果返回值等于 0，则添加活动状态
    if (rrc == 0)
      { ADD_ACTIVE(state_offset + (int)callout_length, 0); }
    }
break;

/* ========================================================================== */
  // 默认情况，不支持的操作码
  default:        /* Unsupported opcode */
  return PCRE2_ERROR_DFA_UITEM;
  }

NEXT_ACTIVE_STATE: continue;
    }      /* End of loop scanning active states */

  /* We have finished the processing at the current subject character. If no
  new states have been set for the next character, we have found all the
  matches that we are going to find. If partial matching has been requested,
  check for appropriate conditions.

  The "forced_ fail" variable counts the number of (*F) encountered for the
  character. If it is equal to the original active_count (saved in
  workspace[1]) it means that (*F) was found on every active state. In this
  case we don't want to give a partial match.

  The "could_continue" variable is true if a state could have continued but
  for the fact that the end of the subject was reached. */

  if (new_count <= 0)
    {
    if (could_continue &&                            /* Some could go on, and */
        forced_fail != workspace[1] &&               /* Not all forced fail & */
        (                                            /* either... */
        (mb->moptions & PCRE2_PARTIAL_HARD) != 0      /* Hard partial */
        ||                                           /* or... */
        ((mb->moptions & PCRE2_PARTIAL_SOFT) != 0 &&  /* Soft partial and */
         match_count < 0)                             /* no matches */
        ) &&                                         /* And... */
        (
        partial_newline ||                   /* Either partial NL */
          (                                  /* or ... */
          ptr >= end_subject &&              /* End of subject and */
            (                                  /* either */
            ptr > mb->start_used_ptr ||        /* Inspected non-empty string */
            mb->allowemptypartial              /* or pattern has lookbehind */
            )                                  /* or could match empty */
          )
        ))
      match_count = PCRE2_ERROR_PARTIAL;
    break;  /* Exit from loop along the subject string */
    }
    /* 一个或多个状态对下一个字符是活动的 */

    ptr += clen;    /* 前进到下一个主题字符 */
  }               /* 循环移动沿着主题字符串 */
# 如果从上面的 "break" 跳转到这里，表示有匹配并且设置了 PCRE2_ENDANCHORED，匹配失败
if (match_count >= 0 &&
    ((mb->moptions | mb->poptions) & PCRE2_ENDANCHORED) != 0 &&
    ptr < end_subject)
  match_count = PCRE2_ERROR_NOMATCH;

# 返回匹配次数
return match_count;
}

/*************************************************
*     Match a pattern using the DFA algorithm    *
*************************************************/

# 使用 DFA 算法匹配模式和主题字符串
# 参数：
#   code：指向编译后的模式
#   subject：主题字符串
#   length：主题字符串长度
#   startoffset：匹配开始位置
#   options：选项位
#   match_data：指向匹配数据结构
#   gcontext：指向匹配上下文
#   workspace：工作空间指针
#   wscount：工作空间大小
# 返回值：
#   > 0 => 放置在偏移量中的匹配偏移对的数量
#   = 0 => 偏移量溢出；最长匹配存在
#   -1 => 匹配失败
#   < -1 => 某种意外问题
PCRE2_EXP_DEFN int PCRE2_CALL_CONVENTION
pcre2_dfa_match(const pcre2_code *code, PCRE2_SPTR subject, PCRE2_SIZE length,
  PCRE2_SIZE start_offset, uint32_t options, pcre2_match_data *match_data,
  pcre2_match_context *mcontext, int *workspace, PCRE2_SIZE wscount)
{
int rc;
int was_zero_terminated = 0;

const pcre2_real_code *re = (const pcre2_real_code *)code;

PCRE2_SPTR start_match;
PCRE2_SPTR end_subject;
PCRE2_SPTR bumpalong_limit;
PCRE2_SPTR req_cu_ptr;

BOOL utf, anchored, startline, firstline;
BOOL has_first_cu = FALSE;
BOOL has_req_cu = FALSE;

#if PCRE2_CODE_UNIT_WIDTH == 8
PCRE2_SPTR memchr_found_first_cu = NULL;
PCRE2_SPTR memchr_found_first_cu2 = NULL;
#endif

PCRE2_UCHAR first_cu = 0;
PCRE2_UCHAR first_cu2 = 0;
PCRE2_UCHAR req_cu = 0;
PCRE2_UCHAR req_cu2 = 0;
/* 定义一个指向无符号8位整数的指针，初始值为NULL */
const uint8_t *start_bits = NULL;

/* 由于下面使用了 IS_NEWLINE 宏，它期望 NLBLOCK 被定义为一个指针，所以需要让 mb 指向一个匹配块 */
pcre2_callout_block cb;
dfa_match_block actual_match_block;
dfa_match_block *mb = &actual_match_block;

/* 为递归调用 internal_dfa_match() 设置一个起始内存块。通过将其放在堆栈上，可以在不需要时最小化资源使用。如果这个内存块太小，将从堆上获取更多内存。每个块的开头都有一个锚点结构。 */
int base_recursion_workspace[RWS_BASE_SIZE];
RWS_anchor *rws = (RWS_anchor *)base_recursion_workspace;
rws->next = NULL;
rws->size = RWS_BASE_SIZE;
rws->free = RWS_BASE_SIZE - RWS_ANCHOR_SIZE;

/* 将 NULL、长度为0的字符串识别为空字符串 */
if (subject == NULL && length == 0) subject = (PCRE2_SPTR)"";

/* 可能性检查 */
if ((options & ~PUBLIC_DFA_MATCH_OPTIONS) != 0) return PCRE2_ERROR_BADOPTION;
if (re == NULL || subject == NULL || workspace == NULL || match_data == NULL)
  return PCRE2_ERROR_NULL;

if (length == PCRE2_ZERO_TERMINATED)
  {
  length = PRIV(strlen)(subject);
  was_zero_terminated = 1;
  }

if (wscount < 20) return PCRE2_ERROR_DFA_WSSIZE;
if (start_offset > length) return PCRE2_ERROR_BADOFFSET;

/* 部分匹配和 PCRE2_ENDANCHORED 目前不允许同时出现 */
if ((options & (PCRE2_PARTIAL_HARD|PCRE2_PARTIAL_SOFT)) != 0 &&
   ((re->overall_options | options) & PCRE2_ENDANCHORED) != 0)
  return PCRE2_ERROR_BADOPTION;

/* DFA 匹配不支持无效 UTF 支持 */
if ((re->overall_options & PCRE2_MATCH_INVALID_UTF) != 0)
  return PCRE2_ERROR_DFA_UINVALID_UTF;

/* 检查块中的第一个字段是否是魔术数字。如果不是，返回 PCRE2_ERROR_BADMAGIC */
if (re->magic_number != MAGIC_NUMBER) return PCRE2_ERROR_BADMAGIC;

/* 检查代码单元宽度 */
# 检查正则表达式的模式是否与 PCRE2_CODE_UNIT_WIDTH/8 不匹配，如果不匹配则返回错误
if ((re->flags & PCRE2_MODE_MASK) != PCRE2_CODE_UNIT_WIDTH/8)
  return PCRE2_ERROR_BADMODE;

# 定义宏 FF 和 OO，用于设置 match-time 标志位
#define FF (PCRE2_NOTEMPTY_SET|PCRE2_NE_ATST_SET)
#define OO (PCRE2_NOTEMPTY|PCRE2_NOTEMPTY_ATSTART)
# 将正则表达式的标志位中的特定位传递给此函数的选项
options |= (re->flags & FF) / ((FF & (~FF+1)) / (OO & (~OO+1)));
# 取消宏定义 FF 和 OO
#undef FF
#undef OO

# 如果在部分匹配后重新启动，则对工作空间的内容进行一些合理性检查
if ((options & PCRE2_DFA_RESTART) != 0)
  {
  if ((workspace[0] & (-2)) != 0 || workspace[1] < 1 ||
    workspace[1] > (int)((wscount - 2)/INTS_PER_STATEBLOCK))
      return PCRE2_ERROR_DFA_BADRESTART;
  }

# 设置一些本地值
utf = (re->overall_options & PCRE2_UTF) != 0;
start_match = subject + start_offset;
end_subject = subject + length;
req_cu_ptr = start_match - 1;
anchored = (options & (PCRE2_ANCHORED|PCRE2_DFA_RESTART)) != 0 ||
  (re->overall_options & PCRE2_ANCHORED) != 0;

# 在查找开始位置时，"必须在行首"标志位在循环中使用
startline = (re->flags & PCRE2_STARTLINE) != 0;
firstline = (re->overall_options & PCRE2_FIRSTLINE) != 0;
bumpalong_limit = end_subject;

# 初始化并设置调用块中的固定字段，并在匹配块中设置指针
mb->cb = &cb;
# 设置回调结构体的版本号为2
cb.version = 2;
# 设置回调结构体的主题为给定的主题
cb.subject = subject;
# 设置回调结构体的主题长度为给定主题的长度
cb.subject_length = (PCRE2_SIZE)(end_subject - subject);
# 设置回调结构体的调用标志为0
cb.callout_flags = 0;
# 设置回调结构体的捕获顶部为1，表示不支持捕获
cb.capture_top      = 1;
# 设置回调结构体的最后捕获为0
cb.capture_last     = 0;
# 设置回调结构体的标记为NULL，表示不支持(*MARK)功能
cb.mark             = NULL;

# 如果匹配上下文为空
if (mcontext == NULL)
  {
  # 设置匹配块的调用为NULL
  mb->callout = NULL;
  # 设置匹配块的内存控制为正则表达式的内存控制
  mb->memctl = re->memctl;
  # 设置匹配块的匹配限制为默认匹配上下文的匹配限制
  mb->match_limit = PRIV(default_match_context).match_limit;
  # 设置匹配块的匹配深度限制为默认匹配上下文的深度限制
  mb->match_limit_depth = PRIV(default_match_context).depth_limit;
  # 设置匹配块的堆限制为默认匹配上下文的堆限制
  mb->heap_limit = PRIV(default_match_context).heap_limit;
  }
# 如果匹配上下文不为空
else
  {
  # 如果匹配上下文的偏移限制不是未设置状态
  if (mcontext->offset_limit != PCRE2_UNSET)
    {
    # 如果正则表达式的整体选项中没有设置使用偏移限制的标志，则返回偏移限制错误
    if ((re->overall_options & PCRE2_USE_OFFSET_LIMIT) == 0)
      return PCRE2_ERROR_BADOFFSETLIMIT;
    # 设置偏移限制的边界为主题加上匹配上下文的偏移限制
    bumpalong_limit = subject + mcontext->offset_limit;
    }
  # 设置匹配块的调用为匹配上下文的调用
  mb->callout = mcontext->callout;
  # 设置匹配块的调用数据为匹配上下文的调用数据
  mb->callout_data = mcontext->callout_data;
  # 设置匹配块的内存控制为匹配上下文的内存控制
  mb->memctl = mcontext->memctl;
  # 设置匹配块的匹配限制为匹配上下文的匹配限制
  mb->match_limit = mcontext->match_limit;
  # 设置匹配块的匹配深度限制为匹配上下文的深度限制
  mb->match_limit_depth = mcontext->depth_limit;
  # 设置匹配块的堆限制为匹配上下文的堆限制
  mb->heap_limit = mcontext->heap_limit;
  }

# 如果匹配块的匹配限制大于正则表达式的匹配限制，则将匹配块的匹配限制设置为正则表达式的匹配限制
if (mb->match_limit > re->limit_match)
  mb->match_limit = re->limit_match;

# 如果匹配块的匹配深度限制大于正则表达式的深度限制，则将匹配块的匹配深度限制设置为正则表达式的深度限制
if (mb->match_limit_depth > re->limit_depth)
  mb->match_limit_depth = re->limit_depth;

# 如果匹配块的堆限制大于正则表达式的堆限制，则将匹配块的堆限制设置为正则表达式的堆限制
if (mb->heap_limit > re->limit_heap)
  mb->heap_limit = re->limit_heap;

# 设置匹配块的起始代码为正则表达式的代码块起始位置
mb->start_code = (PCRE2_UCHAR *)((uint8_t *)re + sizeof(pcre2_real_code)) +
  re->name_count * re->name_entry_size;
# 设置匹配块的表为正则表达式的表
mb->tables = re->tables;
# 设置匹配块的主题起始位置为给定的主题起始位置
mb->start_subject = subject;
# 设置匹配块的主题结束位置为给定的主题结束位置
mb->end_subject = end_subject;
# 设置匹配块的起始偏移为给定的起始偏移
mb->start_offset = start_offset;
# 设置匹配块的允许空部分为正则表达式的最大回顾向后大于0，或者正则表达式的标志中包含PCRE2_MATCH_EMPTY
mb->allowemptypartial = (re->max_lookbehind > 0) ||
  (re->flags & PCRE2_MATCH_EMPTY) != 0;
# 设置匹配块的匹配选项为给定的选项
mb->moptions = options;
# 设置匹配块的整体选项为正则表达式的整体选项
mb->poptions = re->overall_options;
# 设置匹配块的匹配调用计数为0
mb->match_call_count = 0;
# 设置匹配块的堆使用为0
mb->heap_used = 0;

# 设置匹配块的BSR约定为正则表达式的BSR约定
mb->bsr_convention = re->bsr_convention;
# 设置匹配块的新行类型为固定类型
mb->nltype = NLTYPE_FIXED;
# 根据正则表达式的换行约定进行切换
switch(re->newline_convention)
  {
  # 如果是 CR 换行符
  case PCRE2_NEWLINE_CR:
  # 设置换行符长度为 1，换行符为 \r
  mb->nllen = 1;
  mb->nl[0] = CHAR_CR;
  break;

  # 如果是 LF 换行符
  case PCRE2_NEWLINE_LF:
  # 设置换行符长度为 1，换行符为 \n
  mb->nllen = 1;
  mb->nl[0] = CHAR_NL;
  break;

  # 如果是 NUL 换行符
  case PCRE2_NEWLINE_NUL:
  # 设置换行符长度为 1，换行符为 \0
  mb->nllen = 1;
  mb->nl[0] = CHAR_NUL;
  break;

  # 如果是 CRLF 换行符
  case PCRE2_NEWLINE_CRLF:
  # 设置换行符长度为 2，换行符为 \r\n
  mb->nllen = 2;
  mb->nl[0] = CHAR_CR;
  mb->nl[1] = CHAR_NL;
  break;

  # 如果是任意换行符
  case PCRE2_NEWLINE_ANY:
  # 设置换行符类型为任意
  mb->nltype = NLTYPE_ANY;
  break;

  # 如果是任意 CRLF 换行符
  case PCRE2_NEWLINE_ANYCRLF:
  # 设置换行符类型为任意 CRLF
  mb->nltype = NLTYPE_ANYCRLF;
  break;

  # 如果都不是以上情况，返回内部错误
  default: return PCRE2_ERROR_INTERNAL;
  }

# 如果需要，检查 UTF 字符串的有效性
#ifdef SUPPORT_UNICODE
if (utf && (options & PCRE2_NO_UTF_CHECK) == 0)
  {
  # 设置检查的起始位置为匹配的起始位置
  PCRE2_SPTR check_subject = start_match;  /* start_match includes offset */

  # 如果起始偏移大于 0
  if (start_offset > 0)
    {
    # 如果是 8 位或 16 位字符串，需要检查起始偏移是否指向字符的中间
    # 只检查匹配过程中将要检查的部分，从偏移减去最大反向引用到给定长度
    # 这样可以节省时间，当匹配一个大主题的一小部分时
    # 注意，最大回溯是字符数，而不是代码单元
#if PCRE2_CODE_UNIT_WIDTH != 32
    unsigned int i;
    if (start_match < end_subject && NOT_FIRSTCU(*start_match))
      return PCRE2_ERROR_BADUTFOFFSET;
    for (i = re->max_lookbehind; i > 0 && check_subject > subject; i--)
      {
      check_subject--;
      while (check_subject > subject &&
#if PCRE2_CODE_UNIT_WIDTH == 8
      (*check_subject & 0xc0) == 0x80)
#else  /* 16-bit */
      (*check_subject & 0xfc00) == 0xdc00)
#endif /* PCRE2_CODE_UNIT_WIDTH == 8 */
        check_subject--;
      }
#else   /* 在 32 位库中，一个代码单元等于一个字符 */
    check_subject -= re->max_lookbehind;
    if (check_subject < subject) check_subject = subject;
#endif  /* PCRE2_CODE_UNIT_WIDTH != 32 */
    }
    # 验证主题的相关部分。在出现错误后，将偏移量调整为整个字符串中的绝对偏移量。

    # 调用 valid_utf 函数验证主题的相关部分，将验证结果存储在 match_data->rc 中
    match_data->rc = PRIV(valid_utf)(check_subject, length - (PCRE2_SIZE)(check_subject - subject), &(match_data->startchar));
    # 如果验证结果不为0，表示出现错误
    if (match_data->rc != 0)
    {
    # 调整起始字符的位置为整个字符串中的绝对偏移量
    match_data->startchar += (PCRE2_SIZE)(check_subject - subject);
    # 返回验证结果
    return match_data->rc;
    }
  }
/* 如果支持 Unicode，则设置第一个代码单元匹配，如果没有第一个代码单元，则可能有可能有可能有可能有可能有可能有可能有可能有可能有可能有可能有可能有可能有可能有可能有可能有可能有可能有可能有可能有可能有可能有可能有可能有可能有可能有可能有可能有可能有可能有可能有可能有可能有可能有可能有可能有可能有可能有可能有可能有可能有可能有可能有可能有可能有可能有可能有可能有可能有可能有可能有可能有可能有可能有可能有可能有可能有可能有可能有可能有可能有可能有可能有可能有可能有可能有可能有可能有可能有可能有可能有可能有可能有可能有可能有可能有可能有可能有可能有可能有可能有可能有可能有可能有可能有可能有可能有可能有可能有可能有可能有可能有可能有可能有可能有可能有可能有可能有可能有可能有可能
# 将 match_data 结构体中的 subject 指针设置为 NULL，表示没有匹配结果
match_data->subject = NULL;  /* Default for no match */
# 将 match_data 结构体中的 mark 指针设置为 NULL
match_data->mark = NULL;
# 将 match_data 结构体中的 matchedby 设置为 PCRE2_MATCHEDBY_DFA_INTERPRETER
match_data->matchedby = PCRE2_MATCHEDBY_DFA_INTERPRETER;

# 调用主匹配函数，对于失败的匹配，在非锚定的正则表达式后进行循环。如果不是重新开始匹配，则在匹配开始时执行某些优化。

for (;;)
  {
  # ----------------- 开始匹配优化 ----------------

  # 有一些优化可以避免运行匹配，如果找不到已知的起始点，或者如果不存在已知的后续代码单元。但是，有一个选项（在编译时可设置）可以禁用这些优化，用于测试和确保所有调用确实发生。在重新开始 DFA 匹配时，也必须避免这些优化。

  if ((re->overall_options & PCRE2_NO_START_OPTIMIZE) == 0 &&
      (options & PCRE2_DFA_RESTART) == 0)
    {
    # 如果 firstline 为 TRUE，则匹配的开始受到多行字符串的第一行的约束。也就是说，匹配必须在匹配开始后的第一个换行符之前或之处。临时调整 end_subject，以便我们在第一个换行符的第一个字符之后立即停止优化扫描（第一个代码单元可以合法地是一个换行符）。如果匹配失败在换行符处，后续代码会中断这个循环。
    if (firstline)
      {
      PCRE2_SPTR t = start_match;
      # 如果是 UTF-8 编码，循环直到遇到换行符
      if (utf)
        {
        while (t < end_subject && !IS_NEWLINE(t))
          {
          t++;
          ACROSSCHAR(t < end_subject, t, t++);
          }
        }
      else
      while (t < end_subject && !IS_NEWLINE(t)) t++;
      end_subject = t;
      }

    # 锚定：检查是否记录了第一个代码单元。这可能看起来毫无意义，但它可以帮助检测无匹配的情况，而无需扫描所需的代码单元。
    # 如果 anchored 为真，则执行以下代码块
    if (anchored)
      {
      # 如果 has_first_cu 为真，或者 start_bits 不为空，则执行以下代码块
      if (has_first_cu || start_bits != NULL)
        {
        # 判断 start_match 是否小于 end_subject
        BOOL ok = start_match < end_subject;
        # 如果 ok 为真，则执行以下代码块
        if (ok)
          {
          # 获取 start_match 对应的字符
          PCRE2_UCHAR c = UCHAR21TEST(start_match);
          # 判断是否有 first_cu，并且 c 等于 first_cu 或者 c 等于 first_cu2
          ok = has_first_cu && (c == first_cu || c == first_cu2);
          # 如果 ok 为假，并且 start_bits 不为空，则执行以下代码块
          if (!ok && start_bits != NULL)
            {
#if PCRE2_CODE_UNIT_WIDTH != 8
            // 如果 PCRE2_CODE_UNIT_WIDTH 不等于 8，则将字符 c 大于 255 的值设为 255
            if (c > 255) c = 255;
#endif
            // 检查字符 c 是否在 start_bits 数组中对应的位上为 1
            ok = (start_bits[c/8] & (1u << (c&7))) != 0;
            }
          }
        if (!ok) break;
        }
      }

    /* Not anchored. Advance to a unique first code unit if there is one. */

    else
      {
      // 如果有唯一的第一个代码单元
      if (has_first_cu)
        {
        // 如果第一个代码单元不等于第二个代码单元（不区分大小写）
        if (first_cu != first_cu2)  
          {
          // 在 16 位和 32 位模式下，需要自己进行搜索，因此可以同时查找两种情况
#if PCRE2_CODE_UNIT_WIDTH != 8
          // 定义变量 smc，循环查找第一个代码单元或第二个代码单元
          PCRE2_UCHAR smc;
          while (start_match < end_subject &&
                (smc = UCHAR21TEST(start_match)) != first_cu &&
                 smc != first_cu2)
            start_match++;
#endif  /* 8-bit handling */
          }

        // 如果是区分大小写的情况
        else
          {
#if PCRE2_CODE_UNIT_WIDTH != 8
          // 循环查找第一个代码单元
          while (start_match < end_subject && UCHAR21TEST(start_match) !=
                 first_cu)
            start_match++;
#else  /* 8-bit code units */
          // 在 8 位代码单元中查找第一个代码单元
          start_match = memchr(start_match, first_cu, end_subject - start_match);
          if (start_match == NULL) start_match = end_subject;
#endif
          }

        /* 如果在主题的真实结尾处找不到所需的代码单元，就打破循环，强制匹配失败，
        除非进行部分匹配，这时我们让下一个周期在主题的结尾运行。要了解原因，请考虑模式 /(?<=abc)def/,
        它部分匹配 "abc"，即使字符串不包含起始字符 "d"。如果我们没有到达主题的真实结尾（PCRE2_FIRSTLINE 导致 end_subject 被临时修改），
        我们也让周期运行，因为匹配字符串合法地允许以换行符的第一个代码单元开始。 */

        if ((mb->moptions & (PCRE2_PARTIAL_HARD|PCRE2_PARTIAL_SOFT)) == 0 &&
            start_match >= mb->end_subject)
          break;
        }

      /* 如果没有第一个代码单元，根据需要对多行匹配进行换行符后的移动。 */

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
          // 当匹配起始位置小于结束位置并且不是换行符时，继续向前移动匹配起始位置
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

      /* 如果没有第一个代码单元或需要多行起始的要求，如果已经识别到非唯一的第一个代码单元，就向前移动到非唯一的第一个代码单元。位图只包含 256 位。当代码单元宽度为 16 或 32 位时，所有大于 254 的代码单元设置第 255 位。 */
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

        /* 参见上面对第一个代码单元检查的注释。 */
        if ((mb->moptions & (PCRE2_PARTIAL_HARD|PCRE2_PARTIAL_SOFT)) == 0 &&
            start_match >= mb->end_subject)
          break;
        }
      }  /* 第一个代码单元处理结束 */

    /* 恢复修改后的结束位置 */
    end_subject = mb->end_subject;

    /* 下面两个优化对于部分匹配是禁用的。 */
#if PCRE2_CODE_UNIT_WIDTH != 8
            while (p < end_subject)
              {
              uint32_t pp = UCHAR21INCTEST(p);
              if (pp == req_cu || pp == req_cu2) { p--; break; }
              }
#else  /* 8-bit code units */
            // 如果不是8位代码单元，定义指针pp指向p的位置
            PCRE2_SPTR pp = p;
            // 在pp指向的位置到end_subject之间查找req_cu，返回指针p
            p = memchr(pp, req_cu, end_subject - pp);
            // 如果p为NULL
            if (p == NULL)
              {
              // 在pp指向的位置到end_subject之间查找req_cu2，返回指针p
              p = memchr(pp, req_cu2, end_subject - pp);
              // 如果p为NULL，将p指向end_subject
              if (p == NULL) p = end_subject;
              }
#endif /* PCRE2_CODE_UNIT_WIDTH != 8 */
            }

          /* The caseful case */

          else
            {
#if PCRE2_CODE_UNIT_WIDTH != 8
            // 如果不是8位代码单元，循环直到p大于等于end_subject
            while (p < end_subject)
              {
              // 如果p指向的字符等于req_cu，将p向前移动一位并跳出循环
              if (UCHAR21INCTEST(p) == req_cu) { p--; break; }
              }

#else  /* 8-bit code units */
            // 如果是8位代码单元，在p指向的位置到end_subject之间查找req_cu，返回指针p
            p = memchr(p, req_cu, end_subject - p);
            // 如果p为NULL，将p指向end_subject
            if (p == NULL) p = end_subject;
#endif
            }

          /* If we can't find the required code unit, break the matching loop,
          forcing a match failure. */

          // 如果找不到所需的代码单元，跳出匹配循环，强制匹配失败
          if (p >= end_subject) break;

          /* If we have found the required code unit, save the point where we
          found it, so that we don't search again next time round the loop if
          the start hasn't passed this code unit yet. */

          // 如果找到了所需的代码单元，保存找到的位置，以便下次循环时不再搜索
          req_cu_ptr = p;
          }
        }
      }
    }

  /* ------------ End of start of match optimizations ------------ */

  /* Give no match if we have passed the bumpalong limit. */

  // 如果已经超过了bumpalong_limit，不进行匹配
  if (start_match > bumpalong_limit) break;

  /* OK, now we can do the business */

  // 设置mb的start_used_ptr和last_used_ptr为start_match
  mb->start_used_ptr = start_match;
  mb->last_used_ptr = start_match;
  // 将mb的recursive指针设置为NULL
  mb->recursive = NULL;

  // 调用internal_dfa_match函数进行匹配
  rc = internal_dfa_match(
    mb,                           /* fixed match data */
    mb->start_code,               /* this subexpression's code */
    start_match,                  /* where we currently are */
    start_offset,                 /* start offset in subject */
    match_data->ovector,          /* offset vector */
    (uint32_t)match_data->oveccount * 2,  /* actual size of same */
    workspace,                    /* workspace vector */
    (int)wscount,                 /* size of same */
    0,                            /* function recurse level */
    base_recursion_workspace);    /* initial workspace for recursion */

  /* 如果返回值不是 "no match"，则表示匹配完成；否则，只有在没有锚定的情况下才继续。 */

  if (rc != PCRE2_ERROR_NOMATCH || anchored)
    {
    if (rc == PCRE2_ERROR_PARTIAL && match_data->oveccount > 0)
      {
      match_data->ovector[0] = (PCRE2_SIZE)(start_match - subject);
      match_data->ovector[1] = (PCRE2_SIZE)(end_subject - subject);
      }
    match_data->leftchar = (PCRE2_SIZE)(mb->start_used_ptr - subject);
    match_data->rightchar = (PCRE2_SIZE)( mb->last_used_ptr - subject);
    match_data->startchar = (PCRE2_SIZE)(start_match - subject);
    match_data->rc = rc;

    if (rc >= 0 &&(options & PCRE2_COPY_MATCHED_SUBJECT) != 0)
      {
      length = CU2BYTES(length + was_zero_terminated);
      match_data->subject = match_data->memctl.malloc(length,
        match_data->memctl.memory_data);
      if (match_data->subject == NULL) return PCRE2_ERROR_NOMEMORY;
      memcpy((void *)match_data->subject, subject, length);
      match_data->flags |= PCRE2_MD_COPIED_SUBJECT;
      }
    else
      {
      if (rc >= 0 || rc == PCRE2_ERROR_PARTIAL) match_data->subject = subject;
      }
    goto EXIT;
    }

  /* 如果是在行末尾且设置了 firstline，则不前进到下一个主题字符。 */

  if (firstline && IS_NEWLINE(start_match)) break;
  start_match++;
#ifdef SUPPORT_UNICODE
  // 如果支持 Unicode，则执行以下代码块
  if (utf)
    {
    // 在遍历字符时，跳过多字节字符的每个字节
    ACROSSCHAR(start_match < end_subject, start_match, start_match++);
    }
#endif
  // 如果匹配起始位置超过了结束位置，则跳出循环
  if (start_match > end_subject) break;

  /* 如果刚刚经过一个 CR，现在是 LF，并且模式中不包含对 \r 或 \n 的显式匹配，
  并且换行选项是 CRLF 或 ANY 或 ANYCRLF，则将匹配位置再向前移动一个字符。 */
  if (UCHAR21TEST(start_match - 1) == CHAR_CR &&
      start_match < end_subject &&
      UCHAR21TEST(start_match) == CHAR_NL &&
      (re->flags & PCRE2_HASCRORLF) == 0 &&
        (mb->nltype == NLTYPE_ANY ||
         mb->nltype == NLTYPE_ANYCRLF ||
         mb->nllen == 2))
    start_match++;

  }   /* "Bumpalong" loop */

NOMATCH_EXIT:
// 设置未匹配错误码
rc = PCRE2_ERROR_NOMATCH;

EXIT:
// 释放内存
while (rws->next != NULL)
  {
  RWS_anchor *next = rws->next;
  rws->next = next->next;
  mb->memctl.free(next, mb->memctl.memory_data);
  }

// 返回错误码
return rc;
}

/* 这些 #undef 是为了使 CMake 中的统一构建生效。 */

#undef NLBLOCK /* 包含换行符信息的块 */
#undef PSSTART /* 包含处理后字符串起始位置的字段 */
#undef PSEND   /* 包含处理后字符串结束位置的字段 */

/* pcre2_dfa_match.c 的结束 */
```