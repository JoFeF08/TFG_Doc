# 19. Fase 6: Marc Teòric (Neural Fictitious Self-Play)

## 1. Motivació: límit de la Fase 5

La Fase 5 (self-play mixt amb snapshots + regles) va millorar la robustesa, però encara mostra oscil·lacions i episodis de sobreespecialització. El motiu és que PPO continua essent una *best response* local: optimitza contra la distribució actual d'oponents, no contra la política mitjana de tota la seva història.

En jocs de suma zero, aquesta dinàmica pot entrar en cicles tàctics. Per això, la Fase 6 introdueix NFSP: separa explícitament la política de resposta òptima (RL) de la política mitjana (SL).

## 2. NFSP: *best response* + *average policy*

La idea de NFSP (Heinrich & Silver, 2016) és entrenar en paral·lel:

- Un model RL (aquí PPO) que aprèn la *best response*.
- Un model SL (`AveragePolicyNet`) que aproxima la política mitjana històrica.

La convergència cap a Nash no es mesura amb la política RL sola, sinó amb la relació entre la *best response* i aquesta política mitjana.

## 3. Política mitjana amb Reservoir Sampling

La política mitjana no es calcula fent mitjana de pesos; es reformula com un problema supervisat sobre parelles `(observació, acció)` recollides del PPO.

Per evitar biaix temporal, el dataset es manté amb **Reservoir Sampling**:

- Cada mostra vista durant l'entrenament té la mateixa probabilitat de romandre al buffer.
- Això aproxima la distribució global de decisions del PPO, no només les més recents.

La xarxa SL (`AveragePolicyNet`) s'entrena amb Cross-Entropy sobre aquest reservoir.

## 4. Anticipació \(\eta\) i mescla d'oponents

L'entorn usa `NFSPPool`, que combina:

- `SLAgent` amb probabilitat \(\eta\).
- `SelfPlayPool` (regles + snapshots PPO) amb probabilitat \(1-\eta\).

\(\eta\) és, per tant, el **pes de la política mitjana** dins del mostreig d'oponents. Intuïtivament:

- \(\eta\) alt: PPO s'entrena més contra l'`SLAgent` (més estabilitat i menys cicles tàctics).
- \(\eta\) baix: PPO s'entrena més contra la `SelfPlayPool` (més pressió de best-response i adaptació local).

Exemple: si \(\eta = 0.5\), aproximadament la meitat de partides d'entrenament són contra `SLAgent` i l'altra meitat contra `SelfPlayPool`.

En implementació real, \(\eta\) no és totalment estàtic: es pot forçar temporalment a 0 en períodes de baixa qualitat SL (calibració degradada) per estabilitzar l'aprenentatge.

## 5. Com es calcula la metrica Nash

Abans de definir el punt Nash operatiu, convé fixar les metriques tal com s'implementen al codi de F6:

- `metric` (rendiment brut):
	- ve de `evaluar_agent(...)` (importat de Fase 2),
	- combinació ponderada de WR contra random i WR contra regles,
	- forma usada al projecte: `metric = 0.25 * wr_random + 0.75 * wr_regles`.

- `metric_robust` (robustesa entre estils):
	- primer es calcula `wr_pool_mean = mitjana(wr_conservador, wr_agressiu, wr_truc_bot, wr_envit_bot, wr_faroler, wr_equilibrat)`,
	- després `std_pool = desviacio estandard` d'aquests 6 WR,
	- finalment `metric_robust = wr_pool_mean - 0.5 * std_pool`.

- `exploit_vs_sl` (gap empíric a Nash):
	- `wr_vs_sl` = WR del PPO actual contra el seu `SLAgent`, amb posició alternada,
	- `exploit_vs_sl = |wr_vs_sl - 50.0|`.

