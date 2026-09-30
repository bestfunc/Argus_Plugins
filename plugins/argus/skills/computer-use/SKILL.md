---
name: computer-use
display_name: 远程操控
description: 通过 Argus Remote Computer Use 远程操控宿主机 GUI。**优先用语义层**（ui_snapshot 列元素 → ui_act 按元素引用操作 → ui_wait 等状态），不需要看图算坐标；网页内容走 browser_snapshot / browser_act（CDP）；自绘界面才退回截图 + 坐标点击，坐标流程用 computer_batch 一次审批跑完。感知类（ui_snapshot / 截图 / 读剪贴板 / ui_wait / browser_snapshot）是 L1 直接可用；执行类（ui_act / click / drag / key / type_text / computer_batch / browser_act）均为 L3，每次调用一次邮箱审批，务必用批量把一串动作压成一次，并事先与用户确认整体方案。
user-invocable: true
allowed-tools: mcp__argus__list_agents,mcp__argus__ui_snapshot,mcp__argus__ui_act,mcp__argus__ui_wait,mcp__argus__get_screen_info,mcp__argus__get_ui_elements,mcp__argus__set_clipboard,mcp__argus__scroll,mcp__argus__screenshot,mcp__argus__get_clipboard,mcp__argus__wait_stable,mcp__argus__drag,mcp__argus__click,mcp__argus__double_click,mcp__argus__right_click,mcp__argus__type_text,mcp__argus__key,mcp__argus__computer_batch,mcp__argus__browser_snapshot,mcp__argus__browser_screenshot,mcp__argus__browser_navigate,mcp__argus__browser_act,mcp__argus__get_machine_presence,mcp__argus__read_pixels
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
├── 返回 note 说"浏览器窗口" → 网页内容用 browser_snapshot / browser_act（CDP），别用截图硬点
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

> ⚠️ **这份清单默认不含弹出式窗口**：真 Win32 弹出菜单（class=`#32768`）、输入法候选窗、
> 无标题的模态提示框都被滤掉了——而它们恰恰是最需要自动化去点的一类。
> **「清单里没有」不等于「系统上没有」**，要操作它们加 `all=true`：
>
> ```
> ui_snapshot(agent_id, scope="windows", all=true)   # 连无标题/工具窗口一起列，行尾带 class=
> ```
>
> 返回里 `scanned` 是枚举到的窗口总数，和 `matched` 的差就是本次滤掉的量，
> `note` 会说明滤掉了哪几类（不可见 / DWM 隐藏 / 无标题 / 工具窗口）。
> 被 DWM 隐藏(cloaked)的幽灵窗口任何时候都不列——它们不在屏幕上，点了没有任何效果，
> 尤其 UWP 会留下一堆同名的隐藏 `ApplicationFrameWindow`。
>
> 曾经踩过的坑：菜单查不到被读成"树里没有"，差点变成"要给菜单写一套 provider"的工作量。
> 实际上 `#32768` 是独立顶层窗口，`all=true` 列出来拿到 hwnd 就能正常 snapshot / act，
> 而且走的是真 UIA pattern（`ui_act` 回执的 `did` 会显示 `ExpandCollapsePattern.Expand`），
> 不是退化的坐标点击。

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
| `click` / `double_click` / `right_click` / `drag` | L3 | 坐标点击（click 类支持 `modifiers`） |
| `type_text` / `key` | L3 | 键盘输入 |
| `computer_batch` | L3 | **一串坐标动作一次审批**，做完回一张截图（v1.42） |

以上执行类工具都可加 `return_screenshot=true`（v1.42）：做完顺带回一张截图，省掉一次 screenshot 往返；
`settle_ms`（默认 400）控制动作后等多久再截。截图失败只附 `screenshot_error`，**动作本身已经做了，不要重试**。

**坐标系统**：`click` 等的坐标基于最近一次 `screenshot` 的**图片坐标**，把那次截图返回的 `capture_meta` 一并传回来，Server 自动换算成屏幕坐标。

> ⚠️ `capture_meta` 里若出现 `scale_unknown: true`，说明该 Agent 版本旧、没回传真实分辨率，**这张图的坐标不能用于点击**（传给 click 会被直接拒绝）。先 `get_screen_info` 自己换算，或改用 region 模式截图（坐标 1:1）；根治办法是升级 Agent 到 v1.40+。

**归属看 `foreground`，不要看图**（v1.40.907+）：截图回执里的 `foreground` 是抓这张图那一刻的前台窗口（hwnd / pid / 进程名 / 类名 / 标题）。同款程序开多个实例时（标题相同、窗口位置重合），两张全屏截图**看起来完全一样**，「我的窗口没开出来」和「我的窗口在别人后面」在图上分不出来。点之前先核对 `foreground.hwnd` 是不是目标，不是就先激活——否则点击落在压在上面的那个窗口里，**而且不会报错**。

> 真实代价：曾经因为没核对归属，`Ctrl+Shift+N` 在别人的进程里开了一个新窗口，而截图上它看起来就像"我的第二个窗口开出来了"。发现它靠的是"这个新窗口不在我的日志里"，不是靠看。

