# Cheat Engine

{{#include ../../banners/hacktricks-training.md}}

[**Cheat Engine**](https://www.cheatengine.org/downloads.php) es un programa útil para encontrar dónde se guardan valores importantes dentro de la memoria de un juego en ejecución y cambiarlos.\
Cuando lo descargas y lo ejecutas, se te **presenta** un **tutorial** sobre cómo usar la herramienta. Si deseas aprender a usar la herramienta, se recomienda encarecidamente completarlo.

## ¿Qué estás buscando?

![](<../../images/image (762).png>)

Esta herramienta es muy útil para encontrar **dónde se almacena algún valor** (generalmente un número) **en la memoria** de un programa.\
**Generalmente, los números** se almacenan en forma de **4bytes**, pero también podrías encontrarlos en formatos **double** o **float**, o puede que desees buscar algo **diferente a un número**. Por esa razón, necesitas asegurarte de **seleccionar** lo que deseas **buscar**:

![](<../../images/image (324).png>)

También puedes indicar **diferentes** tipos de **búsquedas**:

![](<../../images/image (311).png>)

También puedes marcar la casilla para **detener el juego mientras escanea la memoria**:

![](<../../images/image (1052).png>)

### Teclas de acceso rápido

En _**Editar --> Configuración --> Teclas de acceso rápido**_ puedes establecer diferentes **teclas de acceso rápido** para diferentes propósitos, como **detener** el **juego** (lo cual es bastante útil si en algún momento deseas escanear la memoria). Otras opciones están disponibles:

![](<../../images/image (864).png>)

## Modificando el valor

Una vez que **encontraste** dónde está el **valor** que estás **buscando** (más sobre esto en los siguientes pasos), puedes **modificarlo** haciendo doble clic en él, luego haciendo doble clic en su valor:

![](<../../images/image (563).png>)

Y finalmente **marcando la casilla** para realizar la modificación en la memoria:

![](<../../images/image (385).png>)

El **cambio** en la **memoria** se aplicará inmediatamente (ten en cuenta que hasta que el juego no use este valor nuevamente, el valor **no se actualizará en el juego**).

## Buscando el valor

Entonces, vamos a suponer que hay un valor importante (como la vida de tu usuario) que deseas mejorar, y estás buscando este valor en la memoria.

### A través de un cambio conocido

Suponiendo que estás buscando el valor 100, **realizas un escaneo** buscando ese valor y encuentras muchas coincidencias:

![](<../../images/image (108).png>)

Luego, haces algo para que **el valor cambie**, y **detienes** el juego y **realizas** un **siguiente escaneo**:

![](<../../images/image (684).png>)

Cheat Engine buscará los **valores** que **pasaron de 100 al nuevo valor**. Felicitaciones, **encontraste** la **dirección** del valor que estabas buscando, ahora puedes modificarlo.\
_Si aún tienes varios valores, haz algo para modificar nuevamente ese valor y realiza otro "siguiente escaneo" para filtrar las direcciones._

### Valor desconocido, cambio conocido

En el escenario en que **no conoces el valor** pero sabes **cómo hacerlo cambiar** (e incluso el valor del cambio), puedes buscar tu número.

Así que, comienza realizando un escaneo de tipo "**Valor inicial desconocido**":

![](<../../images/image (890).png>)

Luego, haz que el valor cambie, indica **cómo** el **valor** **cambió** (en mi caso, disminuyó en 1) y realiza un **siguiente escaneo**:

![](<../../images/image (371).png>)

Se te presentarán **todos los valores que fueron modificados de la manera seleccionada**:

![](<../../images/image (569).png>)

Una vez que hayas encontrado tu valor, puedes modificarlo.

Ten en cuenta que hay **muchos cambios posibles** y puedes hacer estos **pasos tantas veces como desees** para filtrar los resultados:

![](<../../images/image (574).png>)

### Dirección de memoria aleatoria - Encontrando el código

Hasta ahora hemos aprendido cómo encontrar una dirección que almacena un valor, pero es muy probable que en **diferentes ejecuciones del juego esa dirección esté en diferentes lugares de la memoria**. Así que vamos a descubrir cómo encontrar siempre esa dirección.

Usando algunos de los trucos mencionados, encuentra la dirección donde tu juego actual está almacenando el valor importante. Luego (deteniendo el juego si lo deseas), haz clic derecho en la **dirección** encontrada y selecciona "**Descubrir qué accede a esta dirección**" o "**Descubrir qué escribe en esta dirección**":

![](<../../images/image (1067).png>)

La **primera opción** es útil para saber qué **partes** del **código** están **usando** esta **dirección** (lo cual es útil para más cosas como **saber dónde puedes modificar el código** del juego).\
La **segunda opción** es más **específica**, y será más útil en este caso ya que estamos interesados en saber **desde dónde se está escribiendo este valor**.

Una vez que hayas seleccionado una de esas opciones, el **depurador** se **adjuntará** al programa y aparecerá una nueva **ventana vacía**. Ahora, **juega** el **juego** y **modifica** ese **valor** (sin reiniciar el juego). La **ventana** debería **llenarse** con las **direcciones** que están **modificando** el **valor**:

![](<../../images/image (91).png>)

Ahora que encontraste la dirección que está modificando el valor, puedes **modificar el código a tu antojo** (Cheat Engine te permite modificarlo rápidamente a NOPs):

![](<../../images/image (1057).png>)

Así que, ahora puedes modificarlo para que el código no afecte tu número, o siempre afecte de manera positiva.

### Dirección de memoria aleatoria - Encontrando el puntero

Siguiendo los pasos anteriores, encuentra dónde está el valor que te interesa. Luego, usando "**Descubrir qué escribe en esta dirección**", descubre qué dirección escribe este valor y haz doble clic en ella para obtener la vista de desensamblado:

![](<../../images/image (1039).png>)

Luego, realiza un nuevo escaneo **buscando el valor hex entre "\[]"** (el valor de $edx en este caso):

![](<../../images/image (994).png>)

(_Si aparecen varios, generalmente necesitas la dirección más pequeña_)\
Ahora, hemos **encontrado el puntero que modificará el valor que nos interesa**.

Haz clic en "**Agregar dirección manualmente**":

![](<../../images/image (990).png>)

Ahora, marca la casilla "Puntero" y agrega la dirección encontrada en el cuadro de texto (en este escenario, la dirección encontrada en la imagen anterior fue "Tutorial-i386.exe"+2426B0):

![](<../../images/image (392).png>)

(Nota cómo la primera "Dirección" se completa automáticamente a partir de la dirección del puntero que introduces)

Haz clic en Aceptar y se creará un nuevo puntero:

![](<../../images/image (308).png>)

Ahora, cada vez que modifiques ese valor, estarás **modificando el valor importante incluso si la dirección de memoria donde se encuentra el valor es diferente.**

### Inyección de código

La inyección de código es una técnica donde inyectas un fragmento de código en el proceso objetivo, y luego rediriges la ejecución del código para que pase por tu propio código escrito (como darte puntos en lugar de restarlos).

Así que, imagina que has encontrado la dirección que está restando 1 a la vida de tu jugador:

![](<../../images/image (203).png>)

Haz clic en Mostrar desensamblador para obtener el **código desensamblado**.\
Luego, haz clic en **CTRL+a** para invocar la ventana de Auto ensamblado y selecciona _**Plantilla --> Inyección de código**_

![](<../../images/image (902).png>)

Completa la **dirección de la instrucción que deseas modificar** (esto generalmente se completa automáticamente):

![](<../../images/image (744).png>)

Se generará una plantilla:

![](<../../images/image (944).png>)

Así que, inserta tu nuevo código ensamblador en la sección "**newmem**" y elimina el código original de "**originalcode**" si no deseas que se ejecute. En este ejemplo, el código inyectado sumará 2 puntos en lugar de restar 1:

![](<../../images/image (521).png>)

**Haz clic en ejecutar y así tu código debería ser inyectado en el programa cambiando el comportamiento de la funcionalidad.**

## Características avanzadas en Cheat Engine 7.x (2023-2025)

Cheat Engine ha seguido evolucionando desde la versión 7.0 y se han añadido varias características de calidad de vida y *ofensivas de reversión* que son extremadamente útiles al analizar software moderno (¡y no solo juegos!). A continuación se presenta una **guía de campo muy condensada** sobre las adiciones que probablemente usarás durante el trabajo de red team/CTF.

### Mejoras en el escáner de punteros 2
* `Los punteros deben terminar con desplazamientos específicos` y el nuevo control deslizante de **Desviación** (≥7.4) reduce en gran medida los falsos positivos cuando vuelves a escanear después de una actualización. Úsalo junto con la comparación de mapas múltiples (`.PTR` → *Comparar resultados con otro mapa de punteros guardado*) para obtener un **único puntero base resistente** en solo unos minutos.
* Atajo de filtro masivo: después del primer escaneo, presiona `Ctrl+A → Espacio` para marcar todo, luego `Ctrl+I` (invertir) para deseleccionar direcciones que fallaron en el reescaneo.

### Ultimap 3 – Trazado de Intel PT
*Desde 7.5, el antiguo Ultimap fue reimplementado sobre **Intel Processor-Trace (IPT)**. Esto significa que ahora puedes grabar *cada* rama que toma el objetivo **sin pasos individuales** (solo en modo usuario, no activará la mayoría de los gadgets anti-depuración).
```
Memory View → Tools → Ultimap 3 → check «Intel PT»
Select number of buffers → Start
```
Después de unos segundos, detén la captura y **haz clic derecho → Guardar lista de ejecución en archivo**. Combina las direcciones de las ramas con una sesión de `Find out what addresses this instruction accesses` para localizar rápidamente los puntos críticos de la lógica del juego de alta frecuencia.

### Plantillas de `jmp` de 1 byte / auto-patch
La versión 7.5 introdujo un *stub JMP de un byte* (0xEB) que instala un manejador SEH y coloca un INT3 en la ubicación original. Se genera automáticamente cuando usas **Auto Assembler → Template → Code Injection** en instrucciones que no se pueden parchear con un salto relativo de 5 bytes. Esto hace posibles los "hooks" "ajustados" dentro de rutinas empaquetadas o con restricciones de tamaño.

### Sigilo a nivel de kernel con DBVM (AMD e Intel)
*DBVM* es el hipervisor de Tipo-2 integrado de CE. Las versiones recientes finalmente añadieron **soporte para AMD-V/SVM**, por lo que puedes ejecutar `Driver → Load DBVM` en hosts Ryzen/EPYC. DBVM te permite:
1. Crear puntos de interrupción de hardware invisibles para las comprobaciones de Ring-3/anti-debug.
2. Leer/escribir regiones de memoria del kernel paginables o protegidas incluso cuando el controlador en modo usuario está deshabilitado.
3. Realizar bypass de ataques de temporización sin VM-EXIT (por ejemplo, consultar `rdtsc` desde el hipervisor).

**Consejo:** DBVM se negará a cargar cuando HVCI/Memory-Integrity esté habilitado en Windows 11 → desactívalo o inicia un host de VM dedicado.

### Depuración remota / multiplataforma con **ceserver**
CE ahora incluye una reescritura completa de *ceserver* y puede conectarse a través de TCP a **Linux, Android, macOS e iOS**. Un fork popular integra *Frida* para combinar la instrumentación dinámica con la GUI de CE, ideal cuando necesitas parchear juegos de Unity o Unreal que se ejecutan en un teléfono:
```
# on the target (arm64)
./ceserver_arm64 &
# on the analyst workstation
adb forward tcp:52736 tcp:52736   # (or ssh tunnel)
Cheat Engine → "Network" icon → Host = localhost → Connect
```
Para el puente Frida, consulta `bb33bb/frida-ceserver` en GitHub.

### Otras herramientas notables
* **Patch Scanner** (MemView → Tools) – detecta cambios de código inesperados en secciones ejecutables; útil para análisis de malware.
* **Structure Dissector 2** – arrastra una dirección → `Ctrl+D`, luego *Guess fields* para evaluar automáticamente estructuras C.
* **.NET & Mono Dissector** – soporte mejorado para juegos de Unity; llama a métodos directamente desde la consola Lua de CE.
* **Tipos personalizados Big-Endian** – escaneo/edición de orden de bytes revertido (útil para emuladores de consola y búferes de paquetes de red).
* **Autosave & tabs** para ventanas de AutoAssembler/Lua, además de `reassemble()` para reescritura de instrucciones en múltiples líneas.

### Notas de instalación y OPSEC (2024-2025)
* El instalador oficial está envuelto con InnoSetup **ofertas publicitarias** (`RAV`, etc.). **Siempre haz clic en *Decline*** *o compila desde la fuente* para evitar PUPs. Los AVs seguirán marcando `cheatengine.exe` como *HackTool*, lo cual es esperado.
* Los controladores anti-trampa modernos (EAC/Battleye, ACE-BASE.sys, mhyprot2.sys) detectan la clase de ventana de CE incluso cuando se renombra. Ejecuta tu copia de reversión **dentro de una VM desechable** o después de deshabilitar el juego en red.
* Si solo necesitas acceso en modo usuario, elige **`Settings → Extra → Kernel mode debug = off`** para evitar cargar el controlador no firmado de CE que puede causar BSOD en Windows 11 24H2 Secure-Boot.

---

## **Referencias**

- [Cheat Engine 7.5 release notes (GitHub)](https://github.com/cheat-engine/cheat-engine/releases/tag/7.5)
- [frida-ceserver cross-platform bridge](https://github.com/bb33bb/frida-ceserver-Mac-and-IOS)
- **Tutorial de Cheat Engine, complétalo para aprender cómo empezar con Cheat Engine**

{{#include ../../banners/hacktricks-training.md}}
