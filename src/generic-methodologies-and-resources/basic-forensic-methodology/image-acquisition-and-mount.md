# Acquisition d'Image & Montage

{{#include ../../banners/hacktricks-training.md}}


## Acquisition

> Acquérez toujours en **lecture seule** et **hachez pendant que vous copiez**. Gardez l'appareil original **bloqué en écriture** et travaillez uniquement sur des copies vérifiées.

### DD
```bash
# Generate a raw, bit-by-bit image (no on-the-fly hashing)
dd if=/dev/sdb of=disk.img bs=4M status=progress conv=noerror,sync
# Verify integrity afterwards
sha256sum disk.img > disk.img.sha256
```
### dc3dd / dcfldd

`dc3dd` est le fork activement maintenu de dcfldd (DoD Computer Forensics Lab dd).
```bash
# Create an image and calculate multiple hashes at acquisition time
sudo dc3dd if=/dev/sdc of=/forensics/pc.img hash=sha256,sha1 hashlog=/forensics/pc.hashes log=/forensics/pc.log bs=1M
```
### Guymager
Imager graphique multithread qui prend en charge les sorties **raw (dd)**, **EWF (E01/EWFX)** et **AFF4** avec vérification parallèle. Disponible dans la plupart des dépôts Linux (`apt install guymager`).
```bash
# Start in GUI mode
sudo guymager
# Or acquire from CLI (since v0.9.5)
sudo guymager --simulate --input /dev/sdb --format EWF --hash sha256 --output /evidence/drive.e01
```
### AFF4 (Advanced Forensics Format 4)

AFF4 est le format d'imagerie moderne de Google conçu pour des preuves *très* volumineuses (sparse, resumable, cloud-native).
```bash
# Acquire to AFF4 using the reference tool
pipx install aff4imager
sudo aff4imager acquire /dev/nvme0n1 /evidence/nvme.aff4 --hash sha256

# Velociraptor can also acquire AFF4 images remotely
velociraptor --config server.yaml frontend collect --artifact Windows.Disk.Acquire --args device="\\.\\PhysicalDrive0" format=AFF4
```
### FTK Imager (Windows & Linux)

Vous pouvez [télécharger FTK Imager](https://accessdata.com/product-download) et créer des images **brutes, E01 ou AFF4** :
```bash
ftkimager /dev/sdb evidence --e01 --case-number 1 --evidence-number 1 \
--description 'Laptop seizure 2025-07-22' --examiner 'AnalystName' --compress 6
```
### Outils EWF (libewf)
```bash
sudo ewfacquire /dev/sdb -u evidence -c 1 -d "Seizure 2025-07-22" -e 1 -X examiner --format encase6 --compression best
```
### Imaging Cloud Disks

*AWS* – créer un **instantané judiciaire** sans éteindre l'instance :
```bash
aws ec2 create-snapshot --volume-id vol-01234567 --description "IR-case-1234 web-server 2025-07-22"
# Copy the snapshot to S3 and download with aws cli / aws snowball
```
*Azure* – utilisez `az snapshot create` et exportez vers une URL SAS. Voir la page HackTricks {{#ref}}
../../cloud/azure/azure-forensics.md
{{#endref}}


## Monter

### Choisir la bonne approche

1. Montez le **disque entier** lorsque vous souhaitez la table de partition originale (MBR/GPT).
2. Montez un **fichier de partition unique** lorsque vous n'avez besoin que d'un volume.
3. Montez toujours en **lecture seule** (`-o ro,norecovery`) et travaillez sur des **copies**.

### Images brutes (dd, extraites d'AFF4)
```bash
# Identify partitions
fdisk -l disk.img

# Attach the image to a network block device (does not modify the file)
sudo modprobe nbd max_part=16
sudo qemu-nbd --connect=/dev/nbd0 --read-only disk.img

# Inspect partitions
lsblk /dev/nbd0 -o NAME,SIZE,TYPE,FSTYPE,LABEL,UUID

# Mount a partition (e.g. /dev/nbd0p2)
sudo mount -o ro,uid=$(id -u) /dev/nbd0p2 /mnt
```
Détachez lorsque vous avez terminé :
```bash
sudo umount /mnt && sudo qemu-nbd --disconnect /dev/nbd0
```
### EWF (E01/EWFX)
```bash
# 1. Mount the EWF container
mkdir /mnt/ewf
ewfmount evidence.E01 /mnt/ewf

# 2. Attach the exposed raw file via qemu-nbd (safer than loop)
sudo qemu-nbd --connect=/dev/nbd1 --read-only /mnt/ewf/ewf1

# 3. Mount the desired partition
sudo mount -o ro,norecovery /dev/nbd1p1 /mnt/evidence
```
Alternativement, convertissez à la volée avec **xmount** :
```bash
xmount --in ewf evidence.E01 --out raw /tmp/raw_mount
mount -o ro /tmp/raw_mount/image.dd /mnt
```
### LVM / BitLocker / VeraCrypt volumes

Après avoir attaché le périphérique de bloc (loop ou nbd) :
```bash
# LVM
sudo vgchange -ay               # activate logical volumes
sudo lvscan | grep "/dev/nbd0"

# BitLocker (dislocker)
sudo dislocker -V /dev/nbd0p3 -u -- /mnt/bitlocker
sudo mount -o ro /mnt/bitlocker/dislocker-file /mnt/evidence
```
### kpartx helpers

`kpartx` mappe automatiquement les partitions d'une image vers `/dev/mapper/` :
```bash
sudo kpartx -av disk.img  # creates /dev/mapper/loop0p1, loop0p2 …
mount -o ro /dev/mapper/loop0p2 /mnt
```
### Erreurs de montage courantes & solutions

| Erreur | Cause typique | Solution |
|-------|---------------|-----|
| `cannot mount /dev/loop0 read-only` | FS journalisé (ext4) non démonté proprement | utiliser `-o ro,norecovery` |
| `bad superblock …` | Mauvais décalage ou FS endommagé | calculer le décalage (`sector*size`) ou exécuter `fsck -n` sur une copie |
| `mount: unknown filesystem type 'LVM2_member'` | Conteneur LVM | activer le groupe de volumes avec `vgchange -ay` |

### Nettoyage

N'oubliez pas de **umount** et **déconnecter** les dispositifs loop/nbd pour éviter de laisser des mappages pendants qui peuvent corrompre le travail ultérieur :
```bash
umount -Rl /mnt/evidence
kpartx -dv /dev/loop0  # or qemu-nbd --disconnect /dev/nbd0
```
## Références

- Annonce et spécification de l'outil d'imagerie AFF4 : https://github.com/aff4/aff4
- Page de manuel qemu-nbd (montage d'images disque en toute sécurité) : https://manpages.debian.org/qemu-system-common/qemu-nbd.1.en.html

{{#include ../../banners/hacktricks-training.md}}
