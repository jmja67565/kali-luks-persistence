# 🐉 Kali Live USB verschlüsselte Persistenz (LUKS)

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

> Verwandle einen Kali Live USB in ein portables System, das deine Änderungen über Neustarts hinweg behält — und für jeden, der den Stick findet, unlesbar bleibt.

**Getestet mit:** Kali 2026.3 Live ISO · 32 GB+ USB 3.0 · beschrieben unter Windows
**Schwierigkeit:** ★★☆☆☆ · **Zeit:** ~20 Minuten · **Risiko:** jeder Befehl zielt auf den USB-Stick — identifizierst du die falsche Festplatte, löschst du dein Laufwerk

---

## 🎯 Was am Ende dabei herauskommt

Das Boot-Menü des USB-Sticks bekommt drei Einträge:

```
Live system (amd64)                        ← einmalig, alles beim Herunterfahren verloren
Live system (amd64) persistence            ← Klartext-Persistenz
Live system (amd64) encrypted persistence  ← das Ziel: erst Passphrase, dann booten
```

Alles, was du unter `~` installierst, konfigurierst und speicherst, überlebt einen Neustart. Wird der Stick gestohlen, sieht ein Angreifer nur einen LUKS-Container, den er nicht öffnen kann.

---

## 🗂️ Inhalt

- 🧠 0. Zuerst drei Konzepte
- 🧰 1. Voraussetzungen
- 💾 2. Das ISO schreiben (DD-Modus)
- 🖥️ 3. Erster Start & den Stick identifizieren
- 🧩 4. Die dritte Partition erstellen
- 🔐 5. Mit LUKS verschlüsseln
- 📦 6. ext4 + das Persistenz-Label
- 📝 7. persistence.conf schreiben
- ✅ 8. Überprüfen
- 🔒 9. Aushängen & schließen
- 🚀 10. Starten und überprüfen
- 🛡️ 11. Härtung
- 🩺 12. Fehlerbehebung
- ⚡ 13. Cheat Sheet
- ❓ FAQ
- 🔗 Referenzen
- 📄 Lizenz

---

## 🧠 0. Zuerst drei Konzepte

Wenn du diese drei verstehst, wird jeder Befehl offensichtlich statt etwas, das du blind kopierst.

**① Kali Live ist schreibgeschützt.**
Bei jedem Start entpackt das System ein Root-Dateisystem aus dem ISO in den RAM. Alles, was du änderst, ist beim Herunterfahren weg. Genau deshalb gibt es Persistenz.

**② Persistenz = eine Partition mit dem Label `persistence` + eine Datei.**
Beim Start durchsucht live-boot die Festplatten nach einer Partition mit `LABEL=persistence` und legt sie dann über das schreibgeschützte Root-Dateisystem. Gelesen wird vom ISO, geschrieben wird auf den Stick. Die Partition braucht nur eine Datei namens `persistence.conf` mit einer einzigen Zeile: `/ union`.

**③ Verschlüsselte Persistenz = diese Partition in LUKS eingehüllt.**
Beim Start fragt die initramfs nach der Passphrase, entsperrt den Container, mappt ihn auf `/dev/mapper/<name>` und behandelt ihn *dann* genau wie eine normale Persistenz-Partition.

Die entscheidende Erkenntnis: **live-boot interessiert nur das Label. Es ist ihm egal, ob das Label auf einer einfachen Partition oder auf einem LUKS-Mapping liegt.**

Endgültiges Layout auf dem Stick:

```
Physisches Layout
┌──────────────────┬──────────┬────────────────────────────────┐
│ sda1   4.6 GB    │ sda2 4 MB│ sda3  restlicher Platz         │
│ ISO9660-Abbild   │ EFI-Boot │ LUKS-Container                 │
│ (nur lesbar)     │          │  └─ /dev/mapper/my_usb         │
│                  │          │       ext4 · LABEL=persistence │
└──────────────────┴──────────┴────────────────────────────────┘
```

---

## 🧰 1. Voraussetzungen

| Element | Anforderung | Warum |
|---|---|---|
| USB-Stick | ≥ 32 GB, USB 3.0+ | Persistenz schreibt ständig; billige Sticks stocken und fallen aus |
| Image | `kali-linux-2026.3-live-amd64.iso` | muss **Live** sein, nicht Installer |
| Schreibprogramm | Rufus (Windows) | **DD-Modus** verwenden |
| Firmware | Secure Boot deaktiviert | Kali Live ist nicht von Microsoft signiert |
| Passphrase | in einem Passwort-Manager gespeichert | verlierst du sie, sind die Daten endgültig weg |

