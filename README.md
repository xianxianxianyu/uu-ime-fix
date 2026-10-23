# uu-ime-fix

> 解决 UU 远程控制 macOS 时「打拼音只出字母、没有候选框」的问题。
> 一条命令修好：`ime` / `ime wx`

![platform](https://img.shields.io/badge/macOS-12%2B-111827?style=flat-square&logo=apple&logoColor=white)
![shell](https://img.shields.io/badge/bash-zsh-4EAA25?style=flat-square&logo=gnubash&logoColor=white)
![deps](https://img.shields.io/badge/deps-none-brightgreen?style=flat-square)
![license](https://img.shields.io/badge/license-MIT-blue?style=flat-square)

## 问题

用 UU 远程连到 Mac mini，键盘能打英文，但打拼音只出字母：

```
nihao        ← 想打「你好」，结果是一串字母
```

同时候选框完全不出现。而人坐在那台机器前直接打字，一切正常。

## 原因

**不是输入法坏了，是系统当前输入源掉回英文布局了。**

macOS 的输入法有两个独立状态，很容易混淆：

| 状态 | 存储位置 | 作用 | 错了会怎样 |
|---|---|---|---|
| **启用** | `AppleEnabledInputSources` | 能否出现在切换循环里 | 列表里看不到这个输入法 |
| **选中** | `AppleSelectedInputSources` | 按键是否真的交给输入法 | 打出来是字母，没有候选框 |

启用 ≠ 选中。把微信输入法加进启用列表，只代表「按 Caps Lock 能切到它」，
不代表按键会进去。真正决定按键流向的是**选中**状态。

对着英文布局打字时，按键不经过任何输入法，作为字面字符直接送进 App —— 所以
既没有候选框，也组不了词。症状就是"能打英文，中文出不来"。

## 修复

把系统当前输入源切换成中文输入法。这等价于点菜单栏的输入法图标：

```swift
TISSelectInputSource(inputSource)   // Carbon / Text Input Source Services
```

### 为什么不用重启输入法

重启输入法进程**没有用**。实测：微信输入法进程重启后，候选框依然不出现 —— 因为
问题从来不在进程，而在"这个输入法压根不是当前选中的那个"。

### 为什么不用 Caps Lock

Caps Lock 在 macOS 上默认绑定的是「切换到 ABC」（英文布局），按它可能正好切到
**英文**，反而更打不出中文。而且它在不同 App 里表现不一致。

## 安装

```sh
git clone https://github.com/xianxianxianyu/uu-ime-fix.git
cd uu-ime-fix
mkdir -p ~/bin
install -m 755 fix-ime ~/bin/fix-ime
```

首次运行会自动用 `swiftc` 编译一个小 helper 缓存到 `~/bin/.imeselect`
（约 10 秒，之后启动是毫秒级）。

### 配置 alias

**zsh**（macOS 默认）：

```sh
echo "alias ime='~/bin/fix-ime'" >> ~/.zshrc
source ~/.zshrc
```

**bash**：

```sh
echo "alias ime='~/bin/fix-ime'" >> ~/.bashrc
source ~/.bashrc
```

**不用 alias，直接用可执行文件**（任意 shell，不依赖配置）：

```sh
~/bin/fix-ime          # 切到系统拼音
~/bin/fix-ime wx       # 切到微信输入法
```

> 也可以把 `~/bin/fix-ime` 重命名成 `~/bin/ime`，这样连 alias 都不用配，
> 只要 `~/bin` 在 `PATH` 里就能直接敲 `ime`。

### 关于 PATH

确认 `~/bin` 在 `PATH` 里：

```sh
echo $PATH | tr ':' '\n' | grep "$HOME/bin"
```

若没有，在 `~/.zshrc` 追加：

```sh
export PATH="$HOME/bin:$PATH"
```

## 用法

```sh
ime                 # 切到系统拼音（默认，最稳）
ime wx              # 切到微信输入法
ime abc             # 切到英文 ABC
ime -s              # 只看当前状态
ime -l              # 列出已启用且可切换的输入源
ime -n              # dry-run，只报告不切换
ime --target-app TextEdit   # 先激活该 App 再切（缓解按 App 记忆）
ime -S              # 打印内嵌的 Swift 源码
ime -h              # 帮助
```

### 典型流程

```sh
$ ime -s
当前输入源: 英文 ABC  (com.apple.keylayout.ABC)
  ⚠️  英文布局 — 这就是打不出中文的原因。跑 `ime` 或 `ime wx` 修复。

$ ime
英文 ABC → 系统拼音
  ✅ 现在是 系统拼音
```

## 一个必须知道的限制

**输入法状态是「按 App 分别记录」的。**

`ime` 改的是系统当前输入源。如果目标 App 仍然用它自己记住的那个，切过去后
那个窗口可能还是不出中文。三种应对，任选：

1. 在那个窗口里按一次 **Caps Lock** 或 **Ctrl+Space**
2. 点**菜单栏的输入法图标**手动选
3. `ime --target-app <App名>` —— 先把目标 App 激活，再切换

脚本会检测这种情况并以 **exit 4** 退出，明确告诉你「切换被前台 App 的记忆改回了」，
而不是假装成功。

## 退出码

| 码 | 含义 |
|---|---|
| `0` | 成功 |
| `1` | 失败（缺 swiftc、编译失败、找不到该输入源） |
| `3` | 切换后状态与目标不符 |
| `4` | 切换被前台 App 的记忆改回 |
| `64` | 参数错误 |

适合放进脚本里判断：

```sh
ime wx || echo "切换失败，请手动按 Caps Lock"
```

## 如何确认修复有效

在**原生 App** 里测最可靠（TextEdit、备忘录），第三方 App 的输入法上下文行为不一，
容易误判：

```sh
open -a TextEdit
# 在里面按 Caps Lock 切到中文，打 nihao，应出现候选框
```

## 内嵌 Swift 源码

`fix-ime` 是单文件自包含的：Swift helper 的源码以 heredoc 内嵌，`ime -S` 可查看。
需要单独编译：

```sh
ime -S > imeselect.swift
swiftc -O -o imeselect imeselect.swift
./imeselect --current   # 打印当前输入源 ID
```

## 环境要求

- macOS 12+
- `swiftc`（Xcode Command Line Tools：`xcode-select --install`）
- 无第三方依赖，不用 Homebrew
- 不需要 sudo

## 相关背景

- [Text Input Source Services](https://developer.apple.com/documentation/carbon/text_input_source_services)
  —— `TISSelectInputSource`、`TISGetInputSourceProperty` 等 API
- `~/Library/Preferences/com.apple.HIToolbox.plist` —— 输入源配置的实际存储位置

## 免责与范围

本工具只处理「系统当前输入源是英文布局」这一种原因导致的输入法失效。

以下情况不在范围内：

- 输入法 App 自身崩溃或卡死（这种情况才需要重启进程）
- 系统级键鼠事件被拦截（如辅助功能权限缺失）
- 第三方 App 未正确实现 NSTextInputClient

## License

MIT
