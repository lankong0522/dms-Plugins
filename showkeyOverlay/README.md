# Keyboard Monitor

Keyboard Monitor 是一个 DMS 桌面组件插件，用于监视键盘输入，并在桌面上以可配置的悬浮卡片形式显示最近按下的按键。它适合录屏演示、操作教学、快捷键展示和日常键盘输入可视化。

该插件基于 `showmethekey-cli` 获取 Wayland 环境下的键盘事件，并通过 DMS desktop widget 在桌面上显示按键历史。插件采用事件驱动方式更新显示内容，不依赖 Desktop Command 的定时轮询机制。

## 主要功能

- 显示最近按下的键盘按键。
- 支持组合键显示，例如 `Ctrl + T`、`Alt + Tab`、`Super + Enter`。
- 单独按下 `Ctrl`、`Alt`、`Shift`、`Super`、`Compose` 时默认不单独显示，但会参与组合键显示。
- 每个按键提示以独立卡片显示。
- 支持多屏显示：每个显示器可以放置一个组件，只有一个实例负责监听键盘，其他实例同步显示共享状态。
- 支持配置显示位置：左上、右上、左下、右下。
- 支持控制新按键是否靠近所选角落显示。
- 支持设置显示时间、最大显示行数、去重间隔、字体大小、边距、卡片间距、卡片内边距。
- 支持使用 DMS 主题主色作为文字颜色。
- 支持文字描边，提升浅色背景下的可读性。
- 支持卡片背景、卡片边框、边框透明度、边框厚度和圆角设置。

## 依赖

### 必需依赖

- DMS / DankMaterialShell
- Quickshell
- Python 3
- `showmethekey-cli`
- `pkexec`

### Arch Linux 安装参考

```bash
sudo pacman -S python polkit
sudo pacman -S showmethekey
```

如果 `showmethekey` 不在你的源中，请使用你当前系统中已经安装的 `showmethekey` 包或 AUR 包。

### 权限说明

`showmethekey-cli` 在 Wayland 下需要读取输入设备事件，因此启动时可能会触发 `pkexec` 权限认证。插件本身不保存长期按键历史，只在当前会话的运行时目录中维护临时状态。

## 文件结构

插件目录建议放在：

```text
~/.config/DankMaterialShell/plugins/showkeyOverlay/
```

目录结构：

```text
showkeyOverlay/
├── plugin.json
├── metadata.json
├── Widget.qml
├── Settings.qml
├── showkeyStreamer.py
└── README.md
```

各文件作用：

| 文件 | 作用 |
|---|---|
| `plugin.json` | DMS 插件清单，定义插件 ID、名称、描述、类型、组件入口和设置入口。 |
| `metadata.json` | 插件元数据，通常与 `plugin.json` 保持一致，用于兼容 DMS 插件识别逻辑。 |
| `Widget.qml` | 插件显示层，负责接收按键状态、渲染桌面卡片、处理布局位置和样式。 |
| `Settings.qml` | 插件设置页，提供显示时间、字体、位置、背景、边框等可调参数。 |
| `showkeyStreamer.py` | 事件流脚本，负责调用 `showmethekey-cli`、解析键盘事件、维护按键历史和多屏共享状态。 |
| `README.md` | 插件说明文档。 |

## 运行时中间文件

插件会在 `$XDG_RUNTIME_DIR` 下创建运行时文件。该目录通常是：

```text
/run/user/<uid>
```

例如：

```text
/run/user/1000
```

运行时文件包括：

```text
$XDG_RUNTIME_DIR/showkey-overlay-dms-state.json
$XDG_RUNTIME_DIR/showkey-overlay-dms-streamer.lock
```

说明：

| 文件 | 作用 |
|---|---|
| `showkey-overlay-dms-state.json` | 多屏共享状态文件。leader 实例把当前按键显示内容写入该文件，follower 实例读取它并同步显示。 |
| `showkey-overlay-dms-streamer.lock` | leader 锁文件，用于保证只有一个插件实例真正启动 `showmethekey-cli` 监听键盘。 |

这些文件位于运行时目录中，属于临时文件。正常情况下，注销、重启或关机后会自动清理。

## 多屏工作机制

多屏显示不是让一个组件跨越所有屏幕，而是：

```text
每个屏幕放置一个 Keyboard Monitor 组件
        ↓
第一个获得锁的组件成为 leader
        ↓
leader 启动 showmethekey-cli 并监听键盘事件
        ↓
leader 将当前按键状态写入运行时状态文件
        ↓
其他组件作为 follower 读取同一个状态文件并同步显示
```

因此，多屏使用时可以在每个显示器上分别添加一个组件。正常情况下只应有一个 `showmethekey-cli` 进程在运行。

检查命令：

```bash
pgrep -af "showkeyStreamer.py|showmethekey-cli"
```

