---
name: computer-use
display_name: 远程操控
description: 通过 Argus Remote Computer Use 远程操控宿主机 GUI——截图、鼠标点击、键盘输入、剪贴板读写、UI 元素识别。感知类（截图 / 读剪贴板 / UI 树）是 L1 直接可用；所有 click / double_click / right_click / drag / key / type_text 均为 L3，每步都要邮箱审批，执行前务必与用户确认整体方案。
user-invocable: true
allowed-tools: mcp__argus__list_agents,mcp__argus__get_screen_info,mcp__argus__get_ui_elements,mcp__argus__set_clipboard,mcp__argus__scroll,mcp__argus__screenshot,mcp__argus__get_clipboard,mcp__argus__wait_stable,mcp__argus__drag,mcp__argus__click,mcp__argus__double_click,mcp__argus__right_click,mcp__argus__type_text,mcp__argus__key
---

# 远程操控（Remote Computer Use）

通过 Argus MCP 远程操控宿主机 GUI，支持 Claude/GPT/Gemini 等任意 AI 模型。

## 可用工具 & 授权等级

### 🟢 L1 感知类（直接可用）

| 工具 | 说明 |
|------|------|
| `get_screen_info` | 屏幕分辨率、DPI |
| `get_ui_elements` | 前台窗口的 UI 控件树（含坐标） |
| `screenshot` | 全屏缩略图或指定 region 高清截图 |
| `wait_stable` | 等屏幕动画结束 |
| `get_clipboard` | 读取剪贴板文字 |
| `set_clipboard(text)` | 写入剪贴板 |
| `scroll(x, y, delta)` | 滚轮 |

### 🔴 L3 执行类（每次邮箱审批，5 分钟验证码）

| 工具 | 说明 |
|------|------|
| `click(x, y)` | 鼠标左键点击 |
| `double_click(x, y)` | 双击 |
| `right_click(x, y)` | 右键 |
| `drag(x1,y1,x2,y2)` | 鼠标拖拽（可能触发文件移动 / 拖入回收站等破坏性动作） |
| `type_text(text)` | 逐字符输入（支持中文） |
| `key(combo)` | 按键/组合键：enter, ctrl+c, alt+f4 |

> ⚠️ **重要**：所有鼠标点击/拖拽、键盘输入都是 L3，**每次调用都要一次邮箱验证码**。
> Computer Use 场景下这会非常慢。做法是：
>
> 1. **先规划**——用 L1 感知工具（`get_ui_elements`、`screenshot`）摸清界面
> 2. **跟用户说清楚整体步骤**——例如"我会依次点 A、输入 B、按 Enter，共 3 次审批"
> 3. **连续执行**——每步调用都带 `_approval_reason`，用户连续几次验证码就完成
> 4. 避免无脑试探；每次 click/drag/key 都有成本

## 操作流程

### 1. 确定目标 Agent

优先使用**用户会话 Agent**（`-bestf`/`-hp` 后缀），它有桌面环境。

### 2. 规划阶段（全 L1，直接可用）

```
get_ui_elements()            → 获取控件树，零截图成本
screenshot()                 → 全屏缩略图
screenshot(region={x,y,w,h}) → 局部高清，确认细节
```

### 3. 坐标系统

**所有 click/scroll/drag 的坐标都是基于最近一次 screenshot 的图片像素坐标。**

MCP Server 自动维护 RegionContext，click 坐标会自动换算为屏幕绝对坐标：
- `screenshot()` 无 region → 缩略图模式，坐标会自动放大
- `screenshot(region={x:100,y:200,w:800,h:600})` → 区域模式，坐标加偏移

无需手动计算。

### 4. 执行阶段（L3 审批）

**每一次操作都是独立审批**：

```python
# 第一次 click
click(
    x=500, y=400,
    _approval_reason="用户让我打开文件菜单，点击左上角 文件 按钮 (500,400)"
)
# → APPROVAL_PENDING，用户查邮箱，给 6 位验证码

click(x=500, y=400, _approval_reason="（同上）", _approval_code="284713")
# → 成功

# 紧接下一步，重新审批
type_text(
    text="hello",
    _approval_reason="用户让我在当前输入框输入 hello"
)
# → APPROVAL_PENDING，等新验证码
```

### 5. 确认阶段（L1）

```
wait_stable()      → 等动画结束
screenshot(region) → 确认结果（L1，无需审批）
```

## 优先级策略

**优先低成本工具，L3 执行作为最后手段**：

```
1. get_ui_elements() ← L1 控件树，结构化，坐标直接可用
2. screenshot()      ← L1 截图（全屏或 region）
3. get_clipboard()   ← L1 读剪贴板
4. set_clipboard()   ← L1 写剪贴板
5. wait_stable()     ← L1 等动画
6. click/drag/key/...← L3，每次审批
```

## 文字输入策略

