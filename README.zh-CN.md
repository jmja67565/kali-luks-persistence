# 🐉 Kali Live USB 加密持久化（LUKS）

<p align="center">
🌐 <b>Languages:</b>
<a href="README.md">🇬🇧 English</a> ·
<a href="README.zh-CN.md">🇨🇳 简体中文</a> ·
<a href="README.zh-TW.md">🇹🇼 繁體中文</a> ·
<a href="README.ja.md">🇯🇵 日本語</a> ·
<a href="README.ko.md">🇰🇷 한국어</a> ·
<a href="README.de.md">🇩🇪 Deutsch</a> ·
<a href="README.fr.md">🇫🇷 Français</a> ·
<a href="README.es.md">🇪🇸 Español</a> ·
<a href="README.ru.md">🇷🇺 Русский</a> ·
<a href="README.pt-BR.md">🇧🇷 Português</a>
</p>

> 把一支 Kali Live USB 变成一个随身携带的系统：重启之后改动仍在 —— 而且任何人捡到这支 U 盘都读不出里面的内容。

**测试环境：** Kali 2026.3 Live ISO · 32 GB 以上 USB 3.0 · 在 Windows 上写入
**难度：** ★★☆☆☆ · **耗时：** 约 20 分钟 · **风险：** 每条命令都指向 U 盘 —— 认错磁盘就会抹掉你的硬盘

---

## 🎯 最终成果

U 盘的启动菜单会多出三个条目：

```
Live system (amd64)                        ← one-off, everything lost on shutdown
Live system (amd64) persistence            ← plaintext persistence
Live system (amd64) encrypted persistence  ← the goal: passphrase first, then boot
```

你在 `~` 下安装、配置和保存的一切都会在重启后存活。如果 U 盘被偷，攻击者看到的只是一个打不开的 LUKS 容器。

---

## 🗂️ 目录

- 🧠 0. 先弄懂三个概念
- 🧰 1. 准备工作
- 💾 2. 写入 ISO（DD 模式）
- 🖥️ 3. 首次启动并确认 U 盘
- 🧩 4. 创建第三个分区
- 🔐 5. 用 LUKS 加密
- 📦 6. ext4 + persistence 标签
- 📝 7. 写入 persistence.conf
- ✅ 8. 校验
- 🔒 9. 卸载并关闭
- 🚀 10. 启动验证
- 🛡️ 11. 加固
- 🩺 12. 故障排查
- ⚡ 13. 速查
- ❓ 常见问题
- 🔗 参考资料
- 📄 许可证

---

## 🧠 0. 先弄懂三个概念

理解这三点之后，下面每一条命令都会变得顺理成章，而不是照着抄。

**① Kali Live 是只读的。**
每次启动，系统都会把根文件系统从 ISO 解包到内存里。你做的任何改动都会在关机时消失。持久化存在的理由正是这个。

**② 持久化 = 一个标签为 `persistence` 的分区 + 一个文件。**
启动时，live-boot 会扫描各磁盘，寻找带有 `LABEL=persistence` 的分区，然后把它叠加到只读的根文件系统之上。读取来自 ISO，写入落在 U 盘上。这个分区只需要一个名为 `persistence.conf` 的文件，内容是一行：`/ union`。

**③ 加密持久化 = 把那个分区包进 LUKS。**
启动时，initramfs 会索要口令，解锁容器，把它映射到 `/dev/mapper/<name>`，然后**才**像对待普通持久化分区那样对待它。

关键认识：**live-boot 只关心标签，并不关心这个标签位于普通分区上还是位于 LUKS 映射上。**

U 盘上的最终布局：

```
Physical layout
┌──────────────────┬──────────┬────────────────────────────────┐
│ sda1   4.6 GB    │ sda2 4 MB│ sda3   remaining space         │
│ ISO9660 image    │ EFI boot │ LUKS container                 │
│ (read-only)      │          │  └─ /dev/mapper/my_usb         │
│                  │          │       ext4 · LABEL=persistence │
└──────────────────┴──────────┴────────────────────────────────┘
```

---

## 🧰 1. 准备工作

