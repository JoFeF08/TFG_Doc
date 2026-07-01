# 20. Fase 6: Implementació i Resultats (NFSP)

Aquest document descriu l'estat actual de la implementació NFSP de la Fase 6 i com interpretar-ne els resultats.

> **Q5:** Pot el component SL (política mitjana) acostar-se a Nash sense degradar el rendiment robust?

## Disseny experimental (versió actual)

La comparació segueix sent F5 vs F6:

| Run | Agent RL | Política SL | Pool d'oponents |
| :-- | :-- | :-- | :-- |
| F5-selfplay (baseline) | PPO + COS frozen | No | SelfPlayPool (regles + snapshots PPO) |
| F6-nfsp | PPO + COS frozen | Sí (`AveragePolicyNet`) | `NFSPPool`: \(\eta\) SL + \((1-\eta)\) SelfPlayPool |

El warm-start continua essent el model robust de F5 (`best_robust.zip`).

## Fitxers clau

### Nucli NFSP

- `RL/models/nfsp/reservoir_buffer.py`
- `RL/models/nfsp/average_policy.py`

### Entrenament Fase 6

- `RL/entrenament/entrenamentsComparatius/fase6/pool_nfsp.py`
- `RL/entrenament/entrenamentsComparatius/fase6/entrenament_fase6.py`
- `RL/entrenament/entrenamentsComparatius/fase6/run_fase6.sh`

## Actualitzacions grans incorporades

### 1) Criteri `nash_valid` amb gates de qualitat

L'arxiu de `best_nash` no depèn només de minimitzar `exploit_vs_sl`. Ara cal que es compleixi:

- Reservoir ple.
- `step >= nash_min_steps`.
- Política no degenerada (`calib_envit` suficient i bon `wr_envit_bot`).

Això elimina falsos Nash inicials.

### 2) Criteri dual per a `best_nash`

Un punt només millora `best_nash` si:

- baixa `exploit_vs_sl`, i
- puja `calib_combined = (calib_envit + calib_truc)/2`.

És un canvi clau: no s'accepten checkpoints “Nash” amb degradació tàctica.

### 3) Calibració integrada al bucle

S'afegeix monitoratge de calibració durant entrenament (`mesurar_calibracio`) i checkpoints específics:

- `best_calib.zip`
- `best_nash.zip`
- `best_robust.zip`
- `best.zip`

També s'exporta `sl_final.pt`.

### 4) Pausa/resum dinàmic del component SL

Si la calibració d'envit de l'SL cau fort respecte al seu pic, es força temporalment `eta=0` (SL off). Quan es recupera, es reactiva SL. Aquest mecanisme evita contaminar l'entrenament RL amb un SL col·lapsat.

### 5) Early stopping orientat a Nash

Hi ha `nash_patience`: si hi ha diversos punts `nash_valid` seguits sense millora dual de `best_nash`, el run s'atura anticipadament.

## Punts importants (resum operatiu)

### 1) Noves metriques i calcul

- exploit_vs_sl = abs(wr_vs_sl - 50.0)
- metric_robust = wr_pool_mean - 0.5 * std_pool
- calib_combined = (calib_envit + calib_truc) / 2

On:

- wr_vs_sl es el win-rate del PPO actual contra el SLAgent.
- wr_pool_mean es la mitjana de wr_conservador, wr_agressiu, wr_truc_bot, wr_envit_bot, wr_faroler i wr_equilibrat.
- std_pool es la desviacio estandard d'aquests sis win-rates.

### 2) Sistema SL: quan esta actiu i quan es desactiva

El component SL funciona amb una logica adaptativa:

- Si el reservoir te massa poques mostres (len < 10 * SL_BATCH_SIZE), eta efectiva passa a 0.0.
- Si la calibracio envit de SL cau fort respecte al seu pic historic i baixa de 25 pp, es força pausa de SL (eta=0.0).
- Si la calibracio es recupera, SL es reactiva i es restaura eta target.

Aixo evita que una average policy degradada contamini l'entrenament RL.

### 3) Que es el Reservoir Buffer i per que es clau

ReservoirBuffer guarda parelles (obs_240, accio) amb mostreig reservoir:

- capacitat fixa (per defecte 200000),
- probabilitat uniforme sobre tota la historia observada,
- sense biaix FIFO cap a les mostres recents.

Aquest buffer alimenta la Cross-Entropy de SL i es el mecanisme que aproxima la politica mitjana real de NFSP.

### 4) Estructura general del sistema

Arquitectura logica en entrenament:

- PPO (best response) apren contra NFSPPool.
- NFSPPool tria oponent: SLAgent amb probabilitat eta, o SelfPlayPool amb 1-eta.
- SelfPlayPool conte regles + snapshots PPO.
- SLAgent inferix amb AveragePolicyNet, entrenada sobre ReservoirBuffer.
- Totes dues branques reutilitzen COS frozen per coherencia representacional.

### 5) Early stopping: criteri exacte

Early stopping no es basa en la metrica classica, sino en estabilitat Nash valida:

- un punt nomes compta si nash_valid=1,
- best_nash millora nomes si baixa exploit_vs_sl i puja calib_combined a la vegada,
- si hi ha nash_patience evals valides consecutives sense aquesta millora dual, el run s'atura.

