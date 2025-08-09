# Protocole calculatoire complet (version : rôle dépendant du type de devoir)

Ce document décrit **un protocole purement calculatoire** (sans code) qui prend en entrée : *type de devoir (T), jours restants (D), difficulté (Diff), importance (I)* et retourne un planning chiffré : nombre de sessions, jours relatifs (J‑x), rôle pédagogique de chaque session et durée (minutes). Toutes les étapes sont déterministes et peuvent être appliquées à la main ou traduites en code.

---

# 1. Variables d’entrée (à fournir)

- **T** ∈ {`bilan`, `connaissance`, `DM`, `exercice`} — type de devoir
- **D** = nombre de jours restants avant la date limite (entier ≥ 0)
- **Diff** ∈ {`simple`, `difficile`, `tres`, `extreme`} — difficulté
- **I** ∈ {1,2,3} — importance de la matière (1 = faible, 2 = normale, 3 = élevée)

---

# 2. Constantes et mappages numériques

## 2.1 Facteur difficulté (nombre et durée)

```
F_diff_sessions =
  simple   = 1.00
  difficile= 1.20
  tres     = 1.50
  extreme  = 2.00
```

## 2.2 Facteur importance (durée)

```
F_importance =
  I=1 → 0.90
  I=2 → 1.00
  I=3 → 1.10
```

## 2.3 Durées de base (minutes) selon rôle pédagogique

```
Base_minutes =
  fiche    = 30
  exercices= 45
  test     = 60
  synthese = 75
  planif   = 40   (pour DM ou session de planification)
```

## 2.4 Nombre minimal de sessions (avant facteurs)

```
Base_sessions_by_T =
  bilan         = 3
  connaissance  = 2
  DM (simple)   = 1
  DM (difficile)= 2
  DM (tres/extreme) = max(3, ceil(D/5))
  exercice (simple/difficile) = 1
  exercice (tres/extreme)     = 2
```

## 2.5 Règles d’arrondi et limites

- Toutes les valeurs de sessions → arrondir **au supérieur** (ceil).
- Minimum sessions = 1, maximum raisonnable = 12 (clip si besoin).

---

# 3. Nouvelle table d’attribution des rôles (T + t\_k)

Le rôle pédagogique dépend désormais **à la fois** du type `T` et de la distance `t_k` (jours avant échéance).

| Type `T`         | t\_k > 20 j                  | 7 < t\_k ≤ 20 j                  | 3 < t\_k ≤ 7 j              | t\_k ≤ 3 j           |
| ---------------- | ---------------------------- | -------------------------------- | --------------------------- | -------------------- |
| **bilan**        | fiche (résumer chapitres)    | exercices (pratique ciblée)      | test (simulation)           | synthese (mix final) |
| **connaissance** | fiche (apprentissage)        | fiche + exercices (mix)          | test rapide (QCM / oral)    | synthese courte      |
| **DM**           | planif (organisation, idées) | rédaction partielle / recherches | rédaction intensive + vérif | relecture / synthese |
| **exercice**     | fiche méthode                | exercices (correction)           | test d’application          | synthese rapide      |

**Notes sur la table** :

- `fiche + exercices` signifie soit **division** de la session en deux mini‑blocs, soit alternance sur deux sessions proches.
- La **première** session d’un `DM` est toujours `planif` (priorité à l’organisation), quelle que soit la tranche.
- La **dernière** session (t\_k ≤ 3) peut être forcée en `synthese` pour clôturer.

---

# 4. Étapes du protocole (A → E)

## Étape A — Calcul du nombre de sessions `N`

1. Choisir la base `B` selon `T` et `Diff` (voir Base\_sessions\_by\_T).
2. Appliquer le facteur difficulté :

```
N_temp = ceil( B * F_diff_sessions[Diff] )
```

3. Ajustement importance :

- Si `I = 3` alors `N_temp = N_temp + 1`.
- Si `I = 1` et `D > 10` alors `N_temp = max(1, N_temp - 1)`.

4. Résultat final :

```
N = clip(N_temp, 1, 12)
```

---

## Étape B — Calcul des jours des sessions (t\_k en jours avant échéance)

On répartit `N` sessions sur l’intervalle `[D .. 0]`.

1. Définir `alpha = 1 / F_diff_sessions[Diff]`.

   - Diff = `simple` → alpha = 1 (uniforme). Diff élevé → alpha < 1 → placement plus précoce.

2. Si `N = 1` :

- `t_1 = round(D / 2)` (session au milieu).

Sinon (N > 1), pour `k = 0..N-1` :

```
frac = k / (N-1)
frac_alpha = frac^(1/alpha)
t_k = round( D * (1 - frac_alpha) )
```

- Bornes : `t_k` ∈ [0, D].
- Fusion : si des `t_k` consécutifs sont identiques, fusionner (somme des minutes ou rapprochement d’un jour) selon préférence.

> Remarque : utiliser `1/alpha` rend la répartition plus précoce quand la difficulté augmente.

---

## Étape C — Attribution du rôle pédagogique pour chaque session (nouvelle règle)

Pour chaque session `k` avec `t_k` :

