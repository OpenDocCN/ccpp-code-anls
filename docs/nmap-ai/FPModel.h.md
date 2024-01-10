# `nmap\FPModel.h`

```
// 如果 _FPMODEL_H_ 未定义，则定义 _FPMODEL_H_
#ifndef _FPMODEL_H_
#define _FPMODEL_H_

// 声明一个名为 FPModel 的结构体变量
extern struct model FPModel;
// 声明一个二维数组 FPscale，包含两列
extern double FPscale[][2];
// 声明一个二维数组 FPmean，包含 695 列
extern double FPmean[][695];
// 声明一个二维数组 FPvariance，包含 695 列
extern double FPvariance[][695];
// 声明一个名为 FPmatches 的 FingerMatch 结构体数组
extern FingerMatch FPmatches[];

// 结束条件，防止重复定义
#endif
```