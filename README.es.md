# 🐉 Persistencia cifrada en un USB Live de Kali (LUKS)

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

> Convierte un USB Live de Kali en un sistema portátil que conserva tus cambios entre reinicios, y que sigue siendo ilegible para cualquiera que encuentre el pendrive.

**Probado con:** Kali 2026.3 Live ISO · USB 3.0 de 32 GB o más · escrito desde Windows
**Dificultad:** ★★☆☆☆ · **Tiempo:** ~20 minutos · **Riesgo:** todos los comandos apuntan al pendrive USB: si identificas mal el disco, borras tu unidad

---

## 🎯 Qué obtienes al final

El menú de arranque del USB gana tres entradas:

```
Live system (amd64)                        ← one-off, everything lost on shutdown
Live system (amd64) persistence            ← plaintext persistence
Live system (amd64) encrypted persistence  ← the goal: passphrase first, then boot
```

Todo lo que instales, configures y guardes bajo `~` sobrevive a un reinicio. Si te roban el pendrive, lo único que verá un atacante es un contenedor LUKS que no puede abrir.

---

## 🗂️ Contenido

- 🧠 0. Primero, tres conceptos
- 🧰 1. Requisitos previos
- 💾 2. Escribe la ISO (modo DD)
- 🖥️ 3. Primer arranque e identificación del pendrive
- 🧩 4. Crea la tercera partición
- 🔐 5. Cifra con LUKS
- 📦 6. ext4 + la etiqueta de persistencia
- 📝 7. Escribe persistence.conf
- ✅ 8. Verifica
- 🔒 9. Desmonta y cierra
- 🚀 10. Arranca y verifica
- 🛡️ 11. Endurecimiento
- 🩺 12. Solución de problemas
- ⚡ 13. Chuleta
- ❓ Preguntas frecuentes
- 🔗 Referencias
- 📄 Licencia

---

## 🧠 0. Primero, tres conceptos

Entiende estos tres y cada comando de abajo se vuelve obvio en lugar de algo que copias a ciegas.

**① Kali Live es de solo lectura.**
En cada arranque, el sistema descomprime un sistema de archivos raíz desde la ISO a la RAM. Todo lo que cambies desaparece al apagar. Esa es exactamente la razón por la que existe la persistencia.

**② Persistencia = una partición con la etiqueta `persistence` + un archivo.**
Al arrancar, live-boot busca en los discos una partición con `LABEL=persistence` y luego le hace overlay encima del sistema de archivos raíz de solo lectura. Las lecturas vienen de la ISO y las escrituras caen en el pendrive. La partición solo necesita un archivo llamado `persistence.conf` con una única línea: `/ union`.

**③ Persistencia cifrada = esa partición envuelta en LUKS.**
Al arrancar, el initramfs pide la frase de contraseña, desbloquea el contenedor, lo mapea a `/dev/mapper/<nombre>` y *después* lo trata exactamente como una partición de persistencia normal.

La idea clave: **a live-boot solo le importa la etiqueta. No le importa si la etiqueta vive en una partición normal o en un mapeo LUKS.**

Distribución final en el pendrive:

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

## 🧰 1. Requisitos previos

| Elemento | Requisito | Por qué |
|---|---|---|
| Pendrive USB | ≥ 32 GB, USB 3.0+ | la persistencia escribe constantemente; los pendrives baratos se atascan y se desconectan |
| Imagen | `kali-linux-2026.3-live-amd64.iso` | debe ser **live**, no el instalador |
| Grabador | Rufus (Windows) | usa el **modo DD** |
| Firmware | Secure Boot desactivado | Kali Live no está firmado por Microsoft |
| Frase de contraseña | guardada en un gestor de contraseñas | si la pierdes, los datos se pierden para siempre |

**Sobre la descarga:** en el directorio oficial `https://cdimage.kali.org/current/` la imagen Live se distribuye **solo como `.torrent`** (unos 443 KiB); la que se ofrece como `.iso` es la del instalador. Usar un cliente BitTorrent es el camino normal aquí, no un apaño para un espejo roto.

