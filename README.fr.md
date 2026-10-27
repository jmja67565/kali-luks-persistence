# 🐉 Kali Live USB à persistance chiffrée (LUKS)

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

> Transforme une clé USB Kali Live en un système portable qui conserve tes modifications d'un redémarrage à l'autre — et qui reste illisible pour quiconque trouve la clé.

**Testé avec :** Kali 2026.3 Live ISO · clé USB 3.0 de 32 Go ou plus · écrit depuis Windows
**Difficulté :** ★★☆☆☆ · **Durée :** ~20 minutes · **Risque :** chaque commande vise la clé USB — identifie le mauvais disque et tu effaces ton disque dur

---

## 🎯 Ce que tu obtiens au final

Le menu d'amorçage de la clé USB gagne trois entrées :

```
Live system (amd64)                        ← one-off, everything lost on shutdown
Live system (amd64) persistence            ← plaintext persistence
Live system (amd64) encrypted persistence  ← the goal: passphrase first, then boot
```

Tout ce que tu installes, configures et enregistres sous `~` survit à un redémarrage. Si la clé est volée, un attaquant ne voit qu'un conteneur LUKS qu'il ne peut pas ouvrir.

---

## 🗂️ Sommaire

- 🧠 0. Trois concepts d'abord
- 🧰 1. Prérequis
- 💾 2. Écrire l'ISO (mode DD)
- 🖥️ 3. Premier démarrage et identification de la clé
- 🧩 4. Créer la troisième partition
- 🔐 5. Chiffrer avec LUKS
- 📦 6. ext4 + l'étiquette de persistance
- 📝 7. Écrire persistence.conf
- ✅ 8. Vérifier
- 🔒 9. Désmonter et fermer
- 🚀 10. Démarrer et vérifier
- 🛡️ 11. Durcissement
- 🩺 12. Dépannage
- ⚡ 13. Aide-mémoire
- ❓ FAQ
- 🔗 Références
- 📄 Licence

---

## 🧠 0. Trois concepts d'abord

Comprends ces trois points et chaque commande ci-dessous devient évidente au lieu d'être quelque chose que tu copies aveuglément.

**① Kali Live est en lecture seule.**
À chaque démarrage, le système décompresse un système de fichiers racine depuis l'ISO vers la RAM. Tout ce que tu modifies disparaît à l'arrêt. C'est exactement pour cela que la persistance existe.

**② Persistance = une partition étiquetée `persistence` + un fichier.**
Au démarrage, live-boot scanne les disques à la recherche d'une partition portant `LABEL=persistence`, puis la superpose au système de fichiers racine en lecture seule. Les lectures viennent de l'ISO, les écritures atterrissent sur la clé. La partition a seulement besoin d'un fichier nommé `persistence.conf` contenant une seule ligne : `/ union`.

**③ Persistance chiffrée = cette partition enveloppée dans LUKS.**
Au démarrage, l'initramfs demande la phrase secrète, déverrouille le conteneur, le mappe sur `/dev/mapper/<name>`, et *ensuite* le traite exactement comme une partition de persistance normale.

L'idée clé : **live-boot ne se soucie que de l'étiquette. Il ne se soucie pas de savoir si l'étiquette vit sur une partition simple ou sur un mapping LUKS.**

Disposition finale sur la clé :

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

## 🧰 1. Prérequis

| Élément | Exigence | Pourquoi |
|---|---|---|
| Clé USB | ≥ 32 Go, USB 3.0+ | la persistance écrit en permanence ; les clés bon marché se bloquent et se déconnectent |
| Image | `kali-linux-2026.3-live-amd64.iso` | doit être **live**, pas l'installeur |
| Outil d'écriture | Rufus (Windows) | utilise le **mode DD** |
| Firmware | Secure Boot désactivé | Kali Live n'est pas signé par Microsoft |
| Phrase secrète | stockée dans un gestionnaire de mots de passe | perds-la et les données sont perdues pour de bon |

**À propos du téléchargement :** dans le répertoire officiel `https://cdimage.kali.org/current/`, l'image Live est distribuée **uniquement sous forme de `.torrent`** (environ 443 KiB) — c'est l'installeur qui est fourni en `.iso`. Utiliser un client BitTorrent est la voie normale ici, pas un contournement pour un miroir cassé.

