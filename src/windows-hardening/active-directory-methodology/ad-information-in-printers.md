# Información en Impresoras

{{#include ../../banners/hacktricks-training.md}}

Hay varios blogs en Internet que **destacan los peligros de dejar impresoras configuradas con LDAP con credenciales de inicio de sesión predeterminadas/débiles**.  \
Esto se debe a que un atacante podría **engañar a la impresora para que se autentique contra un servidor LDAP malicioso** (típicamente un `nc -vv -l -p 389` o `slapd -d 2` es suficiente) y capturar las **credenciales de la impresora en texto claro**.

Además, varias impresoras contendrán **registros con nombres de usuario** o incluso podrían ser capaces de **descargar todos los nombres de usuario** del Controlador de Dominio.

Toda esta **información sensible** y la común **falta de seguridad** hacen que las impresoras sean muy interesantes para los atacantes.

Algunos blogs introductorios sobre el tema:

- [https://www.ceos3c.com/hacking/obtaining-domain-credentials-printer-netcat/](https://www.ceos3c.com/hacking/obtaining-domain-credentials-printer-netcat/)
- [https://medium.com/@nickvangilder/exploiting-multifunction-printers-during-a-penetration-test-engagement-28d3840d8856](https://medium.com/@nickvangilder/exploiting-multifunction-printers-during-a-penetration-test-engagement-28d3840d8856)

---
## Configuración de la Impresora

- **Ubicación**: La lista de servidores LDAP generalmente se encuentra en la interfaz web (por ejemplo, *Red ➜ Configuración LDAP ➜ Configuración de LDAP*).
- **Comportamiento**: Muchos servidores web integrados permiten modificaciones del servidor LDAP **sin volver a ingresar credenciales** (característica de usabilidad → riesgo de seguridad).
- **Explotar**: Redirigir la dirección del servidor LDAP a un host controlado por el atacante y usar el botón *Probar Conexión* / *Sincronización de Libreta de Direcciones* para forzar a la impresora a vincularse contigo.

---
## Capturando Credenciales

### Método 1 – Escucha de Netcat
```bash
sudo nc -k -v -l -p 389     # LDAPS → 636 (or 3269)
```
Small/old MFPs pueden enviar un *simple-bind* en texto claro que netcat puede capturar. Los dispositivos modernos generalmente realizan una consulta anónima primero y luego intentan el bind, por lo que los resultados varían.

### Método 2 – Servidor LDAP rogue completo (recomendado)

Debido a que muchos dispositivos emitirán una búsqueda anónima *antes* de autenticar, levantar un verdadero daemon LDAP produce resultados mucho más confiables:
```bash
# Debian/Ubuntu example
sudo apt install slapd ldap-utils
sudo dpkg-reconfigure slapd   # set any base-DN – it will not be validated

# run slapd in foreground / debug 2
slapd -d 2 -h "ldap:///"      # only LDAP, no LDAPS
```
Cuando la impresora realiza su búsqueda, verás las credenciales en texto claro en la salida de depuración.

> 💡 También puedes usar `impacket/examples/ldapd.py` (Python rogue LDAP) o `Responder -w -r -f` para recolectar hashes NTLMv2 a través de LDAP/SMB.

---
## Vulnerabilidades Recientes de Pass-Back (2024-2025)

El pass-back *no* es un problema teórico: los proveedores siguen publicando avisos en 2024/2025 que describen exactamente esta clase de ataque.

### Xerox VersaLink – CVE-2024-12510 & CVE-2024-12511

El firmware ≤ 57.69.91 de las MFP Xerox VersaLink C70xx permitió a un administrador autenticado (o a cualquiera cuando las credenciales predeterminadas permanecen) hacer lo siguiente:

* **CVE-2024-12510 – LDAP pass-back**: cambiar la dirección del servidor LDAP y activar una búsqueda, lo que provoca que el dispositivo filtre las credenciales de Windows configuradas al host controlado por el atacante.
* **CVE-2024-12511 – SMB/FTP pass-back**: problema idéntico a través de destinos de *scan-to-folder*, filtrando credenciales NetNTLMv2 o credenciales FTP en texto claro.

Un simple listener como:
```bash
sudo nc -k -v -l -p 389     # capture LDAP bind
```
o un servidor SMB no autorizado (`impacket-smbserver`) es suficiente para recopilar las credenciales.

### Canon imageRUNNER / imageCLASS – Aviso 20 de mayo de 2025

Canon confirmó una **vulnerabilidad de retorno de SMTP/LDAP** en docenas de líneas de productos láser y MFP. Un atacante con acceso de administrador puede modificar la configuración del servidor y recuperar las credenciales almacenadas para LDAP **o** SMTP (muchas organizaciones utilizan una cuenta privilegiada para permitir el escaneo a correo).

La guía del proveedor recomienda explícitamente:

1. Actualizar al firmware corregido tan pronto como esté disponible.
2. Usar contraseñas de administrador fuertes y únicas.
3. Evitar cuentas AD privilegiadas para la integración de impresoras.

---
## Herramientas de Enumeración / Explotación Automatizadas

| Herramienta | Propósito | Ejemplo |
|-------------|-----------|---------|
| **PRET** (Printer Exploitation Toolkit) | Abuso de PostScript/PJL/PCL, acceso al sistema de archivos, verificación de credenciales predeterminadas, *descubrimiento SNMP* | `python pret.py 192.168.1.50 pjl` |
| **Praeda** | Recopilar configuración (incluidos libros de direcciones y credenciales LDAP) a través de HTTP/HTTPS | `perl praeda.pl -t 192.168.1.50` |
| **Responder / ntlmrelayx** | Capturar y retransmitir hashes NetNTLM desde el retorno de SMB/FTP | `responder -I eth0 -wrf` |
| **impacket-ldapd.py** | Servicio LDAP no autorizado ligero para recibir enlaces en texto claro | `python ldapd.py -debug` |

---
## Dureza y Detección

1. **Parchear / actualizar firmware** MFPs de inmediato (verificar boletines PSIRT del proveedor).
2. **Cuentas de Servicio de Mínimos Privilegios** – nunca usar Domain Admin para LDAP/SMB/SMTP; restringir a ámbitos de OU *solo de lectura*.
3. **Restringir el Acceso de Gestión** – colocar las interfaces web/IPP/SNMP de la impresora en una VLAN de gestión o detrás de un ACL/VPN.
4. **Deshabilitar Protocolos No Utilizados** – FTP, Telnet, raw-9100, cifrados SSL más antiguos.
5. **Habilitar Registro de Auditoría** – algunos dispositivos pueden syslog fallos de LDAP/SMTP; correlacionar enlaces inesperados.
6. **Monitorear enlaces LDAP en texto claro** de fuentes inusuales (las impresoras normalmente solo deben comunicarse con los DCs).
7. **SNMPv3 o deshabilitar SNMP** – la comunidad `public` a menudo filtra la configuración del dispositivo y LDAP.

---
## Referencias

- [https://grimhacker.com/2018/03/09/just-a-printer/](https://grimhacker.com/2018/03/09/just-a-printer/)
- Rapid7. “Vulnerabilidades de Ataque de Retorno de MFP Xerox VersaLink C7025.” Febrero de 2025.
- Canon PSIRT. “Mitigación de Vulnerabilidades Contra el Retorno de SMTP/LDAP para Impresoras Láser y Multifuncionales de Oficina Pequeña.” Mayo de 2025.

{{#include ../../banners/hacktricks-training.md}}
