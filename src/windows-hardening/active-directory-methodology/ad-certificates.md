# AD Certificates

{{#include ../../banners/hacktricks-training.md}}

## Introducción

### Componentes de un Certificado

- El **Sujeto** del certificado denota su propietario.
- Una **Clave Pública** se empareja con una clave privada para vincular el certificado a su legítimo propietario.
- El **Período de Validez**, definido por las fechas **NotBefore** y **NotAfter**, marca la duración efectiva del certificado.
- Un **Número de Serie** único, proporcionado por la Autoridad de Certificación (CA), identifica cada certificado.
- El **Emisor** se refiere a la CA que ha emitido el certificado.
- **SubjectAlternativeName** permite nombres adicionales para el sujeto, mejorando la flexibilidad de identificación.
- **Restricciones Básicas** identifican si el certificado es para una CA o una entidad final y definen restricciones de uso.
- **Usos de Clave Extendidos (EKUs)** delinean los propósitos específicos del certificado, como la firma de código o la encriptación de correos electrónicos, a través de Identificadores de Objetos (OIDs).
- El **Algoritmo de Firma** especifica el método para firmar el certificado.
- La **Firma**, creada con la clave privada del emisor, garantiza la autenticidad del certificado.

### Consideraciones Especiales

- **Nombres Alternativos del Sujeto (SANs)** amplían la aplicabilidad de un certificado a múltiples identidades, crucial para servidores con múltiples dominios. Los procesos de emisión seguros son vitales para evitar riesgos de suplantación por parte de atacantes que manipulan la especificación SAN.

### Autoridades de Certificación (CAs) en Active Directory (AD)

AD CS reconoce los certificados de CA en un bosque de AD a través de contenedores designados, cada uno con roles únicos:

- El contenedor de **Autoridades de Certificación** contiene certificados de CA raíz de confianza.
- El contenedor de **Servicios de Inscripción** detalla las CAs empresariales y sus plantillas de certificados.
- El objeto **NTAuthCertificates** incluye certificados de CA autorizados para la autenticación de AD.
- El contenedor de **AIA (Acceso a Información de Autoridad)** facilita la validación de la cadena de certificados con certificados de CA intermedios y cruzados.

### Adquisición de Certificados: Flujo de Solicitud de Certificado del Cliente

1. El proceso de solicitud comienza con los clientes encontrando una CA empresarial.
2. Se crea un CSR, que contiene una clave pública y otros detalles, después de generar un par de claves pública-privada.
3. La CA evalúa el CSR en función de las plantillas de certificados disponibles, emitiendo el certificado según los permisos de la plantilla.
4. Tras la aprobación, la CA firma el certificado con su clave privada y se lo devuelve al cliente.

### Plantillas de Certificados

Definidas dentro de AD, estas plantillas describen la configuración y permisos para emitir certificados, incluyendo EKUs permitidos y derechos de inscripción o modificación, críticos para gestionar el acceso a los servicios de certificados.

## Inscripción de Certificados

El proceso de inscripción para certificados es iniciado por un administrador que **crea una plantilla de certificado**, que luego es **publicada** por una Autoridad de Certificación Empresarial (CA). Esto hace que la plantilla esté disponible para la inscripción del cliente, un paso logrado al agregar el nombre de la plantilla al campo `certificatetemplates` de un objeto de Active Directory.

Para que un cliente solicite un certificado, deben otorgarse **derechos de inscripción**. Estos derechos están definidos por descriptores de seguridad en la plantilla de certificado y en la propia CA empresarial. Los permisos deben otorgarse en ambas ubicaciones para que una solicitud sea exitosa.

### Derechos de Inscripción de Plantillas

Estos derechos se especifican a través de Entradas de Control de Acceso (ACEs), detallando permisos como:

- Derechos de **Inscripción de Certificado** y **Autoinscripción de Certificado**, cada uno asociado con GUIDs específicos.
- **Derechos Extendidos**, que permiten todos los permisos extendidos.
- **ControlTotal/TodosGenericos**, proporcionando control completo sobre la plantilla.

### Derechos de Inscripción de CA Empresarial

Los derechos de la CA están delineados en su descriptor de seguridad, accesible a través de la consola de gestión de la Autoridad de Certificación. Algunas configuraciones incluso permiten a usuarios con bajos privilegios acceso remoto, lo que podría ser una preocupación de seguridad.

### Controles Adicionales de Emisión

Ciertos controles pueden aplicarse, como:

- **Aprobación del Gerente**: Coloca las solicitudes en un estado pendiente hasta que sean aprobadas por un gerente de certificados.
- **Agentes de Inscripción y Firmas Autorizadas**: Especifican el número de firmas requeridas en un CSR y los OIDs de Política de Aplicación necesarios.

### Métodos para Solicitar Certificados

Los certificados se pueden solicitar a través de:

1. **Protocolo de Inscripción de Certificados de Cliente de Windows** (MS-WCCE), utilizando interfaces DCOM.
2. **Protocolo Remoto ICertPassage** (MS-ICPR), a través de tuberías nombradas o TCP/IP.
3. La **interfaz web de inscripción de certificados**, con el rol de Inscripción Web de Autoridad de Certificación instalado.
4. El **Servicio de Inscripción de Certificados** (CES), en conjunto con el servicio de Política de Inscripción de Certificados (CEP).
5. El **Servicio de Inscripción de Dispositivos de Red** (NDES) para dispositivos de red, utilizando el Protocolo Simple de Inscripción de Certificados (SCEP).

Los usuarios de Windows también pueden solicitar certificados a través de la GUI (`certmgr.msc` o `certlm.msc`) o herramientas de línea de comandos (`certreq.exe` o el comando `Get-Certificate` de PowerShell).
```bash
# Example of requesting a certificate using PowerShell
Get-Certificate -Template "User" -CertStoreLocation "cert:\\CurrentUser\\My"
```
## Autenticación por Certificado

Active Directory (AD) admite la autenticación por certificado, utilizando principalmente los protocolos **Kerberos** y **Secure Channel (Schannel)**.

### Proceso de Autenticación Kerberos

En el proceso de autenticación Kerberos, la solicitud de un usuario para un Ticket Granting Ticket (TGT) se firma utilizando la **clave privada** del certificado del usuario. Esta solicitud pasa por varias validaciones por parte del controlador de dominio, incluyendo la **validez**, **ruta** y **estado de revocación** del certificado. Las validaciones también incluyen verificar que el certificado provenga de una fuente confiable y confirmar la presencia del emisor en el **almacén de certificados NTAUTH**. Las validaciones exitosas resultan en la emisión de un TGT. El objeto **`NTAuthCertificates`** en AD, que se encuentra en:
```bash
CN=NTAuthCertificates,CN=Public Key Services,CN=Services,CN=Configuration,DC=<domain>,DC=<com>
```
es central para establecer confianza en la autenticación de certificados.

### Autenticación de Canal Seguro (Schannel)

Schannel facilita conexiones seguras TLS/SSL, donde durante un apretón de manos, el cliente presenta un certificado que, si se valida con éxito, autoriza el acceso. La asignación de un certificado a una cuenta de AD puede involucrar la función **S4U2Self** de Kerberos o el **Nombre Alternativo del Sujeto (SAN)** del certificado, entre otros métodos.

### Enumeración de Servicios de Certificados de AD

Los servicios de certificados de AD se pueden enumerar a través de consultas LDAP, revelando información sobre **Autoridades de Certificación (CAs) Empresariales** y sus configuraciones. Esto es accesible para cualquier usuario autenticado en el dominio sin privilegios especiales. Herramientas como **[Certify](https://github.com/GhostPack/Certify)** y **[Certipy](https://github.com/ly4k/Certipy)** se utilizan para la enumeración y evaluación de vulnerabilidades en entornos de AD CS.

Los comandos para usar estas herramientas incluyen:
```bash
# Enumerate trusted root CA certificates and Enterprise CAs with Certify
Certify.exe cas
# Identify vulnerable certificate templates with Certify
Certify.exe find /vulnerable

# Use Certipy (>=4.0) for enumeration and identifying vulnerable templates
certipy find -vulnerable -dc-only -u john@corp.local -p Passw0rd -target dc.corp.local

# Request a certificate over the web enrollment interface (new in Certipy 4.x)
certipy req -web -target ca.corp.local -template WebServer -upn john@corp.local -dns www.corp.local

# Enumerate Enterprise CAs and certificate templates with certutil
certutil.exe -TCAInfo
certutil -v -dstemplate
```
---

## Vulnerabilidades recientes y actualizaciones de seguridad (2022-2025)

| Año | ID / Nombre | Impacto | Conclusiones clave |
|------|-----------|--------|----------------|
| 2022 | **CVE-2022-26923** – “Certifried” / ESC6 | *Escalación de privilegios* al suplantar certificados de cuentas de máquina durante PKINIT. | El parche está incluido en las actualizaciones de seguridad del **10 de mayo de 2022**. Se introdujeron controles de auditoría y mapeo fuerte a través de **KB5014754**; los entornos ahora deberían estar en modo *Full Enforcement*.  |
| 2023 | **CVE-2023-35350 / 35351** | *Ejecución remota de código* en los roles de Inscripción Web de AD CS (certsrv) y CES. | Los PoCs públicos son limitados, pero los componentes vulnerables de IIS a menudo están expuestos internamente. Parche a partir del **julio de 2023** Patch Tuesday.  |
| 2024 | **CVE-2024-49019** – “EKUwu” / ESC15 | Los usuarios con bajos privilegios y derechos de inscripción podrían anular **cualquier** EKU o SAN durante la generación de CSR, emitiendo certificados utilizables para autenticación de cliente o firma de código, lo que lleva a un *compromiso de dominio*. | Abordado en las actualizaciones de **abril de 2024**. Eliminar “Suministrar en la solicitud” de las plantillas y restringir los permisos de inscripción.  |

### Cronología de endurecimiento de Microsoft (KB5014754)

Microsoft introdujo un despliegue en tres fases (Compatibilidad → Auditoría → Aplicación) para mover la autenticación de certificados Kerberos lejos de mapeos implícitos débiles. A partir del **11 de febrero de 2025**, los controladores de dominio cambian automáticamente a **Full Enforcement** si el valor del registro `StrongCertificateBindingEnforcement` no está configurado. Los administradores deben:

1. Parchear todos los DCs y servidores AD CS (mayo de 2022 o posterior).
2. Monitorear el ID de Evento 39/41 para mapeos débiles durante la fase de *Auditoría*.
3. Reemitir certificados de autenticación de cliente con la nueva **extensión SID** o configurar mapeos manuales fuertes antes de febrero de 2025.

---

## Mejoras en la detección y endurecimiento

* El **sensor Defender for Identity AD CS (2023-2024)** ahora presenta evaluaciones de postura para ESC1-ESC8/ESC11 y genera alertas en tiempo real como *“Emisión de certificado de controlador de dominio para un no-DC”* (ESC8) y *“Prevenir la Inscripción de Certificados con Políticas de Aplicación arbitrarias”* (ESC15). Asegúrese de que los sensores estén desplegados en todos los servidores AD CS para beneficiarse de estas detecciones.
* Desactive o limite estrictamente la opción **“Suministrar en la solicitud”** en todas las plantillas; prefiera valores SAN/EKU definidos explícitamente.
* Elimine **Cualquier Propósito** o **Sin EKU** de las plantillas a menos que sea absolutamente necesario (aborda escenarios ESC2).
* Requiera **aprobación del gerente** o flujos de trabajo dedicados de Agente de Inscripción para plantillas sensibles (por ejemplo, WebServer / CodeSigning).
* Restringa la inscripción web (`certsrv`) y los puntos finales de CES/NDES a redes de confianza o detrás de la autenticación de certificados de cliente.
* Aplique cifrado de inscripción RPC (`certutil –setreg CA\InterfaceFlags +IF_ENFORCEENCRYPTICERTREQ`) para mitigar ESC11.

---

## Referencias

- [https://www.specterops.io/assets/resources/Certified_Pre-Owned.pdf](https://www.specterops.io/assets/resources/Certified_Pre-Owned.pdf)
- [https://comodosslstore.com/blog/what-is-ssl-tls-client-authentication-how-does-it-work.html](https://comodosslstore.com/blog/what-is-ssl-tls-client-authentication-how-does-it-work.html)
- [https://support.microsoft.com/en-us/topic/kb5014754-certificate-based-authentication-changes-on-windows-domain-controllers-ad2c23b0-15d8-4340-a468-4d4f3b188f16](https://support.microsoft.com/en-us/topic/kb5014754-certificate-based-authentication-changes-on-windows-domain-controllers-ad2c23b0-15d8-4340-a468-4d4f3b188f16)
- [https://advisory.eventussecurity.com/advisory/critical-vulnerability-in-ad-cs-allows-privilege-escalation/](https://advisory.eventussecurity.com/advisory/critical-vulnerability-in-ad-cs-allows-privilege-escalation/)

{{#include ../../banners/hacktricks-training.md}}
