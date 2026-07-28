# FerrumPix macOS 快捷键核实

核实日期：2026-07-28

## 结论

macOS 适配不能把所有 `KeyModifiers.Control` 机械替换为 `KeyModifiers.Meta`。

- 对“新建、打开、保存、另存、打印、撤销、重做、剪切、复制、粘贴、全选、查找”这类标准命令，应在 macOS 使用 Command 体系。
- 对 FerrumPix 自定义命令，需要逐项检查是否占用了 macOS 标准快捷键。当前至少有 `Ctrl+Q` 收藏、图库 `Ctrl+W` 应用过滤器、`Ctrl+R` 调整尺寸、图库 `Ctrl+D` 转换，以及多组编辑器工具键，不能直接变成对应的 Command 组合。
- 鼠标多选是一个独立的语义：macOS 应使用 Command-单击切换单项、Shift-单击扩展范围。Control-单击是系统约定的辅助单击/上下文菜单，不应继续承担多选。
- 事件处理代码中的 Avalonia `KeyModifiers.Control` 表示物理 Control，`KeyModifiers.Meta` 才表示 macOS Command。只有通过字符串解析的 `KeyGesture`/`HotKey` 中的 `Ctrl` 才会由 Avalonia 在 macOS 自动映射成 Command，不能把这条规则套到当前手写的 `KeyEventArgs` 判断上。

