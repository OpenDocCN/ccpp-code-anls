# `xmrig\src\3rdparty\llhttp\llhttp.c`

```cpp
#if LLHTTP_STRICT_MODE

#include <stdlib.h>  // 包含标准库头文件
#include <stdint.h>  // 包含整数类型定义头文件
#include <string.h>  // 包含字符串处理函数头文件

#ifdef __SSE4_2__  // 如果支持 SSE4.2 指令集
 #ifdef _MSC_VER  // 如果是 Microsoft 编译器
  #include <nmmintrin.h>  // 包含 SSE4.2 指令集头文件
 #else  /* !_MSC_VER */
  #include <x86intrin.h>  // 包含 SSE4.2 指令集头文件
 #endif  /* _MSC_VER */
#endif  /* __SSE4_2__ */

#ifdef _MSC_VER  // 如果是 Microsoft 编译器
 #define ALIGN(n) _declspec(align(n))  // 定义对齐宏
#else  /* !_MSC_VER */
 #define ALIGN(n) __attribute__((aligned(n)))  // 定义对齐宏
#endif  /* _MSC_VER */

#include "llhttp.h"  // 包含自定义头文件 llhttp.h

typedef int (*llhttp__internal__span_cb)(  // 定义函数指针类型
             llhttp__internal_t*, const char*, const char*);

static const unsigned char llparse_blob0[] = {  // 定义静态常量数组
  0xd, 0xa  // 初始化数组
};
static const unsigned char llparse_blob1[] = {  // 定义静态常量数组
  'o', 'n'  // 初始化数组
};
// ... 其他静态常量数组的定义和初始化

#ifdef __SSE4_2__  // 如果支持 SSE4.2 指令集
static const unsigned char ALIGN(16) llparse_blob7[] = {  // 定义对齐的静态常量数组
  0x9, 0x9, ' ', '~', 0x80, 0xff, 0x0, 0x0, 0x0, 0x0, 0x0,  // 初始化数组
  0x0, 0x0, 0x0, 0x0, 0x0  // 初始化数组
};
// ... 其他对齐的静态常量数组的定义和初始化
#endif  /* __SSE4_2__ */

// ... 其他静态常量数组的定义和初始化
# 定义一个包含字符数组的常量 llparse_blob13
static const unsigned char llparse_blob13[] = {
  'p', 'g', 'r', 'a', 'd', 'e'
};
# 定义一个包含字符数组的常量 llparse_blob14
static const unsigned char llparse_blob14[] = {
  'T', 'T', 'P', '/'
};
# 定义一个包含字符数组的常量 llparse_blob15
static const unsigned char llparse_blob15[] = {
  0xd, 0xa, 0xd, 0xa, 'S', 'M', 0xd, 0xa, 0xd, 0xa
};
# 定义一个包含字符数组的常量 llparse_blob16
static const unsigned char llparse_blob16[] = {
  'C', 'E', '/'
};
# 定义一个包含字符数组的常量 llparse_blob17
static const unsigned char llparse_blob17[] = {
  'T', 'S', 'P', '/'
};
# 定义一个包含字符数组的常量 llparse_blob18
static const unsigned char llparse_blob18[] = {
  'N', 'O', 'U', 'N', 'C', 'E'
};
# 定义一个包含字符数组的常量 llparse_blob19
static const unsigned char llparse_blob19[] = {
  'I', 'N', 'D'
};
# 定义一个包含字符数组的常量 llparse_blob20
static const unsigned char llparse_blob20[] = {
  'E', 'C', 'K', 'O', 'U', 'T'
};
# 定义一个包含字符数组的常量 llparse_blob21
static const unsigned char llparse_blob21[] = {
  'N', 'E', 'C', 'T'
};
# 定义一个包含字符数组的常量 llparse_blob22
static const unsigned char llparse_blob22[] = {
  'E', 'T', 'E'
};
# 定义一个包含字符数组的常量 llparse_blob23
static const unsigned char llparse_blob23[] = {
  'C', 'R', 'I', 'B', 'E'
};
# 定义一个包含字符数组的常量 llparse_blob24
static const unsigned char llparse_blob24[] = {
  'L', 'U', 'S', 'H'
};
# 定义一个包含字符数组的常量 llparse_blob25
static const unsigned char llparse_blob25[] = {
  'E', 'T'
};
# 定义一个包含字符数组的常量 llparse_blob26
static const unsigned char llparse_blob26[] = {
  'P', 'A', 'R', 'A', 'M', 'E', 'T', 'E', 'R'
};
# 定义一个包含字符数组的常量 llparse_blob27
static const unsigned char llparse_blob27[] = {
  'E', 'A', 'D'
};
# 定义一个包含字符数组的常量 llparse_blob28
static const unsigned char llparse_blob28[] = {
  'N', 'K'
};
# 定义一个包含字符数组的常量 llparse_blob29
static const unsigned char llparse_blob29[] = {
  'C', 'K'
};
# 定义一个包含字符数组的常量 llparse_blob30
static const unsigned char llparse_blob30[] = {
  'S', 'E', 'A', 'R', 'C', 'H'
};
# 定义一个包含字符数组的常量 llparse_blob31
static const unsigned char llparse_blob31[] = {
  'R', 'G', 'E'
};
# 定义一个包含字符数组的常量 llparse_blob32
static const unsigned char llparse_blob32[] = {
  'C', 'T', 'I', 'V', 'I', 'T', 'Y'
};
# 定义一个包含字符数组的常量 llparse_blob33
static const unsigned char llparse_blob33[] = {
  'L', 'E', 'N', 'D', 'A', 'R'
};
# 定义一个包含字符数组的常量 llparse_blob34
static const unsigned char llparse_blob34[] = {
  'V', 'E'
};
# 定义一个包含字符数组的常量 llparse_blob35
static const unsigned char llparse_blob35[] = {
  'O', 'T', 'I', 'F', 'Y'
};
# 定义一个包含字符数组的常量 llparse_blob36
static const unsigned char llparse_blob36[] = {
  'P', 'T', 'I', 'O', 'N', 'S'
};
# 定义一个包含字符数组的常量 llparse_blob37
static const unsigned char llparse_blob37[] = {
  'C', 'H'
};
# 定义一个包含字符数组的常量 llparse_blob38
static const unsigned char llparse_blob38[] = {
  'S', 'E'
};
# 定义一个包含字符数组的常量 llparse_blob39
static const unsigned char llparse_blob39[] = {
  'A', 'Y'
};
// 定义包含两个字符 'S' 和 'T' 的无符号字符数组
static const unsigned char llparse_blob40[] = {
  'S', 'T'
};
// 定义包含三个字符 'I'、'N' 和 'D' 的无符号字符数组
static const unsigned char llparse_blob41[] = {
  'I', 'N', 'D'
};
// 定义包含四个字符 'A'、'T'、'C' 和 'H' 的无符号字符数组
static const unsigned char llparse_blob42[] = {
  'A', 'T', 'C', 'H'
};
// 定义包含两个字符 'G' 和 'E' 的无符号字符数组
static const unsigned char llparse_blob43[] = {
  'G', 'E'
};
// 定义包含三个字符 'I'、'N' 和 'D' 的无符号字符数组
static const unsigned char llparse_blob44[] = {
  'I', 'N', 'D'
};
// 定义包含三个字符 'O'、'R' 和 'D' 的无符号字符数组
static const unsigned char llparse_blob45[] = {
  'O', 'R', 'D'
};
// 定义包含五个字符 'I'、'R'、'E'、'C' 和 'T' 的无符号字符数组
static const unsigned char llparse_blob46[] = {
  'I', 'R', 'E', 'C', 'T'
};
// 定义包含三个字符 'O'、'R' 和 'T' 的无符号字符数组
static const unsigned char llparse_blob47[] = {
  'O', 'R', 'T'
};
// 定义包含三个字符 'R'、'C' 和 'H' 的无符号字符数组
static const unsigned char llparse_blob48[] = {
  'R', 'C', 'H'
};
// 定义包含九个字符 'P'、'A'、'R'、'A'、'M'、'E'、'T'、'E' 和 'R' 的无符号字符数组
static const unsigned char llparse_blob49[] = {
  'P', 'A', 'R', 'A', 'M', 'E', 'T', 'E', 'R'
};
// 定义包含四个字符 'U'、'R'、'C' 和 'E' 的无符号字符数组
static const unsigned char llparse_blob50[] = {
  'U', 'R', 'C', 'E'
};
// 定义包含七个字符 'B'、'S'、'C'、'R'、'I'、'B' 和 'E' 的无符号字符数组
static const unsigned char llparse_blob51[] = {
  'B', 'S', 'C', 'R', 'I', 'B', 'E'
};
// 定义包含六个字符 'A'、'R'、'D'、'O'、'W' 和 'N' 的无符号字符数组
static const unsigned char llparse_blob52[] = {
  'A', 'R', 'D', 'O', 'W', 'N'
};
// 定义包含三个字符 'A'、'C' 和 'E' 的无符号字符数组
static const unsigned char llparse_blob53[] = {
  'A', 'C', 'E'
};
// 定义包含三个字符 'I'、'N' 和 'D' 的无符号字符数组
static const unsigned char llparse_blob54[] = {
  'I', 'N', 'D'
};
// 定义包含两个字符 'N' 和 'K' 的无符号字符数组
static const unsigned char llparse_blob55[] = {
  'N', 'K'
};
// 定义包含两个字符 'C' 和 'K' 的无符号字符数组
static const unsigned char llparse_blob56[] = {
  'C', 'K'
};
// 定义包含八个字符 'U'、'B'、'S'、'C'、'R'、'I'、'B' 和 'E' 的无符号字符数组
static const unsigned char llparse_blob57[] = {
  'U', 'B', 'S', 'C', 'R', 'I', 'B', 'E'
};
// 定义包含五个字符 'H'、'T'、'T'、'P' 和 '/' 的无符号字符数组
static const unsigned char llparse_blob58[] = {
  'H', 'T', 'T', 'P', '/'
};
// 定义包含两个字符 'A' 和 'D' 的无符号字符数组
static const unsigned char llparse_blob59[] = {
  'A', 'D'
};
// 定义包含三个字符 'T'、'P' 和 '/' 的无符号字符数组
static const unsigned char llparse_blob60[] = {
  'T', 'P', '/'
};

// 定义枚举类型 llparse_match_status_e，包含三个枚举值
enum llparse_match_status_e {
  kMatchComplete,
  kMatchPause,
  kMatchMismatch
};
// 为枚举类型 llparse_match_status_e 定义别名 llparse_match_status_t
typedef enum llparse_match_status_e llparse_match_status_t;

// 定义结构体 llparse_match_s，包含枚举类型成员 status 和指向无符号字符的指针成员 current
struct llparse_match_s {
  llparse_match_status_t status;
  const unsigned char* current;
};
// 为结构体 llparse_match_s 定义别名 llparse_match_t
typedef struct llparse_match_s llparse_match_t;

// 定义静态函数 llparse__match_sequence_id，接受指向 llhttp__internal_t 结构体的指针、两个指向无符号字符的指针作为参数
static llparse_match_t llparse__match_sequence_id(
    llhttp__internal_t* s, const unsigned char* p,
    const unsigned char* endp,
    // 匹配输入序列和给定序列，返回匹配结果
    llparse_match_t llparse_match(const unsigned char* p, const unsigned char* endp, const unsigned char* seq, uint32_t seq_len) {
      uint32_t index; // 定义索引变量
      llparse_match_t res; // 定义匹配结果变量
    
      index = s->_index; // 将状态机中的索引值赋给index
      for (; p != endp; p++) { // 遍历输入序列
        unsigned char current; // 定义当前字符变量
    
        current = *p; // 获取当前字符
        if (current == seq[index]) { // 如果当前字符与给定序列中索引位置的字符相等
          if (++index == seq_len) { // 如果索引增加后等于给定序列长度
            res.status = kMatchComplete; // 设置匹配结果为完全匹配
            goto reset; // 跳转到reset标签
          }
        } else { // 如果当前字符与给定序列中索引位置的字符不相等
          res.status = kMatchMismatch; // 设置匹配结果为不匹配
          goto reset; // 跳转到reset标签
        }
      }
      s->_index = index; // 更新状态机中的索引值
      res.status = kMatchPause; // 设置匹配结果为暂停匹配
      res.current = p; // 设置当前位置
      return res; // 返回匹配结果
    
    reset: // reset标签，用于重置状态
      // 重置状态
      return res; // 返回匹配结果
    }
# 重置函数，将索引重置为0，设置当前状态为p，返回结果
reset:
  s->_index = 0;
  res.current = p;
  return res;
}

# 匹配序列转换为小写，返回匹配结果
static llparse_match_t llparse__match_sequence_to_lower(
    llhttp__internal_t* s, const unsigned char* p,
    const unsigned char* endp,
    const unsigned char* seq, uint32_t seq_len) {
  uint32_t index;
  llparse_match_t res;

  index = s->_index;
  # 遍历序列，将大写字母转换为小写，进行匹配
  for (; p != endp; p++) {
    unsigned char current;

    current = ((*p) >= 'A' && (*p) <= 'Z' ? (*p | 0x20) : (*p));
    if (current == seq[index]) {
      if (++index == seq_len) {
        res.status = kMatchComplete;
        goto reset;
      }
    } else {
      res.status = kMatchMismatch;
      goto reset;
    }
  }
  s->_index = index;
  res.status = kMatchPause;
  res.current = p;
  return res;
reset:
  s->_index = 0;
  res.current = p;
  return res;
}

# 不安全的匹配序列转换为小写，返回匹配结果
static llparse_match_t llparse__match_sequence_to_lower_unsafe(
    llhttp__internal_t* s, const unsigned char* p,
    const unsigned char* endp,
    const unsigned char* seq, uint32_t seq_len) {
  uint32_t index;
  llparse_match_t res;

  index = s->_index;
  # 遍历序列，将字符转换为小写，进行匹配
  for (; p != endp; p++) {
    unsigned char current;

    current = ((*p) | 0x20);
    if (current == seq[index]) {
      if (++index == seq_len) {
        res.status = kMatchComplete;
        goto reset;
      }
    } else {
      res.status = kMatchMismatch;
      goto reset;
    }
  }
  s->_index = index;
  res.status = kMatchPause;
  res.current = p;
  return res;
reset:
  s->_index = 0;
  res.current = p;
  return res;
}

# 枚举状态
typedef enum llparse_state_e llparse_state_t;

# 处理URL
int llhttp__on_url(
    llhttp__internal_t* s, const unsigned char* p,
    const unsigned char* endp);

# 处理header字段
int llhttp__on_header_field(
    llhttp__internal_t* s, const unsigned char* p,
    const unsigned char* endp);

# 处理header值
int llhttp__on_header_value(
    llhttp__internal_t* s, const unsigned char* p,
    const unsigned char* endp);

# 处理body
int llhttp__on_body(
    llhttp__internal_t* s, const unsigned char* p,
    const unsigned char* endp);

# 处理状态
int llhttp__on_status(
    // 定义指向 llhttp__internal_t 结构体的指针 s，参数为指向无符号字符型的指针 p 和 endp
    llhttp__internal_t* s, const unsigned char* p,
    const unsigned char* endp);
# 更新解析状态的完成标志为2
int llhttp__internal__c_update_finish(
    llhttp__internal_t* state,
    const unsigned char* p,
    const unsigned char* endp) {
  state->finish = 2;
  return 0;
}

# 返回解析状态的消息开始标志
int llhttp__on_message_begin(
    llhttp__internal_t* s, const unsigned char* p,
    const unsigned char* endp);

# 返回解析状态的类型
int llhttp__internal__c_load_type(
    llhttp__internal_t* state,
    const unsigned char* p,
    const unsigned char* endp) {
  return state->type;
}

# 存储解析状态的方法
int llhttp__internal__c_store_method(
    llhttp__internal_t* state,
    const unsigned char* p,
    const unsigned char* endp,
    int match) {
  state->method = match;
  return 0;
}

# 判断解析状态的方法是否等于5
int llhttp__internal__c_is_equal_method(
    llhttp__internal_t* state,
    const unsigned char* p,
    const unsigned char* endp) {
  return state->method == 5;
}

# 更新解析状态的 HTTP 主版本号为0
int llhttp__internal__c_update_http_major(
    llhttp__internal_t* state,
    const unsigned char* p,
    const unsigned char* endp) {
  state->http_major = 0;
  return 0;
}

# 更新解析状态的 HTTP 次版本号为9
int llhttp__internal__c_update_http_minor(
    llhttp__internal_t* state,
    const unsigned char* p,
    const unsigned char* endp) {
  state->http_minor = 9;
  return 0;
}

# 返回解析状态的标志是否包含128
int llhttp__internal__c_test_flags(
    llhttp__internal_t* state,
    const unsigned char* p,
    const unsigned char* endp) {
  return (state->flags & 128) == 128;
}

# 返回解析状态的消息块完成标志
int llhttp__on_chunk_complete(
    llhttp__internal_t* s, const unsigned char* p,
    const unsigned char* endp);

# 返回解析状态的消息完成标志
int llhttp__on_message_complete(
    llhttp__internal_t* s, const unsigned char* p,
    const unsigned char* endp);

# 判断解析状态的升级标志是否为1
int llhttp__internal__c_is_equal_upgrade(
    llhttp__internal_t* state,
    const unsigned char* p,
    const unsigned char* endp) {
  return state->upgrade == 1;
}

# 返回解析状态的消息完成后的处理
int llhttp__after_message_complete(
    llhttp__internal_t* s, const unsigned char* p,
    const unsigned char* endp);

# 更新解析状态的完成标志为1
int llhttp__internal__c_update_finish_1(
    # 设置状态对象的完成标志为0
    state->finish = 0;
    # 返回0
    return 0;
}

这是一个函数的结束标志。


int llhttp__internal__c_test_lenient_flags(
    llhttp__internal_t* state,
    const unsigned char* p,
    const unsigned char* endp) {
  return (state->lenient_flags & 4) == 4;
}

这是一个函数，用于测试状态对象的lenient_flags属性是否包含特定标志。


int llhttp__internal__c_test_flags_1(
    llhttp__internal_t* state,
    const unsigned char* p,
    const unsigned char* endp) {
  return (state->flags & 544) == 544;
}

这是一个函数，用于测试状态对象的flags属性是否包含特定标志。


int llhttp__internal__c_test_lenient_flags_1(
    llhttp__internal_t* state,
    const unsigned char* p,
    const unsigned char* endp) {
  return (state->lenient_flags & 2) == 2;
}

这是一个函数，用于测试状态对象的lenient_flags属性是否包含特定标志。


int llhttp__before_headers_complete(
    llhttp__internal_t* s, const unsigned char* p,
    const unsigned char* endp);

这是一个函数声明，用于在头部完成之前执行特定操作。


int llhttp__on_headers_complete(
    llhttp__internal_t* s, const unsigned char* p,
    const unsigned char* endp);

这是一个函数声明，用于在头部完成时执行特定操作。


int llhttp__after_headers_complete(
    llhttp__internal_t* s, const unsigned char* p,
    const unsigned char* endp);

这是一个函数声明，用于在头部完成之后执行特定操作。


int llhttp__internal__c_update_content_length(
    llhttp__internal_t* state,
    const unsigned char* p,
    const unsigned char* endp) {
  state->content_length = 0;
  return 0;
}

这是一个函数，用于更新状态对象的content_length属性。


int llhttp__internal__c_mul_add_content_length(
    llhttp__internal_t* state,
    const unsigned char* p,
    const unsigned char* endp,
    int match) {
  /* Multiplication overflow */
  if (state->content_length > 0xffffffffffffffffULL / 16) {
    return 1;
  }

  state->content_length *= 16;

  /* Addition overflow */
  if (match >= 0) {
    if (state->content_length > 0xffffffffffffffffULL - match) {
      return 1;
    }
  } else {
    if (state->content_length < 0ULL - match) {
      return 1;
    }
  }
  state->content_length += match;
  return 0;
}

这是一个函数，用于根据匹配值更新状态对象的content_length属性，并检查是否发生溢出。


int llhttp__on_chunk_header(
    llhttp__internal_t* s, const unsigned char* p,
    const unsigned char* endp);

这是一个函数声明，用于在块头部处理时执行特定操作。


int llhttp__internal__c_is_equal_content_length(
    llhttp__internal_t* state,
    const unsigned char* p,
    const unsigned char* endp) {
  return state->content_length == 0;
}

这是一个函数，用于检查状态对象的content_length属性是否等于0。


int llhttp__internal__c_or_flags(

这是一个函数声明，用于执行状态对象的flags属性的逻辑或操作。
    # 设置状态对象的标志位，将第 8 位（从右往左数）设置为 1
    state->flags |= 128;
    # 返回整数 0
    return 0;
# 更新解析状态的完成标志位为1
int llhttp__internal__c_update_finish_3(
    llhttp__internal_t* state,
    const unsigned char* p,
    const unsigned char* endp) {
  state->finish = 1;
  return 0;
}

# 更新解析状态的标志位，将第7位（从右往左数）设置为1
int llhttp__internal__c_or_flags_1(
    llhttp__internal_t* state,
    const unsigned char* p,
    const unsigned char* endp) {
  state->flags |= 64;
  return 0;
}

# 更新解析状态的升级标志位为1
int llhttp__internal__c_update_upgrade(
    llhttp__internal_t* state,
    const unsigned char* p,
    const unsigned char* endp) {
  state->upgrade = 1;
  return 0;
}

# 存储解析状态的头部状态
int llhttp__internal__c_store_header_state(
    llhttp__internal_t* state,
    const unsigned char* p,
    const unsigned char* endp,
    int match) {
  state->header_state = match;
  return 0;
}

# 加载解析状态的头部状态
int llhttp__internal__c_load_header_state(
    llhttp__internal_t* state,
    const unsigned char* p,
    const unsigned char* endp) {
  return state->header_state;
}

# 更新解析状态的标志位，将第1位（从右往左数）设置为1
int llhttp__internal__c_or_flags_3(
    llhttp__internal_t* state,
    const unsigned char* p,
    const unsigned char* endp) {
  state->flags |= 1;
  return 0;
}

# 更新解析状态的头部状态为1
int llhttp__internal__c_update_header_state(
    llhttp__internal_t* state,
    const unsigned char* p,
    const unsigned char* endp) {
  state->header_state = 1;
  return 0;
}

# 更新解析状态的标志位，将第2位（从右往左数）设置为1
int llhttp__internal__c_or_flags_4(
    llhttp__internal_t* state,
    const unsigned char* p,
    const unsigned char* endp) {
  state->flags |= 2;
  return 0;
}

# 更新解析状态的标志位，将第3位（从右往左数）设置为1
int llhttp__internal__c_or_flags_5(
    llhttp__internal_t* state,
    const unsigned char* p,
    const unsigned char* endp) {
  state->flags |= 4;
  return 0;
}

# 更新解析状态的标志位，将第4位（从右往左数）设置为1
int llhttp__internal__c_or_flags_6(
    llhttp__internal_t* state,
    const unsigned char* p,
    const unsigned char* endp) {
  state->flags |= 8;
  return 0;
}

# 更新解析状态的头部状态为2
int llhttp__internal__c_update_header_state_2(
    # 设置状态机的头部状态为6
    llhttp__internal_t* state,
    # 定义指向无符号字符的指针p，指向解析的起始位置
    const unsigned char* p,
    # 定义指向无符号字符的指针endp，指向解析的结束位置
    const unsigned char* endp) {
  # 设置状态机的头部状态为6
  state->header_state = 6;
  # 返回0，表示解析成功
  return 0;
# 检查状态对象的 lenient_flags 属性中是否包含特定标志位
int llhttp__internal__c_test_lenient_flags_2(
    llhttp__internal_t* state,
    const unsigned char* p,
    const unsigned char* endp) {
  return (state->lenient_flags & 1) == 1;
}

# 更新状态对象的 header_state 属性为 0
int llhttp__internal__c_update_header_state_4(
    llhttp__internal_t* state,
    const unsigned char* p,
    const unsigned char* endp) {
  state->header_state = 0;
  return 0;
}

# 更新状态对象的 header_state 属性为 5
int llhttp__internal__c_update_header_state_5(
    llhttp__internal_t* state,
    const unsigned char* p,
    const unsigned char* endp) {
  state->header_state = 5;
  return 0;
}

# 更新状态对象的 header_state 属性为 7
int llhttp__internal__c_update_header_state_6(
    llhttp__internal_t* state,
    const unsigned char* p,
    const unsigned char* endp) {
  state->header_state = 7;
  return 0;
}

# 检查状态对象的 flags 属性中是否包含特定标志位
int llhttp__internal__c_test_flags_2(
    llhttp__internal_t* state,
    const unsigned char* p,
    const unsigned char* endp) {
  return (state->flags & 32) == 32;
}

# 对状态对象的 content_length 属性进行乘法和加法操作
int llhttp__internal__c_mul_add_content_length_1(
    llhttp__internal_t* state,
    const unsigned char* p,
    const unsigned char* endp,
    int match) {
  /* Multiplication overflow */
  # 检查乘法是否会导致溢出
  if (state->content_length > 0xffffffffffffffffULL / 10) {
    return 1;
  }

  state->content_length *= 10;

  /* Addition overflow */
  # 检查加法是否会导致溢出
  if (match >= 0) {
    if (state->content_length > 0xffffffffffffffffULL - match) {
      return 1;
    }
  } else {
    if (state->content_length < 0ULL - match) {
      return 1;
    }
  }
  state->content_length += match;
  return 0;
}

# 对状态对象的 flags 属性进行按位或操作
int llhttp__internal__c_or_flags_15(
    llhttp__internal_t* state,
    const unsigned char* p,
    const unsigned char* endp) {
  state->flags |= 32;
  return 0;
}

# 对状态对象的 flags 属性进行按位或操作
int llhttp__internal__c_or_flags_16(
    llhttp__internal_t* state,
    const unsigned char* p,
    const unsigned char* endp) {
  state->flags |= 512;
  return 0;
}

# 对状态对象的 flags 属性进行按位与操作
int llhttp__internal__c_and_flags(
    llhttp__internal_t* state,
    const unsigned char* p,
    const unsigned char* endp) {
  state->flags &= -9;
  return 0;
}
# 更新头部状态为 8
int llhttp__internal__c_update_header_state_7(
    llhttp__internal_t* state,
    const unsigned char* p,
    const unsigned char* endp) {
  state->header_state = 8;
  return 0;
}

# 将状态的标志位与 16 进行或操作
int llhttp__internal__c_or_flags_17(
    llhttp__internal_t* state,
    const unsigned char* p,
    const unsigned char* endp) {
  state->flags |= 16;
  return 0;
}

# 加载方法
int llhttp__internal__c_load_method(
    llhttp__internal_t* state,
    const unsigned char* p,
    const unsigned char* endp) {
  return state->method;
}

# 存储 HTTP 主版本号
int llhttp__internal__c_store_http_major(
    llhttp__internal_t* state,
    const unsigned char* p,
    const unsigned char* endp,
    int match) {
  state->http_major = match;
  return 0;
}

# 存储 HTTP 次版本号
int llhttp__internal__c_store_http_minor(
    llhttp__internal_t* state,
    const unsigned char* p,
    const unsigned char* endp,
    int match) {
  state->http_minor = match;
  return 0;
}

# 更新状态码为 0
int llhttp__internal__c_update_status_code(
    llhttp__internal_t* state,
    const unsigned char* p,
    const unsigned char* endp) {
  state->status_code = 0;
  return 0;
}

# 乘法溢出状态码
int llhttp__internal__c_mul_add_status_code(
    llhttp__internal_t* state,
    const unsigned char* p,
    const unsigned char* endp,
    int match) {
  /* 乘法溢出 */
  if (state->status_code > 0xffff / 10) {
    return 1;
  }

  state->status_code *= 10;

  /* 加法溢出 */
  if (match >= 0) {
    if (state->status_code > 0xffff - match) {
      return 1;
    }
  } else {
    if (state->status_code < 0 - match) {
      return 1;
    }
  }
  state->status_code += match;

  /* 强制最大值 */
  if (state->status_code > 999) {
    return 1;
  }
  return 0;
}

# 更新状态完成
int llhttp__on_status_complete(
    llhttp__internal_t* s, const unsigned char* p,
    const unsigned char* endp);

# 更新类型为 1
int llhttp__internal__c_update_type(
    llhttp__internal_t* state,
    const unsigned char* p,
    const unsigned char* endp) {
  state->type = 1;
  return 0;
}

# 更新类型为 1
int llhttp__internal__c_update_type_1(
    # 设置状态类型为2
    state->type = 2;
    # 返回0
    return 0;
}

// 初始化内部状态结构体，将其内容全部置为0
int llhttp__internal_init(llhttp__internal_t* state) {
  memset(state, 0, sizeof(*state));
  state->_current = (void*) (intptr_t) s_n_llhttp__internal__n_start;
  return 0;
}

// 运行内部状态机
static llparse_state_t llhttp__internal__run(
    llhttp__internal_t* state,
    const unsigned char* p,
    const unsigned char* endp) {
  int match;
  switch ((llparse_state_t) (intptr_t) state->_current) {
    case s_n_llhttp__internal__n_closed:
    s_n_llhttp__internal__n_closed: {
      // 如果指针p已经到达末尾，则返回n_closed状态
      if (p == endp) {
        return s_n_llhttp__internal__n_closed;
      }
      switch (*p) {
        case 10: {
          p++;
          goto s_n_llhttp__internal__n_closed;
        }
        case 13: {
          p++;
          goto s_n_llhttp__internal__n_closed;
        }
        default: {
          p++;
          goto s_n_llhttp__internal__n_error_4;
        }
      }
      /* UNREACHABLE */;
      abort();
    }
    case s_n_llhttp__internal__n_invoke_llhttp__after_message_complete:
    s_n_llhttp__internal__n_invoke_llhttp__after_message_complete: {
      // 调用llhttp__after_message_complete函数，根据返回值跳转到不同的状态
      switch (llhttp__after_message_complete(state, p, endp)) {
        case 1:
          goto s_n_llhttp__internal__n_invoke_update_finish_2;
        default:
          goto s_n_llhttp__internal__n_invoke_update_finish_1;
      }
      /* UNREACHABLE */;
      abort();
    }
    case s_n_llhttp__internal__n_pause_1:
    s_n_llhttp__internal__n_pause_1: {
      // 设置错误码和原因，并跳转到n_invoke_llhttp__after_message_complete状态
      state->error = 0x16;
      state->reason = "Pause on CONNECT/Upgrade";
      state->error_pos = (const char*) p;
      state->_current = (void*) (intptr_t) s_n_llhttp__internal__n_invoke_llhttp__after_message_complete;
      return s_error;
      /* UNREACHABLE */;
      abort();
    }
    case s_n_llhttp__internal__n_invoke_is_equal_upgrade:
    # 定义状态 s_n_llhttp__internal__n_invoke_is_equal_upgrade，处理判断是否需要升级协议的逻辑
    s_n_llhttp__internal__n_invoke_is_equal_upgrade: {
      # 根据当前状态和输入数据判断是否需要升级协议
      switch (llhttp__internal__c_is_equal_upgrade(state, p, endp)) {
        # 如果不需要升级协议，则跳转到处理消息完成后的状态
        case 0:
          goto s_n_llhttp__internal__n_invoke_llhttp__after_message_complete;
        # 如果需要升级协议，则暂停处理
        default:
          goto s_n_llhttp__internal__n_pause_1;
      }
      /* UNREACHABLE */;
      abort();
    }
    # 定义状态 s_n_llhttp__internal__n_invoke_llhttp__on_message_complete_2，处理消息完成后的逻辑
    case s_n_llhttp__internal__n_invoke_llhttp__on_message_complete_2:
    s_n_llhttp__internal__n_invoke_llhttp__on_message_complete_2: {
      # 根据当前状态和输入数据处理消息完成后的逻辑
      switch (llhttp__on_message_complete(state, p, endp)) {
        # 如果处理成功，则判断是否需要升级协议
        case 0:
          goto s_n_llhttp__internal__n_invoke_is_equal_upgrade;
        # 如果需要暂停处理，则暂停
        case 21:
          goto s_n_llhttp__internal__n_pause_5;
        # 如果出现错误，则跳转到错误处理状态
        default:
          goto s_n_llhttp__internal__n_error_13;
      }
      /* UNREACHABLE */;
      abort();
    }
    # 定义状态 s_n_llhttp__internal__n_chunk_data_almost_done，处理几乎完成数据块的逻辑
    case s_n_llhttp__internal__n_chunk_data_almost_done:
    s_n_llhttp__internal__n_chunk_data_almost_done: {
      # 定义匹配结果的结构
      llparse_match_t match_seq;

      # 如果输入数据已经处理完毕，则保持当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_chunk_data_almost_done;
      }
      # 根据当前状态和输入数据处理几乎完成数据块的逻辑
      match_seq = llparse__match_sequence_id(state, p, endp, llparse_blob0, 2);
      p = match_seq.current;
      # 根据匹配结果的状态进行处理
      switch (match_seq.status) {
        # 如果匹配完成，则跳转到处理数据块完成后的状态
        case kMatchComplete: {
          p++;
          goto s_n_llhttp__internal__n_invoke_llhttp__on_chunk_complete;
        }
        # 如果需要暂停处理，则保持当前状态
        case kMatchPause: {
          return s_n_llhttp__internal__n_chunk_data_almost_done;
        }
        # 如果匹配失败，则跳转到错误处理状态
        case kMatchMismatch: {
          goto s_n_llhttp__internal__n_error_8;
        }
      }
      /* UNREACHABLE */;
      abort();
    }
    # 定义状态 s_n_llhttp__internal__n_consume_content_length:
    # 检查可用数据长度和需要的数据长度
    s_n_llhttp__internal__n_consume_content_length: {
      size_t avail;  # 可用数据长度
      size_t need;   # 需要的数据长度

      avail = endp - p;  # 计算可用数据长度
      need = state->content_length;  # 获取需要的数据长度
      if (avail >= need) {  # 如果可用数据长度大于等于需要的数据长度
        p += need;  # 移动指针到需要的数据长度处
        state->content_length = 0;  # 将需要的数据长度置为0
        goto s_n_llhttp__internal__n_span_end_llhttp__on_body;  # 跳转到指定状态
      }

      state->content_length -= avail;  # 减去已经读取的数据长度
      return s_n_llhttp__internal__n_consume_content_length;  # 返回继续读取数据的状态
      /* UNREACHABLE */;  # 不可达代码
      abort();  # 终止程序
    }
    case s_n_llhttp__internal__n_span_start_llhttp__on_body:
    s_n_llhttp__internal__n_span_start_llhttp__on_body: {
      if (p == endp) {  # 如果指针指向末尾
        return s_n_llhttp__internal__n_span_start_llhttp__on_body;  # 返回当前状态
      }
      state->_span_pos0 = (void*) p;  # 设置状态的位置信息
      state->_span_cb0 = llhttp__on_body;  # 设置状态的回调函数
      goto s_n_llhttp__internal__n_consume_content_length;  # 跳转到指定状态
      /* UNREACHABLE */;  # 不可达代码
      abort();  # 终止程序
    }
    case s_n_llhttp__internal__n_invoke_is_equal_content_length:
    s_n_llhttp__internal__n_invoke_is_equal_content_length: {
      switch (llhttp__internal__c_is_equal_content_length(state, p, endp)) {  # 调用函数判断是否相等
        case 0:  # 如果不相等
          goto s_n_llhttp__internal__n_span_start_llhttp__on_body;  # 跳转到指定状态
        default:  # 如果相等
          goto s_n_llhttp__internal__n_invoke_or_flags;  # 跳转到指定状态
      }
      /* UNREACHABLE */;  # 不可达代码
      abort();  # 终止程序
    }
    case s_n_llhttp__internal__n_chunk_size_almost_done:
    s_n_llhttp__internal__n_chunk_size_almost_done: {
      if (p == endp) {  # 如果指针指向末尾
        return s_n_llhttp__internal__n_chunk_size_almost_done;  # 返回当前状态
      }
      switch (*p) {  # 检查当前字符
        case 10: {  # 如果是换行符
          p++;  # 移动指针到下一个字符
          goto s_n_llhttp__internal__n_invoke_llhttp__on_chunk_header;  # 跳转到指定状态
        }
        default: {  # 如果不是换行符
          goto s_n_llhttp__internal__n_error_9;  # 跳转到指定状态
        }
      }
      /* UNREACHABLE */;  # 不可达代码
      abort();  # 终止���序
    }
    case s_n_llhttp__internal__n_chunk_parameters:
    # 如果指针已经指向末尾，则返回当前状态
    s_n_llhttp__internal__n_chunk_parameters: {
      if (p == endp) {
        return s_n_llhttp__internal__n_chunk_parameters;
      }
      # 根据当前字符进行不同的处理
      switch (*p) {
        # 如果是回车符，则跳转到几乎完成状态
        case 13: {
          p++;
          goto s_n_llhttp__internal__n_chunk_size_almost_done;
        }
        # 如果是其他字符，则继续处理 chunk 参数
        default: {
          p++;
          goto s_n_llhttp__internal__n_chunk_parameters;
        }
      }
      # 不可达的代码，中止程序
      /* UNREACHABLE */;
      abort();
    }
    # 如果指针已经指向末尾，则返回当前状态
    case s_n_llhttp__internal__n_chunk_size_otherwise:
    s_n_llhttp__internal__n_chunk_size_otherwise: {
      if (p == endp) {
        return s_n_llhttp__internal__n_chunk_size_otherwise;
      }
      # 根据当前字符进行不同的处理
      switch (*p) {
        # 如果是回车符，则跳转到几乎完成状态
        case 13: {
          p++;
          goto s_n_llhttp__internal__n_chunk_size_almost_done;
        }
        # 如果是空格，则跳转到处理 chunk 参数
        case ' ': {
          p++;
          goto s_n_llhttp__internal__n_chunk_parameters;
        }
        # 如果是分号，则跳转到处理 chunk 参数
        case ';': {
          p++;
          goto s_n_llhttp__internal__n_chunk_parameters;
        }
        # 如果是其他字符，则跳转到错误状态
        default: {
          goto s_n_llhttp__internal__n_error_10;
        }
      }
      # 不可达的代码，中止程序
      /* UNREACHABLE */;
      abort();
    }
    # 其他状态的处理...
    case s_n_llhttp__internal__n_chunk_size:
    }
    case s_n_llhttp__internal__n_chunk_size_digit:
    }
    case s_n_llhttp__internal__n_invoke_update_content_length:
    s_n_llhttp__internal__n_invoke_update_content_length: {
      # 根据当前状态和字符更新内容长度
      switch (llhttp__internal__c_update_content_length(state, p, endp)) {
        default:
          goto s_n_llhttp__internal__n_chunk_size_digit;
      }
      # 不可达的代码，中止程序
      /* UNREACHABLE */;
      abort();
    }
    case s_n_llhttp__internal__n_consume_content_length_1:
    # 定义变量 avail 和 need，分别表示可用数据长度和需要的数据长度
    s_n_llhttp__internal__n_consume_content_length_1: {
      size_t avail;
      size_t need;

      # 计算可用数据长度
      avail = endp - p;
      # 获取需要的数据长度
      need = state->content_length;
      # 如果可用数据长度大于等于需要的数据长度
      if (avail >= need) {
        # 移动指针到需要的数据长度处
        p += need;
        # 将 state->content_length 置为 0
        state->content_length = 0;
        # 跳转到指定状态
        goto s_n_llhttp__internal__n_span_end_llhttp__on_body_1;
      }

      # 减去已经处理的数据长度
      state->content_length -= avail;
      # 返回当前状态
      return s_n_llhttp__internal__n_consume_content_length_1;
      /* UNREACHABLE */;
      # 终止程序
      abort();
    }
    # 处理开始解析 body 的状态
    case s_n_llhttp__internal__n_span_start_llhttp__on_body_1:
    s_n_llhttp__internal__n_span_start_llhttp__on_body_1: {
      # 如果指针指向末尾，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_span_start_llhttp__on_body_1;
      }
      # 设置 span_pos0 和 span_cb0，然后跳转到指定状态
      state->_span_pos0 = (void*) p;
      state->_span_cb0 = llhttp__on_body;
      goto s_n_llhttp__internal__n_consume_content_length_1;
      /* UNREACHABLE */;
      # 终止程序
      abort();
    }
    # 处理结束解析的状态
    case s_n_llhttp__internal__n_eof:
    s_n_llhttp__internal__n_eof: {
      # 如果指针指向末尾，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_eof;
      }
      # 移动指针到下一个位置
      p++;
      # 跳转到指定状态
      goto s_n_llhttp__internal__n_eof;
      /* UNREACHABLE */;
      # 终止程序
      abort();
    }
    # 处理开始解析 body 的状态
    case s_n_llhttp__internal__n_span_start_llhttp__on_body_2:
    s_n_llhttp__internal__n_span_start_llhttp__on_body_2: {
      # 如果指针指向末尾，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_span_start_llhttp__on_body_2;
      }
      # 设置 span_pos0 和 span_cb0，然后跳转到指定状态
      state->_span_pos0 = (void*) p;
      state->_span_cb0 = llhttp__on_body;
      goto s_n_llhttp__internal__n_eof;
      /* UNREACHABLE */;
      # 终止程序
      abort();
    }
    # 调用解析结束后的状态
    case s_n_llhttp__internal__n_invoke_llhttp__after_headers_complete:
    # 处理在解析 HTTP 头部完成后的状态转移
    s_n_llhttp__internal__n_invoke_llhttp__after_headers_complete: {
      # 根据解析结果进行不同的状态转移
      switch (llhttp__after_headers_complete(state, p, endp)) {
        # 如果解析成功，跳转到消息解析完成状态
        case 1:
          goto s_n_llhttp__internal__n_invoke_llhttp__on_message_complete_1;
        # 如果需要更新内容长度，跳转到更新内容长度状态
        case 2:
          goto s_n_llhttp__internal__n_invoke_update_content_length;
        # 如果需要开始解析消息体，跳转到消息体开始状态
        case 3:
          goto s_n_llhttp__internal__n_span_start_llhttp__on_body_1;
        # 如果需要更新完成状态，跳转到更新完成状态
        case 4:
          goto s_n_llhttp__internal__n_invoke_update_finish_3;
        # 如果出现错误，跳转到错误状态
        case 5:
          goto s_n_llhttp__internal__n_error_14;
        # 默认情况下，跳转到消息解析完成状态
        default:
          goto s_n_llhttp__internal__n_invoke_llhttp__on_message_complete;
      }
      # 不可达代码，中止程序
      /* UNREACHABLE */;
      abort();
    }
    # 处理在头部几乎完成时的状态转移
    case s_n_llhttp__internal__n_headers_almost_done:
    s_n_llhttp__internal__n_headers_almost_done: {
      # 如果指针已经到达结束位置，保持当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_headers_almost_done;
      }
      # 根据当前字符进行不同的状态转移
      switch (*p) {
        # 如果遇到换行符，跳转到测试标志状态
        case 10: {
          p++;
          goto s_n_llhttp__internal__n_invoke_test_flags;
        }
        # 默认情况下，跳转到错误状态
        default: {
          goto s_n_llhttp__internal__n_error_17;
        }
      }
      # 不可达代码，中止程序
      /* UNREACHABLE */;
      abort();
    }
    # 处理在头部值完成时的状态转移
    case s_n_llhttp__internal__n_invoke_llhttp__on_header_value_complete:
    s_n_llhttp__internal__n_invoke_llhttp__on_header_value_complete: {
      # 根据解析结果进行不同的状态转移
      switch (llhttp__on_header_value_complete(state, p, endp)) {
        # 默认情况下，跳转到头部字段开始状态
        default:
          goto s_n_llhttp__internal__n_header_field_start;
      }
      # 不可达代码，中止程序
      /* UNREACHABLE */;
      abort();
    }
    # 处理在头部值开始时的状态转移
    case s_n_llhttp__internal__n_span_start_llhttp__on_header_value:
    s_n_llhttp__internal__n_span_start_llhttp__on_header_value: {
      # 如果指针已经到达结束位置，保持当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_span_start_llhttp__on_header_value;
      }
      # 设置状态的起始位置和回调函数，然后跳转到头部值结束状态
      state->_span_pos0 = (void*) p;
      state->_span_cb0 = llhttp__on_header_value;
      goto s_n_llhttp__internal__n_span_end_llhttp__on_header_value;
      # 不可达代码，中止程序
      /* UNREACHABLE */;
      abort();
    }
    # 处理在丢弃头部值线性空白时的状态转移
    case s_n_llhttp__internal__n_header_value_discard_lws:
    # 如果指针已经到达末尾，则返回当前状态
    s_n_llhttp__internal__n_header_value_discard_lws: {
      if (p == endp) {
        return s_n_llhttp__internal__n_header_value_discard_lws;
      }
      # 根据当前字符进行不同的处理
      switch (*p) {
        # 如果是制表符，则指针向后移动，并跳转到丢弃空白字符的状态
        case 9: {
          p++;
          goto s_n_llhttp__internal__n_header_value_discard_ws;
        }
        # 如果是空格，则指针向后移动，并跳转到丢弃空白字符的状态
        case ' ': {
          p++;
          goto s_n_llhttp__internal__n_header_value_discard_ws;
        }
        # 其他情况则跳转到加载头部状态的处理
        default: {
          goto s_n_llhttp__internal__n_invoke_load_header_state;
        }
      }
      # 不可达的代码，中止程序
      /* UNREACHABLE */;
      abort();
    }
    # 如果指针已经到达末尾，则返回当前状态
    case s_n_llhttp__internal__n_header_value_discard_ws_almost_done:
    s_n_llhttp__internal__n_header_value_discard_ws_almost_done: {
      if (p == endp) {
        return s_n_llhttp__internal__n_header_value_discard_ws_almost_done;
      }
      # 根据当前字符进行不同的处理
      switch (*p) {
        # 如果是换行符，则指针向后移动，并跳转到丢弃空白字符的状态
        case 10: {
          p++;
          goto s_n_llhttp__internal__n_header_value_discard_lws;
        }
        # 其他情况则跳转到错误处理
        default: {
          goto s_n_llhttp__internal__n_error_19;
        }
      }
      # 不可达的代码，中止程序
      /* UNREACHABLE */;
      abort();
    }
    # 如果指针已经到达末尾，则返回当前状态
    case s_n_llhttp__internal__n_header_value_lws:
    s_n_llhttp__internal__n_header_value_lws: {
      if (p == endp) {
        return s_n_llhttp__internal__n_header_value_lws;
      }
      # 根据当前字符进行不同的处理
      switch (*p) {
        # 如果是制表符，则跳转到开始头部值的状态
        case 9: {
          goto s_n_llhttp__internal__n_span_start_llhttp__on_header_value_1;
        }
        # 如果是空格，则跳转到开始头部值的状态
        case ' ': {
          goto s_n_llhttp__internal__n_span_start_llhttp__on_header_value_1;
        }
        # 其他情况则跳转到加载头部状态的处理
        default: {
          goto s_n_llhttp__internal__n_invoke_load_header_state_3;
        }
      }
      # 不可达的代码，中止程序
      /* UNREACHABLE */;
      abort();
    }
    # 如果指针已经到达末尾，则返回当前状态
    case s_n_llhttp__internal__n_header_value_almost_done:
    # 如果指针 p 等于结束指针 endp，则返回状态 s_n_llhttp__internal__n_header_value_almost_done
    s_n_llhttp__internal__n_header_value_almost_done: {
      if (p == endp) {
        return s_n_llhttp__internal__n_header_value_almost_done;
      }
      # 根据指针 p 指向的值进行不同的处理
      switch (*p) {
        # 如果指针 p 指向的值为 10（换行符），则指针 p 向后移动一位，跳转到状态 s_n_llhttp__internal__n_header_value_lws
        case 10: {
          p++;
          goto s_n_llhttp__internal__n_header_value_lws;
        }
        # 如果指针 p 指向的值为其它值，则跳转到状态 s_n_llhttp__internal__n_error_20
        default: {
          goto s_n_llhttp__internal__n_error_20;
        }
      }
      # 不可达的代码，中止程序
      /* UNREACHABLE */;
      abort();
    }
    # 如果当前状态为 s_n_llhttp__internal__n_header_value_lenient，则执行以下代码
    case s_n_llhttp__internal__n_header_value_lenient:
    s_n_llhttp__internal__n_header_value_lenient: {
      # 如果指针 p 等于结束指针 endp，则返回状态 s_n_llhttp__internal__n_header_value_lenient
      if (p == endp) {
        return s_n_llhttp__internal__n_header_value_lenient;
      }
      # 根据指针 p 指向的值进行不同的处理
      switch (*p) {
        # 如果指针 p 指向的值为 10（换行符），则跳转到状态 s_n_llhttp__internal__n_span_end_llhttp__on_header_value_1
        case 10: {
          goto s_n_llhttp__internal__n_span_end_llhttp__on_header_value_1;
        }
        # 如果指针 p 指向的值为 13（回车符），则跳转到状态 s_n_llhttp__internal__n_span_end_llhttp__on_header_value_3
        case 13: {
          goto s_n_llhttp__internal__n_span_end_llhttp__on_header_value_3;
        }
        # 如果指针 p 指向的值为其它值，则指针 p 向后移动一位，跳转到状态 s_n_llhttp__internal__n_header_value_lenient
        default: {
          p++;
          goto s_n_llhttp__internal__n_header_value_lenient;
        }
      }
      # 不可达的代码，中止程序
      /* UNREACHABLE */;
      abort();
    }
    # 如果当前状态为 s_n_llhttp__internal__n_header_value_otherwise，则执行以下代码
    case s_n_llhttp__internal__n_header_value_otherwise:
    s_n_llhttp__internal__n_header_value_otherwise: {
      # 如果指针 p 等于结束指针 endp，则返回状态 s_n_llhttp__internal__n_header_value_otherwise
      if (p == endp) {
        return s_n_llhttp__internal__n_header_value_otherwise;
      }
      # 根据指针 p 指向的值进行不同的处理
      switch (*p) {
        # 如果指针 p 指向的值为 10（换行符），则跳转到状态 s_n_llhttp__internal__n_span_end_llhttp__on_header_value_1
        case 10: {
          goto s_n_llhttp__internal__n_span_end_llhttp__on_header_value_1;
        }
        # 如果指针 p 指向的值为 13（回车符），则跳转到状态 s_n_llhttp__internal__n_span_end_llhttp__on_header_value_2
        case 13: {
          goto s_n_llhttp__internal__n_span_end_llhttp__on_header_value_2;
        }
        # 如果指针 p 指向的值为其它值，则跳转到状态 s_n_llhttp__internal__n_invoke_test_lenient_flags_2
        default: {
          goto s_n_llhttp__internal__n_invoke_test_lenient_flags_2;
        }
      }
      # 不可达的代码，中止程序
      /* UNREACHABLE */;
      abort();
    }
    # 如果当前状态为 s_n_llhttp__internal__n_header_value_connection_token，则执行以下代码
    case s_n_llhttp__internal__n_header_value_connection_token:
    # 定义静态数组，用于查找表
    s_n_llhttp__internal__n_header_value_connection_token: {
      static uint8_t lookup_table[] = {
        # 静态查找表的数值
        0, 0, 0, 0, 0, 0, 0, 0, 0, 1, 0, 0, 0, 0, 0, 0,
        0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0,
        1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 2, 1, 1, 1,
        1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1,
        1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1,
        1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1,
        1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 0,
        1, 1, 1, 1, 1, 1, 1
    # 如果当前指针已经到达结束位置，则返回当前状态
    s_n_llhttp__internal__n_header_value_connection_ws: {
      if (p == endp) {
        return s_n_llhttp__internal__n_header_value_connection_ws;
      }
      # 根据当前字符进行不同的处理
      switch (*p) {
        # 如果是换行符或回车符，则跳转到其他状态
        case 10: {
          goto s_n_llhttp__internal__n_header_value_otherwise;
        }
        case 13: {
          goto s_n_llhttp__internal__n_header_value_otherwise;
        }
        # 如果是空格，则移动指针并保持当前状态
        case ' ': {
          p++;
          goto s_n_llhttp__internal__n_header_value_connection_ws;
        }
        # 如果是逗号，则移动指针并跳转到其他状态
        case ',': {
          p++;
          goto s_n_llhttp__internal__n_invoke_load_header_state_4;
        }
        # 其他情况，则跳转到其他状态
        default: {
          goto s_n_llhttp__internal__n_invoke_update_header_state_4;
        }
      }
      # 不可达的代码，中止程序
      /* UNREACHABLE */;
      abort();
    }
    # 如果当前状态为 s_n_llhttp__internal__n_header_value_connection_1，则执行以下操作
    case s_n_llhttp__internal__n_header_value_connection_1:
    s_n_llhttp__internal__n_header_value_connection_1: {
      llparse_match_t match_seq;

      # 如果当前指针已经到达结束位置，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_header_value_connection_1;
      }
      # 尝试匹配指定序列，根据匹配结果执行不同的操作
      match_seq = llparse__match_sequence_to_lower(state, p, endp, llparse_blob3, 4);
      p = match_seq.current;
      switch (match_seq.status) {
        # 如果匹配完成，则移动指针并跳转到其他状态
        case kMatchComplete: {
          p++;
          goto s_n_llhttp__internal__n_invoke_update_header_state_2;
        }
        # 如果匹配暂停，则返回当前状态
        case kMatchPause: {
          return s_n_llhttp__internal__n_header_value_connection_1;
        }
        # 如果匹配不成功，则跳转到其他状态
        case kMatchMismatch: {
          goto s_n_llhttp__internal__n_header_value_connection_token;
        }
      }
      # 不可达的代码，中止程序
      /* UNREACHABLE */;
      abort();
    }
    # 如果当前状态为 s_n_llhttp__internal__n_header_value_connection_2，则执行以下操作
    case s_n_llhttp__internal__n_header_value_connection_2:
    # 定义状态 s_n_llhttp__internal__n_header_value_connection_2，处理 Connection 头部值的第二部分
    s_n_llhttp__internal__n_header_value_connection_2: {
      # 定义匹配结果变量
      llparse_match_t match_seq;

      # 如果指针已经到达结束位置，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_header_value_connection_2;
      }
      # 调用函数，匹配指定长度的字符串并转换为小写
      match_seq = llparse__match_sequence_to_lower(state, p, endp, llparse_blob4, 9);
      # 更新指针位置
      p = match_seq.current;
      # 根据匹配结果进行不同的处理
      switch (match_seq.status) {
        # 如果匹配完成，则继续处理下一个字符
        case kMatchComplete: {
          p++;
          goto s_n_llhttp__internal__n_invoke_update_header_state_5;
        }
        # 如果匹配暂停，则返回当前状态
        case kMatchPause: {
          return s_n_llhttp__internal__n_header_value_connection_2;
        }
        # 如果匹配不成功，则转到处理 Connection 头部值的 token 部分
        case kMatchMismatch: {
          goto s_n_llhttp__internal__n_header_value_connection_token;
        }
      }
      # 不可达的代码，中断程序执行
      /* UNREACHABLE */;
      abort();
    }
    # 定义状态 s_n_llhttp__internal__n_header_value_connection_3，处理 Connection 头部值的第三部分
    case s_n_llhttp__internal__n_header_value_connection_3:
    s_n_llhttp__internal__n_header_value_connection_3: {
      # 定义匹配结果变量
      llparse_match_t match_seq;

      # 如果指针已经到达结束位置，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_header_value_connection_3;
      }
      # 调用函数，匹配指定长度的字符串并转换为小写
      match_seq = llparse__match_sequence_to_lower(state, p, endp, llparse_blob5, 6);
      # 更新指针位置
      p = match_seq.current;
      # 根据匹配结果进行不同的处理
      switch (match_seq.status) {
        # 如果匹配完成，则继续处理下一个字符
        case kMatchComplete: {
          p++;
          goto s_n_llhttp__internal__n_invoke_update_header_state_6;
        }
        # 如果匹配暂停，则返回当前状态
        case kMatchPause: {
          return s_n_llhttp__internal__n_header_value_connection_3;
        }
        # 如果匹配不成功，则转到处理 Connection 头部值的 token 部分
        case kMatchMismatch: {
          goto s_n_llhttp__internal__n_header_value_connection_token;
        }
      }
      # 不可达的代码，中断程序执行
      /* UNREACHABLE */;
      abort();
    }
    # 定义状态 s_n_llhttp__internal__n_header_value_connection，处理 Connection 头部值的第一部分
    # 如果指针已经指向末尾，则返回当前状态
    s_n_llhttp__internal__n_header_value_connection: {
      if (p == endp) {
        return s_n_llhttp__internal__n_header_value_connection;
      }
      # 判断当前字符是否为大写字母，如果是则转换为小写字母
      switch (((*p) >= 'A' && (*p) <= 'Z' ? (*p | 0x20) : (*p))) {
        # 如果是制表符，则指针向后移动，继续处理连接头部值
        case 9: {
          p++;
          goto s_n_llhttp__internal__n_header_value_connection;
        }
        # 如果是空格，则指针向后移动，继续处理连接头部值
        case ' ': {
          p++;
          goto s_n_llhttp__internal__n_header_value_connection;
        }
        # 如果是 'c'，则指针向后移动，进入处理连接头部值的状态1
        case 'c': {
          p++;
          goto s_n_llhttp__internal__n_header_value_connection_1;
        }
        # 如果是 'k'，则指针向后移动，进入处理连接头部值的状态2
        case 'k': {
          p++;
          goto s_n_llhttp__internal__n_header_value_connection_2;
        }
        # 如果是 'u'，则指针向后移动，进入处理连接头部值的状态3
        case 'u': {
          p++;
          goto s_n_llhttp__internal__n_header_value_connection_3;
        }
        # 如果不是以上情况，则进入处理连接头部值的通用状态
        default: {
          goto s_n_llhttp__internal__n_header_value_connection_token;
        }
      }
      /* UNREACHABLE */;
      # 终止程序执行
      abort();
    }
    # 如果当前状态为错误状态23
    case s_n_llhttp__internal__n_error_23:
    s_n_llhttp__internal__n_error_23: {
      # 设置错误码为11
      state->error = 0xb;
      # 设置错误原因为"Content-Length overflow"
      state->reason = "Content-Length overflow";
      # 设置错误位置为当前指针位置
      state->error_pos = (const char*) p;
      # 设置当前状态为错误状态
      state->_current = (void*) (intptr_t) s_error;
      # 返回错误状态
      return s_error;
      /* UNREACHABLE */;
      # 终止程序执行
      abort();
    }
    # 如果当前状态为错误状态24
    case s_n_llhttp__internal__n_error_24:
    s_n_llhttp__internal__n_error_24: {
      # 设置错误码为11
      state->error = 0xb;
      # 设置错误原因为"Invalid character in Content-Length"
      state->reason = "Invalid character in Content-Length";
      # 设置错误位置为当前指针位置
      state->error_pos = (const char*) p;
      # 设置当前状态为错误状态
      state->_current = (void*) (intptr_t) s_error;
      # 返回错误状态
      return s_error;
      /* UNREACHABLE */;
      # 终止程序执行
      abort();
    }
    # 如果当前状态为处理内容长度的空白字符状态
    case s_n_llhttp__internal__n_header_value_content_length_ws:
    # 如果指针已经指向末尾，则返回当前状态
    if (p == endp) {
        return s_n_llhttp__internal__n_header_value_content_length_ws;
    }
    # 根据当前字符进行不同的处理
    switch (*p) {
        # 如果是换行符，则跳转到处理换行的状态
        case 10: {
            goto s_n_llhttp__internal__n_invoke_or_flags_15;
        }
        # 如果是回车符，则跳转到处理换行的状态
        case 13: {
            goto s_n_llhttp__internal__n_invoke_or_flags_15;
        }
        # 如果是空格，则移动指针并继续处理空格
        case ' ': {
            p++;
            goto s_n_llhttp__internal__n_header_value_content_length_ws;
        }
        # 如果是其他字符，则跳转到处理头部值结束的状态
        default: {
            goto s_n_llhttp__internal__n_span_end_llhttp__on_header_value_5;
        }
    }
    # 不可达的代码，中止程序
    /* UNREACHABLE */;
    abort();
    # 处理头部值为 content-length 的情况
    case s_n_llhttp__internal__n_header_value_content_length:
    # 如果指针已经到达末尾，则返回当前状态
    s_n_llhttp__internal__n_header_value_content_length: {
      if (p == endp) {
        return s_n_llhttp__internal__n_header_value_content_length;
      }
      # 根据当前字符进行不同的处理
      switch (*p) {
        # 如果是数字0，则移动指针，设置匹配值为0，跳转到下一个状态
        case '0': {
          p++;
          match = 0;
          goto s_n_llhttp__internal__n_invoke_mul_add_content_length_1;
        }
        # 如果是数字1，则移动指针，设置匹配值为1，跳转到下一个状态
        case '1': {
          p++;
          match = 1;
          goto s_n_llhttp__internal__n_invoke_mul_add_content_length_1;
        }
        # 如果是数字2，则移动指针，设置匹配值为2，跳转到下一个状态
        case '2': {
          p++;
          match = 2;
          goto s_n_llhttp__internal__n_invoke_mul_add_content_length_1;
        }
        # 如果是数字3，则移动指针，设置匹配值为3，跳转到下一个状态
        case '3': {
          p++;
          match = 3;
          goto s_n_llhttp__internal__n_invoke_mul_add_content_length_1;
        }
        # 如果是数字4，则移动指针，设置匹配值为4，跳转到下一个状态
        case '4': {
          p++;
          match = 4;
          goto s_n_llhttp__internal__n_invoke_mul_add_content_length_1;
        }
        # 如果是数字5，则移动指针，设置匹配值为5，跳转到下一个状态
        case '5': {
          p++;
          match = 5;
          goto s_n_llhttp__internal__n_invoke_mul_add_content_length_1;
        }
        # 如果是数字6，则移动指针，设置匹配值为6，跳转到下一个状态
        case '6': {
          p++;
          match = 6;
          goto s_n_llhttp__internal__n_invoke_mul_add_content_length_1;
        }
        # 如果是数字7，则移动指针，设置匹配值为7，跳转到下一个状态
        case '7': {
          p++;
          match = 7;
          goto s_n_llhttp__internal__n_invoke_mul_add_content_length_1;
        }
        # 如果是数字8，则移动指针，设置匹配值为8，跳转到下一个状态
        case '8': {
          p++;
          match = 8;
          goto s_n_llhttp__internal__n_invoke_mul_add_content_length_1;
        }
        # 如果是数字9，则移动指针，设置匹配值为9，跳转到下一个状态
        case '9': {
          p++;
          match = 9;
          goto s_n_llhttp__internal__n_invoke_mul_add_content_length_1;
        }
        # 如果不是数字，则跳转到下一个状态
        default: {
          goto s_n_llhttp__internal__n_header_value_content_length_ws;
        }
      }
      # 不可达的代码，中止程序
      /* UNREACHABLE */;
      abort();
    }
    # 处理下一个状态
    case s_n_llhttp__internal__n_header_value_te_chunked_last:
    # 如果指针已经指向末尾，则返回当前状态
    s_n_llhttp__internal__n_header_value_te_chunked_last: {
      if (p == endp) {
        return s_n_llhttp__internal__n_header_value_te_chunked_last;
      }
      # 根据当前字符进行不同的处理
      switch (*p) {
        # 如果是换行符，则跳转到更新头部状态
        case 10: {
          goto s_n_llhttp__internal__n_invoke_update_header_state_7;
        }
        # 如果是回车符，则跳转到更新头部状态
        case 13: {
          goto s_n_llhttp__internal__n_invoke_update_header_state_7;
        }
        # 如果是空格，则指针向后移动，继续处理当前状态
        case ' ': {
          p++;
          goto s_n_llhttp__internal__n_header_value_te_chunked_last;
        }
        # 其他情况，则跳转到处理分块编码的状态
        default: {
          goto s_n_llhttp__internal__n_header_value_te_chunked;
        }
      }
      /* UNREACHABLE */;
      # 终止程序
      abort();
    }
    # 处理分块编码中的空白符
    case s_n_llhttp__internal__n_header_value_te_token_ows:
    s_n_llhttp__internal__n_header_value_te_token_ows: {
      # 如果指针已经指向末尾，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_header_value_te_token_ows;
      }
      # 根据当前字符进行不同的处理
      switch (*p) {
        # 如果是制表符，则指针向后移动
        case 9: {
          p++;
          goto s_n_llhttp__internal__n_header_value_te_token_ows;
        }
        # 如果是空格，则指针向后移动
        case ' ': {
          p++;
          goto s_n_llhttp__internal__n_header_value_te_token_ows;
        }
        # 其他情况，则跳转到处理分块编码的状态
        default: {
          goto s_n_llhttp__internal__n_header_value_te_chunked;
        }
      }
      /* UNREACHABLE */;
      # 终止程序
      abort();
    }
    # 处理头部值
    case s_n_llhttp__internal__n_header_value:
    }
    # 处理分块编码中的标记
    case s_n_llhttp__internal__n_header_value_te_token:
    # 定义名为 s_n_llhttp__internal__n_header_value_te_token 的静态变量
    s_n_llhttp__internal__n_header_value_te_token: {
      # 定义名为 lookup_table 的静态数组，存储了一系列数字
      static uint8_t lookup_table[] = {
        # 数组内容省略
      };
      # 如果指针 p 等于指针 endp，则返回 s_n_llhttp__internal__n_header_value_te_token
      if (p == endp) {
        return s_n_llhttp__internal__n_header_value_te_token;
      }
      # 根据 lookup_table 中 *p 对应的值进行不同的操作
      switch (lookup_table[(uint8_t) *p]) {
        # 如果值为 1，则指针 p 向后移动一位，然后跳转到标签 s_n_llhttp__internal__n_header_value_te_token
        case 1: {
          p++;
          goto s_n_llhttp__internal__n_header_value_te_token;
        }
        # 如果值为 2，则指针 p 向后移动一位，然后跳转到标签 s_n_llhttp__internal__n_header_value_te_token_ows
        case 2: {
          p++;
          goto s_n_llhttp__internal__n_header_value_te_token_ows;
        }
        # 如果值不是 1 或 2，则跳转到标签 s_n_llhttp__internal__n_invoke_update_header_state_8
        default: {
          goto s_n_llhttp__internal__n_invoke_update_header_state_8;
        }
      }
      # 不可达的代码，中止程序
      /* UNREACHABLE */;
      abort();
    }
    # 定义名为 s_n_llhttp__internal__n_header_value_te_chunked 的情况
    case s_n_llhttp__internal__n_header_value_te_chunked:
    # 定义状态 s_n_llhttp__internal__n_header_value_te_chunked，用于处理解析 HTTP 头部值中的 chunked 编码
    s_n_llhttp__internal__n_header_value_te_chunked: {
      # 定义匹配结果变量
      llparse_match_t match_seq;

      # 如果指针已经到达结束位置，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_header_value_te_chunked;
      }
      # 尝试匹配指针位置开始的 7 个字符是否与 llparse_blob6 相同
      match_seq = llparse__match_sequence_to_lower_unsafe(state, p, endp, llparse_blob6, 7);
      p = match_seq.current;
      # 根据匹配结果进行不同的处理
      switch (match_seq.status) {
        # 如果匹配完成，则指针后移一位，跳转到 s_n_llhttp__internal__n_header_value_te_chunked_last 状态
        case kMatchComplete: {
          p++;
          goto s_n_llhttp__internal__n_header_value_te_chunked_last;
        }
        # 如果匹配暂停，则返回当前状态
        case kMatchPause: {
          return s_n_llhttp__internal__n_header_value_te_chunked;
        }
        # 如果匹配失败，则跳转到 s_n_llhttp__internal__n_header_value_te_token 状态
        case kMatchMismatch: {
          goto s_n_llhttp__internal__n_header_value_te_token;
        }
      }
      # 不可达的代码，中止程序
      /* UNREACHABLE */;
      abort();
    }
    # 定义状态 s_n_llhttp__internal__n_span_start_llhttp__on_header_value_1
    s_n_llhttp__internal__n_span_start_llhttp__on_header_value_1: {
      # 如果指针已经到达结束位置，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_span_start_llhttp__on_header_value_1;
      }
      # 设置状态的 span_pos0 和 span_cb0 属性
      state->_span_pos0 = (void*) p;
      state->_span_cb0 = llhttp__on_header_value;
      # 跳转到 s_n_llhttp__internal__n_invoke_load_header_state_2 状态
      goto s_n_llhttp__internal__n_invoke_load_header_state_2;
      # 不可达的代码，中止程序
      /* UNREACHABLE */;
      abort();
    }
    # 定义状态 s_n_llhttp__internal__n_header_value_discard_ws
    s_n_llhttp__internal__n_header_value_discard_ws: {
      # 如果指针已经到达结束位置，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_header_value_discard_ws;
      }
      # 根据当前字符进行不同的处理
      switch (*p) {
        # 如果是制表符，则指针后移一位，跳转到 s_n_llhttp__internal__n_header_value_discard_ws
        case 9: {
          p++;
          goto s_n_llhttp__internal__n_header_value_discard_ws;
        }
        # 如果是换行符，则指针后移一位，跳转到 s_n_llhttp__internal__n_header_value_discard_lws
        case 10: {
          p++;
          goto s_n_llhttp__internal__n_header_value_discard_lws;
        }
        # 如果是回车符，则指针后移一位，跳转到 s_n_llhttp__internal__n_header_value_discard_ws_almost_done
        case 13: {
          p++;
          goto s_n_llhttp__internal__n_header_value_discard_ws_almost_done;
        }
        # 如果是空格，则指针后移一位，跳转到 s_n_llhttp__internal__n_header_value_discard_ws
        case ' ': {
          p++;
          goto s_n_llhttp__internal__n_header_value_discard_ws;
        }
        # 其他情况，跳转到 s_n_llhttp__internal__n_span_start_llhttp__on_header_value_1
        default: {
          goto s_n_llhttp__internal__n_span_start_llhttp__on_header_value_1;
        }
      }
      # 不可达的代码，中止程序
      /* UNREACHABLE */;
      abort();
    }
    // 当状态为处理完 header field 时
    case s_n_llhttp__internal__n_invoke_llhttp__on_header_field_complete: {
      // 调用处理完 header field 的函数，并根据返回值进行不同的处理
      switch (llhttp__on_header_field_complete(state, p, endp)) {
        default:
          // 转到处理 header value 中丢弃空白字符的状态
          goto s_n_llhttp__internal__n_header_value_discard_ws;
      }
      /* UNREACHABLE */;
      // 终止程序
      abort();
    }
    // 当状态为处理其他 header field 时
    case s_n_llhttp__internal__n_header_field_general_otherwise: {
      // 如果指针已经到达结束位置，则保持当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_header_field_general_otherwise;
      }
      // 根据当前字符进行不同的处理
      switch (*p) {
        case ':': {
          // 转到处理 header field 结束的状态
          goto s_n_llhttp__internal__n_span_end_llhttp__on_header_field_1;
        }
        default: {
          // 转到处理错误的状态
          goto s_n_llhttp__internal__n_error_25;
        }
      }
      /* UNREACHABLE */;
      // 终止程序
      abort();
    }
    // 当状态为处理一般的 header field 时
    case s_n_llhttp__internal__n_header_field_general:
    }
    // 当状态为处理 header field 冒号时
    case s_n_llhttp__internal__n_header_field_colon: {
      // 如果指针已经到达结束位置，则保持当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_header_field_colon;
      }
      // 根据当前字符进行不同的处理
      switch (*p) {
        case ' ': {
          // 移动指针并保持当前状态
          p++;
          goto s_n_llhttp__internal__n_header_field_colon;
        }
        case ':': {
          // 转到处理 header field 结束的状态
          goto s_n_llhttp__internal__n_span_end_llhttp__on_header_field;
        }
        default: {
          // 转到更新 header state 的状态
          goto s_n_llhttp__internal__n_invoke_update_header_state_9;
        }
      }
      /* UNREACHABLE */;
      // 终止程序
      abort();
    }
    // 当状态为处理 header field 3 时
    case s_n_llhttp__internal__n_header_field_3:
    # 定义状态 s_n_llhttp__internal__n_header_field_3，用于解析 HTTP 头部字段
    s_n_llhttp__internal__n_header_field_3: {
      # 定义变量 match_seq，用于存储匹配结果
      llparse_match_t match_seq;

      # 如果指针 p 已经指向输入结束位置，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_header_field_3;
      }
      # 调用 llparse__match_sequence_to_lower 函数，将输入数据转换为小写，并进行匹配
      match_seq = llparse__match_sequence_to_lower(state, p, endp, llparse_blob2, 6);
      # 更新指针位置
      p = match_seq.current;
      # 根据匹配结果进行不同的处理
      switch (match_seq.status) {
        # 如果匹配完成，则向前移动一个字符，设置匹配标志为 1，跳转到存储头部状态的处理状态
        case kMatchComplete: {
          p++;
          match = 1;
          goto s_n_llhttp__internal__n_invoke_store_header_state;
        }
        # 如果匹配暂停，则返回当前状态
        case kMatchPause: {
          return s_n_llhttp__internal__n_header_field_3;
        }
        # 如果匹配失败，则跳转到更新头部状态的处理状态
        case kMatchMismatch: {
          goto s_n_llhttp__internal__n_invoke_update_header_state_10;
        }
      }
      /* UNREACHABLE */;
      # 如果执行到这里，表示代码逻辑错误，中止程序
      abort();
    }
    # 定义状态 s_n_llhttp__internal__n_header_field_4，用于解析 HTTP 头部字段
    case s_n_llhttp__internal__n_header_field_4:
    s_n_llhttp__internal__n_header_field_4: {
      # 定义变量 match_seq，用于存储匹配结果
      llparse_match_t match_seq;

      # 如果指针 p 已经指向输入结束位置，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_header_field_4;
      }
      # 调用 llparse__match_sequence_to_lower 函数，将输入数据转换为小写，并进行匹配
      match_seq = llparse__match_sequence_to_lower(state, p, endp, llparse_blob10, 10);
      # 更新指针位置
      p = match_seq.current;
      # 根据匹配结果进行不同的处理
      switch (match_seq.status) {
        # 如果匹配完成，则向前移动一个字符，设置匹配标志为 2，跳转到存储头部状态的处理状态
        case kMatchComplete: {
          p++;
          match = 2;
          goto s_n_llhttp__internal__n_invoke_store_header_state;
        }
        # 如果匹配暂停，则返回当前状态
        case kMatchPause: {
          return s_n_llhttp__internal__n_header_field_4;
        }
        # 如果匹配失败，则跳转到更新头部状态的处理状态
        case kMatchMismatch: {
          goto s_n_llhttp__internal__n_invoke_update_header_state_10;
        }
      }
      /* UNREACHABLE */;
      # 如果执行到这里，表示代码逻辑错误，中止程序
      abort();
    }
    # 定义状态 s_n_llhttp__internal__n_header_field_2:
    # 定义状态机的状态 s_n_llhttp__internal__n_header_field_2
    s_n_llhttp__internal__n_header_field_2: {
      # 如果指针 p 指向结束位置，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_header_field_2;
      }
      # 判断当前字符是否为大写字母，如果是则转换为小写字母，然后进行 switch 分支判断
      switch (((*p) >= 'A' && (*p) <= 'Z' ? (*p | 0x20) : (*p))) {
        # 如果当前字符为 'n'，则指针向后移动一位，跳转到状态 s_n_llhttp__internal__n_header_field_3
        case 'n': {
          p++;
          goto s_n_llhttp__internal__n_header_field_3;
        }
        # 如果当前字符为 't'，则指针向后移动一位，跳转到状态 s_n_llhttp__internal__n_header_field_4
        case 't': {
          p++;
          goto s_n_llhttp__internal__n_header_field_4;
        }
        # 如果当前字符不是 'n' 或 't'，则跳转到状态 s_n_llhttp__internal__n_invoke_update_header_state_10
        default: {
          goto s_n_llhttp__internal__n_invoke_update_header_state_10;
        }
      }
      # 不可达的代码，中止程序
      /* UNREACHABLE */;
      abort();
    }
    # 定义状态机的状态 s_n_llhttp__internal__n_header_field_1
    case s_n_llhttp__internal__n_header_field_1:
    s_n_llhttp__internal__n_header_field_1: {
      # 定义状态机匹配结果的结构体
      llparse_match_t match_seq;

      # 如果指针 p 指向结束位置，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_header_field_1;
      }
      # 调用函数进行匹配，得到匹配结果
      match_seq = llparse__match_sequence_to_lower(state, p, endp, llparse_blob1, 2);
      # 更新指针位置
      p = match_seq.current;
      # 根据匹配结果进行 switch 分支判断
      switch (match_seq.status) {
        # 如果匹配完成，则指针向后移动一位，跳转到状态 s_n_llhttp__internal__n_header_field_2
        case kMatchComplete: {
          p++;
          goto s_n_llhttp__internal__n_header_field_2;
        }
        # 如果匹配暂停，则返回当前状态
        case kMatchPause: {
          return s_n_llhttp__internal__n_header_field_1;
        }
        # 如果匹配不成功，则跳转到状态 s_n_llhttp__internal__n_invoke_update_header_state_10
        case kMatchMismatch: {
          goto s_n_llhttp__internal__n_invoke_update_header_state_10;
        }
      }
      # 不可达的代码，中止程序
      /* UNREACHABLE */;
      abort();
    }
    # 定义状态机的状态 s_n_llhttp__internal__n_header_field_5:
    # 定义状态 s_n_llhttp__internal__n_header_field_5，用于解析 HTTP 请求头字段
    s_n_llhttp__internal__n_header_field_5: {
      # 定义变量 match_seq，用于存储匹配结果
      llparse_match_t match_seq;

      # 如果指针 p 已经指向输入结束位置，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_header_field_5;
      }
      # 调用 llparse__match_sequence_to_lower 函数，匹配输入数据是否符合指定的序列
      match_seq = llparse__match_sequence_to_lower(state, p, endp, llparse_blob11, 15);
      # 更新指针 p 的位置
      p = match_seq.current;
      # 根据匹配结果的状态进行不同的处理
      switch (match_seq.status) {
        # 如果匹配完成
        case kMatchComplete: {
          # 指针 p 向前移动一位
          p++;
          # 设置匹配标志为 1
          match = 1;
          # 跳转到状态 s_n_llhttp__internal__n_invoke_store_header_state
          goto s_n_llhttp__internal__n_invoke_store_header_state;
        }
        # 如果匹配暂停
        case kMatchPause: {
          # 返回当前状态
          return s_n_llhttp__internal__n_header_field_5;
        }
        # 如果匹配不成功
        case kMatchMismatch: {
          # 跳转到状态 s_n_llhttp__internal__n_invoke_update_header_state_10
          goto s_n_llhttp__internal__n_invoke_update_header_state_10;
        }
      }
      /* UNREACHABLE */;
      # 终止程序执行
      abort();
    }
    # 定义状态 s_n_llhttp__internal__n_header_field_6
    case s_n_llhttp__internal__n_header_field_6:
    s_n_llhttp__internal__n_header_field_6: {
      # 定义变量 match_seq，用于存储匹配结果
      llparse_match_t match_seq;

      # 如果指针 p 已经指向输入结束位置，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_header_field_6;
      }
      # 调用 llparse__match_sequence_to_lower 函数，匹配输入数据是否符合指定的序列
      match_seq = llparse__match_sequence_to_lower(state, p, endp, llparse_blob12, 16);
      # 更新指针 p 的位置
      p = match_seq.current;
      # 根据匹配结果的状态进行不同的处理
      switch (match_seq.status) {
        # 如果匹配完成
        case kMatchComplete: {
          # 指针 p 向前移动一位
          p++;
          # 设置匹配标志为 3
          match = 3;
          # 跳转到状态 s_n_llhttp__internal__n_invoke_store_header_state
          goto s_n_llhttp__internal__n_invoke_store_header_state;
        }
        # 如果匹配暂停
        case kMatchPause: {
          # 返回当前状态
          return s_n_llhttp__internal__n_header_field_6;
        }
        # 如果匹配不成功
        case kMatchMismatch: {
          # 跳转到状态 s_n_llhttp__internal__n_invoke_update_header_state_10
          goto s_n_llhttp__internal__n_invoke_update_header_state_10;
        }
      }
      /* UNREACHABLE */;
      # 终止程序执行
      abort();
    }
    # 定义状态 s_n_llhttp__internal__n_header_field_7:
    # 定义一个名为 s_n_llhttp__internal__n_header_field_7 的状态
    s_n_llhttp__internal__n_header_field_7: {
      # 定义一个名为 match_seq 的变量，用于存储匹配结果
      llparse_match_t match_seq;

      # 如果指针 p 等于结束指针 endp，则返回状态 s_n_llhttp__internal__n_header_field_7
      if (p == endp) {
        return s_n_llhttp__internal__n_header_field_7;
      }
      # 调用 llparse__match_sequence_to_lower 函数，将结果存储在 match_seq 中
      match_seq = llparse__match_sequence_to_lower(state, p, endp, llparse_blob13, 6);
      # 更新指针 p 的位置
      p = match_seq.current;
      # 根据匹配结果的状态进行不同的处理
      switch (match_seq.status) {
        # 如果匹配完成
        case kMatchComplete: {
          # 更新指针 p 的位置
          p++;
          # 设置 match 变量的值为 4
          match = 4;
          # 跳转到状态 s_n_llhttp__internal__n_invoke_store_header_state
          goto s_n_llhttp__internal__n_invoke_store_header_state;
        }
        # 如果匹配暂停
        case kMatchPause: {
          # 返回状态 s_n_llhttp__internal__n_header_field_7
          return s_n_llhttp__internal__n_header_field_7;
        }
        # 如果匹配不成功
        case kMatchMismatch: {
          # 跳转到状态 s_n_llhttp__internal__n_invoke_update_header_state_10
          goto s_n_llhttp__internal__n_invoke_update_header_state_10;
        }
      }
      # 不可达的代码，中断程序执行
      /* UNREACHABLE */;
      abort();
    }
    # 定义一个名为 s_n_llhttp__internal__n_header_field 的状态
    case s_n_llhttp__internal__n_header_field:
    s_n_llhttp__internal__n_header_field: {
      # 如果指针 p 等于结束指针 endp，则返回状态 s_n_llhttp__internal__n_header_field
      if (p == endp) {
        return s_n_llhttp__internal__n_header_field;
      }
      # 根据指针 p 指向的字符进行不同的处理
      switch (((*p) >= 'A' && (*p) <= 'Z' ? (*p | 0x20) : (*p))) {
        # 如果字符为 'c'
        case 'c': {
          # 更新指针 p 的位置
          p++;
          # 跳转到状态 s_n_llhttp__internal__n_header_field_1
          goto s_n_llhttp__internal__n_header_field_1;
        }
        # 如果字符为 'p'
        case 'p': {
          # 更新指针 p 的位置
          p++;
          # 跳转到状态 s_n_llhttp__internal__n_header_field_5
          goto s_n_llhttp__internal__n_header_field_5;
        }
        # 如果字符为 't'
        case 't': {
          # 更新指针 p 的位置
          p++;
          # 跳转到状态 s_n_llhttp__internal__n_header_field_6
          goto s_n_llhttp__internal__n_header_field_6;
        }
        # 如果字符为 'u'
        case 'u': {
          # 更新指针 p 的位置
          p++;
          # 跳转到状态 s_n_llhttp__internal__n_header_field_7
          goto s_n_llhttp__internal__n_header_field_7;
        }
        # 默认情况
        default: {
          # 跳转到状态 s_n_llhttp__internal__n_invoke_update_header_state_10
          goto s_n_llhttp__internal__n_invoke_update_header_state_10;
        }
      }
      # 不可达的代码，中断程序执行
      /* UNREACHABLE */;
      abort();
    }
    # 定义一个名为 s_n_llhttp__internal__n_span_start_llhttp__on_header_field 的状态
    case s_n_llhttp__internal__n_span_start_llhttp__on_header_field:
    s_n_llhttp__internal__n_span_start_llhttp__on_header_field: {
      # 如果指针 p 等于结束指针 endp，则返回状态 s_n_llhttp__internal__n_span_start_llhttp__on_header_field
      if (p == endp) {
        return s_n_llhttp__internal__n_span_start_llhttp__on_header_field;
      }
      # 设置 state 对象的 _span_pos0 属性为指针 p 的位置
      state->_span_pos0 = (void*) p;
      # 设置 state 对象的 _span_cb0 属性为 llhttp__on_header_field 函数
      state->_span_cb0 = llhttp__on_header_field;
      # 跳转到状态 s_n_llhttp__internal__n_header_field
      goto s_n_llhttp__internal__n_header_field;
      # 不可达的代码，中断程序执行
      /* UNREACHABLE */;
      abort();
    }
    # 定义一个名为 s_n_llhttp__internal__n_header_field_start 的状态
    case s_n_llhttp__internal__n_header_field_start:
    # 如果指针已经指向末尾，则返回当前状态
    s_n_llhttp__internal__n_header_field_start: {
      if (p == endp) {
        return s_n_llhttp__internal__n_header_field_start;
      }
      # 根据当前字符进行不同的处理
      switch (*p) {
        # 如果是换行符，则跳转到几乎完成状态
        case 10: {
          goto s_n_llhttp__internal__n_headers_almost_done;
        }
        # 如果是回车符，则向前移动一个字符，然后跳转到几乎完成状态
        case 13: {
          p++;
          goto s_n_llhttp__internal__n_headers_almost_done;
        }
        # 其他情况下，跳转到处理 header field 的状态
        default: {
          goto s_n_llhttp__internal__n_span_start_llhttp__on_header_field;
        }
      }
      # 不可达的代码，中止程序
      /* UNREACHABLE */;
      abort();
    }
    # 如果指针已经指向末尾，则返回当前状态
    case s_n_llhttp__internal__n_url_to_http_09:
    s_n_llhttp__internal__n_url_to_http_09: {
      if (p == endp) {
        return s_n_llhttp__internal__n_url_to_http_09;
      }
      # 根据当前字符进行不同的处理
      switch (*p) {
        # 如果是制表符，则向前移动一个字符，然后跳转到错误状态
        case 9: {
          p++;
          goto s_n_llhttp__internal__n_error_1;
        }
        # 如果是换页符，则向前移动一个字符，然后跳转到错误状态
        case 12: {
          p++;
          goto s_n_llhttp__internal__n_error_1;
        }
        # 其他情况下，跳转到更新 http major 版本号的状态
        default: {
          goto s_n_llhttp__internal__n_invoke_update_http_major;
        }
      }
      # 不可达的代码，中止程序
      /* UNREACHABLE */;
      abort();
    }
    # 如果指针已经指向末尾，则返回当前状态
    case s_n_llhttp__internal__n_url_skip_to_http09:
    s_n_llhttp__internal__n_url_skip_to_http09: {
      if (p == endp) {
        return s_n_llhttp__internal__n_url_skip_to_http09;
      }
      # 根据当前字符进行不同的处理
      switch (*p) {
        # 如果是制表符，则向前移动一个字符，然后跳转到错误状态
        case 9: {
          p++;
          goto s_n_llhttp__internal__n_error_1;
        }
        # 如果是换页符，则向前移动一个字符，然后跳转到错误状态
        case 12: {
          p++;
          goto s_n_llhttp__internal__n_error_1;
        }
        # 其他情况下，向前移动一个字符，然后跳转到转换为 http 0.9 的状态
        default: {
          p++;
          goto s_n_llhttp__internal__n_url_to_http_09;
        }
      }
      # 不可达的代码，中止程序
      /* UNREACHABLE */;
      abort();
    }
    # 如果指针已经指向末尾，则返回当前状态
    case s_n_llhttp__internal__n_url_skip_lf_to_http09_1:
    # 如果指针已经到达末尾，则返回当前状态
    s_n_llhttp__internal__n_url_skip_lf_to_http09_1: {
      if (p == endp) {
        return s_n_llhttp__internal__n_url_skip_lf_to_http09_1;
      }
      # 根据当前字符进行不同的处理
      switch (*p) {
        # 如果是换行符，指针向后移动一位，跳转到状态 s_n_llhttp__internal__n_url_to_http_09
        case 10: {
          p++;
          goto s_n_llhttp__internal__n_url_to_http_09;
        }
        # 如果是其他字符，跳转到错误状态 s_n_llhttp__internal__n_error_26
        default: {
          goto s_n_llhttp__internal__n_error_26;
        }
      }
      /* UNREACHABLE */;
      abort();
    }
    # 如果当前状态为 s_n_llhttp__internal__n_url_skip_lf_to_http09
    case s_n_llhttp__internal__n_url_skip_lf_to_http09:
    s_n_llhttp__internal__n_url_skip_lf_to_http09: {
      # 如果指针已经到达末尾，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_url_skip_lf_to_http09;
      }
      # 根据当前字符进行不同的处理
      switch (*p) {
        # 如果是制表符，指针向后移动一位，跳转到错误状态 s_n_llhttp__internal__n_error_1
        case 9: {
          p++;
          goto s_n_llhttp__internal__n_error_1;
        }
        # 如果是换页符，指针向后移动一位，跳转到错误状态 s_n_llhttp__internal__n_error_1
        case 12: {
          p++;
          goto s_n_llhttp__internal__n_error_1;
        }
        # 如果是回车符，指针向后移动一位，跳转到状态 s_n_llhttp__internal__n_url_skip_lf_to_http09_1
        case 13: {
          p++;
          goto s_n_llhttp__internal__n_url_skip_lf_to_http09_1;
        }
        # 如果是其他字符，跳转到错误状态 s_n_llhttp__internal__n_error_26
        default: {
          goto s_n_llhttp__internal__n_error_26;
        }
      }
      /* UNREACHABLE */;
      abort();
    }
    # 如果当前状态为 s_n_llhttp__internal__n_req_pri_upgrade
    case s_n_llhttp__internal__n_req_pri_upgrade:
    s_n_llhttp__internal__n_req_pri_upgrade: {
      # 定义匹配结果的结构
      llparse_match_t match_seq;

      # 如果指针已经到达末尾，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_req_pri_upgrade;
      }
      # 进行匹配操作，根据匹配结果进行不同的处理
      match_seq = llparse__match_sequence_id(state, p, endp, llparse_blob15, 10);
      p = match_seq.current;
      switch (match_seq.status) {
        # 如果匹配完成，指针向后移动一位，跳转到错误状态 s_n_llhttp__internal__n_error_29
        case kMatchComplete: {
          p++;
          goto s_n_llhttp__internal__n_error_29;
        }
        # 如果匹配暂停，返回当前状态
        case kMatchPause: {
          return s_n_llhttp__internal__n_req_pri_upgrade;
        }
        # 如果匹配不成功，跳转到错误状态 s_n_llhttp__internal__n_error_30
        case kMatchMismatch: {
          goto s_n_llhttp__internal__n_error_30;
        }
      }
      /* UNREACHABLE */;
      abort();
    }
    # 如果当前状态为 s_n_llhttp__internal__n_req_http_complete_1
    case s_n_llhttp__internal__n_req_http_complete_1:
    # 如果当前指针位置等于结束指针位置，则返回当前状态
    s_n_llhttp__internal__n_req_http_complete_1: {
      if (p == endp) {
        return s_n_llhttp__internal__n_req_http_complete_1;
      }
      # 根据当前指针指向的内容进行不同的处理
      switch (*p) {
        # 如果当前指针指向换行符，则指针向后移动一位，跳转到下一个状态
        case 10: {
          p++;
          goto s_n_llhttp__internal__n_header_field_start;
        }
        # 如果指针指向其他内容，则跳转到错误状态
        default: {
          goto s_n_llhttp__internal__n_error_28;
        }
      }
      /* UNREACHABLE */;
      abort();
    }
    # 如果当前状态为请求行解析完成
    case s_n_llhttp__internal__n_req_http_complete:
    s_n_llhttp__internal__n_req_http_complete: {
      # 如果当前指针位置等于结束指针位置，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_req_http_complete;
      }
      # 根据当前指针指向的内容进行不同的处理
      switch (*p) {
        # 如果当前指针指向换行符，则指针向后移动一位，跳转到下一个状态
        case 10: {
          p++;
          goto s_n_llhttp__internal__n_header_field_start;
        }
        # 如果当前指针指向回车符，则指针向后移动一位，跳转到请求行解析完成的下一个状态
        case 13: {
          p++;
          goto s_n_llhttp__internal__n_req_http_complete_1;
        }
        # 如果指针指向其他内容，则跳转到错误状态
        default: {
          goto s_n_llhttp__internal__n_error_28;
        }
      }
      /* UNREACHABLE */;
      abort();
    }
    # 如果当前状态为请求行解析中的次版本号
    case s_n_llhttp__internal__n_req_http_minor:
    # 检查是否已经到达输入的末尾
    s_n_llhttp__internal__n_req_http_minor: {
      if (p == endp) {
        return s_n_llhttp__internal__n_req_http_minor;
      }
      # 根据当前字符进行不同的处理
      switch (*p) {
        # 如果是 '0'，则将匹配值设为 0，并跳转到存储 HTTP 版本号的状态
        case '0': {
          p++;
          match = 0;
          goto s_n_llhttp__internal__n_invoke_store_http_minor;
        }
        # 如果是 '1'，则将匹配值设为 1，并跳转到存储 HTTP 版本号的状态
        case '1': {
          p++;
          match = 1;
          goto s_n_llhttp__internal__n_invoke_store_http_minor;
        }
        # 如果是 '2'，则将匹配值设为 2，并跳转到存储 HTTP 版本号的状态
        case '2': {
          p++;
          match = 2;
          goto s_n_llhttp__internal__n_invoke_store_http_minor;
        }
        # 如果是 '3'，则将匹配值设为 3，并跳转到存储 HTTP 版本号的状态
        case '3': {
          p++;
          match = 3;
          goto s_n_llhttp__internal__n_invoke_store_http_minor;
        }
        # 如果是 '4'，则将匹配值设为 4，并跳转到存储 HTTP 版本号的状态
        case '4': {
          p++;
          match = 4;
          goto s_n_llhttp__internal__n_invoke_store_http_minor;
        }
        # 如果是 '5'，则将匹配值设为 5，并跳转到存储 HTTP 版本号的状态
        case '5': {
          p++;
          match = 5;
          goto s_n_llhttp__internal__n_invoke_store_http_minor;
        }
        # 如果是 '6'，则将匹配值设为 6，并跳转到存储 HTTP 版本号的状态
        case '6': {
          p++;
          match = 6;
          goto s_n_llhttp__internal__n_invoke_store_http_minor;
        }
        # 如果是 '7'，则将匹配值设为 7，并跳转到存储 HTTP 版本号的状态
        case '7': {
          p++;
          match = 7;
          goto s_n_llhttp__internal__n_invoke_store_http_minor;
        }
        # 如果是 '8'，则将匹配值设为 8，并跳转到存储 HTTP 版本号的状态
        case '8': {
          p++;
          match = 8;
          goto s_n_llhttp__internal__n_invoke_store_http_minor;
        }
        # 如果是 '9'，则将匹配值设为 9，并跳转到存储 HTTP 版本号的状态
        case '9': {
          p++;
          match = 9;
          goto s_n_llhttp__internal__n_invoke_store_http_minor;
        }
        # 如果是其他字符，则跳转到错误状态
        default: {
          goto s_n_llhttp__internal__n_error_31;
        }
      }
      /* UNREACHABLE */;
      # 终止程序
      abort();
    }
    # 处理 HTTP 版本号中的小数点
    case s_n_llhttp__internal__n_req_http_dot:
    # 如果指针已经指向末尾，则返回当前状态
    if (p == endp) {
        return s_n_llhttp__internal__n_req_http_dot;
    }
    # 根据当前字符进行不同的处理
    switch (*p) {
        # 如果是'.'，则指针向后移动，进入下一个状态
        case '.': {
            p++;
            goto s_n_llhttp__internal__n_req_http_minor;
        }
        # 如果是其他字符，则进入错误状态
        default: {
            goto s_n_llhttp__internal__n_error_32;
        }
    }
    # 不可达的代码，中止程序
    /* UNREACHABLE */;
    abort();
    # 进入下一个状态
    case s_n_llhttp__internal__n_req_http_major:
    # 检查请求行中 HTTP 版本号的主版本号
    s_n_llhttp__internal__n_req_http_major: {
      # 如果指针已经指向末尾，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_req_http_major;
      }
      # 根据指针指向的字符进行不同的处理
      switch (*p) {
        # 如果是 '0'，则将指针向后移动一位，匹配值设为 0，跳转到存储 HTTP 主版本号的状态
        case '0': {
          p++;
          match = 0;
          goto s_n_llhttp__internal__n_invoke_store_http_major;
        }
        # 如果是 '1'，则将指针向后移动一位，匹配值设为 1，跳转到存储 HTTP 主版本号的状态
        case '1': {
          p++;
          match = 1;
          goto s_n_llhttp__internal__n_invoke_store_http_major;
        }
        # 如果是 '2'，则将指针向后移动一位，匹配值设为 2，跳转到存储 HTTP 主版本号的状态
        case '2': {
          p++;
          match = 2;
          goto s_n_llhttp__internal__n_invoke_store_http_major;
        }
        # 如果是 '3'，则将指针向后移动一位，匹配值设为 3，跳转到存储 HTTP 主版本号的状态
        case '3': {
          p++;
          match = 3;
          goto s_n_llhttp__internal__n_invoke_store_http_major;
        }
        # 如果是 '4'，则将指针向后移动一位，匹配值设为 4，跳转到存储 HTTP 主版本号的状态
        case '4': {
          p++;
          match = 4;
          goto s_n_llhttp__internal__n_invoke_store_http_major;
        }
        # 如果是 '5'，则将指针向后移动一位，匹配值设为 5，跳转到存储 HTTP 主版本号的状态
        case '5': {
          p++;
          match = 5;
          goto s_n_llhttp__internal__n_invoke_store_http_major;
        }
        # 如果是 '6'，则将指针向后移动一位，匹配值设为 6，跳转到存储 HTTP 主版本号的状态
        case '6': {
          p++;
          match = 6;
          goto s_n_llhttp__internal__n_invoke_store_http_major;
        }
        # 如果是 '7'，则将指针向后移动一位，匹配值设为 7，跳转到存储 HTTP 主版本号的状态
        case '7': {
          p++;
          match = 7;
          goto s_n_llhttp__internal__n_invoke_store_http_major;
        }
        # 如果是 '8'，则将指针向后移动一位，匹配值设为 8，跳转到存储 HTTP 主版本号的状态
        case '8': {
          p++;
          match = 8;
          goto s_n_llhttp__internal__n_invoke_store_http_major;
        }
        # 如果是 '9'，则将指针向后移动一位，匹配值设为 9，跳转到存储 HTTP 主版本号的状态
        case '9': {
          p++;
          match = 9;
          goto s_n_llhttp__internal__n_invoke_store_http_major;
        }
        # 如果不是上述任何字符，则跳转到错误状态
        default: {
          goto s_n_llhttp__internal__n_error_33;
        }
      }
      # 不可达的代码，中止程序
      /* UNREACHABLE */;
      abort();
    }
    # 处理请求行中 HTTP 版本号的起始状态
    case s_n_llhttp__internal__n_req_http_start_1:
    # 定义状态 s_n_llhttp__internal__n_req_http_start_1，包含匹配结果的变量
    s_n_llhttp__internal__n_req_http_start_1: {
      llparse_match_t match_seq;

      # 如果指针已经到达结束位置，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_req_http_start_1;
      }
      # 调用 llparse__match_sequence_id 函数进行匹配，获取匹配结果
      match_seq = llparse__match_sequence_id(state, p, endp, llparse_blob14, 4);
      # 更新指针位置
      p = match_seq.current;
      # 根据匹配结果的状态进行不同的处理
      switch (match_seq.status) {
        # 如果匹配完成，则指针后移一位，跳转到下一个状态
        case kMatchComplete: {
          p++;
          goto s_n_llhttp__internal__n_invoke_load_method;
        }
        # 如果匹配暂停，则返回当前状态
        case kMatchPause: {
          return s_n_llhttp__internal__n_req_http_start_1;
        }
        # 如果匹配不成功，则跳转到错误状态
        case kMatchMismatch: {
          goto s_n_llhttp__internal__n_error_36;
        }
      }
      # 不可达的代码，中断程序执行
      /* UNREACHABLE */;
      abort();
    }
    # 定义状态 s_n_llhttp__internal__n_req_http_start_2
    case s_n_llhttp__internal__n_req_http_start_2:
    s_n_llhttp__internal__n_req_http_start_2: {
      llparse_match_t match_seq;

      # 如果指针已经到达结束位置，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_req_http_start_2;
      }
      # 调用 llparse__match_sequence_id 函数进行匹配，获取匹配结果
      match_seq = llparse__match_sequence_id(state, p, endp, llparse_blob16, 3);
      # 更新指针位置
      p = match_seq.current;
      # 根据匹配结果的状态进行不同的处理
      switch (match_seq.status) {
        # 如果匹配完成，则指针后移一位，跳转到下一个状态
        case kMatchComplete: {
          p++;
          goto s_n_llhttp__internal__n_invoke_load_method_2;
        }
        # 如果匹配暂停，则返回当前状态
        case kMatchPause: {
          return s_n_llhttp__internal__n_req_http_start_2;
        }
        # 如果匹配不成功，则跳转到错误状态
        case kMatchMismatch: {
          goto s_n_llhttp__internal__n_error_36;
        }
      }
      # 不可达的代码，中断程序执行
      /* UNREACHABLE */;
      abort();
    }
    # 定义状态 s_n_llhttp__internal__n_req_http_start_3
    case s_n_llhttp__internal__n_req_http_start_3:
    # 定义一个名为 s_n_llhttp__internal__n_req_http_start_3 的状态，用于处理 HTTP 请求的开始部分
    s_n_llhttp__internal__n_req_http_start_3: {
      # 定义一个名为 match_seq 的变量，用于存储匹配结果
      llparse_match_t match_seq;

      # 如果指针 p 已经指向输入的结尾，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_req_http_start_3;
      }
      # 调用 llparse__match_sequence_id 函数，匹配输入的数据和指定的序列 llparse_blob17，长度为 4
      match_seq = llparse__match_sequence_id(state, p, endp, llparse_blob17, 4);
      # 更新指针 p 的位置
      p = match_seq.current;
      # 根据匹配结果的状态进行不同的处理
      switch (match_seq.status) {
        # 如果匹配完成，则指针 p 向后移动一位，跳转到状态 s_n_llhttp__internal__n_invoke_load_method_3
        case kMatchComplete: {
          p++;
          goto s_n_llhttp__internal__n_invoke_load_method_3;
        }
        # 如果匹配暂停，则返回当前状态
        case kMatchPause: {
          return s_n_llhttp__internal__n_req_http_start_3;
        }
        # 如果匹配不成功，则跳转到状态 s_n_llhttp__internal__n_error_36
        case kMatchMismatch: {
          goto s_n_llhttp__internal__n_error_36;
        }
      }
      /* UNREACHABLE */;
      # 如果代码执行到这里，表示出现了不可达的情况，终止程序
      abort();
    }
    # 定义状态 s_n_llhttp__internal__n_req_http_start
    case s_n_llhttp__internal__n_req_http_start:
    s_n_llhttp__internal__n_req_http_start: {
      # 如果指针 p 已经指向输入的结尾，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_req_http_start;
      }
      # 根据指针 p 指向的字符进行不同的处理
      switch (*p) {
        # 如果是空格，则指针 p 向后移动一位，跳转到状态 s_n_llhttp__internal__n_req_http_start
        case ' ': {
          p++;
          goto s_n_llhttp__internal__n_req_http_start;
        }
        # 如果是 'H'，则指针 p 向后移动一位，跳转到状态 s_n_llhttp__internal__n_req_http_start_1
        case 'H': {
          p++;
          goto s_n_llhttp__internal__n_req_http_start_1;
        }
        # 如果是 'I'，则指针 p 向后移动一位，跳转到状态 s_n_llhttp__internal__n_req_http_start_2
        case 'I': {
          p++;
          goto s_n_llhttp__internal__n_req_http_start_2;
        }
        # 如果是 'R'，则指针 p 向后移动一位，跳转到状态 s_n_llhttp__internal__n_req_http_start_3
        case 'R': {
          p++;
          goto s_n_llhttp__internal__n_req_http_start_3;
        }
        # 如果是其他字符，则跳转到状态 s_n_llhttp__internal__n_error_36
        default: {
          goto s_n_llhttp__internal__n_error_36;
        }
      }
      /* UNREACHABLE */;
      # 如果代码执行到这里，表示出现了不可达的情况，终止程序
      abort();
    }
    # 定义状态 s_n_llhttp__internal__n_url_to_http
    case s_n_llhttp__internal__n_url_to_http:
    s_n_llhttp__internal__n_url_to_http: {
      # 如果指针 p 已经指向输入的结尾，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_url_to_http;
      }
      # 根据指针 p 指向的字符进行不同的处理
      switch (*p) {
        # 如果是水平制表符或换页符，则指针 p 向后移动一位，跳转到状态 s_n_llhttp__internal__n_error_1
        case 9: {
          p++;
          goto s_n_llhttp__internal__n_error_1;
        }
        case 12: {
          p++;
          goto s_n_llhttp__internal__n_error_1;
        }
        # 如果是其他字符，则跳转到状态 s_n_llhttp__internal__n_invoke_llhttp__on_url_complete_1
        default: {
          goto s_n_llhttp__internal__n_invoke_llhttp__on_url_complete_1;
        }
      }
      /* UNREACHABLE */;
      # 如果代码执行到这里，表示出现了不可达的情况，终止程序
      abort();
    }
    # 定义状态 s_n_llhttp__internal__n_url_skip_to_http:
    # 如果指针已经指向末尾，则返回当前状态
    if (p == endp) {
        return s_n_llhttp__internal__n_url_skip_to_http;
    }
    # 根据当前字符进行不同的处理
    switch (*p) {
        # 如果是制表符，则指针向后移动，跳转到错误状态1
        case 9: {
            p++;
            goto s_n_llhttp__internal__n_error_1;
        }
        # 如果是换页符，则指针向后移动，跳转到错误状态1
        case 12: {
            p++;
            goto s_n_llhttp__internal__n_error_1;
        }
        # 其他情况下，指针向后移动，跳转到处理 URL 的状态
        default: {
            p++;
            goto s_n_llhttp__internal__n_url_to_http;
        }
    }
    # 不可达的代码，中止程序
    /* UNREACHABLE */;
    abort();
    # 处理 URL 片段的情况
    case s_n_llhttp__internal__n_url_fragment:
    # 定义静态的查找表，用于根据输入字符的值进行快速查找
    s_n_llhttp__internal__n_url_fragment: {
      static uint8_t lookup_table[] = {
        # 查找表的数值，用于快速判断输入字符的类型
        0, 0, 0, 0, 0, 0, 0, 0, 0, 1, 2, 0, 1, 3, 0, 0,
        # ... 省略部分代码 ...
      };
      # 如果输入指针已经到达末尾，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_url_fragment;
      }
      # 根据查找表的值进行不同的处理
      switch (lookup_table[(uint8_t) *p]) {
        case 1: {
          p++;
          # 跳转到指定状态
          goto s_n_llhttp__internal__n_error_1;
        }
        case 2: {
          # 跳转到指定状态
          goto s_n_llhttp__internal__n_span_end_llhttp__on_url_6;
        }
        case 3: {
          # 跳转到指定状态
          goto s_n_llhttp__internal__n_span_end_llhttp__on_url_7;
        }
        case 4: {
          # 跳转到指定状态
          goto s_n_llhttp__internal__n_span_end_llhttp__on_url_8;
        }
        case 5: {
          p++;
          # 跳转到指定状态
          goto s_n_llhttp__internal__n_url_fragment;
        }
        default: {
          # 跳转到指定状态
          goto s_n_llhttp__internal__n_error_37;
        }
      }
      # 不可达的代码，中断程序执行
      /* UNREACHABLE */;
      abort();
    }
    # 处理状态 s_n_llhttp__internal__n_span_end_stub_query_3:
    case s_n_llhttp__internal__n_span_end_stub_query_3:
    # 定义状态 s_n_llhttp__internal__n_span_end_stub_query_3，处理 URL 查询部分
    s_n_llhttp__internal__n_span_end_stub_query_3: {
      # 如果指针 p 到达结束位置 endp，则返回状态 s_n_llhttp__internal__n_span_end_stub_query_3
      if (p == endp) {
        return s_n_llhttp__internal__n_span_end_stub_query_3;
      }
      # 指针 p 向前移动一位
      p++;
      # 跳转到状态 s_n_llhttp__internal__n_url_fragment
      goto s_n_llhttp__internal__n_url_fragment;
      # 不可达代码，用于标记不可达的代码段
      /* UNREACHABLE */;
      # 中止程序执行
      abort();
    }
    # 处理 URL 查询部分的状态
    case s_n_llhttp__internal__n_url_query:
    # 定义静态查找表，用于根据输入字符的值进行快速查找
    s_n_llhttp__internal__n_url_query: {
      static uint8_t lookup_table[] = {
        # 静态查找表的数值
        0, 0, 0, 0, 0, 0, 0, 0, 0, 1, 2, 0, 1, 3, 0, 0,
        # ... （省略部分代码）
      };
      # 如果指针指向末尾，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_url_query;
      }
      # 根据查找表的值进行不同的处理
      switch (lookup_table[(uint8_t) *p]) {
        case 1: {
          p++;
          # 跳转到错误状态1
          goto s_n_llhttp__internal__n_error_1;
        }
        case 2: {
          # 跳转到 URL 处理结束状态9
          goto s_n_llhttp__internal__n_span_end_llhttp__on_url_9;
        }
        case 3: {
          # 跳转到 URL 处理结束状态10
          goto s_n_llhttp__internal__n_span_end_llhttp__on_url_10;
        }
        case 4: {
          # 跳转到 URL 处理结束状态11
          goto s_n_llhttp__internal__n_span_end_llhttp__on_url_11;
        }
        case 5: {
          p++;
          # 继续处理 URL 查询部分
          goto s_n_llhttp__internal__n_url_query;
        }
        case 6: {
          # 跳转到查询结束状态3
          goto s_n_llhttp__internal__n_span_end_stub_query_3;
        }
        default: {
          # 跳转到错误状态38
          goto s_n_llhttp__internal__n_error_38;
        }
      }
      # 不可达的代码，中止程序
      /* UNREACHABLE */;
      abort();
    }
    # 处理 URL 查询部分或片段部分的状态
    case s_n_llhttp__internal__n_url_query_or_fragment:
    # 检查是否已经到达输入的末尾，如果是则返回当前状态
    s_n_llhttp__internal__n_url_query_or_fragment: {
      if (p == endp) {
        return s_n_llhttp__internal__n_url_query_or_fragment;
      }
      # 根据当前字符进行不同的处理
      switch (*p) {
        # 如果是制表符，移动到下一个字符并跳转到错误状态1
        case 9: {
          p++;
          goto s_n_llhttp__internal__n_error_1;
        }
        # 如果是换行符，跳转到URL处理结束状态3
        case 10: {
          goto s_n_llhttp__internal__n_span_end_llhttp__on_url_3;
        }
        # 如果是换页符，移动到下一个字符并跳转到错误状态1
        case 12: {
          p++;
          goto s_n_llhttp__internal__n_error_1;
        }
        # 如果是回车符，跳转到URL处理结束状态4
        case 13: {
          goto s_n_llhttp__internal__n_span_end_llhttp__on_url_4;
        }
        # 如果是空格，跳转到URL处理结束状态5
        case ' ': {
          goto s_n_llhttp__internal__n_span_end_llhttp__on_url_5;
        }
        # 如果是井号，移动到下一个字符并跳转到URL片段状态
        case '#': {
          p++;
          goto s_n_llhttp__internal__n_url_fragment;
        }
        # 如果是问号，移动到下一个字符并跳转到URL查询状态
        case '?': {
          p++;
          goto s_n_llhttp__internal__n_url_query;
        }
        # 对于其他情况，跳转到错误状态39
        default: {
          goto s_n_llhttp__internal__n_error_39;
        }
      }
      # 不可达代码，中止程序
      /* UNREACHABLE */;
      abort();
    }
    # 处理URL路径的情况
    case s_n_llhttp__internal__n_url_path:
    # 定义静态数组 lookup_table，用于快速查找 URL 路径中的特定字符类型
    s_n_llhttp__internal__n_url_path: {
      static uint8_t lookup_table[] = {
        # 256 个元素的查找表，用于标识 URL 路径中每个字符的类型
        0, 0, 0, 0, 0, 0, 0, 0, 0, 1, 0, 0, 1, 0, 0, 0,
        # ... 省略部分元素 ...
      };
      # 如果指针 p 指向末尾，则返回当前状态 s_n_llhttp__internal__n_url_path
      if (p == endp) {
        return s_n_llhttp__internal__n_url_path;
      }
      # 根据 lookup_table 中的值进行不同的处理
      switch (lookup_table[(uint8_t) *p]) {
        # 如果字符类型为 1，则移动指针并跳转到错误状态 s_n_llhttp__internal__n_error_1
        case 1: {
          p++;
          goto s_n_llhttp__internal__n_error_1;
        }
        # 如果字符类型为 2，则移动指针并跳转回当前状态 s_n_llhttp__internal__n_url_path
        case 2: {
          p++;
          goto s_n_llhttp__internal__n_url_path;
        }
        # 其他情况，则跳转到 URL 查询或片段的状态 s_n_llhttp__internal__n_url_query_or_fragment
        default: {
          goto s_n_llhttp__internal__n_url_query_or_fragment;
        }
      }
      # 不可达的代码，中止程序
      /* UNREACHABLE */;
      abort();
    }
    # 其他状态的处理，类似上面的处理方式
    case s_n_llhttp__internal__n_span_start_stub_path_2:
    s_n_llhttp__internal__n_span_start_stub_path_2: {
      if (p == endp) {
        return s_n_llhttp__internal__n_span_start_stub_path_2;
      }
      p++;
      goto s_n_llhttp__internal__n_url_path;
      /* UNREACHABLE */;
      abort();
    }
    case s_n_llhttp__internal__n_span_start_stub_path:
    # ... 省略部分代码 ...
    # 如果当前指针已经到达结束位置，则返回当前状态
    if (p == endp) {
        return s_n_llhttp__internal__n_span_start_stub_path;
    }
    # 指针向前移动一位
    p++;
    # 跳转到状态 s_n_llhttp__internal__n_url_path
    goto s_n_llhttp__internal__n_url_path;
    # 不可达代码，表示不应该执行到这里，如果执行到这里则终止程序
    /* UNREACHABLE */;
    abort();
    # 当前状态为 s_n_llhttp__internal__n_span_start_stub_path_1
    case s_n_llhttp__internal__n_span_start_stub_path_1:
    s_n_llhttp__internal__n_span_start_stub_path_1: {
    # 如果当前指针已经到达结束位置，则返回当前状态
    if (p == endp) {
        return s_n_llhttp__internal__n_span_start_stub_path_1;
    }
    # 指针向前移动一位
    p++;
    # 跳转到状态 s_n_llhttp__internal__n_url_path
    goto s_n_llhttp__internal__n_url_path;
    # 不可达代码，表示不应该执行到这里，如果执行到这里则终止程序
    /* UNREACHABLE */;
    abort();
    # 当前状态为 s_n_llhttp__internal__n_url_server_with_at
    case s_n_llhttp__internal__n_url_server_with_at:
    s_n_llhttp__internal__n_url_server_with_at: {
      static uint8_t lookup_table[] = {  // 创建静态查找表
        0, 0, 0, 0, 0, 0, 0, 0, 0, 1, 2, 0, 1, 3, 0, 0,  // 初始化查找表的数值
        // ... （省略部分初始化数值）
        0, 0, 0, 0, 0, 0, 0, 0  // 初始化查找表的数值
      };
      if (p == endp) {  // 如果指针 p 等于 endp
        return s_n_llhttp__internal__n_url_server_with_at;  // 返回指定的状态
      }
      switch (lookup_table[(uint8_t) *p]) {  // 根据查找表的值进行切换
        case 1: {  // 如果查找表的值为 1
          p++;  // 指针 p 向前移动一位
          goto s_n_llhttp__internal__n_error_1;  // 跳转到指定的状态
        }
        case 2: {  // 如果查找表的值为 2
          goto s_n_llhttp__internal__n_span_end_llhttp__on_url_12;  // 跳转到指定的状态
        }
        case 3: {  // 如果查找表的值为 3
          goto s_n_llhttp__internal__n_span_end_llhttp__on_url_13;  // 跳转到指定的状态
        }
        // ... （省略部分 case）
        default: {  // 如果查找表的值为其他情况
          goto s_n_llhttp__internal__n_error_41;  // 跳转到指定的状态
        }
      }
      /* UNREACHABLE */;  // 不可到达的代码
      abort();  // 终止程序
    # 当前代码块结束
    }
    # 当前情况为内部 URL 服务器
    case s_n_llhttp__internal__n_url_server:
    s_n_llhttp__internal__n_url_server: {
      // 定义一个静态的查找表，用于根据输入字符的 ASCII 值进行快速查找
      static uint8_t lookup_table[] = {
        // ...（省略部分内容）
      };
      // 如果输入指针已经指向结束位置，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_url_server;
      }
      // 根据查找表中对应输入字符的值进行不同的处理
      switch (lookup_table[(uint8_t) *p]) {
        case 1: {
          // 如果查找表值为1，则移动输入指针并跳转到指定状态
          p++;
          goto s_n_llhttp__internal__n_error_1;
        }
        case 2: {
          // 如果查找表值为2，则跳转到指定状态
          goto s_n_llhttp__internal__n_span_end_llhttp__on_url;
        }
        // ...（省略其他case）
        default: {
          // 如果查找表值为其他值，则跳转到指定错误状态
          goto s_n_llhttp__internal__n_error_42;
        }
      }
      /* UNREACHABLE */;
      // 如果执行到这里，表示代码逻辑有误，终止程序
      abort();
    }
    # 当状态为 s_n_llhttp__internal__n_url_schema_delim_1 时执行以下代码块
    case s_n_llhttp__internal__n_url_schema_delim_1: {
      # 如果指针 p 指向末尾，则返回状态 s_n_llhttp__internal__n_url_schema_delim_1
      if (p == endp) {
        return s_n_llhttp__internal__n_url_schema_delim_1;
      }
      # 根据指针指向的字符执行不同的操作
      switch (*p) {
        # 如果字符为 '/'，则指针向后移动一位，跳转到状态 s_n_llhttp__internal__n_url_server
        case '/': {
          p++;
          goto s_n_llhttp__internal__n_url_server;
        }
        # 如果字符为其他情况，则跳转到状态 s_n_llhttp__internal__n_error_44
        default: {
          goto s_n_llhttp__internal__n_error_44;
        }
      }
      # 不可达的代码，中止程序
      /* UNREACHABLE */;
      abort();
    }
    # 当状态为 s_n_llhttp__internal__n_url_schema_delim 时执行以下代码块
    case s_n_llhttp__internal__n_url_schema_delim: {
      # 如果指针 p 指向末尾，则返回状态 s_n_llhttp__internal__n_url_schema_delim
      if (p == endp) {
        return s_n_llhttp__internal__n_url_schema_delim;
      }
      # 根据指针指向的字符执行不同的操作
      switch (*p) {
        # 如果字符为 9、10、12、13、' '，则执行不同的错误状态跳转
        case 9: {
          p++;
          goto s_n_llhttp__internal__n_error_1;
        }
        case 10: {
          p++;
          goto s_n_llhttp__internal__n_error_43;
        }
        case 12: {
          p++;
          goto s_n_llhttp__internal__n_error_1;
        }
        case 13: {
          p++;
          goto s_n_llhttp__internal__n_error_43;
        }
        case ' ': {
          p++;
          goto s_n_llhttp__internal__n_error_43;
        }
        # 如果字符为 '/'，则指针向后移动一位，跳转到状态 s_n_llhttp__internal__n_url_schema_delim_1
        case '/': {
          p++;
          goto s_n_llhttp__internal__n_url_schema_delim_1;
        }
        # 如果字符为其他情况，则跳转到状态 s_n_llhttp__internal__n_error_44
        default: {
          goto s_n_llhttp__internal__n_error_44;
        }
      }
      # 不可达的代码，中止程序
      /* UNREACHABLE */;
      abort();
    }
    # 当状态为 s_n_llhttp__internal__n_span_end_stub_schema 时执行以下代码块
    case s_n_llhttp__internal__n_span_end_stub_schema: {
      # 如果指针 p 指向末尾，则返回状态 s_n_llhttp__internal__n_span_end_stub_schema
      if (p == endp) {
        return s_n_llhttp__internal__n_span_end_stub_schema;
      }
      # 指针向后移动一位，跳转到状态 s_n_llhttp__internal__n_url_schema_delim
      p++;
      goto s_n_llhttp__internal__n_url_schema_delim;
      # 不可达的代码，中止程序
      /* UNREACHABLE */;
      abort();
    }
    # 当状态为 s_n_llhttp__internal__n_url_schema: 时执行以下代码块
    case s_n_llhttp__internal__n_url_schema:
    # 定义静态数组，用于查找 URL schema 的类型
    s_n_llhttp__internal__n_url_schema: {
      static uint8_t lookup_table[] = {
        0, 0, 0, 0, 0, 0, 0, 0, 0, 1, 2, 0, 1, 2, 0, 0,
        0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0,
        2, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0,
        0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 3, 0, 0, 0, 0, 0,
        0, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4,
        4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 0, 0, 0, 0, 0,
        0, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4,
        4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 0, 0, 0, 0
    # 定义静态数组 lookup_table
    s_n_llhttp__internal__n_url_start: {
      static uint8_t lookup_table[] = {
        # 数组内容
        0, 0, 0, 0, 0, 0, 0, 0, 0, 1, 2, 0, 1, 2, 0, 0,
        # ... 省略部分内容 ...
      };
      # 如果指针 p 等于结束指针 endp，则返回 s_n_llhttp__internal__n_url_start
      if (p == endp) {
        return s_n_llhttp__internal__n_url_start;
      }
      # 根据 lookup_table 中 p 指向的值进行不同的处理
      switch (lookup_table[(uint8_t) *p]) {
        # 如果值为 1，则执行下一行代码并跳转到标签 s_n_llhttp__internal__n_error_1
        case 1: {
          p++;
          goto s_n_llhttp__internal__n_error_1;
        }
        # 如果值为 2，则执行下一行代码并跳转到标签 s_n_llhttp__internal__n_error_43
        case 2: {
          p++;
          goto s_n_llhttp__internal__n_error_43;
        }
        # 如果值为 3，则跳转到标签 s_n_llhttp__internal__n_span_start_stub_path_2
        case 3: {
          goto s_n_llhttp__internal__n_span_start_stub_path_2;
        }
        # 如果值为 4，则跳转到标签 s_n_llhttp__internal__n_url_schema
        case 4: {
          goto s_n_llhttp__internal__n_url_schema;
        }
        # 如果值不在 1-4 范围内，则跳转到标签 s_n_llhttp__internal__n_error_46
        default: {
          goto s_n_llhttp__internal__n_error_46;
        }
      }
      # 不可达的代码，中止程序
      /* UNREACHABLE */;
      abort();
    }
    # 如果当前状态为 s_n_llhttp__internal__n_span_start_llhttp__on_url_1，则执行以下代码
    case s_n_llhttp__internal__n_span_start_llhttp__on_url_1:
    s_n_llhttp__internal__n_span_start_llhttp__on_url_1: {
      # 如果指针 p 等于结束指针 endp，则返回 s_n_llhttp__internal__n_span_start_llhttp__on_url_1
      if (p == endp) {
        return s_n_llhttp__internal__n_span_start_llhttp__on_url_1;
      }
      # 设置状态的 _span_pos0 和 _span_cb0 属性，并跳转到标签 s_n_llhttp__internal__n_url_start
      state->_span_pos0 = (void*) p;
      state->_span_cb0 = llhttp__on_url;
      goto s_n_llhttp__internal__n_url_start;
      # 不可达的代码，中止程序
      /* UNREACHABLE */;
      abort();
    }
    # 当 URL 处于正常状态时的处理
    case s_n_llhttp__internal__n_url_entry_normal:
    s_n_llhttp__internal__n_url_entry_normal: {
      # 如果指针已经到达末尾，则返回正常状态
      if (p == endp) {
        return s_n_llhttp__internal__n_url_entry_normal;
      }
      # 根据当前字符进行不同的处理
      switch (*p) {
        # 如果是制表符，则跳转到错误状态
        case 9: {
          p++;
          goto s_n_llhttp__internal__n_error_1;
        }
        # 如果是换页符，则跳转到错误状态
        case 12: {
          p++;
          goto s_n_llhttp__internal__n_error_1;
        }
        # 否则跳转到处理 URL 开始的状态
        default: {
          goto s_n_llhttp__internal__n_span_start_llhttp__on_url_1;
        }
      }
      # 不可达的代码，中止程序
      /* UNREACHABLE */;
      abort();
    }
    # 当 URL 处于开始处理状态时的处理
    case s_n_llhttp__internal__n_span_start_llhttp__on_url:
    s_n_llhttp__internal__n_span_start_llhttp__on_url: {
      # 如果指针已经到达末尾，则返回开始处理状态
      if (p == endp) {
        return s_n_llhttp__internal__n_span_start_llhttp__on_url;
      }
      # 记录当前位置和回调函数，然后跳转到处理 URL 的状态
      state->_span_pos0 = (void*) p;
      state->_span_cb0 = llhttp__on_url;
      goto s_n_llhttp__internal__n_url_server;
      # 不可达的代码，中止程序
      /* UNREACHABLE */;
      abort();
    }
    # 当 URL 处于连接状态时的处理
    case s_n_llhttp__internal__n_url_entry_connect:
    s_n_llhttp__internal__n_url_entry_connect: {
      # 如果指针已经到达末尾，则返回连接状态
      if (p == endp) {
        return s_n_llhttp__internal__n_url_entry_connect;
      }
      # 根据当前字符进行不同的处理
      switch (*p) {
        # 如果是制表符，则跳转到错误状态
        case 9: {
          p++;
          goto s_n_llhttp__internal__n_error_1;
        }
        # 如果是换页符，则跳转到错误状态
        case 12: {
          p++;
          goto s_n_llhttp__internal__n_error_1;
        }
        # 否则跳转到处理 URL 开始的状态
        default: {
          goto s_n_llhttp__internal__n_span_start_llhttp__on_url;
        }
      }
      # 不可达的代码，中止程序
      /* UNREACHABLE */;
      abort();
    }
    # 当请求 URL 前有空格时的处理
    case s_n_llhttp__internal__n_req_spaces_before_url:
    s_n_llhttp__internal__n_req_spaces_before_url: {
      # 如果指针已经到达末尾，则返回请求 URL 前有空格状态
      if (p == endp) {
        return s_n_llhttp__internal__n_req_spaces_before_url;
      }
      # 根据当前字符进行不同的处理
      switch (*p) {
        # 如果是空格，则继续处理请求 URL 前有空格状态
        case ' ': {
          p++;
          goto s_n_llhttp__internal__n_req_spaces_before_url;
        }
        # 否则跳转到调用相等方法的状态
        default: {
          goto s_n_llhttp__internal__n_invoke_is_equal_method;
        }
      }
      # 不可达的代码，中止程序
      /* UNREACHABLE */;
      abort();
    }
    # 切换到处理请求中第一个空格之前的状态
    case s_n_llhttp__internal__n_req_first_space_before_url:
    s_n_llhttp__internal__n_req_first_space_before_url: {
      # 如果指针已经到达末尾，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_req_first_space_before_url;
      }
      # 根据当前字符进行判断
      switch (*p) {
        # 如果是空格，则指针向后移动一位，进入处理 URL 前的空格状态
        case ' ': {
          p++;
          goto s_n_llhttp__internal__n_req_spaces_before_url;
        }
        # 如果不是空格，则进入错误状态
        default: {
          goto s_n_llhttp__internal__n_error_47;
        }
      }
      /* UNREACHABLE */;
      abort();
    }
    # 切换到处理请求中第二个字符的状态
    case s_n_llhttp__internal__n_start_req_2:
    s_n_llhttp__internal__n_start_req_2: {
      # 如果指针已经到达末尾，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_start_req_2;
      }
      # 根据当前字符进行判断
      switch (*p) {
        # 如果是 'L'，则指针向后移动一位，设置匹配值为 19，进入存储方法的调用状态
        case 'L': {
          p++;
          match = 19;
          goto s_n_llhttp__internal__n_invoke_store_method_1;
        }
        # 如果不是 'L'，则进入错误状态
        default: {
          goto s_n_llhttp__internal__n_error_56;
        }
      }
      /* UNREACHABLE */;
      abort();
    }
    # 切换到处理请求中第三个字符的状态
    case s_n_llhttp__internal__n_start_req_3:
    s_n_llhttp__internal__n_start_req_3: {
      llparse_match_t match_seq;

      # 如果指针已经到达末尾，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_start_req_3;
      }
      # 调用匹配序列函数，获取匹配结果
      match_seq = llparse__match_sequence_id(state, p, endp, llparse_blob18, 6);
      p = match_seq.current;
      # 根据匹配结果进行处理
      switch (match_seq.status) {
        # 如果匹配完成，则指针向后移动一位，设置匹配值为 36，进入存储方法的调用状态
        case kMatchComplete: {
          p++;
          match = 36;
          goto s_n_llhttp__internal__n_invoke_store_method_1;
        }
        # 如果匹配暂停，则返回当前状态
        case kMatchPause: {
          return s_n_llhttp__internal__n_start_req_3;
        }
        # 如果匹配不成功，则进入错误状态
        case kMatchMismatch: {
          goto s_n_llhttp__internal__n_error_56;
        }
      }
      /* UNREACHABLE */;
      abort();
    }
    # 切换到处理请求中第一个字符的状态
    case s_n_llhttp__internal__n_start_req_1:
    # 定义状态 s_n_llhttp__internal__n_start_req_1，处理请求开始的状态
    s_n_llhttp__internal__n_start_req_1: {
      # 如果指针 p 等于结束指针 endp，则返回状态 s_n_llhttp__internal__n_start_req_1
      if (p == endp) {
        return s_n_llhttp__internal__n_start_req_1;
      }
      # 根据指针 p 指向的字符进行判断
      switch (*p) {
        # 如果字符为 'C'，则指针 p 向后移动一位，跳转到状态 s_n_llhttp__internal__n_start_req_2
        case 'C': {
          p++;
          goto s_n_llhttp__internal__n_start_req_2;
        }
        # 如果字符为 'N'，则指针 p 向后移动一位，跳转到状态 s_n_llhttp__internal__n_start_req_3
        case 'N': {
          p++;
          goto s_n_llhttp__internal__n_start_req_3;
        }
        # 如果字符不是 'C' 或 'N'，则跳转到状态 s_n_llhttp__internal__n_error_56
        default: {
          goto s_n_llhttp__internal__n_error_56;
        }
      }
      # 不可达的代码，中止程序
      /* UNREACHABLE */;
      abort();
    }
    # 定义状态 s_n_llhttp__internal__n_start_req_4
    case s_n_llhttp__internal__n_start_req_4:
    s_n_llhttp__internal__n_start_req_4: {
      llparse_match_t match_seq;

      # 如果指针 p 等于结束指针 endp，则返回状态 s_n_llhttp__internal__n_start_req_4
      if (p == endp) {
        return s_n_llhttp__internal__n_start_req_4;
      }
      # 调用 llparse__match_sequence_id 函数，匹配指针 p 指向的内容和 llparse_blob19 中的内容，长度为 3
      match_seq = llparse__match_sequence_id(state, p, endp, llparse_blob19, 3);
      p = match_seq.current;
      # 根据匹配的状态进行判断
      switch (match_seq.status) {
        # 如果匹配完成，则指针 p 向后移动一位，设置 match 为 16，跳转到状态 s_n_llhttp__internal__n_invoke_store_method_1
        case kMatchComplete: {
          p++;
          match = 16;
          goto s_n_llhttp__internal__n_invoke_store_method_1;
        }
        # 如果匹配暂停，则返回状态 s_n_llhttp__internal__n_start_req_4
        case kMatchPause: {
          return s_n_llhttp__internal__n_start_req_4;
        }
        # 如果匹配不成功，则跳转到状态 s_n_llhttp__internal__n_error_56
        case kMatchMismatch: {
          goto s_n_llhttp__internal__n_error_56;
        }
      }
      # 不可达的代码，中止程序
      /* UNREACHABLE */;
      abort();
    }
    # 定义状态 s_n_llhttp__internal__n_start_req_6
    case s_n_llhttp__internal__n_start_req_6:
    s_n_llhttp__internal__n_start_req_6: {
      llparse_match_t match_seq;

      # 如果指针 p 等于结束指针 endp，则返回状态 s_n_llhttp__internal__n_start_req_6
      if (p == endp) {
        return s_n_llhttp__internal__n_start_req_6;
      }
      # 调用 llparse__match_sequence_id 函数，匹配指针 p 指向的内容和 llparse_blob20 中的内容，长度为 6
      match_seq = llparse__match_sequence_id(state, p, endp, llparse_blob20, 6);
      p = match_seq.current;
      # 根据匹配的状态进行判断
      switch (match_seq.status) {
        # 如果匹配完成，则指针 p 向后移动一位，设置 match 为 22，跳转到状态 s_n_llhttp__internal__n_invoke_store_method_1
        case kMatchComplete: {
          p++;
          match = 22;
          goto s_n_llhttp__internal__n_invoke_store_method_1;
        }
        # 如果匹配暂停，则返回状态 s_n_llhttp__internal__n_start_req_6
        case kMatchPause: {
          return s_n_llhttp__internal__n_start_req_6;
        }
        # 如果匹配不成功，则跳转到状态 s_n_llhttp__internal__n_error_56
        case kMatchMismatch: {
          goto s_n_llhttp__internal__n_error_56;
        }
      }
      # 不可达的代码，中止程序
      /* UNREACHABLE */;
      abort();
    }
    # 定义状态 s_n_llhttp__internal__n_start_req_8
    # 定义状态 s_n_llhttp__internal__n_start_req_8，处理请求起始部分的状态机
    s_n_llhttp__internal__n_start_req_8: {
      # 定义匹配序列的结果
      llparse_match_t match_seq;

      # 如果指针已经到达结束位置，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_start_req_8;
      }
      # 调用 llparse__match_sequence_id 函数，匹配指定长度的序列
      match_seq = llparse__match_sequence_id(state, p, endp, llparse_blob21, 4);
      # 更新指针位置
      p = match_seq.current;
      # 根据匹配结果的状态进行不同的处理
      switch (match_seq.status) {
        # 如果匹配完成
        case kMatchComplete: {
          # 更新指针位置
          p++;
          # 设置匹配结果
          match = 5;
          # 跳转到状态 s_n_llhttp__internal__n_invoke_store_method_1
          goto s_n_llhttp__internal__n_invoke_store_method_1;
        }
        # 如果匹配暂停
        case kMatchPause: {
          # 返回当前状态
          return s_n_llhttp__internal__n_start_req_8;
        }
        # 如果匹配不成功
        case kMatchMismatch: {
          # 跳转到状态 s_n_llhttp__internal__n_error_56
          goto s_n_llhttp__internal__n_error_56;
        }
      }
      /* UNREACHABLE */;
      # 终止程序执行
      abort();
    }
    # 定义状态 s_n_llhttp__internal__n_start_req_9
    case s_n_llhttp__internal__n_start_req_9:
    s_n_llhttp__internal__n_start_req_9: {
      # 如果指针已经到达结束位置，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_start_req_9;
      }
      # 根据当前字符进行不同的处理
      switch (*p) {
        # 如果当前字符为 'Y'
        case 'Y': {
          # 更新指针位置
          p++;
          # 设置匹配结果
          match = 8;
          # 跳转到状态 s_n_llhttp__internal__n_invoke_store_method_1
          goto s_n_llhttp__internal__n_invoke_store_method_1;
        }
        # 如果当前字符不为 'Y'
        default: {
          # 跳转到状态 s_n_llhttp__internal__n_error_56
          goto s_n_llhttp__internal__n_error_56;
        }
      }
      /* UNREACHABLE */;
      # 终止程序执行
      abort();
    }
    # 定义状态 s_n_llhttp__internal__n_start_req_7
    case s_n_llhttp__internal__n_start_req_7:
    s_n_llhttp__internal__n_start_req_7: {
      # 如果指针已经到达结束位置，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_start_req_7;
      }
      # 根据当前字符进行不同的处理
      switch (*p) {
        # 如果当前字符为 'N'
        case 'N': {
          # 更新指针位置
          p++;
          # 跳转到状态 s_n_llhttp__internal__n_start_req_8
          goto s_n_llhttp__internal__n_start_req_8;
        }
        # 如果当前字符为 'P'
        case 'P': {
          # 更新指针位置
          p++;
          # 跳转到状态 s_n_llhttp__internal__n_start_req_9
          goto s_n_llhttp__internal__n_start_req_9;
        }
        # 如果当前字符既不是 'N' 也不是 'P'
        default: {
          # 跳转到状态 s_n_llhttp__internal__n_error_56
          goto s_n_llhttp__internal__n_error_56;
        }
      }
      /* UNREACHABLE */;
      # 终止程序执行
      abort();
    }
    # 定义状态 s_n_llhttp__internal__n_start_req_5:
    # 如果指针已经指向结束位置，则返回当前状态
    s_n_llhttp__internal__n_start_req_5: {
      if (p == endp) {
        return s_n_llhttp__internal__n_start_req_5;
      }
      # 根据当前字符进行不同的处理
      switch (*p) {
        # 如果当前字符是 'H'，则指针向后移动一位，跳转到状态 s_n_llhttp__internal__n_start_req_6
        case 'H': {
          p++;
          goto s_n_llhttp__internal__n_start_req_6;
        }
        # 如果当前字符是 'O'，则指针向后移动一位，跳转到状态 s_n_llhttp__internal__n_start_req_7
        case 'O': {
          p++;
          goto s_n_llhttp__internal__n_start_req_7;
        }
        # 如果当前字符不是 'H' 或 'O'，则跳转到状态 s_n_llhttp__internal__n_error_56
        default: {
          goto s_n_llhttp__internal__n_error_56;
        }
      }
      # 不可达的代码，中止程序
      /* UNREACHABLE */;
      abort();
    }
    # 如果当前状态是 s_n_llhttp__internal__n_start_req_12
    case s_n_llhttp__internal__n_start_req_12:
    s_n_llhttp__internal__n_start_req_12: {
      llparse_match_t match_seq;

      # 如果指针已经指向结束位置，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_start_req_12;
      }
      # 调用 llparse__match_sequence_id 函数进行匹配
      match_seq = llparse__match_sequence_id(state, p, endp, llparse_blob22, 3);
      p = match_seq.current;
      # 根据匹配的状态进行不同的处理
      switch (match_seq.status) {
        # 如果匹配完成，指针向后移动一位，设置 match 为 0，跳转到状态 s_n_llhttp__internal__n_invoke_store_method_1
        case kMatchComplete: {
          p++;
          match = 0;
          goto s_n_llhttp__internal__n_invoke_store_method_1;
        }
        # 如果匹配暂停，则返回当前状态
        case kMatchPause: {
          return s_n_llhttp__internal__n_start_req_12;
        }
        # 如果匹配不成功，则跳转到状态 s_n_llhttp__internal__n_error_56
        case kMatchMismatch: {
          goto s_n_llhttp__internal__n_error_56;
        }
      }
      # 不可达的代码，中止程序
      /* UNREACHABLE */;
      abort();
    }
    # 如果当前状态是 s_n_llhttp__internal__n_start_req_13
    case s_n_llhttp__internal__n_start_req_13:
    s_n_llhttp__internal__n_start_req_13: {
      llparse_match_t match_seq;

      # 如果指针已经指向结束位置，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_start_req_13;
      }
      # 调用 llparse__match_sequence_id 函数进行匹配
      match_seq = llparse__match_sequence_id(state, p, endp, llparse_blob23, 5);
      p = match_seq.current;
      # 根据匹配的状态进行不同的处理
      switch (match_seq.status) {
        # 如果匹配完成，指针向后移动一位，设置 match 为 35，跳转到状态 s_n_llhttp__internal__n_invoke_store_method_1
        case kMatchComplete: {
          p++;
          match = 35;
          goto s_n_llhttp__internal__n_invoke_store_method_1;
        }
        # 如果匹配暂停，则返回当前状态
        case kMatchPause: {
          return s_n_llhttp__internal__n_start_req_13;
        }
        # 如果匹配不成功，则跳转到状态 s_n_llhttp__internal__n_error_56
        case kMatchMismatch: {
          goto s_n_llhttp__internal__n_error_56;
        }
      }
      # 不可达的代码，中止程序
      /* UNREACHABLE */;
      abort();
    }
    # 如果当前状态是 s_n_llhttp__internal__n_start_req_11
    case s_n_llhttp__internal__n_start_req_11:
    # 定义状态 s_n_llhttp__internal__n_start_req_11，处理请求行的第一个字符
    s_n_llhttp__internal__n_start_req_11: {
      # 如果指针已经指向结束位置，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_start_req_11;
      }
      # 根据当前字符进行判断
      switch (*p) {
        # 如果当前字符为 'L'，则跳转到状态 s_n_llhttp__internal__n_start_req_12
        case 'L': {
          p++;
          goto s_n_llhttp__internal__n_start_req_12;
        }
        # 如果当前字符为 'S'，则跳转到状态 s_n_llhttp__internal__n_start_req_13
        case 'S': {
          p++;
          goto s_n_llhttp__internal__n_start_req_13;
        }
        # 如果当前字符不是 'L' 或 'S'，则跳转到状态 s_n_llhttp__internal__n_error_56
        default: {
          goto s_n_llhttp__internal__n_error_56;
        }
      }
      # 不可达的代码，中止程序
      /* UNREACHABLE */;
      abort();
    }
    # 定义状态 s_n_llhttp__internal__n_start_req_10，处理请求行的第一个字符为 'E'
    case s_n_llhttp__internal__n_start_req_10:
    s_n_llhttp__internal__n_start_req_10: {
      # 如果指针已经指向结束位置，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_start_req_10;
      }
      # 根据当前字符进行判断
      switch (*p) {
        # 如果当前字符为 'E'，则跳转到状态 s_n_llhttp__internal__n_start_req_11
        case 'E': {
          p++;
          goto s_n_llhttp__internal__n_start_req_11;
        }
        # 如果当前字符不是 'E'，则跳转到状态 s_n_llhttp__internal__n_error_56
        default: {
          goto s_n_llhttp__internal__n_error_56;
        }
      }
      # 不可达的代码，中止程序
      /* UNREACHABLE */;
      abort();
    }
    # 定义状态 s_n_llhttp__internal__n_start_req_14，处理请求行的第一个字符为 'E' 之后的字符
    case s_n_llhttp__internal__n_start_req_14:
    s_n_llhttp__internal__n_start_req_14: {
      # 定义匹配结果的结构体
      llparse_match_t match_seq;
      # 如果指针已经指向结束位置，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_start_req_14;
      }
      # 调用 llparse__match_sequence_id 函数进行匹配
      match_seq = llparse__match_sequence_id(state, p, endp, llparse_blob24, 4);
      p = match_seq.current;
      # 根据匹配结果进行判断
      switch (match_seq.status) {
        # 如果匹配完成，则继续处理下一个字符，并跳转到状态 s_n_llhttp__internal__n_invoke_store_method_1
        case kMatchComplete: {
          p++;
          match = 45;
          goto s_n_llhttp__internal__n_invoke_store_method_1;
        }
        # 如果匹配暂停，则返回当前状态
        case kMatchPause: {
          return s_n_llhttp__internal__n_start_req_14;
        }
        # 如果匹配不成功，则跳转到状态 s_n_llhttp__internal__n_error_56
        case kMatchMismatch: {
          goto s_n_llhttp__internal__n_error_56;
        }
      }
      # 不可达的代码，中止程序
      /* UNREACHABLE */;
      abort();
    }
    # 定义状态 s_n_llhttp__internal__n_start_req_17:
    # 定义状态 s_n_llhttp__internal__n_start_req_17，用于解析 HTTP 请求的起始部分
    s_n_llhttp__internal__n_start_req_17: {
      # 定义匹配序列对象
      llparse_match_t match_seq;

      # 如果指针已经到达结束位置，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_start_req_17;
      }
      # 调用 llparse__match_sequence_id 函数，匹配指定长度的数据
      match_seq = llparse__match_sequence_id(state, p, endp, llparse_blob26, 9);
      # 更新指针位置
      p = match_seq.current;
      # 根据匹配状态进行不同的处理
      switch (match_seq.status) {
        # 如果匹配完成
        case kMatchComplete: {
          # 更新指针位置
          p++;
          # 设置匹配值
          match = 41;
          # 跳转到状态 s_n_llhttp__internal__n_invoke_store_method_1
          goto s_n_llhttp__internal__n_invoke_store_method_1;
        }
        # 如果匹配暂停
        case kMatchPause: {
          # 返回当前状态
          return s_n_llhttp__internal__n_start_req_17;
        }
        # 如果匹配不成功
        case kMatchMismatch: {
          # 跳转到状态 s_n_llhttp__internal__n_error_56
          goto s_n_llhttp__internal__n_error_56;
        }
      }
      /* UNREACHABLE */;
      # 终止程序
      abort();
    }
    # 定义状态 s_n_llhttp__internal__n_start_req_16
    case s_n_llhttp__internal__n_start_req_16:
    s_n_llhttp__internal__n_start_req_16: {
      # 如果指针已经到达结束位置，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_start_req_16;
      }
      # 根据当前指针指向的值进行不同的处理
      switch (*p) {
        # 如果当前值为 '_'
        case '_': {
          # 更新指针位置
          p++;
          # 跳转到状态 s_n_llhttp__internal__n_start_req_17
          goto s_n_llhttp__internal__n_start_req_17;
        }
        # 如果当前值不为 '_'
        default: {
          # 设置匹配值
          match = 1;
          # 跳转到状态 s_n_llhttp__internal__n_invoke_store_method_1
          goto s_n_llhttp__internal__n_invoke_store_method_1;
        }
      }
      /* UNREACHABLE */;
      # 终止程序
      abort();
    }
    # 定义状态 s_n_llhttp__internal__n_start_req_15
    case s_n_llhttp__internal__n_start_req_15:
    s_n_llhttp__internal__n_start_req_15: {
      # 定义匹配序列对象
      llparse_match_t match_seq;

      # 如果指针已经到达结束位置，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_start_req_15;
      }
      # 调用 llparse__match_sequence_id 函数，匹配指定长度的数据
      match_seq = llparse__match_sequence_id(state, p, endp, llparse_blob25, 2);
      # 更新指针位置
      p = match_seq.current;
      # 根据匹配状态进行不同的处理
      switch (match_seq.status) {
        # 如果匹配完成
        case kMatchComplete: {
          # 更新指针位置
          p++;
          # 跳转到状态 s_n_llhttp__internal__n_start_req_16
          goto s_n_llhttp__internal__n_start_req_16;
        }
        # 如果匹配暂停
        case kMatchPause: {
          # 返回当前状态
          return s_n_llhttp__internal__n_start_req_15;
        }
        # 如果匹配不成功
        case kMatchMismatch: {
          # 跳转到状态 s_n_llhttp__internal__n_error_56
          goto s_n_llhttp__internal__n_error_56;
        }
      }
      /* UNREACHABLE */;
      # 终止程序
      abort();
    }
    # 定义状态 s_n_llhttp__internal__n_start_req_18
    # 定义状态 s_n_llhttp__internal__n_start_req_18，处理请求开始的状态
    s_n_llhttp__internal__n_start_req_18: {
      # 定义匹配序列的变量
      llparse_match_t match_seq;

      # 如果指针已经到达结束位置，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_start_req_18;
      }
      # 调用 llparse__match_sequence_id 函数，匹配指定的序列
      match_seq = llparse__match_sequence_id(state, p, endp, llparse_blob27, 3);
      # 更新指针位置
      p = match_seq.current;
      # 根据匹配的状态进行处理
      switch (match_seq.status) {
        # 如果匹配完成
        case kMatchComplete: {
          # 指针向前移动一位
          p++;
          # 设置匹配值为 2
          match = 2;
          # 跳转到状态 s_n_llhttp__internal__n_invoke_store_method_1
          goto s_n_llhttp__internal__n_invoke_store_method_1;
        }
        # 如果匹配暂停
        case kMatchPause: {
          # 返回当前状态
          return s_n_llhttp__internal__n_start_req_18;
        }
        # 如果匹配不成功
        case kMatchMismatch: {
          # 跳转到状态 s_n_llhttp__internal__n_error_56
          goto s_n_llhttp__internal__n_error_56;
        }
      }
      # 不可达的代码，中断程序执行
      /* UNREACHABLE */;
      abort();
    }
    # 定义状态 s_n_llhttp__internal__n_start_req_20
    case s_n_llhttp__internal__n_start_req_20:
    s_n_llhttp__internal__n_start_req_20: {
      # 定义匹配序列的变量
      llparse_match_t match_seq;

      # 如果指针已经到达结束位置，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_start_req_20;
      }
      # 调用 llparse__match_sequence_id 函数，匹配指定的序列
      match_seq = llparse__match_sequence_id(state, p, endp, llparse_blob28, 2);
      # 更新指针位置
      p = match_seq.current;
      # 根据匹配的状态进行处理
      switch (match_seq.status) {
        # 如果匹配完成
        case kMatchComplete: {
          # 指针向前移动一位
          p++;
          # 设置匹配值为 31
          match = 31;
          # 跳转到状态 s_n_llhttp__internal__n_invoke_store_method_1
          goto s_n_llhttp__internal__n_invoke_store_method_1;
        }
        # 如果匹配暂停
        case kMatchPause: {
          # 返回当前状态
          return s_n_llhttp__internal__n_start_req_20;
        }
        # 如果匹配不成功
        case kMatchMismatch: {
          # 跳转到状态 s_n_llhttp__internal__n_error_56
          goto s_n_llhttp__internal__n_error_56;
        }
      }
      # 不可达的代码，中断程序执行
      /* UNREACHABLE */;
      abort();
    }
    # 定义状态 s_n_llhttp__internal__n_start_req_21
    case s_n_llhttp__internal__n_start_req_21:
    # 定义状态 s_n_llhttp__internal__n_start_req_21，表示解析 HTTP 请求的起始状态
    s_n_llhttp__internal__n_start_req_21: {
      # 定义匹配结果的结构体
      llparse_match_t match_seq;

      # 如果指针已经到达结束位置，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_start_req_21;
      }
      # 调用 llparse__match_sequence_id 函数进行匹配
      match_seq = llparse__match_sequence_id(state, p, endp, llparse_blob29, 2);
      # 更新指针位置
      p = match_seq.current;
      # 根据匹配结果的状态进行不同的处理
      switch (match_seq.status) {
        # 如果匹配完成
        case kMatchComplete: {
          # 指针后移一位
          p++;
          # 设置匹配结果为 9
          match = 9;
          # 跳转到状态 s_n_llhttp__internal__n_invoke_store_method_1
          goto s_n_llhttp__internal__n_invoke_store_method_1;
        }
        # 如果匹配暂停
        case kMatchPause: {
          # 返回当前状态
          return s_n_llhttp__internal__n_start_req_21;
        }
        # 如果匹配不成功
        case kMatchMismatch: {
          # 跳转到状态 s_n_llhttp__internal__n_error_56
          goto s_n_llhttp__internal__n_error_56;
        }
      }
      /* UNREACHABLE */;
      # 终止程序
      abort();
    }
    # 定义状态 s_n_llhttp__internal__n_start_req_19
    case s_n_llhttp__internal__n_start_req_19:
    s_n_llhttp__internal__n_start_req_19: {
      # 如果指针已经到达结束位置，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_start_req_19;
      }
      # 根据当前指针位置的值进行不同的处理
      switch (*p) {
        # 如果当前字符为 'I'
        case 'I': {
          # 指针后移一位
          p++;
          # 跳转到状态 s_n_llhttp__internal__n_start_req_20
          goto s_n_llhttp__internal__n_start_req_20;
        }
        # 如果当前字符为 'O'
        case 'O': {
          # 指针后移一位
          p++;
          # 跳转到状态 s_n_llhttp__internal__n_start_req_21
          goto s_n_llhttp__internal__n_start_req_21;
        }
        # 如果当前字符不是 'I' 也不是 'O'
        default: {
          # 跳转到状态 s_n_llhttp__internal__n_error_56
          goto s_n_llhttp__internal__n_error_56;
        }
      }
      /* UNREACHABLE */;
      # 终止程序
      abort();
    }
    # 定义状态 s_n_llhttp__internal__n_start_req_23
    case s_n_llhttp__internal__n_start_req_23:
    s_n_llhttp__internal__n_start_req_23: {
      # 定义匹配结果的结构体
      llparse_match_t match_seq;

      # 如果指针已经到达结束位置，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_start_req_23;
      }
      # 调用 llparse__match_sequence_id 函数进行匹配
      match_seq = llparse__match_sequence_id(state, p, endp, llparse_blob30, 6);
      # 更新指针位置
      p = match_seq.current;
      # 根据匹配结果的状态进行不同的处理
      switch (match_seq.status) {
        # 如果匹配完成
        case kMatchComplete: {
          # 指针后移一位
          p++;
          # 设置匹配结果为 24
          match = 24;
          # 跳转到状态 s_n_llhttp__internal__n_invoke_store_method_1
          goto s_n_llhttp__internal__n_invoke_store_method_1;
        }
        # 如果匹配暂停
        case kMatchPause: {
          # 返回当前状态
          return s_n_llhttp__internal__n_start_req_23;
        }
        # 如果匹配不成功
        case kMatchMismatch: {
          # 跳转到状态 s_n_llhttp__internal__n_error_56
          goto s_n_llhttp__internal__n_error_56;
        }
      }
      /* UNREACHABLE */;
      # 终止程序
      abort();
    }
    # 定义状态 s_n_llhttp__internal__n_start_req_24
    # 定义状态 s_n_llhttp__internal__n_start_req_24，开始解析请求的状态
    s_n_llhttp__internal__n_start_req_24: {
      # 定义匹配序列的变量
      llparse_match_t match_seq;

      # 如果指针已经到达结束位置，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_start_req_24;
      }
      # 调用 llparse__match_sequence_id 函数，匹配指定长度的数据
      match_seq = llparse__match_sequence_id(state, p, endp, llparse_blob31, 3);
      # 更新指针位置
      p = match_seq.current;
      # 根据匹配的状态进行处理
      switch (match_seq.status) {
        # 如果匹配完成
        case kMatchComplete: {
          # 指针后移一位
          p++;
          # 设置匹配值
          match = 23;
          # 跳转到指定状态
          goto s_n_llhttp__internal__n_invoke_store_method_1;
        }
        # 如果匹配暂停
        case kMatchPause: {
          # 返回当前状态
          return s_n_llhttp__internal__n_start_req_24;
        }
        # 如果匹配不成功
        case kMatchMismatch: {
          # 跳转到错误状态
          goto s_n_llhttp__internal__n_error_56;
        }
      }
      /* UNREACHABLE */;
      # 终止程序
      abort();
    }
    # 定义状态 s_n_llhttp__internal__n_start_req_26
    case s_n_llhttp__internal__n_start_req_26:
    s_n_llhttp__internal__n_start_req_26: {
      # 定义匹配序列的变量
      llparse_match_t match_seq;

      # 如果指针已经到达结束位置，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_start_req_26;
      }
      # 调用 llparse__match_sequence_id 函数，匹配指定长度的数据
      match_seq = llparse__match_sequence_id(state, p, endp, llparse_blob32, 7);
      # 更新指针位置
      p = match_seq.current;
      # 根据匹配的状态进行处理
      switch (match_seq.status) {
        # 如果匹配完成
        case kMatchComplete: {
          # 指针后移一位
          p++;
          # 设置匹配值
          match = 21;
          # 跳转到指定状态
          goto s_n_llhttp__internal__n_invoke_store_method_1;
        }
        # 如果匹配暂停
        case kMatchPause: {
          # 返回当前状态
          return s_n_llhttp__internal__n_start_req_26;
        }
        # 如果匹配不成功
        case kMatchMismatch: {
          # 跳转到错误状态
          goto s_n_llhttp__internal__n_error_56;
        }
      }
      /* UNREACHABLE */;
      # 终止程序
      abort();
    }
    # 定义状态 s_n_llhttp__internal__n_start_req_28
    case s_n_llhttp__internal__n_start_req_28:
    # 定义状态 s_n_llhttp__internal__n_start_req_28，表示解析 HTTP 请求的起始状态
    s_n_llhttp__internal__n_start_req_28: {
      # 定义变量 match_seq，用于存储匹配结果
      llparse_match_t match_seq;

      # 如果指针 p 已经指向输入结束位置，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_start_req_28;
      }
      # 调用 llparse__match_sequence_id 函数，匹配输入数据和指定的序列 llparse_blob33，长度为 6
      match_seq = llparse__match_sequence_id(state, p, endp, llparse_blob33, 6);
      # 更新指针 p 的位置
      p = match_seq.current;
      # 根据匹配结果的状态进行不同的处理
      switch (match_seq.status) {
        # 如果匹配完成
        case kMatchComplete: {
          # 更新指针 p 的位置
          p++;
          # 更新变量 match 的值
          match = 30;
          # 跳转到状态 s_n_llhttp__internal__n_invoke_store_method_1
          goto s_n_llhttp__internal__n_invoke_store_method_1;
        }
        # 如果匹配暂停
        case kMatchPause: {
          # 返回当前状态
          return s_n_llhttp__internal__n_start_req_28;
        }
        # 如果匹配不成功
        case kMatchMismatch: {
          # 跳转到状态 s_n_llhttp__internal__n_error_56
          goto s_n_llhttp__internal__n_error_56;
        }
      }
      /* UNREACHABLE */;
      # 终止程序执行
      abort();
    }
    # 定义状态 s_n_llhttp__internal__n_start_req_29
    case s_n_llhttp__internal__n_start_req_29:
    s_n_llhttp__internal__n_start_req_29: {
      # 如果指针 p 已经指向输入结束位置，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_start_req_29;
      }
      # 根据指针 p 指向的字符进行不同的处理
      switch (*p) {
        # 如果字符为 'L'
        case 'L': {
          # 更新指针 p 的位置
          p++;
          # 更新变量 match 的值
          match = 10;
          # 跳转到状态 s_n_llhttp__internal__n_invoke_store_method_1
          goto s_n_llhttp__internal__n_invoke_store_method_1;
        }
        # 如果字符不为 'L'
        default: {
          # 跳转到状态 s_n_llhttp__internal__n_error_56
          goto s_n_llhttp__internal__n_error_56;
        }
      }
      /* UNREACHABLE */;
      # 终止程序执行
      abort();
    }
    # 定义状态 s_n_llhttp__internal__n_start_req_27
    case s_n_llhttp__internal__n_start_req_27:
    s_n_llhttp__internal__n_start_req_27: {
      # 如果指针 p 已经指向输入结束位置，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_start_req_27;
      }
      # 根据指针 p 指向的字符进行不同的处理
      switch (*p) {
        # 如果字符为 'A'
        case 'A': {
          # 更新指针 p 的位置
          p++;
          # 跳转到状态 s_n_llhttp__internal__n_start_req_28
          goto s_n_llhttp__internal__n_start_req_28;
        }
        # 如果字符为 'O'
        case 'O': {
          # 更新指针 p 的位置
          p++;
          # 跳转到状态 s_n_llhttp__internal__n_start_req_29
          goto s_n_llhttp__internal__n_start_req_29;
        }
        # 如果字符既不是 'A' 也不是 'O'
        default: {
          # 跳转到状态 s_n_llhttp__internal__n_error_56
          goto s_n_llhttp__internal__n_error_56;
        }
      }
      /* UNREACHABLE */;
      # 终止程序执行
      abort();
    }
    # 定义状态 s_n_llhttp__internal__n_start_req_25:
    # 定义状态 s_n_llhttp__internal__n_start_req_25，处理请求行起始的状态
    s_n_llhttp__internal__n_start_req_25: {
      # 如果指针已经指向输入的结尾，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_start_req_25;
      }
      # 根据当前字符进行不同的处理
      switch (*p) {
        # 如果当前字符是 'A'，则向前移动一个字符，跳转到状态 s_n_llhttp__internal__n_start_req_26
        case 'A': {
          p++;
          goto s_n_llhttp__internal__n_start_req_26;
        }
        # 如果当前字符是 'C'，则向前移动一个字符，跳转到状态 s_n_llhttp__internal__n_start_req_27
        case 'C': {
          p++;
          goto s_n_llhttp__internal__n_start_req_27;
        }
        # 如果当前字符不是 'A' 或 'C'，则跳转到状态 s_n_llhttp__internal__n_error_56
        default: {
          goto s_n_llhttp__internal__n_error_56;
        }
      }
      # 不可达的代码，中止程序
      /* UNREACHABLE */;
      abort();
    }
    # 定义状态 s_n_llhttp__internal__n_start_req_30
    case s_n_llhttp__internal__n_start_req_30:
    s_n_llhttp__internal__n_start_req_30: {
      # 定义变量 match_seq
      llparse_match_t match_seq;

      # 如果指针已经指向输入的结尾，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_start_req_30;
      }
      # 调用 llparse__match_sequence_id 函数，匹配输入序列，返回匹配结果
      match_seq = llparse__match_sequence_id(state, p, endp, llparse_blob34, 2);
      # 更新指针位置
      p = match_seq.current;
      # 根据匹配结果进行不同的处理
      switch (match_seq.status) {
        # 如果匹配完成，向前移动一个字符，设置 match 变量为 11，跳转到状态 s_n_llhttp__internal__n_invoke_store_method_1
        case kMatchComplete: {
          p++;
          match = 11;
          goto s_n_llhttp__internal__n_invoke_store_method_1;
        }
        # 如果匹配暂停，返回当前状态
        case kMatchPause: {
          return s_n_llhttp__internal__n_start_req_30;
        }
        # 如果匹配不成功，跳转到状态 s_n_llhttp__internal__n_error_56
        case kMatchMismatch: {
          goto s_n_llhttp__internal__n_error_56;
        }
      }
      # 不可达的代码，中止程序
      /* UNREACHABLE */;
      abort();
    }
    # 定义状态 s_n_llhttp__internal__n_start_req_22
    case s_n_llhttp__internal__n_start_req_22:
    s_n_llhttp__internal__n_start_req_22: {
      # 如果指针已经指向输入的结尾，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_start_req_22;
      }
      # 根据当前字符进行不同的处理
      switch (*p) {
        # 如果当前字符是 '-'，则向前移动一个字符，跳转到状态 s_n_llhttp__internal__n_start_req_23
        case '-': {
          p++;
          goto s_n_llhttp__internal__n_start_req_23;
        }
        # 如果当前字符是 'E'，则向前移动一个字符，跳转到状态 s_n_llhttp__internal__n_start_req_24
        case 'E': {
          p++;
          goto s_n_llhttp__internal__n_start_req_24;
        }
        # 如果当前字符是 'K'，则向前移动一个字符，跳转到状态 s_n_llhttp__internal__n_start_req_25
        case 'K': {
          p++;
          goto s_n_llhttp__internal__n_start_req_25;
        }
        # 如果当前字符是 'O'，则向前移动一个字符，跳转到状态 s_n_llhttp__internal__n_start_req_30
        case 'O': {
          p++;
          goto s_n_llhttp__internal__n_start_req_30;
        }
        # 如果当前字符不是 '-'、'E'、'K' 或 'O'，则跳转到状态 s_n_llhttp__internal__n_error_56
        default: {
          goto s_n_llhttp__internal__n_error_56;
        }
      }
      # 不可达的代码，中止程序
      /* UNREACHABLE */;
      abort();
    }
    # 定义状态 s_n_llhttp__internal__n_start_req_31
    case s_n_llhttp__internal__n_start_req_31:
    # 定义状态 s_n_llhttp__internal__n_start_req_31，开始解析请求的状态
    s_n_llhttp__internal__n_start_req_31: {
      # 定义匹配序列的变量
      llparse_match_t match_seq;

      # 如果指针已经到达结束位置，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_start_req_31;
      }
      # 调用匹配序列函数，获取匹配结果
      match_seq = llparse__match_sequence_id(state, p, endp, llparse_blob35, 5);
      # 更新指针位置
      p = match_seq.current;
      # 根据匹配结果的状态进行不同的处理
      switch (match_seq.status) {
        # 如果匹配完成
        case kMatchComplete: {
          # 指针后移一位
          p++;
          # 设置匹配值
          match = 25;
          # 跳转到下一个状态
          goto s_n_llhttp__internal__n_invoke_store_method_1;
        }
        # 如果匹配暂停
        case kMatchPause: {
          # 返回当前状态
          return s_n_llhttp__internal__n_start_req_31;
        }
        # 如果匹配不成功
        case kMatchMismatch: {
          # 跳转到错误状态
          goto s_n_llhttp__internal__n_error_56;
        }
      }
      /* UNREACHABLE */;
      # 终止程序
      abort();
    }
    # 定义状态 s_n_llhttp__internal__n_start_req_32
    case s_n_llhttp__internal__n_start_req_32:
    s_n_llhttp__internal__n_start_req_32: {
      # 定义匹配序列的变量
      llparse_match_t match_seq;

      # 如果指针已经到达结束位置，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_start_req_32;
      }
      # 调用匹配序列函数，获取匹配结果
      match_seq = llparse__match_sequence_id(state, p, endp, llparse_blob36, 6);
      # 更新指针位置
      p = match_seq.current;
      # 根据匹配结果的状态进行不同的处理
      switch (match_seq.status) {
        # 如果匹配完成
        case kMatchComplete: {
          # 指针后移一位
          p++;
          # 设置匹配值
          match = 6;
          # 跳转到下一个状态
          goto s_n_llhttp__internal__n_invoke_store_method_1;
        }
        # 如果匹配暂停
        case kMatchPause: {
          # 返回当前状态
          return s_n_llhttp__internal__n_start_req_32;
        }
        # 如果匹配不成功
        case kMatchMismatch: {
          # 跳转到错误状态
          goto s_n_llhttp__internal__n_error_56;
        }
      }
      /* UNREACHABLE */;
      # 终止程序
      abort();
    }
    # 定义状态 s_n_llhttp__internal__n_start_req_35:
    # 定义状态 s_n_llhttp__internal__n_start_req_35，开始解析请求的状态
    s_n_llhttp__internal__n_start_req_35: {
      # 定义匹配序列的变量
      llparse_match_t match_seq;

      # 如果指针已经到达结束位置，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_start_req_35;
      }
      # 调用 llparse__match_sequence_id 函数，匹配指定的序列
      match_seq = llparse__match_sequence_id(state, p, endp, llparse_blob37, 2);
      # 更新指针位置
      p = match_seq.current;
      # 根据匹配的状态进行不同的处理
      switch (match_seq.status) {
        # 如果匹配完成
        case kMatchComplete: {
          # 更新指针位置
          p++;
          # 设置匹配值
          match = 28;
          # 跳转到指定状态
          goto s_n_llhttp__internal__n_invoke_store_method_1;
        }
        # 如果匹配暂停
        case kMatchPause: {
          # 返回当前状态
          return s_n_llhttp__internal__n_start_req_35;
        }
        # 如果匹配不成功
        case kMatchMismatch: {
          # 跳转到错误状态
          goto s_n_llhttp__internal__n_error_56;
        }
      }
      /* UNREACHABLE */;
      # 终止程序
      abort();
    }
    # 定义状态 s_n_llhttp__internal__n_start_req_36
    case s_n_llhttp__internal__n_start_req_36:
    s_n_llhttp__internal__n_start_req_36: {
      # 定义匹配序列的变量
      llparse_match_t match_seq;

      # 如果指针已经到达结束位置，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_start_req_36;
      }
      # 调用 llparse__match_sequence_id 函数，匹配指定的序列
      match_seq = llparse__match_sequence_id(state, p, endp, llparse_blob38, 2);
      # 更新指针位置
      p = match_seq.current;
      # 根据匹配的状态进行不同的处理
      switch (match_seq.status) {
        # 如果匹配完成
        case kMatchComplete: {
          # 更新指针位置
          p++;
          # 设置匹配值
          match = 39;
          # 跳转到指定状态
          goto s_n_llhttp__internal__n_invoke_store_method_1;
        }
        # 如果匹配暂停
        case kMatchPause: {
          # 返回当前状态
          return s_n_llhttp__internal__n_start_req_36;
        }
        # 如果匹配不成功
        case kMatchMismatch: {
          # 跳转到错误状态
          goto s_n_llhttp__internal__n_error_56;
        }
      }
      /* UNREACHABLE */;
      # 终止程序
      abort();
    }
    # 定义状态 s_n_llhttp__internal__n_start_req_34
    case s_n_llhttp__internal__n_start_req_34:
    s_n_llhttp__internal__n_start_req_34: {
      # 如果指针已经到达结束位置，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_start_req_34;
      }
      # 根据当前字符进行不同的处理
      switch (*p) {
        # 如果当前字符为 'T'
        case 'T': {
          # 更新指针位置
          p++;
          # 跳转到指定状态
          goto s_n_llhttp__internal__n_start_req_35;
        }
        # 如果当前字符为 'U'
        case 'U': {
          # 更新指针位置
          p++;
          # 跳转到指定状态
          goto s_n_llhttp__internal__n_start_req_36;
        }
        # 如果当前字符不是 'T' 或 'U'
        default: {
          # 跳转到错误状态
          goto s_n_llhttp__internal__n_error_56;
        }
      }
      /* UNREACHABLE */;
      # 终止程序
      abort();
    }
    # 定义状态 s_n_llhttp__internal__n_start_req_37:
    # 定义状态 s_n_llhttp__internal__n_start_req_37，处理请求开始的状态
    s_n_llhttp__internal__n_start_req_37: {
      # 定义匹配序列的变量
      llparse_match_t match_seq;

      # 如果指针已经到达结束位置，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_start_req_37;
      }
      # 调用 llparse__match_sequence_id 函数，匹配指定的序列
      match_seq = llparse__match_sequence_id(state, p, endp, llparse_blob39, 2);
      # 更新指针位置
      p = match_seq.current;
      # 根据匹配的状态进行处理
      switch (match_seq.status) {
        # 如果匹配完成
        case kMatchComplete: {
          # 指针后移一位
          p++;
          # 设置匹配的状态
          match = 38;
          # 跳转到指定状态
          goto s_n_llhttp__internal__n_invoke_store_method_1;
        }
        # 如果匹配暂停
        case kMatchPause: {
          # 返回当前状态
          return s_n_llhttp__internal__n_start_req_37;
        }
        # 如果匹配不成功
        case kMatchMismatch: {
          # 跳转到错误处理状态
          goto s_n_llhttp__internal__n_error_56;
        }
      }
      # 不可达的代码，中断程序
      /* UNREACHABLE */;
      abort();
    }
    # 定义状态 s_n_llhttp__internal__n_start_req_38
    case s_n_llhttp__internal__n_start_req_38:
    s_n_llhttp__internal__n_start_req_38: {
      # 定义匹配序列的变量
      llparse_match_t match_seq;

      # 如果指针已经到达结束位置，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_start_req_38;
      }
      # 调用 llparse__match_sequence_id 函数，匹配指定的序列
      match_seq = llparse__match_sequence_id(state, p, endp, llparse_blob40, 2);
      # 更新指针位置
      p = match_seq.current;
      # 根据匹配的状态进行处理
      switch (match_seq.status) {
        # 如果匹配完成
        case kMatchComplete: {
          # 指针后移一位
          p++;
          # 设置匹配的状态
          match = 3;
          # 跳转到指定状态
          goto s_n_llhttp__internal__n_invoke_store_method_1;
        }
        # 如果匹配暂停
        case kMatchPause: {
          # 返回当前状态
          return s_n_llhttp__internal__n_start_req_38;
        }
        # 如果匹配不成功
        case kMatchMismatch: {
          # 跳转到错误处理状态
          goto s_n_llhttp__internal__n_error_56;
        }
      }
      # 不可达的代码，中断程序
      /* UNREACHABLE */;
      abort();
    }
    # 定义状态 s_n_llhttp__internal__n_start_req_42
    case s_n_llhttp__internal__n_start_req_42:
    # 定义状态 s_n_llhttp__internal__n_start_req_42，包含匹配结果的变量
    s_n_llhttp__internal__n_start_req_42: {
      llparse_match_t match_seq;

      # 如果指针已经到达结束位置，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_start_req_42;
      }
      # 调用 llparse__match_sequence_id 函数进行匹配，获取匹配结果
      match_seq = llparse__match_sequence_id(state, p, endp, llparse_blob41, 3);
      p = match_seq.current;
      # 根据匹配结果的状态进行不同的处理
      switch (match_seq.status) {
        case kMatchComplete: {
          # 如果匹配完成，则指针后移一位，设置匹配结果为 12，跳转到状态 s_n_llhttp__internal__n_invoke_store_method_1
          p++;
          match = 12;
          goto s_n_llhttp__internal__n_invoke_store_method_1;
        }
        case kMatchPause: {
          # 如果匹配暂停，则返回当前状态
          return s_n_llhttp__internal__n_start_req_42;
        }
        case kMatchMismatch: {
          # 如果匹配失败，则跳转到状态 s_n_llhttp__internal__n_error_56
          goto s_n_llhttp__internal__n_error_56;
        }
      }
      /* UNREACHABLE */;
      # 如果代码执行到这里，说明出现了不可达的情况，终止程序
      abort();
    }
    # 定义状态 s_n_llhttp__internal__n_start_req_43
    case s_n_llhttp__internal__n_start_req_43:
    s_n_llhttp__internal__n_start_req_43: {
      llparse_match_t match_seq;

      # 如果指针已经到达结束位置，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_start_req_43;
      }
      # 调用 llparse__match_sequence_id 函数进行匹配，获取匹配结果
      match_seq = llparse__match_sequence_id(state, p, endp, llparse_blob42, 4);
      p = match_seq.current;
      # 根据匹配结果的状态进行不同的处理
      switch (match_seq.status) {
        case kMatchComplete: {
          # 如果匹配完成，则指针后移一位，设置匹配结果为 13，跳转到状态 s_n_llhttp__internal__n_invoke_store_method_1
          p++;
          match = 13;
          goto s_n_llhttp__internal__n_invoke_store_method_1;
        }
        case kMatchPause: {
          # 如果匹配暂停，则返回当前状态
          return s_n_llhttp__internal__n_start_req_43;
        }
        case kMatchMismatch: {
          # 如果匹配失败，则跳转到状态 s_n_llhttp__internal__n_error_56
          goto s_n_llhttp__internal__n_error_56;
        }
      }
      /* UNREACHABLE */;
      # 如果代码执行到这里，说明出现了不可达的情况，终止程序
      abort();
    }
    # 定义状态 s_n_llhttp__internal__n_start_req_41
    case s_n_llhttp__internal__n_start_req_41:
    s_n_llhttp__internal__n_start_req_41: {
      # 如果指针已经到达结束位置，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_start_req_41;
      }
      # 根据当前指针指向的字符进行不同的处理
      switch (*p) {
        case 'F': {
          # 如果是字符 'F'，则指针后移一位，跳转到状态 s_n_llhttp__internal__n_start_req_42
          p++;
          goto s_n_llhttp__internal__n_start_req_42;
        }
        case 'P': {
          # 如果是字符 'P'，则指针后移一位，跳转到状态 s_n_llhttp__internal__n_start_req_43
          p++;
          goto s_n_llhttp__internal__n_start_req_43;
        }
        default: {
          # 如果不是字符 'F' 或 'P'，则跳转到状态 s_n_llhttp__internal__n_error_56
          goto s_n_llhttp__internal__n_error_56;
        }
      }
      /* UNREACHABLE */;
      # 如果代码执行到这里，说明出现了不可达的情况，终止程序
      abort();
    }
    # 定义状态 s_n_llhttp__internal__n_start_req_40
    # 定义状态 s_n_llhttp__internal__n_start_req_40，处理请求行的第一个字符
    s_n_llhttp__internal__n_start_req_40: {
      # 如果指针已经指向输入的末尾，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_start_req_40;
      }
      # 根据当前字符进行判断
      switch (*p) {
        # 如果当前字符是 'P'，则跳转到状态 s_n_llhttp__internal__n_start_req_41
        case 'P': {
          p++;
          goto s_n_llhttp__internal__n_start_req_41;
        }
        # 如果当前字符不是 'P'，则跳转到状态 s_n_llhttp__internal__n_error_56
        default: {
          goto s_n_llhttp__internal__n_error_56;
        }
      }
      # 不可达的代码，中止程序
      /* UNREACHABLE */;
      abort();
    }
    # 定义状态 s_n_llhttp__internal__n_start_req_39，处理请求行的第一个字符
    s_n_llhttp__internal__n_start_req_39: {
      # 如果指针已经指向输入的末尾，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_start_req_39;
      }
      # 根据当前字符进行判断
      switch (*p) {
        # 如果当前字符是 'I'，则跳转到状态 s_n_llhttp__internal__n_invoke_store_method_1
        case 'I': {
          p++;
          match = 34;
          goto s_n_llhttp__internal__n_invoke_store_method_1;
        }
        # 如果当前字符是 'O'，则跳转到状态 s_n_llhttp__internal__n_start_req_40
        case 'O': {
          p++;
          goto s_n_llhttp__internal__n_start_req_40;
        }
        # 如果当前字符既不是 'I' 也不是 'O'，则跳转到状态 s_n_llhttp__internal__n_error_56
        default: {
          goto s_n_llhttp__internal__n_error_56;
        }
      }
      # 不可达的代码，中止程序
      /* UNREACHABLE */;
      abort();
    }
    # 定义状态 s_n_llhttp__internal__n_start_req_45，处理请求行的第一个字符
    s_n_llhttp__internal__n_start_req_45: {
      # 定义变量 match_seq
      llparse_match_t match_seq;
      # 如果指针已经指向输入的末尾，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_start_req_45;
      }
      # 调用 llparse__match_sequence_id 函数，根据输入的状态、指针和结束指针，进行匹配
      match_seq = llparse__match_sequence_id(state, p, endp, llparse_blob43, 2);
      # 更新指针位置
      p = match_seq.current;
      # 根据匹配的状态进行判断
      switch (match_seq.status) {
        # 如果匹配完成，则跳转到状态 s_n_llhttp__internal__n_invoke_store_method_1
        case kMatchComplete: {
          p++;
          match = 29;
          goto s_n_llhttp__internal__n_invoke_store_method_1;
        }
        # 如果匹配暂停，则返回当前状态
        case kMatchPause: {
          return s_n_llhttp__internal__n_start_req_45;
        }
        # 如果匹配不成功，则跳转到状态 s_n_llhttp__internal__n_error_56
        case kMatchMismatch: {
          goto s_n_llhttp__internal__n_error_56;
        }
      }
      # 不可达的代码，中止程序
      /* UNREACHABLE */;
      abort();
    }
    # 定义状态 s_n_llhttp__internal__n_start_req_44:
    # 定义状态 s_n_llhttp__internal__n_start_req_44，处理请求行起始的状态
    s_n_llhttp__internal__n_start_req_44: {
      # 如果指针已经指向结束位置，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_start_req_44;
      }
      # 根据当前字符进行判断
      switch (*p) {
        # 如果当前字符是 'R'，则跳过当前字符，进入状态 s_n_llhttp__internal__n_start_req_45
        case 'R': {
          p++;
          goto s_n_llhttp__internal__n_start_req_45;
        }
        # 如果当前字符是 'T'，则跳过当前字符，设置 match 为 4，进入状态 s_n_llhttp__internal__n_invoke_store_method_1
        case 'T': {
          p++;
          match = 4;
          goto s_n_llhttp__internal__n_invoke_store_method_1;
        }
        # 如果当前字符不是 'R' 或 'T'，则进入状态 s_n_llhttp__internal__n_error_56
        default: {
          goto s_n_llhttp__internal__n_error_56;
        }
      }
      # 不可达的代码，中止程序
      /* UNREACHABLE */;
      abort();
    }
    # 定义状态 s_n_llhttp__internal__n_start_req_33，处理请求行起始的状态
    case s_n_llhttp__internal__n_start_req_33:
    s_n_llhttp__internal__n_start_req_33: {
      # 如果指针已经指向结束位置，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_start_req_33;
      }
      # 根据当前字符进行判断
      switch (*p) {
        # 如果当前字符是 'A'，则跳过当前字符，进入状态 s_n_llhttp__internal__n_start_req_34
        case 'A': {
          p++;
          goto s_n_llhttp__internal__n_start_req_34;
        }
        # 如果当前字符是 'L'，则跳过当前字符，进入状态 s_n_llhttp__internal__n_start_req_37
        case 'L': {
          p++;
          goto s_n_llhttp__internal__n_start_req_37;
        }
        # 如果当前字符是 'O'，则跳过当前字符，进入状态 s_n_llhttp__internal__n_start_req_38
        case 'O': {
          p++;
          goto s_n_llhttp__internal__n_start_req_38;
        }
        # 如果当前字符是 'R'，则跳过当前字符，进入状态 s_n_llhttp__internal__n_start_req_39
        case 'R': {
          p++;
          goto s_n_llhttp__internal__n_start_req_39;
        }
        # 如果当前字符是 'U'，则跳过当前字符，进入状态 s_n_llhttp__internal__n_start_req_44
        case 'U': {
          p++;
          goto s_n_llhttp__internal__n_start_req_44;
        }
        # 如果当前字符不是 'A'、'L'、'O'、'R'、'U'，则进入状态 s_n_llhttp__internal__n_error_56
        default: {
          goto s_n_llhttp__internal__n_error_56;
        }
      }
      # 不可达的代码，中止程序
      /* UNREACHABLE */;
      abort();
    }
    # 定义状态 s_n_llhttp__internal__n_start_req_48
    case s_n_llhttp__internal__n_start_req_48:
    # 定义状态 s_n_llhttp__internal__n_start_req_48，处理请求开始的状态
    s_n_llhttp__internal__n_start_req_48: {
      # 定义匹配序列的变量
      llparse_match_t match_seq;

      # 如果指针已经到达结束位置，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_start_req_48;
      }
      # 调用 llparse__match_sequence_id 函数，匹配指定的序列
      match_seq = llparse__match_sequence_id(state, p, endp, llparse_blob44, 3);
      # 更新指针位置
      p = match_seq.current;
      # 根据匹配的状态进行处理
      switch (match_seq.status) {
        # 如果匹配完成
        case kMatchComplete: {
          # 更新指针位置
          p++;
          # 设置匹配值
          match = 17;
          # 跳转到指定状态
          goto s_n_llhttp__internal__n_invoke_store_method_1;
        }
        # 如果匹配暂停
        case kMatchPause: {
          # 返回当前状态
          return s_n_llhttp__internal__n_start_req_48;
        }
        # 如果匹配不成功
        case kMatchMismatch: {
          # 跳转到错误处理状态
          goto s_n_llhttp__internal__n_error_56;
        }
      }
      # 不可达的代码，中断程序执行
      /* UNREACHABLE */;
      abort();
    }
    # 定义状态 s_n_llhttp__internal__n_start_req_49，处理请求开始的状态
    s_n_llhttp__internal__n_start_req_49: {
      # 定义匹配序列的变量
      llparse_match_t match_seq;

      # 如果指针已经到达结束位置，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_start_req_49;
      }
      # 调用 llparse__match_sequence_id 函数，匹配指定的序列
      match_seq = llparse__match_sequence_id(state, p, endp, llparse_blob45, 3);
      # 更新指针位置
      p = match_seq.current;
      # 根据匹配的状态进行处理
      switch (match_seq.status) {
        # 如果匹配完成
        case kMatchComplete: {
          # 更新指针位置
          p++;
          # 设置匹配值
          match = 44;
          # 跳转到指定状态
          goto s_n_llhttp__internal__n_invoke_store_method_1;
        }
        # 如果匹配暂停
        case kMatchPause: {
          # 返回当前状态
          return s_n_llhttp__internal__n_start_req_49;
        }
        # 如果匹配不成功
        case kMatchMismatch: {
          # 跳转到错误处理状态
          goto s_n_llhttp__internal__n_error_56;
        }
      }
      # 不可达的代码，中断程序执行
      /* UNREACHABLE */;
      abort();
    }
    # 定义状态 s_n_llhttp__internal__n_start_req_50:
    # 定义状态 s_n_llhttp__internal__n_start_req_50，表示解析 HTTP 请求开始的状态
    s_n_llhttp__internal__n_start_req_50: {
      # 定义变量 match_seq，用于存储匹配结果
      llparse_match_t match_seq;

      # 如果指针已经到达数据结尾，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_start_req_50;
      }
      # 调用 llparse__match_sequence_id 函数，匹配指定长度的数据序列
      match_seq = llparse__match_sequence_id(state, p, endp, llparse_blob46, 5);
      # 更新指针位置
      p = match_seq.current;
      # 根据匹配结果的状态进行不同的处理
      switch (match_seq.status) {
        # 如果匹配完成
        case kMatchComplete: {
          # 指针后移一位
          p++;
          # 设置匹配值为 43
          match = 43;
          # 跳转到状态 s_n_llhttp__internal__n_invoke_store_method_1
          goto s_n_llhttp__internal__n_invoke_store_method_1;
        }
        # 如果匹配暂停
        case kMatchPause: {
          # 返回当前状态
          return s_n_llhttp__internal__n_start_req_50;
        }
        # 如果匹配失败
        case kMatchMismatch: {
          # 跳转到状态 s_n_llhttp__internal__n_error_56
          goto s_n_llhttp__internal__n_error_56;
        }
      }
      /* UNREACHABLE */;
      # 终止程序执行
      abort();
    }
    # 定义状态 s_n_llhttp__internal__n_start_req_51
    case s_n_llhttp__internal__n_start_req_51:
    s_n_llhttp__internal__n_start_req_51: {
      # 定义变量 match_seq，用于存储匹配结果
      llparse_match_t match_seq;

      # 如果指针已经到达数据结尾，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_start_req_51;
      }
      # 调用 llparse__match_sequence_id 函数，匹配指定长度的数据序列
      match_seq = llparse__match_sequence_id(state, p, endp, llparse_blob47, 3);
      # 更新指针位置
      p = match_seq.current;
      # 根据匹配结果的状态进行不同的处理
      switch (match_seq.status) {
        # 如果匹配完成
        case kMatchComplete: {
          # 指针后移一位
          p++;
          # 设置匹配值为 20
          match = 20;
          # 跳转到状态 s_n_llhttp__internal__n_invoke_store_method_1
          goto s_n_llhttp__internal__n_invoke_store_method_1;
        }
        # 如果匹配暂停
        case kMatchPause: {
          # 返回当前状态
          return s_n_llhttp__internal__n_start_req_51;
        }
        # 如果匹配失败
        case kMatchMismatch: {
          # 跳转到状态 s_n_llhttp__internal__n_error_56
          goto s_n_llhttp__internal__n_error_56;
        }
      }
      /* UNREACHABLE */;
      # 终止程序执行
      abort();
    }
    # 定义状态 s_n_llhttp__internal__n_start_req_47
    case s_n_llhttp__internal__n_start_req_47:
    # 如果当前指针指向结束位置，则返回当前状态
    s_n_llhttp__internal__n_start_req_47: {
      if (p == endp) {
        return s_n_llhttp__internal__n_start_req_47;
      }
      # 根据当前字符进行不同的跳转
      switch (*p) {
        # 如果当前字符为'B'，则向前移动一个字符并跳转到状态 s_n_llhttp__internal__n_start_req_48
        case 'B': {
          p++;
          goto s_n_llhttp__internal__n_start_req_48;
        }
        # 如果当前字符为'C'，则向前移动一个字符并跳转到状态 s_n_llhttp__internal__n_start_req_49
        case 'C': {
          p++;
          goto s_n_llhttp__internal__n_start_req_49;
        }
        # 如果当前字符为'D'，则向前移动一个字符并跳转到状态 s_n_llhttp__internal__n_start_req_50
        case 'D': {
          p++;
          goto s_n_llhttp__internal__n_start_req_50;
        }
        # 如果当前字符为'P'，则向前移动一个字符并跳转到状态 s_n_llhttp__internal__n_start_req_51
        case 'P': {
          p++;
          goto s_n_llhttp__internal__n_start_req_51;
        }
        # 如果当前字符不匹配任何情况，则跳转到状态 s_n_llhttp__internal__n_error_56
        default: {
          goto s_n_llhttp__internal__n_error_56;
        }
      }
      /* UNREACHABLE */;
      # 终止程序
      abort();
    }
    # 如果当前状态为 s_n_llhttp__internal__n_start_req_46
    case s_n_llhttp__internal__n_start_req_46: {
      if (p == endp) {
        return s_n_llhttp__internal__n_start_req_46;
      }
      # 根据当前字符进行不同的跳转
      switch (*p) {
        # 如果当前字符为'E'，则向前移动一个字符并跳转到状态 s_n_llhttp__internal__n_start_req_47
        case 'E': {
          p++;
          goto s_n_llhttp__internal__n_start_req_47;
        }
        # 如果当前字符不匹配任何情况，则跳转到状态 s_n_llhttp__internal__n_error_56
        default: {
          goto s_n_llhttp__internal__n_error_56;
        }
      }
      /* UNREACHABLE */;
      # 终止程序
      abort();
    }
    # 如果当前状态为 s_n_llhttp__internal__n_start_req_54
    case s_n_llhttp__internal__n_start_req_54: {
      llparse_match_t match_seq;

      if (p == endp) {
        return s_n_llhttp__internal__n_start_req_54;
      }
      # 进行匹配操作，根据匹配结果进行不同的处理
      match_seq = llparse__match_sequence_id(state, p, endp, llparse_blob48, 3);
      p = match_seq.current;
      switch (match_seq.status) {
        # 如果匹配完成，则向前移动一个字符，设置匹配值为14，并跳转到状态 s_n_llhttp__internal__n_invoke_store_method_1
        case kMatchComplete: {
          p++;
          match = 14;
          goto s_n_llhttp__internal__n_invoke_store_method_1;
        }
        # 如果匹配暂停，则返回当前状态
        case kMatchPause: {
          return s_n_llhttp__internal__n_start_req_54;
        }
        # 如果匹配不成功，则跳转到状态 s_n_llhttp__internal__n_error_56
        case kMatchMismatch: {
          goto s_n_llhttp__internal__n_error_56;
        }
      }
      /* UNREACHABLE */;
      # 终止程序
      abort();
    }
    # 如果当前状态为 s_n_llhttp__internal__n_start_req_56
    # 定义状态 s_n_llhttp__internal__n_start_req_56，处理请求开始的状态
    s_n_llhttp__internal__n_start_req_56: {
      # 如果指针已经到达结束位置，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_start_req_56;
      }
      # 根据当前字符进行判断
      switch (*p) {
        # 如果是 'P'，则向前移动一个字符，设置匹配值为 37，跳转到状态 s_n_llhttp__internal__n_invoke_store_method_1
        case 'P': {
          p++;
          match = 37;
          goto s_n_llhttp__internal__n_invoke_store_method_1;
        }
        # 如果不是 'P'，则跳转到错误状态 s_n_llhttp__internal__n_error_56
        default: {
          goto s_n_llhttp__internal__n_error_56;
        }
      }
      # 不可达的代码，中止程序
      /* UNREACHABLE */;
      abort();
    }
    # 定义状态 s_n_llhttp__internal__n_start_req_57
    case s_n_llhttp__internal__n_start_req_57:
    s_n_llhttp__internal__n_start_req_57: {
      # 定义匹配序列
      llparse_match_t match_seq;

      # 如果指针已经到达结束位置，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_start_req_57;
      }
      # 匹配序列为指定长度的字符序列，返回匹配状态和当前指针位置
      match_seq = llparse__match_sequence_id(state, p, endp, llparse_blob49, 9);
      p = match_seq.current;
      # 根据匹配状态进行处理
      switch (match_seq.status) {
        # 如果匹配完成，则向前移动一个字符，设置匹配值为 42，跳转到状态 s_n_llhttp__internal__n_invoke_store_method_1
        case kMatchComplete: {
          p++;
          match = 42;
          goto s_n_llhttp__internal__n_invoke_store_method_1;
        }
        # 如果匹配暂停，则返回当前状态
        case kMatchPause: {
          return s_n_llhttp__internal__n_start_req_57;
        }
        # 如果匹配不成功，则跳转到错误状态 s_n_llhttp__internal__n_error_56
        case kMatchMismatch: {
          goto s_n_llhttp__internal__n_error_56;
        }
      }
      # 不可达的代码，中止程序
      /* UNREACHABLE */;
      abort();
    }
    # 定义状态 s_n_llhttp__internal__n_start_req_55
    case s_n_llhttp__internal__n_start_req_55:
    s_n_llhttp__internal__n_start_req_55: {
      # 如果指针已经到达结束位置，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_start_req_55;
      }
      # 根据当前字符进行判断
      switch (*p) {
        # 如果是 'U'，则向前移动一个字符，跳转到状态 s_n_llhttp__internal__n_start_req_56
        case 'U': {
          p++;
          goto s_n_llhttp__internal__n_start_req_56;
        }
        # 如果是 '_'，则向前移动一个字符，跳转到状态 s_n_llhttp__internal__n_start_req_57
        case '_': {
          p++;
          goto s_n_llhttp__internal__n_start_req_57;
        }
        # 如果不是 'U' 或 '_'，则跳转到错误状态 s_n_llhttp__internal__n_error_56
        default: {
          goto s_n_llhttp__internal__n_error_56;
        }
      }
      # 不可达的代码，中止程序
      /* UNREACHABLE */;
      abort();
    }
    # 定义状态 s_n_llhttp__internal__n_start_req_53
    # 定义状态 s_n_llhttp__internal__n_start_req_53，处理请求起始部分的状态机
    s_n_llhttp__internal__n_start_req_53: {
      # 如果指针已经到达结束位置，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_start_req_53;
      }
      # 根据当前字符进行判断
      switch (*p) {
        # 如果是 'A'，则指针向前移动一位，跳转到状态 s_n_llhttp__internal__n_start_req_54
        case 'A': {
          p++;
          goto s_n_llhttp__internal__n_start_req_54;
        }
        # 如果是 'T'，则指针向前移动一位，跳转到状态 s_n_llhttp__internal__n_start_req_55
        case 'T': {
          p++;
          goto s_n_llhttp__internal__n_start_req_55;
        }
        # 如果不是 'A' 或 'T'，则跳转到状态 s_n_llhttp__internal__n_error_56
        default: {
          goto s_n_llhttp__internal__n_error_56;
        }
      }
      # 不可达的代码，中止程序
      /* UNREACHABLE */;
      abort();
    }
    # 定义状态 s_n_llhttp__internal__n_start_req_58
    case s_n_llhttp__internal__n_start_req_58:
    s_n_llhttp__internal__n_start_req_58: {
      # 定义匹配结果
      llparse_match_t match_seq;

      # 如果指针已经到达结束位置，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_start_req_58;
      }
      # 进行匹配操作，获取匹配结果
      match_seq = llparse__match_sequence_id(state, p, endp, llparse_blob50, 4);
      p = match_seq.current;
      # 根据匹配结果进行处理
      switch (match_seq.status) {
        # 如果匹配完成，则指针向前移动一位，设置匹配值为 33，跳转到状态 s_n_llhttp__internal__n_invoke_store_method_1
        case kMatchComplete: {
          p++;
          match = 33;
          goto s_n_llhttp__internal__n_invoke_store_method_1;
        }
        # 如果匹配暂停，则返回当前状态
        case kMatchPause: {
          return s_n_llhttp__internal__n_start_req_58;
        }
        # 如果匹配不成功，则跳转到状态 s_n_llhttp__internal__n_error_56
        case kMatchMismatch: {
          goto s_n_llhttp__internal__n_error_56;
        }
      }
      # 不可达的代码，中止程序
      /* UNREACHABLE */;
      abort();
    }
    # 定义状态 s_n_llhttp__internal__n_start_req_59
    case s_n_llhttp__internal__n_start_req_59:
    s_n_llhttp__internal__n_start_req_59: {
      # 定义匹配结果
      llparse_match_t match_seq;

      # 如果指针已经到达结束位置，则返回当前状��
      if (p == endp) {
        return s_n_llhttp__internal__n_start_req_59;
      }
      # 进行匹配操作，获取匹配结果
      match_seq = llparse__match_sequence_id(state, p, endp, llparse_blob51, 7);
      p = match_seq.current;
      # 根据匹配结果进行处理
      switch (match_seq.status) {
        # 如果匹配完成，则指针向前移动一位，设置匹配值为 26，跳转到状态 s_n_llhttp__internal__n_invoke_store_method_1
        case kMatchComplete: {
          p++;
          match = 26;
          goto s_n_llhttp__internal__n_invoke_store_method_1;
        }
        # 如果匹配暂停，则返回当前状态
        case kMatchPause: {
          return s_n_llhttp__internal__n_start_req_59;
        }
        # 如果匹配不成功，则跳转到状态 s_n_llhttp__internal__n_error_56
        case kMatchMismatch: {
          goto s_n_llhttp__internal__n_error_56;
        }
      }
      # 不可达的代码，中止程序
      /* UNREACHABLE */;
      abort();
    }
    # 定义状态 s_n_llhttp__internal__n_start_req_52
    # 定义状态 s_n_llhttp__internal__n_start_req_52，处理请求开始的状态
    s_n_llhttp__internal__n_start_req_52: {
      # 如果指针已经指向结束位置，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_start_req_52;
      }
      # 根据当前字符进行不同的跳转
      switch (*p) {
        # 如果当前字符为 'E'，则向前移动一个字符并跳转到状态 s_n_llhttp__internal__n_start_req_53
        case 'E': {
          p++;
          goto s_n_llhttp__internal__n_start_req_53;
        }
        # 如果当前字符为 'O'，则向前移动一个字符并跳转到状态 s_n_llhttp__internal__n_start_req_58
        case 'O': {
          p++;
          goto s_n_llhttp__internal__n_start_req_58;
        }
        # 如果当前字符为 'U'，则向前移动一个字符并跳转到状态 s_n_llhttp__internal__n_start_req_59
        case 'U': {
          p++;
          goto s_n_llhttp__internal__n_start_req_59;
        }
        # 如果当前字符不匹配任何情况，则跳转到状态 s_n_llhttp__internal__n_error_56
        default: {
          goto s_n_llhttp__internal__n_error_56;
        }
      }
      # 不可达的代码，中止程序
      /* UNREACHABLE */;
      abort();
    }
    # 定义状态 s_n_llhttp__internal__n_start_req_61
    case s_n_llhttp__internal__n_start_req_61:
    s_n_llhttp__internal__n_start_req_61: {
      # 定义匹配结果的结构体
      llparse_match_t match_seq;

      # 如果指针已经指向结束位置，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_start_req_61;
      }
      # 进行序列匹配，获取匹配结果
      match_seq = llparse__match_sequence_id(state, p, endp, llparse_blob52, 6);
      p = match_seq.current;
      # 根据匹配结果的状态进行不同的处理
      switch (match_seq.status) {
        # 如果匹配完成，则向前移动一个字符，设置匹配结果为 40，并跳转到状态 s_n_llhttp__internal__n_invoke_store_method_1
        case kMatchComplete: {
          p++;
          match = 40;
          goto s_n_llhttp__internal__n_invoke_store_method_1;
        }
        # 如果匹配暂停，则返回当前状态
        case kMatchPause: {
          return s_n_llhttp__internal__n_start_req_61;
        }
        # 如果匹配不成功，则跳转到状态 s_n_llhttp__internal__n_error_56
        case kMatchMismatch: {
          goto s_n_llhttp__internal__n_error_56;
        }
      }
      # 不可达的代码，中止程序
      /* UNREACHABLE */;
      abort();
    }
    # 定义状态 s_n_llhttp__internal__n_start_req_62
    case s_n_llhttp__internal__n_start_req_62:
    # 定义状态 s_n_llhttp__internal__n_start_req_62，表示解析 HTTP 请求开始的状态
    s_n_llhttp__internal__n_start_req_62: {
      # 定义匹配序列的变量
      llparse_match_t match_seq;

      # 如果指针已经到达结束位置，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_start_req_62;
      }
      # 调用 llparse__match_sequence_id 函数，匹配指定长度的序列
      match_seq = llparse__match_sequence_id(state, p, endp, llparse_blob53, 3);
      # 更新指针位置
      p = match_seq.current;
      # 根据匹配的状态进行处理
      switch (match_seq.status) {
        # 如果匹配完成
        case kMatchComplete: {
          # 指针后移一位
          p++;
          # 设置匹配值为 7
          match = 7;
          # 跳转到状态 s_n_llhttp__internal__n_invoke_store_method_1
          goto s_n_llhttp__internal__n_invoke_store_method_1;
        }
        # 如果匹配暂停
        case kMatchPause: {
          # 返回当前状态
          return s_n_llhttp__internal__n_start_req_62;
        }
        # 如果匹配不成功
        case kMatchMismatch: {
          # 跳转到状态 s_n_llhttp__internal__n_error_56
          goto s_n_llhttp__internal__n_error_56;
        }
      }
      # 不可达的代码，中断程序
      /* UNREACHABLE */;
      abort();
    }
    # 定义状态 s_n_llhttp__internal__n_start_req_60
    case s_n_llhttp__internal__n_start_req_60:
    s_n_llhttp__internal__n_start_req_60: {
      # 如果指针已经到达结束位置，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_start_req_60;
      }
      # 根据当前指针指向的字符进行不同的处理
      switch (*p) {
        # 如果是字符 'E'
        case 'E': {
          # 指针后移一位
          p++;
          # 跳转到状态 s_n_llhttp__internal__n_start_req_61
          goto s_n_llhttp__internal__n_start_req_61;
        }
        # 如果是字符 'R'
        case 'R': {
          # 指针后移一位
          p++;
          # 跳转到状态 s_n_llhttp__internal__n_start_req_62
          goto s_n_llhttp__internal__n_start_req_62;
        }
        # 其他情况
        default: {
          # 跳转到状态 s_n_llhttp__internal__n_error_56
          goto s_n_llhttp__internal__n_error_56;
        }
      }
      # 不可达的代码，中断程序
      /* UNREACHABLE */;
      abort();
    }
    # 定义状态 s_n_llhttp__internal__n_start_req_65
    case s_n_llhttp__internal__n_start_req_65:
    s_n_llhttp__internal__n_start_req_65: {
      # 定义匹配序列的变量
      llparse_match_t match_seq;

      # 如果指针已经到达结束位置，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_start_req_65;
      }
      # 调用 llparse__match_sequence_id 函数，匹配指定长度的序列
      match_seq = llparse__match_sequence_id(state, p, endp, llparse_blob54, 3);
      # 更新指针位置
      p = match_seq.current;
      # 根据匹配的状态进行处理
      switch (match_seq.status) {
        # 如果匹配完成
        case kMatchComplete: {
          # 指针后移一位
          p++;
          # 设置匹配值为 18
          match = 18;
          # 跳转到状态 s_n_llhttp__internal__n_invoke_store_method_1
          goto s_n_llhttp__internal__n_invoke_store_method_1;
        }
        # 如果匹配暂停
        case kMatchPause: {
          # 返回当前状态
          return s_n_llhttp__internal__n_start_req_65;
        }
        # 如果匹配不成功
        case kMatchMismatch: {
          # 跳转到状态 s_n_llhttp__internal__n_error_56
          goto s_n_llhttp__internal__n_error_56;
        }
      }
      # 不可达的代码，中断程序
      /* UNREACHABLE */;
      abort();
    }
    # 定义状态 s_n_llhttp__internal__n_start_req_67:
    # 定义状态 s_n_llhttp__internal__n_start_req_67，开始解析请求的状态
    s_n_llhttp__internal__n_start_req_67: {
      # 定义匹配序列的变量
      llparse_match_t match_seq;

      # 如果指针已经到达结束位置，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_start_req_67;
      }
      # 调用 llparse__match_sequence_id 函数，匹配指定的序列
      match_seq = llparse__match_sequence_id(state, p, endp, llparse_blob55, 2);
      # 更新指针位置
      p = match_seq.current;
      # 根据匹配的状态进行不同的处理
      switch (match_seq.status) {
        # 如果匹配完成
        case kMatchComplete: {
          # 更新指针位置
          p++;
          # 设置匹配值
          match = 32;
          # 跳转到状态 s_n_llhttp__internal__n_invoke_store_method_1
          goto s_n_llhttp__internal__n_invoke_store_method_1;
        }
        # 如果匹配暂停
        case kMatchPause: {
          # 返回当前状态
          return s_n_llhttp__internal__n_start_req_67;
        }
        # 如果匹配不成功
        case kMatchMismatch: {
          # 跳转到状态 s_n_llhttp__internal__n_error_56
          goto s_n_llhttp__internal__n_error_56;
        }
      }
      /* UNREACHABLE */;
      # 终止程序
      abort();
    }
    # 定义状态 s_n_llhttp__internal__n_start_req_68
    case s_n_llhttp__internal__n_start_req_68:
    s_n_llhttp__internal__n_start_req_68: {
      # 定义匹配序列的变量
      llparse_match_t match_seq;

      # 如果指针已经到达结束位置，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_start_req_68;
      }
      # 调用 llparse__match_sequence_id 函数，匹配指定的序列
      match_seq = llparse__match_sequence_id(state, p, endp, llparse_blob56, 2);
      # 更新指针位置
      p = match_seq.current;
      # 根据匹配的状态进行不同的处理
      switch (match_seq.status) {
        # 如果匹配完成
        case kMatchComplete: {
          # 更新指针位置
          p++;
          # 设置匹配值
          match = 15;
          # 跳转到状态 s_n_llhttp__internal__n_invoke_store_method_1
          goto s_n_llhttp__internal__n_invoke_store_method_1;
        }
        # 如果匹配暂停
        case kMatchPause: {
          # 返回当前状态
          return s_n_llhttp__internal__n_start_req_68;
        }
        # 如果匹配不成功
        case kMatchMismatch: {
          # 跳转到状态 s_n_llhttp__internal__n_error_56
          goto s_n_llhttp__internal__n_error_56;
        }
      }
      /* UNREACHABLE */;
      # 终止程序
      abort();
    }
    # 定义状态 s_n_llhttp__internal__n_start_req_66
    case s_n_llhttp__internal__n_start_req_66:
    s_n_llhttp__internal__n_start_req_66: {
      # 如果指针已经到达结束位置，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_start_req_66;
      }
      # 根据当前指针指向的字符进行不同的处理
      switch (*p) {
        # 如果是字符 'I'
        case 'I': {
          # 更新指针位置
          p++;
          # 跳转到状态 s_n_llhttp__internal__n_start_req_67
          goto s_n_llhttp__internal__n_start_req_67;
        }
        # 如果是字符 'O'
        case 'O': {
          # 更新指针位置
          p++;
          # 跳转到状态 s_n_llhttp__internal__n_start_req_68
          goto s_n_llhttp__internal__n_start_req_68;
        }
        # 如果是其它字符
        default: {
          # 跳转到状态 s_n_llhttp__internal__n_error_56
          goto s_n_llhttp__internal__n_error_56;
        }
      }
      /* UNREACHABLE */;
      # 终止程序
      abort();
    }
    # 定义状态 s_n_llhttp__internal__n_start_req_69:
    # 定义状态 s_n_llhttp__internal__n_start_req_69，用于解析 HTTP 请求的起始部分
    s_n_llhttp__internal__n_start_req_69: {
      # 定义匹配序列的结果变量
      llparse_match_t match_seq;

      # 如果指针已经到达结束位置，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_start_req_69;
      }
      # 调用 llparse__match_sequence_id 函数，匹配指定长度的字节序列
      match_seq = llparse__match_sequence_id(state, p, endp, llparse_blob57, 8);
      # 更新指针位置
      p = match_seq.current;
      # 根据匹配结果的状态进行不同的处理
      switch (match_seq.status) {
        # 如果匹配完成
        case kMatchComplete: {
          # 更新指针位置
          p++;
          # 设置匹配值
          match = 27;
          # 跳转到状态 s_n_llhttp__internal__n_invoke_store_method_1
          goto s_n_llhttp__internal__n_invoke_store_method_1;
        }
        # 如果匹配暂停
        case kMatchPause: {
          # 返回当前状态
          return s_n_llhttp__internal__n_start_req_69;
        }
        # 如果匹配不成功
        case kMatchMismatch: {
          # 跳转到状态 s_n_llhttp__internal__n_error_56
          goto s_n_llhttp__internal__n_error_56;
        }
      }
      /* UNREACHABLE */;
      # 终止程序执行
      abort();
    }
    # 定义状态 s_n_llhttp__internal__n_start_req_64
    case s_n_llhttp__internal__n_start_req_64:
    s_n_llhttp__internal__n_start_req_64: {
      # 如果指针已经到达结束位置，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_start_req_64;
      }
      # 根据当前指针指向的值进行不同的处理
      switch (*p) {
        # 如果当前值为 'B'
        case 'B': {
          # 更新指针位置
          p++;
          # 跳转到状态 s_n_llhttp__internal__n_start_req_65
          goto s_n_llhttp__internal__n_start_req_65;
        }
        # 如果当前值为 'L'
        case 'L': {
          # 更新指针位置
          p++;
          # 跳转到状态 s_n_llhttp__internal__n_start_req_66
          goto s_n_llhttp__internal__n_start_req_66;
        }
        # 如果当前值为 'S'
        case 'S': {
          # 更新指针位置
          p++;
          # 跳转到状态 s_n_llhttp__internal__n_start_req_69
          goto s_n_llhttp__internal__n_start_req_69;
        }
        # 如果当前值不匹配任何情况
        default: {
          # 跳转到状态 s_n_llhttp__internal__n_error_56
          goto s_n_llhttp__internal__n_error_56;
        }
      }
      /* UNREACHABLE */;
      # 终止程序执行
      abort();
    }
    # 定义状态 s_n_llhttp__internal__n_start_req_63
    case s_n_llhttp__internal__n_start_req_63:
    s_n_llhttp__internal__n_start_req_63: {
      # 如果指针已经到达结束位置，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_start_req_63;
      }
      # 根据当前指针指向的值进行不同的处理
      switch (*p) {
        # 如果当前值为 'N'
        case 'N': {
          # 更新指针位置
          p++;
          # 跳转到状态 s_n_llhttp__internal__n_start_req_64
          goto s_n_llhttp__internal__n_start_req_64;
        }
        # 如果当前值不匹配任何情况
        default: {
          # 跳转到状态 s_n_llhttp__internal__n_error_56
          goto s_n_llhttp__internal__n_error_56;
        }
      }
      /* UNREACHABLE */;
      # 终止程序执行
      abort();
    }
    # 定义状态 s_n_llhttp__internal__n_start_req
    # 定义状态机的状态 s_n_llhttp__internal__n_start_req，处理请求行的起始状态
    s_n_llhttp__internal__n_start_req: {
      # 如果指针已经指向输入的末尾，返回起始状态
      if (p == endp) {
        return s_n_llhttp__internal__n_start_req;
      }
      # 根据当前字符进行状态转移
      switch (*p) {
        # 如果当前字符是 'A'，指针向前移动一位，跳转到状态 s_n_llhttp__internal__n_start_req_1
        case 'A': {
          p++;
          goto s_n_llhttp__internal__n_start_req_1;
        }
        # 如果当前字符是 'B'，指针向前移动一位，跳转到状态 s_n_llhttp__internal__n_start_req_4
        case 'B': {
          p++;
          goto s_n_llhttp__internal__n_start_req_4;
        }
        # 如果当前字符是 'C'，指针向前移动一位，跳转到状态 s_n_llhttp__internal__n_start_req_5
        case 'C': {
          p++;
          goto s_n_llhttp__internal__n_start_req_5;
        }
        # 如果当前字符是 'D'，指针向前移动一位，跳转到状态 s_n_llhttp__internal__n_start_req_10
        case 'D': {
          p++;
          goto s_n_llhttp__internal__n_start_req_10;
        }
        # 如果当前字符是 'F'，指针向前移动一位，跳转到状态 s_n_llhttp__internal__n_start_req_14
        case 'F': {
          p++;
          goto s_n_llhttp__internal__n_start_req_14;
        }
        # 如果当前字符是 'G'，指针向前移动一位，跳转到状态 s_n_llhttp__internal__n_start_req_15
        case 'G': {
          p++;
          goto s_n_llhttp__internal__n_start_req_15;
        }
        # 如果当前字符是 'H'，指针向前移动一位，跳转到状态 s_n_llhttp__internal__n_start_req_18
        case 'H': {
          p++;
          goto s_n_llhttp__internal__n_start_req_18;
        }
        # 如果当前字符是 'L'，指针向前移动一位，跳转到状态 s_n_llhttp__internal__n_start_req_19
        case 'L': {
          p++;
          goto s_n_llhttp__internal__n_start_req_19;
        }
        # 如果当前字符是 'M'，指针向前移动一位，跳转到状态 s_n_llhttp__internal__n_start_req_22
        case 'M': {
          p++;
          goto s_n_llhttp__internal__n_start_req_22;
        }
        # 如果当前字符是 'N'，指针向前移动一位，跳转到状态 s_n_llhttp__internal__n_start_req_31
        case 'N': {
          p++;
          goto s_n_llhttp__internal__n_start_req_31;
        }
        # 如果当前字符是 'O'，指针向前移动一位，跳转到状态 s_n_llhttp__internal__n_start_req_32
        case 'O': {
          p++;
          goto s_n_llhttp__internal__n_start_req_32;
        }
        # 如果当前字符是 'P'，指针向前移动一位，跳转到状态 s_n_llhttp__internal__n_start_req_33
        case 'P': {
          p++;
          goto s_n_llhttp__internal__n_start_req_33;
        }
        # 如果当前字符是 'R'，指针向前移动一位，跳转到状态 s_n_llhttp__internal__n_start_req_46
        case 'R': {
          p++;
          goto s_n_llhttp__internal__n_start_req_46;
        }
        # 如果当前字符是 'S'，指针向前移动一位，跳转到状态 s_n_llhttp__internal__n_start_req_52
        case 'S': {
          p++;
          goto s_n_llhttp__internal__n_start_req_52;
        }
        # 如果当前字符是 'T'，指针向前移动一位，跳转到状态 s_n_llhttp__internal__n_start_req_60
        case 'T': {
          p++;
          goto s_n_llhttp__internal__n_start_req_60;
        }
        # 如果当前字符是 'U'，指针向前移动一位，跳转到状态 s_n_llhttp__internal__n_start_req_63
        case 'U': {
          p++;
          goto s_n_llhttp__internal__n_start_req_63;
        }
        # 如果当前字符不匹配任何情况，跳转到错误状态 s_n_llhttp__internal__n_error_56
        default: {
          goto s_n_llhttp__internal__n_error_56;
        }
      }
      /* UNREACHABLE */;
      # 终止程序
      abort();
    }
    # 处理状态 s_n_llhttp__internal__n_invoke_llhttp__on_status_complete
    case s_n_llhttp__internal__n_invoke_llhttp__on_status_complete:
    # 处理状态机状态 s_n_llhttp__internal__n_invoke_llhttp__on_status_complete
    s_n_llhttp__internal__n_invoke_llhttp__on_status_complete: {
      # 调用 llhttp__on_status_complete 处理状态，并根据返回值进行相应操作
      switch (llhttp__on_status_complete(state, p, endp)) {
        default:
          # 跳转到状态 s_n_llhttp__internal__n_header_field_start
          goto s_n_llhttp__internal__n_header_field_start;
      }
      /* UNREACHABLE */;
      # 终止程序执行
      abort();
    }
    # 处理状态机状态 s_n_llhttp__internal__n_res_line_almost_done
    case s_n_llhttp__internal__n_res_line_almost_done:
    s_n_llhttp__internal__n_res_line_almost_done: {
      # 如果指针 p 指向结束位置，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_res_line_almost_done;
      }
      # 根据指针指向的值进行不同操作
      switch (*p) {
        case 10: {
          # 如果指针指向的值为 10，则指针后移一位，跳转到状态 s_n_llhttp__internal__n_invoke_llhttp__on_status_complete
          p++;
          goto s_n_llhttp__internal__n_invoke_llhttp__on_status_complete;
        }
        default: {
          # 否则跳转到状态 s_n_llhttp__internal__n_error_49
          goto s_n_llhttp__internal__n_error_49;
        }
      }
      /* UNREACHABLE */;
      # 终止程序执行
      abort();
    }
    # 处理状态机状态 s_n_llhttp__internal__n_res_status
    case s_n_llhttp__internal__n_res_status:
    s_n_llhttp__internal__n_res_status: {
      # 如果指针 p 指向结束位置，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_res_status;
      }
      # 根据指针指向的值进行不同操作
      switch (*p) {
        case 10: {
          # 如果指针指向的值为 10，则跳转到状态 s_n_llhttp__internal__n_span_end_llhttp__on_status
          goto s_n_llhttp__internal__n_span_end_llhttp__on_status;
        }
        case 13: {
          # 如果指针指向的值为 13，则跳转到状态 s_n_llhttp__internal__n_span_end_llhttp__on_status_1
          goto s_n_llhttp__internal__n_span_end_llhttp__on_status_1;
        }
        default: {
          # 否则指针后移一位，继续保持当前状态
          p++;
          goto s_n_llhttp__internal__n_res_status;
        }
      }
      /* UNREACHABLE */;
      # 终止程序执行
      abort();
    }
    # 处理状态机状态 s_n_llhttp__internal__n_span_start_llhttp__on_status
    case s_n_llhttp__internal__n_span_start_llhttp__on_status: {
      # 如果指针 p 指向结束位置，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_span_start_llhttp__on_status;
      }
      # 记录当前指针位置和处理函数，然后跳转到状态 s_n_llhttp__internal__n_res_status
      state->_span_pos0 = (void*) p;
      state->_span_cb0 = llhttp__on_status;
      goto s_n_llhttp__internal__n_res_status;
      /* UNREACHABLE */;
      # 终止程序执行
      abort();
    }
    # 处理状态机状态 s_n_llhttp__internal__n_res_status_start:
    # 如果指针已经到达末尾，则返回当前状态
    s_n_llhttp__internal__n_res_status_start: {
      if (p == endp) {
        return s_n_llhttp__internal__n_res_status_start;
      }
      # 根据当前字符进行不同的处理
      switch (*p) {
        # 如果是换行符，则指针后移并跳转到状态 s_n_llhttp__internal__n_invoke_llhttp__on_status_complete
        case 10: {
          p++;
          goto s_n_llhttp__internal__n_invoke_llhttp__on_status_complete;
        }
        # 如果是回车符，则指针后移并跳转到状态 s_n_llhttp__internal__n_res_line_almost_done
        case 13: {
          p++;
          goto s_n_llhttp__internal__n_res_line_almost_done;
        }
        # 其他情况下，跳转到状态 s_n_llhttp__internal__n_span_start_llhttp__on_status
        default: {
          goto s_n_llhttp__internal__n_span_start_llhttp__on_status;
        }
      }
      # 不可达的代码，中止程序
      /* UNREACHABLE */;
      abort();
    }
    # 如果状态为 s_n_llhttp__internal__n_res_status_code_otherwise，则执行以下代码
    s_n_llhttp__internal__n_res_status_code_otherwise:
    s_n_llhttp__internal__n_res_status_code_otherwise: {
      # 如果指针已经到达末尾，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_res_status_code_otherwise;
      }
      # 根据当前字符进行不同的处理
      switch (*p) {
        # 如果是换行符，则跳转到状态 s_n_llhttp__internal__n_res_status_start
        case 10: {
          goto s_n_llhttp__internal__n_res_status_start;
        }
        # 如果是回车符，则跳转到状态 s_n_llhttp__internal__n_res_status_start
        case 13: {
          goto s_n_llhttp__internal__n_res_status_start;
        }
        # 如果是空格，则指针后移并跳转到状态 s_n_llhttp__internal__n_res_status_start
        case ' ': {
          p++;
          goto s_n_llhttp__internal__n_res_status_start;
        }
        # 其他情况下，跳转到状态 s_n_llhttp__internal__n_error_50
        default: {
          goto s_n_llhttp__internal__n_error_50;
        }
      }
      # 不可达的代码，中止程序
      /* UNREACHABLE */;
      abort();
    }
    # 如果状态为 s_n_llhttp__internal__n_res_status_code，则执行以下代码
    case s_n_llhttp__internal__n_res_status_code:
    # 定义状态机的状态 s_n_llhttp__internal__n_res_status_code
    s_n_llhttp__internal__n_res_status_code: {
      # 如果指针 p 已经指向结束位置，则返回状态 s_n_llhttp__internal__n_res_status_code
      if (p == endp) {
        return s_n_llhttp__internal__n_res_status_code;
      }
      # 根据指针指向的字符进行不同的处理
      switch (*p) {
        # 如果是字符 '0'，则将指针向后移动一位，设置匹配值为 0，跳转到状态 s_n_llhttp__internal__n_invoke_mul_add_status_code
        case '0': {
          p++;
          match = 0;
          goto s_n_llhttp__internal__n_invoke_mul_add_status_code;
        }
        # 如果是字符 '1'，则将指针向后移动一位，设置匹配值为 1，跳转到状态 s_n_llhttp__internal__n_invoke_mul_add_status_code
        case '1': {
          p++;
          match = 1;
          goto s_n_llhttp__internal__n_invoke_mul_add_status_code;
        }
        # 如果是字符 '2'，则将指针向后移动一位，设置匹配值为 2，跳转到状态 s_n_llhttp__internal__n_invoke_mul_add_status_code
        case '2': {
          p++;
          match = 2;
          goto s_n_llhttp__internal__n_invoke_mul_add_status_code;
        }
        # 如果是字符 '3'，则将指针向后移动一位，设置匹配值为 3，跳转到状态 s_n_llhttp__internal__n_invoke_mul_add_status_code
        case '3': {
          p++;
          match = 3;
          goto s_n_llhttp__internal__n_invoke_mul_add_status_code;
        }
        # 如果是字符 '4'，则将指针向后移动一位，设置匹配值为 4，跳转到状态 s_n_llhttp__internal__n_invoke_mul_add_status_code
        case '4': {
          p++;
          match = 4;
          goto s_n_llhttp__internal__n_invoke_mul_add_status_code;
        }
        # 如果是字符 '5'，则将指针向后移动一位，设置匹配值为 5，跳转到状态 s_n_llhttp__internal__n_invoke_mul_add_status_code
        case '5': {
          p++;
          match = 5;
          goto s_n_llhttp__internal__n_invoke_mul_add_status_code;
        }
        # 如果是字符 '6'，则将指针向后移动一位，设置匹配值为 6，跳转到状态 s_n_llhttp__internal__n_invoke_mul_add_status_code
        case '6': {
          p++;
          match = 6;
          goto s_n_llhttp__internal__n_invoke_mul_add_status_code;
        }
        # 如果是字符 '7'，则将指针向后移动一位，设置匹配值为 7，跳转到状态 s_n_llhttp__internal__n_invoke_mul_add_status_code
        case '7': {
          p++;
          match = 7;
          goto s_n_llhttp__internal__n_invoke_mul_add_status_code;
        }
        # 如果是字符 '8'，则将指针向后移动一位，设置匹配值为 8，跳转到状态 s_n_llhttp__internal__n_invoke_mul_add_status_code
        case '8': {
          p++;
          match = 8;
          goto s_n_llhttp__internal__n_invoke_mul_add_status_code;
        }
        # 如果是字符 '9'，则将指针向后移动一位，设置匹配值为 9，跳转到状态 s_n_llhttp__internal__n_invoke_mul_add_status_code
        case '9': {
          p++;
          match = 9;
          goto s_n_llhttp__internal__n_invoke_mul_add_status_code;
        }
        # 如果是其他字符，则跳转到状态 s_n_llhttp__internal__n_res_status_code_otherwise
        default: {
          goto s_n_llhttp__internal__n_res_status_code_otherwise;
        }
      }
      # 不可达的代码，中止程序
      /* UNREACHABLE */;
      abort();
    }
    # 其他状态 s_n_llhttp__internal__n_res_http_end:
    # 如果指针 p 等于结束指针 endp，则返回状态 s_n_llhttp__internal__n_res_http_end
    s_n_llhttp__internal__n_res_http_end: {
      if (p == endp) {
        return s_n_llhttp__internal__n_res_http_end;
      }
      # 根据指针 p 指向的字符进行判断
      switch (*p) {
        # 如果是空格，则指针 p 向后移动一位，跳转到状态 s_n_llhttp__internal__n_invoke_update_status_code
        case ' ': {
          p++;
          goto s_n_llhttp__internal__n_invoke_update_status_code;
        }
        # 如果不是空格，则跳转到状态 s_n_llhttp__internal__n_error_51
        default: {
          goto s_n_llhttp__internal__n_error_51;
        }
      }
      /* UNREACHABLE */;
      # 终止程序执行
      abort();
    }
    # 处理状态 s_n_llhttp__internal__n_res_http_minor
    case s_n_llhttp__internal__n_res_http_minor:
    # 检查 HTTP 响应的版本号中的次版本号
    s_n_llhttp__internal__n_res_http_minor: {
      # 如果指针已经到达末尾，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_res_http_minor;
      }
      # 根据当前字符进行不同的处理
      switch (*p) {
        # 如果是 '0'，则将匹配值设为 0，并跳转到存储 HTTP 次版本号的状态
        case '0': {
          p++;
          match = 0;
          goto s_n_llhttp__internal__n_invoke_store_http_minor_1;
        }
        # 如果是 '1'，则将匹配值设为 1，并跳转到存储 HTTP 次版本号的状态
        case '1': {
          p++;
          match = 1;
          goto s_n_llhttp__internal__n_invoke_store_http_minor_1;
        }
        # 如果是 '2'，则将匹配值设为 2，并跳转到存储 HTTP 次版本号的状态
        case '2': {
          p++;
          match = 2;
          goto s_n_llhttp__internal__n_invoke_store_http_minor_1;
        }
        # 如果是 '3'，则将匹配值设为 3，并跳转到存储 HTTP 次版本号的状态
        case '3': {
          p++;
          match = 3;
          goto s_n_llhttp__internal__n_invoke_store_http_minor_1;
        }
        # 如果是 '4'，则将匹配值设为 4，并跳转到存储 HTTP 次版本号的状态
        case '4': {
          p++;
          match = 4;
          goto s_n_llhttp__internal__n_invoke_store_http_minor_1;
        }
        # 如果是 '5'，则将匹配值设为 5，并跳转到存储 HTTP 次版本号的状态
        case '5': {
          p++;
          match = 5;
          goto s_n_llhttp__internal__n_invoke_store_http_minor_1;
        }
        # 如果是 '6'，则将匹配值设为 6，并跳转到存储 HTTP 次版本号的状态
        case '6': {
          p++;
          match = 6;
          goto s_n_llhttp__internal__n_invoke_store_http_minor_1;
        }
        # 如果是 '7'，则将匹配值设为 7，并跳转到存储 HTTP 次版本号的状态
        case '7': {
          p++;
          match = 7;
          goto s_n_llhttp__internal__n_invoke_store_http_minor_1;
        }
        # 如果是 '8'，则将匹配值设为 8，并跳转到存储 HTTP 次版本号的状态
        case '8': {
          p++;
          match = 8;
          goto s_n_llhttp__internal__n_invoke_store_http_minor_1;
        }
        # 如果是 '9'，则将匹配值设为 9，并跳转到存储 HTTP 次版本号的状态
        case '9': {
          p++;
          match = 9;
          goto s_n_llhttp__internal__n_invoke_store_http_minor_1;
        }
        # 如果是其他字符，则跳转到错误状态
        default: {
          goto s_n_llhttp__internal__n_error_52;
        }
      }
      /* UNREACHABLE */;
      # 终止程序
      abort();
    }
    # 处理 HTTP 响应版本号中的小数点
    case s_n_llhttp__internal__n_res_http_dot:
    # 如果指针已经指向末尾，则返回当前状态
    if (p == endp) {
        return s_n_llhttp__internal__n_res_http_dot;
    }
    # 根据当前字符进行不同的处理
    switch (*p) {
        # 如果是'.'，则指针向后移动一位，进入下一个状态
        case '.': {
            p++;
            goto s_n_llhttp__internal__n_res_http_minor;
        }
        # 如果不是'.'，则进入错误状态
        default: {
            goto s_n_llhttp__internal__n_error_53;
        }
    }
    # 不可达的代码，中止程序
    /* UNREACHABLE */;
    abort();
    # 进入下一个状态
    case s_n_llhttp__internal__n_res_http_major:
    # 检查解析 HTTP 响应的主版本号
    s_n_llhttp__internal__n_res_http_major: {
      # 如果指针已经到达结束位置，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_res_http_major;
      }
      # 根据当前字符进行不同的处理
      switch (*p) {
        # 如果是 '0'，则匹配成功，跳转到存储 HTTP 主版本号的状态
        case '0': {
          p++;
          match = 0;
          goto s_n_llhttp__internal__n_invoke_store_http_major_1;
        }
        # 如果是 '1'，则匹配成功，跳转到存储 HTTP 主版本号的状态
        case '1': {
          p++;
          match = 1;
          goto s_n_llhttp__internal__n_invoke_store_http_major_1;
        }
        # 如果是 '2'，则匹配成功，跳转到存储 HTTP 主版本号的状态
        case '2': {
          p++;
          match = 2;
          goto s_n_llhttp__internal__n_invoke_store_http_major_1;
        }
        # 如果是 '3'，则匹配成功，跳转到存储 HTTP 主版本号的状态
        case '3': {
          p++;
          match = 3;
          goto s_n_llhttp__internal__n_invoke_store_http_major_1;
        }
        # 如果是 '4'，则匹配成功，跳转到存储 HTTP 主版本号的状态
        case '4': {
          p++;
          match = 4;
          goto s_n_llhttp__internal__n_invoke_store_http_major_1;
        }
        # 如果是 '5'，则匹配成功，跳转到存储 HTTP 主版本号的状态
        case '5': {
          p++;
          match = 5;
          goto s_n_llhttp__internal__n_invoke_store_http_major_1;
        }
        # 如果是 '6'，则匹配成功，跳转到存储 HTTP 主版本号的状态
        case '6': {
          p++;
          match = 6;
          goto s_n_llhttp__internal__n_invoke_store_http_major_1;
        }
        # 如果是 '7'，则匹配成功，跳转到存储 HTTP 主版本号的状态
        case '7': {
          p++;
          match = 7;
          goto s_n_llhttp__internal__n_invoke_store_http_major_1;
        }
        # 如果是 '8'，则匹配成功，跳转到存储 HTTP 主版本号的状态
        case '8': {
          p++;
          match = 8;
          goto s_n_llhttp__internal__n_invoke_store_http_major_1;
        }
        # 如果是 '9'，则匹配成功，跳转到存储 HTTP 主版本号的状态
        case '9': {
          p++;
          match = 9;
          goto s_n_llhttp__internal__n_invoke_store_http_major_1;
        }
        # 如果是其他字符，则跳转到错误状态
        default: {
          goto s_n_llhttp__internal__n_error_54;
        }
      }
      /* UNREACHABLE */;
      # 终止程序
      abort();
    }
    # 解析 HTTP 响应的起始状态
    case s_n_llhttp__internal__n_start_res:
    # 定义状态 s_n_llhttp__internal__n_start_res，表示解析 HTTP 响应的起始状态
    s_n_llhttp__internal__n_start_res: {
      # 定义变量 match_seq，用于存储匹配结果
      llparse_match_t match_seq;

      # 如果指针已经指向输入的结尾，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_start_res;
      }
      # 调用 llparse__match_sequence_id 函数，匹配输入数据和指定的序列 llparse_blob58，长度为 5
      match_seq = llparse__match_sequence_id(state, p, endp, llparse_blob58, 5);
      # 更新指针位置
      p = match_seq.current;
      # 根据匹配结果的状态进行不同的处理
      switch (match_seq.status) {
        # 如果匹配完成，则指针后移一位，跳转到解析 HTTP 响应的主版本号状态
        case kMatchComplete: {
          p++;
          goto s_n_llhttp__internal__n_res_http_major;
        }
        # 如果匹配暂停，则返回当前状态
        case kMatchPause: {
          return s_n_llhttp__internal__n_start_res;
        }
        # 如果匹配不成功，则跳转到错误状态 s_n_llhttp__internal__n_error_57
        case kMatchMismatch: {
          goto s_n_llhttp__internal__n_error_57;
        }
      }
      /* UNREACHABLE */;
      # 如果执行到这里，表示代码逻辑错误，终止程序
      abort();
    }
    # 定义状态 s_n_llhttp__internal__n_req_or_res_method_2
    case s_n_llhttp__internal__n_req_or_res_method_2:
    s_n_llhttp__internal__n_req_or_res_method_2: {
      # 定义变量 match_seq，用于存储匹配结果
      llparse_match_t match_seq;

      # 如果指针已经指向输入的结尾，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_req_or_res_method_2;
      }
      # 调用 llparse__match_sequence_id 函数，匹配输入数据和指定的序列 llparse_blob59，长度为 2
      match_seq = llparse__match_sequence_id(state, p, endp, llparse_blob59, 2);
      # 更新指针位置
      p = match_seq.current;
      # 根据匹配结果的状态进行不同的处理
      switch (match_seq.status) {
        # 如果匹配完成，则指针后移一位，设置 match 变量为 2，跳转到存储方法的状态
        case kMatchComplete: {
          p++;
          match = 2;
          goto s_n_llhttp__internal__n_invoke_store_method;
        }
        # 如果匹配暂停，则返回当前状态
        case kMatchPause: {
          return s_n_llhttp__internal__n_req_or_res_method_2;
        }
        # 如果匹配不成功，则跳转到错误状态 s_n_llhttp__internal__n_error_55
        case kMatchMismatch: {
          goto s_n_llhttp__internal__n_error_55;
        }
      }
      /* UNREACHABLE */;
      # 如果执行到这里，表示代码逻辑错误，终止程序
      abort();
    }
    # 定义状态 s_n_llhttp__internal__n_req_or_res_method_3:
    # 定义一个名为 s_n_llhttp__internal__n_req_or_res_method_3 的状态
    s_n_llhttp__internal__n_req_or_res_method_3: {
      # 定义一个名为 match_seq 的变量，用于存储匹配结果
      llparse_match_t match_seq;

      # 如果指针 p 已经指向输入的结尾，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_req_or_res_method_3;
      }
      # 调用 llparse__match_sequence_id 函数进行匹配，返回匹配结果
      match_seq = llparse__match_sequence_id(state, p, endp, llparse_blob60, 3);
      # 更新指针 p 的位置
      p = match_seq.current;
      # 根据匹配结果的状态进行不同的处理
      switch (match_seq.status) {
        # 如果匹配完成，则指针 p 向后移动一位，跳转到状态 s_n_llhttp__internal__n_invoke_update_type_1
        case kMatchComplete: {
          p++;
          goto s_n_llhttp__internal__n_invoke_update_type_1;
        }
        # 如果匹配暂停，则返回当前状态
        case kMatchPause: {
          return s_n_llhttp__internal__n_req_or_res_method_3;
        }
        # 如果匹配不成功，则跳转到状态 s_n_llhttp__internal__n_error_55
        case kMatchMismatch: {
          goto s_n_llhttp__internal__n_error_55;
        }
      }
      /* UNREACHABLE */;
      # 终止程序执行
      abort();
    }
    # 定义一个名为 s_n_llhttp__internal__n_req_or_res_method_1 的状态
    case s_n_llhttp__internal__n_req_or_res_method_1:
    s_n_llhttp__internal__n_req_or_res_method_1: {
      # 如果指针 p 已经指向输入的结尾，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_req_or_res_method_1;
      }
      # 根据指针 p 指向的字符进行不同的处理
      switch (*p) {
        # 如果字符为 'E'，则指针 p 向后移动一位，跳转到状态 s_n_llhttp__internal__n_req_or_res_method_2
        case 'E': {
          p++;
          goto s_n_llhttp__internal__n_req_or_res_method_2;
        }
        # 如果字符为 'T'，则指针 p 向后移动一位，跳转到状态 s_n_llhttp__internal__n_req_or_res_method_3
        case 'T': {
          p++;
          goto s_n_llhttp__internal__n_req_or_res_method_3;
        }
        # 如果字符不符合预期，则跳转到状态 s_n_llhttp__internal__n_error_55
        default: {
          goto s_n_llhttp__internal__n_error_55;
        }
      }
      /* UNREACHABLE */;
      # 终止程序执行
      abort();
    }
    # 定义一个名为 s_n_llhttp__internal__n_req_or_res_method 的状态
    case s_n_llhttp__internal__n_req_or_res_method:
    s_n_llhttp__internal__n_req_or_res_method: {
      # 如果指针 p 已经指向输入的结尾，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_req_or_res_method;
      }
      # 根据指针 p 指向的字符进行不同的处理
      switch (*p) {
        # 如果字符为 'H'，则指针 p 向后移动一位，跳转到状态 s_n_llhttp__internal__n_req_or_res_method_1
        case 'H': {
          p++;
          goto s_n_llhttp__internal__n_req_or_res_method_1;
        }
        # 如果字符不符合预期，则跳转到状态 s_n_llhttp__internal__n_error_55
        default: {
          goto s_n_llhttp__internal__n_error_55;
        }
      }
      /* UNREACHABLE */;
      # 终止程序执行
      abort();
    }
    # 定义一个名为 s_n_llhttp__internal__n_start_req_or_res 的状态
    case s_n_llhttp__internal__n_start_req_or_res:
    # 定义状态机的状态 s_n_llhttp__internal__n_start_req_or_res
    s_n_llhttp__internal__n_start_req_or_res: {
      # 如果指针 p 指向结束位置，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_start_req_or_res;
      }
      # 根据指针指向的字符进行不同的处理
      switch (*p) {
        # 如果指针指向字符 'H'，则跳转到状态 s_n_llhttp__internal__n_req_or_res_method
        case 'H': {
          goto s_n_llhttp__internal__n_req_or_res_method;
        }
        # 如果指针指向其他字符，则跳转到状态 s_n_llhttp__internal__n_invoke_update_type_2
        default: {
          goto s_n_llhttp__internal__n_invoke_update_type_2;
        }
      }
      # 不可达的代码，中止程序
      /* UNREACHABLE */;
      abort();
    }
    # 定义状态机的状态 s_n_llhttp__internal__n_invoke_load_type
    case s_n_llhttp__internal__n_invoke_load_type:
    s_n_llhttp__internal__n_invoke_load_type: {
      # 根据调用 llhttp__internal__c_load_type 函数的返回值进行不同的处理
      switch (llhttp__internal__c_load_type(state, p, endp)) {
        # 如果返回值为 1，则跳转到状态 s_n_llhttp__internal__n_start_req
        case 1:
          goto s_n_llhttp__internal__n_start_req;
        # 如果返回值为 2，则跳转到状态 s_n_llhttp__internal__n_start_res
        case 2:
          goto s_n_llhttp__internal__n_start_res;
        # 如果返回值为其他值，则跳转到状态 s_n_llhttp__internal__n_start_req_or_res
        default:
          goto s_n_llhttp__internal__n_start_req_or_res;
      }
      # 不可达的代码，中止程序
      /* UNREACHABLE */;
      abort();
    }
    # 定义状态机的状态 s_n_llhttp__internal__n_start
    case s_n_llhttp__internal__n_start:
    s_n_llhttp__internal__n_start: {
      # 如果指针 p 指向结束位置，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_start;
      }
      # 根据指针指向的字符进行不同的处理
      switch (*p) {
        # 如果指针指向换行符，则将指针向后移动一位，然后跳转到状态 s_n_llhttp__internal__n_start
        case 10: {
          p++;
          goto s_n_llhttp__internal__n_start;
        }
        # 如果指针指向回车符，则将指针向后移动一位，然后跳转到状态 s_n_llhttp__internal__n_start
        case 13: {
          p++;
          goto s_n_llhttp__internal__n_start;
        }
        # 如果指针指向其他字符，则跳转到状态 s_n_llhttp__internal__n_invoke_update_finish
        default: {
          goto s_n_llhttp__internal__n_invoke_update_finish;
        }
      }
      # 不可达的代码，中止程序
      /* UNREACHABLE */;
      abort();
    }
    # 默认情况下，中止程序
    default:
      /* UNREACHABLE */
      abort();
  }
  # 定义状态机的状态 s_n_llhttp__internal__n_error_1
  s_n_llhttp__internal__n_error_1: {
    # 设置状态机的错误码和错误原因
    state->error = 0x7;
    state->reason = "Invalid characters in url (strict mode)";
    state->error_pos = (const char*) p;
    state->_current = (void*) (intptr_t) s_error;
    return s_error;
    # 不可达的代码，中止程序
    /* UNREACHABLE */;
    abort();
  }
  # 定义状态机的状态 s_n_llhttp__internal__n_error_43
  s_n_llhttp__internal__n_error_43: {
    # 设置状态机的错误码和错误原因
    state->error = 0x7;
    state->reason = "Invalid characters in url";
    state->error_pos = (const char*) p;
    state->_current = (void*) (intptr_t) s_error;
    return s_error;
    # 不可达的代码，中止程序
    /* UNREACHABLE */;
    abort();
  }
  # 定义状态机的状态 s_n_llhttp__internal__n_invoke_update_finish_2
  switch (llhttp__internal__c_update_finish_1(state, p, endp)) {
    default:
      // 转到起始状态
      goto s_n_llhttp__internal__n_start;
  }
  /* UNREACHABLE */;
  // 终止程序
  abort();
}
s_n_llhttp__internal__n_error_4: {
  state->error = 0x5;
  state->reason = "Data after `Connection: close`";
  state->error_pos = (const char*) p;
  state->_current = (void*) (intptr_t) s_error;
  return s_error;
  /* UNREACHABLE */;
  // 终止程序
  abort();
}
s_n_llhttp__internal__n_invoke_test_lenient_flags: {
  switch (llhttp__internal__c_test_lenient_flags(state, p, endp)) {
    case 1:
      // 转到更新完成状态
      goto s_n_llhttp__internal__n_invoke_update_finish_2;
    default:
      // 转到关闭状态
      goto s_n_llhttp__internal__n_closed;
  }
  /* UNREACHABLE */;
  // 终止程序
  abort();
}
s_n_llhttp__internal__n_invoke_update_finish_1: {
  switch (llhttp__internal__c_update_finish_1(state, p, endp)) {
    default:
      // 转到测试宽松标志状态
      goto s_n_llhttp__internal__n_invoke_test_lenient_flags;
  }
  /* UNREACHABLE */;
  // 终止程序
  abort();
}
s_n_llhttp__internal__n_pause_5: {
  state->error = 0x15;
  state->reason = "on_message_complete pause";
  state->error_pos = (const char*) p;
  state->_current = (void*) (intptr_t) s_n_llhttp__internal__n_invoke_is_equal_upgrade;
  return s_error;
  /* UNREACHABLE */;
  // 终止程序
  abort();
}
s_n_llhttp__internal__n_error_13: {
  state->error = 0x12;
  state->reason = "`on_message_complete` callback error";
  state->error_pos = (const char*) p;
  state->_current = (void*) (intptr_t) s_error;
  return s_error;
  /* UNREACHABLE */;
  // 终止程序
  abort();
}
s_n_llhttp__internal__n_pause_7: {
  state->error = 0x15;
  state->reason = "on_chunk_complete pause";
  state->error_pos = (const char*) p;
  state->_current = (void*) (intptr_t) s_n_llhttp__internal__n_invoke_llhttp__on_message_complete_2;
  return s_error;
  /* UNREACHABLE */;
  // 终止程序
  abort();
}
s_n_llhttp__internal__n_error_16: {
  state->error = 0x14;
  state->reason = "`on_chunk_complete` callback error";
    state->error_pos = (const char*) p;  # 设置状态对象的错误位置为当前位置
    state->_current = (void*) (intptr_t) s_error;  # 设置状态对象的当前状态为错误状态
    return s_error;  # 返回错误状态
    /* UNREACHABLE */;  # 不可达代码
    abort();  # 终止程序
  }
  s_n_llhttp__internal__n_invoke_llhttp__on_chunk_complete_1: {  # 标签
    switch (llhttp__on_chunk_complete(state, p, endp)) {  # 调用函数并根据返回值进行切换
      case 0:  # 如果返回值为0
        goto s_n_llhttp__internal__n_invoke_llhttp__on_message_complete_2;  # 跳转到指定标签
      case 21:  # 如果返回值为21
        goto s_n_llhttp__internal__n_pause_7;  # 跳转到指定标签
      default:  # 其他情况
        goto s_n_llhttp__internal__n_error_16;  # 跳转到指定标签
    }
    /* UNREACHABLE */;  # 不可达代码
    abort();  # 终止程序
  }
  s_n_llhttp__internal__n_error_15: {  # 标签
    state->error = 0x4;  # 设置状态对象的错误码
    state->reason = "Content-Length can't be present with Transfer-Encoding";  # 设置状态对象的错误原因
    state->error_pos = (const char*) p;  # 设置状态对象的错误位置为当前位置
    state->_current = (void*) (intptr_t) s_error;  # 设置状态对象的当前状态为错误状态
    return s_error;  # 返回错误状态
    /* UNREACHABLE */;  # 不可达代码
    abort();  # 终止程序
  }
  s_n_llhttp__internal__n_pause_2: {  # 标签
    state->error = 0x15;  # 设置状态对象的错误码
    state->reason = "on_message_complete pause";  # 设置状态对象的错误原因
    state->error_pos = (const char*) p;  # 设置状态对象的错误位置为当前位置
    state->_current = (void*) (intptr_t) s_n_llhttp__internal__n_pause_1;  # 设置状态对象的当前状态为暂停状态
    return s_error;  # 返回错误状态
    /* UNREACHABLE */;  # 不可达代码
    abort();  # 终止程序
  }
  s_n_llhttp__internal__n_error_5: {  # 标签
    state->error = 0x12;  # 设置状态对象的错误码
    state->reason = "`on_message_complete` callback error";  # 设置状态对象的错误原因
    state->error_pos = (const char*) p;  # 设置状态对象的错误位置为当前位置
    state->_current = (void*) (intptr_t) s_error;  # 设置状态对象的当前状态为错误状态
    return s_error;  # 返回错误状态
    /* UNREACHABLE */;  # 不可达代码
    abort();  # 终止程序
  }
  s_n_llhttp__internal__n_invoke_llhttp__on_message_complete_1: {  # 标签
    switch (llhttp__on_message_complete(state, p, endp)) {  # 调用函数并根据返回值进行切换
      case 0:  # 如果返回值为0
        goto s_n_llhttp__internal__n_pause_1;  # 跳转到指定标签
      case 21:  # 如果返回值为21
        goto s_n_llhttp__internal__n_pause_2;  # 跳转到指定标签
      default:  # 其他情况
        goto s_n_llhttp__internal__n_error_5;  # 跳转到指定标签
    }
    /* UNREACHABLE */;  # 不可达代码
    abort();  # 终止程序
  }
  s_n_llhttp__internal__n_error_11: {  # 标签
    state->error = 0xc;  # 设置状态对象的错误码
    state->reason = "Chunk size overflow";  # 设置状态对象的错误原因
    state->error_pos = (const char*) p;  # 设置状态对象的错误位置为当前位置
    state->_current = (void*) (intptr_t) s_error;  # 设置状态对象的当前状态为错误状态
    return s_error;  # 返回错误状态
    /* UNREACHABLE */;  # 不可达代码
    abort();
  }
  s_n_llhttp__internal__n_pause_3: {
    // 设置状态的错误码为0x15
    state->error = 0x15;
    // 设置状态的原因为"on_chunk_complete pause"
    state->reason = "on_chunk_complete pause";
    // 设置状态的错误位置为p
    state->error_pos = (const char*) p;
    // 设置状态的_current为s_n_llhttp__internal__n_invoke_update_content_length的地址
    state->_current = (void*) (intptr_t) s_n_llhttp__internal__n_invoke_update_content_length;
    // 返回错误状态
    return s_error;
    /* UNREACHABLE */;
    abort();
  }
  s_n_llhttp__internal__n_error_7: {
    // 设置状态的错误码为0x14
    state->error = 0x14;
    // 设置状态的原因为"`on_chunk_complete` callback error"
    state->reason = "`on_chunk_complete` callback error";
    // 设置状态的错误位置为p
    state->error_pos = (const char*) p;
    // 设置状态的_current为s_error的地址
    state->_current = (void*) (intptr_t) s_error;
    // 返回错误状态
    return s_error;
    /* UNREACHABLE */;
    abort();
  }
  s_n_llhttp__internal__n_invoke_llhttp__on_chunk_complete: {
    // 调用llhttp__on_chunk_complete函数，并根据返回值进行不同的跳转
    switch (llhttp__on_chunk_complete(state, p, endp)) {
      case 0:
        // 跳转到s_n_llhttp__internal__n_invoke_update_content_length
        goto s_n_llhttp__internal__n_invoke_update_content_length;
      case 21:
        // 跳转到s_n_llhttp__internal__n_pause_3
        goto s_n_llhttp__internal__n_pause_3;
      default:
        // 跳转到s_n_llhttp__internal__n_error_7
        goto s_n_llhttp__internal__n_error_7;
    }
    /* UNREACHABLE */;
    abort();
  }
  s_n_llhttp__internal__n_error_8: {
    // 设置状态的错误码为0x2
    state->error = 0x2;
    // 设置状态的原因为"Expected CRLF after chunk"
    state->reason = "Expected CRLF after chunk";
    // 设置状态的错误位置为p
    state->error_pos = (const char*) p;
    // 设置状态的_current为s_error的地址
    state->_current = (void*) (intptr_t) s_error;
    // 返回错误状态
    return s_error;
    /* UNREACHABLE */;
    abort();
  }
  s_n_llhttp__internal__n_span_end_llhttp__on_body: {
    // 定义变量start和err
    const unsigned char* start;
    int err;

    // 将_span_pos0赋值给start，并将_span_pos0置为NULL
    start = state->_span_pos0;
    state->_span_pos0 = NULL;
    // 调用llhttp__on_body函数，并根据返回值进行不同的处理
    err = llhttp__on_body(state, start, p);
    if (err != 0) {
      // 如果返回值不为0，设置状态的错误码为err
      state->error = err;
      // 设置状态的错误位置为p
      state->error_pos = (const char*) p;
      // 设置状态的_current为s_n_llhttp__internal__n_chunk_data_almost_done的地址
      state->_current = (void*) (intptr_t) s_n_llhttp__internal__n_chunk_data_almost_done;
      // 返回错误状态
      return s_error;
    }
    // 跳转到s_n_llhttp__internal__n_chunk_data_almost_done
    goto s_n_llhttp__internal__n_chunk_data_almost_done;
    /* UNREACHABLE */;
    abort();
  }
  s_n_llhttp__internal__n_invoke_or_flags: {
    // 调用llhttp__internal__c_or_flags函数，并根据返回值进行不同的跳转
    switch (llhttp__internal__c_or_flags(state, p, endp)) {
      default:
        // 默认情况下跳转到s_n_llhttp__internal__n_header_field_start
        goto s_n_llhttp__internal__n_header_field_start;
    }
    /* UNREACHABLE */;
    abort();
  }
  s_n_llhttp__internal__n_pause_4: {
    state->error = 0x15;  # 设置状态对象的错误码为 0x15
    state->reason = "on_chunk_header pause";  # 设置状态对象的错误原因为 "on_chunk_header pause"
    state->error_pos = (const char*) p;  # 设置状态对象的错误位置为 p 的地址
    state->_current = (void*) (intptr_t) s_n_llhttp__internal__n_invoke_is_equal_content_length;  # 设置状态对象的当前状态为 s_n_llhttp__internal__n_invoke_is_equal_content_length
    return s_error;  # 返回错误状态
    /* UNREACHABLE */;  # 不可达代码
    abort();  # 终止程序执行
  }
  s_n_llhttp__internal__n_error_6: {
    state->error = 0x13;  # 设置状态对象的错误码为 0x13
    state->reason = "`on_chunk_header` callback error";  # 设置状态对象的错误原因为 "`on_chunk_header` callback error"
    state->error_pos = (const char*) p;  # 设置状态对象的错误位置为 p 的地址
    state->_current = (void*) (intptr_t) s_error;  # 设置状态对象的当前状态为 s_error
    return s_error;  # 返回错误状态
    /* UNREACHABLE */;  # 不可达代码
    abort();  # 终止程序执行
  }
  s_n_llhttp__internal__n_invoke_llhttp__on_chunk_header: {
    switch (llhttp__on_chunk_header(state, p, endp)) {  # 调用 llhttp__on_chunk_header 函数，根据返回值进行不同的处理
      case 0:  # 如果返回值为 0
        goto s_n_llhttp__internal__n_invoke_is_equal_content_length;  # 跳转到标签 s_n_llhttp__internal__n_invoke_is_equal_content_length
      case 21:  # 如果返回值为 21
        goto s_n_llhttp__internal__n_pause_4;  # 跳转到标签 s_n_llhttp__internal__n_pause_4
      default:  # 其他情况
        goto s_n_llhttp__internal__n_error_6;  # 跳转到标签 s_n_llhttp__internal__n_error_6
    }
    /* UNREACHABLE */;  # 不可达代码
    abort();  # 终止程序执行
  }
  s_n_llhttp__internal__n_error_9: {
    state->error = 0x2;  # 设置状态对象的错误码为 0x2
    state->reason = "Expected LF after chunk size";  # 设置状态对象的错误原因为 "Expected LF after chunk size"
    state->error_pos = (const char*) p;  # 设置状态对象的错误位置为 p 的地址
    state->_current = (void*) (intptr_t) s_error;  # 设置状态对象的当前状态为 s_error
    return s_error;  # 返回错误状态
    /* UNREACHABLE */;  # 不可达代码
    abort();  # 终止程序执行
  }
  s_n_llhttp__internal__n_error_10: {
    state->error = 0xc;  # 设置状态对象的错误码为 0xc
    state->reason = "Invalid character in chunk size";  # 设置状态对象的错误原因为 "Invalid character in chunk size"
    state->error_pos = (const char*) p;  # 设置状态对象的错误位置为 p 的地址
    state->_current = (void*) (intptr_t) s_error;  # 设置状态对象的当前状态为 s_error
    return s_error;  # 返回错误状态
    /* UNREACHABLE */;  # 不可达代码
    abort();  # 终止程序执行
  }
  s_n_llhttp__internal__n_invoke_mul_add_content_length: {
    switch (llhttp__internal__c_mul_add_content_length(state, p, endp, match)) {  # 调用 llhttp__internal__c_mul_add_content_length 函数，根据返回值进行不同的处理
      case 1:  # 如果返回值为 1
        goto s_n_llhttp__internal__n_error_11;  # 跳转到标签 s_n_llhttp__internal__n_error_11
      default:  # 其他情况
        goto s_n_llhttp__internal__n_chunk_size;  # 跳转到标签 s_n_llhttp__internal__n_chunk_size
    }
    /* UNREACHABLE */;  # 不可达代码
    abort();  # 终止程序执行
  }
  s_n_llhttp__internal__n_error_12: {
    state->error = 0xc;  # 设置状态对象的错误码为 0xc
    state->reason = "Invalid character in chunk size";  # 设置状态对象的错误原因为 "Invalid character in chunk size"
    state->error_pos = (const char*) p;  # 设置状态对象的错误位置为 p 的地址
    state->_current = (void*) (intptr_t) s_error;  # 设置状态对象的当前状态为 s_error
    return s_error;  # 返回错误状态
    /* UNREACHABLE */;  # 不可达代码
    abort();  # 终止程序执行
  // 中止程序
  abort();
  // 设置状态为解析 body 的起始位置
  s_n_llhttp__internal__n_span_end_llhttp__on_body_1: {
    // 定义起始位置和错误码
    const unsigned char* start;
    int err;

    // 获得起始位置
    start = state->_span_pos0;
    state->_span_pos0 = NULL;
    // 调用 llhttp__on_body 函数解析 body，如果出错则设置错误码并跳转到错误状态
    err = llhttp__on_body(state, start, p);
    if (err != 0) {
      state->error = err;
      state->error_pos = (const char*) p;
      state->_current = (void*) (intptr_t) s_n_llhttp__internal__n_invoke_llhttp__on_message_complete_2;
      return s_error;
    }
    // 跳转到解析 message 完成状态
    goto s_n_llhttp__internal__n_invoke_llhttp__on_message_complete_2;
    /* UNREACHABLE */;
    // 中止程序
    abort();
  }
  // 调用 llhttp__internal__c_update_finish_3 函数更新状态
  s_n_llhttp__internal__n_invoke_update_finish_3: {
    switch (llhttp__internal__c_update_finish_3(state, p, endp)) {
      default:
        // 跳转到解析 body 起始状态
        goto s_n_llhttp__internal__n_span_start_llhttp__on_body_2;
    }
    /* UNREACHABLE */;
    // 中止程序
    abort();
  }
  // 设置错误状态和错误信息
  s_n_llhttp__internal__n_error_14: {
    state->error = 0xf;
    state->reason = "Request has invalid `Transfer-Encoding`";
    state->error_pos = (const char*) p;
    state->_current = (void*) (intptr_t) s_error;
    return s_error;
    /* UNREACHABLE */;
    // 中止程序
    abort();
  }
  // 设置错误状态和错误信息
  s_n_llhttp__internal__n_pause: {
    state->error = 0x15;
    state->reason = "on_message_complete pause";
    state->error_pos = (const char*) p;
    state->_current = (void*) (intptr_t) s_n_llhttp__internal__n_invoke_llhttp__after_message_complete;
    return s_error;
    /* UNREACHABLE */;
    // 中止程序
    abort();
  }
  // 设置错误状态和错误信息
  s_n_llhttp__internal__n_error_3: {
    state->error = 0x12;
    state->reason = "`on_message_complete` callback error";
    state->error_pos = (const char*) p;
    state->_current = (void*) (intptr_t) s_error;
    return s_error;
    /* UNREACHABLE */;
    // 中止程序
    abort();
  }
  // 调用 llhttp__on_message_complete 函数解析 message 完成状态
  s_n_llhttp__internal__n_invoke_llhttp__on_message_complete: {
    switch (llhttp__on_message_complete(state, p, endp)) {
      case 0:
        // 跳转到解析 message 完成后状态
        goto s_n_llhttp__internal__n_invoke_llhttp__after_message_complete;
      case 21:
        // 跳转到暂停状态
        goto s_n_llhttp__internal__n_pause;
      default:
        // 跳转到错误状态
        goto s_n_llhttp__internal__n_error_3;
  }
  /* UNREACHABLE */;
  // 如果程序执行到这里，表示代码存在逻辑错误，因为前面的 case 都没有匹配到
  abort();
}
s_n_llhttp__internal__n_invoke_or_flags_1: {
  switch (llhttp__internal__c_or_flags_1(state, p, endp)) {
    default:
      // 转到处理 headers 完成后的状态
      goto s_n_llhttp__internal__n_invoke_llhttp__after_headers_complete;
  }
  /* UNREACHABLE */;
  // 如果程序执行到这里，表示代码存在逻辑错误，因为前面的 case 都没有匹配到
  abort();
}
s_n_llhttp__internal__n_invoke_or_flags_2: {
  switch (llhttp__internal__c_or_flags_1(state, p, endp)) {
    default:
      // 转到处理 headers 完成后的状态
      goto s_n_llhttp__internal__n_invoke_llhttp__after_headers_complete;
  }
  /* UNREACHABLE */;
  // 如果程序执行到这里，表示代码存在逻辑错误，因为前面的 case 都没有匹配到
  abort();
}
s_n_llhttp__internal__n_invoke_update_upgrade: {
  switch (llhttp__internal__c_update_upgrade(state, p, endp)) {
    default:
      // 转到处理 headers 完成后的状态
      goto s_n_llhttp__internal__n_invoke_or_flags_2;
  }
  /* UNREACHABLE */;
  // 如果程序执行到这里，表示代码存在逻辑错误，因为前面的 case 都没有匹配到
  abort();
}
s_n_llhttp__internal__n_pause_6: {
  // 设置错误码
  state->error = 0x15;
  // 设置错误原因
  state->reason = "Paused by on_headers_complete";
  // 设置错误位置
  state->error_pos = (const char*) p;
  // 设置当前状态
  state->_current = (void*) (intptr_t) s_n_llhttp__internal__n_invoke_llhttp__after_headers_complete;
  // 返回错误状态
  return s_error;
  /* UNREACHABLE */;
  // 如果程序执行到这里，表示代码存在逻辑错误，因为前面的 case 都没有匹配到
  abort();
}
s_n_llhttp__internal__n_error_2: {
  // 设置错误码
  state->error = 0x11;
  // 设置错误原因
  state->reason = "User callback error";
  // 设置错误位置
  state->error_pos = (const char*) p;
  // 设置当前状态
  state->_current = (void*) (intptr_t) s_error;
  // 返回错误状态
  return s_error;
  /* UNREACHABLE */;
  // 如果程序执行到这里，表示代码存在逻辑错误，因为前面的 case 都没有匹配到
  abort();
}
s_n_llhttp__internal__n_invoke_llhttp__on_headers_complete: {
  switch (llhttp__on_headers_complete(state, p, endp)) {
    case 0:
      // 转到处理 headers 完成后的状态
      goto s_n_llhttp__internal__n_invoke_llhttp__after_headers_complete;
    case 1:
      // 转到处理 or flags 1 的状态
      goto s_n_llhttp__internal__n_invoke_or_flags_1;
    case 2:
      // 转到处理 update upgrade 的状态
      goto s_n_llhttp__internal__n_invoke_update_upgrade;
    case 21:
      // 转到暂停状态
      goto s_n_llhttp__internal__n_pause_6;
    default:
      // 转到错误状态
      goto s_n_llhttp__internal__n_error_2;
  }
  /* UNREACHABLE */;
  // 如果程序执行到这里，表示代码存在逻辑错误，因为前面的 case 都没有匹配到
  abort();
}
s_n_llhttp__internal__n_invoke_llhttp__before_headers_complete: {
  switch (llhttp__before_headers_complete(state, p, endp)) {
    default:
      // 转到内部状态机处理头部完成前的逻辑
      goto s_n_llhttp__internal__n_invoke_llhttp__on_headers_complete;
  }
  /* UNREACHABLE */;
  // 如果执行到这里，说明出现了不可达的情况，中止程序
  abort();
}

s_n_llhttp__internal__n_invoke_test_lenient_flags_1: {
  switch (llhttp__internal__c_test_lenient_flags_1(state, p, endp)) {
    case 0:
      // 如果测试不通过，转到错误处理状态
      goto s_n_llhttp__internal__n_error_15;
    default:
      // 测试通过，转到处理头部完成前的逻辑
      goto s_n_llhttp__internal__n_invoke_llhttp__before_headers_complete;
  }
  /* UNREACHABLE */;
  // 如果执行到这里，说明出现了不可达的情况，中止程序
  abort();
}

s_n_llhttp__internal__n_invoke_test_flags_1: {
  switch (llhttp__internal__c_test_flags_1(state, p, endp)) {
    case 1:
      // 如果测试通过，转到测试 lenient flags 的逻辑
      goto s_n_llhttp__internal__n_invoke_test_lenient_flags_1;
    default:
      // 测试不通过，转到处理头部完成前的逻辑
      goto s_n_llhttp__internal__n_invoke_llhttp__before_headers_complete;
  }
  /* UNREACHABLE */;
  // 如果执行到这里，说明出现了不可达的情况，中止程序
  abort();
}

s_n_llhttp__internal__n_invoke_test_flags: {
  switch (llhttp__internal__c_test_flags(state, p, endp)) {
    case 1:
      // 如果测试通过，转到处理 chunk 完成后的逻辑
      goto s_n_llhttp__internal__n_invoke_llhttp__on_chunk_complete_1;
    default:
      // 测试不通过，转到测试 flags 1 的逻辑
      goto s_n_llhttp__internal__n_invoke_test_flags_1;
  }
  /* UNREACHABLE */;
  // 如果执行到这里，说明出现了不可达的情况，中止程序
  abort();
}

s_n_llhttp__internal__n_error_17: {
  // 设置错误码和错误原因
  state->error = 0x2;
  state->reason = "Expected LF after headers";
  state->error_pos = (const char*) p;
  state->_current = (void*) (intptr_t) s_error;
  return s_error;
  /* UNREACHABLE */;
  // 如果执行到这里，说明出现了不可达的情况，中止程序
  abort();
}

s_n_llhttp__internal__n_error_18: {
  // 设置错误码和错误原因
  state->error = 0xb;
  state->reason = "Empty Content-Length";
  state->error_pos = (const char*) p;
  state->_current = (void*) (intptr_t) s_error;
  return s_error;
  /* UNREACHABLE */;
  // 如果执行到这里，说明出现了不可达的情况，中止程序
  abort();
}

s_n_llhttp__internal__n_span_end_llhttp__on_header_value: {
  const unsigned char* start;
  int err;

  // 保存头部值的起始位置
  start = state->_span_pos0;
  state->_span_pos0 = NULL;
  // 调用处理头部值的函数
  err = llhttp__on_header_value(state, start, p);
    # 如果错误码不为0，则将错误码赋值给状态对象的error属性
    if (err != 0) {
      state->error = err;
      # 将当前指针位置转换为const char*类型，并赋值给状态对象的error_pos属性
      state->error_pos = (const char*) p;
      # 将_current属性设置为指向函数指针的void*类型，并返回s_error状态
      state->_current = (void*) (intptr_t) s_n_llhttp__internal__n_invoke_llhttp__on_header_value_complete;
      return s_error;
    }
    # 跳转到指定标签处
    goto s_n_llhttp__internal__n_invoke_llhttp__on_header_value_complete;
    # 不可达代码，中止程序执行
    /* UNREACHABLE */;
    abort();
  }
  s_n_llhttp__internal__n_invoke_update_header_state: {
    # 根据状态对象和指针位置更新头部状态，并根据返回值进行跳转
    switch (llhttp__internal__c_update_header_state(state, p, endp)) {
      default:
        # 跳转到指定标签处
        goto s_n_llhttp__internal__n_span_start_llhttp__on_header_value;
    }
    # 不可达代码，中止程序执行
    /* UNREACHABLE */;
    abort();
  }
  s_n_llhttp__internal__n_invoke_or_flags_3: {
    # 根据状态对象和指针位置进行位或操作，并根据返回值进行跳转
    switch (llhttp__internal__c_or_flags_3(state, p, endp)) {
      default:
        # 跳转到指定标签处
        goto s_n_llhttp__internal__n_invoke_update_header_state;
    }
    # 不可达代码，中止程序执行
    /* UNREACHABLE */;
    abort();
  }
  # 其余部分同上，根据状态对象和指针位置进行相应操作，并根据返回值进行跳转
    # 根据状态值调用不同的函数，并跳转到相应的状态
    switch (llhttp__internal__c_load_header_state(state, p, endp)) {
      case 5:
        goto s_n_llhttp__internal__n_invoke_or_flags_3;
      case 6:
        goto s_n_llhttp__internal__n_invoke_or_flags_4;
      case 7:
        goto s_n_llhttp__internal__n_invoke_or_flags_5;
      case 8:
        goto s_n_llhttp__internal__n_invoke_or_flags_6;
      default:
        goto s_n_llhttp__internal__n_span_start_llhttp__on_header_value;
    }
    /* UNREACHABLE */;
    abort();
  }
  s_n_llhttp__internal__n_invoke_load_header_state: {
    # 根据状态值调用不同的函数，并跳转到相应的状态
    switch (llhttp__internal__c_load_header_state(state, p, endp)) {
      case 2:
        goto s_n_llhttp__internal__n_error_18;
      default:
        goto s_n_llhttp__internal__n_invoke_load_header_state_1;
    }
    /* UNREACHABLE */;
    abort();
  }
  s_n_llhttp__internal__n_error_19: {
    # 设置状态的错误码、原因和错误位置，并返回错误状态
    state->error = 0x2;
    state->reason = "Expected LF after CR";
    state->error_pos = (const char*) p;
    state->_current = (void*) (intptr_t) s_error;
    return s_error;
    /* UNREACHABLE */;
    abort();
  }
  s_n_llhttp__internal__n_invoke_update_header_state_1: {
    # 根据状态值调用不同的函数，并跳转到相应的状态
    switch (llhttp__internal__c_update_header_state(state, p, endp)) {
      default:
        goto s_n_llhttp__internal__n_invoke_llhttp__on_header_value_complete;
    }
    /* UNREACHABLE */;
    abort();
  }
  s_n_llhttp__internal__n_invoke_or_flags_7: {
    # 根据状态值调用不同的函数，并跳转到相应的状态
    switch (llhttp__internal__c_or_flags_3(state, p, endp)) {
      default:
        goto s_n_llhttp__internal__n_invoke_update_header_state_1;
    }
    /* UNREACHABLE */;
    abort();
  }
  s_n_llhttp__internal__n_invoke_or_flags_8: {
    # 根据状态值调用不同的函数，并跳转到相应的状态
    switch (llhttp__internal__c_or_flags_4(state, p, endp)) {
      default:
        goto s_n_llhttp__internal__n_invoke_update_header_state_1;
    }
    /* UNREACHABLE */;
    abort();
  }
  s_n_llhttp__internal__n_invoke_or_flags_9: {
    # 根据状态值调用不同的函数，并跳转到相应的状态
    switch (llhttp__internal__c_or_flags_5(state, p, endp)) {
      default:
        goto s_n_llhttp__internal__n_invoke_update_header_state_1;
    }
  /* UNREACHABLE */;
  // 标记代码为不可达，中止程序
  abort();
}
s_n_llhttp__internal__n_invoke_or_flags_10: {
  switch (llhttp__internal__c_or_flags_6(state, p, endp)) {
    default:
      goto s_n_llhttp__internal__n_invoke_llhttp__on_header_value_complete;
  }
  /* UNREACHABLE */;
  // 标记代码为不可达，中止程序
  abort();
}
s_n_llhttp__internal__n_invoke_load_header_state_3: {
  switch (llhttp__internal__c_load_header_state(state, p, endp)) {
    case 5:
      goto s_n_llhttp__internal__n_invoke_or_flags_7;
    case 6:
      goto s_n_llhttp__internal__n_invoke_or_flags_8;
    case 7:
      goto s_n_llhttp__internal__n_invoke_or_flags_9;
    case 8:
      goto s_n_llhttp__internal__n_invoke_or_flags_10;
    default:
      goto s_n_llhttp__internal__n_invoke_llhttp__on_header_value_complete;
  }
  /* UNREACHABLE */;
  // 标记代码为不可达，中止程序
  abort();
}
s_n_llhttp__internal__n_error_20: {
  state->error = 0x3;
  state->reason = "Missing expected LF after header value";
  state->error_pos = (const char*) p;
  state->_current = (void*) (intptr_t) s_error;
  return s_error;
  /* UNREACHABLE */;
  // 标记代码为不可达，中止程序
  abort();
}
s_n_llhttp__internal__n_span_end_llhttp__on_header_value_1: {
  const unsigned char* start;
  int err;

  start = state->_span_pos0;
  state->_span_pos0 = NULL;
  err = llhttp__on_header_value(state, start, p);
  if (err != 0) {
    state->error = err;
    state->error_pos = (const char*) p;
    state->_current = (void*) (intptr_t) s_n_llhttp__internal__n_header_value_almost_done;
    return s_error;
  }
  goto s_n_llhttp__internal__n_header_value_almost_done;
  /* UNREACHABLE */;
  // 标记代码为不可达，中止程序
  abort();
}
s_n_llhttp__internal__n_span_end_llhttp__on_header_value_2: {
  const unsigned char* start;
  int err;

  start = state->_span_pos0;
  state->_span_pos0 = NULL;
  err = llhttp__on_header_value(state, start, p);
    // 如果 err 不等于 0，则将错误码赋值给 state->error
    if (err != 0) {
      state->error = err;
      // 将指向错误位置的指针赋值给 state->error_pos
      state->error_pos = (const char*) (p + 1);
      // 将当前状态设置为 s_n_llhttp__internal__n_header_value_almost_done
      state->_current = (void*) (intptr_t) s_n_llhttp__internal__n_header_value_almost_done;
      // 返回 s_error 状态
      return s_error;
    }
    // 指针 p 自增
    p++;
    // 跳转到 s_n_llhttp__internal__n_header_value_almost_done 状态
    goto s_n_llhttp__internal__n_header_value_almost_done;
    /* UNREACHABLE */;
    // 中止程序
    abort();
  }
  s_n_llhttp__internal__n_span_end_llhttp__on_header_value_3: {
    // 定义无符号字符指针 start 和整型变量 err
    const unsigned char* start;
    int err;

    // 将 state->_span_pos0 赋值给 start，并将 state->_span_pos0 置为 NULL
    start = state->_span_pos0;
    state->_span_pos0 = NULL;
    // 调用 llhttp__on_header_value 函数，将返回值赋值给 err
    err = llhttp__on_header_value(state, start, p);
    // 如果 err 不等于 0，则将错误码赋值给 state->error
    if (err != 0) {
      state->error = err;
      // 将指向错误位置的指针赋值给 state->error_pos
      state->error_pos = (const char*) (p + 1);
      // 将当前状态设置为 s_n_llhttp__internal__n_header_value_almost_done
      state->_current = (void*) (intptr_t) s_n_llhttp__internal__n_header_value_almost_done;
      // 返回 s_error 状态
      return s_error;
    }
    // 指针 p 自增
    p++;
    // 跳转到 s_n_llhttp__internal__n_header_value_almost_done 状态
    goto s_n_llhttp__internal__n_header_value_almost_done;
    /* UNREACHABLE */;
    // 中止程序
    abort();
  }
  s_n_llhttp__internal__n_error_21: {
    // 将错误码 0xa 赋值给 state->error
    state->error = 0xa;
    // 将错误原因赋值为 "Invalid header value char"
    state->reason = "Invalid header value char";
    // 将指向错误位置的指针赋值给 state->error_pos
    state->error_pos = (const char*) p;
    // 将当前状态设置为 s_error
    state->_current = (void*) (intptr_t) s_error;
    // 返回 s_error 状态
    return s_error;
    /* UNREACHABLE */;
    // 中止程序
    abort();
  }
  s_n_llhttp__internal__n_invoke_test_lenient_flags_2: {
    // 调用 llhttp__internal__c_test_lenient_flags_2 函数，并根据返回值进行跳转
    switch (llhttp__internal__c_test_lenient_flags_2(state, p, endp)) {
      case 1:
        // 跳转到 s_n_llhttp__internal__n_header_value_lenient
        goto s_n_llhttp__internal__n_header_value_lenient;
      default:
        // 跳转到 s_n_llhttp__internal__n_error_21
        goto s_n_llhttp__internal__n_error_21;
    }
    /* UNREACHABLE */;
    // 中止程序
    abort();
  }
  s_n_llhttp__internal__n_invoke_update_header_state_3: {
    // 调用 llhttp__internal__c_update_header_state 函数，并根据返回值进行跳转
    switch (llhttp__internal__c_update_header_state(state, p, endp)) {
      default:
        // 跳转到 s_n_llhttp__internal__n_header_value_connection
        goto s_n_llhttp__internal__n_header_value_connection;
    }
    /* UNREACHABLE */;
    // 中止程序
    abort();
  }
  s_n_llhttp__internal__n_invoke_or_flags_11: {
    // 调用 llhttp__internal__c_or_flags_3 函数，并根据返回值进行跳转
    switch (llhttp__internal__c_or_flags_3(state, p, endp)) {
      default:
        // 跳转到 s_n_llhttp__internal__n_invoke_update_header_state_3
        goto s_n_llhttp__internal__n_invoke_update_header_state_3;
    }
    /* UNREACHABLE */;
    // 中止程序
    abort();
  }
  s_n_llhttp__internal__n_invoke_or_flags_12: {
  switch (llhttp__internal__c_or_flags_4(state, p, endp)) {
    default:
      // 转到指定状态
      goto s_n_llhttp__internal__n_invoke_update_header_state_3;
  }
  /* UNREACHABLE */;
  // 终止程序
  abort();
}
s_n_llhttp__internal__n_invoke_or_flags_13: {
  switch (llhttp__internal__c_or_flags_5(state, p, endp)) {
    default:
      // 转到指定状态
      goto s_n_llhttp__internal__n_invoke_update_header_state_3;
  }
  /* UNREACHABLE */;
  // 终止程序
  abort();
}
s_n_llhttp__internal__n_invoke_or_flags_14: {
  switch (llhttp__internal__c_or_flags_6(state, p, endp)) {
    default:
      // 转到指定状态
      goto s_n_llhttp__internal__n_header_value_connection;
  }
  /* UNREACHABLE */;
  // 终止程序
  abort();
}
s_n_llhttp__internal__n_invoke_load_header_state_4: {
  switch (llhttp__internal__c_load_header_state(state, p, endp)) {
    case 5:
      // 转到指定状态
      goto s_n_llhttp__internal__n_invoke_or_flags_11;
    case 6:
      // 转到指定状态
      goto s_n_llhttp__internal__n_invoke_or_flags_12;
    case 7:
      // 转到指定状态
      goto s_n_llhttp__internal__n_invoke_or_flags_13;
    case 8:
      // 转到指定状态
      goto s_n_llhttp__internal__n_invoke_or_flags_14;
    default:
      // 转到指定状态
      goto s_n_llhttp__internal__n_header_value_connection;
  }
  /* UNREACHABLE */;
  // 终止程序
  abort();
}
s_n_llhttp__internal__n_invoke_update_header_state_4: {
  switch (llhttp__internal__c_update_header_state_4(state, p, endp)) {
    default:
      // 转到指定状态
      goto s_n_llhttp__internal__n_header_value_connection_token;
  }
  /* UNREACHABLE */;
  // 终止程序
  abort();
}
s_n_llhttp__internal__n_invoke_update_header_state_2: {
  switch (llhttp__internal__c_update_header_state_2(state, p, endp)) {
    default:
      // 转到指定状态
      goto s_n_llhttp__internal__n_header_value_connection_ws;
  }
  /* UNREACHABLE */;
  // 终止程序
  abort();
}
s_n_llhttp__internal__n_invoke_update_header_state_5: {
  switch (llhttp__internal__c_update_header_state_5(state, p, endp)) {
    default:
      // 转到指定状态
      goto s_n_llhttp__internal__n_header_value_connection_ws;
  }
  /* UNREACHABLE */;
    // 调用中止函数
    abort();
  }
  s_n_llhttp__internal__n_invoke_update_header_state_6: {
    // 调用更新头部状态函数
    switch (llhttp__internal__c_update_header_state_6(state, p, endp)) {
      default:
        // 转到指定状态
        goto s_n_llhttp__internal__n_header_value_connection_ws;
    }
    /* UNREACHABLE */;
    // 调用中止函数
    abort();
  }
  s_n_llhttp__internal__n_span_end_llhttp__on_header_value_4: {
    // 定义无符号字符指针和整数变量
    const unsigned char* start;
    int err;

    // 设置起始位置
    start = state->_span_pos0;
    state->_span_pos0 = NULL;
    // 调用处理头部值函数
    err = llhttp__on_header_value(state, start, p);
    // 如果出现错误
    if (err != 0) {
      state->error = err;
      state->error_pos = (const char*) p;
      state->_current = (void*) (intptr_t) s_n_llhttp__internal__n_error_23;
      return s_error;
    }
    // 转到指定状态
    goto s_n_llhttp__internal__n_error_23;
    /* UNREACHABLE */;
    // 调用中止函数
    abort();
  }
  s_n_llhttp__internal__n_invoke_mul_add_content_length_1: {
    // 调用添加内容长度函数
    switch (llhttp__internal__c_mul_add_content_length_1(state, p, endp, match)) {
      case 1:
        // 转到指定状态
        goto s_n_llhttp__internal__n_span_end_llhttp__on_header_value_4;
      default:
        // 转到指定状态
        goto s_n_llhttp__internal__n_header_value_content_length;
    }
    /* UNREACHABLE */;
    // 调用中止函数
    abort();
  }
  s_n_llhttp__internal__n_invoke_or_flags_15: {
    // 调用或标志函数
    switch (llhttp__internal__c_or_flags_15(state, p, endp)) {
      default:
        // 转到指定状态
        goto s_n_llhttp__internal__n_header_value_otherwise;
    }
    /* UNREACHABLE */;
    // 调用中止函数
    abort();
  }
  s_n_llhttp__internal__n_span_end_llhttp__on_header_value_5: {
    // 定义无符号字符指针和整数变量
    const unsigned char* start;
    int err;

    // 设置起始位置
    start = state->_span_pos0;
    state->_span_pos0 = NULL;
    // 调用处理头部值函数
    err = llhttp__on_header_value(state, start, p);
    // 如果出现错误
    if (err != 0) {
      state->error = err;
      state->error_pos = (const char*) p;
      state->_current = (void*) (intptr_t) s_n_llhttp__internal__n_error_24;
      return s_error;
    }
    // 转到指定状态
    goto s_n_llhttp__internal__n_error_24;
    /* UNREACHABLE */;
    // 调用中止函数
    abort();
  }
  s_n_llhttp__internal__n_error_22: {
    // 设置错误码和原因
    state->error = 0x4;
    state->reason = "Duplicate Content-Length";
    state->error_pos = (const char*) p;  # 将指针 p 转换为 const char* 类型，并赋值给 state->error_pos
    state->_current = (void*) (intptr_t) s_error;  # 将 s_error 转换为 intptr_t 类型，再转换为 void* 类型，并赋值给 state->_current
    return s_error;  # 返回 s_error
    /* UNREACHABLE */;  # 不可达代码，表示此处的代码不会被执行到
    abort();  # 终止程序的执行
  }
  s_n_llhttp__internal__n_invoke_test_flags_2: {
    switch (llhttp__internal__c_test_flags_2(state, p, endp)) {  # 调用 llhttp__internal__c_test_flags_2 函数，根据返回值进行不同的跳转
      case 0:  # 如果返回值为 0
        goto s_n_llhttp__internal__n_header_value_content_length;  # 跳转到标签 s_n_llhttp__internal__n_header_value_content_length
      default:  # 其他情况
        goto s_n_llhttp__internal__n_error_22;  # 跳转到标签 s_n_llhttp__internal__n_error_22
    }
    /* UNREACHABLE */;  # 不可达代码，表示此处的代码不会被执行到
    abort();  # 终止程序的执行
  }
  s_n_llhttp__internal__n_invoke_update_header_state_7: {
    switch (llhttp__internal__c_update_header_state_7(state, p, endp)) {  # 调用 llhttp__internal__c_update_header_state_7 函数，根据返回值进行不同的跳转
      default:  # 默认情况
        goto s_n_llhttp__internal__n_header_value_otherwise;  # 跳转到标签 s_n_llhttp__internal__n_header_value_otherwise
    }
    /* UNREACHABLE */;  # 不可达代码，表示此处的代码不会被执行到
    abort();  # 终止程序的执行
  }
  s_n_llhttp__internal__n_invoke_update_header_state_8: {
    switch (llhttp__internal__c_update_header_state_4(state, p, endp)) {  # 调用 llhttp__internal__c_update_header_state_4 函数，根据返回值进行不同的跳转
      default:  # 默认情况
        goto s_n_llhttp__internal__n_header_value;  # 跳转到标签 s_n_llhttp__internal__n_header_value
    }
    /* UNREACHABLE */;  # 不可达代码，表示此处的代码不会被执行到
    abort();  # 终止程序的执行
  }
  s_n_llhttp__internal__n_invoke_and_flags: {
    switch (llhttp__internal__c_and_flags(state, p, endp)) {  # 调用 llhttp__internal__c_and_flags 函数，根据返回值进行不同的跳转
      default:  # 默认情况
        goto s_n_llhttp__internal__n_header_value_te_chunked;  # 跳转到标签 s_n_llhttp__internal__n_header_value_te_chunked
    }
    /* UNREACHABLE */;  # 不可达代码，表示此处的代码不会被执行到
    abort();  # 终止程序的执行
  }
  s_n_llhttp__internal__n_invoke_or_flags_16: {
    switch (llhttp__internal__c_or_flags_16(state, p, endp)) {  # 调用 llhttp__internal__c_or_flags_16 函数，根据返回值进行不同的跳转
      default:  # 默认情况
        goto s_n_llhttp__internal__n_invoke_and_flags;  # 跳转到标签 s_n_llhttp__internal__n_invoke_and_flags
    }
    /* UNREACHABLE */;  # 不可达代码，表示此处的代码不会被执行到
    abort();  # 终止程序的执行
  }
  s_n_llhttp__internal__n_invoke_or_flags_17: {
    switch (llhttp__internal__c_or_flags_17(state, p, endp)) {  # 调用 llhttp__internal__c_or_flags_17 函数，根据返回值进行不同的跳转
      default:  # 默认情况
        goto s_n_llhttp__internal__n_invoke_update_header_state_8;  # 跳转到标签 s_n_llhttp__internal__n_invoke_update_header_state_8
    }
    /* UNREACHABLE */;  # 不可达代码，表示此处的代码不会被执行到
    abort();  # 终止程序的执行
  }
  s_n_llhttp__internal__n_invoke_load_header_state_2: {
    switch (llhttp__internal__c_load_header_state(state, p, endp)) {
      // 根据当前状态和输入的数据，加载下一个状态
      case 1:
        // 转到处理 header value 的状态
        goto s_n_llhttp__internal__n_header_value_connection;
      case 2:
        // 转到调用测试标志 2 的状态
        goto s_n_llhttp__internal__n_invoke_test_flags_2;
      case 3:
        // 转到调用或标志 16 的状态
        goto s_n_llhttp__internal__n_invoke_or_flags_16;
      case 4:
        // 转到调用或标志 17 的状态
        goto s_n_llhttp__internal__n_invoke_or_flags_17;
      default:
        // 转到处理 header value 的状态
        goto s_n_llhttp__internal__n_header_value;
    }
    /* UNREACHABLE */;
    // 如果执行到这里，表示代码逻辑有误，中止程序
    abort();
  }
  s_n_llhttp__internal__n_span_end_llhttp__on_header_field: {
    const unsigned char* start;
    int err;

    start = state->_span_pos0;
    state->_span_pos0 = NULL;
    err = llhttp__on_header_field(state, start, p);
    if (err != 0) {
      state->error = err;
      state->error_pos = (const char*) (p + 1);
      state->_current = (void*) (intptr_t) s_n_llhttp__internal__n_invoke_llhttp__on_header_field_complete;
      return s_error;
    }
    p++;
    goto s_n_llhttp__internal__n_invoke_llhttp__on_header_field_complete;
    /* UNREACHABLE */;
    abort();
  }
  s_n_llhttp__internal__n_span_end_llhttp__on_header_field_1: {
    const unsigned char* start;
    int err;

    start = state->_span_pos0;
    state->_span_pos0 = NULL;
    err = llhttp__on_header_field(state, start, p);
    if (err != 0) {
      state->error = err;
      state->error_pos = (const char*) (p + 1);
      state->_current = (void*) (intptr_t) s_n_llhttp__internal__n_invoke_llhttp__on_header_field_complete;
      return s_error;
    }
    p++;
    goto s_n_llhttp__internal__n_invoke_llhttp__on_header_field_complete;
    /* UNREACHABLE */;
    abort();
  }
  s_n_llhttp__internal__n_error_25: {
    state->error = 0xa;
    state->reason = "Invalid header token";
    state->error_pos = (const char*) p;
    state->_current = (void*) (intptr_t) s_error;
    return s_error;
    /* UNREACHABLE */;
    abort();
  }
  s_n_llhttp__internal__n_invoke_update_header_state_9: {
    // 省略部分代码
  switch (llhttp__internal__c_update_header_state_4(state, p, endp)) {
    default:
      // 转到通用头字段状态
      goto s_n_llhttp__internal__n_header_field_general;
  }
  /* UNREACHABLE */;
  // 终止程序
  abort();
}
s_n_llhttp__internal__n_invoke_store_header_state: {
  switch (llhttp__internal__c_store_header_state(state, p, endp, match)) {
    default:
      // 转到头字段冒号状态
      goto s_n_llhttp__internal__n_header_field_colon;
  }
  /* UNREACHABLE */;
  // 终止程序
  abort();
}
s_n_llhttp__internal__n_invoke_update_header_state_10: {
  switch (llhttp__internal__c_update_header_state_4(state, p, endp)) {
    default:
      // 转到通用头字段状态
      goto s_n_llhttp__internal__n_header_field_general;
  }
  /* UNREACHABLE */;
  // 终止程序
  abort();
}
s_n_llhttp__internal__n_invoke_llhttp__on_url_complete: {
  switch (llhttp__on_url_complete(state, p, endp)) {
    default:
      // 转到头字段开始状态
      goto s_n_llhttp__internal__n_header_field_start;
  }
  /* UNREACHABLE */;
  // 终止程序
  abort();
}
s_n_llhttp__internal__n_invoke_update_http_minor: {
  switch (llhttp__internal__c_update_http_minor(state, p, endp)) {
    default:
      // 转到 URL 完成状态
      goto s_n_llhttp__internal__n_invoke_llhttp__on_url_complete;
  }
  /* UNREACHABLE */;
  // 终止程序
  abort();
}
s_n_llhttp__internal__n_invoke_update_http_major: {
  switch (llhttp__internal__c_update_http_major(state, p, endp)) {
    default:
      // 转到更新次要 HTTP 版本号状态
      goto s_n_llhttp__internal__n_invoke_update_http_minor;
  }
  /* UNREACHABLE */;
  // 终止程序
  abort();
}
s_n_llhttp__internal__n_span_end_llhttp__on_url_3: {
  const unsigned char* start;
  int err;

  start = state->_span_pos0;
  state->_span_pos0 = NULL;
  err = llhttp__on_url(state, start, p);
  if (err != 0) {
    state->error = err;
    state->error_pos = (const char*) p;
    state->_current = (void*) (intptr_t) s_n_llhttp__internal__n_url_skip_to_http09;
    return s_error;
  }
  // 转到跳转到 HTTP/0.9 状态
  goto s_n_llhttp__internal__n_url_skip_to_http09;
  /* UNREACHABLE */;
  // 终止程序
  abort();
}
s_n_llhttp__internal__n_error_26: {
  state->error = 0x7;
    state->reason = "Expected CRLF";  # 设置状态的原因为“预期CRLF”
    state->error_pos = (const char*) p;  # 设置状态的错误位置为当前指针位置
    state->_current = (void*) (intptr_t) s_error;  # 设置状态的当前处理函数为错误处理函数
    return s_error;  # 返回错误状态
    /* UNREACHABLE */;  # 不可达代码
    abort();  # 终止程序运行
  }
  s_n_llhttp__internal__n_span_end_llhttp__on_url_4: {  # 定义状态处理函数
    const unsigned char* start;  # 声明无符号字符指针变量start
    int err;  # 声明整型变量err

    start = state->_span_pos0;  # 将状态中的_span_pos0赋值给start
    state->_span_pos0 = NULL;  # 将状态中的_span_pos0置为NULL
    err = llhttp__on_url(state, start, p);  # 调用llhttp__on_url函数，将结果赋值给err
    if (err != 0) {  # 如果err不等于0
      state->error = err;  # 设置状态的错误为err
      state->error_pos = (const char*) p;  # 设置状态的错误位置为当前指针位置
      state->_current = (void*) (intptr_t) s_n_llhttp__internal__n_url_skip_lf_to_http09;  # 设置状态的当前处理函数为跳过LF到HTTP09的处理函数
      return s_error;  # 返回错误状态
    }
    goto s_n_llhttp__internal__n_url_skip_lf_to_http09;  # 跳转到n_url_skip_lf_to_http09状态
    /* UNREACHABLE */;  # 不可达代码
    abort();  # 终止程序运行
  }
  s_n_llhttp__internal__n_error_29: {  # 定义状态处理函数
    state->error = 0x17;  # 设置状态的错误为0x17
    state->reason = "Pause on PRI/Upgrade";  # 设置状态的原因为“暂停在PRI/Upgrade”
    state->error_pos = (const char*) p;  # 设置状态的错误位置为当前指针位置
    state->_current = (void*) (intptr_t) s_error;  # 设置状态的当前处理函数为错误处理函数
    return s_error;  # 返回错误状态
    /* UNREACHABLE */;  # 不可达代码
    abort();  # 终止程序运行
  }
  s_n_llhttp__internal__n_error_30: {  # 定义状态处理函数
    state->error = 0x9;  # 设置状态的错误为0x9
    state->reason = "Expected HTTP/2 Connection Preface";  # 设置状态的原因为“预期HTTP/2连接序言”
    state->error_pos = (const char*) p;  # 设置状态的错误位置为当前指针位置
    state->_current = (void*) (intptr_t) s_error;  # 设置状态的当前处理函数为错误处理函数
    return s_error;  # 返回错误状态
    /* UNREACHABLE */;  # 不可达代码
    abort();  # 终止程序运行
  }
  s_n_llhttp__internal__n_error_28: {  # 定义状态处理函数
    state->error = 0x9;  # 设置状态的错误为0x9
    state->reason = "Expected CRLF after version";  # 设置状态的原因为“版本后预期CRLF”
    state->error_pos = (const char*) p;  # 设置状态的错误位置为当前指针位置
    state->_current = (void*) (intptr_t) s_error;  # 设置状态的当前处理函数为错误处理函数
    return s_error;  # 返回错误状态
    /* UNREACHABLE */;  # 不可达代码
    abort();  # 终止程序运行
  }
  s_n_llhttp__internal__n_invoke_load_method_1: {  # 定义状态处理函数
    switch (llhttp__internal__c_load_method(state, p, endp)) {  # 调用c_load_method函数，根据返回值进行判断
      case 34:  # 如果返回值为34
        goto s_n_llhttp__internal__n_req_pri_upgrade;  # 跳转到n_req_pri_upgrade状态
      default:  # 其他情况
        goto s_n_llhttp__internal__n_req_http_complete;  # 跳转到n_req_http_complete状态
    }
    /* UNREACHABLE */;  # 不可达代码
    abort();  # 终止程序运行
  }
  s_n_llhttp__internal__n_invoke_store_http_minor: {  # 定义状态处理函数
    switch (llhttp__internal__c_store_http_minor(state, p, endp, match)) {  # 调用c_store_http_minor函数，根据返回值进行判断
      default:  # 默认情况
        goto s_n_llhttp__internal__n_invoke_load_method_1;  # 跳转到n_invoke_load_method_1状态
    }
  /* UNREACHABLE */;
  // 标记代码为不可达，中止程序
  abort();
}
s_n_llhttp__internal__n_error_31: {
  state->error = 0x9;
  state->reason = "Invalid minor version";
  state->error_pos = (const char*) p;
  state->_current = (void*) (intptr_t) s_error;
  return s_error;
  /* UNREACHABLE */;
  // 标记代码为不可达，中止程序
  abort();
}
s_n_llhttp__internal__n_error_32: {
  state->error = 0x9;
  state->reason = "Expected dot";
  state->error_pos = (const char*) p;
  state->_current = (void*) (intptr_t) s_error;
  return s_error;
  /* UNREACHABLE */;
  // 标记代码为不可达，中止程序
  abort();
}
s_n_llhttp__internal__n_invoke_store_http_major: {
  switch (llhttp__internal__c_store_http_major(state, p, endp, match)) {
    default:
      goto s_n_llhttp__internal__n_req_http_dot;
  }
  /* UNREACHABLE */;
  // 标记代码为不可达，中止程序
  abort();
}
s_n_llhttp__internal__n_error_33: {
  state->error = 0x9;
  state->reason = "Invalid major version";
  state->error_pos = (const char*) p;
  state->_current = (void*) (intptr_t) s_error;
  return s_error;
  /* UNREACHABLE */;
  // 标记代码为不可达，中止程序
  abort();
}
s_n_llhttp__internal__n_error_27: {
  state->error = 0x8;
  state->reason = "Invalid method for HTTP/x.x request";
  state->error_pos = (const char*) p;
  state->_current = (void*) (intptr_t) s_error;
  return s_error;
  /* UNREACHABLE */;
  // 标记代码为不可达，中止程序
  abort();
}
s_n_llhttp__internal__n_invoke_load_method: {
  }
  /* UNREACHABLE */;
  // 标记代码为不可达，中止程序
  abort();
}
s_n_llhttp__internal__n_error_36: {
  state->error = 0x8;
  state->reason = "Expected HTTP/";
  state->error_pos = (const char*) p;
  state->_current = (void*) (intptr_t) s_error;
  return s_error;
  /* UNREACHABLE */;
  // 标记代码为不可达，中止程序
  abort();
}
s_n_llhttp__internal__n_error_34: {
  state->error = 0x8;
  state->reason = "Expected SOURCE method for ICE/x.x request";
  state->error_pos = (const char*) p;
  state->_current = (void*) (intptr_t) s_error;
  return s_error;
  /* UNREACHABLE */;
  // 标记代码为不可达，中止程序
  abort();
}
s_n_llhttp__internal__n_invoke_load_method_2: {
    switch (llhttp__internal__c_load_method(state, p, endp)) {
      case 33:
        // 如果加载方法返回值为33，则跳转到状态 s_n_llhttp__internal__n_req_http_major
        goto s_n_llhttp__internal__n_req_http_major;
      default:
        // 如果加载方法返回值为其他值，则跳转到状态 s_n_llhttp__internal__n_error_34
        goto s_n_llhttp__internal__n_error_34;
    }
    /* UNREACHABLE */;
    // 不可达代码，中止程序
    abort();
  }
  s_n_llhttp__internal__n_error_35: {
    // 设置状态的错误码为0x8
    state->error = 0x8;
    // 设置状态的错误原因为"Invalid method for RTSP/x.x request"
    state->reason = "Invalid method for RTSP/x.x request";
    // 设置状态的错误位置为当前指针位置
    state->error_pos = (const char*) p;
    // 设置状态的当前状态为错误状态
    state->_current = (void*) (intptr_t) s_error;
    // 返回错误状态
    return s_error;
    /* UNREACHABLE */;
    // 不可达代码，中止程序
    abort();
  }
  s_n_llhttp__internal__n_invoke_load_method_3: {
    switch (llhttp__internal__c_load_method(state, p, endp)) {
      case 1:
        // 如果加载方法返回值为1，则跳转到状态 s_n_llhttp__internal__n_req_http_major
        goto s_n_llhttp__internal__n_req_http_major;
      case 3:
        // 如果加载方法返回值为3，则跳转到状态 s_n_llhttp__internal__n_req_http_major
        goto s_n_llhttp__internal__n_req_http_major;
      // ... 其他case省略 ...
      default:
        // 如果加载方法返回值为其他值，则跳转到状态 s_n_llhttp__internal__n_error_35
        goto s_n_llhttp__internal__n_error_35;
    }
    /* UNREACHABLE */;
    // 不可达代码，中止程序
    abort();
  }
  s_n_llhttp__internal__n_invoke_llhttp__on_url_complete_1: {
    switch (llhttp__on_url_complete(state, p, endp)) {
      default:
        // 默认情况下，跳转到状态 s_n_llhttp__internal__n_req_http_start
        goto s_n_llhttp__internal__n_req_http_start;
    }
    /* UNREACHABLE */;
    // 不可达代码，中止程序
    abort();
  }
  s_n_llhttp__internal__n_span_end_llhttp__on_url_5: {
    const unsigned char* start;
    # 定义一个整型变量 err
    int err;

    # 将 state->_span_pos0 的值赋给 start
    start = state->_span_pos0;
    # 将 state->_span_pos0 置为 NULL
    state->_span_pos0 = NULL;
    # 调用 llhttp__on_url 函数，处理 URL，将结果赋给 err
    err = llhttp__on_url(state, start, p);
    # 如果 err 不等于 0，则设置 state 的 error 属性为 err
    state->error = err;
    # 设置 state 的 error_pos 属性为 p
    state->error_pos = (const char*) p;
    # 设置 state 的 _current 属性为 s_n_llhttp__internal__n_url_skip_to_http 的地址
    state->_current = (void*) (intptr_t) s_n_llhttp__internal__n_url_skip_to_http;
    # 返回 s_error
    return s_error;
    # 跳转到 s_n_llhttp__internal__n_url_skip_to_http
    goto s_n_llhttp__internal__n_url_skip_to_http;
    # 不可达代码
    /* UNREACHABLE */;
    # 中止程序
    abort();
  }
  # 定义标签 s_n_llhttp__internal__n_span_end_llhttp__on_url_6
  s_n_llhttp__internal__n_span_end_llhttp__on_url_6: {
    # 定义一个指向无符号字符的指针 start
    const unsigned char* start;
    # 定义一个整型变量 err
    int err;

    # 将 state->_span_pos0 的值赋给 start
    start = state->_span_pos0;
    # 将 state->_span_pos0 置为 NULL
    state->_span_pos0 = NULL;
    # 调用 llhttp__on_url 函数，处理 URL，将结果赋给 err
    err = llhttp__on_url(state, start, p);
    # 如果 err 不等于 0，则设置 state 的 error 属性为 err
    state->error = err;
    # 设置 state 的 error_pos 属性为 p
    state->error_pos = (const char*) p;
    # 设置 state 的 _current 属性为 s_n_llhttp__internal__n_url_skip_to_http09 的地址
    state->_current = (void*) (intptr_t) s_n_llhttp__internal__n_url_skip_to_http09;
    # 返回 s_error
    return s_error;
    # 跳转到 s_n_llhttp__internal__n_url_skip_to_http09
    goto s_n_llhttp__internal__n_url_skip_to_http09;
    # 不可达代码
    /* UNREACHABLE */;
    # 中止程序
    abort();
  }
  # 定义标签 s_n_llhttp__internal__n_span_end_llhttp__on_url_7
  s_n_llhttp__internal__n_span_end_llhttp__on_url_7: {
    # 定义一个指向无符号字符的指针 start
    const unsigned char* start;
    # 定义一个整型变量 err

    # 将 state->_span_pos0 的值赋给 start
    start = state->_span_pos0;
    # 将 state->_span_pos0 置为 NULL
    state->_span_pos0 = NULL;
    # 调用 llhttp__on_url 函数，处理 URL，将结果赋给 err
    err = llhttp__on_url(state, start, p);
    # 如果 err 不等于 0，则设置 state 的 error 属性为 err
    state->error = err;
    # 设置 state 的 error_pos 属性为 p
    state->error_pos = (const char*) p;
    # 设置 state 的 _current 属性为 s_n_llhttp__internal__n_url_skip_lf_to_http09 的地址
    state->_current = (void*) (intptr_t) s_n_llhttp__internal__n_url_skip_lf_to_http09;
    # 返回 s_error
    return s_error;
    # 跳转到 s_n_llhttp__internal__n_url_skip_lf_to_http09
    goto s_n_llhttp__internal__n_url_skip_lf_to_http09;
    # 不可达代码
    /* UNREACHABLE */;
    # 中止程序
    abort();
  }
  # 定义标签 s_n_llhttp__internal__n_span_end_llhttp__on_url_8
  s_n_llhttp__internal__n_span_end_llhttp__on_url_8: {
    # 定义一个指向无符号字符的指针 start
    const unsigned char* start;
    # 定义一个整型变量 err

    # 将 state->_span_pos0 的值赋给 start
    start = state->_span_pos0;
    # 将 state->_span_pos0 置为 NULL
    state->_span_pos0 = NULL;
    # 调用 llhttp__on_url 函数，处理 URL，将结果赋给 err
    err = llhttp__on_url(state, start, p);
    # 如果 err 不等于 0，则设置 state 的 error 属性为 err
    state->error = err;
    # 设置 state 的 error_pos 属性为 p
    state->error_pos = (const char*) p;
    # 设置 state 的 _current 属性为 s_n_llhttp__internal__n_url_skip_to_http 的地址
    state->_current = (void*) (intptr_t) s_n_llhttp__internal__n_url_skip_to_http;
    # 返回 s_error
    return s_error;
    # 跳转到 s_n_llhttp__internal__n_url_skip_to_http
    goto s_n_llhttp__internal__n_url_skip_to_http;
    # 不可达代码
    /* UNREACHABLE */;
    # 中止程序
    abort();
  }
  # 定义标签 s_n_llhttp__internal__n_error_37
  s_n_llhttp__internal__n_error_37: {
    # 设置 state 的 error 属性为 0x7
    state->error = 0x7;
    // 设置状态的原因为“URL片段起始位置存在无效字符”
    state->reason = "Invalid char in url fragment start";
    // 设置错误位置为当前位置
    state->error_pos = (const char*) p;
    // 设置当前状态为错误状态
    state->_current = (void*) (intptr_t) s_error;
    // 返回错误状态
    return s_error;
    /* UNREACHABLE */;  // 不可达代码，表示此处的代码不会被执行
    abort();  // 终止程序执行
  }
  s_n_llhttp__internal__n_span_end_llhttp__on_url_9: {
    const unsigned char* start;
    int err;

    start = state->_span_pos0;
    state->_span_pos0 = NULL;
    err = llhttp__on_url(state, start, p);
    if (err != 0) {
      state->error = err;
      state->error_pos = (const char*) p;
      state->_current = (void*) (intptr_t) s_n_llhttp__internal__n_url_skip_to_http09;
      return s_error;
    }
    goto s_n_llhttp__internal__n_url_skip_to_http09;
    /* UNREACHABLE */;  // 不可达代码，表示此处的代码不会被执行
    abort();  // 终止程序执行
  }
  s_n_llhttp__internal__n_span_end_llhttp__on_url_10: {
    const unsigned char* start;
    int err;

    start = state->_span_pos0;
    state->_span_pos0 = NULL;
    err = llhttp__on_url(state, start, p);
    if (err != 0) {
      state->error = err;
      state->error_pos = (const char*) p;
      state->_current = (void*) (intptr_t) s_n_llhttp__internal__n_url_skip_lf_to_http09;
      return s_error;
    }
    goto s_n_llhttp__internal__n_url_skip_lf_to_http09;
    /* UNREACHABLE */;  // 不可达代码，表示此处的代码不会被执行
    abort();  // 终止程序执行
  }
  s_n_llhttp__internal__n_span_end_llhttp__on_url_11: {
    const unsigned char* start;
    int err;

    start = state->_span_pos0;
    state->_span_pos0 = NULL;
    err = llhttp__on_url(state, start, p);
    if (err != 0) {
      state->error = err;
      state->error_pos = (const char*) p;
      state->_current = (void*) (intptr_t) s_n_llhttp__internal__n_url_skip_to_http;
      return s_error;
    }
    goto s_n_llhttp__internal__n_url_skip_to_http;
    /* UNREACHABLE */;  // 不可达代码，表示此处的代码不会被执行
    abort();  // 终止程序执行
  }
  s_n_llhttp__internal__n_error_38: {
    state->error = 0x7;
    state->reason = "Invalid char in url query";
    state->error_pos = (const char*) p;
    state->_current = (void*) (intptr_t) s_error;
    return s_error;
    /* UNREACHABLE */;  // 不可达代码，表示此处的代码不会被执行
    // 中止程序
    abort();
  }
  // 设置错误状态为 0x7
  s_n_llhttp__internal__n_error_39: {
    state->error = 0x7;
    // 设置错误原因为 "Invalid char in url path"
    state->reason = "Invalid char in url path";
    // 设置错误位置为当前位置
    state->error_pos = (const char*) p;
    // 设置当前状态为 s_error
    state->_current = (void*) (intptr_t) s_error;
    // 返回 s_error 状态
    return s_error;
    /* UNREACHABLE */;
    // 中止程序
    abort();
  }
  // 处理 URL 结束的状态
  s_n_llhttp__internal__n_span_end_llhttp__on_url: {
    const unsigned char* start;
    int err;

    // 获取 URL 开始位置
    start = state->_span_pos0;
    state->_span_pos0 = NULL;
    // 调用 llhttp__on_url 处理 URL
    err = llhttp__on_url(state, start, p);
    // 如果处理出错
    if (err != 0) {
      // 设置错误状态
      state->error = err;
      // 设置错误位置
      state->error_pos = (const char*) p;
      // 设置当前状态为 s_n_llhttp__internal__n_url_skip_to_http09
      state->_current = (void*) (intptr_t) s_n_llhttp__internal__n_url_skip_to_http09;
      // 返回错误状态
      return s_error;
    }
    // 跳转到 s_n_llhttp__internal__n_url_skip_to_http09 状态
    goto s_n_llhttp__internal__n_url_skip_to_http09;
    /* UNREACHABLE */;
    // 中止程序
    abort();
  }
  // 处理 URL 结束的状态
  s_n_llhttp__internal__n_span_end_llhttp__on_url_1: {
    const unsigned char* start;
    int err;

    // 获取 URL 开始位置
    start = state->_span_pos0;
    state->_span_pos0 = NULL;
    // 调用 llhttp__on_url 处理 URL
    err = llhttp__on_url(state, start, p);
    // 如果处理出错
    if (err != 0) {
      // 设置错误状态
      state->error = err;
      // 设置错误位置
      state->error_pos = (const char*) p;
      // 设置当前状态为 s_n_llhttp__internal__n_url_skip_lf_to_http09
      state->_current = (void*) (intptr_t) s_n_llhttp__internal__n_url_skip_lf_to_http09;
      // 返回错误状态
      return s_error;
    }
    // 跳转到 s_n_llhttp__internal__n_url_skip_lf_to_http09 状态
    goto s_n_llhttp__internal__n_url_skip_lf_to_http09;
    /* UNREACHABLE */;
    // 中止程序
    abort();
  }
  // 处理 URL 结束的状态
  s_n_llhttp__internal__n_span_end_llhttp__on_url_2: {
    const unsigned char* start;
    int err;

    // 获取 URL 开始位置
    start = state->_span_pos0;
    state->_span_pos0 = NULL;
    // 调用 llhttp__on_url 处理 URL
    err = llhttp__on_url(state, start, p);
    // 如果处理出错
    if (err != 0) {
      // 设置错误状态
      state->error = err;
      // ��置错误位置
      state->error_pos = (const char*) p;
      // 设置当前状态为 s_n_llhttp__internal__n_url_skip_to_http
      state->_current = (void*) (intptr_t) s_n_llhttp__internal__n_url_skip_to_http;
      // 返回错误状态
      return s_error;
    }
    // 跳转到 s_n_llhttp__internal__n_url_skip_to_http 状态
    goto s_n_llhttp__internal__n_url_skip_to_http;
    /* UNREACHABLE */;
    // 中止程序
    abort();
  }
  // 处理 URL 结束的状态
  s_n_llhttp__internal__n_span_end_llhttp__on_url_12: {
    const unsigned char* start;
    int err;

    // 获取 URL 开始位置
    start = state->_span_pos0;
    state->_span_pos0 = NULL;
    // 调用 llhttp__on_url 处理 URL
    err = llhttp__on_url(state, start, p);
  # 如果 err 不等于 0，则将错误码赋值给 state->error
  if (err != 0) {
    state->error = err;
    # 将当前位置的指针转换为 const char* 类型，赋值给 state->error_pos
    state->error_pos = (const char*) p;
    # 将 void* 类型的指针转换为 intptr_t 类型，再转换为 void* 类型，赋值给 state->_current
    state->_current = (void*) (intptr_t) s_n_llhttp__internal__n_url_skip_to_http09;
    # 返回 s_error 状态
    return s_error;
  }
  # 跳转到标签 s_n_llhttp__internal__n_url_skip_to_http09
  goto s_n_llhttp__internal__n_url_skip_to_http09;
  # 不可达代码，中止程序
  /* UNREACHABLE */;
  abort();
  # 标签 s_n_llhttp__internal__n_span_end_llhttp__on_url_13
  s_n_llhttp__internal__n_span_end_llhttp__on_url_13: {
    # 定义无符号字符指针 start 和整型变量 err
    const unsigned char* start;
    int err;
    # 将 state->_span_pos0 的值赋给 start
    start = state->_span_pos0;
    state->_span_pos0 = NULL;
    # 调用 llhttp__on_url 函数，将返回值赋给 err
    err = llhttp__on_url(state, start, p);
    # 如果 err 不等于 0，则将错误码赋值给 state->error
    if (err != 0) {
      state->error = err;
      # 将当前位置的指针转换为 const char* 类型，赋值给 state->error_pos
      state->error_pos = (const char*) p;
      # 将 void* 类型的指针转换为 intptr_t 类型，再转换为 void* 类型，赋值给 state->_current
      state->_current = (void*) (intptr_t) s_n_llhttp__internal__n_url_skip_lf_to_http09;
      # 返回 s_error 状态
      return s_error;
    }
    # 跳转到标签 s_n_llhttp__internal__n_url_skip_lf_to_http09
    goto s_n_llhttp__internal__n_url_skip_lf_to_http09;
    # 不可达代码，中止程序
    /* UNREACHABLE */;
    abort();
  }
  # 标签 s_n_llhttp__internal__n_span_end_llhttp__on_url_14
  s_n_llhttp__internal__n_span_end_llhttp__on_url_14: {
    # 定义无符号字符指针 start 和整型变量 err
    const unsigned char* start;
    int err;
    # 将 state->_span_pos0 的值赋给 start
    start = state->_span_pos0;
    state->_span_pos0 = NULL;
    # 调用 llhttp__on_url 函数，将返回值赋给 err
    err = llhttp__on_url(state, start, p);
    # 如果 err 不等于 0，则将错误码赋值给 state->error
    if (err != 0) {
      state->error = err;
      # 将当前位置的指针转换为 const char* 类型，赋值给 state->error_pos
      state->error_pos = (const char*) p;
      # 将 void* 类型的指针转换为 intptr_t 类型，再转换为 void* 类型，赋值给 state->_current
      state->_current = (void*) (intptr_t) s_n_llhttp__internal__n_url_skip_to_http;
      # 返回 s_error 状���
      return s_error;
    }
    # 跳转到标签 s_n_llhttp__internal__n_url_skip_to_http
    goto s_n_llhttp__internal__n_url_skip_to_http;
    # 不可达代码，中止程序
    /* UNREACHABLE */;
    abort();
  }
  # 标签 s_n_llhttp__internal__n_error_40
  s_n_llhttp__internal__n_error_40: {
    # 将错误码 0x7 赋值给 state->error
    state->error = 0x7;
    # 将字符串 "Double @ in url" 赋值给 state->reason
    state->reason = "Double @ in url";
    # 将当前位置的指针转换为 const char* 类型，赋值给 state->error_pos
    state->error_pos = (const char*) p;
    # 将 void* 类型的指针转换为 intptr_t 类型，再转换为 void* 类型，赋值给 state->_current
    state->_current = (void*) (intptr_t) s_error;
    # 返回 s_error 状态
    return s_error;
    # 不可达代码，中止程序
    /* UNREACHABLE */;
    abort();
  }
  # 标签 s_n_llhttp__internal__n_error_41
  s_n_llhttp__internal__n_error_41: {
    # 将错误码 0x7 赋值给 state->error
    state->error = 0x7;
    # 将字符串 "Unexpected char in url server" 赋值给 state->reason
    state->reason = "Unexpected char in url server";
    # 将当前位置的指针转换为 const char* 类型，赋值给 state->error_pos
    state->error_pos = (const char*) p;
    # 将 void* 类型的指针转换为 intptr_t 类型，再转换为 void* 类型，赋值给 state->_current
    state->_current = (void*) (intptr_t) s_error;
    # 返回 s_error 状态
    return s_error;
    # 不可达代码，中止程序
    /* UNREACHABLE */;
    abort();
  }
  # 标签 s_n_llhttp__internal__n_error_42
  s_n_llhttp__internal__n_error_42: {
    # 将错误码 0x7 赋值给 state->error
    state->error = 0x7;
    # 将当前位置的指针转换为 const char* 类型，赋值给 state->error_pos
    state->error_pos = (const char*) p;
    state->_current = (void*) (intptr_t) s_error;
    return s_error;
    /* UNREACHABLE */;
    abort();
  }
  s_n_llhttp__internal__n_error_44: {
    state->error = 0x7;
    state->reason = "Unexpected char in url schema";
    state->error_pos = (const char*) p;
    state->_current = (void*) (intptr_t) s_error;
    return s_error;
    /* UNREACHABLE */;
    abort();
  }
  s_n_llhttp__internal__n_error_45: {
    state->error = 0x7;
    state->reason = "Unexpected char in url schema";
    state->error_pos = (const char*) p;
    state->_current = (void*) (intptr_t) s_error;
    return s_error;
    /* UNREACHABLE */;
    abort();
  }
  s_n_llhttp__internal__n_error_46: {
    state->error = 0x7;
    state->reason = "Unexpected start char in url";
    state->error_pos = (const char*) p;
    state->_current = (void*) (intptr_t) s_error;
    return s_error;
    /* UNREACHABLE */;
    abort();
  }
  s_n_llhttp__internal__n_invoke_is_equal_method: {
    switch (llhttp__internal__c_is_equal_method(state, p, endp)) {
      case 0:
        goto s_n_llhttp__internal__n_url_entry_normal;
      default:
        goto s_n_llhttp__internal__n_url_entry_connect;
    }
    /* UNREACHABLE */;
    abort();
  }
  s_n_llhttp__internal__n_error_47: {
    state->error = 0x6;
    state->reason = "Expected space after method";
    state->error_pos = (const char*) p;
    state->_current = (void*) (intptr_t) s_error;
    return s_error;
    /* UNREACHABLE */;
    abort();
  }
  s_n_llhttp__internal__n_invoke_store_method_1: {
    switch (llhttp__internal__c_store_method(state, p, endp, match)) {
      default:
        goto s_n_llhttp__internal__n_req_first_space_before_url;
    }
    /* UNREACHABLE */;
    abort();
  }
  s_n_llhttp__internal__n_error_56: {
    state->error = 0x6;
    state->reason = "Invalid method encountered";
    state->error_pos = (const char*) p;
    state->_current = (void*) (intptr_t) s_error;
    return s_error;
    /* UNREACHABLE */;
    // 中止程序
    abort();
  }
  // 设置状态错误码和原因
  s_n_llhttp__internal__n_error_48: {
    state->error = 0xd;
    state->reason = "Response overflow";
    state->error_pos = (const char*) p;
    state->_current = (void*) (intptr_t) s_error;
    return s_error;
    /* UNREACHABLE */;
    // 中止程序
    abort();
  }
  // 调用内部函数处理状态码
  s_n_llhttp__internal__n_invoke_mul_add_status_code: {
    switch (llhttp__internal__c_mul_add_status_code(state, p, endp, match)) {
      case 1:
        goto s_n_llhttp__internal__n_error_48;
      default:
        goto s_n_llhttp__internal__n_res_status_code;
    }
    /* UNREACHABLE */;
    // 中止程序
    abort();
  }
  // 设置状态错误码和原因
  s_n_llhttp__internal__n_error_49: {
    state->error = 0x2;
    state->reason = "Expected LF after CR";
    state->error_pos = (const char*) p;
    state->_current = (void*) (intptr_t) s_error;
    return s_error;
    /* UNREACHABLE */;
    // 中止程序
    abort();
  }
  // 处理状态码结束
  s_n_llhttp__internal__n_span_end_llhttp__on_status: {
    const unsigned char* start;
    int err;

    start = state->_span_pos0;
    state->_span_pos0 = NULL;
    err = llhttp__on_status(state, start, p);
    if (err != 0) {
      state->error = err;
      state->error_pos = (const char*) (p + 1);
      state->_current = (void*) (intptr_t) s_n_llhttp__internal__n_invoke_llhttp__on_status_complete;
      return s_error;
    }
    p++;
    goto s_n_llhttp__internal__n_invoke_llhttp__on_status_complete;
    /* UNREACHABLE */;
    // 中止程序
    abort();
  }
  // 处理状态码结束
  s_n_llhttp__internal__n_span_end_llhttp__on_status_1: {
    const unsigned char* start;
    int err;

    start = state->_span_pos0;
    state->_span_pos0 = NULL;
    err = llhttp__on_status(state, start, p);
    if (err != 0) {
      state->error = err;
      state->error_pos = (const char*) (p + 1);
      state->_current = (void*) (intptr_t) s_n_llhttp__internal__n_res_line_almost_done;
      return s_error;
    }
    p++;
    goto s_n_llhttp__internal__n_res_line_almost_done;
    /* UNREACHABLE */;
    // 中止程序
    abort();
  }
  // 设置状态错误码
  s_n_llhttp__internal__n_error_50: {
    state->error = 0xd;
    state->reason = "Invalid response status";
    state->error_pos = (const char*) p;
    state->_current = (void*) (intptr_t) s_error;
    return s_error;
    /* UNREACHABLE */;
    abort();
  }
  s_n_llhttp__internal__n_invoke_update_status_code: {
    switch (llhttp__internal__c_update_status_code(state, p, endp)) {
      default:
        goto s_n_llhttp__internal__n_res_status_code;
    }
    /* UNREACHABLE */;
    abort();
  }
  s_n_llhttp__internal__n_error_51: {
    state->error = 0x9;
    state->reason = "Expected space after version";
    state->error_pos = (const char*) p;
    state->_current = (void*) (intptr_t) s_error;
    return s_error;
    /* UNREACHABLE */;
    abort();
  }
  s_n_llhttp__internal__n_invoke_store_http_minor_1: {
    switch (llhttp__internal__c_store_http_minor(state, p, endp, match)) {
      default:
        goto s_n_llhttp__internal__n_res_http_end;
    }
    /* UNREACHABLE */;
    abort();
  }
  s_n_llhttp__internal__n_error_52: {
    state->error = 0x9;
    state->reason = "Invalid minor version";
    state->error_pos = (const char*) p;
    state->_current = (void*) (intptr_t) s_error;
    return s_error;
    /* UNREACHABLE */;
    abort();
  }
  s_n_llhttp__internal__n_error_53: {
    state->error = 0x9;
    state->reason = "Expected dot";
    state->error_pos = (const char*) p;
    state->_current = (void*) (intptr_t) s_error;
    return s_error;
    /* UNREACHABLE */;
    abort();
  }
  s_n_llhttp__internal__n_invoke_store_http_major_1: {
    switch (llhttp__internal__c_store_http_major(state, p, endp, match)) {
      default:
        goto s_n_llhttp__internal__n_res_http_dot;
    }
    /* UNREACHABLE */;
    abort();
  }
  s_n_llhttp__internal__n_error_54: {
    state->error = 0x9;
    state->reason = "Invalid major version";
    state->error_pos = (const char*) p;
    state->_current = (void*) (intptr_t) s_error;
    return s_error;
    /* UNREACHABLE */;
    abort();
  }
  s_n_llhttp__internal__n_error_57: {
    // 设置状态的错误码为 0x8
    state->error = 0x8;
    // 设置状态的原因为 "Expected HTTP/"
    state->reason = "Expected HTTP/";
    // 设置状态的错误位置为当前指针位置
    state->error_pos = (const char*) p;
    // 设置状态的当前状态为 s_error
    state->_current = (void*) (intptr_t) s_error;
    // 返回 s_error 状态
    return s_error;
    /* UNREACHABLE */;
    // 不可达代码，中断程序执行
    abort();
  }
  s_n_llhttp__internal__n_invoke_update_type: {
    // 调用 llhttp__internal__c_update_type 函数，根据返回值进行不同的处理
    switch (llhttp__internal__c_update_type(state, p, endp)) {
      default:
        // 默认情况下跳转到 s_n_llhttp__internal__n_req_first_space_before_url 状态
        goto s_n_llhttp__internal__n_req_first_space_before_url;
    }
    /* UNREACHABLE */;
    // 不可达代码，中断程序执行
    abort();
  }
  // 其余部分同上，根据代码逐行添加注释
    // 调用中止函数
    abort();
  }
  // 当解析器开始解析消息时，调用 llhttp__on_message_begin 函数
  s_n_llhttp__internal__n_invoke_llhttp__on_message_begin: {
    // 根据 llhttp__on_message_begin 函数的返回值进行不同的处理
    switch (llhttp__on_message_begin(state, p, endp)) {
      // 如果返回值为0，跳转到加载类型的状态
      case 0:
        goto s_n_llhttp__internal__n_invoke_load_type;
      // 如果返回值为21，跳转到暂停状态8
      case 21:
        goto s_n_llhttp__internal__n_pause_8;
      // 其他情况，跳转到错误状态
      default:
        goto s_n_llhttp__internal__n_error;
    }
    /* UNREACHABLE */;
    // 调用中止函数
    abort();
  }
  // 调用 llhttp__internal__c_update_finish 函数更新解析器状态
  s_n_llhttp__internal__n_invoke_update_finish: {
    // 根据 llhttp__internal__c_update_finish 函数的返回值进行不同的处理
    switch (llhttp__internal__c_update_finish(state, p, endp)) {
      // 默认情况下，跳转到调用 llhttp__on_message_begin 函数的状态
      default:
        goto s_n_llhttp__internal__n_invoke_llhttp__on_message_begin;
    }
    /* UNREACHABLE */;
    // 调用中止函数
    abort();
  }
}

int llhttp__internal_execute(llhttp__internal_t* state, const char* p, const char* endp) {
  llparse_state_t next;

  /* 检查是否有悬而未决的错误 */
  if (state->error != 0) {
    return state->error;
  }

  /* 重新启动跨度 */
  if (state->_span_pos0 != NULL) {
    state->_span_pos0 = (void*) p;
  }

  next = llhttp__internal__run(state, (const unsigned char*) p, (const unsigned char*) endp);
  if (next == s_error) {
    return state->error;
  }
  state->_current = (void*) (intptr_t) next;

  /* 执行跨度 */
  if (state->_span_pos0 != NULL) {
    int error;

    error = ((llhttp__internal__span_cb) state->_span_cb0)(state, state->_span_pos0, (const char*) endp);
    if (error != 0) {
      state->error = error;
      state->error_pos = endp;
      return error;
    }
  }

  return 0;
}

#else  /* !LLHTTP_STRICT_MODE */

#include <stdlib.h>
#include <stdint.h>
#include <string.h>

#ifdef __SSE4_2__
 #ifdef _MSC_VER
  #include <nmmintrin.h>
 #else  /* !_MSC_VER */
  #include <x86intrin.h>
 #endif  /* _MSC_VER */
#endif  /* __SSE4_2__ */

#ifdef _MSC_VER
 #define ALIGN(n) _declspec(align(n))
#else  /* !_MSC_VER */
 #define ALIGN(n) __attribute__((aligned(n)))
#endif  /* _MSC_VER */

#include "llhttp.h"

typedef int (*llhttp__internal__span_cb)(
             llhttp__internal_t*, const char*, const char*);

#ifdef __SSE4_2__
static const unsigned char ALIGN(16) llparse_blob0[] = {
  0x9, 0x9, 0xc, 0xc, '!', '"', '$', '>', '@', '~', 0x80,
  0xff, 0x0, 0x0, 0x0, 0x0
};
#endif  /* __SSE4_2__ */
static const unsigned char llparse_blob1[] = {
  'o', 'n'
};
static const unsigned char llparse_blob2[] = {
  'e', 'c', 't', 'i', 'o', 'n'
};
static const unsigned char llparse_blob3[] = {
  'l', 'o', 's', 'e'
};
static const unsigned char llparse_blob4[] = {
  'e', 'e', 'p', '-', 'a', 'l', 'i', 'v', 'e'
};
static const unsigned char llparse_blob5[] = {
  'p', 'g', 'r', 'a', 'd', 'e'
};
static const unsigned char llparse_blob6[] = {
  'c', 'h', 'u', 'n', 'k', 'e', 'd'
};
#ifdef __SSE4_2__
static const unsigned char ALIGN(16) llparse_blob7[] = {
  0x9, 0x9, ' ', '~', 0x80, 0xff, 0x0, 0x0, 0x0, 0x0, 0x0,
  0x0, 0x0, 0x0, 0x0, 0x0
};
#endif  /* __SSE4_2__ */
#ifdef __SSE4_2__
static const unsigned char ALIGN(16) llparse_blob8[] = {
  ' ', '!', '#', '\'', '*', '+', '-', '.', '0', '9', 'A',
  'Z', '^', 'z', '|', '|'
};
#endif  /* __SSE4_2__ */
#ifdef __SSE4_2__
static const unsigned char ALIGN(16) llparse_blob9[] = {
  '~', '~', 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0,
  0x0, 0x0, 0x0, 0x0, 0x0
};
#endif  /* __SSE4_2__ */
static const unsigned char llparse_blob10[] = {
  'e', 'n', 't', '-', 'l', 'e', 'n', 'g', 't', 'h'
};
static const unsigned char llparse_blob11[] = {
  'r', 'o', 'x', 'y', '-', 'c', 'o', 'n', 'n', 'e', 'c',
  't', 'i', 'o', 'n'
};
static const unsigned char llparse_blob12[] = {
  'r', 'a', 'n', 's', 'f', 'e', 'r', '-', 'e', 'n', 'c',
  'o', 'd', 'i', 'n', 'g'
};
static const unsigned char llparse_blob13[] = {
  'p', 'g', 'r', 'a', 'd', 'e'
};
static const unsigned char llparse_blob14[] = {
  0xd, 0xa
};
static const unsigned char llparse_blob15[] = {
  'T', 'T', 'P', '/'
};
static const unsigned char llparse_blob16[] = {
  0xd, 0xa, 0xd, 0xa, 'S', 'M', 0xd, 0xa, 0xd, 0xa
};
static const unsigned char llparse_blob17[] = {
  'C', 'E', '/'
};
static const unsigned char llparse_blob18[] = {
  'T', 'S', 'P', '/'
};
static const unsigned char llparse_blob19[] = {
  'N', 'O', 'U', 'N', 'C', 'E'
};
static const unsigned char llparse_blob20[] = {
  'I', 'N', 'D'
};
static const unsigned char llparse_blob21[] = {
  'E', 'C', 'K', 'O', 'U', 'T'
};
static const unsigned char llparse_blob22[] = {
  'N', 'E', 'C', 'T'
};
static const unsigned char llparse_blob23[] = {
  'E', 'T', 'E'
};
static const unsigned char llparse_blob24[] = {
  'C', 'R', 'I', 'B', 'E'
};
static const unsigned char llparse_blob25[] = {
  'L', 'U', 'S', 'H'
};
static const unsigned char llparse_blob26[] = {
  'E', 'T'
};
# 定义一个名为 llparse_blob27 的常量，包含字符串 'PARAMETER'
static const unsigned char llparse_blob27[] = {
  'P', 'A', 'R', 'A', 'M', 'E', 'T', 'E', 'R'
};
# 定义一个名为 llparse_blob28 的常量，包含字符串 'EAD'
static const unsigned char llparse_blob28[] = {
  'E', 'A', 'D'
};
# 定义一个名为 llparse_blob29 的常量，包含字符串 'NK'
static const unsigned char llparse_blob29[] = {
  'N', 'K'
};
# 定义一个名为 llparse_blob30 的常量，包含字符串 'CK'
static const unsigned char llparse_blob30[] = {
  'C', 'K'
};
# 定义一个名为 llparse_blob31 的常量，包含字符串 'SEARCH'
static const unsigned char llparse_blob31[] = {
  'S', 'E', 'A', 'R', 'C', 'H'
};
# 定义一个名为 llparse_blob32 的常量，包含字符串 'RGE'
static const unsigned char llparse_blob32[] = {
  'R', 'G', 'E'
};
# 定义一个名为 llparse_blob33 的常量，包含字符串 'CTIVITY'
static const unsigned char llparse_blob33[] = {
  'C', 'T', 'I', 'V', 'I', 'T', 'Y'
};
# 定义一个名为 llparse_blob34 的常量，包含字符串 'LENDAR'
static const unsigned char llparse_blob34[] = {
  'L', 'E', 'N', 'D', 'A', 'R'
};
# 定义一个名为 llparse_blob35 的常量，包含字符串 'VE'
static const unsigned char llparse_blob35[] = {
  'V', 'E'
};
# 定义一个名为 llparse_blob36 的常量，包含字符串 'OTIFY'
static const unsigned char llparse_blob36[] = {
  'O', 'T', 'I', 'F', 'Y'
};
# 定义一个名为 llparse_blob37 的常量，包含字符串 'PTIONS'
static const unsigned char llparse_blob37[] = {
  'P', 'T', 'I', 'O', 'N', 'S'
};
# 定义一个名为 llparse_blob38 的常量，包含字符串 'CH'
static const unsigned char llparse_blob38[] = {
  'C', 'H'
};
# 定义一个名为 llparse_blob39 的常量，包含字符串 'SE'
static const unsigned char llparse_blob39[] = {
  'S', 'E'
};
# 定义一个名为 llparse_blob40 的常量，包含字符串 'AY'
static const unsigned char llparse_blob40[] = {
  'A', 'Y'
};
# 定义一个名为 llparse_blob41 的常量，包含字符串 'ST'
static const unsigned char llparse_blob41[] = {
  'S', 'T'
};
# 定义一个名为 llparse_blob42 的常量，包含字符串 'IND'
static const unsigned char llparse_blob42[] = {
  'I', 'N', 'D'
};
# 定义一个名为 llparse_blob43 的常量，包含字符串 'ATCH'
static const unsigned char llparse_blob43[] = {
  'A', 'T', 'C', 'H'
};
# 定义一个名为 llparse_blob44 的常量，包含字符串 'GE'
static const unsigned char llparse_blob44[] = {
  'G', 'E'
};
# 定义一个名为 llparse_blob45 的常量，包含字符串 'IND'
static const unsigned char llparse_blob45[] = {
  'I', 'N', 'D'
};
# 定义一个名��� llparse_blob46 的常量，包含字符串 'ORD'
static const unsigned char llparse_blob46[] = {
  'O', 'R', 'D'
};
# 定义一个名为 llparse_blob47 的常量，包含字符串 'IRECT'
static const unsigned char llparse_blob47[] = {
  'I', 'R', 'E', 'C', 'T'
};
# 定义一个名为 llparse_blob48 的常量，包含字符串 'ORT'
static const unsigned char llparse_blob48[] = {
  'O', 'R', 'T'
};
# 定义一个名为 llparse_blob49 的常量，包含字符串 'RCH'
static const unsigned char llparse_blob49[] = {
  'R', 'C', 'H'
};
# 定义一个名为 llparse_blob50 的常量，包含字符串 'PARAMETER'
static const unsigned char llparse_blob50[] = {
  'P', 'A', 'R', 'A', 'M', 'E', 'T', 'E', 'R'
};
# 定义一个名为 llparse_blob51 的常量，包含字符串 'URCE'
static const unsigned char llparse_blob51[] = {
  'U', 'R', 'C', 'E'
};
# 定义一个名为 llparse_blob52 的常量，包含字符串 'BSCRIBE'
static const unsigned char llparse_blob52[] = {
  'B', 'S', 'C', 'R', 'I', 'B', 'E'
};
# 定义一个名为 llparse_blob53 的常量，包含字符串 'ARDOWN'
static const unsigned char llparse_blob53[] = {
  'A', 'R', 'D', 'O', 'W', 'N'
};
# 定义包含字符 'ACE' 的无符号字符数组
static const unsigned char llparse_blob54[] = {
  'A', 'C', 'E'
};
# 定义包含字符 'IND' 的无符号字符数组
static const unsigned char llparse_blob55[] = {
  'I', 'N', 'D'
};
# 定义包含字符 'NK' 的无符号字符数组
static const unsigned char llparse_blob56[] = {
  'N', 'K'
};
# 定义包含字符 'CK' 的无符号字符数组
static const unsigned char llparse_blob57[] = {
  'C', 'K'
};
# 定义包含字符 'UBSCRIBE' 的无符号字符数组
static const unsigned char llparse_blob58[] = {
  'U', 'B', 'S', 'C', 'R', 'I', 'B', 'E'
};
# 定义包含字符 'HTTP/' 的无符号字符数组
static const unsigned char llparse_blob59[] = {
  'H', 'T', 'T', 'P', '/'
};
# 定义包含字符 'AD' 的无符号字符数组
static const unsigned char llparse_blob60[] = {
  'A', 'D'
};
# 定义包含字符 'TP/' 的无符号字符数组
static const unsigned char llparse_blob61[] = {
  'T', 'P', '/'
};

# 定义枚举类型 llparse_match_status_e，包含 kMatchComplete、kMatchPause、kMatchMismatch 三种状态
enum llparse_match_status_e {
  kMatchComplete,
  kMatchPause,
  kMatchMismatch
};
# 定义类型别名 llparse_match_status_t，表示枚举类型 llparse_match_status_e
typedef enum llparse_match_status_e llparse_match_status_t;

# 定义结构体 llparse_match_s，包含状态和当前指针
struct llparse_match_s {
  llparse_match_status_t status;
  const unsigned char* current;
};
# 定义类型别名 llparse_match_t，表示结构体 llparse_match_s
typedef struct llparse_match_s llparse_match_t;

# 定义函数 llparse__match_sequence_to_lower，用于匹配序列到小写
static llparse_match_t llparse__match_sequence_to_lower(
    llhttp__internal_t* s, const unsigned char* p,
    const unsigned char* endp,
    const unsigned char* seq, uint32_t seq_len) {
  uint32_t index;
  llparse_match_t res;

  index = s->_index;
  # 遍历输入序列，将大写字母转换为小写字母，进行匹配
  for (; p != endp; p++) {
    unsigned char current;

    current = ((*p) >= 'A' && (*p) <= 'Z' ? (*p | 0x20) : (*p));
    if (current == seq[index]) {
      if (++index == seq_len) {
        res.status = kMatchComplete;
        # 匹配完成，重置索引并返回结果
        goto reset;
      }
    } else {
      res.status = kMatchMismatch;
      # 匹配失败，重置索引并返回结果
      goto reset;
    }
  }
  s->_index = index;
  res.status = kMatchPause;
  res.current = p;
  return res;
reset:
  s->_index = 0;
  res.current = p;
  return res;
}

# 定义函数 llparse__match_sequence_to_lower_unsafe，用于不安全地匹配序列到小写
static llparse_match_t llparse__match_sequence_to_lower_unsafe(
    llhttp__internal_t* s, const unsigned char* p,
    const unsigned char* endp,
    const unsigned char* seq, uint32_t seq_len) {
  uint32_t index;
  llparse_match_t res;

  index = s->_index;
  # 遍历输入序列，将字符转换为小写字母，进行匹配
  for (; p != endp; p++) {
    unsigned char current;

    current = ((*p) | 0x20);
    # 如果当前字符等于序列中的字符
    if (current == seq[index]) {
      # 如果当前字符等于序列中的字符，则将索引加一
      if (++index == seq_len) {
        # 如果索引加一后等于序列长度，则匹配完成，状态设置为匹配完成，跳转到重置状态
        res.status = kMatchComplete;
        goto reset;
      }
    } else {
      # 如果当前字符不等于序列中的字符，则状态设置为不匹配，跳转到重置状态
      res.status = kMatchMismatch;
      goto reset;
    }
  }
  # 更新索引
  s->_index = index;
  # 状态设置为暂停匹配
  res.status = kMatchPause;
  # 当前字符设置为 p
  res.current = p;
  # 返回结果
  return res;
# 重置函数，将索引重置为0，当前结果设置为p，然后返回结果
reset:
  s->_index = 0;
  res.current = p;
  return res;
}

# 匹配序列ID的函数，接受参数s、p、endp、seq和seq_len
static llparse_match_t llparse__match_sequence_id(
    llhttp__internal_t* s, const unsigned char* p,
    const unsigned char* endp,
    const unsigned char* seq, uint32_t seq_len) {
  uint32_t index;
  llparse_match_t res;

  index = s->_index;
  # 遍历p直到endp
  for (; p != endp; p++) {
    unsigned char current;

    current = *p;
    # 如果当前字符等于序列中的字符
    if (current == seq[index]) {
      # 如果索引加一等于序列长度，表示匹配完成，跳转到reset标签
      if (++index == seq_len) {
        res.status = kMatchComplete;
        goto reset;
      }
    } else {
      # 如果当前字符不等于序列中的字符，表示不匹配，跳转到reset标签
      res.status = kMatchMismatch;
      goto reset;
    }
  }
  # 更新索引和状态，返回结果
  s->_index = index;
  res.status = kMatchPause;
  res.current = p;
  return res;
reset:
  # 重置索引为0，当前结果设置为p，然后返回结果
  s->_index = 0;
  res.current = p;
  return res;
}

# 定义枚举类型llparse_state_t
typedef enum llparse_state_e llparse_state_t;

# 下面是一系列函数，用于处理不同的HTTP请求部分，包括URL、header字段、header值、body、状态等

# 更新完成状态的函数，将状态的完成标志设置为2，然后返回0
int llhttp__internal__c_update_finish(
    llhttp__internal_t* state,
    const unsigned char* p,
    const unsigned char* endp) {
  state->finish = 2;
  return 0;
}

# 加载类型的函数，返回状态的类型
int llhttp__internal__c_load_type(
    llhttp__internal_t* state,
    const unsigned char* p,
    const unsigned char* endp) {
  return state->type;
}

# 存储方法的函数，将匹配结果存储为状态的方法，然后返回0
int llhttp__internal__c_store_method(
    llhttp__internal_t* state,
    const unsigned char* p,
    const unsigned char* endp,
    int match) {
  state->method = match;
  return 0;
}

# 判断方法是否相等的函数
    # 定义一个指向llhttp__internal_t类型的指针state，指向解析器内部状态
    llhttp__internal_t* state,
    # 定义一个指向无符号字符型的指针p，指向待解析的数据的起始位置
    const unsigned char* p,
    # 定义一个指向无符号字符型的指针endp，指向待解析的数据的结束位置
    const unsigned char* endp) {
  # 返回解析器内部状态中的方法是否为5的布尔值
  return state->method == 5;
# 更新 HTTP 请求/响应的主版本号为0
int llhttp__internal__c_update_http_major(
    llhttp__internal_t* state,
    const unsigned char* p,
    const unsigned char* endp) {
  state->http_major = 0;
  return 0;
}

# 更新 HTTP 请求/响应的次版本号为9
int llhttp__internal__c_update_http_minor(
    llhttp__internal_t* state,
    const unsigned char* p,
    const unsigned char* endp) {
  state->http_minor = 9;
  return 0;
}

# 检查是否 URL 解析完成
int llhttp__on_url_complete(
    llhttp__internal_t* s, const unsigned char* p,
    const unsigned char* endp);

# 测试状态标志位是否满足条件
int llhttp__internal__c_test_flags(
    llhttp__internal_t* state,
    const unsigned char* p,
    const unsigned char* endp) {
  return (state->flags & 128) == 128;
}

# 检查是否分块数据解析完成
int llhttp__on_chunk_complete(
    llhttp__internal_t* s, const unsigned char* p,
    const unsigned char* endp);

# 检查是否消息解析完成
int llhttp__on_message_complete(
    llhttp__internal_t* s, const unsigned char* p,
    const unsigned char* endp);

# 检查是否升级协议
int llhttp__internal__c_is_equal_upgrade(
    llhttp__internal_t* state,
    const unsigned char* p,
    const unsigned char* endp) {
  return state->upgrade == 1;
}

# 在消息解析完成之后的处理
int llhttp__after_message_complete(
    llhttp__internal_t* s, const unsigned char* p,
    const unsigned char* endp);

# 更新解析状态为未完成
int llhttp__internal__c_update_finish_1(
    llhttp__internal_t* state,
    const unsigned char* p,
    const unsigned char* endp) {
  state->finish = 0;
  return 0;
}

# 测试宽松模式状态标志位是否满足条件
int llhttp__internal__c_test_lenient_flags(
    llhttp__internal_t* state,
    const unsigned char* p,
    const unsigned char* endp) {
  return (state->lenient_flags & 4) == 4;
}

# 测试状态标志位是否满足条件
int llhttp__internal__c_test_flags_1(
    llhttp__internal_t* state,
    const unsigned char* p,
    const unsigned char* endp) {
  return (state->flags & 544) == 544;
}

# 测试宽松模式状态标志位是否满足条件
int llhttp__internal__c_test_lenient_flags_1(
    llhttp__internal_t* state,
    const unsigned char* p,
    const unsigned char* endp) {
  return (state->lenient_flags & 2) == 2;
}

# 在解析头部完成之前的处理
int llhttp__before_headers_complete(
    llhttp__internal_t* s, const unsigned char* p,
    const unsigned char* endp);
# 处理 HTTP 头部解析完成的回调函数
int llhttp__on_headers_complete(
    llhttp__internal_t* s, const unsigned char* p,
    const unsigned char* endp);

# 处理 HTTP 头部解析完成后的回调函数
int llhttp__after_headers_complete(
    llhttp__internal_t* s, const unsigned char* p,
    const unsigned char* endp);

# 更新内容长度为 0
int llhttp__internal__c_update_content_length(
    llhttp__internal_t* state,
    const unsigned char* p,
    const unsigned char* endp) {
  state->content_length = 0;
  return 0;
}

# 将内容长度乘以 16，并检查溢出
int llhttp__internal__c_mul_add_content_length(
    llhttp__internal_t* state,
    const unsigned char* p,
    const unsigned char* endp,
    int match) {
  /* 乘法溢出 */
  if (state->content_length > 0xffffffffffffffffULL / 16) {
    return 1;
  }

  state->content_length *= 16;

  /* 加法溢出 */
  if (match >= 0) {
    if (state->content_length > 0xffffffffffffffffULL - match) {
      return 1;
    }
  } else {
    if (state->content_length < 0ULL - match) {
      return 1;
    }
  }
  state->content_length += match;
  return 0;
}

# 处理分块编码的头部回调函数
int llhttp__on_chunk_header(
    llhttp__internal_t* s, const unsigned char* p,
    const unsigned char* endp);

# 检查内容长度是否等于 0
int llhttp__internal__c_is_equal_content_length(
    llhttp__internal_t* state,
    const unsigned char* p,
    const unsigned char* endp) {
  return state->content_length == 0;
}

# 将状态标志位设置为 128
int llhttp__internal__c_or_flags(
    llhttp__internal_t* state,
    const unsigned char* p,
    const unsigned char* endp) {
  state->flags |= 128;
  return 0;
}

# 更新完成标志为 1
int llhttp__internal__c_update_finish_3(
    llhttp__internal_t* state,
    const unsigned char* p,
    const unsigned char* endp) {
  state->finish = 1;
  return 0;
}

# 将状态标志位设置为 64
int llhttp__internal__c_or_flags_1(
    llhttp__internal_t* state,
    const unsigned char* p,
    const unsigned char* endp) {
  state->flags |= 64;
  return 0;
}

# 更新升级标志为 1
int llhttp__internal__c_update_upgrade(
    llhttp__internal_t* state,
    const unsigned char* p,
    const unsigned char* endp) {
  state->upgrade = 1;
  return 0;
}

# 存储头部状态的回调函数
int llhttp__internal__c_store_header_state(
    # 设置状态机的头部状态为匹配状态
    state->header_state = match;
    # 返回0，表示匹配成功
    return 0;
# 定义一个函数，处理 header field 完成时的回调
int llhttp__on_header_field_complete(
    llhttp__internal_t* s, const unsigned char* p,
    const unsigned char* endp);

# 定义一个函数，加载 header 状态
int llhttp__internal__c_load_header_state(
    llhttp__internal_t* state,
    const unsigned char* p,
    const unsigned char* endp) {
  return state->header_state;
}

# 定义一个函数，处理 flags 3 的回调
int llhttp__internal__c_or_flags_3(
    llhttp__internal_t* state,
    const unsigned char* p,
    const unsigned char* endp) {
  state->flags |= 1;
  return 0;
}

# 定义一个函数，更新 header 状态
int llhttp__internal__c_update_header_state(
    llhttp__internal_t* state,
    const unsigned char* p,
    const unsigned char* endp) {
  state->header_state = 1;
  return 0;
}

# 定义一个函数，处理 header value 完成时的回调
int llhttp__on_header_value_complete(
    llhttp__internal_t* s, const unsigned char* p,
    const unsigned char* endp);

# 定义一个函数，处理 flags 4 的回调
int llhttp__internal__c_or_flags_4(
    llhttp__internal_t* state,
    const unsigned char* p,
    const unsigned char* endp) {
  state->flags |= 2;
  return 0;
}

# 定义一个函数，处理 flags 5 的回调
int llhttp__internal__c_or_flags_5(
    llhttp__internal_t* state,
    const unsigned char* p,
    const unsigned char* endp) {
  state->flags |= 4;
  return 0;
}

# 定义一个函数，处理 flags 6 的回调
int llhttp__internal__c_or_flags_6(
    llhttp__internal_t* state,
    const unsigned char* p,
    const unsigned char* endp) {
  state->flags |= 8;
  return 0;
}

# 定义一个函数，更新 header 状态为 6
int llhttp__internal__c_update_header_state_2(
    llhttp__internal_t* state,
    const unsigned char* p,
    const unsigned char* endp) {
  state->header_state = 6;
  return 0;
}

# 定义一个函数，测试 lenient flags 2
int llhttp__internal__c_test_lenient_flags_2(
    llhttp__internal_t* state,
    const unsigned char* p,
    const unsigned char* endp) {
  return (state->lenient_flags & 1) == 1;
}

# 定义一个函数，更新 header 状态为 0
int llhttp__internal__c_update_header_state_4(
    llhttp__internal_t* state,
    const unsigned char* p,
    const unsigned char* endp) {
  state->header_state = 0;
  return 0;
}

# 定义一个函数，更新 header 状态为 5
int llhttp__internal__c_update_header_state_5(
    llhttp__internal_t* state,
    const unsigned char* p,
    const unsigned char* endp) {
  state->header_state = 5;
  return 0;
}
# 更新头部状态为7
int llhttp__internal__c_update_header_state_6(
    llhttp__internal_t* state,
    const unsigned char* p,
    const unsigned char* endp) {
  state->header_state = 7;
  return 0;
}

# 检查状态标志位是否包含32
int llhttp__internal__c_test_flags_2(
    llhttp__internal_t* state,
    const unsigned char* p,
    const unsigned char* endp) {
  return (state->flags & 32) == 32;
}

# 计算并更新内容长度，检查是否溢出
int llhttp__internal__c_mul_add_content_length_1(
    llhttp__internal_t* state,
    const unsigned char* p,
    const unsigned char* endp,
    int match) {
  /* Multiplication overflow */
  if (state->content_length > 0xffffffffffffffffULL / 10) {
    return 1;
  }

  state->content_length *= 10;

  /* Addition overflow */
  if (match >= 0) {
    if (state->content_length > 0xffffffffffffffffULL - match) {
      return 1;
    }
  } else {
    if (state->content_length < 0ULL - match) {
      return 1;
    }
  }
  state->content_length += match;
  return 0;
}

# 将状态标志位设置为包含32
int llhttp__internal__c_or_flags_15(
    llhttp__internal_t* state,
    const unsigned char* p,
    const unsigned char* endp) {
  state->flags |= 32;
  return 0;
}

# 将状态标志位设置为包含512
int llhttp__internal__c_or_flags_16(
    llhttp__internal_t* state,
    const unsigned char* p,
    const unsigned char* endp) {
  state->flags |= 512;
  return 0;
}

# 将状态标志位设置为与-9进行按位与操作
int llhttp__internal__c_and_flags(
    llhttp__internal_t* state,
    const unsigned char* p,
    const unsigned char* endp) {
  state->flags &= -9;
  return 0;
}

# 更新头部状态为8
int llhttp__internal__c_update_header_state_7(
    llhttp__internal_t* state,
    const unsigned char* p,
    const unsigned char* endp) {
  state->header_state = 8;
  return 0;
}

# 将状态标志位设置为包含16
int llhttp__internal__c_or_flags_17(
    llhttp__internal_t* state,
    const unsigned char* p,
    const unsigned char* endp) {
  state->flags |= 16;
  return 0;
}

# 加载方法
int llhttp__internal__c_load_method(
    llhttp__internal_t* state,
    const unsigned char* p,
    const unsigned char* endp) {
  return state->method;
}

# 存储 HTTP 主版本号
int llhttp__internal__c_store_http_major(
    llhttp__internal_t* state,
    # 定义函数参数，指向无符号字符的指针p
    const unsigned char* p,
    # 定义函数参数，指向无符号字符的指针endp
    const unsigned char* endp,
    # 定义函数参数，整型match
    int match) {
  # 将match的值赋给state结构体中的http_major成员
  state->http_major = match;
  # 返回0
  return 0;
}

// 更新 HTTP 版本号的次版本号
int llhttp__internal__c_store_http_minor(
    llhttp__internal_t* state,
    const unsigned char* p,
    const unsigned char* endp,
    int match) {
  state->http_minor = match;
  return 0;
}

// 更新状态码为 0
int llhttp__internal__c_update_status_code(
    llhttp__internal_t* state,
    const unsigned char* p,
    const unsigned char* endp) {
  state->status_code = 0;
  return 0;
}

// 乘法溢出检查并更新状态码
int llhttp__internal__c_mul_add_status_code(
    llhttp__internal_t* state,
    const unsigned char* p,
    const unsigned char* endp,
    int match) {
  /* 乘法溢出 */
  if (state->status_code > 0xffff / 10) {
    return 1;
  }

  state->status_code *= 10;

  /* 加法溢出 */
  if (match >= 0) {
    if (state->status_code > 0xffff - match) {
      return 1;
    }
  } else {
    if (state->status_code < 0 - match) {
      return 1;
    }
  }
  state->status_code += match;

  /* 强制最大值 */
  if (state->status_code > 999) {
    return 1;
  }
  return 0;
}

// 状态码解析完成的回调函数声明
int llhttp__on_status_complete(
    llhttp__internal_t* s, const unsigned char* p,
    const unsigned char* endp);

// 更新解析类型为 1
int llhttp__internal__c_update_type(
    llhttp__internal_t* state,
    const unsigned char* p,
    const unsigned char* endp) {
  state->type = 1;
  return 0;
}

// 更新解析类型为 2
int llhttp__internal__c_update_type_1(
    llhttp__internal_t* state,
    const unsigned char* p,
    const unsigned char* endp) {
  state->type = 2;
  return 0;
}

// 初始化内部状态
int llhttp__internal_init(llhttp__internal_t* state) {
  memset(state, 0, sizeof(*state));
  state->_current = (void*) (intptr_t) s_n_llhttp__internal__n_start;
  return 0;
}

// 运行内部状态机
static llparse_state_t llhttp__internal__run(
    llhttp__internal_t* state,
    const unsigned char* p,
    const unsigned char* endp) {
  int match;
  switch ((llparse_state_t) (intptr_t) state->_current) {
    case s_n_llhttp__internal__n_closed:
    s_n_llhttp__internal__n_closed: {
      // 如果指针已经到达结束位置，返回关闭状态
      if (p == endp) {
        return s_n_llhttp__internal__n_closed;
      }
      // 指针向后移动一位
      p++;
      // 跳转到 s_n_llhttp__internal__n_closed 状态
      goto s_n_llhttp__internal__n_closed;
      /* UNREACHABLE */;
      // 终止程序
      abort();
    }
    case s_n_llhttp__internal__n_invoke_llhttp__after_message_complete:
    s_n_llhttp__internal__n_invoke_llhttp__after_message_complete: {
      // 调用 llhttp__after_message_complete 函数，并根据返回值进行不同的跳转
      switch (llhttp__after_message_complete(state, p, endp)) {
        case 1:
          // 跳转到 s_n_llhttp__internal__n_invoke_update_finish_2 状态
          goto s_n_llhttp__internal__n_invoke_update_finish_2;
        default:
          // 跳转到 s_n_llhttp__internal__n_invoke_update_finish_1 状态
          goto s_n_llhttp__internal__n_invoke_update_finish_1;
      }
      /* UNREACHABLE */;
      // 终止程序
      abort();
    }
    case s_n_llhttp__internal__n_pause_1:
    s_n_llhttp__internal__n_pause_1: {
      // 设置状态的错误码和原因，并返回错误状态
      state->error = 0x16;
      state->reason = "Pause on CONNECT/Upgrade";
      state->error_pos = (const char*) p;
      state->_current = (void*) (intptr_t) s_n_llhttp__internal__n_invoke_llhttp__after_message_complete;
      return s_error;
      /* UNREACHABLE */;
      // 终止程序
      abort();
    }
    case s_n_llhttp__internal__n_invoke_is_equal_upgrade:
    s_n_llhttp__internal__n_invoke_is_equal_upgrade: {
      // 调用 llhttp__internal__c_is_equal_upgrade 函数，并根据返回值进行不同的跳转
      switch (llhttp__internal__c_is_equal_upgrade(state, p, endp)) {
        case 0:
          // 跳转到 s_n_llhttp__internal__n_invoke_llhttp__after_message_complete 状态
          goto s_n_llhttp__internal__n_invoke_llhttp__after_message_complete;
        default:
          // 跳转到 s_n_llhttp__internal__n_pause_1 状态
          goto s_n_llhttp__internal__n_pause_1;
      }
      /* UNREACHABLE */;
      // 终止程序
      abort();
    }
    case s_n_llhttp__internal__n_invoke_llhttp__on_message_complete_2:
    s_n_llhttp__internal__n_invoke_llhttp__on_message_complete_2: {
      // 调用 llhttp__on_message_complete 函数，并根据返回值进行不同的跳转
      switch (llhttp__on_message_complete(state, p, endp)) {
        case 0:
          // 跳转到 s_n_llhttp__internal__n_invoke_is_equal_upgrade 状态
          goto s_n_llhttp__internal__n_invoke_is_equal_upgrade;
        case 21:
          // 跳转到 s_n_llhttp__internal__n_pause_5 状态
          goto s_n_llhttp__internal__n_pause_5;
        default:
          // 跳转到 s_n_llhttp__internal__n_error_9 状态
          goto s_n_llhttp__internal__n_error_9;
      }
      /* UNREACHABLE */;
      // 终止程序
      abort();
    }
    case s_n_llhttp__internal__n_chunk_data_almost_done_skip:
    # 如果指针 p 等于结束指针 endp，则返回 s_n_llhttp__internal__n_chunk_data_almost_done_skip 状态
    s_n_llhttp__internal__n_chunk_data_almost_done_skip: {
      if (p == endp) {
        return s_n_llhttp__internal__n_chunk_data_almost_done_skip;
      }
      # 指针 p 向前移动一位
      p++;
      # 跳转到 s_n_llhttp__internal__n_invoke_llhttp__on_chunk_complete 状态
      goto s_n_llhttp__internal__n_invoke_llhttp__on_chunk_complete;
      /* UNREACHABLE */;
      # 中止程序
      abort();
    }
    # 当前状态为 s_n_llhttp__internal__n_chunk_data_almost_done
    case s_n_llhttp__internal__n_chunk_data_almost_done:
    s_n_llhttp__internal__n_chunk_data_almost_done: {
      # 如果指针 p 等于结束指针 endp，则返回 s_n_llhttp__internal__n_chunk_data_almost_done 状态
      if (p == endp) {
        return s_n_llhttp__internal__n_chunk_data_almost_done;
      }
      # 指针 p 向前移动一位
      p++;
      # 跳转到 s_n_llhttp__internal__n_chunk_data_almost_done_skip 状态
      goto s_n_llhttp__internal__n_chunk_data_almost_done_skip;
      /* UNREACHABLE */;
      # 中止程序
      abort();
    }
    # 当前状态为 s_n_llhttp__internal__n_consume_content_length
    case s_n_llhttp__internal__n_consume_content_length:
    s_n_llhttp__internal__n_consume_content_length: {
      # 定义变量 avail 和 need
      size_t avail;
      size_t need;

      # 计算可用长度和需要长度
      avail = endp - p;
      need = state->content_length;
      # 如果可用长度大于等于需要长度
      if (avail >= need) {
        # 指针 p 向前移动需要长度
        p += need;
        # 将状态的 content_length 设为 0
        state->content_length = 0;
        # 跳转到 s_n_llhttp__internal__n_span_end_llhttp__on_body 状态
        goto s_n_llhttp__internal__n_span_end_llhttp__on_body;
      }

      # 状态的 content_length 减去可用长度
      state->content_length -= avail;
      # 返回 s_n_llhttp__internal__n_consume_content_length 状态
      return s_n_llhttp__internal__n_consume_content_length;
      /* UNREACHABLE */;
      # 中止程序
      abort();
    }
    # 当前状态为 s_n_llhttp__internal__n_span_start_llhttp__on_body
    case s_n_llhttp__internal__n_span_start_llhttp__on_body:
    s_n_llhttp__internal__n_span_start_llhttp__on_body: {
      # 如果指针 p 等于结束指针 endp，则返回 s_n_llhttp__internal__n_span_start_llhttp__on_body 状态
      if (p == endp) {
        return s_n_llhttp__internal__n_span_start_llhttp__on_body;
      }
      # 将指针 p 的位置赋给状态的 _span_pos0
      state->_span_pos0 = (void*) p;
      # 将 llhttp__on_body 赋给状态的 _span_cb0
      state->_span_cb0 = llhttp__on_body;
      # 跳转到 s_n_llhttp__internal__n_consume_content_length 状态
      goto s_n_llhttp__internal__n_consume_content_length;
      /* UNREACHABLE */;
      # 中止程序
      abort();
    }
    # 当前状态为 s_n_llhttp__internal__n_invoke_is_equal_content_length
    case s_n_llhttp__internal__n_invoke_is_equal_content_length:
    s_n_llhttp__internal__n_invoke_is_equal_content_length: {
      # 根据 llhttp__internal__c_is_equal_content_length 的返回值进行跳转
      switch (llhttp__internal__c_is_equal_content_length(state, p, endp)) {
        # 如果返回值为 0，则跳转到 s_n_llhttp__internal__n_span_start_llhttp__on_body
        case 0:
          goto s_n_llhttp__internal__n_span_start_llhttp__on_body;
        # 否则跳转到 s_n_llhttp__internal__n_invoke_or_flags
        default:
          goto s_n_llhttp__internal__n_invoke_or_flags;
      }
      /* UNREACHABLE */;
      # 中止程序
      abort();
    }
    # 当前状态为 s_n_llhttp__internal__n_chunk_size_almost_done
    # 如果当前状态为 s_n_llhttp__internal__n_chunk_size_almost_done，则执行以下操作
    s_n_llhttp__internal__n_chunk_size_almost_done: {
      # 如果指针 p 等于结束指针 endp，则返回当前状态 s_n_llhttp__internal__n_chunk_size_almost_done
      if (p == endp) {
        return s_n_llhttp__internal__n_chunk_size_almost_done;
      }
      # 指针 p 向后移动一位
      p++;
      # 跳转到状态 s_n_llhttp__internal__n_invoke_llhttp__on_chunk_header
      goto s_n_llhttp__internal__n_invoke_llhttp__on_chunk_header;
      /* UNREACHABLE */;
      # 中止程序
      abort();
    }
    # 如果当前状态为 s_n_llhttp__internal__n_chunk_parameters，则执行以下操作
    case s_n_llhttp__internal__n_chunk_parameters:
    s_n_llhttp__internal__n_chunk_parameters: {
      # 如果指针 p 等于结束指针 endp，则返回当前状态 s_n_llhttp__internal__n_chunk_parameters
      if (p == endp) {
        return s_n_llhttp__internal__n_chunk_parameters;
      }
      # 根据指针 p 指向的值进行不同的操作
      switch (*p) {
        # 如果指针 p 指向的值为 13，则执行以下操作
        case 13: {
          # 指针 p 向后移动一位
          p++;
          # 跳转到状态 s_n_llhttp__internal__n_chunk_size_almost_done
          goto s_n_llhttp__internal__n_chunk_size_almost_done;
        }
        # 如果指针 p 指向的值为其他值，则执行以下操作
        default: {
          # 指针 p 向后移动一位
          p++;
          # 跳转到状态 s_n_llhttp__internal__n_chunk_parameters
          goto s_n_llhttp__internal__n_chunk_parameters;
        }
      }
      /* UNREACHABLE */;
      # 中止程序
      abort();
    }
    # 如果当前状态为 s_n_llhttp__internal__n_chunk_size_otherwise，则执行以下操作
    case s_n_llhttp__internal__n_chunk_size_otherwise:
    s_n_llhttp__internal__n_chunk_size_otherwise: {
      # 如果指针 p 等于结束指针 endp，则返回当前状态 s_n_llhttp__internal__n_chunk_size_otherwise
      if (p == endp) {
        return s_n_llhttp__internal__n_chunk_size_otherwise;
      }
      # 根据指针 p 指向的值进行不同的操作
      switch (*p) {
        # 如果指针 p 指向的值为 13，则执行以下操作
        case 13: {
          # 指针 p 向后移动一位
          p++;
          # 跳转到状态 s_n_llhttp__internal__n_chunk_size_almost_done
          goto s_n_llhttp__internal__n_chunk_size_almost_done;
        }
        # 如果指针 p 指向的值为 ' '，则执行以下操作
        case ' ': {
          # 指针 p 向后移动一位
          p++;
          # 跳转到状态 s_n_llhttp__internal__n_chunk_parameters
          goto s_n_llhttp__internal__n_chunk_parameters;
        }
        # 如果指针 p 指向的值为 ';'，则执行以下操作
        case ';': {
          # 指针 p 向后移动一位
          p++;
          # 跳转到状态 s_n_llhttp__internal__n_chunk_parameters
          goto s_n_llhttp__internal__n_chunk_parameters;
        }
        # 如果指针 p 指向的值为其他值，则执行以下操作
        default: {
          # 跳转到状态 s_n_llhttp__internal__n_error_6
          goto s_n_llhttp__internal__n_error_6;
        }
      }
      /* UNREACHABLE */;
      # 中止程序
      abort();
    }
    # 如果当前状态为 s_n_llhttp__internal__n_chunk_size，则执行以下操作
    case s_n_llhttp__internal__n_chunk_size:
    }
    # 如果当前状态为 s_n_llhttp__internal__n_chunk_size_digit，则执行以下操作
    case s_n_llhttp__internal__n_chunk_size_digit:
    }
    # 如果当前状态为 s_n_llhttp__internal__n_invoke_update_content_length，则执行以下操作
    case s_n_llhttp__internal__n_invoke_update_content_length:
    s_n_llhttp__internal__n_invoke_update_content_length: {
      # 根据 llhttp__internal__c_update_content_length 函数的返回值进行不同的操作
      switch (llhttp__internal__c_update_content_length(state, p, endp)) {
        # 默认情况下，跳转到状态 s_n_llhttp__internal__n_chunk_size_digit
        default:
          goto s_n_llhttp__internal__n_chunk_size_digit;
      }
      /* UNREACHABLE */;
      # 中止程序
      abort();
    }
    # 如果当前状态为 s_n_llhttp__internal__n_consume_content_length_1:
    # 定义状态 s_n_llhttp__internal__n_consume_content_length_1，用于消耗请求体的内容长度
    s_n_llhttp__internal__n_consume_content_length_1: {
      # 定义变量 avail 为可用内容长度，need 为需要的内容长度
      size_t avail;
      size_t need;

      # 计算可用内容长度和需要的内容长度
      avail = endp - p;
      need = state->content_length;
      # 如果可用内容长度大于等于需要的内容长度
      if (avail >= need) {
        # 移动指针 p 到需要的内容长度处
        p += need;
        # 将状态的内容长度设为 0，表示内容已经全部消耗
        state->content_length = 0;
        # 跳转到状态 s_n_llhttp__internal__n_span_end_llhttp__on_body_1
        goto s_n_llhttp__internal__n_span_end_llhttp__on_body_1;
      }

      # 如果可用内容长度小于需要的内容长度，更新状态的内容长度，继续消耗内容
      state->content_length -= avail;
      return s_n_llhttp__internal__n_consume_content_length_1;
      /* UNREACHABLE */;
      abort();
    }
    # 定义状态 s_n_llhttp__internal__n_span_start_llhttp__on_body_1
    case s_n_llhttp__internal__n_span_start_llhttp__on_body_1:
    s_n_llhttp__internal__n_span_start_llhttp__on_body_1: {
      # 如果指针 p 等于结束指针 endp，表示没有内容可读，返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_span_start_llhttp__on_body_1;
      }
      # 记录当前指针位置和回调函数，跳转到状态 s_n_llhttp__internal__n_consume_content_length_1
      state->_span_pos0 = (void*) p;
      state->_span_cb0 = llhttp__on_body;
      goto s_n_llhttp__internal__n_consume_content_length_1;
      /* UNREACHABLE */;
      abort();
    }
    # 定义状态 s_n_llhttp__internal__n_eof
    case s_n_llhttp__internal__n_eof:
    s_n_llhttp__internal__n_eof: {
      # 如果指针 p 等于结束指针 endp，表示没有内容可读，返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_eof;
      }
      # 移动指针 p，继续处理结束标记
      p++;
      goto s_n_llhttp__internal__n_eof;
      /* UNREACHABLE */;
      abort();
    }
    # 定义状态 s_n_llhttp__internal__n_span_start_llhttp__on_body_2
    case s_n_llhttp__internal__n_span_start_llhttp__on_body_2:
    s_n_llhttp__internal__n_span_start_llhttp__on_body_2: {
      # 如果指针 p 等于结束指针 endp，表示没有内容可读，返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_span_start_llhttp__on_body_2;
      }
      # 记录当前指针位置和回调函数，跳转到状态 s_n_llhttp__internal__n_eof
      state->_span_pos0 = (void*) p;
      state->_span_cb0 = llhttp__on_body;
      goto s_n_llhttp__internal__n_eof;
      /* UNREACHABLE */;
      abort();
    }
    # 定义状态 s_n_llhttp__internal__n_invoke_llhttp__after_headers_complete
    # 定义状态转移函数 s_n_llhttp__internal__n_invoke_llhttp__after_headers_complete
    s_n_llhttp__internal__n_invoke_llhttp__after_headers_complete: {
      # 根据状态和指针调用 llhttp__after_headers_complete 函数，根据返回值进行不同的状态转移
      switch (llhttp__after_headers_complete(state, p, endp)) {
        # 如果返回值为1，跳转到 s_n_llhttp__internal__n_invoke_llhttp__on_message_complete_1
        case 1:
          goto s_n_llhttp__internal__n_invoke_llhttp__on_message_complete_1;
        # 如果返回值为2，跳转到 s_n_llhttp__internal__n_invoke_update_content_length
        case 2:
          goto s_n_llhttp__internal__n_invoke_update_content_length;
        # 如果返回值为3，跳转到 s_n_llhttp__internal__n_span_start_llhttp__on_body_1
        case 3:
          goto s_n_llhttp__internal__n_span_start_llhttp__on_body_1;
        # 如果返回值为4，跳转到 s_n_llhttp__internal__n_invoke_update_finish_3
        case 4:
          goto s_n_llhttp__internal__n_invoke_update_finish_3;
        # 如果返回值为5，跳转到 s_n_llhttp__internal__n_error_10
        case 5:
          goto s_n_llhttp__internal__n_error_10;
        # 如果返回值为其他值，跳转到 s_n_llhttp__internal__n_invoke_llhttp__on_message_complete
        default:
          goto s_n_llhttp__internal__n_invoke_llhttp__on_message_complete;
      }
      # 不可达的代码，中止程序
      /* UNREACHABLE */;
      abort();
    }
    # 定义状态转移函数 s_n_llhttp__internal__n_headers_almost_done
    case s_n_llhttp__internal__n_headers_almost_done:
    s_n_llhttp__internal__n_headers_almost_done: {
      # 如果指针指向结束位置，返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_headers_almost_done;
      }
      # 指针后移一位
      p++;
      # 跳转到 s_n_llhttp__internal__n_invoke_test_flags
      goto s_n_llhttp__internal__n_invoke_test_flags;
      # 不可达的代码，中止程序
      /* UNREACHABLE */;
      abort();
    }
    # 定义状态转移函数 s_n_llhttp__internal__n_invoke_llhttp__on_header_value_complete
    case s_n_llhttp__internal__n_invoke_llhttp__on_header_value_complete:
    s_n_llhttp__internal__n_invoke_llhttp__on_header_value_complete: {
      # 根据状态和指针调用 llhttp__on_header_value_complete 函数，根据返回值进行状态转移
      switch (llhttp__on_header_value_complete(state, p, endp)) {
        # 默认情况下，跳转到 s_n_llhttp__internal__n_header_field_start
        default:
          goto s_n_llhttp__internal__n_header_field_start;
      }
      # 不可达的代码，中止程序
      /* UNREACHABLE */;
      abort();
    }
    # 定义状态转移函数 s_n_llhttp__internal__n_span_start_llhttp__on_header_value
    case s_n_llhttp__internal__n_span_start_llhttp__on_header_value:
    s_n_llhttp__internal__n_span_start_llhttp__on_header_value: {
      # 如果指针指向结束位置，返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_span_start_llhttp__on_header_value;
      }
      # 设置状态的 _span_pos0 和 _span_cb0 属性
      state->_span_pos0 = (void*) p;
      state->_span_cb0 = llhttp__on_header_value;
      # 跳转到 s_n_llhttp__internal__n_span_end_llhttp__on_header_value
      goto s_n_llhttp__internal__n_span_end_llhttp__on_header_value;
      # 不可达的代码，中止程序
      /* UNREACHABLE */;
      abort();
    }
    # 定义状态转移函数 s_n_llhttp__internal__n_header_value_discard_lws:
    # 定义状态机的状态 s_n_llhttp__internal__n_header_value_discard_lws
    s_n_llhttp__internal__n_header_value_discard_lws: {
      # 如果指针已经到达末尾，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_header_value_discard_lws;
      }
      # 根据当前字符进行不同的处理
      switch (*p) {
        # 如果是制表符，则指针向后移动一位，跳转到 s_n_llhttp__internal__n_header_value_discard_ws 状态
        case 9: {
          p++;
          goto s_n_llhttp__internal__n_header_value_discard_ws;
        }
        # 如果是空格，则指针向后移动一位，跳转到 s_n_llhttp__internal__n_header_value_discard_ws 状态
        case ' ': {
          p++;
          goto s_n_llhttp__internal__n_header_value_discard_ws;
        }
        # 如果是其他字符，则跳转到 s_n_llhttp__internal__n_invoke_load_header_state 状态
        default: {
          goto s_n_llhttp__internal__n_invoke_load_header_state;
        }
      }
      /* UNREACHABLE */;
      # 如果代码执行到这里，表示出现了不可达的情况，终止程序
      abort();
    }
    # 定义状态机的状态 s_n_llhttp__internal__n_header_value_discard_ws_almost_done
    case s_n_llhttp__internal__n_header_value_discard_ws_almost_done:
    s_n_llhttp__internal__n_header_value_discard_ws_almost_done: {
      # 如果指针已经到达末尾，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_header_value_discard_ws_almost_done;
      }
      # 指针向后移动一位，跳转到 s_n_llhttp__internal__n_header_value_discard_lws 状态
      p++;
      goto s_n_llhttp__internal__n_header_value_discard_lws;
      /* UNREACHABLE */;
      # 如果代码执行到这里，表示出现了不可达的情况，终止程序
      abort();
    }
    # 定义状态机的状态 s_n_llhttp__internal__n_header_value_lws
    case s_n_llhttp__internal__n_header_value_lws:
    s_n_llhttp__internal__n_header_value_lws: {
      # 如果指针已经到达末尾，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_header_value_lws;
      }
      # 根据当前字符进行不同的处理
      switch (*p) {
        # 如果是制表符，则跳转到 s_n_llhttp__internal__n_span_start_llhttp__on_header_value_1 状态
        case 9: {
          goto s_n_llhttp__internal__n_span_start_llhttp__on_header_value_1;
        }
        # 如果是空格，则跳转到 s_n_llhttp__internal__n_span_start_llhttp__on_header_value_1 状态
        case ' ': {
          goto s_n_llhttp__internal__n_span_start_llhttp__on_header_value_1;
        }
        # 如果是其他字符，则跳转到 s_n_llhttp__internal__n_invoke_load_header_state_3 状态
        default: {
          goto s_n_llhttp__internal__n_invoke_load_header_state_3;
        }
      }
      /* UNREACHABLE */;
      # 如果代码执行到这里，表示出现了不可达的情况，终止程序
      abort();
    }
    # 定义状态机的状态 s_n_llhttp__internal__n_header_value_almost_done
    case s_n_llhttp__internal__n_header_value_almost_done:
    s_n_llhttp__internal__n_header_value_almost_done: {
      # 如果指针已经到达末尾，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_header_value_almost_done;
      }
      # 根据当前字符进行不同的处理
      switch (*p) {
        # 如果是换行符，则指针向后移动一位，跳转到 s_n_llhttp__internal__n_header_value_lws 状态
        case 10: {
          p++;
          goto s_n_llhttp__internal__n_header_value_lws;
        }
        # 如果是其他字符，则跳转到 s_n_llhttp__internal__n_error_14 状态
        default: {
          goto s_n_llhttp__internal__n_error_14;
        }
      }
      /* UNREACHABLE */;
      # 如果代码执行到这里，表示出现了不可达的情况，终止程序
      abort();
    }
    # 定义状态机的状态 s_n_llhttp__internal__n_header_value_lenient:
    # 定义状态机的状态 s_n_llhttp__internal__n_header_value_lenient
    s_n_llhttp__internal__n_header_value_lenient: {
      # 如果指针已经指向结束位置，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_header_value_lenient;
      }
      # 根据当前字符进行不同的处理
      switch (*p) {
        # 如果是换行符，则跳转到处理换行符的状态
        case 10: {
          goto s_n_llhttp__internal__n_span_end_llhttp__on_header_value_1;
        }
        # 如果是回车符，则跳转到处理回车符的状态
        case 13: {
          goto s_n_llhttp__internal__n_span_end_llhttp__on_header_value_3;
        }
        # 如果是其他字符，则指针向后移动，继续处理当前状态
        default: {
          p++;
          goto s_n_llhttp__internal__n_header_value_lenient;
        }
      }
      # 不可达的代码，中断程序执行
      /* UNREACHABLE */;
      abort();
    }
    # 定义状态机的状态 s_n_llhttp__internal__n_header_value_otherwise
    case s_n_llhttp__internal__n_header_value_otherwise:
    s_n_llhttp__internal__n_header_value_otherwise: {
      # 如果指针已经指向结束位置，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_header_value_otherwise;
      }
      # 根据当前字符进行不同的处理
      switch (*p) {
        # 如果是换行符，则跳转到处理换行符的状态
        case 10: {
          goto s_n_llhttp__internal__n_span_end_llhttp__on_header_value_1;
        }
        # 如果是回车符，则跳转到处理回车符的状态
        case 13: {
          goto s_n_llhttp__internal__n_span_end_llhttp__on_header_value_2;
        }
        # 如果是其他字符，则跳转到处理其他字符的状态
        default: {
          goto s_n_llhttp__internal__n_invoke_test_lenient_flags_2;
        }
      }
      # 不可达的代码，中断程序执行
      /* UNREACHABLE */;
      abort();
    }
    # 定义状态机的状态 s_n_llhttp__internal__n_header_value_connection_token:
    case s_n_llhttp__internal__n_header_value_connection_token:
    # 定义静态数组 lookup_table，用于存储对应字符的状态
    s_n_llhttp__internal__n_header_value_connection_token: {
      static uint8_t lookup_table[] = {
        0, 0, 0, 0, 0, 0, 0, 0, 0, 1, 0, 0, 0, 0, 0, 0,
        0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0,
        1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 2, 1, 1, 1,
        1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1,
        1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1,
        1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1,
        1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 0,
        1, 1, 1, 1, 1, 1, 1
    # 如果当前指针已经到达结束位置，则返回当前状态
    s_n_llhttp__internal__n_header_value_connection_ws: {
      if (p == endp) {
        return s_n_llhttp__internal__n_header_value_connection_ws;
      }
      # 根据当前字符进行不同的处理
      switch (*p) {
        # 如果是换行符或回车符，则跳转到其他状态
        case 10: {
          goto s_n_llhttp__internal__n_header_value_otherwise;
        }
        case 13: {
          goto s_n_llhttp__internal__n_header_value_otherwise;
        }
        # 如果是空格，则移动指针并保持当前状态
        case ' ': {
          p++;
          goto s_n_llhttp__internal__n_header_value_connection_ws;
        }
        # 如果是逗号，则移动指针并跳转到其他状态
        case ',': {
          p++;
          goto s_n_llhttp__internal__n_invoke_load_header_state_4;
        }
        # 其他情况，则跳转到其他状态
        default: {
          goto s_n_llhttp__internal__n_invoke_update_header_state_4;
        }
      }
      # 不可达的代码，中止程序
      /* UNREACHABLE */;
      abort();
    }
    # 如果当前状态为 s_n_llhttp__internal__n_header_value_connection_1，则执行以下操作
    case s_n_llhttp__internal__n_header_value_connection_1:
    s_n_llhttp__internal__n_header_value_connection_1: {
      llparse_match_t match_seq;

      # 如果当前指针已经到达结束位置，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_header_value_connection_1;
      }
      # 尝试匹配指定序列，根据匹配结果执行不同的操作
      match_seq = llparse__match_sequence_to_lower(state, p, endp, llparse_blob3, 4);
      p = match_seq.current;
      switch (match_seq.status) {
        # 如果匹配完成，则移动指针并跳转到其他状态
        case kMatchComplete: {
          p++;
          goto s_n_llhttp__internal__n_invoke_update_header_state_2;
        }
        # 如果匹配暂停，则返回当前状态
        case kMatchPause: {
          return s_n_llhttp__internal__n_header_value_connection_1;
        }
        # 如果匹配不成功，则跳转到其他状态
        case kMatchMismatch: {
          goto s_n_llhttp__internal__n_header_value_connection_token;
        }
      }
      # 不可达的代码，中止程序
      /* UNREACHABLE */;
      abort();
    }
    # 如果当前状态为 s_n_llhttp__internal__n_header_value_connection_2，则执行以下操作
    case s_n_llhttp__internal__n_header_value_connection_2:
    # 定义状态 s_n_llhttp__internal__n_header_value_connection_2，处理 Connection 头部值的第二部分
    s_n_llhttp__internal__n_header_value_connection_2: {
      # 定义匹配结果变量
      llparse_match_t match_seq;

      # 如果指针已经到达结束位置，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_header_value_connection_2;
      }
      # 调用函数，匹配指定长度的字符串并转换为小写
      match_seq = llparse__match_sequence_to_lower(state, p, endp, llparse_blob4, 9);
      # 更新指针位置
      p = match_seq.current;
      # 根据匹配结果进行不同的处理
      switch (match_seq.status) {
        # 如果匹配完成，则继续处理下一个字符
        case kMatchComplete: {
          p++;
          goto s_n_llhttp__internal__n_invoke_update_header_state_5;
        }
        # 如果匹配暂停，则返回当前状态
        case kMatchPause: {
          return s_n_llhttp__internal__n_header_value_connection_2;
        }
        # 如果匹配不成功，则转到处理 Connection 头部值的 token 部分
        case kMatchMismatch: {
          goto s_n_llhttp__internal__n_header_value_connection_token;
        }
      }
      # 不可达的代码，中断程序执行
      /* UNREACHABLE */;
      abort();
    }
    # 定义状态 s_n_llhttp__internal__n_header_value_connection_3，处理 Connection 头部值的第三部分
    case s_n_llhttp__internal__n_header_value_connection_3:
    s_n_llhttp__internal__n_header_value_connection_3: {
      # 定义匹配结果变量
      llparse_match_t match_seq;

      # 如果指针已经到达结束位置，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_header_value_connection_3;
      }
      # 调用函数，匹配指定长度的字符串并转换为小写
      match_seq = llparse__match_sequence_to_lower(state, p, endp, llparse_blob5, 6);
      # 更新指针位置
      p = match_seq.current;
      # 根据匹配结果进行不同的处理
      switch (match_seq.status) {
        # 如果匹配完成，则继续处理下一个字符
        case kMatchComplete: {
          p++;
          goto s_n_llhttp__internal__n_invoke_update_header_state_6;
        }
        # 如果匹配暂停，则返回当前状态
        case kMatchPause: {
          return s_n_llhttp__internal__n_header_value_connection_3;
        }
        # 如果匹配不成功，则转到处理 Connection 头部值的 token 部分
        case kMatchMismatch: {
          goto s_n_llhttp__internal__n_header_value_connection_token;
        }
      }
      # 不可达的代码，中断程序执行
      /* UNREACHABLE */;
      abort();
    }
    # 定义状态 s_n_llhttp__internal__n_header_value_connection，处理 Connection 头部值的第一部分
    # 如果指针已经指向末尾，则返回当前状态
    s_n_llhttp__internal__n_header_value_connection: {
      if (p == endp) {
        return s_n_llhttp__internal__n_header_value_connection;
      }
      # 判断当前字符是否为大写字母，如果是则转换为小写字母
      switch (((*p) >= 'A' && (*p) <= 'Z' ? (*p | 0x20) : (*p))) {
        # 如果是制表符，则指针向后移动并跳转到状态 s_n_llhttp__internal__n_header_value_connection
        case 9: {
          p++;
          goto s_n_llhttp__internal__n_header_value_connection;
        }
        # 如果是空格，则指针向后移动并跳转到状态 s_n_llhttp__internal__n_header_value_connection
        case ' ': {
          p++;
          goto s_n_llhttp__internal__n_header_value_connection;
        }
        # 如果是字符 'c'，则指针向后移动并跳转到状态 s_n_llhttp__internal__n_header_value_connection_1
        case 'c': {
          p++;
          goto s_n_llhttp__internal__n_header_value_connection_1;
        }
        # 如果是字符 'k'，则指针向后移动并跳转到状态 s_n_llhttp__internal__n_header_value_connection_2
        case 'k': {
          p++;
          goto s_n_llhttp__internal__n_header_value_connection_2;
        }
        # 如果是字符 'u'，则指针向后移动并跳转到状态 s_n_llhttp__internal__n_header_value_connection_3
        case 'u': {
          p++;
          goto s_n_llhttp__internal__n_header_value_connection_3;
        }
        # 如果不是以上情况，则跳转到状态 s_n_llhttp__internal__n_header_value_connection_token
        default: {
          goto s_n_llhttp__internal__n_header_value_connection_token;
        }
      }
      /* UNREACHABLE */;
      # 终止程序执行
      abort();
    }
    # 如果当前状态为 s_n_llhttp__internal__n_error_17
    case s_n_llhttp__internal__n_error_17:
    s_n_llhttp__internal__n_error_17: {
      # 设置状态的错误码和原因
      state->error = 0xb;
      state->reason = "Content-Length overflow";
      state->error_pos = (const char*) p;
      state->_current = (void*) (intptr_t) s_error;
      return s_error;
      /* UNREACHABLE */;
      # 终止程序执行
      abort();
    }
    # 如果当前状态为 s_n_llhttp__internal__n_error_18
    case s_n_llhttp__internal__n_error_18:
    s_n_llhttp__internal__n_error_18: {
      # 设置状态的错误码和原因
      state->error = 0xb;
      state->reason = "Invalid character in Content-Length";
      state->error_pos = (const char*) p;
      state->_current = (void*) (intptr_t) s_error;
      return s_error;
      /* UNREACHABLE */;
      # 终止程序执行
      abort();
    }
    # 如果当前状态为 s_n_llhttp__internal__n_header_value_content_length_ws
    # 如果指针已经指向末尾，则返回当前状态
    if (p == endp) {
        return s_n_llhttp__internal__n_header_value_content_length_ws;
    }
    # 根据当前字符进行不同的处理
    switch (*p) {
        # 如果是换行符，则跳转到处理换行的状态
        case 10: {
            goto s_n_llhttp__internal__n_invoke_or_flags_15;
        }
        # 如果是回车符，则跳转到处理换行的状态
        case 13: {
            goto s_n_llhttp__internal__n_invoke_or_flags_15;
        }
        # 如果是空格，则移动指针并继续处理空格
        case ' ': {
            p++;
            goto s_n_llhttp__internal__n_header_value_content_length_ws;
        }
        # 如果是其他字符，则跳转到处理头部值结束的状态
        default: {
            goto s_n_llhttp__internal__n_span_end_llhttp__on_header_value_5;
        }
    }
    # 不可达的代码，中止程序
    /* UNREACHABLE */;
    abort();
    # 处理头部值为 content-length 的情况
    case s_n_llhttp__internal__n_header_value_content_length:
    # 如果指针已经到达末尾，则返回当前状态
    s_n_llhttp__internal__n_header_value_content_length: {
      if (p == endp) {
        return s_n_llhttp__internal__n_header_value_content_length;
      }
      # 根据当前字符进行不同的处理
      switch (*p) {
        # 如果是数字0，则移动指针，设置匹配值为0，跳转到下一个状态
        case '0': {
          p++;
          match = 0;
          goto s_n_llhttp__internal__n_invoke_mul_add_content_length_1;
        }
        # 如果是数字1，则移动指针，设置匹配值为1，跳转到下一个状态
        case '1': {
          p++;
          match = 1;
          goto s_n_llhttp__internal__n_invoke_mul_add_content_length_1;
        }
        # 如果是数字2，则移动指针，设置匹配值为2，跳转到下一个状态
        case '2': {
          p++;
          match = 2;
          goto s_n_llhttp__internal__n_invoke_mul_add_content_length_1;
        }
        # 如果是数字3，则移动指针，设置匹配值为3，跳转到下一个状态
        case '3': {
          p++;
          match = 3;
          goto s_n_llhttp__internal__n_invoke_mul_add_content_length_1;
        }
        # 如果是数字4，则移动指针，设置匹配值为4，跳转到下一个状态
        case '4': {
          p++;
          match = 4;
          goto s_n_llhttp__internal__n_invoke_mul_add_content_length_1;
        }
        # 如果是数字5，则移动指针，设置匹配值为5，跳转到下一个状态
        case '5': {
          p++;
          match = 5;
          goto s_n_llhttp__internal__n_invoke_mul_add_content_length_1;
        }
        # 如果是数字6，则移动指针，设置匹配值为6，跳转到下一个状态
        case '6': {
          p++;
          match = 6;
          goto s_n_llhttp__internal__n_invoke_mul_add_content_length_1;
        }
        # 如果是数字7，则移动指针，设置匹配值为7，跳转到下一个状态
        case '7': {
          p++;
          match = 7;
          goto s_n_llhttp__internal__n_invoke_mul_add_content_length_1;
        }
        # 如果是数字8，则移动指针，设置匹配值为8，跳转到下一个状态
        case '8': {
          p++;
          match = 8;
          goto s_n_llhttp__internal__n_invoke_mul_add_content_length_1;
        }
        # 如果是数字9，则移动指针，设置匹配值为9，跳转到下一个状态
        case '9': {
          p++;
          match = 9;
          goto s_n_llhttp__internal__n_invoke_mul_add_content_length_1;
        }
        # 如果不是数字，则跳转到下一个状态
        default: {
          goto s_n_llhttp__internal__n_header_value_content_length_ws;
        }
      }
      # 不可达的代码，中止程序
      /* UNREACHABLE */;
      abort();
    }
    # 处理下一个状态
    case s_n_llhttp__internal__n_header_value_te_chunked_last:
    # 如果指针已经指向末尾，则返回当前状态
    s_n_llhttp__internal__n_header_value_te_chunked_last: {
      if (p == endp) {
        return s_n_llhttp__internal__n_header_value_te_chunked_last;
      }
      # 根据当前字符进行不同的处理
      switch (*p) {
        # 如果是换行符，则跳转到更新头部状态
        case 10: {
          goto s_n_llhttp__internal__n_invoke_update_header_state_7;
        }
        # 如果是回车符，则跳转到更新头部状态
        case 13: {
          goto s_n_llhttp__internal__n_invoke_update_header_state_7;
        }
        # 如果是空格，则指针向后移动，继续处理当前状态
        case ' ': {
          p++;
          goto s_n_llhttp__internal__n_header_value_te_chunked_last;
        }
        # 其他情况，则跳转到处理分块编码的状态
        default: {
          goto s_n_llhttp__internal__n_header_value_te_chunked;
        }
      }
      /* UNREACHABLE */;
      # 终止程序
      abort();
    }
    # 处理分块编码中的空白符
    case s_n_llhttp__internal__n_header_value_te_token_ows:
    s_n_llhttp__internal__n_header_value_te_token_ows: {
      # 如果指针已经指向末尾，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_header_value_te_token_ows;
      }
      # 根据当前字符进行不同的处理
      switch (*p) {
        # 如果是制表符，则指针向后移动
        case 9: {
          p++;
          goto s_n_llhttp__internal__n_header_value_te_token_ows;
        }
        # 如果是空格，则指针向后移动
        case ' ': {
          p++;
          goto s_n_llhttp__internal__n_header_value_te_token_ows;
        }
        # 其他情况，则跳转到处理分块编码的状态
        default: {
          goto s_n_llhttp__internal__n_header_value_te_chunked;
        }
      }
      /* UNREACHABLE */;
      # 终止程序
      abort();
    }
    # 处理头部值
    case s_n_llhttp__internal__n_header_value:
    }
    # 处理分块编码中的标记
    case s_n_llhttp__internal__n_header_value_te_token:
    # 定义名为 s_n_llhttp__internal__n_header_value_te_token 的静态变量
    s_n_llhttp__internal__n_header_value_te_token: {
      # 定义名为 lookup_table 的静态数组，存储了一系列数字
      static uint8_t lookup_table[] = {
        # 数组内容省略
      };
      # 如果指针 p 等于指针 endp，则返回 s_n_llhttp__internal__n_header_value_te_token
      if (p == endp) {
        return s_n_llhttp__internal__n_header_value_te_token;
      }
      # 根据 lookup_table 中 *p 对应的值进行不同的操作
      switch (lookup_table[(uint8_t) *p]) {
        # 如果值为 1，则指针 p 向后移动一位，然后跳转到标签 s_n_llhttp__internal__n_header_value_te_token
        case 1: {
          p++;
          goto s_n_llhttp__internal__n_header_value_te_token;
        }
        # 如果值为 2，则指针 p 向后移动一位，然后跳转到标签 s_n_llhttp__internal__n_header_value_te_token_ows
        case 2: {
          p++;
          goto s_n_llhttp__internal__n_header_value_te_token_ows;
        }
        # 如果值不是 1 或 2，则跳转到标签 s_n_llhttp__internal__n_invoke_update_header_state_8
        default: {
          goto s_n_llhttp__internal__n_invoke_update_header_state_8;
        }
      }
      # 不可达的代码，中止程序
      /* UNREACHABLE */;
      abort();
    }
    # 定义名为 s_n_llhttp__internal__n_header_value_te_chunked 的情况
    case s_n_llhttp__internal__n_header_value_te_chunked:
    # 定义状态 s_n_llhttp__internal__n_header_value_te_chunked，用于处理解析 HTTP 头部值中的 chunked 编码
    s_n_llhttp__internal__n_header_value_te_chunked: {
      # 定义匹配结果变量
      llparse_match_t match_seq;

      # 如果指针已经到达结束位置，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_header_value_te_chunked;
      }
      # 尝试匹配指针位置开始的 7 个字符是否与 llparse_blob6 相同
      match_seq = llparse__match_sequence_to_lower_unsafe(state, p, endp, llparse_blob6, 7);
      p = match_seq.current;
      # 根据匹配结果进行不同的处理
      switch (match_seq.status) {
        # 如果匹配完成，则指针后移一位，跳转到 s_n_llhttp__internal__n_header_value_te_chunked_last 状态
        case kMatchComplete: {
          p++;
          goto s_n_llhttp__internal__n_header_value_te_chunked_last;
        }
        # 如果匹配暂停，则返回当前状态
        case kMatchPause: {
          return s_n_llhttp__internal__n_header_value_te_chunked;
        }
        # 如果匹配失败，则跳转到 s_n_llhttp__internal__n_header_value_te_token 状态
        case kMatchMismatch: {
          goto s_n_llhttp__internal__n_header_value_te_token;
        }
      }
      # 不可达的代码，中止程序
      /* UNREACHABLE */;
      abort();
    }
    # 定义状态 s_n_llhttp__internal__n_span_start_llhttp__on_header_value_1
    s_n_llhttp__internal__n_span_start_llhttp__on_header_value_1: {
      # 如果指针已经到达结束位置，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_span_start_llhttp__on_header_value_1;
      }
      # 设置状态的 span_pos0 和 span_cb0 属性
      state->_span_pos0 = (void*) p;
      state->_span_cb0 = llhttp__on_header_value;
      # 跳转到 s_n_llhttp__internal__n_invoke_load_header_state_2 状态
      goto s_n_llhttp__internal__n_invoke_load_header_state_2;
      # 不可达的代码，中止程序
      /* UNREACHABLE */;
      abort();
    }
    # 定义状态 s_n_llhttp__internal__n_header_value_discard_ws
    s_n_llhttp__internal__n_header_value_discard_ws: {
      # 如果指针已经到达结束位置，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_header_value_discard_ws;
      }
      # 根据当前字符进行不同的处理
      switch (*p) {
        # 如果是制表符，则指针后移一位，跳转到 s_n_llhttp__internal__n_header_value_discard_ws
        case 9: {
          p++;
          goto s_n_llhttp__internal__n_header_value_discard_ws;
        }
        # 如果是换行符，则指针后移一位，跳转到 s_n_llhttp__internal__n_header_value_discard_lws
        case 10: {
          p++;
          goto s_n_llhttp__internal__n_header_value_discard_lws;
        }
        # 如果是回车符，则指针后移一位，跳转到 s_n_llhttp__internal__n_header_value_discard_ws_almost_done
        case 13: {
          p++;
          goto s_n_llhttp__internal__n_header_value_discard_ws_almost_done;
        }
        # 如果是空格，则指针后移一位，跳转到 s_n_llhttp__internal__n_header_value_discard_ws
        case ' ': {
          p++;
          goto s_n_llhttp__internal__n_header_value_discard_ws;
        }
        # 其他情况，跳转到 s_n_llhttp__internal__n_span_start_llhttp__on_header_value_1
        default: {
          goto s_n_llhttp__internal__n_span_start_llhttp__on_header_value_1;
        }
      }
      # 不可达的代码，中止程序
      /* UNREACHABLE */;
      abort();
    }
    // 处理 header field 完成的状态
    case s_n_llhttp__internal__n_invoke_llhttp__on_header_field_complete:
    s_n_llhttp__internal__n_invoke_llhttp__on_header_field_complete: {
      // 调用处理 header field 完成的函数，并根据返回结果进行相应的处理
      switch (llhttp__on_header_field_complete(state, p, endp)) {
        default:
          // 转到处理 header value 中丢弃空白字符的状态
          goto s_n_llhttp__internal__n_header_value_discard_ws;
      }
      /* UNREACHABLE */;
      // 终止程序执行
      abort();
    }
    // 处理一般的 header field 的状态
    case s_n_llhttp__internal__n_header_field_general_otherwise:
    s_n_llhttp__internal__n_header_field_general_otherwise: {
      // 如果指针指向结束位置，则保持当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_header_field_general_otherwise;
      }
      // 根据当前字符进行相应的处理
      switch (*p) {
        case ':': {
          // 转到处理 header field 结束的状态
          goto s_n_llhttp__internal__n_span_end_llhttp__on_header_field_1;
        }
        default: {
          // 转到处理错误的状态
          goto s_n_llhttp__internal__n_error_19;
        }
      }
      /* UNREACHABLE */;
      // 终止程序执行
      abort();
    }
    // 处理一般的 header field 的状态
    case s_n_llhttp__internal__n_header_field_general:
    }
    // 处理 header field 冒号的状态
    case s_n_llhttp__internal__n_header_field_colon:
    s_n_llhttp__internal__n_header_field_colon: {
      // 如果指针指向结束位置，则保持当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_header_field_colon;
      }
      // 根据当前字符进行相应的处理
      switch (*p) {
        case ' ': {
          // 移动指针并保持当前状态
          p++;
          goto s_n_llhttp__internal__n_header_field_colon;
        }
        case ':': {
          // 转到处理 header field 结束的状态
          goto s_n_llhttp__internal__n_span_end_llhttp__on_header_field;
        }
        default: {
          // 转到更新 header state 的状态
          goto s_n_llhttp__internal__n_invoke_update_header_state_9;
        }
      }
      /* UNREACHABLE */;
      // 终止程序执行
      abort();
    }
    // 处理 header field 的状态
    case s_n_llhttp__internal__n_header_field_3:
    # 定义状态 s_n_llhttp__internal__n_header_field_3，用于解析 HTTP 请求头字段
    s_n_llhttp__internal__n_header_field_3: {
      # 定义变量 match_seq，用于存储匹配结果
      llparse_match_t match_seq;

      # 如果指针 p 已经指向输入结束位置，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_header_field_3;
      }
      # 调用 llparse__match_sequence_to_lower 函数，将输入数据转换为小写并进行匹配
      match_seq = llparse__match_sequence_to_lower(state, p, endp, llparse_blob2, 6);
      # 更新指针 p 的位置
      p = match_seq.current;
      # 根据匹配结果的状态进行不同的处理
      switch (match_seq.status) {
        # 如果匹配完成，则向前移动一个字符，设置匹配标志为 1，并跳转到存储头字段状态
        case kMatchComplete: {
          p++;
          match = 1;
          goto s_n_llhttp__internal__n_invoke_store_header_state;
        }
        # 如果匹配暂停，则返回当前状态
        case kMatchPause: {
          return s_n_llhttp__internal__n_header_field_3;
        }
        # 如果匹配失败，则跳转到更新头字段状态
        case kMatchMismatch: {
          goto s_n_llhttp__internal__n_invoke_update_header_state_10;
        }
      }
      /* UNREACHABLE */;
      # 如果代码执行到这里，表示出现了不可达的情况，终止程序
      abort();
    }
    # 定义状态 s_n_llhttp__internal__n_header_field_4，用于解析 HTTP 请求头字段
    case s_n_llhttp__internal__n_header_field_4:
    s_n_llhttp__internal__n_header_field_4: {
      # 定义变量 match_seq，用于存储匹配结果
      llparse_match_t match_seq;

      # 如果指针 p 已经指向输入结束位置，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_header_field_4;
      }
      # 调用 llparse__match_sequence_to_lower 函数，将输入数据转换为小写并进行匹配
      match_seq = llparse__match_sequence_to_lower(state, p, endp, llparse_blob10, 10);
      # 更新指针 p 的位置
      p = match_seq.current;
      # 根据匹配结果的状态进行不同的处理
      switch (match_seq.status) {
        # 如果匹配完成，则向前移动一个字符，设置匹配标志为 2，并跳转到存储头字段状态
        case kMatchComplete: {
          p++;
          match = 2;
          goto s_n_llhttp__internal__n_invoke_store_header_state;
        }
        # 如果匹配暂停，则返回当前状态
        case kMatchPause: {
          return s_n_llhttp__internal__n_header_field_4;
        }
        # 如果匹配失败，则跳转到更新头字段状态
        case kMatchMismatch: {
          goto s_n_llhttp__internal__n_invoke_update_header_state_10;
        }
      }
      /* UNREACHABLE */;
      # 如果代码执行到这里，表示出现了不可达的情况，终止程序
      abort();
    }
    # 定义状态 s_n_llhttp__internal__n_header_field_2:
    # 定义状态机的状态 s_n_llhttp__internal__n_header_field_2
    s_n_llhttp__internal__n_header_field_2: {
      # 如果指针 p 指向结束位置，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_header_field_2;
      }
      # 判断当前字符是否为大写字母，如果是则转换为小写字母，然后进行 switch 分支判断
      switch (((*p) >= 'A' && (*p) <= 'Z' ? (*p | 0x20) : (*p))) {
        # 如果当前字符为 'n'，则指针向后移动一位，跳转到状态 s_n_llhttp__internal__n_header_field_3
        case 'n': {
          p++;
          goto s_n_llhttp__internal__n_header_field_3;
        }
        # 如果当前字符为 't'，则指针向后移动一位，跳转到状态 s_n_llhttp__internal__n_header_field_4
        case 't': {
          p++;
          goto s_n_llhttp__internal__n_header_field_4;
        }
        # 如果当前字符不是 'n' 或 't'，则跳转到状态 s_n_llhttp__internal__n_invoke_update_header_state_10
        default: {
          goto s_n_llhttp__internal__n_invoke_update_header_state_10;
        }
      }
      # 不可达的代码，中止程序
      /* UNREACHABLE */;
      abort();
    }
    # 定义状态机的状态 s_n_llhttp__internal__n_header_field_1
    case s_n_llhttp__internal__n_header_field_1:
    s_n_llhttp__internal__n_header_field_1: {
      # 定义状态机匹配结果的结构体
      llparse_match_t match_seq;

      # 如果指针 p 指向结束位置，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_header_field_1;
      }
      # 调用函数进行匹配，得到匹配结果
      match_seq = llparse__match_sequence_to_lower(state, p, endp, llparse_blob1, 2);
      # 更新指针位置
      p = match_seq.current;
      # 根据匹配结果进行 switch 分支判断
      switch (match_seq.status) {
        # 如果匹配完成，则指针向后移动一位，跳转到状态 s_n_llhttp__internal__n_header_field_2
        case kMatchComplete: {
          p++;
          goto s_n_llhttp__internal__n_header_field_2;
        }
        # 如果匹配暂停，则返回当前状态
        case kMatchPause: {
          return s_n_llhttp__internal__n_header_field_1;
        }
        # 如果匹配不成功，则跳转到状态 s_n_llhttp__internal__n_invoke_update_header_state_10
        case kMatchMismatch: {
          goto s_n_llhttp__internal__n_invoke_update_header_state_10;
        }
      }
      # 不可达的代码，中止程序
      /* UNREACHABLE */;
      abort();
    }
    # 定义状态机的状态 s_n_llhttp__internal__n_header_field_5:
    # 定义状态 s_n_llhttp__internal__n_header_field_5，用于解析 HTTP 请求头字段
    s_n_llhttp__internal__n_header_field_5: {
      # 定义变量 match_seq，用于存储匹配结果
      llparse_match_t match_seq;

      # 如果指针 p 已经指向输入结束位置，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_header_field_5;
      }
      # 调用 llparse__match_sequence_to_lower 函数，匹配输入数据是否符合指定的序列
      match_seq = llparse__match_sequence_to_lower(state, p, endp, llparse_blob11, 15);
      # 更新指针 p 的位置
      p = match_seq.current;
      # 根据匹配结果的状态进行不同的处理
      switch (match_seq.status) {
        # 如果匹配完成
        case kMatchComplete: {
          # 指针 p 向前移动一位
          p++;
          # 设置匹配标志为 1
          match = 1;
          # 跳转到状态 s_n_llhttp__internal__n_invoke_store_header_state
          goto s_n_llhttp__internal__n_invoke_store_header_state;
        }
        # 如果匹配暂停
        case kMatchPause: {
          # 返回当前状态
          return s_n_llhttp__internal__n_header_field_5;
        }
        # 如果匹配不成功
        case kMatchMismatch: {
          # 跳转到状态 s_n_llhttp__internal__n_invoke_update_header_state_10
          goto s_n_llhttp__internal__n_invoke_update_header_state_10;
        }
      }
      /* UNREACHABLE */;
      # 终止程序执行
      abort();
    }
    # 定义状态 s_n_llhttp__internal__n_header_field_6
    case s_n_llhttp__internal__n_header_field_6:
    s_n_llhttp__internal__n_header_field_6: {
      # 定义变量 match_seq，用于存储匹配结果
      llparse_match_t match_seq;

      # 如果指针 p 已经指向输入结束位置，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_header_field_6;
      }
      # 调用 llparse__match_sequence_to_lower 函数，匹配输入数据是否符合指定的序列
      match_seq = llparse__match_sequence_to_lower(state, p, endp, llparse_blob12, 16);
      # 更新指针 p 的位置
      p = match_seq.current;
      # 根据匹配结果的状态进行不同的处理
      switch (match_seq.status) {
        # 如果匹配完成
        case kMatchComplete: {
          # 指针 p 向前移动一位
          p++;
          # 设置匹配标志为 3
          match = 3;
          # 跳转到状态 s_n_llhttp__internal__n_invoke_store_header_state
          goto s_n_llhttp__internal__n_invoke_store_header_state;
        }
        # 如果匹配暂停
        case kMatchPause: {
          # 返回当前状态
          return s_n_llhttp__internal__n_header_field_6;
        }
        # 如果匹配不成功
        case kMatchMismatch: {
          # 跳转到状态 s_n_llhttp__internal__n_invoke_update_header_state_10
          goto s_n_llhttp__internal__n_invoke_update_header_state_10;
        }
      }
      /* UNREACHABLE */;
      # 终止程序执行
      abort();
    }
    # 定义状态 s_n_llhttp__internal__n_header_field_7:
    # 定义一个名为 s_n_llhttp__internal__n_header_field_7 的状态
    s_n_llhttp__internal__n_header_field_7: {
      # 定义一个名为 match_seq 的变量，用于存储匹配结果
      llparse_match_t match_seq;

      # 如果指针 p 等于结束指针 endp，则返回状态 s_n_llhttp__internal__n_header_field_7
      if (p == endp) {
        return s_n_llhttp__internal__n_header_field_7;
      }
      # 调用 llparse__match_sequence_to_lower 函数，将结果存储在 match_seq 中
      match_seq = llparse__match_sequence_to_lower(state, p, endp, llparse_blob13, 6);
      # 更新指针 p 的位置
      p = match_seq.current;
      # 根据匹配结果的状态进行不同的处理
      switch (match_seq.status) {
        # 如果匹配完成
        case kMatchComplete: {
          # 更新指针 p 的位置
          p++;
          # 设置 match 变量的值为 4
          match = 4;
          # 跳转到状态 s_n_llhttp__internal__n_invoke_store_header_state
          goto s_n_llhttp__internal__n_invoke_store_header_state;
        }
        # 如果匹配暂停
        case kMatchPause: {
          # 返回状态 s_n_llhttp__internal__n_header_field_7
          return s_n_llhttp__internal__n_header_field_7;
        }
        # 如果匹配不成功
        case kMatchMismatch: {
          # 跳转到状态 s_n_llhttp__internal__n_invoke_update_header_state_10
          goto s_n_llhttp__internal__n_invoke_update_header_state_10;
        }
      }
      # 不可达的代码，中断程序执行
      /* UNREACHABLE */;
      abort();
    }
    # 定义一个名为 s_n_llhttp__internal__n_header_field 的状态
    case s_n_llhttp__internal__n_header_field:
    s_n_llhttp__internal__n_header_field: {
      # 如果指针 p 等于结束指针 endp，则返回状态 s_n_llhttp__internal__n_header_field
      if (p == endp) {
        return s_n_llhttp__internal__n_header_field;
      }
      # 根据指针 p 指向的字符进行不同的处理
      switch (((*p) >= 'A' && (*p) <= 'Z' ? (*p | 0x20) : (*p))) {
        # 如果字符为 'c'
        case 'c': {
          # 更新指针 p 的位置
          p++;
          # 跳转到状态 s_n_llhttp__internal__n_header_field_1
          goto s_n_llhttp__internal__n_header_field_1;
        }
        # 如果字符为 'p'
        case 'p': {
          # 更新指针 p 的位置
          p++;
          # 跳转到状态 s_n_llhttp__internal__n_header_field_5
          goto s_n_llhttp__internal__n_header_field_5;
        }
        # 如果字符为 't'
        case 't': {
          # 更新指针 p 的位置
          p++;
          # 跳转到状态 s_n_llhttp__internal__n_header_field_6
          goto s_n_llhttp__internal__n_header_field_6;
        }
        # 如果字符为 'u'
        case 'u': {
          # 更新指针 p 的位置
          p++;
          # 跳转到状态 s_n_llhttp__internal__n_header_field_7
          goto s_n_llhttp__internal__n_header_field_7;
        }
        # 默认情况
        default: {
          # 跳转到状态 s_n_llhttp__internal__n_invoke_update_header_state_10
          goto s_n_llhttp__internal__n_invoke_update_header_state_10;
        }
      }
      # 不可达的代码，中断程序执行
      /* UNREACHABLE */;
      abort();
    }
    # 定义一个名为 s_n_llhttp__internal__n_span_start_llhttp__on_header_field 的状态
    case s_n_llhttp__internal__n_span_start_llhttp__on_header_field:
    s_n_llhttp__internal__n_span_start_llhttp__on_header_field: {
      # 如果指针 p 等于结束指针 endp，则返回状态 s_n_llhttp__internal__n_span_start_llhttp__on_header_field
      if (p == endp) {
        return s_n_llhttp__internal__n_span_start_llhttp__on_header_field;
      }
      # 设置 state 对象的 _span_pos0 属性为指针 p 的位置
      state->_span_pos0 = (void*) p;
      # 设置 state 对象的 _span_cb0 属性为 llhttp__on_header_field 函数
      state->_span_cb0 = llhttp__on_header_field;
      # 跳转到状态 s_n_llhttp__internal__n_header_field
      goto s_n_llhttp__internal__n_header_field;
      # 不可达的代码，中断程序执行
      /* UNREACHABLE */;
      abort();
    }
    # 定义一个名为 s_n_llhttp__internal__n_header_field_start 的状态
    case s_n_llhttp__internal__n_header_field_start:
    # 如果指针已经到达末尾，则返回当前状态
    s_n_llhttp__internal__n_header_field_start: {
      if (p == endp) {
        return s_n_llhttp__internal__n_header_field_start;
      }
      # 根据当前字符进行不同的处理
      switch (*p) {
        # 如果是换行符，则跳转到几乎完成状态
        case 10: {
          goto s_n_llhttp__internal__n_headers_almost_done;
        }
        # 如果是回车符，则向前移动一个字符，然后跳转到几乎完成状态
        case 13: {
          p++;
          goto s_n_llhttp__internal__n_headers_almost_done;
        }
        # 如果是其他字符，则跳转到处理 header field 的状态
        default: {
          goto s_n_llhttp__internal__n_span_start_llhttp__on_header_field;
        }
      }
      # 不可达的代码，中止程序
      /* UNREACHABLE */;
      abort();
    }
    # 如果当前状态是跳转到 HTTP/0.9 的状态
    case s_n_llhttp__internal__n_url_skip_to_http09:
    s_n_llhttp__internal__n_url_skip_to_http09: {
      # 如果指针已经到达末尾，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_url_skip_to_http09;
      }
      # 向前移动一个字符，然后跳转到更新 HTTP 主版本号的状态
      p++;
      /* 不可达的代码，中止程序 */
      /* UNREACHABLE */;
      abort();
    }
    # 如果当前状态是跳过 LF 到 HTTP/0.9 的状态
    case s_n_llhttp__internal__n_url_skip_lf_to_http09:
    s_n_llhttp__internal__n_url_skip_lf_to_http09: {
      llparse_match_t match_seq;

      # 如果指针已经到达末尾，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_url_skip_lf_to_http09;
      }
      # 匹配指定的序列，根据匹配结果进行不同的处理
      match_seq = llparse__match_sequence_id(state, p, endp, llparse_blob14, 2);
      p = match_seq.current;
      switch (match_seq.status) {
        # 如果匹配完成，则向前移动一个字符，然后跳转到更新 HTTP 主版本号的状态
        case kMatchComplete: {
          p++;
          goto s_n_llhttp__internal__n_invoke_update_http_major;
        }
        # 如果匹配暂停，则返回当前状态
        case kMatchPause: {
          return s_n_llhttp__internal__n_url_skip_lf_to_http09;
        }
        # 如果匹配不成功，则跳转到错误状态
        case kMatchMismatch: {
          goto s_n_llhttp__internal__n_error_20;
        }
      }
      # 不可达的代码，中止程序
      /* UNREACHABLE */;
      abort();
    }
    # 如果当前状态是请求升级的状态
    case s_n_llhttp__internal__n_req_pri_upgrade:
    # 定义状态 s_n_llhttp__internal__n_req_pri_upgrade，用于处理请求升级的情况
    s_n_llhttp__internal__n_req_pri_upgrade: {
      # 定义匹配序列的变量
      llparse_match_t match_seq;

      # 如果指针已经指向输入的结尾，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_req_pri_upgrade;
      }
      # 调用 llparse__match_sequence_id 函数，匹配指定长度的数据
      match_seq = llparse__match_sequence_id(state, p, endp, llparse_blob16, 10);
      # 更新指针位置
      p = match_seq.current;
      # 根据匹配的状态进行不同的处理
      switch (match_seq.status) {
        # 如果匹配完成，则执行下一步操作
        case kMatchComplete: {
          p++;
          # 跳转到错误处理状态 s_n_llhttp__internal__n_error_23
          goto s_n_llhttp__internal__n_error_23;
        }
        # 如果匹配暂停，则返回当前状态
        case kMatchPause: {
          return s_n_llhttp__internal__n_req_pri_upgrade;
        }
        # 如果匹配不成功，则跳转到错误处理状态 s_n_llhttp__internal__n_error_24
        case kMatchMismatch: {
          goto s_n_llhttp__internal__n_error_24;
        }
      }
      # 不可达的代码，中止程序
      /* UNREACHABLE */;
      abort();
    }
    # 定义状态 s_n_llhttp__internal__n_req_http_complete_1
    case s_n_llhttp__internal__n_req_http_complete_1:
    s_n_llhttp__internal__n_req_http_complete_1: {
      # 如果指针已经指向输入的结尾，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_req_http_complete_1;
      }
      # 根据当前字符进行不同的处理
      switch (*p) {
        # 如果是换行符，则更新指针位置并跳转到头部字段开始状态
        case 10: {
          p++;
          goto s_n_llhttp__internal__n_header_field_start;
        }
        # 如果不是换行符，则跳转到错误处理状态 s_n_llhttp__internal__n_error_22
        default: {
          goto s_n_llhttp__internal__n_error_22;
        }
      }
      # 不可达的代码，中止程序
      /* UNREACHABLE */;
      abort();
    }
    # 定义状态 s_n_llhttp__internal__n_req_http_complete
    case s_n_llhttp__internal__n_req_http_complete:
    s_n_llhttp__internal__n_req_http_complete: {
      # 如果指针已经指向输入的结尾，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_req_http_complete;
      }
      # 根据当前字符进行不同的处理
      switch (*p) {
        # 如果是换行符，则更新指针位置并跳转到头部字段开始状态
        case 10: {
          p++;
          goto s_n_llhttp__internal__n_header_field_start;
        }
        # 如果是回车符，则更新指针位置并跳转到请求 HTTP 完成状态 1
        case 13: {
          p++;
          goto s_n_llhttp__internal__n_req_http_complete_1;
        }
        # 如果不是换行符或回车符，则跳转到错误处理状态 s_n_llhttp__internal__n_error_22
        default: {
          goto s_n_llhttp__internal__n_error_22;
        }
      }
      # 不可达的代码，中止程序
      /* UNREACHABLE */;
      abort();
    }
    # 定义状态 s_n_llhttp__internal__n_req_http_minor:
    # 检查是否已经到达输入的末尾
    s_n_llhttp__internal__n_req_http_minor: {
      if (p == endp) {
        return s_n_llhttp__internal__n_req_http_minor;
      }
      # 根据当前字符进行不同的处理
      switch (*p) {
        # 如果是 '0'，则将 p 向后移动一位，match 设置为 0，跳转到 s_n_llhttp__internal__n_invoke_store_http_minor 标签处
        case '0': {
          p++;
          match = 0;
          goto s_n_llhttp__internal__n_invoke_store_http_minor;
        }
        # 如果是 '1'，则将 p 向后移动一位，match 设置为 1，跳转到 s_n_llhttp__internal__n_invoke_store_http_minor 标签处
        case '1': {
          p++;
          match = 1;
          goto s_n_llhttp__internal__n_invoke_store_http_minor;
        }
        # 如果是 '2'，则将 p 向后移动一位，match 设置为 2，跳转到 s_n_llhttp__internal__n_invoke_store_http_minor 标签处
        case '2': {
          p++;
          match = 2;
          goto s_n_llhttp__internal__n_invoke_store_http_minor;
        }
        # 如果是 '3'，则将 p 向后移动一位，match 设置为 3，跳转到 s_n_llhttp__internal__n_invoke_store_http_minor 标签处
        case '3': {
          p++;
          match = 3;
          goto s_n_llhttp__internal__n_invoke_store_http_minor;
        }
        # 如果是 '4'，则将 p 向后移动一位，match 设置为 4，跳转到 s_n_llhttp__internal__n_invoke_store_http_minor 标签处
        case '4': {
          p++;
          match = 4;
          goto s_n_llhttp__internal__n_invoke_store_http_minor;
        }
        # 如果是 '5'，则将 p 向后移动一位，match 设置为 5，跳转到 s_n_llhttp__internal__n_invoke_store_http_minor 标签处
        case '5': {
          p++;
          match = 5;
          goto s_n_llhttp__internal__n_invoke_store_http_minor;
        }
        # 如果是 '6'，则将 p 向后移动一位，match 设置为 6，跳转到 s_n_llhttp__internal__n_invoke_store_http_minor 标签处
        case '6': {
          p++;
          match = 6;
          goto s_n_llhttp__internal__n_invoke_store_http_minor;
        }
        # 如果是 '7'，则将 p 向后移动一位，match 设置为 7，跳转到 s_n_llhttp__internal__n_invoke_store_http_minor 标签处
        case '7': {
          p++;
          match = 7;
          goto s_n_llhttp__internal__n_invoke_store_http_minor;
        }
        # 如果是 '8'，则将 p 向后移动一位，match 设置为 8，跳转到 s_n_llhttp__internal__n_invoke_store_http_minor 标签处
        case '8': {
          p++;
          match = 8;
          goto s_n_llhttp__internal__n_invoke_store_http_minor;
        }
        # 如果是 '9'，则将 p 向后移动一位，match 设置为 9，跳转到 s_n_llhttp__internal__n_invoke_store_http_minor 标签处
        case '9': {
          p++;
          match = 9;
          goto s_n_llhttp__internal__n_invoke_store_http_minor;
        }
        # 如果是其他字符，则跳转到 s_n_llhttp__internal__n_error_25 标签处
        default: {
          goto s_n_llhttp__internal__n_error_25;
        }
      }
      /* UNREACHABLE */;
      # 终止程序执行
      abort();
    }
    # 处理请求行中 HTTP 版本号的小数点
    case s_n_llhttp__internal__n_req_http_dot:
    # 如果指针已经指向末尾，则返回当前状态
    if (p == endp) {
        return s_n_llhttp__internal__n_req_http_dot;
    }
    # 根据当前字符进行不同的处理
    switch (*p) {
        # 如果是'.'，则指针向后移动一位，进入下一个状态
        case '.': {
            p++;
            goto s_n_llhttp__internal__n_req_http_minor;
        }
        # 如果不是'.'，则进入错误状态
        default: {
            goto s_n_llhttp__internal__n_error_26;
        }
    }
    # 不可达的代码，中止程序
    /* UNREACHABLE */;
    abort();
    # 进入下一个状态
    case s_n_llhttp__internal__n_req_http_major:
    # 检查请求行中 HTTP 版本号的主版本号
    s_n_llhttp__internal__n_req_http_major: {
      # 如果指针已经指向末尾，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_req_http_major;
      }
      # 根据指针指向的字符进行不同的处理
      switch (*p) {
        # 如果是 '0'，则将指针向后移动一位，匹配值设为 0，跳转到存储 HTTP 主版本号的状态
        case '0': {
          p++;
          match = 0;
          goto s_n_llhttp__internal__n_invoke_store_http_major;
        }
        # 如果是 '1'，则将指针向后移动一位，匹配值设为 1，跳转到存储 HTTP 主版本号的状态
        case '1': {
          p++;
          match = 1;
          goto s_n_llhttp__internal__n_invoke_store_http_major;
        }
        # 如果是 '2'，则将指针向后移动一位，匹配值设为 2，跳转到存储 HTTP 主版本号的状态
        case '2': {
          p++;
          match = 2;
          goto s_n_llhttp__internal__n_invoke_store_http_major;
        }
        # 如果是 '3'，则将指针向后移动一位，匹配值设为 3，跳转到存储 HTTP 主版本号的状态
        case '3': {
          p++;
          match = 3;
          goto s_n_llhttp__internal__n_invoke_store_http_major;
        }
        # 如果是 '4'，则将指针向后移动一位，匹配值设为 4，跳转到存储 HTTP 主版本号的状态
        case '4': {
          p++;
          match = 4;
          goto s_n_llhttp__internal__n_invoke_store_http_major;
        }
        # 如果是 '5'，则将指针向后移动一位，匹配值设为 5，跳转到存储 HTTP 主版本号的状态
        case '5': {
          p++;
          match = 5;
          goto s_n_llhttp__internal__n_invoke_store_http_major;
        }
        # 如果是 '6'，则将指针向后移动一位，匹配值设为 6，跳转到存储 HTTP 主版本号的状态
        case '6': {
          p++;
          match = 6;
          goto s_n_llhttp__internal__n_invoke_store_http_major;
        }
        # 如果是 '7'，则将指针向后移动一位，匹配值设为 7，跳转到存储 HTTP 主版本号的状态
        case '7': {
          p++;
          match = 7;
          goto s_n_llhttp__internal__n_invoke_store_http_major;
        }
        # 如果是 '8'，则将指针向后移动一位，匹配值设为 8，跳转到存储 HTTP 主版本号的状态
        case '8': {
          p++;
          match = 8;
          goto s_n_llhttp__internal__n_invoke_store_http_major;
        }
        # 如果是 '9'，则将指针向后移动一位，匹配值设为 9，跳转到存储 HTTP 主版本号的状态
        case '9': {
          p++;
          match = 9;
          goto s_n_llhttp__internal__n_invoke_store_http_major;
        }
        # 如果不是上述任何字符，则跳转到错误状态
        default: {
          goto s_n_llhttp__internal__n_error_27;
        }
      }
      # 不可达的代码，中止程序
      /* UNREACHABLE */;
      abort();
    }
    # 处理请求行中 HTTP 版本号的起始状态
    case s_n_llhttp__internal__n_req_http_start_1:
    # 定义状态 s_n_llhttp__internal__n_req_http_start_1，处理 HTTP 请求开始的状态
    s_n_llhttp__internal__n_req_http_start_1: {
      # 定义匹配序列的结果
      llparse_match_t match_seq;

      # 如果指针已经到达结束位置，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_req_http_start_1;
      }
      # 调用 llparse__match_sequence_id 函数，匹配指定长度的序列
      match_seq = llparse__match_sequence_id(state, p, endp, llparse_blob15, 4);
      # 更新指针位置
      p = match_seq.current;
      # 根据匹配结果进行不同的处理
      switch (match_seq.status) {
        # 如果匹配完成，则向前移动一个位置，跳转到下一个状态
        case kMatchComplete: {
          p++;
          goto s_n_llhttp__internal__n_invoke_load_method;
        }
        # 如果匹配暂停，则返回当前状态
        case kMatchPause: {
          return s_n_llhttp__internal__n_req_http_start_1;
        }
        # 如果匹配失败，则跳转到错误状态
        case kMatchMismatch: {
          goto s_n_llhttp__internal__n_error_30;
        }
      }
      /* UNREACHABLE */;
      # 终止程序执行
      abort();
    }
    # 定义状态 s_n_llhttp__internal__n_req_http_start_2
    case s_n_llhttp__internal__n_req_http_start_2:
    s_n_llhttp__internal__n_req_http_start_2: {
      # 定义匹配序列的结果
      llparse_match_t match_seq;

      # 如果指针已经到达结束位置，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_req_http_start_2;
      }
      # 调用 llparse__match_sequence_id 函数，匹配指定长度的序列
      match_seq = llparse__match_sequence_id(state, p, endp, llparse_blob17, 3);
      # 更新指针位置
      p = match_seq.current;
      # 根据匹配结果进行不同的处理
      switch (match_seq.status) {
        # 如果匹配完成，则向前移动一个位置，跳转到下一个状态
        case kMatchComplete: {
          p++;
          goto s_n_llhttp__internal__n_invoke_load_method_2;
        }
        # 如果匹配暂停，则返回当前状态
        case kMatchPause: {
          return s_n_llhttp__internal__n_req_http_start_2;
        }
        # 如果匹配失败，则跳转到错误状态
        case kMatchMismatch: {
          goto s_n_llhttp__internal__n_error_30;
        }
      }
      /* UNREACHABLE */;
      # 终止程序执行
      abort();
    }
    # 定义状态 s_n_llhttp__internal__n_req_http_start_3
    case s_n_llhttp__internal__n_req_http_start_3:
    # 定义一个名为 s_n_llhttp__internal__n_req_http_start_3 的状态
    s_n_llhttp__internal__n_req_http_start_3: {
      # 定义一个名为 match_seq 的变量，用于存储匹配结果
      llparse_match_t match_seq;

      # 如果指针 p 已经指向输入的结尾，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_req_http_start_3;
      }
      # 调用 llparse__match_sequence_id 函数进行匹配，返回匹配结果
      match_seq = llparse__match_sequence_id(state, p, endp, llparse_blob18, 4);
      # 更新指针 p 的位置
      p = match_seq.current;
      # 根据匹配结果的状态进行不同的处理
      switch (match_seq.status) {
        # 如果匹配完成，则指针 p 向后移动一位，跳转到状态 s_n_llhttp__internal__n_invoke_load_method_3
        case kMatchComplete: {
          p++;
          goto s_n_llhttp__internal__n_invoke_load_method_3;
        }
        # 如果匹配暂停，则返回当前状态
        case kMatchPause: {
          return s_n_llhttp__internal__n_req_http_start_3;
        }
        # 如果匹配不成功，则跳转到状态 s_n_llhttp__internal__n_error_30
        case kMatchMismatch: {
          goto s_n_llhttp__internal__n_error_30;
        }
      }
      /* UNREACHABLE */;
      # 终止程序执行
      abort();
    }
    # 定义一个名为 s_n_llhttp__internal__n_req_http_start 的状态
    case s_n_llhttp__internal__n_req_http_start:
    s_n_llhttp__internal__n_req_http_start: {
      # 如果指针 p 已经指向输入的结尾，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_req_http_start;
      }
      # 根据指针 p 指向的字符进行不同的处理
      switch (*p) {
        # 如果指针 p 指向空格，则指针 p 向后移动一位，跳转到状态 s_n_llhttp__internal__n_req_http_start
        case ' ': {
          p++;
          goto s_n_llhttp__internal__n_req_http_start;
        }
        # 如果指针 p 指向 'H'，则指针 p 向后移动一位，跳转到状态 s_n_llhttp__internal__n_req_http_start_1
        case 'H': {
          p++;
          goto s_n_llhttp__internal__n_req_http_start_1;
        }
        # 如果指针 p 指向 'I'，则指针 p 向后移动一位，跳转到状态 s_n_llhttp__internal__n_req_http_start_2
        case 'I': {
          p++;
          goto s_n_llhttp__internal__n_req_http_start_2;
        }
        # 如果指针 p 指向 'R'，则指针 p 向后移动一位，跳转到状态 s_n_llhttp__internal__n_req_http_start_3
        case 'R': {
          p++;
          goto s_n_llhttp__internal__n_req_http_start_3;
        }
        # 如果指针 p 指向其他字符，则跳转到状态 s_n_llhttp__internal__n_error_30
        default: {
          goto s_n_llhttp__internal__n_error_30;
        }
      }
      /* UNREACHABLE */;
      # 终止程序执行
      abort();
    }
    # 定义一个名为 s_n_llhttp__internal__n_url_skip_to_http 的状态
    case s_n_llhttp__internal__n_url_skip_to_http:
    s_n_llhttp__internal__n_url_skip_to_http: {
      # 如果指针 p 已经指向输入的结尾，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_url_skip_to_http;
      }
      # 指针 p 向后移动一位，跳转到状态 s_n_llhttp__internal__n_invoke_llhttp__on_url_complete_1
      p++;
      /* UNREACHABLE */;
      # 终止程序执行
      abort();
    }
    # 定义一个名为 s_n_llhttp__internal__n_url_fragment 的状态
    # 定义静态查找表，用于根据输入字符进行快速查找
    s_n_llhttp__internal__n_url_fragment: {
      static uint8_t lookup_table[] = {
        # 静态查找表的内容
        # ...
      };
      # 如果指针已经指向输入的末尾，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_url_fragment;
      }
      # 根据查找表的结果进行不同的处理
      switch (lookup_table[(uint8_t) *p]) {
        # 如果查找表结果为1，则指针向后移动一位，继续处理当前状态
        case 1: {
          p++;
          goto s_n_llhttp__internal__n_url_fragment;
        }
        # 如果查找表结果为2，则跳转到指定状态
        case 2: {
          goto s_n_llhttp__internal__n_span_end_llhttp__on_url_6;
        }
        # 如果查找表结果为3，则跳转到指定状态
        case 3: {
          goto s_n_llhttp__internal__n_span_end_llhttp__on_url_7;
        }
        # 如果查找表结果为4，则跳转到指定状态
        case 4: {
          goto s_n_llhttp__internal__n_span_end_llhttp__on_url_8;
        }
        # 如果查找表结果为其他值，则跳转到错误处理状态
        default: {
          goto s_n_llhttp__internal__n_error_31;
        }
      }
      # 不可达的代码，中断程序执行
      /* UNREACHABLE */;
      abort();
    }
    # 定义状态 s_n_llhttp__internal__n_span_end_stub_query_3
    case s_n_llhttp__internal__n_span_end_stub_query_3:
    s_n_llhttp__internal__n_span_end_stub_query_3: {
      # 如果指针已经指向输入的末尾，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_span_end_stub_query_3;
      }
      # 指针向后移动一位，继续处理状态 s_n_llhttp__internal__n_url_fragment
      p++;
      goto s_n_llhttp__internal__n_url_fragment;
      # 不可达的代码，中断程序执行
      /* UNREACHABLE */;
      abort();
    }
    # 定义状态 s_n_llhttp__internal__n_url_query
    # 定义静态的查找表，用于将字符映射为数字
    static uint8_t lookup_table[] = {
        0, 0, 0, 0, 0, 0, 0, 0, 0, 1, 2, 0, 1, 3, 0, 0,
        0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0,
        4, 1, 1, 5, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1,
        1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1,
        1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1,
        1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1,
        1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1,
        1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 
    # 检查是否已经到达输入的末尾，如果是则返回当前状态
    s_n_llhttp__internal__n_url_query_or_fragment: {
      if (p == endp) {
        return s_n_llhttp__internal__n_url_query_or_fragment;
      }
      # 根据当前字符进行不同的跳转
      switch (*p) {
        # 如果是换行符，则跳转到处理 URL 结束的状态
        case 10: {
          goto s_n_llhttp__internal__n_span_end_llhttp__on_url_3;
        }
        # 如果是回车符，则跳转到处理 URL 结束的状态
        case 13: {
          goto s_n_llhttp__internal__n_span_end_llhttp__on_url_4;
        }
        # 如果是空格，则跳转到处理 URL 结束的状态
        case ' ': {
          goto s_n_llhttp__internal__n_span_end_llhttp__on_url_5;
        }
        # 如果是井号，则跳转到处理 URL 片段的状态
        case '#': {
          p++;
          goto s_n_llhttp__internal__n_url_fragment;
        }
        # 如果是问号，则跳转到处理 URL 查询的状态
        case '?': {
          p++;
          goto s_n_llhttp__internal__n_url_query;
        }
        # 如果是其他字符，则跳转到错误处理状态
        default: {
          goto s_n_llhttp__internal__n_error_33;
        }
      }
      /* UNREACHABLE */;
      abort();
    }
    # 处理 URL 路径的状态
    case s_n_llhttp__internal__n_url_path:
    }
    # 处理 URL 路径的状态
    case s_n_llhttp__internal__n_span_start_stub_path_2:
    s_n_llhttp__internal__n_span_start_stub_path_2: {
      # 检查是否已经到达输入的末尾，如果是则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_span_start_stub_path_2;
      }
      # 移动到下一个字符，并跳转到处理 URL 路径的状态
      p++;
      goto s_n_llhttp__internal__n_url_path;
      /* UNREACHABLE */;
      abort();
    }
    # 处理 URL 路径的状态
    case s_n_llhttp__internal__n_span_start_stub_path:
    s_n_llhttp__internal__n_span_start_stub_path: {
      # 检查是否已经到达输入的末尾，如果是则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_span_start_stub_path;
      }
      # 移动到下一个字符，并跳转到处理 URL 路径的状态
      p++;
      goto s_n_llhttp__internal__n_url_path;
      /* UNREACHABLE */;
      abort();
    }
    # 处理 URL 路径的状态
    case s_n_llhttp__internal__n_span_start_stub_path_1:
    s_n_llhttp__internal__n_span_start_stub_path_1: {
      # 检查是否已经到达输入的末尾，如果是则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_span_start_stub_path_1;
      }
      # 移动到下一个字符，并跳转到处理 URL 路径的状态
      p++;
      goto s_n_llhttp__internal__n_url_path;
      /* UNREACHABLE */;
      abort();
    }
    # 处理带有 @ 符号的 URL 服务器状态
    case s_n_llhttp__internal__n_url_server_with_at:
    # 定义静态查找表，用于根据输入字符进行状态转移
    static uint8_t lookup_table[] = {
        0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 1, 0, 0, 2, 0, 0,
        0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0,
        3, 4, 0, 0, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 5,
        4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 0, 4, 0, 6,
        7, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4,
        4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 0, 4, 0, 4,
        0, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4,
        4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 0, 0, 0, 4, 0,
        0, 0, 0, 0, 0,
    # 定义静态的查找表，用于URL解析状态机的状态转移
    s_n_llhttp__internal__n_url_server: {
      static uint8_t lookup_table[] = {
        # 查找表的数值表示状态转移的目标状态
        0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 1, 0, 0, 2, 0, 0,
        0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0,
        3, 4, 0, 0, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 5,
        4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 0, 4, 0, 6,
        7, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4,
        4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 0, 4, 0, 4,
        0, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4,
        4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 0, 0, 0, 4, 0,
        0, 0, 0, 0, 0, 0, 0
    # 如果当前指针指向末尾，则返回当前状态
    s_n_llhttp__internal__n_url_schema_delim_1: {
      if (p == endp) {
        return s_n_llhttp__internal__n_url_schema_delim_1;
      }
      # 根据当前指针指向的字符进行不同的处理
      switch (*p) {
        # 如果是斜杠，则指针后移并跳转到下一个状态
        case '/': {
          p++;
          goto s_n_llhttp__internal__n_url_server;
        }
        # 如果是其他字符，则跳转到错误状态
        default: {
          goto s_n_llhttp__internal__n_error_38;
        }
      }
      /* UNREACHABLE */;
      abort();
    }
    # 如果当前状态为 s_n_llhttp__internal__n_url_schema_delim
    s_n_llhttp__internal__n_url_schema_delim: {
      # 如果当前指针指向末尾，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_url_schema_delim;
      }
      # 根据当前指针指向的字符进行不同的处理
      switch (*p) {
        # 如果是换行符、回车符或空格，则跳转到错误状态
        case 10: {
          p++;
          goto s_n_llhttp__internal__n_error_37;
        }
        case 13: {
          p++;
          goto s_n_llhttp__internal__n_error_37;
        }
        case ' ': {
          p++;
          goto s_n_llhttp__internal__n_error_37;
        }
        # 如果是斜杠，则指针后移并跳转到下一个状态
        case '/': {
          p++;
          goto s_n_llhttp__internal__n_url_schema_delim_1;
        }
        # 如果是其他字符，则跳转到错误状态
        default: {
          goto s_n_llhttp__internal__n_error_38;
        }
      }
      /* UNREACHABLE */;
      abort();
    }
    # 如果当前状态为 s_n_llhttp__internal__n_span_end_stub_schema
    s_n_llhttp__internal__n_span_end_stub_schema: {
      # 如果当前指针指向末尾，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_span_end_stub_schema;
      }
      # 指针后移并跳转到下一个状态
      p++;
      goto s_n_llhttp__internal__n_url_schema_delim;
      /* UNREACHABLE */;
      abort();
    }
    # 如果当前状态为 s_n_llhttp__internal__n_url_schema:
    # 定义静态数组 lookup_table，用于 URL schema 的查找
    s_n_llhttp__internal__n_url_schema: {
      static uint8_t lookup_table[] = {
        # 初始化 lookup_table 数组
        0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 1, 0, 0, 1, 0, 0,
        0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0,
        1, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0,
        0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 2, 0, 0, 0, 0, 0,
        0, 3, 3, 3, 3, 3, 3, 3, 3, 3, 3, 3, 3, 3, 3, 3,
        3, 3, 3, 3, 3, 3, 3, 3, 3, 3, 3, 0, 0, 0, 0, 0,
        0, 3, 3, 3, 3, 3, 3, 3, 3, 3, 3, 3, 3, 3, 3, 3,
        3, 3, 3, 3, 3, 3, 3, 3, 3, 3, 3, 0, 0
    # 定义静态的查找表，用于根据字符的 ASCII 值进行查找
    s_n_llhttp__internal__n_url_start: {
      static uint8_t lookup_table[] = {
        # 静态查找表的内容
        # ...
      };
      # 如果指针已经指向结束位置，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_url_start;
      }
      # 根据查找表的结果进行不同的处理
      switch (lookup_table[(uint8_t) *p]) {
        # 如果查找表结果为1，则移动指针并跳转到错误处理状态
        case 1: {
          p++;
          goto s_n_llhttp__internal__n_error_37;
        }
        # 如果查找表结果为2，则跳转到路径处理状态
        case 2: {
          goto s_n_llhttp__internal__n_span_start_stub_path_2;
        }
        # 如果查找表结果为3，则跳转到 URL schema 处理状态
        case 3: {
          goto s_n_llhttp__internal__n_url_schema;
        }
        # 如果查找表结果为其他值，则跳转到错误处理状态
        default: {
          goto s_n_llhttp__internal__n_error_40;
        }
      }
      # 不可达的代码，中断程序执行
      /* UNREACHABLE */;
      abort();
    }
    # 定义状态 s_n_llhttp__internal__n_span_start_llhttp__on_url_1
    case s_n_llhttp__internal__n_span_start_llhttp__on_url_1:
    s_n_llhttp__internal__n_span_start_llhttp__on_url_1: {
      # 如果指针已经指向结束位置，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_span_start_llhttp__on_url_1;
      }
      # 设置状态的 span_pos0 和 span_cb0 属性，并跳转到 URL 开始处理状态
      state->_span_pos0 = (void*) p;
      state->_span_cb0 = llhttp__on_url;
      goto s_n_llhttp__internal__n_url_start;
      # 不可达的代码，中断程序执行
      /* UNREACHABLE */;
      abort();
    }
    # 定义状态 s_n_llhttp__internal__n_span_start_llhttp__on_url
    case s_n_llhttp__internal__n_span_start_llhttp__on_url:
    # 如果指针已经到达结束位置，则返回当前状态
    if (p == endp) {
        return s_n_llhttp__internal__n_span_start_llhttp__on_url;
    }
    # 将当前指针位置保存到状态对象中
    state->_span_pos0 = (void*) p;
    # 设置回调函数为 llhttp__on_url
    state->_span_cb0 = llhttp__on_url;
    # 跳转到指定状态
    goto s_n_llhttp__internal__n_url_server;
    # 不可达代码，中止程序
    /* UNREACHABLE */;
    abort();
    # 对应状态 s_n_llhttp__internal__n_req_spaces_before_url
    case s_n_llhttp__internal__n_req_spaces_before_url:
    # 如果指针已经到达结束位置，则返回当前状态
    if (p == endp) {
        return s_n_llhttp__internal__n_req_spaces_before_url;
    }
    # 检查当前字符
    switch (*p) {
        # 如果是空格，则继续移动指针
        case ' ': {
            p++;
            goto s_n_llhttp__internal__n_req_spaces_before_url;
        }
        # 如果不是空格，则跳转到指定状态
        default: {
            goto s_n_llhttp__internal__n_invoke_is_equal_method;
        }
    }
    # 不可达代码，中止程序
    /* UNREACHABLE */;
    abort();
    # 对应状态 s_n_llhttp__internal__n_req_first_space_before_url
    case s_n_llhttp__internal__n_req_first_space_before_url:
    # 如果指针已经到达结束位置，则返回当前状态
    if (p == endp) {
        return s_n_llhttp__internal__n_req_first_space_before_url;
    }
    # 检查当前字符
    switch (*p) {
        # 如果是空格，则继续移动指针
        case ' ': {
            p++;
            goto s_n_llhttp__internal__n_req_spaces_before_url;
        }
        # 如果不是空格，则跳转到指定状态
        default: {
            goto s_n_llhttp__internal__n_error_41;
        }
    }
    # 不可达代码，中止程序
    /* UNREACHABLE */;
    abort();
    # 对应状态 s_n_llhttp__internal__n_start_req_2
    case s_n_llhttp__internal__n_start_req_2:
    # 如果指针已经到达结束位置，则返回当前状态
    if (p == endp) {
        return s_n_llhttp__internal__n_start_req_2;
    }
    # 检查当前字符
    switch (*p) {
        # 如果是 'L'，则继续移动指针并设置匹配值
        case 'L': {
            p++;
            match = 19;
            goto s_n_llhttp__internal__n_invoke_store_method_1;
        }
        # 如果不是 'L'，则跳转到指定状态
        default: {
            goto s_n_llhttp__internal__n_error_49;
        }
    }
    # 不可达代码，中止程序
    /* UNREACHABLE */;
    abort();
    # 对应状态 s_n_llhttp__internal__n_start_req_3
    case s_n_llhttp__internal__n_start_req_3:
    # 定义状态 s_n_llhttp__internal__n_start_req_3，表示解析 HTTP 请求的起始状态
    s_n_llhttp__internal__n_start_req_3: {
      # 定义匹配序列对象
      llparse_match_t match_seq;

      # 如果指针已经到达结束位置，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_start_req_3;
      }
      # 调用 llparse__match_sequence_id 函数，匹配指定长度的数据序列
      match_seq = llparse__match_sequence_id(state, p, endp, llparse_blob19, 6);
      # 更新指针位置
      p = match_seq.current;
      # 根据匹配结果状态进行处理
      switch (match_seq.status) {
        # 如果匹配完成
        case kMatchComplete: {
          # 更新指针位置
          p++;
          # 设置匹配值
          match = 36;
          # 跳转到状态 s_n_llhttp__internal__n_invoke_store_method_1
          goto s_n_llhttp__internal__n_invoke_store_method_1;
        }
        # 如果匹配暂停
        case kMatchPause: {
          # 返回当前状态
          return s_n_llhttp__internal__n_start_req_3;
        }
        # 如果匹配不成功
        case kMatchMismatch: {
          # 跳转到状态 s_n_llhttp__internal__n_error_49
          goto s_n_llhttp__internal__n_error_49;
        }
      }
      /* UNREACHABLE */;
      # 终止程序执行
      abort();
    }
    # 定义状态 s_n_llhttp__internal__n_start_req_1
    case s_n_llhttp__internal__n_start_req_1:
    s_n_llhttp__internal__n_start_req_1: {
      # 如果指针已经到达结束位置，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_start_req_1;
      }
      # 根据当前指针指向的值进行处理
      switch (*p) {
        # 如果值为 'C'
        case 'C': {
          # 更新指针位置
          p++;
          # 跳转到状态 s_n_llhttp__internal__n_start_req_2
          goto s_n_llhttp__internal__n_start_req_2;
        }
        # 如果值为 'N'
        case 'N': {
          # 更新指针位置
          p++;
          # 跳转到状态 s_n_llhttp__internal__n_start_req_3
          goto s_n_llhttp__internal__n_start_req_3;
        }
        # 如果值不符合预期
        default: {
          # 跳转到状态 s_n_llhttp__internal__n_error_49
          goto s_n_llhttp__internal__n_error_49;
        }
      }
      /* UNREACHABLE */;
      # 终止程序执行
      abort();
    }
    # 定义状态 s_n_llhttp__internal__n_start_req_4
    case s_n_llhttp__internal__n_start_req_4:
    s_n_llhttp__internal__n_start_req_4: {
      # 定义匹配序列对象
      llparse_match_t match_seq;

      # 如果指针已经到达结束位置，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_start_req_4;
      }
      # 调用 llparse__match_sequence_id 函数，匹配指定长度的数据序列
      match_seq = llparse__match_sequence_id(state, p, endp, llparse_blob20, 3);
      # 更新指针位置
      p = match_seq.current;
      # 根据匹配结果状态进行处理
      switch (match_seq.status) {
        # 如果匹配完成
        case kMatchComplete: {
          # 更新指针位置
          p++;
          # 设置匹配值
          match = 16;
          # 跳转到状态 s_n_llhttp__internal__n_invoke_store_method_1
          goto s_n_llhttp__internal__n_invoke_store_method_1;
        }
        # 如果匹配暂停
        case kMatchPause: {
          # 返回当前状态
          return s_n_llhttp__internal__n_start_req_4;
        }
        # 如果匹配不成功
        case kMatchMismatch: {
          # 跳转到状态 s_n_llhttp__internal__n_error_49
          goto s_n_llhttp__internal__n_error_49;
        }
      }
      /* UNREACHABLE */;
      # 终止程序执行
      abort();
    }
    # 定义状态 s_n_llhttp__internal__n_start_req_6:
    # 定义状态 s_n_llhttp__internal__n_start_req_6，处理请求开始的第六步
    s_n_llhttp__internal__n_start_req_6: {
      # 定义匹配序列
      llparse_match_t match_seq;

      # 如果指针已经到达结束位置，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_start_req_6;
      }
      # 调用 llparse__match_sequence_id 函数，匹配指定长度的序列
      match_seq = llparse__match_sequence_id(state, p, endp, llparse_blob21, 6);
      # 更新指针位置
      p = match_seq.current;
      # 根据匹配状态进行不同的处理
      switch (match_seq.status) {
        # 如果匹配完成
        case kMatchComplete: {
          # 指针后移一位
          p++;
          # 设置匹配值
          match = 22;
          # 跳转到状态 s_n_llhttp__internal__n_invoke_store_method_1
          goto s_n_llhttp__internal__n_invoke_store_method_1;
        }
        # 如果匹配暂停
        case kMatchPause: {
          # 返回当前状态
          return s_n_llhttp__internal__n_start_req_6;
        }
        # 如果匹配不成功
        case kMatchMismatch: {
          # 跳转到状态 s_n_llhttp__internal__n_error_49
          goto s_n_llhttp__internal__n_error_49;
        }
      }
      /* UNREACHABLE */;
      # 中断程序执行
      abort();
    }
    # 定义状态 s_n_llhttp__internal__n_start_req_8
    case s_n_llhttp__internal__n_start_req_8:
    s_n_llhttp__internal__n_start_req_8: {
      # 定义匹配序列
      llparse_match_t match_seq;

      # 如果指针已经到达结束位置，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_start_req_8;
      }
      # 调用 llparse__match_sequence_id 函数，匹配指定长度的序列
      match_seq = llparse__match_sequence_id(state, p, endp, llparse_blob22, 4);
      # 更新指针位置
      p = match_seq.current;
      # 根据匹配状态进行不同的处理
      switch (match_seq.status) {
        # 如果匹配完成
        case kMatchComplete: {
          # 指针后移一位
          p++;
          # 设置匹配值
          match = 5;
          # 跳转到状态 s_n_llhttp__internal__n_invoke_store_method_1
          goto s_n_llhttp__internal__n_invoke_store_method_1;
        }
        # 如果匹配暂停
        case kMatchPause: {
          # 返回当前状态
          return s_n_llhttp__internal__n_start_req_8;
        }
        # 如果匹配不成功
        case kMatchMismatch: {
          # 跳转到状态 s_n_llhttp__internal__n_error_49
          goto s_n_llhttp__internal__n_error_49;
        }
      }
      /* UNREACHABLE */;
      # 中断程��执行
      abort();
    }
    # 定义状态 s_n_llhttp__internal__n_start_req_9
    case s_n_llhttp__internal__n_start_req_9:
    s_n_llhttp__internal__n_start_req_9: {
      # 如果指针已经到达结束位置，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_start_req_9;
      }
      # 根据当前字符进行不同的处理
      switch (*p) {
        # 如果当前字符为 'Y'
        case 'Y': {
          # 指针后移一位
          p++;
          # 设置匹配值
          match = 8;
          # 跳转到状态 s_n_llhttp__internal__n_invoke_store_method_1
          goto s_n_llhttp__internal__n_invoke_store_method_1;
        }
        # 如果当前字符不为 'Y'
        default: {
          # 跳转到状态 s_n_llhttp__internal__n_error_49
          goto s_n_llhttp__internal__n_error_49;
        }
      }
      /* UNREACHABLE */;
      # 中断程序执行
      abort();
    }
    # 定义状态 s_n_llhttp__internal__n_start_req_7
    # 定义状态 s_n_llhttp__internal__n_start_req_7，处理请求行起始的状态
    s_n_llhttp__internal__n_start_req_7: {
      # 如果指针已经指向输入的结尾，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_start_req_7;
      }
      # 根据当前字符进行不同的处理
      switch (*p) {
        # 如果当前字符是 'N'，则指针向后移动一位，跳转到状态 s_n_llhttp__internal__n_start_req_8
        case 'N': {
          p++;
          goto s_n_llhttp__internal__n_start_req_8;
        }
        # 如果当前字符是 'P'，则指针向后移动一位，跳转到状态 s_n_llhttp__internal__n_start_req_9
        case 'P': {
          p++;
          goto s_n_llhttp__internal__n_start_req_9;
        }
        # 如果当前字符不是 'N' 或 'P'，则跳转到状态 s_n_llhttp__internal__n_error_49
        default: {
          goto s_n_llhttp__internal__n_error_49;
        }
      }
      # 不可达的代码，中止程序
      /* UNREACHABLE */;
      abort();
    }
    # 定义状态 s_n_llhttp__internal__n_start_req_5，处理请求行起始的状态
    case s_n_llhttp__internal__n_start_req_5:
    s_n_llhttp__internal__n_start_req_5: {
      # 如果指针已经指向输入的结尾，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_start_req_5;
      }
      # 根据当前字符进行不同的处理
      switch (*p) {
        # 如果当前字符是 'H'，则指针向后移动一位，跳转到状态 s_n_llhttp__internal__n_start_req_6
        case 'H': {
          p++;
          goto s_n_llhttp__internal__n_start_req_6;
        }
        # 如果当前字符是 'O'，则指针向后移动一位，跳转到状态 s_n_llhttp__internal__n_start_req_7
        case 'O': {
          p++;
          goto s_n_llhttp__internal__n_start_req_7;
        }
        # 如果当前字符既不是 'H' 也不是 'O'，则跳转到状态 s_n_llhttp__internal__n_error_49
        default: {
          goto s_n_llhttp__internal__n_error_49;
        }
      }
      # 不可达的代码，中止程序
      /* UNREACHABLE */;
      abort();
    }
    # 定义状态 s_n_llhttp__internal__n_start_req_12，处理请求行起始的状态
    case s_n_llhttp__internal__n_start_req_12:
    s_n_llhttp__internal__n_start_req_12: {
      # 定义变量 match_seq，用于存储匹配结果
      llparse_match_t match_seq;

      # 如果指针已经指向输入的结尾，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_start_req_12;
      }
      # 调用 llparse__match_sequence_id 函数进行匹配，得到匹配结果
      match_seq = llparse__match_sequence_id(state, p, endp, llparse_blob23, 3);
      # 更新指针位置
      p = match_seq.current;
      # 根据匹配结果进行不同的处理
      switch (match_seq.status) {
        # 如果匹配完成，指针向后移动一位，match 置为 0，跳转到状态 s_n_llhttp__internal__n_invoke_store_method_1
        case kMatchComplete: {
          p++;
          match = 0;
          goto s_n_llhttp__internal__n_invoke_store_method_1;
        }
        # 如果匹配暂停，返回当前状态
        case kMatchPause: {
          return s_n_llhttp__internal__n_start_req_12;
        }
        # 如果匹配不成功，跳转到状态 s_n_llhttp__internal__n_error_49
        case kMatchMismatch: {
          goto s_n_llhttp__internal__n_error_49;
        }
      }
      # 不可达的代码，中止程序
      /* UNREACHABLE */;
      abort();
    }
    # 定义状态 s_n_llhttp__internal__n_start_req_13:
    # 定义状态 s_n_llhttp__internal__n_start_req_13，表示解析 HTTP 请求的起始状态
    s_n_llhttp__internal__n_start_req_13: {
      # 定义变量 match_seq，用于存储匹配结果
      llparse_match_t match_seq;

      # 如果指针 p 已经指向输入结束位置，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_start_req_13;
      }
      # 调用 llparse__match_sequence_id 函数，匹配输入数据和指定的序列 llparse_blob24，最大匹配长度为 5
      match_seq = llparse__match_sequence_id(state, p, endp, llparse_blob24, 5);
      # 更新指针 p 为匹配后的位置
      p = match_seq.current;
      # 根据匹配结果的状态进行不同的处理
      switch (match_seq.status) {
        # 如果匹配完成
        case kMatchComplete: {
          # 指针 p 向前移动一位
          p++;
          # 设置变量 match 为 35
          match = 35;
          # 跳转到状态 s_n_llhttp__internal__n_invoke_store_method_1
          goto s_n_llhttp__internal__n_invoke_store_method_1;
        }
        # 如果匹配暂停
        case kMatchPause: {
          # 返回状态 s_n_llhttp__internal__n_start_req_13
          return s_n_llhttp__internal__n_start_req_13;
        }
        # 如果匹配不成功
        case kMatchMismatch: {
          # 跳转到状态 s_n_llhttp__internal__n_error_49
          goto s_n_llhttp__internal__n_error_49;
        }
      }
      # 不可达代码，中断程序执行
      /* UNREACHABLE */;
      abort();
    }
    # 定义状态 s_n_llhttp__internal__n_start_req_11
    case s_n_llhttp__internal__n_start_req_11:
    s_n_llhttp__internal__n_start_req_11: {
      # 如果指针 p 已经指向输入结束位置，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_start_req_11;
      }
      # 根据指针 p 指向的字符进行不同的处理
      switch (*p) {
        # 如果字符为 'L'
        case 'L': {
          # 指针 p 向前移动一位
          p++;
          # 跳转到状态 s_n_llhttp__internal__n_start_req_12
          goto s_n_llhttp__internal__n_start_req_12;
        }
        # 如果字符为 'S'
        case 'S': {
          # 指针 p 向前移动一位
          p++;
          # 跳转到状态 s_n_llhttp__internal__n_start_req_13
          goto s_n_llhttp__internal__n_start_req_13;
        }
        # 如果字符不是 'L' 或 'S'
        default: {
          # 跳转到状态 s_n_llhttp__internal__n_error_49
          goto s_n_llhttp__internal__n_error_49;
        }
      }
      # 不可达代码，中断程序执行
      /* UNREACHABLE */;
      abort();
    }
    # 定义状态 s_n_llhttp__internal__n_start_req_10
    case s_n_llhttp__internal__n_start_req_10:
    s_n_llhttp__internal__n_start_req_10: {
      # 如果指针 p 已经指向输入结束位置，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_start_req_10;
      }
      # 根据指针 p 指向的字符进行不同的处理
      switch (*p) {
        # 如果字符为 'E'
        case 'E': {
          # 指针 p 向前移动一位
          p++;
          # 跳转到状态 s_n_llhttp__internal__n_start_req_11
          goto s_n_llhttp__internal__n_start_req_11;
        }
        # 如果字符不是 'E'
        default: {
          # 跳转到状态 s_n_llhttp__internal__n_error_49
          goto s_n_llhttp__internal__n_error_49;
        }
      }
      # 不可达代码，中断程序执行
      /* UNREACHABLE */;
      abort();
    }
    # 定义状态 s_n_llhttp__internal__n_start_req_14
    # 定义状态 s_n_llhttp__internal__n_start_req_14，开始解析请求的状态机
    s_n_llhttp__internal__n_start_req_14: {
      # 定义匹配序列的变量
      llparse_match_t match_seq;

      # 如果指针已经到达结束位置，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_start_req_14;
      }
      # 调用 llparse__match_sequence_id 函数，匹配指定长度的序列
      match_seq = llparse__match_sequence_id(state, p, endp, llparse_blob25, 4);
      # 更新指针位置
      p = match_seq.current;
      # 根据匹配状态进行不同的处理
      switch (match_seq.status) {
        # 如果匹配完成
        case kMatchComplete: {
          # 更新指针位置
          p++;
          # 设置匹配值
          match = 45;
          # 跳转到状态 s_n_llhttp__internal__n_invoke_store_method_1
          goto s_n_llhttp__internal__n_invoke_store_method_1;
        }
        # 如果匹配暂停
        case kMatchPause: {
          # 返回当前状态
          return s_n_llhttp__internal__n_start_req_14;
        }
        # 如果匹配不成功
        case kMatchMismatch: {
          # 跳转到状态 s_n_llhttp__internal__n_error_49
          goto s_n_llhttp__internal__n_error_49;
        }
      }
      /* UNREACHABLE */;
      # 终止程序执行
      abort();
    }
    # 定义状态 s_n_llhttp__internal__n_start_req_17
    case s_n_llhttp__internal__n_start_req_17:
    s_n_llhttp__internal__n_start_req_17: {
      # 定义匹配序列的变量
      llparse_match_t match_seq;

      # 如果指针已经到达结束位置，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_start_req_17;
      }
      # 调用 llparse__match_sequence_id 函数，匹配指定长度的序列
      match_seq = llparse__match_sequence_id(state, p, endp, llparse_blob27, 9);
      # 更新指针位置
      p = match_seq.current;
      # 根据匹配状态进行不同的处理
      switch (match_seq.status) {
        # 如果匹配完成
        case kMatchComplete: {
          # 更新指针位置
          p++;
          # 设置匹配值
          match = 41;
          # 跳转到状态 s_n_llhttp__internal__n_invoke_store_method_1
          goto s_n_llhttp__internal__n_invoke_store_method_1;
        }
        # 如果匹配暂停
        case kMatchPause: {
          # 返回当前状态
          return s_n_llhttp__internal__n_start_req_17;
        }
        # 如果匹配不成功
        case kMatchMismatch: {
          # 跳转到状态 s_n_llhttp__internal__n_error_49
          goto s_n_llhttp__internal__n_error_49;
        }
      }
      /* UNREACHABLE */;
      # 终止程序执行
      abort();
    }
    # 定义状态 s_n_llhttp__internal__n_start_req_16
    case s_n_llhttp__internal__n_start_req_16:
    s_n_llhttp__internal__n_start_req_16: {
      # 如果指针已经到达结束位置，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_start_req_16;
      }
      # 根据当前指针位置的值进行不同的处理
      switch (*p) {
        # 如果当前字符为下划线
        case '_': {
          # 更新指针位置
          p++;
          # 跳转到状态 s_n_llhttp__internal__n_start_req_17
          goto s_n_llhttp__internal__n_start_req_17;
        }
        # 如果当前字符不为下划线
        default: {
          # 设置匹配值
          match = 1;
          # 跳转到状态 s_n_llhttp__internal__n_invoke_store_method_1
          goto s_n_llhttp__internal__n_invoke_store_method_1;
        }
      }
      /* UNREACHABLE */;
      # 终止程序执行
      abort();
    }
    # 定义状态 s_n_llhttp__internal__n_start_req_15
    # 定义状态 s_n_llhttp__internal__n_start_req_15，处理请求开始的情况
    s_n_llhttp__internal__n_start_req_15: {
      # 定义匹配序列的变量
      llparse_match_t match_seq;

      # 如果指针已经到达结束位置，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_start_req_15;
      }
      # 调用 llparse__match_sequence_id 函数，匹配指定的序列
      match_seq = llparse__match_sequence_id(state, p, endp, llparse_blob26, 2);
      # 更新指针位置
      p = match_seq.current;
      # 根据匹配的状态进行不同的处理
      switch (match_seq.status) {
        # 如果匹配完成
        case kMatchComplete: {
          # 指针向前移动一位，跳转到状态 s_n_llhttp__internal__n_start_req_16
          p++;
          goto s_n_llhttp__internal__n_start_req_16;
        }
        # 如果匹配暂停
        case kMatchPause: {
          # 返回当前状态
          return s_n_llhttp__internal__n_start_req_15;
        }
        # 如果匹配不成功
        case kMatchMismatch: {
          # 跳转到错误状态 s_n_llhttp__internal__n_error_49
          goto s_n_llhttp__internal__n_error_49;
        }
      }
      # 不可达的代码，中止程序
      /* UNREACHABLE */;
      abort();
    }
    # 定义状态 s_n_llhttp__internal__n_start_req_18，处理请求开始的情况
    s_n_llhttp__internal__n_start_req_18: {
      # 定义匹配序列的变量
      llparse_match_t match_seq;

      # 如果指针已经到达结束位置，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_start_req_18;
      }
      # 调用 llparse__match_sequence_id 函数，匹配指定的序列
      match_seq = llparse__match_sequence_id(state, p, endp, llparse_blob28, 3);
      # 更新指针位置
      p = match_seq.current;
      # 根据匹配的状态进行不同的处理
      switch (match_seq.status) {
        # 如果匹配完成
        case kMatchComplete: {
          # 指针向前移动一位，设置匹配值为 2，跳转到状态 s_n_llhttp__internal__n_invoke_store_method_1
          p++;
          match = 2;
          goto s_n_llhttp__internal__n_invoke_store_method_1;
        }
        # 如果匹配暂停
        case kMatchPause: {
          # 返回当前状态
          return s_n_llhttp__internal__n_start_req_18;
        }
        # 如果匹配不成功
        case kMatchMismatch: {
          # 跳转到错误状态 s_n_llhttp__internal__n_error_49
          goto s_n_llhttp__internal__n_error_49;
        }
      }
      # 不可达的代码，中止程序
      /* UNREACHABLE */;
      abort();
    }
    # 定义状态 s_n_llhttp__internal__n_start_req_20:
    # 定义状态 s_n_llhttp__internal__n_start_req_20，开始解析请求的状态
    s_n_llhttp__internal__n_start_req_20: {
      # 定义匹配序列的变量
      llparse_match_t match_seq;

      # 如果指针已经到达结束位置，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_start_req_20;
      }
      # 调用 llparse__match_sequence_id 函数，匹配指定的序列
      match_seq = llparse__match_sequence_id(state, p, endp, llparse_blob29, 2);
      # 更新指针位置
      p = match_seq.current;
      # 根据匹配的状态进行不同的处理
      switch (match_seq.status) {
        # 如果匹配完成
        case kMatchComplete: {
          # 指针向前移动一位
          p++;
          # 设置匹配值
          match = 31;
          # 跳转到状态 s_n_llhttp__internal__n_invoke_store_method_1
          goto s_n_llhttp__internal__n_invoke_store_method_1;
        }
        # 如果匹配暂停
        case kMatchPause: {
          # 返回当前状态
          return s_n_llhttp__internal__n_start_req_20;
        }
        # 如果匹配不成功
        case kMatchMismatch: {
          # 跳转到状态 s_n_llhttp__internal__n_error_49
          goto s_n_llhttp__internal__n_error_49;
        }
      }
      /* UNREACHABLE */;
      # 终止程序
      abort();
    }
    # 定义状态 s_n_llhttp__internal__n_start_req_21
    case s_n_llhttp__internal__n_start_req_21:
    s_n_llhttp__internal__n_start_req_21: {
      # 定义匹配序列的变量
      llparse_match_t match_seq;

      # 如果指针已经到达结束位置，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_start_req_21;
      }
      # 调用 llparse__match_sequence_id 函数，匹配指定的序列
      match_seq = llparse__match_sequence_id(state, p, endp, llparse_blob30, 2);
      # 更新指针位置
      p = match_seq.current;
      # 根据匹配的状态进行不同的处理
      switch (match_seq.status) {
        # 如果匹配完成
        case kMatchComplete: {
          # 指针向前移动一位
          p++;
          # 设置匹配值
          match = 9;
          # 跳转到状态 s_n_llhttp__internal__n_invoke_store_method_1
          goto s_n_llhttp__internal__n_invoke_store_method_1;
        }
        # 如果匹配暂停
        case kMatchPause: {
          # 返回当前状态
          return s_n_llhttp__internal__n_start_req_21;
        }
        # 如果匹配不成功
        case kMatchMismatch: {
          # 跳转到状态 s_n_llhttp__internal__n_error_49
          goto s_n_llhttp__internal__n_error_49;
        }
      }
      /* UNREACHABLE */;
      # 终止程序
      abort();
    }
    # 定义状态 s_n_llhttp__internal__n_start_req_19
    case s_n_llhttp__internal__n_start_req_19:
    s_n_llhttp__internal__n_start_req_19: {
      # 如果指针已经到达结束位置，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_start_req_19;
      }
      # 根据当前指针指向的字符进行不同的处理
      switch (*p) {
        # 如果是字符 'I'
        case 'I': {
          # 指针向前移动一位
          p++;
          # 跳转到状态 s_n_llhttp__internal__n_start_req_20
          goto s_n_llhttp__internal__n_start_req_20;
        }
        # 如果是字符 'O'
        case 'O': {
          # 指针向前移动一位
          p++;
          # 跳转到状态 s_n_llhttp__internal__n_start_req_21
          goto s_n_llhttp__internal__n_start_req_21;
        }
        # 如果是其它字符
        default: {
          # 跳转到状态 s_n_llhttp__internal__n_error_49
          goto s_n_llhttp__internal__n_error_49;
        }
      }
      /* UNREACHABLE */;
      # 终止程序
      abort();
    }
    # 定义状态 s_n_llhttp__internal__n_start_req_23:
    # 定义状态 s_n_llhttp__internal__n_start_req_23，处理请求开始的状态
    s_n_llhttp__internal__n_start_req_23: {
      # 定义匹配序列的变量
      llparse_match_t match_seq;

      # 如果指针已经到达结束位置，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_start_req_23;
      }
      # 调用 llparse__match_sequence_id 函数，匹配指定长度的序列
      match_seq = llparse__match_sequence_id(state, p, endp, llparse_blob31, 6);
      # 更新指针位置
      p = match_seq.current;
      # 根据匹配的状态进行不同的处理
      switch (match_seq.status) {
        # 如果匹配完成
        case kMatchComplete: {
          # 指针向前移动一位
          p++;
          # 设置匹配值为 24
          match = 24;
          # 跳转到状态 s_n_llhttp__internal__n_invoke_store_method_1
          goto s_n_llhttp__internal__n_invoke_store_method_1;
        }
        # 如果匹配暂停
        case kMatchPause: {
          # 返回当前状态
          return s_n_llhttp__internal__n_start_req_23;
        }
        # 如果匹配不成功
        case kMatchMismatch: {
          # 跳转到状态 s_n_llhttp__internal__n_error_49
          goto s_n_llhttp__internal__n_error_49;
        }
      }
      # 不可达的代码，中断程序
      /* UNREACHABLE */;
      abort();
    }
    # 定义状态 s_n_llhttp__internal__n_start_req_24，处理请求开始的状态
    s_n_llhttp__internal__n_start_req_24: {
      # 定义匹配序列的变量
      llparse_match_t match_seq;

      # 如果指针已经到达结束位置，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_start_req_24;
      }
      # 调用 llparse__match_sequence_id 函数，匹配指定长度的序列
      match_seq = llparse__match_sequence_id(state, p, endp, llparse_blob32, 3);
      # 更新指针位置
      p = match_seq.current;
      # 根据匹配的状态进行不同的处理
      switch (match_seq.status) {
        # 如果匹配完成
        case kMatchComplete: {
          # 指针向前移动一位
          p++;
          # 设置匹配值为 23
          match = 23;
          # 跳转到状态 s_n_llhttp__internal__n_invoke_store_method_1
          goto s_n_llhttp__internal__n_invoke_store_method_1;
        }
        # 如果匹配暂停
        case kMatchPause: {
          # 返回当前状态
          return s_n_llhttp__internal__n_start_req_24;
        }
        # 如果匹配不成功
        case kMatchMismatch: {
          # 跳转到状态 s_n_llhttp__internal__n_error_49
          goto s_n_llhttp__internal__n_error_49;
        }
      }
      # 不可达的代码，中断程序
      /* UNREACHABLE */;
      abort();
    }
    # 定义状态 s_n_llhttp__internal__n_start_req_26:
    # 定义一个名为 s_n_llhttp__internal__n_start_req_26 的状态，包含一个名为 match_seq 的变量
    s_n_llhttp__internal__n_start_req_26: {
      # 定义一个名为 match_seq 的变量，用于存储匹配结果
      llparse_match_t match_seq;

      # 如果指针 p 等于结束指针 endp，则返回当前状态 s_n_llhttp__internal__n_start_req_26
      if (p == endp) {
        return s_n_llhttp__internal__n_start_req_26;
      }
      # 调用 llparse__match_sequence_id 函数，对输入进行匹配，返回匹配结果
      match_seq = llparse__match_sequence_id(state, p, endp, llparse_blob33, 7);
      # 更新指针 p 的位置
      p = match_seq.current;
      # 根据匹配结果的状态进行不同的处理
      switch (match_seq.status) {
        # 如果匹配完成
        case kMatchComplete: {
          # 指针 p 向前移动一位
          p++;
          # 设置 match 变量的值为 21
          match = 21;
          # 跳转到状态 s_n_llhttp__internal__n_invoke_store_method_1
          goto s_n_llhttp__internal__n_invoke_store_method_1;
        }
        # 如果匹配暂停
        case kMatchPause: {
          # 返回当前状态 s_n_llhttp__internal__n_start_req_26
          return s_n_llhttp__internal__n_start_req_26;
        }
        # 如果匹配不成功
        case kMatchMismatch: {
          # 跳转到状态 s_n_llhttp__internal__n_error_49
          goto s_n_llhttp__internal__n_error_49;
        }
      }
      # 不可达的代码，中断程序执行
      /* UNREACHABLE */;
      abort();
    }
    # 定义一个名为 s_n_llhttp__internal__n_start_req_28 的状态，包含一个名为 match_seq 的变量
    case s_n_llhttp__internal__n_start_req_28:
    s_n_llhttp__internal__n_start_req_28: {
      # 定义一个名为 match_seq 的变量，用于存储匹配结果
      llparse_match_t match_seq;

      # 如果指针 p 等于结束指针 endp，则返回当前状态 s_n_llhttp__internal__n_start_req_28
      if (p == endp) {
        return s_n_llhttp__internal__n_start_req_28;
      }
      # 调用 llparse__match_sequence_id 函数，对输入进行匹配，返回匹配结果
      match_seq = llparse__match_sequence_id(state, p, endp, llparse_blob34, 6);
      # 更新指针 p 的位置
      p = match_seq.current;
      # 根据匹配结果的状态进行不同的处理
      switch (match_seq.status) {
        # 如果匹配完成
        case kMatchComplete: {
          # 指针 p 向前移动一位
          p++;
          # 设置 match 变量的值为 30
          match = 30;
          # 跳转到状态 s_n_llhttp__internal__n_invoke_store_method_1
          goto s_n_llhttp__internal__n_invoke_store_method_1;
        }
        # 如果匹配暂停
        case kMatchPause: {
          # 返回当前状态 s_n_llhttp__internal__n_start_req_28
          return s_n_llhttp__internal__n_start_req_28;
        }
        # 如果匹配不成功
        case kMatchMismatch: {
          # 跳转到状态 s_n_llhttp__internal__n_error_49
          goto s_n_llhttp__internal__n_error_49;
        }
      }
      # 不可达的代码，中断程序执行
      /* UNREACHABLE */;
      abort();
    }
    # 定义一个名为 s_n_llhttp__internal__n_start_req_29 的状态
    case s_n_llhttp__internal__n_start_req_29:
    s_n_llhttp__internal__n_start_req_29: {
      # 如果指针 p 等于结束指针 endp，则返回当前状态 s_n_llhttp__internal__n_start_req_29
      if (p == endp) {
        return s_n_llhttp__internal__n_start_req_29;
      }
      # 根据指针 p 指向的值进行不同的处理
      switch (*p) {
        # 如果指针 p 指向的值为 'L'
        case 'L': {
          # 指针 p 向前移动一位
          p++;
          # 设置 match 变量的值为 10
          match = 10;
          # 跳转到状态 s_n_llhttp__internal__n_invoke_store_method_1
          goto s_n_llhttp__internal__n_invoke_store_method_1;
        }
        # 如果指针 p 指向的值不为 'L'
        default: {
          # 跳转到状态 s_n_llhttp__internal__n_error_49
          goto s_n_llhttp__internal__n_error_49;
        }
      }
      # 不可达的代码，中断程序执行
      /* UNREACHABLE */;
      abort();
    }
    # 定义一个名为 s_n_llhttp__internal__n_start_req_27 的状态
    case s_n_llhttp__internal__n_start_req_27:
    # 定义状态 s_n_llhttp__internal__n_start_req_27，处理请求开始的状态
    s_n_llhttp__internal__n_start_req_27: {
      # 如果指针已经指向输入的结尾，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_start_req_27;
      }
      # 根据当前字符进行不同的处理
      switch (*p) {
        # 如果当前字符是 'A'，则向前移动一个字符，跳转到状态 s_n_llhttp__internal__n_start_req_28
        case 'A': {
          p++;
          goto s_n_llhttp__internal__n_start_req_28;
        }
        # 如果当前字符是 'O'，则向前移动一个字符，跳转到状态 s_n_llhttp__internal__n_start_req_29
        case 'O': {
          p++;
          goto s_n_llhttp__internal__n_start_req_29;
        }
        # 如果当前字符不是 'A' 或 'O'，则跳转到状态 s_n_llhttp__internal__n_error_49
        default: {
          goto s_n_llhttp__internal__n_error_49;
        }
      }
      # 不可达的代码，中止程序
      /* UNREACHABLE */;
      abort();
    }
    # 定义状态 s_n_llhttp__internal__n_start_req_25，处理请求开始的状态
    case s_n_llhttp__internal__n_start_req_25:
    s_n_llhttp__internal__n_start_req_25: {
      # 如果指针已经指向输入的结尾，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_start_req_25;
      }
      # 根据当前字符进行不同的处理
      switch (*p) {
        # 如果当前字符是 'A'，则向前移动一个字符，跳转到状态 s_n_llhttp__internal__n_start_req_26
        case 'A': {
          p++;
          goto s_n_llhttp__internal__n_start_req_26;
        }
        # 如果当前字符是 'C'，则向前移动一个字符，跳转到状态 s_n_llhttp__internal__n_start_req_27
        case 'C': {
          p++;
          goto s_n_llhttp__internal__n_start_req_27;
        }
        # 如果当前字符既不是 'A' 也不是 'C'，则跳转到状态 s_n_llhttp__internal__n_error_49
        default: {
          goto s_n_llhttp__internal__n_error_49;
        }
      }
      # 不可达的代码，中止程序
      /* UNREACHABLE */;
      abort();
    }
    # 定义状态 s_n_llhttp__internal__n_start_req_30，处理请求开始的状态
    case s_n_llhttp__internal__n_start_req_30:
    s_n_llhttp__internal__n_start_req_30: {
      # 定义变量 match_seq，用于存储匹配结果
      llparse_match_t match_seq;

      # 如果指针已经指向输入的结尾，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_start_req_30;
      }
      # 调用 llparse__match_sequence_id 函数进行匹配，得到匹配结果
      match_seq = llparse__match_sequence_id(state, p, endp, llparse_blob35, 2);
      # 更新指针位置
      p = match_seq.current;
      # 根据匹配结果进行不同的处理
      switch (match_seq.status) {
        # 如果匹配完成，向前移动一个字符，设置 match 为 11，跳转到状态 s_n_llhttp__internal__n_invoke_store_method_1
        case kMatchComplete: {
          p++;
          match = 11;
          goto s_n_llhttp__internal__n_invoke_store_method_1;
        }
        # 如果匹配暂停，返回当前状态
        case kMatchPause: {
          return s_n_llhttp__internal__n_start_req_30;
        }
        # 如果匹配不成功，跳转到状态 s_n_llhttp__internal__n_error_49
        case kMatchMismatch: {
          goto s_n_llhttp__internal__n_error_49;
        }
      }
      # 不可达的代码，中止程序
      /* UNREACHABLE */;
      abort();
    }
    # 定义状态 s_n_llhttp__internal__n_start_req_22:
    # 如果当前指针指向结束位置，则返回当前状态
    s_n_llhttp__internal__n_start_req_22: {
      if (p == endp) {
        return s_n_llhttp__internal__n_start_req_22;
      }
      # 根据当前字符进行不同的跳转
      switch (*p) {
        # 如果当前字符为'-'，则向前移动一个字符并跳转到状态 s_n_llhttp__internal__n_start_req_23
        case '-': {
          p++;
          goto s_n_llhttp__internal__n_start_req_23;
        }
        # 如果当前字符为'E'，则向前移动一个字符并跳转到状态 s_n_llhttp__internal__n_start_req_24
        case 'E': {
          p++;
          goto s_n_llhttp__internal__n_start_req_24;
        }
        # 如果当前字符为'K'，则向前移动一个字符并跳转到状态 s_n_llhttp__internal__n_start_req_25
        case 'K': {
          p++;
          goto s_n_llhttp__internal__n_start_req_25;
        }
        # 如果当前字符为'O'，则向前移动一个字符并跳转到状态 s_n_llhttp__internal__n_start_req_30
        case 'O': {
          p++;
          goto s_n_llhttp__internal__n_start_req_30;
        }
        # 如果当前字符不匹配任何情况，则跳转到状态 s_n_llhttp__internal__n_error_49
        default: {
          goto s_n_llhttp__internal__n_error_49;
        }
      }
      /* UNREACHABLE */;
      # 终止程序执行
      abort();
    }
    # 如果当前状态为 s_n_llhttp__internal__n_start_req_31，则执行以下代码
    case s_n_llhttp__internal__n_start_req_31:
    s_n_llhttp__internal__n_start_req_31: {
      # 定义匹配结果变量
      llparse_match_t match_seq;

      # 如果当前指针指向结束位置，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_start_req_31;
      }
      # 调用匹配函数，匹配指定长度的字符序列
      match_seq = llparse__match_sequence_id(state, p, endp, llparse_blob36, 5);
      p = match_seq.current;
      # 根据匹配结果进行不同的处理
      switch (match_seq.status) {
        # 如果匹配完成，则向前移动一个字符，设置匹配结果并跳转到状态 s_n_llhttp__internal__n_invoke_store_method_1
        case kMatchComplete: {
          p++;
          match = 25;
          goto s_n_llhttp__internal__n_invoke_store_method_1;
        }
        # 如果匹配暂停，则返回当前状态
        case kMatchPause: {
          return s_n_llhttp__internal__n_start_req_31;
        }
        # 如果匹配不成功，则跳转到状态 s_n_llhttp__internal__n_error_49
        case kMatchMismatch: {
          goto s_n_llhttp__internal__n_error_49;
        }
      }
      /* UNREACHABLE */;
      # 终止程序执行
      abort();
    }
    # 如果当前状态为 s_n_llhttp__internal__n_start_req_32，则执行以下代码
    case s_n_llhttp__internal__n_start_req_32:
    # 定义状态 s_n_llhttp__internal__n_start_req_32，表示解析 HTTP 请求的起始状态
    s_n_llhttp__internal__n_start_req_32: {
      # 定义匹配序列的变量
      llparse_match_t match_seq;

      # 如果指针已经到达结束位置，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_start_req_32;
      }
      # 调用 llparse__match_sequence_id 函数，匹配指定长度的数据
      match_seq = llparse__match_sequence_id(state, p, endp, llparse_blob37, 6);
      # 更新指针位置
      p = match_seq.current;
      # 根据匹配的状态进行不同的处理
      switch (match_seq.status) {
        # 如果匹配完成
        case kMatchComplete: {
          # 指针后移一位
          p++;
          # 设置匹配长度为 6
          match = 6;
          # 跳转到状态 s_n_llhttp__internal__n_invoke_store_method_1
          goto s_n_llhttp__internal__n_invoke_store_method_1;
        }
        # 如果匹配暂停
        case kMatchPause: {
          # 返回当前状态
          return s_n_llhttp__internal__n_start_req_32;
        }
        # 如果匹配不成功
        case kMatchMismatch: {
          # 跳转到状态 s_n_llhttp__internal__n_error_49
          goto s_n_llhttp__internal__n_error_49;
        }
      }
      # 不可达的代码，中断程序
      /* UNREACHABLE */;
      abort();
    }
    # 定义状态 s_n_llhttp__internal__n_start_req_35
    case s_n_llhttp__internal__n_start_req_35:
    s_n_llhttp__internal__n_start_req_35: {
      # 定义匹配序列的变量
      llparse_match_t match_seq;

      # 如果指针已经到达结束位置，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_start_req_35;
      }
      # 调用 llparse__match_sequence_id 函数，匹配指定长度的数据
      match_seq = llparse__match_sequence_id(state, p, endp, llparse_blob38, 2);
      # 更新指针位置
      p = match_seq.current;
      # 根据匹配的状态进行不同的处理
      switch (match_seq.status) {
        # 如果匹配完成
        case kMatchComplete: {
          # 指针后移一位
          p++;
          # 设置匹配长度为 28
          match = 28;
          # 跳转到状态 s_n_llhttp__internal__n_invoke_store_method_1
          goto s_n_llhttp__internal__n_invoke_store_method_1;
        }
        # 如果匹配暂停
        case kMatchPause: {
          # 返回当前状态
          return s_n_llhttp__internal__n_start_req_35;
        }
        # 如果匹配不成功
        case kMatchMismatch: {
          # 跳转到状态 s_n_llhttp__internal__n_error_49
          goto s_n_llhttp__internal__n_error_49;
        }
      }
      # 不可达的代码，中断程序
      /* UNREACHABLE */;
      abort();
    }
    # 定义状态 s_n_llhttp__internal__n_start_req_36:
    # 定义状态 s_n_llhttp__internal__n_start_req_36，表示解析 HTTP 请求的起始状态
    s_n_llhttp__internal__n_start_req_36: {
      # 定义变量 match_seq，用于存储匹配结果
      llparse_match_t match_seq;

      # 如果指针 p 已经指向输入结束位置，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_start_req_36;
      }
      # 调用 llparse__match_sequence_id 函数，匹配输入数据和预定义的序列 llparse_blob39，长度为 2
      match_seq = llparse__match_sequence_id(state, p, endp, llparse_blob39, 2);
      # 更新指针 p 的位置
      p = match_seq.current;
      # 根据匹配结果的状态进行不同的处理
      switch (match_seq.status) {
        # 如果匹配完成
        case kMatchComplete: {
          # 指针 p 向前移动一位
          p++;
          # 设置 match 变量的值为 39
          match = 39;
          # 跳转到状态 s_n_llhttp__internal__n_invoke_store_method_1
          goto s_n_llhttp__internal__n_invoke_store_method_1;
        }
        # 如果匹配暂停
        case kMatchPause: {
          # 返回状态 s_n_llhttp__internal__n_start_req_36
          return s_n_llhttp__internal__n_start_req_36;
        }
        # 如果匹配不成功
        case kMatchMismatch: {
          # 跳转到状态 s_n_llhttp__internal__n_error_49
          goto s_n_llhttp__internal__n_error_49;
        }
      }
      # 不可达代码，中断程序执行
      /* UNREACHABLE */;
      abort();
    }
    # 定义状态 s_n_llhttp__internal__n_start_req_34
    case s_n_llhttp__internal__n_start_req_34:
    s_n_llhttp__internal__n_start_req_34: {
      # 如果指针 p 已经指向输入结束位置，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_start_req_34;
      }
      # 根据指针 p 指向的字符进行不同的处理
      switch (*p) {
        # 如果字符为 'T'
        case 'T': {
          # 指针 p 向前移动一位
          p++;
          # 跳转到状态 s_n_llhttp__internal__n_start_req_35
          goto s_n_llhttp__internal__n_start_req_35;
        }
        # 如果字符为 'U'
        case 'U': {
          # 指针 p 向前移动一位
          p++;
          # 跳转到状态 s_n_llhttp__internal__n_start_req_36
          goto s_n_llhttp__internal__n_start_req_36;
        }
        # 如果字符不是 'T' 或 'U'
        default: {
          # 跳转到状态 s_n_llhttp__internal__n_error_49
          goto s_n_llhttp__internal__n_error_49;
        }
      }
      # 不可达代码，中断程序执行
      /* UNREACHABLE */;
      abort();
    }
    # 定义状态 s_n_llhttp__internal__n_start_req_37
    case s_n_llhttp__internal__n_start_req_37:
    s_n_llhttp__internal__n_start_req_37: {
      # 定义变量 match_seq，用于存储匹配结果
      llparse_match_t match_seq;

      # 如果指针 p 已经指向输入结束位置，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_start_req_37;
      }
      # 调用 llparse__match_sequence_id 函数，匹配输入数据和预定义的序列 llparse_blob40，长度为 2
      match_seq = llparse__match_sequence_id(state, p, endp, llparse_blob40, 2);
      # 更新指针 p 的位置
      p = match_seq.current;
      # 根据匹配结果的状态进行不同的处理
      switch (match_seq.status) {
        # 如果匹配完成
        case kMatchComplete: {
          # 指针 p 向前移动一位
          p++;
          # 设置 match 变量的值为 38
          match = 38;
          # 跳转到状态 s_n_llhttp__internal__n_invoke_store_method_1
          goto s_n_llhttp__internal__n_invoke_store_method_1;
        }
        # 如果匹配暂停
        case kMatchPause: {
          # 返回状态 s_n_llhttp__internal__n_start_req_37
          return s_n_llhttp__internal__n_start_req_37;
        }
        # 如果匹配不成功
        case kMatchMismatch: {
          # 跳转到状态 s_n_llhttp__internal__n_error_49
          goto s_n_llhttp__internal__n_error_49;
        }
      }
      # 不可达代码，中断程序执行
      /* UNREACHABLE */;
      abort();
    }
    # 定义状态 s_n_llhttp__internal__n_start_req_38:
    # 定义状态 s_n_llhttp__internal__n_start_req_38，处理请求开始的状态
    s_n_llhttp__internal__n_start_req_38: {
      # 定义匹配序列的变量
      llparse_match_t match_seq;

      # 如果指针已经到达结束位置，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_start_req_38;
      }
      # 调用 llparse__match_sequence_id 函数，匹配指定的序列
      match_seq = llparse__match_sequence_id(state, p, endp, llparse_blob41, 2);
      # 更新指针位置
      p = match_seq.current;
      # 根据匹配的状态进行不同的处理
      switch (match_seq.status) {
        # 如果匹配完成
        case kMatchComplete: {
          # 更新指针位置
          p++;
          # 设置匹配值
          match = 3;
          # 跳转到指定状态
          goto s_n_llhttp__internal__n_invoke_store_method_1;
        }
        # 如果匹配暂停
        case kMatchPause: {
          # 返回当前状态
          return s_n_llhttp__internal__n_start_req_38;
        }
        # 如果匹配不成功
        case kMatchMismatch: {
          # 跳转到错误状态
          goto s_n_llhttp__internal__n_error_49;
        }
      }
      /* UNREACHABLE */;
      # 终止程序
      abort();
    }
    # 定义状态 s_n_llhttp__internal__n_start_req_42，处理请求开始的状态
    case s_n_llhttp__internal__n_start_req_42:
    s_n_llhttp__internal__n_start_req_42: {
      # 定义匹配序列的变量
      llparse_match_t match_seq;

      # 如果指针已经到达结束位置，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_start_req_42;
      }
      # 调用 llparse__match_sequence_id 函数，匹配指定的序列
      match_seq = llparse__match_sequence_id(state, p, endp, llparse_blob42, 3);
      # 更新指针位置
      p = match_seq.current;
      # 根据匹配的状态进行不同的处理
      switch (match_seq.status) {
        # 如果匹配完成
        case kMatchComplete: {
          # 更新指针位置
          p++;
          # 设置匹配值
          match = 12;
          # 跳转到指定状态
          goto s_n_llhttp__internal__n_invoke_store_method_1;
        }
        # 如果匹配暂停
        case kMatchPause: {
          # 返回当前状态
          return s_n_llhttp__internal__n_start_req_42;
        }
        # 如果匹配不成功
        case kMatchMismatch: {
          # 跳转到错误状态
          goto s_n_llhttp__internal__n_error_49;
        }
      }
      /* UNREACHABLE */;
      # 终止程序
      abort();
    }
    # 定义状态 s_n_llhttp__internal__n_start_req_43:
    # 定义状态 s_n_llhttp__internal__n_start_req_43，表示解析 HTTP 请求的起始状态
    s_n_llhttp__internal__n_start_req_43: {
      # 定义匹配序列的结果变量
      llparse_match_t match_seq;

      # 如果指针已经到达结束位置，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_start_req_43;
      }
      # 调用 llparse__match_sequence_id 函数，匹配指定长度的序列
      match_seq = llparse__match_sequence_id(state, p, endp, llparse_blob43, 4);
      # 更新指针位置
      p = match_seq.current;
      # 根据匹配结果的状态进行不同的处理
      switch (match_seq.status) {
        # 如果匹配完成
        case kMatchComplete: {
          # 更新指针位置
          p++;
          # 设置匹配值
          match = 13;
          # 跳转到状态 s_n_llhttp__internal__n_invoke_store_method_1
          goto s_n_llhttp__internal__n_invoke_store_method_1;
        }
        # 如果匹配暂停
        case kMatchPause: {
          # 返回当前状态
          return s_n_llhttp__internal__n_start_req_43;
        }
        # 如果匹配不成功
        case kMatchMismatch: {
          # 跳转到状态 s_n_llhttp__internal__n_error_49
          goto s_n_llhttp__internal__n_error_49;
        }
      }
      /* UNREACHABLE */;
      # 终止程序执行
      abort();
    }
    # 定义状态 s_n_llhttp__internal__n_start_req_41
    case s_n_llhttp__internal__n_start_req_41:
    s_n_llhttp__internal__n_start_req_41: {
      # 如果指针已经到达结束位置，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_start_req_41;
      }
      # 根据当前字符进行不同的处理
      switch (*p) {
        # 如果当前字符为 'F'
        case 'F': {
          # 更新指针位置
          p++;
          # 跳转到状态 s_n_llhttp__internal__n_start_req_42
          goto s_n_llhttp__internal__n_start_req_42;
        }
        # 如果当前字符为 'P'
        case 'P': {
          # 更新指针位置
          p++;
          # 跳转到状态 s_n_llhttp__internal__n_start_req_43
          goto s_n_llhttp__internal__n_start_req_43;
        }
        # 如果当前字符不是 'F' 也不是 'P'
        default: {
          # 跳转到状态 s_n_llhttp__internal__n_error_49
          goto s_n_llhttp__internal__n_error_49;
        }
      }
      /* UNREACHABLE */;
      # 终止程序执行
      abort();
    }
    # 定义状态 s_n_llhttp__internal__n_start_req_40
    case s_n_llhttp__internal__n_start_req_40:
    s_n_llhttp__internal__n_start_req_40: {
      # 如果指针已经到达结束位置，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_start_req_40;
      }
      # 根据当前字符进行不同的处理
      switch (*p) {
        # 如果当前字符为 'P'
        case 'P': {
          # 更新指针位置
          p++;
          # 跳转到状态 s_n_llhttp__internal__n_start_req_41
          goto s_n_llhttp__internal__n_start_req_41;
        }
        # 如果当前字符不是 'P'
        default: {
          # 跳转到状态 s_n_llhttp__internal__n_error_49
          goto s_n_llhttp__internal__n_error_49;
        }
      }
      /* UNREACHABLE */;
      # 终止程序执行
      abort();
    }
    # 定义状态 s_n_llhttp__internal__n_start_req_39:
    # 定义状态 s_n_llhttp__internal__n_start_req_39，处理请求开始的情况
    s_n_llhttp__internal__n_start_req_39: {
      # 如果指针已经指向输入的结尾，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_start_req_39;
      }
      # 根据当前字符进行判断
      switch (*p) {
        # 如果是 'I'，则向前移动一个字符，设置匹配值为 34，跳转到状态 s_n_llhttp__internal__n_invoke_store_method_1
        case 'I': {
          p++;
          match = 34;
          goto s_n_llhttp__internal__n_invoke_store_method_1;
        }
        # 如果是 'O'，则向前移动一个字符，跳转到状态 s_n_llhttp__internal__n_start_req_40
        case 'O': {
          p++;
          goto s_n_llhttp__internal__n_start_req_40;
        }
        # 如果不是以上情况，则跳转到状态 s_n_llhttp__internal__n_error_49
        default: {
          goto s_n_llhttp__internal__n_error_49;
        }
      }
      /* UNREACHABLE */;
      # 终止程序执行
      abort();
    }
    # 定义状态 s_n_llhttp__internal__n_start_req_45
    case s_n_llhttp__internal__n_start_req_45:
    s_n_llhttp__internal__n_start_req_45: {
      # 定义匹配序列
      llparse_match_t match_seq;

      # 如果指针已经指向输入的结尾，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_start_req_45;
      }
      # 匹配输入序列，返回匹配结果
      match_seq = llparse__match_sequence_id(state, p, endp, llparse_blob44, 2);
      p = match_seq.current;
      # 根据匹配结果进行处理
      switch (match_seq.status) {
        # 如果匹配完成，则向前移动一个字符，设置匹配值为 29，跳转到状态 s_n_llhttp__internal__n_invoke_store_method_1
        case kMatchComplete: {
          p++;
          match = 29;
          goto s_n_llhttp__internal__n_invoke_store_method_1;
        }
        # 如果匹配暂停，则返回当前状态
        case kMatchPause: {
          return s_n_llhttp__internal__n_start_req_45;
        }
        # 如果匹配不成功，则跳转到状态 s_n_llhttp__internal__n_error_49
        case kMatchMismatch: {
          goto s_n_llhttp__internal__n_error_49;
        }
      }
      /* UNREACHABLE */;
      # 终止程序执行
      abort();
    }
    # 定义状态 s_n_llhttp__internal__n_start_req_44
    case s_n_llhttp__internal__n_start_req_44:
    s_n_llhttp__internal__n_start_req_44: {
      # 如果指针已经指向输入的结尾，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_start_req_44;
      }
      # 根据当前字符进行判断
      switch (*p) {
        # 如果是 'R'，则向前移动一个字符，跳转到状态 s_n_llhttp__internal__n_start_req_45
        case 'R': {
          p++;
          goto s_n_llhttp__internal__n_start_req_45;
        }
        # 如果是 'T'，则向前移动一个字符，设置匹配值为 4，跳转到状态 s_n_llhttp__internal__n_invoke_store_method_1
        case 'T': {
          p++;
          match = 4;
          goto s_n_llhttp__internal__n_invoke_store_method_1;
        }
        # 如果不是以上情况，则跳转到状态 s_n_llhttp__internal__n_error_49
        default: {
          goto s_n_llhttp__internal__n_error_49;
        }
      }
      /* UNREACHABLE */;
      # 终止程序执行
      abort();
    }
    # 定义状态 s_n_llhttp__internal__n_start_req_33:
    # 如果指针已经指向结束位置，则返回当前状态
    s_n_llhttp__internal__n_start_req_33: {
      if (p == endp) {
        return s_n_llhttp__internal__n_start_req_33;
      }
      # 根据当前字符进行不同的跳转
      switch (*p) {
        # 如果当前字符为'A'，则向前移动一个字符并跳转到指定状态
        case 'A': {
          p++;
          goto s_n_llhttp__internal__n_start_req_34;
        }
        # 如果当前字符为'L'，则向前移动一个字符并跳转到指定状态
        case 'L': {
          p++;
          goto s_n_llhttp__internal__n_start_req_37;
        }
        # 如果当前字符为'O'，则向前移动一个字符并跳转到指定状态
        case 'O': {
          p++;
          goto s_n_llhttp__internal__n_start_req_38;
        }
        # 如果当前字符为'R'，则向前移动一个字符并跳转到指定状态
        case 'R': {
          p++;
          goto s_n_llhttp__internal__n_start_req_39;
        }
        # 如果当前字符为'U'，则向前移动一个字符并跳转到指定状态
        case 'U': {
          p++;
          goto s_n_llhttp__internal__n_start_req_44;
        }
        # 如果当前字符不匹配任何情况，则跳转到错误状态
        default: {
          goto s_n_llhttp__internal__n_error_49;
        }
      }
      /* UNREACHABLE */;
      # 终止程序
      abort();
    }
    # 如果当前状态为指定状态，则执行以下代码
    case s_n_llhttp__internal__n_start_req_48:
    s_n_llhttp__internal__n_start_req_48: {
      llparse_match_t match_seq;

      # 如果指针已经指向结束位置，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_start_req_48;
      }
      # 匹配指定序列，返回匹配结果和当前指针位置
      match_seq = llparse__match_sequence_id(state, p, endp, llparse_blob45, 3);
      p = match_seq.current;
      # 根据匹配结果执行不同的操作
      switch (match_seq.status) {
        # 如果匹配完成，则向前移动一个字符，设置匹配结果并跳转到指定状态
        case kMatchComplete: {
          p++;
          match = 17;
          goto s_n_llhttp__internal__n_invoke_store_method_1;
        }
        # 如果匹配暂停，则返回当前状态
        case kMatchPause: {
          return s_n_llhttp__internal__n_start_req_48;
        }
        # 如果匹配不成功，则跳转到错误状态
        case kMatchMismatch: {
          goto s_n_llhttp__internal__n_error_49;
        }
      }
      /* UNREACHABLE */;
      # 终止程序
      abort();
    }
    # 如果当前状态为指定状态，则执行以下代码
    case s_n_llhttp__internal__n_start_req_49:
    # 定义状态 s_n_llhttp__internal__n_start_req_49，处理请求开始的状态
    s_n_llhttp__internal__n_start_req_49: {
      # 定义匹配序列的变量
      llparse_match_t match_seq;

      # 如果指针已经到达结束位置，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_start_req_49;
      }
      # 调用 llparse__match_sequence_id 函数，匹配指定的序列
      match_seq = llparse__match_sequence_id(state, p, endp, llparse_blob46, 3);
      # 更新指针位置
      p = match_seq.current;
      # 根据匹配的状态进行不同的处理
      switch (match_seq.status) {
        # 如果匹配完成
        case kMatchComplete: {
          # 更新指针位置
          p++;
          # 设置匹配值
          match = 44;
          # 跳转到状态 s_n_llhttp__internal__n_invoke_store_method_1
          goto s_n_llhttp__internal__n_invoke_store_method_1;
        }
        # 如果匹配暂停
        case kMatchPause: {
          # 返回当前状态
          return s_n_llhttp__internal__n_start_req_49;
        }
        # 如果匹配不成功
        case kMatchMismatch: {
          # 跳转到状态 s_n_llhttp__internal__n_error_49
          goto s_n_llhttp__internal__n_error_49;
        }
      }
      /* UNREACHABLE */;
      # 终止程序执行
      abort();
    }
    # 定义状态 s_n_llhttp__internal__n_start_req_50
    case s_n_llhttp__internal__n_start_req_50:
    s_n_llhttp__internal__n_start_req_50: {
      # 定义匹配序列的变量
      llparse_match_t match_seq;

      # 如果指针已经到达结束位置，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_start_req_50;
      }
      # 调用 llparse__match_sequence_id 函数，匹配指定的序列
      match_seq = llparse__match_sequence_id(state, p, endp, llparse_blob47, 5);
      # 更新指针位置
      p = match_seq.current;
      # 根据匹配的状态进行不同的处理
      switch (match_seq.status) {
        # 如果匹配完成
        case kMatchComplete: {
          # 更新指针位置
          p++;
          # 设置匹配值
          match = 43;
          # 跳转到状态 s_n_llhttp__internal__n_invoke_store_method_1
          goto s_n_llhttp__internal__n_invoke_store_method_1;
        }
        # 如果匹配暂停
        case kMatchPause: {
          # 返回当前状态
          return s_n_llhttp__internal__n_start_req_50;
        }
        # 如果匹配不成功
        case kMatchMismatch: {
          # 跳转到状态 s_n_llhttp__internal__n_error_49
          goto s_n_llhttp__internal__n_error_49;
        }
      }
      /* UNREACHABLE */;
      # 终止程序执行
      abort();
    }
    # 定义状态 s_n_llhttp__internal__n_start_req_51
    # 定义状态 s_n_llhttp__internal__n_start_req_51，开始请求的状态
    s_n_llhttp__internal__n_start_req_51: {
      # 定义匹配序列的变量
      llparse_match_t match_seq;

      # 如果指针已经到达结束位置，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_start_req_51;
      }
      # 调用 llparse__match_sequence_id 函数，匹配指定长度的序列
      match_seq = llparse__match_sequence_id(state, p, endp, llparse_blob48, 3);
      # 更新指针位置
      p = match_seq.current;
      # 根据匹配的状态进行不同的处理
      switch (match_seq.status) {
        # 如果匹配完成
        case kMatchComplete: {
          # 更新指针位置
          p++;
          # 设置匹配值
          match = 20;
          # 跳转到状态 s_n_llhttp__internal__n_invoke_store_method_1
          goto s_n_llhttp__internal__n_invoke_store_method_1;
        }
        # 如果匹配暂停
        case kMatchPause: {
          # 返回当前状态
          return s_n_llhttp__internal__n_start_req_51;
        }
        # 如果匹配不成功
        case kMatchMismatch: {
          # 跳转到状态 s_n_llhttp__internal__n_error_49
          goto s_n_llhttp__internal__n_error_49;
        }
      }
      /* UNREACHABLE */;
      # 终止程序
      abort();
    }
    # 定义状态 s_n_llhttp__internal__n_start_req_47
    case s_n_llhttp__internal__n_start_req_47:
    s_n_llhttp__internal__n_start_req_47: {
      # 如果指针已经到达结束位置，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_start_req_47;
      }
      # 根据当前字符进行不同的处理
      switch (*p) {
        # 如果当前字符为 'B'
        case 'B': {
          # 更新指针位置
          p++;
          # 跳转到状态 s_n_llhttp__internal__n_start_req_48
          goto s_n_llhttp__internal__n_start_req_48;
        }
        # 如果当前字符为 'C'
        case 'C': {
          # 更新指针位置
          p++;
          # 跳转到状态 s_n_llhttp__internal__n_start_req_49
          goto s_n_llhttp__internal__n_start_req_49;
        }
        # 如果当前字符为 'D'
        case 'D': {
          # 更新指针位置
          p++;
          # 跳转到状态 s_n_llhttp__internal__n_start_req_50
          goto s_n_llhttp__internal__n_start_req_50;
        }
        # 如果当前字符为 'P'
        case 'P': {
          # 更新指针位置
          p++;
          # 跳转到状态 s_n_llhttp__internal__n_start_req_51
          goto s_n_llhttp__internal__n_start_req_51;
        }
        # 如果当前字符不匹配任何情况
        default: {
          # 跳转到状态 s_n_llhttp__internal__n_error_49
          goto s_n_llhttp__internal__n_error_49;
        }
      }
      /* UNREACHABLE */;
      # 终止程序
      abort();
    }
    # 定义状态 s_n_llhttp__internal__n_start_req_46
    case s_n_llhttp__internal__n_start_req_46:
    s_n_llhttp__internal__n_start_req_46: {
      # 如果指针已经到达结束位置，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_start_req_46;
      }
      # 根据当前字符进行不同的处理
      switch (*p) {
        # 如果当前字符为 'E'
        case 'E': {
          # 更新指针位置
          p++;
          # 跳转到状态 s_n_llhttp__internal__n_start_req_47
          goto s_n_llhttp__internal__n_start_req_47;
        }
        # 如果当前字符不匹配任何情况
        default: {
          # 跳转到状态 s_n_llhttp__internal__n_error_49
          goto s_n_llhttp__internal__n_error_49;
        }
      }
      /* UNREACHABLE */;
      # 终止程序
      abort();
    }
    # 定义状态 s_n_llhttp__internal__n_start_req_54:
    # 定义状态 s_n_llhttp__internal__n_start_req_54，处理请求开始的状态
    s_n_llhttp__internal__n_start_req_54: {
      # 定义匹配序列的变量
      llparse_match_t match_seq;

      # 如果指针已经到达结束位置，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_start_req_54;
      }
      # 调用 llparse__match_sequence_id 函数，匹配指定长度的序列
      match_seq = llparse__match_sequence_id(state, p, endp, llparse_blob49, 3);
      # 更新指针位置
      p = match_seq.current;
      # 根据匹配的状态进行处理
      switch (match_seq.status) {
        # 如果匹配完成
        case kMatchComplete: {
          # 更新指针位置
          p++;
          # 设置匹配值
          match = 14;
          # 跳转到指定状态
          goto s_n_llhttp__internal__n_invoke_store_method_1;
        }
        # 如果匹配暂停
        case kMatchPause: {
          # 返回当前状态
          return s_n_llhttp__internal__n_start_req_54;
        }
        # 如果匹配不成功
        case kMatchMismatch: {
          # 跳转到错误状态
          goto s_n_llhttp__internal__n_error_49;
        }
      }
      /* UNREACHABLE */;
      # 中断程序执行
      abort();
    }
    # 定义状态 s_n_llhttp__internal__n_start_req_56
    case s_n_llhttp__internal__n_start_req_56:
    s_n_llhttp__internal__n_start_req_56: {
      # 如果指针已经到达结束位置，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_start_req_56;
      }
      # 根据当前字符进行不同的处理
      switch (*p) {
        # 如果当前字符为 'P'
        case 'P': {
          # 更新指针位置
          p++;
          # 设置匹配值
          match = 37;
          # 跳转到指定状态
          goto s_n_llhttp__internal__n_invoke_store_method_1;
        }
        # 如果当前字符不符合预期
        default: {
          # 跳转到错误状态
          goto s_n_llhttp__internal__n_error_49;
        }
      }
      /* UNREACHABLE */;
      # 中断程序执行
      abort();
    }
    # 定义状态 s_n_llhttp__internal__n_start_req_57
    case s_n_llhttp__internal__n_start_req_57:
    s_n_llhttp__internal__n_start_req_57: {
      # 定义匹配序列的变量
      llparse_match_t match_seq;

      # 如果指针已经到达结束位置，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_start_req_57;
      }
      # 调用 llparse__match_sequence_id 函数，匹配指定长度的序列
      match_seq = llparse__match_sequence_id(state, p, endp, llparse_blob50, 9);
      # 更新指针位置
      p = match_seq.current;
      # 根据匹配的状态进行处理
      switch (match_seq.status) {
        # 如果匹配完成
        case kMatchComplete: {
          # 更新指针位置
          p++;
          # 设置匹配值
          match = 42;
          # 跳转到指定状态
          goto s_n_llhttp__internal__n_invoke_store_method_1;
        }
        # 如果匹配暂停
        case kMatchPause: {
          # 返回当前状态
          return s_n_llhttp__internal__n_start_req_57;
        }
        # 如果匹配不成功
        case kMatchMismatch: {
          # 跳转到错误状态
          goto s_n_llhttp__internal__n_error_49;
        }
      }
      /* UNREACHABLE */;
      # 中断程序执行
      abort();
    }
    # 定义状态 s_n_llhttp__internal__n_start_req_55:
    # 定义状态 s_n_llhttp__internal__n_start_req_55，处理请求起始部分的状态机
    s_n_llhttp__internal__n_start_req_55: {
      # 如果指针已经指向输入的结尾，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_start_req_55;
      }
      # 根据当前字符进行不同的处理
      switch (*p) {
        # 如果当前字符是 'U'，则向前移动一个字符，跳转到状态 s_n_llhttp__internal__n_start_req_56
        case 'U': {
          p++;
          goto s_n_llhttp__internal__n_start_req_56;
        }
        # 如果当前字符是 '_'，则向前移动一个字符，跳转到状态 s_n_llhttp__internal__n_start_req_57
        case '_': {
          p++;
          goto s_n_llhttp__internal__n_start_req_57;
        }
        # 如果当前字符不是 'U' 或 '_'，则跳转到状态 s_n_llhttp__internal__n_error_49
        default: {
          goto s_n_llhttp__internal__n_error_49;
        }
      }
      # 不可达的代码，中止程序
      /* UNREACHABLE */;
      abort();
    }
    # 定义状态 s_n_llhttp__internal__n_start_req_53，处理请求起始部分的状态机
    s_n_llhttp__internal__n_start_req_53: {
      # 如果指针已经指向输入的结尾，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_start_req_53;
      }
      # 根据当前字符进行不同的处理
      switch (*p) {
        # 如果当前字符是 'A'，则向前移动一个字符，跳转到状态 s_n_llhttp__internal__n_start_req_54
        case 'A': {
          p++;
          goto s_n_llhttp__internal__n_start_req_54;
        }
        # 如果当前字符是 'T'，则向前移动一个字符，跳转到状态 s_n_llhttp__internal__n_start_req_55
        case 'T': {
          p++;
          goto s_n_llhttp__internal__n_start_req_55;
        }
        # 如果当前字符既不是 'A' 也不是 'T'，则跳转到状态 s_n_llhttp__internal__n_error_49
        default: {
          goto s_n_llhttp__internal__n_error_49;
        }
      }
      # 不可达的代码，中止程序
      /* UNREACHABLE */;
      abort();
    }
    # 定义状态 s_n_llhttp__internal__n_start_req_58，处理请求起始部分的状态机
    s_n_llhttp__internal__n_start_req_58: {
      # 定义匹配结果的结构体
      llparse_match_t match_seq;
      # 如果指针已经指向输入的结尾，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_start_req_58;
      }
      # 调用 llparse__match_sequence_id 函数进行匹配
      match_seq = llparse__match_sequence_id(state, p, endp, llparse_blob51, 4);
      # 更新指针位置
      p = match_seq.current;
      # 根据匹配结果进行不同的处理
      switch (match_seq.status) {
        # 如果匹配完成，向前移动一个字符，设置匹配结果为 33，跳转到状态 s_n_llhttp__internal__n_invoke_store_method_1
        case kMatchComplete: {
          p++;
          match = 33;
          goto s_n_llhttp__internal__n_invoke_store_method_1;
        }
        # 如果匹配暂停，返回当前状态
        case kMatchPause: {
          return s_n_llhttp__internal__n_start_req_58;
        }
        # 如果匹配不成功，跳转到状态 s_n_llhttp__internal__n_error_49
        case kMatchMismatch: {
          goto s_n_llhttp__internal__n_error_49;
        }
      }
      # 不可达的代码，中止程序
      /* UNREACHABLE */;
      abort();
    }
    # 定义状态 s_n_llhttp__internal__n_start_req_59:
    s_n_llhttp__internal__n_start_req_59:
    # 定义状态 s_n_llhttp__internal__n_start_req_59，用于解析 HTTP 请求的起始部分
    s_n_llhttp__internal__n_start_req_59: {
      # 定义匹配序列的变量
      llparse_match_t match_seq;

      # 如果指针已经到达结束位置，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_start_req_59;
      }
      # 调用 llparse__match_sequence_id 函数，匹配指定长度的序列
      match_seq = llparse__match_sequence_id(state, p, endp, llparse_blob52, 7);
      # 更新指针位置
      p = match_seq.current;
      # 根据匹配的状态进行不同的处理
      switch (match_seq.status) {
        # 如果匹配完成
        case kMatchComplete: {
          # 指针后移一位
          p++;
          # 设置匹配值为 26
          match = 26;
          # 跳转到状态 s_n_llhttp__internal__n_invoke_store_method_1
          goto s_n_llhttp__internal__n_invoke_store_method_1;
        }
        # 如果匹配暂停
        case kMatchPause: {
          # 返回当前状态
          return s_n_llhttp__internal__n_start_req_59;
        }
        # 如果匹配不成功
        case kMatchMismatch: {
          # 跳转到状态 s_n_llhttp__internal__n_error_49
          goto s_n_llhttp__internal__n_error_49;
        }
      }
      /* UNREACHABLE */;
      # 终止程序执行
      abort();
    }
    # 定义状态 s_n_llhttp__internal__n_start_req_52
    case s_n_llhttp__internal__n_start_req_52:
    s_n_llhttp__internal__n_start_req_52: {
      # 如果指针已经到达结束位置，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_start_req_52;
      }
      # 根据当前指针指向的字符进行不同的处理
      switch (*p) {
        # 如果是字符 'E'
        case 'E': {
          # 指针后移一位，跳转到状态 s_n_llhttp__internal__n_start_req_53
          p++;
          goto s_n_llhttp__internal__n_start_req_53;
        }
        # 如果是字符 'O'
        case 'O': {
          # 指针后移一位，跳转到状态 s_n_llhttp__internal__n_start_req_58
          p++;
          goto s_n_llhttp__internal__n_start_req_58;
        }
        # 如果是字符 'U'
        case 'U': {
          # 指针后移一位，跳转到状态 s_n_llhttp__internal__n_start_req_59
          p++;
          goto s_n_llhttp__internal__n_start_req_59;
        }
        # 如果是其它字符
        default: {
          # 跳转到状态 s_n_llhttp__internal__n_error_49
          goto s_n_llhttp__internal__n_error_49;
        }
      }
      /* UNREACHABLE */;
      # 终止程序执行
      abort();
    }
    # 定义状态 s_n_llhttp__internal__n_start_req_61
    # 定义状态 s_n_llhttp__internal__n_start_req_61，开始解析请求行
    s_n_llhttp__internal__n_start_req_61: {
      # 定义匹配序列的变量
      llparse_match_t match_seq;

      # 如果指针已经到达结束位置，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_start_req_61;
      }
      # 调用 llparse__match_sequence_id 函数，匹配指定长度的序列
      match_seq = llparse__match_sequence_id(state, p, endp, llparse_blob53, 6);
      # 更新指针位置
      p = match_seq.current;
      # 根据匹配状态进行不同的处理
      switch (match_seq.status) {
        # 如果匹配完成
        case kMatchComplete: {
          # 更新指针位置
          p++;
          # 设置匹配值
          match = 40;
          # 跳转到状态 s_n_llhttp__internal__n_invoke_store_method_1
          goto s_n_llhttp__internal__n_invoke_store_method_1;
        }
        # 如果匹配暂停
        case kMatchPause: {
          # 返回当前状态
          return s_n_llhttp__internal__n_start_req_61;
        }
        # 如果匹配不成功
        case kMatchMismatch: {
          # 跳转到状态 s_n_llhttp__internal__n_error_49
          goto s_n_llhttp__internal__n_error_49;
        }
      }
      /* UNREACHABLE */;
      # 终止程序
      abort();
    }
    # 定义状态 s_n_llhttp__internal__n_start_req_62
    s_n_llhttp__internal__n_start_req_62: {
      # 定义匹配序列的变量
      llparse_match_t match_seq;

      # 如果指针已经到达结束位置，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_start_req_62;
      }
      # 调用 llparse__match_sequence_id 函数，匹配指定长度的序列
      match_seq = llparse__match_sequence_id(state, p, endp, llparse_blob54, 3);
      # 更新指针位置
      p = match_seq.current;
      # 根据匹配状态进行不同的处理
      switch (match_seq.status) {
        # 如果匹配完成
        case kMatchComplete: {
          # 更新指针位置
          p++;
          # 设置匹配值
          match = 7;
          # 跳转到状态 s_n_llhttp__internal__n_invoke_store_method_1
          goto s_n_llhttp__internal__n_invoke_store_method_1;
        }
        # 如果匹配暂停
        case kMatchPause: {
          # 返回当前状态
          return s_n_llhttp__internal__n_start_req_62;
        }
        # 如果匹配不成功
        case kMatchMismatch: {
          # 跳转到状态 s_n_llhttp__internal__n_error_49
          goto s_n_llhttp__internal__n_error_49;
        }
      }
      /* UNREACHABLE */;
      # 终止程序
      abort();
    }
    # 定义状态 s_n_llhttp__internal__n_start_req_60
    s_n_llhttp__internal__n_start_req_60: {
      # 如果指针已经到达结束位置，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_start_req_60;
      }
      # 根据当前字符进行不同的处理
      switch (*p) {
        # 如果当前字符为 'E'
        case 'E': {
          # 更新指针位置
          p++;
          # 跳转到状态 s_n_llhttp__internal__n_start_req_61
          goto s_n_llhttp__internal__n_start_req_61;
        }
        # 如果当前字符为 'R'
        case 'R': {
          # 更新指针位置
          p++;
          # 跳转到状态 s_n_llhttp__internal__n_start_req_62
          goto s_n_llhttp__internal__n_start_req_62;
        }
        # 如果当前字符不匹配任何情况
        default: {
          # 跳转到状态 s_n_llhttp__internal__n_error_49
          goto s_n_llhttp__internal__n_error_49;
        }
      }
      /* UNREACHABLE */;
      # 终止程序
      abort();
    }
    # 定义状态 s_n_llhttp__internal__n_start_req_65:
    # 定义状态 s_n_llhttp__internal__n_start_req_65，开始解析请求的状态
    s_n_llhttp__internal__n_start_req_65: {
      # 定义匹配序列的变量
      llparse_match_t match_seq;

      # 如果指针已经到达结束位置，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_start_req_65;
      }
      # 调用 llparse__match_sequence_id 函数，匹配指定长度的序列
      match_seq = llparse__match_sequence_id(state, p, endp, llparse_blob55, 3);
      # 更新指针位置
      p = match_seq.current;
      # 根据匹配的状态进行不同的处理
      switch (match_seq.status) {
        # 如果匹配完成
        case kMatchComplete: {
          # 更新指针位置
          p++;
          # 设置匹配值
          match = 18;
          # 跳转到指定状态
          goto s_n_llhttp__internal__n_invoke_store_method_1;
        }
        # 如果匹配暂停
        case kMatchPause: {
          # 返回当前状态
          return s_n_llhttp__internal__n_start_req_65;
        }
        # 如果匹配不成功
        case kMatchMismatch: {
          # 跳转到错误状态
          goto s_n_llhttp__internal__n_error_49;
        }
      }
      /* UNREACHABLE */;
      # 终止程序执行
      abort();
    }
    # 定义状态 s_n_llhttp__internal__n_start_req_67
    case s_n_llhttp__internal__n_start_req_67:
    s_n_llhttp__internal__n_start_req_67: {
      # 定义匹配序列的变量
      llparse_match_t match_seq;

      # 如果指针已经到达结束位置，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_start_req_67;
      }
      # 调用 llparse__match_sequence_id 函数，匹配指定长度的序列
      match_seq = llparse__match_sequence_id(state, p, endp, llparse_blob56, 2);
      # 更新指针位置
      p = match_seq.current;
      # 根据匹配的状态进行不同的处理
      switch (match_seq.status) {
        # 如果匹配完成
        case kMatchComplete: {
          # 更新指针位置
          p++;
          # 设置匹配值
          match = 32;
          # 跳转到指定状态
          goto s_n_llhttp__internal__n_invoke_store_method_1;
        }
        # 如果匹配暂停
        case kMatchPause: {
          # 返回当前状态
          return s_n_llhttp__internal__n_start_req_67;
        }
        # 如果匹配不成功
        case kMatchMismatch: {
          # 跳转到错误状态
          goto s_n_llhttp__internal__n_error_49;
        }
      }
      /* UNREACHABLE */;
      # 终止程序执行
      abort();
    }
    # 定义状态 s_n_llhttp__internal__n_start_req_68
    # 定义状态 s_n_llhttp__internal__n_start_req_68，开始解析请求的状态
    s_n_llhttp__internal__n_start_req_68: {
      # 定义匹配序列的变量
      llparse_match_t match_seq;

      # 如果指针已经到达结束位置，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_start_req_68;
      }
      # 调用 llparse__match_sequence_id 函数，匹配指定的序列
      match_seq = llparse__match_sequence_id(state, p, endp, llparse_blob57, 2);
      # 更新指针位置
      p = match_seq.current;
      # 根据匹配的状态进行不同的处理
      switch (match_seq.status) {
        # 如果匹配完成
        case kMatchComplete: {
          # 更新指针位置
          p++;
          # 设置匹配值
          match = 15;
          # 跳转到指定状态
          goto s_n_llhttp__internal__n_invoke_store_method_1;
        }
        # 如果匹配暂停
        case kMatchPause: {
          # 返回当前状态
          return s_n_llhttp__internal__n_start_req_68;
        }
        # 如果匹配不成功
        case kMatchMismatch: {
          # 跳转到错误状态
          goto s_n_llhttp__internal__n_error_49;
        }
      }
      /* UNREACHABLE */;
      # 中断程序执行
      abort();
    }
    # 定义状态 s_n_llhttp__internal__n_start_req_66
    case s_n_llhttp__internal__n_start_req_66:
    s_n_llhttp__internal__n_start_req_66: {
      # 如果指针已经到达结束位置，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_start_req_66;
      }
      # 根据当前字符进行不同的处理
      switch (*p) {
        # 如果当前字符为 'I'
        case 'I': {
          # 更新指针位置
          p++;
          # 跳转到指定状态
          goto s_n_llhttp__internal__n_start_req_67;
        }
        # 如果当前字符为 'O'
        case 'O': {
          # 更新指针位置
          p++;
          # 跳转到指定状态
          goto s_n_llhttp__internal__n_start_req_68;
        }
        # 如果当前字符不符合预期
        default: {
          # 跳转到错误状态
          goto s_n_llhttp__internal__n_error_49;
        }
      }
      /* UNREACHABLE */;
      # 中断程序执行
      abort();
    }
    # 定义状态 s_n_llhttp__internal__n_start_req_69
    case s_n_llhttp__internal__n_start_req_69:
    s_n_llhttp__internal__n_start_req_69: {
      # 定义匹配序列的变量
      llparse_match_t match_seq;

      # 如果指针已经到达结束位置，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_start_req_69;
      }
      # 调用 llparse__match_sequence_id 函数，匹配指定的序列
      match_seq = llparse__match_sequence_id(state, p, endp, llparse_blob58, 8);
      # 更新指针位置
      p = match_seq.current;
      # 根据匹配的状态进行不同的处理
      switch (match_seq.status) {
        # 如果匹配完成
        case kMatchComplete: {
          # 更新指针位置
          p++;
          # 设置匹配值
          match = 27;
          # 跳转到指定状态
          goto s_n_llhttp__internal__n_invoke_store_method_1;
        }
        # 如果匹配暂停
        case kMatchPause: {
          # 返回当前状态
          return s_n_llhttp__internal__n_start_req_69;
        }
        # 如果匹配不成功
        case kMatchMismatch: {
          # 跳转到错误状态
          goto s_n_llhttp__internal__n_error_49;
        }
      }
      /* UNREACHABLE */;
      # 中断程序执行
      abort();
    }
    # 定义状态 s_n_llhttp__internal__n_start_req_64:
    # 定义状态机的状态 s_n_llhttp__internal__n_start_req_64
    s_n_llhttp__internal__n_start_req_64: {
      # 如果指针 p 指向结束位置 endp，则返回状态 s_n_llhttp__internal__n_start_req_64
      if (p == endp) {
        return s_n_llhttp__internal__n_start_req_64;
      }
      # 根据指针指向的字符进行不同的处理
      switch (*p) {
        # 如果指针指向字符 'B'，则向前移动一个位置，跳转到状态 s_n_llhttp__internal__n_start_req_65
        case 'B': {
          p++;
          goto s_n_llhttp__internal__n_start_req_65;
        }
        # 如果指针指向字符 'L'，则向前移动一个位置，跳转到状态 s_n_llhttp__internal__n_start_req_66
        case 'L': {
          p++;
          goto s_n_llhttp__internal__n_start_req_66;
        }
        # 如果指针指向字符 'S'，则向前移动一个位置，跳转到状态 s_n_llhttp__internal__n_start_req_69
        case 'S': {
          p++;
          goto s_n_llhttp__internal__n_start_req_69;
        }
        # 如果指针指向其他字符，则跳转到状态 s_n_llhttp__internal__n_error_49
        default: {
          goto s_n_llhttp__internal__n_error_49;
        }
      }
      # 不可达的代码，中止程序
      /* UNREACHABLE */;
      abort();
    }
    # 定义状态机的状态 s_n_llhttp__internal__n_start_req_63
    case s_n_llhttp__internal__n_start_req_63:
    s_n_llhttp__internal__n_start_req_63: {
      # 如果指针 p 指向结束位置 endp，则返回状态 s_n_llhttp__internal__n_start_req_63
      if (p == endp) {
        return s_n_llhttp__internal__n_start_req_63;
      }
      # 根据指针指向的字符进行不同的处理
      switch (*p) {
        # 如果指针指向字符 'N'，则向前移动一个位置，跳转到状态 s_n_llhttp__internal__n_start_req_64
        case 'N': {
          p++;
          goto s_n_llhttp__internal__n_start_req_64;
        }
        # 如果指针指向其他字符，则跳转到状态 s_n_llhttp__internal__n_error_49
        default: {
          goto s_n_llhttp__internal__n_error_49;
        }
      }
      # 不可达的代码，中止程序
      /* UNREACHABLE */;
      abort();
    }
    # 定义状态机的状态 s_n_llhttp__internal__n_start_req
    # 定义状态机的状态 s_n_llhttp__internal__n_start_req
    s_n_llhttp__internal__n_start_req: {
      # 如果指针 p 指向结束位置，则返回状态 s_n_llhttp__internal__n_start_req
      if (p == endp) {
        return s_n_llhttp__internal__n_start_req;
      }
      # 根据指针指向的字符进行不同的处理
      switch (*p) {
        # 如果指针指向字符 'A'，则向前移动一个位置，跳转到状态 s_n_llhttp__internal__n_start_req_1
        case 'A': {
          p++;
          goto s_n_llhttp__internal__n_start_req_1;
        }
        # 如果指针指向字符 'B'，则向前移动一个位置，跳转到状态 s_n_llhttp__internal__n_start_req_4
        case 'B': {
          p++;
          goto s_n_llhttp__internal__n_start_req_4;
        }
        # 如果指针指向字符 'C'，则向前移动一个位置，跳转到状态 s_n_llhttp__internal__n_start_req_5
        case 'C': {
          p++;
          goto s_n_llhttp__internal__n_start_req_5;
        }
        # 如果指针指向字符 'D'，则向前移动一个位置，跳转到状态 s_n_llhttp__internal__n_start_req_10
        case 'D': {
          p++;
          goto s_n_llhttp__internal__n_start_req_10;
        }
        # 如果指针指向字符 'F'，则向前移动一个位置，跳转到状态 s_n_llhttp__internal__n_start_req_14
        case 'F': {
          p++;
          goto s_n_llhttp__internal__n_start_req_14;
        }
        # 如果指针指向字符 'G'，则向前移动一个位置，跳转到状态 s_n_llhttp__internal__n_start_req_15
        case 'G': {
          p++;
          goto s_n_llhttp__internal__n_start_req_15;
        }
        # 如果指针指向字符 'H'，则向前移动一个位置，跳转到状态 s_n_llhttp__internal__n_start_req_18
        case 'H': {
          p++;
          goto s_n_llhttp__internal__n_start_req_18;
        }
        # 如果指针指向字符 'L'，则向前移动一个位置，跳转到状态 s_n_llhttp__internal__n_start_req_19
        case 'L': {
          p++;
          goto s_n_llhttp__internal__n_start_req_19;
        }
        # 如果指针指向字符 'M'，则向前移动一个位置，跳转到状态 s_n_llhttp__internal__n_start_req_22
        case 'M': {
          p++;
          goto s_n_llhttp__internal__n_start_req_22;
        }
        # 如果指针指向字符 'N'，则向前移动一个位置，跳转到状态 s_n_llhttp__internal__n_start_req_31
        case 'N': {
          p++;
          goto s_n_llhttp__internal__n_start_req_31;
        }
        # 如果指针指向字符 'O'，则向前移动一个位置，跳转到状态 s_n_llhttp__internal__n_start_req_32
        case 'O': {
          p++;
          goto s_n_llhttp__internal__n_start_req_32;
        }
        # 如果指针指向字符 'P'，则向前移动一个位置，跳转到状态 s_n_llhttp__internal__n_start_req_33
        case 'P': {
          p++;
          goto s_n_llhttp__internal__n_start_req_33;
        }
        # 如果指针指向字符 'R'，则向前移动一个位置，跳转到状态 s_n_llhttp__internal__n_start_req_46
        case 'R': {
          p++;
          goto s_n_llhttp__internal__n_start_req_46;
        }
        # 如果指针指向字符 'S'，则向前移动一个位置，跳转到状态 s_n_llhttp__internal__n_start_req_52
        case 'S': {
          p++;
          goto s_n_llhttp__internal__n_start_req_52;
        }
        # 如果指针指向字符 'T'，则向前移动一个位置，跳转到状态 s_n_llhttp__internal__n_start_req_60
        case 'T': {
          p++;
          goto s_n_llhttp__internal__n_start_req_60;
        }
        # 如果指针指向字符 'U'，则向前移动一个位置，跳转到状态 s_n_llhttp__internal__n_start_req_63
        case 'U': {
          p++;
          goto s_n_llhttp__internal__n_start_req_63;
        }
        # 如果指针指向其他字符，则跳转到状态 s_n_llhttp__internal__n_error_49
        default: {
          goto s_n_llhttp__internal__n_error_49;
        }
      }
      /* UNREACHABLE */;
      # 终止程序执行
      abort();
    }
    # case s_n_llhttp__internal__n_invoke_llhttp__on_status_complete:
    # 定义状态转移函数 s_n_llhttp__internal__n_invoke_llhttp__on_status_complete
    s_n_llhttp__internal__n_invoke_llhttp__on_status_complete: {
      # 根据状态和指针调用 llhttp__on_status_complete 函数，并根据返回值进行判断
      switch (llhttp__on_status_complete(state, p, endp)) {
        # 默认情况下，跳转到状态 s_n_llhttp__internal__n_header_field_start
        default:
          goto s_n_llhttp__internal__n_header_field_start;
      }
      /* UNREACHABLE */;
      # 终止程序执行
      abort();
    }
    # 定义状态转移函数 s_n_llhttp__internal__n_res_line_almost_done
    case s_n_llhttp__internal__n_res_line_almost_done:
    s_n_llhttp__internal__n_res_line_almost_done: {
      # 如果指针指向结束位置，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_res_line_almost_done;
      }
      # 指针向后移动一位，然后跳转到状态 s_n_llhttp__internal__n_invoke_llhttp__on_status_complete
      p++;
      goto s_n_llhttp__internal__n_invoke_llhttp__on_status_complete;
      /* UNREACHABLE */;
      # 终止程序执行
      abort();
    }
    # 定义状态转移函数 s_n_llhttp__internal__n_res_status
    case s_n_llhttp__internal__n_res_status:
    s_n_llhttp__internal__n_res_status: {
      # 如果指针指向结束位置，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_res_status;
      }
      # 根据指针指向的值进行判断
      switch (*p) {
        # 如果指针指向换行符，则跳转到状态 s_n_llhttp__internal__n_span_end_llhttp__on_status
        case 10: {
          goto s_n_llhttp__internal__n_span_end_llhttp__on_status;
        }
        # 如果指针指向回车符，则跳转到状态 s_n_llhttp__internal__n_span_end_llhttp__on_status_1
        case 13: {
          goto s_n_llhttp__internal__n_span_end_llhttp__on_status_1;
        }
        # 默认情况下，指针向后移动一位，然后跳转到状态 s_n_llhttp__internal__n_res_status
        default: {
          p++;
          goto s_n_llhttp__internal__n_res_status;
        }
      }
      /* UNREACHABLE */;
      # 终止程序执行
      abort();
    }
    # 定义状态转移函数 s_n_llhttp__internal__n_span_start_llhttp__on_status
    case s_n_llhttp__internal__n_span_start_llhttp__on_status:
    s_n_llhttp__internal__n_span_start_llhttp__on_status: {
      # 如果指针指向结束位置，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_span_start_llhttp__on_status;
      }
      # 设置状态的一些属性，然后跳转到状态 s_n_llhttp__internal__n_res_status
      state->_span_pos0 = (void*) p;
      state->_span_cb0 = llhttp__on_status;
      goto s_n_llhttp__internal__n_res_status;
      /* UNREACHABLE */;
      # 终止程序执行
      abort();
    }
    # 定义状态转移函数 s_n_llhttp__internal__n_res_status_start:
    # 如果指针已经指向末尾，则返回当前状态
    s_n_llhttp__internal__n_res_status_start: {
      if (p == endp) {
        return s_n_llhttp__internal__n_res_status_start;
      }
      # 根据当前字符进行不同的处理
      switch (*p) {
        # 如果是换行符，则指针后移并跳转到状态 s_n_llhttp__internal__n_invoke_llhttp__on_status_complete
        case 10: {
          p++;
          goto s_n_llhttp__internal__n_invoke_llhttp__on_status_complete;
        }
        # 如果是回车符，则指针后移并跳转到状态 s_n_llhttp__internal__n_res_line_almost_done
        case 13: {
          p++;
          goto s_n_llhttp__internal__n_res_line_almost_done;
        }
        # 其他情况下，跳转到状态 s_n_llhttp__internal__n_span_start_llhttp__on_status
        default: {
          goto s_n_llhttp__internal__n_span_start_llhttp__on_status;
        }
      }
      # 不可达的代码，中止程序
      /* UNREACHABLE */;
      abort();
    }
    # 如果状态为 s_n_llhttp__internal__n_res_status_code_otherwise，则执行以下代码
    case s_n_llhttp__internal__n_res_status_code_otherwise:
    s_n_llhttp__internal__n_res_status_code_otherwise: {
      # 如果指针已经指向末尾，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_res_status_code_otherwise;
      }
      # 根据当前字符进行不同的处理
      switch (*p) {
        # 如果是换行符，则跳转到状态 s_n_llhttp__internal__n_res_status_start
        case 10: {
          goto s_n_llhttp__internal__n_res_status_start;
        }
        # 如果是回车符，则跳转到状态 s_n_llhttp__internal__n_res_status_start
        case 13: {
          goto s_n_llhttp__internal__n_res_status_start;
        }
        # 如果是空格，则指针后移并跳转到状态 s_n_llhttp__internal__n_res_status_start
        case ' ': {
          p++;
          goto s_n_llhttp__internal__n_res_status_start;
        }
        # 其他情况下，跳转到状态 s_n_llhttp__internal__n_error_43
        default: {
          goto s_n_llhttp__internal__n_error_43;
        }
      }
      # 不可达的代码，中止程序
      /* UNREACHABLE */;
      abort();
    }
    # 如果状态为 s_n_llhttp__internal__n_res_status_code，则执行以下代码
    case s_n_llhttp__internal__n_res_status_code:
    # 定义状态机的状态 s_n_llhttp__internal__n_res_status_code
    s_n_llhttp__internal__n_res_status_code: {
      # 如果指针 p 已经指向结束位置，则返回状态 s_n_llhttp__internal__n_res_status_code
      if (p == endp) {
        return s_n_llhttp__internal__n_res_status_code;
      }
      # 根据指针指向的字符进行不同的处理
      switch (*p) {
        # 如果是字符 '0'，则将指针向后移动一位，设置匹配值为 0，跳转到状态 s_n_llhttp__internal__n_invoke_mul_add_status_code
        case '0': {
          p++;
          match = 0;
          goto s_n_llhttp__internal__n_invoke_mul_add_status_code;
        }
        # 如果是字符 '1'，则将指针向后移动一位，设置匹配值为 1，跳转到状态 s_n_llhttp__internal__n_invoke_mul_add_status_code
        case '1': {
          p++;
          match = 1;
          goto s_n_llhttp__internal__n_invoke_mul_add_status_code;
        }
        # 如果是字符 '2'，则将指针向后移动一位，设置匹配值为 2，跳转到状态 s_n_llhttp__internal__n_invoke_mul_add_status_code
        case '2': {
          p++;
          match = 2;
          goto s_n_llhttp__internal__n_invoke_mul_add_status_code;
        }
        # 如果是字符 '3'，则将指针向后移动一位，设置匹配值为 3，跳转到状态 s_n_llhttp__internal__n_invoke_mul_add_status_code
        case '3': {
          p++;
          match = 3;
          goto s_n_llhttp__internal__n_invoke_mul_add_status_code;
        }
        # 如果是字符 '4'，则将指针向后移动一位，设置匹配值为 4，跳转到状态 s_n_llhttp__internal__n_invoke_mul_add_status_code
        case '4': {
          p++;
          match = 4;
          goto s_n_llhttp__internal__n_invoke_mul_add_status_code;
        }
        # 如果是字符 '5'，则将指针向后移动一位，设置匹配值为 5，跳转到状态 s_n_llhttp__internal__n_invoke_mul_add_status_code
        case '5': {
          p++;
          match = 5;
          goto s_n_llhttp__internal__n_invoke_mul_add_status_code;
        }
        # 如果是字符 '6'，则将指针向后移动一位，设置匹配值为 6，跳转到状态 s_n_llhttp__internal__n_invoke_mul_add_status_code
        case '6': {
          p++;
          match = 6;
          goto s_n_llhttp__internal__n_invoke_mul_add_status_code;
        }
        # 如果是字符 '7'，则将指针向后移动一位，设置匹配值为 7，跳转到状态 s_n_llhttp__internal__n_invoke_mul_add_status_code
        case '7': {
          p++;
          match = 7;
          goto s_n_llhttp__internal__n_invoke_mul_add_status_code;
        }
        # 如果是字符 '8'，则将指针向后移动一位，设置匹配值为 8，跳转到状态 s_n_llhttp__internal__n_invoke_mul_add_status_code
        case '8': {
          p++;
          match = 8;
          goto s_n_llhttp__internal__n_invoke_mul_add_status_code;
        }
        # 如果是字符 '9'，则将指针向后移动一位，设置匹配值为 9，跳转到状态 s_n_llhttp__internal__n_invoke_mul_add_status_code
        case '9': {
          p++;
          match = 9;
          goto s_n_llhttp__internal__n_invoke_mul_add_status_code;
        }
        # 如果是其他字符，则跳转到状态 s_n_llhttp__internal__n_res_status_code_otherwise
        default: {
          goto s_n_llhttp__internal__n_res_status_code_otherwise;
        }
      }
      # 不可达的代码，中止程序
      /* UNREACHABLE */;
      abort();
    }
    # 其他状态 s_n_llhttp__internal__n_res_http_end:
    # 如果指针 p 等于结束指针 endp，则返回状态 s_n_llhttp__internal__n_res_http_end
    s_n_llhttp__internal__n_res_http_end: {
      if (p == endp) {
        return s_n_llhttp__internal__n_res_http_end;
      }
      # 根据指针 p 指向的字符进行不同的处理
      switch (*p) {
        # 如果指针 p 指向空格，则指针 p 向后移动一位，并跳转到状态 s_n_llhttp__internal__n_invoke_update_status_code
        case ' ': {
          p++;
          goto s_n_llhttp__internal__n_invoke_update_status_code;
        }
        # 如果指针 p 指向其他字符，则跳转到状态 s_n_llhttp__internal__n_error_44
        default: {
          goto s_n_llhttp__internal__n_error_44;
        }
      }
      /* UNREACHABLE */;
      # 终止程序执行
      abort();
    }
    # 处理状态 s_n_llhttp__internal__n_res_http_minor
    case s_n_llhttp__internal__n_res_http_minor:
    # 检查是否已经到达输入的末尾
    s_n_llhttp__internal__n_res_http_minor: {
      if (p == endp) {
        return s_n_llhttp__internal__n_res_http_minor;
      }
      # 根据当前字符进行不同的处理
      switch (*p) {
        # 如果是 '0'，则将匹配值设为 0，并跳转到相应状态
        case '0': {
          p++;
          match = 0;
          goto s_n_llhttp__internal__n_invoke_store_http_minor_1;
        }
        # 如果是 '1'，则将匹配值设为 1，并跳转到相应状态
        case '1': {
          p++;
          match = 1;
          goto s_n_llhttp__internal__n_invoke_store_http_minor_1;
        }
        # 如果是 '2'，则将匹配值设为 2，并跳转到相应状态
        case '2': {
          p++;
          match = 2;
          goto s_n_llhttp__internal__n_invoke_store_http_minor_1;
        }
        # 如果是 '3'，则将匹配值设为 3，并跳转到相应状态
        case '3': {
          p++;
          match = 3;
          goto s_n_llhttp__internal__n_invoke_store_http_minor_1;
        }
        # 如果是 '4'，则将匹配值设为 4，并跳转到相应状态
        case '4': {
          p++;
          match = 4;
          goto s_n_llhttp__internal__n_invoke_store_http_minor_1;
        }
        # 如果是 '5'，则将匹配值设为 5，并跳转到相应状态
        case '5': {
          p++;
          match = 5;
          goto s_n_llhttp__internal__n_invoke_store_http_minor_1;
        }
        # 如果是 '6'，则将匹配值设为 6，并跳转到相应状态
        case '6': {
          p++;
          match = 6;
          goto s_n_llhttp__internal__n_invoke_store_http_minor_1;
        }
        # 如果是 '7'，则将匹配值设为 7，并跳转到相应状态
        case '7': {
          p++;
          match = 7;
          goto s_n_llhttp__internal__n_invoke_store_http_minor_1;
        }
        # 如果是 '8'，则将匹配值设为 8，并跳转到相应状态
        case '8': {
          p++;
          match = 8;
          goto s_n_llhttp__internal__n_invoke_store_http_minor_1;
        }
        # 如果是 '9'，则将匹配值设为 9，并跳转到相应状态
        case '9': {
          p++;
          match = 9;
          goto s_n_llhttp__internal__n_invoke_store_http_minor_1;
        }
        # 如果是其他字符，则跳转到错误状态
        default: {
          goto s_n_llhttp__internal__n_error_45;
        }
      }
      /* UNREACHABLE */;
      # 终止程序
      abort();
    }
    # 处理 HTTP 版本号小数点的情况
    case s_n_llhttp__internal__n_res_http_dot:
    # 如果指针已经指向末尾，则返回当前状态
    if (p == endp) {
        return s_n_llhttp__internal__n_res_http_dot;
    }
    # 根据当前字符进行不同的处理
    switch (*p) {
        # 如果是'.'，则指针向后移动一位，进入下一个状态
        case '.': {
            p++;
            goto s_n_llhttp__internal__n_res_http_minor;
        }
        # 如果不是'.'，则进入错误状态
        default: {
            goto s_n_llhttp__internal__n_error_46;
        }
    }
    # 不可达的代码，中止程序
    /* UNREACHABLE */;
    abort();
    # 进入下一个状态
    case s_n_llhttp__internal__n_res_http_major:
    # 检查解析 HTTP 响应的主版本号
    s_n_llhttp__internal__n_res_http_major: {
      # 如果指针已经到达结束位置，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_res_http_major;
      }
      # 根据当前字符进行不同的处理
      switch (*p) {
        # 如果是 '0'，则匹配成功，跳转到存储 HTTP 主版本号的状态
        case '0': {
          p++;
          match = 0;
          goto s_n_llhttp__internal__n_invoke_store_http_major_1;
        }
        # 如果是 '1'，则匹配成功，跳转到存储 HTTP 主版本号的状态
        case '1': {
          p++;
          match = 1;
          goto s_n_llhttp__internal__n_invoke_store_http_major_1;
        }
        # 如果是 '2'，则匹配成功，跳转到存储 HTTP 主版本号的状态
        case '2': {
          p++;
          match = 2;
          goto s_n_llhttp__internal__n_invoke_store_http_major_1;
        }
        # 如果是 '3'，则匹配成功，跳转到存储 HTTP 主版本号的状态
        case '3': {
          p++;
          match = 3;
          goto s_n_llhttp__internal__n_invoke_store_http_major_1;
        }
        # 如果是 '4'，则匹配成功，跳转到存储 HTTP 主版本号的状态
        case '4': {
          p++;
          match = 4;
          goto s_n_llhttp__internal__n_invoke_store_http_major_1;
        }
        # 如果是 '5'，则匹配成功，跳转到存储 HTTP 主版本号的状态
        case '5': {
          p++;
          match = 5;
          goto s_n_llhttp__internal__n_invoke_store_http_major_1;
        }
        # 如果是 '6'，则匹配成功，跳转到存储 HTTP 主版本号的状态
        case '6': {
          p++;
          match = 6;
          goto s_n_llhttp__internal__n_invoke_store_http_major_1;
        }
        # 如果是 '7'，则匹配成功，跳转到存储 HTTP 主版本号的状态
        case '7': {
          p++;
          match = 7;
          goto s_n_llhttp__internal__n_invoke_store_http_major_1;
        }
        # 如果是 '8'，则匹配成功，跳转到存储 HTTP 主版本号的状态
        case '8': {
          p++;
          match = 8;
          goto s_n_llhttp__internal__n_invoke_store_http_major_1;
        }
        # 如果是 '9'，则匹配成功，跳转到存储 HTTP 主版本号的状态
        case '9': {
          p++;
          match = 9;
          goto s_n_llhttp__internal__n_invoke_store_http_major_1;
        }
        # 如果是其他字符，则跳转到错误状态
        default: {
          goto s_n_llhttp__internal__n_error_47;
        }
      }
      # 不可达的代码，中止程序
      /* UNREACHABLE */;
      abort();
    }
    # 解析 HTTP 响应的起始状态
    case s_n_llhttp__internal__n_start_res:
    # 定义状态 s_n_llhttp__internal__n_start_res，用于解析 HTTP 响应起始行
    s_n_llhttp__internal__n_start_res: {
      # 定义匹配序列对象
      llparse_match_t match_seq;

      # 如果指针已经到达结束位置，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_start_res;
      }
      # 调用 llparse__match_sequence_id 函数，匹配指定长度的数据序列
      match_seq = llparse__match_sequence_id(state, p, endp, llparse_blob59, 5);
      # 更新指针位置
      p = match_seq.current;
      # 根据匹配状态进行不同的处理
      switch (match_seq.status) {
        # 如果匹配完成，则向前移动一个位置，跳转到解析 HTTP 响应起始行的下一个状态
        case kMatchComplete: {
          p++;
          goto s_n_llhttp__internal__n_res_http_major;
        }
        # 如果匹配暂停，则返回当前状态
        case kMatchPause: {
          return s_n_llhttp__internal__n_start_res;
        }
        # 如果匹配不成功，则跳转到错误处理状态
        case kMatchMismatch: {
          goto s_n_llhttp__internal__n_error_50;
        }
      }
      /* UNREACHABLE */;
      # 终止程序执行
      abort();
    }
    # 定义状态 s_n_llhttp__internal__n_req_or_res_method_2
    case s_n_llhttp__internal__n_req_or_res_method_2:
    s_n_llhttp__internal__n_req_or_res_method_2: {
      # 定义匹配序列对象
      llparse_match_t match_seq;

      # 如果指针已经到达结束位置，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_req_or_res_method_2;
      }
      # 调用 llparse__match_sequence_id 函数，匹配指定长度的数据序列
      match_seq = llparse__match_sequence_id(state, p, endp, llparse_blob60, 2);
      # 更新指针位置
      p = match_seq.current;
      # 根据匹配状态进行不同的处理
      switch (match_seq.status) {
        # 如果匹配完成，则向前移动一个位置，设置匹配标志为 2，跳转到存储方法的状态
        case kMatchComplete: {
          p++;
          match = 2;
          goto s_n_llhttp__internal__n_invoke_store_method;
        }
        # 如果匹配暂停，则返回当前状态
        case kMatchPause: {
          return s_n_llhttp__internal__n_req_or_res_method_2;
        }
        # 如果匹配不成功，则跳转到错误处理状态
        case kMatchMismatch: {
          goto s_n_llhttp__internal__n_error_48;
        }
      }
      /* UNREACHABLE */;
      # 终止程序执行
      abort();
    }
    # 定义状态 s_n_llhttp__internal__n_req_or_res_method_3:
    # 定义状态 s_n_llhttp__internal__n_req_or_res_method_3，表示解析请求或响应方法的状态
    s_n_llhttp__internal__n_req_or_res_method_3: {
      # 定义匹配序列的变量
      llparse_match_t match_seq;

      # 如果指针已经指向输入的结尾，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_req_or_res_method_3;
      }
      # 调用 llparse__match_sequence_id 函数，匹配输入的数据和指定的序列 llparse_blob61，长度为 3
      match_seq = llparse__match_sequence_id(state, p, endp, llparse_blob61, 3);
      # 更新指针位置
      p = match_seq.current;
      # 根据匹配的状态进行不同的处理
      switch (match_seq.status) {
        # 如果匹配完成，则指针向后移动一位，跳转到状态 s_n_llhttp__internal__n_invoke_update_type_1
        case kMatchComplete: {
          p++;
          goto s_n_llhttp__internal__n_invoke_update_type_1;
        }
        # 如果匹配暂停，则返回当前状态
        case kMatchPause: {
          return s_n_llhttp__internal__n_req_or_res_method_3;
        }
        # 如果匹配不成功，则跳转到状态 s_n_llhttp__internal__n_error_48
        case kMatchMismatch: {
          goto s_n_llhttp__internal__n_error_48;
        }
      }
      /* UNREACHABLE */;
      # 终止程序执行
      abort();
    }
    # 定义状态 s_n_llhttp__internal__n_req_or_res_method_1
    case s_n_llhttp__internal__n_req_or_res_method_1:
    s_n_llhttp__internal__n_req_or_res_method_1: {
      # 如果指针已经指向输入的结尾，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_req_or_res_method_1;
      }
      # 根据当前指针指向的字符进行不同的处理
      switch (*p) {
        # 如果当前字符为 'E'，则指针向后移动一位，跳转到状态 s_n_llhttp__internal__n_req_or_res_method_2
        case 'E': {
          p++;
          goto s_n_llhttp__internal__n_req_or_res_method_2;
        }
        # 如果当前字符为 'T'，则指针向后移动一位，跳转到状态 s_n_llhttp__internal__n_req_or_res_method_3
        case 'T': {
          p++;
          goto s_n_llhttp__internal__n_req_or_res_method_3;
        }
        # 如果当前字符不匹配任何情况，则跳转到状态 s_n_llhttp__internal__n_error_48
        default: {
          goto s_n_llhttp__internal__n_error_48;
        }
      }
      /* UNREACHABLE */;
      # 终止程序执行
      abort();
    }
    # 定义状态 s_n_llhttp__internal__n_req_or_res_method
    case s_n_llhttp__internal__n_req_or_res_method:
    s_n_llhttp__internal__n_req_or_res_method: {
      # 如果指针已经指向输入的结尾，则返回当前状态
      if (p == endp) {
        return s_n_llhttp__internal__n_req_or_res_method;
      }
      # 根据当前指针指向的字符进行不同的处理
      switch (*p) {
        # 如果当前字符为 'H'，则指针向后移动一位，跳转到状态 s_n_llhttp__internal__n_req_or_res_method_1
        case 'H': {
          p++;
          goto s_n_llhttp__internal__n_req_or_res_method_1;
        }
        # 如果当前字符不匹配任何情况，则跳转到状态 s_n_llhttp__internal__n_error_48
        default: {
          goto s_n_llhttp__internal__n_error_48;
        }
      }
      /* UNREACHABLE */;
      # 终止程序执行
      abort();
    }
    # 定义状态 s_n_llhttp__internal__n_start_req_or_res:
    # 定义状态机的状态 s_n_llhttp__internal__n_start_req_or_res
    s_n_llhttp__internal__n_start_req_or_res: {
      # 如果指针 p 指向结束位置 endp，则返回状态 s_n_llhttp__internal__n_start_req_or_res
      if (p == endp) {
        return s_n_llhttp__internal__n_start_req_or_res;
      }
      # 根据指针 p 指向的字符进行判断
      switch (*p) {
        # 如果指针 p 指向字符 'H'，则跳转到状态 s_n_llhttp__internal__n_req_or_res_method
        case 'H': {
          goto s_n_llhttp__internal__n_req_or_res_method;
        }
        # 如果指针 p 指向其他字符，则跳转到状态 s_n_llhttp__internal__n_invoke_update_type_2
        default: {
          goto s_n_llhttp__internal__n_invoke_update_type_2;
        }
      }
      # 不可达的代码，中止程序
      /* UNREACHABLE */;
      abort();
    }
    # 定义状态机的状态 s_n_llhttp__internal__n_invoke_load_type
    case s_n_llhttp__internal__n_invoke_load_type:
    s_n_llhttp__internal__n_invoke_load_type: {
      # 根据调用 llhttp__internal__c_load_type 函数的返回值进行判断
      switch (llhttp__internal__c_load_type(state, p, endp)) {
        # 如果返回值为 1，则跳转到状态 s_n_llhttp__internal__n_start_req
        case 1:
          goto s_n_llhttp__internal__n_start_req;
        # 如果返回值为 2，则跳转到状态 s_n_llhttp__internal__n_start_res
        case 2:
          goto s_n_llhttp__internal__n_start_res;
        # 如果返回值为其他值，则跳转到状态 s_n_llhttp__internal__n_start_req_or_res
        default:
          goto s_n_llhttp__internal__n_start_req_or_res;
      }
      # 不可达的代码，中止程序
      /* UNREACHABLE */;
      abort();
    }
    # 定义状态机的状态 s_n_llhttp__internal__n_start
    case s_n_llhttp__internal__n_start:
    s_n_llhttp__internal__n_start: {
      # 如果指针 p 指向结束位置 endp，则返回状态 s_n_llhttp__internal__n_start
      if (p == endp) {
        return s_n_llhttp__internal__n_start;
      }
      # 根据指针 p 指向的字符进行判断
      switch (*p) {
        # 如果指针 p 指向换行符，则将指针 p 向后移动一位，然后跳转到状态 s_n_llhttp__internal__n_start
        case 10: {
          p++;
          goto s_n_llhttp__internal__n_start;
        }
        # 如果指针 p 指向回车符，则将指针 p 向后移动一位，然后跳转到状态 s_n_llhttp__internal__n_start
        case 13: {
          p++;
          goto s_n_llhttp__internal__n_start;
        }
        # 如果指针 p 指向其他字符，则跳转到状态 s_n_llhttp__internal__n_invoke_update_finish
        default: {
          goto s_n_llhttp__internal__n_invoke_update_finish;
        }
      }
      # 不可达的代码，中止程序
      /* UNREACHABLE */;
      abort();
    }
    # 默认情况下，中止程序
    default:
      /* UNREACHABLE */
      abort();
  }
  # 定义状态机的状态 s_n_llhttp__internal__n_error_37
  s_n_llhttp__internal__n_error_37: {
    # 设置状态机的错误码为 0x7
    state->error = 0x7;
    # 设置错误原因为 "Invalid characters in url"
    state->reason = "Invalid characters in url";
    # 设置错误位置为指针 p 的位置
    state->error_pos = (const char*) p;
    # 设置当前状态为错误状态 s_error
    state->_current = (void*) (intptr_t) s_error;
    # 返回错误状态 s_error
    return s_error;
    # 不可达的代码，中止程序
    /* UNREACHABLE */;
    abort();
  }
  # 定义状态机的状态 s_n_llhttp__internal__n_invoke_update_finish_2
  s_n_llhttp__internal__n_invoke_update_finish_2: {
    # 根据调用 llhttp__internal__c_update_finish_1 函数的返回值进行判断
    switch (llhttp__internal__c_update_finish_1(state, p, endp)) {
      # 默认情况下，跳转到状态 s_n_llhttp__internal__n_start
      default:
        goto s_n_llhttp__internal__n_start;
    }
    # 不可达的代码，中止程序
    /* UNREACHABLE */;
    abort();
  }
  # 定义状态机的状态 s_n_llhttp__internal__n_invoke_test_lenient_flags
    switch (llhttp__internal__c_test_lenient_flags(state, p, endp)) {
      case 1:
        // 如果测试宽松标志成功，则跳转到 s_n_llhttp__internal__n_invoke_update_finish_2 标签处
        goto s_n_llhttp__internal__n_invoke_update_finish_2;
      default:
        // 否则跳转到 s_n_llhttp__internal__n_closed 标签处
        goto s_n_llhttp__internal__n_closed;
    }
    /* UNREACHABLE */;
    // 不可达代码，中断程序执行
    abort();
  }
  s_n_llhttp__internal__n_invoke_update_finish_1: {
    switch (llhttp__internal__c_update_finish_1(state, p, endp)) {
      default:
        // 调用更新完成回调函数失败，则跳转到 s_n_llhttp__internal__n_invoke_test_lenient_flags 标签处
        goto s_n_llhttp__internal__n_invoke_test_lenient_flags;
    }
    /* UNREACHABLE */;
    // 不可达代码，中断程序执行
    abort();
  }
  s_n_llhttp__internal__n_pause_5: {
    // 设置状态错误码和原因
    state->error = 0x15;
    state->reason = "on_message_complete pause";
    state->error_pos = (const char*) p;
    state->_current = (void*) (intptr_t) s_n_llhttp__internal__n_invoke_is_equal_upgrade;
    return s_error;
    /* UNREACHABLE */;
    // 不可达代码，中断程序执行
    abort();
  }
  s_n_llhttp__internal__n_error_9: {
    // 设置状态错误码和原因
    state->error = 0x12;
    state->reason = "`on_message_complete` callback error";
    state->error_pos = (const char*) p;
    state->_current = (void*) (intptr_t) s_error;
    return s_error;
    /* UNREACHABLE */;
    // 不可达代码，中断程序执行
    abort();
  }
  s_n_llhttp__internal__n_pause_7: {
    // 设置状态错误码和原因
    state->error = 0x15;
    state->reason = "on_chunk_complete pause";
    state->error_pos = (const char*) p;
    state->_current = (void*) (intptr_t) s_n_llhttp__internal__n_invoke_llhttp__on_message_complete_2;
    return s_error;
    /* UNREACHABLE */;
    // 不可达代码，中断程序执行
    abort();
  }
  s_n_llhttp__internal__n_error_12: {
    // 设置状态错误码和原因
    state->error = 0x14;
    state->reason = "`on_chunk_complete` callback error";
    state->error_pos = (const char*) p;
    state->_current = (void*) (intptr_t) s_error;
    return s_error;
    /* UNREACHABLE */;
    // 不可达代码，中断程序执行
    abort();
  }
  s_n_llhttp__internal__n_invoke_llhttp__on_chunk_complete_1: {
    switch (llhttp__on_chunk_complete(state, p, endp)) {
      case 0:
        // 如果分块消息完成回调成功，则跳转到 s_n_llhttp__internal__n_invoke_llhttp__on_message_complete_2 标签处
        goto s_n_llhttp__internal__n_invoke_llhttp__on_message_complete_2;
      case 21:
        // 如果分块消息完成回调暂停，则跳转到 s_n_llhttp__internal__n_pause_7 标签处
        goto s_n_llhttp__internal__n_pause_7;
      default:
        // 其他情况跳转到 s_n_llhttp__internal__n_error_12 标签处
        goto s_n_llhttp__internal__n_error_12;
    }
  /* UNREACHABLE */;
  // 标记代码不可达
  abort();
  // 终止程序执行
}
s_n_llhttp__internal__n_error_11: {
  state->error = 0x4;
  state->reason = "Content-Length can't be present with Transfer-Encoding";
  state->error_pos = (const char*) p;
  state->_current = (void*) (intptr_t) s_error;
  return s_error;
  /* UNREACHABLE */;
  abort();
}
// 标记代码不可达
s_n_llhttp__internal__n_pause_2: {
  state->error = 0x15;
  state->reason = "on_message_complete pause";
  state->error_pos = (const char*) p;
  state->_current = (void*) (intptr_t) s_n_llhttp__internal__n_pause_1;
  return s_error;
  /* UNREACHABLE */;
  abort();
}
// 标记代码不可达
s_n_llhttp__internal__n_error_3: {
  state->error = 0x12;
  state->reason = "`on_message_complete` callback error";
  state->error_pos = (const char*) p;
  state->_current = (void*) (intptr_t) s_error;
  return s_error;
  /* UNREACHABLE */;
  abort();
}
// 标记代码不可达
s_n_llhttp__internal__n_invoke_llhttp__on_message_complete_1: {
  switch (llhttp__on_message_complete(state, p, endp)) {
    case 0:
      goto s_n_llhttp__internal__n_pause_1;
    case 21:
      goto s_n_llhttp__internal__n_pause_2;
    default:
      goto s_n_llhttp__internal__n_error_3;
  }
  /* UNREACHABLE */;
  abort();
}
// 标记代码不可达
s_n_llhttp__internal__n_error_7: {
  state->error = 0xc;
  state->reason = "Chunk size overflow";
  state->error_pos = (const char*) p;
  state->_current = (void*) (intptr_t) s_error;
  return s_error;
  /* UNREACHABLE */;
  abort();
}
// 标记代码不可达
s_n_llhttp__internal__n_pause_3: {
  state->error = 0x15;
  state->reason = "on_chunk_complete pause";
  state->error_pos = (const char*) p;
  state->_current = (void*) (intptr_t) s_n_llhttp__internal__n_invoke_update_content_length;
  return s_error;
  /* UNREACHABLE */;
  abort();
}
// 标记代码不可达
s_n_llhttp__internal__n_error_5: {
  state->error = 0x14;
  state->reason = "`on_chunk_complete` callback error";
  state->error_pos = (const char*) p;
    state->_current = (void*) (intptr_t) s_error;  # 将状态机的当前状态设置为错误状态
    return s_error;  # 返回错误状态
    /* UNREACHABLE */;  # 不可达代码，表示此处的代码不会被执行到
    abort();  # 终止程序执行
  }
  s_n_llhttp__internal__n_invoke_llhttp__on_chunk_complete: {  # 进入状态机的处理块
    switch (llhttp__on_chunk_complete(state, p, endp)) {  # 调用处理函数处理 chunk 完成事件
      case 0:  # 如果处理成功
        goto s_n_llhttp__internal__n_invoke_update_content_length;  # 跳转到更新内容长度的状态
      case 21:  # 如果出现错误码 21
        goto s_n_llhttp__internal__n_pause_3;  # 跳转到暂停状态 3
      default:  # 其他情况
        goto s_n_llhttp__internal__n_error_5;  # 跳转到错误状态 5
    }
    /* UNREACHABLE */;  # 不可达代码
    abort();  # 终止程序执行
  }
  s_n_llhttp__internal__n_span_end_llhttp__on_body: {  # 进入状态机的处理块
    const unsigned char* start;  # 定义无符号字符指针 start
    int err;  # 定义整型变量 err

    start = state->_span_pos0;  # 将状态机的 span_pos0 赋值给 start
    state->_span_pos0 = NULL;  # 将状态机的 span_pos0 置为 NULL
    err = llhttp__on_body(state, start, p);  # 调用处理函数处理 body 事件
    if (err != 0) {  # 如果处理出现错误
      state->error = err;  # 设置状态机的错误码为 err
      state->error_pos = (const char*) p;  # 设置状态机的错误位置为 p
      state->_current = (void*) (intptr_t) s_n_llhttp__internal__n_chunk_data_almost_done;  # 将状态机的当前状态设置为 chunk 数据即将完成状态
      return s_error;  # 返回错误状态
    }
    goto s_n_llhttp__internal__n_chunk_data_almost_done;  # 跳转到 chunk 数据即将完成状态
    /* UNREACHABLE */;  # 不可达代码
    abort();  # 终止程序执行
  }
  s_n_llhttp__internal__n_invoke_or_flags: {  # 进入状态机的处理块
    switch (llhttp__internal__c_or_flags(state, p, endp)) {  # 调用处理函数处理 or flags 事件
      default:  # 默认情况
        goto s_n_llhttp__internal__n_header_field_start;  # 跳转到 header field 开始状态
    }
    /* UNREACHABLE */;  # 不可达代码
    abort();  # 终止程序执行
  }
  s_n_llhttp__internal__n_pause_4: {  # 进入状态机的处理块
    state->error = 0x15;  # 设置状态机的错误码为 0x15
    state->reason = "on_chunk_header pause";  # 设置状态机的错误原因为 "on_chunk_header pause"
    state->error_pos = (const char*) p;  # 设置状态机的错误位置为 p
    state->_current = (void*) (intptr_t) s_n_llhttp__internal__n_invoke_is_equal_content_length;  # 将状态机的当前状态设置为判断内容长度是否相等状态
    return s_error;  # 返回错误状态
    /* UNREACHABLE */;  # 不可达代码
    abort();  # 终止程序执行
  }
  s_n_llhttp__internal__n_error_4: {  # 进入状态机的处理块
    state->error = 0x13;  # 设置状态机的错误码为 0x13
    state->reason = "`on_chunk_header` callback error";  # 设置状态机的错误原因为 "`on_chunk_header` callback error"
    state->error_pos = (const char*) p;  # 设置状态机的错误位置为 p
    state->_current = (void*) (intptr_t) s_error;  # 将状态机的当前状态设置为错误状态
    return s_error;  # 返回错误状态
    /* UNREACHABLE */;  # 不可达代码
    abort();  # 终止程序执行
  }
  s_n_llhttp__internal__n_invoke_llhttp__on_chunk_header: {  # 进入状态机的处理块
    switch (llhttp__on_chunk_header(state, p, endp)) {
      case 0:
        // 如果返回值为0，跳转到 s_n_llhttp__internal__n_invoke_is_equal_content_length 标签处
        goto s_n_llhttp__internal__n_invoke_is_equal_content_length;
      case 21:
        // 如果返回值为21，跳转到 s_n_llhttp__internal__n_pause_4 标签处
        goto s_n_llhttp__internal__n_pause_4;
      default:
        // 其他情况跳转到 s_n_llhttp__internal__n_error_4 标签处
        goto s_n_llhttp__internal__n_error_4;
    }
    /* UNREACHABLE */;
    // 不可达代码
    abort();
  }
  s_n_llhttp__internal__n_error_6: {
    state->error = 0xc;
    state->reason = "Invalid character in chunk size";
    state->error_pos = (const char*) p;
    state->_current = (void*) (intptr_t) s_error;
    return s_error;
    /* UNREACHABLE */;
    // 不可达代码
    abort();
  }
  s_n_llhttp__internal__n_invoke_mul_add_content_length: {
    switch (llhttp__internal__c_mul_add_content_length(state, p, endp, match)) {
      case 1:
        // 如果返回值为1，跳转到 s_n_llhttp__internal__n_error_7 标签处
        goto s_n_llhttp__internal__n_error_7;
      default:
        // 其他情况跳转到 s_n_llhttp__internal__n_chunk_size 标签处
        goto s_n_llhttp__internal__n_chunk_size;
    }
    /* UNREACHABLE */;
    // 不可达代码
    abort();
  }
  s_n_llhttp__internal__n_error_8: {
    state->error = 0xc;
    state->reason = "Invalid character in chunk size";
    state->error_pos = (const char*) p;
    state->_current = (void*) (intptr_t) s_error;
    return s_error;
    /* UNREACHABLE */;
    // 不可达代码
    abort();
  }
  s_n_llhttp__internal__n_span_end_llhttp__on_body_1: {
    const unsigned char* start;
    int err;

    start = state->_span_pos0;
    state->_span_pos0 = NULL;
    err = llhttp__on_body(state, start, p);
    if (err != 0) {
      state->error = err;
      state->error_pos = (const char*) p;
      state->_current = (void*) (intptr_t) s_n_llhttp__internal__n_invoke_llhttp__on_message_complete_2;
      return s_error;
    }
    // 跳转到 s_n_llhttp__internal__n_invoke_llhttp__on_message_complete_2 标签处
    goto s_n_llhttp__internal__n_invoke_llhttp__on_message_complete_2;
    /* UNREACHABLE */;
    // 不可达代码
    abort();
  }
  s_n_llhttp__internal__n_invoke_update_finish_3: {
    switch (llhttp__internal__c_update_finish_3(state, p, endp)) {
      default:
        // 默认情况跳转到 s_n_llhttp__internal__n_span_start_llhttp__on_body_2 标签处
        goto s_n_llhttp__internal__n_span_start_llhttp__on_body_2;
    }
    /* UNREACHABLE */;
    // 不可达代码
    abort();
  }
  s_n_llhttp__internal__n_error_10: {
    state->error = 0xf;
    state->reason = "Request has invalid `Transfer-Encoding`";
    // 设置状态的原因为“请求具有无效的`Transfer-Encoding`”
    state->error_pos = (const char*) p;
    // 设置状态的错误位置为指针p所指向的位置
    state->_current = (void*) (intptr_t) s_error;
    // 设置状态的当前值为s_error
    return s_error;
    // 返回s_error状态
    /* UNREACHABLE */;
    // 不可到达的代码，表示此处的代码不会被执行
    abort();
    // 终止程序的执行
  }
  s_n_llhttp__internal__n_pause: {
    state->error = 0x15;
    // 设置状态的错误码为0x15
    state->reason = "on_message_complete pause";
    // 设置状态的原因为“on_message_complete暂停”
    state->error_pos = (const char*) p;
    // 设置状态的错误位置为指针p所指向的位置
    state->_current = (void*) (intptr_t) s_n_llhttp__internal__n_invoke_llhttp__after_message_complete;
    // 设置状态的当前值为s_n_llhttp__internal__n_invoke_llhttp__after_message_complete
    return s_error;
    // 返回s_error状态
    /* UNREACHABLE */;
    // 不可到达的代码，表示此处的代码不会被执行
    abort();
    // 终止程序的执行
  }
  s_n_llhttp__internal__n_error_2: {
    state->error = 0x12;
    // 设置状态的错误码为0x12
    state->reason = "`on_message_complete` callback error";
    // 设置状态的原因为“`on_message_complete`回调错误”
    state->error_pos = (const char*) p;
    // 设置状态的错误位置为指针p所指向的位置
    state->_current = (void*) (intptr_t) s_error;
    // 设置状态的当前值为s_error
    return s_error;
    // 返回s_error状态
    /* UNREACHABLE */;
    // 不可到达的代码，表示此处的代码不会被执行
    abort();
    // 终止程序的执行
  }
  s_n_llhttp__internal__n_invoke_llhttp__on_message_complete: {
    switch (llhttp__on_message_complete(state, p, endp)) {
      case 0:
        goto s_n_llhttp__internal__n_invoke_llhttp__after_message_complete;
      case 21:
        goto s_n_llhttp__internal__n_pause;
      default:
        goto s_n_llhttp__internal__n_error_2;
    }
    /* UNREACHABLE */;
    // 不可到达的代码，表示此处的代码不会被执行
    abort();
    // 终止程序的执行
  }
  s_n_llhttp__internal__n_invoke_or_flags_1: {
    switch (llhttp__internal__c_or_flags_1(state, p, endp)) {
      default:
        goto s_n_llhttp__internal__n_invoke_llhttp__after_headers_complete;
    }
    /* UNREACHABLE */;
    // 不可到达的代码，表示此处的代码不会被执行
    abort();
    // 终止程序的执行
  }
  s_n_llhttp__internal__n_invoke_or_flags_2: {
    switch (llhttp__internal__c_or_flags_1(state, p, endp)) {
      default:
        goto s_n_llhttp__internal__n_invoke_llhttp__after_headers_complete;
    }
    /* UNREACHABLE */;
    // 不可到达的代码，表示此处的代码不会被执行
    abort();
    // 终止程序的执行
  }
  s_n_llhttp__internal__n_invoke_update_upgrade: {
    switch (llhttp__internal__c_update_upgrade(state, p, endp)) {
      default:
        goto s_n_llhttp__internal__n_invoke_or_flags_2;
    }
    /* UNREACHABLE */;
    // 不可到达的代码，表示此处的代码不会被执行
    abort();
    // 终止程序的执行
  }
  s_n_llhttp__internal__n_pause_6: {
    state->error = 0x15;
    // 设置状态的错误码为0x15
    state->reason = "Paused by on_headers_complete";  # 设置状态的原因为“被 on_headers_complete 暂停”
    state->error_pos = (const char*) p;  # 设置错误位置为 p 的地址
    state->_current = (void*) (intptr_t) s_n_llhttp__internal__n_invoke_llhttp__after_headers_complete;  # 设置当前状态为调用 on_headers_complete 后的状态
    return s_error;  # 返回错误状态
    /* UNREACHABLE */;  # 不可达代码
    abort();  # 终止程序
  }
  s_n_llhttp__internal__n_error_1: {  # 错误状态 1
    state->error = 0x11;  # 设置错误码为 0x11
    state->reason = "User callback error";  # 设置状态的原因为“用户回调错误”
    state->error_pos = (const char*) p;  # 设置错误位置为 p 的地址
    state->_current = (void*) (intptr_t) s_error;  # 设置当前状态为错误状态
    return s_error;  # 返回错误状态
    /* UNREACHABLE */;  # 不可达代码
    abort();  # 终止程序
  }
  s_n_llhttp__internal__n_invoke_llhttp__on_headers_complete: {  # 调用 on_headers_complete 状态
    switch (llhttp__on_headers_complete(state, p, endp)) {  # 调用 on_headers_complete 函数
      case 0:  # 如果返回值为 0
        goto s_n_llhttp__internal__n_invoke_llhttp__after_headers_complete;  # 跳转到调用 on_headers_complete 后的状态
      case 1:  # 如果返回值为 1
        goto s_n_llhttp__internal__n_invoke_or_flags_1;  # 跳转到调用 or_flags_1 状态
      case 2:  # 如果返回值为 2
        goto s_n_llhttp__internal__n_invoke_update_upgrade;  # 跳转到调用 update_upgrade 状态
      case 21:  # 如果返回值为 21
        goto s_n_llhttp__internal__n_pause_6;  # 跳转到暂停状态 6
      default:  # 其他情况
        goto s_n_llhttp__internal__n_error_1;  # 跳转到错误状态 1
    }
    /* UNREACHABLE */;  # 不可达代码
    abort();  # 终止程序
  }
  s_n_llhttp__internal__n_invoke_llhttp__before_headers_complete: {  # 调用 before_headers_complete 状态
    switch (llhttp__before_headers_complete(state, p, endp)) {  # 调用 before_headers_complete 函数
      default:  # 默认情况
        goto s_n_llhttp__internal__n_invoke_llhttp__on_headers_complete;  # 跳转到调用 on_headers_complete 状态
    }
    /* UNREACHABLE */;  # 不可达代码
    abort();  # 终止程序
  }
  s_n_llhttp__internal__n_invoke_test_lenient_flags_1: {  # 调用 test_lenient_flags_1 状态
    switch (llhttp__internal__c_test_lenient_flags_1(state, p, endp)) {  # 调用 test_lenient_flags_1 函数
      case 0:  # 如果返回值为 0
        goto s_n_llhttp__internal__n_error_11;  # 跳转到错误状态 11
      default:  # 其他情况
        goto s_n_llhttp__internal__n_invoke_llhttp__before_headers_complete;  # 跳转到调用 before_headers_complete 状态
    }
    /* UNREACHABLE */;  # 不可达代码
    abort();  # 终止程序
  }
  s_n_llhttp__internal__n_invoke_test_flags_1: {  # 调用 test_flags_1 状态
    switch (llhttp__internal__c_test_flags_1(state, p, endp)) {  # 调用 test_flags_1 函数
      case 1:  # 如果返回值为 1
        goto s_n_llhttp__internal__n_invoke_test_lenient_flags_1;  # 跳转到调用 test_lenient_flags_1 状态
      default:  # 其他情况
        goto s_n_llhttp__internal__n_invoke_llhttp__before_headers_complete;  # 跳转到调用 before_headers_complete 状态
    }
    /* UNREACHABLE */;  # 不可达代码
    abort();  # 终止程序
    // 中止当前操作
    abort();
  }
  // 调用内部测试标志函数
  s_n_llhttp__internal__n_invoke_test_flags: {
    switch (llhttp__internal__c_test_flags(state, p, endp)) {
      case 1:
        // 转到处理 chunk 完成的状态
        goto s_n_llhttp__internal__n_invoke_llhttp__on_chunk_complete_1;
      default:
        // 转到处理测试标志的状态
        goto s_n_llhttp__internal__n_invoke_test_flags_1;
    }
    /* 不可达代码 */;
    // 中止当前操作
    abort();
  }
  // 错误状态，设置错误信息并返回错误状态
  s_n_llhttp__internal__n_error_13: {
    state->error = 0xb;
    state->reason = "Empty Content-Length";
    state->error_pos = (const char*) p;
    state->_current = (void*) (intptr_t) s_error;
    return s_error;
    /* 不可达代码 */;
    // 中止当前操作
    abort();
  }
  // 调用处理 header value 的函数
  s_n_llhttp__internal__n_span_end_llhttp__on_header_value: {
    const unsigned char* start;
    int err;

    start = state->_span_pos0;
    state->_span_pos0 = NULL;
    err = llhttp__on_header_value(state, start, p);
    if (err != 0) {
      state->error = err;
      state->error_pos = (const char*) p;
      state->_current = (void*) (intptr_t) s_n_llhttp__internal__n_invoke_llhttp__on_header_value_complete;
      return s_error;
    }
    // 转到处理 header value 完成的状态
    goto s_n_llhttp__internal__n_invoke_llhttp__on_header_value_complete;
    /* 不可达代码 */;
    // 中止当前操作
    abort();
  }
  // 调用更新 header 状态的函数
  s_n_llhttp__internal__n_invoke_update_header_state: {
    switch (llhttp__internal__c_update_header_state(state, p, endp)) {
      default:
        // 转到处理 header value 开始的状态
        goto s_n_llhttp__internal__n_span_start_llhttp__on_header_value;
    }
    /* 不可达代码 */;
    // 中止当前操作
    abort();
  }
  // 调用或运算标志 3 的函数
  s_n_llhttp__internal__n_invoke_or_flags_3: {
    switch (llhttp__internal__c_or_flags_3(state, p, endp)) {
      default:
        // 转到更新 header 状态的状态
        goto s_n_llhttp__internal__n_invoke_update_header_state;
    }
    /* 不可达代码 */;
    // 中止当前操作
    abort();
  }
  // 调用或运算标志 4 的函数
  s_n_llhttp__internal__n_invoke_or_flags_4: {
    switch (llhttp__internal__c_or_flags_4(state, p, endp)) {
      default:
        // 转到更新 header 状态的状态
        goto s_n_llhttp__internal__n_invoke_update_header_state;
    }
    /* 不可达代码 */;
    // 中止当前操作
    abort();
  }
  // 调用或运算标志 5 的函数
  s_n_llhttp__internal__n_invoke_or_flags_5: {
  switch (llhttp__internal__c_or_flags_5(state, p, endp)) {
    // 根据状态和输入指针调用内部函数，执行相应的操作
    default:
      // 转到指定状态
      goto s_n_llhttp__internal__n_invoke_update_header_state;
  }
  /* UNREACHABLE */;
  // 终止程序
  abort();
}
s_n_llhttp__internal__n_invoke_or_flags_6: {
  switch (llhttp__internal__c_or_flags_6(state, p, endp)) {
    // 根据状态和输入指针调用内部函数，执行相应的操作
    default:
      // 转到指定状态
      goto s_n_llhttp__internal__n_span_start_llhttp__on_header_value;
  }
  /* UNREACHABLE */;
  // 终止程序
  abort();
}
s_n_llhttp__internal__n_invoke_load_header_state_1: {
  switch (llhttp__internal__c_load_header_state(state, p, endp)) {
    // 根据状态和输入指针调用内部函数，执行相应的操作
    case 5:
      // 转到指定状态
      goto s_n_llhttp__internal__n_invoke_or_flags_3;
    case 6:
      // 转到指定状态
      goto s_n_llhttp__internal__n_invoke_or_flags_4;
    case 7:
      // 转到指定状态
      goto s_n_llhttp__internal__n_invoke_or_flags_5;
    case 8:
      // 转到指定状态
      goto s_n_llhttp__internal__n_invoke_or_flags_6;
    default:
      // 转到指定状态
      goto s_n_llhttp__internal__n_span_start_llhttp__on_header_value;
  }
  /* UNREACHABLE */;
  // 终止程序
  abort();
}
s_n_llhttp__internal__n_invoke_load_header_state: {
  switch (llhttp__internal__c_load_header_state(state, p, endp)) {
    // 根据状态和输入指针调用内部函数，执行相应的操作
    case 2:
      // 转到指定状态
      goto s_n_llhttp__internal__n_error_13;
    default:
      // 转到指定状态
      goto s_n_llhttp__internal__n_invoke_load_header_state_1;
  }
  /* UNREACHABLE */;
  // 终止程序
  abort();
}
s_n_llhttp__internal__n_invoke_update_header_state_1: {
  switch (llhttp__internal__c_update_header_state(state, p, endp)) {
    // 根据状态和输入指针调用内部函数，执行相应的操作
    default:
      // 转到指定状态
      goto s_n_llhttp__internal__n_invoke_llhttp__on_header_value_complete;
  }
  /* UNREACHABLE */;
  // 终止程序
  abort();
}
s_n_llhttp__internal__n_invoke_or_flags_7: {
  switch (llhttp__internal__c_or_flags_3(state, p, endp)) {
    // 根据状态和输入指针调用内部函数，执行相应的操作
    default:
      // 转到指定状态
      goto s_n_llhttp__internal__n_invoke_update_header_state_1;
  }
  /* UNREACHABLE */;
  // 终止程序
  abort();
}
s_n_llhttp__internal__n_invoke_or_flags_8: {
  switch (llhttp__internal__c_or_flags_4(state, p, endp)) {
    // 根据状态和输入指针调用内部函数，执行相应的操作
    default:
      // 转到指定状态
      goto s_n_llhttp__internal__n_invoke_update_header_state_1;
  }
  /* UNREACHABLE */;
  // 标记代码不可达
  abort();
  // 终止程序执行
}
s_n_llhttp__internal__n_invoke_or_flags_9: {
  // 转到状态机的内部函数处理
  switch (llhttp__internal__c_or_flags_5(state, p, endp)) {
    default:
      // 默认情况下转到更新头部状态的内部函数处理
      goto s_n_llhttp__internal__n_invoke_update_header_state_1;
  }
  /* UNREACHABLE */;
  // 标记代码不可达
  abort();
}
// 以下类似，每个状态都是状态机的内部处理函数
    # 获取状态中的_span_pos0的值，赋给start变量
    start = state->_span_pos0;
    # 将状态中的_span_pos0置为NULL
    state->_span_pos0 = NULL;
    # 调用llhttp__on_header_value函数，传入状态、起始位置start和当前位置p，将返回值赋给err
    err = llhttp__on_header_value(state, start, p);
    # 如果err不等于0，将err赋给状态中的error，将(p+1)转换为const char*赋给状态中的error_pos，将s_n_llhttp__internal__n_header_value_almost_done转换为void*后赋给状态中的_current，返回s_error
    if (err != 0) {
      state->error = err;
      state->error_pos = (const char*) (p + 1);
      state->_current = (void*) (intptr_t) s_n_llhttp__internal__n_header_value_almost_done;
      return s_error;
    }
    # 将p自增1
    p++;
    # 跳转到s_n_llhttp__internal__n_header_value_almost_done标签处
    goto s_n_llhttp__internal__n_header_value_almost_done;
    # 不可达代码
    /* UNREACHABLE */;
    # 中止程序
    abort();
  }
  # 标签s_n_llhttp__internal__n_span_end_llhttp__on_header_value_3
  s_n_llhttp__internal__n_span_end_llhttp__on_header_value_3: {
    # 声明变量start和err
    const unsigned char* start;
    int err;
    # 获取状态中的_span_pos0的值，赋给start变量
    start = state->_span_pos0;
    # 将状态中的_span_pos0置为NULL
    state->_span_pos0 = NULL;
    # 调用llhttp__on_header_value函数，传入状态、起始位置start和当前位置p，将返回值赋给err
    err = llhttp__on_header_value(state, start, p);
    # 如果err不等于0，将err赋给状态中的error，将(p+1)转换为const char*赋给状态中的error_pos，将s_n_llhttp__internal__n_header_value_almost_done转换为void*后赋给状态中的_current，返回s_error
    if (err != 0) {
      state->error = err;
      state->error_pos = (const char*) (p + 1);
      state->_current = (void*) (intptr_t) s_n_llhttp__internal__n_header_value_almost_done;
      return s_error;
    }
    # 将p自增1
    p++;
    # 跳转到s_n_llhttp__internal__n_header_value_almost_done标签处
    goto s_n_llhttp__internal__n_header_value_almost_done;
    # 不可达代码
    /* UNREACHABLE */;
    # 中止程序
    abort();
  }
  # 标签s_n_llhttp__internal__n_error_15
  s_n_llhttp__internal__n_error_15: {
    # 将0xa赋给状态中的error，将"Invalid header value char"赋给状态中的reason，将p转换为const char*赋给状态中的error_pos，将s_error转换为void*后赋给状态中的_current，返回s_error
    state->error = 0xa;
    state->reason = "Invalid header value char";
    state->error_pos = (const char*) p;
    state->_current = (void*) (intptr_t) s_error;
    return s_error;
    # 不可达代码
    /* UNREACHABLE */;
    # 中止程序
    abort();
  }
  # 标签s_n_llhttp__internal__n_invoke_test_lenient_flags_2
  s_n_llhttp__internal__n_invoke_test_lenient_flags_2: {
    # 调用llhttp__internal__c_test_lenient_flags_2函数，传入状态、当前位置p和结束位置endp，根据返回值进行跳转
    switch (llhttp__internal__c_test_lenient_flags_2(state, p, endp)) {
      # 如果返回值为1，跳转到s_n_llhttp__internal__n_header_value_lenient标签处
      case 1:
        goto s_n_llhttp__internal__n_header_value_lenient;
      # 默认情况，跳转到s_n_llhttp__internal__n_error_15标签处
      default:
        goto s_n_llhttp__internal__n_error_15;
    }
    # 不可达代码
    /* UNREACHABLE */;
    # 中止程序
    abort();
  }
  # 标签s_n_llhttp__internal__n_invoke_update_header_state_3
  s_n_llhttp__internal__n_invoke_update_header_state_3: {
    # 调用llhttp__internal__c_update_header_state函数，传入状态、当前位置p和结束位置endp，根据返回值进行跳转
    switch (llhttp__internal__c_update_header_state(state, p, endp)) {
      # 默认情况，跳转到s_n_llhttp__internal__n_header_value_connection标签处
      default:
        goto s_n_llhttp__internal__n_header_value_connection;
    }
    # 不可达代码
    /* UNREACHABLE */;
    # 中止程序
    abort();
  }
  # 标签s_n_llhttp__internal__n_invoke_or_flags_11
  s_n_llhttp__internal__n_invoke_or_flags_11: {
    # 调用llhttp__internal__c_or_flags_3函数，传入状态、当前位置p和结束位置endp，根据返回值进行跳转
    switch (llhttp__internal__c_or_flags_3(state, p, endp)) {
      # 默认情况，跳转到s_n_llhttp__internal__n_invoke_update_header_state_3标签处
      default:
        goto s_n_llhttp__internal__n_invoke_update_header_state_3;
  }
  /* UNREACHABLE */;
  // 如果程序执行到这里，表示代码存在错误，因为这里的代码是不可达的
  abort();
}
s_n_llhttp__internal__n_invoke_or_flags_12: {
  switch (llhttp__internal__c_or_flags_4(state, p, endp)) {
    default:
      goto s_n_llhttp__internal__n_invoke_update_header_state_3;
  }
  /* UNREACHABLE */;
  // 如果程序执行到这里，表示代码存在错误，因为这里的代码是不可达的
  abort();
}
s_n_llhttp__internal__n_invoke_or_flags_13: {
  switch (llhttp__internal__c_or_flags_5(state, p, endp)) {
    default:
      goto s_n_llhttp__internal__n_invoke_update_header_state_3;
  }
  /* UNREACHABLE */;
  // 如果程序执行到这里，表示代码存在错误，因为这里的代码是不可达的
  abort();
}
s_n_llhttp__internal__n_invoke_or_flags_14: {
  switch (llhttp__internal__c_or_flags_6(state, p, endp)) {
    default:
      goto s_n_llhttp__internal__n_header_value_connection;
  }
  /* UNREACHABLE */;
  // 如果程序执行到这里，表示代码存在错误，因为这里的代码是不可达的
  abort();
}
s_n_llhttp__internal__n_invoke_load_header_state_4: {
  switch (llhttp__internal__c_load_header_state(state, p, endp)) {
    case 5:
      goto s_n_llhttp__internal__n_invoke_or_flags_11;
    case 6:
      goto s_n_llhttp__internal__n_invoke_or_flags_12;
    case 7:
      goto s_n_llhttp__internal__n_invoke_or_flags_13;
    case 8:
      goto s_n_llhttp__internal__n_invoke_or_flags_14;
    default:
      goto s_n_llhttp__internal__n_header_value_connection;
  }
  /* UNREACHABLE */;
  // 如果程序执行到这里，表示代码存在错误，因为这里的代码是不可达的
  abort();
}
s_n_llhttp__internal__n_invoke_update_header_state_4: {
  switch (llhttp__internal__c_update_header_state_4(state, p, endp)) {
    default:
      goto s_n_llhttp__internal__n_header_value_connection_token;
  }
  /* UNREACHABLE */;
  // 如果程序执行到这里，表示代码存在错误，因为这里的代码是不可达的
  abort();
}
s_n_llhttp__internal__n_invoke_update_header_state_2: {
  switch (llhttp__internal__c_update_header_state_2(state, p, endp)) {
    default:
      goto s_n_llhttp__internal__n_header_value_connection_ws;
  }
  /* UNREACHABLE */;
  // 如果程序执行到这里，表示代码存在错误，因为这里的代码是不可达的
  abort();
}
s_n_llhttp__internal__n_invoke_update_header_state_5: {
  // 这里缺少代码，需要根据具体情况进行补充
}
  switch (llhttp__internal__c_update_header_state_5(state, p, endp)) {
    default:
      // 转到状态 s_n_llhttp__internal__n_header_value_connection_ws
      goto s_n_llhttp__internal__n_header_value_connection_ws;
  }
  /* UNREACHABLE */;
  // 终止程序
  abort();
}

s_n_llhttp__internal__n_invoke_update_header_state_6: {
  switch (llhttp__internal__c_update_header_state_6(state, p, endp)) {
    default:
      // 转到状态 s_n_llhttp__internal__n_header_value_connection_ws
      goto s_n_llhttp__internal__n_header_value_connection_ws;
  }
  /* UNREACHABLE */;
  // 终止程序
  abort();
}

s_n_llhttp__internal__n_span_end_llhttp__on_header_value_4: {
  const unsigned char* start;
  int err;

  start = state->_span_pos0;
  state->_span_pos0 = NULL;
  err = llhttp__on_header_value(state, start, p);
  if (err != 0) {
    state->error = err;
    state->error_pos = (const char*) p;
    state->_current = (void*) (intptr_t) s_n_llhttp__internal__n_error_17;
    return s_error;
  }
  // 转到状态 s_n_llhttp__internal__n_error_17
  goto s_n_llhttp__internal__n_error_17;
  /* UNREACHABLE */;
  // 终止程序
  abort();
}

s_n_llhttp__internal__n_invoke_mul_add_content_length_1: {
  switch (llhttp__internal__c_mul_add_content_length_1(state, p, endp, match)) {
    case 1:
      // 转到状态 s_n_llhttp__internal__n_span_end_llhttp__on_header_value_4
      goto s_n_llhttp__internal__n_span_end_llhttp__on_header_value_4;
    default:
      // 转到状态 s_n_llhttp__internal__n_header_value_content_length
      goto s_n_llhttp__internal__n_header_value_content_length;
  }
  /* UNREACHABLE */;
  // 终止程序
  abort();
}

s_n_llhttp__internal__n_invoke_or_flags_15: {
  switch (llhttp__internal__c_or_flags_15(state, p, endp)) {
    default:
      // 转到状态 s_n_llhttp__internal__n_header_value_otherwise
      goto s_n_llhttp__internal__n_header_value_otherwise;
  }
  /* UNREACHABLE */;
  // 终止程序
  abort();
}

s_n_llhttp__internal__n_span_end_llhttp__on_header_value_5: {
  const unsigned char* start;
  int err;

  start = state->_span_pos0;
  state->_span_pos0 = NULL;
  err = llhttp__on_header_value(state, start, p);
  if (err != 0) {
    state->error = err;
    state->error_pos = (const char*) p;
    state->_current = (void*) (intptr_t) s_n_llhttp__internal__n_error_18;
    return s_error;
  }
  // 跳转到指定标签
  goto s_n_llhttp__internal__n_error_18;
  // 不可达代码
  /* UNREACHABLE */;
  // 中止程序
  abort();
  // 标签 s_n_llhttp__internal__n_error_16
  s_n_llhttp__internal__n_error_16: {
    // 设置状态错误码为 0x4
    state->error = 0x4;
    // 设置错误原因为 "Duplicate Content-Length"
    state->reason = "Duplicate Content-Length";
    // 设置错误位置为当前位置
    state->error_pos = (const char*) p;
    // 设置当前状态为错误状态
    state->_current = (void*) (intptr_t) s_error;
    // 返回错误状态
    return s_error;
    // 不可达代码
    /* UNREACHABLE */;
    // 中止程序
    abort();
  }
  // 标签 s_n_llhttp__internal__n_invoke_test_flags_2
  s_n_llhttp__internal__n_invoke_test_flags_2: {
    // 根据测试结果跳转到不同的标签
    switch (llhttp__internal__c_test_flags_2(state, p, endp)) {
      case 0:
        // 测试结果为 0，跳转到指定标签
        goto s_n_llhttp__internal__n_header_value_content_length;
      default:
        // 默认情况下跳转到指定标签
        goto s_n_llhttp__internal__n_error_16;
    }
    // 不可达代码
    /* UNREACHABLE */;
    // 中止程序
    abort();
  }
  // 标签 s_n_llhttp__internal__n_invoke_update_header_state_7
  s_n_llhttp__internal__n_invoke_update_header_state_7: {
    // 根据更新结果跳转到不同的标签
    switch (llhttp__internal__c_update_header_state_7(state, p, endp)) {
      default:
        // 默认情况下跳转到指定标签
        goto s_n_llhttp__internal__n_header_value_otherwise;
    }
    // 不可达代码
    /* UNREACHABLE */;
    // 中止程序
    abort();
  }
  // 标签 s_n_llhttp__internal__n_invoke_update_header_state_8
  s_n_llhttp__internal__n_invoke_update_header_state_8: {
    // 根据更新结果跳转到不同的标签
    switch (llhttp__internal__c_update_header_state_4(state, p, endp)) {
      default:
        // 默认情况下跳转到指定标签
        goto s_n_llhttp__internal__n_header_value;
    }
    // 不可达代码
    /* UNREACHABLE */;
    // 中止程序
    abort();
  }
  // 标签 s_n_llhttp__internal__n_invoke_and_flags
  s_n_llhttp__internal__n_invoke_and_flags: {
    // 根据与操作的结果跳转到不同的标签
    switch (llhttp__internal__c_and_flags(state, p, endp)) {
      default:
        // 默认情况下跳转到指定标签
        goto s_n_llhttp__internal__n_header_value_te_chunked;
    }
    // 不可达代码
    /* UNREACHABLE */;
    // 中止程序
    abort();
  }
  // 标签 s_n_llhttp__internal__n_invoke_or_flags_16
  s_n_llhttp__internal__n_invoke_or_flags_16: {
    // 根据或操作的结果跳转到不同的标签
    switch (llhttp__internal__c_or_flags_16(state, p, endp)) {
      default:
        // 默认情况下跳转到指定标签
        goto s_n_llhttp__internal__n_invoke_and_flags;
    }
    // 不可达代码
    /* UNREACHABLE */;
    // 中止程序
    abort();
  }
  // 标签 s_n_llhttp__internal__n_invoke_or_flags_17
  s_n_llhttp__internal__n_invoke_or_flags_17: {
    // 根据或操作的结果跳转到不同的标签
    switch (llhttp__internal__c_or_flags_17(state, p, endp)) {
      default:
        // 默认情况下跳转到指定标签
        goto s_n_llhttp__internal__n_invoke_update_header_state_8;
    }
    // 不可达代码
    /* UNREACHABLE */;
    // 中止程序
    abort();
  }
  // 标签 s_n_llhttp__internal__n_invoke_load_header_state_2
  s_n_llhttp__internal__n_invoke_load_header_state_2: {
    switch (llhttp__internal__c_load_header_state(state, p, endp)) {
      // 根据当前状态和输入的数据，加载下一个状态
      case 1:
        // 转到处理 header value 的状态
        goto s_n_llhttp__internal__n_header_value_connection;
      case 2:
        // 转到调用测试标志 2 的状态
        goto s_n_llhttp__internal__n_invoke_test_flags_2;
      case 3:
        // 转到调用或标志 16 的状态
        goto s_n_llhttp__internal__n_invoke_or_flags_16;
      case 4:
        // 转到调用或标志 17 的状态
        goto s_n_llhttp__internal__n_invoke_or_flags_17;
      default:
        // 转到处理 header value 的状态
        goto s_n_llhttp__internal__n_header_value;
    }
    /* UNREACHABLE */;
    // 如果执行到这里，表示代码逻辑有误，中止程序
    abort();
  }
  s_n_llhttp__internal__n_span_end_llhttp__on_header_field: {
    const unsigned char* start;
    int err;

    start = state->_span_pos0;
    state->_span_pos0 = NULL;
    err = llhttp__on_header_field(state, start, p);
    if (err != 0) {
      state->error = err;
      state->error_pos = (const char*) (p + 1);
      state->_current = (void*) (intptr_t) s_n_llhttp__internal__n_invoke_llhttp__on_header_field_complete;
      return s_error;
    }
    p++;
    goto s_n_llhttp__internal__n_invoke_llhttp__on_header_field_complete;
    /* UNREACHABLE */;
    abort();
  }
  s_n_llhttp__internal__n_span_end_llhttp__on_header_field_1: {
    const unsigned char* start;
    int err;

    start = state->_span_pos0;
    state->_span_pos0 = NULL;
    err = llhttp__on_header_field(state, start, p);
    if (err != 0) {
      state->error = err;
      state->error_pos = (const char*) (p + 1);
      state->_current = (void*) (intptr_t) s_n_llhttp__internal__n_invoke_llhttp__on_header_field_complete;
      return s_error;
    }
    p++;
    goto s_n_llhttp__internal__n_invoke_llhttp__on_header_field_complete;
    /* UNREACHABLE */;
    abort();
  }
  s_n_llhttp__internal__n_error_19: {
    state->error = 0xa;
    state->reason = "Invalid header token";
    state->error_pos = (const char*) p;
    state->_current = (void*) (intptr_t) s_error;
    return s_error;
    /* UNREACHABLE */;
    abort();
  }
  s_n_llhttp__internal__n_invoke_update_header_state_9: {
  switch (llhttp__internal__c_update_header_state_4(state, p, endp)) {
    default:
      // 转到通用的 header field 状态
      goto s_n_llhttp__internal__n_header_field_general;
  }
  /* UNREACHABLE */;
  // 终止程序
  abort();
}
s_n_llhttp__internal__n_invoke_store_header_state: {
  switch (llhttp__internal__c_store_header_state(state, p, endp, match)) {
    default:
      // 转到 header field colon 状态
      goto s_n_llhttp__internal__n_header_field_colon;
  }
  /* UNREACHABLE */;
  // 终止程序
  abort();
}
s_n_llhttp__internal__n_invoke_update_header_state_10: {
  switch (llhttp__internal__c_update_header_state_4(state, p, endp)) {
    default:
      // 转到通用的 header field 状态
      goto s_n_llhttp__internal__n_header_field_general;
  }
  /* UNREACHABLE */;
  // 终止程序
  abort();
}
s_n_llhttp__internal__n_invoke_llhttp__on_url_complete: {
  switch (llhttp__on_url_complete(state, p, endp)) {
    default:
      // 转到 header field start 状态
      goto s_n_llhttp__internal__n_header_field_start;
  }
  /* UNREACHABLE */;
  // 终止程序
  abort();
}
s_n_llhttp__internal__n_invoke_update_http_minor: {
  switch (llhttp__internal__c_update_http_minor(state, p, endp)) {
    default:
      // 转到 invoke llhttp__on_url_complete 状态
      goto s_n_llhttp__internal__n_invoke_llhttp__on_url_complete;
  }
  /* UNREACHABLE */;
  // 终止程序
  abort();
}
s_n_llhttp__internal__n_invoke_update_http_major: {
  switch (llhttp__internal__c_update_http_major(state, p, endp)) {
    default:
      // 转到 invoke update http minor 状态
      goto s_n_llhttp__internal__n_invoke_update_http_minor;
  }
  /* UNREACHABLE */;
  // 终止程序
  abort();
}
s_n_llhttp__internal__n_span_end_llhttp__on_url_3: {
  const unsigned char* start;
  int err;

  start = state->_span_pos0;
  state->_span_pos0 = NULL;
  err = llhttp__on_url(state, start, p);
  if (err != 0) {
    state->error = err;
    state->error_pos = (const char*) p;
    state->_current = (void*) (intptr_t) s_n_llhttp__internal__n_url_skip_to_http09;
    return s_error;
  }
  // 转到 url skip to http09 状态
  goto s_n_llhttp__internal__n_url_skip_to_http09;
  /* UNREACHABLE */;
  // 终止程序
  abort();
}
s_n_llhttp__internal__n_error_20: {
  state->error = 0x7;
    # 设置状态的原因为"Expected CRLF"
    state->reason = "Expected CRLF";
    # 设置错误位置为当前指针位置
    state->error_pos = (const char*) p;
    # 设置当前状态为错误状态
    state->_current = (void*) (intptr_t) s_error;
    # 返回错误状态
    return s_error;
    /* UNREACHABLE */;
    # 中止程序
    abort();
  }
  s_n_llhttp__internal__n_span_end_llhttp__on_url_4: {
    # 定义起始位置指针
    const unsigned char* start;
    # 定义错误变量
    int err;

    # 赋值起始位置指针为状态的起始位置指针
    start = state->_span_pos0;
    state->_span_pos0 = NULL;
    # 调用 llhttp__on_url 函数处理 URL
    err = llhttp__on_url(state, start, p);
    # 如果出现错误
    if (err != 0) {
      # 设置状态的错误为 err
      state->error = err;
      # 设置错误位置为当前指针位置
      state->error_pos = (const char*) p;
      # 设置当前状态为跳转状态
      state->_current = (void*) (intptr_t) s_n_llhttp__internal__n_url_skip_lf_to_http09;
      # 返回错误状态
      return s_error;
    }
    # 跳转到指定状态
    goto s_n_llhttp__internal__n_url_skip_lf_to_http09;
    /* UNREACHABLE */;
    # 中止程序
    abort();
  }
  s_n_llhttp__internal__n_error_23: {
    # 设置状态的错误为 0x17
    state->error = 0x17;
    # 设置状态的原因为"Pause on PRI/Upgrade"
    state->reason = "Pause on PRI/Upgrade";
    # 设置错误位置为当前指针位置
    state->error_pos = (const char*) p;
    # 设置当前状态为错误状态
    state->_current = (void*) (intptr_t) s_error;
    # 返回错误状态
    return s_error;
    /* UNREACHABLE */;
    # 中止程序
    abort();
  }
  s_n_llhttp__internal__n_error_24: {
    # 设置状态的错误为 0x9
    state->error = 0x9;
    # 设置状态的原因为"Expected HTTP/2 Connection Preface"
    state->reason = "Expected HTTP/2 Connection Preface";
    # 设置错误位置为当前指针位置
    state->error_pos = (const char*) p;
    # 设置当前状态为错误状态
    state->_current = (void*) (intptr_t) s_error;
    # 返回错误状态
    return s_error;
    /* UNREACHABLE */;
    # 中止程序
    abort();
  }
  s_n_llhttp__internal__n_error_22: {
    # 设置状态的错误为 0x9
    state->error = 0x9;
    # 设置状态的原因为"Expected CRLF after version"
    state->reason = "Expected CRLF after version";
    # 设置错误位置为当前指针位置
    state->error_pos = (const char*) p;
    # 设置当前状态为错误状态
    state->_current = (void*) (intptr_t) s_error;
    # 返回错误状态
    return s_error;
    /* UNREACHABLE */;
    # 中止程序
    abort();
  }
  s_n_llhttp__internal__n_invoke_load_method_1: {
    # 调用 llhttp__internal__c_load_method 函数处理方法
    switch (llhttp__internal__c_load_method(state, p, endp)) {
      # 如果返回值为 34
      case 34:
        # 跳转到指定状态
        goto s_n_llhttp__internal__n_req_pri_upgrade;
      # 默认情况
      default:
        # 跳转到指定状态
        goto s_n_llhttp__internal__n_req_http_complete;
    }
    /* UNREACHABLE */;
    # 中止程序
    abort();
  }
  s_n_llhttp__internal__n_invoke_store_http_minor: {
    # 调用 llhttp__internal__c_store_http_minor 函数处理 HTTP 版本号
    switch (llhttp__internal__c_store_http_minor(state, p, endp, match)) {
      # 默认情况
      default:
        # 跳转到指定状态
        goto s_n_llhttp__internal__n_invoke_load_method_1;
    }
  /* UNREACHABLE */;
  // 标记代码为不可达，中断程序执行
  abort();
  // 终止程序执行
  }
  s_n_llhttp__internal__n_error_25: {
    state->error = 0x9;
    state->reason = "Invalid minor version";
    state->error_pos = (const char*) p;
    state->_current = (void*) (intptr_t) s_error;
    return s_error;
    /* UNREACHABLE */;
    // 标记代码为不可达，中断程序执行
    abort();
  }
  s_n_llhttp__internal__n_error_26: {
    state->error = 0x9;
    state->reason = "Expected dot";
    state->error_pos = (const char*) p;
    state->_current = (void*) (intptr_t) s_error;
    return s_error;
    /* UNREACHABLE */;
    // 标记代码为不可达，中断程序执行
    abort();
  }
  s_n_llhttp__internal__n_invoke_store_http_major: {
    switch (llhttp__internal__c_store_http_major(state, p, endp, match)) {
      default:
        goto s_n_llhttp__internal__n_req_http_dot;
    }
    /* UNREACHABLE */;
    // 标记代码为不可达，中断程序执行
    abort();
  }
  s_n_llhttp__internal__n_error_27: {
    state->error = 0x9;
    state->reason = "Invalid major version";
    state->error_pos = (const char*) p;
    state->_current = (void*) (intptr_t) s_error;
    return s_error;
    /* UNREACHABLE */;
    // 标记代码为不可达，中断程序执行
    abort();
  }
  s_n_llhttp__internal__n_error_21: {
    state->error = 0x8;
    state->reason = "Invalid method for HTTP/x.x request";
    state->error_pos = (const char*) p;
    state->_current = (void*) (intptr_t) s_error;
    return s_error;
    /* UNREACHABLE */;
    // 标记代码为不可达，中断程序执行
    abort();
  }
  s_n_llhttp__internal__n_invoke_load_method: {
    }
    /* UNREACHABLE */;
    // 标记代码为不可达，中断程序执行
    abort();
  }
  s_n_llhttp__internal__n_error_30: {
    state->error = 0x8;
    state->reason = "Expected HTTP/";
    state->error_pos = (const char*) p;
    state->_current = (void*) (intptr_t) s_error;
    return s_error;
    /* UNREACHABLE */;
    // 标记代码为不可达，中断程序执行
    abort();
  }
  s_n_llhttp__internal__n_error_28: {
    state->error = 0x8;
    state->reason = "Expected SOURCE method for ICE/x.x request";
    state->error_pos = (const char*) p;
    state->_current = (void*) (intptr_t) s_error;
    return s_error;
    /* UNREACHABLE */;
    // 标记代码为不可达，中断程序执行
    abort();
  }
  s_n_llhttp__internal__n_invoke_load_method_2: {
    switch (llhttp__internal__c_load_method(state, p, endp)) {
      case 33:
        // 如果加载方法返回值为33，则跳转到状态 s_n_llhttp__internal__n_req_http_major
        goto s_n_llhttp__internal__n_req_http_major;
      default:
        // 如果加载方法返回值为其他值，则跳转到状态 s_n_llhttp__internal__n_error_28
        goto s_n_llhttp__internal__n_error_28;
    }
    /* UNREACHABLE */;
    // 不可达代码，中止程序
    abort();
  }
  s_n_llhttp__internal__n_error_29: {
    // 设置状态的错误码为0x8
    state->error = 0x8;
    // 设置状态的错误原因为"Invalid method for RTSP/x.x request"
    state->reason = "Invalid method for RTSP/x.x request";
    // 设置状态的错误位置为当前位置
    state->error_pos = (const char*) p;
    // 设置状态的当前状态为错误状态
    state->_current = (void*) (intptr_t) s_error;
    // 返回错误状态
    return s_error;
    /* UNREACHABLE */;
    // 不可达代码，中止程序
    abort();
  }
  s_n_llhttp__internal__n_invoke_load_method_3: {
    switch (llhttp__internal__c_load_method(state, p, endp)) {
      case 1:
        // 如果加载方法返回值为1，则跳转到状态 s_n_llhttp__internal__n_req_http_major
        goto s_n_llhttp__internal__n_req_http_major;
      case 3:
        // 如果加载方法返回值为3，则跳转到状态 s_n_llhttp__internal__n_req_http_major
        goto s_n_llhttp__internal__n_req_http_major;
      // ... 其他case省略 ...
      default:
        // 如果加载方法返回值为其他值，则跳转到状态 s_n_llhttp__internal__n_error_29
        goto s_n_llhttp__internal__n_error_29;
    }
    /* UNREACHABLE */;
    // 不可达代码，中止程序
    abort();
  }
  s_n_llhttp__internal__n_invoke_llhttp__on_url_complete_1: {
    switch (llhttp__on_url_complete(state, p, endp)) {
      default:
        // 默认情况下，跳转到状态 s_n_llhttp__internal__n_req_http_start
        goto s_n_llhttp__internal__n_req_http_start;
    }
    /* UNREACHABLE */;
    // 不可达代码，中止程序
    abort();
  }
  s_n_llhttp__internal__n_span_end_llhttp__on_url_5: {
    const unsigned char* start;
    # 定义一个整型变量 err
    int err;

    # 将 state->_span_pos0 的值赋给 start，并将 state->_span_pos0 置为 NULL
    start = state->_span_pos0;
    state->_span_pos0 = NULL;
    # 调用 llhttp__on_url 函数，处理 URL，将结果赋给 err
    err = llhttp__on_url(state, start, p);
    # 如果 err 不等于 0，则将 err 赋给 state->error，将 p 赋给 state->error_pos，将 s_n_llhttp__internal__n_url_skip_to_http 转换为指针后赋给 state->_current，返回 s_error
    if (err != 0) {
      state->error = err;
      state->error_pos = (const char*) p;
      state->_current = (void*) (intptr_t) s_n_llhttp__internal__n_url_skip_to_http;
      return s_error;
    }
    # 跳转到标签 s_n_llhttp__internal__n_url_skip_to_http
    goto s_n_llhttp__internal__n_url_skip_to_http;
    # 不可达代码
    /* UNREACHABLE */;
    # 中止程序
    abort();
  }
  # 标签 s_n_llhttp__internal__n_span_end_llhttp__on_url_6
  s_n_llhttp__internal__n_span_end_llhttp__on_url_6: {
    # 定义一个指向无符号字符的指针 start，定义一个整型变量 err
    const unsigned char* start;
    int err;

    # 将 state->_span_pos0 的值赋给 start，并将 state->_span_pos0 置为 NULL
    start = state->_span_pos0;
    state->_span_pos0 = NULL;
    # 调用 llhttp__on_url 函数，处理 URL，将结果赋给 err
    err = llhttp__on_url(state, start, p);
    # 如果 err 不等于 0，则将 err 赋给 state->error，将 p 赋给 state->error_pos，将 s_n_llhttp__internal__n_url_skip_to_http09 转换为指针后赋给 state->_current，返回 s_error
    if (err != 0) {
      state->error = err;
      state->error_pos = (const char*) p;
      state->_current = (void*) (intptr_t) s_n_llhttp__internal__n_url_skip_to_http09;
      return s_error;
    }
    # 跳转到标签 s_n_llhttp__internal__n_url_skip_to_http09
    goto s_n_llhttp__internal__n_url_skip_to_http09;
    # 不可达代码
    /* UNREACHABLE */;
    # 中止程序
    abort();
  }
  # 标签 s_n_llhttp__internal__n_span_end_llhttp__on_url_7
  s_n_llhttp__internal__n_span_end_llhttp__on_url_7: {
    # 定义一个指向无符号字符的指针 start，定义一个整型变量 err
    const unsigned char* start;
    int err;

    # 将 state->_span_pos0 的值赋给 start，并将 state->_span_pos0 置为 NULL
    start = state->_span_pos0;
    state->_span_pos0 = NULL;
    # 调用 llhttp__on_url 函数，处理 URL，将结果赋给 err
    err = llhttp__on_url(state, start, p);
    # 如果 err 不等于 0，则将 err 赋给 state->error，将 p 赋给 state->error_pos，将 s_n_llhttp__internal__n_url_skip_lf_to_http09 转换为指针后赋给 state->_current，返回 s_error
    if (err != 0) {
      state->error = err;
      state->error_pos = (const char*) p;
      state->_current = (void*) (intptr_t) s_n_llhttp__internal__n_url_skip_lf_to_http09;
      return s_error;
    }
    # 跳转到标签 s_n_llhttp__internal__n_url_skip_lf_to_http09
    goto s_n_llhttp__internal__n_url_skip_lf_to_http09;
    # 不可达代码
    /* UNREACHABLE */;
    # 中止程序
    abort();
  }
  # 标签 s_n_llhttp__internal__n_span_end_llhttp__on_url_8
  s_n_llhttp__internal__n_span_end_llhttp__on_url_8: {
    # 定义一个指向无符号字符的指针 start，定义一个整型变量 err
    const unsigned char* start;
    int err;

    # 将 state->_span_pos0 的值赋给 start，并将 state->_span_pos0 置为 NULL
    start = state->_span_pos0;
    state->_span_pos0 = NULL;
    # 调用 llhttp__on_url 函数，处理 URL，将结果赋给 err
    err = llhttp__on_url(state, start, p);
    # 如果 err 不等于 0，则将 err 赋给 state->error，将 p 赋给 state->error_pos，将 s_n_llhttp__internal__n_url_skip_to_http 转换为指针后赋给 state->_current，返回 s_error
    if (err != 0) {
      state->error = err;
      state->error_pos = (const char*) p;
      state->_current = (void*) (intptr_t) s_n_llhttp__internal__n_url_skip_to_http;
      return s_error;
    }
    # 跳转到标签 s_n_llhttp__internal__n_url_skip_to_http
    goto s_n_llhttp__internal__n_url_skip_to_http;
    # 不可达代码
    /* UNREACHABLE */;
    # 中止程序
    abort();
  }
  # 标签 s_n_llhttp__internal__n_error_31
  s_n_llhttp__internal__n_error_31: {
    # 将 0x7 赋给 state->error
    state->error = 0x7;
    // 设置状态的原因为“URL片段起始处有无效字符”
    state->reason = "Invalid char in url fragment start";
    // 设置错误位置为当前位置
    state->error_pos = (const char*) p;
    // 设置当前状态为错误状态
    state->_current = (void*) (intptr_t) s_error;
    // 返回错误状态
    return s_error;
    /* UNREACHABLE */;  // 不可达代码
    // 终止程序
    abort();
  }
  s_n_llhttp__internal__n_span_end_llhttp__on_url_9: {
    const unsigned char* start;
    int err;

    start = state->_span_pos0;
    state->_span_pos0 = NULL;
    err = llhttp__on_url(state, start, p);
    if (err != 0) {
      state->error = err;
      state->error_pos = (const char*) p;
      state->_current = (void*) (intptr_t) s_n_llhttp__internal__n_url_skip_to_http09;
      return s_error;
    }
    goto s_n_llhttp__internal__n_url_skip_to_http09;
    /* UNREACHABLE */;  // 不可达代码
    // 终止程序
    abort();
  }
  s_n_llhttp__internal__n_span_end_llhttp__on_url_10: {
    const unsigned char* start;
    int err;

    start = state->_span_pos0;
    state->_span_pos0 = NULL;
    err = llhttp__on_url(state, start, p);
    if (err != 0) {
      state->error = err;
      state->error_pos = (const char*) p;
      state->_current = (void*) (intptr_t) s_n_llhttp__internal__n_url_skip_lf_to_http09;
      return s_error;
    }
    goto s_n_llhttp__internal__n_url_skip_lf_to_http09;
    /* UNREACHABLE */;  // 不可达代码
    // 终止程序
    abort();
  }
  s_n_llhttp__internal__n_span_end_llhttp__on_url_11: {
    const unsigned char* start;
    int err;

    start = state->_span_pos0;
    state->_span_pos0 = NULL;
    err = llhttp__on_url(state, start, p);
    if (err != 0) {
      state->error = err;
      state->error_pos = (const char*) p;
      state->_current = (void*) (intptr_t) s_n_llhttp__internal__n_url_skip_to_http;
      return s_error;
    }
    goto s_n_llhttp__internal__n_url_skip_to_http;
    /* UNREACHABLE */;  // 不可达代码
    // 终止程序
    abort();
  }
  s_n_llhttp__internal__n_error_32: {
    state->error = 0x7;
    state->reason = "Invalid char in url query";
    state->error_pos = (const char*) p;
    state->_current = (void*) (intptr_t) s_error;
    return s_error;
    /* UNREACHABLE */;  // 不可达代码
    // 中止程序
    abort();
  }
  // 设置错误状态为 0x7
  s_n_llhttp__internal__n_error_33: {
    state->error = 0x7;
    // 设置错误原因为 "Invalid char in url path"
    state->reason = "Invalid char in url path";
    // 设置错误位置为当前位置
    state->error_pos = (const char*) p;
    // 设置当前状态为错误状态
    state->_current = (void*) (intptr_t) s_error;
    // 返回错误状态
    return s_error;
    /* UNREACHABLE */;
    // 中止程序
    abort();
  }
  // 处理 URL 结束的状态
  s_n_llhttp__internal__n_span_end_llhttp__on_url: {
    const unsigned char* start;
    int err;

    // 获取 URL 开始位置
    start = state->_span_pos0;
    state->_span_pos0 = NULL;
    // 调用 llhttp__on_url 处理 URL
    err = llhttp__on_url(state, start, p);
    // 如果处理出错
    if (err != 0) {
      // 设置错误状态
      state->error = err;
      // 设置错误位置
      state->error_pos = (const char*) p;
      // 设置当前状态为跳转状态
      state->_current = (void*) (intptr_t) s_n_llhttp__internal__n_url_skip_to_http09;
      // 返回错误状态
      return s_error;
    }
    // 跳转到跳转状态
    goto s_n_llhttp__internal__n_url_skip_to_http09;
    /* UNREACHABLE */;
    // 中止程序
    abort();
  }
  // 处理 URL 结束的状态
  s_n_llhttp__internal__n_span_end_llhttp__on_url_1: {
    const unsigned char* start;
    int err;

    // 获取 URL 开始位置
    start = state->_span_pos0;
    state->_span_pos0 = NULL;
    // 调用 llhttp__on_url 处理 URL
    err = llhttp__on_url(state, start, p);
    // 如果处理出错
    if (err != 0) {
      // 设置错误状态
      state->error = err;
      // 设置错误位置
      state->error_pos = (const char*) p;
      // 设置当前状态为跳转状态
      state->_current = (void*) (intptr_t) s_n_llhttp__internal__n_url_skip_lf_to_http09;
      // 返回错误状态
      return s_error;
    }
    // 跳转到跳转状态
    goto s_n_llhttp__internal__n_url_skip_lf_to_http09;
    /* UNREACHABLE */;
    // 中止程序
    abort();
  }
  // 处理 URL 结束的状态
  s_n_llhttp__internal__n_span_end_llhttp__on_url_2: {
    const unsigned char* start;
    int err;

    // 获取 URL 开始位置
    start = state->_span_pos0;
    state->_span_pos0 = NULL;
    // 调用 llhttp__on_url 处理 URL
    err = llhttp__on_url(state, start, p);
    // 如果处理出错
    if (err != 0) {
      // 设置错误状态
      state->error = err;
      // 设置错误位置
      state->error_pos = (const char*) p;
      // 设置当前状态为跳转状态
      state->_current = (void*) (intptr_t) s_n_llhttp__internal__n_url_skip_to_http;
      // 返回错误状态
      return s_error;
    }
    // 跳转到跳转状态
    goto s_n_llhttp__internal__n_url_skip_to_http;
    /* UNREACHABLE */;
    // 中止程序
    abort();
  }
  // 处理 URL 结束的状态
  s_n_llhttp__internal__n_span_end_llhttp__on_url_12: {
    const unsigned char* start;
    int err;

    // 获取 URL 开始位置
    start = state->_span_pos0;
    state->_span_pos0 = NULL;
    // 调用 llhttp__on_url 处理 URL
    err = llhttp__on_url(state, start, p);
    if (err != 0) {
      # 如果错误码不为0，则将错误码赋值给状态对象的error属性
      state->error = err;
      # 将当前指针位置转换为const char*类型，并赋值给状态对象的error_pos属性
      state->error_pos = (const char*) p;
      # 将void*类型的_current属性设置为指向s_n_llhttp__internal__n_url_skip_to_http09函数的指针
      state->_current = (void*) (intptr_t) s_n_llhttp__internal__n_url_skip_to_http09;
      # 返回s_error状态
      return s_error;
    }
    # 跳转到s_n_llhttp__internal__n_url_skip_to_http09标签处
    goto s_n_llhttp__internal__n_url_skip_to_http09;
    /* UNREACHABLE */;
    # 终止程序执行
    abort();
  }
  s_n_llhttp__internal__n_span_end_llhttp__on_url_13: {
    # 定义无符号字符指针start
    const unsigned char* start;
    # 定义整型变量err
    int err;

    # 将_span_pos0属性的值赋给start
    start = state->_span_pos0;
    # 将_span_pos0属性的值设为NULL
    state->_span_pos0 = NULL;
    # 调用llhttp__on_url函数，将返回值赋给err
    err = llhttp__on_url(state, start, p);
    # 如果err不为0，则将错误码赋值给状态对象的error属性
    if (err != 0) {
      state->error = err;
      # 将当前指针位置转换为const char*类型，并赋值给状态对象的error_pos属性
      state->error_pos = (const char*) p;
      # 将void*类型的_current属性设置为指向s_n_llhttp__internal__n_url_skip_lf_to_http09函数的指针
      state->_current = (void*) (intptr_t) s_n_llhttp__internal__n_url_skip_lf_to_http09;
      # 返回s_error状态
      return s_error;
    }
    # 跳转到s_n_llhttp__internal__n_url_skip_lf_to_http09标签处
    goto s_n_llhttp__internal__n_url_skip_lf_to_http09;
    /* UNREACHABLE */;
    # 终止程序执行
    abort();
  }
  s_n_llhttp__internal__n_span_end_llhttp__on_url_14: {
    # 定义无符号字符指针start
    const unsigned char* start;
    # 定义整型变量err
    int err;

    # 将_span_pos0属性的值赋给start
    start = state->_span_pos0;
    # 将_span_pos0属性的值设为NULL
    state->_span_pos0 = NULL;
    # 调用llhttp__on_url函数，将返回值赋给err
    err = llhttp__on_url(state, start, p);
    # 如果err不为0，则将错误码赋值给状态对象的error属性
    if (err != 0) {
      state->error = err;
      # 将当前指针位置转换为const char*类型，并赋值给状态对象的error_pos属性
      state->error_pos = (const char*) p;
      # 将void*类型的_current属性设置为指向s_n_llhttp__internal__n_url_skip_to_http函数的指针
      state->_current = (void*) (intptr_t) s_n_llhttp__internal__n_url_skip_to_http;
      # 返回s_error状态
      return s_error;
    }
    # 跳转到s_n_llhttp__internal__n_url_skip_to_http标签处
    goto s_n_llhttp__internal__n_url_skip_to_http;
    /* UNREACHABLE */;
    # 终止程序执行
    abort();
  }
  s_n_llhttp__internal__n_error_34: {
    # 将状态对象的error属性设置为0x7
    state->error = 0x7;
    # 将"Double @ in url"赋值给状态对象的reason属性
    state->reason = "Double @ in url";
    # 将当前指针位置转换为const char*类型，并赋值给状态对象的error_pos属性
    state->error_pos = (const char*) p;
    # 将void*类型的_current属性设置为指向s_error函数的指针
    state->_current = (void*) (intptr_t) s_error;
    # 返回s_error状态
    return s_error;
    /* UNREACHABLE */;
    # 终止程序执行
    abort();
  }
  s_n_llhttp__internal__n_error_35: {
    # 将状态对象的error属性设置为0x7
    state->error = 0x7;
    # 将"Unexpected char in url server"赋值给状态对象的reason属性
    state->reason = "Unexpected char in url server";
    # 将当前指针位置转换为const char*类型，并赋值给状态对象的error_pos属性
    state->error_pos = (const char*) p;
    # 将void*类型的_current属性设置为指向s_error函数的指针
    state->_current = (void*) (intptr_t) s_error;
    # 返回s_error状态
    return s_error;
    /* UNREACHABLE */;
    # 终止程序执行
    abort();
  }
  s_n_llhttp__internal__n_error_36: {
    # 将状态对象的error属性设置为0x7
    state->error = 0x7;
    # 将"Unexpected char in url server"赋值给状态对象的reason属性
    state->reason = "Unexpected char in url server";
    # 将当前指针位置转换为const char*类型，并赋值给状态对象的error_pos属性
    state->error_pos = (const char*) p;
    state->_current = (void*) (intptr_t) s_error;
    return s_error;
    /* UNREACHABLE */;
    abort();
  }
  s_n_llhttp__internal__n_error_38: {
    state->error = 0x7;
    state->reason = "Unexpected char in url schema";
    state->error_pos = (const char*) p;
    state->_current = (void*) (intptr_t) s_error;
    return s_error;
    /* UNREACHABLE */;
    abort();
  }
  s_n_llhttp__internal__n_error_39: {
    state->error = 0x7;
    state->reason = "Unexpected char in url schema";
    state->error_pos = (const char*) p;
    state->_current = (void*) (intptr_t) s_error;
    return s_error;
    /* UNREACHABLE */;
    abort();
  }
  s_n_llhttp__internal__n_error_40: {
    state->error = 0x7;
    state->reason = "Unexpected start char in url";
    state->error_pos = (const char*) p;
    state->_current = (void*) (intptr_t) s_error;
    return s_error;
    /* UNREACHABLE */;
    abort();
  }
  s_n_llhttp__internal__n_invoke_is_equal_method: {
    switch (llhttp__internal__c_is_equal_method(state, p, endp)) {
      case 0:
        goto s_n_llhttp__internal__n_span_start_llhttp__on_url_1;
      default:
        goto s_n_llhttp__internal__n_span_start_llhttp__on_url;
    }
    /* UNREACHABLE */;
    abort();
  }
  s_n_llhttp__internal__n_error_41: {
    state->error = 0x6;
    state->reason = "Expected space after method";
    state->error_pos = (const char*) p;
    state->_current = (void*) (intptr_t) s_error;
    return s_error;
    /* UNREACHABLE */;
    abort();
  }
  s_n_llhttp__internal__n_invoke_store_method_1: {
    switch (llhttp__internal__c_store_method(state, p, endp, match)) {
      default:
        goto s_n_llhttp__internal__n_req_first_space_before_url;
    }
    /* UNREACHABLE */;
    abort();
  }
  s_n_llhttp__internal__n_error_49: {
    state->error = 0x6;
    state->reason = "Invalid method encountered";
    state->error_pos = (const char*) p;
    state->_current = (void*) (intptr_t) s_error;
    return s_error;
    /* UNREACHABLE */;
    abort();
    // 中止程序
    abort();
  }
  // 设置错误状态和原因
  s_n_llhttp__internal__n_error_42: {
    state->error = 0xd;
    state->reason = "Response overflow";
    state->error_pos = (const char*) p;
    state->_current = (void*) (intptr_t) s_error;
    return s_error;
    /* UNREACHABLE */;
    // 中止程序
    abort();
  }
  // 调用状态码更新函数
  s_n_llhttp__internal__n_invoke_mul_add_status_code: {
    switch (llhttp__internal__c_mul_add_status_code(state, p, endp, match)) {
      case 1:
        goto s_n_llhttp__internal__n_error_42;
      default:
        goto s_n_llhttp__internal__n_res_status_code;
    }
    /* UNREACHABLE */;
    // 中止程序
    abort();
  }
  // 结束状态码处理函数
  s_n_llhttp__internal__n_span_end_llhttp__on_status: {
    const unsigned char* start;
    int err;

    start = state->_span_pos0;
    state->_span_pos0 = NULL;
    err = llhttp__on_status(state, start, p);
    if (err != 0) {
      state->error = err;
      state->error_pos = (const char*) (p + 1);
      state->_current = (void*) (intptr_t) s_n_llhttp__internal__n_invoke_llhttp__on_status_complete;
      return s_error;
    }
    p++;
    goto s_n_llhttp__internal__n_invoke_llhttp__on_status_complete;
    /* UNREACHABLE */;
    // 中止程序
    abort();
  }
  // 结束状态码处理函数
  s_n_llhttp__internal__n_span_end_llhttp__on_status_1: {
    const unsigned char* start;
    int err;

    start = state->_span_pos0;
    state->_span_pos0 = NULL;
    err = llhttp__on_status(state, start, p);
    if (err != 0) {
      state->error = err;
      state->error_pos = (const char*) (p + 1);
      state->_current = (void*) (intptr_t) s_n_llhttp__internal__n_res_line_almost_done;
      return s_error;
    }
    p++;
    goto s_n_llhttp__internal__n_res_line_almost_done;
    /* UNREACHABLE */;
    // 中止程序
    abort();
  }
  // 设置错误状态和原因
  s_n_llhttp__internal__n_error_43: {
    state->error = 0xd;
    state->reason = "Invalid response status";
    state->error_pos = (const char*) p;
    state->_current = (void*) (intptr_t) s_error;
    return s_error;
    /* UNREACHABLE */;
    // 中止程序
    abort();
  }
  // 调用状态码更新函数
  s_n_llhttp__internal__n_invoke_update_status_code: {
  switch (llhttp__internal__c_update_status_code(state, p, endp)) {
    default:
      // 转到状态机的下一个状态
      goto s_n_llhttp__internal__n_res_status_code;
  }
  /* UNREACHABLE */;
  // 终止程序执行
  abort();
}
s_n_llhttp__internal__n_error_44: {
  state->error = 0x9;
  state->reason = "Expected space after version";
  state->error_pos = (const char*) p;
  state->_current = (void*) (intptr_t) s_error;
  return s_error;
  /* UNREACHABLE */;
  // 终止程序执行
  abort();
}
s_n_llhttp__internal__n_invoke_store_http_minor_1: {
  switch (llhttp__internal__c_store_http_minor(state, p, endp, match)) {
    default:
      // 转到状态机的下一个状态
      goto s_n_llhttp__internal__n_res_http_end;
  }
  /* UNREACHABLE */;
  // 终止程序执行
  abort();
}
s_n_llhttp__internal__n_error_45: {
  state->error = 0x9;
  state->reason = "Invalid minor version";
  state->error_pos = (const char*) p;
  state->_current = (void*) (intptr_t) s_error;
  return s_error;
  /* UNREACHABLE */;
  // 终止程序执行
  abort();
}
s_n_llhttp__internal__n_error_46: {
  state->error = 0x9;
  state->reason = "Expected dot";
  state->error_pos = (const char*) p;
  state->_current = (void*) (intptr_t) s_error;
  return s_error;
  /* UNREACHABLE */;
  // 终止程序执行
  abort();
}
s_n_llhttp__internal__n_invoke_store_http_major_1: {
  switch (llhttp__internal__c_store_http_major(state, p, endp, match)) {
    default:
      // 转到状态机的下一个状态
      goto s_n_llhttp__internal__n_res_http_dot;
  }
  /* UNREACHABLE */;
  // 终止程序执行
  abort();
}
s_n_llhttp__internal__n_error_47: {
  state->error = 0x9;
  state->reason = "Invalid major version";
  state->error_pos = (const char*) p;
  state->_current = (void*) (intptr_t) s_error;
  return s_error;
  /* UNREACHABLE */;
  // 终止程序执行
  abort();
}
s_n_llhttp__internal__n_error_50: {
  state->error = 0x8;
  state->reason = "Expected HTTP/";
  state->error_pos = (const char*) p;
  state->_current = (void*) (intptr_t) s_error;
  return s_error;
  /* UNREACHABLE */;
  // 终止程序执行
  abort();
}
s_n_llhttp__internal__n_invoke_update_type: {
  switch (llhttp__internal__c_update_type(state, p, endp)) {
    // 根据当前状态和输入的数据更新解析器的状态
    default:
      // 转到指定状态
      goto s_n_llhttp__internal__n_req_first_space_before_url;
  }
  /* UNREACHABLE */;
  // 如果执行到这里，说明代码逻辑有问题，中止程序
  abort();
}
s_n_llhttp__internal__n_invoke_store_method: {
  switch (llhttp__internal__c_store_method(state, p, endp, match)) {
    // 根据当前状态和输入的数据存储方法
    default:
      // 转到指定状态
      goto s_n_llhttp__internal__n_invoke_update_type;
  }
  /* UNREACHABLE */;
  // 如果执行到这里，说明代码逻辑有问题，中止程序
  abort();
}
s_n_llhttp__internal__n_error_48: {
  // 设置解析器的错误码和错误原因
  state->error = 0x8;
  state->reason = "Invalid word encountered";
  state->error_pos = (const char*) p;
  state->_current = (void*) (intptr_t) s_error;
  // 返回错误状态
  return s_error;
  /* UNREACHABLE */;
  // 如果执行到这里，说明代码逻辑有问题，中止程序
  abort();
}
s_n_llhttp__internal__n_invoke_update_type_1: {
  switch (llhttp__internal__c_update_type_1(state, p, endp)) {
    // 根据当前状态和输入的数据更新解析器的状态
    default:
      // 转到指定状态
      goto s_n_llhttp__internal__n_res_http_major;
  }
  /* UNREACHABLE */;
  // 如果执行到这里，说明代码逻辑有问题，中止程序
  abort();
}
s_n_llhttp__internal__n_invoke_update_type_2: {
  switch (llhttp__internal__c_update_type(state, p, endp)) {
    // 根据当前状态和输入的数据更新解析器的状态
    default:
      // 转到指定状态
      goto s_n_llhttp__internal__n_start_req;
  }
  /* UNREACHABLE */;
  // 如果执行到这里，说明代码逻辑有问题，中止程序
  abort();
}
s_n_llhttp__internal__n_pause_8: {
  // 设置解析器的错误码和错误原因
  state->error = 0x15;
  state->reason = "on_message_begin pause";
  state->error_pos = (const char*) p;
  state->_current = (void*) (intptr_t) s_n_llhttp__internal__n_invoke_load_type;
  // 返回错误状态
  return s_error;
  /* UNREACHABLE */;
  // 如果执行到这里，说明代码逻辑有问题，中止程序
  abort();
}
s_n_llhttp__internal__n_error: {
  // 设置解析器的错误码和错误原因
  state->error = 0x10;
  state->reason = "`on_message_begin` callback error";
  state->error_pos = (const char*) p;
  state->_current = (void*) (intptr_t) s_error;
  // 返回错误状态
  return s_error;
  /* UNREACHABLE */;
  // 如果执行到这里，说明代码逻辑有问题，中止程序
  abort();
}
s_n_llhttp__internal__n_invoke_llhttp__on_message_begin: {
    switch (llhttp__on_message_begin(state, p, endp)) {
      // 根据解析器状态和指针位置调用消息开始处理函数，根据返回值进行不同的处理
      case 0:
        // 如果返回值为0，跳转到内部状态处理标签
        goto s_n_llhttp__internal__n_invoke_load_type;
      case 21:
        // 如果返回值为21，跳转到内部状态处理标签
        goto s_n_llhttp__internal__n_pause_8;
      default:
        // 其他情况跳转到错误处理标签
        goto s_n_llhttp__internal__n_error;
    }
    /* UNREACHABLE */;
    // 不可达代码，中止程序
    abort();
  }
  s_n_llhttp__internal__n_invoke_update_finish: {
    switch (llhttp__internal__c_update_finish(state, p, endp)) {
      default:
        // 默认情况跳转到调用消息开始处理函数的标签
        goto s_n_llhttp__internal__n_invoke_llhttp__on_message_begin;
    }
    /* UNREACHABLE */;
    // 不可达代码，中止程序
    abort();
  }
}

int llhttp__internal_execute(llhttp__internal_t* state, const char* p, const char* endp) {
  llparse_state_t next;

  /* 检查是否有悬而未决的错误 */
  if (state->error != 0) {
    return state->error;
  }

  /* 重新启动跨度 */
  if (state->_span_pos0 != NULL) {
    state->_span_pos0 = (void*) p;
  }

  // 运行状态机，获取下一个状态
  next = llhttp__internal__run(state, (const unsigned char*) p, (const unsigned char*) endp);
  if (next == s_error) {
    return state->error;
  }
  state->_current = (void*) (intptr_t) next;

  /* 执行跨度处理函数 */
  if (state->_span_pos0 != NULL) {
    int error;

    error = ((llhttp__internal__span_cb) state->_span_cb0)(state, state->_span_pos0, (const char*) endp);
    if (error != 0) {
      state->error = error;
      state->error_pos = endp;
      return error;
    }
  }

  return 0;
}

#endif  /* LLHTTP_STRICT_MODE */
```