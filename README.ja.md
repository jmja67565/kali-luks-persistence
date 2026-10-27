# 🐉 Kali Live USB 暗号化永続化 (LUKS)

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

> Kali Live USB を、再起動しても変更が保持され、USB メモリを拾った他人には中身が読めないポータブルシステムに変えましょう。

**検証環境:** Kali 2026.3 Live ISO · 32 GB 以上の USB 3.0 · Windows から書き込み
**難易度:** ★★☆☆☆ · **所要時間:** 約 20 分 · **リスク:** すべてのコマンドは USB メモリを対象とします — ディスクを間違えるとドライブを消去してしまいます

---

## 🎯 最終的に手に入るもの

USB のブートメニューに 3 つのエントリが追加されます:

```
Live system (amd64)                        ← one-off, everything lost on shutdown
Live system (amd64) persistence            ← plaintext persistence
Live system (amd64) encrypted persistence  ← the goal: passphrase first, then boot
```

インストール・設定し、`~` 以下に保存したものはすべて再起動後も残ります。USB メモリが盗まれても、攻撃者が目にできるのは開くことのできない LUKS コンテナだけです。

---

## 🗂️ 目次

- 🧠 0. まず 3 つの概念
- 🧰 1. 前提条件
- 💾 2. ISO を書き込む（DD モード）
- 🖥️ 3. 初回起動と USB メモリの特定
- 🧩 4. 3 番目のパーティションを作成
- 🔐 5. LUKS で暗号化
- 📦 6. ext4 と persistence ラベル
- 📝 7. persistence.conf を書き込む
- ✅ 8. 検証
- 🔒 9. アンマウントとクローズ
- 🚀 10. 起動して確認
- 🛡️ 11. 堅牢化
- 🩺 12. トラブルシューティング
- ⚡ 13. チートシート
- ❓ FAQ
- 🔗 参考リンク
- 📄 ライセンス

---

## 🧠 0. まず 3 つの概念

この 3 つを理解しておけば、以下のすべてのコマンドは丸暗記の対象ではなく、自明なものになります。

**① Kali Live は読み取り専用です。**
起動のたびに、システムは ISO からルートファイルシステムを RAM に展開します。変更した内容はシャットダウンで消えます。だからこそ永続化が必要なのです。

**② 永続化 = `persistence` というラベルを付けた 1 つのパーティション + 1 つのファイル。**
起動時に live-boot がディスクを走査して `LABEL=persistence` のパーティションを探し、読み取り専用のルートファイルシステムの上に重ねます。読み取りは ISO から、書き込みは USB メモリに届きます。パーティションには `/ union` という 1 行だけを含む `persistence.conf` というファイルがあれば十分です。

**③ 暗号化永続化 = そのパーティションを LUKS で包んだもの。**
起動時に initramfs がパスフレーズを尋ね、コンテナをアンロックし、`/dev/mapper/<name>` にマッピングして、*そのうえで* 通常の永続化パーティションとまったく同じように扱います。

重要なポイント: **live-boot が気にするのはラベルだけです。そのラベルが素のパーティションにあるのか LUKS のマッピング上にあるのかは問いません。**

USB メモリの最終的なレイアウト:

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

## 🧰 1. 前提条件

| 項目 | 要件 | 理由 |
|---|---|---|
| USB メモリ | 32 GB 以上、USB 3.0 以上 | 永続化は常に書き込みを行うため、安価なメモリは停止したり落ちたりします |
| イメージ | `kali-linux-2026.3-live-amd64.iso` | インストーラーではなく **live** 版であること |
| 書き込みツール | Rufus (Windows) | **DD モード**を使用 |
| ファームウェア | Secure Boot 無効 | Kali Live は Microsoft の署名がないため |
| パスフレーズ | パスワードマネージャーに保管 | 失くすとデータは永久に失われます |

**ダウンロードについて:** 公式の `https://cdimage.kali.org/current/` ディレクトリでは、Live イメージは **`.torrent` としてのみ**配布されています（約 443 KiB）— `.iso` で提供されるのはインストーラーのほうです。ここでは BitTorrent クライアントを使うのが正規の手段であり、ミラーの不調に対する回避策ではありません。

