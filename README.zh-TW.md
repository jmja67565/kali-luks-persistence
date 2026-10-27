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

> 把一支 Kali Live USB 變成可攜式系統，重開機後仍保有你做的變更——而且別人撿到這支隨身碟也讀不出任何內容。

**測試環境：** Kali 2026.3 Live ISO · 32 GB 以上 USB 3.0 隨身碟 · 從 Windows 寫入
**難度：** ★★☆☆☆ · **時間：** 約 20 分鐘 · **風險：** 每個指令都指向 USB 隨身碟——認錯磁碟就會把你的硬碟抹掉

---

## 🎯 完成後你會得到什麼

USB 開機選單會多出三個項目：

```
Live system (amd64)                        ← 一次性，關機後一切消失
Live system (amd64) persistence            ← 明文持久化
Live system (amd64) encrypted persistence  ← 目標：先輸入通行碼，再開機
```

你安裝、設定與儲存在 `~` 底下的一切都能撐過重開機。若隨身碟被偷，攻擊者看到的只是一個他打不開的 LUKS 容器。

---

## 🗂️ 目錄

- 🧠 0. 先講三個觀念
- 🧰 1. 事前準備
- 💾 2. 寫入 ISO（DD 模式）
- 🖥️ 3. 第一次開機與辨識隨身碟
- 🧩 4. 建立第三個分割區
- 🔐 5. 用 LUKS 加密
- 📦 6. ext4 與 persistence 標籤
- 📝 7. 撰寫 persistence.conf
- ✅ 8. 驗證
- 🔒 9. 卸載並關閉
- 🚀 10. 開機並驗證
- 🛡️ 11. 加固
- 🩺 12. 疑難排解
- ⚡ 13. 速查
- ❓ 常見問題
- 🔗 參考資料
- 📄 授權

---

## 🧠 0. 先講三個觀念

把這三件事弄懂，底下每個指令就會變得理所當然，而不是照抄。

**① Kali Live 是唯讀的。**
每次開機時，系統會把 ISO 裡的根檔案系統解壓到 RAM。你做的任何變更在關機時就消失。這正是持久化存在的理由。

**② 持久化 = 一個標籤為 `persistence` 的分割區 + 一個檔案。**
開機時 live-boot 會掃描所有磁碟，找出 `LABEL=persistence` 的分割區，然後把它疊在唯讀的根檔案系統之上。讀取來自 ISO，寫入落在隨身碟。這個分割區只需要一個名為 `persistence.conf` 的檔案，內容是單一行：`/ union`。

**③ 加密持久化 = 把那個分割區包進 LUKS。**
開機時 initramfs 會要求輸入通行碼，解開容器，對應到 `/dev/mapper/<name>`，然後*才*把它當成一般的持久化分割區對待。

關鍵洞見：**live-boot 只在意標籤。它不在乎標籤是在一般分割區上，還是在 LUKS 對應裝置上。**

隨身碟的最終配置：

```
實體配置
┌──────────────────┬──────────┬────────────────────────────────┐
│ sda1   4.6 GB    │ sda2 4 MB│ sda3   剩餘空間                │
│ ISO9660 映像     │ EFI 開機 │ LUKS 容器                      │
│ (唯讀)           │          │  └─ /dev/mapper/my_usb         │
│                  │          │       ext4 · LABEL=persistence │
└──────────────────┴──────────┴────────────────────────────────┘
```

---

## 🧰 1. 事前準備

| 項目 | 需求 | 原因 |
|---|---|---|
| USB 隨身碟 | 32 GB 以上，USB 3.0+ | 持久化會不斷寫入；便宜隨身碟會卡住並掉線 |
| 映像檔 | `kali-linux-2026.3-live-amd64.iso` | 必須是 **live** 版，不是安裝版 |
| 寫入工具 | Rufus（Windows） | 使用 **DD 模式** |
| 韌體 | 關閉 Secure Boot | Kali Live 沒有 Microsoft 簽章 |
| 通行碼 | 存放在密碼管理員 | 弄丟了資料就永遠救不回來 |

