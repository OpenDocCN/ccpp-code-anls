# `nmap\libdnet-stripped\include\queue.h`

```
/*    $OpenBSD: queue.h,v 1.22 2001/06/23 04:39:35 angelos Exp $    */
/*    $NetBSD: queue.h,v 1.11 1996/05/16 05:17:14 mycroft Exp $    */

这部分代码是文件的注释，包含了文件的版本信息和作者信息。
/*
 * 版权声明，版权归加利福尼亚大学所有
 * 允许在源代码和二进制形式下进行再发布和使用，无论是否经过修改
 * 需要满足以下条件：
 * 1. 源代码的再发布必须保留上述版权声明、条件列表和以下免责声明
 * 2. 二进制形式的再发布必须在文档和/或其他提供的材料中重现上述版权声明、条件列表和以下免责声明
 * 3. 所有提及此软件特性或使用的广告材料必须显示以下声明：
 *    本产品包括加利福尼亚大学伯克利分校及其贡献者开发的软件
 * 4. 未经特定书面许可，不得使用大学的名称或其贡献者的名称来认可或推广从本软件派生的产品
 *
 * 本软件由大学和贡献者提供，不提供任何明示或暗示的担保，包括但不限于对适销性和特定用途的适用性的暗示担保
 * 在任何情况下，无论是合同责任、严格责任还是侵权（包括疏忽或其他方式）产生的任何损害，大学和贡献者均不对任何直接、间接、附带、特殊、惩罚性或后果性的损害负责
 * 即使已被告知可能发生此类损害，也不得对使用本软件的任何方式负责
 *
 *    @(#)queue.h    8.5 (Berkeley) 8/20/94
 */

#ifndef    _SYS_QUEUE_H_
#define    _SYS_QUEUE_H_

/*
 * 单链表定义
 */
#define SLIST_HEAD(name, type)                        \
struct name {                                \  # 定义一个结构体，用于表示单链表的头部
    struct type *slh_first;    /* first element */            \  # 结构体中包含一个指向第一个元素的指针
}
 
#define    SLIST_HEAD_INITIALIZER(head)                    \
    { NULL }  # 定义一个单链表头部的初始化器，初始化第一个元素为NULL
 
#define SLIST_ENTRY(type)                        \
struct {                                \  # 定义一个结构体，用于表示单链表的元素
    struct type *sle_next;    /* next element */            \  # 结构体中包含一个指向下一个元素的指针
}
 
/*
 * Singly-linked List access methods.
 */
#define    SLIST_FIRST(head)    ((head)->slh_first)  # 获取单链表的第一个元素
#define    SLIST_END(head)        NULL  # 获取单链表的结束标志
#define    SLIST_EMPTY(head)    (SLIST_FIRST(head) == SLIST_END(head))  # 判断单链表是否为空
#define    SLIST_NEXT(elm, field)    ((elm)->field.sle_next)  # 获取指定元素的下一个元素

#define    SLIST_FOREACH(var, head, field)                    \
    for((var) = SLIST_FIRST(head);                    \
        (var) != SLIST_END(head);                    \
        (var) = SLIST_NEXT(var, field))  # 遍历单链表中的每个元素
 
/*
 * Singly-linked List functions.
 */
#define    SLIST_INIT(head) {                        \
    SLIST_FIRST(head) = SLIST_END(head);                \  # 初始化单链表的头部
}

#define    SLIST_INSERT_AFTER(slistelm, elm, field) do {            \
    (elm)->field.sle_next = (slistelm)->field.sle_next;        \  # 在指定元素后插入新元素
    (slistelm)->field.sle_next = (elm);                \
} while (0)

#define    SLIST_INSERT_HEAD(head, elm, field) do {            \
    (elm)->field.sle_next = (head)->slh_first;            \  # 在单链表头部插入新元素
    (head)->slh_first = (elm);                    \
} while (0)

#define    SLIST_REMOVE_HEAD(head, field) do {                \
    (head)->slh_first = (head)->slh_first->field.sle_next;        \  # 移除单链表的第一个元素
} while (0)

#define SLIST_REMOVE(head, elm, type, field) do {            \
    if ((head)->slh_first == (elm)) {                \  # 移除指定元素
        SLIST_REMOVE_HEAD((head), field);            \
    }                                \
    else {                                \  # 如果条件不成立，则执行以下代码块
        struct type *curelm = (head)->slh_first;        \  # 声明并初始化一个指向头部第一个元素的指针
        while( curelm->field.sle_next != (elm) )        \  # 当指针指向的元素的下一个元素不是要删除的元素时执行循环
            curelm = curelm->field.sle_next;        \  # 将指针指向下一个元素
        curelm->field.sle_next =                \  # 将当前元素的下一个元素指针指向下下个元素
            curelm->field.sle_next->field.sle_next;        \  # 获取下下个元素的指针
    }                                \  # 结束 else 代码块
} while (0)

/*
 * List definitions.
 */
# 定义一个叫做 LIST_HEAD 的宏，用于声明一个链表头结构
#define LIST_HEAD(name, type)                        \
struct name {                                \
    struct type *lh_first;    /* first element */            \
}

# 定义一个叫做 LIST_HEAD_INITIALIZER 的宏，用于初始化链表头
#define LIST_HEAD_INITIALIZER(head)                    \
    { NULL }

# 定义一个叫做 LIST_ENTRY 的宏，用于声明链表中的元素结构
#define LIST_ENTRY(type)                        \
struct {                                \
    struct type *le_next;    /* next element */            \
    struct type **le_prev;    /* address of previous next element */    \
}

/*
 * List access methods
 */
# 定义获取链表第一个元素的宏
#define    LIST_FIRST(head)        ((head)->lh_first)
# 定义获取链表末尾的宏
#define    LIST_END(head)            NULL
# 定义判断链表是否为空的宏
#define    LIST_EMPTY(head)        (LIST_FIRST(head) == LIST_END(head))
# 定义获取下一个元素的宏
#define    LIST_NEXT(elm, field)        ((elm)->field.le_next)

# 定义遍历链表的宏
#define LIST_FOREACH(var, head, field)                    \
    for((var) = LIST_FIRST(head);                    \
        (var)!= LIST_END(head);                    \
        (var) = LIST_NEXT(var, field))

/*
 * List functions.
 */
# 定义初始化链表的宏
#define    LIST_INIT(head) do {                        \
    LIST_FIRST(head) = LIST_END(head);                \
} while (0)

# 定义在指定元素后插入新元素的宏
#define LIST_INSERT_AFTER(listelm, elm, field) do {            \
    if (((elm)->field.le_next = (listelm)->field.le_next) != NULL)    \
        (listelm)->field.le_next->field.le_prev =        \
            &(elm)->field.le_next;                \
    (listelm)->field.le_next = (elm);                \
    (elm)->field.le_prev = &(listelm)->field.le_next;        \
} while (0)

# 定义在指定元素前插入新元素的宏
#define    LIST_INSERT_BEFORE(listelm, elm, field) do {            \
    (elm)->field.le_prev = (listelm)->field.le_prev;        \
    (elm)->field.le_next = (listelm);                \
    *(listelm)->field.le_prev = (elm);                \
    (listelm)->field.le_prev = &(elm)->field.le_next;        \
} while (0)

# 定义在链表头插入新元素的宏
#define LIST_INSERT_HEAD(head, elm, field) do {                \
    # 如果链表头的第一个元素存在，则将新元素的下一个指针指向链表头的第一个元素
    if (((elm)->field.le_next = (head)->lh_first) != NULL)        \
        # 如果链表头的第一个元素存在，则将链表头的第一个元素的前一个指针指向新元素的下一个指针
        (head)->lh_first->field.le_prev = &(elm)->field.le_next;\
    # 将链表头的第一个元素指向新元素
    (head)->lh_first = (elm);                    \
    # 将新元素的前一个指针指向链表头的第一个元素
    (elm)->field.le_prev = &(head)->lh_first;            \
} while (0)

这是一个 do-while 循环的结束标记，用于结束 do-while 循环。


#define LIST_REMOVE(elm, field) do {                    \
    if ((elm)->field.le_next != NULL)                \
        (elm)->field.le_next->field.le_prev =            \
            (elm)->field.le_prev;                \
    *(elm)->field.le_prev = (elm)->field.le_next;            \
} while (0)

这是一个宏定义，用于从链表中移除指定元素。


#define LIST_REPLACE(elm, elm2, field) do {                \
    if (((elm2)->field.le_next = (elm)->field.le_next) != NULL)    \
        (elm2)->field.le_next->field.le_prev =            \
            &(elm2)->field.le_next;                \
    (elm2)->field.le_prev = (elm)->field.le_prev;            \
    *(elm2)->field.le_prev = (elm2);                \
} while (0)

这是一个宏定义，用于替换链表中的指定元素。


#define SIMPLEQ_HEAD(name, type)                    \
struct name {                                \
    struct type *sqh_first;    /* first element */            \
    struct type **sqh_last;    /* addr of last next element */        \
}

这是一个宏定义，用于定义简单队列的头部结构。


#define SIMPLEQ_HEAD_INITIALIZER(head)                    \
    { NULL, &(head).sqh_first }

这是一个宏定义，用于初始化简单队列的头部。


#define SIMPLEQ_ENTRY(type)                        \
struct {                                \
    struct type *sqe_next;    /* next element */            \
}

这是一个宏定义，用于定义简单队列的元素结构。


#define    SIMPLEQ_FIRST(head)        ((head)->sqh_first)
#define    SIMPLEQ_END(head)        NULL
#define    SIMPLEQ_EMPTY(head)        (SIMPLEQ_FIRST(head) == SIMPLEQ_END(head))
#define    SIMPLEQ_NEXT(elm, field)    ((elm)->field.sqe_next)

这是一组宏定义，用于访问简单队列的方法，包括获取第一个元素、判断队列是否为空、获取下一个元素等。


#define SIMPLEQ_FOREACH(var, head, field)                \
    for((var) = SIMPLEQ_FIRST(head);                \
        (var) != SIMPLEQ_END(head);                    \
        (var) = SIMPLEQ_NEXT(var, field))

这是一个宏定义，用于遍历简单队列中的元素。


#define    SIMPLEQ_INIT(head) do {                        \
    (head)->sqh_first = NULL;                    \
    (head)->sqh_last = &(head)->sqh_first;                \
} while (0)

这是一个宏定义，用于初始化简单队列。


#define SIMPLEQ_INSERT_HEAD(head, elm, field) do {            \

这是一个宏定义，用于向简单队列的头部插入元素。
    # 如果链表头指针指向的第一个元素为空，则将新元素的下一个指针指向空，并将链表尾指针指向新元素的下一个指针
    if (((elm)->field.sqe_next = (head)->sqh_first) == NULL)    \
        (head)->sqh_last = &(elm)->field.sqe_next;        \
    # 将链表头指针指向新元素
    (head)->sqh_first = (elm);                    \
} while (0)

#define SIMPLEQ_INSERT_TAIL(head, elm, field) do {            \
    // 将要插入的元素的下一个指针设置为NULL
    (elm)->field.sqe_next = NULL;                    \
    // 将头指针的最后一个元素指向要插入的元素
    *(head)->sqh_last = (elm);                    \
    // 将头指针的最后一个元素指针指向要插入的元素的下一个指针
    (head)->sqh_last = &(elm)->field.sqe_next;            \
} while (0)

#define SIMPLEQ_INSERT_AFTER(head, listelm, elm, field) do {        \
    // 如果要插入的元素的下一个指针为NULL，则将头指针的最后一个元素指针指向要插入的元素的下一个指针
    if (((elm)->field.sqe_next = (listelm)->field.sqe_next) == NULL)\
        (head)->sqh_last = &(elm)->field.sqe_next;        \
    // 将要插入的元素的下一个指针指向列表元素的下一个指针
    (listelm)->field.sqe_next = (elm);                \
} while (0)

#define SIMPLEQ_REMOVE_HEAD(head, elm, field) do {            \
    // 如果头指针的第一个元素的下一个指针为NULL，则将头指针的最后一个元素指针指向头指针的第一个元素指针
    if (((head)->sqh_first = (elm)->field.sqe_next) == NULL)    \
        (head)->sqh_last = &(head)->sqh_first;            \
} while (0)

/*
 * Tail queue definitions.
 */
#define TAILQ_HEAD(name, type)                        \
struct name {                                \
    struct type *tqh_first;    /* first element */            \
    struct type **tqh_last;    /* addr of last next element */        \
}

#define TAILQ_HEAD_INITIALIZER(head)                    \
    { NULL, &(head).tqh_first }

#define TAILQ_ENTRY(type)                        \
struct {                                \
    struct type *tqe_next;    /* next element */            \
    struct type **tqe_prev;    /* address of previous next element */    \
}

/* 
 * tail queue access methods 
 */
#define    TAILQ_FIRST(head)        ((head)->tqh_first)
#define    TAILQ_END(head)            NULL
#define    TAILQ_NEXT(elm, field)        ((elm)->field.tqe_next)
#define TAILQ_LAST(head, headname)                    \
    (*(((struct headname *)((head)->tqh_last))->tqh_last))
/* XXX */
#define TAILQ_PREV(elm, headname, field)                \
    (*(((struct headname *)((elm)->field.tqe_prev))->tqh_last))
#define    TAILQ_EMPTY(head)                        \
    (TAILQ_FIRST(head) == TAILQ_END(head))

#define TAILQ_FOREACH(var, head, field)                    \
    # 使用宏定义的方式遍历一个双向链表
    # 初始化变量 var 为链表头部的第一个元素
    # 循环条件为 var 不等于链表的结束标志
    # 每次循环将 var 更新为链表中 var 元素的下一个元素
// 定义一个宏，用于反向遍历双向链表
#define TAILQ_FOREACH_REVERSE(var, head, field, headname)        \
    for((var) = TAILQ_LAST(head, headname);                \
        (var) != TAILQ_END(head);                    \
        (var) = TAILQ_PREV(var, headname, field))

/*
 * 尾队列函数
 */
// 初始化尾队列
#define    TAILQ_INIT(head) do {                        \
    (head)->tqh_first = NULL;                    \
    (head)->tqh_last = &(head)->tqh_first;                \
} while (0)

// 在尾队列头部插入元素
#define TAILQ_INSERT_HEAD(head, elm, field) do {            \
    if (((elm)->field.tqe_next = (head)->tqh_first) != NULL)    \
        (head)->tqh_first->field.tqe_prev =            \
            &(elm)->field.tqe_next;                \
    else                                \
        (head)->tqh_last = &(elm)->field.tqe_next;        \
    (head)->tqh_first = (elm);                    \
    (elm)->field.tqe_prev = &(head)->tqh_first;            \
} while (0)

// 在尾队列尾部插入元素
#define TAILQ_INSERT_TAIL(head, elm, field) do {            \
    (elm)->field.tqe_next = NULL;                    \
    (elm)->field.tqe_prev = (head)->tqh_last;            \
    *(head)->tqh_last = (elm);                    \
    (head)->tqh_last = &(elm)->field.tqe_next;            \
} while (0)

// 在指定元素后面插入元素
#define TAILQ_INSERT_AFTER(head, listelm, elm, field) do {        \
    if (((elm)->field.tqe_next = (listelm)->field.tqe_next) != NULL)\
        (elm)->field.tqe_next->field.tqe_prev =            \
            &(elm)->field.tqe_next;                \
    else                                \
        (head)->tqh_last = &(elm)->field.tqe_next;        \
    (listelm)->field.tqe_next = (elm);                \
    (elm)->field.tqe_prev = &(listelm)->field.tqe_next;        \
} while (0)

// 在指定元素前面插入元素
#define    TAILQ_INSERT_BEFORE(listelm, elm, field) do {            \
    (elm)->field.tqe_prev = (listelm)->field.tqe_prev;        \
    (elm)->field.tqe_next = (listelm);                \
    *(listelm)->field.tqe_prev = (elm);                \
    # 将指向链表元素的指针赋值给elm指向的下一个元素的前一个元素的指针
    (listelm)->field.tqe_prev = &(elm)->field.tqe_next;        \
} while (0)
// do-while 循环，用于定义宏，确保宏在使用时能够像函数一样使用

#define TAILQ_REMOVE(head, elm, field) do {                \
    if (((elm)->field.tqe_next) != NULL)                \
        (elm)->field.tqe_next->field.tqe_prev =            \
            (elm)->field.tqe_prev;                \
    else                                \
        (head)->tqh_last = (elm)->field.tqe_prev;        \
    *(elm)->field.tqe_prev = (elm)->field.tqe_next;            \
} while (0)
// 定义宏 TAILQ_REMOVE，用于从双向链表中移除指定元素

#define TAILQ_REPLACE(head, elm, elm2, field) do {            \
    if (((elm2)->field.tqe_next = (elm)->field.tqe_next) != NULL)    \
        (elm2)->field.tqe_next->field.tqe_prev =        \
            &(elm2)->field.tqe_next;                \
    else                                \
        (head)->tqh_last = &(elm2)->field.tqe_next;        \
    (elm2)->field.tqe_prev = (elm)->field.tqe_prev;            \
    *(elm2)->field.tqe_prev = (elm2);                \
} while (0)
// 定义宏 TAILQ_REPLACE，用于替换双向链表中的指定元素

/*
 * Circular queue definitions.
 */
#define CIRCLEQ_HEAD(name, type)                    \
struct name {                                \
    struct type *cqh_first;        /* first element */        \
    struct type *cqh_last;        /* last element */        \
}
// 定义循环队列头部结构体，包含指向第一个元素和最后一个元素的指针

#define CIRCLEQ_HEAD_INITIALIZER(head)                    \
    { CIRCLEQ_END(&head), CIRCLEQ_END(&head) }
// 初始化循环队列头部结构体

#define CIRCLEQ_ENTRY(type)                        \
struct {                                \
    struct type *cqe_next;        /* next element */        \
    struct type *cqe_prev;        /* previous element */        \
}
// 定义循环队列元素结构体，包含指向下一个元素和上一个元素的指针

/*
 * Circular queue access methods 
 */
#define    CIRCLEQ_FIRST(head)        ((head)->cqh_first)
#define    CIRCLEQ_LAST(head)        ((head)->cqh_last)
#define    CIRCLEQ_END(head)        ((void *)(head))
#define    CIRCLEQ_NEXT(elm, field)    ((elm)->field.cqe_next)
#define    CIRCLEQ_PREV(elm, field)    ((elm)->field.cqe_prev)
#define    CIRCLEQ_EMPTY(head)                        \
    (CIRCLEQ_FIRST(head) == CIRCLEQ_END(head))
// 定义循环队列的访问方法，包括获取第一个元素、最后一个元素、下一个元素、上一个元素以及判断队列是否为空的方法
// 定义一个宏，用于遍历循环队列中的每个元素
#define CIRCLEQ_FOREACH(var, head, field)                \
    for((var) = CIRCLEQ_FIRST(head);                \
        (var) != CIRCLEQ_END(head);                    \
        (var) = CIRCLEQ_NEXT(var, field))

// 定义一个宏，用于反向遍历循环队列中的每个元素
#define CIRCLEQ_FOREACH_REVERSE(var, head, field)            \
    for((var) = CIRCLEQ_LAST(head);                    \
        (var) != CIRCLEQ_END(head);                    \
        (var) = CIRCLEQ_PREV(var, field))

/*
 * 循环队列函数
 */
// 初始化循环队列头部
#define    CIRCLEQ_INIT(head) do {                        \
    (head)->cqh_first = CIRCLEQ_END(head);                \
    (head)->cqh_last = CIRCLEQ_END(head);                \
} while (0)

// 在指定元素后插入新元素
#define CIRCLEQ_INSERT_AFTER(head, listelm, elm, field) do {        \
    (elm)->field.cqe_next = (listelm)->field.cqe_next;        \
    (elm)->field.cqe_prev = (listelm);                \
    if ((listelm)->field.cqe_next == CIRCLEQ_END(head))        \
        (head)->cqh_last = (elm);                \
    else                                \
        (listelm)->field.cqe_next->field.cqe_prev = (elm);    \
    (listelm)->field.cqe_next = (elm);                \
} while (0)

// 在指定元素前插入新元素
#define CIRCLEQ_INSERT_BEFORE(head, listelm, elm, field) do {        \
    (elm)->field.cqe_next = (listelm);                \
    (elm)->field.cqe_prev = (listelm)->field.cqe_prev;        \
    if ((listelm)->field.cqe_prev == CIRCLEQ_END(head))        \
        (head)->cqh_first = (elm);                \
    else                                \
        (listelm)->field.cqe_prev->field.cqe_next = (elm);    \
    (listelm)->field.cqe_prev = (elm);                \
} while (0)

// 在循环队列头部插入新元素
#define CIRCLEQ_INSERT_HEAD(head, elm, field) do {            \
    (elm)->field.cqe_next = (head)->cqh_first;            \
    (elm)->field.cqe_prev = CIRCLEQ_END(head);            \
    if ((head)->cqh_last == CIRCLEQ_END(head))            \
        (head)->cqh_last = (elm);                \
    # 如果链表头部存在元素，则将新元素的前一个元素指向链表头部的前一个元素
    else                                \
        (head)->cqh_first->field.cqe_prev = (elm);        \
    # 将链表头部的第一个元素指向新元素
    (head)->cqh_first = (elm);                    \
# do-while 循环，用于定义 CIRCLEQ_INSERT_TAIL 宏，将元素插入循环队列尾部
#define CIRCLEQ_INSERT_TAIL(head, elm, field) do {            \
    # 设置插入元素的下一个指针指向队列的结束标记
    (elm)->field.cqe_next = CIRCLEQ_END(head);            \
    # 设置插入元素的上一个指针指向队列的最后一个元素
    (elm)->field.cqe_prev = (head)->cqh_last;            \
    # 如果队列的第一个元素是结束标记，则将插入元素设置为队列的第一个元素
    if ((head)->cqh_first == CIRCLEQ_END(head))            \
        (head)->cqh_first = (elm);                \
    else                                \
        # 否则将插入元素设置为队列最后一个元素的下一个元素
        (head)->cqh_last->field.cqe_next = (elm);        \
    # 将插入元素设置为队列的最后一个元素
    (head)->cqh_last = (elm);                    \
} while (0)

# do-while 循环，用于定义 CIRCLEQ_REMOVE 宏，从循环队列中移除元素
#define    CIRCLEQ_REMOVE(head, elm, field) do {                \
    # 如果要移除的元素的下一个指针指向队列的结束标记
    if ((elm)->field.cqe_next == CIRCLEQ_END(head))            \
        # 将队列的最后一个元素设置为要移除元素的上一个元素
        (head)->cqh_last = (elm)->field.cqe_prev;        \
    else                                \
        # 否则将要移除元素的下一个元素的上一个指针指向要移除元素的上一个元素
        (elm)->field.cqe_next->field.cqe_prev =            \
            (elm)->field.cqe_prev;                \
    # 如果要移除的元素的上一个指针指向队列的结束标记
    if ((elm)->field.cqe_prev == CIRCLEQ_END(head))            \
        # 将队列的第一个元素设置为要移除元素的下一个元素
        (head)->cqh_first = (elm)->field.cqe_next;        \
    else                                \
        # 否则将要移除元素的上一个元素的下一个指针指向要移除元素的下一个元素
        (elm)->field.cqe_prev->field.cqe_next =            \
            (elm)->field.cqe_next;                \
} while (0)

# do-while 循环，用于定义 CIRCLEQ_REPLACE 宏，替换循环队列中的元素
#define CIRCLEQ_REPLACE(head, elm, elm2, field) do {            \
    # 如果要替换的元素的下一个指针设置为要替换元素的下一个指针，并且要替换元素的下一个指针指向队列的结束标记
    if (((elm2)->field.cqe_next = (elm)->field.cqe_next) ==        \
        CIRCLEQ_END(head))                        \
        # 将队列的最后一个元素设置为要替换的元素
        (head).cqh_last = (elm2);                \
    else                                \
        # 否则将要替换元素的下一个元素的上一个指针指向要替换元素
        (elm2)->field.cqe_next->field.cqe_prev = (elm2);    \
    # 如果要替换的元素的上一个指针设置为要替换元素的上一个指针，并且要替换元素的上一个指针指向队列的结束标记
    if (((elm2)->field.cqe_prev = (elm)->field.cqe_prev) ==        \
        CIRCLEQ_END(head))                        \
        # 将队列的第一个元素设置为要替换的元素
        (head).cqh_first = (elm2);                \
    else                                \
        # 否则将要替换元素的上一个元素的下一个指针指向要替换元素
        (elm2)->field.cqe_prev->field.cqe_next = (elm2);    \
} while (0)

# 结束宏定义
#endif    /* !_SYS_QUEUE_H_ */
```