**ダウンロードの検証** — 壊れた ISO はもっとも気づきにくい失敗要因です:

```powershell
# Windows PowerShell
certutil -hashfile .\kali-linux-2026.3-live-amd64.iso SHA256
```

```bash
# Linux / macOS
sha256sum kali-linux-2026.3-live-amd64.iso
```

`https://cdimage.kali.org/kali-2026.3/SHA256SUMS` と 1 文字ずつ照合してください。

---

## 💾 2. ISO を書き込む（DD モード）

Rufus で: **Device** → 使用する USB メモリ · **Boot selection** → ISO · モード選択のダイアログが出たら **DD Image mode** を選択 · Start。

> ⚠️ **「Persistent partition size」のスライダーには絶対に触れないでください。**
> Rufus は casper ベースのディストリビューション（Ubuntu、Mint）にパッチを当てることで永続化を実現します。Kali は live-boot + isohybrid を使うため、このスライダーはパーティションレイアウトを書き換えてしまい、GRUB がカーネルを見つけられなくなり、素の `grub>` プロンプトに落ちます。書き込みを最初からやり直してください。

完了したら、Windows のパーティションツール（Disk Management、DiskGenius、AOMEI など）で USB メモリを開かないでください。

---

## 🖥️ 3. 初回起動と USB メモリの特定

USB から起動し、**`Live system (amd64)`**（通常のエントリ）を選びます。

> なぜ永続化エントリではないのか？ まだ永続化パーティションが存在しないので意味がありません — さらに、パーティションは **非永続セッション** から変更する必要があります。そうでなければ、オーバーレイが編集中のディスクを活発に使用しています。

ターミナルを開いてディスクを一覧します:

```bash
lsblk -o NAME,SIZE,RM,FSTYPE,LABEL,MOUNTPOINTS
```

次の 3 つの手がかりを組み合わせて USB メモリを特定します:

- `RM=1`（リムーバブル）
- パーティションの 1 つが `/run/live/medium` または `/lib/live/mount/medium` にマウントされている（実行中の ISO です）
- 容量が手元の USB メモリと一致する

ついでに書き込みモードも確認しておきます:

```bash
lsblk -f
```

- ✅ `sda1` が `iso9660`、`sda2` が `vfat`（約 4 MB）→ **DD モードで正しい**
- ❌ 大きな `vfat` パーティションが 1 つだけ → ISO モードで書き込まれています。手順 2 に戻ってください

**自動検出**（推奨 — ここで 1 文字打ち間違えるとノート PC が消去されます）:

```bash
medium=$(findmnt -no SOURCE /run/live/medium 2>/dev/null || findmnt -no SOURCE /lib/live/mount/medium)
usb=/dev/$(lsblk -no PKNAME "$medium")
echo "USB = $usb"
lsblk "$usb"
```

出力は次のようになります:

```
NAME   MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
sda      8:0    1  57.3G  0 disk
├─sda1   8:1    1   4.6G  0 part /run/live/medium
└─sda2   8:2    1     4M  0 part
```

変数に固定します — 以降のコマンドはすべて `$usb` を参照するので、打ち間違いで別のディスクを叩くことはありません:

```bash
usb=/dev/sda     # 上で確認した値を使用
```

---

## 🧩 4. 3 番目のパーティションを作成

現在のテーブルを確認します:

```bash
sudo fdisk -l $usb
```

`Disklabel type: dos` と 2 つのパーティション（約 4.x GB の ISO と数 MB の EFI）が表示されるはずです。

エディタを起動します:

```bash
sudo fdisk $usb
```

次のキーを順番に入力します:

| キー | 意味 |
|---|---|
| `p` ↵ | テーブルを表示し、sda1 と sda2 だけであることを確認 |
| `n` ↵ | 新しいパーティション |
| `p` ↵ | プライマリ |
| `3` ↵ | パーティション番号（Enter でデフォルトの 3 を採用） |
| ↵ | 開始セクタ: デフォルト、sda2 の直後 |
| ↵ | 終了セクタ: デフォルト、ディスクの末尾まで |
| `w` ↵ | 書き込んで終了 |

