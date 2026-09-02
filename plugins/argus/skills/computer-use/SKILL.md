---
name: computer-use
display_name: 远程操控
description: 通过 Argus Remote Computer Use 远程操控宿主机 GUI。**优先用语义层**（ui_snapshot 列元素 → ui_act 按元素引用操作 → ui_wait 等状态），不需要看图算坐标；语义层够不到时（浏览器网页内容、自绘界面）才退回截图 + 坐标点击。感知类（ui_snapshot / 截图 / 读剪贴板 / ui_wait）是 L1 直接可用；所有执行类（ui_act / click / drag / key / type_text）均为 L3，每步都要邮箱审批，执行前务必与用户确认整体方案。
user-invocable: true
allowed-tools: mcp__argus__list_agents,mcp__argus__ui_snapshot,mcp__argus__ui_act,mcp__argus__ui_wait,mcp__argus__get_screen_info,mcp__argus__get_ui_elements,mcp__argus__set_clipboard,mcp__argus__scroll,mcp__argus__screenshot,mcp__argus__get_clipboard,mcp__argus__wait_stable,mcp__argus__drag,mcp__argus__click,mcp__argus__double_click,mcp__argus__right_click,mcp__argus__type_text,mcp__argus__key
---

# 远程操控（Remote Computer Use）

通过 Argus MCP 远程操控宿主机 GUI。

## 先选层：语义层还是视觉层

**默认走语义层。** 视觉层（截图 + 算坐标）是兜底，不是起点。

| | 语义层（v1.41+） | 视觉层 |
|---|---|---|
| 工具 | `ui_snapshot` / `ui_act` / `ui_wait` | `screenshot` / `click` / `type_text` / `key` |
| 定位方式 | 元素引用 `ref` 或 `{role,name}` | 像素坐标 |
| 单步成本 | ~200ms，几十 token | 3~5 秒，约 1500 token/张图 |
| 可靠性 | 走控件原生调用，不受遮挡/焦点/DPI 影响 | 窗口一动坐标就失效，**且不报错** |
| 适用 | Win32 / WinForms / WPF / UWP 原生控件 | 浏览器网页内容、自绘界面、游戏引擎 |

判断流程：

```
不确定操作哪个窗口 → ui_snapshot(scope="windows") 先选窗口，拿 hwnd
        ↓
ui_snapshot(agent_id, hwnd=...)（L1，免审批）
├── 拿到了目标元素          → 用 ui_act(ref="eN") 操作，全程不碰坐标
├── 返回 note 说"浏览器窗口" → 网页内容用 /argus:remote-browser（CDP），别用截图硬点
└── 元素为空或找不到目标     → 先看 roles 确认类型名，再退回 screenshot + click
```

## 一、语义层工具

### 第 0 步：先确定操作哪个窗口

**不确定目标在哪个窗口时，先列窗口**，别默认操作前台：

```
ui_snapshot(agent_id, scope="windows")
ui_snapshot(agent_id, scope="windows", query="wps")   # 按标题/进程名过滤
```

```
0x70588  计算器  (ApplicationFrameHost.exe) [前台]
0x201A8  WPS Office  (wps.exe)
0x1606F6  base-install.log - Notepad  (Notepad.exe)
0x9026C  cmd.exe  (polter.exe) [最小化]
```

行首就是窗口句柄，填进后续调用的 `hwnd` 参数即可。**不列窗口就只能操作前台窗口**——
想动后台程序时只能猜，而猜错的表现是**在别的窗口上执行了操作**，不会报错。

标 `[最小化]` 的窗口用户看不到：操作它不会有可见反馈，也没法用截图核对。
真要操作，先 `ui_act(hwnd=..., action="restore")` 让它显示出来。

> 已过滤掉无标题窗口、工具窗口、被 DWM 隐藏(cloaked)的幽灵窗口——
> 尤其是 UWP 会留下一堆同名的隐藏 `ApplicationFrameWindow`。

### ui_snapshot（L1）

列出目标窗口**可操作的元素**。**默认紧凑输出**，一行一个：

```
ui_snapshot(agent_id)                          # 前台窗口，默认每页 50 条；返回里先看 roles
ui_snapshot(agent_id, query="保存")             # 按名称/AutomationId/文本检索
ui_snapshot(agent_id, role="button")           # 只看按钮
ui_snapshot(agent_id, role="listitem", limit=20, offset=20)   # 翻页
ui_snapshot(agent_id, hwnd="0x3A02C")          # 指定窗口
ui_snapshot(agent_id, detailed=true)           # 要完整字段时才开
ui_snapshot(agent_id, all=true)                # 连装饰性容器一起看（排查用）
```

