# 🐉 Kali Live USB 암호화 영구 저장(LUKS)

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

> Kali Live USB를 재부팅해도 변경 사항이 그대로 유지되고, USB를 습득한 사람에게는 내용을 읽을 수 없는 휴대용 시스템으로 바꿔 줍니다.

**테스트 환경:** Kali 2026.3 Live ISO · 32GB 이상 USB 3.0 · Windows에서 작성
**난이도:** ★★☆☆☆ · **소요 시간:** 약 20분 · **위험도:** 모든 명령이 USB 스틱을 대상으로 하므로, 디스크를 잘못 지정하면 드라이브가 통째로 지워집니다

---

## 🎯 최종 결과

USB 부트 메뉴에 세 개의 항목이 추가됩니다:

```
Live system (amd64)                        ← one-off, everything lost on shutdown
Live system (amd64) persistence            ← plaintext persistence
Live system (amd64) encrypted persistence  ← the goal: passphrase first, then boot
```

`~` 아래에서 설치하고 설정하고 저장한 모든 것은 재부팅 후에도 유지됩니다. USB를 도난당하더라도 공격자가 보게 되는 것은 열 수 없는 LUKS 컨테이너뿐입니다.

---

## 🗂️ 목차

- 🧠 0. 먼저 알아 둘 세 가지 개념
- 🧰 1. 사전 준비
- 💾 2. ISO 굽기(DD 모드)
- 🖥️ 3. 첫 부팅과 USB 식별
- 🧩 4. 세 번째 파티션 생성
- 🔐 5. LUKS로 암호화
- 📦 6. ext4 + persistence 레이블
- 📝 7. persistence.conf 작성
- ✅ 8. 검증
- 🔒 9. 마운트 해제 및 닫기
- 🚀 10. 부팅 및 검증
- 🛡️ 11. 보안 강화
- 🩺 12. 문제 해결
- ⚡ 13. 치트 시트
- ❓ FAQ
- 🔗 참고 자료
- 📄 라이선스

---

## 🧠 0. 먼저 알아 둘 세 가지 개념

이 세 가지만 이해하면 아래의 모든 명령이 무작정 복사하는 대상이 아니라 당연한 절차로 보이게 됩니다.

**① Kali Live는 읽기 전용입니다.**
부팅할 때마다 시스템은 ISO의 루트 파일 시스템을 RAM으로 풀어 놓습니다. 변경한 내용은 종료와 함께 사라집니다. 영구 저장이 존재하는 이유가 바로 이것입니다.

**② 영구 저장(persistence) = `persistence` 레이블이 붙은 파티션 하나 + 파일 하나.**
부팅 시 live-boot가 디스크를 훑어 `LABEL=persistence`인 파티션을 찾은 뒤, 이를 읽기 전용 루트 파일 시스템 위에 overlay로 얹습니다. 읽기는 ISO에서 일어나고 쓰기는 USB에 기록됩니다. 이 파티션에는 `/ union` 한 줄만 담긴 `persistence.conf` 파일만 있으면 됩니다.

**③ 암호화된 영구 저장 = 그 파티션을 LUKS로 감싼 것입니다.**
부팅 시 initramfs가 암호(passphrase)를 물어보고, 컨테이너를 열어 `/dev/mapper/<name>`에 매핑한 다음, *그 뒤에* 일반 영구 저장 파티션과 완전히 똑같이 취급합니다.

핵심 통찰: **live-boot는 오직 레이블에만 관심이 있습니다. 레이블이 일반 파티션에 있든 LUKS 매핑 위에 있든 상관하지 않습니다.**

USB의 최종 레이아웃:

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

## 🧰 1. 사전 준비