**Zum Herunterladen:** im offiziellen Verzeichnis `https://cdimage.kali.org/current/` wird das Live-Image **nur als `.torrent`** verteilt (etwa 443 KiB) — der Installer ist der, der als `.iso` ausgeliefert wird. Einen BitTorrent-Client zu benutzen ist hier der normale Weg, kein Workaround für einen kaputten Mirror.

**Überprüfe den Download** — ein beschädigtes ISO ist die heimtückischste Fehlerquelle:

```powershell
# Windows PowerShell
certutil -hashfile .\kali-linux-2026.3-live-amd64.iso SHA256
```

```bash
# Linux / macOS
sha256sum kali-linux-2026.3-live-amd64.iso
```

Vergleiche Zeichen für Zeichen mit `https://cdimage.kali.org/kali-2026.3/SHA256SUMS`.

---

## 💾 2. Das ISO schreiben (DD-Modus)

In Rufus: **Device** → dein Stick · **Boot selection** → das ISO · wenn der Modus-Dialog erscheint, **DD Image mode** wählen · Start.

> ⚠️ **Fass den Schieberegler „Persistent partition size“ niemals an.**
> Rufus implementiert Persistenz, indem es casper-basierte Distributionen (Ubuntu, Mint) patcht. Kali nutzt live-boot + isohybrid, also schreibt der Regler das Partitionslayout neu, GRUB findet den Kernel nicht mehr, und du landest in einer nackten `grub>`-Eingabeaufforderung. Schreib den Stick von Grund auf neu.

Danach öffne den Stick **auf keinen Fall** mit einem Windows-Partitionierungstool (Datenträgerverwaltung, DiskGenius, AOMEI…).

---

## 🖥️ 3. Erster Start & den Stick identifizieren

Starte vom USB und wähle **`Live system (amd64)`** (den einfachen Eintrag).

> Warum nicht der Persistenz-Eintrag? Es gibt noch keine Persistenz-Partition, also wäre er sinnlos — und Partitionen müssen aus einer **nicht-persistenten Sitzung** heraus geändert werden, sonst nutzt das Overlay gerade die Festplatte, die du bearbeitest.

Öffne ein Terminal und liste die Festplatten auf:

```bash
lsblk -o NAME,SIZE,RM,FSTYPE,LABEL,MOUNTPOINTS
```

Identifiziere den Stick anhand von drei Signalen:

- `RM=1` (wechselbar)
- eine seiner Partitionen ist unter `/run/live/medium` oder `/lib/live/mount/medium` eingehängt (das ist das laufende ISO)
- die Kapazität zu dem Stick in deiner Hand passt

Prüfe gleich hier den Schreibmodus:

```bash
lsblk -f
```

- ✅ `sda1` ist `iso9660` und `sda2` ist `vfat` (~4 MB) → **DD-Modus, korrekt**
- ❌ eine einzige große `vfat`-Partition → im ISO-Modus geschrieben, geh zurück zu Schritt 2

**Automatische Erkennung** (empfohlen — ein Tippfehler hier löscht dein Laptop):

```bash
medium=$(findmnt -no SOURCE /run/live/medium 2>/dev/null || findmnt -no SOURCE /lib/live/mount/medium)
usb=/dev/$(lsblk -no PKNAME "$medium")
echo "USB = $usb"
lsblk "$usb"
```

Die Ausgabe sieht so aus:

```
NAME   MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
sda      8:0    1  57.3G  0 disk
├─sda1   8:1    1   4.6G  0 part /run/live/medium
└─sda2   8:2    1     4M  0 part
```

Leg ihn in einer Variable fest — jeder spätere Befehl verweist auf `$usb`, damit ein Tippfehler nicht die falsche Festplatte treffen kann:

```bash
usb=/dev/sda     # nimm, was du oben gesehen hast
```

---

## 🧩 4. Die dritte Partition erstellen

Prüfe die aktuelle Tabelle:

```bash
sudo fdisk -l $usb
```

Erwarte `Disklabel type: dos` und zwei Partitionen: etwa 4.x GB (ISO) und ein paar MB (EFI).

Starte den Editor:

```bash
sudo fdisk $usb
```

Gib diese Tasten der Reihe nach ein:

| Taste | Bedeutung |
|---|---|
| `p` ↵ | Tabelle ausgeben, nur sda1 + sda2 bestätigen |
| `n` ↵ | neue Partition |
| `p` ↵ | primär |
| `3` ↵ | Partitionsnummer (Enter akzeptiert die Vorgabe 3) |
| ↵ | erster Sektor: Vorgabe, direkt nach sda2 |
| ↵ | letzter Sektor: Vorgabe, bis zum Ende der Festplatte |
| `w` ↵ | schreiben und beenden |