返回的 `lines` 长这样，行首是**序号**，缩进表示层级：

```
e7    button 新建
e8    button 打开
e12     listitem 报表.xlsx
e15   edit 工件编号 = X2411
e23   menuitem 文件
```

**要操作某个元素，直接把行首序号给 ui_act**：`ui_act(agent_id, ref="e7", action="click")`。

**筛 `role` 之前先看 `roles`。** 每次返回都带这个窗口的角色分布（按数量降序，不受本次筛选影响）：

```
"roles": ["group 39", "button 14", "text 12", "listitem 6", "edit 3", "menuitem 3"]
```

不看它就只能瞎猜类型名——猜 `role="tab"` 而这个应用其实叫 `tabitem`，返回空，
你还会以为界面上没有标签页。**空结果先回头看 `roles`**，而不是断定"没有"。

**不要一次拉整棵树。** 界面复杂时元素上百，全拉回来既费 token 又难读。
先用 `query` / `role` 缩小范围，用 `limit`/`offset` 翻页。返回里 `matched` 是符合条件的总数、
`returned` 是本页条数、`has_more` 表示还有下一页。

几个必须知道的性质：

- **序号只在本次快照内有效**。界面结构一变（新开标签页、切换页面）序号就会偏移。
  ui_act 的回执会如实报告实际点到了什么元素——据此判断序号是否过期。
  跨界面变化的操作请重新 snapshot。
- **`note` 字段必须读**。它说明本次的盲区：浏览器窗口拿不到网页内容、目标窗口不可见、
  UIA 在该窗口完全不工作……不读它就会把"我没看到"当成"界面上没有"。
- 标 `[无名]` 的元素既无名称也无 AutomationId，只能靠序号指，**尤其容易过期**。
- `detailed=true` 才有 `actions` / `bounds` / `aid` / 长引用串。而 **`actions` 只表示元素
  "声称"支持的动作，不保证可用**——实测 WPS 的空名 group、ToDesk 的 image 都声称支持
  `set_value`，那是 provider 乱标。

### ui_act（L3，每次邮箱审批）

对元素执行动作，**不需要坐标**。

```
# 方式一：用 snapshot 清单行首的短序号（推荐）
ui_act(agent_id, ref="e7", action="click",
       _approval_reason="用户让我打开检测记录，点击左侧菜单 检测管理")

# 方式二：直接描述目标，省掉一次 snapshot
ui_act(agent_id, target={"role":"button","name":"保存"}, action="click", ...)
ui_act(agent_id, target={"name_contains":"下载"}, action="click", ...)
ui_act(agent_id, target={"aid":"btnSubmit"}, action="click", ...)
```

动作集（用哪个要看元素的 `actions` 列表）：

| action | 用途 |
|---|---|
| `click` | 点击按钮/链接 |
| `toggle` | 勾选/取消复选框 |
| `select` | 选中列表项、标签页 |
| `expand` / `collapse` | 展开/收起菜单、树节点 |
| `set_value` | **直接写入输入框** |
| `scroll_into_view` | 把元素滚动到可见 |
| `focus` | 获取焦点 |

批量连点见下方「六、一串动作用 steps」。

**`set_value` 是最大的效率提升**：直接写值，比逐字符输入快两个数量级，而且**完全绕开输入法**——输入中文不再需要拼音组字、不会因 IME 状态丢字。只有必须模拟真实键入（某些控件靠按键事件触发校验）时才用 `type_text`。

回执里 **`did` 字段必须看**：

- `InvokePattern.Invoke` — 走的是控件原生调用，最可靠
- `mouse_click@(847,392)` — 该元素没暴露可用 pattern，**已退化为物理点击**，`fallback_reason` 会说明原因。退化后的点击受窗口遮挡、焦点、DPI 影响，和前者不是一回事，后续要多确认一步

找不到目标时会返回 `candidates` 候选清单——直接从里面挑 `ref` 重试，**不要回去截图重找**。

### ui_wait（L1）

等元素达到某状态，**代替「操作后再截一张图确认」**。

```
ui_wait(agent_id, target={"name":"检测完成"}, state="appear", timeout_ms=8000)
ui_wait(agent_id, target={"role":"button","name":"确定"}, state="enabled")
```

`state`：`appear` / `disappear` / `enabled` / `disabled`。

