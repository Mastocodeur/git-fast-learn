# Git au quotidien

Ce chapitre couvre les pratiques Git du quotidien : conventions de commits, workflow de branches, rebase vs merge, et gestion des conflits.

---

## Sommaire

- [Conventions de commits](#conventions-de-commits)
- [Workflow de branches](#workflow-de-branches)
- [Rebase vs Merge](#rebase-vs-merge)
- [Gestion des conflits](#gestion-des-conflits)
- [Commandes utiles](#commandes-utiles)

---

## Conventions de commits

Un bon message de commit répond à la question : **"Que fait ce commit ?"** — pas "Qu'est-ce que j'ai fait", mais ce que le code fait après ce commit.

On suit la convention [Conventional Commits](https://www.conventionalcommits.org/), qui structure chaque message ainsi :

```
<type>(<scope>): <description courte>

[corps optionnel]

[footer optionnel]
```

### Types courants

| Type | Quand l'utiliser |
|---|---|
| `feat` | Ajout d'une nouvelle fonctionnalité |
| `fix` | Correction d'un bug |
| `docs` | Modification de la documentation uniquement |
| `style` | Formatage, espaces, virgules — sans changement logique |
| `refactor` | Restructuration du code sans ajout de fonctionnalité ni correction de bug |
| `test` | Ajout ou correction de tests |
| `chore` | Tâches de maintenance (dépendances, config CI…) |
| `perf` | Amélioration des performances |

### Exemples

```
feat(auth): ajouter l'authentification OAuth2

fix(pipeline): corriger le calcul de la moyenne sur les valeurs nulles

docs(readme): mettre à jour les instructions d'installation

chore(deps): mettre à jour pandas vers 2.2.0

refactor(preprocessing): extraire la normalisation dans une fonction dédiée
```

### Règles de base

- La description est en **minuscules**, sans majuscule initiale ni point final
- Elle fait **moins de 72 caractères**
- Elle est rédigée à l'**impératif** : "ajouter" et non "ajouté" ou "ajout de"
- Le scope (entre parenthèses) est optionnel mais utile sur les gros projets

---

## Workflow de branches

### Nommage des branches

```
<type>/<description-en-kebab-case>

# Exemples :
feat/export-csv
fix/null-values-preprocessing
docs/update-installation-guide
chore/upgrade-python-312
```

### Le workflow standard

```bash
# 1. Toujours partir d'une branche principale à jour
git checkout main
git pull origin main

# 2. Créer sa branche de travail
git checkout -b feat/ma-nouvelle-fonctionnalite

# 3. Travailler, commiter régulièrement
git add src/mon_module.py
git commit -m "feat(module): ajouter la fonction de normalisation"

# 4. Pousser sa branche
git push origin feat/ma-nouvelle-fonctionnalite

# 5. Ouvrir une Pull Request / Merge Request sur GitHub ou GitLab
```

### Règles importantes

- **Ne jamais commiter directement sur `main`** — toujours passer par une branche
- **Une branche = une tâche** — éviter les branches fourre-tout qui traînent des semaines
- **Supprimer les branches mergées** — elles encombrent le dépôt et créent de la confusion

```bash
# Supprimer une branche locale après merge
git branch -d feat/ma-nouvelle-fonctionnalite

# Supprimer la branche distante
git push origin --delete feat/ma-nouvelle-fonctionnalite
```

---

## Rebase vs Merge

Les deux permettent d'intégrer des changements d'une branche dans une autre, mais de manière différente.

### Merge

```bash
git checkout feat/ma-branche
git merge main
```

Le merge crée un **commit de fusion** qui unit les deux historiques. L'historique reflète fidèlement ce qui s'est passé, avec toutes les branches visibles.

```
A---B---C  main
     \   \
      D---E---M  feat (M = commit de merge)
```

**Avantage** : non-destructif, l'historique est exact.
**Inconvénient** : l'historique peut devenir difficile à lire sur un gros projet avec beaucoup de branches.

### Rebase

```bash
git checkout feat/ma-branche
git rebase main
```

Le rebase **réécrit** les commits de la branche comme s'ils avaient été créés à partir du dernier état de `main`. L'historique est linéaire.

```
A---B---C  main
             \
              D'---E'  feat (D et E réécrits)
```

**Avantage** : historique propre et linéaire, plus facile à lire.
**Inconvénient** : réécrit l'historique — ne jamais rebaser une branche partagée avec d'autres.

### Quelle règle appliquer ?

- **Rebase** pour mettre à jour sa branche de travail locale avec `main`
- **Merge** pour intégrer une branche dans `main` (via Pull Request)
- **Jamais de rebase sur une branche partagée** (poussée et utilisée par d'autres)

---

## Gestion des conflits

Un conflit survient quand deux branches ont modifié la même zone d'un fichier. Git ne sait pas quelle version garder et vous demande de trancher.

### Identifier les conflits

```bash
# Après un merge ou rebase qui échoue :
git status
# Les fichiers en conflit apparaissent sous "both modified"
```

### Lire un conflit

Dans le fichier en conflit, Git insère des marqueurs :

```python
<<<<<<< HEAD
# Version de votre branche actuelle
result = compute_mean(values)
=======
# Version de l'autre branche
result = compute_weighted_mean(values, weights)
>>>>>>> feat/weighted-metrics
```

### Résoudre un conflit

1. Ouvrir le fichier dans VS Code — l'éditeur affiche les deux versions avec des boutons "Accept Current", "Accept Incoming", "Accept Both"
2. Choisir la bonne version (ou combiner les deux manuellement)
3. Supprimer tous les marqueurs `<<<<<<<`, `=======`, `>>>>>>>`
4. Sauvegarder le fichier

```bash
# Marquer le conflit comme résolu
git add src/mon_fichier.py

# Continuer le merge ou le rebase
git merge --continue
# ou
git rebase --continue

# En cas de doute, annuler et repartir de zéro
git merge --abort
git rebase --abort
```

### Prévenir les conflits

- Faire des **commits petits et fréquents** — les gros commits sur des fichiers centraux sont la première cause de conflits
- **Synchroniser régulièrement** sa branche avec `main` (`git rebase main` ou `git merge main`)
- **Éviter de modifier les mêmes fichiers** que ses collègues sur des branches parallèles longues

---

## Commandes utiles

```bash
# Voir l'historique de manière lisible
git log --oneline --graph --all

# Annuler le dernier commit (en gardant les modifications)
git reset --soft HEAD~1

# Mettre de côté des modifications en cours sans commiter
git stash
git stash pop   # pour les récupérer

# Voir ce qui a changé avant de commiter
git diff
git diff --staged  # uniquement ce qui est en staging

# Modifier le dernier message de commit (avant push)
git commit --amend -m "fix(module): correction du message"

# Récupérer un fichier spécifique depuis une autre branche
git checkout nom-branche -- src/mon_fichier.py
```
