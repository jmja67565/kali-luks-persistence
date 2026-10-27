# 🐉 Persistência Criptografada em USB Live do Kali (LUKS)

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

> Transforme um USB Live do Kali em um sistema portátil que mantém suas alterações entre reinicializações — e permanece ilegível para qualquer um que encontre o pendrive.

**Testado com:** Kali 2026.3 Live ISO · USB 3.0 de 32 GB+ · gravado a partir do Windows
**Dificuldade:** ★★☆☆☆ · **Tempo:** ~20 minutos · **Risco:** todo comando tem o pendrive como alvo — identifique o disco errado e você apaga a sua unidade

---

## 🎯 O que você vai ter no final

O menu de inicialização do USB ganha três entradas:

```
Live system (amd64)                        ← one-off, everything lost on shutdown
Live system (amd64) persistence            ← plaintext persistence
Live system (amd64) encrypted persistence  ← the goal: passphrase first, then boot
```

Tudo o que você instala, configura e salva em `~` sobrevive a uma reinicialização. Se o pendrive for roubado, tudo o que um atacante vê é um contêiner LUKS que ele não consegue abrir.

---

## 🗂️ Conteúdo

- 🧠 0. Três conceitos primeiro
- 🧰 1. Pré-requisitos
- 💾 2. Gravar o ISO (modo DD)
- 🖥️ 3. Primeiro boot e identificação do pendrive
- 🧩 4. Criar a terceira partição
- 🔐 5. Criptografar com LUKS
- 📦 6. ext4 + o rótulo de persistência
- 📝 7. Escrever o persistence.conf
- ✅ 8. Verificar
- 🔒 9. Desmontar e fechar
- 🚀 10. Inicializar e verificar
- 🛡️ 11. Endurecimento
- 🩺 12. Solução de problemas
- ⚡ 13. Cola
- ❓ FAQ
- 🔗 Referências
- 📄 Licença

---

## 🧠 0. Três conceitos primeiro

Entenda estes três e todo comando abaixo se torna óbvio em vez de algo que você copia às cegas.

**① O Kali Live é somente leitura.**
A cada inicialização o sistema descompacta um sistema de arquivos raiz do ISO para a RAM. Qualquer coisa que você alterar se perde no desligamento. É exatamente por isso que a persistência existe.

**② Persistência = uma partição rotulada `persistence` + um arquivo.**
No boot, o live-boot varre os discos em busca de uma partição com `LABEL=persistence` e então a sobrepõe ao sistema de arquivos raiz, que é somente leitura. As leituras vêm do ISO, as gravações caem no pendrive. A partição só precisa de um arquivo chamado `persistence.conf` contendo uma única linha: `/ union`.

**③ Persistência criptografada = essa partição envolvida em LUKS.**
No boot o initramfs pede a frase secreta, desbloqueia o contêiner, o mapeia para `/dev/mapper/<name>` e *então* o trata exatamente como uma partição de persistência normal.

A ideia central: **o live-boot só se importa com o rótulo. Ele não se importa se o rótulo está em uma partição simples ou em um mapeamento LUKS.**

Layout final no pendrive:

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

## 🧰 1. Pré-requisitos

| Item | Requisito | Por quê |
|---|---|---|
| Pendrive | ≥ 32 GB, USB 3.0+ | a persistência grava o tempo todo; pendrives baratos travam e caem |
| Imagem | `kali-linux-2026.3-live-amd64.iso` | precisa ser **live**, não instalador |
| Gravador | Rufus (Windows) | use o **modo DD** |
| Firmware | Secure Boot desativado | o Kali Live não é assinado pela Microsoft |
| Frase secreta | guardada em um gerenciador de senhas | perca-a e os dados se vão para sempre |

**Sobre o download:** no diretório oficial `https://cdimage.kali.org/current/` a imagem Live é distribuída **apenas como `.torrent`** (cerca de 443 KiB) — o instalador é o que vem como `.iso`. Usar um cliente BitTorrent é o caminho normal aqui, não um contorno para um espelho com defeito.

