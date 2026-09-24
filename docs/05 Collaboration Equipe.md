# Collaboration en Équipe

Ce chapitre couvre les scénarios réels de travail collectif sur un dépôt Git : deux développeurs sur une même branche, une grande équipe qui partage un tronc commun, et les politiques de protection des branches.

---

## Sommaire

- [Scénario 1 — 2 devs sur la même branche](#scénario-1--2-devs-sur-la-même-branche)
- [Scénario 2 — Une grande équipe (10+ devs)](#scénario-2--une-grande-équipe-10-devs)
- [Pourquoi `fetch + rebase` plutôt que `pull`](#pourquoi-fetch--rebase-plutôt-que-pull)
- [Politiques de protection des branches](#politiques-de-protection-des-branches)
- [Résumé des règles d'équipe](#résumé-des-règles-déquipe)

---

## Scénario 1 — 2 devs sur la même branche

**Situation :** Tu travailles avec un collègue sur la même branche `feat/export-csv`. Tu as des modifications en cours non commitées, et ton collègue vient de pousser des changements sur cette branche.

**Problème :** si tu fais `git pull` directement, Git va refuser de mettre à jour si tu as des fichiers modifiés non commités, ou — pire — créer des conflits.

**Solution : Stash → Pull → Stash Apply**

```bash
# 1. Mettre de côté les modifications en cours
git stash

# 2. Récupérer les changements du collègue
git pull origin feat/export-csv

# 3. Réappliquer ses modifications
git stash pop
```

### Ce que fait chaque commande

| Commande | Effet |
|----------|-------|
| `git stash` | Sauvegarde les modifications non commitées (tracked + staged) dans une pile temporaire et remet le working directory propre |
| `git pull` | Récupère et intègre les commits distants dans la branche locale |
| `git stash pop` | Réapplique le dernier stash et le supprime de la pile |

### Si un conflit survient après `stash pop`

```bash
# git stash pop peut créer des conflits si la même zone a été modifiée
# Les conflits sont marqués dans les fichiers avec <<<<<<< / >>>>>>>
# Résoudre les conflits, puis :
git add <fichier-résolu>
git stash drop    # supprimer le stash manuellement (pop a déjà échoué)
```

### Variante : conserver le stash sans le supprimer

```bash
git stash apply   # applique le stash mais le garde dans la pile
git stash list    # lister tous les stashs en attente
git stash drop    # supprimer le dernier stash explicitement
```

### Règle d'équipe sur les branches partagées

Travailler à deux sur la même branche est une exception, pas la règle. Si deux personnes modifient les mêmes fichiers fréquemment, envisager :
- de **découper le travail** en sous-tâches sur des branches séparées
- de **synchroniser plus souvent** pour réduire l'écart entre les deux copies

---

## Scénario 2 — Une grande équipe (10+ devs)

**Situation :** L'équipe est nombreuse, chacun travaille sur sa propre branche, mais la branche principale `main` (ou `dev`) avance vite. Ta branche `feat/nouveau-modele` a été créée il y a 4 jours et `main` a reçu 30 commits depuis.

**Problème :** si tu te contentes d'ouvrir une PR maintenant, les conflits seront nombreux et la review sera plus difficile. Il faut **garder sa branche synchronisée**.

**Solution : Fetch → Rebase**

```bash
# 1. Récupérer l'état du dépôt distant SANS modifier la branche locale
git fetch origin

# 2. Rejouer ses commits par-dessus le dernier main
git rebase origin/main
```

### Pourquoi `fetch` et non `pull` ?

`git pull` = `git fetch` + `git merge`. Le merge crée un commit de fusion supplémentaire qui pollue l'historique.

```
# Avec pull (merge) — historique bruyant
A---B---C---D  origin/main
     \         \
      E---F---M  feat  (M = commit de merge)

# Avec fetch + rebase — historique linéaire
A---B---C---D  origin/main
              \
               E'---F'  feat (commits rejoués)
```

### Workflow complet pour une grande équipe

```bash
# Au début de la journée : synchroniser
git fetch origin
git rebase origin/main

# En cours de journée : committer souvent, en petits incréments
git add src/modele.py
git commit -m "feat(model): ajouter la couche de normalisation"

# Avant d'ouvrir la PR : synchroniser une dernière fois
git fetch origin
git rebase origin/main

# Pousser (et forcer si nécessaire car rebase réécrit l'historique)
git push origin feat/nouveau-modele
# Si la branche a déjà été poussée et rebasée :
git push --force-with-lease origin feat/nouveau-modele
```

### `--force-with-lease` vs `--force`

| Option | Comportement |
|--------|-------------|
| `--force` | Écrase le remote sans vérification — dangereux en équipe |
| `--force-with-lease` | Vérifie d'abord que personne n'a poussé entre-temps — **toujours préférer cette option** |

### Gérer les conflits pendant un rebase

```bash
# Le rebase s'arrête sur le premier commit en conflit
# CONFLICT (content): Merge conflict in src/modele.py

# 1. Résoudre le conflit dans l'éditeur
# 2. Stager le fichier résolu
git add src/modele.py

# 3. Continuer le rebase
git rebase --continue

# En cas de doute : annuler et revenir à l'état d'avant
git rebase --abort
```

### Règle d'or : ne jamais rebaser une branche partagée

```bash
# Sur une branche sur laquelle d'autres ont travaillé :
# → utiliser git merge, pas git rebase

# Sur sa propre branche de feature (non partagée) :
# → utiliser git rebase pour garder un historique propre
```

---

## Pourquoi `fetch + rebase` plutôt que `pull`

| Critère | `git pull` (= fetch + merge) | `git fetch` + `git rebase` |
|---------|-------------------------------|----------------------------|
| Commits de merge | Crée un commit de merge | Aucun commit de merge |
| Historique | Non-linéaire, bruyant | Linéaire, lisible |
| Conflits | Résolus une seule fois | Résolus commit par commit |
| Adapté pour | Branches d'intégration (`main`, `dev`) | Branches de feature individuelles |
| Recommandé quand | On intègre une PR | On met à jour sa branche locale |

**Règle :**
- `git pull` pour `main` et `dev` (branches d'intégration)
- `git fetch` + `git rebase` pour ses propres branches de feature

---

## Politiques de protection des branches

Les politiques de branche empêchent les accidents (commit direct sur `main`, merge sans review, CI rouge). Elles se configurent côté serveur, pas dans Git lui-même.

### Règles essentielles à activer sur `main` (et `dev`)

| Règle | Objectif |
|-------|---------|
| Interdire les push directs | Tout code passe par une PR/MR |
| Exiger au moins 1 reviewer | Pas de merge en solo |
| Exiger que la CI soit verte | Aucun code cassé en production |
| Interdire le force push | L'historique de main est immuable |
| Exiger des commits signés | Optionnel — garantit l'identité des auteurs |

### GitHub — Branch Protection Rules

`Settings → Branches → Add branch protection rule`

```
Branch name pattern: main

☑ Require a pull request before merging
  ☑ Require approvals: 1
  ☑ Dismiss stale pull request approvals when new commits are pushed

☑ Require status checks to pass before merging
  ☑ Require branches to be up to date before merging
  → Ajouter le job CI (ex: "ci / lint and test")

☑ Require conversation resolution before merging

☑ Do not allow bypassing the above settings
```

### GitLab — Protected Branches

`Settings → Repository → Protected branches`

```
Branch: main
Allowed to merge:  Maintainers
Allowed to push:   No one
Allowed to force push: ☐ (désactivé)

→ Ajouter une Approval Rule dans Settings → Merge Requests :
  Approvals required: 1
  Reset approvals on push: ☑
```

Pour lier la CI :
`Settings → Merge Requests → Pipelines → ☑ Pipelines must succeed`

### Azure DevOps — Branch Policies

`Project Settings → Repositories → [repo] → Policies → [branch] main`

```
☑ Require a minimum number of reviewers
  Minimum number of reviewers: 1
  ☑ Reset all approval votes when new changes are pushed

☑ Check for linked work items

☑ Check for comment resolution

☑ Limit merge types
  → Sélectionner : Squash merge (ou Rebase and fast-forward)
  → Désélectionner : Basic merge (no fast-forward)

Build Validation:
  ☑ Add build policy → sélectionner le pipeline CI
  Trigger: Automatic
  Policy requirement: Required
```

### Stratégies de merge autorisées

| Stratégie | Historique | Quand l'utiliser |
|-----------|-----------|-----------------|
| **Merge commit** | Conserve tous les commits + 1 commit de merge | Branches longues, garder la traçabilité |
| **Squash merge** | Écrase tous les commits en 1 seul | Branches courtes avec des commits WIP |
| **Rebase + fast-forward** | Historique linéaire, pas de commit de merge | Équipes qui veulent un `git log` propre |

**Recommandation** : autoriser uniquement **Squash** et **Rebase + fast-forward** sur `main`. Cela garantit un historique lisible et évite les commits de merge inutiles.

---

## Résumé des règles d'équipe

| Situation | Procédure |
|-----------|-----------|
| Récupérer les modifs d'un collègue sur la même branche | `git stash` → `git pull` → `git stash pop` |
| Mettre à jour sa branche de feature | `git fetch origin` → `git rebase origin/main` |
| Pousser après un rebase | `git push --force-with-lease` |
| Merger une feature dans main | Via PR/MR uniquement — jamais en direct |
| Un collègue a pushé sur une branche partagée | `git fetch` → `git merge origin/<branche>` (pas de rebase) |
| La CI est rouge sur ma PR | Corriger localement, commiter, pousser — la CI se relance |
| Ma PR a des conflits avec main | `git fetch origin` → `git rebase origin/main` → résoudre → `git push --force-with-lease` |
