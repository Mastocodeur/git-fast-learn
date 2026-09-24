# Annuler et Corriger

Ce chapitre couvre toutes les façons d'annuler, corriger et récupérer des états dans Git. C'est le sujet le plus souvent mal maîtrisé — et le plus souvent demandé en entretien.

---

## Sommaire

- [Les 4 outils principaux](#les-4-outils-principaux)
- [git restore — annuler dans le working directory](#git-restore--annuler-dans-le-working-directory)
- [git reset — réécrire l'historique local](#git-reset--réécrire-lhistorique-local)
- [git revert — annuler un commit publié](#git-revert--annuler-un-commit-publié)
- [git commit --amend — corriger le dernier commit](#git-commit---amend--corriger-le-dernier-commit)
- [git reflog — le filet de sécurité ultime](#git-reflog--le-filet-de-sécurité-ultime)
- [Tableau de décision](#tableau-de-décision)

---

## Les 4 outils principaux

| Outil | Sur quoi il agit | Réécrit l'historique ? | Sûr après push ? |
|-------|-----------------|------------------------|-----------------|
| `git restore` | Working directory / Staging | Non | Oui |
| `git reset` | Commits locaux | Oui | **Non** |
| `git revert` | Crée un nouveau commit d'annulation | Non | **Oui** |
| `git commit --amend` | Dernier commit non pushé | Oui | **Non** |

**Règle fondamentale :** `git reset` et `--amend` ne s'utilisent **jamais** sur des commits déjà poussés (ni a fortiori sur `main`). Utiliser `git revert` dans ce cas.

---

## git restore — annuler dans le working directory

`git restore` annule des modifications **avant** qu'elles soient commitées.

### Cas 1 — Annuler les modifications d'un fichier (non stagé)

```bash
# Le fichier est modifié mais pas encore stagé (git add)
git restore src/mon_fichier.py
# → remet le fichier à l'état du dernier commit
# ⚠ Les modifications sont PERDUES définitivement
```

### Cas 2 — Désindexer un fichier stagé (enlever du staging sans perdre les modifs)

```bash
# Le fichier a été stagé avec git add mais on veut l'enlever du staging
git restore --staged src/mon_fichier.py
# → le fichier reste modifié dans le working directory, mais n'est plus stagé
```

### Cas 3 — Restaurer un fichier tel qu'il était dans un commit précédent

```bash
git restore --source HEAD~2 src/mon_fichier.py
# → restaure le fichier dans l'état du commit 2 commits en arrière
# Le fichier apparaît comme "modifié" dans le working directory
```

### Résumé restore

```
git restore <fichier>             # annule les modifs non-stagées
git restore --staged <fichier>    # désindexe (unstage) un fichier
git restore --source <sha> <f>    # restaure depuis un commit précis
```

---

## git reset — réécrire l'historique local

`git reset` déplace le pointeur `HEAD` (et la branche) vers un commit antérieur. Il existe 3 modes.

### Les 3 modes

```
             Working Dir   Staging Area   Commits
--soft       intact        intact         annulés  → les modifs restent stagées
--mixed      intact        annulé         annulés  → les modifs restent dans le working dir (défaut)
--hard       annulé        annulé         annulés  → TOUT est perdu
```

### `--soft` — annuler le commit, garder les modifs stagées

```bash
git reset --soft HEAD~1
# → le commit disparaît, les fichiers modifiés sont encore stagés (prêts à recommitter)
# Cas d'usage : corriger le message de commit ou splitter un commit en deux
```

### `--mixed` — annuler le commit et le staging (défaut)

```bash
git reset HEAD~1
# ou explicitement :
git reset --mixed HEAD~1
# → le commit disparaît, les fichiers sont dans le working directory mais plus stagés
# Cas d'usage : recommencer proprement depuis ces fichiers modifiés
```

### `--hard` — tout effacer

```bash
git reset --hard HEAD~1
# → le commit disparaît ET les modifications sont perdues
# ⚠ Irréversible sans reflog — utiliser avec précaution

git reset --hard HEAD
# → annule TOUTES les modifications non-commitées (reset à l'état du dernier commit)
```

### Référencer les commits

```bash
HEAD      # le commit actuel
HEAD~1    # 1 commit en arrière
HEAD~3    # 3 commits en arrière
abc1234   # un SHA précis (obtenu avec git log)
```

---

## git revert — annuler un commit publié

`git revert` crée un **nouveau commit** qui annule les effets d'un commit précédent. L'historique est préservé — c'est la bonne approche pour les commits déjà pushés.

```bash
# Annuler le dernier commit
git revert HEAD

# Annuler un commit précis
git revert abc1234

# Annuler plusieurs commits (de HEAD~3 jusqu'à HEAD inclus)
git revert HEAD~3..HEAD

# Préparer le revert sans commiter immédiatement (pour inspecter)
git revert --no-commit abc1234
git status  # vérifier ce qui va être annulé
git commit -m "revert: annuler la migration défectueuse"
```

### revert vs reset

```
# reset : retire le commit de l'historique
A---B---C---D  →  A---B (C et D disparaissent)

# revert : ajoute un commit qui annule
A---B---C---D---D'  (D' annule les effets de D)
```

**Quand utiliser quoi :**
- `reset` : sur sa branche locale, avant de pusher
- `revert` : sur n'importe quelle branche déjà partagée

---

## git commit --amend — corriger le dernier commit

`--amend` réécrit le dernier commit : on peut corriger son message, ajouter des fichiers oubliés, ou les deux.

```bash
# Corriger uniquement le message
git commit --amend -m "feat(auth): ajouter la validation du token"

# Ajouter un fichier oublié sans changer le message
git add src/fichier_oublie.py
git commit --amend --no-edit

# Modifier à la fois le message et ajouter un fichier
git add src/fichier_oublie.py
git commit --amend -m "feat(auth): ajouter la validation du token et le middleware"
```

**Règle :** `--amend` réécrit l'historique. Ne l'utiliser que sur le dernier commit, et **uniquement si ce commit n'a pas encore été pushé**.

---

## git reflog — le filet de sécurité ultime

Le reflog est le journal de tous les mouvements de HEAD dans le dépôt local. Il permet de retrouver des commits qu'on croyait perdus (après un `reset --hard`, un `checkout` malencontreux, etc.).

```bash
git reflog
# Sortie :
# abc1234 HEAD@{0}: reset: moving to HEAD~1
# def5678 HEAD@{1}: commit: feat(auth): ajouter le token
# ...
```

### Récupérer un commit perdu après reset --hard

```bash
# On a fait git reset --hard par erreur et on veut récupérer le commit perdu

# 1. Trouver le SHA du commit perdu dans le reflog
git reflog
# → def5678 HEAD@{1}: commit: feat(auth): ajouter le token

# 2. Option A : remettre la branche à ce commit
git reset --hard def5678

# 2. Option B : créer une nouvelle branche à partir de ce commit
git checkout -b recovery/feat-auth def5678

# 2. Option C : cherry-pick du commit dans la branche actuelle
git cherry-pick def5678
```

### Récupérer une branche supprimée par erreur

```bash
# On a supprimé une branche avec git branch -D par erreur

git reflog
# → abc1234 HEAD@{3}: commit: le dernier commit de la branche supprimée

git checkout -b feat/ma-branche-recuperee abc1234
```

### Durée de vie du reflog

Le reflog est local — il n'est pas poussé vers le remote. Par défaut, les entrées sont conservées **90 jours**. C'est une information qui ne sort jamais du poste de travail.

---

## Tableau de décision

```
Qu'est-ce que je veux annuler ?
│
├─ Une modification non commitée dans un fichier
│   └─ git restore <fichier>
│
├─ Un fichier stagé par erreur (git add trop vite)
│   └─ git restore --staged <fichier>
│
├─ Le message du dernier commit (non pushé)
│   └─ git commit --amend -m "..."
│
├─ Un fichier oublié dans le dernier commit (non pushé)
│   └─ git add <fichier> && git commit --amend --no-edit
│
├─ Les N derniers commits — en gardant les modifications
│   └─ git reset --soft HEAD~N
│
├─ Les N derniers commits — repartir des fichiers non stagés
│   └─ git reset --mixed HEAD~N (ou git reset HEAD~N)
│
├─ Les N derniers commits — TOUT effacer (local seulement)
│   └─ git reset --hard HEAD~N   ⚠ irréversible sans reflog
│
├─ Un commit déjà pushé (sur main ou branche partagée)
│   └─ git revert <sha>   → crée un commit d'annulation
│
└─ Un commit que je croyais perdu
    └─ git reflog → retrouver le SHA → git checkout ou git cherry-pick
```

---

## Cheatsheet

```bash
# Annuler les modifs d'un fichier non stagé
git restore src/fichier.py

# Désindexer un fichier stagé
git restore --staged src/fichier.py

# Revenir à l'état du dernier commit (tout perdre)
git reset --hard HEAD

# Défaire le dernier commit (garder les modifs stagées)
git reset --soft HEAD~1

# Corriger le message du dernier commit
git commit --amend -m "nouveau message"

# Annuler un commit pushé sans réécrire l'historique
git revert <sha>

# Retrouver un état perdu
git reflog
git checkout -b recovery <sha>
```
