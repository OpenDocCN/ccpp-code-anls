# `xmrig\src\backend\opencl\OclCache.h`

```cpp
/* XMRig
 * 版权所有 (c) 2018-2021 SChernykh   <https://github.com/SChernykh>
 * 版权所有 (c) 2016-2021 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以重新发布或修改它
 *   根据由自由软件基金会发布的 GNU 通用公共许可证的条款，
 *   无论是许可证的第 3 版还是（在您的选择下）任何以后的版本。
 *
 *   本程序是希望它有用的分发，
 *   但没有任何保证；甚至没有对适销性或特定用途的隐含保证。请参阅
 *   GNU 通用公共许可证以获取更多详细信息。
 *
 *   您应该已经收到了 GNU 通用公共许可证的副本
 *   与本程序一起。如果没有，请参阅 <http://www.gnu.org/licenses/>。
 */

#ifndef XMRIG_OCLCACHE_H
#define XMRIG_OCLCACHE_H


#include <string>


使用指针别名定义 cl_program 类型
using cl_program = struct _cl_program *;


在 xmrig 命名空间中定义 IOclRunner 类
namespace xmrig {


定义 OclCache 类
class OclCache
{
public:
    静态方法，用于构建 OpenCL 程序
    static cl_program build(const IOclRunner *runner);
    静态方法，用于生成缓存键值
    static std::string cacheKey(const char *deviceKey, const char *options, const char *source);
    静态方法，用于生成缓存键值
    static std::string cacheKey(const IOclRunner *runner);

private:
    静态方法，用于生成缓存键值的前缀
    static std::string prefix();
    静态方法，用于创建目录
    static void createDirectory();
    静态方法，用于保存程序到文件
    static void save(cl_program program, const std::string &fileName);
};


} // namespace xmrig


#endif /* XMRIG_OCLCACHE_H */
```