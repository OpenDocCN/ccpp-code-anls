# `whisper.cpp\examples\helpers.js`

```cpp
// 定义一个函数，将 TypedArray 转换为指定类型的数组
function convertTypedArray(src, type) {
    // 创建一个与源数组相同长度的 ArrayBuffer
    var buffer = new ArrayBuffer(src.byteLength);
    // 将源数组的内容复制到新的 ArrayBuffer 中
    var baseView = new src.constructor(buffer).set(src);
    // 根据指定类型创建新的数组并返回
    return new type(buffer);
}

// 定义一个函数，用于在页面上输出文本信息
var printTextarea = (function() {
    // 获取页面上的输出文本框元素
    var element = document.getElementById('output');
    // 如果元素存在，则清空其内容（清除浏览器缓存）
    if (element) element.value = '';
    // 返回一个函数，用于输出文本信息
    return function(text) {
        // 如果传入多个参数，则将它们连接成一个字符串
        if (arguments.length > 1) text = Array.prototype.slice.call(arguments).join(' ');
        // 在控制台输出文本信息
        console.log(text);
        // 如果元素存在，则在文本框中追加文本并滚动到底部
        if (element) {
            element.value += text + "\n";
            element.scrollTop = element.scrollHeight;
        }
    };
})();

// 异步函数，用于清除缓存
async function clearCache() {
    // 弹出确认框，确认是否清除缓存
    if (confirm('Are you sure you want to clear the cache?\nAll the models will be downloaded again.')) {
        // 删除 indexedDB 中的数据库
        indexedDB.deleteDatabase(dbName);
        // 重新加载页面
        location.reload();
    }
}

// 使用 Fetch API 从远程 URL 获取文件
async function fetchRemote(url, cbProgress, cbPrint) {
    // 输出下载信息
    cbPrint('fetchRemote: downloading with fetch()...');

    // 发起 GET 请求获取远程文件
    const response = await fetch(
        url,
        {
            method: 'GET',
            headers: {
                'Content-Type': 'application/octet-stream',
            },
        }
    );

    // 如果请求失败，则输出错误信息
    if (!response.ok) {
        cbPrint('fetchRemote: failed to fetch ' + url);
        return;
    }

    // 获取响应头中的 content-length，用于计算下载进度
    const contentLength = response.headers.get('content-length');
    const total = parseInt(contentLength, 10);
    const reader = response.body.getReader();

    // 初始化变量用于存储下载数据和计算下载进度
    var chunks = [];
    var receivedLength = 0;
    var progressLast = -1;
}
    # 循环读取数据流
    while (true) {
        # 从数据流中读取数据
        const { done, value } = await reader.read();

        # 如果读取完成，跳出循环
        if (done) {
            break;
        }

        # 将读取的数据块添加到数组中
        chunks.push(value);
        # 更新已接收数据的长度
        receivedLength += value.length;

        # 如果有内容长度信息
        if (contentLength) {
            # 调用进度回调函数，传入接收到的数据比例
            cbProgress(receivedLength/total);

            # 计算当前进度百分比
            var progressCur = Math.round((receivedLength / total) * 10);
            # 如果当前进度与上次不同
            if (progressCur != progressLast) {
                # 调用打印回调函数，显示当前进度
                cbPrint('fetchRemote: fetching ' + 10*progressCur + '% ...');
                # 更新上次进度
                progressLast = progressCur;
            }
        }
    }

    # 初始化位置变量
    var position = 0;
    # 创建一个包含所有接收数据的 Uint8Array
    var chunksAll = new Uint8Array(receivedLength);

    # 遍历所有数据块
    for (var chunk of chunks) {
        # 将每个数据块拷贝到总数据数组中
        chunksAll.set(chunk, position);
        # 更新位置
        position += chunk.length;
    }

    # 返回包含所有接收数据的 Uint8Array
    return chunksAll;
// 结束函数定义
}

// 加载远程数据
// - 检查数据是否已经存在于 IndexedDB 中
// - 如果不存在，则从远程 URL 获取数据并存储到 IndexedDB 中
function loadRemote(url, dst, size_mb, cbProgress, cbReady, cbCancel, cbPrint) {
    // 检查浏览器是否支持 navigator.storage.estimate 方法
    if (!navigator.storage || !navigator.storage.estimate) {
        // 打印错误信息
        cbPrint('loadRemote: navigator.storage.estimate() is not supported');
    } else {
        // 查询存储配额并打印
        navigator.storage.estimate().then(function (estimate) {
            cbPrint('loadRemote: storage quota: ' + estimate.quota + ' bytes');
            cbPrint('loadRemote: storage usage: ' + estimate.usage + ' bytes');
        });
    }

    // 检查数据是否已经存在于 IndexedDB 中
    var rq = indexedDB.open(dbName, dbVersion);

    rq.onupgradeneeded = function (event) {
        var db = event.target.result;
        if (db.version == 1) {
            var os = db.createObjectStore('models', { autoIncrement: false });
            cbPrint('loadRemote: created IndexedDB ' + db.name + ' version ' + db.version);
        } else {
            // 清空数据库
            var os = event.currentTarget.transaction.objectStore('models');
            os.clear();
            cbPrint('loadRemote: cleared IndexedDB ' + db.name + ' version ' + db.version);
        }
    };

    // 处理打开 IndexedDB 失败的情况
    rq.onerror = function (event) {
        cbPrint('loadRemote: failed to open IndexedDB');
        cbCancel();
    };

    // 处理 IndexedDB 被阻塞的情况
    rq.onblocked = function (event) {
        cbPrint('loadRemote: failed to open IndexedDB: blocked');
        cbCancel();
    };

    // 处理打开 IndexedDB 被中止的情况
    rq.onabort = function (event) {
        cbPrint('loadRemote: failed to open IndexedDB: abort');
        cbCancel();
    };
}
```