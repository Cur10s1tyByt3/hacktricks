# Inyección de Hilo en macOS a través del puerto de Tarea

{{#include ../../../../banners/hacktricks-training.md}}

## Código

- [https://github.com/bazad/threadexec](https://github.com/bazad/threadexec)
- [https://gist.github.com/knightsc/bd6dfeccb02b77eb6409db5601dcef36](https://gist.github.com/knightsc/bd6dfeccb02b77eb6409db5601dcef36)

## 1. Secuestro de Hilo

Inicialmente, se invoca la función `task_threads()` en el puerto de tarea para obtener una lista de hilos de la tarea remota. Se selecciona un hilo para el secuestro. Este enfoque se desvía de los métodos convencionales de inyección de código, ya que crear un nuevo hilo remoto está prohibido debido a la mitigación que bloquea `thread_create_running()`.

Para controlar el hilo, se llama a `thread_suspend()`, deteniendo su ejecución.

Las únicas operaciones permitidas en el hilo remoto implican **detener** y **comenzar** su ejecución y **recuperar**/**modificar** los valores de sus registros. Las llamadas a funciones remotas se inician configurando los registros `x0` a `x7` con los **argumentos**, configurando `pc` para apuntar a la función deseada y reanudando el hilo. Asegurarse de que el hilo no se bloquee después de la devolución requiere detectar la devolución.

Una estrategia implica registrar un **manejador de excepciones** para el hilo remoto utilizando `thread_set_exception_ports()`, configurando el registro `lr` a una dirección inválida antes de la llamada a la función. Esto desencadena una excepción después de la ejecución de la función, enviando un mensaje al puerto de excepciones, lo que permite la inspección del estado del hilo para recuperar el valor de retorno. Alternativamente, como se adoptó del exploit *triple_fetch* de Ian Beer, `lr` se establece para que se ejecute en un bucle infinito; los registros del hilo se monitorean continuamente hasta que `pc` apunta a esa instrucción.

## 2. Puertos Mach para comunicación

La fase siguiente implica establecer puertos Mach para facilitar la comunicación con el hilo remoto. Estos puertos son fundamentales para transferir derechos de envío/recepción arbitrarios entre tareas.

Para la comunicación bidireccional, se crean dos derechos de recepción Mach: uno en la tarea local y el otro en la tarea remota. Posteriormente, se transfiere un derecho de envío para cada puerto a la tarea contraria, permitiendo el intercambio de mensajes.

Centrando en el puerto local, el derecho de recepción es mantenido por la tarea local. El puerto se crea con `mach_port_allocate()`. El desafío radica en transferir un derecho de envío a este puerto en la tarea remota.

Una estrategia implica aprovechar `thread_set_special_port()` para colocar un derecho de envío al puerto local en el `THREAD_KERNEL_PORT` del hilo remoto. Luego, se instruye al hilo remoto para que llame a `mach_thread_self()` para recuperar el derecho de envío.

Para el puerto remoto, el proceso se invierte esencialmente. Se dirige al hilo remoto a generar un puerto Mach a través de `mach_reply_port()` (ya que `mach_port_allocate()` no es adecuado debido a su mecanismo de retorno). Tras la creación del puerto, se invoca `mach_port_insert_right()` en el hilo remoto para establecer un derecho de envío. Este derecho se almacena en el kernel utilizando `thread_set_special_port()`. De vuelta en la tarea local, se utiliza `thread_get_special_port()` en el hilo remoto para adquirir un derecho de envío al nuevo puerto Mach asignado en la tarea remota.

La finalización de estos pasos resulta en el establecimiento de puertos Mach, sentando las bases para la comunicación bidireccional.

## 3. Primitivas Básicas de Lectura/Escritura de Memoria

En esta sección, el enfoque está en utilizar la primitiva de ejecución para establecer primitivas básicas de lectura/escritura de memoria. Estos pasos iniciales son cruciales para obtener más control sobre el proceso remoto, aunque las primitivas en esta etapa no servirán para muchos propósitos. Pronto, se actualizarán a versiones más avanzadas.

### Lectura y escritura de memoria utilizando la primitiva de ejecución

El objetivo es realizar lecturas y escrituras de memoria utilizando funciones específicas. Para **leer memoria**:
```c
uint64_t read_func(uint64_t *address) {
return *address;
}
```
Para **escribir en memoria**:
```c
void write_func(uint64_t *address, uint64_t value) {
*address = value;
}
```
Estas funciones corresponden al siguiente ensamblaje:
```
_read_func:
ldr x0, [x0]
ret
_write_func:
str x1, [x0]
ret
```
### Identificación de funciones adecuadas

Un escaneo de bibliotecas comunes reveló candidatos apropiados para estas operaciones:

1. **Lectura de memoria — `property_getName()`** (libobjc):
```c
const char *property_getName(objc_property_t prop) {
return prop->name;
}
```
2. **Escritura de memoria — `_xpc_int64_set_value()`** (libxpc):
```c
__xpc_int64_set_value:
str x1, [x0, #0x18]
ret
```
Para realizar una escritura de 64 bits en una dirección arbitraria:
```c
_xpc_int64_set_value(address - 0x18, value);
```
Con estas primitivas establecidas, se sienta la base para crear memoria compartida, marcando un progreso significativo en el control del proceso remoto.

## 4. Configuración de Memoria Compartida

El objetivo es establecer memoria compartida entre tareas locales y remotas, simplificando la transferencia de datos y facilitando la llamada a funciones con múltiples argumentos. El enfoque aprovecha `libxpc` y su tipo de objeto `OS_xpc_shmem`, que se basa en entradas de memoria Mach.

### Resumen del Proceso

1. **Asignación de memoria**
* Asigne memoria para compartir usando `mach_vm_allocate()`.
* Use `xpc_shmem_create()` para crear un objeto `OS_xpc_shmem` para la región asignada.
2. **Creación de memoria compartida en el proceso remoto**
* Asigne memoria para el objeto `OS_xpc_shmem` en el proceso remoto (`remote_malloc`).
* Copie el objeto de plantilla local; se requiere una corrección del derecho de envío Mach incrustado en el desplazamiento `0x18`.
3. **Corrección de la entrada de memoria Mach**
* Inserte un derecho de envío con `thread_set_special_port()` y sobrescriba el campo `0x18` con el nombre de la entrada remota.
4. **Finalización**
* Valide el objeto remoto y mapeelo con una llamada remota a `xpc_shmem_remote()`.

## 5. Logrando Control Total

Una vez que la ejecución arbitraria y un canal de retroalimentación de memoria compartida están disponibles, efectivamente posee el proceso objetivo:

* **R/W de memoria arbitraria** — use `memcpy()` entre regiones locales y compartidas.
* **Llamadas a funciones con > 8 args** — coloque los argumentos adicionales en la pila siguiendo la convención de llamada arm64.
* **Transferencia de puerto Mach** — pase derechos en mensajes Mach a través de los puertos establecidos.
* **Transferencia de descriptores de archivo** — aproveche fileports (ver *triple_fetch*).

Todo esto está envuelto en la biblioteca [`threadexec`](https://github.com/bazad/threadexec) para fácil reutilización.

---

## 6. Matices de Apple Silicon (arm64e)

En dispositivos Apple Silicon (arm64e), los **Códigos de Autenticación de Punteros (PAC)** protegen todas las direcciones de retorno y muchos punteros de función. Las técnicas de secuestro de hilos que *reutilizan código existente* continúan funcionando porque los valores originales en `lr`/`pc` ya llevan firmas PAC válidas. Los problemas surgen cuando intentas saltar a memoria controlada por el atacante:

1. Asigne memoria ejecutable dentro del objetivo (remoto `mach_vm_allocate` + `mprotect(PROT_EXEC)`).
2. Copie su carga útil.
3. Dentro del proceso *remoto*, firme el puntero:
```c
uint64_t ptr = (uint64_t)payload;
ptr = ptrauth_sign_unauthenticated((void*)ptr, ptrauth_key_asia, 0);
```
4. Establecer `pc = ptr` en el estado del hilo secuestrado.

Alternativamente, mantenga la conformidad con PAC encadenando gadgets/funciones existentes (ROP tradicional).

## 7. Detección y Fortalecimiento con EndpointSecurity

El marco **EndpointSecurity (ES)** expone eventos del kernel que permiten a los defensores observar o bloquear intentos de inyección de hilos:

* `ES_EVENT_TYPE_AUTH_GET_TASK` – se activa cuando un proceso solicita el puerto de otra tarea (por ejemplo, `task_for_pid()`).
* `ES_EVENT_TYPE_NOTIFY_REMOTE_THREAD_CREATE` – se emite cada vez que se crea un hilo en una tarea *diferente*.
* `ES_EVENT_TYPE_NOTIFY_THREAD_SET_STATE` (agregado en macOS 14 Sonoma) – indica la manipulación de registros de un hilo existente.

Cliente Swift mínimo que imprime eventos de hilos remotos:
```swift
import EndpointSecurity

let client = try! ESClient(subscriptions: [.notifyRemoteThreadCreate]) {
(_, msg) in
if let evt = msg.remoteThreadCreate {
print("[ALERT] remote thread in pid \(evt.target.pid) by pid \(evt.thread.pid)")
}
}
RunLoop.main.run()
```
Consultando con **osquery** ≥ 5.8:
```sql
SELECT target_pid, source_pid, target_path
FROM es_process_events
WHERE event_type = 'REMOTE_THREAD_CREATE';
```
### Consideraciones sobre el runtime endurecido

Distribuir tu aplicación **sin** el derecho `com.apple.security.get-task-allow` impide que atacantes no root obtengan su puerto de tarea. La Protección de Integridad del Sistema (SIP) aún bloquea el acceso a muchos binarios de Apple, pero el software de terceros debe optar por salir explícitamente.

## 8. Herramientas Públicas Recientes (2023-2025)

| Herramienta | Año | Observaciones |
|-------------|-----|---------------|
| [`task_vaccine`](https://github.com/rodionovd/task_vaccine) | 2023 | PoC compacta que demuestra el secuestro de hilos consciente de PAC en Ventura/Sonoma |
| `remote_thread_es` | 2024 | Ayudante de EndpointSecurity utilizado por varios proveedores de EDR para mostrar eventos de `REMOTE_THREAD_CREATE` |

> Leer el código fuente de estos proyectos es útil para entender los cambios en la API introducidos en macOS 13/14 y para mantenerse compatible entre Intel ↔ Apple Silicon.

## Referencias

- [https://bazad.github.io/2018/10/bypassing-platform-binary-task-threads/](https://bazad.github.io/2018/10/bypassing-platform-binary-task-threads/)
- [https://github.com/rodionovd/task_vaccine](https://github.com/rodionovd/task_vaccine)
- [https://developer.apple.com/documentation/endpointsecurity/es_event_type_notify_remote_thread_create](https://developer.apple.com/documentation/endpointsecurity/es_event_type_notify_remote_thread_create)

{{#include ../../../../banners/hacktricks-training.md}}
