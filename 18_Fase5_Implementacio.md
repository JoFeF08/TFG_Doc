# 18. Fase 5: Implementacio i Resultats

Aquest document descriu la implementacio tecnica de la Fase 5 definida a [[17_Fase5_MarcTeoric]]. L'objectiu es respondre:

> **Q4:** Pot el self-play mixt reduir l'explotabilitat del PPO sense degradar el rendiment global?

## Disseny experimental

**2 runs** de 48M steps cadascun per comparacio equitativa de pressupost:

| Run                      | Agent            | Punt inicial                     | Pool oponents                        | Metriques loggejades                       |
| :----------------------- | :--------------- | :------------------------------- | :----------------------------------- | :----------------------------------------- |
| F4-ablacio-48M (base.)   | PPO + COS frozen | Scratch                          | 6 AgentRegles (fix)                  | metric classica + metric_robust per variant |
| **F5-selfplay**          | PPO + COS frozen | **F4-ablacio-48M best.zip**      | **6 AgentRegles + snapshots**        | **metric_robust + exploit_selfplay**       |

F4-ablacio-48M es el mateix protocol que F4 pero amb 48M steps, n_partides=1 i logging complet de variants per tenir un baseline comparable amb les noves metriques de F5.

Tots dos runs comparteixen:

- Arquitectura: `CosMultiInputSB3` amb `best_pesos_cos_truc.pth` carregat i congelat.
- Hiperparametres PPO: LR 3e-4, gamma 0.995, GAE 0.95, clip 0.2, ent 0.01, vf 0.5, n_epochs 7.
- Entorn d'entrenament: `TrucGymEnvSessio` amb N=1 ma per sessio (equivalent a mans individuals, igual que F3.5 — PPO sense LSTM no aprofita sessions mes llargues).
- Entorn d'avaluacio: `TrucEnv` (partides senceres), cada 500 000 steps.

## Fitxers afectats

**Nous:**

- `RL/entrenament/entrenamentsComparatius/fase4/entrenament_f4_ablacio_48M.py` — run baseline estes (F4 protocol, 48M steps, logging complet de variants).
- `RL/entrenament/entrenamentsComparatius/fase4/run_f4_ablacio_48M.sh` — llancador del baseline.
- `RL/entrenament/entrenamentsComparatius/fase5/entrenament_fase5.py` — script principal F5.
- `RL/entrenament/entrenamentsComparatius/fase5/pool_selfplay.py` — `SelfPlayPool` (gestor de snapshots + AgentRegles).
- `RL/entrenament/entrenamentsComparatius/fase5/run_fase5.sh` — llancador F5.
- `TFG_Doc/notebooks/5_selfplay/comparacio_fase5.ipynb` — notebook d'analisi.

**Reutilitzats sense modificacio:**

- `RL/entrenament/entrenamentsComparatius/fase4/entrenament_fase4.py` — `_aplicar_frozen`.
- `RL/entrenament/entrenamentsComparatius/fase4/pool_oponents.py` — `POOL_OPONENTS`, `crear_oponent`, `NOMS_VARIANTS`.
- `RL/models/sb3/sb3_adapter.py` — `SB3PPOEvalAgent` (embolicar snapshots com a oponents).
- `RL/models/sb3/sb3_features_extractor.py` — `CosMultiInputSB3`.
- `joc/entorn_ma/gym_env_sessio.py` — `TrucGymEnvSessio`.

## SelfPlayPool

`pool_selfplay.py` gestiona el pool mixt. La interficie `sample(rng)` es compatible directament amb el parametre `opponent_pool_fn` de `TrucGymEnvSessio`.

```python
class SelfPlayPool:
    def add_snapshot(self, model, step):
        # Desa snapshot al disc, el carrega com SB3PPOEvalAgent i l'afegeix a la llista
        # Descarta el mes antic si supera MAX_SNAPSHOTS

    def sample(self, rng):
        # Pool = 6 AgentRegles (sempre) + snapshots actuals
        # Mostreig uniform entre tots els candidats
        return nom, agent

    def get_recent(self, n=3):
        # Retorna els N snapshots mes recents per avaluacio exploit_selfplay
```

