# `xmrig\src\base\io\log\FileLogWriter.cpp`

```
/* XMRig
 * 版权所有 (c) 2018-2021 SChernykh   <https://github.com/SChernykh>
 * 版权所有 (c) 2016-2021 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以重新发布或修改
 *   根据 GNU 通用公共许可证的条款发布，由
 *   自由软件基金会发布的版本3或
 *   （在您的选择下）任何以后的版本。
 *
 *   本程序是希望它有用，
 *   但没有任何保证；甚至没有暗示的保证
 *   适销性或特定用途的保证。请参阅
 *   GNU 通用公共许可证以获取更多详细信息。
 *
 *   您应该已经收到了 GNU 通用公共许可证的副本
 *   与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */


#include "base/io/log/FileLogWriter.h"
#include "base/io/Env.h"


#include <cassert>
#include <uv.h>


namespace xmrig {


static void fsWriteCallback(uv_fs_t *req)
{
    delete [] static_cast<char *>(req->data);

    uv_fs_req_cleanup(req);
    delete req;
}


} // namespace xmrig


bool xmrig::FileLogWriter::open(const char *fileName)
{
    assert(fileName != nullptr);
    if (!fileName) {
        return false;
    }

    uv_fs_t req{};
    const auto path = Env::expand(fileName);
    m_file          = uv_fs_open(uv_default_loop(), &req, path, O_CREAT | O_WRONLY, 0644, nullptr);

    if (req.result < 0 || !isOpen()) {
        uv_fs_req_cleanup(&req);
        m_file = -1;

        return false;
    }

    uv_fs_req_cleanup(&req);

    uv_fs_stat(uv_default_loop(), &req, path, nullptr);
    m_pos = req.statbuf.st_size;
    uv_fs_req_cleanup(&req);

    return true;
}


bool xmrig::FileLogWriter::write(const char *data, size_t size)
{
    if (!isOpen()) {
        return false;
    }

    uv_buf_t buf = uv_buf_init(new char[size], size);  // 初始化一个大小为size的缓冲区
    memcpy(buf.base, data, size);  // 将数据复制到缓冲区中

    auto req = new uv_fs_t;  // 创建一个新的文件系统请求对象
    req->data = buf.base;  // 将缓冲区的地址赋值给请求对象的数据字段
    # 使用 libuv 的文件系统写入函数将数据写入文件
    uv_fs_write(uv_default_loop(), req, m_file, &buf, 1, m_pos, fsWriteCallback);
    # 更新文件写入位置
    m_pos += size;
    # 返回 true，表示写入操作成功
    return true;
# 实现FileLogWriter类的writeLine方法，用于向文件中写入一行数据
bool xmrig::FileLogWriter::writeLine(const char *data, size_t size)
{
    # 创建一个包含两个缓冲区的数组，用于存储数据和换行符
    const uv_buf_t buf[2] = {
        uv_buf_init(new char[size], size),  # 初始化第一个缓冲区，用于存储数据
        uv_buf_init(const_cast<char *>(m_endl), sizeof(m_endl) - 1)  # 初始化第二个缓冲区，用于存储换行符
    };

    # 将数据复制到第一个缓冲区
    memcpy(buf[0].base, data, size);

    # 创建一个新的文件系统请求对象
    auto req = new uv_fs_t;
    req->data = buf[0].base;  # 将第一个缓冲区的数据作为请求对象的数据

    # 使用libuv库的文件系统写入函数将数据写入文件
    uv_fs_write(uv_default_loop(), req, m_file, buf, 2, m_pos, fsWriteCallback);
    m_pos += (buf[0].len + buf[1].len);  # 更新文件写入位置

    return true;  # 返回写入成功
}
```