**Verifique o download** — um ISO corrompido é o modo de falha mais traiçoeiro:

```powershell
# Windows PowerShell
certutil -hashfile .\kali-linux-2026.3-live-amd64.iso SHA256
```

```bash
# Linux / macOS
sha256sum kali-linux-2026.3-live-amd64.iso
```

Compare caractere por caractere com `https://cdimage.kali.org/kali-2026.3/SHA256SUMS`.

---

## 💾 2. Gravar o ISO (modo DD)

No Rufus: **Device** → o seu pendrive · **Boot selection** → o ISO · quando o diálogo de modo aparecer, escolha **DD Image mode** · Start.

> ⚠️ **Nunca mexa no controle deslizante "Persistent partition size".**
> O Rufus implementa persistência aplicando patches em distros baseadas em casper (Ubuntu, Mint). O Kali usa live-boot + isohybrid, então o controle reescreve o layout de partições, o GRUB deixa de encontrar o kernel e você cai em um prompt `grub>` cru. Refaça a gravação do zero.

Quando terminar, **não** abra o pendrive com nenhuma ferramenta de particionamento do Windows (Gerenciamento de Disco, DiskGenius, AOMEI…).

---

## 🖥️ 3. Primeiro boot e identificação do pendrive

Inicialize pelo USB e escolha **`Live system (amd64)`** (a entrada simples).

> Por que não a entrada de persistência? Ainda não existe partição de persistência, então seria inútil — e as partições precisam ser modificadas a partir de uma **sessão não persistente**, caso contrário o overlay está usando ativamente o disco que você está editando.

Abra um terminal e liste os discos:

```bash
lsblk -o NAME,SIZE,RM,FSTYPE,LABEL,MOUNTPOINTS
```

Identifique o pendrive usando três sinais em conjunto:

- `RM=1` (removível)
- uma de suas partições está montada em `/run/live/medium` ou `/lib/live/mount/medium` (essa é o ISO em execução)
- a capacidade corresponde ao pendrive na sua mão

Aproveite para conferir o modo de gravação:

```bash
lsblk -f
```

- ✅ `sda1` é `iso9660` e `sda2` é `vfat` (~4 MB) → **modo DD, correto**
- ❌ uma única partição `vfat` grande → gravado no modo ISO, volte ao passo 2

**Detecção automática** (recomendado — um erro de digitação aqui apaga o seu notebook):

```bash
medium=$(findmnt -no SOURCE /run/live/medium 2>/dev/null || findmnt -no SOURCE /lib/live/mount/medium)
usb=/dev/$(lsblk -no PKNAME "$medium")
echo "USB = $usb"
lsblk "$usb"
```

A saída se parece com isto:

```
NAME   MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
sda      8:0    1  57.3G  0 disk
├─sda1   8:1    1   4.6G  0 part /run/live/medium
└─sda2   8:2    1     4M  0 part
```

Fixe isso em uma variável — todos os comandos seguintes referenciam `$usb`, então um erro de digitação não pode atingir o disco errado:

```bash
usb=/dev/sda     # use o que você viu acima
```

---

## 🧩 4. Criar a terceira partição

Inspecione a tabela atual:

```bash
sudo fdisk -l $usb
```

Espere `Disklabel type: dos` e duas partições: cerca de 4.x GB (ISO) e alguns MB (EFI).

Inicie o editor:

```bash
sudo fdisk $usb
```

Digite estas teclas na ordem:

| Tecla | Significado |
|---|---|
| `p` ↵ | imprime a tabela, confirme que há só sda1 + sda2 |
| `n` ↵ | nova partição |
| `p` ↵ | primária |
| `3` ↵ | número da partição (Enter aceita o padrão 3) |
| ↵ | primeiro setor: padrão, logo depois de sda2 |
| ↵ | último setor: padrão, até o fim do disco |
| `w` ↵ | grava e sai |