| 项目 | 要求 | 原因 |
|---|---|---|
| U 盘 | ≥ 32 GB，USB 3.0+ | 持久化会持续写入；廉价 U 盘会卡顿、掉盘 |
| 镜像 | `kali-linux-2026.3-live-amd64.iso` | 必须是 **live** 版，不能是 installer 版 |
| 写入工具 | Rufus（Windows） | 使用 **DD 模式** |
| 固件 | 关闭 Secure Boot | Kali Live 没有 Microsoft 签名 |
| 口令 | 保存在密码管理器里 | 一旦丢失，数据就永久没了 |

**关于下载：** 在官方 `https://cdimage.kali.org/current/` 目录中，Live 镜像**只以 `.torrent` 形式**分发（约 443 KiB）—— 以 `.iso` 形式提供的只有 installer。这里用 BitTorrent 客户端是正常途径，而不是镜像损坏时的变通办法。

**校验下载** —— 损坏的 ISO 是最隐蔽的故障模式：

```powershell
# Windows PowerShell
certutil -hashfile .\kali-linux-2026.3-live-amd64.iso SHA256
```

```bash
# Linux / macOS
sha256sum kali-linux-2026.3-live-amd64.iso
```

将结果与 `https://cdimage.kali.org/kali-2026.3/SHA256SUMS` 逐字符比对。

---

## 💾 2. 写入 ISO（DD 模式）

在 Rufus 中：**Device** 选你的 U 盘 · **Boot selection** 选该 ISO · 出现模式对话框时选 **DD Image mode** · 点击 Start。

> ⚠️ **绝不要碰 "Persistent partition size" 滑块。**
> Rufus 是通过给基于 casper 的发行版（Ubuntu、Mint）打补丁来实现持久化的。Kali 用的是 live-boot + isohybrid，这个滑块会重写分区布局，GRUB 再也找不到内核，你只会掉进一个光秃秃的 `grub>` 提示符。那就只能从头重写一遍。

写入完成后，**不要**用任何 Windows 分区工具（磁盘管理、DiskGenius、AOMEI……）打开这支 U 盘。

---

## 🖥️ 3. 首次启动并确认 U 盘

从 USB 启动，选择 **`Live system (amd64)`**（即那个普通条目）。

> 为什么不选持久化条目？因为此时还没有持久化分区，选了也没用 —— 而且必须从**非持久化会话**中修改分区，否则 overlay 正在使用你要编辑的那块磁盘。

打开终端，列出磁盘：

```bash
lsblk -o NAME,SIZE,RM,FSTYPE,LABEL,MOUNTPOINTS
```

综合三个信号来确认 U 盘：

- `RM=1`（可移动）
- 它的某个分区挂载在 `/run/live/medium` 或 `/lib/live/mount/medium`（那就是正在运行的 ISO）
- 容量与你手里这支 U 盘相符

顺便在这里核对一下写入模式：

```bash
lsblk -f
```

- ✅ `sda1` 是 `iso9660`、`sda2` 是 `vfat`（约 4 MB）→ **DD 模式，正确**
- ❌ 只有一个大的 `vfat` 分区 → 是以 ISO 模式写入的，回到第 2 步

**自动检测**（推荐 —— 这里打错一个字就会抹掉你的笔记本）：

```bash
medium=$(findmnt -no SOURCE /run/live/medium 2>/dev/null || findmnt -no SOURCE /lib/live/mount/medium)
usb=/dev/$(lsblk -no PKNAME "$medium")
echo "USB = $usb"
lsblk "$usb"
```

输出大致如下：

```
NAME   MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
sda      8:0    1  57.3G  0 disk
├─sda1   8:1    1   4.6G  0 part /run/live/medium
└─sda2   8:2    1     4M  0 part
```

把它固定到变量里 —— 后面每条命令都引用 `$usb`，这样即使打错字也碰不到错误的磁盘：

```bash
usb=/dev/sda     # 用你上面看到的值
```

---

## 🧩 4. 创建第三个分区

查看当前分区表：

```bash
sudo fdisk -l $usb
```