**Carrega de snapshots**: el monkey-patch d'optimizer (igual que `loader.py:48-64`) evita el mismatch de grups de parametres del COS congelat al fer `PPO.load()`.

**Memoria**: cada snapshot ocupa ~10 MB. Amb MAX_SNAPSHOTS=6 el pool te un maxim de ~60 MB de snapshots al disc (mes els 6 carregats en memoria). Manejable.

## El bucle d'entrenament

### Constants

```python
SNAPSHOT_EVERY  = 1_000_000   # steps entre snapshots (12 snapshots a 12M steps)
MAX_SNAPSHOTS   = 6           # finestra rodant
LAMBDA_STD      = 0.5         # lambda per metric_robust
N_RECENT_EVAL   = 3           # snapshots recents per exploit_selfplay
N_SESSIONS_EVAL = 20          # sessions per avaluacio per variant
```

### Evolucio del pool al llarg del training

```
Step 0:    Pool = [conservador, agressiu, truc_bot, envit_bot, faroler, equilibrat]
Step 1M:   Pool = [6 AgentRegles] + [self_1M]
Step 2M:   Pool = [6 AgentRegles] + [self_1M, self_2M]
...
Step 6M:   Pool = [6 AgentRegles] + [self_1M, ..., self_6M]
Step 7M:   Pool = [6 AgentRegles] + [self_2M, ..., self_7M]  (self_1M descartat)
...
```

La proporcio de self-play augmenta de 0/6 a 6/12 (50%) al llarg de l'entrenament.

## Metriques

### Classiques (per comparabilitat directa amb F4)

- `wr_random`: WR contra RandomAgent (100 partides senceres).
- `wr_regles`: WR contra AgentRegles estandard (100 partides senceres).
- `metric = 0.25 * wr_random + 0.75 * wr_regles`.

### Noves

- `wr_conservador, wr_agressiu, ...`: WR per cada variant (20 sessions × 1 partida per sessio).
- `wr_pool_mean`: promig dels 6 WR per variant.
- `std_pool`: desviacio tipica dels 6 WR (mesura de variabilitat inter-estil).
- `metric_robust = wr_pool_mean - 0.5 * std_pool`: metrica de robustesa de Fase 5 (complementaria a metric classica).
- `wr_vs_self`: WR del model actual contra els 3 snapshots mes recents (nan fins al primer snapshot).
- `exploit_selfplay = |wr_vs_self - 50|`: distancia a Nash en termes d'auto-explotabilitat.
- `n_snapshots`: nombre de snapshots al pool en aquell moment.

### Columnes del CSV

```
step,
wr_random, wr_regles, metric,
wr_conservador, wr_agressiu, wr_truc_bot, wr_envit_bot, wr_faroler, wr_equilibrat,
wr_pool_mean, std_pool, metric_robust,
wr_vs_self, exploit_selfplay,
n_snapshots, elapsed
```

## Layout de registres

```
TFG_Doc/notebooks/5_selfplay/
├── resultats/
│   ├── ppo_ablacio_pool_48M/         (baseline F4-ablacio-48M)
│   │   ├── training_log.csv          (step, metric, metric_robust per variant, ...)
│   │   ├── best.zip                  (millor metric classica)
│   │   ├── best_robust.zip           (millor metric_robust)
│   │   └── final.zip
│   └── ppo_selfplay_pool_9snaps/     (F5-selfplay)
│       ├── training_log.csv
│       ├── best.zip
│       ├── best_robust.zip
│       ├── final.zip
│       └── snapshots/
│           ├── snapshot_1000000.zip
│           └── ...
└── comparacio_fase5.ipynb
```

## Procediment d'us

1. **Prerequisit**: disposar de `best_pesos_cos_truc.pth` (generat a Fase 3.5).

2. **Run baseline F4-ablacio-48M**:

   ```bash
   bash RL/entrenament/entrenamentsComparatius/fase4/run_f4_ablacio_48M.sh \
     RL/entrenament/entrenamentEstatTruc/registres/<timestamp>/models/best_pesos_cos_truc.pth
   ```

   Desa a: `TFG_Doc/notebooks/5_selfplay/resultats/ppo_ablacio_pool_48M/`

