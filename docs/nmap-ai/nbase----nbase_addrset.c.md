# `nmap\nbase\nbase_addrset.c`

```
/* $Id$ */

/* The code in this file has tests in the file ncat/tests/test-addrset.sh. Run that
   program after making any big changes. Also, please add tests for any new
   features. */

#include <limits.h> /* CHAR_BIT */
#include <errno.h>
#include <assert.h>

#include "nbase.h"

/* A fancy logging system to allow this file to take advantage of different logging
   systems used by various programs */

static void default_log_user(const char * a, ...){};

static void (*log_user)(const char *, ...) = default_log_user;

static void default_log_debug(const char * a, ...){};

static void (*log_debug)(const char *, ...) = default_log_debug;

void nbase_set_log(void (*log_user_func)(const char *, ...),void (*log_debug_func)(const char *, ...)){
    if (log_user_func == NULL)
        log_user = default_log_user;
    else
        log_user = log_user_func;
    if (log_debug_func == NULL)
        log_debug = default_log_debug;
    else
        log_debug = log_debug_func;
}

/* Node for a radix tree (trie) used to match certain addresses.
 * Currently, only individual numeric IP and IPv6 addresses are matched using
 * the trie. */
struct trie_node {
  /* The address prefix that this node represents. */
  u32 addr[4];
  /* The prefix mask. Bits in addr that are not within this mask are ignored. */
  u32 mask[4];
  /* Addresses with the next bit after the mask equal to 1 are on this branch. */
  struct trie_node *next_bit_one;
  /* Addresses with the next bit after the mask equal to 0 are on this branch. */
  struct trie_node *next_bit_zero;
};

/* We use bit vectors to represent what values are allowed in an IPv4 octet.
   Each vector is built up of an array of bitvector_t (any convenient integer
   type). */
typedef unsigned long bitvector_t;
/* A 256-element bit vector, representing legal values for one octet. */
typedef bitvector_t octet_bitvector[(256 - 1) / (sizeof(unsigned long) * CHAR_BIT) + 1];
/* 一系列用于集合包含测试的链。如果一个测试通过，则地址在集合中。 */
struct addrset_elem {
  struct {
    /* 每个地址八位的位向量。 */
    octet_bitvector bits[4];
  } ipv4;
  struct addrset_elem *next;
};

/* 一组地址。用于匹配允许/拒绝列表。 */
struct addrset {
    /* struct addset_elem 的链表。 */
    struct addrset_elem *head;
    /* 用于更快匹配某些情况的基数树 */
    struct trie_node *trie;
};

/* 特殊的节点指针，表示“所有可能的地址”
 * 这将用于表示网络掩码规范。 */
static struct trie_node g_TRIE_NODE_TRUE = {0};
#define TRIE_NODE_TRUE &g_TRIE_NODE_TRUE

struct addrset *addrset_new()
{
    struct addrset *set = (struct addrset *) safe_zalloc(sizeof(struct addrset));
    set->head = NULL;

    /* 分配 IPv4 基数树的第一个节点 */
    set->trie = (struct trie_node *) safe_zalloc(sizeof(struct trie_node));
    return set;
}

static void trie_free(struct trie_node *curr)
{
  /* 因为我们只沿着一条路径下降，所以最多积累一棵树的深度，或者 128。
   * 添加 4 作为安全保障，以考虑特殊的根节点和特殊的空栈位置 0。 */
  struct trie_node *stack[128+4] = {NULL};
  int i = 1;

  while (i > 0 && curr != NULL && curr != TRIE_NODE_TRUE) {
    /* 存储 next_bit_one */
    if (curr->next_bit_one != NULL && curr->next_bit_one != TRIE_NODE_TRUE) {
      stack[i++] = curr->next_bit_one;
    }
    /* 如果 next_bit_zero 有效，下降 */
    if (curr->next_bit_zero != NULL && curr->next_bit_zero != TRIE_NODE_TRUE) {
      curr = curr->next_bit_zero;
    }
    else {
      /* next_bit_one 已存储，next_bit_zero 无效。释放它并向上移动栈。 */
      free(curr);
      curr = stack[--i];
    }
  }
}

void addrset_free(struct addrset *set)
{
    struct addrset_elem *elem, *next;
    # 遍历集合中的元素，从头部开始
    for (elem = set->head; elem != NULL; elem = next) {
        # 保存当前元素的下一个元素的指针
        next = elem->next;
        # 释放当前元素的内存空间
        free(elem);
    }

    # 释放集合中使用的字典树的内存空间
    trie_free(set->trie);
    # 释放集合本身的内存空间
    free(set);
/* Public domain log2 function. https://graphics.stanford.edu/~seander/bithacks.html#IntegerLogLookup */
// 定义一个256个元素的数组，用于快速查找log2的值
static const char LogTable256[256] = {
#define LT(n) n, n, n, n, n, n, n, n, n, n, n, n, n, n, n, n
  -1, 0, 1, 1, 2, 2, 2, 2, 3, 3, 3, 3, 3, 3, 3, 3,
  LT(4), LT(5), LT(5), LT(6), LT(6), LT(6), LT(6),
  LT(7), LT(7), LT(7), LT(7), LT(7), LT(7), LT(7), LT(7)
};

/* Returns a mask representing the common prefix between 2 values. */
// 返回一个表示两个值之间公共前缀的掩码
static u32 common_mask(u32 a, u32 b)
{
  u8 r;     // r will be lg(v)  r将是lg(v)
  u32 t, tt; // temporaries  临时变量
  u32 v = a ^ b;  // 计算a和b的异或值
  if (v == 0) {
    /* values are equal, all bits are the same */
    // 值相等，所有位都相同
    return 0xffffffff;  // 返回所有位都为1的掩码
  }

  if ((tt = v >> 16))
  {
    r = (t = tt >> 8) ? 24 + LogTable256[t] : 16 + LogTable256[tt];  // 根据v的值计算r
  }
  else
  {
    r = (t = v >> 8) ? 8 + LogTable256[t] : LogTable256[v];  // 根据v的值计算r
  }
  if (r + 1 >= 32) {
    /* shifting this many bits would overflow. Just return max mask */
    // 移动这么多位会溢出。只需返回最大的掩码
    return 0;
  }
  else {
    return ~((1 << (r + 1)) - 1);  // 返回r+1位为1的掩码
  }
}

/* Given a mask and a value, return the value of the bit immediately following
 * the masked bits. */
// 给定一个掩码和一个值，返回掩码位之后的值
static u32 next_bit_is_one(u32 mask, u32 value) {
  if (mask == 0) {
    /* no masked bits, check the first bit. */
    // 没有掩码位，检查第一位
    return (0x80000000 & value);  // 返回第一位的值
  }
  else if (mask == 0xffffffff) {
    /* Imaginary bit off the end we will say is 0 */
    // 虚构的位在末尾，我们将其视为0
    return 0;  // 返回0
  }
  /* isolate the bit by overlapping the mask with its inverse */
  // 通过将掩码与其反向重叠来隔离位
  return ((mask >> 1) & ~mask) & value;  // 返回位的值
}

/* Given a mask and an address, return true if the first unmasked bit is one */
// 给定一个掩码和一个地址，如果第一个未掩码的位为1，则返回true
static u32 addr_next_bit_is_one(const u32 *mask, const u32 *addr) {
  u32 curr_mask;
  u8 i;
  for (i = 0; i < 4; i++) {
    curr_mask = mask[i];
    if (curr_mask < 0xffffffff) {
      /* Only bother checking the first not-completely-masked portion of the address */
      // 只需检查地址的第一个未完全掩码的部分
      return next_bit_is_one(curr_mask, addr[i]);  // 返回地址的第一个未掩码的位的值
    }
  }
  /* Mask must be all ones, meaning that the next bit is off the end, and clearly not 1. */
  // 掩码必须全部为1，这意味着下一个位在末尾，显然不是1
  return 0;  // 返回0
}
/* 如果 a 和 b 的掩码部分相同，则返回 true */
static int mask_matches(u32 mask, u32 a, u32 b)
{
  return !(mask & (a ^ b));
}

/* 应用掩码并检查两个地址是否相等 */
static int addr_matches(const u32 *mask, const u32 *sa, const u32 *sb)
{
  u32 curr_mask;
  u8 i;
  for (i = 0; i < 4; i++) {
    curr_mask = mask[i];
    if (curr_mask == 0) {
      /* 没有更多适用的位 */
      break;
    }
    else if (!mask_matches(curr_mask, sa[i], sb[i])) {
      /* 不匹配 */
      return 0;
    }
  }
  /* 所有适用的位都匹配 */
  return 1;
}

/* 分配并初始化一个新节点的辅助函数 */
static struct trie_node *new_trie_node(const u32 *addr, const u32 *mask)
{
  u8 i;
  struct trie_node *new_node = (struct trie_node *) safe_zalloc(sizeof(struct trie_node));
  for (i=0; i < 4; i++) {
    new_node->addr[i] = addr[i];
    new_node->mask[i] = mask[i];
  }
  /* 新节点默认匹配为 true。如果不是，则覆盖。 */
  new_node->next_bit_one = new_node->next_bit_zero = TRIE_NODE_TRUE;
  return new_node;
}

/* 将一个节点分成两个部分：一个与地址具有最大公共前缀匹配，一个不匹配。 */
static void trie_split (struct trie_node *this, const u32 *addr, const u32 *mask)
{
  struct trie_node *new_node;
  u32 new_mask[4] = {0,0,0,0};
  u8 i;
  /* 计算公共前缀的掩码 */
  for (i=0; i < 4; i++) {
    new_mask[i] = common_mask(this->addr[i], addr[i]);
    if (new_mask[i] > this->mask[i]){
      /* 地址具有比我们关心的节点更多的公共位。 */
      new_mask[i] = this->mask[i];
    }
    # 如果新的掩码比当前节点的掩码更宽
    if (new_mask[i] > mask[i]) {
      # 更新当前节点的掩码为新掩码的对应部分
      this->mask[i] = mask[i];
      # 将当前节点的掩码后面的部分全部置为0
      for (i++; i < 4; i++) {
        this->mask[i] = 0;
      }
      # 释放当前节点的子节点
      trie_free(this->next_bit_one);
      trie_free(this->next_bit_zero);
      # 将当前节点的子节点指向特殊节点 TRIE_NODE_TRUE
      this->next_bit_one = this->next_bit_zero = TRIE_NODE_TRUE;
      # 返回
      return;
    }
    # 如果新的掩码的第i位小于0xffffffff
    if (new_mask[i] < 0xffffffff) {
      # 跳出循环
      break;
    }
  }
  # 如果i大于等于4或者新的掩码的第i位大于等于当前节点的掩码的第i位
  if (i >= 4 || new_mask[i] >= this->mask[i]) {
    # 当前节点完全包含新的地址和掩码，无需拆分或添加
    return;
  }
  # 复制当前节点以继续匹配
  new_node = new_trie_node(this->addr, this->mask);
  new_node->next_bit_one = this->next_bit_one;
  new_node->next_bit_zero = this->next_bit_zero;
  # 调整当前节点的掩码为较小的掩码
  for (i=0; i < 4; i++) {
    this->mask[i] = new_mask[i];
  }
  # 将新节点放在适当的分支上
  if (addr_next_bit_is_one(this->mask, this->addr)) {
    this->next_bit_one = new_node;
    this->next_bit_zero = NULL;
  }
  else {
    this->next_bit_zero = new_node;
    this->next_bit_one = NULL;
  }
}

/* 插入地址的辅助函数 */
static void _trie_insert (struct trie_node *this, const u32 *addr, const u32 *mask)
{
  /* 进入时，至少第一个比特必须匹配这个节点 */
  assert(this == TRIE_NODE_TRUE || (this->addr[0] ^ addr[0]) < (1 << 31));

  while (this != NULL && this != TRIE_NODE_TRUE) {
    /* 如有必要，拆分节点以确保匹配 */
    trie_split(this, addr, mask);

    /* 此时，该节点与地址匹配到 this->mask。 */
    if (addr_next_bit_is_one(this->mask, addr)) {
      /* 下一个比特为一：在一分支上插入 */
      if (this->next_bit_one == NULL) {
        /* 以前不匹配的分支，拆分时总是这种情况 */
        this->next_bit_one = new_trie_node(addr, mask);
        return;
      }
      else {
        this = this->next_bit_one;
      }
    }
    else {
      /* 下一个比特为零：在零分支上插入 */
      if (this->next_bit_zero == NULL) {
        /* 以前不匹配的分支，拆分时总是这种情况 */
        this->next_bit_zero = new_trie_node(addr, mask);
        return;
      }
      else {
        this = this->next_bit_zero;
      }
    }
  }
}

/* 将 sockaddr 转换为 u32 数组的辅助函数，内部使用 */
static int sockaddr_to_addr(const struct sockaddr *sa, u32 *addr)
{
  if (sa->sa_family == AF_INET) {
    /* IPv4-mapped IPv6 地址 */
    addr[0] = addr[1] = 0;
    addr[2] = 0xffff;
    addr[3] = ntohl(((struct sockaddr_in *) sa)->sin_addr.s_addr);
  }
#ifdef HAVE_IPV6
  else if (sa->sa_family == AF_INET6) {
    u8 i;
    unsigned char *addr6 = ((struct sockaddr_in6 *) sa)->sin6_addr.s6_addr;
    for (i=0; i < 4; i++) {
      addr[i] = (addr6[i*4] << 24) + (addr6[i*4+1] << 16) + (addr6[i*4+2] << 8) + addr6[i*4+3];
    }
  }
#endif
  else {
    return 0;
  }
  return 1;
}

static int sockaddr_to_mask (const struct sockaddr *sa, int bits, u32 *mask)
{
  int i, k;
  if (bits >= 0) {
    if (sa->sa_family == AF_INET) {
      bits += 96;
    }
#ifdef HAVE_IPV6
    // 如果支持 IPv6，则不执行任何操作
    else if (sa->sa_family == AF_INET6) {
      ; /* do nothing */
    }
#endif
    else {
      // 如果不支持 IPv6，则返回 0
      return 0;
    }
  }
  else
    // 设置默认的位数为 128
    bits = 128;
  // 根据位数计算出 k 的值
  k = bits / 32;
  // 遍历数组，根据 k 的值设置 mask 数组的值
  for (i=0; i < 4; i++) {
    if (i < k) {
      mask[i] = 0xffffffff;
    }
    else if (i > k) {
      mask[i] = 0;
    }
    else {
      mask[i] = 0xfffffffe << (31 - bits % 32);
    }
  }
  // 返回 1
  return 1;
}

/* Insert a sockaddr into the trie */
static void trie_insert (struct trie_node *this, const struct sockaddr *sa, int bits)
{
  u32 addr[4] = {0};
  u32 mask[4] = {0};
  // 将 sockaddr 转换为地址数组
  if (!sockaddr_to_addr(sa, addr)) {
    log_debug("Unknown address family %u, address not inserted.\n", sa->sa_family);
    return;
  }
  // 将 sockaddr 转换为掩码数组
  if (!sockaddr_to_mask(sa, bits, mask)) {
    log_debug("Bad netmask length %d for address family %u, address not inserted.\n", bits, sa->sa_family);
    return;
  }
  /* First node doesn't have a mask or address of its own; we have to check the
   * first bit manually. */
  // 第一个节点没有自己的掩码或地址；我们必须手动检查第一个位
  if (0x80000000 & addr[0]) {
    /* First bit is 1, so insert on ones branch */
    // 第一个位是 1，所以插入到 ones 分支
    if (this->next_bit_one == NULL) {
      /* Empty branch, just add it. */
      // 空分支，直接添加
      this->next_bit_one = new_trie_node(addr, mask);
      return;
    }
    _trie_insert(this->next_bit_one, addr, mask);
  }
  else {
    /* First bit is 0, so insert on zeros branch */
    // 第一个位是 0，所以插入到 zeros 分支
    if (this->next_bit_zero == NULL) {
      /* Empty branch, just add it. */
      // 空分支，直接添加
      this->next_bit_zero = new_trie_node(addr, mask);
      return;
    }
    _trie_insert(this->next_bit_zero, addr, mask);
  }
}

/* Helper for matching addresses */
static int _trie_match (const struct trie_node *this, const u32 *addr)
{
  while (this != TRIE_NODE_TRUE && this != NULL
    && addr_matches(this->mask, this->addr, addr)) {
    if (1 & this->mask[3]) {
      /* We've matched all possible bits! Yay! */
      // 匹配了所有可能的位！耶！
      return 1;
    }
    else if (addr_next_bit_is_one(this->mask, addr)) {
      this = this->next_bit_one;
    }
    else {
      this = this->next_bit_zero;
    }
  }
}
    }
  }
  # 如果当前节点为真，则返回1
  if (this == TRIE_NODE_TRUE) {
    return 1;
  }
  # 如果当前节点不为真，则返回0
  return 0;
static int trie_match (const struct trie_node *this, const struct sockaddr *sa)
{
  // 创建一个长度为4的地址数组，并初始化为0
  u32 addr[4] = {0};
  // 如果无法将sockaddr转换为地址数组，则打印错误信息并返回0
  if (!sockaddr_to_addr(sa, addr)) {
    log_debug("Unknown address family %u, cannot match.\n", sa->sa_family);
    return 0;
  }
  /* 手动检查第一个比特位，以决定匹配哪个分支 */
  if (0x80000000 & addr[0]) {
    // 如果第一个比特位为1，则调用_trie_match函数匹配next_bit_one分支
    return _trie_match(this->next_bit_one, addr);
  }
  else {
    // 如果第一个比特位为0，则调用_trie_match函数匹配next_bit_zero分支
    return _trie_match(this->next_bit_zero, addr);
  }
  // 不会执行到这里的代码，因此可以省略
  return 0;
}

/* 用于打印addrset_elem的内容的调试函数。对于IPv4，这是四个位向量。对于IPv6，这是地址和子网掩码。 */
static void addrset_elem_print(FILE *fp, const struct addrset_elem *elem)
{
    // 计算位向量的数量
    const size_t num_bitvector = sizeof(octet_bitvector) / sizeof(bitvector_t);
    int i;
    size_t j;

    // 遍历四个位向量并打印到文件流中
    for (i = 0; i < 4; i++) {
      for (j = 0; j < num_bitvector; j++)
        fprintf(fp, "%0*lX ", (int) (sizeof(bitvector_t) * 2), elem->ipv4.bits[i][num_bitvector - 1 - j]);
      fprintf(fp, "\n");
    }
}

// 打印addrset中每个addrset_elem的内容
void addrset_print(FILE *fp, const struct addrset *set)
{
  const struct addrset_elem *elem;
  // 遍历addrset中的每个addrset_elem，并打印其内容
  for (elem = set->head; elem != NULL; elem = elem->next) {
    fprintf(fp, "addrset_elem: %p\n", elem);
    addrset_elem_print(fp, elem);
  }
}

/* 这是对getaddrinfo的包装，自动处理IPv4/IPv6、TCP/UDP和是否允许名称解析的hints。 */
static int resolve_name(const char *name, struct addrinfo **result, int af, int use_dns)
{
    // 初始化hints结构体
    struct addrinfo hints = { 0 };
    int rc;

    hints.ai_protocol = IPPROTO_TCP;

    /* 首先进行非DNS查找，对任何地址族进行查找（只检查有效的数值地址）。我们无论af的设置如何，都识别数值地址。如果use_dns为false，则这也是最后一步。 */
    hints.ai_flags |= AI_NUMERICHOST;
    hints.ai_family = AF_UNSPEC;
    *result = NULL;
    // 调用getaddrinfo函数进行地址解析
    rc = getaddrinfo(name, NULL, &hints, result);
    // 如果返回值为0或者use_dns为false，则直接返回结果
    if (rc == 0 || !use_dns)
        return rc;
    # 执行 DNS 查询。当我们查询一个名称时，我们只想要与 af 值对应的地址
    # 取消 AI_NUMERICHOST 标志，以便进行实际的主机名解析
    hints.ai_flags &= ~AI_NUMERICHOST;
    # 设置地址族（IPv4或IPv6）
    hints.ai_family = af;
    # 初始化结果指针
    *result = NULL;
    # 执行主机名解析，将结果存储在 result 指针指向的位置
    rc = getaddrinfo(name, NULL, &hints, result);

    # 返回主机名解析的结果
    return rc;
}

/* 这是一个与地址族无关的inet_ntop的版本。*/
static char *address_to_string(const struct sockaddr *sa, size_t sa_len,
                               char *buf, size_t len)
{
    // 将地址转换为可读的字符串形式
    getnameinfo(sa, sa_len, buf, len, NULL, 0, NI_NUMERICHOST);

    return buf;
}

/* 将IPv4地址分解为八位字节的数组。octets[0]包含最高位字节，octets[3]包含最低位字节。*/
static void in_addr_to_octets(const struct in_addr *ia, uint8_t octets[4])
{
    u32 hbo = ntohl(ia->s_addr);

    octets[0] = (uint8_t) ((hbo & (0xFFU << 24)) >> 24);
    octets[1] = (uint8_t) ((hbo & (0xFFU << 16)) >> 16);
    octets[2] = (uint8_t) ((hbo & (0xFFU << 8)) >> 8);
    octets[3] = (uint8_t) (hbo & 0xFFU);
}

#define BITVECTOR_BITS (sizeof(bitvector_t) * CHAR_BIT)
#define BIT_SET(v, n) ((v)[(n) / BITVECTOR_BITS] |= 1UL << ((n) % BITVECTOR_BITS))
#define BIT_IS_SET(v, n) (((v)[(n) / BITVECTOR_BITS] & 1UL << ((n) % BITVECTOR_BITS)) != 0)

static int parse_ipv4_ranges(struct addrset_elem *elem, const char *spec);
static void apply_ipv4_netmask_bits(struct addrset_elem *elem, int bits);

/* 将主机规范添加到地址集中。成功返回1，错误返回0。*/
int addrset_add_spec(struct addrset *set, const char *spec, int af, int dns)
{
    char *local_spec;
    char *netmask_s;
    const char *tail;
    long netmask_bits;
    struct addrinfo *addrs, *addr;
    struct addrset_elem *elem;
    int rc;

    /* 复制规范以便操作。*/
    local_spec = strdup(spec);
    if (local_spec == NULL)
        return 0;

    /* 读取CIDR掩码位数，如果存在。*/
    netmask_s = strchr(local_spec, '/');
    if (netmask_s == NULL) {
        /* 负值表示未指定；默认取决于地址族。*/
        netmask_bits = -1;
    } else {
        // 如果不是 CIDR 格式的子网掩码，则将 netmask_s 指向字符串结尾的空字符
        *netmask_s = '\0';
        // 移动 netmask_s 指针到下一个字符
        netmask_s++;
        // 重置 errno
        errno = 0;
        // 解析 netmask_s 中的长整型数值，存入 netmask_bits，同时检查是否解析成功
        netmask_bits = parse_long(netmask_s, &tail);
        // 如果解析出错或者 tail 指向非空字符，或者 tail 和 netmask_s 指向同一个位置，则输出错误信息，释放内存，返回 0
        if (errno != 0 || *tail != '\0' || tail == netmask_s) {
            log_user("Error parsing netmask in \"%s\".\n", spec);
            free(local_spec);
            return 0;
        }
    }

    /* See if it's a plain IP address */
    // 解析本地地址，存入 addrs，af 为地址族，0 表示不进行 DNS 解析
    rc = resolve_name(local_spec, &addrs, af, 0);
    // 如果解析成功且 addrs 不为空
    if (rc == 0 && addrs != NULL) {
      /* Add all addresses to the trie */
      // 遍历 addrs 中的地址，将其添加到 trie 中
      for (addr = addrs; addr != NULL; addr = addr->ai_next) {
        // 用于存放地址字符串的缓冲区
        char addr_string[128];
        // 如果地址族为 AF_INET 且子网掩码位数大于 32
        if ((addr->ai_family == AF_INET && netmask_bits > 32)
#ifdef HAVE_IPV6
          || (addr->ai_family == AF_INET6 && netmask_bits > 128)
#endif
          ) {
          // 如果地址族为 IPv6 并且子网掩码大于 128，则输出错误信息并释放内存后返回 0
          log_user("Illegal netmask in \"%s\". Must be smaller than address bit length.\n", spec);
          free(local_spec);
          freeaddrinfo(addrs);
          return 0;
        }
        // 将地址转换为字符串形式
        address_to_string(addr->ai_addr, addr->ai_addrlen, addr_string, sizeof(addr_string));
        // 将地址和子网掩码插入到前缀树中
        trie_insert(set->trie, addr->ai_addr, netmask_bits);
        // 输出调试信息
        log_debug("Add IP %s/%d to addrset (trie).\n", addr_string, netmask_bits);
      }
      // 释放内存
      free(local_spec);
      freeaddrinfo(addrs);
      return 1;
    }

    // 分配内存并初始化为 0
    elem = (struct addrset_elem *) safe_malloc(sizeof(*elem));
    memset(elem->ipv4.bits, 0, sizeof(elem->ipv4.bits));

    /* Check if this is an IPv4 address, with optional ranges and wildcards. */
    // 解析 IPv4 地址，包括可选的范围和通配符
    if (parse_ipv4_ranges(elem, local_spec)) {
        // 如果子网掩码大于 32，则输出错误信息并释放内存后返回 0
        if (netmask_bits > 32) {
            log_user("Illegal netmask in \"%s\". Must be between 0 and 32.\n", spec);
            free(local_spec);
            free(elem);
            return 0;
        }
        // 应用子网掩码
        apply_ipv4_netmask_bits(elem, netmask_bits);
        // 输出调试信息
        log_debug("Add IPv4 range %s/%ld to addrset.\n", local_spec, netmask_bits > 0 ? netmask_bits : 32);
        // 将元素插入到链表头部
        elem->next = set->head;
        set->head = elem;
        // 释放内存
        free(local_spec);
        return 1;
    } else {
        // 释放内存
        free(elem);
    }

    /* When all else fails, resolve the name. */
    // 解析名称并获取地址信息
    rc = resolve_name(local_spec, &addrs, af, dns);
    if (rc != 0) {
        // 如果解析失败，则输出错误信息并释放内存后返回 0
        log_user("Error resolving name \"%s\": %s\n", local_spec, gai_strerror(rc));
        free(local_spec);
        return 0;
    }
    if (addrs == NULL)
        // 如果地址信息为空，则输出警告信息
        log_user("Warning: no addresses found for %s.\n", local_spec);
    // 释放内存
    free(local_spec);

    /* Walk the list of addresses and add them all to the set with netmasks. */
    # 遍历地址链表，处理每个地址
    for (addr = addrs; addr != NULL; addr = addr->ai_next) {
        # 用于存储地址的字符串，预留 128 字节
        char addr_string[128];

        # 将地址转换为字符串形式
        address_to_string(addr->ai_addr, addr->ai_addrlen, addr_string, sizeof(addr_string));

        # 注意：在这个循环中，我们可能处理多个地址族的地址（例如 IPv4 和 IPv6）。但是我们最多只有一个网络掩码值。无论我们有什么网络掩码，它都会盲目地应用于所有地址，这可能不是您希望的，如果将 /24 应用于 IPv6 将会导致错误，如果将 /120 应用于 IPv4 也会导致错误。
        if (addr->ai_family == AF_INET) {

            # 如果网络掩码位数大于 32，则打印错误信息并返回
            if (netmask_bits > 32) {
                log_user("Illegal netmask in \"%s\". Must be between 0 and 32.\n", spec);
                freeaddrinfo(addrs);
                return 0;
            }
            # 打印调试信息，将 IPv4 地址和网络掩码位数添加到地址集合（trie）中
            log_debug("Add IPv4 %s/%ld to addrset (trie).\n", addr_string, netmask_bits > 0 ? netmask_bits : 32);
#ifdef HAVE_IPV6
        // 如果支持 IPv6
        } else if (addr->ai_family == AF_INET6) {
            // 如果地址族是 IPv6
            if (netmask_bits > 128) {
                // 如果子网掩码大于128
                log_user("Illegal netmask in \"%s\". Must be between 0 and 128.\n", spec);
                // 记录错误信息
                freeaddrinfo(addrs);
                // 释放地址信息
                return 0;
                // 返回错误
            }
            log_debug("Add IPv6 %s/%ld to addrset (trie).\n", addr_string, netmask_bits > 0 ? netmask_bits : 128);
            // 记录调试信息
#endif
        } else {
            // 如果不是 IPv6 地址
            log_debug("ignoring address %s for %s. Family %d socktype %d protocol %d.\n", addr_string, spec, addr->ai_family, addr->ai_socktype, addr->ai_protocol);
            // 记录忽略的地址信息
            continue;
            // 继续下一次循环
        }

        trie_insert(set->trie, addr->ai_addr, netmask_bits);
        // 将地址和子网掩码插入到 trie 中
    }

    if (addrs != NULL)
        // 如果地址信息不为空
        freeaddrinfo(addrs);
        // 释放地址信息

    return 1;
    // 返回成功
}

/* Add whitespace-separated host specifications from fd into the address set.
   Returns 1 on success, 0 on error. */
int addrset_add_file(struct addrset *set, FILE *fd, int af, int dns)
{
    char buf[1024];
    int c, i;

    for (;;) {
        /* Skip whitespace. */
        // 跳过空白字符
        while ((c = getc(fd)) != EOF) {
            if (!isspace(c))
                break;
        }
        if (c == EOF)
            break;
        ungetc(c, fd);

        i = 0;
        while ((c = getc(fd)) != EOF) {
            if (isspace(c))
                break;
            if (i + 1 > sizeof(buf) - 1) {
                /* Truncate the specification to give a little context. */
                // 截断规范以提供一些上下文
                buf[11] = '\0';
                log_user("Host specification starting with \"%s\" is too long.\n", buf);
                // 记录错误信息
                return 0;
                // 返回错误
            }
            buf[i++] = c;
        }
        buf[i] = '\0';

        if (!addrset_add_spec(set, buf, af, dns))
            return 0;
    }

    return 1;
    // 返回成功
}

/* Parse an IPv4 address with optional ranges and wildcards into bit vectors.
   Each octet must match the regular expression '(\*|#?(-#?)?(,#?(-#?)?)*)',
   where '#' stands for an integer between 0 and 255. Return 1 on success, 0 on
   error. */
# 解析 IPv4 地址范围，将其表示为位向量
static int parse_ipv4_ranges(struct addrset_elem *elem, const char *spec)
{
    const char *p;  # 定义指向 spec 字符串的指针
    int octet_index, i;  # 定义用于迭代的变量

    p = spec;  # 将 spec 字符串的地址赋给指针 p
    octet_index = 0;  # 初始化 octet_index 为 0
    while (*p != '\0' && octet_index < 4) {  # 当指针指向的字符不是结束符且 octet_index 小于 4 时执行循环
        if (*p == '*') {  # 如果当前字符是通配符 *
            for (i = 0; i < 256; i++)  # 遍历 0 到 255
                BIT_SET(elem->ipv4.bits[octet_index], i);  # 将位向量中的每一位都设置为 1
            p++;  # 指针向后移动一位
        } else {  # 如果当前字符不是通配符 *
            for (;;) {  # 无限循环
                long start, end;  # 定义起始和结束值
                const char *tail;  # 定义指向字符串的指针

                errno = 0;  # 清空错误标志
                start = parse_long(p, &tail);  # 解析字符串中的长整型数值
                # 如果解析结果和起始指针相同，表示解析失败
                if (tail == p) {
                    if (*p == '-')  # 如果当前字符是 - 
                        start = 0;  # 将起始值设为 0
                    else
                        return 0;  # 返回解析失败
                }
                # 如果解析出错或者起始值不在 0 到 255 之间，返回解析失败
                if (errno != 0 || start < 0 || start > 255)
                    return 0;
                p = tail;  # 将指针移动到解析后的位置

                # 查找范围
                if (*p == '-') {  # 如果当前字符是 -
                    p++;  # 指针向后移动一位
                    errno = 0;  # 清空错误标志
                    end = parse_long(p, &tail);  # 解析字符串中的长整型数值
                    # 如果解析结果和起始指针相同，表示解析失败
                    if (tail == p)
                        end = 255;  # 将结束值设为 255
                    # 如果解析出错或者结束值不在 0 到 255 之间，或者结束值小于起始值，返回解析失败
                    if (errno != 0 || end < 0 || end > 255 || end < start)
                        return 0;
                    p = tail;  # 将指针移动到解析后的位置
                } else {
                    end = start;  # 如果没有范围，结束值等于起始值
                }

                # 将范围内的值在位向量中设置为 1
                for (i = start; i <= end; i++)
                    BIT_SET(elem->ipv4.bits[octet_index], i);

                if (*p != ',')  # 如果当前字符不是逗号
                    break;  # 退出循环
                p++;  # 指针向后移动一位
            }
        }
        octet_index++;  # octet_index 加一
        if (octet_index < 4) {  # 如果 octet_index 小于 4
            if (*p != '.')  # 如果当前字符不是 .
                return 0;  # 返回解析失败
            p++;  # 指针向后移动一位
        }
    }
    if (*p != '\0' || octet_index < 4)  # 如果当前字符不是结束符或者 octet_index 小于 4
        return 0;  # 返回解析失败

    return 1;  # 返回解析成功
}

/* Expand a single-octet bit vector to include any additional addresses that
   result when mask is applied. */
static void apply_ipv4_netmask_octet(octet_bitvector bits, uint8_t mask)
{
    unsigned int i, j;
    uint32_t chunk_size;

    /* Process the bit vector in chunks, first of size 1, then of size 2, up to
       size 128. Check the next bit of the mask. If it is 1, do nothing.
       Otherwise, pair up the chunks (first with the second, third with the
       fourth, etc.). For each pair of chunks, set a bit in one chunk if it is
       set in the other. chunk_size also serves as an index into the mask. */
    for (chunk_size = 1; chunk_size < 256; chunk_size <<= 1) {
        if ((mask & chunk_size) != 0)
            continue;
        for (i = 0; i < 256; i += chunk_size * 2) {
            for (j = 0; j < chunk_size; j++) {
                if (BIT_IS_SET(bits, i + j))
                    BIT_SET(bits, i + j + chunk_size);
                else if (BIT_IS_SET(bits, i + j + chunk_size))
                    BIT_SET(bits, i + j);
            }
        }
    }
}

/* Expand an addrset_elem's IPv4 bit vectors to include any additional addresses
   that result when the given netmask is applied. The mask is in network byte
   order. */
static void apply_ipv4_netmask(struct addrset_elem *elem, uint32_t mask)
{
    mask = ntohl(mask);
    /* Apply the mask one octet at a time. It's done this way because ranges
       span exactly one octet. */
    apply_ipv4_netmask_octet(elem->ipv4.bits[0], (mask & 0xFF000000) >> 24);
    apply_ipv4_netmask_octet(elem->ipv4.bits[1], (mask & 0x00FF0000) >> 16);
    apply_ipv4_netmask_octet(elem->ipv4.bits[2], (mask & 0x0000FF00) >> 8);
    apply_ipv4_netmask_octet(elem->ipv4.bits[3], (mask & 0x000000FF));
}

/* Expand an addrset_elem's IPv4 bit vectors to include any additional addresses
   that result from the application of a CIDR-style netmask with the given
   number of bits. If bits is negative it is taken to be 32. */
static void apply_ipv4_netmask_bits(struct addrset_elem *elem, int bits)
{
    uint32_t mask;
    # 如果位数大于32，则直接返回，不进行后续操作
    if (bits > 32)
        return;
    # 如果位数小于0，则将位数设置为32
    if (bits < 0)
        bits = 32;

    # 如果位数为0，则将掩码设置为全0
    if (bits == 0)
        mask = htonl(0x00000000);
    # 否则，根据位数计算掩码并进行网络字节序转换
    else
        mask = htonl(0xFFFFFFFF << (32 - bits));
    # 应用IPv4网络掩码
    apply_ipv4_netmask(elem, mask);
static int match_ipv4_bits(const octet_bitvector bits[4], const struct sockaddr *sa)
{
    uint8_t octets[4];

    // 检查传入的地址类型是否为 IPv4
    if (sa->sa_family != AF_INET)
        return 0;

    // 将 IPv4 地址转换为 4 个字节的数组
    in_addr_to_octets(&((const struct sockaddr_in *) sa)->sin_addr, octets);

    // 检查传入的地址是否匹配指定的位向量
    return BIT_IS_SET(bits[0], octets[0])
        && BIT_IS_SET(bits[1], octets[1])
        && BIT_IS_SET(bits[2], octets[2])
        && BIT_IS_SET(bits[3], octets[3]);
}

static int addrset_elem_match(const struct addrset_elem *elem, const struct sockaddr *sa)
{
  // 调用 match_ipv4_bits 函数检查地址是否匹配
  return match_ipv4_bits(elem->ipv4.bits, sa);
}

int addrset_contains(const struct addrset *set, const struct sockaddr *sa)
{
    struct addrset_elem *elem;

    /* First check the trie. */
    // 首先检查 trie 数据结构中是否包含指定地址
    if (trie_match(set->trie, sa))
      return 1;

    /* If that didn't match, check the rest of the addrset_elem in order */
    // 如果 trie 数据结构中没有匹配的地址，则按顺序检查 addrset_elem 中的地址
    if (sa->sa_family == AF_INET) {
      for (elem = set->head; elem != NULL; elem = elem->next) {
        // 调用 addrset_elem_match 函数检查地址是否匹配
        if (addrset_elem_match(elem, sa))
          return 1;
      }
    }

    // 如果没有匹配的地址，则返回 0
    return 0;
}
```