> ⚠️ なぜ GParted ではなく `fdisk` なのか？
> DD で書き込まれたハイブリッド ISO は「ディスクの中のディスク」です。デバイス全体が 1 つの巨大な ISO9660 の塊として見えるため、GParted は USB メモリ全体を 1 つのパーティションとして報告し、空き領域を表示しません。これはバグではなく、イメージの性質です。`fdisk` は下位のパーティションテーブルを直接編集するので問題なく動作します。

**さて、多くの人がつまずくのがこの手順です。**

`w` の後、おそらく次のように表示されます:

```
The kernel still uses the old table.
The new table will be used at the next reboot.
Re-reading the partition table failed.: Device or resource busy
```

理由は単純です: **この USB メモリ自体から起動しているので `sda1` がまだマウントされており**、カーネルはテーブルの再読み取りを拒否します。次に:

```bash
lsblk $usb
```

- **`sda3` がある** → 手順 5 に進む
- **見当たらない** → `partprobe` と格闘しないでください（同じ「busy」で失敗します）。**`Live system (amd64)` に再起動**するだけです。テーブルはすでにディスク上にあり、カーネルが起動時に読み取ります。ディスクを再検出し（手順 3）、`sda3` の存在を確認して続行してください。

---

## 🔐 5. LUKS で暗号化

最後の安全確認 — Enter を押す前にデバイスを表示して目で確認してください:

```bash
[ -b ${usb}3 ] && lsblk -o NAME,SIZE,TYPE ${usb}3
```

次に:

```bash
sudo cryptsetup --verbose --verify-passphrase luksFormat ${usb}3
```

対話の流れ:

1. `WARNING! This will overwrite data on /dev/sda3 irrevocably.` → **`YES`** と入力（すべて大文字。`yes` は拒否されます）
2. `Enter passphrase for /dev/sda3:` → パスフレーズを入力
3. `Verify passphrase:` → もう一度入力

**パスフレーズの選び方（ここで失敗する人が多いところです）:**

- アンロックは initramfs 内で行われ、キーボードレイアウトは **US 英語**です。レイアウトによって位置が変わる文字（`ä`、ドイツ語配列の `@` など）は、意図したとおりには入力されません
- 推奨: **12 文字以上、小文字と数字のみ**、または実際に口に出して言える長いパスフレーズ
- `--verify-passphrase` はタイプミスを防ぐために 2 回尋ねるだけです

> これは起動のたびに入力するパスフレーズです。**失くすとデータは復元できません。** パスワードマネージャーに控えを保管してください。

---

## 📦 6. ext4 と persistence ラベル

コンテナをアンロックします:

```bash
sudo cryptsetup luksOpen ${usb}3 my_usb
```

パスフレーズを入力します。成功すると `/dev/mapper/my_usb` が *復号済み* のデバイスになります — **以降のフォーマットと書き込みはすべてこれを対象とし、`${usb}3` を再度使うことはありません**。

```bash
sudo mkfs.ext4 -L persistence /dev/mapper/my_usb
sudo e2label /dev/mapper/my_usb
```

`e2label` は次のように出力する必要があります:

```
persistence
```

> なぜラベルが `persistence` でなければならないのか？ live-boot は `LABEL=persistence` で永続化パーティションを見つけます。`persist`、`Persistence`、`persistence `（末尾のスペース）はすべて黙って無視されます。

---

## 📝 7. persistence.conf を書き込む

```bash
sudo mkdir -pv /mnt/my_usb
sudo mount -v /dev/mapper/my_usb /mnt/my_usb
echo "/ union" | sudo tee /mnt/my_usb/persistence.conf
sync
```

`/ union` の意味は、ルートファイルシステム `/` 全体をこのパーティションにオーバーレイし、すべての書き込みがここに届くようにする、ということです。

> ⚠️ Linux 内で `echo` を使って書いてください。Windows のメモ帳でファイルを作ると CRLF 改行になり、live-boot は `/ union\r` として解釈し、そのパスは存在しないため、永続化は黙って何も行いません。これがもっともよくある失敗です。

---

