# 00 — Configuration initiale de Git

← [Retour au sommaire](README.md)

> À faire une seule fois sur chaque nouvelle machine. Rien ne fonctionnera correctement sans ces étapes.

---

## Sommaire

- [Identité globale](#identité-globale)
- [Comportements par défaut](#comportements-par-défaut)
- [Aliases utiles](#aliases-utiles)
- [Clé SSH — connexion aux plateformes](#clé-ssh--connexion-aux-plateformes)
- [Vérifier la configuration](#vérifier-la-configuration)
- [Template `.gitconfig` complet](#template-gitconfig-complet)

---

## Identité globale

Chaque commit porte le nom et l'email de son auteur. Ces valeurs doivent correspondre à celles du compte GitHub / GitLab / Azure DevOps utilisé.

```bash
git config --global user.name "Prénom Nom"
git config --global user.email "email@example.com"
```

**Scope `--global`** : s'applique à tous les dépôts de la machine. Peut être surchargé au niveau d'un dépôt spécifique avec `--local` (sans le `--global`).

```bash
# Dans un dépôt précis — surcharger l'email (ex : compte pro vs perso)
git config --local user.email "pro@entreprise.com"
```

---

## Comportements par défaut

```bash
# Nommer la branche initiale "main" (et non "master")
git config --global init.defaultBranch main

# Éditeur de texte pour les messages de commit longs
git config --global core.editor "code --wait"    # VS Code
# git config --global core.editor "vim"          # Vim
# git config --global core.editor "nano"         # Nano

# git pull = fetch + rebase (pas fetch + merge)
# Évite les commits de merge inutiles lors d'un pull
git config --global pull.rebase true

# git push sans --set-upstream : configure automatiquement le tracking
# Disponible depuis Git 2.37
git config --global push.autoSetupRemote true

# Fin de ligne : normaliser automatiquement (important en équipe mixte Windows/Mac/Linux)
git config --global core.autocrlf input    # Mac / Linux
# git config --global core.autocrlf true  # Windows

# Afficher les diffs avec couleurs
git config --global color.ui auto

# Pager de diff plus lisible (optionnel)
git config --global core.pager "less -FRX"
```

---

## Aliases utiles

Les aliases sont des raccourcis pour des commandes longues. Ils sont définis dans `~/.gitconfig`.

```bash
# Historique graphique lisible — la commande la plus utile au quotidien
git config --global alias.lg "log --oneline --graph --all --decorate"

# Raccourcis classiques
git config --global alias.st "status"
git config --global alias.co "checkout"
git config --global alias.br "branch"
git config --global alias.last "log -1 HEAD"

# Voir ce qu'on s'apprête à pousser
git config --global alias.outgoing "log @{u}..HEAD --oneline"

# Voir ce qui est arrivé depuis le dernier pull
git config --global alias.incoming "log HEAD..@{u} --oneline"
```

### Utilisation

```bash
git lg          # → git log --oneline --graph --all --decorate
git st          # → git status
git co feat/x   # → git checkout feat/x
git last        # → git log -1 HEAD
```

---

## Clé SSH — connexion aux plateformes

Cloner en SSH évite de saisir ses identifiants à chaque push. C'est le mode recommandé.

### 1. Générer une clé SSH

```bash
# Générer une clé ED25519 (algorithme recommandé, plus court et plus sécurisé que RSA)
ssh-keygen -t ed25519 -C "email@example.com"

# Si la machine ne supporte pas ED25519 (rare)
ssh-keygen -t rsa -b 4096 -C "email@example.com"
```

Valider les prompts avec Entrée (chemin par défaut `~/.ssh/id_ed25519`, passphrase optionnelle).

### 2. Copier la clé publique

```bash
# Mac
pbcopy < ~/.ssh/id_ed25519.pub

# Linux
xclip -selection clipboard < ~/.ssh/id_ed25519.pub
# ou : cat ~/.ssh/id_ed25519.pub   (copier manuellement)

# Windows (PowerShell)
Get-Content ~/.ssh/id_ed25519.pub | Set-Clipboard
```

### 3. Ajouter la clé sur la plateforme

| Plateforme | Chemin |
|------------|--------|
| **GitHub** | `Settings → SSH and GPG keys → New SSH key` |
| **GitLab** | `Preferences → SSH Keys → Add new key` |
| **Azure DevOps** | `User Settings (icône) → SSH public keys → Add` |

Coller la clé publique (contenu de `id_ed25519.pub`) — **jamais la clé privée**.

### 4. Tester la connexion

```bash
# GitHub
ssh -T git@github.com
# → Hi <username>! You've successfully authenticated...

# GitLab
ssh -T git@gitlab.com
# → Welcome to GitLab, @<username>!

# Azure DevOps
ssh -T git@ssh.dev.azure.com
```

### 5. Cloner en SSH

```bash
# GitHub
git clone git@github.com:<organisation>/<projet>.git

# GitLab
git clone git@gitlab.com:<groupe>/<projet>.git

# Azure DevOps
git clone git@ssh.dev.azure.com:v3/<organisation>/<projet>/<repo>
```

---

## Vérifier la configuration

```bash
# Afficher toute la configuration active (globale + locale)
git config --list

# Afficher une valeur spécifique
git config user.name
git config user.email

# Afficher l'origine de chaque paramètre (system / global / local)
git config --list --show-origin

# Ouvrir le .gitconfig global dans l'éditeur
git config --global --edit
```

---

## Template `.gitconfig` complet

`~/.gitconfig` après avoir appliqué toutes les commandes ci-dessus :

```ini
[user]
    name = Prénom Nom
    email = email@example.com

[core]
    editor = code --wait
    autocrlf = input
    pager = less -FRX

[init]
    defaultBranch = main

[pull]
    rebase = true

[push]
    autoSetupRemote = true

[color]
    ui = auto

[alias]
    lg     = log --oneline --graph --all --decorate
    st     = status
    co     = checkout
    br     = branch
    last   = log -1 HEAD
    outgoing = log @{u}..HEAD --oneline
    incoming = log HEAD..@{u} --oneline
```

**Emplacement du fichier :**
- **Mac / Linux** : `~/.gitconfig`
- **Windows** : `C:\Users\<Utilisateur>\.gitconfig`
