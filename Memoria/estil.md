# Regles d'estil — Memòria TFG

## Acrònims

- Primera aparició: `\acf{}` (forma completa + sigles).
- Usos posteriors: `\ac{}` (sigles soles).
- **Mai** tornar a usar `\acf{}` si l'acrònim ja ha aparegut al document.
- `\acs{}` (sempre abreviatura) i `\acl{}` (sempre forma llarga) NO consumeixen el primer ús en mencions de pas; usa'ls per llistar acrònims sense "gastar" la definició.

### Afegir nous acrònims

Quan cal afegir un acrònim nou: inserir la línia `\acro{}` a `Acronims.tex` en **ordre alfabètic estricte** per la sigla. El paràmetre de `\begin{acronym}[X]` ha de ser la sigla més llarga per alinear correctament la columna.

### Primers usos ja fixats (no tornar a usar `\acf{}`)

| Acrònim | Primer ús |
|---------|-----------|
| IA, AR | `\ac{}` automàtic, Introduccio.tex |
| DRL | `\acf{DRL}`, Introduccio.tex:46 |
| POMDP | `\acf{POMDP}`, Introduccio.tex:63 |
| DQN, NFSP, PPO | `\acf{}`, Introduccio.tex:120 |
| MDP | `\acf{MDP}`, MarcTeoric.tex:31 |
| DRQN | `\acf{DRQN}`, MarcTeoric.tex:486 |
| GRU | `\acf{GRU}`, MarcTeoric.tex:487 |
| LSTM | `\acs{LSTM}` → primer `\ac` pendent |
| MVC | `\acf{MVC}`, JocSimulacio.tex:9 |
| TFG | `\ac{TFG}`, Introduccio.tex:30 |

## Referències bibliogràfiques

- Totes les cites han d'usar `\cite{clau}`, mai text manual d'autor-any com `(Brown, 2019)`.
- Estil numèric (`IEEEtran`): la cita apareix com `[N]` al text, mai entre parèntesis amb l'any.
- Format habitual: `\textbf{Nom del sistema}~\cite{clau}` seguit directament de la descripció.
- Si cal mencionar l'any al cos del text, integrar-lo en prosa: «el 2019, Brown i Sandholm~\cite{clau} estenen...».
- Quan es cita per primera vegada un sistema o mètode, posar la cita immediatament després del nom, no al final del paràgraf.
- Noves entrades al `.bib`: seguir el format existent (camp `author`, `title`, `journal`/`booktabs`/`publisher`, `year`). La clau ha de ser `cognomANYparaula` (ex: `brown1951fictitious`).

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