## ✅ 8. 検証

両方のチェックを通過する必要があります:

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

ファイルは `0a`（LF）で終わっている必要があります。`0d0a` が見えたら CRLF です — 削除して書き直してください。

もう 1 つ、ひと目でわかりやすい確認:

```bash
stat -c '%s bytes' /mnt/my_usb/persistence.conf   # 8 である必要があります
```

---

## 🔒 9. アンマウントとクローズ

```bash
sudo umount -v /mnt/my_usb
sudo cryptsetup luksClose my_usb
sudo reboot
```

順序が重要です: **先にファイルシステムをアンマウントし、次にマッピングをクローズ** — 逆にすると `Device or resource busy` になります。

---

## 🚀 10. 起動して確認

ブートメニューで **`Live system (amd64) encrypted persistence`** を選びます。

次のようなプロンプトが出るはずです:

```
Enter passphrase for /dev/sda3:
```

パスフレーズを入力し、デスクトップに到達したら、2 つのことを確認します。

**① データが本当に暗号化されているか**（盗まれた USB メモリは役に立ちません）:

```bash
lsblk -f
```

`sda3` が `crypto_LUKS` と表示され、その下の `my_usb` マッピングが `ext4` とラベル `persistence` を持っている必要があります。

**② 永続化が本当に機能するか:**

```bash
echo hello > ~/enc-test
sudo reboot
```

同じエントリで再度起動し、アンロックしてから:

```bash
cat ~/enc-test     # hello が表示される → 完了
```

---

## 🛡️ 11. 堅牢化

### 11.1 LUKS ヘッダーをバックアップする

ヘッダーが破損するとパーティション全体が読めなくなります。しかもヘッダーは先頭数メガバイトにあり、1 回の誤った書き込みで壊れ得ます。

```bash
sudo cryptsetup luksHeaderBackup ${usb}3 --header-backup-file ~/luks-header-backup.img
```

その `.img` は別の USB メモリかクラウドストレージに保管してください。**これはヘッダーの破損から守るものであり、パスフレーズの失念から守るものではありません。**

### 11.2 nuke パスワードを設定する

まずこの機能を提供するパッケージをインストールします（Debian のパッチであり、コマンドはデフォルトでは存在しません）:

```bash
sudo apt install -y cryptsetup-nuke-password
sudo cryptsetup luksAddNuke ${usb}3
```

緊急用のパスワードを設定します。もし USB メモリのアンロックを強要されたら、これを入力すれば LUKS ヘッダーが消去され、誰もデータを復号できなくなります。

### 11.3 バックアップ用パスフレーズを追加する

```bash
sudo cryptsetup luksAddKey ${usb}3
```

### 11.4 キースロットを確認する

```bash
sudo cryptsetup luksDump ${usb}3
```

各スロットが一覧表示されます — メインのパスフレーズ、バックアップキー、nuke パスワード。

---

## 🩺 12. トラブルシューティング

| 症状 | 原因 | 対処 |
|---|---|---|
| ブートメニューに `encrypted persistence` のエントリがない | UEFI ではメニューの階層が異なる、または ISO が違う | `Live system` サブメニューの中を確認する。USB メモリから起動していることを確かめる |
| `w` の後に `/dev/sda3` がない | カーネルが使用中のパーティションテーブルを再読み取りできない | Live に再起動する（手順 4） |
| `partprobe` が「busy」を報告する | USB メモリから起動しているため、その `sda1` がマウントされている | 無視して再起動する |
| パスフレーズが何度も拒否される | initramfs は US 配列を使う。パスフレーズにレイアウト依存の文字が含まれている | ASCII の英字と数字だけで再設定する |
| 再起動後に変更が消える | 通常の `Live system` で起動した、ラベルが違う、設定が CRLF | 手順 8 の 2 つのチェックをやり直す |
| `blkid` に `persistence` ラベルが表示されない | ラベルが違う、またはマッピングの外でフォーマットした | `e2label /dev/mapper/my_usb persistence` |
| `fdisk` が `GPT PMBR size mismatch` を警告する | USB メモリに古い GPT テーブルが残っている（Kali のイメージ自体は DOS テーブルなので、本来は起こらない） | `sudo sgdisk -e $usb` でバックアップ GPT を移動し、再度 `fdisk` を実行 |
| `luksFormat` が「Device is in use」と言う | パーティションが自動マウントされた | `sudo umount ${usb}3` して再試行 |
| 起動時に `cryptsetup` でしばらく止まる | 正常 — Live 環境でのアンロックは遅い | 10〜30 秒待つ |