预期会看到 `Disklabel type: dos` 以及两个分区：大约 4.x GB（ISO）和一个几 MB 的分区（EFI）。

启动编辑器：

```bash
sudo fdisk $usb
```

按顺序输入这些按键：

| 按键 | 含义 |
|---|---|
| `p` ↵ | 打印分区表，确认只有 sda1 + sda2 |
| `n` ↵ | 新建分区 |
| `p` ↵ | 主分区 |
| `3` ↵ | 分区号（回车即接受默认值 3） |
| ↵ | 起始扇区：默认，紧接 sda2 之后 |
| ↵ | 结束扇区：默认，一直到磁盘末尾 |
| `w` ↵ | 写入并退出 |

> ⚠️ 为什么用 `fdisk` 而不用 GParted？
> 用 DD 写入的混合 ISO 是"盘中的盘"：整个设备被看成一大块 ISO9660 blob，所以 GParted 会把整支 U 盘报成单一分区，并且不显示任何空闲空间。这不是 bug，而是镜像本身的性质。`fdisk` 直接编辑底层的分区表，工作正常。

**接下来是最容易卡住的一步。**

执行 `w` 之后你多半会看到：

```
The kernel still uses the old table.
The new table will be used at the next reboot.
Re-reading the partition table failed.: Device or resource busy
```

原因很简单：**你就是从这支 U 盘启动的，所以 `sda1` 仍然处于挂载状态**，内核拒绝重新读取分区表。接着执行：

```bash
lsblk $usb
```

- **`sda3` 存在** → 继续第 5 步
- **它不见了** → 别跟 `partprobe` 死磕（它会报同样的 "busy"），直接**再次重启进入 `Live system (amd64)`**。分区表已经写到磁盘上了，内核会在启动时读取它。重新确认磁盘（第 3 步），确认 `sda3` 存在，然后继续。

---

## 🔐 5. 用 LUKS 加密

最后的安全检查 —— 按下回车之前，把设备打印出来看一眼：

```bash
[ -b ${usb}3 ] && lsblk -o NAME,SIZE,TYPE ${usb}3
```

然后：

```bash
sudo cryptsetup --verbose --verify-passphrase luksFormat ${usb}3
```

交互过程：

1. `WARNING! This will overwrite data on /dev/sda3 irrevocably.` → 输入 **`YES`**（必须全大写；`yes` 会被拒绝）
2. `Enter passphrase for /dev/sda3:` → 你的口令
3. `Verify passphrase:` → 再输一遍

**如何选择口令（很多人在这里翻车）：**

- 解锁发生在 initramfs 中，那里的键盘布局是 **US English**。在不同布局之间会发生位移的字符（`ä`、德语键盘上的 `@` 等）打出来的并不是你以为的那个字符
- 建议：**12 个字符以上，只用小写字母和数字**，或者一句你真的能背出来的长口令
- `--verify-passphrase` 只是要求输入两次，以防打错

> 这是你每次启动都要输入的口令。**弄丢它，数据就无法恢复。** 请在密码管理器里留一份。

---

## 📦 6. ext4 + persistence 标签

解锁容器：

```bash
sudo cryptsetup luksOpen ${usb}3 my_usb
```

输入口令。成功后 `/dev/mapper/my_usb` 就是*解密后*的设备 —— **此后所有格式化和写入都针对它，再也不要对 `${usb}3` 操作**。

```bash
sudo mkfs.ext4 -L persistence /dev/mapper/my_usb
sudo e2label /dev/mapper/my_usb
```

`e2label` 必须打印出：

```
persistence
```

> 为什么标签必须是 `persistence`？live-boot 靠 `LABEL=persistence` 来寻找持久化分区。`persist`、`Persistence`，或者 `persistence `（带尾部空格）都会被静默忽略。

---

## 📝 7. 写入 persistence.conf

```bash
sudo mkdir -pv /mnt/my_usb
sudo mount -v /dev/mapper/my_usb /mnt/my_usb
echo "/ union" | sudo tee /mnt/my_usb/persistence.conf
sync
```

`/ union` 的意思是：把整个根文件系统 `/` 叠加到这个分区上，于是所有写入都落在这里。

