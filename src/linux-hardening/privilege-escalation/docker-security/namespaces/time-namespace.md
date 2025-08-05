# Time Namespace

{{#include ../../../../banners/hacktricks-training.md}}

## Informations de base

Le namespace de temps dans Linux permet des décalages par namespace aux horloges monotoniques et de démarrage du système. Il est couramment utilisé dans les conteneurs Linux pour changer la date/heure à l'intérieur d'un conteneur et ajuster les horloges après la restauration d'un point de contrôle ou d'un instantané.

## Laboratoire :

### Créer différents Namespaces

#### CLI
```bash
sudo unshare -T [--mount-proc] /bin/bash
```
En montant une nouvelle instance du système de fichiers `/proc` si vous utilisez le paramètre `--mount-proc`, vous vous assurez que le nouveau namespace de montage a une **vue précise et isolée des informations sur les processus spécifiques à ce namespace**.

<details>

<summary>Erreur : bash : fork : Impossible d'allouer de la mémoire</summary>

Lorsque `unshare` est exécuté sans l'option `-f`, une erreur est rencontrée en raison de la façon dont Linux gère les nouveaux namespaces PID (Process ID). Les détails clés et la solution sont décrits ci-dessous :

1. **Explication du problème** :

- Le noyau Linux permet à un processus de créer de nouveaux namespaces en utilisant l'appel système `unshare`. Cependant, le processus qui initie la création d'un nouveau namespace PID (appelé le processus "unshare") n'entre pas dans le nouveau namespace ; seuls ses processus enfants le font.
- L'exécution de `%unshare -p /bin/bash%` démarre `/bin/bash` dans le même processus que `unshare`. Par conséquent, `/bin/bash` et ses processus enfants se trouvent dans l'espace de noms PID d'origine.
- Le premier processus enfant de `/bin/bash` dans le nouveau namespace devient PID 1. Lorsque ce processus se termine, il déclenche le nettoyage du namespace s'il n'y a pas d'autres processus, car PID 1 a le rôle spécial d'adopter les processus orphelins. Le noyau Linux désactivera alors l'allocation de PID dans ce namespace.

2. **Conséquence** :

- La sortie de PID 1 dans un nouveau namespace entraîne le nettoyage du drapeau `PIDNS_HASH_ADDING`. Cela entraîne l'échec de la fonction `alloc_pid` à allouer un nouveau PID lors de la création d'un nouveau processus, produisant l'erreur "Impossible d'allouer de la mémoire".

3. **Solution** :
- Le problème peut être résolu en utilisant l'option `-f` avec `unshare`. Cette option permet à `unshare` de forker un nouveau processus après avoir créé le nouveau namespace PID.
- L'exécution de `%unshare -fp /bin/bash%` garantit que la commande `unshare` elle-même devient PID 1 dans le nouveau namespace. `/bin/bash` et ses processus enfants sont alors en toute sécurité contenus dans ce nouveau namespace, empêchant la sortie prématurée de PID 1 et permettant une allocation normale de PID.

En veillant à ce que `unshare` s'exécute avec le drapeau `-f`, le nouveau namespace PID est correctement maintenu, permettant à `/bin/bash` et à ses sous-processus de fonctionner sans rencontrer l'erreur d'allocation de mémoire.

</details>

#### Docker
```bash
docker run -ti --name ubuntu1 -v /usr:/ubuntu1 ubuntu bash
```
### Vérifiez dans quel espace de noms se trouve votre processus
```bash
ls -l /proc/self/ns/time
lrwxrwxrwx 1 root root 0 Apr  4 21:16 /proc/self/ns/time -> 'time:[4026531834]'
```
### Trouver tous les espaces de noms temporels
```bash
sudo find /proc -maxdepth 3 -type l -name time -exec readlink {} \; 2>/dev/null | sort -u
# Find the processes with an specific namespace
sudo find /proc -maxdepth 3 -type l -name time -exec ls -l  {} \; 2>/dev/null | grep <ns-number>
```
### Entrer dans un espace de noms temporel
```bash
nsenter -T TARGET_PID --pid /bin/bash
```
## Manipulation des décalages temporels

À partir de Linux 5.6, deux horloges peuvent être virtualisées par espace de noms temporel :

* `CLOCK_MONOTONIC`
* `CLOCK_BOOTTIME`

Leurs deltas par espace de noms sont exposés (et peuvent être modifiés) via le fichier `/proc/<PID>/timens_offsets` :
```
$ sudo unshare -Tr --mount-proc bash   # -T creates a new timens, -r drops capabilities
$ cat /proc/$$/timens_offsets
monotonic 0
boottime  0
```
Le fichier contient deux lignes - une par horloge - avec le décalage en **nanosecondes**. Les processus qui détiennent **CAP_SYS_TIME** _dans l'espace de noms temporel_ peuvent changer la valeur :
```
# advance CLOCK_MONOTONIC by two days (172 800 s)
echo "monotonic 172800000000000" > /proc/$$/timens_offsets
# verify
$ cat /proc/$$/uptime   # first column uses CLOCK_MONOTONIC
172801.37  13.57
```
Si vous avez besoin que l'horloge murale (`CLOCK_REALTIME`) change également, vous devez toujours vous fier aux mécanismes classiques (`date`, `hwclock`, `chronyd`, …) ; elle **n'est pas** dans un espace de noms.

### `unshare(1)` flags d'aide (util-linux ≥ 2.38)
```
sudo unshare -T \
--monotonic="+24h"  \
--boottime="+7d"    \
--mount-proc         \
bash
```
Les options longues écrivent automatiquement les deltas choisis dans `timens_offsets` juste après la création de l'espace de noms, évitant un `echo` manuel.

---

## Support OCI & Runtime

* La **Spécification de Runtime OCI v1.1** (Nov 2023) a ajouté un type d'espace de noms `time` dédié et le champ `linux.timeOffsets` afin que les moteurs de conteneurs puissent demander une virtualisation du temps de manière portable.
* **runc >= 1.2.0** implémente cette partie de la spécification. Un fragment minimal de `config.json` ressemble à :
```json
{
"linux": {
"namespaces": [
{"type": "time"}
],
"timeOffsets": {
"monotonic": 86400,
"boottime": 600
}
}
}
```
Ensuite, exécutez le conteneur avec `runc run <id>`.

>  REMARQUE : runc **1.2.6** (Fév 2025) a corrigé un bug "exec dans le conteneur avec timens privé" qui pouvait entraîner un blocage et un potentiel DoS. Assurez-vous d'être sur ≥ 1.2.6 en production.

---

## Considérations de sécurité

1. **Capacité requise** – Un processus a besoin de **CAP_SYS_TIME** à l'intérieur de son espace de noms utilisateur/temps pour changer les offsets. Supprimer cette capacité dans le conteneur (par défaut dans Docker & Kubernetes) empêche toute manipulation.
2. **Pas de changements d'horloge murale** – Parce que `CLOCK_REALTIME` est partagé avec l'hôte, les attaquants ne peuvent pas falsifier les durées de vie des certificats, l'expiration des JWT, etc. via timens seul.
3. **Évasion des journaux / détection** – Les logiciels qui dépendent de `CLOCK_MONOTONIC` (par exemple, les limiteurs de débit basés sur le temps de disponibilité) peuvent être confus si l'utilisateur de l'espace de noms ajuste l'offset. Préférez `CLOCK_REALTIME` pour les horodatages pertinents en matière de sécurité.
4. **Surface d'attaque du noyau** – Même avec `CAP_SYS_TIME` supprimé, le code du noyau reste accessible ; gardez l'hôte à jour. Linux 5.6 → 5.12 a reçu plusieurs corrections de bugs timens (NULL-deref, problèmes de signe).

### Liste de contrôle de durcissement

* Supprimez `CAP_SYS_TIME` dans le profil par défaut de votre runtime de conteneur.
* Gardez les runtimes à jour (runc ≥ 1.2.6, crun ≥ 1.12).
* Fixez util-linux ≥ 2.38 si vous dépendez des helpers `--monotonic/--boottime`.
* Auditez le logiciel dans le conteneur qui lit **uptime** ou **CLOCK_MONOTONIC** pour une logique critique en matière de sécurité.

## Références

* man7.org – Page de manuel des espaces de noms temporels : <https://man7.org/linux/man-pages/man7/time_namespaces.7.html>
* Blog OCI – "OCI v1.1 : nouveaux espaces de noms time et RDT" (15 nov 2023) : <https://opencontainers.org/blog/2023/11/15/oci-spec-v1.1>

{{#include ../../../../banners/hacktricks-training.md}}
