# Resum Entorn

**Josep Ferriol Font**


## Índex Modular (Obsidian)

Aquesta documentació està estructurada en mòduls temàtics formant un graf interactiu. Els documents s'organitzen seguint la lectura natural del projecte: primer l'arquitectura general, després la lògica del joc i els entorns RL, després els models disponibles, i finalment les fases experimentals (Fase 1 — comparativa d'algorismes, Fase 2 — curriculum learning, Fase 3/3.5 — feature extractor preentrenat, Fase 4 — memòria recurrent, Fase 5 — self-play amb pool, Fase 6 — NFSP).

- [[1_Arquitectura_MVC]]: Arquitectura MVC del joc interactiu, contractes de Vista i Model.
- [[2_Logica_Joc]]: Motor lògic del joc (`TrucGame` i `TrucGameMa`) i sistema de *reward shaping*.
- [[3_Entorns_Simulacio_RL]]: Adaptadors RLCard (`TrucEnv`, `TrucEnvMa`) i wrappers Gymnasium per SB3 (`TrucGymEnv`, `TrucGymEnvMa`).
- [[4_Estructura_Models]]: Organització de `RL/models/` (`core/`, `rlcard_legacy/`, `sb3/`, `model_propi/`) i en profunditat l'`AgentRegles` estocàstic.
- [[5_Fase1_Entrenament]]: Comparativa d'algorismes DQN-RLCard / NFSP-RLCard / DQN-SB3 / PPO-SB3 amb condicions homogènies.
- [[6_Fase1_Resultats]]: Resultats dels experiments de la Fase 1 (per steps fixos i per temps fix).
- [[7_Fase2_MarcTeoric]]: Fase 2 — Marc teòric: entorn per mans (`TrucGameMa`/`TrucEnvMa`/`TrucGymEnvMa`) i el concepte de *curriculum learning* aplicat al Truc.
- [[8_Fase2_Implementacio]]: Fase 2 — Implementació i resultats: disseny experimental (control vs curriculum), scripts d'entrenament i anàlisi de resultats.
- [[9_Checkpoint1]]: Tancament de Fase 1 i Fase 2 — conclusions, decisió dels dos models (DQN-SB3 i PPO-SB3) que continuen a Fase 3.
- [[10_Fase3_MarcTeoric]]: Fase 3 — Marc teòric: integració de `CosMultiInput` preentrenat als models SB3 i els tres règims (scratch / frozen / finetune).
- [[11_Fase3_Implementacio]]: Fase 3 — Implementació: embolcall `CosMultiInputSB3`, script `entrenament_fase3.py` i disseny de les 6 execucions comparatives.
- [[12_Fase35_MarcTeoric]]: Fase 3.5 — Marc teòric: estratègia d'inicialització del COS preentrenat (`preentrenar_cos.py`).
- [[13_Fase35_Implementacio]]: Fase 3.5 — Implementació i resultats de la inicialització del COS.
- [[14_Fase4_MarcTeoric]]: Fase 4 — Marc teòric: memòria d'oponent amb LSTM i sessions multi-mà (`TrucGymEnvSessio`).
- [[15_Fase4_Implementacio]]: Fase 4 — Implementació: `RecurrentPPO`, pool d'oponents (`pool_oponents.py`) i ablació a 48M steps.
- [[16_Checkpoint2]]: Tancament de Fase 3, Fase 3.5 i Fase 4 — conclusions i elecció del model per a desplegament.
- [[17_Fase5_MarcTeoric]]: Fase 5 — Marc teòric: self-play mixt i explotabilitat.
- [[18_Fase5_Implementacio]]: Fase 5 — Implementació: pool de snapshots (`pool_selfplay.py`) i mètrica `metric_robust`.
- [[19_Fase6_MarcTeoric]]: Fase 6 — Marc teòric: NFSP (política mitjana supervisada + best response PPO) i exploitability.
- [[20_Fase6_Implementacio]]: Fase 6 — Implementació: `entrenament_fase6.py`, `AveragePolicyNet`, `ReservoirBuffer` i resultats finals.

---

## Resum de l'Estructura

El projecte **TFG-truc** implementa el joc de cartes Truc amb una arquitectura modular preparada per a l'entrenament d'agents de Reinforcement Learning (RL). S'utilitzen tant els algorismes de la llibreria **RLCard** (DQN, NFSP) com els de **Stable-Baselines3** (DQN, PPO), comparant-los en igualtat de condicions.

### Visió General

L'objectiu principal és proporcionar un entorn robust per simular partides de Truc i entrenar agents intel·ligents, comparant algorismes clàssics i investigant l'efecte del *curriculum learning* (entrenar primer en mans individuals i després en partides senceres) sobre la seva convergència.

### Estructura de Directoris

- `joc/`: Nucli del joc sota una arquitectura MVC.
  - `entorn/`: Motor de simulació per als agents (partides senceres, Single-Agent).
    - `game.py`: Motor lògic del joc (`TrucGame`). Gestiona estats, regles, rondes, mans, apostes (Truc i Envit) i *reward shaping*.
    - `env.py`: Adaptador RLCard (`TrucEnv`). Tradueix l'estat a tensors (6,4,9) + (24,).
    - `gym_env.py`: Wrapper Gymnasium (`TrucGymEnv`) per utilitzar els entorns amb Stable-Baselines3 (SB3).
    - `cartes_accions.py`: Constants compartides (cartes, llistat de 19 accions, senyals).
    - `rols/`: `dealer.py`, `judger.py`, `player.py`.
  - `entorn_ma/`: Variant de l'entorn per mans individuals (1 episodi = 1 mà).
    - `game_ma.py`: Motor lògic per mans (`TrucGameMa`). Reward net normalitzat final de mà.
    - `env_ma.py`: Adaptador RLCard per mans (`TrucEnvMa`).
    - `gym_env_ma.py`: Wrapper Gymnasium per mans (`TrucGymEnvMa`).
    - `gym_env_sessio.py`: Wrapper de sessions multi-mà (`TrucGymEnvSessio`, 1 episodi = N mans) per a la memòria cross-mans de Fase 4.
    - `parallel_env_ma.py`: Versió paral·lela heretada (no utilitzada al pipeline actual, que depèn de `SubprocVecEnv` de SB3).
  - `controlador/`: Gestors i classes de control (arquitectura MVC).
  - `vista/`: Interfícies gràfiques (consola i escriptori, amb recursos a `img_iu/`).
- `RL/`: Flux de treball de Reinforcement Learning.
  - `models/`: Arquitectures i agents.
    - `core/`: `feature_extractor.py` (`CosMultiInput`, `ModelPreEntrenament`), `loader.py`.
    - `rlcard_legacy/`: `model_adapter.py` (wrapper per connectar agents RLCard amb el nostre loader).
    - `sb3/`: `sb3_adapter.py` (`SB3PPOEvalAgent`), `sb3_features_extractor.py` (`CosMultiInputSB3`), `sb3_lstm_eval_agent.py` (`SB3LSTMEvalAgent` per a models recurrents).
    - `nfsp/`: `average_policy.py` (`AveragePolicyNet`, `SLAgent`), `reservoir_buffer.py` (mostreig reservoir per a NFSP, Fase 6).
    - `model_propi/`: `agent_regles.py` (agent Rule-Based estocàstic).
  - `tools/`: `obs_utils.py` (font única de `flatten_obs`), `exportar_pesos.py`.
  - `entrenament/`:
    - `entrenamentEstatTruc/`: Preentrenament supervisat del "Cos" CNN+MLP (`preentrenar_cos.py`).
    - `entrenamentsComparatius/fase1/`: Script de la Fase 1 (`entrenament_comparatiu.py`) i llançadors bash.
    - `entrenamentsComparatius/fase2/`: Scripts de Fase 2 (`entrenament_fase2_curriculum.py`, etc.) i llançadors bash.
    - `entrenamentsComparatius/fase3/`: Scripts de Fase 3 (`fase30/entrenament_fase3.py`) i Fase 3.5 (`fase35/entrenament_fase35.py`).
    - `entrenamentsComparatius/fase4/`: Scripts de Fase 4 (`entrenament_fase4.py`, ablació 48M) i `pool_oponents.py`.
    - `entrenamentsComparatius/fase5/`: Script de Fase 5 (`entrenament_fase5.py`) i `pool_selfplay.py`.
    - `entrenamentsComparatius/fase6/`: Script de Fase 6 (`entrenament_fase6.py`) i `pool_nfsp.py`.
- `demo.py`: Script de demostració interactiu per jugar una partida (humà vs bot).
- `TFG_Doc/`: Documentació teòrica i llibretes de resultats.
  - `notebooks/1_comparacio_inicial/`: Notebook i carpetes `resultats_fase1_*`.
  - `notebooks/2_curriculum_learning/`: Notebook Fase 2.
  - `notebooks/3_feature_extractor/`: Notebooks Fase 3 (`30_comparacio_sense/`) i Fase 3.5 (`35_init_cos/`).
  - `notebooks/4_memoria/`: Notebook Fase 4 i resultats de l'ablació.
  - `notebooks/5_selfplay/`: Notebook Fase 5.
  - `notebooks/6_nfsp/`: Notebook Fase 6.