> ⚠️ Warum `fdisk` und nicht GParted?
> Ein per DD geschriebenes Hybrid-ISO ist eine „Festplatte in einer Festplatte“: Das ganze Gerät wird als ein riesiger ISO9660-Blob gesehen, deshalb meldet GParted den gesamten Stick als eine einzige Partition und zeigt keinen freien Platz. Das ist kein Bug, das ist die Natur des Images. `fdisk` bearbeitet die darunterliegende Partitionstabelle direkt und funktioniert einwandfrei.

**Jetzt kommt der Schritt, an dem die meisten hängen bleiben.**

Nach `w` siehst du wahrscheinlich:

```
The kernel still uses the old table.
The new table will be used at the next reboot.
Re-reading the partition table failed.: Device or resource busy
```

Der Grund ist einfach: **du hast von genau diesem Stick gebootet, also ist `sda1` noch eingehängt** und der Kernel weigert sich, die Tabelle neu zu lesen. Dann:

```bash
lsblk $usb
```

- **`sda3` ist da** → weiter mit Schritt 5
- **sie fehlt** → kämpfe nicht mit `partprobe` (es scheitert mit demselben „busy“), sondern **starte einfach erneut in `Live system (amd64)`**. Die Tabelle liegt schon auf der Platte; der Kernel übernimmt sie beim Start. Erkenne die Festplatte neu (Schritt 3), bestätige, dass `sda3` existiert, und mach weiter.

---

## 🔐 5. Mit LUKS verschlüsseln

Letzte Sicherheitsprüfung — gib das Gerät aus und schau es dir an, bevor du Enter drückst:

```bash
[ -b ${usb}3 ] && lsblk -o NAME,SIZE,TYPE ${usb}3
```

Dann:

```bash
sudo cryptsetup --verbose --verify-passphrase luksFormat ${usb}3
```

Interaktion:

1. `WARNING! This will overwrite data on /dev/sda3 irrevocably.` → tippe **`YES`** (Großbuchstaben; `yes` wird abgelehnt)
2. `Enter passphrase for /dev/sda3:` → deine Passphrase
3. `Verify passphrase:` → noch einmal

**Wie du die Passphrase wählst (hier scheitern die Leute):**

- Das Entsperren passiert in der initramfs, wo das Tastaturlayout **US-Englisch** ist. Zeichen, die zwischen Layouts wandern (`ä`, `@` auf einer deutschen Tastatur, …), tippen nicht, was du denkst
- Empfohlen: **12+ Zeichen, nur Kleinbuchstaben und Ziffern**, oder eine lange Passphrase, die du wirklich aufsagen kannst
- `--verify-passphrase` fragt einfach zweimal nach, um vor Tippfehlern zu schützen

> Das ist die Passphrase, die du bei jedem Start eintippst. **Verlierst du sie, sind die Daten nicht wiederherstellbar.** Bewahre eine Kopie in deinem Passwort-Manager auf.

---

## 📦 6. ext4 + das Persistenz-Label

Entsperre den Container:

```bash
sudo cryptsetup luksOpen ${usb}3 my_usb
```

Gib die Passphrase ein. Bei Erfolg ist `/dev/mapper/my_usb` das *entschlüsselte* Gerät — **jede Formatierung und jedes Schreiben zielt ab jetzt darauf, nie wieder auf `${usb}3`**.

```bash
sudo mkfs.ext4 -L persistence /dev/mapper/my_usb
sudo e2label /dev/mapper/my_usb
```

`e2label` muss ausgeben:

```
persistence
```

> Warum muss das Label `persistence` sein? live-boot findet die Persistenz-Partition über `LABEL=persistence`. `persist`, `Persistence` oder `persistence ` (mit Leerzeichen am Ende) werden alle stillschweigend ignoriert.

---

## 📝 7. persistence.conf schreiben

```bash
sudo mkdir -pv /mnt/my_usb
sudo mount -v /dev/mapper/my_usb /mnt/my_usb
echo "/ union" | sudo tee /mnt/my_usb/persistence.conf
sync
```

`/ union` bedeutet: das gesamte Root-Dateisystem `/` auf diese Partition legen, damit alle Schreibvorgänge hier landen.

> ⚠️ Schreib sie mit `echo` unter Linux. Wenn du die Datei im Windows-Editor erstellst, bekommt sie CRLF-Zeilenenden, live-boot liest `/ union\r`, der Pfad existiert nicht, und die Persistenz macht stillschweigend nichts. Das ist der häufigste Fehler.

