# macOS Servicios de Red y Protocolos

{{#include ../../banners/hacktricks-training.md}}

## Servicios de Acceso Remoto

Estos son los servicios comunes de macOS para acceder a ellos de forma remota.\
Puedes habilitar/deshabilitar estos servicios en `System Settings` --> `Sharing`

- **VNC**, conocido como “Screen Sharing” (tcp:5900)
- **SSH**, llamado “Remote Login” (tcp:22)
- **Apple Remote Desktop** (ARD), o “Remote Management” (tcp:3283, tcp:5900)
- **AppleEvent**, conocido como “Remote Apple Event” (tcp:3031)

Verifica si alguno está habilitado ejecutando:
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

Apple Remote Desktop (ARD) es una versión mejorada de [Virtual Network Computing (VNC)](https://en.wikipedia.org/wiki/Virtual_Network_Computing) adaptada para macOS, que ofrece características adicionales. Una vulnerabilidad notable en ARD es su método de autenticación para la contraseña de la pantalla de control, que solo utiliza los primeros 8 caracteres de la contraseña, lo que la hace propensa a [ataques de fuerza bruta](https://thudinh.blogspot.com/2017/09/brute-forcing-passwords-with-thc-hydra.html) con herramientas como Hydra o [GoRedShell](https://github.com/ahhh/GoRedShell/), ya que no hay límites de tasa predeterminados.

Las instancias vulnerables se pueden identificar utilizando el script `vnc-info` de **nmap**. Los servicios que admiten `VNC Authentication (2)` son especialmente susceptibles a ataques de fuerza bruta debido a la truncación de la contraseña de 8 caracteres.

Para habilitar ARD para varias tareas administrativas como escalada de privilegios, acceso GUI o monitoreo de usuarios, utiliza el siguiente comando:
```bash
sudo /System/Library/CoreServices/RemoteManagement/ARDAgent.app/Contents/Resources/kickstart -activate -configure -allowAccessFor -allUsers -privs -all -clientopts -setmenuextra -menuextra yes
```
ARD proporciona niveles de control versátiles, incluyendo observación, control compartido y control total, con sesiones que persisten incluso después de cambios de contraseña de usuario. Permite enviar comandos Unix directamente, ejecutándolos como root para usuarios administrativos. La programación de tareas y la búsqueda remota de Spotlight son características notables, facilitando búsquedas remotas de bajo impacto para archivos sensibles en múltiples máquinas.

#### Vulnerabilidades recientes de Screen-Sharing / ARD (2023-2025)

| Año | CVE | Componente | Impacto | Solucionado en |
|------|-----|-----------|--------|----------|
|2023|CVE-2023-42940|Screen Sharing|El renderizado incorrecto de la sesión podría causar que se transmitiera el *escritorio* o ventana *incorrectos*, resultando en la filtración de información sensible|macOS Sonoma 14.2.1 (Dic 2023) |
|2024|CVE-2024-23296|launchservicesd / login|Bypass de protección de memoria del kernel que puede encadenarse después de un inicio de sesión remoto exitoso (explotado activamente en el medio)|macOS Ventura 13.6.4 / Sonoma 14.4 (Mar 2024) |

**Consejos de endurecimiento**

* Desactivar *Screen Sharing*/*Remote Management* cuando no sea estrictamente necesario.
* Mantener macOS completamente actualizado (Apple generalmente envía correcciones de seguridad para las tres últimas versiones principales).
* Usar una **Contraseña Fuerte** *y* hacer cumplir la opción *“Los visualizadores de VNC pueden controlar la pantalla con contraseña”* **desactivada** cuando sea posible.
* Colocar el servicio detrás de una VPN en lugar de exponer TCP 5900/3283 a Internet.
* Agregar una regla de Firewall de Aplicaciones para limitar `ARDAgent` a la subred local:

```bash
sudo /usr/libexec/ApplicationFirewall/socketfilterfw --add /System/Library/CoreServices/RemoteManagement/ARDAgent.app/Contents/MacOS/ARDAgent
sudo /usr/libexec/ApplicationFirewall/socketfilterfw --setblockapp /System/Library/CoreServices/RemoteManagement/ARDAgent.app/Contents/MacOS/ARDAgent on
```

---

## Protocolo Bonjour

Bonjour, una tecnología diseñada por Apple, permite que **los dispositivos en la misma red detecten los servicios ofrecidos entre sí**. También conocido como Rendezvous, **Zero Configuration**, o Zeroconf, permite que un dispositivo se una a una red TCP/IP, **elija automáticamente una dirección IP**, y transmita sus servicios a otros dispositivos de la red.

La Red de Configuración Cero, proporcionada por Bonjour, asegura que los dispositivos puedan:

- **Obtener automáticamente una dirección IP** incluso en ausencia de un servidor DHCP.
- Realizar **traducción de nombre a dirección** sin requerir un servidor DNS.
- **Descubrir servicios** disponibles en la red.

Los dispositivos que utilizan Bonjour se asignarán a sí mismos una **dirección IP del rango 169.254/16** y verificarán su unicidad en la red. Los Macs mantienen una entrada en la tabla de enrutamiento para esta subred, verificable a través de `netstat -rn | grep 169`.

Para DNS, Bonjour utiliza el **protocolo Multicast DNS (mDNS)**. mDNS opera sobre **puerto 5353/UDP**, empleando **consultas DNS estándar** pero dirigiéndose a la **dirección de multidifusión 224.0.0.251**. Este enfoque asegura que todos los dispositivos que escuchan en la red puedan recibir y responder a las consultas, facilitando la actualización de sus registros.

Al unirse a la red, cada dispositivo selecciona un nombre, que generalmente termina en **.local**, que puede derivarse del nombre del host o ser generado aleatoriamente.

El descubrimiento de servicios dentro de la red es facilitado por **DNS Service Discovery (DNS-SD)**. Aprovechando el formato de los registros DNS SRV, DNS-SD utiliza **registros DNS PTR** para permitir la enumeración de múltiples servicios. Un cliente que busca un servicio específico solicitará un registro PTR para `<Service>.<Domain>`, recibiendo a cambio una lista de registros PTR formateados como `<Instance>.<Service>.<Domain>` si el servicio está disponible desde múltiples hosts.

La utilidad `dns-sd` puede ser empleada para **descubrir y anunciar servicios de red**. Aquí hay algunos ejemplos de su uso:

### Buscando servicios SSH

Para buscar servicios SSH en la red, se utiliza el siguiente comando:
```bash
dns-sd -B _ssh._tcp
```
Este comando inicia la búsqueda de servicios \_ssh.\_tcp y muestra detalles como la marca de tiempo, las banderas, la interfaz, el dominio, el tipo de servicio y el nombre de la instancia.

### Publicitando un Servicio HTTP

Para publicitar un servicio HTTP, puedes usar:
```bash
dns-sd -R "Index" _http._tcp . 80 path=/index.html
```
Este comando registra un servicio HTTP llamado "Index" en el puerto 80 con una ruta de `/index.html`.

Para luego buscar servicios HTTP en la red:
```bash
dns-sd -B _http._tcp
```
Cuando un servicio se inicia, anuncia su disponibilidad a todos los dispositivos en la subred mediante la difusión de su presencia. Los dispositivos interesados en estos servicios no necesitan enviar solicitudes, sino que simplemente escuchan estos anuncios.

Para una interfaz más amigable, la aplicación **Discovery - DNS-SD Browser** disponible en la App Store de Apple puede visualizar los servicios ofrecidos en su red local.

Alternativamente, se pueden escribir scripts personalizados para explorar y descubrir servicios utilizando la biblioteca `python-zeroconf`. El script [**python-zeroconf**](https://github.com/jstasiak/python-zeroconf) demuestra cómo crear un navegador de servicios para los servicios `_http._tcp.local.`, imprimiendo los servicios añadidos o eliminados:
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
### Enumerando Bonjour a través de la red

* **Nmap NSE** – descubre servicios anunciados por un solo host:

```bash
nmap -sU -p 5353 --script=dns-service-discovery <target>
```

El script `dns-service-discovery` envía una consulta `_services._dns-sd._udp.local` y luego enumera cada tipo de servicio anunciado.

* **mdns_recon** – herramienta de Python que escanea rangos enteros en busca de *mDNS* respondedores *mal configurados* que responden a consultas unicast (útil para encontrar dispositivos accesibles a través de subredes/WAN):

```bash
git clone https://github.com/chadillac/mdns_recon && cd mdns_recon
python3 mdns_recon.py -r 192.0.2.0/24 -s _ssh._tcp.local
```

Esto devolverá hosts que exponen SSH a través de Bonjour fuera del enlace local.

### Consideraciones de seguridad y vulnerabilidades recientes (2024-2025)

| Año | CVE | Severidad | Problema | Parcheado en |
|------|-----|----------|-------|------------|
|2024|CVE-2024-44183|Medio|Un error lógico en *mDNSResponder* permitió que un paquete manipulado activara un **denegación de servicio**|macOS Ventura 13.7 / Sonoma 14.7 / Sequoia 15.0 (Sep 2024) |
|2025|CVE-2025-31222|Alto|Un problema de corrección en *mDNSResponder* podría ser abusado para **escalada de privilegios local**|macOS Ventura 13.7.6 / Sonoma 14.7.6 / Sequoia 15.5 (May 2025) |

**Guía de mitigación**

1. Restringir UDP 5353 al ámbito *link-local* – bloquear o limitar la tasa en controladores inalámbricos, enrutadores y firewalls basados en host.
2. Deshabilitar Bonjour por completo en sistemas que no requieren descubrimiento de servicios:

```bash
sudo launchctl unload -w /System/Library/LaunchDaemons/com.apple.mDNSResponder.plist
```
3. Para entornos donde Bonjour es necesario internamente pero nunca debe cruzar fronteras de red, usar restricciones de perfil de *AirPlay Receiver* (MDM) o un proxy mDNS.
4. Habilitar **System Integrity Protection (SIP)** y mantener macOS actualizado – ambas vulnerabilidades anteriores fueron parcheadas rápidamente pero dependían de que SIP estuviera habilitado para una protección completa.

### Deshabilitando Bonjour

Si hay preocupaciones sobre la seguridad u otras razones para deshabilitar Bonjour, se puede apagar usando el siguiente comando:
```bash
sudo launchctl unload -w /System/Library/LaunchDaemons/com.apple.mDNSResponder.plist
```
## Referencias

- [**The Mac Hacker's Handbook**](https://www.amazon.com/-/es/Charlie-Miller-ebook-dp-B004U7MUMU/dp/B004U7MUMU/ref=mt_other?_encoding=UTF8&me=&qid=)
- [**https://taomm.org/vol1/analysis.html**](https://taomm.org/vol1/analysis.html)
- [**https://lockboxx.blogspot.com/2019/07/macos-red-teaming-206-ard-apple-remote.html**](https://lockboxx.blogspot.com/2019/07/macos-red-teaming-206-ard-apple-remote.html)
- [**NVD – CVE-2023-42940**](https://nvd.nist.gov/vuln/detail/CVE-2023-42940)
- [**NVD – CVE-2024-44183**](https://nvd.nist.gov/vuln/detail/CVE-2024-44183)

{{#include ../../banners/hacktricks-training.md}}