1. Lire la **table T + t\_k** ci‑dessus pour obtenir le rôle de base.
2. Exceptions :
   - Si `k = 0` (première session) et `T = DM` → rôle = `planif` (toujours).
   - Si plusieurs sessions dans la même tranche → alterner rôles (ex. `exercices ↔ fiche`) ou introduire un `test` si une session est proche (≤7).
   - Si le rôle est mixte (`fiche + exercices`) :
     - Option A : diviser la durée totale de la session en 2 mini‑blocs (par défaut 50/50).
     - Option B : répartir les deux sous‑rôles sur deux sessions proches (si possible).
3. Possibilité de forcer la **dernière** session en `synthese` pour clôturer l’apprentissage.

---

## Étape D — Calcul de la durée (minutes) par session

Durée finale par session :

```
Durée = round_to_5min( Base_minutes[rôle]
           × F_diff_sessions[Diff]
           × F_importance[I]
           × Proximity_factor(t_k, rôle) )
```

### Définition de `Proximity_factor(t, rôle)`

1. Calcul intermédiaire :

```
x = clamp( (7 - t) / 7 , -1, 1 )
```

2. Application selon rôle :

```
Proximity_factor =
  fiche     : 1 + x * (-0.20)
  exercices : 1 + x *  0.00
  test      : 1 + x *  0.20
  synthese  : 1 + x *  0.30
  planif    : 1 + x * (-0.05)
```

3. Clamp `Proximity_factor` entre `0.6` et `1.6`.

4. **Cas de rôle mixte** (`fiche + exercices`) :

   - Calculer la durée séparément pour chaque sous‑rôle puis sommer.
   - Si division 50/50 → appliquer facteurs sur chaque sous‑part proportionnellement.

5. **Arrondi final** : arrondir à la minute multiple de 5 la plus proche.

---

## Étape E — Résumé du planning en sortie

Pour chaque session `k = 1..N`, fournir :

- **Session k :** `J - t_k` (ex : J-18)
- **Rôle :** fiche/exercices/test/synthese/planif (ou mix)
- **Durée (min)** : valeur calculée
- **Objectif court** (2–6 mots) selon rôle (ex. fiche → "résumer chap. 3")

Format machine‑friendly conseillé : liste de tuples

```
(ordre, t_k, rôle, durée_min, objectif_court)
```

---

# 5. Exemple complet (appliquer tout)

**Données** : `T = bilan`, `D = 18`, `Diff = difficile`, `I = 3`.

1. Constantes : F\_diff = 1.20 ; F\_importance = 1.10 ; Base B pour `bilan` = 3.
2. N\_temp = ceil(3 \* 1.20) = 4 ; I=3 → N\_temp = 5 → N = 5.
3. alpha = 1 / 1.20 ≈ 0.8333.
4. Calcul t\_k (k=0..4) → J-18, J-14, J-10, J-5, J-0.
5. Rôles (table) : J-18 → exercices ; J-14 → exercices ; J-10 → exercices ; J-5 → test ; J-0 → synthese.
6. Durées (ex. pour J-18 exercices) : Base 45 ×1.2×1.1×Proximity(−1)=45×1.2×1.1≈59.4 → arrondi 60 min. Calculer toutes comme dans la formule.
7. Résumé final (exemple format) :

```
Session 1 — J-18 — exercices — 60 min — objectif : exercices ciblés
Session 2 — J-14 — exercices — 60 min — objectif : exercices corrigés
Session 3 — J-10 — exercices — 60 min — objectif : problèmes complexes
Session 4 — J-5  — test      — 85 min — objectif : test blanc complet
Session 5 — J-0  — synthese  —130 min — objectif : synthèse finale
```

---

# 6. Options paramétrables et remarques pratiques

- **Fusion** : si deux `t_k` identiques → fusionner et sommer durées (ou ajouter 15–30 min si tu préfères garder petite session distincte).
- **Début impératif** : pour `Diff ∈ {tres, extreme}` tu peux forcer `t_0 = D` (commencer immédiatement) si D>0.
- **Densification finale** : pour plus de sessions proches de la date, multiplier `alpha` par 1.2.
- **Mix de rôles** : par défaut 50/50 pour `fiche + exercices`, mais tu peux définir 60/40 selon `I` (I=3 favorise plus d’exercices).
- **Sortie** : si tu veux un CSV/JSON, je peux produire la conversion machine‑readable.

---

# 7. Checklist d’application manuelle

1. Entrer `T, D, Diff, I`.
2. Calculer `B` → `N_temp` → appliquer importance → obtenir `N`.
3. Calculer `alpha = 1 / F_diff_sessions[Diff]`.
4. Pour k=0..N-1 calculer `t_k` (ou `t_mid` si N=1).
5. Déterminer rôle via table (et exceptions DM / mixités).
6. Calculer Durée via la formule multiplicative + arrondir.
7. Vérifier cohérence (fusion/éclatement si sessions doublon).
8. Produire planning final et objectifs courts.

---

Si tu veux, je peux :

- appliquer immédiatement le protocole à des paramètres précis (`T, D, Diff, I`) et générer un tableau ;
- fournir une version machine‑friendly (JSON ou CSV) ;
- modifier les ratios des rôles mixtes (50/50 → autre).

Dis‑moi ce que tu préfères !