**關於下載：** 在官方 `https://cdimage.kali.org/current/` 目錄中，Live 映像檔**只以 `.torrent` 形式**提供（約 443 KiB）——以 `.iso` 形式發佈的是安裝版。在這裡用 BitTorrent 用戶端是正常做法，不是鏡像站壞掉時的變通方案。

**驗證下載檔**——損壞的 ISO 是最陰險的失敗模式：

```powershell
# Windows PowerShell
certutil -hashfile .\kali-linux-2026.3-live-amd64.iso SHA256
```

```bash
# Linux / macOS
sha256sum kali-linux-2026.3-live-amd64.iso
```

逐字元比對 `https://cdimage.kali.org/kali-2026.3/SHA256SUMS`。

---

## 💾 2. 寫入 ISO（DD 模式）

在 Rufus 中：**Device** → 你的隨身碟 · **Boot selection** → 該 ISO · 出現模式對話框時選擇 **DD Image mode** · 開始。

> ⚠️ **絕對不要動「Persistent partition size」滑桿。**
> Rufus 的持久化是靠修改 casper 系發行版（Ubuntu、Mint）來實現的。Kali 用的是 live-boot + isohybrid，所以這個滑桿會改寫分割表，GRUB 再也找不到 kernel，你就會掉進一個光禿禿的 `grub>` 提示字元。只能從頭重寫一次。

完成後，**不要**用任何 Windows 分割工具（磁碟管理、DiskGenius、AOMEI…）打開這支隨身碟。

---

## 🖥️ 3. 第一次開機與辨識隨身碟

從 USB 開機，選擇 **`Live system (amd64)`**（最單純的那個項目）。

> 為什麼不選持久化項目？因為現在還沒有持久化分割區，選了也沒用——而且分割區必須在**非持久化工作階段**中修改，否則 overlay 正在使用你將要編輯的那顆磁碟。

開啟終端機並列出磁碟：

```bash
lsblk -o NAME,SIZE,RM,FSTYPE,LABEL,MOUNTPOINTS
```

同時用三個線索來確認哪支是隨身碟：

- `RM=1`（可移除）
- 它的其中一個分割區掛載在 `/run/live/medium` 或 `/lib/live/mount/medium`（那就是正在運行的 ISO）
- 容量與你手上那支隨身碟相符

順便在這裡檢查寫入模式是否正確：

```bash
lsblk -f
```

- ✅ `sda1` 是 `iso9660`、`sda2` 是 `vfat`（約 4 MB）→ **DD 模式，正確**
- ❌ 只有一個很大的 `vfat` 分割區 → 是以 ISO 模式寫入的，回到步驟 2

**自動偵測**（建議——這裡打錯一個字就會毀了你的筆電）：

```bash
medium=$(findmnt -no SOURCE /run/live/medium 2>/dev/null || findmnt -no SOURCE /lib/live/mount/medium)
usb=/dev/$(lsblk -no PKNAME "$medium")
echo "USB = $usb"
lsblk "$usb"
```

輸出大概長這樣：

```
NAME   MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
sda      8:0    1  57.3G  0 disk
├─sda1   8:1    1   4.6G  0 part /run/live/medium
└─sda2   8:2    1     4M  0 part
```

把它釘進一個變數——之後每個指令都引用 `$usb`，這樣打錯字也打不到錯的磁碟：

```bash
usb=/dev/sda     # 使用你上方看到的結果
```

---

## 🧩 4. 建立第三個分割區

檢視目前的分割表：

```bash
sudo fdisk -l $usb
```

預期會看到 `Disklabel type: dos` 與兩個分割區：大約 4.x GB（ISO）和幾 MB（EFI）。

啟動編輯器：

```bash
sudo fdisk $usb
```

依序輸入這些按鍵：

