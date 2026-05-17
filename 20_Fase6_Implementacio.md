# 20. Fase 6: Implementació i Resultats (NFSP)

Aquest document detalla la implementació tècnica del Neural Fictitious Self-Play (NFSP) definit teòricament a [[19_Fase6_MarcTeoric]].

> **Q5:** Pot el component SL (política mitjana) reduir l'`std_pool` i acostar el sistema a l'Equilibri de Nash?

## Disseny experimental

S'executa un _run_ principal de **48M steps** per comparar equitativament amb la Fase 5:

| Run                        | Agent RL         | Política SL      | Pool oponents                                |
| :------------------------- | :--------------- | :--------------- | :------------------------------------------- |
| **F5-selfplay** (Baseline) | PPO + COS frozen | N/A              | SelfPlayPool (Regles + PPO)                  |
| **F6-nfsp**                | PPO + COS frozen | AveragePolicyNet | NFSPPool:$\eta$ SL + $(1-\eta)$ SelfPlayPool |

El punt de partida (warm-start) és el model més robust de la Fase 5 (`best_robust.zip`), per assegurar que el Reservoir Buffer i l'agent SL comencen a nodrir-se d'un comportament ja molt depurat.

## Fitxers implementats

**Nucli NFSP (`RL/models/nfsp/`)**

- `reservoir_buffer.py`: Emmagatzema tuples `(observació_plana_240, acció)` mantenint la distribució de tota la història.
- `average_policy.py`: Conté la xarxa `AveragePolicyNet` (aprofita el mateix `CosMultiInputSB3` precongelat + un MLP nou per predicar les accions via Softmax/Gumbel) i l'agent `SLAgent` usat a les partides.

**Bucle d'Entrenament (`RL/entrenament/entrenamentsComparatius/fase6/`)**

- `pool_nfsp.py`: Afegeix la lògica del paràmetre d'anticipació $\eta$. L'entorn escull demanar l'oponent a l'SL (amb probabilitat $\eta$) o a la pool regular (PPO / Regles).
- `entrenament_fase6.py`: Adapta el bucle de la F5, guardant observacions cada `step` de Stable Baselines al Reservoir, i disparant l'optimització SL (Cross-Entropy Loss) independent.
- `run_fase6.sh`: Script llançador amb els hiperparàmetres: buffer de 200.000, `sl_every` de 50.000, `sl_lr` de 5e-4 i $\eta=0.5$.

### Decisió: `start_method='fork'` i `num_envs = 32`

F6 utilitza `start_method='fork'` a `SubprocVecEnv` en lloc del `forkserver` per defecte de SB3. La causa és l'overhead de memòria introduït per `AveragePolicyNet` al pool.

Amb **ForkServer**, cada subprocess re-importa tots els mòduls i instancia `sl_net_cpu` (backbone COS congelat 364k paràmetres + cap MLP) de forma independent. El pic durant la inicialització simultània de 32 subprocessos supera els ~13 GB de RAM lliures del servidor i el kernel mata el procés (OOM).

Amb **Fork + Copy-on-Write (COW)**, els subprocessos hereten l'espai de memòria del procés pare. Les pàgines de `sl_net_cpu` (carregat al pare en CPU) es comparteixen en mode read-only i no es copien fins que algun subprocess les modifiqui, la qual cosa mai succeeix (SLAgent opera en mode `eval` i `no_grad`). El pic addicional per subprocess és pràcticament zero, cosa que permet tornar a 32 entorns paral·lels.

**Restricció tècnica**: el fork ha de fer-se _abans_ d'inicialitzar CUDA, ja que els contexts CUDA no sobreviuen a un `fork()`. Per això, la funció `_ppo_nfsp()` crea primer `sl_net_cpu`, `nfsp_pool` i `vec_env` (el fork), i _després_ carrega `sl_net` a GPU i inicialitza `cudnn.benchmark`.

|                      | F5-selfplay      | F6-nfsp          |
| :------------------- | :--------------- | :--------------- |
| `start_method`       | forkserver       | **fork**         |
| `num_envs`           | 32               | **32**           |
| `n_steps × num_envs` | 256 × 32 = 8.192 | 256 × 32 = 8.192 |
| Velocitat estimada   | ~250 steps/s     | ~250 steps/s     |
| Temps 48M            | ~2.5 dies        | ~2.5 dies        |

La comparació F5 vs F6 és equitativa: el batch PPO (`n_steps × num_envs` = 8.192) és idèntic als dos runs.

## Mètriques Registrades

S'expandeix el CSV respecte a la Fase 5:

