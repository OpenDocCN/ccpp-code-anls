# `nmap\NmapOutputTable.h`

```cpp
/* $Id$ */

#ifndef NMAPOUTPUTTABLE_H
#define NMAPOUTPUTTABLE_H

/* 为了安全起见，保持 assert() 的定义 */
#undef NDEBUG
#include <assert.h>

#include "nbase.h" /* __attribute__ */

/**********************  DEFINES/ENUMS ***********************************/

/**********************  STRUCTURES  ***********************************/

/**********************  CLASSES     ***********************************/

// 定义 NmapOutputTableCell 结构体
struct NmapOutputTableCell {
  char *str; // 字符串指针
  int strlength; // 字符串长度
  bool weAllocated; // 如果我们分配了 str，那么我们必须释放它
  bool fullrow; // 是否是完整的一行
};
class NmapOutputTable {
 public:
  // 创建给定维度的表格。在调用 printableTable() 时，任何完全空白的行将被移除。
  // 如果表格行数未知，则应指定可能的最大行数。
  NmapOutputTable(int nrows, int ncols);
  ~NmapOutputTable();

  // 添加项目到指定的行和列。如果 copy 为 true，则必须复制项目。否则，我们将只保存指针（在销毁表格之前不要释放它）。
  // 如果不知道 itemlen，则跳过 itemlen 参数（函数将使用 strlen）。
  void addItem(unsigned int row, unsigned int column, bool copy, const char *item, int itemlen = -1);
  // 如果 fullrow 为 true，则 'item' 跨越所有列。跨越从列参数开始（即 0 将是第一列）。
  void addItem(unsigned int row, unsigned int column, bool fullrow, bool copy, const char *item, int itemlen = -1);

  // 类似于 addItem，但此版本采用 printf 样式的格式字符串，后跟可变参数
  void addItemFormatted(unsigned int row, unsigned int column, bool fullrow, const char *fmt, ...)
          __attribute__ ((format (printf, 5, 6))); // 偏移 1 以考虑隐式的 "this" 参数。

  // 此函数将整个表格放入字符缓冲区。
  // 请注意，如果再次调用该函数，缓冲区可能会被重用，并且如果释放表格，则缓冲区也将无效。
  // 如果 size 不为 NULL，则它将被填充为 ASCII 表格的字节大小（不包括终止的 NUL）。
  // 所有空白行将从返回的字符串中移除
  char *printableTable(int *size);

 private:

  bool emptyRow(unsigned int nrow);
  // 表格，压缩为 1D。通过 getCellAddy 访问成员
  struct NmapOutputTableCell *table;
  struct NmapOutputTableCell *getCellAddy(unsigned int row, unsigned int col) {
    assert(row < numRows);  assert(col < numColumns);
    # 返回表格中指定行列的索引
    return table + row * numColumns + col;
  }
  int *maxColLen; // 一个数组，给出每列的任何成员的最大长度（不包括终止符）
                  // （不包括终止符）
  // 一个数组，告诉每行中有效（长度> 0）项目的数量
  int *itemsInRow;
  unsigned int numRows; // 表格的行数
  unsigned int numColumns; // 表格的列数
  char *tableout; // 如果调用 printableTable()，我们将返回这个
  int tableoutsz; // 为 tableout 分配的空间量。包括为 NUL 分配的空间。
// 结束了类定义的大括号
};

// 函数原型声明部分
/**********************  PROTOTYPES  ***********************************/

// 结束了条件编译指令
#endif /* NMAPOUTPUTTABLE_H */
```