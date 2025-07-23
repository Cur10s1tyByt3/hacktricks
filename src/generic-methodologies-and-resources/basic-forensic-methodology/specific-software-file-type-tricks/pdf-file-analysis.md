# Analyse de fichiers PDF

{{#include ../../../banners/hacktricks-training.md}}

**Pour plus de détails, consultez :** [**https://trailofbits.github.io/ctf/forensics/**](https://trailofbits.github.io/ctf/forensics/)

Le format PDF est connu pour sa complexité et son potentiel à dissimuler des données, ce qui en fait un point focal pour les défis de forensique CTF. Il combine des éléments en texte brut avec des objets binaires, qui peuvent être compressés ou chiffrés, et peut inclure des scripts dans des langages comme JavaScript ou Flash. Pour comprendre la structure d'un PDF, on peut se référer au [matériel d'introduction de Didier Stevens](https://blog.didierstevens.com/2008/04/09/quickpost-about-the-physical-and-logical-structure-of-pdf-files/), ou utiliser des outils comme un éditeur de texte ou un éditeur spécifique aux PDF tel qu'Origami.

Pour une exploration ou une manipulation approfondie des PDF, des outils comme [qpdf](https://github.com/qpdf/qpdf) et [Origami](https://github.com/mobmewireless/origami-pdf) sont disponibles. Les données cachées dans les PDF peuvent être dissimulées dans :

- Couches invisibles
- Format de métadonnées XMP par Adobe
- Générations incrémentales
- Texte de la même couleur que l'arrière-plan
- Texte derrière des images ou images superposées
- Commentaires non affichés

Pour une analyse PDF personnalisée, des bibliothèques Python comme [PeepDF](https://github.com/jesparza/peepdf) peuvent être utilisées pour créer des scripts de parsing sur mesure. De plus, le potentiel du PDF pour le stockage de données cachées est si vaste que des ressources comme le guide de la NSA sur les risques et contre-mesures des PDF, bien qu'il ne soit plus hébergé à son emplacement d'origine, offrent encore des informations précieuses. Une [copie du guide](http://www.itsecure.hu/library/file/Biztons%C3%A1gi%20%C3%BAtmutat%C3%B3k/Alkalmaz%C3%A1sok/Hidden%20Data%20and%20Metadata%20in%20Adobe%20PDF%20Files.pdf) et une collection de [trucs sur le format PDF](https://github.com/corkami/docs/blob/master/PDF/PDF.md) par Ange Albertini peuvent fournir des lectures supplémentaires sur le sujet.

## Constructions malveillantes courantes

Les attaquants abusent souvent d'objets et d'actions PDF spécifiques qui s'exécutent automatiquement lorsque le document est ouvert ou interagi avec. Mots-clés à rechercher :

* **/OpenAction, /AA** – actions automatiques exécutées à l'ouverture ou lors d'événements spécifiques.
* **/JS, /JavaScript** – JavaScript intégré (souvent obfusqué ou réparti sur plusieurs objets).
* **/Launch, /SubmitForm, /URI, /GoToE** – lanceurs de processus externes / URL.
* **/RichMedia, /Flash, /3D** – objets multimédias qui peuvent cacher des charges utiles.
* **/EmbeddedFile /Filespec** – pièces jointes de fichiers (EXE, DLL, OLE, etc.).
* **/ObjStm, /XFA, /AcroForm** – flux d'objets ou formulaires souvent abusés pour cacher du shell-code.
* **Mises à jour incrémentales** – plusieurs marqueurs %%EOF ou un très grand décalage **/Prev** peuvent indiquer des données ajoutées après la signature pour contourner l'AV.

Lorsque l'un des tokens précédents apparaît avec des chaînes suspectes (powershell, cmd.exe, calc.exe, base64, etc.), le PDF mérite une analyse plus approfondie.

---

## Fiche de triche pour l'analyse statique
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
Projets supplémentaires utiles (activement maintenus 2023-2025) :
* **pdfcpu** – Bibliothèque/CLI Go capable de *lint*, *décrypter*, *extraire*, *compresser* et *sanitiser* des PDF.
* **pdf-inspector** – Visualiseur basé sur le navigateur qui rend le graphe d'objets et les flux.
* **PyMuPDF (fitz)** – Moteur Python scriptable qui peut rendre en toute sécurité des pages en images pour faire exploser du JS intégré dans un bac à sable renforcé.

---

## Techniques d'attaque récentes (2023-2025)

* **MalDoc dans un polyglotte PDF (2023)** – JPCERT/CC a observé des acteurs de la menace ajoutant un document Word basé sur MHT avec des macros VBA après le **%%EOF** final, produisant un fichier qui est à la fois un PDF valide et un DOC valide. Les moteurs AV qui analysent uniquement la couche PDF manquent la macro. Les mots-clés statiques PDF sont propres, mais `file` imprime toujours `%PDF`. Considérez tout PDF contenant également la chaîne `<w:WordDocument>` comme hautement suspect.
* **Mises à jour incrémentales d'ombre (2024)** – Les adversaires abusent de la fonctionnalité de mise à jour incrémentale pour insérer un second **/Catalog** avec un `/OpenAction` malveillant tout en gardant la première révision bénigne signée. Les outils qui inspectent uniquement la première table xref sont contournés.
* **Chaîne UAF de parsing de police – CVE-2024-30284 (Acrobat/Reader)** – une fonction vulnérable **CoolType.dll** peut être atteinte à partir de polices CIDType2 intégrées, permettant l'exécution de code à distance avec les privilèges de l'utilisateur une fois qu'un document conçu est ouvert. Corrigé dans APSB24-29, mai 2024.

---

## Modèle de règle YARA rapide
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

## Conseils défensifs

1. **Mettez à jour rapidement** – gardez Acrobat/Reader sur la dernière version continue ; la plupart des chaînes RCE observées dans la nature exploitent des vulnérabilités n-day corrigées des mois plus tôt.
2. **Supprimez le contenu actif à la passerelle** – utilisez `pdfcpu sanitize` ou `qpdf --qdf --remove-unreferenced` pour supprimer JavaScript, fichiers intégrés et actions de lancement des PDF entrants.
3. **Désarmement et reconstruction de contenu (CDR)** – convertissez les PDF en images (ou PDF/A) sur un hôte sandbox pour préserver la fidélité visuelle tout en éliminant les objets actifs.
4. **Bloquez les fonctionnalités rarement utilisées** – les paramètres de “Sécurité améliorée” d'entreprise dans Reader permettent de désactiver JavaScript, multimédia et rendu 3D.
5. **Éducation des utilisateurs** – l'ingénierie sociale (leurres de factures et de CV) reste le vecteur initial ; apprenez aux employés à transférer les pièces jointes suspectes à l'IR.

## Références

* JPCERT/CC – “MalDoc in PDF – Detection bypass by embedding a malicious Word file into a PDF file” (août 2023)
* Adobe – Mise à jour de sécurité pour Acrobat et Reader (APSB24-29, mai 2024)


{{#include ../../../banners/hacktricks-training.md}}