- `calib_envit` i `calib_truc` (de `calibracio.py`):
	- es mesuren sobre la **primera decisió** de cada mà quan el model és mà (`round_counter=0`, sense resposta pendent),
	- `calib_envit = P(apostar_envit | envit_score alt) - P(apostar_envit | envit_score baix)`,
	- buckets usats: `envit_score <= 7` (baix) i `envit_score >= 29` (alt),
	- `calib_truc = P(apostar_truc | ma forta de truc)` amb llindar de força alta (`forca_ma >= 216`),
	- al run F6, `calib_combined = (calib_envit + calib_truc) / 2`.

Aquestes metriques no són decoratives: entren directament als gates de validesa i selecció de checkpoints.

## 6. Nova definició operativa de “punt Nash”

A Fase 6 no n'hi ha prou amb minimitzar `exploit_vs_sl`. Es defineix un gate de validesa `nash_valid` per evitar falsos positius inicials:

- Reservoir ple (`len(reservoir) == capacity`).
- Entrenament prou madur (`step >= nash_min_steps`).
- Política no degenerada en calibració (`calib_envit >= CALIB_ENVIT_MIN`) i bon rendiment contra `envit_bot` (`wr_envit_bot >= ENVIT_BOT_MIN`).

Només en aquests punts es considera vàlid arxivar `best_nash`.

## 7. Mètriques de Fase 6

### Heretades

- `metric`: rendiment brut (mateixa definició que fases anteriors).
- `metric_robust`: robustesa global contra variants.
- `std_pool`: dispersió entre variants.
- `wr_{variant}`: win-rate per estil.
- `wr_vs_self`, `exploit_selfplay`: continuïtat de Fase 5.

### Noves o reforçades a F6

- `sl_loss`: pèrdua supervisada de `AveragePolicyNet`.
- `wr_vs_sl`: win-rate PPO contra `SLAgent`.
- `exploit_vs_sl = |wr_vs_sl - 50|`: distància empírica a Nash.
- `eta_actual`: mescla efectiva SL vs pool.
- `calib_envit`, `calib_truc`: salut tàctica de la política.
- `nash_valid`: indicador de validesa del punt Nash.
- `evals_sense_millora`: comptador per *early stopping* de `best_nash`.

## 8. Bucle RL + SL amb control d'estabilitat

### Bucle RL (PPO)

1. PPO juga contra l'oponent mostrejat de `NFSPPool`.
2. Les transicions alimenten PPO.
3. Paral·lelament, cada `(obs, acció)` s'envia al reservoir.

### Bucle SL (Cross-Entropy)

1. Cada `sl_every` passos s'entrena `AveragePolicyNet` amb minibatches del reservoir.
2. Els pesos SL s'actualitzen també en la còpia CPU usada pel pool d'oponents.
3. Es monitoritza calibració SL durant entrenament per detectar col·lapse.

### Mecanismes de seguretat

- **Pausa/resum de component SL** segons degradació de `calib_envit` (evita que un SL inestable domini el pool).
- **Criteri dual de `best_nash`**: només es guarda si milloren alhora `exploit_vs_sl` i `calib_combined`.
- **Early stopping de Nash**: si hi ha diversos punts `nash_valid` consecutius sense millora dual, es talla el run.

Aquestes tres peces són l'actualització conceptual clau de la Fase 6 respecte a una NFSP “clàssica” minimalista.

## 9. Nota metodològica: Nash operatiu vs Nash nominal

En un pipeline aplicat com aquest, un mínim puntual de `exploit_vs_sl` no implica necessàriament convergència real a Nash. Per això la Fase 6 adopta una definició operativa més estricta:

- només es considera "Nash vàlid" si el punt és madur, amb reservoir representatiu i calibració no degenerada,
- i el model seleccionat com a `best_nash` ha de millorar també la qualitat tàctica (calibració), no només el gap d'explotabilitat.

Aquesta distinció (Nash nominal vs Nash operatiu) és essencial per evitar conclusions optimistes causades per soroll estadístic o per polítiques degenerades que aparentment redueixen `exploit_vs_sl` però perden comportament estratègic útil.
