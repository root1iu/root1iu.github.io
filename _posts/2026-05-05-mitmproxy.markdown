---
layout: post
title:  "抓包工具 mitmproxy 介绍&实践"
date:   2026-05-05 
categories: Agent
---

在看 `Claude Code` 源码的时候，总想要通过 `Claude Code` 和 `Anthropic` 之间的通信内容来验证源码的内容，所以需要有一种支持 HTTPS 的抓包工具，也就是今天想介绍的：[mitmproxy](https://www.mitmproxy.org/)。

### 原理

MITM (Man-In-The-Middle)，指的是中间人，mitmproxy 就是充当 client（当前语境是`Claude Code`）和 server（当前语境是 `Anthropic`） 之间的中间人，从 client 获取数据包，展示到界面上，之后加密发送给 server。

具体原理图如下：
![原理](https://docs.mitmproxy.org/stable/schematics/how-mitmproxy-works-explicit-https.png)

### 安装
``` bash
$ brew install --cask mitmproxy
```

### 实践
#### 监听
``` bash
# shell1
$ mitmproxy -p 8080
```

#### client
``` bash
# shell2
$ export NODE_EXTRA_CA_CERTS=~/.mitmproxy/mitmproxy-ca-cert.pem
$ export HTTPS_PROXY=http://127.0.0.1:8080
$ claude
```
执行 claude 后，就可以在 mitmproxy 界面看到相关请求：

![mitmproxy界面]({{ site.baseurl }}/assets/images/mitmproxy-front.png)

将其中一个请求展开后，可以看到 https request/response 的明文消息：

![https明文body]({{ site.baseurl }}/assets/images/mitmproxy-https-body.png)

#### 保存数据包

在 mitmproxy 界面使用 w (Save flows) 即可保存 flows 到文件中。如果是使用 mitmweb 也可以在 webui 界面直接下载所有 flows。

#### 读取 flows 数据包

可以写脚本来读取 flows 数据包的内容

```
$ mitmdump -r flows -s export.py -n
# -n 就是 --no-server，不启动监听直接读取 dump 下来的文件
```

其中 `export.py` 就是读取数据包时的 handle 脚本，可以在脚本里做一些简单格式转换，比如转成 `json`，具体脚本内容：

```python
from mitmproxy import http
import hashlib
import json

class CustomExporter:
    def __init__(self):
        self._index = 0

    def response(self, flow: http.HTTPFlow):
        # 跳过回环的包
        if flow.request.host.startswith("127.0.0.1"):
            return

        text = flow.request.text or ""
        digest = hashlib.md5(text.encode("utf-8")).hexdigest()

        # 输出的 json 路径
        out_name = f"{self._index}_{digest}.json"
        self._index += 1

        try:
            req = json.loads(text) if text else None
        except json.JSONDecodeError:
            req = text

        data = {
            "url": flow.request.pretty_url,
            "status": flow.response.status_code if flow.response else None,
            "request": req,
        }
        # 写入文件
        with open("debug/"+out_name, "w", encoding="utf-8") as f:
            json.dump(data, f, ensure_ascii=False, indent=2)

addons = [CustomExporter()]
```