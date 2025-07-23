# macOS Injection de Thread via le port de tâche

{{#include ../../../../banners/hacktricks-training.md}}

## Code

- [https://github.com/bazad/threadexec](https://github.com/bazad/threadexec)
- [https://gist.github.com/knightsc/bd6dfeccb02b77eb6409db5601dcef36](https://gist.github.com/knightsc/bd6dfeccb02b77eb6409db5601dcef36)

## 1. Détournement de Thread

Initialement, la fonction `task_threads()` est invoquée sur le port de tâche pour obtenir une liste de threads de la tâche distante. Un thread est sélectionné pour le détournement. Cette approche diverge des méthodes conventionnelles d'injection de code, car la création d'un nouveau thread distant est interdite en raison de l'atténuation qui bloque `thread_create_running()`.

Pour contrôler le thread, `thread_suspend()` est appelé, arrêtant son exécution.

Les seules opérations autorisées sur le thread distant impliquent **l'arrêt** et **le démarrage** de celui-ci et **la récupération**/**la modification** de ses valeurs de registre. Les appels de fonction distants sont initiés en définissant les registres `x0` à `x7` sur les **arguments**, configurant `pc` pour cibler la fonction désirée, et en reprenant le thread. S'assurer que le thread ne plante pas après le retour nécessite de détecter le retour.

Une stratégie consiste à enregistrer un **gestionnaire d'exception** pour le thread distant en utilisant `thread_set_exception_ports()`, en définissant le registre `lr` sur une adresse invalide avant l'appel de fonction. Cela déclenche une exception après l'exécution de la fonction, envoyant un message au port d'exception, permettant l'inspection de l'état du thread pour récupérer la valeur de retour. Alternativement, comme adopté de l'exploit *triple_fetch* d'Ian Beer, `lr` est défini pour boucler indéfiniment ; les registres du thread sont ensuite surveillés en continu jusqu'à ce que `pc` pointe vers cette instruction.

## 2. Ports Mach pour la communication

La phase suivante consiste à établir des ports Mach pour faciliter la communication avec le thread distant. Ces ports sont essentiels pour transférer des droits d'envoi/réception arbitraires entre les tâches.

Pour une communication bidirectionnelle, deux droits de réception Mach sont créés : un dans la tâche locale et l'autre dans la tâche distante. Ensuite, un droit d'envoi pour chaque port est transféré à la tâche correspondante, permettant l'échange de messages.

En se concentrant sur le port local, le droit de réception est détenu par la tâche locale. Le port est créé avec `mach_port_allocate()`. Le défi réside dans le transfert d'un droit d'envoi vers ce port dans la tâche distante.

Une stratégie consiste à tirer parti de `thread_set_special_port()` pour placer un droit d'envoi vers le port local dans le `THREAD_KERNEL_PORT` du thread distant. Ensuite, le thread distant est instruit d'appeler `mach_thread_self()` pour récupérer le droit d'envoi.

Pour le port distant, le processus est essentiellement inversé. Le thread distant est dirigé pour générer un port Mach via `mach_reply_port()` (car `mach_port_allocate()` n'est pas adapté en raison de son mécanisme de retour). Une fois le port créé, `mach_port_insert_right()` est invoqué dans le thread distant pour établir un droit d'envoi. Ce droit est ensuite stocké dans le noyau en utilisant `thread_set_special_port()`. De retour dans la tâche locale, `thread_get_special_port()` est utilisé sur le thread distant pour acquérir un droit d'envoi vers le nouveau port Mach alloué dans la tâche distante.

L'achèvement de ces étapes aboutit à l'établissement de ports Mach, posant les bases d'une communication bidirectionnelle.

## 3. Primitives de Lecture/Écriture Mémoire de Base

Dans cette section, l'accent est mis sur l'utilisation de la primitive d'exécution pour établir des primitives de lecture/écriture mémoire de base. Ces étapes initiales sont cruciales pour obtenir plus de contrôle sur le processus distant, bien que les primitives à ce stade ne serviront pas à beaucoup de choses. Bientôt, elles seront mises à niveau vers des versions plus avancées.

### Lecture et écriture de mémoire en utilisant la primitive d'exécution

L'objectif est d'effectuer des lectures et des écritures de mémoire en utilisant des fonctions spécifiques. Pour **lire la mémoire** :
```c
uint64_t read_func(uint64_t *address) {
return *address;
}
```
Pour **écrire en mémoire** :
```c
void write_func(uint64_t *address, uint64_t value) {
*address = value;
}
```
Ces fonctions correspondent à l'assemblage suivant :
```
_read_func:
ldr x0, [x0]
ret
_write_func:
str x1, [x0]
ret
```
### Identification des fonctions appropriées

Un scan des bibliothèques courantes a révélé des candidats appropriés pour ces opérations :

1. **Lecture de la mémoire — `property_getName()`** (libobjc):
```c
const char *property_getName(objc_property_t prop) {
return prop->name;
}
```
2. **Écriture en mémoire — `_xpc_int64_set_value()`** (libxpc):
```c
__xpc_int64_set_value:
str x1, [x0, #0x18]
ret
```
Pour effectuer une écriture 64 bits à une adresse arbitraire :
```c
_xpc_int64_set_value(address - 0x18, value);
```
Avec ces primitives établies, le terrain est préparé pour créer de la mémoire partagée, marquant une progression significative dans le contrôle du processus distant.

## 4. Configuration de la mémoire partagée

L'objectif est d'établir une mémoire partagée entre les tâches locales et distantes, simplifiant le transfert de données et facilitant l'appel de fonctions avec plusieurs arguments. L'approche s'appuie sur `libxpc` et son type d'objet `OS_xpc_shmem`, qui est construit sur des entrées de mémoire Mach.

### Aperçu du processus

1. **Allocation de mémoire**
* Allouer de la mémoire pour le partage en utilisant `mach_vm_allocate()`.
* Utiliser `xpc_shmem_create()` pour créer un objet `OS_xpc_shmem` pour la région allouée.
2. **Création de la mémoire partagée dans le processus distant**
* Allouer de la mémoire pour l'objet `OS_xpc_shmem` dans le processus distant (`remote_malloc`).
* Copier l'objet modèle local ; un ajustement du droit d'envoi Mach intégré à l'offset `0x18` est toujours nécessaire.
3. **Correction de l'entrée de mémoire Mach**
* Insérer un droit d'envoi avec `thread_set_special_port()` et écraser le champ `0x18` avec le nom de l'entrée distante.
4. **Finalisation**
* Valider l'objet distant et le mapper avec un appel distant à `xpc_shmem_remote()`.

## 5. Obtenir un contrôle total

Une fois l'exécution arbitraire et un canal de retour en mémoire partagée disponibles, vous possédez effectivement le processus cible :

* **R/W mémoire arbitraire** — utiliser `memcpy()` entre les régions locales et partagées.
* **Appels de fonction avec > 8 args** — placer les arguments supplémentaires sur la pile suivant la convention d'appel arm64.
* **Transfert de port Mach** — passer des droits dans des messages Mach via les ports établis.
* **Transfert de descripteur de fichier** — tirer parti des fileports (voir *triple_fetch*).

Tout cela est encapsulé dans la bibliothèque [`threadexec`](https://github.com/bazad/threadexec) pour une réutilisation facile.

---

## 6. Nuances d'Apple Silicon (arm64e)

Sur les appareils Apple Silicon (arm64e), les **Codes d'Authentification de Pointeur (PAC)** protègent toutes les adresses de retour et de nombreux pointeurs de fonction. Les techniques de détournement de thread qui *réutilisent le code existant* continuent de fonctionner car les valeurs originales dans `lr`/`pc` portent déjà des signatures PAC valides. Des problèmes surviennent lorsque vous essayez de sauter vers une mémoire contrôlée par l'attaquant :

1. Allouer de la mémoire exécutable à l'intérieur de la cible (allocation distante `mach_vm_allocate` + `mprotect(PROT_EXEC)`).
2. Copier votre charge utile.
3. À l'intérieur du processus *distant*, signer le pointeur :
```c
uint64_t ptr = (uint64_t)payload;
ptr = ptrauth_sign_unauthenticated((void*)ptr, ptrauth_key_asia, 0);
```
4. Définir `pc = ptr` dans l'état du thread détourné.

Alternativement, restez conforme au PAC en enchaînant des gadgets/fonctions existants (ROP traditionnel).

## 7. Détection et durcissement avec EndpointSecurity

Le cadre **EndpointSecurity (ES)** expose des événements du noyau qui permettent aux défenseurs d'observer ou de bloquer les tentatives d'injection de thread :

* `ES_EVENT_TYPE_AUTH_GET_TASK` – déclenché lorsqu'un processus demande le port d'une autre tâche (par exemple, `task_for_pid()`).
* `ES_EVENT_TYPE_NOTIFY_REMOTE_THREAD_CREATE` – émis chaque fois qu'un thread est créé dans une tâche *différente*.
* `ES_EVENT_TYPE_NOTIFY_THREAD_SET_STATE` (ajouté dans macOS 14 Sonoma) – indique la manipulation des registres d'un thread existant.

Client Swift minimal qui imprime des événements de thread distant :
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
Interroger avec **osquery** ≥ 5.8 :
```sql
SELECT target_pid, source_pid, target_path
FROM es_process_events
WHERE event_type = 'REMOTE_THREAD_CREATE';
```
### Considérations sur le runtime durci

Distribuer votre application **sans** le droit `com.apple.security.get-task-allow` empêche les attaquants non-root d'obtenir son port de tâche. La Protection de l'Intégrité du Système (SIP) bloque toujours l'accès à de nombreux binaires Apple, mais les logiciels tiers doivent explicitement choisir de ne pas y participer.

## 8. Outils publics récents (2023-2025)

| Outil | Année | Remarques |
|-------|-------|-----------|
| [`task_vaccine`](https://github.com/rodionovd/task_vaccine) | 2023 | PoC compact qui démontre le détournement de thread conscient du PAC sur Ventura/Sonoma |
| `remote_thread_es` | 2024 | Helper EndpointSecurity utilisé par plusieurs fournisseurs EDR pour faire remonter les événements `REMOTE_THREAD_CREATE` |

> Lire le code source de ces projets est utile pour comprendre les changements d'API introduits dans macOS 13/14 et pour rester compatible entre Intel ↔ Apple Silicon.

## Références

- [https://bazad.github.io/2018/10/bypassing-platform-binary-task-threads/](https://bazad.github.io/2018/10/bypassing-platform-binary-task-threads/)
- [https://github.com/rodionovd/task_vaccine](https://github.com/rodionovd/task_vaccine)
- [https://developer.apple.com/documentation/endpointsecurity/es_event_type_notify_remote_thread_create](https://developer.apple.com/documentation/endpointsecurity/es_event_type_notify_remote_thread_create)

{{#include ../../../../banners/hacktricks-training.md}}