---

## ⚡ 13. チートシート

`${usb}3` が存在すれば、暗号化から後片付けまでの一連の流れは次のとおりです:

```bash
usb=/dev/sda                      # 起動のたびに再検出する

# 暗号化
sudo cryptsetup --verbose --verify-passphrase luksFormat ${usb}3

# オープン、フォーマット、ラベル
sudo cryptsetup luksOpen ${usb}3 my_usb
sudo mkfs.ext4 -L persistence /dev/mapper/my_usb

# マウントと設定
sudo mkdir -p /mnt/my_usb
sudo mount /dev/mapper/my_usb /mnt/my_usb
echo "/ union" | sudo tee /mnt/my_usb/persistence.conf
sync

# 検証
sudo blkid /dev/mapper/my_usb          # LABEL="persistence" TYPE="ext4"
xxd /mnt/my_usb/persistence.conf       # 0a で終わる

# 後片付け
sudo umount /mnt/my_usb
sudo cryptsetup luksClose my_usb
sudo reboot
```

---

## ❓ FAQ

**暗号化はいらない、素の永続化だけでいい？**
手順 5〜6 の `luksFormat` と `luksOpen` を飛ばし、`sudo mkfs.ext4 -L persistence ${usb}3` を直接実行して、手順 7 から続けます。`persistence` エントリで起動してください。

**新しい Kali がリリースされた — すべてやり直す必要がある？**
はい。DD で ISO を書き直すとパーティションテーブルがリセットされ、永続化パーティションも一緒に消えます。先に `~` を USB メモリからコピーしておいてください。

**永続化パーティションがいっぱい — 拡張できる？**
USB メモリ末尾の空き領域まで拡張してから、次を実行します:

```bash
sudo cryptsetup luksOpen ${usb}3 my_usb
sudo cryptsetup resize my_usb
sudo resize2fs /dev/mapper/my_usb
```

**LUKS のパスフレーズを忘れた。**
復旧する方法はありません。総当たりの近道もなく、ヘッダーのバックアップも役に立ちません（破損に備えるだけです）。だからこそパスフレーズはパスワードマネージャーに入れておくべきなのです。

**`apt upgrade` が普通に動くようにシステムを「インストール」できる？**
それは別の道です: インストーラー ISO で Kali を通常のディスクとして USB メモリにインストールします。より安定しますが、カーネルを更新するたびに GRUB の面倒を見る必要があり、USB メモリの消耗も早くなります。ポータブルなツールキットにはハイブリッド Live の方式が優れています。

**永続化で USB メモリが劣化する？**
書き込みは増えますが、通常の使用なら問題ありません。ノーブランドのメモリは避け、容量を完全に埋めないようにし（オーバーレイがいっぱいになると挙動がおかしくなります）、ときどき `df -h` を確認してください。

**永続化で起動したまま永続化パーティションを変更できる？**
いいえ。オーバーレイが使用中です。そのセッションからパーティションを変更したり再フォーマットしたりすると、ほぼ確実にデータが壊れます。必ず素の `Live system` セッションから行ってください。

---

## 🔗 参考リンク

- Kali ドキュメント: USB persistence - <https://www.kali.org/docs/usb/usb-persistence/>
- Kali ドキュメント: encrypted persistence - <https://www.kali.org/docs/usb/usb-persistence-encryption/>
- Ventoy persistence（もう 1 つの方法: `.dat` ファイルを使い、パーティションを編集しない）- <https://www.ventoy.net/en/plugin_persistence.html>

## 📄 ライセンス

[MIT License](LICENSE) の下で公開されています。Copyright (c) 2026 jmja67565.

ハマった落とし穴についてのプルリクエストや issue 報告を歓迎します。
