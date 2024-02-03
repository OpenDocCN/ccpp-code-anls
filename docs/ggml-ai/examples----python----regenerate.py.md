# `ggml\examples\python\regenerate.py`

```cpp
# 为 ggml 库生成绑定
#
# cffi 需要先对头文件进行 C 预处理，并且它使用 pycparser，对一些东西不太友好
# 所以我们需要做一些处理（例如，用它们的值替换 sizeof 表达式，移除在 Darwin 头文件中找到的奇怪语法）。
import os, sys, re, subprocess
import cffi
from stubs import generate_stubs

# 从环境变量中获取 API 文件名，默认为 'api.h'
API = os.environ.get('API', 'api.h')
# 从环境变量中获取 CC 编译器，默认为 'gcc'
CC = os.environ.get('CC') or 'gcc'
# 从环境变量中获取 C_INCLUDE_DIR，默认为 '../../../llama.cpp'
C_INCLUDE_DIR = os.environ.get('C_INCLUDE_DIR', '../../../llama.cpp')
# 预处理标志
CPPFLAGS = [
    "-I", C_INCLUDE_DIR,
    '-D__fp16=uint16_t',  # pycparser 不支持 __fp16
    '-D__attribute__(x)=',
    '-D_Static_assert(x, m)=',
] + [x for x in os.environ.get('CPPFLAGS', '').split(' ') if x != '']

# 尝试运行预处理命令，获取头文件内容
try: 
    header = subprocess.run([CC, "-E", *CPPFLAGS, API], capture_output=True, text=True, check=True).stdout
except subprocess.CalledProcessError as e: 
    print(f'{e.stderr}\n{e}', file=sys.stderr); 
    raise

# 移除 pycparser 不支持的内容
header = '\n'.join([l for l in header.split('\n') if '__darwin_va_list' not in l]) # pycparser 不支持这个

# 用值替换常量大小表达式（为每个表达式编译并运行一个小型可执行文件，因为为什么不呢）。
# 首先，提取方括号内的内容和任何看起来像 sizeof 调用的东西。
for expr in set(re.findall(f'(?<=\\[)[^\\]]+(?=])|sizeof\\s*\\([^()]+\\)', header)):
    if re.match(r'^(\d+|\s*)$', expr): continue # 跳过常量和空方括号内容
    subprocess.run([CC, "-o", "eval_size_expr", *CPPFLAGS, "-x", "c", "-"], text=True, check=True,
                   input=f'''#include <stdio.h>
                             #include "{API}"
                             int main() {{ printf("%lu", (size_t)({expr})); }}''')
    size = subprocess.run(["./eval_size_expr"], capture_output=True, text=True, check=True).stdout
    print(f'Computed constexpr {expr} = {size}')
    header = header.replace(expr, size)

# 创建 cffi.FFI 对象
ffibuilder = cffi.FFI()
# 定义头文件内容
ffibuilder.cdef(header)
# 设置FFI构建器的源文件和扩展名，这里不编译本地扩展，因为这很快变得复杂
ffibuilder.set_source(f'ggml.cffi', None)
# 编译FFI构建器，打印详细信息
ffibuilder.compile(verbose=True)

# 以写文本模式打开"ggml/__init__.pyi"文件
with open("ggml/__init__.pyi", "wt") as f:
    # 写入生成的存根文件内容到打开的文件中
    f.write(generate_stubs(header))
```