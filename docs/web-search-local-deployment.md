# Web Search Local (SearXNG) 完整部署文档

**更新日期:** 2026-04-14
**状态:** 生产可用

---

## 一、架构概览

```
用户提问 → Claude Code → MCP Server (server.py) → SearXNG (localhost:58080) → 百度/搜狗/360/Brave/Bing
```

- **SearXNG**: 开源元搜索引擎，聚合多个搜索引擎的结果
- **MCP Server**: Python 脚本，将 SearXNG JSON API 暴露为 Claude Code 的 MCP 工具
- **工具名**: `mcp__web-search-local`

---

## 二、前置条件

- Docker Desktop（Windows/Mac/Linux）
- Python 3.8+
- Claude Code CLI

---

## 三、SearXNG 部署

### 3.1 启动容器

```bash
docker run -d \
  --name searxng \
  -p 58080:8080 \
  -v searxng-config:/etc/searxng \
  -v searxng-cache:/var/cache/searxng \
  --restart=always \
  searxng/searxng:latest
```

> **注意**: 初次启动后需等 5-10 秒让服务完全启动。

### 3.2 验证服务

```bash
curl -s -o /dev/null -w "%{http_code}" http://localhost:58080
# 应返回 200
```

浏览器访问 `http://localhost:58080` 确认界面正常。

### 3.3 检查 DNS 解析

```bash
docker exec searxng python3 -c "
import socket
for host in ['www.baidu.com', 'www.bing.com']:
    ips = socket.getaddrinfo(host, 443)
    print(f'{host}: {ips[0][4][0]}')
"
```

如果返回的 IP 明显错误（如 google.com 返回 Facebook 的 IP），说明 DNS 污染，需要重建容器并指定 DNS：

```bash
docker stop searxng && docker rm searxng
docker run -d \
  --name searxng \
  -p 58080:8080 \
  --dns 223.5.5.5 \
  --dns 114.114.114.114 \
  -v searxng-config:/etc/searxng \
  -v searxng-cache:/var/cache/searxng \
  searxng/searxng:latest
```

---

## 四、SearXNG 配置

### 4.1 写入配置文件

```bash
cat > /tmp/searxng-settings.yml << 'EOF'
use_default_settings: true

server:
  secret_key: "searxng-local-key"
  limiter: false
  image_proxy: true

search:
  formats:
    - html
    - json
  languages:
    - zh-CN
    - en

engines:
  # === 国内引擎 ===
  - name: baidu
    engine: baidu
    disabled: false
    categories: general
    timeout: 10

  - name: sogou
    disabled: false
    timeout: 10

  - name: sogou videos
    disabled: false
    timeout: 10

  - name: sogou wechat
    disabled: false
    timeout: 10

  - name: 360search
    disabled: false
    timeout: 10

  - name: 360search videos
    disabled: false
    timeout: 10

  # === 国际引擎 ===
  - name: brave
    disabled: false
    timeout: 10

  - name: bing
    disabled: false
    timeout: 10

  - name: duckduckgo
    disabled: false
    timeout: 10
EOF

docker cp /tmp/searxng-settings.yml searxng:/etc/searxng/settings.yml
docker restart searxng
```

> **关键点**: 百度引擎源码中 `categories = []`（空列表），**必须**在配置中显式指定 `categories: general`，否则不会参与搜索。

### 4.2 等待启动完成

```bash
sleep 10
```

### 4.3 验证搜索

```bash
curl -s "http://localhost:58080/search?q=%E6%B5%8B%E8%AF%95&format=json" | head -c 500
```

应返回包含结果的 JSON。

---

## 五、MCP Server 部署

### 5.1 创建目录

```bash
mkdir -p ~/.claude/mcp-servers/web-search-local
```

### 5.2 编写 server.py

创建 `~/.claude/mcp-servers/web-search-local/server.py`：

```python
#!/usr/bin/env python3
"""MCP Server for local web search via SearXNG."""

import os
import json
import urllib.request
import urllib.parse
from mcp.server.fastmcp import FastMCP

SEARXNG_URL = os.environ.get("SEARXNG_URL", "http://localhost:58080")

mcp = FastMCP("web-search-local")

@mcp.tool()
def web_search_local(query: str, max_results: int = 10) -> str:
    """Search the web using local SearXNG instance (Bing + Google + Baidu)."""
    params = {
        "q": query,
        "format": "json",
        "categories": "general",
    }
    url = f"{SEARXNG_URL}/search?{urllib.parse.urlencode(params)}"
    req = urllib.request.Request(url, headers={"User-Agent": "MCP-WebSearch/1.0"})

    try:
        with urllib.request.urlopen(req, timeout=15) as resp:
            data = json.loads(resp.read())
    except Exception as e:
        return f"Search error: {e}"

    results = data.get("results", [])[:max_results]
    if not results:
        return f"No results found for: {query}"

    output = [f"Search results for: {query}", ""]
    for i, r in enumerate(results, 1):
        output.append(f"[{i}] {r['title']}")
        output.append(f"    URL: {r['url']}")
        output.append(f"    Source: {r.get('engine', 'unknown')}")
        if r.get('content'):
            output.append(f"    {r['content'][:200]}")
        output.append("")

    return "\n".join(output)

if __name__ == "__main__":
    mcp.run()
```

