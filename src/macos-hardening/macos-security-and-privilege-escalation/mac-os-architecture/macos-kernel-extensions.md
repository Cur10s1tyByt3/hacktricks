# macOS Kernel Extensions & Debugging

{{#include ../../../banners/hacktricks-training.md}}

## Información Básica

Las extensiones del kernel (Kexts) son **paquetes** con una extensión **`.kext`** que se **cargan directamente en el espacio del kernel de macOS**, proporcionando funcionalidad adicional al sistema operativo principal.

### Estado de desuso y DriverKit / Extensiones del Sistema
A partir de **macOS Catalina (10.15)**, Apple marcó la mayoría de los KPI heredados como *obsoletos* e introdujo los **marcos de Extensiones del Sistema y DriverKit** que se ejecutan en **espacio de usuario**. Desde **macOS Big Sur (11)**, el sistema operativo *se negará a cargar* kexts de terceros que dependan de KPI obsoletos a menos que la máquina se inicie en modo de **Seguridad Reducida**. En Apple Silicon, habilitar kexts también requiere que el usuario:

1. Reinicie en **Recuperación** → *Utilidad de Seguridad de Inicio*.
2. Seleccione **Seguridad Reducida** y marque **“Permitir la gestión de extensiones del kernel por parte de desarrolladores identificados”**.
3. Reinicie y apruebe el kext desde **Configuración del Sistema → Privacidad y Seguridad**.

Los controladores de espacio de usuario escritos con DriverKit/Extensiones del Sistema reducen drásticamente la **superficie de ataque** porque los bloqueos o la corrupción de memoria se confinan a un proceso en sandbox en lugar de al espacio del kernel.

> 📝 Desde macOS Sequoia (15), Apple ha eliminado por completo varios KPI de red y USB heredados; la única solución compatible hacia adelante para los proveedores es migrar a Extensiones del Sistema.

### Requisitos

Obviamente, esto es tan poderoso que es **complicado cargar una extensión del kernel**. Estos son los **requisitos** que una extensión del kernel debe cumplir para ser cargada:

- Al **ingresar al modo de recuperación**, las **extensiones del kernel deben ser permitidas** para ser cargadas:

<figure><img src="../../../images/image (327).png" alt=""><figcaption></figcaption></figure>

- La extensión del kernel debe estar **firmada con un certificado de firma de código del kernel**, que solo puede ser **otorgado por Apple**. Quien revisará en detalle la empresa y las razones por las que se necesita.
- La extensión del kernel también debe estar **notarizada**, Apple podrá verificarla en busca de malware.
- Luego, el usuario **root** es quien puede **cargar la extensión del kernel** y los archivos dentro del paquete deben **pertenecer a root**.
- Durante el proceso de carga, el paquete debe estar preparado en una **ubicación protegida no root**: `/Library/StagedExtensions` (requiere el otorgamiento de `com.apple.rootless.storage.KernelExtensionManagement`).
- Finalmente, al intentar cargarlo, el usuario [**recibirá una solicitud de confirmación**](https://developer.apple.com/library/archive/technotes/tn2459/_index.html) y, si se acepta, la computadora debe ser **reiniciada** para cargarlo.

### Proceso de carga

En Catalina era así: Es interesante notar que el proceso de **verificación** ocurre en **espacio de usuario**. Sin embargo, solo las aplicaciones con el otorgamiento **`com.apple.private.security.kext-management`** pueden **solicitar al kernel que cargue una extensión**: `kextcache`, `kextload`, `kextutil`, `kextd`, `syspolicyd`

1. **`kextutil`** cli **inicia** el proceso de **verificación** para cargar una extensión
- Se comunicará con **`kextd`** enviando usando un **servicio Mach**.
2. **`kextd`** verificará varias cosas, como la **firma**
- Se comunicará con **`syspolicyd`** para **verificar** si la extensión puede ser **cargada**.
3. **`syspolicyd`** **preguntará** al **usuario** si la extensión no ha sido cargada previamente.
- **`syspolicyd`** informará el resultado a **`kextd`**
4. **`kextd`** finalmente podrá **decirle al kernel que cargue** la extensión

Si **`kextd`** no está disponible, **`kextutil`** puede realizar las mismas verificaciones.

### Enumeración y gestión (kexts cargados)

`kextstat` fue la herramienta histórica, pero está **obsoleta** en las versiones recientes de macOS. La interfaz moderna es **`kmutil`**:
```bash
# List every extension currently linked in the kernel, sorted by load address
sudo kmutil showloaded --sort

# Show only third-party / auxiliary collections
sudo kmutil showloaded --collection aux

# Unload a specific bundle
sudo kmutil unload -b com.example.mykext
```
La sintaxis anterior todavía está disponible para referencia:
```bash
# (Deprecated) Get loaded kernel extensions
kextstat

# (Deprecated) Get dependencies of the kext number 22
kextstat | grep " 22 " | cut -c2-5,50- | cut -d '(' -f1
```
`kmutil inspect` también se puede utilizar para **volcar el contenido de una Colección de Núcleo (KC)** o verificar que un kext resuelve todas las dependencias de símbolos:
```bash
# List fileset entries contained in the boot KC
kmutil inspect -B /System/Library/KernelCollections/BootKernelExtensions.kc --show-fileset-entries

# Check undefined symbols of a 3rd party kext before loading
kmutil libraries -p /Library/Extensions/FancyUSB.kext --undef-symbols
```
## Kernelcache

> [!CAUTION]
> Aunque se espera que las extensiones del kernel estén en `/System/Library/Extensions/`, si vas a esta carpeta **no encontrarás ningún binario**. Esto se debe al **kernelcache** y para revertir un `.kext` necesitas encontrar una manera de obtenerlo.

El **kernelcache** es una **versión precompilada y preenlazada del kernel XNU**, junto con controladores de dispositivo esenciales y **extensiones del kernel**. Se almacena en un formato **comprimido** y se descomprime en la memoria durante el proceso de arranque. El kernelcache facilita un **tiempo de arranque más rápido** al tener una versión lista para ejecutar del kernel y controladores cruciales disponibles, reduciendo el tiempo y los recursos que de otro modo se gastarían en cargar y enlazar dinámicamente estos componentes en el momento del arranque.

### Kernelcache Local

En iOS se encuentra en **`/System/Library/Caches/com.apple.kernelcaches/kernelcache`** en macOS puedes encontrarlo con: **`find / -name "kernelcache" 2>/dev/null`** \
En mi caso en macOS lo encontré en:

- `/System/Volumes/Preboot/1BAEB4B5-180B-4C46-BD53-51152B7D92DA/boot/DAD35E7BC0CDA79634C20BD1BD80678DFB510B2AAD3D25C1228BB34BCD0A711529D3D571C93E29E1D0C1264750FA043F/System/Library/Caches/com.apple.kernelcaches/kernelcache`

#### IMG4

El formato de archivo IMG4 es un formato contenedor utilizado por Apple en sus dispositivos iOS y macOS para **almacenar y verificar de manera segura** componentes de firmware (como **kernelcache**). El formato IMG4 incluye un encabezado y varias etiquetas que encapsulan diferentes piezas de datos, incluyendo la carga útil real (como un kernel o cargador de arranque), una firma y un conjunto de propiedades de manifiesto. El formato admite verificación criptográfica, permitiendo que el dispositivo confirme la autenticidad e integridad del componente de firmware antes de ejecutarlo.

Generalmente está compuesto por los siguientes componentes:

- **Carga útil (IM4P)**:
- A menudo comprimido (LZFSE4, LZSS, …)
- Opcionalmente cifrado
- **Manifiesto (IM4M)**:
- Contiene firma
- Diccionario adicional de clave/valor
- **Información de restauración (IM4R)**:
- También conocido como APNonce
- Previene la repetición de algunas actualizaciones
- OPCIONAL: Generalmente esto no se encuentra

Descomprimir el Kernelcache:
```bash
# img4tool (https://github.com/tihmstar/img4tool)
img4tool -e kernelcache.release.iphone14 -o kernelcache.release.iphone14.e

# pyimg4 (https://github.com/m1stadev/PyIMG4)
pyimg4 im4p extract -i kernelcache.release.iphone14 -o kernelcache.release.iphone14.e
```
### Descargar

- [**KernelDebugKit Github**](https://github.com/dortania/KdkSupportPkg/releases)

En [https://github.com/dortania/KdkSupportPkg/releases](https://github.com/dortania/KdkSupportPkg/releases) es posible encontrar todos los kits de depuración del kernel. Puedes descargarlo, montarlo, abrirlo con la herramienta [Suspicious Package](https://www.mothersruin.com/software/SuspiciousPackage/get.html), acceder a la carpeta **`.kext`** y **extraerlo**.

Verifícalo en busca de símbolos con:
```bash
nm -a ~/Downloads/Sandbox.kext/Contents/MacOS/Sandbox | wc -l
```
- [**theapplewiki.com**](https://theapplewiki.com/wiki/Firmware/Mac/14.x)**,** [**ipsw.me**](https://ipsw.me/)**,** [**theiphonewiki.com**](https://www.theiphonewiki.com/)

A veces Apple lanza **kernelcache** con **símbolos**. Puedes descargar algunos firmwares con símbolos siguiendo los enlaces en esas páginas. Los firmwares contendrán el **kernelcache** entre otros archivos.

Para **extraer** los archivos, comienza cambiando la extensión de `.ipsw` a `.zip` y **descomprímelo**.

Después de extraer el firmware, obtendrás un archivo como: **`kernelcache.release.iphone14`**. Está en formato **IMG4**, puedes extraer la información interesante con:

[**pyimg4**](https://github.com/m1stadev/PyIMG4)**:**
```bash
pyimg4 im4p extract -i kernelcache.release.iphone14 -o kernelcache.release.iphone14.e
```
[**img4tool**](https://github.com/tihmstar/img4tool)**:**
```bash
img4tool -e kernelcache.release.iphone14 -o kernelcache.release.iphone14.e
```
### Inspeccionando kernelcache

Verifica si el kernelcache tiene símbolos con
```bash
nm -a kernelcache.release.iphone14.e | wc -l
```
Con esto ahora podemos **extraer todas las extensiones** o la **que te interesa:**
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
## Vulnerabilidades recientes y técnicas de explotación

| Año | CVE | Resumen |
|------|-----|---------|
| 2024 | **CVE-2024-44243** | Un fallo lógico en **`storagekitd`** permitió a un atacante *root* registrar un paquete de sistema de archivos malicioso que finalmente cargó un **kext** **no firmado**, **eludiendo la Protección de Integridad del Sistema (SIP)** y habilitando rootkits persistentes. Corregido en macOS 14.2 / 15.2.   |
| 2021 | **CVE-2021-30892** (*Shrootless*) | El demonio de instalación con el derecho `com.apple.rootless.install` podría ser abusado para ejecutar scripts post-instalación arbitrarios, deshabilitar SIP y cargar kexts arbitrarios.  |

**Conclusiones para los equipos rojos**

1. **Busque demonios con derechos (`codesign -dvv /path/bin | grep entitlements`) que interactúen con Disk Arbitration, Installer o Kext Management.**
2. **El abuso de eludir SIP casi siempre otorga la capacidad de cargar un kext → ejecución de código en el kernel**.

**Consejos defensivos**

*Mantenga SIP habilitado*, monitoree las invocaciones de `kmutil load`/`kmutil create -n aux` provenientes de binarios que no son de Apple y alerte sobre cualquier escritura en `/Library/Extensions`. Los eventos de Seguridad de Endpoint `ES_EVENT_TYPE_NOTIFY_KEXTLOAD` proporcionan visibilidad casi en tiempo real.

## Depuración del kernel de macOS y kexts

El flujo de trabajo recomendado por Apple es construir un **Kernel Debug Kit (KDK)** que coincida con la versión en ejecución y luego adjuntar **LLDB** a través de una sesión de red de **KDP (Kernel Debugging Protocol)**.

### Depuración local de un pánico en un solo intento
```bash
# Create a symbolication bundle for the latest panic
sudo kdpwrit dump latest.kcdata
kmutil analyze-panic latest.kcdata -o ~/panic_report.txt
```
### Depuración remota en vivo desde otra Mac

1. Descarga e instala la versión exacta de **KDK** para la máquina objetivo.
2. Conecta la Mac objetivo y la Mac host con un **cable USB-C o Thunderbolt**.
3. En la **objetivo**:
```bash
sudo nvram boot-args="debug=0x100 kdp_match_name=macbook-target"
reboot
```
4. En el **host**:
```bash
lldb
(lldb) kdp-remote "udp://macbook-target"
(lldb) bt  # get backtrace in kernel context
```
### Adjuntando LLDB a un kext cargado específico
```bash
# Identify load address of the kext
ADDR=$(kmutil showloaded --bundle-identifier com.example.driver | awk '{print $4}')

# Attach
sudo lldb -n kernel_task -o "target modules load --file /Library/Extensions/Example.kext/Contents/MacOS/Example --slide $ADDR"
```
> ℹ️  KDP solo expone una interfaz **de solo lectura**. Para la instrumentación dinámica, necesitarás parchear el binario en disco, aprovechar el **enganche de funciones del kernel** (por ejemplo, `mach_override`) o migrar el controlador a un **hipervisor** para lectura/escritura completa.

## Referencias

- Seguridad de DriverKit – Guía de Seguridad de la Plataforma de Apple
- Blog de Seguridad de Microsoft – *Analizando la omisión de SIP CVE-2024-44243*

{{#include ../../../banners/hacktricks-training.md}}
