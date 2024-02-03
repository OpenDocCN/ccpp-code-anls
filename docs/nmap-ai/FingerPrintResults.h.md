# `nmap\FingerPrintResults.h`

```cpp
#ifndef FINGERPRINTRESULTS_H
#define FINGERPRINTRESULTS_H

class FingerPrintResults;

#include "FPEngine.h"
#include "osscan.h"
#include "charpool.h"

/* Maximum number of results allowed in one of these things ... */
#define MAX_FP_RESULTS 36

struct OS_Classification_Results {
  struct OS_Classification *OSC[MAX_FP_RESULTS];  // 结构体数组，存储最大结果数的 OS_Classification 指针
  double OSC_Accuracy[MAX_FP_RESULTS];  // 存储每个 OS_Classification 的准确度
  int OSC_num_perfect_matches; // 完全匹配的 OSC 数量
  int OSC_num_matches; // 总匹配的 OSC 数量
  int overall_results; /* OSSCAN_TOOMANYMATCHES, OSSCAN_NOMATCHES, OSSCAN_SUCCESS, etc */  // 总体结果状态
};

/* If the fingerprint is of potentially poor quality, we don't want to
   print it and ask the user to submit it.  In that case, the reason
   for skipping the FP is returned as a static string.  If the FP is
   great and should be printed, NULL is returned. */
  virtual const char *OmitSubmissionFP();  // 如果指纹质量可能较差，则返回跳过的原因；如果指纹很好并应该打印，则返回 NULL

  virtual const char *merge_fpr(const Target *currenths, bool isGoodFP, bool wrapit) const = 0;  // 合并指纹结果

 private:
  bool isClassified; // Whether populateClassification() has been called  // 是否已调用 populateClassification()
  /* Goes through fingerprinting results to populate OSR */  // 遍历指纹结果以填充 OSR

  void populateClassification();  // 填充分类结果
  bool classAlreadyExistsInResults(struct OS_Classification *OSC);  // 检查分类结果是否已存在于结果中
  struct OS_Classification_Results OSR;  // 存储操作系统分类结果
  CharPool *cp; /* Holds small strings allocated for the life of this object */  // 保存为此对象分配的小字符串

};

class FingerPrintResultsIPv4 : public FingerPrintResults {
public:
  FingerPrint **FPs; /* Fingerprint data obtained from host */  // 从主机获取的指纹数据
  int numFPs;  // 指纹数量

  FingerPrintResultsIPv4();  // 构造函数
  virtual ~FingerPrintResultsIPv4();  // 虚析构函数
  const char *merge_fpr(const Target *currenths, bool isGoodFP, bool wrapit) const;  // 合并指纹结果
};

class FingerPrintResultsIPv6 : public FingerPrintResults {
// 声明一个包含指针的数组，用于存储 IPv6 地址的指纹响应
public:
  FPResponse *fp_responses[NUM_FP_PROBES_IPv6];
  // 记录开始时间的结构体
  struct timeval begin_time;
  /* 我们在发送数据包中设置的流标签，用于后续计算偏移量 */
  unsigned int flow_label;

  // 构造函数，用于初始化 FingerPrintResultsIPv6 对象
  FingerPrintResultsIPv6();
  // 虚析构函数，用于释放 FingerPrintResultsIPv6 对象的资源
  virtual ~FingerPrintResultsIPv6();
  // 返回一个指向常量字符的指针，表示忽略提交的指纹
  const char *OmitSubmissionFP();
  // 返回一个指向常量字符的指针，表示合并指纹结果
  const char *merge_fpr(const Target *currenths, bool isGoodFP, bool wrapit) const;
};

#endif /* FINGERPRINTRESULTS_H */
```