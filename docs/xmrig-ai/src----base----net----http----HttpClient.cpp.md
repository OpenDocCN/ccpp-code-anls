# `xmrig\src\base\net\http\HttpClient.cpp`

```cpp
/* XMRig
 * 版权所有 (c) 2014-2019 heapwolf    <https://github.com/heapwolf>
 * 版权所有 (c) 2018-2021 SChernykh   <https://github.com/SChernykh>
 * 版权所有 (c) 2016-2021 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以重新发布或修改
 *   根据由自由软件基金会发布的 GNU 通用公共许可证的条款，版本为 3 或
 *   （根据您的选择）任何以后的版本。
 *
 *   本程序是希望它有用，
 *   但没有任何保证；甚至没有暗示的保证适用于特定用途。请参阅
 *   GNU 通用公共许可证以获取更多详细信息。
 *
 *   您应该已经收到 GNU 通用公共许可证的副本
 *   与本程序一起。如果没有，请参见 <http://www.gnu.org/licenses/>。
 */


#include "base/net/http/HttpClient.h"
#include "3rdparty/llhttp/llhttp.h"
#include "base/io/log/Log.h"
#include "base/kernel/Platform.h"
#include "base/net/dns/Dns.h"
#include "base/net/dns/DnsRecords.h"
#include "base/net/tools/NetBuffer.h"
#include "base/tools/Timer.h"


#include <sstream>
#include <uv.h>


namespace xmrig {


static const char *kCRLF = "\r\n";


} // namespace xmrig


// 构造函数，初始化 HttpClient 对象
xmrig::HttpClient::HttpClient(const char *tag, FetchRequest &&req, const std::weak_ptr<IHttpListener> &listener) :
    HttpContext(HTTP_RESPONSE, listener),
    m_tag(tag),
    m_req(std::move(req))
{
    method  = m_req.method; // 设置请求方法
    url     = std::move(m_req.path); // 设置请求路径
    body    = std::move(m_req.body); // 设置请求体
    headers = std::move(m_req.headers); // 设置请求头

    if (m_req.timeout) {
        m_timer = std::make_shared<Timer>(this, m_req.timeout, 0); // 如果设置了超时时间，创建定时器
    }
}


// 连接到服务器
bool xmrig::HttpClient::connect()
{
    m_dns = Dns::resolve(m_req.host, this); // 解析主机名

    return true;
}


// 解析完成回调函数
void xmrig::HttpClient::onResolved(const DnsRecords &records, int status, const char *error)
{
    this->status = status; // 设置状态
    m_dns.reset(); // 重置 DNS 解析结果
    # 如果状态小于0并且记录为空，则执行以下操作
    if (status < 0 && records.isEmpty()) {
        # 如果不是静默模式，则打印错误日志
        if (!isQuiet()) {
            LOG_ERR("%s " RED("DNS error: ") RED_BOLD("\"%s\""), tag(), error);
        }
        # 返回空值
        return;
    }

    # 创建一个新的 uv_connect_t 对象
    auto req  = new uv_connect_t;
    # 将当前对象的引用赋给 req 的 data 属性
    req->data = this;

    # 使用 TCP 连接到指定地址和端口
    uv_tcp_connect(req, m_tcp, records.get().addr(port()), onConnect);
# 定义了一个名为onTimer的函数，参数为Timer指针
void xmrig::HttpClient::onTimer(const Timer *)
{
    # 调用close函数，传入UV_ETIMEDOUT作为参数
    close(UV_ETIMEDOUT);
}

# 定义了一个名为handshake的函数
void xmrig::HttpClient::handshake()
{
    # 向headers中插入键值对 "Host" 和当前host的数值
    headers.insert({ "Host",       host() });
    # 向headers中插入键值对 "Connection" 和 "close"
    headers.insert({ "Connection", "close" });
    # 向headers中插入键值对 "User-Agent" 和当前平台的用户代理数据
    headers.insert({ "User-Agent", Platform::userAgent().data() });

    # 如果body不为空
    if (!body.empty()) {
        # 向headers中插入键值对 "Content-Length" 和body的大小转换成字符串
        headers.insert({ "Content-Length", std::to_string(body.size()) });
    }

    # 创建一个stringstream对象ss
    std::stringstream ss;
    # 将请求方法、url和HTTP版本号写入stringstream对象ss
    ss << llhttp_method_name(static_cast<llhttp_method>(method)) << " " << url << " HTTP/1.1" << kCRLF;

    # 遍历headers，将键值对写入stringstream对象ss
    for (auto &header : headers) {
        ss << header.first << ": " << header.second << kCRLF;
    }

    # 将headers清空
    headers.clear();

    # 在body的开头插入stringstream对象ss的内容
    body.insert(0, ss.str());
    # 调用write函数，传入body和false作为参数
    write(std::move(body), false);
}

# 定义了一个名为read的函数，参数为const char指针和size_t类型的size
void xmrig::HttpClient::read(const char *data, size_t size)
{
    # 如果解析失败
    if (!parse(data, size)) {
        # 调用close函数，传入UV_EPROTO作为参数
        close(UV_EPROTO);
    }
}

# 定义了一个名为onConnect的函数，参数为uv_connect_t指针和整型status
void xmrig::HttpClient::onConnect(uv_connect_t *req, int status)
{
    # 将req的data转换为HttpClient指针，并赋值给client
    auto client = static_cast<HttpClient *>(req->data);
    # 删除req

    # 如果client为空，直接返回
    if (!client) {
        return;
    }

    # 如果status小于0
    if (status < 0) {
        # 如果status等于UV_ECANCELED，将status赋值为UV_ETIMEDOUT
        if (status == UV_ECANCELED) {
            status = UV_ETIMEDOUT;
        }

        # 如果client不是静默模式，输出错误日志
        if (!client->isQuiet()) {
            LOG_ERR("%s " RED("connect error: ") RED_BOLD("\"%s\""), client->tag(), uv_strerror(status));
        }

        # 调用close函数，传入status作为参数
        return client->close(status);
    }

    # 开始从client的流中读取数据
    uv_read_start(client->stream(), NetBuffer::onAlloc,
        [](uv_stream_t *tcp, ssize_t nread, const uv_buf_t *buf)
        {
            auto client = static_cast<HttpClient*>(tcp->data);

            # 如果nread大于等于0
            if (nread >= 0) {
                # 调用read函数，传入buf->base和nread作为参数
                client->read(buf->base, static_cast<size_t>(nread));
            } else {
                # 如果不是静默模式且nread不等于UV_EOF，输出错误日志
                if (!client->isQuiet() && nread != UV_EOF) {
                    LOG_ERR("%s " RED("read error: ") RED_BOLD("\"%s\""), client->tag(), uv_strerror(static_cast<int>(nread)));
                }

                # 调用close函数，传入nread作为参数
                client->close(static_cast<int>(nread));
            }

            # 释放buf
            NetBuffer::release(buf);
        });
}
    # 客户端发起握手请求
    client->handshake();
# 闭合前面的函数定义
```