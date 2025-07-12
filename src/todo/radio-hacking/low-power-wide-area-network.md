# Red de Área Amplia de Bajo Consumo

{{#include ../../banners/hacktricks-training.md}}

## Introducción

**Red de Área Amplia de Bajo Consumo** (LPWAN) es un grupo de tecnologías de red inalámbrica, de bajo consumo y de área amplia diseñadas para **comunicaciones de largo alcance** a una baja tasa de bits. Pueden alcanzar más de **seis millas** y sus **baterías** pueden durar hasta **20 años**.

Long Range (**LoRa**) es actualmente la capa física LPWAN más desplegada y su especificación de capa MAC abierta es **LoRaWAN**.

---

## LPWAN, LoRa y LoRaWAN

* LoRa – Capa física de Chirp Spread Spectrum (CSS) desarrollada por Semtech (propietaria pero documentada).
* LoRaWAN – Capa MAC/Red abierta mantenida por la LoRa-Alliance. Las versiones 1.0.x y 1.1 son comunes en el campo.
* Arquitectura típica: *dispositivo final → puerta de enlace (retransmisor de paquetes) → servidor de red → servidor de aplicación*.

> El **modelo de seguridad** se basa en dos claves raíz AES-128 (AppKey/NwkKey) que derivan claves de sesión durante el procedimiento de *unión* (OTAA) o están codificadas (ABP). Si alguna clave se filtra, el atacante obtiene plena capacidad de lectura/escritura sobre el tráfico correspondiente.

---

## Resumen de la superficie de ataque

| Capa | Vulnerabilidad | Impacto práctico |
|------|----------------|------------------|
| PHY | Jamming reactivo / selectivo | 100 % de pérdida de paquetes demostrada con un solo SDR y <1 W de salida |
| MAC | Repetición de Join-Accept y data-frame (reutilización de nonce, desbordamiento de contador ABP) | Suplantación de dispositivos, inyección de mensajes, DoS |
| Servidor de Red | Retransmisor de paquetes inseguro, filtros MQTT/UDP débiles, firmware de puerta de enlace desactualizado | RCE en puertas de enlace → pivoteo en la red OT/IT |
| Aplicación | AppKeys codificadas o predecibles | Fuerza bruta/desencriptar tráfico, suplantar sensores |

---

## Vulnerabilidades recientes (2023-2025)

* **CVE-2024-29862** – *ChirpStack gateway-bridge & mqtt-forwarder* aceptó paquetes TCP que eludieron las reglas de firewall con estado en puertas de enlace Kerlink, permitiendo la exposición de la interfaz de gestión remota. Corregido en 4.0.11 / 4.2.1 respectivamente.
* **Dragino LG01/LG308 series** – Múltiples CVEs de 2022-2024 (por ejemplo, 2022-45227 recorrido de directorio, 2022-45228 CSRF) aún observados sin parches en 2025; habilitar volcado de firmware no autenticado o sobrescritura de configuración en miles de puertas de enlace públicas.
* Desbordamiento de *retransmisor de paquetes UDP* de Semtech (aviso no publicado, parcheado en 2023-10): uplink elaborado mayor a 255 B activó stack-smash ‑> RCE en puertas de enlace de referencia SX130x (encontrado por Black Hat EU 2023 “LoRa Exploitation Reloaded”).

---

## Técnicas de ataque prácticas

### 1. Olfatear y desencriptar tráfico
```bash
# Capture all channels around 868.3 MHz with an SDR (USRP B205)
python3 lorattack/sniffer.py \
--freq 868.3e6 --bw 125e3 --rate 1e6 --sf 7 --session smartcity

# Bruteforce AppKey from captured OTAA join-request/accept pairs
python3 lorapwn/bruteforce_join.py --pcap smartcity.pcap --wordlist top1m.txt
```
### 2. OTAA join-replay (reutilización de DevNonce)

1. Captura un **JoinRequest** legítimo.
2. Reenvíalo inmediatamente (o incrementa RSSI) antes de que el dispositivo original transmita de nuevo.
3. El servidor de red asigna un nuevo DevAddr y claves de sesión mientras el dispositivo objetivo continúa con la sesión antigua → el atacante posee la sesión vacante y puede inyectar uplinks falsificados.

### 3. Downgrading de Adaptive Data-Rate (ADR)

Forzar SF12/125 kHz para aumentar el tiempo de aire → agotar el ciclo de trabajo de la puerta de enlace (denegación de servicio) mientras se mantiene bajo el impacto en la batería del atacante (solo enviar comandos MAC a nivel de red).

### 4. Jamming reactivo

*HackRF One* ejecutando un flujo de GNU Radio activa un chirp de banda ancha cada vez que se detecta un preámbulo – bloquea todos los factores de expansión con ≤200 mW TX; interrupción total medida a 2 km de distancia.

---

## Herramientas ofensivas (2025)

| Herramienta | Propósito | Notas |
|-------------|-----------|-------|
| **LoRaWAN Auditing Framework (LAF)** | Crear/analizar/atacar tramas LoRaWAN, analizadores respaldados por DB, fuerza bruta | Imagen de Docker, soporta entrada UDP de Semtech |
| **LoRaPWN** | Utilidad de Python de Trend Micro para fuerza bruta OTAA, generar downlinks, descifrar cargas útiles | Demo lanzada en 2023, agnóstico a SDR |
| **LoRAttack** | Sniffer de múltiples canales + replay con USRP; exporta PCAP/LoRaTap | Buena integración con Wireshark |
| **gr-lora / gr-lorawan** | Bloques OOT de GNU Radio para TX/RX de banda base | Fundación para ataques personalizados |

---

## Recomendaciones defensivas (lista de verificación para pentesters)

1. Preferir dispositivos **OTAA** con DevNonce verdaderamente aleatorio; monitorear duplicados.
2. Hacer cumplir **LoRaWAN 1.1**: contadores de trama de 32 bits, FNwkSIntKey / SNwkSIntKey distintos.
3. Almacenar el contador de trama en memoria no volátil (**ABP**) o migrar a OTAA.
4. Desplegar **elemento seguro** (ATECC608A/SX1262-TRX-SE) para proteger las claves raíz contra la extracción de firmware.
5. Deshabilitar puertos de reenvío de paquetes UDP remotos (1700/1701) o restringir con WireGuard/VPN.
6. Mantener las puertas de enlace actualizadas; Kerlink/Dragino proporcionan imágenes parcheadas de 2024.
7. Implementar **detección de anomalías de tráfico** (por ejemplo, analizador LAF) – marcar reinicios de contadores, uniones duplicadas, cambios repentinos de ADR.

## Referencias

* LoRaWAN Auditing Framework (LAF) – https://github.com/IOActive/laf
* Resumen de Trend Micro LoRaPWN – https://www.hackster.io/news/trend-micro-finds-lorawan-security-lacking-develops-lorapwn-python-utility-bba60c27d57a
{{#include ../../banners/hacktricks-training.md}}