---

## ✅ 8. Überprüfen

Beide Prüfungen müssen bestehen:

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

Die Datei muss mit `0a` (LF) enden. Wenn du `0d0a` siehst, ist es CRLF — löschen und neu anlegen.

Noch eine, auf einen Blick:

```bash
stat -c '%s bytes' /mnt/my_usb/persistence.conf   # muss 8 sein
```

---

## 🔒 9. Aushängen & schließen

```bash
sudo umount -v /mnt/my_usb
sudo cryptsetup luksClose my_usb
sudo reboot
```

Die Reihenfolge zählt: **erst das Dateisystem aushängen, dann das Mapping schließen** — sonst bekommst du `Device or resource busy`.

---

## 🚀 10. Starten und überprüfen

Wähle im Boot-Menü **`Live system (amd64) encrypted persistence`**.

Du solltest ungefähr so aufgefordert werden:

```
Enter passphrase for /dev/sda3:
```

Tippe die Passphrase, komm auf den Desktop und prüfe dann zwei Dinge.

**① Die Daten sind wirklich verschlüsselt** (ein gestohlener Stick ist nutzlos):

```bash
lsblk -f
```

`sda3` muss `crypto_LUKS` zeigen, mit dem `my_usb`-Mapping darunter, das `ext4` und das Label `persistence` trägt.

**② Die Persistenz funktioniert wirklich:**

```bash
echo hello > ~/enc-test
sudo reboot
```

Starte denselben Eintrag erneut, entsperre, dann:

```bash
cat ~/enc-test     # gibt hello aus → fertig
```

---

## 🛡️ 11. Härtung

### 11.1 LUKS-Header sichern

Ein beschädigter Header macht die ganze Partition unlesbar, und er liegt in den ersten paar Megabyte, wo ein einziger Fehlgriff ihn zerstören kann.

```bash
sudo cryptsetup luksHeaderBackup ${usb}3 --header-backup-file ~/luks-header-backup.img
```

Bewahre das `.img` auf einem anderen Stick oder in Cloud-Speicher auf. **Es schützt vor Header-Korruption, nicht vor einer vergessenen Passphrase.**

### 11.2 Ein Nuke-Passwort setzen

Installiere zuerst das Paket mit der Funktion (ein Debian-Patch, den Befehl gibt es standardmäßig nicht):

```bash
sudo apt install -y cryptsetup-nuke-password
sudo cryptsetup luksAddNuke ${usb}3
```

Setze ein Notfall-Passwort. Wenn du jemals gezwungen wirst, den Stick zu entsperren, löscht die Eingabe dieses Passworts den LUKS-Header und niemand kann die Daten entschlüsseln.

### 11.3 Eine Backup-Passphrase hinzufügen

```bash
sudo cryptsetup luksAddKey ${usb}3
```

### 11.4 Die Key-Slots inspizieren

```bash
sudo cryptsetup luksDump ${usb}3
```

Jeder Slot wird aufgelistet — Haupt-Passphrase, Backup-Key, Nuke-Passwort.

---

## 🩺 12. Fehlerbehebung

| Symptom | Ursache | Lösung |
|---|---|---|
| Kein `encrypted persistence`-Eintrag im Boot-Menü | andere Menü-Verschachtelung unter UEFI oder falsches ISO | schau im `Live system`-Untermenü nach; bestätige, dass du vom Stick gestartet hast |
| `/dev/sda3` fehlt nach `w` | der Kernel kann eine belegte Partitionstabelle nicht neu lesen | neu in Live starten (Schritt 4) |
| `partprobe` meldet „busy“ | das `sda1` des Sticks ist eingehängt, weil du davon gebootet hast | ignorieren, neu starten |
| Passphrase wird immer wieder abgelehnt | die initramfs nutzt ein US-Layout; die Passphrase enthält layoutabhängige Zeichen | mit reinen ASCII-Buchstaben und Ziffern neu setzen |
| Änderungen verschwinden nach dem Neustart | einfaches `Live system` gestartet; oder falsches Label; oder CRLF-Konfiguration | die beiden Prüfungen aus Schritt 8 wiederholen |
| `blkid` zeigt kein `persistence`-Label | Label falsch, oder du hast außerhalb des Mappings formatiert | `e2label /dev/mapper/my_usb persistence` |
| `fdisk` warnt `GPT PMBR size mismatch` | der Stick trägt eine übrig gebliebene GPT-Tabelle (das Kali-Image selbst ist eine DOS-Tabelle, das sollte also nicht passieren) | `sudo sgdisk -e $usb`, um die Backup-GPT zu verschieben, dann erneut `fdisk` |
| `luksFormat` sagt „Device is in use“ | die Partition wurde automatisch eingehängt | `sudo umount ${usb}3`, erneut versuchen |
| Der Start hängt eine Weile bei `cryptsetup` | normal — das Entsperren in einer Live-Umgebung ist langsam | 10–30 s warten |