### 5.3 创建 .mcp.json

创建 `~/.claude/mcp-servers/web-search-local/.mcp.json`：

```json
{
  "web-search-local": {
    "command": "python",
    "args": ["${CLAUDE_MCP_SERVERS_DIR}/web-search-local/server.py"],
    "env": {
      "SEARXNG_URL": "http://localhost:58080"
    }
  }
}
```

---

## 六、注册到 Claude Code

### 6.1 Windows 特殊处理

Windows 下需要设置 git-bash 路径：

```bash
set CLAUDE_CODE_GIT_BASH_PATH=E:\zSoftware\Git\usr\bin\bash.exe
```

### 6.2 注册 MCP 服务器

```bash
claude mcp add -s user web-search-local -- python "C:\Users\13811\.claude\mcp-servers\web-search-local\server.py"
```

> **关键点**: 仅创建 `.mcp.json` 文件是不够的，**必须**通过 `claude mcp add` 注册到 `claude.json` 的 `mcpServers` 字段。

### 6.3 验证

```bash
claude mcp list
# 应显示: web-search-local: ... - ✓ Connected
```

### 6.4 重启 Claude Code

使配置生效。

---

## 七、使用方法

### 7.1 工具调用

```
mcp__web-search-local(
  query="搜索关键词",
  max_results=10  # 可选，默认 10
)
```

### 7.2 搜索结果示例

```
Search results for: 电池热管理 专利

[1] 清陶新能源申请热管理控制方法专利
    URL: https://it.sohu.com/a/1008191733_114984
    Source: baidu
    国家知识产权局信息显示...

[2] 河南海威新能源取得电池热管理系统专利
    URL: https://baijiahao.baidu.com/s?id=...
    Source: baidu
    金融界2025年4月23日消息...
```

---

## 八、目录结构

```
~/.claude/
├── mcp-servers/
    ├── searxng-config/
    │   └── settings.yml          # SearXNG 配置（已改用 Docker volume）
    └── web-search-local/
        ├── .mcp.json             # MCP 启动配置
        └── server.py             # MCP Server 脚本
```

---

## 九、常见问题

### Q1: 搜索返回空结果

1. 检查 SearXNG 服务是否正常：`curl http://localhost:58080`
2. 检查容器 DNS：`docker exec searxng python3 -c "import socket; print(socket.getaddrinfo('www.baidu.com', 443))"`
3. 检查引擎状态：访问 `http://localhost:58080/config` 查看 `enabled` 字段
4. 百度引擎必须显式指定 `categories: general`

### Q2: MCP 工具不加载

1. 运行 `claude mcp list` 确认状态为 `✓ Connected`
2. 检查 `claude.json` 中是否有 `mcpServers` 字段
3. 重新注册：`claude mcp add -s user web-search-local -- python "路径/server.py"`
4. 重启 Claude Code

### Q3: Docker 容器 DNS 污染

症状：`socket.getaddrinfo('google.com', 443)` 返回明显错误的 IP。

解决：重建容器并指定 `--dns 223.5.5.5 --dns 114.114.114.114`。

### Q4: 搜索引擎超时

- 可能是反爬机制，增加 `timeout: 10`
- 百度引擎的 `categories = []` 问题（见 Q1）
- 检查容器内是否能直接访问搜索引擎：`docker exec searxng python3 -c "import urllib.request; print(urllib.request.urlopen('https://www.baidu.com', timeout=5).status)"`

---

## 十、当前可用引擎

| 引擎 | 状态 | 特点 |
|------|------|------|
| Baidu | ✅ | 中文搜索最佳 |
| Baidu Images | ✅ | 中文图片 |
| Sogou | ✅ | 中文搜索 |
| Sogou Videos | ✅ | 中文视频 |
| Sogou WeChat | ✅ | 微信文章 |
| 360search | ✅ | 360搜索 |
| 360search Videos | ✅ | 360视频 |
| Brave | ✅ | 英文搜索 |
| Bing | ✅ | 中英混合 |
| DuckDuckGo | ✅ | 隐私搜索 |

---
