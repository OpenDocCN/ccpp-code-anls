# `xmrig\src\base\net\tools\NetBuffer.cpp`

```cpp
/* XMRig
 * 版权所有（c）2018-2020 SChernykh   <https://github.com/SChernykh>
 * 版权所有（c）2016-2020 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以重新发布或修改
 *   根据由自由软件基金会发布的GNU通用公共许可证的条款，
 *   许可证的版本为3或（根据您的选择）任何以后的版本。
 *
 *   本程序是希望它有用，
 *   但没有任何保证；甚至没有暗示的保证适用于特定用途。请参阅
 *   GNU通用公共许可证以获取更多详细信息。
 *
 *   您应该已经收到GNU通用公共许可证的副本
 *   与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */


#include "base/net/tools/NetBuffer.h"
#include "base/kernel/constants.h"
#include "base/net/tools/MemPool.h"


#include <cassert>
#include <uv.h>


namespace xmrig {


static MemPool<XMRIG_NET_BUFFER_CHUNK_SIZE, XMRIG_NET_BUFFER_INIT_CHUNKS> *pool = nullptr;


inline MemPool<XMRIG_NET_BUFFER_CHUNK_SIZE, XMRIG_NET_BUFFER_INIT_CHUNKS> *getPool()
{
    if (!pool) {
        pool = new MemPool<XMRIG_NET_BUFFER_CHUNK_SIZE, XMRIG_NET_BUFFER_INIT_CHUNKS>();
    }

    return pool;
}


} // namespace xmrig


char *xmrig::NetBuffer::allocate()
{
    return getPool()->allocate();
}


void xmrig::NetBuffer::destroy()
{
    if (!pool) {
        return;
    }

    assert(pool->freeSize() == pool->size());

    delete pool;
    pool = nullptr;
}


void xmrig::NetBuffer::onAlloc(uv_handle_t *, size_t, uv_buf_t *buf)
{
    buf->base = getPool()->allocate();
    buf->len  = XMRIG_NET_BUFFER_CHUNK_SIZE;
}


void xmrig::NetBuffer::release(const char *buf)
{
    if (buf == nullptr) {
        return;
    }

    getPool()->deallocate(buf);
}


void xmrig::NetBuffer::release(const uv_buf_t *buf)
{
    if (buf->base == nullptr) {
        return;
    }
    # 调用getPool()函数获取内存池，并释放buf->base指向的内存块
    getPool()->deallocate(buf->base);
# 闭合前面的函数定义
```