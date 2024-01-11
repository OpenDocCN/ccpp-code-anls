# `xmrig\src\3rdparty\adl\adl_sdk.h`

```
// 版权声明，版权所有
// MIT 许可证：允许任何人免费获取和使用该软件及相关文档文件，包括使用、复制、修改、合并、发布、分发、再许可和出售等操作
// 要求在所有副本或实质部分中包含上述版权声明和许可声明
// 本软件按原样提供，不提供任何形式的担保，包括但不限于适销性、特定用途适用性和非侵权性的担保。在任何情况下，作者或版权持有人均不对任何索赔、损害或其他责任负责，无论是合同行为、侵权行为还是其他行为，起因于或与本软件或其使用或其他交易有关。

/// \file adl_sdk.h
/// \brief 包含内存分配回调的定义。\n <b>包含在 ADL SDK 中</b>
///
/// \n\n
/// 该文件包含内存分配回调的定义。\n
/// 还包括相应结构和常量的定义。\n
/// <b> 这是在使用 ADL 的 C/C++ 项目中唯一需要包含的头文件 </b>

#ifndef ADL_SDK_H_
#define ADL_SDK_H_

#include "adl_structures.h"

#if defined (LINUX)
#define __stdcall
#endif /* (LINUX) */

/// 内存分配回调
typedef void* ( __stdcall *ADL_MAIN_MALLOC_CALLBACK )( int );


#endif /* ADL_SDK_H_ */
```