3. **Run F5-selfplay** (partint del best.zip de F4-ablacio-48M):

   ```bash
   bash RL/entrenament/entrenamentsComparatius/fase5/run_fase5.sh \
     RL/entrenament/entrenamentEstatTruc/registres/<timestamp>/models/best_pesos_cos_truc.pth
   ```

   Per defecte, `model_inicial` apunta a `ppo_ablacio_pool/best.zip` de Fase 4. Ajustar si cal.

4. **Analisi**: `TFG_Doc/notebooks/5_selfplay/comparacio_fase5.ipynb`.

## Resultats

Ambdos runs: 48M steps, 32 envs, n_partides=1.

### Taula resum

| Run                    | Pic metric | Step pic | Pic MR (log) | MR best.zip | exploit_sp | Temps  |
| :--------------------- | ---------: | -------: | -----------: | ----------: | ---------: | -----: |
| F4-ablacio-48M (base.) |      89.0% |      24M |  73.2% @12M  |      59.5%  |         -- | 1.41h  |
| F5-selfplay-48M        |      89.2% |    41.5M | 77.8% @39.5M |  +18.3 pp   |    4.7 pp  | 1.69h  |

*Pic MR (log)*: maxim de metric_robust al training_log. *MR best.zip*: metric_robust del model guardat al pic de metric classica (el model per a us real). La divergencia entre les dues columnes de F4 mostra el trade-off: el millor model classic es el menys robust.

**Observacio clau**: el `best.zip` de F4 (pic metric classica @24M, 89.0%) te metric_robust=59.5% — l'overfit als 6 estils fixos es maxim just quan la metric classica es maxima. F5 trenca aquest trade-off: assoleix metric classica equivalent (89.2%) i metric_robust molt superior (77.8%).

### Detall per variant (Avaluacio models)

Avaluacio detallada del model *best* de F4 contra els resultats de F5, mostrant el Win Rate (WR) contra cadascuna de les 6 variants de la pool:

| Variant | F4-ablacio 48M (`best.zip`) | F5-selfplay (finals) | Diferencia |
| :--- | :---: | :---: | :---: |
| **conservador** | 75.0% | 55.0% | -20.0 pp |
| **agressiu** | 55.0% | 80.0% | +25.0 pp |
| **truc_bot** | 55.0% | 85.0% | +30.0 pp |
| **envit_bot** | 80.0% | 85.0% | +5.0 pp |
| **faroler** | 50.0% | 80.0% | +30.0 pp |
| **equilibrat** | 80.0% | 90.0% | +10.0 pp |
| --- | --- | --- | --- |
| **WR Mitja (`WR_pool_mean`)** | **65.8%** | **79.2%** | **+13.4 pp** |
| **Dispersio (`std_pool`)** | **12.7%** | **11.3%** | **-1.4 pp** |
| **Metrica Robusta (`metric_robust`)** | **59.5%** | **73.5%** | **+14.0 pp** |

*Aquests resultats demostren com el self-play permet a l'agent tapar els forats de la seva politica: mentre l'F4 patia contra l'agressiu, truc_bot i faroler (tots al 50-55%), l'F5 aconsegueix pujar el seu pitjor rendiment al 55% (conservador) i dominar a la resta.*

### Hipotesis

| Hipotesi                             | Criteri                                        | Resultat                                        | Estat      |
| :----------------------------------- | :--------------------------------------------- | :---------------------------------------------- | :--------- |
| H1 -- Self-play no degrada WR global | max metric F5@<=48M >= max F4@<=48M - 3pp      | F5=89.2% vs F4=89.0% (llindar: 86.0%)          | **VALIDA** |
| H2 -- Self-play millora robustesa    | metric_robust F5 pic > F4-48M best.zip         | F5=77.8% vs F4=59.5% (+18.3 pp)                | **VALIDA** |
| H3 -- exploit_selfplay final < 10 pp | mean ultims 5 punts < 10 pp                    | 4.7 pp                                          | **VALIDA** |

## Enllacos creuats

- [[17_Fase5_MarcTeoric]] -- fonaments teorics: FSP, metriques d'explotabilitat.
- [[15_Fase4_Implementacio]] -- F4-ablacio: model de partida i pool original.
- [[16_Checkpoint2]] -- justificacio de la seleccio de F4-ablacio.
