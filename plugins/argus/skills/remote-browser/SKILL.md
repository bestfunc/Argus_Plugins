---
name: remote-browser
display_name: 远程浏览器
description: 通过 Argus 隧道 + Chrome CDP 远程控制 Agent 宿主机上的浏览器，执行网页自动化测试、截图、表单填写等操作。启动 Chrome / 清理用 run_command（L3），隧道增删改用 create/delete_tunnel（L2）。
user-invocable: true
allowed-tools: Bash,mcp__argus__list_agents,mcp__argus__run_safe_command,mcp__argus__run_command,mcp__argus__create_tunnel,mcp__argus__delete_tunnel,mcp__argus__list_tunnels
---

# 远程浏览器（Chrome CDP）

通过 Argus Agent 在远程 Windows/Linux 宿主机上启动 Chrome 调试模式，
经 Argus 隧道将 CDP 端口映射到 Server，然后用 WebSocket 控制浏览器。

## 架构

```
Claude Code (本地 Python/Bash)
  → ws://argus.bestfunc.com:{tunnel_port}/devtools/page/{id}
    → Argus 隧道 (Server:tunnel_port → Agent:9222)
      → Chrome --remote-debugging-port=9222
        → 目标网页（内网/公网）
```

## 使用流程

### 1. 查找 Chrome 路径

Windows 直接用 `shell="powershell"`,server 替你处理转义,**不用手动 base64**:

```python
run_command(
    agent_id="windows-xxx",
    shell="powershell",
    command='''$paths = @(
    "$env:ProgramFiles\\Google\\Chrome\\Application\\chrome.exe",
    "${env:ProgramFiles(x86)}\\Google\\Chrome\\Application\\chrome.exe",
    "$env:LocalAppData\\Google\\Chrome\\Application\\chrome.exe",
    "$env:ProgramFiles\\Microsoft\\Edge\\Application\\msedge.exe",
    "${env:ProgramFiles(x86)}\\Microsoft\\Edge\\Application\\msedge.exe"
)
foreach($p in $paths) { if(Test-Path $p) { Write-Host "FOUND: $p" } }''',
    _approval_reason="..."
)
```

Linux 上直接:`which google-chrome chromium-browser 2>/dev/null`

### 2. 启动 Chrome 调试模式

**必须使用用户会话 Agent**(`-bestf`/`-hp` 后缀),SYSTEM 身份无桌面无法启动 Chrome。

Windows(用 `shell="powershell"`,自然脚本无需 base64):
```python
run_command(
    agent_id="windows-xxx-bestf",
    shell="powershell",
    command='Start-Process "C:\\Program Files\\Google\\Chrome\\Application\\chrome.exe" -ArgumentList "--remote-debugging-port=9222","--remote-debugging-address=0.0.0.0","--user-data-dir=C:\\chrome-debug-profile","<目标URL>"',
    _approval_reason="..."
)
```

Linux：
```bash
run_command(agent_id, "google-chrome --remote-debugging-port=9222 --no-first-run --user-data-dir=/tmp/chrome-debug <目标URL> &")
```

### 3. 验证 Chrome 调试端口

```
run_command(agent_id, "netstat -an | findstr 9222")  # Windows
run_command(agent_id, "ss -tlnp | grep 9222")         # Linux
```

应看到 `127.0.0.1:9222 LISTENING`。

### 4. 创建 Argus 隧道

```
create_tunnel(
    agent_id="windows-xxx",  # 系统服务 Agent（隧道走系统服务）
    name="Chrome CDP",
    target_host="127.0.0.1",
    target_port=9222,
    listen_port=59222         # Server 暴露端口
)
```

### 5. 通过 CDP 控制浏览器

**重要**：CDP 请求必须带 `Host: localhost` 头（Chrome 安全限制）。

#### 列出页面
```bash
curl -s -H "Host: localhost" http://argus.bestfunc.com:59222/json/list | python3 -m json.tool
```

#### Python WebSocket 控制模板