**Vérifie le téléchargement** — un ISO corrompu est le mode de défaillance le plus sournois :

```powershell
# Windows PowerShell
certutil -hashfile .\kali-linux-2026.3-live-amd64.iso SHA256
```

```bash
# Linux / macOS
sha256sum kali-linux-2026.3-live-amd64.iso
```

Compare caractère par caractère avec `https://cdimage.kali.org/kali-2026.3/SHA256SUMS`.

---

## 💾 2. Écrire l'ISO (mode DD)

Dans Rufus : **Device** → ta clé · **Boot selection** → l'ISO · quand la boîte de dialogue de mode apparaît, choisis **DD Image mode** · Start.

> ⚠️ **Ne touche jamais au curseur « Persistent partition size ».**
> Rufus implémente la persistance en modifiant les distributions basées sur casper (Ubuntu, Mint). Kali utilise live-boot + isohybrid, donc le curseur réécrit la disposition des partitions, GRUB ne trouve plus le noyau, et tu te retrouves sur une invite `grub>` nue. Refais l'écriture depuis zéro.

Quand c'est terminé, n'ouvre **surtout pas** la clé avec un outil de partitionnement Windows (Gestion des disques, DiskGenius, AOMEI…).

---

## 🖥️ 3. Premier démarrage et identification de la clé

Démarre sur la clé USB et choisis **`Live system (amd64)`** (l'entrée simple).

> Pourquoi pas l'entrée persistence ? Il n'y a pas encore de partition de persistance, ce serait donc inutile — et les partitions doivent être modifiées depuis une **session non persistante**, sinon l'overlay utilise activement le disque que tu es en train de modifier.

Ouvre un terminal et liste les disques :

```bash
lsblk -o NAME,SIZE,RM,FSTYPE,LABEL,MOUNTPOINTS
```

Identifie la clé en combinant trois signaux :

- `RM=1` (amovible)
- l'une de ses partitions est montée sur `/run/live/medium` ou `/lib/live/mount/medium` (c'est l'ISO en cours d'exécution)
- la capacité correspond à la clé que tu as dans la main

Vérifie le mode d'écriture tant que tu y es :

```bash
lsblk -f
```

- ✅ `sda1` est `iso9660` et `sda2` est `vfat` (~4 Mo) → **mode DD, correct**
- ❌ une seule grosse partition `vfat` → écrit en mode ISO, retourne à l'étape 2

**Détection automatique** (recommandée — une faute de frappe ici efface ton portable) :

```bash
medium=$(findmnt -no SOURCE /run/live/medium 2>/dev/null || findmnt -no SOURCE /lib/live/mount/medium)
usb=/dev/$(lsblk -no PKNAME "$medium")
echo "USB = $usb"
lsblk "$usb"
```

La sortie ressemble à ceci :

```
NAME   MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
sda      8:0    1  57.3G  0 disk
├─sda1   8:1    1   4.6G  0 part /run/live/medium
└─sda2   8:2    1     4M  0 part
```

Fixe-le dans une variable — toutes les commandes suivantes utilisent `$usb`, donc une faute de frappe ne peut pas toucher le mauvais disque :

```bash
usb=/dev/sda     # utilise ce que tu as vu ci-dessus
```

---

## 🧩 4. Créer la troisième partition

Inspecte la table actuelle :

```bash
sudo fdisk -l $usb
```

Attends-toi à `Disklabel type: dos` et deux partitions : environ 4,x Go (ISO) et quelques Mo (EFI).

Lance l'éditeur :

```bash
sudo fdisk $usb
```

Saisis ces touches dans l'ordre :

| Touche | Signification |
|---|---|
| `p` ↵ | affiche la table, confirme qu'il n'y a que sda1 + sda2 |
| `n` ↵ | nouvelle partition |
| `p` ↵ | primaire |
| `3` ↵ | numéro de partition (Entrée accepte la valeur par défaut 3) |
| ↵ | premier secteur : par défaut, juste après sda2 |
| ↵ | dernier secteur : par défaut, jusqu'à la fin du disque |
| `w` ↵ | écris et quitte |