> ⚠️ 一定要在 Linux 里用 `echo` 写这个文件。如果你在 Windows 记事本里创建它，它会带上 CRLF 行尾，live-boot 解析出 `/ union\r`，该路径并不存在，持久化就静默地什么都不做。这是最常见的单一故障。

---

## ✅ 8. 校验

两项检查都必须通过：

```bash
sudo blkid /dev/mapper/my_usb
```

```
/dev/mapper/my_usb: LABEL="persistence" TYPE="ext4"
```

```bash
xxd /mnt/my_usb/persistence.conf
```

```
00000000: 2f20 756e 696f 6e0a                      / union.
```

文件必须以 `0a`（LF）结尾。如果看到 `0d0a`，那就是 CRLF —— 删掉重写。

再来一项，一眼即可看懂：

```bash
stat -c '%s bytes' /mnt/my_usb/persistence.conf   # 必须是 8
```

---

## 🔒 9. 卸载并关闭

```bash
sudo umount -v /mnt/my_usb
sudo cryptsetup luksClose my_usb
sudo reboot
```

顺序很重要：**先卸载文件系统，再关闭映射** —— 否则你会得到 `Device or resource busy`。

---

## 🚀 10. 启动验证

在启动菜单中选择 **`Live system (amd64) encrypted persistence`**。

你应该会被提示类似这样的内容：

```
Enter passphrase for /dev/sda3:
```

输入口令，进入桌面，然后验证两件事。

**① 数据确实被加密了**（被偷的 U 盘毫无用处）：

```bash
lsblk -f
```

`sda3` 必须显示 `crypto_LUKS`，其下方的 `my_usb` 映射带有 `ext4` 和 `persistence` 标签。

**② 持久化确实生效：**

```bash
echo hello > ~/enc-test
sudo reboot
```

再次启动同一个条目，解锁，然后：

```bash
cat ~/enc-test     # 输出 hello → 完成
```

---

## 🛡️ 11. 加固

### 11.1 备份 LUKS 头

头部一旦损坏，整个分区就不可读，而它存放在最开始的几 MB 里，一次误写就可能将其摧毁。

```bash
sudo cryptsetup luksHeaderBackup ${usb}3 --header-backup-file ~/luks-header-backup.img
```

把这个 `.img` 存到另一支 U 盘或云存储中。**它防的是头部损坏，不防忘记口令。**

### 11.2 设置 nuke 密码

先安装提供该功能的软件包（这是一个 Debian 补丁，默认并没有这条命令）：

```bash
sudo apt install -y cryptsetup-nuke-password
sudo cryptsetup luksAddNuke ${usb}3
```

设置一个紧急密码。如果你哪天被迫解锁这支 U 盘，输入这个密码就会抹掉 LUKS 头，任何人都无法再解密数据。

### 11.3 添加备用口令

```bash
sudo cryptsetup luksAddKey ${usb}3
```

### 11.4 查看密钥槽

```bash
sudo cryptsetup luksDump ${usb}3
```

每个槽位都会列出 —— 主口令、备用密钥、nuke 密码。

---

## 🩺 12. 故障排查

| 现象 | 原因 | 解决方法 |
|---|---|---|
| 启动菜单里没有 `encrypted persistence` 条目 | UEFI 下菜单嵌套方式不同，或 ISO 不对 | 到 `Live system` 子菜单里找找；确认你是从这支 U 盘启动的 |
| 执行 `w` 之后 `/dev/sda3` 不见了 | 内核无法重新读取被占用的分区表 | 重启进入 Live（第 4 步） |
| `partprobe` 报 "busy" | 因为你是从这支 U 盘启动的，它的 `sda1` 处于挂载状态 | 忽略，重启 |
| 口令反复被拒绝 | initramfs 使用 US 布局；口令含有依赖布局的字符 | 用纯 ASCII 字母和数字重新设置密钥 |
| 重启后改动消失 | 启动的是普通 `Live system`；或标签不对；或配置是 CRLF | 重做第 8 步的两项检查 |
| `blkid` 看不到 `persistence` 标签 | 标签写错，或者你是在映射之外做的格式化 | `e2label /dev/mapper/my_usb persistence` |
| `fdisk` 警告 `GPT PMBR size mismatch` | U 盘上残留了 GPT 表（Kali 镜像本身是 DOS 表，所以本不该出现） | `sudo sgdisk -e $usb` 重定位备份 GPT，然后再次运行 `fdisk` |
| `luksFormat` 提示 "Device is in use" | 分区被自动挂载了 | `sudo umount ${usb}3`，然后重试 |
| 启动时在 `cryptsetup` 处卡住一会儿 | 正常 —— 在 Live 环境里解锁很慢 | 等 10–30 秒 |