```python
import asyncio, json, base64, websockets

CDP_BASE = "argus.bestfunc.com:59222"
HEADERS = {"Host": "localhost"}

async def cdp_call(ws, method, params=None):
    """发送 CDP 命令并等待响应"""
    msg_id = id(method) % 100000
    await ws.send(json.dumps({"id": msg_id, "method": method, "params": params or {}}))
    while True:
        resp = json.loads(await ws.recv())
        if resp.get("id") == msg_id:
            return resp.get("result", {})

async def main():
    # 1. 获取页面列表
    import urllib.request
    req = urllib.request.Request(f"http://{CDP_BASE}/json/list", headers=HEADERS)
    pages = json.loads(urllib.request.urlopen(req).read())
    page_ws = pages[0]["webSocketDebuggerUrl"].replace("localhost", CDP_BASE)

    # 2. 连接页面
    async with websockets.connect(page_ws, additional_headers=HEADERS) as ws:
        # 截图
        result = await cdp_call(ws, "Page.captureScreenshot", {"format": "png"})
        img = base64.b64decode(result["data"])
        with open("/tmp/screenshot.png", "wb") as f:
            f.write(img)
        print(f"截图: {len(img)} bytes")

        # 导航
        await cdp_call(ws, "Page.navigate", {"url": "https://example.com"})
        await asyncio.sleep(2)

        # 执行 JS
        result = await cdp_call(ws, "Runtime.evaluate", {
            "expression": "document.title"
        })
        print(f"标题: {result['result']['value']}")

        # 点击元素
        # 先找到元素节点
        doc = await cdp_call(ws, "DOM.getDocument")
        node = await cdp_call(ws, "DOM.querySelector", {
            "nodeId": doc["root"]["nodeId"],
            "selector": "#login-btn"
        })
        # 获取元素位置
        box = await cdp_call(ws, "DOM.getBoxModel", {"nodeId": node["nodeId"]})
        content = box["model"]["content"]
        x = (content[0] + content[4]) / 2
        y = (content[1] + content[5]) / 2
        # 模拟点击
        await cdp_call(ws, "Input.dispatchMouseEvent", {
            "type": "mousePressed", "x": x, "y": y, "button": "left", "clickCount": 1
        })
        await cdp_call(ws, "Input.dispatchMouseEvent", {
            "type": "mouseReleased", "x": x, "y": y, "button": "left", "clickCount": 1
        })

        # 输入文本
        await cdp_call(ws, "Input.insertText", {"text": "admin"})

asyncio.run(main())
```

## 常用 CDP 命令速查

| 操作 | 方法 | 参数 |
|------|------|------|
| 截图 | `Page.captureScreenshot` | `{format: "png"/"jpeg"}` |
| 导航 | `Page.navigate` | `{url: "..."}` |
| 执行 JS | `Runtime.evaluate` | `{expression: "..."}` |
| 获取 DOM | `DOM.getDocument` | `{}` |
| 查询选择器 | `DOM.querySelector` | `{nodeId, selector}` |
| 点击 | `Input.dispatchMouseEvent` | `{type, x, y, button}` |
| 输入文本 | `Input.insertText` | `{text: "..."}` |
| 按键 | `Input.dispatchKeyEvent` | `{type: "keyDown", key: "Enter"}` |
| 等待加载 | `Page.loadEventFired` | 事件（recv 等待） |
| Cookie | `Network.getCookies` | `{}` |
| 设置 Cookie | `Network.setCookie` | `{name, value, domain}` |
| 清除缓存 | `Network.clearBrowserCache` | `{}` |

## 本地 CDP 代理（可视化调试用）

Chrome CDP 的 HTTP 端点会校验 Host header，`chrome://inspect` 直连隧道会被拒绝。
启动一个本地代理自动注入 `Host: localhost`，即可用 Chrome DevTools 可视化调试。

**启动代理**（在 Claude Code 本机用 Bash 执行）：

