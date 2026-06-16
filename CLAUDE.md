# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Purpose

`TFG_Doc/` is the documentation vault for a TFG (Treball Final de Grau) on Reinforcement Learning applied to the card game *Truc*. It is an [Obsidian](https://obsidian.md) knowledge base with Jupyter notebooks for experiment analysis. All documentation is in **Catalan**.

## File Naming Convention

Markdown files are numbered to define reading order:

- `0_Index_projecte.md` — entry point and navigation guide
- `1_Arquitectura_MVC.md` to `4_Estructura_Models.md` — foundational theory
- `5_` to `20_` — experimental phases (Fase 1–6), each split into `_MarcTeoric` (theory) and `_Implementacio` (results)
- `N_CheckpointM.md` — phase conclusions and decisions about which models proceed

## Notebooks Structure

```
notebooks/
├─ 1_comparacio_inicial/   (Phase 1: algorithm comparison; resultats_fase1_<timestamp>/ dirs)
├─ 2_curriculum_learning/  (Phase 2: curriculum learning; resultats_fase2_*_<timestamp>/ dirs)
├─ 3_feature_extractor/    (Phase 3: CNN feature extractor)
│  ├─ 30_comparacio_sense/ (comparacio_fase30.ipynb)
│  ├─ 35_init_cos/         (comparacio_fase35.ipynb — Phase 3.5)
│  └─ resultats/           (shared by 3 and 3.5: {dqn,ppo}_sb3_{scratch,frozen,finetune,...}/)
├─ 4_memoria/              (Phase 4: LSTM recurrence + opponent pool; resultats/ppo_{complet,ablacio}_pool/)
├─ 5_selfplay/             (Phase 5: self-play pool; resultats/ppo_selfplay_pool_9snaps/ with snapshots/)
├─ 6_nfsp/                 (Phase 6: NFSP; resultats/ppo_nfsp/ with best_nash.zip, sl_final.pt, sl_checkpoints/)
└─ utils/nb_utils.py       (shared plotting/analysis utilities)
```

Phases 1–2 keep one timestamped results dir per training run; phases 3–6 use a single `resultats/` with one subdir per experiment (SB3 `.zip` checkpoints + `training_log.csv`). Analysis notebooks are named `comparacio_fase{N}.ipynb`.

## Experimental Phases Summary

| Phase | Key finding |
|-------|-------------|
| 1 | Baseline: all algorithms plateau at 30–40% without curriculum |
| 2 | Curriculum (hands → full game) boosts PPO-SB3 to 75% (+40 pp) |
| Checkpoint 1 | DQN-SB3 and PPO-SB3 advance; RLCard variants eliminated |
| 3 | CNN feature extractor (`CosMultiInput`) on card grid (6,4,9) |
| 4 | GRU/LSTM memory variants evaluated |
| 5 | Self-play pool improves robustness (`metric_robust` → 77.8%) |
| Checkpoint 2 | See `16_Checkpoint2.md` |
| 6 | NFSP achieves 91.0% win rate @ 14M steps; 3.3 pp Nash gap @ 1M steps |

## Estil de la Memòria LaTeX

Abans de modificar qualsevol fitxer `.tex` de `Memoria/`, llegir **`Memoria/estil.md`**. Conté les regles d'estil obligatòries: ús d'acrònims, puntuació, figures, taules, ordre alfabètic dels acrònims i la taula de primers usos ja fixats.

## Key Documentation Rules

- When any folder is renamed or major refactoring happens in `joc/` or `RL/`, update the corresponding markdown files here (especially `1_Arquitectura_MVC.md` and `4_Estructura_Models.md`).
- The observation structure (card tensor `(6,4,9)` + context vector `(24,)`, flattened to 240 dims) is documented in `3_Entorns_Simulacio_RL.md`. Update it if `feature_extractor.py` changes.
- `Plantilla/` contains the LaTeX thesis (`MemoriaTFG.tex`). Do not modify without explicit instruction.
- `referencies/` stores PDFs of cited papers (PPO, DRQN, Pluribus). Do not delete.

## Obsidian Integration

The vault uses Obsidian with `[[wikilinks]]` for cross-file navigation. Links between markdown files use the `[[FileName]]` syntax without the `.md` extension. When adding new documentation files, follow the numbered prefix convention and add a wikilink entry in `0_Index_projecte.md`.
