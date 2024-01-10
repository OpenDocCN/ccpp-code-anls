# `ggml\examples\python\api.h`

```
/*
  在这里列出你想在 Python 绑定中公开的所有头文件，
  然后运行 `python regenerate.py`（详细信息请参阅 README.md）
*/

#include "ggml.h"  // 包含 ggml.h 头文件
#include "ggml-metal.h"  // 包含 ggml-metal.h 头文件
#include "ggml-opencl.h"  // 包含 ggml-opencl.h 头文件

// 下面的头文件目前只存在于 llama.cpp 存储库中，如果你没有这些头文件，请将它们注释掉。
#include "k_quants.h"  // 包含 k_quants.h 头文件
#include "ggml-alloc.h"  // 包含 ggml-alloc.h 头文件
#include "ggml-cuda.h"  // 包含 ggml-cuda.h 头文件
#include "ggml-mpi.h"  // 包含 ggml-mpi.h 头文件
```