| 항목 | 요구 사항 | 이유 |
|---|---|---|
| USB 스틱 | 32GB 이상, USB 3.0+ | 영구 저장은 끊임없이 쓰기를 하므로, 저가형 스틱은 버벅이다가 끊깁니다 |
| 이미지 | `kali-linux-2026.3-live-amd64.iso` | **live** 버전이어야 하며, installer가 아닙니다 |
| 굽기 도구 | Rufus(Windows) | **DD 모드**를 사용합니다 |
| 펌웨어 | Secure Boot 비활성화 | Kali Live는 Microsoft 서명이 없습니다 |
| 암호(passphrase) | 비밀번호 관리자에 보관 | 잊어버리면 데이터는 영원히 사라집니다 |

**다운로드에 관하여:** 공식 `https://cdimage.kali.org/current/` 디렉터리에서 Live 이미지는 **오직 `.torrent`로만** 배포됩니다(약 443 KiB). `.iso`로 배포되는 것은 installer 쪽입니다. 여기서 BitTorrent 클라이언트를 쓰는 것은 미러가 고장 나서 쓰는 우회책이 아니라 정상적인 경로입니다.

**다운로드 검증** — 손상된 ISO는 가장 찾기 어려운 실패 원인입니다:

```powershell
# Windows PowerShell
certutil -hashfile .\kali-linux-2026.3-live-amd64.iso SHA256
```

```bash
# Linux / macOS
sha256sum kali-linux-2026.3-live-amd64.iso
```

`https://cdimage.kali.org/kali-2026.3/SHA256SUMS`와 문자 단위로 비교하십시오.

---

## 💾 2. ISO 굽기(DD 모드)

Rufus에서: **Device** → 사용할 스틱 · **Boot selection** → ISO 파일 · 모드 선택 창이 뜨면 **DD Image mode** 선택 · Start.

> ⚠️ **"Persistent partition size" 슬라이더는 절대 건드리지 마십시오.**
> Rufus는 casper 기반 배포판(Ubuntu, Mint)을 패치하는 방식으로 영구 저장을 구현합니다. Kali는 live-boot + isohybrid를 쓰기 때문에, 이 슬라이더를 움직이면 파티션 레이아웃이 다시 쓰이고 GRUB이 커널을 찾지 못해 결국 맨 `grub>` 프롬프트에 떨어집니다. 처음부터 다시 구워야 합니다.

굽기가 끝난 뒤에는 Windows의 어떤 파티션 도구로도(디스크 관리, DiskGenius, AOMEI…) 스틱을 열지 마십시오.

---

## 🖥️ 3. 첫 부팅과 USB 식별

USB로 부팅하고 **`Live system (amd64)`**(일반 항목)를 선택하십시오.

> 왜 영구 저장 항목이 아닐까요? 아직 영구 저장 파티션이 없어서 무의미하며, 무엇보다 파티션은 **비영구 세션**에서 수정해야 합니다. 그렇지 않으면 overlay가 지금 편집하려는 디스크를 사용 중이게 됩니다.

터미널을 열고 디스크 목록을 확인하십시오:

```bash
lsblk -o NAME,SIZE,RM,FSTYPE,LABEL,MOUNTPOINTS
```

세 가지 신호를 함께 보고 스틱을 식별하십시오:

- `RM=1`(이동식)
- 파티션 중 하나가 `/run/live/medium` 또는 `/lib/live/mount/medium`에 마운트되어 있음(바로 실행 중인 ISO)
- 용량이 손에 들고 있는 스틱과 일치함

여기서 굽기 모드도 함께 점검하십시오:

```bash
lsblk -f
```

- ✅ `sda1`이 `iso9660`이고 `sda2`가 `vfat`(약 4MB) → **DD 모드, 정상**
- ❌ 커다란 `vfat` 파티션 하나뿐 → ISO 모드로 구운 것이므로 2단계로 돌아가십시오

**자동 감지** (권장 — 여기서 오타 한 번이면 노트북이 지워집니다):

```bash
medium=$(findmnt -no SOURCE /run/live/medium 2>/dev/null || findmnt -no SOURCE /lib/live/mount/medium)
usb=/dev/$(lsblk -no PKNAME "$medium")
echo "USB = $usb"
lsblk "$usb"
```

