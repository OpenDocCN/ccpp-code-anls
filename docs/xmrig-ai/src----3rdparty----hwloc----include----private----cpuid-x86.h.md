# `xmrig\src\3rdparty\hwloc\include\private\cpuid-x86.h`

```
/*
 * 版权声明
 * 版权所有 © 2010-2012, 2014 波尔多大学
 * 版权所有 © 2010思科系统公司
 * 版权所有 © 2014 Inria
 *
 * 请参阅顶层目录中的 COPYING 文件。
 */

/* x86 的 cpuid 内部实现 */

#ifndef HWLOC_PRIVATE_CPUID_X86_H
#define HWLOC_PRIVATE_CPUID_X86_H

#if (defined HWLOC_X86_32_ARCH) && (!defined HWLOC_HAVE_MSVC_CPUIDEX)
static __hwloc_inline int hwloc_have_x86_cpuid(void)
{
  int ret;
  unsigned tmp, tmp2;
  __asm__(
      "mov $0,%0\n\t"   /* 先假设不支持 */

      "pushfl   \n\t"   /* 保存标志寄存器 */

      "pushfl   \n\t"                                           \
      "pop %1   \n\t"   /* 获取标志寄存器的值 */                         \

#define TRY_TOGGLE                                              \
      "xor $0x00200000,%1\n\t"        /* 尝试切换 ID 位 */    \
      "mov %1,%2\n\t"   /* 保存期望值 */               \
      "push %1  \n\t"                                           \
      "popfl    \n\t"   /* 尝试切换 */                     \
      "pushfl   \n\t"                                           \
      "pop %1   \n\t"                                           \
      "cmp %1,%2\n\t"   /* 与期望值比较 */       \
      "jnz 0f\n\t"   /* 意外情况，失败 */               \

      TRY_TOGGLE        /* 尝试设置/清除 */
      TRY_TOGGLE        /* 尝试清除/设置 */

      "mov $1,%0\n\t"   /* 通过测试！ */

      "0: \n\t"
      "popfl    \n\t"   /* 恢复标志寄存器 */

      : "=r" (ret), "=&r" (tmp), "=&r" (tmp2));
  return ret;
}
#endif /* !defined HWLOC_X86_32_ARCH && !defined HWLOC_HAVE_MSVC_CPUIDEX*/
#if (defined HWLOC_X86_64_ARCH) || (defined HWLOC_HAVE_MSVC_CPUIDEX)
static __hwloc_inline int hwloc_have_x86_cpuid(void) { return 1; }
#endif /* HWLOC_X86_64_ARCH */

static __hwloc_inline void hwloc_x86_cpuid(unsigned *eax, unsigned *ebx, unsigned *ecx, unsigned *edx)
{
#ifdef HWLOC_HAVE_MSVC_CPUIDEX
  // 定义一个整型数组regs，用于存储CPUID指令的返回结果
  int regs[4];
  // 调用CPUID指令，将结果存储到regs数组中
  __cpuidex(regs, *eax, *ecx);
  // 将CPUID指令的返回结果分别存储到eax、ebx、ecx、edx寄存器中
  *eax = regs[0];
  *ebx = regs[1];
  *ecx = regs[2];
  *edx = regs[3];
#else /* HWLOC_HAVE_MSVC_CPUIDEX */
  /* 注意：gcc可能会使用bx或堆栈来寻址%1，所以我们不能使用它们 :/ */
#ifdef HWLOC_X86_64_ARCH
  // 如果不支持MSVC的CPUIDEX，且是64位架构，则使用内联汇编执行CPUID指令
  hwloc_uint64_t sav_rbx;
  __asm__(
  "mov %%rbx,%2\n\t"
  "cpuid\n\t"
  "xchg %2,%%rbx\n\t"
  "movl %k2,%1\n\t"
  : "+a" (*eax), "=m" (*ebx), "=&r"(sav_rbx),
    "+c" (*ecx), "=&d" (*edx));
#elif defined(HWLOC_X86_32_ARCH)
  // 如果不支持MSVC的CPUIDEX，且是32位架构，则使用内联汇编执行CPUID指令
  __asm__(
  "mov %%ebx,%1\n\t"
  "cpuid\n\t"
  "xchg %%ebx,%1\n\t"
  : "+a" (*eax), "=&SD" (*ebx), "+c" (*ecx), "=&d" (*edx));
#else
  // 如果既不支持MSVC的CPUIDEX，也不是64位或32位架构，则报错
#error unknown architecture
#endif
#endif /* HWLOC_HAVE_MSVC_CPUIDEX */
}

#endif /* HWLOC_PRIVATE_X86_CPUID_H */
```