| 按鍵 | 意義 |
|---|---|
| `p` ↵ | 印出分割表，確認只有 sda1 + sda2 |
| `n` ↵ | 新分割區 |
| `p` ↵ | 主要（primary） |
| `3` ↵ | 分割區編號（按 Enter 接受預設值 3） |
| ↵ | 起始磁區：預設值，緊接在 sda2 之後 |
| ↵ | 結束磁區：預設值，到磁碟末端 |
| `w` ↵ | 寫入並離開 |

> ⚠️ 為什麼用 `fdisk` 而不是 GParted？
> 以 DD 寫入的混合 ISO 是「磁碟中的磁碟」：整個裝置被視為一大塊 ISO9660，所以 GParted 會把整支隨身碟回報成單一分割區，並顯示沒有可用空間。這不是 bug，而是這個映像檔的本質。`fdisk` 直接編輯底層分割表，運作正常。

**接下來是最多人卡住的一步。**

按下 `w` 之後，你很可能會看到：

```
The kernel still uses the old table.
The new table will be used at the next reboot.
Re-reading the partition table failed.: Device or resource busy
```

原因很簡單：**你就是從這支隨身碟開機的，所以 `sda1` 還掛載著**，kernel 因此拒絕重讀分割表。接著：

```bash
lsblk $usb
```

- **`sda3` 出現了** → 繼續步驟 5
- **沒有出現** → 別跟 `partprobe` 硬拚（它會以同樣的「busy」失敗），直接**再重開機進入 `Live system (amd64)`**。分割表已經寫在磁碟上，kernel 會在開機時讀到。重新偵測磁碟（步驟 3）、確認 `sda3` 存在，然後繼續。

---

## 🔐 5. 用 LUKS 加密

最後的安全檢查——按下 Enter 前先印出裝置並看一眼：

```bash
[ -b ${usb}3 ] && lsblk -o NAME,SIZE,TYPE ${usb}3
```

然後：

```bash
sudo cryptsetup --verbose --verify-passphrase luksFormat ${usb}3
```

互動過程：

1. `WARNING! This will overwrite data on /dev/sda3 irrevocably.` → 輸入 **`YES`**（全大寫；`yes` 會被拒絕）
2. `Enter passphrase for /dev/sda3:` → 你的通行碼
3. `Verify passphrase:` → 再輸入一次

**怎麼挑通行碼（這是很多人失敗的地方）：**

- 解鎖發生在 initramfs 裡，那裡是**美式英文**鍵盤配置。會隨配置改變的符號（`ä`、德式鍵盤的 `@`……）打出來的並不是你以為的字元
- 建議：**12 個字元以上，只用小寫字母與數字**，或者一段你真的能背出來的長通行碼
- `--verify-passphrase` 只是要求輸入兩次，用來防止打錯字

> 這就是你每次開機都要輸入的通行碼。**弄丟了資料就救不回來。** 請在密碼管理員裡留一份。

---

## 📦 6. ext4 與 persistence 標籤

解鎖容器：

```bash
sudo cryptsetup luksOpen ${usb}3 my_usb
```

輸入通行碼。成功後 `/dev/mapper/my_usb` 就是*解密後*的裝置——**從現在起每一次格式化與寫入都指向它，絕不再指向 `${usb}3`**。

```bash
sudo mkfs.ext4 -L persistence /dev/mapper/my_usb
sudo e2label /dev/mapper/my_usb
```

`e2label` 必須印出：

```
persistence
```

> 為什麼標籤一定要是 `persistence`？live-boot 是靠 `LABEL=persistence` 找到持久化分割區的。`persist`、`Persistence` 或 `persistence `（尾端多一個空格）都會被默默忽略。

---

## 📝 7. 撰寫 persistence.conf

```bash
sudo mkdir -pv /mnt/my_usb
sudo mount -v /dev/mapper/my_usb /mnt/my_usb
echo "/ union" | sudo tee /mnt/my_usb/persistence.conf
sync
```

`/ union` 的意思是：把整個根檔案系統 `/` 疊到這個分割區上，因此所有寫入都會落在這裡。

