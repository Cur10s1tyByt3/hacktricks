# Ataques Físicos

{{#include ../banners/hacktricks-training.md}}

## Recuperación de Contraseña de BIOS y Seguridad del Sistema

**Restablecer el BIOS** se puede lograr de varias maneras. La mayoría de las placas base incluyen una **batería** que, al ser retirada durante aproximadamente **30 minutos**, restablecerá la configuración del BIOS, incluida la contraseña. Alternativamente, se puede ajustar un **puente en la placa base** para restablecer estas configuraciones conectando pines específicos.

Para situaciones en las que los ajustes de hardware no son posibles o prácticos, las **herramientas de software** ofrecen una solución. Ejecutar un sistema desde un **Live CD/USB** con distribuciones como **Kali Linux** proporciona acceso a herramientas como **_killCmos_** y **_CmosPWD_**, que pueden ayudar en la recuperación de la contraseña del BIOS.

En casos donde la contraseña del BIOS es desconocida, ingresarla incorrectamente **tres veces** generalmente resultará en un código de error. Este código se puede usar en sitios web como [https://bios-pw.org](https://bios-pw.org) para potencialmente recuperar una contraseña utilizable.

### Seguridad UEFI

Para sistemas modernos que utilizan **UEFI** en lugar del BIOS tradicional, se puede utilizar la herramienta **chipsec** para analizar y modificar configuraciones de UEFI, incluida la desactivación de **Secure Boot**. Esto se puede lograr con el siguiente comando:
```bash
python chipsec_main.py -module exploits.secure.boot.pk
```
---

## Análisis de RAM y Ataques de Arranque en Frío

La RAM retiene datos brevemente después de cortar la energía, generalmente durante **1 a 2 minutos**. Esta persistencia se puede extender a **10 minutos** aplicando sustancias frías, como nitrógeno líquido. Durante este período extendido, se puede crear un **volcado de memoria** utilizando herramientas como **dd.exe** y **volatility** para análisis.

---

## Ataques de Acceso Directo a la Memoria (DMA)

**INCEPTION** es una herramienta diseñada para **manipulación de memoria física** a través de DMA, compatible con interfaces como **FireWire** y **Thunderbolt**. Permite eludir procedimientos de inicio de sesión parcheando la memoria para aceptar cualquier contraseña. Sin embargo, es ineficaz contra sistemas de **Windows 10**.

---

## Live CD/USB para Acceso al Sistema

Cambiar binarios del sistema como **_sethc.exe_** o **_Utilman.exe_** con una copia de **_cmd.exe_** puede proporcionar un símbolo del sistema con privilegios de sistema. Herramientas como **chntpw** se pueden usar para editar el archivo **SAM** de una instalación de Windows, permitiendo cambios de contraseña.

**Kon-Boot** es una herramienta que facilita el inicio de sesión en sistemas Windows sin conocer la contraseña al modificar temporalmente el núcleo de Windows o UEFI. Más información se puede encontrar en [https://www.raymond.cc](https://www.raymond.cc/blog/login-to-windows-administrator-and-linux-root-account-without-knowing-or-changing-current-password/).

---

## Manejo de Características de Seguridad de Windows

### Atajos de Arranque y Recuperación

- **Supr**: Acceder a la configuración de BIOS.
- **F8**: Entrar en modo de recuperación.
- Presionar **Shift** después del banner de Windows puede eludir el inicio de sesión automático.

### Dispositivos BAD USB

Dispositivos como **Rubber Ducky** y **Teensyduino** sirven como plataformas para crear dispositivos **bad USB**, capaces de ejecutar cargas útiles predefinidas al conectarse a una computadora objetivo.

### Copia de Sombra de Volumen

Los privilegios de administrador permiten la creación de copias de archivos sensibles, incluido el archivo **SAM**, a través de PowerShell.

---

## Eludir la Encriptación de BitLocker

La encriptación de BitLocker puede ser eludida si se encuentra la **contraseña de recuperación** dentro de un archivo de volcado de memoria (**MEMORY.DMP**). Herramientas como **Elcomsoft Forensic Disk Decryptor** o **Passware Kit Forensic** se pueden utilizar para este propósito.

---

## Ingeniería Social para Adición de Clave de Recuperación

Se puede agregar una nueva clave de recuperación de BitLocker a través de tácticas de ingeniería social, convenciendo a un usuario para que ejecute un comando que añade una nueva clave de recuperación compuesta de ceros, simplificando así el proceso de descifrado.

---

## Explotación de Interruptores de Intrusión en Chasis / Mantenimiento para Restablecer el BIOS a Configuración de Fábrica

Muchos portátiles modernos y computadoras de escritorio de factor de forma pequeño incluyen un **interruptor de intrusión en chasis** que es monitoreado por el Controlador Embebido (EC) y el firmware BIOS/UEFI. Aunque el propósito principal del interruptor es generar una alerta cuando se abre un dispositivo, los proveedores a veces implementan un **atajo de recuperación no documentado** que se activa cuando el interruptor se alterna en un patrón específico.

### Cómo Funciona el Ataque

1. El interruptor está conectado a una **interrupción GPIO** en el EC.
2. El firmware que se ejecuta en el EC lleva un registro del **tiempo y número de pulsaciones**.
3. Cuando se reconoce un patrón codificado, el EC invoca una rutina de *reinicio de placa base* que **borra el contenido de la NVRAM/CMOS del sistema**.
4. En el siguiente arranque, el BIOS carga valores predeterminados – **la contraseña de supervisor, las claves de Arranque Seguro y toda la configuración personalizada se borran**.

> Una vez que el Arranque Seguro está deshabilitado y la contraseña del firmware ha desaparecido, el atacante puede simplemente iniciar cualquier imagen de SO externo y obtener acceso sin restricciones a las unidades internas.

### Ejemplo del Mundo Real – Portátil Framework 13

El atajo de recuperación para el Framework 13 (11ª/12ª/13ª generación) es:
```text
Press intrusion switch  →  hold 2 s
Release                 →  wait 2 s
(repeat the press/release cycle 10× while the machine is powered)
```
Después del décimo ciclo, el EC establece una bandera que instruye al BIOS para borrar NVRAM en el próximo reinicio. Todo el procedimiento toma ~40 s y requiere **nada más que un destornillador**.

### Procedimiento de Explotación Genérico

1. Encienda o suspenda-reanude el objetivo para que el EC esté en funcionamiento.
2. Retire la cubierta inferior para exponer el interruptor de intrusión/mantenimiento.
3. Reproduzca el patrón de conmutación específico del proveedor (consulte la documentación, foros o realice ingeniería inversa del firmware del EC).
4. Vuelva a ensamblar y reinicie: las protecciones del firmware deberían estar deshabilitadas.
5. Arranque un USB en vivo (por ejemplo, Kali Linux) y realice la explotación posterior habitual (volcado de credenciales, exfiltración de datos, implantación de binarios EFI maliciosos, etc.).

### Detección y Mitigación

* Registre eventos de intrusión en el chasis en la consola de gestión del SO y correlacione con reinicios inesperados del BIOS.
* Emplee **sellos a prueba de manipulaciones** en tornillos/cubiertas para detectar aperturas.
* Mantenga los dispositivos en **áreas físicamente controladas**; asuma que el acceso físico equivale a una completa compromisión.
* Donde sea posible, desactive la función de "reinicio del interruptor de mantenimiento" del proveedor o requiera una autorización criptográfica adicional para los reinicios de NVRAM.

---

## Referencias

- [Pentest Partners – “Framework 13. Press here to pwn”](https://www.pentestpartners.com/security-blog/framework-13-press-here-to-pwn/)
- [FrameWiki – Mainboard Reset Guide](https://framewiki.net/guides/mainboard-reset)

{{#include ../banners/hacktricks-training.md}}
