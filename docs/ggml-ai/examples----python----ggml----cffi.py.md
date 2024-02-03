# `ggml\examples\python\ggml\cffi.py`

```cpp
# 导入_cffi_backend模块，用于与C语言进行交互
import _cffi_backend
# 创建FFI对象，用于定义C语言接口
ffi = _cffi_backend.FFI('ggml.cffi',
    _version = 0x2601,
)
```