---

## ⚡ 13. Cheat Sheet

Sobald `${usb}3` existiert, ist die ganze Abfolge von der Verschlüsselung bis zum Aufräumen:

```bash
usb=/dev/sda                      # bei jedem Start neu erkennen

# verschlüsseln
sudo cryptsetup --verbose --verify-passphrase luksFormat ${usb}3

# öffnen, formatieren, Label setzen
sudo cryptsetup luksOpen ${usb}3 my_usb
sudo mkfs.ext4 -L persistence /dev/mapper/my_usb

# einhängen und konfigurieren
sudo mkdir -p /mnt/my_usb
sudo mount /dev/mapper/my_usb /mnt/my_usb
echo "/ union" | sudo tee /mnt/my_usb/persistence.conf
sync

# überprüfen
sudo blkid /dev/mapper/my_usb          # LABEL="persistence" TYPE="ext4"
xxd /mnt/my_usb/persistence.conf       # endet mit 0a

# aufräumen
sudo umount /mnt/my_usb
sudo cryptsetup luksClose my_usb
sudo reboot
```

---

## ❓ FAQ

**Ich will keine Verschlüsselung — nur einfache Persistenz?**
Überspringe `luksFormat` und `luksOpen` in den Schritten 5–6, führe direkt `sudo mkfs.ext4 -L persistence ${usb}3` aus und mach ab Schritt 7 weiter. Starte den `persistence`-Eintrag.

**Es gibt ein neues Kali-Release — muss ich alles neu machen?**
Ja. Das ISO neu mit DD zu schreiben setzt die Partitionstabelle zurück, und die Persistenz-Partition geht mit. Kopiere vorher dein `~` vom Stick.

**Die Persistenz-Partition ist voll — kann ich sie vergrößern?**
Erweitere sie in den freien Platz am Ende des Sticks, dann:

```bash
sudo cryptsetup luksOpen ${usb}3 my_usb
sudo cryptsetup resize my_usb
sudo resize2fs /dev/mapper/my_usb
```

**Ich habe die LUKS-Passphrase vergessen.**
Es gibt keine Wiederherstellung. Keine Brute-Force-Abkürzung, und ein Header-Backup hilft nicht (es schützt nur vor Korruption). Darum gehört die Passphrase in einen Passwort-Manager.

**Kann ich das System „installieren“, damit `apt upgrade` normal funktioniert?**
Das ist ein anderer Weg: Kali mit dem Installer-ISO auf den Stick installieren, als wäre er eine normale Festplatte. Stabiler, aber jedes Kernel-Upgrade braucht GRUB-Pflege und es nutzt den Stick schneller ab. Der Hybrid-Live-Ansatz ist für ein portables Toolkit besser.

**Nützt die Persistenz meinen USB-Stick ab?**
Sie erzeugt zusätzliche Schreibvorgänge, aber normale Nutzung ist in Ordnung. Meide No-Name-Sticks, füll den Stick nicht komplett (ein volles Overlay verhält sich seltsam) und prüfe gelegentlich `df -h`.

**Kann ich die Persistenz-Partition ändern, während ich mit Persistenz gebootet habe?**
Nein. Das Overlay benutzt sie; neu partitionieren oder formatieren aus dieser Sitzung heraus wird deine Daten sehr wahrscheinlich zerstören. Mach es immer aus einer einfachen `Live system`-Sitzung.

---

## 🔗 Referenzen

- Kali-Doku: USB-Persistenz - <https://www.kali.org/docs/usb/usb-persistence/>
- Kali-Doku: verschlüsselte Persistenz - <https://www.kali.org/docs/usb/usb-persistence-encryption/>
- Ventoy-Persistenz (der andere Weg: eine `.dat`-Datei, keine Partitionsbearbeitung) - <https://www.ventoy.net/en/plugin_persistence.html>

## 📄 Lizenz

Veröffentlicht unter der [MIT License](LICENSE). Copyright (c) 2026 Your Name.

Pull Requests und Issue-Reports zu Stolperfallen, in die du getreten bist, sind willkommen.