출력은 다음과 같습니다:

```
NAME   MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
sda      8:0    1  57.3G  0 disk
├─sda1   8:1    1   4.6G  0 part /run/live/medium
└─sda2   8:2    1     4M  0 part
```

변수에 고정하십시오 — 이후 모든 명령이 `$usb`를 참조하므로 오타가 다른 디스크를 건드릴 수 없습니다:

```bash
usb=/dev/sda     # 위에서 확인한 값을 사용
```

---

## 🧩 4. 세 번째 파티션 생성

현재 파티션 테이블을 확인하십시오:

```bash
sudo fdisk -l $usb
```

`Disklabel type: dos`와 파티션 두 개(대략 4.x GB(ISO)와 몇 MB(EFI))가 보일 것입니다.

편집기를 실행하십시오:

```bash
sudo fdisk $usb
```

다음 키를 순서대로 입력하십시오:

| 키 | 의미 |
|---|---|
| `p` ↵ | 테이블 출력, sda1 + sda2만 있는지 확인 |
| `n` ↵ | 새 파티션 |
| `p` ↵ | primary |
| `3` ↵ | 파티션 번호(Enter를 누르면 기본값 3) |
| ↵ | 시작 섹터: 기본값, sda2 바로 다음 |
| ↵ | 마지막 섹터: 기본값, 디스크 끝까지 |
| `w` ↵ | 기록하고 종료 |

> ⚠️ 왜 GParted가 아니라 `fdisk`일까요?
> DD로 구운 하이브리드 ISO는 "디스크 속의 디스크"입니다. 장치 전체가 하나의 거대한 ISO9660 덩어리로 보이므로 GParted는 스틱 전체를 단일 파티션으로 보고 여유 공간이 없다고 표시합니다. 버그가 아니라 이미지의 성질입니다. `fdisk`는 밑에 깔린 파티션 테이블을 직접 편집하므로 문제없이 동작합니다.

**이제 대부분의 사람이 막히는 지점입니다.**

`w`를 누르면 아마 다음과 같은 메시지가 보일 것입니다:

```
The kernel still uses the old table.
The new table will be used at the next reboot.
Re-reading the partition table failed.: Device or resource busy
```

이유는 간단합니다: **바로 이 스틱에서 부팅했기 때문에 `sda1`이 아직 마운트되어 있고**, 커널이 테이블을 다시 읽기를 거부하는 것입니다. 그다음:

```bash
lsblk $usb
```

- **`sda3`가 보인다** → 5단계로 계속 진행
- **보이지 않는다** → `partprobe`와 싸우지 말고(같은 "busy" 오류로 실패합니다) 그냥 **다시 `Live system (amd64)`로 재부팅**하십시오. 테이블은 이미 디스크에 기록되어 있고 커널은 부팅 시 이를 읽어 들입니다. 디스크를 다시 식별하고(3단계) `sda3`가 존재하는지 확인한 뒤 계속하십시오.

---

## 🔐 5. LUKS로 암호화

마지막 안전 점검 — Enter를 누르기 전에 장치를 출력해서 눈으로 확인하십시오:

```bash
[ -b ${usb}3 ] && lsblk -o NAME,SIZE,TYPE ${usb}3
```

그다음:

```bash
sudo cryptsetup --verbose --verify-passphrase luksFormat ${usb}3
```

대화 흐름:

1. `WARNING! This will overwrite data on /dev/sda3 irrevocably.` → **`YES`**를 입력하십시오(대문자; `yes`는 거부됩니다)
2. `Enter passphrase for /dev/sda3:` → 사용할 암호(passphrase)
3. `Verify passphrase:` → 한 번 더

**암호(passphrase)를 고르는 방법(여기서 사람들이 실패합니다):**

- 잠금 해제는 initramfs에서 이루어지며, 그곳의 키보드 레이아웃은 **US English**입니다. 레이아웃에 따라 위치가 달라지는 문자(독일어 키보드의 `ä`, `@` 등)는 생각한 것과 다르게 입력됩니다
- 권장: **12자 이상, 소문자와 숫자만**, 또는 실제로 외울 수 있는 긴 암호 문장
- `--verify-passphrase`는 단순히 두 번 물어 오타를 막아 줍니다

