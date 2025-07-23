# Adquisición de Imágenes y Montaje

{{#include ../../banners/hacktricks-training.md}}


## Adquisición

> Siempre adquiere **solo lectura** y **hash mientras copias**. Mantén el dispositivo original **bloqueado para escritura** y trabaja solo en copias verificadas.

### DD
```bash
# Generate a raw, bit-by-bit image (no on-the-fly hashing)
dd if=/dev/sdb of=disk.img bs=4M status=progress conv=noerror,sync
# Verify integrity afterwards
sha256sum disk.img > disk.img.sha256
```
### dc3dd / dcfldd

`dc3dd` es el fork mantenido activamente de dcfldd (DoD Computer Forensics Lab dd).
```bash
# Create an image and calculate multiple hashes at acquisition time
sudo dc3dd if=/dev/sdc of=/forensics/pc.img hash=sha256,sha1 hashlog=/forensics/pc.hashes log=/forensics/pc.log bs=1M
```
### Guymager
Imager gráfico y multihilo que soporta **raw (dd)**, **EWF (E01/EWFX)** y **AFF4** como salida con verificación paralela. Disponible en la mayoría de los repositorios de Linux (`apt install guymager`).
```bash
# Start in GUI mode
sudo guymager
# Or acquire from CLI (since v0.9.5)
sudo guymager --simulate --input /dev/sdb --format EWF --hash sha256 --output /evidence/drive.e01
```
### AFF4 (Formato de Forense Avanzado 4)

AFF4 es el formato de imagen moderno de Google diseñado para *muy* grandes evidencias (dispersas, reanudables, nativas de la nube).
```bash
# Acquire to AFF4 using the reference tool
pipx install aff4imager
sudo aff4imager acquire /dev/nvme0n1 /evidence/nvme.aff4 --hash sha256

# Velociraptor can also acquire AFF4 images remotely
velociraptor --config server.yaml frontend collect --artifact Windows.Disk.Acquire --args device="\\.\\PhysicalDrive0" format=AFF4
```
### FTK Imager (Windows y Linux)

Puedes [descargar FTK Imager](https://accessdata.com/product-download) y crear imágenes **raw, E01 o AFF4**:
```bash
ftkimager /dev/sdb evidence --e01 --case-number 1 --evidence-number 1 \
--description 'Laptop seizure 2025-07-22' --examiner 'AnalystName' --compress 6
```
### Herramientas EWF (libewf)
```bash
sudo ewfacquire /dev/sdb -u evidence -c 1 -d "Seizure 2025-07-22" -e 1 -X examiner --format encase6 --compression best
```
### Creación de Imágenes de Discos en la Nube

*AWS* – crear un **snapshot forense** sin apagar la instancia:
```bash
aws ec2 create-snapshot --volume-id vol-01234567 --description "IR-case-1234 web-server 2025-07-22"
# Copy the snapshot to S3 and download with aws cli / aws snowball
```
*Azure* – use `az snapshot create` y exporte a una URL SAS.  Consulte la página de HackTricks {{#ref}}
../../cloud/azure/azure-forensics.md
{{#endref}}


## Montar

### Elegir el enfoque correcto

1. Monte el **disco completo** cuando necesite la tabla de particiones original (MBR/GPT).
2. Monte un **archivo de partición única** cuando solo necesite un volumen.
3. Siempre monte **solo lectura** (`-o ro,norecovery`) y trabaje en **copias**.

### Imágenes en bruto (dd, AFF4-extracted)
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
Desconectar cuando haya terminado:
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
Alternativamente, convierte sobre la marcha con **xmount**:
```bash
xmount --in ewf evidence.E01 --out raw /tmp/raw_mount
mount -o ro /tmp/raw_mount/image.dd /mnt
```
### LVM / BitLocker / VeraCrypt volumes

Después de adjuntar el dispositivo de bloque (loop o nbd):
```bash
# LVM
sudo vgchange -ay               # activate logical volumes
sudo lvscan | grep "/dev/nbd0"

# BitLocker (dislocker)
sudo dislocker -V /dev/nbd0p3 -u -- /mnt/bitlocker
sudo mount -o ro /mnt/bitlocker/dislocker-file /mnt/evidence
```
### kpartx helpers

`kpartx` mapea particiones de una imagen a `/dev/mapper/` automáticamente:
```bash
sudo kpartx -av disk.img  # creates /dev/mapper/loop0p1, loop0p2 …
mount -o ro /dev/mapper/loop0p2 /mnt
```
### Errores comunes de montaje y soluciones

| Error | Causa típica | Solución |
|-------|--------------|----------|
| `cannot mount /dev/loop0 read-only` | FS con journaling (ext4) no desmontado correctamente | usar `-o ro,norecovery` |
| `bad superblock …` | Desplazamiento incorrecto o FS dañado | calcular desplazamiento (`sector*size`) o ejecutar `fsck -n` en una copia |
| `mount: unknown filesystem type 'LVM2_member'` | Contenedor LVM | activar grupo de volúmenes con `vgchange -ay` |

### Limpieza

Recuerda **umount** y **desconectar** dispositivos loop/nbd para evitar dejar mapeos colgantes que pueden corromper trabajos futuros:
```bash
umount -Rl /mnt/evidence
kpartx -dv /dev/loop0  # or qemu-nbd --disconnect /dev/nbd0
```
## Referencias

- Anuncio y especificación de la herramienta de imagen AFF4: https://github.com/aff4/aff4
- Página del manual de qemu-nbd (montando imágenes de disco de manera segura): https://manpages.debian.org/qemu-system-common/qemu-nbd.1.en.html

{{#include ../../banners/hacktricks-training.md}}
