# Regles d'estil — Memòria TFG

## Acrònims

- Primera aparició: `\acf{}` (forma completa + sigles).
- Usos posteriors: `\ac{}` (sigles soles).
- **Mai** tornar a usar `\acf{}` si l'acrònim ja ha aparegut al document.

## Puntuació i incisos

- Substituir els guions llargs `---...---` per parèntesis `(...)`.
- Substituir les cometes altes `"..."` usades per remarcar o introduir termes per parèntesis `(...)`.
- Excepció: les cites textuals directes (`\emph{``...''}`  o `\guillemotleft...\guillemotright`) es mantenen amb les cometes corresponents.
- Evitar construccions que semblin generades per IA (llistes massa llargues, incisos encadenats).

## Figures i taules

- Per a escales d'apostes o informació tabular, usar `tabular` amb el paquet `booktabs` (`\toprule`, `\midrule`, `\bottomrule`).
- No incrustar Mermaid directament: exportar com a PNG/PDF i usar `\includegraphics`, o bé TikZ natiu.

## Portada

- Tutor principal: `\tutor{Dr. Francesc Xavier Gayà Morey}`
- Cotutor: `\cotutor{Dr. Gabriel Moyà Alcover}` (comanda afegida a `TFGEPSUIB.cls`)

## Compilació

Seqüència obligatòria quan canvia la bibliografia:
`pdflatex → bibtex → pdflatex → pdflatex`

Si hi ha referències `[?]` al PDF, esborrar els fitxers `.aux .bbl .blg` i recompilar des de zero.