**Verifica la descarga** — una ISO corrupta es el modo de fallo más traicionero:

```powershell
# Windows PowerShell
certutil -hashfile .\kali-linux-2026.3-live-amd64.iso SHA256
```

```bash
# Linux / macOS
sha256sum kali-linux-2026.3-live-amd64.iso
```

Compara carácter por carácter con `https://cdimage.kali.org/kali-2026.3/SHA256SUMS`.

---

## 💾 2. Escribe la ISO (modo DD)

En Rufus: **Dispositivo** → tu pendrive · **Selección de arranque** → la ISO · cuando aparezca el diálogo de modo, elige **DD Image mode** · Empezar.

> ⚠️ **Nunca toques el deslizador "Persistent partition size".**
> Rufus implementa la persistencia parcheando distros basadas en casper (Ubuntu, Mint). Kali usa live-boot + isohybrid, así que el deslizador reescribe la tabla de particiones, GRUB ya no encuentra el kernel y acabas en un prompt `grub>` pelado. Vuelve a escribir la ISO desde cero.

Cuando termine, **no** abras el pendrive con ninguna herramienta de particiones de Windows (Administración de discos, DiskGenius, AOMEI…).

---

## 🖥️ 3. Primer arranque e identificación del pendrive

Arranca desde el USB y elige **`Live system (amd64)`** (la entrada simple).

> ¿Por qué no la entrada de persistencia? Todavía no existe ninguna partición de persistencia, así que no serviría de nada; y las particiones deben modificarse desde una **sesión no persistente**, porque si no el overlay está usando activamente el disco que estás editando.

Abre un terminal y lista los discos:

```bash
lsblk -o NAME,SIZE,RM,FSTYPE,LABEL,MOUNTPOINTS
```

Identifica el pendrive combinando tres señales a la vez:

- `RM=1` (extraíble)
- una de sus particiones está montada en `/run/live/medium` o `/lib/live/mount/medium` (esa es la ISO en ejecución)
- la capacidad coincide con el pendrive que tienes en la mano

Aprovecha para comprobar el modo de escritura:

```bash
lsblk -f
```

- ✅ `sda1` es `iso9660` y `sda2` es `vfat` (~4 MB) → **modo DD, correcto**
- ❌ una única partición `vfat` grande → escrita en modo ISO, vuelve al paso 2

**Detección automática** (recomendada — una errata aquí borra tu portátil):

```bash
medium=$(findmnt -no SOURCE /run/live/medium 2>/dev/null || findmnt -no SOURCE /lib/live/mount/medium)
usb=/dev/$(lsblk -no PKNAME "$medium")
echo "USB = $usb"
lsblk "$usb"
```

La salida se parece a esto:

```
NAME   MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
sda      8:0    1  57.3G  0 disk
├─sda1   8:1    1   4.6G  0 part /run/live/medium
└─sda2   8:2    1     4M  0 part
```

Fíjalo en una variable: todos los comandos posteriores usan `$usb`, así que una errata no puede golpear el disco equivocado:

```bash
usb=/dev/sda     # usa lo que viste arriba
```

---

## 🧩 4. Crea la tercera partición

Inspecciona la tabla actual:

```bash
sudo fdisk -l $usb
```

Espera ver `Disklabel type: dos` y dos particiones: unos 4.x GB (ISO) y unos pocos MB (EFI).

Abre el editor:

```bash
sudo fdisk $usb
```

Introduce estas teclas en orden:

| Tecla | Significado |
|---|---|
| `p` ↵ | imprime la tabla, confirma que solo hay sda1 + sda2 |
| `n` ↵ | nueva partición |
| `p` ↵ | primaria |
| `3` ↵ | número de partición (Enter acepta el 3 por defecto) |
| ↵ | primer sector: por defecto, justo después de sda2 |
| ↵ | último sector: por defecto, hasta el final del disco |
| `w` ↵ | escribir y salir |

