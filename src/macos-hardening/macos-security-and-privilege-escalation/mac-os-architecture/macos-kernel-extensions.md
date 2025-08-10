# macOS Kernel Extensions & Debugging

{{#include ../../../banners/hacktricks-training.md}}

## Informations de base

Les extensions de noyau (Kexts) sont des **paquets** avec une **extension `.kext`** qui sont **chargés directement dans l'espace noyau de macOS**, fournissant des fonctionnalités supplémentaires au système d'exploitation principal.

### Statut de dépréciation & DriverKit / Extensions système
À partir de **macOS Catalina (10.15)**, Apple a marqué la plupart des KPI hérités comme *dépréciés* et a introduit les **Extensions système & DriverKit** qui fonctionnent dans **l'espace utilisateur**. À partir de **macOS Big Sur (11)**, le système d'exploitation *refusera de charger* des kexts tiers qui dépendent de KPI dépréciés, à moins que la machine ne soit démarrée en mode **Sécurité réduite**. Sur Apple Silicon, l'activation des kexts nécessite également que l'utilisateur :

1. Redémarre en **Récupération** → *Utilitaire de sécurité de démarrage*.
2. Sélectionne **Sécurité réduite** et coche **“Autoriser la gestion des extensions de noyau par les développeurs identifiés”**.
3. Redémarre et approuve le kext depuis **Réglages système → Confidentialité & Sécurité**.

Les pilotes en espace utilisateur écrits avec DriverKit/Extensions système réduisent considérablement la **surface d'attaque** car les plantages ou la corruption de mémoire sont confinés à un processus isolé plutôt qu'à l'espace noyau.

> 📝 À partir de macOS Sequoia (15), Apple a complètement supprimé plusieurs KPI hérités liés au réseau et à l'USB – la seule solution compatible à l'avenir pour les fournisseurs est de migrer vers les Extensions système.

### Exigences

Évidemment, c'est si puissant qu'il est **compliqué de charger une extension de noyau**. Voici les **exigences** qu'une extension de noyau doit respecter pour être chargée :

- Lors de **l'entrée en mode récupération**, les **extensions de noyau doivent être autorisées** à être chargées :

<figure><img src="../../../images/image (327).png" alt=""><figcaption></figcaption></figure>

- L'extension de noyau doit être **signée avec un certificat de signature de code de noyau**, qui ne peut être **accordé que par Apple**. Qui examinera en détail l'entreprise et les raisons pour lesquelles cela est nécessaire.
- L'extension de noyau doit également être **notariée**, Apple pourra la vérifier pour détecter des logiciels malveillants.
- Ensuite, l'utilisateur **root** est celui qui peut **charger l'extension de noyau** et les fichiers à l'intérieur du paquet doivent **appartenir à root**.
- Pendant le processus de téléchargement, le paquet doit être préparé dans un **emplacement protégé non-root** : `/Library/StagedExtensions` (nécessite l'octroi de `com.apple.rootless.storage.KernelExtensionManagement`).
- Enfin, lors de la tentative de chargement, l'utilisateur recevra une [**demande de confirmation**](https://developer.apple.com/library/archive/technotes/tn2459/_index.html) et, si acceptée, l'ordinateur doit être **redémarré** pour le charger.

### Processus de chargement

Dans Catalina, c'était comme ça : Il est intéressant de noter que le processus de **vérification** se produit dans **l'espace utilisateur**. Cependant, seules les applications avec l'octroi **`com.apple.private.security.kext-management`** peuvent **demander au noyau de charger une extension** : `kextcache`, `kextload`, `kextutil`, `kextd`, `syspolicyd`

1. **`kextutil`** cli **démarre** le processus de **vérification** pour charger une extension
- Il communiquera avec **`kextd`** en utilisant un **service Mach**.
2. **`kextd`** vérifiera plusieurs choses, telles que la **signature**
- Il communiquera avec **`syspolicyd`** pour **vérifier** si l'extension peut être **chargée**.
3. **`syspolicyd`** **demande** à l'**utilisateur** si l'extension n'a pas été chargée précédemment.
- **`syspolicyd`** rapportera le résultat à **`kextd`**
4. **`kextd`** pourra enfin **dire au noyau de charger** l'extension

Si **`kextd`** n'est pas disponible, **`kextutil`** peut effectuer les mêmes vérifications.

### Énumération & gestion (kexts chargés)

`kextstat` était l'outil historique mais il est **déprécié** dans les récentes versions de macOS. L'interface moderne est **`kmutil`** :
```bash
# List every extension currently linked in the kernel, sorted by load address
sudo kmutil showloaded --sort

# Show only third-party / auxiliary collections
sudo kmutil showloaded --collection aux

# Unload a specific bundle
sudo kmutil unload -b com.example.mykext
```
La syntaxe plus ancienne est toujours disponible pour référence :
```bash
# (Deprecated) Get loaded kernel extensions
kextstat

# (Deprecated) Get dependencies of the kext number 22
kextstat | grep " 22 " | cut -c2-5,50- | cut -d '(' -f1
```
`kmutil inspect` peut également être utilisé pour **extraire le contenu d'une Kernel Collection (KC)** ou vérifier qu'un kext résout toutes les dépendances de symboles :
```bash
# List fileset entries contained in the boot KC
kmutil inspect -B /System/Library/KernelCollections/BootKernelExtensions.kc --show-fileset-entries

# Check undefined symbols of a 3rd party kext before loading
kmutil libraries -p /Library/Extensions/FancyUSB.kext --undef-symbols
```
## Kernelcache

> [!CAUTION]
> Même si les extensions du noyau sont censées se trouver dans `/System/Library/Extensions/`, si vous allez dans ce dossier, vous **ne trouverez aucun binaire**. Cela est dû au **kernelcache** et pour inverser un `.kext`, vous devez trouver un moyen de l'obtenir.

Le **kernelcache** est une **version précompilée et préliée du noyau XNU**, ainsi que des **drivers** et des **extensions de noyau** essentiels. Il est stocké dans un format **compressé** et est décompressé en mémoire pendant le processus de démarrage. Le kernelcache facilite un **temps de démarrage plus rapide** en ayant une version prête à l'emploi du noyau et des drivers cruciaux disponibles, réduisant le temps et les ressources qui seraient autrement dépensés pour charger et lier dynamiquement ces composants au moment du démarrage.

### Local Kerlnelcache

Dans iOS, il est situé dans **`/System/Library/Caches/com.apple.kernelcaches/kernelcache`** dans macOS, vous pouvez le trouver avec : **`find / -name "kernelcache" 2>/dev/null`** \
Dans mon cas, dans macOS, je l'ai trouvé dans :

- `/System/Volumes/Preboot/1BAEB4B5-180B-4C46-BD53-51152B7D92DA/boot/DAD35E7BC0CDA79634C20BD1BD80678DFB510B2AAD3D25C1228BB34BCD0A711529D3D571C93E29E1D0C1264750FA043F/System/Library/Caches/com.apple.kernelcaches/kernelcache`

#### IMG4

Le format de fichier IMG4 est un format de conteneur utilisé par Apple dans ses appareils iOS et macOS pour **stocker et vérifier en toute sécurité** les composants du firmware (comme le **kernelcache**). Le format IMG4 comprend un en-tête et plusieurs balises qui encapsulent différentes pièces de données, y compris la charge utile réelle (comme un noyau ou un chargeur de démarrage), une signature et un ensemble de propriétés de manifeste. Le format prend en charge la vérification cryptographique, permettant à l'appareil de confirmer l'authenticité et l'intégrité du composant du firmware avant de l'exécuter.

Il est généralement composé des composants suivants :

- **Payload (IM4P)** :
- Souvent compressé (LZFSE4, LZSS, …)
- Optionnellement chiffré
- **Manifest (IM4M)** :
- Contient la signature
- Dictionnaire clé/valeur supplémentaire
- **Restore Info (IM4R)** :
- Également connu sous le nom d'APNonce
- Empêche la répétition de certaines mises à jour
- OPTIONNEL : En général, cela n'est pas trouvé

Décompressez le Kernelcache :
```bash
# img4tool (https://github.com/tihmstar/img4tool)
img4tool -e kernelcache.release.iphone14 -o kernelcache.release.iphone14.e

# pyimg4 (https://github.com/m1stadev/PyIMG4)
pyimg4 im4p extract -i kernelcache.release.iphone14 -o kernelcache.release.iphone14.e
```
### Télécharger

- [**KernelDebugKit Github**](https://github.com/dortania/KdkSupportPkg/releases)

Dans [https://github.com/dortania/KdkSupportPkg/releases](https://github.com/dortania/KdkSupportPkg/releases), il est possible de trouver tous les kits de débogage du noyau. Vous pouvez le télécharger, le monter, l'ouvrir avec l'outil [Suspicious Package](https://www.mothersruin.com/software/SuspiciousPackage/get.html), accéder au dossier **`.kext`** et **l'extraire**.

Vérifiez-le pour les symboles avec :
```bash
nm -a ~/Downloads/Sandbox.kext/Contents/MacOS/Sandbox | wc -l
```
- [**theapplewiki.com**](https://theapplewiki.com/wiki/Firmware/Mac/14.x)**,** [**ipsw.me**](https://ipsw.me/)**,** [**theiphonewiki.com**](https://www.theiphonewiki.com/)

Parfois, Apple publie **kernelcache** avec des **symbols**. Vous pouvez télécharger certains firmwares avec des symbols en suivant les liens sur ces pages. Les firmwares contiendront le **kernelcache** parmi d'autres fichiers.

Pour **extract** les fichiers, commencez par changer l'extension de `.ipsw` à `.zip` et **unzip** le.

Après avoir extrait le firmware, vous obtiendrez un fichier comme : **`kernelcache.release.iphone14`**. Il est au format **IMG4**, vous pouvez extraire les informations intéressantes avec :

[**pyimg4**](https://github.com/m1stadev/PyIMG4)**:**
```bash
pyimg4 im4p extract -i kernelcache.release.iphone14 -o kernelcache.release.iphone14.e
```
[**img4tool**](https://github.com/tihmstar/img4tool)**:**
```bash
img4tool -e kernelcache.release.iphone14 -o kernelcache.release.iphone14.e
```
### Inspection du kernelcache

Vérifiez si le kernelcache a des symboles avec
```bash
nm -a kernelcache.release.iphone14.e | wc -l
```
Avec cela, nous pouvons maintenant **extraire toutes les extensions** ou **celle qui vous intéresse :**
```bash
# List all extensions
kextex -l kernelcache.release.iphone14.e
## Extract com.apple.security.sandbox
kextex -e com.apple.security.sandbox kernelcache.release.iphone14.e

# Extract all
kextex_all kernelcache.release.iphone14.e

# Check the extension for symbols
nm -a binaries/com.apple.security.sandbox | wc -l
```
## Vulnérabilités récentes et techniques d'exploitation

| Année | CVE | Résumé |
|------|-----|---------|
| 2024 | **CVE-2024-44243** | Un défaut logique dans **`storagekitd`** a permis à un attaquant *root* d'enregistrer un bundle de système de fichiers malveillant qui a finalement chargé un **kext non signé**, **contournant la Protection de l'Intégrité du Système (SIP)** et permettant des rootkits persistants. Corrigé dans macOS 14.2 / 15.2.   |
| 2021 | **CVE-2021-30892** (*Shrootless*) | Le démon d'installation avec le droit `com.apple.rootless.install` pouvait être abusé pour exécuter des scripts post-installation arbitraires, désactiver le SIP et charger des kexts arbitraires.  |

**À retenir pour les red-teamers**

1. **Recherchez des démons autorisés (`codesign -dvv /path/bin | grep entitlements`) qui interagissent avec Disk Arbitration, Installer ou Kext Management.**
2. **L'abus des contournements de SIP accorde presque toujours la capacité de charger un kext → exécution de code dans le noyau**.

**Conseils défensifs**

*Gardez le SIP activé*, surveillez les invocations `kmutil load`/`kmutil create -n aux` provenant de binaires non-Apple et alertez sur toute écriture dans `/Library/Extensions`. Les événements de sécurité des points de terminaison `ES_EVENT_TYPE_NOTIFY_KEXTLOAD` fournissent une visibilité quasi en temps réel.

## Débogage du noyau macOS et des kexts

Le flux de travail recommandé par Apple est de construire un **Kernel Debug Kit (KDK)** qui correspond à la version en cours d'exécution, puis de connecter **LLDB** via une session réseau **KDP (Kernel Debugging Protocol)**.

### Débogage local à usage unique d'un panic
```bash
# Create a symbolication bundle for the latest panic
sudo kdpwrit dump latest.kcdata
kmutil analyze-panic latest.kcdata -o ~/panic_report.txt
```
### Débogage à distance en direct depuis un autre Mac

1. Téléchargez et installez la version **KDK** exacte pour la machine cible.
2. Connectez le Mac cible et le Mac hôte avec un **câble USB-C ou Thunderbolt**.
3. Sur le **cible** :
```bash
sudo nvram boot-args="debug=0x100 kdp_match_name=macbook-target"
reboot
```
4. Sur l'**hôte** :
```bash
lldb
(lldb) kdp-remote "udp://macbook-target"
(lldb) bt  # get backtrace in kernel context
```
### Attacher LLDB à un kext chargé spécifique
```bash
# Identify load address of the kext
ADDR=$(kmutil showloaded --bundle-identifier com.example.driver | awk '{print $4}')

# Attach
sudo lldb -n kernel_task -o "target modules load --file /Library/Extensions/Example.kext/Contents/MacOS/Example --slide $ADDR"
```
> ℹ️  KDP n'expose qu'une interface **en lecture seule**. Pour l'instrumentation dynamique, vous devrez patcher le binaire sur disque, tirer parti du **hooking de fonction du noyau** (par exemple, `mach_override`) ou migrer le pilote vers un **hyperviseur** pour un accès complet en lecture/écriture.

## Références

- DriverKit Security – Apple Platform Security Guide
- Microsoft Security Blog – *Analyzing CVE-2024-44243 SIP bypass*

{{#include ../../../banners/hacktricks-training.md}}