Aquest disseny evita parar en falsos minims d'explotabilitat que venen amb degradacio tactica.

## Decisions tècniques de rendiment

### `start_method='fork'`

Es manté `fork` per compartir la xarxa SL CPU via Copy-on-Write entre subprocessos i evitar OOM de `forkserver`.

### Ordre de creació CPU/GPU

Primer es crea `vec_env` (fork), després es mou la xarxa SL d'entrenament a GPU. Això evita problemes de context CUDA amb `fork`.

## Hiperparàmetres i defaults actuals

### Defaults de `entrenament_fase6.py`

- `steps=12_000_000`
- `num_envs=NUM_ENVS_PPO`
- `n_partides=5`
- `eta=0.5`
- `reservoir_cap=200000`
- `sl_lr=5e-4`
- `sl_every=50000`
- `nash_min_steps=10_000_000`
- `nash_patience=3`
- `hidden_layers=[256, 256]`

### Defaults del launcher `run_fase6.sh`

- `STEPS=80000000`
- `NUM_ENVS=32`
- `n_partides=1`
- `nash_min_steps=8000000`
- `nash_patience=8`
- `ent_coef=0.05`

Nota: el launcher i l'script Python tenen defaults diferents; el run real el determinen els arguments del launcher.

## Esquema de log actual (`training_log.csv`)

Columnes rellevants:

- Rendiment clàssic: `metric`, `wr_random`, `wr_regles`.
- Robustesa: `metric_robust`, `std_pool`, `wr_{variant}`.
- Self-play: `wr_vs_self`, `exploit_selfplay`.
- NFSP/SL: `sl_loss`, `wr_vs_sl`, `exploit_vs_sl`, `eta_actual`.
- Qualitat/Nash: `calib_envit`, `calib_truc`, `nash_valid`, `evals_sense_millora`.

## Procediment d'ús (actualitzat)

```bash
# Execució estàndard amb launcher (usa defaults del .sh)
bash RL/entrenament/entrenamentsComparatius/fase6/run_fase6.sh \
     RL/entrenament/entrenamentEstatTruc/registres/<timestamp>/models/best_pesos_cos_truc.pth

# Smoke test ràpid
bash RL/entrenament/entrenamentsComparatius/fase6/run_fase6.sh \
     RL/entrenament/entrenamentEstatTruc/registres/<timestamp>/models/best_pesos_cos_truc.pth \
     500000
```

## Interpretació de resultats (guia)

Per evitar conclusions esbiaixades, cal separar quatre checkpoints conceptuals:

- `best.zip`: millor `metric` (rendiment brut).
- `best_robust.zip`: millor `metric_robust` (generalització entre estils).
- `best_nash.zip`: millor punt Nash **vàlid** (exploit + calibració).
- `best_calib.zip`: millor calibració tàctica.

Amb aquesta versió de Fase 6, l'avaluació principal recomanada és:

1. H1: no regressió de `metric` contra F5.
2. H2: robustesa mantinguda/millorada (`metric_robust` i `std_pool`).
3. H3: existeix almenys un punt amb `nash_valid=1` i `exploit_vs_sl < 8`.

Aquests criteris són els que implementa també el notebook d'anàlisi de Fase 6.

## Decisions subtils i riscos a tenir en compte

### 1) Eta efectiva vs eta real del pool

Al callback es calcula `eta_efectiva=0.0` quan el reservoir és petit (`len < 10 * SL_BATCH_SIZE`) i aquest valor es registra al log. Però el mostreig real d'oponent el fa `NFSPPool.sample()` segons `nfsp_pool.eta`.

Conseqüència:

- el log pot mostrar `eta_actual=0.0` en trams inicials,
- mentre que el pool encara pot estar fent servir SL si `nfsp_pool.eta > 0`.

Per coherència total, si es vol un "SL realment desactivat" en cold-start, cal forçar també `nfsp_pool.set_eta(0.0)` en aquest tram.

### 2) Semàntica de `best_nash.zip` al final

Al tancament del run, si no existeix cap `best_nash.zip`, el codi desa igualment el model final amb aquest nom. Això evita errors de fitxers absents, però semànticament vol dir que `best_nash.zip` no garanteix haver passat cap gate `nash_valid`.

Recomanació de lectura:

- validar sempre `n_nash_valid` i els punts `nash_valid=1` del CSV,
- interpretar `best_nash.zip` com a "best_nash vàlid" només si hi ha almenys un punt vàlid al log.

### 3) Balanceig del criteri dual

`best_nash` exigeix millora simultània de `exploit_vs_sl` i `calib_combined`. És una decisió conservadora (molt robusta), però pot alentir molt l'actualització de `best_nash` si una de les dues metriques s'estanca.

Pràcticament:

- guanyes qualitat metodològica,
- però incrementes probabilitat d'early-stop per manca de "millora dual" encara que hi hagi millora parcial.

### 4) Snapshoting i cost de diagnosi

El sistema guarda snapshots PPO cada 1M i checkpoints SL periòdics. Això és excel·lent per auditoria, però incrementa IO i espai a disc en runs llargs.

En execucions extensives, convé monitoritzar:

- mida de `snapshots/`,
- mida de `sl_checkpoints/`,
- i freqüència de checkpoints segons necessitats d'anàlisi.