> ⚠️ Por que `fdisk` e não GParted?
> Um ISO híbrido gravado em DD é um "disco dentro de um disco": o dispositivo inteiro é visto como um único bloco ISO9660 enorme, então o GParted reporta o pendrive inteiro como uma única partição e não mostra espaço livre. Isso não é um bug, é a natureza da imagem. O `fdisk` edita a tabela de partições subjacente diretamente e funciona bem.

**Agora o passo em que a maioria das pessoas fica travada.**

Depois de `w` você provavelmente verá:

```
The kernel still uses the old table.
The new table will be used at the next reboot.
Re-reading the partition table failed.: Device or resource busy
```

A razão é simples: **você inicializou por este mesmo pendrive, então `sda1` ainda está montada** e o kernel se recusa a reler a tabela. Então:

```bash
lsblk $usb
```

- **`sda3` está lá** → continue no passo 5
- **ela não existe** → não brigue com o `partprobe` (ele falha com o mesmo "busy"), simplesmente **reinicialize para `Live system (amd64)` novamente**. A tabela já está no disco; o kernel a lê no boot. Redetecte o disco (passo 3), confirme que `sda3` existe e continue.

---

## 🔐 5. Criptografar com LUKS

Verificação final de segurança — imprima o dispositivo e olhe para ele antes de pressionar Enter:

```bash
[ -b ${usb}3 ] && lsblk -o NAME,SIZE,TYPE ${usb}3
```

Depois:

```bash
sudo cryptsetup --verbose --verify-passphrase luksFormat ${usb}3
```

Interação:

1. `WARNING! This will overwrite data on /dev/sda3 irrevocably.` → digite **`YES`** (tudo em maiúsculas; `yes` é rejeitado)
2. `Enter passphrase for /dev/sda3:` → sua frase secreta
3. `Verify passphrase:` → de novo

**Como escolher a frase secreta (é aqui que as pessoas falham):**

- O desbloqueio acontece no initramfs, onde o layout do teclado é **inglês americano**. Caracteres que mudam entre layouts (`ä`, `@` em um teclado alemão, …) não vão digitar o que você pensa que estão digitando
- Recomendado: **12+ caracteres, apenas letras minúsculas e dígitos**, ou uma frase secreta longa que você consiga recitar de fato
- `--verify-passphrase` simplesmente pede duas vezes para proteger contra erros de digitação

> Esta é a frase secreta que você digita a cada inicialização. **Se perdê-la, os dados são irrecuperáveis.** Mantenha uma cópia no seu gerenciador de senhas.

---

## 📦 6. ext4 + o rótulo de persistência

Desbloqueie o contêiner:

```bash
sudo cryptsetup luksOpen ${usb}3 my_usb
```

Digite a frase secreta. Em caso de sucesso, `/dev/mapper/my_usb` é o dispositivo *descriptografado* — **toda formatação e gravação daqui em diante tem como alvo ele, nunca mais `${usb}3`**.

```bash
sudo mkfs.ext4 -L persistence /dev/mapper/my_usb
sudo e2label /dev/mapper/my_usb
```

`e2label` deve imprimir:

```
persistence
```

> Por que o rótulo precisa ser `persistence`? O live-boot encontra a partição de persistência por `LABEL=persistence`. `persist`, `Persistence` ou `persistence ` (com espaço no final) são todos ignorados silenciosamente.

---

## 📝 7. Escrever o persistence.conf

```bash
sudo mkdir -pv /mnt/my_usb
sudo mount -v /dev/mapper/my_usb /mnt/my_usb
echo "/ union" | sudo tee /mnt/my_usb/persistence.conf
sync
```

`/ union` significa: sobreponha o sistema de arquivos raiz inteiro `/` nesta partição, para que todas as gravações caiam aqui.

> ⚠️ Escreva com `echo` dentro do Linux. Se você criar o arquivo no Bloco de Notas do Windows, ele fica com finais de linha CRLF, o live-boot interpreta `/ union\r`, o caminho não existe e a persistência silenciosamente não faz nada. Esta é a falha mais comum de todas.

---

## ✅ 8. Verificar

