# Git Avancé

Ce chapitre couvre les commandes Git avancées que tout développeur senior doit connaître : réécriture d'historique, récupération ciblée de commits, débogage, et travail sur plusieurs branches simultanément.

---

## Sommaire

- [git stash — en profondeur](#git-stash--en-profondeur)
- [git cherry-pick — intégrer un commit précis](#git-cherry-pick--intégrer-un-commit-précis)
- [git rebase interactif — réécrire l'historique](#git-rebase-interactif--réécrire-lhistorique)
- [git bisect — trouver le commit fautif](#git-bisect--trouver-le-commit-fautif)
- [git log — lire l'historique comme un pro](#git-log--lire-lhistorique-comme-un-pro)
- [git blame — trouver qui a écrit quoi](#git-blame--trouver-qui-a-écrit-quoi)
- [git worktree — plusieurs branches simultanément](#git-worktree--plusieurs-branches-simultanément)
- [git submodule — dépôts imbriqués](#git-submodule--dépôts-imbriqués)
- [Les refs — internals et plumbing](#les-refs--internals-et-plumbing)

---

## git stash — en profondeur

`git stash` met de côté les modifications en cours dans une pile temporaire, sans commiter.

### Commandes essentielles

```bash
# Mettre de côté toutes les modifications (tracked + staged)
git stash

# Avec un nom explicite (recommandé en équipe)
git stash push -m "WIP: refactoring du loader"

# Inclure les fichiers non-trackés (untracked)
git stash push -u

# Inclure AUSSI les fichiers ignorés (.gitignore)
git stash push -a

# Stasher uniquement certains fichiers
git stash push -m "partial stash" -- src/loader.py src/config.py
```

### Inspecter et restaurer

```bash
# Lister tous les stashs
git stash list
# → stash@{0}: WIP on feat/export: abc1234 feat: ajouter l'export
# → stash@{1}: WIP on feat/loader: def5678 add: loader initial

# Voir ce qu'un stash contient
git stash show stash@{1}
git stash show -p stash@{1}   # diff complet

# Réappliquer le dernier stash et le supprimer de la pile
git stash pop

# Réappliquer un stash précis sans le supprimer
git stash apply stash@{1}

# Supprimer un stash précis
git stash drop stash@{1}

# Vider toute la pile
git stash clear
```

### Créer une branche depuis un stash

Utile quand le travail mis de côté est devenu trop important pour rester dans un stash.

```bash
git stash branch feat/nouveau-loader stash@{0}
# → crée une nouvelle branche, l'applique et supprime le stash
```

---

## git cherry-pick — intégrer un commit précis

`git cherry-pick` copie un ou plusieurs commits d'une branche vers la branche courante, sans merger toute la branche.

### Cas d'usage typique

Un fix critique a été commité sur `feat/payment` mais doit être appliqué sur `main` immédiatement, sans attendre la fin de la feature.

```bash
# 1. Trouver le SHA du commit à récupérer
git log feat/payment --oneline
# → abc1234 fix(payment): corriger le calcul de TVA

# 2. Appliquer ce commit sur main
git checkout main
git cherry-pick abc1234
```

### Plusieurs commits

```bash
# Appliquer plusieurs commits (dans l'ordre)
git cherry-pick abc1234 def5678 ghi9012

# Appliquer une plage de commits (exclut le premier SHA)
git cherry-pick abc1234..ghi9012
```

### Options utiles

```bash
# Appliquer sans commiter (pour inspecter avant)
git cherry-pick --no-commit abc1234

# Garder la date d'origine du commit
git cherry-pick -x abc1234   # ajoute aussi "(cherry picked from ...)" dans le message

# En cas de conflit pendant le cherry-pick
git add <fichier-résolu>
git cherry-pick --continue
# ou annuler
git cherry-pick --abort
```

---

## git rebase interactif — réécrire l'historique

Le rebase interactif (`rebase -i`) permet de réorganiser, fusionner, modifier ou supprimer des commits avant de les pousser. C'est un outil puissant pour nettoyer une branche avant d'ouvrir une PR.

```bash
# Réécrire les 4 derniers commits
git rebase -i HEAD~4
```

Git ouvre un éditeur avec la liste des commits :

```
pick abc1234 feat(model): ajouter la couche d'entrée
pick def5678 wip
pick ghi9012 fix typo
pick jkl3456 feat(model): ajouter la couche de sortie

# Commandes disponibles :
# pick   = garder le commit tel quel
# reword = garder mais modifier le message
# edit   = modifier le contenu du commit
# squash = fusionner dans le commit précédent (garder les deux messages)
# fixup  = fusionner dans le commit précédent (supprimer ce message)
# drop   = supprimer le commit
```

### Exemples pratiques

**Fusionner les commits WIP en un seul :**

```
pick abc1234 feat(model): ajouter la couche d'entrée
fixup def5678 wip
fixup ghi9012 fix typo
pick jkl3456 feat(model): ajouter la couche de sortie
```

Résultat : 2 commits propres au lieu de 4.

**Corriger le message d'un commit :**

```
reword abc1234 feat(model): ajouter la couche d'entrée
pick def5678 wip
```

Git s'arrêtera sur ce commit et ouvrira l'éditeur pour modifier le message.

**Réordonner des commits :**

On peut simplement changer l'ordre des lignes. Git rejoue les commits dans le nouvel ordre.

### Règle

Comme tout rebase, le rebase interactif réécrit l'historique. **Ne jamais le faire sur des commits déjà pushés sur une branche partagée.**

---

## git bisect — trouver le commit fautif

`git bisect` utilise une recherche binaire pour trouver efficacement quel commit a introduit un bug. Très efficace sur de longues plages d'historique.

### Procédure manuelle

```bash
# 1. Démarrer la session bisect
git bisect start

# 2. Indiquer un commit où le bug est présent (souvent HEAD)
git bisect bad

# 3. Indiquer un commit où le bug n'existait pas
git bisect good v1.2.0
# → Git checkout automatiquement un commit au milieu

# 4. Tester, puis indiquer si ce commit est bon ou mauvais
git bisect good   # ou
git bisect bad

# → Répéter jusqu'à ce que Git trouve le commit fautif
# "abc1234 is the first bad commit"

# 5. Terminer la session et revenir au HEAD initial
git bisect reset
```

### Automatisation avec un script de test

```bash
git bisect start
git bisect bad HEAD
git bisect good v1.2.0

# Git lance le script sur chaque commit et décide seul
# Le script doit retourner 0 (success) ou 1 (failure)
git bisect run python tests/test_regression.py
# → Git trouve le commit fautif automatiquement

git bisect reset
```

---

## git log — lire l'historique comme un pro

```bash
# Vue graphique de toutes les branches
git log --oneline --graph --all

# Limiter aux N derniers commits
git log --oneline -10

# Filtrer par auteur
git log --author="Marie"

# Filtrer par date
git log --since="2 weeks ago"
git log --after="2024-01-01" --before="2024-06-01"

# Rechercher dans les messages de commit
git log --grep="feat(auth)"

# Voir les commits qui ont touché un fichier
git log -- src/auth.py

# Voir les commits qui ont touché une fonction
git log -L :nom_de_la_fonction:src/auth.py

# Chercher quand une ligne de code a été ajoutée ou supprimée
git log -S "def compute_score"

# Voir les stats des fichiers modifiés par commit
git log --stat

# Format personnalisé
git log --pretty=format:"%h %ad | %s [%an]" --date=short
```

---

## git blame — trouver qui a écrit quoi

`git blame` montre, ligne par ligne, quel commit et quel auteur ont introduit chaque ligne d'un fichier. Indispensable pour comprendre le contexte d'un code.

```bash
# Blame d'un fichier entier
git blame src/auth.py

# Blame d'une plage de lignes
git blame -L 42,55 src/auth.py

# Blame en ignorant les changements de formatage (reindent, etc.)
git blame -w src/auth.py

# Blame à partir d'un commit précis
git blame abc1234 -- src/auth.py
```

Sortie typique :
```
abc1234 (Marie  2024-03-15 14:22:01 +0100 42) def compute_score(values):
def5678 (Thomas 2024-03-10 09:15:33 +0100 43)     return sum(values) / len(values)
```

---

## git worktree — plusieurs branches simultanément

`git worktree` permet d'avoir plusieurs branches checkoutées **simultanément** dans des dossiers différents, sans avoir à switcher de branche.

**Cas d'usage :** un hotfix urgent arrive pendant qu'on est en plein développement d'une feature. Plutôt que de stasher et switcher, on crée un second working tree.

```bash
# Créer un second working tree sur la branche hotfix/payment
git worktree add ../mon-projet-hotfix hotfix/payment

# Travailler dans le dossier dédié
cd ../mon-projet-hotfix
# ... faire le fix, commiter, pusher ...

# Revenir au travail principal (l'autre dossier est inchangé)
cd ../mon-projet

# Lister les worktrees actifs
git worktree list

# Supprimer le worktree une fois terminé
git worktree remove ../mon-projet-hotfix
```

---

## git submodule — dépôts imbriqués

Les submodules permettent d'inclure un dépôt Git à l'intérieur d'un autre. Utile pour des dépendances internes ou des bibliothèques partagées entre projets.

```bash
# Ajouter un submodule
git submodule add git@github.com:org/shared-lib.git libs/shared-lib

# Cloner un repo avec ses submodules
git clone --recurse-submodules git@github.com:org/mon-projet.git

# Initialiser les submodules après un clone classique
git submodule update --init --recursive

# Mettre à jour tous les submodules vers leur dernier commit
git submodule update --remote

# Voir l'état des submodules
git submodule status
```

**Note :** les submodules ont une réputation d'être complexes à maintenir. Dans la plupart des cas modernes, un gestionnaire de paquets (pip, npm, cargo) ou un monorepo est préférable.

---

## Les refs — internals et plumbing

Une **ref** est un pointeur nommé vers un commit. Tout ce que Git appelle "branche", "tag" ou "HEAD" est en réalité une ref stockée dans `.git/refs/`.

```
.git/
├── HEAD                      ← ref symbolique → refs/heads/main
└── refs/
    ├── heads/main            ← branche locale (contient un SHA)
    ├── remotes/origin/main   ← branche remote
    └── tags/v1.0             ← tag léger
```

Ces commandes sont des outils **plumbing** (bas niveau). On ne les utilise pas dans un workflow quotidien — elles servent à écrire des scripts, des hooks avancés, ou à comprendre les internals de Git.

### git show-ref — lister toutes les refs locales

```bash
git show-ref
# a3f9c12 refs/heads/main
# b1d0e45 refs/heads/feature/login
# c2a1b33 refs/tags/v1.0

git show-ref --heads              # uniquement les branches
git show-ref --tags               # uniquement les tags
git show-ref main                 # filtrer par nom
git show-ref --verify refs/heads/main   # vérifie qu'une ref existe (exit 1 si non)
git show-ref -d                   # dereference les tags annotés (montre le commit pointé)
```

### git for-each-ref — lister et formater les refs (scripting)

La version programmable de `show-ref`. Indispensable pour écrire des scripts sur les branches.

```bash
# Format personnalisé
git for-each-ref --format='%(refname:short) %(objectname:short)' refs/heads/
# main       a3f9c12
# feature/login b1d0e45

# Trier les branches par date de dernier commit
git for-each-ref --sort=-committerdate --format='%(refname:short)' refs/heads/

# Voir auteur et message du dernier commit par branche
git for-each-ref --format='%(refname:short) | %(authorname) | %(subject)' refs/heads/
```

Champs utiles : `%(refname)`, `%(refname:short)`, `%(objectname:short)`, `%(authorname)`, `%(committerdate:relative)`, `%(subject)`, `%(upstream:short)`.

### git symbolic-ref — lire et écrire HEAD

`HEAD` n'est pas un SHA mais une ref symbolique qui pointe vers la branche courante.

```bash
git symbolic-ref HEAD
# refs/heads/main

git symbolic-ref --short HEAD     # → main (format court, équivalent à git branch --show-current)
```

En pratique, c'est ce que les hooks et scripts utilisent pour connaître la branche active.

### git update-ref — créer ou déplacer une ref de façon sûre

Permet de manipuler des refs sans passer par les commandes porcelain. Git l'utilise en interne à chaque `git commit`.

```bash
# Créer une branche de backup pointant sur HEAD
git update-ref refs/heads/backup-before-rebase HEAD

# Supprimer une ref
git update-ref -d refs/heads/vieille-branche

# Déplacer une ref avec vérification : n'opère que si la valeur actuelle est <old-sha>
git update-ref refs/heads/main <new-sha> <old-sha>
```

### git check-ref-format — valider un nom de ref

Utile dans un hook pre-receive ou pre-push pour rejeter des noms de branches invalides.

```bash
git check-ref-format "refs/heads/feat/login"   # exit 0 = valide
git check-ref-format "refs/heads/ma branche"   # exit 1 = invalide (espace)
git check-ref-format --branch "feat/login"     # valide un nom de branche directement
```

### git ls-remote — lister les refs d'un remote (sans fetch)

Inspecte un dépôt distant sans rien télécharger localement.

```bash
git ls-remote origin
git ls-remote --heads origin      # uniquement les branches distantes
git ls-remote --tags origin       # uniquement les tags distants
```

Cas d'usage : vérifier qu'une branche ou un tag existe sur le remote avant de lancer un pipeline.

### git pack-refs — compacter les refs

Git stocke chaque ref dans un fichier séparé. Sur un repo avec des centaines de branches, cela ralentit les opérations. `pack-refs` les regroupe dans `.git/packed-refs`.

```bash
git pack-refs --all    # compacte toutes les refs (heads + tags)
```

Git le fait automatiquement via `git gc`. À appeler manuellement sur un très vieux repo qui n'a jamais été nettoyé.

### Cas d'usage réels

| Besoin | Commande |
|---|---|
| Lister les 5 branches les plus récentes | `git for-each-ref --sort=-committerdate --format='%(refname:short)' refs/heads/ \| head -5` |
| Savoir sur quelle branche on est dans un script | `git symbolic-ref --short HEAD` |
| Vérifier qu'une branche distante existe | `git ls-remote --heads origin feat/ma-branche` |
| Créer un backup de branche avant un rebase risqué | `git update-ref refs/heads/backup HEAD` |
| Valider un nom de branche dans un hook | `git check-ref-format --branch "$BRANCH_NAME"` |