> ⚠️ ¿Por qué `fdisk` y no GParted?
> Una ISO híbrida escrita en DD es un "disco dentro de un disco": todo el dispositivo se ve como un único bloque ISO9660 enorme, así que GParted informa el pendrive entero como una sola partición y no muestra espacio libre. Eso no es un fallo, es la naturaleza de la imagen. `fdisk` edita la tabla de particiones subyacente directamente y funciona bien.

**Ahora el paso donde la mayoría se atasca.**

Después de `w` probablemente veas:

```
The kernel still uses the old table.
The new table will be used at the next reboot.
Re-reading the partition table failed.: Device or resource busy
```

La razón es simple: **has arrancado desde este mismo pendrive, así que `sda1` sigue montada** y el kernel se niega a releer la tabla. Entonces:

```bash
lsblk $usb
```

- **`sda3` está ahí** → continúa con el paso 5
- **no aparece** → no te pelees con `partprobe` (falla con el mismo "busy"), simplemente **reinicia y vuelve a entrar en `Live system (amd64)`**. La tabla ya está en el disco; el kernel la recoge al arrancar. Vuelve a detectar el disco (paso 3), confirma que `sda3` existe y continúa.

---

## 🔐 5. Cifra con LUKS

Comprobación de seguridad final: imprime el dispositivo y míralo antes de pulsar Enter:

```bash
[ -b ${usb}3 ] && lsblk -o NAME,SIZE,TYPE ${usb}3
```

Luego:

```bash
sudo cryptsetup --verbose --verify-passphrase luksFormat ${usb}3
```

Interacción:

1. `WARNING! This will overwrite data on /dev/sda3 irrevocably.` → escribe **`YES`** (todo en mayúsculas; `yes` se rechaza)
2. `Enter passphrase for /dev/sda3:` → tu frase de contraseña
3. `Verify passphrase:` → otra vez

**Cómo elegir la frase de contraseña (aquí es donde la gente falla):**

- El desbloqueo ocurre en el initramfs, donde la distribución del teclado es **inglés de EE. UU.**. Los caracteres que se mueven entre distribuciones (`ä`, `@` en un teclado alemán, …) no escribirán lo que crees que escriben
- Recomendado: **12 caracteres o más, solo letras minúsculas y dígitos**, o una frase de contraseña larga que puedas recitar de verdad
- `--verify-passphrase` simplemente pregunta dos veces para protegerte de las erratas

> Esta es la frase de contraseña que escribes en cada arranque. **Si la pierdes, los datos son irrecuperables.** Guarda una copia en tu gestor de contraseñas.

---

## 📦 6. ext4 + la etiqueta de persistencia

Desbloquea el contenedor:

```bash
sudo cryptsetup luksOpen ${usb}3 my_usb
```

Introduce la frase de contraseña. Si funciona, `/dev/mapper/my_usb` es el dispositivo *descifrado*: **todo formateo y toda escritura a partir de ahora van ahí, nunca más a `${usb}3`**.

```bash
sudo mkfs.ext4 -L persistence /dev/mapper/my_usb
sudo e2label /dev/mapper/my_usb
```

`e2label` debe imprimir:

```
persistence
```

> ¿Por qué la etiqueta debe ser `persistence`? live-boot encuentra la partición de persistencia por `LABEL=persistence`. `persist`, `Persistence` o `persistence ` (con un espacio al final) se ignoran en silencio.

---

## 📝 7. Escribe persistence.conf

```bash
sudo mkdir -pv /mnt/my_usb
sudo mount -v /dev/mapper/my_usb /mnt/my_usb
echo "/ union" | sudo tee /mnt/my_usb/persistence.conf
sync
```

`/ union` significa: hacer overlay de todo el sistema de archivos raíz `/` sobre esta partición, de modo que todas las escrituras caigan aquí.

> ⚠️ Escríbelo con `echo` dentro de Linux. Si creas el archivo con el Bloc de notas de Windows, tendrá finales de línea CRLF, live-boot interpretará `/ union\r`, la ruta no existirá y la persistencia no hará nada en silencio. Este es el fallo más común con diferencia.

---

## ✅ 8. Verifica

Deben pasar las dos comprobaciones:

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