```python
cat > /tmp/cdp-proxy.py << 'PYEOF'
import asyncio

REMOTE_HOST = "argus.bestfunc.com"
REMOTE_PORT = 59222  # 隧道端口，按实际修改
LOCAL_PORT = 9333

async def pipe(reader, writer, inject_host=False):
    try:
        while True:
            data = await reader.read(65536)
            if not data:
                break
            if inject_host and b"Host:" in data:
                import re
                data = re.sub(rb"Host: [^\r\n]+", b"Host: localhost", data)
            writer.write(data)
            await writer.drain()
    except Exception:
        pass
    finally:
        writer.close()

async def handle(local_r, local_w):
    remote_r, remote_w = await asyncio.open_connection(REMOTE_HOST, REMOTE_PORT)
    await asyncio.gather(
        pipe(local_r, remote_w, inject_host=True),
        pipe(remote_r, local_w),
    )

async def main():
    server = await asyncio.start_server(handle, "127.0.0.1", LOCAL_PORT)
    print(f"CDP 代理: localhost:{LOCAL_PORT} → {REMOTE_HOST}:{REMOTE_PORT}")
    print(f"打开 chrome://inspect#devices → Configure → 添加 localhost:{LOCAL_PORT}")
    async with server:
        await server.serve_forever()

asyncio.run(main())
PYEOF
python3 /tmp/cdp-proxy.py &
```

代理启动后：
1. 本地 Chrome 打开 `chrome://inspect#devices`
2. 点 **Configure...** → 添加 `localhost:9333`
3. 远程页面自动出现，点 **inspect** 打开完整 DevTools
4. Elements/Console/Network/Sources 全部可用，和本地调试体验一致

**停止代理**：`kill %1` 或 `pkill -f cdp-proxy`

## React/Vue 表单填写技巧

现代前端框架的 `<input>` 是受控组件，直接设 `.value` 不会触发状态更新。
必须用原生 setter + 派发事件：

```javascript
// 在 Runtime.evaluate 中使用
function fillInput(selector, value) {
    const el = document.querySelector(selector);
    const setter = Object.getOwnPropertyDescriptor(
        HTMLInputElement.prototype, 'value').set;
    setter.call(el, value);
    el.dispatchEvent(new Event('input', {bubbles: true}));
    el.dispatchEvent(new Event('change', {bubbles: true}));
}
fillInput('input[placeholder*="用户名"]', 'admin');
fillInput('input[type="password"]', '123456');
```

## 清理

测试完成后：

1. 关闭 Chrome：`run_command(user_agent_id, "taskkill /f /im chrome.exe")`
2. 删除隧道：`delete_tunnel(mapping_id)`
3. 清理调试 profile：`run_command(agent_id, 'rmdir /s /q C:\\chrome-debug-profile')`
4. 停止本地代理：`pkill -f cdp-proxy`

## MCP 授权说明（v1.29+）

本 skill 全流程要过多次邮箱审批，建议**一次性走完后尽量复用**：

| 工具 | 等级 | 何时用 |
|------|------|--------|
| `list_agents` / `list_tunnels` | 🟢 L1 | 查状态 |
| `run_safe_command` | 🟢 L1 | `netstat` / `which chrome` 等只读诊断 |
| `create_tunnel` / `delete_tunnel` | 🟡 L2 | 建/删 CDP 隧道（首次授权 + 15 分钟快路径） |
| `run_command` | 🔴 L3 | 启动 Chrome、taskkill、删 profile 等（每次 5 分钟验证码） |

典型一次调试会话需要的 L3 审批次数：
1. 启 Chrome（`run_command`）— 1 次
2. 清理 profile / 杀进程（`run_command`）— 1-2 次

**铁律**：验证码从用户邮箱里拿，不要自己编。详见 `/MCP授权` skill。

## 注意事项

- Chrome 必须用**用户会话 Agent** 启动（需要桌面环境）
- 隧道用**系统服务 Agent** 创建（隧道走系统服务的 WS）
- 启动 Chrome 时加 `--remote-allow-origins=*` 允许跨域 WebSocket
- CDP HTTP 端点需要 `Host: localhost` 头（通过本地代理或 curl -H 解决）
- `--user-data-dir` 用独立目录避免和正常 Chrome 冲突
- Chrome 新版默认只监听 `127.0.0.1:9222`，通过隧道访问无影响
- 如果目标机器已有 Chrome 进程在跑，需先关闭或用不同端口
