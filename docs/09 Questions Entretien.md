# Questions d'Entretien Git

> Les 25 questions Git les plus fréquentes en entretien technique, avec des réponses précises et concises.

---

## Sommaire

- [Fondamentaux](#fondamentaux)
- [Branches et historique](#branches-et-historique)
- [Collaboration et workflows](#collaboration-et-workflows)
- [Annuler et corriger](#annuler-et-corriger)
- [Commandes avancées](#commandes-avancées)

---

## Fondamentaux

### 1. Quelle est la différence entre Git et GitHub ?

**Git** est un système de contrôle de version distribué — un outil en ligne de commande qui tourne localement. **GitHub** (ou GitLab, Azure DevOps) est une plateforme web qui héberge des dépôts Git distants et ajoute des fonctionnalités collaboratives : pull requests, code review, CI/CD, gestion des issues.

Git peut exister sans GitHub. GitHub ne peut pas exister sans Git.

---

### 2. Quelles sont les 3 zones de Git ? Comment un fichier passe de l'une à l'autre ?

```
Working Directory  →(git add)→  Staging Area  →(git commit)→  Repository
                                                              →(git push)→  Remote
```

- **Working Directory** : les fichiers tels qu'on les voit sur le disque
- **Staging Area (index)** : l'instantané de ce qui sera inclus dans le prochain commit
- **Repository** : l'historique des commits stocké dans `.git/`

---

### 3. Qu'est-ce qu'un commit dans Git ?

Un commit est un **instantané complet** de l'état du projet à un moment donné, pas un diff. Chaque commit contient :
- un SHA-1 unique (identifiant de 40 caractères)
- un pointeur vers l'arbre de fichiers (tree)
- un pointeur vers le(s) commit(s) parent(s)
- les métadonnées : auteur, date, message

Git calcule le SHA-1 à partir du contenu — deux commits identiques produisent toujours le même SHA.

---

### 4. Qu'est-ce que HEAD ?

`HEAD` est un pointeur qui indique **où on se trouve** dans l'historique. En général, il pointe vers la branche courante, et la branche pointe vers le dernier commit.

```
HEAD → main → commit abc1234
```

En état "detached HEAD" (après `git checkout <sha>`), HEAD pointe directement vers un commit, pas vers une branche.

---

### 5. Quelle est la différence entre `git fetch` et `git pull` ?

- `git fetch` : télécharge les commits distants dans le dépôt local **sans modifier** la branche de travail. Le remote tracking branch (`origin/main`) est mis à jour, mais `main` local ne change pas.
- `git pull` : combine `git fetch` + `git merge` (ou `git rebase` avec `--rebase`). Il met à jour la branche locale immédiatement.

**Bonne pratique :** préférer `fetch` pour voir ce qui a changé avant d'intégrer, surtout sur les branches partagées.

---

## Branches et historique

### 6. Quelle est la différence entre `git merge` et `git rebase` ?

Les deux intègrent les modifications d'une branche dans une autre, mais de façon différente :

**Merge** : crée un commit de fusion (merge commit) qui unit les deux historiques. L'historique reflète ce qui s'est vraiment passé.

```
A---B---C  main
     \   \
      D---M  feat  (M = merge commit)
```

**Rebase** : rejoue les commits de la branche au-dessus du dernier commit de la cible. L'historique est linéaire, comme si le développement avait été séquentiel.

```
A---B---C  main
             \
              D'  feat (D rejoué)
```

**Règle :** rebase pour mettre à jour sa branche locale, merge pour intégrer dans `main` via PR.

---

### 7. Quand NE PAS utiliser git rebase ?

Ne jamais rebaser une branche **déjà partagée** avec d'autres développeurs. Le rebase réécrit les SHAs des commits — les collègues qui ont basé leur travail sur ces commits se retrouveront avec un historique incohérent et des conflits difficiles à résoudre.

Règle : "Ne jamais rebaser ce qui est public."

---

### 8. Qu'est-ce qu'un fast-forward merge ?

Quand la branche cible (ex: `main`) n'a reçu aucun commit depuis la création de la branche de feature, Git peut simplement **avancer** le pointeur de `main` jusqu'au dernier commit de la feature, sans créer de commit de merge.

```
Avant : A---B  main
             \
              C---D  feat

Après (fast-forward) : A---B---C---D  main
```

On peut forcer Git à créer un merge commit même en fast-forward avec `git merge --no-ff`.

---

### 9. Comment voir l'historique des branches sous forme de graphe ?

```bash
git log --oneline --graph --all
```

---

### 10. Quelle est la différence entre une branche légère (lightweight) et un tag annoté ?

| | Tag léger (lightweight) | Tag annoté (annotated) |
|--|------------------------|----------------------|
| Stockage | Simple pointeur vers un commit | Objet Git complet |
| Métadonnées | Aucune | Auteur, date, message |
| Signature GPG | Non | Oui (avec `-s`) |
| Usage recommandé | Marqueur temporaire | Versions officielles |

```bash
git tag v1.0.0                          # léger
git tag -a v1.0.0 -m "Version 1.0.0"  # annoté (recommandé)
```

---

## Collaboration et workflows

### 11. Quelle est la différence entre `git clone` et `git fork` ?

- `git clone` : crée une copie locale d'un dépôt existant, avec un lien (`origin`) vers le dépôt source.
- `fork` : crée une copie **côté serveur** (GitHub/GitLab) d'un dépôt dans son propre espace. On clone ensuite son fork. Utilisé pour contribuer à des projets open source sans accès en écriture au dépôt original.

---

### 12. Quelle est la différence entre une Pull Request et une Merge Request ?

C'est la même chose : une demande de révision et d'intégration de code d'une branche vers une autre.
- GitHub et Azure DevOps utilisent le terme **Pull Request (PR)**
- GitLab utilise le terme **Merge Request (MR)**

---

### 13. Comment synchroniser sa branche avec main en équipe ?

```bash
git fetch origin
git rebase origin/main
```

On préfère `fetch + rebase` à `pull` pour éviter les commits de merge inutiles et garder un historique linéaire.

Si la branche a déjà été poussée, un force push est nécessaire après le rebase :
```bash
git push --force-with-lease origin feat/ma-branche
```

---

### 14. Que faire quand deux développeurs travaillent sur la même branche ?

```bash
git stash                          # mettre de côté les modifs locales
git pull origin <branche>          # récupérer les modifs du collègue
git stash pop                      # réappliquer ses modifs
# résoudre les conflits si nécessaire
```

---

### 15. Qu'est-ce qu'un conflit Git ? Comment le résoudre ?

Un conflit survient quand deux branches ont modifié la même zone d'un fichier. Git insère des marqueurs dans le fichier :

```
<<<<<<< HEAD
# version locale
=======
# version distante
>>>>>>> origin/main
```

Pour résoudre :
1. Ouvrir le fichier et choisir (ou combiner) les deux versions
2. Supprimer tous les marqueurs
3. `git add <fichier>` puis `git merge --continue` (ou `git rebase --continue`)

---

## Annuler et corriger

### 16. Quelle est la différence entre `git reset` et `git revert` ?

| | `git reset` | `git revert` |
|--|-------------|-------------|
| Mécanisme | Déplace HEAD en arrière, supprime les commits | Crée un nouveau commit qui annule |
| Réécrit l'historique | Oui | Non |
| Sûr après push | **Non** | **Oui** |
| Usage | Commits locaux non pushés | Commits déjà partagés |

---

### 17. Comment annuler un `git add` avant de commiter ?

```bash
git restore --staged <fichier>
# ou l'ancienne syntaxe :
git reset HEAD <fichier>
```

---

### 18. Comment modifier le message du dernier commit ?

```bash
git commit --amend -m "nouveau message"
```

Uniquement si le commit n'a **pas encore été pushé**.

---

### 19. Comment récupérer un commit perdu après un `git reset --hard` ?

```bash
git reflog
# retrouver le SHA du commit perdu
git checkout -b recovery <sha>
```

Le reflog conserve tous les mouvements de HEAD pendant 90 jours.

---

### 20. Quelle est la différence entre `git reset --soft`, `--mixed` et `--hard` ?

```
              Working Dir   Staging   Commits
--soft        intact        intact    annulés → modifs restent stagées
--mixed       intact        effacé    annulés → modifs dans le working dir (défaut)
--hard        effacé        effacé    annulés → TOUT est perdu ⚠
```

---

## Commandes avancées

### 21. À quoi sert `git cherry-pick` ?

Il permet de copier un commit précis d'une branche vers la branche courante, sans merger toute la branche.

**Cas typique :** un fix critique sur une branche de feature doit être appliqué immédiatement sur `main`.

```bash
git checkout main
git cherry-pick abc1234   # SHA du commit à intégrer
```

---

### 22. À quoi sert `git bisect` ?

`git bisect` utilise une **recherche binaire** pour trouver quel commit a introduit un bug. On indique un commit "bon" et un commit "mauvais", et Git checkout automatiquement des commits intermédiaires pour tester jusqu'à isoler le commit fautif.

```bash
git bisect start
git bisect bad HEAD
git bisect good v1.2.0
# tester, puis : git bisect good / git bisect bad
git bisect reset  # fin
```

---

### 23. Qu'est-ce que `git stash` et quand l'utiliser ?

`git stash` met temporairement de côté les modifications non commitées dans une pile. Utile pour :
- switcher de branche rapidement sans commiter
- récupérer les modifs d'un collègue sur la même branche (`stash` → `pull` → `stash pop`)
- tester quelque chose sur un état propre

```bash
git stash          # mettre de côté
git stash pop      # récupérer et supprimer
git stash list     # voir la pile
```

---

### 24. Comment annuler toutes les modifications non commitées d'un seul coup ?

```bash
git reset --hard HEAD
```

Cela annule **toutes** les modifications trackées dans le working directory et le staging area. Les fichiers non-trackés ne sont pas touchés — pour les supprimer aussi : `git clean -fd`.

---

### 25. Comment afficher qui a modifié une ligne spécifique d'un fichier ?

```bash
git blame src/mon_fichier.py
# Pour une plage de lignes précise :
git blame -L 42,55 src/mon_fichier.py
```

Chaque ligne affiche le SHA du commit, l'auteur et la date de la dernière modification.

---

## Bonus — Questions de mise en situation

Ces questions testent le raisonnement, pas seulement la mémorisation de commandes.

**"Tu as commité sur `main` par erreur. Comment tu répares ça ?"**
> `git reset --soft HEAD~1` pour défaire le commit et garder les modifs, puis créer une branche et recommiter dessus. Si le commit a déjà été pushé, utiliser `git revert HEAD` pour ne pas réécrire l'historique public.

**"Un bug est apparu en production il y a 3 jours. Il y a 150 commits depuis la dernière version stable. Comment tu trouves le commit fautif ?"**
> `git bisect` : on indique le dernier tag stable (bon) et HEAD (mauvais), et Git fait une recherche binaire en 7-8 étapes au lieu de 150.

**"Tu es en plein développement d'une feature et un hotfix urgent arrive. Comment tu gères ça ?"**
> Deux options : (1) `git stash` pour mettre son travail de côté, switcher sur `main`, créer une branche `hotfix/`, corriger, puis revenir et `git stash pop`. (2) `git worktree add` pour créer un second dossier de travail sur la branche hotfix sans toucher à sa branche en cours.

**"Quelle est la différence entre `origin/main` et `main` ?"**
> `main` est la branche locale. `origin/main` est le tracking branch — la dernière version connue de `main` sur le remote `origin` (mise à jour par `git fetch`). Les deux peuvent diverger si des commits ont été poussés depuis le dernier fetch.