**修饰键**（v1.40.907+）：`click` / `double_click` / `right_click` 支持 `modifiers=["ctrl"]`，点击期间按住。终端里的 OSC 8 超链接要 ctrl+click 才打开，普通点击无效；文件列表多选用 ctrl / shift。

```
click(agent_id, x=.., y=.., capture_meta=.., modifiers=["ctrl"])
```

> 回执里的 `modifiers` 是**实际按住的**。请求了却没回（老 Agent），返回里会带 `warning` 明说"这次是不带修饰键的普通点击"——别把它当成 ctrl+click 成功了。

**文字输入**：长文本/中文用 `set_clipboard`（L1）+ `key("ctrl+v")`（L3），比 `type_text` 快且不受输入法影响。但元素在语义层可见时，直接 `ui_act(action="set_value")` 更好。

**`key` 的返回值要看 `sent` 数组**：它列出实际注入的每个按键（含虚拟键码和扫描码）。`sent` 里没有的键就是没发出去的键——不要用 `success` 本身当作"热键已送达"的判据。被测对象若读的是扫描码（DirectInput / RawInput），加 `injection="scancode_only"`。

### 看不清就 zoom，别在缩略图上硬量（v1.42）

overview 截图是缩略图（约 960 宽），4K 屏上缩了 4 倍——小字、小图标、相邻按钮的边界都看不准，
**直接在缩略图上量坐标去点小目标很容易点偏，而且不报错**。先放大：

```
screenshot(agent_id)                                          # overview，拿到 capture_meta = M
screenshot(agent_id, zoom={"x":600,"y":300,"w":160,"h":90}, capture_meta=M)   # 那块区域的原始分辨率
click(agent_id, x=.., y=.., capture_meta=<放大图的 capture_meta>)             # 在放大图上量的坐标直接用
```

`zoom` 必须带被放大那张图的 `capture_meta`（不带直接拒绝），新图自带 capture_meta，精度是物理像素级。

### 多显示器（v1.42）

默认只截**主屏**；机器有多块屏时截图回执的 `note` 会提示。`get_screen_info` 的 `monitors` 列出各块屏
（主屏恒为 0，其余按位置排序），然后：

```
screenshot(agent_id, display=1)        # 第 1 块副屏
screenshot(agent_id, display="all")    # 整个虚拟桌面（缩得更狠，看细节要 zoom）
```

坐标系是虚拟桌面物理像素：主屏左上角为原点，摆在主屏左边 / 上面的副屏是**负坐标**。
这些都已算进 `capture_meta`，照常把它传给 click 即可，不用自己换算。老 Agent 不认识 `display`，
会报 `AGENT_TOO_OLD` 而不是静默回主屏。

### 坐标流程用 computer_batch 一次审批（v1.42）

「点输入框 → 输入 → 回车」这种只能靠坐标的流程（自绘界面、画布），过去是三次 L3、三封邮件、
再加一次截图确认。现在一次发完：

```
computer_batch(agent_id, capture_meta=M, actions=[
  {"type":"click", "x":412, "y":230},
  {"type":"type_text", "text":"X2411", "wait_ms":200},
  {"type":"key", "combo":"enter"},
  {"type":"wait", "ms":800}
], _approval_reason="在检测软件里输入工件号 X2411 并回车查询")
```

- `type` 支持 click / double_click / right_click / scroll / drag / type_text / key / ui_act（单个语义动作）/ wait；
  字段与同名单个工具相同，坐标统一按顶层 `capture_meta` 换算；最多 20 步，整批等待合计 ≤ 20 秒
- **执行前全部校验**：任何一步写错，一步都不执行；审批邮件逐条列出全部步骤，批准后改参数无效
- 默认做完回一张截图（`return_screenshot=false` 可关）
- 某步失败默认停下：看 `completed` / `failed_at`——**前面成功的步骤已经做了**，看截图确认现状，只补没完成的部分
- 语义层能定位的（按钮名、菜单项）仍然优先用 `ui_act` 的 `steps`；`computer_batch` 是给坐标场景的
- 借用的每种工具单独过权限：没开 `drag` 权限的人，批量里也不能拖拽

## 七、`get_ui_elements`（旧工具，保留）

v1.41 之前的 UI 树工具，只读不能执行，噪音多、没有 AutomationId。**新场景一律用 `ui_snapshot`**，它保留只是为了兼容既有流程。

## 八、何时用 computer-use vs remote-browser vs terminal

目标是**能用 API 就别用 GUI，能用语义层就别用坐标**：

| 目标 | 优先 | 次选 | 最后 |
|---|---|---|---|
| 跑命令 / 看日志 | `terminal`（run_safe_command L1） | `terminal`（run_command L3） | — |
| 访问 HTTP API | `api-query`（proxy_api_get L1） | `proxy_api` L2 | — |
| **操作网页** | `browser_snapshot` / `browser_act`（CDP，本 skill 第十一节） | `remote-browser`（隧道 + 本地脚本，要跑完整 Playwright 时） | `computer-use` 视觉层 |
| 配置工业软件（BestPLC / 扫描仪 / 自研 GUI） | `computer-use` **语义层** | `computer-use` 视觉层 | — |
| 填 Win32 对话框 / 老系统 | `computer-use` **语义层** | 视觉层 | — |