> 이것이 부팅할 때마다 입력할 암호입니다. **잊어버리면 데이터는 복구할 수 없습니다.** 비밀번호 관리자에 사본을 보관하십시오.

---

## 📦 6. ext4 + persistence 레이블

컨테이너를 여십시오:

```bash
sudo cryptsetup luksOpen ${usb}3 my_usb
```

암호를 입력하십시오. 성공하면 `/dev/mapper/my_usb`가 *복호화된* 장치입니다 — **이제부터 모든 포맷과 쓰기는 이 장치를 대상으로 하며, 다시 `${usb}3`를 건드리지 않습니다**.

```bash
sudo mkfs.ext4 -L persistence /dev/mapper/my_usb
sudo e2label /dev/mapper/my_usb
```

`e2label`은 다음을 출력해야 합니다:

```
persistence
```

> 레이블이 왜 `persistence`여야 할까요? live-boot는 `LABEL=persistence`로 영구 저장 파티션을 찾습니다. `persist`, `Persistence`, `persistence `(뒤에 공백)은 모두 아무 경고 없이 무시됩니다.

---

## 📝 7. persistence.conf 작성

```bash
sudo mkdir -pv /mnt/my_usb
sudo mount -v /dev/mapper/my_usb /mnt/my_usb
echo "/ union" | sudo tee /mnt/my_usb/persistence.conf
sync
```

`/ union`은 루트 파일 시스템 `/` 전체를 이 파티션 위에 overlay하라는 뜻이며, 따라서 모든 쓰기가 이곳에 기록됩니다.

> ⚠️ 반드시 Linux 안에서 `echo`로 작성하십시오. Windows 메모장에서 파일을 만들면 CRLF 줄바꿈이 붙어 live-boot가 `/ union\r`을 해석하게 되고, 해당 경로가 존재하지 않아 영구 저장이 아무 소리 없이 동작하지 않습니다. 이것이 가장 흔한 실패 원인입니다.

---

## ✅ 8. 검증

두 가지 검사가 모두 통과해야 합니다:

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

파일은 반드시 `0a`(LF)로 끝나야 합니다. `0d0a`가 보이면 CRLF이므로 파일을 지우고 다시 작성하십시오.

한 가지 더, 한눈에 읽기 쉬운 검사:

```bash
stat -c '%s bytes' /mnt/my_usb/persistence.conf   # 8이어야 함
```

---

## 🔒 9. 마운트 해제 및 닫기

```bash
sudo umount -v /mnt/my_usb
sudo cryptsetup luksClose my_usb
sudo reboot
```

순서가 중요합니다: **파일 시스템을 먼저 마운트 해제하고, 매핑을 나중에 닫으십시오** — 그렇지 않으면 `Device or resource busy`가 발생합니다.

---

## 🚀 10. 부팅 및 검증

부트 메뉴에서 **`Live system (amd64) encrypted persistence`**를 선택하십시오.

다음과 같은 프롬프트가 떠야 합니다:

```
Enter passphrase for /dev/sda3:
```

암호를 입력하고 데스크톱에 도달한 뒤 두 가지를 확인하십시오.

**① 데이터가 실제로 암호화되어 있는지** (도난당한 스틱은 쓸모없어야 합니다):

```bash
lsblk -f
```

`sda3`가 `crypto_LUKS`로 표시되고, 그 아래 `my_usb` 매핑이 `ext4`와 `persistence` 레이블을 가져야 합니다.

**② 영구 저장이 실제로 동작하는지:**

```bash
echo hello > ~/enc-test
sudo reboot
```

같은 항목으로 다시 부팅하고 잠금을 해제한 뒤:

```bash
cat ~/enc-test     # hello가 출력되면 완료
```

---