Apple HIG 的原则是：不要把标准快捷键改作无关操作；自定义快捷键优先以 Command 为主修饰键，Shift 用于相关命令的变体，Option 只用于较少使用的命令，并尽量避免把 Control 用作自定义修饰键。[Apple HIG: Keyboards](https://developer.apple.com/design/human-interface-guidelines/keyboards/)

## Avalonia 按键模型

Avalonia 官方 macOS 指南给出的映射是：

| Avalonia 修饰键 | macOS 实际按键 |
| --- | --- |
| `KeyModifiers.Meta` / `Meta` | Command（⌘） |
| `KeyModifiers.Control` / `Control` | Control（⌃） |
| `KeyModifiers.Alt` / `Alt` | Option（⌥） |
| `KeyModifiers.Shift` / `Shift` | Shift（⇧） |

来源：[Avalonia macOS platform guide](https://docs.avaloniaui.net/docs/platform-specific-guides/macos#keyboard-shortcuts)

Avalonia 的 `KeyGesture`/`HotKey` 字符串另有一层跨平台映射：例如 XAML 中的 `Ctrl+S` 在 macOS 会按 Command-S 工作；官方文档也建议通过 `PlatformSettings.HotkeyConfiguration` 查询复制等平台热键。FerrumPix 当前主要是手写 `KeyDown`/`Pointer` 处理，因此不能依赖这层自动映射。[Avalonia: Keyboard and hotkeys](https://docs.avaloniaui.net/docs/input-interaction/keyboard-and-hotkeys#common-modifier-keys) [Avalonia macOS platform conventions](https://docs.avaloniaui.net/docs/platform-specific-guides/macos#platformhotkeyconfiguration)

## 仓库现状

当前实现集中在：

- `Views/MainWindow.axaml.vb`：全局评分/收藏、打印、尺寸调整、全屏、图库剪切复制粘贴、编辑器保存。
- `Views/GalleryView.axaml.vb`：图库选择、剪切复制粘贴、搜索、应用过滤器、转换、导航、删除、重命名、刷新。
- `Views/ViewerView.axaml.vb`：旋转、信息、进入编辑、缩放、导航、删除、重命名、幻灯片。
- `Views/EditorView.axaml.vb`：保存、撤销/重做、新建、选择/剪贴板、复制对象、尺寸工具、旋转、工具切换、分组等。
- `Views/LayersPanelView.axaml.vb`：图层复制、Command/Control-单击式多选、分组文案。
- `Views/SettingsView.axaml`、各工具提示和图层菜单：大量写死的德语 `Strg+…` 显示文案。

另有三个容易漏掉的非键盘主处理路径：

- 图库、图层和画布对象的 `Control`-单击多选。
- 图库、查看器和编辑器的 `Control`-滚轮缩放。
- 文本对象的 `Control+Enter`/`Shift+Enter` 换行。

## 应直接采用 macOS 标准语义的命令

| 操作 | macOS 建议 | 当前 FerrumPix | 处理建议 |
| --- | --- | --- | --- |
| 新建 | ⌘N | Ctrl+N | macOS 改为 Meta+N |
| 打开 | ⌘O | 没有统一全局键 | 如提供“打开”命令，应使用 ⌘O |
| 保存 | ⌘S | Ctrl+S | macOS 改为 Meta+S |
| 另存为 | ⇧⌘S | Ctrl+S 在不能原位保存时隐式进入“另存” | 增加明确的 ⇧⌘S；保留 ⌘S 的现有降级行为也可以 |
| 打印 | ⌘P | Ctrl+P | macOS 改为 Meta+P |
| 撤销 | ⌘Z | Ctrl+Z | macOS 改为 Meta+Z |
| 重做 | ⇧⌘Z | Ctrl+Y | macOS 必须改为 Shift+Meta+Z，不应使用 ⌘Y |
| 剪切/复制/粘贴 | ⌘X / ⌘C / ⌘V | Ctrl+X/C/V | macOS 改为 Meta；优先使用 Avalonia 平台热键配置 |
| 全选 | ⌘A | Ctrl+A | macOS 改为 Meta+A，同时继续避开文本输入框的自定义处理 |
| 查找 | ⌘F | Ctrl+F | macOS 改为 Meta+F |
| 设置 | ⌘, | 目前设置按钮，无标准键 | 增加 Meta+Comma，并放入原生 App 菜单 |
| 关闭窗口 | ⌘W | 无显式键；图库 Ctrl+W 被过滤器占用 | ⌘W 必须归还“关闭窗口” |
| 退出应用 | ⌘Q | Ctrl+Q 被收藏占用 | ⌘Q 必须由原生菜单退出并进入现有 Closing 流程 |
| 全屏 | ⌃⌘F | F11 | macOS 以 Control+Meta+F 为主；F11 可仅作兼容辅助键 |

上述通用组合由 Apple 官方列表定义；Avalonia 的 macOS 指南也明确列出 ⌘Q、⌘W、⌘M、⌃⌘F、⌘A、⌘F。[Apple: Mac keyboard shortcuts](https://support.apple.com/en-gb/102650) [Avalonia macOS platform conventions](https://docs.avaloniaui.net/docs/platform-specific-guides/macos#standard-shortcuts)

`F11` 不适合作为 macOS 唯一全屏入口：Apple 键盘顶排默认控制系统功能，应用要收到标准 F 键通常需要同时按 Fn/Globe；Apple 的标准应用全屏组合是 Control-Command-F。[Apple: Use function keys on Mac](https://support.apple.com/en-ie/102439) [Apple: Mac keyboard shortcuts](https://support.apple.com/en-gb/102650)

## 图片应用可直接借鉴的 macOS 行为

Apple Photos 是 FerrumPix 最接近的一方参考：

| 操作 | Apple Photos | FerrumPix macOS 建议 |
| --- | --- | --- |
| 收藏 | `.` | 用 `.` 代替当前 Ctrl+Q；文本输入框内不拦截 |
| 逆时针/向左旋转 | ⌘R | 查看器和编辑器采用 Meta+R |
| 顺时针/向右旋转 | ⌥⌘R | 查看器和编辑器采用 Alt+Meta+R |
| 显示信息 | ⌘I | 查看器采用 Meta+I |
| 进入/退出编辑 | Return | 查看器可用 Return 进入编辑；编辑器 Return 的文本/确认上下文需优先 |
| 放大/缩小 | ⌘+ / ⌘- 或触控板捏合 | 增加 Meta+Plus/Minus；不要只依赖 Control-滚轮 |
| 选择非相邻照片 | Command-单击 | 图库、画布对象、图层都改为 Meta-单击 |
| 连续范围选择 | Shift-单击 | 保持 Shift-单击 |
| 全屏 | ⌃⌘F | 采用 Control+Meta+F |
| 删除选中项 | Delete；无确认删除为 ⌘Delete | 先验证 Avalonia 在 MacBook Delete 键上产生 `Key.Back` 还是 `Key.Delete`，再绑定语义 |
| 视频播放/暂停 | ⌥Space | 查看器可考虑 Alt+Space；幻灯片进行中用 Space 暂停/继续仍符合 Photos |

来源：[Apple Photos: Keyboard shortcuts and gestures](https://support.apple.com/en-ae/guide/photos/pht9b4411b24/mac)

Apple Preview 的图片查看快捷键还提供了可选的缩放先例：⌥⌘0 为实际大小、⌥⌘9 为适合窗口、⌥⌘+/- 为所有图像缩放。FerrumPix 可以选择更接近 Photos 的 ⌘+/-，并将“100%/适合窗口”设计成不冲突的明确组合。[Apple Preview: Keyboard shortcuts](https://support.apple.com/en-gb/guide/preview/cpprvw0003/mac)

## 绝不能机械替换的现有 Ctrl 组合

| 当前组合与动作 | 如果改成 Command 的冲突 | 建议 |
| --- | --- | --- |
| Ctrl+Q 收藏 | ⌘Q 是退出应用 | macOS 收藏改为 `.`；绝不拦截 ⌘Q |
| 图库 Ctrl+W 应用过滤器 | ⌘W 是关闭活动窗口 | ⌘W 归还关闭；过滤器保留菜单入口，另选不冲突的应用级键 |
| Ctrl+R 调整图像尺寸 | Photos 中 ⌘R/⌥⌘R 是左右旋转；部分应用 ⌘R 是刷新 | macOS 不把尺寸调整放到 ⌘R；可暂不设键或另选自定义键 |
| 图库 Ctrl+D 转换 | Finder 和 Photos 中 ⌘D 是复制/副本 | 若语义接近导出，可考虑 Photos 的 ⇧⌘E；否则暂保留菜单入口 |
| 编辑器 Ctrl+Y 重做 | macOS 标准重做是 ⇧⌘Z | 改为 Shift+Meta+Z |
| 编辑器 Ctrl+T 文本工具 | ⌘T 是新标签页/显示标签栏的常见标准 | 不映射；画布无文本输入焦点时可考虑无修饰键 `T` |
| 编辑器 Ctrl+M 插入工具 | ⌘M 是最小化窗口 | 不映射；改为画布作用域无修饰工具键或菜单 |
| 编辑器 Ctrl+I 插入图片 | ⌘I 通常是信息/斜体；FerrumPix 查看器也应以它显示信息 | 不映射 |
| 编辑器 Ctrl+K 插入 QR | ⌘K 在系统/应用中已有连接、链接或关键词等常见语义 | 不映射 |
| 编辑器 Ctrl+E 橡皮 | Photos 中 ⌘E 是自动增强，Finder 中是弹出磁盘 | 不映射；画笔工具激活时使用无修饰 `E` 更合理 |
| Ctrl+左右箭头旋转 | macOS 中 Command/Option/Control+箭头承担文本、焦点、导航等语义 | 改用 Photos 的 ⌘R / ⌥⌘R |
| Ctrl+1…5 评分、Ctrl+0 清除 | Photos/Finder 的 ⌘1…4 已用于视图切换 | 不改为 Command 数字；见下一节 |
| Ctrl+Enter 换行 | Command-Return 常被用于提交/确认，不是通用换行标准 | macOS 主提示只显示 Shift-Return；可保留物理 Control-Return 兼容，不改成 Command |

标准组合和冲突依据：[Apple HIG: Keyboards](https://developer.apple.com/design/human-interface-guidelines/keyboards/) [Apple: Mac keyboard shortcuts](https://support.apple.com/en-gb/102650) [Apple Photos shortcuts](https://support.apple.com/en-ae/guide/photos/pht9b4411b24/mac)

## 评分、自定义工具和滚轮

### 评分

Apple Photos 没有星级评分标准键，因此这里不存在可照搬的 macOS 系统约定。不能把 Ctrl+数字改成 Command+数字，因为 Photos 用 ⌘1/2/3 切换照片视图，Finder 用 ⌘1…4 切换视图。

建议分两步：

1. 本轮保留物理 Control+0…5 作为兼容键，在 macOS 文案中准确显示为 `⌃0…5`，不要伪装成 Command。
2. 后续若要进一步原生化，再考虑可配置的无修饰 `0…5`；但 FerrumPix 查看器当前裸 `0` 是“适合窗口”，必须先解决该冲突。

### 编辑器工具

Apple Photos 在编辑视图中直接使用无修饰字母激活工具（例如 A 调整、F 过滤、C 裁剪、R 清理），这比把 FerrumPix 的所有 Ctrl 工具键改成 Command 更接近 macOS 图片应用。

建议：

- 标准文档命令使用 Command。
- 只有编辑画布获得焦点、且焦点不在 TextBox/输入控件时，才处理无修饰工具字母。
- 对象复制可用 ⌘D，因为它与 Photos/Finder 的“创建副本”语义一致。
- 分组/取消分组、插入图片、插入 QR 等没有 Apple 图片应用标准的动作，先保留菜单/按钮或经过单独冲突审查后再设键。

### 滚轮和触控板

Apple Photos 的主要图片缩放方式是触控板捏合或 ⌘+/-，不是 Control-滚轮。Control 还被 macOS 多项系统级功能使用，HIG 建议避免把它作为自定义主修饰键。

本轮至少应增加 ⌘+/-；Control-滚轮可以暂时保留为鼠标兼容路径，但不应作为 macOS 帮助页的首要操作。触控板捏合是否能由 Avalonia 当前版本正确转成缩放事件，需要在真机单独验证，不能从键盘映射推断。[Apple Photos: View photos and videos](https://support.apple.com/en-ie/guide/photos/pht8b9ba4aa9/mac) [Apple HIG: Keyboards](https://developer.apple.com/design/human-interface-guidelines/keyboards/)

## Delete、Backspace、Fn 与功能键

MacBook 键盘上标为 Delete 的键执行“向后删除”；向前删除通常是 Fn-Delete。Apple 明确区分 Delete 和 Forward Delete。[Apple: Intro to Mac keyboard shortcuts](https://support.apple.com/en-ca/guide/mac-help/mchld6b9e240/mac)

FerrumPix 当前文件/对象删除只判断 Avalonia `Key.Delete`，而查看器返回/退出全屏还判断 `Key.Back`。因此不能在没有实机事件记录的情况下直接增加 `Key.Back` 删除，否则可能改变现有“返回”流程。实施时应在 macOS 真机记录紧凑键盘 Delete、Fn-Delete、外接键盘 Forward Delete 各自产生的 Avalonia `Key`，然后在非文本输入上下文按语义分流。

同理，F2/F3/F5/F7/F11 在 Apple 键盘上可能需要 Fn/Globe 才作为标准功能键送到应用。它们可以作为兼容入口，但不应成为 macOS 唯一入口：

- F11 全屏：增加 ⌃⌘F。
- F5 刷新：保留按钮/菜单；不要改成 ⌘R，因为图片上下文中它应优先用于旋转。
- F2 重命名：保留按钮/菜单，真机帮助文案可注明 Fn-F2；不要为了“原生”抢占 Return，因为 FerrumPix/Photos 的 Return 还承担进入编辑。
- F3/F7 搜索：增加标准 ⌘F。

## 推荐实施顺序

1. 新增平台快捷键策略层，把“标准主修饰键”“物理 Control”“多选修饰键”分成不同语义，避免一个 `IsPrimaryModifier` 覆盖所有行为。
2. 先落地无争议标准：⌘N/O/S、⇧⌘S、⌘P、⌘Z、⇧⌘Z、⌘X/C/V/A/F、⌘,、⌘W、⌘Q、⌃⌘F。
3. 落地图片应用先例：`.` 收藏、⌘R/⌥⌘R 旋转、⌘I 信息、⌘+/- 缩放、Command-单击多选。
4. 明确保留而不伪装的平台差异：评分暂为物理 Control+数字；Control-Return 仅兼容，帮助页主推 Shift-Return。
5. 暂缓或重新分配冲突项：过滤器 Ctrl+W、尺寸 Ctrl+R、转换 Ctrl+D、编辑器工具 Ctrl+T/M/I/K/E、Control+箭头旋转。
6. 用平台化属性或转换器生成 Settings/ToolTip/ContextMenu 的快捷键文案，macOS 显示 `⌘`、`⌥`、`⇧`、`⌃`，Windows/Linux 保持现有 `Strg` 文案和行为。
7. macOS 真机记录并核实 Delete/Fn-Delete、功能键、触控板捏合、Command-Q/Command-W 与现有 Closing/未保存确认流程。

## 实机验收重点

- ⌘Q 退出、⌘W 关闭都进入现有未保存确认和资源清理流程；`.` 收藏不会退出应用。
- ⌘S 与 ⇧⌘S 分别保存/另存；⌘Z 与 ⇧⌘Z 分别撤销/重做。
- TextBox 中 ⌘A/C/X/V/Z、Option+箭头等仍由控件正常处理，不被窗口级 handler 抢走。
- Command-单击切换图库、画布对象和图层的离散选择；Control-单击打开上下文菜单。
- ⌘R/⌥⌘R 左右旋转，尺寸调整不会占用 ⌘R。
- ⌃⌘F 全屏；F11 仅作为辅助入口，按 Esc 后原生标题栏恢复。
- MacBook Delete、Fn-Delete、外接 Forward Delete 的 Avalonia 键值和删除结果明确，任何删除都保持现有确认流程。
- Windows/Linux 原有 Ctrl、F 键、Delete、窗口与选择行为不变。