比 `wait_stable` 精确：后者只知道「画面不动了」，它知道「那个按钮真的出现了」。超时不算错误，返回 `ok=false` 并附上当时的界面快照，直接看它判断即可。

## 二、典型流程对比

同一件事（打开菜单 → 点条目 → 填表单 → 提交）：

**旧方式**：截图 → 算坐标 → click → 截图确认 → 截图 → 算坐标 → click → 截图确认 …
约 8 次调用、8 张截图、1.2 万 token、30 秒以上。

**语义层**：

```
ui_snapshot(agent_id, role="menuitem")        # L1，只看菜单，回来是几行文本
ui_act(agent_id, steps=[                      # L3，一次授权跑完四步
  {"ref":"e23", "action":"expand", "wait_ms":200},
  {"target":{"name":"新建工单"}, "action":"click", "wait_ms":300},
  {"target":{"aid":"txtOrderNo"}, "action":"set_value", "value":"X2411"},
  {"target":{"name":"提交"}, "action":"click"}
])
ui_wait(agent_id, target={"name":"提交成功"}, state="appear")   # L1，确认
```

零截图、零坐标，**L3 审批从 4 次降到 1 次**。

## 三、一串动作用 steps 批量发

按 7、按 +、按 8、按 = 这种连续操作，**不要发四次**——一次 `steps` 跑完，
一次授权、一次往返：

```
ui_act(agent_id, steps=[
  {"target":{"aid":"num7Button"}, "action":"click"},
  {"ref":"e12", "action":"click"},
  {"target":{"role":"edit"}, "action":"set_value", "value":"X2411", "wait_ms":200},
  {"target":{"name":"提交"}, "action":"click"}
])
```

- 每步可带 `wait_ms`：点菜单后要留时间给它展开，否则下一步在旧界面上找元素
- 默认**失败即停**（`stop_on_error=true`）：UI 操作有顺序依赖，前一步没成，
  后面就是在另一个界面上执行
- **默认只回一个汇总**（`summary` 是紧凑动作序列，`completed` 是真正成功的步数），
  失败那一步的详情和候选清单始终给；要逐步详情传 `verbose=true`

> **部分成功时，前面那些步是真的执行了、界面已经变了**。不要把 `ok=false` 当成
> "什么都没发生"直接重跑整串——先看 `completed` 和 `stopped_at`，从断点接着做。

窗口级动作（`minimize`/`maximize`/`restore`/`close_window`）**不需要 target**，
直接作用于 `hwnd` 或前台窗口，对自绘界面同样有效。

## 四、语义层在不同应用上差别很大

实测四档，**先判断自己在哪一档**，再决定是走语义层还是退回截图：

| 档 | 典型 | AutomationId | 表现 |
|---|---|---|---|
| A | 计算器、记事本、设置（微软自家 XAML/UWP）、WPF/WinForms | 齐全 | 完全可用，序号和 ref 都稳 |
| B | WPS 等自绘 Office | **无** | 能用，但只能靠名称定位；同名元素会撞 |
| C | ToDesk 等自绘工具 | 无 | 同上，无名元素更多 |
| D | 纯自绘（游戏引擎、部分终端/国产软件） | 无 | **语义层够不到**，snapshot 可能只有 1 个元素 |

- A 档：放心用序号，跨几步操作也不容易过期
- B/C 档：**每次操作前重新 snapshot**，序号和 ref 都当一次性的
- D 档：`note` 会明说"UIA 拿不到元素树"，直接退回 `screenshot` + `click`；
  但**窗口级动作仍然可用**（最小化/最大化/关闭走的是操作系统，与应用无关）

**回执里出现 `ambiguous` 要当心**：界面上有多个元素同样符合条件，这次点中的
不一定是你要的那个（操作本身是成功的）。想精确指定：用带 `aid` 的元素，
或先 snapshot 拿最新序号再立刻操作。

## 五、L3 审批：用批量把它压下来

`ui_act` 和 `click` 一样是 L3，**每次调用一封邮件验证码**。语义层本身省掉的是截图和坐标，
不是审批——但 `steps` 批量能把 N 次审批压成 1 次，这是目前最大的一块效率来源。

所以做法是：

1. **先用 L1 摸清界面**（`ui_snapshot` + `query`/`role`，免审批）
2. **把整条操作链一次规划好**，用 `steps` 一次发出去
3. **告诉用户这一次审批要做哪几件事**——"我会展开菜单、点新建、填单号、提交，
   一次验证码跑完"