| 场景 | 方法 |
|------|------|
| 短文本（< 20 字） | `type_text("hello")` — L3，一次审批 |
| 长文本 / 中文 | `set_clipboard("长文本...")`（L1）→ `key("ctrl+v")`（L3） |
| 特殊字符 | `set_clipboard("特殊内容")`（L1）→ `key("ctrl+v")`（L3） |

对长文本，`set_clipboard + ctrl+v` 的 L3 成本是一次（`ctrl+v`），而 `type_text` 也是一次——但前者通常更快更可靠。

## get_ui_elements 使用说明

- 返回**前台窗口**的所有可交互控件（按钮、输入框、菜单等）
- 每个元素包含 `type`、`name`、`enabled`、`bounds{x,y,w,h}`
- bounds 坐标是**屏幕绝对坐标**，可直接用于 click
- 注意：使用前需要先 `screenshot()` 一次（设置 RegionContext 为全屏），然后用 `get_ui_elements` 返回的 bounds 坐标时，需要换算为图片坐标
- Windows 标准控件（Win32/WPF）效果好，自绘界面 / Electron 应用可能返回空，降级用 screenshot
- 底层通过 PowerShell 子进程调用，不会卡死 Agent

## 典型场景

### 场景1：打开记事本并输入

**规划**（共需约 5 次 L3 审批）：
1. `key("win+r")` — 打开运行
2. `type_text("notepad")` — 输入
3. `key("enter")` — 确认
4. `type_text("Hello World")` — 输入正文
5. `key("ctrl+s")` — 保存（后续文件名对话框再走一两次）

**执行前告诉用户**：
> 接下来要做 5 次键鼠操作，每次都会给你邮箱发验证码。准备好接收验证码吗？

### 场景2：用 UI 元素树精准点击

```
1. get_ui_elements()           → L1，找到 button name="确定" bounds={x:500,y:400}
2. click(500, 400)             → L3，一次审批
```

### 场景3：读取界面上的文字

```
# 方案A：UI 元素树（快，L1）
get_ui_elements()   → 直接从 name 字段读文字

# 方案B：选中 + 复制（通用，但一串 L3）
click(x, y)         → L3 审批 1
key("ctrl+a")       → L3 审批 2
key("ctrl+c")       → L3 审批 3
get_clipboard()     → L1，直接可用
```

## 组合键参考

| 操作 | combo |
|------|-------|
| 复制 | `ctrl+c` |
| 粘贴 | `ctrl+v` |
| 全选 | `ctrl+a` |
| 撤销 | `ctrl+z` |
| 保存 | `ctrl+s` |
| 关闭窗口 | `alt+f4` |
| 切换窗口 | `alt+tab` |
| 打开运行 | `win+r` |
| 打开任务管理器 | `ctrl+shift+escape` |
| Tab 切换焦点 | `tab` |
| 确认 | `enter` |
| 取消 | `escape` |

## 授权协议

调用 L2 / L3 工具走邮箱验证码流程（见 `/MCP授权` skill）：

- **铁律**：验证码必须等用户给，不要自己编
- **铁律**：重放时 Server 用数据库里保存的参数执行，改 x/y/text 无效
- `_approval_reason` 每次都要具体："点击什么按钮"、"输入什么内容"，会写入审计日志

## 何时用 computer-use vs remote-browser vs terminal

目标是**能用 API 就别用 GUI**，能少 L3 审批就少：

| 目标 | 优先 | 次选 | 最后 |
|------|------|------|------|
| 跑命令 / 看日志 | `terminal` (run_safe_command L1) | `terminal` (run_command L3) | — |
| 访问 HTTP API | `api-query` (proxy_api_get L1) | `proxy_api` L2 | — |
| 操作网页 | `remote-browser`（CDP，能自动化 + 结构化取 DOM） | `computer-use`（点坐标） | — |
| 配置工业软件（BestPLC / 扫描仪 / 自研 GUI） | `computer-use`（没 API 只能点） | — | — |
| 填 Win32 对话框 / 老系统 | `computer-use` | — | — |

**remote-browser 比 computer-use 优先**：CDP 能结构化拿 DOM、精确找元素、模拟点击比像素坐标点击可靠 10 倍，而且只需要启动一次 Chrome（一次 L3）之后基本走 CDP 协议（不再审批）。

**computer-use 的 L3 放大问题**：click / double_click / right_click / key / type_text 每一次都是 L3，复杂操作（开软件 → 点菜单 → 填表 → 保存）可能 10 次 L3 = 10 封邮件。事先和用户讲清楚步骤数和预计 L3 次数。

## 注意事项

- 使用**用户会话 Agent**（有桌面环境）
- 截图使用 GDI BitBlt，与远程桌面共存不冲突
- 每次 L3 操作后建议 `wait_stable()` + `screenshot(region)` 确认结果
- 不要盲目连续操作——每步都要邮箱验证码，盲打=浪费用户时间
- 优先用 `get_ui_elements` 和 `get_clipboard` 获取信息，减少截图次数
- Computer Use 是**兜底方案**，如果能用 `run_safe_command` / `execute_select` / `proxy_api_get` / `read_file` 解决的问题，不要用截图操控
