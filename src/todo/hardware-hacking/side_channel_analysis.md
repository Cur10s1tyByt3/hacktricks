# Análisis de Ataques por Canal Lateral

{{#include ../../banners/hacktricks-training.md}}

Los ataques por canal lateral recuperan secretos al observar "filtraciones" físicas o micro-arquitectónicas que están *correlacionadas* con el estado interno pero que *no* son parte de la interfaz lógica del dispositivo. Los ejemplos van desde medir la corriente instantánea consumida por una tarjeta inteligente hasta abusar de los efectos de gestión de energía de la CPU a través de una red.

---

## Principales Canales de Filtración

| Canal | Objetivo Típico | Instrumentación |
|-------|----------------|-----------------|
| Consumo de energía | Tarjetas inteligentes, MCUs IoT, FPGAs | Osciloscopio + resistor de derivación/probe HS (por ejemplo, CW503) |
| Campo electromagnético (EM) | CPUs, RFID, aceleradores AES | Probe de campo H + LNA, ChipWhisperer/RTL-SDR |
| Tiempo de ejecución / cachés | CPUs de escritorio y en la nube | Temporizadores de alta precisión (rdtsc/rdtscp), tiempo de vuelo remoto |
| Acústico / mecánico | Teclados, impresoras 3-D, relés | Micrófono MEMS, vibrómetro láser |
| Óptico y térmico | LEDs, impresoras láser, DRAM | Fotodiodo / cámara de alta velocidad, cámara IR |
| Inducido por fallos | Criptos ASIC/MCU | Glitch de reloj/voltaje, EMFI, inyección láser |

---

## Análisis de Potencia

### Análisis de Potencia Simple (SPA)
Observe un *único* trazo y asocie directamente picos/valles con operaciones (por ejemplo, S-boxes de DES).
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
### Análisis de Potencia Diferencial/Correlacional (DPA/CPA)
Adquiere *N > 1 000* trazas, hipotetiza el byte de clave `k`, calcula el modelo HW/HD y correlaciona con la fuga.
```python
import numpy as np
corr = np.corrcoef(leakage_model(k), traces[:,sample])
```
CPA sigue siendo lo último en tecnología, pero las variantes de aprendizaje automático (MLA, SCA de aprendizaje profundo) ahora dominan competiciones como ASCAD-v2 (2023).

---

## Análisis Electromagnético (EMA)
Las sondas EM de campo cercano (500 MHz–3 GHz) filtran información idéntica al análisis de potencia *sin* insertar shunts. La investigación de 2024 demostró la recuperación de claves a **>10 cm** de un STM32 utilizando correlación de espectro y front-ends RTL-SDR de bajo costo.

---

## Ataques de Tiempo y Microarquitectura
Los CPUs modernos filtran secretos a través de recursos compartidos:
* **Hertzbleed (2022)** – la escalabilidad de frecuencia DVFS se correlaciona con el peso de Hamming, permitiendo la extracción *remota* de claves EdDSA.
* **Downfall / Gather Data Sampling (Intel, 2023)** – ejecución transitoria para leer datos AVX-gather a través de hilos SMT.
* **Zenbleed (AMD, 2023) & Inception (AMD, 2023)** – la predicción errónea especulativa de vectores filtra registros entre dominios.

Para un tratamiento amplio de los problemas de clase Spectre, consulte {{#ref}}
../../cpu-microarchitecture/microarchitectural-attacks.md
{{#endref}}

---

## Ataques Acústicos y Ópticos
* 2024 "​iLeakKeys" mostró un 95 % de precisión en la recuperación de pulsaciones de teclado de una laptop desde un **micrófono de teléfono inteligente a través de Zoom** utilizando un clasificador CNN.
* Fotodiodos de alta velocidad capturan la actividad LED de DDR4 y reconstruyen claves de ronda AES en menos de 1 minuto (BlackHat 2023).

---

## Inyección de Fallos y Análisis Diferencial de Fallos (DFA)
Combinar fallos con filtraciones de canal lateral acorta la búsqueda de claves (por ejemplo, DFA AES de 1 traza). Herramientas recientes a precios de aficionado:
* **ChipSHOUTER & PicoEMP** – glitching de pulso electromagnético sub-1 ns.
* **GlitchKit-R5 (2025)** – plataforma de glitch de reloj/voltaje de código abierto que soporta SoCs RISC-V.

---

## Flujo de Trabajo Típico de Ataque
1. Identificar el canal de filtración y el punto de montaje (pin VCC, capacitor de desacoplamiento, punto de campo cercano).
2. Insertar disparador (GPIO o basado en patrones).
3. Recoger >1 k trazas con muestreo/filtros adecuados.
4. Pre-procesar (alineación, eliminación de media, filtro LP/HP, wavelet, PCA).
5. Recuperación de claves estadística o ML (CPA, MIA, DL-SCA).
6. Validar e iterar sobre los valores atípicos.

---

## Defensas y Fortalecimiento
* Implementaciones **de tiempo constante** y algoritmos de memoria-dura.
* **Enmascaramiento/barajado** – dividir secretos en partes aleatorias; resistencia de primer orden certificada por TVLA.
* **Ocultación** – reguladores de voltaje en chip, reloj aleatorizado, lógica de doble riel, escudos EM.
* **Detección de fallos** – computación redundante, firmas de umbral.
* **Operacional** – deshabilitar DVFS/turbo en núcleos criptográficos, aislar SMT, prohibir la co-localización en nubes multi-inquilino.

---

## Herramientas y Marcos
* **ChipWhisperer-Husky** (2024) – osciloscopio de 500 MS/s + disparador Cortex-M; API de Python como se mencionó.
* **Riscure Inspector & FI** – comercial, soporta evaluación automatizada de filtraciones (TVLA-2.0).
* **scaaml** – biblioteca SCA de aprendizaje profundo basada en TensorFlow (v1.2 – 2025).
* **pyecsca** – marco SCA ECC de código abierto de ANSSI.

---

## Referencias

* [Documentación de ChipWhisperer](https://chipwhisperer.readthedocs.io/en/latest/)
* [Documento de Ataque Hertzbleed](https://www.hertzbleed.com/)


{{#include ../../banners/hacktricks-training.md}}