As duas verificações precisam passar:

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

O arquivo precisa terminar com `0a` (LF). Se você vir `0d0a`, é CRLF — apague e reescreva.

Mais uma, fácil de ler de relance:

```bash
stat -c '%s bytes' /mnt/my_usb/persistence.conf   # deve ser 8
```

---

## 🔒 9. Desmontar e fechar

```bash
sudo umount -v /mnt/my_usb
sudo cryptsetup luksClose my_usb
sudo reboot
```

A ordem importa: **desmonte o sistema de arquivos primeiro, feche o mapeamento depois** — caso contrário você recebe `Device or resource busy`.

---

## 🚀 10. Inicializar e verificar

Escolha **`Live system (amd64) encrypted persistence`** no menu de inicialização.

Você deve ser solicitado com algo como:

```
Enter passphrase for /dev/sda3:
```

Digite a frase secreta, chegue à área de trabalho e então verifique duas coisas.

**① Os dados estão realmente criptografados** (um pendrive roubado é inútil):

```bash
lsblk -f
```

`sda3` precisa mostrar `crypto_LUKS`, com o mapeamento `my_usb` logo abaixo contendo `ext4` e o rótulo `persistence`.

**② A persistência realmente funciona:**

```bash
echo hello > ~/enc-test
sudo reboot
```

Inicialize pela mesma entrada novamente, desbloqueie e então:

```bash
cat ~/enc-test     # imprime hello → pronto
```

---

## 🛡️ 11. Endurecimento

### 11.1 Fazer backup do cabeçalho do LUKS

Um cabeçalho danificado significa que a partição inteira fica ilegível, e ele fica nos primeiros megabytes, onde uma única gravação perdida pode destruí-lo.

```bash
sudo cryptsetup luksHeaderBackup ${usb}3 --header-backup-file ~/luks-header-backup.img
```

Guarde esse `.img` em outro pendrive ou em armazenamento na nuvem. **Ele protege contra corrupção do cabeçalho, não contra uma frase secreta esquecida.**

### 11.2 Definir uma senha nuke

Instale primeiro o pacote que fornece o recurso (é um patch do Debian, o comando não existe por padrão):

```bash
sudo apt install -y cryptsetup-nuke-password
sudo cryptsetup luksAddNuke ${usb}3
```

Defina uma senha de emergência. Se algum dia você for forçado a desbloquear o pendrive, digitar essa senha apaga o cabeçalho do LUKS e ninguém consegue descriptografar os dados.

### 11.3 Adicionar uma frase secreta de backup

```bash
sudo cryptsetup luksAddKey ${usb}3
```

### 11.4 Inspecionar os slots de chave

```bash
sudo cryptsetup luksDump ${usb}3
```

Cada slot é listado — frase secreta principal, chave de backup, senha nuke.

---

## 🩺 12. Solução de problemas

| Sintoma | Causa | Correção |
|---|---|---|
| Nenhuma entrada `encrypted persistence` no menu de inicialização | aninhamento de menu diferente no UEFI, ou ISO errado | procure dentro do submenu `Live system`; confirme que você inicializou pelo pendrive |
| `/dev/sda3` ausente depois de `w` | o kernel não consegue reler uma tabela de partições em uso | reinicialize para o Live (passo 4) |
| `partprobe` reporta "busy" | a `sda1` do pendrive está montada porque você inicializou por ele | ignore, reinicialize |
| Frase secreta rejeitada repetidamente | o initramfs usa layout americano; a frase contém caracteres dependentes de layout | recadastre a chave com letras ASCII simples e dígitos |
| As alterações desaparecem depois de reiniciar | você inicializou o `Live system` simples; ou rótulo errado; ou config com CRLF | refaça as duas verificações do passo 8 |
| `blkid` não mostra o rótulo `persistence` | rótulo errado, ou você formatou fora do mapeamento | `e2label /dev/mapper/my_usb persistence` |
| `fdisk` avisa `GPT PMBR size mismatch` | o pendrive carrega uma tabela GPT residual (a imagem do Kali em si é uma tabela DOS, então isso não deveria acontecer) | `sudo sgdisk -e $usb` para realocar o GPT de backup, depois `fdisk` novamente |
| `luksFormat` diz "Device is in use" | a partição foi montada automaticamente | `sudo umount ${usb}3`, tente de novo |
| A inicialização trava em `cryptsetup` por um tempo | normal — o desbloqueio em um ambiente Live é lento | espere 10–30 s |

