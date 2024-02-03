# `nmap\osscan.h`

```cpp
/* $Id$ */

#ifndef OSSCAN_H
#define OSSCAN_H

#include <nbase.h>
#include <vector>
#include <map>

class Target;
class FingerPrintResultsIPv4;

#define OSSCAN_SUCCESS 0
#define OSSCAN_NOMATCHES -1
#define OSSCAN_TOOMANYMATCHES -2

/* We won't even consider matches with a lower accuracy than this */
#define OSSCAN_GUESS_THRESHOLD 0.85

/* The method used to calculate the Target::distance, included in OS
   fingerprints. */
enum dist_calc_method {
        DIST_METHOD_NONE,
        DIST_METHOD_LOCALHOST,
        DIST_METHOD_DIRECT,
        DIST_METHOD_ICMP,
        DIST_METHOD_TRACEROUTE
};

/**********************  STRUCTURES  ***********************************/

#define NUM_FPTESTS 13
  // T2-T7 and U1 have 11 attributes each
#define FP_MAX_TEST_ATTRS 11
  // RIPCK
#define FP_MAX_NAME_LEN 5

// Short alphanumeric strings.
template<u8 _MaxStrLen>
struct ShortStr {
  char str[_MaxStrLen+1];
  bool trunc;
  ShortStr() : trunc(false) {memset(str, 0, sizeof(str));}
  ShortStr(const char *s) { setStr(s); }
  ShortStr(const char *s, const char *e) { setStr(s, e); }
  void setStr(const char *in);
  void setStr(const char *in, const char *end);
  // Helpers for type conversion
  operator const char *() const {return this->str;}
  operator char *() {return this->str;}
  bool operator==(const char *other) const {
    return (!trunc && strncmp(str, other, _MaxStrLen) == 0);
  }
  bool operator==(const ShortStr &other) const {
    return (!trunc && !other.trunc
        && strncmp(str, other.str, _MaxStrLen) == 0);
  }
  bool operator!=(const ShortStr &other) const {
    return (trunc || other.trunc
        || strncmp(str, other.str, _MaxStrLen) != 0);
  }
  bool operator<(const ShortStr &other) const {
    return (trunc < other.trunc || strncmp(str, other.str, _MaxStrLen) < 0);
  }
};

typedef ShortStr<FP_MAX_NAME_LEN> FPstr;

struct Attr {
  FPstr name;
  int points;
  Attr() : name(), points(0) {}
  Attr(const char *n) : name(n), points(0) {}
};
// 定义指纹测试的数据结构
struct FingerTestDef {
  FPstr name; // 指纹测试的名称
  u8 numAttrs; // 属性的数量
  bool hasR; // 是否有 R 属性
  std::map<FPstr, u8> AttrIdx; // 属性名称到索引的映射
  std::vector<Attr> Attrs; // 属性列表

  // 默认构造函数，初始化名称为空，属性数量为 0，没有 R 属性
  FingerTestDef() : name(), numAttrs(0), hasR(false) {}
  // 构造函数，接受名称和属性数组作为参数
  FingerTestDef(const FPstr &n, const char *a[]);
};

// 定义一些宏，用于将枚举类型转换为整数和将整数转换为枚举类型
#define ID2INT(_i) static_cast<int>(_i)
#define INT2ID(_i) static_cast<FingerPrintDef::TestID>(_i)
class FingerPrintDef {
  public:
  // 定义枚举类型 TestID
  enum TestID { SEQ, OPS, WIN, ECN, T1, T2, T3, T4, T5, T6, T7, U1, IE, INVALID };
  // 定义测试属性的字符串数组
  static const char *test_attrs[NUM_FPTESTS][FP_MAX_TEST_ATTRS];
  // 默认构造函数
  FingerPrintDef();
  // 解析测试字符串
  bool parseTestStr(const char *str, const char *end);
  // 获取指定测试 ID 的测试定义
  FingerTestDef &getTestDef(TestID id) { return TestDefs[ID2INT(id)]; }
  // 获取指定测试 ID 的测试定义（常量版本）
  const FingerTestDef &getTestDef(TestID id) const { return TestDefs[ID2INT(id)]; }
  // 获取测试名称对应的测试索引
  int getTestIndex(const FPstr testname) const { return ID2INT(TestIdx.at(testname)); }
  // 将测试名称转换为测试 ID
  TestID str2TestID(const FPstr testname) const { return TestIdx.at(testname); }

  private:
  std::map<FPstr, TestID> TestIdx; // 测试名称到测试 ID 的映射
  std::vector<FingerTestDef> TestDefs; // 测试定义的列表
};

// 定义操作系统分类的数据结构
struct OS_Classification {
  const char *OS_Vendor; // 操作系统厂商
  const char *OS_Family; // 操作系统家族
  const char *OS_Generation; // 操作系统代数，如果未分类则为 NULL
  const char *Device_Type; // 设备类型
  std::vector<const char *> cpe; // CPE（通用平台标识符）列表
};

// 定义指纹匹配的数据结构
struct FingerMatch {
  int line; // 行号
  unsigned short numprints; // 参与分类的指纹数量
  const char *OS_name; // 操作系统名称
  std::vector<OS_Classification> OS_class; // 操作系统分类列表

  // 默认构造函数，初始化行号为 -1，参与分类的指纹数量为 0，操作系统名称为空
  FingerMatch() : line(-1), numprints(0), OS_name(NULL) {}
};

// 定义指纹测试的数据结构
struct FingerTest {
  FingerPrintDef::TestID id; // 测试 ID
  const FingerTestDef *def; // 测试定义指针
  std::vector<const char *> *results; // 测试结果列表指针

  // 默认构造函数，初始化测试 ID 为 INVALID，测试定义指针为空，测试结果列表指针为空
  FingerTest() : id(FingerPrintDef::INVALID), def(NULL), results(NULL) {}
  // 构造函数，接受测试名称和指纹定义作为参数
  FingerTest(const FPstr &testname, const FingerPrintDef &Defs) {
    # 将测试名称转换为测试ID
    id = Defs.str2TestID(testname);
    # 获取测试定义
    def = &Defs.getTestDef(id);
    # 创建一个包含指定数量空指针的字符串向量
    results = new std::vector<const char *>(def->numAttrs, NULL);
  }
  # 根据测试ID和测试定义创建 FingerTest 对象
  FingerTest(FingerPrintDef::TestID testid, const FingerPrintDef &Defs)
    : id(testid), results(NULL) {
      # 获取测试定义
      def = &Defs.getTestDef(id);
      # 创建一个包含指定数量空指针的字符串向量
      results = new std::vector<const char *>(def->numAttrs, NULL);
    }
  # 复制构造函数，用于创建 FingerTest 对象的副本
  FingerTest(const FingerTest &other) : id(other.id), def(other.def), results(other.results) {}
  # 析构函数，用于释放内存
  ~FingerTest() {
    # results 必须手动释放
    }
  # 清除 FingerTest 对象
  void erase();
  # 将字符串转换为属性值
  bool str2AVal(const char *str, const char *end);
  # 设置属性值
  void setAVal(const char *attr, const char *value);
  # 获取属性值
  const char *getAVal(const char *attr) const;
  # 获取属性值名称
  const char *getAValName(u8 index) const;
  # 获取测试名称
  const char *getTestName() const { return def->name.str; }
  # 获取最大分数
  int getMaxPoints() const;
/* Same struct used for reference prints (DB) and observations */
// 定义 FingerPrint 结构体，用于参考打印（DB）和观察
struct FingerPrint {
  FingerMatch match;  // 匹配指纹
  FingerTest tests[NUM_FPTESTS];  // FingerTest 数组
  void erase();  // 清空函数
  void setTest(const FingerTest &test) {  // 设置测试函数
    tests[ID2INT(test.id)] = test;  // 将测试结果存入数组
  }
};

/* SCAN pseudo-test */
// SCAN 伪测试结构体
struct FingerPrintScan {
  enum Attribute { V, E, D, OT, CT, CU, PV, DS, DC, G, M, TM, P, MAX_ATTR };  // 枚举属性
  static const char *attr_names[static_cast<int>(MAX_ATTR)];  // 属性名称数组

  const char *values[static_cast<int>(MAX_ATTR)];  // 值数组
  bool present;  // 是否存在
  FingerPrintScan() : present(false) {memset(values, 0, sizeof(values));}  // 构造函数
  bool parse(const char *str, const char *end);  // 解析函数
  const char *scan2str() const;  // 转换为字符串函数
};

/* An observation parsed from string representation */
// 从字符串表示中解析的观察结构体
struct ObservationPrint {
  FingerPrint fp;  // FingerPrint 对象
  FingerPrintScan scan_info;  // FingerPrintScan 对象
  std::vector<FingerTest> extra_tests;  // 额外测试的向量
  const char *getInfo(FingerPrintScan::Attribute attr) const {  // 获取信息函数
    if (attr >= FingerPrintScan::MAX_ATTR)  // 如果属性大于等于最大属性
      return NULL;  // 返回空
    return scan_info.values[static_cast<int>(attr)];  // 返回对应属性的值
  }
  void mergeTest(const FingerTest &test) {  // 合并测试函数
    FingerTest &ours = fp.tests[ID2INT(test.id)];  // 获取对应测试
    if (ours.id == FingerPrintDef::INVALID)  // 如果测试无效
      ours = test;  // 覆盖测试
    else {
      extra_tests.push_back(test);  // 添加到额外测试中
    }
  }
};

/* This structure contains the important data from the fingerprint
   database (nmap-os-db) */
// 这个结构体包含来自指纹数据库（nmap-os-db）的重要数据
struct FingerPrintDB {
  FingerPrintDef *MatchPoints;  // 匹配点
  std::vector<FingerPrint *> prints;  // FingerPrint 指针的向量

  FingerPrintDB();  // 构造函数
  ~FingerPrintDB();  // 析构函数
};

/**********************  PROTOTYPES  ***********************************/

const char *fp2ascii(const FingerPrint *FP);  // 将 FingerPrint 转换为 ASCII

/* Parses a single fingerprint from the memory region given.  If a
 non-null fingerprint is returned, the user is in charge of freeing it
 when done.  This function does not require the fingerprint to be 100%
 complete since it is used by scripts such as scripts/fingerwatch for
 which some partial fingerprints are OK. */
// 从给定的内存区域解析单个指纹。如果返回一个非空指纹，则用户负责在完成后释放它。此函数不要求指纹是100%完整的，因为它被脚本（如scripts/fingerwatch）使用，其中一些部分指纹是可以的。
# 解析单个指纹，返回观察结果打印
ObservationPrint *parse_single_fingerprint(const FingerPrintDB *DB, const char *fprint);

/* 这些函数接受文件/数据库名称并打开+解析它，返回包含结果的（已分配的）FingerPrintDB。在出现错误的情况下，它们会退出并显示错误消息。 */
FingerPrintDB *parse_fingerprint_file(const char *fname, bool points_only);
FingerPrintDB *parse_fingerprint_reference_file(const char *dbname);

# 释放指纹文件
void free_fingerprint_file(FingerPrintDB *DB);

/* 比较两个指纹--一个参考FP（可以具有表达属性）和一个观察到的指纹（没有表达式）。如果verbose为非零，将打印差异。返回比较准确度（在0和1之间）。MatchPoints是一个特殊的“指纹”，它告诉每个测试有多少点。 */
double compare_fingerprints(const FingerPrint *referenceFP, const FingerPrint *observedFP,
                            const FingerPrintDef *MatchPoints, int verbose, double threshold);

/* 获取指纹并在传入的参考指纹数据库中查找匹配项。结果存储在FPR中（必须指向一个实例化的FingerPrintResultsIPv4类）--结果将按准确度进行逆排序。不包括准确度低于accuracy_threshold的结果。返回的最大匹配数是适合于FingerPrintResultsIPv4类的最大值。 */
void match_fingerprint(const FingerPrint *FP, FingerPrintResultsIPv4 *FPR,
                       const FingerPrintDB *DB, double accuracy_threshold);

/* 如果完美匹配则返回true--如果num_subtests和num_subtests_succeeded不为null，则更新它们。如果shortcircuit为零，则执行所有测试，否则在第一个失败时返回。 */
// 声明一个名为 WriteSInfo 的函数，接受多个参数并且没有返回值
void WriteSInfo(char *ostr, int ostrlen, bool isGoodFP,
                                const char *engine_id,
                                const struct sockaddr_storage *addr, int distance,
                                enum dist_calc_method distance_calculation_method,
                                const u8 *mac, int openTcpPort,
                                int closedTcpPort, int closedUdpPort);

// 声明一个名为 mergeFPs 的函数，接受多个参数并且返回一个指向常量字符的指针
const char *mergeFPs(FingerPrint *FPs[], int numFPs, bool isGoodFP,
                           const struct sockaddr_storage *addr, int distance,
                           enum dist_calc_method distance_calculation_method,
                           const u8 *mac, int openTcpPort, int closedTcpPort,
                           int closedUdpPort, bool wrapit);

// 内部匹配函数的定义，用于测试目的
// 接受两个字符串参数和一个布尔参数，并且有一个默认参数
bool expr_match(const char *val, size_t vlen, const char *expr, size_t explen, bool do_nested=false);
#endif /*OSSCAN_H*/
```