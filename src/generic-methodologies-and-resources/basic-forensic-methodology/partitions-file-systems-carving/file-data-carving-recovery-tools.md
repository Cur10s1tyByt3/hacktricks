# File/Data Carving & Recovery Tools

{{#include ../../../banners/hacktricks-training.md}}

## Carving & Recovery tools

More tools in [https://github.com/Claudio-C/awesome-datarecovery](https://github.com/Claudio-C/awesome-datarecovery)

### Autopsy

La herramienta más común utilizada en forenses para extraer archivos de imágenes es [**Autopsy**](https://www.autopsy.com/download/). Descárgala, instálala y haz que ingiera el archivo para encontrar archivos "ocultos". Ten en cuenta que Autopsy está diseñado para soportar imágenes de disco y otros tipos de imágenes, pero no archivos simples.

> **2024-2025 actualización** – La versión **4.21** (lanzada en febrero de 2025) agregó un **módulo de carving reconstruido basado en SleuthKit v4.13** que es notablemente más rápido al tratar con imágenes de múltiples terabytes y soporta extracción paralela en sistemas de múltiples núcleos.¹ También se introdujo un pequeño envoltorio CLI (`autopsycli ingest <case> <image>`), lo que hace posible scriptar carving dentro de entornos CI/CD o de laboratorio a gran escala.
```bash
# Create a case and ingest an evidence image from the CLI (Autopsy ≥4.21)
autopsycli case --create MyCase --base /cases
# ingest with the default ingest profile (includes data-carve module)
autopsycli ingest MyCase /evidence/disk01.E01 --threads 8
```
### Binwalk <a href="#binwalk" id="binwalk"></a>

**Binwalk** es una herramienta para analizar archivos binarios y encontrar contenido incrustado. Se puede instalar a través de `apt` y su código fuente está en [GitHub](https://github.com/ReFirmLabs/binwalk).

**Comandos útiles**:
```bash
sudo apt install binwalk         # Installation
binwalk firmware.bin             # Display embedded data
binwalk -e firmware.bin          # Extract recognised objects (safe-default)
binwalk --dd " .* " firmware.bin  # Extract *everything* (use with care)
```
⚠️  **Nota de seguridad** – Las versiones **≤2.3.3** están afectadas por una vulnerabilidad de **Path Traversal** (CVE-2022-4510). Actualiza (o aísla con un contenedor/UID no privilegiado) antes de extraer muestras no confiables.

### Foremost

Otra herramienta común para encontrar archivos ocultos es **foremost**. Puedes encontrar el archivo de configuración de foremost en `/etc/foremost.conf`. Si solo deseas buscar algunos archivos específicos, descomenta esos. Si no descomentas nada, foremost buscará sus tipos de archivo configurados por defecto.
```bash
sudo apt-get install foremost
foremost -v -i file.img -o output
# Discovered files will appear inside the folder "output"
```
### **Scalpel**

**Scalpel** es otra herramienta que se puede utilizar para encontrar y extraer **archivos incrustados en un archivo**. En este caso, necesitarás descomentar del archivo de configuración (_/etc/scalpel/scalpel.conf_) los tipos de archivo que deseas que extraiga.
```bash
sudo apt-get install scalpel
scalpel file.img -o output
```
### Bulk Extractor 2.x

Esta herramienta viene incluida en Kali, pero puedes encontrarla aquí: <https://github.com/simsong/bulk_extractor>

Bulk Extractor puede escanear una imagen de evidencia y extraer **fragmentos de pcap**, **artefactos de red (URLs, dominios, IPs, MACs, correos electrónicos)** y muchos otros objetos **en paralelo utilizando múltiples escáneres**.
```bash
# Build from source – v2.1.1 (April 2024) requires cmake ≥3.16
git clone https://github.com/simsong/bulk_extractor.git && cd bulk_extractor
mkdir build && cd build && cmake .. && make -j$(nproc) && sudo make install

# Run every scanner, carve JPEGs aggressively and generate a bodyfile
bulk_extractor -o out_folder -S jpeg_carve_mode=2 -S write_bodyfile=y /evidence/disk.img
```
Scripts de post-procesamiento útiles (`bulk_diff`, `bulk_extractor_reader.py`) pueden desduplicar artefactos entre dos imágenes o convertir resultados a JSON para la ingestión en SIEM.

### PhotoRec

Puedes encontrarlo en <https://www.cgsecurity.org/wiki/TestDisk_Download>

Viene con versiones GUI y CLI. Puedes seleccionar los **tipos de archivo** que deseas que PhotoRec busque.

![](<../../../images/image (242).png>)

### ddrescue + ddrescueview (imagen de unidades fallidas)

Cuando un disco físico es inestable, es una buena práctica **hacer una imagen primero** y solo ejecutar herramientas de carving contra la imagen. `ddrescue` (proyecto GNU) se centra en copiar de manera confiable discos dañados mientras mantiene un registro de los sectores ilegibles.
```bash
sudo apt install gddrescue ddrescueview   # On Debian-based systems
# First pass – try to get as much data as possible without retries
sudo ddrescue -f -n /dev/sdX suspect.img suspect.log
# Second pass – aggressive, 3 retries on the remaining bad areas
sudo ddrescue -d -r3 /dev/sdX suspect.img suspect.log

# Visualise the status map (green=good, red=bad)
ddrescueview suspect.log
```
Versión **1.28** (diciembre de 2024) introdujo **`--cluster-size`** que puede acelerar la creación de imágenes de SSDs de alta capacidad donde los tamaños de sector tradicionales ya no se alinean con los bloques de flash.

### Extundelete / Ext4magic (EXT 3/4 undelete)

Si el sistema de archivos de origen es basado en Linux EXT, es posible que puedas recuperar archivos eliminados recientemente **sin carving completo**. Ambas herramientas funcionan directamente en una imagen de solo lectura:
```bash
# Attempt journal-based undelete (metadata must still be present)
extundelete disk.img --restore-all

# Fallback to full directory scan; supports extents and inline data
ext4magic disk.img -M -f '*.jpg' -d ./recovered
```
> 🛈 Si el sistema de archivos fue montado después de la eliminación, los bloques de datos pueden haber sido reutilizados; en ese caso, se requiere un carving adecuado (Foremost/Scalpel).

### binvis

Revisa el [código](https://code.google.com/archive/p/binvis/) y la [herramienta de página web](https://binvis.io/#/).

#### Características de BinVis

- Visual y activo **visor de estructuras**
- Múltiples gráficos para diferentes puntos de enfoque
- Enfocándose en porciones de una muestra
- **Viendo cadenas y recursos**, en ejecutables PE o ELF, por ejemplo
- Obteniendo **patrones** para criptoanálisis en archivos
- **Detectando** algoritmos de empaquetado o codificación
- **Identificar** esteganografía por patrones
- **Visual** de diferencias binarias

BinVis es un gran **punto de partida para familiarizarse con un objetivo desconocido** en un escenario de caja negra.

## Herramientas Específicas de Carving de Datos

### FindAES

Busca claves AES buscando sus horarios de clave. Capaz de encontrar claves de 128, 192 y 256 bits, como las utilizadas por TrueCrypt y BitLocker.

Descargar [aquí](https://sourceforge.net/projects/findaes/).

### YARA-X (triaging artefactos tallados)

[YARA-X](https://github.com/VirusTotal/yara-x) es una reescritura en Rust de YARA lanzada en 2024. Es **10-30× más rápida** que YARA clásica y puede ser utilizada para clasificar miles de objetos tallados muy rápidamente:
```bash
# Scan every carved object produced by bulk_extractor
yarax -r rules/index.yar out_folder/ --threads 8 --print-meta
```
La aceleración hace que sea realista **auto-etiquetar** todos los archivos tallados en investigaciones a gran escala.

## Herramientas complementarias

Puedes usar [**viu** ](https://github.com/atanunq/viu) para ver imágenes desde la terminal.  \
Puedes usar la herramienta de línea de comandos de linux **pdftotext** para transformar un pdf en texto y leerlo.

## Referencias

1. Notas de la versión 4.21 de Autopsy – <https://github.com/sleuthkit/autopsy/releases/tag/autopsy-4.21>
{{#include ../../../banners/hacktricks-training.md}}