---

## ⚡ 13. 速查

一旦 `${usb}3` 已存在，从加密到收尾的完整流程就是：

```bash
usb=/dev/sda                      # 每次启动都重新确认

# 加密
sudo cryptsetup --verbose --verify-passphrase luksFormat ${usb}3

# 打开、格式化、打标签
sudo cryptsetup luksOpen ${usb}3 my_usb
sudo mkfs.ext4 -L persistence /dev/mapper/my_usb

# 挂载并配置
sudo mkdir -p /mnt/my_usb
sudo mount /dev/mapper/my_usb /mnt/my_usb
echo "/ union" | sudo tee /mnt/my_usb/persistence.conf
sync

# 校验
sudo blkid /dev/mapper/my_usb          # 应显示 LABEL="persistence" TYPE="ext4"
xxd /mnt/my_usb/persistence.conf       # 以 0a 结尾

# 收尾
sudo umount /mnt/my_usb
sudo cryptsetup luksClose my_usb
sudo reboot
```

---

## ❓ 常见问题

**我不想要加密 —— 只做普通持久化？**
跳过第 5–6 步中的 `luksFormat` 和 `luksOpen`，直接运行 `sudo mkfs.ext4 -L persistence ${usb}3`，然后从第 7 步继续。启动时选择 `persistence` 条目。

**Kali 发布了新版本 —— 我必须全部重做吗？**
是的。用 DD 重写 ISO 会重置分区表，持久化分区也随之消失。请先把你的 `~` 从 U 盘复制出来。

**持久化分区满了 —— 能扩容吗？**
把它扩展到 U 盘末尾的空闲空间，然后：

```bash
sudo cryptsetup luksOpen ${usb}3 my_usb
sudo cryptsetup resize my_usb
sudo resize2fs /dev/mapper/my_usb
```

**我忘了 LUKS 口令。**
没有办法恢复。没有暴力破解的捷径，头部备份也帮不上忙（它只防损坏）。这正是口令应该放进密码管理器的原因。

**我可以"安装"系统，让 `apt upgrade` 正常工作吗？**
那是另一条路线：用 installer ISO 把 Kali 装到 U 盘上，把它当作一块普通磁盘。这样更稳定，但每次内核升级都需要照顾 GRUB，而且会更快磨损 U 盘。对于便携工具盘，混合 Live 的方式更好。

**持久化会磨损我的 U 盘吗？**
它会增加写入量，但正常使用没有问题。别用杂牌 U 盘，不要把 U 盘塞满（写满的 overlay 行为会很怪），并偶尔看一眼 `df -h`。

**我可以在以持久化模式启动时修改持久化分区吗？**
不可以。overlay 正在使用它；在那个会话里重新分区或重新格式化极有可能损坏你的数据。始终从普通的 `Live system` 会话中操作。

---

## 🔗 参考资料

- Kali 文档：USB 持久化 - <https://www.kali.org/docs/usb/usb-persistence/>
- Kali 文档：加密持久化 - <https://www.kali.org/docs/usb/usb-persistence-encryption/>
- Ventoy 持久化（另一条路线：一个 `.dat` 文件，无需编辑分区） - <https://www.ventoy.net/en/plugin_persistence.html>

## 📄 许可证

基于 [MIT License](LICENSE) 发布。Copyright (c) 2026 Your Name。

欢迎提交 PR 和 issue，分享你踩过的坑。