**网页内容一定走 browser_* 工具**：`ui_snapshot` 对浏览器窗口只能看到外壳（标签页、地址栏），网页内容拿不到（Chromium 的 a11y 树不向 UIA 暴露），snapshot 的 `note` 会明说这一点。别看到"元素列表里没有登录按钮"就以为页面上没有。

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

- 桌面在**用户会话 Agent**（`-bestf` / `-hp` 后缀）上。v1.42 起给了服务 Agent（不带后缀）也行：
  该设备只有一个在线会话时 Server 自动转过去，回执 `routed` 写明转给了谁（后续直接用 `routed.to`）；
  多个会话报 `AGENT_AMBIGUOUS` 列出候选；没人登录报 `SCREEN_UNAVAILABLE`（`reason=no_interactive_session`）
- **只支持 Windows**：Linux / macOS Agent 调 computer use 直接报 `PLATFORM_UNSUPPORTED`（重试不会好）；
  但 browser_* 工具不限平台
- 动手前可先 `get_machine_presence` 看键盘前有没有人、是否锁屏、是否有别的 AI 正在操作
- 截图用 GDI BitBlt，与远程桌面共存不冲突；失败时自动回落 DXGI，返回的 `capture_method` 说明画面来自哪条路径
- 每次 L3 操作后用 `ui_wait` 确认（L1），不要用截图确认
- Computer Use 是**兜底方案**：能用 `run_safe_command` / `execute_select` / `proxy_api_get` / `read_file` 解决的，不要用 GUI 操控

## 十一、网页内容：browser_* 工具（v1.42，CDP）

`ui_snapshot` 对浏览器只能看到外壳。网页里的按钮、表单、表格走这四个工具（Agent 经本机 loopback 连浏览器调试端口，
不需要建隧道、不需要本地脚本）：

| 工具 | 级别 | 用途 |
|---|---|---|
| `browser_snapshot` | L1 | `mode="elements"`（默认）列可交互元素，行首 `b3` 是 ref；`mode="text"` 读正文；`mode="tabs"` 列标签页 |
| `browser_screenshot` | L1 | 截网页视口本身：不受窗口遮挡 / 最小化 / 锁屏影响。图上坐标是 CSS 像素，**不能给 click** |
| `browser_navigate` | L2 | 打开网址（http/https/about:blank，或 back / forward / reload） |
| `browser_act` | L3 | 按 ref / selector 执行 click / set_value / select / check / uncheck / focus / scroll_into_view / press；`steps` 批量 ≤ 20 步一次审批 |

```
browser_snapshot(agent_id, tab="erp")                          # 按 url/标题子串选标签页
b3  input[text] 工件号 = ""
b4  select 产线 = "请选择"
b7  button 查询

browser_act(agent_id, tab="erp", steps=[
  {"ref":"b3", "action":"set_value", "value":"X2411"},
  {"ref":"b4", "action":"select", "value":"二线"},
  {"ref":"b7", "action":"click", "wait_ms":800}
], _approval_reason="在 ERP 查询页按工件号 X2411、二线查询")

browser_snapshot(agent_id, tab="erp", mode="text")             # 读查询结果
```

要点：

- **click 发的是真实鼠标事件**，点之前检查点击点有没有被挡：挡住了报 `ELEMENT_OBSCURED`，**没有点下去**
  （先关掉弹窗 / 遮罩；确认要点在遮挡物上才 `force=true`）
- **set_value / select / check 会读回确认**，读回不符报 `NOT_CONFIRMED`——那表示**已经执行、只是没生效**，
  先 snapshot 看现状，别盲目重试
- ref 在页面变化（跳转、弹窗、列表刷新）后会失效，报 `ELEMENT_NOT_FOUND` 时重新 snapshot
- 回执 `navigated` / `url_after` 说明这一下有没有触发跳转
- 密码框的值永远显示 `***`；跨域 iframe 看不到内容（`note` 会说明，不代表页面上没有）

**前提：浏览器要开调试端口**。没开时报 `CDP_UNAVAILABLE`，提示怎么开。现场启动方式：

```
chrome.exe --remote-debugging-port=9222 --user-data-dir=C:\chrome-argus
```

- **Chrome 136+ 对默认用户目录会忽略 `--remote-debugging-port`**，必须给一个非默认的 `--user-data-dir`——
  这是一个独立的浏览器配置，**原来的登录态不在里面，需要重新登录一次**，事先跟用户说清楚
- **不要加 `--remote-debugging-address=0.0.0.0`**：CDP 端口没有任何认证，谁连上谁就能完全控制这个浏览器；
  browser_* 工具只走本机 loopback，不需要对外开
- 端口不是 9222 / 9223 / 9229 时，用 `port` 参数显式指定
