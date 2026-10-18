# arch Linux 安装输入法教程

> 以 **CachyOS / Arch + KDE Plasma (Wayland)** 为例，从零安装 `fcitx5`，
> 并把 Rime 方案换成 **白霜拼音（rime-frost）+ 万象语法模型（RIME-LMDG / rime-wanxiang-data）**。
> 文中 shell 命令统一使用 **fish** 语法。

---

## 目录

1. [概念先理清](#1-概念先理清)
2. [安装 fcitx5 基础环境](#2-安装-fcitx5-基础环境)
3. [安装 Rime 引擎与插件](#3-安装-rime-引擎与插件)
4. [安装白霜拼音方案](#4-安装白霜拼音方案)
5. [安装万象语法模型](#5-安装万象语法模型)
6. [配置并部署](#6-配置并部署)
7. [验证是否成功](#7-验证是否成功)
8. [开机自启与环境变量](#8-开机自启与环境变量)
9. [日常更新与回滚](#9-日常更新与回滚)
10. [常见问题（踩坑记录）](#10-常见问题踩坑记录)

---

## 1. 概念先理清

| 名词 | 说明 |
|---|---|
| **fcitx5** | 输入法框架（负责跟系统、窗口、托盘打交道） |
| **fcitx5-rime** | fcitx5 的 Rime 引擎插件，把 Rime 接进 fcitx5 |
| **librime** | Rime 核心库，真正的输入算法 |
| **Rime 方案** | 一堆 `.yaml` 配置 + 词库，决定“怎么打字”。雾凇、白霜、万象都属于这一类 |
| **语法模型 `.gram`** | 八股文格式的语言模型，让整句输入更准。由 `librime-octagram` 插件加载 |

> 关键点：**换方案不等于换软件**。
> 雾凇 → 白霜 + 万象，本质上只替换 `~/.local/share/fcitx5/rime` 里的数据文件，
> 不需要卸载重装任何 pacman 包。

---

## 2. 安装 fcitx5 基础环境

```fish
sudo pacman -S --needed fcitx5 fcitx5-configtool fcitx5-qt fcitx5-gtk
```

- `fcitx5-configtool`：图形化配置（`fcitx5-configtool` 命令启动）
- `fcitx5-qt` / `fcitx5-gtk`：让 Qt / GTK 程序能接上 fcitx5

KDE 用户额外建议：

```fish
sudo pacman -S --needed fcitx5-qt6 fcitx5-gtk  # 视仓库情况，qt6 版可能已包含
```

---

## 3. 安装 Rime 引擎与插件

```fish
sudo pacman -S --needed fcitx5-rime librime librime-data
```

### 3.1 确认 octagram 插件存在

万象 `.gram` 模型必须靠 `librime-octagram` 才能加载。Arch 的 `librime` 包已经自带：

```fish
ls /usr/lib/rime-plugins/
```

应能看到（至少这几个）：

```
librime-charcode.so  librime-lua.so  librime-octagram.so  librime-predict.so  librime-proto.so
```

如果**没有** `librime-octagram.so`，说明你的发行版把插件拆包了，需要额外安装
`librime-plugin-octagram` / AUR 里的对应包，否则语法模型会静默失效。

---

## 4. 安装白霜拼音方案

白霜拼音没有官方包也没有 AUR 包，官方推荐直接放进 Rime 用户目录。

Rime 用户目录（fcitx5 版）：

```
~/.local/share/fcitx5/rime
```

### 4.1 备份旧方案（强烈建议）

```fish
cd ~/.local/share/fcitx5
set ts (date +%Y%m%d-%H%M)
mv rime "rime-backup-$ts"
```

> 想回滚时把备份目录改回 `rime` 即可。

### 4.2 拉取白霜

```fish
cd ~/.local/share/fcitx5
git clone --depth 1 https://github.com/gaboolic/rime-frost.git rime
```

白霜仓库自带全拼、自然码/小鹤/微软/搜狗双拼、五笔、九键等，默认方案是 `rime_frost`（全拼）。

也可以下载 release 里的 `rime-frost-schemas.zip` 解压到该目录，效果一样。

---

## 5. 安装万象语法模型

官方地址：<https://github.com/amzxyz/rime-wanxiang-data>
（旧地址 `amzxyz/RIME-LMDG` 会自动跳转到这里）

在 `LTS` release 里下载对应模型：

| 文件 | 用途 |
|---|---|
| `wanxiang-lts-zh-hans.gram` | 简体（约 380 MiB） |
| `wanxiang-lts-zh-hant.gram` | 繁体 |

```fish
cd ~/.local/share/fcitx5/rime
curl -L --retry 3 -C - -o wanxiang-lts-zh-hans.gram \
  https://github.com/amzxyz/rime-wanxiang-data/releases/download/LTS/wanxiang-lts-zh-hans.gram
```

**文件名不能随便改**：Rime 是按 `<grammar.language>.gram` 去找文件的，
即 `language: wanxiang-lts-zh-hans` ↔ 文件 `wanxiang-lts-zh-hans.gram`。

检查文件是否完整（大小应为 `398505004` 字节）：

```fish
stat -c '%s' wanxiang-lts-zh-hans.gram
xxd -l 16 wanxiang-lts-zh-hans.gram   # 应看到 Rime::Grammar/1.0
```

---

## 6. 配置并部署

### 6.1 写 `rime_frost.custom.yaml`

白霜仓库里 `grammar.language` 默认是 `zh-moqi`（它自家的小模型，不存在），
需要改成万象的模型名，并套用白霜推荐的惩罚参数：

```fish
cd ~/.local/share/fcitx5/rime
echo 'patch:
  grammar:
    language: wanxiang-lts-zh-hans
    non_collocation_penalty: -4
    collocation_max_length: 5
    collocation_min_length: 2
    collocation_penalty: -14
  translator/contextual_suggestions: true
  translator/max_homophones: 4
  translator/max_homographs: 2' > rime_frost.custom.yaml
```

> `non_collocation_penalty: -4` 很关键。默认 `-12` 对白霜词库惩罚过重，整句会明显变差。
> 若用双拼，把同样的内容写进对应 `rime_frost_double_pinyin*.custom.yaml` 也可以，
> 但一般共享的 `grammar` 段放在主方案就行了。

### 6.2 重新部署

```fish
cd ~/.local/share/fcitx5/rime
rm -rf build
fcitx5-remote -e      # 退出 fcitx5
sleep 2
fcitx5 -d -r          # 重新启动
```

首次部署会把词库编译成 `build/*.bin`，**需要几分钟**（白霜词库很大），期间打字会感觉卡是正常的。
也可以之后在托盘图标右键 → **重新部署**。

---

## 7. 验证是否成功

### 7.1 方案列表

```fish
busctl --user call org.fcitx.Fcitx5 /rime org.fcitx.Fcitx.Rime1 ListAllSchemas
```

应输出 `rime_frost`、`rime_frost_double_pinyin*` 等，而不是 `rime_ice`。

### 7.2 配置是否生效

```fish
grep -A6 '^grammar:' ~/.local/share/fcitx5/rime/build/rime_frost.schema.yaml
```

应看到 `language: "wanxiang-lts-zh-hans"`。

### 7.3 模型是否真的加载

先在任意输入框里**打几个字**（Rime 是懒加载，没会话就不会 mmap 模型），然后：

```fish
set pid (pgrep -x fcitx5 | head -1)
grep -i '\.gram' /proc/$pid/maps
```

能看到 `wanxiang-lts-zh-hans.gram` 就说明模型生效了。
如果日志里出现 `failed to load grammar database`，多半是文件名/`language` 对不上，或缺少 octagram 插件。

---

## 8. 开机自启与环境变量

### 8.1 KDE Plasma (Wayland)

Plasma 会通过 `kwinrc` 自己拉起输入法，**不需要**额外 autostart：

```fish
kreadconfig6 --file kwinrc --group Wayland --key InputMethod
# 应输出 /usr/share/applications/fcitx5-wayland-launcher.desktop
```

### 8.2 其他桌面 / X11

在 `~/.config/environment.d/` 或 `~/.config/fish/config.fish` 里设置：

```fish
set -gx GTK_IM_MODULE fcitx
set -gx QT_IM_MODULE fcitx
set -gx XMODIFIERS @im=fcitx
set -gx SDL_IM_MODULE fcitx
```

自启用 systemd 用户服务更稳：

```fish
systemctl --user enable --now fcitx5
```

（如果 fcitx5 已经被桌面环境拉起，就不要重复启动，避免 “Unable to request dbus name”。）

---

## 9. 日常更新与回滚

### 更新白霜

```fish
cd ~/.local/share/fcitx5/rime
git pull
rm -rf build
# 托盘右键 → 重新部署，或：
fcitx5-remote -e; sleep 2; fcitx5 -d -r
```

### 更新万象模型

直接重新下载覆盖同名 `.gram`，然后重新部署。

### 回滚到旧方案

```fish
cd ~/.local/share/fcitx5
mv rime "rime-frost-$(date +%Y%m%d)"
mv rime-backup-YYYYmmdd-HHMM rime
fcitx5-remote -e; sleep 2; fcitx5 -d -r
```

---

## 10. 常见问题（踩坑记录）

### Q1：只换方案，需要卸载 `fcitx5-rime` / `librime` 吗？

**不需要。** 方案是纯数据，引擎是引擎。删的只是 `~/.local/share/fcitx5/rime` 里的内容。

### Q2：能否直接 `yay -S rime-frost`？

目前 **AUR 和官方仓库都没有** `rime-frost`。用 git clone 或 release zip。
万一同名包出现，注意确认它是不是你要的那个。

### Q3：重启输入法后，Wayland 程序完全打不出中文了

这是本次实操真实踩到的坑：**不要杀 `fcitx5-wayland-launcher`**。

```fish
pgrep -a fcitx5
# 正常应有：
# /usr/bin/fcitx5 -d -r
# /usr/lib/fcitx5-wayland-launcher --reopen   ← 这个由 KWin 启动，负责把 Wayland 输入法连接交给 fcitx5
```

`fcitx5-wayland-launcher` 由 KWin（Plasma）在**会话启动时**拉起，
它接收 KWin 通过 `WAYLAND_SOCKET` 传进来的连接，再通过 D-Bus 交给 fcitx5。
一旦杀掉它，KWin **不会**自动重启它，手动运行也会报
`Failed to open Wayland connection`（因为拿不到 `WAYLAND_SOCKET`）。

判断方法：

```fish
fcitx5-diagnose | grep -E 'Group \[|InputContext'
# 若显示 Group [wayland:] has 0 InputContext(s) 且打字无候选，即为此问题
```

**解决办法：注销重新登录，或重启。** 之后只重启 `fcitx5` 本体即可：

```fish
fcitx5-remote -e; sleep 1; fcitx5 -d -r
```

### Q4：`kwriteconfig6 --key 'InputMethod[$e]'` 写坏了 `kwinrc`

`kwriteconfig6` 会把 `[$e]` 转义成 `\x5b$e\x5d`，产生重复键。
直接编辑 `~/.config/kwinrc` 即可，正确写法：

```ini
[Wayland]
InputMethod[$e]=/usr/share/applications/fcitx5-wayland-launcher.desktop
```

### Q5：部署时报 `error creating directory '/usr/share/rime-data/trash'`

无害。是 Rime 想在只读的共享目录里建回收站，失败不影响使用。
想彻底消掉可以在 `installation.yaml` 里自行同步/管理。

### Q6：候选词很少 / 整句很怪

- 确认 `rime_frost.custom.yaml` 里 `non_collocation_penalty: -4`
- 确认 `.gram` 确实被 mmap（见 7.3）
- 首次部署未完成时先别急着判断，等 `build/` 文件数稳定、`rime_frost.table.bin` 生成完整

### Q7：全拼/双拼/五笔怎么切

默认 `Ctrl+`` `（反引号）或 `F4` 打开方案选单；也可以在 fcitx5 托盘 → 配置 里添加。

### Q8：磁盘占用多大？

| 项目 | 大小 |
|---|---|
| 白霜配置 + 词库 | ~200 MiB |
| `build/` 编译产物 | ~120 MiB |
| 万象 `.gram` | ~380 MiB |
| 旧雾凇备份 | ~210 MiB（确认无问题后可删） |

---

## 参考链接

- 白霜拼音：<https://github.com/gaboolic/rime-frost>
- 万象语法模型：<https://github.com/amzxyz/rime-wanxiang-data>
- 万象拼音方案：<https://github.com/amzxyz/rime-wanxiang>
- 雾凇拼音：<https://github.com/iDvel/rime-ice>
- librime-octagram：<https://github.com/lotem/librime-octagram>
- fcitx5：<https://fcitx-im.org/wiki/Fcitx_5>
