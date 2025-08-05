# Time Namespace

{{#include ../../../../banners/hacktricks-training.md}}

## Información Básica

El espacio de nombres de tiempo en Linux permite desplazamientos por espacio de nombres a los relojes monótonos del sistema y al tiempo de arranque. Se utiliza comúnmente en contenedores de Linux para cambiar la fecha/hora dentro de un contenedor y ajustar los relojes después de restaurar desde un punto de control o instantánea.

## Laboratorio:

### Crear diferentes Espacios de Nombres

#### CLI
```bash
sudo unshare -T [--mount-proc] /bin/bash
```
Al montar una nueva instancia del sistema de archivos `/proc` si usas el parámetro `--mount-proc`, aseguras que el nuevo espacio de montaje tenga una **vista precisa y aislada de la información del proceso específica de ese espacio de nombres**.

<details>

<summary>Error: bash: fork: Cannot allocate memory</summary>

Cuando se ejecuta `unshare` sin la opción `-f`, se encuentra un error debido a la forma en que Linux maneja los nuevos espacios de nombres de PID (Identificación de Proceso). Los detalles clave y la solución se describen a continuación:

1. **Explicación del Problema**:

- El núcleo de Linux permite a un proceso crear nuevos espacios de nombres utilizando la llamada al sistema `unshare`. Sin embargo, el proceso que inicia la creación de un nuevo espacio de nombres de PID (denominado "proceso unshare") no entra en el nuevo espacio de nombres; solo lo hacen sus procesos hijos.
- Ejecutar `%unshare -p /bin/bash%` inicia `/bin/bash` en el mismo proceso que `unshare`. En consecuencia, `/bin/bash` y sus procesos hijos están en el espacio de nombres de PID original.
- El primer proceso hijo de `/bin/bash` en el nuevo espacio de nombres se convierte en PID 1. Cuando este proceso sale, desencadena la limpieza del espacio de nombres si no hay otros procesos, ya que PID 1 tiene el papel especial de adoptar procesos huérfanos. El núcleo de Linux deshabilitará entonces la asignación de PID en ese espacio de nombres.

2. **Consecuencia**:

- La salida de PID 1 en un nuevo espacio de nombres lleva a la limpieza de la bandera `PIDNS_HASH_ADDING`. Esto resulta en que la función `alloc_pid` falle al intentar asignar un nuevo PID al crear un nuevo proceso, produciendo el error "Cannot allocate memory".

3. **Solución**:
- El problema se puede resolver utilizando la opción `-f` con `unshare`. Esta opción hace que `unshare` cree un nuevo proceso después de crear el nuevo espacio de nombres de PID.
- Ejecutar `%unshare -fp /bin/bash%` asegura que el comando `unshare` se convierta en PID 1 en el nuevo espacio de nombres. `/bin/bash` y sus procesos hijos están entonces contenidos de manera segura dentro de este nuevo espacio de nombres, previniendo la salida prematura de PID 1 y permitiendo la asignación normal de PID.

Al asegurarte de que `unshare` se ejecute con la bandera `-f`, el nuevo espacio de nombres de PID se mantiene correctamente, permitiendo que `/bin/bash` y sus subprocesos operen sin encontrar el error de asignación de memoria.

</details>

#### Docker
```bash
docker run -ti --name ubuntu1 -v /usr:/ubuntu1 ubuntu bash
```
### Verifica en qué namespace está tu proceso
```bash
ls -l /proc/self/ns/time
lrwxrwxrwx 1 root root 0 Apr  4 21:16 /proc/self/ns/time -> 'time:[4026531834]'
```
### Encontrar todos los espacios de nombres de tiempo
```bash
sudo find /proc -maxdepth 3 -type l -name time -exec readlink {} \; 2>/dev/null | sort -u
# Find the processes with an specific namespace
sudo find /proc -maxdepth 3 -type l -name time -exec ls -l  {} \; 2>/dev/null | grep <ns-number>
```
### Entrar dentro de un espacio de nombres de tiempo
```bash
nsenter -T TARGET_PID --pid /bin/bash
```
## Manipulación de Desplazamientos de Tiempo

A partir de Linux 5.6, se pueden virtualizar dos relojes por espacio de nombres de tiempo:

* `CLOCK_MONOTONIC`
* `CLOCK_BOOTTIME`

Sus deltas por espacio de nombres se exponen (y pueden ser modificados) a través del archivo `/proc/<PID>/timens_offsets`:
```
$ sudo unshare -Tr --mount-proc bash   # -T creates a new timens, -r drops capabilities
$ cat /proc/$$/timens_offsets
monotonic 0
boottime  0
```
El archivo contiene dos líneas, una por reloj, con el desplazamiento en **nanosegundos**. Los procesos que tienen **CAP_SYS_TIME** _en el espacio de nombres de tiempo_ pueden cambiar el valor:
```
# advance CLOCK_MONOTONIC by two days (172 800 s)
echo "monotonic 172800000000000" > /proc/$$/timens_offsets
# verify
$ cat /proc/$$/uptime   # first column uses CLOCK_MONOTONIC
172801.37  13.57
```
Si también necesitas que el reloj de pared (`CLOCK_REALTIME`) cambie, aún tienes que depender de mecanismos clásicos (`date`, `hwclock`, `chronyd`, …); **no** está en un espacio de nombres.

### `unshare(1)` flags de ayuda (util-linux ≥ 2.38)
```
sudo unshare -T \
--monotonic="+24h"  \
--boottime="+7d"    \
--mount-proc         \
bash
```
Las opciones largas escriben automáticamente los deltas elegidos en `timens_offsets` justo después de que se crea el namespace, ahorrando un `echo` manual.

---

## Soporte de OCI y Runtime

* La **Especificación de Runtime de OCI v1.1** (Nov 2023) agregó un tipo de namespace `time` dedicado y el campo `linux.timeOffsets` para que los motores de contenedores puedan solicitar virtualización de tiempo de manera portátil.
* **runc >= 1.2.0** implementa esa parte de la especificación. Un fragmento mínimo de `config.json` se ve así:
```json
{
"linux": {
"namespaces": [
{"type": "time"}
],
"timeOffsets": {
"monotonic": 86400,
"boottime": 600
}
}
}
```
Luego ejecuta el contenedor con `runc run <id>`.

>  NOTA: runc **1.2.6** (Feb 2025) corrigió un error de "exec en contenedor con timens privado" que podría llevar a un bloqueo y potencial DoS. Asegúrate de estar en ≥ 1.2.6 en producción.

---

## Consideraciones de seguridad

1. **Capacidad requerida** – Un proceso necesita **CAP_SYS_TIME** dentro de su namespace de usuario/tiempo para cambiar los offsets. Eliminar esa capacidad en el contenedor (predeterminado en Docker y Kubernetes) previene manipulaciones.
2. **Sin cambios en el reloj de pared** – Debido a que `CLOCK_REALTIME` se comparte con el host, los atacantes no pueden falsificar la duración de los certificados, la expiración de JWT, etc. solo a través de timens.
3. **Evasión de registro/detección** – El software que depende de `CLOCK_MONOTONIC` (por ejemplo, limitadores de tasa basados en tiempo de actividad) puede confundirse si el usuario del namespace ajusta el offset. Prefiere `CLOCK_REALTIME` para marcas de tiempo relevantes para la seguridad.
4. **Superficie de ataque del kernel** – Incluso con `CAP_SYS_TIME` eliminado, el código del kernel sigue siendo accesible; mantén el host actualizado. Linux 5.6 → 5.12 recibió múltiples correcciones de errores de timens (NULL-deref, problemas de signo).

### Lista de verificación de endurecimiento

* Elimina `CAP_SYS_TIME` en el perfil predeterminado de tu runtime de contenedor.
* Mantén los runtimes actualizados (runc ≥ 1.2.6, crun ≥ 1.12).
* Fija util-linux ≥ 2.38 si dependes de los ayudantes `--monotonic/--boottime`.
* Audita el software en contenedor que lee **uptime** o **CLOCK_MONOTONIC** para lógica crítica de seguridad.

## Referencias

* man7.org – Página del manual de namespaces de tiempo: <https://man7.org/linux/man-pages/man7/time_namespaces.7.html>
* Blog de OCI – "OCI v1.1: nuevos namespaces de tiempo y RDT" (15 de Nov 2023): <https://opencontainers.org/blog/2023/11/15/oci-spec-v1.1>

{{#include ../../../../banners/hacktricks-training.md}}