> ⚠️ Pourquoi `fdisk` et pas GParted ?
> Un ISO hybride écrit en DD est un « disque dans un disque » : tout le périphérique est vu comme un seul gros bloc ISO9660, donc GParted signale toute la clé comme une unique partition et n'affiche aucun espace libre. Ce n'est pas un bug, c'est la nature de l'image. `fdisk` modifie directement la table de partitions sous-jacente et fonctionne très bien.

**Maintenant, l'étape où la plupart des gens se bloquent.**

Après `w`, tu verras probablement :

```
The kernel still uses the old table.
The new table will be used at the next reboot.
Re-reading the partition table failed.: Device or resource busy
```

La raison est simple : **tu as démarré depuis cette clé même, donc `sda1` est toujours monté** et le noyau refuse de relire la table. Ensuite :

```bash
lsblk $usb
```

- **`sda3` est là** → continue à l'étape 5
- **il est absent** → ne te bats pas avec `partprobe` (il échoue avec le même « busy »), **redémarre simplement sur `Live system (amd64)`**. La table est déjà sur le disque ; le noyau la prend en compte au démarrage. Redétecte le disque (étape 3), confirme que `sda3` existe, puis continue.

---

## 🔐 5. Chiffrer avec LUKS

Vérification de sécurité finale — affiche le périphérique et regarde-le avant d'appuyer sur Entrée :

```bash
[ -b ${usb}3 ] && lsblk -o NAME,SIZE,TYPE ${usb}3
```

Ensuite :

```bash
sudo cryptsetup --verbose --verify-passphrase luksFormat ${usb}3
```

Interaction :

1. `WARNING! This will overwrite data on /dev/sda3 irrevocably.` → tape **`YES`** (en majuscules ; `yes` est refusé)
2. `Enter passphrase for /dev/sda3:` → ta phrase secrète
3. `Verify passphrase:` → encore une fois