## 🛡️ 11. 보안 강화

### 11.1 LUKS 헤더 백업

헤더가 손상되면 파티션 전체를 읽을 수 없게 되는데, 헤더는 잘못된 쓰기 한 번으로도 파괴될 수 있는 앞부분 몇 메가바이트에 위치합니다.

```bash
sudo cryptsetup luksHeaderBackup ${usb}3 --header-backup-file ~/luks-header-backup.img
```

이 `.img` 파일을 다른 스틱이나 클라우드 스토리지에 보관하십시오. **이 백업은 헤더 손상만 막아 주며, 잊어버린 암호를 되살려 주지는 않습니다.**

### 11.2 nuke 암호 설정

먼저 이 기능을 제공하는 패키지를 설치하십시오(Debian 패치이므로 기본 상태에서는 명령이 존재하지 않습니다):

```bash
sudo apt install -y cryptsetup-nuke-password
sudo cryptsetup luksAddNuke ${usb}3
```

비상용 암호를 설정하십시오. 만약 누군가에게 스틱 잠금 해제를 강요받는 상황이 되면, 이 암호를 입력하는 순간 LUKS 헤더가 지워져 아무도 데이터를 복호화할 수 없게 됩니다.

### 11.3 백업 암호 추가

```bash
sudo cryptsetup luksAddKey ${usb}3
```

### 11.4 키 슬롯 확인

```bash
sudo cryptsetup luksDump ${usb}3
```

각 슬롯이 나열됩니다 — 주 암호, 백업 키, nuke 암호.

---

## 🩺 12. 문제 해결

| 증상 | 원인 | 해결 |
|---|---|---|
| 부트 메뉴에 `encrypted persistence` 항목이 없음 | UEFI에서 메뉴 중첩 구조가 다르거나 ISO가 잘못됨 | `Live system` 하위 메뉴 안을 살펴보고, 스틱으로 부팅했는지 확인하십시오 |
| `w` 이후에도 `/dev/sda3`가 없음 | 커널이 사용 중인 파티션 테이블을 다시 읽지 못함 | Live로 재부팅하십시오(4단계) |
| `partprobe`가 "busy"를 보고 | 이 스틱에서 부팅했기 때문에 스틱의 `sda1`이 마운트되어 있음 | 무시하고 재부팅하십시오 |
| 암호가 계속 거부됨 | initramfs는 US 배열을 쓰는데 암호에 배열 의존 문자가 포함됨 | 순수 ASCII 문자와 숫자로 다시 설정하십시오 |
| 재부팅 후 변경 사항이 사라짐 | 일반 `Live system`으로 부팅했거나, 레이블이 틀렸거나, 설정 파일이 CRLF임 | 8단계의 두 가지 검사를 다시 하십시오 |
| `blkid`에 `persistence` 레이블이 없음 | 레이블이 틀렸거나 매핑 바깥에서 포맷함 | `e2label /dev/mapper/my_usb persistence` |
| `fdisk`가 `GPT PMBR size mismatch`를 경고 | 스틱에 남은 GPT 테이블이 있음(Kali 이미지 자체는 DOS 테이블이라 원래는 발생하지 않아야 함) | `sudo sgdisk -e $usb`로 백업 GPT를 재배치한 뒤 `fdisk`를 다시 실행하십시오 |
| `luksFormat`이 "Device is in use"를 출력 | 파티션이 자동 마운트됨 | `sudo umount ${usb}3` 후 다시 시도하십시오 |
| 부팅이 `cryptsetup`에서 한동안 멈춤 | 정상 — Live 환경에서의 잠금 해제는 느립니다 | 10~30초 기다리십시오 |

---

## ⚡ 13. 치트 시트

`${usb}3`가 존재하는 상태에서, 암호화부터 정리까지의 전체 순서는 다음과 같습니다:

