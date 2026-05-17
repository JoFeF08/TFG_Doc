# 19. Fase 6: Marc Teòric (Neural Fictitious Self-Play)

## 1. Motivació: La limitació de la Fase 5

La Fase 5 va introduir un *pool* mixt d'agents basats en regles i *snapshots* històrics de l'agent PPO. Tot i que aquesta aproximació (una forma de Fictitious Self-Play simplificat) va aconseguir reduir substancialment l'explotabilitat de la política (pujant la *metric_robust* de 59.5% a 77.8%), va revelar una mancança estructural: la dispersió del rendiment contra les diferents variants de la *pool* (`std_pool`) no va disminuir. Es va mantenir estable al voltant del 10–15% durant tot l'entrenament.

Això passa perquè l'agent PPO, en cada instant $t$, aprèn una *best-response* contra la distribució d'oponents actual. Tot i que l'entorn varia, l'agent reacciona especialitzant-se i des-especialitzant-se periòdicament sense arribar mai a una estratègia completament "equilibrada" (un Equilibri de Nash). 

## 2. Fictitious Play i Neural Fictitious Self-Play (NFSP)

Per assolir l'Equilibri de Nash en jocs de dos jugadors de suma zero (com el Truc, a la pràctica), Heinrich & Silver (2016) van proposar el **Neural Fictitious Self-Play (NFSP)**. La idea central és que, mentre l'agent aprèn la *best-response* (mitjançant un algorisme de RL com PPO), en paral·lel s'ha de calcular i emmagatzemar la **política mitjana** (average policy) de tot l'historial de l'agent.

L'Equilibri de Nash s'assoleix quan els jugadors juguen segons aquesta *política mitjana*, i no segons la *best-response* del darrer pas.

## 3. Average Policy via Reservoir Sampling

Atès que les xarxes neuronals no poden sumar i dividir directament tots els seus pesos històrics per obtenir una "xarxa mitjana", NFSP ho resol com un problema d'Aprenentatge Supervisat (SL):
- Es crea una xarxa neuronal independent (`AveragePolicyNet`).
- Es guarda un historial de les accions preses per l'agent de RL (PPO) en un buffer especial: el **Reservoir Buffer**.
- S'entrena la xarxa SL per predir les accions guardades al buffer, aproximant així el comportament històric mitjà de l'agent.

**Per què Reservoir Sampling?** Un buffer estàndard (com FIFO) sempre estaria esbiaixat cap al comportament més recent. El Reservoir Sampling garanteix matemàticament que cada acció mai presa per l'agent al llarg de *tot* l'entrenament tingui la mateixa probabilitat exacta de romandre al buffer, creant un conjunt de dades que representa perfectament l'average policy real.

## 4. El Paràmetre d'Anticipació ($\eta$)

Si el PPO entrenés només contra l'average policy de l'oponent (FSP pur), el procés pot ser molt lent i rígid. NFSP introdueix el paràmetre d'anticipació $\eta$ (eta). Aquest hiperparàmetre (típicament $\eta = 0.5$) defineix la probabilitat que el model en entrenament (PPO) jugui contra:
- **$\eta$**: L'Average Policy (xarxa SL).
- **$1 - \eta$**: La *best-response* actual de l'oponent (a la nostra arquitectura, el `SelfPlayPool` que conté els snapshots i les regles).

Barrejant ambdues fonts, PPO s'enfronta a l'estratègia estable de fons (SL) i a les variacions tàctiques recents de la pool, afavorint una convergència robusta.

## 5. Resultats esperats de la Fase 6

1. **Reducció de `std_pool`**: Com que l'average policy no té els "cicles d'especialització" d'un snapshot individual, actua com a "àncora". Entrenar contra la barreja d'aquesta àncora i la pool obligarà el PPO a generalitzar, reduint la variància entre estils.
2. **Convergència a Nash**: Si l'explotabilitat envers la pròpia average policy (`exploit_vs_sl`) tendeix a zero (o un nombre petit estabilitzat al voltant del ~5%), vol dir que la *best-response* actual ja no pot explotar la *average policy*, definició empírica de l'apropament a un Equilibri de Nash.

## 6. Reutilització de Mètriques de Fases Anteriors

La Fase 6 hereta totes les mètriques de la Fase 5 i n'afegeix de noves específiques per a NFSP:

### Mètriques heretades (Fases 4–5)
- **`metric`**: Win Rate contra l'agent de regles base i l'agent aleatori, ponderat 70%/30%. Mesura el rendiment brut de l'agent.
- **`metric_robust`**: Mitjana del WR contra totes les variants de la pool (`conservador`, `agressiu`, `truc_bot`, `envit_bot`, `faroler`, `equilibrat`). Mesura la robustesa i generalització.
- **`std_pool`**: Desviació estàndard del WR entre totes les variants. Un valor alt indica especialització; un valor baix, equilibri entre estils.
- **`wr_{variant}`**: WR individual contra cada variant de la pool (6 variants).

### Mètriques noves (Fase 6 / NFSP)
- **`sl_loss`**: Cross-Entropy Loss de l'`AveragePolicyNet` respecte les parelles `(observació, acció)` del Reservoir Buffer. Indica si la política mitjana converge (ha de ser decreixent).
- **`wr_vs_sl`**: Win Rate de l'agent PPO (best-response) jugant contra el propi `SLAgent` (Average Policy).
- **`exploit_vs_sl`**: `|wr_vs_sl − 50.0|`. Quantifica la distància empírica a l'Equilibri de Nash; com més proper a 0, menys explotable és l'average policy i més a prop estem del Nash.
- **`eta_actual`**: Valor real de la proporció SL vs pool durant les partides d'entrenament (ha d'estabilitzar-se al voltant de `η`).

## 7. El Bucle d'Entrenament RL + SL

La Fase 6 combina dos bucles d'optimització independents que s'executen en paral·lel:

### Bucle RL (PPO)
1. L'agent PPO observa l'estat del joc i tria una acció.
2. L'oponent s'escull del `NFSPPool`: amb probabilitat `η` és el `SLAgent`; amb probabilitat `1−η` és un agent de la `SelfPlayPool` (regles o snapshot PPO).
3. Es recull la recompensa i la transició `(s, a, r, s')` es desa al buffer PPO (rollout).
4. Cada `n_steps × num_envs = 8.192` passos, PPO actualitza la seva política per gradient.

### Bucle SL (Cross-Entropy)
1. **Recollida de dades**: A cada pas, es guarda la tupla `(observació_240d, acció)` al **Reservoir Buffer** (capacitat 200k). El Reservoir garanteix que qualsevol acció de tota la història té la mateixa probabilitat de romandre al buffer.
2. **Actualització SL**: Cada `sl_every = 50.000` passos, es fa un epoch de Cross-Entropy sobre un minibatch del Reservoir per entrenar l'`AveragePolicyNet`.
3. La SL Loss és decreixent al llarg del entrenament, reflectint que la xarxa SL aproxima cada cop millor l'historial de decisions del PPO.

### Interacció entre els dos bucles
Els dos bucles operen sobre xarxes separades però comparteixen el backbone COS congelat (`CosMultiInputSB3`, 364k paràmetres). Això permet que l'`AveragePolicyNet` aprofiti les representacions apreses pel feature extractor sense re-entrenar-lo, mantenint la coherència entre la política RL i la política SL.
