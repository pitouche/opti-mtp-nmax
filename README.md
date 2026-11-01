# mtp-nmax — `--spec-draft-n-max` {2, 3, 4} (MTP draft-mtp)

**Verdict : n_max=3 gardé (prod). Expérience close 30/10.**

## Question

Le MTP (draft-mtp) génère `n_max` drafts par verify, puis le verify accepte/rejette
chaque position. `n_max=3` est le default. Question : existe-t-il un `n_max`
meilleur (2 ou 4) sur ce modèle (Qwen3.8-27B, shortlist 131 072 i32) ? Le critère
ultime est le **decode tok/s mesuré** (pas le busy verify, pas le wall/verify —
ceux-ci sont affectés par la segmentation artefactuelle de l'outil d'analyse).

## Résultats (prof_prd_profile_20261030.py, 3 runs figés, ctx 112 060 tok, b2048,
KV q8_0, H2 OFF, FA, MTP ON, .so `8035caf4`)

| n_max | run | decode tok/s | × vs OFF (21,6) | acceptance (per-pos) | mean len (tok/verify) | busy verify | wall/verify |
|---|---|---:|---:|---|---:|---:|---:|
| 2 | 135141 | 41,94 (23,8 ms) | ×1,95 | 0,825 (99/120) | 2,65 | 53,1 ms | 69,3 ms |
| **3** | **114948** | **44,18 (22,6 ms)** | **×2,05** | 0,711 (108/152) | **3,12** | 56,3 ms | 79,6 ms |
| 4 | 142401 | 33,86 (29,5 ms) | ×1,57 | 0,701 (115/164) | 3,80 | 63,4 ms | 147,2 ms* |

\* n_max=4 : le « wall/verify 147 ms » et « gap draft→verify 31 ms » sont
**surestimés** par la segmentation artefactuelle de `analyze_prd_profile.py`
(24 verifies détectés pour 155 tok). Le **mesuré 33,86 t/s, −23 % vs n3**, fait foi.

## Pourquoi n3 est le point d'équilibre (le « knee »)

Courbe d'acceptance par position mesurée : **pos1 ≈ 0,90-0,95, pos2 ≈ 0,71-0,77,
pos3 ≈ 0,51-0,61, pos4 ≈ 0,51**.

- **n_max=2 (−5 %)** : retirer une position de verify ne retire que **3,2 ms busy**
  (56,3→53,1, le matvec/attention est batché) et 1 draft de moins (−21 %, ~2,3 ms).
  Le cycle ne tombe que de 13 % (79,6→69,3 ms) alors que E[tok/verify] tombe de
  15 % (3,12→2,65). **Perte d'espérance > gain de cycle → perte nette.**
- **n_max=4 (−23 %)** : le 4ᵉ draft n'apporte que **~51 % d'acceptance** (comme le
  3ᵉ de n3) mais coûte un draft supplémentaire séquentiel (2,3 ms) **et** élargit le
  verify à 5 positions (+7 ms busy). **Gain marginal déficitaire → effondrement.**

## Contenu

```
results/  # DECISION_NMAX_20261030.md, nmax_bench_summary.md + results.jsonl + gputimes.tsv,
          # NMAX2_RUN_CONFIG.md, prof_prd_analysis_20261030-{114948,135141,142401,164453}.md,
          # bootstrap_mtp_profiling_20261028.md, mtp_prof_stepA_verdict_20261028.md,
          # runs/prof-prd-20261030-{114948,135141,142401}/ (artefacts run : results.json, logs, samplers, perf)
bench/  # nmax_bench.py, prof_prd_profile_20261030.py, analyze_prd_profile.py, gguf_mtp_probe.py
```

Le .so de référence est `8035caf4` (T3+vldg, H2 OFF). Les 3 runs changent **un
seul paramètre** : `--spec-draft-n-max` (line ~468 de `prof_prd_profile_20261030.py`).

## Re-validation

1. **Le plus simple** : relire les 3 `runs/prof-prd-20261030-*/results.json` (le
   decode tok/s y est, mesuré par l'API ollama, pas par nsys). Le run 114948
   (n3) est la référence figée.
2. **Re-lancer** : `python3 bench/bench/supervisor_bench.py run bench/prof_prd_profile_20261030.py --detach`
   avec `--spec-draft-n-max 2|3|4` modifié dans le script (line ~468), ~17 min/run,
   3 runs = ~1 h. Comparer le decode tok/s (critère ultime) à 44,18 / 41,94 / 33,86.
3. **NSYS** (diagnostic, non verdict) : les `.nsys-rep`/`.sqlite` sont dans
   `big/nsys-prof-prd/` (meta-repo) ; `analyze_prd_profile.py` pour la décomposition
   (⚠️ attention à la segmentation artefactuelle sur n4).

## Caveats

- Le critère **decode tok/s mesuré** est le seul fiable ; les décompositions
  nsys (busy/wall/gap) sont affectées par la segmentation de l'outil — n'utiliser
  que pour le diagnostic, jamais pour le verdict.
- L'acceptance dépend du **shortlist** (131 072 i32, couverture 99,2 %) et du
  **prompt** (112 060 tok figés). Un autre shortlist ou un autre prompt décale la
  courbe d'acceptance → re-valider n_max si l'un des deux change.
- n_max interagit avec le kernel FA : n_max=3 limite structurellement `rows` à
  ≤ 4 (rows = n_accepted+1), ce qui fait que **T3 (rows 1-8) couvre 100 % des
  verifies** (cf `repos/t3-volta-small/`). Si n_max change, re-vérifier la
  couverture du kernel FA.