```bash
usb=/dev/sda                      # 부팅할 때마다 다시 식별

# 암호화
sudo cryptsetup --verbose --verify-passphrase luksFormat ${usb}3

# 열기, 포맷, 레이블
sudo cryptsetup luksOpen ${usb}3 my_usb
sudo mkfs.ext4 -L persistence /dev/mapper/my_usb

# 마운트 및 설정
sudo mkdir -p /mnt/my_usb
sudo mount /dev/mapper/my_usb /mnt/my_usb
echo "/ union" | sudo tee /mnt/my_usb/persistence.conf
sync

# 검증
sudo blkid /dev/mapper/my_usb          # LABEL="persistence" TYPE="ext4"
xxd /mnt/my_usb/persistence.conf       # 0a로 끝나야 함

# 정리
sudo umount /mnt/my_usb
sudo cryptsetup luksClose my_usb
sudo reboot
```

---

## ❓ FAQ

**암호화는 원하지 않고 평문 영구 저장만 원합니다.**
5~6단계의 `luksFormat`과 `luksOpen`을 건너뛰고 `sudo mkfs.ext4 -L persistence ${usb}3`를 바로 실행한 뒤 7단계부터 계속하십시오. 부팅할 때는 `persistence` 항목을 선택하십시오.

**새 Kali 릴리스가 나왔는데 전부 다시 해야 하나요?**
네. DD로 ISO를 다시 구우면 파티션 테이블이 초기화되고 영구 저장 파티션도 함께 사라집니다. 먼저 스틱에서 `~`를 복사해 두십시오.

**영구 저장 파티션이 가득 찼는데 늘릴 수 있나요?**
스틱 끝의 여유 공간까지 확장한 뒤 다음을 실행하십시오:

```bash
sudo cryptsetup luksOpen ${usb}3 my_usb
sudo cryptsetup resize my_usb
sudo resize2fs /dev/mapper/my_usb
```

**LUKS 암호를 잊어버렸습니다.**
복구 방법은 없습니다. 무차별 대입 같은 지름길도 없고, 헤더 백업도 도움이 되지 않습니다(손상만 막아 줄 뿐입니다). 암호를 비밀번호 관리자에 보관해야 하는 이유가 여기에 있습니다.

**시스템을 "설치"해서 `apt upgrade`가 정상적으로 동작하게 할 수는 없나요?**
그것은 다른 경로입니다: installer ISO로 Kali를 스틱에 설치하고 스틱을 일반 디스크처럼 다루는 방식입니다. 더 안정적이지만 커널을 업그레이드할 때마다 GRUB를 손봐야 하고 스틱 마모도 더 빠릅니다. 휴대용 툴킷에는 하이브리드 Live 방식이 더 낫습니다.

**영구 저장 때문에 USB 스틱이 닳지 않나요?**
쓰기가 늘어나는 것은 사실이지만 일반적인 사용에는 문제가 없습니다. 이름 없는 저가 스틱은 피하고, 스틱을 완전히 채우지 말고(가득 찬 overlay는 이상하게 동작합니다), 가끔 `df -h`로 확인하십시오.

**영구 저장으로 부팅한 상태에서 영구 저장 파티션을 수정할 수 있나요?**
아니요. overlay가 그 파티션을 사용 중이므로, 그 세션에서 파티션을 나누거나 다시 포맷하면 데이터가 손상될 가능성이 매우 높습니다. 항상 일반 `Live system` 세션에서 작업하십시오.

---

## 🔗 참고 자료

- Kali 문서: USB 영구 저장 - <https://www.kali.org/docs/usb/usb-persistence/>
- Kali 문서: 암호화된 영구 저장 - <https://www.kali.org/docs/usb/usb-persistence-encryption/>
- Ventoy 영구 저장(다른 경로: `.dat` 파일을 쓰며 파티션 편집이 필요 없음) - <https://www.ventoy.net/en/plugin_persistence.html>

## 📄 라이선스

[MIT License](LICENSE)에 따라 배포됩니다. Copyright (c) 2026 Your Name.

직접 겪은 함정에 대한 풀 리퀘스트와 이슈 제보를 환영합니다.