> ⚠️ 在 Linux 裡用 `echo` 寫。如果你在 Windows 記事本裡建立這個檔案，它會帶有 CRLF 換行，live-boot 解析到 `/ union\r`，該路徑不存在，持久化就默默失效。這是最常見的單一失敗原因。

---

## ✅ 8. 驗證

兩項檢查都必須通過：

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

檔案結尾必須是 `0a`（LF）。如果看到 `0d0a`，那就是 CRLF——刪掉重寫。

再一個，一眼就能看懂的：

```bash
stat -c '%s bytes' /mnt/my_usb/persistence.conf   # 必須是 8
```

---

## 🔒 9. 卸載並關閉

```bash
sudo umount -v /mnt/my_usb
sudo cryptsetup luksClose my_usb
sudo reboot
```

順序很重要：**先卸載檔案系統，再關閉對應**——否則你會得到 `Device or resource busy`。

---

## 🚀 10. 開機並驗證

在開機選單選擇 **`Live system (amd64) encrypted persistence`**。

你應該會看到類似這樣的提示：

```
Enter passphrase for /dev/sda3:
```

輸入通行碼，進到桌面，然後驗證兩件事。

**① 資料真的加密了**（被偷的隨身碟毫無用處）：

```bash
lsblk -f
```

`sda3` 必須顯示 `crypto_LUKS`，底下掛著 `my_usb` 對應，並帶有 `ext4` 與標籤 `persistence`。

**② 持久化真的有效：**

```bash
echo hello > ~/enc-test
sudo reboot
```

再次以同一個項目開機、解鎖，然後：

```bash
cat ~/enc-test     # 印出 hello → 完成
```

---

## 🛡️ 11. 加固

### 11.1 備份 LUKS header

header 毀損就代表整個分割區讀不出來，而它就住在最前面幾 MB 裡，一次誤寫就能毀掉它。

```bash
sudo cryptsetup luksHeaderBackup ${usb}3 --header-backup-file ~/luks-header-backup.img
```

把那個 `.img` 存到另一支隨身碟或雲端空間。**它防的是 header 毀損，不是忘記通行碼。**

### 11.2 設定 nuke 密碼

先安裝提供這項功能的套件（這是 Debian 的補丁，指令預設不存在）：

```bash
sudo apt install -y cryptsetup-nuke-password
sudo cryptsetup luksAddNuke ${usb}3
```

設定一組緊急密碼。如果你哪天被逼著解鎖這支隨身碟，輸入這組密碼就會抹掉 LUKS header，沒有人能解密資料。

### 11.3 新增備用通行碼

```bash
sudo cryptsetup luksAddKey ${usb}3
```

### 11.4 檢視金鑰插槽

```bash
sudo cryptsetup luksDump ${usb}3
```

每個插槽都會列出——主要通行碼、備用金鑰、nuke 密碼。

---

## 🩺 12. 疑難排解

| 症狀 | 原因 | 解法 |
|---|---|---|
| 開機選單裡沒有 `encrypted persistence` 項目 | UEFI 下選單層級不同，或 ISO 錯了 | 到 `Live system` 子選單裡找；確認你是從這支隨身碟開機 |
| 按下 `w` 之後 `/dev/sda3` 不見了 | kernel 無法重讀使用中的分割表 | 重開機進入 Live（步驟 4） |
| `partprobe` 回報「busy」 | 你從這支隨身碟開機，所以它的 `sda1` 還掛載著 | 忽略，重開機 |
| 通行碼一直被拒絕 | initramfs 用美式配置；通行碼含有隨配置改變的字元 | 用純 ASCII 字母與數字重新設定金鑰 |
| 重開機後變更消失 | 開的是單純的 `Live system`；或標籤錯了；或設定檔是 CRLF | 重做步驟 8 的兩項檢查 |
| `blkid` 看不到 `persistence` 標籤 | 標籤錯了，或你格式化在對應裝置之外 | `e2label /dev/mapper/my_usb persistence` |
| `fdisk` 警告 `GPT PMBR size mismatch` | 隨身碟留有殘餘的 GPT 表（Kali 映像檔本身是 DOS 表，所以照理不該發生） | `sudo sgdisk -e $usb` 把備份 GPT 搬到正確位置，再執行 `fdisk` |
| `luksFormat` 說「Device is in use」 | 分割區被自動掛載了 | `sudo umount ${usb}3`，重試 |
| 開機時停在 `cryptsetup` 一陣子 | 正常——在 Live 環境裡解鎖很慢 | 等 10–30 秒 |