如果出现多个 `showmethekey-cli`，可以清理后重启 DMS：

```bash
pkill -f showkeyStreamer.py
pkill -f showmethekey-cli
dms restart
```

## 设置项说明

### Runtime

| 设置项 | 含义 | 推荐值 |
|---|---|---|
| `Entry lifetime` | 每条按键提示停留时间，单位为秒。 | `1.5` 到 `1.8` |
| `Maximum rows` | 最多同时显示的按键条数。 | `5` 到 `6` |
| `Deduplicate interval` | 相同按键在极短时间内重复出现时的去重时间，单位为秒。 | `0.05` |

### Layout

| 设置项 | 含义 |
|---|---|
| `Overlay position` | 控制组件内按键卡片的锚定位置：左上、右上、左下、右下。 |
| `Newest entry near selected corner` | 控制最新按键是否靠近所选角落。开启后，右上角模式下最新按键在最上方；右下角模式下最新按键在最下方。 |
| `Horizontal margin` | 水平方向边距。 |
| `Vertical margin` | 垂直方向边距。 |

### Text

| 设置项 | 含义 | 推荐值 |
|---|---|---|
| `Font size` | 按键文字大小。 | `32` 或 `34` |
| `Use DMS primary color for text` | 使用 DMS 当前主题主色作为文字颜色。 | 开启 |
| `Enable text outline` | 为文字添加描边，提升浅色背景下的可读性。 | 开启 |
| `Text outline opacity` | 文字描边透明度，数值越大越明显。 | `80` |

### Card

| 设置项 | 含义 | 推荐值 |
|---|---|---|
| `Enable key box background` | 是否启用按键卡片背景。 | 按需开启 |
| `Key box opacity` | 卡片背景透明度。 | `20` 到 `35` |
| `Enable key box border` | 是否启用卡片边框。 | 开启 |
| `Key box border opacity` | 卡片边框透明度。 | `70` 到 `90` |
| `Key box border thickness` | 卡片边框厚度，单位为像素。 | `1` 到 `3` |
| `Key box corner radius` | 卡片圆角大小。 | `10` 到 `14` |
| `Card horizontal padding` | 卡片内部左右留白。 | 按需调整 |
| `Card vertical padding` | 卡片内部上下留白。 | 按需调整 |
| `Card spacing` | 多个按键卡片之间的间距。 | 按需调整 |

## 推荐配置

日常使用：

```text
Entry lifetime: 1.5
Maximum rows: 5
Deduplicate interval: 0.05
Font size: 30 或 32
Use DMS primary color for text: 开启
Enable text outline: 开启
Text outline opacity: 80
Enable key box background: 开启
Key box opacity: 20 到 25
Enable key box border: 开启
Key box border opacity: 70 到 80
Key box border thickness: 1 到 2
Key box corner radius: 12
```

录屏演示：

```text
Entry lifetime: 1.8
Maximum rows: 6
Deduplicate interval: 0.05
Font size: 32 或 34
Use DMS primary color for text: 开启
Enable text outline: 开启
Text outline opacity: 80
Enable key box background: 开启
Key box opacity: 28 到 35
Enable key box border: 开启
Key box border opacity: 80 到 90
Key box border thickness: 2
Key box corner radius: 12 到 14
```

右上角从上向下显示：

```text
Overlay position: Top right
Newest entry near selected corner: 开启
```

右下角从下向上显示：

```text
Overlay position: Bottom right
Newest entry near selected corner: 开启
```

## 安装方式

将插件目录放入：

```bash
~/.config/DankMaterialShell/plugins/showkeyOverlay
```

确保脚本可执行：

```bash
chmod +x ~/.config/DankMaterialShell/plugins/showkeyOverlay/showkeyStreamer.py
```

重启 DMS：

```bash
dms restart
```

然后在 DMS 的插件或桌面组件管理界面中添加 `Keyboard Monitor`。

## 清理运行时文件

如果需要手动清理：

```bash
pkill -f showkeyStreamer.py
pkill -f showmethekey-cli
rm -f "$XDG_RUNTIME_DIR/showkey-overlay-dms-state.json"
rm -f "$XDG_RUNTIME_DIR/showkey-overlay-dms-streamer.lock"
dms restart
```

不要删除整个 `$XDG_RUNTIME_DIR` 或 `/run/user/<uid>`，其中还包含 Wayland、D-Bus、PipeWire 等桌面会话正在使用的运行时文件。

## 安全说明

该插件运行期间会监听全局键盘输入，用于显示最近按键。插件不会将按键历史写入长期文件，也不会主动上传数据。运行时状态只保存在 `$XDG_RUNTIME_DIR` 下的临时文件中。

在输入密码、密钥、令牌等敏感信息前，建议临时关闭插件或停止相关进程：

```bash
pkill -f showkeyStreamer.py
pkill -f showmethekey-cli
```