4. 只有在必须看中间结果才能决定下一步时，才拆成多次 `ui_act`

`_approval_reason` 要具体（"点击左侧菜单 检测管理"），它会写进审计日志；批量时
把整串动作说清楚。

## 六、视觉层（兜底）

语义层够不到时用。工具和坐标语义没变：

| 工具 | 级别 | 说明 |
|---|---|---|
| `screenshot` | L1 | 全屏缩略图或 region 高清 |
| `wait_stable` | L1 | 等动画结束 |
| `get_screen_info` | L1 | 分辨率 / DPI |
| `get_clipboard` / `set_clipboard` | L1 | 剪贴板读写 |
| `scroll` | L1 | 滚轮 |
| `click` / `double_click` / `right_click` / `drag` | L3 | 坐标点击 |
| `type_text` / `key` | L3 | 键盘输入 |

**坐标系统**：`click` 等的坐标基于最近一次 `screenshot` 的**图片坐标**，把那次截图返回的 `capture_meta` 一并传回来，Server 自动换算成屏幕坐标。

> ⚠️ `capture_meta` 里若出现 `scale_unknown: true`，说明该 Agent 版本旧、没回传真实分辨率，**这张图的坐标不能用于点击**（传给 click 会被直接拒绝）。先 `get_screen_info` 自己换算，或改用 region 模式截图（坐标 1:1）；根治办法是升级 Agent 到 v1.40+。

**文字输入**：长文本/中文用 `set_clipboard`（L1）+ `key("ctrl+v")`（L3），比 `type_text` 快且不受输入法影响。但元素在语义层可见时，直接 `ui_act(action="set_value")` 更好。

**`key` 的返回值要看 `sent` 数组**：它列出实际注入的每个按键（含虚拟键码和扫描码）。`sent` 里没有的键就是没发出去的键——不要用 `success` 本身当作"热键已送达"的判据。被测对象若读的是扫描码（DirectInput / RawInput），加 `injection="scancode_only"`。

## 七、`get_ui_elements`（旧工具，保留）

v1.41 之前的 UI 树工具，只读不能执行，噪音多、没有 AutomationId。**新场景一律用 `ui_snapshot`**，它保留只是为了兼容既有流程。

## 八、何时用 computer-use vs remote-browser vs terminal

目标是**能用 API 就别用 GUI，能用语义层就别用坐标**：

| 目标 | 优先 | 次选 | 最后 |
|---|---|---|---|
| 跑命令 / 看日志 | `terminal`（run_safe_command L1） | `terminal`（run_command L3） | — |
| 访问 HTTP API | `api-query`（proxy_api_get L1） | `proxy_api` L2 | — |
| **操作网页** | `remote-browser`（CDP，结构化取 DOM） | `computer-use` 视觉层 | — |
| 配置工业软件（BestPLC / 扫描仪 / 自研 GUI） | `computer-use` **语义层** | `computer-use` 视觉层 | — |
| 填 Win32 对话框 / 老系统 | `computer-use` **语义层** | 视觉层 | — |

**网页内容一定走 remote-browser**：`ui_snapshot` 对浏览器窗口只能看到外壳（标签页、地址栏），网页内容拿不到（Chromium 的 a11y 树不向 UIA 暴露），snapshot 的 `note` 会明说这一点。别看到"元素列表里没有登录按钮"就以为页面上没有。

## 九、组合键参考

| 操作 | combo | 操作 | combo |
|---|---|---|---|
| 复制 | `ctrl+c` | 关闭窗口 | `alt+f4` |
| 粘贴 | `ctrl+v` | 切换窗口 | `alt+tab` |
| 全选 | `ctrl+a` | 打开运行 | `win+r` |
| 撤销 | `ctrl+z` | 任务管理器 | `ctrl+shift+escape` |
| 保存 | `ctrl+s` | 确认 / 取消 | `enter` / `escape` |

标点键也支持（`` ctrl+` ``、`ctrl+-` 等）。

## 十、注意事项

- 用**用户会话 Agent**（`-bestf` / `-hp` 后缀），它有桌面环境
- 截图用 GDI BitBlt，与远程桌面共存不冲突；失败时自动回落 DXGI，返回的 `capture_method` 说明画面来自哪条路径
- 每次 L3 操作后用 `ui_wait` 确认（L1），不要用截图确认
- Computer Use 是**兜底方案**：能用 `run_safe_command` / `execute_select` / `proxy_api_get` / `read_file` 解决的，不要用 GUI 操控
