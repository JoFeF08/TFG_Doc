# Rúbrica d'avaluació qualitativa del comportament — model desplegat (`best_nash`)

Aquesta rúbrica es defineix **abans** de jugar, per evitar el biaix de racionalitzar el
comportament observat. S'omple jugant **10–20 partides senceres** contra el model desplegat
(`best_nash`, un dels pesos de l'execució de la Fase 6) via `demo.py`.

## Protocol

1. Llançar el joc: `python demo.py` (jugador 0 = humà, jugador 1 = `best_nash`).
2. Jugar **com a mínim 10 partides** (recomanat 15–20), alternant qui fa de mà si es pot.
3. Per **cada partida**, anotar la puntuació de cada criteri (1–5) i notes lliures.
4. Al final, calcular la **mitjana per criteri** i una valoració global.

## Escala (1–5)

| Valor | Significat |
|-------|------------|
| 1 | Molt deficient / incoherent |
| 2 | Deficient |
| 3 | Acceptable |
| 4 | Bo |
| 5 | Excel·lent / indistingible d'un humà fort |

## Criteris

| # | Criteri | Descriptor (què observar) |
|---|---------|---------------------------|
| C1 | **Gestió de l'envit** | Canta/accepta/es retira de l'envit segons els punts de la mà? O l'ignora? |
| C2 | **Credibilitat del farol** | Quan aposta truc amb mà feble, és creïble? Abusa del farol fins a ser predictible? |
| C3 | **Timing del truc** | Canta el truc en moments raonables, o sempre/mai? |
| C4 | **Resposta a l'agressió** | Gestiona bé quan el rival canta truc/envit (accepta/es retira amb criteri)? |
| C5 | **Coherència del joc de cartes** | L'ordre i tria de cartes té sentit estratègic? |
| C6 | **Adaptació al marcador** | Canvia d'estil segons va guanyant/perdent o segons la fase de la partida? |
| C7 | **Errors greus** | Comptador de jugades clarament dolentes (menys = millor; 5 = cap error). |
| C8 | **Sensació global** | Com de fort / humà sembla en conjunt? |

## Full de recollida (una fila per partida)

| Partida | Guanyador | C1 | C2 | C3 | C4 | C5 | C6 | C7 | C8 | Notes |
|---------|-----------|----|----|----|----|----|----|----|----|-------|
| 1 |  |  |  |  |  |  |  |  |  |  |
| 2 |  |  |  |  |  |  |  |  |  |  |
| 3 |  |  |  |  |  |  |  |  |  |  |
| … |  |  |  |  |  |  |  |  |  |  |
| **Mitjana** | | | | | | | | | | |

## Hipòtesi prèvia (de l'anàlisi micro)

L'anàlisi numèrica (notebook `analisi_best_nash.ipynb`) descriu una política amb una
**divisió del treball** clara entre truc i envit:

- **Truc com a farol**, desacoblat de la força de la mà: obre truc més amb mà feble
  (fins al ~75%) que amb mà forta o molt forta (27–40%), no es retira mai davant un truc
  cantat (accepta el 100%) i el reapuja sovint (~59%). L'agressió **escala per ronda**
  (del ~40% a la primera fins al ~97% a la tercera).
- **Envit selectiu i racional**: obre envit només amb punts alts (fins al ~33%, mediana de
  29 punts quan obre vs 6 quan es retira), però accepta el repte molt sovint (~71%).
- **Sense adaptació al marcador**: l'agressió al truc és plana (~50%) tant si va per
  darrere, igualat o per davant.

S'espera, doncs:

- puntuació **baixa a C6 (adaptació al marcador)**, perquè el model no adapta l'agressió a
  la puntuació;
- puntuació **moderada a C2 (credibilitat del farol)**: el farol és sistemàtic i l'escalada
  per ronda és llegible, cosa que el pot fer predictible a la llarga;
- puntuació **positiva a C1 (gestió de l'envit)**, per la selecció clara segons els punts.

La rúbrica servirà per **confirmar o refutar** si aquest estil, tot i ser estadísticament
proper a l'equilibri, resulta creïble i no explotable per a un humà.