El archivo debe terminar en `0a` (LF). Si ves `0d0a`, es CRLF: bórralo y vuelve a escribirlo.

Una más, fácil de leer de un vistazo:

```bash
stat -c '%s bytes' /mnt/my_usb/persistence.conf   # debe ser 8
```

---

## 🔒 9. Desmonta y cierra

```bash
sudo umount -v /mnt/my_usb
sudo cryptsetup luksClose my_usb
sudo reboot
```

El orden importa: **primero desmonta el sistema de archivos, después cierra el mapeo**; si no, obtienes `Device or resource busy`.

---

## 🚀 10. Arranca y verifica

Elige **`Live system (amd64) encrypted persistence`** en el menú de arranque.

Debería aparecerte un aviso como:

```
Enter passphrase for /dev/sda3:
```

Escribe la frase de contraseña, llega al escritorio y verifica dos cosas.

**① Los datos están realmente cifrados** (un pendrive robado es inservible):

```bash
lsblk -f
```

`sda3` debe mostrar `crypto_LUKS`, con el mapeo `my_usb` justo debajo llevando `ext4` y la etiqueta `persistence`.

**② La persistencia funciona de verdad:**

```bash
echo hello > ~/enc-test
sudo reboot
```

Arranca otra vez la misma entrada, desbloquea y luego:

```bash
cat ~/enc-test     # imprime hello → listo
```

---

## 🛡️ 11. Endurecimiento

### 11.1 Haz una copia de seguridad de la cabecera LUKS

Una cabecera dañada hace que toda la partición sea ilegible, y vive en los primeros megabytes, donde una sola escritura perdida puede destruirla.

```bash
sudo cryptsetup luksHeaderBackup ${usb}3 --header-backup-file ~/luks-header-backup.img
```

Guarda ese `.img` en otro pendrive o en el almacenamiento en la nube. **Protege contra la corrupción de la cabecera, no contra una frase de contraseña olvidada.**

### 11.2 Configura una contraseña nuke

Instala primero el paquete que proporciona la función (es un parche de Debian; el comando no existe por defecto):

```bash
sudo apt install -y cryptsetup-nuke-password
sudo cryptsetup luksAddNuke ${usb}3
```

Configura una contraseña de emergencia. Si alguna vez te obligan a desbloquear el pendrive, escribir esta borra la cabecera LUKS y nadie podrá descifrar los datos.

### 11.3 Añade una frase de contraseña de respaldo

```bash
sudo cryptsetup luksAddKey ${usb}3
```

### 11.4 Inspecciona las ranuras de claves

```bash
sudo cryptsetup luksDump ${usb}3
```

Cada ranura aparece listada: frase de contraseña principal, clave de respaldo, contraseña nuke.

---

## 🩺 12. Solución de problemas

| Síntoma | Causa | Solución |
|---|---|---|
| No hay entrada `encrypted persistence` en el menú de arranque | el anidamiento del menú es distinto bajo UEFI, o la ISO es incorrecta | busca dentro del submenú `Live system`; confirma que arrancaste el pendrive |
| Falta `/dev/sda3` después de `w` | el kernel no puede releer una tabla de particiones ocupada | reinicia en Live (paso 4) |
| `partprobe` informa "busy" | la `sda1` del pendrive está montada porque arrancaste desde él | ignóralo, reinicia |
| La frase de contraseña se rechaza una y otra vez | el initramfs usa distribución de EE. UU.; la frase contiene caracteres que dependen de la distribución | vuelve a poner la clave con letras ASCII y dígitos |
| Los cambios desaparecen tras reiniciar | arrancaste el `Live system` simple; o la etiqueta está mal; o la configuración tiene CRLF | repite las dos comprobaciones del paso 8 |
| `blkid` no muestra la etiqueta `persistence` | la etiqueta está mal, o formateaste fuera del mapeo | `e2label /dev/mapper/my_usb persistence` |
| `fdisk` avisa `GPT PMBR size mismatch` | el pendrive arrastra una tabla GPT sobrante (la imagen de Kali es una tabla DOS, así que esto no debería pasar) | `sudo sgdisk -e $usb` para reubicar el GPT de respaldo y luego `fdisk` otra vez |
| `luksFormat` dice "Device is in use" | la partición se montó automáticamente | `sudo umount ${usb}3`, reintenta |
| El arranque se queda un rato en `cryptsetup` | es normal: desbloquear en un entorno Live es lento | espera 10–30 s |

