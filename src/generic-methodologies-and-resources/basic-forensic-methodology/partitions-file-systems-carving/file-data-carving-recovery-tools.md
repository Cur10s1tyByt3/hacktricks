# File/Data Carving & Recovery Tools

{{#include ../../../banners/hacktricks-training.md}}

## Outils de Carving & de Récupération

Plus d'outils sur [https://github.com/Claudio-C/awesome-datarecovery](https://github.com/Claudio-C/awesome-datarecovery)

### Autopsy

L'outil le plus couramment utilisé en criminalistique pour extraire des fichiers d'images est [**Autopsy**](https://www.autopsy.com/download/). Téléchargez-le, installez-le et faites-lui ingérer le fichier pour trouver des fichiers "cachés". Notez qu'Autopsy est conçu pour prendre en charge les images disque et d'autres types d'images, mais pas les fichiers simples.

> **Mise à jour 2024-2025** – La version **4.21** (publiée en février 2025) a ajouté un **module de carving reconstruit basé sur SleuthKit v4.13** qui est nettement plus rapide lors du traitement d'images multi-téraoctets et prend en charge l'extraction parallèle sur des systèmes multi-cœurs.¹ Un petit wrapper CLI (`autopsycli ingest <case> <image>`) a également été introduit, rendant possible le scripting du carving dans des environnements CI/CD ou de laboratoire à grande échelle.
```bash
# Create a case and ingest an evidence image from the CLI (Autopsy ≥4.21)
autopsycli case --create MyCase --base /cases
# ingest with the default ingest profile (includes data-carve module)
autopsycli ingest MyCase /evidence/disk01.E01 --threads 8
```
### Binwalk <a href="#binwalk" id="binwalk"></a>

**Binwalk** est un outil pour analyser des fichiers binaires afin de trouver du contenu intégré. Il est installable via `apt` et sa source est sur [GitHub](https://github.com/ReFirmLabs/binwalk).

**Commandes utiles**:
```bash
sudo apt install binwalk         # Installation
binwalk firmware.bin             # Display embedded data
binwalk -e firmware.bin          # Extract recognised objects (safe-default)
binwalk --dd " .* " firmware.bin  # Extract *everything* (use with care)
```
⚠️  **Note de sécurité** – Les versions **≤2.3.3** sont affectées par une vulnérabilité de **Path Traversal** (CVE-2022-4510). Mettez à jour (ou isolez avec un conteneur/UID non privilégié) avant de carver des échantillons non fiables.

### Foremost

Un autre outil courant pour trouver des fichiers cachés est **foremost**. Vous pouvez trouver le fichier de configuration de foremost dans `/etc/foremost.conf`. Si vous souhaitez simplement rechercher des fichiers spécifiques, décommentez-les. Si vous ne décommentez rien, foremost recherchera ses types de fichiers configurés par défaut.
```bash
sudo apt-get install foremost
foremost -v -i file.img -o output
# Discovered files will appear inside the folder "output"
```
### **Scalpel**

**Scalpel** est un autre outil qui peut être utilisé pour trouver et extraire **des fichiers intégrés dans un fichier**. Dans ce cas, vous devrez décommenter dans le fichier de configuration (_/etc/scalpel/scalpel.conf_) les types de fichiers que vous souhaitez qu'il extraye.
```bash
sudo apt-get install scalpel
scalpel file.img -o output
```
### Bulk Extractor 2.x

Cet outil est inclus dans kali mais vous pouvez le trouver ici : <https://github.com/simsong/bulk_extractor>

Bulk Extractor peut analyser une image de preuve et extraire **des fragments pcap**, **des artefacts réseau (URLs, domaines, IPs, MACs, e-mails)** et de nombreux autres objets **en parallèle en utilisant plusieurs scanners**.
```bash
# Build from source – v2.1.1 (April 2024) requires cmake ≥3.16
git clone https://github.com/simsong/bulk_extractor.git && cd bulk_extractor
mkdir build && cd build && cmake .. && make -j$(nproc) && sudo make install

# Run every scanner, carve JPEGs aggressively and generate a bodyfile
bulk_extractor -o out_folder -S jpeg_carve_mode=2 -S write_bodyfile=y /evidence/disk.img
```
Des scripts de post-traitement utiles (`bulk_diff`, `bulk_extractor_reader.py`) peuvent dédupliquer des artefacts entre deux images ou convertir les résultats en JSON pour l'ingestion SIEM.

### PhotoRec

Vous pouvez le trouver sur <https://www.cgsecurity.org/wiki/TestDisk_Download>

Il est disponible en versions GUI et CLI. Vous pouvez sélectionner les **types de fichiers** que vous souhaitez que PhotoRec recherche.

![](<../../../images/image (242).png>)

### ddrescue + ddrescueview (imagerie de disques défaillants)

Lorsqu'un disque physique est instable, il est recommandé de **l'imager d'abord** et de n'exécuter des outils de carving que sur l'image. `ddrescue` (projet GNU) se concentre sur la copie fiable de disques défectueux tout en tenant un journal des secteurs illisibles.
```bash
sudo apt install gddrescue ddrescueview   # On Debian-based systems
# First pass – try to get as much data as possible without retries
sudo ddrescue -f -n /dev/sdX suspect.img suspect.log
# Second pass – aggressive, 3 retries on the remaining bad areas
sudo ddrescue -d -r3 /dev/sdX suspect.img suspect.log

# Visualise the status map (green=good, red=bad)
ddrescueview suspect.log
```
Version **1.28** (décembre 2024) a introduit **`--cluster-size`** qui peut accélérer l'imagerie des SSD haute capacité où les tailles de secteur traditionnelles ne s'alignent plus avec les blocs flash.

### Extundelete / Ext4magic (Récupération EXT 3/4)

Si le système de fichiers source est basé sur Linux EXT, vous pouvez être en mesure de récupérer des fichiers récemment supprimés **sans carving complet**. Les deux outils fonctionnent directement sur une image en lecture seule :
```bash
# Attempt journal-based undelete (metadata must still be present)
extundelete disk.img --restore-all

# Fallback to full directory scan; supports extents and inline data
ext4magic disk.img -M -f '*.jpg' -d ./recovered
```
> 🛈 Si le système de fichiers a été monté après la suppression, les blocs de données peuvent déjà avoir été réutilisés – dans ce cas, un carving approprié (Foremost/Scalpel) est toujours nécessaire.

### binvis

Vérifiez le [code](https://code.google.com/archive/p/binvis/) et la [page web de l'outil](https://binvis.io/#/).

#### Fonctionnalités de BinVis

- Visualiseur de **structure** visuel et actif
- Plusieurs graphiques pour différents points de focus
- Focalisation sur des portions d'un échantillon
- **Voir les chaînes et les ressources**, dans des exécutables PE ou ELF par exemple
- Obtenir des **modèles** pour la cryptanalyse sur des fichiers
- **Repérer** des algorithmes de packer ou d'encodeur
- **Identifier** la stéganographie par des motifs
- **Différenciation** binaire visuelle

BinVis est un excellent **point de départ pour se familiariser avec une cible inconnue** dans un scénario de black-boxing.

## Outils de Carving de Données Spécifiques

### FindAES

Recherche des clés AES en cherchant leurs plannings de clés. Capable de trouver des clés de 128, 192 et 256 bits, telles que celles utilisées par TrueCrypt et BitLocker.

Téléchargez [ici](https://sourceforge.net/projects/findaes/).

### YARA-X (triage des artefacts sculptés)

[YARA-X](https://github.com/VirusTotal/yara-x) est une réécriture en Rust de YARA publiée en 2024. Elle est **10 à 30 fois plus rapide** que YARA classique et peut être utilisée pour classer des milliers d'objets sculptés très rapidement :
```bash
# Scan every carved object produced by bulk_extractor
yarax -r rules/index.yar out_folder/ --threads 8 --print-meta
```
L'accélération rend réaliste le **tagging automatique** de tous les fichiers extraits lors d'enquêtes à grande échelle.

## Outils complémentaires

Vous pouvez utiliser [**viu** ](https://github.com/atanunq/viu) pour voir des images depuis le terminal.  \
Vous pouvez utiliser l'outil en ligne de commande linux **pdftotext** pour transformer un pdf en texte et le lire.

## Références

1. Notes de version d'Autopsy 4.21 – <https://github.com/sleuthkit/autopsy/releases/tag/autopsy-4.21>
{{#include ../../../banners/hacktricks-training.md}}