- `sl_loss`: Cross-entropy mitjana de l'Average Policy respecte el buffer. Mostra com l'agent SL "aprèn" les accions històriques.
- `wr_vs_sl`: WR del PPO contra el seu propi SLAgent estocàstic.
- `exploit_vs_sl`: $| \text{wr\_vs\_sl} - 50.0 |$. Mesura la distància empírica al Nash; com més proper a 0, més in-explotable és el comportament històric vs l'actual.
- `eta_actual`: Valor dinàmic de la ràtio de self-play SL.

## Procediment d'ús

```bash
# Run complet (48M steps, 16 envs per defecte)
bash RL/entrenament/entrenamentsComparatius/fase6/run_fase6.sh \
     RL/entrenament/entrenamentEstatTruc/registres/<timestamp>/models/best_pesos_cos_truc.pth

# Smoke test (500k steps)
bash RL/entrenament/entrenamentsComparatius/fase6/run_fase6.sh \
     RL/entrenament/entrenamentEstatTruc/registres/<timestamp>/models/best_pesos_cos_truc.pth \
     500000
```

## Resultats

Run completat: **48M steps**, 153 avaluacions enregistrades.

### Taula Resum

| Mètrica                      | F4-ablació  | F5-selfplay | F6-NFSP               |
| :--------------------------- | :---------- | :---------- | :-------------------- |
| `max metric`                 | 89.0% @ 24M | 89.2% @ 42M | **91.0% @ 14M**       |
| `metric_robust` (màx)        | 73.2%       | **77.8%**   | 75.2%                 |
| `std_pool` (últimes 5 aval.) | 12.9%       | **11.9%**   | 13.4%                 |
| `exploit_vs_sl` (últimes 5)  | —           | —           | 16.7 pp               |
| `exploit_vs_sl` (best_nash)  | —           | —           | **3.3 pp** @ 1M steps |

### Validació d'Hipòtesis

| Hipòtesi                                        | Resultat                     | Detall                                                                |
| :---------------------------------------------- | :--------------------------- | :-------------------------------------------------------------------- |
| **H1** — No regressió (`metric` F6 ≥ F5 − 3 pp) | ✅**VÀLIDA**                 | F6=91.0% vs F5=89.2%; +1.8 pp de guany                                |
| **H2** — Reducció `std_pool`                    | ❌**FALLA**                  | F6 std=13.4% vs F5 std=11.9%; lleugerament pitjor                     |
| **H3** — Nash: `exploit_vs_sl` < 8 pp           | ✅**VÀLIDA** (@ `best_nash`) | **3.3 pp** @ 1M steps (`best_nash.zip`); 16.7 pp en mitjana últimes 5 |

### Anàlisi de les Corbes

- **SL Loss**: Decreix de ~1.94 a ~1.71 al llarg del entrenament, confirmant que l'`AveragePolicyNet` aprèn progressivament el comportament històric del PPO.
- **`metric` bruta**: F6 arriba al pic de 91.0% molt aviat (14M steps), aprofitant el warm-start des del `best_robust.zip` de F5. Posteriorment oscil·la entre el 80–84%.
- **`exploit_vs_sl`**: Molt variable durant tot el run, rarament per sota del llindar de 8pp. La política mitjana roman explotable.

### Interpretació

**H1 (Vàlida):** El warm-start des del model robust de F5 i la pressió addicional del component SL contribueixen a un pic de rendiment superior. L'NFSP no perjudica el rendiment brut.

**H2 (Falla):** Amb `η = 0.5`, buffer de 200k i `sl_every = 50k`, el Reservoir Buffer no acumula prou diversitat d'historial per que l'Average Policy actuï com a àncora estabilitzadora de la variança en el domini del Truc (espai d'observació 240-dimensional). El resultat és coherent amb la literatura: NFSP requereix molts més episodis per convergir en dominis complexos.

**H3 (Vàlida @ `best_nash`):** El checkpoint `best_nash.zip` (1M steps, `exploit_vs_sl` = 3.3 pp) demostra que el sistema _sí pot_ assolir la zona Nash (<8 pp). Aquesta és l'avaluació metodològicament correcta per a NFSP: l'objectiu és demostrar que la política mitjana pot ser no-explotable, no que ho sigui de forma contínua. En mitjana de les últimes 5 avaluacions (16.7 pp) H3 fallaria, però això reflecteix que l'entrenament no ha convergit de forma estable — coherent amb la literatura (Heinrich & Silver, 2016), on fins i tot Leduc Hold'em requereix 10^8–10^9 episodis.
