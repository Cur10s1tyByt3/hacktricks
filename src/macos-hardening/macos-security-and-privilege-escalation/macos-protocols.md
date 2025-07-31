# Services et Protocoles Réseau macOS

{{#include ../../banners/hacktricks-training.md}}

## Services d'Accès à Distance

Ce sont les services macOS courants pour y accéder à distance.\
Vous pouvez activer/désactiver ces services dans `System Settings` --> `Sharing`

- **VNC**, connu sous le nom de “Screen Sharing” (tcp:5900)
- **SSH**, appelé “Remote Login” (tcp:22)
- **Apple Remote Desktop** (ARD), ou “Remote Management” (tcp:3283, tcp:5900)
- **AppleEvent**, connu sous le nom de “Remote Apple Event” (tcp:3031)

Vérifiez si l'un d'eux est activé en exécutant :
```bash
rmMgmt=$(netstat -na | grep LISTEN | grep tcp46 | grep "*.3283" | wc -l);
scrShrng=$(netstat -na | grep LISTEN | egrep 'tcp4|tcp6' | grep "*.5900" | wc -l);
flShrng=$(netstat -na | grep LISTEN | egrep 'tcp4|tcp6' | egrep "\\*.88|\\*.445|\\*.548" | wc -l);
rLgn=$(netstat -na | grep LISTEN | egrep 'tcp4|tcp6' | grep "*.22" | wc -l);
rAE=$(netstat -na | grep LISTEN | egrep 'tcp4|tcp6' | grep "*.3031" | wc -l);
bmM=$(netstat -na | grep LISTEN | egrep 'tcp4|tcp6' | grep "*.4488" | wc -l);
printf "\nThe following services are OFF if '0', or ON otherwise:\nScreen Sharing: %s\nFile Sharing: %s\nRemote Login: %s\nRemote Mgmt: %s\nRemote Apple Events: %s\nBack to My Mac: %s\n\n" "$scrShrng" "$flShrng" "$rLgn" "$rmMgmt" "$rAE" "$bmM";
```
### Pentesting ARD

Apple Remote Desktop (ARD) est une version améliorée de [Virtual Network Computing (VNC)](https://en.wikipedia.org/wiki/Virtual_Network_Computing) adaptée pour macOS, offrant des fonctionnalités supplémentaires. Une vulnérabilité notable dans ARD est sa méthode d'authentification pour le mot de passe de l'écran de contrôle, qui n'utilise que les 8 premiers caractères du mot de passe, la rendant sujette à des [attaques par force brute](https://thudinh.blogspot.com/2017/09/brute-forcing-passwords-with-thc-hydra.html) avec des outils comme Hydra ou [GoRedShell](https://github.com/ahhh/GoRedShell/), car il n'y a pas de limites de taux par défaut.

Les instances vulnérables peuvent être identifiées en utilisant le script `vnc-info` de **nmap**. Les services prenant en charge `VNC Authentication (2)` sont particulièrement sensibles aux attaques par force brute en raison de la troncature du mot de passe à 8 caractères.

Pour activer ARD pour diverses tâches administratives comme l'escalade de privilèges, l'accès GUI ou la surveillance des utilisateurs, utilisez la commande suivante :
```bash
sudo /System/Library/CoreServices/RemoteManagement/ARDAgent.app/Contents/Resources/kickstart -activate -configure -allowAccessFor -allUsers -privs -all -clientopts -setmenuextra -menuextra yes
```
ARD offre des niveaux de contrôle polyvalents, y compris l'observation, le contrôle partagé et le contrôle total, avec des sessions persistant même après des changements de mot de passe utilisateur. Il permet d'envoyer des commandes Unix directement, les exécutant en tant que root pour les utilisateurs administratifs. La planification des tâches et la recherche Spotlight à distance sont des fonctionnalités notables, facilitant des recherches à distance et à faible impact pour des fichiers sensibles sur plusieurs machines.

#### Vulnérabilités récentes de partage d'écran / ARD (2023-2025)

| Année | CVE | Composant | Impact | Corrigé dans |
|------|-----|-----------|--------|----------|
|2023|CVE-2023-42940|Partage d'écran|Le rendu incorrect de la session pourrait entraîner la transmission du *mauvais* bureau ou fenêtre, entraînant une fuite d'informations sensibles|macOS Sonoma 14.2.1 (Déc 2023) |
|2024|CVE-2024-23296|launchservicesd / login|Contournement de la protection de la mémoire du noyau pouvant être enchaîné après une connexion à distance réussie (exploité activement dans la nature)|macOS Ventura 13.6.4 / Sonoma 14.4 (Mar 2024) |

**Conseils de durcissement**

* Désactiver *Partage d'écran*/*Gestion à distance* lorsque cela n'est pas strictement nécessaire.
* Garder macOS entièrement à jour (Apple expédie généralement des correctifs de sécurité pour les trois dernières versions majeures).
* Utiliser un **Mot de passe fort** *et* appliquer l'option *“Les visionneuses VNC peuvent contrôler l'écran avec un mot de passe”* **désactivée** lorsque cela est possible.
* Mettre le service derrière un VPN au lieu d'exposer TCP 5900/3283 à Internet.
* Ajouter une règle de pare-feu d'application pour limiter `ARDAgent` au sous-réseau local :

```bash
sudo /usr/libexec/ApplicationFirewall/socketfilterfw --add /System/Library/CoreServices/RemoteManagement/ARDAgent.app/Contents/MacOS/ARDAgent
sudo /usr/libexec/ApplicationFirewall/socketfilterfw --setblockapp /System/Library/CoreServices/RemoteManagement/ARDAgent.app/Contents/MacOS/ARDAgent on
```

---

## Protocole Bonjour

Bonjour, une technologie conçue par Apple, permet aux **appareils sur le même réseau de détecter les services offerts par les autres**. Également connu sous le nom de Rendezvous, **Zero Configuration**, ou Zeroconf, il permet à un appareil de rejoindre un réseau TCP/IP, **de choisir automatiquement une adresse IP**, et de diffuser ses services aux autres appareils du réseau.

Le Zero Configuration Networking, fourni par Bonjour, garantit que les appareils peuvent :

- **Obtenir automatiquement une adresse IP** même en l'absence de serveur DHCP.
- Effectuer une **traduction nom-adresse** sans nécessiter de serveur DNS.
- **Découvrir les services** disponibles sur le réseau.

Les appareils utilisant Bonjour s'assigneront une **adresse IP du range 169.254/16** et vérifieront son unicité sur le réseau. Les Macs maintiennent une entrée de table de routage pour ce sous-réseau, vérifiable via `netstat -rn | grep 169`.

Pour le DNS, Bonjour utilise le **protocole Multicast DNS (mDNS)**. mDNS fonctionne sur **le port 5353/UDP**, utilisant des **requêtes DNS standard** mais ciblant l'**adresse multicast 224.0.0.251**. Cette approche garantit que tous les appareils à l'écoute sur le réseau peuvent recevoir et répondre aux requêtes, facilitant la mise à jour de leurs enregistrements.

Lorsqu'un appareil rejoint le réseau, il se sélectionne un nom, se terminant généralement par **.local**, qui peut être dérivé du nom d'hôte ou généré aléatoirement.

La découverte de services au sein du réseau est facilitée par **DNS Service Discovery (DNS-SD)**. Tirant parti du format des enregistrements DNS SRV, DNS-SD utilise des **enregistrements DNS PTR** pour permettre la liste de plusieurs services. Un client cherchant un service spécifique demandera un enregistrement PTR pour `<Service>.<Domain>`, recevant en retour une liste d'enregistrements PTR formatés comme `<Instance>.<Service>.<Domain>` si le service est disponible depuis plusieurs hôtes.

L'utilitaire `dns-sd` peut être utilisé pour **découvrir et annoncer des services réseau**. Voici quelques exemples de son utilisation :

### Recherche de services SSH

Pour rechercher des services SSH sur le réseau, la commande suivante est utilisée :
```bash
dns-sd -B _ssh._tcp
```
Cette commande initie la recherche de services \_ssh.\_tcp et affiche des détails tels que l'horodatage, les indicateurs, l'interface, le domaine, le type de service et le nom de l'instance.

### Annonce d'un service HTTP

Pour annoncer un service HTTP, vous pouvez utiliser :
```bash
dns-sd -R "Index" _http._tcp . 80 path=/index.html
```
Cette commande enregistre un service HTTP nommé "Index" sur le port 80 avec un chemin de `/index.html`.

Pour ensuite rechercher des services HTTP sur le réseau :
```bash
dns-sd -B _http._tcp
```
Lorsque un service démarre, il annonce sa disponibilité à tous les appareils sur le sous-réseau en diffusant sa présence. Les appareils intéressés par ces services n'ont pas besoin d'envoyer de demandes mais écoutent simplement ces annonces.

Pour une interface plus conviviale, l'application **Discovery - DNS-SD Browser** disponible sur l'Apple App Store peut visualiser les services offerts sur votre réseau local.

Alternativement, des scripts personnalisés peuvent être écrits pour parcourir et découvrir des services en utilisant la bibliothèque `python-zeroconf`. Le script [**python-zeroconf**](https://github.com/jstasiak/python-zeroconf) démontre la création d'un navigateur de services pour les services `_http._tcp.local.`, imprimant les services ajoutés ou supprimés :
```python
from zeroconf import ServiceBrowser, Zeroconf

class MyListener:

def remove_service(self, zeroconf, type, name):
print("Service %s removed" % (name,))

def add_service(self, zeroconf, type, name):
info = zeroconf.get_service_info(type, name)
print("Service %s added, service info: %s" % (name, info))

zeroconf = Zeroconf()
listener = MyListener()
browser = ServiceBrowser(zeroconf, "_http._tcp.local.", listener)
try:
input("Press enter to exit...\n\n")
finally:
zeroconf.close()
```
### Énumération de Bonjour sur le réseau

* **Nmap NSE** – découvrir les services annoncés par un seul hôte :

```bash
nmap -sU -p 5353 --script=dns-service-discovery <target>
```

Le script `dns-service-discovery` envoie une requête `_services._dns-sd._udp.local` puis énumère chaque type de service annoncé.

* **mdns_recon** – outil Python qui scanne des plages entières à la recherche de *mDNS* mal configurés qui répondent aux requêtes unicast (utile pour trouver des appareils accessibles à travers des sous-réseaux/WAN) :

```bash
git clone https://github.com/chadillac/mdns_recon && cd mdns_recon
python3 mdns_recon.py -r 192.0.2.0/24 -s _ssh._tcp.local
```

Cela renverra des hôtes exposant SSH via Bonjour en dehors du lien local.

### Considérations de sécurité & vulnérabilités récentes (2024-2025)

| Année | CVE | Gravité | Problème | Corrigé dans |
|------|-----|----------|-------|------------|
|2024|CVE-2024-44183|Moyenne|Une erreur logique dans *mDNSResponder* a permis à un paquet conçu de déclencher un **déni de service**|macOS Ventura 13.7 / Sonoma 14.7 / Sequoia 15.0 (Sep 2024) |
|2025|CVE-2025-31222|Élevée|Un problème de correction dans *mDNSResponder* pourrait être exploité pour une **élévation de privilèges locale**|macOS Ventura 13.7.6 / Sonoma 14.7.6 / Sequoia 15.5 (Mai 2025) |

**Conseils d'atténuation**

1. Restreindre l'UDP 5353 à la portée *link-local* – bloquer ou limiter le débit sur les contrôleurs sans fil, les routeurs et les pare-feu basés sur l'hôte.
2. Désactiver complètement Bonjour sur les systèmes qui ne nécessitent pas la découverte de services :

```bash
sudo launchctl unload -w /System/Library/LaunchDaemons/com.apple.mDNSResponder.plist
```
3. Pour les environnements où Bonjour est requis en interne mais ne doit jamais franchir les frontières réseau, utiliser des restrictions de profil *AirPlay Receiver* (MDM) ou un proxy mDNS.
4. Activer **System Integrity Protection (SIP)** et maintenir macOS à jour – les deux vulnérabilités ci-dessus ont été corrigées rapidement mais dépendaient de l'activation de SIP pour une protection complète.

### Désactivation de Bonjour

S'il y a des préoccupations concernant la sécurité ou d'autres raisons de désactiver Bonjour, cela peut être désactivé en utilisant la commande suivante :
```bash
sudo launchctl unload -w /System/Library/LaunchDaemons/com.apple.mDNSResponder.plist
```
## Références

- [**The Mac Hacker's Handbook**](https://www.amazon.com/-/es/Charlie-Miller-ebook-dp-B004U7MUMU/dp/B004U7MUMU/ref=mt_other?_encoding=UTF8&me=&qid=)
- [**https://taomm.org/vol1/analysis.html**](https://taomm.org/vol1/analysis.html)
- [**https://lockboxx.blogspot.com/2019/07/macos-red-teaming-206-ard-apple-remote.html**](https://lockboxx.blogspot.com/2019/07/macos-red-teaming-206-ard-apple-remote.html)
- [**NVD – CVE-2023-42940**](https://nvd.nist.gov/vuln/detail/CVE-2023-42940)
- [**NVD – CVE-2024-44183**](https://nvd.nist.gov/vuln/detail/CVE-2024-44183)

{{#include ../../banners/hacktricks-training.md}}