---

## ⚡ 13. 速查

一旦 `${usb}3` 存在，從加密到清理的整套流程就是：

```bash
usb=/dev/sda                      # 每次開機都重新偵測

# 加密
sudo cryptsetup --verbose --verify-passphrase luksFormat ${usb}3

# 開啟、格式化、設定標籤
sudo cryptsetup luksOpen ${usb}3 my_usb
sudo mkfs.ext4 -L persistence /dev/mapper/my_usb

# 掛載並設定
sudo mkdir -p /mnt/my_usb
sudo mount /dev/mapper/my_usb /mnt/my_usb
echo "/ union" | sudo tee /mnt/my_usb/persistence.conf
sync

# 驗證
sudo blkid /dev/mapper/my_usb          # LABEL="persistence" TYPE="ext4"
xxd /mnt/my_usb/persistence.conf       # 結尾為 0a

# 清理
sudo umount /mnt/my_usb
sudo cryptsetup luksClose my_usb
sudo reboot
```

---

## ❓ 常見問題

**我不想加密——只要明文持久化就好？**
跳過步驟 5–6 的 `luksFormat` 與 `luksOpen`，直接執行 `sudo mkfs.ext4 -L persistence ${usb}3`，然後從步驟 7 繼續。開機時選 `persistence` 項目。

**新的 Kali 版本發佈了——我需要全部重做嗎？**
需要。用 DD 重寫 ISO 會重置分割表，持久化分割區也跟著消失。先把你的 `~` 從隨身碟複製出來。

**持久化分割區滿了——可以擴大嗎？**
把它延伸到隨身碟尾端的可用空間，然後：

```bash
sudo cryptsetup luksOpen ${usb}3 my_usb
sudo cryptsetup resize my_usb
sudo resize2fs /dev/mapper/my_usb
```

**我忘記 LUKS 通行碼了。**
沒有救援方式。沒有暴力破解的捷徑，header 備份也幫不上忙（它只防毀損）。這就是通行碼該放在密碼管理員裡的原因。

**我可以「安裝」系統，讓 `apt upgrade` 正常運作嗎？**
那是另一條路：用安裝版 ISO 把 Kali 裝到隨身碟上，把它當成一般磁碟對待。比較穩定，但每次 kernel 升級都要照顧 GRUB，而且更傷隨身碟。混合式 Live 做法更適合可攜式工具箱。

**持久化會操壞我的 USB 隨身碟嗎？**
它會增加寫入，但正常使用沒問題。避免來路不明的隨身碟、不要把隨身碟塞滿（塞滿的 overlay 行為會很怪），偶爾看一下 `df -h`。

**我可以在以持久化開機的狀態下修改持久化分割區嗎？**
不行。overlay 正在使用它；在那個工作階段重新分割或格式化，極有可能損壞你的資料。永遠要在單純的 `Live system` 工作階段做這件事。

---

## 🔗 參考資料

- Kali 文件：USB persistence - <https://www.kali.org/docs/usb/usb-persistence/>
- Kali 文件：encrypted persistence - <https://www.kali.org/docs/usb/usb-persistence-encryption/>
- Ventoy persistence（另一條路：一個 `.dat` 檔案，不需編輯分割區） - <https://www.ventoy.net/en/plugin_persistence.html>

## 📄 授權

以 [MIT License](LICENSE) 發佈。Copyright (c) 2026 Your Name。

歡迎針對你踩到的坑提出 pull request 與 issue。