---

## ⚡ 13. Chuleta

Una vez que existe `${usb}3`, toda la secuencia desde el cifrado hasta la limpieza es:

```bash
usb=/dev/sda                      # vuelve a detectar en cada arranque

# cifrar
sudo cryptsetup --verbose --verify-passphrase luksFormat ${usb}3

# abrir, formatear, etiquetar
sudo cryptsetup luksOpen ${usb}3 my_usb
sudo mkfs.ext4 -L persistence /dev/mapper/my_usb

# montar y configurar
sudo mkdir -p /mnt/my_usb
sudo mount /dev/mapper/my_usb /mnt/my_usb
echo "/ union" | sudo tee /mnt/my_usb/persistence.conf
sync

# verificar
sudo blkid /dev/mapper/my_usb          # LABEL="persistence" TYPE="ext4"
xxd /mnt/my_usb/persistence.conf       # termina en 0a

# limpiar
sudo umount /mnt/my_usb
sudo cryptsetup luksClose my_usb
sudo reboot
```

---

## ❓ Preguntas frecuentes

**No quiero cifrado, ¿solo persistencia normal?**
Sáltate `luksFormat` y `luksOpen` de los pasos 5–6, ejecuta `sudo mkfs.ext4 -L persistence ${usb}3` directamente y luego continúa desde el paso 7. Arranca la entrada `persistence`.

**Ha salido una nueva versión de Kali, ¿tengo que rehacerlo todo?**
Sí. Reescribir la ISO con DD restablece la tabla de particiones y la partición de persistencia se va con ella. Copia tu `~` fuera del pendrive primero.

**La partición de persistencia está llena, ¿puedo ampliarla?**
Extiéndela hasta el espacio libre al final del pendrive y luego:

```bash
sudo cryptsetup luksOpen ${usb}3 my_usb
sudo cryptsetup resize my_usb
sudo resize2fs /dev/mapper/my_usb
```

**He olvidado la frase de contraseña de LUKS.**
No hay recuperación. No hay atajo de fuerza bruta, y una copia de la cabecera no ayuda (solo protege contra la corrupción). Por eso la frase de contraseña va en un gestor de contraseñas.

**¿Puedo "instalar" el sistema para que `apt upgrade` funcione con normalidad?**
Es otra ruta: instalar Kali en el pendrive con la ISO del instalador, tratándolo como un disco normal. Es más estable, pero cada actualización del kernel exige cuidado con GRUB y desgasta el pendrive más rápido. El enfoque Live híbrido es mejor para un kit portátil.

**¿La persistencia desgastará mi pendrive USB?**
Añade escrituras, pero el uso normal está bien. Evita pendrives sin marca, no llenes el pendrive por completo (un overlay lleno se comporta de forma extraña) y revisa `df -h` de vez en cuando.

**¿Puedo modificar la partición de persistencia mientras estoy arrancado con persistencia?**
No. El overlay la está usando; reparticionar o reformatear desde esa sesión muy probablemente corromperá tus datos. Hazlo siempre desde una sesión `Live system` simple.

---

## 🔗 Referencias

- Documentación de Kali: persistencia USB - <https://www.kali.org/docs/usb/usb-persistence/>
- Documentación de Kali: persistencia cifrada - <https://www.kali.org/docs/usb/usb-persistence-encryption/>
- Persistencia con Ventoy (la otra ruta: un archivo `.dat`, sin editar particiones) - <https://www.ventoy.net/en/plugin_persistence.html>

## 📄 Licencia

Publicado bajo la [MIT License](LICENSE). Copyright (c) 2026 jmja67565.

Se agradecen los pull requests y los reportes de problemas sobre los escollos con los que te hayas encontrado.
