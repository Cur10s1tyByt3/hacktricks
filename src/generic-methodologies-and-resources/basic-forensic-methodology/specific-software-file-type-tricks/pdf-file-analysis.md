# Análisis de archivos PDF

{{#include ../../../banners/hacktricks-training.md}}

**Para más detalles, consulta:** [**https://trailofbits.github.io/ctf/forensics/**](https://trailofbits.github.io/ctf/forensics/)

El formato PDF es conocido por su complejidad y su potencial para ocultar datos, lo que lo convierte en un punto focal para los desafíos forenses de CTF. Combina elementos de texto plano con objetos binarios, que pueden estar comprimidos o cifrados, y puede incluir scripts en lenguajes como JavaScript o Flash. Para entender la estructura del PDF, se puede consultar el [material introductorio de Didier Stevens](https://blog.didierstevens.com/2008/04/09/quickpost-about-the-physical-and-logical-structure-of-pdf-files/), o utilizar herramientas como un editor de texto o un editor específico de PDF como Origami.

Para una exploración o manipulación más profunda de los PDFs, están disponibles herramientas como [qpdf](https://github.com/qpdf/qpdf) y [Origami](https://github.com/mobmewireless/origami-pdf). Los datos ocultos dentro de los PDFs pueden estar ocultos en:

- Capas invisibles
- Formato de metadatos XMP de Adobe
- Generaciones incrementales
- Texto del mismo color que el fondo
- Texto detrás de imágenes o imágenes superpuestas
- Comentarios no mostrados

Para un análisis personalizado de PDF, se pueden utilizar bibliotecas de Python como [PeepDF](https://github.com/jesparza/peepdf) para crear scripts de análisis a medida. Además, el potencial del PDF para el almacenamiento de datos ocultos es tan vasto que recursos como la guía de la NSA sobre riesgos y contramedidas de PDF, aunque ya no se aloje en su ubicación original, aún ofrecen valiosos conocimientos. Una [copia de la guía](http://www.itsecure.hu/library/file/Biztons%C3%A1gi%20%C3%BAtmutat%C3%B3k/Alkalmaz%C3%A1sok/Hidden%20Data%20and%20Metadata%20in%20Adobe%20PDF%20Files.pdf) y una colección de [trucos del formato PDF](https://github.com/corkami/docs/blob/master/PDF/PDF.md) de Ange Albertini pueden proporcionar más lecturas sobre el tema.

## Construcciones maliciosas comunes

Los atacantes a menudo abusan de objetos y acciones específicas de PDF que se ejecutan automáticamente cuando se abre o se interactúa con el documento. Palabras clave que vale la pena buscar:

* **/OpenAction, /AA** – acciones automáticas ejecutadas al abrir o en eventos específicos.
* **/JS, /JavaScript** – JavaScript incrustado (a menudo ofuscado o dividido entre objetos).
* **/Launch, /SubmitForm, /URI, /GoToE** – lanzadores de procesos externos / URL.
* **/RichMedia, /Flash, /3D** – objetos multimedia que pueden ocultar cargas útiles.
* **/EmbeddedFile /Filespec** – archivos adjuntos (EXE, DLL, OLE, etc.).
* **/ObjStm, /XFA, /AcroForm** – flujos de objetos o formularios comúnmente abusados para ocultar shell-code.
* **Actualizaciones incrementales** – múltiples marcadores %%EOF o un **/Prev** de gran tamaño pueden indicar datos añadidos después de la firma para eludir AV.

Cuando cualquiera de los tokens anteriores aparece junto con cadenas sospechosas (powershell, cmd.exe, calc.exe, base64, etc.), el PDF merece un análisis más profundo.

---

## Hoja de trucos de análisis estático
```bash
# Fast triage – keyword statistics
pdfid.py suspicious.pdf

# Deep dive – decompress/inspect the object tree
pdf-parser.py -f suspicious.pdf                # interactive
pdf-parser.py -a suspicious.pdf                # automatic report

# Search for JavaScript and pretty-print it
pdf-parser.py -search "/JS" -raw suspicious.pdf | js-beautify -

# Dump embedded files
peepdf "open suspicious.pdf" "objects embeddedfile" "extract 15 16 17" -o dumps/

# Remove passwords / encryptions before processing with other tools
qpdf --password='secret' --decrypt suspicious.pdf clean.pdf

# Lint the file with a Go verifier (checks structure violations)
pdfcpu validate -mode strict clean.pdf
```
Proyectos adicionales útiles (mantenidos activamente 2023-2025):
* **pdfcpu** – Biblioteca/CLI de Go capaz de *lint*, *desencriptar*, *extraer*, *comprimir* y *sanitizar* PDFs.
* **pdf-inspector** – Visualizador basado en navegador que renderiza el gráfico de objetos y flujos.
* **PyMuPDF (fitz)** – Motor de Python scriptable que puede renderizar páginas de forma segura a imágenes para detonar JS incrustado en un sandbox endurecido.

---

## Técnicas de ataque recientes (2023-2025)

* **MalDoc en PDF poliglota (2023)** – JPCERT/CC observó a actores de amenazas añadiendo un documento de Word basado en MHT con macros VBA después del final **%%EOF**, produciendo un archivo que es tanto un PDF válido como un DOC válido. Los motores AV que analizan solo la capa PDF pierden la macro. Las palabras clave estáticas de PDF están limpias, pero `file` aún imprime `%PDF`. Trate cualquier PDF que también contenga la cadena `<w:WordDocument>` como altamente sospechoso.
* **Actualizaciones incrementales de sombra (2024)** – Los adversarios abusan de la función de actualización incremental para insertar un segundo **/Catalog** con `/OpenAction` malicioso mientras mantienen la primera revisión benigna firmada. Las herramientas que inspeccionan solo la primera tabla xref son eludidas.
* **Cadena UAF de análisis de fuentes – CVE-2024-30284 (Acrobat/Reader)** – una función vulnerable de **CoolType.dll** puede ser alcanzada desde fuentes CIDType2 incrustadas, permitiendo la ejecución remota de código con los privilegios del usuario una vez que se abre un documento manipulado. Parcheado en APSB24-29, mayo de 2024.

---

## Plantilla rápida de regla YARA
```yara
rule Suspicious_PDF_AutoExec {
meta:
description = "Generic detection of PDFs with auto-exec actions and JS"
author      = "HackTricks"
last_update = "2025-07-20"
strings:
$pdf_magic = { 25 50 44 46 }          // %PDF
$aa        = "/AA" ascii nocase
$openact   = "/OpenAction" ascii nocase
$js        = "/JS" ascii nocase
condition:
$pdf_magic at 0 and ( all of ($aa, $openact) or ($openact and $js) )
}
```
---

## Consejos defensivos

1. **Parchear rápido** – mantén Acrobat/Reader en la última versión continua; la mayoría de las cadenas RCE observadas en la naturaleza aprovechan vulnerabilidades de días n que fueron corregidas meses antes.
2. **Eliminar contenido activo en la puerta de enlace** – utiliza `pdfcpu sanitize` o `qpdf --qdf --remove-unreferenced` para eliminar JavaScript, archivos incrustados y acciones de lanzamiento de PDFs entrantes.
3. **Desarme y reconstrucción de contenido (CDR)** – convierte PDFs a imágenes (o PDF/A) en un host de sandbox para preservar la fidelidad visual mientras se descartan objetos activos.
4. **Bloquear características poco utilizadas** – las configuraciones de “Seguridad mejorada” en Reader permiten desactivar JavaScript, multimedia y renderizado 3D.
5. **Educación del usuario** – la ingeniería social (engaños de facturas y currículos) sigue siendo el vector inicial; enseña a los empleados a reenviar archivos adjuntos sospechosos a IR.

## Referencias

* JPCERT/CC – “MalDoc en PDF – Detección eludida al incrustar un archivo de Word malicioso en un archivo PDF” (Ago 2023)
* Adobe – Actualización de seguridad para Acrobat y Reader (APSB24-29, May 2024)


{{#include ../../../banners/hacktricks-training.md}}
