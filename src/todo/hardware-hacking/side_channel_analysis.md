# Analyse des Attaques par Canal Latéral

{{#include ../../banners/hacktricks-training.md}}

Les attaques par canal latéral récupèrent des secrets en observant des "fuites" physiques ou micro-architecturales qui sont *corrélées* avec l'état interne mais qui ne font *pas* partie de l'interface logique de l'appareil. Les exemples vont de la mesure du courant instantané tiré par une carte à puce à l'abus des effets de gestion de l'alimentation du CPU sur un réseau.

---

## Principaux Canaux de Fuite

| Canal | Cible Typique | Instrumentation |
|-------|---------------|-----------------|
| Consommation d'énergie | Cartes à puce, MCU IoT, FPGA | Oscilloscope + résistance de shunt/probe HS (par ex. CW503) |
| Champ électromagnétique (EM) | CPU, RFID, accélérateurs AES | Sonde H-field + LNA, ChipWhisperer/RTL-SDR |
| Temps d'exécution / caches | CPU de bureau & cloud | Chronomètres de haute précision (rdtsc/rdtscp), temps de vol à distance |
| Acoustique / mécanique | Claviers, imprimantes 3-D, relais | Microphone MEMS, vibromètre laser |
| Optique & thermique | LEDs, imprimantes laser, DRAM | Photodiode / caméra haute vitesse, caméra IR |
| Induit par des fautes | Cryptos ASIC/MCU | Glitch d'horloge/tension, EMFI, injection laser |

---

## Analyse de Puissance

### Analyse de Puissance Simple (SPA)
Observez une *unique* trace et associez directement les pics/creux avec des opérations (par ex. S-boxes DES).
```python
# ChipWhisperer-husky example – capture one AES trace
from chipwhisperer.capture.api.programmers import STMLink
from chipwhisperer.capture import CWSession
cw = CWSession(project='aes')
trig = cw.scope.trig
cw.connect(cw.capture.scopes[0])
cw.capture.init()
trace = cw.capture.capture_trace()
print(trace.wave)  # numpy array of power samples
```
### Analyse de puissance différentielle/corrélée (DPA/CPA)
Acquérir *N > 1 000* traces, hypothétiser le byte de clé `k`, calculer le modèle HW/HD et corréler avec le leak.
```python
import numpy as np
corr = np.corrcoef(leakage_model(k), traces[:,sample])
```
CPA reste à la pointe de la technologie, mais les variantes d'apprentissage automatique (MLA, SCA par apprentissage profond) dominent désormais des compétitions telles qu'ASCAD-v2 (2023).

---

## Analyse Électromagnétique (EMA)
Les sondes EM en champ proche (500 MHz–3 GHz) divulguent des informations identiques à celles de l'analyse de puissance *sans* insérer de shunts. Des recherches de 2024 ont démontré la récupération de clés à **>10 cm** d'un STM32 en utilisant la corrélation de spectre et des front-ends RTL-SDR à faible coût.

---

## Attaques par Timing & Micro-architecturales
Les CPU modernes divulguent des secrets par le biais de ressources partagées :
* **Hertzbleed (2022)** – le redimensionnement de fréquence DVFS est corrélé avec le poids de Hamming, permettant l'extraction *à distance* des clés EdDSA.
* **Downfall / Gather Data Sampling (Intel, 2023)** – exécution transitoire pour lire les données AVX-gather à travers des threads SMT.
* **Zenbleed (AMD, 2023) & Inception (AMD, 2023)** – la mauvaise prédiction spéculative de vecteurs divulgue des registres inter-domaines.

Pour un traitement large des problèmes de classe Spectre, voir {{#ref}}
../../cpu-microarchitecture/microarchitectural-attacks.md
{{#endref}}

---

## Attaques Acoustiques & Optiques
* 2024 "​iLeakKeys" a montré 95 % de précision dans la récupération des frappes de clavier d'un ordinateur portable à partir d'un **microphone de smartphone via Zoom** en utilisant un classificateur CNN.
* Des photodiodes à haute vitesse capturent l'activité LED DDR4 et reconstruisent les clés de tour AES en moins de 1 minute (BlackHat 2023).

---

## Injection de Pannes & Analyse de Pannes Différentielles (DFA)
Combiner des pannes avec des fuites de canal latéral raccourcit la recherche de clés (par exemple, DFA AES à 1 trace). Outils récents à prix de hobbyiste :
* **ChipSHOUTER & PicoEMP** – glitching par impulsion électromagnétique sub-1 ns.
* **GlitchKit-R5 (2025)** – plateforme de glitching d'horloge/tension open-source supportant les SoCs RISC-V.

---

## Flux de Travail d'Attaque Typique
1. Identifier le canal de fuite & le point de montage (broche VCC, condensateur de découplage, point en champ proche).
2. Insérer un déclencheur (GPIO ou basé sur un motif).
3. Collecter >1 k traces avec un échantillonnage/filtrage appropriés.
4. Prétraiter (alignement, suppression de la moyenne, filtre LP/HP, ondelette, PCA).
5. Récupération de clé statistique ou ML (CPA, MIA, DL-SCA).
6. Valider et itérer sur les valeurs aberrantes.

---

## Défenses & Renforcement
* **Implémentations en temps constant** & algorithmes résistants à la mémoire.
* **Masquage/mélange** – diviser les secrets en parts aléatoires ; résistance de premier ordre certifiée par TVLA.
* **Cacher** – régulateurs de tension sur puce, horloge randomisée, logique à double rail, boucliers EM.
* **Détection de pannes** – calcul redondant, signatures de seuil.
* **Opérationnel** – désactiver DVFS/turbo dans les noyaux cryptographiques, isoler SMT, interdire la co-localisation dans des clouds multi-locataires.

---

## Outils & Cadres
* **ChipWhisperer-Husky** (2024) – oscilloscope 500 MS/s + déclencheur Cortex-M ; API Python comme ci-dessus.
* **Riscure Inspector & FI** – commercial, supporte l'évaluation automatisée des fuites (TVLA-2.0).
* **scaaml** – bibliothèque SCA d'apprentissage profond basée sur TensorFlow (v1.2 – 2025).
* **pyecsca** – cadre SCA ECC open-source de l'ANSSI.

---

## Références

* [Documentation ChipWhisperer](https://chipwhisperer.readthedocs.io/en/latest/)
* [Article sur l'attaque Hertzbleed](https://www.hertzbleed.com/)


{{#include ../../banners/hacktricks-training.md}}
