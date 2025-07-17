# Vulnerabilidades del Kernel de macOS

{{#include ../../../banners/hacktricks-training.md}}

## [Pwning OTA](https://jhftss.github.io/The-Nightmare-of-Apple-OTA-Update/)

[**En este informe**](https://jhftss.github.io/The-Nightmare-of-Apple-OTA-Update/) se explican varias vulnerabilidades que permitieron comprometer el kernel comprometiendo el actualizador de software.\
[**PoC**](https://github.com/jhftss/POC/tree/main/CVE-2022-46722).

---

## 2024: Vulnerabilidades 0-days en la naturaleza (CVE-2024-23225 & CVE-2024-23296)

Apple parcheó dos errores de corrupción de memoria que fueron explotados activamente contra iOS y macOS en marzo de 2024 (corregido en macOS 14.4/13.6.5/12.7.4).

* **CVE-2024-23225 – Kernel**
• Escritura fuera de límites en el subsistema de memoria virtual XNU permite a un proceso no privilegiado obtener lectura/escritura arbitraria en el espacio de direcciones del kernel, eludiendo PAC/KTRR.
• Activado desde el espacio de usuario a través de un mensaje XPC manipulado que desborda un búfer en `libxpc`, luego pivota al kernel cuando se analiza el mensaje.
* **CVE-2024-23296 – RTKit**
• Corrupción de memoria en el RTKit de Apple Silicon (coprocesador en tiempo real).
• Las cadenas de explotación observadas utilizaron CVE-2024-23225 para R/W del kernel y CVE-2024-23296 para escapar del sandbox del coprocesador seguro y deshabilitar PAC.

Detección del nivel de parche:
```bash
sw_vers                 # ProductVersion 14.4 or later is patched
authenticate sudo sysctl kern.osversion  # 23E214 or later for Sonoma
```
Si la actualización no es posible, mitigue deshabilitando los servicios vulnerables:
```bash
launchctl disable system/com.apple.analyticsd
launchctl disable system/com.apple.rtcreportingd
```
---

## 2023: MIG Type-Confusion – CVE-2023-41075

`mach_msg()` solicitudes enviadas a un cliente de usuario IOKit sin privilegios conducen a una **confusión de tipo** en el código de pegado generado por MIG. Cuando el mensaje de respuesta se reinterpreta con un descriptor fuera de línea más grande de lo que se asignó originalmente, un atacante puede lograr una **escritura OOB** controlada en las zonas de heap del kernel y eventualmente escalar a `root`.

Esquema primitivo (Sonoma 14.0-14.1, Ventura 13.5-13.6):
```c
// userspace stub
typed_port_t p = get_user_client();
uint8_t spray[0x4000] = {0x41};
// heap-spray via IOSurfaceFastSetValue
io_service_open_extended(...);
// malformed MIG message triggers confusion
mach_msg(&msg.header, MACH_SEND_MSG|MACH_RCV_MSG, ...);
```
Exploits públicos aprovechan el error al:
1. Rociar los buffers `ipc_kmsg` con punteros de puerto activos.
2. Sobrescribir `ip_kobject` de un puerto colgante.
3. Saltar a shellcode mapeado en una dirección forjada por PAC usando `mprotect()`.

---

## 2024-2025: Bypass de SIP a través de Kexts de terceros – CVE-2024-44243 (también conocido como “Sigma”)

Investigadores de seguridad de Microsoft demostraron que el demonio de alto privilegio `storagekitd` puede ser forzado a cargar una **extensión de kernel no firmada** y así deshabilitar completamente la **Protección de Integridad del Sistema (SIP)** en macOS completamente parcheado (anterior a 15.2). El flujo del ataque es:

1. Abusar del derecho privado `com.apple.storagekitd.kernel-management` para generar un ayudante bajo el control del atacante.
2. El ayudante llama a `IOService::AddPersonalitiesFromKernelModule` con un diccionario de información elaborado que apunta a un paquete de kext malicioso.
3. Debido a que las verificaciones de confianza de SIP se realizan *después* de que el kext es preparado por `storagekitd`, el código se ejecuta en ring-0 antes de la validación y SIP puede ser desactivado con `csr_set_allow_all(1)`.

Consejos de detección:
```bash
kmutil showloaded | grep -v com.apple   # list non-Apple kexts
log stream --style syslog --predicate 'senderImagePath contains "storagekitd"'   # watch for suspicious child procs
```
La remediación inmediata es actualizar a macOS Sequoia 15.2 o posterior.

---

### Hoja de trucos de enumeración rápida
```bash
uname -a                          # Kernel build
kmutil showloaded                 # List loaded kernel extensions
kextstat | grep -v com.apple      # Legacy (pre-Catalina) kext list
sysctl kern.kaslr_enable          # Verify KASLR is ON (should be 1)
csrutil status                    # Check SIP from RecoveryOS
spctl --status                    # Confirms Gatekeeper state
```
---

## Fuzzing & Research Tools

* **Luftrauser** – Fuzzer de mensajes Mach que apunta a subsistemas MIG (`github.com/preshing/luftrauser`).
* **oob-executor** – Generador de primitivas fuera de límites IPC utilizado en la investigación de CVE-2024-23225.
* **kmutil inspect** – Utilidad integrada de Apple (macOS 11+) para analizar estáticamente kexts antes de cargarlos: `kmutil inspect -b io.kext.bundleID`.



## References

* Apple. “About the security content of macOS Sonoma 14.4.” https://support.apple.com/en-us/120895
* Microsoft Security Blog. “Analyzing CVE-2024-44243, a macOS System Integrity Protection bypass through kernel extensions.” https://www.microsoft.com/en-us/security/blog/2025/01/13/analyzing-cve-2024-44243-a-macos-system-integrity-protection-bypass-through-kernel-extensions/
{{#include ../../../banners/hacktricks-training.md}}