---

## ⚡ 13. Cola

Depois que `${usb}3` existe, toda a sequência, da criptografia à limpeza, é:

```bash
usb=/dev/sda                      # redetecte a cada inicialização

# criptografar
sudo cryptsetup --verbose --verify-passphrase luksFormat ${usb}3

# abrir, formatar, rotular
sudo cryptsetup luksOpen ${usb}3 my_usb
sudo mkfs.ext4 -L persistence /dev/mapper/my_usb

# montar e configurar
sudo mkdir -p /mnt/my_usb
sudo mount /dev/mapper/my_usb /mnt/my_usb
echo "/ union" | sudo tee /mnt/my_usb/persistence.conf
sync

# verificar
sudo blkid /dev/mapper/my_usb          # LABEL="persistence" TYPE="ext4"
xxd /mnt/my_usb/persistence.conf       # termina com 0a

# limpar
sudo umount /mnt/my_usb
sudo cryptsetup luksClose my_usb
sudo reboot
```

---

## ❓ FAQ

**Não quero criptografia — só persistência simples?**
Pule `luksFormat` e `luksOpen` nos passos 5–6, rode `sudo mkfs.ext4 -L persistence ${usb}3` diretamente e então continue a partir do passo 7. Inicie pela entrada `persistence`.

**Saiu uma nova versão do Kali — preciso refazer tudo?**
Sim. Regravar o ISO em DD redefine a tabela de partições e a partição de persistência vai junto. Antes, copie o seu `~` para fora do pendrive.

**A partição de persistência está cheia — posso aumentá-la?**
Estenda-a até o espaço livre no fim do pendrive, depois:

```bash
sudo cryptsetup luksOpen ${usb}3 my_usb
sudo cryptsetup resize my_usb
sudo resize2fs /dev/mapper/my_usb
```

**Esqueci a frase secreta do LUKS.**
Não há recuperação. Nem atalho de força bruta, e um backup do cabeçalho não ajuda (ele só protege contra corrupção). É por isso que a frase secreta pertence a um gerenciador de senhas.

**Posso "instalar" o sistema para o `apt upgrade` funcionar normalmente?**
É outra rota: instalar o Kali no pendrive com o ISO instalador, tratando-o como um disco comum. Mais estável, mas cada atualização de kernel exige cuidado com o GRUB e desgasta o pendrive mais rápido. A abordagem híbrida Live é melhor para um kit portátil.

**A persistência vai desgastar o meu pendrive?**
Ela adiciona gravações, mas o uso normal está tranquilo. Evite pendrives sem marca, não encha o pendrive por completo (um overlay cheio se comporta de forma estranha) e confira `df -h` de vez em quando.

**Posso modificar a partição de persistência enquanto estou inicializado com persistência?**
Não. O overlay está usando ela; reparticionar ou reformatar a partir dessa sessão muito provavelmente vai corromper seus dados. Faça isso sempre a partir de uma sessão `Live system` simples.

---

## 🔗 Referências

- Kali docs: persistência em USB - <https://www.kali.org/docs/usb/usb-persistence/>
- Kali docs: persistência criptografada - <https://www.kali.org/docs/usb/usb-persistence-encryption/>
- Persistência do Ventoy (a outra rota: um arquivo `.dat`, sem edição de partições) - <https://www.ventoy.net/en/plugin_persistence.html>

## 📄 Licença

Lançado sob a [Licença MIT](LICENSE). Copyright (c) 2026 Your Name.

Pull requests e relatos de problemas sobre armadilhas que você encontrar são bem-vindos.