**Comment choisir la phrase secrète (c'est là que les gens échouent) :**

- Le déverrouillage se fait dans l'initramfs, où la disposition du clavier est **US English**. Les caractères qui changent de position entre les dispositions (`ä`, `@` sur un clavier allemand, …) ne saisiront pas ce que tu crois saisir
- Recommandé : **12 caractères ou plus, uniquement des minuscules et des chiffres**, ou une longue phrase secrète que tu peux réellement réciter
- `--verify-passphrase` demande simplement la saisie deux fois pour protéger contre les fautes de frappe

> C'est la phrase secrète que tu tapes à chaque démarrage. **Perds-la et les données sont irrécupérables.** Garde une copie dans ton gestionnaire de mots de passe.

---

## 📦 6. ext4 + l'étiquette de persistance

Déverrouille le conteneur :

```bash
sudo cryptsetup luksOpen ${usb}3 my_usb
```

Saisis la phrase secrète. En cas de succès, `/dev/mapper/my_usb` est le périphérique *déchiffré* — **tous les formatages et écritures à partir de maintenant visent celui-ci, plus jamais `${usb}3`**.

```bash
sudo mkfs.ext4 -L persistence /dev/mapper/my_usb
sudo e2label /dev/mapper/my_usb
```

`e2label` doit afficher :

```
persistence
```

> Pourquoi l'étiquette (label) doit-elle être `persistence` ? live-boot trouve la partition de persistance par `LABEL=persistence`. `persist`, `Persistence` ou `persistence ` (espace final) sont tous silencieusement ignorés.

---

## 📝 7. Écrire persistence.conf

```bash
sudo mkdir -pv /mnt/my_usb
sudo mount -v /dev/mapper/my_usb /mnt/my_usb
echo "/ union" | sudo tee /mnt/my_usb/persistence.conf
sync
```

`/ union` signifie : superpose tout le système de fichiers racine `/` sur cette partition, pour que toutes les écritures atterrissent ici.

> ⚠️ Écris-le avec `echo` sous Linux. Si tu crées le fichier dans le Bloc-notes Windows, il obtient des fins de ligne CRLF, live-boot interprète `/ union\r`, le chemin n'existe pas, et la persistance ne fait silencieusement rien. C'est l'échec le plus courant, de loin.

---

## ✅ 8. Vérifier

Les deux vérifications doivent réussir :

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

Le fichier doit se terminer par `0a` (LF). Si tu vois `0d0a`, c'est du CRLF — supprime-le et réécris-le.

Encore une, facile à lire d'un coup d'œil :

```bash
stat -c '%s bytes' /mnt/my_usb/persistence.conf   # doit valoir 8
```

---

## 🔒 9. Désmonter et fermer

```bash
sudo umount -v /mnt/my_usb
sudo cryptsetup luksClose my_usb
sudo reboot
```

L'ordre compte : **démonte d'abord le système de fichiers, ferme ensuite le mapping** — sinon tu obtiens `Device or resource busy`.

---

## 🚀 10. Démarrer et vérifier

Choisis **`Live system (amd64) encrypted persistence`** dans le menu d'amorçage.

Tu devrais voir une invite du genre :

```
Enter passphrase for /dev/sda3:
```

Tape la phrase secrète, arrive au bureau, puis vérifie deux choses.

**① Les données sont vraiment chiffrées** (une clé volée est inutile) :

```bash
lsblk -f
```

`sda3` doit afficher `crypto_LUKS`, avec le mapping `my_usb` en dessous portant `ext4` et l'étiquette `persistence`.

**② La persistance fonctionne vraiment :**

```bash
echo hello > ~/enc-test
sudo reboot
```

Démarre à nouveau sur la même entrée, déverrouille, puis :

```bash
cat ~/enc-test     # affiche hello → terminé
```

---

## 🛡️ 11. Durcissement

### 11.1 Sauvegarder l'en-tête LUKS

Un en-tête endommagé rend toute la partition illisible, et il vit dans les premiers mégaoctets, là où une seule écriture égarée peut le détruire.

```bash
sudo cryptsetup luksHeaderBackup ${usb}3 --header-backup-file ~/luks-header-backup.img
```

Stocke ce `.img` sur une autre clé ou dans un stockage cloud. **Il protège contre la corruption de l'en-tête, pas contre une phrase secrète oubliée.**

### 11.2 Définir un mot de passe d'auto-destruction

Installe d'abord le paquet qui fournit la fonctionnalité (c'est un correctif Debian, la commande n'existe pas par défaut) :

```bash
sudo apt install -y cryptsetup-nuke-password
sudo cryptsetup luksAddNuke ${usb}3
```

Définis un mot de passe d'urgence. Si on te force un jour à déverrouiller la clé, le saisir efface l'en-tête LUKS et personne ne peut déchiffrer les données.

### 11.3 Ajouter une phrase secrète de secours

```bash
sudo cryptsetup luksAddKey ${usb}3
```

### 11.4 Inspecter les emplacements de clés

```bash
sudo cryptsetup luksDump ${usb}3
```

Chaque emplacement est listé — phrase secrète principale, clé de secours, mot de passe d'auto-destruction.

---

## 🩺 12. Dépannage

| Symptôme | Cause | Correctif |
|---|---|---|
| Aucune entrée `encrypted persistence` dans le menu d'amorçage | imbrication du menu différente sous UEFI, ou mauvais ISO | regarde dans le sous-menu `Live system` ; confirme que tu as démarré sur la clé |
| `/dev/sda3` absent après `w` | le noyau ne peut pas relire une table de partitions occupée | redémarre en Live (étape 4) |
| `partprobe` signale « busy » | le `sda1` de la clé est monté parce que tu as démarré dessus | ignore, redémarre |
| Phrase secrète rejetée encore et encore | l'initramfs utilise une disposition US ; la phrase secrète contient des caractères dépendants de la disposition | réenregistre la clé avec des lettres ASCII simples et des chiffres |
| Les modifications disparaissent après redémarrage | tu as démarré sur `Live system` simple ; ou mauvaise étiquette ; ou config CRLF | refais les deux vérifications de l'étape 8 |
| `blkid` n'affiche aucune étiquette `persistence` | étiquette incorrecte, ou tu as formaté en dehors du mapping | `e2label /dev/mapper/my_usb persistence` |
| `fdisk` avertit `GPT PMBR size mismatch` | la clé porte une table GPT résiduelle (l'image Kali elle-même est une table DOS, donc cela ne devrait pas arriver) | `sudo sgdisk -e $usb` pour déplacer le GPT de sauvegarde, puis `fdisk` à nouveau |
| `luksFormat` dit « Device is in use » | la partition a été montée automatiquement | `sudo umount ${usb}3`, réessaie |
| Le démarrage reste bloqué sur `cryptsetup` un moment | normal — le déverrouillage dans un environnement Live est lent | attends 10–30 s |

---

## ⚡ 13. Aide-mémoire

Une fois que `${usb}3` existe, toute la séquence, du chiffrement au nettoyage, est :

```bash
usb=/dev/sda                      # redétecte à chaque démarrage

# chiffrer
sudo cryptsetup --verbose --verify-passphrase luksFormat ${usb}3

# ouvrir, formater, étiqueter
sudo cryptsetup luksOpen ${usb}3 my_usb
sudo mkfs.ext4 -L persistence /dev/mapper/my_usb

# monter et configurer
sudo mkdir -p /mnt/my_usb
sudo mount /dev/mapper/my_usb /mnt/my_usb
echo "/ union" | sudo tee /mnt/my_usb/persistence.conf
sync

# vérifier
sudo blkid /dev/mapper/my_usb          # LABEL="persistence" TYPE="ext4"
xxd /mnt/my_usb/persistence.conf       # se termine par 0a

# nettoyer
sudo umount /mnt/my_usb
sudo cryptsetup luksClose my_usb
sudo reboot
```

---

## ❓ FAQ

**Je ne veux pas de chiffrement — persistance simple uniquement ?**
Saute `luksFormat` et `luksOpen` aux étapes 5–6, lance directement `sudo mkfs.ext4 -L persistence ${usb}3`, puis continue à partir de l'étape 7. Démarre sur l'entrée `persistence`.

**Une nouvelle version de Kali est sortie — dois-je tout refaire ?**
Oui. Réécrire l'ISO en DD réinitialise la table de partitions et la partition de persistance disparaît avec elle. Copie d'abord ton `~` hors de la clé.

**La partition de persistance est pleine — puis-je l'agrandir ?**
Étends-la jusqu'à l'espace libre en fin de clé, puis :

```bash
sudo cryptsetup luksOpen ${usb}3 my_usb
sudo cryptsetup resize my_usb
sudo resize2fs /dev/mapper/my_usb
```

**J'ai oublié la phrase secrète LUKS.**
Il n'y a pas de récupération. Pas de raccourci par force brute, et une sauvegarde d'en-tête n'aide pas (elle ne protège que contre la corruption). C'est pourquoi la phrase secrète appartient à un gestionnaire de mots de passe.

**Puis-je « installer » le système pour que `apt upgrade` fonctionne normalement ?**
C'est une autre voie : installe Kali sur la clé avec l'ISO d'installation, en la traitant comme un disque normal. Plus stable, mais chaque mise à jour du noyau demande de s'occuper de GRUB et cela use la clé plus vite. L'approche Live hybride est meilleure pour une boîte à outils portable.

**La persistance va-t-elle user ma clé USB ?**
Elle ajoute des écritures, mais un usage normal ne pose pas de problème. Évite les clés sans marque, ne remplis pas complètement la clé (un overlay plein se comporte bizarrement) et vérifie `df -h` de temps en temps.

**Puis-je modifier la partition de persistance en étant démarré avec la persistance ?**
Non. L'overlay l'utilise ; repartitionner ou reformater depuis cette session corrompra très probablement tes données. Fais-le toujours depuis une session `Live system` simple.

---

## 🔗 Références

- Docs Kali : persistance USB - <https://www.kali.org/docs/usb/usb-persistence/>
- Docs Kali : persistance chiffrée - <https://www.kali.org/docs/usb/usb-persistence-encryption/>
- Persistance Ventoy (l'autre voie : un fichier `.dat`, pas d'édition de partitions) - <https://www.ventoy.net/en/plugin_persistence.html>

## 📄 Licence

Publié sous la [licence MIT](LICENSE). Copyright (c) 2026 Ton Nom.

Les pull requests et les rapports de problèmes sur les pièges que tu as rencontrés sont les bienvenus.
